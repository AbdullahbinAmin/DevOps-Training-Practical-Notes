# Practical 36 — Probes (Liveness, Readiness, Startup)

## Objective
Understand the three types of Probes Kubernetes uses to check Pod health, and add a Liveness Probe to a real Deployment.

## Concept: What is a Probe?

> **A Probe is a periodic request/check that Kubernetes sends to a Pod to verify it is working correctly.**

There are three types of Probes:

### 1. Liveness Probe
Checks: **"Is this Pod alive (or has it effectively died/hung)?"**
- If a Liveness Probe fails repeatedly, Kubernetes will **restart** the container, assuming it's stuck or broken.

### 2. Readiness Probe
Checks: **"Is this Pod ready to accept traffic?"**
- A Pod might be alive but not yet ready (e.g., still loading data or connecting to a database). If the Readiness Probe fails, Kubernetes will **stop sending traffic** to that Pod (via its Service) until it becomes ready — but it will NOT restart the container.

### 3. Startup Probe
Checks: **"Has this Pod finished starting up correctly?"**
- Useful for applications that take a long time to start. Kubernetes waits for the Startup Probe to succeed before it even begins running Liveness/Readiness checks.

### Real Example
Imagine a Pod running on port 8000. Kubernetes will internally send a request to that port to check if something is actually responding there.

## Prerequisites
- Notes App Deployment (from Chapter 29), running on `containerPort: 8000`

## Step 1 — Add a Liveness Probe to the Deployment

```bash
vim deployment.yaml
```

Add a `livenessProbe` block inside the container spec:

```yaml
      containers:
        - name: notes-app
          image: <YOUR_DOCKERHUB_USERNAME>/notes-app-k8s:latest
          ports:
            - containerPort: 8000
          livenessProbe:
            httpGet:
              path: /
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
```

**Explanation of every field:**
- `livenessProbe` → defines the health check that determines whether the container needs to be restarted.
- `httpGet` → tells Kubernetes to perform an HTTP GET request as the check method (other options include `tcpSocket` and `exec`, for checking a raw TCP connection or running a command, respectively).
- `httpGet.path: /` → the request will be sent to the root path (`/`) of the application.
- `httpGet.port: 8000` → the port to send this request to — matching the application's actual listening port.
- `initialDelaySeconds: 5` → wait 5 seconds after the container starts before running the first probe (gives the app time to boot up).
- `periodSeconds: 10` → run this probe check every 10 seconds thereafter.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 2 — Apply

```bash
kubectl apply -f deployment.yaml
```
**Expected output:**
```text
deployment.apps/notes-app-deployment configured
```

## Step 3 — Verify Pod Status

```bash
kubectl get pods -n nginx
```
Wait for the Pod to become `Running` and `1/1 Ready`.

## Step 4 — Check the Probe Results

```bash
kubectl describe pod <pod-name> -n nginx
```

**Expected output:** Look under the "Events" section near the bottom of the output. You may see something like:
```text
Warning  Unhealthy  ...  Readiness probe failed: Get "http://10.244.x.x:8000/": dial tcp ... connect: connection refused
```

### If Something Goes Wrong — Probe Failing on Cluster IP
**Reason:** In some setups (like when using a Cluster-internal Pod IP that isn't reachable the way you expect within a specific test environment), a probe check may fail even though the application itself is running fine. This is a common source of confusion.
**Fix / Understanding:** Confirm the application is genuinely reachable using the same method the probe uses (e.g., manually run `curl http://localhost:8000/` from inside the Pod, using `kubectl exec`). If it responds correctly internally, the probe configuration is correct, and any failure is typically related to networking specifics of your exact test setup (which would behave correctly on a real cloud-managed cluster like EKS, as noted in this course).

## Step 5 — Confirm the App Itself Works (Manual Check)

```bash
kubectl exec -it <pod-name> -n nginx -- bash
```
Inside the container:
```bash
curl http://localhost:8000/
```
**Expected output:** Should return HTML content from the Django app, confirming the application itself is healthy — even if a probe check in a specific test environment shows a warning.

Exit:
```bash
exit
```

## Key Concept Recap
> **Probes** are simply automated health checks:
> - **Liveness Probe** → "should I restart this container?"
> - **Readiness Probe** → "should I send traffic to this Pod right now?"
> - **Startup Probe** → "has this slow-starting app finished booting?"
>
> They are configured using `httpGet`, `tcpSocket`, or `exec` checks, along with timing controls like `initialDelaySeconds` and `periodSeconds`.

## Final Result
Students now understand:
- The three types of Probes and what each one checks for
- How to add a Liveness Probe with `httpGet` to a Deployment
- How to read probe-related events using `kubectl describe pod`
- How to manually verify an application is actually healthy, to distinguish real failures from environment-specific test quirks

---
**Prerequisite for Chapter 37:** None extra — Chapter 37 (Taints and Tolerations) moves into a new topic about controlling which Nodes a Pod is allowed to be scheduled on.
