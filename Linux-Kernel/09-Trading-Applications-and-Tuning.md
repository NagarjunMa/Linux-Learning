# Trading Applications: Putting It All Together

## The Critical Path Timeline

| Stage | Time | What's Happening |
|-------|------|-----------------|
| Market data arrives at NIC | 0ns | Wire arrival |
| NIC DMA writes to ring buffer | ~100ns | NIC processes frame |
| IRQ fires to CPU | ~200ns | Hardirq |
| Kernel softirq processes packet | ~500ns | NET_RX_SOFTIRQ |
| Delivered to socket buffer | ~1,000ns | Kernel processing |
| App's recv() returns | ~1,500ns | Mode switch + copy |
| App parses market data | ~2,000ns | Application logic |
| Trading decision | ~3,000ns | Strategy computation |
| send() called for order | ~3,500ns | Mode switch in |
| NIC transmits order | ~4,000ns | **Full round trip** |

Eliminating the interrupt via DPDK polling collapses stages 3–6 to ~100ns → sub-1μs total.

## The Complete Tuning Surface

| Tuning Action | Kernel Subsystem Affected |
|--------------|--------------------------|
| `isolcpus` in GRUB | Scheduler — removes CPUs from general scheduling |
| `nohz_full` in GRUB | Scheduler — disables timer ticks on isolated CPUs |
| `SCHED_FIFO` + `chrt` | Scheduler — trading threads preempt everything |
| `taskset` / `numactl` | Scheduler + Memory — pins to CPU and NUMA node |
| Hugepages | Memory — reduces TLB pressure |
| `mlockall()` | Memory — pins all pages to RAM |
| NIC ring buffer size | Network driver — prevents hardware-level drops |
| IRQ affinity | Interrupt system — correct CPUs handle NIC interrupts |
| Interrupt coalescing | NIC driver — controls interrupt rate vs latency |
| Socket buffer sizes | Network stack — prevents socket-level drops |
| `TCP_NODELAY` | Network stack — disables Nagle's algorithm |
| `none` I/O scheduler | Block layer — predictable storage latency |
| `swappiness=0` | Memory — prevents trading pages from being swapped |

## Diagnosing Issues by Symptom

### Latency Issues
| Symptom | Root Layer | Fix |
|---------|-----------|-----|
| Periodic spikes every N ms | Scheduler | `nohz_full`, disable THP, C-state=C0 |
| Spikes correlated with other processes | Scheduler | `isolcpus` + `SCHED_FIFO` |
| Spikes after network events | Network/Interrupt | Fixed coalescing, pin IRQs |
| Spike on first memory access | Memory | `mlockall()` at startup |
| Random spikes, no pattern | Memory | Hugepages (TLB miss), or JVM GC |

### Memory Issues
| Symptom | Root Layer | Fix |
|---------|-----------|-----|
| RSS growing continuously | Memory | Memory leak — check with `valgrind`, `gcore` + `gdb` |
| Process killed unexpectedly | Memory | OOM killer — check `dmesg \| grep oom` |
| Disk write fails, free space exists | Filesystem | Inode exhaustion — `df -i` |
| Processes in 'D' state, unkillable | VFS/Block | NFS hang or disk failure |

### Network Issues
| Symptom | Root Layer | Fix |
|---------|-----------|-----|
| Packet drops, no errors in netstat | Network driver | `ethtool -S eth0 \| grep drop` → `ethtool -G eth0 rx 4096` |
| Socket receive buffer overflow | Network stack | Increase `rmem_max`, fix app drain speed |
| TCP retransmits | Network stack | Check physical network path, MTU, NIC errors |
| Connection refused immediately | Firewall/Stack | `ss -tulnp`, `iptables -L -n -v` |
| Connection hangs (no response) | Firewall | `tcpdump` to verify, `mtr` to trace path |

## The Systems Engineer's Quick Reference

```bash
# SCHEDULER
mpstat -P ALL 1          # per-CPU breakdown
chrt -p <PID>            # scheduling policy
taskset -cp <PID>        # CPU affinity
cat /sys/devices/system/cpu/isolated  # are CPUs isolated?

# MEMORY
free -h                  # overview
cat /proc/<PID>/status | grep Vm  # process memory
pmap -x <PID>            # detailed memory map
watch -n5 'cat /proc/<PID>/status | grep VmRSS'  # leak detection
echo -1000 > /proc/<PID>/oom_score_adj  # protect from OOM killer

# NETWORK
ip -s link show eth0     # interface health
ss -tulnp                # listening sockets
ss -tin                  # TCP info per connection
ethtool -S eth0 | grep drop  # NIC-level drops
tcpdump -i eth0 -nn -c 100   # quick packet capture

# I/O
iostat -xz 1             # device performance
iotop -o -d 1            # per-process I/O
df -h && df -i           # space AND inodes
lsof +L1 | grep deleted  # deleted-but-open files (phantom space)
```
