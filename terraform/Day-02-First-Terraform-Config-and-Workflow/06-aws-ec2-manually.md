# Day 02 — Launching an AWS EC2 Instance Manually (Console)

> Create one EC2 instance by hand in the AWS Console so you can feel exactly how much clicking Terraform will replace.

## Learning Objectives
- Select the correct AWS Region before creating any resource.
- Walk the EC2 Launch Instance wizard end to end (name, AMI, instance type, key pair, network, storage).
- Create a key pair and understand why the `.pem` file must be kept safe.
- Open only the inbound ports you need with a security group.
- Compare the manual click-path with the Terraform code path you will write next.

## Prerequisites
- An AWS account with console access (free tier is enough).
- Permission to create EC2 instances, key pairs, and security groups.
- An SSH client (`ssh` on Linux/macOS/Git Bash, or PowerShell on Windows 10+).
- A browser. No Terraform needed for this lesson.

## Concept
EC2 (Elastic Compute Cloud) gives you a virtual server in AWS. To launch one you must answer five questions: which Region, which operating system image (AMI), how big the machine is (instance type), how you will log in (key pair), and who may reach it over the network (security group). The console asks these questions with a wizard.

Region matters because a Region is a physical location. An instance created in `eu-north-1` (Stockholm) does not appear in `us-east-1` (N. Virginia), and neither do its key pairs or security groups. Pick a Region close to you or your users and stay in it for the whole lab.

Doing this by hand once is worth it. Every field you fill in the wizard maps to one line of Terraform code later, and the manual run is what makes that mapping obvious.

## Step-by-Step Practical

1. Sign in to the AWS Console and select your Region from the top-right Region dropdown. The slide example uses Stockholm because the trainer is in Romania; pick what fits you.

```text
Top-right of console  ->  Region dropdown  ->  Europe (Stockholm) eu-north-1
```

2. Open the EC2 service. Search `EC2` in the top search bar, or find it under All services -> Compute -> EC2.

```text
Search box  ->  type: EC2  ->  click "EC2  |  Virtual Servers in the Cloud"
```

3. On the EC2 Dashboard click the orange **Launch instance** button. This starts the launch wizard.

4. Configure instance details.

```text
Name                 : My-Web-Server
Application and OS   : Amazon Linux 2 AMI (HVM), SSD Volume Type, 64-bit (x86)
Instance type        : t2.micro     (Free tier eligible)
```

5. Create a key pair for SSH login. Choose **Create new key pair**, type `RSA`, format `.pem`.

```text
Key pair name : my-web-server-key-01
Purpose       : SSH access to my web server
Downloads as  : my-web-server-key-01.pem   (save it carefully, AWS never shows it again)
```

On Linux/macOS/Git Bash, tighten the file permissions immediately, otherwise SSH refuses the key.

```bash
mv ~/Downloads/my-web-server-key-01.pem ~/.ssh/
chmod 400 ~/.ssh/my-web-server-key-01.pem
```

6. Network settings. Choose **Create security group** and add two inbound rules.

```text
Inbound rules
Type   Protocol   Port Range   Source
SSH    TCP        22           0.0.0.0/0
HTTP   TCP        80           0.0.0.0/0
```

Security warning: `0.0.0.0/0` on port 22 means the whole internet can attempt SSH. The slide uses it for simplicity in a training lab. For anything real, restrict SSH to your own IP (`My IP` in the dropdown, which writes `YOUR.PUBLIC.IP/32`).

7. Storage. Keep the default root volume.

```text
1 x 8 GiB gp3 root volume
```

8. Review the Summary panel on the right, then click **Launch instance**.

```text
Summary
Instance type       t2.micro
AMI                 Amazon Linux 2
Key pair            my-web-server-key-01
Security group      launch-wizard-1
Storage (volumes)   8 GiB
```

9. Click **View all instances**. Wait until Instance state shows `Running` and a Public IPv4 address appears. Click the Instance ID to see details.

10. Connect over SSH using the `.pem` file and the `ec2-user` account (Amazon Linux default user).

```bash
ssh -i ~/.ssh/my-web-server-key-01.pem ec2-user@<YOUR-PUBLIC-IP>
```

## Expected Output

```text
Instances (1/1)
Name            Instance ID            Instance state   Public IPv4
My-Web-Server   i-0a1b2c3d4e5f67890    Running          18.207.XX.XX

$ ssh -i ~/.ssh/my-web-server-key-01.pem ec2-user@18.207.XX.XX
The authenticity of host '18.207.XX.XX' can't be established.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

[ec2-user@ip-172-31-XX-XX ~]$
```

## Verification
- EC2 -> Instances shows Instance state `Running` and Status checks `2/2 checks passed`.
- The Details tab shows the AMI ID, instance type `t2.micro`, and your key pair name.
- SSH connects and you land on a shell prompt like `[ec2-user@ip-172-31-XX-XX ~]$`.
- Run `uname -a` and `curl -s http://169.254.169.254/latest/meta-data/instance-id` on the box to confirm you are on the instance you think you are.

## Cleanup
Manual resources cost money until you remove them. There is no `terraform destroy` here because Terraform did not create this instance.

1. EC2 -> Instances -> select `My-Web-Server` -> Instance state -> **Terminate instance** -> confirm.
2. EC2 -> Key Pairs -> delete `my-web-server-key-01` if you no longer need it (also delete the local `.pem`).
3. EC2 -> Security Groups -> delete `launch-wizard-1` after the instance is fully terminated.

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| Instance is missing from the list | You switched Region | Select the Region you launched in from the top-right dropdown |
| `Permissions 0644 for 'key.pem' are too open` | `.pem` file is world-readable | `chmod 400 my-web-server-key-01.pem` |
| `Permission denied (publickey)` | Wrong username or wrong key | Use `ec2-user` for Amazon Linux, `ubuntu` for Ubuntu; confirm the key pair name on the instance Details tab |
| SSH connection times out | Port 22 not open, or no public IP | Add an inbound SSH rule to the security group; verify the instance is in a public subnet with auto-assign public IP |
| `InstanceLimitExceeded` | Account vCPU limit reached | Terminate unused instances or request a limit increase in Service Quotas |
| Unexpected charges | Instance type not free tier, or instance left running | Use `t2.micro`/`t3.micro` where free-tier eligible and terminate when done |

## Key Takeaways
- Region first: everything you create lives inside one Region.
- The wizard's five real decisions are AMI, instance type, key pair, security group, and storage.
- The `.pem` private key is downloadable exactly once. Lose it and you lose SSH access.
- Manual launches are many clicks, are not repeatable, and are not reviewable. That is the problem Terraform solves.

## Next: [First Terraform Config to Create an EC2 Instance](07-first-terraform-config.md)
