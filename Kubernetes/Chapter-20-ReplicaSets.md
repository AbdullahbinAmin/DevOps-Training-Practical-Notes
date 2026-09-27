# Practical 20 — ReplicaSets (Hands-on)

## Objective
Create a ReplicaSet manifest by converting the existing Deployment YAML, apply it, and understand practically what a ReplicaSet can and cannot do compared to a Deployment.

## Prerequisites
- `deployment.yaml` from Chapter 18 already exists
- KIND cluster running, `nginx` namespace present

## Step 1 — Clean Up the Deployment First

```bash
kubectl delete -f deployment.yaml
```
**What this does:** Removes the Deployment (and its managed ReplicaSet and Pods), so we can now create a plain ReplicaSet instead, without conflicts.

**Expected output:**
```text
deployment.apps "nginx-deployment" deleted
```

## Step 2 — Copy the Deployment File as a Base

```bash
cp deployment.yaml replicaset.yaml
```
**What this does:** Creates a copy of the Deployment YAML file so we can modify it into a ReplicaSet manifest, since the structure is almost identical.

## Step 3 — Edit the New File

```bash
vim replicaset.yaml
```

Update the content to look like this:

```yaml
kind: ReplicaSet
apiVersion: apps/v1
metadata:
  name: nginx-replicaset
  namespace: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      name: nginx-rs-pod
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

**What changed from the Deployment YAML:**
- `kind: Deployment` → changed to `kind: ReplicaSet`
- `metadata.name: nginx-deployment` → changed to `metadata.name: nginx-replicaset`
- Everything else (`apiVersion: apps/v1`, `selector`, `template`, container spec) stays **exactly the same** — this shows how similar the two resource types are structurally.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 4 — Apply the ReplicaSet

```bash
kubectl apply -f replicaset.yaml
```

**Expected output:**
```text
replicaset.apps/nginx-replicaset created
```

## Step 5 — Verify

```bash
kubectl get pods -n nginx
```
**Expected output:** 2 Pods running (since `replicas: 2`), both marked `Ready`.

```bash
kubectl get replicaset -n nginx
```
**Expected output:**
```text
NAME                DESIRED   CURRENT   READY   AGE
nginx-replicaset    2         2         2       30s
```

## Key Difference: What a ReplicaSet CANNOT Do

Try a rolling update the same way we did with the Deployment:
```bash
kubectl set image replicaset/nginx-replicaset nginx=nginx:1.27.3 -n nginx
```

**What happens:** Unlike a Deployment, a ReplicaSet does **not** perform a smooth rolling update. It does not have built-in rollout history or rollback support. You would need to manually delete and recreate Pods to fully apply certain kinds of changes — there is no graceful, zero-downtime rollout mechanism the way Deployments provide.

⚠️ **Note:** This is exactly why, in real-world usage, you almost always use a **Deployment** instead of a raw ReplicaSet — Deployments are more flexible and handle updates safely. A ReplicaSet is mainly used **internally by Deployments** (created automatically, as seen in Chapter 18) rather than being created directly by users.

## Clean Up

```bash
kubectl delete -f replicaset.yaml
```

## Final Result
Students now understand practically:
- A ReplicaSet YAML is almost identical in structure to a Deployment YAML
- A ReplicaSet keeps a fixed number of Pod replicas running, just like a Deployment
- A ReplicaSet does **not** support safe rolling updates — this is the key feature that only Deployments provide
- In real projects, Deployments are preferred over directly creating ReplicaSets

---
**Prerequisite for Chapter 21:** None extra — StatefulSets will be covered later alongside Storage concepts. Chapter 21 (DaemonSets) continues with the same YAML pattern, adapted for a different purpose.
