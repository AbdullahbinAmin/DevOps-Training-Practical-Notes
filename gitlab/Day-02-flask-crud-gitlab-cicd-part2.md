# From Code to Cloud: CI/CD for a Flask CRUD App Using GitLab and Docker — Part 2

> This builds on Part 1 (GitLab CI/CD fundamentals). Here we take a pre-built Flask CRUD app and build a complete pipeline around it — from Docker image build to automated deployment on AWS EC2 — using GitLab CI/CD, Docker/Docker Compose, Nginx, and Slack notifications.

## 1. Architectural Blueprint & Workflow

This project implements an automated Continuous Integration and Continuous Deployment (CI/CD) pipeline for a containerized Flask CRUD application using GitLab CI/CD, Docker / Docker Compose, Nginx, and an AWS EC2 instance.

### Tech Stack

- **Flask** — a lightweight Python web framework
- **Docker** — to containerize the application
- **GitLab CI/CD** — to automate build, test, and deploy
- **AWS EC2** — the deployment target

### System Workflow

```
[ Developer Push / Merge Request to main ]
                   │
                   ▼
       [ GitLab CI/CD Pipeline ]
                   │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
[ Stage 1: ]   [ Stage 2: ]   [ Stage 3: ]
[   Test   ] ─> [   Build  ] ─> [  Deploy  ]
                   │              │
                   ▼              ▼
           (Push Image to)  (SSH + SCP to AWS EC2)
          [ Docker Registry ]     │
                                  ▼
                        [ Run Container via Compose ]
                                  │
                                  ▼
                       [ Post Slack Notification ]
```

- **Trigger:** A developer pushes code or opens a Merge Request targeting the `main` branch.
- **Execution Agent:** A GitLab Runner picks up the job and executes pipeline stages using Docker-in-Docker (dind).
- **Stage 1 (Test):** Builds temporary test containers with `docker-compose.test.yml`, executes tests via `pytest`, and outputs JUnit XML artifacts.
- **Stage 2 (Build):** Compiles the production application into a Docker image, tags it with the unique git commit SHA (`$CI_COMMIT_SHA`) and `:prod`, then pushes it to Docker Hub.
- **Stage 3 (Deploy):** Connects to AWS EC2 via SSH, transfers `docker-compose.yml` and `nginx.conf`, pulls the latest Docker image, restarts services, and sends a status payload to a Slack webhook.

### What Is Docker-in-Docker (DinD)?

CI/CD jobs in GitLab run inside containers. But to build or push Docker images inside those jobs, a Docker engine must be available — which isn't present by default. DinD solves this by running the Docker daemon as a background service alongside the CI job container, so the job container can talk to it and run normal `docker build` / `docker push` commands.

## 2. Project Repository Structure

```
flask-crud-ci-cd-app/
├── app/
│   ├── models/            # SQLAlchemy ORM models
│   ├── views/              # API route handlers/blueprints
│   ├── __init__.py         # Flask application factory & logger initialization
│   └── extensions.py       # SQLAlchemy and Migrate instances
├── docker/
│   ├── nginx/
│   │   └── nginx.conf       # Nginx reverse proxy routing rules
│   ├── docker-compose.yml       # Production multi-container composition
│   ├── docker-compose.test.yml  # Isolated test execution composition
│   └── docker-compose.dev.yml   # Local development composition
├── migrations/             # Flask-Migrate / Alembic database schemas
├── scripts/
│   └── deploy.sh            # Shell deployment automation script
├── test/                   # Test suites (Pytest)
├── .dockerignore
├── .env.example
├── .env.test                # Test-environment variables (referenced by TEST_ENV_FILE)
├── .gitlab-ci.yml           # Core GitLab pipeline definition
├── config.py                # Application settings loader
├── main.py                  # WSGI entrypoint
└── requirements.txt         # Python package requirements
```

### Key Folders and Files

- **`app/`** — the application containing all Python code for the Flask app.
  - `models/` holds database model files defining how data is stored.
  - `views/` contains route handlers — the logic for each API endpoint.
  - `__init__.py` configures and sets up the Flask app, including registering blueprints and loggers.
  - `extensions.py` sets up Flask extensions like SQLAlchemy for database access and migrations for schema changes.
- **`docker/`** — manages containerization setup.
  - `nginx/` has the configuration file for the Nginx web server, which acts as a reverse proxy.
  - The different `docker-compose` files handle launching containers in production, testing, and development modes.
- **`migrations/`** — stores database migration files, tracking schema changes over time.
- **`scripts/`** — contains `deploy.sh`, triggered in the deployment stage of the pipeline to spin up or update Docker containers on the EC2 instance.
- **`test/`** — holds test cases verifying the application works as expected.
- **`.env` / `.env.test`** — store environment variables for production and testing respectively (database credentials, etc.).
- **`.gitlab-ci.yml`** — the GitLab pipeline configuration file that orchestrates the CI/CD process.
- **`config.py`** — centralizes app configuration, pulling in environment variables and setting up database connections.
- **`main.py`** — the entry point that launches the Flask app.
- **`requirements.txt`** — lists all Python dependencies required for the project.

## 3. Docker & Docker Compose Configurations

### Production Composition (`docker/docker-compose.yml`)

```yaml
version: '3.8'

services:
  backend:
    image: ${DOCKER_IMAGE_NAME}:prod
    restart: always
    env_file:
      - /home/ec2-user/app/.env
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network

  db:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - backend
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  pgdata:
```

> **Path note:** The `env_file` path (`/home/ec2-user/app/.env`) is an absolute path on the EC2 host, not a path relative to the compose file. It must exist at exactly that location before `docker-compose up` runs — this is created in Section 5 ("Preparing the Deployment Directory").

### `docker-compose.test.yml` — For Running Tests

This file is used only during testing. It spins up two services:

- **`app`** — Flask app container that:
  - Builds the image using the Dockerfile
  - Loads environment variables from `.env.test`
  - Waits for the database to be healthy
  - Runs migrations and then executes tests using `pytest`
  - Generates a test report inside `test-reports/`
- **`db`** — PostgreSQL container:
  - Uses the official Postgres image
  - Loads test environment variables
  - Runs a health check so the app waits until the DB is ready

### `docker-compose.yml` — For Running the App (Production)

- **`backend`:** uses the pre-built image pushed to Docker Hub, loads `.env`, waits for the DB to be ready, runs migrations, launches the app with `gunicorn`, and exposes a health check endpoint at `/api/health`.
- **`db`:** PostgreSQL for production, persists data via the `pgdata` volume, configured from `.env`, includes a health check.
- **`nginx`:** the official Nginx image, forwards incoming requests to the backend, uses a custom `nginx.conf` for routing.

## 4. Use Your Own Docker Image (Recommended)

The reference project uses a demo image (`akash2501/flask-crud-api` in the source blog post). This is a third party's personal repository and **may not remain available long-term** — for your own deployment, build and push your own image.

### Steps to Create and Push Your Docker Image

1. Log in to Docker Hub:

   ```bash
   docker login
   ```

2. Create the repository on Docker Hub first: **Docker Hub account → Repositories → Create Repository** (e.g., `flask-crud-api`).
3. Build the image:

   ```bash
   docker build -t your-dockerhub-username/flask-crud-api .
   ```

4. (Optional) Tag the image:

   ```bash
   docker tag your-dockerhub-username/flask-crud-api your-dockerhub-username/flask-crud-api:latest
   ```

5. Push the image to Docker Hub:

   ```bash
   docker push your-dockerhub-username/flask-crud-api
   ```

Update `DOCKER_IMAGE_NAME` in your GitLab CI/CD variables (Section 6) and in `docker-compose.yml` to point at your own image path.

> **Security note:** Prefer a Docker Hub **access token** over your actual account password for `DOCKER_PASSWORD` (used later in GitLab CI/CD variables). Tokens can be scoped and revoked independently without changing your account password.

## 5. AWS Infrastructure & Remote Host Setup

### 5.1 Instance Provisioning

1. Log in to the AWS Console and go to the **EC2 Dashboard**.
2. Click **Launch Instance**.
3. Set the instance details:
   - **Name:** something meaningful, e.g. `flask-app-server`
   - **AMI:** Amazon Linux 2023 (or Ubuntu 22.04 LTS — commands below assume Amazon Linux's `dnf`/`yum` package manager and `ec2-user`; adjust to `apt` and `ubuntu` if using Ubuntu)
   - **Instance Type:** `t3.micro` (Free Tier eligible in most regions, with better burst performance than `t2.micro` — useful headroom for running Postgres, the Flask app, and Nginx together)
   - **Key Pair:** create or select an existing `.pem` key pair — you'll need it to SSH into the machine
4. Configure **Security Group** inbound rules:
   - **SSH / Port 22** → your administrative IP address (not `0.0.0.0/0`, to limit exposure)
   - **Custom TCP / Port 80** → `0.0.0.0/0` (HTTP web traffic)
5. Leave other settings as default and click **Launch Instance**.
6. Once running, note the instance's **Public IPv4 Address** — this is used for SSH access and app access throughout the rest of this guide.

> **Elastic IP recommendation:** By default, a stopped/restarted EC2 instance can get a new public IP, which would break your `DEPLOY_HOST` variable and any hardcoded IPs. Allocate and associate an **Elastic IP** to keep the address stable across restarts.

### 5.2 Key Pair Generation & Public Key Authorization

Generate an SSH key pair on your local system to serve as the deployment key for GitLab CI/CD (separate from the `.pem` key used for your own manual SSH access):

```bash
# Generate key pair
ssh-keygen -t rsa -b 4096 -f deploy_key -N ""

# Set permissions for local private key
chmod 600 deploy_key

# Copy public key to EC2 target host
cat deploy_key.pub | ssh -i /path/to/aws-key.pem ec2-user@<EC2_PUBLIC_IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# Verify passwordless SSH connection
ssh -i deploy_key ec2-user@<EC2_PUBLIC_IP>
```

> **Keep `deploy_key` private and never commit it to the repository.** It will be pasted into a GitLab CI/CD variable (`DEPLOY_PRIVATE_KEY`) in Section 6, not stored in the codebase.

### 5.3 Target Host Package & Directory Initialization

Connect to the EC2 target host and run the setup commands:

```bash
# Update system repositories and install Docker
sudo dnf update -y
sudo dnf install docker -y

# Enable and start Docker daemon
sudo systemctl enable --now docker

# Grant execution rights without sudo to ec2-user
sudo usermod -aG docker ec2-user

# Install standalone Docker Compose V2 binary
DOCKER_COMPOSE_VERSION="v2.24.1"
sudo curl -L "https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify binaries
docker --version
docker-compose --version

# Create application deployment directory
mkdir -p ~/app/docker/nginx

# Populate production environment runtime configuration
cat <<'EOF' > ~/app/.env
POSTGRES_DB=flaskdb
POSTGRES_USER=flaskuser
POSTGRES_PASSWORD=securepassword123
POSTGRES_HOST=db
SECRET_KEY=supersecretkey
DOCKER_IMAGE_NAME=yourdockerhub/flask-crud-api
EOF
```

> **Important:** Log out and reconnect (or run `newgrp docker`) after `usermod -aG docker ec2-user` for the group change to apply to your current session.

> **Security note on `.env`:** The example above uses placeholder secrets (`securepassword123`, `supersecretkey`). Replace these with strong, unique values before any real deployment — this file holds your production database password and Flask secret key in plaintext on disk.

## 6. GitLab CI/CD Environment Variables Configuration

In GitLab, navigate to **Settings → CI/CD → Variables** and add the following. **This step is mandatory** — skipping it will cause the pipeline to fail during Docker login or deployment.

> **Important:** Do **not** check the **Protected** checkbox for these variables. Protected variables only work on protected branches, and your default branch may not be marked as protected — this can silently break your pipeline with no obvious error message.

| Variable Name | Type | Options | Description |
|---|---|---|---|
| `DOCKER_USERNAME` | Variable | Masked | Docker Hub login account username. |
| `DOCKER_PASSWORD` | Variable | Masked | Docker Hub access token with write permissions (preferred over raw password). |
| `DOCKER_IMAGE_NAME` | Variable | Expanded | Registry target path (e.g., `yourdockerhub/flask-crud-api`). |
| `DEPLOY_USER` | Variable | Expanded | SSH runtime user on the EC2 instance (`ec2-user`). |
| `DEPLOY_HOST` | Variable | Expanded | Public IP or DNS of the EC2 instance. |
| `DEPLOY_PRIVATE_KEY` | Variable | Unprotected | Raw text content of the `deploy_key` private key file. |
| `TEST_ENV_FILE` | File | Unprotected | Content of the `.env.test` file used for running tests in pipelines. |
| `SLACK_WEBHOOK_URL` | Variable | Masked | Incoming Webhook URL string from the Slack Apps dashboard. |

### Notes on Specific Variables

- **`DEPLOY_PRIVATE_KEY`:** use the **Variable** type (not File) and paste the private key content directly into the value field.
- **`TEST_ENV_FILE`:** set the **Type** to **File**. Start from `.env.example`, but update values for the test context — e.g., set `POSTGRES_HOST` to match the test DB container name (e.g., `db_test`), and use test-specific `POSTGRES_DB`, `POSTGRES_USER`, and `POSTGRES_PASSWORD`.
- **`DOCKER_USERNAME` / `DOCKER_PASSWORD`:** mask these so their values never appear in job logs, even accidentally.

### Creating a Slack Incoming Webhook

1. Create a Slack channel for CI/CD notifications, e.g. `#cicd-notify`.
2. In the Slack sidebar, click **Apps**.
3. Search for and select **Incoming WebHooks**.
4. Click **Add to Slack**.
5. Select the target channel (e.g., `#cicd-notify`) and click **Add Incoming Webhooks Integration**.
6. Copy the generated Webhook URL.
7. Add it to GitLab as the `SLACK_WEBHOOK_URL` variable.

## 7. Comprehensive `.gitlab-ci.yml` Pipeline

Place this file in the root directory of your repository:

```yaml
stages:
  - test
  - build
  - deploy

workflow:
  rules:
    - if: '$CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "main"'
      when: always
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
      when: always
    - when: never

.default-docker:
  image: docker:24.0.7
  services:
    - docker:24.0.7-dind
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
  before_script:
    - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

test:
  extends: .default-docker
  stage: test
  script:
    - cp "$TEST_ENV_FILE" .env.test
    - docker-compose -f docker/docker-compose.test.yml build
    - docker-compose -f docker/docker-compose.test.yml up --abort-on-container-exit --exit-code-from app
  after_script:
    - docker-compose -f docker/docker-compose.test.yml down -v
  artifacts:
    when: always
    reports:
      junit: test-reports/report.xml
    paths:
      - test-reports/report.xml

build:
  extends: .default-docker
  stage: build
  script:
    - docker build -f docker/Dockerfile -t $DOCKER_IMAGE_NAME:$CI_COMMIT_SHA .
    - docker push $DOCKER_IMAGE_NAME:$CI_COMMIT_SHA
    - |
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        docker tag $DOCKER_IMAGE_NAME:$CI_COMMIT_SHA $DOCKER_IMAGE_NAME:prod
        docker push $DOCKER_IMAGE_NAME:prod
      fi
  needs:
    - job: test
      artifacts: false

deploy:
  stage: deploy
  image: alpine:3.19
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  before_script:
    - apk add --no-cache openssh-client curl bash
    - mkdir -p ~/.ssh
    - echo "$DEPLOY_PRIVATE_KEY" > ~/.ssh/deploy_key
    - chmod 600 ~/.ssh/deploy_key
    - ssh-keyscan -H "$DEPLOY_HOST" >> ~/.ssh/known_hosts
  script:
    - chmod +x scripts/deploy.sh
    - ./scripts/deploy.sh "$DEPLOY_USER" "$DEPLOY_HOST"
  after_script:
    - |
      if [ "$CI_JOB_STATUS" = "success" ]; then
        STATUS_TEXT="*Deployment Status:* Succeeded"
        COLOR_CODE="#2EB886"
      else
        STATUS_TEXT="*Deployment Status:* Failed"
        COLOR_CODE="#A30200"
      fi

      PAYLOAD=$(cat <<EOF
      {
        "attachments": [
          {
            "color": "${COLOR_CODE}",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Flask CRUD App - Pipeline Update*\n${STATUS_TEXT}\n*Branch:* \`${CI_COMMIT_REF_NAME}\`\n*Commit:* \`${CI_COMMIT_SHORT_SHA}\`\n<${CI_PROJECT_URL}/-/pipelines/${CI_PIPELINE_ID}|View Pipeline Details>"
                }
              }
            ]
          }
        ]
      }
      EOF
      )

      curl -X POST "$SLACK_WEBHOOK_URL" \
           -H "Content-Type: application/json" \
           --data "$PAYLOAD"
```

### Pipeline Section Breakdown

**Stages**

```yaml
stages:
  - test
  - build
  - deploy
```

The pipeline is divided into stages that run one after another. GitLab executes all jobs in one stage before moving to the next:

- **Test:** run automated tests to verify the application works correctly.
- **Build:** once tests pass, package the application into a Docker image for consistent deployment.
- **Deploy:** deploy the built Docker image to the remote EC2 instance.

**Workflow Rules**

```yaml
workflow:
  rules:
    - if: '$CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "main"'
      when: always
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
      when: always
    - when: never
```

Without this block, GitLab would run pipelines for every push, even on branches where automation isn't needed. This restricts execution to:

- Merge requests targeting `main` (runs before the code is actually merged).
- Direct pushes to `main` (e.g., after a merge request is accepted).
- Everything else (feature branches, etc.) is skipped entirely.

**Docker Setup with Docker-in-Docker (`.default-docker`)**

```yaml
.default-docker:
  image: docker:24.0.7
  services:
    - docker:24.0.7-dind
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
  before_script:
    - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
```

This is a **hidden job template** (the leading `.` means GitLab won't run it directly — it's `extend`-ed by other jobs). It configures the DinD service and logs in to Docker Hub using the securely stored credentials before any script runs.

> **Note on `DOCKER_TLS_CERTDIR: ""`:** This disables TLS between the job container and the DinD service for simplicity. It's acceptable for most GitLab-hosted or trusted-runner setups, but if you're running your own GitLab Runner in a less trusted environment, consider enabling TLS (`DOCKER_TLS_CERTDIR: "/certs"` with the corresponding `docker:dind` TLS variant) for stronger isolation.

**Test Job**

```yaml
test:
  extends: .default-docker
  stage: test
  script:
    - cp "$TEST_ENV_FILE" .env.test
    - docker-compose -f docker/docker-compose.test.yml build
    - docker-compose -f docker/docker-compose.test.yml up --abort-on-container-exit --exit-code-from app
  after_script:
    - docker-compose -f docker/docker-compose.test.yml down -v
  artifacts:
    when: always
    reports:
      junit: test-reports/report.xml
    paths:
      - test-reports/report.xml
```

- Copies the `TEST_ENV_FILE` GitLab variable's content into `.env.test`.
- Builds and runs the test containers, waiting for the `app` container to exit and propagating its exit code (`--exit-code-from app`) so a failed test suite fails the job.
- Tears down containers and volumes afterward (`-v` also removes the test DB volume, keeping test runs isolated from each other).
- Saves the JUnit XML test report as an artifact (`when: always` means this happens even if tests fail), making results viewable directly inside GitLab's UI.

**Build Job**

```yaml
build:
  extends: .default-docker
  stage: build
  script:
    - docker build -f docker/Dockerfile -t $DOCKER_IMAGE_NAME:$CI_COMMIT_SHA .
    - docker push $DOCKER_IMAGE_NAME:$CI_COMMIT_SHA
    - |
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        docker tag $DOCKER_IMAGE_NAME:$CI_COMMIT_SHA $DOCKER_IMAGE_NAME:prod
        docker push $DOCKER_IMAGE_NAME:prod
      fi
  needs:
    - job: test
      artifacts: false
```

- Builds the image tagged with the unique commit SHA (`$CI_COMMIT_SHA`), then pushes it.
- On `main`, additionally tags and pushes as `:prod` — the tag the production `docker-compose.yml` references.
- `needs: [test]` ensures this only starts after `test` succeeds (and `artifacts: false` means it doesn't need the test job's report files, just its success status).

**Deploy Job**

```yaml
deploy:
  stage: deploy
  image: alpine:3.19
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  before_script:
    - apk add --no-cache openssh-client curl bash
    - mkdir -p ~/.ssh
    - echo "$DEPLOY_PRIVATE_KEY" > ~/.ssh/deploy_key
    - chmod 600 ~/.ssh/deploy_key
    - ssh-keyscan -H "$DEPLOY_HOST" >> ~/.ssh/known_hosts
  script:
    - chmod +x scripts/deploy.sh
    - ./scripts/deploy.sh "$DEPLOY_USER" "$DEPLOY_HOST"
  after_script:
    - |
      # Slack notification payload (see full script above)
```

- Runs only for `main` branch pipelines.
- Uses a lightweight Alpine image, installs `openssh-client`, `curl`, and `bash`.
- Writes the `DEPLOY_PRIVATE_KEY` variable to disk as an SSH key and adds the EC2 host to `known_hosts` (via `ssh-keyscan`) to avoid interactive host-key confirmation prompts, which would otherwise hang the job.
- Runs `scripts/deploy.sh`, which does the actual SSH/SCP work.
- In `after_script`, checks `$CI_JOB_STATUS` (a built-in GitLab variable — `success` or `failed`) and posts a formatted Slack message with the outcome, branch, commit, and a link back to the pipeline.

> **Why `after_script` for the Slack notification:** `after_script` runs even if the `script` section fails, which is exactly why `$CI_JOB_STATUS` can correctly report `failed` — this is what makes the Slack notification useful for catching failed deployments, not just successful ones.

### What Are Artifacts and Why Are They Important?

Artifacts are files saved from a job so they can be accessed later. In this pipeline, test results are saved as artifacts so developers can see whether tests passed or failed directly inside GitLab's UI. Without artifacts, all files produced by a job are lost once it finishes, since jobs run in temporary, disposable environments. Saving test reports as artifacts keeps the pipeline transparent and speeds up debugging.

## 8. Deployment Script (`scripts/deploy.sh`)

Save this script as `scripts/deploy.sh` and make sure it has execution permissions (`chmod +x scripts/deploy.sh`):

```bash
#!/bin/bash
set -eo pipefail

DEPLOY_USER=$1
DEPLOY_HOST=$2

DEPLOY_PATH="/home/${DEPLOY_USER}/app"
COMPOSE_FILE="${DEPLOY_PATH}/docker/docker-compose.yml"
NGINX_CONF="${DEPLOY_PATH}/docker/nginx/nginx.conf"
KEY_PATH="$HOME/.ssh/deploy_key"

echo "==> Preparing remote application directory structure..."
ssh -i "$KEY_PATH" "${DEPLOY_USER}@${DEPLOY_HOST}" "
  mkdir -p ${DEPLOY_PATH}/docker/nginx
"

echo "==> Copying deployment configuration files to remote target..."
scp -i "$KEY_PATH" docker/docker-compose.yml "${DEPLOY_USER}@${DEPLOY_HOST}:${COMPOSE_FILE}"
scp -i "$KEY_PATH" docker/nginx/nginx.conf "${DEPLOY_USER}@${DEPLOY_HOST}:${NGINX_CONF}"

echo "==> Pulling updated containers and executing service restart..."
ssh -i "$KEY_PATH" "${DEPLOY_USER}@${DEPLOY_HOST}" "
  cd ${DEPLOY_PATH}
  docker-compose -f docker/docker-compose.yml pull
  docker-compose -f docker/docker-compose.yml down --remove-orphans
  docker-compose -f docker/docker-compose.yml up -d
"

echo "==> Deployment executed successfully."
```

### Script Walkthrough

1. **Accepting Parameters** — `DEPLOY_USER` and `DEPLOY_HOST` are passed in as positional arguments from the `.gitlab-ci.yml` deploy job (`./scripts/deploy.sh "$DEPLOY_USER" "$DEPLOY_HOST"`).
2. **Define Paths** — sets where the app lives on the remote host, where the compose/nginx configs go, and where the SSH key is stored locally in the job container.
3. **Prepare Remote Directory** — SSHes in and ensures the target directory structure exists before copying files into it.
4. **Copy Configuration Files** — uses `scp` to securely transfer `docker-compose.yml` and `nginx.conf` to the remote host.
5. **Deploy the Containers** — SSHes in again to `pull` the latest image, `down` the running stack (`--remove-orphans` cleans up any leftover containers from removed services), and bring the stack back `up -d` in detached mode.

> **Permissions note:** If the target directory was ever created with `sudo` (as in some earlier manual setup), file ownership can end up mismatched, causing `scp`/`docker-compose` permission errors. If you hit this, run `sudo chown -R $DEPLOY_USER:$DEPLOY_USER ${DEPLOY_PATH}` once on the EC2 host to fix it.

## 9. Verification and API Health Checks

Once the pipeline finishes deploying, open a terminal or browser and test your application running on EC2:

### Command Line Tests

```bash
# Verify container health status on remote host via SSH
ssh -i deploy_key ec2-user@<EC2_PUBLIC_IP> "docker ps"

# Health Check Endpoint
curl -X GET http://<EC2_PUBLIC_IP>/api/health/

# Create an Item Entry
curl -X POST http://<EC2_PUBLIC_IP>/api/posts \
  -H "Content-Type: application/json" \
  -d '{"title":"Automated CI/CD Test","content":"Created via GitLab Pipeline"}'

# Fetch All Items
curl -X GET http://<EC2_PUBLIC_IP>/api/posts

# Update a Post (ID 1)
curl -X PATCH http://<EC2_PUBLIC_IP>/api/posts/1 \
  -H "Content-Type: application/json" \
  -d '{"title":"Updated Title"}'

# Delete a Post (ID 1)
curl -X DELETE http://<EC2_PUBLIC_IP>/api/posts/1
```

Also useful for confirming app containers are running properly:

```bash
docker ps       # Shows currently running containers
docker ps -a    # Lists all containers, including stopped ones
```

### Endpoints Overview

| Method | Endpoint Path | Description | Expected Status |
|---|---|---|---|
| GET | `/api/health/` | Service health status check | 200 OK |
| GET | `/api/posts` | Fetch all records | 200 OK |
| GET | `/api/posts/<id>` | Retrieve a specific record | 200 OK |
| POST | `/api/posts` | Create a new record | 201 Created |
| PATCH | `/api/posts/<id>` | Update an existing record | 200 OK |
| DELETE | `/api/posts/<id>` | Delete a record | 200 OK or 204 No Content |

## 10. Troubleshooting Tips

| Issue | Likely Cause | Fix |
|---|---|---|
| `test` job fails at `docker login` | `DOCKER_USERNAME`/`DOCKER_PASSWORD` missing, mistyped, or marked Protected on an unprotected branch | Re-check Section 6; confirm variables are unprotected and correctly named |
| `deploy` job hangs or times out at SSH step | Missing `ssh-keyscan` entry or wrong `DEPLOY_HOST` | Confirm `DEPLOY_HOST` is correct and reachable; check the EC2 security group allows port 22 from GitLab's runner IP range if using a self-hosted runner with a restricted source |
| `Permission denied (publickey)` during deploy | Public key not correctly added to EC2's `authorized_keys`, or wrong `DEPLOY_PRIVATE_KEY` value | Re-run Section 5.2's key copy step; confirm the private key pasted into GitLab matches the public key on the server exactly |
| `docker-compose: command not found` on EC2 | Compose binary not installed or not on `PATH` | Re-check Section 5.3 install steps; confirm `/usr/local/bin/docker-compose` is executable and in `PATH` |
| App unreachable at `http://<EC2_PUBLIC_IP>/` | Security group missing port 80, or `backend`/`nginx` container not healthy | Check `docker ps` on the EC2 host; confirm port 80 is open in the security group |
| Slack message never arrives | Wrong or expired `SLACK_WEBHOOK_URL`, or webhook removed from the Slack app | Regenerate the webhook in Slack and update the GitLab variable |
| Test job passes locally but fails in CI | `.env.test` values don't match container names used in `docker-compose.test.yml` (e.g., `POSTGRES_HOST`) | Ensure `TEST_ENV_FILE` content matches the service names defined in the test compose file |

## Summary

This guide walked through building a full CI/CD pipeline for a Flask CRUD application using GitLab. The Flask app was Dockerized to ensure consistent environments across local, test, and production. GitLab CI/CD stages were configured to run tests, build and push a Docker image, and automatically deploy to an AWS EC2 instance. Environment variables and SSH credentials were handled securely via GitLab CI/CD Variables rather than hardcoded in the repository, and file permissions were managed carefully to avoid deployment errors. Finally, Slack notifications were integrated to keep the team informed of build and deployment status in real time.

### Suggested Next Steps

- Add a `rollback` job or manual pipeline trigger that redeploys the previous `:prod` tag if a deployment causes issues in production.
- Add resource limits (`mem_limit`, `cpus`) to the Compose services to avoid a single container exhausting the `t3.micro`'s limited memory.
- Consider migrating from a single EC2 instance to a load-balanced setup (e.g., an Application Load Balancer with multiple instances or ECS) once traffic grows beyond what one instance can handle.
- Enable TLS on Nginx (e.g., via Let's Encrypt/Certbot) so the app isn't served over plain HTTP in production.
