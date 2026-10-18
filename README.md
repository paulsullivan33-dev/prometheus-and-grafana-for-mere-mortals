# Prometheus and Grafana for Mere Mortals

Machines fail slowly before they fail loudly. A
filesystem fills for weeks; a queue grows for
days; a service leaks memory one graph-shaped
slope at a time. **Monitoring** is how you see
the slope while it's still cheap to fix — and the
open-source stack that does it almost everywhere
is two programs with a clean split of jobs:

- **Prometheus** collects numbers on a schedule,
  stores them, answers questions about them, and
  decides when something deserves an alert.
- **Grafana** draws the dashboards humans look
  at — it's the display layer, fed entirely by
  queries to Prometheus.

Between them sit **exporters**, small adapters
that publish a machine's or a service's vital
signs, and **Alertmanager**, which turns "an
alert is firing" into an actual notification
without drowning you.

This guide stands the whole stack up on one
machine, runs it against real data — including a
load spike generated on purpose — and follows one
alert from expression to delivered, resolved
notification.

It assumes you use a terminal and have edited
config files before. Networking basics (IP
addresses, ports, what HTTP is) are assumed;
everything about metrics is explained from zero.

## What this guide covers

- [**What monitoring is**](what-monitoring-is.md) —
  metrics vs. logs vs. traces, and who does what
  in the stack
- [**How Prometheus works**](how-prometheus-works.md) —
  the pull model, the text format it reads, labels,
  and the three metric types
- [**Exporters**](exporters.md) — the adapter
  pattern, and an honest tour of node_exporter
- [**Running Prometheus**](running-prometheus.md) —
  the config file line by line, targets, promtool,
  reloads
- [**PromQL**](promql.md) — selectors, `rate()`,
  aggregation, and the working set of queries,
  all run for real
- [**Storage and retention**](storage-and-retention.md) —
  what a sample costs, the 15-day default, and why
  Prometheus is a working set, not an archive
- [**Alerting**](alerting.md) — rules, pending →
  firing → resolved, Alertmanager's routing,
  grouping, and silences
- [**Grafana**](grafana.md) — datasources, panels,
  and dashboards as versionable JSON
- [**A worked example**](worked-example.md) — the
  full stack, a deliberate load spike, and a
  killed exporter, start to finish
- [**Running it for real**](running-it-for-real.md) —
  placement, sizing arithmetic, and the security
  facts stated plainly
- [**Troubleshooting**](troubleshooting.md) —
  symptom → fix, with the real error messages
- [**Glossary**](glossary.md) — the terms,
  collected in one place

## What it doesn't cover

Kubernetes-native monitoring (service discovery,
the Prometheus Operator), long-term and
multi-cluster storage systems (Thanos, Mimir,
remote write — named, not taught), Grafana's
logs/traces products, and alerting via Grafana's
own built-in alerting, which overlaps
Alertmanager and would double this guide's size.

## How to use this guide

Everything in the worked example was run for
real, with the official release binaries —
Prometheus 3.15.0, node_exporter 1.12.1,
Alertmanager 0.34.1, Grafana 13.2.4 — on one
ordinary machine: both scrape targets live, a
genuine CPU spike, an alert observed pending and
then firing, notifications (firing *and*
resolved) delivered to a local webhook receiver,
and a Grafana dashboard created and queried
through Grafana's HTTP API. The one boundary,
stated plainly: Grafana was verified **at the API
level** — datasource, dashboard, and a data query
through it — because the build environment had no
display; what the rendered dashboards look like
is marked **(documented)** wherever it comes up.
Quoted outputs are real captures, sanitized only
for hostnames and local paths.
