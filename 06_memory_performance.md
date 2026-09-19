## Page Tables and TLB

Linux normally manages memory in 4 KiB pages on x86_64.

A process uses virtual memory, while RAM is physical memory. The CPU needs a translation:

$$
\text{Virtual address} \rightarrow \text{Physical address}
$$

Those mapping are stored in **pages tables**.

The **Translation Lookaside Buffer** (TLB) is a CPU cache for recently used virtual-to-physical address translations.

Instead of walking page tables every time, the mapping used:

$$
\text{Virtual address} \rightarrow \text{TLB} \rightarrow \text{Physical address}
$$

If the mapping is not in the TLB:

$$
\text{Virtual address} \rightarrow \text{TLB-miss} \rightarrow \text{Page-table walk} \rightarrow \text{Physical address}
$$

## Example

Huge pages are a solution for memory pages problem. Imagine a 4GB file, with the default memory page with a size of 4KB, how many pages?

$$
\frac{4 \times 1024 \times 1024\space KB}{4\space KB} = 1,048,576 \space pages
$$
For HugePage of 2MB:

$$
\frac{4 \times 1024\space MB}{2}=2,048\space huge \space pages
$$

## Huge Pages and HugeTLB

Huge pages are larger memory pages used to reduce the number of page-table entries required for large amounts of memory. On x86_64, the normal page size is typically **4 KiB**, while common HugeTLB page size is **2 MiB**.

Using larger pages means the CPU can cover more memory with each TLB entry, which can reduce TLB misses and page-table overhead.

**HugeTLB** is the Linux mechanism used to explicitly reserve and manage huge pages.

To check the configured huge-page size:

```bash
grep -i hugepagesize /proc/meminfo
```

```result
Hugepagesize:       2048 kB
```

### Reserving HugeTLB pages

Check the number of huge pages:

```bash
sysctl vm.nr_hugepages
```

```result
vm.nr_hugepages = 0
```

Reserve 100 huge pages:

```bash
sysctl -w vm.nr_hugepages=100
```

If each page is 2 MiB:

$$
100 \times 2 \ \text{MiB} = 200 \ \text{MiB}
$$
Approximately 200 MiB of RAM is reserved for the HugeTLB pool.

## HugeTLB and Transparent Huge Pages

HugeTLB:
- Explicitly configured
- Reserved memory pool
- Configured with `vm.nr_hugepages`

Transparent Huge Pages (THP):
- Automatically managed by the kernel
- No manually reserved fixed pool required
- Kernel tries to use huge pages when appropriate

Check if THP is enabled:

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

```result
[always] madvise never
```

To disable it (runtime):

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

Disabling it permanently by adding `transparent_hugepage=never` flag on GRUB:

```bash
vim /etc/default/grub
```

```result
GRUB_CMDLINE_LINUX="... transparent_hugepage=never"
```

```bash
grubby --update-kernel=ALL --args="transparent_hugepage=never"
```

## HugeTLB on NUMA

On NUMA systems, HugeTLB pages are allocated from specific NUMA nodes. The kernel may distribute them across eligible nodes, but allocation depends on the current memory policy and available contiguous memory.

Check per-node huge pages:

```bash
grep Huge /sys/devices/system/node/node*/meminfo
```

## Page Faults

Each process uses page tables to map virtual memory pages to physical memory pages.

A **page fault** occurs when a process accesses a virtual page that does not currently have a usable mapping.

There are two main types:
- **Minor page fault**: the page can be resolved without reading data from storage.
- **Major page fault**: resolving the page requires reading data from storage, such as swap or a file-backed mapping.

Monitor page faults for a process:

```bash
ps -o pid,minflt,majflt,comm -C firefox
```

Or measure them while running a command:

```bash
perf stat -e minor-faults,major-faults firefox
```

The counters are also available from:

```bash
/proc/<pid>/stat
```

Where the field 10 is the minor faults and field 12 is the major faults [The /proc Filesystem]([https://docs.kernel.org/filesystems/proc.html](https://docs.kernel.org/filesystems/proc.html#process-specific-subdirectories)).

## Swap

Swap provides additional virtual memory by allowing the kernel to move inactive anonymous memory pages from RAM to storage.

Swap can either be configured as either:
- a swap partition or device
- a swap file

Both are managed by the kernel as swap space.

View active swap:

```bash
swapon --show
cat /proc/swaps
```

View memory and swap usage:

```bash
free -h
```

A swap device avoids filesystem-level fragmentation, while a swap file is generally easier to create, resize, and manage.

Swap behavior is influenced by kernel tunables such as:

```bash
sysctl vm.swappiness
```

This controls how aggressively the kernel prefers reclaiming anonymous memory through swap versus reclaiming filesystem cache.

## Memory Usage 

Linux divides memory usage into several categories, including anonymous memory, filesystem page cache, and kernel-managed buffers.

### Anonymous Memory

Anonymous memory is memory allocated by processes that is not backed by a regular file.

Examples: 
- Process heap (`malloc()`) 
- Process stack 
- Anonymous `mmap()` allocations

Anonymous pages must be preserved when reclaimed. The kernel can move them to swap if necessary.

### Page Cache

The page cache stores filesystem data in RAM to reduce repeated disk I/O.

Normal file reads and writes use the page cache. Direct I/O, such as `O_DIRECT`, can bypass the page cache.

Check page-cache usage:

```bash
grep -E '^(Cached|Buffers|Active\(file\)|Inactive\(file\))' /proc/meminfo
```

### Buffers vs Page Cache

- **Buffers**: Kernel memory used for block-device-related buffers and metadata.
- **Cached**: Primarily filesystem page cache, excluding `SwapCached`.

`Buffers` and `Cached` are separate accounting categories in `/proc/meminfo`.

### Active and Inactive Memory

Linux tracks memory using active and inactive LRU lists to help determine which pages are suitable for reclaim.

The main categories are:

```text
Active(anon)
Inactive(anon)
Active(file)
Inactive(file)
```

- `Active(anon)`: Recently used anonymous memory.
- `Inactive(anon)`: Less recently used anonymous memory.
- `Active(file)`: Recently used file-backed memory.
- `Inactive(file)`: Less recently used file-backed memory.

Active and inactive describe reclaim tracking, not whether memory is currently being accessed.

### Memory Reclaim

When memory pressure occurs, the kernel can reclaim memory through different mechanisms:

- Clean file-backed pages can be discarded and loaded again from their backing file.
- Dirty file-backed pages may need writeback before reclaim.
- Anonymous pages may be written to swap to preserve their contents.

The kernel uses reclaim algorithms and memory pressure information to determine what to reclaim.

## Swap and Swappiness

View current swappiness:

```bash
sysctl vm.swappiness
```

The tunable controls the relative I/O cost of swapping anonymous memory versus filesystem paging.

```text
Lower value: Swapping is considered relatively expensive. The kernel favors retaining anonymous memory.

Higher value: Swapping is considered relatively cheaper. The kernel may swap anonymous memory more readily.
```

The supported range is 0–200, and the default is commonly 60.

The value does not directly specify how frequently pages are swapped.

## Memory Monitoring

Useful commands:

```bash
free -h
vmstat 1
cat /proc/meminfo
sysctl vm.swappiness
swapon --show
```

Monitor:
- Available memory (`MemAvailable`)
- Anonymous memory
- Filesystem cache
- Swap usage
- Swap-in / swap-out activity
- Memory pressure and reclaim behavior

## Memory Lab


Create a very large file:

```bash
dd if=/dev/zero of=/large.bin bs=1M count=5120 status=progress
```

Write any dirty pages to filesystem and flush the cache:

```bash
sync
echo 1 > /proc/sys/vm/drop_caches
```

View the memory values first:

```bash
grep -E "MemFree|MemAvailable|^Cached" /proc/meminfo
```

```result
MemFree:         2620152 kB
MemAvailable:    2567580 kB
Cached:           100520 kB
```

Perform a disk I/O:

```bash
cat /large.bin > /dev/null
```

View again the memory values:

```bash
grep -E "MemFree|MemAvailable|^Cached" /proc/meminfo
```

```result
MemFree:          124064 kB
MemAvailable:    2573896 kB
Cached:          2635424 kB
```

Flush again and verify that the cache has been flushed:

```bash
sync
echo 1 > /proc/sys/vm/drop_caches
grep -E "MemFree|MemAvailable|^Cached" /proc/meminfo
```

```result
MemFree:         2745540 kB
MemAvailable:    2675876 kB
Cached:            64572 kB
```


Another test to see how page cache accelerate frequent read:

```bash
sync
drop 1 > /proc/sys/vm/drop_caches
grep -E "MemFree|MemAvailable|^Cached" /proc/meminfo
```

```result
MemFree:         2736812 kB
MemAvailable:    2666700 kB
Cached:            63656 kB
```

Read the large file:

```bash
perf stat dd if=/large.bin of=/dev/null bs=1M count=5120 status=progress
```

```result
2.160154232 seconds time elapsed
```

Verify the file is cached:

```bash
grep -E "MemFree|MemAvailable|^Cached" /proc/meminfo
```

```result
MemFree:          154324 kB
MemAvailable:    2622764 kB
Cached:          2655044 kB
```

Read again and verify the time to read has been reduced:

```bash
perf stat dd if=/large.bin of=/dev/null bs=1M count=5120 status=progress
```

```result
1.543587703 seconds time elapsed
```

View swap usage:

```sh
vmstat -w 2 50
firefox &
```

```
--procs-- -----------------------memory---------------------- ---swap-- -----io---- -system-- --------cpu--------
   r    b         swpd         free         buff        cache   si   so    bi    bo   in   cs  us  sy  id  wa  st
   1    0       590692       882016            0       166492    6    7   169   163   16   20   0   0 100   0   0
   0    0       590692       882016            0       166600    2    0     2     0  108  161   0   0 100   0   0
   0    0       590692       881512            0       166600  138    0   138     0  763  651   0   1  99   0   0
   3    0       587364       216628         2108       640696 3980    0 236048    52 8456 7435   8  10  80   2   0
   2    0       589156       131632            0       468920   56 2562 54488  3066 20612 12317  20  20  60   0   0
   0    0       608324       131968            0       525064   68 24562 83338 25122 10170 8171   8  10  81   2   0
   0    0       608324       109252            0       541996  370    0  8710   596  724  790   0   1  99   0   0
   0    0       643296       174696            0       547088 10086 18716 18168 18953 5637 5401   1   4  94   2   0
   0    0       643296       177720            0       546872   22    0    48    38  340  475   0   0 100   0   0
   0    0       642784       177448            0       547524  398    0   726     0  629  760   0   0  99   0   0
   0    0       642272       180932            0       547524  236    0   252     2  486  590   0   0 100   0   0
```

It is not bad to see swap in and swap out, but it is often bad to see it quite a lot. The two parameters show the movement between the memory and disk.
- swap in (reading): coming in to RAM from disk
- swap out (writing): pages going out from RAM to disk
- `so` and `bo` it's almost the same and that's bad if its happening frequently.

```
               total        used        free      shared  buff/cache   available
Mem:            3622        2809         445           0         130         812
Swap:          10151         583        9568
```
### Overcommitting Memory

- The Linux kernel uses overcommitting to ensure each process can request the amount of virtual memory it needs
- This is based on the principle that RAM requirements never match virtual memory requirements
- Overcommitting is essential for workloads like virtual machines; a VM that requests 4 GiB doesn't normally use all of it
- Overcommitting may lead to a situation where a machine runs out of memory
	- `vm.overcommit_memory` tunable parameter to manage overcommit settings
	- `0` overcommit in an intelligent way
	- `1` always allow
	- `2` never allow

Setting never mode for overcommit:

```sh
echo 2 > /etc/sysctl.d/98-overcommit
sysctl -p /etc/sysctl.d/98-overcommit

grep CommitLimit /proc/meminfo
```

$$
\text{CommitLimit} = (\text{RAM} - \text{HugeTLB RAM})\times \frac{overcommit\_ratio}{100}+Swap
$$
### OOM

- When out of memory, the kernel raises an OOM and a process is killed
- `vm.panic_on_oom` raises kernel panic
- Processes with a higher `/proc/<pid>/oom_score` are more likely to get killed
- Tune `/proc/<pid>/oom_score_adj`, 1000 is more likely, -1000 make a process immune from OOM killer
- `/proc/<pid>/oom_adj` is deprecated
- Systemd can set this parameter:
	- `[System] OOMAdjust=-18`

Experiment with OOM:

```sh
pgrep firewalld
# 1048

echo 200 > /proc/1048/oom_score_adj
echo f > /proc/sysrq-trigger

systemctl status firewalld.service
```

```
kernel: oom-kill:constraint=CONSTRAINT_NONE,nodemask=(null),cpuset=/,mems_allowed=0,global_oom,task_memcg=/system.slice/firewalld.service,task=firewalld,pid=1048,uid=0

kernel: Out of memory: Killed process 1048 (firewalld) total-vm:354068kB, anon-rss:25628kB, file-rss:19268kB, shmem-rss:0kB, UID:0 pgtables:192kB oom_score_adj:200
```

The firewall daemon has been killed by the OOM killer.

Indicators of memory issues:
- Swap usage
- Low available memory
- Low buffers and cache
- OOM messages in logs
- Slow disk/memory performance
- Memory leaks: `valgrind`

Solutions for memory issues:
- Add more RAM
- Use swap *efficiently*
- Use Cgroups
- Consider the use of huge pages
- Tune disk and/or network

## NUMA

**NUMA (Non-Uniform Memory Access)** is a memory architecture in which CPUs are grouped into NUMA nodes, with each node having memory that is physically closer to its CPUs. Accessing local memory is usually faster and has lower latency than accessing memory attached to another NUMA node.

NUMA-aware resource placement can improve performance by keeping a process close to the memory it uses. The `numactl` command can be used to control CPU and memory placement for a process, while `numastat -p <PID>` shows NUMA-related memory usage for a specific process.

NUMA information for a running process can also be inspected through:

```bash
/proc/<PID>/status
/proc/<PID>/numa_maps
```

For services managed by systemd, CPU and NUMA placement can be configured persistently. CPU affinity can be viewed with:

```bash
systemctl show --property CPUAffinity sshd.service
```

and configured with:

```bash
systemctl set-property <service> CPUAffinity=<value>
```

NUMA policy can also be defined directly in a systemd unit:

```ini
[Service]
NUMAPolicy=bind
NUMAMask=0
AllowedCPUs=0-15
AllowedMemoryNodes=0
```

In practice, NUMA configuration can be inspected or controlled in three main ways: with `numactl`, through `/proc/<PID>`, or through systemd service settings.

## Valgrind

Valgrind is a known tool for identifying a memory leak within an application.

```sh
dnf install -y valgrind
```

Valgrind tools:

```bash
valgrind --help
```

```result
usage: valgrind [options] prog-and-args

  tool-selection option, with default in [ ]:
    --tool=<name>             use the Valgrind tool named <name> [memcheck]
                              available tools are:
                              memcheck cachegrind callgrind helgrind drd
                              massif dhat lackey none exp-bbv
```

Cachegrind:

```bash
valgrind --tool=cachegrind ls
```

It outputs how much cache hit did the command do.

```bash
# To know the CPU cache size
dmidecode | grep L1
```

