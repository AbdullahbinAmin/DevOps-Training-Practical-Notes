# Chapter 27 — Terraform Data Source

## Objective
Use **data sources** to read information that already exists (outside your Terraform config), for example the latest AMI ID.

## What is a data source?
A data source lets you **fetch and use information from an external source that already exists** (not created in this config, possibly not even by you) and is useful for getting **dynamic data** you need in your config.

## Where is it useful? (real cases)
1. **Existing VPC:** an admin already created a VPC for the company. You are asked to create an EC2 instance inside it. You cannot create a new VPC — you must read the existing one.
2. **Common security group:** a shared firewall rule already exists. Reuse it instead of creating 5 separate groups.
3. **Dynamic data:** for example the latest AMI ID. Instead of copying an AMI ID from the console each time, let Terraform find it.

## Prerequisites
- `.env` credentials in `tf-aws`
- Region code

## Cost warning
In this chapter we only run `terraform plan` — no resources are created.

## Step 1 — Folder and provider
```bash
source .env
mkdir tf-data-sources
cd tf-data-sources
```

Create `main.tf`:

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
```

## Step 2 — Try a data block with NO filter (see the error)
Add:

```hcl
data "aws_ami" "amazon_linux" {
}

output "aws_ami_result" {
  value = data.aws_ami.amazon_linux
}
```

```bash
terraform init
terraform plan
```
**Expected error:**
```text
Error: Your query returned no results / more than one result ...
```
(In the video: "Your query returned more than one result".)

**Reason:** There are thousands of AMIs. We must filter to get exactly one.

Note: in some provider versions a data block with no arguments may fail validation with a message about a missing `owners` or filter. Either way, the lesson is the same: you must tell Terraform how to pick one AMI.

## Step 3 — Where to read how to filter
Open the provider docs: **https://registry.terraform.io/providers/hashicorp/aws/latest/docs** → search `aws_ami`. Make sure you open the one under **Data Sources** (not Resources). Read the example: it uses `most_recent`, `owners`, and `filter` blocks.

## Step 4 — Add the filters
Replace the data block with:

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
}
```
⚠️ Verification Required: AMI name patterns can change. To check, in the console go to EC2 → **AMIs** → **Public images** → search `al2023-ami-2023` and confirm the naming.

**Explain:**
- `most_recent = true` → pick the newest match.
- `owners = ["amazon"]` → only images published by Amazon (safer).
- `filter { name = "name" ... }` → `values` is a **list** of name patterns; `*` is a wildcard.

## Step 5 — Output only the ID
Change the output:

```hcl
output "aws_ami_id" {
  value = data.aws_ami.amazon_linux.id
}
```

Run:
```bash
terraform plan
```
**Expected:**
```text
Changes to Outputs:
  + aws_ami_id = "ami-xxxxxxxxxxxxxxxxx"
```

## Step 6 — Verify the AMI in the console
EC2 → **AMIs** → **Public images** → paste the AMI ID into the search. You should see an image owned by Amazon with the name matching your filter.

## Step 7 — Use it in an EC2 resource (do NOT apply unless you accept small charges)
```hcl
resource "aws_instance" "my_server" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "<YOUR_INSTANCE_TYPE>"
}
```
- `<YOUR_INSTANCE_TYPE>` → for example `t3.micro`.

```bash
terraform plan
```
**Expected:** `Plan: 1 to add`, with `ami` set to the ID found. Stop at plan. **Do not apply** in class unless you want to create and later destroy an instance.

Important: reference a data source with the prefix `data.` → `data.<type>.<name>.<attribute>`.

## Troubleshooting

### Error 1
```text
Your query returned more than one result. Please try a more specific search criteria
```
**Fix:** Add `most_recent = true` or stronger filters.

### Error 2
```text
Your query returned no results
```
**Reason:** Filter pattern is wrong for your region/date.
**Fix:** Check the AMI name in the console (EC2 → AMIs → Public images) and adjust the pattern.

## Final Result
Terraform automatically finds the latest AMI ID; you do not copy it manually.
