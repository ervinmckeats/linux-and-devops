# Lesson 2. What Linux is: kernel, distributions, shell

## Goal
Understand the layers of a Linux system, so every later command has a place on the map.

## Theory

### Layers

```
┌──────────────────────────────────────────┐
│  Your programs: nginx, python, bash, vim │  user space
├──────────────────────────────────────────┤
│  System libraries: glibc, libssl         │
├──────────────────────────────────────────┤
│  System calls: open, read, fork, execve  │  the border
├──────────────────────────────────────────┤
│  Kernel: processes, memory, files, net   │  kernel space
├──────────────────────────────────────────┤
│  Hardware: CPU, RAM, disk, NIC           │
└──────────────────────────────────────────┘
```

- **Kernel** — the one program that talks to hardware. It schedules processes, gives out memory, manages files and the network. Strictly, "Linux" is just the kernel.
- **User space** — everything else. Programs cannot touch hardware directly; they ask the kernel through **system calls**. A program crash in user space does not take down the kernel.
- **Distribution** — kernel + tools + package manager + defaults, assembled and tested together. Ubuntu, Debian, RHEL, Fedora, Arch, Alpine.
- **Shell** — a program that reads your command, finds the program, starts it and shows the result. Usually `bash` (or `zsh`). It is just a program, not part of the kernel.
- **Terminal** — the window that shows the shell's text. The terminal draws, the shell interprets.

### Distribution families

| Family | Examples | Package manager | Where you meet it |
|---|---|---|---|
| Debian | Debian, Ubuntu | `apt` / `.deb` | cloud VMs, most tutorials |
| Red Hat | RHEL, Rocky, Alma, Fedora | `dnf` / `.rpm` | enterprise, banks, telecom |
| Arch | Arch, Manjaro | `pacman` | desktops, enthusiasts |
| Alpine | Alpine | `apk` | tiny container images |

Pick based on the job: Ubuntu LTS or Debian for servers and learning, Rocky/Alma if your target employer runs RHEL, Alpine for containers.

### Unix philosophy

- Small tools that do one thing well.
- Tools talk through text streams, so you can chain them: `ps aux | grep nginx | wc -l`.
- Everything is a file: disks, terminals, processes (`/proc`), even random numbers (`/dev/urandom`).
- Silence means success. A command that prints nothing usually worked.

## Commands

```bash
uname -r                  # kernel version
cat /etc/os-release       # distribution and version
echo $SHELL               # your login shell
ps -p $$                  # the shell you are in right now
cat /proc/cpuinfo | head  # the kernel exposes hardware as files
ls /dev | head            # devices as files
```

## Practice

1. Find your kernel version and distribution name.
2. Run `bash`, then `ps -p $$`, then `exit`. Notice you opened a shell inside a shell.
3. Read 16 random bytes from the kernel: `head -c 16 /dev/urandom | od -x`.
4. Install `zsh`, start it, then return to bash.

## Check yourself

- What is the difference between the kernel and a distribution?
- What is a system call? Name three.
- What is the difference between a terminal and a shell?
- Why does a program crash not crash the whole system?

## Next
[Lesson 3. Anatomy of a command](../01-terminal/03-commands-and-help.md)
