# Day 08 — Create EC2 using Existing VPC

> Launch an EC2 instance into an already existing VPC, subnet, and security group by fetching all three IDs with data sources.

## Learning Objectives
- Combine `aws_ami`, `aws_vpc`, `aws_subnet`, and `aws_security_group` data sources in one config.
- Scope a subnet lookup to a specific VPC so the lookup can never pick the wrong network.
- Pass fetched IDs into `aws_instance` via `subnet_id` and `vpc_security_group_ids`.
- Apply, verify in the console, and destroy only the EC2 (not the borrowed network).

## Prerequisites
- Parts 25 and 26 of Day 08 completed.
- Terraform 1.x, AWS provider 5.x, credentials via `aws configure` / `AWS_PROFILE`.
- Region `eu-north-1`, and these objects created manually in the **same account and same region**:

| Name | Example ID | Type | Tags |
|---|---|---|---|
| My-VPC | `vpc-0a1b2c3d4e5f67890` | VPC | `Name = My-VPC`, `Env = prod` |
| Private-Subnet | `subnet-0a1b2c3d4e5f67891` | Subnet | `Name = Private-Subnet`, `Env = prod` |
| My-Web-Server-SG | `sg-0abc1234def56784e` | Security Group | `Name = My-Web-Server`, `HTTP = enabled` |

## Concept (data source vs resource)
Here both kinds appear in one config, and the split is the whole point:
- **Data sources (read-only):** the AMI, the VPC, the subnet, the security group. These belong to someone else (an admin or another team). Terraform must not own or delete them.
- **Resource (owned):** only `aws_instance`. This is the single thing this config creates and the only thing `terraform destroy` removes.

That separation is what makes the pattern safe. If you had written `resource "aws_vpc"` instead of `data "aws_vpc"`, a `destroy` would tear down the shared network. Data sources make that impossible.

Three things are required to place an instance: a **VPC**, a **subnet** inside it, and a **security group** in that same VPC.

## Step-by-Step Practical

1. Create the folder.

```bash
mkdir -p ~/tf-ec2-existing-vpc && cd ~/tf-ec2-existing-vpc
```

2. `providers.tf`

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
  region = "eu-north-1"
}
```

3. `data.tf` — fetch the existing VPC by tags.

```hcl
data "aws_vpc" "my_vpc" {
  filter {
    name   = "tag:Name"
    values = ["My-VPC"]
  }

  filter {
    name   = "tag:Env"
    values = ["prod"]
  }
}
```

4. Append the subnet lookup to `data.tf`. Note the first filter: the subnet **must** belong to the VPC found above.

```hcl
data "aws_subnet" "my_subnet" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.my_vpc.id]
  }

  filter {
    name   = "tag:Name"
    values = ["Private-Subnet"]
  }

  filter {
    name   = "tag:Env"
    values = ["prod"]
  }
}
```

5. Append the security group lookup, also scoped to the VPC.

```hcl
data "aws_security_group" "my_sg" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.my_vpc.id]
  }

  filter {
    name   = "tag:Name"
    values = ["My-Web-Server"]
  }

  filter {
    name   = "tag:HTTP"
    values = ["enabled"]
  }
}
```

6. Append the AMI lookup so the image ID is never hardcoded.

```hcl
data "aws_ami" "amazon_linux" {
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
}
```

7. `ec2.tf` — the only resource in this config.

```hcl
resource "aws_instance" "my_ec2" {
  ami           = data.aws_ami.amazon_linux.id      # fetched
  instance_type = "t3.micro"

  subnet_id              = data.aws_subnet.my_subnet.id            # fetched
  vpc_security_group_ids = [data.aws_security_group.my_sg.id]      # fetched

  # Private subnet: no public IP. Reach it via SSM or a bastion.
  associate_public_ip_address = false

  # Require IMDSv2 (security best practice)
  metadata_options {
    http_tokens   = "required"
    http_endpoint = "enabled"
  }

  root_block_device {
    volume_size = 8
    volume_type = "gp3"
    encrypted   = true
  }

  tags = {
    Name      = "My-EC2"
    Env       = "prod"
    ManagedBy = "terraform"
  }
}
```

8. `outputs.tf`

```hcl
output "vpc_id" {
  value = data.aws_vpc.my_vpc.id
}

output "subnet_id" {
  value = data.aws_subnet.my_subnet.id
}

output "subnet_az" {
  value = data.aws_subnet.my_subnet.availability_zone
}

output "sg_id" {
  value = data.aws_security_group.my_sg.id
}

output "ami_id" {
  value = data.aws_ami.amazon_linux.id
}

output "instance_id" {
  value = aws_instance.my_ec2.id
}

output "instance_private_ip" {
  value = aws_instance.my_ec2.private_ip
}
```

9. Initialise, review, then apply. Read the plan before typing `yes` — this one **does** create a billable resource.

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

10. Inspect what Terraform read and created.

```bash
terraform output
terraform state list
```

## Expected Output

`terraform plan`:

```bash
Terraform will perform the following actions:

  # aws_instance.my_ec2 will be created
  + resource "aws_instance" "my_ec2" {
      + ami                    = "ami-0abc1234def567890"
      + instance_type          = "t3.micro"
      + subnet_id              = "subnet-0a1b2c3d4e5f67891"
      + vpc_security_group_ids = [ "sg-0abc1234def56784e" ]
      + private_ip             = (known after apply)
      + tags                   = {
          + "Name" = "My-EC2"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

Note `Plan: 1 to add` — the VPC, subnet, SG, and AMI are reads, not creates.

`terraform apply`:

```bash
aws_instance.my_ec2: Creating...
aws_instance.my_ec2: Still creating... [10s elapsed]
aws_instance.my_ec2: Creation complete after 32s [id=i-0123456789abcdef0]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

ami_id              = "ami-0abc1234def567890"
instance_id         = "i-0123456789abcdef0"
instance_private_ip = "10.0.1.25"
sg_id               = "sg-0abc1234def56784e"
subnet_az           = "eu-north-1a"
subnet_id           = "subnet-0a1b2c3d4e5f67891"
vpc_id              = "vpc-0a1b2c3d4e5f67890"
```

## Verification
1. `terraform state list` shows exactly one entry: `aws_instance.my_ec2`. Data sources are not owned resources.
2. AWS Console → EC2 → Instances → `My-EC2` → state `running`.
3. On the Networking tab, VPC ID, Subnet ID, and Availability Zone match the outputs.
4. On the Security tab, the attached SG ID equals `sg_id` — no new SG was created.
5. `instance_private_ip` falls inside the subnet's CIDR range.
6. Console → EC2 → Security Groups: still only the original SG, count unchanged.
7. CLI cross-check:

```bash
aws ec2 describe-instances --region eu-north-1 \
  --filters "Name=tag:Name,Values=My-EC2" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{Id:InstanceId,Subnet:SubnetId,Vpc:VpcId,SG:SecurityGroups[].GroupId,AZ:Placement.AvailabilityZone}'
```

## Cleanup

```bash
# Review first - confirm ONLY the instance is listed for destruction
terraform plan -destroy

terraform destroy
```

Expected: `Plan: 0 to add, 0 to change, 1 to destroy.` The VPC, subnet, and security group survive because they were only read. Delete them manually in the console if you no longer need them.

```bash
rm -rf .terraform .terraform.lock.hcl terraform.tfstate*
```

## Common Errors & Fixes
| Error | Cause | Fix |
|---|---|---|
| `InvalidParameterValue: Security group sg-... and subnet subnet-... belong to different networks` | SG and subnet are in different VPCs | Add the `vpc-id` filter to both data sources (steps 4 and 5) |
| `no matching EC2 Subnet found` | Tag mismatch, or subnet is in another VPC | Verify tags and that the `vpc-id` filter resolves correctly |
| `multiple EC2 Subnets matched` | Several subnets share the same tags | Add a `filter { name = "availability-zone" ... }` or use plural `aws_subnets` and index the list |
| `Unsupported argument: security_groups` in a VPC | `security_groups` is EC2-Classic style | Use `vpc_security_group_ids = [...]` |
| `InvalidAMIID.NotFound` | AMI belongs to a different region | Data sources are region-scoped; keep provider region consistent |
| `Unsupported instance type` in eu-north-1 | `t2.micro` is unavailable in some regions/AZs | Use `t3.micro` |
| Instance created but unreachable via SSH | Launched in a private subnet with no public IP | Use SSM Session Manager, a bastion host, or a public subnet |
| `terraform destroy` proposes deleting the VPC | You used `resource "aws_vpc"` instead of `data` | Change to a `data` block and `terraform state rm` the wrongly-managed object |
| `VcpuLimitExceeded` | Account instance quota reached | Terminate unused instances or request a quota increase |

## Key Takeaways
- Three lookups (VPC, subnet, SG) plus one AMI lookup are enough to place an EC2 in someone else's network.
- Always scope subnet and SG lookups with `vpc-id` chained from the VPC data source — this prevents cross-VPC mismatch errors.
- `vpc_security_group_ids` (a list) is the correct argument inside a VPC, not `security_groups`.
- `Plan: 1 to add` and a one-line `terraform state list` are your proof that only the instance is Terraform-owned.
- The same account, same region, and consistent tags are hard requirements for tag-based data sources.
- `terraform destroy` removes only what Terraform created — borrowed infrastructure is untouched.

## Next: [Day 09 — Terraform Modules](../Day-09-Modules/README.md)
