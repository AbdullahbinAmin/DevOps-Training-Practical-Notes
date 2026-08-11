# Hands-On Guide: Build, Push, and Deploy Automation via Jenkins, Private GitHub & Docker Hub

This guide details configuring an end-to-end continuous integration and continuous deployment (CI/CD) pipeline in Jenkins using **Pipeline script from SCM** (`Jenkinsfile`). The pipeline securely authenticates with a private GitHub repository for source code and Docker Hub for container image registry management.

## Architecture & Security Workflow

```
[ Private GitHub Repo ] ────(PAT Token: jk-gh-tk)────> [ Jenkins Pipeline ]
                                                              │
                                                        (Build Image)
                                                              │
[ Docker Hub Registry ] <──(Access Token: jk-dh-tk)──────────┼──(Push Image)
                                                              │
                                                       (Deploy Local)
                                                              │
                                                    [ Running Container ]
```

## Prerequisites

- Jenkins server running on AWS EC2 with Docker and Git already installed, and the `jenkins` system user added to the `docker` group.
- A Docker Hub account.
- A GitHub account with permission to create private repositories.
- Security group inbound rules open for ports 22 (SSH), 8080 (Jenkins UI), and 3000 (application).

## Step 1: Environment Cleanup

Run the cleanup script via SSH on your Jenkins server to purge existing containers and unused cached layers:

```bash
# Stop running application container
docker stop webapp_ctr || true

# Remove unused containers, networks, images, and volumes
docker system prune -a -f --volumes

# Verify workspace state
docker ps -a
docker images
```

> **Caution:** `docker system prune -a -f --volumes` removes all unused Docker resources on the host, not just artifacts from this project. Avoid this on a shared or production server without confirming nothing else depends on those resources.

## Step 2: Generate Access Tokens & Configure Jenkins Credentials

Secure authentication requires Personal Access Tokens (PAT) instead of raw account passwords.

### 2.1 Docker Hub Access Token (jk-dh-tk)

1. Log in to Docker Hub → Click **Profile Avatar → Account Settings → Security**.
2. Click **New Access Token**.
3. **Access Token Description:** `jk-dh-tk`
4. Set **Permissions** to **Read, Write, Delete**.
5. Click **Generate** and copy the token string (`dckr_pat_...`).

> **Important:** Docker Hub only shows the token once. Copy it immediately and store it somewhere safe (e.g., a password manager) — you cannot retrieve it again later, only revoke and regenerate.

### 2.2 GitHub Personal Access Token (jk-gh-tk)

1. Log in to GitHub → Click **Profile Avatar → Settings → Developer Settings**.
2. Select **Personal access tokens → Fine-grained tokens → Generate new token**.
3. **Token name:** `jk-gh-tk`
4. **Repository access:** Select **Only select repositories** → Choose `jk-private-gh`.
5. **Permissions:** Under **Repository permissions**, locate **Contents** and set access to **Read and write** (Metadata read-only is applied automatically).
6. Click **Generate token** and copy the key (`github_pat_...`).

> **Tip:** Set an expiration date on both tokens rather than "No expiration," and note it somewhere so you can rotate them before they lapse and silently break the pipeline.

### 2.3 Store Credentials in Jenkins Global Store

1. Open **Jenkins Dashboard → Manage Jenkins → Credentials** (under Security).
2. Click **System → Global credentials (unrestricted) → Add Credentials**.

**Add Docker Hub Credential:**
- **Kind:** Username with password
- **Username:** `<your-dockerhub-username>`
- **Password:** `<your-docker-hub-access-token>`
- **ID:** `jk-dh-tk`
- **Description:** Docker Hub Access Token

**Add GitHub Credential:**
- **Kind:** Username with password
- **Username:** `<your-github-username>`
- **Password:** `<your-github-pat-token>`
- **ID:** `jk-gh-tk`
- **Description:** GitHub Fine-Grained PAT Token

## Step 3: Setup Private GitHub Repository (jk-private-gh)

1. Create a new **Private** GitHub repository named `jk-private-gh`.
2. Add the following project files:

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
<html>
<head>
    <title>Private Pipeline Deployment</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <h1>Automated CI/CD Pipeline</h1>
    <p>Built from Private SCM & Deployed via Docker Hub</p>
</body>
</html>
```

**style.css:**

```css
body {
    font-family: Arial, sans-serif;
    background-color: #282c34;
    color: white;
    text-align: center;
    padding-top: 50px;
}
```

> **Note on port 3000:** As with the earlier public-repo guides, `nginx:alpine` listens on port 80 by default — `EXPOSE 3000` only documents intent. Add an `nginx.conf` with `listen 3000;` and copy it into `/etc/nginx/conf.d/default.conf` in the Dockerfile so the container actually serves traffic on 3000, matching the `-p 3000:3000` mapping used later in the pipeline.

3. Commit these files to the default branch of the repository.

## Step 4: Configure Jenkins Job (Pipeline script from SCM)

1. Go to **Jenkins Dashboard → New Item**.
2. Enter Name: `jk-pipeline-private-gh`.
3. Select **Pipeline** and click **OK**.
4. Scroll to the **Pipeline** section:
   - **Definition:** Select **Pipeline script from SCM**
   - **SCM:** Select **Git**
   - **Repository URL:** `https://github.com/<your-github-username>/jk-private-gh.git`
   - **Credentials:** Select `jk-gh-tk` from the dropdown list (verifies access without repository connection errors)
   - **Branch Specifier:** `*/main` (or `*/master`)
   - **Script Path:** `Jenkinsfile`
5. Click **Save**.

## Step 5: Author Complete Declarative Jenkinsfile

In your `jk-private-gh` repository root, create a file named `Jenkinsfile` containing the full execution pipeline script:

```groovy
pipeline {
    agent any

    environment {
        // Securely bind Docker Hub credentials using the Jenkins credential ID
        DOCKERHUB_CREDENTIALS = credentials('jk-dh-tk')
        DOCKER_USER = '<your-dockerhub-username>' // Replace with your actual Docker Hub username
        IMAGE_NAME = 'webapp'
    }

    stages {
        stage('SCM Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'jk-gh-tk',
                    url: "https://github.com/<your-github-username>/jk-private-gh.git"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_USER}/${IMAGE_NAME}:${BUILD_NUMBER} ."
            }
        }

        stage('Login to Docker Hub') {
            steps {
                // Pass password securely via stdin to prevent process exposure
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                sh "docker push ${DOCKER_USER}/${IMAGE_NAME}:${BUILD_NUMBER}"
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                    docker stop webapp_ctr || true
                    docker run --rm -d -p 3000:3000 --name webapp_ctr ${DOCKER_USER}/${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }
    }

    post {
        always {
            // Logout from Docker Hub to secure local credentials on agent node
            sh 'docker logout'
        }
    }
}
```

> **Important — use your own values consistently:** Replace both `<your-dockerhub-username>` in `DOCKER_USER` and `<your-github-username>` in the checkout URL with your actual account names. They must match the repository and registry you configured in Steps 2–3, or the SCM Checkout / Push stages will fail with authentication or "repository not found" errors.

Commit and push `Jenkinsfile` to your private GitHub repository.

## Step 6: Pipeline Execution & Verification

1. On the Jenkins Dashboard, open `jk-pipeline-private-gh` and click **Build Now**.
2. Open **Console Output** to verify stage steps:
   - **SCM Checkout:** Clones code using `jk-gh-tk` fine-grained authorization.
   - **Build Docker Image:** Generates tag `${DOCKER_USER}/webapp:${BUILD_NUMBER}`.
   - **Login to Docker Hub:** Authenticates via non-interactive `--password-stdin`.
   - **Push Image:** Uploads tagged layers to the Docker Hub registry.
   - **Deploy Application:** Stops active instance, cleans up port bindings, and deploys the new tag on port 3000.
3. **Verify Image Registry:** Open Docker Hub online and verify the new image tag (matching your Jenkins build number) appears in your repository.
4. **Verify Application Access:** Open your web browser and navigate to:

   ```
   http://<ec2-public-ip>:3000
   ```

## Step 7: Rotating and Revoking Tokens (Recommended)

Because both tokens grant write access to sensitive systems (your private repo and your Docker Hub registry):

1. Rotate the GitHub PAT and Docker Hub access token periodically (e.g., every 90 days), or immediately if you suspect exposure.
2. To rotate: generate a new token in GitHub/Docker Hub, then update the corresponding credential in **Manage Jenkins → Credentials** (click the credential ID → **Update**) rather than creating a new credential ID, so the `Jenkinsfile` reference stays unchanged.
3. Revoke the old token immediately after confirming the new one works.

## Troubleshooting Tips

| Issue | Likely Cause | Fix |
|---|---|---|
| SCM Checkout fails with 404 or authentication error | Wrong repo URL, wrong credential ID, or token missing repo access | Confirm the URL matches `jk-private-gh` exactly and the `jk-gh-tk` credential is selected |
| `docker login` fails in Login stage | Docker Hub token expired, revoked, or wrong permissions scope | Regenerate the token with Read/Write/Delete and update the `jk-dh-tk` credential in Jenkins |
| Push Image stage fails with "repository does not exist" or "denied" | `DOCKER_USER` doesn't match the token's account, or repo doesn't exist on Docker Hub yet | Ensure `DOCKER_USER` matches your Docker Hub username; Docker Hub auto-creates the repo on first push if the namespace matches your account |
| `Conflict. The container name ... already in use` | Previous `webapp_ctr` container still running | Confirm `docker stop webapp_ctr \|\| true` runs before `docker run` |
| App unreachable at port 3000 | Security group missing port 3000, or Nginx not listening on 3000 | Open port 3000 in the security group; confirm Nginx config binds to 3000 |
| Credentials visible in Console Output | Using `echo` or `sh` without care can leak secrets to logs | Keep using `--password-stdin` as shown; never `echo` the raw password without piping directly into a command |

## Configuration Summary

| Service | Setting | Identifier / Value |
|---|---|---|
| Jenkins SCM Credential | Kind: Username with password | ID: `jk-gh-tk` |
| Jenkins Registry Credential | Kind: Username with password | ID: `jk-dh-tk` |
| Jenkinsfile SCM Mode | Definition | Pipeline script from SCM |
| In-Line Secret Injection | Environment Block | `credentials('jk-dh-tk')` |
| Non-Interactive Login | Shell Command | `echo $PSW \| docker login -u $USR --password-stdin` |
| Application Endpoint | URL | `http://<ec2-public-ip>:3000` |

## Cleanup (Optional)

```bash
# Stop and remove the running container
docker stop webapp_ctr || true

# Remove all locally tagged webapp images
docker images | grep webapp | awk '{print $3}' | xargs -r docker rmi -f

# Log out of Docker Hub locally
docker logout

# (Optional) Delete the Jenkins job from the UI: jk-pipeline-private-gh → Delete Pipeline
# (Optional) Revoke jk-gh-tk and jk-dh-tk tokens from GitHub/Docker Hub if no longer needed
```
