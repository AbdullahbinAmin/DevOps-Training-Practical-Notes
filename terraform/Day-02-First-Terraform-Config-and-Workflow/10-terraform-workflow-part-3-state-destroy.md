# Day 02 — Workflow Part 3: State Files and terraform destroy

> Understand `terraform.tfstate`, inspect what Terraform is tracking, and tear the lab down cleanly with `terraform destroy`.

## Learning Objectives
- Explain what `terraform.tfstate` and `terraform.tfstate.backup` contain and why they matter.
- Inspect state with `terraform state list`, `terraform show`, and `terraform state show`.
- Run `terraform destroy`, read the destroy plan, and confirm the teardown.
- Target a single resource for destruction and know why targeting is a last resort.
- Keep state out of git and understand the case for a remote backend.

## Prerequisites
- A completed `terraform apply` from lesson 09, with an EC2 instance running.
- `terraform.tfstate` present in `TF-AWS-EC2`.
- AWS credentials with permission to terminate EC2 instances.

## Concept
State is Terraform's memory. When apply creates an instance, Terraform writes a JSON record into `terraform.tfstate` linking the address `aws_instance.my_server` to the real ID `i-0ab12cd34ef567890`, along with every attribute it read back. On the next run, that record is how Terraform knows the instance already exists and does not need creating again. `terraform.tfstate.backup` is simply the previous version, kept automatically so you can recover from a bad write.

This gives state two properties you must respect. First, it is sensitive: attributes are stored in plain text, so any password, key, or token that passes through a resource ends up readable in that file. Second, it is authoritative: Terraform only manages what is in state. Delete the file and Terraform forgets the instance while AWS keeps running and billing it — an orphaned resource you now have to clean up by hand.

`destroy` is apply in reverse. It builds a plan where every resource in state is marked for deletion, shows it, waits for `yes`, then deletes them in dependency-safe order and removes them from state. In a training account this is the command that keeps your bill at zero. In production it is the most dangerous command Terraform has, which is why the confirmation prompt exists.

## Step-by-Step Practical

1. List the resource addresses Terraform is tracking.

```bash
terraform state list
```

2. Inspect the full state in readable form, then a single resource.

```bash
terraform show
terraform state show aws_instance.my_server
```

3. Look at the raw state structure. Read it, do not edit it — hand-editing state is how projects break.

```bash
terraform show -json | head -40
```

```json
{
  "format_version": "1.0",
  "terraform_version": "1.9.0",
  "values": {
    "root_module": {
      "resources": [
        {
          "address": "aws_instance.my_server",
          "type": "aws_instance",
          "name": "my_server",
          "values": {
            "id": "i-0ab12cd34ef567890",
            "instance_type": "t3.micro",
            "public_ip": "18.207.XX.XX",
            "tags": { "Name": "sample-server" }
          }
        }
      ]
    }
  }
}
```

4. Confirm state is excluded from version control before you commit anything.

```text
# .gitignore
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.tfvars
```

```bash
git check-ignore -v terraform.tfstate
```

5. Preview the teardown before running it. `plan -destroy` shows what destroy would do without doing it.

```bash
terraform plan -destroy
```

6. Destroy the infrastructure. Read the plan, then type `yes`.

```bash
terraform destroy
```

7. If you only want to remove one resource out of several, target it. Use this sparingly — targeting bypasses Terraform's full dependency graph and can leave state inconsistent.

```bash
terraform destroy -target=aws_instance.my_server
```

8. For real projects, move state to a remote backend so it is shared, versioned, encrypted, and locked. Add this to `main.tf` and run `terraform init -migrate-state`. The S3 bucket must already exist.

```hcl
terraform {
  backend "s3" {
    bucket       = "my-terraform-state-bucket-CHANGE-ME"
    key          = "tf-aws-ec2/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true # S3-native state locking, AWS provider 5.x era
  }
}
```

## Expected Output

```text
PS C:\TF-AWS-EC2> terraform state list
aws_instance.my_server

PS C:\TF-AWS-EC2> terraform destroy
aws_instance.my_server: Refreshing state... [id=i-0ab12cd34ef567890]

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # aws_instance.my_server will be destroyed
  - resource "aws_instance" "my_server" {
      - ami           = "ami-0c02fb55956c7d316" -> null
      - id            = "i-0ab12cd34ef567890" -> null
      - instance_type = "t3.micro" -> null
      - tags          = {
          - "Name" = "sample-server"
        } -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

aws_instance.my_server: Destroying... [id=i-0ab12cd34ef567890]
aws_instance.my_server: Still destroying... [10s elapsed]
aws_instance.my_server: Destruction complete after 18s

Destroy complete! Resources: 1 destroyed.
```

## Verification
- Destroy ends with `Destroy complete! Resources: 1 destroyed.`
- `terraform state list` returns nothing.
- AWS Console -> EC2 -> Instances shows `No instances found` in that region (a terminated instance may linger in `terminated` state for up to an hour; that is normal and not billed).
- `terraform plan` reports the resource would need creating again.

```bash
terraform state list
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=sample-server" \
  --query "Reservations[].Instances[].[InstanceId,State.Name]" \
  --output table
```

## Cleanup

```bash
terraform destroy
rm -f tfplan plan.json
```

Check the console in every region you experimented in — Terraform only cleans up what is in its state file, so anything you created by hand in lesson 06 must be removed by hand. Also confirm no unattached EBS volumes, Elastic IPs, or key pairs are left behind, since those can carry charges of their own.

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `Destroy complete! Resources: 0 destroyed.` | State is empty or you are in the wrong directory | Check `terraform state list` and your current folder |
| Resource still visible in the console | Instance is in `shutting-down`/`terminated` state | Wait a few minutes and refresh; terminated instances are not billed |
| `DependencyViolation` | Another resource still references it (e.g. ENI, security group) | Let Terraform destroy the whole graph rather than using `-target` |
| `Error acquiring the state lock` | A crashed or concurrent run holds the lock | Wait, then `terraform force-unlock <LOCK_ID>` if you are certain no run is active |
| State file deleted, resources still running | Local state lost, no backend or backup | Restore `terraform.tfstate.backup`, or `terraform import` the resource back, or delete it manually |
| `Instance cannot be terminated` | Termination protection is enabled | Disable `disable_api_termination` on the instance, then destroy |
| Secrets leaked in a repo | `terraform.tfstate` was committed | Rotate the exposed credentials immediately, then purge the file from history and add `.gitignore` |
| Destroy removed more than intended | `destroy` targets everything in state by default | Always read the destroy plan; use a separate directory/workspace per environment |

## Key Takeaways
- `terraform.tfstate` maps your code to real AWS resources; without it Terraform is blind.
- State holds attribute values in plain text — never commit it, and prefer an encrypted remote backend for team work.
- `terraform plan -destroy` lets you review a teardown before committing to it.
- `destroy` has no undo. Read the plan, then type `yes` deliberately.
- Destroy at the end of every lab session so the free tier and your bill stay intact.

## Next: [Day 03 — Terraform Variables, Outputs, and Data Sources](../Day-03-Variables-Outputs-and-Data-Sources/README.md)
