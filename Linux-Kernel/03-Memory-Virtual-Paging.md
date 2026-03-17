# Memory: Virtual Memory, Paging, and the TLB

## The Core Concept: Virtual Memory

Every process believes it owns the **entire 64-bit address space**. This is a lie — a beautiful, useful lie.

The kernel maintains a mapping table that translates each process's virtual addresses to actual physical RAM. Two processes can both use virtual address `0x400000`, but they map to completely different physical memory. This is **process isolation**.

## Pages: The Unit of Memory Management

The kernel divides RAM into fixed-size chunks called **pages**.

| Page Size | When to Use |
|-----------|------------|
| **4KB** (default) | Standard. Fine for most use cases |
| **2MB** hugepage | 512x fewer TLB entries for same memory. Used by DPDK, JVMs, databases |
| **1GB** hugepage | Very large contiguous allocations. Some HFT applications |

## The TLB: Why Hugepages Matter So Much

Every memory access requires a 4-level page table walk (4 memory reads). The **TLB (Translation Lookaside Buffer)** caches recent translations:
- **TLB HIT** → ~1 cycle
- **TLB MISS** → ~200ns page table walk

With 4KB pages: a 1GB buffer pool = 262,144 pages — TLB only has ~1,500 entries = **constant TLB misses**

With 2MB hugepages: the same 1GB pool = 512 pages = **fits in TLB** = near-zero misses

**This is why DPDK mandates hugepages.**

## Page Faults

When a process accesses memory not yet mapped to a physical frame, a page fault fires. The kernel allocates a real RAM page (costs ~1–10 microseconds).

**On trading servers:** prevent page faults at startup with:
```bash
mlockall(MCL_CURRENT | MCL_FUTURE)  # pin all pages to RAM
```

## VSZ vs RSS: Critical for Diagnosis

```bash
cat /proc/<PID>/status | grep Vm
# VmSize (VSZ)  = total virtual address space reserved — NOT real RAM
# VmRSS (RSS)   = actual physical RAM currently used — this is what matters
# VmPeak        = peak virtual memory usage
```

A process with 10GB VSZ and 500MB RSS is using 500MB of real RAM. The OOM killer uses RSS to decide who to kill.

## Memory Regions

| Region | How Kernel Manages It |
|--------|----------------------|
| **Stack** | ~8MB/thread, grows automatically on page fault |
| **Heap (malloc)** | glibc calls `brk()` or `mmap()` — demand paged until first write |
| **mmap (file)** | Mapped directly into virtual memory, read from disk on page fault |
| **Shared memory** | Multiple processes map the same physical pages |
| **Hugepages** | Must be pre-allocated by operator, drastically reduces TLB pressure |

## Configure Hugepages

```bash
# Check hugepage status
grep HugePages /proc/meminfo

# Allocate 1024 x 2MB hugepages
echo 1024 > /proc/sys/vm/nr_hugepages

# Disable Transparent Huge Pages (THP) — prevents compaction spikes
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```
