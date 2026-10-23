
# Memory Pages

Linux manages virtual memory in a fixed-size blocks called **pages**.

![[memory_pages.png]]

A typical x86_64 system, the default page size is 4 KiB. The illustration above shows the pages as a fixed length block, which is called pages.

But the actual total pages for allocating 64 MiB (not covered in the drawing) is actually 16384

$$
\text{Total Pages}=\frac{\text{Total Allocation KiB}}{\text{Page Size KiB}}=\frac{65536}{4}=16384 \text{ pages}
$$

Every virtual page needs a translation to a physical memory location. Linux uses **page tables** to maintain these translations.

# Page Table

Applications work with virtual memory addresses, while physical RAM uses physical addresses. The CPU Memory Management Unit (MMU) translates between them using page tables, which consumes time for the CPU to walk through them.

**TLB** (Translation Lookaside Buffer) caches the recent translation of the page table to a fast cache inside the CPU, instead of walking the page tables each time.

When a CPU requests a memory address, the CPU looks on its cache, if the address is cached, it returns it, if not (**TLB miss**). Otherwise it will walk the whole page table in memory.

The **TLB** has a limited capacity, it can't cache every translation for a large application. This is where **HugePages** become useful. which primarily reduce the number of translations.

**TLB flush** is a cached translations that are invalidated.

# HugePages

HugePages allow Linux to use larger memory pages instead of the standard 4 KiB pages. On x86_64 RHEL systems, supported sizes include:
- Standard page 4 KiB
- HugePage 2 MiB
- Gigantic Page 1 GiB

The first example of  64 MiB allocation with standard page size 4 KiB would require 16384 pages, with HugePages 2 MiB that would only require 32 pages.

The benefits are:
- Smaller page-table memory overhead
- Greater memory coverage per TLB entry
- Potentially fewer TLB misses and page-table walks
- Improved performance for applications accessing large memory regions

# HugeTLB vs. Transparent HugePages (THP)

Linux offers two major mechanisms:

For allocation:
- HugeTLB is explicitly configured
- THP is largely automatic

For page reservation:
- HugeTLB usually preallocated
- THP is allocated on demand

For application support:
- HugeTLB is used through its interface by application
- THP often transparent

For filesystem:
- `hugetlbfs` supported
- Not required for THP
# Inspection

To inspect the default page size:

```bash
getconf PAGE_SIZE # or PAGESIZE, which returns the page size in bytes
```

```result
4096
```

To inspect the HugePages:

```bash
grep -i huge /proc/meminfo
```

```result
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
FileHugePages:     22528 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
```

This shows the default-size HugePage is 2 MiB `2048 kB` and no actual usage of HugePages.

You can pre-allocate HugePages with `vm.nr_hugepages` tunable:

```bash
sysctl -w vm.nr_hugepages=32
```

Run the grep again to view HugePages stats:

```
grep -i huge /proc/meminfo
```

```result
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
FileHugePages:     22528 kB
HugePages_Total:      32
HugePages_Free:       32
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:           65536 kB
```

It shows the total HugePages is 32 and the the free is 32, so no application used it. The total HugePages size is `65536 kB` which is `65536 / 2048 = 32` pages.

Inspect the supported HugePage sizes:

```
ls -l /sys/kernel/mm/hugepages/
```

```result
hugepages-1048576kB
hugepages-2048kB
```

`1048576kB` is the Gigantic HugePages `1 GiB` and `2048kB` is the HugePages `2 MiB` are the supported in this system.

Check whether any `hugetlbfs` filesystem is mounted:

```
findmnt -t hugetlbfs
```

```result
/dev/hugepages hugetlbfs hugetlbfs rw,nosuid,nodev,relatime,seclabel,pagesize=2M
```

# Experiment

C program to allocate 2 MiB HugePage:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <unistd.h>

#define HUGE_PAGE_SIZE (2 * 1024 * 1024)

int main(void)
{
    size_t size = HUGE_PAGE_SIZE;

    // Allocate one 2 MB HugePage
    void *ptr = mmap(NULL, size,
                     PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS |
                     MAP_HUGETLB,
                     -1, 0);

    if (ptr == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    printf("HugePage allocated at: %p\n", ptr);

    // Use the memory
    memset(ptr, 0xAB, size);

    printf("First byte: 0x%02X\n",
           (unsigned char)((char *)ptr)[0]);

    // Sleep for 5 seconds
    sleep(5);

    // Release memory
    if (munmap(ptr, size) == -1) {
        perror("munmap");
        return 1;
    }

    printf("HugePage released successfully\n");

    return 0;
}
```

Compile it with:

```bash
gcc -O2 -Wall hugepages.c -o hugepages
```

Make sure you have at least one HugePages allocation:

```bash
sysctl -w vm.nr_hugepages=1
```

Watch `/proc/meminfo` and run:

```bash
watch -n 0.1 grep -i ^huge /proc/meminfo
./hugepages
```

Now before running, the HugePages stats was:

```
HugePages_Total:       1
HugePages_Free:        1
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:            2048 kB
```

After the program runs, it requests to allocate HugePage through `mmap` and reserved a one HugePage:

```bash
HugePages_Total:       1
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:            2048 kB
```

If we don't have enough HugePage, the program will fail and says:

```
mmap: Cannot allocate memory
```

# HugePages Interface

In Linux, there are four application-facing ways to access HugeTLB pages:
1. System V shared memory: `shmget(SHM_HUGETLB)`
2. hugetlbfs: `mmap()` on hugetlbfs file
3. Direct anonymous mapping: `mmap(MAP_HUGETLB)`
4. HugeTLB memory file descriptor

All these methods use HugeTLB subsystem, which manages the HugePages pools configured by the administrator.


## System V shm

System V shared memory explain in [[sysv_ipc_shm]].

The most important system calls are:
- `shmget()`: Create or locate a shared memory segment
- `shmat()`: Attach a shared memory segment to a process

Ordinary System V shared memory does not automatically imply HugePages. An application can specifically request HugeTLB-backed shared memory using the `SHM_HUGETLB` flag.

Conceptually:

```c
shmget(
	IPC_PRIVATE,
	64 * 1024 * 1024,
	IPC_CREAT | 0600 | SHM_HUGETLB
)
```

This requests 64 MiB shared memory segment backed by HugeTLB pages.

## HugeTLB filesystem `hugetlbfs`

It is a special pseudo-filesystem that lets application use HugeTLB-backed memory through file mappings. It acts as a filesystem interface to the HugeTLB memory management system.

Administrator can create a HugeTLB filesystem:

```bash
mkdir -p /hugetlb
mount -t hugetlbfs none /hugetlb
```

Then check its mounting point:

```bash
findmnt -t hugetlbfs
```

```result
/hugetlb       none      hugetlbfs rw,relatime,seclabel,pagesize=2M
```

Then an application can then open or create (`mmap`) a file such as:

```bash
/hugetlb/memory
```

Mount it persistently in `/etc/fstab`

```bash
none	/hugetlb	hugetlbfs	pagesize=2M 0 0
```

