# Practical 22 — Jobs

## Objective
Understand what a Kubernetes Job is (different from a "job" as in employment!), create one that runs a single task to completion, and observe its full lifecycle.

## Concept: What is a Job?

⚠️ Note: This is NOT about employment/career jobs (MNC vs startup) — this is a **Kubernetes resource type**.

> **A Job runs a container to perform ONE single task, and once that task finishes, the Job is done. It does not run forever like a server.**

### The Key Difference From a Deployment
- A **Deployment** (like our Nginx web server) is meant to run **continuously, forever**, serving requests all day.
- A **Job** is meant to run **once**, complete a specific task, and then **stop**.

### Real Examples of Jobs
- Taking a backup
- Applying a system patch/update
- Creating a folder or processing a file
- Any "do this task, then you're done" kind of work

### Execution Modes
A Job can run:
- **Sequentially (single)** — one task, run once, done
- **In parallel (batches)** — the same task split across multiple Pods running at the same time

This is why a Job's `apiVersion` is `batch/v1` — because Jobs are designed around "batch processing."

## Prerequisites
- KIND cluster running
- `nginx` namespace present

## Step 1 — Create the Job Manifest

```bash
vim job.yaml
```

Paste this complete YAML:

```yaml
kind: Job
apiVersion: batch/v1
metadata:
  name: demo-job
  namespace: nginx
spec:
  completions: 1
  parallelism: 1
  template:
    metadata:
      name: demo-job-pod
      labels:
        app: batch-task
    spec:
      containers:
        - name: batch
          image: busybox:latest
          command: ["sh", "-c", "echo Hello Friends && sleep 10"]
      restartPolicy: Never
```

**Explanation of every field:**
- `kind: Job` → this manifest describes a one-time Job.
- `apiVersion: batch/v1` → Jobs (and CronJobs) use the `batch/v1` API group, since they're built around batch processing.
- `metadata.name` → name of this Job (`demo-job`).
- `metadata.namespace` → the namespace it belongs to.
- `spec.completions: 1` → how many times this task needs to successfully complete — here, just 1 (run once and be done).
- `spec.parallelism: 1` → how many Pods run at the same time — here, just 1 (sequential, not parallel). If you set this to `2`, it would create 2 Pods running the same task simultaneously.
- `spec.template` → the Pod template for the task, same structure as a normal Pod:
  - `image: busybox:latest` → **BusyBox** is a special lightweight container image whose entire purpose is to **run commands** — unlike Nginx (which serves websites) or MySQL (which is a database), BusyBox is just for executing shell commands.
  - `command: ["sh", "-c", "echo Hello Friends && sleep 10"]` → runs a shell command: print "Hello Friends," then wait 10 seconds. This simulates a quick task with a short delay.
  - `restartPolicy: Never` → **critical field, specific to Jobs.** Since this task should run once and finish, we do NOT want Kubernetes to restart the container after it completes. Options are `Never` or `OnFailure` (restart only if it fails) — a Job cannot use `Always` (unlike Pods).

⚠️ **Important:** `restartPolicy` belongs directly under `spec.template.spec`, NOT under the Job's top-level `spec`. This is a common mistake.

**Save and exit** (`Esc`, `:wq`, Enter).

## Step 2 — Apply the Job

```bash
kubectl apply -f job.yaml
```

### If Something Goes Wrong
If you see an error like:
```text
error: error validating "job.yaml": error validating data: ValidationError(Job.spec): unknown field "restartPolicy" in io.k8s.api.batch.v1.JobSpec
```

**Reason:** `restartPolicy` was placed at the wrong indentation level (under the Job's `spec` instead of under `spec.template.spec`).

**Fix:** Move `restartPolicy` inside `template.spec`, matching the exact structure shown in Step 1, then reapply:
```bash
kubectl apply -f job.yaml
```

**Expected output:**
```text
job.batch/demo-job created
```

## Step 3 — Watch the Job's Lifecycle

Immediately check the Pod status (it will still be running the 10-second sleep):
```bash
kubectl get pods -n nginx
```
**Expected output (while running):**
```text
NAME                    READY   STATUS    RESTARTS   AGE
demo-job-xxxxx          1/1     Running   0          3s
```

Wait about 10 seconds, then check again:
```bash
kubectl get pods -n nginx
```
**Expected output (after completion):**
```text
NAME                    READY   STATUS      RESTARTS   AGE
demo-job-xxxxx          0/1     Completed   0          15s
```

**This confirms the Pod's full lifecycle:**
```text
ContainerCreating → Running → Terminating → Completed
```

⚠️ **Interview Tip:** If asked "what are the different Pod states you know?", mention: `Pending`, `ContainerCreating`, `Running`, `Terminating`, `Completed`, `CrashLoopBackOff`, `ImagePullBackOff`, `Error`.

## Step 4 — Check the Job's Logs

```bash
kubectl logs job/demo-job -n nginx
```
**Expected output:**
```text
Hello Friends
```
This confirms the echo command actually ran and printed the expected output.

## Step 5 — Re-run the Job

Once a Job completes, it will NOT restart on its own even if you `apply` the same file again (nothing changed). To run it again, you must delete and recreate it:

```bash
kubectl delete -f job.yaml
kubectl apply -f job.yaml
```

**Verify:**
```bash
kubectl get pods -n nginx
```
**Expected output:** A brand-new Pod in `Running` state, which will again become `Completed` after ~10 seconds.

```bash
kubectl logs job/demo-job -n nginx
```
**Expected output:**
```text
Hello Friends
```

## Step 6 — Clean Up

```bash
kubectl delete -f job.yaml
```

## Key Concept Recap
> A **Job** = "run this container's task ONE time, then stop." Once the Pod's task is done, its status changes to `Completed` and it will NOT restart automatically, since `restartPolicy: Never` was set.

## Final Result
Students now understand:
- What a Job is and how it's different from long-running resources like Deployments
- The full Pod lifecycle: `ContainerCreating → Running → Terminating → Completed`
- The purpose of the BusyBox image (running arbitrary shell commands)
- The importance and correct placement of `restartPolicy` in a Job's YAML
- How to check Job logs and re-run a completed Job

---
**Prerequisite for Chapter 23:** Full understanding of Jobs is required, since a CronJob is essentially a Job that runs on a repeating schedule.
