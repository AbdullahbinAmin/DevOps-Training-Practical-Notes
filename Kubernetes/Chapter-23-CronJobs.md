# Practical 23 — CronJobs

## Objective
Understand what a CronJob is, learn basic Cron scheduling syntax, build a CronJob that performs a repeating backup task, and learn to debug common YAML structure errors along the way.

## Concept: What is a CronJob?

> **Cron** means "following a particular schedule." A **CronJob** is a Job (from Chapter 22) that gets created and run automatically, again and again, according to a defined time schedule.

Unlike a plain Job (which runs once and is done), a CronJob keeps creating new Job instances repeatedly, based on the schedule you define — for example, every minute, every hour, every day, etc.

## Prerequisites
- Completion of Chapter 22 (Jobs) — a CronJob's `jobTemplate` uses the exact same structure as a Job.
- KIND cluster running, `nginx` namespace present

## Part A — Understanding Cron Schedule Syntax

A cron schedule has 5 fields, always in this exact order:

```text
*     *     *     *     *
Min   Hour  Day   Month DayOfWeek
```

- **Minute** (0–59)
- **Hour** (0–23)
- **Day of Month** (1–31)
- **Month** (1–12)
- **Day of Week** (0–6)

### What `*` Means
The asterisk (`*`) means "every" for that field.
- `* * * * *` → every minute, of every hour, of every day, of every month, of every day of the week → basically, **run every single minute**.

### Useful Examples
- `*/2 * * * *` → every 2nd minute
- `0 */2 * * *` → at minute 0, every 2nd hour (i.e., every 2 hours)

⚠️ **Tip:** Cron syntax can feel confusing at first. You can always search for a "crontab guru" tool online (search: **crontab guru**) — it's a website where you type a schedule and it explains it in plain English, and vice versa.

## Part B — Build the CronJob (Backup Task Example)

### Step 1 — Create the CronJob Manifest

```bash
vim cronjob.yaml
```

We will build this step by step so you understand each part. Here is the **final, correct, working version**:

```yaml
kind: CronJob
apiVersion: batch/v1
metadata:
  name: minute-backup
  namespace: nginx
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: backup-task
        spec:
          containers:
            - name: minute-backup-container
              image: busybox:latest
              command:
                - sh
                - -c
                - |
                  echo Backup Started
                  mkdir -p /backups
                  mkdir -p /demo-data
                  cp -r /demo-data/. /backups/
                  echo Backup Completed
              volumeMounts:
                - name: my-volume
                  mountPath: /demo-data
                - name: backup-volume
                  mountPath: /backups
          volumes:
            - name: my-volume
              hostPath:
                path: /demo-data
                type: DirectoryOrCreate
            - name: backup-volume
              hostPath:
                path: /backups
                type: DirectoryOrCreate
          restartPolicy: OnFailure
```

**Explanation of every field:**
- `kind: CronJob` → this manifest describes a CronJob.
- `apiVersion: batch/v1` → same API group as Jobs, since CronJobs are built on top of Jobs.
- `metadata.name` → name of the CronJob (`minute-backup`).
- `metadata.namespace` → the namespace (`nginx`).
- `spec.schedule` → the cron expression (in quotes) — `"* * * * *"` means "run every minute."
- `spec.jobTemplate` → **this is the important structural concept.** A CronJob's spec doesn't directly contain a Pod template — it contains a **Job template**, and that Job template itself contains the actual Pod template inside it. This nesting is: `CronJob.spec.jobTemplate.spec.template.spec.containers`.
- `command` → this task does the following, one line at a time:
  1. Print "Backup Started"
  2. Create a folder called `/backups` (if it doesn't already exist)
  3. Create a folder called `/demo-data` (if it doesn't already exist)
  4. Copy everything from `/demo-data` into `/backups`
  5. Print "Backup Completed"
- `volumeMounts` → tells the **container** where to plug in the volumes defined below (mounting `my-volume` at path `/demo-data`, and `backup-volume` at path `/backups`).
- `volumes` → defined at the **Pod level** (not inside the container) — this is where the actual volume is created:
  - `hostPath` → a type of volume that maps directly to a folder on the **host machine** (the Worker Node's actual filesystem). This is a basic form of storage — Persistent Volumes (a more advanced/proper method) are covered in Chapter 25.
  - `path: /demo-data` → the folder on the host machine
  - `type: DirectoryOrCreate` → if this folder doesn't exist on the host yet, Kubernetes will create it automatically
- `restartPolicy: OnFailure` → if this task fails, restart it; but don't restart if it already completed successfully.

⚠️ **Why Volumes Are Needed Here:** Without a container, data created inside a Pod is lost when the Pod is deleted/recreated (this will be explained in detail in Chapter 24 - Storage). Since this backup task creates and copies files, we need to persist that data properly using a `hostPath` volume.

### Step 2 — Apply the CronJob (and Learn From Errors)

This section deliberately walks through the **exact errors you may encounter** while building complex nested YAML like this, since fixing errors is a core skill.

```bash
kubectl apply -f cronjob.yaml
```

#### Error 1 — Labels in Wrong Place
```text
error: error validating "cronjob.yaml": ... unknown field "labels" in ...JobTemplateSpec.spec.template.spec
```
**Reason:** `labels` was placed inside `spec` instead of inside `metadata`.
**Fix:** Move `labels:` under `template.metadata.labels`, not `template.spec.labels`.

#### Error 2 — Wrong Nesting of spec.spec
```text
error: error validating "cronjob.yaml": ... unknown field "spec" in ...JobTemplateSpec.spec
```
**Reason:** An extra/incorrect `spec:` was added directly under `jobTemplate.spec`, when it should flow through `template.spec` instead.
**Fix:** Follow the exact nesting order: `jobTemplate.spec.template.spec.containers` — no shortcuts.

#### Error 3 — Volumes Under Containers Instead of Pod Spec
```text
error: error validating "cronjob.yaml": ... unknown field "volumes" in ...Container
```
**Reason:** `volumes` was mistakenly placed under `containers` — but `volumes` belongs to the **Pod's spec level**, while `volumeMounts` (note the different name) belongs **inside each container**.
**Fix:** Keep `volumeMounts` inside the container block, and move `volumes` outside, directly under `template.spec` (same level as `containers` and `restartPolicy`).

#### Error 4 — Invalid restartPolicy Value
```text
error: error validating "cronjob.yaml": ... spec.template.spec.restartPolicy: Required value: valid values: "OnFailure", "Never"
```
**Reason:** A CronJob's/Job's Pod template cannot use `restartPolicy: Always` (unlike normal Deployments) — only `OnFailure` or `Never` are valid, since these are meant to be finite tasks, not always-running services.
**Fix:** Use `OnFailure` (or `Never`), as shown in the final YAML above.

⚠️ **Learning Note:** These are exactly the kinds of errors you WILL encounter when writing complex nested Kubernetes YAML. Learning to read the error message carefully (it usually tells you the exact field and location) and fix it is a critical real-world DevOps skill — don't be afraid of these errors.

**Once fixed, expected final output:**
```text
cronjob.batch/minute-backup created
```

### Step 3 — Verify the CronJob

```bash
kubectl get cronjob -n nginx
```
**Expected output:**
```text
NAME             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
minute-backup    * * * * *     False     1        38s             1m
```
This confirms the CronJob is active and has already been scheduled at least once.

### Step 4 — Watch It Run

```bash
kubectl get pods -n nginx
```
**Expected output:** A new Pod appears roughly every minute, running the backup task, then completing:
```text
NAME                          READY   STATUS      RESTARTS   AGE
minute-backup-xxxxxxx-aaaaa   0/1     Completed   0          40s
minute-backup-xxxxxxx-bbbbb   1/1     Running     0          5s
```

**Check logs of a completed run:**
```bash
kubectl logs pod/<pod-name> -n nginx
```
**Expected output:**
```text
Backup Started
Backup Completed
```

⚠️ **Tip:** If you don't want to wait a full minute during a live class, temporarily change the schedule to `*/1 * * * *` (every minute — same as `* * * * *`) or test with the Job from Chapter 22 first to confirm the container logic works, then apply the CronJob.

### Step 5 — Watch Multiple Runs Happen Automatically

Wait a couple of minutes and check again:
```bash
kubectl get pods -n nginx
```
**Expected output:** You should now see multiple completed Pods — one for each minute that has passed — each one having run the backup task independently and automatically, without you doing anything.

## Clean Up

```bash
kubectl delete -f cronjob.yaml
```

## Key Concept Recap
- A **CronJob** = a Job that runs repeatedly on a defined schedule.
- A CronJob's manifest structure nests a full Job template (`jobTemplate`) inside it, which itself nests a Pod template.
- You do NOT need to memorize the entire nested structure — official Kubernetes documentation examples, and careful reading of `kubectl apply` error messages, will guide you to the correct structure.
- Real interview questions test your **understanding and use case knowledge**, not memorization of exact YAML indentation.

## Final Result
Students now understand:
- Basic Cron schedule syntax (5 fields: minute, hour, day, month, day-of-week)
- How to build a complete CronJob YAML with a nested Job and Pod template
- The difference between `volumes` (Pod-level) and `volumeMounts` (container-level)
- How to debug common nested YAML structure errors step by step
- How to verify a CronJob is running on schedule and check its logs

---
**Prerequisite for Chapter 24:** The `hostPath` volume concept introduced here (for the backup demo) leads directly into the next chapter, Storage, which explains volumes, Persistent Volumes, and Persistent Volume Claims in full detail.
