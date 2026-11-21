# Lesson 18. Scheduling: cron and systemd timers

## Goal
Run jobs on a schedule reliably, and know when the job silently failed.

## cron

```
┌──────── minute (0-59)
│ ┌────── hour (0-23)
│ │ ┌──── day of month (1-31)
│ │ │ ┌── month (1-12)
│ │ │ │ ┌ day of week (0-7, 0 and 7 = Sunday)
│ │ │ │ │
* * * * *  command
```

| Expression | Means |
|---|---|
| `*/5 * * * *` | every 5 minutes |
| `0 3 * * *` | every day at 03:00 |
| `30 2 * * 1-5` | 02:30 on weekdays |
| `0 0 1 * *` | first day of each month |
| `@reboot` | once after boot |

Check expressions at crontab.guru before trusting them.

```bash
# edit your crontab
crontab -e

# list
crontab -l

sudo crontab -u app -e

# system jobs (cron.d lines have a USER field)
ls /etc/cron.d/ /etc/cron.daily/
```

### The classic cron traps

1. **Minimal environment.** `PATH` is short (`/usr/bin:/bin`), no `.bashrc`. Use full paths or set `PATH=` at the top of the crontab.
2. **`%` is special**: it means newline. Escape it: `date +\%F`.
3. **Output is lost** (or mailed to nobody). Always redirect: `>> /var/log/job.log 2>&1`.
4. **Overlapping runs.** A slow job started every minute piles up. Use a lock: `flock -n /tmp/job.lock /opt/job.sh`.
5. **Server was off** at 03:00 → job skipped. cron does not catch up.

```
PATH=/usr/local/bin:/usr/bin:/bin
0 3 * * * flock -n /tmp/backup.lock /opt/backup.sh >> /var/log/backup.log 2>&1
```

## systemd timers

Timers fix most cron traps: logs go to the journal, runs never overlap, missed runs can catch up, and you can add resource limits.

`/etc/systemd/system/backup.service`:

```ini
[Unit]
Description=Nightly backup

[Service]
Type=oneshot
User=backup
ExecStart=/opt/backup.sh
Nice=10
IOSchedulingClass=idle
```

`/etc/systemd/system/backup.timer`:

```ini
[Unit]
Description=Run backup nightly

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true            # run after boot if the scheduled time was missed
RandomizedDelaySec=15min   # spread load across a fleet

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
systemctl list-timers

# run once now, to test
sudo systemctl start backup.service

journalctl -u backup.service

# check a schedule
systemd-analyze calendar "Mon..Fri 09:00"
```

Other forms: `OnCalendar=hourly`, `OnCalendar=*:0/15` (every 15 min), `OnBootSec=5min`, `OnUnitActiveSec=1h`.

| | cron | systemd timer |
|---|---|---|
| Setup | one line | two files |
| Logs | do it yourself | journal |
| Missed runs | lost | `Persistent=true` |
| Overlap | needs `flock` | never overlaps |
| Limits, sandboxing | no | yes |

## Alert when a job fails

A job that fails silently is worse than no job. Options:
- `OnFailure=notify@%n.service` in the `[Unit]` section of the service.
- Ping a dead-man's switch (healthchecks-style URL) at the **end** of a successful run; the monitor alerts if pings stop.

## Practice

1. A cron job that writes `date` to `/tmp/cron.log` every minute. Then break it by using `date +%F` without escaping and see what happens.
2. Rewrite it as a systemd timer every 2 minutes. Check `list-timers` and the journal.
3. Make a job that sleeps 90 seconds, schedule it every minute with cron, see overlap with `ps`. Fix with `flock`.
4. Use `systemd-analyze calendar` to check "every 15 minutes during working hours on weekdays".

## Check yourself

- Why does a script work in your shell and fail in cron?
- What does `Persistent=true` do?
- How do you find out that a nightly job has not run for a week?

## Next
[Lesson 19. Disks and filesystems](19-disks-filesystems.md)
