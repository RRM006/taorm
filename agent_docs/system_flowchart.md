# TAORM — System Flowchart

## High-Level Architecture

```
+----------------------------------------------------------+
|                  USER SPACE (your apps)                   |
|   Firefox   |  VS Code  |  Docker  |  Terminal  |  make  |
+-----------------------------+----------------------------+
                              | normal system calls
                              v
+----------------------------------------------------------+
|               TAORM DAEMON  (taormd)                      |
|                                                            |
|  +----------------------------------------------------+   |
|  |           SENSOR LAYER  (reads every 500ms)         |   |
|  |                                                      |   |
|  |  /sys/class/thermal  -->  CPU temperature            |   |
|  |  /sys/class/power_supply  -->  AC or battery         |   |
|  |  /proc/meminfo  -->  memory pressure                 |   |
|  |  /proc/[pid]/stat  -->  per-process CPU usage        |   |
|  |  /proc/[pid]/smaps  -->  page residency              |   |
|  |  /sys/devices/system/cpu/*/cpufreq  -->  freq        |   |
|  +---------------------------+--------------------------+   |
|                              | feature vectors              |
|                              v                              |
|  +----------------------------------------------------+   |
|  |           AI DECISION ENGINE                        |   |
|  |                                                      |   |
|  |  thermal_model  --+--  memory_model                  |   |
|  |                   |                                  |   |
|  |  security_model --+--  io_model                      |   |
|  |                                                      |   |
|  |  All models: online learning, < 1ms inference        |   |
|  +-----+-----------+-----------+-----------+-----------+   |
|        |           |           |           |               |
|        v           v           v           v               |
|  +----------+ +----------+ +----------+ +----------+      |
|  | MODULE 1 | | MODULE 2 | | MODULE 3 | | MODULE 4 |      |
|  | CPU      | | Memory   | | Security | | I/O      |      |
|  | Scheduler| | Manager  | | Monitor  | | Manager  |      |
|  +----+-----+ +----+-----+ +----+-----+ +----+-----+      |
|       |            |            |            |              |
+-------+------------+------------+------------+--------------+
        |            |            |            |
        v            v            v            v
  sched_setattr  madvise     ptrace/seccomp  ioprio_set
  setpriority    mlock       prctl/kill      io_uring
  getrusage      mincore     eBPF            netlink/tc
        |            |            |            |
+-------+------------+------------+------------+--------------+
|                    LINUX KERNEL 6.x                          |
|  CFS Scheduler | Memory Manager | Block Layer | Net Stack   |
|  cgroups v2    | eBPF           | VFS         | perf_events |
+---------------------------+---------------------------------+
                            |
+---------------------------v---------------------------------+
|                       HARDWARE                              |
|  AMD CPU (throttles at ~85C) | 30GB disk | NVMe SSD        |
|  Battery + AC               | Thermal sensors               |
+------------------------------------------------------------+
```

## Data Flow Per Module Cycle

```
MODULE 1 — CPU SCHEDULER (every 500ms)
  [read /proc/pid/stat] --> [extract burst features] --> [decision tree]
  --> [predicted_burst_ms] --> [sched_setattr or setpriority]
  --> [log prediction vs actual] --> [retrain model]

MODULE 2 — MEMORY MANAGER (every 500ms)
  [read /proc/pid/smaps + /proc/meminfo] --> [compute pressure]
  --> [Markov chain predicts next pages] --> [madvise WILLNEED or FREE]
  --> [if pressure > 0.88: compress to zram]

MODULE 3 — SECURITY MONITOR (per syscall via eBPF)
  [eBPF sys_enter tracepoint] --> [record pid + syscall_nr]
  --> [n-gram model computes anomaly score]
  --> [if score > threshold: apply seccomp filter]
  --> [if confirmed malicious: kill process]

MODULE 4 — I/O MANAGER (every 1 second)
  [read /proc/net + /proc/pid/io] --> [classify traffic type]
  --> [linear regression predicts I/O burst]
  --> [ioprio_set per process + power state]
```

## Power State Machine

```
                    AC plugged in
          +----------------------------+
          v                            |
  +--------------+    unplug    +---------------+
  |   AC MODE    | ----------> | BATTERY MODE  |
  | Performance  |             |   Efficiency  |
  +------+-------+ <---------- +------+--------+
         |          plug in           |
         v                            v
  Full prefetch ON           Prefetch disabled
  Compile max priority       Compile deferred
  I/O: throughput mode       I/O: power-save
  Thermal: warn at 82C      Thermal: warn at 75C
```
