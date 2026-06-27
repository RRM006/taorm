# TAORM — Project Constitution

These principles do not change. They guide every decision we make.

---

## 1. User Space Only

We never write kernel modules. TAORM runs as a user-space daemon and interacts
with the kernel through standard Linux syscalls, /proc, /sys, cgroups v2, and eBPF.
This keeps the project safe, debuggable, and portable.

## 2. Real Syscalls, No Simulation

Every module must use actual Linux system calls — sched_setattr, madvise, ptrace,
seccomp, ioprio_set, io_uring. We do not simulate kernel interfaces. If a syscall
is not available on our target kernel, we document it and find an alternative.

## 3. Lightweight Models Only

All AI models must meet these constraints:
- Inference latency: < 1 millisecond
- Total memory usage: < 50 MB across all models
- No deep learning (LSTM, transformers, RNN, CNN) on the laptop
- Allowed: decision trees, Markov chains, n-gram models, linear regression,
  Naive Bayes, one-class SVM

The reason: our machine has limited resources. Heavy models would consume the
very resources we are trying to manage.

## 4. Graceful Fallback

When the AI model is uncertain (confidence below threshold), we fall back to the
kernel's default policy. TAORM must never make the system worse than stock Linux.

## 5. Thermal + Power Awareness

Every resource decision considers:
- CPU temperature (from /sys/class/thermal)
- Power source (AC vs battery from /sys/class/power_supply)

This is what makes TAORM unique compared to generic OS resource managers.

## 6. Arch Linux Native

We integrate properly with Arch:
- systemd service unit
- pacman hook for kernel upgrades
- Config in /etc/taorm/
- Logs in journald
- Packages from official repos (no snap, no flatpak)

## 7. Measurable Outcomes

Every module must have a quantifiable metric. "Works well" is not enough.
We measure page fault rates, CPU temperatures, detection latency, power draw,
and compare against a baseline with TAORM disabled.

## 8. Test on Real Hardware

Benchmarks run on our actual Arch laptop, not just in a VM. Thermal behavior,
battery life, and OOM pressure are hardware-specific and must be tested on the
real machine.
