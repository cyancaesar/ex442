System V IPC is a set of three mechanisms in Unix-like operating systems that let independent programs share data and coordinate their work.

The three mechanisms are:
- Message Queue
- Semaphores
- Shared Memory

The scope will focus only on SYSV IPC shared memory.

The shared memory can be controlled by kernel parameters:
- `kernel.shmmax`
- `kernel.shmall`
- `kernel.shmmni`
- `kernel.shm_rmid_forced`

The `kernel.shmmax` is a kernel parameter for setting the maximum limit size of a **single** shared memory segment (*bytes*).

The `kernel.shmall` is the total shared memory allocation limit (*system page units*).
- `shmall` should always be at least `ceil(shmmax/PAGE_SIZE)`

The `kernel.shmmni` is the maximum number of System V shared memory segments.

The `kernel.shm_rmid_forced` automatically remove segments when no processes remain attached.

Inspecting the values of these parameters can be done using `sysctl` of `ipcs`:

```bash
sysctl kernel.shmmax
sysctl kernel.shmall
sysctl kernel.shmmni
ipcs -lm
```

```result
------ Shared Memory Limits --------
max number of segments = 4096
max seg size (kbytes) = 18014398509465599
max total shared memory (kbytes) = 18446744073709551612
min seg size (bytes) = 1

```


> [!QUESTION] Configure 2 GiB shared memory limits
> Configure the system to allow individual System V shared memory segments of up to 1 GiB, with total allocation limit of 2 GiB and a maximum of 4096 segments. Make the configuration persistent.

`shmmax` would be $1 \text{ }GiB \times 1024 \times 1024 \times 1024=1073741824 \text{ }bytes$.

`shmall` would be $2 \text{ }GiB \times 1024 \times 1024 \times 1024/4096=524288 \text{ }pages$

`shmni` is `4096` segments

```sh
sysctl -w kernel.shmmax=1073741824
sysctl -w kernel.shmall=524288
sysctl -w kernel.shmmni=4096
```

Apply them persistently by creating a tunable file at `/etc/sysctl.d/90-sysv-sh.conf` then load it with `sysctl --system`.