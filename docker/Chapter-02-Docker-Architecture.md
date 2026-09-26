# Chapter 2 — Docker Architecture

## Objective
Understand the internal architecture of Docker: Docker Engine, Docker Daemon (dockerd), containerd, and Docker CLI, and how a command you type actually reaches a running container.

## Prerequisites
* OS: Any (concept chapter; commands used to prove architecture come in Chapter 3 after installation)
* Required software: None yet (Docker will be installed in Chapter 3)
* Required knowledge: Chapter 1 concepts (what Docker is)

## Architecture / Flow
```text
You (typing a command)
   → Docker CLI (docker ...)
      → Docker Daemon (dockerd)
         → containerd
            → creates/manages the actual container
```

## The Three Core Components

### 1. Docker Engine
**What it is:**
Docker Engine is also called the **Docker Application Container Engine**. It is the core software on which all your containers run. In the background, Docker Engine is made up of the Docker Daemon and containerd, working together.

### 2. Docker Daemon (dockerd)
**What it is:**
The Docker Daemon (`dockerd`) is a background service that manages your containers. It listens for Docker API requests and manages Docker objects like images, containers, networks, and volumes.

**How it relates to containerd:**
Inside dockerd, there is another component called **containerd**.

### 3. containerd
**What it is:**
containerd is a separate tool (also a CNCF project) that is responsible for actually creating and managing lightweight, portable containers. It is written in the Go programming language.

**Why this matters:**
Docker uses containerd internally as its container runtime. So the actual work of creating/running/stopping a container is delegated by dockerd to containerd.

### 4. Docker CLI (Command Line Interface)
**What it is:**
This is what you interact with — the `docker` commands you type, such as:
```bash
docker login
docker ps
docker build
docker run
```

**How it works:**
Every command you type through the Docker CLI is sent to the Docker Daemon (dockerd). The Docker Daemon then tells containerd what to do (create a container, start it, stop it, etc.).

### 5. Docker Client / Docker Desktop
**What it is:**
The Docker Client (or Docker Desktop GUI) is another way to see and interact with everything Docker is managing — running containers, images, volumes, etc.

**How it works:**
The Docker Client connects to Docker Engine through an **API** (Application Programmable Interface — a way for the front-end client to communicate with the back-end engine). Whether you use the CLI or Docker Desktop, both are talking to the same Docker Engine underneath.

## Full Request Flow (Example: `docker ps`)
1. You type `docker ps` in the terminal (this goes through the **Docker CLI**).
2. The CLI sends this command to the **Docker Daemon (dockerd)**.
3. dockerd talks to **containerd**, which knows the actual state of all containers.
4. The information (list of running containers) is sent back up and displayed in your terminal.

The same flow applies to Docker Desktop — it talks to the same backend via the API instead of the CLI.

## Diagram (Text Form)
```text
┌─────────────────────────────────────────────┐
│              Docker Engine                    │
│                                                │
│   Docker CLI  ───────┐                        │
│   (docker run, ps..) │                        │
│                       ▼                        │
│              Docker Daemon (dockerd)          │
│                       │                        │
│                       ▼                        │
│                  containerd                    │
│                       │                        │
│                       ▼                        │
│              Actual Containers                │
│                                                │
│   Docker Desktop / Docker Client ── API ──────┤ (talks directly to Engine)
└─────────────────────────────────────────────┘
```

## Key Takeaways (Summary)
1. **Docker Engine** = the overall system (also called Docker Application Container Engine) that runs your containers.
2. **Docker Daemon (dockerd)** = the background service/process that manages containers.
3. **containerd** = the actual component (inside dockerd) that creates and manages the lightweight containers. It is a separate CNCF project written in Go.
4. **Docker CLI** = the command-line tool you use to send commands (`docker run`, `docker ps`, `docker build`, etc.) to the Docker Daemon.
5. **Docker Client / Docker Desktop** = a GUI alternative that talks to the same Docker Engine through an API.
6. Whether you use the CLI or the GUI, the underlying flow is: **Your input → dockerd → containerd → actual container action**.

## What's Next
Chapter 3 will show this architecture live by installing Docker (locally and on an AWS EC2 instance) and running real commands like `docker ps` and checking the `dockerd` service status.

> ⚠️ Note: This chapter is conceptual. All commands referenced (docker ps, systemctl status docker, etc.) are demonstrated hands-on in Chapter 3 (Installing Docker) once Docker is actually installed.
