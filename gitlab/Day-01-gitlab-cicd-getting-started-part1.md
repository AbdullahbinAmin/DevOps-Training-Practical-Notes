# Getting Started with GitLab CI/CD: A Beginner-Friendly Guide to Pipelines and Automation — Part 1

## 1. Introduction & Core Concepts

In modern DevOps workflows, getting code from a developer's local workstation to a production server requires speed, repeatability, and reliability. Manual deployments — such as manually SSHing into servers, pulling git changes, and restarting services — introduce human error, configuration drift, and release bottlenecks.

### What is CI/CD?

- **Continuous Integration (CI):** The practice of automatically building and testing code every time a developer commits changes to a shared repository.
- **Continuous Delivery (CD):** Ensures that tested code can be deployed to staging or production automatically with a single manual approval.
- **Continuous Deployment (CD):** Automates the entire release cycle. Passing code automatically flows directly to production without manual intervention.

## 2. GitLab CI/CD Architecture: Server vs. Runner

GitLab CI/CD relies on two decoupled components:

```
[ Developer Push ] ──> [ GitLab Server (Manager) ]
                               │
                      (Assigns Job & Logs)
                               │
                               ▼
                   [ GitLab Runner (Worker) ]
                               │
                   (Pulls Docker Container)
                               │
                   (Executes Shell Scripts)
```

- **GitLab Server (Manager):** Manages repository state, processes `.gitlab-ci.yml`, coordinates pipeline execution, and stores build logs/artifacts.
- **GitLab Runner (Worker):** An isolated service (running on Linux, macOS, Windows, or Kubernetes) that polls the GitLab Server, picks up pending jobs, and executes them inside an isolated environment (such as a Docker container).

### Where Does the Runner Come From?

The steps below assume a Runner is already available to execute jobs. In practice, you have two options, and it's worth knowing which one you're using before your first pipeline runs:

1. **GitLab.com Shared Runners:** If you're using GitLab.com (the hosted SaaS version), shared runners are typically available to your project automatically, and you can skip runner setup entirely for this walkthrough. Free tiers include a limited number of CI/CD minutes per month.
2. **Self-Hosted Runner:** If you're using a self-managed GitLab instance, or want dedicated compute for your own project, you need to install and register a runner yourself (covered in Step 3.0 below).

Check whether a runner is available for your project under **Settings → CI/CD → Runners** before proceeding. If no runners are shown as available, complete Step 3.0 first.

## 3. Step-by-Step Hands-On Practical Setup

### Step 3.0: Install and Register a GitLab Runner (If Not Using Shared Runners)

If your project has no available runner, install one on a Linux server (this can be the same EC2-style VM used elsewhere in your labs, or any Linux host with internet access):

```bash
# Download and install the official GitLab Runner package
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install gitlab-runner -y

# Verify installation
gitlab-runner --version
```

Register the runner against your project or group:

1. In GitLab, go to your project → **Settings → CI/CD → Runners**.
2. Click **New project runner** (or **New group runner** for a group-wide runner).
3. Select the operating system (Linux) and optionally add tags to control which jobs this runner picks up.
4. Click **Create runner**, then copy the generated registration command and authentication token shown.
5. On your runner host, run the registration command provided (it will look similar to):

   ```bash
   sudo gitlab-runner register \
     --url https://gitlab.com/ \
     --token <your-runner-authentication-token>
   ```

6. When prompted for an executor, choose `docker` (matches the `image:` keyword used in the pipeline below) and specify a default image such as `ubuntu:22.04` if asked.
7. Confirm the runner shows as **Online** (green dot) under **Settings → CI/CD → Runners**.

> **Note:** If using the `docker` executor, Docker must also be installed on the runner host itself (`curl -fsSL https://get.docker.com | sudo sh`), since the runner uses the local Docker daemon to pull and run job containers.

### Step 3.1: Create a GitLab Repository

1. Log in to your GitLab account.
2. Click **New project → Create blank project**.
3. **Project name:** `ci-cd-demo`.
4. Set **Visibility Level** to Private or Public.
5. Uncheck **Initialize repository with a README**.
6. Click **Create project**.

### Step 3.2: Create the .gitlab-ci.yml Pipeline Configuration

1. In your project page, press the **.** (dot) key on your keyboard to launch the browser-based GitLab Web IDE.
2. Create a new file in the root directory named `.gitlab-ci.yml` (must start with a dot).
3. Paste the following configuration:

```yaml
# Define global default Docker image for jobs if unassigned
image: ubuntu:22.04

# Define sequential pipeline stages
stages:
  - build
  - test
  - deploy

# Stage 1: Build Job
build-job:
  stage: build
  script:
    - echo "Building the application..."
    - echo "Compilation step completed successfully."

# Stage 2: Test Job
test-job:
  stage: test
  script:
    - echo "Running unit and integration tests..."
    - echo "All tests passed."

# Stage 3: Deploy Job
deploy-job:
  stage: deploy
  script:
    - echo "Deploying application to environment..."
    - echo "Deployment verified."
```

4. Commit the changes to the default branch (`main`).

> **YAML indentation matters:** GitLab CI/CD configuration is standard YAML, which is whitespace-sensitive. Use consistent 2-space indentation (as shown above) — mismatched indentation is one of the most common causes of a pipeline failing to even start, with GitLab reporting a "yaml invalid" error before any job runs.

## 4. Pipeline Execution & Anatomy of Job Logs

Once committed, GitLab automatically triggers a pipeline run.

### Navigating Pipelines in GitLab UI

1. Go to **Build → Pipelines** in the left sidebar.
2. Click on the status badge (e.g., **running** or **passed**).
3. Click on individual job names (`build-job`, `test-job`, `deploy-job`) to inspect the real-time execution console.

### Breakdown of Job Execution Logs

When you open a job log, the Runner performs the following sequence under the hood:

**Runner Allocation & Container Provisioning:**

```
Running with gitlab-runner 16.5.0 (j2nyww-s)
Using Docker executor with image ubuntu:22.04 ...
Pulling docker image ubuntu:22.04 ...
Using docker image sha256:... for ubuntu:22.04 ...
```

**Repository Fetch & Checkout:**

```
Fetching changes with git depth set to 20...
Reinitialized existing Git repository in /builds/user/ci-cd-demo/.git/
Checking out 8824638d as main...
```

**Script Execution:**

```
$ echo "Building the application..."
Building the application...
$ echo "Compilation step completed successfully."
Compilation step completed successfully.
Job succeeded
```

### Validating the Full Pipeline Ran in Order

Beyond checking each job individually, open the **Pipeline** overview graph (**Build → Pipelines → click the pipeline**) to see all three stages laid out left to right with connecting lines. Confirm `build-job` → `test-job` → `deploy-job` each show a green checkmark and ran in that sequence — this visual is the quickest way to confirm the `stages` ordering is being respected before moving on to more complex pipelines.

## 5. Common First-Run Issues

| Issue | Likely Cause | Fix |
|---|---|---|
| Pipeline shows "yaml invalid" and never starts | Incorrect indentation or tabs used instead of spaces in `.gitlab-ci.yml` | Re-check the file with consistent 2-space indentation; GitLab's **CI Lint** tool (**Build → Pipelines → CI Lint**) validates syntax before committing |
| Jobs stuck in "pending" indefinitely | No runner available/online for this project | Check **Settings → CI/CD → Runners**; complete Step 3.0 if none are active, or verify shared runners are enabled for your namespace |
| Job fails with "docker: not found" or executor errors | Runner registered with the wrong executor type | Re-register the runner selecting the `docker` executor, or install Docker on the runner host |
| Pipeline doesn't trigger on push | CI/CD disabled for the project, or `.gitlab-ci.yml` not in the repo root | Confirm **Settings → CI/CD** has pipelines enabled and the file is named exactly `.gitlab-ci.yml` at the project root |

## 6. Summary & Key Takeaways

| Syntax Keyword | Purpose |
|---|---|
| `stages` | Declares the global execution order for job groups (e.g., build → test → deploy). |
| `stage` | Assigns a specific job to one of the predefined stages. |
| `script` | Specifies the exact shell commands executed sequentially by the GitLab Runner inside the job container. |
| `image` | Specifies the Docker container environment where the job commands execute. |

### What's Next

This pipeline only echoes placeholder text — it doesn't build a real application, run real tests, or deploy anywhere yet. Natural next steps for a Part 2 guide would include:

- Replacing the placeholder `script` steps with real build commands (e.g., `docker build`, a test runner like `pytest` or `npm test`).
- Using `artifacts` to pass files (like a compiled binary or test report) between stages.
- Using CI/CD **Variables** (**Settings → CI/CD → Variables**) to store secrets such as registry credentials, instead of hardcoding them in the YAML file.
- Adding `rules` or `only`/`except` to control which branches or events trigger which jobs.
- Configuring a manual approval gate (`when: manual`) on the deploy stage to move from Continuous Delivery to a controlled, approval-based release.
