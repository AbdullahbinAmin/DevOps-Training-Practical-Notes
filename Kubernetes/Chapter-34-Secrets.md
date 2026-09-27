# Practical 34 — Secrets

## Objective
Move the MySQL root password out of the StatefulSet manifest into a Kubernetes Secret, using Base64 encoding, and understand exactly what Base64 encoding does (and does NOT) protect.

## Concept: What is a Secret?

Just like a ConfigMap stores plain-text configuration, a **Secret** stores sensitive data (like passwords, tokens, API keys) — but the values are stored **Base64 encoded** rather than as plain text.

> **A Secret is used the same way as a ConfigMap, but its data is Base64 encoded before being stored.**

## ⚠️ Important: What Base64 Encoding Actually Does

This is a commonly misunderstood point, so let's be very clear:

**Base64 encoding is NOT strong security/encryption.** It is trivially reversible. Anyone with access to the encoded value can decode it back to plain text in seconds.

### Demonstration of Why This Matters
If you encode the word `root`:
```bash
echo -n "root" | base64
```
**Output (example):**
```text
cm9vdA==
```

Now decode it back:
```bash
echo "cm9vdA==" | base64 --decode
```
**Output:**
```text
root
```

> As you can see, decoding a Base64 value takes just 2 seconds — Base64 provides **zero real security** on its own.

### So Why Does Kubernetes Use Base64 for Secrets At All?

The real purpose of Base64 encoding in Secrets is **not security** — it's about **data format compatibility**:

> When a Secret is created, Kubernetes needs to store it as a **binary file** internally (something a computer understands in 0s and 1s). Base64 encoding is simply a standard way to safely represent that binary data as text inside a YAML file. Once turned into this binary format, a human cannot simply "read" the value directly out of the raw etcd storage the way they could with plain text — but this is about storage/format handling, not encryption strength.

⚠️ **Real Security Note:** For actual production-grade secret security, Kubernetes Secrets should be combined with additional measures like **encryption at rest** (encrypting etcd itself), **RBAC** (restricting who can even read Secret objects — covered in a later chapter), and ideally an external secrets manager (like AWS Secrets Manager or HashiCorp Vault).

## Prerequisites
- MySQL StatefulSet from Chapter 32/33

## Step 1 — Generate the Base64 Encoded Value

```bash
echo -n "root" | base64
```
**Expected output (example):**
```text
cm9vdA==
```

⚠️ **Note the `-n` flag:** `echo -n` avoids adding a trailing newline character, which would change the encoded result. Always use `-n` when encoding values for Secrets.

Copy this output value — you'll need it in the next step.

## Step 2 — Create the Secret Manifest

```bash
vim secret.yaml
```

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: mysql-secret
  namespace: mysql
data:
  mysql-root-password: cm9vdA==
```

**Explanation of every field:**
- `kind: Secret` → this manifest describes a Secret resource (note: `Secret`, singular — not `Secrets`).
- `apiVersion: v1` → core API version.
- `metadata.name: mysql-secret` → the name of this Secret.
- `metadata.namespace: mysql` → same namespace as the StatefulSet using it.
- `data.mysql-root-password` → the key name, with its value being the **Base64-encoded** string generated in Step 1 (not the plain text `root`).

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 3 — Apply the Secret

```bash
kubectl apply -f secret.yaml
```
**Expected output:**
```text
secret/mysql-secret created
```

## Step 4 — Reference the Secret in the StatefulSet

```bash
vim statefulset.yaml
```

Update the `MYSQL_ROOT_PASSWORD` environment variable to pull from the Secret instead of being hardcoded:

```yaml
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: mysql-root-password
            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: mysql-config
                  key: database
```

**What changed:**
- `valueFrom.secretKeyRef` → same pattern as `configMapKeyRef` from Chapter 33, but for Secrets.
- `secretKeyRef.name: mysql-secret` → which Secret to reference.
- `secretKeyRef.key: mysql-root-password` → which specific key inside that Secret to use.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 5 — Delete and Recreate the StatefulSet

```bash
kubectl delete statefulset mysql-statefulset -n mysql
kubectl apply -f statefulset.yaml
```
**Expected output:**
```text
statefulset.apps/mysql-statefulset created
```

## Step 6 — Verify Everything Still Works

```bash
kubectl get pods -n mysql
```
**Expected output:** All 3 Pods `Running`.

**Confirm the app never had the password exposed directly, yet still works:**
```bash
kubectl exec -it mysql-statefulset-1 -n mysql -- bash
```
Inside the container:
```bash
mysql -u root -p
```
When prompted for the password, type the ORIGINAL plain-text password:
```text
root
```
**Expected result:** You should log in successfully.

```sql
SHOW DATABASES;
```
**Expected output:** The `devops` database should still be listed (still coming correctly from the ConfigMap, from Chapter 33).

```sql
exit
```
```bash
exit
```

> Notice: at no point in the StatefulSet manifest did we write the word `root` (the password) directly, nor `devops` (the database name) directly — both values were referenced indirectly from a Secret and a ConfigMap, respectively — yet the application still works exactly the same.

## Clean Up (Optional)

```bash
kubectl delete namespace mysql
```
**What this does:** Removes everything related to this MySQL practical — the StatefulSet, Secret, ConfigMap, Service, and Persistent Volume Claims — all at once. This demonstrates the convenience of using namespaces to group related resources (as discussed in Chapter 14).

## Key Concept Recap
> A **Secret** works exactly like a ConfigMap for referencing values in your workload manifests, but its data is stored Base64 encoded. Base64 encoding is about **binary-safe storage format**, NOT real encryption — for true production security, Secrets should be combined with encryption at rest, RBAC restrictions, and ideally an external secrets manager.

## Final Result
Students now understand:
- The difference between a ConfigMap (plain text) and a Secret (Base64 encoded)
- How to correctly Base64 encode a value using `echo -n | base64`
- Why Base64 encoding does NOT provide real security on its own
- How to reference a Secret's value in a container using `secretKeyRef`
- How to combine ConfigMaps and Secrets together in a single StatefulSet, keeping the main manifest clean of any sensitive or environment-specific hardcoded values

---
**Prerequisite for Chapter 35:** None extra — Chapter 35 moves into a new topic area, Resource Quotas and Limits, which controls how much CPU/memory each Pod can consume.
