The USE method is a framework made by Brenden Gregg. It achieves a structured way of reading and identifying the issue in systems.

USE is an acronym of:
1. **Utilization**: how busy is the resource?
2. **Saturation**: is there more work than the resource can handle, causing queues or delays?
3. **Errors**: are there failures or error events?

A USE method checklist for Linux performance: https://www.brendangregg.com/USEmethod/use-linux.html

Brenden separated two domains of resources: physical and software resources.

The physical resources are:
- CPU
- Memory
- Disk I/O
- Network
- Storage Capacity

# CPU

For CPU utilization, you look for the percentage of CPU time spent doing work:

```bash
mpstat -P ALL 1
top
```

For CPU saturation, you look for the runnable tasks waiting for CPU time:

```sh
vmstat 1 # if r > cpu, then we have a saturated cpu
sar -q 1
```

For CPU errors, hardware faults might happening:

```bash
journalctl -k
dmesg
```

What to interpret:

> High CPU utilization does not necessarily indicate a bottleneck (70%, 80%). Sustained run-queue pressure suggests CPU saturation

# Memory

For memory utilization, you look for available RAM and usage:

```bash
free -m
sar -r 1
```

For memory saturation, you look for swapping, reclaim pressure, page allocation stalls:

```bash
vmstat 1 # si, si
sar -B 1
```

For memory error, out-of-memory events and memory faults:

```bash
journalctl -k
dmesg
```

What to interpret:

> Low free RAM does not necessarily mean memory contention because Linux uses unused RAM for caching and reclaim quickly if the system needs available memory.

# Disk I/O

For I/O utilization, you interpret device activity and throughput:

```bash
iostat -xz 1 # %util, r/s, w/s
```

For I/O saturation, you look for I/O latency and queue depth:

```bash
iostat -xz 1 # r_await, w_await, aqu-sz
```

For I/O errors, failures or device errors might happen:

```bash
journalctl -k
dmesg
```

What to interpret:

> High `%util` does not always mean disk contention, especially with SSDs, NVMe, and parallel storage. Look for increasing latency and queue depth under load.

# Network

For network utilization, you look for network throughput relative to link capacity:

```bash
sar -n DEV 1
ethtool
```

For network saturation, packet drops and interface queue pressure:

```bash
sar -n EDEV 1
tc -s qdisc show
```

For network errors, interface errors, CRC errors, dropped packets:

```bash
ip -s link
ethtool -S
```

What to interpret:

> Compare observed throughput against the interface's negotiated link speed. Packet drops may indicate congestion or buffer pressure, but can also have other causes.

# Storage Capacity

For storage capacity utilization, filesystem space and inode usage:

```bash
df -h
df -i
```

For saturation, little or no remaining space or available inode:

```bash
df -h
df -i
```

Errors indicates no-space and filesystem allocation failures:

```bash
journalctl -k
```

What to interpret:

> Storage capacity is different from disk I/O. A filesystem can have plenty of free capacity while the underlying storage is I/O saturated.


