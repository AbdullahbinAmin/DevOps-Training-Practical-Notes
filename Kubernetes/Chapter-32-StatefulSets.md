# Practical 32 — StatefulSets (Hands-on with MySQL)

## Objective
Build a real StatefulSet for a MySQL database, understand why databases need StatefulSets instead of Deployments, and see how Kubernetes maintains a Pod's identity even after it's deleted and recreated.

## Concept: Stateless vs Stateful Applications

### Stateless Applications
Applications like Django, Flask, or Spring Boot (typical web/API applications) are usually **stateless**. This means:
- If the app doesn't perfectly "remember" every past interaction, it's usually fine
- Users can connect and disconnect anytime, and it doesn't break anything critical
- These use **Deployments, ReplicaSets, DaemonSets**, etc.

### Stateful Applications
Applications like **MySQL, MongoDB** (databases) are **stateful**. This means:
- The data must always be consistently stored (persisted)
- Read/write transactions must happen correctly and reliably, every time
- You **cannot** "play around" with the state of a database the way you might with a stateless web app

> **For stateful applications, you use StatefulSets instead of Deployments.**

## Concept: What Makes a StatefulSet Different?

A StatefulSet is similar to a Deployment/ReplicaSet in that it maintains a number of replicas — but with one key difference:

> **Each Pod created by a StatefulSet gets a stable, numbered identity (Pod-0, Pod-1, Pod-2, Pod-3...) which is preserved even if that specific Pod is deleted and recreated.**

### Example
If you create 5 MySQL replicas using a StatefulSet:
- Pod-0, Pod-1, Pod-2, Pod-3, Pod-4 are created
- If Pod-3 is deleted, Kubernetes creates a **new Pod that is again named Pod-3** (not a random new name)
- That new Pod-3 also carries forward the same state (storage, configuration) as before

Compare this to a Deployment: if you delete a Pod, the replacement Pod gets a completely **random new name**.

## Prerequisites
- KIND cluster running
- Understanding of Persistent Volumes/Claims (Chapters 25–27) — a StatefulSet's storage is defined inline, combined into the same manifest.

## Step 1 — Create the Namespace

```bash
mkdir mysql
cd mysql
vim namespace.yaml
```

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: mysql
```

```bash
kubectl apply -f namespace.yaml
```
**Expected output:**
```text
namespace/mysql created
```

## Step 2 — Write the StatefulSet Manifest

```bash
vim statefulset.yaml
```

We'll build this step by step, and show the exact errors you're likely to encounter (this is intentional — learning to debug is a key skill). Here is the **final, correct, working version**:

```yaml
kind: StatefulSet
apiVersion: apps/v1
metadata:
  name: mysql-statefulset
  namespace: mysql
spec:
  serviceName: mysql-service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "root"
            - name: MYSQL_DATABASE
              value: "devops"
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

**Explanation of every field:**
- `kind: StatefulSet` → this manifest describes a StatefulSet resource.
- `apiVersion: apps/v1` → same API group as Deployments (StatefulSets are also a "workload" type).
- `metadata.name` / `metadata.namespace` → the name (`mysql-statefulset`) and namespace (`mysql`) for this resource.
- `spec.serviceName: mysql-service` → **critical field, unique to StatefulSets.** This links the StatefulSet to a specific Service (created in Step 4), which is required for StatefulSets to work correctly, since each Pod needs a stable network identity.
- `spec.replicas: 3` → we want 3 numbered MySQL Pods.
- `spec.selector.matchLabels` / `template.metadata.labels` → same Labels-and-Selectors pattern from Chapter 18 (`app: mysql` on both sides).
- `template.spec.containers` → same container structure as always:
  - `image: mysql:8.0` → using MySQL version 8.
  - `ports.containerPort: 3306` → MySQL's default port.
  - `env` → MySQL containers **require** certain environment variables to start correctly:
    - `MYSQL_ROOT_PASSWORD` → sets the root user's password (here, `"root"` — use a stronger, secret-managed value in real production, as shown in Chapter 34).
    - `MYSQL_DATABASE` → automatically creates a database with this name (`"devops"`) when the container starts.
  - `volumeMounts` → mounts a volume named `mysql-data` at `/var/lib/mysql`, which is the folder where MySQL actually stores its database files.
- `spec.volumeClaimTemplates` → **this is the special/unique part of a StatefulSet.** Instead of referencing a separately-created PVC (like we did with Deployments in Chapter 27), a StatefulSet defines a **template** for creating a Persistent Volume Claim **automatically for each replica**. This means Pod-0, Pod-1, and Pod-2 will each get their own separate, dedicated storage claim.
  - `metadata.name: mysql-data` → must match the `volumeMounts[].name` above, connecting the two.
  - `spec.accessModes` / `resources.requests.storage` → same as a normal PVC (Chapter 27) — here requesting 1Gi per replica.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 3 — Apply and Debug (Learning From Real Errors)

```bash
kubectl apply -f statefulset.yaml
```

This section walks through the **exact common errors** you're likely to hit while building a StatefulSet manifest like this:

#### Error 1 — `templates` instead of `template`
```text
error: error validating "statefulset.yaml": ... strict decoding error: unknown field "spec.templates"
```
**Reason:** Typo — it's `template` (singular), not `templates`.
**Fix:** Correct the field name.

#### Error 2 — `caRef` capitalization (`ref` vs `Ref`)
When referencing things like `configMapKeyRef` or `secretKeyRef` (used in later chapters), field names are **case-sensitive**. A lowercase `ref` where `Ref` (capital R) is required will cause a decoding error.
**Fix:** Always match the exact capitalization shown in the official Kubernetes API documentation.

#### Error 3 — Wrong Indentation for `volumeClaimTemplates`
```text
error: error validating "statefulset.yaml": ... unknown field "volumeClaimTemplates" in ...PodSpec
```
**Reason:** `volumeClaimTemplates` was accidentally nested INSIDE `template.spec` (the Pod's spec) instead of being at the **top-level `spec`** of the StatefulSet itself (same indentation level as `selector` and `template`).
**Fix:** Move `volumeClaimTemplates` so it's a direct child of the StatefulSet's `spec`, NOT inside `template.spec`. Compare carefully against the final YAML shown in Step 2 above — correct indentation is critical in YAML.

⚠️ **Learning Note:** As shown throughout this course, you don't need to memorize the exact YAML structure. Even official certification exams (like the CKA — Certified Kubernetes Administrator) allow you to reference the official Kubernetes documentation during the exam. Learn to read error messages carefully — they tell you exactly which field and location has a problem.

**Once fixed, expected final output:**
```text
statefulset.apps/mysql-statefulset created
```

## Step 4 — Create the Headless Service (Required for StatefulSets)

```bash
vim service.yaml
```

```yaml
kind: Service
apiVersion: v1
metadata:
  name: mysql-service
  namespace: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
      protocol: TCP
```

**Explanation of the key field:**
- `spec.clusterIP: None` → **this is what makes it a "Headless Service."** A Headless Service does NOT get a normal cluster IP address, and is NOT meant to be exposed externally. Its purpose is purely internal: it gives each Pod in the StatefulSet a **stable DNS identity**, so other Pods can reliably connect to a specific replica by name (e.g., `mysql-statefulset-0.mysql-service`).

Apply it:
```bash
kubectl apply -f service.yaml
```
**Expected output:**
```text
service/mysql-service created
```

## Step 5 — Verify the StatefulSet

```bash
kubectl get pods -n mysql
```

**Expected output — notice the naming pattern, unlike Deployments:**
```text
NAME                    READY   STATUS    RESTARTS   AGE
mysql-statefulset-0     1/1     Running   0          40s
mysql-statefulset-1     1/1     Running   0          30s
mysql-statefulset-2     1/1     Running   0          20s
```

**Important observation:** Notice the Pods are created **one at a time, in strict order** (0, then 1, then 2) — this is different from Deployments, where all replicas are typically created in parallel.

You can watch this happen live:
```bash
kubectl get pods -n mysql --watch
```

## Step 6 — Enter the Pod and Verify MySQL

```bash
kubectl exec -it mysql-statefulset-0 -n mysql -- bash
```

Inside the container:
```bash
mysql -u root -p
```
Enter the password when prompted:
```text
root
```

Once inside MySQL:
```sql
SHOW DATABASES;
```
**Expected output:** You should see a database named `devops`, matching the `MYSQL_DATABASE` environment variable we set.

**Exit MySQL and the container:**
```sql
exit
```
```bash
exit
```

## Step 7 — Demonstrate the StatefulSet's Special Identity Behavior

Delete one specific Pod:
```bash
kubectl delete pod mysql-statefulset-0 -n mysql
```
**Expected output:**
```text
pod "mysql-statefulset-0" deleted
```

**Verify it comes back with the EXACT SAME name:**
```bash
kubectl get pods -n mysql
```
**Expected output:**
```text
NAME                    READY   STATUS    RESTARTS   AGE
mysql-statefulset-0     1/1     Running   0          8s
mysql-statefulset-1     1/1     Running   0          5m
mysql-statefulset-2     1/1     Running   0          5m
```

> This is **the special behavior of StatefulSets**: the replacement Pod is named `mysql-statefulset-0` again (not a random name), and it's tightly coupled with the same volume claim, environment variables, and service — so its state is automatically reconnected.

## Key Concept Recap

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod naming | Random suffix each time | Fixed, numbered (`-0`, `-1`, `-2`...) |
| Pod identity after delete/recreate | New random name | Same exact name preserved |
| Best for | Stateless apps (web servers, APIs) | Stateful apps (databases) |
| Rolling updates | Yes | Limited |
| Storage | Manually created/referenced PVC | Auto-generated per-Pod via `volumeClaimTemplates` |

## Final Result
Students now understand:
- The difference between stateless and stateful applications
- How to write a full StatefulSet manifest for MySQL, including `volumeClaimTemplates`
- Why a Headless Service (`clusterIP: None`) is required for StatefulSets
- How to debug common StatefulSet YAML structure errors
- The special "same identity after recreation" behavior that makes StatefulSets unique

---
**Prerequisite for Chapter 33:** This MySQL StatefulSet will be reused in Chapter 33 (ConfigMaps) and Chapter 34 (Secrets) to move its environment variables out of the main manifest.
