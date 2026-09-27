# Practical 39 — Vertical Pod Autoscaler (VPA)

## Objective
Install the Vertical Pod Autoscaler components from the official Kubernetes Autoscaler repository, configure VPA for the Apache Deployment, and observe a single Pod's resource allocation grow automatically under load (instead of creating more replicas, like HPA does).

## Concept: Recap — HPA vs VPA

- **HPA** (Chapter 38): increases the **number of Pods** (horizontal = "sideways," more copies).
- **VPA** (this chapter): increases the **resources given to an existing Pod** (vertical = "upward," a bigger/stronger single Pod).

VPA is generally more useful for **stateful applications** (like databases) where you can't simply add more identical replicas without extra complexity (data consistency, replication setup, etc.) — so instead, you make the single instance more powerful.

## Prerequisites
- KIND cluster running
- Metrics Server installed and working (from Chapter 38)
- Apache Deployment and Service from Chapter 38

## Step 1 — Clone the Kubernetes Autoscaler Repository

VPA is not a built-in Kubernetes feature — it must be installed from the official `autoscaler` project repository.

```bash
git clone https://github.com/kubernetes/autoscaler.git
```

**What this does:** Downloads the full `autoscaler` project source code, which contains components for Cluster Autoscaler, Vertical Pod Autoscaler, and related add-ons.

⚠️ **Note:** This repository is large, and the clone may take a few minutes depending on your connection.

## Step 2 — Navigate to the VPA Folder and Install It

```bash
cd autoscaler/vertical-pod-autoscaler
```

Check the official installation instructions inside this folder (usually in its README), which typically point to a helper script:

```bash
./hack/vpa-up.sh
```

**What this does:** Runs the official installation script that sets up all required VPA components: the VPA Recommender, VPA Updater, VPA Admission Controller, and their associated RBAC permissions.

**Expected output:** A series of "created" messages for multiple resources (Deployments, Services, ClusterRoles, etc.), ending with confirmation that the VPA components are deployed.

**Verify installation:**
```bash
kubectl get pods -n kube-system | grep vpa
```
**Expected output:** You should see VPA-related Pods (recommender, updater, admission-controller) in `Running` state.

## Step 3 — Write the VPA Manifest for Apache

```bash
cd ~/kubernetes-in-one-shot/apache
vim vpa.yaml
```

```yaml
kind: VerticalPodAutoscaler
apiVersion: autoscaling.k8s.io/v1
metadata:
  name: apache-vpa
  namespace: apache
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-deployment
  updatePolicy:
    updateMode: "Auto"
```

**Explanation of every field:**
- `kind: VerticalPodAutoscaler` → this manifest describes a VPA resource.
- `apiVersion: autoscaling.k8s.io/v1` → VPA uses its own custom API group, `autoscaling.k8s.io` (different from HPA's built-in `autoscaling/v2`, since VPA is installed as an add-on, not a core Kubernetes feature).
- `spec.targetRef` → same idea as HPA's `scaleTargetRef` — identifies the exact Deployment to manage (`apache-deployment`).
- `spec.updatePolicy.updateMode` → controls HOW the VPA applies its recommendations:
  - `"Off"` → VPA only calculates recommendations but doesn't apply them automatically (useful for observing before committing).
  - `"Initial"` → VPA only sets resource values when a Pod is first created, not afterward.
  - `"Auto"` → VPA actively updates the resource requests/limits automatically, based on observed usage (this is what we're using here).

⚠️ **Common Error — Spelling `spec` incorrectly (e.g., `spce`):**
```text
error: error validating "vpa.yaml": ... unknown field "spce" ...
```
**Fix:** Simply correct the typo to `spec`.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 4 — Remove the HPA First (Avoid Conflicts)

Since HPA and VPA both try to manage the same Deployment's scaling behavior, running both on the same target at once can cause conflicts (especially over CPU, if both are configured to react to it).

```bash
kubectl delete hpa apache-hpa -n apache
```

## Step 5 — Apply the VPA

```bash
kubectl apply -f vpa.yaml
```
**Expected output:**
```text
verticalpodautoscaler.autoscaling.k8s.io/apache-vpa created
```

## Step 6 — Verify

```bash
kubectl get vpa -n apache
```
**Expected output:**
```text
NAME          MODE   CPU   MEM         PROVIDED   AGE
apache-vpa    Auto   25m   262144k     True       15s
```
**Field meaning:** `PROVIDED: True` confirms the VPA has successfully connected to the Metrics Server and is actively generating recommendations.

## Step 7 — Watch the VPA Live in a Separate Terminal

```bash
kubectl get vpa -n apache --watch
```

## Step 8 — Generate Load (Same Method as Chapter 38)

In another terminal tab, restart port forwarding if needed:
```bash
kubectl port-forward service/apache-service -n apache 8082:80 --address 0.0.0.0
```

Create a load generator Pod:
```bash
kubectl run load-generator --image=busybox:latest --restart=Always --tty --namespace apache -- /bin/sh
```

Once inside:
```bash
while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done
```

## Step 9 — Observe the VPA Scaling the Pod's Resources

Watch the VPA output over time:
```bash
kubectl get vpa -n apache --watch
```

**Expected behavior:** Over a few minutes, you should see the CPU and memory values recommended by the VPA gradually increase — for example:
```text
NAME          MODE   CPU    MEM         PROVIDED   AGE
apache-vpa    Auto   25m    262144k     True       1m
apache-vpa    Auto   85m    271000k     True       3m
apache-vpa    Auto   109m   285000k     True       6m
```

**Confirm the number of Pods stayed the same (unlike HPA):**
```bash
kubectl get pods -n apache
```
**Expected output:** Still just **1 Apache Pod** — because VPA does not add more replicas; it makes the existing Pod more powerful instead.

**Confirm which node is under the most load:**
```bash
kubectl top node
```
**Expected output:** The specific worker node running the Apache Pod should show notably higher CPU usage than the others.

```bash
kubectl get pods -n apache -o wide
```
**What this does:** Confirms which node the load-generator Pod and the Apache Pod are both running on.

## Step 10 — Clean Up

```bash
kubectl delete pod load-generator -n apache
```

## Key Concept Recap
> **VPA (Vertical Pod Autoscaler)** monitors a target Deployment/StatefulSet's real resource usage via the Metrics Server, and automatically adjusts that Pod's `requests`/`limits` (CPU and memory) to match actual demand — growing (or shrinking) the Pod's resource allocation over time, without changing the number of replicas. This makes VPA well-suited for stateful workloads where scaling out (more replicas) isn't practical.

## Final Result
Students now understand:
- How to install VPA from the official Kubernetes autoscaler repository
- How to write a VPA manifest with different `updateMode` options
- Why HPA and VPA should generally not target the same resource simultaneously
- How to observe a single Pod's resource allocation grow automatically under sustained load
- The clear practical distinction between HPA (more Pods) and VPA (bigger Pods)

---
**Prerequisite for Chapter 40:** None extra — Chapter 40 (Node Affinity) covers another Pod-placement concept, complementary to Taints and Tolerations from Chapter 37.
