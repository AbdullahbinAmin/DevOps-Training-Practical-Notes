# Chapter 14 — Terraform: Resource Delete (and `validate`)

## Objective
Destroy everything created by the config, and use `terraform validate` to check your files for errors.

## Prerequisite from Chapter 13
An instance exists from `aws-ec2`. Terminal is in `aws-ec2` and credentials are loaded.

## Part A — Destroy

### Step 1 — Run destroy
```bash
terraform destroy
```
**What this does:** deletes all resources tracked in the state file.

**Expected message:**
```text
Do you really want to destroy all resources?
Terraform will destroy all your managed infrastructure...
```
Type:
```text
yes
```

### Step 2 — Verify
**Expected:**
```text
Destroy complete! Resources: 1 destroyed.
```
Console → EC2 → Instances → refresh. The instance is **Shutting-down**, then **Terminated**.

Tip: `terraform apply -auto-approve` and `terraform destroy -auto-approve` skip the yes/no question. Use only when you are sure.

## Part B — Summary of commands so far

| Command | Purpose |
|---|---|
| `terraform init` | Set up folder, download providers (first time in a folder) |
| `terraform plan` | Preview changes |
| `terraform apply` | Create/change resources |
| `terraform destroy` | Delete resources |
| `terraform validate` | Check config syntax |

## Part C — `terraform validate`

### Step 1 — Validate a good file
```bash
terraform validate
```
**Expected:**
```text
Success! The configuration is valid.
```

### Step 2 — Make an intentional mistake
In `main.tf`, change `provider "aws"` to `provide "aws"` (remove the `r`). Save.

```bash
terraform validate
```
**Expected:** an error like `Unsupported block type` that points to the file and line.

### Step 3 — Fix it
Change it back to `provider "aws"`, save, run `terraform validate` again. **Expected:** Success.

## Troubleshooting

### Error 1
```text
Error: Unsupported block type
```
**Reason:** A block name is spelled wrong.
**Fix:** Read the file and line number in the error. Correct the spelling.

## Final Result
All resources destroyed, and you can use `validate` to catch mistakes before `plan`.
