# Chapter 46 — Terraform Cloud (HCP Terraform) with GitHub GitOps

## Objective
Implement a production **GitOps CI/CD workflow** using **HCP Terraform (Terraform Cloud)** and **GitHub**:
1. Create an HCP Terraform organization and a **VCS-driven Workspace**.
2. Connect Terraform Cloud to a GitHub repository using OAuth.
3. Securely configure AWS credentials (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) as sensitive environment variables in Terraform Cloud.
4. Trigger automated speculative plans on Git commits and pushes.
5. Review and confirm applies directly in the Terraform Cloud Web UI.
6. Trigger an automated teardown using Terraform Cloud's destruction workflow.

## Prerequisites
- GitHub account
- AWS IAM Access Key & Secret Key with administrative permissions
- Free account on [app.terraform.io](https://app.terraform.io)

---

## 1. What is HCP Terraform (Terraform Cloud)?

HCP Terraform is a managed service provided by HashiCorp that replaces manual CLI operations with centralized, automated infrastructure management:
- **Remote State Management & Locking**: Never worry about state files on local disks or concurrent apply conflicts.
- **VCS Integration (GitOps)**: Every `git push` or pull request automatically triggers a Terraform plan.
- **Speculative Plans on Pull Requests**: Preview infrastructure changes before merging pull requests.
- **Secure Secret Management**: Cloud provider credentials stored centrally and encrypted.
- **Team Access Control**: Role-based access control (RBAC) across projects and workspaces.

---

## Step 1 — Create GitHub Repository

1. Open [GitHub.com](https://github.com) → Click **New repository**.
2. **Repository name**: `tf-cloud-demo`
3. **Visibility**: Select **Private** (or Public).
4. Add `.gitignore` (`Terraform`) and `README.md`.
5. Click **Create repository**.

---

## Step 2 — Set Up HCP Terraform Account & Organization

1. Navigate to [https://app.terraform.io](https://app.terraform.io).
2. Sign in with GitHub or your HCP account.
3. If this is your first time, create an Organization:
   - Click **Create organization**.
   - Name: e.g. `MyOrg-DevOps-Lab` (must be unique).
   - Enter your email address.

---

## Step 3 — Create a VCS-Driven Workspace

1. Inside your Organization, click **Workspaces** → **Create workspace** (or **New Workspace**).
2. Choose Workflow Type: **Version control workflow**.
3. Connect to VCS Provider: Select **GitHub** → **GitHub.com**.
4. Authorize HashiCorp on GitHub and select repository: `tf-cloud-demo`.
5. **Workspace Name**: `tf-cloud-demo` (defaults to repo name).
6. Click **Create workspace**.

---

## Step 4 — Configure AWS Credentials in Terraform Cloud

Because Terraform runs inside HashiCorp's cloud runners, you must provide your AWS credentials as Workspace Variables.

1. In your Workspace, click the **Variables** tab in the left sidebar.
2. Under **Workspace variables**, click **+ Add variable**.
3. Add the following **Environment Variables** (select the **Environment variable** radio button, NOT Terraform variable):

| Variable Type | Key | Value | Sensitive |
|---|---|---|---|
| **Environment** | `AWS_ACCESS_KEY_ID` | `AKIA...` (your key) | Check **Yes** |
| **Environment** | `AWS_SECRET_ACCESS_KEY` | `wJalr...` (your secret) | Check **Yes** |
| **Environment** | `AWS_DEFAULT_REGION` | `eu-north-1` (your region) | No |

> [!CAUTION]
> Always mark `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as **Sensitive** so their values are write-only and hidden in logs.

---

## Step 5 — Write Configuration Locally & Push to GitHub

Clone the repository to your local machine:
```bash
git clone https://github.com/<YOUR_GITHUB_USERNAME>/tf-cloud-demo.git
cd tf-cloud-demo
```

Create `main.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  # Region is automatically injected from AWS_DEFAULT_REGION variable
}

resource "random_id" "server_id" {
  byte_length = 6
}

resource "aws_s3_bucket" "cloud_bucket" {
  bucket = "tf-cloud-automated-${random_id.server_id.hex}"

  tags = {
    ManagedBy = "Terraform-Cloud"
    Pipeline  = "GitOps"
  }
}

output "bucket_name" {
  value = aws_s3_bucket.cloud_bucket.bucket
}
```

Commit and push to GitHub:
```bash
git add main.tf
git commit -m "feat: add s3 bucket configuration for terraform cloud"
git push origin main
```

---

## Step 6 — Review Speculative Plan & Apply in Cloud UI

1. Open your workspace dashboard on [app.terraform.io](https://app.terraform.io).
2. Go to the **Runs** tab.
3. Observe: A run was automatically triggered seconds after your `git push`!
4. Click on the active run:
   - Notice the commit message: `"feat: add s3 bucket configuration for terraform cloud"`.
   - Inspect the **Plan output**: Terraform Cloud displays interactive additions (`+ 2 to add`).
5. Status displays: **Needs Confirmation** (waiting for manual approval).
6. Click **Confirm & Apply**.
7. Add an optional comment (e.g. `Approved initial deployment`) → Click **Confirm Plan**.
8. The runner applies the plan. Once complete, status switches to **Applied** with green checkmarks.

---

## Step 7 — Verify Live Infrastructure

1. Open **AWS Management Console → S3**.
2. Find the bucket named `tf-cloud-automated-<HEX_ID>`.
3. Verify tags: `ManagedBy: Terraform-Cloud`, `Pipeline: GitOps`.
4. In Terraform Cloud, click the **States** tab:
   - The state is stored, versioned, and backed up in Terraform Cloud.

---

## Step 8 — Test Automated Drift / Updates via Git

Edit `main.tf` locally to change the byte length:
```hcl
resource "random_id" "server_id" {
  byte_length = 8 # Changed from 6 to 8
}
```

Push change:
```bash
git add main.tf
git commit -m "chore: update random byte length to 8"
git push origin main
```

Return to Terraform Cloud:
- A new run is automatically created.
- The UI highlights: `~ replacement required for random_id.server_id`.
- Click **Confirm & Apply** to execute the rolling update.

---

## Step 9 — Clean Up (Destroy via Terraform Cloud)

In VCS-driven workspaces, you destroy infrastructure directly through the Terraform Cloud UI:

1. In your Workspace, click **Settings** (top menu) → **Destruction and Deletion**.
2. Under **Queue destroy plan**, click **Queue destroy plan**.
3. Type the workspace name to confirm (e.g. `tf-cloud-demo`).
4. Click **Queue destroy plan**.
5. Go to the **Runs** tab:
   - A plan is generated to destroy all resources.
6. Click **Confirm & Apply** to execute destruction.
7. Verify in AWS Console that the S3 bucket is completely removed.
