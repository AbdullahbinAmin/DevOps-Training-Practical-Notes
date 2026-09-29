# Chapter 6: Users and Groups Management in Linux

## Why do we manage users and groups?

In a company, many people use the same servers. Each person should have their own login, and each person should only be able to do what their role allows. Linux does this with **users** and **groups**. Later, these users get proper **permissions** on files and folders (Chapter 7).

**Example from the course:** A company has 2 DevOps engineers (`ubuntu`, `jethalal`) and 3 testers (`iyer`, `tappu`, `bhide`). DevOps engineers need one set of permissions and testers need another. You cannot give permissions to hundreds of people one by one, so you make **groups** (for example `devops` and `testers`) and give permissions to the group.

The house example: the server is a house. Each user has their own room. The **root user** is the father who has all permissions.

---

## 1. Important places and commands to check users

| What | Command / file |
|------|----------------|
| Current logged-in user | `whoami` |
| All users who logged in | `who` |
| Your user ID and groups | `id` |
| List of all users | `cat /etc/passwd` |
| List of all groups | `cat /etc/group` |

`/etc/passwd` has one line per user (name, user ID, home folder, shell). At the end of the file you will see users you added.

---

## 2. Adding a user

Only the superuser can add users, so use `sudo`.

```bash
sudo useradd -m jethalal
```

- `useradd` creates the user.
- `-m` creates a **home directory** for that user (`/home/jethalal`). Without `-m`, no home folder is created.

Trying to make a folder inside `/home` yourself with `mkdir` fails with "Permission denied", because only the superuser can add things there.

### Set a password for the user

```bash
sudo passwd jethalal
```
It asks for a new password twice. Now the user can log in with a password.

A normal user cannot change another user's password: you get "Permission denied", because you are not the superuser.

---

## 3. Switching users

```bash
su jethalal        # switch user (asks for jethalal's password)
whoami             # shows jethalal
pwd                # e.g. /home/jethalal
exit               # go back to the previous user
```

`su` means **switch user**. The prompt may look different for a secondary user, because the first user (`ubuntu`) has a special customized prompt.

### User ID (UID)

Run `id` while logged in as a user. Example: `ubuntu` has UID `1000`, and the next user gets `1001`, and so on. Each user also gets a group with the same name (for example group `jethalal`).

You can also see all users with:
```bash
cat /etc/passwd
```

---

## 4. Deleting a user

```bash
sudo userdel jethalal
```
After this, `su jethalal` gives "user does not exist". Adding `-r` also removes the home folder:
```bash
sudo userdel -r jethalal
```

---

## 5. Working with groups

### Create a group
```bash
sudo groupadd devops
sudo groupadd testers
```

### See all groups
```bash
cat /etc/group
```
Whenever a user is created, a group with that user's name is also created (you can see `ubuntu`, `jethalal`, `docker` etc. in the list). The groups you made (`devops`, `testers`) also appear at the end.

### Add a user to a group
```bash
sudo gpasswd -a jethalal devops     # add one user
sudo gpasswd -a ubuntu devops
```
In the `/etc/group` file, the line for `devops` will now list `jethalal` and `ubuntu` as members.

### Add many users at once
If you had 25,000 employees you would not run a command 25,000 times. Use `-M` (capital M) with a comma-separated list:
```bash
sudo gpasswd -M iyer,tappu,bhide testers
```
Check with `cat /etc/group` and you will see the three testers inside the `testers` group.

Important: `-M` **sets** the whole member list (it replaces existing members), so give the full list.

### Delete a group
```bash
sudo groupdel testers
```
The users in that group are **not** deleted. Only the group is removed.

---

## 6. Important note about `id` after adding to a group

If you add the currently logged-in user to a new group and run `id` immediately, the new group may not appear yet. The user must **log out and log in again** (or start a new session) for the new group to show.

---

## Full example flow (like a company setup)

```bash
# add users
sudo useradd -m jethalal
sudo useradd -m iyer
sudo useradd -m tappu
sudo useradd -m bhide

# create groups
sudo groupadd devops
sudo groupadd testers

# add users to groups
sudo gpasswd -a jethalal devops
sudo gpasswd -a ubuntu devops
sudo gpasswd -M iyer,tappu,bhide testers

# verify
cat /etc/passwd
cat /etc/group
```

## Quick summary

- Users live in `/etc/passwd`, groups in `/etc/group`.
- `useradd -m`, `passwd`, `su`, `userdel` manage users.
- `groupadd`, `gpasswd -a` (add one), `gpasswd -M` (set many), `groupdel` manage groups.
- Use `sudo` because only the superuser can do these tasks.
- Groups make it easy to give permissions to many users at once.
