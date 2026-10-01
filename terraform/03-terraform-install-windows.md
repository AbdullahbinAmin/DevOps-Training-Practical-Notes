# Chapter 3 — Terraform Installation (Windows)

## Objective
Install Terraform on Windows and confirm it works from Command Prompt.

## Prerequisites
- OS: Windows 10 or 11 (64-bit)
- Permissions: you can create a folder and edit Environment Variables
- Internet access

## Official Installation Guide
**https://developer.hashicorp.com/terraform/install**

Open this page and choose **Windows**. ⚠️ Verification Required: the page layout can change; pick the **AMD64** (64-bit) binary download.

## Step 1 — Download the binary
1. Open the official page above.
2. Under **Windows**, click the **AMD64** download (a `.zip` file).

**Expected:** a file like `terraform_X.X.X_windows_amd64.zip` in your Downloads folder.

## Step 2 — Extract the zip
1. Right-click the zip file → **Extract All…** → **Extract**.
2. Inside the folder you will see `terraform.exe` and a `LICENSE` file (newer versions may also include a few extra text files).

## Step 3 — Put `terraform.exe` in a permanent folder
1. Create a folder, for example `C:\terraform`.
2. Copy `terraform.exe` into `C:\terraform`.

**Why:** we need a stable folder to add to PATH. You can use another path (for example under `C:\Program Files`) if you prefer.

## Step 4 — Add the folder to PATH
1. Press the Windows key and search **Edit the system environment variables**. Open it.
2. Click **Environment Variables…**
3. Under **User variables** (or System variables), select **Path** → **Edit**.
4. Click **New** and type: `C:\terraform`
5. Click **OK → OK → OK** to close all windows.

**Why:** PATH tells Windows where to find `terraform.exe` so you can type `terraform` from any folder.

## Step 5 — Verify
Close every old Command Prompt window. Open a **new** Command Prompt (`cmd`) and run:

```cmd
terraform version
```

**What this does:** prints the installed Terraform version.

**Expected output (example, your version will differ):**
```text
Terraform v1.x.x
on windows_amd64
```

## Troubleshooting

### Error 1
```text
'terraform' is not recognized as an internal or external command
```
**Reason:** PATH was not set, or you are using an old terminal window.
**Fix:** Re-check Step 4 (folder path must contain `terraform.exe`). Close all terminals, open a new one, run `terraform version` again.

## Final Result
`terraform version` prints a version number in a new Command Prompt.
