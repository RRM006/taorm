# AI-Aware OS Resource Manager (AIORM)
### CSE 323 — Operating Systems Design | Project Design Document

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Problem Statement](#2-problem-statement)
3. [Goals and Objectives](#3-goals-and-objectives)
4. [System Architecture](#4-system-architecture)
5. [Core Modules](#5-core-modules)
   - 5.1 [AI CPU Scheduler](#51-module-1--ai-cpu-scheduler)
   - 5.2 [Predictive Memory Manager](#52-module-2--predictive-memory-manager)
   - 5.3 [AI Security Monitor](#53-module-3--ai-security-monitor)
   - 5.4 [AI Network QoS Manager](#54-module-4--ai-network-qos-manager)
6. [System Call Design](#6-system-call-design)
7. [AI Model Design](#7-ai-model-design)
8. [Device Optimization Strategy](#8-device-optimization-strategy)
9. [Security Considerations](#9-security-considerations)
10. [Implementation Plan](#10-implementation-plan)
11. [Expected Outcomes and Results](#11-expected-outcomes-and-results)
12. [Evaluation Metrics](#12-evaluation-metrics)
13. [Tools and Technologies](#13-tools-and-technologies)
14. [References](#14-references)

---

## 1. Project Overview

**Project Name:** AI-Aware OS Resource Manager (AIORM)

**Course:** CSE 323 — Operating Systems Design

**Type:** Kernel-level / System-level implementation with AI integration

**Platform:** Linux (Ubuntu 22.04 LTS or later), x86-64

**Language:** C (primary), Python (AI model training and inference bridge)

AIORM is a kernel-adjacent, user-space daemon that intercepts OS resource management decisions — CPU scheduling, memory allocation, disk I/O, and network bandwidth — and replaces static, rule-based policies with adaptive, AI-driven ones. Instead of the OS always applying the same fixed algorithm (e.g., Round-Robin or FCFS), AIORM trains lightweight machine learning models on live system behavior and feeds predictions back into the kernel through real Linux system calls.

This project addresses a real and growing challenge in modern computing: AI workloads running on the OS create unique resource pressures that traditional schedulers and memory managers were never designed for. AIORM treats the OS resource manager itself as an AI system.

---

## 2. Problem Statement

Traditional operating systems manage resources using algorithms designed in the 1970s–1990s: fixed-priority scheduling, LRU page replacement, SCAN disk scheduling. These algorithms have two fundamental limitations:

**1. They use no historical context.** A Round-Robin scheduler gives every process the same time slice regardless of whether that process has always completed in 2ms or always needs 50ms. Every new scheduling decision starts from scratch.

**2. They cannot adapt to device type at runtime.** The same kernel runs on a 2GB RAM smartphone and a 512GB RAM server, yet uses nearly identical memory management logic. The policies are tuned at compile time, not at runtime per device.

When AI workloads (LLM inference, ML training, computer vision pipelines) are added to the system, these limitations become severe:

- AI processes have irregular, bursty CPU usage that confuses schedulers.
- AI workloads allocate and free gigabytes of memory unpredictably, causing page thrashing.
- AI-driven decisions open new security vulnerabilities because behavior is non-deterministic and harder to whitelist.
- Static network QoS rules cannot distinguish an AI inference request (latency-sensitive) from a bulk data transfer (throughput-sensitive).

**AIORM solves these problems** by inserting an AI prediction layer between the OS and its resource management decisions.

---

## 3. Goals and Objectives

### User Goals
| Goal | Description |
|------|-------------|
| Responsive applications | Users experience lower latency in foreground apps even under system load |
| No unexpected freezes | Memory pressure is predicted and relieved before OOM kills occur |
| Security by default | Malicious or anomalous processes are detected and blocked before causing damage |
| Fair network access | Interactive applications get priority over background bulk transfers |

### System Goals
| Goal | Description |
|------|-------------|
| Minimize CPU idle waste | Ensure no core sits idle while runnable processes wait |
| Reduce page fault rate | Prefetch pages before processes need them |
| Detect intrusions in real time | Flag anomalous syscall sequences within milliseconds |
| Prevent deadlocks proactively | Predict deadlock risk before it occurs rather than detecting it after |
| Optimize per device class | Apply different resource policies for mobile, desktop, and server targets |

### Project-Specific Goals
- Implement and demonstrate all four resource management modules with working code.
- Use actual Linux system calls in every module — no simulation of kernel interfaces.
- Show measurable improvement over baseline OS behavior through benchmarks.
- Produce a system that is extensible — new AI models can be plugged in without rewriting the manager.

---

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  USER SPACE (Applications)              │
│     Browser   Database   Game Engine   AI Workload      │
└─────────────────────┬───────────────────────────────────┘
                      │  system calls (read, write, fork…)
┌─────────────────────▼───────────────────────────────────┐
│              AIORM DAEMON (your project)                │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ AI CPU      │  │ AI Memory    │  │ AI Security   │  │
│  │ Scheduler   │  │ Manager      │  │ Monitor       │  │
│  └──────┬──────┘  └──────┬───────┘  └───────┬───────┘  │
│         │                │                   │          │
│  ┌──────▼────────────────▼───────────────────▼───────┐  │
│  │            AI Decision Engine                     │  │
│  │   (4 ML models — online learning, C + Python)    │  │
│  └──────────────────────┬────────────────────────────┘  │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐  │
│  │        /proc filesystem reader  (live feed)       │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────┬───────────────────────────────────┘
                      │  sched_setattr, madvise, ptrace,
                      │  seccomp, io_uring, netlink…
┌─────────────────────▼───────────────────────────────────┐
│                LINUX KERNEL                             │
│   Scheduler │ Memory Manager │ VFS │ Net Stack │ IPC    │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│              HARDWARE                                   │
│        CPU │ RAM │ Disk │ NIC │ GPU                     │
└─────────────────────────────────────────────────────────┘
```

### Key Architectural Decisions

**AIORM runs in user space, not kernel space.** Modifying the kernel is dangerous and difficult to debug. Instead, AIORM uses Linux's rich user-space interface (ptrace, /proc, cgroups, eBPF) to observe and influence kernel behavior from outside. This is the same approach used by production tools like `systemd`, `cgroups managers`, and `Linux perf`.

**AI models are kept small and fast.** Each model must make a decision in under 1ms — latency matters more than accuracy in an OS context. AIORM uses decision trees and online linear models, not deep neural networks.

**Fallback to default OS behavior.** If the AI model is uncertain or has insufficient data, AIORM falls back to the kernel's built-in policy. The system degrades gracefully, never worse than the baseline OS.

---

## 5. Core Modules

### 5.1 Module 1 — AI CPU Scheduler

**Priority:** High (essential core function)

#### What It Does
AIORM monitors every process's CPU usage history using `getrusage()` and `/proc/[pid]/stat`. A lightweight machine learning model (online linear regression or decision tree) is trained on this data to predict the next CPU burst duration for each process. These predictions are fed to a priority-aware scheduling policy applied via `sched_setattr()`.

Processes predicted to need short CPU bursts are given higher scheduling priority (Shortest Job First style), while long-burst processes are deprioritized. Unlike traditional SJF, the burst prediction is continuously updated — the AI model adapts as process behavior changes.

#### Process Flow
```
1. Read /proc/[pid]/stat for all running processes (every 100ms)
2. Extract: recent CPU time, context switch count, voluntary yields
3. Feed features into burst-prediction model
4. Model outputs: predicted_burst_ms for each process
5. Call sched_setattr(pid, SCHED_DEADLINE or SCHED_OTHER with dynamic nice value)
6. Log prediction vs actual burst → retrain model
```

#### Deadlock Handling
A separate sub-component of this module tracks resource allocation using a directed graph (`process → resource` edges). When a cycle is detected (deadlock), or when the AI model predicts a high deadlock risk based on current wait patterns, AIORM either:
- Denies the resource request preemptively (Banker's Algorithm style), or
- Selects the lowest-priority process in the cycle for termination.

#### System Calls Used
| System Call | Purpose |
|-------------|---------|
| `sched_setattr(2)` | Apply custom scheduling policy to a process |
| `sched_getattr(2)` | Read current scheduling parameters |
| `getrusage(2)` | Read CPU time consumed by a process |
| `clock_gettime(2)` | High-resolution burst timing |
| `setpriority(2)` | Adjust nice value based on AI prediction |

---

### 5.2 Module 2 — Predictive Memory Manager

**Priority:** High (essential core function) | **Focus optimization module**

This is the module optimized across all three device classes.

#### What It Does
AIORM monitors which memory pages each process is currently using via `mincore()` and `/proc/[pid]/smaps`. A prediction model learns each process's memory access pattern — specifically, which pages it will need in the near future. Based on these predictions:

- Pages predicted to be needed soon are prefetched using `madvise(MADV_WILLNEED)`.
- Pages not expected to be accessed are released using `madvise(MADV_FREE)`.
- On memory pressure, the AI model selects which process's pages to reclaim first based on predicted future access likelihood.

#### Process Flow
```
1. Read /proc/[pid]/smaps for each process (every 500ms)
2. Record which pages are in RAM, which are swapped
3. Build access sequence per process: [page_addr_1, page_addr_2, ...]
4. Feed sequence into Markov chain or LSTM model
5. Model outputs: predicted_next_pages[] for each process
6. Call madvise(MADV_WILLNEED, predicted_pages) — prefetch
7. On low memory: madvise(MADV_FREE, least_likely_pages) — reclaim
```

#### Device-Specific Optimization
| Device Class | Memory Policy | Model Size | Target Metric |
|--------------|--------------|------------|---------------|
| Mobile (≤4GB RAM) | Aggressive reclaim of background apps | Decision tree, depth 4 | 30% fewer OOM kills |
| Desktop (8–32GB) | Balanced prefetch + reclaim | Decision tree, depth 8 | 20% lower page fault rate |
| Server (≥64GB) | Maximum prefetch, fill all available RAM | Small LSTM (3 layers) | 25% throughput improvement |

AIORM detects device class at startup by reading `/proc/meminfo` and the number of CPU cores, then selects the appropriate model configuration.

#### System Calls Used
| System Call | Purpose |
|-------------|---------|
| `madvise(2)` | Prefetch or release memory pages |
| `mincore(2)` | Check which pages are currently in RAM |
| `mmap(2)` | Map memory regions for monitoring |
| `mlock(2)` | Pin critical pages to prevent swapping |
| `process_vm_readv(2)` | Read another process's memory metadata |

---

### 5.3 Module 3 — AI Security Monitor

**Priority:** High (essential core function)

#### What It Does
Every process makes a sequence of system calls as it runs. Normal processes have predictable syscall patterns — a web server mostly calls `accept`, `read`, `write`, `close`. A malicious process deviates: it might call `ptrace`, `setuid`, `execve` in unusual combinations suggesting privilege escalation, or call `open` on `/etc/shadow` which a browser has no business doing.

AIORM traces the syscall sequence of every process using `ptrace()` (development) or eBPF (production). An n-gram language model is trained on normal syscall sequences. When a process's syscall sequence produces a low probability under the normal model (anomaly score above threshold), AIORM flags it and optionally applies a `seccomp` filter to block further dangerous calls.

This module also handles the AI-specific security risk: AI models making non-deterministic decisions can mask malicious behavior. AIORM applies extra scrutiny to processes that spawn subprocesses (a common pattern in AI inference pipelines that can be exploited).

#### Process Flow
```
1. Attach to all processes using ptrace(PTRACE_SYSCALL)
   (or load eBPF program for zero-overhead production use)
2. Record syscall ID sequence per process: [read, write, open, read, close, ...]
3. Build n-gram frequency model on first 60 seconds (baseline phase)
4. Compute anomaly score for each new syscall: -log P(syscall | last n syscalls)
5. If rolling_average(anomaly_score) > threshold:
      → alert: log PID, process name, anomalous syscall sequence
      → optionally: apply seccomp whitelist filter to block further calls
6. For deadlock risk: maintain resource allocation graph
      → if cycle detected or ML risk > 0.8: intervene
```

#### Threat Models Covered
| Threat | Detection Method |
|--------|-----------------|
| Privilege escalation | `setuid`/`setgid` outside expected pattern |
| Data exfiltration | Unusual `open` + `read` on sensitive paths |
| Rootkit injection | `ptrace` calls from non-debugger processes |
| Fork bomb | Rapid `clone`/`fork` calls exceeding baseline rate |
| AI model poisoning | Subprocess spawning with unusual argument patterns |

#### System Calls Used
| System Call | Purpose |
|-------------|---------|
| `ptrace(2)` | Intercept and inspect every syscall a process makes |
| `seccomp(2)` | Apply syscall whitelist/blacklist to flagged processes |
| `prctl(2)` | Set process security attributes and capabilities |
| `kill(2)` | Terminate processes that exceed threat threshold |
| `waitpid(2)` | Wait for traced process to stop at syscall boundary |

---

### 5.4 Module 4 — AI Network QoS Manager

**Priority:** Medium (system reliability and usability)

#### What It Does
AIORM reads per-process network statistics from `/proc/net/tcp` and `/proc/[pid]/net/dev`. A classification model categorizes each process's traffic as one of: interactive (low-latency), bulk-transfer (high-throughput), or background (best-effort). Linux's `tc` (Traffic Control) tool is then used via netlink sockets to apply appropriate QoS rules — interactive processes get priority queuing, bulk transfers get rate limiting.

The AI component predicts congestion before it happens: if a bulk transfer process is about to saturate the link (based on its send rate trend), AIORM preemptively throttles it before latency for interactive processes spikes.

#### System Calls / APIs Used
| Interface | Purpose |
|-----------|---------|
| `socket(AF_NETLINK)` | Communicate with kernel's traffic control subsystem |
| `setsockopt(2)` | Apply socket-level QoS settings |
| `/proc/net/tcp` | Read per-connection statistics |
| `tc` (via netlink) | Apply qdisc rules: HTB, fq_codel |

---

## 6. System Call Design

All four modules interact with the kernel exclusively through standard POSIX/Linux system calls. No kernel modules are written. This design is intentional: it mirrors how production infrastructure tools (systemd, Docker, cgroups managers) work.

### System Call Summary Table

| Module | System Call | Frequency | Data Direction |
|--------|-------------|-----------|----------------|
| CPU Scheduler | `sched_setattr` | Per scheduling cycle | AIORM → Kernel |
| CPU Scheduler | `getrusage` | Every 100ms | Kernel → AIORM |
| Memory Manager | `madvise` | Per prediction | AIORM → Kernel |
| Memory Manager | `mincore` | Every 500ms | Kernel → AIORM |
| Security Monitor | `ptrace` | Per syscall of traced process | Both directions |
| Security Monitor | `seccomp` | On threat detection | AIORM → Kernel |
| Security Monitor | `prctl` | On process start | AIORM → Kernel |
| Network QoS | `socket(AF_NETLINK)` | Per QoS update | AIORM → Kernel |
| All modules | `/proc` filesystem | Continuous | Kernel → AIORM |

### System Call Interaction Pattern (per module cycle)

```c
// Example: Memory Manager cycle (simplified)
void memory_manager_cycle(pid_t pid) {
    // 1. Read current page state from kernel
    unsigned char *vec = malloc(page_count);
    mincore(addr, length, vec);           // kernel → AIORM
    
    // 2. Feed into AI model
    float *features = extract_features(vec, page_count);
    int *predicted_pages = model_predict(features);  // AI decision
    
    // 3. Act on prediction via system call
    madvise(predicted_pages, size, MADV_WILLNEED);   // AIORM → kernel
    
    free(vec);
}
```

---

## 7. AI Model Design

### Design Principles
- Models must make predictions in **under 1 millisecond** (OS latency requirements).
- Models use **online learning** — they update continuously as new data arrives, without retraining from scratch.
- Models have **explicit uncertainty outputs** — when unsure, fall back to default OS policy.
- Models are **explainable** — a decision tree's choices can be printed for debugging.

### Model Specifications

| Module | Model Type | Input Features | Output | Update Frequency |
|--------|-----------|---------------|--------|-----------------|
| CPU Scheduler | Online decision tree (depth 6) | Recent burst times, context switch rate, voluntary yields, process type | Predicted burst (ms), suggested priority | Per scheduling event |
| Memory Manager | Markov chain (order 3) | Page access sequence, recency, process type | Probability of page access in next 500ms | Every 500ms |
| Security Monitor | n-gram language model (n=5) | Syscall ID sequence | Anomaly score (0.0–1.0) | Per syscall |
| Network QoS | Naive Bayes classifier | Packet rate, avg packet size, inter-arrival time variance | Traffic class (interactive/bulk/background) | Every 1 second |

### Training Data Sources
All training data comes from live system observation — no external datasets are used.

| Data Source | What It Provides |
|-------------|-----------------|
| `/proc/[pid]/stat` | CPU burst durations, context switch counts |
| `/proc/[pid]/smaps` | Page residency, access patterns |
| `ptrace` syscall trace | Syscall ID sequences per process |
| `/proc/net/tcp` | Network connection statistics |

### AI Lifecycle

```
[System Startup]
      │
      ▼
[Observation Phase] ── 60 seconds ──► Build baseline model
      │
      ▼
[Active Phase] ──────────────────────► Predict + Act + Log
      │                                         │
      │                               [Outcome logged]
      │                                         │
      └────────── Model updates continuously ◄──┘
```

---

## 8. Device Optimization Strategy

AIORM selects one subsystem — the **Memory Manager** — as the device-specific optimization focus. The same codebase applies different policies depending on detected device class.

### Device Detection (startup)

```c
typedef enum { DEVICE_MOBILE, DEVICE_DESKTOP, DEVICE_SERVER } device_class_t;

device_class_t detect_device() {
    long total_ram_mb = read_total_ram();   // from /proc/meminfo
    int cpu_cores     = get_nprocs();
    
    if (total_ram_mb < 4096)  return DEVICE_MOBILE;
    if (total_ram_mb < 32768) return DEVICE_DESKTOP;
    return DEVICE_SERVER;
}
```

### Per-Device Memory Policy

**Mobile (≤4GB RAM)**
- Goal: survive under memory pressure, prevent OOM kills.
- Policy: Aggressive reclaim. After any process goes to background, AIORM calls `madvise(MADV_FREE)` on its pages within 10 seconds.
- Model: Shallow decision tree (depth 4). Trained only on most recent 30 events — fast to retrain, low memory footprint.
- cgroup limit: Background app group capped at 20% of total RAM.

**Desktop (8–32GB RAM)**
- Goal: balance responsiveness and memory efficiency.
- Policy: Moderate prefetch for active processes, lazy reclaim for idle ones.
- Model: Decision tree (depth 8). Balances prediction quality vs overhead.
- cgroup limit: None by default; applied when memory > 80% full.

**Server (≥64GB RAM)**
- Goal: maximize throughput, minimize I/O wait.
- Policy: Aggressive prefetch. Fill all available free RAM with predicted future pages.
- Model: Small LSTM (3 hidden layers, 32 units each). Higher accuracy worth the extra compute cost.
- cgroup limit: Per-workload limits configured at launch.

---

## 9. Security Considerations

Integrating AI into OS resource management introduces new security concerns that AIORM must itself address:

| Risk | AIORM Mitigation |
|------|-----------------|
| AI model poisoning via crafted process behavior | Outlier detection on training data; reject anomalous training samples |
| False positive — legitimate process blocked | Confidence threshold tuning; alert without blocking until confidence > 0.95 |
| AIORM daemon itself is compromised | AIORM runs as a dedicated service account with minimum capabilities via `prctl(PR_SET_SECUREBITS)` |
| Adversarial syscall sequences that evade n-gram model | Maintain a hard-coded blacklist of absolutely forbidden syscall combinations in addition to the ML model |
| Resource starvation of AIORM daemon | AIORM's own processes are pinned to a dedicated cgroup with guaranteed CPU and memory |
| Privacy — AI models learn sensitive process behavior | All training data stays in-process memory; never written to disk; models destroyed on daemon shutdown |

---

## 10. Implementation Plan

### Phase 1 — Infrastructure (Weeks 1–2)
- Set up the AIORM daemon process with a main loop and signal handling.
- Implement the `/proc` reader: parse `/proc/[pid]/stat`, `/proc/[pid]/smaps`, `/proc/meminfo` for all running processes.
- Implement the logging system: timestamped structured logs for all AI decisions and outcomes.
- Deliverable: a daemon that prints a live table of per-process CPU and memory stats every second.

### Phase 2 — CPU Scheduler Module (Weeks 3–4)
- Implement burst-time measurement using `clock_gettime` and `getrusage`.
- Implement and train the burst-prediction decision tree in C.
- Wire predictions to `sched_setattr` calls.
- Benchmark: compare average process waiting time and turnaround time against the default Linux CFS scheduler under a mixed workload.
- Deliverable: benchmark chart showing AI scheduler vs baseline.

### Phase 3 — Memory Manager Module (Weeks 5–6)
- Implement page residency reader using `mincore`.
- Implement the Markov chain page-prediction model.
- Wire predictions to `madvise(MADV_WILLNEED)` and `madvise(MADV_FREE)`.
- Test device optimization: simulate mobile (2GB), desktop (16GB), server (64GB) with memory limits via cgroups.
- Deliverable: chart showing page fault rate and OOM event count across three device configs.

### Phase 4 — Security Monitor Module (Weeks 7–8)
- Implement `ptrace`-based syscall tracer for all processes.
- Implement n-gram language model for syscall sequence anomaly detection.
- Wire anomaly detection to `seccomp` filter application.
- Implement deadlock detection via resource allocation graph.
- Test: inject a simulated malicious process (privilege escalation pattern), verify detection and blocking.
- Deliverable: demo video/screenshots showing malicious process blocked within milliseconds.

### Phase 5 — Network QoS Module (Week 9)
- Implement `/proc/net` reader for per-process traffic stats.
- Implement Naive Bayes traffic classifier.
- Wire classifications to `tc` rules via netlink socket.
- Test: run interactive process + bulk download simultaneously; measure latency improvement.
- Deliverable: latency comparison chart with and without QoS module.

### Phase 6 — Integration and Evaluation (Weeks 10–11)
- Run all four modules simultaneously.
- Conduct full benchmark suite across all device classes.
- Build a simple terminal dashboard showing live AI decisions.
- Stress test for stability: run for 24 hours without crash or performance regression.

### Phase 7 — Report and Presentation (Week 12)
- Write final project report.
- Prepare 15-minute demonstration.
- Document lessons learned and limitations.

---

## 11. Expected Outcomes and Results

### Functional Outcomes
By the end of this project, AIORM will:

1. **Run as a stable daemon** on Linux, monitoring all processes in real time without crashing or interfering with normal system operation.

2. **Improve CPU scheduling efficiency** — demonstrate that the AI-predicted scheduling policy achieves lower average waiting time and turnaround time than the default CFS scheduler for a mixed workload of short and long CPU-bound tasks.

3. **Reduce memory-related performance problems** — show measurably fewer page faults and OOM events compared to baseline, across at least two of the three device classes (mobile and server).

4. **Detect and block simulated security threats** — the n-gram anomaly detector successfully identifies a crafted malicious process and blocks it via `seccomp` before it completes a simulated privilege escalation sequence.

5. **Improve interactive network latency under load** — the QoS module preserves interactive application response times when a bulk file transfer is running concurrently.

6. **Prevent deadlocks** — the resource allocation graph tracker detects at least one simulated circular wait and resolves it before a deadlock fully forms.

### Quantitative Performance Targets

| Module | Metric | Baseline | Target | Measurement Method |
|--------|--------|----------|--------|--------------------|
| CPU Scheduler | Average waiting time (ms) | CFS default | ≥15% reduction | `time` command, custom benchmark script |
| CPU Scheduler | Turnaround time (ms) | CFS default | ≥10% reduction | Process exit timestamp logging |
| Memory Manager (mobile) | OOM kill events / hour | Linux default | ≥30% reduction | `/proc/vmstat` OOM counter |
| Memory Manager (server) | Page fault rate (faults/sec) | Linux default | ≥20% reduction | `perf stat -e page-faults` |
| Security Monitor | Threat detection time (ms) | N/A (no baseline monitor) | < 50ms from syscall to alert | AIORM internal timestamps |
| Security Monitor | False positive rate | N/A | < 2% of legitimate processes flagged | Manual review of 100-process test run |
| Network QoS | Interactive app latency under load (ms) | Without QoS | ≥25% reduction | `ping` RTT measurement |
| Deadlock Prevention | Deadlocks resolved | N/A | 100% of injected test deadlocks resolved | Test harness |

### Qualitative Outcomes

- A deeper practical understanding of how Linux system calls are used to influence kernel behavior from user space.
- Experience implementing online machine learning models under strict latency constraints.
- Hands-on exposure to the real tension between AI adaptability and OS security requirements.
- A working codebase that serves as a foundation for a graduate-level kernel research project or industry internship portfolio piece.

---

## 12. Evaluation Metrics

### Benchmark Setup
All benchmarks run on a single machine (or VM) with identical hardware/software configuration. Each benchmark runs three times and results are averaged.

**Benchmark workload profiles:**

| Profile | Description | Used to test |
|---------|-------------|-------------|
| Mixed CPU | 10 CPU-bound processes with varying burst lengths | CPU scheduler |
| Memory pressure | 5 processes allocating/freeing large arrays | Memory manager |
| Adversarial | 1 malicious process + 9 normal processes | Security monitor |
| Network contention | 1 interactive process (ping/HTTP) + 1 bulk transfer (scp) | Network QoS |
| Deadlock scenario | Circular resource request pattern across 4 processes | Deadlock prevention |

### Metrics Collected

```
For each benchmark run, record:
  - wall_clock_time (seconds)
  - cpu_utilization_percent (average across all cores)
  - page_fault_count (from /proc/vmstat)
  - context_switches_per_second
  - oom_events (from /var/log/kern.log)
  - anomaly_detections (AIORM log)
  - false_positives (manual review)
  - network_latency_p50_ms, p95_ms, p99_ms
  - ai_decision_latency_us (AIORM internal timing)
```

### Comparison Baseline
Every metric is measured in two conditions:
1. **Baseline:** Standard Linux with no AIORM daemon running.
2. **AIORM active:** AIORM daemon running with all four modules enabled.

A 10% improvement in any metric is considered a meaningful result. A 20% improvement in the memory manager (the focused optimization module) is the primary success criterion.

---

## 13. Tools and Technologies

| Category | Tool / Technology | Purpose |
|----------|------------------|---------|
| Language | C (C11 standard) | All modules, system call interface, AI models |
| Language | Python 3.10+ | Model prototyping, training data analysis, visualization |
| OS | Linux 6.x (Ubuntu 22.04) | Target platform |
| Kernel interface | `/proc` filesystem | Live process and system statistics |
| Kernel interface | `ptrace(2)` | Syscall tracing for security module |
| Kernel interface | `eBPF` (optional, advanced) | Zero-overhead syscall tracing (production-grade replacement for ptrace) |
| Resource control | `cgroups v2` | Hard resource limits per process group |
| Network control | `tc` (iproute2) via netlink | Traffic shaping and QoS rules |
| Build system | `make` | Build automation |
| Version control | `git` | Source code management |
| Benchmarking | `perf`, `time`, `sar` | Performance measurement |
| Visualization | Python `matplotlib` | Benchmark result charts |
| Documentation | Markdown | Design documents and reports |

### Dependencies
```bash
# Required packages (Ubuntu/Debian)
sudo apt install build-essential linux-tools-$(uname -r) \
     iproute2 bpfcc-tools python3 python3-pip \
     libseccomp-dev

# Python packages
pip install numpy matplotlib scikit-learn
```

---

## 14. References

1. **Linux man pages:** `man 2 ptrace`, `man 2 madvise`, `man 2 sched_setattr`, `man 2 seccomp`, `man 2 mincore` — primary reference for all system call interfaces.

2. **Love, R. (2010).** *Linux Kernel Development, 3rd Edition.* Addison-Wesley. — Reference for understanding CFS scheduler internals and memory management subsystems.

3. **Silberschatz, A., Galvin, P. B., & Gagne, G. (2018).** *Operating System Concepts, 10th Edition.* Wiley. — Theoretical foundation for scheduling algorithms, memory management, deadlock handling, and security models.

4. **Tanenbaum, A. S. (2014).** *Modern Operating Systems, 4th Edition.* Pearson. — Reference for virtual memory, IPC, and file system concepts.

5. **Forrest, S., Hofmeyr, S. A., Somayaji, A., & Longstaff, T. A. (1996).** *A Sense of Self for Unix Processes.* IEEE Symposium on Security and Privacy. — Foundational paper for syscall-based anomaly detection (n-gram approach used in Module 3).

6. **Mao, H., et al. (2019).** *Learning Scheduling Algorithms for Data Processing Clusters.* ACM SIGCOMM. — Industry example of replacing OS scheduling with ML models; validates the AIORM approach.

7. **Google Borg Scheduler** — Public documentation on AI-assisted cluster scheduling: https://research.google/pubs/pub43438/

8. **Linux cgroups v2 documentation:** https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html

9. **eBPF documentation:** https://ebpf.io/what-is-ebpf/ — Reference for the production-grade syscall tracing approach in Module 3.

10. **io_uring documentation:** https://kernel.dk/io_uring.pdf — Reference for asynchronous I/O interface used in disk scheduling component.

---

*Document version: 1.0*
*Course: CSE 323 — Operating Systems Design*
*Last updated: June 2026*
