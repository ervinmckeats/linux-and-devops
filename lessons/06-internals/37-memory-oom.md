# Lesson 37. Memory: virtual memory, page cache, swap, OOM killer

## Goal
Read memory numbers correctly, tell a real memory problem from normal caching, and investigate an OOM kill.

## Theory

### Virtual memory
Each process sees its own large, private address space. The kernel maps it to physical RAM in **pages** (usually 4 KB) only when the process actually touches them.

- **VSZ / VIRT**: all address space the process reserved. Often huge, mostly meaningless.
- **RSS / RES**: pages currently in RAM. Closer to real usage, but counts shared libraries in every process.
- **PSS**: RSS with shared pages divided between users. Fairest number (`smem`).

Because memory is reserved lazily, Linux **overcommits**: the total promised can exceed RAM. Fine until everyone tries to use it at once.

### Page cache
Free RAM is wasted RAM. The kernel fills unused memory with cached file data, so reads come from RAM. This cache is **given back instantly** when apps need memory.

```
$ free -h
               total   used   free   shared  buff/cache  available
Mem:            15Gi   4.1Gi  0.5Gi  0.3Gi       11Gi        10Gi
```

Look at **available**, not **free**. Here only 0.5 Gi is "free", but 10 Gi is available. This server is fine.

### Swap
Disk space for pages that have not been used for a while. A bit of swap use is normal. A problem is **constant swapping in and out** (`si`/`so` columns in `vmstat`), which makes everything slow.

### The OOM killer
When memory is truly exhausted (RAM + swap, or a cgroup's limit), the kernel picks a process and kills it with `SIGKILL`. The choice is based on `oom_score` (mostly memory size, adjusted by `oom_score_adj`).

Two kinds:
- **System OOM**: the whole host is out of memory.
- **cgroup OOM**: a container or systemd service hit its own `MemoryMax` / memory limit. The host may have plenty of free RAM. This is what Kubernetes reports as **OOMKilled** (lesson 54).

Signs: exit code `137` (128 + 9), process gone without its own error log, kernel log lines.

## Commands

```bash
free -h
cat /proc/meminfo | head -20
vmstat 1 5
ps aux --sort=-rss | head
top -o %MEM
smem -tk -s pss | tail
pmap -x PID | tail -1
cat /proc/PID/status | grep -E 'VmRSS|VmSwap'
cat /proc/pressure/memory
```

- `vmstat 1 5`: si/so = swap in/out per second; r = run queue
- `ps aux --sort=-rss | head`: biggest by RSS
- `top -o %MEM`: or htop, press M
- `smem -tk -s pss | tail`: PSS per process (package smem)
- `pmap -x PID | tail -1`: memory map summary of a process
- `cat /proc/pressure/memory`: PSI: time tasks waited for memory (lesson 39)

### Investigating an OOM kill

```bash
sudo dmesg -T | grep -i -E 'out of memory|oom-kill|killed process'
journalctl -k | grep -i oom
journalctl -u myapp | tail
cat /sys/fs/cgroup/system.slice/myapp.service/memory.events
```

- `journalctl -u myapp | tail`: systemd: "A process of this unit has been killed by the OOM killer"
- `cat /sys/fs/cgroup/system.slice/myapp.service/memory.events`: oom, oom_kill counters

A kernel OOM report shows the memory state and a table of processes with their RSS at the time of the kill. Read it to see **who** was big, not only who was killed.

### Protecting important processes

```bash
echo -1000 | sudo tee /proc/PID/oom_score_adj
cat /proc/PID/oom_score
```

- `echo -1000 | sudo tee /proc/PID/oom_score_adj`: never kill (e.g. sshd); +1000 = kill first

In systemd: `OOMScoreAdjust=-500`, and better, set `MemoryMax=` on the noisy service so **it** gets killed, not the database.

`systemd-oomd` / `earlyoom` kill earlier based on memory pressure, before the system freezes.

### Leak or just growth?

1. Graph RSS of the process over hours/days (monitoring, lesson 56).
2. Saw-tooth pattern (grows, drops after GC) = normal for many runtimes.
3. Steady growth that never drops, ending in OOM = leak. Restart buys time; profiling finds the cause.
4. JVM / Go / Node have their own heap settings (`-Xmx`, `GOMEMLIMIT`, `--max-old-space-size`). Set them **below** the container limit, leaving room for non-heap memory.

### Drop caches (testing only)

```bash
sync; echo 3 | sudo tee /proc/sys/vm/drop_caches
```

Useful for cold-cache benchmarks. Never as a "fix" in production: it only makes the next reads slow.

## Practice

1. Read `free -h` on your VM and explain each column.
2. Read a large file twice with `time cat bigfile > /dev/null`. Why is the second time faster? Watch `buff/cache`.
3. Run a memory hog in a limited unit and watch it get OOM-killed:
   ```bash
   sudo systemd-run --unit=hog -p MemoryMax=100M -p MemorySwapMax=0 \
     python3 -c 'a=[]; [a.append(" " * 10**6) for _ in range(1000)]'
   journalctl -u hog; systemctl status hog
   ```
   Find the exit code and the kernel log line.
4. Use `stress-ng --vm 1 --vm-bytes 90% -t 30s` and watch `vmstat 1` and `/proc/pressure/memory`.

## Check yourself

- `free` shows 200 MB free. Is the server out of memory?
- What is the difference between a host OOM and a cgroup OOM?
- A container exits with 137 and no error in its logs. What happened and where do you confirm it?

## Next
[Lesson 38. cgroups and namespaces](38-cgroups-namespaces.md)
