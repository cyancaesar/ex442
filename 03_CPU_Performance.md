## Schedulers

- The Linux kernel uses schedulers to determine which process runs next
- Based on the process priority, the process will be placed in the run queue
- Real time processes have a static priority between 1 and 99. Highest static priority goes first
- Dynamic priority is adjusted by the scheduler. Processes that wait longer will get a higher priority
- Nice values can be used to adjust priority for non-realtime processes

To show process priority, RT priority, nice value and scheduler policy:

```bash
ps axo pid,pri,rtprio,ni,cls,comm
```

```result
PID PRI RTPRIO  NI CLS COMMAND
      1  19      -   0  TS systemd
      2  19      -   0  TS kthreadd
      3  19      -   0  TS pool_workqueue_
      4  39      - -20  TS kworker/R-rcu_g
      5  39      - -20  TS kworker/R-sync_
      6  39      - -20  TS kworker/R-slub_
      7  39      - -20  TS kworker/R-netns
      9  39      - -20  TS kworker/0:0H-events_highpri
     10  19      -   0  TS kworker/u512:0-events_unbound
     11  39      - -20  TS kworker/R-mm_pe
```

What is CPU time (slice):
- Only one process can run at a time *on a core*, which is why each process gets a time slice before it gets pre-empted
- Each time a process yields the processor, its current state is stored by the kernel so that when resuming it can easily continue where it left of
- This is known as **context switching**
- a **voluntary context switch** is initiated by the process, a **non-voluntary context switch** is initiated by the kernel

Scheduling policies:
- `SCHED_FIFO` `-f`: a real time policy which runs a process until it is blocked by IO, calls `sched_yield`, or a higher priority process comes. This scheduler should only be used for kernel processes
- `SCHED_RR` `-r`: a real time policy that uses time slices
- `SCHED_NORMAL` `-o`: the normal scheduling policy, where processes run using time sharing
- `SCHED_BATCH` `-b`: optimized for larger (batch) processes, as it does not preempt as often as `SCHED_NORMAL`
- `SCHED_IDLE` `-i`: runs processes with the lowest priority (lower than nice value 19)
- `SCHED_DEADLINE` `-d`: schedules tasks according to a deadline, where the process with the earlies deadline goes first

SCHED_DEADLINE:
- Added by Red Hat
- It ensures a process can get the CPU time it needs and can be tweaked

Tuning parameters:
- period is the activation pattern (how often) of a realtime task
- runtime is the amount of time the process needs to produce its output
- deadline is the maximum amount of time the process gets to complete its task

If a new video frame is received every 20 milliseconds, the period is 20 ms, if processing normally takes 5 ms, that is the runtime, and a 10 ms deadline can be set to ensure timely process termination

```bash
chrt -d --sched-runtime 5000000 --sched-deadline 10000000 --sched-period 20000000 0 videotask
```


> [!NOTE] Beware of measurements
> CPU is measured in nanoseconds
> Memory in microseconds
> Disk I/O in milliseconds


**chrt**:
- Can be used to run process in a specific scheduler policy
- `chrt -p 2123` shows current scheduler properties of a process
- `chrt -f 44 /bin/task` runs task in the FIFO scheduler with a priority of 44
- `chrt -o -p 2123` puts the process in the OTHER scheduler

Systemd Scheduling options:
- `CPUSchedulingPolicy` determines the scheduling policy, currently deadline is not supported
- `CPUSchedulingPrioritry` sets the priority

CPU Affinity and Isolation:
- Each time a process is rescheduled, it can run on any CPU
- To improve the chance of having a cache hit, **CPU pinning** can be used
- `CPUAffinity` Systemd setting allows to specify on which CPUs a process is allowed to run

