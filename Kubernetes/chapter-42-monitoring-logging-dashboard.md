# Chapter 42: Monitoring and Logging using Kubernetes Dashboard

## 1. Why cluster-level access?

In the RBAC chapter we made service accounts with access only inside a **namespace** (for example `apache-user`). But if you want to:

- see metrics of the **whole cluster**,
- monitor everything,
- read logs of all pods and deployments,

then you need **cluster-level** access. That means a **ClusterRole** and **ClusterRoleBinding**.

The **Kubernetes Dashboard** is a web UI that shows everything running in your cluster: pods, deployments, services, ingress, config maps, secrets, storage, logs, events and more.

---

## 2. Steps to set up the Dashboard

### Step 1: Install the dashboard manifests

The dashboard manifest comes from the official Kubernetes GitHub repository. It creates many things:

- a namespace `kubernetes-dashboard`
- a service account
- secrets and a config map
- a Role and a ClusterRole
- a RoleBinding and a ClusterRoleBinding
- a deployment and a service

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/<version>/aio/deploy/recommended.yaml
```
(Use the URL from the official dashboard page for the current version.)

So this one file uses almost every concept we learned: namespace, service account, service, secret, config map, RBAC, deployment.

### Step 2: Create an admin service account

Make a folder called `dashboard` and create `dashboard-admin-user.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

Points to remember:

- The **namespace must be `kubernetes-dashboard`**.
- We do not create a ClusterRole ourselves, because `cluster-admin` already exists. We only need a **ClusterRoleBinding** that connects our user to it.
- You can put many YAML objects in one file by separating them with `---`. This is fully fine.
- `subjects` is a list, so indentation matters. (In the video, a wrong or missing field in `subjects` gave an error: "invalid subject, apiGroup unsupported". Fix the fields and apply again.)

```bash
kubectl apply -f dashboard-admin-user.yaml
```

To find the name of a ClusterRole: `kubectl get clusterrole`. There are many, but `cluster-admin` is the one that has all permissions.

### Step 3: Create a login token

The dashboard needs a token to log in:

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

Copy the long token and save it somewhere safe.

### Step 4: Start the proxy

```bash
kubectl proxy
```
By default this serves only on `127.0.0.1` (localhost) at port **8001**.

To reach it from another machine (like an EC2 server), use:

```bash
kubectl proxy --address 0.0.0.0 --accept-hosts '.*'
```
- `--address 0.0.0.0` lets it listen on all network interfaces.
- `--accept-hosts '.*'` lets it accept requests from any host name. (Without this you get a "Forbidden" error.)
- Add `&` at the end to run it in the background.

### Step 5: Open the dashboard URL

```
http://<address>:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

If the server is on AWS EC2, open **port 8001** in the security group inbound rules.

---

## 3. Problem faced in the video: "Insecure access detected"

When the dashboard was opened over plain HTTP using a public IP, the login page showed:

> Insecure access detected. Sign in will not be available. Access Dashboard securely over HTTPS or using localhost.

**Reason:** the dashboard sign-in works only over **HTTPS** or through **localhost**.

**Solution used:** run the cluster and the proxy on the **local machine**, and open `http://localhost:8001/...`. That worked and the token login succeeded.

Lesson: real engineers face many errors. Search the error, read the docs, and fix step by step.

---

## 4. Local setup used in the video (kind cluster)

To run this locally with kind:

```bash
kind create cluster --name twa-cluster --config kind-config.yaml
kubectl get nodes
```

Then repeat the dashboard steps: apply manifests, apply `dashboard-admin-user.yaml`, create the token, run `kubectl proxy`, open the localhost URL, paste the token, click sign in.

Extra note about Git in the video: when pushing project files to GitHub, folders that already have a `.git` inside (like a cloned repo) cause the message "adding embedded git repository". Remove the inner `.git` folder (`rm -rf .git` inside that sub-folder) before `git add .`.

---

## 5. What you see in the Dashboard

After creating some workloads (for example the nginx namespace with deployments, daemon sets, a cron job and ingress), you can switch namespaces at the top and see:

- **Workloads:** cron jobs, daemon sets, deployments, jobs, pods, replica sets, replication controllers, stateful sets, and their status.
- **Service:** ingresses, ingress classes, services.
- **Config and Storage:** config maps, persistent volume claims, secrets, storage classes.
- **Cluster:** cluster roles, cluster role bindings, events, namespaces, network policies, nodes, persistent volumes, roles, role bindings, service accounts, CRDs.
- **Logs and events:** open any pod to see logs, events, and where volumes are mounted.
- Pending pods are shown clearly.

If the namespace selected has nothing, it shows "There is nothing to display here". Change the namespace at the top.

---

## 6. Summary

- Use **ClusterRole + ClusterRoleBinding** when the user needs access to the whole cluster.
- The dashboard needs: manifests, admin service account, cluster-admin binding, token, and `kubectl proxy`.
- Sign-in only works via **localhost or HTTPS**.
- The dashboard is a good visual way to see workloads, logs and metrics. For serious monitoring we later use **Prometheus and Grafana** (Chapter 48).
