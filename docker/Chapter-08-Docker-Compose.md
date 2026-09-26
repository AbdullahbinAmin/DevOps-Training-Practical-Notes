# Chapter 8 — Docker Compose

## Objective
Automate everything learned so far — building images, running containers, networking, volumes, health checks, and dependency ordering — into a single YAML configuration file, controlled by one command instead of many manual `docker build`/`docker run` commands.

## Prerequisites
* OS: Ubuntu EC2 instance with Docker installed
* Required software: Docker, `docker-compose-plugin` (installed in this chapter)
* Required setup: The two-tier Flask + MySQL project from Chapters 6–7
* Required knowledge: Docker images, networks, volumes (Chapters 4, 6, 7)

## Concept — Why Docker Compose?

**The problem:**
So far, every time you wanted to run this two-tier application, you had to manually run several separate commands in the correct order:
```bash
docker network create two-tier
docker volume create mysql_data
docker run -d --name mysql --network two-tier -e ... -v ... mysql:latest
docker build -t two-tier-backend .
docker run -d --name flaskapp --network two-tier -p 5000:5000 -e ... two-tier-backend
```
As a DevOps engineer, doing this manually every single time is repetitive and error-prone. The goal is to **automate as much as possible**.

**The solution — Docker Compose:**
Docker Compose lets you describe your **entire multi-container application** in a single configuration file (a YAML file, usually named `docker-compose.yml`), and then bring up (or tear down) the whole application with a single command.

**What YAML means:**
YAML stands for "YAML Ain't Markup Language" (a recursive acronym) — it's a simple, human-readable format using key-value pairs and indentation, commonly used for configuration files.

## Step 1 — Install Docker Compose
```bash
sudo apt-get install docker-compose-v2
```
**What this does:** Installs Docker Compose version 2 (the current, actively maintained version — the older `docker-compose` v1 syntax is considered legacy).

**Verify:**
```bash
docker compose version
```
**Expected result:** Version info is printed (no error), confirming Compose is installed. (Note: Compose v2 uses `docker compose` — with a space — instead of the old `docker-compose` with a hyphen.)

## Step 2 — Directory / File Structure
```text
two-tier-flask-app/
├── app.py
├── Dockerfile
├── requirements.txt
└── docker-compose.yml
```

## Step 3 — Write the Compose File (First Attempt, Building It Up Piece by Piece)

```bash
cd ~/projects/two-tier-flask-app
nano docker-compose.yml
```

### Full docker-compose.yml (Final, Correct Version)
```yaml
version: "3.8"

services:
  mysql:
    image: mysql:latest
    container_name: mysql
    environment:
      MYSQL_DATABASE: devops
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - two-tier
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-proot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: always

  flaskapp:
    build:
      context: .
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

volumes:
  mysql_data:

networks:
  two-tier:
```

### Explanation of Each Section

| Section | Meaning |
|---|---|
| `version: "3.8"` | The Compose file format version being used. |
| `services:` | Lists every container you want to create. Here we define two: `mysql` and `flaskapp`. |
| `image: mysql:latest` | For the `mysql` service, use a pre-built public image directly (no Dockerfile needed for this one). |
| `container_name:` | Same as `--name` in `docker run` — sets a fixed, predictable container name. |
| `environment:` | Same as `-e` flags in `docker run` — sets environment variables. |
| `volumes:` (inside a service) | Same as `-v` in `docker run` — mounts a volume to a container path. |
| `networks:` (inside a service) | Attaches this service/container to the named network defined at the bottom of the file. |
| `ports:` | Same as `-p` in `docker run` — maps host port to container port. |
| `healthcheck:` | Defines a test command Compose runs repeatedly to check if the service is actually ready (not just "started"), covered in detail below. |
| `restart: always` | Automatically restarts this container if it crashes or if Docker restarts. |
| `build: context: .` | For the `flaskapp` service (which has no `image:` line), Compose will build an image from the Dockerfile found in the given context folder (`.` = current directory). |
| `depends_on:` | Controls startup order — `flaskapp` will wait for `mysql` before starting. |
| `volumes:` (top-level) | Declares the named volumes used anywhere in this file (`mysql_data`), similar to running `docker volume create`. |
| `networks:` (top-level) | Declares the named networks used anywhere in this file (`two-tier`), similar to running `docker network create`. |

**Save and exit** (`Ctrl+X`, `Y`, `Enter`).

## Step 4 — Why `depends_on` Alone Is NOT Enough (Important Concept)

If you only use plain `depends_on: - mysql` (without a health check condition), Compose will start the MySQL container FIRST and the Flask app SECOND — but "started" does not mean "ready to accept connections." MySQL takes a few extra seconds internally to fully initialize before it can accept connections. If the Flask app tries to connect too early, it will fail with an error like:
```text
Can't connect to MySQL server on 'mysql'
```

**The fix — Health Checks:**
A **health check** is a test command that Docker Compose runs repeatedly (at a set interval) against a container to determine if it is *actually* ready — not just "process started," but "genuinely accepting connections."

**Health check fields explained:**
* `test:` → The actual command to run inside the container to test readiness (here, `mysqladmin ping` checks if MySQL responds).
* `interval: 10s` → Run this test every 10 seconds.
* `timeout: 5s` → If the test takes longer than 5 seconds, consider it failed for that attempt.
* `retries: 5` → Try up to 5 times before giving up and marking the container "unhealthy."
* `start_period: 30s` → Give the container up to 30 seconds of grace period at startup before health check failures start counting.

By setting `depends_on: mysql: condition: service_healthy`, the `flaskapp` service will only start AFTER MySQL's health check reports "healthy" — completely solving the timing problem.

## Step 5 — Bring Up the Whole Application
```bash
docker compose up
```
**What this does:** Reads `docker-compose.yml`, then automatically:
1. Creates the `two-tier` network (if it doesn't exist).
2. Creates the `mysql_data` volume (if it doesn't exist).
3. Pulls the `mysql` image and starts the `mysql` container with all its settings.
4. Waits until MySQL's health check passes.
5. Builds the `flaskapp` image from your Dockerfile.
6. Starts the `flaskapp` container, connected to the same network.

**Expected output (final lines):**
```text
mysql        | ... ready for connections
flaskapp     | ... Running on http://0.0.0.0:5000
```

## Step 6 — Run in Detached (Background) Mode
Press `Ctrl+C` to stop the foreground run, then:
```bash
docker compose up -d
```
**What this does:** Same as above but runs everything in the background, freeing your terminal.

**Verify:**
```bash
docker ps
```
**Expected result:** Both `mysql` and `two-tier-backend` (flaskapp) containers show as `Up` and `healthy`.

## Step 7 — Force a Rebuild (Important When You Change Code or the Dockerfile)
```bash
docker compose up -d --build
```
- `--build` → Forces Compose to rebuild images instead of using cached ones. **Always use this flag after changing your Dockerfile or application source code** — otherwise Compose may reuse an old cached image and your changes won't appear.

## Step 8 — Test the Application
1. Open the AWS Security Group and confirm port **5000** is open (from Chapter 6).
2. Browse to:
```text
http://<YOUR_EC2_PUBLIC_IP>:5000
```
3. Submit test data through the form.
4. Restart everything and confirm data survives:
```bash
docker compose down
docker compose up -d
```
5. Refresh the browser — your previously submitted data should still be there (because the `mysql_data` volume persisted it, even though the containers were fully removed and recreated by `docker compose down`).

## Step 9 — Understanding `docker compose down` vs `docker compose up`
```bash
docker compose down
```
**What this does:** Stops AND removes all containers defined in the Compose file (but does NOT remove named volumes by default, so your data is safe).

```bash
docker compose up -d
```
**What this does:** Recreates and starts everything again from the same configuration file.

## Troubleshooting

### Error 1
```text
yaml: line X: mapping values are not allowed in this context
```
**Reason:** Incorrect YAML indentation, or a typo like using a tab instead of spaces, or accidentally adding an extra colon.

**Fix:** Carefully re-check indentation (YAML requires consistent spaces, not tabs) and compare against the working example above.

### Error 2
```text
service "flaskapp" depends on undefined service "mysql": invalid compose project
```
**Reason:** The service name in `depends_on:` does not exactly match the service name defined under `services:`.

**Fix:** Make sure the name under `depends_on:` matches character-for-character with the actual service name key (e.g., both must say `mysql`, not one saying `mysql_db` and the other `mysql`).

### Error 3
```text
network <name> declared as external, cannot be created
```
**Reason:** You referenced a network as `external: true` but it doesn't actually exist yet outside Compose.

**Fix:** Remove the `external: true` flag if you want Compose to create and manage the network itself, or manually create the network first with `docker network create <name>` if you truly need to reuse an existing one.

### Error 4
```text
Error response from daemon: Ports are not available: exposing port TCP 0.0.0.0:3306 -> ... port is already allocated
```
**Reason:** Another MySQL container (from a previous chapter) is still running and using port 3306.

**Fix:** Run `docker ps -a`, then stop/remove the older container: `docker stop <id> && docker rm <id>`.

## Cleanup
```bash
docker compose down
```
**What this does:** Stops and removes all containers created by this Compose file, keeping the named volume intact for future reuse.

To remove everything including volumes (careful — this deletes your data):
```bash
docker compose down -v
```

## Final Result
* A single `docker-compose.yml` file fully describes the two-tier application (Flask + MySQL).
* `docker compose up -d` brings up both containers, correctly networked, volumed, and health-checked, in the right startup order — with just one command.
* `docker compose down` cleanly tears everything down.
* Data persists across `down`/`up` cycles thanks to the named volume.

## What's Next
Chapter 9 covers Docker Registry — how to tag and push your own images to Docker Hub so they can be pulled and run from anywhere, and how to reference a pushed image directly inside Docker Compose instead of always building locally.
