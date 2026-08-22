# 14 AWS Projects for Beginners — A Complete Step-by-Step Guide

This guide walks you through **14 hands-on AWS projects**, explained in simple, beginner-friendly steps. Each project is self-contained, so you can do them in any order, but doing them in the order listed below will help you build knowledge progressively — starting with cost control, then compute, containers, databases, storage, messaging, serverless, content delivery, identity, compliance, and infrastructure as code.

By the end, you'll have hands-on experience with: **CloudWatch, AWS Budgets, EC2, ECR, Docker, AWS CLI, RDS, DynamoDB, S3, SNS, Lambda, CloudFront, IAM, AWS Config, and CloudFormation.**

---

## Before You Start — Prerequisites

- An **AWS account** (the AWS Free Tier covers most of these projects, but a few — like RDS SQL Server and NAT-heavy setups — may incur small charges if left running).
- A **credit/debit card** on file (required by AWS even for free tier).
- A computer with a web browser and, for some projects, a terminal (Mac/Linux) or PowerShell/PuTTY (Windows).
- Basic comfort clicking around a web console — no prior AWS experience needed.

> ⚠️ **Very important — Cleanup matters:** AWS charges you for resources that are *running*, even if you're not using them. At the end of **every single project**, we've included a **Cleanup** section — please follow it every time to avoid unexpected charges. Project 1 and 2 (Billing Alarm and Budget) will help you catch it early if something is left running.

---

## Project 1: Create Three Billing Alarms

### Overview

Set up three billing alarms — at **$5**, **$25**, and **$100** — so you get an email notification if your AWS spending crosses these thresholds. This is the very first thing every beginner should set up, **before** touching any other AWS service.

### What You'll Learn

- How to monitor your AWS spending in real time.
- How to respond to alerts (e.g., if you hit the $5 alarm, review what's running and shut it down if needed).

### Service Used

**AWS CloudWatch** (for monitoring and creating alarms) + **AWS Billing** (to enable billing alerts).

### Step-by-Step Instructions

#### Step 1.1: Enable Billing Alerts

1. Sign in to the **AWS Console** as the **root user** or an IAM user with billing permissions.
2. Click your account name (top-right corner) → **Billing and Cost Management**.
3. In the left sidebar, click **Billing Preferences** (or **Billing Preferences** under **Preferences**).
4. Check the box **Receive CloudWatch Billing Alerts**.
5. Click **Save preferences**.

> This step is required — without it, CloudWatch cannot see your billing data.

#### Step 1.2: Switch to US East (N. Virginia) Region

Billing metrics are **only available in the `us-east-1` (N. Virginia)** region, regardless of where your other resources live.

1. Click the region dropdown (top-right).
2. Select **US East (N. Virginia)**.

#### Step 1.3: Create the $5 Billing Alarm

1. Search for **CloudWatch** in the search bar and open it.
2. In the left sidebar, click **Alarms → Billing** (or **All alarms → Create alarm**).
3. Click **Create alarm**.
4. Click **Select metric**.
5. Choose **Billing → Total Estimated Charge**.
6. Select the checkbox next to **USD**, then click **Select metric**.
7. Under **Conditions**:
   - Threshold type: **Static**
   - Whenever EstimatedCharges is: **Greater than**
   - Enter the threshold value: **5**
8. Click **Next**.
9. Under **Notification**:
   - Alarm state trigger: **In alarm**
   - Select **Create new topic**.
   - Give the topic a name (e.g., `billing-alarm-5-dollars`).
   - Enter your **email address**.
   - Click **Create topic**.
10. Click **Next**.
11. Give the alarm a name (e.g., `Billing-Alarm-5-USD`) and a description.
12. Click **Next**, review, then click **Create alarm**.
13. **Check your email inbox** — AWS SNS will send you a confirmation email. Click **Confirm subscription** in that email, or you won't receive alerts.

#### Step 1.4: Create the $25 Billing Alarm

Repeat **Step 1.3**, but:
- Threshold value: **25**
- You can reuse the same SNS topic (select **Select an existing topic** and choose the one you created) or create a new one.
- Name the alarm `Billing-Alarm-25-USD`.

#### Step 1.5: Create the $100 Billing Alarm

Repeat the same process again:
- Threshold value: **100**
- Name the alarm `Billing-Alarm-100-USD`.

### Cleanup

- These alarms cost nothing to keep running — you can leave them active permanently. No cleanup needed.

---

## Project 2: Create a Cost Budget

### Overview

Use **AWS Budgets** to set a monthly spending limit and get alerted as you approach or exceed it — this is more flexible than billing alarms because it can track forecasted spend too.

### What You'll Learn

- How different AWS services contribute to your overall bill.
- How to proactively catch cost overruns before they happen.

### Service Used

**AWS Budgets**

### Step-by-Step Instructions

1. Search for **Billing and Cost Management** in the console search bar.
2. In the left sidebar, click **Budgets**.
3. Click **Create budget**.
4. Choose a budget type: select **Use a template (simplified)**, then pick **Monthly cost budget**, **or** choose **Customize (advanced)** for more control. For beginners, the template option is easiest.
5. Enter your **Budgeted amount** (e.g., `$10`).
6. Under **Email recipients**, enter your email address.
7. Click **Create budget**.

If you chose **Customize (advanced)** instead, you'll additionally be able to:
- Set multiple **alert thresholds** (e.g., alert at 80% of budget, and again at 100%).
- Choose whether to track **actual** spend or **forecasted** spend.

### Verify It Works

1. Go back to **Budgets** — you should see your new budget listed with a progress bar showing $0 / $10 spent (or whatever you set).
2. Wait for AWS to send you an alert email once your spending crosses the threshold (this can take up to 24 hours to reflect the first time).

### Cleanup

- Budgets don't cost anything to keep active. No cleanup needed. You can delete it later via **Budgets → select your budget → Delete** if you want.

---

## Project 3: Launch a "Hello World" Website on the Internet

### Overview

Launch a Linux EC2 instance in a public subnet, install a web server, and serve a simple "Hello, World!" webpage accessible from any browser on the internet.

### What You'll Learn

- The basics of EC2 instances and how they work.
- How to configure a web server (Apache) to serve static files.
- How to connect to a server using SSH.

### Service Used

**Amazon EC2**

### Step-by-Step Instructions

#### Step 3.1: Launch the EC2 Instance

1. Search for **EC2** and open the service.
2. Click **Launch instance**.
3. **Name**: `hello-world-server`.
4. **AMI**: choose **Amazon Linux 2023 AMI** (free tier eligible).
5. **Instance type**: keep **t2.micro** (free tier eligible).
6. **Key pair**: click **Create new key pair**.
   - Name it (e.g., `hello-world-key`).
   - Key pair type: **RSA**.
   - Private key file format: **.pem** (for Mac/Linux/PuTTY-compatible) or **.ppk** (for PuTTY on Windows only).
   - Click **Create key pair** — this downloads the private key file. **Save it somewhere safe; you cannot download it again.**
7. Under **Network settings**, click **Edit**:
   - Make sure **Auto-assign public IP** is **Enabled**.
   - Under **Firewall (security groups)**, choose **Create security group**.
   - Add these inbound rules:
     - **SSH**, Port 22, Source: **My IP** (this auto-fills your current IP — safer than "Anywhere").
     - **HTTP**, Port 80, Source: **Anywhere (0.0.0.0/0)**.
     - **HTTPS**, Port 443, Source: **Anywhere (0.0.0.0/0)**.
8. Leave everything else as default, and click **Launch instance**.
9. Wait 1–2 minutes for the instance state to become **Running**.

#### Step 3.2: Connect to the Instance via SSH

**On Mac/Linux:**

1. Open Terminal, navigate to where you saved your `.pem` file.
2. Restrict its permissions (required by SSH):
   ```bash
   chmod 400 hello-world-key.pem
   ```
3. Get your instance's **Public IPv4 address** from the EC2 console (select the instance, look under the **Details** tab).
4. Connect:
   ```bash
   ssh -i hello-world-key.pem ec2-user@<YOUR-PUBLIC-IP>
   ```
5. Type `yes` if prompted about the host's authenticity.

**On Windows (using PuTTY):**

1. Download and install **PuTTY** and **PuTTYgen** if you don't have them.
2. Open **PuTTYgen**, click **Load**, select your `.pem` file (change file filter to "All Files" to see it).
3. Click **Save private key** to convert it to `.ppk` format (click "Yes" to save without a passphrase, for simplicity).
4. Open **PuTTY**:
   - Host Name: `ec2-user@<YOUR-PUBLIC-IP>`
   - Port: `22`
   - In the left tree: **Connection → SSH → Auth → Credentials**, browse and select your `.ppk` file.
   - Click **Open**, accept the security alert.

#### Step 3.3: Install and Configure the Web Server

Once connected via SSH, run these commands one by one:

```bash
# Update the system
sudo yum update -y

# Install Apache web server
sudo yum install -y httpd

# Start the web server
sudo systemctl start httpd

# Enable it to start automatically on reboot
sudo systemctl enable httpd

# Confirm it's running
sudo systemctl status httpd
```

#### Step 3.4: Add the "Hello, World!" Page

```bash
# Create a simple HTML page with a Hello World header
echo "<h1>Hello, World!</h1>" | sudo tee /var/www/html/index.html
```

Verify the file was created correctly:
```bash
cat /var/www/html/index.html
```

#### Step 3.5: Access the Website from Your Browser

1. Go back to the EC2 console, copy the instance's **Public IPv4 address**.
2. Open a browser and go to:
   ```
   http://<YOUR-PUBLIC-IP>
   ```
   > **Tip:** Use `http://`, **not** `https://` — you haven't configured SSL, so HTTPS won't work and will show a connection error.
3. You should see: **Hello, World!**

### Cleanup

1. Go to **EC2 → Instances**.
2. Select `hello-world-server`.
3. Click **Instance state → Terminate instance**.
4. Confirm termination.
5. (Optional) Delete the security group you created if it's not used elsewhere: **EC2 → Security Groups → select it → Actions → Delete**.

---

## Project 4: Push a Docker Image to Amazon ECR

### Overview

Build a simple Docker image and push it to **Amazon Elastic Container Registry (ECR)**, a fully managed private container registry.

### What You'll Learn

- Basic Docker concepts: images, containers, tags.
- Hands-on CLI experience authenticating and pushing images to ECR.

### Services/Tools Used

**Amazon ECR**, **Docker**, **AWS CLI**

### Prerequisites

- Install **Docker Desktop** (Mac/Windows) or **Docker Engine** (Linux) — https://docs.docker.com/get-started/get-docker/
- Install and configure the **AWS CLI** (see Project 7 below if you haven't already done this).

### Step-by-Step Instructions

#### Step 4.1: Create a Simple Docker Image

1. On your local machine, create a new folder:
   ```bash
   mkdir hello-docker && cd hello-docker
   ```
2. Create a file named `Dockerfile` (no extension) with this content:
   ```dockerfile
   FROM nginx:alpine
   COPY index.html /usr/share/nginx/html/index.html
   ```
3. Create a file named `index.html`:
   ```html
   <!DOCTYPE html>
   <html>
     <head><title>Hello from Docker</title></head>
     <body><h1>Hello, World from my Docker container!</h1></body>
   </html>
   ```
4. Build the Docker image:
   ```bash
   docker build -t hello-docker-app .
   ```
5. (Optional) Test it locally:
   ```bash
   docker run -d -p 8080:80 hello-docker-app
   ```
   Visit `http://localhost:8080` in your browser to confirm it works. Then stop it:
   ```bash
   docker ps
   docker stop <container-id>
   ```

#### Step 4.2: Create Your ECR Repository

1. Search for **ECR** (Elastic Container Registry) in the AWS console.
2. Click **Create repository**.
3. Visibility settings: **Private**.
4. Repository name: `hello-docker-app`.
5. Leave other settings as default, click **Create repository**.
6. Once created, click on the repository and copy the **URI** shown (e.g., `123456789012.dkr.ecr.us-east-1.amazonaws.com/hello-docker-app`).

#### Step 4.3: Authenticate Docker to Your ECR Registry

In your terminal, run (replace the region and account ID with your own):

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <YOUR-ACCOUNT-ID>.dkr.ecr.us-east-1.amazonaws.com
```

You should see: `Login Succeeded`.

#### Step 4.4: Tag Your Image for ECR

```bash
docker tag hello-docker-app:latest <YOUR-ACCOUNT-ID>.dkr.ecr.us-east-1.amazonaws.com/hello-docker-app:latest
```

#### Step 4.5: Push the Image to ECR

```bash
docker push <YOUR-ACCOUNT-ID>.dkr.ecr.us-east-1.amazonaws.com/hello-docker-app:latest
```

#### Step 4.6: Verify

1. Go back to the ECR console, click on your repository.
2. Under **Images**, you should see your `latest` tagged image with its size and push date.

### Troubleshooting

- **`no basic auth credentials`**: your Docker login expired or failed — repeat Step 4.3 (ECR tokens expire after 12 hours).
- **`denied: User is not authorized`**: check your IAM user/role has the `AmazonEC2ContainerRegistryFullAccess` permission (or equivalent).

### Cleanup

1. Go to **ECR → Repositories → select `hello-docker-app`**.
2. Click **Delete**, type the repository name to confirm, and click **Delete**.

---

## Project 5: Create an Amazon RDS DB Instance (MS SQL Server)

### Overview

Launch a fully managed Microsoft SQL Server database using Amazon RDS, and connect to it from your local machine using SQL Server Management Studio (SSMS) or Azure Data Studio.

### What You'll Learn

- How to configure an RDS database instance.
- How to set up security groups for database access.
- How to navigate SQL Server Management Studio (SSMS).

### Service Used

**Amazon RDS**

> ⚠️ **Cost warning:** SQL Server instances (even `db.t3.micro`) can incur charges depending on the SQL Server edition and license. Use **SQL Server Express Edition** (free tier eligible in supported regions) and remember to delete the instance when done (see Cleanup).

### Step-by-Step Instructions

#### Step 5.1: Create the RDS SQL Server Instance

1. Search for **RDS** and open the service.
2. Click **Databases → Create database**.
3. Choose **Standard create** (Full configuration) for complete control.
4. **Engine type**: select **Microsoft SQL Server**.
5. **Edition**: select **SQL Server Express Edition** (free tier eligible).
6. Under **Templates**, select **Free tier** if available.
7. **Settings**:
   - DB instance identifier: `hello-sqlserver-db`
   - Master username: `admin` (or your choice)
   - Uncheck **Auto generate password**, and set your own master password. Write it down.
8. **Instance configuration**: keep **db.t3.micro** (free tier eligible).
9. **Storage**: keep defaults (20 GiB, General Purpose SSD).
10. **Connectivity**:
    - Compute resource: **Don't connect to an EC2 compute resource** (for now).
    - Virtual private cloud (VPC): keep default.
    - Public access: select **Yes** (so you can connect from your local machine — for learning purposes only; not recommended for production).
    - VPC security group: **Create new**, name it `sqlserver-sg`.
11. Leave other settings as default, scroll down, and click **Create database**.
12. Wait 5–10 minutes for the status to change to **Available**.

#### Step 5.2: Allow Inbound Access to the Database

1. Once available, click on the database instance.
2. Under **Connectivity & security**, click the **VPC security group** link.
3. Click **Edit inbound rules → Add rule**.
4. Type: **MS SQL**, Port: **1433** (auto-filled).
5. Source: **My IP** (safer than Anywhere).
6. Click **Save rules**.

#### Step 5.3: Note Your Connection Details

From the RDS console, under **Connectivity & security**, note down:
- **Endpoint** (hostname)
- **Port** (1433)

#### Step 5.4: Connect Using SQL Server Management Studio (SSMS)

1. Download and install **SSMS** (Windows) from Microsoft's website, or use **Azure Data Studio** (cross-platform) as an alternative.
2. Open SSMS, click **Connect → Database Engine**.
3. **Server name**: paste your RDS Endpoint (e.g., `hello-sqlserver-db.xxxxxxxx.us-east-1.rds.amazonaws.com,1433`).
4. **Authentication**: **SQL Server Authentication**.
5. **Login**: your master username.
6. **Password**: your master password.
7. Click **Connect**.

You should now see the SQL Server instance in the **Object Explorer** panel, and you can run queries, create tables, etc.

### Cleanup

1. Go to **RDS → Databases → select `hello-sqlserver-db`**.
2. Click **Actions → Delete**.
3. **Uncheck** "Create final snapshot" (unless you want to keep a backup — this incurs additional storage cost).
4. Type `delete me` to confirm.
5. Click **Delete**.

---

## Project 6: Create a DynamoDB Table

### Overview

Create a NoSQL table in **Amazon DynamoDB**, insert data, and run scan and query operations on it.

### What You'll Learn

- How to create a DynamoDB table.
- How to insert items into a table.
- How to run scans (read everything) and queries (read specific items) on a table.

### Service Used

**Amazon DynamoDB**

### Step-by-Step Instructions

#### Step 6.1: Create the Table

1. Search for **DynamoDB** and open the service.
2. Click **Create table**.
3. **Table name**: `Music`.
4. **Partition key**: `Artist` (String).
5. **Sort key**: `SongTitle` (String).
6. Under **Table settings**, choose **Customize settings**.
7. Under **Read/write capacity settings**, select **Provisioned**, and set:
   - Read capacity units: `5`
   - Write capacity units: `5`
   - (Uncheck auto scaling for simplicity, since this is a learning exercise.)
8. Leave everything else as default, click **Create table**.
9. Wait for the table **Status** to change to **Active**.

#### Step 6.2: Add Three Items to the Table

1. Click on your `Music` table.
2. Click the **Explore table items** tab.
3. Click **Create item**.
4. Add the following fields (click **Add new attribute** for each additional field):
   - `Artist`: `The Beatles`
   - `SongTitle`: `Hey Jude`
   - `Genre`: `Rock`
5. Click **Create item**.
6. Repeat to add two more items, e.g.:
   - `Artist`: `Queen`, `SongTitle`: `Bohemian Rhapsody`, `Genre`: `Rock`
   - `Artist`: `Michael Jackson`, `SongTitle`: `Thriller`, `Genre`: `Pop`

#### Step 6.3: Run a Scan (Returns All Items)

1. Still on the **Explore table items** tab, click **Scan** (this may already be the default view mode).
2. Click **Run**.
3. You should see all three items you inserted.

#### Step 6.4: Run a Query (Returns a Single Item)

1. Switch the mode from **Scan** to **Query**.
2. Under **Partition key**, enter: `The Beatles`.
3. (Optional) Under **Sort key**, choose `=` and enter: `Hey Jude`.
4. Click **Run**.
5. You should see only the one matching item returned.

### Cleanup

1. Go to **DynamoDB → Tables → select `Music`**.
2. Click **Delete**.
3. Type `confirm` and click **Delete table**.

---

## Project 7: Install & Configure AWS CLI, Then Create an S3 Bucket

### Overview

Install the AWS Command Line Interface (CLI) on your local machine, configure it with your credentials, and use it to create, list, and delete an S3 bucket.

### What You'll Learn

- How to install and configure the AWS CLI.
- How to create, list, and delete S3 buckets from the command line.

### Service Used

**Amazon S3** + **AWS CLI**

### Step-by-Step Instructions

#### Step 7.1: Create an IAM User with Programmatic Access (If You Don't Have One)

> It's best practice to **never** use your root account credentials with the CLI. Create an IAM user instead.

1. Search for **IAM** and open it.
2. Click **Users → Create user**.
3. Username: `cli-user`.
4. Click **Next**.
5. Choose **Attach policies directly**, and attach **AmazonS3FullAccess** (for this project; use more restrictive policies in real projects).
6. Click **Next → Create user**.
7. Click on the new user → **Security credentials** tab.
8. Under **Access keys**, click **Create access key**.
9. Use case: **Command Line Interface (CLI)**.
10. Check the confirmation box, click **Next → Create access key**.
11. **Copy the Access Key ID and Secret Access Key** (or download the .csv file) — you cannot view the secret key again later.

#### Step 7.2: Install the AWS CLI

**Windows:**
1. Download the AWS CLI MSI installer from: https://awscli.amazonaws.com/AWSCLIV2.msi
2. Run the installer and follow the prompts.

**Mac:**
```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

**Linux:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verify installation:
```bash
aws --version
```

#### Step 7.3: Configure the AWS CLI

```bash
aws configure
```

You'll be prompted for:
```
AWS Access Key ID [None]: <paste your access key>
AWS Secret Access Key [None]: <paste your secret key>
Default region name [None]: us-east-1
Default output format [None]: json
```

#### Step 7.4: Create an S3 Bucket

> S3 bucket names must be **globally unique** across all AWS accounts.

```bash
aws s3 mb s3://<your-unique-bucket-name>
```

Example:
```bash
aws s3 mb s3://abdullah-cli-test-bucket-2026
```

#### Step 7.5: Check That the Bucket Was Created

```bash
aws s3 ls
```

You should see your new bucket listed.

#### Step 7.6: Delete the Bucket When You're Done

```bash
aws s3 rb s3://<your-unique-bucket-name>
```

> If the bucket has objects in it, this command will fail. Use `--force` to delete the bucket and all its contents:
> ```bash
> aws s3 rb s3://<your-unique-bucket-name> --force
> ```

### Cleanup

- Already handled in Step 7.6. Optionally, delete the `cli-user` IAM user and its access keys if you no longer need CLI access: **IAM → Users → cli-user → Delete**.

---

## Project 8: Create an S3 Bucket and Store an Object in It

### Overview

Use the AWS Console (not the CLI this time) to create an S3 bucket and upload a file into it.

### What You'll Learn

- What Amazon S3 is and how it works as scalable object storage.

### Service Used

**Amazon S3**

### Step-by-Step Instructions

#### Step 8.1: Create the Bucket

1. Search for **S3** and open the service.
2. Click **Create bucket**.
3. **Bucket name**: choose a globally unique name (e.g., `abdullah-my-first-bucket-2026`).
4. **AWS Region**: choose one close to you.
5. Leave **Block all public access** checked (default, and safest for this exercise).
6. Leave other settings as default.
7. Click **Create bucket**.

#### Step 8.2: Upload an Object

1. Click on your newly created bucket.
2. Click **Upload**.
3. Click **Add files**, and select any file from your computer (e.g., a photo or text file).
4. Scroll down and click **Upload**.
5. Once the upload completes, click **Close**.

#### Step 8.3: Verify

1. You should now see your file listed inside the bucket.
2. Click on the file name to see its details, including its S3 URI and Object URL.

### Cleanup

1. Open the bucket, select the uploaded file's checkbox, click **Delete**, and confirm.
2. Go back to **S3 → Buckets**, select your bucket, click **Delete**.
3. Type the bucket name to confirm, and click **Delete bucket**.

---

## Project 9: Introduction to SNS (Simple Notification Service)

### Overview

Create an SNS topic, subscribe to it with your email, and send a test notification.

### What You'll Learn

- How notifications are sent and managed for real-time updates in applications.

### Service Used

**Amazon SNS**

### Step-by-Step Instructions

#### Step 9.1: Create an SNS Topic

1. Search for **SNS** and open the service.
2. In the left sidebar, click **Topics → Create topic**.
3. Type: **Standard**.
4. Name: `hello-world-topic`.
5. Leave other settings as default, click **Create topic**.

#### Step 9.2: Subscribe to the Topic Using Your Email

1. On the topic's page, click **Create subscription**.
2. Protocol: **Email**.
3. Endpoint: enter your email address.
4. Click **Create subscription**.

#### Step 9.3: Confirm the Subscription

1. Check your email inbox — you'll receive a message titled "AWS Notification - Subscription Confirmation."
2. Click **Confirm subscription** inside that email.
3. Back in the SNS console, refresh the **Subscriptions** tab — the status should change from **Pending confirmation** to **Confirmed**.

#### Step 9.4: Send a Test Message

1. Go back to your topic's page.
2. Click **Publish message**.
3. Subject: `Test Notification`.
4. Message body: `Hello! This is a test message from SNS.`
5. Click **Publish message**.

#### Step 9.5: Verify

Check your email inbox — you should receive the test message within a few seconds.

### Cleanup

1. Go to **SNS → Subscriptions**, select your subscription, click **Delete**.
2. Go to **SNS → Topics**, select `hello-world-topic`, click **Delete**, confirm.

---

## Project 10: Create a Lambda Function to Add Two Numbers

### Overview

Create a simple AWS Lambda function using Python that takes two numbers as input, adds them, and logs the result.

### What You'll Learn

- How to create serverless functions using AWS Lambda.
- How Lambda functions receive input (events) and produce output.

### Service Used

**AWS Lambda**

### Step-by-Step Instructions

#### Step 10.1: Create the Lambda Function

1. Search for **Lambda** and open the service.
2. Click **Create function**.
3. Choose **Author from scratch**.
4. **Function name**: `AddTwoNumbers`.
5. **Runtime**: **Python 3.12** (or latest available).
6. **Architecture**: `x86_64` (default).
7. Leave **Execution role** as default (**Create a new role with basic Lambda permissions**).
8. Click **Create function**.

#### Step 10.2: Write the Function Code

1. In the **Code source** section, you'll see a default `lambda_function.py` file. Replace its content with:

```python
def lambda_handler(event, context):
    # Get the two numbers from the event input.
    # Provide default values in case they're missing, so testing is easier.
    num1 = event.get('num1', 0)
    num2 = event.get('num2', 0)

    result = num1 + num2

    print(f"Adding {num1} + {num2} = {result}")

    return {
        'statusCode': 200,
        'result': result
    }
```

2. Click **Deploy** to save your changes.

#### Step 10.3: Test the Function

1. Click the **Test** button (near the top).
2. Create a new test event:
   - Event name: `TestAddTwoNumbers`.
   - Event JSON:
     ```json
     {
       "num1": 15,
       "num2": 27
     }
     ```
3. Click **Save**, then click **Test** again to run it.
4. You should see an **Execution result: succeeded**, with a returned value:
   ```json
   {
     "statusCode": 200,
     "result": 42
   }
   ```

#### Step 10.4: View the Logs

1. Click the **Monitor** tab, then **View CloudWatch logs**.
2. Open the most recent log stream.
3. You should see your `print()` output: `Adding 15 + 27 = 42`.

### Cleanup

1. Go to **Lambda → Functions → select `AddTwoNumbers`**.
2. Click **Actions → Delete**, confirm.
3. (Optional) The auto-created IAM role (`AddTwoNumbers-role-xxxxx`) can also be deleted from **IAM → Roles** if you don't need it.

---

## Project 11: Host a Simple Static Webpage with S3 and CloudFront

### Overview

Host a static webpage in an S3 bucket, then place a CloudFront distribution in front of it so that the content is delivered securely and efficiently, and is **only** accessible through CloudFront (not directly from S3).

### What You'll Learn

- How to configure an S3 bucket for static website hosting.
- How to create and configure a CloudFront distribution.

### Services Used

**Amazon S3**, **Amazon CloudFront**

### Step-by-Step Instructions

#### Step 11.1: Create Your Webpage Files

Create a simple `index.html` on your computer:

```html
<!DOCTYPE html>
<html>
  <head><title>My Static Website</title></head>
  <body>
    <h1>Hello! This is my static website hosted on S3 + CloudFront.</h1>
  </body>
</html>
```

Also create an `error.html` (optional but recommended):
```html
<!DOCTYPE html>
<html>
  <head><title>404 - Not Found</title></head>
  <body><h1>Oops! Page not found.</h1></body>
</html>
```

#### Step 11.2: Create the S3 Bucket

1. Search for **S3**, click **Create bucket**.
2. Bucket name: a globally unique name (e.g., `abdullah-static-site-2026`).
3. **Uncheck** "Block all public access" is **not required** here since we'll restrict access via CloudFront (Origin Access Control) instead of making the bucket public — leave **Block all public access CHECKED** (default). This keeps the bucket private and secure.
4. Click **Create bucket**.
5. Open the bucket, click **Upload**, add `index.html` and `error.html`, click **Upload**.

#### Step 11.3: Create a CloudFront Distribution

1. Search for **CloudFront** and open the service.
2. Click **Create distribution**.
3. **Origin domain**: click the field and select your S3 bucket from the dropdown (it will show as `<bucket-name>.s3.<region>.amazonaws.com`).
4. **Origin access**: select **Origin access control settings (recommended)**.
   - Click **Create control setting**, keep defaults, click **Create**.
5. **Viewer protocol policy**: **Redirect HTTP to HTTPS**.
6. Under **Default root object**, enter: `index.html`.
7. Leave other settings as default, click **Create distribution**.

#### Step 11.4: Update the S3 Bucket Policy (CloudFront Will Prompt You)

1. After creating the distribution, AWS will show a banner saying the S3 bucket policy needs to be updated to allow CloudFront access.
2. Click **Copy policy**.
3. Go to your S3 bucket → **Permissions** tab → **Bucket policy** → **Edit**.
4. Paste the copied policy, click **Save changes**.

> This policy allows **only your specific CloudFront distribution** to read objects from the bucket — direct S3 access remains blocked.

#### Step 11.5: Set the Error Page (Optional but Recommended)

1. Go back to your CloudFront distribution → **Error pages** tab.
2. Click **Create custom error response**.
3. HTTP error code: **403: Forbidden** (S3 returns this for missing objects when the bucket is private).
4. Customize error response: **Yes**.
5. Response page path: `/error.html`.
6. HTTP Response code: **404**.
7. Click **Create custom error response**.

#### Step 11.6: Access Your Website

1. Go to your CloudFront distribution's main page.
2. Copy the **Distribution domain name** (e.g., `d1234abcd.cloudfront.net`).
3. Wait 3–5 minutes for the distribution status to show **Enabled/Deployed**.
4. Open a browser and visit: `https://d1234abcd.cloudfront.net`
5. You should see your webpage.

#### Step 11.7: Confirm Direct S3 Access Is Blocked

Try visiting the direct S3 URL (e.g., `https://abdullah-static-site-2026.s3.amazonaws.com/index.html`) — you should get an **Access Denied** error, confirming the content is only accessible through CloudFront.

### Cleanup

1. Go to **CloudFront → Distributions**, select yours, click **Disable**. Wait until it's fully disabled (this can take several minutes).
2. Once disabled, select it again and click **Delete**.
3. Go to **S3**, empty the bucket (select all objects → **Delete**), then delete the bucket itself.

---

## Project 12: Create an IAM User

### Overview

Create a new IAM user with console access, add them to an "Admins" group with administrator permissions, enable MFA on the root account, and apply a strong password policy.

### What You'll Learn

- How to create IAM users, groups, and attach policies.
- The principle of least privilege and why root account protection matters.

### Service Used

**AWS IAM**

### Step-by-Step Instructions

#### Step 12.1: Create the "Admins" Group

1. Search for **IAM** and open it.
2. In the left sidebar, click **User groups → Create group**.
3. Group name: `Admins`.
4. Under **Attach permissions policies**, search for and check **AdministratorAccess**.
5. Click **Create group**.

#### Step 12.2: Create the New IAM User

1. In the left sidebar, click **Users → Create user**.
2. Username: e.g., `abdullah-admin`.
3. Check **Provide user access to the AWS Management Console**.
4. Choose **I want to create an IAM user**.
5. Set a custom password, or let AWS auto-generate one.
6. (Recommended) Check **Users must create a new password at next sign-in**.
7. Click **Next**.
8. Under **Permissions options**, choose **Add user to group**.
9. Select the **Admins** group you just created.
10. Click **Next**, review, then click **Create user**.
11. **Download the .csv** with the sign-in credentials, or copy the console sign-in URL, username, and password — you'll need these to log in.

#### Step 12.3: Enable MFA for the Root User

1. Sign out of the IAM user, sign back in as the **root user** (using your original account email).
2. Click your account name (top-right) → **Security credentials**.
3. Under **Multi-factor authentication (MFA)**, click **Assign MFA device**.
4. Give it a name (e.g., `root-mfa`).
5. Choose **Authenticator app**.
6. Scan the QR code using an authenticator app (Google Authenticator, Microsoft Authenticator, Authy, etc.).
7. Enter two consecutive MFA codes generated by the app.
8. Click **Add MFA**.

#### Step 12.4: Apply an IAM Password Policy

1. While still in the IAM console, click **Account settings** in the left sidebar (or **Password policy**).
2. Click **Edit**.
3. Configure a strong policy, for example:
   - Minimum password length: **14**
   - Require at least one uppercase letter: ✅
   - Require at least one lowercase letter: ✅
   - Require at least one number: ✅
   - Require at least one non-alphanumeric character: ✅
   - Enable password expiration: **90 days**
   - Prevent password reuse: **5 previous passwords**
4. Click **Save changes**.

#### Step 12.5: Log In as the New IAM User

1. Sign out.
2. Go to the IAM sign-in URL (format: `https://<account-id-or-alias>.signin.aws.amazon.com/console`).
3. Sign in using the username and password you created in Step 12.2.
4. If you enabled "must create a new password," you'll be prompted to set a new one.

### Cleanup

- This project sets up permanent security infrastructure — **no cleanup needed**. In fact, keep it! Using an IAM admin user instead of root for daily work is an AWS best practice.

---

## Project 13: Use a Managed Config Rule

### Overview

Enable **AWS Config** in `us-east-1`, apply a managed rule that checks whether EBS volumes are encrypted, then launch an EC2 instance with an unencrypted volume to see AWS Config flag it as non-compliant.

### What You'll Learn

- How AWS Config continuously monitors your resources for compliance.
- How it can trigger alerts when a resource violates a rule.

### Services Used

**AWS Config**, **Amazon EC2**

### Step-by-Step Instructions

#### Step 13.1: Switch to US East (N. Virginia)

1. Set your console region to **us-east-1**.

#### Step 13.2: Enable AWS Config

1. Search for **Config** and open **AWS Config**.
2. If this is your first time, click **Get started**.
3. Under **Resource types to record**, choose **Record all resources supported in this region** (simplest for learning).
4. Under **Amazon S3 bucket**, choose **Create a bucket** (AWS Config needs somewhere to store its configuration history).
5. Under **Amazon SNS topic**, you can skip this (optional) or create one to get notified of changes.
6. Click **Next**.

#### Step 13.3: Add the Managed Config Rule

1. On the **Rules** page (during setup, or later via **Rules → Add rule**), search for **encrypted-volumes**.
2. Select it.
3. Leave the default parameters (no exceptions/allowed KMS key restrictions needed for this exercise).
4. Click **Next**, review, then click **Confirm** (or **Save**).

#### Step 13.4: Launch an EC2 Instance WITHOUT an Encrypted EBS Volume

1. Go to **EC2 → Launch instance**.
2. Name: `unencrypted-volume-test`.
3. AMI: **Amazon Linux 2023**.
4. Instance type: **t2.micro**.
5. Key pair: select an existing one, or **Proceed without a key pair**.
6. Under **Configure storage**, expand the volume details:
   - Make sure **Encrypted** is set to **Not Encrypted** (this may be the default already).
7. Click **Launch instance**.

#### Step 13.5: Monitor AWS Config for the Compliance Result

1. Go back to **AWS Config → Rules**.
2. Click on the **encrypted-volumes** rule.
3. Wait a few minutes, then check the **Resources** tab (or refresh).
4. You should see your new EC2 instance's EBS volume listed with compliance status: **Noncompliant**.

> AWS Config evaluates resources periodically and on configuration changes — it may take a few minutes to a few hours to detect and display the noncompliant resource depending on the evaluation trigger type.

### Cleanup

1. Go to **EC2 → Instances**, select `unencrypted-volume-test`, click **Instance state → Terminate instance**.
2. Go to **AWS Config → Settings**, click **Turn off AWS Config** (or delete the recorder/delivery channel) to avoid ongoing recording charges.
3. Go to **AWS Config → Rules**, delete the **encrypted-volumes** rule if it wasn't removed automatically.
4. (Optional) Delete the S3 bucket AWS Config created for storing configuration history, once you're sure you don't need the records.

---

## Project 14: Deploy a CloudFormation Template from the AWS Console

### Overview

Use **AWS CloudFormation** to deploy a simple template that automatically provisions a DynamoDB table and an S3 bucket — a first taste of Infrastructure as Code (IaC).

### What You'll Learn

- How to create, update, and delete AWS resources in a controlled, repeatable way using CloudFormation stacks.

### Service Used

**AWS CloudFormation**

### Step-by-Step Instructions

#### Step 14.1: Create a Simple CloudFormation Template

Since the original article references "a simple template" without providing it directly, here is a ready-to-use one. Save this as `simple-stack.yaml` on your computer:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  A simple CloudFormation template that creates a DynamoDB table and an S3 bucket.

Resources:
  MyDynamoDBTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: CFN-Demo-Table
      AttributeDefinitions:
        - AttributeName: Id
          AttributeType: S
      KeySchema:
        - AttributeName: Id
          KeyType: HASH
      ProvisionedThroughput:
        ReadCapacityUnits: 5
        WriteCapacityUnits: 5

  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "cfn-demo-bucket-${AWS::AccountId}-${AWS::Region}"

Outputs:
  DynamoDBTableName:
    Description: Name of the created DynamoDB table
    Value: !Ref MyDynamoDBTable

  S3BucketName:
    Description: Name of the created S3 bucket
    Value: !Ref MyS3Bucket
```

> Using `!Sub "cfn-demo-bucket-${AWS::AccountId}-${AWS::Region}"` automatically makes the bucket name globally unique, so the stack won't fail due to a name collision.

#### Step 14.2: Go to the CloudFormation Console

1. Search for **CloudFormation** and open the service.

#### Step 14.3: Create the Stack

1. Click **Create stack → With new resources (standard)**.
2. Under **Prerequisite - Prepare template**, choose **Template is ready**.
3. Under **Specify template**, choose **Upload a template file**.
4. Click **Choose file**, select your `simple-stack.yaml`.
5. Click **Next**.

#### Step 14.4: Configure Stack Options

1. **Stack name**: `simple-demo-stack`.
2. Click **Next**.
3. On the **Configure stack options** page, **leave everything at the default settings**.
4. Click **Next**.

#### Step 14.5: Review and Deploy

1. Review the summary page.
2. Scroll down and click **Submit**.

#### Step 14.6: Watch the Deployment

1. You'll be redirected to the stack's detail page, with status **CREATE_IN_PROGRESS**.
2. Click the **Events** tab to watch AWS create each resource in real time.
3. Wait for the status to change to **CREATE_COMPLETE**.

#### Step 14.7: Verify the Resources Were Created

1. Click the **Resources** tab — you should see both the DynamoDB table and S3 bucket listed, each linked to its console page.
2. Click the **Outputs** tab to see the table name and bucket name.
3. (Optional) Go to **DynamoDB** and **S3** consoles directly to confirm the resources exist there too.

### Cleanup

1. Go back to **CloudFormation → Stacks**, select `simple-demo-stack`.
2. Click **Delete**.
3. Confirm the deletion.
4. Watch the **Events** tab — status will show **DELETE_IN_PROGRESS**, then the stack will disappear once fully deleted (status **DELETE_COMPLETE**).
5. Go to **DynamoDB** and **S3** consoles to confirm both resources have been deleted.

> **Note:** If the S3 bucket had any objects manually added to it, CloudFormation may fail to delete it (since CloudFormation only deletes empty buckets by default). If that happens, manually empty the bucket via the S3 console, then retry deleting the stack.

---

## Conclusion

Congratulations! 🎉 By completing all 14 projects, you've gained hands-on experience with a wide range of core AWS services:

- **AWS CloudWatch** — monitoring and billing alarms
- **AWS Budgets** — cost management
- **Amazon EC2** — virtual servers
- **Amazon ECR** — container image registry
- **Docker** — containerization
- **AWS CLI** — command-line automation
- **Amazon RDS** — managed relational databases
- **Amazon DynamoDB** — managed NoSQL databases
- **Amazon S3** — object storage
- **Amazon SNS** — notifications and messaging
- **AWS Lambda** — serverless compute
- **Amazon CloudFront** — content delivery network (CDN)
- **AWS IAM** — identity and access management
- **AWS Config** — resource compliance monitoring
- **AWS CloudFormation** — infrastructure as code

### General Best Practices to Remember Going Forward

- **Always set up billing alarms and budgets first** (Projects 1 & 2) — before doing anything else in a new AWS account.
- **Never use root account credentials for daily work** — create an IAM user with appropriate permissions instead (Project 12).
- **Enable MFA** on both your root account and IAM users.
- **Clean up resources** after every project/exercise — AWS bills for what's running, not what you're actively using.
- **Use "My IP" instead of "Anywhere (0.0.0.0/0)"** for SSH/RDP/database security group rules whenever possible.
- **Prefer Infrastructure as Code (CloudFormation, Terraform, etc.)** over manual console clicks once you're comfortable — it makes your setups repeatable, version-controlled, and easy to tear down safely.

### Suggested Next Steps

- Combine several of these services into a mini real-world project (e.g., a Lambda function triggered by an S3 upload, which writes a record into DynamoDB and sends an SNS notification).
- Explore **AWS Free Tier limits** at https://aws.amazon.com/free to understand what's free and for how long.
- Try converting one of the CloudFormation templates into **Terraform** to compare Infrastructure as Code tools.

*Happy building!* 🚀
