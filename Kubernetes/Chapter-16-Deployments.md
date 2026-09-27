# Practical 16 — Deployments

## Objective
Understand why a Pod alone is not enough for real applications, and learn what a Deployment is and why it's needed.

## Architecture / Flow
```text
Deployment (manages replicas) → creates/controls → ReplicaSet → creates/controls → Pods (containers)
```

## Concept: Why Do We Need a Deployment?

We already know how to create a Pod directly. But creating a single Pod has limits.

### The Problem
Imagine you're building something like Amazon.com. On a big sale day (like "Big Billion Days"), traffic increases massively. One single Pod cannot handle that much traffic — it would crash.

You don't need just 1, 2, or 3 individual Pods created manually. You need a **group of Pod replicas** (identical copies) that can:
- Handle heavy traffic together
- Automatically replace themselves if one crashes
- Scale up or down as needed

This is exactly what a **Deployment** gives you.

## What is a Deployment?

A Deployment is a Kubernetes resource that manages a **desired number of identical Pod replicas**, and adds extra capabilities like:
- Auto-healing (if a Pod crashes, a new one is created automatically)
- Scaling (increase/decrease the number of replicas easily)
- Rolling updates (updating your application without downtime — covered in Chapter 19)

## Basic Deployment YAML Structure

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
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

**Explanation of fields:**
- `kind: Deployment` → tells Kubernetes this manifest describes a Deployment resource.
- `apiVersion: apps/v1` → **important difference from Pods!** Pods use `v1`, but Deployments use `apps/v1`. Different resource types use different API version groups. If you're unsure which API version a resource uses, search "kubernetes <resource-name>" (e.g., "kubernetes deployment") and check the official documentation.
- `metadata.name` → name of the Deployment.
- `metadata.namespace` → which namespace this Deployment belongs to.
- `spec.replicas: 2` → how many identical Pod copies you want (covered in more detail in Chapter 18 — Labels and Selectors).
- `spec.selector` / `spec.template` → covered fully in the next chapter.

## Key Concept Recap
> A Pod by itself has no auto-healing or scaling ability. A **Deployment** wraps around Pods to give you replication, self-healing, scaling, and rolling updates — capabilities essential for real production applications.

## Final Result
Students understand the core motivation for Deployments: handling scale and reliability, which a single Pod cannot do on its own.

---
**Prerequisite for Chapter 17:** Understanding this motivation is required before comparing Deployments with ReplicaSets and StatefulSets.
