# ADR-0004: The root agent does not link a network stack (two artifacts, one schema)

- **Status:** Accepted
- **Date:** 2026-10-20
- **Amends:** ADR-0001 ("one binary") — the *spirit* survives, the letter does not
- **Related:** `docs/PLAN.md` §2-3, `docs/DEPLOYMENT.md` §6

## Context

ADR-0001 chose a single binary to avoid agent/collector version skew. That reasoning was about the
**protocol**, not about the artifact. Meanwhile, three facts argue against a fat binary:

1. The agent runs as root (or with capabilities) and parses attacker-influenced data. Linking a TLS
   stack, an HTTP server, and an MQTT client into it enlarges the attack surface of the most
   privileged process in the system.
2. `RestrictAddressFamilies=AF_UNIX` blocks sockets at runtime, but **not linking the code at all**
   is a stronger, auditable guarantee (`nm`/`ldd` can prove absence).
3. A Pi user pays the binary size and compile time for features they will never enable.

## Decision

1. **Two artifacts from one workspace, released together, same version:**
   - `frictionpoint` — agent + CLI. **Default build links no networking code.**
   - `frictionpoint-hub` — optional server (post-1.0). Networking, read-only API.
2. **One schema, one `fp-core`,** so there is no protocol skew. A single `git tag` and a single
   changelog cover both.
3. The agent may opt into an outbound-only **client** via the `http` feature (`ureq` + `rustls`),
   used solely to push findings. It still never listens.
4. The hub adds no new data model; it stores and joins the same `Finding` documents.

## Consequences

- Attack surface and binary size are feature-selected, not accidental.
- Release engineering is two artifacts, not one (a few extra CI minutes). Acceptable.
- ADR-0001's real goal — no version skew — is preserved by shared crates and a shared tag.
- Users who only want local diagnosis never install a server.
- MQTT, OTLP, and any future transport stay out of the agent core; they live in sinks/binaries.

## Alternatives rejected

- **Single fat binary.** Root + network + TLS + UI, with no way to prove the network code is
  unreachable. Rejected on security grounds.
- **Separate repositories per tier.** Guarantees schema drift and doubles the maintenance load.
  Rejected.
- **Agent shells out to `curl`.** Removes a dependency and adds a command-injection surface.
  Rejected outright.

## Review trigger

Never for the agent. The hub may itself split into `hub` + `pane` later without touching this.
