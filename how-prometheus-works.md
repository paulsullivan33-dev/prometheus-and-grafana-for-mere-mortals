# How Prometheus Works

**Scope:** the pull model, the text format every
target speaks, and the label system that is —
entirely — Prometheus's data model.

## Prometheus comes to you

Most monitoring folklore involves agents *pushing*
numbers to a central server. Prometheus inverts
it: **Prometheus pulls.** Every target exposes an
HTTP endpoint, conventionally `/metrics`, and
Prometheus fetches it on a schedule (every 15
seconds by default in this guide's config). One
fetch is a **scrape**.

The pull model buys three things:

- **Health for free.** A failed scrape *is* a
  down signal — Prometheus knows the target didn't
  answer because it was there asking. Push systems
  can't distinguish "target died" from "network
  ate the message."
- **Targets need no configuration** beyond serving
  the page. Adding a machine to monitor is a
  Prometheus-side change.
- **You can look at the raw material yourself.**
  `curl http://127.0.0.1:9100/metrics` shows
  exactly what Prometheus sees. No binary protocol,
  no agent to interrogate.

## The exposition format, line by line

A `/metrics` page is plain text. Real lines,
captured from node_exporter and from Prometheus
itself:

```
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 7611.12
node_cpu_seconds_total{cpu="0",mode="user"} 422.68
```

| Line kind | Meaning |
|---|---|
| `# HELP` | One sentence of documentation for the metric |
| `# TYPE` | The metric's type: `counter`, `gauge`, `histogram`, or `summary` |
| Samples | Metric name, optional `{labels}`, current value |

That is the whole format. `node_cpu_seconds_total`
is a running total of seconds CPU 0 has spent in
each mode since boot — 7,611.12 of them idle.

## Labels are the data model

Prometheus stores no tables and has no schema to
design. What it stores are **series**: one metric
name plus one unique set of **labels** (key=value
pairs). `node_cpu_seconds_total{cpu="0",mode="idle"}`
and `node_cpu_seconds_total{cpu="1",mode="idle"}`
are two different series.

Prometheus adds two labels to everything it
scrapes: `job` (which scrape job collected it) and
`instance` (the target's address). When you query,
you select series by name and labels — that's the
entire trick, and the Troubleshooting chapter
shows how a one-letter label typo makes a query
return *successfully empty*.

The flip side is **cardinality**: every unique
label combination is another series to store and
keep in memory. A label with a handful of values
(`mode`, `cpu`) is fine. A label whose value is a
user ID or a URL has thousands — and is the
classic way to make a Prometheus server fall over
(see Troubleshooting).

## The three types that matter

**Counter** — only goes up (resets to zero when
the process restarts, which Prometheus handles).
Bytes transferred, requests served, CPU seconds.
A counter alone is boring; its *rate of increase*
is the interesting number, which is why PromQL has
`rate()` (chapter 5).

**Gauge** — goes up and down: temperature, free
memory, number of connections right now. Read it
directly.

**Histogram** — counts observations into buckets
so you can later estimate percentiles. Real lines
from Prometheus measuring its own query endpoint:

```
# TYPE prometheus_http_request_duration_seconds histogram
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/query",le="0.1"} 21
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/query",le="+Inf"} 21
```

Read: of 21 queries observed, all 21 finished
within 0.1 s — the `le` ("less or equal") buckets
are cumulative. The companion `_sum` and `_count`
series carry the total time and the observation
count. Percentiles from histograms are estimates —
good ones — computed by `histogram_quantile()` in
chapter 5.

**Check:** fetch any target's `/metrics` with
`curl` and identify one counter, one gauge, and
its HELP/TYPE lines. You've just read the same
page Prometheus reads.
