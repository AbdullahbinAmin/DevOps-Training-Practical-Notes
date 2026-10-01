# Chapter 4 — Terraform Installation (Mac)

## Objective
Install Terraform on macOS with Homebrew and choose a cloud provider for the course.

## Prerequisites
- OS: macOS
- Homebrew installed
- Terminal app

## Official Installation Guide
**https://developer.hashicorp.com/terraform/install** → choose **macOS**.

## Step 1 — Check Homebrew
```bash
brew --version
```
**Expected:** prints a Homebrew version.

If it says `command not found`, install Homebrew from its official site **https://brew.sh** (copy the one install command shown there), then run the check again.

## Step 2 — Add the HashiCorp tap
```bash
brew tap hashicorp/tap
```
**What this does:** adds the official HashiCorp package repository to Homebrew.
**Expected:** the command finishes without error.

## Step 3 — Install Terraform
```bash
brew install hashicorp/tap/terraform
```
**What this does:** downloads and installs Terraform.

## Step 4 — Verify
```bash
terraform version
```
**Expected:** `Terraform v1.x.x` and your platform (for example `darwin_arm64` or `darwin_amd64`).

You can also run:
```bash
terraform -help
```
**Expected:** a list of Terraform commands.

## Step 5 — Decide the cloud provider (class discussion)
Terraform works with many providers. Go to **https://registry.terraform.io** → **Browse → Providers**. You will see AWS, Azure, Google Cloud, Oracle and many more. Use the category filter (for example **Database**) to see partner providers.

**In this course we use AWS.**

## Troubleshooting

### Error 1
```text
zsh: command not found: terraform
```
**Reason:** Install did not finish, or the terminal was open before install.
**Fix:**
```bash
brew install hashicorp/tap/terraform
```
Then open a new terminal window and run `terraform version`.

## Final Result
`terraform version` works in Terminal.
