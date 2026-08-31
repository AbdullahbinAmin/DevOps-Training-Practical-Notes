# Day 03 — Terraform Config and State Overview

> How declarative HCL configuration and the `terraform.tfstate` file work together, plus HCL vs JSON and local vs remote state.

## Learning Objectives
- Explain what "declarative" means for Terraform configuration.
- Describe what the state file stores and why Terraform needs it.
- Inspect state with `terraform show`, `state list`, and `state show`.
- Compare HCL and JSON configuration syntax.
- Configure remote state in S3 with DynamoDB locking.

## Prerequisites
- Terraform 1.x installed and AWS credentials configured outside the code.
- A project folder with a `main.tf`.
- For the remote-state section: permission to create an S3 bucket and a DynamoDB table.

## Concept (plain English, beginner friendly)

**Configuration is declarative.** You write HCL (HashiCorp Configuration Language) describing the *desired state* — "an EC2 instance, this AMI, this type, this tag." You do not write the sequence of API calls. Terraform reads the config, works out the steps, talks to AWS, and produces the result.

**State is Terraform's memory.** After `apply`, Terraform records what it actually created in `terraform.tfstate` (JSON). On the next run it compares three things — your config (what you want), the state (what it made last time), and reality in AWS — to decide what to create, update, or delete. Without state, Terraform would have no idea that `aws_instance.my_server` already exists and would try to make another one.

**Where state lives matters.** By default it is a local file in your working directory, which is fine for solo learning. In a team, a local file means nobody else can see what exists and two people can apply at once and corrupt things. Remote state (S3 for storage, DynamoDB for a lock) fixes both: shared visibility plus one-writer-at-a-time.

**HCL vs JSON.** Terraform accepts both. HCL (`.tf`) is clean and readable; JSON (`.tf.json`) is verbose and brace-heavy, and mostly used when a program generates the config. Write HCL by hand.

## Step-by-Step Practical

1. Create the declarative configuration in `main.tf`:

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
  ami           = "ami-0abcd1234abcdef0" # use a valid AMI for your region
  instance_type = "t3.micro"

  tags = {
    Name = "Sample-Server"
  }
}
```

2. Initialize, validate, plan, and apply:

```bash
terraform init
terraform validate
terraform plan
terraform apply
```

3. Look at what Terraform now remembers:

```bash
ls -l terraform.tfstate
terraform state list
terraform show
terraform state show aws_instance.my_server
```

4. Read the raw state as JSON (read only — never hand-edit this file):

```bash
terraform show -json | jq '.values.root_module.resources[] | {address, type, id: .values.id}'
```

5. See the same `provider` block expressed in JSON. This is for comparison only — do not keep both files in one folder:

```json
{
  "provider": {
    "aws": {
      "region": "us-east-1"
    }
  }
}
```

The HCL equivalent, which is what you should actually write:

```hcl
provider "aws" {
  region = "us-east-1"
}
```

6. Refresh state against reality (useful if someone changed something in the console):

```bash
terraform plan -refresh-only
terraform apply -refresh-only
```

7. Move to remote state. First create the backend resources once, using the AWS CLI:

```bash
aws s3api create-bucket --bucket my-tfstate-demo-bucket-1234 --region us-east-1
aws s3api put-bucket-versioning --bucket my-tfstate-demo-bucket-1234 \
  --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket my-tfstate-demo-bucket-1234 \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
aws dynamodb create-table --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST --region us-east-1
```

8. Add the backend block to `main.tf` inside the `terraform { }` block:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tfstate-demo-bucket-1234"
    key            = "day-03/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

9. Migrate the existing local state to S3 and answer `yes`:

```bash
terraform init -migrate-state
terraform state list
```

## Expected Output

After apply:

```text
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

`terraform state list`:

```text
aws_instance.my_server
```

`terraform state show aws_instance.my_server` (trimmed):

```text
# aws_instance.my_server:
resource "aws_instance" "my_server" {
    ami           = "ami-0abcd1234abcdef0"
    id            = "i-01234abcd"
    instance_type = "t3.micro"
    public_ip     = "178.141.12.45"
    tags          = {
        "Name" = "Sample-Server"
    }
}
```

Backend migration:

```text
Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.
```

## Verification
- `terraform.tfstate` exists locally (before migration) and its `resources` array contains your instance.
- `terraform plan` on an unchanged config reports:

```text
No changes. Your infrastructure matches the configuration.
```

- After migrating, the local state is a stub and the real object is in S3:

```bash
aws s3 ls s3://my-tfstate-demo-bucket-1234/day-03/
```

## Cleanup

```bash
terraform destroy
```

Remove the backend resources only when no environment needs them:

```bash
aws s3 rm s3://my-tfstate-demo-bucket-1234 --recursive
aws s3api delete-bucket --bucket my-tfstate-demo-bucket-1234
aws dynamodb delete-table --table-name terraform-locks
```

State files are sensitive: they hold resource IDs, IPs, and any values marked sensitive in plain text. Keep them out of version control:

```text
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl  # keep this one committed in real projects; ignore only if instructed
*.tfvars
```

Commit `.terraform.lock.hcl` in real projects so provider versions are reproducible; always ignore `*.tfstate`.

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `Backend initialization required: please run "terraform init"` | Backend block added or changed after init | Run `terraform init -migrate-state` |
| `Error loading state: AccessDenied` | IAM identity lacks S3/DynamoDB permissions | Grant `s3:GetObject/PutObject/ListBucket` on the bucket and `dynamodb:GetItem/PutItem/DeleteItem` on the table |
| `Error acquiring the state lock` | Another apply running, or a crashed run left the lock | Wait; if certain nobody is running, `terraform force-unlock <LOCK_ID>` |
| `state snapshot was created by Terraform v1.x.y, which is newer` | State written by a newer Terraform than your CLI | Upgrade your Terraform binary; never downgrade state |
| `Resource already exists` on apply | Resource created manually or state was deleted | `terraform import aws_instance.my_server i-01234abcd` |
| Duplicate resource error with both `.tf` and `.tf.json` | Same block defined in HCL and JSON in one folder | Keep only one format per configuration |
| Editing `terraform.tfstate` by hand breaks everything | Manual JSON edits desync state | Use `terraform state mv/rm/import` instead; restore from the S3 version history if needed |

## Key Takeaways
- Terraform is declarative: you declare the desired end state, it figures out the steps.
- `terraform.tfstate` is JSON, created after `apply`, and lets Terraform make intelligent decisions.
- State can be local (default, solo work) or remote (S3 + DynamoDB locking) for teams.
- Remote state enables collaboration, sharing across environments, and safer concurrent work.
- HCL is recommended over JSON for readability; JSON is for machine-generated configs.
- Command recap: `init`, `validate`, `plan`, `apply`, `destroy`.

## Next: [Terraform Variables and Outputs](./14-variables-and-outputs.md)
