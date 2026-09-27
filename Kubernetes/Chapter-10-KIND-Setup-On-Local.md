# Practical 10 — Installing KIND on Your Local Machine

## Objective
Install KIND, kubectl, and Docker on your **local laptop** (not AWS), so you can practice Kubernetes without needing a cloud server.

## Prerequisites
- A local machine (Mac, Windows, or Linux)
- Docker Desktop installed (required — KIND runs Kubernetes inside Docker containers)
- Internet connection
- Terminal access

## Pre-Check
```bash
docker --version
```
**Expected result:** Docker version should print. If Docker is not installed, install Docker Desktop first from the official site: https://www.docker.com/products/docker-desktop/

## Step 1 — Download KIND for Your OS

Go to the official KIND installation page to get the exact command for your operating system:
**Official Installation Guide:** https://kind.sigs.k8s.io/docs/user/quick-start/#installation

### For Mac (Apple Silicon / M1 and above)
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.25.0/kind-darwin-arm64
```

### For Mac (Intel processor)
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.25.0/kind-darwin-amd64
```

### For Linux
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.25.0/kind-linux-amd64
```

### For Windows
Use the `.exe` download link shown on the official KIND installation page above, or use `choco install kind` if you have Chocolatey installed.

⚠️ **Verification Required:** Always confirm the latest version number and correct file name (arm64 vs amd64) from the official page before downloading, since it may change.

**What this does:** Downloads the KIND binary file matching your computer's operating system and processor type.

## Step 2 — Make KIND Executable and Move It to PATH

```bash
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```
**What this does:**
- `chmod +x ./kind` makes the downloaded file runnable as a program.
- `sudo mv ./kind /usr/local/bin/kind` moves it into a folder that is already in your system's PATH, so you can run `kind` as a command from anywhere in the terminal.

## Step 3 — Verify KIND Installation

```bash
kind --version
```
**Expected output:** `kind version 0.25.0` (or whichever version you downloaded)

## Step 4 — Verify Docker and kubectl

```bash
docker --version
```
**Expected output:** Docker version details should print.

```bash
kubectl version --client
```
**Expected output:** kubectl client version should print. If kubectl is not installed, follow the official guide: https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/ (or the Linux/Windows equivalent).

## Final Result
Your local machine now has **Docker**, **KIND**, and **kubectl** installed. You now have KIND working both on your AWS EC2 instance (Chapter 9) and on your local machine (this chapter), so you can practice on either one.

---
**Prerequisite from Chapter 9:** Understanding of what KIND is and why it needs Docker.

**Prerequisite for Chapter 11:** KIND, Docker, and kubectl must be installed (either on local machine or AWS EC2, this course continues using AWS EC2 going forward for consistency).
