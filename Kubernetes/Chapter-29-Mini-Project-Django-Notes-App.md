# Practical 29 — Mini Project: Deploying the Django Notes App on Kubernetes

## Objective
Apply everything learned so far (Namespaces, Deployments, Services) to deploy a real, complete application — a Django-based Notes App — from source code, all the way to a live URL accessible in a browser.

## Architecture / Flow
```text
GitHub Repo (source code) → Docker Build → Docker Hub (push image)
→ Kubernetes Namespace → Deployment (pulls image from Docker Hub) → Service → Port Forward → Browser
```

## Prerequisites
- Docker installed and working on your EC2 instance
- A Docker Hub account (free to create at https://hub.docker.com)
- Git installed
- KIND cluster running
- `kubectl` configured

## Pre-Check
```bash
docker --version
git --version
kubectl version --client
```
**Expected result:** All three should print version info with no errors.

## Step 1 — Clone the Application Source Code

This practical uses an example repository (`django-notes-app`) with a `dev` branch containing a ready-to-use Dockerfile.

⚠️ **Verification Required:** Replace the repository URL below with the actual project repository you are using, and confirm which branch contains the Dockerfile.

```bash
git clone <YOUR_REPOSITORY_URL>
cd django-notes-app
git checkout dev
```

**What this does:**
- Clones the source code repository to your local machine/EC2 instance.
- Switches to the `dev` branch, which contains the application code and Dockerfile needed for this practical.

## Step 2 — Build the Docker Image

```bash
docker build -t notes-app-k8s .
```

**What this does:**
- `docker build` → builds a Docker image from the `Dockerfile` in the current directory (`.`)
- `-t notes-app-k8s` → tags (names) the image `notes-app-k8s`

**Expected output:** A series of build steps (installing Python, dependencies, copying code, etc.), ending with:
```text
Successfully tagged notes-app-k8s:latest
```

⚠️ **Note:** This step may take a few minutes, especially if the base image (e.g., Python 3.9) needs to be downloaded first.

**Verify the image was created:**
```bash
docker images
```
**Expected output:** `notes-app-k8s` should be listed.

## Step 3 — Log In to Docker Hub

Since your local Docker image only exists on your machine, and Kubernetes needs to **pull** it from somewhere accessible, you must push it to Docker Hub (a public image registry).

**Step 1:** Go to https://hub.docker.com and log in (or sign up if you don't have an account).

**Step 2:** Create an **Access Token** (safer than using your account password directly):
- Go to **Account Settings → Security → Access Tokens**
- Click **Generate New Token**
- Name it something like `k8s-in-one-shot`
- Permissions: **Read, Write, Delete**
- Click **Generate**, then copy the token shown (you won't be able to see it again).

**Step 3:** Log in from your terminal:
```bash
docker login -u <YOUR_DOCKERHUB_USERNAME>
```
**What this does:** Prompts you for a password — paste your **Access Token** here (not your actual account password).

**Expected output:**
```text
Login Succeeded
```

## Step 4 — Tag the Image for Docker Hub

```bash
docker image tag notes-app-k8s:latest <YOUR_DOCKERHUB_USERNAME>/notes-app-k8s:latest
```

**What this does:** Renames/tags your local image with your Docker Hub username as a prefix, which is required for Docker to know where to push it. Format: `<dockerhub-username>/<image-name>:<tag>`.

**Verify:**
```bash
docker images
```
**Expected output:** A new image entry like `<YOUR_DOCKERHUB_USERNAME>/notes-app-k8s` should now be listed alongside the original.

## Step 5 — Push the Image to Docker Hub

```bash
docker push <YOUR_DOCKERHUB_USERNAME>/notes-app-k8s:latest
```

**What this does:** Uploads your image to your Docker Hub account, making it available for Kubernetes to pull from.

⚠️ **Note:** By default, this image will be **public** (unless you set the repository to private on Docker Hub, which then requires extra Kubernetes secret configuration to pull — not covered in this basic practical).

**Expected output:** A series of "Pushed" layers, ending with something like:
```text
latest: digest: sha256:xxxxxxxx size: xxxx
```

**Verify on Docker Hub:** Go to https://hub.docker.com/r/<YOUR_DOCKERHUB_USERNAME>/notes-app-k8s and confirm the image is listed there.

## Step 6 — Create a Folder for Kubernetes Manifests

```bash
mkdir k8s
cd k8s
```

## Step 7 — Create the Namespace

```bash
vim namespace.yaml
```

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: notes-app
```

**What this does:** Same pattern from Chapter 14 — creates a dedicated namespace called `notes-app` to keep this project's resources isolated.

Apply it:
```bash
kubectl apply -f namespace.yaml
```
**Expected output:**
```text
namespace/notes-app created
```

## Step 8 — Create the Deployment

```bash
vim deployment.yaml
```

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: notes-app-deployment
  namespace: notes-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: notes-app
  template:
    metadata:
      name: notes-app-pod
      labels:
        app: notes-app
    spec:
      containers:
        - name: notes-app
          image: <YOUR_DOCKERHUB_USERNAME>/notes-app-k8s:latest
          ports:
            - containerPort: 8000
```

**Explanation of what's different from earlier examples:**
- `metadata.namespace: notes-app` → this Deployment (and its Pod) will run inside our new `notes-app` namespace.
- `spec.replicas: 1` → for this demo, we just need 1 running copy.
- `spec.selector.matchLabels` / `template.metadata.labels` → both set to `app: notes-app` (must match, as covered in Chapter 18).
- `containers[].image` → this now points to the image **you pushed to Docker Hub** in Step 5, NOT a public image like `nginx`.
- `containers[].ports.containerPort: 8000` → this Django application runs on port **8000** (different from Nginx's default port 80) — always confirm which port your application actually listens on.

**Save and exit** (`Esc`, `:wq`, Enter).

Apply it:
```bash
kubectl apply -f deployment.yaml
```

**Verify:**
```bash
kubectl get pods -n notes-app
```
**Expected output:**
```text
NAME                                      READY   STATUS    RESTARTS   AGE
notes-app-deployment-xxxxxxxxx-aaaaa      1/1     Running   0          20s
```

### If Something Goes Wrong
If the Pod shows `ImagePullBackOff`:
**Reason:** Incorrect image name/tag, or the image wasn't successfully pushed to Docker Hub, or the Docker Hub repository is private.
**Fix:** Double-check the exact image name matches what's shown on your Docker Hub repository page, and confirm the repository visibility is set to Public.

## Step 9 — Create the Service

```bash
vim service.yaml
```

```yaml
kind: Service
apiVersion: v1
metadata:
  name: notes-app-service
  namespace: notes-app
spec:
  selector:
    app: notes-app
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
  type: ClusterIP
```

**Explanation:**
- `spec.selector.app: notes-app` → matches the label on our Deployment's Pods.
- `ports.port: 8000` / `targetPort: 8000` → both set to 8000, since that's the port our Django app runs on (unlike the earlier Nginx example, where the outside port was 80 but could differ from the internal port).
- `type: ClusterIP` → since we'll use port-forwarding again for local access, same as Chapter 28.

**Save and exit** (`Esc`, `:wq`, Enter).

Apply it:
```bash
kubectl apply -f service.yaml
```
**Expected output:**
```text
service/notes-app-service created
```

**Verify:**
```bash
kubectl get svc -n notes-app
```

## Step 10 — Port Forward to Access the App

```bash
kubectl port-forward service/notes-app-service -n notes-app 8000:8000 --address 0.0.0.0
```

**Expected output:**
```text
Forwarding from 0.0.0.0:8000 -> 8000
```

## Step 11 — Open the Port in AWS Security Group

**Step 1:** Go to EC2 Console → your instance → **Security** tab → click the Security Group.

**Step 2:** Click **Edit inbound rules** → **Add rule**:
- Type: Custom TCP
- Port range: `8000`
- Source: Anywhere (IPv4: `0.0.0.0/0`)

**Step 3:** Click **Save rules**.

## Step 12 — Access the Application in Your Browser

Open your browser and go to:
```text
http://<YOUR_EC2_PUBLIC_IP>:8000
```

**Expected result:** The Django Notes App's homepage should load successfully, confirming the application is now running live on your Kubernetes cluster.

## Troubleshooting

### Error: `ImagePullBackOff`
**Reason:** Wrong image name, image not pushed, or repository set to private.
**Fix:** Re-verify Docker Hub repository name and visibility; re-push if needed.

### Error: Port forward works, but browser shows "connection refused"
**Reason:** Security Group port not opened, or `--address 0.0.0.0` flag missing from the port-forward command (defaults to localhost-only otherwise).
**Fix:** Confirm the Security Group rule is saved, and that the port-forward command includes `--address 0.0.0.0`.

### Error: Pod crashes immediately (`CrashLoopBackOff`)
**Reason:** The Django app may need environment variables (like a database connection) that aren't set.
**Fix:**
```bash
kubectl logs <pod-name> -n notes-app
```
Check the logs for the specific error and adjust the Deployment YAML with any required `env:` variables.

## Final Result
You have now taken a real application from source code all the way through the complete Kubernetes journey:
```text
Git Clone → Docker Build → Docker Hub Push → Namespace → Deployment → Service → Port Forward → Live in Browser
```

This mirrors the exact same pattern used for the Nginx examples throughout this course — proving that **any containerized application** can be deployed this same way, just by swapping the image and adjusting ports/configuration.

## Classroom Safety Check
- [x] Prerequisites covered (Docker, Docker Hub account, Git)
- [x] Docker build and push covered
- [x] Namespace, Deployment, Service manifests covered field by field
- [x] Port-forward and Security Group steps covered
- [x] Troubleshooting for common errors covered
- [x] Final expected result stated

---
**Note:** This was explicitly called a "mini project" — a bigger, full production-style project (3-tier application with MongoDB, Python app, CI/CD via Jenkins, GitOps via ArgoCD, monitoring via Prometheus/Grafana, deployed on AWS EKS) is planned later in this course, as outlined in Chapter 1's roadmap.

**Chapters still pending (to be added later, as per your instructions):** StatefulSets (with storage), Horizontal/Vertical Pod Autoscaling, Node Affinity, Limits, Probes (readiness/liveness), Ingress, RBAC, Helm, Monitoring (Prometheus/Grafana), CI/CD (Jenkins), GitOps (ArgoCD), and the final 3-tier mega project on EKS.
