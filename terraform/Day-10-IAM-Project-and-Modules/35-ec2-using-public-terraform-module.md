# Day 10 — AWS EC2 using a Public Terraform Module

> Launch a VPC and an EC2 instance with the community `terraform-aws-modules` packages from the public registry, with pinned versions.

## Learning Objectives
- Find and evaluate a module on `registry.terraform.io`.
- Read registry docs correctly: Inputs, Outputs, Examples, Versions, Submodules.
- Call `terraform-aws-modules/vpc/aws` and `terraform-aws-modules/ec2-instance/aws` with pinned versions.
- Wire one module's outputs into another module's inputs.
- Understand what `terraform init` does with registry modules and where it stores them.

## Prerequisites
- Terraform 1.x, AWS provider 5.x, credentials configured outside the code.
- Internet access — `terraform init` downloads modules from the registry.
- A default or new VPC quota headroom in the target region.
- Empty folder, e.g. `tf-module-ec2/`.

## Concept (plain English)
You do not have to write VPC and EC2 resources by hand. The public registry hosts maintained modules,
and `terraform-aws-modules` is the most widely used AWS collection: popular, well maintained, covers
many use cases, and exposes a lot of input options.

How to find one:

```
registry.terraform.io  ->  Browse Modules  ->  filter: AWS  ->  search "ec2 instance"
   -> terraform-aws-modules/ec2-instance/aws   (by terraform-aws-modules)
```

The source address always has three parts: `<NAMESPACE>/<NAME>/<PROVIDER>`.

When you run `terraform init`, Terraform resolves the module from the registry, downloads it and
caches it locally:

```
tf-module-ec2/
├── versions.tf
├── provider.tf
├── main.tf                 <- your ROOT module
├── variables.tf
├── outputs.tf
└── .terraform/
    └── modules/
        ├── modules.json    <- resolved versions and local paths
        ├── vpc/            <- downloaded child module
        └── ec2/            <- downloaded child module
```

Always pin `version`. Without it, a later `init` in a clean checkout can pull a new major release and
change or destroy infrastructure. `.terraform/modules/` is a cache, not source — never commit it.

Before writing any code, read the module docs and note four things: **Inputs** (what you may set),
**Outputs** (what you get back), **Examples** (working copy-paste starting points) and **Versions**
(latest, plus the changelog for breaking changes). Community modules are not part of Terraform core;
their inputs change between major versions, so the docs for *your pinned version* are the truth.

## Step-by-Step Practical

1. Create the folder.

```bash
mkdir -p tf-module-ec2 && cd tf-module-ec2
```

2. `versions.tf` — pin Terraform and the provider.

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

3. `provider.tf` — region and default tags. Credentials stay out of the code.

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Terraform   = "true"
      Environment = var.environment
      Project     = var.project
    }
  }
}
```

4. `variables.tf`.

```hcl
variable "aws_region" {
  description = "AWS region to deploy into."
  type        = string
  default     = "us-east-1"
}

variable "project" {
  description = "Name prefix for all resources."
  type        = string
  default     = "day10-ec2"
}

variable "environment" {
  description = "Environment tag."
  type        = string
  default     = "dev"
}

variable "instance_type" {
  description = "EC2 instance type."
  type        = string
  default     = "t3.micro"
}

variable "my_ip_cidr" {
  description = "Your public IP in CIDR form. Used to scope HTTP ingress."
  type        = string
  default     = "0.0.0.0/0"

  validation {
    condition     = can(cidrnetmask(var.my_ip_cidr))
    error_message = "my_ip_cidr must be a valid CIDR, e.g. 203.0.113.25/32."
  }
}
```

5. `terraform.tfvars` — set your own IP so the instance is not open to the world.

```hcl
aws_region    = "us-east-1"
project       = "day10-ec2"
environment   = "dev"
instance_type = "t3.micro"
my_ip_cidr    = "203.0.113.25/32"
```

6. `main.tf` — data sources, the VPC module, a security group, and the EC2 module.

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

# Latest Amazon Linux 2023 AMI - never hardcode an AMI ID; they differ per region.
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

locals {
  azs             = slice(data.aws_availability_zones.available.names, 0, 2)
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
}
```

7. The VPC module — `terraform-aws-modules/vpc/aws`, pinned.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.8.1" # exact pin; check the registry Versions tab before bumping

  name = "${var.project}-vpc"
  cidr = "10.0.0.0/16"

  azs             = local.azs
  public_subnets  = local.public_subnets
  private_subnets = local.private_subnets

  # Public subnets need an IGW route; keep NAT off to avoid hourly cost in a lab.
  enable_nat_gateway = false
  single_nat_gateway = false

  enable_dns_hostnames = true
  enable_dns_support   = true

  map_public_ip_on_launch = true

  tags = {
    Environment = var.environment
  }
}
```

8. A security group for the instance. Scoped ingress, no SSH — use SSM for shell access.

```hcl
module "web_sg" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "5.1.2"

  name        = "${var.project}-web-sg"
  description = "HTTP from my IP only; all egress allowed."
  vpc_id      = module.vpc.vpc_id

  ingress_with_cidr_blocks = [
    {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      description = "HTTP from my IP"
      cidr_blocks = var.my_ip_cidr
    },
  ]

  egress_rules = ["all-all"]

  tags = {
    Environment = var.environment
  }
}
```

9. The EC2 module — `terraform-aws-modules/ec2-instance/aws`, pinned. Inputs come from the VPC module's outputs.

```hcl
module "ec2" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "5.6.1"

  name = "${var.project}-web"

  ami                    = data.aws_ami.al2023.id
  instance_type           = var.instance_type
  subnet_id              = module.vpc.public_subnets[0]        # <- VPC module output
  vpc_security_group_ids = [module.web_sg.security_group_id]   # <- SG module output

  associate_public_ip_address = true

  # IMDSv2 only.
  metadata_options = {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  root_block_device = [
    {
      volume_type = "gp3"
      volume_size = 8
      encrypted   = true
    },
  ]

  user_data_replace_on_change = true
  user_data                   = <<-EOT
    #!/bin/bash
    dnf -y install nginx
    echo "Hello from ${var.project} via a public Terraform module" > /usr/share/nginx/html/index.html
    systemctl enable --now nginx
  EOT

  tags = {
    Terraform   = "true"
    Environment = var.environment
  }
}
```

10. `outputs.tf`.

```hcl
output "vpc_id" {
  description = "VPC created by the vpc module."
  value       = module.vpc.vpc_id
}

output "public_subnet_ids" {
  description = "Public subnet IDs from the vpc module."
  value       = module.vpc.public_subnets
}

output "instance_id" {
  description = "EC2 instance ID."
  value       = module.ec2.id
}

output "instance_public_ip" {
  description = "Public IP of the instance."
  value       = module.ec2.public_ip
}

output "security_group_id" {
  description = "Security group attached to the instance."
  value       = module.web_sg.security_group_id
}

output "web_url" {
  description = "Open this from the IP you allowed."
  value       = "http://${module.ec2.public_ip}"
}
```

11. Initialize — this is the step that downloads the modules.

```bash
terraform init
```

12. Format, validate, preview, apply.

```bash
terraform fmt -recursive
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

13. Inspect what the registry gave you, without leaving the terminal.

```bash
cat .terraform/modules/modules.json | head -40
ls .terraform/modules/vpc
terraform providers
```

## Expected Output

```bash
$ terraform init
Initializing the backend...
Initializing modules...
Downloading registry.terraform.io/terraform-aws-modules/vpc/aws 5.8.1 for vpc...
- vpc in .terraform/modules/vpc
Downloading registry.terraform.io/terraform-aws-modules/security-group/aws 5.1.2 for web_sg...
- web_sg in .terraform/modules/web_sg
Downloading registry.terraform.io/terraform-aws-modules/ec2-instance/aws 5.6.1 for ec2...
- ec2 in .terraform/modules/ec2
Initializing provider plugins...
- Installing hashicorp/aws v5.62.0...
Terraform has been successfully initialized!
```

```
$ terraform apply tfplan
module.vpc.aws_vpc.this[0]: Creation complete after 3s [id=vpc-0abc123def4567890]
module.vpc.aws_subnet.public[0]: Creation complete after 2s
module.vpc.aws_internet_gateway.this[0]: Creation complete after 2s
module.vpc.aws_route_table_association.public[0]: Creation complete after 1s
module.web_sg.aws_security_group.this_name_prefix[0]: Creation complete after 3s
module.ec2.aws_instance.this[0]: Still creating... [30s elapsed]
module.ec2.aws_instance.this[0]: Creation complete after 34s [id=i-0123456789abcdef0]

Apply complete! Resources: 17 added, 0 changed, 0 destroyed.

Outputs:

instance_id        = "i-0123456789abcdef0"
instance_public_ip = "54.85.12.34"
public_subnet_ids  = ["subnet-0aaa111", "subnet-0bbb222"]
security_group_id  = "sg-0ccc333"
vpc_id             = "vpc-0abc123def4567890"
web_url            = "http://54.85.12.34"
```

What the `ec2-instance` module creates behind the scenes: the EC2 instance, its root EBS volume, the
ENI attachment, and all tags. The `vpc` module creates the VPC, subnets, route tables, internet
gateway and default security group.

## Verification

```bash
terraform state list
terraform output web_url

curl -s "$(terraform output -raw web_url)"

aws ec2 describe-instances \
  --instance-ids "$(terraform output -raw instance_id)" \
  --query 'Reservations[].Instances[].{State:State.Name,Type:InstanceType,IP:PublicIpAddress,Subnet:SubnetId,SG:SecurityGroups[].GroupId}'
```

Console checks:

| Check | What to verify |
| --- | --- |
| EC2 -> Instances | Instance state is `running`, correct instance type |
| Networking tab | Correct VPC, expected public subnet, module security group attached |
| Storage tab | gp3 root volume, encrypted |
| Tags tab | `Terraform = true`, `Environment`, `Name` all present |
| VPC dashboard | VPC, 2 public + 2 private subnets, IGW, route tables |

`curl` should return the `Hello from ...` page. If it hangs, your `my_ip_cidr` probably does not match
your current public IP — check with `curl -s https://checkip.amazonaws.com`.

## Cleanup

```bash
terraform destroy
```

Confirm nothing lingers:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=day10-ec2" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].InstanceId'
```

Do not leave lab resources running. Even a `t3.micro` plus an unused NAT gateway or Elastic IP bills
by the hour.

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `Module not installed` | `init` not run after adding a module block | `terraform init` |
| `Failed to download module ... registry service unreachable` | No internet / proxy blocking `registry.terraform.io` | Check connectivity, set `HTTPS_PROXY`, or vendor the module locally |
| `Unsupported argument` on a module input | Input name differs in the version you pinned | Open the registry Inputs tab for that exact version; module APIs change across majors |
| `Module ... does not have a version matching "~> 6.0"` | Constraint has no published release | Check the Versions tab and pick a real version |
| `InvalidAMIID.NotFound` | AMI ID hardcoded from another region | Use the `aws_ami` data source as shown |
| `VcpuLimitExceeded` / `InstanceLimitExceeded` | Account quota reached | Terminate unused instances or request a quota increase |
| `UnsupportedOperation: t3.micro not supported in this AZ` | Instance type unavailable in the chosen AZ | Pick another AZ or instance type (`t3a.micro`, `t2.micro`) |
| Instance created but no public IP | `map_public_ip_on_launch` false or private subnet used | Use `module.vpc.public_subnets[0]` and `associate_public_ip_address = true` |
| `curl` times out | Security group ingress does not include your current IP | Update `my_ip_cidr` and re-apply |
| `DependencyViolation` during destroy | Something outside Terraform attached to the VPC (manual ENI, SG) | Remove the manual resource, then re-run destroy |
| `Error: Invalid count argument` inside the module | Passing an unknown value (computed at apply) into a count-driven input | Pass a static list, or split the apply with `-target` for the first run |

## Key Takeaways
- The registry is your first stop: `registry.terraform.io -> Browse Modules -> AWS`, then read Inputs, Outputs, Examples and Versions.
- Source format is `<NAMESPACE>/<NAME>/<PROVIDER>`; always add a pinned `version`.
- `terraform init` downloads registry modules into `.terraform/modules/` and records resolutions in `modules.json` — that folder is cache, keep it out of Git.
- Compose modules: `module.vpc.vpc_id` and `module.vpc.public_subnets[0]` feed the EC2 module directly, and Terraform infers the dependency order.
- Modules make configuration reusable, clean and maintainable, but you still own the security posture: scoped ingress, encrypted volumes, IMDSv2, no hardcoded credentials.
- Start from the module's `examples/` directory, then customize — do not invent inputs.
- Destroy when the lab is over.

## Congratulations — Course Complete

You have gone from `terraform init` on a single resource to composing public registry modules and
running a data-driven IAM project. Suggested next steps:

- Move state to a remote backend: S3 with native state locking (or DynamoDB on older setups), versioning and encryption enabled.
- Adopt workspaces or separate directories per environment (`dev`/`stage`/`prod`) with per-environment `.tfvars`.
- Write and publish your own module, tagged with semantic versions, and consume it via a pinned Git or registry source.
- Add automated checks: `terraform fmt -check`, `terraform validate`, `tflint`, `checkov` or `tfsec` in CI.
- Learn `import` blocks and `terraform state mv` for adopting existing infrastructure safely.
- Explore `moved` blocks, `precondition`/`postcondition` and `check` blocks for safer refactors.
- Test infrastructure with the native `terraform test` framework or Terratest.
- Build a CI/CD pipeline (GitHub Actions, GitLab CI) that runs plan on pull requests and apply on merge, with OIDC-based AWS auth instead of long-lived keys.
- Study Terragrunt or Terraform Stacks for large multi-account setups.
