# Project Guide: Deploying a 3-Tier Application with Docker Compose

## Project Overview

In this hands-on lab, students will containerize and orchestrate a complete 3-Tier Web Application using Docker Compose. This setup mirrors standard cloud-native application deployments by decoupling the presentation, logic, and persistence layers into separate, isolated containers.

## Application Architecture & Data Flow

```
[ Browser (User) ]
       │ HTTP (Port 80)
       ▼
[ Tier 1: Frontend Container ]  ── (React + Nginx)
       │ HTTP (Internal REST API Call)
       ▼
[ Tier 2: Backend Container ]   ── (Node.js + Express)
       │ SQL Connection (Port 5432)
       ▼
[ Tier 3: Database Container ]  ── (PostgreSQL 15)
```

## Component Breakdown

### Tier 1 — Presentation (Frontend)
- **Tech Stack:** React, Tailwind CSS, Nginx web server.
- **Role:** Serves static assets to the user's browser and issues API requests (fetch/AJAX) to the backend service over HTTP.

### Tier 2 — Application Logic (Backend)
- **Tech Stack:** Node.js, Express.js.
- **Role:** Exposes RESTful API endpoints, processes incoming requests, enforces business rules, and interacts with the database layer.

### Tier 3 — Data Persistence (Database)
- **Tech Stack:** PostgreSQL 15 (Official Image).
- **Role:** Stores data (user records, product catalogs, app state) persistently.

## Inter-Container Communication

All three containers attach to a single user-defined Docker Bridge Network.

Docker's internal DNS allows containers to discover each other using service names (e.g., `http://backend:5000` or `db:5432`) rather than hardcoded container IP addresses.

## Prerequisites

Before beginning the deployment:

- **Operating System:** Ubuntu 24.04 LTS (AWS EC2 instance or local virtual machine).
- **Security Group / Firewall Settings:** Ensure inbound access is allowed for:
  - Port 22 (SSH) — for terminal management.
  - Port 80 (HTTP) — to access the React web app.
- **User Privileges:** sudo access on the target machine.

## Step-by-Step Execution Guide

### 1. Connect to the Server and Update Packages

*Prerequisites setup.*

Connect to your AWS EC2 instance or Ubuntu server via SSH and ensure all system packages are up to date:

```bash
ssh -i "your-key.pem" ubuntu@your-server-public-ip
sudo apt update && sudo apt upgrade -y
sudo apt install git curl -y
```

### 2. Install Docker and Docker Compose

*Core container engine installation.*

Install Docker Engine and the Docker Compose plugin on Ubuntu 24.04:

```bash
# Install Docker
sudo apt install docker.io -y

# Add current user to the Docker group (eliminates requiring 'sudo' for Docker commands)
sudo usermod -aG docker $USER

# Install Docker Compose Plugin
sudo apt install docker-compose-v2 -y

# Verify installations
docker --version
docker compose version
```

> **Note:** If you just added your user to the docker group, log out and log back in, or run `newgrp docker` to apply the group membership.

### 3. Clone the Application Repository

*Target branch: `react-tailwind-website`.*

Create a project folder and clone the specific branch containing the 3-tier application code:

```bash
mkdir -p ~/projects
cd ~/projects

# Clone the specific branch using HTTPS
git clone -b react-tailwind-website https://github.com/bhavukm/3tier-react-tailwind.git

cd 3tier-react-tailwind
```

### 4. Inspect the Project Structure and Docker Compose File

*Review service definitions and environment bindings.*

Inspect the directory layout and view `docker-compose.yml` to understand how the services link together:

```bash
ls -la
cat docker-compose.yml
```

Key parameters to look for inside `docker-compose.yml`:

- **frontend service:** Maps host port `80:80` to route web traffic to Nginx.
- **backend service:** Exposes port `5000` (or similar internal API port) and defines connection parameters for PostgreSQL.
- **db service:** Defines `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` environment variables alongside volume mounts for data persistence.
- **networks block:** Connects all three containers to a shared bridge network.

### 5. Build and Deploy the 3-Tier Stack

*Run containers in detached mode.*

Execute Docker Compose to pull base images, build custom frontend/backend containers, and start all services in the background:

```bash
docker compose up -d --build
```

The `--build` flag forces Docker to build images locally from the Dockerfiles before starting containers.

### 6. Verify Running Containers and Network Status

*Validation phase.*

Check the operational state of all three containers:

```bash
# View stack status via compose
docker compose ps

# List all running containers
docker ps

# Inspect container logs if troubleshooting is needed
docker compose logs -f
```

Ensure all three services (frontend, backend, db) show a status of `Up` / `Running`.

### 7. Access the Application in Your Browser

*Testing Tier 1 through Tier 3 data flow.*

Open your browser and navigate to the public IP address of your EC2 instance:

```
http://YOUR_SERVER_PUBLIC_IP:80
```

**Frontend Verification:** The React application interface will load.

**End-to-End Verification:** Perform an action on the UI (e.g., submitting a form or adding a record). This confirms that:

1. React issued an HTTP request to the backend.
2. Express processed the request and wrote/read data from PostgreSQL.
3. PostgreSQL returned the dataset back to the UI.

### 8. Tear Down and Resource Cleanup

*Post-lab reset.*

Once testing and verification are complete, stop and remove all containers, networks, and persistent volumes created by the stack:

```bash
# Remove containers, networks, images, and named volumes
docker compose down --rmi all -v
```

## Student Challenge & Practice Exercises

To deepen student understanding of multi-container management, assign the following extension tasks:

1. **Network Inspection:** Run `docker network ls` and `docker network inspect <network_name>` to view the assigned internal IP addresses of each container.
2. **Database Verification:** Execute `docker exec -it <db_container_id> psql -U <user> -d <dbname>` to directly query tables created by the backend.
3. **Persistency Check:** Restart the db container (`docker compose restart db`) and confirm that user-generated data remains available across restarts.
