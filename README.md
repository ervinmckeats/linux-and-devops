# Linux, DevOps and SRE

Free course from zero to a job as a Linux admin, DevOps or SRE engineer. 67 lessons, in order. Each one is short, practical, and ends with tasks you do in a real terminal.

**Start here:** [Lesson 1. Set up a lab](lessons/00-start/01-lab-setup.md), then go down the list.

## Start

1. [How to use the course and set up a lab](lessons/00-start/01-lab-setup.md)
2. [What Linux is: kernel, distributions, shell](lessons/00-start/02-what-is-linux.md)

## Terminal

3. [Anatomy of a command and getting help](lessons/01-terminal/03-commands-and-help.md)
4. [Navigating the filesystem](lessons/01-terminal/04-navigation-fhs.md)
5. [Files and directories, globbing](lessons/01-terminal/05-files-and-directories.md)
6. [Viewing files](lessons/01-terminal/06-viewing-files.md)
7. [Editors: nano and vim](lessons/01-terminal/07-editors.md)
8. [Inodes, hard and symbolic links](lessons/01-terminal/08-inodes-and-links.md)
9. [Searching: find and grep](lessons/01-terminal/09-find-and-grep.md)
10. [Redirection, pipes, exit codes, xargs](lessons/01-terminal/10-pipes-and-redirection.md)
11. [Text processing: cut, sort, uniq, sed, awk](lessons/01-terminal/11-text-processing.md)

## System administration

12. [Users, groups, sudo](lessons/02-sysadmin/12-users-groups-sudo.md)
13. [Permissions, special bits, umask, ACL](lessons/02-sysadmin/13-permissions.md)
14. [Processes, signals, jobs, tmux](lessons/02-sysadmin/14-processes-signals.md)
15. [Package managers: apt, dnf, pacman](lessons/02-sysadmin/15-package-managers.md)
16. [systemd and services](lessons/02-sysadmin/16-systemd.md)
17. [Logs: journalctl, syslog, logrotate](lessons/02-sysadmin/17-logs.md)
18. [Scheduling: cron and systemd timers](lessons/02-sysadmin/18-cron-and-timers.md)
19. [Disks, partitions, mount, fstab, swap](lessons/02-sysadmin/19-disks-filesystems.md)
20. [LVM, RAID, snapshots](lessons/02-sysadmin/20-lvm-raid.md)
21. [Archives and backups](lessons/02-sysadmin/21-archives-backups.md)
22. [Boot process and recovery](lessons/02-sysadmin/22-boot-and-recovery.md)

## Networking

23. [Network basics: IP, ports, TCP/UDP, DNS](lessons/03-networking/23-network-basics.md)
24. [Network tools: ip, ss, ping, traceroute, dig, curl](lessons/03-networking/24-network-tools.md)
25. [SSH: keys, config, scp, rsync, tunnels](lessons/03-networking/25-ssh.md)
26. [Firewalls: ufw, firewalld, nftables](lessons/03-networking/26-firewalls.md)
27. [HTTP, TLS, nginx reverse proxy](lessons/03-networking/27-http-tls-nginx.md)
28. [Advanced networking: netplan, namespaces, tcpdump](lessons/03-networking/28-advanced-networking.md)

## Automation: Bash, Git, Python

29. [Bash scripting basics](lessons/04-automation/29-bash-basics.md)
30. [Bash for professionals](lessons/04-automation/30-bash-pro.md)
31. [Git for operations](lessons/04-automation/31-git.md)
32. [Python for operations](lessons/04-automation/32-python.md)

## Security

33. [Server hardening](lessons/05-security/33-server-hardening.md)
34. [SELinux, AppArmor, auditd, capabilities](lessons/05-security/34-mac-audit-capabilities.md)
35. [Secrets and encryption](lessons/05-security/35-secrets-encryption.md)

## Linux internals and performance

36. [Kernel: sysctl, modules, limits](lessons/06-internals/36-kernel-sysctl-limits.md)
37. [Memory: virtual memory, page cache, swap, OOM killer](lessons/06-internals/37-memory-oom.md)
38. [cgroups v2 and namespaces](lessons/06-internals/38-cgroups-namespaces.md)
39. [Performance: USE method, first 60 seconds, PSI](lessons/06-internals/39-performance.md)
40. [Tracing: strace, ltrace, perf, eBPF](lessons/06-internals/40-tracing.md)

## Containers

41. [Docker basics](lessons/07-containers/41-docker-basics.md)
42. [Dockerfile best practices](lessons/07-containers/42-dockerfile.md)
43. [Docker Compose, networks, volumes](lessons/07-containers/43-compose-networks-volumes.md)

## DevOps: CI/CD, IaC, cloud

44. [CI/CD concepts and pipelines](lessons/08-devops/44-cicd.md)
45. [Ansible](lessons/08-devops/45-ansible.md)
46. [Terraform and infrastructure as code](lessons/08-devops/46-terraform.md)
47. [Cloud fundamentals](lessons/08-devops/47-cloud-fundamentals.md)
48. [Virtualization: KVM and QEMU](lessons/08-devops/48-kvm-qemu.md)

## Kubernetes

49. [Kubernetes architecture and core objects](lessons/09-kubernetes/49-k8s-architecture.md)
50. [kubectl in daily work](lessons/09-kubernetes/50-kubectl.md)
51. [Deployments, Services, Ingress, ConfigMaps, Secrets](lessons/09-kubernetes/51-workloads-networking-config.md)
52. [Requests, limits, probes, autoscaling](lessons/09-kubernetes/52-resources-probes-autoscaling.md)
53. [Helm and GitOps](lessons/09-kubernetes/53-helm-gitops.md)
54. [Kubernetes troubleshooting: CrashLoopBackOff, OOMKilled, Pending and friends](lessons/09-kubernetes/54-k8s-troubleshooting.md)

## SRE

55. [Observability: metrics, logs, traces](lessons/10-sre/55-observability.md)
56. [Prometheus, Grafana, Alertmanager](lessons/10-sre/56-prometheus-grafana.md)
57. [SLI, SLO, error budgets](lessons/10-sre/57-slo-error-budgets.md)
58. [Incident response and on-call](lessons/10-sre/58-incident-response.md)
59. [Postmortems and runbooks](lessons/10-sre/59-postmortems-runbooks.md)
60. [Reliability patterns and capacity planning](lessons/10-sre/60-reliability-patterns.md)

## Practice and career

61. [Troubleshooting playbook: "what to do if..."](lessons/11-practice/61-troubleshooting-playbook.md)
62. [Dangerous commands](lessons/11-practice/62-dangerous-commands.md)
63. [Real-world scenarios](lessons/11-practice/63-real-world-scenarios.md)
64. [Labs with solutions](lessons/11-practice/64-labs.md)
65. [Capstone projects](lessons/11-practice/65-capstone-projects.md)
66. [Interview questions](lessons/11-practice/66-interview-questions.md)
67. [Glossary and resources](lessons/11-practice/67-glossary-resources.md)

## How to study

1. One lesson at a time, top to bottom.
2. Type every command yourself in a VM. Take a snapshot before risky tasks.
3. Do the **Practice** block, then answer **Check yourself** without looking back.
4. After every module, do the related tasks in [Labs](lessons/11-practice/64-labs.md).
5. Finish with two or three [capstone projects](lessons/11-practice/65-capstone-projects.md) for your GitHub.

## Contributing

Found a mistake or have a better example? Open an issue or a pull request.

## License

[MIT](LICENSE)
