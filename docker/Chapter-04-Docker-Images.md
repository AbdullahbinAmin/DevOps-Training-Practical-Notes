# Chapter 4 — Docker Images

## Objective
Understand what a Docker Image is, how it differs from a Docker Container, log in to Docker Hub securely using a Personal Access Token, and pull/run pre-built images.

## Prerequisites
* OS: Ubuntu EC2 instance (from Chapter 3) or Docker Desktop locally
* Required software: Docker installed and working (`docker ps` runs without error)
* Required account: Docker Hub account
* Required ports: None yet
* Required files: None

## Pre-Check
```bash
docker --version
docker ps
```
**What this does:** Confirms Docker is installed and the daemon is reachable before proceeding.

## Concept — What is a Docker Image?

**Analogy used:** Think of an image like a "cheat sheet" (chit) you might have used in school/college — a small piece of paper with condensed notes. From that cheat sheet, you produce your actual answer sheet (the exam paper you submit). If you give that same cheat sheet to a friend, they can also produce a similar answer sheet.

**Applied to Docker:**
* The **Docker Image** is like the cheat sheet — a blueprint containing all the instructions, environment, code, and dependencies needed to run an application.
* The **Docker Container** is like the answer sheet — the actual running instance created from that image.
* You can take the same image and create as many containers from it as you like, and each one will behave identically.

**How an image is built:**
An image is built using a **Dockerfile**, which contains instructions (steps) describing what environment is needed, what code to include, and how to run the application. This will be covered fully in Chapter 5.

**The core "mantra" to remember:**
```text
Dockerfile  →  builds  →  Docker Image  →  run  →  Docker Container
```

You don't always have to write your own Dockerfile — you can also **pull** ready-made, pre-built images from a public registry like Docker Hub.

## Step 1 — View Existing Images
```bash
docker images
```
**What this does:** Lists all Docker images currently stored on your local machine/server. If none exist yet, the list will be empty except for column headers.

## Step 2 — Log In to Docker Hub

You can log in using your password, but it is **more reliable** to use a **Personal Access Token (PAT)** instead, since passwords are easy to forget and tokens can be regenerated anytime.

### Step 2a — Try Basic Login (for understanding)
```bash
docker login
```
It will prompt for your **Username** and **Password**. If you forget your password, this fails — this is exactly why a Personal Access Token is recommended.

### Step 2b — Generate a Personal Access Token (Recommended Method)
1. Go to https://hub.docker.com and log in.
2. Click on your profile icon → **Account Settings**.
3. Go to the **Personal Access Tokens** section.
4. Click **Generate New Token**.
5. Give it a name, e.g., `docker-in-one-shot`.
6. Set permissions: **Read, Write, Delete** (or as needed).
7. Click **Generate**, and **copy the token immediately** (it will not be shown again).

### Step 2c — Log In Using the Token
```bash
docker login
```
- **Username:** your Docker Hub username
- **Password:** paste the Personal Access Token you just copied (not your actual account password)

**Expected result:**
```text
Login Succeeded
```

**Why this matters:** Docker Hub is the central public registry where Docker images are stored (similar to how GitHub stores code repositories). Logging in lets you pull private images and push your own images later (covered in Chapter 9 — Docker Registry).

## Step 3 — Pull an Image from Docker Hub
```bash
docker pull hello-world
```
**What this does:** Downloads the `hello-world` image from Docker Hub to your local machine — similar to how `git clone` downloads a repository. This does NOT run the image; it only downloads it.

**Verify:**
```bash
docker images
```
**Expected result:** You should see `hello-world` listed with its image ID, tag, and size.

## Step 4 — Run a Container From That Image
```bash
docker run hello-world
```
**What this does:** Creates and starts a container from the `hello-world` image.

**Expected output:**
```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

**Important note:** You do NOT always need to run `docker pull` first. If the image is not found locally, `docker run` will automatically pull it first and then run it — as one combined step.

**Behind the scenes flow for `docker run hello-world`:**
1. Docker Client (CLI) contacts the Docker Daemon.
2. The Docker Daemon pulls the `hello-world` image (if not already local).
3. The Docker Daemon creates a container from that image.
4. The container's output is displayed on your terminal.

## Step 5 — Pull a Real-World Image (MySQL Database)
```bash
docker pull mysql
```
**What this does:** Downloads the official MySQL database image from Docker Hub.

**Verify:**
```bash
docker images
```
**Expected result:** `mysql` should now appear in the list of images (roughly 600+ MB in size, since it includes a full database engine).

## Step 6 — Run the MySQL Image as a Container (With a Required Environment Variable)

MySQL is a server and requires login credentials (username/password) to be set. This is passed using an **environment variable**.

```bash
docker run -e MYSQL_ROOT_PASSWORD=root mysql
```
- `-e MYSQL_ROOT_PASSWORD=root` → Sets an environment variable named `MYSQL_ROOT_PASSWORD` with the value `root` (this becomes the MySQL root user's password inside the container).
- `mysql` → The name of the image to run.

**Expected output:** You should see logs like:
```text
[Note] InnoDB: ... initialization has ended
[Note] mysqld: ready for connections
```

**Important behavior:** Once you run a container in this "attached" (foreground) mode, your terminal gets **blocked** by the running container — you cannot type any more commands in that same terminal window until the container stops.

## Step 7 — Unblocking Your Terminal (Two Methods)

### Method A — Open a Second Terminal / SSH Session
1. Open a new terminal tab/window.
2. SSH into the same EC2 instance again (repeat Chapter 3, Step 4).
3. In this new terminal, run:
```bash
docker ps
```
**Expected result:** You will see your MySQL container listed as running, with a status like "Up about a minute."

4. Stop the container using its Container ID:
```bash
docker stop <CONTAINER_ID>
```
- `<CONTAINER_ID>` → Copy this from the output of `docker ps`.

**What happens:** The container stops, and — importantly — your **original terminal (the blocked one) becomes unblocked automatically**, because the process that was blocking it has ended.

### Method B — Run in Detached (Background) Mode from the Start (Recommended)
```bash
docker run -d -e MYSQL_ROOT_PASSWORD=root mysql
```
- `-d` → Runs the container in **detached mode**, meaning it runs in the background and does not block your terminal.

**Verify:**
```bash
docker ps
```
**Expected result:** The MySQL container appears as running, and your terminal remains free to use.

## Step 8 — Viewing All Containers (Including Stopped Ones)
```bash
docker ps -a
```
- `-a` → Shows **all** containers, including ones that have exited/stopped, not just currently running ones.

**Why this matters:** A container that finishes its task (like `hello-world`, which just prints a message and exits) will not show up in a plain `docker ps`, but it WILL show up in `docker ps -a` with a status of "Exited."

## Concept — Why Some Containers Stop Immediately

**Explanation:** Every container's lifecycle depends on the command it was told to run. The `hello-world` image's job is simply to print a message and then finish — so its container exits right after. This is expected behavior, not an error.

To keep a container running continuously (like a server), you must use flags that keep it alive, for example:
```bash
docker run -itd ubuntu
```
- `-i` → Interactive mode (keeps STDIN open).
- `-t` → Allocates a pseudo-terminal (TTY), so the container behaves like an interactive shell.
- `-d` → Detached (background) mode.

Without `-itd`, a plain OS image like `ubuntu` will start, have nothing to actively do, and immediately exit.

**Verify:**
```bash
docker ps
```
**Expected result:** The `ubuntu` container should now show as running continuously, because it was given an interactive terminal to keep it alive.

## Troubleshooting

### Error 1
```text
permission denied while trying to connect to the Docker daemon socket
```
**Reason:** Your user isn't part of the `docker` group yet (see Chapter 3, Step 9 for the fix).

**Fix:**
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Error 2
```text
Error response from daemon: Get "https://registry-1.docker.io/v2/": unauthorized: incorrect username or password
```
**Reason:** Wrong password was used for `docker login`.

**Fix:** Generate a Personal Access Token from Docker Hub (see Step 2b) and use that instead of your actual account password.

## Cleanup (Optional, for this chapter)
```bash
docker ps -a
docker stop <CONTAINER_ID>
docker rm <CONTAINER_ID>
```
**What this does:** Stops and removes any test containers created during this chapter (`hello-world`, `mysql`, `ubuntu`) to keep your environment clean before the next chapter.

## Final Result
* You are logged in to Docker Hub using a Personal Access Token.
* `hello-world` image pulled and run successfully.
* `mysql` image pulled and run successfully in detached mode.
* You understand the difference between an Image (blueprint) and a Container (running instance), and how `-d`, `-e`, `-i`, `-t` flags work.

## What's Next
Chapter 5 will teach you how to write your own Dockerfile from scratch to containerize a real Java application, including how images and containers are built layer by layer.
