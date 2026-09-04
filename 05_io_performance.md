I/O performance problems can usually be identified by observing CPU I/O wait, active disk usage, and read/write throughput. Common tools for this include `top`, `iotop`, `iostat`, `dd` and `fio`.

## Identifying I/O Bottlenecks

Common indicators of poor I/O performance include:
- High `%wa` in `top`, indicating CPU time spent waiting for I/O operations to complete
- High disk activity reported by `iotop`
- Slow read or write performance
- High device utilization or latency reported by `iostat`

Simple I/O tests can be performed with `dd`, while more realistic and configurable workloads can be generated with `fio`.

## Observing I/O Wait with top

One way to generate disk activity is to run multiple direct-read operations:

```bash
dd if=/dev/sda of=/dev/null bs=1M iflag=direct &
dd if=/dev/sda of=/dev/null bs=1M iflag=direct &
dd if=/dev/sda of=/dev/null bs=1M iflag=direct &
```

While the workload is running, `top` can be used to observe CPU activity:

```text
%Cpu(s): 0.1 us, 22.5 sy, 0.0 ni, 43.1 id, 18.8 wa, 14.3 hi, 1.2 si, 0.0 st
```

In this example, `%wa` is `18.8`, meaning a noticeable portion of CPU time is being spent waiting for I/O operations.

## Observing Process I/O with iotop

The `iotop` utility shows which processes or threads are generating disk I/O.

For example, generate a large write workload with:

```bash
dd if=/dev/zero of=/giga.bin bs=1G count=10
```

While the command is running, `iotop` may show output similar to:

```text
Total DISK READ : 0.00 B/s | Total DISK WRITE : 1783.31 M/s
Actual DISK READ: 0.00 B/s | Actual DISK WRITE: 1769.06 M/s

TID PRIO USER DISK READ DISK WRITE COMMAND
73444 be/4 root 0.00 B/s 1783.31 M/s dd if=/dev/zero of=/giga.bin bs=1G count=10
```

This indicates that the `dd` process is generating roughly `1.8 GB/s` of write traffic.

## Observing Device I/O with iostat

The `iostat` command reports CPU statistics and per-device I/O activity.

To collect five reports at two-second intervals:

```bash
iostat 2 5
```

```result
avg-cpu: %user %nice %system %iowait %steal %idle
          0.00 0.00 37.14 0.08 0.00 62.77

Device tps kB_read/s kB_wrtn/s kB_dscd/s
dm-0 900.50 0.00 1854978.00 0.00
dm-1 0.00 0.00 0.00 0.00
sda 2215.50 0.00 1855234.00 0.00
```

Useful fields include:
- `tps`: I/O operations issued to the device per second
- `kB_read/s`: kilobytes read per second
- `kB_wrtn/s`: kilobytes written per second
- `%iowait`: percentage of CPU time spent waiting for I/O

Extended `iostat` output can provide additional metrics such as device utilization, queue depth, and request latency.

## I/O Testing Tools

`dd` is useful for simple sequential read and write tests. It is easy to use, but it does not accurately represent many real-world workloads.

`fio` is a more advanced I/O workload generator. It can simulate sequential or random reads and writes, different block sizes, queue depths, concurrency levels, and mixed workloads.

For serious benchmarking or workload simulation, `fio` is generally more useful than `dd`.

## I/O Schedulers

I/O schedulers determine how block I/O requests are queued and dispatched to storage devices. Modern Linux systems use the multi-queue block layer, which allows requests to be distributed across multiple hardware or software queues.

Common schedulers include:

- `mq-deadline`: prioritizes requests using deadlines to prevent excessive waiting and provide predictable latency.
- `kyber`: designed for fast storage devices and attempts to control latency by limiting the number of synchronous and asynchronous requests in flight.
- `bfq`: focuses on fairness and interactive responsiveness. It is often useful on slower or shared devices where latency matters more than maximum throughput.
- `none`: performs minimal software scheduling and is commonly suitable for devices such as NVMe drives that already perform their own internal scheduling.

To view the scheduler available for a device:

```bash
cat /sys/block/sda/queue/scheduler
```

```result
none [mq-deadline] kyber bfq
```

The scheduler enclosed in brackets is currently active.

To change it temporarily:

```bash
echo bfq > /sys/block/sda/queue/scheduler
```

The change can be verified with:

```bash
cat /sys/block/sda/queue/scheduler
```

## Dirty Pages and Writeback

Normal buffered writes are first placed into the Linux page cache rather than being written immediately to storage. Pages containing modified data that has not yet been written to the backing device are called **dirty pages**.

Memory is organized into pages, commonly 4 KiB on x86-64 systems, although page size depends on the architecture and configuration.

Kernel writeback behavior is controlled through parameters under:

```bash
ls /proc/sys/vm/dirty*
```

Common parameters include:

```text
dirty_background_bytes
dirty_background_ratio
dirty_bytes dirty_expire_centisecs
dirty_ratio
dirtytime_expire_seconds
dirty_writeback_centisecs
```

Two important timing parameters are:

- `vm.dirty_expire_centisecs`: determines how old dirty data must become before it is considered eligible for background writeback.
- `vm.dirty_writeback_centisecs`: determines how frequently kernel writeback threads wake up to inspect dirty pages.

These values are expressed in **centiseconds**, where one centisecond equals `0.01` seconds.

For example:

```bash
cat /proc/sys/vm/dirty_expire_centisecs
```

```result
3000
```

This represents $3000 \times 0.01=30 \ \text{seconds}$

And:

```bash
cat /proc/sys/vm/dirty_writeback_centisecs
```

```result
1500
```

This represents $1500 \times 0.01 = 15 \ \text{seconds}$

A writeback interval of 15 seconds does **not** mean every dirty page is automatically written after 15 seconds. It means the kernel periodically wakes to inspect dirty pages, while `dirty_expire_centisecs` determines when dirty data becomes old enough for normal background writeback.

Dirty-page information can be inspected with:

```bash
egrep -i "buffers|writeback|dirty" /proc/meminfo
```

## Testing I/O Performance

I/O performance should be tested using workloads that resemble the actual application behavior. Useful test patterns include:

- Sequential reads
- Sequential writes
- Random reads
- Random writes
- Small-block operations
- Large-block operations
- Mixed read/write workloads
- Different queue depths and levels of concurrency

### Testing with fio

`fio` provides much more control than `dd` and is better suited for realistic I/O benchmarking.

To view `fio` manual page:

```bash
curl cht.sh/fio
```


Example:

```bash
fio \
	--name=test1 \
	--ioengine=libaio \
	--iodepth=1 \
	--rw=randwrite \
	--bs=4k \
	--direct=1 \
	--size=512M \
	--numjobs=2 \
	--group_reporting \
	--filename=/tmp/testfile
```

Important options include:

- `--rw=randwrite`: performs random writes.
- `--bs=4k`: uses a 4 KiB block size.
- `--iodepth=1`: allows one outstanding I/O request per job.
- `--numjobs=2`: runs two workers.
- `--direct=1`: performs direct I/O and bypasses the normal page cache.
- `--size=512M`: defines the test workload size.

Useful output metrics include:

- **IOPS**: number of I/O operations completed per second.
- **Bandwidth**: amount of data transferred per second.
- **Latency**: time required to complete an I/O request.
- **Completion latency (`clat`**): time between submission of an I/O request and its completion.

For latency measurements, lower values are generally better.
## Improving I/O Performance

Poor I/O performance can originate from several layers of the storage stack, so tuning should not focus only on the I/O scheduler.

Areas to investigate include:

- Storage architecture
    - Separate heavily used workloads across different devices.
    - Consider SAN, RAID, NVMe, or other storage architectures where appropriate.
- Filesystem selection
    - XFS
    - Ext4
    - Btrfs
    - ReiserFS
    - ZFS
- Filesystem mount options
- Journaling behavior
- I/O scheduler selection
- Application I/O patterns
- Queue depth and concurrency
- Memory and page-cache behavior

The best filesystem or scheduler depends on the workload rather than there being one universally fastest option.

## Filesystem Journaling

Filesystems such as Ext4 use a journal to protect filesystem consistency after crashes.

Ext4 supports several data journaling modes through the `data=` mount option.

### ordered

This is the normal default mode. Metadata is journaled, while file data is written to its normal filesystem location before the corresponding metadata transaction is committed.

```result
data=ordered
```

It provides a balance between performance and consistency.
### writeback

Only metadata is journaled, and there is no ordering guarantee that file data reaches disk before the corresponding metadata transaction is committed.

```
data=writeback
```

This may reduce some write-ordering overhead but provides weaker guarantees about file contents after a crash.

### journal

Both file data and metadata are first written to the journal before being written to their final filesystem locations.

```
data=journal
```

This provides stronger journaling behavior but usually introduces additional write overhead.

A **journal** and **Copy-on-Write (CoW)** are different mechanisms. Journaling records changes in a transaction log so that filesystem operations can be recovered consistently after a crash. Copy-on-Write instead writes modified data or metadata to new locations before updating references to them. Filesystems such as Btrfs and ZFS rely heavily on CoW, while Ext4 primarily uses journaling.

## Basic Testing with dd

`dd` can be useful for simple throughput tests.

Example:

```bash
time dd if=/dev/zero of=/bigfile bs=1M count=10240
```

```result
10240+0 records in
10240+0 records out
10737418240 bytes (11 GB, 10 GiB) copied, 4.90766 s, 2.2 GB/s

real    0m4.909s
user    0m0.002s
sys     0m4.790s
```

The timing values mean:

- `real`: total elapsed wall-clock time.
- `user`: CPU time spent executing user-space code.
- `sys`: CPU time spent executing kernel code on behalf of the process.

`real` time is still meaningful because it tells you how long the operation took from the user's perspective. However, it includes time spent waiting for I/O, being descheduled, and competing with other workloads, so it should not be interpreted as pure CPU execution time.

Another important consideration is caching. A normal `dd` write may complete partly through the page cache rather than representing sustained physical storage performance. For storage benchmarking, options such as direct I/O or `fio` are usually more appropriate.
## Demo

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