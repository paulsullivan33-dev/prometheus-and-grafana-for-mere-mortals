# Running Prometheus

**Scope:** the server itself — one config file, how
to know your targets are being scraped, how to
prove the config is valid before Prometheus does,
and how to reload without a restart.

Prometheus is a single binary plus a data
directory. Run it like this (versions used for
this guide: Prometheus 3.15.0, node_exporter
1.12.1, Alertmanager 0.34.1, Grafana 13.2.4 — all
official release binaries):

```
prometheus --config.file=prometheus.yml \
  --storage.tsdb.path=./data \
  --web.listen-address=127.0.0.1:9090 \
  --web.enable-lifecycle
```

`--web.enable-lifecycle` turns on the reload
endpoint used below. Binding to `127.0.0.1` keeps
the server reachable only from this machine —
chapter 10 explains why that matters.

## The config, taken apart

This is `examples/prometheus.yml`, the config
every capture in this guide was made with:

```yaml
global:
  scrape_interval: 15s      # how often targets are scraped
  evaluation_interval: 15s  # how often alerting/recording rules run

rule_files:
  - "alert-rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["127.0.0.1:9093"]

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["127.0.0.1:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["127.0.0.1:9100"]
```

Line by line:

- **`global`** sets defaults for everything below.
  `scrape_interval` is the heartbeat of the whole
  system: each target's `/metrics` is fetched this
  often, and every sample you've seen exists
  because of it. 15s is a sane default; 1-minute
  intervals are fine for sleepy machines and halve
  your storage.
- **`rule_files`** names the files holding alert
  (and recording) rules — chapter 7's territory.
  Paths are relative to the config file.
- **`alerting`** tells Prometheus where
  Alertmanager lives. Prometheus evaluates rules;
  it hands *firing* alerts to this address and
  washes its hands of delivery.
- **`scrape_configs`** is the target list. Each
  entry is a **job** — a named group of targets
  scraped the same way. `static_configs` is the
  simple, honest way to list them: literal
  addresses. (Prometheus can also *discover*
  targets from cloud APIs, Kubernetes, DNS and
  more; discovery is real and useful, and
  **(documented)** here — static lists are what a
  first setup needs.)
- The **first job scrapes Prometheus itself.**
  Prometheus publishes its own `/metrics` (ingest
  rates, query timings, storage stats). Monitoring
  the monitor is not vanity: when numbers look
  wrong, these series are how you check the
  checker. It also guarantees the setup has a
  target that can only be down if everything is.

Every scraped series gains the labels `job` (the
job name) and `instance` (the target address) —
that's how later chapters' queries can say
`up{job="node"}`.

## Check the config before Prometheus does

`promtool` ships in the same download. Two
commands earn their keep:

```
$ promtool check config prometheus.yml
Checking prometheus.yml
  SUCCESS: 1 rule files found
 SUCCESS: prometheus.yml is valid prometheus config file syntax

$ promtool check rules alert-rules.yml
Checking alert-rules.yml
  SUCCESS: 2 rules found
```

Both are verbatim from the runs behind this
guide. Run them after every edit; a config
Prometheus rejects at reload time keeps the old
config running (good), but only the log tells you
so (easy to miss).

## Are the targets actually up?

The web UI's **Status → Targets** page is the
ground truth, and the same data is one API call:

```
$ curl -s 'http://127.0.0.1:9090/api/v1/targets?state=active'
```

At the start of the worked example it reported:

```
node        http://127.0.0.1:9100/metrics  health=up  lastScrapeDuration=0.0168s
prometheus  http://127.0.0.1:9090/metrics  health=up  lastScrapeDuration=0.0057s
```

(Trimmed from the real JSON.) Each target shows
its last error if it has one — remember that page
when chapter 11 sends you hunting a target that's
down. And the very first query everyone runs:

```
up{job="node"}   →  1
```

`up` is a series Prometheus synthesizes per
target: 1 if the last scrape succeeded, 0 if it
didn't. It is the cheapest health signal in the
whole system, and chapter 7 builds an alert on
it.

## Reloading without restarting

Config and rule edits don't need a restart. Send
`SIGHUP` to the process, or — with the lifecycle
flag from the run command above:

```
$ curl -X POST http://127.0.0.1:9090/-/reload
```

Verbatim from Prometheus's log when this was run
for the guide:

```
msg="Loading configuration file" filename=.../prometheus.yml
msg="Completed loading of configuration file" ... scrape=8.46ms ... rules=1.09ms
```

The reload is atomic and fast (about a tenth of
a second end to end here): if the new config is
invalid, Prometheus logs the error and keeps the
old one. **Check:** after a reload, query
`prometheus_config_last_reload_success` — it
reads 1 when the last reload worked.
