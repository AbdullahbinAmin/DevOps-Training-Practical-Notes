# Practical 7 — Ansible Vault (Encrypting Secrets)

## Objective
Store secrets (username, password, API key) in a separate variables file, import them into a playbook, understand why plain-text secrets are dangerous, then encrypt the secrets file using Ansible Vault so it cannot be read without a password.

## Prerequisites
* OS: Ubuntu (worker nodes)
* Required software: Ansible installed
* Required files: `hosts.ini` inventory file
* Required ports: SSH (22)
* Required permissions: `chmod 600` on the vault password file (owner read/write only)

## Pre-Check

```bash
ansible -i ../hosts.ini servers -m ping
```

**Expected result:** Both nodes respond with `pong`, confirming connectivity before proceeding.

---

## Directory / File Structure

```text
playbooks/
├── hosts.ini            (already exists one level up, referenced via ../hosts.ini)
├── secrets.yml           (holds the actual secret values)
├── show-secrets.yml       (playbook that imports and prints the secrets)
└── vault-password.txt     (the password used to encrypt/decrypt secrets.yml)
```

---

## Step 1 — Create a Secrets Variables File

```bash
vim secrets.yml
```

Type the complete file:

```yaml
---
username: shubham
password: test@123
api_key: qwerty@111
```

**What this holds:** Plain variable definitions — a username, a password, and an API key. These are example secret values.

Save and exit.

---

## Step 2 — Create a Playbook That Imports and Prints the Secrets

```bash
vim show-secrets.yml
```

Type the complete file:

```yaml
---
- name: show secrets
  hosts: servers
  become: yes
  vars_files:
    - secrets.yml
  tasks:
    - name: show password
      debug:
        msg: "My password is {{ password }}"

    - name: show api key
      debug:
        msg: "My API key is {{ api_key }}"
```

**Explain:**
* `vars_files:` → A list of external files to load variables from. Note it must be a **list** (with a `-` prefix), even if you only have one file — Ansible will error with "vars_files is not a valid list" if you write it as a plain string.
* `secrets.yml` → The file created in Step 1, whose variables (`username`, `password`, `api_key`) are now available to use in this playbook.
* `{{ password }}`, `{{ api_key }}` → Jinja2 templating pulls in the values directly from `secrets.yml`.

Save and exit.

---

## Step 3 — Run the Playbook (See the Problem)

```bash
ansible-playbook -i ../hosts.ini show-secrets.yml
```

**Expected result:**
```text
"msg": "My password is test@123"
"msg": "My API key is qwerty@111"
```

**⚠️ Problem:** Your password and API key are now visible in plain text in the terminal output and in any logs. This is a **security risk** — nobody wants secrets exposed like this.

---

## Step 4 — Create a Vault Password File

To encrypt `secrets.yml`, we first need a password that will be used for encryption/decryption.

```bash
vim vault-password.txt
```

Type a password of your choice (example):

```text
jethalal
```

Save and exit.

**Secure this file immediately:**

```bash
chmod 600 vault-password.txt
```

**What this does:** Restricts the file so only your user can read/write it — no one else can view your vault password. This file itself should never be committed to any public repository.

---

## Step 5 — Encrypt `secrets.yml` Using Ansible Vault

```bash
ansible-vault encrypt secrets.yml --vault-password-file vault-password.txt
```

**What this does:**
* `ansible-vault encrypt` → The Ansible Vault command used to encrypt a file.
* `secrets.yml` → The file to encrypt.
* `--vault-password-file vault-password.txt` → Tells Ansible Vault which password file to use for the encryption key (instead of typing the password interactively each time).

**Expected result:**
```text
Encryption successful
```

If you now open `secrets.yml` with `cat secrets.yml`, you'll see scrambled, unreadable encrypted content instead of the original plain text.

---

## Step 6 — Try Running the Playbook Without the Vault Password (See It Fail)

```bash
ansible-playbook -i ../hosts.ini show-secrets.yml
```

**Expected result:**
```text
ERROR! Attempting to decrypt but no vault secrets found
```

**Why:** `secrets.yml` is now encrypted, and Ansible cannot read the variables inside it without the vault password.

---

## Step 7 — Run the Playbook Correctly, Providing the Vault Password File

```bash
ansible-playbook -i ../hosts.ini show-secrets.yml --vault-password-file vault-password.txt
```

**Expected result:** The playbook runs successfully and decrypts `secrets.yml` on the fly, printing:
```text
"msg": "My password is test@123"
"msg": "My API key is qwerty@111"
```

**Key takeaway:** Anyone without `vault-password.txt` **cannot** run this playbook or view the secret values — the file is protected.

---

## Step 8 — Hide the Secret Values From Terminal Output (`no_log`)

Even though the file is encrypted at rest, when the playbook runs, the decrypted values are still printed on screen. To prevent that:

```bash
vim show-secrets.yml
```

Add `no_log: true` to the tasks handling secrets:

```yaml
---
- name: show secrets
  hosts: servers
  become: yes
  vars_files:
    - secrets.yml
  tasks:
    - name: show password
      debug:
        msg: "My password is {{ password }}"
      no_log: true

    - name: show api key
      debug:
        msg: "My API key is {{ api_key }}"
      no_log: true
```

**Explain:**
* `no_log: true` → The task will still execute (and decrypt/use the secret), but its output/logs will be hidden — you will simply see `"censored"` or no output for that task.

> **Rule of thumb:** Whenever a task involves passwords, credentials, or security-sensitive information, you should **always** add `no_log: true` to that task.

Save, exit, and re-run:

```bash
ansible-playbook -i ../hosts.ini show-secrets.yml --vault-password-file vault-password.txt
```

**Expected result:** The playbook completes successfully (`ok`/`changed`), but the actual secret values are **not printed** anywhere in the output.

---

## Troubleshooting

### Error 1 — `vars_files is not a valid list`
**Reason:** `vars_files` was written as a plain string instead of a YAML list.
**Fix:** Always write it as:
```yaml
vars_files:
  - secrets.yml
```

### Error 2 — `ERROR! Attempting to decrypt but no vault secrets found`
**Reason:** Forgot to pass `--vault-password-file` when running an encrypted playbook.
**Fix:** Always include `--vault-password-file vault-password.txt` when running playbooks that reference encrypted files.

### Error 3 — Secrets still showing in terminal despite encryption
**Reason:** The file is encrypted at rest, but the task printing the value did not have `no_log: true`.
**Fix:** Add `no_log: true` to every task that touches or prints secret values.

---

## Final Result
By the end of this chapter:
* `secrets.yml` is encrypted using Ansible Vault and cannot be read without `vault-password.txt`
* The playbook `show-secrets.yml` correctly decrypts and uses the secrets only when given the correct vault password file
* `no_log: true` ensures secret values are never printed in terminal output or logs

## What's Next
**Chapter 8** covers **File Operations and Service Management** — copying a custom file to a server, installing and configuring Nginx, restarting/enabling services, all through a playbook.
