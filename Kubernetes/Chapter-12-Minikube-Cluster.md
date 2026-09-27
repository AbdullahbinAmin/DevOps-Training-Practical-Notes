# Practical 12 — What is Minikube, Installation, and Cluster Creation

## Objective
Understand what Minikube is, install it, create a Minikube cluster, and learn how to switch between multiple Kubernetes clusters (KIND vs Minikube) using `kubectl` contexts.

## Prerequisites
- Docker installed and running
- AWS EC2 instance (or local machine) from earlier chapters
- KIND cluster from Chapter 11 still running (used later for context-switching practice)
- Architecture: `amd64` (x86) — this practical uses the amd64 build

## Pre-Check
```bash
docker --version
```
**Expected result:** Docker version should print without error.

## Step 1 — Create a Folder for Minikube

```bash
mkdir minikube
cd minikube
```
**What this does:** Creates and enters a dedicated working folder (optional but keeps things organized).

## Step 2 — Install Prerequisite Packages

```bash
sudo apt update
```
**What this does:** Refreshes the package index so the system knows about the latest available package versions.

```bash
sudo apt install -y curl wget apt-transport-https
```
**What this does:** Installs basic tools (`curl`, `wget`) needed to download files, plus `apt-transport-https` which allows package downloads over HTTPS.

## Step 3 — Make Sure Docker is Enabled and Running

```bash
sudo systemctl enable docker
```
**What this does:** Ensures Docker automatically starts again if the machine restarts (Enable = "on startup, the service will restart").

```bash
sudo systemctl status docker
```
**Expected output:** `active (running)`

## Step 4 — Download and Install Minikube

**Official Installation Guide:** https://minikube.sigs.k8s.io/docs/start/

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```
**What this does:** Downloads the latest Minikube binary for Linux (amd64/64-bit systems).

```bash
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```
**What this does:** Makes the downloaded file executable and moves it into `/usr/local/bin`, so it becomes available as the `minikube` command from anywhere.

⚠️ **Verification Required:** For Mac or Windows, use the OS-specific commands from the official installation guide link above.

**Verify installation:**
```bash
minikube version
```
**Expected output:** Something like `minikube version: v1.34.0`

## Step 5 — Start the Minikube Cluster

```bash
minikube start --driver=docker --vm=true
```

**What this does:**
- `--driver=docker` → tells Minikube to run the cluster using Docker as the underlying engine (same idea as KIND).
- `--vm=true` → needed specifically when running on a cloud VM like an EC2 instance, since Minikube needs to know it's running inside a virtual machine environment.

**Expected output (will take a few minutes):**
```text
minikube v1.34.0
Pulling base image ...
Creating docker container (CPUs=2, Memory=...) ...
Preparing Kubernetes v1.31.0 on Docker 27... 
kubectl is now configured to use "minikube" cluster and "default" namespace
```

## Step 6 — Verify the Minikube Cluster

```bash
kubectl get nodes
```

**Expected output:** Only **1 node** (Minikube by default creates a single-node cluster, unlike our 4-node KIND cluster):
```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.31.0
```

⚠️ **Important:** After installing Minikube, `kubectl` automatically switches its default context to point at the Minikube cluster. This is why `kubectl get nodes` now shows only 1 node instead of your 4-node KIND cluster.

## Step 7 — Switch Between Clusters Using Context

To check nodes on your **KIND** cluster specifically:
```bash
kubectl get nodes --context kind-k8s-in-one-shot
```
**Expected output:** Should show your original 4 KIND nodes again.

To permanently switch kubectl's active/default cluster:
```bash
kubectl config use-context kind-k8s-in-one-shot
```
**What this does:** Changes kubectl's default target cluster to your KIND cluster, so future commands (without `--context`) will act on KIND instead of Minikube.

**Verify:**
```bash
kubectl get nodes
```
**Expected output:** Now shows your 4 KIND nodes again by default.

## Step 8 — (Optional) Delete the Minikube Cluster

If you don't need Minikube anymore:
```bash
minikube delete
```
**What this does:** Completely removes the Minikube cluster and its resources.

### If Something Goes Wrong
If after deleting Minikube you run `kubectl get nodes` and see:
```text
The connection to the server ... was refused
```

**Reason:**
kubectl's current context is still pointing to the now-deleted Minikube cluster.

**Fix:**
```bash
kubectl config get-contexts
kubectl config use-context kind-k8s-in-one-shot
```
**What this does:** Lists all available contexts and switches back to your still-running KIND cluster.

## Final Result
You now understand:
- How to install and start a Minikube cluster
- That Minikube creates a single-node cluster by default
- How to check and switch between multiple cluster contexts using `kubectl config use-context` and `--context`
- How to delete a Minikube cluster when done

---
**Prerequisite for Chapter 13:** None extra — Chapter 13 (kubeadm) uses fresh, separate EC2 instances, not the KIND/Minikube setup from this chapter.
