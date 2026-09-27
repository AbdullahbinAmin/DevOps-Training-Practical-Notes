# Practical 24 — Storage (Concept Introduction)

## Objective
Understand the fundamental problem that Kubernetes Storage solves: why data inside a Pod gets lost, and why we need a way to make it persist (survive Pod deletion/recreation).

## Concept: The Data Loss Problem

Recall from earlier chapters: when you create a Pod, that Pod runs on a specific Node (Server). A Pod can be **deleted or destroyed at any time** — this could happen because:
- You manually deleted it
- It crashed and needed to be auto-healed (recreated) by a ReplicaSet/Deployment
- The node it was running on had an issue

### The Problem
Whatever **data** was stored inside that Pod (files, folders, database records, etc.) is stored on that Pod's own temporary filesystem. When the Pod is destroyed, **that data is destroyed along with it.**

### Demonstration (Recap From Earlier Practicals)
```bash
kubectl apply -f deployment.yaml
kubectl get pods -n nginx
```
Suppose this creates 2 Pods. If you delete one Pod directly:
```bash
kubectl delete pod <pod-name> -n nginx
```
**What happens:** The ReplicaSet controller behind the Deployment immediately notices that the actual replica count dropped below the desired count, and automatically creates a **brand new Pod** to replace it (this is the self-healing behavior we discussed in Chapter 16).

**But here's the catch:** This new Pod is a fresh, empty Pod. Any data that existed inside the old, now-deleted Pod is **gone permanently** — it does NOT carry over to the new replacement Pod.

## The Solution: Persisting Data

To prevent this data loss, we need to store the data **outside of the Pod itself** — specifically, tied to the **Host machine** (the actual server/Node), so that even if the Pod is destroyed and recreated, the data still exists on the host and can be reconnected.

This is done using:
1. **Persistent Volume (PV)** — covered in Chapter 25
2. **Persistent Volume Claim (PVC)** — covered in Chapter 27
3. **Storage Classes** — covered in Chapter 26

## Final Result
Students understand the core problem storage solves in Kubernetes: **Pods are temporary/disposable, but data often needs to survive beyond a single Pod's lifetime.** This sets up the need for Persistent Volumes in the following chapters.

---
**Prerequisite for Chapter 25:** This concept (why data is lost when a Pod is deleted) is essential before understanding how Persistent Volumes solve the problem.
