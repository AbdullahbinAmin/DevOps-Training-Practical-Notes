# Practical 37 — Taints and Tolerations

## Objective
Understand how to prevent Pods from being scheduled on specific Nodes (Taints), and how to allow specific Pods to bypass that restriction (Tolerations).

## Concept: What Are Taints and Tolerations?

### Recap: How Scheduling Normally Works
Recall from Chapter 6: when you run `kubectl apply`, the API Server tells the **Scheduler** to decide which Worker Node a new Pod should run on.

### Taint — "Don't Schedule Here"
> A **Taint** is a way of telling your Kubernetes cluster: "Do NOT schedule any Pods on this particular Node."

### Analogy
Imagine a relative who constantly makes rude, mean comments (taunts/"taunt" — sounds like "taint"). Because of this, you might decide not to visit their house anymore. Similarly, if you "taint" a Node, the Scheduler will **avoid** placing Pods on it.

### Toleration — "I'm OK With That Taint"
> But sometimes, even if a relative taunts you, you still go visit them anyway — you **tolerate** it. Similarly, a **Toleration** is added to a specific Pod to say: "I am OK with this Node's taint — go ahead and schedule me there anyway."

## Prerequisites
- KIND cluster with multiple Worker Nodes (from Chapter 11)

## Step 1 — Check Current Nodes

```bash
kubectl get nodes
```
**Expected output:** Your control-plane node plus multiple worker nodes (e.g., `worker`, `worker2`, `worker3`).

## Step 2 — Create a Test Pod (Before Tainting)

```bash
mkdir nginx
cd nginx
vim pod.yaml
```

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: nginx-pod
  namespace: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

```bash
kubectl apply -f pod.yaml
```
**Expected output:** Pod schedules normally onto any available worker node.

Clean it up for this demo:
```bash
kubectl delete pod nginx-pod -n nginx
```

## Step 3 — Taint All the Worker Nodes

```bash
kubectl taint node <worker-node-1-name> prod=true:NoSchedule
kubectl taint node <worker-node-2-name> prod=true:NoSchedule
kubectl taint node <worker-node-3-name> prod=true:NoSchedule
```

**What this does:**
- `kubectl taint node <NODE_NAME>` → applies a taint to the specified node.
- `prod=true` → the taint's key-value pair (you can name this anything — here, saying "this is a production node").
- `:NoSchedule` → the **effect** of this taint. `NoSchedule` means Pods without a matching Toleration will NOT be scheduled here at all. (Other effects exist, like `PreferNoSchedule` — a softer version — and `NoExecute`, which also evicts already-running Pods.)

**Expected output:**
```text
node/<node-name> tainted
```

## Step 4 — Try Scheduling a Pod (It Will Fail)

```bash
kubectl apply -f pod.yaml
```

```bash
kubectl get pods -n nginx
```
**Expected output:** The Pod status stays `Pending`.

**Debug why:**
```bash
kubectl describe pod nginx-pod -n nginx
```
**Expected output:**
```text
Warning  FailedScheduling  ...  0/4 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 3 node(s) had untolerated taint {prod: true}.
```

**What this means:**
- The control-plane node is **always** tainted by default (Pods never get scheduled there under normal circumstances — this is a built-in Kubernetes safety behavior).
- All 3 worker nodes now have our new `prod=true:NoSchedule` taint, which this Pod does not tolerate.
- Result: there is nowhere left for this Pod to be scheduled.

## Step 5 — Remove the Taint From One Node (Quick Fix)

```bash
kubectl taint node <worker-node-2-name> prod=true:NoSchedule-
```

**What this does:** The trailing `-` (minus sign) after the taint removes it from that node.

**Expected output:**
```text
node/<worker-node-2-name> untainted
```

**Verify the Pod now schedules:**
```bash
kubectl get pods -n nginx
```
**Expected output:** The Pod should now be `Running`, scheduled onto the un-tainted worker node.

⚠️ **Note:** Once a Pod is successfully scheduled onto an un-tainted node, even if you re-taint that node afterward, the **already-running Pod stays put** (with the `NoSchedule` effect) — the taint only prevents *new* Pods from being scheduled there, it doesn't evict existing ones.

## Step 6 — Add a Toleration Instead (Proper Method)

Re-taint the node you removed the taint from, so all 3 workers are tainted again:
```bash
kubectl taint node <worker-node-2-name> prod=true:NoSchedule
```

Delete the test Pod:
```bash
kubectl delete pod nginx-pod -n nginx
```

Now edit the Pod manifest to add a **Toleration**:
```bash
vim pod.yaml
```

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: nginx-pod
  namespace: nginx
spec:
  tolerations:
    - key: "prod"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx:latest
```

**Explanation of the new field:**
- `spec.tolerations` → a list of taints this Pod is willing to tolerate.
  - `key: "prod"` → matches the taint's key set in Step 3.
  - `operator: "Equal"` → this toleration matches when the taint's value is **equal** to the value specified below.
  - `value: "true"` → matches the taint's value.
  - `effect: "NoSchedule"` → matches the taint's effect.

## Step 7 — Apply and Verify

```bash
kubectl apply -f pod.yaml
```

```bash
kubectl get pods -n nginx
```
**Expected output:** The Pod should now schedule successfully onto ANY of the tainted worker nodes, because it explicitly tolerates that specific taint.

## Key Concept Recap
> **Taint a Node** → "Don't schedule Pods here" (`kubectl taint node <name> key=value:effect`).
> **Add a Toleration to a Pod** → "I'm OK scheduling on a Node with this specific taint" (`spec.tolerations` in the Pod's manifest).
> Together, Taints and Tolerations let you reserve specific Nodes for specific workloads (e.g., production-only Nodes, GPU Nodes, etc.), while still allowing explicitly-approved Pods to use them.

## Final Result
Students now understand:
- How to apply a taint to a Node using `kubectl taint node`
- How to remove a taint using the trailing `-` syntax
- How to write a Toleration in a Pod's spec to bypass a specific taint
- How Taints and Tolerations work together to control Pod placement at the Node level

---
**Prerequisite for Chapter 38:** None extra — Chapter 38 (HPA) covers automatic scaling based on resource usage, building on the `resources` concept from Chapter 35.
