# Prior Art & Competitive Validation

**Question:** is "a low-overhead, cross-platform watcher that names the process causing
micro-stalls and I/O write locks" already an open-source solution?

**Answer: the primitives are a commodity; the synthesis is not.**

Everything FrictionPoint needs to *read* already exists, is well documented, and is mostly
free. What does not exist is a maintained product that (1) detects a stall window,
(2) attributes it to a process/unit/device/file, (3) explains it in plain language with
evidence and an honest confidence level, and (4) is light enough to leave running on a 1 GB
SBC. That gap is the project.

---

## 1. The closest thing that exists

### `varbees/stallwatch` — archived October 2026, 2 stars, Rust

> "Names the systemd unit, and the process inside it, stalling your Linux system, via
> per-cgroup PSI."

This is the single most relevant prior art, and it is genuinely good work: 46 commits, v0.2.2
on crates.io, static musl binaries, a Varlink daemon, a `doctor` command, and a carefully
written limitations section. It validates the core mechanism (PSI + cgroup v2 attribution is
cheap and effective on Linux).

It is **not** the product:

| Dimension | stallwatch | FrictionPoint |
| --- | --- | --- |
| Scope | Desktop freeze attribution, Linux only | Storage/IO **device vs workload** diagnosis, multi-platform |
| Attribution | systemd unit / cgroup, unit-level | unit + process + device + file, with confidence tiers |
| Root-cause classes | Pressure exists, name the unit | Distinguishes fsync storm vs dirty throttling vs journal vs dying card |
| Hardware | Not a concern | Card wear, undervoltage, `mmc` errors, thermal — first-class |
| Cross-platform | No | Linux -> servers -> Windows -> Android (sequenced) |
| Extensibility | Closed with the archive | Detector catalogue + community recipes |
| Distribution | Ended | Intended to be installed by non-experts |
| Remediation | "Prints the command you might run" | Separate consent-gated companion + recipe registry |

Its own limitations section is a gift: it documents traps we must design around —
`/proc/pressure` is not namespaced, containers see their own (idle) cgroup, per-cgroup PSI
triggers require privilege, a culprit that exits takes its cgroup with it, and "frozen N ms"
is bounded by the sampling window. Those are now explicit requirements in
`docs/ARCHITECTURE.md` §2.

**A 2-star archive is not a competitor. It is a proof of concept that nobody productized.**

---

## 2. Landscape by category

### Stall / pressure readers (Linux)

| Project | What it does | Where it stops |
| --- | --- | --- |
| `cloudflare/psi_exporter` | PSI -> Prometheus | Metrics only; no attribution, no narrative |
| `holmanb/psimon` | Interactive PSI monitor | Visualizes pressure; doesn't name a culprit |
| `joedefen/psistat` | Prints PSI values + exceptions | Threshold display |
| `samber/go-psi`, `chise0713/presutaoru` | PSI wrappers/libraries | Building blocks, not products |

These prove the signal is easy to obtain and cheap to read. None answer "who did it, and what
should I change?"

### Kernel tracing toolkits

| Tool | Strength | Gap for our user |
| --- | --- | --- |
| `bcc` / `bpftrace` tools (`biolatency`, `biosnoop`, `filetop`, `ext4slower`, `iosnoop`) | Excellent per-request truth | Expert-only, ad-hoc, root + headers, no correlation, no persistence, no advice |
| `fatrace` (279★, GPL-3.0, C) | Per-file access events from all processes | Events, not latency or impact; root; no ranking; noisy on a busy box |
| `iotop`, `atop`, `pidstat`, `iostat`, `vmstat` | Ubiquitous accounting | Bytes and utilization, not blocked time or causality |
| `perf` / `ftrace` | Anything is possible | Steep learning curve, no opinion about what matters |

FrictionPoint should *use* these sources (and optionally shell out to none of them), then do
the part they deliberately don't: decide what matters and explain it.

### Full monitoring platforms

| Product | Strength | Gap |
| --- | --- | --- |
| **Netdata** | Per-second metrics, per-process `apps.plugin`, ML anomaly bits, alerting, strong UI | Metric-first; no per-request latency; no causal culprit narrative; footprint and operational shape are heavier than a Pi Zero budget |
| Prometheus + Grafana + node_exporter | Industry standard | You must already know what to ask; no root cause |
| Zabbix / Checkmk / Nagios | Fleet alerting | Thresholds, not diagnosis |

Netdata is the most credible potential competitor: it already has the collectors and the ML
infrastructure. Its product identity is dashboards and fleet observability, though, and its
answers are "this chart is anomalous," not "arcane-agent's 4 KiB writes froze your box for
12.4 seconds; move its data directory." It is also a real risk to watch.

### Windows

| Tool | Strength | Gap |
| --- | --- | --- |
| **Intel PresentMon** (2.6k*, MIT) | Best-in-class frame/present timing, ETW-based | Graphics/frame focus; not general system I/O or storage attribution |
| **ETWAnalyzer** (144*, MIT, Siemens) | Powerful offline queries over ETL traces, stacktags, module versions | Offline, expert CLI, Windows-only; you must know what to record and what to ask |
| **LatencyMon** (closed) | DPC/ISR latency per driver - the "why does audio crackle" tool | Windows-only, no storage/peripheral attribution, no plain-language remediation |
| WPR / WPA / xperf | Complete truth | Expert tooling; hours of learning |
| Task Manager / Resource Monitor | Ubiquitous | Aggregates; the exact thing this project replaces |
| SRUM | Per-app resource history | 1-hour buckets; forensic, not diagnostic |

Windows is *underserved in exactly the way the user describes*: the data exists in ETW, but
turning it into "your webcam reset because the bus was oversubscribed" is a manual, expert
exercise. PresentMon is not a competitor; it is a neighbouring tool with a different question.

### Android

| Tool | Strength | Gap |
| --- | --- | --- |
| **Perfetto** | Superb system tracing (ftrace/atrace), timeline UI | Expert, offline, large traces, not an everyday diagnosis tool |
| `storaged` / `/proc/uid_io/stats` | Per-UID read/write bytes and **fsync counts**, maintained by the platform | Raw platform data, no user-facing explanation, no latency view |
| `dumpsys procstats/diskstats/meminfo` | Per-app history and memory state | Vague, version-dependent, unaggregated |
| Vendor "device care" UIs | Storage/memory cleanup | Not diagnostic |

Android is the most underserved of all, and the most constrained: unrooted access is limited
enough that the honest product is an **advisor**, not a tracer.

### Adjacent (deliberately not competitors)

`systemd-analyze blame` (boot only), `powermetrics`/`fs_usage` (macOS, expert), Steam's
performance overlay (games), `smartctl` (drive health, not workload), `btrfs`/ZFS tooling
(filesystem-specific).

---

## 3. What is actually missing

1. **Stall-centric presentation.** The industry reports utilization; almost nobody leads with
   lost time.
2. **Device-vs-workload discrimination.** "Queue depth low + latency high = the card is sick"
   versus "queue depth high because we flooded it" is the decision users actually need, and
   it is absent from consumer tooling.
3. **Attribution with honest confidence.** Tools either guess silently or dump raw traces.
   Three explicit tiers (`observed`/`correlated`/`traced`) are new.
4. **Request-based blame.** Every byte-based tool mis-blames fsync-heavy workloads.
5. **Cross-platform continuity.** A homelab is heterogeneous. Nothing speaks the same
   vocabulary on a Pi, a server, and a Windows desktop.
6. **Reproducible scenario corpus.** Nobody publishes recorded, labelled incident fixtures
   that detectors can be scored against. This is both a testing strategy and a credibility
   asset.
7. **Remediation separation.** Diagnosis tools don't advise; "optimizer" tools guess and
   break things. The split (core explains, recipes advise, companion acts) is unoccupied.

---

## 4. Why now

- **PSI is mainstream.** In-kernel since 4.20; used in production by `systemd-oomd` and
  Android's LMKD. The cheap, correct stall metric already ships on the machines we target.
- **cgroup v2 is the default** on modern distros, giving per-unit attribution for free.
- **Rust's eBPF story matured** (`aya`, 4.9k*): pure-Rust, CO-RE, no libbpf, single static
  binary. Deep tracing no longer requires a C toolchain or distro-specific build. (Not used in
  v1.0; it is the planned upgrade path for the optional `trace` feature.)
- **SBCs are everywhere** and their storage is the bottleneck. The Pi is the perfect forcing
  function for a low-overhead design.
- **Streaming/peripheral complexity went mainstream**, and nobody explains USB bus failures to
  normal people.
- **Agents can now build this.** The work is decomposable into pure, testable detectors — which
  is precisely the experiment the user wants to run.

---

## 5. Honest risk assessment

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| Netdata or a vendor ships "culprit attribution" | Medium | Win on narrative quality, device-vs-workload reasoning, SBC footprint, and openness; stay export-friendly rather than competing on dashboards |
| Scope sprawl across platforms kills it | **High** | Enforced sequence; capability negotiation; one platform at a time |
| Correlation is wrong often enough to lose trust | Medium | Confidence tiers, evidence, conservative thresholds, `clean_baseline` false-positive gate |
| Kernel variance (no PSI, cgroup v1, no BTF) | High | `doctor`, fallbacks, exit code 3, never silently claim health |
| Nobody installs it until they have an incident | Medium | Baseline report must be interesting on a healthy box; scenario corpus as marketing |
| eBPF/ETW maintenance burden | High | Optional feature flag; fanotify first; community probes behind traits |
| "Yet another monitoring tool" fatigue | Medium | Position as a *diagnoser*, not a monitor. No dashboards. |

---

## 6. One-line positioning

> **Task Manager shows utilization. FrictionPoint shows unavailability — the seconds your
> system spent frozen, and who caused them.**

That sentence is not claimed by any project in this landscape.
