# ADR-0002: Findings are the wire format; no database, no central server in v0.1

- **Status:** Accepted
- **Date:** 2026-10-20
- **Deciders:** project owner
- **Related:** `docs/DEPLOYMENT.md` §6-7, `docs/ARCHITECTURE.md` §3

## Context

It is tempting to model FrictionPoint as a time-series system: sample everything, store it, chart
it, alert on it. That design needs a database, a retention policy, a query language, and a server,
and it turns the project into a worse Netdata. On an SD-card device it also creates sustained write
load — the exact pathology the tool is meant to expose.

The founding incident (a log flood on one host taking down ingress on another) is fundamentally a
*cross-host* correlation, which creates pressure to build a fleet server immediately.

## Decision

1. **The unit of output is a `Finding`**: a small, immutable, self-describing JSON document
   containing kind, severity, confidence, culprit, evidence, advice ids, `resource_keys`,
   host/boot identity, visibility context, and `schema_version`.
2. **No database.** In-memory ring buffers for live analysis; optional bounded spool; no SQLite.
3. **No central server in v0.1.** Transports are `Sink` adapters (MQTT, OTLP, webhook, JSONL) added
   behind a trait. The collector is a subcommand of the same binary, added when demanded.
4. **Cascade is a seam, not a feature.** Findings carry namespaced `resource_keys` and an optional
   declarative `topology.toml`, so a future collector can join them without a schema change.
5. **Sinks are downstream of findings.** Metrics are exported for people who want them, but the
   product's contract is the finding document.
6. **Tier carve-out.** The no-database rule applies to the *monitored-host agent*. A future hub
   tier (`frictionpoint hub`, see `docs/FLEET.md`) may keep a bounded event store and index,
   because it is not the system under diagnosis. The `Finding` document remains the only contract
   between tiers.

## Consequences

- Upgrades have no migrations. Rollback is replacing a file.
- Historical recordings (`.fpz` fixtures) let detectors be re-scored forever, which is what makes
  the agentic-development benchmark meaningful.
- Cross-host cascade detection is deferred but not designed out.
- Users who want long-term history export to their existing system instead of getting a worse one
  from us.
- `Finding` becomes an expensive-to-change API. It must be reviewed carefully and versioned from
  day one.

## Alternatives rejected

- **SQLite time-series store.** Write amplification on SD, migration burden, and it turns a
  diagnoser into a database product. Rejected.
- **Server-first fleet architecture.** Solves the rarest case (multi-host cascade) before the
  common one (single-host stall), and adds a listening service to every installation. Rejected.
- **Prometheus-first.** Counters cannot carry evidence, confidence, or narrative. Useful export,
  wrong contract. Rejected.

## Review trigger

Add a real collector/storage tier only when there is evidence users are running 5+ hosts and
asking for cross-host correlation. Even then, the finding schema stays the contract.
