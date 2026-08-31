# Day 02 — Workflow Part 1: terraform init

> Initialize the working directory, download the AWS provider plugin, and understand every file `init` leaves behind.

## Learning Objectives
- Run `terraform init` and read its output line by line.
- Identify what `.terraform/`, `.terraform.lock.hcl`, and `LICENSE.txt` are for.
- Explain when re-running `init` is required.
- Use `-upgrade`, `-reconfigure`, and `-backend=false` when appropriate.
- Commit the lock file and ignore the plugin cache.

## Prerequisites
- `main.tf` in place from lesson 07.
- Terraform 1.x installed.
- Outbound HTTPS access to `registry.terraform.io`.

## Concept
`terraform init` is the setup step. It reads the `required_providers` block, contacts the Terraform Registry, picks a version that satisfies your constraint, downloads that plugin binary, and verifies its signature. It also configures the backend — the place state will be stored. With no `backend` block, the backend is local, meaning `terraform.tfstate` sits in your project folder.

Init is safe and idempotent. It never touches infrastructure, so you can run it as often as you like. You must run it in each new directory, after cloning a repo, and any time you add or change a provider, module, or backend.

Two artefacts matter afterwards. `.terraform/` holds the downloaded plugin binaries — large, machine-specific, and disposable, so it belongs in `.gitignore`. `.terraform.lock.hcl` records the exact provider versions and their checksums; it is small and should be committed so every teammate and CI runner resolves identical versions.

## Step-by-Step Practical

1. Move into the project directory. Init is always relative to your current directory.

```bash
cd TF-AWS-EC2
ls
```

2. Confirm the provider requirement Terraform will resolve.

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

3. Run init.

```bash
terraform init
```

4. Inspect what appeared.

```bash
ls -a
tree .terraform -L 4
```

5. Read the lock file. It names the resolved version and its checksums.

```bash
cat .terraform.lock.hcl
```

```hcl
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.46.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:...",
    "zh:...",
  ]
}
```

6. Learn the flags you will actually use.

```bash
terraform init -upgrade        # allow newer versions within the constraint
terraform init -reconfigure    # reconfigure the backend, ignore saved settings
terraform init -backend=false  # install plugins only, skip backend setup
terraform init -migrate-state  # move existing state to a new backend
```

7. Confirm `.terraform/` is ignored and the lock file is not.

```text
# .gitignore
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.tfvars

# Do NOT ignore .terraform.lock.hcl — commit it.
```

## Expected Output

```text
PS C:\TF-AWS-EC2> terraform init

Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.46.0...
- Installed hashicorp/aws v5.46.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure.
```

Directory afterwards:

```text
TF-AWS-EC2/
├── main.tf
├── .gitignore
├── .terraform/
│   └── providers/
│       └── registry.terraform.io/
│           └── hashicorp/aws/5.46.0/...
└── .terraform.lock.hcl
```

## Verification
- The final line is `Terraform has been successfully initialized!`
- `.terraform/providers/registry.terraform.io/hashicorp/aws/` contains a version directory.
- `terraform providers` lists the AWS provider with its constraint.
- `terraform version` reports both Terraform and the provider version.
- `terraform validate` succeeds — validate requires an initialized directory.

```bash
terraform providers
terraform version
terraform validate
```

## Cleanup
`init` creates nothing in AWS, so there is nothing to destroy. To reset the local setup and start over:

```bash
rm -rf .terraform .terraform.lock.hcl
terraform init
```

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `Failed to query available provider packages` | No network, proxy, or firewall blocking the registry | Check connectivity/proxy env vars, then retry `terraform init` |
| `Inconsistent dependency lock file` | Constraint changed but lock file is stale | `terraform init -upgrade` |
| `provider registry.terraform.io/hashicorp/aws: locked provider does not match` | Lock file recorded checksums for another platform | `terraform providers lock -platform=linux_amd64 -platform=windows_amd64` |
| `Unsupported Terraform Core version` | Your Terraform is older than `required_version` | Upgrade Terraform, or relax the constraint |
| `Backend configuration changed` | You added or edited a `backend` block | `terraform init -reconfigure` or `-migrate-state` |
| `Error: Could not load plugin` after moving the folder | Absolute paths in `.terraform/` are stale | Delete `.terraform/` and re-init |
| Init succeeds but plan fails on credentials | Init does not need AWS credentials; plan does | Run `aws configure` and verify with `aws sts get-caller-identity` |

## Key Takeaways
- `init` downloads providers and prepares the backend. It changes nothing in the cloud.
- Run it once per directory, and again after any provider, module, or backend change.
- Commit `.terraform.lock.hcl`; ignore `.terraform/`.
- Init needs internet access but not AWS credentials.

## Next: [Workflow Part 2 — terraform plan and apply](09-terraform-workflow-part-2-plan-apply.md)
