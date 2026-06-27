# TAORM — Thermal-Aware AI OS Resource Manager
### CSE 323 — Operating Systems Design | Project Design Document v2.0

> **Environment:** Arch Linux (Kernel 6.x) · 12 GB RAM · Laptop · Single-user · x86-64
> **What makes this unique:** Every resource decision is co-driven by thermal state and power source
> (AC vs battery) — something no stock OS scheduler does, but every laptop user feels.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Why This Is Unique](#2-why-this-is-unique)
3. [Real Problem on Your Actual Machine](#3-real-problem-on-your-actual-machine)
4. [Goals — User, System, OS](#4-goals)
5. [System Architecture](#5-system-architecture)
6. [Core Modules — Detailed Design](#6-core-modules)
   - 6.1 [Thermal-Aware CPU Scheduler](#61-module-1--thermal-aware-ai-cpu-scheduler)
   - 6.2 [12 GB Memory Pressure Manager](#62-module-2--12-gb-memory-pressure-manager)
   - 6.3 [AI Syscall Security Monitor](#63-module-3--ai-syscall-security-monitor)
   - 6.4 [Power-State Disk I/O Manager](#64-module-4--power-state-disk-io-manager)
7. [System Call Map](#7-system-call-map)
8. [AI Model Design](#8-ai-model-design)
9. [Arch Linux Integration](#9-arch-linux-integration)
10. [Processes, Deadlocks, Disk Scheduling, Security — Coverage Map](#10-os-topic-coverage-map)
11. [Implementation Plan — Week by Week](#11-implementation-plan)
12. [Expected Results and Outcomes](#12-expected-results-and-outcomes)
13. [Benchmarks and Evaluation](#13-benchmarks-and-evaluation)
14. [Tools and Setup on Arch](#14-tools-and-setup-on-arch)
15. [References](#15-references)

---

## 1. Project Overview

**Project Name:** TAORM — Thermal-Aware AI OS Resource Manager

**Course:** CSE 323 — Operating Systems Design

**Target Machine:** Your Arch Linux laptop, 12 GB RAM, x86-64

**Implementation Language:** C (system layer) + Python 3 (AI models, dashboard)

**Kernel Interface:** User-space daemon — uses `/proc`, `/sys/class/thermal`, `cgroups v2`,
`eBPF` (via `bcc`), `systemd-dbus`, and direct Linux system calls. No kernel module required.

---

TAORM is a system daemon that makes the Linux kernel smarter about resource allocation
by combining two signals that the stock kernel never uses together:

1. **Thermal state** — how hot the CPU/chassis currently is
2. **AI predictions** — what each running process will need next, learned from behavior

On a laptop, these two signals matter more than on any other hardware class. When your CPU
hits 90°C the kernel still schedules processes the same way it did at 45°C — it doesn't
know that a 100ms time slice at 90°C actually delivers 60ms of real work because the core
is throttled. TAORM knows. It adjusts scheduling, memory, I/O, and security policy in real
time as thermal and power conditions change.

The result is a system that runs cooler, wastes less battery, prevents OOM crashes during
heavy development work, and detects misbehaving processes — all on your real 12 GB Arch machine.

---

## 2. Why This Is Unique

Most OS course projects simulate resource management or implement it in an abstract sandbox.
TAORM runs on your real machine and solves problems you actually experience:

| Problem You Experience | What TAORM Does About It |
|------------------------|--------------------------|
| Fan spins up hard during compilation, laptop gets hot, everything slows | Thermal scheduler detects throttling onset at 80°C, demotes background compile jobs before the kernel throttles — so foreground stays fast |
| Browser (Chromium/Firefox) eats 4–6 GB RAM, leaving 6 GB for everything else | Memory manager tracks browser tab processes specifically, reclaims idle tabs via `MADV_FREE` before OOM hits |
| Pacman upgrades in background eat disk I/O, making everything else laggy | I/O manager detects power state and demotes `pacman` I/O priority on battery, promotes it on AC |
| No swap on Arch by default — OOM killer fires unpredictably | Memory manager sets up and manages `zram` swap dynamically; AI decides when to compress vs evict |
| On battery, normal Linux still burns CPU on background processes | All four modules switch to a power-conservative policy the moment `/sys/class/power_supply/AC/online` reads 0 |

**The uniqueness factor:** No existing tool combines thermal state, power source, AI prediction,
and per-subsystem kernel control into a single unified daemon. `thermald` handles thermal.
`TLP` handles power. `earlyoom` handles memory. They don't talk to each other.
TAORM is the unified brain that connects all of them through one AI decision layer.

---

## 3. Real Problem on Your Actual Machine

### Your Hardware Reality

```
Machine:   Laptop (x86-64)
OS:        Arch Linux, Kernel 6.x (rolling)
RAM:       12 GB (middle ground — not low-end, not server)
Storage:   NVMe SSD (typical for modern laptops)
Power:     Battery + AC, thermal throttling under load
Swap:      None by default on Arch (or minimal)
Init:      systemd
Cgroups:   v2 (Arch default since 2021)
```

### The 12 GB Problem

12 GB sounds large. In practice on Arch with a browser open during development:

```
Firefox/Chromium (10 tabs):   3.5 – 5.0 GB
VS Code + LSP server:         0.8 – 1.5 GB
Docker (1 container):         0.6 – 1.0 GB
Terminal + misc:              0.3 – 0.5 GB
System / kernel:              0.4 – 0.6 GB
─────────────────────────────────────────
Remaining for compilation:    1.5 – 3.0 GB
```

A `make -j$(nproc)` build on a C++ project can need 4–8 GB. This hits OOM. Without swap,
the kernel's OOM killer fires — killing a random process, often the browser or the build itself.
TAORM prevents this by watching memory pressure build up and reclaiming proactively.

### The Thermal Problem

Arch Linux with a modern kernel uses `intel_pstate` or `amd-pstate` CPU frequency scaling.
Under sustained load, the CPU reaches 85–95°C and the firmware throttles it — dropping clock
frequency by 20–40% silently. The scheduler still thinks it's getting 100% CPU. It is not.

```
CPU at 45°C:  3.8 GHz effective, 100% capacity
CPU at 80°C:  3.2 GHz effective, ~84% capacity   ← thermal threshold
CPU at 90°C:  2.4 GHz effective, ~63% capacity   ← throttle zone
CPU at 95°C:  1.6 GHz effective, ~42% capacity   ← thermal limit
```

TAORM reads `/sys/class/thermal/thermal_zone*/temp` every 500ms and adjusts
process scheduling urgency before throttling occurs — keeping the CPU cooler by
distributing load differently, rather than letting the firmware do a hard frequency cut.

---

## 4. Goals

### User Goals

| ID | Goal | How TAORM Achieves It |
|----|------|-----------------------|
| U1 | Laptop stays cool during heavy work | CPU scheduler pre-empts background processes at 78°C before firmware throttles at 85°C |
| U2 | Browser + compile job can coexist on 12 GB | Memory manager reclaims idle browser tabs when compile job starts |
| U3 | Battery lasts longer during normal use | All modules switch to conservative policy on battery; background I/O and CPU work deferred |
| U4 | System does not randomly OOM-kill during builds | Proactive memory reclaim + zram management prevents OOM events |
| U5 | Malicious or runaway processes detected fast | Syscall anomaly monitor flags unusual behavior within 30ms |

### System Goals

| ID | Goal | Measurable Target |
|----|------|-------------------|
| S1 | Reduce peak CPU temperature under sustained load | ≥8°C reduction at 100% CPU workload vs baseline |
| S2 | Reduce page fault rate during memory-heavy workloads | ≥20% fewer page faults vs baseline |
| S3 | Prevent OOM events during realistic dev workflow | 0 OOM kills in 1-hour browser + compile test |
| S4 | Reduce battery drain during light workloads | ≥10% reduction in power draw (measured via `powertop`) |
| S5 | Detect simulated threat processes | 100% detection rate, < 2% false positive rate |
| S6 | Reduce disk I/O latency for foreground apps on battery | ≥15% lower foreground I/O wait time |

### OS Design Goals (CSE 323 alignment)

| OS Topic | Coverage in TAORM |
|----------|-------------------|
| Process scheduling | Module 1 — custom priority policy via `sched_setattr` |
| Memory management | Module 2 — `madvise`, `zram`, pressure monitoring |
| Deadlock handling | Module 2 sub-component — resource graph + prediction |
| Disk scheduling | Module 4 — `ionice`, `io_uring`, I/O priority by power state |
| Security | Module 3 — ptrace/eBPF syscall anomaly detection |
| IPC | systemd D-Bus integration; module-to-module event bus |
| Processes and threads | All modules fork/thread correctly; process group management |

---

## 5. System Architecture

```
╔══════════════════════════════════════════════════════════════════╗
║                   USER SPACE — YOUR APPS                        ║
║   Firefox  │  VS Code  │  Docker  │  pacman  │  Terminal        ║
╚══════════════════╤═══════════════════════════════════════════════╝
                   │  normal system calls
╔══════════════════▼═══════════════════════════════════════════════╗
║              TAORM DAEMON  (pid: taormd)                        ║
║                                                                  ║
║  ┌─────────────────────────────────────────────────────────┐    ║
║  │              SENSOR LAYER  (reads every 500ms)          │    ║
║  │  /sys/class/thermal  │  /proc/meminfo  │  /proc/stat    │    ║
║  │  /sys/class/power_supply  │  /proc/[pid]/stat + smaps   │    ║
║  └────────────────────────┬────────────────────────────────┘    ║
║                           │  feature vectors                    ║
║  ┌────────────────────────▼────────────────────────────────┐    ║
║  │              AI DECISION ENGINE                         │    ║
║  │   thermal_model │ memory_model │ security_model │       │    ║
║  │   io_model      │ power_policy │ deadlock_graph │       │    ║
║  └──┬──────────────┬──────────────┬─────────────┬──────────┘    ║
║     │              │              │             │               ║
║  ┌──▼──┐       ┌───▼──┐      ┌───▼──┐     ┌───▼──┐            ║
║  │ M1  │       │  M2  │      │  M3  │     │  M4  │            ║
║  │ CPU │       │ MEM  │      │ SEC  │     │ DISK │            ║
║  └──┬──┘       └───┬──┘      └───┬──┘     └───┬──┘            ║
║     │              │              │             │               ║
╚═════╪══════════════╪══════════════╪═════════════╪═══════════════╝
      │              │              │             │
      │   sched_     │   madvise    │  ptrace /   │  ionice /
      │   setattr    │   mlock      │  seccomp    │  io_uring
      │   setpriority│   zram ctl   │  prctl      │  ioprio_set
      │              │              │             │
╔═════▼══════════════▼══════════════▼═════════════▼═══════════════╗
║                     LINUX KERNEL 6.x                            ║
║   CFS + DEADLINE Scheduler │ Memory Manager │ VFS │ Block Layer ║
║   cgroups v2  │  eBPF  │  netfilter  │  perf_events             ║
╚═════════════════════════════════════════════════════════════════╝
                               │
╔══════════════════════════════▼═══════════════════════════════════╗
║           HARDWARE                                              ║
║  CPU (throttles at 85°C) │ 12 GB RAM │ NVMe SSD │ Battery       ║
╚══════════════════════════════════════════════════════════════════╝
```

### Arch-Specific Integration Points

```
systemd unit:    /etc/systemd/system/taormd.service
cgroups v2:      /sys/fs/cgroup/taorm.slice/
config file:     /etc/taorm/taorm.conf
log target:      systemd journal (journalctl -u taormd)
thermal source:  /sys/class/thermal/thermal_zone*/temp
power source:    /sys/class/power_supply/AC/online
zram device:     /dev/zram0  (managed by taormd)
eBPF programs:   /usr/lib/taorm/bpf/*.o
```

### Power State Machine

TAORM's entire behavior changes based on power source. This is the core architectural decision
that makes it laptop-native:

```
                    AC plugged in
          ┌─────────────────────────────┐
          ▼                             │
  ┌──────────────┐    unplug      ┌─────┴────────┐
  │  AC MODE     │ ─────────────► │ BATTERY MODE │
  │  Performance │                │  Efficiency  │
  │  + Security  │ ◄───────────── │  + Security  │
  └──────┬───────┘    plug in     └─────┬────────┘
         │                              │
         ▼                              ▼
  Full prefetch ON            Prefetch disabled
  Compile at max priority     Compile deferred
  I/O: throughput mode        I/O: power-save mode
  Thermal: warn at 82°C       Thermal: warn at 75°C
  Security: standard          Security: strict
```

---

## 6. Core Modules

### 6.1 Module 1 — Thermal-Aware AI CPU Scheduler

**Priority:** High | **Unique to TAORM**

#### The Core Insight

The Linux CFS scheduler treats a CPU core running at 3.8 GHz identically to one running at
1.8 GHz due to thermal throttling. From the scheduler's perspective, both are "100% utilized."
From the user's perspective, the throttled system feels 50% slower. TAORM bridges this gap.

#### How It Works

```
Every 500ms:
  1. Read CPU temperature from /sys/class/thermal/thermal_zone0/temp
  2. Read current CPU frequency from /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq
  3. Compute effective_capacity = current_freq / max_freq  (e.g. 0.63 at 90°C)
  4. Read per-process CPU burst data from /proc/[pid]/stat
  5. Feed [temp, effective_capacity, burst_history] into thermal_model
  6. Model outputs: process_urgency_score[] for each running process
  7. Apply: sched_setattr(pid, policy based on urgency and thermal zone)
```

#### Thermal Zones and Actions

| Zone | Temperature | Action |
|------|-------------|--------|
| Cool | < 65°C | Normal scheduling, no intervention |
| Warm | 65–75°C | Background processes get lower nice value (-5) |
| Hot | 75–82°C | TAORM pre-empts non-interactive background jobs, logs thermal pressure |
| Critical | 82–88°C | Only foreground + system processes get CPU time; others paused via SIGSTOP |
| Emergency | > 88°C | Emergency SIGSTOP on all non-essential processes; alert user via libnotify |

These thresholds are lower than firmware throttling (typically 85–95°C), which means TAORM
acts proactively before the hardware degrades performance.

#### Deadlock Prevention Sub-component

A resource allocation graph tracks which processes are waiting on which locks (inferred from
`futex` syscall patterns via eBPF). When a cycle is detected (circular wait):

```c
// Deadlock resolution policy
if (deadlock_detected(&graph)) {
    pid_t victim = select_victim_process(&graph);  // lowest priority in cycle
    // Option 1: preempt and retry
    sched_setattr(victim, SCHED_IDLE, 0);
    sleep_ms(50);
    sched_setattr(victim, original_policy, original_priority);
    // Option 2: if cycle persists after 3 retries, escalate to user
}
```

#### System Calls Used

| System Call | Purpose |
|-------------|---------|
| `sched_setattr(2)` | Apply SCHED_DEADLINE, SCHED_FIFO, SCHED_OTHER per thermal zone |
| `sched_getattr(2)` | Read current process scheduling parameters |
| `setpriority(2)` | Adjust nice value for non-RT processes |
| `getrusage(2)` | Read CPU burst time consumed |
| `clock_gettime(2)` | High-resolution timing for burst measurement |
| `kill(SIGSTOP/SIGCONT)` | Pause non-essential processes in emergency thermal zone |

#### AI Model: Thermal Burst Predictor

```
Input features (per process, per cycle):
  - cpu_temp_C                    (from /sys/class/thermal)
  - effective_capacity            (current_freq / max_freq)
  - recent_burst_ms[10]           (last 10 measured bursts)
  - process_type                  (interactive/compile/background — detected by cmdline)
  - is_foreground                 (has active window? — from /proc/[pid]/status)
  - context_switch_rate           (voluntary + involuntary per second)

Output:
  - predicted_burst_ms            (how long this process will run before yielding)
  - urgency_score [0.0–1.0]       (how much this process needs CPU right now)
  - suggested_sched_policy        (SCHED_IDLE / SCHED_OTHER / SCHED_FIFO)

Model type:   Online decision tree, depth 8
Update freq:  After each process exits a CPU burst
Latency:      < 0.3ms per prediction
```

---

### 6.2 Module 2 — 12 GB Memory Pressure Manager

**Priority:** High | **Tuned for your exact RAM size**

#### The 12 GB Constraint

12 GB is Arch's most common laptop RAM configuration. It is large enough to run a full
development environment but small enough to OOM under realistic loads. This module is
designed specifically for that tight margin.

#### Three-Layer Memory Strategy

```
Layer 1: PREDICTION — watch pressure build, act before OOM
Layer 2: RECLAIM    — free idle pages from known memory hogs
Layer 3: COMPRESSION — use zram as intelligent swap before evicting to disk
```

#### Layer 1 — Pressure Prediction

TAORM reads `/proc/meminfo` every 500ms and tracks:

```
memory_pressure = (MemTotal - MemAvailable) / MemTotal

Pressure zones:
  Green  (0.00 – 0.65):  No action. Normal operation.
  Yellow (0.65 – 0.78):  Start monitoring top-5 RSS consumers.
  Orange (0.78 – 0.88):  Reclaim browser idle tabs via madvise(MADV_FREE).
  Red    (0.88 – 0.94):  Compress cold pages to zram, demote compile jobs.
  Black  (> 0.94):       Emergency — suspend non-critical processes, notify user.
```

The AI model predicts which pressure zone the system will be in 5 seconds from now,
allowing proactive action rather than reactive OOM response.

#### Layer 2 — Intelligent Reclaim

TAORM identifies process categories and applies targeted reclaim:

```
Browser processes (firefox, chromium, electron):
  - Enumerate tab-renderer processes via /proc/[pid]/cmdline
  - Rank tabs by last-access time (inferred from CPU activity)
  - Apply madvise(MADV_FREE) to idle-tab working sets
  - Effect: kernel reclaims those pages when pressure rises

Compilation jobs (make, gcc, g++, clang, rustc):
  - These allocate predictably in phases (parse → AST → codegen → link)
  - Prefetch next-phase pages via madvise(MADV_WILLNEED) when current phase ends
  - Effect: fewer major page faults during build phase transitions

IDE processes (code, jetbrains, nvim with LSP):
  - High priority — never reclaim while user is active (foreground window)
  - On battery: pause LSP server background indexing via SIGSTOP
```

#### Layer 3 — zram Management

Arch Linux does not set up swap by default. TAORM manages a `zram` device dynamically:

```
On startup:
  modprobe zram
  echo lz4 > /sys/block/zram0/comp_algorithm  # lz4: fast, good ratio
  echo 4G  > /sys/block/zram0/disksize         # 4GB virtual, ~1.5GB actual
  mkswap /dev/zram0
  swapon /dev/zram0 -p 32767                   # highest swap priority

AI controls swappiness dynamically:
  - On AC + cool:     vm.swappiness = 60  (normal)
  - On battery:       vm.swappiness = 80  (prefer compress over evict)
  - Memory pressure > 0.88:  vm.swappiness = 100 (compress aggressively)
  - Foreground compile job:  vm.swappiness = 10  (keep build in RAM)
```

#### Deadlock in Memory Context

Memory deadlocks occur when Process A waits for Process B to free memory and B waits for A.
TAORM detects this via `futex` wait chains (eBPF probe on `futex_wait`):

```
if (circular_wait_detected_in_futex_graph()) {
    // Identify the process with the largest anonymous RSS
    pid_t blocker = find_largest_rss_in_cycle();
    // Force release: madvise its anonymous mappings as MADV_FREE
    madvise_rss_pages(blocker, MADV_FREE);
    // This gives the kernel permission to reclaim those pages,
    // breaking the memory starvation cycle
}
```

#### System Calls Used

| System Call | Purpose |
|-------------|---------|
| `madvise(2)` | Prefetch (MADV_WILLNEED) or release (MADV_FREE) pages |
| `mincore(2)` | Check which pages are currently in physical RAM |
| `mlock(2)` | Pin critical process pages (IDE, shell) |
| `mmap(2)` | Map shared memory regions for monitoring |
| `sysctl(2)` / `write /proc/sys/vm/` | Adjust swappiness dynamically |

#### AI Model: Memory Pressure Predictor

```
Input features (collected every 500ms):
  - mem_pressure_now              (0.0–1.0)
  - pressure_delta_last_5s        (rate of change)
  - top5_rss_mb[]                 (top 5 processes by RSS)
  - active_compile_job            (bool)
  - browser_tab_count             (inferred from renderer PIDs)
  - power_source                  (AC / battery)
  - zram_usage_ratio              (zram used / zram size)

Output:
  - predicted_pressure_5s_from_now
  - recommended_action            (NONE / RECLAIM_BROWSER / COMPRESS / SUSPEND_BG)
  - target_process_pid            (which process to act on)

Model type:   3-step Markov chain + logistic regression for action selection
Latency:      < 1ms
```

---

### 6.3 Module 3 — AI Syscall Security Monitor

**Priority:** High | **Arch-specific threat model**

#### Why Arch Needs This

Arch users compile software from the AUR (Arch User Repository). AUR packages are
community-maintained — a compromised or malicious AUR package is a real threat vector.
When you run `makepkg` on an AUR package, it runs arbitrary build scripts. TAORM monitors
every process's syscall behavior and flags anything that deviates from expected build-tool patterns.

#### How It Works

TAORM uses eBPF (via `bcc` — available in Arch's `extra` repository as `python-bcc`) to
attach a tracepoint to `sys_enter`. This gives per-syscall visibility with near-zero overhead,
unlike `ptrace` which doubles process context-switch overhead.

```
eBPF program (kernel side, < 512 instructions):
  - Fires on every syscall entry
  - Records: pid, syscall_nr, timestamp
  - Writes to a perf ring buffer

TAORM daemon (user side):
  - Reads from ring buffer in a tight loop
  - Maintains per-process syscall n-gram (n=5) sliding window
  - Computes anomaly score: -log P(syscall_nr | last_4_syscalls)
  - If rolling_mean(anomaly_score, window=50) > threshold:
      → classify threat type
      → apply seccomp filter
      → log to journald with full syscall trace
      → optionally notify user via libnotify
```

#### Threat Categories and Responses

| Threat | Syscall Signature | TAORM Response |
|--------|------------------|----------------|
| AUR build exfiltration | `execve("/bin/sh")` after `open("/home/*")` outside build dir | Alert + `seccomp` block `connect` syscall |
| Privilege escalation | `setuid(0)` or `setgid(0)` outside expected pattern | Immediate `seccomp` deny + kill |
| Crypto miner (from AUR) | High-frequency `futex` + unusual `mmap(PROT_EXEC)` | CPU cgroup hard limit (10%) + alert |
| Fork bomb | `clone()` rate > 50/sec sustained for > 2s | Kill process group + alert |
| Rootkit injection | `ptrace(PTRACE_ATTACH)` from non-debugger process | Block via `seccomp` + log |
| Suspicious browser plugin | Renderer process calls `socket(AF_INET)` unexpectedly | Alert (browsers manage their own network; renderer shouldn't call socket directly) |

#### AI Security Model

```
Input features (per process, rolling 50-syscall window):
  - syscall_ngram_score           (log probability under normal model)
  - unique_syscall_count          (entropy of syscall sequence)
  - sensitive_file_opens          (/etc, /home, /root accesses)
  - network_connect_from_build    (unexpected network in makepkg context)
  - exec_chain_depth              (how many exec() calls deep)
  - process_lineage               (parent → grandparent → ...)

Output:
  - anomaly_score [0.0–1.0]
  - threat_class  (BENIGN / SUSPICIOUS / MALICIOUS)
  - recommended_action (MONITOR / ALERT / RESTRICT / KILL)

Model type:   n-gram language model (n=5) for baseline
              + one-class SVM for threat classification
Baseline:     Built from first 120 seconds of system observation
              + pre-seeded with known-good patterns (shell, gcc, pacman, systemd)
False positive target:  < 2%
```

#### System Calls Used

| System Call / Interface | Purpose |
|-------------------------|---------|
| `eBPF` (via bcc tracepoint) | Zero-overhead syscall tracing — attaches to `sys_enter` |
| `seccomp(2)` + `prctl(PR_SET_SECCOMP)` | Apply syscall filter to flagged processes |
| `ptrace(2)` | Fallback if eBPF unavailable; detailed inspection of suspicious PIDs |
| `kill(2)` | Terminate processes confirmed as MALICIOUS |
| `open("/proc/[pid]/cmdline")` | Identify process identity for classification |

---

### 6.4 Module 4 — Power-State Disk I/O Manager

**Priority:** Medium | **Battery-aware**

#### The Problem

On a laptop, NVMe SSD disk I/O is fast but expensive — both in power and in heat. When
`pacman -Syu` runs a full system upgrade in the background while you compile, both compete
for NVMe bandwidth. On battery, this doubles power draw. TAORM assigns I/O priorities
dynamically based on power state and process context.

#### I/O Priority Policy

```
Process categories and their I/O priority:

  Interactive  (terminal, browser navigation):
    AC:      ioprio_set(IOPRIO_CLASS_RT, level=4)     ← real-time I/O class
    Battery: ioprio_set(IOPRIO_CLASS_BE, level=2)     ← best-effort, high

  Active compile job (make, cargo, cmake):
    AC:      ioprio_set(IOPRIO_CLASS_BE, level=4)     ← best-effort, medium
    Battery: ioprio_set(IOPRIO_CLASS_BE, level=6)     ← best-effort, lower

  Package manager (pacman, paru, yay):
    AC:      ioprio_set(IOPRIO_CLASS_BE, level=7)     ← best-effort, lowest
    Battery: ioprio_set(IOPRIO_CLASS_IDLE, 0)         ← idle — runs only when disk free

  Background indexers (updatedb, journald flush):
    Always:  ioprio_set(IOPRIO_CLASS_IDLE, 0)         ← idle class
```

#### AI Model: I/O Burst Predictor

The AI model predicts which processes will generate large I/O bursts in the next 2 seconds,
so TAORM can pre-set priorities before the burst hits rather than reacting to it.

```
Input features:
  - recent_io_bytes[5]            (last 5 measured I/O bursts per process)
  - process_phase                 (compile: parse/codegen/link detected by CPU pattern)
  - disk_queue_depth              (from /sys/block/nvme0n1/queue/nr_requests)
  - power_source                  (AC / battery)
  - battery_level_pct             (from /sys/class/power_supply/BAT0/capacity)

Output:
  - predicted_io_bytes_2s         (expected I/O in next 2 seconds)
  - suggested_ioprio_class        (RT / BE / IDLE)
  - suggested_ioprio_level        (0–7)
```

#### Disk Scheduling Algorithm Integration

On NVMe, Linux uses the `none` or `mq-deadline` I/O scheduler (no head seeking needed).
TAORM does not replace the scheduler but interacts with it via `io_uring` for async I/O
and `ioprio_set` for priority management. For any HDD (detected via
`/sys/block/*/queue/rotational`), TAORM switches to SSTF-informed read-ahead tuning.

#### System Calls Used

| System Call | Purpose |
|-------------|---------|
| `ioprio_set(2)` | Set I/O scheduling class and priority per process |
| `ioprio_get(2)` | Read current I/O priority of a process |
| `io_uring_setup(2)` | Set up async I/O ring for monitoring I/O completions |
| `fallocate(2)` | Pre-allocate disk space for compile artifacts when on AC |
| `sync_file_range(2)` | Control when dirty pages are flushed to NVMe |

---

## 7. System Call Map

Complete map of every system call TAORM makes, grouped by operation type:

```
OBSERVATION (kernel → TAORM, read-only):
  getrusage(2)          — CPU time per process
  clock_gettime(2)      — high-resolution timing
  mincore(2)            — page residency check
  ioprio_get(2)         — current I/O priority
  openat + read /proc   — /proc/[pid]/stat, smaps, cmdline, net/dev
  openat + read /sys    — thermal, power_supply, cpufreq

SCHEDULING CONTROL (TAORM → kernel):
  sched_setattr(2)      — set scheduling policy (DEADLINE, FIFO, IDLE)
  sched_getattr(2)      — read scheduling parameters
  setpriority(2)        — adjust nice value
  kill(SIGSTOP/CONT)    — pause/resume in thermal emergency

MEMORY CONTROL (TAORM → kernel):
  madvise(2)            — prefetch (MADV_WILLNEED) / release (MADV_FREE)
  mlock(2) / munlock    — pin/unpin critical pages
  sysctl write          — vm.swappiness, vm.dirty_ratio adjustment

SECURITY CONTROL (TAORM → kernel):
  ptrace(2)             — attach/detach for deep inspection
  seccomp(2)            — apply syscall filter to flagged processes
  prctl(2)              — set process capabilities
  kill(2)               — terminate MALICIOUS-class processes

I/O CONTROL (TAORM → kernel):
  ioprio_set(2)         — set I/O priority per process
  io_uring_setup(2)     — async I/O monitoring
  fallocate(2)          — pre-allocate on AC power
  sync_file_range(2)    — control flush timing

INTER-PROCESS (TAORM internal):
  eventfd(2)            — module-to-module event notification
  signalfd(2)           — clean signal handling in daemon
  timerfd_create(2)     — 500ms polling timer
  epoll_create1(2)      — multiplexed event loop over all fds
```

---

## 8. AI Model Design

### Guiding Constraints (Your Machine)

- **12 GB RAM** — AI models must not consume more than 50 MB total.
- **Laptop CPU** — prediction latency must be under 1ms (no GPU inference, no large models).
- **Arch rolling release** — no pinned Python versions; use stdlib + numpy only.
- **Single user** — no need for per-user models; one system-wide model set.

### Model Summary

| Module | Model | Why This Model | Size | Latency |
|--------|-------|----------------|------|---------|
| CPU / Thermal | Online decision tree (depth 8) | Interpretable, fast, handles non-linear thermal curves | < 2 KB | 0.2ms |
| Memory pressure | 3rd-order Markov chain | Memory pressure has strong temporal autocorrelation; Markov captures it exactly | < 100 KB | 0.4ms |
| Security (normal behavior) | 5-gram language model (hash table) | Proven in research; efficient O(1) lookup per syscall | < 5 MB (for 300 syscalls × 5-gram table) | 0.1ms |
| Security (threat classification) | One-class SVM (RBF kernel, 200 support vectors) | Trained only on normal data; automatically flags anything novel | < 500 KB | 0.8ms |
| I/O prediction | Online linear regression | I/O bursts have near-linear relationship with past bursts; simplest correct model | < 1 KB | 0.1ms |

### Training Pipeline

```
Phase 1 — Cold start (0–120 seconds after daemon starts):
  All models in OBSERVATION mode only.
  Collect baseline data: syscall sequences, burst times, memory patterns.
  No interventions made.

Phase 2 — Warm models (120 seconds onward):
  Models transition to ACTIVE mode.
  Predictions made and acted upon.
  Each outcome (actual burst time, actual pressure, threat/no-threat) fed back
  as a training sample → online update.

Phase 3 — Steady state:
  Models continuously improve.
  Security model re-baselines every 24 hours (rolling window, not full retrain).
  Config: /etc/taorm/models/ stores serialized model state.
  On reboot: models loaded from previous state — no 120s cold start after first run.
```

### Fallback Policy

When a model outputs uncertainty above threshold, TAORM falls back gracefully:

```c
decision_t make_decision(model_t *m, features_t *f) {
    prediction_t p = model_predict(m, f);
    if (p.confidence < CONFIDENCE_THRESHOLD) {
        // Don't intervene — let the kernel's default policy handle it
        return DECISION_DEFER_TO_KERNEL;
    }
    return apply_prediction(p);
}
```

This ensures TAORM never makes the system worse, even with an untrained model.

---

## 9. Arch Linux Integration

TAORM is designed as a first-class Arch citizen, not a Linux-generic tool ported to Arch.

### systemd Service Unit

```ini
# /etc/systemd/system/taormd.service
[Unit]
Description=TAORM — Thermal-Aware AI OS Resource Manager
After=multi-user.target
Wants=systemd-udevd.service

[Service]
Type=notify
ExecStart=/usr/bin/taormd --config /etc/taorm/taorm.conf
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s

# Security hardening
NoNewPrivileges=no        # needs capabilities for ptrace, sched_setattr
AmbientCapabilities=CAP_SYS_PTRACE CAP_SYS_NICE CAP_NET_ADMIN CAP_SYS_RESOURCE
CapabilityBoundingSet=CAP_SYS_PTRACE CAP_SYS_NICE CAP_NET_ADMIN CAP_SYS_RESOURCE
ProtectSystem=strict
ProtectHome=read-only
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

### cgroups v2 Slice

```bash
# /etc/systemd/system/taorm.slice
[Slice]
MemoryHigh=40M          # TAORM itself stays small
CPUWeight=50            # modest CPU weight — doesn't compete with user work
IOWeight=100            # normal I/O weight
```

### Configuration File

```toml
# /etc/taorm/taorm.conf

[thermal]
poll_interval_ms = 500
warm_threshold_C = 65
hot_threshold_C  = 75
critical_threshold_C = 82
emergency_threshold_C = 88
thermal_zone_path = "/sys/class/thermal/thermal_zone0/temp"

[memory]
pressure_yellow = 0.65
pressure_orange = 0.78
pressure_red    = 0.88
pressure_black  = 0.94
zram_size_gb    = 4
zram_algorithm  = "lz4"
browser_patterns = ["firefox", "chromium", "electron", "chrome"]
compiler_patterns = ["make", "gcc", "g++", "clang", "rustc", "cargo"]

[security]
ebpf_enabled     = true
ngram_n          = 5
anomaly_threshold = 0.72
baseline_seconds = 120
aur_watch        = true           # extra scrutiny on makepkg processes
notify_desktop   = true           # send libnotify alerts

[io]
nvme_device      = "/dev/nvme0n1"
power_supply_path = "/sys/class/power_supply/AC/online"

[models]
save_path        = "/var/lib/taorm/models/"
cold_start_s     = 120
confidence_min   = 0.60
```

### Pacman Hook (auto-install)

```ini
# /etc/pacman.d/hooks/taorm-reload.hook
[Trigger]
Operation = Upgrade
Type = Package
Target = linux

[Action]
Description = Reloading TAORM after kernel upgrade...
When = PostTransaction
Exec = /bin/systemctl restart taormd
```

### Dashboard (terminal)

A live terminal dashboard using Python `curses` showing real-time TAORM decisions:

```
┌─ TAORM v1.0 ─────── Arch Linux ─── 12.0 GB RAM ─── AC ──────────────────────┐
│ Thermal: 71°C [WARM]  ████████░░  CPU: 3.4 GHz (89% capacity)               │
│ Memory:  8.1 GB used  ████████░░  Pressure: 0.68 [YELLOW]  zram: 1.2/4.0 GB  │
├─ AI Decisions (last 10) ─────────────────────────────────────────────────────┤
│ 14:32:01  SCHED   firefox:renderer:4821  → SCHED_IDLE  (thermal hot, bg)     │
│ 14:32:01  MEM     firefox:renderer:5104  → MADV_FREE   (idle tab reclaim)    │
│ 14:32:03  IO      pacman:7823           → IOPRIO_IDLE  (battery: defer)      │
│ 14:31:58  SEC     [OK] gcc:9201 — normal syscall pattern                     │
│ 14:31:45  SCHED   make:8803            → SCHED_FIFO   (AC, active compile)   │
├─ Active Processes ──────────────────────────────────────────────────────────-┤
│ PID   NAME              CPU%  RSS MB   IOPRIO     SCHED       THREAT          │
│ 4821  firefox:renderer   2%   312 MB   BE/4       IDLE        BENIGN          │
│ 9201  gcc                45%  890 MB   BE/4       FIFO        BENIGN          │
│ 7823  pacman             1%   48 MB    IDLE       OTHER       BENIGN          │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. OS Topic Coverage Map

Mapping TAORM to all required CSE 323 topics:

| OS Topic | Where in TAORM | Implementation Detail |
|----------|---------------|----------------------|
| **Processes** | All modules | TAORM manages every process via PID; uses `fork()` for module workers; handles SIGCHLD |
| **CPU Scheduling** | Module 1 | `sched_setattr` with SCHED_DEADLINE / SCHED_FIFO / SCHED_IDLE based on thermal + AI |
| **Deadlocks** | Module 1 + Module 2 | Resource allocation graph (Module 1); `futex` wait-chain analysis (Module 2) |
| **Memory Management** | Module 2 | `madvise`, `mincore`, `mlock`; zram management; pressure prediction |
| **Disk Scheduling** | Module 4 | `ioprio_set` with RT/BE/IDLE classes; NVMe vs HDD policy; AI burst prediction |
| **Security** | Module 3 | eBPF syscall tracing; n-gram anomaly detection; `seccomp` enforcement |
| **IPC** | Daemon design | `eventfd` between modules; `epoll` event loop; systemd D-Bus for notifications |
| **Virtual Memory** | Module 2 | Page residency (`mincore`); zram swap; swappiness control |
| **I/O Management** | Module 4 | `io_uring` async I/O; `sync_file_range`; power-aware flush scheduling |
| **System Calls** | All modules | 20+ distinct syscalls; all interactions with kernel through standard POSIX interface |

---

## 11. Implementation Plan

### Phase 0 — Environment Setup (Day 1–2)

```bash
# Install required Arch packages
sudo pacman -S base-devel linux-headers bcc python-bcc \
               python-numpy iproute2 libnotify \
               perf powertop stress-ng

# Enable cgroups v2 (verify)
cat /sys/fs/cgroup/cgroup.controllers

# Verify eBPF works
sudo python /usr/share/bcc/tools/execsnoop
```

Deliverable: All tools verified working on your machine.

### Phase 1 — Sensor Layer and Daemon Skeleton (Week 1–2)

Build the core daemon with:
- `epoll`-based event loop (single-threaded, non-blocking)
- `/proc` reader for all running processes (CPU, memory, I/O stats)
- `/sys/class/thermal` poller (500ms interval)
- `/sys/class/power_supply` monitor (AC / battery state change events)
- `timerfd` for periodic wake-ups
- `signalfd` for graceful shutdown (SIGTERM, SIGHUP)
- Structured logging to `journald` via `sd_journal_send`

Deliverable: `taormd` daemon starts, reads all sensors, prints structured logs.

### Phase 2 — Module 1: CPU Scheduler (Week 3–4)

- Implement per-process burst measurement
- Build online decision tree in C (no external ML library needed — decision trees are 50 lines)
- Add thermal zone logic and `sched_setattr` calls
- Implement resource allocation graph for deadlock detection
- Add `futex` eBPF probe for deadlock via wait-chain

Deliverable: Benchmark — average CPU waiting time and turnaround time vs baseline CFS.
Run `stress-ng --cpu 8` and compare thermal profile with and without TAORM.

### Phase 3 — Module 2: Memory Manager (Week 5–6)

- Implement memory pressure zone tracker
- Build browser-tab process identifier (parse `/proc/*/cmdline`)
- Implement `madvise(MADV_FREE)` reclaim on yellow/orange transition
- Set up `zram` device management (modprobe, mkswap, swapon)
- Implement swappiness controller
- Build Markov chain pressure predictor

Deliverable: Run realistic test: `firefox` with 20 tabs + `make -j8` on Linux kernel source.
Measure: OOM events, page fault rate, build completion time vs baseline.

### Phase 4 — Module 3: Security Monitor (Week 7–8)

- Write eBPF program to trace `sys_enter` (in C, loaded via `bcc`)
- Build per-process n-gram table (hash map: 5-gram → count)
- Compute anomaly scores from n-gram log probabilities
- Implement one-class SVM in Python (scikit-learn), export as C-callable shared lib
- Wire `seccomp` filter application on threat detection
- Build simulated attack scripts for testing (privilege escalation, fork bomb, crypto miner)

Deliverable: Demo — inject simulated AUR malicious package. TAORM detects and blocks it.
Measure: detection latency (ms), false positive rate over 1-hour normal use.

### Phase 5 — Module 4: I/O Manager (Week 9)

- Implement process I/O category classifier (interactive / compile / package manager / background)
- Implement `ioprio_set` calls per category + power state
- Build online linear regression I/O burst predictor
- Read and respond to power source changes in real time

Deliverable: Benchmark — run `pacman -Syu` during active compile. Measure foreground
I/O latency on AC vs battery vs TAORM-managed.

### Phase 6 — Integration (Week 10)

- Connect all four modules through the central event bus (`eventfd`)
- Ensure thermal state drives all four modules simultaneously
- Build the `curses` dashboard
- Run full 24-hour stability test
- Tune AI model thresholds to minimize false positives

### Phase 7 — Benchmarks and Report (Week 11–12)

- Run full benchmark suite (see Section 13)
- Write final report
- Prepare 15-minute demo on your actual Arch laptop
- Record terminal dashboard video

---

## 12. Expected Results and Outcomes

### Functional Results

By project completion, TAORM will:

1. **Run stably as a `systemd` service** on Arch Linux, surviving kernel upgrades via the
   pacman hook, and restarting automatically on failure.

2. **Keep the laptop cooler under load** — by pre-empting background processes at 75°C
   (before firmware throttling at 85°C), the CPU spends more time at higher effective
   frequencies. Expected outcome: 6–10°C lower sustained temperature during compilation.

3. **Survive the browser-plus-build test** — `firefox` with 20 tabs open plus
   `make -j$(nproc)` on a medium C++ project (e.g., LLVM) will complete without OOM
   on 12 GB RAM. Without TAORM, this combination reliably OOM-kills either the build
   or a browser tab on stock Arch.

4. **Detect simulated AUR malicious behavior** — a test script that mimics a compromised
   AUR `PKGBUILD` (opens `/etc/shadow`, establishes outbound connection, attempts `setuid`)
   will be detected and blocked within 50ms of the anomalous syscall sequence.

5. **Extend battery life during light use** — on battery, background pacman updates,
   indexers, and compile jobs are deferred to `IOPRIO_IDLE` and `SCHED_IDLE`, reducing
   CPU and disk activity. Expected outcome: 8–12% reduction in power draw measured by
   `powertop --time=60`.

6. **Prevent all test-case deadlocks** — three injected circular-wait scenarios across
   `make` jobs sharing a lock file will be detected and resolved by the resource graph tracker.

### Quantitative Targets

| Metric | Without TAORM | With TAORM Target | Test Method |
|--------|--------------|-------------------|-------------|
| Peak CPU temp during `stress-ng --cpu 8` (60s) | ~91°C | ≤ 83°C | `sensors` (lm_sensors) |
| Page fault rate during browser + compile | baseline | ≥ 20% reduction | `perf stat -e page-faults` |
| OOM kills in browser-20-tabs + `make -j8` test | ≥ 1 kill | 0 kills | `/var/log/kernel` OOM count |
| Power draw (battery, light use, 10 min avg) | baseline | ≥ 10% reduction | `powertop --time=600` |
| AUR threat detection latency | N/A | < 50ms | TAORM internal timestamps |
| False positive rate (1h normal use) | N/A | < 2% of processes flagged | Manual audit |
| `make -j8` completion time (AC, LLVM small) | baseline | ≤ 5% slower (overhead acceptable) | `time make` |
| Foreground I/O latency during `pacman -Syu` (battery) | baseline | ≥ 15% reduction | `iostat -x 1` |

---

## 13. Benchmarks and Evaluation

### Test Environment

```
Machine:    Your Arch Linux laptop
Kernel:     6.x (current Arch default)
RAM:        12 GB (no DIMM swap — TAORM manages zram)
Storage:    NVMe SSD
Benchmark:  Each test run 3× — results averaged
Baseline:   Same machine, same workload, taormd.service stopped
```

### Benchmark Suite

**Benchmark 1 — Thermal Pressure Test**
```bash
# Run 8 CPU-bound workers for 60 seconds
stress-ng --cpu 8 --timeout 60s &
# Monitor temperature every 2 seconds
watch -n 2 sensors | grep 'Core 0'
# Compare: peak temp, sustained temp, frequency throttling events
```

**Benchmark 2 — Memory Pressure Test (the 12 GB test)**
```bash
# Open Firefox with 20 tabs (use selenium or manual)
# Then run medium C++ build
cd /tmp && git clone --depth=1 https://github.com/microsoft/mimalloc
cd mimalloc && mkdir build && cd build
cmake .. && time make -j$(nproc)
# Measure: OOM events, page fault count, build time
```

**Benchmark 3 — Battery Power Draw**
```bash
# Unplug AC, run light workload (browser + editor)
powertop --auto-tune  # just for measurement
powertop --csv=battery_test.csv --time=600
# Compare: average power draw (W) with and without TAORM
```

**Benchmark 4 — Security Detection Test**
```bash
# Simulated malicious process (safe — no actual exploit)
cat > /tmp/fake_malware.sh << 'SCRIPT'
#!/bin/bash
# Simulates privilege escalation attempt pattern
cat /etc/hostname     # benign read
cat /etc/hosts        # benign read
cat /etc/shadow 2>/dev/null || true   # anomalous — triggers TAORM
python3 -c "import socket; socket.create_connection(('8.8.8.8',53))"  # unexpected network
SCRIPT
bash /tmp/fake_malware.sh
# Measure: time from first anomalous syscall to TAORM alert in journald
```

**Benchmark 5 — I/O Priority Test**
```bash
# On battery, run pacman upgrade + foreground file operation simultaneously
sudo pacman -Syu &
dd if=/dev/zero of=/tmp/testfile bs=4K count=10000
# Measure: dd latency with and without TAORM I/O priority management
```

**Benchmark 6 — Deadlock Detection Test**
```bash
# Create circular file lock scenario between 3 make jobs
# (provided as test harness in project repo)
./tests/deadlock_test.sh
# Measure: time to detect and resolve each injected deadlock
```

### Metrics Collection Script

```bash
#!/bin/bash
# collect_metrics.sh — run before and after enabling taormd

echo "=== CPU ===" && cat /proc/loadavg
echo "=== Memory ===" && grep -E 'MemTotal|MemAvailable|PageTables' /proc/meminfo
echo "=== Faults ===" && grep -E 'pgfault|pgmajfault|oom' /proc/vmstat
echo "=== Thermal ===" && cat /sys/class/thermal/thermal_zone0/temp
echo "=== Freq ===" && cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
echo "=== Power ===" && cat /sys/class/power_supply/BAT0/power_now 2>/dev/null
```

---

## 14. Tools and Setup on Arch

### Required Packages

```bash
# Core build tools
sudo pacman -S base-devel linux-headers git cmake

# System monitoring and control
sudo pacman -S procps-ng iproute2 lm_sensors powertop stress-ng

# eBPF toolchain
sudo pacman -S bcc python-bcc bpf libbpf

# Python AI layer
sudo pacman -S python python-numpy python-scikit-learn python-psutil

# Notifications
sudo pacman -S libnotify

# Optional: better terminal dashboard
sudo pacman -S python-urwid

# AUR helper (for AUR security testing)
# install paru or yay as usual
```

### Project Directory Structure

```
taorm/
├── src/
│   ├── main.c              ← daemon main loop, epoll, signalfd
│   ├── sensor.c            ← /proc and /sys readers
│   ├── scheduler.c         ← Module 1: CPU scheduler
│   ├── memory.c            ← Module 2: memory manager
│   ├── security.c          ← Module 3: syscall monitor
│   ├── io_manager.c        ← Module 4: I/O manager
│   ├── ai/
│   │   ├── decision_tree.c ← online decision tree (CPU model)
│   │   ├── markov.c        ← Markov chain (memory model)
│   │   ├── ngram.c         ← n-gram model (security baseline)
│   │   └── linear.c        ← online linear regression (I/O model)
│   └── bpf/
│       └── syscall_trace.c ← eBPF program (kernel-side tracer)
├── python/
│   ├── svm_classifier.py   ← one-class SVM threat classifier
│   ├── dashboard.py        ← curses terminal dashboard
│   └── benchmark.py        ← benchmark runner and plotter
├── config/
│   └── taorm.conf          ← default configuration
├── systemd/
│   └── taormd.service      ← systemd unit file
├── tests/
│   ├── deadlock_test.sh    ← deadlock injection test
│   ├── memory_test.sh      ← OOM pressure test
│   └── security_test.sh    ← simulated threat test
├── Makefile
└── README.md
```

### Build and Run

```bash
# Build
cd taorm && make

# Install
sudo make install
sudo systemctl enable --now taormd

# Watch logs
journalctl -u taormd -f

# Launch dashboard
python python/dashboard.py

# Run benchmarks
sudo python python/benchmark.py --all --output results/
```

---

## 15. References

1. **Arch Wiki — Power Management:**
   https://wiki.archlinux.org/title/Power_management —
   Primary reference for `/sys/class/power_supply`, TLP, and battery management on Arch.

2. **Arch Wiki — Improving Performance:**
   https://wiki.archlinux.org/title/Improving_performance —
   Reference for `ioprio_set`, I/O schedulers, cgroups v2, and zram on Arch.

3. **Arch Wiki — cgroups:**
   https://wiki.archlinux.org/title/Cgroups —
   Reference for cgroups v2 hierarchy, systemd slice integration.

4. **Linux man pages:** `man 2 sched_setattr`, `man 2 madvise`, `man 2 ptrace`,
   `man 2 seccomp`, `man 2 mincore`, `man 2 ioprio_set`, `man 2 io_uring_setup` —
   Primary reference for all system call interfaces used.

5. **Love, R. (2010).** *Linux Kernel Development, 3rd Edition.* Addison-Wesley. —
   Chapters 4 (Scheduling), 12 (Memory Management), 14 (Block I/O).

6. **Silberschatz, A., Galvin, P. B., & Gagne, G. (2018).** *Operating System Concepts, 10th Ed.* Wiley. —
   Chapters 5 (CPU Scheduling), 9 (Virtual Memory), 11 (Storage Management), 17 (Security).

7. **Forrest, S. et al. (1996).** *A Sense of Self for Unix Processes.*
   IEEE Symposium on Security and Privacy. —
   Foundational n-gram syscall anomaly detection paper underpinning Module 3.

8. **Gregg, B. (2020).** *Systems Performance: Enterprise and the Cloud, 2nd Ed.* Pearson. —
   Reference for `perf`, thermal profiling, I/O analysis, and `io_uring`.

9. **eBPF documentation and BCC reference:**
   https://ebpf.io and https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md —
   Primary reference for the eBPF-based syscall tracer in Module 3.

10. **Linux kernel documentation — zram:**
    https://www.kernel.org/doc/html/latest/admin-guide/blockdev/zram.html —
    Reference for `zram` setup and `comp_algorithm` configuration in Module 2.

11. **Mao, H. et al. (2019).** *Learning Scheduling Algorithms for Data Processing Clusters.*
    ACM SIGCOMM 2019. —
    Industry validation that replacing OS scheduling heuristics with ML is production-viable.

12. **Linux kernel documentation — I/O schedulers:**
    https://www.kernel.org/doc/html/latest/block/iosched.html —
    Reference for `mq-deadline`, `bfq`, and NVMe `none` scheduler interaction in Module 4.

---

*Document version: 2.0 — Redesigned for Arch Linux, 12 GB RAM, laptop environment*
*Course: CSE 323 — Operating Systems Design*
*Date: June 2026*
