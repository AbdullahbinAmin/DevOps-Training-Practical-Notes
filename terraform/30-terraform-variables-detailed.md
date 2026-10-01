# Chapter 30 — Terraform Variables: Types, Validation, ENV, tfvars, auto.tfvars, Locals

## Objective
Learn every common way to define and provide variable values: `type`, `default`, `validation`, `object`, `map`, environment variables (`TF_VAR_`), `terraform.tfvars`, `*.auto.tfvars`, command line `-var`, and `locals`.

## Prerequisites
- `.env` in `tf-aws` (credentials)
- Region code, AMI ID (from your region)
- You will only run `terraform plan` (no resources need to be created) except where noted. If you `apply`, run `terraform destroy` afterwards.

## Variable value priority (low → high)
```text
1. default value in variable block   (lowest)
2. Environment variable  TF_VAR_<name>
3. terraform.tfvars
4. *.auto.tfvars   (files are loaded in alphabetical order)
5. -var / -var-file on the command line   (highest)
```
Later ones override earlier ones. ⚠️ Verification Required: confirm the order in the official page "Terraform → Language → Variables".

## Step 1 — Folder and starting config
```bash
source .env
mkdir tf-variable
cd tf-variable
```

Create `main.tf` with **hard-coded** values:

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

resource "aws_instance" "my_server" {
  ami           = "<YOUR_AMI_ID>"
  instance_type = "t3.micro"

  root_block_device {
    volume_size = 30
    volume_type = "gp2"
  }

  tags = {
    Name = "sample-server"
  }
}
```
- `<YOUR_REGION>` → for example `eu-north-1`.
- `<YOUR_AMI_ID>` → an AMI ID from that region.

`root_block_device` sets the root disk: `volume_size` in GB and `volume_type` (`gp2`, `gp3`...).

**Problem:** `t3.micro` is hard-coded. Testing may need a small machine and production may need a large one, so the value must be changeable.

```bash
terraform init
```

## Step 2 — Create a variable for the instance type
Create `variables.tf`:

```hcl
variable "instance_type" {
  description = "What type of instance you want to create"
}
```
In `main.tf` replace the hard-coded value:

```hcl
  instance_type = var.instance_type
```

Run:
```bash
terraform plan
```
**Expected:** Terraform **asks** in the terminal: `var.instance_type  What type of instance you want to create`. Type `t3.micro` and press Enter.

**Result:** Plan works.

## Step 3 — Add a `type`
Without `type`, a wrong value like `t3.micr` is accepted by `plan` and only fails at `apply` (AWS says it is not valid).

Update the variable:

```hcl
variable "instance_type" {
  description = "What type of instance you want to create"
  type        = string
}
```
`type = string` → the value must be a string. Other types: `number`, `bool`, `list(...)`, `map(...)`, `object({...})`.

## Step 4 — Number variable with default
Add to `variables.tf`:

```hcl
variable "root_volume_size" {
  description = "Root volume size in GB"
  type        = number
  default     = 20
}

variable "root_volume_type" {
  description = "Root volume type"
  type        = string
  default     = "gp2"
}
```
In `main.tf`:

```hcl
  root_block_device {
    volume_size = var.root_volume_size
    volume_type = var.root_volume_type
  }
```
Test the `number` type: temporarily remove the `default` line for `root_volume_size`, run `terraform plan`, and when it asks, type `abc`.

**Expected:** an error such as `a number is required`. Put the `default = 20` back. With a default, Terraform does not ask for this value anymore.

```bash
terraform plan
```
**Expected:** asks only for `instance_type`.

## Step 5 — Validation
Stop wrong values (for example a huge expensive instance). Update the `instance_type` variable:

```hcl
variable "instance_type" {
  description = "What type of instance you want to create"
  type        = string

  validation {
    condition     = var.instance_type == "t2.micro" || var.instance_type == "t3.micro"
    error_message = "Only t2.micro and t3.micro are allowed."
  }
}
```
Test:
```bash
terraform plan
```
Type `t3.large`.

**Expected:**
```text
Error: Invalid value for variable
Only t2.micro and t3.micro are allowed.
```
Run again and type `t3.micro`. **Expected:** plan works.

## Step 6 — Combine related variables into one `object`
`root_volume_size` and `root_volume_type` belong together (both are root disk settings). Replace both variables with one object variable in `variables.tf`:

```hcl
variable "ec2_config" {
  description = "Root volume configuration"

  type = object({
    v_size = number
    v_type = string
  })

  default = {
    v_size = 20
    v_type = "gp2"
  }
}
```
(Delete the two old variables `root_volume_size` and `root_volume_type`.)

Update `main.tf`:

```hcl
  root_block_device {
    volume_size = var.ec2_config.v_size
    volume_type = var.ec2_config.v_type
  }
```
**Important:** `default` needs an equal sign: `default = { ... }`. Without `=` you get `Unsupported block type`.

```bash
terraform plan
```
Type `t3.micro`. **Expected:** volume size `20`, type `gp2`.

## Step 7 — `map` variable for extra tags
Add to `variables.tf`:

```hcl
variable "additional_tags" {
  description = "Extra tags for the instance"
  type        = map(string)
  default     = {}
}
```
`map(string)` = key = value pairs where the value is a string. The default is empty `{}`.

Update `main.tf` tags with `merge`:

```hcl
  tags = merge(var.additional_tags, {
    Name = "sample-server"
  })
```
`merge` joins the extra tags with the default `Name` tag.

```bash
terraform plan
```
Type `t3.micro`. **Expected:** only 1 tag (`Name`). Extra tags will appear in Step 9.

## Step 8 — Environment variable
Format: `TF_VAR_<variable_name>`.

**Mac / Linux / Git Bash:**
```bash
export TF_VAR_instance_type="t3.micro"
```
**PowerShell:**
```powershell
$env:TF_VAR_instance_type="t3.micro"
```

```bash
terraform plan
```
**Expected:** Terraform does **not** ask for `instance_type`, and the plan shows `t3.micro`.

**Prove that it comes from the environment:** change the variable to `t2.micro`:
```bash
export TF_VAR_instance_type="t2.micro"
terraform plan
```
**Expected:** plan now shows `t2.micro`.

Environment variables are useful for sensitive values (passwords, keys).

## Step 9 — `terraform.tfvars` (best for a full set of values)
Use it when you have a full set of values for one environment (for example testing). Create `terraform.tfvars` (exact name):

```hcl
instance_type = "t3.micro"

ec2_config = {
  v_size = 30
  v_type = "gp3"
}

additional_tags = {
  department = "QA"
  project    = "my-project-QA"
}
```
Note: if you write `t3.small` here, the validation from Step 5 will reject it. Use an allowed value or change the validation.

```bash
terraform plan
```
**Expected:** no questions. The plan shows `volume_size = 30`, `volume_type = "gp3"`, and **3 tags** (`Name`, `department`, `project`). `terraform.tfvars` overrides defaults and environment variables (for `instance_type` it uses the tfvars value).

## Step 10 — `*.auto.tfvars` (extra override, e.g. production)
Create `prod.auto.tfvars` (any name + `.auto.tfvars`):

```hcl
ec2_config = {
  v_size = 40
  v_type = "gp3"
}
```
```bash
terraform plan
```
**Expected:** `volume_size = 40`. The `.auto.tfvars` file is loaded automatically and has **higher priority** than `terraform.tfvars`. The other values still come from `terraform.tfvars`.

## Step 11 — Command line `-var` (highest priority)
```bash
terraform plan -var='ec2_config={v_size=50,v_type="gp2"}'
```
**Expected:** `volume_size = 50` and `volume_type = "gp2"`. Single quotes wrap the value; the double quotes inside are for the string.

**Windows note:** quoting differs in Command Prompt/PowerShell. In PowerShell you may need `--%` or different quote styles. Alternative that works everywhere: use `-var-file=<file>`, for example `terraform plan -var-file="custom.tfvars"`. ⚠️ Verification Required for your shell.

## Step 12 — Locals (beginning)
Locals are constants inside the config. They are useful when a value repeats many times or is complicated. **You cannot pass locals from outside.**

Add to `main.tf`:

```hcl
locals {
  owner = "abc"
  name  = "my-server"
}
```
Use it:

```hcl
  tags = merge(var.additional_tags, {
    Name = local.name
  })
```
Note: `local.` (singular) when using, `locals` (plural) when defining.

```bash
terraform validate
terraform plan
```
**Expected:** valid and the `Name` tag becomes `my-server`. (More about locals will come in the next chapter file. ⚠️ The transcript ended here, so only this beginning is covered.)

## Troubleshooting

### Error 1
```text
Error: Unsupported block type — Default or Type is not expected here
```
**Reason:** `default { ... }` was written without an equal sign.
**Fix:** `default = { ... }`

### Error 2
```text
Error: Invalid value for variable ... Only t2.micro and t3.micro are allowed.
```
**Fix:** Use an allowed instance type or update the validation condition.

### Error 3
```text
Error: Value for undeclared variable
```
**Reason:** The name in `terraform.tfvars` does not match a `variable` block.
**Fix:** Spell the variable name exactly the same in both files.

### Error 4
```text
Error: Inconsistent conditional / Invalid value for input variable (object)
```
**Reason:** The object keys are wrong (`v_size`, `v_type`) or types do not match (`number` vs string).
**Fix:** Use exactly `v_size` (number) and `v_type` (string).

## Cleanup
This chapter only used `plan`. If you ran `apply`, run `terraform destroy`.

To clear the environment variable:
```bash
unset TF_VAR_instance_type
```
(PowerShell: `Remove-Item Env:TF_VAR_instance_type`)

## Final Result
You can supply values using default, environment variable, `terraform.tfvars`, `*.auto.tfvars`, `-var`, and you can validate values with `type` and `validation`.

## Chapter files at a glance
```text
tf-variable/
├── main.tf
├── variables.tf
├── terraform.tfvars
└── prod.auto.tfvars
```
