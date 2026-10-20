# Metrics catalogue — what is worth investigating, per platform

**Purpose:** decide *what to measure* without letting the examples become the scope. The unit of
design is a **friction mechanism**, not a metric. "USB throughput" is one instance of *shared-bus
allocation*; the same mechanism explains a saturated PCIe lane, an oversubscribed NFS mount, or a
saturated 1 GbE link.

## Selection rule (what earns a place)

A metric is included only if **all three** hold:

1. **It maps to a mechanism** in §1 — it explains a halt, hitch, drop, or misbehaviour, not just load.
2. **It has a cheap source** — a kernel file, an ETW provider, or a `dumpsys` call. Not a profiler.
3. **It is underserved** — today it requires expert tooling, root, or custom scripts to see, and no
   mainstream tool turns it into a sentence.

A metric that fails (1) is trivia. A metric that fails (2) is a profiler. A metric that fails (3) is
someone else's product, and we should export to it rather than reimplement it.

**Corollary:** every included metric must eventually have a detector, a narrative, and a negative test.
A probe with no consumer is dead weight and will be deleted.

---

## 1. The mechanism taxonomy

This is the part that generalises. Ten mechanisms cover almost every "my machine misbehaves" report.

| # | Mechanism | The question it answers | Typical evidence |
| --- | --- | --- | --- |
| **M1** | **Contention** | What is waiting, and on what? | PSI stall time, run-queue wait, D-state census, blocked-share |
| **M2** | **Saturation** | What queue is full? | in-flight I/O, queue depth, softnet backlog, send-Q |
| **M3** | **Tail latency** | Is the average fine while the p99 is awful? | latency histograms, service-time buckets |
| **M4** | **Amplification** | Why does small work cause huge cost? | request size vs bytes, write amplification, retransmits, fsync storms |
| **M5** | **Fairness / throttling** | Who is being artificially limited? | cgroup `cpu.stat` throttling, `io.max`, `memory.high`, QoS |
| **M6** | **Errors & retries** | What is failing and recovering invisibly? | `mmc` timeouts, USB resets, TCP retransmits, NIC CRC errors |
| **M7** | **State transitions** | What breaks at boundaries? | sleep/wake, hotplug, autosuspend, remount-ro, thermal trip |
| **M8** | **Topology / allocation** | Is the layout the problem? | USB tiers and bus budget, IRQ affinity, NUMA, shared host |
| **M9** | **Degradation** | What is wearing out or being limited? | card wear, undervoltage, thermal throttle, SMR write stalls |
| **M10** | **Shared-service starvation** | What dependency silently gates everything? | DNS, auth, log sink, NFS, database |

Two derived rules the whole product depends on:

- **Latency is charged per request, not per byte** (M4). Byte-based blame is wrong for fsync storms.
- **A sample interval is not a duration** (M1/M3). Lost time comes from cumulative counters.

---

## 2. Exposure legend ("underserved" made checkable)

| Rating | Meaning |
| --- | --- |
| **None** | No mainstream tool surfaces it. Requires custom scripts or expert tracing. |
| **Expert** | Visible only through expert CLI (`perf`, `bpftrace`, `iostat -x`, `nstat`, `nfsstat`, `ethtool -S`). |
| **Partial** | Visible in a GUI or vendor tool, but without attribution or a plain-language verdict. |
| **Served** | Commonly shown by default (CPU %, memory used). We do not compete here. |

A catalogue where most rows are **None** or **Expert** is a product. Most of the rows below are.

---

## 3. Shared vocabulary across platforms

The `Finding` contract already normalises these. When a metric does not fit, the mechanism is wrong,
not the schema.

| Concept | Linux | Windows | Android |
| --- | --- | --- | --- |
| stall time | PSI `full`/`some` | DPC/ISR + wait chains, `% DPC Time` | PSI where available, frame/jank proxies |
| attribution unit | cgroup / systemd unit | process, service, driver | UID / package |
| request latency | block layer / `diskstats` | ETW FileIo + StorPort durations | `storaged`, stalled-diskstats events |
| queue depth | `in_flight` | `Current Disk Queue Length` | limited |
| throttling | cgroup `cpu.stat`, `io.max` | EcoQoS / power throttling | thermal + LMKD |
| topology | USB sysfs, IRQ affinity | USB node tree, PCI | USB port state (limited) |
| errors | kernel log | ETW + System log | `logcat -b events`, `dumpsys` |

---

## 4. Linux / SBC

The richest platform, because the kernel exposes mechanisms rather than aggregates. Most rows are
**None** or **Expert**: the information exists and nobody turns it into a sentence.

### 4.1 I/O and storage

| Metric | Mech | Source (illustrative) | Exposure | Phase |
| --- | --- | --- | --- | --- |
| PSI `io` some/full + cumulative total | M1 | `/proc/pressure/io` | **None** | v1.0 |
| Per-cgroup `wios`/`wbytes`/`rios` + mean request size | M4/M5 | `/sys/fs/cgroup/**/io.stat` | **None** | v1.0 |
| Device in-flight, weighted time, service deltas | M2/M3 | `/proc/diskstats` | Expert | v1.0 |
| Per-task block-I/O wait (`delayacct_blkio_ticks`) | M1 | `/proc/<pid>/stat` field 42 | **None** | v1.x (armed) |
| Dirty/writeback pages vs `dirty_ratio` | M4 | `/proc/vmstat`, `/proc/sys/vm/*` | Expert | v1.0 |
| Direct reclaim / writeback stalls | M2 | `vmstat`, PSI | **None** | v1.x |
| ext4 journal commit rate and size | M4 | `/proc/fs/jbd2/*/info` | Expert | v1.x |
| `fsync`/`sync_file_range` rate | M4 | eBPF tracepoint | **None** | v1.x (`trace`) |
| FUSE waiting requests and congestion | M1/M3 | `/sys/fs/fuse/connections/*/waiting` | **None** | v1.x — rclone/sshfs/gvfs stalls |
| NFS client RPC latency and retransmits | M10 | `/proc/net/rpc/nfs` | Expert | post-1.0 |
| Queue depth, scheduler, `rotational`, physical block size | M8 | `/sys/block/*/queue/*` | **None** | v1.x — misalignment and queue tuning |
| Filesystem capacity / inode exhaustion | M6 | `statvfs`, kernel log | Partial | v1.1 |
| Discard/TRIM backlog | M4 | btrfs sysfs, kernel log | **None** | post-1.0 |
| eMMC `life_time`, SD manufacture date, CID | M9 | `/sys/block/mmcblk*/device/*` | **None** | v1.0 |
| `mmc` timeouts, CRC errors, resets | M6 | kernel log | Expert | v1.0 |
| `remount-ro`, ext4/xfs errors | M7 | kernel log + `/proc/mounts` | Expert | v1.0 |
| NVMe `percentage_used`, media errors, unsafe shutdowns | M9 | `nvme` ioctl (root) | Expert | post-1.0 |
| SMR write stalls (NAS drives) | M9 | model + latency-at-low-queue signature | **None** | post-1.0 |

Why this matters: an SD card at 30% "utilization" can add 400 ms per request. `%util` is not a
latency metric, and on flash it is actively misleading. The useful pair is **queue depth + latency
distribution + request size**, which is why M3 and M4 sit next to M2.

### 4.2 Memory

| Metric | Mech | Source | Exposure | Phase |
| --- | --- | --- | --- | --- |
| PSI `memory` full/some | M1 | `/proc/pressure/memory` | **None** | v1.x |
| `MemAvailable` (not `MemFree`) | M2 | `/proc/meminfo` | Partial | v1.x |
| Direct reclaim, pages scanned/stealed, major faults | M1/M2 | `/proc/vmstat` | Expert | v1.x |
| Swap in/out rate | M4 | `/proc/vmstat` | Expert | v1.x |
| Swap on SD card | M9 | `/proc/swaps` | Partial | **v1.0** |
| zram compression ratio, same-filled pages | M4 | `/sys/block/zram*/mm_stat` | **None** | v1.x |
| cgroup `memory.events` high/max/oom_kill | M5 | `memory.events` | **None** | v1.x |
| OOM kills and their victims | M7 | kernel log, cgroup events | Partial | v1.x |
| Slab / dentry / inode pressure | M1 | `/proc/slabinfo`, `/proc/meminfo` | **None** | post-1.0 |
| THP compaction stalls | M1 | `vmstat compact_stall` | **None** | post-1.0 |

Why this matters: `MemFree` is the number users quote and the number that lies, because page cache is
reclaimable. `MemAvailable` plus PSI `memory full` is the honest pair. On a 1 GB Pi, `memory.events`
`max` counts are the difference between "slow" and "silently thrashing".

### 4.3 CPU, scheduler, and interrupt pressure

| Metric | Mech | Source | Exposure | Phase |
| --- | --- | --- | --- | --- |
| PSI `cpu` some | M1 | `/proc/pressure/cpu` | **None** | v1.x |
| Per-task run-queue wait | M1 | `/proc/<pid>/schedstat` | **None** | v1.x |
| Involuntary context switches | M1 | `/proc/<pid>/status` | Expert | v1.x |
| CPU **steal** time (VM / noisy neighbour) | M1 | `/proc/stat` | Partial | v1.x — the VPS story |
| cgroup CPU throttling (`nr_throttled`, `throttled_usec`) | **M5** | `cpu.stat` | **None** | v1.x — top-3 underserved signal |
| IRQ / softirq pressure | M1 | `/proc/pressure/irq` where present, `/proc/softirqs` | **None** | post-1.0 |
| Per-IRQ counts and affinity | M8 | `/proc/interrupts`, `/proc/irq/*/smp_affinity` | Expert | post-1.0 |
| cpufreq transitions, current limit | M9 | `cpufreq/stats/time_in_state` | Expert | post-1.0 |
| RAPL energy per cgroup | adjacent | `/sys/class/powercap` | Expert | not planned |

**`nr_throttled` deserves emphasis.** A container can be at 20% average CPU and still add 200 ms to
p99 because CFS throttles it every 100 ms period. Almost no user-facing tool surfaces this, and it is
the single most common cause of "the service is fast except sometimes" in containers.

### 4.4 Power, thermal, degradation

| Metric | Mech | Source | Exposure | Phase |
| --- | --- | --- | --- | --- |
| Undervoltage / throttle history (Pi) | M9 | `vcgencmd get_throttled` bitmask | **None** | **v1.0** |
| Thermal zone temperatures vs trip points | M9 | `/sys/class/thermal/*` | Partial | **v1.0** |
| eMMC wear class, SD age | M9 | sysfs | **None** | **v1.0** |
| Battery / power-supply health | M9 | `power_supply` class | Partial | post-1.0 |
| Suspend/resume counters, wake sources | M7 | `/sys/power/*`, kernel log | Expert | post-1.0 |

### 4.5 Network

| Metric | Mech | Source | Exposure | Phase |
| --- | --- | --- | --- | --- |
| TCP retransmits | M4 | `/proc/net/snmp` (`TcpRetransSegs`) | Expert | post-1.0 |
| Socket errors, listen overflows | M6 | `nstat`, `/proc/net/netstat` | Expert | post-1.0 |
| Softnet drops and `time_squeeze` | **M2** | `/proc/net/softnet_stat` | **None** | post-1.0 |
| qdisc drops | M2 | `tc -s` | Expert | post-1.0 |
| Conntrack table exhaustion | M6 | kernel log + `nf_conntrack_count` | Expert | post-1.0 (routers) |
| NIC CRC / link / carrier errors | M6 | `ethtool -S` | Expert | post-1.0 |
| Wi-Fi retries, signal, rate drops | M6/M9 | `/proc/net/wireless`, `iw` | Expert | post-1.0 |
| DNS resolution latency | **M10** | not directly observable | **None** | out of scope (app-level) |

"Network is slow" is usually one of three unrelated things: retransmits (M4), a full queue (M2), or a
blocked dependency that happens to use the network (M10). Naming which one is the value; measuring
throughput is not.

### 4.6 Kernel health events

| Metric | Mech | Source | Exposure | Phase |
| --- | --- | --- | --- | --- |
| Hung task, soft lockup, RCU stall | M7 | kernel log | Expert | v1.x |
| ext4/xfs/JBD2 errors | M6 | kernel log | Expert | **v1.0** |
| OOM kills and victims | M7 | kernel log | Partial | v1.x |
| USB disconnect / reset / overcurrent | M6 | kernel log | Expert | v1.x |
| Kernel taint flags | adjacent | `/proc/sys/kernel/tainted` | **None** | post-1.0 |
| Boot and shutdown duration | adjacent | `systemd-analyze` | **Served** | not planned |

These are the events that explain a **whole-system freeze** with no resource spike. A five-second
`soft lockup` or a `hung_task` warning is a complete diagnosis that almost no user ever sees, because
it lives in a log they never read.

### 4.7 USB and peripherals — the underserved domain

USB is not a pipe with a throughput number. It is a **scheduled bus with per-endpoint reservations**,
frame/microframe timing, and a topology of tiers behind shared controllers. This is why "USB throughput"
is the wrong headline metric: a device can reserve most of a bus while moving almost no data, and the
failure appears when a *second* device asks for a slice.

| Metric | Mech | Source | Exposure | Phase |
| --- | --- | --- | --- | --- |
| Topology: controller -> hubs -> ports -> devices; tier depth; negotiated speed | **M8** | `/sys/bus/usb/devices/*` | **None** | v1.x |
| Endpoint descriptors: transfer type, `wMaxPacketSize`, `bInterval` | M8/M4 | `/sys/kernel/debug/usb/devices` (root) or usbfs | **None** | v1.x |
| **Reserved bandwidth per endpoint** (derived) | **M8** | derived from descriptors | **None** | v1.x |
| URB submission rate (`urbnum` delta) | M2 | `.../urbnum` | **None** | v1.x |
| Actual per-URB size and interval | M2/M4 | `usbmon` (root) | **None** | post-1.0 (`trace`) |
| Autosuspend policy and runtime state | **M7** | `power/control`, `power/runtime_status` | **None** | v1.x — classic random disconnect |
| Resets, disconnects, descriptor-read errors, over-current | **M6** | kernel log | Expert | v1.x |
| Port power budget (`bMaxPower` sum vs controller) | M8 | sysfs | **None** | v1.x |
| xHCI interrupt distribution / MSI count | M8 | `/proc/interrupts`, `/proc/irq/*` | Expert | post-1.0 |
| USB 3 link errors (LTSSM, CRC, retries) | M6 | kernel log | Expert | post-1.0 |

**The derived metric that matters:**

```text
reservation(endpoint) ~= wMaxPacketSize * (1 / bInterval)
bus_reservation_ratio = sum(reservation) / negotiated_bus_capacity
```

For interrupt and isochronous endpoints this is a *reservation*, not an average. A webcam can hold a
large fraction of a high-speed bus before it sends a single extra byte; the camera "worked until it
needed a hair more" because the reservation crossed the remaining headroom and the hub renegotiated.
That is M8, not M2, and no consumer tool shows it.

Also note the fallback trap: a USB 3 device plugged into a USB 2 port or hub negotiates down, so the
same topology that was fine yesterday saturates today. Report **negotiated speed vs device capability**,
not just link speed.

On Linux, `usbmon` is the only source of actual transfer sizes, and it needs root. On Windows, ETW
gives resets and errors but not per-endpoint bandwidth; derive reservations from descriptors instead.
On Android, `dumpsys usb` exposes port state but no bandwidth — say so rather than pretend.

---

## 5. Windows

ETW is the source of truth; PDH gives cheap aggregates; the **Event Logs are the underserved part** —
Windows already writes plain-language performance diagnoses that almost nobody reads.

| Metric | Mech | Source | Exposure | Phase |
| --- | --- | --- | --- | --- |
| Per-process file I/O with durations | M3 | ETW `Kernel-File` / `StorPort` | Expert | post-1.0 |
| Physical disk service time and queue length | M2/M3 | PDH `Avg. Disk sec/Transfer`, `Current Disk Queue Length` | Partial | post-1.0 |
| DPC/ISR time **per driver** | M1/M3 | ETW `PerfInfo` DPC/ISR with stacks; PDH `% DPC Time` | Expert | post-1.0 |
| Context switches, processor queue length | M1 | PDH | Partial | post-1.0 |
| Hard page faults, `Pages/sec`, available MBytes | M2 | PDH | Partial | post-1.0 |
| Memory exhaustion events | M7 | `Resource-Exhaustion-Detector` log | **None** | post-1.0 |
| Sleep/resume transitions, wake source, resume duration | M7 | `Kernel-Power`, `Power-Troubleshooter` | Partial | post-1.0 |
| NDIS miniport reset, link state, TCP retransmits | M6/M4 | ETW `NDIS`, `Kernel-Network`; PDH `Segments Retransmitted/sec` | Expert | post-1.0 |
| USB port reset, descriptor failure, over-current | M6 | ETW `USB-USBPORT`, `USB-USBHUB3`, `USB-USBXHCI` | Expert | post-1.0 |
| USB topology, negotiated speed, port power | M8 | SetupAPI node tree | **None** | post-1.0 |
| Power throttling / EcoQoS | M5 | ETW `Kernel-Processor-Power` | **None** | post-1.0 |
| Hardware errors (WHEA: correctable/uncorrectable) | M9 | `Microsoft-Windows-WHEA-Logger` | **None** | post-1.0 — failing hardware, invisible |
| Boot/shutdown/standby degradation verdicts | M9 | `Diagnostics-Performance/Operational` | **None** | post-1.0 |
| Driver/update install timeline (for correlation) | M10 | Setup log | Partial | post-1.0 |
| ETW lost events / buffer overruns | meta | session statistics | **None** | post-1.0 — never lie about coverage |
| GPU engine utilization | adjacent | PDH `GPU Engine` | Partial | **not ours** (PresentMon owns frame timing) |
| Audio glitches | adjacent | ETW `Audio` | Expert | post-1.0, if tractable |

**The Windows wedge is M7 + M8 + M9**: post-sleep recovery, USB resets, and hardware degradation.
Frame timing is PresentMon's; per-process profiling is WPA's. We turn existing ETW into a verdict.

---

## 6. Android

Unrooted visibility is genuinely limited. The honest product is an **advisor** using platform services,
not a tracer. Rows marked *root* are explicitly out of the unrooted tier.

| Metric | Mech | Source | Exposure | Phase |
| --- | --- | --- | --- | --- |
| Per-UID read/write bytes and **fsync counts** | M4 | `dumpsys storaged --hours 24`, `/proc/uid_io/stats` | **None** | post-1.0 |
| Stalled-diskstat / storage latency histograms | M3 | storaged event log where supported | **None** | post-1.0 (device-dependent) |
| Lifetime writes / flash wear estimates | M9 | `dumpsys diskstats` | **None** | post-1.0 |
| Per-app PSS, swap, reclaim | M2 | `dumpsys meminfo`, `dumpsys procstats` | Partial | post-1.0 |
| LMKD kills and kill reasons | M7 | `logcat -b events` (`lmk`, `am_kill`) | Expert | post-1.0 |
| Thermal status / throttling | M9 | `dumpsys thermalservice` | Partial | post-1.0 |
| Battery temperature and health | M9 | `dumpsys battery` | Partial | post-1.0 |
| Job storms and wakeups | M4/M10 | `dumpsys jobscheduler`, `deviceidle` | **None** | post-1.0 |
| Per-UID network bytes | M4 | `dumpsys netstats` | Expert | post-1.0 |
| USB port state and role | M8 | `dumpsys usb` | Partial | post-1.0 — **no bandwidth available** |
| PSI memory/cpu/io | M1 | `/proc/pressure/*` where readable | **None** | post-1.0 |
| Per-UID CPU time | adjacent | `dumpsys cpuinfo` | Partial | post-1.0 |
| Frame jank | adjacent | `gfxinfo`, Perfetto | Partial | **not ours** |

Android's unique advantage is that `storaged` already attributes requests **and fsync counts** per UID —
exactly the M4 signal that is invisible on Linux without eBPF. Its limitation is latency: the platform
records bytes and counts, not per-request service time, and unrooted access to `/proc/<pid>` is gone.

---

## 7. Adjacent platforms and use cases (why the taxonomy matters)

These are not separate products we are committing to. They are evidence that the mechanism list
generalises, and a guard against over-fitting the examples.

| Context | Same mechanisms | Highest-value metrics | Implication |
| --- | --- | --- | --- |
| **VPS / cloud** | M1, M2, M9 | CPU **steal**, host I/O latency, ballooning, noisy-neighbour disk | The Linux agent already works; the narrative is "not your fault, your host" |
| **Containers / K8s** | M5, M1, M2 | cgroup `cpu.stat` throttling, `memory.events`, `io.stat`, PSI | Top-3 underserved; consume native K8s PSI, do not re-collect |
| **NAS / file server** | M2, M3, M9 | NFS/SMB RPC latency, ZFS ARC hit rate, **SMR write stalls**, spin-up | A distinct product; same "tail latency under saturation" logic |
| **Router / OpenWrt** | M2, M6 | conntrack exhaustion, softnet drops, Wi-Fi retries, bufferbloat | Different product; do not absorb into core |
| **Steam Deck / gaming Linux** | M1, M9 | PSI, GPU clock/thermal, shader compile stalls | Same agent, different narratives |
| **CI runners** | M1, M2, M5 | disk saturation, CPU steal, cgroup throttling | "Why is CI slow" is the same question |
| **macOS / BSD** | M1, M3 | `powermetrics`, `fs_usage`, DTrace | Not planned; no PSI equivalent, poor unprivileged access |

If a proposed metric does not fit any mechanism here, it is probably a profiler feature or a dashboard
metric, and it should be exported rather than built.

---

## 8. Metric domains we deliberately do not own

| Domain | Owner | Why not us |
| --- | --- | --- |
| Frame/present timing, input latency | PresentMon, CapFrameX | Mature, graphics-specific, high-frequency |
| Application profiling / flame graphs | `perf`, WPA, Perfetto | Different granularity and skill level |
| GPU utilisation and VRAM | vendor tools, PDH | Adjacent; export if needed |
| Security / intrusion attribution | Falco, Tetragon, Sysdig | Requires privileged tracing by design |
| Battery-life modelling | vendor tools | Needs long-horizon app data |
| Benchmark scores | Phoronix, Geekbench | Synthetic, not diagnostic |

---

## 9. Metric naming (stable ids)

Evidence `source` strings and metric names are part of the contract's vocabulary. Use
`<domain>.<mechanism>.<metric>`:

```text
io.pressure.full          io.cgroup.wios          io.disk.in_flight
io.request.mean_bytes     io.fuse.waiting          io.emmc.life_time
mem.reclaim.direct        mem.cgroup.max_events    mem.zram.ratio
sched.throttle.usec       sched.runqueue.wait      sched.steal.usec
irq.pressure.some         irq.affinity.spread
usb.reserve.ratio         usb.urb.rate             usb.reset.count
usb.autosuspend.state     usb.power.budget
win.dpc.time              win.disk.queue           win.usb.reset
win.power.resume_ms       win.whea.correctable
android.storaged.fsync    android.uid.wbytes      android.lmk.kill
android.thermal.status    android.job.wakeup
```

Platform-only metrics are prefixed (`win.`, `android.`). Shared mechanisms keep the same name across
platforms so findings and narratives can be compared. Renaming one is a breaking change.

---

## 10. Decisions (based on user experience and common repo practice)

| Decision | Choice | Rationale |
| --- | --- | --- |
| **Repo layout** | `frictionpoint` (Linux, this repo), `frictionpoint-win`, `frictionpoint-android` | The suffix convention is familiar; separate repos keep each agent's context small and release cadence independent. Create each only when the previous one is useful. |
| **Schema location** | Stay in `docs/SCHEMA.md` + `schema/finding.schema.json` in this repo until a second consumer exists | Common practice is to extract a schema package only when there are two consumers; earlier extraction creates version churn and a second copy to drift. |
| **Fixtures in git** | Commit `.fpz` files when < 2 MB each and < 20 MB total; LFS beyond that | Matches GitHub's own guidance; keeps clone cheap. Windows `.etl` is never committed — it is large and machine-specific; commit slimmed JSON plus the provider list. |
| **Android delivery** | ADB-pushed CLI first; app later, if ever | How people already inspect Android (`dumpsys`, Perfetto). An app adds permissions, background restrictions, and Play policy for no diagnostic gain. |
| **Windows delivery** | Single binary, ETW session, no kernel driver; `doctor` checks admin / Performance Log Users | Same requirement PresentMon has. Failing early with an actionable message beats a silent empty trace. |
| **CLI verbs** | `why` (recent episode in a sentence), `explain <id>` (evidence and arithmetic), `doctor` (visibility), `record`/`replay` | Users ask "why did it freeze", then "show me the numbers". This mirrors `git show` / `kubectl describe`; `why` is the highest-value verb and should be an alias for `report --since 15m --last`. |
| **Metric admission** | Mechanism + cheap source + exposure + detector + negative test | Prevents metric sprawl; a probe with no consumer is deleted. |
| **Unprivileged default** | Every metric above is tagged with the privilege it needs; the diagnosis must be complete at unit granularity without any | Keeps the tool usable on servers and in containers, and prevents a root-gated cause branch |

---

## 11. How to add a metric (the one-page process)

1. Name it (`domain.mechanism.metric`) and state which of M1-M10 it explains.
2. Give the source and its cost per sample, plus the privilege required.
3. Rate its exposure today. If it is **Served**, stop — we are not competing there.
4. Write the detector that consumes it, the narrative sentence, and the **negative test** that proves
   silence on a healthy host.
5. Add a fixture from a real machine, or the metric does not exist.
6. Update this table. A metric that is not in this table has no business in the code.
