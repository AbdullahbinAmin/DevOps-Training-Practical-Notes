# Chapter 37 — AWS EC2 Using Public Terraform Module

## Objective
Deploy an Amazon EC2 instance using the official public **`terraform-aws-modules/ec2-instance/aws`** module, and interconnect it with the **`terraform-aws-modules/vpc/aws`** module created in Chapter 36.

## Prerequisites
- AWS credentials loaded (`source .env`)
- Region AMI ID (e.g., Ubuntu or Amazon Linux 2023 for your region)
- `tf-module-vpc` lab folder from Chapter 36 (or a new combined setup)

## Cost Warning
EC2 instance + EBS volume will incur standard AWS charges if left running. Destroy after testing.

---

## Step 1 — Project Structure

We will place networking in `network.tf` and compute in `instance.tf` inside the root module:

```text
tf-module-ec2/
├── provider.tf
├── network.tf    # VPC module
├── instance.tf   # EC2 instance module
└── outputs.tf
```

```bash
source .env
mkdir -p tf-module-ec2 && cd tf-module-ec2
```

---

## Step 2 — Configure Provider & Data Source (`provider.tf`)

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
  region = "eu-north-1" # Set to your active region
}

data "aws_availability_zones" "available" {
  state = "available"
}
```

---

## Step 3 — Define the VPC Module (`network.tf`)

Create `network.tf`:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "test-vpc-module"
  cidr = "10.0.0.0/16"

  azs             = slice(data.aws_availability_zones.available.names, 0, 2)
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = false
  enable_vpn_gateway = false

  tags = {
    Environment = "dev"
  }
}
```

---

## Step 4 — Define the EC2 Instance Module (`instance.tf`)

Create `instance.tf`:

```hcl
module "ec2_instance" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "~> 5.0"

  name = "module-project-ec2"

  # Replace with a valid AMI ID for your region
  ami                    = "ami-0705384c0b33c194c"
  instance_type          = "t3.micro"
  monitoring             = false

  # Connect EC2 instance to the VPC module's outputs
  vpc_security_group_ids = [module.vpc.default_security_group_id]
  subnet_id              = module.vpc.public_subnets[0]

  tags = {
    Terraform   = "true"
    Environment = "dev"
  }
}
```

> [!IMPORTANT]
> Notice how module outputs are chained:
> - `module.vpc.public_subnets[0]` passes the first public subnet ID into the EC2 module.
> - `module.vpc.default_security_group_id` attaches the VPC's default security group.

---

## Step 5 — Expose Outputs (`outputs.tf`)

Create `outputs.tf`:

```hcl
output "ec2_instance_id" {
  description = "The ID of the EC2 instance"
  value       = module.ec2_instance.id
}

output "ec2_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = module.ec2_instance.public_ip
}

output "vpc_id" {
  description = "The ID of the created VPC"
  value       = module.vpc.vpc_id
}
```

---

## Step 6 — Initialize & Download Modules

Every time a new module source is introduced, re-run `terraform init`:

```bash
terraform init
```

Terraform downloads both modules:
- `terraform-aws-modules/vpc/aws`
- `terraform-aws-modules/ec2-instance/aws`

---

## Step 7 — Plan & Apply

```bash
terraform plan
```
Plan shows: **13 resources to add** (12 VPC network components + 1 EC2 instance).

Apply:
```bash
terraform apply -auto-approve
```

---

## Step 8 — Verification

1. **Console Check**:
   - Open **AWS Console → EC2 → Instances**.
   - Find instance `module-project-ec2`.
   - Verify **VPC ID** matches `module.vpc.vpc_id`.
   - Verify **Subnet ID** matches the public subnet `10.0.101.0/24`.
2. **Output Check**:
   ```bash
   terraform output
   ```

---

## Step 9 — Clean Up

```bash
terraform destroy -auto-approve
```
