# Lesson 1. How to use the course and set up a lab

## Goal
Get a safe Linux machine you can break and restore in seconds.

## Theory

You learn Linux by doing, and doing means making mistakes. A lab is a machine where mistakes cost nothing:

- **Virtual machine (VM)** — a full computer emulated by software. Best option: real kernel, real boot, snapshots.
- **WSL2** (Windows) — a real Linux kernel inside Windows. Good for daily commands, weak for boot, disks and systemd topics.
- **Cloud VM** — a small server from any provider. Good for networking and SSH lessons. Costs money; destroy it when done.
- **Container** — fast, but it is *not* a full system (no real boot, no kernel of its own). Use it later, in Module 7.

**Snapshot** = a saved state of the VM disk. Take one before every risky task. If you break the system, roll back in seconds.

Recommended distribution for this course: **Ubuntu Server LTS** (24.04 or newer). Most examples also work on Debian. Where Red Hat family (RHEL, Rocky, Alma, Fedora) differs, the lesson says so.

## Commands

Option A. Multipass (Linux, macOS, Windows) — one command per VM:

```bash
multipass launch --name lab --cpus 2 --memory 2G --disk 20G
multipass shell lab          # open a shell inside
multipass snapshot lab       # save state (VM must be stopped: multipass stop lab)
multipass restore lab.snapshot1
multipass delete --purge lab # remove completely
```

Option B. VirtualBox / UTM / VMware with an Ubuntu Server ISO:

1. Create a VM: 2 CPU, 2 GB RAM, 20 GB disk.
2. Install with default options, enable **OpenSSH server**.
3. After the first boot, take a snapshot named `clean`.

Option C. WSL2:

```powershell
wsl --install -d Ubuntu
```

First commands in a fresh system:

```bash
sudo apt update && sudo apt upgrade -y   # update package lists and packages
sudo apt install -y curl git vim tree htop tldr
hostnamectl                               # what machine am I on?
```

## Study habits that work

- Type commands by hand. Muscle memory matters.
- Keep your own notes: command, what it does, one example.
- Read `--help` before searching the web.
- Repeat a lesson a few days later. Spaced repetition beats rereading.
- Run a small real project (home server, pet site). Real problems teach fastest.

## Practice

1. Create a VM with one of the options above.
2. Update it and install the packages listed.
3. Take a snapshot named `clean`.
4. Run `sudo rm -rf /usr/bin/ls` (yes, on purpose). Try `ls`. Restore the snapshot. Check that `ls` works again.

## Check yourself

- Why is a container not enough for learning the boot process?
- What is a snapshot and when do you take one?
- Which distribution family uses `apt`, which uses `dnf`?

## Next
[Lesson 2. What Linux is](02-what-is-linux.md)
