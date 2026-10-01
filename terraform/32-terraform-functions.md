# Chapter 32 — Terraform Functions

## Objective
Learn the most common **built-in functions** in Terraform, organized by category. Test them live in `terraform console`.

## Prerequisites
- Terraform 1.x installed
- No provider needed — zero cost
- Previous chapter (operators & expressions) completed

## Concept Summary
Terraform has **no user-defined functions** — only built-in ones. Categories:
- **String**: `upper`, `lower`, `title`, `replace`, `split`, `join`, `trimspace`, `substr`, `format`
- **Numeric**: `min`, `max`, `abs`, `ceil`, `floor`, `pow`, `log`
- **Collection**: `length`, `element`, `index`, `flatten`, `merge`, `lookup`, `keys`, `values`, `contains`, `distinct`, `sort`, `reverse`, `concat`, `zipmap`, `range`
- **Type Conversion**: `tostring`, `tonumber`, `tobool`, `tolist`, `toset`, `tomap`
- **Encoding**: `base64encode`, `base64decode`, `jsonencode`, `jsondecode`, `yamlencode`, `yamldecode`
- **Filesystem**: `file`, `fileexists`, `templatefile`, `basename`, `dirname`, `pathexpand`
- **Date/Time**: `timestamp`, `formatdate`, `timeadd`
- **Hash/Crypto**: `md5`, `sha256`, `bcrypt`, `uuid`
- **IP/Network**: `cidrsubnet`, `cidrhost`

---

## Step 1 — Create the lab folder

```bash
mkdir -p ~/tf-functions-demo && cd ~/tf-functions-demo
```

## Step 2 — Create `versions.tf`

```hcl
terraform {
  required_version = ">= 1.0"
}
```

## Step 3 — Initialize and open console

```bash
terraform init
terraform console
```

## Step 4 — String Functions

```text
> upper("hello terraform")
"HELLO TERRAFORM"

> lower("HELLO TERRAFORM")
"hello terraform"

> title("hello terraform")
"Hello Terraform"

> replace("hello world", "world", "terraform")
"hello terraform"

> split(",", "a,b,c,d")
tolist(["a", "b", "c", "d"])

> join("-", ["a", "b", "c"])
"a-b-c"

> trimspace("  hello  ")
"hello"

> substr("hello world", 0, 5)
"hello"

> format("Server-%03d", 5)
"Server-005"
```

## Step 5 — Numeric Functions

```text
> min(10, 20, 5, 30)
5

> max(10, 20, 5, 30)
30

> abs(-15)
15

> ceil(4.3)
5

> floor(4.9)
4

> pow(2, 8)
256
```

## Step 6 — Collection Functions

```text
> length(["a", "b", "c"])
3

> element(["a", "b", "c"], 1)
"b"

> index(["a", "b", "c"], "b")
1

> contains(["a", "b", "c"], "b")
true

> contains(["a", "b", "c"], "z")
false

> distinct(["a", "b", "a", "c", "b"])
tolist(["a", "b", "c"])

> sort(["banana", "apple", "cherry"])
tolist(["apple", "banana", "cherry"])

> reverse(["a", "b", "c"])
["c", "b", "a"]

> concat(["a", "b"], ["c", "d"])
["a", "b", "c", "d"]

> flatten([["a", "b"], ["c", "d"], ["e"]])
["a", "b", "c", "d", "e"]

> merge({a = 1}, {b = 2}, {c = 3})
{ "a" = 1, "b" = 2, "c" = 3 }

> keys({name = "raju", age = "25"})
tolist(["age", "name"])

> values({name = "raju", age = "25"})
tolist(["25", "raju"])

> lookup({name = "raju", age = "25"}, "name", "unknown")
"raju"

> lookup({name = "raju", age = "25"}, "email", "not-found")
"not-found"

> zipmap(["name", "age"], ["raju", "25"])
{ "age" = "25", "name" = "raju" }

> range(5)
tolist([0, 1, 2, 3, 4])

> range(1, 6)
tolist([1, 2, 3, 4, 5])
```

## Step 7 — Type Conversion Functions

```text
> tostring(42)
"42"

> tonumber("42")
42

> tobool("true")
true

> tolist(toset(["a", "b", "a"]))
tolist(["a", "b"])

> toset(["a", "b", "a", "c"])
toset(["a", "b", "c"])
```

## Step 8 — Encoding Functions

```text
> base64encode("hello terraform")
"aGVsbG8gdGVycmFmb3Jt"

> base64decode("aGVsbG8gdGVycmFmb3Jt")
"hello terraform"

> jsonencode({name = "raju", role = "admin"})
"{\"name\":\"raju\",\"role\":\"admin\"}"

> jsondecode("{\"name\":\"raju\"}")
{ "name" = "raju" }

> yamlencode({name = "raju", role = "admin"})
"name: raju\nrole: admin\n"
```

## Step 9 — Filesystem Functions

Create a sample file first (exit console):
```bash
echo "Hello from file" > sample.txt
terraform console
```

```text
> file("sample.txt")
"Hello from file\n"

> fileexists("sample.txt")
true

> fileexists("missing.txt")
false

> basename("/home/user/project/main.tf")
"main.tf"

> dirname("/home/user/project/main.tf")
"/home/user/project"
```

## Step 10 — Date/Time Functions

```text
> timestamp()
"2024-01-15T10:30:00Z"    # (your current time)

> formatdate("DD-MM-YYYY", timestamp())
"15-01-2024"               # (your current date)

> formatdate("hh:mm:ss", timestamp())
"10:30:00"                 # (your current time)
```

## Step 11 — Hash & UUID Functions

```text
> md5("hello")
"5d41402abc4b2a76b9719d911017c592"

> sha256("hello")
"2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824"

> uuid()
"a1b2c3d4-e5f6-7890-abcd-ef1234567890"   # (random each time)
```

## Step 12 — Network Functions

```text
> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"

> cidrsubnet("10.0.0.0/16", 8, 2)
"10.0.2.0/24"

> cidrhost("10.0.1.0/24", 5)
"10.0.1.5"
```

## Step 13 — Practical example using functions in config

Create `main.tf`:

```hcl
locals {
  project    = "nexskill"
  env        = "dev"
  raw_name   = "  My Terraform Project  "
  numbers    = [10, 20, 30, 40, 50]
  nested     = [["a", "b"], ["c", "d"], ["e"]]

  # Real-world usage
  clean_name    = trimspace(local.raw_name)
  resource_name = format("%s-%s-server", local.project, local.env)
  flat_list     = flatten(local.nested)
  max_number    = max(local.numbers...)
  min_number    = min(local.numbers...)
  timestamp     = formatdate("YYYY-MM-DD", timestamp())
}

output "resource_name" {
  value = local.resource_name
}

output "clean_name" {
  value = local.clean_name
}

output "flat_list" {
  value = local.flat_list
}

output "max_number" {
  value = local.max_number
}

output "timestamp" {
  value = local.timestamp
}
```

Run:
```bash
terraform plan
```

Expected output shows all computed values.

## Cleanup

No resources were created:

```bash
cd ..
rm -rf ~/tf-functions-demo
```

## Key Takeaways
- Terraform has **only built-in functions** — no user-defined.
- Use `terraform console` to quickly test any function.
- **Most used**: `length`, `element`, `lookup`, `merge`, `flatten`, `format`, `join`, `split`, `file`, `yamldecode`, `cidrsubnet`.
- Functions can be **combined**: `upper(trimspace("  hello  "))` → `"HELLO"`.
- `max(list...)` and `min(list...)` use the `...` splat to expand a list into arguments.
