# TAORM — Codebase Map

Where everything lives. Update when files are added or moved.

**Last updated:** 2026-07-02

---

## Root: ~/Desktop/taorm/ (project source) + ~/Workspace/Projects/taorm/ (design docs)

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

## taorm/ — Project Source

Created 2026-07-02. Located at `~/Desktop/taorm/`. Structure:

```
taorm/
  src/
    taormd.c    [DONE]  <- daemon main loop, epoll, signalfd
    sensors.h   [DONE]  <- sensor data types
    sensors.c   [DONE]  <- sysfs readers (temp, fan, power, freq, load)
    scheduler.c [NEXT]  <- Module 1: CPU scheduler
    scheduler.h [NEXT]  <- Module 1: CPU scheduler header
    proc.c      [NEXT]  <- /proc reader for process classification
    proc.h      [NEXT]  <- /proc reader interface
    memory.c    [PLAN]  <- Module 2: memory manager
    security.c  [PLAN]  <- Module 3: syscall monitor
    io_manager.c [PLAN] <- Module 4: I/O manager
    ai/
      decision_tree.c [PLAN] <- online decision tree (CPU model)
      markov.c        [PLAN] <- Markov chain (memory model)
      ngram.c         [PLAN] <- n-gram model (security baseline)
      linear.c        [PLAN] <- online linear regression (I/O model)
    bpf/
      syscall_trace.c [PLAN] <- eBPF program (kernel-side tracer)
  python/
    svm_classifier.py [PLAN] <- one-class SVM threat classifier
    dashboard.py      [PLAN] <- curses terminal dashboard
    benchmark.py      [PLAN] <- benchmark runner and plotter
  config/
    taorm.conf        [DONE] <- default configuration
  systemd/
    taorm.service     [DONE] <- systemd unit file
  tests/
    [PLAN]
  Makefile            [DONE] <- build system
  README.md           [DONE] <- (empty)
```
