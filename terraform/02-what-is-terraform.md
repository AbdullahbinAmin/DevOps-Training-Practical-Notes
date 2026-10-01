# Chapter 2 — What is Terraform?

## Objective
Understand what Terraform is, why we use Infrastructure as Code (IaC), and the basic Terraform workflow.

## One-line definition
> Terraform is an **open source Infrastructure as Code (IaC)** tool. You write your infrastructure in simple configuration files, and Terraform creates and manages it for you.

## Simple example
You want an AWS EC2 instance (a virtual machine). In the console you click many screens. With Terraform you write only the **desired state**:

- I want one EC2 instance.
- It should use this operating system image.
- It should have this size.

Terraform does all the steps to make it real. You describe **what** you want, not **how** to do it.

## Why not create things manually?
| Problem with manual work | How IaC solves it |
|---|---|
| Repeating work: creating 10 servers means 10 times the clicks | Write config once, reuse it many times |
| Inconsistency: you may change a setting by mistake next time | Same config gives the same result every time |
| Team work: nobody knows what you created | Config files can be versioned (Git), shared and reviewed |
| Tracking: hard to know what exists | Terraform keeps a **state** of what it created |

## Terraform workflow (very important)

```text
1. Requirement gathering  → decide provider (AWS, Azure, GCP...) and resources
2. Write config file      → .tf files
3. terraform init         → download provider plugins, set up folder
4. terraform plan         → preview what will change
5. terraform apply        → create/change the real infrastructure
6. Infrastructure ready to use
```

## Key facts to remember
- Terraform supports many providers (AWS, Azure, Google Cloud, Kubernetes, Oracle, databases like MongoDB Atlas, and many more). It is **not** only for AWS.
- The configuration language is **HCL** (HashiCorp Configuration Language). It is human-readable and **declarative**.
- Terraform tracks resources using a **state file**.
- Terraform Cloud is a managed service for team collaboration (covered later in the course).

## Classroom demo idea (optional, 2 minutes)
Show the Terraform Registry page: **https://registry.terraform.io** → Browse → Providers. Point out AWS, Azure, Google Cloud. Use the category filter on the left (for example "Database") to show that providers exist for many tools.

## Final Result
You can explain in your own words: IaC, declarative, provider, plan, apply, state.
