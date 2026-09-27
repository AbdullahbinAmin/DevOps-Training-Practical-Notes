# Practical 13 — Kubeadm: Setup, Installation, and Cluster Creation

## Objective
Build a real, manual, "bare-metal style" Kubernetes cluster using **kubeadm**, with one Master Node and one Worker Node, both running on separate AWS EC2 instances. This is closer to how production clusters are actually built.

## Architecture / Flow
```text
2 EC2 Instances (Master + Worker) → Install containerd + kubeadm/kubelet/kubectl on both
→ kubeadm init on Master → Install Calico CNI → kubeadm join from Worker → Cluster Ready
```

## Prerequisites
- AWS account
- 2 EC2 instances needed:
  - 1 for Master Node
  - 1 for Worker Node
- Instance type: minimum **t2.medium** for both
- OS: Ubuntu 24.04
- Storage: 10 GB is enough for this demo
- Security Group: both instances must be in the **same security group**
- Required port: **6443** (used for worker nodes to join the master) must be opened

## Step 1 — Launch 2 EC2 Instances

**Step 1:** Open AWS Console → EC2 → Launch Instance.

**Step 2:** Configure:
- Name: `master-k8s` (you will rename the second one after launch)
- AMI: Ubuntu 24.04, x86 architecture
- Key pair: reuse your existing key (e.g., `k8s-in-one-shot`) or create a new one
- Instance type: `t2.medium`
- Number of instances: **2**
- Storage: 10 GB

**Step 3:** Click **Launch Instance**.

**Step 4:** After launch, rename the instances for clarity:
- Rename one to `master-k8s`
- Rename the other to `worker-k8s`

(Tip: use the **Running instances** filter in the EC2 console to find them quickly, and give them different tag colors to avoid confusion.)

## Step 2 — Open Required Port in Security Group

**Step 1:** Go to your instance → **Security** tab → click the attached Security Group.

**Step 2:** Click **Edit inbound rules**.

**Step 3:** Click **Add rule**:
- Type: Custom TCP
- Port range: `6443`
- Source: Anywhere (`0.0.0.0/0`) — for demo purposes only
- Description: `kubeadm joining port`

**Step 4:** Click **Save rules**.

⚠️ **Note:** Opening port `6443` to "Anywhere" is fine for a classroom demo, but in a real production environment, restrict this to only trusted IP ranges.

## Step 3 — Connect to Both Instances via SSH

Open two separate terminal windows/tabs — one for Master, one for Worker.

**For Master:**
```bash
ssh -i "<KEY_NAME>.pem" ubuntu@<MASTER_PUBLIC_IP>
```

**For Worker:**
```bash
ssh -i "<KEY_NAME>.pem" ubuntu@<WORKER_PUBLIC_IP>
```

Replace `<KEY_NAME>` with your `.pem` key file name, and `<MASTER_PUBLIC_IP>` / `<WORKER_PUBLIC_IP>` with the actual public IPs shown in the AWS Console for each instance.

**Expected result:** Two separate terminal sessions, one logged into each server.

---

## PART A — Steps to Run on BOTH Master AND Worker

Run every command below on **both** terminals (Master and Worker), one at a time.

### Step 4 — Disable Swap Memory

```bash
sudo swapoff -a
```
**What this does:** Kubernetes requires swap memory to be disabled for the kubelet to work correctly.

### Step 5 — Load Required Kernel Modules for Networking

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```
**What this does:** Loads two kernel modules (`overlay` and `br_netfilter`) that are required for Kubernetes networking to work correctly between pods and nodes.

### Step 6 — Set Required sysctl (Networking) Parameters

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```
**What this does:** Configures IP tables and IP forwarding settings needed for Kubernetes networking, since here we have multiple separate instances that need to talk to each other (unlike Minikube/KIND, where everything runs on one single instance).

### Step 7 — Install containerd

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y containerd.io

sudo systemctl restart containerd
sudo systemctl status containerd
```

**What this does:**
- Installs required certificate/key tools
- Adds Docker's official GPG key (containerd is distributed via Docker's repository)
- Adds Docker's APT repository
- Installs `containerd.io` — this is the actual container runtime that Kubernetes' control plane (etcd, API Server, Scheduler, Controller Manager) and worker node (Kubelet, Service Proxy) components will run on top of, instead of Docker directly
- Restarts and checks the status of containerd

**Expected result:** `containerd` should show status `active (running)`.

⚠️ **Verification Required:** Check the official containerd/Docker installation guide for your Ubuntu version if this repository setup changes: https://docs.docker.com/engine/install/ubuntu/

### Step 8 — Install kubeadm, kubelet, and kubectl

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**What this does:**
- Installs prerequisite tools
- Adds Kubernetes' official GPG signing key
- Adds Kubernetes' official APT repository for version **v1.29** (you may use a newer stable version like v1.32 — just change the version number in the URL above; check the official Kubernetes release page for the latest)
- Installs `kubelet`, `kubeadm`, and `kubectl`
- `apt-mark hold` locks these packages at their current version so they don't get accidentally auto-upgraded by future `apt upgrade` commands (which could break the cluster)

**Official reference:** https://kubernetes.io/docs/tasks/tools/install-kubeadm/

⚠️ **Verification Required:** Confirm the current stable Kubernetes minor version (e.g., v1.29, v1.31, v1.32) from the official docs before running this, since the URL path changes with each release line.

### Step 9 — Verify Installation (on both Master and Worker)

```bash
kubectl version --client
kubeadm version
kubelet --version
```
**Expected output:** All three commands should print version numbers with no errors.

---

## PART B — Steps to Run ONLY on the Master Node

### Step 10 — Initialize the Master Node

```bash
sudo kubeadm init
```

**What this does:** Turns this server into the **Master Node (control plane)**. It installs and starts the API Server, Scheduler, Controller Manager, and etcd — exactly the components explained in Chapter 6.

⚠️ **Important:** Only run `kubeadm init` on the server you intend to be the Master. If you accidentally run this on the Worker node, it will also try to become a master — use `sudo kubeadm reset` to undo it (see Troubleshooting section below).

**Expected output (partial):**
```text
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

...

kubeadm join <MASTER_IP>:6443 --token <TOKEN> \
	--discovery-token-ca-cert-hash sha256:<HASH>
```

⚠️ **IMPORTANT:** Copy and save the full `kubeadm join ...` command shown at the bottom of this output — you will need it to connect the Worker node in Step 13.

### Step 11 — Set Up kubectl Access for Your Normal User

By default, `kubeadm init` was run using `sudo` (root user), but you should not always run cluster commands as root.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**What this does:**
- Creates a `.kube` folder in your normal user's home directory
- Copies the cluster's admin config file into it
- Changes ownership of that config file to your current user, so you (e.g., `ubuntu`) can run `kubectl` commands without needing `sudo` every time

### Step 12 — Install the CNI Network Plugin (Calico)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

**What this does:** Installs **Calico**, a Container Network Interface (CNI) plugin, which allows the Master Node and Worker Node(s) to communicate with each other over the network (this is the same CNI concept explained in Chapter 6).

⚠️ **Verification Required:** Check the official Calico documentation for the latest compatible version before installing: https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

**Expected output:** A long list of resources being created — `customresourcedefinition`, `clusterrole`, `deployment`, etc. — ending without errors.

---

## PART C — Steps to Run ONLY on the Worker Node

### Step 13 — Join the Worker Node to the Cluster

⚠️ **Important:** If you accidentally ran `kubeadm init` on the worker node by mistake, reset it first:
```bash
sudo kubeadm reset
```
This clears any accidental master-node setup so the server is clean before joining.

On the **Worker** terminal, run the `kubeadm join` command you copied from Step 10 (from the Master's output), with `sudo`:

```bash
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> \
	--discovery-token-ca-cert-hash sha256:<HASH>
```
- `<MASTER_IP>` → the Master node's IP address
- `<TOKEN>` → the join token generated by `kubeadm init`
- `<HASH>` → the discovery token CA certificate hash generated by `kubeadm init`

**What this does:** Connects this server to the cluster as a Worker Node. This starts the Kubelet service on this node.

**Expected output:**
```text
This node has joined the cluster:
...
```

### If the Join Token Expired
If too much time has passed and the token is no longer valid, generate a new one **on the Master**:
```bash
kubeadm token create --print-join-command
```
**What this does:** Generates a brand-new join command with a fresh token, which you can then run on the Worker node.

---

## Step 14 — Final Verification (on Master)

```bash
kubectl get nodes
```
**Expected output (initially the worker may show "NotReady" for a few seconds while it finishes setup):**
```text
NAME         STATUS     ROLES           AGE   VERSION
master-k8s   Ready      control-plane   5m    v1.29.x
worker-k8s   NotReady   <none>          10s   v1.29.x
```

To watch it live until it becomes Ready:
```bash
kubectl get nodes --watch
```
**What this does:** Refreshes the node status automatically every couple of seconds so you can see the moment the worker node becomes `Ready`.

**Expected final output:**
```text
NAME         STATUS   ROLES           AGE   VERSION
master-k8s   Ready    control-plane   6m    v1.29.x
worker-k8s   Ready    <none>          1m    v1.29.x
```

## Step 15 — Test the Cluster

```bash
kubectl run nginx --image=nginx:latest
```
**What this does:** Creates a test Pod running the Nginx image, to confirm the cluster can actually schedule and run workloads.

**Expected output:**
```text
pod/nginx created
```

Verify:
```bash
kubectl get pods
```
**Expected output:** `nginx` pod should show status `Running`.

## Troubleshooting

### Error: worker node accidentally became a master (ran `kubeadm init` by mistake)
**Reason:** `kubeadm init` was run on the wrong node.
**Fix:**
```bash
sudo kubeadm reset
```
Then run the correct `kubeadm join` command on that node.

### Error: `kubectl get nodes` on Worker says "connection refused" or cluster not found
**Reason:** kubectl is only configured with cluster access on the Master by default (via `$HOME/.kube/config`), not automatically on the Worker.
**Fix:** This is expected — you generally run `kubectl` commands from the Master node, since that's where the admin config was copied in Step 11.

### Error: Nodes stuck in "NotReady" for a long time
**Reason:** CNI (Calico) may not have installed correctly, or nodes cannot reach each other on required ports.
**Fix:**
```bash
kubectl get pods -n kube-system
```
Check if Calico pods are running properly. Also double check that port `6443` and Calico's required ports are open in your Security Group.

## Final Result
You now have a real 2-node Kubernetes cluster (1 Master + 1 Worker) built manually with `kubeadm`, using `containerd` as the runtime and `Calico` as the CNI — and you successfully ran a test Nginx pod on it.

---
**Prerequisite for Chapter 14:** A working cluster (KIND, Minikube, or kubeadm) — Chapter 14 (Namespaces) continues using the **KIND cluster from Chapter 11**.
