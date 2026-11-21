# Lesson 46. Terraform and infrastructure as code

## Goal
Create and change cloud infrastructure from code with review, plans and safe state handling.

## Why IaC

Clicking in a console is fast once and painful forever: nobody knows what exists, environments drift apart, rebuilding after a disaster is guesswork. With IaC:
- infrastructure is described in files, reviewed in PRs, versioned in Git;
- `plan` shows exactly what will change before it changes;
- dev, staging and prod are built from the same code with different variables.

**Terraform** (and its open-source fork **OpenTofu**, same language, `tofu` command) is the most common tool. Others: Pulumi (real programming languages), CloudFormation (AWS only), Crossplane (Kubernetes-native).

## Core concepts

- **Provider**: plugin for an API: AWS, GCP, Azure, Kubernetes, Cloudflare, GitHub...
- **Resource**: something Terraform creates and owns (`aws_instance`, `aws_s3_bucket`).
- **Data source**: something that exists and you only read (latest AMI, existing VPC).
- **Variables / outputs / locals**: inputs, results, helper values.
- **State**: Terraform's record of which real objects belong to which resource. It is how Terraform knows what to change.
- **Module**: a reusable folder of resources with inputs and outputs.

## Workflow

```bash
# download providers, set up backend
terraform init

terraform fmt -recursive
terraform validate
terraform plan -out=tfplan

# apply exactly what was reviewed
terraform apply tfplan

# careful
terraform destroy
```

Read every plan. Symbols: `+` create, `~` update in place, `-/+` **replace** (destroy then create: data loss for databases and disks!), `-` destroy.

## Example: a small web server on AWS

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket         = "example-tfstate"
    key            = "web/prod.tfstate"
    region         = "eu-central-1"
    dynamodb_table = "tf-locks"     # state locking (newer versions: use_lockfile = true)
    encrypt        = true
  }
}

provider "aws" {
  region = var.region
  default_tags {
    tags = { project = "web", env = var.env, managed_by = "terraform" }
  }
}
```

`variables.tf`:

```hcl
variable "region" {
  type    = string
  default = "eu-central-1"
}

variable "env" {
  type = string
}

variable "instance_type" {
  type    = string
  default = "t3.small"
}

variable "admin_cidr" {
  type        = string
  description = "IP range allowed to SSH"
}
```

`main.tf`:

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]   # Canonical
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }
}

resource "aws_security_group" "web" {
  name = "web-${var.env}"

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  vpc_security_group_ids = [aws_security_group.web.id]
  user_data              = file("${path.module}/cloud-init.yaml")

  metadata_options {
    http_tokens = "required"       # IMDSv2 only
  }
  root_block_device {
    volume_size = 20
    encrypted   = true
  }
  tags = { Name = "web-${var.env}" }
}
```

`outputs.tf`:

```hcl
output "public_ip" { value = aws_instance.web.public_ip }
```

`prod.tfvars`:

```hcl
env        = "prod"
admin_cidr = "203.0.113.10/32"
```

```bash
terraform plan -var-file=prod.tfvars -out=tfplan
terraform apply tfplan
terraform output public_ip
```

## State

- **Remote backend** (S3, GCS, Azure Blob, Terraform Cloud) with **locking**: never keep team state on a laptop or in Git.
- State can contain **secrets** (DB passwords, keys). Encrypt it, restrict access.
- One state per environment and per component (network, database, app) to limit blast radius and speed up plans.

```bash
terraform state list
terraform state show aws_instance.web

# adopt an existing resource
terraform import aws_s3_bucket.logs my-existing-bucket

# after refactoring
terraform state mv aws_instance.web module.web.aws_instance.this

# detect drift (manual console changes)
terraform plan -refresh-only
```

Newer versions also support `import {}` and `moved {}` blocks in code, which go through normal review.

## Modules

```hcl
module "network" {
  source   = "./modules/network"
  cidr     = "10.20.0.0/16"
  env      = var.env
}

resource "aws_instance" "web" {
  subnet_id = module.network.public_subnet_ids[0]
  # ...
}
```

Pin versions of registry modules (`version = "~> 5.1"`).

## Protect important resources

```hcl
resource "aws_db_instance" "main" {
  # ...
  deletion_protection = true
  lifecycle {
    prevent_destroy = true
    ignore_changes  = [password]
  }
}
```

## Terraform in CI

- On PR: `fmt -check`, `validate`, `tflint`, security scan (`checkov`/`trivy config`), `plan` posted as a comment.
- On merge: `apply` the saved plan, with manual approval for prod.
- Cloud credentials via OIDC, not long-lived keys.

## Practice

1. Without a cloud account: use the `local` and `random` providers or the Docker provider to create resources and practice `plan`/`apply`/`destroy` and state commands.
2. With a cloud free tier: create the web server above, then change the instance type and read the plan (in place or replace?).
3. Move state to a remote backend with locking.
4. Make a manual change in the console, then find it with `plan -refresh-only`.
5. Extract the security group into a module and use `moved {}` so nothing is recreated.

## Check yourself

- What is Terraform state for, and why must it be remote and locked?
- What does `-/+` mean in a plan, and why is it dangerous for a database?
- Terraform or Ansible for creating a VM? For configuring nginx on it?

## Next
[Lesson 47. Cloud fundamentals](47-cloud-fundamentals.md)
