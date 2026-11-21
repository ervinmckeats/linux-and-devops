# Lesson 63. Real-world scenarios

## Goal
Practice complete tasks the way they come at work: a goal, constraints, and several lessons combined. Each scenario lists the steps; do them in your lab and keep notes.

## Scenario 1. New Ubuntu server in 15 minutes

**Task:** a fresh cloud VM must be safe to expose to the internet.

```bash
sudo apt update && sudo apt full-upgrade -y
sudo adduser admin && sudo usermod -aG sudo admin
# on YOUR machine:
# test key login
ssh-copy-id admin@SERVER && ssh admin@SERVER

# on the server, only after the key login works:
printf 'PermitRootLogin no\nPasswordAuthentication no\n' | sudo tee /etc/ssh/sshd_config.d/10-hardening.conf
sudo sshd -t && sudo systemctl reload ssh
sudo ufw allow OpenSSH && sudo ufw enable
sudo apt install -y fail2ban unattended-upgrades
sudo timedatectl set-timezone UTC
sudo hostnamectl set-hostname web-01
```

**Done when:** root and password logins fail, only port 22 is open, updates are automatic. → [lesson 33](../05-security/33-server-hardening.md)

## Scenario 2. Website on nginx with free HTTPS

**Task:** serve `site.example.com` from `/var/www/site` with HTTPS and HTTP→HTTPS redirect.

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
sudo mkdir -p /var/www/site && echo '<h1>ok</h1>' | sudo tee /var/www/site/index.html
sudo tee /etc/nginx/sites-available/site.conf > /dev/null <<'NGINX'
server {
    listen 80;
    server_name site.example.com;
    root /var/www/site;
    index index.html;
}
NGINX
sudo ln -s /etc/nginx/sites-available/site.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo ufw allow 'Nginx Full'
sudo certbot --nginx -d site.example.com
curl -I https://site.example.com
```

**Done when:** `curl -I http://...` returns 301 to HTTPS, the certificate is valid, `certbot renew --dry-run` passes. → [lesson 27](../03-networking/27-http-tls-nginx.md)

## Scenario 3. Nightly backup with rotation, off-site copy and restore test

**Task:** back up `/etc`, `/var/www` and a PostgreSQL database every night, keep 14 days, copy off-site, prove restore works.

1. Script `/usr/local/bin/backup.sh` with `set -Eeuo pipefail`, `pg_dump -Fc`, `tar --zstd`, retention with `find -mtime +14 -delete`.
2. Off-site: `restic` or `rsync` to another server / object storage.
3. systemd service + timer with `Persistent=true`.
4. Alert if the backup did not succeed in 26 hours (dead-man's switch).
5. Restore into a fresh VM and start the app on it.

→ [lessons 18](../02-sysadmin/18-cron-and-timers.md), [21](../02-sysadmin/21-archives-backups.md)

## Scenario 4. "Who is hammering us?": nginx log analysis

**Task:** the site is slow; find out whether it is an attack, a bot or a real traffic spike.

```bash
LOG=/var/log/nginx/access.log

# top IPs
awk '{print $1}' $LOG | sort | uniq -c | sort -rn | head

# status codes
awk '{print $9}' $LOG | sort | uniq -c | sort -rn

# top paths
awk '{print $7}' $LOG | sort | uniq -c | sort -rn | head

# user agents
awk -F'"' '{print $6}' $LOG | sort | uniq -c | sort -rn | head

# requests per minute
awk '{print substr($4,2,17)}' $LOG | uniq -c | tail -30
```

**Then:** block an abusive IP (`ufw deny from IP`), add `limit_req` in nginx, or scale up if it is real traffic. → [lessons 11](../01-terminal/11-text-processing.md), [27](../03-networking/27-http-tls-nginx.md)

## Scenario 5. Add a disk and move data to it without downtime surprises

**Task:** `/var/lib/app` is almost full. Add a new disk and move the data there.

1. Attach the disk; `lsblk`; partition; `mkfs.ext4`; mount at `/mnt/new`.
2. `rsync -aHAX /var/lib/app/ /mnt/new/` while the app runs (first pass).
3. Stop the app; second `rsync` pass (only differences, fast).
4. Mount the new disk at `/var/lib/app` via UUID in `/etc/fstab`; `mount -a`.
5. Start the app; verify; keep the old data *until you are sure*.

Better long-term: put it on LVM so next time you just `lvextend -r`. → [lessons 19](../02-sysadmin/19-disks-filesystems.md), [20](../02-sysadmin/20-lvm-raid.md)

## Scenario 6. Migrate a site to a new server

1. Lower the DNS TTL to 300 s a day before.
2. Build the new server with your Ansible playbook (not by hand).
3. Copy data with `rsync` repeatedly while the old server is live; copy the database with a dump and restore (or replication).
4. Test the new server with `curl --resolve site.com:443:NEW_IP https://site.com/`.
5. Maintenance window: stop writes, last rsync / final dump, switch DNS.
6. Watch logs and metrics on both; keep the old server for a few days; then raise TTL back.

→ [lessons 23](../03-networking/23-network-basics.md), [45](../08-devops/45-ansible.md)

## Scenario 7. Alerts from scripts to a chat

**Task:** a health-check script sends a message to a chat (Slack, Telegram, Matrix, Teams) when a site is down.

```bash
#!/usr/bin/env bash
# /usr/local/bin/healthcheck.sh - run from a systemd timer every 5 minutes
set -euo pipefail

# WEBHOOK_URL=..., URLS="https://a https://b"; chmod 600
source /etc/default/healthcheck

for url in $URLS; do
    if ! curl -fsS --max-time 10 -o /dev/null "$url"; then
        curl -fsS -X POST -H 'Content-Type: application/json' \
             -d "{\"text\":\"DOWN: $url from $(hostname) at $(date -u +%FT%TZ)\"}" \
             "$WEBHOOK_URL" > /dev/null
    fi
done
```

Improve it: alert only after 2 failures in a row (state file), send "RECOVERED" messages. In real systems use Prometheus + blackbox exporter + Alertmanager instead. → [lesson 56](../10-sre/56-prometheus-grafana.md)

## Scenario 8. Log rotation for your own application

```
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 myapp myapp
    postrotate
        systemctl kill -s HUP myapp.service
    endscript
}
```

Test with `logrotate -d`, then `-f`; check the app writes into the new file (`ls -l`, `lsof`). → [lesson 17](../02-sysadmin/17-logs.md)

## Scenario 9. Containerize and ship an app with CI/CD

1. Multi-stage Dockerfile, non-root user, health check.
2. Compose file for local development with the database.
3. CI pipeline: lint, test, build image tagged with commit SHA, scan, push.
4. Deploy to a server with `docker compose pull && up -d`, or to Kubernetes.

→ [lessons 42](../07-containers/42-dockerfile.md), [43](../07-containers/43-compose-networks-volumes.md), [44](../08-devops/44-cicd.md)

## Scenario 10. Kubernetes app with GitOps, monitoring and an SLO

1. Helm chart or Kustomize for the app with requests/limits, probes, PDB, HPA.
2. Argo CD Application pointing at a config repo.
3. kube-prometheus-stack; ServiceMonitor for the app; RED dashboard.
4. SLO with burn-rate alert and a runbook.
5. Game day: deploy a bad version, get paged, roll back via Git, write a postmortem.

→ Modules [9](../09-kubernetes/49-k8s-architecture.md) and [10](../10-sre/55-observability.md)

## Next
[Lesson 64. Labs with solutions](64-labs.md)
