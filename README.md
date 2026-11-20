# Linux, DevOps and SRE — from zero to job-ready

A free, hands-on course that takes you from your first terminal command to running production systems on Kubernetes with SLOs, alerts and postmortems.

- **Practice first.** Every lesson ends with tasks you do in a real terminal.
- **One path.** Lessons build on each other. Go top to bottom.
- **No fluff.** Theory only where it explains *why* a command behaves the way it does.

## How to use this course

1. Set up a lab (lesson 1). Never practice on a server you care about.
2. Read a lesson, type every command by hand. Do not copy and paste.
3. Do the **Practice** block. Answer **Check yourself** without looking back.
4. Break things on purpose in your VM, then fix them. Take snapshots first.
5. Tick the lesson in your fork of this README.

Each lesson has the same shape: **Goal → Theory → Commands → Practice → Check yourself → Next**.

## Roadmap

| Level | Modules | You can... |
|---|---|---|
| Beginner | 0–2 | work in the terminal and run a single Linux box |
| Junior admin | 3–5 | run a secured, networked server and automate it with Bash and Git |
| Engineer | 6–8 | debug performance, ship containers, build CI/CD and infrastructure as code |
| DevOps / SRE | 9–11 | run Kubernetes, measure reliability, handle incidents, pass interviews |

## Lessons

Status: ✅ ready · ⏳ in progress

### Module 0 — Start
| # | Lesson | Status |
|---|---|---|
| 1 | [How to use the course and set up a lab](lessons/00-start/01-lab-setup.md) | ✅ |
| 2 | [What Linux is: kernel, distributions, shell](lessons/00-start/02-what-is-linux.md) | ✅ |

### Module 1 — Terminal
| # | Lesson | Status |
|---|---|---|
| 3 | [Anatomy of a command and getting help](lessons/01-terminal/03-commands-and-help.md) | ✅ |
| 4 | [Navigating the filesystem](lessons/01-terminal/04-navigation-fhs.md) | ✅ |
| 5 | [Files and directories, globbing](lessons/01-terminal/05-files-and-directories.md) | ✅ |
| 6 | [Viewing files](lessons/01-terminal/06-viewing-files.md) | ✅ |
| 7 | [Editors: nano and vim](lessons/01-terminal/07-editors.md) | ✅ |
| 8 | [Inodes, hard and symbolic links](lessons/01-terminal/08-inodes-and-links.md) | ✅ |
| 9 | [Searching: find and grep](lessons/01-terminal/09-find-and-grep.md) | ✅ |
| 10 | [Redirection, pipes, exit codes, xargs](lessons/01-terminal/10-pipes-and-redirection.md) | ✅ |
| 11 | [Text processing: cut, sort, uniq, sed, awk](lessons/01-terminal/11-text-processing.md) | ✅ |

### Module 2 — System administration
| # | Lesson | Status |
|---|---|---|
| 12 | [Users, groups, sudo](lessons/02-sysadmin/12-users-groups-sudo.md) | ✅ |
| 13 | [Permissions, special bits, umask, ACL](lessons/02-sysadmin/13-permissions.md) | ✅ |
| 14 | [Processes, signals, jobs, tmux](lessons/02-sysadmin/14-processes-signals.md) | ✅ |
| 15 | [Package managers: apt, dnf, pacman](lessons/02-sysadmin/15-package-managers.md) | ✅ |
| 16 | [systemd and services](lessons/02-sysadmin/16-systemd.md) | ✅ |
| 17 | [Logs: journalctl, syslog, logrotate](lessons/02-sysadmin/17-logs.md) | ✅ |
| 18 | [Scheduling: cron and systemd timers](lessons/02-sysadmin/18-cron-and-timers.md) | ✅ |
| 19 | [Disks, partitions, mount, fstab, swap](lessons/02-sysadmin/19-disks-filesystems.md) | ✅ |
| 20 | [LVM, RAID, snapshots](lessons/02-sysadmin/20-lvm-raid.md) | ✅ |
| 21 | [Archives and backups](lessons/02-sysadmin/21-archives-backups.md) | ✅ |
| 22 | [Boot process and recovery](lessons/02-sysadmin/22-boot-and-recovery.md) | ✅ |

### Module 3 — Networking
| # | Lesson | Status |
|---|---|---|
| 23 | [Network basics: IP, ports, TCP/UDP, DNS](lessons/03-networking/23-network-basics.md) | ✅ |
| 24 | [Network tools: ip, ss, ping, traceroute, dig, curl](lessons/03-networking/24-network-tools.md) | ✅ |
| 25 | [SSH: keys, config, scp, rsync, tunnels](lessons/03-networking/25-ssh.md) | ✅ |
| 26 | [Firewalls: ufw, firewalld, nftables](lessons/03-networking/26-firewalls.md) | ✅ |
| 27 | [HTTP, TLS, nginx reverse proxy](lessons/03-networking/27-http-tls-nginx.md) | ✅ |
| 28 | [Advanced networking: netplan, namespaces, tcpdump](lessons/03-networking/28-advanced-networking.md) | ✅ |

### Module 4 — Automation
| # | Lesson | Status |
|---|---|---|
| 29 | [Bash scripting basics](lessons/04-automation/29-bash-basics.md) | ✅ |
| 30 | [Bash for professionals](lessons/04-automation/30-bash-pro.md) | ✅ |
| 31 | [Git for operations](lessons/04-automation/31-git.md) | ✅ |
| 32 | [Python for operations](lessons/04-automation/32-python.md) | ✅ |

### Module 5 — Security
| # | Lesson | Status |
|---|---|---|
| 33 | Server hardening | ⏳ |
| 34 | SELinux, AppArmor, auditd, capabilities | ⏳ |
| 35 | Secrets and encryption | ⏳ |

### Module 6 — Internals and performance
| # | Lesson | Status |
|---|---|---|
| 36 | Kernel: sysctl, modules, limits | ⏳ |
| 37 | Memory: virtual memory, page cache, swap, OOM killer | ⏳ |
| 38 | cgroups v2 and namespaces | ⏳ |
| 39 | Performance: USE method, first 60 seconds, PSI | ⏳ |
| 40 | Tracing: strace, perf, eBPF | ⏳ |

### Module 7 — Containers
| # | Lesson | Status |
|---|---|---|
| 41 | Docker basics | ⏳ |
| 42 | Dockerfile best practices | ⏳ |
| 43 | Docker Compose, networks, volumes | ⏳ |

### Module 8 — DevOps
| # | Lesson | Status |
|---|---|---|
| 44 | CI/CD concepts and pipelines | ⏳ |
| 45 | Ansible | ⏳ |
| 46 | Terraform and infrastructure as code | ⏳ |
| 47 | Cloud fundamentals | ⏳ |
| 48 | Virtualization: KVM and QEMU | ⏳ |

### Module 9 — Kubernetes
| # | Lesson | Status |
|---|---|---|
| 49 | Architecture and core objects | ⏳ |
| 50 | kubectl in daily work | ⏳ |
| 51 | Deployments, Services, Ingress, ConfigMaps, Secrets | ⏳ |
| 52 | Requests, limits, probes, autoscaling | ⏳ |
| 53 | Helm and GitOps | ⏳ |
| 54 | Troubleshooting: CrashLoopBackOff, OOMKilled, Pending | ⏳ |

### Module 10 — SRE
| # | Lesson | Status |
|---|---|---|
| 55 | Observability: metrics, logs, traces | ⏳ |
| 56 | Prometheus, Grafana, Alertmanager | ⏳ |
| 57 | SLI, SLO, error budgets | ⏳ |
| 58 | Incident response and on-call | ⏳ |
| 59 | Postmortems and runbooks | ⏳ |
| 60 | Reliability patterns and capacity planning | ⏳ |

### Module 11 — Practice and career
| # | Lesson | Status |
|---|---|---|
| 61 | Troubleshooting playbook | ⏳ |
| 62 | Dangerous commands | ⏳ |
| 63 | Real-world scenarios | ⏳ |
| 64 | Labs with solutions | ⏳ |
| 65 | Capstone projects | ⏳ |
| 66 | Interview questions | ⏳ |
| 67 | Glossary and resources | ⏳ |

## Contributing

Found a mistake or have a better example? Open an issue or a pull request. Keep examples short and tested.

## License

[MIT](LICENSE)
