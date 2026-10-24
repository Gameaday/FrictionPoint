# Pre-Launch Audit — FrictionPoint (2026-10-24)

> Companion to [ROADMAP](./ROADMAP.md). Phase 3 product (May–Jul 2027), pre-development. This audit defines the 1.0 bar now so the phase starts sharp.

## Verdict summary

| Area | Status | One-liner |
|---|---|---|
| Core focus | ✅ tightest of the four | "Why did it hitch?" — one question, answered |
| Scope sprawl | ✅ none yet, one tripwire | "Cross-platform everywhere" is the risk phrase |
| Tier design | ✅ sound | Free monitor, $15 one-time Pro reports |
| Polish | n/a → defined | No code yet; this doc sets the pre-launch bar |

## Core promise & scope check

The promise: *attribute micro-stalls to their cause.* The ROADMAP is properly narrow — monitor + hitch timeline (free), per-hitch root-cause reports (Pro). Guardrails to adopt at phase start:
- **v1.0 = Linux + Raspberry Pi, CLI/TUI only.** Windows is month 2. macOS, BSD, container images: not until after the gate.
- Every feature must serve the question "what stalled, who caused it, how do I fix it." Anything that only shows *load* is a commodity (htop parity) — skip it.
- **Remediation is a report, not an action.** FrictionPoint diagnoses; it never auto-tunes, auto-kills processes, or applies "fixes." That keeps liability, scope, and support load at zero.

## Tier design review

Free/Pro split is right (monitor free, answers paid). Two notes:
- Keep the iStat precedent in mind: **paid major upgrades** are the accepted one-time-pricing escape valve for a product that must chase kernels and hardware. Decide the free-update window at 1.0 (e.g., lifetime within 1.x, paid 2.0) and say so on the sales page — it prevents the FileBot backlash dynamic.
- Alert rules and export are the right Pro features (recurring utility, zero marginal cost).

## Professionalism & polish bar (defined now, checked at launch)

| Item | Bar |
|---|---|
| Install | one-liner (curl | sh / cargo install / AUR package for RPi) |
| Demo | one GIF of a hitch being attributed, on the README, day one |
| Sample output | a real per-hitch report in the README — the product *is* the report |
| CI | lint + test + cross-build for aarch64 on tag |
| Benchmarks | a tiny "works on a Pi 4 under load" claim, measured, on the landing page |
| CHANGELOG + semver | from the first tag |

## "As-is" vs 1.0

- **As-is:** nothing shippable — correct for a pre-phase repo. Current job: stay out of the way and keep the thesis written down (done).
- **FrictionPoint 1.0 (Jul 2027 bar):** Linux/RPi + Windows, hitch attribution live, report + export + alert rules in Pro, $15 one-time, r/raspberry_pi + r/homelab launch.

## Ordered steps when the phase starts

1. Spike: eBPF/tracepoint-based stall attribution on a real Pi — feasibility before UI.
2. Hitch timeline + CLI dump (the free core).
3. Root-cause report engine (the Pro core).
4. Everything on the polish bar above.
