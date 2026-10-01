# Chapter 7 — AWS CLI Setup (Windows)

## Objective
Install AWS CLI v2 on Windows so you can talk to AWS from your terminal.

## Prerequisites
- Windows 10/11 64-bit
- Admin rights to install software

## Official Installation Guide
Search for **"Install or update to the latest version of the AWS CLI"** in the official AWS documentation:
**https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html**

Go to this page and follow the **Windows** section for your version. ⚠️ Verification Required: the download link on that page is the source of truth.

## Step 1 — Download the installer
On the official page, in the Windows section, download the **64-bit MSI installer**.

## Step 2 — Run the installer
1. Double-click the downloaded `.msi` file.
2. Accept the license terms.
3. Keep all default settings and click **Next → Next → Install**.
4. Click **Finish**.

## Step 3 — Verify
Open a **new** Command Prompt and run:

```cmd
aws --version
```

**Expected output (example):**
```text
aws-cli/2.x.x Python/3.x.x Windows/10 exe/AMD64
```

Also try:
```cmd
aws
```
**Expected:** a usage message (this shows the command exists). It will also say you need to provide a command.

## Troubleshooting

### Error 1
```text
'aws' is not recognized as an internal or external command
```
**Reason:** Old terminal window, or install did not finish.
**Fix:** Close all terminals, open a new one. If it still fails, run the MSI installer again.

## Final Result
`aws --version` prints a version. (We will connect it to your AWS account in Chapter 9.)
