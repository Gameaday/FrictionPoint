# ROADMAP — FrictionPoint

> **Phase 3 of the side-hustle year (May – Jul 2027). Starts only if a prior gate is green.**
> Master plan lives in the home repo: side-hustle-master-plan.md

**One-liner:** find out *why* your system hitches — root-cause diagnostics, not another CPU meter.

**Differentiation:** htop and glances show what is loaded. FrictionPoint attributes micro-stalls:
I/O contention, context-switch storms, storage stalls, thermal throttling.
Rust, cross-platform, SBC-first (native on Raspberry Pi).

**Precedent:** iStat Menus sells a one-time-priced system monitor (with paid major upgrades)
and survives against free open-source rivals. Depth beats free when it answers a question free tools cannot.

## Rules

Same as the master plan: local tool = zero marginal cost, AI for development only,
one-time pricing with paid major upgrades instead of subscriptions, ≤ 2 hrs/week maintenance.

## Quarter plan

**Month 1 (May 2027)**
- [ ] Core monitor (free): CPU/GPU/mem/net/disk + a hitch event timeline
- [ ] CLI + TUI on Linux/Raspberry Pi first (Windows in month 2)

**Month 2 (Jun 2027)**
- [ ] Root-cause engine: per-hitch report — what stalled, who caused it, how to fix it
- [ ] Windows support

**Month 3 (Jul 2027)**
- [ ] Launch: $15 one-time Pro (reports, alert rules, history/export)
- [ ] Channels: r/raspberry_pi, r/homelab, r/selfhosted + the shared storefront and email list

## Pricing sketch

| Tier | Price | Includes |
|---|---|---|
| Free | $0 | monitoring + hitch timeline |
| Pro | $15 one-time | root-cause reports, alert rules, export |

## Gate criteria (end of Jul 2027)

- **Green (≥ 25 Pro sales):** Phase 4 (TShock suite) proceeds as the bonus/outlier phase.
- **Otherwise:** stop starting new things. Iterate.
