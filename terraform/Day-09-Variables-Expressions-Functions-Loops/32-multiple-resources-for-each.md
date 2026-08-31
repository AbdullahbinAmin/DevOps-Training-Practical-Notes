# Day 09 — Multiple Resources with `for_each`

> Create 2 EC2 instances from different AMIs in different subnets using `for_each` over a map of objects — named keys, no index confusion, no cascading recreation.

## Learning Objectives
- Use `for_each` with a map of objects and with a set of strings.
- Use `each.key` and `each.value` inside a resource.
- Address resources by key in state: `aws_instance.main["ubuntu"]`.
- Convert a list to a set with `toset()` and a list of objects to a map with a `for` expression.
- Explain precisely why `for_each` avoids the `count` index-shifting problem, and when `count` is still the right choice.

## Prerequisites
- Terraform 1.x.
- File 31 (`count`) completed — this lab reuses the same VPC and subnets.
- AWS credentials supplied via `aws configure`, SSO or an IAM role. No keys in code.
- Cost warning: 2 `t3.micro` instances are billable outside the free tier. Destroy after verifying.

## Concept (plain English)
`for_each` runs a resource block once per element of a **map** or a **set of strings**. Each instance is keyed by a meaningful name instead of a number:

```
aws_instance.main["ubuntu"]
aws_instance.main["amazon2"]
```

Inside the block you get two values:
- `each.key` — the map key (e.g. `"ubuntu"`) or the set element.
- `each.value` — the map value (here an object with `ami` and `instance_type`). For a set, `each.value` equals `each.key`.

The whole difference from `count`:

| | `count` | `for_each` |
|---|---|---|
| Input | a number | a map or a set of strings |
| Identity | position: `[0]`, `[1]` | key: `["ubuntu"]` |
| Iterator | `count.index` | `each.key`, `each.value` |
| Remove a middle item | every later index shifts → destroy/recreate | only that key is destroyed |
| Reference all | `res.name[*].id` | `values(res.name)[*].id` or `{for k, v in res.name : k => v.id}` |
| Best for | identical, interchangeable copies; on/off toggles | resources with distinct identity or per-item settings |

This is the index-shifting pitfall solved: keys are stable, so adding or removing one entry touches only that entry.

## Step-by-Step Practical

1. Create the lab folder.

```bash
mkdir -p ~/tf-foreach-demo && cd ~/tf-foreach-demo
aws sts get-caller-identity
```

2. `versions.tf`.

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
  region = var.region
}
```

3. `variables.tf` — a map of objects; the key names each instance.

```hcl
variable "region" {
  type    = string
  default = "eu-north-1"
}

variable "ec2_map" {
  description = "key = instance name, value = object of instance settings"
  type = map(object({
    ami           = string
    instance_type = string
  }))

  validation {
    condition     = length(var.ec2_map) > 0
    error_message = "ec2_map must contain at least one entry."
  }
}

variable "subnet_count" {
  type    = number
  default = 2
}

variable "extra_sg_names" {
  description = "Demo of for_each over a set of strings"
  type        = set(string)
  default     = ["web", "app"]
}
```

4. `terraform.tfvars` — two objects means two EC2 instances. AMI IDs are region-specific; the ones below are placeholders for `eu-north-1`, so replace them with real IDs for your region (step 5 shows how to look them up).

```hcl
ec2_map = {
  ubuntu = {
    ami           = "ami-0e55b159cbfafe1f0" # Ubuntu 22.04, eu-north-1
    instance_type = "t3.micro"
  }
  amazon2 = {
    ami           = "ami-0c02fb55956c7d316" # Amazon Linux 2, eu-north-1
    instance_type = "t3.micro"
  }
}
```

5. Look up real AMI IDs for your region rather than trusting hardcoded ones.

```bash
aws ssm get-parameters --region eu-north-1 \
  --names /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id \
          /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 \
  --query "Parameters[].{Name:Name,AMI:Value}" --output table
```

Paste the returned IDs into `terraform.tfvars`.

6. `main.tf` — VPC, then the subnets with `count` (already familiar), then the instances with `for_each`.

```hcl
data "aws_availability_zones" "az" {
  state = "available"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "project-01-vpc"
  }
}

# 2 Subnets (count example — identical, interchangeable)
resource "aws_subnet" "main" {
  count = var.subnet_count

  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = data.aws_availability_zones.az.names[count.index]

  tags = {
    Name = "project-01-subnet-${count.index}"
  }
}
```

7. Append the EC2 instances using `for_each` over the map.

```hcl
resource "aws_instance" "main" {
  for_each = var.ec2_map

  ami           = each.value.ami
  instance_type = each.value.instance_type

  # Place in a subnet based on the key's position in the map (round-robin)
  subnet_id = aws_subnet.main[
    index(keys(var.ec2_map), each.key) % length(aws_subnet.main)
  ].id

  tags = {
    Name = "project-01-instance-${each.key}"
  }
}
```

Iteration flow:

| Run | `each.key` | `each.value` | key index | `% 2` | Subnet used |
|-----|-----------|--------------|-----------|-------|-------------|
| 1 | amazon2 | `{ami, type}` | 0 | 0 | Subnet-0 |
| 2 | ubuntu | `{ami, type}` | 1 | 1 | Subnet-1 |

`keys()` returns map keys **alphabetically**, so `amazon2` sorts before `ubuntu`. Rely on the key, not on the order you typed the entries.

8. Append a `for_each` over a set of strings, where `each.key == each.value`.

```hcl
resource "aws_security_group" "extra" {
  for_each = var.extra_sg_names

  name        = "project-01-sg-${each.key}"
  description = "Security group for ${each.value} tier"
  vpc_id      = aws_vpc.main.id

  # No ingress rules on purpose: nothing is reachable from the internet.
  # Add narrowly scoped ingress only when a lab actually needs it.
  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "project-01-sg-${each.key}"
  }
}
```

These instances have no inbound rules and no public IP association configured, so nothing is exposed. If you later add SSH, restrict `cidr_blocks` to your own IP — never `0.0.0.0/0`.

9. Convert a list into something `for_each` accepts. `for_each` rejects a plain list.

```hcl
locals {
  # list -> set of strings
  tier_names = toset(["web", "app", "db"])

  # list of objects -> map, keyed by a unique field
  person_list = [
    { fname = "Raju", lname = "Rastogi" },
    { fname = "Shyam", lname = "Paul" },
  ]
  people_map = { for p in local.person_list : p.fname => p }
}
```

10. `outputs.tf`.

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_ids" {
  description = "count resources use splat"
  value       = aws_subnet.main[*].id
}

output "instance_ids" {
  description = "for_each resources produce a map keyed by name"
  value       = { for k, v in aws_instance.main : k => v.id }
}

output "instance_private_ips" {
  value = { for k, v in aws_instance.main : k => v.private_ip }
}

output "instance_placement" {
  value = { for k, v in aws_instance.main : k => v.subnet_id }
}

output "one_instance_id" {
  description = "Address a single for_each instance by key"
  value       = aws_instance.main["ubuntu"].id
}

output "all_instance_ids_as_list" {
  value = values(aws_instance.main)[*].id
}

output "sg_ids" {
  value = { for k, v in aws_security_group.extra : k => v.id }
}
```

11. Run the workflow.

```bash
terraform init
terraform validate
terraform fmt
terraform plan
terraform apply
```

12. Inspect the state addresses — this is where `for_each` pays off.

```bash
terraform state list
```

13. Prove that `for_each` does not shift. Add a third entry to `terraform.tfvars`.

```hcl
  debian = {
    ami           = "ami-0e55b159cbfafe1f0" # replace with a real Debian AMI id
    instance_type = "t3.micro"
  }
```

Then plan:

```bash
terraform plan
```

Only `aws_instance.main["debian"]` is added. Compare with `count`, where inserting an item in the middle of the list would recreate every later index.

14. Now remove `amazon2` from `terraform.tfvars` and plan again.

```bash
terraform plan
```

Only `aws_instance.main["amazon2"]` is destroyed; `ubuntu` and `debian` are untouched.

15. If you inherit a `count`-based config, migrate keys rather than recreating resources.

```bash
terraform state mv 'aws_instance.main[0]' 'aws_instance.main["ubuntu"]'
terraform state mv 'aws_instance.main[1]' 'aws_instance.main["amazon2"]'
terraform plan   # should report: No changes.
```

## Expected Output

`terraform apply` tail:

```
Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

Outputs:

all_instance_ids_as_list = [
  "i-0aa11bb22cc33dd44",
  "i-0bb22cc33dd44ee55",
]
instance_ids = {
  "amazon2" = "i-0aa11bb22cc33dd44"
  "ubuntu"  = "i-0bb22cc33dd44ee55"
}
instance_placement = {
  "amazon2" = "subnet-0abc111"
  "ubuntu"  = "subnet-0def222"
}
one_instance_id = "i-0bb22cc33dd44ee55"
sg_ids = {
  "app" = "sg-0111aaa"
  "web" = "sg-0222bbb"
}
subnet_ids = [ "subnet-0abc111", "subnet-0def222" ]
vpc_id     = "vpc-09f8e7d6c5b4a3210"
```

`terraform state list`:

```
data.aws_availability_zones.az
aws_instance.main["amazon2"]
aws_instance.main["ubuntu"]
aws_security_group.extra["app"]
aws_security_group.extra["web"]
aws_subnet.main[0]
aws_subnet.main[1]
aws_vpc.main
```

Step 13 plan:

```
Terraform will perform the following actions:

  # aws_instance.main["debian"] will be created
  + resource "aws_instance" "main" {
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

Step 14 plan:

```
  # aws_instance.main["amazon2"] will be destroyed

Plan: 0 to add, 0 to change, 1 to destroy.
```

## Verification

```bash
terraform state list
terraform output instance_placement

aws ec2 describe-instances --region eu-north-1 \
  --filters "Name=tag:Name,Values=project-01-instance-*" "Name=instance-state-name,Values=running,pending" \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,AMI:ImageId,Subnet:SubnetId,Type:InstanceType}" \
  --output table
```

Checks to confirm:
- VPC `project-01-vpc` exists.
- 2 subnets: `project-01-subnet-0` (10.0.0.0/24) and `project-01-subnet-1` (10.0.1.0/24).
- 2 instances: `project-01-instance-ubuntu` and `project-01-instance-amazon2`, each with a different `ImageId`.
- The two instances sit in **different** subnets — the `index(keys(...)) % length(...)` round-robin worked.
- State addresses are keyed by name, not by number.
- Adding a key adds exactly one resource; removing a key destroys exactly one (steps 13 and 14).
- `terraform plan` right after apply reports `No changes.`

## Cleanup

```bash
cd ~/tf-foreach-demo
terraform destroy
# review, then confirm: yes

terraform state list   # should be empty
aws ec2 describe-instances --region eu-north-1 \
  --filters "Name=tag:Name,Values=project-01-instance-*" "Name=instance-state-name,Values=running,pending" \
  --query "Reservations[].Instances[].InstanceId" --output text

cd ~ && rm -rf ~/tf-foreach-demo
```

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Invalid for_each argument: must be a map, or set of strings` | Passed a list | Wrap it: `toset(var.list)`, or build a map with `{for x in list : x.name => x}` |
| `Invalid for_each argument: set includes a duplicate` | Duplicate values in the list you converted | `distinct()` before `toset()`, or pick a unique key |
| `The given "for_each" argument value is unsuitable: null` | Variable is `null` | Give it a default `{}` and add a `validation` for non-empty |
| `Because aws_instance.main has for_each set, its attributes must be accessed on specific instances` | Wrote `aws_instance.main.id` | Use `aws_instance.main["ubuntu"].id` or `values(aws_instance.main)[*].id` |
| `aws_instance.main[*].id` returns nothing useful | Splat is for `count`, not `for_each` | Use `values(...)[*].id` or a `for` expression |
| `each.value.ami` — `object has no attribute "ami"` | Map values are strings, not objects | Declare `map(object({ami = string, ...}))` |
| `Invalid for_each argument ... depends on resource attributes that cannot be determined until apply` | Keys derived from a not-yet-created resource | Key on static variables/locals; apply in stages if unavoidable |
| Resources recreated after refactoring | Changed the map keys | Keys are identity — use `terraform state mv` to rename instead |
| `InvalidAMIID.NotFound` | AMI ID belongs to a different region | Look IDs up per region via SSM parameters or `data.aws_ami` |
| `count` and `for_each` both set on one resource | Only one meta-argument allowed | Choose one |
| `each` is not available | Used `each.key` in a block without `for_each` | `each` only exists inside a `for_each` block |

## Key Takeaways
- `for_each` accepts a **map** or a **set of strings** only. Convert lists with `toset()` or a `for` expression.
- `each.key` is the identity, `each.value` is the payload; for a set the two are equal.
- State keys are names (`["ubuntu"]`), so adding or removing one entry affects only that entry — this is the fix for `count`'s index shifting.
- Map keys are identity: renaming a key destroys and recreates. Use `terraform state mv` to rename safely.
- `keys()` sorts alphabetically; never depend on the order you wrote the entries.
- Reference all `for_each` instances with `values(res)[*].attr` or a `for` expression, not `[*]`.
- Rule of thumb: `count` for identical interchangeable copies and on/off toggles; `for_each` for everything with a distinct identity or per-item configuration. When in doubt, `for_each`.
- Destroy billable labs as soon as verification passes.

## Next: [Day 10 — Terraform Modules](../Day-10-Modules/README.md)
