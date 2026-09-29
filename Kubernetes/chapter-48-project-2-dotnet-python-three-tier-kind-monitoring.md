# Chapter 48: Project 2 - .NET, Python, Three-tier App on KIND (with Monitoring)

## 1. Project overview

This project uses the **voting application** (a "Cats vs Dogs" voting app). It has microservices in different languages:

| Component | Technology | Job |
|---|---|---|
| Voting service | Python | Users vote for cat or dog (port 5000) |
| Result service | Node.js | Shows results (port 5001) |
| Worker | .NET | Reads votes from Redis and writes to the database |
| Redis | Redis | Temporary queue for votes |
| Database | PostgreSQL | Stores final votes |

The Kubernetes manifests for it are in a GitHub repo (`k8s-kind-voting-app`). They include: DB deployment and service, Redis deployment and service, result deployment and service, voting deployment and service, and worker deployment.

Cluster type for this project: **KIND**. The main topic of this chapter is **monitoring the cluster with Prometheus and Grafana using Helm**.

### Quick deploy of the voting app

```bash
git clone <k8s-kind-voting-app-repo>
cd k8s-kind-voting-app
kubectl apply -f .
kubectl get pods
kubectl get svc
```
All the pods (db, redis, result, vote, worker) come up in the `default` namespace.

Expose the services:
```bash
kubectl port-forward svc/vote 5000:5000 --address 0.0.0.0 &
kubectl port-forward svc/result 5001:5001 --address 0.0.0.0 &
```
Open ports 5000 and 5001 in the AWS security group (Custom TCP, anywhere), then open `http://<public-ip>:5000` to vote and `http://<public-ip>:5001` to see results.

---

## 2. What is Observability?

Observability stands on **three pillars**:

| Pillar | Answers | Tools |
|---|---|---|
| **Metrics** (monitoring) | **What** is happening in my server? (CPU, RAM, network) | Prometheus, Grafana |
| **Logs** (logging) | **Why** did it happen? (error lines) | Loki, Promtail, ELK |
| **Traces** (tracing) | **How** did it happen? (path of a request) | Jaeger, OpenTelemetry |

Visualisation tools: Grafana, Kibana, Power BI, Datadog.

---

## 3. Kubernetes monitoring architecture

A Kubernetes cluster has:
- **Control plane (master):** API server, scheduler, etcd, controller manager (these run in the `kube-system` namespace).
- **Worker nodes:** each runs a **kubelet** and your pods.

To monitor everything you need more than one exporter:

1. **Node Exporter**
   - Runs on every node (a DaemonSet).
   - Exports node information like CPU, RAM, disk, network.
   - Exposes on port **9100**.
   - If you have 1 master and 3 workers, you get 4 node exporters.
2. **kube-state-metrics**
   - Shows the **state of the Kubernetes objects and control plane**: is the scheduler running, is the controller manager healthy, how many pods are pending, deployment status, kubelet, and so on.
   - Node exporter alone is not enough; you also need this. This gives a 360 degree view.
3. **Prometheus**
   - A **time series database** that **scrapes** (pulls) data from many sources: node exporters, kube-state-metrics, API server, scheduler, kubelet, and more.
   - Has a query language **PromQL** and a query server; can also draw simple graphs.
4. **Grafana**
   - **Visualises** the data from Prometheus (data source) with dashboards and graphs.

Flow: `Node exporter + kube-state-metrics + API server ... -> Prometheus (store & query) -> Grafana (dashboards)`.

Doing all of this one by one is hard. **Helm** installs the whole stack with one command.

---

## 4. Install the monitoring stack with Helm

### 4.1 Install Helm
```bash
mkdir monitoring && cd monitoring
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

### 4.2 Create a namespace for monitoring
```bash
kubectl create namespace monitoring
```
Everything monitoring-related lives in this isolated namespace.

### 4.3 Add the Prometheus community repo
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm repo list
```
`helm repo update` fetches the latest charts.

### 4.4 Install kube-prometheus-stack

```bash
helm install kube-prom-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.service.type=NodePort \
  --set prometheus.service.nodePort=30000 \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=31000
```

Explanation:
- `kube-prom-stack` = release name.
- `--namespace monitoring` = install in that namespace.
- `--set` = override values from `values.yaml` on the command line.
- By default services are **ClusterIP**. We change them to **NodePort** so we can reach them from outside. NodePort range is **30000 to 32000**.
- Prometheus nodePort = 30000, Grafana nodePort = 31000.

Check:
```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```
You will see: node exporters (one per node, for example 4 for 1 master + 3 workers), kube-state-metrics, Prometheus, Alertmanager, Grafana, and the operator.

The stack includes everything: Prometheus, Grafana, node exporter, kube-state-metrics, alert manager, with many ready scraping configurations (`prometheus.yml` is already configured by Helm).

---

## 5. Access Prometheus

```bash
kubectl get svc -n monitoring
kubectl port-forward svc/kube-prom-stack-kube-prome-prometheus 9090:9090 -n monitoring --address 0.0.0.0 &
```
- Open **port 9090** in the security group.
- Open `http://<public-ip>:9090`.
- Go to **Status, Targets** to see all things being scraped: alertmanager, API server, kubelet, node exporters, kube-state-metrics, scheduler, and so on. A few may be down; that is fine to ignore for learning.

Use the query box (PromQL) to check metrics.

---

## 6. Access Grafana

```bash
kubectl port-forward svc/kube-prom-stack-grafana 3000:80 -n monitoring --address 0.0.0.0 &
```
- Grafana runs inside on port 80; we map it to 3000 outside.
- Open **port 3000** in the security group.
- Open `http://<public-ip>:3000`.

### Grafana login

Default `admin/admin` may fail. Get the real password from the Secret:

```bash
kubectl get secret -n monitoring kube-prom-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```
- Secrets store data in `data:` in **base64** form.
- `base64 --decode` converts it back. (The password in the video was `prom-operator`.)
- User name: `admin`.

The lesson: anyone who can read a Secret can decode it. So restrict access with RBAC (Chapter 41).

---

## 7. Grafana dashboards

### 7.1 Data source
Go to **Connections, Data sources**. Prometheus is **already added** by Helm with the URL `http://kube-prom-stack-kube-prome-prometheus.monitoring:9090`. (Format is `service-name.namespace`.)

### 7.2 Ready-made dashboards
Go to **Dashboards**. Many are already made: Node Exporter, Kubernetes scheduler, persistent volumes, pods, namespaces, cluster, and more. Just click and view.

### 7.3 Import a community dashboard
1. Search "Grafana dashboards" on grafana.com and open Grafana Labs dashboards.
2. Search "kubernetes prometheus" and pick one.
3. Click **Copy ID to clipboard**.
4. In your Grafana: **New, Import**, paste the ID, click **Load**, select the **Prometheus** data source, and click **Import**.
5. You now see your pods, CPU use, which node has the most load, and more.

### 7.4 Build your own visualisation
- **Add visualization**, pick the Prometheus data source.
- Choose a metric, for example `container_network_receive_bytes_total`, filter by `namespace`.
- You can set colours with **value mappings** or overrides. For example, make **terminated containers red**, **running containers green**, and **waiting containers blue**. This helps tell a story at a glance.

---

## 8. Practice: monitor the voting app

1. Deploy the voting app (default namespace).
2. In Grafana (Kubernetes networking dashboards), choose namespace **default** and select the vote or result workload.
3. Send many votes. The **current rate of bytes received** goes up on the dashboard.
4. In the pod dashboard, you can see which **node** has the highest load (for example `cluster-worker-1`). Then run `kubectl get pods -o wide` to see which pod is on that node.

This is how DevOps engineers spot pressure and decide to **scale** (using Horizontal Pod Autoscaler or Vertical Pod Autoscaler). Big streaming platforms watch such dashboards all the time during high traffic.

---

## 9. Extra: Kubernetes objects used

Everything here uses ideas from earlier chapters: Namespace, Deployment, Service (ClusterIP, NodePort), Secret, DaemonSet (node exporter), Helm, port-forwarding, and security group rules.

## 10. Summary

- Observability = metrics + logs + traces.
- Metrics: **Prometheus** (collect) + **Grafana** (visualise). Node exporter (nodes) + kube-state-metrics (cluster objects) give complete monitoring.
- Install everything with one Helm chart: `prometheus-community/kube-prometheus-stack`.
- Change service type to **NodePort** (30000-32000) or port-forward to access outside.
- Get the Grafana password from the Secret and decode it with `base64 --decode`.
- Import ready dashboards using an ID, or build your own.
- Use the dashboards to find heavy nodes and decide on scaling.
