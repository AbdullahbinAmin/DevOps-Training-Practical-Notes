# Practical 9 — Ansible Roles (Building a Reusable Docker Role)

## Objective
Understand what an Ansible Role is and why it's used, create a Docker role using `ansible-galaxy`, fill in its tasks/handlers/defaults, and call the role from a playbook.

## Prerequisites
* OS: Ubuntu (worker nodes)
* Required software: Ansible installed on Control Node
* Required files: `hosts.ini` inventory file
* Required ports: SSH (22)
* Required permissions: `become: yes`

## Pre-Check

```bash
ansible -i ../hosts.ini servers -m ping
```

**Expected result:** Both worker nodes respond `pong`.

---

## Concept — What is a Role?

A **Role** is a **reusable template** for playbooks. Despite the name, it has **nothing to do with role-based access control (RBAC) or user permissions** — it's simply a standard, reusable folder structure for organizing your automation so you don't repeat yourself.

Without roles, every playbook you've written so far (`hello.yml`, `install-packages.yml`, `setup-nginx.yml`) only works for the exact hosts/logic hardcoded inside that one file. To reuse the same logic elsewhere, you'd have to copy and edit the whole file again. **Roles solve this** by making your automation logic truly reusable.

**Role folder structure (auto-generated):**

```text
roles/
└── docker/
    ├── defaults/    (default variables — lowest priority)
    ├── files/       (static files to copy, e.g. index.html)
    ├── handlers/    (tasks triggered only when notified, e.g. restart service)
    ├── meta/        (metadata about the role)
    ├── tasks/       (the main task list — main.yml)
    ├── templates/   (Jinja2 templates)
    ├── tests/       (test inventory files)
    └── vars/        (variables — higher priority than defaults)
```

---

## Step 1 — Create a Role Using Ansible Galaxy

**Concept — What is Ansible Galaxy?**
Just like `pip` is the package manager for Python packages, **Ansible Galaxy** is the package manager/tool for Ansible roles. `ansible-galaxy init` is used to scaffold (auto-generate) the standard role folder structure.

```bash
ansible-galaxy init roles/docker
```

**What this does:** Creates a complete role skeleton named `docker` inside a `roles/` folder, with all the standard subfolders (`tasks`, `handlers`, `defaults`, `vars`, `files`, `templates`, `meta`, `tests`) already created for you.

**Expected result:**
```text
Role docker was created successfully
```

**Verify the structure:**

```bash
sudo apt-get install tree   # if 'tree' is not already installed
tree roles/docker
```

**Expected result:** Shows the full folder tree exactly as listed above.

---

## Step 2 — Write the Main Tasks (`tasks/main.yml`)

```bash
vim roles/docker/tasks/main.yml
```

This file is normally empty by default — it's a template that expects you to write **only tasks** here (no `hosts:`, no `become:` — those live in the playbook that calls the role).

Type the complete file:

```yaml
---
- name: update system
  apt:
    update_cache: yes

- name: install docker
  apt:
    name: docker.io
    state: latest

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

- name: print docker version
  command: docker --version
  register: docker_version

- name: show docker version
  debug:
    var: docker_version.stdout
```

**Explain each task:**
* **update system** → `apt: update_cache: yes` is the module way of writing `sudo apt-get update`.
* **install docker** → Installs the `docker.io` package with `state: latest`.
* **add users to docker group** → Uses the `user` module with `groups: docker` and `append: true` (append, so we don't overwrite the user's other existing groups). `loop: "{{ docker_users }}"` allows adding **multiple users** at once — `docker_users` is a variable (a list) we will define in Step 3.
* **restart docker** → Restarts the Docker service so group membership and installation take effect.
* **print docker version / register** → `register: docker_version` captures the output of the `docker --version` command into a variable, which is then displayed with `debug: var: docker_version.stdout`.

Save and exit.

---

## Step 3 — Define the `docker_users` Variable (`vars/main.yml`)

```bash
vim roles/docker/vars/main.yml
```

Type:

```yaml
---
docker_users:
  - ubuntu
  - ec2-user
```

**Explain:** This defines the list of usernames that will be looped over and added to the `docker` group. Adjust these usernames based on which users actually exist on your target machines (e.g. `ubuntu` for Ubuntu AMIs).

Save and exit.

---

## Step 4 — Use `handlers` for Service Management (Best Practice)

Instead of directly restarting Docker inside `tasks/main.yml`, best practice is to use a **handler** — a task that only runs when explicitly notified by another task.

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

**Why use handlers:** Handlers only run **once**, at the end of the play, and only if notified — even if multiple tasks would have triggered the same restart, it happens just once. This is more efficient than restarting a service after every single change.

> For this practical, it is acceptable to keep the restart task directly inside `tasks/main.yml` (simpler for beginners) — using handlers with `notify:` is the recommended best practice for production code.

Save and exit.

---

## Step 5 — Create a Playbook That Uses the Role

```bash
cd playbooks
vim install-docker-with-role.yml
```

Type the complete file:

```yaml
---
- name: install docker on ubuntu
  hosts: servers
  become: yes
  roles:
    - docker
```

**Explain:**
* `roles:` → A list of role names to run for this play.
* `- docker` → The name of the role folder (`roles/docker`) to execute. This one line runs **everything** defined inside that role's `tasks/main.yml`.

Save and exit.

---

## Step 6 — Make Sure the `roles` Folder is in the Right Location

Ansible looks for roles relative to the playbook's location, inside a folder literally named `roles` (not `role`).

```bash
ls ~/roles
```

If your `roles/docker` folder is one directory above the `playbooks` folder, move it so it sits next to (or is discoverable from) your `playbooks` folder:

```bash
mv ~/roles ~/playbooks/roles
```

**Why:** If Ansible reports `the role 'docker' was not found`, it usually means the `roles/` folder isn't in a location Ansible searches by default (same directory as the playbook, or a `roles_path` set in `ansible.cfg`).

---

## Step 7 — Run the Playbook

```bash
cd ~/playbooks
ansible-playbook -i ../hosts.ini install-docker-with-role.yml
```

**Expected result:**
```text
PLAY [install docker on ubuntu]
TASK [Gathering Facts] ... ok
TASK [docker : update system] ... changed
TASK [docker : install docker] ... changed
TASK [docker : add users to docker group] ... changed
TASK [docker : restart docker] ... changed
TASK [docker : print docker version] ... changed
TASK [docker : show docker version] ... ok
PLAY RECAP ... changed=5 ok=6
```

---

## Step 8 — Verify Docker Installed Correctly

SSH into a worker node and run:

```bash
docker --version
docker ps
```

**Expected result:** Shows the installed Docker version, and `docker ps` shows the empty running-containers table (headers: CONTAINER ID, IMAGE, COMMAND, CREATED, STATUS, PORTS, NAMES) with no errors.

> If you see a "permission denied" error running `docker ps` as a normal user, run `newgrp docker` in that session, or simply log out and log back in for the group membership to take effect.

---

## Troubleshooting

### Error 1 — `the role 'docker' was not found`
**Reason:** The `roles/` folder is not in a location Ansible searches by default (must typically be named exactly `roles`, sitting next to the playbook).
**Fix:** Move/rename the folder so the structure is `playbooks/roles/docker/...` and re-run from inside `playbooks/`.

### Error 2 — `Apeend is set, but no groups are specified`
**Reason:** Typo — using `group:` (singular) instead of `groups:` (plural) in the `user` module.
**Fix:** Use `groups: docker` (plural key name), even when adding to just one group.

### Error 3 — `docker ps` fails with "permission denied"
**Reason:** The current shell session hasn't picked up the new `docker` group membership yet.
**Fix:** Run `newgrp docker`, or log out and log back in.

---

## Final Result
By the end of this chapter:
* You have a fully reusable `docker` role created via `ansible-galaxy init`
* The role installs Docker, adds specified users to the `docker` group, restarts the service, and prints the Docker version
* A one-line playbook (`roles: - docker`) can now install Docker on **any** number of servers, without rewriting any logic

## What's Next
**Chapter 10** extends this role to handle **multiple different operating systems** (Ubuntu, Red Hat, Amazon Linux) using `include_tasks` and OS-specific task files — a real production-style Docker role.
