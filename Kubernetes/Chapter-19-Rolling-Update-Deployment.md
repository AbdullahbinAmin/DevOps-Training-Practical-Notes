# Practical 19 — Rolling Update in Deployment

## Objective
Understand and practically demonstrate how Rolling Updates work in a Deployment — updating the container image without causing downtime.

## Architecture / Flow
```text
kubectl set image → Deployment starts updating Pods one at a time
→ Old Pods keep serving traffic while new Pods are being created
→ Once new Pods are healthy, old Pods are terminated → Zero downtime
```

## Prerequisites
- `nginx-deployment` from Chapter 18, scaled to 5 replicas (for a clearer demonstration)

## Step 1 — Scale Back Up to See the Rolling Update Clearly

```bash
kubectl scale deployment nginx-deployment --replicas=5 -n nginx
```

**Verify:**
```bash
kubectl get pods -n nginx
```
**Expected output:** 5 Pods running.

## Step 2 — See More Details About Where Pods Are Running

```bash
kubectl get pods -n nginx -o wide
```

**What this does:** The `-o wide` flag shows extra columns, including which **Worker Node** each Pod is running on (e.g., Pod 1 on worker-1, Pod 2 on worker-2, etc.)

## Step 3 — Find Available Nginx Image Versions

Before updating, you need to know what versions are available.

Search on Docker Hub:
```text
https://hub.docker.com/_/nginx/tags
```

⚠️ **Verification Required:** Always check the official Docker Hub tags page for currently available versions, since new versions are released over time. This practical uses `1.27.3` and `1.27.1` as example versions available at the time of recording.

## Step 4 — Update the Image Using Rolling Update

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.27.3 -n nginx
```

**What this does:**
- `set image` → command used to update the container image of a running Deployment (this triggers a rolling update automatically)
- `deployment/nginx-deployment` → the target Deployment
- `nginx=nginx:1.27.3` → format is `<container-name>=<new-image>:<tag>` — this says: "for the container named `nginx` inside this Deployment, use the image `nginx:1.27.3`"
- `-n nginx` → the namespace

⚠️ **Important:** The container name (`nginx` before the `=`) must exactly match the `name:` field you set under `containers:` in your Deployment YAML — NOT the Deployment's own name.

**Expected output:**
```text
deployment.apps/nginx-deployment image updated
```

### If Something Goes Wrong (Wrong Image Tag)
If you accidentally type an invalid tag (e.g., `1.2 7.3` with a typo/space, or a version that doesn't exist):

```bash
kubectl get pods -n nginx
```
**You might see:**
```text
NAME                                READY   STATUS             RESTARTS   AGE
nginx-deployment-xxxxxxxxx-aaaaa    0/1     ImagePullBackOff   0          10s
```

**Reason:** `ImagePullBackOff` means Kubernetes could not find/pull the image you specified — usually a typo in the version tag.

**Fix:** Correct the image tag and run the command again:
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.27.3 -n nginx
```

**Verify:**
```bash
kubectl get pods -n nginx
```
**Expected output:** All Pods should now show `Running` with the corrected image.

## Step 5 — Observe the Rolling Update in Action

Run this command **right after** triggering an image update, to watch it happen live:

```bash
kubectl get pods -n nginx --watch
```

**What you should observe:**
- NOT all 5 Pods restart at the same time
- Some Pods stay in `Running` state (old version) while others show `ContainerCreating` (new version)
- One by one, old Pods terminate and new Pods become ready
- At every moment during the update, **some Pods are still available to serve traffic** — this is the core benefit of a Deployment's rolling update, versus a plain ReplicaSet which would restart everything at once.

Press `Ctrl + C` to stop watching once all Pods show `Running`.

## Step 6 — Try Another Update (Downgrade Example)

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.26.0 -n nginx
```

**Verify with watch again:**
```bash
kubectl get pods -n nginx --watch
```
**Expected behavior:** Same rolling pattern — some Pods terminate and get recreated with the new image, while others continue running, until all 5 Pods are updated.

## Step 7 — Set Back to the Latest Image

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:latest -n nginx
```

**Verify:**
```bash
kubectl get pods -n nginx
```
**Expected output:** All Pods `Running` with the `latest` image.

## Troubleshooting

### Error: `ImagePullBackOff` or `ErrImagePull`
**Reason:** Incorrect image name or tag (typo), or the tag doesn't exist on Docker Hub.
**Fix:** Double-check the exact tag name on Docker Hub, then rerun `kubectl set image` with the corrected value.

### Error: image updated but nothing seems to change
**Reason:** You may have used the Deployment's name instead of the container's name in the `set image` command.
**Fix:** Confirm the container name from your `deployment.yaml` under `spec.template.spec.containers[].name`, and use that exact name in the command.

## Key Concept Recap
> A **Rolling Update** ensures that when you change something (like the container image) in a Deployment, Kubernetes updates the Pods gradually — a few at a time — rather than all at once. This keeps your application available throughout the entire update process, with **zero downtime**.

## Final Result
Students now understand:
- How to trigger an image update on a running Deployment using `kubectl set image`
- How to watch a Rolling Update happen live using `kubectl get pods --watch`
- Why Deployments are safer than plain ReplicaSets for production applications
- How to debug a failed image update (`ImagePullBackOff`)

---
**Prerequisite for Chapter 20:** None extra — Chapter 20 (ReplicaSets) will reuse the same YAML structure, converted from a Deployment into a plain ReplicaSet, to show the practical difference.
