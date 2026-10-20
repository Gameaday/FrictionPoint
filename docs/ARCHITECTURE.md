# FrictionPoint — Architecture & Phase 1 Specification

This document is the working technical contract for Phase 1 (Raspberry Pi / embedded Linux).
It is intentionally concrete: crate boundaries, traits, data sources, math, and budgets.

---

## 1. Phase 1 scope

**Goal:** On a Raspberry Pi, answer *"why did this box stop responding?"* with a defensible,
evidence-backed culprit, using < 1% CPU and no disk writes at idle.

### In scope

- Stall detection from PSI (`/proc/pressure/*`, per-cgroup `*.pressure`).
- Per-process and per-systemd-unit I/O request attribution.
- Block-device latency, queue depth, and request-size analysis (`/proc/diskstats`).
- Dirty-page / writeback throttling (`/proc/vmstat`, `/proc/sys/vm/*`).
- Small-write and fsync-storm detection (procfs + optional fanotify/eBPF).
- Pi-specific hardware checks: undervoltage/throttling (`vcgencmd get_throttled`), swap on
  SD, card/eMMC wear indicators, `mmc` errors, read-only remounts.
- Human-readable report + JSON output + `explain` + `doctor`.
- Record/replay so every detector is testable without root or a faulted machine.

### Explicitly out of scope for Phase 1

Windows, Android, macOS, GUI, remote fleet, auto-remediation, vulnerability scanning,
driver/firmware updates, LLM calls at runtime, long-term metric storage.

---

## 2. Data sources (Linux)

Everything below is read-only. Cost is per sample on a Pi-class device.

| Source | Path | Gives us | Privilege | Cost |
| --- | --- | --- | --- | --- |
| **PSI (system)** | `/proc/pressure/{cpu,io,memory}` | `some`/`full` stall share + cumulative µs | none | one small read |
| **PSI (cgroup)** | `/sys/fs/cgroup/<path>/*.pressure` | per-unit / per-container stalls | none (read) | O(units) reads |
| **cgroup I/O** | `/sys/fs/cgroup/<path>/io.stat` | per-unit bytes **and request counts** per device | none (read) | O(units) |
| **Per-process I/O** | `/proc/<pid>/io` | `rchar,wchar,syscr,syscw,read_bytes,write_bytes` | own processes; root for all | O(procs) |
| **Scheduler** | `/proc/<pid>/schedstat`, `/proc/<pid>/status` | run-queue wait, voluntary/involuntary switches | root for all | O(procs) |
| **Block device** | `/proc/diskstats` | IOPS, in-flight queue depth, service time | none | tiny |
| **Memory / writeback** | `/proc/vmstat`, `/proc/meminfo` | dirty pages, writeback, direct reclaim, major faults | none | tiny |
| **ext4 journal** | `/proc/fs/jbd2/<dev>-<n>/info` | commit count/rate, transaction size | root | small |
| **Thermal / power** | `/sys/class/thermal/thermal_zone*/temp`, `vcgencmd get_throttled` | throttling, undervoltage history | video group for `vcgencmd` | small |
| **Card identity** | `/sys/block/mmcblk*/device/{name,manfid,date,cid,life_time}` | card age, eMMC wear class | none | tiny |
| **Swap** | `/proc/swaps`, `/sys/block/zram*` | swap-on-SD, zram sizing | none | tiny |
| **Kernel log** | `journalctl -k` / `dmesg` (ring) | `mmc` timeouts, ext4 errors, OOM, USB resets | `adm`/root | event-driven |
| **fanotify** | `fanotify_init` + mount marks | per-file write/open events with PID | `CAP_SYS_ADMIN` | medium |
| **eBPF** | kprobes/tracepoints (`block_rq_issue`/`complete`, `sys_enter_fsync`) | **per-request latency attribution** | root + BTF | high, armed only |

### Traps we must handle explicitly

- **PSI can be absent.** `/proc/pressure` exists only if the kernel was built with
  `CONFIG_PSI` and, when built with `CONFIG_PSI_DEFAULT_DISABLED`, booted with `psi=1`.
  Probe for it, and if missing: degrade to procfs-derived proxies, tell the user in `doctor`,
  and never silently report "no stalls."
- **Containers see their own cgroup.** A container's `io.pressure` is genuinely idle while the
  host burns. Require a read-only host cgroup mount for container/fleet mode; without it,
  warn loudly.
- **`/proc/pressure` is not namespaced.** A container reads the *host's* numbers with no
  indication. Any container mode must label provenance.
- **A culprit that exits takes its cgroup with it.** Keep a short-lived tombstone of recently
  removed cgroups + PIDs so findings can still name the dead process.
- **`/proc/<pid>/io` does not count `fsync`.** `syscw` counts `write(2)`, not `fsync(2)` or
  `sync_file_range(2)`. fsync frequency requires fanotify (`FAN_FSYNC`-capable kernels) or an
  eBPF syscall tracepoint. Do not infer fsync rate from `syscw`.
- **`read_bytes`/`write_bytes` are bytes submitted to the block layer**, not completed bytes;
  they're a demand signal, not a latency measurement.
- **`%util` from `diskstats` is a lie on flash.** `time_io` measures busy time, and an SD card
  can be "30% utilized" while adding 400ms per request. Use in-flight queue depth plus
  latency distribution, never `%util`, in narratives.
- **SD cards expose no SMART.** eMMC has `life_time`; SD has manufacture date and error
  counters at best. State what is measurable and label the rest as a hypothesis.
- **eBPF needs BTF.** Many SBC kernels lack `CONFIG_DEBUG_INFO_BTF`; aya's CO-RE load will
  fail. Detect, fall back, and say so.
- **Clock discipline matters.** Use a monotonic clock for deltas; wall clock only for display.

---

## 3. Domain model

The vocabulary is deliberately small. If a concept doesn't fit here, it probably belongs in an
exporter or the Fix companion, not the core.

```text
raw kernel reads  ->  Probe  ->  Sample  ->  Series (ring buffer)
                                              |
                                         Detector (pure fn over a WindowView)
                                              |
                                           Finding
                                    (kind, severity, confidence,
                                     culprit, evidence, advice)
                                              |
                              Episode (correlated Findings)  ->  sinks
                                                              (stdout/TUI/JSONL/
                                                               Prometheus/OTLP)
```

```rust
// fp-core/src/model.rs  (illustrative, not final)

pub type Millis = u64;

#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub struct EntityId(u32);          // interner handle: pid, cgroup, device, mount

#[derive(Clone, Copy)]
pub enum Resource { Cpu, Memory, Io }

pub enum Signal {
    PsiSome(Resource),             // percent * 100, from avg10/60/300
    PsiFull(Resource),
    PsiFullTotalMicros(Resource),  // cumulative, for exact lost-time math
    Rios, Wios, Rbytes, Wbytes,    // per entity, per device
    MeanRequestBytes,              // derived: wbytes / wios
    QueueDepth,                    // in-flight from diskstats
    ServiceTimeMicros,             // completed requests
    RunQueueWaitMicros,
    InvoluntarySwitches,
    DirtyBytes, WritebackBytes, DirectReclaimStalls,
    ThermalMilliCelsius, ThrottleFlags,
}

pub struct Sample { pub at: MonoTime, pub entity: EntityId, pub signal: Signal, pub value: i64 }

pub enum Culprit {
    Process { pid: u32, comm: String, start_time: u64 },
    Unit    { name: String, cgroup: String },
    Device  { name: String, major_minor: (u32, u32) },
    Mount   { path: String, device: String },
    Driver  { name: String },
}

pub enum Confidence { Observed, Correlated, Traced }

pub struct Severity {
    pub stalled: Millis,           // wall-clock time lost in this window
    pub window: Millis,
}

pub struct EvidenceItem {
    pub source: &'static str,      // "psi", "diskstats", "cgroup.io_stat"
    pub path: String,              // exact path read, e.g. /sys/fs/cgroup/.../io.stat
    pub value: String,             // the raw text/values used
}

pub struct Finding {
    pub id: FindingId,             // stable hash, addressable via `explain`
    pub kind: FindingKind,
    pub window: (MonoTime, MonoTime),
    pub severity: Severity,
    pub confidence: Confidence,
    pub culprit: Option<Culprit>,
    pub evidence: Vec<EvidenceItem>,
    pub advice: &'static [AdviceId],  // stable ids -> recipe lookup

    // --- forward-compatible fleet fields (see docs/FLEET.md) ---
    pub role_hint: RoleHint,          // Origin | Victim | Bottleneck | Unknown
    pub impact: Impact,               // SelfStalled | RemoteWait | Degraded | Observed
    pub resource_keys: Vec<String>,   // "unit:traefik.service@pi-central"
    pub clock: ClockProvenance,       // mono window + wall anchor + skew bound
}
```

A finding with no credible culprit is still valuable: "the device is stalling with almost no
queue depth" is a hardware hypothesis, not a failure to diagnose. Never invent a culprit to
fill the field.

**Why those last four fields exist now.** They cost nothing (no dependency, no I/O, no network)
and they are expensive to add later because they require re-collecting history. `role_hint` and
`impact` are what let a future hub distinguish an origin from a victim; `resource_keys` is the
join key for cross-host causality; `clock` is what stops the hub from inventing causal ordering
out of NTP skew. The hub, pane, and federation are specified separately in `docs/FLEET.md`.

---

## 4. Crates and dependency direction

```text
crates/
  fp-core/        model, detectors, scoring, narrative, advice ids
                  -> zero I/O, #![deny(unsafe_code)], no platform deps
  fp-probe/       Probe trait, capability negotiation, registry, replay driver
  fp-store/       ring buffers, per-entity series, windowing, interning
  fp-linux/       PSI, procfs, cgroup v2, diskstats, mmc, vcgencmd, fanotify, eBPF(feature)
  fp-windows/     ETW, PDH, GetProcessIoCounters, USB topology
  fp-android/     storaged/dumpsys/procfs subset
  fp-daemon/      supervisor, config, episodes, sinks, signals
  fp-cli/         clap commands, ratatui live view, report/explain/doctor/record/replay
  fp-export/      JSONL, Prometheus, OTLP
  fp-mcp/         MCP server exposing findings to agents (later)
xtask/            verify pipeline, fixture tooling, perf gates
fixtures/         recorded .fpz scenarios + ground-truth manifests
docs/
```

Rules:

- `fp-core` never depends on a platform crate. Platform crates never depend on each other.
- Detectors live in `fp-core` and receive a `WindowView`; they cannot open files, read clocks,
  or spawn tasks. This is what makes them testable and agent-friendly.
- All platform-specific parsing lives behind `Probe`; a parser bug is a fuzz target, not a
  detector bug.

---

## 5. Core traits

```rust
// fp-probe/src/lib.rs
pub trait Probe: Send {
    fn id(&self) -> ProbeId;
    fn capabilities(&self) -> Capabilities;

    /// Cheap, non-blocking poll. Must not allocate after warmup.
    fn poll(&mut self, now: MonoTime, out: &mut SampleSink<'_>) -> Result<(), ProbeError>;

    /// Escalate to expensive sampling (fanotify marks, eBPF programs).
    fn arm(&mut self, _why: ArmReason) -> Result<(), ProbeError> { Ok(()) }
    fn disarm(&mut self) -> Result<(), ProbeError> { Ok(()) }
}

pub struct Capabilities {
    pub per_process_io: bool,
    pub per_cgroup_io: bool,
    pub psi: bool,
    pub per_request_latency: bool,   // eBPF/ETW only
    pub fsync_events: bool,          // fanotify/eBPF only
    pub thermal: bool,
    pub usb_topology: bool,
}
```

```rust
// fp-core/src/detect.rs
pub trait Detector {
    fn id(&self) -> &'static str;
    fn window(&self) -> Duration;
    fn evaluate(&mut self, w: &WindowView<'_>, out: &mut Findings);
}
```

Every capability that is `false` must surface in `frictionpoint doctor` and must downgrade the
corresponding finding's confidence rather than silently changing its meaning.

---

## 6. Two-tier sampling: "armed capture"

This is the single most important performance decision. Deep tracing is what makes attribution
credible, and it is also what makes monitoring tools unacceptable on a Pi. The resolution is
to make depth *event-triggered and time-boxed* — a camera trap, not a live stream.

```text
  LEVEL 0  idle          1 Hz   PSI, diskstats, per-pid io, vmstat, thermal
      |                          ~1 thread, no allocations, no disk writes
      |  PSI full delta > threshold  OR  anomaly on a cheap signal
      v
  LEVEL 1  focus        10 Hz   + cgroup io.stat per unit, per-pid io, wchan/D-state census,
      |                          + fanotify marks on the suspect mount (if permitted)
      |  culprit still ambiguous OR user passed --trace
      v
  LEVEL 2  trace        50 Hz   + eBPF block_rq_issue/complete + sys_enter_fsync,
      |                          per-request latency -> pid/comm/file     <= 10s
      v
  cooldown                      <= 60s of trace per hour, <= 1 escalation per 5 min
```

Consequences:

- Idle cost is independent of trace capability, so the tool is safe to leave running forever.
- Escalation is where the interesting data comes from, and it is bounded.
- `--trace` lets a user force Level 2 for a known-bad reproduction.
- Every finding records which level produced it, which is exactly the confidence tier.

---

## 7. Phase 1 detector catalogue

Each detector is a small, independently testable unit with a recorded fixture and a golden
output. This is also the unit of work for agent-driven development (one detector = one PR).

| ID | Detector | Signature (what makes it fire) | Culprit source | Remedy direction |
| --- | --- | --- | --- | --- |
| `io.full_stall` | I/O full stall | PSI `io full` delta over window exceeds floor (e.g. 200ms) and share > 2% | none (severity carrier) | context for other findings |
| `io.small_write_storm` | Small-write / fsync storm | High `wios` share + mean request < 16 KiB + elevated service time | cgroup `io.stat`, `/proc/<pid>/io` | buffer/batch writes, raise log level, move data dir, `commit=` |
| `io.writeback_throttle` | Dirty-page throttling | `nr_dirty` approaching `dirty_ratio`, writeback active, tasks in D-state on `balance_dirty_pages` | vmstat + wchan census | lower `dirty_ratio`, smooth bursty writers |
| `io.journal_serialization` | ext4 journal bottleneck | jbd2 commit rate at ceiling, jbd2 thread in D-state, many small metadata ops | `/proc/fs/jbd2/*/info` | `commit=`, `noatime`, fewer files/renames |
| `io.device_stall` | Device is slow (hardware) | High p95/p99 latency at **low** queue depth, uncorrelated with any single writer | diskstats + absence of blame | replace card, move to USB/NVMe, check power |
| `io.remount_readonly` | Rootfs went read-only | ext4 error -> remount-ro event | kernel log + mount flags | card failure; back up now, replace |
| `mem.reclaim_pressure` | Memory pressure / reclaim | PSI `memory full`, direct reclaim, major faults, swap-in | vmstat, per-pid RSS growth | right-size services, zram tuning, add RAM |
| `mem.swap_on_sd` | Swap is on the SD card | `/proc/swaps` on `mmcblk*` | `/proc/swaps` | disable/relocate swap, prefer zram |
| `cpu.runqueue_delay` | CPU starvation | `schedstat` run-queue wait high, PSI `cpu some` | per-pid schedstat | pin/renice, reduce oversubscription |
| `cpu.ctx_switch_storm` | Wakeup storm | involuntary switches/sec far above baseline | per-pid status | fix polling loops, increase intervals |
| `hw.thermal_throttle` | Thermal / undervoltage | `get_throttled` bits, temp above trip | vcgencmd + thermal zones | PSU/cable/cooling |
| `hw.mmc_errors` | Card/controller errors | `mmc` timeouts/resets in kernel log | kernel log counters | card or power; not a software fix |
| `fs.log_growth` | Runaway log growth | per-file write volume rate (fanotify, armed) | fanotify events | logrotate, rate-limit, remote ship |

### Distinguishing the four I/O stories

This table is the actual product. All four look identical in `htop`, and each has a different
fix. Getting this wrong is worse than saying nothing.

| Story | Latency | Queue depth | Throughput | Request size | Other signal | Fix |
| --- | --- | --- | --- | --- | --- | --- |
| **fsync / small-write storm** | high | moderate | low | tiny (< 16 KiB) | one dominant writer, high `wios` | batch, buffer, tmpfs, log level |
| **dirty-page throttling** | high | high | high | mixed | `nr_dirty` at limit, writeback saturated | tune `dirty_ratio`, smooth bursts |
| **journal serialization** | high | low | low | metadata | jbd2 D-state, commit ceiling | `commit=`, `noatime`, fewer renames |
| **card / controller stall** | high | **low** | low | any | no dominant culprit, `mmc` errors | replace hardware, fix power |

> "Queue depth is low but latency is high" is the fingerprint of a *device* problem.
> "Queue depth is high because we flooded it" is the fingerprint of a *workload* problem.
> Most tools cannot tell you which one you have. That distinction is the point.

---

## 8. Attribution math

All shares are computed over the same window `W` and the same device `D`, using deltas from
`io.stat` (per cgroup, per device).

```text
blame_share_i   = Δwios_i / Σ_j Δwios_j            # request share, not byte share
mean_request_i  = Δwbytes_i / Δwios_i
small_factor_i  = clamp(log2(64 KiB / mean_request_i), 0, 4)
friction_i      = blame_share_i * (1 + small_factor_i)
severity        = Δ(psi_io_full_total) / |W|        # fraction of wall clock lost
```

Worked example (the README incident):

| Unit | Δwios | share | mean request | small_factor | friction |
| --- | --- | --- | --- | --- | --- |
| `arcane-agent.service` | 570,000 | 0.74 | 4.1 KiB | 4.0 | **3.70** |
| `docker.service` | 150,000 | 0.19 | 120 KiB | 0.0 | 0.19 |
| everything else | 54,000 | 0.07 | mixed | — | ~0.10 |

Result: `arcane-agent.service` is the culprit with 74% request share and a 4x small-write
penalty. Note that by **bytes** the docker container may well have won — which is exactly how
byte-based tools blame the wrong process.

Guardrails:

- If no entity exceeds 40% share and the top two don't exceed 65%, report `io.device_stall` or
  an unattributed stall rather than picking a winner.
- Always publish the denominator (total requests observed) with the share.
- If the window contains a cgroup that disappeared, attribute to its tombstone and say so.
- Never mix devices: shares are per `major:minor`, then aggregated only for display.

---

## 9. Narrative engine

Narratives are deterministic templates, not generated text at runtime. Reasons: offline-first,
zero token cost, no data leaving the host, reproducible snapshots, and no hallucinated causes.

```rust
// fp-core/src/narrative.rs
pub struct Template {
    pub id: &'static str,          // "io.small_write_storm"
    pub headline: &'static str,    // "Storage stall on {device} - {stalled} of {window} frozen"
    pub why: &'static str,
    pub pattern: &'static str,
    pub try_this: &'static str,
}
```

Rules:

- Lead with the **measured** number (lost time), then the **hypothesis**, then the fix.
- Hedge language is mandatory on `correlated` findings: "accounts for", "during this window".
- Every narrative is backed by `EvidenceItem`s the user can re-read from the kernel.
- If a template's slots are missing, print the evidence and omit the story. Never interpolate
  a guess into a sentence that reads as fact.
- Golden-file tests (`insta`) cover every template at three severities.
- Optional later: `--explain-with-llm` posts only the finding JSON, opt-in, redacted. It is an
  enhancement, never a dependency.

---

## 10. Budgets and how they are enforced

| Metric | Idle (Level 0) | Armed (Level 1-2) |
| --- | --- | --- |
| CPU, Pi Zero 2 W | < 1% of one core | < 5% |
| RSS | < 15 MiB | < 25 MiB |
| Disk writes | **0 bytes** | findings only, < 1 MiB per incident |
| Timer wakeups | <= 5 / s | <= 60 / s |
| Binary (arm64, stripped, musl) | < 8 MiB | same |
| Cold start | < 50 ms | n/a |
| Allocations after warmup | **0** in the poll path | bounded |

Enforcement, not aspiration:

- A counting global allocator in integration tests asserts zero allocations per poll cycle.
- `xtask perf` runs a synthetic 10-minute trace and fails CI on budget regressions.
- A self-reported overhead section in `frictionpoint doctor` (CPU, RSS, wakeups, bytes written)
  so the tool can be audited by the user it is auditing.
- FrictionPoint must exclude itself from blame, and prove it: a test asserts that its own
  probe never appears as a culprit.
- Nightly job on real hardware (Pi 4 + Pi Zero 2 W) since emulation cannot measure I/O stalls
  or thermal behavior honestly.

---

## 11. Testing: record / replay is the foundation

Detectors that need a faulted kernel are untestable. So the primary test artifact is a
**recorded scenario**: raw probe output, not derived metrics.

```bash
frictionpoint record --for 120s --label pi_sd_fsync_storm --out fixtures/pi/sd-fsync.fpz
frictionpoint replay fixtures/pi/sd-fsync.fpz --json | jq .findings
```

A `.fpz` file contains: raw probe samples with monotonic timestamps, host metadata (kernel,
arch, cgroup layout, capabilities), the recorded kernel-log window, and an optional ground-truth
manifest. Recording *raw* data (not findings) means detectors can be rewritten and re-scored
against every historical scenario.

### Scenario corpus (ground truth)

```yaml
# fixtures/pi/sd-fsync.yaml
id: pi_sd_fsync_storm
host: rpi4-8gb / bookworm / 6.6
inject: fio --name=storm --rw=randwrite --bs=4k --fsync=1 --runtime=120 --size=256m
expect:
  kind: io.small_write_storm
  culprit: fio
  min_confidence: correlated
forbid:
  - io.device_stall        # must not blame the card for our own flood
```

Initial corpus (build these first, they are also the marketing):

1. `sd_fsync_storm` - 4 KiB fsync loop from one process.
2. `sd_gc_stall` - card-level latency spikes at low queue depth (real hardware, aged card).
3. `journal_commit` - many small metadata operations / renames.
4. `dirty_throttle` - large buffered writers exceeding `dirty_ratio`.
5. `swap_on_sd` - swap thrash on `mmcblk0`.
6. `memory_reclaim` - cgroup memory limit causing direct reclaim.
7. `cpu_undersubscribed` - run-queue delay without I/O involvement.
8. `thermal_throttle` - sustained load with `get_throttled` bits set.
9. `remount_ro` - injected ext4 error / dying card.
10. `clean_baseline` - healthy host; **must produce no findings** (false-positive gate).

### Test layers

| Layer | Tool | What it catches |
| --- | --- | --- |
| Parsers | `proptest` + `cargo-fuzz` | procfs/diskstats edge cases, truncated reads, counter rollover |
| Detectors | `insta` golden + replay corpus | wrong culprit, wrong kind, confidence inflation |
| Scoring | property tests | shares never exceed 1.0, no division by zero, monotonic severity |
| Ring buffer | `loom` + `miri` | concurrency and UB in the only unsafe code |
| Narratives | `insta` snapshots | misleading wording, missing evidence |
| Budgets | custom allocator + `xtask perf` | overhead regressions |
| Live | fault injection on real Pi | everything above being wrong in reality |

---

## 12. Using this project to evaluate agentic developers

This codebase is an unusually good benchmark for coding agents, because success is objective
and multi-dimensional:

- **Bounded, pure units of work.** A detector is a function over recorded data with a golden
  output. No ambiguous "make it nicer."
- **Real constraints.** Zero-allocation hot path, no `unsafe` in core, cross-platform `cfg`.
- **Gradable outcomes.** Culprit precision@1, finding-kind F1, budget pass/fail, clippy clean.
- **A corpus the agent cannot overfit to** (held-out scenarios + `clean_baseline`).
- **Genuine systems difficulty**: parsers, concurrency, kernel interfaces, eBPF, ETW.

Suggested harness (`xtask`):

```bash
cargo xtask task new detect-small-write-storm   # scaffolds fixture, failing test, ADR stub
cargo xtask task score detect-small-write-storm # runs held-out corpus, emits a scorecard
```

Scorecard dimensions: task success, held-out culprit precision@1, budget compliance, clippy
warnings, `unsafe` introduced, test quality (mutation score), diff size, wall time, tokens,
and human review notes. Keep tasks in tiers: parser fixes -> single detector -> cross-detector
correlation -> eBPF probe -> Windows ETW probe.

Guardrails that make agent output mergeable: `#![deny(unsafe_code)]` in `fp-core`, `clippy -D
warnings`, required fixture per detector, required `explain` evidence, and a CI perf gate.

---

## 13. CLI surface (v0.1)

```text
frictionpoint baseline [--for 60s] [--json]     # the first-run "what's wrong here?" report
frictionpoint watch [--tui]                      # live view; findings stream, not charts
frictionpoint report [--since 1h]                # episode summary from the daemon
frictionpoint explain <finding-id>               # raw evidence + the arithmetic behind a finding
frictionpoint doctor                             # what this host can and cannot see, and why
frictionpoint record --for 120s --label <name>   # capture a scenario fixture
frictionpoint replay <file.fpz> [--json]         # deterministic detector run over a fixture
frictionpoint export --format prometheus|otlp|jsonl
frictionpoint selftest                           # liveness probe (unlike doctor, a degraded agent is alive)
frictionpoint daemon                             # resident mode (foreground until M2)
frictionpoint collect                            # fleet aggregation; same binary, no DB (later)
```

Exit codes (scriptable, alert-friendly):

| Code | Meaning |
| --- | --- |
| 0 | No friction above threshold |
| 1 | Friction detected |
| 2 | Usage/config error |
| 3 | Insufficient visibility (missing PSI, permissions, unsupported kernel) — *not* "healthy" |

Code 3 is a first-class outcome. Conflating "I can't see" with "all good" is how monitoring
tools betray people.

---

## 14. Roadmap and acceptance criteria

### M0 - Walking skeleton (about a week)

Workspace, `fp-core` model, PSI probe, `io.full_stall` detector, text `report` against live
data, `doctor`. No TUI, no daemon, no eBPF.

**Done when:** `cargo run -- report --since 60s` produces a truthful stall summary on a Pi and
on a laptop, and exits 3 with an explanation when PSI is unavailable.

### M1 - Pi alpha (weeks 2-5)

Six detectors (`io.full_stall`, `io.small_write_storm`, `io.device_stall`, `mem.swap_on_sd`,
`hw.thermal_throttle`, `io.remount_readonly`), record/replay, five fixtures, `explain`,
perf gate, static `aarch64-unknown-linux-musl` release, systemd unit.

**Done when:**

- `clean_baseline` produces zero findings; the nine seeded scenarios name the right culprit in
  at least seven, with correct confidence tiers.
- Idle budgets met on a Pi Zero 2 W, measured by the tool's own `doctor`.
- A new user can install, run `baseline`, and read the output without reading documentation.

### M2 - Pi beta (weeks 6-10)

Blame scoring, episodes/cascades, TUI live view, JSONL/Prometheus export, alert hooks
(`systemd-cat`, webhook), Pi hardware checks (card age, `mmc` errors, undervoltage history),
fanotify file-level attribution, held-out benchmark corpus.

### M3 - Servers and containers

Read-only `/sys/fs/cgroup` DaemonSet, per-container attribution, OTLP, cascade detection across
co-tenanted services. This is where the original log-flood incident becomes a first-class
finding (`cascade.shared_storage`).

### M4 - Windows alpha (the streamer wedge)

ETW: per-process file I/O latency, DPC/ISR time per driver, `Microsoft-Windows-USB-*` resets,
`Kernel-Power` sleep/wake transitions. PDH fallback. No parity claims; capability table is
honest.

### M5 - Android advisor mode

`dumpsys storaged`, `/proc/uid_io/stats` where readable, `dumpsys diskstats`, memory pressure
events. Unrooted = advisory only. Rooted/Perfetto = deeper tier.

### M6 - Fix companion + recipe registry

Separate binary, dry-run by default, signed recipe index, explicit consent for every change.

### M7 - Fleet phases, then MCP

Fleet is sequenced in `docs/FLEET.md` §10 and gated on real multi-host users: v0.2 one-way
MQTT/webhook export, v0.3 `hub` + read-only `/v1` API + `fleet` TUI, v0.4 causal reasoner
(roles, edges, blast radius), v0.5 web pane, v0.6 hub-to-hub federation. An MCP server exposing
findings and episodes comes after the API stabilizes, so agents query structure rather than
scraping output.

---

## 15. Workspace layout

```text
Cargo.toml                 # [workspace] resolver = "2"
crates/fp-core/
  src/model.rs
  src/detect/{io,mem,cpu,hw}.rs
  src/score.rs
  src/narrative.rs
crates/fp-probe/src/{lib.rs,replay.rs,capabilities.rs}
crates/fp-store/src/{ring.rs,series.rs,intern.rs}
crates/fp-linux/src/{psi.rs,cgroup.rs,procfs.rs,diskstats.rs,mmc.rs,log.rs,fanotify.rs,ebpf.rs}
crates/fp-cli/src/{main.rs,cmd/*,tui/*}
crates/fp-daemon/src/{main.rs,supervisor.rs,episode.rs,sinks/*}
crates/fp-export/src/{prometheus.rs,otlp.rs,jsonl.rs}
fixtures/{pi,server,clean}/*.fpz + *.yaml
xtask/src/main.rs
docs/{VISION,ARCHITECTURE,PRIOR-ART,LAUNCH}.md
docs/adr/0001-*.md
```

Workspace hygiene from day one: `resolver = "2"`, shared `[workspace.lints]`, `rust-version`
pinned, `cargo-deny` for licenses/advisories, `cargo-nextest` in CI, `cargo fmt --check` and
`clippy -D warnings` as required gates.

---

## 16. Open decisions (need your call)

1. **Distribution:** *decided* - one static binary is canonical; systemd on the host is the
   full-fidelity mode; containers are a self-declaring degraded mode. See `docs/DEPLOYMENT.md`
   and `docs/adr/0001-single-binary-capability-negotiation.md`. Note: a Home Assistant **add-on**
   cannot see the host cgroup tree reliably, so the HA path is a host install plus MQTT
   integration, not an add-on wrapper.
2. **Daemon or not:** ship a resident daemon in v0.1, or make `watch`/`report` foreground tools
   until the model stabilizes? Suggest foreground first; daemon once detectors are trusted.
3. **eBPF in v0.1:** keep it strictly opt-in behind a feature flag, or omit until M2?
   Suggest: omit from the binary until M2; ship fanotify (simpler, no BTF) as the first
   file-level source.
4. **TUI at launch:** a live view sells the project in GIFs; it also doubles the UI surface.
   Suggest a minimal `watch` table (ratatui) at M2, not M1.
5. **MSRV:** latest stable only, or N-2 for distro packaging?
6. **Alerting defaults:** what should wake a user up? Suggest: nothing by default; opt-in
   webhook/systemd-cat with a severity floor.
7. **Name checks:** crates.io, npm, domain, and a quick trademark scan before announcing.
