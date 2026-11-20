# Lesson 64. Labs with solutions

## Goal
Check your skills on practical tasks. Try each one yourself first; open the solution only after you have an answer. Many tasks have more than one good solution.

## Lab setup

Run this once in an empty directory in your VM. It creates the files the tasks use.

```bash
mkdir -p ~/lab && cd ~/lab

# 1. web server log (nginx combined format), 2000 lines
for i in $(seq 2000); do
  printf '10.0.%d.%d - - [10/Jan/2026:%02d:%02d:%02d +0000] "%s /%s HTTP/1.1" %s %d "-" "%s"\n' \
    $((RANDOM%3)) $((RANDOM%30)) $((10+RANDOM%4)) $((RANDOM%60)) $((RANDOM%60)) \
    $(shuf -n1 -e GET GET GET POST) $(shuf -n1 -e api/users api/orders login static/app.js health) \
    $(shuf -n1 -e 200 200 200 200 201 301 404 500 502) $((RANDOM%9000)) \
    $(shuf -n1 -e curl/8.5 Mozilla/5.0 Googlebot/2.1)
done > access.log

# 2. CSV with employees
cat > staff.csv <<'CSV'
id,name,department,salary,city
1,Alex,devops,5200,Berlin
2,Maria,backend,4800,Lisbon
3,John,devops,6100,Berlin
4,Kim,frontend,4300,Seoul
5,Lena,backend,5600,Berlin
6,Omar,sre,6500,Dubai
7,Ivy,sre,5900,Lisbon
CSV

# 3. config with comments and empty lines
cat > app.conf <<'CONF'
# main settings
port = 8080
host = 0.0.0.0

# performance
workers = 4
# timeout = 30
max_connections = 1000
CONF

# 4. many files of different age and size
mkdir -p files
for i in $(seq 1 30); do head -c $((RANDOM * 30)) /dev/urandom > files/data_$i.bin; done
for i in $(seq 1 10); do touch -d "$((i * 5)) days ago" files/data_$i.bin; done
touch "files/report with spaces.txt" files/-strange-name.txt
```

---

## Level 1: basics

**1.1.** Show all lines of `app.conf` without comments and empty lines.

<details><summary>Solution</summary>

```bash
grep -v -E '^\s*(#|$)' app.conf
```
</details>

**1.2.** Delete the file `files/-strange-name.txt`.

<details><summary>Solution</summary>

```bash
rm -- files/-strange-name.txt
```

- `rm -- files/-strange-name.txt`: or `rm ./files/-strange-name.txt`
`--` tells `rm` that what follows is a file name, not an option.
</details>

**1.3.** Copy `files/report with spaces.txt` to `/tmp` and rename it to `report.txt` on the way.

<details><summary>Solution</summary>

```bash
cp "files/report with spaces.txt" /tmp/report.txt
```
</details>

**1.4.** Find files in `files/` older than 20 days.

<details><summary>Solution</summary>

```bash
find files -type f -mtime +20
```
</details>

**1.5.** Find the 5 largest files in `files/`, show sizes in human form.

<details><summary>Solution</summary>

```bash
ls -lhS files | head -6
# or
du -h files/* | sort -h | tail -5
```
</details>

**1.6.** Create `projects/{api,web}/{src,tests}` in one command and show the tree.

<details><summary>Solution</summary>

```bash
mkdir -p projects/{api,web}/{src,tests} && tree projects
```
</details>

**1.7.** Make a script `hello.sh` that prints `Hello, NAME` (name is the first argument, default `world`) and is executable only by you.

<details><summary>Solution</summary>

```bash
cat > hello.sh <<'SH'
#!/usr/bin/env bash
echo "Hello, ${1:-world}"
SH
chmod 700 hello.sh
./hello.sh Ann
```
</details>

## Level 2: data processing

**2.1.** Top 5 IP addresses in `access.log` by number of requests.

<details><summary>Solution</summary>

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -5
```
</details>

**2.2.** How many requests returned 5xx? What percentage of all requests is that?

<details><summary>Solution</summary>

```bash
awk '$9 ~ /^5/ {e++} END {printf "%d errors, %.1f%%\n", e, e*100/NR}' access.log
```
</details>

**2.3.** Requests per hour (field 4 contains `[10/Jan/2026:HH:...`).

<details><summary>Solution</summary>

```bash
awk '{split($4, t, ":"); print t[2]}' access.log | sort | uniq -c
```
</details>

**2.4.** Total bytes sent (field 10) per status code, sorted by bytes.

<details><summary>Solution</summary>

```bash
awk '{b[$9] += $10} END {for (s in b) print b[s], s}' access.log | sort -rn
```
</details>

**2.5.** Share of requests by user agent, with a bar of `*` characters (one `*` per 50 requests).

<details><summary>Solution</summary>

```bash
awk -F'"' '{print $6}' access.log | sort | uniq -c | sort -rn |
  awk '{bar=""; for (i = 0; i < $1/50; i++) bar = bar "*"; printf "%-14s %5d %s\n", $2, $1, bar}'
```
</details>

**2.6.** From `staff.csv`: average salary per department, highest first.

<details><summary>Solution</summary>

```bash
awk -F, 'NR > 1 {s[$3] += $4; n[$3]++} END {for (d in s) printf "%s %.0f\n", d, s[d]/n[d]}' staff.csv | sort -k2 -rn
```
</details>

**2.7.** Names of people in Berlin earning more than 5000, as a comma-separated list.

<details><summary>Solution</summary>

```bash
awk -F, '$5 == "Berlin" && $4 > 5000 {print $2}' staff.csv | paste -sd,
```
</details>

**2.8.** In `app.conf`, change `workers = 4` to `workers = 8` and uncomment the timeout line, keeping a backup.

<details><summary>Solution</summary>

```bash
sed -i.bak -e 's/^workers = 4$/workers = 8/' -e 's/^# *timeout/timeout/' app.conf
diff app.conf.bak app.conf
```
</details>

## Level 3: scripts and system

**3.1.** Write `cleanup.sh DIR DAYS` that deletes files older than DAYS in DIR, prints what it deleted, refuses to run on `/` and has a dry-run flag `-n`.

<details><summary>Solution</summary>

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
dry=0
while getopts ":n" o; do case $o in n) dry=1 ;; *) exit 2 ;; esac; done
shift $((OPTIND - 1))
dir=${1:?usage: cleanup.sh [-n] DIR DAYS}
days=${2:?usage: cleanup.sh [-n] DIR DAYS}
[[ $days =~ ^[0-9]+$ ]] || { echo "DAYS must be a number" >&2; exit 2; }
real=$(realpath -- "$dir")
[[ $real != / ]] || { echo "refusing to run on /" >&2; exit 1; }
if (( dry )); then
    find "$real" -type f -mtime +"$days" -print
else
    find "$real" -type f -mtime +"$days" -print -delete
fi
```
</details>

**3.2.** Keep only the 3 newest backup files out of 12. Prepare:
`for i in $(seq 1 12); do touch -d "$i days ago" backup_$i.tar.gz; done`

<details><summary>Solution</summary>

```bash
ls -1t backup_*.tar.gz | tail -n +4 | xargs -r rm --
ls -lt backup_*.tar.gz
```
`ls -t` sorts newest first; `tail -n +4` skips the first three. (Fine here because names have no spaces; with arbitrary names use `find -printf '%T@ %p\0'` and `sort -z`.)
</details>

**3.3.** A script that checks a list of hosts and ports from a file (`host port` per line) and prints `OK`/`FAIL`, exiting non-zero if anything failed.

<details><summary>Solution</summary>

```bash
#!/usr/bin/env bash
set -uo pipefail
rc=0
while read -r host port; do
    [[ -z ${host:-} || $host == \#* ]] && continue
    if timeout 3 bash -c "</dev/tcp/$host/$port" 2>/dev/null; then
        echo "OK   $host:$port"
    else
        echo "FAIL $host:$port"; rc=1
    fi
done < "${1:?usage: check.sh FILE}"
exit $rc
```
</details>

**3.4.** A process ignores `kill`. Reproduce with `bash -c 'trap "" TERM; sleep 1000' &` and stop it. What signal did you need and why?

<details><summary>Solution</summary>

```bash
kill %1
kill -9 %1
```

- `kill %1`: SIGTERM is ignored by the trap
- `kill -9 %1`: SIGKILL cannot be caught or ignored
</details>

**3.5.** You deleted a 500 MB log, but `df` still shows the space as used. Reproduce and fix without a reboot.

<details><summary>Solution</summary>

```bash
fallocate -l 500M /tmp/big.log
tail -f /tmp/big.log > /dev/null &
rm /tmp/big.log; df -h /tmp
sudo lsof +L1 | grep big.log
kill %1
df -h /tmp
```

- `sudo lsof +L1 | grep big.log`: the process still holds it
- `kill %1`: or restart the service; or truncate via `/proc/PID/fd/N`
</details>

**3.6.** Turn `python3 -m http.server 8000` into a systemd service running as a dedicated user, starting on boot, restarting on failure, limited to 100 MB of memory.

<details><summary>Solution</summary>

```bash
sudo useradd -r -s /usr/sbin/nologin web
sudo tee /etc/systemd/system/miniweb.service > /dev/null <<'UNIT'
[Unit]
Description=Mini web server
After=network-online.target

[Service]
User=web
WorkingDirectory=/tmp
ExecStart=/usr/bin/python3 -m http.server 8000
Restart=on-failure
MemoryMax=100M
NoNewPrivileges=yes

[Install]
WantedBy=multi-user.target
UNIT
sudo systemctl daemon-reload
sudo systemctl enable --now miniweb
systemctl status miniweb; curl -I localhost:8000
```
</details>

**3.7.** Run the `miniweb` service every hour as a one-shot job instead: which two unit files do you need, and how do you check the next run time?

<details><summary>Solution</summary>

A `.service` with `Type=oneshot` and a `.timer` with `OnCalendar=hourly` (+ `Persistent=true`), then `systemctl enable --now name.timer` and `systemctl list-timers`.
</details>

## Level 4: containers and Kubernetes

**4.1.** Run nginx in Docker, reachable only from the host on port 8080, with your own `index.html` mounted read-only.

<details><summary>Solution</summary>

```bash
mkdir -p site && echo '<h1>lab</h1>' > site/index.html
docker run -d --name lab-nginx -p 127.0.0.1:8080:80 -v "$PWD/site":/usr/share/nginx/html:ro nginx:1.27
curl localhost:8080
```
</details>

**4.2.** A container exits with code 137. How do you find out whether it was OOM-killed?

<details><summary>Solution</summary>

```bash
docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' NAME
# Kubernetes:
kubectl describe pod POD | grep -A5 'Last State'
```
</details>

**4.3.** A Kubernetes Deployment is updated, but the Service returns 503. List the checks in order.

<details><summary>Solution</summary>

1. `kubectl get pods -l app=...`: are new pods Running and Ready?
2. `kubectl describe pod`: readiness probe failing? CrashLoopBackOff?
3. `kubectl get endpointslices -l kubernetes.io/service-name=...`: any endpoints?
4. Selector labels in the Service match the pod template labels?
5. `targetPort` matches the container port? App listens on `0.0.0.0`?
6. Roll back with `kubectl rollout undo` if users are affected, then continue debugging.
</details>

**4.4.** Write the `resources` block for a container that normally uses 300 MB and 0.2 CPU, with peaks to 450 MB, for a latency-sensitive API.

<details><summary>Solution</summary>

```yaml
resources:
  requests:
    cpu: 200m
    memory: 350Mi
  limits:
    memory: 600Mi
# no CPU limit to avoid throttling; requests reflect normal use, memory limit covers peaks with headroom
```
</details>

## Open tasks (no solutions)

1. Write a script that finds duplicate files in a directory by checksum and prints groups.
2. Make a weekly report from `access.log`: total requests, unique IPs, error rate, top 10 slow paths (if you add a response time field).
3. Build a two-VM setup: a bastion and a private web server reachable only via `ProxyJump`. Automate it with Ansible.
4. Write a Prometheus alert for "more than 3 container restarts in 15 minutes" and test it with a crashing pod.
5. Take one real incident (yours or a public postmortem from a well-known company) and write a 1-page summary: impact, cause, contributing factors, fixes.

## Next
[Lesson 65. Capstone projects](65-capstone-projects.md)
