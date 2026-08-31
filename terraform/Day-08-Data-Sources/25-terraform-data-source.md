# Day 08 — Terraform Data Source

> Fetch existing/dynamic information from AWS (like the latest AMI ID) and use it in your Terraform config instead of hardcoding.

## Learning Objectives
- Explain what a Terraform data source is and how it differs from a resource.
- Write a `data "aws_ami"` block with `most_recent` and `filter` to get one specific AMI.
- Use `data.<type>.<name>.<attribute>` inside a resource and in `output`.
- Debug the classic "your query returned more than one result" error.
- Verify a fetched AMI ID in the AWS Console.

## Prerequisites
- Terraform 1.x installed (`terraform -version`).
- AWS CLI configured with a non-production profile (`aws configure`) — never hardcode keys in `.tf` files.
- AWS provider 5.x (pinned below).
- Region used in these notes: `eu-north-1` (Stockholm).

## Concept (data source vs resource)
| | Resource | Data source |
|---|---|---|
| Keyword | `resource` | `data` |
| What it does | **Creates / updates / deletes** real infrastructure | **Only reads** infrastructure that already exists |
| Terraform state | Managed and owned by Terraform | Read-only lookup, refreshed each plan |
| Destroy | Deleted on `terraform destroy` | Nothing is deleted |
| Costs money | Usually yes | No |

Plain English: a resource is "make this for me", a data source is "go find this and tell me its ID".

Use a data source when:
- The AMI ID changes over time (latest Amazon Linux) — do not hardcode `ami-0xxxxxxxxxxxx`.
- The VPC / subnet / security group was created by another team or another Terraform state.
- You want one shared security group reused by EC2-1 … EC2-5 instead of recreating it.

Benefits: less hardcoding, reusable, dynamic, easier to maintain.

## Step-by-Step Practical

1. Create a working folder.

```bash
mkdir -p ~/tf-data-sources && cd ~/tf-data-sources
```

2. Create `providers.tf` with the provider block (no credentials inside the file).

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
  region = "eu-north-1" # Stockholm
  # Credentials come from environment vars or ~/.aws/credentials
}
```

3. First, see the problem. Create `main.tf` with a data block that has **no filters**.

```hcl
# BAD on purpose - too many AMIs match
data "aws_ami" "amz" {}

output "aws_ami" {
  value = data.aws_ami.amz
}
```

4. Initialise and plan.

```bash
terraform init
terraform plan
```

You will get an error because thousands of AMIs match (see Expected Output).

5. Now fix it. Replace `main.tf` with a filtered lookup.

```hcl
data "aws_ami" "amz" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
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

6. Create `outputs.tf` to print only the ID (not the whole object).

```hcl
output "ami_id" {
  description = "Latest Amazon Linux 2 AMI ID in this region"
  value       = data.aws_ami.amz.id
}

output "ami_name" {
  value = data.aws_ami.amz.name
}

output "ami_creation_date" {
  value = data.aws_ami.amz.creation_date
}
```

7. Plan again and read the outputs.

```bash
terraform fmt
terraform validate
terraform plan
```

8. Use the fetched ID in a resource — replace the hardcoded AMI.

```hcl
# ec2.tf  -- DO NOT APPLY unless you accept AWS charges
resource "aws_instance" "my_server" {
  ami           = data.aws_ami.amz.id # dynamic, from data source
  instance_type = "t3.micro"

  tags = {
    Name = "my-server"
  }
}
```

9. Optional: read the values without creating anything. `terraform plan` already refreshes data sources; to print outputs without an apply of real resources, keep only the `data` + `output` blocks and run:

```bash
terraform apply -auto-approve   # safe ONLY while ec2.tf is absent/commented
terraform output ami_id
```

Important: this lab is designed to stop at `terraform plan`. Applying an `aws_instance` starts billing.

## Expected Output

Step 4 (unfiltered) fails:

```bash
Error: Your query returned more than one result. Please try a more specific
search criteria, or set `most_recent` attribute to true.
```

Step 7 (filtered) succeeds:

```bash
Changes to Outputs:
  + ami_creation_date = "2024-xx-xxT00:00:00.000Z"
  + ami_id            = "ami-0abc1234def567890"
  + ami_name          = "amzn2-ami-hvm-2.0.20240xxx.0-x86_64-gp2"
```

## Verification
1. Copy the `ami_id` value from the plan output.
2. AWS Console → EC2 → Images → **AMIs** → change the dropdown to **Public images** (or open AMI Catalog) → paste the ID in the search box.
3. Confirm: Owner = `amazon`, Root device type = `ebs`, Virtualization = `hvm`, Architecture = `x86_64`.
4. Confirm the region selector shows **Europe (Stockholm) eu-north-1** — data sources are region-specific.
5. CLI cross-check:

```bash
aws ec2 describe-images --region eu-north-1 \
  --image-ids ami-0abc1234def567890 \
  --query 'Images[0].{Name:Name,Owner:OwnerId,Root:RootDeviceType}'
```

## Cleanup

```bash
# Only needed if you actually applied something
terraform destroy
```

Data sources create nothing, so if you only ran `init` + `plan` there is nothing to delete. To clear local files:

```bash
rm -rf .terraform .terraform.lock.hcl terraform.tfstate*
```

## Common Errors & Fixes
| Error | Cause | Fix |
|---|---|---|
| `Your query returned more than one result` | Data source matched many AMIs | Add `owners`, `filter` blocks, and `most_recent = true` |
| `Your query returned no results` | Filter name pattern or region wrong | Check the AMI name pattern and the provider `region` |
| `Reference to undeclared resource` | Wrote `aws_ami.amz.id` instead of `data.aws_ami.amz.id` | Always prefix data source references with `data.` |
| `NoCredentialProviders` / `no valid credential sources` | AWS credentials not configured | Run `aws configure` or export `AWS_PROFILE` |
| `UnauthorizedOperation` on DescribeImages | IAM user lacks EC2 read permission | Attach a read policy such as `AmazonEC2ReadOnlyAccess` |
| Output prints a huge object | `value = data.aws_ami.amz` | Output a single attribute: `data.aws_ami.amz.id` |
| Unexpected AMI after some weeks | `most_recent = true` picks up new releases | Pin an exact AMI ID or a fixed name if you need immutability |

## Key Takeaways
- `data` reads, `resource` creates — data sources never cost money and never get destroyed.
- Reference syntax is `data.<TYPE>.<NAME>.<ATTRIBUTE>`.
- `most_recent = true` plus `owners` plus `filter` narrows the result to exactly one AMI.
- Data sources are refreshed at plan/apply time, so values stay dynamic.
- Every data source is account- and region-specific.
- Stop at `terraform plan` while learning to avoid charges.

## Next: [VPC & SG with Data Source](./26-vpc-and-sg-with-data-source.md)
