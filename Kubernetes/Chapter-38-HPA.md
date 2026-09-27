# Practical 38 — Horizontal Pod Autoscaler (HPA)

## Objective
Install a Metrics Server on the KIND cluster, deploy a sample Apache application, and configure a Horizontal Pod Autoscaler (HPA) that automatically increases/decreases the number of Pod replicas based on live CPU usage.

## Concept: Auto Scaling Overview

As applications move to microservices architecture (Chapter 5), they need to scale automatically based on real traffic. There are a few auto-scaling approaches in Kubernetes:

1. **HPA (Horizontal Pod Autoscaler)** → increases/decreases the **number of Pod replicas**.
2. **VPA (Vertical Pod Autoscaler)** → increases/decreases the **resources (CPU/memory) given to each existing Pod** (covered in Chapter 39).
3. **KEDA (Kubernetes Event-Driven Autoscaling)** → scales based on external event sources/metrics (not commonly used in typical setups, but useful to know the name for interviews).

### HPA vs VPA — Simple Distinction
- **HPA**: "Traffic increased a lot → create 3 more Pod replicas."
- **VPA**: "Traffic increased a lot → make the existing Pod itself bigger/stronger (increase its CPU/memory limits)."

> **VPA is typically used for stateful applications** (like MySQL), while **HPA is typically used for stateless applications** (like a web server).

### Metrics — The Foundation of Both
Both HPA and VPA rely on **Metrics** — quantifiable resource usage numbers (how much CPU% is being used, how much memory is being used). Without a way to measure these metrics, autoscaling cannot work.

## Prerequisites
- KIND cluster running
- kubectl configured

## Part A — Install the Metrics Server

### Step 1 — Check If Metrics Are Available

```bash
kubectl top node
```
**Expected output (before installing):**
```text
error: Metrics API not available
```

```bash
kubectl top pod
```
**Expected output:** Same error.

**Reason:** By default, a KIND cluster does NOT come with a Metrics Server installed — it needs to be installed separately.

⚠️ **Note:** If you are using **Minikube**, this is much simpler:
```bash
minikube addons enable metrics-server
```

### Step 2 — Install Metrics Server (KIND-Specific)

⚠️ **Verification Required:** Always check the official Metrics Server GitHub releases page for the latest recommended install command, since URLs and versions change: https://github.com/kubernetes-sigs/metrics-server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**What this does:** Installs the Metrics Server Deployment, Service, and required RBAC roles (Role-Based Access Control, covered in a later chapter) into the `kube-system` namespace.

**Expected output:** A series of "created" messages.

### Step 3 — Fix TLS Issues (Required for KIND/Self-Signed Certificate Setups)

Since our cluster doesn't have a properly signed HTTPS certificate (unlike a production cloud cluster), the Metrics Server needs a small adjustment to bypass strict certificate validation for internal communication:

```bash
kubectl edit deployment metrics-server -n kube-system
```

**What this does:** Opens the Metrics Server's Deployment YAML directly in your terminal editor for live editing.

Find the `containers.args` section and add these two lines:
```yaml
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
```

**Explanation:**
- `--kubelet-insecure-tls` → tells the Metrics Server to trust the Kubelet's connection even without a fully valid TLS/HTTPS certificate (fine for local/lab clusters; not recommended without understanding the security trade-off in real production).
- `--kubelet-preferred-address-types` → tells the Metrics Server which type of address to use when reaching each Kubelet.

⚠️ **Watch your indentation carefully** when pasting these lines into the `args` list — incorrect indentation will cause a validation error when you save.

**Save and exit** (`Esc`, `:wq`, Enter).
**Expected output:**
```text
deployment.apps/metrics-server edited
```

### Step 4 — Restart the Metrics Server

```bash
kubectl rollout restart deployment metrics-server -n kube-system
```
**What this does:** Gradually restarts the Metrics Server Pods (using the same Rolling Update concept from Chapter 19) so the new configuration takes effect.

**Expected output:**
```text
deployment.apps/metrics-server restarted
```

### Step 5 — Verify Metrics Server Is Running

```bash
kubectl get pods -n kube-system
```

### If Something Goes Wrong — Pod Stuck `Pending`
If the metrics-server Pod stays `Pending`, check for **Taints** left over from Chapter 37:
```bash
kubectl describe pod <metrics-server-pod-name> -n kube-system
```
If you see taint-related scheduling failures, remove any leftover taints on your worker nodes:
```bash
kubectl taint node <node-name> prod=true:NoSchedule-
```
(Repeat for each tainted node from Chapter 37's practical.)

**Once fixed, verify:**
```bash
kubectl top node
```
**Expected output:**
```text
NAME                   CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
kind-control-plane     ...          4%     669Mi           ...
kind-worker            ...          ...    238Mi           ...
kind-worker2           ...          ...    284Mi           ...
kind-worker3           ...          ...    299Mi           ...
```

```bash
kubectl top pod -n nginx
```
**Expected output:** Should show live CPU/memory usage of each Pod, confirming the Metrics Server is now fully functional.

## Part B — Deploy a Sample Apache Application

### Step 1 — Create the Namespace

```bash
mkdir apache
cd apache
vim namespace.yaml
```
```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: apache
```
```bash
kubectl apply -f namespace.yaml
```

### Step 2 — Create the Deployment

```bash
vim deployment.yaml
```
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: apache-deployment
  namespace: apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
        - name: apache
          image: httpd:latest
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

**Explanation of new elements:**
- `image: httpd:latest` → **Apache HTTP Server** (the `httpd` image), an Nginx-like web server used here to demonstrate HPA with a fresh example. Version `2.4` (or `latest`) is recommended.
- The `resources` block is essential here — this is exactly what HPA uses to know each Pod's CPU baseline, so it can calculate percentage utilization.

```bash
kubectl apply -f deployment.yaml
```

### If Something Goes Wrong (typo)
If you see:
```text
error: error validating "deployment.yaml": ... unknown field "replica" ...
```
**Fix:** Correct to `replicas` (plural).

If you see:
```text
error: error validating "deployment.yaml": ... selector does not match template labels
```
**Reason:** Forgot to set `labels: app: apache` under `template.metadata`.
**Fix:** Add matching labels under `template.metadata.labels`, as covered in Chapter 18.

**Verify:**
```bash
kubectl get pods -n apache
```

### Step 3 — Create the Service

```bash
vim service.yaml
```
```yaml
kind: Service
apiVersion: v1
metadata:
  name: apache-service
  namespace: apache
spec:
  selector:
    app: apache
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

### If Something Goes Wrong
```text
error: ... field selector of type string can not unmarshal ...
```
**Reason:** `selector` under a Service's `spec` should be a plain key-value map (`app: apache`), NOT nested under a `matchLabels` field (that structure is only used for Deployments/ReplicaSets, not Services).
**Fix:** Use `selector: {app: apache}` directly, as shown above.

```bash
kubectl apply -f service.yaml
```

**Verify everything is running:**
```bash
kubectl get all -n apache
```

## Part C — Configure HPA for Apache

### Step 1 — Write the HPA Manifest

```bash
vim hpa.yaml
```

```yaml
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2
metadata:
  name: apache-hpa
  namespace: apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 5
```

**Explanation of every field:**
- `kind: HorizontalPodAutoscaler` → this manifest describes an HPA resource.
- `apiVersion: autoscaling/v2` → HPA uses the `autoscaling/v2` API group (note the different naming pattern from other resources — always confirm via official docs).
- `spec.scaleTargetRef` → which resource this HPA should scale:
  - `apiVersion: apps/v1`, `kind: Deployment`, `name: apache-deployment` → this exactly identifies our Apache Deployment from Part B.
- `spec.minReplicas: 1` → never scale below 1 replica.
- `spec.maxReplicas: 5` → never scale above 5 replicas, even under heavy load.
- `spec.metrics` → a list of metrics used to decide when to scale.
  - `type: Resource` → base scaling on a standard resource metric (CPU or memory).
  - `resource.name: cpu` → specifically track CPU usage.
  - `resource.target.type: Utilization` → measure as a **percentage utilization** (relative to the `requests.cpu` value set in the Deployment).
  - `resource.target.averageUtilization: 5` → if average CPU utilization across Pods reaches **5%** (intentionally set very low here, just for an easy demo — in real usage this would typically be something like 50-80%), scale up.

**Save and exit** (`Esc`, `:wq`, Enter).

### Step 2 — Apply the HPA

```bash
kubectl apply -f hpa.yaml
```
**Expected output:**
```text
horizontalpodautoscaler.autoscaling/apache-hpa created
```

**Verify:**
```bash
kubectl get hpa -n apache
```
**Expected output:**
```text
NAME          REFERENCE                       TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
apache-hpa    Deployment/apache-deployment     1%/5%     1         5         1          10s
```

## Part D — Generate Load to Trigger Auto Scaling

### Step 1 — Expose the Apache Service Temporarily

```bash
kubectl port-forward service/apache-service -n apache 8082:80 --address 0.0.0.0
```
Open the corresponding port (`8082`) in your AWS Security Group (same process as earlier chapters).

Confirm it works in a browser:
```text
http://<YOUR_EC2_PUBLIC_IP>:8082
```

### Step 2 — Create a Load Generator Pod

Open a new terminal tab, and create a temporary Pod to continuously send requests:

```bash
kubectl run load-generator --image=busybox:latest --restart=Always --tty --namespace apache -- /bin/sh
```

**What this does:**
- `--image=busybox:latest` → a lightweight image good for running arbitrary commands (recall Chapter 22).
- `--restart=Always` → keeps this Pod running continuously (default behavior for a "Pod-generating" command like this).
- `--tty` → allocates a terminal, so the container waits for interactive input rather than exiting immediately.
- `--namespace apache` → runs in the same namespace, so it can resolve the Apache Service by name.
- `-- /bin/sh` → the shell to run inside (note: BusyBox does NOT have `bash` — it only has `sh`; using `bash` will fail with an error like "executable not found").

### If Something Goes Wrong — `CrashLoopBackOff`
**Reason:** Either `bash` was used (not available in BusyBox), or no long-running command was given so the container exits immediately.
**Fix:**
```bash
kubectl delete pod load-generator -n apache
kubectl run load-generator --image=busybox:latest --restart=Always --tty --namespace apache -- /bin/sh
```

Once inside the shell, run a continuous loop that hits the Apache Service repeatedly:
```bash
while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done
```

**Explanation:**
- `while true; do ... done` → an infinite loop, repeating the command forever.
- `wget -q -O- <url>` → sends a request to the given URL, printing the result quietly (`-q` suppresses extra logging, `-O-` sends output to the terminal instead of a file).
- `apache-service.apache.svc.cluster.local` → the **internal DNS name** of the Apache Service. The format is always:
```text
<service-name>.<namespace>.svc.cluster.local
```
This only works from **inside the cluster** (like from another Pod) — it will NOT work from your own browser outside the cluster.

**Expected output:** Repeated `<html>It works!</html>` (or similar) responses printing rapidly.

## Part E — Watch the Auto Scaling Happen

Open another terminal tab and watch the HPA live:
```bash
kubectl get hpa -n apache --watch
```

**Expected behavior:** As the load generator hammers the Apache Service, you'll see the CPU utilization percentage climb — once it crosses the 5% threshold, watch the `REPLICAS` column increase automatically:
```text
NAME          REFERENCE                       TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
apache-hpa    Deployment/apache-deployment    18%/5%     1         5         4          2m
```

In a separate tab, confirm new Pods were created:
```bash
kubectl get pods -n apache
```
**Expected output:** Multiple new `apache-deployment-xxxxx` Pods appearing automatically, all `Running`.

## Step — Stop the Load and Watch It Scale Back Down

Go back to the load generator's terminal and press `Ctrl + C` to stop the infinite loop.

Then delete the load generator Pod entirely:
```bash
kubectl delete pod load-generator -n apache
```

Watch the HPA again:
```bash
kubectl get hpa -n apache --watch
```
**Expected behavior:** Over the next few minutes, as CPU usage drops back down, the `REPLICAS` count will gradually decrease back toward `minReplicas: 1`.

## Key Concept Recap
> **HPA (Horizontal Pod Autoscaler)** watches a specified metric (commonly CPU utilization %) via the Metrics Server, and automatically increases or decreases the number of Pod replicas for a Deployment/StatefulSet, between a defined `minReplicas` and `maxReplicas`, keeping your application responsive under variable load without manual intervention.

## Final Result
Students now understand:
- How to install and fix TLS issues for a Metrics Server on a KIND cluster
- How to check live resource usage with `kubectl top node` / `kubectl top pod`
- How to write a complete HPA manifest targeting a Deployment
- How to generate test load using a BusyBox Pod and internal cluster DNS
- How to observe automatic scale-up and scale-down behavior live

---
**Prerequisite for Chapter 39:** The Metrics Server installed here is also required for Chapter 39 (VPA — Vertical Pod Autoscaler).
