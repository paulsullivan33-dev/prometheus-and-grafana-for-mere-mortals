# What Monitoring Is

**Scope:** the problem this whole stack exists to
solve, and the two questions every monitoring system
is really answering — *is it healthy?* and *is it
getting worse?*

## Three kinds of evidence

When a machine misbehaves, you can look at three
different kinds of evidence. They are not
interchangeable.

| Kind | What it is | The question it answers |
|---|---|---|
| **Metrics** | Numbers sampled on a schedule: CPU busy, bytes free, requests per second | "How much, how fast, trending which way?" |
| **Logs** | Timestamped text written when events happen | "What exactly occurred, in what order?" |
| **Traces** | One request's journey through many services, timed hop by hop | "Where did *this* request spend its time?" |

This guide is about the first row. Metrics are the
smoke detector: cheap to collect, always on, and
they tell you *that* something is wrong and roughly
where. Logs are the investigation afterwards.
Traces matter when one slow request crosses five
services; most small setups never need them.

## Why numbers over time

A single number is nearly useless. "Disk 78% full"
says nothing until you know last week it was 71%
and the week before 64% — at which point it says:
*you have about three weeks, and here is the graph
to prove it.*

That is the real product of a monitoring system:
**measurements with a past**, so that "unusual"
means something more precise than a feeling. Every
design choice in Prometheus — sampling on an
interval, storing samples with their timestamps,
a query language built around rates and ranges —
follows from that one idea.

## The division of labor

This guide covers a stack of cooperating programs,
not one program. Knowing who does what is half the
battle:

| Program | Job |
|---|---|
| **Exporters** | Sit next to a thing (an OS, a database, an app) and publish its numbers in a format Prometheus can read |
| **Prometheus** | Visits every exporter on a schedule, stores what it finds, answers queries, evaluates alert rules |
| **Alertmanager** | Takes Prometheus's firing alerts and handles notification: grouping, deduplication, silences, delivery |
| **Grafana** | Draws dashboards — it queries Prometheus (and other sources) and renders panels for humans |

Two consequences of that split surprise beginners:

1. **Prometheus has almost no pretty pictures.**
   Its built-in web UI is a diagnostic tool. The
   dashboards people show in screenshots are
   Grafana's.
2. **Prometheus never sends you an email.**
   It decides an alert is *firing*; Alertmanager
   decides who hears about it, how often, and how
   to stop the noise when you already know.

## What this stack is good at — and not

Good at: infrastructure numbers from machines,
containers, and services; spotting trends days
before they become outages; answering "when did
this start?" with evidence.

Not good at: anything per-event (one specific
failed request belongs in a log), long-term
archiving measured in years (see the Storage
chapter — Prometheus is a working set, not a
vault), and business dashboards (Grafana can query
a SQL database too, but that's another guide's
territory).

**Check:** if you can state which of the four
programs collects, stores, notifies, and displays,
you have the map. The rest of the guide walks it
one program at a time.
