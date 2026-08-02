# Docker Day 1: Hands-On Practical Guide (Installation, Images & Containers)

> **Note:** Modules below begin at Module 2 (as provided). Module 1 is reserved for Docker concepts & prerequisites — fill in with your intro/theory content.

---

## Module 1: Docker Concepts & Prerequisites *(placeholder)*

Cover the following before running commands:

- What Docker is and why containers vs. virtual machines.
- Core objects: **Images** (read-only templates) vs. **Containers** (running instances).
- Docker Hub registry basics.
- A single Linux (Ubuntu) EC2 instance with SSH access is enough for this practical.

---

## Module 2: Docker Engine Installation (Ubuntu Linux)

### Step 2: System Update & Docker Installation

```bash
sudo apt-get update
sudo apt-get install docker.io -y
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
docker --version
```

**Why we run this:** Installs Docker daemon service, enables automatic service startup on boot, grants non-root docker execution permissions to active user, and verifies version.

**Word Breakdown:**

- `apt-get install docker.io`: Downloads and installs core Docker engine packages.
- `-y`: Automatically accepts installation prompts.
- `systemctl enable --now`: Enables service on boot and starts it immediately.
- `usermod -aG docker $USER`: Adds current user (`$USER`) to `docker` group.

> **Tip:** After `usermod`, log out and back in (or run `newgrp docker`) so the new group membership takes effect and you can run Docker without `sudo`.

---

## Module 3: Docker Images Operations

### Step 3: Pull Image from Docker Hub Registry

```bash
docker pull nginx:latest
```

**Why we run this:** Downloads the read-only Nginx web server base template image layer by layer from Docker Hub public registry.

**Word Breakdown:**

- `docker`: Calls Docker command line tool.
- `pull`: Downloads image from remote registry to local host engine.
- `nginx:latest`: Image name (`nginx`) and tag version (`latest`).

### Step 4: List Local Images

```bash
docker images
```

**Why we run this:** Displays all locally downloaded read-only Docker images, including repository name, tag, image ID, and size.

**Word Breakdown:**

- `docker images`: Lists local cached container templates.

---

## Module 4: Docker Containers Operations

### Step 5: Run Nginx Container in Detached Mode with Port Mapping

```bash
docker run -d -p 8080:80 --name my-web-server nginx:latest
```

**Why we run this:** Creates and starts an isolated container instance from `nginx:latest` image in background mode, mapping host port 8080 to container port 80.

**Word Breakdown:**

- `run`: Creates and starts a container from specified image.
- `-d`: Detached mode (runs container silently in background).
- `-p 8080:80`: Port mapping flag (`HostPort:ContainerPort`).
- `--name my-web-server`: Assigns custom readable name to container.

> **Verify:** Open `http://<your-server-ip>:8080` in a browser (ensure port `8080` is allowed in the Security Group) to see the Nginx welcome page.

### Step 6: List Containers (Active & Stopped)

```bash
docker ps
docker ps -a
```

**Why we run this:** `docker ps` shows actively running containers; `docker ps -a` shows all containers including stopped/exited ones.

**Word Breakdown:**

- `ps`: Process status — shows active containers.
- `-a`: All flag — displays active and inactive container instances.

### Step 7: View Live Container Logs

```bash
docker logs -f my-web-server
```

**Why we run this:** Streams real-time stdout/stderr log output produced inside container for troubleshooting application behavior.

**Word Breakdown:**

- `logs`: Fetches console log records.
- `-f`: Follow flag — streams logs continuously in real time.

### Step 8: Stop, Start, and Force-Remove Container

```bash
docker stop my-web-server
docker start my-web-server
docker rm -f my-web-server
```

**Why we run this:** Gracefully stops container execution, restarts it, and forcefully removes container instance along with its writable layer.

**Word Breakdown:**

- `stop`: Gracefully stops running process.
- `start`: Resumes stopped container.
- `rm -f`: Forcefully deletes container.

---

## Module 5: Additional Docker Commands (Extra Day 1 Practice)

These commands round out the fundamentals and are excellent to practice in the same session.

### Step 9: Inspect Docker Service Status & Info

```bash
sudo systemctl status docker
docker info
```

**Why we run this:** Confirms the Docker daemon is running and healthy, and shows engine-wide details (containers count, images count, storage driver, root directory).

**Word Breakdown:**

- `systemctl status docker`: Shows the running state of the Docker service.
- `docker info`: Displays system-wide Docker engine configuration and statistics.

### Step 10: Execute Commands Inside a Running Container

```bash
docker exec -it my-web-server bash
```

**Why we run this:** Opens an interactive shell session inside the running container to inspect files, check configs, or debug from the inside.

**Word Breakdown:**

- `exec`: Runs a new command inside an already-running container.
- `-it`: Combines interactive (`-i`) and pseudo-TTY (`-t`) flags for a usable terminal.
- `bash`: The shell to launch inside the container. (Use `sh` if `bash` is unavailable.)

> **Tip:** Type `exit` to leave the container shell without stopping the container.

### Step 11: Inspect Low-Level Container / Image Details

```bash
docker inspect my-web-server
```

**Why we run this:** Returns detailed JSON metadata (network settings, mounts, environment variables, IP address, state) about a container or image.

**Word Breakdown:**

- `inspect`: Dumps full low-level configuration in JSON format.

### Step 12: View Container Resource Usage (Live)

```bash
docker stats
```

**Why we run this:** Streams a live view of CPU, memory, network, and disk I/O usage for running containers — useful for spotting resource-heavy workloads.

**Word Breakdown:**

- `stats`: Displays a real-time resource-usage dashboard for containers.

### Step 13: Rename & Restart a Container

```bash
docker rename my-web-server web-01
docker restart web-01
```

**Why we run this:** Renames a container to a clearer identifier and restarts it in one step (stop + start).

**Word Breakdown:**

- `rename`: Changes a container's assigned name.
- `restart`: Stops and then starts the container again.

### Step 14: Copy Files Between Host and Container

```bash
docker cp index.html web-01:/usr/share/nginx/html/index.html
docker cp web-01:/etc/nginx/nginx.conf ./nginx.conf
```

**Why we run this:** Transfers files into a container (e.g., a custom web page) or pulls files out of it (e.g., a config for review) without rebuilding the image.

**Word Breakdown:**

- `cp`: Copy files/folders between the host filesystem and a container.
- `web-01:/path`: `ContainerName:PathInsideContainer` reference.

### Step 15: Tag an Image

```bash
docker tag nginx:latest myrepo/nginx:v1
```

**Why we run this:** Creates a new named reference (tag) for an existing image — typically to prepare it for pushing to a personal/organization registry.

**Word Breakdown:**

- `tag`: Adds a new name/tag pointing to an existing image ID.
- `myrepo/nginx:v1`: Target `repository/image:tag` naming.

### Step 16: Remove Images

```bash
docker rmi nginx:latest
docker rmi -f nginx:latest
```

**Why we run this:** Deletes a local image to free disk space. `-f` forces removal even if the image is referenced by stopped containers.

**Word Breakdown:**

- `rmi`: Remove Image.
- `-f`: Force removal flag.

### Step 17: Clean Up Unused Docker Resources

```bash
docker container prune -f
docker image prune -f
docker system prune -a -f
```

**Why we run this:** Reclaims disk space by removing stopped containers, dangling images, and (with `-a`) all unused images, networks, and build cache.

**Word Breakdown:**

- `container prune`: Removes all stopped containers.
- `image prune`: Removes dangling (untagged) images.
- `system prune -a`: Removes all unused containers, networks, and images.
- `-f`: Skips the confirmation prompt.

### Step 18: View Image Build History & Disk Usage

```bash
docker history nginx:latest
docker system df
```

**Why we run this:** `docker history` shows the layer-by-layer build steps of an image; `docker system df` summarizes disk space used by images, containers, and volumes.

**Word Breakdown:**

- `history`: Lists the layers and commands that built an image.
- `system df`: Disk-usage report for Docker objects.

### Step 19: Search Docker Hub from CLI

```bash
docker search ubuntu
```

**Why we run this:** Searches the Docker Hub registry for available public images directly from the terminal.

**Word Breakdown:**

- `search`: Queries Docker Hub for image repositories matching a keyword.

### Step 20: Run an Interactive Throwaway Container

```bash
docker run -it --rm ubuntu:latest bash
```

**Why we run this:** Launches a temporary Ubuntu container with an interactive shell for quick testing; `--rm` auto-deletes it on exit so no clutter is left behind.

**Word Breakdown:**

- `-it`: Interactive terminal session.
- `--rm`: Automatically removes the container when it stops.
- `ubuntu:latest bash`: Image to run and the command (shell) to start.
