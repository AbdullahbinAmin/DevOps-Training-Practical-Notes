# Chapter 11 — First Terraform Config to Create an EC2 Instance

## Objective
Write your first `main.tf` that creates an EC2 instance. (Running it is done in Chapter 12.)

## Prerequisites
- Chapter 3/4 (Terraform installed)
- Chapter 9 (`.env` file with AWS keys in `tf-aws`)
- Your **AMI ID** and **instance type** from Chapter 10
- Your **region code** (for example `eu-north-1`)

## Architecture / Flow
```text
main.tf (terraform block → provider block → resource block)
```

## Step 1 — Create the project folder and file
In VS Code Explorer, inside `tf-aws`, create a folder `aws-ec2`. Inside it create a file named `main.tf`.

**Why `.tf`:** Terraform reads files that end with `.tf`.

## Step 2 — Install the Terraform VS Code extension
1. Click the **Extensions** icon (left bar).
2. Search `HashiCorp Terraform`.
3. Install the one published by **HashiCorp**.

**Why:** gives auto-complete, colours and formatting for `.tf` files.

## Step 3 — Find your region code
AWS Console → look at the top-right region name. Click it to see the code (example: Stockholm = `eu-north-1`). Use **your** region.

## Step 4 — Find your AMI ID and instance type
Console → EC2 → **Launch instance**. The AMI ID appears under the OS image name; the instance type is under **Instance type**. Copy them. (Do **not** click launch. Just close the page.)

**Important:** AMI ID is different for every region.

## Step 5 — Write `main.tf`

Open the official AWS provider page for reference: **https://registry.terraform.io/providers/hashicorp/aws/latest** → click **Use Provider** to see the standard code.

Put this in `aws-ec2/main.tf`:

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
  instance_type = "<YOUR_INSTANCE_TYPE>"

  tags = {
    Name = "sample-server"
  }
}
```

Replace the placeholders:
- `<YOUR_REGION>` → your region code, for example `eu-north-1`.
- `<YOUR_AMI_ID>` → the AMI ID from Step 4, for example `ami-0123456789abcdef0` (yours will be different).
- `<YOUR_INSTANCE_TYPE>` → for example `t3.micro` (must be available in your region).

⚠️ Verification Required: `~> 5.0` is an example provider version. You may use the version shown on the Registry page.

## Explanation of each block
- `terraform { required_providers { ... } }` → tells Terraform which provider plugin to download. `source = "hashicorp/aws"` is the important part. The local name `aws` is only a name (you could call it anything) but `aws` is easiest to read.
- `provider "aws" { region = ... }` → settings that apply to all AWS resources in this folder. Region is a high level setting, so it goes here.
- `resource "aws_instance" "my_server"` → `aws_instance` is the resource type (an EC2 instance). `my_server` is the name of this block inside Terraform (not the name in AWS).
- `ami` → operating system image. `instance_type` → CPU/RAM size.
- `tags` → optional labels. `Name` tag becomes the name you see in the console.

Save with **Ctrl + S**.

## Step 6 — Move to the folder and load credentials
Open the terminal (Terminal → New Terminal):

```bash
source .env
```
(run from `tf-aws`)

Then:
```bash
cd aws-ec2
```
Check you are in the right place:
```bash
ls
```
**Expected:** `main.tf` is listed. (On Windows Command Prompt use `dir`.)

## Final Result
`aws-ec2/main.tf` is ready and your terminal is in the `aws-ec2` folder with AWS credentials loaded. Continue to **Chapter 12**.
