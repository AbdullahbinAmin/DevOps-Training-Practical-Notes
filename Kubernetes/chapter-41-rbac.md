# Chapter 41: Role Based Access Control (RBAC)

## 1. What is RBAC?

RBAC means **Role Based Access Control**. It decides **who can do what** inside a Kubernetes cluster.

### Simple home example

Think about a house:

- The house has many people. Each person has a **role**.
- One person is the **bread earner** (brings money).
- One person is the **home maker**.
- One person has the role **"bring vegetables"**.

Now map this to Kubernetes:

| Home world | Kubernetes world |
|---|---|
| The house | The cluster |
| A person (Shubham, his sister) | A **user** or **service account** |
| The job "bring vegetables" | A **Role** (a list of allowed actions) |
| Giving the job to a person | A **RoleBinding** |

A user and a role cannot talk to each other directly. They are joined by a **binding**. The same role can be given to many users (for example, both Shubham and his sister can be told to bring vegetables). Each one needs their own binding.

### In one line

RBAC controls **which Kubernetes resources** (pods, deployments, statefulsets, services, and so on) a user can access, and **which actions** (get, list, delete, and so on) they can do on them.

---

## 2. Main building blocks

### 2.1 Service Account and User

- A **Service Account** is like a temporary user. You can name it anything, like `manager`, `admin-user` or `apache-user`. Treat it as a user.
- A **User** is a normal user for the whole cluster.
- Service accounts can live inside a **namespace**. Users are generally cluster level.

### 2.2 Namespace level vs Cluster level

| Level | What you use |
|---|---|
| Inside one namespace | **Role** + **RoleBinding** |
| Whole cluster | **ClusterRole** + **ClusterRoleBinding** |

- **Role**: says what can be done inside one namespace (for example, "can get pods in namespace apache").
- **RoleBinding**: attaches a Role to a user or service account.
- **ClusterRole**: says what can be done across the whole cluster.
- **ClusterRoleBinding**: attaches a ClusterRole to a user or service account.

---

## 3. Why do we need RBAC?

By default, the cluster admin (`kubernetes-admin`) can do **everything**. That is fine for an admin. But imagine:

- You have a very important **production** deployment.
- A new **intern** joins the team.
- If the intern can delete deployments, that is very risky.

So we create a **Role** that allows only the actions the intern needs (for example, only "get pods"). This follows the rule of **least privilege**: give only the access that is really needed.

---

## 4. Useful auth commands

```bash
# Who am I right now?
kubectl auth whoami

# Can I do this action?
kubectl auth can-i get pods
kubectl auth can-i delete deployments -n apache

# Can another user do this action? (use --as)
kubectl auth can-i get pods --as system:serviceaccount:apache:apache-user -n apache
```

`kubectl auth can-i` answers only `yes` or `no`. It is the best way to test your RBAC.

---

## 5. Hands-on example (namespace level)

### Step 1: Create the namespace

`namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: apache
```
```bash
kubectl apply -f namespace.yaml
```

As the admin, `kubectl auth can-i get pods -n apache` gives **yes**, because the admin can do everything.

### Step 2: Create a Role

`role.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: apache-manager
  namespace: apache
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "deployments", "services"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Understand each field:

- **apiVersion / kind**: this object is a `Role`. The API group is `rbac.authorization.k8s.io/v1`.
- **namespace**: a Role works only inside this namespace.
- **rules** has three important parts:
  - **apiGroups**: which API group the resource belongs to.
    - `""` (blank) is the **core group**. Pods and Services are here (apiVersion is just `v1`).
    - `apps` is used for Deployments, ReplicaSets, StatefulSets, DaemonSets (apiVersion `apps/v1`).
    - `batch` is used for Jobs and CronJobs.
    - `*` means all groups.
  - **resources**: which objects (pods, deployments, services, and so on).
  - **verbs**: which actions (the same words you use in `kubectl`).
    - `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`.

```bash
kubectl apply -f role.yaml
```

### Step 3: Create a Service Account

`serviceaccount.yaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: apache-user
  namespace: apache
```
```bash
kubectl apply -f serviceaccount.yaml
```

Right now this service account has **no access** at all. Testing shows:

```bash
kubectl auth can-i get pods -n apache --as system:serviceaccount:apache:apache-user
# no
```

Note the format: `system:serviceaccount:<namespace>:<service-account-name>`.

### Step 4: Create a RoleBinding

`rolebinding.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: apache-role-binding
  namespace: apache
subjects:
  - kind: ServiceAccount
    name: apache-user
    namespace: apache
roleRef:
  kind: Role
  name: apache-manager
  apiGroup: rbac.authorization.k8s.io
```

- **subjects**: WHO gets the access. This is a **list**, so remember the dash (`-`) and correct indentation.
- **roleRef**: WHICH role is given.

```bash
kubectl apply -f rolebinding.yaml
```

> **Note from the video:** in the video the subject was first written as `kind: User`, and the test worked only with `--as apache-user`. The correct way for a service account is `kind: ServiceAccount` with a `namespace`, as shown above. Then `--as system:serviceaccount:apache:apache-user` works.

### Step 5: Test

```bash
kubectl auth can-i get pods -n apache --as system:serviceaccount:apache:apache-user
# yes
kubectl auth can-i get deployments -n apache --as system:serviceaccount:apache:apache-user
# yes
```

---

## 6. Changing permissions (very important)

To limit access, edit the role. Example: allow only `get` on pods:

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get"]
```

**Golden rule:** after you edit a Role YAML, you MUST run `kubectl apply -f role.yaml` again. In the video, the deployment permission still showed "yes" only because the edited role was **not applied**.

Test results after applying:

| Test | Result | Reason |
|---|---|---|
| can-i get pods | yes | verb `get` on `pods` allowed |
| can-i get deployments | no | deployments not in the role |
| can-i delete pods | no | verb `delete` not allowed |
| can-i list pods | no | verb `list` not allowed |

Then add more:

- Add `deployments` and `services` under `resources` with apiGroups `["", "apps"]` and the verb `get`. Deployment access becomes **yes**, but HorizontalPodAutoscaler (HPA) access stays **no** because it is not listed (HPA is in the `autoscaling` group).
- Add `delete` to verbs. Deleting pods becomes **yes**.
- Add `list`. Listing pods becomes **yes**.

---

## 7. Cluster-level RBAC

Use **ClusterRole** and **ClusterRoleBinding** when you need access across the entire cluster, for example for:

- Monitoring the whole cluster
- Reading logs from all namespaces
- Dashboards (next chapter)

Ready-made ClusterRoles exist, like `cluster-admin` (full access), `admin`, `edit`, `view`.

Example ClusterRoleBinding:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user-binding
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

---

## 8. Quick summary

- RBAC = who can do what.
- **Role / RoleBinding** = inside one namespace.
- **ClusterRole / ClusterRoleBinding** = whole cluster.
- **Service Account** = an identity (like a user) for people or apps.
- Roles are made of `apiGroups`, `resources`, `verbs`.
- Always `kubectl apply` after editing.
- Test with `kubectl auth can-i ... --as ...`.
- Keep permissions as small as possible.
