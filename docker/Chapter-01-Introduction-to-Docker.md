# Chapter 1 — Introduction to Docker

## Objective
Understand what Docker is, why it is used in the industry, and the problem it solves (the classic "it works on my machine" problem).

## Prerequisites
* OS: Any (this is a theory/introduction chapter, no hands-on commands yet)
* RAM: N/A
* CPU: N/A
* Required software: None yet
* Required account: None yet
* Required repository: None
* Required ports: None
* Required permissions: None
* Required files: None

## Pre-Check
No commands to run in this chapter. This is a concept-building chapter before we touch the terminal.

## Concept 1 — The "Works on My Machine" Problem

**What it is:**
Imagine you are a developer (freelancer, employee, or student). You built a website on your laptop and it works perfectly. You deliver it to a client. The client tries to run it on their own laptop, and it does not work.

**Why this happens:**
Your machine has certain libraries, certain versions of software, and a certain operating system installed. The client's machine may have different versions, missing libraries, or a completely different OS (for example, you developed on Windows, but the client uses macOS). Because the environments are different, the same code behaves differently or fails to run.

This is called the **"It works on my machine" problem**, and it is extremely common in the IT industry.

## Concept 2 — What Docker Does

**What it is:**
Docker is a tool that packages your application together with everything it needs (libraries, dependencies, configuration, environment) into a single unit called a **container**.

**Why this matters:**
Instead of shipping just your code, you ship the code AND the exact environment it needs to run, all bundled inside a container. You then package this container as an **image** and give that image to the client (or any other computer). When the client runs that image, the application behaves exactly the same as it did on your machine — because the environment travels with it.

**In short:**
* Docker packages an application + its environment into a **container**.
* Containers are built from **images** (like a blueprint).
* You can run that same image on any computer and get identical behavior.

## Concept 3 — History of Docker

* Docker originated at a company called **dotCloud**, which faced this exact "works on my machine" problem internally.
* They solved it by containerizing their applications — this is where Docker was born.
* Docker was made publicly available in **2013** at a conference called **PyCon**.
* In **2017**, Docker was contributed to the **Cloud Native Computing Foundation (CNCF)** — the foundation that manages many popular open-source cloud-native tools.
* Today, almost every organization uses Docker containers in their technology stack.

## Concept 4 — Why We Use Docker (Virtual Environments)

**The problem:**
Sometimes an application only works on your computer because of specific library versions, specific tools, or specific configurations present only on that machine. Another computer without those exact items will not be able to run the same application.

**The solution:**
You create a **virtual/isolated environment** that contains everything the application needs. This environment can then run identically on any other computer.

## Concept 5 — Docker vs Virtual Machines (Virtualization vs Containerization)

This is one of the most commonly asked interview questions, so understand it clearly.

| Concept | Virtualization | Containerization |
|---|---|---|
| Unit used | Virtual Machines (VMs) | Containers |
| Tools | VMware, VirtualBox, Hyper-V | Docker, Podman, containerd |
| Uses own OS? | Yes, each VM has its own full OS | No, containers share the host OS |
| Resource usage | Heavy (each VM needs allocated RAM, CPU, disk) | Lightweight (containers share host resources) |
| How many can you run? | Very few on a normal machine (limited by RAM) | Many containers can run at once |

**How Virtual Machines work:**
A **hypervisor** sits on top of the physical hardware/OS. The hypervisor allocates a fixed chunk of resources (RAM, CPU, disk) to each Virtual Machine, and each VM runs its own complete Operating System (Windows, macOS, Linux, etc.) on top of that allocation.

**How Containers work:**
The **Docker Engine** does NOT have its own separate operating system. Instead, it uses (shares) the **host machine's operating system**. Because containers do not need to boot a full OS and do not need dedicated resource allocation, they are much **lighter** than VMs.

**Practical example given:**
* With 8 GB of RAM, you can typically run only 1–2 Virtual Machines comfortably before running out of resources.
* With the same 8 GB of RAM, you can run **many** Docker containers, because they are lightweight and share the host OS kernel.

**Other container tools (besides Docker):**
* **Podman** — an alternative to Docker for building/running containers.
* **containerd** — another container runtime, often used as Docker's backend.

## Diagram — Conceptual Difference

```text
VIRTUALIZATION:
[ Host Operating System ]
        │
   [ Hypervisor ]
   /      |      \
[VM1]   [VM2]   [VM3]
(own OS)(own OS)(own OS)
= Heavy, resources fully allocated per VM

CONTAINERIZATION:
[ Host Operating System ]
        │
  [ Docker Engine ]
   /      |      \
[C1]    [C2]    [C3]
(shares host OS kernel, shares resources)
= Lightweight, many containers possible
```

## Key Takeaways (Summary)
1. Docker solves the "works on my machine, not on yours" problem by packaging code + environment together.
2. A **Dockerfile** → builds a **Docker Image** → an Image is run to create a **Docker Container** (this exact chain will be explained fully in later chapters).
3. Docker uses **containerization**, not full virtualization — containers share the host OS kernel, making them lightweight compared to Virtual Machines.
4. Docker was released in 2013 and joined CNCF in 2017; it is an industry-standard tool used by almost every company today.
5. Next chapter covers the **Docker Architecture** (Docker Engine, Docker Daemon, Docker CLI) — how Docker actually works internally.

## What's Next
Chapter 2 will explain the Docker Architecture: Docker Engine, Docker Daemon (dockerd), containerd, and Docker CLI — and how they all communicate with each other.
