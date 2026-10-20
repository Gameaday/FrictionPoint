# Lessons from `stallwatch` (the closest prior art)

`varbees/stallwatch` is the nearest thing to FrictionPoint that exists: Rust, per-cgroup PSI, static
musl binaries, `doctor`, Varlink, a DaemonSet, ~46 commits, archived in October 2026 with **2 stars**.
It solved the same core problem and then documented, with numbers, what broke in production.

That post-mortem is worth more to us than the code. This document records what we copy, what we fix,
what they closed that we should also close, and the concrete changes to our plan.

> Their framing of the product: *"a smoke alarm and a black box"* — silent and free while the machine
> is healthy, already knows the answer when it stalls. That is better than "watcher". Steal it.

---

## 1. Their 30-day post-mortem, and what it means for us

They ran it continuously on a daily driver for a month, then read the **~520,000 incidents** it had
written. Three failures were invisible to a 132-test suite that passed the entire time.

| Their failure | Evidence from the repo | Our countermeasure | How we prove it |
| --- | --- | --- | --- |
| **The cause branch never executed** | `Role::Cause` required `/proc/<pid>/io`, unreadable for processes you don't own — and the processes that stall a machine are root daemons and kernel threads. **Zero causes named in 29 days**, while every report looked complete. | cgroup `io.stat` is *world-readable* and becomes the **primary** attribution source; per-PID data is only a refinement. Ladder: unit (unprivileged) → process (root) → file (trace). | A **non-root** integration test asserting a culprit is named. Release gate. |
| **Reported the sampling window as the freeze duration** | `capture_ms = 400`; only **3 of ~520,000** incidents ever reported > 1 s, because the number printed was the capture length. | Severity always from **cumulative PSI counters over the report window**, never from a sample interval. `frozen_ms` is explicitly a **lower bound** when coverage < 100%. | Property test: changing the sampling interval must not change reported severity. |
| **The notification policy was never tested against reality** | Projected **2,489 notifications/month**; one 400 ms capture of a *continuous* condition; named a systemd unit rather than an app; accused the stalled unit instead of the one moving data. Rebuilt around **episodes → 38/month**. | Episodes + cooldown + per-day budget are required **before** the first alert sink. Cause vs victim decided by **request share vs blocked-time share**, not by who looks busy. | Alert-budget test: 10k findings ≤ N notifications/day. Fixture where the busy unit is the victim. |
| **Nothing checked whether the evidence existed** | No capability probing for a month; the bug survived because nothing asked "can I actually read this?" | `doctor` states every missing capability **and what it costs** ("no per-PID attribution: culprits limited to units; confidence capped at `correlated`"). Daemon logs the same at startup. | Golden `doctor` output per environment; startup log assertion. |
| **Tests asserted against fabricated data** | 132 tests green throughout. *"A test suite that only asserts against data the test itself fabricated proves self-consistency, not truth."* | Fixtures must be **real captures** from real hosts, plus a **truth audit**: sample 50 emitted findings after a 7-day soak and verify each by hand. | The soak + audit is an acceptance criterion, not a nicety. |
| **Per-cgroup PSI triggers need privilege** | System-wide triggers register unprivileged; per-cgroup returns `EACCES`, so capture wakes system-wide and attributes afterwards. | Polling is the unprivileged default. Triggers are an **optional fast-wake mode** with a polling fallback, and the daemon states which mode it is in. | Capability test + mode reported in `doctor`. |

The deepest lesson is row five. Our fixture/replay strategy is necessary but **not sufficient** — it is
exactly the self-consistency they had. Truth comes from reading what the tool actually wrote on a real
machine over time.

---

## 2. What to copy (good engineering, cheap for us)

| Practice | Why it worked | Where it lands |
| --- | --- | --- |
| **"Smoke alarm and a black box"** | One sentence that explains the product without a diagram; implicitly contrasts with dashboards | README / launch copy |
| **Zero cost while healthy** | PSI triggers mean no timer at all; nothing is spent until something is wrong | Optional trigger mode (v0.3) with polling fallback |
| **`doctor` with cost-of-missing** | Turns "degraded" into an actionable fact; prevents silent blindness | v0.1, upgraded from presence-only |
| **Every claim shows its `/sys` path** | Lets the user re-read the kernel and check the arithmetic | Already our `EvidenceItem`; keep it mandatory |
| **Episodes before notifications** | Turned 2,489/month into 38/month with better signal | v0.3 gate before alerts |
| **watch / explain / advise / act** | A clean permission ladder; `advise` prints the command and does not run it | Adopt the vocabulary in VISION §5 |
| **`config --explain`** | Prints the resolved value *and which file won* | CLI v0.3; we already have layered config |
| **Shape *and* sentence on every screen** | A ribbon answers "is something wrong, and when"; a sentence answers "what, and whose fault" | TUI spec, v0.3 |
| **Zero-dep engine, UI deps quarantined** | Keeps the engine embeddable and auditable; the TUI pays its own way | Already our feature split; publish the agent dep count |
| **Static musl + checksums + `install.sh` + crates.io** | Installation that survives distro churn | v0.2 packaging |
| **Graceful archive** | *"Shipped and finished, not abandoned mid-way"* — final release, working installer | Add an exit policy to PLAN §14 |
| **Publish `fp-core` as a library** | Their C ABI ambition without the ABI: other tools consume detectors/findings as a crate | Post-1.0, near-zero cost |

---

## 3. What we must do better (their gaps are our wedge)

| Their limitation | Our answer |
| --- | --- |
| Unit-level attribution only | Unit -> process -> device -> file, with the confidence tier stated for each |
| No device-vs-workload discrimination | The four-story table: fsync storm vs dirty throttling vs journal vs dying card |
| No hardware diagnosis | `vcgencmd` undervoltage, `mmc` errors, card age, swap-on-SD |
| No cross-host causality | The hub + edges + roles seam already specified in `FLEET.md` |
| No real-capture corpus | `record`/`replay` with fixtures that must come from hardware |
| Byte-based reasoning | Request-share blame + blocked-time cross-check |
| No coverage denominator | `attributed %` on every finding; eviction counters |
| 400 ms capture window | Severity from cumulative counters over the report window |
| systemd names only | Unit -> application name mapping with fallbacks (OpenRC, Docker, K8s) |
| cgroup v2 only, no v1 fallback | Same requirement, but **degrade honestly** and say so instead of reporting "no stalls" |

---

## 4. Directions they closed, which we should also close

They did the research so we don't have to. Recording these prevents us re-opening them on a hunch.

| Direction | Why they closed it | Our position |
| --- | --- | --- |
| **Kubernetes/fleet PSI collection** | Kubernetes ships PSI natively at node, pod and container granularity (their research: stable and locked on since v1.36). *"The platform does natively what an exporter here would offer, better placed."* | **Do not build a PSI exporter.** M3 consumes native cluster metrics. Our hub's value is **causality and narrative across hosts**, not collection. Adjust the M3 scope accordingly. |
| **A desktop notifier product** | `psi-notify` (a Meta systemd maintainer) is packaged across Debian/Fedora/EPEL/Ubuntu/Arch: **242 stars after six years, no revenue**. btop 34k, bottom 14k, htop 8k — all free. | Never build a notifier. Emit MQTT/webhook/systemd-cat and let existing tools notify. Confirms our "export, don't dashboard" stance. |
| **A security pivot** | *"Attribution is the whole of a security product, and attribution without privilege is exactly what the constraints here forbid."* | Agreed. Falco/Tetragon/Sysdig own privileged attribution. We stay read-only diagnostics. |
| **eBPF "because it is fashionable"** | *"It needs root and kernel headers, and would cost the property that this runs anywhere as anyone."* | Already cut from v1.0. Adopt the stronger principle: **zero privilege to diagnose; privilege only to refine.** |
| **A query DSL before users** | They argued (with Wireshark) that a filter language is the highest-leverage surface — then shipped it before having an audience. | We skip the DSL until there is demand. `--json` + `jq` is the query language; typed, stable field names are the real enabler. |

---

## 5. What not to copy

- **400 ms capture windows.** Long enough to be cheap, too short to measure a freeze, and it produced a
  number that looked like a duration but wasn't.
- **Unit-only attribution as the ceiling.** It is the right *floor* (unprivileged, actionable), not the
  ceiling.
- **Notifications without episodes.** 2,489/month is how a useful tool gets muted forever.
- **`Role::Cause` gated behind data only root can read.** If the primary causal signal needs privilege,
  the feature is dead on arrival for most users — and it fails *silently*.
- **Building the TUI/DSL/ABI before distribution.** They got to a very sophisticated engine with zero
  audience. Depth is not reach.
- **Zero dependencies as an end in itself.** It is a means (embeddability, auditability). We accept
  three deps in the agent for correctness and call it a win.

What *is* worth copying from their build: `opt-level = "s"`, LTO, strip, a resolver-3 workspace, and a
README that leads with the symptom rather than the architecture.

---

## 6. Concrete changes to our plan

These are now requirements, not suggestions.

1. **New principle: zero privilege to diagnose; privilege only to refine.** PSI and cgroup `io.stat`
   are world-readable, so a complete unit-level diagnosis must work as an unprivileged user. Root /
   capabilities only add process- and file-level refinement. **CI runs the detector suite as a
   non-root user and asserts a culprit is named.** (PLAN §1, §9)
2. **`doctor` states the cost of each missing capability**, not just its absence, and the daemon logs
   the same summary at startup. "No per-PID attribution: culprits limited to units; confidence capped
   at `correlated`." (PLAN §8, §14)
3. **Severity semantics are fixed by rule and test:** `frozen_ms` derives from cumulative PSI totals
   over the report window; it is a **lower bound** whenever coverage < 100%; a sample interval is never
   a duration. Property test: changing the tick rate must not change reported severity. (ARCHITECTURE §8)
4. **Episodes and an alert budget ship before any alert sink.** One notification per episode, a
   cooldown, and a hard per-day cap. Until that exists, `--sink webhook` is not released. (PLAN §11)
5. **Cause vs victim is explicit.** Request share answers "who sent the work"; `delayacct` blocked
   share answers "who was blocked". A unit with high blocked share and low request share is a
   **victim**, and the narrative must say so. Add a `victim_heavy` fixture. (ARCHITECTURE §8)
6. **Truth gate before v1.0:** fixtures must originate from real hardware captures; run a 7-day soak
   on three hosts; manually audit 50 emitted findings and record the miss rate. Fabricated fixtures are
   not acceptable evidence. (PLAN §11 acceptance criteria)
7. **Optional trigger-driven idle mode (v0.3).** Block on a system-wide PSI trigger instead of a 1 Hz
   timer, keep a low-rate context ring, and poll only while pressure is elevated. Polling remains the
   default because per-cgroup triggers need privilege and unprivileged trigger windows must be
   multiples of 2 s. The daemon reports which mode it is in. (ARCHITECTURE §6)
8. **TUI carries a shape and a sentence** (a stall ribbon plus one sentence), and `config --explain`
   prints resolved values and their source file. (PLAN §11)
9. **`act` mode, when it exists, is bounded, reversible, receipted and verified:** only cgroup knobs
   (`io.max`, `memory.high`), never kill/restart/sysctl/package installs; every action writes the prior
   value and the exact revert command; afterwards it measures whether the stall stopped. (VISION §5)
10. **Publish `fp-core` as a library** for embedders (htop-like consumers). No C ABI, no plugin ABI.
11. **Read `/proc/pressure/irq` where the kernel provides it.** They flagged IRQ storms as unread and
    they are directly relevant to audio/USB stutter. Probe for it; do not require it.
12. **Distribution assets are part of the build:** a reproducible `demo/demo.sh` (not just a GIF) and a
    public fixture corpus, so claims can be checked by strangers.
13. **Exit policy:** if development stops, publish a final tagged release, keep the installer working,
    and leave a fork guide. "Shipped and finished, not abandoned mid-way."

---

## 7. The uncomfortable strategic lesson

Their own words, in `DIRECTION.md`:

> *"Every repository in this portfolio has zero stars. The engineering is not the bottleneck;
> distribution is."*

They shipped a zero-dependency engine, a filter language, a rules engine, Varlink, a DaemonSet, static
binaries, a website, and a demo — and ended with 2 stars. Meanwhile btop has 34k, bottom 14k, htop 8k,
and a systemd maintainer's PSI notifier has 242 after six years. None of them make money; all of them
out-distribute a technically superior niche tool.

Implications for FrictionPoint:

- **Do not try to out-engineer the field.** We already match or beat the prior art on architecture.
  That is table stakes, not differentiation.
- **The differentiators are persona and specificity:** the Pi that goes read-only, the homelab where
  ingress dies because of a log flood, the SD card that stalls. Monitor tools are generic; we are
  specific, and specific stories get repeated.
- **Trust assets are distribution:** the `clean_baseline` false-positive gate, reproducible fixtures,
  and a public truth audit are marketing that happens to be engineering.
- **One optional reach play:** their `/proc/pressure` namespacing observation (not namespaced, not in
  containerd's `MaskedPaths`, so a limited container reads host counters) is a two-hour PoC and a short
  writeup. It is adjacent, honest, and reaches the exact homelab/security audience we want.

---

## 8. The one-paragraph version

`stallwatch` proves the mechanism (PSI + cgroup attribution + static binary + `doctor`) and proves the
product gap (unit-level only, Linux-only, no hardware story, no fleet, 2 stars). Its post-mortem proves
the two traps we were most likely to fall into: **an attribution path that silently requires root**, and
**tests that validate self-consistency instead of truth**. We adopt its `doctor` rigour, its episode
discipline, its mode ladder, and its zero-cost-while-healthy goal — and we differentiate on storage
vs hardware discrimination, real-capture fixtures, confidence tiers, and cross-host causality. Its
stars are a warning: the architecture is not the hard part. Being believed is.
