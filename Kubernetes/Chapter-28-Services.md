# Practical 28 — Services

## Objective
Learn what a Kubernetes Service is, create one to expose the Nginx Deployment to the outside world, and access it through a web browser.

## Concept: What is a Service?

Recall from Chapter 6: Pods are **isolated by default** — you cannot access them directly from outside the cluster. To make a Pod (or group of Pods managed by a Deployment) accessible from outside, you need a **Service**.

### Analogy
Imagine you have many small shops (Pods), all grouped together under one big brand (Deployment). A **Service** acts like a receptionist or redirect desk — when a customer (outside user) arrives, the Service redirects them to any one of the available shops (Pods) that match what they're looking for.

> **A Service exposes a set of Pods (usually managed by a Deployment) to be accessible, either within the cluster or from outside it.**

## Prerequisites
- `nginx-deployment` running (from Chapter 27) with the label `app: nginx`
- `nginx` namespace present

## Step 1 — Search for the Official Reference (If Needed)

Just like with Pods and Deployments, you can always search **"kubernetes service"** on Google and refer to the official documentation for the exact field structure if you forget something.

## Step 2 — Create the Service Manifest

```bash
vim service.yaml
```

Paste this complete YAML:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx-service
  namespace: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: ClusterIP
```

**Explanation of every field:**
- `kind: Service` → this manifest describes a Service resource.
- `apiVersion: v1` → Services use the core `v1` API version (same as Pods and Namespaces).
- `metadata.name: nginx-service` → the name of this Service.
- `metadata.namespace: nginx` → the namespace it belongs to.
- `spec.selector.app: nginx` → **this is how the Service finds its Pods** — same labels-and-selectors concept from Chapter 18. Any Pod with the label `app: nginx` will be included and exposed by this Service.
- `spec.ports` → defines the port mapping:
  - `port: 80` → the port that the **outside world** connects to, on the Service itself.
  - `targetPort: 80` → the port that the **Pod's container** is actually listening on (Nginx listens on port 80 by default, as set in our Deployment's `containerPort`).
  - `protocol: TCP` → the network protocol used.
- `spec.type: ClusterIP` → the type of Service. Types include:
  - **ClusterIP** (default if not specified) → only accessible from **within** the cluster, gets an internal cluster IP address.
  - **NodePort** → exposes the Service on a specific port (in the range 30000–32000) on every Node, making it accessible from outside using `<NodeIP>:<NodePort>`.
  - **LoadBalancer** → typically used with cloud providers, automatically provisions an external load balancer.
  - **ExternalIP** → manually assign an external/static IP address to the Service.
  - **Headless Service** → typically used with StatefulSets (no single cluster IP; each Pod gets direct DNS resolution) — not covered in this practical yet.

⚠️ For this practical, we use `ClusterIP` since it's simplest for local testing (combined with `kubectl port-forward` to actually view it in a browser).

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 3 — Apply the Service

```bash
kubectl apply -f service.yaml
```

### If Something Goes Wrong (typo)
If you see:
```text
error: error validating "service.yaml": ... unknown field "port" in io.k8s.api.core.v1.ServiceSpec
```
**Reason:** Typo — should be `ports` (plural, a list), not `port`.
**Fix:** Correct the field name and reapply:
```bash
kubectl apply -f service.yaml
```

**Expected output:**
```text
service/nginx-service created
```

## Step 4 — Verify the Service

```bash
kubectl get svc -n nginx
```
**Expected output:**
```text
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.96.xxx.xxx   <none>        80/TCP    10s
```

## Step 5 — Try Accessing Directly (Will Fail — Expected)

Try accessing the ClusterIP address directly in a browser using its IP and port 80 — this will **not work**, because:
1. `ClusterIP` is only reachable from **inside** the cluster network, not from your own browser/machine.
2. Our entire cluster is running **inside a Docker container** (KIND), so even the Node's own network is isolated from our EC2 host machine's network.

## Step 6 — Use Port Forwarding to Access From Your Browser

```bash
kubectl port-forward service/nginx-service -n nginx 8080:80 --address 0.0.0.0
```

**What this does:**
- `port-forward` → creates a temporary tunnel from a port on your local/host machine into a port inside the cluster.
- `service/nginx-service` → target the Service (instead of a specific Pod).
- `-n nginx` → the namespace.
- `8080:80` → format is `<local-port>:<service-port>` — forwards local port `8080` to the Service's port `80`.
- `--address 0.0.0.0` → makes this port-forward accessible from any network interface (not just `localhost`), which is necessary when running on an EC2 instance you're accessing remotely.

### If Something Goes Wrong (Permission or Port Conflict)
If you see:
```text
bind: permission denied
```
**Reason:** Low-numbered ports (like 80) generally require root/admin permission.
**Fix:** Run the command with elevated privileges:
```bash
sudo -E kubectl port-forward service/nginx-service -n nginx 8080:80 --address 0.0.0.0
```

If you see:
```text
bind: address already in use
```
**Reason:** The local port you chose is already occupied by another process.
**Fix:** Choose a different local port, e.g.:
```bash
kubectl port-forward service/nginx-service -n nginx 8081:80 --address 0.0.0.0
```

**Expected output:**
```text
Forwarding from 0.0.0.0:8081 -> 80
```

## Step 7 — Open the Port in AWS Security Group

Since this traffic needs to reach your EC2 instance from the outside internet, you must open the chosen port in your instance's Security Group (same process as Chapter 13, Step 2):

**Step 1:** Go to EC2 Console → your instance → **Security** tab → click the Security Group.

**Step 2:** Click **Edit inbound rules** → **Add rule**:
- Type: Custom TCP
- Port range: `8081` (or whichever port you used)
- Source: Anywhere (`0.0.0.0/0`)

**Step 3:** Click **Save rules**.

## Step 8 — Access in Your Browser

Open your browser and go to:
```text
http://<YOUR_EC2_PUBLIC_IP>:8081
```

**Expected result:** You should see the **"Welcome to nginx!"** page load successfully in your browser.

## Full Journey Recap

```text
Namespace → Container (via Deployment's Pod template) → Pod → Deployment → Service → You (the User)
```

This is the complete path any application takes in Kubernetes to go from being just a container image, to something a real user can access through their browser.

## Key Concept Recap
> A **Service** exposes a group of Pods (matched via labels/selectors) so they can be accessed — either within the cluster (`ClusterIP`), across all nodes (`NodePort`), or from the internet via a cloud load balancer (`LoadBalancer`). For local testing (like our KIND cluster), `kubectl port-forward` combined with a `ClusterIP` Service lets us view our application in a browser.

## Final Result
Students now understand:
- Why Services are needed to expose Pods
- How Services use labels/selectors to find their target Pods (same concept as Deployments)
- The different Service types (ClusterIP, NodePort, LoadBalancer, ExternalIP, Headless)
- How to use `kubectl port-forward` to access a ClusterIP service from a browser
- How to open the necessary port on AWS EC2's Security Group
- The complete request journey: Namespace → Pod → Deployment → Service → User

---
**Prerequisite for Chapter 29:** All concepts from Chapters 14–28 (Namespaces, Pods, Deployments, Storage, Services) come together in Chapter 29's mini real-world project.
