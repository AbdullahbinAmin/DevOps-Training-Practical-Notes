# Day 05 — Terraform Backend: Remote State Management with S3 (Part II)

> Harden the S3 backend: state locking (DynamoDB table and the newer native `use_lockfile`), versioning-based recovery, and the day-to-day team workflow.

## Learning Objectives
- Explain why concurrent applies corrupt state and how locking prevents it.
- Configure locking two ways: a DynamoDB lock table, and S3 native locking with `use_lockfile = true`.
- Recover a broken state from S3 object versions.
- Use one bucket for many projects and environments via the `key` prefix.
- Follow a safe team workflow and know when `force-unlock` is appropriate.

## Prerequisites
- Part I completed: a private, encrypted, versioned S3 bucket holding `backend.tfstate`.
- Terraform 1.10 or newer if you want `use_lockfile` (S3-native locking). Older 1.x needs DynamoDB.
- IAM permissions for locking: `dynamodb:GetItem`, `PutItem`, `DeleteItem` on the lock table, **or** `s3:PutObject`/`s3:DeleteObject` on `<bucket>/<key>.tflock` for native locking.

## Concept

**Why remote state, recapped.** With local state, a laptop crash loses the state and no teammate ever sees your latest changes. Stored in S3, one central `backend.tfstate` is read and written by everyone, updated automatically after every `init / plan / apply / destroy`.

**Local vs remote.**

| | Local state | Remote state (S3) |
| --- | --- | --- |
| Location | `./terraform.tfstate` on one machine | `s3://bucket/key` |
| Survives laptop loss | No | Yes |
| Team access | No | Yes |
| Locking | None | DynamoDB or S3 lockfile |
| History / rollback | Single `.backup` file | Full S3 object versions |
| Encryption | Plain file on disk | SSE-S3 or SSE-KMS at rest |

**Locking, in plain English.** State is a single file. If two engineers run `apply` at the same moment, both read the same starting state and both write their own result — the second write erases the first, and Terraform's record no longer matches reality (duplicate or orphaned resources). A lock is a small "in use" marker: Terraform claims it before touching state and releases it when done. Anyone else who tries meanwhile gets a clear "state locked by <user>" message instead of silent corruption.

Two implementations:
- **DynamoDB table** — the long-standing approach. A table with a partition key named exactly `LockID`; Terraform writes a lock item there. Works on every 1.x version.
- **S3 native locking (`use_lockfile = true`)** — since Terraform 1.10, S3's conditional writes let Terraform hold the lock as an object `<key>.tflock` in the same bucket. No extra table, no extra cost, one less thing to manage. This is now the recommended default; the `dynamodb_table` argument is deprecated in newer AWS provider/backend versions.

**Versioning is your undo.** Every write creates a new S3 object version. If a state file is truncated or a bad `state rm` happens, you restore the previous version instead of rebuilding infrastructure.

**Still remember:** state contains secrets in plaintext. Private bucket, `encrypt = true`, least-privilege bucket policy, and `terraform.tfstate*` in `.gitignore`.

## Step-by-Step Practical

1. Option A — S3 native locking (Terraform >= 1.10, preferred). Edit the backend block only:

```hcl
terraform {
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket       = "demo-bucket-abc12345"
    key          = "dev/ec2/backend.tfstate"
    region       = "ap-northeast-1"
    encrypt      = true
    use_lockfile = true # lock held as dev/ec2/backend.tfstate.tflock
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

resource "aws_instance" "my_server" {
  ami           = "ami-0abcd1234abcd1234" # replace with a valid AMI
  instance_type = "t3.micro"

  tags = {
    Name = "my-server"
  }
}
```

2. Option B — DynamoDB lock table (any Terraform 1.x). Create the table once:

```bash
aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-northeast-1
```

Then reference it:

```hcl
  backend "s3" {
    bucket         = "demo-bucket-abc12345"
    key            = "dev/ec2/backend.tfstate"
    region         = "ap-northeast-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
```

The partition key must be named `LockID` (case-sensitive) and be type String, or locking fails.

3. Apply the changed backend configuration. Changing bucket, key or locking settings requires a re-init:

```bash
terraform init -migrate-state   # moves state to the new key, keeps resources
terraform plan                  # expect: No changes
terraform apply
```

4. Prove locking works. In terminal 1 start a slow apply, and in terminal 2 run another command while it is in flight:

```bash
# terminal 1
terraform apply

# terminal 2, at the same time
terraform plan
```

5. Keep multiple projects and environments in one bucket by separating the `key`:

```text
s3://demo-bucket-abc12345/
├── dev/network/terraform.tfstate
├── dev/ec2/backend.tfstate
├── stage/ec2/terraform.tfstate
└── prod/ec2/terraform.tfstate
```

Never let two configurations share one `key`.

6. Optional — keep credentials and environment values out of the file with a partial backend config:

```hcl
  backend "s3" {} # values supplied at init time
```

```bash
# backend-dev.hcl  (no secrets, just locations)
terraform init -backend-config=backend-dev.hcl
```

```bash
bucket       = "demo-bucket-abc12345"
key          = "dev/ec2/backend.tfstate"
region       = "ap-northeast-1"
encrypt      = true
use_lockfile = true
```

7. Recover a damaged state from versioning:

```bash
aws s3api list-object-versions \
  --bucket demo-bucket-abc12345 \
  --prefix dev/ec2/backend.tfstate

aws s3api get-object \
  --bucket demo-bucket-abc12345 \
  --key dev/ec2/backend.tfstate \
  --version-id <GOOD_VERSION_ID> \
  restored.tfstate

# Review restored.tfstate first, then push it back
terraform state push restored.tfstate
```

`terraform state push` overwrites remote state — inspect the file and confirm before running it.

## Expected Output

Init with locking configured:

```text
Initializing the backend...
Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.
```

Every locked operation prints:

```text
Acquiring state lock. This may take a few moments...
...
Releasing state lock. This may take a few moments...
```

A second, concurrent command is refused:

```text
Error: Error acquiring the state lock

Error message: operation error S3: PutObject, ConditionalRequestConflict
Lock Info:
  ID:        3f1c2b90-8f6f-4a1e-9d2f-11a2b3c4d5e6
  Path:      demo-bucket-abc12345/dev/ec2/backend.tfstate
  Operation: OperationTypeApply
  Who:       student@lab-machine
  Created:   2026-08-31 09:14:02 UTC

Terraform acquires a state lock to protect the state from being written
by multiple users at the same time.
```

After `terraform destroy`, resources are deleted in AWS and the state file in S3 is updated automatically; the local folder still has no `terraform.tfstate`.

## Verification

```bash
# Locking artefact present during an apply (native locking)
aws s3api list-objects-v2 --bucket demo-bucket-abc12345 --prefix dev/ec2/

# DynamoDB approach: a LockID item appears while an apply runs
aws dynamodb scan --table-name terraform-locks --region ap-northeast-1

# State is readable and complete
terraform state list
terraform show

# Versioning gives you history
aws s3api list-object-versions --bucket demo-bucket-abc12345 --prefix dev/ec2/

# Bucket still private and encrypted
aws s3api get-public-access-block --bucket demo-bucket-abc12345
aws s3api get-bucket-encryption   --bucket demo-bucket-abc12345
```

After a successful run, the `.tflock` object (or the DynamoDB item) is gone — a leftover lock means a crashed run.

## Cleanup

```bash
terraform destroy   # type: yes
```

Then, only if you no longer need the shared backend (destructive — check with your team first):

```bash
aws dynamodb delete-table --table-name terraform-locks --region ap-northeast-1

# Remove every object version before deleting a versioned bucket
aws s3 rm s3://demo-bucket-abc12345 --recursive
aws s3api delete-bucket --bucket demo-bucket-abc12345 --region ap-northeast-1
```

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `Error acquiring the state lock` with lock info shown | Someone else is genuinely running Terraform | Wait for them to finish; do not force-unlock a live run |
| Lock persists after a crashed/cancelled run | Terraform died before releasing the lock | Confirm nobody is running, then `terraform force-unlock <LOCK_ID>` |
| `ResourceNotFoundException: Requested resource not found` | `dynamodb_table` name wrong or table in another region | Create the table in the backend's region and match the name |
| `ValidationException: One or more parameter values were invalid` | Lock table partition key is not exactly `LockID` (String) | Recreate the table with `AttributeName=LockID,AttributeType=S` as HASH key |
| `Unsupported argument: use_lockfile` | Terraform older than 1.10 | Upgrade Terraform, or use the `dynamodb_table` approach |
| `Warning: dynamodb_table is deprecated` | Newer backend prefers native locking | Switch to `use_lockfile = true` and drop `dynamodb_table` |
| `ConditionalRequestConflict` on every run | Stale `.tflock` object left in the bucket | `terraform force-unlock <LOCK_ID>`, or delete the `.tflock` object after verifying no run is active |
| Two projects overwrite each other's state | Same `key` reused | Give each project/environment a unique `key` prefix |
| `Backend configuration changed` after editing locking settings | Backend definition differs from the cached one | `terraform init -migrate-state` (keep state) or `-reconfigure` (start fresh) |
| State file corrupted or emptied | Manual edit or interrupted write | Restore a previous S3 object version and `terraform state push` it |

## Key Takeaways
- Locking is what makes shared state safe: it serialises applies instead of letting them overwrite each other.
- Prefer `use_lockfile = true` on Terraform 1.10+; use a `LockID` DynamoDB table on older versions.
- `encrypt = true` plus versioning plus blocked public access is the minimum secure backend, because state stores secrets in plaintext.
- One bucket can serve many projects — isolate them with distinct `key` paths, never a shared key.
- S3 object versions are your rollback path; `terraform force-unlock` is a last resort, only after confirming no run is active.
- Remote state with locking is what turns Terraform from a solo tool into a team tool.

## Next: [Terraform Workspaces and Environment Separation](../Day-06-Workspaces/20-terraform-workspaces.md)
