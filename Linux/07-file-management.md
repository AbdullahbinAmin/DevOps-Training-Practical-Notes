# Chapter 7: File Management in Linux

This chapter covers **file permissions**, **umask**, **ownership**, and **compression** (zip, gzip, tar).

---

## 1. Why do files need permissions?

Suppose the `ubuntu` user creates an important file. Should `jethalal` be able to read, change or run it? Only if the owner allows it. Files can hold sensitive data, so permissions are very important.

---

## 2. Reading the output of `ls -l`

```bash
ls -l
```
Example line:

```
drwxrwxr-x 2 ubuntu ubuntu 4096 Dec 15 09:00 cloud
-rw-r--r-- 1 ubuntu ubuntu   16 Dec 15 09:00 myfile.txt
```

The first block of 10 characters is broken into parts:

```
d   rwx   rwx   r-x
|    |     |     |
|    |     |     +-- Other users
|    |     +-------- Group
|    +-------------- User (owner)
+------------------- File type (d = directory, - = file)
```

- The first character: `d` means directory, `-` means a regular file.
- Then three groups of three characters: **User**, **Group**, **Other**.
  - **User (u)**: the owner who created the file.
  - **Group (g)**: a collection of users.
  - **Other (o)**: everyone else who is not the owner and not in the group.

### What r, w, x mean

| Letter | Meaning | Explanation |
|--------|---------|-------------|
| `r` | Read | Open and view the file |
| `w` | Write | Change the content of the file |
| `x` | Execute | Run the file like a program or shell script |
| `-` | No permission | That permission is not given |

A house example: the user, the group and the others are three kinds of people coming to a house with doors. The door lets the family group in, but a stranger (other) cannot enter.

---

## 3. Numeric (octal) permissions

Each permission has a number:

| Permission | Number |
|------------|--------|
| r (read) | 4 |
| w (write) | 2 |
| x (execute) | 1 |
| none | 0 |

Add the numbers for each group:

| Digit | Meaning | Binary idea |
|-------|---------|-------------|
| 7 | rwx (4+2+1) | 111 |
| 6 | rw- (4+2) | 110 |
| 5 | r-x (4+1) | 101 |
| 4 | r-- | 100 |
| 3 | -wx | 011 |
| 2 | -w- | 010 |
| 1 | --x | 001 |
| 0 | --- | 000 |

The three digits are for **User, Group, Other**, in that order.

Examples:

- `775` = rwx for user, rwx for group, r-x for other.
- `777` = everyone can do everything (open for all, not safe).
- `664` = rw- rw- r-- (a common default for files).
- `700` = only the owner has all permissions, nobody else has any.
- `400` = only the owner can read (used for private key files).

### Trick to remember

Think of the binary: for each of the three positions, a `1` means the permission is on and `0` means off. Write 0 to 7 in binary and match them with `r w x`. Practice a few times and it becomes easy.

---

## 4. `chmod` – change permissions

`chmod` means **change mode**.

```bash
chmod 777 cloud          # give everyone read, write, execute
chmod 700 demo.txt       # only the owner can read/write/execute
chmod 400 mykey.pem      # only the owner can read
```

After using `chmod 777` on a folder, `ls -l` shows `drwxrwxrwx` and the folder changes colour (open for all).

### What does execute mean?

Execute means you can **run** the file, like a shell script or program. When you give a file execute permission, `ls` shows it in **green**.

---

## 5. `umask` – default permissions of new files

`umask` decides the **default permission** given to any new file or folder. It is a number (usually 4 digits, such as `0002` or `0022`).

- It works like a mask that **removes** permissions from the maximum default.
- The maximum default is **777 for folders** and **666 for files**. The umask value is subtracted.

| umask | New files | New folders |
|-------|-----------|-------------|
| `0002` | 664 (rw-rw-r--) | 775 (rwxrwxr-x) |
| `0022` | 644 (rw-r--r--) | 755 (rwxr-xr-x) |

Check your value:
```bash
umask
```
On the machine in the course, one system showed `0002` (Ubuntu on AWS) and the Mac showed `0022`. Different machines can have different defaults. The default is stored in shell configuration files such as `.bashrc` or `/etc/profile`.

---

## 6. `chown` – change owner

Every file has an owner (user) and a group.

```bash
sudo chown jethalal demo.txt
```
Now `ls -l` shows `jethalal` as the owner, while the group is still `ubuntu`.

You can change owner and group together:
```bash
sudo chown jethalal:devops demo.txt
```

## 7. `chgrp` – change group

```bash
sudo chgrp devops demo.txt
```
Now the group of `demo.txt` is `devops`. The owner stays as it was.

Interview view: `chown` changes the **owner (user)** and `chgrp` changes the **group** of a file.

---

## 8. Compression: zip, unzip, gzip, gunzip, tar

**Compress** = make files smaller. **Decompress / extract** = make them normal again.

Why compress? If you have many files to send to someone, you put them in one compressed file, send it, and the receiver extracts it.

### zip and unzip

Install first (not present by default):
```bash
sudo apt install zip
```

```bash
zip -r cloud.zip cloud       # -r = recursive (needed for folders)
unzip cloud.zip
```
`unzip` is usually installed together with `zip`. The output shows each file being added. If a file name has a spelling mistake, unzip says "cannot find or open" – check spelling carefully.

### gzip and gunzip

```bash
gzip file.txt        # creates file.txt.gz
gunzip file.txt.gz   # restores file.txt
```
Zip files end with `.zip`; gzip files end with `.gz`.

### tar (very important)

`tar` bundles (archives) files and can compress them using gzip. Files end with `.tar.gz`.

**Create a compressed archive:**
```bash
tar -cvzf cloud.tar.gz cloud
```

**Extract:**
```bash
tar -xvzf cloud.tar.gz
tar -xvzf cloud.tar.gz -C /path/to/folder     # extract into another folder
```

### Meaning of the flags

| Flag | Meaning |
|------|---------|
| `c` | **Create** an archive (compress/bundle) |
| `x` | **Extract** files |
| `v` | **Verbose** (show what is happening on screen) |
| `z` | Use **gzip** compression |
| `f` | The next word is the **file name** |
| `-C` | Extract into this directory |

Order note: the archive file name comes right after `f`, then the folder to compress, so write `tar -cvzf name.tar.gz folder`.

You do not have to memorize flags: run `tar --help` to see them.

So: **tar -c... = compress**, **tar -x... = extract**.

---

## Quick summary

- `ls -l` shows type, permissions (user/group/other), owner, group, size and date.
- r=4, w=2, x=1. Add them per group. `chmod 755 file` sets permissions.
- `umask` sets default permissions for new files/folders.
- `chown` changes the owner, `chgrp` changes the group.
- Use `zip/unzip`, `gzip/gunzip` and `tar -cvzf / tar -xvzf` for compression.
