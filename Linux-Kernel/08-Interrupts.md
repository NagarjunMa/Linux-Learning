# Interrupts: How Hardware Talks to the Kernel

## Why Interrupts Matter for Trading

Hardware cannot politely wait. When a NIC receives a packet, it forces the CPU to stop whatever it's doing and run the interrupt handler.

For a latency-critical trading thread, an interrupt **steals 1–10μs**.

```
Without isolation:
[order logic] [tick interrupt] [order logic] [NIC interrupt] [order logic]
                ↑ 1-5μs stolen   ↑ 1-5μs stolen

With isolcpus + nohz_full on CPU 4:
[order logic] [order logic] [order logic] [order logic]
No interruptions — CPU 4 dedicated to trading
```

## Two Levels: Hardirq vs Softirq

| Type | Behavior |
|------|----------|
| **Hardirq (top half)** | Runs immediately when interrupt fires. Interrupts disabled — nothing preempts it. Must be as short as possible. Just acknowledges hardware and schedules softirq |
| **Softirq (bottom half)** | Deferred work. Handles heavy lifting: `NET_RX_SOFTIRQ` (packet processing), `NET_TX_SOFTIRQ` (send queue), `BLOCK_SOFTIRQ` (I/O completion) |

```bash
# Monitor hardware interrupt rate per CPU
watch -n1 'cat /proc/interrupts'

# Monitor softirq activity
watch -n1 'cat /proc/softirqs'
# NET_RX growing = NIC receiving intensively
# SCHED growing = many context switches

# Identify which IRQ belongs to which NIC
grep eth0 /proc/interrupts
# 30: 0  0  50000  0  PCI-MSI 524288-edge  eth0-rx-0
# IRQ 30 handled 50000 times on CPU 2
```

## IRQ Affinity: Critical Rule

**The IRQ-handling CPU must be on the SAME NUMA node as the NIC and the application.**

```bash
# Route IRQ 30 to CPU 3 (bitmask 0b1000 = 8)
echo 8 > /proc/irq/30/smp_affinity

# Multi-queue NICs distribute packets across RX queues
# Each queue can be pinned to its own CPU for parallel processing
ethtool -l eth0        # show number of RX queues
ethtool -L eth0 combined 8  # set 8 combined queues
```

## irqbalance: Disable on Trading Servers

```bash
# irqbalance auto-moves IRQs between CPUs — creates jitter
systemctl stop irqbalance
systemctl disable irqbalance
```

## The Full Anti-Latency Setup

```bash
# GRUB parameters (in /etc/default/grub):
# GRUB_CMDLINE_LINUX="isolcpus=4-7 nohz_full=4-7 rcu_nocbs=4-7"
# isolcpus  = remove CPUs 4-7 from OS scheduler
# nohz_full = disable timer ticks on CPUs 4-7
# rcu_nocbs = no RCU callbacks on CPUs 4-7

# After reboot:
taskset -c 4 ./trading_app    # pin trading app to isolated CPU
chrt -f 99 ./trading_app      # SCHED_FIFO priority 99
```
