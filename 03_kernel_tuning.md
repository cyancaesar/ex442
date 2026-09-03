## Kernel Modules

Linux kernel modules can be inspected and configured without rebuilding the kernel. The `modinfo <module>` command displays information about a module, including its available parameters.

For example, to inspect the `st` module:

```bash
modinfo st
```

```result
parm: buffer_kbs:Default driver buffer size for fixed block mode (KB; 32) (int)
```

Module parameters can be configured persistently through files under `/etc/modprobe.d/`. For example:

```sh
vim /etc/modprobe.d/st.conf
```

```result
options st buffer_kbs=32
```

After changing a module parameter, the module normally needs to be unloaded and loaded again:

```bash
modprobe -r st
modprobe st
```

Some modules cannot be unloaded while they are actively in use.

## Kernel Parameters and `/proc`

The `/proc` virtual filesystem exposes information about the running kernel and processes. Many Linux utilities, including `ps` and other monitoring tools, obtain system information from `/proc` or related kernel interfaces.

Information about a specific process can be found under:

```text
/proc/<PID>/
```

System-wide memory statistics are available in:

```text
/proc/meminfo
```

Useful fields include:

- `Active`
    
- `Inactive`
    
- `Active(anon)`
    
- `Inactive(anon)`
    
- `Active(file)`
    
- `Inactive(file)`
    
- `Dirty`
    
- `Buffers`
    
- `Cached`
    
- `VmallocTotal`
    

### Magic SysRq

`/proc/sysrq-trigger` is an administrative interface for sending **Magic SysRq** commands directly to the kernel. It is primarily intended for low-level troubleshooting and emergency recovery, such as inspecting blocked tasks, synchronizing filesystems, or triggering a crash for debugging.

To display the available SysRq commands:

```bash
echo h > /proc/sysrq-trigger
dmesg | tail
```

For example, deliberately triggering a kernel crash for crash-dump testing can be done with:

```bash
echo c > /proc/sysrq-trigger
```

This should only be performed on a disposable test system configured for crash-dump analysis, because it intentionally crashes the kernel.

### Kernel Tunables

Runtime kernel parameters are exposed under `/proc/sys/`. They are grouped into categories such as:

- `abi`
- `crypto`
- `debug`
- `dev`
- `fs`
- `kernel`
- `net`
- `user`
- `vm`

For performance tuning, the `kernel`, `net`, and `vm` categories are particularly important.

The `sysctl` command provides a convenient interface for viewing and modifying these parameters:

```bash
sysctl -a
```

A kernel tunable can be changed temporarily with:

```bash
sysctl -w <parameter>=<value>
```

Changes made this way last until the next reboot.

### Swappiness

`vm.swappiness` influences how aggressively the kernel prefers reclaiming anonymous memory through swap relative to reclaiming file-backed cache.

Its current value can be viewed with:

```bash
sysctl vm.swappiness
```

or:

```bash
cat /proc/sys/vm/swappiness
```

A temporary value can be set with:

```bash
sysctl -w vm.swappiness=30
```

For persistent configuration:

```bash
vim /etc/sysctl.conf
```

```resylt
vm.swappiness = 30
```

The configuration can then be applied with:

```bash
sysctl -p
```

### Network Kernel Parameters

Kernel networking behavior is also configurable through `sysctl`.

For example, IPv4 ICMP Echo replies can be disabled with:

```bash
sysctl -w net.ipv4.icmp_echo_ignore_all=1
```

To disable IPv6 at runtime:

```bash
sysctl -w net.ipv6.conf.all.disable_ipv6=1
```

Persistent values can be placed in `/etc/sysctl.conf` or, preferably on many modern systems, in a dedicated file under:

```
/etc/sysctl.d/
```

### Kernel Parameter Documentation

Kernel documentation can be useful when interpreting tunables such as `net.ipv4.tcp_rmem`.

On RHEL-based systems, install the kernel documentation package:

```bash
dnf install -y kernel-doc
```

Then search the installed documentation:

```bash
grep -iRl tcp_rmem /usr/share/doc/kernel-doc-*/Documentation
```

A parameter such as `tcp_rmem` is documented in the networking kernel documentation, commonly under:

```
networking/ip-sysctl.rst
```

## TuneD

**TuneD** is a system tuning service that applies predefined or custom performance profiles. These profiles can adjust kernel parameters, CPU behavior, storage settings, networking options, and other system-level performance characteristics.

To list available profiles:

```sh
tuned-adm list
```

To see which profile TuneD recommends for the current system:

```sh
tuned-adm recommend
```

A profile can be activated with:

```sh
tuned-adm profile <profile_name>
```

For example, the configuration of the `network-throughput` profile can be inspected with:

```sh
cat /usr/lib/tuned/network-throughput/tuned.conf
```

Custom profiles can be created under:

```text
/etc/tuned/<profile_name>/tuned.conf
```

and then activated with:

```sh
tuned-adm profile <profile_name>
```

### TuneD and sysctl

TuneD profiles can modify kernel parameters that are also configurable through `sysctl`. For example, `vm.swappiness` may be set manually in `/etc/sysctl.conf` while a TuneD profile defines a different value.

A useful test is:

1. Set `vm.swappiness` to `20` in `/etc/sysctl.conf`.
    
2. Configure the active TuneD profile to set `vm.swappiness` to `40`.
    
3. Reboot the system.
    
4. Check the resulting value with:
    

```sh
sysctl vm.swappiness
```

The final value depends on the order in which the settings are applied and on TuneD's configuration, so this behavior should be verified rather than treated as a fixed precedence rule.

TuneD's main configuration file is:

```text
/etc/tuned/tuned-main.conf
```

The `reapply_sysctl` option controls whether TuneD reapplies system `sysctl` settings after applying its own profile settings. When enabled, system `sysctl` configuration may overwrite values previously set by the TuneD profile. Setting it to `0` disables that reapplication behavior.

TuneD is therefore useful when performance tuning should be managed as a consistent profile rather than as a collection of individually maintained kernel parameters.