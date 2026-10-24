# FrictionPoint

> **Find out *why* your system hitches** — root-cause diagnostics, not another CPU meter.

FrictionPoint is a lightweight, cross-platform OS and hardware diagnostic watcher written in Rust. htop and glances tell you *what is loaded*; FrictionPoint attributes the actual **micro-stalls**: I/O contention, context-switch storms, storage stalls, thermal throttling. Starting with Raspberry Pi and scaling everywhere.

## The idea

Every sysadmin and homelab owner has lived this: the system "freezes for a second" now and then, CPU looks fine, RAM looks fine, and nothing explains it. FrictionPoint watches for those hitches and answers the questions a load meter cannot:

- What stalled — disk, thermal, memory pressure, or a driver?
- Who caused it — which process or interrupt storm?
- What to do about it — a per-hitch report with the likely fix.

## Status

Pre-development. Scheduled as **Phase 3** of the side-hustle year (May – Jul 2027), starting only if the prior product's gate is green. The quarter-by-quarter plan, pricing sketch, and gate criteria live in the [ROADMAP](./ROADMAP.md).

Pricing philosophy: free monitoring for everyone, one-time Pro (planned $15) for root-cause reports, alert rules, and export. No subscription — you own the tool.

## License

[MIT](./LICENSE)
