
# CPU

Generate a CPU workload using `stress-ng`:

```bash
stress-ng --cpu 1 --timeout 30s
```

Start capturing 30 samples with 2 seconds interval:

```bash
sar -o cpu_test_02.sa 2 30
```

Find the most active CPU:

```bash
sadf cpu_test_02.sa -- -u | grep %user | awk '{ print $3,$4,$7,$8 }' | sort -n -k 4
```

# Memory

One thing to remember, in Linux, memory is used for application otherwise it's used as a cache.

Generate a memory pressure workload:

```bash
stress-ng --vm 1 --vm-bytes 6G --vm-keep --timeout 30s
```

Start capturing:

```bash
sar -o memory_pressure_test.sa -r -B -W -S 2 45
```

Interpret the memory, paging and swapping:

```bash
sar -f memory_pressure_test.sa -r -B -W
```

Explanation of the columns:

| Column      | Meaning                                                                                      |
| ----------- | -------------------------------------------------------------------------------------------- |
| `kbmemfree` | The completely free RAM that is not used for anything not even cache                         |
| `kbavail`   | Estimation of available memory which is a free RAM plus a reclaimable memory used as a cache |
| `kbmemused` | Classified as use memory, excluding cache and other kernel memory categories                 |
| `%memused`  | Used memory percentage                                                                       |
| `kbbuffers` | Memory used for block-device buffers                                                         |
| `kbcached`  | Cached filesystem data                                                                       |
| `kbcommit`  | Estimated memory committed to workloads                                                      |
| `%commit`   | Commitment as a percentage of RAM plus swap                                                  |

We can view how `kbcached` and `kbmemfree` correlate:

```bash
dd if=/bigfile.bin of=/dev/null bs=1M
```

Linux kernel will cache the read in the memory so a next read won't hit a disk I/O. After generating this workload, you can see `kbcached` increased while `kbmemfree` decreases. `kbavail` remain relatively the same.

The read is actual in the cache and is reclaimable if another workload needs a memory.

Drop the cache `echo 1 > /proc/sys/vm/drop_caches` and see the `kbcached` went low.