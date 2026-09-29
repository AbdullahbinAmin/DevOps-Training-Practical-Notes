# Chapter 8: File Transfer Commands

Sometimes you must move files between **your local computer** and a **remote server** (for example an EC2 instance), or between two servers. The normal `cp` command works only inside one system. For two different systems you use **scp** or **rsync**.

Both of these use **SSH** underneath, so you need the same **private key (.pem) file** you use to connect.

---

## 1. Basic words

- **Local**: your own computer (laptop).
- **Remote**: the server far away (EC2 instance).
- **Source**: where the file is now.
- **Destination**: where you want to copy it.

---

## 2. `scp` – Secure Copy

`scp` copies files securely using SSH.

### Syntax

```bash
scp -i <path-to-private-key.pem> <source> <destination>
```

### Example A: Local -> Remote (upload a file)

Suppose on your laptop you have a file `secret.txt` with "This is a secret text". You want it on the Ubuntu server in `/home/ubuntu`.

```bash
scp -i ~/Downloads/linux-for-devops-key.pem secret.txt ubuntu@<ec2-public-dns>:/home/ubuntu
```

Parts of the command:

| Part | Meaning |
|------|---------|
| `scp` | Secure copy command |
| `-i key.pem` | Use this private key to authenticate |
| `secret.txt` | Source file (on local) |
| `ubuntu@<ec2-dns>:/home/ubuntu` | Destination: user@server:path |

The first time it asks "Are you sure you want to continue connecting?" – type `yes`. You will then see the transfer at **100%** progress.

To check, log in to the server and run `ls` and `cat secret.txt`. The content is copied word for word.

### Example B: Remote -> Local (download a folder)

To copy a folder you must use `-r` (recursive):

```bash
scp -i key.pem -r ubuntu@<ec2-public-dns>:/home/ubuntu/linux-for-devops .
```

- `-r` : copy the folder and everything inside.
- The source is on the server: `user@server:path`.
- The destination is `.` which means "the current folder on my local machine".

After it finishes, run `ls` locally and you will see the copied files (zip, tar files, demo files, etc.).

**Remember the direction:** always write `scp source destination`. If the server is the source, write the server part first.

---

## 3. `rsync` – Remote Sync

`rsync` is like `scp`, but smarter. It **synchronizes** two folders. It only transfers the **differences**, so it is faster for repeated transfers. This is why it is very popular in DevOps.

Install if needed:
```bash
sudo apt install rsync
```
It must exist on both machines.

### Syntax with SSH and a key

```bash
rsync -avz -e "ssh -i /path/to/key.pem" <source> <user@server:destination>
```

### The flags

| Flag | Meaning |
|------|---------|
| `-a` | **Archive** mode (keeps permissions, timestamps, and copies folders recursively) |
| `-v` | **Verbose** (shows what is being done) |
| `-z` | **Compress** data while sending |
| `-e` | Tell rsync which remote shell to use, here `ssh -i key.pem` |

### Example

```bash
rsync -avz -e "ssh -i ~/Downloads/linux-for-devops-key.pem" linux-for-devops ubuntu@<ec2-public-dns>:/home/ubuntu
```

It sends the local `linux-for-devops` folder to the server and prints "done" for the files.

### Two-way sync

If the remote has fewer files and local has more, run `rsync` with local as the source. If the local has fewer files, just **swap the source and destination**. Rsync will only copy what is missing or changed.

---

## 4. scp vs rsync

| Point | scp | rsync |
|-------|-----|-------|
| Speed on repeated copies | Copies everything again | Copies only changes |
| Best for | One-time simple copies | Regular syncing/backups |
| Uses SSH | Yes | Yes (with `-e ssh`) |
| Compress while sending | Optional | `-z` |

## Important tips

- Always keep the `.pem` key safe and set `chmod 400` on it.
- If the EC2 public IP/DNS changed (after stop/start), use the new address.
- These commands are commonly asked in interviews. Practice them.

## Quick summary

- Use `scp` for a simple secure copy between machines. Add `-r` for folders.
- Use `rsync -avz -e "ssh -i key.pem"` for smart syncing.
- Both need the source, the destination and SSH authentication.
