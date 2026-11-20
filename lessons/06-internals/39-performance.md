# Lesson 39. Performance: USE method, first 60 seconds, PSI

## Goal
Find the bottleneck on a slow server in minutes with a method, not guessing.

## The USE method

For every resource (CPU, memory, disk, network, and limits like file descriptors) check:

- **U**tilization — how busy is it? (% time busy, % capacity used)
- **S**aturation — is work queuing? (run queue, swap, I/O wait queue, dropped packets)
- **E**rrors — any errors? (disk errors, NIC errors, OOM kills)

Saturation is usually what users feel. 100% CPU with no queue is fine; 60% disk utilization with a long queue is not.

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| CPU | `mpstat`, `top` %us+%sy | load > cores, `vmstat r`, PSI cpu, throttling | — |
| Memory | `free` available | swapping `vmstat si/so`, PSI memory | OOM in `dmesg` |
| Disk | `iostat %util` | `iostat aqu-sz`, `await`, PSI io | `dmesg` I/O errors, `smartctl` |
| Network | `sar -n DEV`, `iftop` | drops, retransmits `nstat`, `ss` queues | `ip -s link` errors |
| FDs | `/proc/PID/fd` count | — | "Too many open files" |

## The first 60 seconds

Log in and run, in order:

```bash
uptime                          # load average trend: rising or falling?
dmesg -T | tail                 # OOM kills, disk errors, network issues
vmstat 1 5                      # r (run queue), si/so (swap), wa (I/O wait), us/sy/id
mpstat -P ALL 1 3               # one core at 100%? (single-threaded bottleneck)
pidstat 1 3                     # which processes use CPU
iostat -xz 1 3                  # per disk: r/s w/s await aqu-sz %util
free -h                         # memory available, swap used
sar -n DEV 1 3                  # network throughput per interface
sar -n TCP,ETCP 1 3             # connections/s, retransmits
top                             # overview, sort by CPU (P) or memory (M)
```

(`mpstat`, `pidstat`, `iostat`, `sar` come from the `sysstat` package.)

### Reading the signs

| You see | Likely |
|---|---|
| high `us` | application code busy → profile it (lesson 40) |
| high `sy` | lots of syscalls, context switches, small I/O → `strace -c`, `perf` |
| high `wa`, `iostat await` high | disk is slow/saturated → which process: `iotop`, `pidstat -d` |
| `si`/`so` not zero for long | memory pressure → lesson 37 |
| one CPU 100%, others idle | single-threaded process or IRQ on one core |
| high `st` (steal) | noisy neighbour on the hypervisor → move instance/bigger type |
| load high, CPU idle | processes in `D` state waiting for I/O or NFS |
| retransmits | network loss, overloaded peer, MTU |

## PSI — Pressure Stall Information

Since kernel 4.20 the kernel reports **how much time tasks were stalled** waiting for a resource. It is the most direct saturation signal.

```bash
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
```

```
some avg10=12.50 avg60=8.00 avg300=3.10 total=...
full avg10=4.00  avg60=2.10 avg300=0.80 total=...
```

- `some` — % of time at least one task was waiting.
- `full` — % of time **all** non-idle tasks were waiting (nothing useful got done).
- `avg10=12.50` on memory: in the last 10 s, tasks waited for memory 12.5% of the time.

Also per cgroup: `/sys/fs/cgroup/<path>/memory.pressure`. Great for alerts: "memory full avg60 > 5%".

## Disk performance

```bash
iostat -xz 1
#  r_await/w_await   average latency in ms (SSD: < 1-2 ms; HDD: 5-15 ms)
#  aqu-sz            average queue length
#  %util             time the device was busy (misleading for SSD/NVMe with parallel queues)
sudo iotop -oPa      # processes doing I/O
pidstat -d 1
fio --name=randread --rw=randread --bs=4k --size=1G --runtime=30 --time_based --direct=1   # benchmark
```

## Network performance

```bash
ss -s
ss -tin dst :443 | head      # per-connection rtt, cwnd, retrans
nstat -az | grep -i -E 'retrans|drop|overflow'
ip -s link show eth0
```

## Latency, not only throughput

Averages hide problems. Look at percentiles (p95, p99) from your app metrics (lesson 56). A server at 40% CPU can still have a terrible p99 because of GC pauses, lock contention, or CPU throttling (lesson 38).

## Load testing

Measure before and after a change:

```bash
ab -n 10000 -c 100 http://localhost/              # apache bench, simple
wrk -t4 -c200 -d30s --latency http://localhost/   # better, shows latency distribution
hey -n 10000 -c 100 http://localhost/
```

## Practice

1. Run the first-60-seconds list on an idle VM and write down normal values.
2. `stress-ng --cpu 2 -t 60s` and repeat. What changed?
3. `stress-ng --io 4 --hdd 2 -t 60s` — find the disk saturation in `iostat` and PSI.
4. Run `wrk` against nginx; watch `mpstat`, `pidstat`, and `ss -s` during the test.
5. Write your own one-page "slow server" runbook from this lesson.

## Check yourself

- What does USE stand for, and why is saturation the most important column?
- Load is 20, CPU is 90% idle. What is going on?
- What does `full avg10=30` in `/proc/pressure/io` mean?

## Next
[Lesson 40. Tracing](40-tracing.md)
