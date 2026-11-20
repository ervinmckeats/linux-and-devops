# Lesson 33. Server hardening

## Goal
Take a fresh internet-facing server to a sane security baseline in 15 minutes, and know why each step matters.

## Threat model first

Who attacks a random small server? Mostly **bots**: they scan the whole internet for open SSH with weak passwords, outdated software with known CVEs, exposed databases and admin panels. Defending against them covers 90% of real risk:

1. Patch quickly.
2. No password logins; keys only.
3. Expose only the ports you need.
4. Least privilege everywhere.
5. Know when something changes (logs, alerts).

## 1. Updates

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
cat /var/run/reboot-required 2>/dev/null
# RHEL: sudo dnf install dnf-automatic && sudo systemctl enable --now dnf-automatic.timer
```

- `sudo dpkg-reconfigure -plow unattended-upgrades`: security updates automatically
- `cat /var/run/reboot-required 2>/dev/null`: kernel updates need a reboot

## 2. A personal admin user

```bash
sudo adduser admin
sudo usermod -aG sudo admin
sudo rsync -a --chown=admin:admin ~/.ssh/ /home/admin/.ssh/
```

- `sudo rsync -a --chown=admin:admin ~/.ssh/ /home/admin/.ssh/`: copy your authorized_keys

Log in as `admin` **in a new terminal** and check `sudo -v` before going on.

## 3. SSH server

`/etc/ssh/sshd_config.d/10-hardening.conf`:

```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 20
X11Forwarding no
AllowUsers admin deploy
ClientAliveInterval 300
ClientAliveCountMax 2
```

```bash
sudo sshd -t && sudo systemctl reload ssh
```

- `sudo sshd -t && sudo systemctl reload ssh`: unit is "sshd" on RHEL

Keep your current session open and test a **new** login before closing it.

Changing the port (e.g. 2222) reduces log noise from bots but is not real security. If you change it, open the new port in the firewall first.

## 4. Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80,443/tcp
sudo ufw enable
```

Plus the cloud security group: same rules, SSH ideally only from your IP or a VPN/bastion.

## 5. fail2ban

Bans IPs after repeated failed logins.

`/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
backend  = systemd

[sshd]
enabled = true
```

```bash
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip 203.0.113.5
```

## 6. What listens?

```bash
sudo ss -tulpn
```

Everything that does not need the internet should listen on `127.0.0.1` or a private IP: databases, Redis, admin UIs, metrics. Remove services you do not use: `sudo apt purge <pkg>`, `systemctl disable --now <svc>`.

## 7. Least privilege

- Each service runs as its own user (lesson 12), never root.
- systemd sandboxing (lesson 16): `NoNewPrivileges`, `ProtectSystem`, `PrivateTmp`.
- `sudo` for exact commands, not `ALL` (lesson 12).
- Files with secrets: `600`, owned by the service user.

## 8. Quick audit

```bash
sudo lastb | head
last -n 20
sudo journalctl -u ssh --since today | grep -i -E "accepted|failed"
sudo find / -xdev -perm -4000 -type f 2>/dev/null
sudo find / -xdev -perm -0002 -type f 2>/dev/null
awk -F: '$3 == 0' /etc/passwd
sudo awk -F: '$2 == ""' /etc/shadow
sudo apt install -y lynis && sudo lynis audit system
```

- `sudo lastb | head`: failed logins
- `last -n 20`: successful logins
- `sudo find / -xdev -perm -4000 -type f 2>/dev/null`: SUID files
- `sudo find / -xdev -perm -0002 -type f 2>/dev/null`: world-writable files
- `awk -F: '$3 == 0' /etc/passwd`: only root should have UID 0
- `sudo awk -F: '$2 == ""' /etc/shadow`: accounts without password
- `sudo apt install -y lynis && sudo lynis audit system`: full audit with suggestions

## 9. Time, logs, backups

- Time sync: `timedatectl` should show `System clock synchronized: yes`. Wrong time breaks TLS, logs and 2FA.
- Ship logs off the server (lesson 55): an attacker with root can edit local logs.
- Backups with an off-site, immutable copy (lesson 21).

## Checklist for a new server

- [ ] system updated, automatic security updates on
- [ ] personal user with sudo, key login tested
- [ ] root login and password auth disabled
- [ ] firewall: default deny, only needed ports; cloud security group matches
- [ ] fail2ban active
- [ ] no unexpected listening ports
- [ ] services run as dedicated users
- [ ] time synchronized
- [ ] logs shipped, backups running and restore-tested
- [ ] monitoring and alerting for the host

## Automate it

Doing this by hand on every server leads to drift. Turn the checklist into a script, then into an Ansible role (lesson 45). A minimal idempotent script skeleton:

```bash
#!/usr/bin/env bash
# harden.sh - baseline for a fresh Ubuntu server. Test in a VM first.
set -Eeuo pipefail
ADMIN_USER="${ADMIN_USER:?set ADMIN_USER}"
SSH_PORT="${SSH_PORT:-22}"

[[ $EUID -eq 0 ]] || { echo "run as root" >&2; exit 1; }
[[ -s "/home/$ADMIN_USER/.ssh/authorized_keys" ]] || { echo "no SSH key for $ADMIN_USER" >&2; exit 1; }

apt-get update -q
DEBIAN_FRONTEND=noninteractive apt-get -y -q full-upgrade
DEBIAN_FRONTEND=noninteractive apt-get -y -q install unattended-upgrades ufw fail2ban

id -nG "$ADMIN_USER" | grep -qw sudo || usermod -aG sudo "$ADMIN_USER"

cat > /etc/ssh/sshd_config.d/10-hardening.conf <<CONF
Port $SSH_PORT
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
MaxAuthTries 3
AllowUsers $ADMIN_USER
CONF
sshd -t

ufw default deny incoming
ufw default allow outgoing
ufw allow "$SSH_PORT"/tcp
ufw --force enable

printf '[sshd]\nenabled = true\nbackend = systemd\n' > /etc/fail2ban/jail.local
systemctl enable --now fail2ban
systemctl reload ssh
echo "done. Test a NEW ssh session as $ADMIN_USER on port $SSH_PORT before closing this one."
```

## Practice

1. Harden a fresh VM by hand using the checklist.
2. From another VM, try to log in as root and with a password. Both must fail.
3. Trigger fail2ban with wrong logins and unban yourself.
4. Run `lynis audit system`, fix three suggestions.
5. Destroy the VM, create a new one and run the script above. Compare with your manual result.

## Check yourself

- Why does changing the SSH port not count as real protection?
- Why keep the old SSH session open while changing sshd config?
- Why ship logs off the server?

## Next
[Lesson 34. SELinux, AppArmor, auditd, capabilities](34-mac-audit-capabilities.md)
