# Day 04 — Terraform: S3 Bucket Create and Upload Files

> Use Terraform to create an S3 bucket and upload a local file into it as an object, with the standard init / validate / plan / apply workflow.

## Learning Objectives
- Write a minimal Terraform configuration for the AWS provider 5.x.
- Create an S3 bucket with `aws_s3_bucket`.
- Upload a local file as an object with `aws_s3_object`.
- Run the core Terraform workflow and read its output.

## Prerequisites
- Terraform 1.x installed (`terraform -version`).
- AWS credentials configured (`aws configure` or environment variables). **Never hard-code keys in `.tf` files.**
- IAM permissions for `s3:CreateBucket`, `s3:PutObject`, `s3:DeleteObject`, `s3:DeleteBucket`, `s3:GetBucket*`.
- Completed the S3 overview lesson.

## Concept (plain English, beginner friendly)
Terraform describes infrastructure as code. You declare what you want in `.tf` files, and Terraform figures out the API calls needed to get there.

Two resources do all the work here:
- **`aws_s3_bucket`** — creates the bucket (the container).
- **`aws_s3_object`** — puts one object inside the bucket. `key` is the name it gets in S3, `source` is the local file path.

The object resource references the bucket with `aws_s3_bucket.demo_bucket.bucket`. That reference is important for two reasons: it avoids repeating the name, and it makes Terraform build an implicit **dependency** so the bucket is created before the upload.

Bucket names must be globally unique, so a hard-coded name will eventually collide. That is fine for one lesson; the next lesson replaces it with a random suffix.

## Step-by-Step Practical

1. Create the project folder and the local file to upload.

```bash
mkdir -p aws-s3
cd aws-s3
echo "Hello World" > myfile.txt
```

Target structure:

```text
aws-s3/
├── main.tf
├── myfile.txt
└── .terraform/        # created by terraform init
```

2. Create `main.tf`. **Change the bucket name to something unique to you.**

```hcl
# main.tf

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
  region = "ap-northeast-1" # pick your region
}

# ---------------------------------------------------------------
# 1. Create the S3 bucket
# ---------------------------------------------------------------
resource "aws_s3_bucket" "demo_bucket" {
  bucket = "demo-bucket-abcd12345" # MUST be globally unique

  tags = {
    Name        = "demo-bucket"
    Environment = "training"
    ManagedBy   = "terraform"
  }
}

# Keep the bucket private (this is the default, made explicit)
resource "aws_s3_bucket_public_access_block" "demo_bucket" {
  bucket = aws_s3_bucket.demo_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Encrypt objects at rest with S3-managed keys
resource "aws_s3_bucket_server_side_encryption_configuration" "demo_bucket" {
  bucket = aws_s3_bucket.demo_bucket.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# ---------------------------------------------------------------
# 2. Upload a local file to S3 as an object
# ---------------------------------------------------------------
resource "aws_s3_object" "bucket_data" {
  bucket = aws_s3_bucket.demo_bucket.bucket
  key    = "mydata.txt"       # name inside the bucket
  source = "myfile.txt"       # local file path
  etag   = filemd5("myfile.txt") # re-upload when the file changes

  content_type = "text/plain"
}
```

Note: no `acl` argument. On buckets created today ACLs are disabled (bucket owner enforced), so setting `acl` fails with `AccessControlListNotSupported`.

3. Add outputs so the useful values are printed. Create `outputs.tf`:

```hcl
# outputs.tf

output "bucket_name" {
  description = "Name of the created S3 bucket"
  value       = aws_s3_bucket.demo_bucket.bucket
}

output "bucket_arn" {
  description = "ARN of the created S3 bucket"
  value       = aws_s3_bucket.demo_bucket.arn
}

output "object_key" {
  description = "Key of the uploaded object"
  value       = aws_s3_object.bucket_data.key
}
```

4. Run the Terraform workflow.

```bash
terraform init      # download the AWS provider, initialise the working dir
terraform fmt       # format the files consistently
terraform validate  # check the configuration is syntactically valid
terraform plan      # preview what will be created
terraform apply     # create the resources (type: yes)
```

5. Inspect what Terraform recorded in state.

```bash
terraform state list
terraform output bucket_name
```

## Expected Output

`terraform init`:

```text
Terraform has been successfully initialized!
```

`terraform validate`:

```text
Success! The configuration is valid.
```

`terraform plan` (trimmed):

```text
Terraform will perform the following actions:

  # aws_s3_bucket.demo_bucket will be created
  + resource "aws_s3_bucket" "demo_bucket" {
      + bucket = "demo-bucket-abcd12345"
      ...
    }

  # aws_s3_object.bucket_data will be created
  + resource "aws_s3_object" "bucket_data" {
      + key    = "mydata.txt"
      + source = "myfile.txt"
      ...
    }

Plan: 4 to add, 0 to change, 0 to destroy.
```

`terraform apply`:

```text
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:

bucket_arn  = "arn:aws:s3:::demo-bucket-abcd12345"
bucket_name = "demo-bucket-abcd12345"
object_key  = "mydata.txt"
```

## Verification
1. AWS Console -> **S3** -> your bucket appears in the list.
2. Open the bucket -> **Objects** -> `mydata.txt` is listed (size ~12 B).
3. Select the object -> **Open** -> content shows `Hello World`.
4. From the terminal:

```bash
aws s3 ls s3://demo-bucket-abcd12345/
aws s3 cp s3://demo-bucket-abcd12345/mydata.txt - | cat
terraform plan   # should report: No changes. Your infrastructure matches the configuration.
```

An empty `terraform plan` after apply is the strongest signal that reality matches your code.

## Cleanup

```bash
terraform destroy   # type: yes
```

```text
Destroy complete! Resources: 4 destroyed.
```

`terraform destroy` deletes the object first, then the bucket, following the dependency graph in reverse. If you uploaded extra files by hand, Terraform does not know about them and the bucket delete will fail — empty it first with `aws s3 rm s3://<bucket>/ --recursive`.

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `BucketAlreadyExists` | Bucket name taken globally | Change the name; better, add a random suffix (next lesson) |
| `AccessControlListNotSupported` | Used `acl = "..."` on a bucket with ACLs disabled | Remove the `acl` argument; use policies instead |
| `no valid credential sources found` | AWS credentials not configured | Run `aws configure`, or export `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_REGION` |
| `Invalid function argument: no file exists for "myfile.txt"` | `source`/`filemd5` path wrong or you ran Terraform from another directory | `cd` into the project folder; use a path relative to the config, or `${path.module}/myfile.txt` |
| `Error: Missing required argument: bucket` | Typo in the resource reference | Reference `aws_s3_bucket.demo_bucket.bucket` or `.id` |
| Object not updated after editing the local file | No `etag`, so Terraform sees no change | Add `etag = filemd5("myfile.txt")` |
| `BucketNotEmpty` during destroy | Objects created outside Terraform | `aws s3 rm s3://<bucket>/ --recursive`, then destroy again |
| `Error acquiring the state lock` | A previous run crashed or another run is active | Wait for it, or `terraform force-unlock <LOCK_ID>` only if you are certain nothing else is running |

## Key Takeaways
- `aws_s3_bucket` creates the container; `aws_s3_object` uploads a single file.
- `key` = name in S3, `source` = local path; add `etag = filemd5(...)` so edits are re-uploaded.
- Referencing one resource from another creates the dependency order automatically.
- Workflow: `init` -> `validate` -> `plan` -> `apply`, and `destroy` to remove.
- Keep credentials out of `.tf` files; keep buckets private unless there is a reason not to.
- Hard-coded bucket names do not scale — generate uniqueness instead.

## Next: [Terraform Random Provider](./17-terraform-random-provider.md)
