# ADR-0003: Synchronous single-threaded agent, no async runtime, no `unsafe`

- **Status:** Accepted
- **Date:** 2026-10-20
- **Supersedes:** the ring-buffer/crossbeam/loom parts of the original architecture sketch
- **Related:** `docs/PLAN.md` §2, §4; `docs/AUDIT.md` §C2-C3

## Context

The agent's job is to read about a dozen small kernel files once per second, do arithmetic, and
occasionally emit a finding. That is a `while` loop with a sleep.

The earlier design assumed a lock-free ring buffer, `crossbeam`, `loom` tests, `miri`, and an async
runtime for sinks. Each of those is a maintenance contract: more dependencies to audit, more
concurrency reasoning, and a larger surface for the exact class of bug a monitoring tool must not
have (blocking, leaking, or panicking).

## Decision

1. **One thread, one loop.** Blocking reads with short timeouts. `std::thread::sleep` between ticks.
2. **No async runtime.** No `tokio`, `async-std`, `futures`, or executors anywhere in the agent.
3. **One optional worker thread** for blocking sinks (HTTP), fed by a **bounded** `std::sync::mpsc`
   queue. Dropping oldest on overflow is correct behavior for telemetry.
4. **No `unsafe`** in `fp-core`, `fp-linux`, `fp-agent`, `fp-cli`, `fp-hub`. Data structures are
   fixed-size arrays with an index — no atomics, no lock-free code.
5. **`unsafe` is quarantined** in `fp-trace` (fanotify/eBPF syscalls), which is not in the default
   feature set and is asserted by `cargo geiger` in CI.
6. The TUI renders from the same loop (poll → detect → render), so there is no shared mutable state
   and no locks.

## Consequences

- Roughly **80-100 fewer transitive dependencies**, 1-2 MB less runtime state, and a much smaller
  audit surface.
- `loom` and `miri` are no longer required (the unsafe-free crates cannot have data races), which
  removes slow nightly jobs.
- A sink cannot stall the probe loop for longer than its timeout, by construction.
- No parallel probing. At 1 Hz across ~20 cgroups this is irrelevant; if a future probe needs
  streaming rates, it belongs in `fp-trace` or the hub.
- Deterministic execution makes replay tests trivially reproducible.

## Alternatives rejected

- **`tokio` everywhere.** Solves concurrency we do not have; costs a runtime, a thread pool, and
  shutdown complexity in a program whose main duty is to never block.
- **Thread per probe.** Shared state, locking, and a new class of partial-read bugs for no gain at
  1 Hz.
- **Lock-free ring buffer.** A single-threaded ring is a `Vec` and a modulo. `unsafe` here is pure
  liability.

## Review trigger

Only if a core probe must exceed ~100 Hz of continuous sampling, or if the hub (a separate artifact)
needs real concurrency — and the hub may use threads or a small sync HTTP server without touching
this decision.
