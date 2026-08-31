# Day 09 — Multiple Resources with `count`

> Build one VPC, 2 subnets and 4 EC2 instances from three resource blocks using `count` and `count.index` — and learn the index-shifting trap before it bites you.

## Learning Objectives
- Use `count` to create N copies of a resource block.
- Use `count.index` (0-based) in CIDRs, AZs and Name tags.
- Distribute instances across subnets with `element()`, `length()` and `%`.
- Reference all instances with the splat operator `[*]` and one instance with `[i]`.
- Understand and avoid the index-shifting pitfall.

## Prerequisites
- Terraform 1.x.
- AWS credentials configured **outside** the code (`aws configure`, an SSO profile, or an IAM role). Never put keys in `.tf` files.
- An AWS account and region — this lab uses `eu-north-1`.
- Cost warning: 4 `t3.micro` EC2 instances are billable unless you are inside the free tier. Destroy immediately after verifying.

## Concept (plain English)
`count = N` tells Terraform to run one resource block N times. Each copy gets an index starting at 0, available as `count.index`, and is tracked in state as `aws_subnet.main[0]`, `aws_subnet.main[1]`, and so on.

| Run | `count.index` | `cidr_block` | Name tag |
|-----|---------------|--------------|----------|
| 1st | 0 | 10.0.0.0/24 | project-01-subnet-0 |
| 2nd | 1 | 10.0.1.0/24 | project-01-subnet-1 |

Three helpers make round-robin placement work:
- `element(list, index)` — item at that index, wrapping if the index is too big.
- `length(list)` — number of items.
- `%` (modulo) — cycles the index: 0, 1, 0, 1, …

The pitfall: because `count` addresses resources by **position**, deleting a middle item shifts everything after it. Remove the second of three subnets and Terraform does not delete one subnet — it destroys and recreates index 1 and index 2 so the list closes up. That is fine for interchangeable things (a pool of identical workers) and dangerous for anything with identity or data.

## Step-by-Step Practical

1. Create the lab folder. Everything stays in `main.tf` for simplicity.

```bash
mkdir -p ~/tf-count-demo && cd ~/tf-count-demo
```

2. Confirm your credentials work and are coming from the environment, not from code.

```bash
aws sts get-caller-identity
```

3. `main.tf` — providers block.

```hcl
terraform {
  required_version = ">= 1.0"
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

4. Append the VPC. Tag pattern: `<project-name>-<resource>-<index>`.

```hcl
resource "aws_vpc" "my_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "project-01-vpc"
  }
}
```

5. Append 2 subnets using `count`.

```hcl
resource "aws_subnet" "main" {
  count = 2

  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = "eu-north-1${element(["a", "b", "c"], count.index)}"

  tags = {
    Name = "project-01-subnet-${count.index}"
  }
}
```

The slide writes the AZ as `"eu-north-1${count.index}"`, which would produce the invalid AZ `eu-north-10`. Mapping the index to a letter via `element()` is the correct form.

6. Append a data source for the latest Amazon Linux 2 AMI, so no AMI ID is hardcoded per region.

```hcl
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

7. Append 4 EC2 instances, 2 in each subnet via round-robin.

```hcl
resource "aws_instance" "main" {
  count = 4

  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t3.micro"

  # Select subnet dynamically (round-robin)
  subnet_id = element(
    aws_subnet.main[*].id,
    count.index % length(aws_subnet.main[*].id)
  )

  tags = {
    Name = "project-01-instance-${count.index}"
  }
}
```

Placement works out as:

| Instance | `count.index` | `index % 2` | Placed in |
|----------|---------------|-------------|-----------|
| instance-0 | 0 | 0 | Subnet-0 |
| instance-1 | 1 | 1 | Subnet-1 |
| instance-2 | 2 | 0 | Subnet-0 |
| instance-3 | 3 | 1 | Subnet-1 |

8. Append the outputs — `[*]` collects an attribute from every copy.

```hcl
output "vpc_id" {
  value = aws_vpc.my_vpc.id
}

output "subnet_ids" {
  description = "List of all subnet IDs"
  value       = aws_subnet.main[*].id
}

output "subnet_cidrs" {
  value = aws_subnet.main[*].cidr_block
}

output "instance_ids" {
  description = "List of all instance IDs"
  value       = aws_instance.main[*].id
}

output "first_instance_private_ip" {
  description = "A single element, by index"
  value       = aws_instance.main[0].private_ip
}

output "instance_to_subnet" {
  description = "Which instance landed in which subnet"
  value       = { for i, inst in aws_instance.main : inst.tags.Name => inst.subnet_id }
}
```

9. Run the workflow.

```bash
terraform init
terraform validate
terraform plan
terraform apply
```

10. Make `count` dynamic instead of hardcoded — the real reason to use it. Create `variables.tf`.

```hcl
variable "subnet_count" {
  description = "How many subnets"
  type        = number
  default     = 2

  validation {
    condition     = var.subnet_count >= 1 && var.subnet_count <= 3
    error_message = "subnet_count must be between 1 and 3 (only 3 AZs mapped)."
  }
}

variable "instance_count" {
  description = "How many EC2 instances"
  type        = number
  default     = 4
}
```

Then swap the hardcoded numbers:

```hcl
# in aws_subnet.main
count = var.subnet_count

# in aws_instance.main
count = var.instance_count
```

11. Scale up without touching the code.

```bash
terraform plan -var "instance_count=6"
```

12. Demonstrate the index-shifting pitfall, without applying it.

```bash
terraform state list
terraform plan -var "instance_count=3"
```

Terraform destroys `aws_instance.main[3]` only — shrinking from the tail is clean. Now imagine instead that you wanted to remove instance **1** of 4. There is no way to express that with `count`; the closest you get is reordering the underlying list, which forces destroy/recreate of every index after the removal. That is the pitfall.

13. `count` also works as a conditional on/off switch.

```hcl
variable "create_bastion" {
  type    = bool
  default = false
}

resource "aws_instance" "bastion" {
  count = var.create_bastion ? 1 : 0

  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.main[0].id

  tags = {
    Name = "project-01-bastion"
  }
}
```

Reference it as `aws_instance.bastion[0].id`, or safely as `one(aws_instance.bastion[*].id)`.

## Expected Output

`terraform plan` summary:

```
Plan: 7 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + instance_ids = (known after apply)
  + subnet_cidrs = [ "10.0.0.0/24", "10.0.1.0/24" ]
  + subnet_ids   = (known after apply)
  + vpc_id       = (known after apply)
```

`terraform apply` tail:

```
Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

Outputs:

instance_ids = [
  "i-0aa11bb22cc33dd44",
  "i-0bb22cc33dd44ee55",
  "i-0cc33dd44ee55ff66",
  "i-0dd44ee55ff66aa77",
]
instance_to_subnet = {
  "project-01-instance-0" = "subnet-0abc111"
  "project-01-instance-1" = "subnet-0def222"
  "project-01-instance-2" = "subnet-0abc111"
  "project-01-instance-3" = "subnet-0def222"
}
subnet_cidrs = [ "10.0.0.0/24", "10.0.1.0/24" ]
subnet_ids   = [ "subnet-0abc111", "subnet-0def222" ]
vpc_id       = "vpc-09f8e7d6c5b4a3210"
```

`terraform state list`:

```
data.aws_ami.amazon_linux_2
aws_instance.main[0]
aws_instance.main[1]
aws_instance.main[2]
aws_instance.main[3]
aws_subnet.main[0]
aws_subnet.main[1]
aws_vpc.my_vpc
```

## Verification

```bash
terraform state list
terraform output subnet_ids
terraform output instance_to_subnet

# Cross-check against the API
aws ec2 describe-subnets --region eu-north-1 \
  --filters "Name=tag:Name,Values=project-01-subnet-*" \
  --query "Subnets[].{Name:Tags[?Key=='Name']|[0].Value,Cidr:CidrBlock,AZ:AvailabilityZone}" \
  --output table

aws ec2 describe-instances --region eu-north-1 \
  --filters "Name=tag:Name,Values=project-01-instance-*" "Name=instance-state-name,Values=running,pending" \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,Subnet:SubnetId}" \
  --output table
```

Checks to confirm:
- VPC `project-01-vpc` exists with CIDR `10.0.0.0/16`.
- 2 subnets, indexes 0 and 1, CIDRs `10.0.0.0/24` and `10.0.1.0/24`, in different AZs.
- 4 instances named `project-01-instance-0` through `-3`.
- 2 instances in Subnet-0, 2 in Subnet-1 — the modulo round-robin worked.
- `terraform plan` after apply reports `No changes.` (config matches reality).

## Cleanup

Do this as soon as verification passes — running instances cost money.

```bash
cd ~/tf-count-demo
terraform destroy
# review the plan, then confirm with: yes

# confirm nothing is left
terraform state list
aws ec2 describe-instances --region eu-north-1 \
  --filters "Name=tag:Name,Values=project-01-instance-*" "Name=instance-state-name,Values=running,pending" \
  --query "Reservations[].Instances[].InstanceId" --output text

cd ~ && rm -rf ~/tf-count-demo
```

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Invalid availability zone: eu-north-10` | Concatenated the index onto the region: `"eu-north-1${count.index}"` | Map index to a letter: `element(["a","b","c"], count.index)`, or use `data.aws_availability_zones` |
| `Because aws_subnet.main has count set, its attributes must be accessed on specific instances` | Wrote `aws_subnet.main.id` | Use `aws_subnet.main[0].id` or `aws_subnet.main[*].id` |
| `The "count" object can only be used in ... resource blocks` | Used `count.index` outside a block that declares `count` | Only valid inside the counted block |
| `Invalid CIDR block: 10.0.256.0/24` | `count` above 256 with `"10.0.${count.index}.0/24"` | Use `cidrsubnet(var.base_cidr, 8, count.index)` |
| `InvalidSubnet.Conflict: CIDR overlaps` | Two subnets computed the same CIDR | Ensure the index actually varies the CIDR |
| Plan destroys/recreates unrelated resources | Index shifting after removing a middle element | Switch to `for_each`, or `terraform state mv` the indexes |
| `Error: Invalid count argument` — value not known until apply | `count` depends on an unknown attribute of another resource | Base `count` on variables/locals, or apply in two stages with `-target` |
| `VpcLimitExceeded` / `InstanceLimitExceeded` | Account quota reached | Destroy old labs, or request a quota increase |
| `UnauthorizedOperation` | IAM identity lacks EC2/VPC permissions | Attach the right policy; check `aws sts get-caller-identity` |
| `No valid credential sources found` | Credentials not configured | Run `aws configure` or export `AWS_PROFILE` — never hardcode keys |
| `count` and `for_each` both set | Only one meta-argument is allowed | Pick one per resource block |

## Key Takeaways
- `count = N` creates N copies; `count.index` is 0-based; state keys are `[0]`, `[1]`, …
- `element(list, i % length(list))` is the round-robin idiom for spreading instances across subnets.
- `[*]` (splat) gathers an attribute from every copy; `[i]` picks one.
- `count = condition ? 1 : 0` is the standard conditional-resource pattern.
- `count` addresses by position, so removing a middle element shifts every later index and forces destroy/recreate.
- Use `count` for genuinely identical, interchangeable resources, and for on/off toggles. Use `for_each` for anything with distinct identity — that is next.
- Always `terraform destroy` after a billable lab.

## Next: [Multiple Resources with for_each](32-multiple-resources-for-each.md)
