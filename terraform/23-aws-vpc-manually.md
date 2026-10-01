# Chapter 23 — AWS VPC Manually (Console)

## Objective
Build the VPC design by hand: VPC → 2 subnets → route table → internet gateway → route → subnet association → launch an EC2 instance inside the VPC.

## Prerequisites
- Chapter 22 theory
- IAM user login; region selected (all resources must be created in the same region)

## Architecture / Flow
```text
my-vpc (10.0.0.0/16)
├── public-subnet  10.0.1.0/24 ──(my-route-table: 0.0.0.0/0 → my-igw)
└── private-subnet 10.0.2.0/24 ──(main route table)
```

## Part 1 — Create the VPC (GUI)

**Step 1:** Console search → `VPC` → open **VPC**.

**Step 2:** Click **Create VPC**.

**Step 3:** Choose **VPC only** (to keep it simple and understand each part).

**Step 4:** Enter:
- **Name tag:** `my-vpc`
- **IPv4 CIDR block:** **IPv4 CIDR manual input**
- **IPv4 CIDR:** `10.0.0.0/16`
- **IPv6 CIDR block:** **No IPv6 CIDR block**
- **Tenancy:** Default

**Step 5:** Click **Create VPC**.

**Expected:** VPC created with about 65,000 IPs. In the VPC details, open the **Resource map**: you will see the VPC, 0 subnets, and a **main route table** (created automatically).

## Part 2 — Create the subnets

**Step 1:** Left menu → **Subnets** → **Create subnet**.

**Step 2:** **VPC ID:** select `my-vpc`.

**Step 3 — Subnet 1:**
- **Subnet name:** `public-subnet`
- **Availability Zone:** choose the first one in the list (for example `eu-north-1a`)
- **IPv4 subnet CIDR block:** `10.0.1.0/24`

**Step 4:** Click **Add new subnet**.

**Step 5 — Subnet 2:**
- **Subnet name:** `private-subnet`
- **Availability Zone:** any (same or different, both work because they are in the same VPC)
- **IPv4 subnet CIDR block:** `10.0.2.0/24`

**Step 6:** Click **Create subnet**.

**Expected:** two subnets, each with 256 IP addresses shown.

## Part 3 — Create a route table

**Step 1:** Left menu → **Route tables** → **Create route table**.

**Step 2:** **Name:** `my-route-table`. **VPC:** `my-vpc`.

**Step 3:** Click **Create route table**.

## Part 4 — Create and attach an Internet Gateway

**Step 1:** Left menu → **Internet gateways** → **Create internet gateway**.

**Step 2:** **Name tag:** `my-igw` → **Create internet gateway**.

**Step 3:** On the next page click **Actions → Attach to VPC**.

**Step 4:** Select `my-vpc` → **Attach internet gateway**.

**Expected:** state becomes **Attached**.

## Part 5 — Associate the public subnet with the route table

**Step 1:** **Route tables** → click `my-route-table`.

**Step 2:** Open the **Subnet associations** tab. It shows both subnets under "subnets without explicit associations" (they use the main route table by default).

**Step 3:** Click **Edit subnet associations**.

**Step 4:** Tick **only** `public-subnet` → **Save associations**.

## Part 6 — Add the internet route

**Step 1:** In `my-route-table`, open the **Routes** tab → **Edit routes**.

**Step 2:** Click **Add route**:
- **Destination:** `0.0.0.0/0`
- **Target:** **Internet Gateway** → select `my-igw`

**Step 3:** Click **Save changes**.

## Part 7 — Verify with the Resource map
VPC → open `my-vpc` → **Resource map**.

**Expected:**
- `public-subnet` → `my-route-table` → `my-igw`.
- `private-subnet` → main route table (no internet gateway).

## Part 8 — Launch an EC2 instance inside this VPC

**Step 1:** EC2 → **Launch instance**.

**Step 2:** **Name:** `my-web-server`. Keep default image and type (free tier eligible).

**Step 3:** **Key pair:** choose an existing key pair (from Chapter 10) or continue as needed.

**Step 4:** **Network settings → Edit**:
- **VPC:** `my-vpc`
- **Subnet:** `private-subnet` (example: a database server)
- **Auto-assign public IP:** Disable (a private subnet does not need it)
- **Security group:** keep the default option (**Create security group**); rules can be adjusted later.

**Step 5:** **Launch instance**.

**Step 6 — Verify:** EC2 → Instances → click the instance. **Subnet ID** should be the private subnet, and **Private IPv4 address** should start with `10.0.2.` (within the range of that subnet).

## Cleanup (in this order)
1. Terminate the EC2 instance.
2. VPC → Internet gateways → `my-igw` → **Actions → Detach from VPC**, then **Delete**.
3. Delete `my-route-table` (first remove its explicit associations if asked).
4. Delete both subnets.
5. Delete `my-vpc`.

Or select `my-vpc` → **Actions → Delete VPC** (this can remove related items). ⚠️ Verification Required for the exact behaviour in your console.

## Troubleshooting

### Error 1
Cannot delete the internet gateway.
**Reason:** It is still attached to a VPC.
**Fix:** Detach first, then delete.

### Error 2
The instance cannot be created in the subnet.
**Reason:** Region mismatch (console region is different from the VPC region).
**Fix:** Switch the top-right region to the one where `my-vpc` exists.

## Final Result
A working manual VPC with public and private subnets, internet gateway, route table, and an EC2 instance with a private IP in `10.0.2.x`.
