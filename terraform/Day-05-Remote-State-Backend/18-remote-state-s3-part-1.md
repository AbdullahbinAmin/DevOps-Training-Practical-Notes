# Day 05 — Terraform Backend: Remote State Management with S3 (Part I)

> Move Terraform state out of your laptop and into a private, encrypted, versioned S3 bucket so a whole team can safely share one source of truth.

## Learning Objectives
- Explain what `terraform.tfstate` is and what it stores.
- Describe why local state is risky (laptop crash, no collaboration).
- Write an S3 `backend` block with bucket, key, region and encryption.
- Run `terraform init` to initialise the backend and confirm state lands in S3.
- Understand why the state bucket must be private and encrypted.

## Prerequisites
- Terraform 1.x installed (`terraform version`).
- AWS CLI configured with a profile or environment credentials (never hardcode keys in `.tf` files).
- An existing S3 bucket you own, in a known region (slides use `demo-bucket-abc12345` in `ap-northeast-1`).
- IAM permissions: `s3:ListBucket`, `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on that bucket.

## Concept

**What is state?** State is Terraform's memory of your infrastructure. After every `apply`, Terraform records every resource it created — IDs, ARNs, attributes and outputs — in a JSON file called `terraform.tfstate`.

```json
{
  "version": 4,
  "terraform_version": "1.7.0",
  "serial": 5,
  "resources": [
    {
      "type": "aws_s3_bucket",
      "name": "demo_bucket",
      "instances": [
        {
          "attributes": {
            "id": "demo-bucket-12345",
            "arn": "arn:aws:s3:::demo-bucket-12345",
            "bucket_domain_name": "demo-bucket-12345.s3.amazonaws.com"
          }
        }
      ]
    }
  ],
  "outputs": {}
}
```

**Local state problems**
1. Laptop crash or a deleted folder means the state is gone. Terraform then no longer knows what it built, and the next `apply` tries to recreate everything.
2. Team collaboration is impossible. Your teammate's copy of the state does not have your latest changes, so two people apply conflicting plans.

**Remote state benefits**
- Stored centrally in S3, not on one machine.
- Every team member reads and writes the same state file.
- Survives laptop loss; S3 versioning gives you rollback.
- Enables locking (Part II) so two applies cannot run at once.

**How it works.** You run `terraform init/plan/apply` → the AWS provider creates real resources → Terraform writes the updated state straight to `s3://<bucket>/<key>` instead of the local folder. The backend is simply the answer to the question "where does state live?". Other backend options: `s3`, `azurerm`, `gcs`, `kubernetes`, `local`.

**Security warning.** State is plain JSON and it stores resource attributes verbatim — including database passwords, generated secrets, private keys and tokens. Treat the state bucket as a secrets store: block all public access, enable server-side encryption, restrict the bucket policy to your team's roles, and never commit `terraform.tfstate` to Git.

## Step-by-Step Practical

1. Create the project folder.

```bash
mkdir tf-backend
cd tf-backend
touch main.tf
```

2. Prepare the state bucket (one-time, done outside the project that uses it). Slides create it manually; the CLI equivalent, with the security settings that matter:

```bash
BUCKET=demo-bucket-abc12345
REGION=ap-northeast-1

aws s3api create-bucket \
  --bucket "$BUCKET" \
  --region "$REGION" \
  --create-bucket-configuration LocationConstraint="$REGION"

# Versioning: keeps every previous state file so you can roll back
aws s3api put-bucket-versioning \
  --bucket "$BUCKET" \
  --versioning-configuration Status=Enabled

# Default encryption at rest
aws s3api put-bucket-encryption \
  --bucket "$BUCKET" \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Keep it private — state contains secrets
aws s3api put-public-access-block \
  --bucket "$BUCKET" \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

3. Write `main.tf` with the backend block plus a small resource to test with.

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Backend block: WHERE Terraform stores state
  backend "s3" {
    bucket  = "demo-bucket-abc12345" # existing S3 bucket
    key     = "backend.tfstate"      # path of the state file inside the bucket
    region  = "ap-northeast-1"       # bucket's region
    encrypt = true                   # encrypt state at rest
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

resource "aws_instance" "my_server" {
  ami           = "ami-0abcd1234abcd1234" # replace with a valid AMI for your region
  instance_type = "t3.micro"

  tags = {
    Name = "my-server"
  }
}
```

Note the backend block accepts only literal values — no variables, no interpolation.

4. Initialise the backend and apply.

```bash
terraform init      # downloads provider + configures the S3 backend
terraform validate
terraform plan      # see what will change
terraform apply     # type: yes
```

5. If this project already had a local `terraform.tfstate`, adopt it into S3 instead of starting empty:

```bash
terraform init -migrate-state
```

Terraform asks to copy the existing local state to the new backend — answer `yes`. Keep the local `terraform.tfstate.backup` until you have confirmed the remote copy is good.

## Expected Output

```text
Initializing the backend...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Terraform has been successfully initialized!
```

```text
aws_instance.my_server: Creating...
aws_instance.my_server: Creation complete after 32s [id=i-0123456789abcdef0]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The EC2 instance exists in AWS, and `backend.tfstate` now exists in the bucket. Your local folder contains `main.tf` and `.terraform/` but **no** `terraform.tfstate`.

## Verification

```bash
# 1. State object exists in the bucket
aws s3 ls s3://demo-bucket-abc12345/

# 2. Read state through Terraform (never edit the object by hand)
terraform state list
terraform show

# 3. Confirm encryption and versioning are on
aws s3api get-bucket-encryption --bucket demo-bucket-abc12345
aws s3api get-bucket-versioning --bucket demo-bucket-abc12345

# 4. Confirm the bucket is not public
aws s3api get-public-access-block --bucket demo-bucket-abc12345
```

In the AWS console: S3 → `demo-bucket-abc12345` → you should see an object `backend.tfstate` of a few KB, type `tfstate`. It contains the full detail of the EC2 instance.

## Cleanup

```bash
terraform destroy   # type: yes
```

Resources are deleted in AWS and the state file in S3 is updated automatically to reflect an empty infrastructure. The local folder still has no `terraform.tfstate`.

Keep the bucket if you continue to Part II. To remove it entirely (destructive — confirm before running):

```bash
aws s3 rm s3://demo-bucket-abc12345/backend.tfstate
aws s3api delete-bucket --bucket demo-bucket-abc12345 --region ap-northeast-1
```

Versioned buckets need all object versions removed before deletion.

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `Error: Failed to get existing workspaces: S3 bucket does not exist` | Bucket named in the backend block was never created, or wrong name | Create the bucket first, then re-run `terraform init` |
| `AccessDenied` / `403 Forbidden` on init | IAM identity lacks `s3:ListBucket` / `s3:GetObject` / `s3:PutObject` | Attach a policy granting those actions on the bucket and `bucket/*` |
| `BucketRegionError: incorrect region` | `region` in backend block does not match the bucket's real region | Set the correct region, delete `.terraform/`, re-init |
| `Backend configuration changed` | Bucket/key/region edited after a previous init | `terraform init -reconfigure` (fresh state) or `terraform init -migrate-state` (move state) |
| `Variables not allowed` in backend block | Used `var.*` or interpolation inside `backend "s3"` | Use literals, or pass values with `terraform init -backend-config=backend.hcl` |
| `NoSuchBucket` after a teammate applied | Two projects sharing one `key` and overwriting each other | Give each project a unique `key`, e.g. `prod/network/terraform.tfstate` |
| Plan wants to recreate everything | Init ran without migrating the old local state | Restore the local state file and run `terraform init -migrate-state` |

## Key Takeaways
- `terraform.tfstate` is Terraform's memory of your infrastructure; losing it means losing control of it.
- The `backend` block tells Terraform *where* to store state; `s3` is the standard AWS choice.
- Always create the bucket first, then configure the backend, then `terraform init`.
- Turn on versioning and encryption, and block public access — state holds secrets.
- Use `terraform init -migrate-state` to move existing local state into S3 without recreating resources.
- Remote state makes Terraform safe, reliable and team-friendly.

## Next: [Remote State with S3 — Part II (locking, versioning, team workflow)](./19-remote-state-s3-part-2.md)
