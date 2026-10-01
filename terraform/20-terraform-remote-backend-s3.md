# Chapter 20 — Terraform Backend: Remote State Management with S3

## Objective
Store the Terraform state file in an S3 bucket instead of on your laptop.

## Prerequisites from Chapter 19
An S3 bucket already exists (for example `demo-bucket-<random hex>`). **The backend bucket must exist before you use it.** Have the exact bucket name ready (S3 console → copy the name).

Also needed: `.env` credentials, AMI ID and region.

## Why remote state?
- If the laptop crashes, local `terraform.tfstate` is lost.
- Team members need the same state to work on the same infrastructure.
- Remote state (S3) is safe and shared.

## Architecture / Flow
```text
Terraform (laptop) ──> reads/writes state ──> S3 bucket (backend.tfstate)
```

## Step 1 — Look at local state (a reminder)
In `tf-aws/s3` you saw `terraform.tfstate` locally. In the new folder it will not appear.

## Step 2 — New folder
From the `tf-aws` folder:

```bash
source .env
mkdir tf-backend
cd tf-backend
```

## Step 3 — Create `main.tf`
We use an EC2 instance as the example resource.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket = "<STATE_BUCKET_NAME>"
    key    = "backend.tfstate"
    region = "<YOUR_REGION>"
  }
}

provider "aws" {
  region = "<YOUR_REGION>"
}

resource "aws_instance" "my_server" {
  ami           = "<YOUR_AMI_ID>"
  instance_type = "<YOUR_INSTANCE_TYPE>"

  tags = {
    Name = "sample-server"
  }
}
```

Replace:
- `<STATE_BUCKET_NAME>` → the existing bucket name, for example `demo-bucket-3f9a1c0b7d2e4a55`.
- `<YOUR_REGION>` → the region where **that bucket** exists.
- `<YOUR_AMI_ID>` → an AMI ID from the same region.
- `<YOUR_INSTANCE_TYPE>` → for example `t3.micro`.

**Explanation:**
- `backend "s3"` lives inside the `terraform` block.
- `bucket` → bucket that will store the state.
- `key` → the file name (path) of the state file inside the bucket. You can choose it; here `backend.tfstate`.
- `region` → the region of the bucket.
- Other backends exist: Azure, GCS, local, Kubernetes, etc.

⚠️ Verification Required: newer Terraform versions support state locking in S3 using `use_lockfile = true` in the backend block (older setups used a DynamoDB table). This was not shown in the video; check the official S3 backend documentation before using it in real projects.

## Step 4 — Initialize
```bash
terraform init
```
**Expected:**
```text
Successfully configured the backend "s3"!
Terraform has been successfully initialized!
```

## Step 5 — Apply
```bash
terraform apply
```
Type `yes`. **Expected:** `Apply complete! Resources: 1 added`.

## Step 6 — Verify remote state
1. Look at the local folder: there is **no** `terraform.tfstate` file (only `.terraform` and lock file).
2. Console → S3 → open your bucket → refresh. A new object `backend.tfstate` appears next to `mydata.txt`.
3. Click `backend.tfstate` → **Open** (or download) to see the JSON with the instance details.

## Step 7 — Destroy and see state update
```bash
terraform destroy
```
Type `yes`. **Expected:** `Destroy complete! Resources: 1 destroyed.`

Now open `backend.tfstate` in the bucket again (refresh, open latest). The `resources` list is now empty — the state file was updated in S3.

## Important warnings
- Do not delete the backend bucket while any project still uses it.
- Do not put the state bucket in the same Terraform config that uses it as a backend (in real projects create it separately, as done here).
- Restrict who can access the bucket (state can contain sensitive data).

## Troubleshooting

### Error 1
```text
Error: Failed to get existing workspaces: S3 bucket does not exist
```
**Reason:** Wrong bucket name or region, or bucket was destroyed.
**Fix:** Copy the bucket name from the S3 console exactly and confirm the region.

### Error 2
```text
Backend configuration changed
```
**Reason:** You edited the backend block after `init`.
**Fix:**
```bash
terraform init -reconfigure
```

### Error 3
`AccessDenied` on the bucket.
**Reason:** Credentials are not loaded.
**Fix:** `source ../.env`

## Final Result
State is stored in S3 (`backend.tfstate`), local folder has no state file, and destroy updates the remote state.
