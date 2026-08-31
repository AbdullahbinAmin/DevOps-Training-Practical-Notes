# Day 09 — Terraform Operators and Expressions (List, Map, Object, Loops)

> Work with lists, maps and objects, then transform them using operators, conditionals and `for` expressions — all provable in `terraform console`.

## Learning Objectives
- Use arithmetic, comparison and logical operators inside `locals`.
- Index a list, look up a map key, and read an object field.
- Write `for` expressions that produce lists and maps, with optional `if` filters.
- Use the conditional (ternary) operator and the splat operator `[*]`.
- Explore all of it interactively with `terraform console`.

## Prerequisites
- Terraform 1.x.
- Day 09 file 28 finished (variable types).
- No provider needed — pure language features, zero cost.

## Concept (plain English)
An expression is anything that produces a value: a literal, a `var.` reference, a function call, or a combination joined by operators. Operators are the usual set:

| Group | Operators |
|-------|-----------|
| Arithmetic | `+` `-` `*` `/` `%` and unary `-` |
| Comparison | `==` `!=` `>` `<` `>=` `<=` |
| Logical | `&&` `\|\|` `!` |
| Conditional | `condition ? true_value : false_value` |
| Grouping | `( )` |

A `for` expression loops over a collection and builds a new one. Brackets decide the output type: `[for ...]` yields a list, `{for ... => ...}` yields a map. Add `if` at the end to filter.

`locals` are named intermediate values — computed once, referenced as `local.<name>`. They are the natural place to park expressions so your resources stay readable.

## Step-by-Step Practical

1. Create the lab folder.

```bash
mkdir -p ~/tf-expression-demo && cd ~/tf-expression-demo
```

2. `versions.tf` — no provider needed for this demo.

```hcl
terraform {
  required_version = ">= 1.0"
}
```

3. `variables.tf` — a number list, a list of objects, and a map.

```hcl
variable "num_list" {
  description = "List of numbers, indexed from 0"
  type        = list(number)
  default     = [1, 2, 3, 4, 5]
}

variable "person_list" {
  description = "List of objects — each object is a person"
  type = list(object({
    fname = string
    lname = string
  }))
  default = [
    { fname = "Raju", lname = "Rastogi" },
    { fname = "Shyam", lname = "Paul" },
  ]
}

variable "map_list" {
  description = "Map of key-value pairs, like a dictionary"
  type        = map(number)
  default = {
    "one"   = 1
    "two"   = 2
    "three" = 3
  }
}

variable "value" {
  description = "A sample string"
  type        = string
  default     = "Hello World"
}

variable "environment" {
  type    = string
  default = "dev"
}
```

4. `main.tf` — operators and basic calculations.

```hcl
locals {
  # Arithmetic
  multiply  = 2 * 2      # 4
  add       = 10 + 5     # 15
  subtract  = 10 - 5     # 5
  divide    = 10 / 4     # 2.5
  remainder = 10 % 3     # 1

  # Comparison
  equal     = 2 == 2     # true
  not_equal = 2 != 3     # true
  greater   = 5 > 3      # true

  # Logical
  both   = (2 > 1) && (3 > 2)  # true
  either = (2 > 5) || (3 > 2)  # true
  negate = !(2 > 5)            # true

  # Precedence: * before +, so use () when unsure
  precedence_trap = 2 + 3 * 4        # 14
  precedence_fix  = (2 + 3) * 4      # 20

  # Conditional (ternary)
  instance_type = var.environment == "prod" ? "m5.large" : "t3.micro"
  replica_count = var.environment == "prod" ? 3 : 1

  # Collection access
  first_number  = var.num_list[0]              # 1
  last_number   = var.num_list[length(var.num_list) - 1]  # 5
  value_of_two  = var.map_list["two"]           # 2
  first_person  = var.person_list[0].fname      # "Raju"

  # Safe map access — no crash on a missing key
  safe_lookup = lookup(var.map_list, "four", 0) # 0

  # String interpolation and a multi-line heredoc
  greeting = "Hi ${var.person_list[0].fname}, env is ${upper(var.environment)}"

  banner = <<-EOT
    Environment : ${var.environment}
    Instance    : ${local.instance_type}
  EOT
}
```

5. `loops.tf` — `for` expressions over the list.

```hcl
locals {
  # a) Double each number
  double = [for n in var.num_list : n * 2]              # [2, 4, 6, 8, 10]

  # b) Odd numbers only (if filter)
  odd = [for n in var.num_list : n if n % 2 != 0]       # [1, 3, 5]

  # c) Sum of all numbers
  sum = sum(var.num_list)                               # 15

  # d) Index and value together
  indexed = [for i, n in var.num_list : "${i}:${n}"]    # ["0:1","1:2",...]
}
```

6. `loops-objects.tf` — `for` expressions over the list of objects.

```hcl
locals {
  # First names only
  fname_list = [for p in var.person_list : p.fname]     # ["Raju", "Shyam"]

  # Last names only
  lname_list = [for p in var.person_list : p.lname]     # ["Rastogi", "Paul"]

  # Full names
  full_names = [for p in var.person_list : "${p.fname} ${p.lname}"]

  # List of objects -> map, keyed by first name
  people_by_fname = { for p in var.person_list : p.fname => p.lname }
  # { Raju = "Rastogi", Shyam = "Paul" }

  # Splat operator — shorthand for a list of one attribute
  fname_splat = var.person_list[*].fname                # ["Raju", "Shyam"]
}
```

7. `loops-maps.tf` — `for` expressions over the map.

```hcl
locals {
  # a) Keys only
  keys_list = [for key, value in var.map_list : key]    # ["one","three","two"]

  # b) Values only
  values_list = [for k, v in var.map_list : v]          # [1, 3, 2]

  # c) Transform the map — double every value
  double_map = { for k, v in var.map_list : k => v * 2 }
  # { one = 2, three = 6, two = 4 }

  # d) Filter the map
  big_map = { for k, v in var.map_list : k => v if v > 1 }
  # { three = 3, two = 2 }

  # e) Uppercase the keys
  upper_map = { for k, v in var.map_list : upper(k) => v }
}
```

8. `outputs.tf`.

```hcl
output "multiply"        { value = local.multiply }
output "add"             { value = local.add }
output "equal"           { value = local.equal }
output "not_equal"       { value = local.not_equal }
output "instance_type"   { value = local.instance_type }
output "double"          { value = local.double }
output "odd"             { value = local.odd }
output "sum"             { value = local.sum }
output "fname_list"      { value = local.fname_list }
output "lname_list"      { value = local.lname_list }
output "keys_list"       { value = local.keys_list }
output "values_list"     { value = local.values_list }
output "double_map"      { value = local.double_map }
output "people_by_fname" { value = local.people_by_fname }
output "greeting"        { value = local.greeting }
```

9. Initialize and plan.

```bash
terraform init
terraform plan
```

10. Explore interactively — the fastest way to learn expressions.

```bash
terraform console
```

Type these one per line:

```hcl
2 * 2
10 % 3
2 + 3 * 4
var.num_list
var.num_list[0]
var.map_list["two"]
var.person_list[0].fname
[for n in var.num_list : n * 2]
[for n in var.num_list : n if n % 2 != 0]
{ for k, v in var.map_list : k => v * 2 }
var.environment == "prod" ? "m5.large" : "t3.micro"
local.double
local.double_map
exit
```

11. Apply to lock the outputs in and read them back.

```bash
terraform apply -auto-approve
terraform output
terraform output -json double_map
```

## Expected Output

`terraform console` session:

```
> 2 * 2
4
> 10 % 3
1
> 2 + 3 * 4
14
> var.num_list[0]
1
> var.map_list["two"]
2
> [for n in var.num_list : n * 2]
[
  2,
  4,
  6,
  8,
  10,
]
> [for n in var.num_list : n if n % 2 != 0]
[
  1,
  3,
  5,
]
> { for k, v in var.map_list : k => v * 2 }
{
  "one" = 2
  "three" = 6
  "two" = 4
}
```

`terraform output`:

```
add             = 15
double          = [ 2, 4, 6, 8, 10 ]
double_map      = { "one" = 2, "three" = 6, "two" = 4 }
equal           = true
fname_list      = [ "Raju", "Shyam" ]
instance_type   = "t3.micro"
keys_list       = [ "one", "three", "two" ]
lname_list      = [ "Rastogi", "Paul" ]
multiply        = 4
not_equal       = true
odd             = [ 1, 3, 5 ]
people_by_fname = { "Raju" = "Rastogi", "Shyam" = "Paul" }
sum             = 15
values_list     = [ 1, 3, 2 ]
```

## Verification
- `terraform validate` → `Success! The configuration is valid.`
- `local.double` is `[2, 4, 6, 8, 10]` — the original list is unchanged; `for` always builds a new value.
- `local.odd` is `[1, 3, 5]` — the `if` filter dropped the even entries.
- `keys_list` comes back alphabetically sorted (`one, three, two`), **not** in declaration order — maps in Terraform are ordered by key.
- `values_list` is `[1, 3, 2]`, matching that same key order — this is the classic surprise.
- Switching `-var "environment=prod"` flips `instance_type` to `m5.large`, proving the ternary.

## Cleanup

```bash
cd ~/tf-expression-demo
terraform destroy -auto-approve
cd ~ && rm -rf ~/tf-expression-demo
```

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Invalid index` / `element out of range` | Indexed past the end, e.g. `var.num_list[9]` | Guard with `length()`, or use `element(list, i)` which wraps |
| `Invalid index: given key does not identify an element` | Missing map key, e.g. `var.map_list["four"]` | Use `lookup(var.map_list, "four", 0)` or `try(...)` |
| `Invalid 'for' expression: key expression is required` | Used `{ }` without `=>` | Write `{ for k, v in m : k => v }`, or switch to `[ ]` |
| `Duplicate object key` | Two iterations produced the same map key | Make the key unique, or add `...` to group: `k => v...` |
| `Unsupported attribute: object has no attribute "fname"` | Field name typo or wrong object shape | Check the declared `object({...})` type |
| `Invalid operand: number required` | Arithmetic on a string, e.g. `"2" * 2` | Convert first: `tonumber(var.x) * 2` |
| Ternary returns `Inconsistent conditional result types` | Branches return different types | Make both branches the same type (both string, both number) |
| `Cycle: local.a, local.b` | Two locals reference each other | Break the loop; locals must form a DAG |
| Map output order looks wrong | Maps sort by key, always | Use a list if order matters |
| `Self-reference` in locals | A local referenced itself | Compute in two steps with two locals |

## Key Takeaways
- `[for ...]` produces a list, `{for ... => ...}` produces a map; trailing `if` filters.
- Operator precedence follows normal maths: `!` then `* / %` then `+ -` then comparison then `&&` then `||`. Parenthesise anything non-obvious.
- Lists keep insertion order; maps and sets do not — maps sort by key.
- `var.list[*].field` (splat) is shorthand for `[for x in var.list : x.field]`.
- Use `lookup()` / `try()` for keys that might be absent instead of letting the plan crash.
- `terraform console` is your REPL: test an expression there before pasting it into a resource.

## Next: [Terraform Functions](30-terraform-functions.md)
