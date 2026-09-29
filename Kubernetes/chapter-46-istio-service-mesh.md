# Chapter 46: Istio Service Mesh

## 1. What is a Service Mesh?

A real application is not one program. It is many small services (microservices). Examples:

- **Voting app:** a voting service, a result service, a worker service that counts votes.
- **Zomato-like app:** menu service, order service, delivery service, restaurant service, events service.

These services must talk to each other. Each runs on its own port. This creates the same problem as heavy road traffic in a city: something must decide which request goes where.

A **service mesh** is a layer that manages this traffic between services. It gives you:

- **Routing / gateway:** decide where each request goes (like a traffic police officer).
- **Service to service communication** with **encryption** (mTLS).
- **Automatic load balancing.**
- **Fine-grained traffic control** (retries, splitting traffic, and so on).
- **Metrics, logs and traces** for every service.
- A visual map of which service calls which.

The word *mesh* means a net: many services connected to each other.

## 2. What is Istio?

**Istio** is a very popular, open-source service mesh. It layers transparently on top of running applications. Other names in this space: Linkerd, Consul.

### How Istio works (architecture)

Istio has two parts:

1. **Data plane**
   - Made of **Envoy sidecar proxies**. An Envoy proxy runs as a **sidecar container** next to each microservice (this uses the sidecar idea from Chapter 45).
   - It handles all incoming (ingress) and outgoing (egress) traffic of that microservice.
2. **Control plane**: **istiod**. It provides:
   - **Service discovery** (which services exist)
   - **Pilot:** configures the proxies
   - **Citadel:** issues certificates (security)
   - **Galley:** validates and ingests configuration

Interview line: *"Istio uses Envoy sidecar proxies per microservice to handle ingress and egress traffic, and istiod as the control plane."*

Istio is not an overlay. It simplifies and enhances the existing network.

---

## 3. Setup on a kind cluster

Istio has separate prerequisites for each platform (GKE, EKS, k3d, minikube, OpenShift, and so on). For **kind**, use the kind page in the Istio docs.

### 3.1 Create a cluster (local machine)

```bash
mkdir istio-practice && cd istio-practice
kind create cluster --name istio-testing
kubectl get nodes
```
(If you already have a multi-user or existing cluster running, you may see errors. Use a fresh local kind cluster.)

### 3.2 Download Istio

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.x.x
ls
```
Contents: `LICENSE`, `bin/` (contains `istioctl`), `manifests/`, `samples/`, `tools/`. Always read the `README.md` when you get a new folder.

`istioctl` is to Istio what `kubectl` is to Kubernetes.

### 3.3 Add istioctl to PATH

```bash
sudo mv bin/istioctl /usr/local/bin/
istioctl version
```
(Use `sudo` if you get permission denied.)

### 3.4 Install Istio

```bash
istioctl install --set profile=demo -y
```
This installs `istiod` (control plane) and the ingress/egress gateways. (Use the demo profile for learning.)

### 3.5 Enable automatic sidecar injection

Istio is a third-party tool. To let it add its sidecar in a namespace, label that namespace:

```bash
kubectl label namespace default istio-injection=enabled
kubectl get ns --show-labels
```
Now every pod created in `default` gets an Envoy sidecar automatically. (You may see "not labeled" if it is already labeled; use `--overwrite` if needed.)

### 3.6 Install Gateway API CRDs

CRD means Custom Resource Definition (Chapter 43). The Gateway API is not built into Kubernetes, so install its CRDs:

```bash
kubectl get crd gateways.gateway.networking.k8s.io
# if not found, install as shown in the Istio docs:
kubectl apply -f <gateway-api-crds-url-from-istio-docs>
```

---

## 4. Deploy the Bookinfo sample application

Bookinfo is a sample bookstore with four microservices:

| Service | Language | Job |
|---|---|---|
| **productpage** | Python | Front page, calls details and reviews |
| **details** | Ruby on Rails | Book details |
| **reviews** | Java | Book reviews (has v1, v2, v3) |
| **ratings** | Node.js | Star ratings used by reviews |

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl get pods
kubectl get svc
```
It creates services, service accounts, and deployments for all four. Pods show `2/2` ready because each has the app container **plus** the Istio sidecar.

### Check that the app works inside the cluster

```bash
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```
Expected output: `<title>Simple Bookstore App</title>`.

### Open access to outside traffic with a Gateway

```bash
kubectl apply -f samples/bookinfo/gateway-api/bookinfo-gateway.yaml
```
The Gateway acts like a traffic inspector: requests to `/productpage`, `/static`, `/login`, `/logout`, `/api/v1/products` are sent to the right service.

By default the gateway service type is LoadBalancer. On a local cluster change it to ClusterIP:

```bash
kubectl annotate gateway bookinfo-gateway networking.istio.io/service-type=ClusterIP --namespace=default
kubectl get gateway
```

Access it with port-forward:

```bash
kubectl port-forward svc/bookinfo-gateway-istio 8080:80
```
Open `http://localhost:8080/productpage`. You will see "The Comedy of Errors" with book details and reviews. Refresh a few times and you will see different review versions (no stars, black stars, red stars), which shows traffic going to different reviews services.

Without a service mesh you would have to port-forward the productpage, ratings, and reviews services one by one. With 50 microservices that becomes impossible to manage. That is why we use Istio.

---

## 5. See the mesh with Kiali

**Kiali** is a dashboard that draws the service mesh.

### Install add-ons
```bash
kubectl apply -f samples/addons
kubectl rollout status deployment/kiali -n istio-system
```
This installs Kiali (and other add-ons like Prometheus, Grafana, Jaeger).

### Generate some traffic (100 requests)

```bash
for i in $(seq 1 100); do curl -s -o /dev/null "http://localhost:8080/productpage"; done
```
Each request goes to productpage, then details, then reviews, then ratings.

### Open Kiali

```bash
istioctl dashboard kiali
```

In the Kiali UI:
- Go to **Applications** to see details, productpage, ratings and reviews.
- Go to **Traffic Graph**, select the `default` namespace, and set the time range (last 5 minutes).
- You will see how traffic flows: gateway, then productpage, then details and reviews, then ratings.
- **Triangles** are services and **squares** are deployments/workloads (see the legend).
- Arrows show which service calls which. You can replay traffic and see it live.
- You can view the mesh config, service names, and Istio config (`bookinfo` config).

The traffic graph shows something you would never see easily otherwise: **reviews calls ratings**. That is the power of a service mesh.

---

## 6. Summary

- Service mesh = a smart layer to manage traffic between microservices.
- **Istio** = the popular mesh tool. Data plane = Envoy sidecars. Control plane = istiod.
- Install: `istioctl install --set profile=demo`, label namespace `istio-injection=enabled`, install Gateway API CRDs.
- Deploy the sample app **Bookinfo** and expose it using a **Gateway**.
- Use **Kiali** to see the live traffic map and understand your services.
- Interview points: what is Istio, why it uses a sidecar proxy, what istiod does.
