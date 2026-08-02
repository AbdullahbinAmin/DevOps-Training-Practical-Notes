# Week 2 — Day 3: Java Application Containerization with Docker

In this lab, students will:

- Clone a Java source code repository from GitHub.
- Build a custom Dockerfile using OpenJDK.
- Compile the Java source code (`.java` file) into bytecode (`.class` file) inside the Docker image.
- Run the Java application inside a Docker container.
- Push their lab code to GitHub professionally.

---

## ✅ Prerequisites (Check Before Starting)

Confirm the tools from earlier days are installed and working:

```bash
docker --version      # Docker installed
git --version         # Git installed
docker ps             # Docker daemon running (no permission error)
```

> If `docker ps` gives a **permission denied** error, run `sudo usermod -aG docker $USER`, then log out and back in (or run `newgrp docker`).

---

## 📌 Module 1: Workspace & Directory Setup

### Step 1: Set Up Project Directory

```bash
# 1. Navigate to your docker training directory
cd ~/devops-training-notes/docker

# 2. Create today's lab directory and move inside
mkdir Day-03-Java-App-Container
cd Day-03-Java-App-Container
```

**Why we run this:** Creates a dedicated workspace folder for Day 3 Java containerization lab inside your central repository structure.

**Word Breakdown:**

- `cd`: Change Directory — moves to a specified folder location.
- `mkdir`: Make Directory — creates a new folder.

### Step 2: Clone the Java Project Repository

```bash
git clone https://github.com/LondheShubham153/simple-java-docker.git .
```

**Why we run this:** Downloads the Java application source files (including `Main.java`) directly into your current working directory.

**Word Breakdown:**

- `git`: Invokes the Git version control CLI tool.
- `clone`: Copies a remote repository from GitHub to your local system.
- `https://...`: Target GitHub URL containing Java project source code.
- `.`: Tells Git to clone files directly into the current directory instead of creating a subfolder.

### Step 3: Inspect Cloned Source Files

```bash
ls -l
cat Main.java
```

**Why we run this:** Checks the list of downloaded files and views the contents of `Main.java` to understand what the Java application does before containerizing it.

**Word Breakdown:**

- `ls -l`: Lists files in long format (showing permissions, size, and modification date).
- `cat`: Concatenate — prints file content directly in terminal output.

---

## 📌 Module 2: Create the Java Dockerfile

### Step 4: Write Custom Dockerfile

```bash
nano Dockerfile
```

Paste the following configuration into your Dockerfile:

```dockerfile
# 1. Use official OpenJDK runtime image as base image
FROM openjdk:11

# 2. Set working directory inside the container
WORKDIR /app

# 3. Copy local Java source code into container working directory
COPY Main.java /app/

# 4. Compile Java code into bytecode (.class file)
RUN javac Main.java

# 5. Define default command to execute the Java application
CMD ["java", "Main"]
```

**Why we run this:** Creates a step-by-step blueprint that tells Docker how to pull Java JDK runtime, copy code into the image, compile `Main.java` using `javac`, and execute the application when a container starts.

**🔍 Dockerfile Instructions Breakdown:**

- `FROM openjdk:11`: Sets Java Development Kit version 11 as base operating environment.
- `WORKDIR /app`: Creates `/app` directory inside container filesystem and sets it as active working directory.
- `COPY Main.java /app/`: Transfers `Main.java` from host machine into `/app/` inside image.
- `RUN javac Main.java`: Compiles Java source file into executable bytecode (`Main.class`) during build time.
- `CMD ["java", "Main"]`: Specifies runtime startup command executed when container launches.

> **Heads-up:** The `COPY` and `CMD` assume the Java class is named `Main` (file `Main.java`). If the cloned repo's file or class name differs (check Step 3 output), update `COPY`, `javac`, and `CMD` to match — for example `App.java` → `CMD ["java", "App"]`.

---

## 📌 Module 3: Build & Inspect Docker Image

### Step 5: Build Docker Image

```bash
docker build -t simple-java-app:v1 .
```

**Why we run this:** Reads instructions from Dockerfile, downloads `openjdk:11`, copies `Main.java`, compiles it into bytecode, and packages everything into a read-only Docker image named `simple-java-app:v1`.

**Word Breakdown:**

- `docker`: Calls Docker daemon CLI tool.
- `build`: Compiles an image using Dockerfile.
- `-t`: Tag option — assigns a custom name and version tag (`name:tag`).
- `simple-java-app:v1`: Target image repository name and version tag.
- `.`: Build context flag — tells Docker to search for Dockerfile in current directory.

### Step 6: Verify Local Images

```bash
docker images
```

**Why we run this:** Confirms that `simple-java-app:v1` and `openjdk:11` images are saved successfully in local image cache.

**Word Breakdown:**

- `docker images`: Displays list of all available images on local host system.

---

## 📌 Module 4: Run & Monitor Container

### Step 7: Run Java Container

```bash
docker run --name my-java-container simple-java-app:v1
```

**Why we run this:** Launches a container from `simple-java-app:v1` image, executes Java bytecode (`java Main`), prints output in terminal, and completes execution.

**Word Breakdown:**

- `run`: Creates and starts a container instance from an image.
- `--name my-java-container`: Assigns custom human-readable name to container.
- `simple-java-app:v1`: Image repository name and tag used to spin up container.

### Step 8: Check Container Execution History

```bash
docker ps -a
```

**Why we run this:** Java console apps exit automatically once code execution finishes. `docker ps -a` shows container status, exit code (`Exited (0)`), and creation timestamp.

**Word Breakdown:**

- `ps`: Process Status — lists active containers.
- `-a`: All flag — displays both running and exited container instances.

### Step 9: Check Container Logs

```bash
docker logs my-java-container
```

**Why we run this:** Retrieves printed output generated by `Main.java` from container stdout logs.

**Word Breakdown:**

- `docker logs`: Fetches console logs from specified container name or container ID.

### Step 10: Clean Up Container Instance

```bash
docker rm my-java-container
```

**Why we run this:** Removes exited container instance from host filesystem to free system resources.

**Word Breakdown:**

- `rm`: Remove command — deletes target container instance.

---

## 📌 Module 5: Push Your Lab Code to GitHub (Professionally)

Save your work to your **own** GitHub repository so it becomes part of your portfolio.

### Step 11: Point Origin to Your Own Repository

```bash
# Detach the original cloned repo's remote
git remote remove origin

# Link to YOUR new empty GitHub repo (replace the URL)
git remote add origin https://github.com/<your-username>/Day-03-Java-Docker.git
```

**Why we run this:** Ensures you push to **your** repository instead of the original author's project.

### Step 12: Add a README and .gitignore (Professional Touch)

```bash
# Ignore compiled bytecode so only source + Dockerfile are tracked
echo "*.class" > .gitignore

# Create a short README describing the lab
echo "# Java App Containerization with Docker" > README.md
echo "Compiles and runs a Java app inside an OpenJDK 11 container." >> README.md
```

**Why we run this:** A `.gitignore` keeps build artifacts (`.class`) out of version control, and a `README.md` makes the repository look professional to recruiters and reviewers.

### Step 13: Stage, Commit, and Push

```bash
git add Dockerfile Main.java README.md .gitignore
git commit -m "Week2 Day3: Containerized Java app with OpenJDK 11 Dockerfile"
git branch -M main
git push -u origin main
```

**Why we run this:** Publishes your lab work to GitHub with a clean, descriptive commit and sets `main` as the default upstream branch.

**Word Breakdown:**

- `git add`: Stages the files you want to commit.
- `git commit -m`: Records a snapshot with a descriptive message.
- `git push -u origin main`: Uploads commits and sets `main` as the tracked upstream branch.

---

## 🛠️ Troubleshooting (Common Beginner Errors)

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `COPY failed: Main.java: no such file or directory` | Source file name differs from `Main.java` | Check `ls -l` output; update `COPY`/`javac`/`CMD` to the real file & class name. |
| `error: class Main is public, should be declared in a file named Main.java` | Class name ≠ file name | Rename the file to match the public class, or fix the class name. |
| Container shows `Exited (1)` | Runtime error in the Java code | Read `docker logs my-java-container` for the stack trace. |
| `docker build` can't pull `openjdk:11` | No internet / registry blocked | Check network; retry, or use an available JDK tag (e.g. `eclipse-temurin:11`). |
| `Conflict. The container name "/my-java-container" is already in use` | Container name already exists | `docker rm -f my-java-container`, then re-run. |
| `permission denied` on Docker socket | User not in docker group | `sudo usermod -aG docker $USER`, then re-login. |

---

## 🧹 Cleanup (After the Lab)

```bash
# Remove the container (if it still exists)
docker rm -f my-java-container

# Remove the image to free disk space
docker rmi simple-java-app:v1

# Verify nothing is left
docker ps -a
docker images
```

---

## 📋 Quick Command Reference

```bash
git clone <repo-url> .                              # Clone into current folder
cat Main.java                                       # Inspect source
nano Dockerfile                                     # Create the Dockerfile
docker build -t simple-java-app:v1 .                # Build image
docker run --name my-java-container simple-java-app:v1   # Run container
docker ps -a                                        # See exit status
docker logs my-java-container                       # View program output
docker rm my-java-container                         # Remove container
git push -u origin main                             # Push lab to GitHub
docker rmi simple-java-app:v1                       # Cleanup image
```
