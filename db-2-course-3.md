# SQL Snippets



> [2024/02/10] | [00:37:00]
> DB2 — Chapter 02 — Video #04 — HDD/SSD/RAM access times, random vs. sequential reading

---

### Unary Table Storage

#### Rotating Magnetic Hard Disk Drives (HDDs)

Steadily rotating patters and read / write heads of a HDD



#### HDDs: Tracks, Sectors, Blocks

1. Seek: Stepper motor positions array of R / W heads over wanted track
2. Rotate: Wait for wanted sector of blocks to rotate under R / W heads
3. Transfer: Activate one head to read / write block data



#### HDDs: Access Time

A HDD design that involves motors, mechanical parts, and thus inertia has severe implications on the access time `t` needed to read / write one block.

- Amortize seek time and rotational delay by transferring one block at a time (random block access)
- Transfer a sequence of adjacent blocks (sequential block access)



#### Solid State Disk Drives (SSDs)

SSDs rely on non-volatile flash memory and contain no moving / electro-mechanical parts:

- Non-volatility (battery-powered DRAM or NAND memory cells) ensures data persistence even on power outage.
- No seek time, no rotational delay, no motor spin-up time, no R / W head array jitter.
- Admits low-latency random read access to large data blocks (typical: 128 kB), however, slow random writes.

Groups of data blocks need to be erased, then can be written again. Memory cells wear out after 10^4 to 10^5 write cycles - SSDs use wear-leveling to spread data evenly across the device memory.



#### SSDs: Still a Disk? Already like RAM?

SSD transfer speed test (write 4 GB of zeros):

```bash
cd /tmp
# This command - dd (disk dump) will count the time of transferred bytes
time dd if=/dev/zero of=bitbucket bs=1024 count=4096
```

DRAM transfer speed test (write 4 GB of 64-bit values).

[NOTE] You will need to write a little C program for this case

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <sys/time.h>
#include <stdint.h>

#define MICROSECS(t) (1000000 * (t).tv_sec + (t).tv_usec)

/* overall size of memory to scan (4 GB) */
#define MEMSIZE (4UL * 1024 * 1024 * 1024)

/* size of scan area (8 MB, exceeds L1-L3 cache sizes) */
#define SCANSIZE(8 * 1024 * 1024)

/* scan the memory, do pseudo work */
void scan(int64_t *mem)
{
    for (size_t loop = 0; loop < MEMSIZE / SCANSIZE; loop += 1) {
        for (size_t i = 0; i < SCANSIZE / sizeof(int64_t); i += 1) {
            mem[i] = 42;
        }
    }
}

int main()
{
    int64_t *area;
    
    struct timeval t0, t1;
    unsigned long duration;
    
    /* allocate sccan area */
    area = (int64_t*)malloc(SCANSIZE);
    assert(area);
    
    gettimeofday(&t0, NULL);
    scan(area);
    gettimeofday(&t1, NULL);
    
    duration = MICROSECS(t1) - MICROSECS(t0);
    printf("time: %1ums\n", duration)
}
```



1. Allocate memory area of 8 MB
2. Repeatedly scan the area, writing 64-bit by 64-bit

```bash
cc -Wall -O2 transfer.c -o transfer
./transfer
```

Still faster: use SIMD instructions (r / w up to 256 bits) and multiple CPU cores (but: bus bandwidth is limited)

































