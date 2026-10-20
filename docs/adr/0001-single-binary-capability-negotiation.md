# ADR-0001: One binary, capability negotiation, deployment as packaging

- **Status:** Accepted
- **Date:** 2026-10-20
- **Deciders:** project owner
- **Related:** `docs/DEPLOYMENT.md`

## Context

FrictionPoint must eventually run on Raspberry Pi, servers, containers, Kubernetes, Windows, and
Android. The obvious but wrong move is to fork the build per target: a `docker` feature, a
`pi` feature, an "agent" binary and a "server" binary. That produces behaviour drift, a
combinatorial test matrix, and a class of bugs that only appear in one packaging.

There is a second, subtler trap. Containerised deployments *look* fine while being blind: Docker
defaults to a private cgroup namespace on cgroup v2, so a container sees only its own subtree and
would honestly report "no stalls" while the host is in full I/O stall.

## Decision

1. **One static binary** (`frictionpoint`) with subcommands: `baseline`, `watch`, `report`,
   `explain`, `doctor`, `selftest`, `record`, `replay`, `export`, `daemon`, `collect`.
2. **Deployment mode is discovered at runtime, never compiled in.** Probes answer "can I read X?"
   and the answers become a `Capabilities` record.
3. **Degradation is announced.** Startup banner, report header, and `doctor` all state what is and
   is not visible. `exit code 3` means "insufficient visibility", never "healthy".
4. **Packaging is orthogonal.** `.tar.gz`, `.deb`, OCI image, Home Assistant integration, and
   Ansible role all wrap the same artifact and differ only in which capabilities the environment
   grants.

## Consequences

- One test suite, one behaviour contract, no version skew, no per-target build flags.
- The tool can be honest in environments where it is weak, which is the entire trust premise.
- Container recipes must be explicit and slightly verbose (`pid: host`, `cgroupns: host`,
  read-only mounts). That verbosity is documentation, not friction.
- Some capabilities are simply unavailable in some modes (file-level attribution in Kubernetes,
  `vcgencmd` off-Pi). Detectors must treat these as `Option`, not assume presence.

## Alternatives rejected

- **Separate agent/server binaries.** Adds release engineering and version skew to save a few
  megabytes. Rejected.
- **`--features docker` / `--features pi`.** Compile-time variance multiplies CI and hides
  behaviour differences from tests. Rejected.
- **Container-first.** Loses fanotify, thermal, `vcgencmd`, and honest cgroup attribution — i.e.
  it loses the reasons the tool exists. Rejected as the default mode.

## Review trigger

Revisit only if a platform requires code that cannot be feature-gated at runtime (for example a
kernel module). Even then, the split should be a separate *probe crate*, not a separate product.
