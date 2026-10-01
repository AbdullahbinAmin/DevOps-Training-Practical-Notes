# Chapter 42 — Resource Validations: `precondition` and `postcondition`

## Objective
Implement advanced resource-level validation rules using **`precondition`** and **`postcondition`** blocks inside the `lifecycle` meta-argument to guarantee infrastructure invariants and catch misconfigurations with descriptive errors.

## Prerequisites
- AWS credentials loaded (`source .env`)
- Valid AMI ID for your region

---

## 1. Precondition vs Postcondition

| Feature | `precondition` | `postcondition` |
|---|---|---|
| **When it runs** | **Before** creating or updating the resource | **After** creating or updating the resource |
| **Typical Target** | Dependencies, inputs, and external resources | Newly computed attributes of the resource itself via `self` |
| **Why not `depends_on`?** | `depends_on` only ensures execution order; `precondition` checks the **actual state value** of that resource. |

---

## Step 1 — Lab Setup

```bash
source .env
mkdir -p tf-validations && cd tf-validations
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
  name        = "validation-test-sg"
  description = "Security group for validation tests"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Step 2 — Implementing a `precondition`

Add the EC2 instance resource with a `precondition` that ensures the security group ID exists and is non-empty before starting EC2 creation:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0705384c0b33c194c"
  instance_type = "t3.micro"

  vpc_security_group_ids = [aws_security_group.web_sg.id]

  lifecycle {
    precondition {
      condition     = aws_security_group.web_sg.id != ""
      error_message = "CRITICAL: Security group ID must not be blank before creating EC2 instance."
    }
  }

  tags = {
    Name = "PreconditionServer"
  }
}
```

Run:
```bash
terraform init
terraform apply -auto-approve
```
> [!NOTE]
> Terraform verifies the precondition condition evaluates to `true` before provisioning `aws_instance.web_server`.

---

## Step 3 — Implementing a `postcondition`

A `postcondition` checks attributes **after** the resource is provisioned, referencing the resource via `self`.

Let's test an invariant: **"Every web server must have a public IP address assigned."**

Update `aws_instance.web_server` in `main.tf`:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0705384c0b33c194c"
  instance_type = "t3.micro"

  vpc_security_group_ids = [aws_security_group.web_sg.id]

  lifecycle {
    # Precondition: Verified before creation
    precondition {
      condition     = aws_security_group.web_sg.id != ""
      error_message = "CRITICAL: Security group ID must not be blank."
    }

    # Postcondition: Verified after creation
    postcondition {
      condition     = self.public_ip != ""
      error_message = "VALIDATION FAILED: EC2 instance was created without a public IP address!"
    }
  }

  tags = {
    Name = "PostconditionServer"
  }
}
```

Apply:
```bash
terraform apply -auto-approve
```
Because instances created in the default VPC receive a public IP, the postcondition passes successfully.

---

## Step 4 — Testing a Postcondition Failure

Let's intentionally fail the postcondition by forcing the instance to launch without a public IP in a custom private subnet:

Add private VPC and subnet to `main.tf`:

```hcl
resource "aws_vpc" "private_vpc" {
  cidr_block = "10.0.0.0/16"
  tags       = { Name = "validation-vpc" }
}

resource "aws_subnet" "private_sub" {
  vpc_id            = aws_vpc.private_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "eu-north-1a"
}

# Update instance to use private subnet without public IP
resource "aws_instance" "web_server" {
  ami           = "ami-0705384c0b33c194c"
  instance_type = "t3.micro"

  subnet_id                   = aws_subnet.private_sub.id
  associate_public_ip_address = false # EXPLICITLY DISABLE PUBLIC IP

  lifecycle {
    postcondition {
      condition     = self.public_ip != ""
      error_message = "VALIDATION FAILED: EC2 instance was created without a public IP address!"
    }
  }
}
```

Run:
```bash
terraform apply -auto-approve
```

### Observe Failure:
```text
Error: Resource postcondition failed

  on main.tf line 35, in resource "aws_instance" "web_server":
  35:       condition     = self.public_ip != ""
    ├ self.public_ip is ""

VALIDATION FAILED: EC2 instance was created without a public IP address!
```
Terraform flags the resource as tainted/failed and halts the pipeline immediately with your customized error message.

---

## Step 5 — Clean Up

```bash
terraform destroy -auto-approve
```
