First thing first is to look at top.

Throughput and Latency:
- Some systems need high throughput and some need low latency
- **Throughput**: how much data can be processed by a resource
- **Latency**: how fast data can be processed by a resource
- Optimizing both can be impossible

## Unit of Measurement

SI (System International) is the way how units are presented in a decimal way and used by hardware manufacturers:
- Kilobyte (KB)  = 1,000 bytes
- Megabyte (MB) = 1,000,000 bytes
- Gigabyte (GB) = 1,000,000,000 bytes
- Terabyte (TB) = 1,000,000,000,000 bytes

IEC "International Electrotechnical Commission" uses binary prefixes and found on most software programs:
- Kibibytes (KiB) = 1,024 bytes
- Mebibytes (MiB) = 1,048,576 bytes
- Gibibytes (GiB) = 1,073,741,824 bytes
- Tebibytes (TiB) = 1,099,511,627,776 bytes

How many Tebibytes (TiB) is reported on a 20 Terabyte (TB) disk?

$$
20 \times 1000 \times 1000 \times 1000 \times 1000 = 20,000,000,000,000 \ bytes
$$
$$
20,000,000,000,000 \ bytes \div 1024 \div 1024 \div 1024 \div 1024 = 18.19 \ TiB
$$

How many bytes per second can be transferred by a gigabit network card?

$$
1GbE = 1 \times 1000 \times 1000 \times 1000 = 1,000,000,000 \ \text{bit per second}
$$
$$
1,000,000,000 \div 8 = 125,000,000 \ \text{bytes per second} = 125,000,000 \div 1000000 = 125 \ Mbps
$$

## Hardware Resources

Linux provides several commands for inspecting system hardware. The `dmidecode` command displays detailed DMI/SMBIOS information about components such as the BIOS, motherboard, memory, and processor; because its output is extensive, it is often combined with `grep` to locate specific information.

The `lshw -short` command provides a concise overview of the system's hardware configuration, while `lspci` lists devices connected to the PCI bus, such as network adapters, storage controllers, and graphics devices.

## System Metrics and Common Tools

Overview of a running system metrics:
- Load average
- CPU utilization
- Memory
- Swap usage
- I/O
- Most active processes

Overview of the Linux system stack and observability tools for each stack:

![[linux_observability_tools.png|550]]

Common performance tools:
- strace
- ss
- stap
- iotop
- top
- ps
- vmstat
- numastat
- perf
- sar
- /proc directory
## Processes and Threads

Processes are the default managed entities in the Linux kernel. A process can use multiple threads to perform multitasking. Kernel threads are managed similarly to processes, and kernel routines may run as threads on individual CPUs. User threads are not managed as separate processes, although they can be displayed individually and scheduled independently. More precisely, the Linux kernel schedules **tasks** rather than processes; a task may represent either a single-threaded process or an individual thread within a multithreaded process. To display the threads associated with a specific command, the `ps -C command -L` command can be used, while `ps -efL` displays threads for all running processes.

## Process States

Linux processes can exist in several different states. **Runnable processes** are either currently being serviced by the CPU or are waiting to be scheduled. **Blocking processes**, commonly shown with the state `D`, are in uninterruptible sleep. This usually means the process is waiting for a resource and cannot be interrupted by normal signals. Common causes include I/O waits, NFS or other remote filesystem operations, and hardware-related issues.

**Sleeping processes** are temporarily paused while waiting for an event or resource before they can continue execution. **Zombie processes** are processes that have already completed execution but have not yet been properly cleaned up by their parent process.

A process becomes a zombie when it finishes its task but its parent has not collected its exit status. Although a zombie no longer consumes CPU or normal process memory, it still occupies an entry in the process table, including a PID.

To remove a zombie process, the parent process must reap it by collecting its exit status. Sending the parent a `SIGCHLD` signal may prompt it to perform this cleanup if it has an appropriate signal handler. Another option is to terminate the parent process. The zombie is then re-parented to `init` or another designated subreaper, which normally collects and removes it.

