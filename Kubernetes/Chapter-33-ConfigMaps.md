# Practical 33 — ConfigMaps

## Objective
Move plain-text configuration values (like the MySQL database name) out of the StatefulSet manifest into a separate ConfigMap, so the main manifest doesn't need to be edited every time a config value changes.

## Concept: Why Do We Need ConfigMaps?

In Chapter 32, we hardcoded the database name directly inside the StatefulSet:
```yaml
env:
  - name: MYSQL_DATABASE
    value: "devops"
```

### The Problem
If you ever need to change this database name, you'd have to go back into the big StatefulSet file (which also contains the Pod template, volume claim templates, service name, and everything else) and edit it directly. This is risky and inconvenient, especially for a file with many moving parts.

### The Solution
> **A ConfigMap stores configuration data (plain-text key-value pairs) separately from your main workload manifest, so you can update configuration without touching the Deployment/StatefulSet file itself.**

## Prerequisites
- MySQL StatefulSet from Chapter 32

## Step 1 — Create the ConfigMap Manifest

```bash
vim configmap.yaml
```

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: mysql-config
  namespace: mysql
data:
  database: "devops"
```

**Explanation of every field:**
- `kind: ConfigMap` → this manifest describes a ConfigMap resource.
- `apiVersion: v1` → core API version.
- `metadata.name: mysql-config` → the name of this ConfigMap.
- `metadata.namespace: mysql` → must be in the same namespace as the StatefulSet that will use it.
- `data` → a plain key-value map. Here, key `database` has the value `"devops"`.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 2 — Apply the ConfigMap

```bash
kubectl apply -f configmap.yaml
```
**Expected output:**
```text
configmap/mysql-config created
```

**Verify:**
```bash
kubectl get configmap -n mysql
```

## Step 3 — Reference the ConfigMap in the StatefulSet

```bash
vim statefulset.yaml
```

Update the `MYSQL_DATABASE` environment variable to pull its value from the ConfigMap instead of being hardcoded:

```yaml
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "root"
            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: mysql-config
                  key: database
```

**What changed:**
- Instead of `value: "devops"`, we now use `valueFrom.configMapKeyRef`, which tells Kubernetes: "get this environment variable's value from a ConfigMap."
- `configMapKeyRef.name: mysql-config` → which ConfigMap to read from.
- `configMapKeyRef.key: database` → which specific key inside that ConfigMap to use.

⚠️ **Common Error — Capitalization:** The field is `configMapKeyRef` — note the capital `R` in `Ref`. Writing it as `configMapKeyref` (lowercase r) will cause a strict decoding error:
```text
error: error validating "statefulset.yaml": ... strict decoding error: unknown field ...
```
**Fix:** Correct the capitalization to exactly match `configMapKeyRef`, as shown in the official Kubernetes documentation (search "kubernetes configmap environment variable" to find the reference example).

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 4 — Delete and Recreate the StatefulSet

Since we're changing the Pod template significantly, it's cleanest to delete and recreate:

```bash
kubectl delete statefulset mysql-statefulset -n mysql
kubectl apply -f statefulset.yaml
```

**Expected output:**
```text
statefulset.apps/mysql-statefulset created
```

## Step 5 — Verify

```bash
kubectl get pods -n mysql
```
**Expected output:** All 3 Pods should be `Running` with no errors, confirming the value was successfully pulled from the ConfigMap.

**Double-check the database was created with the ConfigMap's value:**
```bash
kubectl exec -it mysql-statefulset-0 -n mysql -- bash
mysql -u root -p
```
(Enter password: `root`)
```sql
SHOW DATABASES;
```
**Expected output:** The `devops` database should still be listed — confirming the ConfigMap value was read correctly.

```sql
exit
```
```bash
exit
```

## Key Concept Recap
> A **ConfigMap** separates plain-text configuration data from your workload manifests. Any environment variable (or, as covered in official docs, even entire config files) can be sourced from a ConfigMap using `valueFrom.configMapKeyRef`, making your configuration reusable and easy to update independently.

## Final Result
Students now understand:
- Why separating configuration from workload manifests is a good practice
- How to create a ConfigMap with key-value data
- How to reference a ConfigMap's value in an environment variable using `configMapKeyRef`
- How to debug capitalization-related YAML errors

---
**Prerequisite for Chapter 34:** ConfigMaps store **plain, unencoded** data — for sensitive values like passwords, Secrets (covered next) should be used instead.
