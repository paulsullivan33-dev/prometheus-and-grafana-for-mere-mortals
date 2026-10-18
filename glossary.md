# Glossary

**Alertmanager** — The companion program that
receives Prometheus's firing alerts and owns
notification: routing, grouping, deduplication,
silences, and delivery to receivers.

**Annotation** — Human-facing text attached to an
alert rule (summary, description), templated with
the alert's labels and value, rendered when the
alert fires.

**Block** — An immutable two-hour chunk of stored
samples in Prometheus's TSDB; blocks are compacted
into larger ones over time.

**Cardinality** — The number of unique series,
driven by unique label combinations. The main
scaling limit of a Prometheus server.

**Counter** — A metric that only increases (and
resets on restart). Use `rate()` before believing
it means anything.

**Datasource** — Grafana's named connection to a
system it queries — here, a Prometheus server.

**Evaluation interval** — How often Prometheus
runs rule expressions (`evaluation_interval`).

**Exporter** — An adapter program that publishes
some system's statistics as a Prometheus-format
`/metrics` page.

**Exposition format** — The plain-text format of
a `/metrics` page: HELP and TYPE comment lines,
then `name{labels} value` samples.

**Firing** — An alert whose expression has stayed
true past its `for:` duration, now handed to
Alertmanager. Before that, it is **pending**;
after the expression goes false, **resolved**.

**Gauge** — A metric that goes up and down:
temperature, free space, current load.

**Grafana** — The dashboard layer. It stores no
metrics; every panel is a live query to a
datasource.

**Head block** — The TSDB's in-memory recent
window, backed by the write-ahead log.

**Histogram** — A metric type that counts
observations into `le` buckets (plus `_sum` and
`_count`), enabling estimated percentiles via
`histogram_quantile()`.

**Instance** — The label Prometheus adds naming
the scraped target's address.

**Job** — A named group of targets in
`scrape_configs`, sharing scrape settings; added
to every series as the `job` label.

**Label** — A `key="value"` pair on a series.
Name + label set is a series' identity — and its
cost.

**node_exporter** — The standard operating-system
exporter for Linux/Unix machines.

**Panel** — One visualization on a Grafana
dashboard, bound to one or more queries.

**Pending** — An alert whose expression is true
but whose `for:` clock hasn't expired yet. The
anti-flap state.

**PromQL** — Prometheus's query language:
selectors, `rate()`, aggregations, functions.

**promtool / amtool** — The command-line checkers
and debuggers shipped with Prometheus and
Alertmanager (`check config`, `check rules`).

**Pull model** — Prometheus initiates every
scrape; targets just serve a page. The opposite
of agents pushing data in.

**Range vector** — A selector with a window —
`metric[5m]` — meaning "samples from the last
five minutes." What `rate()` consumes.

**Receiver** — An Alertmanager notification
destination: webhook, email, chat, pager.

**Recording rule** — A rule that stores a query's
result as a new series on a schedule, pre-paying
an expensive expression.

**Retention** — How long samples are kept before
whole blocks are deleted: 15 days by default,
set by age and/or size flags.

**Scrape** — One fetch of one target's `/metrics`
page; happens per target every `scrape_interval`.

**Series** — One metric name plus one unique label
set, and all its timestamped samples. The atom of
storage, memory, and cost.

**Silence** — A time-boxed Alertmanager mute
matching labels: the planned "we know" for
maintenance and known incidents.

**TSDB** — Prometheus's local time-series
database: head block, WAL, and compacted blocks
on disk.

**`up`** — The synthetic series Prometheus makes
per target: 1 when the last scrape succeeded, 0
when it didn't. The cheapest health signal there
is.
