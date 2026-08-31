# Day 09 — Terraform Built-in Functions

> Terraform ships ~100 built-in functions for strings, numbers, collections, encoding, dates and type conversion — and `terraform console` lets you test every one of them instantly.

## Learning Objectives
- Use the main string functions: `lower`, `upper`, `split`, `join`, `replace`, `substr`, `startswith`, `trimspace`, `format`.
- Use numeric functions: `max`, `min`, `abs`, `floor`, `ceil`, `sum`, `pow`.
- Use collection functions: `length`, `contains`, `keys`, `values`, `lookup`, `merge`, `flatten`, `distinct`, `sort`, `zipmap`, `element`, `concat`, `slice`, `toset`, `tolist`.
- Handle failures safely with `can()` and `try()`.
- Know that Terraform has no user-defined functions in the classic sense (1.8+ adds provider-defined functions).

## Prerequisites
- Terraform 1.x.
- Files 28 and 29 of Day 09 completed.
- No provider, no cost.

## Concept (plain English)
A function takes arguments and returns a value: `lower("Hello World")` → `"hello world"`. Functions never mutate their input — they return something new. You can only use the built-ins; you cannot define your own function in HCL (Terraform 1.8+ can call functions supplied by a provider, e.g. `provider::aws::arn_parse`).

Rough categories:

| Category | Examples |
|----------|----------|
| String | `lower`, `upper`, `title`, `split`, `join`, `replace`, `substr`, `startswith`, `endswith`, `trimspace`, `format`, `regex`, `regexall` |
| Numeric | `max`, `min`, `abs`, `floor`, `ceil`, `round`, `sum`, `pow`, `signum`, `parseint` |
| Collection | `length`, `contains`, `keys`, `values`, `lookup`, `merge`, `flatten`, `distinct`, `sort`, `reverse`, `zipmap`, `element`, `index`, `concat`, `slice`, `chunklist`, `setunion`, `setintersection` |
| Type conversion | `tostring`, `tonumber`, `tobool`, `tolist`, `toset`, `tomap` |
| Encoding | `jsonencode`, `jsondecode`, `yamlencode`, `base64encode`, `base64decode`, `urlencode` |
| Filesystem | `file`, `fileexists`, `templatefile`, `abspath`, `basename`, `dirname` |
| Date/time | `timestamp`, `timeadd`, `formatdate` |
| Network | `cidrsubnet`, `cidrhost`, `cidrnetmask` |
| Logical | `can`, `try`, `coalesce`, `coalescelist`, `one` |
| Hash/ID | `uuid`, `md5`, `sha256`, `filesha256` |

## Step-by-Step Practical

1. Create the lab folder.

```bash
mkdir -p ~/tf-functions-demo && cd ~/tf-functions-demo
```

2. `versions.tf`.

```hcl
terraform {
  required_version = ">= 1.0"
}
```

3. `variables.tf`.

```hcl
variable "value" {
  description = "A sample string"
  type        = string
  default     = "Hello World"
}

variable "num_list" {
  description = "List of numbers"
  type        = list(number)
  default     = [1, 2, 3, 4, 5]
}

variable "string_list" {
  description = "List of server names"
  type        = list(string)
  default     = ["server1", "server2", "server3"]
}

variable "person_list" {
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
  type = map(number)
  default = {
    "one"   = 1
    "two"   = 2
    "three" = 3
  }
}

variable "base_cidr" {
  type    = string
  default = "10.0.0.0/16"
}
```

4. `functions-string.tf`.

```hcl
locals {
  f_lower      = lower(var.value)                    # "hello world"
  f_upper      = upper(var.value)                    # "HELLO WORLD"
  f_title      = title("hello world")                # "Hello World"
  f_startswith = startswith(var.value, "Hello")      # true
  f_endswith   = endswith(var.value, "World")        # true
  f_split      = split(" ", var.value)               # ["Hello", "World"]
  f_join       = join(":", var.string_list)           # "server1:server2:server3"
  f_replace    = replace(var.value, "World", "Terraform") # "Hello Terraform"
  f_substr     = substr(var.value, 0, 5)             # "Hello"
  f_trim       = trimspace("  padded  ")             # "padded"
  f_strlen     = length(var.value)                   # 11 (characters)
  f_format     = format("%s-%03d", "web", 7)         # "web-007"
  f_formatlist = formatlist("srv-%s", var.string_list)
  f_regex      = regex("^([a-z]+)([0-9]+)$", "server42") # ["server", "42"]
  f_regexall   = regexall("[0-9]+", "a1b22c333")     # ["1","22","333"]
}
```

5. `functions-numeric.tf`.

```hcl
locals {
  f_max   = max(var.num_list...)   # 5  -- note the ... expansion
  f_min   = min(var.num_list...)   # 1
  f_abs   = abs(-15)               # 15
  f_sum   = sum(var.num_list)      # 15
  f_floor = floor(2.7)             # 2
  f_ceil  = ceil(2.1)              # 3
  f_round = round(2.5)             # 3
  f_pow   = pow(2, 10)             # 1024
  f_int   = parseint("ff", 16)     # 255
}
```

`max()` and `min()` take separate arguments, not a list — the `...` spread operator expands the list. `max(var.num_list)` is an error.

6. `functions-collection.tf`.

```hcl
locals {
  f_length   = length(var.string_list)                       # 3
  f_contains = contains(var.string_list, "server2")          # true
  f_keys     = keys(var.map_list)                            # ["one","three","two"]
  f_values   = values(var.map_list)                          # [1, 3, 2]
  f_lookup   = lookup(var.map_list, "four", 0)               # 0 (default)
  f_merge    = merge(var.map_list, { "four" = 4 })
  f_element  = element(var.string_list, 1)                   # "server2"
  f_wrap     = element(var.string_list, 4)                   # "server2" — wraps!
  f_index    = index(var.string_list, "server3")             # 2
  f_concat   = concat(var.num_list, [6, 7])                  # [1..7]
  f_slice    = slice(var.num_list, 1, 3)                     # [2, 3]
  f_distinct = distinct([1, 1, 2, 2, 3])                     # [1, 2, 3]
  f_sort     = sort(["c", "a", "b"])                         # ["a","b","c"]
  f_reverse  = reverse(var.num_list)                         # [5,4,3,2,1]
  f_flatten  = flatten([[1, 2], [3, 4]])                     # [1,2,3,4]
  f_zipmap   = zipmap(var.string_list, slice(var.num_list, 0, 3))
  f_toset    = toset([1, 1, 2])                              # {1, 2}
  f_chunk    = chunklist(var.num_list, 2)                    # [[1,2],[3,4],[5]]
}
```

`zipmap` needs both lists the same length, which is why `slice()` trims the number list to 3 items.

7. `functions-encoding-time-net.tf`.

```hcl
locals {
  f_json    = jsonencode({ name = "web", count = 2 })
  f_decode  = jsondecode("{\"a\":1}")            # { a = 1 }
  f_yaml    = yamlencode({ name = "web" })
  f_b64     = base64encode("hello")              # "aGVsbG8="
  f_now     = timestamp()                        # e.g. "2026-08-31T10:00:00Z"
  f_date    = formatdate("YYYY-MM-DD", timestamp())
  f_plus24  = timeadd(timestamp(), "24h")
  f_subnet0 = cidrsubnet(var.base_cidr, 8, 0)    # "10.0.0.0/24"
  f_subnet1 = cidrsubnet(var.base_cidr, 8, 1)    # "10.0.1.0/24"
  f_host    = cidrhost(var.base_cidr, 5)         # "10.0.0.5"
  f_mask    = cidrnetmask("10.0.0.0/24")         # "255.255.255.0"
}
```

`timestamp()` and `uuid()` change on every run, so they force a diff on every plan. Never use them for resource names or tags you want to stay stable.

8. `functions-safe.tf` — `can()` and `try()`.

```hcl
locals {
  f_can_ok   = can(tonumber("42"))                # true
  f_can_bad  = can(tonumber("abc"))               # false
  f_try      = try(var.map_list["four"], 0)       # 0
  f_try_deep = try(var.person_list[9].fname, "unknown")  # "unknown"
  f_coalesce = coalesce(null, "", "fallback")     # "" — empty string counts!
  f_coalesce_safe = coalesce(null, "fallback")    # "fallback"
}
```

9. `functions-loops.tf` — functions plus `for` expressions, from the slide.

```hcl
locals {
  double_list = [for n in var.num_list : n * 2]                  # [2,4,6,8,10]
  odd_list    = [for n in var.num_list : n if n % 2 != 0]        # [1,3,5]
  fname_list  = [for p in var.person_list : p.fname]             # ["Raju","Shyam"]
  keys_only   = [for k, v in var.map_list : k]                   # ["one","three","two"]
  double_map  = { for k, v in var.map_list : k => v * 2 }        # {one=2,three=6,two=4}

  # Realistic combo: build a subnet map from a count
  subnet_cidrs = { for i in range(3) : "subnet-${i}" => cidrsubnet(var.base_cidr, 8, i) }
}
```

10. `outputs.tf`.

```hcl
output "f_lower"      { value = local.f_lower }
output "f_startswith" { value = local.f_startswith }
output "f_split"      { value = local.f_split }
output "f_max"        { value = local.f_max }
output "f_min"        { value = local.f_min }
output "f_abs"        { value = local.f_abs }
output "f_length"     { value = local.f_length }
output "f_join"       { value = local.f_join }
output "f_contains"   { value = local.f_contains }
output "double_list"  { value = local.double_list }
output "odd_list"     { value = local.odd_list }
output "fname_list"   { value = local.fname_list }
output "keys_only"    { value = local.keys_only }
output "double_map"   { value = local.double_map }
output "subnet_cidrs" { value = local.subnet_cidrs }
```

11. Run it.

```bash
terraform init
terraform validate
terraform plan
terraform apply -auto-approve
terraform output
```

12. Test functions interactively — do this constantly.

```bash
terraform console
```

```hcl
lower("Hello World")
startswith("Hello World", "Hello")
split(" ", "Hello World")
max(var.num_list...)
min(var.num_list...)
abs(-15)
length(var.string_list)
join(":", var.string_list)
contains(var.string_list, "server2")
keys(var.map_list)
values(var.map_list)
merge(var.map_list, { four = 4 })
element(var.string_list, 4)
cidrsubnet("10.0.0.0/16", 8, 1)
try(var.map_list["four"], 0)
can(tonumber("abc"))
exit
```

## Expected Output

`terraform console`:

```
> lower("Hello World")
"hello world"
> startswith("Hello World", "Hello")
true
> split(" ", "Hello World")
[
  "Hello",
  "World",
]
> max(var.num_list...)
5
> min(var.num_list...)
1
> abs(-15)
15
> length(var.string_list)
3
> join(":", var.string_list)
"server1:server2:server3"
> contains(var.string_list, "server2")
true
> element(var.string_list, 4)
"server2"
> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"
> can(tonumber("abc"))
false
```

`terraform output`:

```
double_list  = [ 2, 4, 6, 8, 10 ]
double_map   = { "one" = 2, "three" = 6, "two" = 4 }
f_abs        = 15
f_contains   = true
f_join       = "server1:server2:server3"
f_length     = 3
f_lower      = "hello world"
f_max        = 5
f_min        = 1
f_split      = [ "Hello", "World" ]
f_startswith = true
fname_list   = [ "Raju", "Shyam" ]
keys_only    = [ "one", "three", "two" ]
odd_list     = [ 1, 3, 5 ]
subnet_cidrs = {
  "subnet-0" = "10.0.0.0/24"
  "subnet-1" = "10.0.1.0/24"
  "subnet-2" = "10.0.2.0/24"
}
```

## Verification
- `terraform validate` → `Success! The configuration is valid.`
- `element(var.string_list, 4)` returns `"server2"` rather than erroring — proof that `element` wraps modulo the list length, unlike `[4]`.
- `max(var.num_list...)` works while `max(var.num_list)` fails — proof the `...` spread is required.
- `lookup(var.map_list, "four", 0)` returns `0` instead of crashing.
- `can(tonumber("abc"))` is `false`, so validation conditions built on it are safe.
- Run `terraform plan` twice: `timestamp()`-derived values change each time, showing why they must not be used in resource attributes.

## Cleanup

```bash
cd ~/tf-functions-demo
terraform destroy -auto-approve
cd ~ && rm -rf ~/tf-functions-demo
```

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Invalid function argument: list of number required` on `max` | Passed a list to `max`/`min` | Spread it: `max(var.num_list...)` |
| `Call to function "lower" failed: string required` | Argument is a number/list | Convert: `lower(tostring(var.x))` |
| `Invalid index` on `var.list[4]` | Index beyond list length | Use `element(list, 4)` (wraps) or `try(list[4], null)` |
| `given key does not identify an element` | Missing map key | `lookup(map, key, default)` or `try(map[key], default)` |
| `Call to function "zipmap" failed: length mismatch` | Key and value lists differ in length | Trim with `slice()` so lengths match |
| `Error: Function calls not allowed` in a `variable` block | Functions are not permitted in `variable` `default` | Move the computation into `locals` |
| Plan shows changes every run | `timestamp()` / `uuid()` in a resource attribute | Use static values, or `ignore_changes` in `lifecycle` |
| `regex` errors on no match | `regex()` throws when the pattern misses | Wrap in `can()` or use `regexall()` (returns `[]`) |
| `coalesce` returned `""` unexpectedly | `coalesce` only skips `null`, not empty strings | Filter explicitly, or use `try` with a check |
| `There is no function named "toarray"` | Guessed the name | Real names: `tolist`, `toset`, `tomap`, `tostring` |

## Key Takeaways
- Functions are pure: they return new values and never mutate arguments.
- You cannot write your own HCL functions; only built-ins (plus provider-defined functions on Terraform 1.8+).
- `max`/`min` need the `...` spread; `sum`/`length` take the collection directly.
- `element()` wraps around, `[index]` errors — pick deliberately.
- `lookup()`, `try()` and `can()` are your defence against missing keys and bad conversions.
- `cidrsubnet()` beats hand-written CIDR strings for generating subnets.
- Avoid `timestamp()` and `uuid()` inside resource arguments — they cause perpetual diffs.
- Live in `terraform console` while learning: it evaluates against your real variables and locals.

## Next: [Multiple Resources with count](31-multiple-resources-count.md)
