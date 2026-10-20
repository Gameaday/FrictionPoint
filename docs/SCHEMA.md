# SCHEMA v0 — the `Finding` contract

**Authority:** this file wins over every other document, on every project.
**Audience:** coding agents and reviewers. It is deliberately short.

A `Finding` is the only output that crosses a boundary: host → CLI, host → hub, Linux → Windows
tooling, future → past. Detectors, probes, UIs and exporters may change freely. This may not.

---

## 1. Versioning rules

| Rule | Meaning |
| --- | --- |
| `schema: "0"` today | Pre-1.0. `1` is frozen at the v1.0 release. |
| **Additive only** | New optional fields are allowed. Removing, renaming, or changing the meaning of a field is a breaking change and requires a new major. |
| **Readers ignore unknown fields** | Never `deny_unknown_fields` on findings. A newer agent must be readable by an older consumer. |
| **Consumers accept N-1** | A consumer must accept the previous major during a deprecation window. |
| **Platform-specific data is quarantined** | Anything Windows/Android-only goes in `extra`, never in a top-level field. |

---

## 2. Required fields

| Field | Type | Invariant |
| --- | --- | --- |
| `schema` | string | `"0"` |
| `id` | string(16) | Deterministic. See §4. |
| `kind` | string | Namespaced, lowercase, dot-separated: `io.small_write_storm`. Stable forever. |
| `detector` | `{id, version}` | `version` increments when behaviour changes. |
| `host` | `{id, boot_id, label}` | `id` stable across reboots; `boot_id` changes each boot. |
| `time` | object | Monotonic window + wall anchor + skew bound. See §3. |
| `severity` | `{stalled_ms, window_ms, coverage_pct}` | `0 <= stalled_ms <= window_ms`; `coverage_pct` in `0..=100`. |
| `confidence` | enum | `observed` \| `correlated` \| `traced`. |
| `impact` | enum | `self_stalled` \| `remote_wait` \| `degraded` \| `observed`. |
| `role_hint` | enum | `origin` \| `victim` \| `bottleneck` \| `unknown`. |
| `culprit` | object \| null | `null` is valid and must be preserved. See §5. |
| `resource_keys` | array of string | Stable join keys; may be empty. |
| `evidence` | array | **Never empty.** Each: `source`, `path`, `value`. |
| `advice` | array of string | Stable ids; may be empty. Never prose. |
| `visibility` | `{mode, degraded}` | What the agent could see; `degraded` lists missing capabilities. |
| `extra` | object | Platform-specific. Consumers ignore what they do not know. |

Optional: `episode_id` (string), `trace` (object, only when `confidence == "traced"`).

---

## 3. Time rules (non-negotiable)

```json
"time": {
  "mono_start_ns": 123456789,
  "mono_end_ns":   128456789,
  "wall_start": "2026-10-20T03:14:02Z",
  "wall_end":   "2026-10-20T03:14:07Z",
  "skew_bound_ms": 50
}
```

- All arithmetic uses `mono_*`. Wall clock exists only to join across hosts and to display.
- `skew_bound_ms` is required. Unknown means a conservative value (1000), not zero.
- **A sample interval is never a duration.** `stalled_ms` comes from cumulative counters over the
  window. If `coverage_pct < 100`, `stalled_ms` is a **lower bound** and the narrative must say so.
- Timestamps are RFC 3339 UTC with fixed width. No locale formatting, no local time in the document.

---

## 4. `id` algorithm (stable, dependency-free)

```text
id = hex16( fnv1a64( host.id || 0x1F || host.boot_id || 0x1F ||
                     detector.id || 0x1F || detector.version || 0x1F ||
                     window_start_ns || 0x1F || culprit_key ) )
```

- `window_start_ns` is decimal ASCII.
- `culprit_key` is `""` when there is no culprit, else `type:name` (e.g. `unit:traefik.service`).
- Separator is a single `0x1F` byte.
- FNV-1a 64 is specified so every implementation, on every platform, produces identical ids.
- **Test vectors are required** in `fp-core` for: no culprit, unit culprit, non-ASCII name, empty
  label. An id that changes between runs breaks dedupe, replay snapshots and hub joins.

---

## 5. Culprit

```json
"culprit": {
  "type": "unit",
  "name": "traefik.service",
  "detail": { "pid": 412, "comm": "traefik", "major_minor": "179:2", "cgroup": "/system.slice/traefik.service" }
}
```

- `type` states the attribution **granularity**, which is also the confidence floor: `unit` is
  reachable unprivileged; `process` needs root for other users' processes; file-level needs tracing.
- If the culprit exited, keep its name and set `detail.exited: true`. Never drop the finding.
- Never invent a culprit to fill the field. Unknown is `null`, and that is a legitimate diagnosis.
- `type` values: `unit` | `process` | `device` | `mount` | `driver`.

---

## 6. Evidence

```json
{ "source": "cgroup.io_stat",
  "path": "/sys/fs/cgroup/system.slice/traefik.service/io.stat",
  "value": "179:2 wios=570000 wbytes=2334720000" }
```

- `path` is the exact file read. `value` is the raw text or the exact numbers used.
- Evidence is how a user checks the arithmetic. A finding without evidence is a bug.
- Redaction happens before serialization; if a path is redacted, emit `"path": "<redacted>"` rather
  than omitting the item.
- `source` is a stable identifier (`psi`, `diskstats`, `cgroup.io_stat`, `procfs.io`, `kernel_log`,
  `vcgencmd`, `swaps`). New sources are additive.

---

## 7. Invariants an agent must not break

1. `evidence` is never empty.
2. `severity.stalled_ms <= severity.window_ms`.
3. `confidence == "traced"` requires a `trace` object. In v1.0 this is unreachable; do not emit it.
4. `id` is deterministic across runs and machines (same inputs -> same id).
5. Unknown fields are ignored on read, never required.
6. No floats for time. Integers only.
7. `kind` is stable: renaming one is a breaking change.
8. No absolute user paths unless the redaction tier allows it.
9. `culprit: null` is valid; consumers must handle it.
10. Windows/Android-only data lives in `extra`.
11. `severity.coverage_pct` is always present; omitting it hides unattributed work.
12. `visibility.degraded` is honest: never list a capability as present when the probe failed.

---

## 8. Fixture contract (separate version)

Fixtures record **raw probe output**, never findings, so detectors can be rewritten and re-scored.
`fixture_schema` is versioned independently of `schema`.

Format v0 is line-delimited JSON (optionally zstd-compressed as a whole):

```text
{"fixture_schema":"0","host":{"id":"...","boot_id":"..."},"kernel":"6.6.31",
 "arch":"aarch64","capabilities":["psi","cgroup_v2"],"started_ns":123456789}
{"t":123456789,"probe":"psi","path":"/proc/pressure/io","raw":"some avg10=..."}
{"t":123556789,"probe":"diskstats","path":"/proc/diskstats","raw":"179 0 mmcblk0 ..."}
{"end":true,"samples":2,"dropped":0}
```

Rules:

- `raw` is the exact text read. Never store derived values or findings.
- `t` is monotonic nanoseconds, taken at read time; ordering is authoritative.
- `dropped` records probe loss so replay can reproduce coverage, not invent it.
- Ground truth lives in a sidecar YAML (`expect: kind, culprit, min_confidence; forbid: [...]`).
- Replay must be deterministic and reproduce identical `id` values.
- Target size: a 2-minute capture is under 1 MB; larger captures must be downsampled or split.

---

## 9. What this file replaces

The `Finding` sketch in `docs/ARCHITECTURE.md` §3 is illustrative. **This file is the contract.**
If they disagree, this file wins and ARCHITECTURE gets fixed.
