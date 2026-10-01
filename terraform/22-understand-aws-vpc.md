# Chapter 22 — Understand AWS VPC

## Objective
Understand VPC, regions, availability zones, subnets, CIDR blocks, route tables, internet gateways and security groups. This is theory for the next three chapters. No commands are needed.

## What is a VPC?
**VPC = Virtual Private Cloud.** It is a private, isolated network inside the AWS cloud where you launch and manage your resources securely.

**Example:** When you create an EC2 instance without choosing a network, AWS uses a shared default network. If you want more control (an important project, a website, many servers), create your own private network first (a VPC) and create your instances inside it. Other AWS users cannot reach it.

## Region and Availability Zone (AZ)
- AWS has data centers all over the world, grouped into **Regions** (example: Mumbai, Stockholm).
- Each region has multiple **Availability Zones** (AZs), for example `a`, `b`, `c`. An AZ is one or more separate data centers.
- Why multiple AZs? To avoid a **single point of failure** (power cut, fire, etc.).

## Subnet
A subnet is a smaller part of your VPC.

```text
Region (Mumbai)
└── VPC (10.0.0.0/16)                ← belongs to the whole region
    ├── Public Subnet  (10.0.1.0/24) ← lives in ONE availability zone
    └── Private Subnet (10.0.2.0/24) ← lives in ONE availability zone
```

Key rules:
- **A VPC belongs to a region.**
- **A subnet belongs to one availability zone.**
- **EC2 instances are created inside a subnet.**

Typical use:
- **Public subnet** → web server / front end (reachable from the internet).
- **Private subnet** → database (not reachable from the internet, so lower risk).

## CIDR block
When you create a VPC you give a **CIDR block** = a range of IP addresses.

Example: `10.0.0.0/16`
- An IPv4 address has **32 bits**.
- `/16` means the first 16 bits are the network part. The remaining 16 bits are for hosts.
- Number of addresses = 2^(32−16) = 2^16 = **65,536** (about 65,000).

For a subnet: `10.0.1.0/24`
- First 24 bits fixed, last 8 bits free → 2^8 = **256** addresses.
- Range: `10.0.1.0` to `10.0.1.255`.
- The other subnet `10.0.2.0/24` → `10.0.2.0` to `10.0.2.255`.

Note: AWS keeps 5 addresses in each subnet for its own use, so the usable number is 251. The console may still show 256 as the total.

CIDR stands for **Classless Inter-Domain Routing**. In the console the VPC CIDR allowed prefix range is from `/16` (largest) to `/28` (smallest).

## Route table
A route table is a set of rules (routes) that decides **where network traffic from a subnet goes**.
- Every subnet must be associated with a route table.
- Example rule: "traffic to `0.0.0.0/0` (anything on the internet) → send to the Internet Gateway."
- When you create a VPC, AWS makes a **main route table** automatically. Subnets use it by default.
- The main route table only has a **local** route (traffic inside the VPC). Without an internet route, the network cannot reach the internet.

## Internet Gateway (IGW)
A VPC is private. To make a service (for example a website) reachable from a browser you need an **Internet Gateway** attached to the VPC, plus a route in the route table that sends internet traffic to it.

## Security Group (SG)
A security group is a **firewall for instances**.
- **Inbound rules** (also called **ingress**) = traffic coming in. Example: allow HTTP (port 80) and SSH (port 22).
- **Outbound rules** (also called **egress**) = traffic going out.

## Full picture

```text
Internet
   │
[Internet Gateway]
   │
[Route Table]  0.0.0.0/0 → IGW
   │
[Public Subnet 10.0.x.0/24]  ── EC2 (web server) + Security Group
[Private Subnet 10.0.y.0/24] ── EC2 (database)   (no internet route)
   \___________ all inside one VPC (10.0.0.0/16) ___________/
```

## Quick class questions (self-check)
1. Is a VPC tied to a region or to an AZ? (Region)
2. Is a subnet tied to a region or to an AZ? (AZ)
3. How many IP addresses are in `/24`? (256)
4. What makes a subnet "public"? (Its route table has a route to an Internet Gateway)

## Final Result
You understand every term used in the next chapters.
