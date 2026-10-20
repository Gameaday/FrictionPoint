# Demo

The demo is a **reproducible script**, not a GIF. A stranger must be able to run it and see the same
result on their own hardware.

## Target: 15 seconds, one terminal

```bash
# 1. create a small-write / fsync storm
fio --name=storm --rw=randwrite --bs=4k --fsync=1 --runtime=30 --size=64m &

# 2. ask the question a user asks
frictionpoint why

# 3. expected shape
#   Your system lost 9.7s to I/O stall in the last 30s.
#   fio (unit: session-3.scope) accounts for 91% of write requests, mean 4.0 KiB.
#   The device is not slow: queue depth 1.1, 0.4 MiB/s, 1,900 IOPS.

# 4. stop the storm, ask again
kill %1
frictionpoint why
#   No friction above threshold.
```

## Rules

- No root beyond the documented capability; if a scenario needs more, say so on screen.
- Clean up every process and temporary file, including on failure.
- Deterministic output for the fixture corpus; live runs may vary but must be explainable.
- No hidden network access.
- Keep it under 30 seconds total.

`demo.sh` is added in A1. Until then this file is the specification.
