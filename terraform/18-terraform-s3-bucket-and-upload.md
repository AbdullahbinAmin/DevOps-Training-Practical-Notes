# Chapter 18 — Terraform: S3 Bucket Create and Upload Files

## Objective
Create an S3 bucket and upload a file to it using Terraform.

## Prerequisites
- Chapter 9 `.env` file in `tf-aws`
- Chapter 17 understanding of buckets and objects
- Your region code

## Architecture / Flow
```text
myfile.txt (local) ──> aws_s3_object ──> aws_s3_bucket ──> S3
```

## Step 1 — Create folder and load credentials
Open the terminal in the `tf-aws` folder:

```bash
source .env
mkdir s3
cd s3
```
**Expected:** you are inside `tf-aws/s3`.

## Step 2 — Create the file to upload
Create a file `myfile.txt` inside `s3` with the text:

```text
Hello World
```

Verify:
```bash
cat myfile.txt
```

## Step 3 — Create `main.tf`

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

resource "aws_s3_bucket" "demo_bucket" {
  bucket = "<UNIQUE_BUCKET_NAME>"
}

resource "aws_s3_object" "bucket_data" {
  bucket = aws_s3_bucket.demo_bucket.bucket
  source = "myfile.txt"
  key    = "mydata.txt"
}
```

Replace:
- `<YOUR_REGION>` → for example `eu-north-1`.
- `<UNIQUE_BUCKET_NAME>` → a globally unique lowercase name, for example `demo-bucket-abcd12345-yourname`.

**Explanation:**
- `aws_s3_bucket` creates the bucket. `bucket` is the real bucket name in AWS. The block name `demo_bucket` is only for Terraform.
- `aws_s3_object` uploads a file (files inside a bucket are called objects).
- `bucket` → which bucket to upload to. We refer to the first resource with `aws_s3_bucket.demo_bucket.bucket`.
- `source` → the local file path (current folder).
- `key` → the file name to use inside the bucket.

## Step 4 — Run Terraform
```bash
terraform init
terraform validate
terraform plan
terraform apply
```
Type `yes` when asked.

**Expected:**
```text
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```
(Bucket creation takes a fraction of a second.)

## Step 5 — Verify in the console
1. S3 → click your bucket (refresh the list if needed).
2. You should see the object `mydata.txt`.
3. Click it → **Open** (or use the Object URL / **Download**). Text shows `Hello World`.

## Troubleshooting

### Error 1
```text
BucketAlreadyExists
```
**Reason:** Bucket name not unique.
**Fix:** Change the `bucket` value in `main.tf` to something more unique. (Chapter 19 solves this automatically.)

### Error 2
```text
Error: Invalid bucket name
```
**Fix:** Use only lowercase letters, numbers and hyphens.

### Error 3
```text
no such file or directory: myfile.txt
```
**Reason:** File missing or you are in the wrong folder.
**Fix:** Run `ls` and make sure `myfile.txt` is next to `main.tf`.

## Cleanup
Do not destroy yet if you continue directly to Chapter 19 (it modifies this same config). Otherwise:
```bash
terraform destroy
```

## Final Result
Bucket exists and `mydata.txt` in it contains `Hello World`.
