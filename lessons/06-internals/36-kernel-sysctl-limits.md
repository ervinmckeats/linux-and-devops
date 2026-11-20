# Lesson 36. Kernel: sysctl, modules, limits

## Goal
Tune kernel parameters safely, manage modules, and fix "too many open files" for good.

## User space and kernel space

Programs ask the kernel for everything through **system calls**: `open`, `read`, `write`, `mmap`, `fork`, `execve`, `socket`, `connect`. A switch into the kernel costs time, which is why a program doing millions of tiny syscalls is slow (see `strace -c`, lesson 40).

The kernel exposes its state as files:
- `/proc` — processes (`/proc/PID/`) and kernel tunables (`/proc/sys/`).
- `/sys` — devices, drivers, cgroups (`/sys/fs/cgroup`).

## sysctl

```bash
sysctl -a | less                         # all parameters
sysctl net.ipv4.ip_forward               # read
cat /proc/sys/net/ipv4/ip_forward        # same thing
sudo sysctl -w vm.swappiness=10          # change now (lost on reboot)
```

Persist in `/etc/sysctl.d/90-custom.conf`:

```
vm.swappiness = 10
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.ip_local_port_range = 10240 65535
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288
```

```bash
sudo sysctl --system                     # load all files
```

Parameters people actually touch:

| Parameter | Why |
|---|---|
| `vm.swappiness` | how eagerly to swap (0–200; 10–30 for servers with apps, 60 default) |
| `vm.overcommit_memory` | memory overcommit policy (Redis docs ask for 1) |
| `vm.max_map_count` | Elasticsearch/OpenSearch need 262144+ |
| `net.core.somaxconn` | max queue of accepted-but-not-yet-served connections |
| `net.ipv4.ip_local_port_range` | ephemeral ports for outgoing connections (proxies) |
| `net.ipv4.tcp_tw_reuse` | reuse `TIME_WAIT` sockets for outgoing connections |
| `net.ipv4.ip_forward` | routing / containers / VPN |
| `fs.inotify.max_user_watches` | file watchers (IDEs, kubelet, log agents) |

Do not copy "ultimate tuning" lists from the internet. Change one thing, measure, keep notes.

## Kernel modules

Drivers and features loaded on demand.

```bash
lsmod | head
modinfo br_netfilter
sudo modprobe br_netfilter               # load
sudo modprobe -r module_name             # unload
echo br_netfilter | sudo tee /etc/modules-load.d/k8s.conf      # load at boot
echo "blacklist pcspkr" | sudo tee /etc/modprobe.d/blacklist-pcspkr.conf
```

Kubernetes nodes need `overlay` and `br_netfilter` plus `net.bridge.bridge-nf-call-iptables=1` (lesson 49).

## Kernel boot parameters

```bash
cat /proc/cmdline
sudoedit /etc/default/grub               # GRUB_CMDLINE_LINUX="... param=value"
sudo update-grub && sudo reboot
```

Examples: `systemd.unified_cgroup_hierarchy=1`, `mitigations=auto`, `console=ttyS0` (serial console on VMs).

## Process limits (ulimit)

Every process has limits; the famous one is **open files** (`nofile`). A busy proxy or database needs tens of thousands of file descriptors (every socket is a file).

```bash
ulimit -a                 # limits of this shell
ulimit -n                 # open files (soft)
ulimit -Hn                # hard limit
cat /proc/PID/limits      # limits of a running process - check THIS, not your shell
ls /proc/PID/fd | wc -l   # how many it uses now
```

Where to raise them:

| Process started by | Set limit in |
|---|---|
| systemd service | unit: `LimitNOFILE=65536` (via `systemctl edit`) |
| user login session | `/etc/security/limits.d/*.conf` (PAM) |
| Docker container | `--ulimit nofile=65536:65536` or daemon defaults |

```
# /etc/security/limits.d/90-nofile.conf
*    soft  nofile  65536
*    hard  nofile  131072
```

`limits.conf` does **not** apply to systemd services. This is the #1 reason "I raised the limit but nginx still says Too many open files".

## Kernel messages and panics

```bash
sudo dmesg -T -w                 # follow kernel log
journalctl -k -b -1              # kernel log of previous boot
cat /proc/sys/kernel/panic       # seconds before auto-reboot on panic (0 = hang)
```

Set `kernel.panic = 10` on servers so they reboot instead of hanging. For crash analysis configure `kdump`.

## Practice

1. Read `net.core.somaxconn`, change it with `sysctl -w`, persist it in `/etc/sysctl.d/`, reboot and check.
2. Run a Python script that opens files in a loop until it fails; read the error. Raise its limit with `ulimit -n` and try again.
3. Start a systemd service, check `/proc/PID/limits`, raise `LimitNOFILE` with an override, check again.
4. Load `br_netfilter` and make it load at boot.

## Check yourself

- Why does `/etc/security/limits.conf` not affect nginx started by systemd?
- How do you see the real limits of a running process?
- Why is copying a big sysctl tuning list a bad idea?

## Next
[Lesson 37. Memory and the OOM killer](37-memory-oom.md)
