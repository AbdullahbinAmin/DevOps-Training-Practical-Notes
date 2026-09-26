# Practical 4 — Creating the Ansible Inventory File and Running Ad-hoc Commands

## Objective
Create a static inventory file (`hosts.ini`) that tells Ansible which servers exist and how to connect to them, then run ad-hoc (one-off) commands against those servers.

## Prerequisites
* OS: Ubuntu (control node), Ubuntu (worker nodes)
* Required software: Ansible installed on Control Node (Chapter 3)
* Required files: `ansible_master_key` private key (Chapter 3), public key already on both workers
* Required ports: SSH (22)
* Required permissions: `chmod 400` on the private key

## Pre-Check

```bash
ansible --version
```

**Expected result:** Ansible version is shown, and `config file = None` (confirms no config file exists yet, which is fine — we will use an inventory file directly).

---

## Directory / File Structure

We will create a working folder with the SSH keys and the inventory file:

```text
~/ (home directory on control node)
├── ansible_master_key
├── ansible_master_key.pub
└── hosts.ini
```

---

## Step 1 — Create the Inventory File

```bash
nano hosts.ini
```

(You can use `vim` instead of `nano` if you prefer.)

**What this does:** Creates a new file called `hosts.ini`. This is Ansible's **inventory file** — a special file that lists all your servers, grouped and configured with connection details.

---

## Step 2 — Write the Inventory File Content

Type the following complete content into `hosts.ini`:

```ini
[servers]
worker-node-1 ansible_host=<WORKER_NODE_1_IP>
worker-node-2 ansible_host=<WORKER_NODE_2_IP>

[servers:vars]
ansible_ssh_private_key_file=/home/ubuntu/ansible_master_key
ansible_python_interpreter=/usr/bin/python3
ansible_user=ubuntu
ansible_host_key_checking=false
```

**Explain each part:**
* `[servers]` → This defines a **group** named `servers`. `worker-node-1` and `worker-node-2` are the **children** (hosts) of this group.
* `ansible_host` → A built-in Ansible variable that holds the IP address of that specific host.
* `[servers:vars]` → Common variables applied to **all** hosts inside the `servers` group (instead of repeating them for every host).
* `ansible_ssh_private_key_file` → Path to the private key used to SSH into these hosts. Replace `/home/ubuntu/ansible_master_key` with the actual path where you generated the key in Chapter 3.
* `ansible_python_interpreter` → Tells Ansible exactly which Python binary to use on the target machine. Find this with `which python3` on the worker node — it is usually `/usr/bin/python3`.
* `ansible_user` → The SSH username to log in as (for Ubuntu AMIs, this is `ubuntu`).
* `ansible_host_key_checking=false` → Skips the interactive "Are you sure you want to continue connecting (yes/no)?" prompt on first-time SSH connections. Without this, Ansible would need manual "yes" confirmation the first time it touches a new host.

Save and exit:

```text
Esc  :wq  Enter
```

---

## Step 3 — Explore Available Ansible Commands (Optional but Useful)

```bash
ansible
```

**What this does:** Running `ansible` alone (with no arguments) shows you the help/usage options — including things like listing hosts, checking version, dry-run mode, and vault usage. This is a quick way to explore what Ansible can do.

---

## Step 4 — Ping All Servers (First Ad-hoc Command)

```bash
ansible -i hosts.ini servers -m ping
```

**What this does:**
* `-i hosts.ini` → Tells Ansible to use `hosts.ini` as the inventory file
* `servers` → The group of hosts to target (as defined in the inventory)
* `-m ping` → Runs the built-in `ping` **module**. This module attempts an SSH connection and, if successful, returns a `pong` response. (Note: this is NOT a network ICMP ping — it verifies Ansible connectivity + Python availability.)

**Expected output:** For both worker-node-1 and worker-node-2:

```json
worker-node-1 | SUCCESS => {
    "ping": "pong"
}
worker-node-2 | SUCCESS => {
    "ping": "pong"
}
```

If prompted "Are you sure you want to continue connecting?", type `yes` (this only happens the first time if `ansible_host_key_checking=false` was not yet applied).

---

## Step 5 — Run Other Useful Ad-hoc Commands

**Check uptime on all servers:**

```bash
ansible -i hosts.ini servers -a "uptime"
```

**What this does:** `-a` runs a raw shell command (using the default `command` module) on all hosts in the `servers` group. `uptime` shows how long each server has been running.

**Check disk space:**

```bash
ansible -i hosts.ini servers -a "df -h"
```

**Check memory usage:**

```bash
ansible -i hosts.ini servers -a "free -h"
```

**Update all servers in parallel:**

```bash
ansible -i hosts.ini servers -a "sudo apt-get update"
```

**Expected result:** Both worker nodes update **at the same time** (in parallel), not one after another — this is the core benefit of using Ansible.

---

## Step 6 — Install a Package Using an Ad-hoc Command (with Root Privileges)

To install Nginx across both worker nodes in one command:

```bash
ansible -i hosts.ini servers -a "apt-get install nginx" --become
```

* `--become` → Runs the command as a **root/superuser** (like using `sudo`). Required because package installation needs elevated privileges.

**To see detailed/verbose output** (useful for debugging or seeing exactly what's happening):

```bash
ansible -i hosts.ini servers -a "apt-get install nginx" --become -v
```

**To auto-confirm any yes/no prompts during install:**

```bash
ansible -i hosts.ini servers -m apt -a "name=nginx state=present" --become
```

**What this does:** Uses the `apt` module (instead of a raw shell command) with `name=nginx state=present`, which is the proper Ansible way to install a package idempotently (it won't reinstall if already present).

**Verify:**
Open the Public IP of either worker node in a browser:

```text
http://<WORKER_NODE_1_IP>
http://<WORKER_NODE_2_IP>
```

**Expected result:** The default "Welcome to nginx!" page should load on both.

---

## Troubleshooting

### Error 1 — `UNREACHABLE` error on ping
```text
"msg": "Failed to connect to the host via ssh"
```
**Reason:** Wrong IP address in inventory, wrong private key path, or security group blocking port 22.
**Fix:** Double check `ansible_host` values and `ansible_ssh_private_key_file` path. Confirm `chmod 400` is set on the key.

### Error 2 — Prompted "Are you sure you want to continue connecting?" every time
**Reason:** `ansible_host_key_checking=false` was not added, or wasn't saved correctly in the inventory.
**Fix:** Re-check the `[servers:vars]` section of `hosts.ini` includes this line, then re-run.

### Error 3 — Permission denied errors on package install
**Reason:** Forgot the `--become` flag.
**Fix:** Add `--become` to any ad-hoc command that needs root access (installing packages, restarting services, etc.).

---

## Final Result
By the end of this chapter you have:
* A working static inventory file `hosts.ini` with 2 worker nodes grouped under `[servers]`
* Verified SSH+Python connectivity to both nodes using `ansible ... -m ping`
* Run multiple ad-hoc commands (uptime, disk space, memory, update, install nginx) across both servers simultaneously, without manually logging into each one

## What's Next
**Chapter 5** introduces **Playbooks** — reusable YAML files that define multi-step automation tasks, along with variables and Jinja2 templating.
