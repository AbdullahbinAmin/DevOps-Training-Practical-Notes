# Day 06 — AWS VPC Manually

> Build a working VPC by hand — CIDR, two subnets, IGW, route table, association, and an EC2 in the private subnet — so you understand what Terraform will automate next.

## Learning Objectives
- Create a VPC with a planned CIDR block.
- Create a public and a private subnet in different AZs.
- Create an Internet Gateway and attach it to the VPC.
- Create a route table, add the `0.0.0.0/0 -> IGW` route, and associate it with the public subnet.
- Launch an EC2 instance in a private subnet and confirm it gets a private IP only.

## Prerequisites
- AWS account with permissions on EC2/VPC.
- AWS CLI v2 installed and configured (`aws configure`), or console access.
- A region chosen — this lab uses `eu-north-1` (Stockholm), matching the slide.
- An EC2 key pair if you plan to connect to an instance.

## Concept

### CIDR block
CIDR defines the IP range of your VPC and subnets. Example: `10.0.0.0/16` = 65,536 addresses. AWS accepts `/16` through `/28`. Each subnet takes a slice of that range, and AWS reserves 5 IPs per subnet.

Plan the range before you click anything: overlapping ranges break peering and VPN forever.

### Target architecture

```
Region eu-north-1
+----------------------------- VPC 10.0.0.0/16 ------------------------------+
|            AZ-a                    |             AZ-b                     |
|  Public  10.0.1.0/24               |   Public  10.0.3.0/24                |
|     EC2 (web)  ------+             |      EC2 ------+                     |
|                      |             |                |                     |
|  Private 10.0.2.0/24 |             |   Private 10.0.4.0/24                |
|     EC2 / RDS --------\            |      RDS ---\  |                     |
+-----------------------|------------+-------------|--|---------------------+
                        |                          |  |
                  [public route table] --0.0.0.0/0--> [IGW] --> Internet
                        |                             ^
                  [main route table] ---0.0.0.0/0---> [NAT GW] (outbound only)
```

### Components you will touch

| Component | Role in this lab |
|---|---|
| VPC | `10.0.0.0/16` container for everything |
| Subnet | `10.0.1.0/24` public, `10.0.2.0/24` private |
| Internet Gateway | Gives the public subnet internet access |
| Route Table | Holds `0.0.0.0/0 -> igw-...` |
| Association | Binds the route table to the public subnet |
| Security Group | Instance-level stateful firewall |
| Network ACL | Subnet-level stateless firewall (default left as-is) |
| NAT Gateway | Optional: outbound internet for the private subnet |

### SG vs NACL quick reference

| | Security Group | Network ACL |
|---|---|---|
| Scope | Instance / ENI | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow + Deny, ordered |

## Step-by-Step Practical

The order matters: VPC -> subnets -> IGW -> attach -> route table -> route -> associate -> instances.

1. Set shell variables so later commands stay readable.

```bash
export AWS_REGION=eu-north-1
export AZ_A=eu-north-1a
export AZ_B=eu-north-1b
```

2. Create the VPC and enable DNS hostnames.

```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=my-vpc}]' \
  --query 'Vpc.VpcId' --output text)
echo "VPC_ID=$VPC_ID"

aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames
```

3. Create the public subnet (`10.0.1.0/24`) and the private subnet (`10.0.2.0/24`).

```bash
PUB_SUBNET=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" --cidr-block 10.0.1.0/24 --availability-zone "$AZ_A" \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet}]' \
  --query 'Subnet.SubnetId' --output text)

PRIV_SUBNET=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" --cidr-block 10.0.2.0/24 --availability-zone "$AZ_A" \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet}]' \
  --query 'Subnet.SubnetId' --output text)

echo "public=$PUB_SUBNET private=$PRIV_SUBNET"
```

4. Auto-assign public IPs in the public subnet only.

```bash
aws ec2 modify-subnet-attribute --subnet-id "$PUB_SUBNET" --map-public-ip-on-launch
```

5. Create the Internet Gateway and attach it to the VPC.

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=my-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway --vpc-id "$VPC_ID" --internet-gateway-id "$IGW_ID"
```

6. Create a route table for the public subnet.

```bash
RT_ID=$(aws ec2 create-route-table --vpc-id "$VPC_ID" \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-route-table}]' \
  --query 'RouteTable.RouteTableId' --output text)
```

7. Add the default route to the IGW.

```bash
aws ec2 create-route --route-table-id "$RT_ID" \
  --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID"
```

8. Associate the route table with the **public** subnet. The private subnet keeps the main route table, which has no internet route — that is exactly what makes it private.

```bash
aws ec2 associate-route-table --route-table-id "$RT_ID" --subnet-id "$PUB_SUBNET"
```

9. Create a security group. **Warning:** `0.0.0.0/0` on port 22 means the whole internet can attempt SSH. Acceptable in a throwaway lab only — in any real environment restrict it to your own IP (`<your-ip>/32`) or use AWS Systems Manager Session Manager instead of SSH.

```bash
SG_ID=$(aws ec2 create-security-group \
  --group-name lab-web-sg --description "Lab web SG" --vpc-id "$VPC_ID" \
  --query 'GroupId' --output text)

MY_IP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --protocol tcp --port 22 --cidr "${MY_IP}/32"
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
```

10. Launch an EC2 instance in the **private** subnet with no public IP (slide example: name `my-web-server`, auto-assign public IP = No).

```bash
AMI_ID=$(aws ssm get-parameters \
  --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameters[0].Value' --output text)

aws ec2 run-instances \
  --image-id "$AMI_ID" --instance-type t3.micro \
  --subnet-id "$PRIV_SUBNET" --security-group-ids "$SG_ID" \
  --no-associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=my-web-server}]'
```

11. (Optional) Give the private subnet outbound internet with a NAT Gateway. This costs money per hour — skip unless required.

```bash
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
NAT_ID=$(aws ec2 create-nat-gateway --subnet-id "$PUB_SUBNET" \
  --allocation-id "$EIP_ALLOC" --query 'NatGateway.NatGatewayId' --output text)
```

## Expected Output
Each create call returns JSON; with `--query` you get bare IDs:

```json
{
  "Subnet": {
    "SubnetId": "subnet-0abc123def4567890",
    "CidrBlock": "10.0.2.0/24",
    "AvailabilityZone": "eu-north-1a",
    "MapPublicIpOnLaunch": false,
    "State": "available"
  }
}
```

The private instance receives a private IPv4 address from the subnet range, e.g. `10.0.2.15`, and **no** public IP.

## Verification
1. Console path: VPC Dashboard -> Your VPCs / Subnets / Internet Gateways / Route Tables. Confirm the route table shows `10.0.0.0/16 -> local` and `0.0.0.0/0 -> igw-xxxx`, and that the association lists the public subnet.
2. Console path for the instance: EC2 Dashboard -> click the instance ID -> check **Subnet ID** and **Private IPv4 address**.
3. From the CLI:

```bash
aws ec2 describe-route-tables --route-table-ids "$RT_ID" \
  --query 'RouteTables[0].{Routes:Routes,Assoc:Associations[].SubnetId}'

aws ec2 describe-instances --filters "Name=tag:Name,Values=my-web-server" \
  --query 'Reservations[].Instances[].{Id:InstanceId,Subnet:SubnetId,Private:PrivateIpAddress,Public:PublicIpAddress}' \
  --output table
```

`Public` should be `None` for the private-subnet instance.

## Cleanup
Delete in reverse order of creation, otherwise dependency errors block you. Review each ID before running — these commands are destructive.

```bash
# 1. terminate instances
aws ec2 terminate-instances --instance-ids <instance-id>
aws ec2 wait instance-terminated --instance-ids <instance-id>

# 2. NAT gateway + EIP (only if created)
aws ec2 delete-nat-gateway --nat-gateway-id "$NAT_ID"
aws ec2 release-address --allocation-id "$EIP_ALLOC"

# 3. route table association, route table, IGW
aws ec2 disassociate-route-table --association-id <assoc-id>
aws ec2 delete-route-table --route-table-id "$RT_ID"
aws ec2 detach-internet-gateway --vpc-id "$VPC_ID" --internet-gateway-id "$IGW_ID"
aws ec2 delete-internet-gateway --internet-gateway-id "$IGW_ID"

# 4. security group, subnets, VPC
aws ec2 delete-security-group --group-id "$SG_ID"
aws ec2 delete-subnet --subnet-id "$PUB_SUBNET"
aws ec2 delete-subnet --subnet-id "$PRIV_SUBNET"
aws ec2 delete-vpc --vpc-id "$VPC_ID"
```

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `InvalidVpcRange: The CIDR '10.0.0.0/8' is invalid` | VPC CIDR must be /16 to /28 | Use `10.0.0.0/16` |
| `InvalidSubnet.Conflict` | New subnet range overlaps an existing one | Pick a non-overlapping slice such as `10.0.3.0/24` |
| `InvalidParameterValue: subnet CIDR not within VPC CIDR` | Subnet outside the VPC range | Keep subnets inside `10.0.0.0/16` |
| `DependencyViolation` on `delete-vpc` | Subnets, IGW, SGs or ENIs still exist | Delete children first, in reverse order |
| `Gateway.NotAttached` when adding a route | IGW created but never attached | Run `attach-internet-gateway` before `create-route` |
| Public instance unreachable | Route table not associated, or no public IP | Associate the route table and enable auto-assign public IP |
| `Client.InternetGatewayLimitExceeded` | One IGW per VPC already exists | Reuse the existing IGW |
| Private instance cannot run `yum update` | No NAT Gateway | Create a NAT GW and route `0.0.0.0/0` from the private route table to it |

## Key Takeaways
- Order of operations is the whole lesson: VPC, subnets, IGW, attach, route table, route, associate, launch.
- A subnet is public only because its route table points `0.0.0.0/0` at an IGW.
- The private subnet stays isolated by simply not having that route.
- NAT Gateway gives private resources outbound-only internet, and it is billed hourly.
- Never leave SSH open to `0.0.0.0/0` outside a disposable lab.
- Doing this by hand takes ~15 minutes and is not repeatable — which is exactly why the next note uses Terraform.

## Next: [AWS VPC Using Terraform](./23-aws-vpc-using-terraform.md)

