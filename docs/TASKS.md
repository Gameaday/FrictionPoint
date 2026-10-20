# TASKS — the execution plan for agents

This is the file you hand to a coding agent. It is deliberately ordered, sized, and testable.
Each task is independently mergeable and leaves the repository green.

## Rules

1. One task per branch and PR. Do not start the next task.
2. Every task begins with the failing test named in its card.
3. A task may not add dependencies, `unsafe`, async, threads, or schema fields.
4. A task may not touch files outside its "Files" list (plus its own tests).
5. If a task exceeds ~400 changed lines, stop and report.
6. **Write the full card for task N+1 at the end of task N.** Rolling detail prevents stale specs;
   the table below is the stable ordering.

## Area map (release gates)

Tasks are ordered by technical dependency. **Areas are the release gates** (see `AREAS.md`). Do not
start area N+1 until area N passes its exit checklist.

| Area | Tasks | Gate |
| --- | --- | --- |
| **A0** Foundations | T0-T6, T12 | Replay deterministic; `doctor` accurate on 3 hosts; zero-alloc tick; `clean_baseline` runs and emits nothing |
| **A1** Storage stall | T7-T11, T14, T19 (+T15/T16 in the first release) | Storm names the unit; GC stall names the device with `culprit: null`; victim case not blamed; `why` in one sentence |
| **A2** Hardware/power | T13 (thermal, `vcgencmd`, `mmc`, kmsg) + T20-T22 | Hardware separated from workload; `remount_ro` advises backup; baseline silent |
| **A3** Configuration | T13 (swaps, mounts) + T23-T26 | Every finding has a reversible command; nothing applied; baseline silent |
| **A4** Memory | T27-T29 | Cache churn distinguished from real pressure; cgroup named; baseline silent |
| **A5** CPU/scheduler | T30-T32 | Throttling reported with `nr_throttled`; cgroup named; baseline silent |
| **A6** Dependencies/network | T33-T35 | Never blames the local disk for a remote wait; confidence honest |
| **A7** Peripherals/USB | Epic, post-1.0 | Explains a dropout or states that `usbmon` is required |

T20+ are carded when the preceding area is complete (see rule 6). T15 and T16 are release plumbing:
needed once for the first alpha, then maintained rather than re-earned per area.

## Dependency graph

```text
T0 workspace + gates
 └─ T1 core primitives (ids, time, interner)
     └─ T2 series ring
         └─ T3 histogram

T0 ─┬─ T4 PSI parser ─────┐
    ├─ T5 diskstats ──────┼─ T7 io.full_stall ── T11 record/replay
    └─ T6 vmstat/meminfo ─┘                          │
                                                     ▼
                                     T8 blame math + cgroup io.stat
                                                     │
                                                     ▼
                                     T9 io.small_write_storm (fixture)
                                                     │
                                                     ▼
                                     T10 agent loop + staggered schedule
                                                     │
                    ┌────────────────────────────────┼──────────────┐
                    ▼                                ▼              ▼
              T12 doctor/selftest             T13 Pi hardware   T14 JSONL+redaction
                    │                                │              │
                    │                                └──────┬───────┘
                    │                                       ▼
                    │                             T19 explain / evidence
                    │                                       ▼
                    │                             T15 TUI (shape + sentence)
                    │                                       ▼
                    │                             T16 packaging + release
                    │                                       ▼
                    │                             T18 episodes + alert budget
                    │                                       ▼
                    └───────────────────────────── T17 truth-audit soak
```

## Task table

| ID | Task | Depends | Files (primary) | Acceptance evidence | Size |
| --- | --- | --- | --- | --- | --- |
| **T0** | Workspace, lints, CI, gates | — | root `Cargo.toml`, crate skeletons, `xtask/`, `deny.toml`, CI | `cargo xtask verify` passes on an empty skeleton; `cargo tree -p fp-agent` shows 0 deps | S |
| **T1** | ids, time, interner | T0 | `fp-core/src/{ids,time,intern}.rs` | id test vectors (4 cases from SCHEMA §4); mono/wall conversion; interner cap + eviction | M |
| **T2** | Series ring | T1 | `fp-core/src/series.rs` | wrap-around, eviction count, zero allocs per push, window view bounds | S |
| **T3** | Histogram | T1 | `fp-core/src/hist.rs` | p95/p99 within 5% of exact on a known set; empty and single-sample cases | S |
| **T4** | PSI parser | T0 | `fp-linux/src/psi.rs` | parses sample lines; missing file -> capability false, not error; truncated line -> typed error, no panic; proptest | S |
| **T5** | diskstats parser | T0 | `fp-linux/src/diskstats.rs` | known line -> exact fields; `in_flight` and `weighted_ms`; rollover -> discarded delta; malformed line skipped + counted | S |
| **T6** | vmstat + meminfo | T0 | `fp-linux/src/vmstat.rs` | `nr_dirty`, `nr_writeback`, `pgscan_direct`, `pgmajfault`; missing key -> `None`; unknown keys ignored | S |
| **T7** | `io.full_stall` | T1,T2,T4 | `fp-core/src/detect/io.rs` | severity from cumulative counter delta; **tick-rate independence property**; evidence non-empty; silent below threshold; golden JSON | M |
| **T8** | Blame math + cgroup `io.stat` | T2,T5 | `fp-core/src/score.rs`, `fp-linux/src/cgroup.rs` | shares sum <= 1.0; small-write factor table; `< 40%` share -> `culprit: null`; no division by zero; coverage denominator | M |
| **T9** | `io.small_write_storm` | T7,T8,T11 | `fp-core/src/detect/io.rs` | fires on `sd_fsync_storm`, names the unit; **silent on `clean_baseline`**; advice id; coverage populated | M |
| **T10** | Agent loop + staggered schedule | T7,T9 | `fp-agent/src/{loop,schedule}.rs` | 1 Hz tick; only 1/5 PIDs per tick; injectable clock; deterministic in tests; zero allocs after warmup | M |
| **T11** | record / replay | T7 | `fp-agent/src/{record,replay}.rs` | raw samples recorded; replay reproduces identical findings **and ids**; versioned header; 2-min fixture < 1 MB | M |
| **T12** | `doctor` + `selftest` | T4,T10 | `fp-cli/src/cmd/{doctor,selftest}.rs` | golden output for full/degraded/blind hosts; **states the cost of each missing capability**; exit 3 when blind | S |
| **T13** | Pi hardware checks | T5,T6 | `fp-linux/src/{swap,thermal,mmc,kmsg}.rs`, `detect/hw.rs` | `swap_on_sd`, `thermal_throttle`, `undervoltage` bitmask, `remount_readonly`, `mmc_errors`; each with a negative test | M |
| **T14** | JSONL sink + redaction | T10 | `fp-agent/src/sink/jsonl.rs`, `redact.rs` | tier 0 contains no paths; tier 3 contains full evidence; no partial lines; bounded queue; `gap` marker on overflow | M |
| **T15** | `watch` TUI | T12 | `fp-cli/src/tui.rs` (feature `tui`) | renders at 80x24 and 200x50 without panic; **ribbon + one sentence**; default build unaffected | M |
| **T16** | Packaging + release | T12 | `systemd/`, `install.sh`, CI release job | `systemd-analyze security` passes; install idempotent; binary < 5 MB; runs unprivileged with caps | M |
| **T17** | Truth-audit soak harness | T14 | `xtask/`, `scripts/soak.sh` | collects 7 days of findings, samples 50, emits a review template; no automated judgment | S |
| **T18** | Episodes + alert budget | T10,T14 | `fp-core/src/episode.rs`, `fp-agent/src/sink/webhook.rs` | 10k findings -> at most N notifications/day; one per episode; cooldown honoured | M |
| **T19** | `explain` | T7 | `fp-cli/src/cmd/explain.rs` | prints evidence and arithmetic; unknown id -> exit 2 with a useful message | S |

---

## Task cards (the first four)

### T0 — Workspace, lints, CI, gates

**Goal:** an empty but fully gated workspace. No product code.

**Files:** root `Cargo.toml` (workspace, `resolver = "2"`, `[workspace.lints]`), `rust-toolchain.toml`,
`.rustfmt.toml`, `deny.toml`, `crates/{fp-core,fp-linux,fp-agent,fp-cli}/Cargo.toml` + `src/lib.rs`
(or `main.rs` for cli), `xtask/src/main.rs`, `.github/workflows/ci.yml`.

**Tests (write first):**
1. `cargo xtask verify` exists and exits 0 on the empty skeleton.
2. A test reads each crate root and asserts `#![forbid(unsafe_code)]` is present.
3. `xtask` asserts `fp-agent` has **zero** entries under `[dependencies]`.

**Do not:** add `serde`, `clap`, or any dependency; create parsers, detectors, or modules; create
`fp-trace` (it is v1.x).

**Acceptance:** CI green in under 2 minutes; `cargo xtask verify` runs fmt, clippy `-D warnings`,
test, dependency count, and a binary-size report.

---

### T1 — ids, time, interner

**Goal:** the deterministic primitives every finding depends on.

**Files:** `crates/fp-core/src/{ids,time,intern}.rs`, `crates/fp-core/src/lib.rs`.

**API (exact):**

```rust
pub fn fnv1a64(bytes: &[u8]) -> u64;

impl FindingId {
    pub fn compute(host_id: &str, boot_id: &str, detector_id: &str,
                   detector_version: u32, window_start_ns: u64,
                   culprit_key: &str) -> FindingId;
    pub fn to_hex(&self) -> String;   // 16 lowercase hex chars
}

impl Interner {
    pub fn intern(&mut self, s: &str) -> u32;
    pub fn resolve(&self, id: u32) -> &str;
    pub fn len(&self) -> usize;
    pub fn evictions(&self) -> u64;
}

pub struct MonoTime(u64);
pub struct ClockProvenance { pub mono_start_ns: u64, pub mono_end_ns: u64,
                             pub wall_start_s: i64, pub wall_end_s: i64,
                             pub skew_bound_ms: u32 }
```

**Tests (write first) — these vectors are normative:**

| Case | host | boot | detector | ver | window_start_ns | culprit_key | expected id |
| --- | --- | --- | --- | --- | --- | --- | --- |
| unit culprit | `pi-lab-3` | `b-1` | `io.small_write_storm` | 1 | `1000000000` | `unit:traefik.service` | `7e6dce5610127c31` |
| no culprit | `pi-lab-3` | `b-1` | `io.full_stall` | 1 | `1000000000` | `` | `ef875ba84885d4cf` |
| non-ASCII | `pi-lab-3` | `b-1` | `io.full_stall` | 1 | `1700000000` | `unit:träfik-日本語.service` | `6814b6e5b9551e2d` |
| device | `server-01` | `b-9` | `io.device_stall` | 2 | `42` | `device:mmcblk0p2` | `ddecfccecbde215a` |

Also: same inputs twice → same id; different `boot_id` → different id; interner returns the same id
for the same string, enforces its cap, and increments `evictions`; `skew_bound_ms` defaults to 1000
when unknown.

**Do not:** use `DefaultHasher`, `RandomState`, UUIDs, or floats. Do not make the id depend on map
iteration order.

**Acceptance:** the four vectors pass exactly; `cargo clippy -D warnings` clean.

---

### T4 — PSI parser

**Goal:** parse `/proc/pressure/*` without allocating after warmup and without panicking on garbage.

**Files:** `crates/fp-linux/src/psi.rs`.

**API (exact):**

```rust
pub struct PsiValues { pub avg10_bp: u32, pub avg60_bp: u32, pub avg300_bp: u32, pub total_us: u64 }
pub struct PsiLine { pub some: PsiValues, pub full: PsiValues }
pub fn parse(input: &str) -> Result<PsiLine, PsiError>;
pub fn read(path: &Path) -> Result<PsiLine, ProbeError>;  // NotFound -> ProbeError::Missing
```

Percentages are **basis points** (`61.23% -> 6123`). No floats.

**Tests (write first):**
1. Canonical two-line input parses to exact values.
2. Missing `full` line → `PsiError::MissingFull`.
3. Truncated line (`total=381`) → typed error, no panic.
4. Unknown key (`foo=1`) ignored.
5. `read()` on a nonexistent path → `ProbeError::Missing`, **not** an error that stops the daemon.
6. A helper computes a delta and returns `None` when `total` decreases (rollover).
7. `proptest`: arbitrary UTF-8 never panics.

**Do not:** open files inside `parse`; return `String` errors; use `f32`; allocate per call beyond one
reused buffer.

**Acceptance:** all tests pass, including the property test.

---

### T7 — `io.full_stall` (the first real finding)

**Goal:** turn PSI deltas into one honest finding. This is the reference implementation every other
detector copies.

**Files:** `crates/fp-core/src/detect/io.rs`, `crates/fp-core/src/detect/mod.rs`.

**API (exact):**

```rust
pub struct FullStall { pub threshold_ms: u64 }

impl Detector for FullStall {
    fn id(&self) -> &'static str { "io.full_stall" }
    fn version(&self) -> u32 { 1 }
    fn evaluate(&mut self, w: &WindowView<'_>, out: &mut Findings);
}
```

**Computation:** `stalled_ms = (end.total_us - start.total_us) / 1000`; `window_ms` from the window
timestamps; fire when `stalled_ms >= threshold_ms` and `window_ms > 0`.

**Tests (write first):**
1. `1_000_000_000us -> 1_012_400_000us` over `300_000ms` yields `stalled_ms = 12_400`,
   `window_ms = 300_000`, `coverage_pct = 100`.
2. **Tick-rate independence:** the same total delta sampled at 1 Hz and at 10 Hz yields identical
   `stalled_ms` (property test).
3. Below threshold → no finding.
4. Evidence contains exactly two `psi` items whose `value` includes `total=`.
5. `culprit: null`, `confidence: observed`, `impact: self_stalled`, `role_hint: unknown`.
6. Golden JSON snapshot (fixed host/boot/window, so the id is stable).

**Do not:** use `avg10`/`avg60` as severity; divide by sample count; invent a culprit; emit when
`window_ms == 0`; set `traced`.

**Acceptance:** all six tests pass; `cargo test -p fp-core detect::io` is under 1 s.

---

## How to hand this to an agent

1. Create a branch named `task/T4-psi-parser`.
2. Give the agent exactly three inputs: **`AGENTS.md`**, **`docs/SCHEMA.md`**, and **one task card**.
   Not the whole task table, not `VISION.md`, not `ARCHITECTURE.md`.
3. Require the five-line restatement before it writes code. If the restatement is wrong, stop and fix
   the card — that is cheaper than fixing the diff.
4. Review with the seven-question checklist in `AGENTS.md`.
5. Merge only when `cargo xtask verify` is green and the acceptance evidence is pasted into the PR.
6. At the end of the task, have it write the next card. Then start a fresh agent context.

**Anti-pattern to avoid:** *"build FrictionPoint."* An agent given the whole vision will build a
plausible, confident, wrong version of all of it. An agent given one card with a failing test builds
the thing that passes.

---

## Evaluation harness (the actual point of the project)

Success is not "the code exists." It is "the code is correct and the model behaved."

### Held-out fixtures

Keep at least three fixtures the agent never sees: one clean baseline, one storage stall, one
misleading case where the busy unit is the **victim**. They live in `fixtures/heldout/`, are excluded
from agent context by instruction, and are scored by `xtask task score`.

### Scorecard per task

| Metric | Why it is measured |
| --- | --- |
| Task pass (acceptance tests) | The floor, not the goal |
| Held-out culprit precision@1 | Prevents overfitting to the fixture it was given |
| `clean_baseline` findings | Must be 0; one false positive erodes trust |
| Budget pass (CPU/RSS/alloc) | The agent will happily blow the budget without a gate |
| Dependency delta | Must be 0; catches "just one crate" |
| `unsafe` delta | Must be 0 |
| Diff size / files touched | Catches scope creep and drive-by refactors |
| Tests added (and negative tests) | Quality signal, not quantity |
| Human review notes | The only judgment that catches wrong-but-passing |
| Wall time / tokens | Operational cost for planning the next task |

### Pilot protocol

Run three agents on **T4** (parser, mechanical) and three on **T7** (detector, judgment). Compare
conformance, not speed. The harness is good when independent runs produce diffs that a human would
merge without edits. If they do not, the card is under-specified — fix the card, not the agent.

### What we are testing

Whether a strong model can follow a **narrow, testable contract** in a codebase with strict
invariants: no deps, no `unsafe`, no async, zero-alloc hot path, honest confidence, evidence-backed
output. That is a much better proxy for real software work than "write a CRUD app."
