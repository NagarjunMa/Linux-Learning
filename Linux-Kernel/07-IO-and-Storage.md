# I/O and Storage: From Disk to Application

## The Page Cache: The Kernel's Read Buffer

When you read a file, the kernel reads it from disk into the **Page Cache** — RAM managed by the kernel as a file cache. The next read returns from RAM instantly.

The page cache is NOT application memory. It's reclaimable — freed when RAM is needed.

```bash
# Interpreting free -h output
free -h
#              total   used   free  shared  buff/cache  available
# Mem:          125G    20G    5G    100M       100G       104G

# 'available' is what matters — it includes reclaimable cache
# A system with 5G free but 100G cache is HEALTHY
```

## I/O Path: From System Call to Disk

```
App calls write(fd, data, len)
    │
    ▼
VFS (Virtual Filesystem Switch) — abstracts ext4/xfs/btrfs/nfs
    │
    ├── Buffered I/O (default)        ├── Direct I/O (O_DIRECT)
    │   Writes to page cache           │   Writes directly to block device
    │   Returns fast                   │   Waits for disk completion
    │
    ▼ (background writeback)
Block Layer (I/O scheduler)
    │
    ▼
Device Driver (NVMe, SATA)
    │
    ▼
Physical Storage
```

## I/O Schedulers

| Scheduler | When to Use |
|-----------|------------|
| **none / noop** | SSDs and NVMe. No reordering — FIFO. **Best for trading servers** |
| **mq-deadline** | Assigns deadlines, prevents starvation. Good for SSDs |
| **CFQ** | Completely Fair — gives each process fair I/O share. **BAD for trading** |

```bash
# Check current scheduler
cat /sys/block/nvme0n1/queue/scheduler

# Set to none (best for NVMe)
echo none > /sys/block/nvme0n1/queue/scheduler

# Monitor I/O per process
iotop -o -d 1

# Monitor per-device performance
iostat -xz 1
# %util — how busy (100% = saturated)
# await — avg wait in ms (<1ms for NVMe)
```

## File Descriptors: The Kernel's Handle for Everything

In Linux, **everything is a file** — sockets, pipes, devices, epoll instances. An FD is an integer indexing into the kernel's per-process FD table.

```bash
# System-wide FD limit
cat /proc/sys/fs/file-max

# Per-process FD limit
ulimit -n

# Count open FDs for a process
ls /proc/<PID>/fd | wc -l

# See what each FD points to
ls -la /proc/<PID>/fd/
# 0 → /dev/pts/0         (stdin)
# 3 → socket:[123456]    (TCP connection)
# 4 → /var/log/app.log   (log file)

# Detect FD leaks
watch -n5 'ls /proc/<PID>/fd | wc -l'  # steadily growing = FD leak

# Raise FD limit in systemd unit file
[Service]
LimitNOFILE=1048576
```

## Key Disk Diagnostics

```bash
df -h          # disk space usage
df -i          # inode usage (can be 100% while space is free!)
du -sh /var/*  # find biggest directories
lsof +L1       # deleted files still held open (wasted space)
```
