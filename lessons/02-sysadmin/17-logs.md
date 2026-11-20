# Lesson 17. Logs: journalctl, syslog, logrotate

## Goal
Find the log line that explains a failure, fast, and keep logs from filling the disk.

## Theory

Two log systems usually live side by side:

- **journald** (systemd): binary, indexed journal. Stores stdout/stderr of every service plus metadata (unit, PID, priority, boot). Read with `journalctl`.
- **rsyslog**: classic text files in `/var/log` (`syslog` / `messages`, `auth.log` / `secure`). Many distros forward the journal to it.

Applications may also write their own files: `/var/log/nginx/`, `/var/log/postgresql/`.

| File | Contents |
|---|---|
| `/var/log/syslog` (Debian) / `/var/log/messages` (RHEL) | general system log |
| `/var/log/auth.log` / `/var/log/secure` | logins, sudo, SSH |
| `/var/log/kern.log`, `dmesg` | kernel |
| `/var/log/dpkg.log`, `/var/log/apt/` | package changes |

**Priorities** (lower = worse): 0 emerg, 1 alert, 2 crit, 3 err, 4 warning, 5 notice, 6 info, 7 debug.

## journalctl

```bash
journalctl -u nginx
journalctl -u nginx -f
journalctl -u nginx -n 100 --no-pager
journalctl -p err -b
journalctl -b -1
journalctl --since "1 hour ago"
journalctl --since "2026-01-10 10:00" --until "2026-01-10 10:30"
journalctl -k
journalctl _PID=1234
journalctl -u myapp -o json-pretty
journalctl -u myapp -g "timeout"
journalctl --disk-usage
sudo journalctl --vacuum-size=500M
```

- `journalctl -u nginx`: one service
- `journalctl -u nginx -f`: follow
- `journalctl -p err -b`: errors since this boot
- `journalctl -b -1`: previous boot (needs persistent journal)
- `journalctl -k`: kernel messages (like dmesg)
- `journalctl -u myapp -o json-pretty`: all fields
- `journalctl -u myapp -g "timeout"`: grep inside journal
- `sudo journalctl --vacuum-size=500M`: shrink

Make the journal survive reboots (if `/var/log/journal` does not exist):

```bash
sudo mkdir -p /var/log/journal && sudo systemctl restart systemd-journald
```

Limit its size in `/etc/systemd/journald.conf`: `SystemMaxUse=1G`.

## dmesg

```bash
sudo dmesg -T | tail -50
sudo dmesg -T -l err,warn
sudo dmesg -T | grep -i -E "oom|killed process|i/o error|segfault"
```

- `sudo dmesg -T | tail -50`: readable timestamps

## Writing to logs

```bash
logger -t backup "backup finished in 42s"
journalctl -t backup
```

- `logger -t backup "backup finished in 42s"`: into syslog/journal

## logrotate

Stops text logs from growing forever. Runs daily (timer or cron). Config: `/etc/logrotate.conf` and `/etc/logrotate.d/`.

`/etc/logrotate.d/myapp`:

```
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 app app
    postrotate
        systemctl reload myapp > /dev/null 2>&1 || true
    endscript
}
```

- `postrotate` tells the app to reopen its file. Without it, the app keeps writing to the **renamed** file (same inode, lesson 8).
- If the app cannot reopen files, use `copytruncate` (small risk of losing lines during the copy).

```bash
sudo logrotate -d /etc/logrotate.d/myapp
sudo logrotate -f /etc/logrotate.d/myapp
```

- `sudo logrotate -d /etc/logrotate.d/myapp`: dry run
- `sudo logrotate -f /etc/logrotate.d/myapp`: force now

## Investigation routine

1. `systemctl status app`: last lines and exit code.
2. `journalctl -u app --since "10 min ago"`: full story.
3. `journalctl -p warning -b`: anything else wrong at the same time?
4. `dmesg -T | tail`: kernel: OOM, disk errors?
5. Application's own log files.

## Practice

1. Show all errors of the current boot.
2. Show SSH logins from the last day (`journalctl -u ssh --since yesterday` or `auth.log`).
3. Send a message with `logger` and find it in the journal.
4. Write a logrotate config for `/tmp/test/*.log`, run it with `-d`, then `-f`.
5. Make the journal persistent and read the previous boot after a reboot.

## Check yourself

- Why does an app keep writing to `app.log.1` after rotation?
- How do you see logs from before the last reboot?
- Where do you look first when a service fails to start?

## Next
[Lesson 18. Scheduling](18-cron-and-timers.md)
