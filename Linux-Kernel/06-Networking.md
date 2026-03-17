# Networking: From Wire to Application

## The Complete Packet Journey (Standard Linux)

```
[WIRE]
  │ DMA
  ▼
NIC Hardware receives bits, writes sk_buff to kernel ring buffer
  │ INTERRUPT
  ▼
IRQ handler runs (hardirq) — acknowledges, schedules NAPI poll
  │ SOFTIRQ
  ▼
NET_RX_SOFTIRQ: NAPI polls RX ring, runs protocol processing
  │ Protocol Stack
  ▼
Ethernet → IP (checksum, TTL) → TCP (sequence, ACKs) → find socket
  │
  ▼
sk_buff copied into socket's rcvbuf (kernel memory)
  │ SYSCALL
  ▼
App calls recv() → kernel copies to user-space buffer

Total latency: 10–50 microseconds
Memory copies: 2
```

## Socket Buffers: The Most Misunderstood Part

Every socket has a **receive buffer (rcvbuf)** and **send buffer (sndbuf)**. If your app is too slow to drain the rcvbuf, it fills up — and the kernel **DROPS incoming packets silently**.

```bash
# Check/increase buffer sizes
sysctl net.core.rmem_max                     # max receive buffer
sysctl -w net.core.rmem_max=268435456        # set to 256MB
sysctl -w net.ipv4.tcp_rmem='4096 87380 268435456'

# Diagnose buffer exhaustion
ss -tn | awk '{print $3}'                    # Recv-Q > 0 = backed up
netstat -s | grep -i 'receive buffer errors' # drop counter
```

## NIC RX Ring Buffer: Drops Before the Kernel

The NIC has its own hardware ring buffer. If the kernel is too slow, packets are dropped here — **invisible to netstat/ss**.

```bash
# Check NIC ring buffer sizes
ethtool -g eth0

# Increase RX ring buffer to maximum
ethtool -G eth0 rx 4096

# Check for NIC-level drops (invisible to netstat!)
ethtool -S eth0 | grep -iE 'miss|drop|error|fifo|discard'
watch -n1 'ethtool -S eth0 | grep drop'
```

## Interrupt Coalescing: Latency vs CPU Trade-off

The NIC can wait for a batch of packets before firing one interrupt. Good for throughput, bad for latency.

```bash
# View current settings
ethtool -c eth0

# Lowest latency (one interrupt per packet — high CPU)
ethtool -C eth0 rx-usecs 0 rx-frames 1 adaptive-rx off

# Balanced
ethtool -C eth0 rx-usecs 50 adaptive-rx off

# Adaptive coalescing is BAD for trading — unpredictable variance
```

## IRQ Affinity: Which CPU Handles NIC Interrupts

```bash
# See which CPU handles NIC IRQs
grep eth0 /proc/interrupts
cat /proc/interrupts

# Bind NIC interrupt to specific CPU
echo 8 > /proc/irq/30/smp_affinity  # CPU 3 (bitmask: 1000 = 8)

# Disable irqbalance (it moves IRQs unpredictably = jitter)
systemctl stop irqbalance
systemctl disable irqbalance
```

## TCP State Diagnosis

```bash
# Connection state counts
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
# Lots of CLOSE_WAIT = app not closing connections
# Lots of TIME_WAIT = high connection churn (normal)

# Per-connection TCP stats (RTT, retransmits)
ss -ti
ss -ti dst :443
```
