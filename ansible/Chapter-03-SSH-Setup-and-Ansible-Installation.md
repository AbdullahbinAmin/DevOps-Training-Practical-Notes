# Practical 3 — SSH Key Setup Between Control Node and Worker Nodes + Installing Ansible

## Objective
Secure the AWS key file, generate a new SSH key pair on the Control Node (master key), distribute the public key to both Worker Nodes, verify SSH access, and install Ansible on the Control Node.

## Prerequisites
* OS: Ubuntu on all 3 instances (from Chapter 2)
* RAM: t3.micro default
* CPU: t3.micro default
* Required software: None yet — will install Ansible in this chapter
* Required account: AWS account with the 3 running instances
* Required repository: None
* Required ports: SSH (22) open on all instances
* Required permissions: Read access to the downloaded `.pem` key file
* Required files: `ansible-in-one-shot-key.pem` (downloaded in Chapter 2)

## Pre-Check
Confirm the `.pem` file exists on your local machine:

```bash
ls ~/Downloads/ansible-in-one-shot-key.pem
```

**Expected output:** The file path should be listed (no "No such file" error).

---

## Architecture / Flow

```text
Your Laptop → SSH (.pem key) → Control Node → SSH (new master key) → Worker Node 1, Worker Node 2
```

---

## Step 1 — Fix Permissions on the AWS `.pem` Key

By default, downloaded key files have very open permissions, which SSH will reject.

```bash
cd ~/Downloads
chmod 400 ansible-in-one-shot-key.pem
```

**What this does:**
`chmod 400` makes the file **read-only for your user only**, and gives no access to anyone else. SSH requires private key files to have restricted permissions, or it will refuse to use them.

**Verify:**

```bash
ls -l ansible-in-one-shot-key.pem
```

**Expected result:** Permissions should show `-r--------`.

---

## Step 2 — SSH into the Control Node

**Step 1:** Go to `EC2 → Instances → control-node → Connect` tab, and copy the SSH command shown (it uses the "SSH client" tab example).

**Step 2:** Run the command (example — replace `<PUBLIC_IP>` with your actual control node public IP/DNS):

```bash
ssh -i ansible-in-one-shot-key.pem ubuntu@<PUBLIC_IP>
```

* `<PUBLIC_IP>` → Public IPv4 address or Public DNS of your `control-node` instance (found on the EC2 Instances page)

**Step 3:** When prompted:

```text
Are you sure you want to continue connecting (yes/no)?
```

Type `yes`.

**Expected result:** You should now see a shell prompt like `ubuntu@ip-xxx-xxx-xxx-xxx:~$` — this confirms you are logged into the Control Node.

---

## Step 3 — Update the Control Node and Install Ansible

Run these two commands on the Control Node:

```bash
sudo apt-get update
```

**What this does:** Refreshes the list of latest available package versions.

```bash
sudo apt-get install ansible
```

**What this does:** Installs Ansible (written in Python) along with its dependencies (e.g. Kerberos libraries for hashing, DNS/SELinux related packages).

When prompted `Do you want to continue? [Y/n]`, type `y`.

> ⚠️ If you are NOT using Ubuntu, do not guess the install command. Go to the **Official Installation Guide**:
> `https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html`
> Follow the instructions for your OS/version (Linux via package manager, or via `pip install ansible` since Ansible is a Python-based tool).

**Verify:**

```bash
ansible --version
```

**Expected result:** Output should show the installed Ansible version (e.g. `ansible [core 2.16.3]`), along with:
* `config file = None` (no config file yet — this is normal, we will create one later)
* Jinja version (templating engine used by Ansible)
* Python version being used

> **Important:** Since Ansible is built on Python, **Python must be installed on every machine** — the Control Node AND all Worker Nodes. This is a core prerequisite for Ansible to work.

---

## Step 4 — Generate an SSH Key Pair on the Control Node (Master Key)

We now need a dedicated SSH key pair that the Control Node will use to connect to the Worker Nodes.

```bash
ssh-keygen
```

**What this does:** Starts an interactive key generation process.

When prompted:
* `Enter file in which to save the key:` → type `ansible_master_key`
* `Enter passphrase (empty for no passphrase):` → press Enter (leave empty)
* `Enter same passphrase again:` → press Enter (leave empty)

**Expected result:** Two files are created in your current directory:
* `ansible_master_key` (private key — stays on the Control Node)
* `ansible_master_key.pub` (public key — goes to the Worker Nodes)

---

## Step 5 — Copy the Public Key to Both Worker Nodes

**Step 1:** Display the public key content so you can copy it:

```bash
cat ansible_master_key.pub
```

**Step 2:** Open a **separate** connection to `worker-node-1` (use AWS Console → EC2 Instance Connect, or SSH using the `.pem` file the same way as Step 2 above, but targeting worker-node-1's IP).

**Step 3:** On `worker-node-1`, edit the authorized keys file:

```bash
vim ~/.ssh/authorized_keys
```

**What to do:** Go into insert mode, paste the public key content (from `ansible_master_key.pub`) on a new line, then save and exit:

```text
Esc  :wq  Enter
```

**Step 4:** Repeat Step 2 and Step 3 for `worker-node-2` — add the same public key to `~/.ssh/authorized_keys` on `worker-node-2` as well.

**Why:** SSH key-based login works with a private/public key pair. The **private key** stays on the machine that wants to connect (Control Node). The **public key** must be placed on the machine being connected to (each Worker Node), inside `~/.ssh/authorized_keys`.

---

## Step 6 — Verify SSH Access From Control Node to Each Worker Node

Back on the **Control Node** terminal:

```bash
ssh -i ansible_master_key ubuntu@<WORKER_NODE_1_IP>
```

* `<WORKER_NODE_1_IP>` → Public IP address of worker-node-1

When prompted `Are you sure you want to continue connecting?`, type `yes`.

**Expected result:** You should log into `worker-node-1` successfully without being asked for a password.

Exit back to the Control Node:

```bash
exit
```

Repeat the same test for worker-node-2:

```bash
ssh -i ansible_master_key ubuntu@<WORKER_NODE_2_IP>
```

**Expected result:** Successful login to `worker-node-2` as well. Exit back with `exit`.

**What this confirms:**
```text
Control Node CAN SSH into Worker Node 1
Control Node CAN SSH into Worker Node 2
```

This means Ansible (which relies purely on SSH) will now be able to connect to both worker nodes.

---

## Troubleshooting

### Error 1 — `Permission denied (publickey)` when SSHing to worker node
**Reason:** The public key was not correctly pasted into `~/.ssh/authorized_keys` on the worker node, or the private key permissions are too open.
**Fix:**
```bash
chmod 600 ansible_master_key
```
Then re-check that the public key line in `~/.ssh/authorized_keys` on the worker node matches exactly, with no line breaks in the middle.

### Error 2 — `ansible: command not found`
**Reason:** Ansible installation did not complete, or the update step was skipped.
**Fix:** Re-run `sudo apt-get update` followed by `sudo apt-get install ansible`.

### Error 3 — Connection timeout when SSHing
**Reason:** Security group on the target instance does not allow inbound SSH (port 22) from the Control Node.
**Fix:** Go to the target instance's Security Group in AWS Console and confirm SSH (port 22) inbound rule is present.

---

## Final Result
At the end of this chapter:
* Ansible is installed and verified on the Control Node (`ansible --version` works)
* A dedicated master SSH key pair (`ansible_master_key` / `ansible_master_key.pub`) exists on the Control Node
* The public key is present on both `worker-node-1` and `worker-node-2` in `~/.ssh/authorized_keys`
* SSH access from Control Node to both Worker Nodes is confirmed working, without password prompts

## What's Next
**Chapter 4** will cover creating an Ansible **inventory file** (`hosts.ini`) to formally register these worker nodes with Ansible, and running your first **ad-hoc commands**.
