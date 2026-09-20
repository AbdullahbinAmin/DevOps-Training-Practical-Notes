# Practical 5: Observability, Monitoring, Logging and Alerting (Prometheus + Grafana + Alertmanager + Loki)

## Objective

Build a complete observability lab **from scratch** on one EC2 server using Docker Compose:

* **Metrics:** Prometheus collects metrics from the server (Node Exporter), the containers (cAdvisor), and a demo web app.
* **Dashboards:** Grafana shows the metrics in dashboards.
* **Alerting:** Prometheus alert rules + Alertmanager raise alerts when something is wrong.
* **Logging:** Grafana Alloy collects container logs and sends them to Loki. You read the logs in Grafana.

At the end you will **break things on purpose** (stop the app, create errors, create CPU load) and watch monitoring, logs and alerts react.

---

# PART A — THEORY (read this first, 10 minutes)

## A1. Monitoring vs Observability

| | Monitoring | Observability |
|---|---|---|
| Question it answers | "**Is** something wrong?" | "**Why** is it wrong?" |
| How | Watch known numbers and alert on them (CPU > 80%) | Explore data (metrics + logs + traces) to find unknown problems |
| Example | Alert: "Website error rate is high" | Look at logs and metrics to find "the database is slow after the 3 PM deploy" |

**Easy example (car):**

* Dashboard lights (fuel, engine temperature) = **monitoring**. They tell you something is wrong.
* The mechanic connecting a scanner and reading detailed data = **observability**. It tells you why.

Monitoring is a part of observability. You need both.

## A2. The Three Pillars of Observability

| Pillar | What it is | Example | Tool in this lab |
|---|---|---|---|
| **Metrics** | Numbers measured over time | CPU 45%, 120 requests/second | Prometheus |
| **Logs** | Text lines written by apps | `ERROR Simulated error on /error` | Loki (collected by Alloy) |
| **Traces** | The path of one request across many services | Frontend → API → Database (took 300 ms) | *Not in this lab* (tools: Tempo, OpenTelemetry). Mentioned for your next step. |

**How they work together:** An alert (metric) says "errors are high". You open the logs to see the error message. A trace (if you have it) shows which service was slow.

## A3. What to Monitor: The Four Golden Signals

Google SRE recommends watching these four for any service:

| Signal | Meaning | Example in this lab |
|---|---|---|
| **Latency** | How long requests take | `http_request_duration_seconds` (p95) |
| **Traffic** | How many requests | `http_requests_total` rate |
| **Errors** | How many requests fail | Requests with status 5xx |
| **Saturation** | How full the resources are | CPU, memory, disk usage |

Two easy methods to remember:

* **USE** (for infrastructure): **U**tilization, **S**aturation, **E**rrors. Example: CPU utilization.
* **RED** (for apps): **R**ate, **E**rrors, **D**uration.

## A4. How Prometheus Works

Prometheus is a **metrics database + collector**.

```text
Targets (apps/exporters)  <--- Prometheus scrapes (pulls) /metrics every 15s
        |
        v
Prometheus stores data (time series database)  --->  You query with PromQL
        |
        v
Alert rules are checked  --->  Alertmanager sends notifications
```

Key ideas:

* **Pull model:** Prometheus **pulls** (scrapes) metrics from a `/metrics` HTTP page of each target. Apps do not push.
* **Target:** Anything Prometheus scrapes (for example `demo-app:8000`).
* **Exporter:** A small program that exposes metrics of something that does not have a `/metrics` page. Node Exporter (for the Linux server) and cAdvisor (for containers) are exporters.
* **Time series:** A metric name + labels + values over time. Example:
  `http_requests_total{endpoint="/", method="GET", status="200"}`
* **Labels:** Key/value tags on a metric. They let you filter and group (`status="500"`).
* **`up` metric:** Prometheus creates `up` for every target. `1` = scrape worked, `0` = target is down.
* **PromQL:** The query language of Prometheus.

### Metric types

| Type | Meaning | Example | How to use |
|---|---|---|---|
| **Counter** | A number that only goes up (resets on restart) | `http_requests_total` | Use `rate()` to see "per second" |
| **Gauge** | A number that goes up and down | `node_memory_MemAvailable_bytes` | Use directly |
| **Histogram** | Counts values in buckets (good for latency) | `http_request_duration_seconds_bucket` | Use `histogram_quantile()` for p95 |

### PromQL cheat sheet (we use these in the lab)

| Query | Meaning |
|---|---|
| `up` | Is each target up? (1 = yes) |
| `rate(http_requests_total[1m])` | Requests per second, averaged over the last 1 minute |
| `sum(rate(http_requests_total[1m])) by (endpoint)` | Requests per second, grouped by endpoint |
| `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[1m])) by (le))` | 95th percentile latency (95% of requests were faster than this) |

## A5. How Grafana Works

Grafana is a **visualization tool**. It does not store data. It **connects to data sources** (Prometheus, Loki) and draws them.

* **Data source:** A connection to Prometheus or Loki.
* **Dashboard:** A page with many panels.
* **Panel:** One graph, gauge, table or log view. Each panel has a query.
* **Explore:** A page to run queries by hand (great for logs and troubleshooting).
* **Provisioning:** Data sources defined in a file, so they are ready when Grafana starts (we do this).

## A6. How Alerting Works

```text
Prometheus alert rule (expr + for)  -->  Alert becomes Pending  -->  Firing
                                                                       |
                                                                       v
                                            Alertmanager: group, route, silence  -->  Email / Slack / Webhook
```

* **Alert rule:** An expression that becomes true when something is wrong, for example `up == 0`.
* **`for: 1m`:** The condition must stay true for 1 minute before the alert fires. This avoids false alarms from tiny spikes.
* **Alert states:** `Inactive` (fine) → `Pending` (condition true, waiting for `for`) → `Firing` (alert sent).
* **Alertmanager:** Receives firing alerts from Prometheus. It **groups** similar alerts, **routes** them to a receiver (email, Slack, PagerDuty), and supports **silences** (mute for maintenance) and **inhibition** (do not send a small alert if a big one already fired).
* Prometheus **detects**. Alertmanager **notifies**.

## A7. How Logging Works (Loki + Alloy)

* **Loki** stores logs. It is like Prometheus, but for logs. It indexes only the **labels** (like `container="demo-app"`), not the full text. This makes it small and cheap.
* **Alloy** (Grafana Alloy) is the **collector**. It reads Docker container logs and sends them to Loki.
* **Promtail** was the old collector. It is deprecated, so we use **Alloy**.
* **LogQL** is the Loki query language. Example: `{container="demo-app"} |= "ERROR"` = "logs of demo-app that contain ERROR".

```text
Container stdout/stderr --> Docker --> Alloy (reads via Docker socket) --> Loki --> Grafana Explore
```

## A8. Our Lab Architecture

```text
                       Browser (you)
                            |
        +-------------------+---------------------+
        |                   |                     |
   Grafana :3000     Prometheus :9090      Alertmanager :9093
        |   \                | \                  ^
        |    \               |  \ alerts ---------+
        |     \              v
        |      \      scrapes (pull) every 15s
        |       \       |        |          |            |
        |        \  node-exporter  cadvisor   demo-app    alertmanager
        |         \  (server)     (containers) :8000 /metrics
        |
        +---- Loki :3100  <----  Alloy  <---- Docker logs of all containers
```

| Component | Job | Port (host) |
|---|---|---|
| demo-app | Small web app that exposes `/metrics` and writes logs | 8000 |
| Prometheus | Collects and stores metrics, checks alert rules | 9090 |
| Alertmanager | Handles alerts | 9093 |
| Node Exporter | Server metrics (CPU, RAM, disk) | internal only |
| cAdvisor | Container metrics (CPU, RAM per container) | internal only |
| Grafana | Dashboards and log explorer | 3000 |
| Loki | Log storage | 3100 (localhost only) |
| Alloy | Log collector | 12345 (localhost only) |

---

# PART B — PRACTICAL

## Prerequisites

* **AWS account** (permission to launch EC2)
* **Browser**
* **Laptop terminal** (only if you connect by SSH; not needed for the browser connect option)
* **Server (created in Step 1):**
  * OS: Ubuntu Server 24.04 LTS (22.04 LTS also works)
  * Instance type: `t3.medium` (2 vCPU, 4 GB RAM). Use `t3.large` if you want it faster.
  * Disk: 30 GiB
* **Required ports (Security Group inbound rules):**
  * `22` → SSH
  * `3000` → Grafana
  * `9090` → Prometheus
  * `9093` → Alertmanager
  * `8000` → Demo app
* **Not needed from earlier practicals:** This practical is **independent**. You do **not** need Kind or ArgoCD. Use a **new** EC2 server. (If you reuse the old server, delete the Kind cluster first: `kind delete cluster --name argocd-cluster`.)

### Versions used (checked in September 2026)

| Software | Version | Notes |
|---|---|---|
| Ubuntu | 24.04 LTS | 22.04 LTS also supported by Docker |
| Docker Engine + Compose plugin | Latest from Docker's official apt repo (Engine 29.x) | Installed in Step 3 |
| Prometheus | `v3.13.0` | This is an LTS release. Latest at the time was 3.14.0. |
| Alertmanager | `v0.33.1` | Newer 0.34.x exists |
| Node Exporter | `v1.12.1` | Latest at the time |
| cAdvisor | `v0.60.5` | Image from `ghcr.io`. **Use a new version.** Old cAdvisor versions cannot see containers on Docker 29. |
| Grafana | `12.4.11` | Newer 13.2.x exists. The screens in this guide are for 12.x. |
| Loki | `3.6.17` | |
| Grafana Alloy | `v1.19.2` | |
| Python (demo app) | 3.12 (image `python:3.12-slim`) | Flask 3.0.3, prometheus-client 0.20.0 |

> ⚠️ **Version warning:** If you change to newer versions (for example Grafana 13.x), configuration may still work, but Grafana screens and menu names can look different.
>
> ⚠️ Verification Required: Please run this whole guide once before class. Image pulls need internet access on the server.

## Legend (where to run each step)

* ☁️ **AWS Console** = AWS website in the browser
* 🖥️ **Server** = the EC2 server terminal
* 💻 **Laptop** = your own computer terminal
* 🌐 **Browser** = Grafana / Prometheus / Alertmanager pages

## Final Folder Structure (we create this in Steps 4 to 10)

```text
observability-lab/
├── .env
├── docker-compose.yml
├── app/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app.py
├── prometheus/
│   ├── prometheus.yml
│   └── alert_rules.yml
├── alertmanager/
│   └── alertmanager.yml
├── loki/
│   └── loki-config.yml
├── alloy/
│   └── config.alloy
└── grafana/
    └── provisioning/
        └── datasources/
            └── datasources.yml
```

---

## Step 1 — Create the EC2 Server ☁️ (AWS Console)

> The AWS Console screen may look a little different depending on the console version.

**Step 1:** Open the AWS Console and sign in.

**Step 2:** At the top right, choose a **Region** (for example, the closest to you). Remember it.

**Step 3:** In the search bar type `EC2` and open **EC2**.

**Step 4:** Click **Instances** in the left menu, then click **Launch instances**.

**Step 5:** Enter the Name:

* Name: `observability-server`

**Step 6:** In **Application and OS Images (Amazon Machine Image)**:

* Click **Ubuntu**
* Choose **Ubuntu Server 24.04 LTS**
* Architecture: **64-bit (x86)**

**Step 7:** In **Instance type**, choose `t3.medium` (or `t3.large`).

**Step 8:** In **Key pair (login)**:

* Click **Create new key pair**
* Key pair name: `observability-key`
* Key pair type: `RSA`
* Private key file format: `.pem`
* Click **Create key pair**

The file `observability-key.pem` is downloaded. **Keep it safe.** You cannot download it again. (Create it even if you will use the browser connect option, because the form asks for it.)

**Step 9:** In **Network settings**, click **Edit** and set:

* Auto-assign public IP: **Enable**
* Firewall (security groups): **Create security group**
* Security group name: `observability-sg`
* Description: `Observability lab`

**Step 10:** In **Inbound security group rules**, set the rules below. Rule 1 is already there. For rules 2 to 5, click **Add security group rule** each time.

| # | Type | Port range | Source |
|---|---|---|---|
| 1 | SSH | 22 | My IP |
| 2 | Custom TCP | 3000 | My IP |
| 3 | Custom TCP | 9090 | My IP |
| 4 | Custom TCP | 9093 | My IP |
| 5 | Custom TCP | 8000 | My IP |

> Use `My IP` for safety. For a classroom demo you can use `Anywhere-IPv4`, but then anyone can open your Grafana and Prometheus. Delete the server after class.

**Step 11:** In **Configure storage**: Size `30` GiB, Type `gp3`.

**Step 12:** Click **Launch instance**.

**Step 13:** Click **View all instances**. Wait until:

* **Instance state:** `Running`
* **Status check:** `2/2 checks passed` (1 to 2 minutes)

**Step 14:** Click your instance. Copy the **Public IPv4 address**. We call it `<EC2_PUBLIC_IP>` in this guide.

> ⚠️ If you **Stop** and **Start** the instance later, the Public IP changes. Copy the new one.

**Expected:** Instance `observability-server` is `Running` with `2/2 checks passed`.

---

## Step 2 — Connect to the Server

Use **Option A** (browser) **or** **Option B** (SSH). Do not do both.

### Option A — Browser (EC2 Instance Connect) ☁️

**Step 1:** In **EC2 → Instances**, select `observability-server`.

**Step 2:** Click **Connect**.

**Step 3:** Open the tab **EC2 Instance Connect**. Keep user name `ubuntu`.

**Step 4:** Click **Connect**.

**Expected:** A terminal opens with a prompt like `ubuntu@ip-172-31-x-x:~$`.

### Option B — SSH from your laptop 💻

```bash
cd ~/Downloads
chmod 400 observability-key.pem
ssh -i observability-key.pem ubuntu@<EC2_PUBLIC_IP>
```

**Replace:**

* `<EC2_PUBLIC_IP>` → the Public IPv4 address from Step 1

**What this does:** Goes to the folder with the key, makes the key private (`chmod 400`, Linux/Mac only; skip on Windows), and connects. Type `yes` if asked "Are you sure you want to continue connecting?".

**Expected:** Prompt `ubuntu@ip-172-31-x-x:~$`.

### Check you are inside the server 🖥️

```bash
whoami
cat /etc/os-release
```

**Expected:** `ubuntu`, and the OS is Ubuntu 24.04.

> ✅ From here, every command runs on the server terminal, unless marked otherwise.

---

## Step 3 — Install Docker Engine and Docker Compose 🖥️

**Official Installation Guide:** Docker Docs → "Install Docker Engine on Ubuntu" → "Install using the apt repository" (`https://docs.docker.com/engine/install/ubuntu/`). The commands below are from that page.

> We use Docker's official repository because Ubuntu's `docker.io` package does **not** include the `docker compose` plugin that we need.

### 3.1 Remove old conflicting packages (safe to run on a new server)

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc docker-buildx podman-docker containerd runc | cut -f1)
```

**What this does:** Removes unofficial Docker packages that can conflict.
**Expected:** apt may say that none of these packages are installed. That is fine.

### 3.2 Set up Docker's apt repository

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

**What this does:** Installs required tools, creates the keyring folder, and downloads Docker's official signing key.

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

**What this does:** Adds the Docker repository to apt.

```bash
sudo apt update
```

**What this does:** Refreshes the package list, now including Docker packages.

### 3.3 Install Docker

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**What this does:** Installs Docker Engine, the CLI, and the Compose plugin. Type `Y` if asked.

### 3.4 Check that the Docker service is running and starts at boot

```bash
sudo systemctl status docker
```

**Expected:** `active (running)`. Press `q` to exit.

If it is not running:

```bash
sudo systemctl start docker
```

```bash
sudo systemctl enable docker
```

**What this does:** Makes Docker start automatically after a reboot.

### 3.5 Run Docker without `sudo`

```bash
sudo usermod -aG docker $USER && newgrp docker
```

**What this does:** Adds your user to the `docker` group. `newgrp docker` applies it in the current terminal.

### 3.6 Verify

```bash
docker --version
docker compose version
docker run --rm hello-world
```

**Expected:**

* `docker --version` prints a version (29.x)
* `docker compose version` prints a version (Docker Compose version v2.x or newer)
* `hello-world` prints "Hello from Docker!" and exits

> If `docker run` shows a permission error, close the terminal, connect again (Step 2), and try again.

---

## Step 4 — Create the Project Folders 🖥️

```bash
cd ~
mkdir -p observability-lab/{app,prometheus,alertmanager,loki,alloy,grafana/provisioning/datasources}
cd observability-lab
```

**What this does:** Creates the project folder and all sub-folders in one command, then goes inside. (`{a,b}` is bash brace expansion.)

**Verify:**

```bash
find . -type d | sort
```

**Expected:** These folders are listed: `./alertmanager`, `./alloy`, `./app`, `./grafana/provisioning/datasources`, `./loki`, `./prometheus` (plus parent folders).

> ✅ All next commands run from `~/observability-lab`. If you open a new terminal, run `cd ~/observability-lab` first.

---

## Step 5 — Create the Demo App 🖥️

The demo app is a small Python (Flask) web app. It has:

* `/` → fast page
* `/slow` → slow page (0.5 to 2 seconds)
* `/error` → always returns HTTP 500 and writes an ERROR log
* `/metrics` → metrics for Prometheus

It counts requests (`http_requests_total`) and measures latency (`http_request_duration_seconds`). This shows how **your own app** exposes metrics.

> **How the commands below work:** `cat > file <<'EOF' ... EOF` writes everything between the two `EOF` lines into the file. Copy the **whole block** (from `cat` to the last `EOF`) and paste it in the terminal.

### 5.1 `app/requirements.txt`

```bash
cat > app/requirements.txt <<'EOF'
Flask==3.0.3
prometheus-client==0.20.0
EOF
```

**What this does:** Lists the Python packages the app needs.

### 5.2 `app/app.py`

```bash
cat > app/app.py <<'EOF'
import logging
import random
import time

from flask import Flask, Response, g, request
from prometheus_client import CONTENT_TYPE_LATEST, Counter, Histogram, generate_latest

app = Flask(__name__)

# Logs go to stdout so Docker can collect them
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger("demo-app")

# Metrics
REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status"],
)
REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds",
    "HTTP request latency in seconds",
    ["endpoint"],
)


@app.before_request
def start_timer():
    g.start_time = time.time()


@app.after_request
def record_metrics(response):
    if request.path != "/metrics":
        endpoint = request.url_rule.rule if request.url_rule else "unmatched"
        latency = time.time() - g.start_time
        REQUEST_COUNT.labels(request.method, endpoint, response.status_code).inc()
        REQUEST_LATENCY.labels(endpoint).observe(latency)
        logger.info("%s %s %s %.3fs", request.method, request.path, response.status_code, latency)
    return response


@app.route("/")
def home():
    return "Hello from the demo app!\n"


@app.route("/slow")
def slow():
    time.sleep(random.uniform(0.5, 2.0))
    return "That was slow.\n"


@app.route("/error")
def error():
    logger.error("Simulated error on /error")
    return Response("Simulated server error\n", status=500)


@app.route("/metrics")
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
EOF
```

**What this does:** Creates the app. Important parts:

* `REQUEST_COUNT` (Counter) counts requests by method, endpoint and status code.
* `REQUEST_LATENCY` (Histogram) measures how long each request takes.
* `after_request` records both for every request (except `/metrics`) and writes a log line.
* `/metrics` returns all metrics in the Prometheus text format.

### 5.3 `app/Dockerfile`

```bash
cat > app/Dockerfile <<'EOF'
FROM python:3.12-slim
WORKDIR /app
ENV PYTHONUNBUFFERED=1
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 8000
CMD ["python", "app.py"]
EOF
```

**What this does:** Builds the app image. `PYTHONUNBUFFERED=1` makes Python print logs immediately (needed so logs reach Loki in real time).

**Verify:**

```bash
ls app
cat app/requirements.txt
```

**Expected:** `Dockerfile  app.py  requirements.txt`, and the two package lines.

---

## Step 6 — Create the Prometheus Configuration 🖥️

### 6.1 `prometheus/prometheus.yml`

```bash
cat > prometheus/prometheus.yml <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

rule_files:
  - /etc/prometheus/alert_rules.yml

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: alertmanager
    static_configs:
      - targets: ["alertmanager:9093"]

  - job_name: node-exporter
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: cadvisor
    static_configs:
      - targets: ["cadvisor:8080"]

  - job_name: demo-app
    metrics_path: /metrics
    static_configs:
      - targets: ["demo-app:8000"]
EOF
```

**What this does:** Tells Prometheus what to scrape and where to send alerts.

**Important fields:**

| Field | Meaning |
|---|---|
| `scrape_interval: 15s` | Scrape every target every 15 seconds |
| `evaluation_interval: 15s` | Check alert rules every 15 seconds |
| `alerting.alertmanagers` | Where to send alerts (`alertmanager` is the container name on the Docker network) |
| `rule_files` | The alert rules file (created next) |
| `scrape_configs` | The list of jobs. Each `job_name` has its targets (`container-name:port`) |

> Inside Docker Compose, containers reach each other by **service name** (for example `demo-app:8000`). That is why we do not use IP addresses.

### 6.2 `prometheus/alert_rules.yml`

```bash
cat > prometheus/alert_rules.yml <<'EOF'
groups:
  - name: infrastructure-alerts
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Target {{ $labels.instance }} is down"
          description: "Job {{ $labels.job }} target {{ $labels.instance }} has been down for more than 1 minute."

      - alert: HighCpuUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 80% for more than 2 minutes."

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is above 80% for more than 2 minutes."

      - alert: LowDiskSpace
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 20
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Less than 20% of the root disk is free."

  - name: application-alerts
    rules:
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[1m])) / sum(rate(http_requests_total[1m])) > 0.05
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "High error rate in demo-app"
          description: "More than 5% of requests return a 5xx status for more than 1 minute."
EOF
```

**What this does:** Defines 5 alerts.

| Alert | Fires when | Type |
|---|---|---|
| `InstanceDown` | Any target has `up == 0` for 1 minute | Availability |
| `HighCpuUsage` | Server CPU is above 80% for 2 minutes | Infrastructure |
| `HighMemoryUsage` | Server memory is above 80% for 2 minutes | Infrastructure |
| `LowDiskSpace` | Less than 20% of the root disk is free | Infrastructure |
| `HighErrorRate` | More than 5% of requests are 5xx for 1 minute | Application |

Each rule has: `alert` (name), `expr` (PromQL condition), `for` (wait time), `labels` (extra tags such as severity), `annotations` (human text; `{{ $labels.instance }}` inserts the label value).

**Verify:**

```bash
ls prometheus
```

**Expected:** `alert_rules.yml  prometheus.yml`.

---

## Step 7 — Create the Alertmanager Configuration 🖥️

```bash
cat > alertmanager/alertmanager.yml <<'EOF'
route:
  receiver: default-receiver
  group_by: ["alertname"]
  group_wait: 10s
  group_interval: 1m
  repeat_interval: 1h

receivers:
  - name: default-receiver
EOF
```

**What this does:** Sets a simple routing for all alerts.

| Field | Meaning |
|---|---|
| `receiver` | Where alerts go by default (`default-receiver`) |
| `group_by` | Alerts with the same `alertname` are grouped in one notification |
| `group_wait: 10s` | Wait 10 seconds to collect more alerts for the group before sending |
| `group_interval: 1m` | Wait 1 minute before sending new alerts of an existing group |
| `repeat_interval: 1h` | Repeat the notification every hour if the alert is still firing |
| `receivers` | List of receivers. Ours has **no** email/Slack settings, so alerts only appear in the Alertmanager web page. This is fine for the lab. Slack is shown in the Optional section at the end. |

**Verify:**

```bash
cat alertmanager/alertmanager.yml
```

---

## Step 8 — Create the Loki and Alloy Configuration 🖥️

### 8.1 `loki/loki-config.yml`

```bash
cat > loki/loki-config.yml <<'EOF'
auth_enabled: false

server:
  http_listen_port: 3100

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

analytics:
  reporting_enabled: false
EOF
```

**What this does:** A minimal single-container Loki that stores logs on disk.

| Field | Meaning |
|---|---|
| `auth_enabled: false` | No multi-tenant login (fine for a lab) |
| `http_listen_port: 3100` | Loki API port |
| `common.path_prefix: /loki` | Data folder inside the container (we mount a volume here) |
| `schema_config` (`tsdb`, `v13`) | The current recommended index format for Loki 3.x |
| `analytics.reporting_enabled: false` | Do not send usage statistics |

> ⚠️ Logs are kept forever in this lab (no retention set). For production, configure retention.

### 8.2 `alloy/config.alloy`

```bash
cat > alloy/config.alloy <<'EOF'
// 1. Find all Docker containers
discovery.docker "linux" {
  host = "unix:///var/run/docker.sock"
}

// 2. Create friendly labels from Docker metadata
discovery.relabel "containers" {
  targets = []

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "/(.*)"
    target_label  = "container"
  }

  rule {
    source_labels = ["__meta_docker_container_label_com_docker_compose_service"]
    target_label  = "service"
  }
}

// 3. Read the logs of those containers
loki.source.docker "default" {
  host          = "unix:///var/run/docker.sock"
  targets       = discovery.docker.linux.targets
  labels        = { "platform" = "docker" }
  relabel_rules = discovery.relabel.containers.rules
  forward_to    = [loki.write.local.receiver]
}

// 4. Send the logs to Loki
loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
EOF
```

**What this does:** Defines the log pipeline (Alloy uses a "component" style config):

1. `discovery.docker` finds all containers through the Docker socket.
2. `discovery.relabel` creates two labels: `container` (for example `demo-app`) and `service` (the Compose service name).
3. `loki.source.docker` reads the container logs and adds the label `platform="docker"`.
4. `loki.write` sends everything to Loki at `http://loki:3100`.

**Verify:**

```bash
ls loki alloy
```

**Expected:** `loki-config.yml` and `config.alloy`.

---

## Step 9 — Create the Grafana Data Source File and the `.env` File 🖥️

### 9.1 `grafana/provisioning/datasources/datasources.yml`

```bash
cat > grafana/provisioning/datasources/datasources.yml <<'EOF'
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    uid: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
    jsonData:
      timeInterval: 15s

  - name: Loki
    type: loki
    uid: loki
    access: proxy
    url: http://loki:3100
    editable: true
EOF
```

**What this does:** Grafana reads this file at start and creates two data sources automatically: Prometheus (default) and Loki. `access: proxy` means the Grafana server (not your browser) talks to them, using the Docker service names.

### 9.2 `.env` (Grafana admin password)

```bash
nano .env
```

Type this one line:

```text
GRAFANA_ADMIN_PASSWORD=<YOUR_STRONG_PASSWORD>
```

**Replace:**

* `<YOUR_STRONG_PASSWORD>` → a password you choose (for example `Lab2026Secure`). Do **not** use spaces, quotes, or the `$` character. Write it **without** `<` and `>`.

Save and exit: `Ctrl + O`, `Enter`, `Ctrl + X`.

**Verify:**

```bash
cat .env
```

**Expected:** One line `GRAFANA_ADMIN_PASSWORD=...` with your real password.

> Docker Compose reads `.env` automatically and fills `${GRAFANA_ADMIN_PASSWORD}` in `docker-compose.yml`.

---

## Step 10 — Create `docker-compose.yml` 🖥️

```bash
cat > docker-compose.yml <<'EOF'
name: observability-lab

services:
  demo-app:
    build: ./app
    container_name: demo-app
    ports:
      - "8000:8000"
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:v3.13.0
    container_name: prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=7d"
      - "--web.enable-lifecycle"
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml:ro
      - prometheus-data:/prometheus
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:v0.33.1
    container_name: alertmanager
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
      - "--storage.path=/alertmanager"
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager-data:/alertmanager
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:v1.12.1
    container_name: node-exporter
    command:
      - "--path.rootfs=/host"
    pid: host
    volumes:
      - "/:/host:ro,rslave"
    restart: unless-stopped

  cadvisor:
    image: ghcr.io/google/cadvisor:v0.60.5
    container_name: cadvisor
    privileged: true
    devices:
      - /dev/kmsg:/dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    restart: unless-stopped

  loki:
    image: grafana/loki:3.6.17
    container_name: loki
    command: -config.file=/etc/loki/loki-config.yml
    ports:
      - "127.0.0.1:3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml:ro
      - loki-data:/loki
    restart: unless-stopped

  alloy:
    image: grafana/alloy:v1.19.2
    container_name: alloy
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - --storage.path=/var/lib/alloy/data
      - /etc/alloy/config.alloy
    ports:
      - "127.0.0.1:12345:12345"
    volumes:
      - ./alloy/config.alloy:/etc/alloy/config.alloy:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - alloy-data:/var/lib/alloy/data
    restart: unless-stopped

  grafana:
    image: grafana/grafana:12.4.11
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    restart: unless-stopped

volumes:
  prometheus-data:
  alertmanager-data:
  loki-data:
  alloy-data:
  grafana-data:
EOF
```

**What this does:** Defines all 8 containers. Important points:

| Service | Important settings |
|---|---|
| `demo-app` | `build: ./app` builds the image from our Dockerfile. `container_name` gives it a fixed name (used in logs and metrics). |
| `prometheus` | Config and rules mounted read-only (`:ro`). `--web.enable-lifecycle` lets us reload the config without restart. Data in a volume, kept 7 days. |
| `alertmanager` | Config mounted read-only. Data in a volume. |
| `node-exporter` | `pid: host` and mounting `/` at `/host` let it read the **real server's** metrics. `--path.rootfs=/host` tells it where the host files are. |
| `cadvisor` | `privileged` and the mounted folders let it read container information from Docker. |
| `loki` | Config mounted. Port `3100` is published to **localhost only** (`127.0.0.1`), so it is not open to the internet. |
| `alloy` | Mounts the Docker socket (`/var/run/docker.sock`) to read container logs. Port `12345` (Alloy UI) is localhost only. |
| `grafana` | Admin password comes from `.env`. Provisioning folder mounted. `GF_USERS_ALLOW_SIGN_UP=false` blocks public sign-up. |
| `volumes` | Named volumes keep data (metrics, logs, dashboards) when containers restart. |

> ⚠️ `node-exporter` and `cadvisor` ports are **not published**. Only Prometheus (inside the Docker network) scrapes them.

**Verify the structure:**

```bash
find . -type f | sort
```

**Expected:**

```text
./.env
./alertmanager/alertmanager.yml
./alloy/config.alloy
./app/Dockerfile
./app/app.py
./app/requirements.txt
./docker-compose.yml
./grafana/provisioning/datasources/datasources.yml
./loki/loki-config.yml
./prometheus/alert_rules.yml
./prometheus/prometheus.yml
```

If any file is missing, go back to its step.

---

## Step 11 — Validate the Configuration Files Before Starting 🖥️

### 11.1 Validate the Compose file

```bash
docker compose config --quiet && echo "compose file OK"
```

**What this does:** Checks the syntax of `docker-compose.yml` and the `.env` substitution.
**Expected:** `compose file OK`. If there is an error, read the line number in the message and fix that file.

### 11.2 Validate the Prometheus config and alert rules

```bash
docker run --rm --entrypoint promtool \
  -v "$PWD/prometheus:/etc/prometheus:ro" \
  prom/prometheus:v3.13.0 check config /etc/prometheus/prometheus.yml
```

**What this does:** Runs Prometheus's own checker (`promtool`) on the config and the alert rules file. The first time, it downloads the Prometheus image.
**Expected:** Lines ending with `SUCCESS` for the config and for the rules file (`1 rule files found`, `SUCCESS: 5 rules found`).

### 11.3 Validate the Alertmanager config

```bash
docker run --rm --entrypoint amtool \
  -v "$PWD/alertmanager:/etc/alertmanager:ro" \
  prom/alertmanager:v0.33.1 check-config /etc/alertmanager/alertmanager.yml
```

**What this does:** Runs `amtool` to check the Alertmanager config.
**Expected:** `SUCCESS` and a summary such as `Found: - global config - route - 0 inhibit rules - 1 receivers - 0 templates`.

---

## Step 12 — Start the Whole Stack 🖥️

```bash
docker compose up -d --build
```

**What this does:** Builds the demo app image, downloads all other images, and starts all containers in the background (`-d`). The first run takes 2 to 5 minutes.
**Expected:** Ends with lines like `Container prometheus Started`, `Container grafana Started` for all services.

### 12.1 Check that all containers are running

```bash
docker compose ps
```

**Expected:** All 8 services (`alertmanager`, `alloy`, `cadvisor`, `demo-app`, `grafana`, `loki`, `node-exporter`, `prometheus`) show `running` (or `Up`). `cadvisor` may also show `(healthy)`.

If one is `Restarting` or `Exited`, see **Step 20 Troubleshooting** and read its logs:

```bash
docker compose logs <SERVICE_NAME>
```

**Replace:**

* `<SERVICE_NAME>` → the service name, for example `prometheus`

### 12.2 Health checks from the server

```bash
curl -s http://localhost:9090/-/healthy
curl -s http://localhost:9093/-/healthy
curl -s http://localhost:3000/api/health
curl -s http://localhost:8000/
```

**Expected:**

* Prometheus: `Prometheus Server is Healthy.`
* Alertmanager: `OK`
* Grafana: a JSON text with `"database": "ok"`
* Demo app: `Hello from the demo app!`

### 12.3 Check Loki is ready

```bash
curl -s http://localhost:3100/ready
```

**Expected:** `ready`. If it says `Ingester not ready: waiting for 15s after being ready`, wait 30 seconds and run it again.

### 12.4 Look at the app metrics (this is what Prometheus scrapes)

```bash
curl -s http://localhost:8000/metrics | grep http_request
```

**Expected:** Lines with `http_requests_total` and `http_request_duration_seconds_...`. (They may be empty until the app gets traffic. Run `curl http://localhost:8000/` once and repeat.)

### 12.5 Your URLs

| Tool | URL |
|---|---|
| Grafana | `http://<EC2_PUBLIC_IP>:3000` |
| Prometheus | `http://<EC2_PUBLIC_IP>:9090` |
| Alertmanager | `http://<EC2_PUBLIC_IP>:9093` |
| Demo app | `http://<EC2_PUBLIC_IP>:8000` |

**Replace:**

* `<EC2_PUBLIC_IP>` → Public IPv4 address of your server. Use `http`, not `https`.

---

## Step 13 — Verify Prometheus 🌐

### 13.1 Check the targets

**Step 1:** Open `http://<EC2_PUBLIC_IP>:9090`.

**Step 2:** In the top menu click **Status → Target health** (in some versions: **Status → Targets**).

**Expected:** 5 jobs, all with state **UP**: `prometheus`, `alertmanager`, `node-exporter`, `cadvisor`, `demo-app`.

If a target is `DOWN`, wait 30 seconds and refresh. If it stays down, see Troubleshooting.

### 13.2 Run your first queries

**Step 1:** Click **Query** in the top menu (or **Graph**).

**Step 2:** Type this and click **Execute**:

```text
up
```

**Expected:** 5 results, each with value `1`.

**Step 3:** Try this (server memory available, in bytes):

```text
node_memory_MemAvailable_bytes
```

**Expected:** One result with a big number.

**Step 4:** Try this (per-container memory):

```text
container_memory_working_set_bytes{name!=""}
```

**Expected:** One result per container (`demo-app`, `prometheus`, `grafana`, ...). If this is empty, see Troubleshooting (cAdvisor).

### 13.3 Check the alert rules

**Step 1:** Click **Alerts** in the top menu.

**Expected:** 5 rules are listed (`InstanceDown`, `HighCpuUsage`, `HighMemoryUsage`, `LowDiskSpace`, `HighErrorRate`), all **Inactive** (green).

> The Prometheus screens differ a little between versions. If a menu name is different, look for "Targets", "Query/Graph" and "Alerts".

---

## Step 14 — Grafana: Login, Data Sources and Dashboards 🌐

### 14.1 Log in

**Step 1:** Open `http://<EC2_PUBLIC_IP>:3000`.

**Step 2:** Enter:

* Email or username: `admin`
* Password: the password you set in `.env` (Step 9.2)

**Step 3:** Click **Log in**. If Grafana asks you to change the password, you can click **Skip**.

**Expected:** The Grafana home page opens.

### 14.2 Check the data sources (they were created automatically)

**Step 1:** In the left menu, click **Connections → Data sources**.

**Step 2:** You should see **Prometheus** and **Loki**.

**Step 3:** Click **Prometheus**. Scroll down and click **Save & test**.

**Expected:** A green message like "Successfully queried the Prometheus API."

**Step 4:** Go back, click **Loki**, scroll down and click **Save & test**.

**Expected:** A green message like "Data source successfully connected." (If it fails right after start, wait 30 seconds and test again.)

### 14.3 Import a ready-made dashboard: Node Exporter Full (server metrics)

**Step 1:** In the left menu, click **Dashboards**.

**Step 2:** Click **New → Import**.

**Step 3:** In "Find and import dashboards for common applications at grafana.com/dashboards", type:

* `1860`

**Step 4:** Click **Load**.

**Step 5:** In the **Prometheus** drop-down, select **Prometheus**.

**Step 6:** Click **Import**.

**Expected:** The "Node Exporter Full" dashboard opens with CPU, memory, disk and network graphs of your EC2 server. At the top, the **Job** and **Host** selectors should be set (if a panel says "No data", choose job `node-exporter` in the top selector).

> Dashboard ID `1860` is the popular community dashboard "Node Exporter Full" from grafana.com. The Grafana server needs internet access to download it.
>
> ⚠️ Verification Required: The Grafana menu names can differ in other Grafana versions. Network graphs show the container network (not the full server network), because Node Exporter runs in a container.

### 14.4 Import a ready-made dashboard: cAdvisor (container metrics)

Repeat the same steps as 14.3, with ID:

* `14282`

**Expected:** A dashboard with CPU, memory and network per container.

> ⚠️ Verification Required: Community dashboard `14282` ("Cadvisor exporter") is a popular one, but community dashboards can change over time. If it shows "No data", do not worry. In Step 14.5 we build our own container panels.

### 14.4b If an imported dashboard shows "No data"

Check the top of the dashboard: the **time range** (top right) should be "Last 15 minutes" or "Last 1 hour", and the data source must be **Prometheus**. If needed, go to Step 13.2 and confirm the query works in Prometheus first.

### 14.5 Build your own dashboard: "Demo App Overview"

We create 5 panels. This teaches you how a dashboard is made.

**Create the dashboard:**

**Step 1:** Left menu → **Dashboards → New → New dashboard**.

**Step 2:** Click **+ Add visualization**.

**Step 3:** In "Select data source", choose **Prometheus**.

**Panel 1 — Request rate per endpoint (Traffic):**

**Step 4:** In the query section, click **Code** (switch from "Builder" to "Code").

**Step 5:** Paste this query:

```text
sum(rate(http_requests_total[1m])) by (endpoint)
```

**Step 6:** In the right panel, set **Title**: `Requests per second by endpoint`.

**Step 7:** Click **Back to dashboard** (top left).

**Add each next panel:** Click **Add → Visualization** (top bar), choose data source **Prometheus**, click **Code**, paste the query, set the **Title**, click **Back to dashboard**.

**Panel 2 — Error percentage (Errors):**

Query:

```text
100 * sum(rate(http_requests_total{status=~"5.."}[1m])) / sum(rate(http_requests_total[1m]))
```

Title: `Error percentage (5xx)`

**Panel 3 — p95 latency (Latency):**

Query:

```text
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[1m])) by (le))
```

Title: `p95 latency (seconds)`

**Panel 4 — Container CPU (Saturation):**

Query:

```text
sum(rate(container_cpu_usage_seconds_total{name!=""}[1m])) by (name)
```

Title: `Container CPU (cores)`

**Panel 5 — Server CPU % (Saturation):**

Query:

```text
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
```

Title: `Server CPU usage %`

**Panel 6 (logs) — Application logs:**

**Step 1:** Click **Add → Visualization**.

**Step 2:** Choose data source **Loki**.

**Step 3:** Click **Code** and paste:

```text
{container="demo-app"}
```

**Step 4:** At the top right of the panel editor, change the visualization type to **Logs**.

**Step 5:** Title: `Demo app logs`. Click **Back to dashboard**.

**Save the dashboard:**

**Step 1:** Click **Save dashboard** (top right).

**Step 2:** Title: `Demo App Overview`.

**Step 3:** Click **Save**.

**Step 4:** Set the time range (top right) to **Last 15 minutes** and the refresh (top right) to **10s**.

**Expected:** The dashboard has 6 panels. The graphs are empty for now (no traffic yet). Next step: create traffic.

> ⚠️ Verification Required: The exact button names ("Add → Visualization", "Back to dashboard") can be a little different in other Grafana versions.

---

## Step 15 — Generate Traffic and Watch Metrics 🖥️

Open your **Demo App Overview** dashboard in the browser (with 10s refresh). Then run these commands on the server.

### 15.1 Normal traffic

```bash
for i in $(seq 1 100); do curl -s -o /dev/null http://localhost:8000/; sleep 0.2; done
```

**What this does:** Sends 100 requests to `/` (one every 0.2 seconds).

### 15.2 Slow traffic

```bash
for i in $(seq 1 15); do curl -s -o /dev/null http://localhost:8000/slow; done
```

**What this does:** Sends 15 requests to `/slow`. Each takes 0.5 to 2 seconds, so this runs for about 20 seconds.

### 15.3 Error traffic

```bash
for i in $(seq 1 30); do curl -s -o /dev/null http://localhost:8000/error; sleep 0.3; done
```

**What this does:** Sends 30 requests to `/error`, which returns HTTP 500.

### 15.4 What you should see in Grafana 🌐

* **Requests per second by endpoint:** lines for `/`, `/slow` and `/error`
* **Error percentage:** goes up while `/error` is called
* **p95 latency:** goes up after the `/slow` calls
* **Demo app logs:** log lines like `GET / 200 0.001s` and `ERROR Simulated error on /error`

### 15.5 Explain what happened (for students)

* **Traffic** = the request rate panel.
* **Errors** = the error percentage panel.
* **Latency** = the p95 panel.
* These are the golden signals from Theory A3.

---

## Step 16 — Explore the Logs with Loki 🌐

**Step 1:** In Grafana, left menu → **Explore**.

**Step 2:** At the top, choose data source **Loki**.

**Step 3:** Click **Code** (or "Builder"→"Code") and run these queries one by one (click **Run query**, top right):

| Goal | Query |
|---|---|
| All logs of the demo app | `{container="demo-app"}` |
| Only errors | `{container="demo-app"} \|= "ERROR"` |
| Only requests to `/slow` | `{container="demo-app"} \|= "/slow"` |
| Logs of all containers | `{platform="docker"}` |
| Logs of Prometheus | `{container="prometheus"}` |
| Logs of every Compose service, count per second | `sum(rate({platform="docker"}[1m])) by (container)` |

> In the "Only errors" query, type the pipe character `|` normally: `{container="demo-app"} |= "ERROR"`. (It is shown with a backslash above only because of the table format.)

**Expected:**

* Query 1 shows the request log lines.
* Query 2 shows only `Simulated error on /error` lines.
* The last query draws a graph of log lines per second per container.

**Label browser (optional):** Use the **Label browser** / label filters button to click through labels (`container`, `service`, `platform`) without typing.

**Explain to students:** Loki finds logs by **labels** first (`{container="demo-app"}`), then filters the text (`|= "ERROR"`).

---

## Step 17 — Test Alerting 🖥️ 🌐

You will trigger three alerts. Open two browser tabs first:

* Prometheus alerts: `http://<EC2_PUBLIC_IP>:9090/alerts`
* Alertmanager: `http://<EC2_PUBLIC_IP>:9093`

### Test 1 — `InstanceDown` (stop the app)

**Step 1:** 🖥️ Stop the demo app:

```bash
docker compose stop demo-app
```

**What this does:** Stops the app container, so Prometheus cannot scrape it.

**Step 2:** 🌐 In Prometheus → **Status → Target health**, wait about 15 to 30 seconds. `demo-app` becomes **DOWN**.

**Step 3:** 🌐 In Prometheus → **Alerts**, refresh every 20 seconds:

* `InstanceDown` becomes **Pending** (yellow), because the condition is true but the `for: 1m` timer is running.
* After about 1 minute it becomes **Firing** (red).

**Step 4:** 🌐 Open Alertmanager (`:9093`). Refresh.

**Expected:** The alert `InstanceDown` (job `demo-app`, severity `critical`) is shown in Alertmanager. (Alertmanager needs about 10 to 30 more seconds because of `group_wait`.)

**Step 5:** 🖥️ Start the app again:

```bash
docker compose start demo-app
```

**Expected:** After a short time the alert becomes **Inactive** in Prometheus and is **resolved** (disappears) in Alertmanager.

### Test 2 — `HighErrorRate` (create errors)

**Step 1:** 🖥️ Send a mix of good and bad requests for about 3 minutes (half of them are errors):

```bash
for i in $(seq 1 180); do curl -s -o /dev/null http://localhost:8000/; curl -s -o /dev/null http://localhost:8000/error; sleep 1; done
```

**What this does:** Every second sends 1 good request and 1 error request. That is 50% errors, and the alert limit is 5%. (Press `Ctrl + C` to stop early.)

**Step 2:** 🌐 In Prometheus → **Alerts**: `HighErrorRate` becomes **Pending**, then **Firing** after about 1 to 2 minutes.

**Step 3:** 🌐 In Alertmanager: the alert `HighErrorRate` appears.

**Step 4:** 🌐 In Grafana → **Explore → Loki**, run `{container="demo-app"} |= "ERROR"` to find the cause (the `/error` endpoint).

**Step 5:** Wait 2 minutes after the loop stops. The alert becomes **Inactive** again.

**Explain to students:** The alert tells you **something is wrong** (metrics). The logs tell you **why** (logs). This is observability.

### Test 3 — `HighCpuUsage` (create CPU load)

**Step 1:** 🖥️ Start 2 CPU-burning processes (the server has 2 vCPUs):

```bash
for i in 1 2; do yes > /dev/null & done
```

**What this does:** Runs `yes` twice in the background. Each one uses 100% of one CPU core.

**Step 2:** 🌐 Watch the Grafana dashboard **Node Exporter Full** (CPU Busy goes to about 100%) and the panel **Server CPU usage %** in your dashboard.

**Step 3:** 🌐 In Prometheus → **Alerts**: `HighCpuUsage` becomes **Pending** and then **Firing** after about 3 to 4 minutes (the rate window plus the `for: 2m` timer).

**Step 4:** 🖥️ Stop the load:

```bash
pkill yes
```

**Expected:** CPU drops. The alert becomes **Inactive** after a few minutes.

> If you used a `t3.large` (also 2 vCPUs), the steps are the same. If you use a server with more CPUs, start more `yes` processes (one per CPU).

---

## Step 18 — Lab Exercises (Hands-on Activities)

Give these to students. Each has a clear expected result.

### Exercise 1 — Find the slowest endpoint

1. Send traffic with 15.1, 15.2 and 15.3.
2. In Grafana → **Explore → Prometheus**, run:

```text
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint))
```

**Expected:** `/slow` has the highest value (about 1.9 seconds). `/` and `/error` are near zero.

### Exercise 2 — Which container uses the most memory?

1. In Explore → Prometheus run:

```text
topk(3, container_memory_working_set_bytes{name!=""})
```

**Expected:** The 3 containers with the highest memory (usually `prometheus`, `grafana`, `loki`).

### Exercise 3 — Add your own alert

1. Open `prometheus/alert_rules.yml`:

```bash
nano prometheus/alert_rules.yml
```

2. Inside the `application-alerts` group, at the end of the file, add this rule (keep the same indentation as `HighErrorRate`; it must start with `      - alert:`):

```yaml
      - alert: SlowRequests
        expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[1m])) by (le)) > 1
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Slow requests in demo-app"
          description: "The p95 latency is above 1 second for more than 1 minute."
```

3. Save (`Ctrl + O`, `Enter`, `Ctrl + X`). Check the file, then reload Prometheus **without restart**:

```bash
docker run --rm --entrypoint promtool -v "$PWD/prometheus:/etc/prometheus:ro" prom/prometheus:v3.13.0 check rules /etc/prometheus/alert_rules.yml
curl -X POST http://localhost:9090/-/reload
```

**What this does:** `promtool` checks the rules. The `curl` command tells Prometheus to reload its config (possible because of `--web.enable-lifecycle`).
**Expected:** `SUCCESS: 6 rules found`. In Prometheus → Alerts you now see `SlowRequests`.

4. Trigger it: keep calling the slow endpoint for 2 minutes:

```bash
for i in $(seq 1 80); do curl -s -o /dev/null http://localhost:8000/slow; done
```

**Expected:** `SlowRequests` becomes Pending, then Firing.

### Exercise 4 — Silence an alert (Alertmanager)

1. Make an alert fire (for example Test 1: `docker compose stop demo-app`).
2. In Alertmanager (`:9093`), click **Silence** next to the alert (or **New Silence**).
3. Set a duration (for example 30 minutes), write a comment, click **Create**.

**Expected:** The alert disappears from the active list and is shown as silenced. (Start the app again after the test: `docker compose start demo-app`.)

### Exercise 5 — Correlate metrics and logs

1. Run the error traffic (15.3).
2. In your dashboard, find the time when "Error percentage" went up.
3. In the "Demo app logs" panel of the same time range, find the `ERROR` lines.

**Expected:** The error time in the graph matches the ERROR log lines. This is how you find the cause of a problem.

---

## Step 19 — Useful Operation Commands 🖥️ (reference)

Run these in `~/observability-lab`.

| Command | What it does |
|---|---|
| `docker compose ps` | Shows the state of all services |
| `docker compose logs -f <SERVICE_NAME>` | Follows the logs of a service (`Ctrl + C` to stop) |
| `docker compose restart <SERVICE_NAME>` | Restarts one service (for example after changing its config) |
| `docker compose stop` | Stops all containers (data is kept) |
| `docker compose start` | Starts them again |
| `docker compose down` | Removes containers and network (data volumes are kept) |
| `docker compose down -v` | Removes containers **and all data** (metrics, logs, dashboards) |
| `docker stats --no-stream` | Shows CPU and memory of each container |

> After changing `prometheus.yml` or `alert_rules.yml`, reload Prometheus with `curl -X POST http://localhost:9090/-/reload`. After changing `alertmanager.yml`, `alloy/config.alloy` or `loki-config.yml`, restart that service.

---

## Step 20 — Troubleshooting

### Error 1

```text
permission denied while trying to connect to the Docker daemon socket
```

**Reason:** Your user is not in the `docker` group yet, or the group is not applied.

**Fix:**

```bash
sudo usermod -aG docker $USER && newgrp docker
```

If it still fails, close the terminal and connect again (Step 2).

### Error 2

```text
docker: 'compose' is not a docker command
```

**Reason:** The Compose plugin is not installed (for example Docker was installed as `docker.io`).

**Fix:** Repeat Step 3 (remove old packages, add the official repository, install `docker-compose-plugin`).

### Error 3

```text
Bind for 0.0.0.0:3000 failed: port is already allocated
```

(or `address already in use` for `9090`, `9093`, `8000`)

**Reason:** Another program or container already uses that port.

**Fix:**

```bash
sudo ss -tulpn | grep -E ':(3000|9090|9093|8000)\b'
```

Stop the program that uses the port (for example an old container with `docker ps` and `docker stop <CONTAINER_NAME>`), then run `docker compose up -d` again.

### Error 4

A Prometheus target is `DOWN` (Status → Target health), for example `demo-app`.

**Reason:** The container is stopped or crashed, or the target name/port in `prometheus.yml` is wrong.

**Fix:**

```bash
docker compose ps
docker compose logs demo-app
```

Start it with `docker compose start demo-app`. Check that the target in `prometheus/prometheus.yml` is `demo-app:8000`.

### Error 5

Prometheus or Alertmanager container keeps restarting.

**Reason:** A syntax error in its config file (usually wrong indentation in YAML).

**Fix:**

```bash
docker compose logs prometheus
docker run --rm --entrypoint promtool -v "$PWD/prometheus:/etc/prometheus:ro" prom/prometheus:v3.13.0 check config /etc/prometheus/prometheus.yml
```

Fix the line shown in the message (use **spaces**, not tabs), then run `docker compose up -d`. For Alertmanager use the `amtool` command from Step 11.3.

### Error 6

Cannot log in to Grafana (wrong password).

**Reason:** `GRAFANA_ADMIN_PASSWORD` is only used the **first time** Grafana starts with an empty data volume. If you changed `.env` later, Grafana still has the old password.

**Fix (keep your data):**

```bash
docker compose exec grafana grafana cli admin reset-admin-password <NEW_PASSWORD>
```

**Replace:**

* `<NEW_PASSWORD>` → the new password you want

**Other fix (deletes all Grafana data, including your dashboards):**

```bash
docker compose stop grafana
docker volume rm observability-lab_grafana-data
docker compose up -d grafana
```

### Error 7

A Grafana panel shows **No data**.

**Reason:** No traffic yet, wrong time range, or the query does not work.

**Fix:** Send traffic (Step 15). Set the time range to **Last 15 minutes**. Copy the panel query into **Prometheus → Query** and run it there. If it also returns nothing there, the metric name is wrong or the target is down (Error 4).

### Error 8

cAdvisor container metrics have no `name` label, or `container_memory_working_set_bytes{name!=""}` returns nothing.

**Reason:** An old cAdvisor version cannot read containers on Docker 29 (new Docker uses the containerd image store).

**Fix:**

```bash
docker compose logs cadvisor | tail -30
docker compose ps cadvisor
```

Make sure the image in `docker-compose.yml` is `ghcr.io/google/cadvisor:v0.60.5` (not the old `gcr.io/cadvisor/cadvisor`). Then run `docker compose up -d`.

> ⚠️ Verification Required: cAdvisor behaviour depends on the Docker version. Please check this once before class.

### Error 9

No logs in Grafana Explore (Loki).

**Reason:** Loki is not ready, Alloy cannot read the Docker socket, or the label in your query is wrong.

**Fix:**

```bash
curl -s http://localhost:3100/ready
docker compose logs alloy | tail -30
docker compose logs loki | tail -30
```

Wait until Loki says `ready`. Check that `docker-compose.yml` mounts `/var/run/docker.sock` into `alloy`. In Explore, start with the query `{platform="docker"}` (all logs) to confirm that logs exist.

### Error 10

Node Exporter fails to start with a message about the mount propagation (`rslave`).

**Reason:** On some systems the `rslave` option is not accepted.

**Fix:** In `docker-compose.yml`, change `"/:/host:ro,rslave"` to `"/:/host:ro"`, then run `docker compose up -d`.

### Error 11

The browser cannot open a page (timeout), for example `http://<EC2_PUBLIC_IP>:3000`.

**Reason:** The port is not open in the Security Group, the Public IP changed, or your own IP changed (if the rule is `My IP`).

**Fix (AWS Console):** EC2 → Instances → your server → **Security** tab → click the Security Group → **Edit inbound rules**. Check ports `3000`, `9090`, `9093`, `8000`. Choose `My IP` again if your IP changed. Copy the current Public IPv4 address.

### Error 12

An alert stays **Pending** and never fires.

**Reason:** This is normal until the `for` time passes. Or the condition is not true long enough.

**Fix:** Wait for the `for` duration (1 to 2 minutes, plus the rate window). Copy the alert `expr` into **Prometheus → Query** and check that it returns a result.

---

## Step 21 — Cleanup (only after the class)

### 21.1 On the server 🖥️

```bash
cd ~/observability-lab
docker compose down -v
```

**What this does:** Stops and removes all containers and deletes the data volumes.

### 21.2 In the AWS Console (important, to avoid extra cost) ☁️

**Step 1:** Go to **EC2 → Instances**.

**Step 2:** Select `observability-server`.

**Step 3:** Click **Instance state → Terminate (delete) instance**, then confirm.

**Step 4 (optional):** Delete the security group `observability-sg` and the key pair `observability-key` if you do not need them.

> **Stop** only pauses the server (disk cost continues, and the Public IP changes). **Terminate** deletes it.

---

## Optional Alternative — Send Alerts to Slack

By default, alerts only show in the Alertmanager web page. To also send them to Slack:

**Step 1:** Create an **Incoming Webhook** in your Slack workspace (follow Slack's official documentation for "Incoming Webhooks"). Copy the webhook URL.

> ⚠️ Verification Required: I did not check the current Slack screens. Follow the Slack documentation for your workspace.

**Step 2:** Replace `alertmanager/alertmanager.yml` with:

```bash
nano alertmanager/alertmanager.yml
```

```yaml
route:
  receiver: slack-notifications
  group_by: ["alertname"]
  group_wait: 10s
  group_interval: 1m
  repeat_interval: 1h

receivers:
  - name: slack-notifications
    slack_configs:
      - api_url: "<SLACK_WEBHOOK_URL>"
        channel: "#<CHANNEL_NAME>"
        send_resolved: true
        title: "{{ .Status | toUpper }}: {{ .CommonLabels.alertname }}"
        text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"
```

**Replace:**

* `<SLACK_WEBHOOK_URL>` → the webhook URL from Step 1 (keep the double quotes)
* `<CHANNEL_NAME>` → the Slack channel name, without `#` (the `#` is already written)

**Step 3:** Save, then validate and restart:

```bash
docker run --rm --entrypoint amtool -v "$PWD/alertmanager:/etc/alertmanager:ro" prom/alertmanager:v0.33.1 check-config /etc/alertmanager/alertmanager.yml
docker compose restart alertmanager
```

**Expected:** `SUCCESS`. Trigger Test 1 from Step 17; a message appears in the Slack channel.

> Never commit the webhook URL to Git. It is a secret.

---

## Final Result

At the end, these must be working:

* EC2 server `observability-server` with Docker and Docker Compose
* 8 containers running: `demo-app`, `prometheus`, `alertmanager`, `node-exporter`, `cadvisor`, `loki`, `alloy`, `grafana`
* Prometheus shows 5 targets **UP** and 5 alert rules
* Grafana has the data sources **Prometheus** and **Loki**, the imported dashboards, and your own **Demo App Overview** dashboard
* You can read the app logs in Grafana Explore with Loki
* You triggered and saw the alerts `InstanceDown`, `HighErrorRate` and `HighCpuUsage` in Prometheus and Alertmanager

## Key Takeaways

* **Monitoring** tells you *that* something is wrong. **Observability** helps you find *why*.
* Use the three pillars together: **metrics** (Prometheus), **logs** (Loki), and **traces** (next step).
* Watch the four golden signals: **latency, traffic, errors, saturation**.
* Prometheus **pulls** metrics, Grafana **shows** them, Alertmanager **notifies**.
* Keep all configuration in files (as in this lab), so the setup is repeatable.

## Classroom Safety Check

- [x] Theory (monitoring vs observability, pillars, golden signals, Prometheus, Grafana, alerting, logging)
- [x] AWS Console: EC2 creation, key pair, security group ports (22, 3000, 9090, 9093, 8000)
- [x] Server connection (browser and SSH)
- [x] Docker and Compose install from the official Docker page, verified
- [x] Folder structure, and every file with complete content
- [x] Config validation with `promtool`, `amtool`, and `docker compose config`
- [x] All services started and health-checked
- [x] Prometheus targets, queries, and alert rules verified
- [x] Grafana login, data sources, imported dashboards, custom dashboard (GUI steps)
- [x] Traffic generation and logs in Loki
- [x] Alert tests with expected results
- [x] Lab exercises for students
- [x] Troubleshooting, and cleanup (containers and EC2 termination)
