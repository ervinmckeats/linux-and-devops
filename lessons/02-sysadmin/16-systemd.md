# Lesson 16. systemd and services

## Goal
Run any program as a reliable service: start on boot, restart on crash, limited resources, readable logs.

## Theory

**systemd** is PID 1 on almost every modern distribution. It starts the system, supervises services, collects logs (journald), handles timers, mounts and sockets.

It manages **units**, described in text files:

| Unit type | For |
|---|---|
| `.service` | a daemon or a one-shot job |
| `.timer` | a schedule (lesson 18) |
| `.socket` | start a service on first connection |
| `.mount` | a mount point |
| `.target` | a group of units, a "state" (`multi-user.target`, `graphical.target`) |

Unit file locations (higher wins):
1. `/etc/systemd/system/`: yours, admin overrides
2. `/run/systemd/system/`: runtime
3. `/usr/lib/systemd/system/` (or `/lib/...`): from packages. **Do not edit these**: upgrades overwrite them.

## Commands

```bash
systemctl status nginx
sudo systemctl start | stop | restart nginx
sudo systemctl reload nginx
sudo systemctl enable --now nginx
sudo systemctl disable nginx
systemctl is-active nginx; systemctl is-enabled nginx
systemctl list-units --type=service --state=running
systemctl --failed
systemctl cat nginx
systemctl show nginx -p MainPID,Restart,MemoryCurrent
sudo systemctl mask bluetooth
sudo systemctl daemon-reload
systemd-analyze blame
systemctl list-dependencies multi-user.target
```

- `sudo systemctl reload nginx`: re-read config without dropping connections (if supported)
- `sudo systemctl enable --now nginx`: start on boot + start now
- `systemctl --failed`: what is broken
- `systemctl cat nginx`: the unit file and all overrides
- `sudo systemctl mask bluetooth`: forbid starting at all
- `sudo systemctl daemon-reload`: after editing unit files
- `systemd-analyze blame`: what slows the boot

## Your own service

`/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My web app
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=app
Group=app
WorkingDirectory=/opt/myapp
EnvironmentFile=/etc/myapp/env
ExecStart=/opt/myapp/venv/bin/python app.py
Restart=on-failure
RestartSec=5
# Resource limits (cgroups)
MemoryMax=512M
CPUQuota=50%
# Hardening
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/var/lib/myapp

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
journalctl -u myapp -f
systemd-analyze security myapp
```

- `systemd-analyze security myapp`: score of hardening options

`Type=`: `simple` (process stays in foreground: best for most apps), `exec`, `forking` (old daemons that fork), `oneshot` (scripts that finish), `notify` (app tells systemd when ready).

## Changing a packaged service

*Never* edit `/usr/lib/systemd/system/nginx.service`. Create an override:

```bash
sudo systemctl edit nginx
```
```ini
[Service]
LimitNOFILE=65536
Restart=always
```
It is saved to `/etc/systemd/system/nginx.service.d/override.conf` and survives upgrades.

To replace `ExecStart`, clear it first: `ExecStart=` then `ExecStart=/new/command`.

## User services (no root)

```bash
mkdir -p ~/.config/systemd/user
systemctl --user enable --now myjob.service
loginctl enable-linger $USER
```

- `loginctl enable-linger $USER`: keep running after logout

## Practice

1. Write a service for `python3 -m http.server 8000` running as a dedicated user. Enable it, reboot, check it is up.
2. Kill its process with `kill -9`. Watch systemd restart it (`journalctl -u`).
3. Add `MemoryMax=50M` and check it with `systemctl show -p MemoryMax`.
4. Override nginx to `Restart=always` with `systemctl edit`. Find the file it created.
5. Run `systemd-analyze security` on your service and improve the score.

## Check yourself

- Why must you not edit unit files in `/usr/lib/systemd/system`?
- What is the difference between `enable` and `start`?
- When do you need `daemon-reload`?

## Next
[Lesson 17. Logs](17-logs.md)
