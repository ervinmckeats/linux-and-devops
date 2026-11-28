# Lesson 14. Processes, signals, jobs, tmux

## Goal
See what runs on a system, stop what misbehaves, and keep long jobs alive after you disconnect.

## Theory

A **process** is a running program with its own PID, memory, open files, user and parent.

- New processes are made by **fork** (copy the parent) + **exec** (replace with a new program). That is why every process has a parent (PPID). PID 1 (`systemd`) is the root of the tree.
- When a process ends, it stays a **zombie** (`Z`) until the parent reads its exit code. Zombies use no memory, but many of them mean a buggy parent.
- If a parent dies first, the orphan is adopted by PID 1 (or a "subreaper").

### Process states (`STAT` column)

| State | Meaning |
|---|---|
| `R` | running or ready to run |
| `S` | sleeping, waiting for an event (normal) |
| `D` | uninterruptible sleep, usually waiting for disk/NFS. Cannot be killed. Many `D` = I/O problem |
| `Z` | zombie |
| `T` | stopped (Ctrl+Z or SIGSTOP) |

### Signals

| Signal | No. | Meaning | Can be caught? |
|---|---|---|---|
| `SIGHUP` | 1 | terminal closed; daemons often reload config | yes |
| `SIGINT` | 2 | Ctrl+C | yes |
| `SIGKILL` | 9 | kill now, no cleanup | **no** |
| `SIGTERM` | 15 | please exit (default of `kill`) | yes |
| `SIGSTOP` / `SIGCONT` | 19 / 18 | pause / resume | no / yes |

Always try `SIGTERM` first and wait. `SIGKILL` skips cleanup: half-written files, stale locks, lost buffers.

## Commands

```bash
# all processes, BSD style
ps aux

# tree
ps -ef --forest

ps -eo pid,ppid,user,%cpu,%mem,stat,cmd --sort=-%mem | head
pstree -p

# find by name
pgrep -a nginx

# live: P=cpu M=mem k=kill 1=per-cpu q=quit
top

# friendlier top
htop

# SIGTERM
kill PID

# SIGKILL, last resort
kill -9 PID

# often: reload config
kill -HUP PID

# by full command line
pkill -f "python app.py"

killall nginx

# start with lower priority (19 = lowest)
nice -n 10 ./heavy.sh

renice -n 5 -p PID

# everything the kernel knows
cat /proc/PID/status

# open files
ls -l /proc/PID/fd

lsof -p PID
```

### Load average

`uptime` → `load average: 0.50, 1.20, 2.10` = average number of processes running **or waiting** (including `D` state) over 1, 5, 15 minutes. Compare with CPU count (`nproc`). Load 8 on 8 cores is busy but fine; on 2 cores it is a queue.

## Jobs

```bash
# run in background
long_task &

# list
jobs

# pause the foreground job
Ctrl+Z

# continue in background / bring back
bg %1; fg %1

# survive logout
nohup ./backup.sh > backup.log 2>&1 &

# detach a running job from the shell
disown %1
```

## tmux: sessions that survive disconnects

```bash
# new session
tmux new -s work

# Ctrl+b d                # detach (everything keeps running)
tmux ls
tmux attach -t work
# Ctrl+b c  new window    Ctrl+b n/p  next/prev
# Ctrl+b %  split vert.   Ctrl+b "    split horiz.   Ctrl+b arrows  move
```

Run every long operation on a remote server (upgrades, migrations, big copies) inside tmux.

## Practice

1. Find the 5 processes using the most memory.
2. Start `sleep 1000` in background, pause it, resume it, then stop it with SIGTERM.
3. Write a script that ignores SIGTERM (`trap '' TERM; sleep 1000`). Try `kill`, then `kill -9`.
4. Create a zombie: `bash -c 'sleep 1 & exec sleep 30'` and find it with `ps` (`Z` state).
5. Start `tmux`, run `top`, detach, log out, log in, attach.

## Check yourself

- Why can't you kill a process in `D` state?
- Why should `SIGKILL` be the last resort?
- Load average is 6 on a 4-core box. Is that bad? What else do you check?

---

<p align="left">
  <a href="media/14-processes-signals.mp4"><img width="640" src="media/14-processes-signals.gif" alt="Lesson 14 animation: Processes, signals, jobs, tmux"></a>
</p>

## Next
[Lesson 15. Package managers](15-package-managers.md)
