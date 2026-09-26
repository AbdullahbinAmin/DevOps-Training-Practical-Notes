# Practical 3 — Setting Up Two Linux Machines on AWS and Installing Git

## Objective
Launch two Ubuntu EC2 instances on AWS (used to represent two developers working from different locations — e.g. "Mumbai" and "Singapore"), connect to them via SSH, and install Git on both.

## Prerequisites
* OS: Ubuntu (Linux)
* Required account: AWS account (free tier eligible)
* Required software: None yet — installed in this chapter
* Required ports: SSH (22)
* Required permissions: Access to AWS Console, ability to launch EC2 instances and create key pairs

## Pre-Check
Log in to the AWS Console and go to `EC2 → Instances`.
**Expected result:** No unwanted instances currently running (to avoid confusion and unnecessary billing).

---

## Architecture / Flow

```text
Developer's Laptop (PuTTY / SSH client)
        │
        ├── SSH ──> EC2 Instance "Mumbai" (Ubuntu) — Git installed here
        │
        └── SSH ──> EC2 Instance "Singapore" (Ubuntu) — Git installed here
```

Two machines are used here to simulate two different developers working on the same project from two different locations — a common real-world scenario that helps demonstrate how Git's distributed model works.

---

## Step 1 — Launch the First EC2 Instance ("Mumbai")

**Step 1:** Go to `EC2 → Instances → Launch Instance`.

**Step 2:** Enter:
* Name: `Mumbai Git`
* AMI: **Ubuntu** (latest LTS)
* Instance type: **t3.micro** (free tier eligible)

**Step 3:** Create a new key pair:
* Key pair name: `Mumbai Key`
* Key pair type: RSA
* Private key format: **.pem**

Click **Create key pair** — this downloads the `.pem` file automatically.

**Step 4:** In Network Settings, allow:
* **SSH** (port 22) — Anywhere
* **HTTP** (port 80) — Anywhere

**Step 5:** Storage — leave at default (8–20 GB is enough).

**Step 6:** Click **Launch Instance**.

**Why:** Ubuntu + t3.micro are free-tier eligible. Allowing SSH lets you log in; allowing HTTP is useful for any later web-based practicals.

---

## Step 2 — Launch the Second EC2 Instance ("Singapore")

Repeat the same steps as above, but:
* Switch the AWS region (top-right region selector) to a different region — e.g. **Singapore** — before launching (this is optional but mirrors the "two different locations" idea from the lecture; you can also launch both in the same region if preferred).
* Name: `Singapore Git`
* Key pair name: `Singapore Key` (a **separate** key pair — do not reuse the Mumbai key)
* Same AMI (Ubuntu), same instance type (t3.micro), same security group rules (SSH + HTTP)

**Expected result:** You now have 2 running EC2 instances — `Mumbai Git` and `Singapore Git` — each with its own downloaded `.pem` key file.

---

## Step 3 — Secure the Downloaded Key Files

On your local machine, wherever the `.pem` files were downloaded (e.g. Downloads folder):

```bash
chmod 400 "Mumbai Key.pem"
chmod 400 "Singapore Key.pem"
```

**What this does:** Restricts the private key files to read-only for your user, which SSH requires.

> If you are on Windows using PuTTY, convert the `.pem` file to a `.ppk` file using PuTTYgen before use, as shown in the video (Conversions → Load the `.pem` → Save private key as `.ppk`).

---

## Step 4 — Connect to the Mumbai Instance via SSH/PuTTY

**Step 1:** Go to `EC2 → Instances → Mumbai Git → Connect` tab, note the Public IP.

**Step 2 (Linux/Mac):**
```bash
ssh -i "Mumbai Key.pem" ubuntu@<MUMBAI_PUBLIC_IP>
```

**Step 2 (Windows/PuTTY):** Open PuTTY, paste the Public IP into "Host Name", go to `Connection → SSH → Auth → Credentials`, browse to the `.ppk` file, then click **Open**. Log in as `ubuntu`.

**Expected result:** You are logged into the Mumbai machine, shown by a prompt like `ubuntu@ip-xxx-xxx-xxx-xxx:~$`.

---

## Step 5 — Become Root User (Optional but Used Throughout)

```bash
sudo su
```

**What this does:** Switches to the root user, so you don't need to prefix every command with `sudo`. This is used throughout the rest of these practicals for convenience.

---

## Step 6 — Update the System and Install Git

```bash
apt update -y
```

**What this does:** Refreshes the list of available packages. `-y` auto-confirms any prompts.

```bash
apt install git -y
```

**What this does:** Installs Git from Ubuntu's default package repository — no manual download needed, since Git is a very common Linux package.

> ⚠️ If you are using a non-Ubuntu OS, refer to Git's **Official Installation Guide**: `https://git-scm.com/downloads` and follow the instructions for your OS rather than guessing commands.

**Verify:**

```bash
git --version
```

**Expected result:** Shows the installed Git version, e.g. `git version 2.34.1`.

> **Interview tip:** If asked which Git version you use, don't guess randomly — check with `git --version` and state the actual version you're running. This is a common real-world interview follow-up.

**Also verify install location (optional):**

```bash
which git
```

**Expected result:** Shows the path where Git binaries were installed (e.g. `/usr/bin/git`).

---

## Step 7 — Repeat the Exact Same Steps on the Singapore Instance

Connect to `Singapore Git` via SSH/PuTTY using `Singapore Key.pem`/`.ppk`, then run the same commands:

```bash
sudo su
apt update -y
apt install git -y
git --version
```

**Expected result:** Git is installed and verified on both machines identically.

---

## Troubleshooting

### Error 1 — `Permission denied (publickey)` when connecting via SSH
**Reason:** Incorrect key file, or key permissions too open.
**Fix:** Confirm you're using the correct `.pem`/`.ppk` file for that specific instance, and that `chmod 400` was applied (Linux/Mac).

### Error 2 — `git: command not found` after installation
**Reason:** The `apt install git` step failed silently or wasn't run as root/sudo.
**Fix:** Re-run `sudo apt update -y && sudo apt install git -y`, and check for error messages during installation.

### Error 3 — Cannot connect, connection times out
**Reason:** Security group does not allow inbound SSH (port 22).
**Fix:** Go to the instance's Security Group in AWS Console and confirm an SSH (port 22) inbound rule exists.

---

## Final Result
By the end of this chapter:
* Two independent Ubuntu EC2 instances are running (representing two developers — "Mumbai" and "Singapore")
* Git is installed and verified (`git --version`) on both machines
* You are comfortable connecting to both via SSH/PuTTY

## What's Next
**Chapter 4** covers initializing a Git repository, making your first commit, creating a GitHub account, and pushing your local repository to GitHub — plus your first ad-hoc `git status`/`git log` commands.
