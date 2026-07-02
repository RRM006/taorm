# TAORM — Test Log

Records what we tested, how, and the result. Includes failed runs.

**Last updated:** 2026-06-27

---

## Template

```
## YYYY-MM-DD — Module N — <what was tested>
- Setup: <model/library/version, machine, sample data>
- Metric(s): <what we measured>
- Result: <the numbers>
- Notes: <what helped, what failed, next idea>
```

---

## Entries

## 2026-07-02 — Phase 1 — taormd daemon smoke test

### Test 1: Build
- **Setup:** `make clean && make` in ~/Desktop/taorm/
- **Result:** Compiles cleanly, 0 warnings, binary `taormd` produced (14KB stripped)
- **Notes:** Fixed `sys/loadavg.h` (doesn't exist on Linux — getloadavg in stdlib.h)

### Test 2: Foreground mode + sensors
- **Setup:** `./taormd --foreground --interval 1` for 4 seconds
- **Result:** Prints JSON lines every 1s: cpu_temp, power, battery%, cpu_freq, fan_rpm,
  load, nvme_temp, gpu_temp. All values non-zero/valid.
- **Sample:** `{"cpu_temp_C":59.2,"power":"AC","battery_pct":100,"cpu_freq_MHz":1647,...}`
- **Notes:** Initial AC detection failed (symlink `/sys/class/power_supply/AC0` not
  matched by `DT_DIR` check) — fixed by removing d_type filter.

### Test 3: AC power detection
- **Setup:** `cat /sys/class/power_supply/AC0/online` = 1. Daemon shows `"power":"AC"`
- **Result:** Correct

### Test 4: Signal handling — SIGUSR1
- **Setup:** `./taormd --foreground --interval 10 &` then `kill -USR1 $PID`
- **Result:** Prints `--- STATUS DUMP ---` followed by sensor JSON
- **Notes:** Status dump uses `print_sensor_json()` (same format as periodic)

### Test 5: Signal handling — SIGTERM
- **Setup:** Same as Test 4, then `kill -TERM $PID`
- **Result:** Exits cleanly with `TAORM: shutdown complete`

### Test 6: Journald logging
- **Setup:** `./taormd --interval 1` (daemon mode), then SIGUSR1 + SIGTERM
- **Result:** `journalctl -n 5 -o cat | grep TAORM` shows:
  ```
  TAORM sensor: cpu_temp=56.1 power=AC battery=100% freq=1539MHz fan=2800rpm ...
  TAORM status: cpu_temp=56.6 power=AC battery=100% freq=1397MHz load=1.27
  TAORM: received signal 15, shutting down
  TAORM: shutdown complete
  ```
- **Notes:** All log levels correct: sensors at INFO, status at NOTICE, shutdown at NOTICE
