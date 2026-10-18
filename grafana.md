# Grafana

**Scope:** the display layer — what it adds over
Prometheus's own web UI, the three concepts
(datasource, panel, dashboard) that everything
else hangs off, and dashboards as files you can
keep. Verified here at the API level; the rendered
visuals are **(documented)**, and this chapter
says exactly where that line falls.

## Why not just use Prometheus's UI?

Prometheus's built-in web UI is an excellent
*diagnostic* tool: an expression browser, the
targets page, the rules and alerts states. It is
not a dashboard system — no saved layouts, no
mixed panels from several queries at a glance, no
at-a-glance wall display. Grafana (13.2.4 here) is
the layer built for that. It owns no data: **every
panel is a query sent live to a datasource**, so a
Grafana dashboard and the PromQL you've learned
are the same thing wearing different clothes.

## Datasources: where the numbers live

A **datasource** is a named connection — here,
one entry: name `Prometheus`, type `prometheus`,
URL `http://127.0.0.1:9090`. Created via the API
for this guide:

```
POST /api/datasources
→ {"message": "Datasource added", "id": 1,
   "datasource": {"uid": "prometheus", "name": "Prometheus", ...}}
```

and Grafana immediately vouched for it:

```
GET /api/datasources/uid/prometheus/health
→ {"status": "OK",
   "message": "Successfully queried the Prometheus API."}
```

That health check is Grafana asking Prometheus a
real question and getting an answer — the same
test the UI's "Save & test" button performs. A
Grafana server usually has several datasources
(logs, SQL databases, other Prometheus servers);
panels name theirs explicitly, in this guide by
the uid `prometheus`.

## Panels, rows, variables

- A **panel** is one visualization — a time-series
  graph, a single stat, a gauge — bound to one or
  more queries, each a PromQL expression you could
  paste into Prometheus's own browser. If you can
  write the query (chapter 5), you can build the
  panel; the editor mostly adds units, thresholds,
  and colors.
- A **dashboard** is a grid of panels plus a time
  range, refreshed on an interval (30 s in the
  shipped example).
- **Variables** are dashboard-level dropdowns
  whose value is substituted into panel queries —
  one dashboard, a `$instance` picker, every
  machine you own. Mentioned at the conceptual
  level here; the worked example's dashboard is
  deliberately variable-free.

## Dashboards are JSON, and that's the point

A dashboard is a JSON document: title, panels,
each panel's queries and position. The one in
`examples/machine-dashboard.json` — four panels
(CPU busy %, Memory used %, Load, Disk used %) —
was created by posting the file itself:

```
POST /api/dashboards/db
→ {"status": "success", "uid": "machine-honestly",
   "url": "/d/machine-honestly/one-machine-honestly", "version": 1}

GET /api/dashboards/uid/machine-honestly
→ "One Machine, Honestly" — panels: CPU busy %,
  Memory used %, Load average (1m), Disk used % (root filesystem)
```

Consequences worth internalizing:

- Dashboards can live in **git**, be reviewed in
  diffs, and be provisioned onto a fresh Grafana
  by dropping the file in a directory — no
  clicking archaeology to rebuild them.
- The exported JSON is verbose and mentions
  internal IDs; treat it as build output you
  version, not prose you hand-edit for fun.
- Community dashboards (grafana.com's library,
  imported by number) are the same JSON from
  strangers — convenient, and worth reading before
  trusting, since a dashboard is also a bundle of
  queries your server will run.

## The honesty boundary

Everything above was verified without rendering a
single pixel: Grafana's API is the same interface
its own frontend uses, and a query *through*
Grafana returned the worked example's data (32
points spanning the load spike, min 36.2, max
85.8). What the dashboard **looks like** — panel
editors, legends, threshold colors, the Explore
view, screenshot/image rendering — was not
exercised here and is **(documented)**. Nothing
in this guide describes a screen it didn't see;
when you open `/d/machine-honestly/one-machine-honestly`
you'll be seeing it before the author of these
words did.

Two practical notes from the real run:

- **First start is slow.** Grafana initialized its
  database with 719 migrations and took the better
  part of two minutes to answer `/api/health`
  (which then reported `{"database": "ok",
  "version": "13.2.4"}`) on the modest test
  machine. Subsequent starts are quick.
- **Default login is `admin` / `admin`**, and
  Grafana insists you change it. That insistence
  is correct — chapter 10 has the security sermon
  in full.
