## Memory Performance Tuning

Things to know about memory:
- Virtual vs. Resident Memory
- Swap vs. Cache
- Huge pages
- Memory overcommitting

Huge pages are a solution for memory pages problem. Imagine a 4GB file, with the default memory page with a size of 4KB, how many pages?

$$
\frac{4 \times 1024 \times 1024\space KB}{4\space KB} = 1,048,576 \space pages
$$
For HugePage of 2MB:

$$
\frac{4 \times 1024\space MB}{2}=2,048\space huge \space pages
$$

HugePages introduces two types:
- Transparent HugePages
- HugePages (2MB memory pages)

So what HugePages enhance is what it is called **Translation Lookaside Buffer** (TLB). It is a very fast cache inside the CPU that stores the mapping of `Virtual Address --> Physical Address`.
It makes the CPU to look for an address in the TLB, if it doesn't exist (called TLB-miss), then the CPU would walk through all the page tables to figure it out.

Memory overcommitting is related to virtual and resident memory, memory overcommitting means a process could ask for more memory allocation than what's available.

Memory in Linux:
- Linux addresses memory in pages, with a standard size on x86_64 of 4Kib, to confirm use `getconf -a | grep PAGE`
- Processes do not address memory directly, each process gets a virtual address space
- When process requests memory, the kernel maps that physical address of a page to a virtual address in the process address space (TLB)
- It promotes isolation among processes, so each gets its own memory space
- The total amount of virtual memory that can be addresses is determined by the CPU address sizes `/proc/cpuinfo`
- The total amount of virtual memory is seen in `/proc/meminfo`

```sh
watch -n 1 cat /proc/meminfo # in pane 1
dd if=/dev/zero of=/mnt/xfs/boot.bin bs=1M count=5120 # in pane 2

# notice the *Dirty* and *Writeback* section
```

The resident memory (RSS) is a pointer to the physical memory address.

- VmallocTotal: the total amount of kernel virtual address space available for memory allocations made through the Linux `vmalloc` mechanism.
- It is not physical RAM. It represents virtual address space the kernel can use to map memory that is virtually contiguous even if the underlying physical pages are scattered.

Page Faults:
- Each process has a page table that tracks mappings of virtual pages to physical pages
- When a process accesses a page that has not yet been mapped to a physical memory page, this is a **minor page fault**
- A major page fault occurs when the memory page needs to be fetched from disk (swap)

To monitor page faults:

```sh
ps -o pid,minflt,majflt,comm -C firefox
perf stat -e minor-faults,major-faults firefox
```

The information is obtained from `/proc` directory, `/proc/<pid>/stats` fields: 10 and 12.

```sh
perf stat -e minor-faults,major-faults bash
id
ls
systemctl start httpd
systemctl stop httpd
exit
```

```
 Performance counter stats for 'bash':

            18,011      minor-faults
                 3      major-faults

      22.435913590 seconds time elapsed

       0.062857000 seconds user
       0.096931000 seconds sys
```

Swap file and swap device:
- Swap device is equal to a swap file
- But swap device is better than a fragmented swap file (fragmentation happens at the file system level)

## Page Tables and TLB

- CPU has Translation Look-aside Buffer which is a cache of address translation mapping from VIRT to RES.
- Huge Tables can be used (default size of 2 GiB) to enhance the TLB address space.
- On NUMA systems, the kernel spreads huge pages across all NUMA nodes

View TLB size:

```sh
cat /proc/cpuinfo | grep "TLB"
# TLB size        : 2560 4K pages
```

HugePages:

```sh
sysctl -a | grep huge
echo 200 > /proc/sys/vm/nr_hugepages

# Check the default HugePage size
cat /proc/meminfo | grep "Huge"
```

```sh
sysctl -a | grep huge
# vm.nr_hugepages = 0

free -m
#               total        used        free      shared  buff/cache   available
# Mem:            3622         878        *2689*           5         307        2744
# Swap:          10151          99       10052

echo 100 > /proc/sys/vm/nr_hugepages
free -m
#               total        used        free      shared  buff/cache   available
# Mem:            3622        1078        *2489*           5         307        2544
# Swap:          10151          99       10052
```

Because 100 huge pages times 2 MB (default size of HugePages) is 200, so it allocated 200 MB of memory.

### Memory Usage

- Page cache is used to store data being read from and written to disk
- Buffers is the part of page cache that is used to store blocks of data directly read from a block device (without using an FS)
- Anonymous memory are memory pages that processes allocate and use to store working data

```sh
cat /proc/meminfo

# The Active is the sum of Anonymous Active memory and File Active memory
# The Inactive is the sum of Anonymous Inactive memory and File Inactive memory
# MemTotal - (Active + Inactive) doesn't give you MemFree, because there are other areas, in Kernel, where it reserves its memory. 
```

About cache, swap, and paging:
- Unused memory is for caching
- Some applications need direct I/O to bypass cache
- When using direct I/O, significant amounts of swap are needed to guarantee that contagious memory can be assigned
- Recommended to use SSD devices for swapping
- When under memory pressure, some applications reclaim page cache and/or move anonymous memory to swap
	- `vm.swappiness` dictates the kernel how often to move to swap. High values `200` is moving to swap is preferred, lowers `0` means dropping page cache is preferred.
- When a page from swap is needed, a major page fault is raised

Dropping page cache:

```sh
sync # flushes dirty filesystem data first
echo 1 > /proc/sys/vm/drop_caches

# Consult the man page `man proc`
```

Testing page cache:

```sh
# Create a very big file
dd if=/dev/zero of=/vbf bs=1M count=5120 status=progress

# Flush the cache
sync
drop 1 > /proc/sys/vm/drop_caches

# Check the values first
grep -E "MemFree|MemAvailable|^Cached" /vbf
# MemFree:         2620152 kB
# MemAvailable:    2567580 kB
# Cached:           100520 kB

# Read the file
cat /vbf > /dev/null

# Double check
grep -E "MemFree|MemAvailable|^Cached" /vbf
# MemFree:          124064 kB
# MemAvailable:    2573896 kB
# Cached:          2635424 kB

# Flush the cache
sync
drop 1 > /proc/sys/vm/drop_caches

# Trible check
grep -E "MemFree|MemAvailable|^Cached" /vbf
# MemFree:         2745540 kB
# MemAvailable:    2675876 kB
# Cached:            64572 kB
```

Another test to see how page cache accelerate frequent read:

```sh
# Flush the cache
sync
drop 1 > /proc/sys/vm/drop_caches

grep -E "MemFree|MemAvailable|^Cached" /proc/meminfo
# MemFree:         2736812 kB
# MemAvailable:    2666700 kB
# Cached:            63656 kB

# Read the very big file to nothing
perf stat dd if=/vbf of=/dev/null bs=1M count=5120 status=progress
# 2.160154232 seconds time elapsed

grep -E "MemFree|MemAvailable|^Cached" /proc/meminfo
# MemFree:          154324 kB
# MemAvailable:    2622764 kB
# Cached:          2655044 kB

perf stat dd if=/vbf of=/dev/null bs=1M count=5120 status=progress
# 1.543587703 seconds time elapsed
```

View swap usage:

```sh
echo 2000 > /proc/sys/vm/nr_hugepages # allocate all available memory. to force new apps to use the swap
free -m

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

Free memory plus buffer cache is about ~500 MB, which is bad and red flag.

```sh
echo 0 > /proc/sys/vm/nr_hugepages
```

```
               total        used        free      shared  buff/cache   available
Mem:            3622         965        2589          10         298        2657
Swap:          10151         956        9195
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

### OOM

- When out of memory, the kernel raises an OOM and a process is killed
- `vm.panic_on_oom` raises kernel panic
- Processes with a higher `/proc/<pid>/oom_score` are more likely to get killed
- Tune `/proc/<pid>/oom_scire_adj`, 1000 is more likely, -1000 make a process immune from OOM killer
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
- Add more RAM :)
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

