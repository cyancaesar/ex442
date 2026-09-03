
## System Activity Tools

The **sysstat** package is an optional Linux package that provides a collection of useful tools for analyzing system performance data collected by the kernel. It includes `mpstat` for monitoring CPU statistics, `iostat` for reporting CPU and filesystem-related I/O statistics, `pidstat` for displaying statistics about individual processes and PIDs, `tapestat` for reporting activity on connected tape drives, and `sar` (System Activity Reporter) for collecting and displaying a wide range of historical and real-time system activity information.

The system activity reporter must be started in order to gather data:

```bash
dnf install -y sysstat
```

```bash
systemctl edit sysstat-collect.timer
```

```result
[Timer]
OnCalendar=*:00/02
```

```bash
systemctl enable --now sysstat-collect.timer
```

## sar

The **system activity reporter** is a tool that is within **sysstat** package that collects wide range of statistics across Linux subsystems.

![[sar.png|550]]

Show all statistics:

```bash
sar -A
```

```result
Linux 6.12.0-211.7.3.el10_2.x86_64 (linperf.local)      09/02/2026      _x86_64_        (2 CPU)

10:43:17 PM  LINUX RESTART      (2 CPU)

10:43:29 PM     CPU      %usr     %nice      %sys   %iowait    %steal      %irq     %soft    %guest    %gnice     %idle
10:44:01 PM     all      0.16      0.00      0.22      0.00      0.00      0.19      0.09      0.00      0.00     99.34
10:44:01 PM       0      0.16      0.00      0.22      0.00      0.00      0.19      0.06      0.00      0.00     99.37
10:44:01 PM       1      0.16      0.00      0.22      0.00      0.00      0.19      0.13      0.00      0.00     99.31
```

Show all the data gathered from a specific log file:

```bash
sar -A -f /var/log/sa/sa02
```

Collect all statistics 5 times with an interval of 2 seconds:

```bash
sar -A 2 5 -o /tmp/sar.log
```

system activity data formatter (sadf):

```bash
sadf /tmp/sar.log -- -A
```

Show information related to `ens160` network interface device from `/tmp/sar.log`:

```bash
sar -n DEV --iface=ens160 2 5 -f /tmp/sar.log
```

The same as above using sadf:

```bash
sadf /tmp/sar.log -- -n DEV --iface=ens160
```

## PCP

Performance Co-Pilot (PCP) is a set of tools for collecting and analyzing statistic data on local and remote systems and It is part of **sysstat** package. The **Performance Metric Domain Agent** (PMDA) gathers data on a a system. And **Performance Metric Collector Daemon** (PMCD) controls the agents.

Setting up PMCD:

```bash
dnf install -y sysstat pcp-gui pcp-system-tools
firewall-cmd --add-service=pmcd
firewall-cmd --add-service=pmcd --permanent
systemctl enable --now pmcd
systemctl enable --now sysstat-collect.timer
```

PCP commands:

```bash
pcp free
pcp dstat
pcp htop
pcp iostat
pcp mpstat
```

pminfo shows all possible performance metrics with their description, use it in conjunction with pmval:

```bash
pminfo # prints all performance metrics provided by PCP
pminfo | grep kernel.all.load
```

pmval to monitor the metric:

```bash
dd if=/dev/zero of=/junk &
pmval -s 5 disk.dev.write
```

```result
metric:    disk.dev.write
host:      linperf.local
semantics: cumulative counter (converting to rate)
units:     count (converting to count / sec)
samples:   5
interval:  1.00 sec

           nvme0n1
             224.8
             220.8
             227.7
             193.8
             210.8
```

Parse a file (required pmlogger to run at least one):

```bash
systemctl enable --now pmlogger
cd /var/log/pcp/pmlogger/linperf.local
pmval -a 20260902.23.15.0 mem.freemem -s 5
```

Find the highest kernel load average:

```bash
pmval -f 2 -a 20260902.23.15.0 kernel.all.load | sort -k2n
```

```result
23:16:37.703         0.18          0.09          0.02
```

## SystemTap

**SystemTap** is a Linux tracing and performance-analysis framework that observes system activity by placing probes at specific points in the kernel or user-space programs. When a probed event occurs, SystemTap executes a handler defined in the tracing script, allowing detailed inspection of system behavior.

SystemTap scripts are normally compiled into a kernel module before execution. A script can be compiled explicitly with:

```bash
stap -p 4 -v -m syscalls_by_proc /usr/share/systemtap/examples/process/syscalls_by_proc.stp
```

This produces a kernel module such as `syscalls_by_proc.ko`, which can then be executed with:

```bash
staprun syscalls_by_proc.ko
```

Alternatively, SystemTap can compile, load, and run the script directly:

```bash
stap /usr/share/systemtap/examples/process/syscalls_by_proc.stp
```

There are therefore two common ways to use SystemTap:
1. Install the full SystemTap development environment and run scripts directly.
2. Compile a SystemTap kernel module on a prepared system and run that module on another compatible system using `staprun`.

The second method is useful for remote systems where installing compilers, kernel debugging information, and the complete SystemTap toolchain is undesirable.

### Setup

SystemTap requires kernel development and debugging packages that match the running kernel. On RHEL 9, the appropriate debug repositories can first be enabled:

```bash
dnf config-manager --enable rhel-10-for-$(uname -m)-baseos-debug-rpms
dnf config-manager --enable rhel-10-for-$(uname -m)-appstream-debug-rpms
```

Install the required kernel packages:

```bash
dnf install -y kernel-debuginfo-$(uname -r) kernel-debuginfo-common-$(uname -m)-$(uname -r) kernel-devel-$(uname -r) gcc elfutils
```

Install SystemTap:

```bash
dnf install -y systemtap systemtap-runtime
```

The `stap-prep` command can then be used to prepare the system and install any additional dependencies required for the current kernel.

### Unprivileged User

SystemTap provides groups for delegating limited tracing capabilities to non-root users. Members of the `stapusr` group can run previously compiled SystemTap modules with `staprun`, typically from:

```
/lib/modules/$(uname -r)/systemtap
```

Members of the `stapdev` group have broader privileges and can compile and run SystemTap scripts that require elevated access.

In short, SystemTap provides **programmable dynamic tracing** of Linux systems: probes observe events, scripts define what should happen when those events occur, and the resulting instrumentation is compiled and executed as a kernel module.