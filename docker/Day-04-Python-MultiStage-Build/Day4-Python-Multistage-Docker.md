# Week 2 — Day 2: Python Application Containerization (Multi-Stage Docker Build)

In this lab, students will:

- Clone a Python web application repository from GitHub.
- Understand the concept of Multi-Stage Builds (reducing image size and keeping runtime clean).
- Write a multi-stage Dockerfile using Python base images.
- Build, run, inspect, and test the Python container.
- Push their practical work to GitHub professionally.

---

## 💡 What Is a Multi-Stage Build? (Quick Concept)

A multi-stage Dockerfile uses **two or more `FROM` stages** in one file:

- **Build stage** — includes compilers and build tools needed to install/compile dependencies. This stage is heavy.
- **Runtime stage** — a fresh, clean image that copies *only* the finished dependencies and app code from the build stage, leaving the heavy build tools behind.

**Result:** a smaller, more secure production image, because the compiler toolchain never ships to production. You'll see this size difference yourself in Step 6.

---

## ✅ Prerequisites (Check Before Starting)

Confirm the tools from earlier days are installed and working:

```bash
docker --version      # Docker installed
git --version         # Git installed
docker ps             # Docker daemon running (no permission error)
```

> If `docker ps` gives a **permission denied** error, run `sudo usermod -aG docker $USER`, then log out and back in (or run `newgrp docker`).

---

## 📌 Module 1: Workspace & Directory Setup

### Step 1: Set Up Project Directory

```bash
# 1. Navigate to your docker training directory
cd ~/devops-training-notes/docker

# 2. Create today's lab directory and move inside
mkdir Day-02-Python-App-Container
cd Day-02-Python-App-Container
```

**Why we run this:** Creates a dedicated workspace folder for the Day 2 Python containerization lab inside your central repository structure.

**Word Breakdown:**

- `cd`: Change Directory — moves to a specified folder location.
- `mkdir`: Make Directory — creates a new folder.

### Step 2: Clone the Python Project Repository

```bash
git clone https://github.com/LondheShubham153/python-multistage-docker.git .
```

**Why we run this:** Downloads the Python application files directly into your current working directory.

**Word Breakdown:**

- `git`: Invokes the Git version control CLI tool.
- `clone`: Copies a remote repository from GitHub to your local machine.
- `https://...`: Target GitHub URL containing the Python project source code.
- `.`: Tells Git to clone files directly into the current directory instead of creating a subfolder.

### Step 3: Inspect Cloned Source Files

```bash
ls -la
```

**Why we run this:** Lists all files in the directory to confirm `app.py`, `requirements.txt`, or existing configuration files are present.

**Word Breakdown:**

- `ls`: List directory contents.
- `-la`: Displays all files (including hidden ones) in detailed long format.

> **Check first:** Confirm the app's entry file is `app.py` and note the port it listens on (open `app.py` with `cat app.py`). If either differs, update the `CMD` and `EXPOSE`/`-p` values in the steps below to match.

---

## 📌 Module 2: Write the Multi-Stage Dockerfile

### Step 4: Create custom Dockerfile

```bash
nano Dockerfile
```

Paste the following multi-stage configuration:

```dockerfile
# ==========================================
# STAGE 1: Build Stage (Dependency Builder)
# ==========================================
FROM python:3.9-slim AS builder

# Set working directory inside build container
WORKDIR /app

# Install build dependencies required for compiling Python packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements file into container
COPY requirements.txt .

# Install dependencies into a temporary local directory (/install)
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ==========================================
# STAGE 2: Final Runtime Stage (Lean Image)
# ==========================================
FROM python:3.9-slim

# Set working directory inside runtime container
WORKDIR /app

# Copy installed Python packages from STAGE 1 (builder)
COPY --from=builder /install /usr/local

# Copy application code into runtime container
COPY . /app

# Expose application port (e.g., 5000)
EXPOSE 5000

# Set default startup command to run Python app
CMD ["python", "app.py"]
```

**Why we run this:** Multi-stage builds separate the build environment (which includes compiler tools and temporary dependencies) from the final production runtime environment. This produces much smaller, more secure Docker images.

**🔍 Dockerfile Instructions Breakdown:**

- `FROM python:3.9-slim AS builder`: Defines Stage 1 base image and names it `builder`.
- `WORKDIR /app`: Creates `/app` folder and sets working context inside container.
- `RUN pip install ... --prefix=/install`: Downloads and packages Python modules into isolated folder `/install`.
- `FROM python:3.9-slim`: Starts Stage 2 (fresh clean operating environment).
- `COPY --from=builder /install /usr/local`: Copies only compiled packages from Stage 1 into final image, ignoring build tools.
- `EXPOSE 5000`: Documents that application listens on port 5000 inside container.
- `CMD ["python", "app.py"]`: Runs `app.py` script when container starts.

---

## 📌 Module 3: Build & Inspect Image

### Step 5: Build Multi-Stage Docker Image

```bash
docker build -t python-multi-app:v1 .
```

**Why we run this:** Executes Dockerfile instructions across both build and runtime stages, packaging Python code and dependencies into image `python-multi-app:v1`.

**Word Breakdown:**

- `docker build`: Compiles Docker image using instructions in Dockerfile.
- `-t python-multi-app:v1`: Assigns tag name `python-multi-app` with version `v1`.
- `.`: Specifies current directory as build context.

### Step 6: Verify Local Images & Size

```bash
docker images
```

**Why we run this:** Confirms that `python-multi-app:v1` is created and displays image size details, demonstrating the benefit of small multi-stage images.

**Word Breakdown:**

- `docker images`: Displays list of all cached Docker images on local system.

> **Learning moment:** Note the size of `python-multi-app:v1`. Compare it mentally to a single-stage build that ships `build-essential` — the multi-stage image is noticeably leaner because the compiler tools stayed in Stage 1.

---

## 📌 Module 4: Run & Test Container

### Step 7: Run Python Container in Detached Mode

```bash
docker run -d -p 5000:5000 --name my-python-container python-multi-app:v1
```

**Why we run this:** Starts container in background (detached mode), forwarding host port 5000 to container port 5000.

**Word Breakdown:**

- `docker run`: Creates and executes a container instance.
- `-d`: Detached mode flag (runs silently in background).
- `-p 5000:5000`: Port forwarding (`HostPort:ContainerPort`).
- `--name my-python-container`: Assigns custom human-readable name to container instance.

### Step 8: Verify Active Container Status

```bash
docker ps
```

**Why we run this:** Verifies container state, assigned port mappings, and running uptime.

**Word Breakdown:**

- `docker ps`: Process status command listing active containers.

### Step 9: View Container Real-Time Logs

```bash
docker logs -f my-python-container
```

**Why we run this:** Streams Python application console output for testing and debugging.

**Word Breakdown:**

- `docker logs`: Displays standard output logs from container.
- `-f`: Follow flag to continuously stream log lines.

> (Press **Ctrl + C** to exit log streaming — this only stops watching the logs; the container keeps running.)

### Step 10: Test Application Endpoint

```bash
curl http://localhost:5000
```

**Why we run this:** Sends an HTTP request to local port 5000 to verify Python application responds successfully from inside container.

**Word Breakdown:**

- `curl`: Command line utility used to transfer data from or to a server URL.

> On an **AWS EC2 instance**, test with `curl http://<EC2-PUBLIC-IP>:5000` from your browser and make sure port **5000** is open in the Security Group. If `curl` isn't installed, run `sudo apt-get install curl -y`.

### Step 11: Stop & Remove Container

```bash
docker stop my-python-container
docker rm my-python-container
```

**Why we run this:** Stops running Python app gracefully and cleans up container instance from local host engine.

**Word Breakdown:**

- `stop`: Stops active container execution.
- `rm`: Deletes stopped container instance.

---

## 📌 Module 5: Push Your Lab Code to GitHub (Professionally)

Save your work to your **own** GitHub repository so it becomes part of your portfolio.

### Step 12: Point Origin to Your Own Repository

```bash
# Detach the original cloned repo's remote
git remote remove origin

# Link to YOUR new empty GitHub repo (replace the URL)
git remote add origin https://github.com/<your-username>/Week2-Day2-Python-Docker.git
```

**Why we run this:** Ensures you push to **your** repository instead of the original author's project.

### Step 13: Add a .gitignore and README (Professional Touch)

```bash
# Ignore Python cache/venv artifacts
printf "__pycache__/\n*.pyc\nvenv/\n.env\n" > .gitignore

# Create a short README describing the lab
echo "# Python App Containerization (Multi-Stage Docker Build)" > README.md
echo "Builds a lean production image by separating build and runtime stages." >> README.md
```

**Why we run this:** A `.gitignore` keeps junk (cache, virtualenvs, secrets) out of version control, and a `README.md` makes the repository look professional to recruiters and reviewers.

### Step 14: Stage, Commit, and Push

```bash
git add Dockerfile app.py requirements.txt README.md .gitignore
git commit -m "Week2 Day2: Python app with multi-stage Docker build"
git branch -M main
git push -u origin main
```

**Why we run this:** Publishes your lab work to GitHub with a clean, descriptive commit and sets `main` as the default upstream branch.

**Word Breakdown:**

- `git add`: Stages the files you want to commit.
- `git commit -m`: Records a snapshot with a descriptive message.
- `git push -u origin main`: Uploads commits and sets `main` as the tracked upstream branch.

---

## 🛠️ Troubleshooting (Common Beginner Errors)

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `curl: (56) Recv failure` / empty reply | App still starting, or listening on wrong host | Check `docker logs`; ensure the app binds to `0.0.0.0`, not `127.0.0.1`. |
| Container shows `Exited (1)` right after start | Missing dependency or code error | Read `docker logs my-python-container` for the traceback. |
| `Bind for 0.0.0.0:5000 failed: port is already allocated` | Port 5000 already in use | Use a different host port, e.g. `-p 5001:5000`. |
| `COPY failed: requirements.txt not found` | File missing or different name | Confirm with `ls -la`; adjust the `COPY` line to the real filename. |
| `curl: command not found` | curl not installed on host | `sudo apt-get install curl -y`. |
| Can't reach app on EC2 | Security Group blocks the port | Open inbound port `5000` for your IP (or `0.0.0.0/0` for testing). |
| `permission denied` on Docker socket | User not in docker group | `sudo usermod -aG docker $USER`, then re-login. |

---

## 🧹 Cleanup (After the Lab)

```bash
# Remove the container (if it still exists)
docker rm -f my-python-container

# Remove the image to free disk space
docker rmi python-multi-app:v1

# Optional: remove dangling build-cache layers from the intermediate build stage
docker image prune -f

# Verify nothing is left
docker ps -a
docker images
```

---

## 📋 Quick Command Reference

```bash
git clone <repo-url> .                                           # Clone into current folder
ls -la && cat app.py                                             # Inspect source & port
nano Dockerfile                                                  # Write multi-stage Dockerfile
docker build -t python-multi-app:v1 .                           # Build image
docker images                                                    # Check size
docker run -d -p 5000:5000 --name my-python-container python-multi-app:v1   # Run
docker ps && docker logs -f my-python-container                 # Verify
curl http://localhost:5000                                       # Test endpoint
docker stop my-python-container && docker rm my-python-container # Stop & remove
git push -u origin main                                          # Push lab to GitHub
docker rmi python-multi-app:v1                                   # Cleanup image
```
