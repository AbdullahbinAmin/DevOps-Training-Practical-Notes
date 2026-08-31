# Day 10 — Terraform Modules

> A module is just a folder of `.tf` files that does one related job — package it once, call it many times with different inputs.

## Learning Objectives
- Define what a module is and distinguish the root module from child modules.
- Lay out a standard module structure (`main.tf`, `variables.tf`, `outputs.tf`, `README.md`).
- Write a local child module and call it from the root with a `module` block.
- Pass inputs, consume outputs (`module.<name>.<output>`), and use `count`/`for_each` on modules.
- Understand source types and version pinning, and when `terraform init` / `terraform get` is required.

## Prerequisites
- Terraform 1.x, AWS provider 5.x, working AWS credentials outside the code.
- Comfort with variables, outputs and `terraform init/plan/apply`.
- A fresh folder for the lab, e.g. `tf-modules-lab/`.

## Concept (plain English)
Every Terraform configuration you run commands in is already a module — the **root module**. When the
root calls another folder with a `module` block, that folder is a **child module**.

Why modules:
- Package and reuse resource configurations instead of copy-pasting.
- Share code across a team and across projects.
- Consistent, repeatable, versioned infrastructure.
- Less duplication, fewer mistakes, easier to maintain and scale.

A module groups *related* resources. A VPC module, for example, owns the VPC, its public and private
subnets, internet gateway, NAT, route tables and security groups — one call, whole network.

Interface of a module:
- **Inputs** = `variable` blocks (what the caller may set).
- **Outputs** = `output` blocks (what the module hands back).
- Everything else is an implementation detail the caller should not care about.

Standard structure:

```
my-module/
├── README.md      # how to use it: inputs, outputs, examples
├── main.tf        # the resources
├── variables.tf   # input variables
├── outputs.tf     # returned values
├── versions.tf    # (optional) required_version / required_providers
└── locals.tf      # (optional) computed helpers
```

Full project layout for this lab:

```
tf-modules-lab/                 <- ROOT MODULE
├── versions.tf
├── provider.tf
├── main.tf                     <- calls the child modules
├── variables.tf
├── outputs.tf
├── terraform.tfvars
└── modules/
    ├── network/                <- CHILD MODULE
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── outputs.tf
    │   └── README.md
    └── web-server/             <- CHILD MODULE
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── README.md
```

Rule of thumb: put `provider` blocks and backends in the **root** only. Child modules declare which
providers they need (`required_providers`) but do not configure credentials or regions.

## Step-by-Step Practical

1. Create the skeleton.

```bash
mkdir -p tf-modules-lab/modules/{network,web-server}
cd tf-modules-lab
```

2. Child module `modules/network/variables.tf`.

```hcl
variable "name" {
  description = "Name prefix applied to all network resources."
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC."
  type        = string
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "vpc_cidr must be a valid IPv4 CIDR block."
  }
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets, one per AZ."
  type        = list(string)
  default     = ["10.0.0.0/24", "10.0.1.0/24"]
}

variable "azs" {
  description = "Availability zones to spread subnets across."
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
}

variable "tags" {
  description = "Extra tags merged onto every resource."
  type        = map(string)
  default     = {}
}
```

3. Child module `modules/network/main.tf`.

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

locals {
  common_tags = merge(var.tags, { Module = "network" })
}

resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(local.common_tags, { Name = "${var.name}-vpc" })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = merge(local.common_tags, { Name = "${var.name}-igw" })
}

resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.azs[count.index % length(var.azs)]
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, { Name = "${var.name}-public-${count.index + 1}" })
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = merge(local.common_tags, { Name = "${var.name}-public-rt" })
}

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

4. Child module `modules/network/outputs.tf` — the module's public contract.

```hcl
output "vpc_id" {
  description = "ID of the created VPC."
  value       = aws_vpc.this.id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC."
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "List of public subnet IDs."
  value       = aws_subnet.public[*].id
}

output "igw_id" {
  description = "Internet Gateway ID."
  value       = aws_internet_gateway.this.id
}
```

5. Child module `modules/web-server/variables.tf`.

```hcl
variable "name" {
  description = "Name prefix for the instance and its security group."
  type        = string
}

variable "vpc_id" {
  description = "VPC to place the security group in."
  type        = string
}

variable "subnet_id" {
  description = "Subnet to launch the instance in."
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type."
  type        = string
  default     = "t3.micro"
}

variable "allowed_http_cidrs" {
  description = "CIDRs allowed to reach port 80."
  type        = list(string)
  default     = ["0.0.0.0/0"]
}

variable "tags" {
  description = "Extra tags merged onto every resource."
  type        = map(string)
  default     = {}
}
```

6. Child module `modules/web-server/main.tf` and `outputs.tf`.

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Look up the latest Amazon Linux 2023 AMI instead of hardcoding an AMI ID.
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_security_group" "web" {
  name        = "${var.name}-web-sg"
  description = "Allow inbound HTTP and all egress."
  vpc_id      = var.vpc_id

  # NOTE: no SSH ingress on purpose. Use SSM Session Manager for shell access.
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = var.allowed_http_cidrs
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.tags, { Name = "${var.name}-web-sg" })
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.al2023.id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOT
    #!/bin/bash
    dnf -y install nginx
    echo "Hello from ${var.name}" > /usr/share/nginx/html/index.html
    systemctl enable --now nginx
  EOT

  metadata_options {
    http_tokens = "required" # IMDSv2 only
  }

  tags = merge(var.tags, { Name = "${var.name}-web" })
}
```

```hcl
output "instance_id" {
  description = "EC2 instance ID."
  value       = aws_instance.web.id
}

output "public_ip" {
  description = "Public IP of the web server."
  value       = aws_instance.web.public_ip
}

output "security_group_id" {
  description = "Security group protecting the instance."
  value       = aws_security_group.web.id
}
```

7. Root module `versions.tf` and `provider.tf`.

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      ManagedBy = "Terraform"
      Project   = var.project
    }
  }
}
```

8. Root module `variables.tf` and `terraform.tfvars`.

```hcl
variable "aws_region" {
  description = "AWS region."
  type        = string
  default     = "us-east-1"
}

variable "project" {
  description = "Project / name prefix."
  type        = string
  default     = "day10-modules"
}

variable "my_ip_cidr" {
  description = "Your public IP in CIDR form, for scoped HTTP access."
  type        = string
  default     = "0.0.0.0/0"
}
```

```hcl
aws_region = "us-east-1"
project    = "day10-modules"
my_ip_cidr = "203.0.113.25/32"
```

9. Root module `main.tf` — the `module` blocks. Notice how outputs of one feed inputs of the other.

```hcl
module "network" {
  source = "./modules/network"

  name                = var.project
  vpc_cidr            = "10.0.0.0/16"
  azs                 = ["${var.aws_region}a", "${var.aws_region}b"]
  public_subnet_cidrs = ["10.0.0.0/24", "10.0.1.0/24"]

  tags = {
    Environment = "dev"
  }
}

module "web" {
  source = "./modules/web-server"

  name               = "${var.project}-app"
  vpc_id             = module.network.vpc_id            # <- module output as input
  subnet_id          = module.network.public_subnet_ids[0]
  instance_type      = "t3.micro"
  allowed_http_cidrs = [var.my_ip_cidr]

  tags = {
    Environment = "dev"
  }

  # Implicit dependency already exists via vpc_id; depends_on is rarely needed.
}
```

10. Calling the same module many times — `for_each` on a module block.

```hcl
locals {
  environments = {
    dev  = "t3.micro"
    test = "t3.small"
  }
}

module "web_per_env" {
  source   = "./modules/web-server"
  for_each = local.environments

  name               = "${var.project}-${each.key}"
  vpc_id             = module.network.vpc_id
  subnet_id          = module.network.public_subnet_ids[0]
  instance_type      = each.value
  allowed_http_cidrs = [var.my_ip_cidr]

  tags = { Environment = each.key }
}
```

Reference the results as `module.web_per_env["dev"].public_ip`.

11. Root module `outputs.tf` — re-export what callers of the root care about.

```hcl
output "vpc_id" {
  description = "VPC created by the network module."
  value       = module.network.vpc_id
}

output "public_subnet_ids" {
  description = "Public subnet IDs."
  value       = module.network.public_subnet_ids
}

output "web_public_ip" {
  description = "Public IP of the web server."
  value       = module.web.public_ip
}

output "web_url" {
  description = "Browse here after apply."
  value       = "http://${module.web.public_ip}"
}
```

12. Source types and version pinning — what you may put in `source`.

```hcl
# Local path: no version argument allowed, read straight from disk.
module "network" {
  source = "./modules/network"
}

# Terraform Registry: version constraint REQUIRED in practice.
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.8.1" # exact pin, most reproducible
}

# Pessimistic constraint: allows 5.8.x patch updates, blocks 5.9.0
module "vpc_patch_only" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.8.0"
}

# Git over SSH, pinned to a tag (immutable). Never leave a git source unpinned.
module "internal" {
  source = "git::ssh://git@github.com/nexskill/tf-modules.git//network?ref=v1.4.0"
}
```

13. Initialize and run. `terraform init` installs modules into `.terraform/modules/`.

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```

If you only changed module sources (not providers), you can refresh just the modules:

```bash
terraform get -update
```

## Expected Output

```bash
$ terraform init
Initializing the backend...
Initializing modules...
- network in modules/network
- web in modules/web-server
Initializing provider plugins...
- Installing hashicorp/aws v5.62.0...
Terraform has been successfully initialized!
```

```
$ terraform apply
module.network.aws_vpc.this: Creation complete after 3s [id=vpc-0a1b2c3d4e5f6a7b8]
module.network.aws_subnet.public[0]: Creation complete after 2s
module.network.aws_internet_gateway.this: Creation complete after 2s
module.web.aws_security_group.web: Creation complete after 3s
module.web.aws_instance.web: Creation complete after 32s [id=i-0123456789abcdef0]

Apply complete! Resources: 9 added, 0 changed, 0 destroyed.

Outputs:

public_subnet_ids = ["subnet-0aaa...", "subnet-0bbb..."]
vpc_id            = "vpc-0a1b2c3d4e5f6a7b8"
web_public_ip     = "54.85.12.34"
web_url           = "http://54.85.12.34"
```

Note the addressing: resources inside a child module are named `module.<name>.<resource>.<label>`.

Module inputs and outputs summarized:

| Module | Input | Type | Default | Purpose |
| --- | --- | --- | --- | --- |
| network | `name` | string | — | Name prefix |
| network | `vpc_cidr` | string | `10.0.0.0/16` | VPC CIDR |
| network | `public_subnet_cidrs` | list(string) | two /24s | Public subnets |
| web-server | `vpc_id` | string | — | From `module.network.vpc_id` |
| web-server | `subnet_id` | string | — | From `module.network.public_subnet_ids[0]` |

| Module | Output | Meaning |
| --- | --- | --- |
| network | `vpc_id` | Created VPC ID |
| network | `public_subnet_ids` | Public subnet IDs |
| web-server | `instance_id` | EC2 instance ID |
| web-server | `public_ip` | Public IP for testing |

## Verification

```bash
terraform state list          # addresses are prefixed with module.<name>
terraform output web_url
terraform providers           # shows provider requirements per module

curl -s "$(terraform output -raw web_url)"
aws ec2 describe-instances --instance-ids "$(terraform output -raw web_public_ip >/dev/null; echo)" 2>/dev/null || true
```

Inspect what init downloaded:

```bash
ls -R .terraform/modules
cat .terraform/modules/modules.json
```

Console check: VPC dashboard shows the VPC, 2 subnets, IGW and route table; EC2 shows one running
instance in that subnet with the module's security group attached.

## Cleanup

```bash
terraform destroy
```

Targeted destroy of just one module (use sparingly, it can leave dependencies dangling):

```bash
terraform destroy -target='module.web'
```

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `Module not installed` / `Module source has changed` | New or edited `module` block not initialized | Run `terraform init` (or `terraform get -update`) |
| `Unsupported argument: version` | `version` used with a local `source = "./..."` path | Remove `version`; local modules are versioned by your VCS |
| `Unsupported argument: <name>` in a module block | Passing an input the child module does not declare | Add the `variable` to the child, or fix the argument name |
| `Missing required argument` | Child variable has no default and none was passed | Pass it, or give the variable a sensible `default` |
| `Unsupported attribute: This object has no argument named "vpc_id"` | Referencing something the child does not `output` | Add an `output` block in the child module |
| `Reference to undeclared module` | Typo in `module.<name>` or module removed | Match the module block label exactly |
| `Error: Cycle: module.a -> module.b -> module.a` | Two modules consume each other's outputs | Break the cycle: pass a plain value, or merge/split the modules |
| Provider config in a child module conflicts with root | `provider` block declared inside the child | Keep `provider` blocks in the root; children only declare `required_providers` |
| Plan shows destroy/recreate after renaming a module | Resource addresses changed | `terraform state mv 'module.old' 'module.new'` or use a `moved` block |
| `Failed to download module: could not download module` | Bad source URL, no network, or missing Git credentials | Verify the source string; for Git use `?ref=<tag>` and confirm SSH access |

## Key Takeaways
- Every configuration is a module; the one you run `terraform apply` in is the root.
- Standard files: `main.tf`, `variables.tf`, `outputs.tf`, `README.md` (+ optional `versions.tf`, `locals.tf`).
- A module's API is its variables (in) and outputs (out) — wire modules together with `module.<name>.<output>`.
- `for_each`/`count` work on module blocks, so one module can produce a whole environment matrix.
- Pin versions for registry and Git modules; local paths take no `version`.
- `terraform init` installs modules into `.terraform/modules/`; re-run it whenever sources change.
- Configure providers only in the root, and read a module's README before using it.

## Next: Launching EC2 with a public Terraform Registry module.
