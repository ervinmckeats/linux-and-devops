# Lesson 40. Tracing: strace, ltrace, perf, eBPF

## Goal
Look inside a running process to see what it is waiting for, what files it opens, and where CPU time goes.

## When to use what

| Question | Tool |
|---|---|
| Why does it fail? What files/sockets does it touch? | `strace` |
| Where does it hang? | `strace -p`, `cat /proc/PID/stack`, `gdb`/`py-spy` |
| Which functions burn CPU? | `perf top`, `perf record` + flame graph |
| What happens system-wide at low overhead? | eBPF: `bpftrace`, BCC tools |

## strace: system calls

```bash
# every syscall and its result
strace ls /nope

# only file opens, follow children
strace -f -e trace=openat,stat nginx -t

strace -f -e trace=network curl -s example.com > /dev/null

# attach to a running process
strace -p PID

# show 200 chars of data
strace -f -p PID -e trace=read,write -s 200

# summary: which syscalls, how many, how long (Ctrl+C)
strace -c -p PID

# timestamps and time spent in each call
strace -tt -T -p PID

# one log file per thread/process
strace -o trace.log -ff ./app
```

Typical findings:
- `openat("/etc/app/config.yml", ...) = -1 ENOENT`: the app looks for config in a different place than you think.
- `connect(... 5432 ...) = -1 ETIMEDOUT`: network to the DB.
- A process stuck in `futex(...)`: waiting on a lock in another thread.
- Stuck in `read(5, ...`: check `ls -l /proc/PID/fd/5` to see what fd 5 is (socket, pipe, file).
- `strace -c` shows millions of tiny `write` calls: missing buffering.

**Warning:** strace slows the traced process a lot (sometimes 10×+). Use it briefly in production, or use eBPF tools.

`ltrace` does the same for library calls (`malloc`, `strlen`): slower still, mostly for debugging.

## Where is it stuck?

```bash
cat /proc/PID/status | grep State

# kernel stack (for D state)
sudo cat /proc/PID/stack

# kernel function it sleeps in
cat /proc/PID/wchan; echo

# open files and sockets
ls -l /proc/PID/fd

# native stacks (needs symbols)
gdb -p PID -batch -ex 'thread apply all bt'

# Python stacks
py-spy dump --pid PID

# Java threads
jstack PID
```

## perf: CPU profiling

```bash
sudo apt install linux-tools-common linux-tools-$(uname -r)

# live: hottest functions system-wide
sudo perf top

sudo perf top -p PID

# cycles, instructions, cache misses, context switches
sudo perf stat -p PID sleep 10

# sample stacks 99 times/s for 30 s
sudo perf record -F 99 -g -p PID -- sleep 30

sudo perf report
```

### Flame graphs

```bash
git clone https://github.com/brendangregg/FlameGraph
sudo perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > cpu.svg
```

How to read: the x-axis is **share of samples** (not time order); width = how much CPU a function (with its children) used. Look for wide plateaus at the top.

For interpreted languages use their profilers (py-spy, async-profiler for JVM, pprof for Go): they show your function names instead of interpreter internals.

## eBPF: safe programs inside the kernel

eBPF runs small verified programs on kernel events with very low overhead. It powers modern tracing, networking (Cilium) and security tools (Falco, Tetragon).

### BCC tools (ready-made)

```bash
# tools end with -bpfcc on Debian/Ubuntu
sudo apt install bpfcc-tools

# every new process (catch short-lived ones top misses)
sudo execsnoop-bpfcc

# files opened
sudo opensnoop-bpfcc -p PID

# disk I/O latency histogram
sudo biolatency-bpfcc

# outgoing TCP connections
sudo tcpconnect-bpfcc

# TCP retransmits with addresses
sudo tcpretrans-bpfcc

# how long tasks wait for CPU
sudo runqlat-bpfcc

# OOM kills with details
sudo oomkill-bpfcc

# CPU profile, low overhead
sudo profile-bpfcc -F 99 30
```

### bpftrace one-liners

```bash
# syscalls per process
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'

# read size histogram
sudo bpftrace -e 'kprobe:vfs_read { @bytes[comm] = hist(arg2); }'
```

## Debugging recipe: "the app is slow sometimes"

1. Metrics: is it CPU, I/O, or waiting on something else? (lesson 39)
2. `runqlat`: waiting for CPU? `biolatency`: slow disk? `tcpretrans`: network loss?
3. `strace -c -p` for 10 seconds: unexpected syscall pattern?
4. `perf record` / language profiler: where is CPU spent?
5. Fix, then measure again with the same tools.

## Practice

1. Run `strace -e trace=openat` on `cat /etc/hostname` and on `python3 -c 'import json'`. Count files opened.
2. Find the config file a program reads using strace (try `ssh -G host`).
3. Use `strace -c` on `dd if=/dev/zero of=/dev/null bs=1 count=1000000` vs `bs=1M count=1`.
4. Generate a flame graph of a busy process (e.g. `stress-ng --cpu 1` or `gzip` of a big file).
5. Run `execsnoop` while a cron job runs. Catch the short-lived processes.

## Check yourself

- Why is strace risky on a production database?
- What does the width of a box in a flame graph mean?
- Which tool shows processes that live for 50 ms and disappear?

---

<p align="left">
  <a href="media/40-tracing.mp4"><img width="640" src="media/40-tracing.gif" alt="Lesson 40 animation: Tracing: strace, ltrace, perf, eBPF"></a>
</p>

## Next
[Lesson 41. Docker basics](../07-containers/41-docker-basics.md)
