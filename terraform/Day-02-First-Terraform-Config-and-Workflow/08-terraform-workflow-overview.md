# Day 02 — Terraform Workflow Overview: Init, Plan, Apply, Destroy

> The four-command loop that takes you from a `.tf` file to running infrastructure and back to a clean account.

## Learning Objectives
- Name the four core Terraform commands and state what each one does.
- Describe the files and folders that appear after `init` and after `apply`.
- Read a plan summary line such as `Plan: 1 to add, 0 to change, 0 to destroy.`
- Run the full loop against a single EC2 instance.
- Explain why `plan` before `apply` is a habit, not an optional extra.

## Prerequisites
- The `TF-AWS-EC2` project and `main.tf` from the previous lesson.
- Terraform 1.x, AWS provider 5.x, working AWS credentials.
- A region and AMI ID valid for your account.

## Concept
The Terraform workflow is a loop: write config, init, plan, apply, and eventually destroy.

`init` prepares the directory. It downloads the provider plugins your config asks for and sets up the backend where state will live.

`plan` is a dry run. Terraform reads your config (the desired state), reads the state file plus the real cloud (the current state), and prints the difference as a list of additions, changes, and deletions. Nothing in AWS is touched.

`apply` executes that difference. It prints the same plan, asks you to type `yes`, then calls the AWS APIs and records what it created in `terraform.tfstate`.

`destroy` is the inverse of apply. It removes everything Terraform is tracking in that state file, which keeps a training account clean and cheap.

The thing that makes this work is state. `terraform.tfstate` is Terraform's memory of which real resources correspond to which blocks in your code. Delete it and Terraform forgets it ever created anything — while the resources keep running and keep billing you.

## Step-by-Step Practical

1. Confirm your project structure. Only `main.tf` is yours to write; the rest is generated.

```text
TF-AWS-EC2/
├── main.tf                     # your configuration
├── .gitignore                  # you write this
├── .terraform/                 # created by init: provider plugins
├── .terraform.lock.hcl         # created by init: pinned provider versions
├── terraform.tfstate           # created by apply: real resource mapping
└── terraform.tfstate.backup    # previous state, kept automatically
```

2. This is the configuration the whole workflow operates on.

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
  region = "us-east-1" # change as per your region
}

resource "aws_instance" "my_server" {
  ami           = "ami-0c02fb55956c7d316" # Amazon Linux 2
  instance_type = "t3.micro"              # free tier eligible

  tags = {
    Name = "sample-server"
  }
}
```

3. Initialize the directory.

```bash
terraform init
```

4. Preview the changes. Read the summary line at the bottom before doing anything else.

```bash
terraform plan
```

5. Create the infrastructure. Type `yes` when prompted.

```bash
terraform apply
```

6. Verify in the AWS Console: EC2 -> Instances should show `sample-server` in state `Running`.

7. Tear it down when the lab is finished. Type `yes` when prompted.

```bash
terraform destroy
```

## Expected Output

```text
PS C:\TF-AWS-EC2> terraform init
Terraform has been successfully initialized!

PS C:\TF-AWS-EC2> terraform plan
Terraform will perform the following actions:

  # aws_instance.my_server will be created
  + resource "aws_instance" "my_server" {
      + ami           = "ami-0c02fb55956c7d316"
      + instance_type = "t3.micro"
      + id            = (known after apply)
      + public_ip     = (known after apply)
      + tags          = {
          + "Name" = "sample-server"
        }
      # ... (many other attributes)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

PS C:\TF-AWS-EC2> terraform apply
Do you want to perform these actions?
  Enter a value: yes

aws_instance.my_server: Creating...
aws_instance.my_server: Still creating... [10s elapsed]
aws_instance.my_server: Creation complete after 18s [id=i-0a1b2c3d4e5f67890]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

PS C:\TF-AWS-EC2> terraform destroy
Plan: 0 to add, 0 to change, 1 to destroy.
  Enter a value: yes

aws_instance.my_server: Destroying... [id=i-0a1b2c3d4e5f67890]
aws_instance.my_server: Destruction complete after 12s

Destroy complete! Resources: 1 destroyed.
```

## Verification
- After `init`: `.terraform/` and `.terraform.lock.hcl` exist.
- After `plan`: the summary reads `Plan: 1 to add, 0 to change, 0 to destroy.`
- After `apply`: `terraform.tfstate` exists and lists the instance ID; the console shows the instance `Running`.
- A second `terraform plan` right after apply reports `No changes. Your infrastructure matches the configuration.`
- After `destroy`: the console shows no instances in that region.

```bash
terraform state list
terraform show
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=sample-server" \
  --query "Reservations[].Instances[].[InstanceId,State.Name]" \
  --output table
```

## Cleanup

```bash
terraform destroy
```

Never commit state. `terraform.tfstate` is plain JSON and can contain sensitive attribute values, so keep it out of git via `.gitignore` and, on real teams, store it in a remote backend such as S3 with DynamoDB locking.

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `Backend initialization required` | Config changed since last init | Run `terraform init` again |
| `Error acquiring the state lock` | Another Terraform run is active, or one crashed | Wait for it to finish; as a last resort `terraform force-unlock <LOCK_ID>` |
| Plan shows unexpected destroys | Someone changed resources in the console, or a field forces replacement | Read the `# forces replacement` markers before typing `yes` |
| `No changes` when you expected changes | Editing a file outside the working directory | Confirm you are in the right folder and the file ends in `.tf` |
| Resources still exist after `destroy` | They were created manually, not by Terraform | Delete them in the console; Terraform only manages what is in its state |
| Lost `terraform.tfstate` | State file deleted or never committed to a shared backend | Re-import with `terraform import`, or delete the orphaned resources manually |

## Key Takeaways
- Four commands cover the whole cycle: `init`, `plan`, `apply`, `destroy`.
- `plan` is free and safe. Run it every time before `apply`.
- State is the link between your code and real AWS resources — protect it and never commit it.
- The workflow's real payoff is repeatability: the same config produces the same infrastructure every time.

## Next: [Workflow Part 1 — terraform init](08a-terraform-workflow-part-1-init.md)
