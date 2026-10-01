# Chapter 36 — Terraform Modules: Overview & Public VPC Module

## Objective
Understand the architecture of **Terraform Modules** and provision a production-grade AWS VPC using the official public **`terraform-aws-modules/vpc/aws`** module from the Terraform Registry.

## Prerequisites
- AWS credentials loaded (`source .env`)
- Region configured (e.g. `eu-north-1`)
- Terraform 1.x installed

## Cost Warning
The standard public VPC module creates network resources (VPC, subnets, route tables, IGW) which are free. If `enable_nat_gateway = true` is enabled, AWS charges per hour for NAT Gateways. In this lab, NAT gateway is kept disabled or destroyed immediately.

---

## What is a Terraform Module?

A **module** is a container for multiple resources that are used together.
- **Root Module**: The top-level working directory containing the `*.tf` files where you run `terraform apply`.
- **Child Module**: Any module called inside another module using a `module "name" { source = "..." }` block.
- **Benefits**:
  - Reusability across teams and environments.
  - Standardized infrastructure patterns.
  - Massive code reduction (1 module block creates 10–20+ underlying resources).

### Standard Module File Structure
```text
module-name/
├── README.md        # Documentation, requirements, inputs, outputs
├── main.tf          # Resource declarations
├── variables.tf     # Input variables with descriptions & defaults
├── outputs.tf       # Exported values (IDs, ARNs, DNS names)
└── versions.tf      # Minimum Terraform and provider versions
```

---

## Step 1 — Lab Setup

```bash
source .env
mkdir -p tf-module-vpc && cd tf-module-vpc
```

## Step 2 — Create Provider & Availability Zones Data Source (`provider.tf`)

Create `provider.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-north-1"
}

# Dynamically fetch available AZs in our region
data "aws_availability_zones" "available" {
  state = "available"
}
```

---

## Step 3 — Call the Public AWS VPC Module (`network.tf`)

Create `network.tf`:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "test-vpc-module"
  cidr = "10.0.0.0/16"

  # Use the first 2 availability zones dynamically
  azs             = slice(data.aws_availability_zones.available.names, 0, 2)
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = false
  enable_vpn_gateway = false

  tags = {
    Terraform   = "true"
    Environment = "training"
  }
}
```

---

## Step 4 — Define Outputs (`outputs.tf`)

Create `outputs.tf` to expose module outputs to your terminal:

```hcl
output "vpc_id" {
  description = "The ID of the VPC"
  value       = module.vpc.vpc_id
}

output "public_subnets" {
  description = "List of IDs of public subnets"
  value       = module.vpc.public_subnets
}

output "private_subnets" {
  description = "List of IDs of private subnets"
  value       = module.vpc.private_subnets
}

output "default_security_group_id" {
  description = "Default security group created by the VPC module"
  value       = module.vpc.default_security_group_id
}
```

---

## Step 5 — Initialize & Inspect Downloaded Module

Run:
```bash
terraform init
```

Notice the download step:
```text
Downloading terraform-aws-modules/vpc/aws 5.1.2 for vpc...
- vpc in .terraform/modules/vpc
```

### Inspect the Module Cache
Inspect what Terraform downloaded inside `.terraform/modules/vpc`:
```bash
ls -la .terraform/modules/vpc
```
You will find `main.tf`, `variables.tf`, `outputs.tf`, `subnets.tf`, `routes.tf` — demonstrating that public registry modules are standard Terraform code repositories.

---

## Step 6 — Plan & Apply

```bash
terraform plan
```
> [!NOTE]
> Notice the plan: **12+ resources to add** from just one `module` block (VPC, 2 public subnets, 2 private subnets, Internet Gateway, 3 Route Tables, Route Table Associations, Network ACLs, Default Security Group).

Apply:
```bash
terraform apply -auto-approve
```

---

## Step 7 — Verification in AWS Console

1. Navigate to **VPC Console → Your VPCs**.
2. Select `test-vpc-module`.
3. Open the **Resource map** tab:
   - View the VPC linked to 4 Subnets (2 public, 2 private).
   - Verify Route tables: Public subnets route `0.0.0.0/0` to the Internet Gateway (`igw-*`).
   - Private subnets have local-only routing.

---

## Step 8 — Clean Up

```bash
terraform destroy -auto-approve
```
