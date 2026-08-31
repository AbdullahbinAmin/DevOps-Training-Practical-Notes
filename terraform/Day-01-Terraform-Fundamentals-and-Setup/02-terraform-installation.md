# Day 01 — Terraform Installation

> Download the Terraform binary, put it on your system PATH, and confirm it works with `terraform version` — on Windows, Linux, or macOS.

## Learning Objectives
- Download the correct Terraform build for your operating system and architecture.
- Install Terraform on Windows by extracting the zip and editing the system PATH.
- Install Terraform on Linux/macOS using a package manager or a manual binary copy.
- Verify the installation and read the version output correctly.
- Enable shell tab-completion for the `terraform` command.

## Prerequisites
- Administrator (Windows) or `sudo` (Linux/macOS) rights on your machine.
- Internet access to `developer.hashicorp.com`.
- A terminal: Command Prompt or PowerShell on Windows, bash/zsh elsewhere.

## Concept

Terraform ships as a **single self-contained executable**. There is no service to run and nothing to configure at install time. "Installing" Terraform means only two things:

1. Put the `terraform` binary somewhere on disk.
2. Make sure your shell can find it, by adding that folder to the `PATH` environment variable.

`PATH` is the list of folders your shell searches when you type a command name. If `terraform` is not on the PATH you get "command not found" even though the file exists.

| OS | Download | Typical install location |
|---|---|---|
| Windows | `terraform_1.x.x_windows_amd64.zip` | `C:\Program Files\Terraform\` |
| Linux | `terraform_1.x.x_linux_amd64.zip` or apt/yum repo | `/usr/local/bin/terraform` |
| macOS | Homebrew, or `darwin_amd64` / `darwin_arm64` zip | `/usr/local/bin/terraform` |

Pick `arm64` instead of `amd64` on Apple Silicon Macs and ARM Linux (for example AWS Graviton).

## Step-by-Step Practical

### Part A — Windows

1. Open the official downloads page in your browser.

```
https://developer.hashicorp.com/terraform/downloads
```

2. Choose your platform: **Windows → AMD64 (64-bit)** and download the zip file.

3. Extract the downloaded zip (right click → **Extract All...**). Inside you get two files.

```
terraform_1.x.x_windows_amd64/
  terraform.exe    <- the CLI
  LICENSE.txt
```

4. Create a permanent home for the binary and move `terraform.exe` into it.

```powershell
New-Item -ItemType Directory -Force -Path "C:\Program Files\Terraform"
Move-Item "$HOME\Downloads\terraform_1.*_windows_amd64\terraform.exe" "C:\Program Files\Terraform\"
```

5. Add that folder to the **system** PATH using the GUI:
   - Press `Win + S` and search for **"Edit the system environment variables"**.
   - Click **Environment Variables**.
   - Under **System variables**, select **Path** → **Edit** → **New**.
   - Add the folder where `terraform.exe` lives, for example `C:\Program Files\Terraform\`.
   - Click **OK** on all windows.

   Or do the same from an **Administrator** PowerShell:

```powershell
[Environment]::SetEnvironmentVariable(
  "Path",
  [Environment]::GetEnvironmentVariable("Path", "Machine") + ";C:\Program Files\Terraform",
  "Machine"
)
```

6. Close every open terminal and open a **new** Command Prompt (PATH changes only apply to new shells), then verify.

```powershell
terraform version
```

7. Optional — install via a package manager instead of steps 2-6.

```powershell
choco install terraform
# or
winget install --id HashiCorp.Terraform -e
```

### Part B — Linux (Debian/Ubuntu, official HashiCorp repo)

8. Install the prerequisites and add HashiCorp's signing key and repository.

```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl
```

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

```bash
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
```

```bash
sudo apt-get update && sudo apt-get install -y terraform
```

9. On RHEL/CentOS/Amazon Linux use `yum`/`dnf` instead.

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum install -y terraform
```

10. Manual binary install on any Linux (works when you cannot add a repo). Replace `1.9.8` with the current version shown on the downloads page.

```bash
TF_VERSION=1.9.8
curl -fsSLO "https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_amd64.zip"
unzip "terraform_${TF_VERSION}_linux_amd64.zip"
sudo install -o root -g root -m 0755 terraform /usr/local/bin/terraform
rm -f terraform "terraform_${TF_VERSION}_linux_amd64.zip"
```

### Part C — macOS

11. Homebrew is the simplest route.

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

12. Or install the binary manually — use `darwin_arm64` on Apple Silicon, `darwin_amd64` on Intel.

```bash
TF_VERSION=1.9.8
curl -fsSLO "https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_darwin_arm64.zip"
unzip "terraform_${TF_VERSION}_darwin_arm64.zip"
sudo mv terraform /usr/local/bin/
```

### Part D — Finish up (all platforms)

13. Enable tab completion for the `terraform` command (bash/zsh).

```bash
terraform -install-autocomplete
```

14. Confirm the install and see the available commands.

```bash
terraform version && terraform -help
```

## Expected Output

```
C:\> terraform version
Terraform v1.6.6
on windows_amd64

Your version of Terraform is up to date!
```

On Linux/macOS the platform line changes but the shape is identical:

```
$ terraform version
Terraform v1.9.8
on linux_amd64
```

If you see a version number, Terraform is successfully installed and working.

## Verification

1. `terraform version` prints a version and platform — not an error.
2. The binary is found on the PATH, not only in your Downloads folder.

```bash
# Linux/macOS
which terraform
```

```powershell
# Windows
where.exe terraform
```

3. A subcommand runs without a crash.

```bash
terraform -help plan
```

## Cleanup

Nothing was created in the cloud, so there is no billing cleanup. To uninstall Terraform later:

```powershell
# Windows
Remove-Item "C:\Program Files\Terraform\terraform.exe"
# then remove C:\Program Files\Terraform from the system Path variable
```

```bash
# Linux/macOS
sudo rm -f /usr/local/bin/terraform
# or, if installed via a package manager
sudo apt-get remove -y terraform   # brew uninstall terraform
```

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `'terraform' is not recognized as an internal or external command` | Folder not on PATH, or terminal opened before the PATH edit | Re-check the Path entry, then open a brand-new terminal |
| `terraform: command not found` (Linux/macOS) | Binary not in `/usr/local/bin` or not executable | `sudo install -m 0755 terraform /usr/local/bin/` |
| `Access is denied` when moving `terraform.exe` | `C:\Program Files` needs elevation | Run PowerShell as Administrator, or install to `C:\Terraform\` |
| `cannot execute binary file: Exec format error` | Downloaded the wrong architecture (amd64 vs arm64) | Re-download the build matching `uname -m` |
| `zsh: killed terraform` on macOS | Gatekeeper quarantined the manual download | `xattr -d com.apple.quarantine /usr/local/bin/terraform` |
| Old version still reported after upgrade | A second `terraform` earlier on the PATH | `where.exe terraform` / `which -a terraform`, delete the stale copy |

## Key Takeaways
- Terraform is one binary; installation is just "place it and PATH it".
- PATH changes only take effect in newly opened terminals.
- Package managers (apt, yum, brew, choco, winget) make upgrades far easier than manual zips.
- Match the architecture: `amd64` for most PCs, `arm64` for Apple Silicon and ARM servers.
- `terraform version` is your single proof that the install worked.

## Next: [AWS User Setup](03-aws-user-setup.md)
