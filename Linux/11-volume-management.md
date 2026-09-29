# Chapter 11: Linux Volume Management

This chapter is about **storage**: what blocks/disks/volumes are, how to add extra storage to a server using **AWS EBS**, and how to make the storage usable by **mounting** it.

---

## 1. Basic ideas

### Block, disk, volume

- A **block device** is the same thing as a **disk**, **hard disk** or **volume**. These names are used loosely for the same idea: a place where data is stored.
- On a Windows laptop you see drives like **C:** and **D:**. Each drive is basically a volume on a hard disk.

### AWS EBS (Elastic Block Store)

- **EC2** gives you virtual **servers**.
- **EBS** gives you virtual **volumes (disks)** that you can attach to EC2 servers.
- By default an EC2 instance comes with a root volume of at least **8 GB**. If 8 GB becomes too small, you can attach extra volumes. This is the main idea of this chapter.
- Azure does not have "EBS" by that name, so this lesson is for AWS.

### Demo plan

We will create **three extra volumes**: 10 GB, 12 GB and 14 GB, and attach them to one Ubuntu instance. Then we will use them in different ways (in Chapter 12 also).

---

## 2. Create the EC2 instance

1. Launch a new Ubuntu instance (t2.micro, free tier), name it for example `volume-instance`.
2. Create a new key pair (for example `volumes-key`).
3. Allow SSH traffic in network settings.
4. Storage: the default root volume is **8 GB** (minimum for Ubuntu on AWS; 7 GB gives an error). You can choose more, but 8 GB is fine.
5. Launch the instance.

### Connect through SSH

```bash
chmod 400 volumes-key.pem
ssh -i volumes-key.pem ubuntu@<public-dns>
```

---

## 3. Two essential commands

### `lsblk` – list block devices
```bash
lsblk
```
Shows all disks/volumes attached to the machine, their sizes, and partitions.

Example: `xvda` (8 GB) is the root disk. `xvda1` and others (`xvda14`, `xvda15`, ...) are its **partitions**.

- `xvda` name: the root volume. Its full device path is `/dev/xvda`.
- `/dev` means "device".

### `df -h` – disk free
```bash
df -h
```
Shows the **mounted** file systems, their sizes and where they are mounted. Example: `/dev/root` about 6.8 GB total, mounted on `/`.

**Remember:** `lsblk` shows **all attached** volumes. `df -h` shows only those that are **mounted**.

---

## 4. Attach vs Mount (important interview question)

| Term | Meaning |
|------|---------|
| **Attach** | Connect a volume (disk) to an instance. Now the machine can see the block. Done in AWS console. |
| **Mount** | Bind the volume to a **folder/location** in Linux so you can actually use it (store files). Done inside Linux. |

Think of a USB pen drive: **attaching** = plugging the USB into the device. **Mounting** = making its files accessible in a folder.

A mount point is a directory. For example the root file system is mounted at `/`.

---

## 5. Create the three volumes on AWS

1. In the EC2 console side bar, click **Elastic Block Store -> Volumes**.
2. Click **Create volume**:
   - Type: **General Purpose SSD**.
   - Size: **10 GB**.
   - **Availability Zone**: **must be the same as your instance's zone**, for example `us-west-2b`. (A volume in a different zone cannot be attached.)
   - Snapshot: choose "Don't create volume from a snapshot" (a snapshot is a backup of a volume; we want a fresh volume).
3. Create two more volumes: **12 GB** and **14 GB** with the same settings.
4. In the list you will see 8 GB (root), 10 GB, 12 GB and 14 GB. The three new ones should be in the **Available** state.

---

## 6. Attach the volumes to the instance

1. Select the 10 GB volume -> **Actions -> Attach volume**.
2. Choose your instance.
3. Give the **device name**.

### Device name rules

- Root volumes use names like `/dev/xvda` or `/dev/sda1`.
- For **extra data volumes**, AWS suggests names from **`/dev/sdf` to `/dev/sdp`**. Start from **f**.
- If you choose `sdb` (which is already in use by the root device), you get an error: "attachment point is already in use".

Use:

| Volume | Device name |
|--------|-------------|
| 10 GB | `/dev/sdf` |
| 12 GB | `/dev/sdg` |
| 14 GB | `/dev/sdh` |

On Linux these appear inside the instance as **`xvdf`, `xvdg`, `xvdh`** (the `sd` name becomes `xvd`).

After attaching, the volume state becomes **In use**.

### Verify inside the instance

```bash
lsblk
```
You will now see `xvda` (8 GB), `xvdf` (10 GB), `xvdg` (12 GB) and `xvdh` (14 GB).

At this point the volumes are **attached but not mounted**. `df -h` will not show them yet.

### Why three volumes?

Because the following chapter uses them to explain LVM: some volumes are combined into a group and one is used directly as a disk.

---

## 7. Preparing and mounting a disk

You must use the **root user** for storage commands. Either put `sudo` before each command or switch to root:

```bash
sudo su -
```

### Step 1: Format the disk (create a file system)

A new disk has no file system. **Formatting** creates a file system on it (an empty structure to store files).

```bash
sudo mkfs.ext4 /dev/xvdh
```
(`mkfs.xfs /dev/xvdh` is another file system type option.)

If the disk already has some previous LVM signature, the tool asks whether to proceed. Answer `y` if you are sure.

### Step 2: Create a mount point (a folder)

```bash
sudo mkdir /mnt/twsdisk-mount
```
`/mnt` is the usual place for extra mounts.

### Step 3: Mount

```bash
sudo mount /dev/xvdh /mnt/twsdisk-mount
```
Format: `mount <source-device> <destination-folder>`.

### Step 4: Check

```bash
df -h
```
You should see `/dev/xvdh` mounted on `/mnt/twsdisk-mount` with its size.

### Step 5: Use it

```bash
cd /mnt/twsdisk-mount
sudo touch file.txt
```
Anything you save here is stored on that extra volume, so your instance's storage will not run out easily.

### Unmount

```bash
sudo umount /mnt/twsdisk-mount
```
After unmount the folder is empty/unusable for that disk; files inside are not accessible until you mount it again. Mount again with the same `mount` command and the files are back. So you can reuse volumes whenever you want.

---

## 8. Mounting a disk that has no LVM (direct mount) vs LVM

You can mount a disk **directly** (as above) or use **LVM** (Logical Volume Manager) first to combine disks and create flexible partitions. LVM is explained fully in Chapter 12. Understand both ways:

| Way | Steps |
|-----|-------|
| Direct disk | Format the disk (`mkfs`), then mount it |
| Using LVM | Create physical volume -> volume group -> logical volume -> format -> mount |

---

## 9. From physical volume to logical volume (LVM introduction)

To use the volumes with LVM you convert them like this:

1. **Physical Volume (PV)**: a disk (like your 10 GB and 12 GB volumes) made ready for LVM.
2. **Volume Group (VG)**: a pool made by combining physical volumes. For example 10 GB + 12 GB = about 22 GB.
3. **Logical Volume (LV)**: a slice cut from the volume group. You can grow or shrink it as needed.

Example (numbers from the course): if the three volumes were combined, the total would be 10 + 12 + 14 = 36 GB. From the volume group, you can cut smaller slices and give them names, like **C drive** and **D drive** on Windows.

Commands (run as root):

```bash
lvm                                       # opens the LVM prompt
pvcreate /dev/xvdf /dev/xvdg /dev/xvdh    # make physical volumes
vgcreate tws-vg /dev/xvdf /dev/xvdg       # make a volume group from two of them
lvcreate -L 10G -n tws-lv tws-vg          # make a 10 GB logical volume
```

Show information:
```bash
pvdisplay      # info about physical volumes
vgdisplay      # info about the volume group
lvdisplay      # info about logical volumes
```
`vgdisplay` shows **VG Size** (about 22 GB), **PE size**, how many PVs it has, and **Free** space.

Note: `pvcreate`, `vgcreate`, `lvcreate` and `lvdisplay` etc. work both inside the `lvm` prompt and directly in the normal shell.

---

## 10. Format and mount a logical volume

The logical volume path looks like `/dev/<vg-name>/<lv-name>`, for example `/dev/tws-vg/tws-lv`. It also appears under `/dev/mapper/`.

```bash
sudo mkfs.ext4 /dev/tws-vg/tws-lv
sudo mkdir /mnt/tws-lv-mount
sudo mount /dev/tws-vg/tws-lv /mnt/tws-lv-mount
df -h
```
`df -h` shows `/dev/mapper/tws--vg-tws--lv` mounted on `/mnt/tws-lv-mount`.

Test it:
```bash
cd /mnt/tws-lv-mount
sudo mkdir devops
sudo vi hello.txt      # write some text
```
Unmount and mount again to see that the data stays.

---

## 11. Quick summary

- Block = disk = volume. **EBS** gives extra volumes for EC2.
- The volume and the instance must be in the **same Availability Zone**.
- Extra device names start from `/dev/sdf` (seen in Linux as `xvdf`...).
- `lsblk` lists all attached volumes; `df -h` lists only mounted ones.
- **Attach** = connect to the instance. **Mount** = bind to a folder so it can be used.
- To use a fresh disk: **format (`mkfs`) -> create folder -> `mount`**.
- Use `umount` to detach it from the folder.
