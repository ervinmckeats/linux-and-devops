# Lesson 4. Navigating the filesystem

## Goal
Move around the filesystem with confidence and know what lives where.

## Theory

Linux has **one tree** that starts at `/` (root). There are no drive letters: other disks are **mounted** into folders of this tree.

- **Absolute path** starts at `/`: `/etc/nginx/nginx.conf`
- **Relative path** starts from the current directory: `nginx/nginx.conf`
- `.` current directory, `..` parent, `~` your home, `-` previous directory
- Names are **case-sensitive**: `File` and `file` are different.
- Files starting with `.` are hidden (`.bashrc`, `.ssh`).

### Filesystem Hierarchy Standard (FHS)

| Path | Contents |
|---|---|
| `/etc` | system configuration (text files) |
| `/var` | variable data: logs (`/var/log`), databases, caches, mail |
| `/home` | user home directories |
| `/root` | home of the root user |
| `/usr/bin`, `/usr/sbin` | installed programs |
| `/usr/lib` | libraries |
| `/usr/local` | software installed by hand, not by the package manager |
| `/opt` | third-party software in its own folder |
| `/tmp` | temporary files, often cleared on reboot |
| `/dev` | devices as files (`/dev/sda`, `/dev/null`) |
| `/proc`, `/sys` | virtual: live kernel and process data, no disk space used |
| `/boot` | kernel and bootloader files |
| `/mnt`, `/media` | mount points for extra disks and USB |
| `/run` | runtime data (PID files, sockets), cleared on boot |

When something breaks: config is in `/etc`, logs are in `/var/log`, the binary is in `/usr/bin`.

## Commands

```bash
pwd
cd /var/log
cd ..
cd
cd -
ls
ls -la
ls -lh
ls -lt
ls -R
tree -L 2 /etc
```

- `pwd`: print working directory
- `cd /var/log`: absolute
- `cd ..`: up one level
- `cd`: home (same as `cd ~`)
- `cd -`: back to previous directory
- `ls`: list
- `ls -la`: long format, include hidden
- `ls -lh`: human-readable sizes
- `ls -lt`: sort by time, newest first
- `ls -R`: recursive
- `tree -L 2 /etc`: tree view, 2 levels

Reading `ls -l`:

```
-rw-r--r-- 1 alice dev 4096 Jan 10 12:00 notes.txt
│└──┬────┘ │ │     │   │    │            └ name
│   │      │ │     │   │    └ modification time
│   │      │ │     │   └ size in bytes
│   │      │ │     └ group
│   │      │ └ owner
│   │      └ hard link count
│   └ permissions (lesson 13)
└ type: - file, d directory, l symlink, c/b device, s socket, p pipe
```

## Practice

1. Go to `/etc`, list only files that end with `.conf`.
2. Find where `bash` lives with `which bash`, then `ls -l` it.
3. Use `cd -` to jump between `/var/log` and `/etc` three times.
4. Look at `/proc/self/status`: this file describes the process that reads it.
5. Run `du -sh /proc` and explain why it is (almost) zero.

## Check yourself

- Where do you look for nginx's config? Its logs?
- What is the difference between `/usr/bin` and `/usr/local/bin`?
- What does the first character in `ls -l` output mean?

## Next
[Lesson 5. Files and directories](05-files-and-directories.md)
