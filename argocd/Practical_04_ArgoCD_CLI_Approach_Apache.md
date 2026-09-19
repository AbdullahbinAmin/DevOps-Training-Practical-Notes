# Practical 4: First App Deployment with ArgoCD — CLI Approach (Apache)

## Objective

Log in to ArgoCD (UI first, then CLI), add the Kind cluster, create an application using the `argocd app create` command, sync it, open the Apache HTTPD page in the browser, and test the flow by changing replicas in Git.

## Theory (short)

* In the CLI approach, the command `argocd app create` creates the `Application` resource (CRD) **inside the cluster** for us.
* The app definition lives only in the cluster, not in Git. It is fast and powerful for admins, but it is **not true GitOps**.
* The ArgoCD CLI is only a client. It talks to the ArgoCD **server** API (like `kubectl` talks to the Kubernetes API). If the ArgoCD server is not running, the CLI cannot create or sync apps.

| Approach | How the app is created | Where config lives |
|---|---|---|
| UI (Practical 2) | ArgoCD Dashboard | Cluster only |
| Declarative (Practical 3) | Application CRD YAML | Git + cluster |
| **CLI (this practical)** | **`argocd app create ...`** | **Cluster only** |

## Prerequisites

> **Prerequisite from Practical 1:** EC2 server `Running`, Kind cluster `argocd-cluster` with 3 `Ready` nodes, ArgoCD running, ArgoCD CLI installed.
>
> **Prerequisite from Practical 2:** You already forked and cloned `argocd-demos` on your laptop, VS Code and Git are installed, your Git name/email are set, and you can push to your fork (GitHub sign-in or Personal Access Token).

* **On your laptop:** Browser, Git, VS Code, the cloned `argocd-demos` repo
* **On the EC2 server:** Kind cluster, ArgoCD, ArgoCD CLI, kubectl
* **Required ports (Security Group inbound rules):**
  * `8080` → ArgoCD UI (opened in Practical 1)
  * `8082` → Apache app (we open it in Step 5)
* **Required repository:** your fork `argocd-demos`, branch `main`
* **Required files:** `cli_approach/apache/apache_deployment.yml` and `apache_svc.yml` (already in the repo)

> ⚠️ **Where to run each step.** Every step is marked:
> * 💻 **Laptop terminal** = your own computer
> * 🖥️ **EC2 server** = the server terminal (SSH or EC2 Instance Connect)
> * 🌐 **Browser** = ArgoCD UI or GitHub
> * ☁️ **AWS Console** = AWS website in the browser

## Architecture / Flow

```text
EC2 server: argocd CLI → ArgoCD server API → Application "apache-app" (created in the cluster)
GitHub fork (argocd-demos, path cli_approach/apache) → ArgoCD → Auto-sync
      → Kind cluster (default namespace): Apache Deployment + apache-service
      → Port-forward 8082 → Browser
```

## Directory Structure (the part we use)

```text
argocd-demos/
└── cli_approach/
    └── apache/
        ├── apache_deployment.yml
        └── apache_svc.yml
```

> Note: In this approach there is **no Application YAML** in Git. The application is created by the CLI command in Step 6.

---

## Step 1 — Pre-Check on the EC2 Server 🖥️

Connect to your EC2 server (EC2 Instance Connect or SSH, same as Practical 1).

> ⚠️ If you stopped and started the EC2 instance since the last practical, the **Public IP has changed**. Copy the new **Public IPv4 address** from the EC2 dashboard and use it as `<EC2_PUBLIC_IP>` below.

### 1.1 Check the cluster and ArgoCD

```bash
kubectl get nodes
kubectl get pods -n argocd
```

**Expected:** 3 nodes `Ready`. All ArgoCD pods `Running`.

### 1.2 Check the CLI is installed

```bash
argocd version --client
```

**Expected:** The client version is printed. If you get `command not found`, install the CLI using Practical 1 (Step 8).

### 1.3 Check the ArgoCD port-forward

```bash
ps aux | grep "port-forward svc/argocd-server" | grep -v grep
```

**Expected:** One line is printed. If nothing is printed, start it:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address=0.0.0.0 &
```

**What this does:** Forwards port `8080` on the server to the ArgoCD server. Press `Enter` to get the prompt back.

---

## Step 2 — Open the Repo in VS Code and Check the Files 💻

**Step 1:** Open **VS Code**. Click **File → Open Folder...** and select your `argocd-demos` folder. (If VS Code asks whether you trust the authors, click **Yes, I trust the authors**.)

**Step 2:** In the Explorer panel, open `cli_approach → apache`. You should see 2 files.

**Step 3:** In a laptop terminal, go to the folder and look at the files:

```bash
cd ~/argocd-demos/cli_approach/apache
ls
```

**Replace:** If you cloned the repo in a different folder in Practical 2, use that path instead of `~/argocd-demos`.

**Expected:**

```text
apache_deployment.yml  apache_svc.yml
```

```bash
cat apache_deployment.yml
cat apache_svc.yml
```

**What this does:** Shows the manifests ArgoCD will deploy. Check:

* A Deployment with pods named like `apache-deployment-xxxx`
* A `replicas` value (the chapter text says 3)
* A Service named `apache-service` on port `80`

> ⚠️ Verification Required: I could not open the GitHub repo directly (automated access is blocked), so these names and the replica count come from the chapter text. Run the `cat` commands once before class and confirm them.

---

## Step 3 — Login to ArgoCD (UI first, then CLI)

**Rule from the chapter:** Always log in with the **UI first** to make sure the server is running, then log in with the CLI.

### 3.1 Get the admin password 🖥️

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

**What this does:** Reads the admin password from the secret and decodes it.
**Expected:** A password is printed. Copy it. We call it `<ADMIN_PASSWORD>`.

### 3.2 Login from the browser 🌐

**Step 1:** Open a new browser tab and go to:
`https://<EC2_PUBLIC_IP>:8080`

**Replace:**

* `<EC2_PUBLIC_IP>` → Public IPv4 address of your EC2 instance

> The chapter text shows `http://`, but in our setup (Practical 1) the port-forward uses TLS, so use **`https://`**.

**Step 2:** If the browser shows a certificate warning, click **Advanced → Proceed**.

**Step 3:** Enter:

* Username: `admin`
* Password: `<ADMIN_PASSWORD>`

**Step 4:** Click **Sign In**.

**Expected:** The ArgoCD dashboard opens.

### 3.3 Login from the CLI 🖥️

```bash
argocd login <EC2_PUBLIC_IP>:8080 \
  --username admin \
  --password <ADMIN_PASSWORD> \
  --insecure
```

**Replace:**

* `<EC2_PUBLIC_IP>` → Public IPv4 address of your EC2 instance
* `<ADMIN_PASSWORD>` → the password from Step 3.1

**Why `--insecure`:** Port-forward uses a self-signed TLS certificate.
**Expected:** `'admin:login' logged in successfully`.

> There must be **no space** after the `\` at the end of a line. Or write the whole command on one line.

### 3.4 Verify

```bash
argocd account get-user-info
```

**Expected:** `Logged In: true` and `Username: admin`.

---

## Step 4 — Add the Cluster to ArgoCD 🖥️ (if not already added)

> If you already did this in Practical 2 or 3, only run 4.3 to confirm and copy the server URL. Skip 4.2.

### 4.1 Check the kubectl contexts

```bash
kubectl config get-contexts
```

**Expected:** A context named `kind-argocd-cluster`.

### 4.2 Add the cluster (skip if already added)

```bash
argocd cluster add kind-argocd-cluster --name argocd-cluster --insecure
```

**What this does:** Registers your Kind cluster in ArgoCD with the name `argocd-cluster`.
**Expected:** ArgoCD may ask "Do you want to continue? [y/N]". Type `y` and press `Enter`.

### 4.3 Verify and note the server URL

```bash
argocd cluster list
```

**Expected:** A row for `argocd-cluster`, for example:

```text
SERVER                          NAME            VERSION  STATUS   ...
https://172.31.x.x:33893        argocd-cluster  ...
```

**Important:** Copy the full value in the `SERVER` column of the `argocd-cluster` row (it looks like `https://<EC2_PRIVATE_IP>:33893`). You need it in Step 6. We call it `<ARGOCD_CLUSTER_SERVER_URL>`.

> The status can be `Unknown` until an app is deployed on that cluster. That is normal.

---

## Step 5 — Open Port 8082 in the Security Group ☁️ (AWS Console)

**Step 1:** Open the AWS Console → **EC2 → Instances**.

**Step 2:** Select `argocd-server`. Click the **Security** tab. Click the **Security group** name (for example `argocd-sg`).

**Step 3:** Click **Edit inbound rules → Add rule**.

**Step 4:** Enter:

* Type: `Custom TCP`
* Port range: `8082`
* Source: `My IP` (or `Anywhere-IPv4` for a classroom demo only)

**Step 5:** Click **Save rules**.

**Expected:** The inbound rules list port `8082` (along with `22`, `8080`, and any ports from earlier practicals).

> The AWS screen may look a little different depending on the console version.

---

## Step 6 — Create the Application with the CLI 🖥️

```bash
argocd app create apache-app \
  --repo https://github.com/<YOUR_GITHUB_USERNAME>/argocd-demos.git \
  --path cli_approach/apache \
  --dest-server <ARGOCD_CLUSTER_SERVER_URL> \
  --dest-namespace default \
  --sync-policy automated \
  --self-heal \
  --auto-prune
```

**Replace these two values:**

* `<YOUR_GITHUB_USERNAME>` → your GitHub username (your fork, not `Amitabh-DevOps`)
* `<ARGOCD_CLUSTER_SERVER_URL>` → the full `SERVER` value from Step 4.3, for example `https://172.31.x.x:33893`

> ⚠️ Do **not** type the `<` and `>` characters. Remove them when you replace the values. If you keep them, the shell shows a `syntax error near unexpected token` error.

**Example after replacing (yours will be different):**

```bash
argocd app create apache-app \
  --repo https://github.com/alikhan/argocd-demos.git \
  --path cli_approach/apache \
  --dest-server https://172.31.10.25:33893 \
  --dest-namespace default \
  --sync-policy automated \
  --self-heal \
  --auto-prune
```

**What the flags mean:**

| Flag | Meaning |
|---|---|
| `apache-app` | Name of the ArgoCD application |
| `--repo` | Git repo that holds the manifests (your fork) |
| `--path` | Folder in the repo where the manifests live |
| `--dest-server` | Target cluster (the one added in Step 4) |
| `--dest-namespace` | Namespace to deploy into (`default`) |
| `--sync-policy automated` | Auto-sync is on |
| `--self-heal` | Fixes drift if someone changes or deletes resources manually |
| `--auto-prune` | Removes resources from the cluster if they are removed from Git |

**Expected:** The message `application 'apache-app' created`.

---

## Step 7 — Verify the Application

### 7.1 List the apps 🖥️

```bash
argocd app list
```

**Expected:** `apache-app` is in the list, like this (your cluster URL and repo will be different):

```text
NAME               CLUSTER                      NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  ...  PATH                 TARGET
argocd/apache-app  https://172.31.x.x:33893     default    default  Synced  Healthy  Auto-Prune  ...  cli_approach/apache  
```

If the status is `OutOfSync` or `Progressing`, wait a few seconds and run it again.

### 7.2 Check in the UI 🌐

Open the ArgoCD UI → **Applications**. You should see the tile `apache-app` (it may show it is being created, then **Synced** and **Healthy**).

### 7.3 Check the app details 🖥️

```bash
argocd app get apache-app
```

**Expected:** Shows the repo, path, destination server and namespace, sync status, health status, and a list of resources (Deployment, Service, pods).

---

## Step 8 — Sync the Application 🖥️ (only if needed)

Because auto-sync is on, the app usually syncs by itself. If `argocd app get apache-app` shows **OutOfSync**, run:

```bash
argocd app sync apache-app
```

**What this does:** Applies the manifests from Git to the cluster.
**Expected:** The output shows the resources being synced, and ends with the sync status `Synced` and health `Healthy`.

---

## Step 9 — Verify in Kubernetes 🖥️

```bash
kubectl get pods -n default
kubectl get svc -n default
```

**Expected:**

* Apache pods named like `apache-deployment-xxxxx-xxxxx`, all `Running` (if `ContainerCreating`, wait and run again)
* A service `apache-service` of type `ClusterIP` exposing port `80`

> You may also see pods and services from earlier practicals (for example `nginx-...`, `online-shop-...`). Ignore them.

---

## Step 10 — Access Apache in the Browser

### 10.1 Port-forward the service 🖥️

```bash
kubectl port-forward svc/apache-service 8082:80 --address=0.0.0.0 &
```

**What this does:** Forwards port `8082` on the EC2 server to port `80` of `apache-service`. `--address=0.0.0.0` allows access from outside. `&` runs it in the background.
**Expected:** `Forwarding from 0.0.0.0:8082 -> 80`. Press `Enter` to get the prompt back.

### 10.2 Open the app 🌐

Open a new browser tab and go to:

`http://<EC2_PUBLIC_IP>:8082`

**Replace:**

* `<EC2_PUBLIC_IP>` → Public IPv4 address of your EC2 instance. Use `http`, not `https`.

**Expected:** The default Apache HTTPD test page (usually a page with the text **"It works!"**).

---

## Step 11 — Testing: Make a Change in Git

**Goal:** Change replicas in Git and see ArgoCD apply it.

### 11.1 Check the current replicas 💻

```bash
cd ~/argocd-demos/cli_approach/apache
grep replicas apache_deployment.yml
```

**Expected:** A line like `replicas: 3`.

### 11.2 Edit the manifest ✏️ (VS Code)

**Step 1:** In VS Code, open `cli_approach/apache/apache_deployment.yml`.

**Step 2:** Find the `replicas` line under `spec:` and change `3` to `4`:

```yaml
spec:
  replicas: 4
```

**Step 3:** Save the file (`Ctrl + S`).

### 11.3 Check the change 💻

```bash
grep replicas apache_deployment.yml
```

**Expected:** `replicas: 4`.

### 11.4 Commit and push 💻

```bash
git add apache_deployment.yml
git commit -m "Scale Apache replicas to 4"
git push origin main
```

**What this does:** Adds the file, saves the change, and sends it to your fork.
**Expected:** The push output ends with `main -> main`.

> ⚠️ If Git asks for a login, sign in through the browser window, or use your GitHub username and a Personal Access Token as the password.

### 11.5 Observe in ArgoCD 🖥️

```bash
argocd app get apache-app
```

**Expected:** The app may show `OutOfSync` for a moment, then ArgoCD syncs it automatically (because of `--sync-policy automated`).

> ArgoCD checks Git about every 3 minutes by default. To make it check right now, use the hard refresh:

```bash
argocd app get apache-app --hard-refresh
```

**What this does:** Forces ArgoCD to re-read Git and refresh the app.

### 11.6 Verify 🖥️

```bash
kubectl get pods -n default
```

**Expected:** **4** Apache pods are `Running`.

---

## Step 12 — Common ArgoCD CLI Commands (reference)

| Command | Description |
|---|---|
| `argocd login <host>:<port> --username admin --password <pwd> --insecure` | Log in to the ArgoCD API server |
| `argocd account get-user-info` | Show info about the logged-in user |
| `argocd cluster list` | List clusters registered with ArgoCD |
| `argocd cluster add <context-name>` | Add a cluster from kubeconfig to ArgoCD |
| `argocd repo list` | List connected Git repositories |
| `argocd repo add <repo-url>` | Connect a Git repo to ArgoCD |
| `argocd app list` | List all ArgoCD applications |
| `argocd app create <app-name> --repo <url> --path <dir> ...` | Create a new application |
| `argocd app get <app-name>` | Show details of an application |
| `argocd app sync <app-name>` | Synchronize (deploy) an application |
| `argocd app delete <app-name>` | Delete an application from ArgoCD |
| `argocd app rollback <app-name> <revision>` | Roll back to a previous revision |
| `argocd app set <app-name> --sync-policy automated` | Change app settings (for example, enable auto-sync) |
| `argocd logout <host>` | Log out from the ArgoCD server |

**Tip:** Run `argocd <command> --help` to see all flags of a command.

---

## Step 13 — Troubleshooting

### Error 1

```text
zsh/bash: syntax error near unexpected token `newline'
```

**Reason:** The `<` and `>` characters of the placeholders were typed in the command.

**Fix:** Remove the `<` and `>` and write only the real value. See the example in Step 6.

### Error 2

```text
argocd: command not found
```

**Reason:** The ArgoCD CLI is not installed on the server.

**Fix:** Install it using Practical 1 (Step 8), then run `argocd version --client`.

### Error 3

The CLI says `Unauthenticated`, or it cannot connect (connection refused).

**Reason:** The port-forward is not running, the login session expired, or the Public IP changed.

**Fix:** Repeat Step 1.3 and Step 3.3 (login again).

### Error 4

`argocd app create` fails and says the repository is not accessible or not found.

**Reason:** Wrong `--repo` URL (for example your username is wrong, or you used `Amitabh-DevOps`), or the repo is private.

**Fix:** Correct the URL and run the command again. Open the same URL in the browser to check that it exists. For a private repo, add it first with `argocd repo add <repo-url>` (and your username and token).

### Error 5

`argocd app create` fails and mentions the destination cluster (cluster not configured or not found).

**Reason:** `--dest-server` does not match the `SERVER` value of `argocd cluster list`.

**Fix:**

```bash
argocd cluster list
```

Copy the exact `SERVER` value (with `https://`) and run the create command again.

### Error 6

The app already exists (error when you run `argocd app create` a second time).

**Reason:** `apache-app` was already created with different settings.

**Fix:** Delete it and create again:

```bash
argocd app delete apache-app
```

Type `y` to confirm, then run Step 6 again.

### Error 7

The app stays `OutOfSync` or shows the old replicas after the push.

**Reason:** ArgoCD has not checked Git yet (about 3 minutes), or the push went to a different repo or branch.

**Fix:**

```bash
argocd app get apache-app --hard-refresh
```

Also confirm on GitHub that the change is in your fork, on the `main` branch.

### Error 8

Pods are in `ImagePullBackOff` or `ErrImagePull`.

**Reason:** The server cannot download the Apache image (no internet, or a Docker Hub limit).

**Fix:**

```bash
kubectl describe pod <POD_NAME> -n default
```

Replace `<POD_NAME>` with a pod name from `kubectl get pods -n default`. Read the Events at the bottom. Wait and retry if it is a temporary network problem.

### Error 9

The browser cannot open `http://<EC2_PUBLIC_IP>:8082` (timeout).

**Reason:** Port `8082` is not open in the Security Group, the port-forward is not running, or the Public IP changed.

**Fix:** Repeat Step 5. Check the port-forward:

```bash
ps aux | grep "port-forward svc/apache-service" | grep -v grep
```

If it is missing, run Step 10.1 again.

### Error 10

```text
bind: address already in use
```

**Reason:** An old port-forward is already using port `8082`.

**Fix:**

```bash
pkill -f "port-forward svc/apache-service"
```

Then run Step 10.1 again.

### Error 11

`git push` fails with an authentication error.

**Reason:** GitHub does not accept account passwords for HTTPS push.

**Fix:** Sign in through the browser prompt (Git Credential Manager), or use a Personal Access Token as the password.

---

## Step 14 — Cleanup

> ⚠️ **Do not delete the cluster if you are going to do another practical.**

### 14.1 Stop the Apache port-forward (optional) 🖥️

```bash
pkill -f "port-forward svc/apache-service"
```

### 14.2 Remove the app from ArgoCD (optional) 🖥️

> Because `--self-heal` is on, if you delete the pods or Deployment manually, ArgoCD will create them again. Delete the **application** first.

```bash
argocd app delete apache-app
```

**What this does:** Deletes the application from ArgoCD.
**Expected:** ArgoCD asks for confirmation. Type `y` and press `Enter`.

Verify:

```bash
argocd app list
kubectl get pods -n default
```

**Expected:** `apache-app` is gone from the list and the Apache pods are gone (they may show `Terminating` for a short time).

> ⚠️ Verification Required: By default the CLI delete should also remove the app's resources from the cluster, but please confirm this once before class. If the Apache pods are still there, remove them with `kubectl delete deployment <DEPLOYMENT_NAME> -n default` and `kubectl delete svc apache-service -n default` (get the Deployment name from `kubectl get deploy -n default`).

### 14.3 Complete destroy (only when all practicals are finished) 🖥️

```bash
kind delete cluster --name argocd-cluster
```

**What this does:** Deletes the whole Kind cluster (ArgoCD and all apps).

Then remove the extra Security Group rules (`8081`, `3000`, `8082`) and terminate the EC2 instance (see Practical 1, Step 11.2).

---

## Wrap-Up

* You deployed an app with the **ArgoCD CLI**.
* The CLI is powerful for admins and operators, but the app definition lives only in the cluster (not in Git).
* The CLI needs the ArgoCD server running in the cluster. Always log in with the UI first, then the CLI.
* Real GitOps needs declarative Application CRDs stored in Git (Practical 3).

## Final Result

At the end, these must be working:

* CLI is logged in as `admin` and the cluster `argocd-cluster` is added
* Application `apache-app` (created by the CLI) shows `Synced` and `Healthy`
* Apache pods are `Running` and `apache-service` exposes port `80`
* The Apache page opens at `http://<EC2_PUBLIC_IP>:8082`
* After the Git change, 4 Apache pods are running

## Classroom Safety Check

- [x] Prerequisites from Practical 1 and 2 stated
- [x] Pre-check of cluster, CLI, and port-forward
- [x] Login order: UI first, then CLI
- [x] Cluster add (or skip if done) and server URL copied
- [x] Full `argocd app create` command, placeholders explained, example given
- [x] Verification with CLI, UI, and kubectl
- [x] Security Group port 8082 (AWS Console)
- [x] Git change test with hard refresh
- [x] CLI command reference table
- [x] Troubleshooting and cleanup (with warning not to delete the cluster early)
