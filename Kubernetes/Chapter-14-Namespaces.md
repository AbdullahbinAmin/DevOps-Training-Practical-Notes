# Practical 14 — Namespaces

## Objective
Understand what a Namespace is, view default namespaces, and practice creating a namespace both with a direct command and with a YAML manifest file.

## Architecture / Flow
```text
Cluster → Contains multiple Namespaces → Each Namespace isolates its own Pods/Deployments/Services
```

## Concept: What is a Namespace?

Imagine WhatsApp groups. You have different groups: a friends group, a family group, a work group. Each group is separate and keeps its own members and conversations isolated from other groups.

In Kubernetes, a **Namespace** works the same way:
- It is a way to **group related resources together** (Pods, Deployments, Services, etc.)
- Resources inside one namespace do **not disturb or interfere** with resources in another namespace
- This makes managing a busy cluster (with many different applications running) much easier

### Example
- An Nginx Pod inside the `nginx` namespace does not affect a MySQL Pod inside a different namespace.
- This isolation makes the cluster easier to manage.

## Prerequisites
- A working KIND cluster (from Chapter 11) or Minikube/kubeadm cluster
- `kubectl` configured and pointing to your cluster

## Pre-Check
```bash
kubectl cluster-info
```
**Expected result:** Should show your cluster's control plane address (confirms kubectl is connected properly).

## Step 1 — Create a Working Folder for This Course

```bash
mkdir kubernetes-in-one-shot
cd kubernetes-in-one-shot
```
**What this does:** Creates one main folder to keep all YAML files organized for the rest of this course.

## Step 2 — View Existing Namespaces

```bash
kubectl get namespace
```
or the short form:
```bash
kubectl get ns
```

**Expected output (default namespaces that come built-in with every cluster):**
```text
NAME                 STATUS   AGE
default              Active   10m
kube-node-lease      Active   10m
kube-public          Active   10m
kube-system          Active   10m
local-path-storage   Active   10m
```

**What each default namespace means:**
- `default` → If you don't specify any namespace when creating a resource, it automatically goes here. (Just like an orphan automatically belongs to "no one specific" — resources with no namespace specified land here by default.)
- `kube-node-lease` → Stores information/heartbeat about each node in the cluster.
- `kube-public` → For resources that need to be publicly accessible across the whole cluster.
- `kube-system` → Where Kubernetes' own system services run — API Server, Scheduler, Controller Manager, etcd, CoreDNS, kube-proxy, etc.
- `local-path-storage` → Used for storage-related Pods (relevant to Persistent Volumes, covered in a later chapter).

## Step 3 — Check What's Running Inside `kube-system`

```bash
kubectl get pods
```
**Expected output:**
```text
No resources found in default namespace.
```
**Why:** Because nothing has been created in the `default` namespace yet.

Now check the `kube-system` namespace:
```bash
kubectl get pods -n kube-system
```
**What this does:** The `-n` flag lets you specify which namespace to look into.

**Expected output:** You should see all the core Kubernetes components running as Pods, such as:
```text
NAME                                             READY   STATUS    RESTARTS   AGE
coredns-xxxxxxxxx-xxxxx                          1/1     Running   0          10m
etcd-k8s-in-one-shot-control-plane               1/1     Running   0          10m
kube-apiserver-k8s-in-one-shot-control-plane     1/1     Running   0          10m
kube-controller-manager-k8s-in-one-shot-...      1/1     Running   0          10m
kube-proxy-xxxxx                                 1/1     Running   0          10m
kube-scheduler-k8s-in-one-shot-control-plane     1/1     Running   0          10m
```
This confirms: all the components we studied in Chapter 6 (API Server, etcd, Controller Manager, Scheduler, kube-proxy) are literally running as Pods inside the `kube-system` namespace.

## Step 4 — Create a Namespace Using a Direct Command

```bash
kubectl create ns nginx
```
**What this does:** Creates a new namespace called `nginx` directly via command line.

**Expected output:**
```text
namespace/nginx created
```

**Verify:**
```bash
kubectl get ns
```
**Expected output:** `nginx` should now appear in the list.

## Step 5 — Create a Pod Inside a Specific Namespace (Command Line Method)

```bash
kubectl run nginx --image=nginx
```
**What this does:** Creates a Pod named `nginx` using the `nginx` Docker image (pulled from Docker Hub).

**Expected output:**
```text
pod/nginx created
```

Now check if it landed in the `nginx` namespace:
```bash
kubectl get pods -n nginx
```
**Expected output:**
```text
No resources found in nginx namespace.
```

**Why:** Because we did NOT specify `-n nginx` when creating the Pod, so it went into the `default` namespace instead!

### If Something Goes Wrong (Pod in wrong namespace)
**Fix:** Delete the pod and recreate it correctly:
```bash
kubectl delete pod nginx
```
**Expected output:**
```text
pod "nginx" deleted
```

Now recreate it, this time specifying the namespace:
```bash
kubectl run nginx --image=nginx -n nginx
```
**Expected output:**
```text
pod/nginx created
```

**Verify:**
```bash
kubectl get pods -n nginx
```
**Expected output:** The `nginx` pod should now show up, with status `Running`.

## Step 6 — Clean Up Command-Line Created Resources

Since this course prefers YAML manifest files over command-line creation (easier to track and remember what was done), clean up what we just made:

```bash
kubectl delete pod nginx -n nginx
kubectl delete namespace nginx
```
**Expected output:**
```text
pod "nginx" deleted
namespace "nginx" deleted
```

## Step 7 — Create a Namespace Using a YAML Manifest File (Recommended Method)

**Step 1:** Create a folder specifically for this Nginx example:
```bash
mkdir nginx
cd nginx
```

**Step 2:** Create the manifest file:
```bash
vim namespace.yaml
```

**Step 3:** Paste this complete YAML content:
```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: nginx
```

**Explanation of fields:**
- `kind: Namespace` → tells Kubernetes what type of resource this file describes (a Namespace).
- `apiVersion: v1` → the Kubernetes API version this resource type uses (since Kubernetes was on major version 1 at the time of recording — `v1.31.0` — the core API server version is `v1`).
- `metadata` → an object containing extra information about the resource.
- `name: nginx` → the name you want to give this namespace.

**Step 4:** Save and exit (`Esc`, `:wq`, Enter).

**Step 5:** Verify the file content:
```bash
cat namespace.yaml
```

## Step 8 — Apply the Manifest File

```bash
kubectl apply -f namespace.yaml
```

**What this does:** `apply` reads your YAML file and creates (or updates, if it already exists) the resource described inside it on the cluster, via the API Server.

⚠️ **Note on `create` vs `apply`:**
- `kubectl create -f file.yaml` → only works the first time (creates once); running it again on an existing resource gives an error.
- `kubectl apply -f file.yaml` → works for both creating AND updating; safe to run multiple times. This course will always use `apply`.

**Expected output:**
```text
namespace/nginx created
```

**Verify:**
```bash
kubectl get ns
```
**Expected output:** `nginx` should be listed with status `Active`.

## Key Concept Recap

> **Everything in Kubernetes is a manifest file.** When you `apply` a manifest file, it gets sent to the API Server, which creates a matching Kubernetes Object/Resource in the cluster.

## Troubleshooting

### Error: `No resources found in <namespace> namespace`
**Reason:** Either nothing has been created yet in that namespace, or the resource was created in a different namespace by mistake.
**Fix:** Double-check the namespace with `kubectl get pods -A` (the `-A` flag shows Pods across ALL namespaces).

### Error: `Error from server (AlreadyExists): namespaces "nginx" already exists`
**Reason:** You tried to `create` a namespace that already exists.
**Fix:** Use `kubectl apply -f namespace.yaml` instead of `create`, since `apply` won't fail if it already exists.

## Final Result
You now understand:
- What Namespaces are and why they matter for isolating resources
- The 5 default namespaces every cluster comes with
- How to create a namespace both via direct command and via a YAML manifest file
- The difference between `kubectl create` and `kubectl apply`
- That Kubernetes system components themselves run as Pods inside `kube-system`

---
**Prerequisite for Chapter 15:** The `nginx` namespace created in this chapter will be reused in Chapter 15 to create a Pod properly inside it.
