# Hands-On Guide: Creating a Jenkins Pipeline using 'Pipeline Script from SCM'

This step-by-step tutorial covers configuring a Jenkins Pipeline using the **Pipeline Script from SCM** approach. Instead of embedding script logic directly inside the Jenkins server configuration, we store our workflow in a `Jenkinsfile` directly inside the GitHub repository. This adopts the Pipeline-as-Code pattern for version control and disaster recovery.

## Prerequisites & Cleanup

Ensure your Jenkins server on AWS EC2 is active with Docker and Git configured, and that the `jenkins` system user has been added to the `docker` group (`sudo usermod -aG docker jenkins && sudo systemctl restart jenkins`).

### Clean Up Local Environment

Before executing the pipeline, purge pre-existing containers, images, and cached layers to ensure a clean slate:

```bash
# Stop active application container
docker stop webapp_ctr || true

# Force removal of unused containers, networks, images, and volumes
docker system prune -a -f --volumes

# Verify workspace clean state
docker ps -a
docker images
```

> **Caution:** `docker system prune -a -f --volumes` removes all unused Docker resources on the host, not just artifacts from this project. Avoid this on a shared or production server without confirming nothing else depends on those resources.

## Step 1: Create a Jenkinsfile in GitHub

1. Open your public repository on GitHub (`jk-public-gh`).
2. Click **Add file → Create new file**.
3. Name the file `Jenkinsfile` (case-sensitive).
4. Paste the following declarative pipeline script:

```groovy
pipeline {
    agent any
    stages {
        stage('Clone Repository') {
            steps {
                // SCM checkout occurs automatically when using 'Pipeline script from SCM'
                echo 'Checking out source code from SCM...'
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
                    # docker stop webapp_ctr || true
                    docker run --rm -d -p 3000:3000 --name webapp_ctr webapp:${BUILD_NUMBER}
                '''
            }
        }
    }
}
```

> **Fix — comment syntax inside `sh` blocks:** The content inside a Groovy `sh '''...'''` block is executed as a **shell (bash) script**, not Groovy — so shell comments must use `#`, not Groovy's `//`. Writing `// docker stop webapp_ctr || true` inside the `sh` block would make bash try to run `//` as a command and fail with a `command not found` error. The script above has been corrected to use `#`. Groovy's `//` and `/* ... */` comment syntax only applies to lines written directly in the pipeline's Groovy structure (like the `echo` step above), not to text inside `sh` string blocks.

> **Note on the commented-out `docker stop`:** This line is commented out here only because no `webapp_ctr` container exists yet on the very first run. From the **second build onward**, leave it commented out and Jenkins will fail with a "container name already in use" / "port already allocated" error, since the previous container is still running. Uncomment that line before your second build (or just leave it uncommented from the start — `docker stop webapp_ctr || true` is always safe to run, even when no container exists, because of the `|| true` fallback).

5. Click **Commit changes...** to save the Jenkinsfile to the default branch (`main`).

## Step 2: Configure Jenkins Pipeline Job

1. Go to your Jenkins Dashboard and click **New Item**.
2. Enter Item Name: `jk-pipeline_script_from_SCM-public-gh`.
3. Select **Pipeline** and click **OK**.
4. (Optional) Add a brief description for your project.
5. Scroll down to the **Pipeline** section and configure:
   - **Definition:** Select **Pipeline script from SCM**
   - **SCM:** Select **Git**
   - **Repository URL:** `https://github.com/<your-username>/jk-public-gh.git`
   - **Credentials:** `- none -` (for public repositories)
   - **Branch Specifier:** `*/main` (or `*/master`)
   - **Script Path:** `Jenkinsfile`
6. Click **Save**.

## Step 3: Execute and Validate Pipeline

1. Click **Build Now** on the project page.
2. Observe the **Stage View** as Jenkins performs the following workflow:
   - Automatically checks out the repository containing the Jenkinsfile.
   - Parses the script instructions directly from SCM.
   - Executes the **Build Image** and **Deploy Application** stages sequentially.
3. Open **Console Output** to verify step execution.
4. Verify the container status in your server SSH session:

   ```bash
   docker ps
   ```

5. Open your web browser and navigate to:

   ```
   http://<ec2-public-ip>:3000
   ```

## Step 4: Run a Second Build (Confirming Idempotency)

To fully validate the pipeline behaves correctly on repeated runs — not just the first one — trigger **Build Now** a second time:

1. If the `docker stop webapp_ctr || true` line is still commented out from Step 1, this build will fail with a container name/port conflict.
2. Edit the `Jenkinsfile` in GitHub, uncomment that line, commit the change, and click **Build Now** again.
3. Confirm the build now succeeds cleanly regardless of whether a previous container was running — this is what makes the pipeline safely repeatable.

## Step 5: Automate with a Webhook (Optional Next Step)

Once the SCM-based pipeline works reliably, configure a GitHub webhook so builds trigger automatically on every push instead of requiring manual **Build Now** clicks — see the companion guide on GitHub Webhook automation for the full setup (Payload URL, `github-webhook/` trigger, and troubleshooting).

## Troubleshooting Tips

| Issue | Likely Cause | Fix |
|---|---|---|
| `command not found: //docker` or similar in Console Output | Used Groovy `//` comment syntax inside an `sh '''...'''` block | Use `#` for comments inside shell blocks, not `//` |
| `Conflict. The container name ... already in use` on 2nd+ build | `docker stop webapp_ctr` line still commented out | Uncomment it — `|| true` makes it safe even on the first run |
| `permission denied ... docker.sock` | `jenkins` user not in `docker` group | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| Jenkins reports "Jenkinsfile not found" | Wrong Script Path, branch mismatch, or file not committed to the configured branch | Confirm Script Path is `Jenkinsfile` and Branch Specifier matches the branch the file was committed to |
| App unreachable at port 3000 | Security group missing port 3000, or Nginx not listening on 3000 | Open port 3000 in the security group; confirm Nginx config binds to 3000 |

## Key Benefits of Pipeline from SCM

| Feature | Embedded Pipeline Script | Pipeline Script from SCM |
|---|---|---|
| Storage Location | Jenkins internal XML configuration | SCM repository (`Jenkinsfile`) |
| Version Control | Limited build history logs | Full Git commit history & auditing |
| Disaster Recovery | Script lost if Jenkins server crashes | Preserved safely inside GitHub repository |
| Collaboration | Restricted to Jenkins UI users | Managed via Pull Requests & code reviews |

## Cleanup (Optional)

```bash
# Stop and remove the running container
docker stop webapp_ctr || true

# Remove all locally tagged webapp images
docker images | grep webapp | awk '{print $3}' | xargs -r docker rmi -f

# (Optional) Delete the Jenkins job from the UI: jk-pipeline_script_from_SCM-public-gh → Delete Pipeline
```
