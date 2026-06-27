# Current Task

**Last updated:** 2026-06-27

---

## Where We Are

The project memory system is set up (agent_docs/ created). You now understand what TAORM
is, what you need to build, how to present it, and what topics to learn. The design docs
are complete. No code exists yet. The Arch laptop has 30GB free and an AMD processor.

## What To Do RIGHT NOW

**Phase 0: Environment Setup** — Install the tools TAORM needs on your Arch laptop.

This MUST be done on your actual Arch laptop, not on this Windows machine. Open a
terminal on your Arch laptop and run these commands one by one.

### Step 1: Update the system

```bash
sudo pacman -Syu
```

Type `Y` and press Enter if it asks to confirm.

### Step 2: Install build tools

```bash
sudo pacman -S base-devel linux-headers git cmake
```

### Step 3: Install kernel interface tools

```bash
sudo pacman -S bcc python-bcc libbpf libseccomp liburing iproute2
```

### Step 4: Install Python AI layer

```bash
sudo pacman -S python python-numpy python-scikit-learn python-psutil python-matplotlib
```

### Step 5: Install monitoring tools

```bash
sudo pacman -S lm_sensors powertop stress-ng procps-ng
```

### Step 6: Verify eBPF works

```bash
sudo python /usr/share/bcc/tools/execsnoop
```

Should print process executions in real time. Press Ctrl+C to stop.
If it fails, search "Arch Wiki eBPF" for troubleshooting.

### Step 7: Verify thermal sensors

```bash
sudo modprobe k10temp
sensors
```

Should show AMD core temperatures. If nothing shows, run `sudo sensors-detect`
and say yes to all prompts, then run `sensors` again.

### Step 8: Verify cgroups v2

```bash
cat /sys/fs/cgroup/cgroup.controllers
```

Expected: `cpuset cpu io memory hugetlb pids rdma misc`

### Step 9: Create the project directory structure

```bash
cd ~/Desktop
mkdir -p taorm/{src/ai,src/bpf,python,config,systemd,tests}
cd taorm
touch Makefile README.md
```

## Done Means

- All 9 steps completed without errors
- `sensors` shows CPU temperature
- `execsnoop` runs and prints output
- `cat /sys/fs/cgroup/cgroup.controllers` shows controllers

## After This

Come back here, report which steps succeeded and which failed.
Update changelog.md, then we move to Phase 1 (Sensor Layer + Daemon).
