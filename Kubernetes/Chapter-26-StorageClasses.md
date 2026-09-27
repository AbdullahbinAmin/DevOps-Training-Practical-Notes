# Practical 26 — Storage Classes

## Objective
Understand what a Storage Class is and how it relates to the `storageClassName` field used in the Persistent Volume from Chapter 25.

## Concept: What is a Storage Class?

A **Storage Class** simply describes **what kind/method of storage** is being used to back a Persistent Volume.

### Real-World Analogy
Think about the different ways you personally save data:
- A Hard Disk
- A Pen Drive (USB stick)
- A DVD

Each of these is a different "class" or "type" of storage. Similarly, in Kubernetes, storage can come from different sources/types:

- **Local Storage** — storage that lives directly on the Host Machine (your EC2 instance's own disk). This is what we used in Chapter 25.
- **Cloud Storage** — storage provided by a cloud provider (e.g., AWS EBS, Google Persistent Disk).
- **Network Storage** — storage accessed over a network (e.g., NFS).

## How This Connects to Our Persistent Volume

In Chapter 25, our PV manifest had this field:
```yaml
spec:
  storageClassName: local-storage
```

This `local-storage` name refers to the fact that we are using the **Host Machine's own local storage** — since our `hostPath` pointed to `/mnt/data` directly on the EC2 instance's disk.

## Verifying This in Your Cluster

Check your existing namespaces (recall from Chapter 14):
```bash
kubectl get ns
```
You may remember seeing a namespace called `local-path-storage` in the default namespace list. This is directly related — KIND clusters come with a built-in local storage provisioner that binds storage to the underlying Docker container (acting as your Node), which is what makes `local-storage` work as a storage class name in this setup.

You can also check actual defined Storage Class objects in your cluster:
```bash
kubectl get storageclass
```
or short form:
```bash
kubectl get sc
```
**Expected output (in a KIND cluster):** You should see a default storage class (often named `standard`), which KIND uses internally for dynamic local storage provisioning.

⚠️ **Note:** In this course's example, we manually defined `storageClassName: local-storage` as a **label/reference name** inside our PV and PVC manifests to logically connect them — this is a simplified, educational approach to show the underlying concept. In more advanced/production setups (especially in the cloud), Storage Classes are formal Kubernetes objects (created via their own YAML with `kind: StorageClass`) that enable **dynamic provisioning** (automatically creating storage on demand, instead of manually pre-creating a PV).

## Key Concept Recap
> A **Storage Class** identifies WHAT TYPE of storage backend is being used (local disk, cloud disk, network storage, etc.). The `storageClassName` field is how a Persistent Volume Claim (next chapter) knows which "type" of storage it is requesting, and how it matches up with an available Persistent Volume.

## Final Result
Students understand that Storage Classes represent different storage backend types (local, cloud, network), and that our example uses `local-storage` to represent the Host Machine's own local disk — directly linking back to the `hostPath`-based Persistent Volume created in Chapter 25.

---
**Prerequisite for Chapter 27:** This `local-storage` class name must match exactly between the Persistent Volume (Chapter 25) and the Persistent Volume Claim (next chapter) for the claim to successfully bind.
