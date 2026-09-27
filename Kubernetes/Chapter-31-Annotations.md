# Practical 31 — Annotations (Fixing Ingress Path Routing)

## Objective
Fix the `/app` routing issue from Chapter 30 using an Ingress **Annotation** that rewrites the URL path before forwarding the request to the backend Service.

## Concept: What is an Annotation?

> **An Annotation is extra, non-identifying information attached to a Kubernetes resource's metadata** — used to configure extra behavior for tools (like an Ingress Controller), rather than to identify/select the resource (that's what Labels are for).

### The Specific Problem (Recap from Chapter 30)
When a request comes in at `http://<IP>:8080/app`, the Ingress Controller forwards it to `notes-app-service` — but it forwards the **full path**, including `/app`. The Django application behind that Service expects requests at just `/` (the root path), not `/app`, so it doesn't recognize the route and returns "Page Not Found."

### The Solution
We need to tell the Nginx Ingress Controller: **"Before forwarding this request, rewrite/strip the path so the backend sees `/` instead of `/app`."** This is done using a specific annotation:

```text
nginx.ingress.kubernetes.io/rewrite-target: /
```

## Prerequisites
- Ingress `nginx-notes-ingress` from Chapter 30 already applied

## Step 1 — Edit the Ingress Manifest to Add the Annotation

```bash
vim ingress.yaml
```

Update the content to add an `annotations` field under `metadata`:

```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: nginx-notes-ingress
  namespace: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
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

**Explanation of the new field:**
- `metadata.annotations` → a key-value map (just like `labels`, but with a different purpose) where you can attach extra configuration instructions for tools that understand them.
- `nginx.ingress.kubernetes.io/rewrite-target: /` → this specific annotation is understood by the **Nginx Ingress Controller**. It tells the controller: "whatever path prefix matched this rule (`/nginx` or `/app`), strip it and rewrite the request path to `/` before sending it to the backend Service."

⚠️ **Note:** Different Ingress Controllers (Nginx, Traefik, HAProxy, etc.) each support their own specific set of annotations, with their own naming conventions (e.g., `nginx.ingress.kubernetes.io/...`). Always check that specific controller's official documentation to see what annotations are available — things like custom timeouts, SSL settings, and more can all be configured this way. The **rewrite-target** annotation is the most commonly needed one for basic path-based routing.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 2 — Apply the Updated Ingress

```bash
kubectl apply -f ingress.yaml
```

**Expected output:**
```text
ingress.networking.k8s.io/nginx-notes-ingress configured
```

## Step 3 — Restart Port Forwarding (If Needed)

```bash
kubectl port-forward service/ingress-nginx-controller -n ingress-nginx 8080:80 --address 0.0.0.0
```

## Step 4 — Test Both Routes Again

Open:
```text
http://<YOUR_EC2_PUBLIC_IP>:8080/nginx
```
**Expected result:** "Welcome to nginx!" page loads correctly.

Open:
```text
http://<YOUR_EC2_PUBLIC_IP>:8080/app
```
**Expected result:** The Django Notes App now loads correctly — the annotation fixed the path issue by rewriting `/app` to `/` before forwarding.

## Key Concept Recap
> **Annotations** let you attach extra configuration metadata to a Kubernetes resource that specific tools (like an Ingress Controller) read and act on. Unlike Labels (used for selecting/grouping resources), Annotations are for passing extra instructions/settings. You can add as many services as you like to your Ingress rules — just keep adding new `path` blocks under `spec.rules[].http.paths`, and they will all route correctly.

## Final Result
Students now understand:
- What an Annotation is, and how it differs from a Label
- How the `rewrite-target` annotation solves a common real-world Ingress routing problem
- How to test and confirm multiple services are correctly routed through one single Ingress Controller
- That debugging real errors (like the "Page Not Found" issue) is often the best way to deeply understand how Ingress actually works

---
**Prerequisite for Chapter 32:** None extra — Chapter 32 moves into a new topic, StatefulSets, using a MySQL example.
