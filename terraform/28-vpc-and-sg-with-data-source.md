# Chapter 28 — VPC & SG with Data Source

## Objective
Read an **existing security group**, an **existing VPC**, the **availability zones**, the **account details** and the **region** using data sources.

## Prerequisites from Chapter 27
Folder `tf-data-sources` with `provider` and `main.tf`. `source ../.env` done.

## Create the existing resources first (GUI)
Data sources only read things that already exist. Create these in the **same region** as your provider.

**A) Security group**
1. EC2 → **Security Groups** → **Create security group**.
2. **Name:** `my-web-server-group`. **Description:** `Allow HTTP`. **VPC:** any VPC (default is fine for this chapter).
3. **Inbound rules → Add rule:** Type `HTTP`, Source `Anywhere-IPv4`.
4. **Outbound rules:** keep **All traffic**.
5. **Tags → Add new tag:** Key `my-web-server`, Value `http` (you can choose other tags; use the same ones in the code below).
6. **Create security group**.

**B) VPC with tags**
1. VPC → **Your VPCs** → **Create VPC** → **VPC only**.
2. **Name tag:** `my-vpc`. **IPv4 CIDR:** `10.0.0.0/16`.
3. **Tags → Add new tag:** Key `Env`, Value `prod`.
4. **Create VPC**.

(If you kept the Terraform VPC from an earlier chapter, you can add these tags there.)

## Step 1 — Security group data source
Add to `main.tf`:

```hcl
data "aws_security_group" "my_sg" {
  tags = {
    "my-web-server" = "http"
  }
}

output "security_group_id" {
  value = data.aws_security_group.my_sg.id
}
```
**Explain:** Terraform searches your account in the current region for a security group with this tag. Tags are the easiest way to find exactly one.

```bash
terraform plan
```
**Expected:** output shows the security group ID (`sg-...`). Compare it with the console.

If you use `value = data.aws_security_group.my_sg` (whole object) you see many fields: description, VPC ID, name, tags, ID. You can print only what you need, for example `.vpc_id`, `.name`, `.id`.

## Step 2 — VPC data source
```hcl
data "aws_vpc" "my_vpc" {
  tags = {
    Env  = "prod"
    Name = "my-vpc"
  }
}

output "vpc_id" {
  value = data.aws_vpc.my_vpc.id
}
```
We gave two tags because `prod` may have more than one VPC.

```bash
terraform plan
```
**Expected:** `vpc_id = "vpc-..."`. Verify it in VPC → Your VPCs.

**Note:** data sources search **in the provider's region and account**. If the VPC lives in another region, the search fails.

## Step 3 — Availability zones
```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

output "zones" {
  value = data.aws_availability_zones.available.names
}
```
`names` is a **list** (plural), because there are many zones.

```bash
terraform plan
```
**Expected:** a list such as `eu-north-1a`, `eu-north-1b`, `eu-north-1c` (yours depends on the region).

## Step 4 — Account details (who is running Terraform)
```hcl
data "aws_caller_identity" "current" {}

output "caller_info" {
  value = {
    account_id = data.aws_caller_identity.current.account_id
    user_id    = data.aws_caller_identity.current.user_id
    arn        = data.aws_caller_identity.current.arn
  }
}
```
**Expected:** your account ID, and your IAM user (for example `tf-user`) ARN. Useful for logs and information.

## Step 5 — Current region details
```hcl
data "aws_region" "current" {}

output "region_name" {
  value = data.aws_region.current.name
}
```
**Expected:** your region name (older provider versions print the code such as `eu-north-1`).

⚠️ Verification Required: in newer AWS provider versions the attribute used for the region code may differ (`id` or `region` instead of `name`). If you see a deprecation warning or error, read the `aws_region` data source page in the docs.

## Step 6 — Run everything
```bash
terraform validate
terraform plan
```
**Expected:** `Changes to Outputs:` with all outputs listed, and no resources to create.

## Troubleshooting

### Error 1
```text
Error: no matching EC2 Security Group found
```
**Reason:** Tag key/value or region is different.
**Fix:** Open the security group in the console → **Tags** tab. Copy the exact key and value into the code.

### Error 2
```text
multiple VPCs matched
```
**Reason:** Your tags match more than one VPC.
**Fix:** Add more tags (for example `Name`).

### Error 3
```text
Error: Unsupported block type / Unsupported argument
```
**Reason:** A small typo (for example `data` written twice: `data.data.`).
**Fix:** Reference format is `data.<type>.<name>.<attribute>` — write `data` once.

## Cleanup
Nothing was created by Terraform. Delete the security group and VPC you made manually if you do not need them (keep them if you continue to Chapter 29).

## Final Result
Terraform reads the SG, VPC, zones, caller identity and region without creating anything.
