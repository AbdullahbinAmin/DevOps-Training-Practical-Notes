# Chapter 31 — Terraform Operators & Expressions: List, Map, Object, Loops

## Objective
Use **operators**, **expressions**, **for loops**, and the **conditional (ternary) operator** inside `locals` and `outputs` — all verifiable via `terraform console`.

## Prerequisites
- Terraform 1.x installed
- No provider needed — pure language features, zero cost
- Previous chapter (variables) completed

## Concept Summary

| Group | Operators |
|-------|-----------|
| Arithmetic | `+` `-` `*` `/` `%` and unary `-` |
| Comparison | `==` `!=` `>` `<` `>=` `<=` |
| Logical | `&&` `\|\|` `!` |
| Conditional | `condition ? true_value : false_value` |
| Grouping | `( )` |

A `for` expression loops over a collection and builds a new one:
- `[for ...]` → yields a **list**
- `{for ... => ...}` → yields a **map**
- Add `if` at the end to **filter**.

---

## Step 1 — Create the lab folder

```bash
mkdir -p ~/tf-expression-demo && cd ~/tf-expression-demo
```

## Step 2 — Create `versions.tf` (no provider needed)

```hcl
terraform {
  required_version = ">= 1.0"
}
```

## Step 3 — Create `variables.tf`

```hcl
variable "numbers" {
  type    = list(number)
  default = [10, 20, 30, 40, 50]
}

variable "users" {
  type = list(object({
    name = string
    role = string
  }))
  default = [
    { name = "raju",     role = "admin" },
    { name = "shyam",    role = "developer" },
    { name = "baburao",  role = "viewer" },
  ]
}

variable "instance_config" {
  type = map(string)
  default = {
    ami           = "ami-0abcdef1234567890"
    instance_type = "t3.micro"
    environment   = "dev"
  }
}
```

## Step 4 — Create `locals.tf` with arithmetic & comparison operators

```hcl
locals {
  # ---------- Arithmetic ----------
  sum        = 10 + 20           # 30
  difference = 50 - 15           # 35
  product    = 6 * 7             # 42
  quotient   = 100 / 4           # 25
  remainder  = 10 % 3            # 1

  # ---------- Comparison ----------
  is_equal     = (10 == 10)      # true
  is_not_equal = (10 != 20)      # true
  is_greater   = (50 > 30)       # true
  is_less      = (5 < 10)        # true

  # ---------- Logical ----------
  both_true  = (true && true)    # true
  either_one = (true || false)   # true
  negation   = !false            # true

  # ---------- Conditional (ternary) ----------
  env          = "production"
  instance_type = local.env == "production" ? "t3.large" : "t3.micro"
}
```

## Step 5 — Initialize and verify with `terraform console`

```bash
terraform init
terraform console
```

Inside the console, test each expression:

```text
> local.sum
30
> local.instance_type
"t3.large"
> local.is_equal
true
> local.both_true
true
```

Type `exit` to leave the console.

## Step 6 — Working with Lists

Add to `locals.tf`:

```hcl
locals {
  # ... previous locals ...

  # ---------- List operations ----------
  fruits      = ["apple", "banana", "cherry", "date"]
  first_fruit = local.fruits[0]          # "apple"
  last_fruit  = local.fruits[3]          # "date"
  fruit_count = length(local.fruits)     # 4
}
```

Verify:

```text
> local.first_fruit
"apple"
> local.fruit_count
4
```

## Step 7 — Working with Maps

Add to `locals.tf`:

```hcl
locals {
  # ... previous locals ...

  # ---------- Map operations ----------
  server_ports = {
    http  = 80
    https = 443
    ssh   = 22
  }

  http_port   = local.server_ports["http"]       # 80
  ssh_port    = local.server_ports["ssh"]         # 22
  all_keys    = keys(local.server_ports)          # ["http", "https", "ssh"]
  all_values  = values(local.server_ports)        # [80, 443, 22]
}
```

Verify:

```text
> local.http_port
80
> local.all_keys
tolist(["http", "https", "ssh"])
```

## Step 8 — Working with Objects

Add to `locals.tf`:

```hcl
locals {
  # ... previous locals ...

  # ---------- Object operations ----------
  server = {
    name          = "web-server-01"
    instance_type = "t3.micro"
    ami           = "ami-0abcdef1234567890"
  }

  server_name = local.server.name                  # "web-server-01"
  server_ami  = local.server["ami"]                # "ami-0abcdef..."
}
```

## Step 9 — `for` expressions: create lists

Add to `locals.tf`:

```hcl
locals {
  # ... previous locals ...

  # ---------- for expression → list ----------
  doubled   = [for n in var.numbers : n * 2]
  # Result: [20, 40, 60, 80, 100]

  uppercase_fruits = [for f in local.fruits : upper(f)]
  # Result: ["APPLE", "BANANA", "CHERRY", "DATE"]

  user_names = [for u in var.users : u.name]
  # Result: ["raju", "shyam", "baburao"]
}
```

Verify:

```text
> local.doubled
tolist([20, 40, 60, 80, 100])
> local.user_names
tolist(["raju", "shyam", "baburao"])
```

## Step 10 — `for` expressions: create maps

```hcl
locals {
  # ... previous locals ...

  # ---------- for expression → map ----------
  user_roles = { for u in var.users : u.name => u.role }
  # Result: { "raju" = "admin", "shyam" = "developer", "baburao" = "viewer" }
}
```

Verify:

```text
> local.user_roles
tomap({ "baburao" = "viewer", "raju" = "admin", "shyam" = "developer" })
> local.user_roles["raju"]
"admin"
```

## Step 11 — `for` expressions with `if` filter

```hcl
locals {
  # ... previous locals ...

  # ---------- for expression with if → filter ----------
  big_numbers = [for n in var.numbers : n if n > 25]
  # Result: [30, 40, 50]

  admin_users = [for u in var.users : u.name if u.role == "admin"]
  # Result: ["raju"]
}
```

Verify:

```text
> local.big_numbers
tolist([30, 40, 50])
> local.admin_users
tolist(["raju"])
```

## Step 12 — Splat operator `[*]`

```hcl
locals {
  # ... previous locals ...

  # ---------- Splat operator ----------
  all_user_names_splat = var.users[*].name
  # Result: ["raju", "shyam", "baburao"]
}
```

Verify:

```text
> local.all_user_names_splat
tolist(["raju", "shyam", "baburao"])
```

## Step 13 — Create `outputs.tf` to display results

```hcl
output "arithmetic_results" {
  value = {
    sum       = local.sum
    product   = local.product
    remainder = local.remainder
  }
}

output "conditional_instance_type" {
  value = local.instance_type
}

output "user_roles_map" {
  value = local.user_roles
}

output "admin_users" {
  value = local.admin_users
}

output "doubled_numbers" {
  value = local.doubled
}
```

## Step 14 — Run plan to see all outputs

```bash
terraform plan
```

Expected: All outputs display correctly with computed values.

## Cleanup

No resources were created — just delete the folder:

```bash
cd ..
rm -rf ~/tf-expression-demo
```

## Key Takeaways
- **Operators**: `+`, `-`, `*`, `/`, `%`, `==`, `!=`, `>`, `<`, `&&`, `||`, `!` work inside any expression.
- **Conditional**: `condition ? true_val : false_val` is great for environment-based config.
- **`for` expressions**: `[for ... : ...]` → list, `{for ... : ... => ...}` → map, add `if` to filter.
- **Splat**: `collection[*].attribute` is shorthand for `[for item in collection : item.attribute]`.
- **`terraform console`** is your best friend for testing expressions interactively.
