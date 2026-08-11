# Install and Configure Jenkins on AWS (Ubuntu 22.04 LTS)

## Overview

This guide walks through provisioning an AWS EC2 instance and installing Jenkins on it, from launching the instance to logging into the Jenkins dashboard for the first time.

## Prerequisites

- An active AWS account with permissions to create EC2 instances, key pairs, and security groups.
- A local terminal (or AWS Cloud9) with SSH access and internet connectivity.

## Step 1: Launch an AWS EC2 Instance

1. Navigate to the **AWS Management Console** and open **EC2 Services**.
2. Click **Launch Instance** and configure the instance details:
   - **Name:** `jk-server`
   - **AMI:** Ubuntu 22.04 LTS
   - **Instance Type:** `t3.micro` (Free Tier eligible in most regions; a good baseline for Jenkins since it has better burst performance than `t2.micro`)
   - **Key Pair:** Create or select `jk-ssh-key` (`.pem` format)
   - **Network Settings:** Default VPC, auto-assign public IP enabled
   - **Storage:** At least 15–20 GB gp3 volume (Jenkins, plugins, and build artifacts can quickly exceed the default 8 GB)
   - **Security Group:** Create a security group named `jk-sg` allowing SSH (Port 22) from your IP/Anywhere
3. Click **Launch Instance**.

## Step 2: Prepare SSH Key and Connect to Instance

1. Move your downloaded key file `jk-ssh-key.pem` to your working directory (e.g., Cloud9 or local terminal).
2. Set strict read-only permissions for the key:

   ```bash
   chmod 400 jk-ssh-key.pem
   ```

3. Connect to your EC2 instance via SSH:

   ```bash
   ssh -i jk-ssh-key.pem ubuntu@<ec2-public-ip>
   ```

## Step 3: Configure AWS Security Group (Port 8080)

Jenkins runs on default HTTP port `8080`. Before accessing the web dashboard:

1. Open **EC2 Console → Security Groups → Select `jk-sg`**.
2. Click **Edit inbound rules**.
3. Add Rule:
   - **Type:** Custom TCP
   - **Port Range:** `8080`
   - **Source:** `0.0.0.0/0` (Anywhere IPv4) or your specific IP address
4. Click **Save rules**.

> **Security tip:** Restricting the source to your own IP (`My IP`) instead of `0.0.0.0/0` is safer for both the SSH and Jenkins ports, especially once Jenkins is holding real credentials and pipeline secrets.

## Step 4: Install Java OpenJDK 21 and Jenkins

Modern Jenkins releases require Java 17 or Java 21. Execute the following commands to update packages, install Java, import the official signed GPG key, and install Jenkins:

```bash
# Update local package repository index and install Java dependencies
sudo apt update -y
sudo apt install fontconfig openjdk-21-jre -y

# Verify Java installation
java -version

# Add the official Jenkins GPG keyring directory and key
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

# Add the Jenkins apt repository entry
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Refresh package index and install Jenkins
sudo apt update -y
sudo apt install jenkins -y
```

## Step 5: Start and Enable Jenkins Service

1. Verify the Jenkins package version:

   ```bash
   jenkins --version
   ```

2. Enable and start the system service (so Jenkins survives reboots and starts automatically):

   ```bash
   sudo systemctl enable jenkins
   sudo systemctl start jenkins
   ```

3. Verify service status:

   ```bash
   systemctl status jenkins
   ```

   (Press `q` to exit the status view.)

4. Verify that Jenkins is listening on port `8080`:

   ```bash
   ps -ef | grep jenkins
   ```

   Alternatively, confirm the listening port directly:

   ```bash
   sudo ss -tulpn | grep 8080
   ```

5. If the service fails to start, check the logs for troubleshooting:

   ```bash
   sudo journalctl -u jenkins -n 100 --no-pager
   ```

## Step 6: Unlock Jenkins Web Interface

1. Open your web browser and navigate to:

   ```
   http://<ec2-public-ip>:8080
   ```

2. Retrieve the initial administrator password from the server terminal:

   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```

3. Copy the string output and paste it into the **Administrator password** field on the web page.
4. Click **Continue**.

## Step 7: Complete Initial Setup Wizard

1. **Customize Jenkins:** Click **Install suggested plugins** (this handles baseline tools like Git, Pipeline, SSH agents, and Gradle).
2. **Create First Admin User:** Fill in your credentials:
   - **Username:** Set your username
   - **Password:** Set your secure password
   - **Full Name:** Your full name
   - **E-mail Address:** Your email address
3. **Instance Configuration:** Confirm the URL (`http://<ec2-public-ip>:8080/`).
4. Click **Save and Continue → Save and Finish**.
5. Click **Start using Jenkins** to enter the main administration dashboard.

## Step 8: Post-Installation Verification

1. Confirm you can see the Jenkins dashboard ("Welcome to Jenkins!") at `http://<ec2-public-ip>:8080`.
2. Create a simple **Freestyle project** or **Pipeline** job and run a basic build (e.g., an `echo "Hello from Jenkins"` shell step) to confirm the executor is working end to end.
3. Under **Manage Jenkins → Plugins**, confirm the suggested plugins installed without errors.

## Troubleshooting Tips

| Issue | Likely Cause | Fix |
|---|---|---|
| Can't reach `http://<ec2-public-ip>:8080` | Security group missing port 8080 rule | Re-check Step 3 inbound rules |
| SSH connection refused | Wrong key permissions or security group missing port 22 | Run `chmod 400 jk-ssh-key.pem`; verify port 22 rule |
| Jenkins service won't start | Java not installed or wrong version | Confirm `java -version` shows 17 or 21 |
| Out of disk space during builds | Default EBS volume too small | Resize the EBS volume or attach additional storage |

## Cleanup (Optional)

If this was a lab/test environment, terminate the instance to avoid ongoing charges:

1. Go to **EC2 Console → Instances**.
2. Select `jk-server`.
3. Click **Instance State → Terminate Instance**.
4. Optionally delete the `jk-sg` security group and `jk-ssh-key` key pair if no longer needed.
