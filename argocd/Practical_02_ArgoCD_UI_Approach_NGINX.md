# Practical 2: First App Deployment with ArgoCD — UI Approach (NGINX)

## Objective

Fork and clone the `argocd-demos` repository, connect it to ArgoCD, add the Kind cluster to ArgoCD, create an application from the ArgoCD UI, sync it, open NGINX in the browser, and then test the GitOps flow by changing replicas in Git.

## Chapter 4 Overview (very short)

There are three ways to deploy an app with ArgoCD. **This practical covers only the first one (UI).**

| Approach | How the app is created | Where config lives | Best for |
|---|---|---|---|
| **UI (NGINX)** ← this practical | ArgoCD Dashboard | In the cluster only | Learning, demos |
| CLI (Apache) | `argocd app create ...` | In the cluster only | Operators, scripting |
| Declarative (Online Shop) | Application CRD YAML | Git + cluster | Production (true GitOps) |

**Key point of the UI approach:** the app definition is created from the dashboard and lives only in the cluster (not in Git). It is fast for demos, but it is not true GitOps.

## Prerequisites

> **Prerequisite from Practical 1:** Make sure all of these are already working on your EC2 server:
> * EC2 server `argocd-server` is `Running`
> * Kind cluster `argocd-cluster` with 3 `Ready` nodes
> * ArgoCD is running in the `argocd` namespace
> * ArgoCD CLI is installed

* **On your laptop:**
  * Browser
  * **Git** installed (official download page: `https://git-scm.com/downloads`)
  * **VS Code** installed (official download page: `https://code.visualstudio.com/download`)
  * A **GitHub account**
* **On the EC2 server:** Kind cluster, ArgoCD, ArgoCD CLI, kubectl (from Practical 1)
* **Required ports (Security Group inbound rules):**
  * `8080` → ArgoCD UI (already opened in Practical 1)
  * `8081` → NGINX app (we open it in Step 9)
* **Required repository:** `argocd-demos` (we fork it in Step 1)
* **Branch used:** `main`

> ⚠️ **Where to run each step.** Every step is marked with where you do it:
> * 💻 **Laptop terminal** = your own computer
> * 🖥️ **EC2 server** = the server terminal (SSH or EC2 Instance Connect)
> * 🌐 **Browser** = ArgoCD UI or GitHub
> * ☁️ **AWS Console** = browser, AWS website

## Architecture / Flow

```text
GitHub (your fork: argocd-demos)  →  ArgoCD (connected repo + added cluster)
        →  Application "nginx-app" (created from UI)  →  Sync
        →  Kind cluster (default namespace): nginx Deployment + nginx-service
        →  Port-forward 8081  →  Browser
```

## Directory Structure (the part we use)

```text
argocd-demos/
└── ui_approach/
    └── nginx/
        ├── nginx_deployment.yml
        └── nginx_svc.yml
```

> The repo has other folders too (for the CLI and Declarative approaches). We do not use them in this practical.

---

## Step 1 — Fork the Repository 🌐 (GitHub UI)

**What to do:** Make your own copy of the demo repo. You need your own copy because you will push a change to it later.

**Step 1:** Open the browser and sign in to GitHub.

**Step 2:** Go to this URL:
`https://github.com/Amitabh-DevOps/argocd-demos`

**Step 3:** Click **Fork** (top right).

**Step 4:** On the "Create a new fork" page:

* Owner: your GitHub account
* Repository name: `argocd-demos` (keep it the same)

**Step 5:** Click **Create fork**.

**Expected:** A new page opens with the address `https://github.com/<YOUR_GITHUB_USERNAME>/argocd-demos`. Your username is shown at the top-left of the repo name.

> The GitHub screen may look a little different depending on the GitHub version.

---

## Step 2 — Pre-Check on the Laptop 💻

```bash
git --version
```

**What this does:** Checks that Git is installed.
**Expected:** A version like `git version 2.x.x`. If you get `command not found`, install Git from the official download page above and open a new terminal.

Check if your Git name and email are set (needed for commits):

```bash
git config --global user.name
git config --global user.email
```

**Expected:** Both commands print a value. If any of them prints nothing, set them:

```bash
git config --global user.name "<YOUR_NAME>"
git config --global user.email "<YOUR_GITHUB_EMAIL>"
```

**Replace:**

* `<YOUR_NAME>` → your name (for example `Ali Khan`). Keep the double quotes.
* `<YOUR_GITHUB_EMAIL>` → the email of your GitHub account. Keep the double quotes.

---

## Step 3 — Clone Your Fork to the Laptop 💻

### 3.1 Choose a folder

```bash
cd ~
```

**What this does:** Goes to your home folder. (You can use any folder you like. On Windows PowerShell you can use `cd $HOME`.)

### 3.2 Clone

```bash
git clone https://github.com/<YOUR_GITHUB_USERNAME>/argocd-demos.git
```

**Replace:**

* `<YOUR_GITHUB_USERNAME>` → your GitHub username (**your** fork, not `Amitabh-DevOps`)

**What this does:** Downloads your fork into a new folder `argocd-demos`.
**Expected:** `Cloning into 'argocd-demos'...` and ends with `done`.

### 3.3 Go into the repo

```bash
cd argocd-demos
```

### 3.4 Go to the NGINX manifest folder and check the files

```bash
cd ui_approach/nginx
ls
```

**Expected:**

```text
nginx_deployment.yml  nginx_svc.yml
```

Look at the files:

```bash
cat nginx_deployment.yml
cat nginx_svc.yml
```

**What this does:** Shows the manifests that ArgoCD will deploy. In the Deployment, find the line `replicas: 3`. The Service is named `nginx-service` and uses port `80`.

> ⚠️ Verification Required: I could not open the GitHub repo directly (automated access is blocked), so these file details (3 replicas, Deployment name starting with `nginx-deployment`, Service `nginx-service` of type ClusterIP on port 80) come from the chapter text. Please run the two `cat` commands once before class and confirm them.

---

## Step 4 — Open the Repo in VS Code 💻 (GUI)

**Step 1:** Open **VS Code**.

**Step 2:** Click **File → Open Folder...**

**Step 3:** Select the `argocd-demos` folder you cloned. Click **Select Folder** (or **Open**).

**Step 4:** If VS Code asks "Do you trust the authors of the files in this folder?", click **Yes, I trust the authors**.

**Step 5:** In the left Explorer panel, open `ui_approach → nginx`. You will see `nginx_deployment.yml` and `nginx_svc.yml`.

**Why:** We will use VS Code later to change the manifest and push it to Git.

---

## Step 5 — Pre-Check ArgoCD on the EC2 Server 🖥️

Connect to your EC2 server (same way as Practical 1: EC2 Instance Connect or SSH).

> ⚠️ If you stopped and started the EC2 instance since Practical 1, the **Public IP has changed**. Copy the new **Public IPv4 address** from the EC2 dashboard and use it everywhere below as `<EC2_PUBLIC_IP>`.

### 5.1 Check the cluster and ArgoCD

```bash
kubectl get nodes
kubectl get pods -n argocd
```

**Expected:** 3 nodes `Ready`. All ArgoCD pods `Running`.

If the Kind cluster is not there (for example the server was recreated), go back to Practical 1.

### 5.2 Check the port-forward

```bash
ps aux | grep "port-forward" | grep -v grep
```

**Expected:** One line that contains `port-forward svc/argocd-server`. If nothing is printed, start it:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address=0.0.0.0 &
```

**What this does:** Forwards port `8080` on the server to the ArgoCD server. Press `Enter` to get the prompt back.

### 5.3 Get the admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

**Expected:** The admin password is printed. Copy it.

### 5.4 Check the CLI login

```bash
argocd account get-user-info
```

**Expected:** `Logged In: true` and `Username: admin`.

If it says you are not logged in (or the session expired), log in again:

```bash
argocd login <EC2_PUBLIC_IP>:8080 --username admin --password <INITIAL_PASSWORD> --insecure
```

**Replace:**

* `<EC2_PUBLIC_IP>` → Public IPv4 address of your EC2 instance
* `<INITIAL_PASSWORD>` → the password from Step 5.3

**Expected:** `'admin:login' logged in successfully`.

---

## Step 6 — Log in to the ArgoCD UI 🌐

**Step 1:** Open a new browser tab and go to:
`https://<EC2_PUBLIC_IP>:8080`

> The chapter text shows `http://`, but in our setup (Practical 1) the port-forward uses TLS, so use **`https://`**.

**Step 2:** If the browser shows a security warning (self-signed certificate), click **Advanced → Proceed**.

**Step 3:** Enter:

* Username: `admin`
* Password: the password from Step 5.3

**Step 4:** Click **Sign In**.

**Expected:** The ArgoCD dashboard opens (Applications page, with no applications yet).

---

## Step 7 — Connect Your Git Repository 🌐 (ArgoCD UI)

**Step 1:** In the left menu, click **Settings**.

**Step 2:** Click **Repositories**.

**Step 3:** Click **Connect Repo**.

**Step 4:** Choose the connection method: **VIA HTTPS**.

**Step 5:** Fill in:

* Type: `git`
* Project: `default`
* Repository URL: `https://github.com/<YOUR_GITHUB_USERNAME>/argocd-demos.git`
* Username / Password: leave **empty** (your fork is public)

**Replace:**

* `<YOUR_GITHUB_USERNAME>` → your GitHub username (your fork)

**Step 6:** Click **Connect**.

**Expected:** Your repo appears under **Connected Repositories** with **Connection Status: Successful** (green tick).

> The ArgoCD screen and option names can differ a little between ArgoCD versions. If your repo is private, you must also enter your GitHub username and a Personal Access Token as the password.

---

## Step 8 — Add the Cluster to ArgoCD 🖥️ (EC2 server)

### 8.1 Check the kubectl contexts

```bash
kubectl config get-contexts
```

**What this does:** Lists all cluster contexts in your kubeconfig.
**Expected:** A context named `kind-argocd-cluster` (marked with `*` as the current one).

### 8.2 Add the cluster

```bash
argocd cluster add kind-argocd-cluster --name argocd-cluster --insecure
```

**What this does:** Registers your Kind cluster in ArgoCD with the name `argocd-cluster`. ArgoCD creates a service account for itself inside the cluster.

**Expected:** ArgoCD shows a warning and asks a question like "Do you want to continue? [y/N]". Type `y` and press `Enter`. At the end you should see a message that the cluster was added.

> ⚠️ Verification Required: The confirmation question is normal ArgoCD CLI behavior but is not written in the chapter. The exact text can be a little different in your ArgoCD version.

### 8.3 Verify

```bash
argocd cluster list
```

**Expected (before the first app):**

```text
SERVER                          NAME            VERSION  STATUS   MESSAGE                                                  PROJECT
https://172.31.x.x:33893        argocd-cluster                Unknown  ...
https://kubernetes.default.svc  in-cluster                    Unknown  Cluster has no applications and is not being monitored.
```

The `SERVER` value of `argocd-cluster` is `https://<EC2_PRIVATE_IP>:33893` (the address from your `kind-config.yaml`).

> **Note:** The status `Unknown` is normal for now. It changes to `Successful` after you deploy your first application (Step 11).

### 8.4 Check in the UI (optional) 🌐

In the ArgoCD UI go to **Settings → Clusters**. You should see `argocd-cluster` in the list.

---

## Step 9 — Open Port 8081 in the Security Group ☁️ (AWS Console)

**What to do:** Allow the browser to reach the NGINX app on port `8081` (we need this in Step 12).

**Step 1:** Open the AWS Console → **EC2 → Instances**.

**Step 2:** Select `argocd-server`. Click the **Security** tab. Click the **Security group** name (for example `argocd-sg`).

**Step 3:** Click **Edit inbound rules → Add rule**.

**Step 4:** Enter:

* Type: `Custom TCP`
* Port range: `8081`
* Source: `My IP` (or `Anywhere-IPv4` for a classroom demo only)

**Step 5:** Click **Save rules**.

**Expected:** The inbound rules now list `22`, `8080` and `8081`.

---

## Step 10 — Create the Application in ArgoCD UI 🌐

**Step 1:** In the ArgoCD UI, click **Applications** in the left menu.

**Step 2:** Click **+ New App**.

**Step 3:** Fill in the form:

**General**

* Application Name: `nginx-app`
* Project Name: `default`
* Sync Policy: `Manual` (leave it as it is)

**Source**

* Repository URL: select your connected repo from the list (`https://github.com/<YOUR_GITHUB_USERNAME>/argocd-demos.git`)
* Revision: `main`
* Path: `ui_approach/nginx`

**Destination**

* Cluster URL: select the cluster you added (`https://<EC2_PRIVATE_IP>:33893`, name `argocd-cluster`). **Do not** select `https://kubernetes.default.svc`.
* Namespace: `default`

**Step 4:** Click **Create** (top of the form).

**Expected:** A new tile `nginx-app` appears on the Applications page with status **Missing** and **OutOfSync**.

> The form layout (sections, field names) can differ a little between ArgoCD versions.

---

## Step 11 — Sync the Application 🌐

**Step 1:** Click the `nginx-app` tile to open it.

**Step 2:** Status shows **OutOfSync**. This means Git has manifests that are not yet in the cluster.

**Step 3:** Click **Sync**, then click **Synchronize**.

**What this does:** ArgoCD applies `nginx_deployment.yml` and `nginx_svc.yml` into the `default` namespace of your cluster.

**Expected:** After a few seconds, the status changes to **Synced** and **Healthy**. You will see the Deployment, ReplicaSet, pods and the Service in the resource tree.

### Check the cluster status again 🖥️

```bash
argocd cluster list
```

**Expected:** The `argocd-cluster` row now shows `Successful` (and version like `1.33`).

---

## Step 12 — Verify the Deployment 🖥️

```bash
kubectl get pods -n default
kubectl get svc -n default
```

**Expected:**

* NGINX pods named like `nginx-deployment-xxxxx-xxxxx`, all `Running` (3 pods at this point)
* A service `nginx-service` of type `ClusterIP` exposing port `80` (you will also see the default `kubernetes` service. Ignore it.)

---

## Step 13 — Access NGINX in the Browser

### 13.1 Port-forward the NGINX service 🖥️

```bash
kubectl port-forward svc/nginx-service 8081:80 --address=0.0.0.0 &
```

**What this does:** Forwards port `8081` on the EC2 server to port `80` of `nginx-service`. `--address=0.0.0.0` allows access from outside. `&` runs it in the background.
**Expected:** `Forwarding from 0.0.0.0:8081 -> 80`. Press `Enter` to get the prompt back.

### 13.2 Open the app 🌐

Open a new browser tab and go to:

`http://<EC2_PUBLIC_IP>:8081`

**Replace:**

* `<EC2_PUBLIC_IP>` → Public IPv4 address of your EC2 instance. Use `http`, not `https`, for this URL.

**Expected:** The default **"Welcome to nginx!"** page.

---

## Step 14 — Testing: Make a Change in Git (GitOps flow)

**Goal:** Change replicas from `3` to `5` in Git and see ArgoCD detect it.

### 14.1 Edit the manifest ✏️ (VS Code)

**Step 1:** In VS Code, open `ui_approach/nginx/nginx_deployment.yml`.

**Step 2:** Find this part:

```yaml
spec:
  replicas: 3
```

**Step 3:** Change it to:

```yaml
spec:
  replicas: 5
```

**Step 4:** Save the file (`Ctrl + S`).

### 14.2 Check the change 💻

In the laptop terminal:

```bash
cd ~/argocd-demos/ui_approach/nginx
grep replicas nginx_deployment.yml
```

**Replace:** If you cloned in a different folder in Step 3.1, use that path instead of `~/argocd-demos`.

**Expected:** `replicas: 5`.

### 14.3 Commit and push 💻

```bash
git add nginx_deployment.yml
```

**What this does:** Adds the changed file to the commit.

```bash
git commit -m "Increase replicas to 5"
```

**What this does:** Saves the change in your local Git history.
**Expected:** `1 file changed, 1 insertion(+), 1 deletion(-)`.

```bash
git push origin main
```

**What this does:** Sends the commit to your fork on GitHub.
**Expected:** Ends with something like `main -> main`.

> ⚠️ **GitHub login:** GitHub does **not** accept your account password for HTTPS push. When Git asks for authentication, sign in through the browser window (Git Credential Manager), or enter your GitHub username and a **Personal Access Token** as the password.

**Verify:** Open `https://github.com/<YOUR_GITHUB_USERNAME>/argocd-demos` in the browser, go to `ui_approach/nginx/nginx_deployment.yml`, and confirm it shows `replicas: 5`.

### 14.4 Observe ArgoCD 🌐

**Step 1:** Go to the ArgoCD UI and click `nginx-app`.

**Step 2:** Wait for the status to change to **OutOfSync**.

> ArgoCD checks Git about every 3 minutes by default. To see it immediately, click **Refresh** (top of the app page).

**Step 3:** Click **Sync → Synchronize**.

**Expected:** Status becomes **Synced** and **Healthy**.

### 14.5 Verify the change 🖥️

```bash
kubectl get pods -n default
```

**Expected:** **5** NGINX pods are `Running`. In the ArgoCD UI, the `nginx-app` resource tree now shows 5 pods.

---

## Step 15 — Troubleshooting

### Error 1

```text
Unauthenticated
```

(or the CLI says it cannot connect / connection refused to `localhost:8080` or `<EC2_PUBLIC_IP>:8080`)

**Reason:** The ArgoCD CLI session expired, the port-forward is not running, or the Public IP changed.

**Fix:** Repeat Step 5.2 to 5.4.

### Error 2

The repo shows **Connection Status: Failed** in ArgoCD.

**Reason:** Wrong Repository URL (for example you used `Amitabh-DevOps` instead of your own username), a typo, or the repo is private.

**Fix:** In **Settings → Repositories**, remove the repo and connect again with your fork URL from Step 7. For a private repo, add your username and a Personal Access Token.

### Error 3

`argocd cluster add` fails and says the context does not exist.

**Reason:** The context name is different from `kind-argocd-cluster`.

**Fix:**

```bash
kubectl config get-contexts
```

Use the exact name from the `NAME` column in the `argocd cluster add` command.

### Error 4

The app shows an error like **"app path does not exist"**, or a **ComparisonError**.

**Reason:** Wrong `Path`, wrong `Revision`, or the wrong repo was selected.

**Fix:** In the app page click **App Details → Edit**. Check:

* Path: `ui_approach/nginx`
* Revision: `main`
* Repository URL: your fork

Click **Save**, then **Refresh**.

### Error 5

The app stays **OutOfSync** after you pushed the change, or shows the old replicas.

**Reason:** ArgoCD has not checked Git yet (about 3 minutes), or the push went to a different repo or branch.

**Fix:** Click **Refresh** in the app page. Confirm the change is visible on GitHub in your fork, on the `main` branch.

### Error 6

The browser cannot open `http://<EC2_PUBLIC_IP>:8081` (timeout).

**Reason:** Port `8081` is not open in the Security Group, the NGINX port-forward is not running, or the Public IP changed.

**Fix:** Repeat Step 9. Check the port-forward:

```bash
ps aux | grep "port-forward" | grep -v grep
```

If the `nginx-service` port-forward is missing, run Step 13.1 again.

### Error 7

```text
bind: address already in use
```

**Reason:** An old port-forward is already using port `8081` (or `8080`).

**Fix:**

```bash
pkill -f "port-forward svc/nginx-service"
```

Then run Step 13.1 again. (For port `8080`, see Practical 1 troubleshooting.)

### Error 8

`git push` fails with an authentication error.

**Reason:** GitHub does not accept account passwords for HTTPS push.

**Fix:** Sign in through the browser prompt (Git Credential Manager), or use a Personal Access Token as the password.

---

## Step 16 — Cleanup

> ⚠️ **Do not delete the cluster if you are going to do the next practical.** The next practicals use the same Kind cluster and ArgoCD.

### 16.1 Stop the NGINX port-forward (optional) 🖥️

```bash
pkill -f "port-forward svc/nginx-service"
```

### 16.2 Complete destroy (only when all practicals are finished) 🖥️

```bash
kind delete cluster --name argocd-cluster
```

**What this does:** Deletes the whole Kind cluster (ArgoCD and NGINX with it).

Then remove the `8081` rule from the Security Group and terminate the EC2 instance (see Practical 1, Step 11.2).

---

## Wrap-Up

* You deployed an app through the ArgoCD **UI**.
* The UI approach is **imperative**: fast for demos, but the app definition lives only in the cluster.
* Real GitOps needs declarative Application CRDs stored in Git (covered in a later practical).

## Final Result

At the end, these must be working:

* Your fork `argocd-demos` is cloned on your laptop
* The repo is connected in ArgoCD (status Successful)
* The cluster `argocd-cluster` is added in ArgoCD (status Successful after the first sync)
* Application `nginx-app` is `Synced` and `Healthy`
* NGINX opens at `http://<EC2_PUBLIC_IP>:8081`
* After the Git change, 5 NGINX pods are running

## Classroom Safety Check

- [x] Prerequisite from Practical 1 clearly stated
- [x] Fork (GitHub UI), clone, Git identity, VS Code steps
- [x] Pre-check of ArgoCD, port-forward, CLI login, password
- [x] ArgoCD UI steps: connect repo, create app, sync
- [x] Cluster add with CLI and verification
- [x] Security Group port 8081 (AWS Console)
- [x] NGINX access and verification
- [x] GitOps test: edit, commit, push, refresh, sync, verify 5 pods
- [x] Troubleshooting and cleanup (with warning not to delete the cluster early)
