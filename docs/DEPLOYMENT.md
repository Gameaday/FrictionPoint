# Deployment & Distribution

> **Decision:** one static binary is the canonical artifact. **Host + systemd is the default and
> the only mode with full fidelity.** Containers are a supported, *self-declaring degraded* mode
> for servers/Kubernetes and for demo/CI. No central server in v0.1 — findings are the wire
> format, transports are adapters added later.

This document exists to prevent the most expensive class of rework: discovering, six months in,
that the deployment model quietly changed what the tool can see and therefore what it can
truthfully claim.

---

## 1. Host-first is a correctness decision, not a convenience one

A diagnostic tool must run where the thing it diagnoses runs. Inside a container, several of
FrictionPoint's primary signals are structurally unavailable or actively misleading:

| Problem | Why it breaks diagnosis |
| --- | --- |
| **Private cgroup namespace** (Docker default on cgroup v2) | `/sys/fs/cgroup` shows only the container's own subtree. The host can be in full I/O stall while the container's cgroup is genuinely idle. The tool would report **"healthy"** — the exact failure mode `stallwatch` documents in its own limitations. |
| **Mount namespace** | fanotify marks are attached to mounts/filesystems in *your* namespace. Host filesystem activity is not observable through container-local marks. File-level attribution dies. |
| **Overlay filesystem** | Copy-up and different write patterns change the I/O you are trying to measure. The observer perturbs the observation. |
| **Container logging** | Docker's default `json-file` driver produces small, frequent writes. On an SD card, the monitoring container can become a *source* of the pathology it is supposed to find. |
| **`/proc`** | You cannot meaningfully bind-mount the host's `/proc` over the container's. `--pid=host` is the only sane route, and even then cgroup-related files reflect the container's view. |
| **Devices** | No `/dev/vchiq` means no `vcgencmd get_throttled` → no undervoltage or thermal-throttle detection on a Pi. No `/dev/kmsg` means no `mmc` timeout / ext4 error events. |
| **Swap & block topology** | `/proc/swaps` and `/sys/block` may be partial or absent, hiding the "swap on SD card" class of finding. |

So the rule is: **the observer must share the observed host's kernel view.** Containers are used
where that is impossible (managed Kubernetes nodes), and there the tool must say what it cannot
see rather than pretend.

---

## 2. The design rule that avoids the rearchitecture

> **Deployment mode is a runtime capability negotiation, never a compile-time feature.**

- No `--features docker`, no `#[cfg(target_env = "musl")]` behavior forks, no separate "agent"
  and "server" binaries.
- At startup, probes answer *"can I read X?"* and the answers are recorded as `Capabilities`.
- `frictionpoint doctor` prints the full visibility report.
- **Every finding carries its visibility context**, and the report header prints it:

```text
visibility: container/private-cgroup  (degraded: no per-unit attribution, no file-level, no thermal)
```

A degraded mode that announces itself is honest. A degraded mode that silently returns "no
problems found" is a bug that destroys trust. This is why the capability model is an
invariant (see `docs/adr/0001-single-binary-capability-negotiation.md`), not an implementation
detail.

---

## 3. Capability matrix by deployment mode

`F` = full, `P` = partial, `-` = unavailable.

| Signal | Host (systemd) | Docker (defaults) | Docker (host-visibility recipe) | K8s DaemonSet (hostPath) |
| --- | --- | --- | --- | --- |
| System PSI | F | F (not namespaced; label provenance) | F | F |
| Per-cgroup / per-unit attribution | F | **-** (private cgroup ns) | F (`cgroupns: host`) | F (hostPath) |
| Per-process I/O (`/proc/<pid>/io`) | F (root) / P (own user) | P | F (`pid: host`) | F (`hostPID`) |
| Block device latency / queue depth | F | P | F (`/sys/block`) | F |
| Thermal zones | F | P (bind-mount) | F | P |
| `vcgencmd` throttle/undervoltage (Pi) | F | - | F (`--device /dev/vchiq`) | - |
| Kernel log (`mmc`/ext4 errors) | F (`adm` group) | P (`/dev/kmsg`) | F | P |
| Swap on SD detection | F | P | F | P |
| fanotify file-level attribution | F (`CAP_SYS_ADMIN`) | **-** | P (host mount ns needed) | **-** |
| eBPF per-request latency | F (root, BTF) | - | P (privileged + tracefs + BTF) | P |
| USB topology | F (`/sys/bus/usb`) | P | F | P |
| **Overall claim strength** | **correlated** (traced with the optional `trace` feature) | **severity only** | **correlated** | **correlated** |

Two consequences worth designing around now:

1. **`Docker (defaults)` is the most dangerous mode** because it looks like it works. Refuse to
   run it silently: print the degraded banner and make `doctor` explicit.
2. **File-level attribution is a host-mode feature.** Do not promise it in container mode, and
   do not let the detector assume a `Culprit::File` is always available.

---

## 4. Host mode (recommended for Pi and single boxes)

Install: static binary + checksum, then a unit. Later: distro packages and an Ansible role.

```ini
[Unit]
Description=FrictionPoint system friction auditor
Documentation=https://github.com/Gameaday/FrictionPoint
After=local-fs.target
StartLimitIntervalSec=60
StartLimitBurst=5

[Service]
Type=notify
WatchdogSec=30s
ExecStart=/usr/local/bin/frictionpoint daemon --config /etc/frictionpoint.toml
User=frictionpoint
SupplementaryGroups=adm systemd-journal video
# Only what attribution needs; drop everything else.
AmbientCapabilities=CAP_SYS_ADMIN CAP_BPF CAP_PERFMON
CapabilityBoundingSet=CAP_SYS_ADMIN CAP_BPF CAP_PERFMON
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=read-only
PrivateTmp=yes
# Needs /dev/vchiq (Pi) and/or /dev/kmsg; keep devices but allow-list them.
DeviceAllow=/dev/vchiq rw
DeviceAllow=/dev/kmsg r
RestrictAddressFamilies=AF_UNIX
LockPersonality=yes
MemoryDenyWriteExecute=yes
SystemCallArchitectures=native
SystemCallFilter=@system-service
MemoryMax=48M
CPUQuota=5%
TasksMax=32
IOAccounting=yes
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Why each group matters:

- `frictionpoint` user + `adm`/`systemd-journal` gives read access to the kernel log; `video`
  gives `vcgencmd`. Everything else works unprivileged.
- `AmbientCapabilities` + `NoNewPrivileges` is the least-privilege way to get fanotify
  (`CAP_SYS_ADMIN`) and eBPF (`CAP_BPF`, `CAP_PERFMON`) without running as uid 0.
- `RestrictAddressFamilies=AF_UNIX` **enforces the "never listens" invariant in the kernel**.
  Exporters that need TCP get a drop-in that widens only that unit.
- `ProtectSystem=strict` + `MemoryMax` + `CPUQuota` make the observer incapable of becoming the
  bottleneck, and make the budget a kernel-enforced fact rather than a benchmark claim.

Config lives in `/etc/frictionpoint.toml` with a drop-in directory
`/etc/frictionpoint.d/*.toml` (systemd convention). That is the single most useful property for
fleet management: ship one base config plus per-role drop-ins.

---

## 5. Container mode (servers, Kubernetes, demo)

Only meaningful with host visibility turned on deliberately. Everything below is load-bearing.

```yaml
services:
  frictionpoint:
    image: ghcr.io/frictionpoint/frictionpoint:0     # pin major, never :latest
    pid: host                # host /proc -> per-process attribution
    cgroupns: host           # full cgroup tree (Docker defaults to PRIVATE on cgroup v2)
    network_mode: none       # core never talks; use a separate exporter profile
    read_only: true
    cap_drop: [ALL]
    # enable only when tracing is turned on:
    # cap_add: [SYS_ADMIN, BPF, PERFMON]
    security_opt: ["no-new-privileges:true"]
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
      - /sys/block:/sys/block:ro
      - /sys/bus/usb:/sys/bus/usb:ro
      - /sys/class/thermal:/sys/class/thermal:ro
      - /dev/kmsg:/dev/kmsg:ro            # optional: mmc/ext4 error events
    # - /dev/vchiq:/dev/vchiq            # Raspberry Pi only
    mem_limit: 64m
    pids_limit: 32
    logging:                              # do not let the monitor fill the disk it audits
      driver: local
      options: { max-size: "1m", max-file: "1" }
    restart: unless-stopped
    command: ["daemon"]
```

Kubernetes DaemonSet (the node-exporter/cAdvisor pattern):

```yaml
spec:
  template:
    spec:
      hostPID: true
      priorityClassName: system-node-critical
      tolerations:
        - operator: Exists            # run on control-plane nodes too
      containers:
        - name: frictionpoint
          image: ghcr.io/frictionpoint/frictionpoint:0
          args: ["daemon"]
          securityContext:
            runAsNonRoot: true
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }   # add caps only for the tracing tier
          resources:
            requests: { cpu: 10m, memory: 24Mi }
            limits:   { cpu: 100m, memory: 64Mi }
          livenessProbe:
            exec: { command: ["frictionpoint", "selftest"] }
            periodSeconds: 60
          volumeMounts:
            - { name: cgroup, mountPath: /sys/fs/cgroup, readOnly: true }
            - { name: block,  mountPath: /sys/block,      readOnly: true }
      volumes:
        - { name: cgroup, hostPath: { path: /sys/fs/cgroup } }
        - { name: block,  hostPath: { path: /sys/block } }
```

Notes:

- `hostPID` + `hostPath /sys/fs/cgroup` is the standard pattern for node agents; verify with
  `kubectl exec ... -- frictionpoint doctor` on each distro, because cgroup-namespace behaviour
  varies by container runtime and kernel.
- `livenessProbe` uses `selftest`, *not* `doctor`: a degraded-but-working agent is alive.
- The image is static/distroless, non-root, read-only rootfs, no shell. There is no reason for
  a diagnostic binary to ship a package manager.
- Do **not** advertise file-level attribution in Kubernetes. It is a host-mode capability.

---

## 6. Fleet / network architecture

### 6.1 Push, not pull (for the audience that needs this most)

A homelab is behind NAT. A Pi is behind CGNAT. Any design that requires the collector to connect
*to* each host needs port forwards, VPNs, or a mesh. Any design that requires the host to run an
HTTP server adds attack surface and memory to a 512 MB device.

Therefore:

- **Agents push outbound only.** They never listen. This is enforced by systemd
  (`RestrictAddressFamilies=AF_UNIX`) and by the absence of a listener in the code.
- **The collector is optional and separate.** It can be a laptop, a NAS, or a $5 VPS.
- **Prometheus scraping is a secondary, metrics-only path** for people who already run it. It is
  a poor fit for findings (events with evidence), so it is never the primary transport.

### 6.2 Transports, in build order

| Order | Transport | Why |
| --- | --- | --- |
| 1 | **MQTT** | NAT-friendly, tiny payloads, broker handles offline buffering, native to Home Assistant and homelabs. TLS + per-host credentials. Topics: `frictionpoint/<host_id>/finding` (QoS 1), `frictionpoint/<host_id>/status` (retained). |
| 2 | **OTLP/HTTP** | Shops that already run an OTel collector. Findings as log records with attributes; raw signals as metrics. |
| 3 | **Webhook** | Boring and universal. HMAC-signed, one POST per finding, exponential backoff. |
| 4 | **Prometheus scrape** | Metrics only (`frictionpoint_*`); explicitly not the findings path. |

All four sit behind one `Sink` trait and share one serialization path. Adding a transport must
never touch a detector.

### 6.3 Bandwidth and spooling

- Idle: **0 bytes/second**, no connection churn beyond a keepalive.
- One finding: roughly **2-4 KB** of JSON (narratives + evidence paths dominate).
- A terrible day: 100 findings ≈ 400 KB. This tool is not a bandwidth problem.
- Offline: keep the last N findings in a **bounded** in-memory ring (default 256). Optional disk
  spool is **off by default**, size-capped, and prefers tmpfs when available — because writing
  logs to the SD card to report on SD card write storms would be self-defeating.

### 6.4 Collector (`frictionpoint collect`)

Same binary, different subcommand, different unit. It:

- subscribes to MQTT / accepts webhooks,
- joins findings in a sliding window (default 120s),
- emits cascade findings when resource keys overlap across hosts,
- has **no database** — in-memory window plus optional bounded spool.

One binary means no agent/collector version skew. That is worth more than the few megabytes a
split would save.

---

## 7. The cascade seam (build the shape now, the feature later)

Your founding incident — a log flood on one Pi killing ingress on another — is a *cross-host*
correlation. It cannot be detected by any single-host agent, and it should not force a distributed
database into v0.1. The seam is two cheap things:

1. **Every finding carries `resource_keys`**, stable, namespaced strings:

```text
unit:traefik.service@pi-central
device:mmcblk0p2@pi-central
sink:logs@pi-central
host:pi-lab-3
```

2. **An optional `topology.toml`** declares shared resources and edges (declarative, no discovery):

```toml
[[node]]
id = "pi-central"
role = "ingress"

[[node]]
id = "pi-lab-3"
role = "worker"

[[edge]]
from = "pi-lab-3"
to   = "pi-central"
kind = "logs"
```

With those two, cascade detection is a join in the collector. Without them, it needs a schema
change and a re-collection of history. This is the highest-leverage "foresight" item in the
entire design: **add the keys in v0.1 even though nothing consumes them yet.**

---

## 8. Identity, config, and upgrades

**Identity (from day one, cheap now, painful later):**

- `host_id` from `/etc/machine-id` (fallback: SMBIOS UUID), plus `boot_id`, plus a display
  label and role from config.
- `host_id` must survive reboots and package reinstalls; `boot_id` distinguishes "restarted"
  from "was stalled."

**Config precedence:**

```text
CLI flag  >  FRICTIONPOINT_* env  >  /etc/frictionpoint.d/*.toml  >  /etc/frictionpoint.toml  >  defaults
```

The drop-in directory is what makes Ansible/Salt/Puppet deployments sane: one base file, one
small file per role, no templating of a monolith.

**Versioning (three independent numbers):**

| Version | Governs | Compatibility rule |
| --- | --- | --- |
| binary | the executable | semver |
| `finding_schema` | the wire format | collectors accept N-1 |
| `fixture_schema` | recorded scenarios | replay refuses unknown major |

**Upgrades:** atomic binary replacement, `systemctl restart`, done. No migrations, because there
is no database. That is a deliberate consequence of the "no DB" invariant, and it is one of the
reasons to keep it.

---

## 9. Bulk deployment and architecture matrix

Ansible role sketch (idempotent, non-interactive, no prompts):

1. detect `uname -m` and `dpkg --print-architecture`,
2. fetch the matching static binary + checksums file, verify,
3. create the `frictionpoint` user and add to `adm`, `systemd-journal`, `video`,
4. template `/etc/frictionpoint.toml` and drop-ins,
5. install the unit, `daemon-reload`, `enable --now`,
6. run `frictionpoint doctor --json` and assert `exit != 3` (fail the play if visibility
   silently degraded).

Target matrix:

| Target | Use |
| --- | --- |
| `aarch64-unknown-linux-musl` | Pi 3/4/5/Zero 2 W on a 64-bit OS; most modern SBCs |
| `armv7-unknown-linux-musleabihf` | 32-bit Pi OS on Pi 2/3/4/Zero 2 W |
| `arm-unknown-linux-musleabihf` | Pi 1 / Zero (armv6) |
| `x86_64-unknown-linux-musl` | servers, containers, CI |

Verify the armv6 target early: it is the most likely cross-compilation papercut, and the Pi Zero
is exactly the device the budget claims to support.

---

## 10. Security posture

- **Least privilege ladder:** unprivileged (PSI, diskstats, thermal) → `adm`/`systemd-journal`
  (kernel log) → `video` (`vcgencmd`) → capabilities (`CAP_SYS_ADMIN` fanotify, `CAP_BPF` +
  `CAP_PERFMON` eBPF) → full root (only if a user explicitly forces it). Each rung is opt-in and
  reported by `doctor`.
- **No shell-outs.** No `sh -c`, no `eval`, no parsing of `ls` output. Spawning `vcgencmd` is the
  one narrow, documented exception, and it takes no user-controlled arguments.
- **Attacker-controlled input.** Process names, file paths, kernel log lines, and cgroup names are
  all untrusted. Parsers are fuzzed; `fp-core` denies `unsafe`; output is escaped.
- **Privacy.** Process names and paths can reveal browsing, work, and health information. Local by
  default, no telemetry, `--redact-home` and path hashing available, and evidence paths are only
  emitted to sinks the user configured.
- **Supply chain.** `cargo-deny`, signed release artifacts, checksums, SBOM, pinned major tags.
- **The collector is a server.** mTLS or scoped tokens, no anonymous writes, rate limits, and no
  rendering of untrusted strings into HTML (MQTT/HA dashboards included).

---

## 11. Anti-patterns that would force a rearchitecture

| Anti-pattern | Why it is fatal later |
| --- | --- |
| Compile-time deployment/platform forks | Doubles the test matrix and guarantees behavior drift between "the Pi build" and "the Docker build" |
| Metrics-first data model | Findings are the product; metrics are a derived export. Inverting this makes every narrative a special case |
| Embedded database (SQLite) | Write amplification on the exact device being diagnosed; also introduces migrations and upgrade complexity |
| Agent that listens on a port | Attack surface on every host, NAT problems, and contradicts the root-cause-isolation story |
| Logging to a file by default | The tool becomes an SD-card writer and a false positive in its own reports |
| Separate server binary/version | Version skew between agent and collector, duplicated release engineering |
| Remediation in the core | Privilege, rollback, and bricking risk contaminate a read-only diagnostic |
| Storing only derived findings | You can never re-score a new detector against old incidents |
| Hardcoding systemd name mapping | Breaks on OpenRC, runit, Docker, Kubernetes, and non-Linux; attribution needs a lookup layer with fallbacks |

---

## 12. What is reversible, and what must be decided now

| Cheap to change later | Expensive — decide now |
| --- | --- |
| TUI framework and layout | `Finding` schema and evidence shape |
| Exporter/transport choice | Probe trait boundary and capability model |
| Detector set and thresholds | "One binary, deployment = packaging" |
| Recipe/advice content | Monotonic time model; wall clock only at the boundary |
| Alert routing | Host/boot identity |
| Packaging extras (deb, HA, Helm) | "No DB, no log files" invariant |
| Documentation | Detector purity (pure functions over recorded windows) |
| Dashboard integrations | `resource_keys` on findings (cascade seam) |
| | Read-only-by-default privilege model |

---

## 13. Minimum viable host

The agent is small, but it depends on kernel features. `doctor` probes all of these and states the
cost of each missing one; it never assumes.

| Requirement | Minimum | Recommended | If missing |
| --- | --- | --- | --- |
| Kernel | 5.10 | 6.1+ | Older kernels may lack PSI or cgroup v2; exit 3 |
| PSI | present (may need `psi=1`) | enabled by default | Fall back to proxy signals; severity marked a lower bound |
| cgroup v2 unified | preferred | unified | Per-unit attribution degraded to system-wide + per-PID where readable |
| `/proc` | mounted, not `hidepid=2` for the agent user | full | Exit 3 for per-process data |
| Architecture | aarch64, armv7, armv6, x86_64 | aarch64 (Pi 3+) | Unsupported binary |
| RAM | 256 MB | 1 GB | Risk of OOM under pressure; `doctor` warns |
| Free disk | 20 MB (binary only) | 100 MB | Cannot install |
| Time sync | any | chrony/NTP | Cross-host joins degrade; `skew_bound_ms` becomes conservative |

Raspberry Pi OS Bullseye (5.10) and Bookworm (6.1/6.6) are the reference platforms. Anything older is
best-effort and must say so.

---

## 14. Removal and data lifecycle

A diagnostic tool must be as easy to remove as it was to install.

```bash
sudo systemctl disable --now frictionpoint
sudo rm /etc/systemd/system/frictionpoint.service
sudo rm -rf /etc/frictionpoint.toml /etc/frictionpoint.d /var/lib/frictionpoint
sudo userdel frictionpoint
```

- There is **no database**, so there is nothing to migrate or export before removal.
- The optional spool is bounded and removed with `/var/lib/frictionpoint`; it is off by default.
- Findings live in memory and in whatever sinks the user configured; the agent keeps no history.
- Fixtures shipped in the repository are project artifacts, not user data.
- A future `frictionpoint uninstall --dry-run` will print these commands rather than run them.
