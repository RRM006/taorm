# TAORM — Thermal-Aware AI OS Resource Manager

**Course:** CSE 323 — Operating Systems Design

**Project:** A user-space Linux daemon that uses lightweight ML models to make smarter
resource management decisions (CPU scheduling, memory, security, I/O) — driven by
thermal state and power source (AC vs battery).

**Target machine:** Arch Linux, AMD CPU, 30GB free space, laptop

**Status:** Just started — Arch installed, project not yet built

---

## Design Documents (read for full detail)

| File | What it contains |
|------|-----------------|
| `../AIORM_Design_Document.md` | Original AI-OS concept (generic Linux) |
| `../TAORM_Design_Document_v2.md` | **Current design** — Arch-specific, thermal-aware |
| `../taorm_tech_stack.html` | Tech stack reference, install commands, do/don't |

## Agent Docs (this folder)

| File | Purpose |
|------|---------|
| `constitution.md` | Project principles — what we always do |
| `system_flowchart.md` | ASCII diagram of how TAORM works |
| `context_cse323.md` | Course requirements + topic mapping |
| `decisions.md` | Design decisions log (ADR format) |
| `changelog.md` | Session-by-session history |
| `current_task.md` | What we are doing RIGHT NOW |
| `milestone_log.md` | Big-picture status board |
| `test_log.md` | Test results with real numbers |
| `codebase_map.md` | Where everything lives in the repo |
| `session_protocol.md` | How to start and end every session |

## Quick Commands

```bash
# Check thermal sensors
sensors

# Check memory
free -h

# Check eBPF support
sudo bpftool feature

# Check cgroups v2
cat /sys/fs/cgroup/cgroup.controllers

# Build the project (once code exists)
cd E:\summer-26\cse323 && make
```
