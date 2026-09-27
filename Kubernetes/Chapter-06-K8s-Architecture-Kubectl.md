# Chapter 6 — Kubernetes Architecture & kubectl

## Objective
Explain the full internal architecture of Kubernetes — Master Node and Worker Node, and every component inside them — using a simple company (MNC) analogy so it stays easy to remember for interviews.

## The Logo Story
Docker's logo is a ship/whale carrying containers. The person who "steers" that ship uses a **steering wheel** — and that steering wheel is exactly what the Kubernetes logo represents. Kubernetes is the "captain" that drives/controls your Docker containers.

## The Company (MNC) Analogy

Think of Kubernetes like a Multinational Company (MNC):

- A single Docker container running alone = a **Startup** (hard to scale, breaks easily, unreliable on its own).
- Kubernetes = a full **MNC (Multinational Company)** with proper structure, so it can scale and stay reliable.

In an MNC:
- The **Head Office** doesn't do the actual hands-on work — it manages and gives instructions.
- The **Branch Offices / Locations** (like Pune, Bangalore, Noida) are where the actual work happens.

In Kubernetes terms:
- **Master Node** = Head Office (management only, no actual application work happens here)
- **Worker Node(s)** = Branch Offices (actual application containers run here)

### Node = Server
Whenever we say "Node" in Kubernetes, it simply means a **Server**.
- **Single Node** = one server
- **Multiple Nodes** = more than one server, and this group of nodes together is called a **Cluster**

In Kubernetes, we always work at the **cluster level** — meaning we manage the whole group of nodes, not just a single server.

## Master Node (Control Plane) — The "Head Office"

The Master Node does NOT run your actual application containers. It only manages and instructs. It contains these key services:

### 1. API Server
- Acts like a **communication gateway** — every other component talks to each other THROUGH the API Server.
- API = Application Programmable Interface — a "medium" you must go through to talk to something.
- You (the user) also cannot talk to the Kubernetes cluster directly — you must go **via the API Server**.
- This is the most important unit for communication within the cluster.

### 2. Scheduler
- Analogy: like an **HR** who assigns a person (container) to the right job (worker node).
- The Scheduler's job is to decide **which Worker Node** a new container/Pod should run on.
- Example: If you need an Nginx container (reverse proxy server) or a MySQL container (database), the Scheduler tells the API Server: "I will schedule this on a particular worker node."
- Scheduler is responsible for scheduling **Pods** (the unit that holds one or more containers) to run on Worker Nodes.

### 3. etcd
- Analogy: like a **database that stores every record**, such as when a Pod was created, what job opening came, etc.
- etcd is a **key-value pair data store**.
- All data about everything running in the Kubernetes cluster is stored in etcd.

### 4. Controller Manager
- Analogy: like a strict **Project Manager** who checks whether everything is running properly everywhere.
- The Controller Manager continuously checks:
  - Are the Nodes running fine?
  - Are the Pods running fine?
  - Is the Cluster healthy overall?
  - Are all services working correctly?
- It is a very important component for keeping the cluster's actual state matching the desired state.

## Worker Node — The "Branch Office"

This is where the **actual work happens** — actual Docker containers run here. Key services on the Worker Node:

### 1. Kubelet
- Analogy: like a local branch manager who checks if you (employee) are working properly, logged in correctly, submitting tasks correctly.
- Kubelet runs at the **Worker Node level** (unlike Controller Manager which works at the whole cluster level).
- Kubelet checks if Pods are running properly on that specific worker node.
- If something is wrong, Kubelet reports it back to the API Server, which then updates the Controller Manager and etcd.

### 2. Pods
- Pods are isolated by default — you cannot access them directly from outside, just like outsiders cannot enter a company office without an ID card.

### 3. Service Proxy (kube-proxy)
- Analogy: like an insider ("Lanka Bhedi") who lets information move between inside and outside.
- Also called **kube-proxy**.
- Because Pods are isolated, if you (as an outside user) want to access what's running inside a Pod, you need this Service Proxy.
- This is why **Services** are very important when building real applications in Kubernetes.

## Networking Between Nodes (CNI)

- All the Nodes (Master and Worker) need to talk to each other over network.
- This communication happens through a **Container Network Interface (CNI)**.
- Popular examples of CNI: **Calico**, **Weave Net (Wnet)**.
- CNI is what allows all worker nodes and the master node to communicate with each other.

## kubectl — The "CEO" / Director

- Every company has a CEO who gives direction/instructions.
- In Kubernetes, that role is played by **kubectl** (also called kube-controller/kube CLI tool).
- kubectl is the tool you use to give commands to your cluster — for example:
  - "Show me how many Pods are running"
  - "Show me how many Nodes are active"
  - "Create this resource"
  - "Run this"
- kubectl works by talking to the **API Server**, which then relays instructions to the rest of the cluster.

## Full Communication Flow (Summary)

```text
User → kubectl → API Server → Scheduler / Controller Manager / etcd
                             → (via CNI network) → Worker Node → Kubelet → Pod (Containers)
                             → Service Proxy (kube-proxy) → allows outside access to Pods
```

## Final Result
Students should be able to draw and explain the full Kubernetes architecture diagram from memory:
- Master Node: API Server, Scheduler, Controller Manager, etcd
- Worker Node: Kubelet, Pods (containers), Service Proxy (kube-proxy)
- CNI network connecting all nodes
- kubectl as the tool to control everything through the API Server

---
**Prerequisite for Chapter 7:** Full understanding of this chapter's architecture, since Chapter 7 is a hands-on "draw it yourself" exercise based on this diagram.
