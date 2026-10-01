# Chapter 39 — Terraform Registry: Publishing Our Own Module

## Objective
Package our custom module, push it to **GitHub**, and publish it to the official **Terraform Registry** (`registry.terraform.io`) so that anyone in the team or community can consume it via standard registry syntax.

## Prerequisites
- GitHub Account
- Git installed on your machine
- Module files created in Chapter 38

---

## 1. Naming & Requirements for Terraform Registry

The Terraform Public Registry enforces strict rules for module repositories:
1. **GitHub Repository Name Format**: Must follow:
   ```text
   terraform-<PROVIDER>-<NAME>
   ```
   *Example*: `terraform-aws-custom-vpc` (Provider: `aws`, Name: `custom-vpc`).
2. **Repository Visibility**: Must be **Public**.
3. **Semantic Version Tags**: Releases must be tagged with SemVer format: `vX.Y.Z` or `X.Y.Z` (e.g., `v1.0.0`).
4. **Standard File Structure**:
   ```text
   terraform-aws-custom-vpc/
   ├── README.md              # Explains module, requirements, usage
   ├── LICENSE                # MIT, Apache-2.0, etc.
   ├── main.tf                # Module implementation
   ├── variables.tf           # Module inputs
   ├── outputs.tf             # Module outputs
   ├── versions.tf            # Provider & Terraform constraints
   └── examples/
       └── complete/
           ├── main.tf        # Working example using the module
           ├── outputs.tf
           └── README.md
   ```

---

## Step 1 — Create GitHub Repository

1. Open [GitHub.com](https://github.com) → Click **New repository**.
2. **Repository name**: `terraform-aws-custom-vpc` (or your chosen unique suffix).
3. **Visibility**: Select **Public**.
4. **Initialize with**:
   - Check **Add a README file** (or push existing).
   - **Add .gitignore**: `Terraform`.
   - **Choose a license**: `MIT License`.
5. Click **Create repository**.

---

## Step 2 — Prepare Module Files Locally

Clone your newly created GitHub repository:
```bash
git clone https://github.com/<YOUR_GITHUB_USERNAME>/terraform-aws-custom-vpc.git
cd terraform-aws-custom-vpc
```

Copy the child module files created in Chapter 38 into the root of this git repository:
- `main.tf`
- `variables.tf`
- `outputs.tf`
- `versions.tf`

### Add `examples/complete/main.tf`
```bash
mkdir -p examples/complete
```

Create `examples/complete/main.tf`:
```hcl
provider "aws" {
  region = "eu-north-1"
}

module "vpc_example" {
  source = "../../"

  vpc_config = {
    cidr_block = "10.0.0.0/16"
    name       = "example-vpc"
  }

  subnet_config = {
    "pub-subnet-1" = {
      cidr_block = "10.0.1.0/24"
      az         = "eu-north-1a"
      public     = true
    }
    "priv-subnet-1" = {
      cidr_block = "10.0.2.0/24"
      az         = "eu-north-1b"
      public     = false
    }
  }
}

output "vpc_id" {
  value = module.vpc_example.vpc_id
}
```

Update `README.md` with:
- Module overview
- Usage snippet with copyable code block
- List of inputs and outputs

---

## Step 3 — Commit and Release with Git Tag

Terraform Registry detects module versions **only through git tags**:

```bash
git add .
git commit -m "feat: complete initial release of custom AWS VPC module"
git push origin main

# Create and push SemVer release tag
git tag v1.0.0
git push origin v1.0.0
```

Verify on GitHub: the tag `v1.0.0` is visible under **Releases / Tags**.

---

## Step 4 — Publish on Terraform Registry

1. Open [https://registry.terraform.io](https://registry.terraform.io).
2. Click **Sign in** in the top right → Choose **Sign in with GitHub**.
3. Authorize HashiCorp to access your public repositories.
4. Click **Publish** → Select **Module**.
5. Select your repository: `<YOUR_GITHUB_USERNAME>/terraform-aws-custom-vpc`.
6. Agree to the Terms of Use and click **Publish Module**.
7. Terraform Registry imports the module, parses `variables.tf` (inputs), `outputs.tf` (outputs), `README.md`, and creates the public documentation page.

---

## Step 5 — Test Consuming the Published Module

Now test using your published module like any public Terraform module!

Create a separate test directory:
```bash
mkdir -p ../test-published-module && cd ../test-published-module
```

Create `main.tf`:
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-north-1"
}

module "production_vpc" {
  source  = "<YOUR_GITHUB_USERNAME>/custom-vpc/aws"
  version = "1.0.0"

  vpc_config = {
    cidr_block = "10.10.0.0/16"
    name       = "prod-vpc-from-registry"
  }

  subnet_config = {
    "public-1" = {
      cidr_block = "10.10.1.0/24"
      az         = "eu-north-1a"
      public     = true
    }
  }
}
```

Run:
```bash
terraform init
```
Notice Terraform downloads your custom module directly from the Terraform Registry!

```bash
terraform plan
terraform apply -auto-approve
```

---

## Step 6 — Clean Up

```bash
terraform destroy -auto-approve
```
