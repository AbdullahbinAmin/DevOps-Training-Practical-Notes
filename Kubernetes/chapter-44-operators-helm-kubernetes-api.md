# Chapter 44: Operators, Helm, Kubernetes API

This chapter has three parts:

1. Operators
2. Kubernetes API
3. Helm (the main hands-on part)

---

## Part 1: Operators

- In programming (Go, Python), an **operator** is a program that manages something for you.
- In Kubernetes, once you create a Custom Resource (like `DevopsBatch`), you may want to **program** it: schedule it, extend it, react to changes, run tasks.
- An **Operator** is a piece of code (a controller) that watches custom resources and takes action.
- Examples: **Argo CD operator**, **Prometheus operator**.
- You can write operators in Python with the **Kopf** framework ("Kubernetes Operator Pythonic Framework"), or in Go.

## Part 2: Kubernetes API

- Everything in Kubernetes (pods, deployments, replica sets, services, ingress) can be accessed through the **Kubernetes API**, not only by `kubectl`.
- So you can write programs (for example in Python) that talk to the API to create or read resources.
- Operators use this API.

### Interview note from the video

CRDs, API and Operators are **generally not asked** in interviews (they are for open-source contribution). But **Helm**, **Service Mesh**, and **Security** (Pod Security Standards, image scanning) **are asked**.

---

## Part 3: Helm

### 3.1 Why Helm?

For every application (nginx, MySQL, your own app) you write many manifest files: Deployment, Service, Ingress, HPA, Secret, and so on. For dev, staging and production you may copy them again and again. Most of the structure (kind, apiVersion, metadata, namespace) is the same each time.

**Helm** is the **package manager for Kubernetes**.

Just like:
- `apt-get install unzip` installs software on Ubuntu,
- `helm install` installs applications on Kubernetes using packaged manifests.

Benefits:
- No need to write all manifest files again and again.
- One package can be reused for many environments (dev, staging, prod).
- Easy upgrade, rollback and uninstall.

### 3.2 Install Helm

```bash
mkdir helm && cd helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```
(Use the exact command from the official Helm website.) `helm version` also shows the Go version, since Helm is written in Go.

Run just `helm` to see the help: create, lint, list, package, pull, push, search, install, upgrade, rollback, uninstall, and more.

### 3.3 Create a chart

A **chart** is a Helm package.

```bash
helm create apache-helm
cd apache-helm
sudo apt install tree
tree
```

Helm generates this structure:

```
apache-helm/
├── Chart.yaml       # name, description, chart type, version, appVersion
├── values.yaml      # default values you can change
├── charts/          # dependencies (sub charts)
└── templates/       # manifest templates
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    ├── _helpers.tpl
    └── ...
```

### 3.4 How templates work

Open `templates/deployment.yaml`. You will see values inside double curly brackets, like:

```yaml
replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

This is a **templating engine**. Helm reads values from `values.yaml` and **injects** them into the templates. So you edit only `values.yaml` and never rewrite the whole Deployment or Service.

Small manual change from the video: in `templates/service.yaml` add a target port:

```yaml
ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
    protocol: TCP
```

and in `values.yaml` add:

```yaml
service:
  type: ClusterIP
  port: 80
  targetPort: 80
```

### 3.5 Edit values.yaml (example for Apache)

```yaml
replicaCount: 2

image:
  repository: httpd          # Apache image
  pullPolicy: IfNotPresent   # pull only if the image is not present
  tag: "2.4"

service:
  type: ClusterIP
  port: 80
  targetPort: 80
```

Other things you can set here: service account creation, security context, service type (ClusterIP / NodePort), ingress on/off and class name, CPU and memory limits, probes, autoscaling. All of it flows into the templates.

`Chart.yaml` has chart details: name, description, type (`application`), `version` (chart version) and `appVersion` (application version).

### 3.6 Package the chart

```bash
cd ..                      # go outside the chart folder
helm package apache-helm
# Successfully packaged chart and saved it to: apache-helm-0.1.0.tgz
```
> If you run `helm package .` from inside the chart folder, the `.tgz` gets created inside it, which is not right. Package from outside.

### 3.7 Install a release

```bash
helm install dev-apache apache-helm
```
- `dev-apache` = the **release name** (you choose it).
- `apache-helm` = the chart.

Now `kubectl get pods`, `kubectl get svc`, `kubectl get deployments` all show the Apache resources. With just a few commands, Apache is running.

### 3.8 Use separate namespaces per environment

By default it installs in the `default` namespace, which is not ideal. Use:

```bash
helm install dev-apache apache-helm -n dev-apache --create-namespace
helm install production-apache apache-helm -n production-apache --create-namespace
```
- `-n` sets the namespace.
- `--create-namespace` creates the namespace if it does not exist.

Then:
```bash
kubectl get pods -n dev-apache
kubectl get pods -n production-apache
```

Same chart, different environments. Even a staging one takes only two minutes.

### 3.9 Upgrade

Change something, for example set `replicaCount: 3` in `values.yaml` and raise the `version` in `Chart.yaml` (for example `0.1.0` to `0.1.1`). Then:

```bash
helm package apache-helm
helm upgrade production-apache apache-helm -n production-apache
```
Now this is **revision 2**. Production shows 3 pods.

> Trying to upgrade a release that does not exist gives: "has no deployed releases". Install first.

### 3.10 Rollback

Go back to revision 1 (2 pods):

```bash
helm rollback production-apache 1 -n production-apache
```
Output: "Rollback was a success. Happy Helming!"

### 3.11 List, history, uninstall

```bash
helm list                      # releases in default namespace
helm list -n mongo             # releases in a namespace
helm list -A                   # all namespaces
helm history production-apache -n production-apache
helm uninstall dev-apache -n dev-apache
```
Uninstall deletes pods, deployments and services of that release.

**Helm lifecycle:** create, install, upgrade, rollback, delete (uninstall).

### 3.12 A Node.js app with Helm

```bash
helm create nodejs-app
```
In its `values.yaml`:
- `image.repository: trainwithshubham/node-app`
- `pullPolicy: IfNotPresent`
- `tag: latest`
- service port and targetPort: `8000` (the container runs on 8000)

```bash
helm package nodejs-app
helm install dev-nodejs-app nodejs-app -n dev-node --create-namespace
kubectl get pods -n dev-node
kubectl get svc -n dev-node
kubectl port-forward svc/<service-name> 8000:8000 -n dev-node --address 0.0.0.0
```
Open port 8000 in the security group and visit `http://<public-ip>:8000`. No YAML file was written by hand. Only the image was put into `values.yaml`.

### 3.13 Helm repositories (install ready-made apps)

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo list
helm search repo nginx
helm repo update
```

Install from a public registry (example: Nginx from Bitnami, OCI registry):
```bash
helm install my-nginx oci://registry-1.docker.io/bitnamicharts/nginx
kubectl get pods
helm uninstall my-nginx
```

Install MongoDB the same way:
```bash
helm install mongodb oci://registry-1.docker.io/bitnamicharts/mongodb -n mongo --create-namespace
kubectl get pods -n mongo
kubectl exec -it <pod-name> -n mongo -- bash
# then use mongosh inside
helm uninstall mongodb -n mongo
```
A working MongoDB in two minutes.

Helm is also used to install Argo CD, Prometheus and other tools (Prometheus repo: `helm repo add prometheus-community ...`).

### 3.14 Summary

- Helm = package manager for Kubernetes.
- A chart = templates + values.
- `helm create`, `helm package`, `helm install`, `helm upgrade`, `helm rollback`, `helm uninstall`.
- Use `-n` and `--create-namespace` to separate dev, staging and production.
- Helm repos let you install popular apps with one command.
