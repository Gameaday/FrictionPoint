# Project portfolio — three products, one contract

**Decision:** FrictionPoint is **three independent projects**, not one codebase with `cfg` branches.
They share a *data contract*, not source code. Each must be useful on its own.

| Project | Repo | Purpose (one sentence) | Primary users | v1 must answer | Explicitly not in v1 |
| --- | --- | --- | --- | --- | --- |
| **FrictionPoint Linux** | this repo | Explain storage and pressure stalls on Linux/SBCs, naming the responsible unit, process, or device. | Homelabbers, Pi owners, server operators | "Is this box stalling, and is it the card, the config, or an app?" | Windows, Android, fleet hub, tracing |
| **FrictionPoint Windows** | `frictionpoint-win` | Explain hitches, dropouts, and failed recoveries after sleep using ETW. | Streamers, gamers, power users | "Why did my stream hitch / why did downloads fail after sleep / why did the camera drop?" | PSI, cgroups, Linux, WS **L** |
| **FrictionPoint Android** | `frictionpoint-android` (exploratory) | Explain per-app storage and memory pressure, unrooted where possible. | Android power users | "Which app is eating my storage or memory, and is it the device or the app?" | A daemon, root-only features, Play Store promises |
| **Contract** | `docs/SCHEMA.md` here, published as `frictionpoint-schema` | The `Finding` document, fixture format, kind registry, advice ids, test vectors. | All three | — | Behaviour of any kind |

## Why separate projects

1. **Context economy for agents.** A repository is a context window. A repo with three platforms
   teaches an agent that `cfg` branching is normal and that half the code is irrelevant. A 6k-line
   Linux repo is something a model can hold; a 25k-line cross-platform repo is not.
2. **Dependency isolation.** `windows-sys`/ETW has no business in a Pi binary; `libc` fanotify has no
   business on Windows. Separate lockfiles make the audit surface per-platform.
3. **Release cadence.** A Windows ETW schema change should not force a Pi release.
4. **Failure isolation.** A bug in one platform cannot regress the others.
5. **`cfg` sprawl is the leading cause of unmaintainable cross-platform Rust.** It is also the leading
   cause of agents producing plausible code that compiles nowhere.

The one thing that must not diverge is the **contract**. That is why `SCHEMA.md` outranks every other
document and why the test vectors are normative.

## What each project must own to be "useful"

A project is not allowed to start until it can answer its own question on day one of its alpha.

### Linux (this repo)

- 3 detectors: `io.full_stall`, `io.small_write_storm`, `mem.swap_on_sd`
- `doctor`, `report`, `explain`, `selftest`
- `record`/`replay` with two real captures
- Works unprivileged at unit granularity
- Gate: ~6 weeks; ships as `v0.1.0-alpha.1`

### Windows

- 3 findings, each with an ETW provider and a plain-language verdict:
  1. **Post-sleep recovery failure** — `Kernel-Power` + `NDIS` + `Kernel-Network`: a service or adapter
     failed to re-initialise after resume (the Steam download symptom).
  2. **Disk/DPC latency offender** — `PerfInfo` DPC/ISR + `StorPort`: which driver is adding latency.
  3. **USB reset / bus saturation** — `USB-*` providers + topology: why the camera dropped.
- A `report` and an `explain`; no live dashboard.
- Gate: works on a clean Windows VM with no Linux tooling; sample `.etl`-free (live session only).

### Android (exploratory)

- 2 findings via `dumpsys storaged`, `/proc/uid_io/stats`, `dumpsys meminfo`, PSI if readable:
  1. per-UID write storm / fsync-heavy app;
  2. memory pressure and reclaim churn.
- **Unrooted only**, advisor mode, no daemon, no background service.
- Gate: useful on one real device without root, or it does not ship. "We cannot see that without root"
  is an acceptable answer and should be in `doctor`.

## Sharing rules

| Share | Do not share |
| --- | --- |
| `Finding` schema, fixture format, kind + advice registries, id algorithm and vectors | Probes, parsers, scheduling |
| Pure scoring and narrative templates | Any `cfg(target_os)` code |
| Test corpus format | Dependency graphs |

`fp-core` is pure and platform-agnostic, so Windows/Android may depend on it at a pinned version.
`fp-linux` never travels. If a change is needed in shared code, it lands here first, gets a version,
and the other projects bump deliberately.

## Contract change protocol

1. Open a schema issue **in this repo** describing the additive change and why.
2. Update `SCHEMA.md` + `schema/finding.schema.json` + test vectors in one PR.
3. Bump `schema` to the next minor; keep N-1 readable.
4. Update this project. Then, and only then, update the other projects.

Cross-repo copy-paste of schema types is forbidden; it produces silent divergence.

## Anti-patterns

- **A monorepo with three platforms.** Context bloat, cfg sprawl, and an agent that "helpfully" adds
  Windows code to a Linux function.
- **A shared "utils" crate.** If it is not schema or pure scoring, it is not shared.
- **An Android daemon.** Android will kill it, and the model is wrong.
- **Windows via WSL `/proc`.** That diagnoses a Linux VM, not Windows.
- **Starting the next platform before the previous one is useful.** Three half-tools are worse than one
  finished one.
