# FrictionPoint — Consolidated Build Plan (v1.0)

Authoritative plan after [`AUDIT.md`](AUDIT.md). Where this document and an earlier design note
disagree, **this document wins**; the others get amended.

**v1.0 promise:** on Linux and SBCs, explain storage/pressure stalls with evidence, name the
responsible process/unit/device, and stay under 1% CPU while doing it. Everything else — Windows,
Android, the fleet hub, tracing — is explicitly *later* and must not shape v1.0's code.

---

## 1. Principles (the tie-breakers)

1. **Never harm the host.** No blocking, no unbounded memory, no writes of our own, no panics.
2. **Never lie.** Confidence tiers, coverage denominators, `doctor`, and exit code 3.
3. **Stall time over utilization.** Lost milliseconds is the metric.
4. **Requests over bytes.** Latency is charged per request.
5. **Cheap by default, deep on demand.** Armed capture with a time and frequency budget.
6. **One detector = one file + one fixture + one golden test.**
7. **Boring tech in the hot path.** std, arrays, integers, no async, no `unsafe`.
8. **Small surface beats clever surface.** Every dependency is a maintenance contract.
9. **Defer, don't preclude.** Schema seams now; services later.
10. **If it isn't measured, it isn't a claim.**
11. **Zero privilege to diagnose; privilege only to refine.** PSI and cgroup `io.stat` are
    world-readable, so a complete unit-level diagnosis must work as an unprivileged user. Root
    or capabilities may only add process/file refinement.
12. **Truth over self-consistency.** Tests against hand-made data prove the code agrees with
    itself. Real captures, a field soak, and a manual audit are the only evidence that counts.

---

## 2. Crate map

**Five crates in v1.0** (plus two later artifacts: `fp-hub`, `fp-windows`). Not nine. Each has
one job and a hard LOC budget.

```text
                         ┌──────────────────────────────┐
                         │           fp-core            │  pure, no I/O, no OS, no unsafe
                         │ model · ids · time · series  │  interned strings, histograms,
                         │ detectors · scoring ·        │  scoring, narrative
                         └──────────────▲───────────────┘
                                        │
              ┌─────────────────────────┼──────────────────────────┐
              │                         │                          │
      ┌───────┴────────┐      ┌─────────┴─────────┐      ┌─────────┴────────┐
      │   fp-linux     │      │     fp-agent      │      │     fp-hub       │
      │ 9 parsers +    │─────▶│ loop · schedule   │      │ ingest · store   │  (post-1.0)
      │ probes         │      │ coverage · sinks  │      │ /v1 API · joins  │
      │ no unsafe      │      │ redaction · spool │      │ no DB (JSONL)    │
      └───────▲────────┘      └─────────▲─────────┘      └──────────────────┘
              │                         │
      ┌───────┴────────┐      ┌─────────┴─────────┐
      │   fp-trace     │      │      fp-cli       │
      │ fanotify/eBPF  │      │ binary: report ·  │
      │ ONLY unsafe    │      │ explain · doctor ·│
      │ optional       │      │ record · replay · │
      └────────────────┘      │ tui (feature)     │
                              └───────────────────┘
```

| Crate | Responsibility | Deps | LOC budget | `unsafe` |
| --- | --- | --- | --- | --- |
| `fp-core` | Model, deterministic ids, clock, series ring, interner, histogram, detectors, scoring, narratives, advice ids | `serde`, `thiserror` | 5,000 | **forbid** |
| `fp-linux` | Parsers + probes: PSI, cgroup v2, procfs, diskstats, vmstat, jbd2, mmc/sysfs, thermal/`vcgencmd`, swaps, kmsg | none beyond core | 3,000 | **forbid** |
| `fp-agent` | Tick loop, staggered scheduling, coverage accounting, sinks (JSONL, HTTP feature), spool, redaction, sd_notify, signal handling | core, linux, `serde_json`, optional `ureq` | 2,500 | **forbid** |
| `fp-cli` | `frictionpoint` binary: `baseline`, `report`, `explain`, `doctor`, `selftest`, `record`, `replay`, `export`, `watch` | `clap`, `time`, optional `ratatui`+`crossterm` | 2,500 | **forbid** |
| `fp-trace` | fanotify (v1.x), eBPF (post-1.0) | `libc` | 1,000 | allowed, quarantined |
| `fp-hub` | Ingest, retention, `/v1`, episodes, causality | `tiny_http`, `serde_json` | 2,000 | **forbid** |
| `fp-windows` | ETW/PDH probes — **separate artifact, post-1.0** | `windows-sys`, `ferrisetw` | — | contained |

**Dependency direction is enforced by review, not tooling:** platform crates never depend on each
other; `fp-core` depends on nothing local; `fp-hub` and `fp-windows` link `fp-core` only.

Why this shape:

- `fp-core` is pure, so **every detector is testable with no root, no kernel, no hardware.**
- `fp-linux` is allowed to be tedious (parsers) and is fuzzable in isolation.
- `fp-agent` owns time and policy; it is the only place that knows about ticks and sinks.
- `fp-cli` owns presentation; it never opens `/proc` itself.
- `fp-trace` is the single quarantine zone for `unsafe`.

---

## 3. Dependency budget

**Agent default build: 3 direct dependencies.** Everything else is optional.

| Crate | Where | Why | Weight | Alternative rejected |
| --- | --- | --- | --- | --- |
| `serde` (derive) | core | the `Finding` contract | compile time only, 0 runtime | hand-rolled JSON — more code, more bugs |
| `thiserror` | core | typed errors at boundaries | proc macro only | hand-rolled enums (~120 LOC, acceptable fallback) |
| `serde_json` | agent/cli | JSON at the boundary only | ~200 KB | hand-rolled writer — reimplementing escapes is a security risk |
| `clap` (builder) | cli | 9 subcommands + help | ~500 KB | `lexopt` (~1 dep, hand-written help) |
| `time` | cli | timestamp formatting | ~50 KB | manual civil-time math |
| `ratatui` + `crossterm` | cli, **feature `tui`** | live view | ~400 KB, 2 crates | plain stdout table |
| `ureq` + `rustls` | agent, **feature `http`** | push to hub | ~1.5 MB, ~8 transitive | loopback-only hub (rejected: NAT) |
| `tiny_http` | hub | read API | ~40 KB | hand-rolled HTTP/1.1 (rejected: security) |
| `libc` | `fp-trace` only | fanotify/eBPF syscalls | tiny | `rustix` (safer, but another dep) |

**Total direct deps, all features: ~10-11.** Budget: ≤ 8 for the agent, ≤ 16 overall.

Banned list and reasons: see `AUDIT.md` §C5. CI enforces the budget with `cargo-machete`
(unused deps) and a checked-in count.

**Feature matrix**

| Feature | Adds | Binary size | Who needs it |
| --- | --- | --- | --- |
| *(default)* | parsers, detectors, JSONL export, report/doctor/replay | ~3-5 MB | everyone |
| `tui` | `ratatui`, `crossterm` | +400 KB | interactive use |
| `http` | `ureq`, `rustls` | +1.5 MB | fleet push |
| `trace` | `fp-trace`, `libc` | +50 KB | file-level attribution (v1.x) |

The default Pi binary **does not link a network stack**. That is a security property, not a size
optimization (ADR-0004).

---

## 4. Data flow and hot-path rules

```text
 tick (1 Hz)                     cold path (only on finding)
 ───────────                     ───────────────────────────
 read PSI        ─┐               score shares
 read cgroups     │               select culprit
 read diskstats   ├─▶ parse into  build narrative  ─▶ Finding ─▶ sink
 read vmstat      │   fixed bufs   (formats, no alloc
 staggered pids   │   (no alloc)    in the loop)
 D-state census  ─┘
        │
        ▼
   series ring (fixed capacity, overwrite oldest)
        │
   detectors (pure, over WindowView)
        │
   reusable finding buffer [Option<Finding>; 8]
```

Hot-path rules, all testable:

| Rule | Enforcement |
| --- | --- |
| Zero heap allocations per tick after warmup | counting allocator test |
| No `String` construction inside detectors | `&str` from interner; clippy review |
| No `format!` outside finding emission | grep gate in CI + review |
| No file opens from detectors | detectors only receive `WindowView` |
| Bounded everything | capacity constants + eviction counters |
| No blocking calls without timeout | sink thread + 200 ms timeouts |

---

## 5. Memory and CPU budget

| Component | Budget |
| --- | --- |
| Series rings: 24 channels × 4096 samples × 8 B | ~786 KB |
| Entity table: 1024 × 96 B | ~96 KB |
| String arena (interner) | 256 KB |
| Histograms: 3 × 128 entities × 64 buckets × 8 B | ~196 KB |
| Episode/tombstone ring: 256 × 512 B | ~128 KB |
| Config, paths, counters | ~32 KB |
| **Heap total** | **~1.5 MB** |
| **RSS target (idle)** | **≤ 12 MB** measured |
| **RSS target (armed)** | **≤ 20 MB** measured |
| Stack: main + one sink thread | 256 KB + 64 KB |

| Metric | Idle | Armed | Trace burst |
| --- | --- | --- | --- |
| CPU (Pi Zero 2 W) | < 1% | < 5% | < 15%, ≤ 10 s |
| Wakeups | ≤ 5/s | ≤ 60/s | ≤ 200/s |
| Disk writes of our own | 0 | findings only | 0 |
| Network (default build) | none | none | none |

Enforcement: counting allocator test, nightly hardware job, `MemoryMax=48M` + `CPUQuota=5%` in the
unit (the kernel enforces what the benchmark claims), and `doctor` self-reporting its own RSS/CPU.

**Eviction policy:** when a cap is hit, evict least-recently-seen and increment
`evicted_entities`. Coverage is reported as `attributed_requests / observed_requests`; below 90%,
the finding is emitted as `correlated` with a coverage caveat, never silently.

---

## 6. Error-class coverage map

The six classes from `VISION.md`, plus the honest seventh. "Coverage" means *how much of this class
the tool can name*, not how much exists.

| # | Class | Detectors | Primary sources | Phase | Coverage |
| --- | --- | --- | --- | --- | --- |
| 1 | **Resource starvation** | `io.full_stall`, `mem.reclaim_pressure`, `cpu.runqueue_delay`, `cpu.ctx_switch_storm` | PSI, vmstat, schedstat | v0.2-0.3 | **High** |
| 2 | **Latency amplification** | `io.small_write_storm`, `io.writeback_throttle`, `io.journal_serialization` | cgroup `io.stat` + `delayacct` cross-check, vmstat, jbd2 | v0.2-0.3 | **High** |
| 3 | **Cascade / shared fate** | `cascade.shared_storage`, `cascade.remote_dependency` | findings + edges (hub) | v0.5 | **Medium** (hub only) |
| 4 | **State-transition failure** | `io.remount_readonly`, `hw.mmc_errors`, `device_lost`, `visibility_lost` | kmsg, mount table, probe health | v0.2 | **Medium** (Linux) |
| 5 | **Hardware degradation** | `io.device_stall`, `hw.thermal_throttle`, `hw.undervoltage`, `hw.mmc_errors` | diskstats, sysfs, `vcgencmd`, kmsg | v0.2 | **High on Pi**, none elsewhere |
| 6 | **Misconfiguration with delayed cost** | `mem.swap_on_sd`, `fs.mount_misconfig`, `fs.log_growth` | `/proc/swaps`, `/proc/mounts`, fanotify | v0.2-0.3 | **Medium** |
| 7 | **Unknown / residual** | `unknown_friction` | coverage accounting + armed capture | v0.3 | **By design** — says "I can't attribute this" |

### Explicit gaps (do not hand-wave)

- Application-internal slowness (lock contention, GC, thread pools) is **out of scope**. We name the
  process, not the code path.
- GPU, audio, and display pipelines are **out of scope** in v1.0.
- Network path congestion is only inferred, never traced. "The NIC is saturated" is not
  "the switch is dropping frames."
- Kernel-thread work (kworkers, jbd2) is attributed to the *originating* unit only when the
  correlation is strong; otherwise it is reported as kernel work.

---

## 7. User-story claim ladder

What we can honestly say at each stage. Anything not listed is "not claimed."

| Story | v0.2 (Pi alpha) | v1.0 (Linux) | Post-1.0 |
| --- | --- | --- | --- |
| **Log flood takes down ingress** | "B stalled; 74% of write requests came from `receiver.service`; mean request 4 KiB." (the *bottleneck half*) | Same, with advice (buffer, move data dir, `commit=`), plus `blast_radius` from declared edges | Full chain: **A -> B -> C** with hub + topology; `cascade.log_sink` |
| **SD card write lock / read-only rootfs** | `io.remount_readonly` + `io.device_stall` + `hw.mmc_errors`: "card failure, back up now" | Same + card age/wear hints + `fs.mount_misconfig` | Windows analogue (deferred) |
| **Pi gets slow after hours** | `mem.swap_on_sd`, `io.writeback_throttle`, `io.journal_serialization` with named unit | Full ranking by lost time, per-unit advice | Cross-host: shared storage |
| **"Which box is sick?"** | Per-host `baseline` | Same, plus JSONL export for whatever dashboard you own | Hub fleet grid + blast radius |
| **Steam fails downloads after sleep** | *Not claimed* | *Not claimed* | Windows artifact (ETW power/NDIS) |
| **Streamer USB dropout** | *Not claimed* | *Not claimed* | Windows artifact (ETW USB topology) |
| **Android memory/storage opacity** | *Not claimed* | *Not claimed* | Exploratory advisor mode; may never ship |

This ladder is the marketing. Do not publish a row above its column.

---

## 8. Resilience playbook

"Resilient" for a monitor does **not** mean clustering and failover. It means: nothing that happens
on the host can make the tool harmful, silent, or dishonest.

| Condition | Behavior | User-visible |
| --- | --- | --- |
| PSI missing / disabled | Fall back to proxy signals | `doctor`: degraded; exit 3 |
| Permission lost mid-run | Drop the capability, keep running | `visibility_lost` finding once |
| Counter rollover / reset | Discard window, count anomaly | internal counter, not a finding |
| PID reused | Verify `starttime`; if mismatch, drop | no misattribution |
| cgroup removed with culprit | Tombstone for 60 s | finding still names the dead unit |
| Device/mount disappears | Close handles, rescan, emit `device_lost` | finding + advice |
| Clock jump | Compare mono vs wall delta | `skew_unknown`; no cross-host joins |
| Ring full | Evict oldest, count | coverage % in findings |
| Sink/hub down | Bounded queue, drop oldest, one log line | `gap` marker on reconnect |
| Config invalid | **Refuse to start** with a precise message | exit 2; `--check-config` |
| Detector panics | `catch_unwind`, disable that detector | `detector_disabled` once |
| Own RSS over budget | Drop to least-priority sampling | `doctor` warning |
| SIGTERM | Flush bounded queue with deadline, exit 0 | clean shutdown |
| Storage full (spool) | Stop spooling, keep analyzing | one log line |

Rationale for fail-closed config: a monitor that silently runs with half its thresholds missing is
worse than one that refuses to start.

---

## 9. Security controls (implementation checklist)

Threat model: `AUDIT.md` §E. These are the controls that must exist in code, not just in prose.

| Control | Where | Test |
| --- | --- | --- |
| `#![forbid(unsafe_code)]` | core, linux, agent, cli, hub | `cargo geiger` asserts only `fp-trace` |
| No shell execution | all crates | grep for `Command`; one audited exception in `fp-linux` for `vcgencmd` with fixed argv |
| Fixed path allowlist in privileged paths | `fp-linux` | unit test: no user string reaches `open()` |
| `O_NOFOLLOW` + `0700` state dir | agent | integration test with a planted symlink |
| Socket `0600` + peer credential check | cli/agent | integration test |
| Capability ceiling | systemd | `systemd-analyze security` in CI on the unit file |
| No network in default build | Cargo features | `ldd`/`nm` check in release CI |
| Parser fuzz targets | `fp-linux` | `cargo-fuzz`, 5 min/night, corpus committed |
| Redaction before sink | agent | test: path appears nowhere in tier-0 output |
| Rate-limited logging | agent | test: 10k findings emit ≤ 5 log lines/min |
| Unprivileged completeness | agent | CI runs the detector suite as non-root and asserts a culprit is named |
| Dependency policy | CI | `cargo-deny` (advisories, licenses, bans) + `machete` |
| Signed, checksummed artifacts | release | `cosign`/minisign + SHA256SUMS |
| SBOM | release | `cargo-sbom` or `syft` |

No remote command execution in any tier. Remediation, when it exists, is local and consent-gated.

---

## 10. Testing and CI map

| Layer | Tooling | Scope | Runtime |
| --- | --- | --- | --- |
| Parsers | `proptest` + `cargo-fuzz` | truncated lines, rollover, weird names | 5 min nightly |
| Detectors | `insta` golden over fixtures | kind, culprit, confidence, coverage | < 30 s |
| Scoring | property tests | shares ≤ 1.0, no div-by-zero, monotonic severity | < 5 s |
| Agent loop | integration with a **fake probe** | scheduling, eviction, coverage, sinks | < 30 s |
| Allocation/perf | counting allocator + `xtask perf` | zero allocs, RSS delta, tick p99 | < 60 s |
| Narratives | `insta` snapshots | misleading wording, missing evidence | < 10 s |
| Unit file | `systemd-analyze security` | privilege ceiling | < 10 s |
| Fixtures | `record`/`replay` roundtrip | schema and determinism | < 30 s |
| Hardware | self-hosted Pi (nightly) | real PSI, real `mmc`, budgets | ~10 min |

**CI jobs (5, target ≤ 5 min total):**

1. `fmt` + `clippy -D warnings` + `machete`
2. `test` (nextest or plain cargo test)
3. `deny` (advisories/licenses/bans) + dep-count check
4. `cross` build `aarch64-musl` + `x86_64-musl` + binary-size check
5. `perf` (allocation + synthetic trace)

Nightly: fuzz, hardware Pi, `geiger`. Weekly: dependency review.

### Fixture corpus (start with six, not ten)

| Fixture | Injected fault | Ground truth |
| --- | --- | --- |
| `clean_baseline` | none | **zero findings** (false-positive gate) |
| `sd_fsync_storm` | `fio` 4 KiB `fsync=1` | `io.small_write_storm` -> fio |
| `sd_gc_stall` | aged card, natural stall | `io.device_stall`, no culprit |
| `swap_on_sd` | enable `dphys-swapfile` | `mem.swap_on_sd` |
| `dirty_throttle` | large buffered writers | `io.writeback_throttle` |
| `remount_ro` | injected ext4 error | `io.remount_readonly` + backup advice |

Stretch: `journal_commit`, `memory_reclaim`, `thermal_throttle`.

---

## 11. Schedule (part-time, ~10-12 h/week)

| Weeks | Milestone | Deliverables | Gate |
| --- | --- | --- | --- |
| 1-2 | **v0.1 skeleton** | workspace + lints; PSI, diskstats, vmstat parsers; `io.full_stall`; `report`; `doctor`; `selftest`; zero-alloc tick test | Truthful summary on a real Pi; exit 3 with explanation when PSI is absent |
| 3-4 | **Fixtures** | `record`/`replay`, fixture format, `clean_baseline` + `sd_fsync_storm`; `io.small_write_storm`; blame math + blocked-time cross-check | Replay is deterministic; clean baseline emits nothing |
| 5-6 | **Pi alpha (v0.2)** | `io.device_stall`, `hw.thermal_throttle`, `hw.undervoltage`, `mem.swap_on_sd`, `io.remount_readonly`; coverage accounting; kmsg reader | 6 fixtures, >= 5 correct culprits, budgets met on Pi Zero 2 W |
| 7-8 | **Agent + packaging** | daemon loop, staggered scheduling, JSONL sink, redaction tiers, systemd unit, `install.sh`, aarch64/armv7/armv6 builds | `systemd-analyze security` passes; binary < 5 MB; installer is idempotent |
| 9-10 | **Beta (v0.3)** | `watch` TUI, webhook sink, `explain`, `unknown_friction` residual, fuzz corpus, perf gate, docs | Fuzz clean 24 h; zero findings on baseline; CI <= 5 min |
| 11-12 | **Field validation** | Run on 3 real hosts (Pi 4, Zero 2 W, x86 server) for 2 weeks; tune thresholds; publish 3 case studies | Each case study has a reproducible fixture |
| 13-14 | **v1.0 Linux** | Launch kit, demo GIF, README polish, signed release, Home Assistant MQTT discovery doc | Seven acceptance criteria below |

### v1.0 acceptance criteria

1. On all three reference hosts, `baseline --for 60s` uses < 2 s of CPU and reports either a true
   finding or an explicit "nothing above threshold."
2. The six-fixture corpus scores >= 5 correct culprit/kind pairs, with **zero** findings on
   `clean_baseline`.
3. Idle RSS <= 12 MB and CPU < 1% on a Pi Zero 2 W, verified by `doctor` on hardware.
4. Zero `unsafe` outside `fp-trace`; `cargo-deny` clean; no panics in a 7-day soak.
5. A new user can install, run `baseline`, and understand the output without reading the docs.
6. **Truth audit:** 7-day soak on three hosts; 50 emitted findings manually verified; miss rate
   recorded. Corpus fixtures must be real captures, not hand-authored.
7. **Unprivileged completeness:** the detector suite runs as a non-root user and still names a
   culprit at unit granularity. If it cannot, the release is blocked.

If criterion 2 fails, **cut a detector rather than weaken the baseline gate.**


---

## 12. Post-1.0 order (each gated on real demand)

1. **`fp-hub` + `/v1` + `fleet` TUI** — only after >= 3 hosts are in use.
2. **Causal reasoner** — roles, edges, episodes, blast radius; requires `topology.toml` in the wild.
3. **MQTT sink + Home Assistant discovery** — the homelab distribution win.
4. **Web pane** — separate crate, public API only.
5. **`trace` feature (fanotify, then eBPF)** — promotes findings from `correlated` to `traced`.
6. **Windows (`fp-windows`, separate artifact)** — ETW disk, DPC/ISR, USB resets, sleep/wake.
7. **`act` mode** — bounded, reversible, receipted, verified; cgroup knobs only (`io.max`,
   `memory.high`), never kill/restart/sysctl/package installs.
8. **Android advisor** — exploratory; may be a separate repo or never ship.

Nothing on this list may add a dependency or a code path to the v1.0 agent binary.

---

## 13. Risk register

| Risk | L | I | Mitigation |
| --- | --- | --- | --- |
| Scope re-expansion (new platform/UI/detector mid-build) | **H** | **H** | This document is the gate; new ideas go to §12, not into the milestone |
| Threshold tuning rabbit hole | H | M | Fixed conservative defaults + fixtures; tune only from recorded cases |
| PSI unavailable on target kernels | M | H | Proxy fallbacks, `doctor`, exit 3 |
| Correlation wrong enough to lose trust | M | H | Confidence tiers, coverage %, conservative blame guardrails |
| Parser bug on unusual `/proc` content | M | H | Fuzz + proptest + length caps |
| One-dev burnout / stall | M | H | 2-week milestones, each shippable; no milestone depends on another landing |
| Upstream dependency churn | L | M | 3 deps in the agent; pinned; deny/audit |
| Netdata/vendor ships attribution | L | M | Our wedge is device-vs-workload reasoning + honesty, not collectors |

---

## 14. Definition of done

**Per detector:** spec comment (5 lines) + fixture + golden test + negative test (must *not* fire on
`clean_baseline`) + coverage accounting + narrative template + advice id + one docs line.

**Per release:** fmt/clippy clean, no new deps without an entry in §3, CI <= 5 min, budgets measured
on hardware, `doctor` output reviewed, changelog written.

**Per irreversible decision:** an ADR. Currently: 0001 (capability negotiation), 0002 (findings are
the wire format), 0003 (sync loop, no async), 0004 (agent does not link networking).

---

## 15. Open decisions (small, non-blocking)

1. `clap` vs `lexopt` for the CLI (binary size vs hand-written help).
2. Is `tui` a default feature for release binaries, or a separate build?
3. Hub placement (Pi next to broker vs NAS vs VPS) — only matters at v0.4.
4. Hub auth: shared token vs per-host token vs mTLS.
5. Whether `trace` ships in v1.x or waits for v2.
6. Run the `perf_event` spike only if the idle CPU budget is missed; otherwise do not revisit.

None of these changes the crate map, the schema, or the v1.0 promise.

