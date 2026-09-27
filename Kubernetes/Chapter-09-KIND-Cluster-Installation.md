# Practical 9 — What is KIND Cluster & Installing It (on AWS EC2)

## Objective
Explain what KIND is, and install KIND, kubectl, and Docker on the AWS EC2 server created in Chapter 8.

## What is KIND?
**KIND = Kubernetes IN Docker**

It is a tool that lets you run a full Kubernetes cluster **inside Docker containers**, all on a single machine/instance. This is great for learning and local testing without needing multiple servers.

## Prerequisites
- AWS EC2 instance from Chapter 8, running and connected via SSH
- OS: Ubuntu 24.04
- Internet access on the instance (to download KIND and kubectl)

## Pre-Check
```bash
whoami
```
**What this does:** Confirms which user you are logged in as (should be `ubuntu`).

## Step 1 — Create the Installation Script

**Step 1:** Create a new file called `install.sh`:
```bash
vim install.sh
```

**Step 2:** Paste the following content into the file:
```bash
#!/bin/bash

# Install KIND
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.25.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

**What this does:**
- The first block downloads a specific version of KIND (`v0.25.0`, the latest stable version at the time of this recording) from its official release URL, makes it executable, and moves it into `/usr/local/bin` so it becomes a usable command (`kind`).
- The second block downloads the latest stable `kubectl` binary, makes it executable, and moves it into `/usr/local/bin` so it becomes a usable command (`kubectl`).

**Official Installation Guide (for latest versions/verification):**
- KIND: https://kind.sigs.k8s.io/docs/user/quick-start/
- kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

⚠️ **Verification Required:** Always check the official KIND releases page for the latest stable version number before installing, in case a newer version is available.

**Step 3:** Save and exit the file (in `vim`, press `Esc`, then type `:wq` and press Enter).

**Step 4:** Verify the file was saved:
```bash
cat install.sh
```
**Expected output:** The script content should be printed exactly as you wrote it.

## Step 2 — Give the Script Execute Permission and Run It

```bash
chmod +777 install.sh
```
**What this does:** Gives the script full read/write/execute permission so it can be run.

```bash
./install.sh
```
**What this does:** Runs the script, which downloads and installs both KIND and kubectl.

**Expected result:** Message should show something like "KIND and kubectl installation complete" with no errors.

## Step 3 — Install Docker

KIND needs Docker to run (since Kubernetes will run "inside Docker" containers).

**Step 1:** Update the package list:
```bash
sudo apt update
```
**What this does:** Refreshes the list of available packages from Ubuntu's repositories.

**Step 2:** Install Docker:
```bash
sudo apt install docker.io -y
```
**What this does:** Installs Docker Engine from Ubuntu's default repository.

**Step 3:** Verify Docker is running:
```bash
docker ps
```
**Expected output:** A table with column headers like `CONTAINER ID, IMAGE, COMMAND...` and no error. If you see nothing else, that is fine (it just means no containers are running yet).

### If Something Goes Wrong
If you see:
```text
permission denied while trying to connect to the Docker daemon socket
```

**Reason:**
Your current user (`ubuntu`) does not have permission to run Docker commands directly (only `root`/`sudo` can by default).

**Fix:**
```bash
sudo usermod -aG docker $USER
newgrp docker
```
**What this does:**
- `sudo usermod -aG docker $USER` adds your current user to the `docker` group, giving it permission to run Docker without `sudo`.
- `newgrp docker` refreshes your current session so the new group membership takes effect immediately (without needing to log out and log back in).

**Step 4:** Verify again:
```bash
docker ps
```
**Expected result:** Should now run without any permission error.

## Step 4 — Final Verification of All Tools

```bash
docker --version
```
**Expected output:** Something like `Docker version 24.x.x`

```bash
kind --version
```
**Expected output:** Something like `kind version 0.25.0`

```bash
kubectl version --client
```
**Expected output:** Should show the client version of kubectl (server version will show "connection refused" until a cluster is created — this is expected at this stage).

## Final Result
Your AWS EC2 instance now has **Docker**, **KIND**, and **kubectl** installed and working. You are ready to create your first KIND-based Kubernetes cluster (covered in Chapter 11).

---
**Prerequisite from Chapter 8:** Make sure your EC2 instance is running and you are connected via SSH before starting this chapter.

**Prerequisite for Chapter 10:** This chapter (9) was done on the AWS EC2 server. Chapter 10 repeats similar installation steps, but on your **local machine** (e.g., Mac/Windows/Linux laptop) instead of AWS.
