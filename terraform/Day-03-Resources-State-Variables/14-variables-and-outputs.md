# Day 03 — Terraform Variables and Outputs

> Replace hardcoded values with input variables and surface useful results (like the EC2 public IP) using outputs.

## Learning Objectives
- Declare input variables in `variables.tf` with `description`, `type`, and `default`.
- Reference variables with `var.<name>` and keep configs DRY.
- Override variables using `-var`, a `.tfvars` file, and `TF_VAR_` environment variables.
- Declare outputs in `outputs.tf` and read them after apply.
- Query outputs for scripting with `terraform output`.

## Prerequisites
- Terraform 1.x and AWS credentials configured outside the code.
- A working folder from the previous topics (or a fresh empty folder).
- A valid AMI ID for your chosen region.

## Concept (plain English, beginner friendly)

**Variables** exist because the same value keeps showing up — region, AMI, instance type. Hardcoding it in five places means five edits and one you will forget. Declare it once as a variable, reference it as `var.region`, and change one line to change everything. That is the DRY principle: Don't Repeat Yourself. Variables also make a config reusable across dev, staging, and prod without editing the code — you just feed different values in.

A variable declaration has three parts worth setting: `description` (so the next person knows what it is for), `type` (so Terraform rejects wrong input), and `default` (optional — omit it to force the caller to supply a value).

**Outputs** are the other direction. After `apply`, you often need a fact that only exists once the resource is real: the public IP, a DNS name, a bucket name, an ARN. An `output` block names that value so Terraform prints it after apply and makes it machine-readable for scripts and CI pipelines.

## Step-by-Step Practical

1. Create `variables.tf`:

```hcl
variable "region" {
  description = "AWS region to use"
  type        = string
  default     = "us-east-1"
}

variable "ami_id" {
  description = "AMI ID for the EC2 instance (region specific)"
  type        = string
  default     = "ami-0abcd1234abcdef0"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "instance_name" {
  description = "Value of the Name tag"
  type        = string
  default     = "Sample-Server"
}

variable "tags" {
  description = "Extra tags applied to all resources"
  type        = map(string)
  default = {
    Environment = "training"
    ManagedBy   = "terraform"
  }
}
```

2. Create `main.tf` that uses those variables instead of literals:

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
  region = var.region # using variable
}

resource "aws_instance" "my_server" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = merge(
    var.tags,
    { Name = var.instance_name }
  )
}
```

3. Create `outputs.tf`:

```hcl
output "instance_public_ip" {
  description = "Public IP of EC2 instance"
  value       = aws_instance.my_server.public_ip
}

output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.my_server.id
}

output "instance_public_dns" {
  description = "Public DNS name of the EC2 instance"
  value       = aws_instance.my_server.public_dns
}
```

4. Run the standard sequence:

```bash
terraform init
terraform validate
terraform plan
terraform apply
```

5. Override a variable at runtime with `-var`:

```bash
terraform apply -var="region=us-west-2"
```

6. Override several values with a `.tfvars` file. Create `env.tfvars`:

```hcl
region        = "us-west-2"
instance_type = "t3.small"
instance_name = "Dev-Server"
ami_id        = "ami-0fedcba9876543210"
```

Then apply with it:

```bash
terraform apply -var-file="env.tfvars"
```

7. Or use environment variables (handy in CI):

```bash
export TF_VAR_region="eu-west-1"
export TF_VAR_instance_type="t3.nano"
terraform plan
```

8. Read outputs after apply:

```bash
terraform output
terraform output instance_public_ip
terraform output -raw instance_public_ip
terraform output -json
```

9. Use an output in a script:

```bash
IP=$(terraform output -raw instance_public_ip)
echo "Instance is reachable at $IP"
```

Note: a file named `terraform.tfvars` or `*.auto.tfvars` is loaded automatically without any flag. Precedence, lowest to highest: `default` → environment `TF_VAR_*` → `terraform.tfvars` → `*.auto.tfvars` → `-var-file` → `-var`.

## Expected Output

Plan (trimmed):

```text
Terraform will perform the following actions:

  # aws_instance.my_server will be created
  + resource "aws_instance" "my_server" {
      + ami           = "ami-0abcd1234abcdef0"
      + instance_type = "t3.micro"
      + public_ip     = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

After apply:

```text
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

instance_id = "i-01234abcd"
instance_public_dns = "ec2-178-141-12-45.compute-1.amazonaws.com"
instance_public_ip = "178.141.12.45"
```

## Verification
- `terraform output instance_public_ip` returns the same IP shown after apply.
- Confirm against AWS:

```bash
aws ec2 describe-instances \
  --instance-ids "$(terraform output -raw instance_id)" \
  --query "Reservations[].Instances[].PublicIpAddress" --output text
```

- AWS Console: **EC2 Dashboard → Instances → Sample-Server → Instance details** shows the same **Public IPv4 address**.
- Confirm a variable override took effect:

```bash
terraform console
> var.instance_type
```

## Cleanup

```bash
terraform destroy
```

If you applied with a var file, destroy with the same values so Terraform targets the right region:

```bash
terraform destroy -var-file="env.tfvars"
```

`.tfvars` files often hold environment-specific and sensitive values — do not commit them, and never put access keys in them. Commit a `terraform.tfvars.example` with placeholder values instead. Also keep state out of Git:

```text
*.tfstate
*.tfstate.*
.terraform/
*.tfvars
!terraform.tfvars.example
```

Mark sensitive outputs so they are not printed:

```hcl
output "db_password" {
  description = "Generated DB password"
  value       = random_password.db.result
  sensitive   = true
}
```

Sensitive values are hidden in CLI output but still stored in plain text in the state file — one more reason state must never be committed.

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `No value for required variable` | Variable declared without a `default` and none supplied | Add a `default`, or pass `-var`, `-var-file`, or `TF_VAR_<name>` |
| `Reference to undeclared input variable` | `var.foo` used but no `variable "foo"` block exists | Declare it in `variables.tf` (check spelling) |
| `Invalid value for input variable: string required` | Supplied value does not match the declared `type` | Fix the value or widen the type (e.g. `map(string)`, `list(string)`) |
| `Unsupported attribute: This object has no argument named ...` | Wrong attribute name in an output | Check the provider docs; use `terraform state show` to list real attributes |
| Output is empty or `null` | Instance has no public IP (private subnet, or `associate_public_ip_address = false`) | Output `private_ip`, or place the instance in a public subnet with auto-assign IP |
| `Output refers to sensitive values` | Output derives from a sensitive input without being marked | Add `sensitive = true` to the output |
| Overrides seem ignored | Precedence, or the file is not auto-loaded | Rename to `terraform.tfvars` / use `-var-file` explicitly; `-var` always wins |
| `InvalidAMIID.NotFound` after changing region | AMI IDs are region-specific | Supply a matching AMI for the new region, or use an `aws_ami` data source |

## Key Takeaways
- Variables keep configs DRY, reusable, and easy to maintain — change one place, applies everywhere.
- Reference variables with `var.<name>`; give every variable a `description` and `type`.
- Override with `-var`, `-var-file="env.tfvars"`, or `TF_VAR_*`; `-var` has the highest precedence.
- Outputs expose post-apply facts like public IP, DNS, and IDs for humans and scripts.
- Use meaningful names, mark secrets `sensitive = true`, and never commit `.tfvars` or `.tfstate`.
- Standard flow stays the same: `init → validate → plan → apply`.

## Next: [Day 04 — Terraform Provisioners and Remote Backends](../Day-04-Provisioners-and-Backends/README.md)
