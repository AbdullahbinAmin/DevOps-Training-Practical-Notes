# Day 03 — Terraform Resource Delete

> Complete the create → modify → delete lifecycle with `terraform destroy`, and use `terraform validate` to catch config mistakes early.

## Learning Objectives
- Destroy Terraform-managed infrastructure safely with `terraform destroy`.
- Understand what "there is no undo" means for destroy.
- Recall the core command set: `init`, `plan`, `apply`, `destroy`, `validate`.
- Read and fix a `terraform validate` error.
- Destroy a single resource with `-target` instead of everything.

## Prerequisites
- Terraform 1.x and configured AWS credentials (profile or environment variables, no keys in `.tf`).
- The EC2 instance from the previous topic still running, with `terraform.tfstate` in the folder.
- You are inside the project directory (for example `C:\aws-ec2`).

## Concept (plain English, beginner friendly)
The lifecycle you have now walked through is:

```text
Create  ->  Modify  ->  Delete
apply       apply       destroy
```

`terraform destroy` reads the state file, works out everything Terraform created, and removes it in dependency-safe order. It only touches what is in *your* configuration and state — resources someone else created by hand are untouched.

There is no undo. Once the instance terminates, the instance ID and root volume data are gone. That is why Terraform demands you type the literal word `yes`; nothing else is accepted.

`terraform validate` is the cheap safety net. It checks HCL syntax and internal consistency without calling AWS at all, so it catches typos and misplaced blocks in a second instead of failing halfway through an apply.

## Step-by-Step Practical

1. Confirm what Terraform currently manages:

```bash
terraform state list
terraform show
```

2. Validate the configuration syntax first:

```bash
terraform validate
```

3. Preview the destruction without doing it (recommended before every destroy):

```bash
terraform plan -destroy
```

4. Destroy the resources and confirm with `yes`:

```bash
terraform destroy
```

The prompt looks like this:

```text
Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to approve.

  Enter a value: yes
```

5. Optional — destroy only one resource instead of everything:

```bash
terraform destroy -target=aws_instance.my_server
```

6. Optional — see a validation failure on purpose. Add a broken nested block to `main.tf`:

```hcl
resource "aws_instance" "my_server" {
  ami           = "ami-0abcd1234abcdef0"
  instance_type = "t3.nano"

  provider {           # invalid: provider is not a block inside a resource
    region = "us-east-1"
  }
}
```

7. Run validate and read the error, then remove those lines and re-validate:

```bash
terraform validate
terraform fmt
terraform validate
```

8. Review the full command set you now know:

```bash
terraform init      # initialize working directory, download providers
terraform validate  # check configuration syntax
terraform plan       # show what changes will happen
terraform apply      # create or update resources
terraform destroy    # delete all managed resources
```

Quick flow: `init → plan → apply → (destroy)`.

## Expected Output

Destroy:

```text
aws_instance.my_server: Destroying... [id=i-01234abcd]
aws_instance.my_server: Still destroying... [10s elapsed]
aws_instance.my_server: Destruction complete after 41s

Destroy complete! Resources: 1 destroyed.
```

Valid configuration:

```text
Success! The configuration is valid.
```

Invalid configuration:

```text
Error: Unsupported block type

  on main.tf line 2, in provider:
   2:   provider {

Blocks of type "provider" are not expected here.
```

## Verification
- `terraform state list` returns nothing after a full destroy.
- AWS CLI shows the instance terminated:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=Sample-Server" \
  --query "Reservations[].Instances[].{Id:InstanceId,State:State.Name}" \
  --output table
```

- AWS Console: **EC2 → Instances** shows `Sample-Server` as `terminated` (it disappears from the list after roughly an hour).
- Also check **EC2 → Volumes** and **Elastic IPs** for anything left behind that would keep billing.

## Cleanup
Destroy *is* the cleanup for AWS. Locally, keep your `.tf` files (they are your source of truth) and remove generated artefacts only if you no longer need the workspace:

```bash
rm -rf .terraform
```

Do not delete `terraform.tfstate` by hand while resources still exist — you would orphan real AWS resources and keep paying for them. And never commit it:

```text
*.tfstate
*.tfstate.*
.terraform/
*.tfvars
```

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `Unsupported block type ... provider { }` | A `provider` block nested inside a resource | Move `provider "aws" { region = ... }` to the top level of the file |
| Destroy prompt rejects your input | You typed `y`, `Y`, or `Yes` | Type exactly `yes` |
| `Error: Instance cannot be destroyed` | Resource has `lifecycle { prevent_destroy = true }` | Remove that lifecycle setting, then destroy |
| `DependencyViolation` | Something still attached (ENI, security group in use, non-empty S3 bucket) | Destroy dependents first, or add `force_destroy = true` for buckets |
| `No changes. No objects need to be destroyed.` | Empty or missing state file, or wrong directory | Run in the correct folder; confirm with `terraform state list` |
| Resource still visible in AWS after destroy | Console cache, or it was created manually and is not in state | Refresh the console; `terraform import` unmanaged resources if you want Terraform to own them |
| `Error acquiring the state lock` | Concurrent or crashed run | Wait for it to finish, or `terraform force-unlock <LOCK_ID>` only when certain |

## Key Takeaways
- `destroy` deletes only what is in your configuration and state — nothing else.
- There is no undo; only the literal `yes` is accepted.
- Run `terraform plan -destroy` before destroying in any shared environment.
- `terraform validate` catches syntax mistakes before AWS is ever contacted.
- Keep your `.tf` files safe; treat the state file as sensitive and never commit it.
- Infrastructure as Code = power plus responsibility.

## Next: [Terraform Config and State Overview](./13-config-and-state-overview.md)
