# Chapter 16 — Terraform Variables & Outputs (Basics)

## Objective
Remove hard-coded values using a **variable**, split code into multiple files, and print useful information using an **output**.

## Prerequisites from Chapter 11
`aws-ec2/main.tf` exists and works. Terminal in `aws-ec2` with `source ../.env` done.

## Architecture / Flow
```text
variables.tf (region) ──> main.tf (var.region)
main.tf (aws_instance) ──> outputs.tf (public IP printed after apply)
```

## Part A — Variables

### Why
If the same value (for example region) is used in 5 places, changing it means editing 5 places. With a variable you change it once.

### Step 1 — Create `variables.tf`
In `aws-ec2` create a file named `variables.tf`:

```hcl
variable "region" {
  description = "Value of region"
  type        = string
  default     = "<YOUR_REGION>"
}
```
- `<YOUR_REGION>` → your region code, for example `eu-north-1`.

**Explanation:** `region` is the variable name. `description` explains it. `type` says what kind of value (string, number, ...). `default` is used if no other value is given.

### Step 2 — Use it in `main.tf`
Change the provider block:

```hcl
provider "aws" {
  region = var.region
}
```
`var.region` means "use the value of variable `region`".

### Step 3 — Validate
```bash
terraform validate
```
**Expected:** `Success! The configuration is valid.`

### Note on multiple files
Terraform reads **all** `.tf` files in the same folder (where you ran `terraform init`). You do not need any import statement. Only rule: files must be in the same folder. You can also split the code: `provider.tf`, `resources.tf`, etc.

### Step 4 — Apply
```bash
terraform apply
```
Type `yes`.

**Expected:** resource created (`1 added`). Check the console: instance `sample-server` is running.

## Part B — Outputs

### Why
After the instance is created, we want its public IP printed in the terminal so we do not need to open the console.

### Step 1 — Create `outputs.tf`
```hcl
output "instance_public_ip" {
  description = "Public IP of the instance"
  value       = aws_instance.my_server.public_ip
}
```
**Explanation:** `aws_instance.my_server.public_ip` = `<resource type>.<block name>.<attribute>`. When you type a dot after `aws_instance.my_server`, VS Code lists available attributes (for example `public_dns`, `public_ip`, `id`).

### Step 2 — Apply
```bash
terraform apply
```
Type `yes`.

**Expected at the end:**
```text
Outputs:

instance_public_ip = "x.x.x.x"
```

### Step 3 — Verify against the console
EC2 → Instances → click the instance → **Public IPv4 address**. It must match the output.

### Step 4 — Show the outputs any time
```bash
terraform output
```
**Expected:** the same output values.

## Troubleshooting

### Error 1
```text
Error: Reference to undeclared resource
```
**Reason:** The name in `outputs.tf` does not match the resource block name in `main.tf`.
**Fix:** Confirm the resource is `resource "aws_instance" "my_server"` and the output uses `aws_instance.my_server`.

### Error 2
```text
Error: Reference to undeclared input variable
```
**Reason:** `variables.tf` is missing or is not in the same folder.
**Fix:** Check the file name and folder.

## Cleanup
```bash
terraform destroy
```
Type `yes`.

## Final Result
Region comes from a variable, and after apply the terminal prints the public IP.
