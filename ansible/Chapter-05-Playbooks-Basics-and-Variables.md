# Practical 5 — Ansible Playbooks Basics, Variables, and Jinja2 Templating

## Objective
Learn what a Playbook is, write a simple "Hello" playbook, then extend it to install packages, and finally use variables with Jinja2 templating syntax.

## Prerequisites
* OS: Ubuntu (control node + worker nodes)
* Required software: Ansible installed
* Required files: `hosts.ini` inventory file (from Chapter 4), in the parent directory of your playbooks folder
* Required ports: SSH (22)
* Required permissions: `--become` where root access is required

## Pre-Check

```bash
ansible -i hosts.ini servers -m ping
```

**Expected result:** Both worker nodes return `SUCCESS => {"ping": "pong"}`. If not, revisit Chapter 4 before continuing.

---

## Concept — What is a Playbook?

A **Playbook** is a YAML file where you define the tasks you want Ansible to perform. Unlike ad-hoc commands (which run once and are gone), playbooks are **saved, reusable, and version-controllable**.

---

## Directory / File Structure

```text
~/
├── ansible_master_key
├── hosts.ini
└── playbooks/
    ├── hello.yml
    ├── install-packages.yml
    └── show-secrets.yml   (created in a later chapter)
```

---

## Step 1 — Create the Playbooks Folder

```bash
mkdir playbooks
cd playbooks
```

**What this does:** Creates a dedicated folder to keep all playbook YAML files organized, separate from the inventory file.

---

## Step 2 — Write a Simple "Hello" Playbook

```bash
vim hello.yml
```

Enter insert mode and type the complete file:

```yaml
---
- name: hello dosto
  hosts: servers
  become: yes
  tasks:
    - name: greet user
      command: echo Hello Shubham
```

**Explain each field:**
* `name:` (top level) → A human-readable name for this playbook/play.
* `hosts: servers` → Which group from the inventory file this playbook runs against.
* `become: yes` → Run as a superuser/root (equivalent to `--become` flag).
* `tasks:` → The list of actual actions to perform.
* `name:` (inside a task) → A label for that specific task, shown in the output.
* `command: echo Hello Shubham` → Runs the shell command `echo Hello Shubham` on each target host.

Save and exit:

```text
Esc  :wq  Enter
```

---

## Step 3 — Run the Playbook

```bash
ansible-playbook -i ../hosts.ini hello.yml
```

**What this does:**
* `ansible-playbook` → The command used to execute playbooks (different from `ansible` used for ad-hoc commands).
* `-i ../hosts.ini` → Points to the inventory file, which is one directory above (`../`) the `playbooks` folder.
* `hello.yml` → The playbook file to run.

> ⚠️ If you get the error `No inventory was parsed... Only implicit localhost is available`, it means you forgot the `-i <path-to-inventory>` flag or the path is wrong.

**Expected result:** Output shows:
```text
PLAY [hello dosto]
TASK [Gathering Facts] ... ok
TASK [greet user] ... changed
PLAY RECAP ...
```

---

## Step 4 — See the Actual Command Output (Verbose Mode)

By default, the `command` module output isn't printed on screen. To see it:

```bash
ansible-playbook -i ../hosts.ini hello.yml -v
```

**What this does:** `-v` (verbose) shows the standard output (stdout) of each task, so you'll now see `Hello Shubham` printed for both worker nodes.

---

## Step 5 — Turn the Hardcoded Text into a Variable

Edit `hello.yml` again:

```bash
vim hello.yml
```

Update it to:

```yaml
---
- name: hello dosto
  hosts: servers
  become: yes
  vars:
    my_name: Shubham
  tasks:
    - name: greet user
      command: echo Hello {{ my_name }}
```

**Explain:**
* `vars:` → A section where you define playbook-level variables.
* `my_name: Shubham` → A variable named `my_name` with the value `Shubham`.
* `{{ my_name }}` → **Jinja2 templating syntax**. Anything inside double curly braces is replaced by the actual value of that variable at runtime.

> ⚠️ **Important naming rule:** Avoid using `name` as a variable name — it is a **reserved word** in Ansible (used for playbook/task names). Use something like `my_name` instead, or you'll get a "Found variable using reserved name" warning.

> Also avoid passing variables as a list of dictionaries under `vars` — this is deprecated. Use the simple `key: value` format shown above. See official docs: `https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html`

Save and exit, then re-run:

```bash
ansible-playbook -i ../hosts.ini hello.yml -v
```

**Expected result:** Output shows `Hello Shubham` for both hosts, with no warnings.

Try changing `my_name: Shubham` to `my_name: dosto` and re-run — output should now show `Hello dosto`.

---

## Step 6 — Write a Playbook to Install Packages

```bash
vim install-packages.yml
```

Type the complete file:

```yaml
---
- name: install packages
  hosts: servers
  become: yes
  tasks:
    - name: install package
      apt:
        name: tree
        state: present
```

**Explain:**
* `apt:` → The Ansible module used for managing packages on Debian/Ubuntu systems.
* `name: tree` → The package to install (in this case, the `tree` command-line utility).
* `state: present` → Ensures the package is installed (installs the latest available version if not already present).

Save, exit, and run:

```bash
ansible-playbook -i ../hosts.ini install-packages.yml
```

**Expected result:**
```text
PLAY [install packages]
TASK [Gathering Facts] ... ok
TASK [install package] ... changed
PLAY RECAP ... changed=1
```

---

## Step 7 — Understand "Gathering Facts"

Every time Ansible connects to a host, before running your tasks, it automatically runs a task called `Gathering Facts`. This collects detailed system information about the target machine (OS family, distribution, CPU, memory, mount points, etc.) using the built-in `setup` module.

**View all gathered facts manually:**

```bash
ansible -i ../hosts.ini servers -m setup
```

**Expected result:** A large JSON output showing everything about the target machines — number of processors, product name, storage, mount points, and much more.

**Filter facts to find something specific (e.g. OS family):**

```bash
ansible -i ../hosts.ini servers -m setup | grep -i "os_family"
```

**Expected result:** Shows `"ansible_os_family": "Debian"` (for Ubuntu-based systems).

**Filter for distribution:**

```bash
ansible -i ../hosts.ini servers -m setup | grep -i "\"ansible_distribution\":"
```

**Expected result:** Shows `"ansible_distribution": "Ubuntu"`.

> These facts (like `ansible_distribution`, `ansible_os_family`) become very important later for writing conditional logic (Chapter 6) that behaves differently based on the OS.

---

## Troubleshooting

### Error 1 — `No inventory was parsed`
**Reason:** Missing or incorrect `-i` path to the inventory file.
**Fix:** Confirm the relative or absolute path, e.g. `-i ../hosts.ini` if running from inside the `playbooks/` folder.

### Error 2 — Warning: "Found variable using reserved name"
**Reason:** You used `name` as a custom variable name.
**Fix:** Rename the variable (e.g. `my_name`, `my_var`) — anything not clashing with Ansible's reserved keywords.

### Error 3 — Warning about deprecated list-of-dictionaries variable format
**Reason:** Old-style variable syntax was used under `vars`.
**Fix:** Use plain `key: value` pairs instead of a YAML list under `vars`.

---

## Final Result
By the end of this chapter you can:
* Write and run a basic Ansible Playbook
* Use `vars` and Jinja2 `{{ }}` templating to make playbooks dynamic
* Install packages using the `apt` module
* Understand and query Ansible's automatically gathered "facts"

## What's Next
**Chapter 6** covers **Loops** (repeating a task for multiple items, like installing several packages at once) and **Conditionals** (`when`, based on gathered facts like OS distribution).
