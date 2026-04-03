# Memory Monitoring - Q&A

## Top 5 Commands

### Q: What are the top 5 commands to understand memory quickly?

```bash
free -h
vmstat 1
cat /proc/meminfo
top
ps aux --sort=-%mem | head
```

### Q: When do I use each one?

| Command | Use |
|---------|-----|
| `free -h` | Quick memory and swap snapshot |
| `vmstat 1` | Live memory, swap, I/O, CPU behavior |
| `cat /proc/meminfo` | Full kernel memory details |
| `top` | Live process-level memory usage |
| `ps aux --sort=-%mem | head` | Top memory-consuming processes |

## Table of Contents
1. [Top 5 Commands](#top-5-commands)
2. [Memory Fundamentals](#1-memory-fundamentals)
3. [How Linux Sees RAM](#2-how-linux-sees-ram)
4. [Page Tables & Memory Translation](#3-page-tables--memory-translation)
5. [Memory Zones & Kernel vs User Space](#4-memory-zones--kernel-vs-user-space)
6. [How Memory Gets Consumed](#5-how-memory-gets-consumed)
7. [The Page Cache (Why Cached Memory Exists)](#6-the-page-cache-why-cached-memory-exists)
8. [Understanding Memory Metrics](#7-understanding-memory-metrics)
9. [MemFree vs MemAvailable - Why Both Exist](#8-memfree-vs-memavailable---why-both-exist)
10. [Active, Inactive, Buffers, Cached Explained](#9-active-inactive-buffers-cached-explained)
11. [When Memory is Freed](#10-when-memory-is-freed)
12. [Swap & Virtual Memory](#11-swap--virtual-memory)
13. [The OOM Killer](#12-the-oom-killer)
14. [Monitoring Commands](#13-monitoring-commands)
15. [Quick Reference](#14-quick-reference)

---

## 1. Memory Fundamentals

### Q: What is RAM physically?

RAM (Random Access Memory) is just **electronic chips** - millions of tiny switches that can be on (1) or off (0). Each "locker" is 4KB (called a **page** - the fundamental unit of memory).

```
┌─────────────────────────────────────────────┐
│              Physical RAM                    │
│  [Locker 1] [Locker 2] [Locker 3] ... [Locker N] │
│   4KB       4KB        4KB              4KB   │
└─────────────────────────────────────────────┘
```

### Q: What is the page size on Linux?

```bash
getconf PAGE_SIZE
```

On most systems (x86_64, ARM64): **4096 bytes (4KB)**

| Architecture | Page Size |
|--------------|------------|
| x86_64 (Intel/AMD) | 4096 (4KB) |
| ARM64 (AWS Graviton, Mac M1+) | 4096 (4KB) |
| Older 32-bit x86 | 4096 (4KB) |
| Some embedded ARM | 4096 or 8192 |
| IA64 (Itanium) | 4096 or 65536 |

**Hugepages** are different - they're 2MB or 1GB instead of 4KB.

### Q: What is the difference between pages and HugePages?

| Type | Typical Size | Use |
|------|--------------|-----|
| **Normal page** | 4KB | Default Linux memory management |
| **HugePage** | 2MB or 1GB | Large memory workloads with lower overhead |

Linux always uses normal pages by default.

**Why HugePages exist:** Linux manages memory in pages. If an application uses a lot of RAM, `4KB` pages create a lot of page-table entries and address-translation overhead. HugePages make the unit bigger, so Linux tracks fewer pages and the CPU does less translation work.

HugePages are **optional** and are usually enabled only for memory-intensive workloads like:

- PostgreSQL / Oracle
- large JVMs
- high-performance networking

### Q: Do I need to install pages or HugePages?

**Normal pages**: No, they already exist by default.

**HugePages**: No installation, but you may need to **reserve/configure** them if an application needs them.

---

## 2. How Linux Sees RAM

### Q: Does Linux segregate RAM for each app?

**No.** Linux sees RAM as **one big shared pool**:

```
┌──────────────────────────────────────────────┐
│                   RAM (16GB)                  │
│   ┌────────┐ ┌────────┐ ┌────────┐           │
│   │ Process│ │ Process│ │ Kernel │           │
│   │   A    │ │   B    │ │        │           │
│   └────────┘ └────────┘ └────────┘           │
│          ↑         ↑          ↑               │
│          └─────────┴──────────┘               │
│              Linux manages this               │
└──────────────────────────────────────────────┘
```

Linux allocates chunks from wherever is free, dynamically.

---

## 3. Page Tables & Memory Translation

### Q: What is the translation problem?

Every process thinks it owns the **entire memory space**. But we have multiple processes sharing the same physical RAM.

**The solution**: Page tables.

### Simple Analogy: Hotel Room Keys

Imagine a hotel with 100 rooms (physical RAM). You have 500 guests (processes), each thinking they have their own room.

```
Guest #1 thinks they're in "Room 10" ──▶ Actually goes to ──▶ Physical Room #45
Guest #2 thinks they're in "Room 10" ──▶ Actually goes to ──▶ Physical Room #12
Guest #3 thinks they're in "Room 10" ──▶ Actually goes to ──▶ Physical Room #89
```

The **page table** is like the **reception desk** that translates "Room 10" (virtual address) → "Room #45" (physical address).

Each process has its own page table - its own "translation key" to the physical RAM.

### Q: What is demand paging?

Linux doesn't allocate memory when you ask for it. It allocates when you **first touch** it.

```c
// In your program
char *buffer = malloc(1000000);  // "I want 1MB!"
```

At this exact moment:
- Linux just notes: "This process might use 1MB someday"
- No actual RAM is used yet

**Real allocation happens on first access** (this is called a **page fault**):

```
1. Process tries to READ the buffer
2. CPU notices: "Hey, this address has no real RAM!"
3. CPU asks kernel: "Can I have RAM for this?"
4. Kernel: "Sure, here's a page"
5. Translation table updated: virtual → physical
6. Process continues, now has real RAM
```

This is **demand paging** - memory is given "on demand."

### Q: What is an offset in simple words?

Think of memory as being divided into equal-size pages, usually `4KB` each.

An address is split into:
- **page number** = which page/block
- **offset** = the exact position inside that page

Example with page size `4096` bytes:
- address `5000`
- page number = `5000 / 4096`
- offset = `5000 % 4096 = 904`

So the **offset** just means: "where inside this page is the data?"

### Q: What is a page fault in simple words?

A **page fault** happens when a program tries to use a page that is not ready in RAM yet.

Simple flow:
1. Program touches memory
2. CPU sees the page is not currently mapped the way it expects
3. CPU pauses the program and asks the kernel for help
4. Kernel sets up the mapping or loads the page
5. Program continues

Most page faults are **normal**. They happen because Linux uses demand paging and only prepares memory when it is first used.

### Q: Is a page fault always an error?

No.

- **Normal page fault**: Linux brings in or maps the page, then the program continues
- **Invalid page access**: the program touched bad memory and may crash with a segmentation fault

---

## 4. Memory Zones & Kernel vs User Space

### Q: What is Kernel Space vs User Space?

Linux splits memory into two worlds:

| Space | Who Uses It | Can Access | Example |
|-------|-------------|------------|---------|
| **Kernel Space** | Linux itself | Everything | Managing processes, drivers |
| **User Space** | Your applications | Only their own | Chrome, Python, MySQL |

```
┌────────────────────────────────────┐
│      Kernel Space (privileged)    │  ← Linux OS lives here
│                                    │
└────────────────────────────────────┘  ← Invisible barrier
┌────────────────────────────────────┐
│        User Space (restricted)    │
│  ┌─────────┐ ┌─────────┐          │
│  │ Process │ │ Process │          │
│  │    A    │ │    B    │          │
│  └─────────┘ └─────────┘          │
└────────────────────────────────────┘
```

**Why separate them?** Security. A bug in Chrome can't read what MySQL is doing.

### Q: What are Memory Zones?

Even though RAM is one pool, Linux divides it into "zones" for efficiency:

```
┌─────────────────────────────────────────────────────┐
│                      RAM                            │
├──────────────┬──────────────┬──────────────────────┤
│    ZONE_DMA  │  ZONE_NORMAL │    ZONE_HIGHMEM       │
│  (0-16MB)    │  (16MB-896MB) │   (896MB+)            │
│              │               │                      │
│ Legacy       │ Most memory   │ Can't be directly    │
│ devices      │ operations    │ mapped by kernel      │
└──────────────┴──────────────┴──────────────────────┘
```

This is an internal optimization - you don't need to worry about it.

---

## 5. How Memory Gets Consumed

### Q: What consumes memory on a Linux system?

| Consumer | What It Uses | Example |
|----------|--------------|---------|
| **Application code** | Text segment | Your program's instructions |
| **Heap** | Dynamic allocations | `malloc()`, `new` |
| **Stack** | Function calls, local vars | Function parameters, recursion |
| **Memory-mapped files** | Shared libraries, mmap'd files | `libc.so`, database files |
| **Page cache** | Disk cache | Recently read files |
| **Slab allocator** | Kernel objects | Network buffers, inode cache |

### Q: What are the different memory segments in an app?

When you run a program, Linux reserves space for different things:

1. **Code Segment (Text)**
   ```
   Your program: "Hello World"
   ┌─────────────────────┐
   │  Machine code       │  ← Read-only, fixed size
   │  (the actual logic) │
   └─────────────────────┘
   ```

2. **Heap (Dynamic Memory)**
   ```
   malloc(100MB) ──▶ Kernel reserves 100MB here
                    (but only uses real RAM when touched)
   ```

3. **Stack (Function Calls)**
   ```
   void foo() {
       int x = 5;    ← x lives here, on stack
   }
   ```
   Grows/shrinks as functions call/return.

---

## 6. The Page Cache (Why Cached Memory Exists)

### Q: What is cached memory?

**Cached = copies of files Linux saved in RAM**

```
┌─────────────────────────────────────────────┐
│                  RAM                        │
│  ┌─────────────────────────────────────┐   │
│  │   Cached (4.1GB)                    │   │
│  │                                      │   │
│  │   /etc/passwd ──▶ [copy in RAM]    │   │
│  │   /var/log/syslog ──▶ [copy]       │   │
│  │   library.so ──▶ [copy]            │   │
│  │   app data ──▶ [copy]              │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

**Who caches?** The **Linux kernel** (not your app).

**Why?** Speed.

| Scenario | Without Cache | With Cache |
|----------|---------------|------------|
| Read file | 10ms (from disk) | 0.01ms (from RAM) |
| 1000 reads | 10 seconds | Instant |

### Q: Why can cached be freed so easily?

**Simple answer: It's just a COPY. The original file is still on disk.**

```
RAM Cache                    Disk
┌──────────────┐           ┌──────────┐
│ copy of file │   ←───    │ original │
│              │   (mirror)│  file    │
└──────────────┘           └──────────┘
```

Linux: "I'll throw away my copy. The original is safe on disk."

**If app needs RAM → cache dropped instantly → no data loss**

### Analogy: Your Desk

```
┌────────────────────────────────────────────┐
│  Your Desk (RAM)                          │
│                                            │
│  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │ Papers   │  │ Reference│  │ Notes   │ │
│  │ you need │  │ books    │  │         │ │
│  │ (apps)   │  │ (cache)  │  │ (free)  │ │
│  └──────────┘  └──────────┘  └─────────┘ │
└────────────────────────────────────────────┘

- Papers you need  = Apps using memory
- Reference books  = Cached files (can put away)
- Empty space      = Free memory
```

When you need more space for papers (apps), you put away reference books (cache). No problem - you can get the books from the shelf (disk) anytime.

---

## 7. Understanding Memory Metrics

### Q: What does /proc/meminfo show?

```
MemTotal:        8212920 kB    ← Total RAM in system
MemFree:         2227824 kB    ← Completely empty lockers
MemAvailable:    6837292 kB    ← REAL free for new apps
Buffers:          452480 kB    ← Filesystem metadata
Cached:          4113112 kB    ← Files cached in RAM
SwapCached:            0 kB    ← Pages in swap also in RAM
Active:          1422316 kB    ← Recently used memory
Inactive:        3911664 kB    → Not recently used (reclaimable)
Active(anon):      75536 kB    ← App heap/stack (used)
Inactive(anon):   843160 kB    → App memory (not recently used)
Active(file):    1346780 kB    ← Files currently being read
Inactive(file):  3068504 kB    → Cached files (not recently used)
```

### Q: What do the categories mean?

| Category | Description | Can Be Freed? |
|----------|-------------|----------------|
| **Active(file)** | Files currently being used by apps | Rarely |
| **Active(anon)** | App heap/stack (malloc memory being used) | Only if app exits |
| **Inactive(anon)** | App memory not recently used | YES → to swap or free |
| **Inactive(file)** | Cached files not recently read | YES → instantly |
| **Cached** | All file cache combined | YES → instantly |
| **Buffers** | Filesystem block metadata | YES → instantly |
| **MemFree** | Empty RAM | N/A (already free) |

---

## 8. MemFree vs MemAvailable - Why Both Exist

### Q: Why show two "free" memory numbers?

**Historical accident + gradual fix:**

| Metric | When Added | Purpose |
|--------|-----------|---------|
| **MemFree** | Old (1990s) | "Empty" pages with nothing in them |
| **MemAvailable** | Linux 3.14 (2014) | "Actually usable" memory |

**What happened:**

```
1990s:  Linux only had MemFree
        Everyone panicked: "Only 2MB free! System dying!"
        Reality: 200MB in cache, but "free" meant empty pages
        
2014:   Linux added MemAvailable
        "Okay, we show you the real number now"
```

### Q: Why not use ALL free memory for cache?

Linux DOES use almost ALL free memory for cache:

```
              total    used    free   buff/cache   available
Mem:          7.8Gi   1.4Gi   0.5Gi    4.5Gi        6.8Gi
                              ↑
                          This small amount is kept
                          as "emergency reserve"
```

**Why keep even 0.5GB free?**

| Reason | Purpose |
|--------|---------|
| **Burst capacity** | New processes starting need memory FAST |
| **Kernel itself** | Kernel code needs some free pages to work |
| **Atomic allocations** | Some operations need memory immediately |

**Analogy**: Your bank account:

```
Total: $10,000

- You keep:     $500   ← "free" (for emergencies)
- Investments: $9,500  ← "cache" (working for you)

You COULD invest all $10,000, but then gas/food = problem.
```

### Q: Is MemAvailable = MemFree + Cached?

**No. It's more complex.** The formula (simplified):

```
MemAvailable = MemFree - low_water_mark + (page cache - min_page_cache)
```

Linux keeps a **safety margin** so it can always allocate memory instantly when needed.

**Example from user's system:**

```
MemFree:       2.26 GB
Cached:        4.11 GB
─────────────────────────────────
Simple sum:    6.37 GB

Actual MemAvailable:  6.87 GB  ← DIFFERENT!
```

The difference comes from other reclaimable things like:
- **Buffers**: filesystem metadata
- **Inactive(file)**: files not recently used

---

## 9. Active, Inactive, Buffers, Cached Explained

### Q: What is the difference between Active and Inactive?

- **Active**: Memory that was recently accessed. Kernel assumes it's "hot" and keeps it in RAM.
- **Inactive**: Memory that hasn't been used recently. Kernel considers it "cold" and is okay with evicting it.

### Q: What are Buffers vs Cached?

| Term | What It Stores |
|------|----------------|
| **Buffers** | Raw filesystem block metadata |
| **Cached** | File content (what you read from disk) |

Both are reclaimable, but serve different purposes.

### Q: What is the relationship between these categories?

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TOTAL RAM: 7.83 GB                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │           APP MEMORY (what programs use)                     │   │
│  │                                                              │   │
│  │   Active(file): 1.28 GB  ── Currently used by apps          │   │
│  │   Active(anon): 0.07 GB  ── Anonymous (heap/stack)          │   │
│  │   Inactive(anon): 0.79 GB ── Not recently used, can reclaim  │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │           KERNEL CACHE (disk files in RAM)                  │   │
│  │                                                              │   │
│  │   Cached: 3.92 GB     ── Files Linux saved for speed        │   │
│  │   Buffers: 0.43 GB    ── Filesystem metadata                 │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │           FREE (truly empty)                                 │   │
│  │   MemFree: 2.13 GB     ── Empty lockers                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 10. When Memory is Freed

### Q: When is memory actually freed?

| Scenario | What Happens |
|----------|--------------|
| **Process exits** | All pages returned to free pool |
| **malloc + free** | Returns to process heap (not to Linux!) |
| **Cache eviction** | Page cache dropped when RAM needed |
| **OOM Killer** | Process terminated when RAM exhausted |

**Critical**: `free()` doesn't return memory to OS - it goes back to the process's heap for reuse. Only `munmap()` or process exit returns it to the system.

### Q: Why does free() not return memory to Linux?

When you call `free()`:
1. Memory goes back to your process's internal heap
2. Linux still thinks your process owns that memory
3. Your process can reuse it without asking Linux again
4. Only when your process exits does Linux get the memory back

This is called **memory pooling** - apps keep a "private cache" of freed memory.

---

## 11. Swap & Virtual Memory

### Q: Why does swap exist?

**Early computers had a problem:**

```
┌─────────────────────────────────────────────┐
│              Early System                    │
│                                              │
│  RAM: 4MB                                    │
│  App1: needs 2MB                             │
│  App2: needs 2MB                             │
│  App3: needs 2MB   ──▶ "Out of Memory!"      │
│                                              │
│  What do we do? Throw away App1?             │
└─────────────────────────────────────────────┘
```

**Solution**: Use disk as "backup memory"

```
┌─────────────────────────────────────────────┐
│              Add Swap (Disk)                  │
│                                              │
│  RAM: 4MB    (fast, expensive)               │
│  Swap: 4GB   (slow, cheap)                   │
│                                              │
│  "Move inactive stuff to disk"               │
│  Free up RAM for active apps                 │
└─────────────────────────────────────────────┘
```

### Analogy: Your Desk vs Filing Cabinet

```
┌────────────────────┐     ┌────────────────────┐
│    Your Desk       │     │  Filing Cabinet    │
│    (RAM)           │     │    (Swap/Disk)     │
│                    │     │                    │
│  Quick access      │     │  Slow access       │
│  Limited space     │     │  Lots of space     │
│                    │     │                    │
│  Working papers    │     │  Old documents     │
└────────────────────┘     └────────────────────┘

When desk is full:
- Put rarely-used papers in cabinet
- Keep frequently-used on desk

Linux does the same with RAM ↔ Swap
```

### Q: How does swap actually work?

**Step 1: Page becomes inactive**

```
App has memory at 0x00400000 (some address)
Linux: "You haven't used this in a while"
```

**Step 2: Move to swap**

```
RAM: 0x00400000 ──▶ Disk: Swap offset 0x1000
      (page contents)    (swapped out)
      
Page table updated:
0x00400000 ──▶ "Not in RAM, look in swap at offset 0x1000"
```

**Step 3: Access from swap (Slow!)**

```
App reads from 0x00400000
CPU: "Page fault! Not in RAM!"
Kernel: "Okay okay, let me fetch from disk..."
       (reads from swap, slow!)
       (puts back in RAM)
```

### Q: Does more swap prevent OOM?

**No.** It just delays the inevitable. If you're truly out of memory, swap just means "I killed your process more slowly."

| Scenario | Result |
|----------|--------|
| RAM full + no swap | OOM killer kills process |
| RAM full + swap | Everything runs painfully slow |

### Q: Is using swap discouraged on modern systems?

Not exactly.

Modern guidance:
- swap is **not** a replacement for enough RAM
- a **small amount of swap** can help absorb short memory spikes
- **heavy swap usage** usually means poor performance
- relying on swap as your main memory strategy is discouraged

### Q: Why do some teams avoid swap?

- disk is much slower than RAM
- latency-sensitive apps can become very slow when pages are swapped out
- heavy swap can hide a memory problem until the whole system becomes sluggish

### Q: Why do some teams still keep some swap?

- it gives the kernel breathing room during brief pressure spikes
- it can reduce the chance of immediate OOM in some cases
- it can hold rarely used pages while active pages stay in RAM

### Q: What is the practical rule of thumb?

- **No swap at all**: faster OOM when memory spikes
- **Small, controlled swap**: often useful
- **Constant swapping**: a sign the system needs more RAM or tuning

So the right conclusion is:

**Using swap is not discouraged. Depending heavily on swap is discouraged.**

### Q: How is swap allocated? Is it manual or automated?

**Two ways:**

| Method | Who Does It | When |
|--------|-------------|------|
| **Manual** | Sysadmin creates | At setup time |
| **Automatic** | Cloud provider | When you provision a VM |

#### Method 1: Manual (Traditional)

```bash
# Step 1: Create a file (or use a partition)
sudo fallocate -l 2G /swapfile

# Step 2: Set correct permissions (must be 600!)
sudo chmod 600 /swapfile

# Step 3: Format as swap
sudo mkswap /swapfile

# Step 4: Enable it
sudo swapon /swapfile

# Step 5: Make it permanent (add to /etc/fstab)
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

```bash
# Verify
swapon --show
free -h
```

#### Method 2: Cloud/Automated (Modern)

**On AWS, GCP, Azure**: Swap is often **NOT configured by default**

| Cloud | Default Swap | How to Add |
|-------|--------------|------------|
| AWS EC2 | None | Use swap file |
| GCP | None | Use swap file |
| Azure | None | Use swap file |

**Why no default?** Cloud providers assume you provision enough RAM.

#### Method 3: LVM Swap (Enterprise)

```bash
# Create LVM logical volume for swap
sudo lvcreate -L 2G -n swap_vg vg_name

# Format as swap
sudo mkswap /dev/vg_name/swap_vg

# Enable
sudo swapon /dev/vg_name/swap_vg
```

### Q: Can you have multiple swap areas?

Yes:

```bash
# Add multiple swap files
sudo swapon /swapfile1
sudo swapon /swapfile2

# Or add multiple partitions
swapon /dev/sda2
swapon /dev/sdb1

# See all
swapon --show
```

**Priority matters** (higher = used first):

```bash
sudo swapon --priority 100 /swapfile
```

### Q: Is swap permanent?

| Storage Type | Survives Reboot? |
|--------------|------------------|
| `/etc/fstab` entry | Yes |
| `swapon` command only | No (gone after reboot) |

```bash
# Make permanent - add to /etc/fstab
# Format: <device>  <mount point>  <type>  <options>  <dump>  <pass>
/swapfile   none    swap    sw      0         0
```

### Q: What is `/etc/fstab`?

`/etc/fstab` is the **filesystem table**.

It tells Linux what filesystems or swap areas to enable automatically at boot.

Example:

```fstab
/swapfile   none    swap    sw      0         0
```

Meaning: "Enable this swap file automatically after reboot."

### Q: When should I use swap?

| Scenario | Recommendation |
|----------|-----------------|
| Desktop with hibernation | Yes |
| Server with < 2GB RAM | Yes |
| Server with 8GB+ RAM (dev/test) | Optional |
| Server with 8GB+ RAM (production) | Usually NO |
| Containers | NO (disable swap in Kubernetes) |

### Q: What is the rule of thumb for swap size?

| RAM Size | Swap Recommendation |
|----------|---------------------|
| < 2GB | 2x RAM |
| 2-8GB | 1x RAM |
| 8-64GB | 0.5x RAM or none |
| 64GB+ | Usually none |

### Q: Your system - check if you have swap

```bash
free -h
swapon --show
```

### Q: Why modern systems often disable swap?

| Era | Swap Usage |
|-----|------------|
| 1990s | Essential (RAM was 4-16MB) |
| 2000s | Common (RAM was 512MB-2GB) |
| 2020s | Often disabled (RAM is 8-64GB) |

**Why disable?**

1. **Swap is SLOW** - disk is 1000x slower than RAM
2. **Enough RAM** - modern servers have plenty
3. **Better alternatives** - OOM killer is more honest

---

## 12. The OOM Killer

### Q: What happens when RAM is truly exhausted?

When RAM + swap exhausted:
1. Kernel invokes OOM killer
2. Selects "best" process to sacrifice (not just largest)
3. Kills it, frees its pages
4. System survives, one process dies

### Q: How does OOM killer decide what to kill?

It uses a scoring algorithm considering:
- Process memory usage (bigger = more likely to die)
- Process uptime (newer = more likely to die)
- Nice value (lower priority = more likely to die)
- OOM score adj (manual tuning)

### Q: How to check if OOM killed something?

```bash
dmesg | grep -i "out of memory"
journalctl -b | grep -i oom
```

---

## 13. Monitoring Commands

### Q: Quick memory check?

```bash
# Simplest view
free -h

# Detailed view
cat /proc/meminfo

# Per-process memory
ps aux --sort=-%mem | head

# Watch changes
vmstat 1
```

### Q: What to check in production?

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| MemAvailable | > 1GB | < 500MB | < 100MB |
| Swap used | Low and stable | Growing steadily | Heavy active use |

### Q: Is there a healthy memory utilization percentage?

There is no single healthy RAM percentage like CPU.

Linux uses spare RAM for cache, so high memory usage is often normal.

- `80%` used can be fine
- `90%` used can still be fine
- high usage becomes a problem when available memory gets low or the system starts swapping heavily

### Q: When should I worry about RAM?

Worry when several of these happen together:

- `MemAvailable` keeps falling and stays low
- swap usage is growing
- `vmstat` shows swap-in or swap-out activity (`si` / `so`)
- applications slow down
- the kernel logs OOM kills

### Q: What is the practical rule of thumb?

| Used RAM % | Meaning |
|------------|---------|
| `< 70%` | Usually comfortable |
| `70-85%` | Usually still fine |
| `85-95%` | Watch closely |
| `> 95%` | Risky if available memory is low or swap is active |

The key idea:

**Healthy memory is not about a fixed percentage. It is healthy when the server still has enough available memory and is not actively swapping or hitting OOM.**

### Q: How to find which process uses most memory?

```bash
ps aux --sort=-%mem | head -10
# Or
top (press M to sort by memory)
```

---

## 14. Quick Reference

### Key Rules for Memory Monitoring

```
┌─────────────────────────────────────────────────────┐
│              LOOK AT THIS:                          │
│                                                     │
│   available > 1GB?    →  Healthy                   │
│   available < 500MB?  →  Warning                   │
│   available < 100MB?  →  Trouble!                  │
│                                                     │
│   swap low/stable?   →  Usually okay               │
│   swap growing fast? →  Memory pressure            │
└─────────────────────────────────────────────────────┘
```

### Summary Commands

```bash
# Quick health check
free -h

# What's using memory per process
ps aux --sort=-%mem | head

# Detailed memory info
cat /proc/meminfo

# Watch memory changes
vmstat 1

# Check for OOM kills
dmesg | grep -i "out of memory"
```

### The Only Numbers That Matter

| Metric | Why It Matters |
|--------|----------------|
| **MemAvailable** | Real free memory for new apps |
| **Swap trend** | Stable is okay; growing or active swapping is a warning |

**Everything else** (MemFree, Cached, Buffers) - Linux handles automatically. Don't worry about it.

---

## 15. Kubernetes Swap Notes

### Q: Are pages just blocks of memory?

Yes. A **page** is a fixed-size block of memory, usually **4KB** on Linux.

A process does **not** live in RAM as one giant chunk. Its memory is split into many pages:

- Some pages are **hot** (used often)
- Some pages are **cold** (not used recently)

### Q: Does Linux swap out whole processes?

**No.** Linux usually swaps out **pages**, not entire processes.

That means one process can be partly in RAM and partly in swap:

```text
Process A
├── Page 1  -> RAM
├── Page 2  -> RAM
├── Page 3  -> Swap
└── Page 4  -> Swap
```

### Q: Why is swap risky on Kubernetes nodes?

Because Kubernetes wants memory pressure to be **predictable**.

If swap is enabled:

1. A process can look inactive, so some of its pages are moved to swap
2. Later, that process becomes active again
3. Now Linux must read those pages back from disk
4. That delay can slow down important node components

This is especially bad for:

- `kubelet`
- container runtime (`containerd`, `CRI-O`)
- DNS
- CNI/networking agents

These components may be quiet for a while, but when needed, they must wake up **fast**.

### Q: What is the noisy-neighbor risk with swap?

One pod can consume too much memory and trigger reclaim/swap pressure for the **whole node**.

Result:

- Pod A uses too much memory
- Linux starts reclaiming/swapping pages
- Pod B or system daemons lose hot pages from RAM
- Pod B becomes slow, even though Pod B did nothing wrong

This is called a **noisy neighbor** problem.

### Q: What is the short rule for Kubernetes?

**Swap works technically, but it makes node behavior less deterministic.**

Kubernetes generally prefers:

- fast detection of memory pressure
- clear eviction behavior
- predictable OOM behavior

instead of:

- hidden slowdown
- disk thrashing
- delayed recovery

### Q: Is high memory usage always a problem?

**No. High memory usage is not the same as memory pressure.**

It can be completely healthy if:

- the workload is expected to use memory
- `MemAvailable` is still healthy
- swap is not actively being used
- there are no OOM kills or latency issues

Examples:

- PostgreSQL using shared buffers
- Java using a large heap
- Linux using RAM for page cache

The real problem is:

- low `MemAvailable`
- active swapping
- OOM kills
- application slowdown
