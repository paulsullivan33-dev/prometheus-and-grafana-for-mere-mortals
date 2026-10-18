# Troubleshooting

**Scope:** the failures you'll actually meet,
symptom → fix. Messages and numbers are verbatim
from this guide's runs unless marked (documented).

## Target shows DOWN in Status → Targets

**Symptom:** a job's target is red, with an error
like this real one:

```
Get "http://127.0.0.1:9100/metrics": dial tcp 127.0.0.1:9100: connect: connection refused
```

**Fix:** Prometheus is telling you it knocked and
nobody answered. Work outward: is the exporter
process running at all? Is it listening on the
address and port the config names (config says
`127.0.0.1:9100`; exporter bound elsewhere —
`0.0.0.0`, a different port — is the classic
mismatch)? Does `curl <scrapeUrl>` from the
Prometheus machine succeed? Note that Prometheus
considers it no tragedy if *you* can fetch the page
but the error persists — re-check after the next
scrape interval, and confirm `up{job="..."}` flips
back to 1, as it did in the worked example.

## The query returns nothing — and "success"

**Symptom:** a PromQL query you believe in returns
an empty result, with `status: "success"`.

**Fix:** this is usually a label typo, and
Prometheus cannot know — a selector that matches
zero series is a valid query (the live server
happily returned success-and-empty for
`{mode="idl"}`). Debug by subtraction:

1. Query the bare metric name. Nothing? The target
   isn't scraped or the metric name is wrong —
   check `/api/v1/label/__name__/values` (603
   distinct names on the guide's server) or the
   target's own `/metrics`.
2. Add selectors back one at a time until the
   result vanishes; the last one is your typo.
   Check its values with
   `/api/v1/label/mode/values`-style calls.
3. If it's a `rate()` query: the window must
   contain at least two samples. `[30s]` at a 15 s
   scrape interval is borderline by arithmetic,
   not by superstition.

## The panel is empty but the query works in Prometheus

**Symptom:** Grafana shows "No data" for an
expression that returns values in Prometheus's own
browser.

**Fix:** the query is fine, so the plumbing isn't.
In order: the panel's **datasource** (does it
point at the right Prometheus? Grafana's
datasource health endpoint answers this in one
call — here it returned *"Successfully queried the
Prometheus API"*), the dashboard **time range**
(an empty *now − 5 minutes* proves nothing about
an event an hour ago), and a subtle one — the
expression returns data but the panel's
visualization can't show a single instant value.
(documented) UI details vary by version; the
datasource and time-range checks don't.

## Everything is slow and the disk is shrinking: cardinality

**Symptom:** Prometheus uses more memory every
week; queries drag; the TSDB status shows series
in the hundreds of thousands.

**Fix:** you have a **cardinality** problem — too
many unique label combinations. Ground truth,
from the guide's toy setup: two targets already
made **1,827 series**, and each scrape cycle
ingests ~1,800 samples (863 from node_exporter,
972 from Prometheus itself). One bad label — a
user ID, a full URL, a session token — multiplies
a metric's series by every distinct value it ever
takes. Find the offenders with
`/api/v1/status/tsdb` (`seriesCountByMetricName`,
`labelValueCountByLabelName` — the guide's top
label by value-count was `le` at 88, which is just
histograms being histograms). Fixes, in order of
preference: stop exporting the offending metric
or label (exporter flags, or Prometheus
`metric_relabel_configs` drops), never put
unbounded values in labels in your own
instrumentation, and treat anyone proposing a
`user_id` label as a performance incident that
hasn't happened yet.

## Alerts fire in Prometheus; nothing arrives

**Symptom:** `/api/v1/alerts` shows firing;
Alertmanager is silent.

**Fix:** walk the hand-off. Is the `alerting:`
block pointing at the real Alertmanager address?
`curl http://<alertmanager>/api/v2/alerts` shows
what it holds — empty means Prometheus isn't
delivering (config/address); populated but no
notification means routing or the receiver
(check the route tree with `amtool config
routes`, look for a **silence** matching your
alert — created during a past incident and
forgotten is the genre classic — and respect
`group_wait`: a group holds its first delivery
for that long *by design*).

## Numbers look time-shifted; rates are nonsense

**Symptom:** graphs from one target seem offset
from reality; events line up a few seconds wrong.

**Fix:** clock skew. Prometheus stamps samples
with *its own* clock when it scrapes, so its data
is self-consistent — but exporters like
node_exporter also export *their* clock, and
anything comparing the two (or alert timestamps
against your wall clock) exposes drift:

```
node_time_seconds - time()   →  -3.15
```

That real reading says the target's clock sat
3.15 s behind when scraped. A few seconds is
noise (scrape timing included); minutes means a
target without working NTP, and the fix is to fix
the clock, not the query.

**Check:** most entries above were found the same
way in the real runs — targets page first,
`up` second, one query at a time third. The order
matters more than any single trick.
