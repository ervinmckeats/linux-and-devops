# Lesson 65. Capstone projects

## Goal
Build projects that prove your skills to an employer. Each one is a public Git repository with a clear README, architecture diagram, and instructions to reproduce it.

Recruiters and interviewers rarely read certificates; they do open GitHub. A small project done properly beats ten tutorials.

## What every project repo must have

- **README**: what problem it solves, architecture diagram, how to run it, what you would improve.
- Everything as code: no "then click here in the console".
- One command (or a short sequence) to create and to destroy the environment.
- Secrets never in Git; an example `.env.example`.
- CI that at least lints and validates.
- Short "Decisions" section: why this tool and not another.

## Project 1. Hardened self-hosted service (Junior)

**Build:** a VPS or VM with a personal website or a tool (wiki, URL shortener, status page) behind nginx with HTTPS.

**Requirements:**
- Baseline hardening as an Ansible playbook (users, SSH, firewall, fail2ban, automatic updates).
- nginx reverse proxy, Let's Encrypt certificates with auto-renewal.
- App runs as a systemd service or with Docker Compose.
- Nightly backups with restic to object storage; documented restore test.
- Basic monitoring: node exporter + uptime probe + alert to chat.

**Skills shown:** Linux administration, networking, security, automation, backups.

## Project 2. CI/CD pipeline for a containerized app (Junior → Middle)

**Build:** a small API (any language) with a database.

**Requirements:**
- Multi-stage Dockerfile, non-root, health check; image < 150 MB.
- Compose file for local development.
- Pipeline: lint → unit tests → integration tests with a DB service → build → vulnerability scan → push image tagged with commit SHA.
- Deploy to staging automatically and to production with manual approval.
- Rollback documented and tested.

**Skills shown:** Docker, CI/CD, testing culture, release safety.

## Project 3. Infrastructure as code on a cloud (Middle)

**Build:** the network and compute for a web application on AWS, GCP or Azure free tier.

**Requirements:**
- Terraform/OpenTofu with remote state and locking; modules for network and compute.
- VPC with public and private subnets in 2 zones; app in private subnets behind a load balancer; managed database.
- IAM roles with least privilege; no long-lived keys; access via SSM/IAP instead of public SSH.
- PR pipeline: `fmt`, `validate`, `tflint`, security scan, `plan` as a PR comment.
- Cost estimate in the README and a `destroy` command.

**Skills shown:** cloud fundamentals, IaC, security, cost awareness.

## Project 4. Kubernetes platform with GitOps (Middle → Senior)

**Build:** a local (kind/k3s) or managed cluster running 2–3 services.

**Requirements:**
- Helm charts or Kustomize with dev/prod overlays.
- Argo CD (or Flux) app-of-apps; the cluster is fully reproducible from Git.
- Ingress with TLS via cert-manager; External Secrets or SOPS for secrets.
- Requests/limits, probes, PDBs, HPA, NetworkPolicies with default deny.
- kube-prometheus-stack, Loki for logs, dashboards as code.

**Skills shown:** Kubernetes, GitOps, platform engineering, security.

## Project 5. SRE package for a service (Senior)

**Build:** on top of Project 4 (or any running service).

**Requirements:**
- Two SLIs and SLOs per user journey; recording rules; error budget dashboard.
- Multi-window burn-rate alerts with runbooks linked.
- OpenTelemetry tracing across services; logs linked by trace ID.
- Load test that finds the breaking point; capacity plan for 3× traffic with one zone lost.
- Chaos experiment (pod kill, latency injection) with a hypothesis and results.
- A game day and a full blameless postmortem published in the repo.

**Skills shown:** SRE practice end-to-end: the strongest signal for SRE roles.

## Project 6. Your own tool (any level)

Automate something you or your team actually do: a CLI that checks certificate expiry for a list of domains, a Kubernetes cost report, a log analyzer, a Slack/Telegram bot for deploy notifications, a Prometheus exporter for an API you use.

Small, useful, tested, documented. Bonus: someone else uses it.

## How to present projects

- Pin the best 3–4 repositories on your GitHub profile.
- Each README starts with a 2–3 sentence summary and a diagram.
- On your CV: one line per project with the result ("GitOps platform for 3 services, rebuilt from Git in 12 minutes").
- In interviews: be ready to explain one trade-off and one thing that broke and how you fixed it.

## Next
[Lesson 66. Interview questions](66-interview-questions.md)
