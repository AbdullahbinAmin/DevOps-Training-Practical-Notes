# Chapter 5 — Writing a Dockerfile (Java App + Python/Flask App)

## Objective
Learn to write a Dockerfile from scratch to containerize a simple Java application, understand every Dockerfile instruction (FROM, WORKDIR, COPY, RUN, CMD, ENTRYPOINT), then repeat the same process for a Python Flask application, including port mapping and running in the background.

## Prerequisites
* OS: Ubuntu EC2 instance with Docker installed (Chapters 3–4 completed)
* Required software: `git`, Docker
* Required account: None
* Required repository: A simple Java project (example repo used: `laundeShubham153/simple-java-docker`) and a Flask project (example repo used: `laundeShubham153`'s Flask app)
* Required ports: Port 80 (for the Flask app, opened via AWS Security Group)
* Required permissions: Sudo access to install `git` if not already present

## Pre-Check
```bash
git --version
docker --version
```
**What this does:** Confirms git and Docker are both available before starting.

## Concept — What Goes Into a Dockerfile

**Analogy used:** Think of making instant noodles (Maggi). The recipe has fixed steps: (1) take a pot, (2) add water, (3) add the noodles/masala and heat it, (4) turn on the gas and cook. Everyone might do step order slightly differently (some add masala before boiling, some after), but the steps are still clearly defined and sequential.

A Dockerfile works the same way — it is a **recipe/blueprint** containing a sequence of instructions describing:
1. What base environment is needed (e.g., which OS/runtime).
2. Where the application code should live inside the container.
3. How to copy the code in.
4. What commands to run to prepare the app (install dependencies, compile, etc.).
5. What command should run when the container finally starts.

---

## Part A — Directory / File Structure (Java App)

```text
projects/
└── simple-java-docker/
    ├── Dockerfile
    ├── Dummy
    └── src/
        └── Main.java
```

## Part A — Step-by-Step: Java Application Dockerfile

### Step 1 — Create a Projects Folder
```bash
mkdir projects
cd projects
```
**What this does:** Creates a dedicated folder to keep all your practice projects organized as you go through this course.

### Step 2 — Clone the Java Sample Project
```bash
git clone https://github.com/LaundeShubham153/simple-java-docker
```
**What this does:** Downloads a simple, ready-made Java application (with a plain `Main.java` file that prints the current date) to your server.

**Verify:**
```bash
ls
```
**Expected result:** A folder named `simple-java-docker` should now be present.

```bash
cd simple-java-docker
ls
```
**Expected result:** You should see `Dockerfile`, `Dummy`, and `src` (the folder containing `Main.java`).

### Step 3 — Remove the Existing Dockerfile (We Will Write It Ourselves)
```bash
rm -v Dockerfile
```
- `-v` → Verbose mode; shows confirmation of what was deleted.

**Expected result:** Output confirms `removed 'Dockerfile'`.

### Step 4 — Create a New Dockerfile
```bash
nano Dockerfile
```
**What this does:** Opens the `nano` text editor to create/edit a file named `Dockerfile` in the current directory.

Type the following content into the Dockerfile:

```dockerfile
FROM openjdk:17-alpine

WORKDIR /app

COPY . .

RUN javac src/Main.java

CMD ["java", "-cp", "src", "Main"]
```

**Explanation of each instruction:**

| Instruction | Meaning |
|---|---|
| `FROM openjdk:17-alpine` | Pulls a **base image** that already has Java Development Kit (JDK) version 17 installed, using the lightweight `alpine` variant (small image size). This saves you from installing Java manually. |
| `WORKDIR /app` | Creates (or switches into) a folder called `/app` inside the container — this becomes the default working directory for all following instructions. |
| `COPY . .` | Copies everything from your current host folder (the build context, `.`) into the container's current working directory (`/app`). |
| `RUN javac src/Main.java` | Executes a command **at build time** — here it compiles the Java source file. Java code must be compiled (`javac`) before it can be executed. |
| `CMD ["java", "-cp", "src", "Main"]` | The default command that runs **when the container starts** (not during build). It runs the compiled `Main` class using the classpath `src`. |

**Save the file and exit** (in `nano`: press `Ctrl+X`, then `Y`, then `Enter`).

**Verify:**
```bash
cat Dockerfile
```
**Expected result:** The file content should match exactly what you typed above.

### Step 5 — Understand CMD vs RUN vs ENTRYPOINT (Important Concept)

* Every Dockerfile instruction (`FROM`, `WORKDIR`, `COPY`, `RUN`) creates an **intermediate layer** used while **building the image**.
* `CMD` is special — it does NOT run during build. It only runs **after the container is created**, as the default startup command.
* `CMD` **can be overridden** — when you run `docker run <image> some-other-command`, that overrides the CMD.
* `ENTRYPOINT` is similar to CMD but **cannot be overridden** as easily — it always runs, and any extra arguments you pass get appended to it instead of replacing it.

**Analogy used:** If your container is a sealed bottle full of water, `CMD`/`ENTRYPOINT` is like the straw you use to drink from it — the container's contents are ready, but you need a defined way (a command) to actually access/run them.

### Step 6 — Build the Docker Image
```bash
docker build -t java-app .
```
- `-t java-app` → Tags (names) the resulting image `java-app`.
- `.` → The **build context** — meaning "use the current directory," which is where the Dockerfile and source code are located.

**Expected output:** You should see multiple numbered steps executing, for example:
```text
Step 1/5 : FROM openjdk:17-alpine
Step 2/5 : WORKDIR /app
...
Removing intermediate container ...
Successfully built ...
Successfully tagged java-app:latest
```

**Concept check:** Each `Step X out of Y` corresponds to one **layer**. Docker builds a container for each layer, then removes the intermediate container, keeping only the final resulting layer — this is what makes Docker images efficient to rebuild (via caching).

**Verify:**
```bash
docker images
```
**Expected result:** `java-app` should appear in the list, created just a few seconds ago.

### Step 7 — Run the Container
```bash
docker run java-app
```
**Expected output:** Something like:
```text
Hello Docker Current date is Wednesday, November 6
```
(the exact greeting text and date depend on your `Main.java` source code and the day you run it).

### Step 8 — Modify the Code and Rebuild (Understanding Image Updates)

1. Edit the source file:
```bash
nano src/Main.java
```
Change the printed message text (for example, from "Hello Docker" to "Hello friends, please subscribe").

2. Try running the OLD image again without rebuilding:
```bash
docker run java-app
```
**Expected result:** The OLD output still appears — this is expected! **The image does not automatically pick up code changes on the host.** You must rebuild.

3. Rebuild the image:
```bash
docker build -t java-app .
```
**What happens:** Docker will reuse the cached layers for `FROM` and `WORKDIR` (since they haven't changed), but it will re-run the `COPY` and `RUN javac` steps because the source code changed — you'll see `Using cache` for unchanged steps and fresh execution for changed ones.

4. Run the container again:
```bash
docker run java-app
```
**Expected result:** Now the NEW output/message appears, confirming the image was updated correctly.

**Key Takeaway:** Any time your application source code changes, you MUST rebuild the image (`docker build`) before running a container to see the changes; simply re-running `docker run` on the old image will not reflect new code.

---

## Part B — Directory / File Structure (Python Flask App)

```text
projects/
└── flask-app/
    ├── Dockerfile
    ├── app.py
    ├── run.py
    └── requirements.txt
```

## Part B — Step-by-Step: Python Flask Application Dockerfile

### Step 1 — Get the Flask Project
```bash
cd ~/projects
git clone <YOUR_FLASK_APP_REPO_URL>
cd flask-app
```
**What this does:** Clones a Flask-based web application. Flask is a lightweight Python web framework.

### Step 2 — Understand the Project Files
```bash
cat app.py
cat run.py
cat requirements.txt
```
**What each file is:**
* `app.py` → Contains the actual Flask application code (routes, logic).
* `run.py` → The file used to actually start/run the application.
* `requirements.txt` → Lists Python package dependencies (e.g., `Flask==2.2` — a specific version).

### Step 3 — Remove Any Existing Dockerfile
```bash
rm Dockerfile
```

### Step 4 — Create a New Dockerfile
```bash
nano Dockerfile
```

```dockerfile
FROM python:3.7

WORKDIR /app

COPY . .

RUN pip install -r requirements.txt

ENTRYPOINT ["python"]
CMD ["run.py"]
```

**Explanation:**

| Instruction | Meaning |
|---|---|
| `FROM python:3.7` | Base image containing Python 3.7 already installed. (Use whichever version your developer/project specifies — check with a quick search like "how to install python requirements.txt" if unsure of the exact pip command.) |
| `WORKDIR /app` | Sets the working directory inside the container. |
| `COPY . .` | Copies your Flask project code into the container. |
| `RUN pip install -r requirements.txt` | Installs all Python dependencies listed in `requirements.txt`, at build time. |
| `ENTRYPOINT ["python"]` | This part will ALWAYS run and cannot be easily overridden — it ensures `python` is always the interpreter used. |
| `CMD ["run.py"]` | This is the default argument passed to the ENTRYPOINT (so effectively `python run.py` runs by default) — but this part CAN be overridden if you pass a different command when running the container. |

**Save and exit.**

### Step 5 — Build the Image
```bash
docker build -t flask-app .
```
**Expected output:** Downloads the Python base image (large, ~200+ MB), installs requirements, and finishes with:
```text
Successfully tagged flask-app:latest
```
> This will be optimized later in Chapter 10 (Multi-stage Docker Builds) to shrink this large image size significantly.

### Step 6 — Find Which Port the App Uses
Check with your developer or the app's documentation/code. In this example, the Flask app runs on **port 80**.

### Step 7 — Run With Port Mapping
```bash
docker run -d -p 8080:80 flask-app
```
- `-d` → Detached (background) mode.
- `-p 8080:80` → Maps **host port 8080** to **container port 80**. Format is always `<HOST_PORT>:<CONTAINER_PORT>`.
  - `<HOST_PORT>` → The port on YOUR server/machine that you will actually browse to.
  - `<CONTAINER_PORT>` → The port the application is listening on INSIDE the container (do not change this unless you also change the app's config).

**Why port mapping is needed:**
The Docker Engine shares the host's networking resources but keeps each container's internal ports isolated by default. Port mapping (also called "publishing" a port) creates a bridge/binding between a port on your host machine and a port inside the container, so external traffic can reach the app.

### Step 8 — Open the Port on AWS (Security Group)
1. Go to your EC2 instance in AWS Console → **Security** tab → click the linked Security Group.
2. Go to **Inbound rules → Edit inbound rules → Add rule**.
3. Type: **Custom TCP**, Port range: **8080**, Source: **Anywhere (0.0.0.0/0)** (for learning purposes only — restrict this in real production use).
4. Click **Save rules**.

### Step 9 — Access the Application
Open your browser and go to:
```text
http://<YOUR_EC2_PUBLIC_IP>:8080
```
**Expected result:** The Flask application's homepage loads (e.g., "Welcome to DevOps").

Try the health check route:
```text
http://<YOUR_EC2_PUBLIC_IP>:8080/health
```
**Expected result:** A message like `Server is up and running`.

### Step 10 — View Live Logs (Debugging)
```bash
docker ps
docker logs <CONTAINER_ID>
```
**What this does:** Shows the container's logged output, including which routes were hit (e.g., `/health`, `/profile`, or a 404 for unknown routes).

To watch logs **live in real time** instead of a one-time snapshot:
```bash
docker attach <CONTAINER_ID>
```
**What this does:** Attaches your terminal directly to the container's console output — as you refresh the app in your browser, new log lines appear instantly.

> ⚠️ Attaching blocks your terminal, similar to running a container in foreground mode. To detach without stopping the container, try `Ctrl+P` then `Ctrl+Q` (this may not always work depending on your terminal — if it doesn't, open a second terminal/SSH session as shown in Chapter 4).

### Step 11 — Stop and Start Containers
```bash
docker ps
docker stop <CONTAINER_ID>
```
**Verify it stopped:**
```bash
docker ps -a
```
**Expected result:** Status shows `Exited`; browsing the app URL now fails ("site can't be reached").

Restart the same container (no need to rebuild or run again):
```bash
docker start <CONTAINER_ID>
```
**Verify:**
```bash
docker ps
```
Then refresh the browser — the app should be reachable again.

### Step 12 — Enter a Running Container's Shell (Useful for Debugging Databases Too)
```bash
docker ps
docker exec -it <CONTAINER_ID> bash
```
- `exec` → Execute a command inside a running container.
- `-it` → Interactive terminal (so you get a usable shell prompt).
- `bash` → The shell to start inside the container.

**Expected result:** Your terminal prompt changes to something like `root@<container-id>:/app#`, meaning you are now literally inside the running container's filesystem.

Type `exit` to leave the container's shell and return to your host terminal.

## Troubleshooting

### Error 1
```text
javac: file not found: src/Main.java
```
**Reason:** The `COPY . .` step did not run before `RUN javac`, or your project folder structure does not match what the Dockerfile expects.

**Fix:** Ensure `COPY . .` comes before the `RUN javac` line in the Dockerfile, and that `src/Main.java` genuinely exists in your project folder (`ls src/`).

### Error 2
```text
This site can't be reached — <IP>:8080 took too long to respond
```
**Reason:** The Security Group inbound rule for the port (e.g., 8080) has not been added yet.

**Fix:** Add an inbound rule for that port in the EC2 Security Group settings (see Step 8 above).

### Error 3
Old code/output appears even after editing the source file.
**Reason:** You forgot to rebuild the image after changing source code.
**Fix:** Run `docker build -t <image-name> .` again before running a new container.

## Cleanup
```bash
docker ps -a
docker stop <CONTAINER_ID>
docker rm <CONTAINER_ID>
docker rmi java-app flask-app
```
**What this does:** Stops, removes containers, and removes the images built in this chapter, to keep the environment clean for the next chapter (optional — you can also keep them if you plan to reuse them).

## Final Result
* You wrote a Dockerfile from scratch for a Java application and successfully built + ran it.
* You wrote a Dockerfile from scratch for a Python Flask application, mapped a port, opened it on AWS, and accessed it via a browser.
* You understand `FROM`, `WORKDIR`, `COPY`, `RUN`, `CMD`, and `ENTRYPOINT`, and the difference between build-time and run-time instructions.
* You know how to view logs (`docker logs`, `docker attach`) and get an interactive shell inside a running container (`docker exec -it ... bash`).

## What's Next
Chapter 6 covers Docker Networking — how two separate containers (like a Flask app and a MySQL database) can talk to each other.
