# Day 01 — AWS CLI Setup

> Install AWS CLI v2 on Windows, Linux, or macOS, verify it with `aws --version`, and configure a profile with `aws configure`.

## Learning Objectives
- Install AWS CLI v2 using the official installer for your operating system.
- Verify the installation and read the `aws --version` output.
- Configure credentials, default region, and output format with `aws configure`.
- Locate and understand the `credentials` and `config` files AWS CLI writes.
- Test connectivity to AWS without exposing your secret key.

## Prerequisites
- Completed [AWS User Setup](03-aws-user-setup.md) — you have an IAM user such as `tf-user`.
- Administrator / `sudo` rights to run an installer.
- An access key pair for that IAM user. If you do not have one yet, note 05 covers creating it; you can install now and configure later.

## Concept

Terraform talks to AWS over the same HTTPS API that the AWS CLI uses, and it reads credentials from the **same places** the CLI does. So configuring the CLI properly means Terraform is configured too.

AWS CLI looks for credentials in this order (first match wins):

| Priority | Source |
|---|---|
| 1 | Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) |
| 2 | Shared credentials file (`~/.aws/credentials`) |
| 3 | Shared config file (`~/.aws/config`) |
| 4 | IAM role attached to an EC2 instance / ECS task |

Always use **AWS CLI v2** — v1 is end-of-life. On Windows you download the 64-bit `.msi` installer.

## Step-by-Step Practical

### Part A — Windows

1. Open the official AWS CLI page (searching "aws cli" also lands you here).

```
https://aws.amazon.com/cli/
```

2. Go to the download page and choose **Windows → 64-bit**. AWS CLI v2 is the recommended version; download the `.msi` installer.

3. Run the installer:
   - Double-click the downloaded `.msi` file.
   - **Next** through the Setup Wizard.
   - Tick **I accept the terms in the License Agreement**.
   - On **Custom Setup**, leave the default (**Anyone who uses this computer**).
   - Click **Install**, then **Finish**. Keep all default settings throughout.

4. Or install silently from an **Administrator** PowerShell instead of steps 2-3.

```powershell
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi /qn
```

5. Open a **new** Command Prompt or PowerShell and verify.

```powershell
aws --version
```

### Part B — Linux

6. Download and run the official installer bundle (use `aarch64` instead of `x86_64` on ARM).

```bash
curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip -q awscliv2.zip
sudo ./aws/install
```

7. Clean up the downloaded files and verify.

```bash
rm -rf awscliv2.zip aws && aws --version
```

Avoid `apt install awscli` — distro repositories usually ship the retired v1.

### Part C — macOS

8. Install the signed package.

```bash
curl -fsSL "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o AWSCLIV2.pkg
sudo installer -pkg AWSCLIV2.pkg -target /
aws --version
```

9. Homebrew works too.

```bash
brew install awscli
```

### Part D — Configure the CLI (all platforms)

10. Run the interactive configuration and enter your IAM user's key pair, region, and output format.

```bash
aws configure
```

You will be prompted for four values:

```
AWS Access Key ID [None]:     YOUR_ACCESS_KEY
AWS Secret Access Key [None]: YOUR_SECRET_KEY
Default region name [None]:   ap-south-1
Default output format [None]: json
```

11. Prefer a **named profile** so you can keep several accounts side by side.

```bash
aws configure --profile tf-user
```

```bash
export AWS_PROFILE=tf-user     # Linux/macOS
```

```powershell
$env:AWS_PROFILE = "tf-user"   # PowerShell
```

12. Inspect what was written. Note that `credentials` holds the secret in **plain text** — protect this file.

```bash
cat ~/.aws/config
```

```bash
cat ~/.aws/credentials   # contains your secret key: never share, never commit
```

The resulting `credentials` file looks like this:

```json
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
```

13. Tighten the file permissions on Linux/macOS.

```bash
chmod 600 ~/.aws/credentials ~/.aws/config
```

14. Test that the credentials actually work.

```bash
aws sts get-caller-identity
```

## Expected Output

```
C:\> aws --version
aws-cli/2.15.20 Python/3.11.6 Windows/10 exe/AMD64
```

If the version appears, AWS CLI is installed successfully.

```json
$ aws sts get-caller-identity
{
    "UserId": "AIDAEXAMPLE1111",
    "Account": "YOUR_ACCOUNT_ID",
    "Arn": "arn:aws:iam::YOUR_ACCOUNT_ID:user/tf-user"
}
```

## Verification

1. `aws --version` reports a version starting with `aws-cli/2`.
2. `aws sts get-caller-identity` returns the ARN of `tf-user`, not an error.
3. The configured region is what you expect.

```bash
aws configure get region
```

4. A real API call succeeds.

```bash
aws iam list-users --output table
```

5. Confirm the config location if anything looks odd.

```bash
aws configure list
```

## Cleanup

No billable resources were created. To remove the local configuration:

```bash
rm -f ~/.aws/credentials ~/.aws/config
```

```powershell
Remove-Item "$env:USERPROFILE\.aws\credentials","$env:USERPROFILE\.aws\config"
```

To uninstall the CLI: Windows → **Settings → Apps → AWS Command Line Interface v2 → Uninstall**; Linux → `sudo rm -rf /usr/local/aws-cli /usr/local/bin/aws`.

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `'aws' is not recognized` / `aws: command not found` | Terminal opened before install finished | Close and reopen the terminal; check `where.exe aws` / `which aws` |
| `aws-cli/1.x` reported | Old v1 install shadowing v2 | Remove the v1 package (`pip uninstall awscli` / `apt remove awscli`) and reinstall v2 |
| `Unable to locate credentials` | `aws configure` never run, or wrong profile | Run `aws configure --profile tf-user` and set `AWS_PROFILE` |
| `InvalidClientTokenId: The security token included in the request is invalid` | Access key deleted, deactivated, or mistyped | Recreate the key in IAM and re-run `aws configure` |
| `SignatureDoesNotMatch` | Secret key copied with a trailing space or newline | Re-enter the secret key carefully |
| `You must specify a region` | No default region configured | `aws configure set region ap-south-1` or pass `--region` |
| `AccessDenied` on a specific call | IAM policy does not permit that action | Check the user's group policies |
| `RequestExpired` | Machine clock is skewed | Sync system time (NTP / "Set time automatically") |

## Key Takeaways
- Always install AWS CLI v2; v1 from distro repos is retired.
- `aws --version` proves the install, `aws sts get-caller-identity` proves the credentials.
- Terraform reads the same credential chain as the CLI, so configuring one configures both.
- `~/.aws/credentials` stores your secret key in plain text — `chmod 600` it and never commit it.
- Named profiles (`--profile`) keep multiple AWS accounts cleanly separated.

## Next: [AWS CLI User Access Config on VS Code](05-aws-cli-config-vscode.md)
