# Chapter 15 — Docker Scout and Docker Init (Bonus Tools)

## Objective
Learn two additional Docker Desktop bonus tools: **Docker Scout**, which scans your images for security vulnerabilities, and **Docker Init**, which auto-generates a Dockerfile, docker-compose.yml, .dockerignore, and README for you based on a few simple questions.

## Prerequisites
* OS: Any machine with Docker Desktop installed (both tools ship as Docker Desktop add-ons/CLI plugins)
* Required software: Docker Desktop (or Docker Engine with the Scout/Init CLI plugins installed separately)
* Required account: Docker Hub account (for Docker Scout to fully analyze pushed images)

## Pre-Check
```bash
docker --version
docker scout version
docker init --version
```
**What this does:** Confirms Docker, and the Scout/Init plugins, are available. If `docker scout`/`docker init` are not recognized, they may need to be installed/enabled separately depending on your Docker Desktop version — check Docker's official documentation for your platform.

---

## Part A — Docker Scout (Image Vulnerability Scanning)

### Concept — Why Scan Images?

**The problem:**
Every Docker image you build is based on a **base image** (from a `FROM` instruction). This base image might contain known security vulnerabilities or outdated packages. If you don't check for this, you could be shipping a vulnerable application without realizing it.

**The solution — Docker Scout:**
Docker Scout is Docker's own official tool (similar in purpose to a well-known third-party tool called **Trivy**, which also scans images and file systems for vulnerabilities) that analyzes an image and reports:
* **Critical** severity issues
* **High** severity issues
* **Medium** severity issues
* **Low** severity issues

### Step 1 — Access Docker Scout via Docker Desktop GUI
1. Open Docker Desktop.
2. In the left sidebar, look for **Docker Scout** (near Containers, Images, Volumes, Builds).
3. Click it.

### Step 2 — Analyze an Image via the GUI
1. From the dropdown/search, select one of your pushed images (e.g., `trainwithshubham/two-tier-backend:latest` from Chapter 9).
2. Docker Scout will automatically run a **vulnerability scan**.

**Expected result (example):**
```text
Critical: 0
High: 6
Medium: 6
Low: 52
```
This tells you: no critical issues, but 6 high-severity and 6 medium-severity issues exist somewhere in this image's layers — worth investigating.

### Step 3 — Analyze an Image via the CLI
```bash
docker images
```
Copy the exact image name you want to scan, then run:
```bash
docker scout quickview <IMAGE_NAME>:<TAG>
```
**What this does:** Authenticates with Docker Hub (make sure you're logged in via `docker login` first) and gives a quick summary of vulnerabilities, similar to the GUI view.

**Expected output (example):**
```text
Target: trainwithshubham/two-tier-backend:latest
   digest sha256:xxxx

  Critical  High     Medium   Low
  0         6        6        52
```
It will also usually show a breakdown by layer — for example, telling you that your **base image** (like `python:3.8-slim`) itself contributes some of these vulnerabilities (e.g., 3 high, 1 medium, 28 low), separate from vulnerabilities introduced by your own added packages.

### Step 4 — Deep-Dive Into Specific Vulnerabilities (CVEs)
```bash
docker scout cves <IMAGE_NAME>:<TAG>
```
**What this does:** CVE stands for **Common Vulnerabilities and Exposures** — a standardized reference system for known security issues. This command lists every specific vulnerability found, along with:
* The exact affected package.
* A reference link where you can read more details about that specific vulnerability.

**Expected output (example):**
```text
64 vulnerabilities found in 28 packages
  ✗ CVE-2023-xxxx  [High]  affects package: libssl
    -> https://... (link to details)
  ✗ CVE-2022-xxxx  [Medium] affects package: zlib
    -> https://... (link to details)
```

**What to do with this report:**
As a DevOps engineer, you don't necessarily fix the vulnerability code yourself — you share this report with your team/developers, who can then decide, for example, to switch to a different, more secure or updated base image, or update a specific vulnerable package.

---

## Part B — Docker Init (Auto-Generate Docker Boilerplate Files)

### Concept — Why Use Docker Init?

**The problem:**
Writing a Dockerfile, `docker-compose.yml`, and `.dockerignore` from scratch every single time takes effort — especially remembering all the small details (author labels, maintainer info, base image versions, etc.).

**The solution — Docker Init:**
`docker init` is an interactive command that asks you a few simple questions about your project (what language/platform, what port, etc.) and then automatically generates:
* A working `Dockerfile`
* A `docker-compose.yml`
* A `.dockerignore` file
* A `README.Docker.md` file with helpful commands

### Step 1 — Create a Test Folder
```bash
mkdir docker-init-test
cd docker-init-test
```

### Step 2 — Run Docker Init
```bash
docker init
```

### Step 3 — Answer the Interactive Questions

**Question 1: "What application platform does your project use?"**
Use the arrow keys to scroll through options (Node, Python, Go, Java, PHP, .NET, Rust, etc.) and select the one matching your project. Example: **Go**.

**Question 2: "What version of Go do you want to use?"**
Check the current latest stable version (e.g., search "Go latest version" if unsure) and enter it, e.g., `1.23`.

**Question 3: "What is the relative directory (with a leading .) that contains your go.mod file?"**
If your `go.mod` (or equivalent main package/entry file) is in the current directory, just enter:
```text
.
```

**Question 4: "What port does your server listen on?"**
Enter the port your application listens on internally, e.g., `8000`.

**Expected result:**
```text
✔ Created ./Dockerfile
✔ Created ./docker-compose.yml
✔ Created ./.dockerignore
✔ Created ./README.Docker.md
```
It may also show a warning like `no go.mod file was found` if your test folder is genuinely empty — this is expected for a pure test run, but it will still generate the boilerplate files correctly for you to fill in with your real project code afterward.

### Step 4 — Review the Generated Files
```bash
ls
cat docker-compose.yml
cat Dockerfile
cat README.Docker.md
```
**What you'll see:**
* `docker-compose.yml` → Includes commented sections showing example service definitions, build context, target port, and even example database service configuration (e.g., a commented-out PostgreSQL service block) that you can uncomment and adapt if needed.
* `Dockerfile` → A properly structured multi-stage Dockerfile (often including good practices like creating a non-root user for security) tailored to the platform you selected.
* `README.Docker.md` → Contains ready-made reference commands, such as the exact `docker build` command, the exact `docker run` command, and the exact `docker push` command for this project.

## Troubleshooting

### Error 1
```text
docker: 'scout' is not a docker command
```
**Reason:** Docker Scout CLI plugin is not installed/enabled in your Docker version.

**Fix:** Update Docker Desktop to the latest version, or install the Docker Scout CLI plugin manually — check Docker's official documentation for your OS.

### Error 2
```text
docker: 'init' is not a docker command
```
**Reason:** Similar to above — your Docker CLI version may be too old.

**Fix:** Update Docker Desktop / Docker Engine to a recent version that includes the `docker init` command.

### Error 3
Docker Scout shows "authentication required."
**Reason:** Not logged into Docker Hub.

**Fix:**
```bash
docker login
```

## Cleanup
```bash
cd ..
rm -rf docker-init-test
```

## Final Result
* You can scan any Docker image for security vulnerabilities using `docker scout quickview` and `docker scout cves`, both via CLI and Docker Desktop's GUI.
* You understand the difference between severity levels (Critical, High, Medium, Low) and how to act on a vulnerability report.
* You can use `docker init` to auto-generate a working Dockerfile, docker-compose.yml, .dockerignore, and README for a new project in seconds, based on simple interactive questions.

## What's Next
This completes the full "Docker in One Shot" course. The natural next step is a dedicated Kubernetes course, since Chapter 11 introduced why container orchestration is needed in production environments.
