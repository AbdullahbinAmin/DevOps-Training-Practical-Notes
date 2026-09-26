# Chapter 9 — Docker Registry (Push/Pull to Docker Hub)

## Objective
Learn how to tag your local Docker image correctly, push it to Docker Hub (a Docker Registry), pull it back down on any machine, and update your Docker Compose file to use the pushed image instead of always rebuilding locally. Also learn how to clean up unused images/containers/volumes.

## Prerequisites
* OS: Ubuntu EC2 instance with Docker installed
* Required software: Docker
* Required account: Docker Hub account with a Personal Access Token (from Chapter 4)
* Required setup: The `two-tier-backend` image built in Chapters 6–8

## Concept — What is a Docker Registry?

**The problem:**
All the images you've built so far (`two-tier-backend`, `java-app`, `flask-app`) exist only on YOUR local machine/server. If you want to run them on a different server, or share them with a teammate, you need a central place to store and retrieve them.

**The solution — Docker Registry:**
A Docker Registry is a storage and distribution system for Docker images. **Docker Hub** is the most well-known public registry (similar in concept to how GitHub hosts code repositories, but for container images).

## Step 1 — Confirm You Are Logged In
```bash
docker login
```
Enter your Docker Hub username and your Personal Access Token as the password (see Chapter 4, Step 2, if you need to generate one).

**Expected result:**
```text
Login Succeeded
```

## Step 2 — Tag Your Image Correctly

**Concept:** `docker image tag` does not create a duplicate copy of the image data — it simply gives your existing image an **additional name/label**. To push an image to Docker Hub, its name MUST be prefixed with your Docker Hub username, in this format:
```text
<your-dockerhub-username>/<image-name>:<tag>
```

```bash
docker image tag two-tier-flask:latest trainwithshubham/two-tier-backend:latest
```
- `two-tier-flask:latest` → Your existing local image name and tag (replace with your actual image name — for example, `two-tier-backend:latest` if that's what you built in Chapter 6/8).
- `trainwithshubham/two-tier-backend:latest` → The new tag, where `trainwithshubham` is replaced with **your own Docker Hub username**.

**Verify:**
```bash
docker images
```
**Expected result:** You will now see TWO entries — the old plain name AND the new fully-qualified tagged name — both pointing to the same underlying image (same Image ID).

## Step 3 — Push the Image to Docker Hub
```bash
docker push trainwithshubham/two-tier-backend:latest
```
**What this does:** Uploads your locally built and tagged image from your machine/server up to your Docker Hub account's repository.

**Expected output:** Progress bars showing each image layer being pushed, ending with something like:
```text
latest: digest: sha256:xxxxxxxx size: xxxx
```

## Step 4 — Verify on Docker Hub Website
1. Go to https://hub.docker.com and log in.
2. Navigate to your repositories.
3. You should see `two-tier-backend` listed, with "Last pushed less than a minute ago."

## Step 5 — Use the Pushed Image Directly in Docker Compose (Instead of Building Locally)

Instead of using `build: context: .` in your `docker-compose.yml` (which rebuilds every time), you can now reference the image directly from Docker Hub:

```yaml
services:
  flaskapp:
    image: trainwithshubham/two-tier-backend:latest
    container_name: two-tier-backend
    ports:
      - "5000:5000"
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: root
      MYSQL_DB: devops
    networks:
      - two-tier
    depends_on:
      mysql:
        condition: service_healthy
    restart: always
```
**What changed:** The `build:` key is removed, and `image: trainwithshubham/two-tier-backend:latest` is used instead. Now, when you run this Compose file — even on a completely different machine — it will pull this image directly from Docker Hub rather than needing your source code and Dockerfile locally.

**Bring it up:**
```bash
docker compose down
docker compose up -d
```
**Expected result:** The image is pulled from Docker Hub instead of built locally. Anyone on any machine with this same `docker-compose.yml` file can run the exact same application, without needing your source code at all.

## Step 6 — Pull the Image Yourself (Simulating a Different Machine/Teammate)
```bash
docker pull trainwithshubham/two-tier-backend:latest
```
If not already logged in on this machine:
```bash
docker login
```
Then retry the pull.

**Expected result:** The image downloads successfully, proving anyone with Docker Hub access can now run your application.

## Step 7 — Making a Repository Private (Optional)
By default, pushed repositories can be **public** (anyone can pull) or **private** (only you/your team can pull).
1. On Docker Hub, go to your repository → **Settings**.
2. Toggle visibility to **Private** if you don't want it publicly accessible.

> ⚠️ Note: Free Docker Hub accounts have a limit on the number of private repositories. Check current limits on Docker Hub's pricing page if this matters to you.

## Step 8 — Cleaning Up Unused Docker Resources

As you build many images and containers throughout this course, your disk will accumulate unused/leftover items. Docker provides commands to clean these up.

### Remove All Stopped Containers, Unused Networks, Dangling Images, and Build Cache in One Command
```bash
docker system prune
```
**What this does:** Docker will show a warning listing what will be removed:
```text
WARNING! This will remove:
  - all stopped containers
  - all networks not used by at least one container
  - all dangling images
  - all dangling build cache
```
Type `y` (or press Enter) to confirm.

### Remove a Specific Image
```bash
docker rmi <IMAGE_ID>
```
**What this does:** Removes one specific image by its ID.

### Remove ALL Images at Once
```bash
docker rmi -f $(docker images -aq)
```
**What this does:**
- `docker images -aq` → Lists only the **IDs** (`-q` = quiet, only IDs) of ALL images (`-a` = all).
- The `$(...)` syntax substitutes that list of IDs as arguments into the outer command.
- `docker rmi -f` → Force-removes each image ID passed to it.

**Expected result:** Every image on the system is deleted (except ones still in use by a running or stopped-but-not-removed container — remove those containers first).

### Remove All Stopped Containers via Compose
```bash
docker compose down
```
**What this does:** For any project managed with Compose, this stops and removes exactly the containers/networks defined by that Compose file in one command.

### Force a Full Rebuild + Re-pull
```bash
docker compose up -d --build
```
**What this does:** Forces Docker Compose to rebuild any services that use `build:`, and re-pull/verify any services that use `image:`.

## Troubleshooting

### Error 1
```text
denied: requested access to the resource is denied
```
**Reason:** You are not logged in, or your image tag's username prefix does not match your actual logged-in Docker Hub username.

**Fix:** Run `docker login` again, and double check the tag format: `docker image tag <local-name> <your-exact-dockerhub-username>/<repo-name>:<tag>`.

### Error 2
```text
unauthorized: incorrect username or password
```
**Reason:** Using your real account password instead of a Personal Access Token, or an expired/revoked token.

**Fix:** Generate a fresh Personal Access Token from Docker Hub (Chapter 4, Step 2b) and use that as the password.

### Error 3
```text
Error response from daemon: conflict: unable to remove repository reference "..." (must force) - container ... is using its referenced image
```
**Reason:** You are trying to delete an image that is still being used by an existing (even stopped) container.

**Fix:** Remove the container first (`docker rm <container_id>`), then remove the image.

## Cleanup
```bash
docker compose down
docker system prune -a
```
**What this does:** Tears down the running application and removes ALL unused Docker resources (containers, networks, images, build cache) — use `-a` carefully, since it also removes images not currently referenced by any container.

## Final Result
* Your application's image is tagged correctly with your Docker Hub username.
* The image is pushed to Docker Hub and visible on your Docker Hub profile.
* Your `docker-compose.yml` now references the image directly from Docker Hub (`image:` instead of `build:`).
* You can pull and run this image from any machine with Docker installed.
* You know how to clean up unused Docker resources using `docker system prune`, `docker rmi`, and `docker compose down`.

## What's Next
Chapter 10 covers Multi-stage Docker Builds — a technique to dramatically shrink your image sizes (for example, reducing a 1 GB image down to under 150 MB).
