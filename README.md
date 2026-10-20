# FrictionPoint

**Task Manager shows utilization. FrictionPoint shows unavailability: the seconds your
system spent frozen, and who caused them.**

A tiny Rust diagnostic engine that correlates PSI stalls, disk latency, fsync storms, memory
pressure, and peripheral dropouts into plain-English root causes — naming the process, unit,
and file responsible. Linux, Windows, and SBCs. Local-first, low overhead.

> **Status:** pre-alpha / design phase. Phase 1 targets Raspberry Pi & embedded Linux storage
> stalls. Nothing here is stable yet, including the architecture.

---

## The problem: your meters are lying by omission

A Raspberry Pi can report **15% CPU and 40% free RAM** while being completely unusable. `htop`
shows idle. SSH hangs. Docker health checks fail. Nothing looks guilty, because *nothing is
busy* — everything is **waiting**.

Utilization is a leading indicator. **Stall time is the metric the user actually experiences.**
Traditional monitors count how hard the engine is revving; they never count the seconds the
wheels weren't turning.

That gap is where the worst outages live:

- A single Pi starts shipping logs to a central Pi that also runs Traefik. Log volume spikes,
  the SD card saturates, and *the whole network's ingress disappears*. Nothing is "down."
  Everything is just waiting.
- A Pi's SD card hits a garbage-collection stall, so every process touching the rootfs blocks
  for 400ms at a time. `iostat` says the disk is 30% utilized.
- A Steam client fails to resume downloads after sleep. Network errors, but zero network
  problem.
- A streamer's USB webcam needs *a hair* more bandwidth than the shared hub can give, and
  every peripheral on the bus resets at once.

In all four cases the standard toolset answers the wrong question. FrictionPoint asks:
**what was blocked, for how long, and who made it block?**

---

## What it does

1. **Detects stall windows** — not averages. "This host was dead in the water for a cumulative
   12.4s over the last 5 minutes," sourced from kernel pressure/stall accounting, not
   sampling guesswork.
2. **Attributes the friction** — which process, which systemd unit / cgroup, which device,
   which file. With an explicit confidence tier so it never pretends to know more than it does.
3. **Explains it in human terms** — a short narrative plus the exact kernel paths and numbers
   behind every claim, so you can verify the arithmetic yourself.

```text
[FRICTION] Storage stall on /dev/mmcblk0p2 — 12.4s of 5m frozen (4.1% wall clock)

  WHY        io.pressure "full" climbed 0.8% -> 61% at 03:14:02Z and stayed elevated.
             41 tasks sat in uninterruptible sleep (D-state) at peak.
  CULPRIT    arcane-agent.service (PID 412, cgroup .../arcane-agent.service)
             74% of write *requests* this window, mean request 4.1 KiB.
  PATTERN    Small-write / fsync storm against a single rotating-ish device.
             Not throughput-bound: queue depth 1.3, 0.4 MiB/s, 1,900 IOPS.
  EVIDENCE   /sys/fs/cgroup/system.slice/arcane-agent.service/io.stat
             /proc/pressure/io  /proc/diskstats  /proc/412/io
  TRY        Batch or buffer the agent's log writes; move its data dir off the SD card;
             raise ext4 commit interval. See `frictionpoint explain <id>`.
  NOTE       Confidence: correlated. Re-run with `--trace` for per-request proof.
```

---

## Principles

- **Stall-centric, not utilization-centric.** Every headline number is time the system was
  unavailable, not how busy it was.
- **Explain, don't chart.** You already have dashboards. FrictionPoint's job is the sentence
  you say out loud after reading it.
- **Show your work.** Every finding carries its evidence: raw values and the kernel paths they
  came from. No black boxes, no opaque "health score."
- **Never lie about certainty.** Findings are tagged `observed`, `correlated`, or `traced`.
  Correlation is presented as correlation.
- **Cheap by default, expensive only when it matters.** ~1 Hz polling at idle; deep tracing
  arms itself only during a detected stall, then disarms. The observer must never become the
  bottleneck it's hunting.
- **Local-first and quiet.** No telemetry, no cloud, no writes to disk unless a finding is
  worth persisting.
- **Diagnosis is separate from remediation.** The core only explains. Fixing things is a
  different binary with different privileges, always dry-run first.

---

## Platforms

| Platform | Phase | Notes |
| --- | --- | --- |
| Linux / Raspberry Pi / SBCs | **1 (now)** | PSI, cgroup v2, procfs, optional eBPF. Primary target. |
| Linux servers & containers | 2 | Headless agent, read-only cgroup mount, Prometheus/OTLP export. |
| Windows | 3 | ETW: DPC/ISR latency, disk, USB bus resets, sleep/wake re-init. Streamer wedge. |
| Android | 4 | `storaged`/`dumpsys` per-UID I/O (unrooted "advisor mode"); Perfetto for deep mode. |
| macOS / others | Later | Only with honest, limited capability reporting. |

We deliberately ship **one platform well** before adding a second. Cross-platform parity is
the graveyard this kind of project usually dies in — so capabilities are negotiated at runtime
and reported honestly rather than faked.

### Deployment

One static binary, three supported shapes:

| Shape | Fidelity | Use |
| --- | --- | --- |
| **Host + systemd** | Full (traced-capable) | **Recommended.** Raspberry Pi, single boxes, homelab nodes |
| **Container (host-visibility recipe)** | Partial (correlated) | Servers, Kubernetes DaemonSet |
| **Container (defaults)** | Severity only — *announced as degraded* | Try-before-install, CI |

Host-first is a **correctness** choice, not a convenience one: a container with a private cgroup
namespace sees only its own subtree and would report "healthy" while the host is in full I/O
stall. The tool always states what it can and cannot see, and exits `3` rather than claiming
health when it is blind. See [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md).

Agents are **push-only and never listen**; findings are the wire format (MQTT / OTLP / webhook),
so fleet mode arrives later without changing the core.

---

## Documentation

- [`docs/VISION.md`](docs/VISION.md) — why this exists, who it's for, what it refuses to be.
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — data model, crates, detectors, perf budget,
  Phase 1 spec, testing strategy.
- [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) — host vs container vs Kubernetes, capability matrix,
  fleet topology, privileges, upgrade and identity rules.
- [`docs/PRIOR-ART.md`](docs/PRIOR-ART.md) — the honest competitive landscape and why this is
  still worth building.
- [`docs/LAUNCH.md`](docs/LAUNCH.md) — GitHub About text, topics, first issues, launch post.
- [`docs/adr/`](docs/adr/) — decision records for the choices that are expensive to reverse.

---

## License

Intended to be dual-licensed under MIT or Apache-2.0, at your option.
The license files land with the first code commit.
