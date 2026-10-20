# Areas — vertical delivery, not metric checklists

The unit of delivery is an **area**: one question a user asks, answered end-to-end from kernel signal
to a sentence, with evidence and an explicit "I can't see that" path.

A **metric** is an ingredient. A **detector** is a claim. An **area** is a finished answer.

Why not build metric-by-metric: a thin check ("io.wait > 20% → warn") is a threshold with no
attribution, no evidence, and no negative test. It fires on healthy machines, misses the real cause,
and teaches users to ignore the tool. Ten of those are worse than one complete area.

---

## 1. What counts as an area

Every area must ship all seven:

| Element | Requirement |
| --- | --- |
| **Question** | One sentence a user would actually say out loud |
| **Severity** | Lost time from cumulative counters, with coverage |
| **Attribution** | Culprit at a stated granularity, or explicitly `null` with a reason |
| **Evidence** | The exact paths and values, re-readable by the user |
| **Verdict** | One sentence, deterministic template, hedged when correlated |
| **Advice** | Stable advice ids, printed verbatim, never auto-applied |
| **Proof** | Fixture from real hardware + **negative test** on `clean_baseline` + a red-team case |

An area that cannot attribute must still ship: it says *"I can see the stall but not the cause, and
here is what would be needed to see it"* (privilege, trace, kernel feature). That is a diagnosis, not
a failure.

---

## 2. Anti-thin-check rules

1. **No detector whose entire logic is `metric > threshold`.** It must attribute, or explicitly refuse.
2. **No probe without a consumer.** A metric added "for later" is deleted.
3. **No detector without a fixture and a negative test.**
4. **No area without a red-team fixture**: a plausible-but-wrong culprit the tool must not blame.
5. **No area is "done" while it can fire on `clean_baseline`.**
6. **Depth before breadth.** Do not start area N+1 while area N can be fooled by its red-team case.
7. **Gaps are declared, not hidden.** Every area reports `coverage_pct`; unmeasured time is visible.
8. **One area per milestone.** The release is the area, not a collection of metrics.

---

## 3. The areas

Ordered by user value × data availability × reuse of the attribution engine.
**v1.0 ships A1 + A2.** A3-A6 are v1.1-v1.4; A7 is post-1.0.

### A0 — Foundations (not shippable)

**Question:** none. **Purpose:** the pipeline and the gates.

Workspace, ids/time/interner/series/histogram, parsers (PSI, diskstats, vmstat, procfs, cgroup),
record/replay, `doctor`, `selftest`, JSONL sink, packaging. No findings required.

**This is the only place a probe may exist without a detector**, and only for the files listed.

**Exit:** replay deterministic; `doctor` accurate on three hosts; zero allocations per tick;
`clean_baseline` runs end-to-end and emits nothing.

### A1 — "The box froze. Was it I/O, and who did it?"

The founding story and the engine every later area reuses.

| | |
| --- | --- |
| Signals | PSI `io` some/full/total, `diskstats` in-flight and service time, cgroup `io.stat`, `delayacct` |
| Detectors | `io.full_stall` (severity), `io.small_write_storm` (culprit), `io.device_stall` (no culprit / low queue), `unknown_friction` (residual) |
| Fixtures | `clean_baseline`, `sd_fsync_storm`, `sd_gc_stall`, `victim_heavy` |
| Exit | storm names the unit; GC stall names the device with `culprit: null`; victim case names the writer, not the stalled unit; baseline silent; `why` answers in one sentence |
| Not covered | memory, CPU, power, config, remote waits |

### A2 — "Is the card, the power, or the thermal envelope failing?" (v1.0)

Cheap, high-value, and the distinction between "fix the app" and "replace the hardware".

| | |
| --- | --- |
| Signals | `vcgencmd get_throttled`, thermal zones, `mmc` errors, mount flags, eMMC `life_time`, SD date |
| Detectors | `hw.undervoltage`, `hw.thermal_throttle`, `hw.mmc_errors`, `io.remount_readonly`, `mem.swap_on_sd`, `hw.card_wear` (advisory) |
| Fixtures | undervoltage bitmask, `remount_ro` (loop-mounted image only), `mmc` log capture, thermal soak, `swap_on_sd` |
| Exit | separates hardware/power from workload; `remount_ro` advises backup before anything else; baseline silent |
| Not covered | NVMe SMART, battery health |

### A3 — "Is a setting making this worse?" (v1.1)

| | |
| --- | --- |
| Signals | `/proc/swaps`, `/proc/mounts`, `/proc/sys/vm/*`, jbd2 stats, per-file write growth |
| Detectors | `fs.mount_misconfig`, `io.writeback_throttle`, `io.journal_serialization`, `fs.capacity` |
| Fixtures | `dirty_throttle`, `journal_commit`, `mount_misconfig`, `fs_full` |
| Exit | every finding carries a concrete, reversible command; nothing is applied; baseline silent |
| Not covered | application configuration, kernel tuning beyond these knobs |

### A4 — "Am I out of memory, or thrashing?" (v1.2)

| | |
| --- | --- |
| Signals | PSI `memory`, `vmstat` reclaim, `MemAvailable`, zram, cgroup `memory.events`, OOM events |
| Detectors | `mem.reclaim_pressure`, `mem.cgroup_limit`, `mem.oom_event` |
| Fixtures | cgroup memory limit, zram pressure, OOM inside a cgroup |
| Exit | distinguishes page-cache churn from real pressure; names the cgroup; baseline silent |
| Not covered | leak detection over days, THP behaviour |

### A5 — "Is something starving for CPU?" (v1.3)

| | |
| --- | --- |
| Signals | PSI `cpu`, `schedstat` run-queue wait, context switches, cgroup `cpu.stat` throttling, steal |
| Detectors | `cpu.runqueue_delay`, `cpu.ctx_switch_storm`, `cpu.cgroup_throttle`, `cpu.steal` |
| Fixtures | container CPU limit, run-queue contention, synthetic steal |
| Exit | reports `nr_throttled`/`throttled_usec` when present; names the cgroup; baseline silent |
| Not covered | IRQ affinity, real-time scheduling |

### A6 — "Am I waiting on something that isn't local?" (v1.4)

| | |
| --- | --- |
| Signals | D-state census with `wchan`, `/proc/net/softnet_stat`, retransmits, FUSE waiting requests |
| Detectors | `dep.remote_wait`, `io.fuse_stall`, `net.queue_drop`, `net.retransmit` (advisory) |
| Fixtures | sshfs/rclone stall, synced NFS wait, synthetic softnet drops |
| Exit | never blames the local disk for a remote wait; confidence is `correlated`; states what proof would require |
| Not covered | DNS, application HTTP, packet capture |

### A7 — "Is my peripheral flaky?" (v1.x Linux, post-1.0 Windows)

| | |
| --- | --- |
| Signals | USB topology, endpoint descriptors, reservation ratio, autosuspend state, resets, over-current |
| Detectors | `usb.reservation_exhausted`, `usb.autosuspend_reset`, `usb.overcurrent`, `usb.reset_storm` |
| Fixtures | recorded sysfs topology, reset log, reservation-heavy descriptor set |
| Exit | explains a camera dropout without packet capture; otherwise says `usbmon` (root) is required |
| Not covered | actual per-URB throughput without root; Android bandwidth does not exist |

---

## 4. Coverage matrix — every question is answered or explicitly declined

Gaps are fine. **Unlabelled gaps are not.** A user question that no area covers must have a named
alternative, and `doctor` must say so on the host.

| User question | Area | v1.0? | If not covered |
| --- | --- | --- | --- |
| "Why did my box freeze?" | A1 | **yes** | — |
| "Was it the disk, or an app?" | A1 | **yes** | — |
| "Is my SD card / eMMC dying?" | A2 | **yes** | — |
| "Is power or heat the problem?" | A2 | **yes** | — |
| "Did a setting cause this?" | A3 | **yes** | — |
| "Why can't my app write?" (full disk / inodes) | A3 (`fs.capacity`) | **yes** | advisory only, no auto-cleanup |
| "Am I out of memory?" | A4 | v1.1 | exits 3 with "memory area not implemented" |
| "Is something starving for CPU?" | A5 | v1.1 | exits 3 with a named gap |
| "Am I waiting on a remote dependency?" | A6 | partial | reports `RemoteWait` from `wchan`, confidence `correlated`; proof needs `trace` |
| "Is my USB device flaky?" | A7 | post-1.0 | "requires `usbmon` (root)" or the Windows artifact |
| "Which host caused this?" | Hub | post-1.0 | single-host answer only; `resource_keys` already recorded |
| "Why is the app itself slow?" | — | **declined** | profiler (`perf`, WPA, Perfetto) |
| "Which file is being hit?" | — | post-1.0 | `trace` tier (`fanotify`) |
| "Why is my GPU/audio glitching?" | — | **declined** | PresentMon, vendor tools |
| "Why is boot slow?" | — | **declined** | `systemd-analyze` |
| "Is the network down?" | — | **declined** | monitoring tools; we only explain local queueing |
| "Is it malware?" | — | **declined** | Falco / Tetragon / EDR |

---

## 5. Order and why

| Order | Area | Why here |
| --- | --- | --- |
| 1 | A0 | Nothing can be tested without fixtures and gates |
| 2 | **A1** | The founding story; builds the attribution engine every other area reuses |
| 3 | **A2** | Cheap, Pi-specific, and needed to separate "sick hardware" from "heavy app" |
| 4 | **A3** | Reuses A1 signals; delivers the highest-value advice (swap, dirty ratio, journal) |
| 5 | A4 | Independent resource; needed before claiming "whole system" coverage |
| 6 | A5 | Independent resource; cgroup throttling is a top-3 underserved signal |
| 7 | A6 | Needs the D-state/`wchan` census; where adjacent cases land |
| 8 | A7 | Different data model (topology/descriptors); mostly post-1.0 and partly Windows |

The order is **not** M1→M10. Mechanisms are not independent; user questions are. Areas are shaped like
the question, and each one reuses the same severity/attribution/evidence pipeline.

---

## 6. How gaps stay visible at runtime

1. `unknown_friction` fires whenever severity exceeds threshold and no detector attributes it, with the
   reason (coverage low, privilege missing, unsupported kernel feature).
2. Every finding carries `severity.coverage_pct`; the narrative says "lower bound" when < 100.
3. `doctor` lists each area and states its status: `active`, `degraded` (with cost), or `not implemented`.
4. `why` refuses to over-claim: "I can see 9.4s of I/O stall; I cannot attribute it without root".
5. The matrix above is the source of truth for claims, and `README` must not exceed it.

---

## 7. Area exit checklist (the release gate)

- [ ] 2-4 detectors, each with a spec comment, fixture, golden output, **and negative test**
- [ ] red-team fixture passes (the tool does not blame the plausible-but-wrong culprit)
- [ ] `clean_baseline` produces zero findings for this area
- [ ] `coverage_pct` populated; `unknown_friction` behaves correctly
- [ ] one `why` scenario in `demo/demo.sh` and one case study
- [ ] advice ids exist and print exact, reversible commands
- [ ] catalogue row added to `METRICS.md` and `ARCHITECTURE.md` §7
- [ ] `doctor` reports the area's status and the cost of anything missing
- [ ] all tasks in the area merged; CI green; budgets unchanged

An area that passes 8/9 is not done. It is a thin check with paperwork.
