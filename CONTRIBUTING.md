# Contributing

FrictionPoint is a one-developer side project with strict invariants. Small, focused PRs get reviewed
fastest; large ones may sit.

**Read [`AGENTS.md`](AGENTS.md) even if you are human.** The rules are the same, and they are the
reason the codebase stays small.

## The short version

1. **One area/detector per PR.** A detector is a fixture, a golden output, a negative test, and a
   narrative — not just a function.
2. **Never weaken a test or edit a fixture to make a PR pass.**
3. **No new dependencies** without first opening an issue that updates `PLAN.md` §3. The agent budget
   is three direct dependencies.
4. **No `unsafe`** outside `fp-trace`, no async runtime, no threads except the sink worker, no
   `unwrap`/`expect`/`panic!` in non-test code.
5. **No allocation in the tick path.**
6. **No shell execution and no network** in the default build.
7. **Run the gates:** `cargo fmt --all --check`, `cargo clippy --all-targets --all-features -D
   warnings`, `cargo test`, `cargo xtask verify`, `cargo deny check`.
8. **Sign off your commits** (`git commit -s`) to certify the [DCO](https://developercertificate.org/).
   There is no CLA.

## Adding a detector

- Pick an area in [`docs/AREAS.md`](docs/AREAS.md); do not invent a new one in a PR.
- Add the detector id to `docs/ARCHITECTURE.md` §7 and the metric to `docs/METRICS.md` if it is new.
- Provide a fixture recorded from **real hardware** (not hand-authored), plus a `clean_baseline`
  negative test and a red-team case where the obvious suspect is innocent.
- Populate `severity.coverage_pct`, add the narrative template and an advice id.
- Keep the diff under ~400 lines.

## Commit messages

```text
area(scope): imperative summary

T7: parse PSI total counters and emit io.full_stall
```

## What we will decline

Dashboards, query DSLs, automatic remediation, a new platform before Linux v1.0, a plugin ABI, and
any metric that is not in the catalogue. These are recorded as closed directions in
[`docs/AUDIT.md`](docs/AUDIT.md) and [`docs/LESSONS-STALLWATCH.md`](docs/LESSONS-STALLWATCH.md).

## Security

Never open a public issue for a vulnerability. See [`SECURITY.md`](SECURITY.md).

## Review expectations

Best effort, usually within a week. If a PR is quiet for two weeks, ping it once — do not assume it
was ignored.
