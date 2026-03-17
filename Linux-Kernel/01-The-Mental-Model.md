# The Linux Kernel: The Mental Model

The kernel is the permanent middleman between your application and the physical hardware. Your trading app never touches the CPU directly, never reads from RAM directly, never sends a packet directly. Every single thing it does requires asking the kernel's permission.

## The Two Worlds: User Space and Kernel Space

```
┌──────────────────────────────────────────────────────────────────┐
│                          USER SPACE                              │
│   Trading App  │  Python Scripts  │  Salt Agent  │  Monitoring   │
│                                                                  │
│  • Can only access its OWN memory                                │
│  • Cannot touch hardware directly                                │
│  • Runs in CPU Ring 3 (unprivileged)                             │
│  • If it crashes, ONLY that process dies                         │
├──────────────────────────────────────────────────────────────────┤
│              SYSTEM CALL BOUNDARY (the only door)                │
├──────────────────────────────────────────────────────────────────┤
│                         KERNEL SPACE                             │
│   Scheduler  │  Memory Manager  │  Network Stack  │  Drivers     │
│                                                                  │
│  • Has access to ALL memory on the system                        │
│  • Controls all hardware via device drivers                      │
│  • Runs in CPU Ring 0 (privileged)                               │
│  • If it crashes, THE ENTIRE MACHINE goes down (kernel panic)    │
├──────────────────────────────────────────────────────────────────┤
│                        HARDWARE LAYER                            │
│      CPUs  │  RAM DIMMs  │  NIC  │  SSD/NVMe  │  PCIe Bus        │
└──────────────────────────────────────────────────────────────────┘
```

This separation is called **process isolation** and is enforced by the CPU itself — not just software.

## The Six Kernel Subsystems

| Subsystem | What It Does | Why It Matters |
|-----------|-------------|----------------|
| **Process Scheduler** | Decides which process/thread gets CPU time | #1 source of latency issues |
| **Memory Manager** | Allocates RAM, manages virtual address spaces, hugepages, swap | #1 source of OOM issues |
| **Network Stack** | Receives/sends packets, manages socket buffers, TCP state | #1 source of latency/packet loss |
| **VFS / Filesystem** | Abstracts storage behind a common interface | Source of disk I/O issues |
| **Device Drivers** | Kernel code controlling hardware (NIC, disk, GPU) | Source of hardware issues |
| **IPC** | Pipes, signals, sockets, shared memory, semaphores | How processes communicate |

## Interview Key Insight

When an interviewer describes a symptom — "latency spikes", "memory growing", "packet drops" — your **first job is to identify which subsystem is involved**. That narrows the tools you use and the questions you ask.
