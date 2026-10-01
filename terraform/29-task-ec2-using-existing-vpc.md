# Chapter 29 — Task: Create EC2 Using Existing VPC, Security Group and Subnet

## Objective
Create an EC2 instance **using existing** VPC, subnet and security group, found through data sources. Learn to troubleshoot a common error.

## Prerequisites — three things must already exist (GUI or earlier chapters)
1. A **VPC** named `my-vpc` with tags `Env = prod` and `Name = my-vpc`.
2. A **subnet** in that VPC, for example a private subnet with tags `Env = prod` and `Name = private-subnet` (CIDR for example `10.0.1.0/24`).
3. A **security group** (created in step "Fix the error" below).

You can create the VPC and subnet by hand (Chapter 23) or with Terraform (Chapter 24), then add the tags in the console: select the resource → **Tags** tab → **Manage tags**.

Where to see tags: VPC → your VPC → **Tags** tab. Subnets → your subnet → **Tags** tab.

## Cost warning
This task creates an EC2 instance. Destroy it at the end.

## Step 1 — Folder
Continue in `tf-data-sources` or create a new folder `task-existing-vpc` (with `provider` block and `source ../.env`).

## Step 2 — Write `main.tf`

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

# Existing VPC
data "aws_vpc" "my_vpc" {
  tags = {
    Env  = "prod"
    Name = "my-vpc"
  }
}

# Existing subnet (inside that VPC)
data "aws_subnet" "private_subnet" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.my_vpc.id]
  }

  tags = {
    Name = "private-subnet"
  }
}

# Existing security group (from the SAME VPC)
data "aws_security_group" "my_sg" {
  vpc_id = data.aws_vpc.my_vpc.id

  tags = {
    Name = "my-sg"
    Env  = "prod"
  }
}

# Latest Amazon Linux AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}

resource "aws_instance" "my_server" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "<YOUR_INSTANCE_TYPE>"
  subnet_id              = data.aws_subnet.private_subnet.id
  vpc_security_group_ids = [data.aws_security_group.my_sg.id]

  tags = {
    Name = "instance-in-existing-vpc"
  }
}
```
- `<YOUR_REGION>` → region where the VPC exists.
- `<YOUR_INSTANCE_TYPE>` → for example `t3.micro`.

**Explain:**
- Subnet is found by **two** things: which VPC it belongs to (`vpc-id` filter) and its `Name` tag.
- The security group must belong to the **same VPC** as the subnet. That is why we added `vpc_id` in its data block.
- For instances inside a **non-default VPC** use `vpc_security_group_ids` (IDs). The argument `security_groups` is meant for names in the default VPC and can fail with IDs.
- The `filter` block inside a subnet data source is how you use AWS API filters (the name `vpc-id` is an EC2 API filter name).

## Step 3 — Create the security group in the correct VPC (GUI)
If you use an old security group from a **different VPC**, `terraform apply` fails with an error such as:

```text
Error: ... Security group sg-xxxx and subnet subnet-xxxx belong to different networks.
```

**Why:** A security group belongs to one VPC. A subnet belongs to one VPC. Both must be in the same VPC.

**Fix (GUI):**
1. VPC → **Security groups** → **Create security group**.
2. **Name:** `my-sg`. **Description:** `Allow HTTP`. **VPC:** select `my-vpc` (this is the important part).
3. **Inbound rules:** add **HTTP**, Source `Anywhere-IPv4` (or as required).
4. **Tags:** `Name = my-sg`, `Env = prod`.
5. **Create security group**.
6. Open it and confirm the **VPC ID** is the one of `my-vpc`.

## Step 4 — Run
```bash
source ../.env
terraform init
terraform validate
terraform plan
terraform apply
```
Type `yes`.

**Expected:** `Apply complete! Resources: 1 added`.

## Step 5 — Verify in the console
EC2 → Instances → open `instance-in-existing-vpc`:
- **VPC ID** = `my-vpc`.
- **Subnet ID** = the private subnet.
- **Security** tab → your `my-sg` group with its inbound rule.
- **Tags**: `Name = instance-in-existing-vpc`.

## Troubleshooting

### Error 1
```text
security group and subnet belong to different networks
```
**Fix:** Recreate the security group inside `my-vpc` (Step 3) and update the `tags` in the data block if needed.

### Error 2
```text
no matching EC2 Subnet found
```
**Reason:** Tag value typo, or wrong region.
**Fix:** Open the subnet → **Tags** tab, copy the exact `Name`.

### Error 3
```text
InvalidParameterValue: ... security group ... default VPC
```
**Reason:** `security_groups` was used with an ID for a non-default VPC.
**Fix:** Use `vpc_security_group_ids = [ ... ]`.

## Cleanup
```bash
terraform destroy
```
Type `yes`. This removes only the instance (VPC, subnet and SG were not created by this config).

## Final Result
An instance runs inside the existing VPC and subnet using the existing security group — found only through data sources.
