# Day 01 — AWS CLI User Access Config on VS Code

> Create an access key for your IAM user, keep it in a git-ignored `.env` file, load it into your VS Code terminal as environment variables, and prove the connection with `aws iam list-users`.

## Learning Objectives
- Explain why a local terminal needs credentials before it can reach AWS.
- Create a CLI access key for an IAM user and store it safely.
- Use the `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` environment variables.
- Load a `.env` file into a terminal session on both bash and PowerShell.
- Protect credentials from git with `.gitignore` before the first commit.

## Prerequisites
- Completed [AWS User Setup](03-aws-user-setup.md) — IAM user `tf-user` exists.
- Completed [AWS CLI Setup](04-aws-cli-setup.md) — `aws --version` works.
- VS Code installed, with a project folder such as `TF-AWS` open.

## Concept

Your local machine has no idea which AWS account to talk to. Every AWS API call must be signed with an **Access Key ID** and a **Secret Access Key**, so you must supply them.

```
Local (VS Code / Terminal)  ---- needs credentials ---->  AWS Cloud (your account)
        aws cli
```

Environment variables sit at the **top** of the AWS credential chain, so they override anything in `~/.aws/credentials`. That makes them ideal for per-project switching.

| Variable | Meaning |
|---|---|
| `AWS_ACCESS_KEY_ID` | Your Access Key ID (public part, starts with `AKIA...`) |
| `AWS_SECRET_ACCESS_KEY` | Your Secret Access Key (private — shown only once) |
| `AWS_DEFAULT_REGION` | Optional default region, e.g. `ap-south-1` |

A `.env` file keeps those values out of your shell history and out of your `.tf` files. It must **never** be committed. Note the exact spelling `AWS_SECRET_ACCESS_KEY` — a typo here is the single most common cause of "Unable to locate credentials".

## Step-by-Step Practical

### Part A — Create the access key

1. In the AWS Console go to **IAM → Users → tf-user → Security credentials → Access keys → Create access key**.

2. Choose the use case **Command Line Interface (CLI)**, acknowledge the recommendation warning, and click **Next**, then **Create access key**.

3. On the success screen, **Download .csv** or copy both values immediately.

```
Access Key ID     : YOUR_ACCESS_KEY        (looks like AKIA************)
Secret Access Key : YOUR_SECRET_KEY        (shown only once — you will not see it again)
```

### Part B — Store the key in a .env file

4. Open your project folder in VS Code and create the ignore rule **before** the `.env` file. Create `.gitignore`:

```bash
# .gitignore
.env
*.env
.aws/
credentials

# Terraform
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
crash.log
```

5. Now create `.env` in the project root (`TF-AWS/.env`) and replace the placeholders with your own key values.

```bash
AWS_ACCESS_KEY_ID=YOUR_ACCESS_KEY
AWS_SECRET_ACCESS_KEY=YOUR_SECRET_KEY
AWS_DEFAULT_REGION=ap-south-1
```

6. Commit a placeholder template instead, so teammates know what is required — `.env.example`:

```bash
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=ap-south-1
```

### Part C — Load the variables into your terminal

7. In VS Code open the integrated terminal (`Ctrl + \``) and `cd` into the project.

```bash
cd ~/TF-AWS
```

8. **bash / zsh / Git Bash** — export every variable in the file.

```bash
set -a && source .env && set +a
```

9. **PowerShell** — `source` does not exist, so parse the file instead.

```powershell
Get-Content .env | Where-Object { $_ -match '^\s*[^#].*=' } | ForEach-Object {
  $name, $value = $_ -split '=', 2
  [Environment]::SetEnvironmentVariable($name.Trim(), $value.Trim(), "Process")
}
```

10. **Command Prompt (cmd)** — use a `for` loop.

```powershell
for /F "usebackq tokens=1,2 delims==" %i in (".env") do set %i=%j
```

11. Confirm the variables are present in this session. Print only the **non-secret** one.

```bash
echo "$AWS_ACCESS_KEY_ID"
```

```powershell
$env:AWS_ACCESS_KEY_ID
```

Never `echo` the secret key — terminal output ends up in logs, screen shares, and recordings.

### Part D — Test the connection

12. List the IAM users in the account.

```bash
aws iam list-users
```

13. Confirm which identity you are actually using.

```bash
aws sts get-caller-identity
```

14. Confirm Terraform picks up the same environment variables. Create `main.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# No credentials here on purpose — the provider reads them from the environment.
provider "aws" {
  region = "ap-south-1"
}

data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}
```

15. Initialise and read the value back through Terraform.

```bash
terraform init && terraform plan
```

Remember: environment variables live only in the current terminal session. **If you open a new terminal, load `.env` again.**

## Expected Output

```json
PS C:\TF-AWS> aws iam list-users
{
    "Users": [
        { "UserName": "tf-user",  "UserId": "AIDA....1111", "Arn": "arn:aws:iam::YOUR_ACCOUNT_ID:user/tf-user" },
        { "UserName": "han-user", "UserId": "AIDA....2222", "Arn": "arn:aws:iam::YOUR_ACCOUNT_ID:user/han-user" }
    ]
}
```

JSON like this means AWS CLI is working from VS Code.

```
$ terraform plan
Changes to Outputs:
  + account_id = "YOUR_ACCOUNT_ID"

You can apply this plan to save these new output values...
```

## Verification

1. `aws sts get-caller-identity` returns the `tf-user` ARN.
2. `aws iam list-users` returns JSON, not an `AccessDenied` or credentials error.
3. `.env` is ignored by git — this must print `.env`:

```bash
git check-ignore -v .env
```

4. Nothing sensitive is staged.

```bash
git status --short
```

5. `terraform plan` resolves your account ID without any credentials written in `.tf` files.

## Cleanup

1. Unset the variables when you are done with the session.

```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_DEFAULT_REGION
```

```powershell
Remove-Item Env:AWS_ACCESS_KEY_ID, Env:AWS_SECRET_ACCESS_KEY, Env:AWS_DEFAULT_REGION
```

2. Remove the Terraform working directory created by `init` (no cloud resources were created, so no `destroy` is needed).

```bash
rm -rf .terraform .terraform.lock.hcl
```

3. If a key was ever exposed — pasted in chat, committed, screenshotted — deactivate and delete it in **IAM → tf-user → Security credentials**, then create a new one.

```bash
aws iam delete-access-key --user-name tf-user --access-key-id YOUR_ACCESS_KEY
```

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `Unable to locate credentials` | `.env` not loaded, or loaded in a different terminal | Re-run `set -a && source .env && set +a` in the terminal you are using |
| `source: command not found` in PowerShell | `source` is a bash builtin | Use the PowerShell loop from step 9, or switch the terminal to Git Bash |
| `InvalidClientTokenId` | Key mistyped, deactivated, or deleted | Recreate the access key and update `.env` |
| `SignatureDoesNotMatch` | Secret key has a stray space, quote, or trailing newline | Re-copy the secret; do not wrap it in quotes |
| Variables vanish in a new terminal | Environment variables are per-session | Load `.env` again, or use `aws configure --profile` for persistence |
| `AMS_SECRET_ACCESS_KEY` has no effect | Typo — must be `AWS_SECRET_ACCESS_KEY` | Fix the spelling in `.env` and reload |
| `AccessDenied` on `iam:ListUsers` | IAM policy lacks the permission | Verify `tf-user` is in the `admins` group |
| `.env` appears in `git status` | `.gitignore` added after the file was staged | `git rm --cached .env`, then commit the `.gitignore` |
| Old credentials keep being used | `~/.aws/credentials` or a stale env var is shadowing | `aws configure list` shows which source won |

## Key Takeaways
- AWS API calls need credentials; environment variables are the highest-priority source.
- A secret access key is displayed exactly once — save it immediately or recreate the key.
- `.env` plus `.gitignore` keeps secrets out of git; write the `.gitignore` first.
- Environment variables are per-terminal-session; reload them in every new terminal.
- Terraform reads the same environment variables, so never hardcode keys in `.tf` files.
- Treat any exposed key as compromised: deactivate, delete, rotate.

## Next: [Day 02 — Creating an EC2 Instance (Manual vs Terraform)](../Day-02-EC2-Manual-vs-Terraform/01-ec2-manual-vs-terraform.md)
