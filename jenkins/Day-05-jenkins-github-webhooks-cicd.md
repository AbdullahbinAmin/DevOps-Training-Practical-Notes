# Hands-On Guide: Automated CI/CD Pipeline Triggers using GitHub Webhooks & Jenkins

This guide demonstrates how to turn a manual Jenkins pipeline into a fully automated Continuous Deployment (CD) pipeline. By setting up a GitHub Webhook, every code push or modification committed to the repository will automatically trigger the Jenkins build, push the new image to Docker Hub, and deploy the updated application to production without manual intervention.

## Architectural Overview: Continuous Delivery vs. Continuous Deployment

- **Continuous Delivery (CD):** Code changes are automatically built, tested, and staged for release (e.g., pushed to Docker Hub), but production deployment requires manual approval.
- **Continuous Deployment (CD):** Code changes flow through the pipeline and are automatically deployed directly to production upon passing all pipeline checks.

```
[ Git Push to GitHub ] ──(Webhook Trigger)──> [ Jenkins Pipeline ]
                                                     │
                                           (Build & Test Container)
                                                     │
                                           (Push to Docker Hub)
                                                     │
                                           (Auto-Deploy to Server)
                                                     │
                                           [ Application Updated ]
```

## Prerequisites

- Active Jenkins server configured with Docker and Git permissions.
- An existing Jenkins Pipeline project configured using **Pipeline script from SCM** (e.g., `jk-pipeline-private-gh`).
- AWS EC2 Security Group configured to allow inbound traffic on:
  - Port 8080 (Jenkins Web UI / Webhook receiver endpoint).
  - Port 3000 (Application traffic).
- The GitHub Plugin installed in Jenkins (bundled with the "suggested plugins" set during initial setup — confirm under **Manage Jenkins → Plugins → Installed plugins**).

> **Note on IP stability:** Webhooks call a fixed URL. If your EC2 instance doesn't have an Elastic IP attached, the public IP can change on stop/start, silently breaking the webhook. Consider allocating and associating an Elastic IP before setting this up so the Payload URL stays valid.

## Step 1: Enable GitHub Webhook Trigger in Jenkins

1. Log in to your Jenkins Dashboard.
2. Open your project (`jk-pipeline-private-gh`) and click **Configure** in the side menu.
3. Scroll down to the **Build Triggers** section.
4. Check the box for **GitHub hook trigger for GITScm polling**.
5. Click **Save**.

## Step 2: Configure Webhook in GitHub Repository

1. Open your GitHub repository (`jk-private-gh`).
2. Navigate to **Settings → Webhooks** (under the Options menu).
3. Click **Add webhook**.
4. Configure the webhook parameters:
   - **Payload URL:** `http://<YOUR-JENKINS-EC2-PUBLIC-IP>:8080/github-webhook/`
     > **Critical Note:** The trailing slash `/` at the end of `/github-webhook/` is mandatory.
   - **Content type:** `application/json`
   - **Secret:** (Leave blank unless configured in Jenkins Global Security — see the security note below)
   - **Which events would you like to trigger this webhook?** Select **Just the push event**.
   - **Active:** Ensure the checkbox is checked.
5. Click **Add webhook**.
6. Refresh the Webhooks page and check for a **Green Checkmark** (✓) next to the Payload URL, indicating successful HTTP ping delivery (200 OK).

> **Security note — webhook secret:** Since this Payload URL is reachable from the public internet on an unauthenticated path, anyone who discovers it could send fake push events to trigger builds. For anything beyond a lab exercise, set a **Secret** in the GitHub webhook form and configure a matching secret token in Jenkins (**Manage Jenkins → System → GitHub → Advanced → Shared secrets**, or via a credential bound to the webhook). This lets Jenkins verify the payload's signature and reject unauthenticated requests.

## Step 3: Test Automated Continuous Deployment

To verify the end-to-end automation, make a change in your application repository:

### 1. Update the Dockerfile

In your `jk-private-gh` repository, update your Dockerfile to copy all static assets:

```dockerfile
FROM nginx:alpine
WORKDIR /usr/share/nginx/html
# Update file copy instruction to capture style.css and index.html
COPY . .
EXPOSE 3000
```

### 2. Commit and Push Changes

Commit the changes directly to the main branch:

```bash
git add Dockerfile
git commit -m "Fix asset copy path in Dockerfile"
git push origin main
```

## Step 4: Verify Pipeline Execution and Deployment

1. **Check Jenkins Dashboard:** Return to Jenkins. Within seconds of pushing to GitHub, a new build (e.g., #2 or #3) will automatically appear in **Build History**, triggered by "Started by GitHub push by `<username>`".
2. **Review Console Output:** Open the running build's Console Output. Verify SCM checkout, image tagging (`${DOCKER_USER}/webapp:${BUILD_NUMBER}`), Docker Hub upload, and local container execution.
3. **Verify Web Application:** Open your web browser and reload:

   ```
   http://<ec2-public-ip>:3000
   ```

   Confirm that the updated UI and CSS styles render correctly without requiring manual intervention.

## Step 5: Inspecting Webhook Deliveries in GitHub

If a push doesn't trigger a build, GitHub keeps a log of every delivery attempt, which is often faster to check than Jenkins logs:

1. Go to **Settings → Webhooks** in your repository and click on the configured webhook.
2. Open the **Recent Deliveries** tab.
3. Click any delivery to see the **Request** (payload sent) and **Response** (what Jenkins returned).
4. A response other than `200` indicates the failure point — e.g., a `404` usually means the URL path is wrong, and a connection timeout usually means the security group is blocking the request.
5. Use the **Redeliver** button to resend a past payload without needing a new commit, useful for iterating on a fix.

## Step 6: Fallback — SCM Polling (When Webhooks Aren't an Option)

If your Jenkins instance isn't publicly reachable (e.g., behind a corporate firewall or VPN with no inbound access), webhooks won't work. As a fallback:

1. In the job's **Configure → Build Triggers**, check **Poll SCM** instead of (or in addition to) the GitHub hook trigger.
2. Set a schedule using cron syntax, e.g. `H/5 * * * *` to check for changes every 5 minutes.
3. This is less immediate than a webhook (there's polling latency) but doesn't require any inbound network access to Jenkins.

## Summary Troubleshooting Checklist

| Symptom | Cause | Solution |
|---|---|---|
| GitHub Webhook Red Exclamation Point | Security Group blocking port 8080 or incorrect URL syntax | Verify EC2 Security Group allows HTTP on port 8080 and ensure the URL ends with `/github-webhook/` |
| Webhook HTTP 500 Error | Missing plugin or SCM mismatch | Ensure the GitHub Plugin is installed in Jenkins and the repository URL in the job matches the GitHub repository exactly |
| Build Not Triggered on Push | "GitHub hook trigger for GITScm polling" not enabled | Open job configuration and ensure the trigger checkbox under Build Triggers is selected |
| Webhook shows `200` but no build runs | Branch pushed doesn't match the job's configured Branch Specifier | Confirm the job's Branch Specifier (`*/main`) matches the branch you pushed to |
| Webhook Payload URL unreachable after instance restart | EC2 public IP changed (no Elastic IP) | Allocate and associate an Elastic IP, then update the webhook Payload URL |
| Anyone can trigger a build by hitting the URL | No webhook secret configured | Add a Secret in GitHub and configure a matching one in Jenkins |

## Cleanup (Optional)

```bash
# Stop and remove the running container
docker stop webapp_ctr || true

# Remove all locally tagged webapp images
docker images | grep webapp | awk '{print $3}' | xargs -r docker rmi -f
```

To remove the automation itself:

1. In GitHub, go to **Settings → Webhooks**, select the webhook, and click **Delete**.
2. In Jenkins, open the job's **Configure** page, uncheck **GitHub hook trigger for GITScm polling**, and click **Save**.
