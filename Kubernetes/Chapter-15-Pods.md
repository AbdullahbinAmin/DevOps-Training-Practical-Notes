# Practical 15 — Pods

## Objective
Learn what a Pod manifest file looks like, create an Nginx Pod inside the `nginx` namespace using YAML, access/enter the running Pod, and learn basic debugging commands (`describe`, `exec`).

## Architecture / Flow
```text
pod.yaml (manifest) → kubectl apply → API Server → Scheduler assigns Pod to a Worker Node
→ Kubelet on that node pulls the image → Container starts running inside the Pod
```

## Prerequisites
- KIND cluster running (from Chapter 11)
- `nginx` namespace already created (from Chapter 14)
- Working folder: `kubernetes-in-one-shot/nginx/` (from Chapter 14)

## Pre-Check
```bash
kubectl get ns
```
**Expected result:** The `nginx` namespace should be listed as `Active`.

## Concept: Finding YAML Fields Without Memorizing Them

You do not need to memorize every YAML field for every Kubernetes resource. A simple trick:
1. Go to Google and search: `kubernetes pod` or `pods kubernetes`.
2. Open the official Kubernetes documentation page for Pods.
3. Copy the example/reference `spec` structure shown there and adjust it for your use case.

**Official Reference:** https://kubernetes.io/docs/concepts/workloads/pods/

## Step 1 — Create the Pod Manifest File

Make sure you are inside your working folder:
```bash
cd ~/kubernetes-in-one-shot/nginx
```

Create the file:
```bash
vim pod.yaml
```

Paste this complete YAML content:
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
      ports:
        - containerPort: 80
```

**Explanation of every field:**
- `kind: Pod` → tells Kubernetes this manifest describes a Pod resource.
- `apiVersion: v1` → the core API version (Pods use `v1`, the base/stable API group).
- `metadata.name: nginx-pod` → the name you're giving this Pod.
- `metadata.namespace: nginx` → this Pod must be created inside the `nginx` namespace (the one from Chapter 14), NOT the `default` namespace.
- `spec` → the actual specification/configuration of the Pod.
- `spec.containers` → a **list** (since a Pod can contain one or more containers) of container definitions.
  - `name: nginx` → the name of this specific container inside the Pod.
  - `image: nginx:latest` → which Docker image to run; `latest` pulls the newest available version from Docker Hub.
  - `ports.containerPort: 80` → the port the Nginx application listens on inside the container (Nginx's default port is 80).

**Save and exit** (`Esc`, `:wq`, Enter).

**Verify the file:**
```bash
cat pod.yaml
```

## Step 2 — Apply the Manifest

```bash
kubectl apply -f pod.yaml
```
**Expected output:**
```text
pod/nginx-pod created
```

## Step 3 — Verify the Pod

⚠️ **Important:** Since the Pod was created inside the `nginx` namespace (not `default`), you MUST specify the namespace when checking it.

```bash
kubectl get pods
```
**Expected output:**
```text
No resources found in default namespace.
```
**Why:** Because kubectl by default looks in the `default` namespace, and our Pod lives in the `nginx` namespace.

Correct command:
```bash
kubectl get pods -n nginx
```
**Expected output:**
```text
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          30s
```

## Step 4 — Enter (Exec Into) the Running Pod

Just like you can go inside a running Docker container with `docker exec`, you can go inside a Pod using `kubectl exec`.

```bash
kubectl exec -it nginx-pod -n nginx -- bash
```

**What this does:**
- `exec` → run a command inside a container
- `-it` → interactive + terminal mode (so you get a live shell session)
- `nginx-pod` → the name of the Pod to enter
- `-n nginx` → the namespace it lives in
- `-- bash` → the command to run once inside (opens a bash shell); the space before and after `--` is important, otherwise kubectl won't understand the command correctly

**Expected result:** Your terminal prompt changes to something like `root@nginx-pod:/#`, meaning you are now inside the container.

## Step 5 — Confirm Nginx is Working (from inside the Pod)

While inside the Pod:
```bash
ls
```
**Expected output:** Standard Linux folder structure, confirming you are inside a working container.

Test that Nginx is actually serving content:
```bash
curl 127.0.0.1:80
```
**Expected output:** Should include the text:
```text
Welcome to nginx!
```

**Exit the Pod shell:**
```bash
exit
```

## Step 6 — Debugging Commands

### `kubectl describe` — See Full Details and Event History of a Pod

```bash
kubectl describe pod nginx-pod -n nginx
```

**What this does:** Shows detailed information about the Pod, including:
- Pod name and namespace
- Priority (default if none was set)
- Service account used
- Which Node the Pod was scheduled onto
- Start time
- Pod IP address
- Container ID and image used
- Current state (e.g., `Running`)
- Default volume mounts
- **Events** — a full history log of what happened step by step:
  1. Scheduler assigned the Pod to a specific worker node
  2. Kubelet on that node pulled the Nginx image
  3. Kubelet started the container
  4. This information was reported back to the API Server, which stored it in etcd

**Why this matters:** `kubectl describe` is one of the most important debugging commands — it shows you exactly what happened and where something might have gone wrong (e.g., failed image pull, scheduling issues, etc.).

### `kubectl exec` — Recap
As shown in Step 4, this is the second most important command for debugging — letting you go inside a running container to check things directly (files, running processes, test connectivity with `curl`, etc.).

## Troubleshooting

### Error: `No resources found in default namespace`
**Reason:** Forgot to add `-n nginx` when checking the Pod.
**Fix:**
```bash
kubectl get pods -n nginx
```

### Error: Pod status stuck in `ImagePullBackOff` or `ErrImagePull`
**Reason:** Incorrect image name, or no internet access from the cluster nodes to reach Docker Hub.
**Fix:** Double-check the `image:` field spelling in `pod.yaml`, and confirm your nodes have internet access.

### Error: Pod stuck in `CrashLoopBackOff`
**Reason:** The container is starting but immediately crashing (application error inside the container).
**Fix:**
```bash
kubectl logs nginx-pod -n nginx
```
**What this does:** Shows the container's log output, which usually reveals why it's crashing.

### Error: `error: unable to upgrade connection: container not found` (while running exec)
**Reason:** Wrong Pod name, wrong namespace, or Pod not yet in `Running` state.
**Fix:** Confirm the Pod is Running first:
```bash
kubectl get pods -n nginx
```
Then retry the `exec` command with the exact correct Pod name and namespace.

## Final Result
You now know how to:
- Write a complete Pod manifest YAML file from scratch
- Apply it correctly inside a specific namespace
- Verify a Pod is running (remembering to specify `-n <namespace>`)
- Enter a running Pod using `kubectl exec -it ... -- bash`
- Use `kubectl describe` to debug and understand exactly what happened when a Pod was created
- Use `kubectl logs` to check application errors when a Pod crashes

## Classroom Safety Check
- [x] Prerequisites covered (cluster + namespace)
- [x] Manifest file explained field by field
- [x] Apply command covered
- [x] Verification with correct namespace flag covered
- [x] Exec/debug commands covered
- [x] Troubleshooting for common Pod errors covered
- [x] Final expected result stated

---
**Note:** A Pod created directly like this (without a Deployment) is NOT auto-healing or scalable. The next topics in this course (Deployments, ReplicaSets, Services) will build on top of this Pod concept to add auto-healing, scaling, and external access — as outlined in Chapter 1's roadmap.

**Chapters still pending (to be added later, as per your instructions):** Deployments, ReplicaSets, DaemonSets, StatefulSets, CronJobs, Services, Ingress, Persistent Volumes/Storage Classes/PVC, Horizontal/Vertical Pod Autoscaling, Probes, RBAC, Helm, Monitoring (Prometheus/Grafana), CI/CD (Jenkins), GitOps (ArgoCD), and the final 3-tier mega project on EKS.
