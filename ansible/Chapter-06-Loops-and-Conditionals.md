# Practical 6 — Loops and Conditionals in Playbooks

## Objective
Learn how to install multiple packages using a `loop`, print custom messages per loop item, and use `when` conditionals to control whether a task runs based on gathered facts (like OS distribution).

## Prerequisites
* OS: Ubuntu (worker nodes)
* Required software: Ansible installed on Control Node
* Required files: `hosts.ini` inventory, `playbooks/install-packages.yml` (from Chapter 5)
* Required ports: SSH (22)
* Required permissions: `become: yes` for package installation

## Pre-Check

```bash
ansible-playbook -i ../hosts.ini install-packages.yml -v
```

**Expected result:** Confirms the `tree` package installs successfully on both workers (from Chapter 5). This confirms your environment is ready before adding loops.

---

## Concept — What is a Loop in Ansible?

Anything you want to do **repeatedly** (like installing many packages one after another) is handled using a **loop**, instead of writing a separate task for each package.

---

## Step 1 — Define a List of Packages as a Variable

Edit `install-packages.yml`:

```bash
vim install-packages.yml
```

Update the file to:

```yaml
---
- name: install packages
  hosts: servers
  become: yes
  vars:
    packages_to_install:
      - tree
      - zip
      - unzip
      - jq
      - wget
      - apache2
  tasks:
    - name: install package
      apt:
        name: "{{ item }}"
        state: present
      loop: "{{ packages_to_install }}"
```

**Explain each part:**
* `packages_to_install:` → A list (array) variable containing all package names to install.
* `loop: "{{ packages_to_install }}"` → Tells Ansible to repeat this task once for every item in the list.
* `"{{ item }}"` → A special Jinja2 variable automatically provided by `loop` — it represents the **current value** being processed in each iteration.

**How a loop works (concept):** If your list has 5 items, the loop runs 5 times. On each run, `item` takes the value of the next element in the list — first `tree`, then `zip`, then `unzip`, and so on.

Save and exit.

---

## Step 2 — Run and Verify the Loop

```bash
ansible-playbook -i ../hosts.ini install-packages.yml
```

**Expected result:** Output shows the `install package` task running once per package, for both worker nodes — `tree`, `zip`, `unzip`, `jq`, `wget`, `apache2` all get installed in sequence.

---

## Step 3 — Print a Message for Each Installed Item (Using `debug`)

You cannot mix `apt` and `debug` inside a single task (they conflict). So we add a **second task**:

```yaml
---
- name: install packages
  hosts: servers
  become: yes
  vars:
    packages_to_install:
      - tree
      - zip
      - unzip
      - jq
      - wget
      - apache2
  tasks:
    - name: install package
      apt:
        name: "{{ item }}"
        state: present
      loop: "{{ packages_to_install }}"

    - name: show message
      debug:
        msg: "Installing {{ item }}"
      loop: "{{ packages_to_install }}"
```

**Explain:**
* `debug:` → A module used purely to print output/messages — useful for logging and troubleshooting.
* `msg:` → The message to print. Here it uses `{{ item }}` again, but this time in the **second task's own loop**, so it correctly shows which package was installed on that server.

Save, exit, and re-run:

```bash
ansible-playbook -i ../hosts.ini install-packages.yml
```

**Expected result:** After all packages install, you'll see debug messages like:
```text
"msg": "Installing tree"
"msg": "Installing zip"
"msg": "Installing unzip"
...
```
for both worker nodes.

---

## Concept — What is a Conditional in Ansible?

A **conditional** lets a task run **only if** a certain condition is true. In Ansible, this is done using the `when` keyword.

---

## Step 4 — Add a `when` Condition Based on OS Distribution

Suppose you only want the `install-packages.yml` playbook to run on **Ubuntu** systems (since some packages/commands only work on Ubuntu).

Edit `install-packages.yml` and add a `when` clause to both tasks:

```yaml
---
- name: install packages
  hosts: servers
  become: yes
  vars:
    packages_to_install:
      - tree
      - zip
      - unzip
      - jq
      - wget
      - apache2
  tasks:
    - name: install package
      apt:
        name: "{{ item }}"
        state: present
      loop: "{{ packages_to_install }}"
      when: ansible_facts['distribution'] == "Ubuntu"

    - name: show message
      debug:
        msg: "Installing {{ item }}"
      loop: "{{ packages_to_install }}"
      when: ansible_facts['distribution'] == "Ubuntu"
```

**Explain:**
* `when:` → The condition that must be `true` for this task to run.
* `ansible_facts['distribution']` → Pulls the `distribution` fact gathered automatically at the start of the play (from Chapter 5's `Gathering Facts` step). **Note:** the correct variable is `ansible_facts`, not `ansible_fact` (common typo that causes a syntax error).
* `== "Ubuntu"` → The condition is true only if the distribution is exactly `Ubuntu`.

> ⚠️ If you see the error: `The conditional check 'ansible_fact['distribution'] == ...' failed`, check your spelling — it must be `ansible_facts` (plural), not `ansible_fact`.

Save, exit, and run:

```bash
ansible-playbook -i ../hosts.ini install-packages.yml
```

**Expected result:** Since both worker nodes are Ubuntu, the tasks run normally (not skipped).

---

## Step 5 — Test the Condition Failing (For Understanding)

Temporarily change the condition to test a non-matching distribution:

```yaml
when: ansible_facts['distribution'] == "RedHat"
```

Run again:

```bash
ansible-playbook -i ../hosts.ini install-packages.yml
```

**Expected result:**
```text
TASK [install package] skipping: [worker-node-1]
TASK [install package] skipping: [worker-node-2]
```

**Why this happened:** The actual `ansible_facts['distribution']` value on these servers is `Ubuntu`, not `RedHat`, so the condition evaluates to `false` and Ansible **skips** the task entirely — it does not throw an error, it simply does nothing.

Change the condition back to `"Ubuntu"` before continuing to the next chapter.

---

## Troubleshooting

### Error 1 — `The conditional check '...' failed`
**Reason:** Typo — using `ansible_fact` instead of `ansible_facts`.
**Fix:** Always use `ansible_facts['distribution']` (plural "facts").

### Error 2 — Task fails saying "conflicting action statements: apt and debug"
**Reason:** Trying to use two modules (`apt` and `debug`) in the same single task.
**Fix:** Split into two separate tasks, each with their own `loop`, as shown in Step 3.

### Error 3 — All tasks silently skipped, no error shown
**Reason:** Your `when` condition doesn't match the actual server's facts.
**Fix:** Run `ansible -i ../hosts.ini servers -m setup | grep -i distribution` to confirm the real value, then match your `when` condition exactly (case-sensitive).

---

## Final Result
By the end of this chapter you can:
* Install a list of multiple packages using `loop` and `{{ item }}`
* Print custom debug messages for each loop iteration
* Use `when` conditionals to control task execution based on OS facts (`ansible_facts['distribution']`)
* Understand that failed conditions cause tasks to be **skipped**, not errored

## What's Next
**Chapter 7** covers **Ansible Vault** — encrypting sensitive variables (passwords, API keys) so secrets aren't stored in plain text.
