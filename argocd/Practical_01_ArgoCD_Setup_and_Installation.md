# Practical 1: ArgoCD Setup and Installation (EC2 + Kind + ArgoCD UI and CLI)

## Objective

Create an EC2 server in AWS, connect to it, install Docker, Kind and kubectl, create a Kind cluster, install ArgoCD using the official manifest, open the ArgoCD UI in the browser, and log in with the ArgoCD CLI.

## Prerequisites

* **AWS account** with permission to launch EC2 instances
* **Browser** (Chrome / Firefox / Edge)
* **Laptop terminal** (only if you connect by SSH; not needed for the browser connect option)
* **EC2 server (we create it in Step 1):**
  * OS: Ubuntu Server LTS (22.04 or 24.04)
  * Instance type: `t3.large` (2 vCPU, 8 GB RAM) recommended. Minimum `t3.medium` (4 GB RAM), but it can be slow because we run 3 Kind nodes.
  * Disk: 30 GiB
* **Required ports (Security Group inbound rules):**
  * `22` → SSH (to connect to the server)
  * `8080` → ArgoCD UI (port-forward)
  * `33893` → Kind API server (used **inside** the server only; **do not** open it in the Security Group)
* **Software (installed on the server in Step 4 and Step 8):** Docker, Kind, kubectl, ArgoCD CLI
* **Versions used:**
  * Kubernetes node image: `kindest/node:v1.33.1`
  * Kind: `v0.33.0` (stable version shown on the official Kind Quick Start page at the time of writing)
  * kubectl: `v1.33.1` (same version as the cluster)

> ⚠️ **Where to run commands:** Step 1 and Step 2 are done in the **AWS Console (browser)**. From Step 3 onwards, all commands are run **inside the EC2 server terminal**, unless written otherwise.

## Architecture / Flow

```text
Student → AWS Console → EC2 Server (Ubuntu) → Docker → Kind Cluster (1 control-plane + 2 workers)
        → ArgoCD (namespace: argocd) → Port-forward 8080 → Browser / ArgoCD CLI
```

---

## Step 1 — Create the EC2 Server (AWS Console)

**What to do:** Launch one Ubuntu server that will run everything.

> The AWS Console screen may look a little different depending on the console version. The option names below are the usual ones.

**Step 1:** Open the AWS Console and sign in.

**Step 2:** At the top right, choose a **Region** (for example, the region closest to you). Remember it, because you must use the same region when you come back later.

**Step 3:** In the search bar at the top, type `EC2` and open **EC2**.

**Step 4:** Click **Instances** in the left menu, then click **Launch instances** (orange button).

**Step 5:** Enter the Name:

* Name: `argocd-server`

**Step 6:** In **Application and OS Images (Amazon Machine Image)**:

* Click **Ubuntu**
* Choose **Ubuntu Server 24.04 LTS** (or 22.04 LTS)
* Architecture: **64-bit (x86)**

**Step 7:** In **Instance type**:

* Choose `t3.large` (recommended)
* If you want a cheaper option, use `t3.medium` (minimum)

**Step 8:** In **Key pair (login)**:

* Click **Create new key pair**
* Key pair name: `argocd-key`
* Key pair type: `RSA`
* Private key file format: `.pem`
* Click **Create key pair**

The file `argocd-key.pem` is downloaded to your laptop. **Keep this file safe.** You cannot download it again. (Even if you will connect only with the browser option in Step 3, create the key pair because the launch form asks for it.)

**Step 9:** In **Network settings**, click **Edit**, then set:

* Auto-assign public IP: **Enable**
* Firewall (security groups): **Create security group**
* Security group name: `argocd-sg`
* Description: `ArgoCD practical`

**Step 10:** Under **Inbound security group rules**, set up two rules:

* **Rule 1 (already there):**
  * Type: `SSH`
  * Port: `22`
  * Source type: `My IP` (or `Anywhere` for a classroom demo only)
* Click **Add security group rule**, then enter **Rule 2:**
  * Type: `Custom TCP`
  * Port range: `8080`
  * Source type: `My IP` (or `Anywhere` for a classroom demo only)

**Step 11:** In **Configure storage**:

* Size: `30` GiB
* Type: `gp3`

**Step 12:** Check the **Summary** panel on the right. Click **Launch instance**.

**Step 13:** Click **View all instances** (or **Instances** in the left menu).

**Step 14:** Wait until:

* **Instance state:** `Running`
* **Status check:** `2/2 checks passed` (this takes 1 to 2 minutes)

**Step 15:** Click your instance `argocd-server`. In the details panel, copy and note down:

* **Public IPv4 address** → we call it `<EC2_PUBLIC_IP>` in these notes
* **Private IPv4 address** → we call it `<EC2_PRIVATE_IP>` in these notes (you can also get it later with `hostname -I`)

> ⚠️ **Public IP changes:** If you **Stop** and **Start** the instance later, the Public IP changes. Copy the new IP from the console each time.

**Expected:** One instance named `argocd-server` in `Running` state with `2/2 checks passed`.

---

## Step 2 — Check the Security Group (AWS Console)

**What to do:** Make sure port `22` and port `8080` are open. (If you did Step 10 above exactly, just verify. Use this step to fix if something is missing.)

**Step 1:** In **EC2 → Instances**, select your instance `argocd-server`.

**Step 2:** Click the **Security** tab. Under **Inbound rules**, check that these exist:

* `22` (SSH)
* `8080` (Custom TCP)

**Step 3 (only if `8080` is missing):**

* Click the **Security group** name (for example `argocd-sg`)
* Click **Edit inbound rules → Add rule**
* Type: `Custom TCP`, Port range: `8080`, Source: `My IP` (or `Anywhere-IPv4` for a classroom demo only)
* Click **Save rules**

**Expected:** Both ports `22` and `8080` are listed in the inbound rules.

---

## Step 3 — Connect to the EC2 Server

Use **Option A** (browser, easiest) **or** **Option B** (SSH from your laptop). Do not do both.

### Option A — Connect from the Browser (EC2 Instance Connect)

**Step 1:** In **EC2 → Instances**, select `argocd-server`.

**Step 2:** Click **Connect** (top right).

**Step 3:** Open the tab **EC2 Instance Connect**. Keep the default user name `ubuntu`.

**Step 4:** Click **Connect**.

**Expected:** A terminal opens in a new browser tab with a prompt like `ubuntu@ip-172-31-x-x:~$`.

> ⚠️ Keep this browser tab open during the class. If you close it, background jobs like the port-forward may stop. If the tab does not connect, use Option B.

### Option B — Connect with SSH from your Laptop

**Step 1:** Open a terminal on your laptop (Linux/Mac: Terminal. Windows: PowerShell).

**Step 2:** Go to the folder where `argocd-key.pem` was downloaded (usually Downloads):

```bash
cd ~/Downloads
```

(On Windows PowerShell use: `cd $HOME\Downloads`)

**Step 3 (Linux/Mac only):** Set the key file permission:

```bash
chmod 400 argocd-key.pem
```

**What this does:** SSH refuses a key file that other users can read. This makes it private.

**Step 4:** Connect:

```bash
ssh -i argocd-key.pem ubuntu@<EC2_PUBLIC_IP>
```

**Replace:**

* `<EC2_PUBLIC_IP>` → the Public IPv4 address from Step 1 (Step 15)

**Step 5:** When asked `Are you sure you want to continue connecting?`, type `yes` and press `Enter`.

**Expected:** Prompt changes to `ubuntu@ip-172-31-x-x:~$`.

### Check you are inside the server

```bash
whoami
hostname
cat /etc/os-release
```

**Expected:** `whoami` shows `ubuntu`. OS shows Ubuntu.

> ✅ **From here, every command runs on this EC2 server terminal.**

---

## Step 4 — Install Prerequisites on the Server

### 4.1 Update packages

```bash
sudo apt-get update
```

**What this does:** Refreshes the package list.

### 4.2 Check the CPU type (needed to pick the correct download)

```bash
uname -m
```

**Expected:** `x86_64` (this is amd64). If it shows `aarch64`, use the `arm64` commands where they are given below.

### 4.3 Install Docker

```bash
sudo apt install docker.io -y
```

**What this does:** Installs Docker.

```bash
sudo systemctl enable --now docker
```

**What this does:** Starts the Docker service and makes it start at boot.

```bash
sudo usermod -aG docker $USER && newgrp docker
```

**What this does:** Adds your user to the `docker` group so you can run Docker without `sudo`. `newgrp docker` applies the change in the current terminal.

**Verify:**

```bash
docker --version
docker ps
```

**Expected:** `docker --version` shows a version. `docker ps` shows an empty table (headers only) and **no permission error**.

> If `docker ps` shows a permission error, close the terminal, connect to the server again (Step 3), and run `docker ps` again.

### 4.4 Install Kind

**Official Installation Guide:** Kind Quick Start → "Installing From Release Binaries" (`https://kind.sigs.k8s.io/docs/user/quick-start/`). The commands below are from that page.

**For x86_64 (AMD64) servers:**

```bash
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-amd64
```

**For aarch64 (ARM64) servers only:**

```bash
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-arm64
```

**What this does:** Downloads the Kind binary into the current folder. (Run only the line that matches your CPU type. The other line will do nothing.)

```bash
chmod +x ./kind
```

**What this does:** Makes the file executable.

```bash
sudo mv ./kind /usr/local/bin/kind
```

**What this does:** Moves Kind into `/usr/local/bin` so you can run `kind` from anywhere.

**Verify:**

```bash
kind version
```

**Expected:** Prints something like `kind v0.33.0 ...`.

> ⚠️ Verification Required: The stable Kind version on the official page can change with time. If the page shows a newer version, replace `v0.33.0` in the URL with the version on the page. Before class, run the cluster creation once (Step 5.3) to confirm that the node image `kindest/node:v1.33.1` works with your Kind version. If it fails, open the Kind releases page on GitHub, find your Kind version, and use one of the node images listed in its release notes.

### 4.5 Install kubectl

**Official Installation Guide:** Kubernetes docs → "Install and Set Up kubectl on Linux" (`https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/`). Below we use the same download method shown on that page, but with the version pinned to `v1.33.1`, the same version as our cluster.

**For x86_64 (AMD64) servers:**

```bash
curl -LO "https://dl.k8s.io/release/v1.33.1/bin/linux/amd64/kubectl"
```

**For aarch64 (ARM64) servers only:**

```bash
curl -LO "https://dl.k8s.io/release/v1.33.1/bin/linux/arm64/kubectl"
```

**What this does:** Downloads the `kubectl` binary (run only the line that matches your CPU type).

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

**What this does:** Installs `kubectl` into `/usr/local/bin` with the correct permissions.

```bash
rm kubectl
```

**What this does:** Removes the downloaded file from the current folder.

**Verify:**

```bash
kubectl version --client
```

**Expected:** `Client Version: v1.33.1`.

### 4.6 Pre-Check (run all)

```bash
docker --version
kind version
kubectl version --client
```

**Expected:** All three commands print versions with no error.

---

## Step 5 — Create the Kind Cluster

### 5.1 Find your EC2 private IP

```bash
hostname -I
```

**What this does:** Prints the IP addresses of the machine. The first one is the private IP (for example `172.31.x.x`). It must match the Private IPv4 address in the EC2 dashboard.

### 5.2 Create the config file

```bash
nano kind-config.yaml
```

Paste this complete content:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "<EC2_PRIVATE_IP>"
  apiServerPort: 33893
nodes:
  - role: control-plane
    image: kindest/node:v1.33.1
  - role: worker
    image: kindest/node:v1.33.1
  - role: worker
    image: kindest/node:v1.33.1
```

**Replace:**

* `<EC2_PRIVATE_IP>` → your EC2 private IP from `hostname -I`. Keep the double quotes. (The original guide has `172.31.19.178`. **You must change it to your own IP.**)

**Important fields:**

* `apiServerAddress` and `apiServerPort` → fix the API server address and port. This avoids random localhost ports, so ArgoCD pods can reach the API server.
* `nodes` → 1 control-plane and 2 worker nodes.

Save and exit: press `Ctrl + O`, `Enter`, then `Ctrl + X`.

**Verify the file:**

```bash
cat kind-config.yaml
```

**Expected:** Your IP is inside the quotes.

### 5.3 Create the cluster

```bash
kind create cluster --name argocd-cluster --config kind-config.yaml
```

**What this does:** Creates a cluster named `argocd-cluster`. It downloads the node images, so it may take a few minutes.
**Expected:** Ends with a message like "Set kubectl context to kind-argocd-cluster".

### 5.4 Verify the cluster

```bash
kubectl cluster-info
kubectl get nodes
```

**Expected:** 3 nodes (1 control-plane, 2 workers), all in `Ready` status. If they show `NotReady`, wait 1 minute and run again.

---

## Step 6 — Install ArgoCD (Official Manifest)

### 6.1 Create the namespace

```bash
kubectl create namespace argocd
```

**What this does:** Creates a separate namespace for ArgoCD (do not install into `default`).
**Expected:** `namespace/argocd created`.

### 6.2 Apply the ArgoCD manifest

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**What this does:** Installs all ArgoCD components from the official manifest. The `stable` link installs the latest stable version.
**Expected:** Many lines ending with `created`.

### 6.3 Verify

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

**Expected:** All pods become `Running` (1 to 3 minutes). Run the command again to check. You should see the service `argocd-server`.

---

## Step 7 — Open the ArgoCD UI

### 7.1 Port-forward

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address=0.0.0.0 &
```

**What this does:** Forwards port `8080` on the EC2 server to ArgoCD server port `443`. `--address=0.0.0.0` allows access from outside the server. `&` runs it in the background.
**Expected:** `Forwarding from 0.0.0.0:8080 -> 8080`. Press `Enter` to get the prompt back.

> Do not close this terminal. If you close it, the port-forward stops.

### 7.2 Get the initial admin password

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

**What this does:** Reads the admin password from the secret and decodes it.
**Expected:** A random password is printed. Copy it.

### 7.3 Login from the browser (GUI)

**Step 1:** Open a **new browser tab** on your laptop and go to:
`https://<EC2_PUBLIC_IP>:8080`

**Replace:**

* `<EC2_PUBLIC_IP>` → the Public IPv4 address of your EC2 instance (from the EC2 dashboard). Use `https`, not `http`.

**Step 2:** The browser will show a security warning (self-signed certificate). Click **Advanced → Proceed** (wording depends on your browser).

**Step 3:** Enter:

* Username: `admin`
* Password: the password from Step 7.2

**Step 4:** Click **Sign In**.

**Expected:** The ArgoCD dashboard opens (Applications page).

---

## Step 8 — Install and Use the ArgoCD CLI

### 8.1 Download and install

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
```

**What this does:** Downloads the ArgoCD CLI binary. (If `uname -m` showed `aarch64`, download the `arm64` file from the ArgoCD releases page on GitHub instead.)

```bash
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
```

**What this does:** Installs it into `/usr/local/bin` with read and execute permission, so it is in your PATH.

```bash
rm argocd-linux-amd64
```

**What this does:** Removes the downloaded file.

### 8.2 Verify

```bash
argocd version --client
```

**Expected:** The client version is printed.

### 8.3 Login with the CLI

```bash
argocd login <EC2_PUBLIC_IP>:8080 --username admin --password <INITIAL_PASSWORD> --insecure
```

**Replace:**

* `<EC2_PUBLIC_IP>` → your EC2 public IP (the same one used in the browser).
* `<INITIAL_PASSWORD>` → the password from Step 7.2.

**Why `--insecure`:** Port-forward uses a self-signed TLS certificate. In production, use a proper certificate and remove this flag.
**Expected:** `'admin:login' logged in successfully`.

### 8.4 Get user info

```bash
argocd account get-user-info
```

**Expected:** Shows `Logged In: true` and `Username: admin`.

---

## Step 9 — Final Testing

Run these on the server:

```bash
kubectl get nodes
kubectl get pods -n argocd
argocd account get-user-info
```

**Expected:**

* 3 nodes `Ready`
* All ArgoCD pods `Running`
* CLI shows you are logged in as `admin`
* The browser shows the ArgoCD dashboard

---

## Step 10 — Troubleshooting

### Error 1

```text
permission denied while trying to connect to the Docker daemon socket
```

**Reason:** Your user is not in the `docker` group yet, or the group is not applied.

**Fix:**

```bash
sudo usermod -aG docker $USER && newgrp docker
```

If it still fails, close the terminal and connect to the server again (Step 3).

### Error 2

```text
Permissions 0644 for 'argocd-key.pem' are too open.
```

**Reason:** The key file is readable by other users (SSH, Option B).

**Fix:**

```bash
chmod 400 argocd-key.pem
```

### Error 3

SSH: `Connection timed out`

**Reason:** Port `22` is not open in the Security Group, the Public IP is wrong or changed, or your own IP changed (if the rule is `My IP`).

**Fix (AWS Console):** Repeat Step 2. Copy the current Public IPv4 address from the EC2 dashboard. Edit the inbound rule for SSH and choose `My IP` again.

### Error 4

```text
kind: command not found
```

**Reason:** Kind was not moved to `/usr/local/bin`, or the download failed (for example, wrong CPU type line was used).

**Fix:** Check the CPU type with `uname -m`, then repeat Step 4.4 with the matching download line.

### Error 5

Kind cluster creation fails, or `kubectl` cannot connect.

**Reason:** Wrong `apiServerAddress` in `kind-config.yaml`.

**Fix:** Run `hostname -I`, correct the IP in the file, then delete and recreate the cluster:

```bash
kind delete cluster --name argocd-cluster
kind create cluster --name argocd-cluster --config kind-config.yaml
```

### Error 6

Browser cannot open `https://<EC2_PUBLIC_IP>:8080` (timeout).

**Reason:** Port 8080 is not open in the Security Group, the Public IP changed, or the port-forward is not running.

**Fix:** Repeat Step 2. Check the port-forward:

```bash
ps aux | grep port-forward
```

If it is not running, run the port-forward command from Step 7.1 again.

### Error 7

Pods are stuck in `Pending` or `ContainerCreating` for a long time.

**Reason:** Image pulling is slow, or the server is low on resources.

**Fix:**

```bash
kubectl describe pod <POD_NAME> -n argocd
```

Replace `<POD_NAME>` with the pod name from `kubectl get pods -n argocd`. Read the Events section at the bottom. If resources are low, use a bigger instance type (for example `t3.large`).

### Error 8

```text
Error from server (NotFound): secrets "argocd-initial-admin-secret" not found
```

**Reason:** ArgoCD is not fully ready yet, or the secret was deleted after the password was changed.

**Fix:** Wait for all pods to be `Running`, then try again.

### Error 9

```text
bind: address already in use
```

**Reason:** An old port-forward is still using port 8080.

**Fix:**

```bash
pkill -f "port-forward"
```

Then run the port-forward command again.

### Error 10

```text
metadata.annotations: Too long: must have at most 262144 bytes
```

**Reason:** One large ArgoCD resource is too big for normal `kubectl apply`.

**Fix:** Apply again with server-side apply:

```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

> ⚠️ Verification Required: This error was not in the original guide. It can appear with some ArgoCD versions. Run Step 6.2 once before class to check that plain `kubectl apply` works for you.

---

## Step 11 — Cleanup (only after the class)

### 11.1 On the server

```bash
pkill -f "port-forward"
kind delete cluster --name argocd-cluster
```

**What this does:** Stops the port-forward and deletes the whole Kind cluster.

### 11.2 In the AWS Console (important, to avoid extra cost)

**Step 1:** Go to **EC2 → Instances**.

**Step 2:** Select `argocd-server`.

**Step 3:** Click **Instance state → Terminate (delete) instance**.

**Step 4:** Click **Terminate (delete)** to confirm.

**Step 5 (optional):** Delete the security group `argocd-sg` (EC2 → Security Groups) and the key pair `argocd-key` (EC2 → Key Pairs) if you do not need them.

> **Stop** only pauses the server (disk cost continues, and the Public IP changes on restart). **Terminate** deletes it.

---

## Final Result

At the end, these must be working:

* An EC2 Ubuntu server `argocd-server` in `Running` state
* Docker, Kind and kubectl installed on the server
* A Kind cluster `argocd-cluster` with 3 `Ready` nodes
* ArgoCD running in the `argocd` namespace
* ArgoCD UI open in the browser at `https://<EC2_PUBLIC_IP>:8080`
* ArgoCD CLI installed and logged in as `admin`

## Classroom Safety Check

- [x] AWS account, region, and EC2 creation (AMI, type, key pair, storage)
- [x] Security group ports 22 and 8080
- [x] Connect to server (browser and SSH, key permission)
- [x] Prerequisites and pre-check covered
- [x] Docker, Kind and kubectl install commands (Kind commands from the official Kind page)
- [x] Config file complete, with placeholder explained
- [x] One ArgoCD install method (official manifest)
- [x] GUI login steps
- [x] CLI install and login
- [x] Verification, testing, troubleshooting
- [x] Cleanup (cluster and EC2 termination)
