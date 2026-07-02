# TAORM — Milestone Log

Big-picture status. Update when a module's state changes.

**Last updated:** 2026-06-27

---

## Status Keys

| Key | Meaning |
|-----|---------|
| Not started | Nothing done yet |
| In progress | Actively working on it |
| Blocked | Stuck — needs something before continuing |
| Done | Complete and tested with real numbers |
| Retired | No longer part of the plan |

---

## Phases

### Phase 0 — Environment Setup
- **Status:** Done
- **Done means:** All packages installed, eBPF works, sensors work, cgroups v2 active
- **Started:** 2026-07-02
- **Completed:** 2026-07-02

### Phase 1 — Sensor Layer + Daemon Skeleton
- **Status:** Done
- **Done means:** taormd starts, reads all sensors, prints structured logs to journald
- **Started:** 2026-07-02
- **Completed:** 2026-07-02

### Phase 2 — Module 1: Thermal-Aware CPU Scheduler
- **Status:** Not started
- **Done means:** Benchmark shows >= 8C lower peak temp vs baseline under stress-ng
- **Started:** —
- **Completed:** —

### Phase 3 — Module 2: Memory Pressure Manager
- **Status:** Not started
- **Done means:** 0 OOM kills in browser-20-tabs + make -j8 test on 12GB
- **Started:** —
- **Completed:** —

### Phase 4 — Module 3: Security Monitor
- **Status:** Not started
- **Done means:** Detects simulated AUR malicious process within 50ms, < 2% false positives
- **Started:** —
- **Completed:** —

### Phase 5 — Module 4: I/O Manager
- **Status:** Not started
- **Done means:** >= 15% lower foreground I/O latency during pacman -Syu on battery
- **Started:** —
- **Completed:** —

### Phase 6 — Integration
- **Status:** Not started
- **Done means:** All 4 modules run simultaneously for 24 hours without crash
- **Started:** —
- **Completed:** —

### Phase 7 — Report + Demo
- **Status:** Not started
- **Done means:** Report written, 15-min demo recorded, dashboard working
- **Started:** —
- **Completed:** —

---

## Overall Progress

```
Phase 0  [####################] 100%
Phase 1  [####################] 100%
Phase 2  [                    ] 0%
Phase 3  [                    ] 0%
Phase 4  [                    ] 0%
Phase 5  [                    ] 0%
Phase 6  [                    ] 0%
Phase 7  [                    ] 0%
```
