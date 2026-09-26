# Chapter 12 — Project 1: Deploying a 3-Tier Application (Nginx + Django + MySQL)

## Objective
Deploy a complete 3-tier web application — Nginx (reverse proxy) + Django "Notes App" (backend) + MySQL (database) — fully automated using Docker Compose, with proper networking, health checks, and dependency ordering between all three containers.

## Prerequisites
* OS: Ubuntu EC2 instance with Docker + Docker Compose installed
* Required software: Docker, Docker Compose, git
* Required repository: A Django "Notes App" project (example: `laundeShubham153/notes-app`)
* Required ports: Port 80 (opened via AWS Security Group)
* Required knowledge: Chapters 5–9 (Dockerfile, Networking, Volumes, Compose, Registry)

## Architecture / Flow
```text
User Browser
     │  (HTTP request to Port 80)
     ▼
 Nginx Container (reverse proxy)
     │  routes "/" requests internally
     ▼
 Django Container ("notes_container", port 8000 internally)
     │  reads/writes data
     ▼
 MySQL Container ("mysql", port 3306)
```

**Why Nginx is used:**
The Django app itself runs on an internal port (e.g., 8000). Instead of exposing that port directly to the internet, **Nginx acts as a reverse proxy**: it listens on the standard web port (80) and internally forwards ("redirects") incoming requests to the Django app's port. This means users can access your app with a plain URL (no port number needed) while your backend can run on any internal port you like.

## Directory / File Structure
```text
notes-app/
├── notes_app/                (Django backend code)
│   ├── settings.py
│   ├── ... 
│   └── manage.py
├── my_notes/                 (Frontend, already built HTML/CSS/JS)
│   └── build/
├── nginx/
│   └── default.conf
├── Dockerfile
├── docker-compose.yml
└── .env
```

## Step 1 — Clone the Project
```bash
cd ~/projects
git clone https://github.com/LaundeShubham153/notes-app
cd notes-app
```

## Step 2 — Understand the Project Components

| Folder/File | Purpose |
|---|---|
| `notes_app/` (backend, Django) | Contains the actual application logic, written in the Django framework. |
| `my_notes/` (frontend) | Already pre-built front-end HTML/CSS/JS files. |
| `nginx/` | Holds Nginx's configuration file, which controls request routing. |
| `settings.py` | Django's main configuration file, showing where the front-end build files are located (`my_notes/build`). |
| `.env` | Contains environment variables needed for the database connection (DB name, user, password, host). |

## Step 3 — Understand the `.env` File
```bash
cat .env
```
**Expected content (example):**
```env
DB_NAME=test_db
DB_USER=root
DB_PASSWORD=root
DB_HOST=mysql
DB_PORT=3306
```
**Why this matters:** `DB_HOST` must exactly match the **container name** you will give to your MySQL container (this is the same DNS-based service discovery concept from Chapter 6).

## Step 4 — Understand the Existing Dockerfile
```bash
cat Dockerfile
```
**Expected content (similar structure to Chapter 5/6):**
```dockerfile
FROM python:3.7
WORKDIR /app/backend
COPY requirements.txt /app/backend
RUN apt update && apt upgrade -y
RUN apt install -y default-libmysqlclient-dev
RUN pip install mysqlclient
RUN pip install -r requirements.txt
COPY . /app/backend
EXPOSE 8000
```
**Explanation:**
* `default-libmysqlclient-dev` and `mysqlclient` → Required so Python/Django can talk to a MySQL database.
* `EXPOSE 8000` → Documents that this app runs on port 8000 internally (this is informational; actual port publishing still happens via Compose `ports:`).
* No `CMD` is present yet on purpose — this app will be run through a specific command defined later, directly inside `docker-compose.yml`.

## Step 5 — Understand the Nginx Configuration File
```bash
cat nginx/default.conf
```
**Expected content (example):**
```nginx
server {
    listen 80;

    location / {
        proxy_pass http://notes_container:8000;
    }
}
```
**Explanation:**
* `listen 80;` → Nginx listens on port 80 (the standard web port).
* `location / { proxy_pass http://notes_container:8000; }` → Any request to `/` gets internally forwarded to `notes_container` (the Django container's name) on port 8000.

> ⚠️ Critical rule: `notes_container` here MUST exactly match the `container_name` you assign to your Django service in `docker-compose.yml` later. This is the single most common mistake in this project.

## Step 6 — Write the Dockerfile for Nginx
```bash
nano nginx/Dockerfile
```
```dockerfile
FROM nginx:1.23-alpine
COPY default.conf /etc/nginx/conf.d/default.conf
```
**Explanation:**
* `FROM nginx:1.23-alpine` → A lightweight official Nginx image.
* `COPY default.conf /etc/nginx/conf.d/default.conf` → Copies your custom routing configuration into Nginx's expected configuration folder inside the container, overwriting the default one.

## Step 7 — Write the Full docker-compose.yml

```bash
nano docker-compose.yml
```

```yaml
version: "3.8"

services:
  nginx:
    build:
      context: ./nginx
    container_name: nginx_container
    ports:
      - "80:80"
    restart: always
    depends_on:
      - django

  django:
    build:
      context: .
    container_name: notes_container
    command: >
      sh -c "python manage.py migrate --noinput &&
             gunicorn notes_app.wsgi:application --bind 0.0.0.0:8000"
    env_file:
      - .env
    ports:
      - "8000:8000"
    networks:
      - notes_app
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      mysql:
        condition: service_healthy

  mysql:
    image: mysql:latest
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: test_db
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - notes_app
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-proot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: always

volumes:
  mysql_data:

networks:
  notes_app:
```

**Explanation of new/important parts:**

| Part | Meaning |
|---|---|
| `command: sh -c "python manage.py migrate ... && gunicorn ..."` | Since the Dockerfile has no `CMD`, we define the startup command directly in Compose. First it runs Django's database **migration** (creates the necessary tables/columns in MySQL), then starts the app using **Gunicorn** — a production-grade Python web server (instead of Django's basic development server). |
| `--noinput` | Prevents the migration command from pausing to ask for confirmation — needed since there's no interactive terminal. |
| `env_file: - .env` | Loads all environment variables from the `.env` file automatically, instead of listing each one manually under `environment:`. |
| `nginx` service `depends_on: - django` | Nginx should only start after Django starts (though for full correctness, a health check on Django, shown above, is even better — Nginx will still work, but may show a brief error until Django is ready if no health condition is used). |
| `django` service `depends_on: mysql: condition: service_healthy` | Django will only start after MySQL reports healthy — same concept from Chapter 8. |
| **Networks** | Every single service (`nginx`, `django`, `mysql`) must be attached to the SAME custom network (`notes_app`) for them to resolve each other by container name. |

**Save and exit.**

## Step 8 — Build and Run Everything
```bash
docker compose up -d --build
```
**What this does:** Builds the Django image, builds the Nginx image, pulls the MySQL image, creates the network + volume, and starts all three containers in the correct dependency order.

**Verify:**
```bash
docker ps
```
**Expected result:** Three containers running: `nginx_container`, `notes_container`, `mysql`, all showing `Up` (and `healthy` where health checks are defined).

## Step 9 — Open Port 80 and Access the App
1. Go to your EC2 Security Group → **Inbound rules → Add rule**.
2. Type: **Custom TCP** (or HTTP), Port **80**, Source **Anywhere**.
3. Save.

Open your browser:
```text
http://<YOUR_EC2_PUBLIC_IP>
```
(No port number needed — Nginx listens on the default HTTP port 80 and internally proxies to Django on 8000.)

**Expected result:** The Notes App front-end loads successfully.

## Step 10 — Test the Application End-to-End
1. Sign up for a new account (username, email, password).
2. Log in.
3. Add a note (e.g., category "Entertainment," description of an expense/note).
4. Confirm it saves without error.

## Troubleshooting

### Error 1
```text
host not found in upstream "notes_container" in /etc/nginx/conf.d/default.conf
```
**Reason:** The `proxy_pass` value inside `nginx/default.conf` does not match the actual `container_name` given to the Django service in `docker-compose.yml` (e.g., the config file says `notes_container` but the Compose file names it `django` or something else).

**Fix:** Make sure `nginx/default.conf`'s `proxy_pass` target EXACTLY matches the Django service's `container_name:` value in `docker-compose.yml`.

### Error 2
```text
django.db.utils.OperationalError: (2003, "Can't connect to MySQL server on 'mysql'")
```
even though the `depends_on` with health check is configured.
**Reason (most common cause seen in this project):** The `django` service was NOT actually added to the same custom network (`notes_app`) as the `mysql` service — a very easy mistake to make when writing the Compose file by hand.

**Fix:** Double-check that **every single service** (`nginx`, `django`, `mysql`) has the same `networks: - notes_app` entry. Missing this on even one service breaks the whole chain.

### Error 3
```text
Additional property version is not allowed
```
or similar YAML validation errors.
**Reason:** A typo in the YAML file, often an extra/misplaced key, or incorrect indentation.

**Fix:** Carefully review indentation; YAML is very sensitive to spacing (use 2 spaces per level consistently, never tabs).

### Error 4
```text
mkdir permission denied
```
on the `mysql_data` volume path.
**Reason:** A leftover local folder (accidentally created as a bind-mount path instead of a proper named volume) with restrictive permissions already exists at that location.

**Fix:**
```bash
sudo rm -rf ./mysql_data
docker volume create mysql_data
docker compose up -d --build
```

### Error 5
```text
networks must be a mapping
```
**Reason:** Using a dash (`-`) before network entries where a plain key-value mapping is expected (or vice versa), which is a common YAML syntax confusion between lists and mappings.

**Fix:** For top-level `networks:` declarations, use a plain key with no dash:
```yaml
networks:
  notes_app:
```
Not:
```yaml
networks:
  - notes_app:
```
(The dash form is only correct when referencing networks INSIDE a service's `networks:` list, not when declaring them at the top level.)

## Cleanup
```bash
docker compose down
```
To remove volumes too (deletes data):
```bash
docker compose down -v
```

## Final Result
* A complete 3-tier application (Nginx + Django + MySQL) runs with a single `docker compose up -d --build` command.
* Nginx reverse-proxies all traffic on port 80 to the Django backend running internally on port 8000.
* Django successfully connects to MySQL using Docker's internal DNS-based container-name resolution.
* Data persists in the `mysql_data` named volume.
* You can sign up, log in, and create notes in the browser, with data confirmed to be saved in the database.

## What's Next
Chapter 13 is a second full project: a 3-tier Java/Spring Boot "Expense Tracker" application with Maven multi-stage builds, MySQL, and full Docker Compose automation.
