# Process and Threads: How Code Runs on a CPU

## What a Process Actually Is

A process is the kernel's **unit of resource ownership**. It owns:
- A virtual address space
- Open file descriptors
- A set of threads
- Signal handlers
- User/group credentials

A process does **NOT** own a CPU. The CPU is a shared resource — the scheduler decides when your process gets to use it.

## Process States

```
fork()
  │
  ▼
┌─────────┐  scheduled  ┌──────────┐
│RUNNABLE │ ──────────► │ RUNNING  │
│  (R)    │ ◄────────── │  (R)     │
└─────────┘  preempted  └──────────┘
     │                       │
 I/O wait              syscall wait
     │                       │
     ▼                       ▼
┌─────────┐           ┌──────────┐
│INTERRUP-│           │UNINTRPT. │
│ TIBLE   │           │  SLEEP   │
│SLEEP (S)│           │   (D)    │
└─────────┘           └──────────┘
```

| State | Meaning |
|-------|---------|
| **R** — Running/Runnable | On CPU now, or ready and waiting for a CPU slot. High R count = CPU saturation |
| **S** — Interruptible Sleep | Waiting for I/O, timer, network. Normal state for most processes |
| **D** — Uninterruptible Sleep | Waiting for hardware I/O. **Cannot be killed. kill -9 does nothing.** |
| **Z** — Zombie | Process exited, parent hasn't called `wait()`. Indicates bug in parent |
| **T** — Stopped | Paused by SIGSTOP or debugger |

## Threads vs Processes: The Linux Reality

In Linux, there is **no fundamental difference** at the kernel level. Both are `task_struct` — the kernel's schedulable entity.

- `fork()` → new process, new virtual address space (copy-on-write). Expensive.
- `clone()` with `CLONE_VM` → new thread, **shares** virtual address space. Much cheaper. This is what `pthread_create()` calls.

**Practical difference:** Threads in the same process share memory — a crash in one can corrupt all threads. Processes are isolated.

## The task_struct: What the Kernel Knows

```c
struct task_struct {
    pid_t pid;              // ps PID column
    char  comm[16];         // ps COMMAND column (process name)
    int   state;            // ps STAT column (R, S, D, Z, T)
    struct mm_struct *mm;   // /proc/PID/maps (virtual memory)
    struct files_struct *files; // /proc/PID/fd/ (open FDs)
    unsigned int policy;    // SCHED_NORMAL, SCHED_FIFO, SCHED_RR
    int static_prio;        // nice value (-20 to +19)
    cpumask_t cpus_allowed; // taskset -cp PID (CPU affinity)
};
```

## Useful Commands

```bash
# View process states
ps aux | grep ' D '          # uninterruptible sleep — unkillable
ps -eLf | grep <PID>         # show all threads of a process

# Inspect task internals
cat /proc/<PID>/status        # state, VmRSS, threads, signals
cat /proc/<PID>/wchan         # which kernel function it's waiting in
cat /proc/<PID>/cmdline | tr '\0' ' '  # full command line
```
