# Chapter 6 — Docker Networking

## Objective
Understand why isolated containers cannot talk to each other by default, learn the different Docker network types, create a custom network, and connect a Flask app container to a MySQL container using that network (a "two-tier" application).

## Prerequisites
* OS: Ubuntu EC2 instance with Docker installed
* Required software: Docker, git
* Required repository: A "two-tier Flask app" repository (example: `laundeShubham153/two-tier-flask-app`)
* Required ports: Port 5000 (opened via AWS Security Group)
* Required knowledge: Chapters 4 and 5 (images, containers, Dockerfile)

## Concept — Why Containers Are Isolated by Default

**The problem:**
Imagine you have two separate Docker containers running on the same computer:
* Container A → a Flask app
* Container B → a MySQL database

By default, **each container is isolated** — Container A cannot talk to Container B, and Container B cannot talk to Container A, even though they are on the same physical machine.

**The solution:** You need a **Docker Network** to let them communicate.

## Concept — The 7 Types of Docker Networks (Drivers)

Docker officially supports several network drivers. Here they are, from most important (used daily) to least important (advanced/legacy):

| # | Network Type | Description |
|---|---|---|
| 1 | **host** | The container shares the exact same network as the host machine. If the container listens on port 80, it directly uses the host's port 80 — no separate mapping needed. |
| 2 | **bridge** (default) | The default network Docker creates automatically. Acts like a "bridge" connecting the host to containers — this is what makes port mapping (`-p host:container`) work. |
| 3 | **user-defined bridge** (custom bridge) | You create your own named bridge network. Containers attached to the same custom bridge network can reach each other by container name, not just via the host. |
| 4 | **none** | The container has NO network at all — completely isolated, no internet, no communication with anything. Useful when a container just needs to run something internally without any external access. |
| 5 | macvlan | Assigns containers their own MAC address so they behave like physical devices on the network. Used mainly in **Docker Swarm** clusters. |
| 6 | ipvlan | Similar concept to macvlan, also mainly used with Docker Swarm. |
| 7 | overlay | Used for multi-host networking in Docker Swarm clusters. |

> ⚠️ The last three (macvlan, ipvlan, overlay) are mostly outdated for typical use because most orchestration today is done with Kubernetes instead of Docker Swarm. The four important ones to focus on are: **host, bridge, user-defined/custom bridge, and none.**

## Step 1 — List Existing Networks
```bash
docker network ls
```
**Expected result:** By default, you should see three networks already present:
```text
NETWORK ID     NAME      DRIVER    SCOPE
xxxxxxxxxxxx   bridge    bridge    local
xxxxxxxxxxxx   host      host      local
xxxxxxxxxxxx   none      null      local
```

## Step 2 — Create a Custom (User-Defined) Network
```bash
docker network create mynetwork -d bridge
```
- `mynetwork` → The name you are giving your custom network (choose any name).
- `-d bridge` → Specifies the **driver** to use — here, `bridge` (a custom bridge network).

**Verify:**
```bash
docker network ls
```
**Expected result:** `mynetwork` should now appear, listed with driver type `bridge`.

## Step 3 — The Two-Tier Project (Flask + MySQL)

### Directory Structure
```text
two-tier-flask-app/
├── app.py
├── Dockerfile
└── requirements.txt
```

### Step 3a — Clone the Project
```bash
cd ~/projects
git clone https://github.com/LaundeShubham153/two-tier-flask-app
cd two-tier-flask-app
```

### Step 3b — Understand the App's Database Connection
```bash
cat app.py
```
**What to look for:** The code reads database connection details from **environment variables**. In this example, four environment variables are required:
* `MYSQL_HOST`
* `MYSQL_USER`
* `MYSQL_PASSWORD`
* `MYSQL_DB`

**Why this matters:** As a DevOps engineer, you don't need to fully understand the application's internal logic (that is the developer's job). You only need to know: **does this app connect to a database, and which environment variables does it need to do so?**

### Step 3c — Understand the Dockerfile
```bash
cat Dockerfile
```
**Expected content (example):**
```dockerfile
FROM python:3.8-slim
WORKDIR /app
COPY requirements.txt .
RUN apt update && apt upgrade -y
RUN apt install -y default-mysql-client
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```
**Explanation:**
* `python:3.8-slim` → A smaller ("slim") version of the Python 3.8 base image — smaller size than the full Python image.
* `apt update && apt upgrade -y` → Updates the container's OS packages.
* `apt install -y default-mysql-client` → Installs a MySQL client library, since this app needs to talk to a MySQL server.
* `pip install -r requirements.txt` → Installs Python dependencies.
* `CMD ["python", "app.py"]` → Runs the Flask app when the container starts.

## Step 4 — Build the Flask App's Image
```bash
docker build -t two-tier-backend .
```
**Expected result:** Image builds successfully and appears under `docker images` as `two-tier-backend`.

## Step 5 — First Attempt: Run WITHOUT a Shared Network (To See the Failure)

```bash
docker run -d --name mysql --network two-tier -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=devops mysql:latest
```
*(If you have not created a network called `two-tier` yet, this specific command will fail — that is intentional, to demonstrate the concept step by step. Follow Step 6 below for the correct working version.)*

## Step 6 — The Correct Working Setup

### Step 6a — Create a Dedicated Network for This App
```bash
docker network create two-tier -d bridge
```

### Step 6b — Run the MySQL Container on This Network
```bash
docker run -d \
  --name mysql \
  --network two-tier \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=devops \
  mysql:latest
```
- `--name mysql` → Names this container `mysql` (this name will act as the hostname other containers use to reach it).
- `--network two-tier` → Attaches this container to the `two-tier` custom network.
- `-e MYSQL_ROOT_PASSWORD=root` → Sets the MySQL root password.
- `-e MYSQL_DATABASE=devops` → Creates a database named `devops` automatically on startup.

**Verify:**
```bash
docker ps
```
**Expected result:** The `mysql` container should show as running.

### Step 6c — Run the Flask App Container on the SAME Network
```bash
docker run -d \
  --name flaskapp \
  --network two-tier \
  -p 5000:5000 \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=root \
  -e MYSQL_DB=devops \
  two-tier-backend:latest
```
- `--network two-tier` → **This is the critical part** — both containers must be on the exact same custom network to reach each other.
- `-e MYSQL_HOST=mysql` → **The value here must exactly match the `--name` given to the MySQL container.** Docker's custom bridge network automatically resolves container names to their internal IP addresses (this is called **DNS-based service discovery**), so `mysql` as a hostname will correctly route to the MySQL container.
- `-p 5000:5000` → Maps host port 5000 to container port 5000 (the Flask app's port in this example).

## Step 7 — Verify Everything is Running and Connected
```bash
docker ps
```
**Expected result:** Both `mysql` and `flaskapp` containers show as running (Up).

```bash
docker logs flaskapp
```
**Expected result:** No connection errors — the app should be logged as running cleanly, with no `Unknown MySQL server host` errors.

## Step 8 — Inspect the Network to Confirm Both Containers are Connected
```bash
docker network inspect two-tier
```
**What this does:** Shows detailed JSON output about the network, including a `"Containers"` section listing every container attached to it.

**Expected result:** You should see both the `mysql` container and the `flaskapp` container listed under this network, confirming they can communicate with each other.

## Step 9 — Open the Port and Test in Browser

1. Go to your EC2 instance's Security Group → **Inbound rules → Add rule**.
2. Add: Custom TCP, Port **5000**, Source **Anywhere**.
3. Save.

Open in your browser:
```text
http://<YOUR_EC2_PUBLIC_IP>:5000
```
**Expected result:** The application loads (e.g., "Let's make a two-tier application, Flask and MySQL").

Submit a form field (e.g., type "Hello friends" and click Submit).

## Step 10 — Verify the Data Actually Reached the Database

```bash
docker ps
docker exec -it <MYSQL_CONTAINER_ID> bash
```
Inside the container's shell:
```bash
mysql -u root -p
```
When prompted, enter the password: `root`

```sql
SHOW DATABASES;
USE devops;
SELECT * FROM messages;
```
**Expected result:** The message you submitted through the web form (e.g., "Hello friends") appears as a row in the `messages` table — confirming that:
1. The front-end (Flask app) is correctly connected to the back-end (MySQL database) using Docker networking.
2. Data flows correctly through the whole stack.

Type `exit` twice to leave the MySQL prompt and then the container shell.

## Troubleshooting

### Error 1
```text
sqlalchemy.exc.OperationalError: (MySQLdb.OperationalError) (2005, "Unknown MySQL server host 'mysql'")
```
**Reason:** The Flask app container and the MySQL container are NOT on the same custom Docker network (or the `MYSQL_HOST` value does not match the MySQL container's `--name`).

**Fix:** Make sure both containers use `--network two-tier` (or whatever custom network name you chose), and that `MYSQL_HOST` exactly matches the MySQL container's `--name` value.

### Error 2
```text
docker: Error response from daemon: network with name two-tier already exists
```
**Reason:** You tried to create the same network twice.

**Fix:** Skip re-creating it, or check existing networks first with `docker network ls`.

### Error 3
Application loads but submitted form data doesn't appear in the database.
**Reason:** Wrong database name in `MYSQL_DB` environment variable, or wrong table name when querying.
**Fix:** Confirm `MYSQL_DATABASE` (set on the MySQL container) exactly matches `MYSQL_DB` (set on the app container), and check the app's code/README for the correct table name.

## Cleanup
```bash
docker stop flaskapp mysql
docker rm flaskapp mysql
docker network rm two-tier mynetwork
```
**What this does:** Stops and removes both containers and the custom networks created in this chapter.

## Final Result
* A custom Docker network is created.
* A MySQL container and a Flask app container are both attached to the same custom network.
* The Flask app successfully connects to MySQL using the container's name as the hostname.
* Data submitted through the web app is verified to be stored correctly inside the MySQL database.

## What's Next
Chapter 7 covers Docker Volumes and Storage — how to make sure your data survives even if a container is deleted or restarted.
