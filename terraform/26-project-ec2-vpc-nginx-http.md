# Chapter 26 — Project: EC2 + VPC + NGINX + HTTP Access

## Objective
Build a complete project with clean file structure: your own VPC, a public EC2 instance, NGINX installed automatically, a security group that allows HTTP, and the public URL printed in the terminal.

## Prerequisites
- Chapters 24 and 25 (VPC and EC2 in Terraform)
- `.env` in `tf-aws`
- Region code, AMI ID, instance type

**AMI choice is important:** use an **Amazon Linux 2023** AMI (the script uses `yum install nginx`). On the older Amazon Linux 2, `nginx` is not in the normal repository and the install can fail. ⚠️ Verification Required: check the AMI name in the console before using.

## Architecture / Flow
```text
Browser ──HTTP:80──> Internet Gateway ──> Route table ──> Public subnet ──> EC2 (NGINX)
                                                        Security group allows port 80
```

## Good practice used here
Split the config into several files (Terraform reads all of them):

```text
aws-vpc-ec2-nginx/
├── main.tf              (terraform block + required providers)
├── providers.tf         (provider aws)
├── vpc.tf               (VPC, subnets, IGW, route table, association)
├── security_groups.tf   (firewall rules)
├── ec2.tf               (instance + NGINX script)
└── outputs.tf           (public IP + URL)
```

## Step 1 — Create the folder
```bash
source .env
mkdir aws-vpc-ec2-nginx
cd aws-vpc-ec2-nginx
```

## Step 2 — `main.tf`
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

## Step 3 — `providers.tf`
```hcl
provider "aws" {
  region = "<YOUR_REGION>"
}
```
- `<YOUR_REGION>` → for example `eu-north-1`.

## Step 4 — `vpc.tf`
```hcl
resource "aws_vpc" "my_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "my-vpc"
  }
}

resource "aws_subnet" "private_subnet" {
  vpc_id     = aws_vpc.my_vpc.id
  cidr_block = "10.0.1.0/24"

  tags = {
    Name = "private-subnet"
  }
}

resource "aws_subnet" "public_subnet" {
  vpc_id     = aws_vpc.my_vpc.id
  cidr_block = "10.0.2.0/24"

  tags = {
    Name = "public-subnet"
  }
}

resource "aws_internet_gateway" "my_igw" {
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "my-igw"
  }
}

resource "aws_route_table" "my_rt" {
  vpc_id = aws_vpc.my_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.my_igw.id
  }

  tags = {
    Name = "my-rt"
  }
}

resource "aws_route_table_association" "public_subnet_asso" {
  route_table_id = aws_route_table.my_rt.id
  subnet_id      = aws_subnet.public_subnet.id
}
```

## Step 5 — `security_groups.tf`
```hcl
resource "aws_security_group" "nginx_sg" {
  vpc_id = aws_vpc.my_vpc.id

  # Inbound: allow HTTP from anywhere
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Outbound: allow everything
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "nginx-sg"
  }
}
```

**Explain the fields:**
- `ingress` = inbound (traffic coming in). `egress` = outbound (traffic going out). Use **blocks**, not `=`.
- `from_port` / `to_port` = port range. HTTP is port `80`, and NGINX listens on port 80 by default. Same value for both because we need only one port.
- `protocol = "tcp"` → HTTP runs over TCP.
- `cidr_blocks = ["0.0.0.0/0"]` → requests from any IP address (quotes and square brackets are needed).
- Egress `0`, `0`, `-1` → all ports, all protocols (`-1` means all), to all IPs. The instance needs outbound access to download NGINX.
- `tags.Name` is the label you see in the console. The block name (`nginx_sg`) is only for Terraform.

## Step 6 — `ec2.tf`
```hcl
resource "aws_instance" "nginx_server" {
  ami                         = "<YOUR_AMI_ID>"
  instance_type               = "<YOUR_INSTANCE_TYPE>"
  subnet_id                   = aws_subnet.public_subnet.id
  vpc_security_group_ids      = [aws_security_group.nginx_sg.id]
  associate_public_ip_address = true

  user_data = <<-EOF
    #!/bin/bash
    yum install nginx -y
    systemctl enable nginx
    systemctl start nginx
  EOF

  depends_on = [aws_route_table_association.public_subnet_asso]

  tags = {
    Name = "nginx-server"
  }
}
```
Replace:
- `<YOUR_AMI_ID>` → Amazon Linux 2023 AMI ID from your region.
- `<YOUR_INSTANCE_TYPE>` → for example `t3.micro`.

**Explain the fields:**
- `vpc_security_group_ids` → attaches our security group (a list with `[ ]`).
- `associate_public_ip_address = true` → gives the instance a public IP so a browser can reach it.
- `user_data` → a script AWS runs **once when the instance first boots**. Terraform uses it to install and start NGINX. `-y` avoids the "Are you sure?" prompt, which would stop the script. `systemctl enable` starts NGINX after reboot too. `user_data` runs as root, so `sudo` is not required.
- `<<-EOF ... EOF` is a multi-line string. Keep `#!/bin/bash` as the first line.
- `depends_on` makes sure the internet route exists before the instance tries to download NGINX.

## Step 7 — `outputs.tf`
```hcl
output "instance_public_ip" {
  description = "Public IP of the NGINX server"
  value       = aws_instance.nginx_server.public_ip
}

output "instance_url" {
  description = "URL to open in the browser"
  value       = "http://${aws_instance.nginx_server.public_ip}"
}
```

## Step 8 — Run
```bash
terraform init
terraform validate
terraform apply
```
Type `yes`.

**Expected:**
```text
Apply complete! Resources: 9 added, 0 changed, 0 destroyed.

Outputs:
instance_public_ip = "x.x.x.x"
instance_url = "http://x.x.x.x"
```

## Step 9 — Verify in the console
- EC2 → Instances: `nginx-server` is **Running** and shows a **Public IPv4 address**.
- VPC = `my-vpc`, Subnet = `public-subnet`.
- Instance → **Security** tab: security group `nginx-sg`; **Inbound rules**: HTTP, port 80, `0.0.0.0/0`; **Outbound rules**: all traffic.

## Step 10 — Test in the browser
Wait **1–2 minutes** after apply (the script takes time). Open the `instance_url` value in a new browser tab:

```text
http://<PUBLIC_IP>
```
**Expected:** **Welcome to nginx!** (or the NGINX test page).

Use **http**, not **https**. If the browser tries https, edit the address manually.

## Troubleshooting

### Error 1 — Browser shows "This site can't be reached" / keeps loading
**Reasons and checks:**
1. Script is still running → wait 2 minutes and refresh.
2. Security group has no inbound port 80 → check the **Security** tab.
3. Route table is not associated with the public subnet → check the VPC Resource map.
4. Browser is using https → use `http://`.

### Error 2 — NGINX not installed
Check the boot log: EC2 → select instance → **Actions → Monitor and troubleshoot → Get system log** and look for errors from `yum`.
**Reason:** Wrong AMI (for example old Amazon Linux 2) or no internet route.
**Fix:** Use an Amazon Linux 2023 AMI. To re-run the script: change `user_data`, then
```bash
terraform apply
```
(Changing `user_data` may need `user_data_replace_on_change = true` inside the `aws_instance` block to recreate the instance. ⚠️ Verification Required.)

### Error 3
```text
Error: Unsupported argument: "ingress"
```
**Reason:** `ingress = { ... }` was written with an `=` sign.
**Fix:** Use `ingress { ... }` (block).

## Cleanup
```bash
terraform destroy
```
Type `yes`.

## Final Result
Opening the instance URL in the browser shows the NGINX welcome page, and everything (VPC, subnets, IGW, routes, SG, EC2, NGINX) was created by Terraform.
