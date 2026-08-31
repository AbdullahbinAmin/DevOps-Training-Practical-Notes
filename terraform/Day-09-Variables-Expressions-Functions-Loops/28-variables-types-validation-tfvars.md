# Day 09 — Terraform Variables: Types, Validation, ENV, tfvars, auto.tfvars

> Write the config once, feed it different values per environment using variables, tfvars files, CLI flags and `TF_VAR_` environment variables.

## Learning Objectives
- Declare input variables with `description`, `type`, `default`, `sensitive` and `validation`.
- Use every supported type: `string`, `number`, `bool`, `list`, `set`, `map`, `object`, `tuple`, `any`.
- Supply values 5 ways: default, `terraform.tfvars`, `*.auto.tfvars`, `-var` / `-var-file`, `TF_VAR_*`.
- Recite and demonstrate the variable precedence order.
- Keep secrets out of code and out of CLI output.

## Prerequisites
- Terraform 1.x installed (`terraform version`).
- A shell (bash / Git Bash / WSL).
- No cloud provider needed — this lab is language-only, so it costs nothing.

## Concept (plain English)
A variable is a labelled input slot. `variables.tf` declares the slot; `main.tf` / outputs read it as `var.<name>`; values arrive from outside. That gives you reusability (same code, dev vs prod), no hardcoding, and one obvious place to change a value.

Precedence — **later wins** over earlier:

| # | Source | Notes |
|---|--------|-------|
| 1 | `default` in the `variable` block | lowest priority |
| 2 | `TF_VAR_<name>` environment variable | name is case-sensitive |
| 3 | `terraform.tfvars` | auto-loaded |
| 4 | `terraform.tfvars.json` | auto-loaded |
| 5 | `*.auto.tfvars` / `*.auto.tfvars.json` | auto-loaded, alphabetical order |
| 6 | `-var` and `-var-file` on the command line | highest priority, left-to-right |

Note the trap: `dev.tfvars` is **not** auto-loaded — only `terraform.tfvars` and files ending in `.auto.tfvars` are. Anything else needs `-var-file`.

## Step-by-Step Practical

1. Create the lab folder.

```bash
mkdir -p ~/tf-variable-demo && cd ~/tf-variable-demo
```

2. `versions.tf` — pin Terraform, no provider required.

```hcl
terraform {
  required_version = ">= 1.0"
}
```

3. `variables.tf` — every type plus validation and a sensitive variable.

```hcl
variable "instance_type" {
  description = "What type of instance you want to create"
  type        = string
  default     = "t3.micro"

  validation {
    condition     = contains(["t3.micro", "t3.small", "m5.large"], var.instance_type)
    error_message = "instance_type must be one of: t3.micro, t3.small, m5.large."
  }
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"

  validation {
    condition     = can(regex("^(dev|test|prod)$", var.environment))
    error_message = "environment must be dev, test or prod."
  }
}

variable "instance_count" {
  description = "How many servers"
  type        = number
  default     = 2

  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 10
    error_message = "instance_count must be between 1 and 10."
  }
}

variable "enable_monitoring" {
  description = "Turn detailed monitoring on"
  type        = bool
  default     = true
}

variable "num_list" {
  description = "A list of numbers (indexed from 0)"
  type        = list(number)
  default     = [1, 2, 3, 4, 5]
}

variable "allowed_ports" {
  description = "A set removes duplicates and has no order"
  type        = set(number)
  default     = [22, 80, 443]
}

variable "map_list" {
  description = "Key-value pairs, like a dictionary"
  type        = map(number)
  default = {
    "one"   = 1
    "two"   = 2
    "three" = 3
  }
}

variable "person_list" {
  description = "A list of objects — each object is a person"
  type = list(object({
    fname = string
    lname = string
  }))
  default = [
    { fname = "Raju", lname = "Rastogi" },
    { fname = "Shyam", lname = "Paul" },
  ]
}

variable "server" {
  description = "One object with typed fields; optional() needs Terraform >= 1.3"
  type = object({
    name  = string
    size  = optional(string, "t3.micro")
    tags  = optional(map(string), {})
  })
  default = {
    name = "web-01"
  }
}

variable "pair" {
  description = "A tuple has a fixed length and per-position types"
  type    = tuple([string, number, bool])
  default = ["az-a", 3, true]
}

variable "anything" {
  description = "Escape hatch — avoid unless you must"
  type        = any
  default     = null
}

variable "db_password" {
  description = "Never hardcode this. Pass via TF_VAR_db_password."
  type        = string
  sensitive   = true
  default     = null

  validation {
    condition     = var.db_password == null || length(var.db_password) >= 12
    error_message = "db_password must be at least 12 characters."
  }
}
```

4. `main.tf` — consume the variables in `locals` (no cloud resources, so nothing to bill).

```hcl
locals {
  name_prefix = "${var.environment}-app"
  first_names = [for p in var.person_list : p.fname]
  two_value   = var.map_list["two"]
}
```

5. `outputs.tf` — show what the variables resolved to.

```hcl
output "instance_type"  { value = var.instance_type }
output "environment"    { value = var.environment }
output "instance_count" { value = var.instance_count }
output "name_prefix"    { value = local.name_prefix }
output "first_names"    { value = local.first_names }
output "two_value"      { value = local.two_value }
output "server_size"    { value = var.server.size }

output "db_password" {
  value     = var.db_password
  sensitive = true
}
```

6. `terraform.tfvars` — auto-loaded, plain `key = value` pairs. Keep real secrets out of git.

```hcl
instance_type = "t3.small"
environment   = "dev"
```

7. `dev.auto.tfvars` — also auto-loaded, and it beats `terraform.tfvars`.

```hcl
instance_count = 3
```

8. `prod.tfvars` — **not** auto-loaded; requires `-var-file`.

```hcl
instance_type  = "m5.large"
environment    = "prod"
instance_count = 5
```

9. Add a `.gitignore` so state and secret tfvars never get committed.

```bash
cat > .gitignore <<'EOF'
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
!example.tfvars
EOF
```

10. Initialize and see the auto-loaded values.

```bash
terraform init
terraform plan
```

11. Override with an environment variable (precedence #2 — loses to tfvars).

```bash
export TF_VAR_instance_type=t3.micro
export TF_VAR_db_password='ChangeMe-NotARealSecret-123'
terraform plan
```

`instance_type` still shows `t3.small`, because `terraform.tfvars` outranks `TF_VAR_`. `db_password` is only set by the env var, so it is used.

12. Override on the command line (precedence #6 — wins over everything).

```bash
terraform plan -var "instance_type=m5.large"
terraform plan -var-file="prod.tfvars"
terraform plan -var-file="prod.tfvars" -var "instance_type=t3.micro"
```

13. Trip a validation block on purpose.

```bash
terraform plan -var "instance_type=m5.xlarge"
terraform plan -var "environment=staging"
terraform plan -var "instance_count=0"
```

14. Apply and inspect the outputs.

```bash
terraform apply -auto-approve
terraform output
terraform output -raw instance_type
terraform output -raw db_password
```

## Expected Output

Step 10:

```
Changes to Outputs:
  + environment    = "dev"
  + first_names    = ["Raju", "Shyam"]
  + instance_count = 3
  + instance_type  = "t3.small"
  + name_prefix    = "dev-app"
  + server_size    = "t3.micro"
  + two_value      = 2
```

Step 12, with `-var-file="prod.tfvars"`:

```
  + environment    = "prod"
  + instance_count = 5
  + instance_type  = "m5.large"
  + name_prefix    = "prod-app"
```

Step 13, the failing validation:

```
Error: Invalid value for variable

  on variables.tf line 1, in variable "instance_type":
   1: variable "instance_type" {

instance_type must be one of: t3.micro, t3.small, m5.large.
```

Step 14, `terraform output`:

```
db_password    = <sensitive>
environment    = "dev"
first_names    = [ "Raju", "Shyam" ]
instance_count = 3
instance_type  = "t3.small"
name_prefix    = "dev-app"
server_size    = "t3.micro"
two_value      = 2
```

## Verification
- `terraform validate` returns `Success! The configuration is valid.`
- `terraform fmt -check` returns nothing (files already formatted).
- Precedence proven: `terraform.tfvars` beats `TF_VAR_instance_type`; `-var` beats both.
- `dev.auto.tfvars` took effect without any CLI flag (`instance_count = 3`).
- `prod.tfvars` had **no** effect until you passed `-var-file`.
- `db_password` prints as `<sensitive>` in plan and in `terraform output` without `-raw`.
- All three validation blocks reject bad input with your own error message.

## Cleanup

```bash
cd ~/tf-variable-demo
terraform destroy -auto-approve   # nothing real to destroy; clears state
unset TF_VAR_instance_type TF_VAR_db_password
cd ~ && rm -rf ~/tf-variable-demo
```

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `No value for required variable` | Variable has no `default` and nothing supplied it | Add a `default`, put it in `terraform.tfvars`, or pass `-var` |
| `Invalid value for variable ... must be one of` | A `validation` `condition` returned false | Pass an allowed value, or widen the condition |
| tfvars file appears to be ignored | Named e.g. `prod.tfvars` — not auto-loaded | Rename to `*.auto.tfvars` or pass `-var-file="prod.tfvars"` |
| `TF_VAR_instance_type` has no effect | A tfvars file or `-var` outranks env vars | Remove the higher-precedence source, or use `-var` instead |
| `Variables not allowed` in a `variable` block | Referenced `var.other` inside `type`/`default` | Variables cannot reference other variables; use `locals` |
| `Unsuitable value type: number required` | Quoted a number in tfvars (`instance_count = "3"`) | Drop the quotes, or relax the declared `type` |
| `Output refers to sensitive values` | Output reads a `sensitive = true` variable | Add `sensitive = true` to that `output` block |
| `Invalid function call: can()` misuse | `regex()` errors outside `can()` on non-match | Wrap it: `can(regex("^(dev|test|prod)$", var.environment))` |
| Secret leaked into git history | `terraform.tfvars` was committed | Add `*.tfvars` to `.gitignore`, rotate the secret, purge history |

## Key Takeaways
- Precedence, lowest to highest: `default` → `TF_VAR_*` → `terraform.tfvars` → `terraform.tfvars.json` → `*.auto.tfvars` → `-var` / `-var-file`.
- Only `terraform.tfvars` and `*.auto.tfvars` auto-load. Everything else needs `-var-file`.
- Always set `type` and `description`; add `validation` to fail fast at plan time instead of mid-apply.
- `sensitive = true` masks values in plan and output, but they are still stored in plaintext in the state file — protect the backend.
- Prefer `TF_VAR_` env vars for secrets in CI, never `-var` (shell history) and never a committed tfvars file.
- Variable names are case-sensitive; `type = any` throws away the safety you get from typing.

## Next: [Terraform Operators and Expressions](29-operators-and-expressions.md)
