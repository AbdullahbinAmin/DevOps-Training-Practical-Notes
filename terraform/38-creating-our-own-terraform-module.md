# Chapter 38 — Creating Our Own Custom Terraform Module

## Objective
Build a complete, reusable custom **VPC & Subnet Module** from scratch:
1. Accept VPC CIDR and Name with built-in input validation.
2. Accept dynamic subnets via a map of objects with `optional(bool, false)` for public/private designation.
3. Automatically provision an **Internet Gateway (IGW)**, **Route Table**, and **Route Table Associations** conditionally only when public subnets exist.
4. Expose clean outputs (`vpc_id`, `public_subnets`, `private_subnets`).
5. Test the custom module from a root configuration.

## Prerequisites
- AWS credentials loaded (`source .env`)
- Terraform 1.x installed

---

## Directory Architecture

```text
tf-own-module-vpc/
├── main.tf                 # Root module invoking our child module
├── outputs.tf              # Root outputs displaying module results
├── provider.tf             # AWS provider configuration
└── modules/
    └── vpc/                # Our Custom Child Module
        ├── versions.tf     # Terraform & provider constraints
        ├── variables.tf    # Inputs with custom validation rules
        ├── main.tf         # Resources: VPC, Subnets, IGW, Route Tables
        └── outputs.tf      # Child module exported values
```

```bash
source .env
mkdir -p tf-own-module-vpc/modules/vpc
cd tf-own-module-vpc
```

---

## Part 1 — Building the Child Module (`modules/vpc`)

### 1. Module Constraints (`modules/vpc/versions.tf`)

```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}
```

---

### 2. Module Input Variables & Validation (`modules/vpc/variables.tf`)

```hcl
variable "vpc_config" {
  description = "CIDR block and name for the VPC"
  type = object({
    cidr_block = string
    name       = string
  })

  validation {
    condition     = can(cidrnetmask(var.vpc_config.cidr_block))
    error_message = "Invalid VPC CIDR block format. Must be valid IPv4 notation (e.g., 10.0.0.0/16)."
  }
}

variable "subnet_config" {
  description = "Map of subnet configurations"
  type = map(object({
    cidr_block = string
    az         = string
    public     = optional(bool, false)
  }))

  validation {
    condition = alltrue([
      for cfg in var.subnet_config : can(cidrnetmask(cfg.cidr_block))
    ])
    error_message = "All subnet CIDR blocks must be in valid IPv4 CIDR notation (e.g., 10.0.1.0/24)."
  }
}
```

---

### 3. Module Resources & Automation Logic (`modules/vpc/main.tf`)

```hcl
# 1. Create VPC
resource "aws_vpc" "main" {
  cidr_block = var.vpc_config.cidr_block

  tags = {
    Name = var.vpc_config.name
  }
}

# 2. Create Subnets
resource "aws_subnet" "main" {
  for_each          = var.subnet_config
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.az

  tags = {
    Name = each.key
  }
}

# 3. Data Processing Locals
locals {
  # Filter public subnets
  public_subnets = {
    for key, cfg in var.subnet_config : key => cfg if cfg.public
  }
  # Filter private subnets
  private_subnets = {
    for key, cfg in var.subnet_config : key => cfg if !cfg.public
  }
}

# 4. Conditionally create IGW only if at least one public subnet exists
resource "aws_internet_gateway" "main" {
  count  = length(local.public_subnets) > 0 ? 1 : 0
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.vpc_config.name}-igw"
  }
}

# 5. Conditionally create Public Route Table
resource "aws_route_table" "main" {
  count  = length(local.public_subnets) > 0 ? 1 : 0
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main[0].id
  }

  tags = {
    Name = "${var.vpc_config.name}-public-rt"
  }
}

# 6. Associate all public subnets with the Public Route Table
resource "aws_route_table_association" "main" {
  for_each       = local.public_subnets
  subnet_id      = aws_subnet.main[each.key].id
  route_table_id = aws_route_table.main[0].id
}
```

---

### 4. Module Outputs (`modules/vpc/outputs.tf`)

```hcl
output "vpc_id" {
  description = "The ID of the created VPC"
  value       = aws_vpc.main.id
}

output "public_subnets" {
  description = "Map of public subnet IDs and availability zones"
  value = {
    for key, cfg in local.public_subnets : key => {
      subnet_id = aws_subnet.main[key].id
      az        = aws_subnet.main[key].availability_zone
    }
  }
}

output "private_subnets" {
  description = "Map of private subnet IDs and availability zones"
  value = {
    for key, cfg in local.private_subnets : key => {
      subnet_id = aws_subnet.main[key].id
      az        = aws_subnet.main[key].availability_zone
    }
  }
}
```

---

## Part 2 — Consuming the Custom Module from Root

Go to the root directory `tf-own-module-vpc/`.

### 1. Root Provider (`provider.tf`)

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
```

### 2. Root Invocation (`main.tf`)

```hcl
module "my_vpc" {
  source = "./modules/vpc"

  vpc_config = {
    cidr_block = "10.0.0.0/16"
    name       = "my-custom-vpc"
  }

  subnet_config = {
    "public-sub-1" = {
      cidr_block = "10.0.1.0/24"
      az         = "eu-north-1a"
      public     = true
    }
    "public-sub-2" = {
      cidr_block = "10.0.2.0/24"
      az         = "eu-north-1b"
      public     = true
    }
    "private-sub-1" = {
      cidr_block = "10.0.3.0/24"
      az         = "eu-north-1a"
      # public omitted -> defaults to false
    }
  }
}
```

### 3. Root Outputs (`outputs.tf`)

```hcl
output "vpc_id" {
  value = module.my_vpc.vpc_id
}

output "public_subnets" {
  value = module.my_vpc.public_subnets
}

output "private_subnets" {
  value = module.my_vpc.private_subnets
}
```

---

## Part 3 — Testing & Verification

### 1. Test Input Validation (Negative Test)
Temporarily pass an invalid CIDR:
```hcl
vpc_config = {
  cidr_block = "10.0.0.999/16"
  name       = "broken-vpc"
}
```
Run `terraform plan`:
```text
Error: Invalid VPC CIDR block format. Must be valid IPv4 notation (e.g., 10.0.0.0/16).
```
Revert back to `10.0.0.0/16`.

### 2. Initialize and Apply
```bash
terraform init
terraform plan
```
Plan: **8 resources to add** (1 VPC + 3 Subnets + 1 IGW + 1 Route Table + 2 Associations).

Apply:
```bash
terraform apply -auto-approve
```

### 3. Check Outputs
```bash
terraform output
```
Verify the structured maps of public and private subnets with their IDs and AZs.

### 4. AWS Console Verification
- **VPC Console → Resource map**:
  - `public-sub-1` and `public-sub-2` route traffic to Internet Gateway.
  - `private-sub-1` has no internet route.

---

## Part 4 — Clean Up

```bash
terraform destroy -auto-approve
```
