# Chapter 41 — Terraform Resource Lifecycle Customization

## Objective
Control the provisioning and destruction behavior of AWS resources using the `lifecycle` meta-argument:
1. **`create_before_destroy`** — Zero-downtime resource replacement.
2. **`prevent_destroy`** — Guard critical infrastructure against accidental deletion.
3. **`ignore_changes`** — Ignore out-of-band updates or prevent password/attribute overwrite.
4. **`replace_triggered_by`** — Force resource recreation when another resource updates.

## Prerequisites
- AWS credentials loaded (`source .env`)
- Valid AMI ID for your region

---

## 1. Lifecycle Rules Overview

```hcl
resource "aws_instance" "example" {
  # ... configuration ...

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [tags, user_data]
    replace_triggered_by  = [aws_security_group.web_sg.ingress]
  }
}
```

---

## Step 1 — Lab Setup

```bash
source .env
mkdir -p tf-lifecycle && cd tf-lifecycle
```

Create base `main.tf`:

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

resource "aws_security_group" "web_sg" {
  name        = "lifecycle-demo-sg"
  description = "Security group for lifecycle testing"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Step 2 — `create_before_destroy` (Zero Downtime)

### Default Behavior (Downtime):
Without this rule, modifying an attribute that forces recreation (like `ami`) causes Terraform to:
1. **Destroy** the existing instance (website goes down!).
2. **Create** the new instance.

### With `create_before_destroy = true`:
Add the instance block with lifecycle in `main.tf`:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0705384c0b33c194c" # Replace with valid AMI 1
  instance_type = "t3.micro"

  vpc_security_group_ids = [aws_security_group.web_sg.id]

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "WebServer"
  }
}
```

Apply initial state:
```bash
terraform init
terraform apply -auto-approve
```

### Test Replacement:
Update the `ami` attribute to another valid AMI ID in your region (e.g. an Ubuntu or AL2023 AMI). Run `terraform apply`:
```text
aws_instance.web_server: Creating...
aws_instance.web_server: Creation complete after 15s
aws_instance.web_server (deposed): Destroying...
aws_instance.web_server (deposed): Destruction complete after 30s
```
> [!NOTE]
> The new instance is created and verified **first**, before the old instance is terminated.

---

## Step 3 — `prevent_destroy` (Accidental Deletion Protection)

Update the `lifecycle` block in `main.tf`:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0705384c0b33c194c"
  instance_type = "t3.micro"

  lifecycle {
    prevent_destroy = true
  }
}
```

Apply the updated lifecycle rule to state:
```bash
terraform apply -auto-approve
```

### Attempt to Destroy:
```bash
terraform destroy
```

**Output:**
```text
Error: Instance cannot be destroyed

  on main.tf line 20: Resource aws_instance.web_server has lifecycle.prevent_destroy set,
  but the plan calls for this resource to be destroyed.
```
Terraform refuses to destroy the resource!

### How to Safely Remove a Protected Resource:
1. Comment out or set `prevent_destroy = false` in `main.tf`.
2. Run `terraform apply -auto-approve` (updates state).
3. Now run `terraform destroy -auto-approve`.

---

## Step 4 — `ignore_changes` (Preserve External State)

When outside processes (e.g. Auto-scaling, deployment agents, or manual tagging) update an attribute, Terraform would ordinarily revert it.

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0705384c0b33c194c"
  instance_type = "t3.micro"

  tags = {
    Name = "WebServer"
    Tier = "Frontend"
  }

  lifecycle {
    ignore_changes = [
      tags,
      user_data
    ]
  }
}
```
Any changes made to `tags` or `user_data` (either in the AWS console or code) are completely ignored by `terraform plan` and `terraform apply`.

---

## Step 5 — `replace_triggered_by` (Cascade Recreation)

Force a server to be recreated when its dependent configuration updates.

Add `replace_triggered_by` to the instance block:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0705384c0b33c194c"
  instance_type = "t3.micro"

  lifecycle {
    replace_triggered_by = [
      aws_security_group.web_sg.ingress
    ]
  }
}
```

Apply:
```bash
terraform apply -auto-approve
```

### Test Trigger:
Change the security group ingress port in `main.tf` from `80` to `8080`.
Run:
```bash
terraform plan
```
Notice the plan:
```text
~ resource "aws_security_group" "web_sg" { ... }
-/+ resource "aws_instance" "web_server" {
    # forces replacement due to:
    # replace_triggered_by = [aws_security_group.web_sg.ingress]
  }
```
Updating the security group port automatically triggers a full replacement of the EC2 instance!

---

## Step 6 — Clean Up

Remove or comment out `prevent_destroy` before destroying:
```bash
terraform destroy -auto-approve
```
