# Launch Kit

Everything needed to create the GitHub project, fill the About box, and open the first issues.

---

## 1. GitHub About / description (< 350 characters)

**Recommended (283 chars):**

```text
Task Manager shows utilization. FrictionPoint shows unavailability: the seconds your system spent frozen, and who caused them. A lightweight Rust engine that correlates PSI stalls, disk latency, fsync storms, and peripheral dropouts into actionable root causes. Linux, Windows, SBCs.
```

**Alternative, keyword-heavier (299 chars):**

```text
Find out why your system hitches. FrictionPoint is a tiny Rust engine that turns kernel stall data - PSI, disk latency, fsync storms, context switches, USB dropouts - into plain-English root causes, naming the process, unit, and file responsible. Linux, Windows, and SBCs. Local-first, low overhead.
```

**Alternative, shortest (257 chars):**

```text
A tiny Rust diagnostic engine that finds the seconds your system spent frozen - and who caused them. Correlates disk stalls, fsync storms, memory pressure, and USB dropouts into plain-English root causes on Linux, Windows, and SBCs. No dashboards, no bloat.
```

All are under the 350-character limit. The recommended one leads with the contrast
(utilization vs unavailability), which is the whole thesis, and still contains the searchable
terms: Rust, PSI, disk latency, fsync, Linux, Windows, SBC.

**GitHub topics (max 20):**

```text
rust  linux  windows  performance  diagnostics  observability  raspberry-pi  sbc
ebpf  psi  io-latency  storage  bottleneck  troubleshooting  system-monitoring
homelab  etw  android  cli  low-overhead
```

**Website field:** leave empty until the first release, or point at the repo's `docs/`.

---

## 2. Repository settings

- License: MIT OR Apache-2.0 (dual), matching Rust ecosystem norms.
- Enable Discussions (this project will live on incident reports).
- Enable Issues with templates: `bug`, `false-positive`, `new-detector`, `platform-support`.
- `SECURITY.md`: this tool reads process names, paths, and kernel counters. State plainly:
  local-only, no telemetry, no network calls by default, and how to report a privilege bug.
- `CONTRIBUTING.md`: "one detector = one PR" with a required fixture and golden output.
- Branch protection on `main`: `fmt`, `clippy -D warnings`, `nextest`, `xtask perf`.
- Releases: static `aarch64-unknown-linux-musl`, `x86_64-unknown-linux-musl`, checksums,
  and a signed `install.sh`. No release until Phase 1 acceptance criteria pass.

---

## 3. The 15-second demo (build this before the launch post)

A GIF that shows one terminal and nothing else:

1. Start a 4 KiB `fsync` loop with `fio` in the background.
2. Run `frictionpoint baseline --for 30s`.
3. Output: `Storage stall: 9.7s of 30s frozen`, culprit `fio (PID 3121)`, mean request
   `4.0 KiB`, "not throughput-bound: queue depth 1.1".
4. Kill `fio`, re-run, output: `No friction above threshold.`

The before/after in one screen is the entire pitch. It also happens to be the
`sd_fsync_storm` fixture test.

---

## 4. Launch post draft

### Show HN (short)

> **Show HN: FrictionPoint - find the seconds your Linux box spent frozen, and who caused them**
>
> Every monitor tells me my Pi is at 15% CPU and 40% free RAM while SSH hangs. Utilization
> isn't what I experience; unavailability is.
>
> FrictionPoint reads kernel stall accounting (PSI), per-cgroup I/O request counts, and
> block-device latency, then names the process/unit responsible and explains whether it's a
> workload problem (fsync storm, dirty-page throttling) or a hardware problem (card stalling
> at low queue depth). Written in Rust, ~1% CPU idle, writes nothing to disk unless it finds
> something.
>
> Phase 1 is Raspberry Pi storage stalls because that's where it hurts most. Windows/Android
> are planned, not claimed.
>
> The most useful thing you can tell me: what's the *last* mysterious slowdown you gave up
> on?

### Homelab / selfhosted framing (story-first)

> My server vanished from the internet because a *different* Pi got updated.
>
> Logs from the whole fleet went to one central Pi that also ran Traefik. The update raised
> one host's log volume, the central Pi's SD card saturated, and ingress died. Nothing
> crashed. Nothing showed high CPU. Every dashboard was green.
>
> I spent an evening on this. The answer was "small writes on a different machine." No tool
> connects those dots, so I'm building one: FrictionPoint watches for stalls (not averages),
> attributes them to a process/unit/device, and tells you whether to fix your app, your
> config, or your hardware.

---

## 5. First issues (agent-sized, in dependency order)

| # | Issue | Size | Notes |
| --- | --- | --- | --- |
| 1 | Scaffold workspace, lints, CI, `cargo-deny` | S | Good first issue |
| 2 | PSI parser + `proptest` coverage | S | Must handle missing files, partial lines, rollover |
| 3 | `diskstats` parser + counter-rollover tests | S | Field indices are a classic bug source |
| 4 | `procfs` per-pid IO parser with PID-reuse guard | M | Use `starttime` to detect reuse |
| 5 | `Sample`/`Series` ring buffer + `loom` test | M | Only place `unsafe` is allowed |
| 6 | `io.full_stall` detector + fixture | S | First end-to-end finding |
| 7 | `io.small_write_storm` detector + blame math | M | Core differentiator |
| 8 | `io.device_stall` detector (low queue depth signature) | M | Needs negative test vs storm |
| 9 | `mem.swap_on_sd` + `hw.thermal_throttle` checks | S | Instant Pi wins |
| 10 | `frictionpoint doctor` capability report | S | Exit code 3 semantics |
| 11 | `record`/`replay` `.fpz` container | M | zstd + msgpack, versioned header |
| 12 | `explain` output + evidence rendering | S | Trust surface |
| 13 | JSONL exporter + Prometheus exporter | M | Keep out of detector path |
| 14 | Perf gate: counting allocator + `xtask perf` | M | Enforces the budget |
| 15 | arm64 musl release workflow + `install.sh` | S | Distro-free install |
| 16 | Narrative templates + `insta` snapshots | S | Wording is product |
| 17 | Detection-task harness (`xtask task new/score`) | M | For the agentic experiment |
| 18 | Docs: "what these numbers mean" | S | Explains PSI to normal humans |

---

## 6. Two-week starter plan

**Days 1-2** - workspace, lints, CI, license, this doc set, `cargo-deny`.

**Days 3-4** - PSI + diskstats + procfs parsers with tests. No detectors yet.

**Day 5** - `io.full_stall`, text report, exit codes. First honest output on a real machine.

**Days 6-7** - `record`/`replay` plus one real captured fixture from a Pi.

**Week 2** - `io.small_write_storm` + blame math, `doctor`, `explain`, perf gate, arm64 musl
release, the demo GIF. Tag `v0.1.0-alpha.1` only if the `clean_baseline` fixture produces zero
findings and the budgets are met.

---

## 7. Naming / identity checklist (before announcing)

- [ ] `crates.io` name `frictionpoint` (binary crate + `fp-*` library crates)
- [ ] npm/PyPI not needed, but squat the org handle if free
- [ ] `frictionpoint.dev` / `.io` availability
- [ ] Quick trademark search in software/observability classes
- [ ] GitHub org vs user: an org (`frictionpoint`) reads better once there are 3+ repos
      (core, recipes, fixtures)

---

## 8. What *not* to claim at launch

- Do not claim Windows or Android support. "Planned" is fine; screenshots are not.
- Do not claim causality without the confidence tier; say "correlated" where it is correlated.
- Do not promise automated fixes. The value at launch is the diagnosis.
- Do not call it AI-powered. It is deterministic, and that is a feature.
