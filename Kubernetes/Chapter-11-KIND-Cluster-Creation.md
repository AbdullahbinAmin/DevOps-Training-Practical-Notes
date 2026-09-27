# Practical 11 — Creating a KIND Cluster (with YAML Config File)

## Objective
Learn basic YAML syntax, then write a KIND cluster configuration file to create a 4-node Kubernetes cluster (1 control-plane/master node + 3 worker nodes), and finally create the cluster.

## Architecture / Flow
```text
YAML Config File → kind create cluster → Docker Containers (acting as Nodes) → Kubernetes Cluster Ready
```

## Prerequisites
- Docker, KIND, and kubectl installed (from Chapter 9 or 10)
- Logged into your AWS EC2 instance (or local machine) via terminal

## Part A — Quick YAML Basics

YAML stands for **YAML Ain't Markup Language**. File extension: `.yaml` or `.yml` (prefer `.yaml` — it is one character less to type but either works).

### 1. Key-Value Pair
```yaml
name: Shubham
```
Here `name` is the **key** and `Shubham` is the **value**.

### 2. List of Values (for one key)
```yaml
courses:
  - devops
  - aws
  - python
  - genai
```
Here `courses` is a key, and it holds a **list** of multiple values (each value starts with a dash `-`).

### 3. Object (Key holding multiple key-value pairs)
```yaml
info:
  name: Shubham
  age: 28
```
Here `info` is an **object** — a key that itself contains multiple key-value pairs inside it.

## Part B — Understand What We Are Building

We want to build a cluster exactly like the architecture diagram from Chapter 6/7, but with a small difference:
- Instead of just 1 worker node, we will create **3 worker nodes**
- Plus **1 control-plane (master) node**
- **Total = 4 nodes**

## Step 1 — Create a Project Folder

```bash
mkdir kind-cluster
cd kind-cluster
```
**What this does:** Creates a dedicated folder to keep your KIND cluster configuration file, then moves into it.

## Step 2 — Find the Correct API Version for KIND Config

⚠️ **Verification Required:** Always check the official KIND documentation for the current API version, since it can change between versions.

**Official reference:** https://kind.sigs.k8s.io/docs/user/configuration/

At the time of this recording, the API version used was:
```text
kind.x-k8s.io/v1alpha4
```

## Step 3 — Create the Cluster Config File

```bash
vim config.yaml
```

Paste the following complete configuration:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.31.0
  - role: worker
    image: kindest/node:v1.31.0
  - role: worker
    image: kindest/node:v1.31.0
  - role: worker
    image: kindest/node:v1.31.0
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

**Explanation of important fields:**
- `kind: Cluster` → tells KIND that this YAML file describes a **cluster configuration** (this is a different "kind" field than a Kubernetes Pod's `kind` — here it refers to the type of config file).
- `apiVersion` → the version of the KIND config format being used.
- `nodes` → a **list** of nodes to create.
- `role: control-plane` → marks this node as the Master Node (runs API Server, Scheduler, Controller Manager, etcd).
- `role: worker` → marks this node as a Worker Node (runs actual containers via Kubelet).
- `image: kindest/node:v1.31.0` → the Docker image used to create each node; `v1.31.0` was the latest stable Kubernetes version at the time of this recording.
- `extraPortMappings` → since the whole cluster runs inside a Docker container (and that container has its own internal ports separate from your EC2 instance's ports), you must map ports so outside traffic can reach your cluster.
  - `containerPort: 80` → the port inside the container (HTTP)
  - `hostPort: 80` → the port on your actual EC2/local machine that maps to it
  - `containerPort: 443` / `hostPort: 443` → same mapping for HTTPS
  - `protocol: TCP` → the network protocol used

⚠️ **Verification Required:** Confirm the latest stable Kubernetes version at https://github.com/kubernetes/kubernetes/releases before choosing the `kindest/node` image tag — this course used `v1.31.0` as the latest stable version at time of recording.

**Step 4:** Save and exit (`Esc`, then `:wq`, then Enter).

**Step 5:** Verify the file:
```bash
cat config.yaml
```
**Expected output:** The YAML content exactly as typed above.

## Step 4 — Create the Cluster

```bash
kind create cluster --name k8s-in-one-shot --config config.yaml
```

**What this does:**
- `--name k8s-in-one-shot` → gives your cluster a custom name (replace with any name you like)
- `--config config.yaml` → tells KIND to build the cluster exactly according to your YAML file

**Expected output (step by step, this will take a few minutes):**
```text
Creating cluster "k8s-in-one-shot" ...
 ✓ Ensuring node image (kindest/node:v1.31.0) 🖼
 ✓ Preparing nodes 📦📦📦📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-k8s-in-one-shot"
You can now use your cluster with:

kubectl cluster-info --context kind-k8s-in-one-shot
```

## Step 5 — Verify the Cluster

```bash
kubectl cluster-info --context kind-k8s-in-one-shot
```
**Expected output:** Should show the Kubernetes control plane running at a local address, like:
```text
Kubernetes control plane is running at https://127.0.0.1:xxxxx
```

```bash
kubectl get nodes
```
**Expected output:** A table listing 4 nodes — 1 with role `control-plane` and 3 workers, all with status `Ready`.

Example:
```text
NAME                            STATUS   ROLES           AGE   VERSION
k8s-in-one-shot-control-plane   Ready    control-plane   1m    v1.31.0
k8s-in-one-shot-worker          Ready    <none>          1m    v1.31.0
k8s-in-one-shot-worker2         Ready    <none>          1m    v1.31.0
k8s-in-one-shot-worker3         Ready    <none>          1m    v1.31.0
```

### If Something Goes Wrong
If `kubectl get nodes` shows an error like `The connection to the server ... was refused`:

**Reason:**
Your kubectl context might be pointing to a different/older cluster, or the cluster is not fully ready yet.

**Fix:**
```bash
kubectl config get-contexts
kubectl config use-context kind-k8s-in-one-shot
```
**What this does:** Lists all available cluster contexts, then switches kubectl to point at your KIND cluster specifically.

## Troubleshooting

### Error: "permission denied" while creating cluster
**Reason:** Your user does not have Docker permissions.
**Fix:**
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Error: cluster creation hangs or fails on "Preparing nodes"
**Reason:** Not enough resources (CPU/RAM) on your machine/instance.
**Fix:** Use at least a `t2.medium` (2 vCPU, 4 GB RAM) instance. Delete and recreate the cluster once resources are sufficient:
```bash
kind delete cluster --name k8s-in-one-shot
kind create cluster --name k8s-in-one-shot --config config.yaml
```

## Final Result
You now have a fully working 4-node Kubernetes cluster (1 control-plane + 3 workers) running locally inside Docker containers via KIND, verified using `kubectl get nodes`.

---
**Prerequisite for Chapter 12:** This KIND cluster should remain running, since Chapter 12 will create a *second* cluster using Minikube, and you will practice switching between the two using `kubectl` contexts.
