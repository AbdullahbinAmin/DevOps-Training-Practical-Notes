# Practical 11 — Terraform + Ansible Integration (Dynamic Inventory)

## Objective
Use Terraform to automatically provision 1 Control Node + 3 Worker Nodes (Ubuntu, Red Hat, Amazon Linux) on AWS, and automatically generate the Ansible inventory file using Terraform templates — so nothing is created manually.

## Prerequisites
* OS: Local machine with Terraform and Git installed; Ubuntu Control Node (provisioned by Terraform)
* Required software: Terraform (any recent version), Git, AWS CLI configured with valid credentials, Ansible (installed later on the control node)
* Required account: AWS account with permission to create EC2 instances, security groups, and key pairs
* Required repository: `https://github.com/<train-with-shubham-devops-community>/ansible-in-one-shot` (exact URL to be confirmed from the video description/GitHub — clone this repository)
* Required ports: SSH (22), HTTP (80), HTTPS (443)
* Required permissions: AWS credentials configured locally (`aws configure`), local SSH key generation ability
* Required files: `EC2.tf`, `inventory.tpl`, Terraform variable files (from the cloned repository)

## Pre-Check

```bash
terraform -version
git --version
aws sts get-caller-identity
```

**Expected result:** Terraform version is shown, Git version is shown, and `aws sts get-caller-identity` returns your AWS account details (confirms AWS CLI is authenticated).

> ⚠️ Verification Required: Confirm your AWS CLI is configured (`aws configure`) with a valid Access Key, Secret Key, and default region before proceeding.

---

## Architecture / Flow

```text
Terraform (local) → Provisions → 1 Control Node + 3 Worker Nodes (Ubuntu, RedHat, Amazon Linux)
                   → Also generates → Dynamic hosts.ini (via inventory.tpl template)

Control Node → Ansible installed → Uses the auto-generated hosts.ini → Configures all 3 Worker Nodes
```

---

## Step 1 — Clone the Reference Repository

```bash
mkdir -p ~/Documents/ansible-terraform-project
cd ~/Documents/ansible-terraform-project
git clone <REPOSITORY_URL>
cd ansible-in-one-shot/terraform
```

* `<REPOSITORY_URL>` → The GitHub repository URL containing the pre-written Terraform code for this project (referenced in the source video as being on the "Train with Shubham DevOps community" GitHub).

**What this does:** Downloads the pre-written Terraform configuration files (`EC2.tf`, variables, and the inventory template) so you don't have to write the AWS provisioning code from scratch.

---

## Step 2 — Understand the Key Terraform File: `EC2.tf`

Open `EC2.tf` to review it (do not blindly run it — understand it first, per the GitHub-repo-research rule):

```bash
cat EC2.tf
```

**What it contains (conceptually):**
* A `for_each` meta-argument that loops over a `variable` called `instances`.
* The `instances` variable is a **map object** — for each entry, it stores:
  * `ami` → The AMI ID (different for Ubuntu, Red Hat, Amazon Linux — each OS has a unique AMI ID per AWS region)
  * `user` → The SSH username for that OS (`ubuntu` for Ubuntu, `ec2-user` for Red Hat/Amazon Linux)
  * `os_family` → A label used later for the dynamic inventory template
  * `instance_type` → e.g. `t3.micro`

**Because `for_each` iterates over this map, each entry in the map creates one EC2 instance.** With 4 entries (control node + 3 worker OS types), Terraform creates **4 instances** from this single block of code.

**Finding the correct AMI IDs:** Go to `EC2 → Launch Instance → Application and OS Images`, search for each OS (Ubuntu, Red Hat, Amazon Linux), and note down the AMI ID shown for your selected AWS region. Each OS has a different AMI ID, and AMI IDs also differ by region.

---

## Step 3 — Understand the Security Group and Key Pair Blocks

The Terraform code also defines:
* A **security group** attached to the default VPC, allowing inbound SSH (22), HTTP (80), HTTPS (443). Additional ports (e.g. for Prometheus, Grafana) can be added later the same way.
* A **key pair** resource that uses a local public key file, expected at a path like `~/.ssh/terra-key-ansible.pub`.

**Generate this key pair locally first** (before running Terraform):

```bash
ssh-keygen
```

When prompted for the filename, enter the path matching what the Terraform code expects, e.g.:

```text
Enter file in which to save the key: ~/.ssh/terra-key-ansible
```

Press Enter twice for no passphrase.

**Expected result:** Two files created: `terra-key-ansible` (private) and `terra-key-ansible.pub` (public). Terraform's `key_pair` resource will read the `.pub` file and register it as an AWS key pair automatically.

---

## Step 4 — Understand the Dynamic Inventory Mechanism (`inventory.tpl` + `local_file`)

Terraform can write local files using the `local_file` resource. This project uses it to generate the Ansible inventory automatically.

**`inventory.tpl` (template file) — conceptual structure:**

```ini
[all:vars]
ansible_ssh_private_key_file=${ssh_key_path}
ansible_python_interpreter=/usr/bin/python3

%{ for name, ip in ubuntu_hosts ~}
${name} ansible_host=${ip} ansible_user=ubuntu
%{ endfor ~}

%{ for name, ip in redhat_hosts ~}
${name} ansible_host=${ip} ansible_user=ec2-user
%{ endfor ~}

%{ for name, ip in amazon_hosts ~}
${name} ansible_host=${ip} ansible_user=ec2-user
%{ endfor ~}
```

**Explain the concept:**
* This is a **Terraform template file** (`.tpl`), similar in spirit to a Jinja2 template — fixed/common parts stay as-is, and variable parts (IP, name, OS-specific loop) are injected at generation time.
* A separate Terraform resource (commonly named something like `generate_inventory`, using `local_file`) takes the **outputs** of the EC2 resources (public IP, username, OS family — pulled per-instance via the same `for_each` used for instance creation) and renders them into this template, producing a real, final `hosts.ini` file (typically written to `inventories/dev/hosts.ini` or similar, depending on the repo's folder layout).
* Every time you run `terraform apply`, this inventory file is regenerated automatically — reflecting whatever instances currently exist.

> This is exactly what is meant by a **Dynamic Inventory File** — instead of manually typing IP addresses into `hosts.ini` (as done in Chapter 4), Terraform generates it for you based on the actual provisioned infrastructure.

---

## Step 5 — Add a Tag to Identify Terraform-Managed Resources (Recommended)

Before applying, it's good practice to add an identifying tag in `EC2.tf`, so these instances can be filtered apart from any manually created ones:

```hcl
tags = {
  Name      = each.key
  ManagedBy = "Terraform"
  Project   = "ansible-training"
}
```

**Why:** If your AWS account already has other EC2 instances (e.g. from Chapters 2–10), this tag makes it easy to filter and identify exactly which instances belong to this Terraform-managed project.

---

## Step 6 — Initialize Terraform

```bash
terraform init
```

**What this does:** Downloads the required Terraform provider plugins (e.g. the AWS provider) and sets up the local backend.

**Expected result:** Output ends with `Terraform has been successfully initialized!`.

---

## Step 7 — Validate the Configuration

```bash
terraform validate
```

**Expected result:** `Success! The configuration is valid.`

---

## Step 8 — Preview the Plan

```bash
terraform plan
```

**Expected result:** Terraform shows a plan to create:
* 4 EC2 instances (1 control node + 3 worker nodes: Ubuntu, Red Hat, Amazon Linux)
* 1 security group
* 1 key pair
* 1 local inventory file (via `local_file`)

Review this output carefully before applying — confirm the resource counts match your expectations (e.g. "Plan: 8 to add" if counting all sub-resources including the inventory file).

---

## Step 9 — Apply the Terraform Configuration

```bash
terraform apply
```

When prompted:

```text
Do you want to perform these actions?
  Enter a value:
```

Type `yes`.

**What this does:** Terraform provisions all AWS resources (instances, security group, key pair) and generates the local dynamic inventory file, all in one command.

**Expected result:** Terraform outputs values including the Control Node's public IP address, and confirms all resources were created (e.g. `Apply complete! Resources: 8 added, 0 changed, 0 destroyed.`).

---

## Step 10 — SSH Into the Control Node

Using the Terraform output for the control node's public IP:

```bash
chmod 400 ~/.ssh/terra-key-ansible
ssh -i ~/.ssh/terra-key-ansible ubuntu@<CONTROL_NODE_PUBLIC_IP>
```

When prompted `Are you sure you want to continue connecting?`, type `yes`.

**Expected result:** You are logged into the newly Terraform-provisioned Control Node.

---

## Step 11 — Install Ansible on the New Control Node

```bash
sudo apt-get update
sudo apt-get install ansible
```

**Verify:**

```bash
ansible --version
```

---

## Step 12 — Copy the Terraform-Generated Inventory and Keys to the Control Node

While the Control Node's Ansible installation completes, from your **local machine** (a separate terminal), copy the generated inventory file and the private key to the Control Node:

```bash
scp -i ~/.ssh/terra-key-ansible \
  ./inventories/dev/hosts.ini \
  ubuntu@<CONTROL_NODE_PUBLIC_IP>:/home/ubuntu/hosts.ini

scp -i ~/.ssh/terra-key-ansible \
  ~/.ssh/terra-key-ansible \
  ubuntu@<CONTROL_NODE_PUBLIC_IP>:/home/ubuntu/terra-key-ansible
```

**What this does:** `scp` (secure copy) transfers files from your local machine directly to the Control Node over SSH — the actual local path to `hosts.ini` depends on where the `local_file` resource wrote it (check your Terraform output or the `local_file` resource block for the exact path).

**On the Control Node**, fix the private key permissions:

```bash
chmod 400 terra-key-ansible
```

**Verify the inventory arrived correctly:**

```bash
cat hosts.ini
```

**Expected result:** You should see a fully-populated inventory file with the control node and all 3 worker nodes (Ubuntu, Red Hat, Amazon Linux), each with correct IPs, usernames, and the private key path — generated automatically, with **zero manual typing**.

---

## Step 13 — Test Connectivity Using the Dynamic Inventory

```bash
ansible -i hosts.ini servers -m ping
```

**Expected result:** All 3 worker nodes (Ubuntu, Red Hat, Amazon Linux) respond `pong` — confirming the auto-generated inventory works correctly.

---

## Step 14 — Run the Multi-OS Docker Role (From Chapter 10) Against This New Infrastructure

Clone or copy your `roles/docker` folder (built in Chapters 9–10) onto this Control Node, along with the `install-docker-with-role.yml` playbook, then run:

```bash
ansible-playbook -i hosts.ini install-docker-with-role.yml
```

**Expected result:** Because of the `include_tasks: "install_{{ ansible_distribution | lower }}.yml"` logic from Chapter 10, Docker installs correctly on **all three different operating systems** in a single run — no manual per-OS commands needed.

**Verify:**

```bash
ansible -i hosts.ini servers -a "docker --version" --become
```

---

## Step 15 — Cleanup (Destroy Infrastructure)

When you are done practicing, tear down everything Terraform created (to avoid ongoing AWS charges):

```bash
terraform destroy
```

When prompted, type `yes`.

**What this does:** Deletes all 4 EC2 instances, the security group, the key pair (AWS-side), and removes the locally generated inventory file — everything Terraform created is cleanly removed.

**Verify:**

```text
AWS Console → EC2 → Instances
```

All Terraform-managed instances should now show as **Terminated**.

---

## Troubleshooting

### Error 1 — `terraform apply` fails with AMI not found
**Reason:** AMI IDs are region-specific and change over time; the AMI ID in the repo's variables may not exist in your selected AWS region.
**Fix:** Go to `EC2 → Launch Instance → AMI Catalog`, find the current AMI ID for Ubuntu/RHEL/Amazon Linux in your region, and update the `instances` variable map accordingly.

### Error 2 — Generated `hosts.ini` has blank or missing IPs
**Reason:** `terraform apply` may not have fully completed, or the `local_file` resource ran before the EC2 `public_ip` outputs were available.
**Fix:** Re-run `terraform apply` — Terraform automatically handles resource dependency ordering, but if a partial apply occurred (e.g. it was interrupted), a clean re-apply usually fixes it.

### Error 3 — SSH fails to worker nodes when running the Docker role from the new Control Node
**Reason:** The private key (`terra-key-ansible`) wasn't copied to the Control Node, or its permissions aren't `400`.
**Fix:** Re-check Step 12 — confirm the key file exists on the Control Node and `chmod 400` was applied.

### Error 4 — Confusing "control node trying to SSH to itself" error
**Reason:** If the control node is accidentally also included in the `servers` group being targeted for SSH-based tasks meant for workers, Ansible may try to SSH from the control node to itself, which can behave unexpectedly depending on setup.
**Fix:** Keep the control node in a separate inventory group (e.g. `[control]`) from the worker nodes (`[servers]`), and only target `servers` for the Docker/Nginx playbooks.

---

## Final Result
By the end of this chapter, you have a **fully automated, end-to-end pipeline**:
* Terraform provisions 1 Control Node + 3 Worker Nodes (Ubuntu, Red Hat, Amazon Linux) on AWS
* Terraform automatically generates a working Ansible dynamic inventory file (`hosts.ini`) — no manual IP entry
* Ansible (installed on the Control Node) uses this dynamic inventory to run the multi-OS Docker role (from Chapter 10) across all 3 different operating systems in a single playbook run
* `terraform destroy` cleanly tears down the entire environment when done

This is the complete Terraform + Ansible integration workflow — a real-world, production-style DevOps automation pipeline.
