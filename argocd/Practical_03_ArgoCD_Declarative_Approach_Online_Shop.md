# Practical 3: First App Deployment with ArgoCD — Declarative Approach (Online Shop)

## Objective

Write an ArgoCD **Application CRD** (YAML) for the Online Shop app, store it in Git, apply it to the cluster, and see ArgoCD deploy the app **automatically**. Then test auto-sync (change in Git) and self-heal (manual change in the cluster is reverted).

## Theory (short)

* In the Declarative approach, we write the ArgoCD `Application` resource (CRD) in YAML and keep it in Git.
* When we create an app from the UI or CLI, ArgoCD creates the same CRD inside the cluster for us. In the Declarative approach **we write it ourselves**.
* Why this is real GitOps: it is version-controlled, reproducible, auditable, and Git is the source of truth.
* This app uses the image `amitabhdevops/online_shop:latest`.

| Approach | How the app is created | Where config lives |
|---|---|---|
| UI (Practical 2) | ArgoCD Dashboard | Cluster only |
| **Declarative (this practical)** | **Application CRD YAML** | **Git + cluster** |

## Prerequisites

> **Prerequisite from Practical 1:** EC2 server `Running`, Kind cluster `argocd-cluster` with 3 `Ready` nodes, ArgoCD running, ArgoCD CLI installed.
>
> **Prerequisite from Practical 2:** You already forked and cloned `argocd-demos` on your laptop, VS Code and Git are installed, your Git name/email are set, and you can push to your fork (GitHub sign-in or Personal Access Token).

* **On your laptop:** Browser, Git, VS Code, the cloned `argocd-demos` repo
* **On the EC2 server:** Kind cluster, ArgoCD, ArgoCD CLI, kubectl, Git (checked in Step 5)
* **Required ports (Security Group inbound rules):**
  * `8080` → ArgoCD UI (opened in Practical 1)
  * `3000` → Online Shop app (we open it in Step 9)
* **Required repository:** your fork `argocd-demos`, branch `main`
* **Required files:** `online_shop_app.yml` (already in the repo, we edit it in Step 4)

> ⚠️ **Where to run each step.** Every step is marked:
> * 💻 **Laptop terminal** = your own computer
> * 🖥️ **EC2 server** = the server terminal (SSH or EC2 Instance Connect)
> * 🌐 **Browser** = ArgoCD UI or GitHub
> * ☁️ **AWS Console** = AWS website in the browser

## Architecture / Flow

```text
Laptop (edit online_shop_app.yml) → push → GitHub fork (argocd-demos)
Server: kubectl apply Application CRD (namespace: argocd)
      → ArgoCD reads Git path declarative_approach/online_shop
      → Auto-sync → Kind cluster (default namespace): online-shop Deployment + online-shop-service
      → Port-forward 3000 → Browser
```

## Directory Structure (the part we use)

```text
argocd-demos/
└── declarative_approach/
    └── online_shop/
        ├── online_shop_deployment.yml
        ├── online_shop_svc.yml
        └── online_shop_app.yml      # ArgoCD Application CRD
```

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

### 1.2 Check the ArgoCD port-forward

```bash
ps aux | grep "port-forward svc/argocd-server" | grep -v grep
```

**Expected:** One line is printed. If nothing is printed, start it:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address=0.0.0.0 &
```

**What this does:** Forwards port `8080` on the server to the ArgoCD server. Press `Enter` to get the prompt back.

### 1.3 Check the CLI login

```bash
argocd account get-user-info
```

**Expected:** `Logged In: true` and `Username: admin`.

If you are not logged in, get the password and log in again:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

```bash
argocd login <EC2_PUBLIC_IP>:8080 --username admin --password <INITIAL_PASSWORD> --insecure
```

**Replace:**

* `<EC2_PUBLIC_IP>` → Public IPv4 address of your EC2 instance
* `<INITIAL_PASSWORD>` → the password printed by the first command

**Expected:** `'admin:login' logged in successfully`.

### 1.4 Open the ArgoCD UI 🌐

Open `https://<EC2_PUBLIC_IP>:8080` in the browser and log in with `admin` and the password. (Click **Advanced → Proceed** if the browser shows a certificate warning.)

**Expected:** The ArgoCD dashboard opens. If you still have `nginx-app` from Practical 2, that is fine. It does not conflict with this practical.

---

## Step 2 — Open the Repo in VS Code and Check the Files 💻

**Step 1:** Open **VS Code**. Click **File → Open Folder...** and select your `argocd-demos` folder. (If VS Code asks whether you trust the authors, click **Yes, I trust the authors**.)

**Step 2:** In the Explorer panel, open `declarative_approach → online_shop`. You should see 3 files.

**Step 3:** In a laptop terminal, go to the folder and look at the files:

```bash
cd ~/argocd-demos/declarative_approach/online_shop
ls
```

**Replace:** If you cloned the repo in a different folder in Practical 2, use that path instead of `~/argocd-demos`.

**Expected:**

```text
online_shop_app.yml  online_shop_deployment.yml  online_shop_svc.yml
```

```bash
cat online_shop_deployment.yml
cat online_shop_svc.yml
```

**What this does:** Shows the manifests ArgoCD will deploy. Check:

* Deployment named `online-shop`, with label `app: online-shop`, image `amitabhdevops/online_shop:latest`
* A `replicas` value (the chapter text says 2)
* Service named `online-shop-service` on port `3000`

> ⚠️ Verification Required: I could not open the GitHub repo directly (automated access is blocked), so these names, labels and the replica count come from the chapter text. Run the `cat` commands once before class and confirm them. The later steps use `online-shop`, `app=online-shop` and `online-shop-service`.

---

## Step 3 — Add the Cluster to ArgoCD 🖥️ (EC2 server)

> If you already did this in Practical 2 (Step 8), you only need to run 3.1 and 3.3 to confirm, and skip 3.2.

### 3.1 Check the kubectl contexts

```bash
kubectl config get-contexts
```

**Expected:** A context named `kind-argocd-cluster`.

### 3.2 Add the cluster (skip if already added)

```bash
argocd cluster add kind-argocd-cluster --name argocd-cluster --insecure
```

**What this does:** Registers your Kind cluster in ArgoCD with the name `argocd-cluster`.
**Expected:** ArgoCD may ask "Do you want to continue? [y/N]". Type `y` and press `Enter`.

### 3.3 Verify and note the server URL

```bash
argocd cluster list
```

**Expected:** A row for `argocd-cluster` like:

```text
SERVER                          NAME            VERSION  STATUS   ...
https://172.31.x.x:33893        argocd-cluster  ...
```

**Important:** Copy the value in the `SERVER` column of the `argocd-cluster` row (it looks like `https://<EC2_PRIVATE_IP>:33893`). You need it in Step 4. We call it `<ARGOCD_CLUSTER_SERVER_URL>`.

> The status can be `Unknown` until an app is deployed on that cluster. That is normal. If you already deployed `nginx-app` in Practical 2, it may show `Successful`.

---

## Step 4 — Write the Application CRD and Push It to Git 💻

### 4.1 Open the file

In VS Code, open `declarative_approach/online_shop/online_shop_app.yml`.

### 4.2 Put this complete content in the file

Replace everything in the file with the content below:

```yaml
apiVersion: argoproj.io/v1alpha1   # API group for ArgoCD resources
kind: Application                  # Resource type is "Application"
metadata:
  name: online-shop-app            # Name of this ArgoCD application
  namespace: argocd                # Must be created in the 'argocd' namespace
spec:
  project: default                 # ArgoCD Project (logical grouping of apps)
  source:
    repoURL: https://github.com/<YOUR_GITHUB_USERNAME>/argocd-demos.git   # Git repo containing manifests
    targetRevision: main           # Git branch or tag (e.g., main, dev, release-1.0)
    path: declarative_approach/online_shop   # Path inside repo where manifests live
  destination:
    server: <ARGOCD_CLUSTER_SERVER_URL>   # Target cluster API
    namespace: default             # Namespace in which to deploy the app
  syncPolicy:                      # Defines how ArgoCD syncs the app
    automated:                     # Enable auto-sync
      prune: true                  # Delete resources removed from Git
      selfHeal: true               # Fix drift if resources are changed manually
```

**Replace these two values:**

* `<YOUR_GITHUB_USERNAME>` → your GitHub username (your fork, not `Amitabh-DevOps`)
* `<ARGOCD_CLUSTER_SERVER_URL>` → the `SERVER` value you copied in Step 3.3 (for example `https://172.31.x.x:33893`). Do not add quotes.

**Important fields:**

| Field | Meaning |
|---|---|
| `metadata.namespace: argocd` | The Application resource must live in the `argocd` namespace |
| `source.repoURL` | Your Git repo that holds the manifests |
| `source.targetRevision` | Branch to watch (`main`) |
| `source.path` | Folder in the repo with the app manifests |
| `destination.server` | The cluster to deploy to (the one added in Step 3) |
| `destination.namespace` | Namespace for the app (`default`) |
| `syncPolicy.automated` | ArgoCD syncs by itself. No need to click **Sync** |
| `prune: true` | Deletes cluster resources that were removed from Git |
| `selfHeal: true` | Reverts manual changes made in the cluster |

Save the file: `Ctrl + S`.

> YAML uses spaces for indentation. Do **not** use the Tab key. Keep the indentation exactly as above.

### 4.3 Verify the file 💻

```bash
cd ~/argocd-demos/declarative_approach/online_shop
grep -E "repoURL|server" online_shop_app.yml
```

**Expected:** Your real username in `repoURL` and your real server URL in `server`. No `<...>` placeholders left.

### 4.4 Commit and push 💻

```bash
git add online_shop_app.yml
```

**What this does:** Adds the file to the commit.

```bash
git commit -m "Add Online Shop Application CRD"
```

**What this does:** Saves the change in your local Git history.

```bash
git push origin main
```

**What this does:** Sends the commit to your fork on GitHub.
**Expected:** The output ends with something like `main -> main`.

> ⚠️ If Git asks for a login, sign in through the browser window, or use your GitHub username and a Personal Access Token as the password (GitHub does not accept the account password for HTTPS).

**Verify in the browser 🌐:** Open your fork on GitHub, go to `declarative_approach/online_shop/online_shop_app.yml`, and check that it shows your values.

---

## Step 5 — Get the Manifest onto the EC2 Server 🖥️

We push to Git from the laptop, but `kubectl apply` must run on the server (where the cluster is). So we clone your fork on the server.

### 5.1 Check that Git is installed

```bash
git --version
```

**Expected:** A version is printed. If you get `command not found`:

```bash
sudo apt-get update
sudo apt install git -y
```

### 5.2 Clone your fork on the server

```bash
cd ~
git clone https://github.com/<YOUR_GITHUB_USERNAME>/argocd-demos.git
```

**Replace:**

* `<YOUR_GITHUB_USERNAME>` → your GitHub username

**Expected:** `Cloning into 'argocd-demos'...` and ends with `done`. (Your fork is public, so no login is needed.)

> If the folder already exists on the server, run `cd ~/argocd-demos && git pull` instead.

### 5.3 Go to the folder and check the file

```bash
cd ~/argocd-demos/declarative_approach/online_shop
ls
grep -E "repoURL|server" online_shop_app.yml
```

**Expected:** The 3 files are listed, and the file shows **your** username and server URL (the version you pushed in Step 4).

---

## Step 6 — Apply the Application CRD 🖥️

```bash
kubectl apply -f online_shop_app.yml -n argocd
```

**What this does:** Creates the `Application` resource `online-shop-app` in the `argocd` namespace. From now on ArgoCD watches your Git path and deploys the app by itself.
**Expected:**

```text
application.argoproj.io/online-shop-app created
```

---

## Step 7 — Verify

### 7.1 Verify the Application resource 🖥️

```bash
kubectl get applications -n argocd
```

**Expected:** `online-shop-app` is listed. `SYNC STATUS` becomes `Synced` and `HEALTH STATUS` becomes `Healthy` (after a short time). Run the command again if it shows `OutOfSync` or `Progressing`.

### 7.2 Verify in the ArgoCD UI 🌐

**Step 1:** Open the ArgoCD UI and click **Applications**.

**Step 2:** You should see the tile `online-shop-app`.

**Step 3:** Click the tile.

**Expected:** Status **Synced** and **Healthy**. You will see the Deployment, ReplicaSet, pods and the Service. You did **not** click **Sync**, because auto-sync is on.

### 7.3 Verify in Kubernetes 🖥️

```bash
kubectl get pods -n default
kubectl get svc -n default
```

**Expected:**

* Online Shop pods (name starting with `online-shop-`) are `Running`. If it shows `ContainerCreating`, wait and run again (the image download can take a minute).
* A service `online-shop-service` exposing port `3000`.

> You may also see `nginx-...` pods and `nginx-service` from Practical 2. Ignore them.

---

## Step 8 — Open Port 3000 in the Security Group ☁️ (AWS Console)

**Step 1:** Open the AWS Console → **EC2 → Instances**.

**Step 2:** Select `argocd-server`. Click the **Security** tab. Click the **Security group** name (for example `argocd-sg`).

**Step 3:** Click **Edit inbound rules → Add rule**.

**Step 4:** Enter:

* Type: `Custom TCP`
* Port range: `3000`
* Source: `My IP` (or `Anywhere-IPv4` for a classroom demo only)

**Step 5:** Click **Save rules**.

**Expected:** The inbound rules now list port `3000` (along with `22` and `8080`, and `8081` if you did Practical 2).

> The AWS screen may look a little different depending on the console version.

---

## Step 9 — Access the Online Shop App

### 9.1 Port-forward the service 🖥️

```bash
kubectl port-forward svc/online-shop-service 3000:3000 --address=0.0.0.0 &
```

**What this does:** Forwards port `3000` on the EC2 server to port `3000` of `online-shop-service`. `--address=0.0.0.0` allows access from outside. `&` runs it in the background.
**Expected:** `Forwarding from 0.0.0.0:3000 -> 3000`. Press `Enter` to get the prompt back.

### 9.2 Open the app 🌐

Open a new browser tab and go to:

`http://<EC2_PUBLIC_IP>:3000`

**Replace:**

* `<EC2_PUBLIC_IP>` → Public IPv4 address of your EC2 instance. Use `http`, not `https`.

**Expected:** The Online Shop application UI opens.

---

## Step 10 — Test 1: Change in Git (Auto-Sync)

**Goal:** Increase the replicas in Git and see ArgoCD apply it **without clicking Sync**.

### 10.1 Check the current replicas 💻

```bash
cd ~/argocd-demos/declarative_approach/online_shop
grep replicas online_shop_deployment.yml
```

**Expected:** A line like `replicas: 2` (the chapter text says 2).

### 10.2 Edit the manifest ✏️ (VS Code)

**Step 1:** In VS Code, open `declarative_approach/online_shop/online_shop_deployment.yml`.

**Step 2:** Find the `replicas` line under `spec:` and change the number to `4`. Example:

```yaml
spec:
  replicas: 4
```

**Step 3:** Save the file (`Ctrl + S`).

### 10.3 Check the change 💻

```bash
grep replicas online_shop_deployment.yml
```

**Expected:** `replicas: 4`.

### 10.4 Commit and push 💻

```bash
git add online_shop_deployment.yml
git commit -m "Scale Online Shop replicas"
git push origin main
```

**What this does:** Adds the file, saves the change, and sends it to your fork.
**Expected:** The push output ends with `main -> main`.

### 10.5 Observe in ArgoCD 🌐

**Step 1:** Open the ArgoCD UI and click `online-shop-app`.

**Step 2:** Wait for ArgoCD to detect the change and sync by itself.

> ArgoCD checks Git about every 3 minutes by default. To see it immediately, click **Refresh** at the top of the app page. With auto-sync on, ArgoCD then applies the change without clicking **Sync**.

**Expected:** The app goes through **OutOfSync / Syncing** and returns to **Synced** and **Healthy**. The resource tree shows more pods.

### 10.6 Verify 🖥️

```bash
kubectl get pods -n default
```

**Expected:** 4 `online-shop-...` pods are `Running`. In the ArgoCD UI the pod count in the resource tree also increases.

---

## Step 11 — Test 2: Self-Heal (Drift Correction)

**Idea:** Even if someone changes the cluster directly, ArgoCD restores the state that is defined in Git.

### 11.1 Delete a pod manually 🖥️

Open a second terminal to the server, or run the watch command first:

```bash
kubectl get pods -n default -w
```

**What this does:** Shows pod changes live. (Stop it with `Ctrl + C` when done.)

In another terminal (or after stopping the watch), run:

```bash
kubectl delete pod -l app=online-shop -n default
```

**What this does:** Deletes all pods that have the label `app=online-shop`.
**Expected:** The pods go `Terminating`, and new pods are created by the ReplicaSet automatically. In the ArgoCD UI, the app may show **Progressing** for a moment and then **Healthy** again.

> Note: this test shows Kubernetes (ReplicaSet) recreating pods. Test 11.2 shows ArgoCD self-heal.

### 11.2 Scale the deployment manually 🖥️

```bash
kubectl scale deployment online-shop --replicas=1 -n default
```

**What this does:** Changes the replicas directly in the cluster to `1` (Git says `4`).

Check right away:

```bash
kubectl get pods -n default
```

**Expected:** ArgoCD notices the difference (drift) and **automatically scales the Deployment back** to the number written in Git (`4`). Run the command again after a few seconds if you still see fewer pods.

**Verify:**

```bash
kubectl get deployment online-shop -n default
```

**Expected:** `READY` shows `4/4`.

In the ArgoCD UI, `online-shop-app` briefly shows **OutOfSync** and then returns to **Synced** and **Healthy**.

> ⚠️ Verification Required: If the Deployment in the repo is not named `online-shop`, the scale command fails. Check the name with `kubectl get deploy -n default`.

---

## Step 12 — Troubleshooting

### Error 1

```text
error: unable to recognize "online_shop_app.yml": no matches for kind "Application" in version "argoproj.io/v1alpha1"
```

**Reason:** ArgoCD is not installed in the cluster you are connected to (wrong cluster or ArgoCD was not installed).

**Fix:**

```bash
kubectl config current-context
kubectl get pods -n argocd
```

The context should be `kind-argocd-cluster` and the ArgoCD pods should be `Running`. If not, go back to Practical 1.

### Error 2

```text
error converting YAML to JSON: yaml: line ...
```

**Reason:** Wrong indentation or a Tab character in `online_shop_app.yml`.

**Fix:** Open the file in VS Code, use spaces (not Tab) and keep the indentation exactly as in Step 4.2. Push again and run `git pull` on the server (Step 5.2 note), then apply again.

### Error 3

The app shows an error such as **ComparisonError**, or the repo cannot be found.

**Reason:** `repoURL` still has `<YOUR_GITHUB_USERNAME>` or the wrong username, or the `path` / `targetRevision` is wrong.

**Fix:** Correct `repoURL`, `path` and `targetRevision` in `online_shop_app.yml`, push to Git, run `git pull` on the server, then:

```bash
kubectl apply -f online_shop_app.yml -n argocd
```

### Error 4

The app shows an error about the **destination cluster** (cluster not found, or the app cannot deploy).

**Reason:** `destination.server` is wrong. It does not match the `SERVER` value in `argocd cluster list`.

**Fix:**

```bash
argocd cluster list
```

Copy the exact `SERVER` value of `argocd-cluster` into `online_shop_app.yml`, push, `git pull` on the server, and apply again.

### Error 5

Pods are in `ImagePullBackOff` or `ErrImagePull`.

**Reason:** The server cannot download `amitabhdevops/online_shop:latest` (no internet, wrong image name, or a Docker Hub limit).

**Fix:**

```bash
kubectl describe pod <POD_NAME> -n default
```

Replace `<POD_NAME>` with a pod name from `kubectl get pods -n default`. Read the Events at the bottom. Wait and retry if it is a temporary network or rate-limit problem.

### Error 6

The browser cannot open `http://<EC2_PUBLIC_IP>:3000` (timeout).

**Reason:** Port `3000` is not open in the Security Group, the port-forward is not running, or the Public IP changed.

**Fix:** Repeat Step 8. Check the port-forward:

```bash
ps aux | grep "port-forward svc/online-shop-service" | grep -v grep
```

If it is missing, run Step 9.1 again.

### Error 7

```text
bind: address already in use
```

**Reason:** An old port-forward is already using port `3000`.

**Fix:**

```bash
pkill -f "port-forward svc/online-shop-service"
```

Then run Step 9.1 again.

### Error 8

The app does not sync by itself, or self-heal does not revert your manual change.

**Reason:** The `syncPolicy.automated` block is missing or has wrong indentation, or ArgoCD has not checked Git yet.

**Fix:** Check what the cluster has:

```bash
kubectl get application online-shop-app -n argocd -o yaml
```

Look for `syncPolicy` → `automated` → `prune: true` and `selfHeal: true`. If missing, fix `online_shop_app.yml`, push, `git pull` on the server, and apply again. Also click **Refresh** in the ArgoCD UI.

### Error 9

`git push` fails with an authentication error.

**Reason:** GitHub does not accept account passwords for HTTPS push.

**Fix:** Sign in through the browser prompt (Git Credential Manager), or use a Personal Access Token as the password.

---

## Step 13 — Cleanup

> ⚠️ **Do not delete the cluster if you are going to do the next practical.**

### 13.1 Stop the app port-forward (optional) 🖥️

```bash
pkill -f "port-forward svc/online-shop-service"
```

### 13.2 Remove the app from ArgoCD (optional) 🌐

> ⚠️ Because `selfHeal` is on, if you delete the pods or Deployment manually, ArgoCD will create them again. Delete the **application** first.

**Step 1:** In the ArgoCD UI, go to **Applications**.

**Step 2:** Click `online-shop-app`, then click **Delete** (top of the app page).

**Step 3:** Type the app name `online-shop-app` to confirm. Keep the cascade (delete resources) option selected. Click **OK**.

**Expected:** The application and its pods/service are removed.

> ⚠️ Verification Required: The delete dialog wording can differ between ArgoCD versions. If the pods are still there after deleting the app, remove them with `kubectl delete deployment online-shop -n default` and `kubectl delete svc online-shop-service -n default`.

### 13.3 Complete destroy (only when all practicals are finished) 🖥️

```bash
kind delete cluster --name argocd-cluster
```

**What this does:** Deletes the whole Kind cluster (ArgoCD and all apps).

Then remove the extra Security Group rules (`8081`, `3000`) and terminate the EC2 instance (see Practical 1, Step 11.2).

---

## Wrap-Up

* You deployed an app with the **Declarative GitOps** approach.
* The Application CRD is stored in Git, so it is version-controlled, auditable and reproducible.
* Auto-sync applied the Git change by itself, and self-heal reverted the manual change.
* Key takeaway: UI and CLI are good for learning. **Declarative is what you use in production.**

## Final Result

At the end, these must be working:

* `online_shop_app.yml` with your username and cluster URL is pushed to your fork
* Application `online-shop-app` exists in the `argocd` namespace and shows `Synced` and `Healthy`
* Online Shop pods are `Running` and `online-shop-service` exposes port `3000`
* The app opens at `http://<EC2_PUBLIC_IP>:3000`
* A Git change (replicas) is applied automatically
* A manual scale in the cluster is reverted by ArgoCD

## Classroom Safety Check

- [x] Prerequisites from Practical 1 and 2 stated
- [x] Pre-check of cluster, ArgoCD, port-forward and CLI login
- [x] Cluster add (or skip if done) and server URL copied
- [x] Complete Application CRD file with placeholders explained
- [x] Push from laptop, clone on server, apply on server
- [x] Verification in kubectl, UI and browser
- [x] Security Group port 3000 (AWS Console)
- [x] Auto-sync test and self-heal tests
- [x] Troubleshooting and cleanup (with warning not to delete the cluster early)
