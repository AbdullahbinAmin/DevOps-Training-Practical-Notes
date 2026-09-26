# Chapter 7 — Docker Volumes and Storage

## Objective
Understand why data inside a container is lost when the container is removed, and learn how to use Docker Volumes (named volumes and bind-mount/path volumes) to make data persist permanently, independent of the container's lifecycle.

## Prerequisites
* OS: Ubuntu EC2 instance with Docker installed
* Required software: Docker
* Required setup: Continuing from Chapter 6's two-tier Flask + MySQL setup (a running `mysql` container is assumed, or you can recreate one)
* Required knowledge: Chapter 6 (Docker Networking)

## Concept — The Problem: Data Loss on Container Restart/Removal

**The problem demonstrated:**
1. You have a MySQL container with data inside it (e.g., messages submitted via the two-tier Flask app from Chapter 6).
2. If you **restart** the container (`docker restart`), the data usually still exists (it depends — but even restart is risky if MySQL's internal state gets reset).
3. If you **remove** the container (`docker stop` + `docker rm`) and then create a brand-new container from the same image, **all previous data is gone** — because a container's filesystem is deleted along with the container itself.

**Why this happens:**
By default, all data written inside a container lives ONLY inside that container's own writable layer. When the container is deleted, that layer — and all data in it — is deleted too.

**The solution — Docker Volumes:**
A **Volume** maps a folder/path inside the container to a folder/path on the **host machine's** filesystem. Because the data physically lives on the host disk (not just inside the container), it survives even if the container is stopped, removed, or recreated.

## Step 1 — Reproduce the Data-Loss Problem (For Understanding)

1. Assume a running MySQL container from Chapter 6 has some test data in it.
2. Restart the container:
```bash
docker restart <MYSQL_CONTAINER_ID>
```
3. Check the data again through the app or via `mysql` shell — sometimes it survives a simple restart.
4. Now fully remove the container:
```bash
docker ps
docker stop <MYSQL_CONTAINER_ID>
docker rm <MYSQL_CONTAINER_ID>
```
5. Create a brand-new container with the same name and image:
```bash
docker run -d --name mysql --network two-tier -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=devops mysql:latest
```
6. Refresh the app in the browser.
**Expected result:** All previous data is GONE — the database is completely empty again. This confirms the data-loss problem.

## Step 2 — Method 1: Named Volumes

### Step 2a — List Existing Volumes
```bash
docker volume ls
```
**Expected result:** May be empty, or show volumes automatically created by images that declare them internally.

### Step 2b — Create a Named Volume
```bash
docker volume create mysql_data
```
**What this does:** Creates a persistent storage volume named `mysql_data`, managed by Docker.

**Verify:**
```bash
docker volume ls
```
**Expected result:** `mysql_data` appears in the list.

### Step 2c — Inspect Where the Volume Physically Lives on the Host
```bash
docker volume inspect mysql_data
```
**Expected result (relevant field):**
```json
"Mountpoint": "/var/lib/docker/volumes/mysql_data/_data"
```
This confirms the volume is a real folder path that exists on your host machine's filesystem, managed by Docker.

> Note: Accessing this path directly with `cd`/`ls` usually requires root/sudo permissions, since Docker's internal volume storage is owned by the root user.

### Step 2d — Remove the Old MySQL Container (Without the Volume Attached)
```bash
docker ps
docker stop <MYSQL_CONTAINER_ID>
docker rm <MYSQL_CONTAINER_ID>
```

### Step 2e — Run a New MySQL Container WITH the Volume Attached
```bash
docker run -d \
  --name mysql \
  --network two-tier \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=devops \
  -v mysql_data:/var/lib/mysql \
  mysql:latest
```
- `-v mysql_data:/var/lib/mysql` → Mounts (binds) your named volume `mysql_data` to `/var/lib/mysql` inside the container — **this is the exact path where MySQL stores its actual database files internally.**

**How did we know the internal path is `/var/lib/mysql`?**
This is documented as MySQL's standard internal data directory (you can confirm via the official MySQL Docker Hub page or documentation if unsure for other databases).

### Step 2f — Test Data Persistence

1. Access the app and add new data (e.g., submit "Test Dummy Data" via the two-tier Flask app from Chapter 6, or insert directly via MySQL shell).
2. Remove the MySQL container completely:
```bash
docker ps
docker stop <MYSQL_CONTAINER_ID>
docker rm <MYSQL_CONTAINER_ID>
```
3. Confirm the data still physically exists on the host, inside the volume:
```bash
sudo su
cd /var/lib/docker/volumes/mysql_data/_data
ls
exit
```
**Expected result:** MySQL's internal data files are still present on disk, even though the container was deleted.

4. Recreate a fresh MySQL container, reattaching the SAME volume:
```bash
docker run -d \
  --name mysql \
  --network two-tier \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=devops \
  -v mysql_data:/var/lib/mysql \
  mysql:latest
```
5. Restart the Flask app container so it reconnects properly:
```bash
docker restart <FLASK_APP_CONTAINER_ID>
```
6. Refresh the app in your browser and check your data.

**Expected result:** Your previous data ("Test Dummy Data") is STILL THERE, even though the container was fully deleted and recreated — proving the volume successfully persisted the data.

## Step 3 — Method 2: Bind Mounts (Host Path Volumes)

This is an alternative to named volumes — instead of letting Docker manage the storage location, you choose your own folder path on the host.

### Step 3a — Create a Folder on the Host
```bash
cd ~
mkdir volumes
mkdir volumes/mysql
pwd
```
**Expected result:** `pwd` shows something like `/home/ubuntu/volumes/mysql` — note this exact path down.

### Step 3b — Remove the Old Container
```bash
docker stop mysql
docker rm mysql
```

### Step 3c — Run the Container With a Bind-Mount Path Instead of a Named Volume
```bash
docker run -d \
  --name mysql \
  --network two-tier \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=devops \
  -v /home/ubuntu/volumes/mysql:/var/lib/mysql \
  mysql:latest
```
- `-v /home/ubuntu/volumes/mysql:/var/lib/mysql` → Instead of a named volume (`mysql_data`), this directly binds a specific host folder path to the container's data path. Both approaches use the same `-v` flag; the difference is simply whether you give it a **Docker-managed name** or a **direct host path**.

**Verify:**
```bash
cd ~/volumes/mysql
ls
```
**Expected result:** MySQL's internal data files appear directly inside this folder — confirming the bind mount is working and data is now backed up to a location you control directly.

## Concept — Named Volume vs Bind Mount (Path Volume)

| Aspect | Named Volume | Bind Mount (Path Volume) |
|---|---|---|
| Managed by | Docker (`/var/lib/docker/volumes/...`) | You (any folder you choose on the host) |
| Created with | `docker volume create <name>` then `-v <name>:<container_path>` | `-v <host_path>:<container_path>` directly |
| Best for | General persistence without caring about exact host location | When you want direct, easy access to the files from the host |

## Troubleshooting

### Error 1
```text
docker: Error response from daemon: driver failed programming external connectivity...port is already allocated
```
**Reason:** A previous container is still using the same port; this is unrelated to volumes directly but often shows up when recreating containers quickly.

**Fix:** Run `docker ps` to find and stop/remove the conflicting container first.

### Error 2
```text
Permission denied
```
when trying to `cd` into `/var/lib/docker/volumes/...` directly.
**Reason:** Docker's internal volume storage is owned by `root`.

**Fix:** Use `sudo su` (or prefix commands with `sudo`) to access it.

### Error 3
Data still lost after using a volume.
**Reason:** The internal container path in `-v <source>:<container_path>` does not match the actual path the database uses internally (e.g., using `/var/lib/mysqldata` instead of the correct `/var/lib/mysql`).

**Fix:** Always verify the exact internal data directory path from the official image's documentation on Docker Hub.

## Cleanup
```bash
docker stop mysql flaskapp
docker rm mysql flaskapp
docker volume rm mysql_data
rm -rf ~/volumes
```
**What this does:** Removes containers, the named volume, and the bind-mount folder created during this chapter's practice.

## Final Result
* You understand why container data is lost by default when a container is removed.
* You created and used a **named volume** to persist MySQL data across container recreation.
* You created and used a **bind mount (host path volume)** as an alternative method.
* You verified data survives even after fully deleting and recreating the container, as long as the volume/path is reattached.

## What's Next
Chapter 8 introduces Docker Compose — a tool to automate everything you've done manually so far (build, run, network, volume) into a single configuration file and a single command.
