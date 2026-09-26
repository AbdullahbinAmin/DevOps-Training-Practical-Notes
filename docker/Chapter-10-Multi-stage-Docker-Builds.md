# Chapter 10 — Multi-stage Docker Builds

## Objective
Understand why Docker images can become unnecessarily large, and learn how to use multi-stage Dockerfile builds to dramatically shrink image size — using the Flask app from earlier chapters as a live example (shrinking it from over 1 GB down to under 150 MB).

## Prerequisites
* OS: Ubuntu EC2 instance with Docker installed
* Required software: Docker
* Required setup: The Flask app project from Chapter 5 (or any similar app with a Dockerfile)
* Required knowledge: Dockerfile basics (Chapter 5)

## Concept — Why Images Get Large

**The problem demonstrated:**
```bash
cd ~/projects/flask-app
docker build -t flask-app:latest .
docker images
```
**Observed result:** The `flask-app` image size is roughly **1.01 GB** — quite large for what is a very simple application.

**Why this happens:**
An image's total size is basically:
```text
Total Image Size = Base Image Size + Your Code + Installed Dependencies
```
The **base image** (`FROM python:3.7` in this case) is itself very large (around 994 MB), because it contains a full Python installation with many tools that are only needed to INSTALL dependencies — not to actually RUN the finished application.

**The key insight:**
You need a big/full base image to **install** dependencies and compile things, but once that is done, you do NOT need that big image anymore just to **run** the already-built application. This is exactly the gap that multi-stage builds solve.

## Concept — What is a "Stage" in a Dockerfile?

**Definition:** Every time you write a `FROM` instruction in a Dockerfile, that starts a NEW **stage**. A traditional (single-stage) Dockerfile has only one `FROM`, meaning everything — install, build, AND run — happens inside that one big image.

A **multi-stage build** uses **two (or more) `FROM` instructions**:
1. **Stage 1 (the "builder" stage):** Uses a large/full base image to install dependencies and prepare everything needed.
2. **Stage 2 (the final stage):** Uses a small, lightweight base image, and simply **copies only the necessary finished files** (not the whole toolchain) from Stage 1.

The final image only contains what Stage 2 has — meaning the large tools used only for installation in Stage 1 are never included in the final image, dramatically reducing size.

## Step 1 — The Original (Single-Stage) Dockerfile
```dockerfile
FROM python:3.7
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
ENTRYPOINT ["python"]
CMD ["run.py"]
```
**Verify current size:**
```bash
docker build -t flask-app:latest .
docker images
```
**Expected result:** `flask-app` shows approximately **1.01 GB**.

## Step 2 — Rewrite the Dockerfile Using Multi-stage Build

```bash
nano Dockerfile
```

```dockerfile
# ---------- Stage 1: Builder ----------
FROM python:3.7 AS builder

WORKDIR /app

COPY . .

RUN pip install -r requirements.txt

# ---------- Stage 2: Final (small) image ----------
FROM python:3.7-slim

WORKDIR /app

COPY --from=builder /usr/local/lib/python3.7/site-packages /usr/local/lib/python3.7/site-packages
COPY --from=builder /app /app

CMD ["python", "run.py"]
```

**Explanation, line by line:**

| Line | Meaning |
|---|---|
| `FROM python:3.7 AS builder` | Starts Stage 1, and gives it a nickname/alias: `builder`. This is the large base image, used ONLY for installing dependencies. |
| `WORKDIR /app` | Sets up the working directory inside Stage 1's temporary image. |
| `COPY . .` | Copies your source code into Stage 1 so `pip install` has access to `requirements.txt`. |
| `RUN pip install -r requirements.txt` | Installs the dependencies. In this stage, they get installed into a specific system folder (Python's "site-packages" directory). |
| `FROM python:3.7-slim` | Starts Stage 2, a brand-new, much smaller base image (`slim` variant of Python 3.7 — around 125 MB instead of 994 MB). |
| `WORKDIR /app` | Sets up the working directory for the FINAL image. |
| `COPY --from=builder /usr/local/lib/python3.7/site-packages /usr/local/lib/python3.7/site-packages` | **This is the key multi-stage instruction.** It copies ONLY the already-installed Python packages from the `builder` stage into this new, small image — without needing to re-run `pip install` or carry over the large base image. |
| `COPY --from=builder /app /app` | Copies your actual application source code from the `builder` stage into the final image. |
| `CMD ["python", "run.py"]` | Runs the application using the final, lightweight image. |

**How did we know the exact site-packages path?**
This is Python's standard internal location where `pip` installs packages when using a Python base image: `/usr/local/lib/python<version>/site-packages`. You can confirm this by checking the official Python Docker Hub documentation, or by exploring inside a running container (`docker exec -it <container> bash` then `python -c "import site; print(site.getsitepackages())"`).

**Save and exit.**

## Step 3 — Build the New Multi-stage Image
```bash
docker build -t flask-app-mini:latest .
```
**Expected output:** You will notice the build log clearly shows two separate stages executing (e.g., `[1/2]`, `[2/2]` groupings, or numbered steps per stage).

## Step 4 — Compare the Sizes
```bash
docker images
```
**Expected result (example):**
```text
REPOSITORY        TAG       SIZE
flask-app-mini    latest    142MB
flask-app         latest    1.01GB
```
**This is the power of multi-stage builds and caching** — the final image dropped from over 1 GB down to roughly 140 MB, simply by separating the "install/build" concerns from the "run" concerns.

## Step 5 — Run and Verify the Slim Image Still Works
```bash
docker run -d -p 80:80 flask-app-mini:latest
```
**Verify:**
```bash
docker ps
```
Open your browser at:
```text
http://<YOUR_EC2_PUBLIC_IP>:80
```
**Expected result:** The application works exactly the same as before — proving multi-stage builds do not remove functionality, only unnecessary bloat.

## Concept Recap — Why This Works So Well

```text
Stage 1 (builder):  python:3.7 (~994 MB) + pip install → produces installed packages
Stage 2 (final):    python:3.7-slim (~125 MB) + COPY only installed packages + app code
                     = ~140 MB final image (large builder stage is discarded entirely)
```

The large `python:3.7` image is only ever used temporarily during the build process — it is never shipped or stored as part of your final image. Docker automatically discards intermediate stages once the build finishes (unless you explicitly reference them again).

## Troubleshooting

### Error 1
```text
ModuleNotFoundError: No module named 'flask'
```
after switching to multi-stage build.
**Reason:** The `COPY --from=builder` path for site-packages does not exactly match the Python version used, or packages were installed into a different location (e.g., using `pip install --user`).

**Fix:** Ensure both stages use the EXACT SAME Python version (e.g., both `python:3.7` and `python:3.7-slim`, not mismatched versions like 3.7 and 3.9), and confirm the site-packages path matches that exact version.

### Error 2
Final image still large.
**Reason:** You forgot to also switch the second `FROM` to a slim/smaller base image, and are instead using the same large image for both stages.

**Fix:** Make sure Stage 2 uses a genuinely smaller variant (e.g., `-slim` or `-alpine` tag).

### Error 3
```text
COPY failed: no source files were specified
```
**Reason:** Typo or incorrect path in the `--from=builder` copy source path.

**Fix:** Double-check the exact folder paths used inside the builder stage (e.g., re-verify with `docker exec -it` into an intermediate build if needed, or add a temporary `RUN find / -name "site-packages"` command to locate it).

## Cleanup
```bash
docker stop <CONTAINER_ID>
docker rm <CONTAINER_ID>
docker rmi flask-app flask-app-mini
```

## Final Result
* You understand what a "stage" is in a Dockerfile, and why multiple `FROM` instructions can be used.
* You rewrote a Flask app's Dockerfile using a multi-stage build.
* The final image size dropped from ~1.01 GB to ~140 MB, while the application still works identically.
* You understand this same pattern (a "builder" stage + a "final" stage) can be applied to any language/framework (Java, Node.js, etc.) to reduce final image size significantly.

---

## Bonus Section — Monitoring and Logging in Docker

### Objective
Learn how to view container logs, and how to redirect a container's live logs into a file for later analysis.

### Step 1 — Run a Container to Monitor
```bash
docker run -d -p 80:80 flask-app-mini:latest
```

### Step 2 — View Logs (One-Time Snapshot)
```bash
docker ps
docker logs <CONTAINER_ID>
```
**What this does:** Displays all log output produced by the container so far — useful for debugging why something isn't working.

### Step 3 — Redirect Live Logs Into a File Using `nohup` + `docker attach`
```bash
docker ps
nohup docker attach <CONTAINER_ID> &
```
- `nohup` → Runs the following command in a way that keeps it running in the background even if you close the terminal, and redirects its output into a file called `nohup.out` by default.
- `docker attach <CONTAINER_ID>` → Attaches your terminal session directly to the container's live output stream.
- `&` → Runs the whole command in the background of your current shell.

**Expected output:**
```text
nohup: ignoring input and appending output to 'nohup.out'
```

### Step 4 — Generate Some Traffic and Check the Log File
Visit a few different routes in your browser (e.g., `/health`, `/error`, `/random-path`), then check:
```bash
cat nohup.out
```
**Expected result:** Every request you made (health checks, errors, random paths) appears logged inside this file — this is a simple but effective way to collect logs from a container into a persistent file on the host.

### Why This Matters
In real production environments, dedicated logging/monitoring tools (like the ELK stack, Prometheus + Grafana, or cloud-native logging services) are used instead of `nohup`, but the underlying concept — capturing a container's log stream somewhere durable — is the same idea, just automated and centralized at scale.

## What's Next
Chapter 11 gives a conceptual introduction to Kubernetes and container orchestration — why Docker containers alone are usually not run directly in production.
