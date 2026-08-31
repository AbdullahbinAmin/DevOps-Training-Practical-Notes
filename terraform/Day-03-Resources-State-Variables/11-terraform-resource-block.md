# Day 03 — Terraform Resource Block: Modifying an Existing Resource

> Change an already-created EC2 instance by editing its resource block, then learn which changes update in place and which force a replacement.

## Learning Objectives
- Read and edit an `aws_instance` resource block.
- Modify an argument (`instance_type`) and apply the change in place.
- Recognise a *force new resource* change (`ami`) that destroys and recreates the instance.
- Interpret the `plan` symbols `~`, `-/+`, `+`, `-`.

## Prerequisites
- Terraform 1.x installed (`terraform version`).
- AWS CLI configured with a non-production profile (`aws configure`) — never hardcode keys in `.tf` files.
- An EC2 instance already created by Terraform in this folder (from Day 02) with a `terraform.tfstate` present.
- Working directory example: `C:\aws-ec2` (Windows) or `~/aws-ec2` (Linux).

## Concept (plain English, beginner friendly)
A **resource block** is you telling Terraform: "I want one of these, configured like this." The `.tf` file is the desired state; AWS holds the real thing.

When you change the file, Terraform compares desired state against the recorded state and decides the smallest action needed:

- **Minor change** (like `instance_type`) → AWS supports updating it, so Terraform *modifies in place*. Plan shows `~`.
- **Major change** (like `ami`) → AWS cannot swap an AMI on a running instance, so Terraform *replaces* it: destroy old, create new. Plan shows `-/+ ... will be replaced` and the reason `(forces replacement)`.

That difference is why you always read the plan before typing `yes`. In-place is cheap; replacement means a new instance ID, a new public IP, and any data on the old root volume is gone.

## Step-by-Step Practical

1. Confirm the starting configuration. Your `main.tf` should look like this (before change):

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
  region = "us-east-1"
}

resource "aws_instance" "my_server" {
  ami           = "ami-0abcd1234abcdef0" # replace with a valid AMI in your region
  instance_type = "t3.micro"

  tags = {
    Name = "Sample-Server"
  }
}
```

2. Check the current real state before touching anything:

```bash
terraform show
terraform state list
```

3. Pick a new instance type. In the AWS Console this is under **EC2 Dashboard → Launch Instance → Instance Type**. Common options:

```text
t3.micro   t3.small   t3.medium   t3.large
t3a.medium t3a.large  t3.nano
```

4. Edit `main.tf` and change only the instance type, then save:

```hcl
resource "aws_instance" "my_server" {
  ami           = "ami-0abcd1234abcdef0"
  instance_type = "t3.nano" # changed from t3.micro

  tags = {
    Name = "Sample-Server"
  }
}
```

5. Validate and preview the change:

```bash
terraform validate
terraform plan
```

6. Apply it and confirm with `yes` when prompted:

```bash
terraform apply
```

7. Now make a **replacement-forcing** change. Edit the AMI to a different valid AMI ID and save:

```hcl
resource "aws_instance" "my_server" {
  ami           = "ami-0fedcba9876543210" # different AMI -> forces replacement
  instance_type = "t3.nano"

  tags = {
    Name = "Sample-Server"
  }
}
```

8. Preview it and read the plan carefully before approving:

```bash
terraform plan
```

9. Apply the replacement:

```bash
terraform apply
```

Terraform will destroy the old instance, create a new one, and report the new instance as running.

## Expected Output

In-place modify (step 5 plan, trimmed):

```text
Terraform will perform the following actions:

  # aws_instance.my_server will be updated in-place
  ~ resource "aws_instance" "my_server" {
      ~ instance_type = "t3.micro" -> "t3.nano"
        id            = "i-01234abcd"
        # (other attributes unchanged)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

After apply:

```text
Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

Replacement (step 8 plan, trimmed):

```text
  # aws_instance.my_server must be replaced
-/+ resource "aws_instance" "my_server" {
      ~ ami = "ami-0abcd1234abcdef0" -> "ami-0fedcba9876543210" # forces replacement
      ~ id  = "i-01234abcd" -> (known after apply)
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

After apply:

```text
Apply complete! Resources: 1 added, 0 changed, 1 destroyed.
```

## Verification
- CLI:

```bash
terraform state show aws_instance.my_server | grep -E "instance_type|ami|id|public_ip"
```

- AWS CLI:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=Sample-Server" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].{Id:InstanceId,Type:InstanceType,Ami:ImageId,State:State.Name}" \
  --output table
```

- AWS Console: **EC2 → Instances** shows `Sample-Server` with Instance Type `t3.nano` and Status `Running`. After the AMI change the Instance ID is different and the old one shows `terminated`.

## Cleanup

```bash
terraform destroy
```

Type `yes` to confirm. Verify in the console that the instance is `terminated`.

Never commit state or secrets. Add this `.gitignore` in the project folder:

```bash
printf '%s\n' '*.tfstate' '*.tfstate.*' '.terraform/' '*.tfvars' 'crash.log' > .gitignore
```

`terraform.tfstate` can contain resource details and sensitive values in plain text — keep it out of Git and use a remote backend for teams.

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `InvalidAMIID.NotFound` | AMI ID does not exist in the selected region | AMIs are region-specific; look up a valid AMI in your region or use an `aws_ami` data source |
| `Unsupported operation: The instance configuration ... is not supported` | Chosen instance type is unavailable in that AZ or incompatible with the AMI architecture | Pick another type (e.g. `t3.small`) or a matching x86/arm64 AMI |
| `IncorrectInstanceState: instance is not in the 'stopped' state` | Some attribute changes need the instance stopped | Stop the instance, or let Terraform replace it, or set `user_data_replace_on_change` where relevant |
| Plan shows replacement when you expected an update | You changed a `ForceNew` attribute (ami, subnet_id, availability_zone) | Accept the replacement knowingly, or revert that attribute |
| `Error acquiring the state lock` | A previous run crashed or another apply is running | Wait, or `terraform force-unlock <LOCK_ID>` only when certain no other run is active |
| `No changes. Your infrastructure matches the configuration.` | File not saved, or you edited a different folder's file | Save the file and re-run in the correct directory |

## Key Takeaways
- Minor changes are modified in place; major changes replace the resource.
- Always read `terraform plan` before `terraform apply` — look for `forces replacement`.
- `~` = update, `-/+` = replace, `+` = create, `-` = destroy.
- Replacement gives a new instance ID and new public IP; plan for downtime and data loss.
- Terraform sequences destroy and create automatically, and removing unused resources saves money.

## Next: [Terraform Resource Delete](./12-terraform-resource-delete.md)
