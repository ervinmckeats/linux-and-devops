# Lesson 61. Troubleshooting playbook: "what to do if..."

## Goal
A quick reference for the most common production problems: symptom → first commands → usual causes. Each entry links to the lesson with details.

## General method

1. **What exactly is broken, for whom, since when?** Get a precise symptom.
2. **What changed?** Deploys, config, traffic, infrastructure, certificates, time.
3. **Check from the outside in**: user → DNS → load balancer → app → dependencies → host.
4. **One hypothesis at a time**, write down what you tried.
5. **Mitigate first** (rollback, failover, scale), then find the root cause.

## The server is slow

```bash
uptime; dmesg -T | tail
vmstat 1 5; mpstat -P ALL 1 3; iostat -xz 1 3; free -h
pidstat 1 3; top
cat /proc/pressure/{cpu,memory,io}
```
CPU-bound app, I/O wait, swapping, steal time, one hot core. → [lesson 39](../06-internals/39-performance.md)

## Disk is full

```bash
df -h; df -i
sudo du -xh --max-depth=1 / | sort -h | tail
sudo lsof +L1
sudo journalctl --vacuum-size=500M; docker system df
```
Logs without rotation, deleted-but-open files, Docker images, inodes exhausted. → [lessons 8](../01-terminal/08-inodes-and-links.md), [19](../02-sysadmin/19-disks-filesystems.md)

## A process was killed / out of memory

```bash
sudo dmesg -T | grep -i -E 'oom|killed process'
journalctl -k -b | grep -i oom
free -h; ps aux --sort=-rss | head
```
Host OOM vs cgroup limit (`MemoryMax`, container limit). → [lesson 37](../06-internals/37-memory-oom.md)

## A service will not start

```bash
systemctl status app
journalctl -u app -b --no-pager | tail -50
systemctl cat app
sudo -u app /path/to/binary --config /etc/app/config
```

- `sudo -u app /path/to/binary --config /etc/app/config`: run by hand as the service user
Wrong path/permissions, port already in use, missing env file, SELinux. → [lesson 16](../02-sysadmin/16-systemd.md)

## Port already in use

```bash
sudo ss -tlnp | grep :8080
sudo lsof -i :8080
```

## "Permission denied"

```bash
id; namei -l /full/path; ls -l file; getfacl file; lsattr file
mount | grep noexec
sudo ausearch -m avc -ts recent
sudo journalctl -k | grep -i apparmor
```

- `sudo ausearch -m avc -ts recent`: SELinux
→ [lessons 13](../02-sysadmin/13-permissions.md), [34](../05-security/34-mac-audit-capabilities.md)

## "Too many open files"

```bash
cat /proc/$(pgrep -o nginx)/limits | grep 'open files'
ls /proc/PID/fd | wc -l
```
Raise `LimitNOFILE` in the systemd unit, not only in `limits.conf`. → [lesson 36](../06-internals/36-kernel-sysctl-limits.md)

## Cannot connect to a host/port

```bash
getent hosts host; dig +short host
ip route get <ip>
nc -zv host port
sudo ss -tlnp
sudo ufw status; sudo nft list ruleset
sudo tcpdump -ni any port <port>
```

- `nc -zv host port`: refused vs timeout
- `sudo ss -tlnp`: on the server: listening? on 0.0.0.0 or 127.0.0.1?
- `sudo ufw status; sudo nft list ruleset`: plus cloud security groups
→ [lessons 23](../03-networking/23-network-basics.md), [24](../03-networking/24-network-tools.md), [26](../03-networking/26-firewalls.md)

## DNS problems

```bash
resolvectl status; cat /etc/resolv.conf
dig name; dig @1.1.1.1 name; dig +trace name
getent hosts name
```
Stale cache/TTL, wrong resolver, split DNS, `/etc/hosts` override, search domains.

## SSH does not work

```bash
ssh -v user@host
sudo journalctl -u ssh -f
sudo sshd -t
ls -ld ~/.ssh; ls -l ~/.ssh/authorized_keys
```

- `sudo journalctl -u ssh -f`: on the server
Permissions, wrong user, password auth disabled, firewall/port, host key changed. → [lesson 25](../03-networking/25-ssh.md)

## nginx returns 502 / 504

```bash
sudo tail -50 /var/log/nginx/error.log
ss -tlnp | grep <upstream port>
curl -v http://127.0.0.1:<port>/health
```
App down, listening on another address, too slow (raise timeout or fix the app), SELinux. → [lesson 27](../03-networking/27-http-tls-nginx.md)

## TLS certificate errors

```bash
echo | openssl s_client -connect host:443 -servername host 2>/dev/null | openssl x509 -noout -dates -subject -ext subjectAltName
sudo certbot certificates; systemctl list-timers | grep certbot
timedatectl
```
Expired, missing intermediate, wrong name, wrong system time.

## Cron job did not run

```bash
grep CRON /var/log/syslog | tail; journalctl -u cron --since today
systemctl list-timers
```
PATH, `%` not escaped, no output redirect, server was off. → [lesson 18](../02-sysadmin/18-cron-and-timers.md)

## Server does not boot

Serial/console → GRUB `e` → `systemd.unit=emergency.target` or `init=/bin/bash`, check `/etc/fstab`, or chroot from a live ISO. → [lesson 22](../02-sysadmin/22-boot-and-recovery.md)

## Container exits immediately

```bash
docker ps -a; docker logs c; docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' c
docker run -it --entrypoint sh image
```
→ [lesson 41](../07-containers/41-docker-basics.md)

## Kubernetes pod is not healthy

```bash
kubectl get pods -o wide; kubectl describe pod P; kubectl logs P --previous
kubectl get events --sort-by=.lastTimestamp | tail
```
Pending, ImagePullBackOff, CrashLoopBackOff, OOMKilled, not Ready. → [lesson 54](../09-kubernetes/54-k8s-troubleshooting.md)

## Error rate jumped after a deploy

Roll back first: `kubectl rollout undo`, revert in Git (GitOps), or redeploy the previous artifact. Then compare versions, configs and logs. → [lessons 44](../08-devops/44-cicd.md), [58](../10-sre/58-incident-response.md)

## Next
[Lesson 62. Dangerous commands](62-dangerous-commands.md)
