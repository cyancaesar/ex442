Common tools for network analysis and performance:
- `ip -s link` for packet statistics of an NIC
- `ethtool ens160` for statistics about an NIC
- **qperf** tool for bandwidth and latency performance


> [!NOTE] qperf
> qperf is in EPEL repository, to install on RHEL 9:
> 
> ```bash
> dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
> ```

Packet statistics:

```bash
ip -s link show ens160
```

```result
RX:  bytes packets errors dropped  missed   mcast
     211926104  361308      0       0       0     334
TX:  bytes packets errors dropped carrier collsns
     601490711  452328      0       0       0       0
```

Way to approach fixing network performance goes up in the stack:
- Involve networking hardware
- Consider modifying MTU
- Analyze availability of network buffers in memory

Gather bandwidth and latency between two hosts using **qperf**, on the server host:

```bash prompt:"[root@sysperf ~]#"
qperf
```

On the client hosts:

```bash prompt:"[root@sysperf-client ~]#"
qperf sysperf tcp_bw tcp_lat
```

```result
tcp_bw:
    bw  =  632 MB/sec
tcp_lat:
    latency  =  111 us
```