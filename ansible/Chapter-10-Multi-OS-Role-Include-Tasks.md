# Practical 10 — Multi-OS Docker Role Using `include_tasks` (Ubuntu, Red Hat, Amazon Linux)

## Objective
Extend the Docker role from Chapter 9 to correctly install Docker on **three different operating systems** (Ubuntu, Red Hat, Amazon Linux) using OS-specific task files, automatically included based on the `ansible_distribution` fact — without writing repeated loops or if/else chains.

## Prerequisites
* OS: A mixed environment — one Ubuntu Control Node, and worker nodes on **Ubuntu**, **Red Hat**, and **Amazon Linux** (this chapter assumes you have such a mixed setup; see Chapter 11 for how to provision this automatically with Terraform)
* Required software: Ansible installed on Control Node
* Required files: `hosts.ini` inventory covering all 3 worker OS types, existing `roles/docker` folder (from Chapter 9)
* Required ports: SSH (22)
* Required permissions: `become: yes`

## Pre-Check

```bash
ansible -i ../hosts.ini servers -m ping
```

**Expected result:** All worker nodes (Ubuntu, Red Hat, Amazon Linux) respond `pong`. If any fail, confirm Python is installed on that OS and the SSH user/key is correct for that AMI (e.g. `ec2-user` for Amazon Linux/RHEL instead of `ubuntu`).

---

## Concept — The Problem With a Single-OS Role

Different Linux distributions use different package managers:

| OS | Package Manager |
|---|---|
| Ubuntu | `apt` |
| Red Hat | `dnf` |
| Amazon Linux | `dnf` |

A single `tasks/main.yml` written only with the `apt` module (as in Chapter 9) will **fail** on Red Hat and Amazon Linux. We need OS-specific installation logic, automatically selected at runtime.

---

## Directory / File Structure (Target)

```text
roles/docker/
├── tasks/
│   ├── main.yml                    (includes the correct OS file automatically)
│   ├── install_ubuntu.yml
│   ├── install_redhat.yml
│   └── install_amazon.yml
├── defaults/
│   └── main.yml                    (docker_users, docker_service_enabled)
├── handlers/
│   └── main.yml                    (restart docker)
```

---

## Step 1 — Create Separate OS-Specific Task Files

```bash
cd roles/docker/tasks
vim install_ubuntu.yml
```

Type:

```yaml
---
- name: update apt cache
  apt:
    update_cache: yes

- name: install docker on ubuntu
  apt:
    name: docker.io
    state: latest
```

Save and exit. Repeat for Amazon Linux:

```bash
vim install_amazon.yml
```

Type:

```yaml
---
- name: update dnf cache
  dnf:
    update_cache: yes

- name: install docker on amazon linux
  dnf:
    name: docker
    state: latest
```

Save and exit. Now Red Hat:

```bash
vim install_redhat.yml
```

Type:

```yaml
---
- name: remove old docker versions
  dnf:
    name:
      - podman
      - runc
      - docker
    state: absent

- name: install dnf plugins core
  dnf:
    name: dnf-plugins-core
    state: present

- name: add docker ce repo
  command: dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

- name: install docker ce
  dnf:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: present

- name: start and enable docker
  systemd:
    name: docker
    state: started
    enabled: true
```

**Explain:**
* Red Hat needs old conflicting packages (`podman`, `runc`, older `docker`) removed first (`state: absent`) before installing Docker CE properly.
* `dnf-plugins-core` provides `dnf config-manager`, needed to add the official Docker CE repository.
* The Docker CE repo is added, then `docker-ce`, `docker-ce-cli`, and `containerd.io` are installed together.
* The Docker daemon (service) is then explicitly started and enabled.

> ⚠️ Verification Required: Always check the current official Docker installation instructions for your exact Red Hat version at `https://docs.docker.com/engine/install/rhel/` since repository URLs and steps can change over time.

Save and exit.

---

## Step 2 — Write `tasks/main.yml` to Auto-Include the Correct OS File

```bash
vim roles/docker/tasks/main.yml
```

Type:

```yaml
---
- name: include os specific docker installation
  include_tasks: "install_{{ ansible_distribution | lower }}.yml"

- name: add users to docker group
  user:
    name: "{{ item }}"
    groups: docker
    append: true
  loop: "{{ docker_users }}"

- name: restart docker
  service:
    name: docker
    state: restarted
  when: docker_service_enabled | default(true)

- name: print docker version
  command: docker --version
  register: docker_version

- name: show docker version
  debug:
    var: docker_version.stdout
```

**Explain the key line:**
* `include_tasks: "install_{{ ansible_distribution | lower }}.yml"` → This is the core trick. `ansible_distribution` is the fact gathered automatically (e.g. `Ubuntu`, `RedHat`, `Amazon`). The `| lower` **Jinja2 filter** converts it to lowercase, because Ansible's fact value is capitalized (e.g. `Ubuntu`), but our file names are lowercase (`install_ubuntu.yml`).
* At runtime, Ansible dynamically picks:
  * `ansible_distribution == "Ubuntu"` → includes `install_ubuntu.yml`
  * `ansible_distribution == "RedHat"` → includes `install_redhat.yml`
  * `ansible_distribution == "Amazon"` → includes `install_amazon.yml`
* This means **no manual `when` conditions or loops are needed** — the correct installation file is automatically selected per server, based on its own facts.

> **Tip:** To check the exact value of `ansible_distribution` on your servers before writing filenames, run:
> ```bash
> ansible -i ../hosts.ini servers -m setup | grep -i "\"ansible_distribution\":"
> ```

Save and exit.

---

## Step 3 — Move Common/Shared Tasks to Defaults

```bash
vim roles/docker/defaults/main.yml
```

Type:

```yaml
---
docker_users:
  - ubuntu
  - ec2-user

docker_service_enabled: true
```

**Explain:**
* Values in `defaults/` are the **lowest priority** — they act as a fallback. If the calling playbook overrides `docker_users` or `docker_service_enabled`, that override wins. If not, these defaults are used.
* `docker_service_enabled: true` is used in the `when:` condition in Step 2 to optionally skip the restart task if set to `false`.

Save and exit.

---

## Step 4 — Move the Restart Logic Into a Handler (Best Practice)

```bash
vim roles/docker/handlers/main.yml
```

Type:

```yaml
---
- name: restart docker
  service:
    name: docker
    state: restarted
```

To use this handler properly, update `tasks/main.yml` to **notify** it instead of calling it directly:

```yaml
- name: install docker (os specific)
  include_tasks: "install_{{ ansible_distribution | lower }}.yml"
  notify: restart docker
```

**Explain:** `notify:` tells Ansible to trigger the `restart docker` handler **only if** this task made a change (i.e. Docker was actually installed/updated). Handlers only run once, at the very end of the play — more efficient than restarting on every run.

Save and exit.

---

## Step 5 — Run the Multi-OS Playbook

Using the same playbook from Chapter 9 (`install-docker-with-role.yml`, which simply calls `roles: - docker`):

```bash
cd ~/playbooks
ansible-playbook -i ../hosts.ini install-docker-with-role.yml
```

**Expected result:**
```text
TASK [docker : include os specific docker installation]
included: .../install_ubuntu.yml for worker-ubuntu
included: .../install_redhat.yml for worker-redhat
included: .../install_amazon.yml for worker-amazon
```

Each server automatically runs only the installation file matching its own OS.

---

## Step 6 — Verify on Each Server

```bash
ansible -i ../hosts.ini servers -a "docker --version" --become
```

**Expected result:** Docker version printed for all three servers (Ubuntu, Red Hat, Amazon Linux), confirming successful cross-OS installation.

---

## Troubleshooting

### Error 1 — `could not find included file`
**Reason:** File naming mismatch — e.g. `ansible_distribution` returned `Amazon` but your file is named `install_amazonlinux.yml`.
**Fix:** Run the `setup` module filter command (shown in Step 2's Tip) to get the exact distribution string, and name your file `install_<exact_lowercase_value>.yml`.

### Error 2 — Docker install fails on Red Hat with repo errors
**Reason:** The Docker CE repo URL may have changed, or `dnf-plugins-core` wasn't installed first.
**Fix:** Verify against the official docs (`https://docs.docker.com/engine/install/rhel/`) and confirm task order: remove old packages → install plugins-core → add repo → install docker-ce packages.

### Error 3 — `docker` command not found on Amazon Linux after install
**Reason:** Docker package name differs by Amazon Linux version (Amazon Linux 2 vs Amazon Linux 2023 may need different repo/package names).
**Fix:** Check `cat /etc/os-release` on the target machine and cross-reference with AWS's official Docker installation documentation for that specific Amazon Linux version.

---

## Final Result
By the end of this chapter:
* A single Docker role now correctly installs Docker across **Ubuntu, Red Hat, and Amazon Linux** automatically
* OS detection and file selection happens dynamically via `ansible_distribution | lower`, with zero manual `when` conditions needed for OS branching
* Common tasks (add user to docker group, restart, print version) remain shared across all OS types
* This is the production-grade pattern used by real DevOps engineers to support heterogeneous infrastructure

## What's Next
**Chapter 11** covers using **Terraform** to automatically provision this exact mixed-OS infrastructure (1 control node + 3 workers on different OS) AND automatically generate the Ansible **dynamic inventory file**, so nothing has to be created manually.
