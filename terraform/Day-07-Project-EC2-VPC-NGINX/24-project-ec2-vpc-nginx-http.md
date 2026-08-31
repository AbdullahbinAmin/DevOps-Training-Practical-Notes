# Day 07 — Project: EC2 + VPC + NGINX + HTTP Access with Terraform

> Build a complete AWS environment from scratch with Terraform: a custom VPC with public and private subnets, an Internet Gateway and route table, a security group, and an EC2 instance that installs NGINX via user data — then open its default page over HTTP in your browser.

## Learning Objectives

By the end of this project you will be able to:

- Structure a multi-file Terraform project (`provider.tf`, `variables.tf`, `vpc.tf`, `security.tf`, `ec2.tf`, `outputs.tf`, `user_data.sh`).
- Create a custom VPC with a public and a private subnet.
- Attach an Internet Gateway and route `0.0.0.0/0` through it for the public subnet.
- Write a security group that allows inbound HTTP (80) and all outbound traffic.
- Use a **data source** to look up the latest Amazon Linux 2023 AMI instead of hardcoding an AMI ID.
- Bootstrap software with `user_data` (install, enable and start NGINX).
- Expose useful values with `output` blocks (public IP, public DNS, ready-to-click URL).
- Verify the deployment with `curl` and a browser, then destroy everything cleanly.

## Prerequisites

| Requirement | Details |
|---|---|
| Terraform | 1.x (`terraform version` → v1.5 or newer) |
| AWS provider | 5.x (pinned in `provider.tf`) |
| AWS account | With permission to create VPC, subnet, IGW, route table, security group, EC2 |
| Credentials | Configured via `aws configure`, environment variables, or an SSO profile. **Never put keys in `.tf` files.** |
| AWS CLI | Optional but handy for verification |
| Region | Defaults to `eu-north-1` (Stockholm) — change with a variable |
| Cost note | `t3.micro`/`t2.micro` may be free-tier eligible. Always run `terraform destroy` when finished. |

> Security warning (read before you start): this lab opens HTTP (port 80) to `0.0.0.0/0` so that anyone can load the NGINX page. That is intentional for a public web server. If you also enable the optional SSH rule, port 22 open to `0.0.0.0/0` is **lab-only** — it exposes your instance to the entire internet. In real environments restrict SSH to your own IP (`x.x.x.x/32`), or better, use AWS Systems Manager Session Manager and no SSH rule at all.

## Architecture

```
                        ┌──────────────┐
                        │   Internet   │
                        └──────┬───────┘
                               │
                    ┌──────────▼──────────┐
                    │  Internet Gateway   │
                    │       (igw)         │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────────┐
                    │ Route Table (public-rt) │
                    │   0.0.0.0/0 -> IGW      │
                    └──────────┬──────────────┘
                               │ association
 ┌─────────────────────────────▼──────────────────────────────────┐
 │ VPC  10.0.0.0/16                                               │
 │                                                                │
 │  ┌───────────────────────────┐   ┌──────────────────────────┐  │
 │  │ Public Subnet 10.0.2.0/24 │   │ Private Subnet           │  │
 │  │       (eu-north-1a)       │   │ 10.0.1.0/24              │  │
 │  │                           │   │ (eu-north-1a)            │  │
 │  │   ┌───────────────────┐   │   │                          │  │
 │  │   │ EC2: nginx-server │   │   │   (no internet route —   │  │
 │  │   │  AL2023 + NGINX   │   │   │    reserved for DB /     │  │
 │  │   │  SG: 80 open      │   │   │    app tier later)       │  │
 │  │   └───────────────────┘   │   │                          │  │
 │  └───────────────────────────┘   └──────────────────────────┘  │
 └────────────────────────────────────────────────────────────────┘

 Browser ──HTTP:80──> Public IP ──> NGINX default page
```

### Component table

| Component | Terraform resource | Purpose |
|---|---|---|
| VPC | `aws_vpc.main` | Isolated `10.0.0.0/16` network |
| Public subnet | `aws_subnet.public` | `10.0.2.0/24`, hosts the web server, auto-assigns public IPs |
| Private subnet | `aws_subnet.private` | `10.0.1.0/24`, no internet route (future app/DB tier) |
| Internet Gateway | `aws_internet_gateway.igw` | Gives the VPC a door to the internet |
| Route table | `aws_route_table.public` | `0.0.0.0/0 → IGW` |
| Association | `aws_route_table_association.public` | Binds the route table to the public subnet |
| Security group | `aws_security_group.nginx` | Inbound 80 (and optional 22), all outbound |
| AMI lookup | `data.aws_ami.al2023` | Latest Amazon Linux 2023 image |
| AZ lookup | `data.aws_availability_zones.available` | Picks a valid AZ for the region |
| EC2 instance | `aws_instance.nginx_server` | Runs NGINX, bootstrapped by `user_data.sh` |
| Outputs | `outputs.tf` | Public IP, public DNS, clickable URL |

## Project File Layout

```
aws-vpc-ec2-nginx/
├── provider.tf      # terraform block, required_providers, AWS provider + region
├── variables.tf     # region, CIDRs, instance type, project name, ssh toggle
├── vpc.tf           # VPC, subnets, IGW, route table, association, AZ data source
├── security.tf      # security group (HTTP 80, optional SSH 22, egress all)
├── ec2.tf           # AMI data source + EC2 instance with user_data
├── outputs.tf       # public IP / DNS / URL / VPC & subnet IDs
└── user_data.sh     # bash bootstrap: install + enable + start nginx
```

Keeping files separate is a convention, not a requirement — Terraform loads every `*.tf` file in the directory as one configuration. Separation just keeps the code readable.

## Step-by-Step Practical

### 1. Create the project directory

```bash
mkdir -p aws-vpc-ec2-nginx
cd aws-vpc-ec2-nginx
```

### 2. `provider.tf` — pin Terraform and the AWS provider

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0" # AWS provider 5.x
    }
  }
}

provider "aws" {
  region = var.aws_region

  # Credentials come from your environment / AWS CLI profile / SSO.
  # NEVER hardcode access keys in Terraform files.

  default_tags {
    tags = {
      Project   = var.project_name
      ManagedBy = "Terraform"
      Env       = "lab"
    }
  }
}
```

### 3. `variables.tf` — make the project configurable

```hcl
variable "aws_region" {
  description = "AWS region to deploy into."
  type        = string
  default     = "eu-north-1" # Stockholm
}

variable "project_name" {
  description = "Name prefix applied to all resources."
  type        = string
  default     = "nginx-lab"
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC."
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidr" {
  description = "CIDR block for the public subnet."
  type        = string
  default     = "10.0.2.0/24"
}

variable "private_subnet_cidr" {
  description = "CIDR block for the private subnet."
  type        = string
  default     = "10.0.1.0/24"
}

variable "instance_type" {
  description = "EC2 instance type."
  type        = string
  default     = "t3.micro"
}

variable "allowed_http_cidr" {
  description = "Who may reach port 80. 0.0.0.0/0 is expected for a public web server."
  type        = string
  default     = "0.0.0.0/0"
}

variable "enable_ssh" {
  description = "LAB ONLY. Adds an inbound SSH rule. Keep false unless you need it."
  type        = bool
  default     = false
}

variable "ssh_cidr" {
  description = "CIDR allowed to SSH. Use YOUR_IP/32 — never 0.0.0.0/0 outside a throwaway lab."
  type        = string
  default     = "0.0.0.0/0"
}

variable "key_name" {
  description = "Optional existing EC2 key pair name for SSH. Leave null to skip."
  type        = string
  default     = null
}
```

### 4. `vpc.tf` — network foundation

```hcl
# --- Pick a valid AZ for whatever region we are in ---
data "aws_availability_zones" "available" {
  state = "available"
}

# --- VPC ---
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# --- Public subnet (web tier) ---
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidr
  availability_zone       = data.aws_availability_zones.available.names[0]
  map_public_ip_on_launch = true # instances get a public IP automatically

  tags = {
    Name = "${var.project_name}-public-subnet"
    Tier = "public"
  }
}

# --- Private subnet (no internet route; reserved for app/DB tier) ---
resource "aws_subnet" "private" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.private_subnet_cidr
  availability_zone       = data.aws_availability_zones.available.names[0]
  map_public_ip_on_launch = false

  tags = {
    Name = "${var.project_name}-private-subnet"
    Tier = "private"
  }
}

# --- Internet Gateway ---
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

# --- Public route table: all outbound traffic via the IGW ---
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

# --- Bind the route table to the public subnet ---
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

A subnet is "public" only because its route table points at an Internet Gateway. Without the association in the last block, the instance would have a public IP but no path to the internet.

### 5. `security.tf` — firewall rules

```hcl
resource "aws_security_group" "nginx" {
  name        = "${var.project_name}-nginx-sg"
  description = "Allow inbound HTTP to NGINX and all outbound traffic"
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-nginx-sg"
  }

  lifecycle {
    create_before_destroy = true
  }
}

# --- Ingress: HTTP 80 from the internet (intended for a public web server) ---
resource "aws_vpc_security_group_ingress_rule" "http" {
  security_group_id = aws_security_group.nginx.id
  description       = "HTTP from anywhere"
  cidr_ipv4         = var.allowed_http_cidr
  from_port         = 80
  to_port           = 80
  ip_protocol       = "tcp"
}

# --- Ingress: SSH 22 — LAB ONLY, disabled by default ---
# WARNING: with ssh_cidr = "0.0.0.0/0" this exposes SSH to the entire internet.
# Acceptable only in a short-lived throwaway lab. In real environments use
# YOUR_IP/32, a bastion host, or AWS SSM Session Manager (no SSH rule at all).
resource "aws_vpc_security_group_ingress_rule" "ssh" {
  count = var.enable_ssh ? 1 : 0

  security_group_id = aws_security_group.nginx.id
  description       = "SSH (lab only)"
  cidr_ipv4         = var.ssh_cidr
  from_port         = 22
  to_port           = 22
  ip_protocol       = "tcp"
}

# --- Egress: allow everything out (needed to download nginx packages) ---
resource "aws_vpc_security_group_egress_rule" "all" {
  security_group_id = aws_security_group.nginx.id
  description       = "All outbound traffic"
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1" # all protocols and ports
}
```

Provider 5.x prefers the standalone `aws_vpc_security_group_*_rule` resources over inline `ingress {}` / `egress {}` blocks — separate rules can be added and removed without recreating the whole group.

### 6. `user_data.sh` — bootstrap NGINX

```bash
#!/bin/bash
set -euxo pipefail

# Amazon Linux 2023 uses dnf (yum is a symlink) and the package is "nginx".
dnf -y update
dnf -y install nginx

# Simple custom landing page so we can prove OUR page is being served.
cat >/usr/share/nginx/html/index.html <<'HTML'
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>Terraform Lab - NGINX</title>
  </head>
  <body>
    <h1>Hello from NGINX on EC2</h1>
    <p>Provisioned end-to-end with Terraform: VPC, subnets, IGW, route table,
       security group, EC2 and user data.</p>
  </body>
</html>
HTML

systemctl enable nginx
systemctl start nginx
```

`enable` makes NGINX survive a reboot; `start` brings it up now. User data runs as root on the **first** boot only.

### 7. `ec2.tf` — AMI data source + instance

```hcl
# --- Latest Amazon Linux 2023 AMI (never hardcode AMI IDs: they are region-specific) ---
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}

resource "aws_instance" "nginx_server" {
  ami           = data.aws_ami.al2023.id
  instance_type = var.instance_type

  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.nginx.id]
  associate_public_ip_address = true
  key_name                    = var.key_name

  # Bootstrap script. base64gzip keeps us well inside the 16 KB user-data limit.
  user_data                   = file("${path.module}/user_data.sh")
  user_data_replace_on_change = true # re-create the instance if the script changes

  root_block_device {
    volume_size           = 8
    volume_type           = "gp3"
    encrypted             = true
    delete_on_termination = true
  }

  metadata_options {
    http_tokens   = "required" # enforce IMDSv2
    http_endpoint = "enabled"
  }

  tags = {
    Name = "${var.project_name}-nginx-server"
    Role = "web"
  }

  # Network must exist before the instance tries to reach the internet.
  depends_on = [aws_route_table_association.public]
}
```

### 8. `outputs.tf` — surface the useful values

```hcl
output "instance_id" {
  description = "EC2 instance ID."
  value       = aws_instance.nginx_server.id
}

output "instance_public_ip" {
  description = "Public IPv4 address of the NGINX server."
  value       = aws_instance.nginx_server.public_ip
}

output "instance_public_dns" {
  description = "Public DNS name of the NGINX server."
  value       = aws_instance.nginx_server.public_dns
}

output "nginx_url" {
  description = "Open this in your browser."
  value       = "http://${aws_instance.nginx_server.public_ip}"
}

output "ami_id" {
  description = "AMI ID resolved by the Amazon Linux 2023 data source."
  value       = data.aws_ami.al2023.id
}

output "vpc_id" {
  description = "ID of the created VPC."
  value       = aws_vpc.main.id
}

output "subnet_ids" {
  description = "Public and private subnet IDs."
  value = {
    public  = aws_subnet.public.id
    private = aws_subnet.private.id
  }
}
```

## Deploy

```bash
# 1. Download the AWS provider and initialise the backend
terraform init

# 2. Normalise formatting (recursively)
terraform fmt -recursive

# 3. Check syntax and internal consistency
terraform validate

# 4. Preview exactly what will be created — read this before applying
terraform plan -out=tfplan

# 5. Create the infrastructure
terraform apply tfplan
#    (or, without a saved plan:  terraform apply )

# 6. Re-read outputs at any time
terraform output
terraform output -raw nginx_url
```

Optional overrides without editing files:

```bash
terraform apply -var="aws_region=eu-central-1" -var="instance_type=t2.micro"
```

## Expected Output

`terraform plan` ends with something like:

```
Plan: 9 to add, 0 to change, 0 to destroy.
```

(VPC, public subnet, private subnet, IGW, route table, association, security group + 3 rules, instance — the exact count varies with the optional SSH rule.)

`terraform apply` finishes with:

```
Apply complete! Resources: 9 added, 0 changed, 0 destroyed.

Outputs:

ami_id              = "ami-0abcdef1234567890"
instance_id         = "i-0a1b2c3d4e5f60718"
instance_public_dns = "ec2-54-93-27-123.eu-north-1.compute.amazonaws.com"
instance_public_ip  = "54.93.27.123"
nginx_url           = "http://54.93.27.123"
subnet_ids          = {
  "private" = "subnet-0aaa111bbb222ccc3"
  "public"  = "subnet-0ddd444eee555fff6"
}
vpc_id              = "vpc-0123456789abcdef0"
```

(IPs and IDs will differ in your account.)

## Verification

Give the instance 60–120 seconds after `apply` — user data still has to install NGINX.

```bash
# 1. Terraform-side: read the URL
terraform output -raw nginx_url

# 2. HTTP status only — expect 200
curl -s -o /dev/null -w "%{http_code}\n" "$(terraform output -raw nginx_url)"

# 3. Full page body
curl -i "http://$(terraform output -raw instance_public_ip)"

# 4. Via DNS name instead of IP
curl -s "http://$(terraform output -raw instance_public_dns)" | head -20
```

Expected: `HTTP/1.1 200 OK`, `Server: nginx/1.2x.x`, and the "Hello from NGINX on EC2" heading.

Browser test: copy the `nginx_url` value into your browser. You should see the custom page (or, if you removed the `index.html` block from the script, NGINX's "Welcome to nginx!" default page).

Console checklist:

| Where | What to confirm |
|---|---|
| VPC → Your VPCs | `nginx-lab-vpc` with CIDR `10.0.0.0/16` |
| VPC → Subnets | Public `10.0.2.0/24` and private `10.0.1.0/24` |
| VPC → Internet gateways | `nginx-lab-igw`, state **Attached** |
| VPC → Route tables | `nginx-lab-public-rt` has `0.0.0.0/0 → igw-…` and is associated with the public subnet |
| EC2 → Security groups | `nginx-lab-nginx-sg` inbound shows TCP 80 |
| EC2 → Instances | `nginx-lab-nginx-server` **running**, 2/2 checks passed, has a public IP |

If SSH is enabled, you can inspect the bootstrap directly:

```bash
ssh -i ~/.ssh/your-key.pem ec2-user@$(terraform output -raw instance_public_ip)
sudo systemctl status nginx
sudo cat /var/log/cloud-init-output.log   # user_data execution log
```

## Cleanup

Always destroy the lab — a running instance and an unused Elastic IP cost money.

```bash
# Preview the deletions first
terraform plan -destroy

# Tear everything down
terraform destroy
# Type: yes
```

Expected tail:

```
Destroy complete! Resources: 9 destroyed.
```

Then confirm in the console that the instance is `terminated` and the VPC is gone. If `destroy` hangs on the VPC, something outside Terraform (a manually created ENI, NAT gateway, or endpoint) is still attached — delete it, then re-run.

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `curl: (7) Failed to connect ... port 80: Connection refused` | NGINX not up yet, or user data failed | Wait 1–2 min; then check `/var/log/cloud-init-output.log` on the instance |
| `curl` hangs / times out (no refusal) | Security group missing the port 80 ingress rule, or no route to IGW | Verify `aws_vpc_security_group_ingress_rule.http` and the route table association |
| `InvalidAMIID.NotFound` | Hardcoded AMI from another region | Use the `data.aws_ami.al2023` lookup as shown — never a literal AMI ID |
| `Your query returned no results` (data.aws_ami) | Name filter or architecture doesn't match the region's images | Loosen the filter to `al2023-ami-*-x86_64`; use `arm64` if you chose a Graviton instance type |
| `InvalidParameterValue: Value (t3.micro) ... not supported` | Instance type unavailable in that AZ | Change `instance_type`, or pick another AZ from the data source |
| `Error launching source instance: Unsupported` / no public IP | `map_public_ip_on_launch = false` on the subnet | Set it to `true` (or `associate_public_ip_address = true` on the instance) |
| `UnauthorizedOperation` / `AccessDenied` | IAM user lacks EC2/VPC permissions | Attach a policy allowing the `ec2:*` actions used, or use an admin lab account |
| `NoCredentialProviders` / `no valid credential sources` | AWS credentials not configured | Run `aws configure`, or export `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_REGION` |
| `InvalidSubnet.Range` / `InvalidVpc.Range` | Subnet CIDR outside the VPC CIDR, or overlapping subnets | Keep subnets inside `10.0.0.0/16` and non-overlapping |
| `VpcLimitExceeded` | Default 5 VPCs per region reached | Delete unused VPCs or request a quota increase |
| `DependencyViolation` during destroy | Manually created resource still inside the VPC | Remove it in the console, then re-run `terraform destroy` |
| Changed `user_data.sh` but nothing happened | User data only runs on first boot | `user_data_replace_on_change = true` (already set) forces instance replacement |
| `dnf: command not found` in cloud-init log | You used an Amazon Linux 2 / Ubuntu AMI | Match the package manager to the AMI (`yum` for AL2, `apt-get` for Ubuntu) |
| `Instance does not have a public IPv4 address` | Instance launched into the private subnet | Set `subnet_id = aws_subnet.public.id` |

## Key Takeaways

- A VPC gives you an isolated, custom network; nothing in it is reachable until you add an IGW **and** a route to it.
- "Public subnet" is not a property — it is a subnet whose route table sends `0.0.0.0/0` to an Internet Gateway.
- Security groups are stateful allow-lists: open 80 inbound for the web server, and keep egress open so the instance can download packages.
- Data sources (`aws_ami`, `aws_availability_zones`) keep configurations portable across regions and free of stale hardcoded IDs.
- `user_data` turns a bare AMI into a working service, making the whole stack reproducible with no manual SSH.
- Outputs are the project's API — expose IP, DNS and URL so humans and other modules can consume them.
- Splitting code into `provider/variables/vpc/security/ec2/outputs` files costs nothing at runtime and pays for itself in readability.
- Opening SSH to `0.0.0.0/0` is a lab shortcut, not a pattern. Restrict to your own IP or use SSM Session Manager.
- Finish every lab with `terraform destroy` so you are not billed for idle infrastructure.

## Next: [Terraform Data Source](../Day-08-Data-Sources/25-terraform-data-source.md)
