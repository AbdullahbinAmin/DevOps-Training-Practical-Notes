# Chapter 47: Project 1 - Chat Application (3-tier) on Minikube

## 1. Project overview

We deploy a **three-tier full stack real-time chat application** on **Minikube** (a local Kubernetes cluster).

| Tier | Technology |
|---|---|
| Frontend | React app served by **Nginx** |
| Backend | **Node.js** |
| Database | **MongoDB** |

The application code comes from a community learner's project on GitHub (full stack chat app). We fork it and write our own Kubernetes manifests.

The other projects in the course are spread over different cluster types: this one on Minikube, the next on KIND, and the mega project on EKS.

### The plan (what we build)

1. Docker image for the frontend and the backend (MongoDB uses the ready image).
2. Push the images to Docker Hub.
3. Kubernetes manifests: **Namespace, Deployments, Services, Secret, PersistentVolume, PersistentVolumeClaim, Ingress**.

---

## 2. Prepare the local machine

1. Start Minikube with the Docker driver:
   ```bash
   minikube start --driver=docker
   kubectl get nodes
   ```
2. Install **VS Code** (free, open-source code editor) and open a terminal in it.
3. **Fork** the chat app repository on GitHub (fork copies a repo into your own account), then **clone** it:
   ```bash
   git clone https://github.com/<your-username>/fullstack-chat-app.git
   ```
4. Delete the existing `k8s` folder from the cloned code. We write our own from scratch.

Project folders: `backend/` (with `src` and a `Dockerfile`) and `frontend/` (with a `Dockerfile`).

---

## 3. Build and push Docker images

### Login to Docker Hub

Create a **personal access token** in Docker Hub (Account settings, Personal access token, Read/Write/Delete permission). Then:

```bash
docker login -u <your-dockerhub-username>
# paste the token as the password
```

### Backend image

```bash
cd backend
docker build -t <username>/chatapp-backend:latest .
docker push <username>/chatapp-backend:latest
```

### Frontend image

```bash
cd ../frontend
docker build -t <username>/chatapp-frontend:latest .
docker push <username>/chatapp-frontend:latest
```

To rename the tag of an image later: `docker tag <old> <new>`, then `docker push <new>`.

### MongoDB image

No need to build. Use the official `mongo:latest` image directly from Docker Hub.

---

## 4. Kubernetes manifests (inside a `k8s/` folder)

Tip: install the **Kubernetes extension** in VS Code. It gives suggestions and templates. Typing `deployment` or `service` and pressing Enter auto-generates the full YAML.

### 4.1 Namespace

`namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: chat-app
```
```bash
cd k8s
kubectl create -f namespace.yaml    # or apply
```
(`create` is for the first time; `apply` is used to configure existing or new objects. Both work.)

### 4.2 Backend Deployment

Before writing it, check the Dockerfile for the **port** to expose (5001 for the backend) and for **environment variables** (`NODE_ENV=production`).

`backend-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: chat-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: chat-backend
          image: <username>/chatapp-backend:latest
          ports:
            - containerPort: 5001
          env:
            - name: NODE_ENV
              value: "production"
            - name: PORT
              value: "5001"
            - name: MONGODB_URI
              value: "mongodb://mongoadmin:secret@mongodb:27017/dbname?authSource=admin"
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: chat-app-secrets
                  key: jwt-secret
```

Important points:
- The label in `selector.matchLabels` must **match** the label in `template.metadata.labels`.
- `env` values must be **strings** (use quotes). A number without quotes gives an error like "cannot unmarshal number into Go struct field ... of type string".
- The **Mongo URI** format: `mongodb://<user>:<password>@<mongodb-service-name>:<port>/<db>?authSource=admin`. The host name is the **name of the MongoDB Service**.
- Always read the project's README to find which environment variables it needs (here `MONGODB_URI`, `JWT_SECRET`, `PORT`).

### 4.3 Frontend Deployment

Same shape. Port is **80** (Nginx), image `<username>/chatapp-frontend:latest`, label `app: frontend`, container name like `chat-frontend`, env `NODE_ENV: production`.

The frontend's Nginx config (`nginx.conf`) forwards `/api` to `http://backend:5001`, so **your backend Service must be named `backend`** and be in the same namespace. If not, the frontend pod crashes with:

> host not found in upstream "backend"

This is like "tightly coupled code". Lesson: an engineer must be able to read the code and find why a pod crashes (`kubectl logs`, `kubectl describe pod`).

### 4.4 MongoDB Deployment

`mongodb-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  namespace: chat-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:latest
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              value: "mongoadmin"
            - name: MONGO_INITDB_ROOT_PASSWORD
              value: "secret"
          volumeMounts:
            - name: mongo-data
              mountPath: /data/db
      volumes:
        - name: mongo-data
          persistentVolumeClaim:
            claimName: mongodb-pvc
```
Default MongoDB port is **27017**. The volume is mounted at `/data/db`, where MongoDB stores its data.

### 4.5 Persistent Volume and Claim

`mongodb-pv.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/mongodb
```

`mongodb-pvc.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
  namespace: chat-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

- **PV** = the actual storage (5 GB on the minikube node at `/data/...`).
- **PVC** = a request/claim for that storage. The pod uses the PVC.
- Apply order: PV, then PVC, then the MongoDB Deployment.

### 4.6 Services

Create one Service for each of the three components.

`backend-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend            # must be "backend" (the nginx config depends on it)
  namespace: chat-app
spec:
  selector:
    app: backend
  ports:
    - port: 5001
      targetPort: 5001
```

`frontend-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: chat-app
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```

`mongodb-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb            # this name is used in MONGODB_URI
  namespace: chat-app
spec:
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
```
The selector must match the pod labels; the Service `port` and `targetPort` should match the container ports. Tip: in VS Code just type `service` and press Enter for the template.

### 4.7 Secret (for the JWT key)

`secrets.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: chat-app-secrets
  namespace: chat-app
type: Opaque
data:
  jwt-secret: <base64-encoded-value>
```
- Generate a JWT secret key with any online generator.
- Secret values **must be base64 encoded**. Encode with: `echo -n "yourkey" | base64` (or an online base64 encoder). If not encoded you see: "illegal base64 data".
- In the Deployment, read it with `valueFrom.secretKeyRef` (name = the secret name, key = `jwt-secret`).
- Note: in production, use a proper secrets manager. Base64 is encoding, not encryption.

---

## 5. Apply everything (order matters)

```bash
kubectl apply -f namespace.yaml
kubectl apply -f mongodb-pv.yaml
kubectl apply -f mongodb-pvc.yaml
kubectl apply -f mongodb-deployment.yaml
kubectl apply -f mongodb-service.yaml
kubectl apply -f secrets.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml

kubectl get pods -n chat-app
kubectl get svc  -n chat-app
```

### Troubleshooting flow shown in the video

| Problem | How it was found | Fix |
|---|---|---|
| Frontend pod in `CrashLoopBackOff` | `kubectl logs` showed "host not found in upstream backend" | Create the Service named **backend** |
| Backend error connecting to database | `kubectl logs` showed a MongoDB connection error | Add `MONGODB_URI` env and create the **mongodb** Service |
| Missing JWT | App needs `JWT_SECRET` | Create the secret and use `secretKeyRef` |
| Number in env value | Apply error "cannot unmarshal number ... of type string" | Put env values in quotes |
| Secret error "illegal base64 data" | Applying the secret | Base64-encode the value |

Useful commands: `kubectl describe pod <name> -n chat-app`, `kubectl logs <pod> -n chat-app`, `kubectl get pods -w -n chat-app`.

---

## 6. Access the app: Port-forward

```bash
# Backend
kubectl port-forward svc/backend 5001:5001 -n chat-app --address 0.0.0.0 &

# Frontend (port 80 needs sudo on some systems)
sudo -E kubectl port-forward svc/frontend 80:80 -n chat-app --address 0.0.0.0 &
```
Open `http://localhost`. Create an account (name, email, password) and log in.

### Test the chat

Open a second browser window in **incognito mode**, create another account (for example a friend), and send messages between the two users. The chat works in real time.

---

## 7. Access the app: Ingress

Instead of port-forwarding, use an **Ingress** so you can open a domain name like `chat.local` (example host).

1. Enable the Ingress add-on for Minikube:
   ```bash
   minikube addons enable ingress
   kubectl get ns              # a new namespace ingress-nginx appears
   kubectl get pods -n ingress-nginx
   ```
   (This takes a little time while it verifies.)
2. Create `ingress.yaml`:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: chat-app-ingress
     namespace: chat-app
     annotations:
       nginx.ingress.kubernetes.io/rewrite-target: /
       nginx.ingress.kubernetes.io/ssl-redirect: "false"
   spec:
     ingressClassName: nginx
     rules:
       - host: chat.example.com
         http:
           paths:
             - path: /
               pathType: Prefix
               backend:
                 service:
                   name: frontend
                   port:
                     number: 80
             - path: /api
               pathType: Prefix
               backend:
                 service:
                   name: backend
                   port:
                     number: 5001
   ```
   - `/` goes to the frontend service (port 80).
   - `/api` goes to the backend service (port 5001).
   - `rewrite-target: /` avoids path problems.
   - `ssl-redirect: "false"` stops warnings if you do not use HTTPS.
3. Map the host name to your machine in `/etc/hosts`:
   ```
   127.0.0.1  chat.example.com
   ```
4. Minikube on Docker: open a tunnel, or port-forward the ingress controller service:
   ```bash
   minikube tunnel
   # or
   sudo kubectl port-forward svc/ingress-nginx-controller 80:80 -n ingress-nginx
   ```
   > If port 80 is "already in use", find what uses it (`sudo lsof -i :80`) and stop it (for example delete the old port-forward), or use another port.
5. Visit `http://chat.example.com`. The app now works through the Ingress. `kubectl get ingress -n chat-app` shows the rule.

---

## 8. Push the manifests to Git

```bash
git add .
git commit -m "added k8s manifest"
git push origin main
```
Now your `k8s/` folder with all manifests is stored in your GitHub repo. You can use it in your resume project.

---

## 9. Concepts used in this project (revision)

Docker image, Docker Hub, Deployments, Services, Ingress, Namespace, PV, PVC, Secrets, environment variables, port-forwarding, Minikube add-ons, and troubleshooting with logs.

## 10. Summary

- 3-tier app: React (Nginx) + Node.js + MongoDB.
- Build images, push to Docker Hub, write manifests, apply in order.
- Service **names matter** (`backend`, `mongodb`) because other components refer to them.
- Use a Secret (base64) for sensitive values.
- Expose using port-forward or Ingress.
- Read logs and code to fix errors.
