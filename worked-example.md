# A Worked Example

**Scope:** the whole stack, one sitting — install,
scrape, break the machine on purpose, watch the
query, the alert, the notification, and the
dashboard all tell the same story. Every output
below is from this exact procedure run for the
guide (Prometheus 3.15.0, node_exporter 1.12.1,
Alertmanager 0.34.1, Grafana 13.2.4).

## 1. Stand it up

Unpack the four official binaries somewhere
convenient. Put `examples/prometheus.yml`,
`examples/alert-rules.yml`, and
`examples/alertmanager.yml` in the working
directory, then start all four programs:

```
node_exporter --web.listen-address=127.0.0.1:9100
alertmanager --config.file=alertmanager.yml \
  --web.listen-address=127.0.0.1:9093
prometheus --config.file=prometheus.yml \
  --storage.tsdb.path=./data \
  --web.listen-address=127.0.0.1:9090 \
  --web.enable-lifecycle
grafana-server --homepath <grafana dir>   # serves :3000
```

Plus the tiny webhook receiver from the Alerting
chapter listening on port 5001 (any program that
accepts an HTTP POST will do — it's where the
notification lands).

**Check:** validate before you trust:

```
promtool check config prometheus.yml   → SUCCESS
promtool check rules  alert-rules.yml  → SUCCESS: 2 rules found
amtool check-config   alertmanager.yml → SUCCESS
curl 127.0.0.1:9090/api/v1/targets     → both jobs "up"
```

## 2. Take the baseline

A minute in, the machine's story at rest:

```
up{job="node"}                                  → 1
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100)
                                                → 51.4   (CPU busy %)
100 * (1 - (node_memory_MemAvailable_bytes
            / node_memory_MemTotal_bytes))      → 90.3   (memory used %)
node_load1                                      → 0.97
```

The test machine had other work, so 51% CPU and
90% memory are its honest resting state, not a
fault. Write your own baseline numbers down;
monitoring without a baseline is just weather
guessing.

## 3. Break it on purpose

Pin both cores for five minutes (two shell loops,
a stress tool, or — as here — a few lines of
Python each spinning a core):

```
load average climbs: node_load1 → 2.46 on a 2-core box
```

Within about a minute the CPU-busy expression is
climbing through the 70s. The range query tells
the story as 13 points: baseline values, a knee,
and **80.9** at the top.

## 4. Watch the alert do its three acts

From burner start, polling `/api/v1/alerts` every
20 seconds:

```
t+100s   (nothing — expression not yet over 80)
t+120s   HighCpuBusy: pending
t+140s   HighCpuBusy: pending
t+160s   HighCpuBusy: firing
```

The pending gap is `for: 30s` doing its job: the
expression crossed 80, stayed there, and only
then — at a measured value of **82.0%** — did
Prometheus hand the alert to Alertmanager.
Seconds later the webhook receiver's log held the
firing delivery (chapter 7 shows the payload).
When the burner stopped, the expression fell, and
a second delivery arrived: `status: "resolved"`,
reason *"all alerts resolved"*. If your alert
pipeline has never been tested end to end like
this, it is not an alert pipeline; it is a hope.

## 5. Put it on the dashboard

In Grafana, add the Prometheus datasource
(`http://127.0.0.1:9090`), then import
`examples/machine-dashboard.json` — or post it,
as this guide did:

```
POST /api/dashboards/db  (the examples file, as-is)
→ {"status": "success", "uid": "machine-honestly",
   "url": "/d/machine-honestly/one-machine-honestly"}
```

Asking Grafana itself for the panel's data,
across the spike window:

```
POST /api/ds/query   expr: the CPU-busy expression
→ 32 points: min 36.2, max 85.8, settling at 58.8
```

Same numbers, same shape as the raw Prometheus
range query — which is the whole point of the
architecture: Grafana adds presentation, never
its own version of the truth.

## 6. Second act: kill the exporter

Stop node_exporter and look at **Status →
Targets**:

```
job=node  health=down
lastError: Get "http://127.0.0.1:9100/metrics":
  dial tcp 127.0.0.1:9100: connect: connection refused

up{job="node"}  →  0
```

That `up == 0` is `InstanceDown`'s expression —
left down for a minute, it fires exactly the way
`HighCpuBusy` did. Restart the exporter and `up`
returns to 1. Two failure modes — *sick* and
*gone* — and the stack distinguishes them cleanly:
that distinction is what you actually bought.

**Check:** you now have a system that noticed
load, noticed absence, told a program you control,
and proved it in writing both times. Everything
after this chapter is operating it like you mean
it.
