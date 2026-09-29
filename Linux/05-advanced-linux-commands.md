# Chapter 5: Advanced Linux Commands for DevOps

This chapter covers system-level commands: SSH, disk usage, processes, memory, and more. Later parts of the course (user management, package managers) connect with these ideas.

---

## 1. Port numbers and SSH

A computer has many **ports**, like a harbor has many docks where ships arrive. Each service listens on a port. **SSH** (Secure Shell) uses **port 22** by default. To connect to a server with SSH, port 22 must be reachable.

### Public key and private key (easy example)

Imagine **Rahul** wants to talk to **Anjali**. Anjali's phone number is private. She gives it only to people she trusts. Rahul's name is public.

For SSH:

- You create a **key pair**: a **public key** and a **private key**.
- The **public key** stays on the **server** (the machine you want to enter).
- The **private key** stays with **you** (your local computer).
- Only if both match will the server let you in.

When you create a key pair in AWS, the public key goes to the instance and the private key file (`.pem`) is downloaded to your computer. Behind the scenes, a command called `ssh-keygen` creates these keys.

### Connect using SSH from your local machine

1. Locate your `.pem` private key file (for example in Downloads).
2. Protect it so others cannot read it:
   ```bash
   chmod 400 mykey.pem
   ```
3. Connect:
   ```bash
   ssh -i mykey.pem ubuntu@<public-DNS-of-instance>
   ```
   - `-i` means "identity file" (path to your private key).
   - `ubuntu` is the user name.
   - The last part is the public DNS or IP of the EC2 instance.
4. Type `yes` when it asks if you want to continue connecting.

Windows users can also use **PuTTY**. If the instance IP changes after a stop/start (AWS changes public IPs), copy the new SSH command from the **Connect** page of AWS.

---

## 2. System information commands

### `uname`
Shows the platform/OS. `uname` prints `Linux` on Ubuntu, and `Darwin` on macOS.

### `uptime`
Shows how long the system has been running, how many users are logged in, and the load average.

### `date`
Shows the current date and time.

### `who` vs `whoami` (interview question)
- `who`: shows a **list of all users** logged in and when they logged in.
- `whoami`: shows **only your current user name**.

### `which`
Shows the location of a command or program.
```bash
which bash        # /usr/bin/bash
which python
which java
```
Very useful when a developer says "you need Java version 8" and you must find which Java is installed.

### `id`
Shows the **user ID (UID)**, **group ID (GID)** and the groups of the current user.
```bash
id
```
For example the `ubuntu` user usually has UID 1000. The groups may include `ubuntu`, `adm`, `sudo`, `audio`, `video`, `dip`, and more.

---

## 3. What is `sudo`? (the "father" example)

Imagine a house. You (user `ubuntu`) and another person (`jethalal`) are both normal users. But in a house, **the father is above everyone**. He can enter any room and do anything.

- In Linux, the father is the **root user (superuser)**.
- Normal users cannot do everything. For example, you cannot enter `/root` or shut down the machine.
- `sudo` = **s**uper**u**ser **do**. It lets you run a command with superuser permission.
- `sudo` is also a **group**. Users in this group can use `sudo`.

Examples:
```bash
cd /root              # Permission denied
sudo ls /root         # works
shutdown              # fails as normal user
sudo shutdown         # works
```

### `shutdown` and `reboot`
- `sudo shutdown` : the system shuts down (you must start the instance again from AWS console).
- `sudo reboot` : the system restarts. It takes a minute or two, then you connect again.

Practice these on your test machine so you gain confidence. Do not worry about breaking it.

---

## 4. Package managers (installing software)

Windows users install software from Control Panel or by downloading `.exe` files. In Linux, you use a **package manager**.

| Distribution | Package manager |
|--------------|-----------------|
| Ubuntu / Debian | `apt` (also `apt-get`) |
| CentOS / Fedora | `yum` / `dnf` |
| Red Hat | `rpm` (low-level) / `yum` |
| Arch Linux | `pacman` |
| Gentoo | `portage` (`emerge`) |

On Ubuntu:

```bash
sudo apt update              # refresh the list of available packages (and get security updates)
sudo apt install docker.io   # install Docker
sudo apt remove docker.io    # uninstall Docker
which docker                 # find where it is installed (e.g. /usr/bin/docker)
```

If `apt install` says "Package has no installation candidate" or "not available", first run `sudo apt update`. Use `sudo` always, otherwise you see "Permission denied" or "unable to acquire lock".

Tip: press **Ctrl + R** in the terminal to search your command history (reverse search), then press Enter to run the found command.

With package managers you can install Docker, Kubernetes tools, Jenkins, Terraform, etc.

---

## 5. Disk usage

### `df -h`
Shows **disk free** space for all file systems in human-readable form (GB, MB).
```bash
df -h
```
Example meaning: total 7.6 GB, 1.7 GB used, 6 GB available. It also shows special file systems like `tmpfs` (temporary file system in memory).

### `du`
Shows **disk usage** of a folder.
```bash
du .        # usage of current directory
```
Hidden folders start with a dot. `ls -a` shows them.

---

## 6. Processes

### `top`
Shows live processes with CPU and memory use. Press **q** to quit.

### `ps`
Shows processes running now. Useful forms:
```bash
ps            # processes of current shell
ps aux        # all processes with details
ps -ef
```

### `fuser`
Tells which process is using a certain file or file system (less commonly used).

### `kill`
Stops a process using its PID.
```bash
kill <PID>
```
You need permission. Otherwise you get "Operation not permitted". Do not kill random processes.

---

## 7. Memory (RAM)

```bash
free          # shows total, used, free memory
free -h       # human-readable (MB, GB)
```
Example: about 1 GB total RAM on a t2.micro, some used, some free.

### `vmstat`
Shows **virtual memory statistics**: free memory, cache, and more.
```bash
vmstat
vmstat -a     # also shows active and inactive memory
```

---

## 8. `nohup` – run something and keep the output

`nohup` means "no hang up". It runs a command so that it continues even if the terminal closes, and it saves output in a file called `nohup.out`.

```bash
nohup free -h        # output goes into nohup.out
nohup free -h >> nohup.out   # example of appending output
cat nohup.out
```

Run again and the new output is **added below** the previous one, so logs keep collecting. You can then use `head -n 5 nohup.out` for the first 5 lines or `tail -n 5 nohup.out` for the last 5.

---

## Quick summary

- SSH uses a key pair: private key on your machine, public key on the server, port 22.
- `sudo` gives superuser power. `who` vs `whoami`, `id`, `which` are handy checks.
- `apt update` and `apt install` install software on Ubuntu.
- `df -h`, `du`, `top`, `ps`, `free -h`, `vmstat` show system health.
- `kill` stops a process (with permission). `nohup` runs commands and stores output.
