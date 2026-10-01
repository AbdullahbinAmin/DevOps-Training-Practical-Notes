# Chapter 45 — Terraform Workspaces: Multi-Environment Management

## Objective
Manage multiple distinct deployment environments (**dev**, **stage**, **prod**) from a single codebase using **Terraform Workspaces**, isolate their state files, and leverage the built-in **`terraform.workspace`** variable.

## Prerequisites
- AWS credentials loaded (`source .env`)
- Terraform 1.x installed

---

## 1. What are Terraform Workspaces?

- **Without Workspaces**: Teams often copy-paste the entire Terraform folder into `infra-dev/`, `infra-stage/`, `infra-prod/`. This leads to code drift and maintenance nightmares.
- **With Workspaces**: A single set of `.tf` files manages multiple parallel environments.
- **State Isolation**:
  - `default` workspace state: `terraform.tfstate`
  - Custom workspaces state: `terraform.tfstate.d/<workspace_name>/terraform.tfstate`

---

## Step 1 — Lab Setup

```bash
source .env
mkdir -p tf-workspaces && cd tf-workspaces
```

Create `main.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = "eu-north-1"
}

resource "random_id" "bucket_suffix" {
  byte_length = 4
}

# The bucket name dynamically embeds the active workspace name
resource "aws_s3_bucket" "env_bucket" {
  bucket = "company-app-${terraform.workspace}-${random_id.bucket_suffix.hex}"

  tags = {
    Environment = terraform.workspace
    ManagedBy   = "Terraform"
  }
}

output "workspace_bucket_name" {
  value = aws_s3_bucket.env_bucket.bucket
}

output "active_workspace" {
  value = terraform.workspace
}
```

Initialize:
```bash
terraform init
```

---

## Step 2 — Workspace CLI Basics

### 1. List Workspaces
```bash
terraform workspace list
```
**Output:**
```text
* default
```
The asterisk (`*`) indicates the currently active workspace.

### 2. Show Active Workspace
```bash
terraform workspace show
```
**Output:**
```text
default
```

---

## Step 3 — Create and Deploy to `dev` Workspace

### 1. Create `dev` Workspace
```bash
terraform workspace new dev
```
**Output:**
```text
Created and switched to workspace "dev"!

You're now on a new, empty workspace. Workspaces isolate their state,
so if you run "terraform plan" Terraform will not see any existing state
for this configuration.
```

### 2. Apply to `dev`
```bash
terraform apply -auto-approve
```
Notice the output bucket name:
```text
workspace_bucket_name = "company-app-dev-3a1b8c9d"
active_workspace      = "dev"
```

---

## Step 4 — Create and Deploy to `prod` Workspace

### 1. Create `prod` Workspace
```bash
terraform workspace new prod
```

### 2. Apply to `prod`
```bash
terraform apply -auto-approve
```
Notice the output bucket name:
```text
workspace_bucket_name = "company-app-prod-7f2e4a1c"
active_workspace      = "prod"
```

### 3. Verify in AWS Console
Open **AWS Console → S3**:
- Both `company-app-dev-...` and `company-app-prod-...` exist simultaneously, created from the exact same code!

### 4. Inspect Local State Structure
```bash
ls -la terraform.tfstate.d/
```
Output:
```text
terraform.tfstate.d/
├── dev/
│   └── terraform.tfstate
└── prod/
    └── terraform.tfstate
```

---

## Step 5 — Switching Workspaces (`select`)

Switch back to `dev`:
```bash
terraform workspace select dev
```

Run `terraform state list`:
```text
aws_s3_bucket.env_bucket
random_id.bucket_suffix
```
Terraform immediately loads only the resources created in the `dev` workspace!

---

## Step 6 — Deleting Workspaces Safely

Terraform protects you: you **cannot** delete a workspace that still contains active cloud resources, and you cannot delete the currently selected workspace.

### 1. Destroy resources in `dev`:
```bash
terraform workspace select dev
terraform destroy -auto-approve
```

### 2. Destroy resources in `prod`:
```bash
terraform workspace select prod
terraform destroy -auto-approve
```

### 3. Switch to `default`:
```bash
terraform workspace select default
```

### 4. Delete the empty workspaces:
```bash
terraform workspace delete dev
terraform workspace delete prod
```

Verify with `terraform workspace list`:
```text
* default
```
Only the `default` workspace remains.
