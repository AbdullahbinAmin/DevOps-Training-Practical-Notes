# Practical 40 — Node Affinity

## Objective
Understand Node Affinity as a way to tell a Pod which specific Node(s) it should prefer or require for scheduling, and see how it compares to Taints and Tolerations.

## Concept: What is Node Affinity?

> **Affinity** means being drawn toward, or attracted to, something. If you're very thirsty, you'll have a strong "affinity" toward water — you'll actively seek it out based on its defining characteristic (`water = true`, so to speak).

**Node Affinity** applies this same idea to Pod scheduling:

> **Node Affinity lets you specify that a Pod should be scheduled only on Nodes that match certain labels/characteristics** (e.g., `disktype=ssd`, `zone=us-east-1a`, or any custom label you've applied to your Nodes).

### Node Affinity vs Node Selector
Node Affinity is a more powerful, flexible version of the older/simpler `nodeSelector` field — both achieve a similar goal (targeting specific Nodes), but Node Affinity supports more complex matching rules (like "prefer this, but don't strictly require it").

### Node Affinity vs Taints/Tolerations — Key Difference

| Concept | Direction of Control | What It Says |
|---|---|---|
| **Taint** (on a Node) + **Toleration** (on a Pod) | Node-driven exclusion | "This Node REJECTS Pods, unless they explicitly tolerate it." |
| **Node Affinity** (on a Pod) | Pod-driven preference | "This Pod WANTS to go specifically to a Node with these characteristics." |

> Think of it this way: **Taints/Tolerations** control which Pods are **allowed** onto a Node (a Node-side restriction). **Node Affinity** controls which Node a Pod actively **wants** to go to (a Pod-side preference/requirement).

## Prerequisites
- KIND cluster with multiple worker nodes
- Familiarity with Taints/Tolerations (Chapter 37) is helpful for contrast

## Step 1 — Label a Node

Before using Node Affinity, the target Node needs a distinguishing label.

```bash
kubectl get nodes
```

```bash
kubectl label node <worker-node-name> zone=production
```

**What this does:** Attaches a custom label (`zone=production`) to a specific Node, which we can then reference in a Pod's Node Affinity rule.

**Expected output:**
```text
node/<worker-node-name> labeled
```

**Verify:**
```bash
kubectl get nodes --show-labels
```
**Expected output:** Your target node should now show `zone=production` among its labels.

## Step 2 — Write a Pod With Node Affinity

```bash
vim pod-affinity.yaml
```

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: nginx-affinity-pod
  namespace: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: zone
                operator: In
                values:
                  - production
  containers:
    - name: nginx
      image: nginx:latest
```

**Explanation of every field:**
- `spec.affinity.nodeAffinity` → this block defines the Node Affinity rule.
- `requiredDuringSchedulingIgnoredDuringExecution` → a **required (hard)** rule, meaning the Pod will ONLY be scheduled on a Node matching this rule — if no match is found, the Pod stays `Pending`.
  - (There is also a **softer** version: `preferredDuringSchedulingIgnoredDuringExecution`, which tells the Scheduler to *try* to match this, but schedule elsewhere if no match is found, rather than leaving the Pod stuck.)
  - "IgnoredDuringExecution" means: once the Pod is already running, if the Node's label later changes (no longer matches), the Pod will NOT be evicted — the rule only applies at scheduling time.
- `nodeSelectorTerms` → a list of matching conditions.
  - `matchExpressions` → conditions the Node's labels must satisfy.
    - `key: zone` → the label key to check.
    - `operator: In` → the Node's label value for `zone` must be **in** the list of values given below. (Other operators include `NotIn`, `Exists`, `DoesNotExist`.)
    - `values: [production]` → the only acceptable value is `production`.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 3 — Apply and Verify

```bash
kubectl apply -f pod-affinity.yaml
```

```bash
kubectl get pods -n nginx -o wide
```
**Expected output:** The Pod should be `Running`, and the `NODE` column should show exactly the node you labeled in Step 1.

## Step 4 — Test the Negative Case (No Matching Node)

Delete the Pod:
```bash
kubectl delete pod nginx-affinity-pod -n nginx
```

Remove the label from the Node:
```bash
kubectl label node <worker-node-name> zone-
```
**What this does:** The trailing `-` removes the `zone` label from that Node (same pattern as removing a taint in Chapter 37).

Reapply the Pod:
```bash
kubectl apply -f pod-affinity.yaml
```

```bash
kubectl get pods -n nginx
```
**Expected output:** The Pod stays `Pending`, since no Node now has the label `zone=production`.

**Debug:**
```bash
kubectl describe pod nginx-affinity-pod -n nginx
```
**Expected output:**
```text
Warning  FailedScheduling  ...  0/4 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.
```

**Fix:** Re-label a node to match, then the Pod should schedule successfully:
```bash
kubectl label node <worker-node-name> zone=production
```

## Key Concept Recap

> - **Taints (on Nodes) + Tolerations (on Pods)** → a Node-side mechanism: "keep Pods OUT unless they explicitly tolerate me."
> - **Node Affinity (on Pods)** → a Pod-side mechanism: "I specifically WANT to go to a Node matching these labels."
> Both are used together in real-world clusters to achieve precise, intentional Pod placement — for example, tainting GPU nodes so only GPU-requiring workloads (which tolerate the taint AND have matching Node Affinity) get scheduled there.

## Final Result
Students now understand:
- What Node Affinity is and how it differs conceptually from Taints/Tolerations
- How to label a Node and reference that label in a Pod's `nodeAffinity` rule
- The difference between `requiredDuringSchedulingIgnoredDuringExecution` (hard rule) and `preferredDuringSchedulingIgnoredDuringExecution` (soft rule)
- How to debug a Pod stuck in `Pending` due to unmatched Node Affinity

---
**Note:** This concludes the "Scaling and Scheduling" unit of the course (Resource Limits, Probes, Taints/Tolerations, HPA, VPA, Node Affinity). The next unit moving forward covers **Cluster Administration** topics.

**Chapters still pending (to be added later, as per your instructions):** Cluster Administration, RBAC (Role-Based Access Control), Helm, Monitoring (Prometheus/Grafana), CI/CD (Jenkins), GitOps (ArgoCD), and the final 3-tier mega project on AWS EKS.
