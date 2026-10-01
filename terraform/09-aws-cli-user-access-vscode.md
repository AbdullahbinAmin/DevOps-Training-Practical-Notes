# Chapter 9 — AWS CLI User Access Configure on VS Code

## Objective
Create access keys for the IAM user, save them in a `.env` file in VS Code, and prove the AWS CLI can talk to your account. Terraform will use the **same credentials** later.

## Why we need this
Your laptop (local) and your AWS account (cloud) are not linked. The terminal does not know which account to use. Access keys are like a username and password for tools.

## Prerequisites
- Chapter 6 done (IAM user exists)
- Chapter 7 or 8 done (AWS CLI installed)
- VS Code installed (https://code.visualstudio.com)

## Architecture / Flow
```text
IAM user → Access key (ID + Secret) → .env file → source .env → aws cli / terraform
```

## Step 1 — Create a folder and open it in VS Code
1. Create an empty folder named `tf-aws` (for example on Desktop or in your Documents).
2. Open VS Code → **File → Open Folder** → select `tf-aws`.

**Expected:** Explorer is empty and shows `tf-aws`.

## Step 2 — Open the integrated terminal
In VS Code use menu **Terminal → New Terminal**.

**Windows users:** the default terminal may be PowerShell or Command Prompt. The `source .env` command below works in **Git Bash** or a Mac/Linux shell. If you use PowerShell, use the PowerShell version in Step 6.
To open Git Bash in VS Code: click the small **down arrow** next to `+` in the terminal panel → **Git Bash** (needs Git for Windows installed).

## Step 3 — See the problem first
```bash
aws iam list-users
```
**What this does:** asks AWS to list IAM users.

**Expected error:**
```text
Unable to locate credentials. You can configure credentials by running "aws configure".
```
This is expected. It proves the CLI has no credentials yet.

## Step 4 — Create the access key (GUI)

**Step 1:** Sign in to the AWS Console as your IAM user (`tf-user`).

**Step 2:** Search `IAM` → **Users** → click your user.

**Step 3:** Open the **Security credentials** tab.

**Step 4:** Scroll to **Access keys** → click **Create access key**.

**Step 5:** Use case: select **Command Line Interface (CLI)**. Tick the confirmation checkbox at the bottom. Click **Next**.

**Step 6:** Description tag value: `terraform-aws` (any text). Click **Create access key**.

**Step 7:** You now see the **Access key** and **Secret access key**. Click **Show** for the secret. **Copy both values** or click **Download .csv file**.

**Important:** The secret is shown only once. Keep it private.

Note: screen text may differ slightly between console versions. ⚠️ Verification Required if the wording is different.

## Step 5 — Create the `.env` file
In VS Code Explorer click **New File** and name it `.env` (inside `tf-aws`).

Paste this and replace the two values:

```bash
export AWS_ACCESS_KEY_ID="<YOUR_ACCESS_KEY_ID>"
export AWS_SECRET_ACCESS_KEY="<YOUR_SECRET_ACCESS_KEY>"
```

- `<YOUR_ACCESS_KEY_ID>` → the Access key ID you copied (starts with `AKIA...`).
- `<YOUR_SECRET_ACCESS_KEY>` → the Secret access key you copied.

We do not set `AWS_DEFAULT_REGION` here because Terraform will get the region from the `provider` block, and the region can change per project.

Save the file with **Ctrl + S**.

Verify the file exists:
```bash
cat .env
```
**Expected:** the two `export` lines are printed.

## Step 6 — Load the variables into the terminal

**Bash / Git Bash / Mac:**
```bash
source .env
```
**What this does:** sets the two environment variables for the current terminal window.

**PowerShell equivalent (only if you use PowerShell):**
```powershell
$env:AWS_ACCESS_KEY_ID="<YOUR_ACCESS_KEY_ID>"
$env:AWS_SECRET_ACCESS_KEY="<YOUR_SECRET_ACCESS_KEY>"
```

**Important:** These variables are **temporary**. Every time you open a new terminal, run `source .env` again from the `tf-aws` folder. (If you are inside a sub folder, use `source ../.env`.)

## Step 7 — Verify
```bash
aws iam list-users
```
**Expected:** JSON output that lists your users, including `tf-user`. This proves the CLI works with your account.

If the output opens in a pager (a screen with `:` at the bottom), press `q` to exit.

## Step 8 — Protect the secret
If you ever use Git, add this to `.gitignore`:
```text
.env
```
Never share the `.env` file.

## Troubleshooting

### Error 1
```text
Unable to locate credentials
```
**Reason:** You did not run `source .env` in this terminal.
**Fix:** run `source .env` (from the folder that contains `.env`).

### Error 2
```text
An error occurred (InvalidClientTokenId) ... The security token included in the request is invalid
```
**Reason:** Key or secret has a typo, or extra spaces, or quotes are broken.
**Fix:** Open `.env`, re-copy both values carefully, save, run `source .env`.

### Error 3
```text
source: command not found  (Windows Command Prompt)
```
**Reason:** `source` is a Bash command.
**Fix:** use Git Bash, or use the PowerShell lines from Step 6.

## Optional Alternative
You can run `aws configure` and type the key, secret and default region. It saves them in your user profile instead of `.env`. This course uses the `.env` method as the main method.

## Final Result
`aws iam list-users` returns your IAM users from the terminal.
