# Practical 30 — Ingress

## Objective
Understand what Ingress is, install an Ingress Controller on a KIND cluster, and route traffic to multiple different Services based on URL path (e.g., `/nginx` goes to one service, `/app` goes to another).

## Concept: What is Ingress?

Imagine you have multiple Services running:
- An Nginx Service
- A Notes App Service

You want:
- When someone visits `/nginx`, it goes to the Nginx Service
- When someone visits `/app` (or similar), it goes to the Notes App Service

This kind of **traffic routing based on URL path** is done using **Ingress**.

> **Ingress manages external access to Services in a cluster, typically HTTP/HTTPS, and routes traffic based on rules (like hostname or path) to different Services.**

### Why This Matters
As applications grow into a **microservices architecture** (many small independent services, as discussed in Chapter 5), you need a smart way to route incoming traffic to the correct service — this is exactly the problem Ingress solves.

### High-Level Diagram
```text
Cluster
 ├── Deployment A → Pods → Service A
 └── Deployment B → Pods → Service B

Without Ingress: each Service is accessed separately
With Ingress: one entry point routes "/path-a" → Service A, "/path-b" → Service B
```

## Prerequisites
- KIND cluster running (from Chapter 11)
- Nginx Deployment + Service (from Chapters 18, 28)
- Notes App Deployment + Service (from Chapter 29)

## Step 1 — Put Everything in the Same Namespace

For Ingress routing to work cleanly in this demo, all the Services need to be reachable, generally kept in a consistent namespace setup. Move the Notes App resources into the `nginx` namespace (or otherwise ensure your Ingress rules correctly reference each Service's actual namespace):

```bash
kubectl delete deployment notes-app-deployment -n notes-app
kubectl delete service notes-app-service -n notes-app
kubectl delete namespace notes-app
```

**What this does:** Cleans up the previous Notes App setup so it can be recreated in the `nginx` namespace for this Ingress demo.

Edit `notes-app` Deployment and Service YAML files, changing `metadata.namespace: notes-app` to `metadata.namespace: nginx`, then reapply:
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

**Verify both apps are running in the same namespace:**
```bash
kubectl get pods -n nginx
kubectl get svc -n nginx
```
**Expected output:** You should see both the Nginx Deployment/Pods/Service AND the Notes App Deployment/Pods/Service, all within the `nginx` namespace.

## Step 2 — Install an Ingress Controller

Ingress rules alone do nothing — you need an **Ingress Controller**, a tool that actually performs the routing at the cluster level. This course uses the **Nginx Ingress Controller** (an open-source project, not to be confused with the Nginx web server itself — this is a separate community-built controller).

⚠️ **Important:** Since we're using a KIND cluster, you must use the **KIND-specific installation manifest** for the Nginx Ingress Controller (not the generic one) — check the official KIND documentation for this: https://kind.sigs.k8s.io/docs/user/ingress/

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

**What this does:** Installs all the components (Deployment, Service, RBAC roles, etc.) needed to run the Nginx Ingress Controller inside your KIND cluster.

**Expected output:** A series of "created" messages for multiple resources.

## Step 3 — Verify the Ingress Controller

```bash
kubectl get ns
```
**Expected output:** A new namespace called `ingress-nginx` should now exist.

```bash
kubectl get pods -n ingress-nginx
```
**Expected output:**
```text
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-xxxxx        0/1     Completed   0          30s
ingress-nginx-admission-patch-xxxxx         0/1     Completed   0          30s
ingress-nginx-controller-xxxxxxxxx-xxxxx    1/1     Running     0          30s
```
**Field meaning:** `Completed` means those were one-time Jobs (recall Chapter 22) that already finished. The `ingress-nginx-controller` Pod, marked `Running`, is the actual controller that will handle routing.

```bash
kubectl get svc -n ingress-nginx
```
**Expected output:** You should see a Service (usually named `ingress-nginx-controller`) with type `LoadBalancer` or `NodePort` — this Service will need to be exposed later (Step 7).

## Step 4 — Write the Ingress Manifest

```bash
vim ingress.yaml
```

Paste this complete YAML:

```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: nginx-notes-ingress
  namespace: nginx
spec:
  rules:
    - host: "training.shubham.com"
      http:
        paths:
          - path: /nginx
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: notes-app-service
                port:
                  number: 8000
```

**Explanation of every field:**
- `kind: Ingress` → this manifest describes an Ingress resource.
- `apiVersion: networking.k8s.io/v1` → Ingress belongs to the `networking.k8s.io` API group (different from `v1` for Pods, or `apps/v1` for Deployments — always check the official docs when unsure).
- `metadata.name` / `metadata.namespace` → name and namespace of this Ingress. Keeping it in the same namespace as the Services it routes to keeps things simpler.
- `spec.rules` → a list of routing rules.
  - `host` → the hostname this rule applies to (e.g., a domain name). This can be a placeholder value for local testing.
  - `http.paths` → a list of URL paths and where each should route to:
    - `path: /nginx` → whenever the URL path starts with `/nginx`...
    - `pathType: Prefix` → matches any URL that **starts with** this path (as opposed to `Exact`, which would only match the exact path).
    - `backend.service.name: nginx-service` → ...route it to the `nginx-service` Service.
    - `backend.service.port.number: 80` → ...specifically on port 80.
    - The second path block (`/app`) does the same thing, but routes to `notes-app-service` on port `8000`.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 5 — Apply the Ingress

```bash
kubectl apply -f ingress.yaml
```

### If Something Goes Wrong (typo)
If you see:
```text
error: error validating "ingress.yaml": ... unknown field "kind" ... (or similar case-sensitivity error)
```
**Reason:** A common typo is writing `king` instead of `kind`, or similar small typos — always re-read carefully.
**Fix:** Correct the typo and reapply:
```bash
kubectl apply -f ingress.yaml
```

**Expected output:**
```text
ingress.networking.k8s.io/nginx-notes-ingress created
```

## Step 6 — Verify Everything Is Running

```bash
kubectl get all -n nginx
```
**Expected output:** You should see the Nginx Deployment/Pods/Service, the Notes App Deployment/Pods/Service, AND your new Ingress resource, all listed together.

## Step 7 — Expose the Ingress Controller

```bash
kubectl get svc -n ingress-nginx
```
**Note the Service name** (usually `ingress-nginx-controller`).

```bash
kubectl port-forward service/ingress-nginx-controller -n ingress-nginx 8080:80 --address 0.0.0.0
```

### If Something Goes Wrong (port already in use)
If you see `bind: address already in use`, choose a different local port:
```bash
kubectl port-forward service/ingress-nginx-controller -n ingress-nginx 8080:80 --address 0.0.0.0
```

**Expected output:**
```text
Forwarding from 0.0.0.0:8080 -> 80
```

## Step 8 — Open the Port in AWS Security Group

Follow the same steps as Chapter 28: EC2 Console → Security Group → Edit inbound rules → Add rule for port `8080`, source `0.0.0.0/0` → Save.

## Step 9 — Test in the Browser

Open:
```text
http://<YOUR_EC2_PUBLIC_IP>:8080/nginx
```
**Expected result:** The Welcome to Nginx page loads.

Open:
```text
http://<YOUR_EC2_PUBLIC_IP>:8080/app
```
**Expected result:** This may show a `404 Page Not Found` error at this stage. This is expected and will be debugged and fixed using **Annotations**, covered in the next chapter (31).

## Troubleshooting

### Error: `/app` shows "Page Not Found"
**Reason:** The Django application at `notes-app-service` expects requests at the root path `/`, not `/app` — so when Ingress forwards the request still carrying the `/app` prefix, the Django app doesn't recognize the route.
**Fix:** This requires an Ingress **Annotation** to rewrite the URL path before forwarding it — covered in full detail in Chapter 31.

## Key Concept Recap
> **Ingress** is how you manage smart, path-based (or host-based) routing of external traffic to multiple internal Services — but it requires an **Ingress Controller** (like the Nginx Ingress Controller) actually running in your cluster to do the real work. Ingress rules alone are just instructions; the Controller is what executes them.

## Final Result
Students now understand:
- Why Ingress is needed for microservices architecture
- How to install the Nginx Ingress Controller on a KIND cluster
- How to write Ingress rules for path-based routing to multiple Services
- How to expose the Ingress Controller and test routing in a browser
- That some backend applications need extra configuration (Annotations) to handle path-based routing correctly

---
**Prerequisite for Chapter 31:** The `/app` routing issue identified here is fixed using Annotations in the next chapter.
