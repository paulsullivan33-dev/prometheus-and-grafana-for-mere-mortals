# Running It for Real

**Scope:** the operational decisions the worked
example postponed — where the stack lives, how big
it needs to be, and the security facts that are
not optional.

## Where Prometheus should live

The worked example ran everything on one machine,
which is right for learning and wrong as a final
answer, for one structural reason: **a monitor on
the machine it watches dies with the machine.**
Total death — kernel panic, power, network — is
precisely the case where the only signal is a
scrape failing from *somewhere else*.

The minimal honest topology for more than one
machine:

- node_exporter (and service exporters) on **every
  machine** you care about.
- **One Prometheus on a different machine** than the
  most important things it watches — the spare box,
  the NAS, the small always-on computer. One
  Prometheus scrapes many targets happily; that's
  its design center.
- Alertmanager next to Prometheus. Grafana wherever
  is convenient — it's replaceable presentation.

## Sizing, without folklore

Chapter 6's formula is the sizing tool:
samples/day = series × (86,400 / interval), at
roughly 1–2 stored bytes per sample after
compaction. Two worked points:

- This guide's setup: ~1,800 series, 15 s → ~10M
  samples/day → a few tens of MB/day. A 15-day
  retention fits in well under a gigabyte.
- A home fleet of five chatty machines might reach
  10,000 series; at 30 s that's ~29M samples/day
  → still well under a gigabyte a day.

CPU is rarely the issue at this scale; memory
tracks series count (the head block holds recent
samples for every series), and disk is the
retention formula above. When in doubt: lengthen
the interval before buying hardware. Going from
15 s to 60 s is a 4× discount on everything, and
almost no home question needs finer grain.

## Security, stated plainly

This is the chapter the criteria insist on:

- **Prometheus has no built-in authentication and
  no TLS** in the setup shown. Anyone who can reach
  port 9090 can read every metric you collect (a
  surprisingly good map of your network) *and* —
  with the lifecycle API enabled — reload your
  config. Alertmanager's API can create silences:
  an attacker who can silence your alerts owns
  your blind spots. node_exporter will cheerfully
  describe your machine to any asker.
- Therefore: **never expose these ports raw to the
  internet.** Bind to localhost (as the examples
  do), to a LAN you trust, or to a VPN interface —
  and put internet-facing access, if you truly need
  it, behind a reverse proxy that adds TLS and
  authentication **(documented)**.
- **Grafana is the exception** — it has real user
  accounts — and still: change the default
  `admin`/`admin` immediately, and don't publish it
  to the internet without the same proxy treatment
  unless you enjoy strangers in your dashboards.
- Metrics themselves are sensitive in aggregate:
  they reveal machine names, software versions in
  labels, work hours, and habits. Treat read
  access accordingly.

## Comforts worth adding later

- **Run them as services.** Packaged installs and
  systemd units (restart on boot, logs in the
  journal) are the normal production shape; the
  worked example ran binaries by hand so that
  nothing hid behind a package **(documented)**.
- **Config in git.** prometheus.yml, the rules,
  Alertmanager's config, and dashboard JSONs are
  all text — the same discipline as the rest of
  this series' tooling.
- **Monitor the monitor.** A second tiny check —
  even a scheduled `curl` of `/-/healthy` from
  another machine that complains loudly — covers
  the "Prometheus itself died quietly" case.
- **Federation and long-term stores** (Thanos,
  Mimir, remote write) exist for many-cluster and
  many-year problems. At the scale of this guide
  they are solutions looking for a problem you
  don't have yet **(documented)**.

**Check:** from another device, try to open your
Prometheus port. If it answers, so can anyone
else's device — fix the binding before anything
else on this page.
