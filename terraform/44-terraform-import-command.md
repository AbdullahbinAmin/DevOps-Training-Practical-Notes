# Chapter 44 — Terraform Import: Adopting Existing Infrastructure

## Objective
Bring pre-existing, manually created AWS resources under Terraform state management using the **`terraform import`** command without destroying or rebuilding them.

## Prerequisites
- AWS credentials loaded (`source .env`)
- An existing AWS resource (we will create a manual S3 bucket or EC2 instance to import)

---

## 1. When to Use `terraform import`

In enterprise environments:
- Infrastructure was built manually through the AWS Management Console or AWS CLI.
- You are migrating from CloudFormation, Ansible, or manual operations to Terraform.
- **Goal**: Start tracking and modifying the existing resource through code without downtime or data loss.

---

## Step 1 — Create a Manual AWS Resource for Testing

1. Open **AWS Management Console → S3 → Create bucket**.
2. Give it a globally unique name: e.g. `my-manual-import-demo-20261001`.
3. Choose your active region (e.g. `eu-north-1`).
4. Keep all defaults and click **Create bucket**.

---

## Step 2 — Set Up the Terraform Lab Folder

```bash
source .env
mkdir -p tf-import-s3 && cd tf-import-s3
```

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
  region = "eu-north-1" # Must match bucket region
}
```

---

## Step 3 — Create an Empty Resource Block in `main.tf`

Terraform requires a target resource address in your code before it can bind cloud infrastructure to state.

Create `main.tf`:

```hcl
# Empty resource block shell for the import target
resource "aws_s3_bucket" "main" {
  # Attributes will be filled in after import
}
```

Initialize the directory:
```bash
terraform init
```

---

## Step 4 — Run `terraform import`

Syntax:
```bash
terraform import <RESOURCE_TYPE>.<RESOURCE_NAME> <CLOUD_RESOURCE_ID>
```

Execute the import command using your bucket name:
```bash
terraform import aws_s3_bucket.main my-manual-import-demo-20261001
```

**Output:**
```text
aws_s3_bucket.main: Importing from ID "my-manual-import-demo-20261001"...
aws_s3_bucket.main: Import prepared!
  Prepared aws_s3_bucket for import
aws_s3_bucket.main: Refreshing state... [id=my-manual-import-demo-20261001]

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.
```

---

## Step 5 — Inspect the Imported State

Verify that Terraform's state now contains the bucket:
```bash
terraform state list
```
**Output:**
```text
aws_s3_bucket.main
```

Inspect all attributes discovered by Terraform:
```bash
terraform state show aws_s3_bucket.main
```
You will see the `bucket`, `arn`, `region`, `hosted_zone_id`, etc.

---

## Step 6 — Reconcile HCL Code with Live State

`terraform import` only updates `terraform.tfstate`; it does **not** automatically write your HCL code.

If you run `terraform plan` right now with an empty block, Terraform may attempt in-place changes or report missing required attributes.

Update `main.tf` to match the live imported state:

```hcl
resource "aws_s3_bucket" "main" {
  bucket = "my-manual-import-demo-20261001"

  tags = {
    ManagedBy = "Terraform-Imported"
  }
}
```

Run `terraform plan`:
```bash
terraform plan
```
Once your HCL matches the live state, Terraform reports:
```text
Plan: 0 to add, 1 to change (tags), 0 to destroy.
```

Apply the tag update:
```bash
terraform apply -auto-approve
```

---

## Step 7 — Managing & Destroying via Terraform

The pre-existing bucket is now 100% under Terraform control!

Run:
```bash
terraform destroy -auto-approve
```
Verify in the AWS Console that the bucket has been deleted by Terraform.
