# Writing Criteria

Every guide in this repo is written to the same standard. If you're
adding or editing a guide — human or AI — hold it to these.

1. **Plain words, real ideas.** Simple language, never dumbed-down
   concepts. Explain jargon on first use; collect terms in the
   glossary.
2. **No hype.** No marketing adjectives, no "revolutionary," no
   promises about what something will do for the reader.
3. **Honest about limits.** Say what the thing *can't* do. State
   uncertainty instead of smoothing it over. Validation notes include
   what remains unverified.
4. **No invented numbers.** Benchmarks, defaults, and specs are either
   measured or qualified with "depends on your setup." If it wasn't
   tested, say so.
5. **Checkable.** Commands are meant to be run, with "Check:" blocks
   showing what success looks like. Sources are linked, not asserted.
6. **Respect the reader's setup.** Call out Windows/Mac/Linux
   differences. Don't assume a GPU, a budget, or prior experience.
7. **Safety isn't skipped.** Real risks get stated plainly, not buried.
8. **Scoped.** Each guide says what it covers and points elsewhere for
   the rest. No guide pretends to be the whole subject.
9. **Structure over prose.** Tables, before/after examples,
   symptom → fix. Scannable beats essay.

## Verification notes for this guide

Standard 3 and 4 applied to this guide, specifically.
All four components ran as official standalone release
binaries on one machine, bound to 127.0.0.1 (no
containers — the build sandbox's kernel can't host
them, and none were needed): Prometheus and promtool
3.15.0, node_exporter 1.12.1, Alertmanager and amtool
0.34.1, Grafana 13.2.4.

**Verified by running:**

- node_exporter live: its full `/metrics` page
  (1,445 lines, 813 `node_*` samples); the samples
  quoted in the Exporters chapter are verbatim
  captures.
- Config validation with the real tools:
  `promtool check config` (SUCCESS, 1 rule file),
  `promtool check rules` (SUCCESS, 2 rules),
  `amtool check-config` (SUCCESS, 1 receiver).
- Prometheus scraping both shipped jobs; targets
  API showed both `health=up` with real scrape
  durations; `up` queries returned 1.
- Every PromQL expression in chapter 5 and the
  cheat sheet was executed against the live server:
  the network-rate query against the lab machine's
  real interface name (the cheat sheet shows a
  placeholder device), and the disk queries —
  used %, free GiB, and `predict_linear` in both
  value form (returned ≈ 8.4e10, i.e. not filling)
  and alert form (`< 0`, correctly empty) —
  against the lab's data-volume mountpoint where
  the guide's text uses `/`. Instant and range
  queries, `rate()`, aggregation by mode and by
  instance, memory/disk percentages, and
  `histogram_quantile` (0.09 s p90 on Prometheus's
  own query handler) are real returns.
- Config reload via `POST /-/reload` (HTTP 200;
  the server's log shows the reload completing;
  the reload-timestamp metric advanced).
- The alert lifecycle end to end: a CPU burner
  pinned both cores; the shipped HighCpuBusy rule
  was observed pending (~+120 s), then firing
  (~+160 s, value 82.0) via `/api/v1/alerts` and
  `/api/v1/rules`; Alertmanager held it as active;
  a local webhook receiver (a short Python script)
  received the firing delivery and, after the
  burner stopped, the resolved delivery
  ("all alerts resolved"). An Alertmanager silence
  was created and listed via `/api/v2/silences`.
- Target failure and recovery: node_exporter was
  stopped; the targets API reported `health=down`
  with the verbatim "connection refused" error and
  `up` returned 0; on restart `up` returned 1.
- Storage facts measured, not asserted: 1,827 head
  series and 987 label pairs from two targets
  (`/api/v1/status/tsdb`), 603 distinct metric
  names, ~1,800 samples per scrape cycle
  (`scrape_samples_scraped`), and default retention
  `15d` from `/api/v1/status/runtimeinfo`.
- Grafana at the API level: `/api/health`
  (database ok, version 13.2.4); datasource created
  via `POST /api/datasources` with uid `prometheus`
  and its health endpoint answering "Successfully
  queried the Prometheus API"; the exact dashboard
  JSON shipped in `examples/` was created via
  `POST /api/dashboards/db` (status success, uid
  `machine-honestly`) and read back with all four
  panels; a PromQL query through
  `POST /api/ds/query` returned 32 points spanning
  the load spike (min 36.2, max 85.8).

**Documented-only (not run), and why:**

- Grafana's rendered visuals — what the dashboard
  looks like, panel editors, legends, Explore,
  image rendering. The build environment has no
  display; the API Grafana's own frontend uses was
  verified instead, and chapters say so wherever
  the visual side comes up.
- The dashboard's disk panel ships with
  `mountpoint="/"` (a reader's root filesystem).
  The sandbox has no `/` mount in node_exporter's
  output; the identical query form was verified
  against the lab's data volume (7.5% used) and
  through Prometheus directly. The panel's other
  three queries were verified through Grafana
  itself (the CPU panel) or against Prometheus
  with identical expressions (memory, load).
- Recording rules: explained, not present in the
  shipped examples and not evaluated.
- Non-webhook receivers (email, chat, pager
  services): configuration discussed from the
  Alertmanager documentation; only the webhook
  receiver was run.
- Alertmanager clustering/HA, inhibition rules
  (named, not configured), service discovery for
  targets, federation, remote write and the
  long-term storage systems, systemd units, and
  reverse-proxy TLS/auth in front of the stack:
  documented from upstream documentation, not run.
- Windows and macOS specifics: node_exporter is a
  Unix exporter (Windows has windows_exporter);
  the other three binaries publish Windows/macOS
  builds, which were not exercised here.
- Quoted outputs are real captures, sanitized only
  where a sandbox hostname or home-directory path
  appeared (replaced with generic equivalents);
  values, messages, and timings are as returned.
