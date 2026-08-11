# Hands-On Guide: Creating a Jenkins Pipeline using 'Pipeline Script' Definition

This step-by-step guide walks you through building a Jenkins Pipeline using the embedded **Pipeline Script** Definition syntax. We will configure a multi-stage continuous integration and deployment pipeline (Clone, Build, Deploy) using Jenkins, Docker, Git, and GitHub.

## Prerequisites & Cleanup

- Jenkins server running on AWS EC2 (Ubuntu 22.04 LTS), with Docker and Git pre-installed.
- The Jenkins system user has execution permissions for Docker:

  ```bash
  sudo usermod -aG docker jenkins
  sudo systemctl restart jenkins
  ```

- A public GitHub repository (e.g., `jk-public-gh`) containing a `Dockerfile` and the application source files, as set up in the Freestyle project guide.
- Security group inbound rules open for ports 22 (SSH), 8080 (Jenkins UI), and 3000 (application).

### Cleanup Existing Docker Resources

Before creating the pipeline, clean up any previous containers and images from earlier runs so the pipeline starts from a known-clean state:

```bash
# Stop any running instances of the web application
docker stop webapp_ctr || true

# Forcefully remove unused containers, networks, images, and volumes
docker system prune -a -f --volumes

# Verify the environment is clean
docker ps -a
docker images
```

> **Caution:** `docker system prune -a -f --volumes` removes **all** unused images, containers, networks, and volumes on the host — not just artifacts from this project. Avoid running this on a shared or production Jenkins server without confirming nothing else depends on those resources.

## Step 1: Create a New Pipeline Item in Jenkins

1. Go to the Jenkins Dashboard and click **New Item**.
2. Enter the project name: `jk-pipeline_script-public-gh`.
3. Select **Pipeline** and click **OK**.
4. (Optional) Enter a brief description for your pipeline.
5. Scroll down to the **Pipeline** section and ensure **Definition** is set to **Pipeline script**.

## Step 2: Configure Stage 1 — Cloning Git Repository

Use Jenkins' built-in **Pipeline Syntax** generator to create the checkout script:

1. Click the **Pipeline Syntax** link located below the pipeline script text area.
2. Under **Sample Step**, select `git: Git`.
3. Fill in the parameters:
   - **Repository URL:** `https://github.com/<your-username>/jk-public-gh.git`
   - **Branch:** `main` (or `master`)
   - **Credentials:** `- none -` (for public repositories)
4. Click **Generate Pipeline Script**.
5. Copy the output snippet, which looks similar to:

   ```groovy
   git branch: 'main', url: 'https://github.com/<your-username>/jk-public-gh.git'
   ```

### Add Stage 1 to Pipeline Script

Paste the generated snippet into your Pipeline script window:

```groovy
pipeline {
    agent any

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/<your-username>/jk-public-gh.git'
            }
        }
    }
}
```

Click **Save**, then **Build Now** to verify stage execution in the **Stage View**.

## Step 3: Configure Stage 2 — Building the Docker Image

Add a second stage to compile the Docker image using the dynamic `${BUILD_NUMBER}` environment variable:

```groovy
pipeline {
    agent any

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/<your-username>/jk-public-gh.git'
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t webapp:${BUILD_NUMBER} .'
            }
        }
    }
}
```

> **Note on Syntax Validation:** Jenkins includes built-in Groovy syntax validation. If you introduce syntax errors (e.g., an extra closing brace `}`), clicking outside the script box will trigger an inline notification like `unexpected token: }`.

Click **Save**, run **Build Now**, and verify the newly generated image on your EC2 instance via SSH:

```bash
docker images
```

## Step 4: Configure Stage 3 — Deploying the Application

Add the third stage to handle container deployment while preventing port allocation and container naming conflicts.

### Handling Container Conflicts

If a container named `webapp_ctr` is already running on port 3000, running `docker run` directly produces:

```
docker: Error response from daemon: Conflict. The container name "/webapp_ctr" is already in use by container...
Bind for 0.0.0.0:3000 failed: port is already allocated.
```

To resolve this, execute `docker stop webapp_ctr` prior to initiating the run step. Multi-line shell commands in Jenkins pipelines use triple single-quotes (`'''`).

### Complete Pipeline Script

```groovy
pipeline {
    agent any

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/<your-username>/jk-public-gh.git'
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t webapp:${BUILD_NUMBER} .'
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                    docker stop webapp_ctr || true
                    docker run --rm -d -p 3000:3000 --name webapp_ctr webapp:${BUILD_NUMBER}
                '''
            }
        }
    }
}
```

## Step 5: Validate Deployment

1. Click **Save** and select **Build Now**.
2. Review the **Stage View** to verify all three stages (**Clone Repository**, **Build Image**, **Deploy Application**) display green status indicators.
3. Verify the container status in your SSH session:

   ```bash
   docker ps
   ```

4. Access the running web app in your browser:

   ```
   http://<ec2-public-ip>:3000
   ```

## Step 6: Reviewing Build Logs and the Console Output

If any stage shows a red/failed status in the Stage View, click the stage box and select **Logs**, or open **Console Output** from the build's left-hand menu, to see the exact failing command and error message. This is the fastest way to diagnose issues like missing Docker permissions, network errors during `git clone`, or a bad Dockerfile path.

## Troubleshooting Tips

| Issue | Likely Cause | Fix |
|---|---|---|
| `permission denied ... docker.sock` in Build Image stage | `jenkins` user not in `docker` group | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| `Conflict. The container name ... already in use` / `port is already allocated` | Previous `webapp_ctr` container still running | Ensure `docker stop webapp_ctr \|\| true` runs before `docker run` |
| `unexpected token: }` when editing script | Mismatched braces in Groovy pipeline syntax | Recheck stage/step brace pairs; use the Pipeline Syntax generator to avoid hand-typing errors |
| Clone Repository stage fails | Wrong branch name or repo URL, or repo is private without credentials configured | Confirm default branch name matches GitHub; add credentials if repo is private |
| App unreachable at port 3000 | Security group missing port 3000, or container not actually listening on 3000 | Open port 3000 in security group; confirm Dockerfile/Nginx config actually binds to 3000 |

## Key Takeaway & Architecture Limitation

**Architectural Risk:** In this setup, the Groovy Pipeline Script is stored directly inside the Jenkins internal configuration database. If the Jenkins EC2 instance crashes or gets terminated, the entire pipeline definition is lost.

**Best Practice Solution:** Store the pipeline definition as code inside a file named `Jenkinsfile` directly in the SCM repository (**Pipeline script from SCM**). This enforces Pipeline-as-Code principles, version control, and infrastructure disaster recovery.

### Migrating to Pipeline Script from SCM (Recommended Next Step)

1. Create a file named `Jenkinsfile` in the root of your `jk-public-gh` repository, containing the same pipeline block used above.
2. Commit and push it to GitHub.
3. In the Jenkins job, click **Configure**.
4. Under the **Pipeline** section, change **Definition** from `Pipeline script` to `Pipeline script from SCM`.
5. Set:
   - **SCM:** Git
   - **Repository URL:** `https://github.com/<your-username>/jk-public-gh.git`
   - **Branch:** `*/main`
   - **Script Path:** `Jenkinsfile` (default)
6. Click **Save**, then **Build Now**.

From this point on, any change to the pipeline logic is made by editing and committing the `Jenkinsfile` in the repository, giving you full version history and making the pipeline recoverable even if the Jenkins server itself is lost.

## Summary Checklist

| Component | Path / Command |
|---|---|
| Pipeline Job Name | `jk-pipeline_script-public-gh` |
| Pipeline Definition | Pipeline script (embedded) |
| Docker Permission Command | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| Container Handling Script | `docker stop webapp_ctr \|\| true` |
| Application Endpoint | `http://<ec2-public-ip>:3000` |
| Recommended Upgrade | Move script into `Jenkinsfile` → Pipeline script from SCM |

## Cleanup (Optional)

```bash
# Stop and remove the running container
docker stop webapp_ctr || true

# Remove all webapp images
docker images | grep webapp | awk '{print $3}' | xargs -r docker rmi -f

# (Optional) Delete the Jenkins job from the UI: jk-pipeline_script-public-gh → Delete Pipeline
```
