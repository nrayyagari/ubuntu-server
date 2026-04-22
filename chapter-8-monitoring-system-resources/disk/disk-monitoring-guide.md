# Disk & Storage Monitoring - Q&A

## Table of Contents
1. [System Resources Overview](#1-system-resources-overview)
2. [Storage Types: HDD vs SSD vs NVMe](#2-storage-types-hdd-vs-ssd-vs-nvme)
3. [IOPS vs Throughput](#3-iops-vs-throughput)
4. [Disk Problems in Distributed Systems](#4-disk-problems-in-distributed-systems)
5. [Inodes Explained](#5-inodes-explained)
6. [Filesystem vs Disk/Volume](#6-filesystem-vs-diskvolume)
7. [Log Rotation](#7-log-rotation)
8. [Monitoring Commands](#8-monitoring-commands)
9. [ncdu vs df vs du](#9-ncdu-vs-df-vs-du)
10. [SOP: Disk Troubleshooting](#10-sop-disk-troubleshooting)
11. [StatefulSets and Storage Challenges](#11-statefulsets-and-storage-challenges)

---

## 1. System Resources Overview

### Q: What are the main system resources to manage for distributed systems?

**Priority Order (1 = Most Critical):**

| Priority | Resource | Why It Matters |
|----------|----------|----------------|
| 1 | Network | Distributed systems depend on network; partitions cause failures |
| 2 | Memory | OOM kills, evictions, cascading failures |
| 3 | Disk/Storage | Database bottlenecks, write failures |
| 4 | CPU | Throttling causes latency spikes |
| 5 | Connections | FD exhaustion, DB connection limits |

### Q: What are Google's Golden Signals for monitoring?

The four signals every SRE should monitor:
- **Latency** - Response time (user experience)
- **Traffic** - Request volume
- **Errors** - Failure rates
- **Saturation** - Resource exhaustion

---

## 2. Storage Types: HDD vs SSD vs NVMe

### Q: What is the progression of storage technology?

| Type | Technology | Speed | Use Case |
|------|------------|-------|----------|
| **HDD** | Magnetic spinning platters | Slowest | Cold storage, bulk |
| **SSD** | NAND flash chips | Fast | Databases, apps |
| **NVMe** | PCIe flash | Fastest | High-performance |

### Q: Why is NVMe faster than SSD?

| Factor | SSD (SATA) | NVMe |
|--------|------------|------|
| Interface | 6 Gbps (bandwidth limit) | 32 Gbps (PCIe 3.0) |
| Queue Depth | ~32 commands | ~64k commands |
| Latency | ~100μs | ~20μs |
| Max IOPS | ~100k | ~1M+ |

**Key insight**: SSD still uses SATA (designed for HDDs). NVMe goes direct to PCIe bus - no translation layer.

### Q: How do cloud providers handle storage types?

| Cloud | HDD-like | SSD-like | Notes |
|-------|----------|----------|-------|
| AWS | st1, sc1 | gp2, gp3, io1 | gp3 = provisionable IOPS |
| Azure | Standard | Premium SSD | Managed service abstracts hardware |
| GCP | Standard | SSD, Extreme PD | You set capacity + IOPS |

### Q: Why do people say "disk" in cloud if storage is virtual?

Clouds expose **logical block devices** even though hardware is abstracted.

| Cloud | Common Term | What You Actually Manage |
|-------|-------------|--------------------------|
| AWS | EBS **Volume** | Attached block volume |
| Azure | Managed **Disk** | Attached managed disk |
| GCP | Persistent **Disk** / Hyperdisk | Attached persistent disk |

**Practical point**: "disk" is common language; "volume" is often the precise API term (especially in AWS EBS).

### Q: When do I need NVMe vs SSD?

- **Boot volumes**: Cheap HDD fine
- **Data/stateful apps**: SSD required
- **etcd (Kubernetes)**: Always SSD - slow disk kills cluster
- **Caches/Redis**: NVMe if possible
- **Logs/Kafka**: HDD can work (sequential writes)

### Q: How does DynamoDB (and similar managed services) adjust IOPS?

**What's happening**: AWS manages the underlying hardware. You reserve guaranteed IOPS from their distributed infrastructure.

| Aspect | Raw Storage | Managed Service (DynamoDB) |
|--------|-------------|---------------------------|
| Control | Full (you buy hardware) | Abstracted |
| IOPS | Hardware limit | Provisioned capacity |
| Scaling | Add/replace hardware | Just pay more |
| Cost | CapEx (buy disks) | OpEx (pay for usage) |

**Key difference**: You're buying **predictable performance**, not specific hardware.

---

## 3. IOPS vs Throughput

### Q: Are IOPS and throughput the same?

**No.** They measure different things:

| Term | Measures | Unit |
|------|----------|------|
| **IOPS** | Operations per second | operations/sec |
| **Throughput** | Data transferred per second | MB/s |

**Analogy - Highway:**

| Metric | Highway Equivalent |
|--------|-------------------|
| IOPS | Vehicles per minute |
| Throughput | Total cargo (tons) moved per minute |

### Q: Can you have high IOPS but low throughput?

Yes. Example: many tiny database queries.

| Scenario | IOPS | Throughput |
|----------|------|-------------|
| Small random reads (DB queries) | High | Low |
| Large sequential writes (logs) | Low | High |

### Q: Is throughput the same as bandwidth?

**Yes**, in the context of data transfer.

- **Throughput/Bandwidth** = how much data moves (MB/s, GB/s)
- **IOPS** = how many operations

### Q: Why do we monitor throughput if SSD solves disk speed?

Two constraints even with SSD:
1. **Disk throughput** - upgrade SSD → solved
2. **Network throughput** - shared infrastructure, can't easily upgrade

**Network throughput is critical** because:
- Shared/metered bandwidth
- Cross-AZ/region traffic costs money
- Can't just "add faster network" like local SSD

---

## 4. Disk Problems in Distributed Systems

### Q: How does disk become problematic in distributed systems?

| Scenario | Symptom | Cause |
|----------|---------|-------|
| Pod stuck in CrashLoopBackOff | App can't start | Volume full |
| Database pod evictions | Pods killed | Node disk pressure |
| etcd unhealthy | API down | Slow disk I/O |
| Log aggregation fails | Missing logs | Loki/Prometheus disk full |
| Build pipeline stuck | CI hangs | Build artifacts filling disk |

### Q: What are real production incidents related to disk?

**Grafana Labs (2020)**: GCP persistent disk issue → 23-hour outage
- Disk became read-only → couldn't write → cascade failure

**Kubernetes kubelet issues**: Network blip → volume unmounts → pods crash

### Q: What are Kubelet eviction thresholds?

```yaml
--eviction-hard=memory.available<100Mi
--eviction-hard=nodefs.available<10%   # disk on node
--eviction-hard=imagefs.available<10%  # disk for images
```

### Q: Are those values always the Kubernetes defaults?

Not always. Common Linux defaults are typically:

```yaml
memory.available<100Mi
nodefs.available<10%
imagefs.available<15%
nodefs.inodesFree<5%
imagefs.inodesFree<5%
```

If you set custom values (for example `imagefs.available<10%`), those are cluster/node-specific tuning choices.

### Q: Why do disk problems cause cascading failures?

1. Node hits disk pressure
2. Kubelet evicts pods
3. Pods reschedule to other nodes
4. Other nodes get overloaded
5. Cluster becomes unstable

### Q: Where are inode eviction thresholds watched in Kubernetes?

On each node by **kubelet** (node-local eviction manager), not by the scheduler.

Kubelet tracks signals like:
- `nodefs.inodesFree`
- `imagefs.inodesFree`

When thresholds are crossed, kubelet can mark disk pressure and evict pods.

---

## 5. Inodes Explained

### Q: What is an inode?

An **inode** = one file/folder entry in filesystem.

### Q: What are the two limits for storage?

| Limit | Measures | Command |
|-------|----------|---------|
| **Disk space** | Total bytes | `df -h` |
| **Inodes** | Total file count | `df -i` |

### Q: Can you run out of inodes before disk space?

**Yes!** This is a real problem:

```
df -h    → 50% used (plenty of space)
df -i    → 100% used (CAN'T create new files!)
```

### Q: What files have inodes?

Every file and folder has an inode:
- Regular files (`*.log`, `*.yaml`)
- Directories
- Symbolic links
- Device files (`/dev/sda1`)
- Sockets and pipes

### Q: How to inspect inodes?

```bash
# See inode number for a file
ls -li file.txt

# See filesystem inode usage
df -i

# Find files by inode
find / -inum 123456

# Which directory has most inodes
find / -xdev -printf '%i\n' | sort | uniq -c | sort -rn | head
```

### Q: How do I read `ls -li` output fields?

Example:
```bash
2112 -rw-r--r-- 1 laborant laborant 17733 Apr 22 02:27 AGENTS.md
```

Field order:
1. inode number (`2112`)
2. permissions/type (`-rw-r--r--`)
3. hard link count (`1`)
4. owner (`laborant`)
5. group (`laborant`)
6. size in bytes (`17733`)
7. timestamp (`Apr 22 02:27`)
8. filename (`AGENTS.md`)

### Q: Why does a symlink show a different inode than its target?

Because the symlink itself is its own filesystem object with its own inode.

Example:
```bash
48772 lrwxrwxrwx ... CLAUDE.md -> /home/laborant/AGENTS.md
```

`48772` is inode of `CLAUDE.md` (the link object), not the target file inode.

Use this to show target details via the link path:
```bash
ls -liL CLAUDE.md
```

### Q: What causes inode exhaustion?

| Scenario | Why |
|----------|-----|
| Millions of unrotated logs | Logs pile up |
| Temp files not deleted | `/tmp` fills |
| Email queue | Millions of queued messages |
| Docker images not pruned | Too many layers |

### Q: How many inodes does my server have?

From your server:
```
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/root        2.5M  189K  2.3M    8% /
```

- 2.5M inodes = max ~2.5 million files
- 8% used = ~189K files currently
- Plenty of headroom unless something goes wrong

---

## 6. Filesystem vs Disk/Volume

### Q: Why does `df -h` show "Filesystem" instead of "Disk"?

Historical/technical terminology:

| Term | Meaning |
|------|---------|
| **Filesystem** | How data is organized (structure), not the physical disk |
| **Disk/Volume** | The physical or virtual storage device |

`df` shows **mounted filesystems**, which can be:
- Physical partitions (`/dev/sda1`)
- Virtual/network drives (NFS)
- Temporary filesystems (`tmpfs`) - in RAM!
- Cloud volumes (EBS)

### Q: What do the different "filesystems" mean on my server?

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        40G  5.0G   33G   14% /           → Main disk (root partition)
devtmpfs        4.0G     0  4.0G    0% /dev        → Virtual device files (in RAM)
tmpfs           4.0G     0  4.0G    0% /dev/shm   → Shared memory (RAM)
tmpfs           1.6G  128M  1.5G    8% /run       → Runtime files (RAM)
tmpfs           803M  4.0K  803M    1% /run/user/1001 → User temp space (RAM)
```

**Key insight**: `tmpfs` entries are in RAM - they don't use disk space!

### Q: If I `cd /` on Linux, is that a disk or a volume?

`/` is the **root filesystem mount point**.

On EC2 it is usually backed by the root EBS volume, so both statements can be true:
- Linux view: root filesystem
- Cloud view: root volume

### Q: In AWS, what is OS disk vs data disk?

Both are usually EBS volumes; difference is role:
- **OS disk** = root/boot volume (contains OS and boot files)
- **Data disk** = additional non-root volumes for app data/logs/db

### Q: If I attach an EBS volume, is it auto-mounted in Linux?

Usually **no**. Attach makes block device visible, but you normally still must:
1. detect device (`lsblk`)
2. format if new (`mkfs`, one-time)
3. mount (`mount /dev/... /data`)
4. persist (`/etc/fstab` with UUID)

### Q: When does auto-mount happen?

Only when preconfigured automation exists, for example:
- custom AMI with startup mount logic
- cloud-init user-data scripts
- provisioning tools (Ansible/Terraform scripts)

Without that, attached volume is present but not mounted to a filesystem path.

### Q: Is an attached-but-unmounted volume useful?

Not for normal file-path access by apps/users. It exists as a block device, but typical workloads need filesystem + mount point.

---

## 7. Log Rotation

### Q: What is log rotation?

**Problem**: Logs grow forever → eventually fill disk.

**Solution**: Rotate them:
```
app.log → app.log.1 → app.log.2 → app.log.3 (deleted)
```

| Without Rotation | With Rotation |
|------------------|----------------|
| app.log (100GB forever!) | app.log (100MB) |
| | app.log.1 (100MB) |
| | app.log.2 (100MB) |
| | app.log.3 (deleted) |

### Q: What tool handles log rotation?

`logrotate` - Linux standard utility.

### Q: What happens without log rotation?

- Disk fills up
- App crashes (can't write logs)
- Inode exhaustion (millions of small log files)
- Node becomes unresponsive

### Q: How do I configure log rotation in practice?

Typical `/etc/logrotate.d/myapp` example:

```conf
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 myapp myapp
    sharedscripts
    postrotate
        systemctl kill -s HUP myapp.service >/dev/null 2>&1 || true
    endscript
}
```

### Q: What are common log rotation best practices?

1. Rotate by time and/or size (`daily`, `weekly`, `size 100M`).
2. Keep bounded history (`rotate N`) and compress old logs.
3. Ensure applications reopen log files after rotation (`HUP` or app-specific signal).
4. Set correct ownership and permissions for newly created log files (`create`).
5. Test config safely before rollout (`logrotate -d`, then controlled `-f`).
6. Monitor both disk space and inode usage (`df -h`, `df -i`).
7. For containerized workloads, prefer stdout/stderr with centralized logging.

---

## 8. Monitoring Commands

### Q: How to check disk usage?

```bash
df -h           # Human-readable disk usage
df -i           # Inode usage
du -h           # Directory sizes
du -h /path | sort -rh | head -20   # Largest directories
```

### Q: How to find which directory has the most files?

```bash
find / -xdev -type f | awk '{print $6}' | sort | uniq -c | sort -rn | head
```

### Q: How to clean up disk space?

```bash
# Clean old journal logs
journalctl --vacuum-time=7d

# Clean Docker
docker system prune -a

# Clean npm cache
npm cache clean --force
```

### Q: How to check Kubernetes disk pressure?

```bash
kubectl describe node | grep -A 10 "Conditions"
```

### Q: What are the important things to check in `iostat -x`?

Start with:
```bash
iostat -x 1 5
```

Focus on these columns first:

| Column | What It Means | Practical Signal |
|--------|----------------|------------------|
| `%util` | Device busy time | Sustained `>70-80%` suggests saturation risk |
| `r_await`, `w_await` | Read/write latency per request (ms) | Single-digit ms is usually healthy; sustained high double/triple digits is bad |
| `aqu-sz` | Average queue depth | If consistently growing or `>1-2` for light workloads, device may be bottlenecked |
| `%iowait` (CPU row) | CPU time waiting on storage | High + high disk latency/util usually confirms I/O bottleneck |
| `r/s`, `w/s`, `rkB/s`, `wkB/s` | IOPS and throughput shape | Use for workload pattern, not as absolute "good/bad" alone |

Quick interpretation pattern:
1. Check `%util` (is device saturated?).
2. Check `r_await`/`w_await` (is latency high?).
3. Check `aqu-sz` (is queue building?).
4. Correlate with CPU `%iowait`.
5. Then use IOPS/throughput columns to understand the traffic profile.

Example (from your sample):
- `%util` around `0.40` on `vda` -> device is mostly idle.
- `r_await 0.37 ms`, `w_await 1.44 ms` -> very low latency.
- `aqu-sz 0.07` -> negligible queueing.
- `%iowait 0.02` -> no storage wait pressure.

Conclusion: no disk bottleneck indicated in that sample.

---

## 9. ncdu vs df vs du

### Q: When should I use ncdu vs df vs du?

Each has its place:

| Command | Use Case | Speed |
|---------|----------|-------|
| `df -h` | Quick: "how much disk space left?" | Instant |
| `df -i` | Quick: "any inode issues?" | Instant |
| `du -h` | Know directory sizes | Medium |
| `du -sh *` | Quick size of immediate subdirs | Medium |
| `ncdu` | Interactive exploration | Slow (scans all) |

### Q: What are ncdu advantages?

- Interactive navigation
- Shows file count + size
- Can delete directly
- Visual progress bar

### Q: What are ncdu disadvantages?

- Scans entire tree (slow on large filesystems)
- No real-time monitoring

### Q: Practical workflow

```bash
# 1. Quick check - what's full?
df -h

# 2. Which directory is the problem?
du -sh /var/*

# 3. Deep dive - what's inside?
ncdu /var/log
```

**Bottom line**: ncdu = diagnosis tool. df = monitoring tool.

### Q: How to use ncdu?

```bash
# Install
sudo dnf install ncdu    # RHEL/Rocky
sudo apt install ncdu    # Ubuntu

# Usage
ncdu                     # Scan current directory
ncdu /home              # Scan specific path
ncdu -o - /path | jq    # Export to JSON
```

**Navigation in ncdu:**
- `Arrow keys` = navigate
- `Enter` = open directory
- `d` = delete file/directory
- `q` = quit
- `?` = help

---

## 10. SOP: Disk Troubleshooting

### Phase 1: Quick Health Check (60 seconds)

#### Step 1: Identify the Problem Type
```bash
# Is it disk full?
df -h

# Is it inode exhaustion?
df -i

# Is it I/O slowness?
iostat -x 1 5
```

#### Step 2: Identify Affected Filesystem
```bash
# Which mount point is the issue?
df -h | grep -E "(100%|9[5-9]%)"

# Is it a specific filesystem type?
mount | grep -E "^/dev"
```

### Phase 2: Diagnosis by Environment

#### A. Physical/Virtual Linux Servers (RHEL/CentOS/Rocky/Ubuntu)

| Step | Command | What It Shows |
|------|---------|---------------|
| 1 | `df -h` | Disk space usage by mount |
| 2 | `df -i` | Inode usage |
| 3 | `du -sh /path/*` | Top-level directory sizes |
| 4 | `ncdu /path` | Interactive deep dive |
| 5 | `ls -lah /tmp` | Temp files |
| 6 | `journalctl --disk-usage` | Log size |
| 7 | `docker system df` | Docker space |

#### B. Common Cleanup Commands
```bash
# Clean journal logs (RHEL/Rocky)
sudo journalctl --vacuum-time=7d
sudo journalctl --vacuum-size=500M

# Clean apt (Ubuntu)
sudo apt-get clean
sudo apt-get autoremove

# Clean yum/dnf cache (RHEL/Rocky)
sudo dnf clean all

# Clean Docker
docker system prune -a
docker volume prune

# Clean old kernels (RHEL/Rocky)
sudo package-cleanup --oldkernels --count=2
```

#### C. Kubernetes Pods/Containers

| Scenario | Command | Action |
|----------|---------|--------|
| **Pod not starting** | `kubectl describe pod <name>` | Check Events for volume issues |
| **Node disk pressure** | `kubectl describe node <node>` | Check Conditions |
| **PVC stuck** | `kubectl get pvc` | Check Status |
| **Container disk** | `kubectl exec -it pod -- df -h` | Inside container |

#### D. Kubernetes Node Diagnostics
```bash
# Check kubelet eviction
kubectl describe node | grep -A 5 "Conditions"

# Check which pods use most disk
kubectl top pods --sort-by=memory

# Check PV reclaim policy
kubectl get pv -o wide

# Check PVC events
kubectl describe pvc <name>
```

### Phase 3: Root Cause Analysis

#### Common Issues & Solutions

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| "No space left" but df shows space | Inode exhaustion | `df -i`, find excessive files |
| "Read-only file system" | Disk error/filesystem corruption | `dmesg`, `fsck` |
| High I/O wait | Disk bottleneck | `iotop`, `iostat -x` |
| Pod evictions | Node disk pressure | `kubectl describe node` |
| Slow database | Disk I/O latency | Check IOPS/throughput |

#### I/O Troubleshooting
```bash
# Who is using disk most?
iotop

# Block device stats
iostat -x 1 5

# Check for I/O errors
dmesg | grep -i "error\|fail\|io"

# Check LVM (if applicable)
lvs
pvs
vgdisplay
```

### Phase 4: Prevention (Monitoring)

#### What to Monitor
```bash
# Prometheus alerts (example)
- alert: DiskSpaceHigh
  expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) < 0.1
  for: 5m
  labels:
    severity: warning

- alert: DiskInodesHigh
  expr: (node_filesystem_files_free{mountpoint="/"} / node_filesystem_files{mountpoint="/"}) < 0.1
  for: 5m
  labels:
    severity: warning
```

#### Automation
```bash
# Logrotate config (prevention)
cat /etc/logrotate.conf

# Docker prune cron (container cleanup)
0 0 * * 0 docker system prune -af
```

### Quick Reference Matrix

| Environment | First Command | Second Command | Third Command |
|-------------|---------------|----------------|---------------|
| Linux Server | `df -h` | `du -sh /*` | `ncdu` |
| Containers | `docker system df` | `docker system prune` | Check logs |
| Kubernetes Pod | `kubectl describe pod` | `kubectl exec -- df -h` | Check PVC |
| Kubernetes Node | `kubectl describe node` | `kubectl top node` | Check kubelet |
| I/O Issues | `iostat -x` | `iotop` | `dmesg` |

---

## 11. StatefulSets and Storage Challenges

### Q: Why are StatefulSets more challenging than Deployments?

| Aspect | Stateless (Deployment) | StatefulSet |
|--------|------------------------|-------------|
| Restart | Kill anywhere, recreate | Must return to same identity |
| Storage | Ephemeral | PersistentVolume |
| Scaling | Easy | Hard (ordered identity) |
| Failure | Replace | Must recover existing state |

### Q: What are real challenges with StatefulSets?

| Problem | Impact |
|---------|--------|
| Volume not available | Pod stuck in Pending |
| Storage node failure | Pod can't reschedule |
| Data loss | PV deleted = data gone |
| Cross-zone costs | PV in us-east-1a can't attach to us-east-1b |
| Backup complexity | Must snapshot PVCs |

### Q: When should I use StatefulSets?

- **Yes**: Databases (etcd, PostgreSQL, MongoDB)
- **Maybe**: Caches (Redis) - consider ephemeral
- **No**: Frontends, APIs, stateless workers

**Rule**: Only use StatefulSets when you truly need persistent identity and data.

---

## Quick Reference Commands

```bash
# Disk space
df -h

# Inodes
df -i

# Directory sizes (sorted)
du -h /path | sort -rh | head -20

# Largest files
find /path -type f -exec du -h {} + | sort -rh | head -20

# Interactive disk analyzer
ncdu /path

# Clean Docker
docker system prune -a

# Clean logs
journalctl --vacuum-time=7d

# Kubernetes node pressure
kubectl describe node | grep -A 10 "Conditions"

# I/O monitoring
iostat -x 1 5
iotop
```
