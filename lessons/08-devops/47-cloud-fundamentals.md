# Lesson 47. Cloud fundamentals

## Goal
Understand the building blocks every cloud provider has, so you can work in AWS, GCP or Azure, and avoid the classic expensive or insecure mistakes.

Examples use AWS names; the table maps them to the others.

## Building blocks

| Concept | AWS | GCP | Azure |
|---|---|---|---|
| Virtual network | VPC | VPC | VNet |
| Subnet | Subnet (per AZ) | Subnet (regional) | Subnet |
| Firewall per instance | Security Group | Firewall rules | NSG |
| VM | EC2 | Compute Engine | Virtual Machines |
| Block disk | EBS | Persistent Disk | Managed Disk |
| Object storage | S3 | Cloud Storage | Blob Storage |
| Managed SQL | RDS / Aurora | Cloud SQL | Azure SQL / PostgreSQL |
| Load balancer | ALB / NLB | Cloud Load Balancing | Load Balancer / App Gateway |
| Managed Kubernetes | EKS | GKE | AKS |
| Identity and access | IAM | IAM | Entra ID + RBAC |
| DNS | Route 53 | Cloud DNS | Azure DNS |
| Metrics/logs | CloudWatch | Cloud Monitoring/Logging | Azure Monitor |
| Secrets | Secrets Manager | Secret Manager | Key Vault |
| Functions | Lambda | Cloud Functions/Run | Functions |

## Regions and availability zones

- **Region**: a geographic area (`eu-central-1`). Choose by latency to users, data residency law, price.
- **Availability Zone (AZ)**: an independent data center inside a region. Spread instances, load balancers and databases across **at least 2 AZs** to survive a data-center failure.
- Traffic between AZs and especially out to the internet costs money.

## Networking: a standard VPC

```
VPC 10.0.0.0/16
├── public subnets   (10.0.0.0/24, 10.0.1.0/24 in two AZs)
│      load balancer, NAT gateway, bastion        → route 0.0.0.0/0 via Internet Gateway
└── private subnets  (10.0.10.0/24, 10.0.11.0/24)
       app servers, databases, Kubernetes nodes  → route 0.0.0.0/0 via NAT gateway
```

- Only the load balancer is public. App servers and databases have no public IP.
- **Security groups** are stateful firewalls attached to instances; reference other groups instead of IPs ("DB allows 5432 from the app security group").
- Admin access via SSM Session Manager / IAP / VPN instead of open SSH.
- NAT gateways are convenient and **expensive** with heavy traffic; use VPC endpoints for S3/registries.

## Identity and access (IAM)

The most important cloud skill: most real breaches are leaked credentials or over-broad permissions.

- **Users** for humans (with MFA, better via SSO), **roles** for machines.
- Instances, containers and CI get **roles with temporary credentials** (instance profile, IRSA / workload identity for Kubernetes, OIDC for CI). No long-lived access keys on servers or in CI.
- **Least privilege**: start with nothing, add exact actions on exact resources.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::example-app-uploads/*"
  }]
}
```

- Separate **accounts/projects** for prod and non-prod. A mistake in dev cannot touch prod.
- Turn on the audit log (CloudTrail / Cloud Audit Logs) in every account.

## Storage

- **Block** (EBS): a disk for one VM. Snapshots for backup.
- **Object** (S3): files by key over HTTP, practically unlimited, cheap, durable. Backups, static sites, logs, artifacts.
  - Block public access by default. Most "data leak" headlines are public buckets.
  - Versioning + lifecycle rules (move old objects to cheaper classes, expire them).
  - Object Lock for immutable backups (lesson 21).
- **File** (EFS / Filestore): NFS shared between many VMs.

## Managed services: build or buy?

A managed database gives backups, failover, patching and monitoring for a price. For most teams that is cheaper than an engineer's time. Self-host when you need control or the managed option is missing/too expensive at your scale.

## CLI basics

```bash
# log in via SSO
aws configure sso

# who am I? run this first, always
aws sts get-caller-identity

aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,State.Name,PrivateIpAddress]' --output table
aws s3 ls s3://example-bucket/
aws s3 sync ./site s3://example-site --delete
aws logs tail /aws/lambda/fn --follow

# shell without SSH
aws ssm start-session --target i-0123456789abcdef0
```

Inside an instance, the metadata service gives identity and temporary credentials at `169.254.169.254`. Require IMDSv2 (session tokens): it blocks a common SSRF credential theft.

## Cost awareness

The cloud bills for everything you forget. Habits:
- Tags on every resource (`project`, `env`, `owner`) so costs can be attributed.
- Budgets and alerts from day one.
- Turn off or schedule dev environments at night.
- Watch the usual surprises: NAT gateway data, cross-AZ traffic, egress to the internet, unattached disks and old snapshots, idle load balancers, oversized instances, log retention "forever".
- Right-size from metrics, then use savings plans/reserved capacity for the steady base and spot instances for interruptible work.

## Shared responsibility

The provider secures the hardware, hypervisor and managed service internals. **You** secure your configuration: IAM, network exposure, OS patches on VMs, encryption settings, your data and your code.

## Practice

1. Create a free-tier account. Enable MFA on the root user, then stop using root; create an SSO or IAM admin user.
2. Set a budget alert of a few dollars.
3. Using Terraform (lesson 46): VPC with public and private subnets in two AZs, a VM in the private subnet reachable only via SSM Session Manager.
4. Create an S3 bucket with public access blocked, versioning and a lifecycle rule. Give the VM a role that can only write to that bucket; prove it cannot list other buckets.
5. Destroy everything and check the billing page the next day.

## Check yourself

- Why should app servers live in private subnets?
- Why use roles instead of access keys on a VM?
- Name three things that unexpectedly increase a cloud bill.

## Next
[Lesson 48. Virtualization: KVM and QEMU](48-kvm-qemu.md)
