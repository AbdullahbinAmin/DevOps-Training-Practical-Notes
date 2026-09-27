# Practical 21 — DaemonSets

## Objective
Understand what a DaemonSet is, why it's different from a ReplicaSet/Deployment, and create one to see it automatically place one Pod on every Worker Node.

## Concept: What is a DaemonSet?

Think of a **"langar" or community meal event** — everyone present gets a meal, no exceptions. A DaemonSet works on the same idea:

> **A DaemonSet ensures that at least one copy of a Pod runs on EVERY node in the cluster.**

### Why This Matters — The Problem With Regular Deployments/ReplicaSets

When you created replicas using a Deployment or ReplicaSet earlier, the Scheduler placed those Pods on whichever Worker Nodes it decided — not necessarily one on every node.

### Example (from earlier chapters)
If you have 3 Worker Nodes (Worker-1, Worker-2, Worker-3) and you create a Deployment with 2 replicas:
```bash
kubectl get pods -n nginx -o wide
```
You might see:
- One Pod landed on Worker-1
- One Pod landed on Worker-2
- **Worker-3 got nothing at all**

This happens because Deployments/ReplicaSets only care about the **total replica count**, not about spreading Pods evenly across every node.

### The DaemonSet Solution
A **DaemonSet** guarantees that **every single node** gets at least one copy of the Pod — no node is left out. This is commonly used for things like:
- Log collection agents
- Monitoring agents
- Network plugins

that need to run on every node in the cluster.

## Prerequisites
- KIND cluster with 3 worker nodes (from Chapter 11)
- `nginx` namespace present

## Step 1 — Clean Up Previous Resources

```bash
kubectl delete -f replicaset.yaml
```
(If not already deleted from Chapter 20.)

## Step 2 — Create the DaemonSet Manifest

The easiest way is to copy your existing ReplicaSet file, since the structure is nearly identical.

```bash
cp replicaset.yaml daemonset.yaml
vim daemonset.yaml
```

Update the content to look like this:

```yaml
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: nginx-daemonset
  namespace: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      name: nginx-ds-pod
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

**What changed from the ReplicaSet YAML:**
- `kind: ReplicaSet` → changed to `kind: DaemonSet`
- `metadata.name` → changed to `nginx-daemonset`
- ⚠️ **`spec.replicas` field is REMOVED entirely.** A DaemonSet does NOT use a `replicas` count — it automatically creates exactly one Pod per available node. You never specify a number.
- Everything else (`selector`, `template`, container spec) stays the same.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 3 — Apply the DaemonSet

```bash
kubectl apply -f daemonset.yaml
```

**Expected output:**
```text
daemonset.apps/nginx-daemonset created
```

## Step 4 — Verify

```bash
kubectl get pods -n nginx
```

**Expected output:** Since our KIND cluster has 3 worker nodes, exactly **3 Pods** should be created — you did NOT specify any replica count anywhere:

```text
NAME                    READY   STATUS    RESTARTS   AGE
nginx-daemonset-aaaaa   1/1     Running   0          20s
nginx-daemonset-bbbbb   1/1     Running   0          20s
nginx-daemonset-ccccc   1/1     Running   0          20s
```

**Confirm one Pod is on each node:**
```bash
kubectl get pods -n nginx -o wide
```
**Expected output:** Each Pod should be listed on a **different** worker node:
```text
NAME                    NODE
nginx-daemonset-aaaaa   worker-1
nginx-daemonset-bbbbb   worker-2
nginx-daemonset-ccccc   worker-3
```

## Key Concept Recap: How to Remember Each Controller's Purpose

| Controller | Core Job |
|---|---|
| **ReplicaSet** | Creates exactly as many replicas as you tell it to (a fixed number) |
| **StatefulSet** | Creates replicas AND maintains a stable, numbered state/identity for each one |
| **Deployment** | Creates replicas AND handles rolling updates, scaling, auto-healing |
| **DaemonSet** | Ensures at least one Pod runs on EVERY node — no number needed |

## Clean Up

```bash
kubectl delete -f daemonset.yaml
```

## Final Result
Students now understand:
- Why regular Deployments/ReplicaSets don't guarantee Pod placement on every node
- What a DaemonSet is and its real-world use case (monitoring/logging agents on every node)
- How to write a DaemonSet YAML (very similar to ReplicaSet, minus the `replicas` field)
- How to verify that exactly one Pod landed on each Worker Node

---
**Prerequisite for Chapter 22:** None extra — Chapter 22 (Jobs) introduces a completely different concept: running a one-time task instead of a long-running service.
