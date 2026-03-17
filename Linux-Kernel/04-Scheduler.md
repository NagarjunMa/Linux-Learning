# The Scheduler: How the Kernel Decides Who Runs

## The Core Job

The scheduler runs thousands of times per second and answers one question: **"Right now, which task should each CPU be running?"**

For a machine with 32 CPUs and 1,000 runnable tasks, it must select 32 winners every few milliseconds.

## The Three Scheduling Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **SCHED_OTHER / SCHED_NORMAL** | CFS — gives every process a fair share. Uses "virtual runtime" | General workloads. **NEVER for latency-critical threads** |
| **SCHED_FIFO** | Real-time. Runs until it yields or is preempted by higher priority. No timeslices | Critical trading polling loop. `chrt -f 99 ./app` |
| **SCHED_RR** | Real-time with timeslices. Rotates between equal-priority threads | Multiple equal-priority RT threads sharing time |

## CFS — Completely Fair Scheduler (Default)

- Tracks "virtual runtime" (vruntime) per task — lowest vruntime runs next
- **nice values** (-20 to +19): lower nice = higher priority = more CPU share
- **Timer tick (HZ=1000):** fires every 1ms to check if preemption needed — costs ~1–5μs of jitter

```bash
# Set nice value
renice -n -10 -p <PID>

# Disable timer ticks on specific CPUs (eliminates 1ms jitter)
# Add to GRUB: nohz_full=4-7
```

## Real-Time Scheduling

```bash
# View current scheduling policy
chrt -p <PID>

# Set to SCHED_FIFO priority 99 (highest possible)
chrt -f 99 ./trading_app

# Change running process to real-time
chrt -f -p 99 <PID>

# Allow unlimited real-time CPU usage
sysctl -w kernel.sched_rt_runtime_us=-1

# WARNING: SCHED_FIFO priority-99 cannot be preempted — requires CPU isolation
```

## CPU Affinity: Pinning to Specific CPUs

When a thread migrates between CPUs, it loses its L1/L2 cache. First thousands of memory accesses after migration = cache misses (~50–200ns each). **Pinning eliminates migration.**

```bash
# View current CPU affinity
taskset -cp <PID>

# Pin process to CPUs 4 and 5 only
taskset -cp 4,5 <PID>

# Launch pinned to CPU 6
taskset -c 6 ./trading_app

# Launch with NUMA-aware memory binding
numactl --cpunodebind=0 --membind=0 ./trading_app
```

## CPU Isolation: Remove CPUs from the OS Scheduler

```bash
# Add to GRUB_CMDLINE_LINUX in /etc/default/grub:
isolcpus=4-7      # CPUs 4-7 removed from scheduler
nohz_full=4-7     # No timer ticks on these CPUs
rcu_nocbs=4-7     # No RCU callbacks on these CPUs

# Then: update-grub && reboot
# Then: taskset -c 4 ./trading_app
```

## Context Switches: The Hidden Latency Cost

A context switch saves current task registers and loads next task's registers (~1–10 microseconds, causes cache pollution).

```bash
# Measure context switch rate for a process
pidstat -w -p <PID> 1
# cswch/s  = voluntary (process chose to yield) — normal for I/O-bound
# nvcswch/s = nonvoluntary (kernel forced it) — bad for RT threads

# System-wide context switch rate
vmstat 1 | awk '{print $12}'  # cs column
```
