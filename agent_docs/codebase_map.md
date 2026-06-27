# TAORM — Codebase Map

Where everything lives. Update when files are added or moved.

**Last updated:** 2026-06-27

---

## Root: E:\summer-26\cse323\

| Path | What |
|------|------|
| `AIORM_Design_Document.md` | Original AI-OS concept design (generic Linux) |
| `TAORM_Design_Document_v2.md` | Current design — Arch-specific, thermal-aware |
| `taorm_tech_stack.html` | Tech stack reference, install steps, do/don't |
| `ai_os_project_blueprint.html` | Interactive HTML blueprint for original concept |

## agent_docs/ — Memory System

| Path | What |
|------|------|
| `agent_docs/opencode.md` | OpenCode auto-read entry point |
| `agent_docs/constitution.md` | Project principles |
| `agent_docs/system_flowchart.md` | ASCII architecture diagram |
| `agent_docs/context_cse323.md` | Course requirements |
| `agent_docs/decisions.md` | Design decisions log |
| `agent_docs/changelog.md` | Session history |
| `agent_docs/current_task.md` | Current focus (overwritten each session) |
| `agent_docs/milestone_log.md` | Status board |
| `agent_docs/test_log.md` | Test results |
| `agent_docs/codebase_map.md` | This file |
| `agent_docs/session_protocol.md` | Session start/end checklist |

## (Future) taorm/ — Project Source

Not yet created. Will contain:

```
taorm/
  src/
    main.c              <- daemon main loop, epoll, signalfd
    sensor.c            <- /proc and /sys readers
    scheduler.c         <- Module 1: CPU scheduler
    memory.c            <- Module 2: memory manager
    security.c          <- Module 3: syscall monitor
    io_manager.c        <- Module 4: I/O manager
    ai/
      decision_tree.c   <- online decision tree (CPU model)
      markov.c          <- Markov chain (memory model)
      ngram.c           <- n-gram model (security baseline)
      linear.c          <- online linear regression (I/O model)
    bpf/
      syscall_trace.c   <- eBPF program (kernel-side tracer)
  python/
    svm_classifier.py   <- one-class SVM threat classifier
    dashboard.py        <- curses terminal dashboard
    benchmark.py        <- benchmark runner and plotter
  config/
    taorm.conf          <- default configuration
  systemd/
    taormd.service      <- systemd unit file
  tests/
    deadlock_test.sh    <- deadlock injection test
    memory_test.sh      <- OOM pressure test
    security_test.sh    <- simulated threat test
  Makefile
  README.md
```
