# Chapter 49: Build an Amazon EKS Cluster (Practical Lab)

## What you will build

By the end of this lab you will have a working **Amazon EKS** (Elastic Kubernetes Service) cluster on AWS, and your terminal will be connected to it with `kubectl`.

This cluster is the foundation for the DevOps mega project: a three-tier application deployed with a Jenkins CI/CD pipeline (Trivy, SonarQube, Docker, email notifications), GitOps with Argo CD, and Prometheus and Grafana monitoring installed with Helm. This chapter covers **only the cluster**. The later parts of the project are built on top of it.

| Item | Value used in this lab |
|---|---|
| Cluster name | `twa-cluster` |
| Region | `ap-south-1` (Mumbai). You may use any region, but use the **same region in every command** |
| Kubernetes version | `1.35` |
| Worker nodes | 2 x `t3.medium` |
| Tool that builds the cluster | `eksctl` (uses AWS CloudFormation behind the scenes) |

### Why EKS?

In Kubernetes, the **control plane** (API server, etcd, controller manager, scheduler) is the "brain" of the cluster. With EKS:

- AWS **runs and maintains the control plane** for you.
- You only create **worker nodes** (EC2 machines that run your applications) and deploy your apps.

### Cost warning (read before you start)

> **EKS is a paid service. It is not covered by the AWS Free Tier.**
> You pay for the control plane (per hour, roughly USD 0.10 per hour at the time of writing), for the EC2 worker nodes, and for the NAT gateway and other networking that `eksctl` creates. Check the current prices on the AWS pricing pages.
> **Delete everything when you finish practising.** The cleanup section is at the end of this chapter. Do not skip it.

---

## Prerequisites

| Requirement | Details |
|---|---|
| AWS account | With a valid payment method. You must be able to sign in to the AWS Management Console |
| Workstation | Ubuntu 22.04 or 24.04 (physical machine, VM, WSL2 on Windows, or an EC2 instance). Commands below are for **Ubuntu/Debian**. macOS users can use Homebrew equivalents noted in each step |
| Internet access | To download tools and reach AWS |
| Permissions | An IAM user with **AdministratorAccess** (created in Part 1). Learning only. In real companies give the minimum permissions needed |
| Ports | No inbound ports need to be opened on your workstation. The worker nodes get an SSH rule (port 22) only if you use `--ssh-access` (see Part 6) |

You do not need Docker or Terraform for this chapter.

### Open a terminal

- **Ubuntu desktop:** press `Ctrl + Alt + T`.
- **Windows with WSL2:** open the Start menu, type `Ubuntu`, press Enter.
- **macOS:** press `Cmd + Space`, type `Terminal`, press Enter.
- **Remote Ubuntu server or EC2:** connect using SSH (`ssh -i your-key.pem ubuntu@SERVER_IP`).

All commands in this lab are typed in that terminal, one block at a time.

---

## Part 1: Create an IAM user and access keys (AWS Console)

Your terminal must prove to AWS who you are. You do this with an **IAM user** and **access keys**. Do **not** use the root account for daily work.

### Step 1: Sign in and choose a region

1. Open `https://console.aws.amazon.com` and sign in.
2. In the top-right corner, click the **region name** (for example "N. Virginia") and choose **Asia Pacific (Mumbai) `ap-south-1`**.

### Step 2: Create the user

**Click:**

```text
AWS Console → search "IAM" → IAM → Users → Create user
```

1. **User name:** `eks-user`.
2. Leave **Provide user access to the AWS Management Console** **unchecked** (this user is only for the terminal). Click **Next**.
3. Select **Attach policies directly**.
4. In the search box type `AdministratorAccess` and tick it.
5. Click **Next**, then **Create user**.

### Step 3: Create access keys

**Click:**

```text
IAM → Users → eks-user → Security credentials tab → Create access key
```

1. Choose **Command Line Interface (CLI)**.
2. Tick the confirmation checkbox, click **Next**.
3. Click **Create access key**.
4. Copy the **Access key ID** and the **Secret access key**, or click **Download .csv file**. The secret is shown **only once**.

> **Security:** never paste these keys into chat, GitHub, screenshots, or videos. If you leak them, delete the key in the same IAM screen immediately.

### Verify

**Expected result:** the IAM Users list shows `eks-user`, and its Security credentials tab shows one access key with status **Active**.

---

## Part 2: Install the command-line tools

### Step 4: Install `unzip` and the AWS CLI

The AWS CLI lets you control AWS from the terminal.

**Ubuntu/Debian (x86_64 / Intel / AMD):**

```bash
sudo apt-get update
sudo apt-get install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

If your machine is ARM (check with `uname -m`: `aarch64` means ARM), replace the download URL with:

```text
https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip
```

**macOS (Homebrew):** `brew install awscli`

#### Verify

```bash
aws --version
```

**Expected result:** a line starting with `aws-cli/2.` (any 2.x version is fine).

### Step 5: Install `kubectl`

`kubectl` is the command-line tool for talking to a Kubernetes cluster.

**Ubuntu/Debian:**

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```

**macOS (Homebrew):** `brew install kubectl`

**Why v1.35?** `kubectl` should be within one minor version of your cluster (1.35).

#### Verify

```bash
kubectl version --client
```

**Expected result:** `Client Version: v1.35.x` (or close to it).

### Step 6: Install `eksctl`

`eksctl` is the official command-line tool for creating EKS clusters. It builds all the AWS resources (VPC, subnets, security groups, control plane) using **CloudFormation** in the background.

**Linux (x86_64):**

```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl
```

For an ARM Linux machine set `ARCH=arm64` in the first line.

**macOS (Homebrew):**

```bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

#### Verify

```bash
eksctl version
```

**Expected result:** a version number such as `0.2xx.x`. The exact number changes often. Always use a recent version, because old `eksctl` versions do not know new Kubernetes versions.

---

## Part 3: Connect your terminal to your AWS account

### Step 7: Configure the AWS CLI

```bash
aws configure
```

Answer the four prompts:

| Prompt | What to type |
|---|---|
| AWS Access Key ID | The access key ID from Part 1 |
| AWS Secret Access Key | The secret access key from Part 1 |
| Default region name | `ap-south-1` |
| Default output format | Press Enter (leave empty) |

This saves your settings in two files on your machine:

```text
~/.aws/credentials    (your keys, keep private)
~/.aws/config         (region and output format)
```

### Verify

```bash
aws sts get-caller-identity
```

**Expected result:** JSON showing your `Account` (12 digits) and an `Arn` ending with `user/eks-user`.

Then:

```bash
aws s3 ls
```

**Expected result:** a list of your S3 buckets, or **no output at all** if you have none. Both mean the connection works. An error means it does not.

### Common Problems

**Problem:** `Unable to locate credentials` or `InvalidClientTokenId`
**Possible reason:** the keys were typed wrongly, or the key was deleted or deactivated.
**Fix:** run `aws configure` again and paste the keys carefully, without spaces.

**Problem:** `aws: command not found`
**Possible reason:** the AWS CLI install did not finish.
**Fix:** repeat Step 4 and check for errors during `sudo ./aws/install`.

---

## Part 4: (Optional) Create an EC2 key pair for SSH access to nodes

Skip this part if you do not need to log in to worker nodes. If you skip it, remove `--ssh-access` and `--ssh-public-key` from the node group command in Part 6.

**Click:**

```text
AWS Console → EC2 → Network & Security → Key Pairs → Create key pair
```

1. **Name:** `twa-eks-key`.
2. **Key pair type:** RSA. **Private key file format:** `.pem`.
3. Click **Create key pair**. A `.pem` file downloads. Keep it safe.

> A key pair exists **per region**. Create it in the same region as your cluster (`ap-south-1`), or Part 6 will fail with "key pair does not exist".

---

## Part 5: Create the EKS control plane

### Step 8: Create the cluster without worker nodes

```bash
eksctl create cluster \
  --name twa-cluster \
  --region ap-south-1 \
  --version 1.35 \
  --without-nodegroup
```

| Option | Meaning |
|---|---|
| `--name` | The cluster name |
| `--region` | The AWS region where it is created |
| `--version` | The Kubernetes version. Use a version that is in **standard support** on EKS (check the EKS console or AWS documentation). Old versions move into paid **extended support** at a much higher hourly price |
| `--without-nodegroup` | Create only the control plane and networking. Worker nodes come in Part 6 |

**This takes about 10 to 20 minutes.** Leave the terminal open. You may see a warning about recommended policies for the VPC CNI add-on. You can ignore it for this lab.

Success ends with a line like:

```text
EKS cluster "twa-cluster" in "ap-south-1" region is ready
```

### Verify in the AWS Console (CloudFormation)

**Click:**

```text
AWS Console → CloudFormation → Stacks
```

**Expected result:** a stack named `eksctl-twa-cluster-cluster` with status `CREATE_COMPLETE`. Open it and click the **Resources** tab. You will see roughly 30 resources, such as the VPC, subnets, route tables, internet gateway, NAT gateway, the control plane and its security group.

**Click:**

```text
AWS Console → EKS → Clusters → twa-cluster
```

**Expected result:** cluster status **Active**, Kubernetes version `1.35`.

### Common Problems

**Problem:** `AccessDenied` or `UnauthorizedOperation` during creation
**Possible reason:** the IAM user lacks permissions.
**Fix:** confirm `AdministratorAccess` is attached to `eks-user` (IAM → Users → eks-user → Permissions).

**Problem:** `unsupported Kubernetes version`
**Possible reason:** your `eksctl` is too old, or the version is not offered in your region.
**Fix:** update `eksctl` (repeat Step 6), or choose another version listed in the EKS console (EKS → Create cluster → Kubernetes version drop-down).

**Problem:** the stack shows `ROLLBACK_COMPLETE` or `CREATE_FAILED`
**Possible reason:** a service quota was reached (for example the number of VPCs or Elastic IPs per region), or permissions were missing.
**Fix:** open the stack → **Events** tab and read the first red error. Delete the failed cluster with `eksctl delete cluster --name twa-cluster --region ap-south-1`, fix the cause, and run Step 8 again.

---

## Part 6: Connect an OIDC provider and add worker nodes

### Step 9: Associate the IAM OIDC provider

OIDC (OpenID Connect) lets Kubernetes service accounts securely receive AWS permissions later (for example for load balancers or storage add-ons). You enable it once per cluster.

```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster twa-cluster \
  --approve
```

#### Verify

```bash
aws eks describe-cluster --name twa-cluster --region ap-south-1 \
  --query "cluster.identity.oidc.issuer" --output text
```

**Expected result:** a URL like `https://oidc.eks.ap-south-1.amazonaws.com/id/XXXXXXXX`.

You can also check in the console: **IAM → Identity providers**. You should see an entry with the same `oidc.eks.ap-south-1...` address.

### Step 10: Create the node group (worker nodes)

A **node group** is a set of EC2 machines that join your cluster and run your pods.

```bash
eksctl create nodegroup \
  --cluster twa-cluster \
  --region ap-south-1 \
  --name twa-cluster-ng \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 2 \
  --node-volume-size 20 \
  --ssh-access \
  --ssh-public-key twa-eks-key
```

| Option | Meaning |
|---|---|
| `--name` | Node group name (`ng` = node group) |
| `--node-type` | EC2 size. `t3.medium` (2 vCPU, 4 GiB RAM) is the smallest sensible size. `t3.micro` and `t3.small` run out of memory quickly. Use `t3.large` if you plan to install Jenkins, monitoring and many pods |
| `--nodes` | Desired number of nodes |
| `--nodes-min` / `--nodes-max` | Autoscaling range. For example min `2`, max `5` allows growth to 5 nodes (autoscaling needs an extra component, the Cluster Autoscaler or Karpenter, which is not installed in this lab) |
| `--node-volume-size` | Disk size in GB for each node |
| `--ssh-access` | Opens port **22 (TCP)** on the nodes so you can SSH in |
| `--ssh-public-key` | Name of the **existing EC2 key pair** (Part 4). Remove both SSH options if you skipped Part 4 |

> **Note:** each `t3.medium` node can run at most about **17 pods**. This is enough for this lab, but the later mega project (Argo CD, Jenkins agents, Prometheus, Grafana, the application) needs more capacity. Use `t3.large` or more nodes when you build it.

Wait a few minutes. Success ends with:

```text
created 2 managed nodes in "twa-cluster-ng"
nodegroup "twa-cluster-ng" has 2 node(s)
```

### Verify

```bash
eksctl get nodegroup --cluster twa-cluster --region ap-south-1
```

**Expected result:** one row for `twa-cluster-ng` with desired capacity `2`.

**Click:**

```text
AWS Console → EKS → Clusters → twa-cluster → Compute tab
```

**Expected result:** the node group is **Active** and two nodes are listed. In **EC2 → Instances** you will see two running instances.

### Common Problems

**Problem:** `InvalidKeyPair.NotFound` / "key pair does not exist"
**Possible reason:** the key pair name is wrong, or it was created in a different region.
**Fix:** check the name in EC2 → Key Pairs in the **same region** (`ap-south-1`), or remove the two `--ssh-...` options.

**Problem:** node group creation fails with `VcpuLimitExceeded` or similar
**Possible reason:** a new AWS account has a low EC2 vCPU quota.
**Fix:** request a quota increase (AWS Console → Service Quotas → Amazon EC2 → "Running On-Demand Standard instances"), or use fewer or smaller nodes.

---

## Part 7: Connect `kubectl` to the cluster

### Step 11: Update your kubeconfig

`kubectl` reads the file `~/.kube/config` to know which cluster to talk to. Let AWS write the correct entry:

```bash
aws eks update-kubeconfig --region ap-south-1 --name twa-cluster
```

(`eksctl create cluster` usually does this for you already. Running the command again is safe.)

### Step 12: Check the connection

```bash
kubectl config current-context
kubectl get nodes
```

**Expected result:**

- The context name contains `twa-cluster` and `ap-south-1`.
- `kubectl get nodes` shows **two nodes** with status `Ready`. The names look like `ip-192-168-x-x.ap-south-1.compute.internal`.
- The **control plane is not listed**. That is normal, because AWS manages it.

If you still see an old local cluster (for example from kind or Minikube), switch context:

```bash
kubectl config get-contexts
kubectl config use-context <context-name-that-contains-twa-cluster>
```

You can keep many clusters (kind, Minikube, EKS) in one kubeconfig and switch between them with contexts.

### Step 13: Smoke test the cluster

Check the system pods that EKS runs for you:

```bash
kubectl get pods -A
```

**Expected result:** pods such as `aws-node`, `coredns` and `kube-proxy` in the `kube-system` namespace with status `Running`.

Deploy a test application, check it, then remove it:

```bash
kubectl create deployment nginx-test --image=nginx --replicas=2
kubectl get pods -o wide
kubectl delete deployment nginx-test
```

**Expected result:** two `nginx-test-...` pods reach `Running` (spread over the two nodes). After the delete command they disappear.

### Common Problems

**Problem:** `error: You must be logged in to the server (Unauthorized)`
**Possible reason:** the AWS CLI is using a different IAM identity from the one that created the cluster.
**Fix:** run `aws sts get-caller-identity`. It must show `eks-user`. If not, run `aws configure` again. Only the identity that created the cluster has admin access by default.

**Problem:** `The connection to the server localhost:8080 was refused`
**Possible reason:** `kubectl` has no cluster configured.
**Fix:** run Step 11 again.

**Problem:** nodes show `NotReady`
**Possible reason:** they are still starting.
**Fix:** wait 2 to 3 minutes and run `kubectl get nodes` again.

---

## Files and folders created in this lab

This lab creates **no project files**. The only local files that change are:

```text
~/.aws/credentials   (your access keys)
~/.aws/config        (default region)
~/.kube/config       (cluster connection details)
```

---

## Cleanup (delete everything to stop charges)

Do this as soon as you finish practising.

### Step 14: Delete the cluster

```bash
eksctl delete cluster --name twa-cluster --region ap-south-1
```

This removes the node group, control plane, and the networking that `eksctl` created. It takes about 10 to 15 minutes.

### Verify

- **CloudFormation → Stacks:** no stack starting with `eksctl-twa-cluster` remains.
- **EKS → Clusters:** the cluster is gone.
- **EC2 → Instances:** the two worker nodes are `terminated`.
- **VPC → NAT gateways** and **Elastic IPs:** none left from this cluster (unused Elastic IPs still cost money).

### Step 15: Remove credentials you no longer need

1. **IAM → Users → eks-user → Security credentials:** delete the access key (or delete the user).
2. **EC2 → Key Pairs:** delete `twa-eks-key` if you created it.
3. On your machine, remove the saved keys if the computer is shared:

```bash
rm -f ~/.aws/credentials
```

---

## Summary

| Step | What you did | Command or console path |
|---|---|---|
| 1 | Created IAM user with access key | IAM → Users → Create user |
| 2 | Installed AWS CLI, kubectl, eksctl | Steps 4 to 6 |
| 3 | Connected terminal to AWS | `aws configure`, `aws sts get-caller-identity` |
| 4 | Created the control plane | `eksctl create cluster ... --without-nodegroup` |
| 5 | Enabled OIDC | `eksctl utils associate-iam-oidc-provider ...` |
| 6 | Created worker nodes | `eksctl create nodegroup ...` |
| 7 | Connected kubectl | `aws eks update-kubeconfig ...`, `kubectl get nodes` |
| 8 | Cleaned up | `eksctl delete cluster ...` |

Key ideas:

- EKS is managed Kubernetes. AWS runs the control plane, you manage the worker nodes.
- `eksctl` builds AWS resources through CloudFormation.
- A node group is a set of EC2 worker machines.
- EKS costs money. Always delete the cluster after practice.
