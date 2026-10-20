# Audit: assumptions, overclaims, and corrections

This is a self-review of the plan to date, judged against one constraint that overrides the
others: **one developer, part-time, must still be maintaining this in a year.**

Tags: `[OK]` holds up · `[FIX]` needs correction · `[CUT]` remove from scope ·
`[UNPROVEN]` needs measurement before it is a claim · `[OVERTURNED]` my earlier decision was wrong.

---

## A. Claim audit

| # | Claim made | Verdict | Correction |
| --- | --- | --- | --- |
| A1 | "Cross-platform: Linux, Windows, Android, SBCs, + more" | `[CUT]` for v1.0 | v1.0 is **Linux/SBC only**. Windows is a second product-sized effort (ETW schemas, DPC/ISR, USB topology) and becomes a separate crate + binary after 1.0. Android is advisor-mode-only and may never ship. macOS is dropped. |
| A2 | "< 1% CPU on a Pi Zero" | `[UNPROVEN]` | At 1 Hz, polling 200 PIDs × 3 files ≈ **600 opens/sec**, which alone is roughly 0.6% of a Pi Zero core. Correction: **cgroup-first attribution + staggered PID sampling** (§C1). Claim becomes "measured on Pi Zero 2 W, target < 1%", with a CI gate. |
| A3 | "0 bytes written to disk at idle" | `[FIX]` | False as stated: `stdout` goes to journald, which may persist it. Correct claim: **"no data writes of our own; state changes only, rate-limited; journald persistence is the operator's choice."** |
| A4 | "Identify all classes of error" | `[CUT]` | Impossible, and claiming it invites distrust. Replace with a **coverage matrix** (§PLAN) plus a first-class `unknown_friction` outcome and a capture-escalation path. |
| A5 | "Help resolve issues" | `[FIX]` | Core **advises only**. Remediation is a separate, consent-gated companion (post-1.0). Never automatic. |
| A6 | "One binary, many subcommands" | `[OVERTURNED]` | A root-running agent must **not link a network stack**. Two artifacts (`frictionpoint`, `frictionpoint-hub`) from one workspace, one `Finding` schema, versioned together. See ADR-0004. |
| A7 | "12 detectors in Phase 1" | `[CUT]` | Eight detector rows for v1.0 (areas A1+A2 in `AREAS.md`), plus residual. The original twelve was still too many. |
| A8 | "eBPF gives `traced` confidence" | `[CUT]` from v1.0 | `aya` + BTF + a build pipeline + CO-RE testing is a project in itself, and many SBC kernels lack BTF. v1.0 caps at `correlated`; `traced` arrives with the optional `fp-trace` feature. **Do not market `traced` before it exists.** |
| A9 | "A hub can be added later without rework" | `[OK]` | Holds **only if** the five schema seams in `FLEET.md` ship in v0.1. That is now a release gate. |
| A10 | "Streamers / USB / sleep-wake story" | `[CUT]` from v1.0 | Keep it as the *reason Windows matters*, not as a v1.0 promise. It requires ETW + USB topology + a second platform. |
| A11 | "Runs in Docker" | `[FIX]` | Runs, but **degraded and self-declaring**; never full fidelity. Already true in `DEPLOYMENT.md`, but the README must not imply parity. |
| A12 | "Agents can build this" | `[OK]` | Contingent on fixtures-first (record/replay). Without them, agents have no scoreboard. |

---

## B. Scope cuts (explicit, with reasons)

| Cut | Reason |
| --- | --- |
| Windows, Android, macOS in v1.x | Each is a full platform effort. One dev cannot maintain three ETW/Perfetto/procfs backends. |
| eBPF / per-request tracing in v1.0 | Toolchain + BTF + kernel variance + unsafe + a second build system. Fanotify covers file-level attribution cheaply. |
| Any automatic remediation | Privilege, rollback, bricking, and trust. Advice only, forever, in-core. |
| Vulnerability scanning, firmware/driver updates | Different domain, different threat model, different update cadence. |
| Web pane before 1.0 | `fleet --hub` TUI proves the API for a day of work. A web app is a second product. |
| MQTT before v0.4 | Pulls an async runtime and TLS into the agent. Webhook/loopback first. |
| Topology auto-discovery | Declared + discovered edges only. Inferred edges cannot justify causality anyway. |
| SQLite in the hub (initially) | Compressed JSONL + in-memory index is enough for 15 hosts at ~1.2 MB/day. |
| TLS termination in the hub | Run it on loopback/unix socket behind Caddy/nginx. Removes an entire dependency and key-management burden. |
| ML/anomaly models | Robust thresholds + EWMA/MAD is auditable and explainable. "The model decided" is not a diagnosis. |
| Localization, plugin ABI, scripting | Sprawl. Not now, probably not ever. |
| Prometheus as a primary path | Counters cannot carry evidence, confidence, or narrative. Export only. |

---

## C. Technical corrections (these change the design)

### C1. Do not poll every PID on every tick `[FIX]`

At 1 Hz with 200 processes, reading `/proc/<pid>/io`, `/status`, `schedstat` is ~600 opens/sec.
That is not free on a Pi Zero and it is pure waste: most processes are idle.

**Correction — attribution pyramid:**

| Layer | Frequency | Cost | Role |
| --- | --- | --- | --- |
| PSI (system) | 1 Hz | 1 read × 3 | severity truth |
| `diskstats` / `vmstat` | 1 Hz | 2 reads | device truth |
| cgroup `io.stat` / `*.pressure` | 1 Hz | O(units) ≈ 20 | **primary attribution** |
| per-PID counters | 1 Hz, **1/5 of PIDs, round-robin** | ~40 opens/sec | suspect refinement |
| per-PID all | armed only, ≤ 10 s | burst | second window |

Consequences: the cgroup layer is always complete and cheap; PID-level detail is a refinement that
converges in ~5 s; escalation covers the cases where 5 s is too slow. This also means **v1.0 needs
cgroup v2 for good attribution** — another reason to report cgroup v1 hosts as degraded.

### C2. No async runtime in the agent `[FIX]`

Nothing in a 1 Hz poll loop needs `tokio`. Dropping it removes roughly **80-100 transitive
dependencies**, 1-2 MB of runtime state, a thread pool, and a class of shutdown/backpressure bugs
that an asynchronous design invites in a program that must never block.

**Correction:** one thread, one loop, blocking reads with short timeouts. Sinks that can block
(HTTP) drain from a bounded queue on **one** worker thread via `std::sync::mpsc`. No executor, no
`select!`, no cancellation tokens. The CLI TUI renders from the same loop (poll → detect → render)
so there is no shared mutable state and no locking.

### C3. No `unsafe` in the core, and no need for it anywhere in the agent `[FIX]`

My earlier plan used a lock-free ring buffer, `crossbeam`, `loom`, and `miri`. A single-threaded
agent needs none of that. A fixed-size array plus an index is safe, simpler, and faster.

**Correction:** `#![forbid(unsafe_code)]` across `fp-core`, `fp-linux`, `fp-agent`, `fp-cli`.
The only crate permitted `unsafe` is the optional `fp-trace` (fanotify/eBPF syscalls), which is
excluded from the default build. `cargo geiger` in CI asserts the quarantine holds. This deletes
`loom`, `miri`, and the concurrency review from the maintenance burden.

### C4. Hand-parse the procfs files we need `[FIX]`

The `procfs` crate allocates `String`s per field, tracks kernel-version details we do not need, and
adds a dependency we would have to audit on every bump. We need **nine** files with stable formats.

**Correction:** a small parser module per file (~400-600 LOC total), parsing into reused buffers,
with `proptest` + `cargo-fuzz` coverage and explicit rollover handling. Formats are kernel ABI and
change far less often than crate APIs.

### C5. Banned dependencies `[CUT]`

| Banned | Why | Replacement |
| --- | --- | --- |
| `tokio`, `async-std`, `futures` | async runtime in a 1 Hz loop | std threads / blocking I/O |
| `crossbeam`, `parking_lot`, `loom` | no shared mutable state exists | plain arrays, `std::sync::mpsc` |
| `procfs`, `sysinfo` | allocation churn, wide surface | hand-rolled parsers |
| `regex` | large, backtracking, unsafe to feed kernel strings | manual parsing |
| `chrono` | size/timezone surface | `std::time` + `time` in CLI only |
| `tracing`, `log` | macro machinery for ~200 lines of logging | 60-line stderr/journal writer |
| `anyhow` | type-erased errors at library boundaries | `thiserror` or hand-rolled enums |
| `reqwest`, `hyper`, `axum` | runtime + TLS + huge graph | `ureq` (optional sink), `tiny_http` (hub) |
| `rusqlite`, `sqlx` | C library / async + migrations | JSONL + in-memory index |
| `openssl` | C + system coupling | `rustls` where TLS is needed |
| `zstd` | C build pipeline | uncompressed fixtures first; `flate2` (pure Rust) if needed |
| `clap` **derive**? | acceptable, but see PLAN | `clap` builder or `lexopt` |

### C6. `stdout` is a disk write `[FIX]`

journald persists service output depending on `Storage=`. A tool that claims "zero writes" while
logging every tick is lying.

**Correction:** log only state transitions (start, degrade, recover, finding emitted, sink failure),
rate-limited to ≤ 5 lines/minute, and document that journald may persist them. Add
`LogRateLimitBurst`/`LogRateLimitIntervalSec` to the unit.

### C7. PSI does not see every remote wait `[FIX]`

Network-filesystem and socket waits may not enter block-layer accounting. A fleet story built only
on PSI will miss `RemoteWait` victims.

**Correction:** maintain an independent **D-state census with `wchan`** at 1 Hz (cheap: read
`/proc/<pid>/stat` state + `wchan` only for D-state tasks, typically < 5). This is also the signal
that distinguishes "hardware stalling" from "waiting on a peer."

### C8. Allocation discipline is a feature, not a style `[OK]` + enforcement

- All per-tick buffers fixed-size, allocated at startup.
- Strings interned to `u32` ids; one arena with a hard cap and LRU eviction.
- Entity maps capped (1024 PIDs / 256 cgroups) with eviction counters.
- **Coverage accounting:** report the share of observed I/O requests that were attributable. If we
evict or fail to attribute, say so. A diagnosis without a denominator is a guess.
- Zero allocations per tick, asserted by a counting allocator in integration tests.

### C9. Hub without a TLS stack `[FIX]`

Terminating TLS in the hub means keys, rotation, and a crypto dependency in a service that only
needs to serve JSON on a LAN.

**Correction:** hub binds to **loopback or a unix socket**; operators put Caddy/nginx in front if
they want remote access. Token auth regardless. This removes `rustls` from the hub entirely.

### C10. Keep the Pi binary small with features `[FIX]`

TUI and HTTPS sink are **optional features**, off in the default agent build. Target: default
`frictionpoint` binary < 5 MB stripped, no network code linked, no terminfo, no TLS.

---

## D. Missing pieces found during the audit

| Gap | Why it matters | Fix |
| --- | --- | --- |
| Coverage denominator | Attribution can silently degrade | Report `attributed %` with every finding |
| Self-exclusion test | The observer must not appear as a culprit | Assert in CI; exclude own cgroup/PID from scoring |
| PID reuse guard | A recycled PID misattributes blame | Match `starttime` from `/proc/<pid>/stat` |
| Counter rollover / reset | Negative deltas produce nonsense | Detect decrease → discard window, count anomaly |
| Clock jump detection | Mono/wall divergence corrupts joins | Compare deltas; mark `skew_unknown`, drop cross-host joins |
| Device/mount disappearance | Reads fail mid-window (USB/SD removal) | Re-resolve handles; close and rescan; emit `device_lost` |
| Permission lost at runtime | Silently half-blind | Re-run capability probe every 60 s; emit `visibility_lost` |
| Graceful shutdown | Lost in-flight findings on SIGTERM | Flush bounded queue with a deadline before exit |
| Own-log rate limiting | Log flood becomes the pathology | 5 lines/min cap, state-transition only |
| Config validation | A typo silently disables a detector | Strict schema, unknown keys rejected, `--check-config` |
| Uninstall path | Root tool leaves users/systemd units behind | Documented removal, `frictionpoint uninstall --dry-run` |
| Contribution policy | Side project, needs low-friction governance | DCO, "one detector = one PR", fixture required |
| Error message taxonomy | Users need next steps, not errno | Every error has a cause + a suggested action |
| Fixture size policy | Armed traces can be huge | Cap + downsample; 2-min fixture target < 1 MB |

---

## E. Security review (threat model)

A root-running diagnostic that reads process names, paths, and kernel logs is a high-value target.
The design must make it boring.

| Threat | Surface | Mitigation |
| --- | --- | --- |
| Parser exploit via attacker-controlled strings (process names, cgroup names, dmesg) | every parser | No `unsafe`, no regex, no shell, bounds-checked manual parsing, `cargo-fuzz` in CI, length caps |
| Symlink / TOCTOU on config, spool, or fixture paths | file opens | `openat` + `O_NOFOLLOW` for writable paths; `0700` state dir; never accept user paths in the agent |
| Local user reads findings via the CLI socket | unix socket | `0600`, `SO_PEERCRED` uid check, no remote socket by default |
| Privilege escalation via excess capabilities | systemd unit | 3 capabilities max, `NoNewPrivileges`, `SystemCallFilter=@system-service`, no exec |
| Agent coerced into network access | linked code | **Default binary does not link a network stack**; `RestrictAddressFamilies=AF_UNIX`; sinks are a build feature |
| Hub exposed to the internet | hub | loopback/unix socket bind, token auth, read-only API, no remediation path, reverse proxy for TLS |
| Untrusted hub input | ingest | strict schema, size caps, rate limit, reject unknown fields *at ingest only*, no shell |
| Supply chain | dependencies | ≤ 12 direct deps, `cargo-deny` (licenses/advisories/bans), `cargo-vet` optional, checksummed + signed releases, SBOM |
| Sensitive data exfiltration | sinks | Agent-side redaction tiers, explicit opt-in, no telemetry, no default network |
| DoS by the tool itself (log flood, disk fill) | own output | Rate-limited logs, no files by default, bounded queues, systemd `MemoryMax`/`CPUQuota` |
| Root reading arbitrary files | probe design | Fixed path allowlist; no user-supplied path arguments in privileged mode |

**Non-negotiable:** no remote command execution, ever, in any tier. Remediation is local and
consent-gated, or it does not exist.

---

## F. Maintainability review (the real constraint)

| Dimension | Target | Rationale |
| --- | --- | --- |
| Core LOC | ≤ 6,000 (excl. tests) | Auditable in an evening |
| Total v1.0 LOC incl. tests | ≤ 18,000 | One dev can hold it in their head |
| Direct dependencies (agent) | ≤ 8 | Supply-chain and bump burden |
| Direct dependencies (total) | ≤ 16 | Including hub/trace |
| CI wall time | ≤ 5 min | Otherwise it gets skipped |
| Public API | `pub(crate)` by default; documented crate roots only | Small contracts |
| `unsafe` blocks | 0 outside `fp-trace` | Enforced by `cargo geiger` |
| Panics in non-test code | 0 (`clippy::unwrap_used`/`expect_used` denied) | A monitor must not die |
| Detector unit | 1 file + 1 fixture + 1 golden + 1 doc line | The "one detector = one PR" rule |
| Macros | none beyond `derive` | Readability |
| ADRs | one per irreversible decision | Prevents re-litigating |

**Error policy:** libraries return typed enums (`thiserror` at most); binaries translate to a
user-facing message with a *cause* and a *next action*; no `unwrap`/`expect`/`panic!` outside
tests and startup invariants; `catch_unwind` at the daemon loop boundary so a bug degrades to
"detector disabled" rather than "monitoring stopped."

**Complexity budget:** if a detector needs more than ~150 lines, it is probably two detectors or a
missing helper. If a probe needs more than ~250, it is probably doing detection too.

---

## G. What we will never be able to do (state this publicly)

1. Diagnose a victim we do not monitor.
2. Prove causality without a declared or discovered edge.
3. Count `fsync` from procfs alone (needs fanotify or tracing).
4. Read SD-card internals (no SMART, no NCQ, no trustworthy `%util`).
5. Guarantee PSI exists on every kernel.
6. See host filesystem events from inside a default container.
7. Fix anything automatically.
8. Attribute 100% of I/O (caps, evictions, kernel threads).
9. Replace a profiler for application-internal slowness.
10. Be equally useful on Windows/Android without a second dedicated effort.

Each of these becomes a line in `doctor` or the docs. Honesty is the product.

---

## I. Second-opinion review (an external outline)

A reviewer proposed: in-memory ring buffer with disk/alerts only on breach; use `/proc`, `/sys`,
`inotify`, and `perf_event`; human-readable wait-state translation; then ETW/PresentMon for Windows
and USB topology; then an "action" companion layer; and for agentic development, modular crates
(core / linux / CLI) and strict error handling.

**Overall:** directionally aligned with about 60% of our Phase 1 — and useful as independent
validation that the shape is obvious to others. It is an outline, not an architecture. It omits the
signals and mechanisms that make the diagnosis defensible.

### Adopted

| Suggestion | What we take | How it lands |
| --- | --- | --- |
| Per-process "disk wait times" | **`delayacct_blkio_ticks`** (field 42 of `/proc/<pid>/stat`) — cumulative per-task block-I/O wait in clock ticks | New data source. Used as a **blocked-time cross-check on top suspects when armed** (see `ARCHITECTURE.md` §8). Cheap, and it validates or contradicts IOPS-share blame |
| Core/companion separation | Already our Core / Recipes / Fix / Fleet split | Confirmed; no change |
| Crate separation (core / linux / CLI) | Already our crate map | Confirmed; but see the correction below about "core = ingestion" |
| Strict error handling | Already policy | Reinforced: no panics, degrade, re-probe permissions |

### Rejected, with reasons (now written down so nobody re-suggests them)

| Suggestion | Why not |
| --- | --- |
| **`inotify`** for process I/O attribution | inotify **does not report the accessing PID** — it says a file changed, not who changed it. It is per-inode/per-directory (watch explosion, `max_user_watches`), not per-mount. Wrong tool. **fanotify** is the correct choice: mount-wide marks plus PID/FID reporting |
| **`perf_event`** as a primary source | Plausible for context-switch counting (`PERF_COUNT_SW_CONTEXT_SWITCHES`, `inherit=1` aggregates a thread group in one fd), but it costs an fd per tracked process, depends on `perf_event_paranoid`, and CAP_PERFMON. **Spike candidate if the CPU budget is missed**, not a v1.0 dependency |
| **PresentMon** principles for sleep/network bugs | Category error: PresentMon measures frame/present timing via ETW. It cannot explain NDIS/power transitions. The right providers are `Kernel-Power`, `NDIS`, `Kernel-Network`, `StorPort`, `USB-*` |
| "Commit to disk when a threshold is breached" | Reintroduces small writes on the device we are diagnosing. Corrected: **no disk writes by default**; optional bounded spool, off by default, prefer tmpfs |
| "friction-core for data ingestion" | `fp-core` must stay **pure** (no files, no clocks, no OS). Ingestion belongs in `fp-linux`; scheduling in `fp-agent`. That separation is what makes detectors testable without root |

### The biggest omission

**No PSI.** Their plan infers "disk wait" from per-process counters; that cannot produce the metric
the user actually feels — *how long the whole system was stalled* — nor distinguish "some tasks
stalled" from "everything stalled at once." PSI `full` is the headline number in this product, and
it is also the cheapest read in the entire design. Without it there is no lost-time denominator and
no way to prioritise findings by user impact.

Also absent: cgroup v2 attribution (the cheapest per-unit blame), confidence tiers, coverage
accounting, clock/skew handling for multi-host, fixture-based testing, and any budget enforcement.
Those are the difference between a demo and a tool you can trust at 3 a.m.

### Correction this review surfaced

Our own trap list said fsync counts could come from fanotify. **fanotify has no `FAN_FSYNC` event.**
It reports opens/reads/writes/closes/creates/moves/errors — useful for *file* attribution, but not
for counting `fsync`. Actual fsync counting requires an eBPF syscall tracepoint, so v1.0 must infer
fsync storms from write-syscall rate + small mean request size + elevated latency, and label that
inference `correlated`. Fixed in `ARCHITECTURE.md`.

---


## H. Verdict

**Feasible, if the scope above is enforced.** The plan as originally written was 2-3x too large for
one part-time developer; with the cuts it becomes a focused Linux/SBC tool with a credible path to
fleet mode.

| Item | Estimate |
| --- | --- |
| v0.1 walking skeleton (PSI + full_stall + report + doctor) | 1.5-2 weeks part-time |
| v0.2 Pi alpha (A1: storage stall, 4 rows, fixtures, replay, perf gate) | +4 weeks |
| v0.3 Pi beta (TUI, alerts, packaging, hardening, fuzzing) | +4 weeks |
| v1.0 Linux (field validation, corpus, docs, launch) | +4 weeks |
| **Total to v1.0** | **~14 weeks part-time** |

> **Superseded:** these figures assumed ~100 LOC/hour. `PLAN.md` §11 estimates ~7 months
> part-time at a realistic 30-40 LOC/hour.
> **Superseded:** `PLAN.md` §11 now estimates ~7 months part-time at 30-40 LOC/hour. The table
> below assumed ~100 LOC/hour and was optimistic.

| Fleet hub (v0.4) | +4-6 weeks |
| Windows | +8-12 weeks (separate effort, separate artifact) |

The biggest risk is not technical. It is **scope re-expansion** — adding a platform, a UI, or a
detector because it is interesting. The coverage matrix and the six-detector gate exist to make
that visible before it happens.
