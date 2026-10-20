# Central Exposure & Fleet Architecture

> **Decision up front:** build the *seams* now, the *service* later.
> Five schema fields and one identifier rule go into v0.1. A hub, an API, a UI, and peering are
> deferred, but nothing about them will require changing the agent's contract.
>
> **The invariant to protect:** the host agent stays a small, read-only, push-only, no-database
> process. Central capability lives in a **separate tier** with different constraints — not as
> feature flags bolted onto the agent.

---

## 1. The real problem is propagation, not aggregation

Your incident has three distinct roles, and almost every tool collapses them into one:

```text
  ORIGIN                     BOTTLENECK                      VICTIM
  Pi A: update floods logs -> Pi B: SD card saturates      ->  Pi C: ingress dead,
  (writes look normal)       (full I/O stall)                 users see nothing
```

- **Origin** — the host generating the load. Its own metrics may look *fine*. This is why the
  search fails: you look at the host that is sick, and the host that is sick is B, but B is not
  the cause.
- **Bottleneck** — the shared resource that gives way (a card, a disk, a network link, a
  database, a UPS).
- **Victim** — whoever feels it. May be a *fourth* host, a container, or a human on a laptop.

Central metrics (Grafana et al.) can show that A's log rate and B's disk latency rose together.
They cannot say "A caused B, and C is affected," because they have no model of **edges** or
**roles**. That is the gap a hub fills — and it is a *causality* product, not a dashboard.

### The honest scope limit

If the victim is a device you don't monitor (a phone, a smart TV, a guest laptop), no tool can
observe its experience. What a hub *can* do without victim telemetry is compute **blast radius**:
"B is degraded; 6 monitored services depend on B; 3 of them are already showing remote-wait
stalls." That is usually enough to act, and it is honest.

---

## 2. Topology options

| Option | What it is | Strengths | Weaknesses | Verdict |
| --- | --- | --- | --- | --- |
| **A. Per-host only** | Each box explains itself | Zero infra, zero privacy risk | Cannot see propagation; the founding incident stays invisible | v0.1 baseline |
| **B. Central metrics** (Prometheus/Grafana) | Scrape counters, alert on thresholds | Familiar, powerful querying | No roles, no edges, no narrative; you must already know what to ask | Not our product |
| **C. Central findings hub** | Agents push `Finding` docs; hub joins on time + resource keys | Tiny storage, causal chains, blast radius, works behind NAT | One more service to run | **Recommended core of fleet mode** |
| **D. Broker-as-hub** (MQTT retained) | No correlator; consumers read the bus | Almost zero build; HA-friendly; resilient | Correlation logic lands in every consumer; no history | v0.2 stepping stone |
| **E. Agent-to-agent peering** | Agents gossip findings directly | No central box, no single point | NAT traversal, clock trust, N² links, impossible to reason about | Reject |
| **F. Federated hubs** | Hub per site, hubs exchange signed summaries | Multi-site, privacy-preserving, scales | More moving parts; identity/trust design | Later, for multi-site |
| **G. Integration-only** | Emit to Home Assistant / Grafana / Uptime Kuma | Free distribution, no new UI | No cross-host joins; limited to what the target can express | Complementary, not a substitute |

**Recommendation:** A -> D -> C -> F. Never E. Use G throughout as distribution rather than as a
substitute for the hub.

Why C wins: a finding is 2-4 KB and self-contained. Fifteen hosts at a pessimistic 20 findings
per day is **~1.2 MB/day**. A hub is not a metrics database — it is a small event store plus a
graph plus a join window. It can run on a Pi Zero next to the broker, which is a genuinely novel
claim in this space.

---

## 3. The causal model

### 3.1 Roles and edges

Findings carry a local **impact** hint; the hub assigns **roles** across hosts.

```rust
// fp-core, present from v0.1 even though only one host consumes it at first
pub enum Impact {
    SelfStalled,     // this host lost wall-clock time
    RemoteWait,      // stalled while waiting on a dependency (NFS, DB, DNS, HTTP)
    Degraded,        // not stalled, but slower / retrying
    Observed,        // reported an event about a peer/shared resource without being hit
}

pub enum RoleHint { Origin, Victim, Bottleneck, Unknown }

pub enum EdgeKind {
    LogsTo,          // central log sink
    DependsOn,       // HTTP/gRPC/DB/DNS dependency
    SharedStorage,   // NFS, iSCSI, SMB, Ceph
    SharedPower,     // same PSU/UPS/brownout domain
    SharedThermal,   // same enclosure/fan
    NetworkPath,     // same switch/uplink/VLAN
    HostsService,    // this host runs a service others consume
}

pub enum Provenance { Declared, Discovered, Inferred }
```

`Provenance` is not decoration. It is the guardrail that keeps the hub from inventing causality:

| Provenance | Source | May support a causal claim? |
| --- | --- | --- |
| `Declared` | `topology.toml`, explicit config | **Yes** |
| `Discovered` | DNS/SRV, active connections, systemd/Docker dependencies, ARP/LLDP | **Yes**, but labelled |
| `Inferred` | Co-occurrence in time only | **No** — reported as "concurrent," never "caused" |

### 3.2 Hard guardrails

1. **No edge, no claim.** If A's writes cannot physically reach B, the hub may report "concurrent
   and unrelated." It may never blame A for B's stall.
2. **No time overlap, no claim.** Overlap is evaluated against each finding's clock uncertainty,
   not a naive timestamp comparison.
3. **Never promote correlation to causation silently.** The hub emits the same
   `observed` / `correlated` / `traced` tiers as the agent, and a cross-host chain is
   `correlated` unless a shared resource was directly traced on both sides.
4. **Origin is a hypothesis until the chain closes.** "A's log rate rose; B's io.stat attributes
   74% of requests to the receiver; the receiver's upstream connection is A" is a chain. "A and B
   both got busy" is not.

### 3.3 Propagation signatures worth modelling

| Path | Local fingerprint on the bottleneck | Local fingerprint on the victim | Edge |
| --- | --- | --- | --- |
| **Log centralization** | small-write storm from one receiver unit | — (victim may be off-host) | `LogsTo` |
| **Network storage** (NFS/iSCSI/SMB) | device stall *and* high network TX from the server | D-state tasks in `rpc_wait_*` / `nfs_*` wchan, `RemoteWait` | `SharedStorage` |
| **Service dependency** (DB/API) | DB host shows run-queue or I/O stall | client shows socket waits and latency, not CPU | `DependsOn` |
| **DNS/DHCP failure** | resolver host degraded | many hosts stall on name resolution simultaneously | `DependsOn` |
| **Network path saturation** | uplink `tx` queue, retransmits | transfers slow, no local disk pressure | `NetworkPath` |
| **Shared power** | undervoltage/throttle on multiple hosts at once | same | `SharedPower` |

**Important technical caveat:** PSI's `io` accounting does not reliably capture *every* kind of
remote wait (network-filesystem and socket waits may not enter the block-layer accounting path).
The agent therefore needs an **independent D-state census with `wchan`** so `RemoteWait` can be
detected even when PSI is quiet. Do not build the fleet story on PSI alone.

### 3.4 Victim detection without victim telemetry

A host is a likely **victim** when: it reports `SelfStalled` or `RemoteWait`, has **no credible
local culprit**, has at least one `Declared`/`Discovered` edge to a host that is simultaneously
stalled, and its wait channels point at that dependency. That is a four-condition test, not a
vibe, and it maps directly onto detectors the hub can run over findings.

---

## 4. What must go into the agent NOW (cheap now, migration later)

These are the only fleet-related things that belong in v0.1. None of them adds a dependency, a
thread, or a byte of network traffic.

### 4.1 Deterministic, globally unique `finding_id`

```text
finding_id = H(host_id || boot_id || detector_id || detector_version
               || window_start_ns || culprit_key)[..16]
```

Properties that matter later:

- **Idempotent delivery.** A retried MQTT publish must not create a duplicate episode.
- **Dedupe across redundant hubs.** Two hubs receiving the same finding converge.
- **Reproducible replay.** Re-running a fixture produces the same ids, so snapshots are stable.

A random UUID destroys all three. Use a hash.

### 4.2 Stable `host_id` — and do not trust `/etc/machine-id` alone

This is a real SBC trap: **Pis flashed from the same image can share `/etc/machine-id`.** Fifteen
clones can collapse into one host identity, silently merging unrelated findings.

Resolution order:

1. DMI/board serial (Pi: `/proc/device-tree/serial-number` or `/sys/firmware/devicetree/base/serial-number`).
2. SMBIOS UUID (`/sys/class/dmi/id/product_uuid`) on PCs/servers.
3. A random id generated once and persisted to `/var/lib/frictionpoint/host-id`.
4. `/etc/machine-id` — **last resort**, and warn if it is the only source.

Also record `boot_id` (`/proc/sys/kernel/random/boot_id`) so "restarted" and "was stalled" are
distinguishable, and `host_label` from config for display.

### 4.3 Clock provenance on every finding

Cross-host correlation is a time-join, and unexamined time assumptions are the #1 source of false
causality.

```rust
pub struct TimeWindow {
    pub mono_start_ns: u64,      // local, monotonic, comparable within a boot only
    pub mono_end_ns: u64,
    pub wall_start: SystemTime,  // anchored at sample time
    pub wall_end: SystemTime,
    pub skew_bound_ms: u32,      // from chrony/NTP tracking, else conservative default
}
```

Rules:

- All internal math uses `mono_*`. Wall clock exists only to join across hosts and to display.
- `skew_bound_ms` comes from the time daemon when available (`chronyc tracking`, `ntpq -c rv`,
  `timedatectl`). If unknown, assume a conservative bound (e.g. 1000 ms) and say so.
- Causal joins require overlap **after** expanding each window by its own skew bound. If the
  overlap disappears under expansion, the hub reports "concurrent, ordering unknown."
- A chain that depends on sub-second ordering across hosts with 1 s skew is not a chain.

### 4.4 `impact` and `role_hint`

Local evidence can distinguish "I stalled" from "I stalled while waiting on something I can't
see" — and that distinction is the entire victim-detection signal (§3.4). `RemoteWait` requires
the D-state/`wchan` census, which is already planned for other detectors.

### 4.5 `resource_keys`

Namespaced, stable, resolvable back to an entity on the origin host:

```text
host:pi-lab-3
unit:traefik.service@pi-central
device:mmcblk0p2@pi-central
service:postgres@db-1
sink:logs@pi-central
```

The hub joins on these. Adding them later means re-collecting history; adding them now is a
`Vec<String>` in one struct.

### 4.6 Forward compatibility

- `schema_version` on every finding document; hub accepts **N-1**.
- Serde: **never** `deny_unknown_fields` on finding types. Unknown fields are how a newer agent
  talks to an older hub without a coordinated upgrade.
- Evidence items are `(source, path, value)` strings so new sources need no schema change.

---

## 5. The hub: a separate tier with different constraints

### 5.1 Carve-out: the no-DB rule is for the agent, not the hub

ADR-0002 forbids a database **on the monitored host**, because it creates the write amplification
the tool exists to find. The hub is a different tier: it is not being diagnosed, it usually runs
on a NAS or VPS, and it needs history to answer "what changed?".

So: **agent = no DB, no log files, no listeners. Hub = may use an embedded store, bounded
retention, and a read-only API.** Stating this now avoids both dogma-driven pain ("the hub can't
remember anything") and scope creep ("the agent needs SQLite").

### 5.2 Hub storage and retention

- Default: append-only **compressed JSONL** of findings + an in-memory index; optional SQLite for
  query convenience, never required.
- Retention by whichever comes first: **14 days** or **200k findings**, then rotate (and keep
  episode summaries longer than raw findings).
- Episodes and causal chains are **derived, not authoritative** — they can always be recomputed
  from retained findings, which means detector/reasoner upgrades can re-score history.
- Cap and drop-oldest on pressure. Never block an ingest because storage is full, and alert on it.

### 5.3 Read-only API (the anti-lock-in contract)

```text
GET /v1/hosts                       # identity, last-seen, role hints
GET /v1/findings?since=&host=&kind=&min_severity=
GET /v1/episodes?since=&host=
GET /v1/episodes/{id}               # chain, roles, blast radius, evidence refs
GET /v1/graph                       # nodes, edges, provenance
GET /v1/stream                      # SSE, live findings/episodes
POST /v1/ingest                     # optional push path (MQTT is primary)
```

Rules: versioned, JSON, **read-only except ingest**, no query language, no arbitrary aggregation.
The CLI and any future pane consume this same API — that is what keeps the UI optional.

### 5.4 Agent back-pressure (design it now)

When the hub is down or slow:

- buffer a bounded ring (default 256 findings), drop **oldest**, never block the probe loop;
- never retry in a tight loop, never log-spam; a single line per outage window;
- on reconnect, send a `gap` marker so the hub knows coverage is incomplete rather than
  silently assuming health.

The `gap` marker matters more than it looks: a missing finding and a clean host are not the same
thing, and the hub must never conflate them.

---

## 6. The pane: five questions, not ten charts

A central pane earns its place only if it answers questions a per-host tool cannot. It should
open on this, in this order:

1. **What is broken right now?** Episodes ranked by `severity x blast radius`, not by hostname.
2. **Who is affected?** The dependency graph with the blast radius highlighted; victims listed
   even when they have not reported yet.
3. **Why?** The causal chain, hop by hop, each hop showing its evidence and provenance
   (`declared` / `discovered` / `inferred`).
4. **What changed?** First-seen finding kinds, new offenders, hosts that stopped reporting.
5. **What do I do?** One recommended action per episode, linked to the host's `explain` output.

Plus one visual that does the emotional work: a **fleet grid** — one tile per host, coloured by
current lost-time percentage, with a connecting line animated only when a correlation survives the
skew guardrail. Fifteen tiles is instantly legible; fifteen dashboards is not.

Anti-features, on purpose: no metric explorer, no arbitrary query builder, no user-defined
dashboards, no long-term analytics, no alert-rule DSL. Those are Grafana's job, and competing there
is how this project loses its identity.

Form factor: ship `frictionpoint fleet --hub URL` (ratatui) first — it is a day of work and
proves the API. A web pane is a separate crate and a separate release, consuming only the public
API, so the core never grows a web server by accident.

---

## 7. Privacy is a fleet-level escalation

Individual findings contain process names, unit names, and file paths. Centralizing them turns
per-host data into a network-wide dataset, which is a different risk class than a local CLI.

| Tier | What leaves the host | Use |
| --- | --- | --- |
| **0 - metadata** | host, kind, severity, resource keys, timestamps | Untrusted or shared hubs; enough for cascade + blast radius |
| **1 - + names** | unit/process/device names | Default for a trusted home network |
| **2 - + paths** | evidence paths | Debugging, opt-in per host |
| **3 - + raw evidence** | full evidence values | Only on a hub you own; default for local-only mode |

Design rules:

- Redaction happens **on the agent**, before the sink. The hub cannot un-receive data.
- Redaction is **per sink**: local JSONL can be tier 3 while MQTT is tier 1.
- The config declares the tier explicitly; there is no "smart" inference. A warning is printed
  the first time a remote sink is enabled above tier 0.
- Path hashing is available (stable per host) so the hub can group without knowing the path.

---

## 8. Peering and federation (for later, not now)

- **Never agent-to-agent.** With 15 devices that is 105 links, each with its own NAT and clock
  assumptions, and no place to reason about causality.
- **Hub-to-hub only.** Two homes, or home + cloud, exchange **signed summaries** (episodes and
  graph edges, not raw findings). Each hub keeps its own detail, so privacy is preserved by
  construction and bandwidth stays tiny.
- Trust is explicit: a peer relationship is configured with a public key and a direction
  (one-way or two-way). No auto-discovery, no transitive trust.
- Dedupe by `finding_id` and `episode_id`; an episode seen by both hubs keeps the higher-confidence
  version and a note that it was corroborated.
- Clock skew between sites is larger than within a site; federated joins default to a coarser
  window (seconds, not milliseconds) and say so.

---

## 9. Failure modes to design for now

| Failure | Consequence without foresight | Design response |
| --- | --- | --- |
| Hub down | Agents block, drop data, or spam logs | Bounded ring, drop oldest, single log line, `gap` marker on reconnect |
| Clock skew | Fabricated causal chains | `skew_bound_ms` per finding; expand windows; degrade to "concurrent" |
| Two hubs (split brain) | Duplicate episodes | Deterministic `finding_id`; idempotent ingest |
| Cloned Pi images | Many hosts collapse into one `host_id` | Board serial first; persisted random id; machine-id last |
| Host reflash | Identity changes, history orphans | `host_id` persisted outside the rootfs where possible; document the reset |
| Hub disk full | Silent data loss | Cap + drop oldest + alert; never block ingest |
| Sensitive paths centralized | Privacy incident | Agent-side redaction tiers, explicit opt-in |
| Pane compromise | Remote control of the fleet | Pane is read-only; no remediation path through the hub in v1 |

---

## 10. Sequencing (and what NOT to build)

| Stage | Scope | Gate to proceed |
| --- | --- | --- |
| **v0.1** | Schema seams only: deterministic id, identity, time provenance, `impact`/`role_hint`, `resource_keys`, `schema_version` | Single-host detectors are trustworthy |
| **v0.2** | One-way export (`mqtt`, `webhook`); no hub | Users are running 3+ hosts |
| **v0.3** | `hub` subcommand: ingest, retention, `/v1` read API, `fleet` TUI | Users ask "which host caused this?" |
| **v0.4** | Causal reasoner: roles, edges, episodes, blast radius | `topology.toml` exists and is populated |
| **v0.5** | Web pane (separate crate) | The TUI has been used for a month |
| **v0.6** | Hub-to-hub federation | A real multi-site user exists |

**Do not build now:** hub, API, auth, web UI, topology auto-discovery, peering, remote
remediation, a query language, or a metrics store. Every one of them is downstream of a schema
that does not exist yet.

---

## 11. Open questions

1. **Hub placement:** Pi next to the MQTT broker, NAS, or a small VPS? (Affects whether the hub
   ships as an OCI image first or a binary.)
2. **Pane form:** TUI-first is cheap and proves the API; is a web pane a launch feature or a
   post-1.0 feature?
3. **Auth:** single shared token, per-host tokens, or mTLS from day one? (mTLS is more work but
   removes a class of problems.)
4. **Home Assistant:** publish findings as MQTT events with discovery, or only as a sensor count
   with attributes? The former is nicer and noisier.
5. **Multi-site:** is federation a real requirement for you, or a hypothetical? It changes how
   much identity/trust work goes into v0.3.
