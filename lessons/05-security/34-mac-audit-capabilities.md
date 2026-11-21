# Lesson 34. SELinux, AppArmor, auditd, capabilities

## Goal
Understand the security layers beyond file permissions, and stop "just disable SELinux" as a reflex.

## DAC vs MAC

- **DAC** (discretionary access control): normal permissions (lesson 13). The owner decides, and root bypasses everything.
- **MAC** (mandatory access control): a system-wide policy that even root processes must follow. If nginx is compromised, MAC still stops it from reading `/home` or opening random ports.

| | SELinux | AppArmor |
|---|---|---|
| Distros | RHEL, Fedora, Rocky, Alma, Android | Ubuntu, Debian, SUSE |
| Model | labels on every file and process | profiles by program path |
| Feel | powerful, more complex | simpler |

## SELinux

Every process and file has a **context** (label): `user:role:type:level`. The policy says which process types may access which file types. The **type** is what matters day to day.

```bash
# Enforcing / Permissive / Disabled
getenforce

sestatus

# file labels: httpd_sys_content_t
ls -Z /var/www/html

# process label: httpd_t
ps -eZ | grep nginx

# permissive until reboot (logs, does not block) - for debugging only
sudo setenforce 0
```

### The usual problem and the right fix

You put a site in `/srv/site`; nginx gets 403 even though permissions are fine.

```bash
# find the denial
sudo ausearch -m avc -ts recent

# human explanation (setroubleshoot)
sudo sealert -a /var/log/audit/audit.log

# label the directory correctly - permanently
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/site(/.*)?"
sudo restorecon -Rv /srv/site
```

Other common fixes:

```bash
getsebool -a | grep httpd

# nginx as reverse proxy to an app
sudo setsebool -P httpd_can_network_connect 1

# allow a web server on a new port
sudo semanage port -a -t http_port_t -p tcp 8081
```

`chcon` changes a label temporarily (lost on relabel); `semanage fcontext` + `restorecon` is permanent.

Rule: **find the denial, fix the label or boolean**. Disabling SELinux removes a whole layer of protection for every service.

## AppArmor

```bash
# loaded profiles, enforce/complain
sudo aa-status

ls /etc/apparmor.d/

# log only
sudo aa-complain /usr/sbin/nginx

sudo aa-enforce /usr/sbin/nginx

# denials ("apparmor=DENIED")
sudo journalctl -k | grep -i apparmor

# build a profile interactively (apparmor-utils)
sudo aa-genprof /usr/local/bin/myapp
```

Docker and snap use AppArmor profiles on Ubuntu.

## auditd: who touched what

The kernel audit system logs syscalls and file access according to rules.

```bash
sudo apt install -y auditd

# watch writes/attribute changes
sudo auditctl -w /etc/passwd -p wa -k identity

sudo auditctl -w /etc/ssh/sshd_config -p wa -k sshd

# every command run as root
sudo auditctl -a always,exit -F arch=b64 -S execve -F euid=0 -k root-cmds

sudo auditctl -l

# -i: readable names
sudo ausearch -k identity -i

sudo aureport --summary
```

Make rules permanent in `/etc/audit/rules.d/*.rules`, then `sudo augenrules --load`.

Logging every `execve` is noisy; in production scope rules carefully and ship logs to central storage.

## Capabilities: slices of root

Root's power is split into ~40 **capabilities**. Give a program only the one it needs instead of running it as root.

| Capability | Allows |
|---|---|
| `CAP_NET_BIND_SERVICE` | bind ports < 1024 |
| `CAP_NET_ADMIN` | configure network, firewall |
| `CAP_NET_RAW` | raw sockets (ping, tcpdump) |
| `CAP_SYS_ADMIN` | a huge set; almost root. Avoid |
| `CAP_CHOWN`, `CAP_DAC_OVERRIDE` | bypass file ownership/permissions |
| `CAP_SYS_PTRACE` | trace other processes |

```bash
# cap_net_raw=ep
getcap /usr/bin/ping

# bind :80 without root
sudo setcap 'cap_net_bind_service=+ep' /opt/app/server

# capabilities of a process
getpcaps $$

grep Cap /proc/$$/status; capsh --decode=0000003fffffffff
```

In systemd:

```ini
[Service]
User=app
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
```

In containers (lesson 42): `docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE ...`. `--privileged` gives everything: avoid it.

## Practice

1. (Rocky/Alma VM) Serve nginx from `/srv/site`, get the 403, find the AVC denial, fix it with `semanage fcontext` + `restorecon`.
2. (Ubuntu VM) Put nginx's AppArmor profile in complain mode and back.
3. Add an audit rule on `/etc/sudoers`, edit it with `visudo`, find the event with `ausearch`.
4. Let a Python server bind port 80 as a normal user using capabilities (on a copy of the binary or via systemd `AmbientCapabilities`).

## Check yourself

- Why does MAC help even when an attacker gets the nginx process?
- `chcon` vs `semanage fcontext`: which survives a relabel?
- Why is `CAP_SYS_ADMIN` almost as dangerous as root?

## Next
[Lesson 35. Secrets and encryption](35-secrets-encryption.md)
