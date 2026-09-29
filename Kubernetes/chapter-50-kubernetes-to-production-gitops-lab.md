# Chapter 50: Kubernetes to Production with EKS Auto Mode, Terraform, Argo CD and GitHub Actions (Practical Lab)

## What you will build

You will deploy a real multi-service e-commerce application (the **Retail Store Sample App**) on **Amazon EKS Auto Mode**, then automate its delivery with **GitOps**.

```text
 Developer ──push──▶ GitHub (gitops branch)
                        │
                        ▼
                 GitHub Actions (CI)
        detect changes → docker build → push to Amazon ECR
                        │
                        ▼
        update image tag in Helm values.yaml → commit back to Git
                        │
                        ▼
              Argo CD (watches the Git repo)
                        │  syncs
                        ▼
        Amazon EKS Auto Mode cluster  ◀── built by Terraform
        (NGINX Ingress + cert-manager + 5 microservices)
                        │
                        ▼
              AWS Load Balancer → your browser
```

| Tool | Role |
|---|---|
| **AWS CLI** | Talk to AWS from the terminal |
| **Terraform** | Builds the VPC, EKS cluster, Argo CD, NGINX Ingress and cert-manager from code |
| **Helm** | Packages the Kubernetes manifests of each service (charts) |
| **Argo CD** | Deploys whatever is in Git to the cluster (GitOps) |
| **NGINX Ingress Controller** | Sends outside traffic to the correct service |
| **cert-manager** | Issues SSL/TLS certificates |
| **GitHub Actions** | Builds Docker images and updates Helm values automatically |
| **Amazon ECR** | Private registry that stores your Docker images |

The lab has three phases:

| Phase | Goal |
|---|---|
| **Phase 1** | Build the infrastructure with Terraform and open the running app (public images) |
| **Phase 2** | Read and understand the code, Dockerfiles, Helm charts and how services connect |
| **Phase 3** | Build the CI pipeline with GitHub Actions so that a `git push` deploys your own private images |

### Cost warning (read before you start)

> **This lab costs real money.** EKS control plane, EC2 nodes, a NAT gateway, and an AWS load balancer all bill by the hour. Expect a few US dollars for a full day. Finish the lab in one or two sittings and run **`terraform destroy`** (see the Cleanup part) when you are done. Then also delete the ECR repositories and IAM access keys by hand.

### Two important notes about versions

1. **Kubernetes version.** EKS moves old Kubernetes versions into paid *extended support*, which costs several times more per hour. Before you run Terraform, open `terraform/variables.tf` and set the Kubernetes version to one in **standard support** (this guide uses `1.35`). Part 6 shows how.
2. **NGINX Ingress retirement.** The upstream Kubernetes project retired *ingress-nginx* in March 2026: it receives no more fixes or security patches. This project still uses it, so it works for learning, but do **not** copy this pattern to real production without a migration plan (for example to the Gateway API or another ingress controller).

---

## Prerequisites

### Accounts

| Account | Why | How to get it |
|---|---|---|
| **AWS account** | Runs EKS, ECR, VPC | `https://aws.amazon.com`. Needs a payment method |
| **GitHub account** | Stores your fork of the code and runs GitHub Actions | `https://github.com/signup` |

### Workstation

Ubuntu/Debian Linux is used in the commands (Ubuntu 22.04 or 24.04, or WSL2 on Windows). macOS alternatives (Homebrew) are noted where they differ. You need about 10 GB of free disk space and a stable internet connection.

### Tools

| Tool | Minimum | Needed for |
|---|---|---|
| Git | 2.x | Cloning and pushing code |
| AWS CLI | v2 | Talking to AWS |
| Terraform | 1.5 or newer | Building the infrastructure |
| kubectl | within one minor version of the cluster (1.34 to 1.36) | Talking to Kubernetes |
| Helm | 3.x | Inspecting charts (Terraform installs charts by itself) |
| Docker | 20+ | **Only** for the optional manual ECR demo in Phase 3 |
| A code editor | VS Code recommended | Reading and editing files |

Installation is in Part 2. After installing, verify all of them:

```bash
git --version
aws --version
terraform version
kubectl version --client
helm version
docker --version
```

### Permissions

- An IAM user with **AdministratorAccess** for the Terraform part (created in Part 1). This is acceptable for learning only.
- A **separate**, limited IAM user for GitHub Actions (created in Phase 3).

### Ports and network

Your workstation needs **outbound** internet access only. No inbound ports need to be opened by you. Terraform creates the load balancer and security rules automatically.

| Port | Protocol | Purpose | Where |
|---|---|---|---|
| 80 | TCP | HTTP to the shop through the load balancer | AWS load balancer to NGINX Ingress |
| 443 | TCP | HTTPS (used when you add a domain and certificate) | AWS load balancer to NGINX Ingress |
| 8080 (local) | TCP | Your browser to Argo CD through `kubectl port-forward` | Your workstation only |
| 30000-32767 | TCP | Kubernetes NodePort range, opened by `terraform/security.tf` | Between load balancer and nodes |

### Secrets you will handle

| Secret | Where it lives | Never do this |
|---|---|---|
| AWS access keys (admin user) | `~/.aws/credentials` on your machine | Commit to Git or share |
| AWS access keys (GitHub Actions user) | **GitHub repository secrets** | Put in any file in the repo |
| Argo CD admin password | Kubernetes secret in the cluster | Reuse elsewhere |

Use placeholders like `YOUR_AWS_ACCOUNT_ID` in files you share. Never paste real keys anywhere public.

---

## The application: "The Most Public Secret Shop"

A sample retail store. In the browser you can browse a catalog, open a product, add it to the cart, check out with a name and address, and receive an **Order ID**.

The official sample is from the AWS Containers organisation; the course repository is a fork with the GitOps, Terraform and Argo CD additions.

### The five microservices

| Service | Language | Data store |
|---|---|---|
| **UI** | Java (Spring Boot, built with Maven) | Calls the other services |
| **Catalog** | Go | MySQL |
| **Cart** | Java | DynamoDB |
| **Checkout** | Node.js | Redis (cache) |
| **Orders** | Java | PostgreSQL |

In this lab the databases run **inside the cluster** as public container images (for example DynamoDB Local), so you do not need to create RDS or DynamoDB in AWS.

**Why microservices?** A failure in one service (for example Orders) does not stop the others; each service can be updated and scaled on its own, which is exactly what Kubernetes is good at.

---

## Concepts you need (short version)

### EKS and EKS Auto Mode

A Kubernetes cluster has a **control plane** (API server, scheduler, controller manager, etcd) and **worker nodes** (where your pods run).

- **Standard EKS:** AWS runs the control plane. **You** manage the *data plane*: EC2 nodes, storage, networking add-ons, and load balancers.
- **EKS Auto Mode:** AWS manages the control plane **and** the data plane. It provisions nodes on demand (using Karpenter internally), scales them up and down, and uses immutable, regularly replaced node images (nodes are refreshed roughly every 21 days at most), which reduces security risk.

Consequence you will see in the lab: with Auto Mode a new cluster shows **zero nodes** until you deploy a workload. Nodes appear only when pods need them.

### GitOps

**GitOps** means **Git is the single source of truth**. What is written in Git is exactly what must run in the cluster. If Git says 3 replicas, the cluster has 3. Nobody changes production by hand. Argo CD keeps the cluster in sync with Git, and GitHub Actions is the CI part that updates Git after building images.

---

# PHASE 1: Build the infrastructure

## Part 1: Create an admin IAM user and configure the AWS CLI

### Step 1: Sign in and choose a region

1. Open `https://console.aws.amazon.com` and sign in.
2. Top-right corner: click the region name and choose a region, for example **US West (Oregon) `us-west-2`**. Use this **same region everywhere** in this lab. The examples below use `us-west-2`; replace it with yours if different.

### Step 2: Create the IAM user

**Click:**

```text
AWS Console → IAM → Users → Create user
```

1. **User name:** `tws-cli-user`.
2. Leave console access **unchecked**. Click **Next**.
3. Choose **Attach policies directly**, search for **AdministratorAccess**, tick it.
4. Click **Next**, then **Create user**.

### Step 3: Create access keys

**Click:**

```text
IAM → Users → tws-cli-user → Security credentials → Create access key
```

1. Choose **Command Line Interface (CLI)**, tick the confirmation, click **Next**.
2. Click **Create access key**.
3. Copy the **Access key ID** and **Secret access key** (or download the `.csv`). The secret is shown once.

### Step 4: Configure the AWS CLI

Open a terminal (Ubuntu: `Ctrl + Alt + T`; WSL2: open "Ubuntu" from the Start menu; macOS: Terminal). Then run:

```bash
aws configure
```

| Prompt | Value |
|---|---|
| AWS Access Key ID | your access key ID |
| AWS Secret Access Key | your secret key |
| Default region name | `us-west-2` (your chosen region) |
| Default output format | press Enter |

> The course repository README says to use the **root user** credentials. Do not do that. Root keys are dangerous. An IAM user with AdministratorAccess works the same way for this lab and is safer.

### Verify

```bash
aws sts get-caller-identity
aws s3 ls
```

**Expected result:** the first command prints your account ID and an ARN ending in `user/tws-cli-user`. The second lists your buckets (or prints nothing if you have none). Neither should show an error.

---

## Part 2: Install the tools

Skip any tool that already passes its verify command (see Prerequisites).

### Ubuntu/Debian

**Basic packages, unzip and Git:**

```bash
sudo apt-get update
sudo apt-get install -y curl unzip git gnupg software-properties-common apt-transport-https ca-certificates
```

**AWS CLI v2** (x86_64; for ARM use `awscli-exe-linux-aarch64.zip`):

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Terraform** (HashiCorp's official apt repository):

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update
sudo apt-get install -y terraform
```

**kubectl:**

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```

**Helm 3:**

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

(`chmod 700` gives only you permission to read, write and run the script.)

**Docker** (only for the optional manual ECR demo; use Docker's official instructions at `https://docs.docker.com/engine/install/ubuntu/`, or on a personal machine install **Docker Desktop**).

**`watch`** (optional, used to watch pods appear):

```bash
sudo apt-get install -y watch
```

### macOS (Homebrew)

```bash
brew install awscli terraform kubectl helm git watch
brew install --cask docker
```

### Verify

```bash
aws --version
terraform version
kubectl version --client
helm version
git --version
```

**Expected result:** every command prints a version and none says `command not found`. Use `helm version`, not `helm --version`.

### Common Problems

**Problem:** `Permission denied` when running `./get_helm.sh`
**Fix:** `chmod 700 get_helm.sh` first, then run it again.

**Problem:** `terraform: command not found` after install
**Fix:** open a **new** terminal window so your shell reloads its path, then run `terraform version` again.

---

## Part 3: Fork the repository and clone it

### Step 5: Fork the repository (GitHub website)

**Click:**

```text
Browser → https://github.com/LondheShubham153/retail-store-sample-app → Fork (top right) → Create a new fork
```

**Very important:** on the "Create a new fork" page **untick "Copy the `main` branch only"**. The course uses a second branch called `gitops`, and it must be copied into your fork. Then click **Create fork**.

### Verify

**Click:** in **your** fork, open the branch drop-down (it says `main`) → **View all branches**.

**Expected result:** both `main` and `gitops` are listed.

If you already forked with only `main`, add the missing branch later with these commands (after Step 6):

```bash
git remote add upstream https://github.com/LondheShubham153/retail-store-sample-app.git
git fetch upstream gitops
git checkout -b gitops upstream/gitops
git push -u origin gitops
```

### Step 6: Clone your fork

Go to your fork on GitHub, click the green **Code** button, and copy the **HTTPS** URL. Then:

```bash
mkdir -p ~/projects/retail-store
cd ~/projects/retail-store
git clone https://github.com/YOUR_GITHUB_USERNAME/retail-store-sample-app.git
cd retail-store-sample-app
```

Open the folder in **VS Code**: menu **File → Open Folder…** and choose `retail-store-sample-app` (or run `code .` in the terminal). Install the **HashiCorp Terraform** extension (Extensions icon on the left, search "HashiCorp Terraform") for colour highlighting.

> An AI coding assistant (for example Amazon Q Developer or Kiro) is optional. It can help you explain code or draft files, but **you must read and verify everything it produces**. Nothing in this lab requires it.

### Verify

```bash
git branch -a
ls
```

**Expected result:** `git branch -a` lists `main` (and `remotes/origin/gitops`). `ls` shows `argocd`, `docs`, `src`, `terraform`, `README.md`, `BRANCHING_STRATEGY.md`.

---

## Part 4: Understand the project structure

```text
retail-store-sample-app/
├── argocd/                  # Argo CD Project and Application YAML files
│   ├── projects/
│   └── applications/
├── docs/                    # Documentation images (not important)
├── src/                     # All microservices (most important folder)
│   ├── ui/                  # Dockerfile, source, chart/ (Helm chart)
│   ├── catalog/
│   ├── cart/
│   ├── checkout/
│   ├── orders/
│   └── app/chart/           # "Umbrella" chart used on the main branch
├── terraform/               # All infrastructure code
│   ├── main.tf
│   ├── variables.tf
│   ├── versions.tf
│   ├── security.tf
│   ├── argocd.tf
│   ├── locals.tf
│   └── outputs.tf
├── BRANCHING_STRATEGY.md
└── README.md
```

Folder layout can change slightly between repository versions. Run `ls argocd src terraform` to see your exact contents.

### The Terraform files

| File | Purpose |
|---|---|
| `main.tf` | Creates the VPC and the EKS cluster using ready-made modules |
| `variables.tf` | Inputs: cluster name, Kubernetes version, VPC CIDR, Argo CD namespace |
| `versions.tf` | Providers: AWS, Helm, Kubernetes, time, random |
| `security.tf` | Security-group rules (NodePort range 30000-32767, HTTP and HTTPS) |
| `argocd.tf` | Installs Argo CD and applies the Argo CD projects and applications |
| `locals.tf` | Local computed values |
| `outputs.tf` | Values printed after `terraform apply` (URLs, names, commands) |

### The two branches

| | `main` branch | `gitops` branch |
|---|---|---|
| Images | **Public** images (stable version) | **Your private** ECR images |
| Deployment | One "umbrella" Argo CD app (`retail-store-app`) | One Argo CD app per service |
| CI/CD | None | GitHub Actions pipeline |
| Use | Learning, demos | GitOps and production-style flow |

You start with `main` (Phase 1), then switch to `gitops` (Phase 3).

---

## Part 5: Read the Terraform code

Open each file in VS Code and read it. You do not have to change anything except the Kubernetes version (Part 6).

**`main.tf` builds two things using prebuilt modules from the Terraform Registry:**

1. **VPC module** (`terraform-aws-modules/vpc/aws`): a private network with public and private subnets, an internet gateway and a NAT gateway. The VPC name and address range come from variables.
2. **EKS module** (`terraform-aws-modules/eks/aws`): the cluster itself. Key settings:
   - Cluster name and Kubernetes version from variables.
   - Public **and** private API endpoint access enabled.
   - `enable_cluster_creator_admin_permissions = true`: gives the identity that runs Terraform admin access to the cluster. This is why `kubectl` works right away.
   - **Auto Mode** is enabled by this block:

     ```hcl
     cluster_compute_config = {
       enabled = true
     }
     ```

**`argocd.tf`** installs Argo CD with Helm and applies the Argo CD project and application YAML from the `argocd/` folder. It also installs **NGINX Ingress** and **cert-manager** (as Helm releases).

### Why the apply is done in stages

Terraform has to build things in order, because each layer needs the previous one:

1. VPC first.
2. EKS cluster needs the VPC.
3. Argo CD, NGINX and cert-manager need a running cluster (the Helm and Kubernetes providers must connect to it).
4. The applications need Argo CD.

That is why you first apply only the VPC and the cluster with `-target`, then run a full apply.

---

## Part 6: Build the infrastructure

### Step 7: Set the Kubernetes version

Find the version variable:

```bash
cd terraform
grep -n -i "version" variables.tf
```

**Expected result:** a variable such as `kubernetes_version` or `cluster_version` with a `default = "1.xx"` line. Open `variables.tf` in VS Code and change that default to a version that is in **standard support** on EKS, for example:

```hcl
default = "1.35"
```

Check current standard-support versions in the EKS console (**EKS → Create cluster → Kubernetes version** drop-down) or in the AWS documentation page "Amazon EKS Kubernetes versions".

You can also change the cluster name, the environment (`dev`, `production`) and the VPC range in the same file. The examples below keep the defaults. Save the file (`Ctrl + S`).

### Step 8: Initialise Terraform

Make sure you are in the `terraform` folder:

```bash
pwd
terraform init
```

`terraform init` downloads the providers and modules from the internet.

**Expected result:** the last lines say **Terraform has been successfully initialized!**

### Step 9: Create the VPC

```bash
terraform apply -target=module.vpc
```

- `module.vpc` is the name of the VPC module in `main.tf`.
- Terraform prints a plan (roughly 20 to 30 resources). Type `yes` and press Enter.
- It takes a few minutes.

#### Verify

Copy the VPC ID from the output. **Click:**

```text
AWS Console → VPC → Your VPCs → (paste the VPC ID in the search box)
```

**Expected result:** a VPC named like `retail-store-...` exists. The region selector (top right) must match your region.

### Step 10: Create the EKS cluster

Find the EKS module name in `main.tf`:

```bash
grep -n 'module "' main.tf
```

**Expected result:** two module blocks, for example `module "vpc"` and `module "retail_app_eks"`. Use the exact EKS module name you see:

```bash
terraform apply -target=module.retail_app_eks
```

Type `yes` when asked. This creates roughly 40 resources and takes about **10 minutes**.

#### Verify

**Click:**

```text
AWS Console → EKS → Clusters
```

**Expected result:** a cluster named `retail-store-...` with status **Active** and Kubernetes version as you set it.

### Step 11: Connect `kubectl` to the new cluster

Get the exact cluster name. It has a random suffix, for example `retail-store-dev-xxxx`:

```bash
aws eks list-clusters --region us-west-2
```

Then update your kubeconfig with **that** name and **your** region:

```bash
aws eks update-kubeconfig --region us-west-2 --name YOUR_CLUSTER_NAME
```

#### Verify

```bash
kubectl config current-context
kubectl get nodes
```

**Expected result:** the context contains your cluster name. `kubectl get nodes` prints **`No resources found`**. This is correct for Auto Mode: no nodes exist until something needs to run.

### Common Problems

**Problem:** `Error from server (Forbidden): nodes is forbidden ... cannot list resource "nodes" in API group "" at the cluster scope`
**Possible reason:** your IAM identity has no *access entry* on the cluster (for example you created the cluster in the console with another identity).
**Fix (console):**

```text
AWS Console → EKS → Clusters → your cluster → Access tab → Create access entry
```

1. **IAM principal:** choose your IAM user (`tws-cli-user`). Click **Next**.
2. **Access policy:** add `AmazonEKSClusterAdminPolicy`, scope **Cluster**. Click **Next**, then **Create**.
3. Run `kubectl get nodes` again.

(For the Terraform-created cluster this is already done through `enable_cluster_creator_admin_permissions = true`.)

**Problem:** `Unable to connect to the server` or `no such host`
**Fix:** check the region in the command matches the cluster's region and that the cluster status is Active.

### Step 12: Watch Auto Mode create a node on demand

```bash
kubectl create namespace demo-nginx
kubectl run nginx --image=nginx -n demo-nginx
watch kubectl get pods -n demo-nginx
```

The pod starts in `Pending`. After about **30 to 60 seconds** it becomes `Running`. Press `Ctrl + C` to leave `watch`.

#### Verify

```bash
kubectl get nodes
```

**Expected result:** now **one node** exists. Auto Mode created it because a pod needed room to run.

Clean up the demo:

```bash
kubectl delete namespace demo-nginx
```

(Auto Mode removes the empty node again after a while.)

---

## Part 7: Deploy the platform and the application

### Step 13: Full Terraform apply

```bash
terraform apply
```

Type `yes`. Terraform refreshes what exists and creates the rest (about 15 to 20 more resources): the **NGINX Ingress Controller**, **cert-manager**, **Argo CD**, the Argo CD **project** and **applications**. A **load balancer** is created automatically for the ingress controller.

The monitoring stack (Prometheus and Grafana) is available as a commented block in the Terraform code. Leave it commented for now (see Homework at the end).

### Step 14: Read the outputs

```bash
terraform output
```

You will see, among other values: the Argo CD namespace, a port-forward command for Argo CD, the cluster endpoint and name, the OIDC provider, security group IDs, and the ingress load balancer address. The Argo CD admin password is marked *sensitive*; to show it:

```bash
terraform output -raw argocd_admin_password 2>/dev/null || terraform output
```

(The output name can differ. Use the name shown by `terraform output`.)

### Step 15: Check that everything is running

```bash
kubectl get pods -A
```

**Expected result:** pods in these namespaces, all eventually `Running` or `Completed` (allow 5 to 10 minutes, as nodes are created on demand):

- `argocd` (Argo CD components)
- `cert-manager`
- `ingress-nginx` (the controller)
- `retail-store` (cart, catalog, checkout, orders, UI, plus their databases)

```bash
kubectl get pods -n retail-store
kubectl get ingress -n retail-store
```

### Common Problems

**Problem:** pods stay `Pending` for more than 5 minutes
**Fix:** run `kubectl describe pod POD_NAME -n NAMESPACE` and read the **Events** at the bottom. Often it is just a node still starting. Check `kubectl get nodes`.

**Problem:** `terraform apply` fails with a provider connection error (Helm or Kubernetes)
**Possible reason:** the cluster was not created before the full apply.
**Fix:** run Step 10 first (targeted apply of the EKS module), then run `terraform apply` again.

**Problem:** `Error: creating EKS Cluster ... InvalidParameterException: unsupported Kubernetes version`
**Fix:** choose a version listed in the EKS console (Step 7).

---

## Part 8: Open the application and Argo CD

### Step 16: Open the shop

Get the load balancer address:

```bash
kubectl get svc -n ingress-nginx
```

**Expected result:** a service `ingress-nginx-controller` of type `LoadBalancer` with an **EXTERNAL-IP** column showing a long DNS name ending in `.elb.us-west-2.amazonaws.com` (yours will match your region). If it says `<pending>`, wait one or two minutes and run the command again. The same address is in `terraform output`.

Copy the address into your browser, ideally in a private/incognito window, using **`http://`** (not https):

```text
http://YOUR_LOAD_BALANCER_ADDRESS
```

- If the browser warns that the site is **Not secure**, that is expected. There is no domain or certificate yet. Continue to the site.
- **Expected result:** the retail store loads. Add a product to the cart, check out, enter any name and address, and place the order. You receive an **Order ID** and an "Order placed" message. The application is now running on EKS.

### Step 17: Open the Argo CD web interface

Start a port-forward (keep this terminal open, or open a second terminal):

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get the admin password in another terminal:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

In your browser open:

```text
https://localhost:8080
```

Accept the certificate warning (the certificate is self-signed). Sign in with:

- **Username:** `admin`
- **Password:** the value printed above

**Expected result:** the Argo CD dashboard lists the applications and they show **Synced** and **Healthy** (green). Click an application, then **Details** to see its source repository, branch and path.

### How Argo CD knows what to deploy

Argo CD reads YAML from the `argocd/` folder of the repo (applied by Terraform):

- **Project** (`retail-store-project`): which Git repositories may be used as sources and which clusters/namespaces are allowed as destinations. In the UI: **Settings → Projects**.
- **Applications**: one per service (or one umbrella app on `main`). Each defines the **repository URL**, **branch** (`targetRevision`), and the **path** of a Helm chart (for example `src/ui/chart`), plus the destination namespace.

Argo CD then reads the chart's `values.yaml` and `templates/` and applies the result to the cluster.

> **Phase 1 is complete.** You have an EKS Auto Mode cluster, NGINX Ingress, cert-manager, Argo CD and a running application, all created by Terraform. Currently the apps use **public images** from the **`main`** branch.

---

# PHASE 2: Understand the application and Kubernetes objects

A DevOps engineer does not only run tools. You must be able to read the code and understand what is being built: the language, the build tool, the database, and how services talk to each other.

## Part 9: Read each microservice

For every service, open three things: the **Dockerfile**, the Helm chart **`values.yaml`**, and the **README**. Use VS Code or the terminal, for example:

```bash
cd ~/projects/retail-store/retail-store-sample-app
ls src/cart
cat src/cart/Dockerfile
cat src/cart/chart/values.yaml
```

| Service | How to recognise the language | What its Dockerfile does | Data store in `values.yaml` |
|---|---|---|---|
| **Cart** (Java) | `pom.xml`, `.java` files | Multi-stage build: uses Maven (`mvn clean install`) to create a **JAR**, then runs the JAR as a **non-root user** on a Java 21 base image | A public **DynamoDB Local** image (or real DynamoDB endpoints if you set them) |
| **Catalog** (Go) | `go.mod`, `main.go` | Downloads Go modules (`go mod download`), compiles (`go build`), then runs the binary | A public **MySQL** image with a pinned version, a service port and storage settings |
| **Checkout** (Node.js) | `package.json` | Installs Node dependencies and runs the app | A public **Redis** image |
| **Orders** (Java) | `pom.xml` | Like Cart: Maven build, JAR, non-root user | A public **PostgreSQL** image |
| **UI** (Java) | `pom.xml`, Spring Boot, `templates/` HTML | Maven build, runs the Spring Boot app | Calls the other services; no database |

The lesson: writing a Dockerfile is easy when you know which language the developers used. Developers provide the build commands; you package them.

Contents can change between versions of the repository. Always read your own copy.

## Part 10: Service-to-service communication and exposure

**How services find each other.** Look at the umbrella chart values:

```bash
cat src/app/chart/values.yaml
```

It contains the **endpoints** (Kubernetes service names) of cart, catalog, checkout, orders and UI, so the UI can call the others by name inside the cluster.

**How services are exposed.** Every service listens on **port 80** and is of type `ClusterIP` (reachable only inside the cluster). The **NGINX Ingress Controller** is the only door from the outside.

```bash
kubectl get svc -n retail-store
kubectl get ingress -n retail-store
kubectl describe ingress -n retail-store
```

**Expected result:** services of type `ClusterIP` on port 80; one ingress that routes the host/path rules to the UI service. The UI chart's ingress settings (`src/ui/chart/values.yaml` and its ingress template) hold the host and path rules. If you use a domain later, you set the host there.

## Part 11: Helm and Argo CD in one picture

- **Helm** is the package manager for Kubernetes. Instead of writing separate Deployment, Service and Ingress files, each service chart has ready templates in `templates/`. You only change **values** in `values.yaml` (image, tag, resources, replicas, autoscaling).
- Changing the `image` values changes the Deployment image.
- **Argo CD** reads the chart from Git and applies it to EKS.

Try it, read-only, to see the manifest that Helm builds:

```bash
helm template ui src/ui/chart | head -60
```

**Expected result:** rendered Kubernetes YAML (Deployment, Service, and so on). Nothing is deployed.

> **Phase 2 is complete.**

---

# PHASE 3: GitOps (CI/CD with GitHub Actions)

## Part 12: The problem with manual image pushes

Right now the cluster runs **public** images. To run **your own** changed code you must build an image, push it to a private registry, and update the image in the Helm values. Doing this by hand is slow, error-prone, and not recorded in Git. It breaks the GitOps rule (Git is the source of truth). The next parts automate it.

### (Optional) Step 18: Do it manually once with Amazon ECR to understand it

Requires Docker running on your machine.

**Click:**

```text
AWS Console → ECR → Private registry → Repositories → Create repository
```

1. **Visibility:** Private. **Repository name:** `demo-private/ui`.
2. Leave **Tag immutability** as **Mutable** and click **Create**.
3. Open the repository and click **View push commands**. You get four commands: **login**, **build**, **tag**, **push**.
4. Run the **login** command in your terminal. It looks like:

   ```bash
   aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin YOUR_AWS_ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com
   ```

   **Expected result:** `Login Succeeded`.
5. Go to the service folder and run the **build**, **tag** and **push** commands exactly as shown in the console:

   ```bash
   cd src/ui
   ```

6. Refresh the repository page. **Expected result:** an image with the tag `latest` appears.

When you are done, delete this demo repository (ECR → the repository → **Delete**). The rest of the lab automates all of these steps.

---

## Part 13: Understand the pipeline

**CI (Continuous Integration)** after each code change:

1. Check out the code.
2. Detect **which** services changed.
3. `docker build` each changed service.
4. Tag with the short commit ID.
5. Push to **Amazon ECR**.
6. Update the image repository and tag in that service's Helm `values.yaml` and **commit it back to Git**.

**CD (Continuous Delivery)**: **Argo CD** sees the new commit and deploys the change to EKS. Full chain:

```text
git push → GitHub Actions → ECR → values.yaml updated in Git → Argo CD → EKS
```

Important design points:

- The build runs on a **GitHub-hosted runner** (a temporary Ubuntu machine), not on your laptop.
- The workflow runs only for pushes to **`gitops`** that change files under **`src/`**. A change to a README at the repo root must not trigger a build.
- A **matrix** builds all changed services in **parallel**.
- The commit that updates `values.yaml` is made with the built-in `GITHUB_TOKEN`. GitHub does **not** start workflows for commits made with that token, so the pipeline does not loop.

---

## Part 14: Create a limited IAM user for GitHub Actions

Do **not** give GitHub the admin user. Use a separate user that can only work with ECR, following the least-privilege principle.

### Step 19: Create the IAM policy

**Click:**

```text
AWS Console → IAM → Policies → Create policy → JSON tab
```

Delete everything in the editor and paste the policy below. Replace `YOUR_AWS_REGION` (for example `us-west-2`) and `YOUR_AWS_ACCOUNT_ID` (12 digits; find it by clicking your account name at the top right of the console).

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EcrLogin",
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Sid": "EcrCreateAndPushRetailStoreRepos",
      "Effect": "Allow",
      "Action": [
        "ecr:CreateRepository",
        "ecr:DescribeRepositories",
        "ecr:ListTagsForResource",
        "ecr:DescribeImages",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "arn:aws:ecr:YOUR_AWS_REGION:YOUR_AWS_ACCOUNT_ID:repository/retail-store*"
    }
  ]
}
```

**Purpose:** `GetAuthorizationToken` lets the pipeline log in to ECR (it cannot be limited to one repository). The second statement allows creating the repositories and pushing images, but **only** for repositories whose names start with `retail-store`. It cannot create clusters or touch any other AWS service.

Click **Next**, name the policy `github-actions-ecr-policy`, click **Create policy**.

### Step 20: Create the user and its access key

**Click:**

```text
IAM → Users → Create user
```

1. **User name:** `github-actions-ecr-user`. No console access. **Next**.
2. **Attach policies directly** → search `github-actions-ecr-policy` → tick it → **Next** → **Create user**.
3. Open the user → **Security credentials** → **Create access key** → **Command Line Interface (CLI)** → confirm → **Create access key**.
4. Copy the **Access key ID** and **Secret access key**.

### Common Problems

**Problem (seen later in the pipeline):** `User ... is not authorized to perform: ecr:CreateRepository` or `ecr:ListTagsForResource`
**Fix:** open the policy (IAM → Policies → `github-actions-ecr-policy` → **Edit**), make sure that action is in the list above, and save. Then re-run the failed job (Part 18).

---

## Part 15: Add the GitHub repository secrets

**Click:**

```text
GitHub → your fork → Settings → Secrets and variables → Actions → New repository secret
```

Create these four secrets (one at a time: type the **Name**, paste the **Secret**, click **Add secret**):

| Secret name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key ID of `github-actions-ecr-user` |
| `AWS_SECRET_ACCESS_KEY` | Secret access key of `github-actions-ecr-user` |
| `AWS_REGION` | Your region, for example `us-west-2` (the same one as your cluster) |
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID |

Repository secrets are encrypted, are not shown again after saving, and are not printed in workflow logs. Use your real values, not example values from tutorials or AI tools.

### Enable Actions in your fork

GitHub disables workflows in new forks. **Click:**

```text
GitHub → your fork → Actions tab → "I understand my workflows, go ahead and enable them"
```

Then **click:**

```text
Settings → Actions → General → Workflow permissions → Read and write permissions → Save
```

The pipeline needs write permission to commit the updated `values.yaml` back to the repository.

---

## Part 16: The workflow file

Switch to the `gitops` branch:

```bash
cd ~/projects/retail-store/retail-store-sample-app
git status
git checkout gitops
git pull origin gitops
ls -a .github/workflows 2>/dev/null
```

The `gitops` branch of the course repository may already contain a workflow file in `.github/workflows/`. **Keep exactly one workflow file.** Two workflows would build everything twice.

- **Option A:** if a working file such as `deploy.yml` is already there, read it, and continue to Part 17.
- **Option B (recommended so you understand it):** delete the existing one and create your own with the complete file below.

For Option B:

```bash
git rm -f .github/workflows/*.yml .github/workflows/*.yaml 2>/dev/null
mkdir -p .github/workflows
```

## Create the File

**File:**

```text
.github/workflows/deploy.yml
```

**Content:**

```yaml
name: Build, push to ECR and update Helm values

on:
  push:
    branches: [gitops]
    paths:
      - "src/**"
  workflow_dispatch:        # lets you start the workflow manually (builds all services)

permissions:
  contents: write           # needed to commit the updated values.yaml

concurrency:
  group: gitops-deploy      # never run two deployments at the same time
  cancel-in-progress: false

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}
  AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
  ECR_PREFIX: retail-store

jobs:
  # ---------------------------------------------------------------
  # Job 1: find out which services changed
  # ---------------------------------------------------------------
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.set-services.outputs.services }}
      has_changes: ${{ steps.set-services.outputs.has_changes }}
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Detect changed services
        id: filter
        if: github.event_name != 'workflow_dispatch'
        uses: dorny/paths-filter@v3
        with:
          filters: |
            ui:
              - 'src/ui/**'
            catalog:
              - 'src/catalog/**'
            cart:
              - 'src/cart/**'
            checkout:
              - 'src/checkout/**'
            orders:
              - 'src/orders/**'

      - name: Build the list of services
        id: set-services
        run: |
          if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            SERVICES='["ui","catalog","cart","checkout","orders"]'
          else
            SERVICES='${{ steps.filter.outputs.changes }}'
          fi
          echo "Services to build: ${SERVICES}"
          echo "services=${SERVICES}" >> "$GITHUB_OUTPUT"
          if [ "${SERVICES}" = "[]" ]; then
            echo "has_changes=false" >> "$GITHUB_OUTPUT"
          else
            echo "has_changes=true" >> "$GITHUB_OUTPUT"
          fi

  # ---------------------------------------------------------------
  # Job 2: build and push one image per changed service (in parallel)
  # ---------------------------------------------------------------
  build-and-push:
    needs: detect-changes
    if: needs.detect-changes.outputs.has_changes == 'true'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service: ${{ fromJSON(needs.detect-changes.outputs.services) }}
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Create the ECR repository if it does not exist
        run: |
          REPO="${ECR_PREFIX}/${{ matrix.service }}"
          aws ecr describe-repositories --repository-names "${REPO}" > /dev/null 2>&1 \
            || aws ecr create-repository --repository-name "${REPO}"

      - name: Build and push the image
        env:
          REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        run: |
          TAG="${GITHUB_SHA::7}"
          IMAGE="${REGISTRY}/${ECR_PREFIX}/${{ matrix.service }}"
          docker build -t "${IMAGE}:${TAG}" -t "${IMAGE}:latest" "src/${{ matrix.service }}"
          docker push --all-tags "${IMAGE}"

  # ---------------------------------------------------------------
  # Job 3: write the new image tags into the Helm values and commit
  # ---------------------------------------------------------------
  update-helm-values:
    needs: [detect-changes, build-and-push]
    if: needs.detect-changes.outputs.has_changes == 'true' && needs.build-and-push.result == 'success'
    runs-on: ubuntu-latest
    steps:
      - name: Check out the gitops branch
        uses: actions/checkout@v4
        with:
          ref: ${{ github.ref_name }}

      - name: Update image repository and tag in each chart
        run: |
          TAG="${GITHUB_SHA::7}"
          REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
          for SERVICE in $(echo '${{ needs.detect-changes.outputs.services }}' | jq -r '.[]'); do
            FILE="src/${SERVICE}/chart/values.yaml"
            echo "Updating ${FILE}"
            yq -i ".image.repository = \"${REGISTRY}/${ECR_PREFIX}/${SERVICE}\" | .image.tag = \"${TAG}\"" "${FILE}"
          done

      - name: Commit and push the change
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add src/*/chart/values.yaml
          if git diff --cached --quiet; then
            echo "No value changes to commit"
            exit 0
          fi
          git commit -m "ci: update image tags to ${GITHUB_SHA::7} [skip ci]"
          git pull --rebase origin "${{ github.ref_name }}"
          git push origin "HEAD:${{ github.ref_name }}"

  # ---------------------------------------------------------------
  # Job 4: print a short summary
  # ---------------------------------------------------------------
  summary:
    needs: [detect-changes, build-and-push, update-helm-values]
    if: always() && needs.detect-changes.outputs.has_changes == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Write deployment summary
        run: |
          {
            echo "## Deployment summary"
            echo "- Commit: \`${GITHUB_SHA::7}\`"
            echo "- Branch: \`${{ github.ref_name }}\`"
            echo "- Triggered by: \`${{ github.actor }}\`"
            echo "- Services built: \`${{ needs.detect-changes.outputs.services }}\`"
            echo "- Build and push result: \`${{ needs.build-and-push.result }}\`"
            echo "- Helm values update result: \`${{ needs.update-helm-values.result }}\`"
            echo "- Argo CD will detect the new commit and sync the applications."
          } >> "$GITHUB_STEP_SUMMARY"
```

**Purpose:** it is the whole CI pipeline. The four jobs are: find changed services, build and push images (parallel matrix), write new image tags into `src/<service>/chart/values.yaml` and commit them, and print a summary.

**How it fits with the rest:**

| Item in the workflow | Must match |
|---|---|
| `branches: [gitops]` | The branch Argo CD watches (Part 17) |
| `src/<service>/` with a `Dockerfile` | The Docker build context for each service |
| `src/<service>/chart/values.yaml` with `image.repository` and `image.tag` | The Helm chart keys. **Open each `values.yaml` and confirm these keys exist.** If your chart uses different key names, change the `yq` line |
| Secrets `AWS_*` | The four secrets from Part 15 |
| `${ECR_PREFIX}/<service>` = `retail-store/<service>` | The IAM policy resource `repository/retail-store*` |

> Image repository names in this lab are `retail-store/cart`, `retail-store/catalog`, and so on. The course repository's own workflow may name them `retail-store-cart` and so on. Both match the IAM policy above; just be consistent.

> **Reviewing is your job.** If an AI tool generates a workflow for you, check the trigger, the branch, the secrets, the AWS region and account ID, and the chart paths before you commit. Do not trust it blindly.

### Verify the file

```bash
ls .github/workflows
```

**Expected result:** exactly one file, `deploy.yml`.

---

## Part 17: Point Argo CD at your fork and the `gitops` branch

Argo CD must watch **your fork** (where the pipeline commits), on the **`gitops`** branch, with one application per service.

### Step 21: Check the repository URL in the Argo CD files

```bash
grep -rn "repoURL\|targetRevision" argocd/
grep -rn -i "github.com" terraform/*.tf
```

**Expected result:** lines showing `repoURL:` values and `targetRevision:` values. If `repoURL` points to `https://github.com/LondheShubham153/retail-store-sample-app.git` (the original repository), the pipeline's commits in **your** fork would never be seen. Change every such URL to your fork:

```text
https://github.com/YOUR_GITHUB_USERNAME/retail-store-sample-app.git
```

Also check that each application file under `argocd/applications/` uses `targetRevision: gitops` on this branch, and that the project file lists your fork under its allowed source repositories.

> Your fork must be **public** so Argo CD can read it without credentials. (For a private repository you would add repository credentials in Argo CD under **Settings → Repositories**.)

### Step 22: Commit the change

```bash
git status
git add argocd/
git commit -m "Point Argo CD applications to my fork on the gitops branch"
git push origin gitops
```

### Step 23: Switch from the umbrella app to the per-service apps

Using umbrella and per-service apps at the same time causes a "SharedResourceWarning" because two apps would own the same resources. Remove the umbrella app first, then apply the per-service apps. Make sure `kubectl` still points to your cluster (`kubectl get nodes`).

```bash
kubectl delete application retail-store-app -n argocd
kubectl apply -f argocd/applications/ -n argocd
```

If the first command says the application is not found, that is fine; continue. If `kubectl apply` complains about the umbrella file, apply only the per-service files listed by `ls argocd/applications/`.

#### Verify

```bash
kubectl get applications -n argocd
```

**Expected result:** five applications (`retail-store-ui`, `-catalog`, `-cart`, `-checkout`, `-orders`), each with `gitops` as the target revision. In the Argo CD UI (Step 17) click one → **Details** and check **Target revision: gitops**. They may show **Progressing** for a few minutes while pods restart. Wait until they are **Healthy** and **Synced**.

---

## Part 18: Run the pipeline for the first time

### Step 24: Make a small change in every service

Adding a line to each service README is enough. These files are under `src/`, so they trigger the workflow:

```bash
for s in cart catalog checkout orders ui; do echo "test commit" >> src/$s/README.md; done
git status
```

**Expected result:** five modified `README.md` files.

### Step 25: Commit and push

```bash
git add src
git commit -m "Test commit for CI pipeline"
git push origin gitops
```

If Git asks for a username and password, use your GitHub username and a **personal access token** (GitHub → Settings → Developer settings → Personal access tokens) instead of a password.

### Step 26: Watch the pipeline (GitHub website)

**Click:**

```text
GitHub → your fork → Actions tab → click the newest workflow run
```

**Expected result:** four jobs. **detect-changes** finishes first. **build-and-push** shows five parallel matrix jobs (cart, catalog, checkout, orders, ui). Then **update-helm-values** and **summary** run. All jobs become green. The Java builds take several minutes.

### Common Problems

**Problem:** The Actions tab shows no workflow run
**Possible reasons and fixes:**
- Workflows are not enabled in the fork → Part 15 (Enable Actions).
- You pushed to `main`, not `gitops` → check `git branch`.
- You changed only files outside `src/` → the path filter ignored it.
- The file is not in `.github/workflows/` on the **`gitops`** branch.

**Problem:** `User: arn:aws:iam::...:user/github-actions-ecr-user is not authorized to perform: ecr:CreateRepository` (or `ecr:ListTagsForResource`, `ecr:DescribeRepositories`)
**Possible reason:** the IAM policy is missing an action.
**Fix:** add the missing action in IAM → Policies → `github-actions-ecr-policy` → **Edit** (Step 19), save. Then open the failed run in GitHub and click **Re-run failed jobs**. No new commit is needed.

**Problem:** `Could not load credentials from any providers` or `The security token included in the request is invalid`
**Fix:** re-check the secret names (they are case-sensitive) and values in Part 15.

**Problem:** `update-helm-values` fails with `remote: Permission to ... denied to github-actions[bot]` (HTTP 403)
**Fix:** Settings → Actions → General → **Workflow permissions → Read and write permissions**.

**Problem:** `git pull --rebase` or push conflict in the last job
**Fix:** re-run the failed job. The `concurrency` setting normally prevents this.

**Problem:** `yq` changes nothing / the values do not update
**Fix:** open `src/ui/chart/values.yaml` and check that an `image:` section with `repository:` and `tag:` exists; adjust the `yq` expression if the keys differ.

### Step 27: Verify the result end to end

1. **Job summary:** open the run → **summary** job (or the run's Summary page). It lists the commit, branch, who triggered it, and the services built.
2. **ECR:**

   ```text
   AWS Console → ECR → Private registry → Repositories
   ```

   **Expected result:** repositories `retail-store/cart`, `catalog`, `checkout`, `orders`, `ui`, each with an image tagged with the short commit ID and `latest`.
3. **Git:** in your fork, branch `gitops`, open **Commits**. **Expected result:** a new commit from `github-actions[bot]` titled `ci: update image tags to ...`. Open `src/ui/chart/values.yaml` and check that `image.repository` now points to your ECR URL.
4. **Pull it locally:**

   ```bash
   git pull origin gitops
   ```

5. **Argo CD and pods:**

   ```bash
   kubectl get applications -n argocd
   kubectl get pods -n retail-store
   kubectl get pods -n retail-store -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
   ```

   **Expected result:** applications **Synced/Healthy**; service pods running images from `YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/retail-store/...`.

**Problem:** pods show `ImagePullBackOff` or `ErrImagePull`
**Fix:** run `kubectl describe pod POD_NAME -n retail-store` and read the Events. Common causes: the pipeline has not finished yet (image does not exist yet), a wrong repository name or tag in `values.yaml`, or the repository is in a different region from the cluster.

### Step 28: Open the shop again

```bash
kubectl get svc -n ingress-nginx
```

Open `http://EXTERNAL-IP-ADDRESS` in your browser (continue past the "not secure" warning). **Expected result:** the shop works, now running your own private images.

---

## Part 19: Prove GitOps with a UI change

1. Make sure you are on the `gitops` branch (`git branch`). Changes on `main` do nothing here.
2. Open this file in VS Code:

   ```text
   src/ui/src/main/resources/templates/home.html
   ```

3. Change a visible piece of text, for example add `Hello from my GitOps pipeline!` near the page heading. Save the file.
4. Commit and push:

   ```bash
   git add src/ui
   git commit -m "Change UI home page text"
   git push origin gitops
   ```

5. Watch **GitHub → Actions**. **Expected result:** only **one** matrix job (`ui`) runs, because only the UI folder changed. It builds, pushes, updates `src/ui/chart/values.yaml`, and commits.
6. Watch Argo CD (`https://localhost:8080`). **Expected result:** the `retail-store-ui` application becomes **OutOfSync/Progressing**, then **Healthy/Synced** within a few minutes. You touched nothing by hand.
7. Refresh the shop in your browser. **Expected result:** your new text appears.

> **GitOps is working.** Change code in Git → pipeline builds and updates values → Argo CD deploys.

**Rollback:** to undo a bad deployment, revert the pipeline's commit in Git:

```bash
git revert HEAD
git push origin gitops
```

Argo CD then deploys the previous version. (If the last commit was a code commit, revert that instead and let the pipeline run again.)

---

# Extra topics

## Homework: enable monitoring (Prometheus and Grafana)

1. In the Terraform code (for example `terraform/argocd.tf` or the add-ons block in `main.tf`), find the **commented** block for the monitoring stack (search for `prometheus`).
2. Remove the comment characters (`#`, or `/* ... */`).
3. Run:

   ```bash
   cd terraform
   terraform apply
   ```

4. Check: `kubectl get pods -A | grep -i -E "prometheus|grafana"`.

**Expected result:** monitoring pods are `Running`. Monitoring pods need additional nodes, so Auto Mode creates them, which adds cost.

## Enable HTTPS with a real domain

The load-balancer address shows "Not secure" because there is no certificate.

1. Buy or use a domain (Route 53, GoDaddy, and so on).
2. Open `src/ui/chart/values.yaml` and find the **ingress** section (NGINX). Set the **host** to your domain and set the **ssl-redirect** option to `true`.
3. In your DNS provider, create a **CNAME** record from your domain (or subdomain) to the load-balancer address.
4. In the cert-manager configuration (the ClusterIssuer), set **your own e-mail address**.
5. Commit and push to `gitops`. cert-manager requests a certificate and the site opens on `https://`.

Ports used: **80** (also needed for certificate validation) and **443**.

## Upgrading the cluster

EKS Auto Mode updates **nodes** automatically. It does **not** upgrade the Kubernetes version of the cluster (control plane) for you. Do that yourself in the console (**EKS → your cluster → Upgrade version**) or by changing the version in Terraform. Read the release notes first.

---

# Cleanup (do not skip)

Everything below removes billable resources. Do the steps **in order**.

### Step 29: Stop deployments first

Optional but tidy: delete the Argo CD applications so nothing recreates resources.

```bash
kubectl delete applications --all -n argocd
```

### Step 30: Destroy the Terraform infrastructure

```bash
cd ~/projects/retail-store/retail-store-sample-app/terraform
terraform destroy
```

Type `yes`. Terraform removes only what it created: the applications and Helm releases, Argo CD, the EKS cluster, the VPC, and so on. This takes about 10 to 20 minutes.

### Step 31: Delete what Terraform did not create

1. **ECR repositories:**

   ```text
   AWS Console → ECR → Repositories → select each retail-store/* repository → Delete
   ```

   (Type `delete` to confirm. Repositories that contain images must be deleted with their images.)
2. **IAM:** delete both access keys and the users `github-actions-ecr-user` and (if you no longer need it) `tws-cli-user`, plus the policy `github-actions-ecr-policy`.
3. **GitHub:** delete the four repository secrets (Settings → Secrets and variables → Actions).
4. Any cluster or repository you created by hand in the console.

### Verify

Check that nothing billable is left in **your region**:

```bash
aws eks list-clusters --region us-west-2
aws elbv2 describe-load-balancers --region us-west-2 --query "LoadBalancers[].LoadBalancerName"
aws ec2 describe-nat-gateways --region us-west-2 --filter Name=state,Values=available --query "NatGateways[].NatGatewayId"
aws ec2 describe-volumes --region us-west-2 --query "Volumes[?State=='available'].VolumeId"
```

**Expected result:** empty lists (`[]`). Also glance at **EC2 → Elastic IPs**, **VPC → Your VPCs** and **CloudFormation → Stacks** in the console. If something is left, delete it manually (leftover load balancers and NAT gateways are the usual causes of surprise bills).

**Problem:** `terraform destroy` hangs or fails on the VPC ("DependencyViolation")
**Possible reason:** a load balancer, network interface, or security group created by Kubernetes still exists.
**Fix:** delete the leftover load balancer in **EC2 → Load Balancers**, wait a few minutes, then run `terraform destroy` again.

---

## Final project state

Files you created or changed in your fork (everything else came with the repository):

```text
retail-store-sample-app/
├── .github/
│   └── workflows/
│       └── deploy.yml            # created by you (gitops branch)
├── argocd/
│   ├── projects/                 # repoURL changed to your fork
│   └── applications/             # repoURL changed to your fork, targetRevision: gitops
├── src/
│   ├── ui/
│   │   ├── README.md             # test line added
│   │   ├── chart/values.yaml     # image tag updated by the pipeline
│   │   └── src/main/resources/templates/home.html   # edited in Part 19
│   ├── cart/     (README.md + chart/values.yaml)
│   ├── catalog/  (README.md + chart/values.yaml)
│   ├── checkout/ (README.md + chart/values.yaml)
│   └── orders/   (README.md + chart/values.yaml)
└── terraform/
    └── variables.tf              # Kubernetes version changed
```

---

## Command quick reference

```bash
# 1. Connect AWS
aws configure
aws sts get-caller-identity

# 2. Clone your fork
git clone https://github.com/YOUR_GITHUB_USERNAME/retail-store-sample-app.git
cd retail-store-sample-app/terraform

# 3. Infrastructure (in stages)
terraform init
terraform apply -target=module.vpc
terraform apply -target=module.retail_app_eks     # use the module name from main.tf
aws eks update-kubeconfig --region YOUR_REGION --name YOUR_CLUSTER_NAME
kubectl get nodes                                  # empty is normal in Auto Mode
terraform apply
terraform output
kubectl get pods -A

# 4. Open the app and Argo CD
kubectl get svc -n ingress-nginx
kubectl port-forward svc/argocd-server -n argocd 8080:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# 5. GitOps (from the repo root)
git checkout gitops
git add src && git commit -m "change" && git push origin gitops
kubectl delete application retail-store-app -n argocd
kubectl apply -f argocd/applications/ -n argocd

# 6. Cleanup
cd terraform && terraform destroy
```

## Troubleshooting summary

| Symptom | Likely cause | Fix |
|---|---|---|
| `kubectl get nodes` says `No resources found` | Normal for Auto Mode | Deploy something; nodes appear |
| `forbidden ... cannot list resource "nodes"` | No EKS access entry for your IAM user | Create access entry with `AmazonEKSClusterAdminPolicy` (Part 6) |
| Terraform provider connection error | Cluster not created yet | Run the targeted EKS apply first |
| Load balancer EXTERNAL-IP `<pending>` | Still provisioning | Wait 1 to 3 minutes |
| No workflow run appears | Actions disabled, wrong branch, or path outside `src/` | Part 15 and Part 18 |
| `not authorized to perform ecr:...` | IAM policy incomplete | Edit the policy, re-run failed jobs |
| HTTP 403 when the bot pushes | Workflow permissions read-only | Set read and write permissions |
| Argo CD does not pick up changes | `repoURL` still points to the original repo, or wrong branch | Part 17 |
| `ImagePullBackOff` | Image or tag missing, wrong region | Check ECR and `values.yaml`, then `kubectl describe pod` |
| Higher-than-expected AWS bill | Cluster on an extended-support Kubernetes version, or resources left running | Use a standard-support version; run the full Cleanup |

## Portfolio summary (optional)

- Deployed a multi-language microservices retail application (Java, Go, Node.js with MySQL, PostgreSQL, DynamoDB, Redis) on **Amazon EKS Auto Mode**.
- Provisioned the infrastructure with **Terraform** (VPC, EKS, NGINX Ingress, cert-manager, Argo CD).
- Built a **GitHub Actions** CI pipeline (change detection, parallel matrix builds, push to **Amazon ECR**, automatic Helm values update) and continuous delivery with **Argo CD** following **GitOps**.
- Applied **least-privilege IAM** for the CI user.
