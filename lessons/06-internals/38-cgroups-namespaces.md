# Lesson 38. cgroups v2 and namespaces

## Goal
Understand the two kernel features that make containers possible, and use them directly.

**A container is a normal Linux process with:**
- **namespaces**: what it can **see** (its own PIDs, network, mounts, hostname, users)
- **cgroups**: how much it can **use** (CPU, memory, I/O, number of processes)
- plus a root filesystem (image), capabilities and seccomp filters.

## cgroups v2

A single tree under `/sys/fs/cgroup`. systemd organizes it into **slices** (`system.slice`, `user.slice`), **services** and **scopes**.

```bash
mount | grep cgroup2
systemd-cgls
systemd-cgtop
cat /proc/self/cgroup
ls /sys/fs/cgroup/system.slice/nginx.service/
```

- `mount | grep cgroup2`: check v2 is used
- `systemd-cgls`: tree of cgroups and processes
- `systemd-cgtop`: live usage per cgroup
- `cat /proc/self/cgroup`: my cgroup

Key files in a cgroup directory:

| File | Meaning |
|---|---|
| `memory.max` | hard memory limit → OOM kill when exceeded |
| `memory.high` | soft limit → throttling and reclaim |
| `memory.current` | current usage |
| `memory.events` | counters: `high`, `max`, `oom`, `oom_kill` |
| `cpu.max` | `quota period`, e.g. `50000 100000` = half a CPU |
| `cpu.weight` | relative share when CPUs are contended (default 100) |
| `cpu.stat` | `nr_throttled`, `throttled_usec`: CPU throttling |
| `io.max` | read/write bandwidth or IOPS limits per device |
| `pids.max` | max number of processes (fork bomb protection) |
| `*.pressure` | PSI for this cgroup |

### Using cgroups via systemd

```bash
sudo systemctl set-property nginx.service MemoryMax=1G CPUQuota=200%
sudo systemd-run --scope -p MemoryMax=200M -p CPUQuota=25% bash
systemctl show nginx -p MemoryCurrent,CPUUsageNSec
```

- `sudo systemd-run --scope -p MemoryMax=200M -p CPUQuota=25% bash`: a limited shell

### CPU throttling: the hidden latency killer

With `cpu.max` (Kubernetes CPU **limits**), a process that uses its quota early in a 100 ms period is **paused** until the next period. Average CPU looks low, but latency spikes. Check:

```bash
cat /sys/fs/cgroup/<path>/cpu.stat
```

- `cat /sys/fs/cgroup/<path>/cpu.stat`: nr_throttled growing = throttled

That is why many teams set CPU **requests** but no CPU **limits** in Kubernetes (lesson 52).

## Namespaces

| Namespace | Isolates |
|---|---|
| `pid` | process IDs: the container's first process is PID 1 |
| `net` | interfaces, IPs, routes, firewall, ports |
| `mnt` | mount points, root filesystem |
| `uts` | hostname |
| `ipc` | shared memory, semaphores |
| `user` | UID mapping: root inside, normal user outside (rootless containers) |
| `cgroup` | view of the cgroup tree |
| `time` | clocks (rare) |

```bash
lsns
ls -l /proc/$$/ns
```

- `lsns`: all namespaces on the host
- `ls -l /proc/$$/ns`: namespaces of my shell

### Build a "container" with unshare

```bash
sudo unshare --fork --pid --mount-proc --uts --net bash
hostname inside-box; hostname
ps aux
ip addr
exit
```

- `hostname inside-box; hostname`: changes only here
- `ps aux`: only a few processes, bash is PID 1
- `ip addr`: only lo, and it is DOWN

Add a root filesystem and you are very close to Docker:

```bash
mkdir -p /tmp/rootfs
docker export $(docker create alpine) | tar -x -C /tmp/rootfs
sudo unshare --fork --pid --mount-proc --uts --net --root=/tmp/rootfs /bin/sh
```

- `docker export $(docker create alpine) | tar -x -C /tmp/rootfs`: or debootstrap

### Enter a running container's namespaces

When a container image has no debugging tools, use the host's tools inside its namespaces:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' mycontainer)
sudo nsenter -t "$PID" -n ss -tlnp
sudo nsenter -t "$PID" -n tcpdump -ni eth0 port 80
sudo nsenter -t "$PID" -a sh
```

- `sudo nsenter -t "$PID" -n ss -tlnp`: network namespace only: host's ss, container's sockets
- `sudo nsenter -t "$PID" -a sh`: all namespaces

## Practice

1. Find the cgroup of your SSH session and of `nginx` with `systemd-cgls`.
2. Start a CPU hog in a scope with `CPUQuota=20%` and watch `top` and `cpu.stat`.
3. Create a scope with `pids.max` of 20 (`-p TasksMax=20`) and try a fork bomb inside it (`:(){ :|:& };:`). The host survives.
4. Use `unshare` to make a shell with its own hostname, PID and network namespace.
5. Run an nginx container and inspect its sockets from the host with `nsenter -n`.

## Check yourself

- Namespaces vs cgroups: which one limits memory, which one hides processes?
- Why can a container be OOM-killed while the host has free memory?
- What is CPU throttling and how do you detect it?

## Next
[Lesson 39. Performance analysis](39-performance.md)
