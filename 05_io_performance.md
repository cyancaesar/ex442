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