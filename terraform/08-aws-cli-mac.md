# Chapter 8 — AWS CLI Setup (Mac)

## Objective
Install AWS CLI v2 on macOS.

## Prerequisites
- macOS
- Admin password for `sudo`

## Official Installation Guide
**https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html** → choose the **macOS** section.

Go to this page and follow the macOS instructions. The GUI installer (`.pkg`) is the easiest for beginners. ⚠️ Verification Required: confirm the current download link on the page.

## Option used for notes (command line version from the official page)

### Step 1 — Download the package
```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
```
**What this does:** downloads the installer into the current folder.

### Step 2 — Install it
```bash
sudo installer -pkg AWSCLIV2.pkg -target /
```
**What this does:** installs AWS CLI for all users. It asks for your Mac password.

### Step 3 — Verify
```bash
aws --version
```
**Expected (example):**
```text
aws-cli/2.x.x Python/3.x.x Darwin/xx exe/arm64
```

Running just `aws` shows a usage message. That also proves the command works.

## Troubleshooting

### Error 1
```text
zsh: command not found: aws
```
**Reason:** Install did not finish or old terminal.
**Fix:** Run Step 2 again, then open a new Terminal window.

## Final Result
`aws --version` works.
