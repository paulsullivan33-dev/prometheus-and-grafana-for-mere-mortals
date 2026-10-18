# Storage and Retention

**Scope:** where the samples live, what they cost,
and why Prometheus keeps weeks of data, not years.

## The shape of the local store

Prometheus writes to a local **time-series
database** (TSDB) in its data directory:

- Incoming samples land in the **head block** —
  in memory, backed by a write-ahead log (WAL) on
  disk so a crash loses almost nothing.
- Every two hours the head is cut into a
  **block**: immutable, compressed files on disk.
  Blocks are compacted together over time.
- Queries read across head and blocks alike; you
  never manage any of this by hand. You do watch
  two numbers: how many series exist, and how
  long you're keeping them.

Both are one API call (`/api/v1/status/tsdb`).
Measured on this guide's two-target setup after
about 20 minutes of scraping:

```
head series: 1827, label pairs: 987
top metrics by series: prometheus_http_request_duration_seconds_bucket (80),
  prometheus_http_response_size_bytes_bucket (72), prometheus_http_requests_total (64)
distinct metric names: 603
```

Sit with that first number. **Two targets — one
of them just an OS exporter — produced 1,827
series.** And the fattest contributor is
Prometheus's own HTTP instrumentation, because
histograms multiply: each handler × each status
code × each `le` bucket is a series. That's not
waste; it's the honest price of the percentile
queries in chapter 5. But it teaches the central
storage lesson: *series count, not target count,
is what grows your database.*

## What a sample costs

The arithmetic is worth doing once:

```
samples/day = series × (86,400 / scrape_interval_seconds)
```

1,827 series at 15 s → ≈ 10.5 million samples a
day. After block compaction, Prometheus's own
documentation puts the cost at roughly **1–2
bytes per sample** — call it 10–20 MB a day for
this toy setup, and scale the formula, not the
anecdote, to your fleet. (On disk here the whole
data directory was a few hundred kilobytes after
an hour, because nearly everything was still in
the head and WAL; compaction is what earns the
1–2 byte figure, and it happens on the two-hour
cycle.)

Three knobs follow from the arithmetic:

- **Scrape interval.** 60 s instead of 15 s
  quarters the sample rate. Sleepy machines don't
  need 15 s resolution.
- **Series count.** The cardinality chapter of
  your life: every label value multiplies. See
  Troubleshooting before adding a label.
- **Retention.** Below.

## Retention: the honest delete key

Default retention is **15 days** (confirmed on the
guide's server via `/api/v1/status/runtimeinfo`,
which reported `storageRetention: 15d`). Two
flags control it:

```
--storage.tsdb.retention.time=30d     # keep by age
--storage.tsdb.retention.size=20GB    # ...or by size; first hit wins
```

When retention expires data, it deletes whole
blocks. There is no downsampling, no "keep hourly
averages forever" tier in Prometheus itself.

That is a deliberate design stance, and it's the
answer to "why not keep everything": Prometheus
is the **working set** — the recent past, in full
resolution, on fast local disk, sized to the
machine it runs on. If you genuinely need years
of history or a hundred machines' worth in one
place, the ecosystem's answers are remote-storage
systems (Thanos, Cortex, Grafana Mimir, VictoriaMetrics
among them) that Prometheus can stream samples to
via **remote write**. They are named here, not
taught **(documented)** — a single-machine setup
will not miss them.

**Check:** `curl -s
http://127.0.0.1:9090/api/v1/status/tsdb` and find
your `numSeries`. Multiply it out with the
formula above before you promise anyone a
retention period.
