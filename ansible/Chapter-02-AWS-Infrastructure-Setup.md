# Practical 2 — Setting Up AWS Infrastructure (Control Node + Worker Nodes)

## Objective
Manually create 3 AWS EC2 instances (1 Control Node + 2 Worker Nodes) using the AWS Console (GUI), so we have real machines to practice Ansible on.

## Prerequisites
* OS: Ubuntu (for all 3 instances)
* RAM: Default (t3.micro — free tier)
* CPU: Default (t3.micro — free tier)
* Required software: None yet (installed in later chapters)
* Required account: AWS account (free tier eligible)
* Required repository: None yet
* Required ports: SSH (22), HTTP (80), HTTPS (443)
* Required permissions: Ability to create EC2 instances, key pairs, and security groups
* Required files: None yet

## Pre-Check
Before starting, log in to the AWS Console and confirm:

```text
AWS Console → EC2 → Instances
```

**Expected result:** No instances currently running (to avoid confusion and unnecessary billing).

> ⚠️ Verification Required: AWS Console layout can change slightly over time. If menu names differ from below, look for the closest equivalent (e.g. "Launch Instance" button).

---

## Architecture / Flow

```text
Student → AWS Console (GUI) → Launch 3 EC2 Instances → 1 Control Node + 2 Worker Nodes
```

---

## Step 1 — Open EC2 and Start Launch Wizard

**Step 1:** Open the AWS Console and go to:
`EC2 → Instances`

**Step 2:** Click:
`Launch Instance`

---

## Step 2 — Configure the Instance (Name, OS, Type)

**Step 3:** Enter:
* Name: `control-node` (we will create 3 instances total — see Step 5 for multi-instance count)
* Application and OS Images (AMI): **Ubuntu** (latest LTS shown by default)
* Instance type: **t3.micro** (free tier eligible)

**Why Ubuntu + t3.micro:**
Ubuntu is free-tier eligible, widely used, and well documented. t3.micro is free-tier eligible so you avoid unnecessary billing.

---

## Step 3 — Create a Key Pair

**Step 4:** Click:
`Create new key pair`

**Step 5:** Enter:
* Key pair name: `ansible-in-one-shot-key`
* Key pair type: RSA
* Private key file format: **.pem** (works well with OpenSSH on Linux/Mac; use .ppk only if you are using PuTTY on Windows)

**Step 6:** Click **Create key pair**.

**Why:** This key pair lets you log in (SSH) to your EC2 instances. AWS will download the `.pem` file to your computer automatically — save it somewhere safe (e.g. your Downloads folder).

---

## Step 4 — Configure Network Settings (Security Group)

**Step 7:** In **Network settings**, make sure the following inbound rules are enabled:
* **SSH** (port 22) — Allow (needed to connect to the instance)
* **HTTP** (port 80) — Allow (needed later if we run Nginx)
* **HTTPS** (port 443) — Allow (needed later for secure web traffic)

**Why:** Without these ports open, you will not be able to SSH into the machine or access any web service running on it later.

---

## Step 5 — Configure Storage

**Step 8:** Set storage size to **20 GB** (gp3/gp2 EBS volume).

**Why:** 20 GB is more than enough for this practical (even 10 GB would work). AWS free tier allows up to 30 GB of EBS storage total, and we are creating 3 instances, so keep storage modest.

---

## Step 6 — Set Number of Instances and Launch

**Step 9:** In the **Summary** panel, set:
* Number of instances: **3**

**Step 10:** Click **Launch Instance**.

**Why 3 instances:** Our architecture needs 1 Control Node + 2 Worker Nodes = 3 total instances.

---

## Step 7 — Rename the Instances

By default, all 3 instances will be launched with the same name (`control-node`). We need to rename them so each has a distinct role.

**Step 11:** Go to `EC2 → Instances`. Select one instance, click the pencil/edit icon next to the Name field, and rename:
* Instance 1 → `control-node`
* Instance 2 → `worker-node-1`
* Instance 3 → `worker-node-2`

---

## Step 8 — Verification

**Step 12:** Go to `EC2 → Instances` and confirm you see:

```text
control-node    → running
worker-node-1   → running
worker-node-2   → running
```

**Expected result:** All 3 instances show status **Running** and pass the status checks (2/2 checks passed, may take 1–2 minutes).

---

## Troubleshooting

### Error 1 — Instance stuck in "Pending" state
**Reason:** AWS is still provisioning the instance. This is normal and can take 1–2 minutes.
**Fix:** Wait and refresh the console.

### Error 2 — Cannot download the .pem key pair again
**Reason:** AWS only lets you download the private key once, at creation time.
**Fix:** If lost, you must create a new key pair and associate it with a new instance (existing running instances cannot have their key pair changed easily). Always save the `.pem` file immediately and keep a backup.

### Error 3 — SSH port not open
**Reason:** Security group inbound rule for port 22 (SSH) was missed during instance launch.
**Fix:** Go to `EC2 → Instances → select instance → Security tab → Security Groups → Edit inbound rules` and add an SSH (port 22) rule.

---

## Final Result
You should now have **3 running EC2 instances**:
* `control-node` (Ubuntu, t3.micro)
* `worker-node-1` (Ubuntu, t3.micro)
* `worker-node-2` (Ubuntu, t3.micro)

All using the same key pair (`ansible-in-one-shot-key.pem`), with SSH/HTTP/HTTPS ports open.

> **Prerequisite for Chapter 3:** Make sure all 3 instances are in "Running" state and you have the `.pem` key file saved locally before continuing.
