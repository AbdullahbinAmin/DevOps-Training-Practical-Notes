# Day 06 — AWS VPC Using Terraform

> Rebuild the same VPC, subnets, IGW, route table and EC2 as code — repeatable, reviewable and destroyable with one command.

## Learning Objectives
- Write a complete `main.tf` that provisions a VPC, two subnets, an IGW, a route table, a route and an association.
- Understand implicit dependencies created by resource references such as `aws_vpc.my_vpc.id`.
- Run the core workflow: `init`, `validate`, `plan`, `apply`, `destroy`.
- Use variables and outputs instead of hard-coded values.
- Verify the result in the AWS Console and with `terraform state`.

## Prerequisites
- Terraform >= 1.5 installed (`terraform version`).
- AWS provider 5.x (pulled automatically by `terraform init`).
- AWS credentials available via `aws configure`, environment variables or an IAM role. Never put keys in `.tf` files.
- Note 22 completed and its manual resources destroyed, so CIDRs do not clash.

## Concept

### Plan (step by step, as on the slide)
1. Create VPC — `10.0.0.0/16`
2. Create 2 subnets — public `10.0.1.0/24`, private `10.0.2.0/24`
3. Create Internet Gateway
4. Create Route Table
5. Add route `0.0.0.0/0 -> IGW` for the public subnet
6. Associate the public subnet with the route table

### Flow

```
[1 VPC 10.0.0.0/16] -> [2 Subnets pub 10.0.1.0/24 / priv 10.0.2.0/24] -> [3 IGW]
     -> [4 Route Table] -> [5 Route 0.0.0.0/0 -> IGW] -> [6 Associate public subnet]
     -> [Resources: EC2, RDS ...] -> Internet
```

### Refresher on the vocabulary

| Term | One-line meaning |
|---|---|
| CIDR | IP range as `address/prefix`; `/16` = 65,536 IPs, `/24` = 256 IPs |
| Subnet | Slice of the VPC CIDR living in one Availability Zone |
| Route table | Destination -> target rules; `local` stays inside the VPC |
| IGW | Two-way internet door for public subnets |
| NAT GW | Outbound-only internet for private subnets (hourly cost) |
| Security Group | Stateful, instance-level firewall, allow-rules only |
| Network ACL | Stateless, subnet-level firewall, allow and deny, ordered |

## Step-by-Step Practical

1. Create the working directory.

```bash
mkdir -p ~/terraform-labs/day06-vpc && cd ~/terraform-labs/day06-vpc
```

2. Create `versions.tf` — provider and version constraints.

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

provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Project   = "day06-vpc"
      ManagedBy = "terraform"
    }
  }
}
```

3. Create `variables.tf` so nothing important is hard-coded.

```hcl
variable "region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "eu-north-1" # Stockholm
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidr" {
  description = "CIDR block for the public subnet"
  type        = string
  default     = "10.0.1.0/24"
}

variable "private_subnet_cidr" {
  description = "CIDR block for the private subnet"
  type        = string
  default     = "10.0.2.0/24"
}

variable "availability_zone" {
  description = "AZ for both subnets in this lab"
  type        = string
  default     = "eu-north-1a"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "ssh_allowed_cidr" {
  description = "CIDR allowed to SSH. Set to <your-ip>/32. Never 0.0.0.0/0 outside a lab."
  type        = string
  default     = "0.0.0.0/0"
}
```

4. Create `main.tf` — the networking layer. Note how every child resource references `aws_vpc.my_vpc.id`; that reference is what tells Terraform the correct creation order, so you never write dependencies by hand.

```hcl
# --- 1. VPC ---
resource "aws_vpc" "my_vpc" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "my-vpc"
  }
}

# --- 2. Subnets ---
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.my_vpc.id
  cidr_block              = var.public_subnet_cidr
  availability_zone       = var.availability_zone
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet"
  }
}

resource "aws_subnet" "private_subnet" {
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = var.private_subnet_cidr
  availability_zone = var.availability_zone

  tags = {
    Name = "private-subnet"
  }
}

# --- 3. Internet Gateway ---
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "my-igw"
  }
}
```

5. Append the routing to `main.tf`.

```hcl
# --- 4. Route Table ---
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "public-route-table"
  }
}

# --- 5. Route in Route Table ---
resource "aws_route" "public_route" {
  route_table_id         = aws_route_table.public_rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}

# --- 6. Associate Public Subnet with Route Table ---
resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}
```

The private subnet is deliberately left on the VPC's main route table, which has only the `local` route — that is what keeps it private.

6. Create `ec2.tf` — an optional web server in the public subnet. Look up the AMI dynamically instead of hard-coding an ID like `ami-0c02fb55956c7d316`, which is region-specific and goes stale.

**Security warning:** the default value of `ssh_allowed_cidr` is `0.0.0.0/0`, meaning the entire internet may attempt SSH. That is only acceptable in a throwaway lab. In any shared or production environment set it to `<your-ip>/32`, or drop port 22 entirely and use AWS Systems Manager Session Manager.

```hcl
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}
```

7. Add the security group and the instance to `ec2.tf`.

```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Allow HTTP from anywhere and SSH from a trusted CIDR"
  vpc_id      = aws_vpc.my_vpc.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    # LAB ONLY when this is 0.0.0.0/0 - restrict to <your-ip>/32
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.ssh_allowed_cidr]
  }

  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "web-sg"
  }
}

resource "aws_instance" "web_server" {
  ami                         = data.aws_ami.al2023.id
  instance_type               = var.instance_type
  subnet_id                   = aws_subnet.public_subnet.id
  vpc_security_group_ids      = [aws_security_group.web_sg.id]
  associate_public_ip_address = true

  tags = {
    Name = "sample-server"
  }
}
```

8. Create `outputs.tf`.

```hcl
output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.my_vpc.id
}

output "public_subnet_cidr" {
  description = "CIDR block of the public subnet"
  value       = aws_subnet.public_subnet.cidr_block
}

output "private_subnet_cidr" {
  description = "CIDR block of the private subnet"
  value       = aws_subnet.private_subnet.cidr_block
}

output "internet_gateway_id" {
  description = "ID of the Internet Gateway"
  value       = aws_internet_gateway.igw.id
}

output "web_server_public_ip" {
  description = "Public IP of the sample web server"
  value       = aws_instance.web_server.public_ip
}
```

9. Run the workflow.

```bash
terraform init
terraform fmt
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

10. Read the outputs.

```bash
terraform output
terraform output -raw web_server_public_ip
```

## Expected Output

```text
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Plan: 8 to add, 0 to change, 0 to destroy.

aws_vpc.my_vpc: Creating...
aws_vpc.my_vpc: Creation complete after 3s [id=vpc-0a1b2c3d4e5f6a7b8]
aws_internet_gateway.igw: Creation complete after 1s [id=igw-05f6a7b8c9d0e1f23]
aws_subnet.public_subnet: Creation complete after 2s [id=subnet-0aa11bb22cc33dd44]
aws_subnet.private_subnet: Creation complete after 2s [id=subnet-0ee55ff66aa77bb88]
aws_route_table.public_rt: Creation complete after 2s [id=rtb-0c1d2e3f4a5b6c7d8]
aws_route.public_route: Creation complete after 1s
aws_route_table_association.public_assoc: Creation complete after 1s
aws_instance.web_server: Creation complete after 32s [id=i-0123456789abcdef0]

Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

Outputs:

internet_gateway_id  = "igw-05f6a7b8c9d0e1f23"
private_subnet_cidr  = "10.0.2.0/24"
public_subnet_cidr   = "10.0.1.0/24"
vpc_id               = "vpc-0a1b2c3d4e5f6a7b8"
web_server_public_ip = "13.53.xxx.xxx"
```

## Verification

Console — VPC Dashboard, check each item:
- Your VPC `my-vpc` with CIDR `10.0.0.0/16`
- 2 subnets (public `10.0.1.0/24`, private `10.0.2.0/24`) in `eu-north-1a`
- Internet Gateway `my-igw`, state **Attached**
- Route table `public-route-table` with these routes:

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | igw-xxxxxxxx |

- Subnet association: `public-subnet` listed under the public route table
- The instance in the public subnet has a private IP from `10.0.1.0/24` plus a public IP

CLI:

```bash
terraform state list
terraform show -json | jq '.values.root_module.resources[].address'

aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$(terraform output -raw vpc_id)" \
  --query 'RouteTables[].Routes' --output table
```

A clean run of `terraform plan` after apply should report **No changes** — proof the code matches reality.

## Cleanup

Destroy everything so the lab costs nothing. This is irreversible — read the plan before confirming.

```bash
terraform plan -destroy
terraform destroy
```

Expect `Destroy complete! Resources: 8 destroyed.` Then confirm the VPC is gone:

```bash
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=my-vpc" --query 'Vpcs'
```

Local state files (`terraform.tfstate*`, `.terraform/`) can be removed afterwards; never commit them to git.

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `Error: No valid credential sources found` | Credentials not configured | `aws configure`, or export `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` |
| `InvalidParameterValue: Value (eu-north-1a) for parameter availabilityZone is invalid` | AZ does not belong to the provider region | Match `availability_zone` to `region`, or use a `data "aws_availability_zones"` lookup |
| `InvalidSubnet.Range` | Subnet CIDR outside the VPC CIDR | Keep subnets inside `10.0.0.0/16` |
| `RouteAlreadyExists` | A `0.0.0.0/0` route already exists in that table, often from an inline `route` block | Use either `aws_route` resources or inline `route` blocks, never both on the same table |
| `Error: creating EC2 Instance: InvalidAMIID.NotFound` | Hard-coded AMI belongs to another region | Use the `data "aws_ami"` lookup shown above |
| `DependencyViolation` during destroy | Resources created outside Terraform inside this VPC | Delete the manual resources, then re-run `terraform destroy` |
| `Error acquiring the state lock` | A previous run crashed | Wait, then `terraform force-unlock <lock-id>` only if you are certain no run is active |
| Instance reachable on port 22 from anywhere | `ssh_allowed_cidr` left at `0.0.0.0/0` | Set it to `<your-ip>/32` or remove the SSH rule |
| `Provider produced inconsistent final plan` | Provider version drift | Pin with `version = "~> 5.0"` and commit `.terraform.lock.hcl` |

## Key Takeaways
- VPC is your own private network inside AWS; subnets divide it into smaller parts.
- Internet Gateway + a route table entry is what makes a subnet public; the private subnet stays isolated with no direct internet route.
- Resource references (`aws_vpc.my_vpc.id`) create the dependency graph automatically — no manual ordering needed.
- Commands used: `terraform init`, `terraform validate`, `terraform plan`, `terraform apply`, and `terraform destroy` to clean up.
- Variables and outputs keep the config reusable across regions and environments; look AMIs up with a data source instead of hard-coding IDs.
- The same build that took many console clicks in note 22 is now repeatable and version-controlled.
- Keep SSH restricted and never store credentials in `.tf` files or state in git.

## Next: [Day 07 — Terraform Modules](../Day-07-Modules/README.md)

