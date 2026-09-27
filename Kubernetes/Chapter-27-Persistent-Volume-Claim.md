# Practical 27 — Persistent Volume Claim (PVC)

## Objective
"Claim" the Persistent Volume created in Chapter 25 using a Persistent Volume Claim (PVC), then connect that claim to a Deployment's Pod so data actually persists on the host machine.

## Concept: What is a Persistent Volume Claim?

### Analogy
Imagine you see a water bottle (sipper) lying around. It's unclear who owns it. If it's actually yours, you need to **claim** it — pick it up and say "this is mine" — otherwise, anyone else could take it.

Similarly: in Chapter 25, we created a 1 GB Persistent Volume from our host machine. It is sitting there with status `Available` — but nothing is actually using it yet. To connect it to a Pod, you need to **claim** it using a **Persistent Volume Claim (PVC)**.

> **A PVC is a request: "I want to use X amount of storage from an available Persistent Volume."**

## Prerequisites
- Persistent Volume `local-pv` from Chapter 25 already applied, status `Available`
- `nginx` namespace present

## Step 1 — Create the PVC Manifest

⚠️ **Important lesson from this practical:** Both the PV and PVC (and the Deployment that uses them) generally need to be in the **same namespace** for things to bind and work correctly. In Chapter 25, our PV was created without specifying a namespace, so it defaulted to the `default` namespace — but our Deployment lives in the `nginx` namespace. We need to fix this.

### First, delete the old PV and recreate it properly in the `nginx` namespace:

```bash
kubectl delete pv local-pv
```

⚠️ **Verification Required:** PersistentVolumes are actually **cluster-scoped** resources (they technically don't belong to a namespace the way Pods do), but the way this course structures the demo, we ensure the PVC and Deployment are correctly matched to avoid confusion — always double check the namespace consistency between your PVC and the Deployment using it.

Reapply the PV (with the `nginx` namespace context, or confirm it's accessible):
```bash
kubectl apply -f persistent-volume.yaml
```

**Verify it's available again:**
```bash
kubectl get pv
```
**Expected output:** `local-pv` should show status `Available` again.

## Step 2 — Create the PVC Manifest

```bash
vim persistent-volume-claim.yaml
```

Paste this complete YAML:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: local-pvc
  namespace: nginx
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1000Mi
  storageClassName: local-storage
```

**Explanation of every field:**
- `kind: PersistentVolumeClaim` → this manifest is a PVC (a request/claim for storage).
- `apiVersion: v1` → core API version.
- `metadata.name: local-pvc` → name of this claim.
- `metadata.namespace: nginx` → **critical field** — this PVC must be in the same namespace as the Deployment that will use it.
- `spec.accessModes` → must match (or be compatible with) the PV's access mode — here, `ReadWriteOnce`, same as our PV.
- `spec.resources.requests.storage: 1000Mi` → how much storage this claim is requesting — here, requesting the full 1000Mi (1 GB) that our PV has available.
- `spec.storageClassName: local-storage` → must exactly match the `storageClassName` defined in the PV (Chapter 25/26), so Kubernetes knows which type of volume to bind this claim to.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 3 — Apply the PVC

```bash
kubectl apply -f persistent-volume-claim.yaml
```

### If Something Goes Wrong (typo)
If you see:
```text
error: error validating "persistent-volume-claim.yaml": ... unknown field "resource" in ...PersistentVolumeClaimSpec
```
**Reason:** Typo — should be `resources` (plural), not `resource`.
**Fix:** Correct the spelling and reapply:
```bash
kubectl apply -f persistent-volume-claim.yaml
```

**Expected output:**
```text
persistentvolumeclaim/local-pvc created
```

## Step 4 — Verify the Claim Bound Successfully

```bash
kubectl get pvc -n nginx
```
**Expected output:**
```text
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
local-pvc   Bound    local-pv   1000Mi     RWO            local-storage   10s
```

**Field meaning:** `STATUS: Bound` confirms our PVC successfully claimed the 1 GB PV — it went from `Available` (unused) to `Bound` (now in use by this specific claim).

**Double-check the PV's status too:**
```bash
kubectl get pv
```
**Expected output:** `local-pv` should also now show `STATUS: Bound` (instead of `Available`).

## Step 5 — Connect the PVC to a Deployment (So Data Persists)

Now we update our Deployment so its Nginx container's web content folder (`/usr/share/nginx/html`) actually uses this claimed 1 GB volume, instead of the Pod's temporary internal storage.

```bash
kubectl delete -f deployment.yaml
vim deployment.yaml
```

Update the content to look like this:

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
      name: nginx-dep-pod
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          volumeMounts:
            - name: my-volume
              mountPath: /usr/share/nginx/html
      volumes:
        - name: my-volume
          persistentVolumeClaim:
            claimName: local-pvc
```

**What's new compared to the original Deployment YAML:**
- `containers[].volumeMounts` → tells this specific container: "mount a volume named `my-volume` at the path `/usr/share/nginx/html`" (this is the exact folder inside the Nginx container where its website's HTML files live).
- `spec.volumes` (at the Pod level, same indentation as `containers`) → defines what `my-volume` actually IS:
  - `persistentVolumeClaim.claimName: local-pvc` → connects this volume directly to our PVC (`local-pvc`), which in turn is bound to our PV (`local-pv`), which in turn is mapped to the `/mnt/data` folder on the host machine.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 6 — Apply and Debug

```bash
kubectl apply -f deployment.yaml
```

```bash
kubectl get pods -n nginx
```

### If Something Goes Wrong — Pod Stuck in `Pending`
If a Pod stays stuck in `Pending`:
```bash
kubectl describe pod <pod-name> -n nginx
```
**You might see:**
```text
0/4 nodes are available for persistent volume "local-pvc" not found
```

**Reason:** The PV/PVC created earlier may have been in the wrong namespace (e.g., `default` instead of `nginx`).

**Fix:**
1. Delete the incorrectly-namespaced PV and PVC:
```bash
kubectl delete pvc local-pvc
kubectl delete pv local-pv
```
2. Reapply both, making sure both YAML files explicitly declare `namespace: nginx` in their `metadata`:
```bash
kubectl apply -f persistent-volume.yaml
kubectl apply -f persistent-volume-claim.yaml
```
3. Verify both are bound correctly:
```bash
kubectl get pv
kubectl get pvc -n nginx
```
4. Reapply the Deployment:
```bash
kubectl apply -f deployment.yaml
```

**Expected result after fixing:**
```bash
kubectl get pods -n nginx
```
```text
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-xxxxxxxxx-aaaaa    1/1     Running   0          20s
nginx-deployment-xxxxxxxxx-bbbbb    1/1     Running   0          20s
```

## Step 7 — Confirm the Volume Is Actually Mounted (Checking on the Host)

Since our cluster is a KIND cluster, the "Worker Node" is actually a Docker container running on our EC2 host — so to see the actual folder on the host, we need a couple of extra steps.

**Step 1:** Find which Worker Node a Pod is running on:
```bash
kubectl get pods -n nginx -o wide
```
**Expected output:** Shows the Pod along with its assigned Node (e.g., `kind-cluster-worker`).

**Step 2:** Since that "worker node" is really a Docker container, find its container ID:
```bash
docker ps
```
**Expected output:** A list of running containers, including one matching the worker node's name.

**Step 3:** Enter that worker node's container:
```bash
docker exec -it <container-id> bash
```

**Step 4:** Inside that container (which represents the worker node), navigate to the mount path:
```bash
ls /mnt
cd /mnt/data
ls
```
**Expected result:** This folder should exist (since our PV pointed here), confirming the storage path was successfully created and bound. If nothing has been written into the Nginx HTML folder from inside the Pod yet, this folder may appear empty at first — but any file created inside the Pod's `/usr/share/nginx/html` will now show up here, and will **survive even if the Pod is deleted and recreated**.

**Exit back out:**
```bash
exit
```

## Key Concept Recap
> A **Persistent Volume Claim (PVC)** is how a Pod actually "claims" and uses a Persistent Volume. The full data flow is:
> ```text
> Host Machine folder (e.g. /mnt/data)
>   → Persistent Volume (PV) — carves out storage from host
>     → Persistent Volume Claim (PVC) — claims/requests that storage
>       → Deployment's volumeMounts — mounts the claimed storage into a specific folder inside the container
> ```
> This ensures that even if a Pod is deleted and a new one is created, the data in that mounted folder persists, because it's really stored on the host machine, not inside the temporary Pod itself.

## Final Result
Students now understand the complete Persistent Storage flow in Kubernetes:
- Persistent Volume (PV) → carves storage from the host machine
- Storage Class → identifies what type of storage backend is used
- Persistent Volume Claim (PVC) → claims/requests the available PV storage
- Deployment `volumeMounts` + `volumes` → connects the claimed storage into a specific container folder
- How to debug namespace mismatch issues between PV, PVC, and Deployment
- How to verify data is actually persisted on the host machine (by entering the worker node's Docker container)

---
**Prerequisite for Chapter 28:** None extra — Chapter 28 (Services) moves on to networking: exposing this Deployment to the outside world.
