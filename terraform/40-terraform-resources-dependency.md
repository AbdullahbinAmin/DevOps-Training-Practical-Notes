# Chapter 40 — Terraform Resource Dependencies: Implicit vs Explicit (`depends_on`)

## Objective
Master how Terraform resolves the **Dependency Graph (DAG)** to determine resource creation order. Compare **Implicit Dependencies** with **Explicit Dependencies** (`depends_on`) and observe how Terraform handles failures.

## Prerequisites
- AWS credentials loaded (`source .env`)
- Region and AMI ID available

## Cost Warning
Creates an EC2 instance and Security Group. Destroy when finished.

---

## 1. Concepts: Implicit vs Explicit Dependencies

| Type | How It Works | Example |
|---|---|---|
| **Default** | Independent resources have no relationship and are created **concurrently in parallel**. | Two unconnected S3 buckets |
| **Implicit** | Resource A directly interpolates an attribute from Resource B. Terraform automatically infers that B must exist before A. | `vpc_security_group_ids = [aws_security_group.web_sg.id]` |
| **Explicit** | No direct attribute reference exists in the code block, but logical order is enforced using the `depends_on` meta-argument. | `depends_on = [aws_security_group.web_sg]` |

---

## Step 1 — Lab Setup

```bash
source .env
mkdir -p tf-dependencies && cd tf-dependencies
```

## Step 2 — Baseline: Observe Default Parallel Execution

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

resource "aws_security_group" "web_sg" {
  name        = "web-server-sg"
  description = "Security group for web server"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web_server" {
  ami           = "ami-0705384c0b33c194c" # Replace with your region AMI
  instance_type = "t3.micro"

  tags = {
    Name = "WebServer"
  }
}
```

Run:
```bash
terraform init
terraform apply -auto-approve
```
> [!NOTE]
> Observe the terminal output: Terraform creates `aws_security_group.web_sg` and `aws_instance.web_server` **at the exact same time in parallel** because there is no link between them.

Destroy before next step:
```bash
terraform destroy -auto-approve
```

---

## Step 3 — Explicit Dependency Using `depends_on`

Add `depends_on` to the EC2 resource in `main.tf`:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0705384c0b33c194c"
  instance_type = "t3.micro"

  # Explicit Dependency: Do NOT start EC2 creation until web_sg is fully ready
  depends_on = [
    aws_security_group.web_sg
  ]

  tags = {
    Name = "WebServer"
  }
}
```

Run:
```bash
terraform apply -auto-approve
```
> [!NOTE]
> Terminal output shows:
> 1. `aws_security_group.web_sg: Creating...`
> 2. `aws_security_group.web_sg: Creation complete after 3s`
> 3. `aws_instance.web_server: Creating...` (Starts ONLY after SG finishes).

---

## Step 4 — Negative Test: How Dependency Prevents Broken Resources

Simulate a syntax error in the security group by changing the protocol to an invalid value (`protocol = "tcpp"`):

```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-server-sg"
  description = "Security group for web server"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcpp" # INVALID PROTOCOL
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Run:
```bash
terraform apply -auto-approve
```
**Result:**
- `aws_security_group.web_sg` fails with: `InvalidParameterValue: Value (tcpp) for parameter protocol is invalid`.
- Because `aws_instance.web_server` has `depends_on = [aws_security_group.web_sg]`, Terraform **aborts** before creating the EC2 instance! No orphan server is created.

Revert `protocol = "tcp"`.

---

## Step 5 — Implicit Dependency (Best Practice)

In real-world architectures, you usually don't need `depends_on` if you pass the resource attribute directly:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0705384c0b33c194c"
  instance_type = "t3.micro"

  # Implicit Dependency: Referencing .id tells Terraform web_sg must be created first
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  tags = {
    Name = "WebServer"
  }
}
```

Terraform automatically establishes the exact same dependency ordering as `depends_on`.

---

## Step 6 — Clean Up

```bash
terraform destroy -auto-approve
```
