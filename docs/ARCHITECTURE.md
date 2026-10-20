# FrictionPoint — Architecture & v1.0 Specification
> **Scope and schedule are authoritative in [`PLAN.md`](PLAN.md); the audit of prior
> assumptions is in [`AUDIT.md`](AUDIT.md).** This document specifies mechanisms and data
> sources. Where they disagree, PLAN wins.


This document is the working technical contract for v1.0 (Linux / Raspberry Pi / embedded).
It is intentionally concrete: crate boundaries, traits, data sources, math, and budgets.

---

## 1. v1.0 scope

**Goal:** On a Raspberry Pi, answer *"why did this box stop responding?"* with a defensible,
evidence-backed culprit, using < 1% CPU and no disk writes at idle.

### In scope

- Stall detection from PSI (`/proc/pressure/*`, per-cgroup `*.pressure`).
- Per-process and per-systemd-unit I/O request attribution.
- Block-device latency, queue depth, and request-size analysis (`/proc/diskstats`).
- Dirty-page / writeback throttling (`/proc/vmstat`, `/proc/sys/vm/*`).
- Small-write and fsync-storm detection (procfs inference; true fsync counts need eBPF, post-1.0).
- Pi-specific hardware checks: undervoltage/throttling (`vcgencmd get_throttled`), swap on
  SD, card/eMMC wear indicators, `mmc` errors, read-only remounts.
- Human-readable report + JSON output + `explain` + `doctor`.
- Record/replay so every detector is testable without root or a faulted machine.

### Explicitly out of scope for v1.0

Windows, Android, macOS, GUI, remote fleet, auto-remediation, vulnerability scanning,
driver/firmware updates, LLM calls at runtime, long-term metric storage.

---

## 2. Data sources (Linux)

Everything below is read-only. Cost is per sample on a Pi-class device.

| Source | Path | Gives us | Privilege | Cost |
| --- | --- | --- | --- | --- |
| **PSI (system)** | `/proc/pressure/{cpu,io,memory}` (+ `irq` where the kernel provides it) | `some`/`full` stall share + cumulative µs; `irq` catches interrupt storms that PSI cpu/io hide | none | one small read each (probe `irq`) |
| **PSI (cgroup)** | `/sys/fs/cgroup/<path>/*.pressure` | per-unit / per-container stalls | none (read) | O(units) reads |
| **cgroup I/O** | `/sys/fs/cgroup/<path>/io.stat` | per-unit bytes **and request counts** per device | none (read) | O(units) |
| **Per-process I/O** | `/proc/<pid>/io` | `rchar,wchar,syscr,syscw,read_bytes,write_bytes` | own processes; root for all | O(procs) |
| **Per-task block wait** | `/proc/<pid>/stat` field 42 (`delayacct_blkio_ticks`); `/proc/<pid>/task/*/stat` | cumulative block-I/O wait per task (10 ms ticks), the closest cheap proxy for per-process blocked time | own; root for all | O(procs) main thread; O(tasks) **armed only** |
| **Scheduler** | `/proc/<pid>/schedstat`, `/proc/<pid>/status` | run-queue wait, voluntary/involuntary switches | root for all | O(procs) |
| **Block device** | `/proc/diskstats` | IOPS, in-flight queue depth, service time | none | tiny |
| **Memory / writeback** | `/proc/vmstat`, `/proc/meminfo` | dirty pages, writeback, direct reclaim, major faults | none | tiny |
| **ext4 journal** | `/proc/fs/jbd2/<dev>-<n>/info` | commit count/rate, transaction size | root | small |
| **Thermal / power** | `/sys/class/thermal/thermal_zone*/temp`, `vcgencmd get_throttled` | throttling, undervoltage history | video group for `vcgencmd` | small |
| **Card identity** | `/sys/block/mmcblk*/device/{name,manfid,date,cid,life_time}` | card age, eMMC wear class | none | tiny |
| **Swap** | `/proc/swaps`, `/sys/block/zram*` | swap-on-SD, zram sizing | none | tiny |
| **Kernel log** | `journalctl -k` / `dmesg` (ring) | `mmc` timeouts, ext4 errors, OOM, USB resets | `adm`/root | event-driven |
| **fanotify** | `fanotify_init` + mount marks | per-file write/open events with PID | `CAP_SYS_ADMIN` | medium |
| **eBPF** (v1.x, optional) | kprobes/tracepoints (`block_rq_issue`/`complete`, `sys_enter_fsync`) | **per-request latency attribution** | root + BTF | high, armed only |

### Rejected or conditional sources

| Source | Verdict | Reason |
| --- | --- | --- |
| **`inotify`** | **Rejected** | Reports *that* a file changed, not **which process** changed it. Per-inode watches, `max_user_watches` limits, no mount-wide scope. `fanotify` is the correct API (mount marks + PID/FID). |
| **`perf_event`** | **Spike only** | Can count context switches per thread group in one fd (`inherit=1`), but costs an fd per tracked process and depends on `perf_event_paranoid` + `CAP_PERFMON`. Try it only if the CPU budget is missed. |
| **`ptrace` / `LD_PRELOAD`** | **Rejected** | Serializes or injects into the target; breaks performance and security guarantees. |
| **PresentMon** | **Not a source** | Frame/present timing, not system I/O or power transitions. |
| **`delayacct_blkio_ticks`** | **Conditional** | Requires `CONFIG_TASK_DELAY_ACCT` (and not `nodelayacct`); 10 ms granularity; per-task, so a thread group needs `/proc/<pid>/task/*/stat` — use it as an armed cross-check, not a per-tick source. |

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
  `sync_file_range(2)`. **fanotify does not report fsync either** (no `FAN_FSYNC`); it reports
  open/read/write/close/create/move/error. Actual fsync counting needs an eBPF syscall tracepoint,
  so v1.0 *infers* fsync storms from write-syscall rate + small mean request size + elevated
  latency, and labels them `correlated`. Do not infer fsync rate from `syscw`.
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
- **A sample interval is never a duration.** `frozen_ms` derives from cumulative PSI totals over the
  report window, and is a **lower bound** whenever coverage < 100%. The prior art shipped a 400 ms
  capture window and reported it as freeze length for ~520,000 incidents; only 3 appeared to exceed
  1 s. This is a correctness rule, not a formatting detail.

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

Authoritative crate map, dependency budget, and LOC budgets: [`PLAN.md`](PLAN.md) §2-3.
Summary:

| Crate | Role | `unsafe` |
| --- | --- | --- |
| `fp-core` | pure model, detectors, scoring, narrative | forbid |
| `fp-linux` | parsers + probes (9 files) | forbid |
| `fp-agent` | tick loop, scheduling, coverage, sinks | forbid |
| `fp-cli` | binary: report/explain/doctor/record/replay/watch | forbid |
| `fp-trace` | optional fanotify/eBPF | **only quarantine** |
| `fp-hub` | optional fleet server (post-1.0) | forbid |
| `fp-windows` | optional ETW backend (post-1.0) | contained |

Rules:

- `fp-core` never depends on a platform crate, and opens no files. Platform crates never depend on
  each other.
- Detectors live in `fp-core` and receive a `WindowView`; they cannot read clocks, open files, or
  spawn tasks. This is what makes them testable and agent-friendly.
- All platform parsing lives behind `Probe`; a parser bug is a fuzz target, not a detector bug.
- The default agent binary links no networking code (ADR-0004).

---

## 5. Core traits

```rust
// fp-core/src/probe.rs  (trait; implemented by fp-linux, replayed by fp-agent)
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
    pub fsync_events: bool,          // eBPF tracepoint only; fanotify cannot see fsync
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
- `--trace` lets a user force Level 2 for a known-bad reproduction (post-1.0; **v1.0 caps at
  Level 1**, so all v1.0 findings are `observed` or `correlated`).
- Every finding records which level produced it, which is exactly the confidence tier.

---

## 7. Detector catalogue

**v1.0 ships areas A1 + A2** — the ten rows marked `v1.0` below, including the residual and one
advisory check. Everything else is phase-marked backlog, not a commitment. Area definitions and exit
criteria live in [`AREAS.md`](AREAS.md).

Each detector is a small, independently testable unit with a recorded fixture, a golden output, and a
negative test. This is also the unit of work for agent-driven development (one detector = one PR).

| ID | Detector | Signature (what makes it fire) | Culprit source | Phase |
| --- | --- | --- | --- | --- |
| `io.full_stall` | I/O full stall | PSI `io full` delta above floor; the severity carrier | none (system-wide) | **v1.0** |
| `io.small_write_storm` | Small-write / fsync storm | high request share + mean request < 16 KiB + elevated service time | cgroup `io.stat`, per-pid `io` | **v1.0** |
| `io.device_stall` | Device is slow (hardware) | high p95/p99 at **low** queue depth, no dominant writer | `diskstats` + absence of blame | **v1.0** |
| `unknown_friction` | Unattributed stall | severity above floor, no detector attributes it, reason attached | none | **v1.0** |
| `hw.undervoltage` | Undervoltage / throttle | `get_throttled` bits, now and since boot | `vcgencmd` | **v1.0** |
| `hw.thermal_throttle` | Thermal throttle | temperature above trip or frequency capped | thermal zones, `vcgencmd` | **v1.0** |
| `hw.mmc_errors` | Card/controller errors | `mmc` timeouts, CRC errors, resets | kernel log | **v1.0** |
| `io.remount_readonly` | Rootfs went read-only | ext4/xfs error -> remount-ro | kernel log + mount flags | **v1.0** |
| `mem.swap_on_sd` | Swap on the SD card | `/proc/swaps` entry on `mmcblk*` | `/proc/swaps` | **v1.0** |
| `hw.card_wear` | Card wear (advisory) | eMMC `life_time` class or SD manufacture date | sysfs | **v1.0** (advisory) |
| `io.writeback_throttle` | Dirty-page throttling | `nr_dirty` near `dirty_ratio`, tasks in `balance_dirty_pages` | vmstat + `wchan` | v1.1 |
| `io.journal_serialization` | ext4 journal bottleneck | commit rate at ceiling, jbd2 in D-state | `/proc/fs/jbd2/*/info` | v1.1 |
| `fs.mount_misconfig` | Costly mount options | `atime`, `commit=`, sync mounts on flash | `/proc/mounts` | v1.1 |
| `fs.capacity` | Full filesystem / inodes | usage above threshold, inode exhaustion, ENOSPC errors | `statvfs`, kernel log | v1.1 |
| `mem.reclaim_pressure` | Memory pressure | PSI `memory full`, direct reclaim, major faults | vmstat, PSI | v1.2 |
| `mem.cgroup_limit` | Cgroup memory ceiling | `memory.events` high/max above zero | cgroup files | v1.2 |
| `mem.oom_event` | OOM kill | OOM event with victim | kernel log / cgroup | v1.2 |
| `cpu.runqueue_delay` | CPU starvation | `schedstat` run-queue wait, PSI `cpu some` | per-task schedstat | v1.3 |
| `cpu.ctx_switch_storm` | Wakeup storm | involuntary switches/sec far above baseline | `/proc/<pid>/status` | v1.3 |
| `cpu.cgroup_throttle` | CFS throttling | `nr_throttled` / `throttled_usec` rising | `cpu.stat` | v1.3 |
| `cpu.steal` | Stolen CPU (VM) | steal share above baseline | `/proc/stat` | v1.3 |
| `dep.remote_wait` | Waiting on a dependency | D-state tasks in `rpc_wait_*`, `nfs_*`, socket waits | `wchan` census | v1.4 |
| `io.fuse_stall` | FUSE stall | FUSE `waiting` growing, task D-state | fuse sysfs | v1.4 |
| `net.queue_drop` | Local queue drops | `softnet_stat` drops, `time_squeeze` | `/proc/net/softnet_stat` | v1.4 |
| `net.retransmit` | Retransmit rate (advisory) | retransmit share above baseline | `/proc/net/snmp` | v1.4 |
| `fs.log_growth` | Runaway log growth | per-file write rate (armed) | fanotify | post-1.0 |
| `irq.storm` | IRQ/softirq storm | `pressure/irq` high while cpu pressure flat | where present | post-1.0 |
| `usb.reservation_exhausted` | USB bus oversubscribed | derived reservation ratio above budget | sysfs + descriptors | post-1.0 |
| `usb.autosuspend_reset` | Autosuspend resets | reset following runtime suspend | sysfs + log | post-1.0 |
| `usb.overcurrent` | Port over-current | over-current event on a port | kernel log | post-1.0 |
| `usb.reset_storm` | Repeated bus resets | resets/errors per device above baseline | kernel log | post-1.0 |

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

### Cross-check: blocked-time share

Request share answers "who sent the work." A second, independent signal answers "who was blocked":
`delayacct_blkio_ticks` deltas for the top suspects.

```text
blocked_share_i = delta(delayacct_blkio_ticks_i) / sum_j delta(delayacct_blkio_ticks_j)
```

Agreement between `friction_i` and `blocked_share_i` promotes the finding's confidence. Disagreement
is diagnostically interesting and must be surfaced, not averaged away:

| Pattern | Interpretation |
| --- | --- |
| High request share, high blocked share | straightforward culprit |
| High request share, **low** blocked share | cheap requests (large sequential writes) — probably not the pain |
| Low request share, **high** blocked share | few, pathological requests (fsync/small random) — real culprit |
| Neither, high PSI | device or kernel work (`io.device_stall`, jbd2) |

Caveats: 10 ms tick granularity (fine for 60 s windows, useless for 1 s), per-task accounting
(thread groups need `/proc/<pid>/task/*/stat`), and the same block-layer bias as PSI. Use it when
armed, on ≤ 16 suspects, never for every process on every tick.

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

Authoritative numbers and enforcement: [`PLAN.md`](PLAN.md) §5.

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
| Ring buffer | unit + property tests | wrap-around, eviction, no concurrency |
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
frictionpoint why                                # recent episode in one sentence (alias: report --since 15m --last)
frictionpoint watch [--tui]                      # live view; findings stream, not charts
frictionpoint report [--since 1h]                # episode summary from the daemon
frictionpoint explain <finding-id>               # raw evidence + the arithmetic behind a finding
frictionpoint doctor                             # what this host can and cannot see, and why
frictionpoint record --for 120s --label <name>   # capture a scenario fixture
frictionpoint replay <file.fpz> [--json]         # deterministic detector run over a fixture
frictionpoint export --format prometheus|otlp|jsonl
frictionpoint config --explain                    # resolved value + which file won
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

## 14. Roadmap

**Authoritative schedule and acceptance criteria: [`PLAN.md`](PLAN.md) §11-12.** Summary only:

| Milestone | Content | Gate |
| --- | --- | --- |
| v0.1 | A0 partial: PSI, diskstats, `io.full_stall`, `report`, `doctor` | truthful output; exit 3 when blind |
| v0.2 | **A1 complete**: storage stall engine, 4 detectors, fixtures, replay, coverage | storm names the unit; GC stall names the device; victim not blamed |
| v0.3 | **A2 complete** + TUI + packaging | hardware separated from workload; baseline silent |
| v1.0 | field validation, truth audit, docs, launch | seven acceptance criteria in PLAN §11 |
| v1.1-v1.4 | A3-A6: config, memory, CPU, dependencies | one area per release |
| post-1.0 | A7 USB, hub, trace, Windows, Android | separate efforts |

**v1.0 ships A1 + A2** (ten rows in §7). Everything else is backlog, and a release is an area, not a
metric.

---

## 15. Repository skeleton (files, not crates)

```text
crates/fp-core/src/
  lib.rs model.rs ids.rs time.rs series.rs intern.rs hist.rs score.rs narrative.rs
  detect/mod.rs detect/io.rs detect/mem.rs detect/cpu.rs detect/hw.rs detect/residual.rs
crates/fp-linux/src/
  lib.rs psi.rs cgroup.rs procfs.rs diskstats.rs vmstat.rs jbd2.rs mmc.rs thermal.rs swap.rs kmsg.rs
crates/fp-agent/src/
  lib.rs loop.rs config.rs schedule.rs coverage.rs spool.rs redact.rs notify.rs sink/{mod,jsonl,http}.rs
crates/fp-cli/src/
  main.rs tui.rs cmd/{mod,baseline,report,explain,doctor,selftest,record,replay,export,watch}.rs
crates/fp-trace/src/       # optional, only crate with unsafe
  lib.rs fanotify.rs
crates/fp-hub/src/         # post-1.0
  main.rs ingest.rs store.rs api.rs episode.rs
xtask/src/main.rs          # verify pipeline, perf gate, fixture tooling
fixtures/{pi,server,clean}/*.yaml
fixtures/bin/*.fpz         # recorded raw samples (small, committed)
docs/adr/*.md
```

Hygiene from day one: `resolver = "2"`, `[workspace.lints]`, pinned `rust-version`,
`#![forbid(unsafe_code)]` in every crate except `fp-trace`, `cargo-deny` for licenses/advisories,
`cargo-machete` for unused dependencies, and `cargo fmt --check` + `clippy -D warnings` as required
gates.

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
