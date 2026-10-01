# Chapter 33 — Multiple Resources Using `count`

## Objective
Create **multiple instances** of the same resource using `count`, the `element()` function, and `length()` — without duplicating resource blocks.

## Prerequisites
- `.env` credentials in `tf-aws`
- Region code and AMI ID from your region
- Previous chapters (variables, functions) completed

## Cost Warning
This chapter creates EC2 instances. Run `terraform destroy` when done.

## Concept Summary
- `count = N` → Terraform creates N copies of that resource.
- Each copy is accessed by `count.index` (0, 1, 2, ...).
- Use `element(list, index)` to pick values from a list by index.
- Use `length(list)` to dynamically set count from a list.

---

## Step 1 — Create the lab folder

```bash
source .env
mkdir tf-count-demo && cd tf-count-demo
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

## Step 3 — Hardcoded `count` — create 3 instances

Add to `main.tf`:

```hcl
resource "aws_instance" "my_server" {
  count         = 3
  ami           = "<YOUR_AMI_ID>"
  instance_type = "t3.micro"

  tags = {
    Name = "Server-${count.index}"
  }
}
```

This will create:
- `Server-0`
- `Server-1`
- `Server-2`

## Step 4 — Init, Plan, Apply

```bash
terraform init
terraform plan
```

Expected: `Plan: 3 to add`

```bash
terraform apply -auto-approve
```

Verify in AWS Console → EC2 → Instances → three instances named `Server-0`, `Server-1`, `Server-2`.

## Step 5 — Destroy and improve with a list

```bash
terraform destroy -auto-approve
```

Now use a **variable list** for instance names:

## Step 6 — Create `variables.tf`

```hcl
variable "instance_names" {
  type    = list(string)
  default = ["web-server", "app-server", "db-server"]
}
```

## Step 7 — Update `main.tf` to use `length()` and `element()`

Replace the resource block:

```hcl
resource "aws_instance" "my_server" {
  count         = length(var.instance_names)
  ami           = "<YOUR_AMI_ID>"
  instance_type = "t3.micro"

  tags = {
    Name = element(var.instance_names, count.index)
  }
}
```

- `length(var.instance_names)` → returns `3` (dynamic count).
- `element(var.instance_names, count.index)` → picks name by index.

## Step 8 — Plan and Apply

```bash
terraform plan
```

Expected: `Plan: 3 to add` — names will be `web-server`, `app-server`, `db-server`.

```bash
terraform apply -auto-approve
```

Verify in AWS Console → three instances with correct names.

## Step 9 — Access individual resources by index

In `outputs.tf`:

```hcl
output "first_instance_id" {
  value = aws_instance.my_server[0].id
}

output "all_instance_ids" {
  value = aws_instance.my_server[*].id
}

output "all_instance_names" {
  value = [for instance in aws_instance.my_server : instance.tags["Name"]]
}
```

Run:

```bash
terraform plan
```

You will see the instance IDs in the output.

## Step 10 — Problem with `count` (understanding the limitation)

**Problem**: If you remove `"app-server"` from the middle of the list:

```hcl
variable "instance_names" {
  default = ["web-server", "db-server"]
}
```

Terraform will:
1. Keep index 0 (`web-server`) ✅
2. **Rename** index 1 from `app-server` to `db-server` (destroy + recreate) ❌
3. **Destroy** index 2 (old `db-server`) ❌

This happens because `count` tracks resources **by index number**. Removing an item shifts all subsequent indices.

**Verify** — run `terraform plan` with the shortened list and observe the destroy/create plan.

> ⚠️ This is why `for_each` (next chapter) is preferred for named resources.

## Cleanup

```bash
terraform destroy -auto-approve
cd ..
rm -rf tf-count-demo
```

## Key Takeaways
- `count` creates multiple copies of a resource, tracked by **index** (0, 1, 2...).
- `element(list, index)` picks values from a list; `length(list)` sets count dynamically.
- `resource_name[*].attribute` (splat) returns all values as a list.
- **Limitation**: Removing an item from the middle of the list causes unintended destroy/recreate of other resources.
- Use `count` for **identical resources** (same config). Use `for_each` when each resource needs **unique configuration**.
