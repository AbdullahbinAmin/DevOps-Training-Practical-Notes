# Chapter 10 — AWS EC2 Manually (Console)

## Objective
Create an EC2 instance by hand in the AWS Console. Later chapters create the same thing with Terraform, so you can compare.

## Prerequisites
- IAM user login (Chapter 6)
- Free tier eligible instance type available in your region

## Cost warning
Use only **Free tier eligible** options. **Terminate the instance** at the end of this practical.

## Step 1 — Check the region
Look at the top-right corner of the console. The selected region (example: **Stockholm / eu-north-1**) is where your server will be created. Remember it. Terraform will use the same region later.

## Step 2 — Open EC2
Search `EC2` in the top bar (or **Services → Compute → EC2**). On the EC2 dashboard click **Launch instance**.

## Step 3 — Configure the launch page (GUI)

**Name and tags:**
- Name: `my-web-server`

**Application and OS Images (AMI):**
- Choose **Amazon Linux** and select an AMI marked **Free tier eligible** (the video used an Amazon Linux AMI). ⚠️ Verification Required: options shown depend on the region and date. Amazon Linux 2023 is recommended for later chapters (NGINX install uses `yum`/`dnf`).
- Note the **AMI ID** shown under the AMI name (looks like `ami-0abc...`). Save it — you need it in Chapter 11.

**Instance type:**
- Choose the one marked **Free tier eligible** (video: `t3.micro`; some accounts show `t2.micro`).

**Key pair (login):**
1. Click **Create new key pair**.
2. Key pair name: `my-web-server-key`
3. Type: **RSA**, format: **.pem**
4. Click **Create key pair**. A file downloads to your computer. Keep it safe because you need it to log in remotely.

**Network settings:**
1. Click **Edit** (if needed) and choose **Create security group**.
2. Tick **Allow SSH traffic from** → **Anywhere** (used only for learning).
3. Tick **Allow HTTP traffic from the internet** (needed if you run a web server).

**Configure storage:** keep default.

## Step 4 — Launch
Check the **Summary** panel on the right, then click **Launch instance**.

**Expected:** "Successfully initiated launch of instance" with an instance ID.

## Step 5 — Verify
1. Click **Instances** in the left menu. Click refresh.
2. You should see `my-web-server` with **Instance state: Running**.
3. Click the **instance ID** to see details (public IP, instance type, AMI).

## Step 6 — Cleanup
Select the instance → **Instance state → Terminate (delete) instance** → confirm. State becomes **Terminated**.

## Troubleshooting

### Error 1
Cannot find the instance after launch.
**Reason:** You are looking at a different region.
**Fix:** Switch the region in the top-right to the one you launched in.

### Error 2
`Launch failed: You are not authorized...`
**Reason:** The IAM user has no permission.
**Fix:** Check that the user is in the `admins` group (Chapter 6).

## Final Result
You created and terminated an EC2 instance manually and you know where the AMI ID, instance type and region are found.
