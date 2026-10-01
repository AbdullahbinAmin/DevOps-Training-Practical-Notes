# Chapter 43 — Terraform State Modifications & Management

## Objective
Safely inspect, manipulate, and modify the **Terraform State (`terraform.tfstate`)** using CLI subcommands (`list`, `show`, `mv`, `rm`, `pull`, `push`) without triggering accidental resource destruction in the cloud.

## Prerequisites
- AWS credentials loaded (`source .env`)
- Terraform 1.x installed

---

## 1. Why State Commands are Essential

Terraform relies on resource addresses (e.g., `aws_instance.main`) mapped to real cloud IDs (e.g., `i-0123456789abcdef0`).
- If you simply rename `aws_instance.main` to `aws_instance.my_server` in your `.tf` file, Terraform thinks you want to **DELETE** `main` and **CREATE** a brand new `my_server`.
- In production, destroying an active server causes catastrophic downtime.
- State subcommands allow you to update Terraform's internal registry without touching the physical cloud infrastructure.

---

## Step 1 — Lab Setup

```bash
source .env
mkdir -p tf-state-manipulation && cd tf-state-manipulation
```

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
  region = "eu-north-1"
}

resource "aws_security_group" "main" {
  name        = "state-demo-sg"
  description = "State demonstration SG"
}

resource "aws_instance" "main" {
  ami           = "ami-0705384c0b33c194c" # Replace with valid AMI
  instance_type = "t3.micro"

  tags = {
    Name = "OriginalInstance"
  }
}
```

Apply initial infrastructure:
```bash
terraform init
terraform apply -auto-approve
```

---

## Step 2 — Inspecting State (`list` & `show`)

### 1. `terraform state list`
Lists every resource tracked in the active state:
```bash
terraform state list
```
**Output:**
```text
aws_instance.main
aws_security_group.main
```

### 2. `terraform state show`
Inspect all live cloud attributes recorded for a specific resource:
```bash
terraform state show aws_instance.main
```
Shows: instance ID, private IP, public IP, ARN, attached storage, tags, etc.

---

## Step 3 — The Renaming Problem & Safe Refactoring (`terraform state mv`)

Suppose you want to rename `aws_instance.main` to `aws_instance.my_server` for better readability.

### The Problem:
If you only change `main.tf` to `resource "aws_instance" "my_server"` and run `terraform plan`:
```text
Plan: 1 to add, 0 to change, 1 to destroy.
```
Terraform wants to terminate your running production server!

### The Solution: `terraform state mv`

1. Update `main.tf` to the new name:
   ```hcl
   resource "aws_instance" "my_server" {
     ami           = "ami-0705384c0b33c194c"
     instance_type = "t3.micro"

     tags = {
       Name = "OriginalInstance"
     }
   }
   ```
2. Before running apply, tell Terraform about the move in state:
   ```bash
   terraform state mv aws_instance.main aws_instance.my_server
   ```
   **Output:**
   ```text
   Move "aws_instance.main" to "aws_instance.my_server"
   Successfully moved 1 object(s).
   ```
3. Now run `terraform plan`:
   ```bash
   terraform plan
   ```
   **Output:**
   ```text
   No changes. Your infrastructure matches the configuration.
   ```
   Zero downtime, zero destruction!

---

## Step 4 — Untracking a Resource without Deleting It (`terraform state rm`)

Suppose a resource (e.g. `aws_security_group.main`) is being transferred to another team or managed outside Terraform. You want to stop tracking it without destroying it in AWS.

If you delete the code block from `main.tf` and run `terraform apply`, AWS will delete the security group.

### To Untrack Safely:
1. Remove the resource from state:
   ```bash
   terraform state rm aws_security_group.main
   ```
   **Output:**
   ```text
   Removed aws_security_group.main
   Successfully removed 1 resource instance(s).
   ```
2. Verify with `terraform state list`:
   ```bash
   terraform state list
   ```
   Only `aws_instance.my_server` remains.
3. Remove or comment out the `resource "aws_security_group" "main"` block from `main.tf`.
4. Run `terraform plan`:
   ```bash
   terraform plan
   ```
   No changes planned! The security group still exists untouched inside the AWS Console.

---

## Step 5 — Remote State CLI Commands (`pull` and `push`)

When using remote backends (such as AWS S3 or Terraform Cloud):
- **`terraform state pull`**: Dumps the current remote state JSON directly to stdout.
  ```bash
  terraform state pull > state-backup.json
  ```
- **`terraform state push <file>`**: Manually pushes state to remote storage (use with extreme caution, typically only for disaster recovery).

---

## Step 6 — Clean Up

Destroy remaining tracked resources:
```bash
terraform destroy -auto-approve
```
> [!NOTE]
> Remember: Any resource untracked via `terraform state rm` (such as `state-demo-sg`) will NOT be deleted by `terraform destroy`. If you want to remove it, delete it manually in the AWS Console.
