# Chapter 7 — Create the Kubernetes Architecture Diagram (Practice Exercise)

## Objective
Give students a step-by-step method to draw the full Kubernetes architecture diagram by themselves — a common interview exercise (whiteboard question).

## Prerequisites
- Chapter 6 fully understood (all component names and roles)

## Step-by-Step: How to Draw the Diagram

### Step 1 — Draw the Cluster Boundary
Draw one big box/boundary. This represents your whole Kubernetes Cluster.

### Step 2 — Add Master Node and Worker Node(s)
Inside the cluster boundary, draw:
- One **Master Node** box
- One or more **Worker Node** box(es)

### Step 3 — Fill in the Master Node
Inside the Master Node box, draw and label these 4 components:
1. **API Server** — the communication hub; everything connects to it
2. **Scheduler** — connects to API Server; decides which worker node runs the pod
3. **Controller Manager** — connects to API Server; manages overall cluster health
4. **etcd** — connects to API Server; stores all cluster data (key-value store)

Draw arrows from the API Server to each of these three (Scheduler, Controller Manager, etcd) since the API Server communicates with all of them.

### Step 4 — Fill in the Worker Node
Inside the Worker Node box, draw and label:
1. **Kubelet** — connects to API Server; makes sure pods are running fine on this node
2. **Pods** — sit inside the worker node; contain the actual Docker containers
3. **Service Proxy (kube-proxy)** — connects to API Server; gives outside users access to Pods

### Step 5 — Add the CNI Network
Draw a network layer connecting Master Node and Worker Node(s) together. Label it **CNI (Container Network Interface)** — example tools: Calico, Weave Net.

### Step 6 — Add kubectl
Draw kubectl outside the cluster boundary, with an arrow pointing to the API Server. Label it as the tool used to give commands/instructions to the whole cluster.

## Final Diagram Structure (Text Summary)

```text
                         kubectl
                            |
                            v
                      [ API Server ]  <-- Master Node
                       /    |    \
              [Scheduler] [etcd] [Controller Manager]
                            |
                    (CNI Network - e.g. Calico)
                            |
                     Worker Node
               ---------------------------
              [ Kubelet ] -- [ Pods (Containers) ] -- [ Service Proxy / kube-proxy ]
```

## Final Result
Students should be able to draw this complete diagram from memory within a few minutes, and explain each component's role out loud — this is a very common Kubernetes interview question.

---
**Prerequisite for Chapter 8:** None — this was a recap/practice exercise. Chapter 8 moves into actual hands-on setup on AWS.
