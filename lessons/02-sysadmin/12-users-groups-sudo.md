# Lesson 12. Users, groups, sudo

## Goal
Create and manage accounts, and give admin rights without sharing the root password.

## Theory

- Every process runs as a **user** (UID) and a primary **group** (GID), plus supplementary groups. Permissions are checked against these numbers, not names.
- **root** has UID 0 and bypasses permission checks.
- **System users** (UID < 1000 on most distros) run services: `www-data`, `postgres`, `sshd`. They usually have no password and no login shell.
- **sudo** runs one command as another user (root by default) after checking `/etc/sudoers`. Every use is logged. Nobody needs the root password.

### Account files

| File | Contents |
|---|---|
| `/etc/passwd` | `name:x:UID:GID:comment:home:shell` — readable by all |
| `/etc/shadow` | password hashes, expiry — root only |
| `/etc/group` | `group:x:GID:members` |
| `/etc/sudoers`, `/etc/sudoers.d/` | who may run what |
| `/etc/skel/` | template files copied into new home directories |

## Commands

```bash
id                          # my UID, GID, groups
id alice
whoami; groups
getent passwd alice         # works with LDAP/SSSD too, not only local files
sudo useradd -m -s /bin/bash -c "Alice" alice   # low-level, all distros
sudo adduser bob            # interactive, Debian/Ubuntu
sudo passwd alice           # set password
sudo usermod -aG sudo alice # add to group (-a = append! without it you REPLACE groups)
sudo usermod -aG docker alice
sudo gpasswd -d alice docker   # remove from group
sudo groupadd devs
sudo userdel -r bob         # delete with home directory
sudo chage -l alice         # password expiry info
sudo passwd -l alice        # lock account
sudo usermod -s /usr/sbin/nologin svc   # no interactive login
su - alice                  # switch user (needs alice's password)
sudo -i                     # root login shell
sudo -u postgres psql       # run as another user
sudo -l                     # what am I allowed to run?
last; lastlog; who; w       # who logged in, who is here now
```

New group membership applies to **new** logins. Run `newgrp docker` or log out and in.

### sudoers

Always edit with `visudo` — it checks syntax. A broken sudoers file can lock you out of sudo.

```bash
sudo visudo -f /etc/sudoers.d/deploy
```

```
# user  hosts=(run-as) commands
alice   ALL=(ALL:ALL) ALL
%devs   ALL=(ALL) /usr/bin/systemctl restart myapp, /usr/bin/journalctl -u myapp
deploy  ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp
```

Least privilege: grant exact commands, not `ALL`. Note that some commands are an escape to a full shell (`vim`, `less`, `find -exec`, `tar`) — see GTFOBins.

## Practice

1. Create users `dev1`, `dev2` and group `devs`; add both to it.
2. Allow `devs` to run only `systemctl status *` and `journalctl` with sudo.
3. Log in as `dev1`, check `sudo -l`, try `sudo apt update` (should fail).
4. Run `usermod -G devs dev1` (without `-a`) on a test user and see what happened to its groups.
5. Create a service user `app` with no home login: `useradd -r -s /usr/sbin/nologin app`.

## Check yourself

- Why is `usermod -G` without `-a` dangerous?
- Why use `visudo` instead of a normal editor?
- Where are password hashes stored and who can read them?

## Next
[Lesson 13. Permissions](13-permissions.md)
