# Day 10 — Project: AWS IAM Management

> Drive IAM users, login profiles and policy attachments from a single data file using `for_each`, `yamldecode()` and `flatten()`.

## Learning Objectives
- Model users and their roles/policies as external data (YAML or a Terraform variable map).
- Use `yamldecode()`, `flatten()` and `for` expressions to reshape data.
- Create `aws_iam_user`, `aws_iam_group`, `aws_iam_user_login_profile` and `aws_iam_user_policy_attachment` with `for_each`.
- Understand why `for_each` needs unique map keys, and how `each.key` / `each.value` behave.
- Apply least privilege and keep secrets out of code — and know that state holds sensitive values.

## Prerequisites
- Terraform 1.x installed (`terraform version`).
- AWS provider 5.x, credentials via `aws configure`, env vars, or an IAM role — never in `.tf` files.
- IAM permissions to manage users, groups and policy attachments.
- An empty working folder, e.g. `tf-iam-project/`.

## Concept (plain English)
IAM = Identity and Access Management. Instead of writing one `resource` block per user, you keep the
*list of people* in data (`users.yaml` or a variable map) and let Terraform loop over it.

The flow from the slide:

```
users.yaml  ->  yamldecode()  ->  flatten()  ->  resources (users, logins, attachments)  ->  AWS IAM
 (data)          read/decode      normalize        create                                     manage
```

Each user can have one *or more* policies, so the raw data is a list of objects where `roles` is
itself a list. `flatten()` turns that nested shape into a flat list of `{ username, role }` pairs.
Because `for_each` needs **unique keys**, we build the key as `"${username}-${role}"`.

Remember:
- `for_each` works with a map or a set (not a list of objects directly).
- `each.key` is the map key (here `username-role`).
- `each.value` is the object (`{ username = ..., role = ... }`).
- Groups are usually the better production pattern: attach policies to a group, put users in it.

## Step-by-Step Practical

1. Create the project folder and the data file.

```bash
mkdir -p tf-iam-project && cd tf-iam-project
```

2. `users.yaml` — the only file you edit to onboard/offboard people.

```yaml
users:
  - username: raju
    roles:
      - AmazonEC2FullAccess
  - username: shyam
    roles:
      - AmazonS3ReadOnlyAccess
  - username: baburao
    roles:
      - AmazonS3ReadOnlyAccess
      - AmazonEC2FullAccess
```

3. `versions.tf` — pin Terraform and the provider.

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
```

4. `provider.tf` — region only. No keys, ever.

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      ManagedBy = "Terraform"
      Project   = "iam-management"
    }
  }
}
```

5. `variables.tf` — region, path prefix, and an optional variable-map alternative to YAML.

```hcl
variable "aws_region" {
  description = "AWS region for the provider."
  type        = string
  default     = "us-east-1"
}

variable "users_file" {
  description = "Path to the YAML file describing users and their managed policies."
  type        = string
  default     = "users.yaml"
}

# Alternative to YAML: define the same data directly as a map.
# Set users_file = "" and populate this to use it instead.
variable "users_map" {
  description = "Map of username => list of AWS managed policy names."
  type        = map(list(string))
  default     = {}
}

variable "create_login_profiles" {
  description = "Create console passwords. Passwords land in Terraform state."
  type        = bool
  default     = true
}

variable "pgp_key" {
  description = "Optional base64 PGP key or keybase:username to encrypt generated passwords."
  type        = string
  default     = null
}
```

6. `locals.tf` — read, decode, flatten, key.

```hcl
locals {
  # Read the YAML file only if a path was provided.
  raw_users = var.users_file != "" ? file(var.users_file) : ""

  # List of objects: [{ username = "raju", roles = [...] }, ...]
  users_from_yaml = var.users_file != "" ? yamldecode(local.raw_users).users : []

  # Same shape, but built from the variable map (fallback path).
  users_from_map = [
    for name, roles in var.users_map : {
      username = name
      roles    = roles
    }
  ]

  users_data = length(local.users_from_yaml) > 0 ? local.users_from_yaml : local.users_from_map

  # username => object, used for aws_iam_user
  users = {
    for u in local.users_data : u.username => u
  }

  # Flatten nested roles into one pair per (user, policy)
  user_role_pairs = flatten([
    for user in local.users_data : [
      for r in user.roles : {
        username = user.username
        role     = r
      }
    ]
  ])

  # "username-role" => pair  (unique keys required by for_each)
  user_role_map = {
    for p in local.user_role_pairs : "${p.username}-${p.role}" => p
  }
}
```

7. `main.tf` — the resources.

```hcl
# 7A. IAM users
resource "aws_iam_user" "users" {
  for_each = local.users

  name          = each.value.username
  path          = "/training/"
  force_destroy = true # also removes keys/profiles so destroy succeeds

  tags = {
    Name = each.value.username
  }
}

# 7B. Console login profile (password) per user
resource "aws_iam_user_login_profile" "login" {
  for_each = var.create_login_profiles ? aws_iam_user.users : {}

  user                    = each.value.name
  password_length         = 20
  password_reset_required = true
  pgp_key                 = var.pgp_key

  lifecycle {
    # Password is generated once; do not churn it on every plan.
    ignore_changes = [password_length, password_reset_required, pgp_key]
  }
}

# 7C. Attach AWS managed policies, one attachment per (user, policy) pair
resource "aws_iam_user_policy_attachment" "attach" {
  for_each = local.user_role_map

  user       = aws_iam_user.users[each.value.username].name
  policy_arn = "arn:aws:iam::aws:policy/${each.value.role}"
}
```

8. Optional but recommended — group-based least privilege instead of per-user attachments.

```hcl
resource "aws_iam_group" "readonly" {
  name = "training-readonly"
  path = "/training/"
}

resource "aws_iam_group_policy_attachment" "readonly" {
  group      = aws_iam_group.readonly.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}

# Custom least-privilege inline policy: read one bucket only.
data "aws_iam_policy_document" "bucket_read" {
  statement {
    sid       = "ReadOneBucket"
    effect    = "Allow"
    actions   = ["s3:GetObject", "s3:ListBucket"]
    resources = [
      "arn:aws:s3:::nexskill-training-bucket",
      "arn:aws:s3:::nexskill-training-bucket/*",
    ]
  }
}

resource "aws_iam_policy" "bucket_read" {
  name        = "training-bucket-read"
  description = "Least-privilege read access to one training bucket."
  policy      = data.aws_iam_policy_document.bucket_read.json
}
```

The rendered JSON looks like this — useful to know when reviewing policies:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOneBucket",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::nexskill-training-bucket",
        "arn:aws:s3:::nexskill-training-bucket/*"
      ]
    }
  ]
}
```

9. `outputs.tf` — expose non-secret data; mark secrets sensitive.

```hcl
output "user_names" {
  description = "Created IAM user names."
  value       = [for u in aws_iam_user.users : u.name]
}

output "user_arns" {
  description = "Created IAM user ARNs."
  value       = { for k, u in aws_iam_user.users : k => u.arn }
}

output "policy_attachment_keys" {
  description = "The username-role keys used by for_each."
  value       = keys(local.user_role_map)
}

output "console_passwords" {
  description = "Generated console passwords. Sensitive; also stored in state."
  value       = { for k, p in aws_iam_user_login_profile.login : k => p.password }
  sensitive   = true
}
```

10. Run it.

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```

11. Do NOT do this unless you truly need programmatic keys — and understand the consequence.

```hcl
# Creating access keys in Terraform writes the SECRET into terraform.tfstate in plaintext.
# Prefer IAM roles / short-lived credentials / AWS Identity Center.
# resource "aws_iam_access_key" "user" {
#   for_each = aws_iam_user.users
#   user     = each.value.name
# }
```

## Expected Output

```
Plan: 9 to add, 0 to change, 0 to destroy.
...
aws_iam_user.users["raju"]: Creation complete after 1s [id=raju]
aws_iam_user.users["shyam"]: Creation complete after 1s [id=shyam]
aws_iam_user.users["baburao"]: Creation complete after 1s [id=baburao]
aws_iam_user_login_profile.login["raju"]: Creation complete after 2s
aws_iam_user_policy_attachment.attach["baburao-AmazonEC2FullAccess"]: Creation complete after 1s
aws_iam_user_policy_attachment.attach["baburao-AmazonS3ReadOnlyAccess"]: Creation complete after 1s
...
Apply complete! Resources: 9 added, 0 changed, 0 destroyed.

Outputs:
console_passwords = <sensitive>
user_names = ["baburao", "raju", "shyam"]
```

Resulting access:

| User | Policies |
| --- | --- |
| raju | AmazonEC2FullAccess |
| shyam | AmazonS3ReadOnlyAccess |
| baburao | AmazonS3ReadOnlyAccess, AmazonEC2FullAccess |

## Verification

```bash
terraform state list
terraform output user_names

aws iam list-users --path-prefix /training/ --query 'Users[].UserName'
aws iam list-attached-user-policies --user-name baburao
aws iam get-login-profile --user-name baburao
```

Console check: IAM -> Users -> pick a user -> Permissions tab shows the attached managed policies.

Optional login test: sign in as `baburao` with the generated password (`terraform output -json console_passwords`),
change the password when prompted, then confirm EC2 and S3 read work while unrelated actions return
`AccessDenied`. That failure is the proof least privilege is in effect.

## Cleanup

```bash
terraform destroy
```

Then verify nothing is left:

```bash
aws iam list-users --path-prefix /training/
```

Because state contains passwords, delete or securely store `terraform.tfstate*` after destroy,
and never commit it. Add to `.gitignore`:

```bash
printf '%s\n' '*.tfstate' '*.tfstate.*' '.terraform/' '*.tfvars' > .gitignore
```

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `The given "for_each" argument value is unsuitable... must be a map, or set of strings` | Passing a list of objects to `for_each` | Convert to a map with a `for` expression: `{ for p in list : "${p.username}-${p.role}" => p }` |
| `Duplicate object key` in the `for` expression | Two pairs produced the same key (same user + same policy twice) | Deduplicate the YAML, or use `distinct()` on the roles list |
| `EntityAlreadyExists: User with name raju already exists` | User exists in AWS but not in state | `terraform import 'aws_iam_user.users["raju"]' raju` or delete the manual user |
| `DeleteConflict: must delete policies/login profile first` on destroy | User has attachments/keys created outside Terraform | Set `force_destroy = true` on `aws_iam_user`, re-apply, then destroy |
| `NoSuchEntity: Policy arn:aws:iam::aws:policy/AmazonS3ReadOnly does not exist` | Typo in managed policy name | Use exact names, e.g. `AmazonS3ReadOnlyAccess`; verify with `aws iam list-policies --scope AWS` |
| `Error: Output refers to sensitive values` | Returning a password without marking it | Add `sensitive = true` to the output |
| `Invalid function argument` on `yamldecode` | YAML indentation broken, or file path wrong | Validate the YAML; the path is relative to the root module — use `${path.module}/users.yaml` if needed |
| `AccessDenied` when applying | Your own credentials lack IAM write permissions | Use an admin/IAM-admin principal for the lab |

## Key Takeaways
- Keep people data outside the code; `yamldecode()` + `flatten()` + `for_each` turns data into infrastructure.
- `for_each` requires unique keys — compose them (`username-role`) when one entity has many relations.
- `each.key` is the key, `each.value` is the object; keys appear in resource addresses, so keep them stable.
- Prefer groups and scoped custom policies over broad managed policies; least privilege is the default posture.
- Never hardcode credentials. Creating `aws_iam_access_key` (or login profiles) stores secrets in plaintext state — protect state with a remote backend + encryption, or avoid creating keys at all.
- Destroy lab resources when finished; idle IAM entities are an attack surface, not just clutter.

## Next: Terraform Modules — packaging this configuration for reuse.
