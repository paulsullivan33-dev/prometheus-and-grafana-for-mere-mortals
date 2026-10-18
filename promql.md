# PromQL

**Scope:** the query language — just enough of it,
properly understood, to answer the questions a
small system actually asks. Every query in this
chapter was run against the live two-target setup
from the previous chapters; values are the real
returns, lightly trimmed.

## Two kinds of answer

An **instant query** evaluates at one moment:

```
node_load1
→ instance="127.0.0.1:9100", job="node"  ⇒  0.97
```

A **range query** evaluates the same expression at
many moments and returns a series of points. The
web UI's graph view is a range query; so is every
Grafana panel. Over four minutes spanning the
load spike in the worked example, the CPU-busy
expression returned 13 points (at a 15 s step)
climbing from the 40–75 range to 80.9 — the whole
spike, as data.

In the HTTP API these are `/api/v1/query` and
`/api/v1/query_range`, returning `vector` and
`matrix` results respectively. Everything else in
this chapter is expressions — the API just carries
them.

## Selectors: asking for series by name

A bare metric name returns every series with that
name. **Label selectors** narrow it:

```
node_cpu_seconds_total{mode="idle"}
```

There are four matchers: `=` exact, `!=` not
equal, `=~` regex, `!~` not-regex. One warning
that saves real pain: a selector that matches
nothing is *not an error*. `node_cpu_seconds_total{mode="idl"}`
returns `status: "success"` and an empty result —
verbatim from the live server. PromQL cannot know
you typo'd. Chapter 11 has the debugging routine.

## Why counters need rate()

Counters only ever rise, so their raw values are
meaningless across restarts and uptimes. `rate()`
takes a **range vector** — a selector with a time
window — and returns the per-second increase,
handling resets:

```
rate(node_cpu_seconds_total{mode="idle"}[5m])
→ cpu="0" ⇒ 0.126    cpu="1" ⇒ 0.129
```

Read: on each core, the idle-seconds counter grew
by about 0.13 s per second — the cores were idle
~13% of the time in that window. The window needs
at least two samples inside it: at a 15 s scrape
interval, `[5m]` holds ~20 samples, `[2m]` about
8, and `[30s]` is skating on two. Prefer `[5m]`
for graphs, `[2m]` when an alert must react
sooner — and know the trade you're making.

## The greatest hits, run for real

**CPU busy %** — the complement of idle:

```
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

`avg()` collapses the per-core series into one
number. The alerting-shaped version keeps one row
per machine: swap `avg(...)` for
`avg by (instance) (...)` and give the window as
`[2m]`. On the test machine that expression read
**51.4** at rest and **80.9** mid-spike.

**Where did the CPU time go?** Aggregation with
`by`:

```
sum by (mode) (rate(node_cpu_seconds_total[5m]))
→ idle 0.256 · user 0.011 · system 0.015 · iowait 0.016 · softirq 0.001 · steal 0.000
```

(On a 2-core box the modes sum to at most 2.0.)
This is the query to run before believing a
"CPU problem": user time means your programs;
iowait means the disk is the queue; steal means
the *host* is oversubscribed and your neighbours
are eating your lunch.

**Memory used %:**

```
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
→ 90.3
```

The test machine was busy with other work — that
number is honest, not broken. Note the formula
uses `MemAvailable`, the kernel's own estimate of
what's freeable (it credits the page cache), not
`MemFree`, which would cry wolf on any healthy
Linux box.

**Disk used %:**

```
100 - (100 * node_filesystem_avail_bytes{mountpoint="/"}
            / node_filesystem_size_bytes{mountpoint="/"})
```

(Shown with the reader's root filesystem; the
live run targeted the test machine's data volume
and returned 7.5.) Add the oldest trick in
operations — extrapolate the disk's own trend:

```
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 4*24*3600) < 0
```

This is true when free space, falling at its
6-hour rate, hits zero within four days: the
"disk filling" alert that has saved more weekends
than any other in this guide.

**Percentiles from histograms:**

```
histogram_quantile(0.9, sum by (le) (rate(prometheus_http_request_duration_seconds_bucket[5m])))
→ 0.09
```

The 90th percentile of Prometheus's own query
latency: 0.09 s. Two non-obvious parts: you
`sum by (le)` *first* (the function needs the
bucket series grouped by their `le` label), and
the input is a rate over buckets — percentiles of
*recent* observations, not of all time.

**Check:** the full set lives in
`examples/promql-cheatsheet.txt`. Run each against
your own node target; the values will differ, the
shapes shouldn't.
