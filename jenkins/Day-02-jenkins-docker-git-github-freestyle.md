# Hands-On Guide: Creating a Freestyle Project with Jenkins, Docker, Git & GitHub

This detailed, step-by-step walkthrough covers setting up an automated container deployment pipeline using a Jenkins Freestyle job on AWS EC2, complete with common error fixes and best practices.

## Prerequisites & Architecture Overview

- **Cloud Infrastructure:** AWS EC2 instance (`jk-server`) running Ubuntu 22.04 LTS, with Jenkins already installed and running (see the Jenkins installation guide).
- **GitHub Account:** Needed to host the sample application repository.
- **Security Group Rules:**
  - Port 22 (SSH) — Remote terminal access.
  - Port 8080 (HTTP) — Jenkins Web UI.
  - Port 3000 (HTTP) — Web Application access.

### Open Port 3000 in the Security Group

Before deploying the container, make sure inbound traffic on port 3000 is allowed:

1. Open **EC2 Console → Security Groups → Select `jk-sg`**.
2. Click **Edit inbound rules → Add rule**.
3. Configure:
   - **Type:** Custom TCP
   - **Port Range:** `3000`
   - **Source:** `0.0.0.0/0` (Anywhere IPv4) or your specific IP
4. Click **Save rules**.

## Step 1: Install Git and Docker on Jenkins Server

Connect to your EC2 instance via SSH:

```bash
ssh -i jk-ssh-key.pem ubuntu@<ec2-public-ip>
```

### 1.1 Install Git & Docker Engine

```bash
# Update repositories and install Git
sudo apt update -y
sudo apt install git -y

# Verify Git installation
git --version

# Download and execute the official Docker installation script
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Verify Docker installation
docker -v
```

### 1.2 Grant Docker Access to the ubuntu System User

```bash
# Add the current user (ubuntu) to the docker group
sudo usermod -aG docker $USER

# Restart Docker daemon
sudo systemctl restart docker
```

> **Note:** Log out and log back in (or start a new SSH session) for group membership changes to take effect on your user session. You can also run `newgrp docker` to apply it immediately in the current shell.

### 1.3 Verify Docker Is Running

```bash
sudo systemctl status docker
docker run hello-world
```

This confirms the Docker daemon is active and can pull and run images successfully before moving on to Jenkins configuration.

## Step 2: Setup GitHub Repository (jk-public-gh)

1. Log in to GitHub and create a new **Public** repository named `jk-public-gh`.
2. Upload or create the following application files in the root directory:

**Dockerfile:**

```dockerfile
FROM nginx:alpine
WORKDIR /usr/share/nginx/html
COPY . .
EXPOSE 3000
```

**index.html:**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Sample Web App</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="container">
        <h1>Jenkins & Docker Continuous Deployment</h1>
        <p>Application successfully deployed via Jenkins Freestyle Job!</p>
    </div>
</body>
</html>
```

**style.css:**

```css
body {
    font-family: Arial, sans-serif;
    background-color: #f4f6f9;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
    margin: 0;
}
.container {
    background: #ffffff;
    padding: 40px;
    border-radius: 8px;
    box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
    text-align: center;
}
h1 { color: #2c3e50; }
p { color: #7f8c8d; }
```

3. Commit all changes to the default branch (`main` or `master`).

> **Note on Nginx and port 3000:** The stock `nginx:alpine` image listens on port 80 by default. The `EXPOSE 3000` line only documents intent — it doesn't make Nginx actually listen on 3000. Since the guide maps host port 3000 to container port 3000 (`-p 3000:3000`), add a minimal Nginx config or adjust the default listen directive so the container truly serves on port 3000; otherwise the app will be unreachable at `http://<ec2-public-ip>:3000` despite the container running. A quick fix is adding a `nginx.conf` that sets `listen 3000;` and copying it into `/etc/nginx/conf.d/default.conf` in the Dockerfile.

## Step 3: Configure Jenkins Freestyle Project

1. Access Jenkins UI at `http://<ec2-public-ip>:8080`.
2. Click **New Item** on the dashboard side menu.
3. Enter Item Name: `jk-freestyle-public-gh`.
4. Select **Freestyle project** and click **OK**.

### Source Code Management (SCM)

1. Select **Git**.
2. **Repository URL:** `https://github.com/<your-username>/jk-public-gh.git`
3. **Branch Specifier:** `*/main` (or `*/master`, matching your GitHub repository's default branch).

> **Tip:** If the repository is private, add credentials under **Manage Jenkins → Credentials** and select them in the SCM section instead of relying on a public HTTPS URL.

## Step 4: Add Build Steps & Resolve Docker Permissions

### 4.1 Initial Shell Configuration

1. Under the **Build Steps** section, click **Add build step → Execute shell**.
2. Add the following commands:

```bash
# Build Docker image tagged with the Jenkins build number environment variable
docker build -t webapp:${BUILD_NUMBER} .

# Run container exposing port 3000
docker run --rm -d -p 3000:3000 --name webapp_ctr webapp:${BUILD_NUMBER}
```

3. Click **Save** and click **Build Now**.

### 4.2 Error Handling: Docker Permission Denied

**Problem**

The build fails with a red status icon. Checking Console Output displays the following error:

```
got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```

**Root Cause**

The `jenkins` service user created during installation does not belong to the `docker` system group, preventing it from accessing the Docker daemon socket `/var/run/docker.sock`.

**Solution**

Execute the following commands on your EC2 instance via SSH:

```bash
# Verify the jenkins user exists
grep jenkins /etc/passwd

# Add jenkins user to the docker group
sudo usermod -aG docker jenkins

# Confirm docker group membership
grep docker /etc/group

# Restart Jenkins service to apply new group privileges
sudo systemctl restart jenkins
```

Return to the Jenkins dashboard and click **Build Now**. The build will succeed.

## Step 5: Fix Container Name Conflicts

### 5.1 Error Handling: Container Name Conflict

**Problem**

Updating files on GitHub and clicking **Build Now** triggers build failure with the console message:

```
docker: Error response from daemon: Conflict. The container name "/webapp_ctr" is already in use by container...
```

**Root Cause**

A running container named `webapp_ctr` from the previous build is occupying port 3000 and holding the specified container name.

**Solution**

Update the build steps in Jenkins to gracefully stop any existing container before spinning up a new one.

1. Open `jk-freestyle-public-gh` project and click **Configure**.
2. Go to **Build Steps**.
3. Update the Execute shell script:

```bash
# Gracefully stop running container if it exists (prevents build failure on first run)
docker stop webapp_ctr || true

# Build new docker image tagged with current build number
docker build -t webapp:${BUILD_NUMBER} .

# Deploy new container (auto-removed on stop via --rm)
docker run --rm -d -p 3000:3000 --name webapp_ctr webapp:${BUILD_NUMBER}
```

4. Click **Save**.

> **Optional cleanup improvement:** Old images from previous build numbers accumulate on disk since each build tags a new `webapp:${BUILD_NUMBER}` image. Consider adding `docker image prune -f` after the `docker run` step, or periodically running `docker images | grep webapp` to review and remove stale tags.

## Step 6: Verify Deployment

1. Click **Build Now** on Jenkins.
2. Open **Console Output** to verify the workflow execution:
   - Source code cloning location: `/var/lib/jenkins/workspace/jk-freestyle-public-gh`
   - Image generation tag: `webapp:<BUILD_NUMBER>`
   - Container startup confirmation.
3. Verify running components via SSH terminal:

```bash
# Check created images
docker images

# Check active container status
docker ps
```

4. Access the web application in your browser:

```
http://<ec2-public-ip>:3000
```

## Step 7: Automate Builds with a GitHub Webhook (Optional)

Manually clicking **Build Now** after every commit doesn't scale. To trigger builds automatically on push:

1. In the Jenkins job, go to **Configure → Build Triggers** and check **GitHub hook trigger for GITScm polling**.
2. In your GitHub repository, go to **Settings → Webhooks → Add webhook**.
3. Set:
   - **Payload URL:** `http://<ec2-public-ip>:8080/github-webhook/`
   - **Content type:** `application/json`
   - **Events:** Just the push event
4. Save the webhook, then push a commit to confirm a build triggers automatically in Jenkins.

> **Note:** This requires port 8080 to be reachable from GitHub's servers, so the security group source for port 8080 should not be restricted to only your personal IP if you want webhooks to work.

## Troubleshooting Tips

| Issue | Likely Cause | Fix |
|---|---|---|
| `permission denied ... docker.sock` | `jenkins` user not in `docker` group | Run `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| `Conflict. The container name ... already in use` | Previous container still running | Add `docker stop webapp_ctr \|\| true` before build/run |
| App unreachable at port 3000 | Security group missing port 3000 rule, or Nginx not actually listening on 3000 | Open port 3000 in `jk-sg`; add an Nginx config setting `listen 3000;` |
| Webhook not triggering builds | Port 8080 not publicly reachable, or webhook misconfigured | Check security group source and webhook payload URL/recent deliveries in GitHub |
| Disk filling up over time | Old `webapp:<BUILD_NUMBER>` images never removed | Add `docker image prune -f` or a scheduled cleanup job |

## Summary Checklist

| Component | Path / Command |
|---|---|
| Jenkins Workspace | `/var/lib/jenkins/workspace/jk-freestyle-public-gh` |
| Docker Permission Command | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| Container Handling Script | `docker stop webapp_ctr \|\| true` |
| Application Endpoint | `http://<ec2-public-ip>:3000` |
| Security Group Ports | 22 (SSH), 8080 (Jenkins UI), 3000 (App) |

## Cleanup (Optional)

To reset the environment after testing:

```bash
# Stop and remove the running container
docker stop webapp_ctr || true

# Remove all webapp images
docker images | grep webapp | awk '{print $3}' | xargs -r docker rmi -f

# (Optional) Delete the Jenkins job from the UI: jk-freestyle-public-gh → Delete Project
```
