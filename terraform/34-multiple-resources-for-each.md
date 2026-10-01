# Chapter 34 — Multiple Resources Using `for_each`

## Objective
Create multiple resources using `for_each` with **maps** — each resource tracked by a **unique key** instead of an index number. This avoids the index-shift problem of `count`.

## Prerequisites
- `.env` credentials in `tf-aws`
- Region code and AMI ID from your region
- Previous chapter (`count`) completed

## Cost Warning
This chapter creates EC2 instances. Run `terraform destroy` when done.

## Concept Summary
- `for_each` accepts a **map** or a **set of strings**.
- Each resource is tracked by its **key** (not index).
- Inside the resource: `each.key` and `each.value` give you the current item.
- Removing an item from the middle does **not** affect other resources.

---

## Step 1 — Create the lab folder

```bash
source .env
mkdir tf-foreach-demo && cd tf-foreach-demo
```

## Step 2 — Create `main.tf` with provider

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
  region = "<YOUR_REGION>"
}
```

## Step 3 — Define a map variable in `variables.tf`

```hcl
variable "instances" {
  type = map(object({
    ami           = string
    instance_type = string
  }))

  default = {
    "web-server" = {
      ami           = "<YOUR_AMI_ID>"
      instance_type = "t3.micro"
    }
    "app-server" = {
      ami           = "<YOUR_AMI_ID>"
      instance_type = "t3.micro"
    }
    "db-server" = {
      ami           = "<YOUR_AMI_ID>"
      instance_type = "t3.micro"
    }
  }
}
```

## Step 4 — Create resource with `for_each`

Add to `main.tf`:

```hcl
resource "aws_instance" "my_server" {
  for_each      = var.instances

  ami           = each.value.ami
  instance_type = each.value.instance_type

  tags = {
    Name = each.key
  }
}
```

- `each.key` → `"web-server"`, `"app-server"`, `"db-server"`
- `each.value` → the object `{ ami = "...", instance_type = "..." }`

## Step 5 — Init, Plan, Apply

```bash
terraform init
terraform plan
```

Expected: `Plan: 3 to add` — you will see resource names like:
- `aws_instance.my_server["web-server"]`
- `aws_instance.my_server["app-server"]`
- `aws_instance.my_server["db-server"]`

```bash
terraform apply -auto-approve
```

Verify in AWS Console → three instances with correct names.

## Step 6 — Outputs

Create `outputs.tf`:

```hcl
output "instance_ids" {
  value = { for k, v in aws_instance.my_server : k => v.id }
}

output "instance_public_ips" {
  value = { for k, v in aws_instance.my_server : k => v.public_ip }
}
```

Run:

```bash
terraform plan
```

You will see a map of `name → instance_id` and `name → public_ip`.

## Step 7 — Remove middle item (compare with `count`)

Remove `"app-server"` from `variables.tf`:

```hcl
variable "instances" {
  type = map(object({
    ami           = string
    instance_type = string
  }))

  default = {
    "web-server" = {
      ami           = "<YOUR_AMI_ID>"
      instance_type = "t3.micro"
    }
    "db-server" = {
      ami           = "<YOUR_AMI_ID>"
      instance_type = "t3.micro"
    }
  }
}
```

Run:

```bash
terraform plan
```

Expected:
- `Plan: 0 to add, 0 to change, 1 to destroy`
- Only `app-server` is destroyed ✅
- `web-server` and `db-server` are untouched ✅

> 🎯 This is the key advantage of `for_each` over `count`. Resources are tracked by **key**, not by index.

## Step 8 — Using `for_each` with a simple set of strings

Alternative: if all instances use the same config, you can use a **set**:

```hcl
variable "server_names" {
  type    = set(string)
  default = ["web-server", "app-server", "db-server"]
}

resource "aws_instance" "my_server" {
  for_each      = var.server_names

  ami           = "<YOUR_AMI_ID>"
  instance_type = "t3.micro"

  tags = {
    Name = each.key    # each.key == each.value for sets
  }
}
```

## Step 9 — Using `for_each` with `toset()` conversion

If your variable is a `list`, convert it:

```hcl
variable "server_names" {
  type    = list(string)
  default = ["web-server", "app-server", "db-server"]
}

resource "aws_instance" "my_server" {
  for_each      = toset(var.server_names)

  ami           = "<YOUR_AMI_ID>"
  instance_type = "t3.micro"

  tags = {
    Name = each.key
  }
}
```

## Cleanup

```bash
terraform destroy -auto-approve
cd ..
rm -rf tf-foreach-demo
```

## `count` vs `for_each` — Quick Comparison

| Feature | `count` | `for_each` |
|---------|---------|------------|
| Tracking | By index (0, 1, 2...) | By key name |
| Remove middle item | Shifts all indices ❌ | Only removes that item ✅ |
| Access | `resource[0]`, `resource[1]` | `resource["key"]` |
| Input type | Number | Map or Set |
| Best for | Identical resources | Resources with unique configs |

## Key Takeaways
- `for_each` tracks resources by **key** — safe to add/remove items from the middle.
- Use a **map** when each resource needs different configuration.
- Use a **set** (or `toset()`) when all resources share the same config but need unique names.
- `each.key` = the map key (or set value), `each.value` = the map value (or set value).
- **Prefer `for_each` over `count`** for production — it is more predictable and safer.
