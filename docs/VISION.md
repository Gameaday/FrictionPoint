# FrictionPoint — Vision

> Find the seconds your system spent frozen, and name who caused them.

## 1. The core insight

Every mainstream performance tool measures **utilization**: CPU %, memory %, disk %,
network throughput. Utilization answers "how hard is the hardware working?"

Users do not experience utilization. Users experience **unavailability** — the cursor that
won't move, the SSH prompt that hangs, the stream that stutters, the download that dies.

These are different quantities, and they diverge exactly when things go wrong:

| What the meter says | What the user experiences | What actually happened |
| --- | --- | --- |
| CPU 15%, RAM 40% free | Box is unusable for 12s | 41 tasks in D-state waiting on one SD card |
| Disk 30% utilized | Editor freezes every keystroke | Garbage collection stalls inside the card |
| Network 2% | Can't reach any service | Ingress daemon starved by a log flood |
| GPU 60% | Camera drops off, everything resets | USB bus renegotiated after a bandwidth cliff |

So the headline metric of this project is not a percentage. It is **lost milliseconds**:
how long the system was not able to make progress, and on what.

### Corollary: bytes are not friction. Requests are.

A 4 GiB sequential copy and 40,000 tiny 4 KiB fsync'd log appends can move similar byte
counts while having wildly different latency consequences. Every tool that blames the process
with the most bytes blames the wrong process. FrictionPoint attributes by **request count,
request size, and blocked time** — because latency is charged per request, not per byte.

This single choice is the difference between "ntpd is using the disk" and "the agent that
writes 1,900 four-kilobyte records per second is why your SSH session hangs."

---

## 2. Where the idea came from (and what it generalizes to)

Three unrelated-feeling incidents, one shared structure: **a local resource contention
cascades into a system-wide symptom that looks like something else entirely.**

### The log-flood cascade (homelab)

A fleet of Pis, all shipping logs to one central Pi. That Pi also runs Traefik — the network's
ingress. One unrelated Pi gets updated, its log volume jumps, the central Pi's SD card
saturates, and **the server disappears from the internet**. Nothing crashed. Every component
reported healthy. Diagnosing it required guessing across machines because no tool connected
"writes on Pi A" to "ingress dead on Pi B."

FrictionPoint's job: produce the sentence that would have ended that investigation in thirty
seconds — *"ingress died because traefik's box spent 68% of the last 10 minutes in full I/O
stall, driven by small writes from the log receiver, which is being fed by a 40x log-volume
increase from host X."*

### The SD card wall (embedded)

A Pi becomes read-only. Or worse, it stays writable but stalls unpredictably as the card's
controller does wear-leveling and garbage collection. The user cannot tell:

- Is this a **card** problem (buy an SSD / replace a dying card)?
- A **workload** problem (an app writing far more than it should)?
- A **configuration** problem (swap on SD, `atime`, journal commit interval)?
- A **power** problem (brownout causing controller resets)?

These have completely different remedies, and the difference is visible in the data — if
anyone correlates it. This is Phase 1.

### The peripheral cliff (streaming / gaming)

A webcam, capture card, audio interface, and high-polling mouse share a hub. Everything works
until the camera needs *slightly* more bandwidth than the bus can grant, and the entire bus
resets. From the user's side: "my camera dropped off randomly and my audio got crunchy."
Mystifying, but mechanically exposable — USB topology, per-device bandwidth, and reset/error
counters are all queryable.

### The post-sleep zombie (Windows)

After sleep/lock, a client's network stack or storage handles fail to re-initialize. The
symptom surfaces as "downloads fail with network errors," which sends everyone hunting the
network. The actual event is a driver/service failing a power-transition contract.

**Generalized failure classes FrictionPoint should eventually name:**

1. **Resource starvation** — something needed a resource and couldn't get it (I/O, CPU, memory,
   bus bandwidth).
2. **Latency amplification** — a small number of slow operations block a large number of
   dependents (journal commits, fsync storms, lock contention).
3. **Cascade / shared-fate** — unrelated components fail together because they share a
   bottleneck (co-tenanted ingress + logging, shared USB hub, shared SD card).
4. **State-transition failure** — things break at boundaries: sleep/wake, hotplug, driver
   update, network roam, cgroup move.
5. **Hardware degradation** — the device is failing slowly (card wear, cable/PSU marginality,
   thermal throttling, USB signal integrity).
6. **Misconfiguration with delayed cost** — swap on SD, unbounded logs, tiny `tmpfs`,
   `atime` on a write-heavy mount.

Each class has distinct signatures and distinct remedies. The long-term product is a
**catalogue of friction patterns**, not a graph of counters.

---

## 3. Who this is for

Ordered by urgency, not market size:

1. **The homelabber with a fleet of small boxes.** Wants to know which host is sick and why,
   before the whole stack topples. Needs a single static binary, near-zero overhead, and
   answers that don't require a PhD in kernel tracing.
2. **The SBC tinkerer.** Pi, Armbian, Rockchip, ODROID, router boards. Constrained storage and
   power. Needs the tool to fit in the budget it's measuring.
3. **The streamer / power user with many peripherals.** Wants to know why frames drop, audio
   crackles, or devices vanish — and which USB port/hub to move things to.
4. **The server operator.** Wants stall attribution per container/unit without running a
   heavy APM stack, and wants it exported to the observability system they already have.
5. **The everyday user.** "My PC is slow" is the most common tech-support sentence on earth,
   and current answers are "reinstall Windows" or "buy more RAM."
6. **Android users.** Storage speed and memory headroom are opaque and shifting. Even
   directional, per-app I/O attribution would be new information for most people.

The through-line: **help people get more out of the hardware they already own.** Most "slow"
machines are not underpowered; they are badly scheduled, badly configured, or blocked behind
one pathological workload.

---

## 4. The first five minutes (the product's most important surface)

Monitoring tools fail at onboarding: they show a wall of charts and leave the interpretation
to the user. FrictionPoint's first run should do the opposite.

1. `frictionpoint baseline --for 60s`
2. Produces a short, opinionated report:
   - "Your system lost **4.2 seconds** to stalls in the last minute (7% of wall clock)."
   - "**Two** patterns explain 91% of it." (named, ranked)
   - "One is cheap to fix: swap is on the SD card. That alone accounts for 2.8s."
   - "One is a hardware suspicion: card latency spikes at low queue depth."
     (confidence: correlated)
3. Every claim links to `frictionpoint explain <finding-id>` -> the raw numbers and paths.
4. Nothing is written to disk except the report the user asked for.

If the first run can't find anything, it says so plainly and explains what it *would* watch
for. A monitor that manufactures findings to look useful is worse than no monitor.

---

## 5. Product shape: one core, optional limbs

Scope creep kills this category. The split below is the guardrail.

| Layer | What it is | Privileges | Ships as |
| --- | --- | --- | --- |
| **Core** | Collectors, detectors, attribution, narratives | Read-only, mostly unprivileged | `frictionpoint` (one static binary) |
| **Recipes** | Declarative TOML advice keyed to finding types | None (pure data) | Community-contributed, bundled or fetched |
| **Trace** | Optional deep per-request tracing (eBPF/ETW) | Root/admin | Feature flag on the same binary |
| **Fix** | Applies remedies: config edits, driver/firmware links | Elevated, explicit | Separate binary, dry-run first |
| **Fleet** | Optional hub + pane; causal chains across hosts | Network | Separate tier (hub subcommand + optional UI) |

Rules that keep the core slim:

- The core **never downloads or installs anything**. It emits a recommendation with a stable
  identifier (`pi.swap_on_sd`) and lets a recipe or the Fix companion act.
- Remediation is **never automatic by default** and always prints the exact command it would
  run. Trust is the product; a tool that silently edits `fstab` loses it permanently.
- Update/driver/firmware logic is explicitly **out of the core**. It's a different problem with
  different failure modes (supply chain, rollback, hardware bricking) and deserves its own
  threat model.
- Integrations (Prometheus, OTLP, Home Assistant, webhooks) are **exporters**, not features in
  the detector path.

---

## 6. Trust model: three tiers of knowing

The fastest way to lose a user is to confidently name the wrong culprit. So every finding
declares how it knows:

| Tier | Meaning | Typical source | Language used |
| --- | --- | --- | --- |
| `observed` | Directly measured fact | PSI, diskstats, thermal | "The system lost 12.4s." |
| `correlated` | Statistically dominant during the stall | cgroup `io.stat` share, per-pid counters | "X accounts for 74% of write requests in this window." |
| `traced` | Causally linked per request | eBPF / ETW stack attribution | "X issued the 1,842 requests that blocked for 11.9s." |

The UI must never render `correlated` as certainty. "70% of calls during slow downs came from
Y" is a *correlation* claim, and is presented as such — with the sample size.

This also gives a natural upgrade path: the same finding gets *promoted* from correlated to
traced when the user opts into deeper instrumentation, without changing the vocabulary.

---

## 7. Non-goals (deliberate)

- **Not a dashboard.** No pretty graphs as the primary artifact. Export metrics if you want
  them; we won't compete with Grafana.
- **Not an APM / distributed tracing product.** No agent-per-service instrumentation, no
  request tracing inside your app.
- **Not an antivirus / security scanner.** Vulnerability scanning was considered and cut:
  different domain, different trust requirements.
- **Not a benchmarking tool.** We report what your system is doing, not what a synthetic load
  says it could do.
- **Not a cloud service.** Optional fleet aggregation, never a mandatory account.
- **Not cross-platform at launch.** One excellent Linux/SBC implementation first.

---

## 8. How we'll know it's working

- A user resolves a real incident using only the report (qualitative, tracked in discussions).
- Median time-to-culprit on a seeded fault-injection scenario: **< 60 seconds** of reading.
- Overhead on a Pi Zero 2 W: **< 1% CPU**, **< 15 MiB RSS**, **0 bytes written** at idle.
- False-culprit rate on the labelled scenario corpus stays **< 10%** as detectors are added.
- Someone runs it on a machine that feels fine and still learns something true and actionable.

---

## 9. Honest risks

- **Correlation isn't causation.** Mitigated by confidence tiers, evidence, and optional
  tracing — not by pretending.
- **Kernel variance is brutal.** PSI may be compiled out or disabled (`psi=1`), cgroup v1
  systems exist, containers see their own cgroup, non-systemd distros name things differently.
  A `frictionpoint doctor` must always explain what it *can't* see on this host.
- **Narratives can be wrong.** Templates must be conservative, and every finding must degrade
  to raw evidence rather than inventing a story.
- **Jack of all trades, master of none.** The platform roadmap is where this project would
  die. Sequence is enforced: Pi -> servers -> Windows -> Android.
- **The "vitamin" problem.** Most people don't know they need this until something breaks.
  Mitigation: the baseline report has to be interesting on a healthy machine, and the
  incident-replay corpus has to become the marketing.
- **Hardware truth is sometimes unavailable.** SD cards expose no SMART, no NCQ, and lie about
  utilization. We must prefer kernel-observed latency and say "the card is stalling" rather
  than pretend to know the card's internal state.

---

## 10. Name

A **friction point** is where two surfaces stop moving freely — the place where motion is
converted into heat instead of progress. It's also the clutch term for the exact moment
engagement happens. That's the product: find the point where your system's forward motion is
being turned into waste, and tell someone which surface is responsible.

Short, pronounceable, searchable, and not another `sys-`/`perf-` compound. Verify crates.io,
npm, and domain availability before public launch.
