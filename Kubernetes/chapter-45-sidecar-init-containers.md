# Chapter 45: Sidecar and Init Containers

## 1. Big idea

A **Pod** is the smallest unit in Kubernetes. Inside a Pod you can run more than one container. The manifest kind is always `Pod` (or a Deployment that creates pods). The difference is in the `spec`:

- **Main container**: runs your real application.
- **Init container**: runs **before** the main container, does a setup job, and **finishes**. Only after it completes successfully does the main container start.
- **Sidecar container**: runs **at the same time** as the main container as a helper (for example collecting logs).

### Simple stories

- **Init container:** before you can run a project, you may need a folder or connection to exist. An init container prepares it first (like running `mkdir`), then the main container starts.
- **Sidecar container:** Batman fights alone, then he gets a sidekick and it becomes Batman and Robin. They fight side by side. The sidecar is the helper (Robin) of the main container (Batman).

---

## 2. Init container example

Folder setup:
```bash
mkdir pods && cd pods
vim init-container.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-test
spec:
  initContainers:
    - name: init-container
      image: busybox:latest
      command: ["sh", "-c", "echo Initialization started...; sleep 10; echo Initialization completed"]
  containers:
    - name: main-container
      image: busybox:latest
      command: ["sh", "-c", "echo Main container started"]
```

### What happens

1. The init container prints "Initialization started...", waits 10 seconds, prints "Initialization completed", and exits.
2. Only then the main container starts and prints "Main container started".

```bash
kubectl apply -f init-container.yaml
kubectl get pods -w
```

Status changes: `Init:0/1` then `PodInitializing` then `Completed`. In the video it took about 18 seconds in total (10 seconds sleep + image pull and start time).

### Check logs of each container

A pod can hold many containers, so you must give the container name:

```bash
kubectl logs init-test -c init-container
kubectl logs init-test -c main-container
```
If you do not give `-c` you may get an error like "container not found" or a default container message.

### Important rules learned

- You **cannot edit** the containers of a running pod that has already initialized. Kubernetes gives: "Pod updates may not change fields other than ... spec.containers[*].image". Delete the pod first (`kubectl delete pod init-test`), then apply again.
- In a shell command, use `;` to separate commands inside `sh -c "..."`.
- If a command is wrong (for example a typo like `main` instead of `echo main`), the log says `main: not found`.

---

## 3. Sidecar container example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-test
spec:
  containers:
    # Container 1: produces logs
    - name: main-container
      image: busybox:latest
      command: ["sh", "-c", "while true; do echo Hello friends >> /var/log/app.log; sleep 5; done"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log

    # Container 2: sidecar, displays logs
    - name: sidecar-container
      image: busybox:latest
      command: ["sh", "-c", "tail -f /var/log/app.log"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log

  volumes:
    - name: shared-logs
      emptyDir: {}
```

### How it works

- The **main container** runs an infinite loop (`while true`). Every 5 seconds it writes "Hello friends" into the file `/var/log/app.log`.
- The **sidecar container** runs `tail -f` on that same file and prints the new lines to the screen.
- They **share data** through a **volume** named `shared-logs` (type `emptyDir`, an empty folder that lives as long as the pod). Both containers mount it at `/var/log`.
- The volume must be defined under `spec.volumes`. If you forget this, you get "volumeMounts name not found".
- `containers` items and `volumes` must be at the correct indentation level.

```bash
kubectl apply -f sidecar.yaml
kubectl get pods
kubectl logs sidecar-test -c sidecar-container    # shows Hello friends repeatedly
kubectl logs sidecar-test -c main-container       # shows nothing (it writes to a file, not to screen)
```

So one container produces logs and the other displays them. Both run in parallel, like two brothers working together.

---

## 4. Difference: Init vs Sidecar

| | Init container | Sidecar container |
|---|---|---|
| When it runs | Before the main container | At the same time as the main container |
| How long it lives | Runs once and finishes | Runs as long as the pod runs |
| Where in YAML | `spec.initContainers` | `spec.containers` |
| Purpose | Setup, checks, waiting for dependencies | Helper: logging, proxy, syncing |
| Pod status while running | `Init:0/1` | `Running` |

---

## 5. Real-world use cases

**Init containers**
- Wait for a database (for example MySQL) to be ready before the backend app starts. The init container uses a busybox image with a command that keeps checking until MySQL is up. Then it exits and the backend starts.
- Create folders, download config files, set file permissions, run migrations.

**Sidecar containers**
- Log collectors that ship logs to another system.
- Nginx running in a pod with a container that reads its logs.
- **Istio service mesh**: every microservice gets an **Envoy sidecar proxy** that handles traffic (see Chapter 46).

---

## 6. Summary

- One pod, many containers.
- Init container: runs first, finishes, then the main container starts.
- Sidecar container: runs beside the main container as a helper.
- Use `-c <container-name>` in `kubectl logs`.
- Share data between containers with volumes such as `emptyDir`.
- You cannot change a running pod's container spec. Delete and recreate it.
