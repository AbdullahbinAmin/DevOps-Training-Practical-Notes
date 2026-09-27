# Practical 35 — Resource Requests and Limits

## Objective
Understand why every Pod should be given a defined amount of CPU/memory it needs and can use, and add `requests` and `limits` to a container definition.

## Concept: Why Do We Need Requests and Limits?

### The Problem
Imagine your Host Machine (a `t2.medium` EC2 instance) has:
- **4 GB RAM**
- **2 vCPU cores**

Now imagine you run an Nginx Pod on it, and that Pod suddenly receives massive traffic. Without any limits, this **single Pod could consume the entire 4 GB of RAM**, leaving **nothing** for any other Pod (like a MySQL Pod) that needs to run on the same node.

### The Solution
> Just like you might tell your mom "give me only 2 rotis" when eating — a **request** — you also set a maximum: "I won't eat more than this" — a **limit**. Similarly, every container in Kubernetes should be given a **request** (minimum guaranteed resources) and a **limit** (maximum it's allowed to consume).

### Definitions
- **Requests** → the **minimum** amount of CPU/memory a container needs to run properly. Kubernetes uses this value to decide which Node has enough capacity to schedule the Pod.
- **Limits** → the **maximum** amount of CPU/memory a container is allowed to consume. If it tries to exceed this, Kubernetes will restrict or terminate it (depending on the resource type).

## Prerequisites
- An existing Deployment (this practical reuses the `nginx` Deployment)

## Step 1 — Add Resources to the Container Spec

```bash
vim deployment.yaml
```

Add a `resources` block inside the container definition:

```yaml
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
```

**Explanation of every field:**
- `resources.requests.cpu: "100m"` → the minimum CPU this container needs to run. `100m` means "100 millicores," which is **0.1 of a full CPU core**.
- `resources.requests.memory: "128Mi"` → the minimum memory (RAM) needed — `128Mi` means 128 mebibytes.
- `resources.limits.cpu: "200m"` → the maximum CPU this container is allowed to use — 0.2 of a full core.
- `resources.limits.memory: "256Mi"` → the maximum memory this container is allowed to use.

⚠️ **Common Error — `resource` vs `resources`:**
```text
error: error validating "deployment.yaml": ... unknown field "resource" in ...Container
```
**Reason:** The correct field name is `resources` (plural).
**Fix:** Correct the spelling.

⚠️ **Common Error — `requests` vs `request`:**
```text
error: error validating "deployment.yaml": ... unknown field "request" in ...ResourceRequirements
```
**Reason:** The correct field name is `requests` (plural), NOT `request`.
**Fix:** Correct the spelling. Remembering the difference: **Requests** = what a Pod needs to start running. **Limits** = the ceiling it can't exceed while running.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 2 — Apply the Updated Deployment

```bash
kubectl apply -f deployment.yaml
```
**Expected output:**
```text
deployment.apps/nginx-deployment configured
```

## Step 3 — Verify Resources Were Applied

```bash
kubectl get pods -n nginx
```

```bash
kubectl describe pod <pod-name> -n nginx
```
**Expected output:** Under the container's details, you should see:
```text
Limits:
  cpu:     200m
  memory:  256Mi
Requests:
  cpu:     100m
  memory:  128Mi
```

## Step 4 — Apply the Same Concept to a StatefulSet

The exact same `resources` block can be added to any Pod-based workload, including StatefulSets:

```bash
cd ../mysql
vim statefulset.yaml
```

Add the same `resources` block under the MySQL container's spec (same structure as shown in Step 1), then reapply:
```bash
kubectl apply -f statefulset.yaml
```

## Key Concept Recap
> **Requests** and **Limits** ensure that no single Pod can starve other Pods of CPU/memory on the same Node. This keeps your entire cluster stable — instead of one Pod consuming all available resources and crashing the whole Node, each Pod is contained within its defined boundaries. If a workload genuinely needs more capacity than its limit allows, the correct approach is to **scale out** (create more replicas), which then distributes and load-balances the incoming traffic across multiple Pods — rather than letting a single Pod grow unchecked.

## Final Result
Students now understand:
- Why resource requests and limits are essential for cluster stability
- How to correctly add `requests` and `limits` for both CPU and memory to a container spec
- How to verify these settings using `kubectl describe pod`
- That this same pattern applies to any workload type (Deployment, StatefulSet, DaemonSet, etc.)

---
**Prerequisite for Chapter 36:** None extra — Chapter 36 (Probes) covers a related but different concept: checking whether a Pod is actually healthy and ready, not just how many resources it's allowed to use.
