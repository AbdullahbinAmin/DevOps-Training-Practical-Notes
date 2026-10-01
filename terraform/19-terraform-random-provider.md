# Chapter 19 — Terraform: Random Provider

## Objective
Use the `random` provider to generate a random ID so the S3 bucket name is always unique, without guessing names.

## Prerequisite from Chapter 18
Folder `tf-aws/s3` with `main.tf` and `myfile.txt`. Bucket is applied (or at least the config exists). Terminal is in `s3` and `source ../.env` was run.

## Why the random provider?
Bucket names must be globally unique. Trying names like `abcd12345` again and again is trial and error. The random provider creates values such as random IDs, random passwords, shuffled strings and UUIDs automatically.

Official page: search `random` provider in the Registry: **https://registry.terraform.io/providers/hashicorp/random/latest**.

## Step 1 — Add the provider
Update the top `terraform` block in `main.tf` to include `random` as well:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}
```
⚠️ Verification Required: `~> 3.0` is an example. Use the version shown on the Registry page if you prefer.

The `provider "random" {}` block is not needed (the video also skipped it).

## Step 2 — Add the `random_id` resource
Add below the `provider "aws"` block:

```hcl
resource "random_id" "rand_id" {
  byte_length = 8
}
```
`byte_length = 8` means 8 random bytes. In hex format that gives 16 characters.

## Step 3 — Add an output to see it
```hcl
output "random_id" {
  value = random_id.rand_id.hex
}
```
Other formats available on the same resource: `dec`, `hex`, `b64_url`, `b64_std`, `id`. Try `hex` first.

## Step 4 — Initialize again (new provider added)
```bash
terraform init
```
**Expected:** it downloads the `hashicorp/random` provider. Message: `Terraform has been successfully initialized!`

**Why:** every time you add a new provider you must run `init` again.

## Step 5 — Apply and observe
```bash
terraform apply
```
Type `yes`. **Expected:** one more resource added and an output like:
```text
random_id = "3f9a1c0b7d2e4a55"
```
(Value will be different.) Optional: change `.hex` to `.b64_url` and apply again to see a different format. Then set it back to `.hex`.

## Step 6 — Use the random ID in the bucket name
Change the bucket resource:

```hcl
resource "aws_s3_bucket" "demo_bucket" {
  bucket = "demo-bucket-${random_id.rand_id.hex}"
}
```
**Explanation:** `${ ... }` inserts a value inside a string. Here the final name will be `demo-bucket-` + random hex. Bucket name length stays within 63 characters.

## Step 7 — Apply
```bash
terraform apply
```
**Expected in plan:** the bucket has `# forces replacement` (the name changed). The object is also replaced because it depends on the bucket. Type `yes`.

**Expected:**
```text
Apply complete! ...  (some added, some destroyed)
```
Terraform destroys the old bucket first and then creates the new one.

## Step 8 — Verify
S3 → refresh. You will see a bucket named like `demo-bucket-<random hex>` containing `mydata.txt`. Open it — `Hello World`.

**Write down this full bucket name.** You will use it in Chapter 20.

## Troubleshooting

### Error 1
```text
Error: Inconsistent dependency lock file / provider requirements have changed
```
**Reason:** You added `random` but did not run `terraform init`.
**Fix:** `terraform init`

### Error 2
```text
Error: Reference to undeclared resource "random_id.rand_id"
```
**Reason:** The resource name in the bucket line and the resource block do not match.
**Fix:** Make sure `resource "random_id" "rand_id"` and `random_id.rand_id.hex` use the same name.

## Cleanup
**Do NOT destroy this bucket yet.** Chapter 20 uses it to store Terraform state. (If you already destroyed it, run `terraform apply` again to recreate it.)

## Final Result
Bucket name is generated automatically and always unique.
