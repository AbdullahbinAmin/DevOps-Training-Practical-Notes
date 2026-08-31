# Day 02 — Workflow Part 2: terraform plan and terraform apply

> Preview exactly what Terraform will do, then create the EC2 instance and verify it in the AWS Console.

## Learning Objectives
- Run `terraform plan` and read the `+`, `~`, `-`, and `-/+` action symbols.
- Interpret `(known after apply)` and the `Plan: X to add, Y to change, Z to destroy.` summary.
- Save a plan to a file and apply exactly that plan.
- Run `terraform apply`, confirm with `yes`, and read the creation log.
- Verify the instance in both the console and the CLI.

## Prerequisites
- `terraform init` completed successfully in `TF-AWS-EC2`.
- Valid AWS credentials with permission to run and terminate EC2 instances.
- A region and AMI ID valid for your account.

## Concept
`plan` is a dry run. Terraform reads your desired state from the `.tf` files, refreshes the current state from AWS, diffs them, and prints an execution plan. Nothing is created. It is safe to run as many times as you want, which is why it is the right way to check any change before it happens.

The symbols tell you the action for each resource: `+` create, `~` update in place, `-` destroy, and `-/+` destroy then recreate. That last one deserves attention — it means the resource will be replaced and anything on it will be lost. Attributes AWS assigns at creation time, such as `id` and `public_ip`, show as `(known after apply)` because Terraform genuinely cannot know them yet.

`apply` performs the plan. By default it recomputes the plan, shows it, and waits for you to type `yes` — only the literal word `yes` is accepted. Then it calls the AWS APIs, streams progress lines, and writes results into `terraform.tfstate`. That state file is now the record of what exists.

## Step-by-Step Practical

1. Start from a validated, initialized project.

```bash
cd TF-AWS-EC2
terraform fmt
terraform validate
```

2. The configuration being applied:

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
  region = "us-east-1"
}

resource "aws_instance" "my_server" {
  ami           = "ami-0c02fb55956c7d316" # Amazon Linux 2
  instance_type = "t3.micro"              # free tier eligible

  tags = {
    Name = "sample-server"
  }
}
```

3. Generate the plan and read the summary line at the bottom.

```bash
terraform plan
```

4. Optional but good practice: save the plan to a binary file so `apply` runs exactly what you reviewed, with no re-computation and no drift between review and execution.

```bash
terraform plan -out=tfplan
terraform show tfplan          # human readable
terraform show -json tfplan > plan.json
```

5. Apply the configuration. Read the plan again, then type `yes`.

```bash
terraform apply
```

Or apply the saved plan — this skips the prompt because you already approved that plan.

```bash
terraform apply tfplan
```

6. Add useful outputs so you do not have to dig through state for the IP. Append to `main.tf`, then re-apply.

```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.my_server.id
}

output "public_ip" {
  description = "Public IPv4 address of the EC2 instance"
  value       = aws_instance.my_server.public_ip
}
```

```bash
terraform apply
terraform output public_ip
```

7. Verify in the AWS Console: EC2 -> Instances. You should see `sample-server`, state `Running`, type `t3.micro`, with a public IPv4 address.

8. Confirm from the CLI too.

```bash
terraform state list
terraform show
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=sample-server" \
  --query "Reservations[].Instances[].[InstanceId,State.Name,PublicIpAddress]" \
  --output table
```

Note: `-auto-approve` skips the confirmation prompt. It belongs in CI pipelines where a saved plan was already reviewed, not in your interactive shell while learning.

## Expected Output

```text
PS C:\TF-AWS-EC2> terraform plan
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_instance.my_server will be created
  + resource "aws_instance" "my_server" {
      + ami                          = "ami-0c02fb55956c7d316"
      + instance_type                = "t3.micro"
      + id                           = (known after apply)
      + private_ip                   = (known after apply)
      + public_ip                    = (known after apply)
      + tags                         = {
          + "Name" = "sample-server"
        }
      # ... (many other attributes)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

PS C:\TF-AWS-EC2> terraform apply

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_instance.my_server: Creating...
aws_instance.my_server: Still creating... [10s elapsed]
aws_instance.my_server: Still creating... [20s elapsed]
aws_instance.my_server: Creation complete after 25s [id=i-0ab12cd34ef567890]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

instance_id = "i-0ab12cd34ef567890"
public_ip = "18.207.XX.XX"
```

## Verification
- `apply` ends with `Apply complete! Resources: 1 added, 0 changed, 0 destroyed.`
- `terraform state list` prints `aws_instance.my_server`.
- Running `terraform plan` again prints `No changes. Your infrastructure matches the configuration.` — this is the real proof that state, config, and AWS all agree.
- The console shows the instance `Running` with status checks passing.

```bash
terraform plan
terraform output
```

## Cleanup
Leaving an instance running burns free-tier hours and can cost money. Destroy it as soon as the exercise is done.

```bash
terraform destroy
```

Also delete the artefacts from step 4, which can contain sensitive values:

```bash
rm -f tfplan plan.json
```

Reminder: `terraform.tfstate` records real attribute values and must never be committed.

## Common Errors & Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `InvalidAMIID.NotFound` | AMI does not exist in the configured region | Look up the AMI for your region and update `ami` |
| `UnauthorizedOperation: ec2:RunInstances` | IAM user lacks EC2 permissions | Attach a policy allowing `ec2:RunInstances` and related actions |
| `VcpuLimitExceeded` / `InstanceLimitExceeded` | Account quota reached | Terminate unused instances or request a quota increase |
| `Saved plan is stale` | Infrastructure or config changed after `-out` | Regenerate the plan with `terraform plan -out=tfplan` |
| Typed `y` instead of `yes` | Only the literal `yes` is accepted | Re-run apply and type `yes` |
| `Error acquiring the state lock` | A previous run crashed holding the lock | Wait, then if needed `terraform force-unlock <LOCK_ID>` |
| `Unsupported instance type` in your region | `t3.micro` unavailable in that AZ/region | Use `t2.micro` or another supported type |
| Plan shows `-/+ destroy and then create` | You changed an immutable attribute such as `ami` | Expected behaviour; confirm you accept the replacement before applying |

## Key Takeaways
- `plan` is a free, read-only preview. `apply` is the action.
- `+ ~ - -/+` are the four action symbols; `-/+` means replacement and data loss.
- `(known after apply)` marks attributes AWS assigns at creation time.
- `plan -out=tfplan` then `apply tfplan` guarantees you execute exactly what you reviewed.
- A clean second `plan` showing `No changes` is the best confirmation of a successful apply.

## Next: [Workflow Part 3 — state and terraform destroy](10-terraform-workflow-part-3-state-destroy.md)
