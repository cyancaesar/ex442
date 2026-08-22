# Analyzing and Tuning I/O Perf.

Basic indicators of bad I/O:
- High `wa` parameter in top
	- `wa` is waiting for I/O
- High usage in iotop
- Slow writes
	- Simple tests using `dd`
	- Complex tests using `fio`

Doing stress copy on an a disk:

```sh
dd if=/dev/sda of=/dev/null bs=1M iflag=direct &
dd if=/dev/sda of=/dev/null bs=1M iflag=direct &
dd if=/dev/sda of=/dev/null bs=1M iflag=direct &
```

```
top - 18:46:46 up 19:31,  3 users,  load average: 0.71, 0.20, 0.53
Tasks: 368 total,   4 running, 364 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.1 us, 22.5 sy,  0.0 ni, 43.1 id, 18.8 wa, 14.3 hi,  1.2 si,  0.0 st
MiB Mem :   3622.3 total,    675.1 free,   1409.8 used,   1818.7 buff/cache
MiB Swap:   4008.0 total,   4007.7 free,      0.3 used.   2212.5 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  73371 root      20   0  222036   2980   1844 R  62.5   0.1   0:07.53 dd
  73370 root      20   0  222036   2984   1852 R  61.5   0.1   0:08.10 dd
  73369 root      20   0  222036   2984   1852 R  57.5   0.1   0:08.65 dd
```

You can spot the `wa` is `18.8%`

Running another writing command:

```sh
dd if=/dev/zero of=/giga.bin bs=1G count=10
```

```
Total DISK READ :       0.00 B/s | Total DISK WRITE :    1783.31 M/s
Actual DISK READ:       0.00 B/s | Actual DISK WRITE:    1769.06 M/s
    TID  PRIO  USER     DISK READ DISK WRITE>    COMMAND
  73444 be/4 root        0.00 B/s 1783.31 M/s dd if=/dev/zero of=/vbf bs=1G count=10
      1 be/4 root        0.00 B/s    0.00 B/s systemd rhgb --switched-root --system --deserialize 31
```

iotop reports that disks write are ~1800Mb/s.

Another utility for I/O observation is `iostat`:

```sh
iostat 2 5
```

```
Linux 5.14.0-611.5.1.el9_7.x86_64 (sysperf.lab)         08/17/2026      _x86_64_        (6 CPU)
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.00    0.00   37.14    0.08    0.00   62.77

Device             tps    kB_read/s    kB_wrtn/s    kB_dscd/s    kB_read    kB_wrtn    kB_dscd
dm-0            900.50         0.00   1854978.00         0.00          0    3709956          0
dm-1              0.00         0.00         0.00         0.00          0          0          0
sda            2215.50         0.00   1855234.00         0.00          0    3710468          0
sr0               0.00         0.00         0.00         0.00          0          0          0
sr1               0.00         0.00         0.00         0.00          0          0          0

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.08    0.00    0.08    0.00    0.00   99.83

Device             tps    kB_read/s    kB_wrtn/s    kB_dscd/s    kB_read    kB_wrtn    kB_dscd
dm-0              0.50         2.00         0.00         0.00          4          0          0
dm-1              0.00         0.00         0.00         0.00          0          0          0
sda               0.50         2.00         0.00         0.00          4          0          0
sr0               0.00         0.00         0.00         0.00          0          0          0
sr1               0.00         0.00         0.00         0.00          0          0          0
```

## I/O Schedulers

I/O schedulers optimizes for read requests or write requests. New multi-queue schedulers have been introduced to allow I/O operations to be mapped to multiple hardware or software request queues.
- mq-deadline: requests are sorted by expiration time to ensure that operations are executed before expiration
- kyber: specific to support fast throughput on SSD, where read requests are handled synchronous, and write requests are asynchronous.
- bfq: recommended on slower I/O devices, and where interactive system response is a high priority. Don't use if throughput is a priority
- none: use on a very fast I/O devices (NVMe) to not optimize anything

View write buffers kernel parameters, called *dirty*:

```sh
ls /proc/sys/vm/dirty*
# dirty_background_bytes
# dirty_background_ratio
# dirty_bytes
# dirty_expire_centisecs
# dirty_ratio
# dirtytime_expire_seconds
# dirty_writeback_centisecs
```

The dirty parameters are expressed in centisecs, 1 centisecs is 0.01 seconds. Important parameters are `dirty_expire_centisecs` and `dirty_writeback_centisecs`.
- `dirty_expire_centisecs`: used to define when dirty data is old enough to be eligible for writeout by the kernel flusher threads.
	- Data which has been dirty in-memory for longer than this interval will be written out next time a flusher thread wakes up.
- `dirty_writeback_centisecs`: The kernel flusher threads will periodically wake up and write old data out to disk.

```sh
cat dirty_expire_centisecs
# 3000
# means 30 seconds in total

cat dirty_writeback_centisecs
# 1500
# means 15 seconds in total
```

When a data is written to disk, it is first written in a buffer, and kept written into the buffer, so it gets written in disk in the most efficient way.

First the data will wait in the buffer until the dirty cache to expire:
- Cache is in a memory
- Memory is organized in pages
- By default, page size is 4Kb.

So these pages are flagged as dirty, and when these pages haven't yet committed to disk, after 15 seconds, it is written.

View buffer information:

```sh
egrep -i "buffers|writeback|dirty" /proc/meminfo
```

Check the active scheduler:

```sh
cat /sys/block/sda/queue/scheduler
# none [mq-deadline] kyber bfq

echo bfq > /sys/block/sda/queue/scheduler
# none mq-deadline kyber [bfq]
```

For testing I/O performance, different behavior should be tested:
- sequential reads
- random reads
- writing small files
- writing large files

Testing with `fio`:
```sh
fio --name=test1 --ioengine=libaio --iodepth=1 --rw=randwrite --bs=4k --direct=1 --size=512M --numjobs=2 --group-reporting --filename=/tmp/testfile
```

- Monitor `clat (used)`: lower is better
- `--direct=1`: bypass writing to memory page cache

```sh
curl cht.sh/fio
```

Fixing bad I/O performance, you must look at these options:
- Architecture: separate disk devices and SAN storage
- File System choice
	- XFS, Ext4, Btrfs, ZFS, (ReiserFS -> dark end) .. and ZFS
- File System Journaling options
- I/O Scheduler choice

Ext4 and other Ext-family file system, have an option for mounting called `data=` which could be either:
- `ordered` (default): journals only the metadata, forces data blocks to be written to disk before metadata is committed to the journal
- `writeback`: journals only the metadata, data blocks are written to disk independently of metadata updates
- `journal`: full journaling mode, both file and metadata are written to the journal before being placed into the permanent block

Journal is a transaction log where Copy on Write (CoW) writes the data to a new data block and updates pointers atomically without overwriting the old data.

```sh
[root@sysperf ~]# time dd if=/dev/zero of=/bigfile bs=1M count=10240
10240+0 records in
10240+0 records out
10737418240 bytes (11 GB, 10 GiB) copied, 4.90766 s, 2.2 GB/s

real    0m4.909s
user    0m0.002s
sys     0m4.790s
```

the `real` time is the clock time, and in a multi-tasking system, it is unreliable, because it includes the time of the process that has pre-emptively moved from CPU (context switching).

- Prepare a new virtual disk and partition it into two partitions and format both as Ext4, XFS.
- Use the simple I/O test using dd: `dd if=/dev/zero of=<output> bs=1M count=10240`
- Use `iotop` for monitoring the tests 
- Use different I/O scheduler options through sysfs and view any improvement?
- Use different journaling options for Ext4 and view any improvement?

Creating one 20GB NVMe virtual disk:

```sh
fdisk /dev/nvme0n1
# partition it into two 5GB paritions
lsblk
```

```
nvme0n1       259:0    0    20G  0 disk
├─nvme0n1p1   259:1    0     5G  0 part
└─nvme0n1p2   259:2    0     5G  0 part
```

Format `nvme0n1p1` to Ext4 filesystem and `nvme0n1p2` to XFS filesystem:

```sh
mkfs.ext4 /dev/nvme0n1p1
mkfs.xfs /dev/nvme0n1p2
```

Create dedicated directory for each and mount them:

```sh
mkdir -p /mnt/ext4 /mntxfs
mount -t ext4 -o data=ordered /dev/nvme0n1p1 /mnt/ext4 # data=ordered isn't needed since its the default option
mount -t xfs /dev/nvme0n1p2 /mnt/xfs
```

Run `iotop` on a separate terminal and perform a simple `dd` test on Ext4 directory:

```sh
cat /sys/block/nvme0n1/queue/scheduler
# [none] mq-deadline kyber bfq

dd if=/dev/zero of=/mnt/ext4/test.bin bs=1M count=10240
# Total DISK READ :       0.00 B/s | Total DISK WRITE :    1081.28 M/s
# Actual DISK READ:       0.00 B/s | Actual DISK WRITE:    1118.21 M/s
# 5138022400 bytes (5.1 GB, 4.8 GiB) copied, 4.56393 s, 1.1 GB/s

umount /mnt/ext4
mount -t ext4 -o data=writeback /dev/nvme0n1p1 /mnt/ext4
dd if=/dev/zero of=/mnt/xfs/test.bin bs=1M count=4900
# Total DISK READ :       0.00 B/s | Total DISK WRITE :    1061.02 M/s
# Actual DISK READ:       0.00 B/s | Actual DISK WRITE:    1280.29 M/s
# 5138022400 bytes (5.1 GB, 4.8 GiB) copied, 4.54832 s, 1.1 GB/s

umount /mnt/ext4
mount -t ext4 -o data=journal /dev/nvme0n1p1 /mnt/ext4
dd if=/dev/zero of=/mnt/xfs/test.bin bs=1M count=4900
# Total DISK READ :       0.00 B/s | Total DISK WRITE :     446.64 M/s
# Actual DISK READ:       0.00 B/s | Actual DISK WRITE:     895.80 M/s
# 5138022400 bytes (5.1 GB, 4.8 GiB) copied, 11.0868 s, 463 MB/s

echo bfq > /sys/block/nvme0n1/queue/scheduler
mount -t ext4 -o data=journal /dev/nvme0n1p1 /mnt/ext4
dd if=/dev/zero of=/mnt/xfs/test.bin bs=1M count=4900
# 5138022400 bytes (5.1 GB, 4.8 GiB) copied, 103.559 s, 49.6 MB/s
# THIS TAKES MUCH MORE

echo kyber > /sys/block/nvme0n1/queue/scheduler
# BAD

echo none > /sys/block/nvme0n1/queue/scheduler
dd if=/dev/zero of=/mnt/xfs/test.bin bs=1M count=4900
# Total DISK READ :       0.00 B/s | Total DISK WRITE :       0.00 B/s
# Actual DISK READ:       0.00 B/s | Actual DISK WRITE:    1175.81 M/s
# 5138022400 bytes (5.1 GB, 4.8 GiB) copied, 2.2857 s, 2.2 GB/s

echo kyber > /sys/block/nvme0n1/queue/scheduler
dd if=/dev/zero of=/mnt/xfs/test.bin bs=1M count=4900
# Total DISK READ :       0.00 B/s | Total DISK WRITE :       2.06 G/s
# Actual DISK READ:       0.00 B/s | Actual DISK WRITE:       2.02 G/s
# 5138022400 bytes (5.1 GB, 4.8 GiB) copied, 2.282 s, 2.3 GB/s
```


> [!INFO] Test Review
> This simple test shows XFS outperforms Ext4 filesystem by ~50%, with no I/O scheduler set and with I/O scheduler set to kyber.
> 
> While Ext4 with no I/O scheduler set does an equal performance of `ordered` and `writeback`. With `journal` data option it takes more than the double.
> 
> With Ext4 data option set to `journal` and an I/O scheduler, it takes a really long time.


> [!IMPORTANT] TuneD and I/O Scheduler
> Review on how to set a TuneD profile with a custom elevator parameter that specifies disk I/O scheduler.


# tmpfs

Temporary mount of `/tmp` on memory:

```sh
systemctl list-unit-files tmp*
systemctl cat tmp.mount
systemctl enable tmp.mount
```

Be careful of starting it now, it will wipe the `/tmp`.

# Memory Performance Tuning

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

#### Page Tables and TLB

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

#### Memory Usage

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

#### Overcommitting Memory

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

#### OOM

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

