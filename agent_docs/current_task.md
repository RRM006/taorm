# Current Task

**Last updated:** 2026-07-02

---

## Where We Are

Phase 0 (Environment Setup) and Phase 1 (Sensor Layer + Daemon Skeleton) are **done**.
Now moving to **Phase 2: Thermal-Aware CPU Scheduler**.

## What To Do RIGHT NOW

**Phase 2: Thermal-Aware CPU Scheduler** — Build the scheduler module that adjusts
process scheduling priorities based on thermal state and power source.

### Architecture

```
taormd (epoll loop) ──timerfd→ [every 500ms/1s]
    │
    ├── sensors_read()                          ← from Phase 1
    │
    └── scheduler_tick(sensor_data_t *data)     ← NEW
              │
              ├── determine_thermal_zone(temp) → COOL/WARM/HOT/CRITICAL/EMERGENCY
              ├── classify_processes() → classify each PID as FOREGROUND / COMPILE / BACKGROUND
              ├── compute_urgency(pid, zone, power_source) → score 0.0–1.0
              └── apply_policy(pid, urgency) → sched_setattr / setpriority / SIGSTOP
```

### Files to create/modify in `~/Desktop/taorm/`

| File | Action | Purpose |
|------|--------|---------|
| `src/scheduler.h` | Create | Scheduler interface |
| `src/scheduler.c` | Create | Thermal zone logic, process classification, sched_setattr |
| `src/proc.h` | Create | /proc reader interface (per-process stats) |
| `src/proc.c` | Create | Read /proc/[pid]/stat, cmdline, status for process classification |
| `Makefile` | Modify | Add scheduler.o, proc.o to SRC |
| `config/taorm.conf` | Modify | Add scheduler thresholds |

### Thermal Zones (from design doc)

| Zone | Temp Range | Action |
|------|-----------|--------|
| COOL | < 65°C | Normal scheduling, no intervention |
| WARM | 65–75°C | Background processes get lower nice value (-5) |
| HOT | 75–82°C | Pre-empt non-interactive background jobs, log thermal pressure |
| CRITICAL | 82–88°C | Only foreground + system processes get CPU time; others paused via SIGSTOP |
| EMERGENCY | > 88°C | Emergency SIGSTOP on all non-essential processes |

### Process Classification

Read `/proc/[pid]/` to classify processes:
- **FOREGROUND** — has active terminal (read /proc/[pid]/stat session leader, or check
  /proc/[pid]/status for the process with foreground PID via `focused` heuristics)
  For now: treat shell, IDE, browser as foreground based on cmdline matching
- **COMPILE** — cmdline matches make, gcc, g++, clang, rustc, cargo, cmake, ninja
- **BACKGROUND** — everything else (system services, indexers, etc.)

### Syscalls Used

| Syscall | Purpose |
|---------|---------|
| `sched_setattr(2)` | Apply SCHED_DEADLINE, SCHED_FIFO, SCHED_IDLE |
| `sched_getattr(2)` | Read current scheduling parameters |
| `setpriority(2)` | Adjust nice value for non-RT processes |
| `kill(SIGSTOP/SIGCONT)` | Pause/resume processes in emergency zones |

### First Integration

```bash
# Build the scheduler into taormd
make && sudo ./taormd --foreground --interval 1

# Test: run stress-ng in another terminal
stress-ng --cpu 4 --timeout 30s

# Watch: taormd should detect thermal zones, adjust process priorities
```

## Done Means

- `make` compiles without warnings
- `./taormd --foreground` shows thermal zone and process classifications
- Running `stress-ng --cpu 4` while taormd runs shows priority changes in journald
- `sched_setattr` calls are logged (don't need to verify effectiveness yet — Phase 2
  is about building the mechanism, Phase 6/7 is for benchmark validation)

## After This

Report results, update changelog, then Phase 2 → Done.
Move to Phase 3 (Memory Pressure Manager).
