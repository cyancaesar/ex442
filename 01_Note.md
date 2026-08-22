The first hour covers `top` command and interpret its output and playing with `nice` and how to set it persistent using `systemd`.

Viewing on-line CPUs and turning them off.

Memory diagram overview which holds anonymous (application), kernel, buffer/cache and the use of swap.

## Throughput and Latency

- Some systems need high throughput and some need low latency
- Throughput: how much data can be processed by a resource
- Latency: how fast data can be processed by a resource
- Optimizing both can be impossible

## Unit of Measurement

- SI "System International" is the way how units are presented in a decimal way and used by hardware manufacturers:
	- Kilobyte = 1,000 bytes
	- Megabyte = 1,000,000 bytes
	- Gigabyte = 1,000,000,000 bytes
	- Terabyte = 1,000,000,000,000 bytes
- IEC "International Electrotechnical Commission" uses binary prefixes and found on most software programs:
	- Kibibytes = 1,024 bytes
	- Mebibytes = 1,048,576 bytes
	- Gibibytes = 1,073,741,824 bytes
	- Tebibytes = 1,099,511,627,776 bytes

How many Tebibytes is reported on a 20 Terabytes disk
```
20,000,000,000,000 bytes / 1024 bytes = 19,531,250,000 kibibytes
19,531,250,000 kibibytes / 10244 bytes = 19,073,486.328125 mebibytes
19,073,486 mebibytes / 1024 bytes = 18,626 gebibytes
18,626 gebibytes / 1024 bytes = 18.189 tebibytes
```

How many bytes per second can be transferred by a gigabit network card?
```
8 bits = 1 byte
and 1 Gbps = 1_000_000_000 bits
1_000_000_000 / 8 = 125_000_000 bytes/sec = 125 MB/sec
```


## Using Common Tools

Basic overview metrics to look for:
- Load average
- CPU utilization
- Memory
- Swap usage
- Most active processes

Overview of the Linux system stack and performance observability tools:
![[linux_observability_tools.png|550]]

Important tools to know how to use:
- `strace`
- `ss`
- `stap`
- `iotop`
- `top`
- `ps`
- `vmstat`
- `numastat`
- `perf`
- `sar`
- `/proc`

## Processes and Threads

- Processes are the default managed entities by the Linux kernel
- Processes can use threads to implement multitasking
- Kernel threads are managed as processes, and for each kernel routine one thread is started on each CPU
- User threads are not individually managed, but can be shown and are scheduled independently
- Linux kernel schedules tasks (not processes) and a task can be either a single-threaded process or a thread in a multi-threaded process
- `ps -C command -L` to see specific command threads
- `ps -efL` to see threads for all processes

Process stats:
- Runnable processes are processes that are being services or waiting to be services
- Blocking processes (showing as D) are in an uninterruptable sleep, which means they are waiting on some resources and cannot be interrupted by signals
	- I/O Wait
	- NFS or remote filesystems
	- Hardware issues
- Sleeping processes have been temporarily paused
- Zombie processes have not been cleaned up properly

A process can become a Zombie when it has completed its task, but the parent process hasn't collected its execution status, a disadvantage of a zombie is that is occupies a PID.

To remove the zombie, basically send a `SIGCHLD` signal to the parent to asks it to reap the zombie.
Or kill the parent process, the the zombie becomes an orphan, which will be picked by the init process.


## Sysstat packages

An optional package that contains a set of useful tools that analyze raw data collected by the kernel:
- `mpstat`: CPU statistics
- `iostat`: filesystem related statistics
- `pidstat`: statistics regarding PIDs
- `tapestat`: statistics regarding connected tape drives
- `sar`: the system activity reporter

The system activity reporter must be started in order to gather data:

```sh
dnf install -y sysstat
systemctl edit sysstat-collect.timer
# [Timer]
# OnCalendar=*:00/02
systemctl enable --now sysstat-collect.timer
```

Virtual memory usage with `vmstat`

Virtual memory is about memory addresses that can be allocated, and it is not related to physical memory.

```sh
vmstat 2 5
```

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0   6180 683400   6340 1733928    0    0    29     1   40   33  0  1 98  0  0
```

`si` and `so` are swap in and swap out.

## System Activity Report

It collects a lot of statistics across all the subsystems, make sure how to use it.

`sar` overview:
![[sar.png|550]]

`sar` commands:

```sh
sar -A # All statistics - equivalent to `-bBdFHSvwWy -I SUM -m ALL -n ALL -q ALL -r ALL -u ALL`
sar -A -f /var/log/sa/xxxx # Show all the data from a specific log file
sar -A 5 10 -o /tmp/sar.output # Gather statistics in an interval of 5 seconds for 10 times and write it to a file

sadf /tmp/sar.output -- -A # Another way of working with sar and important

sar -n DEV -f /tmp/sar.output # Show information related to network devices
sadf /tmp/sar.output -- -n DEV # Same as above but in csv format

sar -n DEV -f /tmp/sar.dat | grep '^[0-9].*ens160' | awk '{ print $1,$4 }' # to find the most packet received
```

## Performance Co-Pilot (PCP)

- Its a set of tools that allows for collection and analyzing data on local and remote systems
- The Performance Metrics Domain Agent (PMDA) gathers data on a system, the Performance Metric Collector Daemon (PMCD) controls PMDA
- Client applications connect to the PMCD to gather data
- `sysstat` package is needed to collect metrics


PCP setup:

```sh
dnf install -y pcp-gui pcp-system-tools sysstat
firewall-cmd --permanent --add-service=pmcd
firewall-cmd --reload
systemctl enable --now pmcd
systemctl enable --now sysstat-collect.timer
```

```sh
# Displays memory statistics
pcp free

# Displays disks statistics
pcp dstat
```

List of all possible parameters for `pcp`
```sh
pminfo
```

Viewing disk write statistics:
```sh
dd if=/dev/zero of=/home/sysperf/zero.bin &
pmval -s 10 disk.dev.write
```

Analyzing a PCP file:
```sh
pmval -a sysperf.lab/20260606.17.52.0 mem.freemem
```

`pmlogger` is a PCP archive logger which stores metric in a file, enabling it:
```sh
systemctl enable --now pmlogger

# Verify its state with pcp
pcp
# pmlogger: primary logger: /var/log/pcp/pmlogger/sysperf.lab/20260606.17.52
```

You can analyze with `pmval`, `pmchart` and `pmdumplog`

```sh
pmdumplog -L /var/log/pcp/pmlogger/sysperf.lab/20260606.17.52.0
pminfo | grep load # kernel.all.load

pmval -a /var/log/pcp/pmlogger/sysperf.lab/20260606.17.52.0 kernel.all.load
pmval -f 2 -a /var/log/pcp/pmlogger/sysperf.lab/20260606.17.52.0 kernel.all.load

# Find the highest one minute load average
pmval -f 2 -a /var/log/pcp/pmlogger/sysperf.lab/20260606.17.52.0 kernel.all.load | sort -k2n
```

## SystemTap

What is SystemTap?

- SystemTap analyzes events by adding kernel probes.
- SystemTap captures when a specific event happens, the kernel runs a handler provided by SystemTap.
- To run all SystemTap functionality, you need to install source code, kernel handlers, compilers and more.
- Cross-instrumentation can be used to generate a kernel module that can be executed in remote system that don't have full installation.
- While using SystemTap, during script compilation with `stap -p 4 -v -m` a kernel module is compiled, and inserted while running the script.

Two distinct method of using SystemTap:
1. Install everything and run the SystemTap script
2. Insert a SystemTap generated kernel module

Before using SystemTap, add and enable debug repositories:
- `dnf config-manager --enable rhel-9-for-$(uname -m)-baseos-debug-rpms`
- `dnf config-manager --enable rhel-9-for-$(uname -m)-appstream-debug-rpms`

Setting up SystemTap:
- `dnf install kernel-debuginfo-$(uname -r) kernel-debuginfo-common-$(uname -m)-$(uname -r) kernel-devel-$(uname -r) systemtap systemtap-runtime gcc elfutils`
- `stap-prep`
- Compile SystemTap script including the kernel module
	- `stap -p 4 -v -m syscalls_by_proc /usr/share/systemtap/examples/process/syscalls_by_proc.stp`
- After compilation, run it using
	- `staprun syscalls_by_proc.ko`
	- or `stap /usr/share/systemtap/examples/process/syscalls_by_proc.stp`


Running SystemTap as unprivileged user:
- User can run SystemTap modules if they are member of `stapusr` or `stapdev` group
- `stapusr` group members can only use `staprun` to run modules in `/lib/modules/$(uname -r)/systemtap`
- `stapdev` group members can run scripts with elevated privileges

## Hardware Resources

Show hardware resources of a system:
- `dmidecode` gives a lot of information about hardware resources, find the relevant using `grep`
- `lshw -short` gives an overview of hardware resources
- `lspci` also does the same

## Kernel Module

Altering kernel module is simple, view its parameters with `modinfo <module>`.

View `st` module:

```sh
modinfo st
# parm:           buffer_kbs:Default driver buffer size for fixed block mode (KB; 32) (int)
```

Modify `buffer_kbs` parameter:

```sh
vim /etc/modprobes.d/st.conf
# options buffer_kbs=32
```

Unload and load the module again:

```sh
modprobe -r st
modprobe st
```

## NUMA

- NUMA is a Non Uniform Memory Access
- It can be used to allocate specific memory slots to specific CPUs
- It is recommended to use memory slots close to a specific CPU socket (local memory) for lower latency
- Accessing other CPUs memory is called (foreign memory)
- `numactl` can be used to run a process on a specific memory slot
- To have services always on the same memory slot, include the NUMA configuration in the systemd unit file
- To analyze where a process currently is running, check `/proc/<PID>/status` and `/proc/<PID>/numa_maps`
- Or use `numastat -p <PID>`

NUMA-aware resource placement can be set persistently using systemd with `CPUAffinity=` option.

- To view CPU affinity of sshd service: `systemctl show --property CPUAffinity sshd.service`.
- To set CPU affinity value: `systemctl set-property <service name> CPUAffinity=<value>`

Setting up NUMA policy in a systemd unit file:

```ini
[Service]
NUMAPolicy=bind
NUMAMask=0
AllowedCPUs=0-15
AllowedMemoryNodes=0
```

Finally, three ways to look for NUMA:
- `numactl`
- `/proc/<pid>` subdirectory
- systemd unit file
## Optimizing Kernel Parameters

 `/proc/<pid>` is the way to know what the process is doing, all Linux commands like `ps`, `lscpu`, and tools alike, are inheritably use that directory to gets the needed information.

Memory information can be found in the file `/proc/meminfo`, and some values to look at:
- Active
- Inactive
- Active(anon)
- Inactive(anon)
- Active(file)
- Inactive(file)
- Dirty
- Buffers
- Cached
- VmallocTotal

`/proc/sysrq-trigger` is an administrative interface to send Magic SysRq commands directory to the kernel. It allows root user to perform low-level debugging, unfreezing a crashed system, or force a reboot when the OS is completely unresponsive.

To see the help menu:

```sh
echo h > /proc/sysrq-trigger
dmesg | tail
```

```log
[ 1853.101362] sysrq: HELP : loglevel(0-9) reboot(b) crash(c) terminate-all-tasks(e) memory-full-oom-kill(f) kill-all-tasks(i) thaw-filesystems(j) sak(k) show-backtrace-all-active-cpus(l) show-memory-usage(m) nice-all-RT-tasks(n) poweroff(o) show-registers(p) show-all-timers(q) unraw(r) sync(s) show-task-states(t) unmount(u) force-fb(v) show-blocked-tasks(w) dump-ftrace-buffer(z) 
```

Enable watchdog module and crash the system (experiment):

```sh
modprobe softdog
echo c > /proc/sysrq-trigger
```

`/proc/sys` directory is about kernel tunables and are organized in different categories:
- `abi`
- `crypto`
- `debug`
- `dev`
- `fs`
- `kernel`
- `net`
- `user`
- `vm`

The important tunables to work with are `kernel`, `net`, and `vm`.

`/proc/sys/vm/swappiness` controls how aggressively the system reclaims memory by balancing between moving inactive application data to swap space and dropping file-backed pages from the system cache. Its defaulted to `60` in most distributions.

Changing `swappiness` at the run-time: `echo 30 > /proc/sys/vm/swappiness`.

To view all the kernel tunables: `sysctl -a`.

Put it persistently:

```sh
vim /etc/sysctl.conf
# vm.swappiness = 30
```

`/proc/sys/net/ipv4/icmp_echo_ignore_all` to dictate the Linux kernel whether the system responds to ICMP Echo requests or not.

```sh
sysctl -w net.ipv4.icmp_echo_ignore_all=1
sysctl -w net.ipv6.icmp_echo_ignore_all=1
```

`/proc/sys/net/ipv6/conf/all/disable_ipv6` to turn off IPv6.

```sh
echo 1 > /proc/sys/net/ipv6/conf/all/disable_ipv6
```

You make it persistently by putting it again in `/etc/sysctl.conf`.

Install the kernel documentation for finding what values in a kernel parameter means like `/proc/sys/net/ipv4/tcp_rmem`:

```sh
dnf install -y kernel-doc
```

cd into `/usr/share/doc/kernel-doc-5.14.0-687.24.1/Documentation` and find `tcp_rmem`:

```sh
grep -iRl tcp_rmem *
# networking/ip-sysctl.rst
```

## TuneD

It allows to select a preset of performance profiles.

List profiles:
```sh
tuned-adm list
```

To see the profile settings, like `network-throughput`, view the configuration file:
```sh
cat /usr/lib/tuned/network-throughput/tuned.conf
```

You can create your profile in `/etc/tuned/<profile_name>/tuned.conf` and use `tuned-adm profile <profile_name>` to select it.

TuneD recommend:
```sh
tuned-adm recommend
```

Testing scenario:
- Set `vm.swappiness` using `sysctl` to 20
- Adjust the default TuneD profile to set `vm.swappiness` to 40
- Reboot and view what has changed
- Make a change to let the opposite option to set the `vm.swappiness`.

After setting `vm.swappiness = 20` in `sysctl.conf` and changing the default TuneD profile which is `throughput-performance` to set it to 40. The `sysctl.conf` takes the precedence.

`/etc/tuned/tuned-main.conf` has a flag for disabling any `sysctl` command, that is called `reapply_sysctl` which is defaulted to 1. Set it to 0, and then `sysctl` will not be reapplied.

## Cgroups

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

```sh
systemctl list-units -t slice
```

View CGroup slices:

```sh
systemd-cgtop
```

View CGroup ls:

```sh
systemd-cgls
```

Creating a stress test:

```sh
# Copy `stress1.service` and stress2.service to /etc/systemd/system
cp stress* /etc/systemd/system
systemctl daemon-reload

# Make sure the system is a one-cpu system
echo 0 > /sys/bus/cpu/devices/cpu1/online
echo 0 > /sys/bus/cpu/devices/cpu2/online
echo 0 > /sys/bus/cpu/devices/cpu3/online
echo 0 > /sys/bus/cpu/devices/cpu4/online
echo 0 > /sys/bus/cpu/devices/cpu5/online

# Verify it's only one
lscpu | grep "^CPU(s):" -A 2

# Start the stress services
systemctl start stress1
systemctl start stress2

# View the processes
top
```

After viewing the top command you can see the first `dd` process takes `~60%` of CPU cycles, and the other takes `~30%` CPU cycles. If we ran on the shell this long-running command: `while true; do true; done`, the `bash` process will takes `~50%` CPU cycles, and the other are now `30%` and `20%`.

This is because the services are running inside the `system.slice` and the bash is running inside the `user.slice`. And by default, both slices are equal.

![[top_cpu_weights.png|582]]

We are going to modify the `CPUWeight` of the `system` slice and set it to the double of the `user` slice `CPUWeight=200`.

```sh
systemctl set-property system.slice CPUWeight=200
```

View again the top output:

```
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

The key to different settings in CGroup:

```sh
man systemd.directives
# Find `CPUWeight=`
# It is defined in `systemd.resource-control`

man systemd.resource-control
```

Common properties for resource control:
- `CPUWeight`: specifies CPU time weight to the processes executed, ranges from 1 to 1000, defaults to 100.
- `CPUQuota`: specifies CPU time quota to the processes executed, takes percentage.
- `AllowedCPUs`: bind processes to be executed on specific CPUs, takes list of indices, or range.
- `AllowedMemoryNodes`: bind processes to be executed on a specific memory NUMA nodes.
- `MemoryMin`: specifies minimum memory usage protection of the executed processes.
- `MemoryMax`: specifies absolute maximum memory usage.

Demonstrate `MemoryMax`:

```sh
dnf install -y httpd
systemctl edit httpd.service

# [Service]
# MemoryMax=4M # Ridiculously low

systemctl start httpd.service
```

It won't start, view the journal log `journalctl -u httpd`:

```
Aug 17 01:25:26 sysperf.lab systemd[1]: Starting The Apache HTTP Server...
Aug 17 01:25:27 sysperf.lab systemd[1]: Started The Apache HTTP Server.
Aug 17 01:25:27 sysperf.lab httpd[47226]: Server configured, listening on: port 80
Aug 17 01:25:27 sysperf.lab systemd[1]: httpd.service: A process of this unit has been killed by the OOM killer.
Aug 17 01:25:27 sysperf.lab systemd[1]: httpd.service: Main process exited, code=killed, status=9/KILL
Aug 17 01:25:27 sysperf.lab systemd[1]: httpd.service: Killing process 47227 (httpd) with signal SIGKILL.
```

It was killed by the OOM killer.

-- Demo of creating a copy of `user` slice and modify the `CPUWeight` to 400, then running a service inside that slice by using `systemctl edit stress2.service`.

Run commands inside a slice:

```sh
systemd-run --slice=aziz.slice <command>
# systemd-cgls /aziz.slice/<command>
# otherwise, it will run it inside the default slice, system.slice

# Or edit the Service unit file and include:
# Slice=<slice>
```

For applying runtime resource limit:

```sh
systemctl set-property sshd.service MemoryAccounting=yes
systemctl set-property sshd.service MemoryLimit=2048M
cat /sys/fs/cgroup/memory/system.slice/sshd.service/memory.limit_in_bytes # old

# in cgroup v2
[root@sysperf ~] cat /sys/fs/cgroup/system.slice/cpu.weight
200
[root@sysperf ~] cat /sys/fs/cgroup/aziz.slice/cpu.weight
400
[root@sysperf ~] cat /sys/fs/cgroup/system.slice/stress1.service/cpu.weight
100
[root@sysperf ~] cat /sys/fs/cgroup/aziz.slice/stress2.service/cpu.weight
200
```

### Formula to Calculate the CPU Weight

Given two slices `system.slice` with `CPUWeight=200` and `aziz.slice` with `CPUWeight=400`:
```
* system.slice 200
* aziz.slice 400
```

Get the total of CPU weight `200 + 400 = 600`:
```
* system.slice 200 / 600 = 33.33%
* aziz.slice 400 / 600 = 66.66%
```

Process `stress1.service` is within `system.slice` and `stress2.service` within `aziz.slice`:
```
* system.slice 33.33%
  * stress1.service 33.33%
* aziz.slice 66.66%
  * stress2.service 66.66%
```

Whatever their definition of `CPUWeight` of `CPUShare` (deprecated), if there is no siblings and only one process, that process will allocate all the slice's CPU weight.

Now if `stress1.service` has `CPUWeight=100` and `stress3.service` has `CPUWeight=300` both are in `system.slice`:
```
* system.slice 33.33%
  * stress1.service 100
  * stress3.service 300
* aziz.slice 66.66%
  * stress2.service 66.66%
```

The total `CPUWeight` in `system.slice` is `100 + 300 = 400`. So, `stress1.service` would get `100 / 400 = 0.25 * 33.33 = ~8.3%`.
And `stress3.service` would get `300 / 400 = 0.75 * 33.33 = ~24.9%`
```
* system.slice 33.33%
  * stress1.service 8.3%
  * stress3.service 24.9%
* aziz.slice 66.66%
  * stress2.service 66.66%
```

```
Control Group                                                                         Tasks   %CPU   Memory  Input/s Output/s
/                                                                                       787   98.9     1.3G        -        -
aziz.slice                                                                                1   64.3   448.0K        -        -
aziz.slice/stress2.service                                                                1   64.3   204.0K        -        -
system.slice                                                                            282   32.2     1.5G        -        -
system.slice/stress3.service                                                              1   24.1   212.0K        -        -
system.slice/stress1.service                                                              1    8.0   216.0K        -        -
```