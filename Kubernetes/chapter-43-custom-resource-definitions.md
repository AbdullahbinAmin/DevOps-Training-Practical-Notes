# Chapter 43: Custom Resource Definitions (CRDs)

## 1. What is a Resource?

In Kubernetes, things like **Pod, Service, Deployment, Ingress, Namespace** are all **resources**. Kubernetes already knows their structure. For example, when you create a Pod, you write `containers`, then `name` and `image`.

But how does Kubernetes know that structure? Someone defined it in code. Every resource has:

- **apiVersion** made of a **group** and a **version**
  - Pod: group is blank (core), version `v1`. So `apiVersion: v1`.
  - Deployment: group `apps`, version `v1`. So `apiVersion: apps/v1`.
- **kind** (Pod, Deployment, ...)
- **metadata**
- **spec** (the details)

---

## 2. What is a Custom Resource Definition (CRD)?

Sometimes you want a resource that **Kubernetes does not have by default**. Example: you teach batches, and you want a resource called `DevopsBatch` that stores batch name, duration, platform and mode.

Kubernetes does not know what a `DevopsBatch` is. A **CRD** teaches it:

- **CRD** = the **definition / structure** (a blueprint).
- **CR (Custom Resource)** = an actual object created from that blueprint.

Real tools use CRDs too: Prometheus, Argo CD, Thanos, Vertical Pod Autoscaler all add their own custom resources.

---

## 3. Create a CRD (step by step)

```bash
mkdir crd && cd crd
vim devops-crd.yaml
```

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: devopsbatches.trainwithshubham.com
spec:
  group: trainwithshubham.com
  names:
    plural: devopsbatches
    singular: devopsbatch
    kind: DevopsBatch
    shortNames:
      - dvb
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                name:
                  type: string
                  description: This is the name of the devops batch
                duration:
                  type: string
                  description: This is the duration of the devops batch
                mode:
                  type: string
                  description: Mode of the batch, for example live or recorded
                platform:
                  type: string
                  description: Platform where the batch is taught
```

### Explanation of every field

- **apiVersion: apiextensions.k8s.io/v1** and **kind: CustomResourceDefinition**: always these for a CRD. `apiextensions.k8s.io` is the group name for CRDs.
- **metadata.name**: must be in the format `<plural>.<group>`. Here: `devopsbatches.trainwithshubham.com`. (If you write this wrongly, you get errors like "resource name may not be empty". Also remember `metadata` must be all lowercase; a capital `M` or `D` gives errors.)
- **spec.group**: your own group name, like `trainwithshubham.com`.
- **names**:
  - `plural`: used in `kubectl get devopsbatches` (like `pods`).
  - `singular`: like `pod`.
  - `kind`: the name used in YAML, written in CamelCase (like `Pod`, `Deployment`).
  - `shortNames`: short forms, like `svc` for service. Here `dvb`.
- **scope**: `Namespaced` (lives inside a namespace) or `Cluster`.
- **versions**: you can add new versions over time (v1, v2, ...).
  - `served: true` means the API serves this version, so it can be used.
  - `storage: true` means this version is the one stored in etcd.
- **schema.openAPIV3Schema**: the structure of your resource. OpenAPI v3 is a standard way to say: type is object, properties are these. Inside `spec` we define `name`, `duration`, `mode`, `platform`, each with a `type` (string) and a `description`.

Apply it:

```bash
kubectl apply -f devops-crd.yaml
kubectl get crd
# devopsbatches.trainwithshubham.com
```

Now Kubernetes knows about `DevopsBatch`.

---

## 4. Create a Custom Resource (CR)

`devops-cr.yaml`
```yaml
apiVersion: trainwithshubham.com/v1
kind: DevopsBatch
metadata:
  name: junoon-batch-na
spec:
  name: Zero to Hero Junoon Batch
  duration: 3 months from 25 January 2025
  platform: trainwithshubham.com
  mode: live
```

- `apiVersion` = `<group>/<version>` = `trainwithshubham.com/v1`.
- `kind` = the kind from your CRD.
- `spec` = the fields you defined in the schema.

```bash
kubectl apply -f devops-cr.yaml
kubectl get devopsbatches
kubectl get dvb              # short name
kubectl describe devopsbatch junoon-batch-na
```

You can create as many as you want. Just change the `metadata.name`, for example `junoon-batch-10`, and apply again.

Common mistake from the video: writing `-f` correctly is needed (`kubectl apply -f file.yaml`); a typo like `hain f` gives an error.

Note: a small mistake in the CRD (like a wrong name) can waste a lot of time. Always check the official Kubernetes documentation for the CRD template.

---

## 5. Why do we use CRDs?

- Store your own kind of data in Kubernetes.
- Build your own platform objects (like a Database, a Batch, a Backup job).
- Combine with **Operators** to make Kubernetes do work automatically when such a resource appears (see Chapter 44).

---

## 6. Interview note

The instructor says CRDs are usually **not** asked in interviews. They are used mostly for open-source contribution and advanced platform work. But it is very good to understand them.

---

## 7. Summary

- Resource = a thing Kubernetes already knows (Pod, Service, ...).
- **CRD** defines a new resource type. **CR** is an object of that type.
- CRD needs: group, names (plural, singular, kind, shortNames), scope, versions, schema.
- Command flow: `kubectl apply -f crd.yaml` then `kubectl apply -f cr.yaml`, then `kubectl get <plural or shortname>`.
