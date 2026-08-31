# Day 06 — Understand AWS VPC

> A VPC is your own private, isolated virtual network inside AWS where you control IP ranges, subnets, routing and security.

## Learning Objectives
- Explain what a VPC is and why the AWS default shared network is not enough.
- Read a CIDR block and work out how many IPs it gives you.
- Describe the region -> AZ -> VPC -> subnet -> resource hierarchy.
- Tell public and private subnets apart, and know which component makes each work.
- Compare Security Groups with Network ACLs.

## Prerequisites
- Basic IP address and networking vocabulary (IP, subnet mask, gateway).
- An AWS account (free tier is fine) — no resources are created in this lesson.
- Console access to the VPC dashboard for read-only exploration.

## Concept

### What is a VPC?
A **VPC (Virtual Private Cloud)** is a private, isolated network inside the AWS cloud where you launch and manage resources securely. Think: *your own virtual network in AWS*.

Without a VPC, EC2 instances land in AWS's default shared network — if 10 users launch EC2, all of them sit in the same shared space. With your own VPC, you define the network and everything inside it is isolated from other tenants.

Key points:
- Full control over the network.
- Security through isolation.
- Custom IP ranges.
- Can connect to other networks (on-premises or another VPC).
- Required for any serious production application.

### How it fits together

```
1. Region            2. Availability Zones      3. VPC              4. Subnets            5. Resources
   (ap-south-1)  ->    AZ-a  AZ-b  AZ-c    ->   10.0.0.0/16   ->   public / private  ->   EC2, RDS ...
   where in the        multiple datacenters      your private        smaller blocks         run inside
   world                per region               network             of the VPC             subnets
```

### CIDR in plain English
CIDR (Classless Inter-Domain Routing) is just "an IP range written as address/prefix". The number after the slash says how many bits are *fixed*; the rest are free for hosts. Bigger prefix = smaller network.

| CIDR | Total IPs | Usable in AWS | Typical use |
|---|---|---|---|
| `10.0.0.0/16` | 65,536 | 65,531 | whole VPC |
| `10.0.1.0/24` | 256 | 251 | one subnet |
| `10.0.1.0/28` | 16 | 11 | tiny subnet |

AWS allows VPC CIDRs from `/16` down to `/28`. AWS reserves 5 addresses in every subnet (network address, VPC router, DNS, future use, broadcast). Choose ranges that do **not** overlap with your other VPCs or your office network, otherwise peering and VPN will never work.

### Subnets
A subnet is a slice of the VPC CIDR that lives in exactly **one Availability Zone**.
- **Public subnet** — its route table sends `0.0.0.0/0` to an Internet Gateway, so resources can reach (and be reached from) the internet.
- **Private subnet** — no route to an IGW. It can reach the internet *outbound only* via a NAT Gateway, and cannot be reached from the internet.

There is no checkbox called "public". A subnet is public purely because of its route table.

```
                 VPC 10.0.0.0/16
   +---------------------------+---------------------------+
   |          AZ-a             |          AZ-b             |
   |  Public  10.0.1.0/24      |  Public  10.0.3.0/24      |     +----------+
   |     EC2 (web)  ---------------- IGW ------------------------|  Internet |
   |  Private 10.0.2.0/24      |  Private 10.0.4.0/24      |     +----------+
   |     RDS (db) ----------------- NAT GW --------(outbound only)^
   +---------------------------+---------------------------+
```

### Core components

| Component | What it does |
|---|---|
| VPC | Your virtual private network with a chosen CIDR |
| Subnet | Smaller network block inside one AZ |
| Internet Gateway (IGW) | Doorway between a public subnet and the internet |
| Route Table | Decides where traffic goes (destination -> target) |
| Security Group | Stateful firewall at the **instance / ENI** level |
| Network ACL | Stateless firewall at the **subnet** level |
| NAT Gateway | Lets private-subnet resources make outbound internet calls |
| Elastic IP | Static public IPv4 address |
| VPC Peering | Private connection between two VPCs |
| VPN / Direct Connect | Connects on-premises networks to the VPC |

### Security Group vs Network ACL

| Aspect | Security Group | Network ACL |
|---|---|---|
| Attaches to | Instance / ENI | Subnet |
| State | Stateful (return traffic auto-allowed) | Stateless (must allow both directions) |
| Rules | Allow only | Allow **and** deny |
| Evaluation | All rules evaluated together | Rules in number order, first match wins |
| Default | Deny all inbound, allow all outbound | Default NACL allows all in/out |
| Typical use | Day-to-day access control | Coarse subnet-wide blocks (e.g. ban an IP) |

## Step-by-Step Practical
This lesson is read-only exploration — you build things in notes 22 and 23.

1. Confirm your CLI identity and region.

```bash
aws sts get-caller-identity
aws configure get region
```

2. List the VPCs that already exist, including the default VPC.

```bash
aws ec2 describe-vpcs \
  --query 'Vpcs[].{Id:VpcId,Cidr:CidrBlock,Default:IsDefault}' \
  --output table
```

3. List the subnets of the default VPC and note which AZ each one sits in.

```bash
aws ec2 describe-subnets \
  --query 'Subnets[].{Id:SubnetId,Cidr:CidrBlock,AZ:AvailabilityZone,AutoPublicIP:MapPublicIpOnLaunch}' \
  --output table
```

4. Inspect the route tables and find the `0.0.0.0/0` route.

```bash
aws ec2 describe-route-tables \
  --query 'RouteTables[].{Id:RouteTableId,Vpc:VpcId,Routes:Routes[].{Dest:DestinationCidrBlock,GW:GatewayId}}'
```

5. Do the CIDR maths by hand for `10.0.0.0/16`, `10.0.1.0/24` and `10.0.1.0/28` and check against the table above.

## Expected Output
Step 2 should show at least the default VPC:

```
------------------------------------------------
|                  DescribeVpcs                |
+------------------------+---------------+------+
|          Id            |     Cidr      | Def  |
+------------------------+---------------+------+
|  vpc-0a1b2c3d4e5f6a7b8 |  172.31.0.0/16|  True|
+------------------------+---------------+------+
```

A default-VPC route table will contain two routes: `172.31.0.0/16 -> local` and `0.0.0.0/0 -> igw-xxxxxxxx`.

## Verification
- You can name your region and at least two AZs in it.
- You can point at the route entry that makes a subnet public.
- You can state, without looking, that `/24` = 256 addresses and 251 usable in AWS.
- You can say which firewall is stateful and which is stateless.

## Cleanup
Nothing was created, so nothing to delete. Do **not** delete the default VPC — other tutorials rely on it.

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `Unable to locate credentials` | AWS CLI not configured | Run `aws configure` and set an access key, secret and region |
| `An error occurred (UnauthorizedOperation)` | IAM user lacks `ec2:Describe*` | Attach a read policy such as `AmazonVPCReadOnlyAccess` |
| Empty subnet list | Region has no default VPC or you are in the wrong region | Add `--region <region>` or recreate the default VPC |
| CIDR overlap when peering later | Two networks share the same range | Plan non-overlapping ranges up front (e.g. 10.0/16, 10.1/16) |
| Instance in "public" subnet has no internet | Missing `0.0.0.0/0 -> igw` route or no public IP | Add the route and enable auto-assign public IP |

## Key Takeaways
- A VPC is your own private network in AWS; everything else in this day hangs off it.
- CIDR size decides how many IPs you get — `/16` for a VPC, `/24` for subnets is a safe default.
- Subnets live in a single AZ; spread across AZs for high availability.
- "Public" is a property of the route table, not of the subnet itself.
- Security Groups are stateful and instance-level; NACLs are stateless and subnet-level.

## Next: [AWS VPC Manually](./22-aws-vpc-manually.md)

