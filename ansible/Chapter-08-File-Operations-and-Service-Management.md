# Practical 8 — File Operations and Service Management (Custom Nginx Page)

## Objective
Install Nginx via a playbook, copy a custom `index.html` file to the server (replacing the default Nginx welcome page), restart and enable the Nginx service, and verify the custom page loads in a browser.

## Prerequisites
* OS: Ubuntu (worker nodes)
* Required software: Ansible installed on Control Node
* Required files: `hosts.ini` inventory file, a custom `index.html` file
* Required ports: SSH (22), HTTP (80) open in the security group
* Required permissions: `become: yes` (root access needed to install packages and copy into `/var/www/html`)

## Pre-Check

```bash
ansible -i ../hosts.ini servers -m ping
```

**Expected result:** Both worker nodes respond `pong`.

---

## Directory / File Structure

```text
playbooks/
├── index.html              (your custom HTML file)
└── setup-nginx.yml         (the playbook)
```

---

## Step 1 — Prepare a Custom `index.html` File

Create a simple custom HTML page (any editor/tool can be used to generate this — even AI tools). Example minimal content:

```bash
nano index.html
```

```html
<!DOCTYPE html>
<html>
<head>
  <title>Train with Shubham</title>
</head>
<body style="background:#111; color:#fff; text-align:center; padding-top:100px;">
  <h1>Hello Dosto, Welcome to Train with Shubham Channel</h1>
</body>
</html>
```

Save and exit. This file must be in the **same folder** as your playbook, since we will reference it as a relative path (source).

---

## Step 2 — Write the `setup-nginx.yml` Playbook

```bash
vim setup-nginx.yml
```

Type the complete file:

```yaml
---
- name: setup nginx with custom file
  hosts: servers
  become: yes
  tasks:
    - name: install nginx
      apt:
        name: nginx
        state: present

    - name: copy the custom html file
      copy:
        src: index.html
        dest: /var/www/html/index.html

    - name: restart nginx
      service:
        name: nginx
        state: reloaded

    - name: enable nginx
      systemd:
        name: nginx
        enabled: true
```

**Explain each task:**
* **install nginx** → Uses the `apt` module to install the Nginx package (`state: present`).
* **copy the custom html file** → Uses the built-in `copy` module.
  * `src: index.html` → Source file, relative to the playbook's directory (on the Control Node).
  * `dest: /var/www/html/index.html` → Destination path on the target server — this is the default folder where Nginx serves its `index.html` from.
* **restart nginx** → Uses the `service` module with `state: reloaded` to restart the Nginx service so it picks up the new file. The module name to search for in Ansible docs is `ansible.builtin.service`.
* **enable nginx** → Uses the `systemd` module with `enabled: true` (NOT `state: enabled`, which is an incorrect/older syntax). This ensures Nginx automatically starts if the server reboots.

> ⚠️ Common mistake: Writing `state: enabled` under the `service`/`systemd` module for enabling on boot. The correct syntax with the `systemd` module is `enabled: true`, as a separate key — not `state: enabled`.

Save and exit.

---

## Step 3 — Run the Playbook

```bash
ansible-playbook -i ../hosts.ini setup-nginx.yml
```

**Expected result:**
```text
PLAY [setup nginx with custom file]
TASK [Gathering Facts] ... ok
TASK [install nginx] ... changed
TASK [copy the custom html file] ... changed
TASK [restart nginx] ... changed
TASK [enable nginx] ... changed
PLAY RECAP ... changed=4
```

---

## Step 4 — Verify in the Browser

Open in a browser:

```text
http://<WORKER_NODE_1_IP>
http://<WORKER_NODE_2_IP>
```

**Expected result:** Instead of the default "Welcome to nginx!" page, you should see your custom HTML content ("Hello Dosto, Welcome to Train with Shubham Channel").

---

## Step 5 — Verify the Service Directly on the Server (Optional but Recommended)

SSH into a worker node and check:

```bash
sudo systemctl status nginx
```

**Expected result:** Service shows `active (running)` and `enabled` (so it will auto-start on reboot).

---

## Troubleshooting

### Error 1 — Custom file not showing, default Nginx page still visible
**Reason:** `dest:` path is wrong, or the `copy` task ran before `nginx` was installed (folder `/var/www/html` may not exist yet).
**Fix:** Confirm task order: install nginx → copy file → restart → enable, in that exact sequence. Confirm the file path is exactly `/var/www/html/index.html`.

### Error 2 — `state: enabled` gives a module error
**Reason:** Incorrect argument syntax for the `systemd` module.
**Fix:** Use:
```yaml
- name: enable nginx
  systemd:
    name: nginx
    enabled: true
```

### Error 3 — Page not loading in browser at all
**Reason:** Port 80 (HTTP) not open in the AWS security group.
**Fix:** Go to `EC2 → Instance → Security → Security Groups → Edit inbound rules` and add an HTTP (port 80) rule.

---

## Final Result
By the end of this chapter:
* Nginx is installed on both worker nodes via Ansible
* A custom `index.html` file replaces the default Nginx welcome page
* Nginx is restarted (reloaded) to pick up changes, and enabled to auto-start on reboot
* The custom page is confirmed working by visiting each worker node's public IP in a browser

## What's Next
**Chapter 9** introduces **Ansible Roles** — a way to package reusable, structured automation (using Ansible Galaxy) so the same logic (like installing Docker) can be reused across any playbook without rewriting it.
