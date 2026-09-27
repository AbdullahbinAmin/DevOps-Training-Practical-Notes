# Practical 8 — AWS EC2 Setup for Kubernetes Hands-on

## Objective
Create an AWS EC2 instance that will be used to install and run a Kubernetes cluster (using KIND — Kubernetes IN Docker) for hands-on practice.

## Prerequisites
- An AWS account (new account gets 1 year of free-tier usage)
- A valid email ID and card for account signup (not covered in this practical — this is not an account-creation tutorial)
- Basic understanding of what a Node/Cluster is (from Chapter 6)

## Pre-Check
Before starting, make sure you can log in to the AWS Console:
```text
https://console.aws.amazon.com/
```
**Expected result:** You should be able to see the AWS Management Console dashboard after login.

## Step 1 — Select Your AWS Region

**Step 1:** Open the AWS Console.

**Step 2:** In the top-right corner, click the region dropdown.

**Step 3:** Select your nearest region. Example used in this practical: **Asia Pacific (Mumbai) — ap-south-1**.

⚠️ **Verification Required:** Exact region names may vary slightly by AWS console version. Choose the region closest to your physical location for lower latency.

## Step 2 — Launch an EC2 Instance

**Step 1:** Go to:
`Services → EC2`

**Step 2:** Click **Launch Instance**.

**Step 3:** Enter:
- Name: `k8s-in-one-shot` (or any name you like)

**Step 4:** Under **Application and OS Images (AMI)**, select:
- OS: **Ubuntu**
- Version: **Ubuntu 24.04 LTS** (released April 2024 — a stable version)
- Architecture: **x86 (64-bit)** — do NOT choose ARM, because ARM processors can cause issues while installing Docker later.

**Step 5:** Under **Key pair (login)**:
- Click **Create new key pair**
- Name: `k8s-in-one-shot`
- Key pair type: **RSA**
- Private key file format: **.pem**
- Click **Create key pair** — this will download a `.pem` file to your computer (usually in the Downloads folder).

⚠️ **Important:** Save this `.pem` file safely. You need it to SSH into your server. If you lose it, you cannot connect to this instance again.

**Step 6:** Under **Instance type**:
- Do NOT use `t2.micro` (free tier) — it is too small to run a full Kubernetes cluster with multiple nodes.
- Select at least **t2.medium** (or `t2.large` for smoother performance).

⚠️ Note: `t2.medium` and above are **not** covered by the AWS free tier, so this will incur a small cost. You can delete the instance after practice to avoid charges.

**Step 7:** Under **Network settings**:
- Click **Edit**
- Enable (check the box for):
  - **Allow SSH traffic** (port 22)
  - **Allow HTTPS traffic from the internet** (port 443)
  - **Allow HTTP traffic from the internet** (port 80)

**Step 8:** Under **Configure storage**:
- Increase the default storage (8 GB is not enough for a Kubernetes cluster with multiple applications).
- Set storage to **20 GB or 30 GB** (30 GB is within AWS free-tier storage limits, so it is recommended).

**Step 9:** Click **Launch Instance**.

**Expected result:** A message should appear saying "Successfully initiated launch of instance," and your instance should show status **Running** shortly after.

## Step 3 — Connect to the EC2 Instance via SSH

**Step 1:** In the EC2 console, select your running instance.

**Step 2:** Click **Connect** → go to the **SSH client** tab.

**Step 3:** Copy the SSH command shown there. It will look like:
```bash
ssh -i "<KEY_NAME>.pem" ubuntu@<PUBLIC_IP_OR_DNS>
```
- `<KEY_NAME>` → the name of your downloaded `.pem` key file
- `<PUBLIC_IP_OR_DNS>` → the public address of your EC2 instance (AWS provides this automatically)

**Step 4:** Open your terminal and go to the folder where the `.pem` file was downloaded:
```bash
cd Downloads
```
**What this does:** Moves you into the folder containing your private key file.

**Step 5:** Paste and run the SSH command copied in Step 3:
```bash
ssh -i "k8s-in-one-shot.pem" ubuntu@<PUBLIC_IP_OR_DNS>
```

### If Something Goes Wrong
If you see an error like:
```text
Permissions 0644 for 'k8s-in-one-shot.pem' are too open.
```

**Reason:**
Your private key file has open/unsafe permissions, and SSH refuses to use it until it is locked down.

**Fix:**
```bash
chmod 400 k8s-in-one-shot.pem
```
**What this does:** Restricts the key file so only you (the owner) can read it, which is required for SSH to accept it.

Then run the SSH command again:
```bash
ssh -i "k8s-in-one-shot.pem" ubuntu@<PUBLIC_IP_OR_DNS>
```

**Expected result:** You should now be logged into your EC2 Ubuntu server's terminal (prompt changes to something like `ubuntu@ip-xxx-xxx-xxx-xxx:~$`).

## Extra: Ways to Create a Kubernetes Cluster (Overview)

Before continuing, it helps to know there are multiple ways to create a Kubernetes cluster. This course will cover most of them:

1. **kubeadm** — You take 2 or more EC2 instances and manually install `kubeadm` on each, then join them into one cluster. This is more realistic to production but costs more (needs multiple servers).
2. **Minikube** — Creates a single-node Kubernetes cluster on your local machine or a single EC2 instance. Good for local learning.
3. **KIND (Kubernetes IN Docker)** — Runs Kubernetes clusters **inside Docker containers** on a single machine. This is what we will use first in this course (covered in Chapter 9).
4. **Managed Kubernetes Services (Cloud)**:
   - **EKS** (Elastic Kubernetes Service) — AWS
   - **AKS** (Azure Kubernetes Service) — Microsoft Azure
   - **GKE** (Google Kubernetes Engine) — Google Cloud
   - With these, the cloud provider manages the cluster's control plane for you.
5. **Other enterprise tools**: Rancher (RKE - Rancher Kubernetes Engine) for production/enterprise-level clusters, and various cloud providers offering similar services.

This course starts with **KIND** and **Minikube** for local practice (Chapters 9–12), then covers **kubeadm** (Chapter 13) for a more manual/production-style setup, and later (in a future mega project) will cover **EKS**.

## Final Result
You now have a running AWS EC2 Ubuntu server (t2.medium or larger, 30 GB storage), connected via SSH, ready to install Docker, KIND, and kubectl in the next chapter.

---
**Prerequisite for Chapter 9:** This EC2 instance must be running and connected via SSH before continuing.
