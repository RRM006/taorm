# TAORM — Design Decisions Log

This file records real design choices: what we chose, why, and what we rejected.
When a decision is revisited and changed, add a new ADR and mark the old one as superseded.

---

## ADR-0001 — 2026-06-27 — Use TAORM over generic AIORM

- **Decision:** Use the thermal-aware variant (TAORM) instead of the generic AI-OS concept (AIORM)
- **Why:** TAORM targets our actual machine (Arch laptop, AMD CPU). The thermal + power
  angle gives the project a unique selling point that the generic version lacks. It also
  makes the project more practical and relatable.
- **Rejected:** Generic AIORM — too abstract, targets "any Linux machine", no thermal awareness,
  no device-specific tuning for our hardware.
- **Status:** Accepted

---

## ADR-0002 — 2026-06-27 — Target Arch Linux, not Ubuntu

- **Decision:** Target Arch Linux as the primary platform, not Ubuntu
- **Why:** Arch is what the user actually runs. The project should work on the real machine,
  not a hypothetical Ubuntu VM. Arch uses systemd, cgroups v2 by default, and rolling kernel
  releases that support modern features like eBPF.
- **Rejected:** Ubuntu 22.04 — the original AIORM doc targets this, but it is not the user's
  actual OS. Testing on a VM adds a layer of abstraction that hides real thermal/battery behavior.
- **Status:** Accepted

---

## ADR-0003 — 2026-06-27 — Use eBPF over ptrace for syscall tracing

- **Decision:** Use eBPF (via bcc/libbpf) as the primary syscall tracer for Module 3 (Security)
- **Why:** ptrace doubles process context-switch overhead and is too slow for production use.
  eBPF attaches to tracepoints with near-zero overhead and gives per-syscall visibility.
  bcc is available in Arch's official repos.
- **Rejected:** ptrace as primary tracer — acceptable as fallback for deep inspection of
  suspicious PIDs, but not for continuous monitoring of all processes.
- **Status:** Accepted

---

## ADR-0004 — 2026-06-27 — Lightweight models only, no deep learning

- **Decision:** Use only lightweight ML models: decision trees, Markov chains, n-gram
  hash tables, linear regression, Naive Bayes, one-class SVM
- **Why:** Our laptop has limited CPU and RAM. Deep learning models (LSTM, transformers,
  random forests with 100+ trees) would consume the very resources we are trying to
  manage. They also exceed the 1ms inference latency requirement.
- **Rejected:** LSTM for CPU/memory prediction, transformer models, heavy random forests,
  TensorFlow/PyTorch dependency chains.
- **Status:** Accepted

---

## ADR-0005 — 2026-06-27 — Run as systemd service, not cron or manual

- **Decision:** TAORM installs as a systemd service (taormd.service) with proper
  capability bounding, cgroup limits, and auto-restart
- **Why:** systemd is Arch's init system. A service with sd_notify integration
  survives reboots, gets proper signal handling, and integrates with journald
  for logging. A pacman hook restarts it after kernel upgrades.
- **Rejected:** cron job (no real-time loop), manual start (user must remember),
  SysV init script (Arch does not use SysV).
- **Status:** Accepted

---

## ADR-0006 — 2026-06-27 — User-space daemon, no kernel module

- **Decision:** TAORM runs entirely in user space, using ptrace, /proc, /sys,
  cgroups, and eBPF to observe and influence kernel behavior
- **Why:** Writing kernel modules is dangerous, hard to debug, and unnecessary.
  Linux provides a rich user-space interface for resource management. This is
  the same approach used by systemd, Docker, and cgroups managers.
- **Rejected:** Loadable kernel module (LKM) — too risky for a course project,
  requires root, hard to debug, unnecessary for our goals.
- **Status:** Accepted

---

## ADR-0007 — 2026-07-02 — epoll + timerfd + signalfd event loop

- **Decision:** Use epoll to multiplex timerfd (periodic polling) and signalfd
  (signal handling) in a single-threaded event loop
- **Why:** Single-threaded avoids locking, race conditions, and context-switch
  overhead. timerfd integrates with epoll — no busy-waiting. signalfd converts
  signals into fd events that fit naturally in the epoll pattern.
- **Rejected:** Multi-threaded (mutex complexity, harder to debug), sleep-based
  polling (can't handle signals cleanly), signal() handlers (interrupt epoll_wait
  with EINTR).
- **Status:** Accepted

---

## ADR-0008 — 2026-07-02 — Open sysfs files once, keep fd across reads

- **Decision:** Open sysfs files in sensors_init() and keep the fd. Each read()
  calls lseek() to reset position.
- **Why:** sysfs files are virtual — open/close per read adds unnecessary syscalls.
  Holding fds open is standard for monitoring daemons.
- **Rejected:** fopen/fclose per read (slower), reading via shell commands
  (popening cat/sensors).
- **Status:** Accepted

---

## ADR-0009 — 2026-07-02 --foreground flag for development

- **Decision:** Add --foreground flag that prints JSON to stdout (default: journald).
- **Why:** During development, tailing journalctl is slow. JSON to stdout allows
  piping, redirecting, and quick testing.
- **Rejected:** Always-journald (hard to debug), always-stdout (breaks systemd service).
- **Status:** Accepted
