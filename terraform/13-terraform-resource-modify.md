# Chapter 13 — Terraform: Resource Modify

## Objective
Change an existing resource by editing the config and applying again. See the difference between an **in-place change** and a **replacement**.

## Prerequisite from Chapter 12
The instance `sample-server` is running (created with `terraform apply`). Your terminal is in `aws-ec2` and credentials are loaded (`source ../.env`).

## Cost warning
Larger instance types cost money. Destroy after this practical (Chapter 14).

## Part A — Change the instance type (in-place update)

### Step 1 — Choose a new type
In the AWS Console → EC2 → **Launch instance** → open **Instance type** list and note another type. The video changed `t3.micro` to `t3.nano`. Close the page without launching.

### Step 2 — Edit `main.tf`
Change only this line:

```hcl
  instance_type = "t3.nano"
```
Save with **Ctrl + S**.

### Step 3 — Plan
```bash
terraform plan
```
**Expected:** a line like `~ instance_type = "t3.micro" -> "t3.nano"` and
```text
Plan: 0 to add, 1 to change, 0 to destroy.
```
The `~` symbol means **update in place**.

### Step 4 — Apply
```bash
terraform apply
```
Type `yes`.

**Expected:**
```text
Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```
During this change AWS stops the instance, changes the type and starts it again. In the console the instance may briefly show as stopping/stopped before running.

### Step 5 — Verify
EC2 → Instances → refresh → click the instance → **Instance type** should be `t3.nano`.

## Part B — Change the AMI (replacement)

### Step 1 — Get a different AMI ID
EC2 → **Launch instance** → pick a different OS image (for example another Amazon Linux or Ubuntu AMI) and copy its AMI ID. Close the page.

### Step 2 — Edit `main.tf`
Replace the `ami` value:

```hcl
  ami = "<NEW_AMI_ID>"
```
- `<NEW_AMI_ID>` → the new AMI ID you copied (must be from the same region).

Save.

### Step 3 — Apply
```bash
terraform apply
```
**Expected in the plan:** the `ami` line shows `# forces replacement`, and a summary:
```text
Plan: 1 to add, 0 to change, 1 to destroy.
```
The symbol `-/+` means destroy and create again. Type `yes`.

**What happens:** Terraform first destroys the old instance (status shutting down → terminated), then creates a new one.

### Step 4 — Verify
EC2 → Instances → refresh. The old instance is **Terminated** and a new one is **Running**.

**Why Terraform destroys first:** cloud resources are billed. Terraform avoids leaving duplicate machines running.

## How does Terraform know what to change?
It compares your `.tf` file with the **state file** (`terraform.tfstate`). See Chapter 15.

## Troubleshooting

### Error 1
`InvalidAMIID.NotFound`
**Reason:** New AMI ID is from another region.
**Fix:** Copy the AMI ID again while the console is in your Terraform region.

### Error 2
Instance type not supported for this AMI/architecture.
**Reason:** For example an ARM AMI with an Intel instance type.
**Fix:** Use an x86_64 AMI with `t2/t3` types.

## Final Result
You changed `instance_type` (1 changed) and `ami` (1 added, 1 destroyed) and confirmed both in the console.
