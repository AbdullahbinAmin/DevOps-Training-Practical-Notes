# Chapter 35 — Project: AWS IAM Management

## Objective
Automate **AWS Identity & Access Management (IAM)** using Terraform driven by a **YAML data file** (`users.yaml`):
1. Read and parse external YAML user configuration using `yamldecode()` and `file()`.
2. Create multiple IAM users dynamically using `for_each` and `toset()`.
3. Generate console login passwords using `aws_iam_user_login_profile` and protect them with `lifecycle.ignore_changes`.
4. Process complex nested data (users with multiple roles) using nested `for` expressions and `flatten()`.
5. Attach AWS managed IAM policies using `aws_iam_user_policy_attachment` with unique key-based `for_each`.

## Prerequisites
- `.env` AWS credentials configured with Administrator access
- Terraform 1.x installed
- Working directory: `tf-aws/aws-iam-management`

## Cost Warning
IAM users, groups, and policies are free in AWS. Minimal or zero cost. Destroy at the end to keep your AWS account clean.

---

## Step 1 — Lab Directory Setup

```bash
source .env
mkdir -p aws-iam-management && cd aws-iam-management
```

## Step 2 — Define User Data in `users.yaml`

Create `users.yaml`:

```yaml
users:
  - username: "raju"
    roles:
      - "AdministratorAccess"
  - username: "shyam"
    roles:
      - "AmazonS3ReadOnlyAccess"
  - username: "baburao"
    roles:
      - "AmazonEC2FullAccess"
      - "AmazonS3ReadOnlyAccess"
```

> [!NOTE]
> `baburao` has two roles. This creates a nested list inside the list of users, which requires restructuring and flattening before Terraform can attach policies without duplicate key errors.

---

## Step 3 — Create Provider Configuration (`main.tf`)

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
  region = "eu-north-1" # Use your active region
}
```

---

## Step 4 — Read and Decode YAML Data with `locals`

Add the `locals` block to `main.tf`:

```hcl
locals {
  # Read YAML file content and decode into Terraform object/list structure
  users_data = yamldecode(file("${path.module}/users.yaml")).users

  # Transform nested list of user-roles into a single flattened list of objects
  user_role_pairs = flatten([
    for user in local.users_data : [
      for role in user.roles : {
        username = user.username
        role     = role
      }
    ]
  ])
}
```

### Visualizing the Data Transformation:
1. **Raw YAML decode:**
   `users_data` = `[ {username="raju", roles=["Admin"]}, {username="baburao", roles=["EC2", "S3"]} ]`
2. **Nested for loops:**
   `[ [{username="raju", role="Admin"}], [{username="baburao", role="EC2"}, {username="baburao", role="S3"}] ]`
3. **`flatten()` result:**
   `[ {username="raju", role="Admin"}, {username="baburao", role="EC2"}, {username="baburao", role="S3"} ]`

---

## Step 5 — Create IAM Users

Add to `main.tf`:

```hcl
# Extract unique usernames into a set for for_each
resource "aws_iam_user" "users" {
  for_each = toset([for u in local.users_data : u.username])
  name     = each.value

  tags = {
    ManagedBy = "Terraform"
  }
}
```

---

## Step 6 — Generate Console Login Passwords

Add to `main.tf`:

```hcl
resource "aws_iam_user_login_profile" "profile" {
  for_each        = aws_iam_user.users
  user            = each.value.name
  password_length = 12

  # CRITICAL: Prevent Terraform from resetting existing passwords on subsequent applies
  lifecycle {
    ignore_changes = [
      password_length,
      password_reset_required,
      pgp_key
    ]
  }
}
```

---

## Step 7 — Attach Managed Policies to IAM Users

Add to `main.tf`:

```hcl
resource "aws_iam_user_policy_attachment" "main" {
  # Convert flattened list into a map with unique keys: "<username>-<role>"
  for_each = {
    for pair in local.user_role_pairs :
    "${pair.username}-${pair.role}" => pair
  }

  user       = aws_iam_user.users[each.value.username].name
  policy_arn = "arn:aws:iam::aws:policy/${each.value.role}"
}
```

---

## Step 8 — Add Outputs (`outputs.tf`)

Create `outputs.tf`:

```hcl
output "created_users" {
  description = "List of created IAM user names"
  value       = keys(aws_iam_user.users)
}

output "user_passwords" {
  description = "Generated initial passwords for IAM users (sensitive)"
  value = {
    for u, p in aws_iam_user_login_profile.profile : u => p.password
  }
  sensitive = false # Set false in training lab to inspect passwords
}
```

---

## Step 9 — Execution & Verification

### 1. Initialize & Plan
```bash
terraform init
terraform plan
```
Check plan output:
- 3 users to create (`raju`, `shyam`, `baburao`)
- 3 login profiles
- 4 policy attachments (`raju-AdministratorAccess`, `shyam-AmazonS3ReadOnlyAccess`, `baburao-AmazonEC2FullAccess`, `baburao-AmazonS3ReadOnlyAccess`)

### 2. Apply Configuration
```bash
terraform apply -auto-approve
```

### 3. Verify in AWS Management Console
1. Open **AWS Console → IAM → Users**.
2. Verify users `raju`, `shyam`, `baburao` are created.
3. Click on `baburao` → **Permissions** tab:
   - Verify both `AmazonEC2FullAccess` and `AmazonS3ReadOnlyAccess` are attached.
4. Click on `shyam` → **Permissions** tab:
   - Verify `AmazonS3ReadOnlyAccess` is attached.

### 4. Test Console Login (Optional Demo)
1. Get the AWS Account ID or Alias:
   `aws sts get-caller-identity --query Account --output text`
2. Login URL format:
   `https://<ACCOUNT_ID>.signin.aws.amazon.com/console`
3. Sign in as user `baburao` with the password from Terraform state/output.
4. Confirm user can view EC2 instances and S3 buckets, but is denied access to other services (e.g., IAM).

---

## Step 10 — Clean Up

```bash
terraform destroy -auto-approve
```
Confirm in AWS IAM console that users and policy attachments are removed.
