# Exporters

**Scope:** the adapter pattern that connects
Prometheus to everything it monitors, and a tour of
the one exporter almost everyone runs first.

## The adapter idea

Prometheus speaks one text format (previous
chapter). The world does not. An **exporter** is a
small program that stands next to a thing — an
operating system, a database, a piece of network
gear — reads its native statistics, and republishes
them as a `/metrics` page.

That separation is why the ecosystem works:
Prometheus itself never needs to learn how MySQL
counts queries or how a switch reports port
errors. Whoever knows the thing writes its
exporter.

You will meet three kinds:

| Kind | Examples | Notes |
|---|---|---|
| **System exporters** | node_exporter (Linux/Unix), windows_exporter | One per machine, maintained by the Prometheus project or its community |
| **Service exporters** | mysqld_exporter, postgres_exporter, blackbox_exporter (probes websites and endpoints), snmp_exporter | One per service or device family |
| **Built-in instrumentation** | Applications linked with a Prometheus client library | The app serves its own `/metrics`; no adapter needed. The gold standard where you control the code |

## node_exporter, honestly toured

node_exporter is the canonical machine exporter:
run it on a Linux host and it publishes the
operating system's vital signs. On the test
machine its page was **1,445 lines**, of which
**813 lines were samples** in the `node_*` family.
Nobody reads them all. These are the ones you will
actually use, verbatim from that capture:

```
node_cpu_seconds_total{cpu="0",mode="idle"} 7611.12
node_cpu_seconds_total{cpu="0",mode="user"} 422.68
node_memory_MemTotal_bytes 8.320606208e+09
node_memory_MemAvailable_bytes 1.032441856e+09
node_load1 1.73
node_load5 0.77
node_boot_time_seconds 1.792274691e+09
node_context_switches_total 3.2720784e+07
node_network_receive_bytes_total{device="host0"} 9.05033185e+08
node_filesystem_size_bytes{device="/dev/mapper/rv",fstype="btrfs",mountpoint="/home/user"} 1.07355308032e+11
```

Things worth noticing:

- **The names tell you the unit and the kind.**
  `_bytes` and `_seconds` suffixes are the
  convention; `_total` marks a counter. The classic
  gotcha: `node_memory_MemTotal_bytes` is in
  *bytes* — 8.32e9, i.e. about 8 GB — despite the
  kernel source it came from reporting KiB.
- **CPU time is a counter per mode.** Idle, user,
  system, iowait, steal and friends, per core.
  Chapter 5 turns these into "CPU busy %."
- **`load1`** is the kernel's 1-minute load
  average as a gauge — the same number `uptime`
  prints.
- **Filesystem sizes exist per mountpoint**, with
  labels for device and filesystem type — which is
  why disk queries always name a `mountpoint`.

## Collectors: the moving parts inside

node_exporter is itself a bundle of **collectors**,
one per kernel subsystem (cpu, meminfo, diskstats,
netdev, filesystem, …). It reports on them, too:

```
node_scrape_collector_success{collector="cpu"} 1
node_scrape_collector_success{collector="conntrack"} 0
```

A `0` means that collector found nothing to read
on this machine (no connection-tracking table to
report), not that the exporter is broken. You can
disable noisy collectors or enable extra ones with
`--collector.<name>` / `--no-collector.<name>`
flags — useful on odd hardware, rarely needed on
a normal server.

**Check:** `curl http://127.0.0.1:9100/metrics |
grep -c '^node_'` should return several hundred.
If it returns nothing, the exporter isn't running —
and that's the first thing Prometheus's targets
page would tell you, next chapter but one.
