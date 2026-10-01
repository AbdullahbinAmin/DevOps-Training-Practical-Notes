# Chapter 12 — Terraform INIT, PLAN, APPLY (and DESTROY overview)

## Objective
Run the core Terraform workflow on the config from Chapter 11 and see an EC2 instance appear in AWS.

## Prerequisites from Chapter 11
`aws-ec2/main.tf` exists, and your terminal is inside `aws-ec2` with `source .env` already run.

## Pre-Check
```bash
terraform version
aws iam list-users
pwd
ls
```
**Expected:** Terraform version prints, AWS lists users, you are in `aws-ec2`, and `main.tf` is listed.

## Step 1 — `terraform init`
```bash
terraform init
```
**What this does:** prepares the folder. It downloads the AWS provider plugin and sets up the backend (where state is kept). Think of `git init`.

**Expected:**
```text
Terraform has been successfully initialized!
```

Look at the Explorer. New items appear:
- `.terraform/` folder → downloaded provider files
- `.terraform.lock.hcl` → records exact provider versions

## Step 2 — `terraform plan`
```bash
terraform plan
```
**What this does:** shows what Terraform **will** do, without changing anything. Read it: you will see `aws_instance.my_server will be created`, the `ami`, and many values marked `(known after apply)` (for example the public IP).

**Expected last line:**
```text
Plan: 1 to add, 0 to change, 0 to destroy.
```

## Step 3 — `terraform apply`
```bash
terraform apply
```
**What this does:** shows the plan again and asks for confirmation. Type:
```text
yes
```
**Expected:**
```text
aws_instance.my_server: Creating...
...
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

## Step 4 — Verify in the console
1. AWS Console → **EC2 → Instances**. Refresh.
2. You should see an instance named `sample-server`, **Running**, with the instance type and AMI you chose.

Compare it with the instance you created by hand in Chapter 10.

## Step 5 — Look at what was created in the folder
```bash
ls
```
A new file `terraform.tfstate` appears. This is the state file (explained in Chapter 15).

## Step 6 — (Idea) Multiple instances
You can create 3, 4 or 5 instances from the same config, and the config is written once. We will do this properly later with count/loops.

## Important
Do **not** destroy yet. Chapter 13 uses this same instance to show modification. Chapter 14 shows destroy.

## Troubleshooting

### Error 1
```text
Error: No valid credential sources found
```
**Reason:** `source .env` was not run in this terminal.
**Fix:**
```bash
source ../.env
terraform plan
```

### Error 2
```text
Error: creating EC2 Instance: InvalidAMIID.NotFound
```
**Reason:** The AMI ID belongs to a different region than the one in `provider "aws"`.
**Fix:** Copy the AMI ID again from the console in the same region as `region = "..."`.

### Error 3
```text
InvalidParameterCombination - The specified instance type is not eligible for Free Tier
```
**Reason:** Your account or region does not allow that type for free tier.
**Fix:** Use the type marked "Free tier eligible" in your console (`t2.micro` or `t3.micro`).

## Final Result
`terraform init`, `plan` and `apply` worked and the EC2 instance `sample-server` is running.
