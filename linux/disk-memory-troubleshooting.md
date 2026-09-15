# Linux Disk and Memory Troubleshooting

Disk and memory pressure can cause application crashes, failed deployments, slow servers, database failures, and unexpected service restarts.

This guide documents a practical troubleshooting workflow for Linux-based cloud servers.

---

## Troubleshooting Flow

```text
Server Performance Issue
        ↓
Check disk usage
        ↓
Check inode usage
        ↓
Identify large files/directories
        ↓
Check memory
        ↓
Check swap
        ↓
Identify high-resource processes
        ↓
Check system logs
        ↓
Fix root cause
```

---

## 1. Check Disk Usage

```bash
df -h
```

Important columns:

```text
Filesystem
Size
Used
Avail
Use%
Mounted on
```

Pay attention to filesystems approaching:

```text
80%
90%
100%
```

A completely full filesystem can prevent applications from writing logs, temporary files, uploads, or database data.

---

## 2. Check Directory Size

To identify large directories:

```bash
sudo du -sh /* 2>/dev/null
```

For a specific directory:

```bash
sudo du -sh /var/* 2>/dev/null
```

Sort directories by size:

```bash
sudo du -h /var 2>/dev/null | sort -h | tail -20
```

Common locations that may grow:

```text
/var/log
/var/lib/docker
/home
/tmp
application log directories
database storage
```

---

## 3. Find Large Files

Example:

```bash
sudo find / -type f -size +500M 2>/dev/null
```

Or:

```bash
sudo find /var -type f -size +100M -exec ls -lh {} \; 2>/dev/null
```

Do not delete large files blindly.

First determine:

- what created the file;
- whether it is currently in use;
- whether it contains required application data;
- whether log rotation should be configured.

---

## 4. Check Inode Usage

A filesystem may report free disk space but still fail to create new files because all inodes are consumed.

Check:

```bash
df -i
```

High inode usage is often caused by a very large number of small files.

Examples:

```text
cache files
temporary files
session files
application-generated files
```

---

## 5. Check Memory Usage

```bash
free -h
```

Important fields include:

```text
total
used
free
shared
buff/cache
available
```

On Linux, low `free` memory does not always mean the server is out of memory.

The `available` value is usually more useful when evaluating remaining usable memory.

---

## 6. Check Real-Time Resource Usage

Use:

```bash
top
```

If available:

```bash
htop
```

Look for:

```text
CPU %
Memory %
Load average
High-resource processes
```

---

## 7. Identify High-Memory Processes

```bash
ps aux --sort=-%mem | head
```

High-CPU processes:

```bash
ps aux --sort=-%cpu | head
```

This helps identify the process consuming the most resources.

---

## 8. Check Swap

```bash
swapon --show
```

Memory overview:

```bash
free -h
```

If swap usage is consistently high, investigate:

```text
insufficient RAM
memory leaks
application workload
oversized services
resource limits
```

Swap can help during short memory spikes, but it should not hide persistent memory problems.

---

## 9. Check for Out-of-Memory Events

The Linux kernel may terminate processes when the system runs out of memory.

Check:

```bash
dmesg | grep -i -E "out of memory|killed process"
```

or:

```bash
journalctl -k | grep -i -E "out of memory|killed process"
```

Typical indicators:

```text
Out of memory
Killed process
oom-killer
```

If an application disappears unexpectedly, check for OOM events.

---

## 10. Check System Load

```bash
uptime
```

Example:

```text
load average: 0.45, 0.60, 0.55
```

Load average should be interpreted relative to the number of CPU cores and workload characteristics.

Check CPU count:

```bash
nproc
```

---

## 11. Check Log Usage

```bash
sudo du -sh /var/log
```

Identify large logs:

```bash
sudo find /var/log -type f -exec du -h {} + | sort -h | tail -20
```

Check log rotation configuration:

```bash
ls -l /etc/logrotate.d/
```

Growing logs should normally be controlled through proper rotation rather than manual deletion.

---

## 12. Docker Disk Usage

Docker hosts can accumulate:

```text
unused images
stopped containers
volumes
build cache
```

Check:

```bash
docker system df
```

Inspect the Docker storage directory:

```bash
sudo du -sh /var/lib/docker
```

Do not run aggressive cleanup commands on production systems without checking what will be removed.

---

## Common Problems

| Problem | What to Check |
|---|---|
| Disk almost full | `df -h` |
| Too many files | `df -i` |
| Unknown large directory | `du` |
| Huge files | `find` |
| High memory usage | `free -h`, `top` |
| Process using excessive RAM | `ps aux --sort=-%mem` |
| High CPU | `ps aux --sort=-%cpu` |
| Application killed unexpectedly | OOM logs |
| Docker consuming disk | `docker system df` |
| Logs consuming disk | `/var/log` and logrotate |

---

## Recommended Troubleshooting Order

```text
Check filesystem
      ↓
Check inode usage
      ↓
Find large directories/files
      ↓
Check memory
      ↓
Check swap
      ↓
Check processes
      ↓
Check OOM events
      ↓
Check logs
      ↓
Identify root cause
```

---

## Key Principle

Do not immediately delete files or restart services when a Linux server runs out of resources.

First identify:

```text
What resource is exhausted?
        ↓
What process or data caused it?
        ↓
Is it temporary or persistent?
        ↓
What is the safe corrective action?
```

The goal is to fix the cause of resource exhaustion rather than only recovering disk space or memory temporarily.
