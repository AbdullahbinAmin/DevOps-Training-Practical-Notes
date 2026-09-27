# Practical 25 — Persistent Volume (PV)

## Objective
Understand and create a Persistent Volume (PV) — a chunk of storage taken from the Host Machine, made available for Pods to use.

## Concept: What is a Persistent Volume?

Recall from Chapter 8: our EC2 instance (Host Machine) was configured with **30 GB of storage**.

A **Persistent Volume (PV)** is how you **carve out (allocate) a specific piece** of that host machine's storage for Kubernetes to use — for example, taking 1 GB out of your 30 GB total.

### Analogy
Think of your Host Machine's 30 GB storage as a big block of space. A Persistent Volume is you saying: "From this 30 GB, set aside exactly 1 GB, and make it available for use by Kubernetes."

## Prerequisites
- KIND cluster running (Host Machine = your EC2 instance with 30 GB storage from Chapter 8)
- `nginx` namespace present

## Step 1 — Create the Persistent Volume Manifest

```bash
vim persistent-volume.yaml
```

Paste this complete YAML:

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: local-pv
  labels:
    app: local
spec:
  capacity:
    storage: 1000Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: /mnt/data
    type: DirectoryOrCreate
```

**Explanation of every field:**
- `kind: PersistentVolume` → this manifest describes a Persistent Volume resource.
- `apiVersion: v1` → the core API version.
- `metadata.name: local-pv` → the name of this Persistent Volume.
- `metadata.labels` → labels help identify this resource (e.g., `app: local`).
- `spec.capacity.storage: 1000Mi` → how much storage this PV should have (~1 GB; `Mi` = mebibytes, so `1000Mi` is approximately 1 GB — the exact size depends on your host machine's available storage).
- `spec.accessModes` → controls how this volume can be accessed:
  - `ReadWriteOnce` → the volume can be mounted as read-write, but only by **one node at a time**. (Other options include `ReadOnlyMany` and `ReadWriteMany`, used for different scenarios.)
- `spec.persistentVolumeReclaimPolicy: Retain` → **important field.** This controls what happens to the actual data on the host machine if this PV is deleted:
  - `Retain` → even if you delete the PV resource in Kubernetes, the actual data on the host machine's `/mnt/data` folder stays intact (not automatically erased).
  - (Other options exist, like `Delete`, but `Retain` is safer for learning and for important data.)
- `spec.storageClassName: local-storage` → links this PV to a specific **Storage Class** (covered in Chapter 26) — this name must match the Storage Class name exactly.
- `spec.hostPath` → tells Kubernetes this volume maps to a folder on the actual host machine:
  - `path: /mnt/data` → the exact folder path on the host machine where this 1 GB will be allocated from.
  - `type: DirectoryOrCreate` → if this folder doesn't already exist, create it automatically.

⚠️ **Note on capitalization:** the `path` field's `P` inside `hostPath` is capitalized as part of the sub-field name convention — this refers to a required sub-field, and getting the exact field name right matters, since YAML validation is strict.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 2 — Apply the Persistent Volume

```bash
kubectl apply -f persistent-volume.yaml
```

**Expected output:**
```text
persistentvolume/local-pv created
```

## Step 3 — Verify

```bash
kubectl get pv
```
**Expected output:**
```text
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      STORAGECLASS    AGE
local-pv   1000Mi     RWO            Retain           Available   local-storage   10s
```

**Field meaning:**
- `CAPACITY: 1000Mi` → confirms this is a ~1 GB volume, carved out from your host machine's total 30 GB.
- `ACCESS MODES: RWO` → shorthand for `ReadWriteOnce`.
- `RECLAIM POLICY: Retain` → confirms data will be retained even if the PV is deleted.
- `STATUS: Available` → this means the 1 GB volume has been successfully allocated from the host and is now **available**, but **not yet claimed/used by anything**.

## Key Concept Recap
> A **Persistent Volume (PV)** is storage carved out from the Host Machine and made available for use inside the Kubernetes cluster. Creating a PV alone does NOT connect it to any Pod yet — it is just sitting there, "Available." To actually use it, you need a **Persistent Volume Claim** (Chapter 27), which "claims" this available volume for a specific Pod to use.

## Final Result
Students now have a working Persistent Volume (`local-pv`), 1 GB in size, status `Available`, ready to be claimed in the next chapters.

---
**Prerequisite for Chapter 26:** Understanding that this PV references a `storageClassName: local-storage`, which needs to correspond to an actual Storage Class concept, explained next.
