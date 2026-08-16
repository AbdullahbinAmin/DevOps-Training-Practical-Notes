# Deploy a 3-Tier Architecture on AWS — A Complete Beginner's Step-by-Step Guide

This guide will walk you, step by step, through building a **3-tier web application architecture** on AWS. It is written for beginners, so every step is explained in simple words. By the end, you will have a live website running on your own domain name, backed by a real database, with load balancing, auto-scaling, and security best practices.

---

## What is a 3-Tier Architecture?

A 3-tier architecture splits an application into **three separate layers**, so each layer can be scaled, secured, and managed on its own:

1. **Web Tier** — Handles requests from users. Runs Nginx web servers serving a React.js website.
2. **Application Tier** — Handles the business logic. Runs Node.js servers that process data.
3. **Database Tier** — Stores the data. Uses Amazon Aurora (MySQL-compatible).

### How Traffic Flows

- A user visits your website → request goes through the **Internet Gateway** → hits the **External (Internet-facing) Load Balancer** → forwarded to the **Web Tier EC2 servers** (Nginx + React).
- The Web Tier calls the API → request goes to the **Internal Load Balancer** → forwarded to the **Application Tier EC2 servers** (Node.js).
- The Application Tier reads/writes data in the **Aurora MySQL Database**, then sends the result back up the chain to the user.
- **NAT Gateways** allow the private servers (App Tier) to reach the internet (e.g., for updates) without being reachable from the internet.
- **Auto Scaling Groups** and **Health Checks** keep the app tier and web tier available even under heavy load or server failure.
- **S3** stores the application code that gets pulled into the EC2 servers.
- **CloudWatch** monitors the health and performance of the servers.
- **AWS Certificate Manager (ACM)** + a **custom domain** (in this guide we use **DigitalPlat** instead of GoDaddy) give your app HTTPS and a friendly web address.

> **Note:** The exact CIDR blocks, Availability Zones (AZs), and names used below are just examples. You can use different values as long as they follow the same logic.

---

## Prerequisites

Before you start, make sure you have:

- An **AWS account** (free tier is fine, but some services like RDS Standard and NAT Gateways will incur small charges).
- Basic familiarity with the AWS Console (or willingness to click around and learn).
- A computer with internet access and a text/code editor (VS Code recommended).
- A free domain name from **DigitalPlat** (https://dash.domain.digitalplat.org/domains) — we'll register this in Step 10.

---

## Step 1: Download the Application Code from GitHub

The application code (both the web tier React app and the app tier Node.js app) is provided by AWS in a public GitHub repository.

1. Open your browser and go to the **`aws-three-tier-web-architecture-workshop`** GitHub repository.
2. Click the green **Code** button, then click **Download ZIP**.
3. Once downloaded, **extract (unzip)** the folder somewhere easy to find, like your Desktop.
4. Inside, you should see a folder named **`application-code`**. This is what we will use later in Step 7.

Keep this folder open — we will come back to it soon.

---

## Step 2: Create the VPC and Networking Resources

A **VPC (Virtual Private Cloud)** is your own private, isolated section of the AWS cloud. It lets you control IP ranges, subnets, and routing so that some parts of your app (like the database) stay hidden from the public internet, while other parts (like the website) are publicly reachable.

### 2.1 Create the VPC

1. In the AWS Console, search for **VPC** and open the service.
2. Click **Create VPC**.
3. Choose **VPC only** (not "VPC and more") — this helps you understand every piece as you build it manually.
4. Give it a **Name tag** (e.g., `three-tier-vpc`).
5. Enter an **IPv4 CIDR block**, for example `10.0.0.0/16`.
6. Leave everything else as default and click **Create VPC**.

Your new VPC will now appear under **Your VPCs**.

### 2.2 Create Six Subnets

We need **6 subnets total** — 3 subnets in each of 2 Availability Zones:

| Subnet Name | Tier | Type | Example AZ |
|---|---|---|---|
| Public Subnet AZ1 | Web Tier | Public | us-east-1a |
| Public Subnet AZ2 | Web Tier | Public | us-east-1b |
| Private App Subnet AZ1 | App Tier | Private | us-east-1a |
| Private App Subnet AZ2 | App Tier | Private | us-east-1b |
| Private DB Subnet AZ1 | DB Tier | Private | us-east-1a |
| Private DB Subnet AZ2 | DB Tier | Private | us-east-1b |

Steps:

1. In the left sidebar, click **Subnets**, then **Create subnet**.
2. Select the VPC you just created from the **VPC ID** dropdown.
3. For each subnet, fill in a unique **Subnet name**, choose an **Availability Zone**, and give it a unique **IPv4 CIDR block** (e.g., `10.0.1.0/24`, `10.0.2.0/24`, etc.).
4. Click **Add new subnet** and repeat until all 6 subnets are added.
5. Click **Create subnet**.

### 2.3 Mark Two Subnets as Public

By default, all subnets are "private" in behavior. We now flag the two web-tier subnets to auto-assign public IPs:

1. Select the checkbox next to a subnet you want to make public.
2. Click **Actions → Edit subnet settings**.
3. Under **Auto-assign IP settings**, check **Enable auto-assign public IPv4 address**.
4. Click **Save**.
5. Repeat for the second public subnet.

> This setting alone does **not** make a subnet public — the routing table (done later) is what truly makes it public.

### 2.4 Create an Internet Gateway

An **Internet Gateway (IGW)** allows two-way communication between your public subnets and the internet.

1. In the left sidebar, click **Internet gateways → Create internet gateway**.
2. Give it a name and click **Create internet gateway**.
3. Select it, click **Actions → Attach to VPC**, and attach it to your VPC.

### 2.5 Create Two NAT Gateways

A **NAT Gateway** lets private resources (like app servers) reach the internet (for updates, API calls) **without** allowing inbound traffic from the internet. We create **two**, one per public subnet, for high availability.

1. In the left sidebar, click **NAT gateways → Create NAT gateway**.
2. Give it a name.
3. Under **Subnet**, choose one of your **public** subnets.
4. Leave **Connectivity type** as **Public**.
5. Click **Allocate Elastic IP**.
6. Click **Create NAT gateway**.
7. Repeat this process for the second public subnet.

### 2.6 Create Route Tables

We need **3 route tables**:

1. **Public Route Table** — sends internet traffic (`0.0.0.0/0`) to the **Internet Gateway**. Attached to both public subnets.
2. **Private Route Table AZ1** — sends internet traffic to **NAT Gateway 1**. Attached to the private App subnet in AZ1.
3. **Private Route Table AZ2** — sends internet traffic to **NAT Gateway 2**. Attached to the private App subnet in AZ2.

Steps for each route table:

1. Go to **Route tables → Create route table**.
2. Give it a name, select your VPC, click **Create route table**.
3. Repeat to create all three.

**Configure the Public Route Table:**

1. Select it, go to the **Routes** tab, click **Edit routes**.
2. Click **Add route**. Set destination to `0.0.0.0/0`, and target to **Internet Gateway** → select the one you created.
3. Click **Save changes**.
4. Go to the **Subnet associations** tab → **Edit subnet associations**.
5. Select both **public** subnets and click **Save associations**.

**Configure Private Route Table AZ1:**

1. Go to **Routes → Edit routes**.
2. Add route `0.0.0.0/0` → target **NAT Gateway** → select the NAT Gateway in AZ1.
3. Save changes.
4. Go to **Subnet associations → Edit subnet associations**, select the private **App** subnet in AZ1, and save.

**Configure Private Route Table AZ2:** Repeat the same, but use NAT Gateway AZ2 and the private App subnet in AZ2.

---

## Step 3: Create an IAM Role for EC2

This role will be attached to your EC2 servers so they can safely download code from S3 and be managed via Session Manager (no need for SSH keys).

1. Search for **IAM** in the AWS console and open it.
2. Click **Roles → Create role**.
3. Trusted entity type: **AWS service**. Use case: **EC2**. Click **Next**.
4. In the permissions search box, type `s3readonly` and check **AmazonS3ReadOnlyAccess**.
5. Clear the search, type `ssmmanaged`, and check **AmazonSSMManagedInstanceCore**.
6. Click **Next**.
7. Give the role a name, e.g., `three-tier-ec2-role`.
8. Confirm both policies are listed, then click **Create role**.

---

## Step 4: Create the RDS (Aurora MySQL) Database

1. Search for **RDS** and open the service.
2. Click **Databases → Create database**.
3. Choose **Easy create** (cheaper, though it deploys into the *default* VPC, not the one we built — we'll fix connectivity below).
4. Engine type: **Aurora (MySQL Compatible)**.
5. DB instance size: **Dev/Test**.
6. Enter a **DB cluster identifier** (e.g., `three-tier-db`).
7. Enter a **Master username** (write this down!).
8. Under **Credentials management**, choose **Self managed**.
9. Uncheck **Auto generate password**, and set your own **Master password** (write this down too!).
10. Click **Create database**.

Once created, click on the database and note down these **5 important values**:

- **Endpoint** (hostname to connect to)
- **Port** (default `3306`)
- **VPC ID** (the *default* VPC, different from yours)
- **Availability Zone**
- **VPC security group**

### 4.1 Allow Inbound MySQL Traffic

1. Click the security group name under **VPC security groups**.
2. Click **Edit inbound rules → Add rule**.
3. Type: **MYSQL/Aurora** (auto-fills port `3306`).
4. Source: **Anywhere-IPv4** (`0.0.0.0/0`) — *for learning purposes only; not recommended for production*.
5. Click **Save rules**.

> **Security note:** In a real production system, you should restrict this to only allow traffic from the application tier's security group. We open it to "Anywhere" here only because the database sits in a different VPC and we haven't linked the two yet.

### 4.2 Connect the Two VPCs with VPC Peering

Since RDS (Easy create) lives in the **default VPC**, and your app servers live in **your own VPC**, we must link them.

1. Go back to the **VPC console → Peering connections → Create peering connection**.
2. Give it a name.
3. **VPC ID (Requester)**: select the **default VPC**.
4. **VPC ID (Accepter)**: select **your VPC**.
5. Click **Create peering connection**.
6. Select the new connection → **Actions → Accept request**.

### 4.3 Find the Database Subnet's CIDR

1. Go to **Subnets** in the VPC console.
2. Find the subnet in the same Availability Zone as your RDS instance (from the info you noted in Step 4).
3. Note its **IPv4 CIDR** (e.g., `172.31.16.0/20`).

### 4.4 Update Route Tables for Peering

**On your App Tier private route tables (both AZ1 and AZ2):**

1. Select the route table → **Routes → Edit routes → Add route**.
2. Destination: the RDS subnet's CIDR (from 4.3). Target: **Peering Connection** → select yours.
3. Save changes. Repeat for the second App Tier route table.

**On the default VPC's route table:**

1. Find the route table belonging to the **default VPC**.
2. Go to **Routes → Edit routes → Add route**.
3. Destination: your App Tier subnet 1 CIDR. Target: **Peering Connection**.
4. Add another route for App Tier subnet 2 CIDR too.
5. Save changes.
6. Go to **Subnet associations → Edit subnet associations**, select the RDS subnet, and save.

Now your app servers and database can talk to each other across the two VPCs.

---

## Step 5: Create Security Groups

Security groups act like firewalls attached to your resources, controlling exactly what traffic is allowed in and out.

Create **4 security groups**, each with one inbound rule:

| Security Group | Allowed Traffic | Port | Source |
|---|---|---|---|
| External ALB SG | HTTP | 80 | Anywhere (0.0.0.0/0) |
| Web Tier SG | HTTP | 80 | External ALB SG |
| Internal ALB SG | HTTP | 80 | Web Tier SG |
| App Tier SG | Custom TCP | 4000 | Internal ALB SG |

> **Optional improvement (recommended):** Also create a **Database Tier SG** that only allows MySQL/Aurora traffic (port 3306) from the App Tier SG. This is more secure than allowing "Anywhere" as we did in Step 4 — use it if your RDS instance is in the same VPC (e.g., if you choose Standard create instead of Easy create).

Steps to create each one:

1. Go to **EC2 → Security groups → Create security group**.
2. Give it a clear name and description.
3. Select your VPC.
4. Under **Inbound rules**, click **Add rule**, set the Type/Port/Source as per the table above.
5. Leave outbound rules as default.
6. Click **Create security group**.

Repeat for all four.

---

## Step 6: Create Load Balancers and Target Groups

We need **two Application Load Balancers (ALBs)**:

- **External ALB** — internet-facing, receives traffic from users, forwards to Web Tier.
- **Internal ALB** — internal only, receives traffic from Web Tier, forwards to App Tier.

### 6.1 Create the External Load Balancer

1. Go to **EC2 → Load Balancers → Create load balancer**.
2. Choose **Application Load Balancer**.
3. Name it, keep **Scheme** as **Internet-facing**, IP type **IPv4**.
4. Under **Network mapping**, select your VPC, check both Availability Zones, and pick the **public** subnets.
5. Under **Security groups**, remove the default one, and select the **External ALB SG**.
6. Under **Listeners and routing**, keep listener as **HTTP:80**.
7. Click **Create target group** (opens a new tab):
   - Target type: **Instances**
   - Name it (e.g., `web-tier-tg`)
   - Protocol : Port → **HTTP : 80**
   - IP address type: IPv4, select your VPC
   - Protocol version: HTTP1
   - Health check protocol: HTTP
   - Health check path: `/health`
   - Click **Next**, skip registering targets for now, click **Create target group**.
8. Back in the load balancer tab, select this new target group from the dropdown (refresh if needed).
9. Click **Create load balancer**.

### 6.2 Create the Internal Load Balancer

Repeat the same process, with these differences:

- **Scheme**: **Internal**
- **Subnets**: choose the **private App Tier subnets**
- **Security group**: **Internal ALB SG**
- **Target group**: Protocol : Port → **HTTP : 4000** (since Node.js app runs on port 4000), name it something like `app-tier-tg`

Once both are created, go to **Load Balancers** and **note down the DNS name of each** — you'll need them in the next steps.

---

## Step 7: Create an S3 Bucket for Application Code

1. Search for **S3** and open the service.
2. Click **Create bucket**.
3. Give it a **globally unique name** (e.g., `yourname-three-tier-app-code`).
4. Leave everything else default, click **Create bucket**.

### 7.1 Edit the Code Before Uploading

Go into the `application-code` folder you downloaded in Step 1:

**a) Edit `nginx.conf` (inside the web-tier folder path):**

1. Open it with a text editor.
2. Find the line that sets the proxy destination for the internal ALB.
3. Replace the placeholder with the **DNS name of your Internal Load Balancer** (from Step 6.2).
4. Save the file.

**b) Edit `DbConfig.js` (inside the app-tier folder):**

1. Open it with a code editor.
2. Fill in the following values inside the quotes:
   - `DB_HOST`: the RDS **Endpoint** (from Step 4)
   - `DB_USER`: the **Master username** you set
   - `DB_PWD`: the **Master password** you set
   - `DB_DATABASE`: a database name of your choice (you'll create it with this exact name later)
3. Save the file.

### 7.2 Upload to S3

1. Go back to your S3 bucket, click **Upload**.
2. Drag and drop the whole **`application-code`** folder (or click **Add folder**).
3. Click **Upload**.

---

## Step 8: Create the Application (Backend) Servers

### 8.1 Launch a Base EC2 Instance

1. Go to **EC2 → Instances → Launch instances**.
2. Name it (e.g., `app-tier-server`).
3. AMI: keep default **Amazon Linux 2023 AMI**.
4. Instance type: keep default **t2.micro**.
5. Key pair: select **Proceed without a key pair** (we'll use Session Manager instead of SSH — more secure).
6. Click **Edit** under **Network settings**:
   - VPC: your VPC
   - Subnet: one of the **private App Tier subnets**
   - Auto-assign public IP: **Disable**
   - Firewall: select existing **App Tier SG**
7. Expand **Advanced details**, and under **IAM instance profile**, choose the role from Step 3.
8. Click **Launch instance**.

### 8.2 Connect and Configure the Server

1. Select the instance, click **Connect**.
2. Go to the **Session Manager** tab, click **Connect**.
3. If you see an error like "SSM Agent is not online," refresh the page. If it keeps happening, double-check your security groups and route tables.

Run the following commands **one at a time** in the shell:

```bash
# =========================================
# COMMANDS TO RUN IN THE APPLICATION SERVER
# =========================================

sudo -su ec2-user

sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm

sudo dnf install mysql80-community-release-el9-1.noarch.rpm -y

sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023

sudo dnf install mysql-community-client -y

mysql --version

# Test connection between the app server and the database
mysql -h <RDS-Database-Instance-Endpoint> -u <username> -p
# (press Enter, then type your password)
```

Once connected to MySQL, run:

```sql
-- Create the database (use the same name you put in DbConfig.js)
CREATE DATABASE <DB_DATABASE>;

-- Confirm it was created
SHOW DATABASES;

-- Switch to using it
USE <DB_DATABASE>;

-- Create a table
CREATE TABLE IF NOT EXISTS myexpenses (
    id INT NOT NULL AUTO_INCREMENT,
    amount DECIMAL(10,2),
    description VARCHAR(100),
    PRIMARY KEY(id)
);

-- Confirm the table exists
SHOW TABLES;

-- Insert a sample row
INSERT INTO myexpenses (amount, description) VALUES ('5000', 'clothes');

-- Confirm the row is there
SELECT * FROM myexpenses;
```

> Note: The original tutorial had a typo `VARCAHR(100)` — the correct spelling is **`VARCHAR(100)`**, used above.

Exit MySQL (type `exit`), then continue:

```bash
#===============================
# COPYING CONTENT FROM S3 BUCKET
#===============================
cd /home/ec2-user

# Replace with your actual S3 bucket name
sudo aws s3 cp s3://<YOUR-S3-BUCKET-NAME>/application-code/app-tier app-tier --recursive

cd app-tier

sudo chown -R ec2-user:ec2-user /home/ec2-user/app-tier

sudo chmod -R 755 /home/ec2-user/app-tier

#===============================
# INSTALLING NODEJS
#===============================

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

source ~/.bashrc

nvm install 16

nvm use 16

npm install -g pm2

npm install

npm audit fix

#===============================
# STARTING THE APP WITH PM2
#===============================

# Start the app (PM2 keeps Node.js apps running in the background)
pm2 start index.js

# Set PM2 to start automatically when the server reboots
pm2 startup

sudo env PATH=$PATH:/home/ec2-user/.nvm/versions/node/v16.20.2/bin /home/ec2-user/.nvm/versions/node/v16.20.2/lib/node_modules/pm2/bin/pm2 startup systemd -u ec2-user --hp /home/ec2-user

# Save the current process list
pm2 save

# Test the health check endpoint
curl http://localhost:4000/health
```

If you see a healthy response, everything worked. If not, re-check each command carefully.

Click **Terminate** to leave the Session Manager session (this closes the session, **not** the EC2 instance).

### 8.3 Create an AMI (Blueprint) From This Server

1. Select the instance checkbox → **Actions → Image and templates → Create image**.
2. Give it a name (e.g., `app-tier-ami`).
3. Click **Create image**.

### 8.4 Create a Launch Template

1. Go to **Launch Templates → Create launch template**.
2. Name it, add a version description.
3. Enable **Auto Scaling guidance**.
4. Under **Application and OS Images**, click **My AMIs** and select your new AMI.
5. Instance type: **t2.micro**.
6. Leave Key pair and Subnet unset (the Auto Scaling Group will handle subnets).
7. Select the **App Tier SG**.
8. Click **Create launch template**.

### 8.5 Create an Auto Scaling Group

1. Go to **Auto Scaling Groups → Create Auto Scaling group**.
2. Name it, select the launch template you just made. Click **Next**.
3. Under **Network**, select your VPC, both Availability Zones, and the **private App Tier subnets**. Click **Next**.
4. Under **Load balancing**, choose **Attach to an existing load balancer**, and select the **App Tier target group** (`app-tier-tg`). Click **Next**.
5. Set:
   - **Desired capacity**: 2
   - **Min desired capacity**: 2
   - **Max desired capacity**: 4
6. Under **Automatic scaling**, choose **Target tracking scaling policy**.
7. Metric type: **Average CPU utilization**.
8. Target value: `50`, Instance warmup: `30` seconds.
9. Click through to review, then **Create Auto Scaling group**.

Go back to **Instances** — you should now see **2 new app-tier instances** launching automatically.

---

## Step 9: Create the Web (Frontend) Servers

Repeat a very similar process for the Web Tier.

### 9.1 Launch a Base EC2 Instance

1. **Launch instances**, name it (e.g., `web-tier-server`).
2. AMI: default Amazon Linux 2023. Instance type: t2.micro.
3. Key pair: **Proceed without a key pair**.
4. Edit Network settings:
   - VPC: your VPC
   - Subnet: one of the **public** Web Tier subnets
   - Auto-assign public IP: **Enable**
   - Firewall: select the **Web Tier SG**
5. Under **Advanced details**, choose the IAM role from Step 3.
6. Click **Launch instance**.

### 9.2 Connect and Configure the Server

Connect via **Session Manager**, then run:

```bash
# =========================================
# COMMANDS TO RUN IN THE WEB SERVER
# =========================================

sudo -su ec2-user

cd /home/ec2-user

# Replace with your actual S3 bucket name
sudo aws s3 cp s3://<YOUR-S3-BUCKET-NAME>/application-code/web-tier web-tier --recursive

sudo chown -R ec2-user:ec2-user /home/ec2-user

sudo chmod -R 755 /home/ec2-user

# =========================================
# INSTALLING NODEJS (needed to build the React app)
# =========================================

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

source ~/.bashrc

nvm install 16

nvm use 16

cd /home/ec2-user/web-tier

npm install

# =========================================
# BUILDING THE APP FOR PRODUCTION
# =========================================

npm run build

# =========================================
# INSTALLING NGINX (WEB SERVER SOFTWARE)
# =========================================

sudo yum install nginx -y

cd /etc/nginx

sudo mv nginx.conf nginx-backup.conf

# Replace with your actual S3 bucket name
sudo aws s3 cp s3://<YOUR-S3-BUCKET-NAME>/application-code/nginx.conf .

sudo chmod -R 755 /home/ec2-user

sudo service nginx restart

sudo chkconfig nginx on
```

Click **Terminate** to leave the Session Manager session.

### 9.3 Create an AMI, Launch Template, and Auto Scaling Group

Repeat the exact same steps as in **8.3, 8.4, and 8.5**, but for the Web Tier:

- AMI name: e.g., `web-tier-ami`
- Launch template security group: **Web Tier SG**
- Auto Scaling Group subnets: **public Web Tier subnets**
- Target group: the External ALB's target group (`web-tier-tg`)
- Desired/Min/Max capacity: 2 / 2 / 4 (same as before)

Go to **Instances** — you should now see 2 more instances launching for the Web Tier.

---

## Step 9.5: Test Your Application (Before Adding a Domain)

1. Go to **Load Balancers**, copy the **DNS name of the External Load Balancer**.
2. Paste it into your browser and press Enter.
3. You should see your web application load successfully.
4. Test each button on the page, and specifically try the **DB Demo** page — add and delete values to confirm the database connection works end-to-end.

If this works, your entire 3-tier architecture is functioning. Now let's add a proper domain name.

---

## Step 10: Set Up a Custom Domain (Using DigitalPlat Instead of GoDaddy)

Instead of GoDaddy, this guide uses **DigitalPlat**, a registrar that offers **free domains**, which is great for students and personal projects.

👉 Domain dashboard: **https://dash.domain.digitalplat.org/domains**

### 10.1 Register a Free Domain on DigitalPlat

1. Go to **https://dash.domain.digitalplat.org/domains**.
2. Sign up for an account if you don't already have one, then log in.
3. Use the domain search tool to search for a domain name you like.
4. If it's available, select it and follow the on-screen steps to **register it for free**.
5. Complete any required verification (email confirmation, etc.) to activate the domain.
6. Once registered, go to your **Domain List** / **Dashboard**, and find the **DNS Management** or **DNS Records** section for your new domain. This is where we will add records, similar to how you would in GoDaddy.

### 10.2 Validate Domain Ownership with AWS Certificate Manager (ACM)

We need an SSL/TLS certificate so our site can use HTTPS.

1. Go to the AWS Console, search for **ACM** (AWS Certificate Manager).
2. Make sure your region is set to **N. Virginia (us-east-1)** — ACM certificates used with Application Load Balancers must be requested in this region (or the same region as your ALB, but us-east-1 is required if you plan to also use CloudFront).
3. Click **Request a certificate**.
4. Keep **Request a public certificate** selected, click **Next**.
5. Under **Fully qualified domain name**, enter your domain with a wildcard, e.g., `*.yourdomain.com`.
6. Validation method: **DNS validation**.
7. Leave the rest as default and click **Request**.
8. Click **List certificates**, select your new certificate.
9. Under the **Domains** section, AWS will show you a **CNAME name** and **CNAME value** — copy both of these.

### 10.3 Add the Validation CNAME Record in DigitalPlat

1. Log in to your **DigitalPlat dashboard**: https://dash.domain.digitalplat.org/domains
2. Open your domain's **DNS Management / DNS Records** page.
3. Click **Add Record** (or similar button).
4. Set **Type** to **CNAME**.
5. In the **Name/Host** field, paste the CNAME name from ACM — but **remove the part that repeats your domain name** at the end.
   - Example: if ACM gives you `_123456abcdef.yourdomain.com.`, only enter `_123456abcdef` in the Name field.
6. In the **Value/Points to** field, paste the CNAME value from ACM — **remove the trailing dot** at the very end.
7. Leave **TTL** as default and **Save**.

Wait a few minutes. AWS will automatically detect the record and validate your domain. Once done, your ACM certificate status will change to **Issued**.

### 10.4 Point Your Domain to the Load Balancer

1. In DigitalPlat's DNS Management page, click **Add Record** again.
2. Type: **CNAME**.
3. **Name/Host**: choose a subdomain, e.g., `webapp` (so your site becomes `webapp.yourdomain.com`).
4. **Value/Points to**: paste the **DNS name of your External Load Balancer** (from Step 6.1).
5. Leave TTL as default and **Save**.

Wait a few minutes for DNS to propagate.

### 10.5 (Optional but Recommended) Enable HTTPS on the Load Balancer

To actually serve your site securely over `https://`, add an HTTPS listener to your External Load Balancer:

1. Go to **EC2 → Load Balancers**, select your External Load Balancer.
2. Go to the **Listeners** tab, click **Add listener**.
3. Protocol: **HTTPS**, Port: **443**.
4. Under **Default SSL/TLS certificate**, select the ACM certificate you created (make sure it shows status **Issued**).
5. Forward it to your **Web Tier target group**.
6. Save.
7. (Optional) Edit the existing HTTP:80 listener to **redirect** to HTTPS:443, so all traffic is automatically secured.

### 10.6 Visit Your Website

Open your browser and go to: `https://webapp.yourdomain.com` (replace with your actual subdomain and domain).

🎉 If everything is configured correctly, your website will load using your own custom domain name, with a valid SSL certificate.

---

## Congratulations! 🎉

You have successfully deployed a complete, production-style **3-tier architecture on AWS**, including:

- A custom VPC with public and private subnets across two Availability Zones
- Internet Gateway and NAT Gateways for secure internet access
- Layered security groups following the least-privilege principle
- An Aurora MySQL database
- Auto Scaling web and application tiers behind two load balancers
- Application code stored and deployed from S3
- A live custom domain with HTTPS, registered through DigitalPlat

---

## Common Mistakes to Watch Out For

- **Forgetting to update `nginx.conf` and `DbConfig.js` before uploading to S3** — the app won't connect properly if these still contain placeholder values.
- **Using the wrong subnets** for Auto Scaling Groups (mixing up public/private).
- **Security group misconfiguration** — double-check that each SG only allows the correct source SG, not "Anywhere," except for the External ALB.
- **Forgetting to remove the trailing dot** from the ACM CNAME value when pasting into DNS records.
- **SSM Agent not online** — usually caused by missing NAT Gateway routes, an incorrect IAM role, or the wrong security group blocking outbound traffic. Recheck Steps 2 and 3.
- **Database connection errors** — double-check the RDS Endpoint, username, and password in `DbConfig.js`, and confirm VPC Peering routes are set up correctly (Step 4).
- **ACM certificate stuck at "Pending validation"** — this almost always means the CNAME record in DigitalPlat wasn't entered correctly (extra dots, wrong name field, or DNS hasn't propagated yet — wait 15–30 minutes and check again).

## Suggested Improvements for a Real Production Setup

- Use **Standard create** for RDS instead of Easy create, so the database lives inside your own VPC (removing the need for VPC Peering) and supports **Multi-AZ** for automatic failover.
- Add a **dedicated Database security group** that only accepts traffic from the App Tier SG on port 3306, instead of allowing "Anywhere."
- Store database credentials in **AWS Secrets Manager** instead of hardcoding them in `DbConfig.js`.
- Enable **CloudWatch Alarms and Dashboards** for proactive monitoring.
- Set up **AWS WAF** on the External Load Balancer to protect against common web attacks.
- Use **Infrastructure as Code** (e.g., Terraform or AWS CloudFormation) to make this whole setup repeatable and version-controlled.

---

*Happy building!* 🚀
