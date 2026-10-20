# Fixtures

Fixtures are the testing foundation: **raw probe output**, never findings, plus a sidecar of ground
truth. Detectors can be rewritten and re-scored against them forever.

## Rules

1. **Real hardware only.** A fixture authored by the person writing the test proves self-consistency,
   not truth. The closest prior art shipped a cause branch that never executed for 29 days behind 132
   passing tests built on fabricated data.
2. **Never edit a fixture to make a test pass.** Fix the detector, or record a new fixture.
3. **Destructive faults only on disposable storage.** `remount_ro` and ext4 error injection belong on
   a loop-mounted image or a sacrificial card — never on a working rootfs.
4. **Small.** A 2-minute capture is under 1 MB. Downsample or split longer ones.
5. **Named for the area and the fault:** `sd_fsync_storm`, `swap_on_sd`, `dirty_throttle`.
6. **Held-out fixtures are not for agents.** `fixtures/heldout/` is used only by `xtask task score`.

## Layout

```text
fixtures/
  clean/clean_baseline.fpz      # healthy host: must produce ZERO findings
  pi/sd_fsync_storm.fpz         # + .yaml ground truth
  pi/sd_gc_stall.fpz
  pi/victim_heavy.fpz           # busy unit is innocent
  server/...
  heldout/                      # scoring only
```

Ground truth sidecar:

```yaml
id: pi_sd_fsync_storm
host: rpi4-8gb / bookworm / 6.6
inject: fio --name=storm --rw=randwrite --bs=4k --fsync=1 --runtime=120 --size=256m
expect:
  kind: io.small_write_storm
  culprit: fio
  min_confidence: correlated
forbid:
  - io.device_stall
```

## Recording

```bash
frictionpoint record --for 120s --label sd_fsync_storm --out fixtures/pi/sd_fsync_storm.fpz
```

The format is specified in [`../docs/SCHEMA.md`](../docs/SCHEMA.md) §8. Replay must be deterministic
and must reproduce identical `id` values.
