
# Performance Co-Pilot

![[pcp.png]]

The main components of PCP:
- `pmcd`: Performance Metric Collector Daemon, it provides access to metrics collected by agents.
- `pmlogger`: Records metrics periodically into archive files.
- `pminfo`: Discovers metrics and displays their metadata or values.
- `pmval`: Displays metric values, including historical samples.
- `pmlogsummary`: Summarizes archived metrics, including averages, maximum and minimum.
- `pmafm`: Archive Folio Manager. Manages and processes groups of archives.

An example metric:

```
disk.dev.write_bytes
```

This metric represents bytes written to disk devices, as a **cumulative counter**

For historical analysis, PCP can calculate the rate of change between samples to determine write throughput.

PCP metrics may be **cumulative counters** or **instantaneous values**.

## PCP Archives

When `pmlogger` records performance data, it creates an archive consisting of multiple files:

```
/var/log/pcp/pmlogger/linperf.local/
├── 20261021.010000.meta
├── 20261021.010000.index
├── 20261021.010000.0
└── 20261021.010000.1
```

- `.meta` describes metrics, instances, types and units
- `.index` is a temporal index for locating samples
- `.0` first volume containing recorded values
- `.1` second volume, if logging rolls over

All these files belong to a one logical archive, and is analyzed by using the base name of the archive:

```bash
pmlogsummary -a /var/log/pcp/pmlogger/linperf.local/20261021.010000
```

## PCP Folio

A folio is a control file that groups one or more PCP archives together. It's like a catalog of recorded performance data.

```
/root/.pcp/pmlogger/
├── 20261021.06.00.01.folio
├── linperf.local/
│   ├── recording.meta
│   ├── recording.index
│   └── recording.0
└── slownix.local/
    ├── recording.meta
    ├── recording.index
    └── recording.0
```

A `.folio` file references archives, it doesn't contain a performance sample.

Viewing an archives that is inside a folio:

```bash
pmafm /root/.pcp/pmlogger/20261021.06.00.01.folio list
```
# Setup PCP

Install PCP:

```bash
dnf install -y pcp
```

Activate it:

```bash
systemctl enable --now pmcd.service
```

Check PCP configuration:

```bash
pcp
```

```result
Performance Co-Pilot configuration on linperf.local:

 platform: Linux linperf.local 6.12.0-211.51.1.el10_2.x86_64 #1 SMP PREEMPT_DYNAMIC Tue Sep 1 00:07:46 EDT 2026 x86_64
 hardware: 8 cpus, 5 disks, 2 nodes, 7688MB RAM
 timezone: +03-3
 services: pmcd
     pmcd: Version 7.0.3-5, 9 agents, 4 clients
     pmda: root pmcd proc pmproxy xfs linux mmv kvm jbd2
 pmlogger: primary logger: /var/log/pcp/pmlogger/linperf.local/20261021.02.36
     pmie: primary engine: /var/log/pcp/pmie/linperf.local/pmie.log
```

Discover available metrics:

```bash
pminfo
```

```result
disk.wwid.avg_rqsz
disk.wwid.r_avg_rqsz
disk.wwid.w_avg_rqsz
disk.wwid.d_avg_rqsz
disk.wwid.util
proc.psinfo.age
proc.io.total_bytes
proc.hog.cpu
proc.hog.mem
proc.hog.disk
...
```

There are around 2861 metrics available.

# Metric Description and Value

Finding metrics under kernel:

```bash
pminfo kernel
```

Memory metrics:

```bash
pminfo mem.util
```

Disk related metrics:

```bash
pminfo disk.dev
```

For inspecting the metric description:

```bash
pminfo -d disk.dev.write_bytes
```

```result
disk.dev.write_bytes
    Data Type: 64-bit unsigned int  InDom: 60.1 0xf000001
    Semantics: counter  Units: Kbyte
```

It shows that this metric is a counter and the recorded unit is KiB.

Retrieving the current value of a metric:

```bash
pminfo -f disk.dev.write_bytes
```

`pmval` is actually used to retrieve live and historical metrics:

```bash
pmval -t 2sec -s 5 mem.util.user
```

```result
metric:    mem.util.used
host:      linperf.local
semantics: instantaneous value
units:     Kbyte
samples:   5
interval:  2.00 sec
              3512708
              3512744
              3511768
              3511644
              3511164
```

# Recording Performance Metrics `pmlogger`

Setting up a quick lab:

```bash
mkdir -p ~/pcp
cat > ~/pcp/logger.conf <<'EOF'
log mandatory on every 2 seconds {
	kernel.all.cpu.user
	kernel.all.cpu.sys
	kernel.all.cpu.idle
	mem.util.used
	mem.util.free
	disk.dev.write_bytes
	disk.dev.read_bytes
}
EOF
```

Start recording:

```bash
pmlogger -c ~/pcp/logger.conf -l ~/pcp/pmlogger.log -T 2min ~/linperf
```

Create some workload for CPU, memory and disk I/O while the recording is running:

```bash
stress-ng --cpu 4 --timeout 20s
stress-ng --vm 1 --vm-bytes 512M --vm-keep --timeout 20s
stress-ng --hdd 1 --hdd-bytes 256M --hdd-opts direct --temp-path ~/pcp --timeout 20s
```

After the logger finishes inspect the archives:

```bash
ls ~/pcp
```

```result
linperf.0
linperf.index
linperf.meta
logger.conf
pmlogger.log
```

We can verify the recording by viewing the label:

```bash
pmdumplog -l linperf
```

```result
Log Label (Log Format Version 3)
Performance metrics from host linperf.local
    commencing Wed Oct 21 02:43:53.894127452 2026
    ending     Wed Oct 21 02:45:53.873952880 2026
```

This shows you this recording started at `10/21/2026 02:43:53` and ended at `10/21/2026 02:45:53`, only 2 minutes recording.

# Historical Analysis

This part will cover how we can analyze the recording and discover the metrics stored in an archive.

Discover recorded metrics:

```bash
pminfo -a linperf
```

```result
kernel.all.cpu.user
kernel.all.cpu.sys
disk.dev.read_bytes
disk.dev.write_bytes
mem.util.used
mem.util.free
```

For disk I/O metric, there is a way to know the recorded instances (devices):

```
pminfo -a linperf -f disk.dev.write_bytes
```

```result
disk.dev.write_bytes
    inst [0 or "nvme0n1"] value 3970273
    inst [1 or "sdd"] value 0
    inst [2 or "sdb"] value 0
    inst [3 or "sda"] value 0
    inst [4 or "sdc"] value 0
```

We can examine now to read the historical samples with `pmval`.

Read performance data about disk write throughput over time:

```bash
pmval -a linperf -t 2sec -f 3 disk.dev.write_bytes -i nvme0n1
```

```result
                          nvme0n1
02:44:45.894                0.000
02:44:47.894                2.000
02:44:49.894                0.000
02:44:51.894                0.000
02:44:53.894             7209.000
02:44:55.894           244979.500
02:44:57.894           148938.500
02:44:59.894           249885.500
02:45:01.894           150561.000
02:45:03.894           243074.500
02:45:05.894           141161.500
02:45:07.894           255590.000
02:45:09.894           168482.000
02:45:11.894           225244.000
02:45:13.894           147292.500
02:45:15.894                7.000

```

It shows that the block device `nvme0n1` has been active the most (removed other devices column).

We can identify at `02:45:07.894` that disk write throughput was `255590.000 Kbyte/s` = `249.5996 Mbyte/s` = `261.7241 MB/s` which is its peak.

Is this the way of finding the peak? No, and it's not practical. The `pmlogsummary` tool gives you aggregations about averages, lowest, highest value for the recorded metrics.

The actual use of `pmval` is to spot the burst of activitie

# Average, Maximum, Minimum with `pmlogsummary`

We can use `pmlogsummary` to find the highest and the when the highest record occurred.

Find the the maximum disk write:

```bash
pmlogsummary -H -M -I linperf disk.dev.write_bytes
```

```result
metric time_average maximum maximum_time units
disk.dev.write_bytes ["nvme0n1"] 33604.764 255590.548 02:45:07.894 Kbyte / sec
disk.dev.write_bytes ["sdd"] 0.000 0.000 02:43:53.894 Kbyte / sec
disk.dev.write_bytes ["sdb"] 0.000 0.000 02:43:53.894 Kbyte / sec
disk.dev.write_bytes ["sda"] 0.000 0.000 02:43:53.894 Kbyte / sec
disk.dev.write_bytes ["sdc"] 0.000 0.000 02:43:53.894 Kbyte / sec
```

This gives you a better aggregations.

# Create a Folio

Create a folio:

```bash
/usr/libexec/pcp/bin/mkaf linperf.0 > linperf.folio
```

```result
PCPFolio
Version: 1
# use pmafm(1) to process this PCP archive folio
#
Created: on linperf.local at Wed Oct 21 05:37:34 AM +03 2026
Creator: pmchart
# 		Host			Basename
#
Archive:        linperf.local           linperf
```

As simple as that!