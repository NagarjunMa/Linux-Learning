# Crisis Scenarios: 12 Live Investigation Walkthroughs

## The Crisis Decision Framework

Every scenario follows this 5-phase structure:

| Phase | What You Do |
|-------|------------|
| **ORIENT** | What server, what time, what's affected? `uptime`, `hostname`, `who` |
| **SCOPE** | How bad? One process, one server, or systemic? |
| **HYPOTHESIZE** | Form 2–3 theories BEFORE running commands |
| **ISOLATE** | Narrow from system-wide → process → function |
| **RESOLVE** | Fix with minimal blast radius. Communicate before acting |

### The Three Questions You Ask First
1. **Is trading impacted right now?** — If yes, restore service first, root cause later
2. **What changed recently?** — 80% of incidents are caused by recent changes
3. **Is this isolated or spreading?** — Check a second server before concluding

---

## Scenario 1: Phantom Disk Space — df Full, ls Shows Nothing

**Situation:** `/var` is 100% full. `du -sh /var/*` shows only 2GB used out of 50GB.

```bash
# Step 1: Confirm the discrepancy
df -h /var
du -sh /var/* | sort -rh | head -10
# df says 50GB used, du says 2GB → 48GB "phantom"

# Step 2: Find deleted-but-open files
lsof +L1
lsof +L1 | grep deleted
lsof +L1 | awk '{print $1, $2, $7, $9}' | sort -k3 -rn | head -20

# Step 3: Identify the offending process
lsof +L1 | grep deleted | awk '{print $2}' | sort -u
cat /proc/<PID>/cmdline | tr '\0' ' '

# Step 4: Free space WITHOUT restarting the process
ls -la /proc/<PID>/fd/ | grep deleted
truncate -s 0 /proc/<PID>/fd/<FD_NUMBER>
# This frees disk space immediately — process continues running
```

**HRT Angle:** Restarting a trading process to free disk space is a **business decision**, not a technical one. Your instinct to truncate via `/proc` without killing the process shows operational maturity.

---

## Scenario 2: Silent Network Killer — Packets Drop, No Errors in netstat

**Situation:** Market data latency spiking. Ping fine. `netstat` shows no errors. Packets drop silently.

```bash
# Step 1: Rule out app-level drops first
ss -s
ss -tulnp | grep <market_data_port>
# Large Recv-Q = packets arriving but app not reading fast enough

# Step 2: Check NIC-level drops — INVISIBLE to netstat
ethtool -S eth0 | grep -i 'miss\|drop\|error\|fifo'
ip -s link show eth0
watch -n1 'ethtool -S eth0 | grep drop'
# rx_missed_errors, rx_fifo_errors non-zero = NIC ring buffer overflow

# Step 3: Check socket buffer overflow
netstat -s | grep -i 'receive buffer\|overflow\|drop'
sysctl net.core.rmem_max

# Step 4: Confirm with packet capture
tcpdump -i eth0 -nn -c 1000 port <feed_port> -w /tmp/cap.pcap
```

**Fix:** `ethtool -G eth0 rx 4096` + `sysctl -w net.core.rmem_max=268435456`

**HRT Angle:** `netstat` shows ZERO but `ethtool -S` shows NIC-level drops. Only an experienced engineer thinks to check the NIC layer.

---

## Scenario 3: Rogue Thread — One Core at 100%, App Degraded

**Situation:** Trading app latency 10x worse. Process at 100% CPU but has 16 threads.

```bash
# Step 1: Confirm it's one thread, not the whole process
top -H -p <PID>
# If one thread shows 100% and others ~0% → hot thread problem

# Step 2: Is the 100% intentional?
taskset -cp <TID>    # is it pinned to an isolated CPU?
chrt -p <TID>        # SCHED_FIFO on isolated CPU = intentional polling

# Step 3: If NOT intentional — find what it's doing
perf top -p <PID>    # live hotspot (>90% one function = found it)
strace -p <TID> -c -e trace=all -- sleep 3  # syscall profile
# thousands of futex() calls = lock contention

# Step 4: Capture call stack
perf record -g -p <PID> -t <TID> -- sleep 5
perf report --stdio | head -40
```

**HRT Angle:** You DON'T immediately restart. First check if 100% CPU is intentional — an HFT polling thread IS supposed to peg its CPU.

---

## Scenario 4: Latency Spike Every 7 Seconds — OS Interference

**Situation:** Exactly 200–300μs latency spike every ~7 seconds. Normal is 20μs.

```bash
# Step 1: Check Transparent Huge Pages compaction
cat /sys/kernel/mm/transparent_hugepage/enabled  # 'always' = bad
dmesg -T | grep -i 'thp\|compact\|huge'          # compaction events?

# Step 2: Check timer tick configuration
cat /proc/cmdline | grep nohz     # nohz_full set?
cat /sys/devices/system/cpu/isolated  # isolated CPUs?

# Step 3: Check CPU frequency and C-states
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor  # should be 'performance'
cpupower idle-info    # C-states enabled?

# Fix:
echo never > /sys/kernel/mm/transparent_hugepage/enabled
cpupower frequency-set -g performance
# Add to GRUB: isolcpus=N nohz_full=N intel_idle.max_cstate=0
```

**HRT Angle:** The exact period is the clue. THP compaction runs periodically — 200–300μs stall matches exactly. Real scenario at HFT firms.

---

## Scenario 5: Zombie Storm — Hundreds of Zombies, Fork Bomb Aftermath

```bash
# Step 1: Quantify
ps aux | grep -c ' Z '
cat /proc/sys/kernel/pid_max

# Step 2: Find the parent (you must kill the PARENT — not the zombies)
ps -eo pid,ppid,stat,comm | grep ' Z '  # see PPID column
ps -eo pid,ppid,stat,comm | awk '$3~/Z/{print $2}' | sort | uniq -c

# Step 3: Kill the parent (zombies vanish automatically — systemd reaps them)
kill -15 <PARENT_PID>
sleep 5 && ps aux | grep -c ' Z '    # confirm they cleared
kill -9 <PARENT_PID>                  # if graceful fails
```

**Key insight:** `kill -9` on a zombie does NOTHING. You must kill the parent.

---

## Scenario 6: Memory Leak — RSS Growing 1GB/Hour, OOM in 5 Hours

```bash
# Step 1: Confirm growth rate
watch -n5 'cat /proc/<PID>/status | grep VmRSS'
# Growing steadily = real leak. One-time spike = normal allocation

# Step 2: Estimate time to OOM — make data-driven decision
free -h
# Rate: 1GB/hr × remaining hours = GB needed
# If MemAvailable > 10GB, you may have time to diagnose first

# Step 3: Capture forensics WITHOUT killing (for dev team)
gcore <PID>   # core dump without killing
pmap -x <PID> > /tmp/pmap_$(date +%s).txt

# Step 4: Protect trading process from OOM killer
echo -1000 > /proc/<PID>/oom_score_adj
```

**HRT Angle:** Calculate time to OOM, check headroom, make a data-driven decision. Capture `gcore` (not `kill`) to preserve evidence.

---

## Scenario 7: Kernel Panic — Server Rebooted Unexpectedly

```bash
# Step 1: Check PREVIOUS boot logs (not current!)
journalctl --list-boots
journalctl -b -1 --since '09:40' --until '09:48'  # previous boot

# Step 2: Find the panic message
dmesg -T | grep -i 'panic\|oops\|bug\|error\|fault'
journalctl -b -1 -p err
ls /var/crash/    # crash dumps

# Step 3: Check hardware errors (MCE)
journalctl | grep -i 'mce\|machine check'
dmesg -T | grep -i 'mce\|corrected\|uncorrected'
# Uncorrected memory error = bad RAM → hardware replacement required
```

**Key insight:** The crash evidence is in **`journalctl -b -1`** (previous boot), NOT the current boot.

---

## Scenario 8: Inode Trap — Free Space but Writes Fail

```bash
# Step 1: Check inodes (df -h won't show this)
df -i
df -ih
# IUse% = 100% = no new files possible

# Step 2: Find the inode hoarder directory
find /var -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head -20

# Step 3: Clean up safely
find /var/spool/offending_dir -type f -mtime +7 -delete
# Verify: df -i | grep /var
```

**Key insight:** `df -h shows space but writes fail` → always run `df -i` first.

---

## Scenario 9–12: Quick Reference

| Scenario | Key Diagnostic | Key Fix |
|----------|---------------|---------|
| **Thundering Herd** (48 workers after failover) | `cat /proc/<PID>/wchan` — same for all | Staggered restart with jitter |
| **NFS Hang** (D-state, unkillable) | `cat /proc/<PID>/wchan` shows `nfs_...` | `umount -f /mnt/nfs_path` |
| **Permission Wall** (app can't read config) | `ls -la /etc/config`, `systemctl show service \| grep User` | Fix ownership or `User=` in unit file |
| **Degraded Failover** (standby at 3x latency) | `cat /proc/cmdline` vs primary, `numastat -p <PID>` | Apply missing tuning: governor, NUMA, coalescing |

---

## The Spoken Answer Formula

| Phase | What to Say |
|-------|------------|
| **Orient first** | "The symptom is X. Before I touch anything, let me understand the blast radius." |
| **Hypothesis before commands** | "My first hypothesis is Y, so I'm going to check Z to confirm or rule it out." |
| **Narrate findings** | "This shows RSS growing 50MB every 2 minutes — consistent with a memory leak." |
| **Eliminate, don't guess** | "This rules out network — NIC shows no drops. Now focused on application layer." |
| **Communicate before acting** | "Before I restart this, I want to flag it's trading-critical. Should I coordinate first?" |
| **Propose, don't just fix** | "Immediate fix is X. Long-term solution is Y, requiring a config change and deploy." |
