# Chapter 3 — Installing Docker (Local + AWS EC2)

## Objective
Install Docker on two environments: (1) your local machine using Docker Desktop, and (2) a cloud Ubuntu server on AWS EC2 (used for all further practicals in this course, since it is free/cheap and avoids "it doesn't work on my Windows" issues).

## Prerequisites
* OS: Windows/macOS for local install; Ubuntu (via AWS EC2) for the server install
* RAM: At least 1 GB (t2.micro is Free Tier eligible but has only 1 GB RAM; t2.medium with 4 GB RAM is recommended for running multiple projects)
* CPU: 1 vCPU minimum
* Required software: A web browser
* Required account: Docker Hub account (free), AWS account (free tier or paid)
* Required repository: None yet
* Required ports: SSH port 22 (to connect to EC2); more ports opened in later chapters as needed
* Required permissions: Sudo/root access on the EC2 instance
* Required files: A `.pem` key file (downloaded during EC2 instance creation) — needed to SSH into the instance

## Pre-Check
Before starting, make sure you have:
```bash
ssh -V
```
**What this does:** Confirms an SSH client is available on your local computer (needed to connect to the EC2 instance). This is usually pre-installed on macOS/Linux and available via PowerShell on modern Windows.

---

## Part A — Installing Docker Desktop Locally

### Step 1 — Download Docker Desktop
Go to your browser and search "Download Docker Desktop", or go directly to the official page.

**Official Installation Guide:**
https://www.docker.com/products/docker-desktop/

> ⚠️ Always use this official page to download Docker Desktop for your OS (Windows/macOS/Linux), since installer steps and system requirements change over time.

### Step 2 — Create a Docker Account
Docker Desktop requires you to sign in.
1. On the Docker website, click **Create an account**.
2. Fill in your details and complete sign-up.
3. Sign in with your new credentials.

### Step 3 — Download and Install Docker Desktop
1. Go to **Get Docker** section on the Docker website.
2. Download the Docker Desktop installer for your operating system.
3. Run the installer and follow the on-screen instructions.
4. Once installed, open Docker Desktop from your applications.

**Why this matters:**
Docker Desktop gives you a GUI where you can see running containers, images, volumes, builds, and use extensions — all without typing CLI commands. It also includes an embedded terminal.

### Verify
Open the terminal built into Docker Desktop and run:
```bash
docker ps
```
**Expected result:** An empty table with columns like `CONTAINER ID`, `IMAGE`, `COMMAND`, etc. (no error means Docker is working).

---

## Part B — Installing Docker on AWS EC2 (Ubuntu)

This is the environment used for the rest of this course's practicals.

### Step 1 — Log in to AWS Console
Go to https://aws.amazon.com and log in to your account.

### Step 2 — Launch an EC2 Instance
1. Navigate to **EC2 → Instances**.
2. Click **Launch Instance**.
3. **Name:** Give it a name, e.g. `docker-in-one-shot`.
4. **Image (AMI):** Choose **Ubuntu** (an LTS version, easy to use and well supported).
5. **Instance type:**
   - `t2.micro` → Free Tier eligible (750 hours/month free), but only 1 vCPU and 1 GB RAM. Good enough for practicing one project at a time.
   - `t2.medium` → Not free tier, billed per hour, but gives 4 GB RAM — recommended if you plan to run multiple containers/projects simultaneously (as done later in this course).
6. **Key pair:** Click **Create new key pair**.
   - Name it (e.g., `docker-in-one-shot-key`).
   - Choose **PEM** format if you will SSH from macOS/Linux/PowerShell, or **PPK** if you plan to use PuTTY on Windows.
   - Click **Create key pair** — this downloads a `.pem` (or `.ppk`) file. **Save this file safely; you cannot re-download it.**
7. **Network settings:** Make sure **Allow SSH traffic** is enabled (port 22).
8. **Storage:** Default is around 8 GB; increase it if you plan to build many Docker images (e.g., set it to 15 GB or more, since Docker images and layers consume disk space).
9. Click **Launch Instance**.

**Why this matters:**
An EC2 instance is a virtual server in the cloud. Since it runs Ubuntu (Linux), Docker installs cleanly and works identically for every student, regardless of whether their own laptop is Windows or macOS.

### Step 3 — Verify Instance is Running
Go to **Instances** in the AWS Console.

**Expected result:** Instance state should show **Running**.

> If you need to pause billing, you can select the instance → **Instance State → Stop**. To resume, select **Instance State → Start**.

### Step 4 — Connect to the Instance via SSH

1. Select your instance → click **Connect**.
2. Go to the **SSH client** tab and copy the example SSH command shown (it will look like `ssh -i "your-key.pem" ubuntu@<public-ip>`).
3. On your local terminal, navigate to the folder where your `.pem` file was downloaded:
```bash
cd Downloads
```
4. List the file to confirm it exists:
```bash
ls
```
**Expected output:** You should see your `.pem` file listed (e.g., `docker-in-one-shot-key.pem`).

5. Restrict permissions on the private key (required by SSH; otherwise it will refuse to use the key):
```bash
chmod 400 docker-in-one-shot-key.pem
```
**What this does:** Sets the key file to read-only for the owner only. SSH requires private keys to not be publicly viewable/writable.

6. Connect to the instance using the exact command AWS gave you:
```bash
ssh -i "docker-in-one-shot-key.pem" ubuntu@<YOUR_EC2_PUBLIC_IP>
```
- `<YOUR_EC2_PUBLIC_IP>` → Replace with the actual Public IPv4 address of your instance shown in the AWS console.

7. When prompted with a fingerprint confirmation, type `yes` and press Enter.

**Expected result:** You are now logged into the Ubuntu server's terminal (prompt changes to something like `ubuntu@ip-xxx-xxx-xxx-xxx:~$`).

### Step 5 — Update the System
```bash
sudo apt-get update
```
**What this does:** Refreshes the list of available packages and their versions from Ubuntu's repositories, and pulls the latest security patches list. Always do this first on a fresh server.

### Step 6 — Install Docker
```bash
sudo apt-get install docker.io
```
**What this does:** Installs the `docker.io` package, which is Ubuntu's bundled Docker package. It installs everything needed: Docker Engine, Docker Daemon (dockerd), Docker CLI, and containerd — exactly the components you learned about in Chapter 2.

When prompted for confirmation (download size, e.g. ~289 MB), type `Y` and press Enter to proceed.

> ⚠️ Note: `apt-get install docker.io` is a quick way to install Docker on Ubuntu for learning/practice purposes. For production environments, Docker's official documentation recommends installing from Docker's own official APT repository. If you want the very latest Docker version with all features, check the official install guide:
> **Official Installation Guide:** https://docs.docker.com/engine/install/ubuntu/

### Step 7 — Verify Docker Components Were Installed
```bash
docker --version
```
**Expected output:** Something like `Docker version 24.x.x, build xxxxxxx` (exact version may vary).

### Step 8 — Verify the Docker Service (dockerd) is Running
```bash
sudo systemctl status docker
```
**Expected result:** Output should show:
```text
Active: active (running)
```
This confirms the Docker Application Container Engine (dockerd) is up and running — proving the architecture from Chapter 2 in real life.

Press `q` to exit the status view if needed.

### Step 9 — First Run of a Docker Command (and Fixing the Permission Error)
Try:
```bash
docker ps
```
**Expected error (this is normal on a fresh Ubuntu install):**
```text
permission denied while trying to connect to the Docker daemon socket
```

**Reason:** By default, only the `root` user (or users in the `docker` group) can talk to the Docker daemon socket. Your current user (`ubuntu`) is not yet in the `docker` group.

**Fix — Step 9a: Add your user to the docker group**
```bash
sudo usermod -aG docker $USER
```
**What this does:** Adds the current logged-in user (`$USER`, which resolves to `ubuntu`) to the `docker` group, which has permission to talk to the Docker daemon.

**Fix — Step 9b: Refresh your group membership**
```bash
newgrp docker
```
**What this does:** Applies the new group membership to your current terminal session immediately (otherwise you would need to log out and log back in for the group change to take effect).

### Step 10 — Final Verification
```bash
docker ps
```
**Expected result:** A clean table with headers only (no error), for example:
```text
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
```
This confirms that Docker CLI → Docker Daemon → containerd communication is working correctly (exactly as explained in Chapter 2's architecture).

### Step 11 (Optional) — Verify via Docker Desktop's Embedded Terminal
If using Docker Desktop locally, open the embedded terminal (bottom corner icon) and run:
```bash
docker ps
```
**Expected result:** Same clean output, confirming Docker Desktop is also correctly connected to its local Docker Engine.

## Troubleshooting

### Error 1
```text
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```
**Reason:** Your Linux user is not part of the `docker` group.

**Fix:**
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Error 2
```text
ssh: connect to host <ip> port 22: Connection timed out
```
**Reason:** Either the instance is not running, the Security Group does not allow inbound SSH (port 22) from your IP, or you used the wrong IP address.

**Fix:** Confirm instance state is "Running", check the Security Group's inbound rules include SSH (port 22) from your IP or Anywhere (0.0.0.0/0 for practice only), and re-copy the current Public IPv4 address (it can change if the instance is stopped/started without an Elastic IP).

## Cleanup
Not required at this stage — this instance will be reused throughout the entire course for all practicals.

## Final Result
* Docker Desktop is installed and working on your local machine.
* An AWS EC2 Ubuntu instance is running with Docker fully installed.
* `docker --version` works.
* `sudo systemctl status docker` shows `active (running)`.
* `docker ps` runs without permission errors on both environments.

## What's Next
Chapter 4 will explain Docker Images and Docker Containers — what they are, how to pull images from Docker Hub, and how to run your first containers (`hello-world`, `mysql`).
