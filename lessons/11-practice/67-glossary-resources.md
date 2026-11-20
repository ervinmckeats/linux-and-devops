# Lesson 67. Glossary and resources

## Glossary

| Term | Meaning |
|---|---|
| ACL | Access control list: extra per-user/group permissions on a file |
| AZ | Availability zone: an independent data center inside a cloud region |
| Bastion | A hardened host used as the only SSH entry point into a private network |
| Blue-green | Deploy to a second full environment, then switch traffic |
| Burn rate | How fast an error budget is being spent relative to the SLO window |
| Canary | Release to a small share of traffic first, watch metrics, then expand |
| Capability | A slice of root privileges (e.g. `CAP_NET_BIND_SERVICE`) |
| Cardinality | Number of unique label combinations of a metric |
| cgroup | Kernel feature that limits and accounts CPU, memory, I/O, PIDs |
| CIDR | IP range notation: `10.0.0.0/16` |
| CNI | Container Network Interface: plugins that give pods networking |
| Circuit breaker | Stops calls to a failing dependency for a while to fail fast |
| CRI | Container Runtime Interface used by kubelet (containerd, CRI-O) |
| Daemon | A background service process |
| DORA metrics | Deploy frequency, lead time, change failure rate, time to restore |
| Drift | Difference between real infrastructure and its code definition |
| Error budget | `1 − SLO`: the allowed amount of unreliability |
| etcd | Key-value store holding Kubernetes cluster state |
| Exit code | Number a process returns: 0 = success, 128+N = killed by signal N |
| FHS | Filesystem Hierarchy Standard: what lives in `/etc`, `/var`, `/usr`... |
| File descriptor | Integer handle for an open file, socket or pipe |
| GitOps | Git as the source of truth; an agent applies it to the cluster |
| Glob | Shell filename pattern: `*.log`, `file?.txt` |
| Golden signals | Latency, traffic, errors, saturation |
| HPA | Horizontal Pod Autoscaler |
| IaC | Infrastructure as code |
| Idempotent | Running again gives the same result without extra changes |
| Image layer | A read-only filesystem diff that makes up a container image |
| Incident Commander | The person who coordinates an incident response |
| Ingress | Kubernetes object for HTTP(S) routing into the cluster |
| initramfs | Temporary root filesystem used during boot to mount the real root |
| Inode | Filesystem record of a file's metadata and data location |
| Kernel | The core of the OS that manages hardware, processes and memory |
| Load average | Average number of runnable + uninterruptible tasks |
| LVM | Logical Volume Manager: flexible volumes over physical disks |
| MAC | Mandatory access control: SELinux, AppArmor |
| MTTR | Mean time to restore/recover |
| Namespace (Linux) | Kernel isolation of PIDs, network, mounts, hostname, users, IPC |
| Namespace (Kubernetes) | A scope for names, quotas and access control |
| NAT | Network address translation: many private IPs behind one public IP |
| OOM killer | Kernel mechanism that kills a process when memory runs out |
| Observability | Ability to understand a system's state from its outputs |
| Page cache | RAM used to cache file data; reclaimable |
| PDB | PodDisruptionBudget: minimum pods kept during voluntary disruptions |
| Pipe | `|` connects one process's stdout to another's stdin |
| Pod | Smallest deployable unit in Kubernetes: one or more containers |
| Postmortem | Written review of an incident and the follow-up actions |
| Probe | Kubernetes health check: startup, readiness, liveness |
| PSI | Pressure Stall Information: time tasks waited for CPU, memory, I/O |
| RAID | Combining disks for redundancy and/or speed |
| RBAC | Role-based access control |
| Reconcile loop | Controller logic that drives actual state to desired state |
| RPO / RTO | Max acceptable data loss / downtime after a disaster |
| Runbook | Step-by-step instructions for handling an alert or task |
| Saturation | How much work is queued because a resource is full |
| Security group | Stateful cloud firewall attached to instances |
| Service (Kubernetes) | Stable IP and DNS name in front of a set of pods |
| Shell | Command interpreter: bash, zsh |
| Signal | Asynchronous notification to a process: TERM, KILL, HUP... |
| SLI / SLO / SLA | Indicator / objective / agreement for service reliability |
| SNI | TLS extension carrying the requested hostname |
| State (Terraform) | Mapping between code resources and real infrastructure |
| Sticky bit | On a directory: only owners can delete their files |
| SUID | Run a file with its owner's privileges |
| Swap | Disk space used as overflow memory |
| Syscall | A request from a program to the kernel |
| Throttling (CPU) | Pausing a cgroup that used its CPU quota for the period |
| TTL | Time to live: how long DNS answers are cached |
| umask | Mask that removes permissions from newly created files |
| Unit (systemd) | A managed object: service, timer, socket, mount, target |
| USE method | Utilization, saturation, errors per resource |
| veth | Virtual Ethernet pair connecting network namespaces |
| VPC | Virtual private cloud network |
| Zombie | Exited process whose parent has not collected its exit status |

## Resources

All free unless noted.

### Linux
- `man` pages, `tldr`, and the documentation of your distribution: always first.
- *The Linux Command Line* by William Shotts: free book, from first command to scripting.
- Arch Wiki: excellent documentation for any distribution.
- *Bash Guide* and *Bash Pitfalls* on wooledge.org.
- ShellCheck: finds bugs in shell scripts.
- explainshell.com: explains each part of a command line.
- GTFOBins: how ordinary binaries can be abused for privilege escalation (know it to defend against it).

### Networking
- *Beej's Guide to Network Programming*: sockets in depth.
- Julia Evans' zines and blog: networking, Linux and debugging explained simply.
- Mozilla MDN HTTP documentation.

### Performance and internals
- Brendan Gregg's site: Linux performance, USE method, flame graphs, eBPF.
- *Systems Performance* by Brendan Gregg (book, paid): the reference.
- The kernel documentation at docs.kernel.org.

### Containers and Kubernetes
- Official Docker and Kubernetes documentation (kubernetes.io/docs, including tutorials).
- *Kubernetes the Hard Way* by Kelsey Hightower: build a cluster by hand once.
- Killercoda: free interactive Kubernetes and Linux scenarios in the browser.
- CNCF landscape: the map of cloud-native tools.

### DevOps and IaC
- Ansible, Terraform/OpenTofu, Argo CD and Flux official docs and tutorials.
- The *DORA* research reports (State of DevOps).
- *The Twelve-Factor App*: principles for cloud-native apps.

### SRE
- *Site Reliability Engineering* and *The Site Reliability Workbook* (Google): free online.
- *Implementing Service Level Objectives* by Alex Hidalgo (book, paid).
- Public postmortem collections (search "awesome postmortems"): real incidents from many companies.
- OpenTelemetry and Prometheus documentation.

### Practice
- Your own VMs: break and fix things on purpose.
- Cloud free tiers: always set a budget alert first.
- Contribute docs or fixes to an open-source project you use.

## What next

1. Finish the labs (lesson 64) and at least two capstone projects (lesson 65).
2. Pick a direction: platform engineering, SRE, cloud, security, or databases: and go deep.
3. Optional certifications that are respected in the market: LFCS (Linux), CKA/CKAD/CKS (Kubernetes), a cloud associate-level certificate, Terraform Associate. They help pass HR filters; projects and interviews get you the job.
4. Keep a public log of what you learn (blog, notes repo). Teaching is the fastest way to understand.

[Back to the course overview](../../README.md)
