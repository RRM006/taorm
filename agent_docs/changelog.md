# TAORM — Session History

Newest entry at the top. One short entry per session.

---

## Session 3 — 2026-07-02 — Phase 0 + Phase 1 completed

- **Did:** Completed Phase 0 (installed bcc-examples, bcc-libbpf-tools, bpf; verified
  execsnoop and bpftool; created project tree). Built Phase 1: `taormd.c`, `sensors.c`,
  `sensors.h`, `Makefile`, `systemd/taorm.service`, `config/taorm.conf`. Verified on Arch:
  foreground JSON output, journald logging, SIGUSR1 status dump, SIGTERM clean shutdown,
  AC/battery detection, all sensors (CPU/GPU/NVMe temp, fan RPM, CPU freq, load avg, battery %).
- **Decided:** Phase 1 Done. Move to Phase 2 (Thermal-Aware CPU Scheduler).
- **Broke / problem:** `sys/loadavg.h` doesn't exist on Linux (getloadavg in stdlib.h).
  `/sys/class/power_supply/AC0` is a symlink, not DT_DIR — fixed.
- **Deferred:** —
- **Next:** Phase 2 — Build thermal-aware CPU scheduler with sched_setattr.

- **Did:** Installed remaining tools (bcc-examples, bcc-libbpf-tools, bpf), verified
  execsnoop and bpftool work, created project directory tree at ~/Desktop/taorm/.
  Updated milestone_log and changelog.
- **Decided:** Phase 0 done. Moving to Phase 1 (Sensor Layer + Daemon Skeleton).
- **Broke / problem:** Nothing.
- **Deferred:** —
- **Next:** Phase 1 — Build sensor layer and daemon skeleton in C.

## Session 2 — 2026-06-27 — Project explained in simple terms

- **Did:** Explained the entire project in 6th-grade-friendly language: what TAORM is,
  what the dashboard looks like, inputs/outputs, how to present to faculty, Q&A prep
  (60-second pitch, "why unique", "why not deep learning", "is this realistic",
  "limitations"), topics to learn (processes, scheduling, memory, syscalls, eBPF),
  vibe coding rules (plumbing yes, brain must understand).
- **Decided:** No new decisions.
- **Broke / problem:** Nothing.
- **Deferred:** All implementation still deferred. User needs to run Phase 0 commands
  on their actual Arch laptop.
- **Next:** User runs the 9 environment setup steps on Arch laptop, reports results.
