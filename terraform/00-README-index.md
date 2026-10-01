# Terraform + AWS Practical Course — Chapter Notes (Chapters 1–46)

These notes follow the video chapter by chapter. Each file is written so you can open a terminal or browser and follow **Step 1 → Final Result** in class.

## Chapter List

| File | Chapter | Type |
|---|---|---|
| 01-course-intro.md | Course Intro | Overview + demo |
| 02-what-is-terraform.md | What is Terraform? | Theory (short) |
| 03-terraform-install-windows.md | Terraform Installation – Windows | Practical |
| 04-terraform-install-mac.md | Terraform Installation – Mac | Practical |
| 05-aws-account-signup.md | AWS Account – Sign Up | GUI |
| 06-aws-user-setup.md | AWS User Setup (IAM + MFA) | GUI |
| 07-aws-cli-windows.md | AWS CLI Setup – Windows | Practical |
| 08-aws-cli-mac.md | AWS CLI Setup – Mac | Practical |
| 09-aws-cli-user-access-vscode.md | AWS CLI user access in VS Code | Practical |
| 10-aws-ec2-manually.md | AWS EC2 Manually | GUI |
| 11-first-terraform-config-ec2.md | First Terraform config (EC2) | Practical |
| 12-terraform-init-plan-apply-destroy.md | init, plan, apply | Practical |
| 13-terraform-resource-modify.md | Modify a resource | Practical |
| 14-terraform-resource-delete.md | Delete a resource, validate | Practical |
| 15-terraform-config-and-state-overview.md | Config (HCL/JSON) and State | Practical |
| 16-terraform-variables-and-outputs.md | Variables and Outputs (basic) | Practical |
| 17-aws-s3-overview.md | AWS S3 Overview + manual bucket | GUI |
| 18-terraform-s3-bucket-and-upload.md | S3 bucket + file upload | Practical |
| 19-terraform-random-provider.md | Random provider | Practical |
| 20-terraform-remote-backend-s3.md | Remote state in S3 | Practical |
| 21-project-static-website.md | Project: static website on S3 | Project |
| 22-understand-aws-vpc.md | Understand AWS VPC | Theory |
| 23-aws-vpc-manually.md | AWS VPC manually | GUI |
| 24-aws-vpc-using-terraform.md | AWS VPC using Terraform | Practical |
| 25-aws-ec2-plus-vpc.md | EC2 inside our VPC | Practical |
| 26-project-ec2-vpc-nginx-http.md | Project: EC2 + VPC + NGINX + HTTP | Project |
| 27-terraform-data-source.md | Data Sources | Practical |
| 28-vpc-and-sg-with-data-source.md | VPC & SG with data source | Practical |
| 29-task-ec2-using-existing-vpc.md | Task: EC2 in existing VPC | Task |
| 30-terraform-variables-detailed.md | Variables: types, validation, env, tfvars | Practical |
| 31-terraform-operators-expressions.md | Operators & Expressions: List, Map, Loops | Practical |
| 32-terraform-functions.md | Built-in Functions & Console | Practical |
| 33-multiple-resources-count.md | Multiple Resources using `count` | Practical |
| 34-multiple-resources-for-each.md | Multiple Resources using `for_each` | Practical |
| 35-project-aws-iam-management.md | Project: AWS IAM Management (YAML driven) | Project |
| 36-terraform-modules.md | Terraform Modules & Public VPC Module | Practical |
| 37-aws-ec2-public-terraform-module.md | AWS EC2 using Public Terraform Module | Practical |
| 38-creating-our-own-terraform-module.md | Creating Our Own Terraform Module | Practical |
| 39-terraform-registry-publish-module.md | Terraform Registry: Publishing Our Module | Practical |
| 40-terraform-resources-dependency.md | Resource Dependencies: Implicit vs Explicit | Practical |
| 41-terraform-resources-lifecycle.md | Resource Lifecycle Rules | Practical |
| 42-terraform-validations.md | Validations: Precondition & Postcondition | Practical |
| 43-terraform-state-modifications.md | State Modifications: mv, rm, show, list | Practical |
| 44-terraform-import-command.md | Terraform Import Command | Practical |
| 45-terraform-workspace.md | Terraform Workspaces: Multi-Environment | Practical |
| 46-terraform-cloud-with-github.md | Terraform Cloud with GitHub GitOps | Project |

## Things that apply to the WHOLE course

1. **Region.** The course default is `eu-north-1` (Stockholm). You may use any region, but use the **same region in the console and in `provider "aws"`**.
2. **AMI IDs are region-specific.** An AMI ID that works in one region will NOT work in another. Always copy the AMI ID from your own console (Chapter 10 / 11 show how).
3. **Instance type.** Use one that shows **Free tier eligible** in your console (`t3.micro` or `t2.micro`).
4. **Credentials file.** Keep a `.env` file with your AWS keys. In every new terminal run `source .env` first. **Never upload `.env` to GitHub.**
5. **Cost.** Destroy everything after each practical (`terraform destroy`).
6. **File names.** Terraform reads every `*.tf` file in the current folder. Always run Terraform commands from inside the correct project folder.
7. **Versions.** Provider version numbers in these notes (for example `~> 5.0`) are examples. Check current versions on registry.terraform.io.

## Course Lab Folder Layout

```text
tf-aws/
├── .env
├── aws-ec2/                 (Ch 11–14, 16)
├── s3/                      (Ch 18–19)
├── tf-backend/              (Ch 20)
├── project-static-website/  (Ch 21)
├── aws-vpc/                 (Ch 24–25)
├── aws-vpc-ec2-nginx/       (Ch 26)
├── tf-data-sources/         (Ch 27–29)
├── tf-variable/             (Ch 30)
├── tf-expressions-demo/     (Ch 31–32)
├── tf-count-demo/           (Ch 33)
├── tf-for-each-demo/        (Ch 34)
├── aws-iam-management/      (Ch 35)
├── tf-module-vpc/           (Ch 36)
├── tf-module-ec2/           (Ch 37)
├── tf-own-module-vpc/       (Ch 38)
├── terraform-aws-custom-vpc/(Ch 39)
├── tf-dependencies/         (Ch 40)
├── tf-lifecycle/            (Ch 41)
├── tf-validations/          (Ch 42)
├── tf-state-manipulation/   (Ch 43)
├── tf-import-s3/            (Ch 44)
├── tf-workspaces/           (Ch 45)
└── tf-cloud-demo/           (Ch 46)
```
