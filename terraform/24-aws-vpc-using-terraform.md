# Chapter 24 — AWS VPC Using Terraform

## Objective
Create the same VPC design with one Terraform config: VPC, 2 subnets, internet gateway, route table with a route, and route table association.

## Prerequisites
- Chapter 23 (you know each step manually)
- `.env` in `tf-aws`; region code

## Architecture / Flow
```text
aws_vpc → aws_subnet (private) + aws_subnet (public) → aws_internet_gateway
        → aws_route_table (with route 0.0.0.0/0 → IGW)
        → aws_route_table_association (public subnet ↔ route table)
```

Note: in this chapter the private subnet uses `10.0.1.0/24` and the public subnet uses `10.0.2.0/24` (this is how the video wrote it). This is different from the manual chapter, and it does not matter as long as the two ranges do not overlap.

## Step 1 — Folder and credentials
```bash
source .env
mkdir aws-vpc
cd aws-vpc
```

## Step 2 — Create `main.tf`

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
  region = "<YOUR_REGION>"
}

# 1) VPC
resource "aws_vpc" "my_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "my-vpc"
  }
}

# 2) Private subnet
resource "aws_subnet" "private_subnet" {
  vpc_id     = aws_vpc.my_vpc.id
  cidr_block = "10.0.1.0/24"

  tags = {
    Name = "private-subnet"
  }
}

# 3) Public subnet
resource "aws_subnet" "public_subnet" {
  vpc_id     = aws_vpc.my_vpc.id
  cidr_block = "10.0.2.0/24"

  tags = {
    Name = "public-subnet"
  }
}

# 4) Internet gateway
resource "aws_internet_gateway" "my_igw" {
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "my-igw"
  }
}

# 5) Route table with a route to the internet gateway
resource "aws_route_table" "my_rt" {
  vpc_id = aws_vpc.my_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.my_igw.id
  }

  tags = {
    Name = "my-rt"
  }
}

# 6) Associate the public subnet with the route table
resource "aws_route_table_association" "public_subnet_asso" {
  route_table_id = aws_route_table.my_rt.id
  subnet_id      = aws_subnet.public_subnet.id
}
```

Replace `<YOUR_REGION>` with your region, for example `eu-north-1`.

**Explanation:**
- `aws_vpc.my_vpc.id` → we refer to another resource's ID so Terraform knows the order (VPC first, then subnets).
- `aws_internet_gateway` with `vpc_id` → creates the IGW and attaches it to the VPC (one resource does both).
- The `route { ... }` block inside the route table is one routing rule.
- `aws_route_table_association` links a subnet to the route table. Without it the route table rule applies to no subnet.

## Step 3 — Show the empty state in the console (optional)
VPC → **Your VPCs**. You will see only the default VPC.

## Step 4 — Run
```bash
terraform init
terraform validate
terraform apply
```
Type `yes`.

**Expected:**
```text
Apply complete! Resources: 6 added, 0 changed, 0 destroyed.
```

## Step 5 — Verify
VPC → Your VPCs → open `my-vpc` → **Resource map**.

**Expected:**
- 2 subnets (`public-subnet`, `private-subnet`).
- `public-subnet` → `my-rt` → `my-igw`.
- `private-subnet` → main (default) route table.

## Troubleshooting

### Error 1
```text
Error: Unsupported argument ... "route" ... / Incorrect attribute value type
```
**Reason:** `route` was written as `route = { ... }`.
**Fix:** Use a block, not an equal sign: `route { ... }`. Same for `ingress` and `egress` in Chapter 26.

### Error 2
```text
Error: Reference to undeclared resource
```
**Reason:** The name after the type (for example `my_vpc`) is different in the reference.
**Fix:** Make the names match exactly.

## How to find code yourself
Go to the AWS provider documentation and search `aws_route_table`. The **Example Usage** section shows the `route` block.

## Cleanup
Keep this VPC for Chapter 25. Otherwise run `terraform destroy`.

## Final Result
A full VPC is created in a single command.
