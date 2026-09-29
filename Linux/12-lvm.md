# Chapter 12: LVM (Logical Volume Manager) in Linux

This chapter goes deeper into **LVM** and shows the main benefit: **dynamic storage management**. That means you can grow a volume when you run out of space, without moving data or stopping your work.

(Chapter 11 showed how to create physical volumes, volume groups and logical volumes, and how to mount them. Here we recap the idea, then focus on **extending** storage and understanding the parts.)

---

## 1. The three building blocks

```
Disk 1 (10 GB)   Disk 2 (12 GB)
     |                |
 Physical Volume  Physical Volume        <- PV
        \            /
        Volume Group (about 22 GB)       <- VG
                |
        Logical Volume(s) (slices)       <- LV
                |
        Format (mkfs) and Mount
```

| Term | Short | Meaning |
|------|-------|---------|
| Physical Volume | PV | A real disk or volume that has been prepared for LVM (`pvcreate`) |
| Volume Group | VG | A pool of storage made by combining one or more PVs (`vgcreate`) |
| Logical Volume | LV | A "partition" carved out of a VG. Its size can be increased or decreased (`lvcreate`) |

**Windows example:** you have a 500 GB computer. You give some part to the C drive and some to the D drive. In LVM, the disk is the pool (VG) and C and D are the logical volumes (LV). When C is full, you can move some free space to it.

**LVM** stands for **Logical Volume Manager**. It manages all these layers.

---

## 2. Commands for each layer

| Task | Create | Show |
|------|--------|------|
| Physical volume | `pvcreate /dev/xvdf /dev/xvdg` | `pvdisplay`, `pvs` |
| Volume group | `vgcreate tws-vg /dev/xvdf /dev/xvdg` | `vgdisplay`, `vgs` |
| Logical volume | `lvcreate -L 10G -n tws-lv tws-vg` | `lvdisplay`, `lvs` |

- `-L 10G` sets the size to 10 GB.
- `-n tws-lv` sets the name.
- `tws-vg` says from which volume group to take the space.

Run these as **root** (`sudo su -`), otherwise LVM warns "Running as a non-root user. Functionality may be unavailable."

`lvm` on its own opens an **LVM prompt**. The same commands (`pvdisplay`, `vgdisplay`, `lvdisplay`, `lvextend`) work **inside** that prompt and also **outside** in the normal shell. There is no difference in result. It is just that the LVM tool (created by Canonical, the makers of Ubuntu) offers the same commands in both places.

---

## 3. Recap: format and mount a logical volume

```bash
mkfs.ext4 /dev/tws-vg/tws-lv          # create a file system
mkdir /mnt/tws-lv-mount               # create a mount folder
mount /dev/tws-vg/tws-lv /mnt/tws-lv-mount
df -h                                  # verify
```
In `df -h` you see the name `/dev/mapper/tws--vg-tws--lv` (a double dash means one dash from the name).

---

## 4. Dynamic storage management: extending a logical volume

### The problem

You created a 10 GB logical volume and installed a big software on it. Now the space is getting full. With LVM you can add more space **while it is mounted and in use**.

### Step 1: Check the current size

```bash
df -h
```
Note the size of the logical volume mount (about 9.8 GB).

### Step 2: Extend the logical volume

Only possible if the **volume group has free space**. Check with `vgdisplay` (see "Free PE / Size").

```bash
lvextend -L +5G /dev/tws-vg/tws-lv
```

- `-L +5G` means "add 5 GB more".
- The path `/dev/<volume-group>/<logical-volume>` tells which LV to extend.
- You do **not** need to mention the volume group separately as an option; the path already contains it. (In the video, first attempt with `tws-vg` typed as an extra word failed with "Physical volume ... not found in volume group". Use the LV path only.)

Output: "Size of logical volume changed from 10.00 GiB to 15.00 GiB. Logical volume successfully resized."

### Step 3: Check with `lsblk`

```bash
lsblk
```
The LV now shows 15 GB, and you can see that the space was taken from the underlying disks. Some space was taken from `xvdf` and some from `xvdg`. LVM handles this automatically.

### Step 4: Make the file system use the new space (important extra step)

Extending the logical volume makes the **block device** bigger, but the **file system** inside (ext4) may still show the old size in `df -h`. You must grow the file system:

```bash
resize2fs /dev/tws-vg/tws-lv
df -h
```
Now `df -h` shows about 15 GB.

Shortcut: use the `-r` option with lvextend, which resizes the file system for you:
```bash
lvextend -r -L +5G /dev/tws-vg/tws-lv
```

For an XFS file system, use `xfs_growfs /mount-point` instead of `resize2fs`.

### Why this is powerful

- No downtime: the volume stays mounted and the data stays safe.
- You can increase storage anytime as long as the volume group has free space.
- If the volume group is full, add a new disk: create a PV, then run `vgextend tws-vg /dev/xvdh`, and then extend the LV again.

---

## 5. Reducing a logical volume (extra note)

Shrinking is riskier than growing, because you can lose data if you shrink the file system below its used size. Basic idea for ext4:

1. Unmount the volume.
2. Check the file system and shrink it (`e2fsck`, `resize2fs` with a smaller size).
3. Reduce the logical volume with `lvreduce`.
4. Mount again.

Always back up first.

---

## 6. Attach vs mount and disk vs LVM – final comparison

| Question | Answer |
|----------|--------|
| What is attach? | Adding a block (volume) to a machine (done in AWS) |
| What is mount? | Binding the block to a folder so it can be used |
| How do you mount a disk without LVM? | `mkfs` on the disk, then `mount /dev/xvdh /mnt/folder` |
| How do you mount an LVM logical volume? | `mkfs` on `/dev/vg/lv`, then `mount /dev/vg/lv /mnt/folder` |
| What is the benefit of LVM? | You can combine disks and resize storage easily |

Example from the course with one direct disk mount:

```bash
mkdir /mnt/tws-disk-mount
mkfs.ext4 /dev/xvdh        # if it warns about existing LVM data, only continue if you are sure
mount /dev/xvdh /mnt/tws-disk-mount
df -h
```
Then `df -h` shows both mounts: the LVM one (`/mnt/tws-lv-mount`) and the direct disk one (`/mnt/tws-disk-mount`).

---

## 7. Useful command list for this topic

| Command | Purpose |
|---------|---------|
| `lsblk` | List all block devices (attached) |
| `df -h` | Show mounted file systems and free space |
| `pvcreate` / `pvdisplay` | Make / show physical volumes |
| `vgcreate` / `vgdisplay` / `vgextend` | Make / show / grow volume groups |
| `lvcreate` / `lvdisplay` / `lvextend` | Make / show / grow logical volumes |
| `mkfs.ext4` | Create a file system |
| `mount` / `umount` | Mount / unmount |
| `resize2fs` | Grow an ext4 file system after extending |

---

## 8. Summary

- LVM lets you group disks into one pool and cut flexible logical volumes from it.
- Flow: **Disks -> PV -> VG -> LV -> format -> mount**.
- The big advantage is **dynamic storage management**: extend volumes anytime with `lvextend`, then grow the file system with `resize2fs` (or use `lvextend -r`).
- Know the difference between **attaching** (AWS side) and **mounting** (Linux side).
- Practice the full flow yourself: create EBS volumes, attach, `pvcreate`, `vgcreate`, `lvcreate`, `mkfs`, `mount`, `lvextend`, `resize2fs`.

## Final tip from the teacher

Try all commands on your own server and share your practice on LinkedIn, tagging the teacher. Practice is the best way to remember Linux.
