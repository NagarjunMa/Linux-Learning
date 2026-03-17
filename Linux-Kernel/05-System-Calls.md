# System Calls: The Bridge Between App and Kernel

## What a System Call Is

A system call is the **ONLY legitimate way** for a user-space process to request kernel services.

```
Trading App: bytes_sent = send(sock_fd, order_buf, 128, 0);
  │
  1. App executes SYSCALL instruction (x86-64)
  2. CPU switches from Ring 3 → Ring 0
  3. CPU jumps to kernel's syscall handler
  4. Kernel runs sys_sendmsg() on app's behalf
  5. Kernel copies data, enqueues to NIC
  6. CPU switches back Ring 0 → Ring 3
  7. App resumes after send() call

Total cost: ~100–400ns per syscall
```

At 10M packets/sec: 400ns × 10M = **4 seconds/second of syscall overhead** → why DPDK avoids syscalls.

## Critical Syscalls for Trading Systems

| Syscall | What It Does |
|---------|-------------|
| `send()` / `recv()` | Core networking — every non-DPDK order/market data |
| `epoll_wait()` | Event loop — wait for I/O on many FDs. Scales to thousands |
| `mmap()` / `mlock()` | Memory mapping and pinning — critical for hugepages |
| `mlockall()` | Pin ALL pages to RAM — no swap, no page faults |
| `clock_gettime()` | High-resolution timestamps (uses vDSO — no mode switch) |
| `clone()` | Create threads (with `CLONE_VM`) |
| `futex()` | Fast mutex — kernel only called on contention |

## strace: See Every Syscall

```bash
# Count syscalls and time spent — best for profiling
strace -c -p <PID> -- sleep 5
# If send() = 80% of time → network syscall overhead
# If futex() = thousands → lock contention

# Trace specific syscalls
strace -e trace=network -p <PID>
strace -e trace=memory  -p <PID>

# Show time spent in each call (-T)
strace -T -p <PID>

# Watch a stuck syscall
# [pid 1234] epoll_wait(5, ...) ← blocked here for 230ms
```

## vDSO: Syscalls That Aren't Really Syscalls

The kernel maps some implementations directly into every process's user address space — **no mode switch at all**.

```bash
# clock_gettime() uses vDSO:  ~5ns instead of ~400ns
# Verify it's mapped:
cat /proc/<PID>/maps | grep vdso

# Always use:
clock_gettime(CLOCK_MONOTONIC, &ts)   # ✅ vDSO, never goes backwards
# NOT:
gettimeofday()                         # ❌ lower resolution
# NOT:
CLOCK_REALTIME                         # ❌ can jump due to NTP
```
