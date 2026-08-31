# Day 02 — First Terraform Config to Create an EC2 Instance

> Write your first `main.tf` with a provider block and an `aws_instance` resource, then initialize the project with `terraform init`.

## Learning Objectives
- Create a Terraform project folder and a `main.tf` file.
- Write a `terraform` block with `required_providers` pinning the AWS provider to 5.x.
- Write a `provider "aws"` block with the correct region.
- Write an `aws_instance` resource with `ami`, `instance_type`, and `tags`.
- Run `terraform init` and explain what it downloads.

## Prerequisites
- Terraform 1.x installed and on your PATH (`terraform version`).
- AWS CLI installed and configured with credentials (`aws sts get-caller-identity` works).
- VS Code (or any editor). The HashiCorp Terraform extension is recommended.
- The AMI ID for Amazon Linux 2 in **your** region, copied from the EC2 console.

Never put access keys inside `.tf` files. Use `aws configure`, environment variables, or an IAM role.

```bash
terraform version
aws sts get-caller-identity
```

## Concept
Terraform reads every `.tf` file in the current directory and builds a picture of the infrastructure you want. Three block types get you started.

The `terraform` block is configuration about Terraform itself. Inside it, `required_providers` names each provider and pins its version, so your project keeps working when a new provider release changes behaviour. `"~> 5.0"` means "any 5.x, but not 6.0".

The `provider "aws"` block configures that provider — mainly which region to talk to. Credentials are picked up from your environment, not written here.

A `resource` block describes one real object. `resource "aws_instance" "my_server"` has a type (`aws_instance`, decided by the provider) and a local name (`my_server`, chosen by you). You refer to it elsewhere as `aws_instance.my_server`. Inside, `ami` is the OS image, `instance_type` is the hardware size, and `tags` are optional key-value labels — the `Name` tag is what shows up in the console's Name column.

## Step-by-Step Practical

1. Create the project folder and open it in VS Code.

```bash
mkdir TF-AWS-EC2
cd TF-AWS-EC2
code .
```

2. Install the Terraform extension for syntax highlighting, autocomplete, and inline docs. In VS Code press `Ctrl+Shift+X`, search `HashiCorp Terraform`, and install the official extension (publisher: HashiCorp).

3. Find the Amazon Linux 2 AMI ID for your region. AMI IDs are region-specific, so the slide's `ami-0c02fb55956c7d316` is only valid in `us-east-1`. Either copy it from EC2 -> Launch instance -> Quick Start, or query it from the CLI.

```bash
aws ssm get-parameters \
  --names /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 \
  --region us-east-1 \
  --query "Parameters[0].Value" \
  --output text
```

4. Create `main.tf` with the full configuration below. This is the complete file — copy it as is and change only the region and AMI.

```hcl
# main.tf

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1" # change to your region, e.g. eu-north-1
}

resource "aws_instance" "my_server" {
  ami           = "ami-0c02fb55956c7d316" # Amazon Linux 2 (us-east-1 only)
  instance_type = "t3.micro"              # free tier eligible

  tags = {
    Name = "sample-server"
  }
}
```

5. Optional: add a second instance by copying the resource block and changing both the local name and the `Name` tag. The slide mentions creating two instances this way.

```hcl
resource "aws_instance" "my_server_2" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t3.micro"

  tags = {
    Name = "sample-server-2"
  }
}
```

6. Create a `.gitignore` before your first commit. State files can contain sensitive values and must never be pushed.

```text
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.tfvars
```

7. Format and validate the configuration. `fmt` fixes indentation, `validate` catches syntax and type errors.

```bash
terraform fmt
terraform validate
```

8. Initialize the working directory. Run this once per directory, and again whenever you change providers or modules.

```bash
terraform init
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
selections it made above.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure.
```

## Verification
- `terraform validate` prints `Success! The configuration is valid.`
- A `.terraform/` directory and a `.terraform.lock.hcl` file now exist in the project folder.
- `terraform providers` lists `registry.terraform.io/hashicorp/aws`.
- No infrastructure exists yet — nothing has been created in AWS at this point.

```bash
terraform validate
terraform providers
ls -a
```

## Cleanup
Nothing was created in AWS yet, so there is nothing to destroy. If you want to reset the local project state:

```bash
rm -rf .terraform .terraform.lock.hcl
```

Once you have run `apply` in the next lesson, always finish with:

```bash
terraform destroy
```

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `Could not load plugin` / `Inconsistent dependency lock file` | Provider block changed after init | Re-run `terraform init -upgrade` |
| `InvalidAMIID.NotFound` | AMI ID belongs to a different region | Look up the AMI ID for your own region |
| `No valid credential sources found` | AWS CLI not configured | Run `aws configure`, or export `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` |
| `Error: Unsupported argument` | Typo in an attribute name, e.g. `instance-type` | Use underscores and check the provider docs for `aws_instance` |
| `Missing required argument: ami` | Resource block incomplete | Both `ami` and `instance_type` are required for `aws_instance` |
| `Blocks of type "reovidce" are not expected here` | Misspelled `resource` keyword | Fix the spelling; `terraform validate` points at the line |
| Init hangs or fails to download | Network or proxy blocking registry.terraform.io | Check connectivity/proxy settings, then retry `terraform init` |

## Key Takeaways
- Terraform loads all `.tf` files in the directory; `main.tf` is just a convention.
- Pin provider versions with `~> 5.0` so builds stay reproducible.
- AMI IDs are region-specific — always look up your own.
- `terraform init` only prepares the directory and downloads plugins. It creates nothing in AWS.
- Add `.gitignore` before your first commit so `*.tfstate` never reaches the repo.

## Next: [Terraform Workflow Overview — Init, Plan, Apply, Destroy](08-terraform-workflow-overview.md)
