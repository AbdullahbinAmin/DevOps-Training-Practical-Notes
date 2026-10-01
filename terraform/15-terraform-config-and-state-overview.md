# Chapter 15 — Terraform Config and State Overview

## Objective
Understand the configuration language (HCL, JSON), and understand what the **state file** is, where it lives, and why we should not keep it only on a laptop.

## Prerequisites
- Folder `aws-ec2` with `main.tf` from Chapter 11.
- To see a real state file, you need an applied resource. If you destroyed everything in Chapter 14, run `terraform apply` again (type `yes`) and destroy at the end of this chapter.

## Part 1 — Configuration language

### What is HCL?
- HCL = **HashiCorp Configuration Language**. Terraform's own language. Easy to read.
- It is **declarative**: you write the final state you want, and Terraform decides the steps.

Look at your `main.tf`. You only wrote: "I want an EC2 instance, with this OS and this size." You did not write the steps to create it.

### JSON is also supported
Terraform can read JSON files that end with `.tf.json`. HCL is cleaner; JSON is useful when programs generate the config.

Same provider settings in JSON (example file name `provider.tf.json`):

```json
{
  "terraform": {
    "required_providers": {
      "aws": {
        "source": "hashicorp/aws",
        "version": "~> 5.0"
      }
    }
  },
  "provider": {
    "aws": {
      "region": "<YOUR_REGION>"
    }
  }
}
```
- `<YOUR_REGION>` → for example `eu-north-1`.

Note: this is for showing the difference only. Do not keep both `provider` definitions in one folder or Terraform will complain about duplicates. In class, show it in a temporary folder or read-only.

## Part 2 — State management

### Step 1 — Apply the config (if needed)
```bash
cd aws-ec2
source ../.env
terraform apply
```
Type `yes`.

### Step 2 — See the state file
In the VS Code Explorer you see `terraform.tfstate`. After changes you may also see `terraform.tfstate.backup`.

```bash
ls
```
**Expected:** `terraform.tfstate` is listed.

### Step 3 — Read the state safely
```bash
terraform state list
```
**What this does:** lists resources Terraform is tracking.
**Expected:**
```text
aws_instance.my_server
```

```bash
terraform show
```
**What this does:** prints the current state in a readable way.

Optionally open `terraform.tfstate` in VS Code to show that it is a JSON file containing the instance ID, AMI, IP addresses and so on.

### Step 4 — Why state matters
Remember Chapter 13. When you changed the AMI, Terraform knew an instance already existed and so it destroyed the old one before creating a new one. It knew this **from the state file**.

## Part 3 — Local vs remote state

| Local state | Remote state |
|---|---|
| File lives on your laptop | File lives in cloud storage (for example an S3 bucket) |
| If laptop crashes or folder is deleted, you lose the state and Terraform forgets your infrastructure | Safe and available |
| Team members cannot see it | Team can share it (collaboration) |

Remote state is done in **Chapter 20**.

## Rules for the state file
1. Do not edit it by hand.
2. Do not delete it while resources exist.
3. Do not upload it to public Git (it can contain sensitive values).

## Troubleshooting

### Error 1
```text
No state file was found!
```
**Reason:** You ran `terraform show` before `apply`, or in the wrong folder.
**Fix:** `cd aws-ec2` and run `terraform apply` first.

## Cleanup
```bash
terraform destroy
```
Type `yes`.

## Final Result
You can explain HCL vs JSON, show the state file, and explain why remote state is better.
