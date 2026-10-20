# AGENTS.md — operating contract for coding agents

You are implementing one task in FrictionPoint, a Linux/SBC system-friction auditor written in Rust.
The architecture is settled. Your job is to execute a task precisely, not to improve the design.

## Read exactly these, in this order

1. `docs/SCHEMA.md` — the `Finding` contract. **Wins over everything else.**
2. `AGENTS.md` (this file) — the rules.
3. `docs/TASKS.md` — the task list. Take **one** task card.
4. `docs/PLAN.md` §2 and §5 **only if the task card references them** — crate map and budgets.

Do **not** load `VISION.md`, `ARCHITECTURE.md`, `AUDIT.md`, `FLEET.md`, `DEPLOYMENT.md`,
`PRIOR-ART.md`, `LESSONS-STALLWATCH.md`, `METRICS.md`, or the ADRs into context unless a task card
tells you to.
They are reference material for humans. Loading them dilutes the invariants that matter.

## Non-negotiable rules

1. **One task = one branch = one PR.** Do not start the next task.
2. **Test first.** Write the failing test named in the task card, run it, show it failing, then
   implement. No production code before a failing test, except files created by task T0.
3. **No new dependencies.** Not one, not "small". A dependency requires a human-approved issue that
   first updates `PLAN.md` §3.
4. **No `unsafe`** outside `fp-trace`. **No async runtime.** **No threads** except the single
   optional sink worker fed by `std::sync::mpsc`.
5. **No `unwrap`, `expect`, `panic!`, `todo!`, or `unimplemented!`** in non-test code. Return typed
   errors. Unknown data is `Option`/`None`, not a panic.
6. **Never weaken a test to make it pass.** Never edit a fixture to match your output. Never add a
   sleep to fix a race.
7. **No allocation in the tick path.** Buffers are allocated at startup and reused.
8. **No shell execution, no network** in the default build. `vcgencmd` is the one audited exception,
   with a fixed argv and no user-controlled arguments.
9. **Do not change `docs/SCHEMA.md`**, the fixture format, or any `kind` string. Those require an
   explicit schema task.
10. **Do not reformat unrelated code.** Keep the diff under ~400 lines. If it grows, stop and split.
11. **Do not add configuration** to avoid a design decision. Choose, document, and move on.
12. **If a task needs root to test, stop and ask.** Tests must pass as an unprivileged user.

## Commands (the only gates that matter)

```bash
cargo fmt --all --check
cargo clippy --all-targets --all-features -- -D warnings
cargo test
cargo xtask verify          # added by T0; runs fmt+clippy+test+perf+dep budget
cargo deny check            # licenses, advisories, bans
```

A task is not done because the code compiles. It is done when `cargo xtask verify` passes and the
task card's acceptance evidence exists.

## Task protocol

1. **Restate** the task in five lines: goal, files you will touch, interfaces used, test you will
   write, what you will *not* touch.
2. **Write the failing test** named in the card. Run it. Paste the failure.
3. **Implement the minimum** that makes it pass. No speculative abstraction, no extra modules.
4. **Run the gates.** Fix real failures. Do not silence them.
5. **Show evidence**: test names, a sample `Finding` JSON if relevant, and the measured number if the
   card asks for one.
6. **Stop.** Report what changed and what you did not do.

## Stop and ask a human when

- you think you need a dependency, an `unsafe` block, a thread, or a schema change;
- the task card conflicts with `SCHEMA.md`;
- the diff would exceed ~400 lines;
- the only way to test is as root;
- a performance or memory budget cannot be met;
- you find an existing test that contradicts the task.

## Domain anti-patterns (these are the ways this project dies)

| Anti-pattern | Why it is fatal |
| --- | --- |
| Polling every PID every tick | Blows the CPU budget on a Pi; use cgroup-first + staggered sampling |
| Treating a sample window as a duration | The prior art reported 400 ms captures as freeze lengths for ~520k incidents |
| Blaming the process with the most bytes | Latency is per request; use request share + blocked-time cross-check |
| Emitting a finding with no evidence | The trust model is "show the kernel path and the number" |
| Naming a culprit below the share threshold | Report `culprit: null` and say "unattributed" instead |
| Silently degrading | Degradation goes in `visibility.degraded` and in `doctor` |
| A detector without a negative test | It *will* fire on `clean_baseline` and destroy trust |
| Adding a detector that needs root | The unprivileged path is the product |
| "Just this once" dependency | Three agent dependencies is a feature, not a starting point |

## Definition of done (per detector)

- five-line spec comment above the detector;
- fixture recorded from a real machine, plus a golden output;
- a negative test proving silence on `clean_baseline`;
- `severity.coverage_pct` populated;
- narrative template and at least one `advice` id;
- one line in the detector catalogue (`ARCHITECTURE.md` §7).

## What a human reviewer checks

1. Did the diff add a dependency, `unsafe`, async, or a thread?
2. Does `clean_baseline` still produce zero findings?
3. Is there a negative test, or only a happy-path test?
4. Does the finding carry evidence and coverage?
5. Is the reported `stalled_ms` derived from counters, not from a sample interval?
6. Does it work unprivileged?
7. Is the diff within budget, and does it avoid unrelated reformatting?
