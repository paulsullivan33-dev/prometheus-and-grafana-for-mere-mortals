# Alerting

**Scope:** the two-program alert pipeline — rules
in Prometheus decide *what's wrong*, Alertmanager
decides *who hears about it* — with one alert
taken from first evaluation to delivered (and
resolved) notification, for real.

## Alert rules are just queries with a threshold

From `examples/alert-rules.yml`, the two rules
used throughout this guide:

```yaml
groups:
  - name: machine-health
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"

      - alert: HighCpuBusy
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 80
        for: 30s
        labels:
          severity: warning
        annotations:
          summary: "CPU busy above 80% on {{ $labels.instance }}"
          description: "Current value: {{ $value | printf \"%.1f\" }}%."
```

Every piece earns its place:

- **`expr`** is ordinary PromQL. When it returns
  anything, the alert exists; the value it
  returned is available as `$value`.
- **`for`** is the anti-flap delay. The expression
  must stay true *continuously* for this long
  before the alert fires. Without it, every
  one-sample blip becomes a notification, and
  within a week nobody reads notifications.
- **`labels`** are attached to the alert —
  `severity` is the one Alertmanager routing
  conventionally keys on.
- **`annotations`** are the human text. The
  `{{ ... }}` templates are evaluated when the
  alert fires: the payload below shows the
  rendered result, "Current value: 82.0%."

(Rules files can also hold **recording rules** —
queries Prometheus evaluates on a schedule and
stores as new series, so expensive expressions
are pre-computed. Same file format; not used by
this guide's examples **(documented)**.)

## Pending, firing, resolved — measured

The states, exactly as observed when a CPU burner
pinned both of the test machine's cores (the full
story is the Worked Example chapter):

| Time from burner start | State | What happened |
|---|---|---|
| +0 s | — | Expression false (baseline ≈ 51% busy) |
| ~+120 s | **pending** | Expression crossed 80 and stayed; the `for: 30s` clock started |
| ~+160 s | **firing** | Clock expired; Prometheus handed the alert to Alertmanager with value **82.0** |
| after burner stop | **resolved** | Expression went false; a resolved notification followed |

Pending is not a footnote: it's the state that
keeps a 20-second spike out of your pocket.
`/api/v1/alerts` and `/api/v1/rules` show the
states live; the rules endpoint reported, mid-event:

```
HighCpuBusy  state=firing  health=ok  activeAlerts=1
InstanceDown state=inactive health=ok  activeAlerts=0
```

You can check your rules file before any of this
with `promtool check rules` (chapter 4's output:
`SUCCESS: 2 rules found`).

## Alertmanager: routing, grouping, silence

Prometheus sends firing alerts to Alertmanager
and stops there. Alertmanager (0.34.1 here) does
the notification craft:

- **Routing** — a tree of matchers on labels
  (`severity=critical` goes to the pager,
  `severity=warning` goes to the chat room). This
  guide's `examples/alertmanager.yml` has the
  simplest honest route: everything to one
  receiver.
- **Grouping** — alerts sharing the `group_by`
  label values arrive as *one* notification
  listing all of them, after `group_wait`. The
  delivered payload's `groupLabels` were
  `{alertname="HighCpuBusy", instance="127.0.0.1:9100"}`
  because the config groups by exactly those.
- **Deduplication & repeats** — a firing alert is
  not re-sent every evaluation; `repeat_interval`
  (1 h here) re-notifies only while it keeps
  firing.
- **Silences** — planned mutes: "I know, I'm
  working on it." Created for real against the
  guide's Alertmanager via its API:

  ```
  POST /api/v2/silences  →  {"silenceID": "e13879fb-…"}
  GET  /api/v2/silences  →  matchers alertname=HighCpuBusy,
      instance=127.0.0.1:9100, comment "Silence while
      the load test is re-run on purpose"
  ```

  A silence matches labels, expires on schedule,
  and is the difference between a team that trusts
  alerts and a team that mutes the channel.

`amtool` is Alertmanager's `promtool`:
`amtool check-config alertmanager.yml` returned
`SUCCESS`, finding 1 receiver and 0 inhibit rules
(**inhibition** — a critical alert suppressing
related warnings — is the other noise control;
named, not configured here).

## The delivery itself

The receiver in the example config is a
**webhook**: Alertmanager POSTs JSON to a URL.
For the guide, that URL was a 30-line Python
script on the same machine; the first delivery it
received, trimmed (hostnames generalized):

```json
{
  "receiver": "local-webhook",
  "status": "firing",
  "alerts": [{
    "status": "firing",
    "labels": {"alertname": "HighCpuBusy",
               "instance": "127.0.0.1:9100",
               "severity": "warning"},
    "annotations": {
      "summary": "CPU busy above 80% on 127.0.0.1:9100",
      "description": "CPU on 127.0.0.1:9100 has been over 80% busy for 30 seconds. Current value: 82.0%."},
    "startsAt": "2026-10-18T00:41:10.066Z"
  }],
  "groupLabels": {"alertname": "HighCpuBusy", "instance": "127.0.0.1:9100"}
}
```

Everything in it is information Prometheus and
Alertmanager already had: the evaluated
annotations, the labels, the group. In production
the receiver is a chat integration, an email
gateway, or a paging service, all built into
Alertmanager as receiver types — the webhook is
the universal fallback, and the only one this
guide needed to prove the pipeline end to end.
Because the config sets `send_resolved: true`,
the same script later received the matching
`status: "resolved"` delivery with reason
*"all alerts resolved"*.

**Check:** `curl -s
http://127.0.0.1:9093/api/v2/alerts` shows what
Alertmanager currently holds. If Prometheus says
firing and Alertmanager says nothing, the problem
is between them — the `alerting:` block of
prometheus.yml, and nothing else.
