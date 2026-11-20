# Lesson 66. Interview questions

## Goal
Prepare for Linux, DevOps and SRE interviews. Short answers point to the lesson with details. Practice saying answers out loud: interviewers care about **reasoning**, not memorized definitions.

## Linux

**What happens when you type `ls -l` and press Enter?**
The shell parses the line, expands globs/variables, finds `ls` (alias → builtin → `$PATH`), forks a child, the child `execve`s `/usr/bin/ls`, which makes syscalls (`openat`, `getdents64`, `statx`, `write`); the shell waits and reads the exit code. → lessons 3, 14, 40

**Hard link vs symlink?**
Hard link = another name for the same inode, same filesystem only. Symlink = file containing a path; can cross filesystems; breaks if the target is removed. → lesson 8

**`df` shows 100% but `du` finds much less. Why?**
Deleted files still held open by a process (`lsof +L1`), or files hidden under a mount point; also reserved blocks. If `df -i` is 100%, inodes are exhausted. → lessons 8, 19

**What is load average?**
Average number of tasks running or waiting (including uninterruptible I/O wait) over 1/5/15 min. Compare with number of CPUs. → lesson 14

**Process states? What is a zombie?**
R, S, D, Z, T. A zombie has exited but its parent has not read its exit status; it uses no memory but a PID. Fix the parent. → lesson 14

**SIGTERM vs SIGKILL?**
TERM can be handled for graceful shutdown; KILL cannot and gives no cleanup. → lesson 14

**How does the system boot?**
Firmware → GRUB → kernel + initramfs → mount root → systemd → default target. → lesson 22

**Free memory is low. Problem?**
Not necessarily: look at `available`; page cache is reclaimable. Problem signs: swapping in/out, OOM kills, memory PSI. → lesson 37

**How does the OOM killer choose a victim?**
By `oom_score` (mostly memory use, adjusted by `oom_score_adj`). In containers, the cgroup limit triggers OOM inside that cgroup. → lesson 37

**What are namespaces and cgroups?**
Namespaces isolate what a process sees (PID, net, mount, UTS, IPC, user); cgroups limit what it uses (CPU, memory, I/O, PIDs). Containers combine them. → lesson 38

**How do you find what is slow on a server?**
USE method; first 60 seconds: `uptime`, `dmesg`, `vmstat`, `mpstat`, `pidstat`, `iostat`, `free`, `sar`. → lesson 39

**`chmod 755` on a directory vs a file?**
On a file: `x` = execute. On a directory: `r` = list, `w` = create/delete entries, `x` = enter/access. → lesson 13

**What do SUID and the sticky bit do?**
SUID runs a file as its owner (`passwd`). Sticky bit on a directory: only owners delete their files (`/tmp`). → lesson 13

## Networking

**What happens when you open `https://example.com` in a browser?**
DNS resolution (cache, resolver, recursive lookup), TCP handshake to the IP, TLS handshake with SNI and certificate validation, HTTP request, server processing (load balancer → app → DB), response, rendering. Be ready to go deep on any step. → lessons 23, 27

**TCP vs UDP?** → lesson 23

**"Connection refused" vs "timeout"?**
Refused: host reached, nothing listening or rejected. Timeout: packets dropped — firewall, routing, host down. → lesson 23

**How do you debug "service A cannot reach service B"?**
Bottom-up: DNS → route → port reachable (`nc`) → listening address (`ss`) → firewalls/security groups/NetworkPolicy → `tcpdump` on both sides → app logs. → lesson 24

**502 vs 503 vs 504?** → lesson 27

**What is a CIDR /24? How many hosts in /20?**
/24 = 256 addresses (254 usable); /20 = 4096 (4094 usable).

## Containers and Kubernetes

**Container vs VM?** → lessons 41, 48

**How do you make a Docker image smaller and safer?**
Small base, multi-stage, `.dockerignore`, combine `RUN` layers, non-root user, no secrets in layers, scan. → lesson 42

**Kubernetes control plane components?** → lesson 49

**What happens when you run `kubectl apply -f deployment.yaml`?**
kubectl sends the object to the API server → validated, stored in etcd → Deployment controller creates a ReplicaSet → ReplicaSet controller creates Pods → scheduler assigns nodes → kubelet pulls the image and starts containers via the runtime → probes → endpoints updated → Service routes traffic. → lessons 49–51

**Requests vs limits?** What happens when a container exceeds each? → lesson 52

**Readiness vs liveness probe? Why should liveness not check the database?** → lesson 52

**A pod is in CrashLoopBackOff. What do you do?**
`describe` (Last State, exit code, events), `logs --previous`, check config/secrets/dependencies, OOMKilled, probes. → lesson 54

**Pod is OOMKilled but the node has free memory. Why?**
The container's memory limit (cgroup) was exceeded, not the node's RAM. → lessons 37, 54

**How does a Service find its pods?** Label selector → EndpointSlices of Ready pods. → lesson 51

**How do you do zero-downtime deployments?**
Rolling update with `maxUnavailable: 0`, readiness probes, graceful shutdown (SIGTERM handling, preStop), PDBs, backward-compatible migrations. → lessons 44, 52

## DevOps

**What is CI/CD? Continuous delivery vs deployment?** → lesson 44

**Blue-green vs canary?** → lesson 44

**What is idempotency and why does it matter in Ansible/Terraform?** → lessons 45, 46

**What is Terraform state? What if two people apply at once?**
State maps code to real resources; without locking, concurrent applies corrupt it. Use a remote backend with locking. → lesson 46

**What does `-/+` in a Terraform plan mean?** Replace: destroy and recreate. → lesson 46

**How do you manage secrets?** → lesson 35

**GitOps: what and why?** → lesson 53

**What DORA metrics do you know?** → lesson 44

## SRE

**SLI vs SLO vs SLA? What is an error budget?** → lesson 57

**How much downtime is 99.9% per month?** About 43 minutes. → lesson 57

**Why alert on symptoms, not causes? What is burn-rate alerting?** → lessons 56, 57

**Golden signals? RED vs USE?** → lessons 39, 55

**Walk me through how you handle an incident.**
Acknowledge, declare, assign IC, assess impact, mitigate first (rollback/failover), communicate on a cadence, resolve, postmortem with action items. → lesson 58

**What makes a postmortem blameless and useful?** → lesson 59

**How do retries cause outages? How do you prevent it?**
Retry storms multiply load; use backoff with jitter, limits, retry budgets, retries in one layer, circuit breakers. → lesson 60

**How would you plan capacity for a 3× traffic event?** → lesson 60

## Scenario questions (practice out loud)

1. "The website is slow for some users since this morning." Walk through your investigation.
2. "Disk on the database server fills up by 5% per hour." What do you do now, and what afterwards?
3. "After a deploy, error rate is 2%. Rollback or fix forward?"
4. "A developer accidentally committed an AWS key to a public repo." Steps in order.
5. "Design monitoring and alerting for a new payment service."
6. "Kubernetes nodes keep going NotReady at night."
7. "Our CI pipeline takes 40 minutes. Make it faster."
8. "Design a highly available setup for a web app and its PostgreSQL database."

For each, structure your answer: **clarify** (who is affected, since when, what changed) → **mitigate** → **diagnose** with specific commands → **fix** → **prevent**.

## Behavioural questions

- Tell me about an outage you handled. What did you learn?
- A time you disagreed with a teammate on a technical decision.
- Something you automated that saved real time.
- A mistake you made in production.

Use STAR: Situation, Task, Action, Result. Have numbers ready ("deploy time from 40 to 8 minutes", "pages per week from 30 to 4").

## Next
[Lesson 67. Glossary and resources](67-glossary-resources.md)
