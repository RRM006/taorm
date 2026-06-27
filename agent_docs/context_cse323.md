# CSE 323 — Operating Systems Design: Course Context

## Course Info

- **Course:** CSE 323 — Operating Systems Design
- **Semester:** Summer 2026
- **Project type:** Kernel-level / System-level implementation with AI integration
- **Platform:** Linux (user-space daemon, no kernel modules)

## Required OS Topics and How TAORM Covers Them

| OS Topic | Where in TAORM | Implementation |
|----------|---------------|----------------|
| Processes | All modules | PID management, fork(), signal handling |
| CPU Scheduling | Module 1 | sched_setattr with thermal-aware priority |
| Memory Management | Module 2 | madvise, mincore, mlock, zram, swappiness |
| Deadlocks | Module 1 + 2 | Resource allocation graph + futex wait-chain analysis |
| Disk Scheduling | Module 4 | ioprio_set, io_uring, NVMe vs HDD policy |
| Security | Module 3 | eBPF syscall tracing, n-gram anomaly, seccomp |
| IPC | Daemon design | eventfd between modules, epoll, systemd D-Bus |
| Virtual Memory | Module 2 | Page residency, zram swap, pressure prediction |
| I/O Management | Module 4 | io_uring async I/O, sync_file_range, power-aware flush |
| System Calls | All modules | 20+ distinct syscalls through POSIX interface |

## Deliverables

1. **Working code** — all 4 modules implemented and tested
2. **Benchmarks** — quantitative comparison: TAORM active vs baseline (TAORM disabled)
3. **Report** — design document, implementation details, results, lessons learned
4. **Demo** — 15-minute live demonstration on actual Arch laptop
5. **Dashboard** — terminal-based (curses) live view of TAORM decisions

## Benchmark Targets

| Metric | Target |
|--------|--------|
| Peak CPU temp reduction | >= 8 degrees C lower than baseline |
| Page fault rate reduction | >= 20% fewer faults |
| OOM kills prevented | 0 in 1-hour browser + compile test |
| Battery power reduction | >= 10% less draw |
| Threat detection latency | < 50ms |
| False positive rate | < 2% |

## References

- Love, R. (2010). Linux Kernel Development, 3rd Ed.
- Silberschatz et al. (2018). Operating System Concepts, 10th Ed.
- Forrest et al. (1996). A Sense of Self for Unix Processes (n-gram anomaly detection)
- Mao et al. (2019). Learning Scheduling Algorithms (ML for OS scheduling)
