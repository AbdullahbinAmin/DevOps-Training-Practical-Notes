# Chapter 25 — AWS EC2 + VPC

## Objective
Create an EC2 instance inside your own VPC using the subnet ID from Terraform.

## Prerequisite from Chapter 24
Folder `tf-aws/aws-vpc` with the VPC `main.tf`. It should be applied (VPC exists). Terminal in `aws-vpc` with credentials loaded.

## Step 1 — Add the EC2 resource
At the **end** of `aws-vpc/main.tf` add:

```hcl
# EC2 instance in the public subnet
resource "aws_instance" "my_server" {
  ami           = "<YOUR_AMI_ID>"
  instance_type = "<YOUR_INSTANCE_TYPE>"
  subnet_id     = aws_subnet.public_subnet.id

  tags = {
    Name = "sample-server"
  }
}
```
- `<YOUR_AMI_ID>` → an AMI ID from your region (see Chapter 11).
- `<YOUR_INSTANCE_TYPE>` → for example `t3.micro`.

**What's new:** `subnet_id` tells AWS in which subnet (and so in which VPC) to create the instance. Without it the instance goes into the default VPC.

## Step 2 — Apply
```bash
terraform apply
```
Type `yes`.

**Expected:** `Resources: 1 added` (the VPC parts already exist).

## Step 3 — Verify
Console → EC2 → Instances → open `sample-server`.

Check:
- **VPC ID** = your `my-vpc`.
- **Subnet ID** = `public-subnet`.
- **Private IPv4 address** is in `10.0.2.x` (the public subnet range).
- **Public IPv4 address** is **empty**. Reason: we did not ask for one. Subnets created by Terraform do not auto-assign public IPs. We fix this in Chapter 26 with `associate_public_ip_address = true`.

If you had used the private subnet, the IP would be `10.0.1.x`.

## Troubleshooting

### Error 1
```text
InvalidAMIID.NotFound
```
**Fix:** Use an AMI from the same region as the provider.

## Cleanup
Chapter 26 uses a fresh project. You can destroy this now:
```bash
terraform destroy
```

## Final Result
An EC2 instance is created inside the VPC that Terraform built, in the public subnet range.
