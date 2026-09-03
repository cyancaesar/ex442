
- Cgroups places resources in controllers that represent the type of resource, like CPU, memory and blkio.
- By default, processes get an equal amount of resources
- CGroup are used to limit the availability of resources using limits and weights
- CGroup currently are integrated with systemd
- CGroup works with top level element called slice
- systemd uses 3 slices to identify resource types
	- system for system services and daemons
	- machine for virtual machines and containers
	- user for user sessions
- By default, each slice has access to an equal amount of resources


> [!NOTE] Another way describing Cgroup
> A resource limitation and management to load balance the availability of CPU cycles, memory, networking, and block I/O (blkio). 

List slices:

```bash
systemctl list-units -t slice
```

View cgroups slices:

```bash
systemd-cgtop
```

List cgroups slices:

```sh
systemd-cgls
```

Creating stress test and viewing it's CPU usage:

```bash
cp stress1.service /etc/systemd/system
cp stress2.service /etc/systemd/system

# Make sure the system is a one-cpu system
echo 0 > /sys/bus/cpu/devices/cpu1/online
echo 0 > /sys/bus/cpu/devices/cpu2/online
echo 0 > /sys/bus/cpu/devices/cpu3/online
echo 0 > /sys/bus/cpu/devices/cpu4/online
echo 0 > /sys/bus/cpu/devices/cpu5/online
```

Start the stress services:

```bash
systemctl start stress1
systemctl start stress2
```

After viewing the top command you can see the first dd process takes ~60% of CPU cycles, and the second one takes ~30% CPU cycles. If we ran in the shell this long-running command:
```bash
while true; do true; done
```

The bash process will takes ~50% CPU cycles, and the second is 30% and 20%.

This is because the services are running inside the `system.slice` and the bash is running inside the `user.slice`. By default, both slices are equal.

![[top_cpu_weights.png|582]]

Modify the `CPUWeight` of the `system.slice` and set it to the double of the `user.slice`'s.

```bash
systemctl set-property system.slice CPUWeight=200
```

View again the top output:

```result
top - 00:44:51 up  1:30,  4 users,  load average: 3.02, 2.86, 1.85
Tasks: 353 total,   4 running, 349 sleeping,   0 stopped,   0 zombie
%Cpu(s): 47.5 us, 50.8 sy,  0.0 ni,  0.0 id,  0.0 wa,  1.6 hi,  0.0 si,  0.0 st
MiB Mem :   3622.3 total,   1316.1 free,   1326.8 used,   1241.3 buff/cache
MiB Swap:   4008.0 total,   4008.0 free,      0.0 used.   2295.5 avail Mem

PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
8135 root      20   0  221000   1968   1852 R  43.4   0.1   6:24.61 dd
8143 root      20   0  224108   5684   3696 R  32.6   0.2   5:21.26 bash
8132 root      20   0  221000   1968   1852 R  21.7   0.1   3:15.05 dd
```


> [!info] Renice a process
> If you reniced, say PID `8135`, to -10, which is expected to make this process go aggressively, but actually it is not, and will stay.
> This is because nice works in the context of the slices.
> 
> So. Don't use nice, when CGroup exists. Don't be nice.

Manual pages for looking up cgroups parameters:

```bash
man systemd.directives
man systemd.resource-control
```

Common properties for resource control:
- `CPUWeight`: specifies CPU time weight to the processes executed, ranges from 1 to 1000, defaults to 100.
- `CPUQuota`: specifies CPU time quota to the processes executed, takes percentage.
- `AllowedCPUs`: bind processes to be executed on specific CPUs, takes list of indices, or range.
- `AllowedMemoryNodes`: bind processes to be executed on a specific memory NUMA nodes.
- `MemoryMin`: specifies minimum memory usage protection of the executed processes.
- `MemoryMax`: specifies absolute maximum memory usage.

Demonstrate `MemoryMax` parameter with httpd server:

```bash
dnf install -y httpd
systemctl edit httpd.service
# [Service]
# MemoryMax=4M
systemctl start httpd.service
```

The httpd will fail to start, notice the messages by the journal which tells you it was killed by the Linux OOM killer:

```log
Aug 17 01:25:26 sysperf.lab systemd[1]: Starting The Apache HTTP Server...
Aug 17 01:25:27 sysperf.lab systemd[1]: Started The Apache HTTP Server.
Aug 17 01:25:27 sysperf.lab httpd[47226]: Server configured, listening on: port 80
Aug 17 01:25:27 sysperf.lab systemd[1]: httpd.service: A process of this unit has been killed by the OOM killer.
Aug 17 01:25:27 sysperf.lab systemd[1]: httpd.service: Main process exited, code=killed, status=9/KILL
Aug 17 01:25:27 sysperf.lab systemd[1]: httpd.service: Killing process 47227 (httpd) with signal SIGKILL.
```

## Custom Slice

Creating a slice named console:

```bash
vim /etc/systemd/system/console.slice
```

```ini
[Unit]
Description=Console Services

[Slice]
CPUWeight=200
MemoryMax=4G
MemoryHigh=3G
IOWeight=200
```

Creating two stress services:

```bash
vim /etc/systemd/system/stress01.service
```

```ini
[Unit]
Description=Create I/O Stress

[Service]
Type=simple
ExecStart=/usr/bin/dd if=/dev/zero of=/dev/null
CPUWeight=100
```

```bash
vim /etc/systemd/system/stress02.service
```

```ini
[Unit]
Description=Create Twice I/O Stress

[Service]
Type=simple
ExecStart=/usr/bin/dd if=/dev/zero of=/dev/null
CPUWeight=200
Slice=console.slice
```


> [!NOTE] Running Inside a Slice
> To run a command inside as slice
> ```bash
> systemd-run --slice=console.slice COMMAND
> ```
> 
> To make sure a service runs inside a slice, edit the service unit `Slice` parameter:
> ```bash
> Slice=SLICE
> ```

## Runtime Resource Limitation

To apply runtime resource limit, use systemctl:

```bash
systemctl set-property sshd.service MemoryAccounting=yes
systemctl set-property sshd.service MemoryLimit=2048M
```

To view the resource limits of a cgroups v2:

```bash
cat /sys/fs/cgroup/system.slice/cpu.weight
cat /sys/fs/cgroup/console.slice/cpu.weight
```

For current running services:

```bash
cat /sys/fs/cgroup/system.slice/stress01.service/cpu.weight
```

## Calculate CPU Weight

Two slices:
- system.slice `CPUWeight=200` with `stress01.service`
- console.slice `CPUWeight=400` with `stress02.service`

Get the total slices CPU weight: $200 + 400 = 600$. Divide the total CPU weight by the service's slice weight:
- `stress01.service`: $200 / 600 = 0.333 = 33.3\%$
- `stress02.service`: $400 / 600 = 0.666 = 66.6\%$

Introduce `stress03.service` with a `CPUWeight=300` in system.slice:
- The total CPU weight of process in system.slice: 100 +300 = 400
- `stress01.service`: $100 / 400 = 0.25 \times 33.3=8.3\%$
- `stress03.service`: $300 / 400 = 0.75 \times 33.3=24.9\%$
