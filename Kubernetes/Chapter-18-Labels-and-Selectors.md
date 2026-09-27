# Practical 18 — Labels and Selectors (with Deployment Hands-on)

## Objective
Understand what Labels and Selectors are, why a Deployment needs them, write a complete Deployment YAML file, apply it, and practice scaling the number of replicas up and down.

## Architecture / Flow
```text
Pod gets a Label (name tag) → Deployment's Selector looks for that Label
→ Any Pod matching the label gets managed/replicated by the Deployment
```

## Concept: What Are Labels and Selectors?

### Labels
A **Label** is like a name tag or ID badge you attach to a Pod, so it can be identified. For example, you might label a Pod with `app: nginx` to say "this Pod belongs to the Nginx application."

### Selectors
A **Selector** is how a Deployment says: "Find me any Pod that has this specific label, and manage/replicate it."

### Why This Matters
If you want a Deployment to create 2 replicas of your Nginx Pod, the Deployment needs a way to **recognize** which Pod template to duplicate. This is done by:
1. Giving the Pod a **label** (e.g., `app: nginx`)
2. Telling the Deployment's **selector** to match that same label (e.g., `matchLabels: app: nginx`)

If the label on the Pod and the selector's expected label match, the Deployment will create replicas of that Pod.

## Prerequisites
- KIND cluster running
- `nginx` namespace created (from Chapter 14)
- Working folder: `kubernetes-in-one-shot/nginx/`

## Step 1 — Write the Deployment Manifest

```bash
cd ~/kubernetes-in-one-shot/nginx
vim deployment.yaml
```

Paste this complete YAML:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nginx-deployment
  namespace: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      name: nginx-dep-pod
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

**Explanation of every field:**
- `kind: Deployment` → this manifest describes a Deployment.
- `apiVersion: apps/v1` → Deployments use the `apps/v1` API group (different from Pods, which use `v1`).
- `metadata.name` → the name of this Deployment (`nginx-deployment`).
- `metadata.namespace` → which namespace it belongs to (`nginx`).
- `spec.replicas: 2` → you want **2 identical copies** of this Pod running at all times.
- `spec.selector.matchLabels` → tells the Deployment: "find and manage any Pod with the label `app: nginx`."
- `spec.template` → this is the **Pod template** — basically a blueprint of what each replica Pod should look like. This is the same structure as a normal Pod manifest (from Chapter 15), just nested inside the Deployment.
  - `template.metadata.name` → a name for the Pod template (e.g., `nginx-dep-pod`).
  - `template.metadata.labels.app: nginx` → **this is the actual label being applied to each Pod** — it MUST match the `selector.matchLabels` value above, otherwise the Deployment won't recognize its own Pods.
  - `template.spec.containers` → same container specification structure as a normal Pod (name, image, port).

⚠️ **Important:** The value under `selector.matchLabels` and the value under `template.metadata.labels` must be **exactly the same**. This is what links the Deployment to the Pods it creates.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 2 — Apply the Deployment

```bash
kubectl apply -f deployment.yaml
```

### If Something Goes Wrong
If you see an error like:
```text
error: error validating "deployment.yaml": error validating data: ValidationError(Deployment.spec): unknown field "replica" in io.k8s.api.apps.v1.DeploymentSpec
```

**Reason:**
A common typing mistake — the correct field name is `replicas` (plural), not `replica`.

**Fix:** Correct the spelling in your YAML file and reapply:
```bash
kubectl apply -f deployment.yaml
```

**Expected output:**
```text
deployment.apps/nginx-deployment created
```

## Step 3 — Verify the Deployment

```bash
kubectl get pods -n nginx
```
**Expected output:** 2 Pods should be listed, both `Running`, since we set `replicas: 2`:
```text
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-xxxxxxxxx-aaaaa    1/1     Running   0          30s
nginx-deployment-xxxxxxxxx-bbbbb    1/1     Running   0          30s
```

**Note:** Since a Pod was already created directly in Chapter 15 (`nginx-pod`), you can clean that one up if it's still running:
```bash
kubectl delete pod nginx-pod -n nginx
```

## Step 4 — Scale the Deployment Up

Let's simulate heavy traffic and scale up to 5 replicas:

```bash
kubectl scale deployment nginx-deployment --replicas=5 -n nginx
```

**What this does:**
- `scale` → command to change the replica count of a resource
- `deployment nginx-deployment` → the specific Deployment to scale
- `--replicas=5` → the new desired replica count
- `-n nginx` → the namespace it's in

**Expected output:**
```text
deployment.apps/nginx-deployment scaled
```

**Verify:**
```bash
kubectl get pods -n nginx
```
**Expected output:** 5 Pods should now be `Running`.

## Step 5 — Scale the Deployment Down

Now let's say traffic dropped, and we only need 1 replica:

```bash
kubectl scale deployment nginx-deployment --replicas=1 -n nginx
```

**Verify:**
```bash
kubectl get pods -n nginx
```
**Expected output:** Only 1 Pod should be `Running`; the extra Pods will be automatically terminated.

## Step 6 — Check the ReplicaSet Behind the Deployment

Remember: a Deployment internally uses a **ReplicaSet** to manage its Pods.

```bash
kubectl get replicaset -n nginx
```
or short form:
```bash
kubectl get rs -n nginx
```
**Expected output:** You'll see a ReplicaSet automatically created by your Deployment, showing the desired/current/ready Pod counts.

**Try a bigger scale test (optional, for fun/demo):**
```bash
kubectl scale deployment nginx-deployment --replicas=10 -n nginx
kubectl get pods -n nginx
```
**Expected output:** 10 Pods, all `Running`, created almost instantly.

Then scale back down to 1:
```bash
kubectl scale deployment nginx-deployment --replicas=1 -n nginx
```

## Troubleshooting

### Error: Pods stuck in `Pending` after scaling up
**Reason:** Not enough CPU/memory resources on your worker nodes to schedule all the requested replicas.
**Fix:** Scale down to a lower number, or use a bigger instance type.

### Error: `unknown field "replica"` (singular) instead of `replicas`
**Reason:** Common typo in the YAML file.
**Fix:** Always double-check exact field names against the official Kubernetes documentation when unsure.

## Final Result
Students now understand:
- What Labels and Selectors are, and why they connect a Deployment to its Pods
- How to write a full Deployment YAML manifest
- How to apply a Deployment and verify its Pods
- How to scale a Deployment up and down using `kubectl scale`
- That a Deployment automatically creates and manages a ReplicaSet behind the scenes

---
**Prerequisite for Chapter 19:** This same `nginx-deployment` (currently scaled to 1 replica) will be reused in Chapter 19 to demonstrate Rolling Updates.
