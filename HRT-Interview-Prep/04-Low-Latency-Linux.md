# Low-Latency Linux: SysTrade-Specific Q&A

These questions are unique to the SysTrade team and probe whether you understand **why latency matters** and how Linux choices affect it.

## Core Q&A

### Q: What kernel parameters would you tune to reduce latency on a trading server?

**A:**
- CPU governor to `performance` → prevents frequency scaling
- Disable C-states → CPU doesn't sleep between packets
- Disable THP → prevents compaction spikes
- Increase socket buffers → `net.core.rmem_max`, `wmem_max`
- `vm.swappiness=0` → never swap trading process pages
- `kernel.sched_rt_runtime_us=-1` → unlimited CPU for real-time processes

```bash
cpupower frequency-set -g performance
echo never > /sys/kernel/mm/transparent_hugepage/enabled
sysctl -w vm.swappiness=0
sysctl -w kernel.sched_rt_runtime_us=-1
# Add to GRUB: intel_idle.max_cstate=0
```

---

### Q: What is CPU isolation and why does it matter for trading?

**A:** `isolcpus` removes CPUs from the general kernel scheduler — the OS won't schedule **anything** on them unless explicitly told to. Combined with:
- `taskset` → pin trading process to isolated CPUs
- `nohz_full` → stop timer ticks (eliminates 1ms jitter)
- `SCHED_FIFO priority 99` → preempts everything

Even a 10-microsecond OS interruption can cause a missed trade.

```bash
# Add to GRUB_CMDLINE_LINUX:
isolcpus=4-7 nohz_full=4-7 rcu_nocbs=4-7
```

---

### Q: What is TCP_NODELAY and when would you use it?

**A:** Disables **Nagle's algorithm**. Nagle buffers small packets and waits up to 200ms to coalesce them before sending. For bulk transfers this improves efficiency. For trading — small order messages, microseconds matter — it's catastrophic.

`TCP_NODELAY` forces immediate send.

```c
setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, 1);
```
Or system-wide: `sysctl -w net.ipv4.tcp_low_latency=1`

---

### Q: What is kernel bypass networking and why would HRT use it?

**A:** In normal Linux: NIC → kernel network stack → socket buffer → user app. Each layer adds latency + context switches.

**Kernel bypass** eliminates the kernel from the data path:
- **DPDK** — application polls the NIC directly from user space
- **Solarflare OpenOnload** — same for Solarflare NICs

Result: **sub-microsecond latency** vs 10–50μs for kernel networking.

HRT uses this for most latency-sensitive market data feeds and order entry paths.

---

### Q: What is NUMA and how does it affect a trading application?

**A:** **Non-Uniform Memory Access**. In multi-socket servers, each CPU has its own memory controller and local memory.
- Local memory access: ~100ns
- Remote socket memory access: ~150–200ns

If your trading process runs on CPU socket 0 but its memory was allocated on socket 1, **every memory access is 50–100ns slower**.

```bash
# Pin process AND memory to same NUMA node
numactl --cpunodebind=0 --membind=0 ./trading_app

# Verify NUMA alignment
numastat -p <PID>
# Low NumaHit + high NumaMiss = running on wrong NUMA node
```

---

### Q: How would you safely roll out a config change to production trading servers?

**A:**
1. Never push to all servers at once during trading hours
2. Roll to one non-critical server first, monitor 10–15 minutes
3. Check: CPU, latency percentiles, error rates
4. Use Ansible/Salt with change in version control (rollback = one command)
5. Coordinate with trading team for critical servers — roll at market close
6. Keep rollback runbook written **before** starting
7. Have rollback command ready in terminal

---

## SSH Debugging Round: First Steps on Unknown Server

```bash
# === STEP 1: Orient ===
hostname && uname -r      # what machine? what kernel?
cat /etc/os-release       # OS and version
uptime                    # load averages?

# === STEP 2: Check for alarms ===
dmesg -T | tail -30       # recent kernel messages
journalctl -p err -n 50 -b  # errors since boot
systemctl list-units --failed  # failed services

# === STEP 3: Resource snapshot ===
top -b -n1 | head -20     # CPU, mem, top processes
free -h                    # memory
df -h && df -i             # disk space AND inodes
ss -s                      # socket summary

# === STEP 4: Recent changes ===
last -n 20                 # recent logins
find / -newer /proc/1 -maxdepth 4 -type f 2>/dev/null  # recently modified
journalctl --since '1 hour ago'  # what happened?
```

## Quick-Fire Speed Round

| Question | Answer |
|----------|--------|
| Process in 'D' state, kill -9 does nothing — what now? | Uninterruptible sleep (NFS or disk I/O). Check `cat /proc/PID/wchan`. If NFS: `umount -f`. Last resort: reboot |
| df -h shows 20% used but writes fail — first command? | `df -i` — inode exhaustion from millions of small files |
| Service was running, now it's not, no app logs — first 3 commands? | `systemctl status`, `journalctl -u service -n 50`, run binary manually as service user |
| Which process is listening on port 8042? | `ss -tulnp \| grep :8042` or `lsof -i :8042` |
| CPU 100% but no single process above 5%? | Multiple processes summing. `mpstat -P ALL 1`. High %si = IRQ storm |
| Network connection slow but ping is fine? | `mtr <host>` for live per-hop loss. `ss -tin dst <ip> \| grep rtt` for TCP RTT |
| Check NIC hardware drops? | `ethtool -S eth0 \| grep -i drop` — invisible to netstat/ss |
| Which kernel function is a stuck process waiting in? | `cat /proc/PID/wchan` |
| Monitor RSS for a leak? | `watch -n5 'cat /proc/PID/status \| grep VmRSS'` |
