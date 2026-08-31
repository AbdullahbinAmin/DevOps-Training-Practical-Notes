# Day 08 — VPC & SG with Data Source

> Use tag-based data sources to look up an existing Security Group, VPC, the region's Availability Zones, the caller identity, and the current region.

## Learning Objectives
- Look up an existing security group by tags with `data "aws_security_group"`.
- Look up an existing VPC by tags with `data "aws_vpc"` and read its `cidr_block`.
- List all subnets of that VPC with `data "aws_subnets"`.
- List available AZs with `data "aws_availability_zones"`.
- Identify who Terraform is authenticating as with `data "aws_caller_identity"` and where with `data "aws_region"`.
- Understand that all data sources are account- and region-scoped.

## Prerequisites
- Day 08 part 25 completed (you can write and read a data source).
- Terraform 1.x, AWS provider 5.x, credentials via `aws configure` / `AWS_PROFILE`.
- Region: `eu-north-1` (Stockholm).
- Already existing in the console (create manually if needed):
  - Security group `my-web-server-group`: inbound HTTP 80 allowed, outbound all traffic, tags `Name = my-web-server`, `Value = HTTP`.
  - VPC `my-vpc`, CIDR `10.0.0.0/16`, tags `Name = my-vpc`, `Environment = prod`.

## Concept (data source vs resource)
A `resource "aws_security_group"` would **create a new** SG every time you run it. A `data "aws_security_group"` **finds the one that already exists** and gives you its `id` — nothing is created, nothing is billed, nothing is destroyed.

This is the "one SG, many EC2s" pattern from the slide: the SG `common-rules` (HTTP 80, SSH 22) is created once by an admin, then EC2-1 … EC2-5 all reference `data.aws_security_group.my_web_sg.id`. No duplicate SGs.

Two ways to find things:
- **By tags** — `filter { name = "tag:Name" ... }`. Portable and readable; requires disciplined tagging.
- **By ID** — `id = "vpc-0abc..."`. Exact, but hardcoded again.

Tag-based lookup is preferred, which is why "use proper tags" is a rule, not a suggestion.

## Step-by-Step Practical

1. Create the folder and provider config.

```bash
mkdir -p ~/tf-vpc-sg-datasource && cd ~/tf-vpc-sg-datasource
```

```hcl
# providers.tf
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
  region = "eu-north-1" # Stockholm - data sources are region specific
}
```

2. Confirm the SG and VPC exist and are tagged, before writing Terraform.

```bash
aws ec2 describe-security-groups --region eu-north-1 \
  --filters "Name=tag:Name,Values=my-web-server" "Name=tag:Value,Values=HTTP" \
  --query 'SecurityGroups[].{Id:GroupId,Name:GroupName,VpcId:VpcId}'

aws ec2 describe-vpcs --region eu-north-1 \
  --filters "Name=tag:Name,Values=my-vpc" "Name=tag:Environment,Values=prod" \
  --query 'Vpcs[].{Id:VpcId,Cidr:CidrBlock}'
```

3. Security group data source — `data-sg.tf`.

```hcl
data "aws_security_group" "my_web_sg" {
  filter {
    name   = "tag:Name"
    values = ["my-web-server"]
  }

  filter {
    name   = "tag:Value"
    values = ["HTTP"]
  }
}
```

Meaning: find the SG in the current account and region where `tag:Name = my-web-server` **and** `tag:Value = HTTP`. This data source must match exactly one SG.

4. VPC data source — `data-vpc.tf`.

```hcl
data "aws_vpc" "my_vpc" {
  filter {
    name   = "tag:Environment"
    values = ["prod"]
  }

  filter {
    name   = "tag:Name"
    values = ["my-vpc"]
  }
}

# All subnets that belong to that VPC
data "aws_subnets" "my_vpc_subnets" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.my_vpc.id]
  }
}
```

5. Availability zones, caller identity, and region — `data-account.tf`.

```hcl
data "aws_availability_zones" "azs" {
  state = "available"
}

data "aws_caller_identity" "me" {}

data "aws_region" "current" {}
```

6. Outputs — `outputs.tf`.

```hcl
# ---- Security Group ----
output "sg_id" {
  value = data.aws_security_group.my_web_sg.id
}

output "sg_name" {
  value = data.aws_security_group.my_web_sg.name
}

output "sg_arn" {
  value = data.aws_security_group.my_web_sg.arn
}

output "sg_description" {
  value = data.aws_security_group.my_web_sg.description
}

output "sg_vpc_id" {
  value = data.aws_security_group.my_web_sg.vpc_id
}

# ---- VPC ----
output "vpc_id" {
  value = data.aws_vpc.my_vpc.id
}

output "vpc_cidr_block" {
  value = data.aws_vpc.my_vpc.cidr_block
}

output "vpc_subnet_ids" {
  value = data.aws_subnets.my_vpc_subnets.ids
}

# ---- Region / AZs / Identity ----
output "az_names" {
  value = data.aws_availability_zones.azs.names
}

output "account_id" {
  value = data.aws_caller_identity.me.account_id
}

output "user_arn" {
  value = data.aws_caller_identity.me.arn
}

output "region_name" {
  value = data.aws_region.current.name
}
```

7. Run it. Nothing is created, so `plan` is enough.

```bash
terraform init
terraform fmt
terraform validate
terraform plan
```

8. Optional — since this folder has only data sources and outputs, an apply creates zero resources and lets you query outputs individually.

```bash
terraform apply -auto-approve
terraform output sg_id
terraform output -json vpc_subnet_ids
```

## Expected Output

```bash
Changes to Outputs:
  + account_id      = "123456789012"
  + az_names        = [
      + "eu-north-1a",
      + "eu-north-1b",
      + "eu-north-1c",
    ]
  + region_name     = "eu-north-1"
  + sg_arn          = "arn:aws:ec2:eu-north-1:123456789012:security-group/sg-0a1b2c3d4e5f6784e"
  + sg_description  = "Allow HTTP access"
  + sg_id           = "sg-0a1b2c3d4e5f6784e"
  + sg_name         = "my-web-server-group"
  + sg_vpc_id       = "vpc-0abc1234def567890"
  + user_arn        = "arn:aws:iam::123456789012:user/terraform-user"
  + vpc_cidr_block  = "10.0.0.0/16"
  + vpc_id          = "vpc-0abc1234def567890"
  + vpc_subnet_ids  = [
      + "subnet-0a1b2c3d4e5f67891",
    ]

You can apply this plan to save these new output values to the Terraform
state, without changing any real infrastructure.
```

## Verification
1. `sg_id` ends in `784e` in the plan — open AWS Console → EC2 → Security Groups and confirm the same suffix.
2. Console SG name shows `my-web-server-group`, inbound rule HTTP/TCP/80, outbound all traffic.
3. VPC → Your VPCs: `my-vpc` has ID matching `vpc_id` and IPv4 CIDR `10.0.0.0/16`.
4. `sg_vpc_id` should equal `vpc_id` — proof the SG lives in that VPC.
5. `account_id` matches the account number in the console top-right menu.
6. `region_name` matches the console region selector (Stockholm).
7. CLI cross-check of AZs and identity:

```bash
aws sts get-caller-identity
aws ec2 describe-availability-zones --region eu-north-1 \
  --query 'AvailabilityZones[?State==`available`].ZoneName'
```

## Cleanup

```bash
# No AWS resources were created, so destroy only clears data-source state
terraform destroy -auto-approve
rm -rf .terraform .terraform.lock.hcl terraform.tfstate*
```

Do **not** delete the existing VPC or security group — the next lab reuses them.

## Common Errors & Fixes
| Error | Cause | Fix |
|---|---|---|
| `multiple Security Groups matched; use additional constraints` | Tag filters not unique enough | Add another `filter` (e.g. `tag:Value`) or filter on `vpc-id` |
| `no matching SecurityGroup found` | Wrong tag key/value, or wrong region | Tag keys are case-sensitive; check `provider "aws"` region |
| `no matching EC2 VPC found` | VPC missing the `Environment = prod` tag | Add the tag in the console, or drop that filter |
| `multiple EC2 VPCs matched` | Several VPCs share the same tags | Add a `cidr-block` filter or pass `id` directly |
| `Invalid filter: tag:name` gives no results | Used `tag:name` instead of `tag:Name` | Match the tag key exactly as created |
| `data "aws_subnet"` errors with multiple matches | Singular `aws_subnet` must match one subnet | Use plural `aws_subnets` for a list of IDs |
| Empty `vpc_subnet_ids` list | VPC has no subnets yet | Create a subnet, or verify the `vpc-id` filter value |
| `UnauthorizedOperation` | IAM user lacks describe permissions | Attach `AmazonEC2ReadOnlyAccess` |

## Key Takeaways
- Tag-based filters (`tag:Name`, `tag:Environment`) are the standard way to find existing AWS objects.
- A singular data source (`aws_vpc`, `aws_security_group`, `aws_subnet`) must resolve to exactly one object; plural (`aws_subnets`) returns a list.
- Chain data sources: `data.aws_vpc.my_vpc.id` feeds the `vpc-id` filter of `aws_subnets`.
- `aws_availability_zones`, `aws_caller_identity`, and `aws_region` need no arguments and are useful for guardrails and dynamic AZ placement.
- All data sources are account + region specific — change the region and results change or disappear.
- This whole lab is read-only: zero cost, zero risk.

## Next: [Create EC2 using Existing VPC](./27-create-ec2-using-existing-vpc.md)
