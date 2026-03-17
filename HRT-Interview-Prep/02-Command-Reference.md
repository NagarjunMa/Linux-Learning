# Linux Command Reference: HRT Level

This is the complete command reference for HRT systems roles — not the beginner DigitalOcean list, but the commands that separate candidates.

## Process & CPU

```bash
ps aux --sort=-%cpu | head          # top CPU consumers right now
top -H -p <PID>                     # show threads of a specific process
cat /proc/<PID>/status              # state, VmRSS, threads, signals
cat /proc/<PID>/cmdline | tr '\0' ' '  # full command line of process
cat /proc/<PID>/wchan               # which kernel function it's blocked in
perf top -p <PID>                   # find the hot function inside a process
strace -cp <PID>                    # count syscalls — is it in kernel a lot?
taskset -cp <PID>                   # which CPUs is this process allowed on?
chrt -p <PID>                       # what scheduling policy is it using?
mpstat -P ALL 1                     # per-CPU breakdown (find uneven load)
```

## Memory

```bash
free -h                             # quick overview: total, used, free, cache
cat /proc/meminfo                   # detailed: MemFree, Buffers, Cached, HugePages
vmstat 1 5                          # si/so columns = swap in/out (bad if non-zero)
pmap -x <PID> | tail -1             # process total VSZ, RSS, dirty
cat /proc/<PID>/smaps_rollup        # accurate RSS + PSS for a process
dmesg | grep -i oom                 # OOM killer messages — what got killed?
slabtop                             # kernel slab cache usage
watch -n5 'cat /proc/<PID>/status | grep VmRSS'  # watch for memory leak
```

## Disk I/O

```bash
df -h && df -i                      # space AND inodes (both can be full!)
du -sh /var/* | sort -rh | head     # find biggest directories
iotop -o -d 1                       # which processes are doing I/O right now?
iostat -xz 1                        # per-device: %util, await, r/s, w/s
lsof +L1 | grep deleted             # deleted files still held open (wasted space)
find / -size +500M -type f          # find large files anywhere
```

## Networking

```bash
ss -tulnp                           # ALL listening sockets with process names
ss -s                               # socket summary: total, TCP states, UDP
ss -tin                             # TCP info per connection (RTT, retransmits)
ip -s link show eth0                # RX/TX bytes, errors, dropped packets
ethtool eth0                        # link speed, duplex, state
ethtool -S eth0 | grep drop         # NIC-level drops (ring buffer overflows)
ethtool -g eth0                     # ring buffer sizes
ethtool -c eth0                     # interrupt coalescing settings
tcpdump -i eth0 -nn -c 100         # quick capture, no DNS resolution
mtr <host>                          # better than traceroute — shows live packet loss per hop
netstat -s | grep -i retran         # TCP retransmit count
cat /proc/interrupts                # interrupt rate per CPU (IRQ storm?)
```

## System-Wide Triage

```bash
uptime                              # load averages: 1m/5m/15m vs CPU count
dmesg -T | tail -30                 # recent kernel messages with timestamps
journalctl -p err -b -n 50         # errors since last boot
systemctl list-units --failed      # services that crashed
lsof -u username | wc -l           # open file count for a user
watch -n1 'cat /proc/interrupts'   # interrupt rate per CPU live
sar -n DEV 1 5                     # historical network utilization
lscpu                               # CPU topology: cores, NUMA nodes, cache sizes
```

## Commands MISSING from the DigitalOcean List (Most Important)

| Command | Why It Matters |
|---------|---------------|
| `ss` | Modern replacement for `netstat`. `ss -tulnp` is the go-to |
| `perf top` | Real profiler. Finds the hot function inside a high-CPU process |
| `perf record/report` | Records CPU samples, generates call graphs |
| `perf stat` | Counts hardware events (cache misses, instructions, cycles) |
| `mpstat -P ALL` | Per-CPU breakdown — critical for multi-threaded apps |
| `iotop` | Shows which process is doing I/O right now |
| `lsof +L1` | Finds deleted files held open — the #1 phantom disk space cause |
| `taskset -cp` | Shows/sets CPU affinity |
| `chrt -p` | Shows/sets scheduling policy |
| `numactl` | NUMA-aware process launch |
| `numastat` | NUMA hit/miss ratio per process |
| `ethtool -S` | NIC hardware stats including drops |
| `cat /proc/<PID>/wchan` | Which kernel function a stuck process is waiting in |
| `truncate -s 0` | Clears file WITHOUT deleting — preserves open file handles |
| `watch -n1` | Runs any command repeatedly — essential for live monitoring |
| `mtr` | Better traceroute with live packet loss per hop |
