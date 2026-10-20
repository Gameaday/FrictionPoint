# Security policy

FrictionPoint runs with read access to process names, paths, kernel counters, and (optionally) a few
capabilities. That makes it a security-relevant component even though it changes nothing.

## Supported versions

Pre-1.0: **the latest tagged release only.** There are no backports before v1.0.

## Reporting a vulnerability

Use GitHub's **private vulnerability reporting** (Security tab → "Report a vulnerability").
Do **not** open a public issue.

Please include:

- `frictionpoint --version --json` output;
- distribution, kernel version, architecture;
- whether it ran as root, with capabilities, or unprivileged;
- the smallest reproduction you have, and the impact you believe it has;
- whether local access is required.

We aim to acknowledge within **7 days**, triage within **14**, and coordinate disclosure within
**90 days**. Credit is given unless you prefer otherwise.

## In scope

| Class | Examples |
| --- | --- |
| Memory safety | A parser that panics, over-reads, or is driven out of bounds by `/proc`, kernel-log, or cgroup content |
| Privilege escalation | Escaping the systemd capability set, abusing a setuid helper, writing outside the state dir |
| Path handling | Symlink or TOCTOU attacks on config, spool, or fixture paths |
| Data exposure | Sending paths/process names to a sink the user did not configure; ignoring redaction tiers |
| Supply chain | Dependency confusion, unsigned release artifacts, compromised install script |
| Host impact | The agent making the host worse (unbounded memory, disk fill, log flood) beyond its documented budget |

## Out of scope

- A wrong or missing diagnosis on an unsupported kernel or without the required capability. Use
  `frictionpoint doctor`; that is a correctness bug, not a vulnerability.
- Resource usage inside the documented budgets (see `PLAN.md` §5).
- Attacks that already require root, or physical access.
- Anything requiring the user to run a modified binary.

## Design guarantees you can test against

1. The default build **links no network stack** and opens no listening socket.
2. **No shell execution.** `vcgencmd` is the single fixed-argv exception.
3. Read-only by default; no remediation in the agent.
4. **No telemetry.** Nothing leaves the host unless the user configures a sink.
5. Bounded memory and no data writes by default.
6. Complete unit-level diagnosis **unprivileged**; capabilities only add refinement.

A report that demonstrates one of these is false is a security report, and a welcome one.

## Hardening for operators

Run as the dedicated `frictionpoint` user with the capability set in `DEPLOYMENT.md`, under the
provided systemd unit (`ProtectSystem=strict`, `NoNewPrivileges=yes`, `RestrictAddressFamilies=AF_UNIX`,
`MemoryMax`, `CPUQuota`). In containers, use the host-visibility recipe, never `--privileged` unless
you are deliberately enabling the optional trace tier.
