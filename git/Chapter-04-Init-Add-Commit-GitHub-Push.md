# Practical 4 — git init, add, commit, status, log, Creating a GitHub Account, and git push

## Objective
Initialize a local Git repository, make your first commit, create a GitHub account, connect your local repository to a GitHub remote repository, and push your code to it.

## Prerequisites
* OS: Ubuntu (Mumbai instance from Chapter 3)
* Required software: Git installed (Chapter 3)
* Required account: A GitHub account (created in this chapter)
* Required permissions: Root/sudo access on the EC2 instance

## Pre-Check

```bash
git --version
```

**Expected result:** Git version is shown, confirming Git is ready to use.

---

## Directory / File Structure

```text
~/
└── mumbai-git/          (this folder becomes our local Git repository)
    └── myfile1.txt
```

---

## Step 1 — Create a Working Directory and Initialize Git

```bash
mkdir mumbai-git
cd mumbai-git
git init
```

**What this does:**
* `mkdir mumbai-git` → creates a plain folder.
* `git init` → converts this plain folder into a **Git repository**. Internally, this creates a hidden `.git` folder inside it, which is what actually makes Git track this directory. From this point, the three logical areas (Working Directory, Staging Area, Local Repository) exist for this folder.

**Verify:**

```bash
ls -la
```

**Expected result:** You should see a hidden `.git` folder listed.

---

## Step 2 — Create a File and Write Some Content

```bash
cat > myfile1.txt
```

Type some content, for example:

```text
Dil-e-nadaan tujhe hua kya hai
```

Press `Ctrl+D` to save and exit.

**What this does:** Creates a new file `myfile1.txt` in your Working Directory with the content you typed.

---

## Step 3 — Check the Status

```bash
git status
```

**What this does:** Shows the current state of your Working Directory and Staging Area.

**Expected result:** `myfile1.txt` appears in **red**, listed as an **untracked file**. This means Git sees the file exists but is not yet tracking any changes to it — it hasn't been staged.

---

## Step 4 — Stage the File (`git add`)

```bash
git add myfile1.txt
```

Or, to stage everything in the current directory at once:

```bash
git add .
```

**What this does:** Moves the file from the Working Directory into the **Staging Area**.

**Verify:**

```bash
git status
```

**Expected result:** `myfile1.txt` now appears in **green**, meaning it is staged and ready to be committed.

---

## Step 5 — Commit the File

```bash
git commit -m "My first commit from Mumbai"
```

**What this does:**
* `-m` → lets you provide a commit message inline (in quotes).
* This takes everything in the Staging Area and permanently saves it as a snapshot in the **Local Repository**.

**Expected result:** Output confirms 1 file changed, and shows how many lines were inserted.

**Verify status is now clean:**

```bash
git status
```

**Expected result:** `nothing to commit, working tree clean` — confirming your Working Directory is now empty of pending changes (everything has been safely committed).

---

## Step 6 — View Commit History

```bash
git log
```

**What this does:** Shows the full commit history — each commit's unique Commit ID, author, date, and commit message.

**View a compact, one-line-per-commit summary:**

```bash
git log --oneline
```

**Expected result:** A short list, one line per commit, showing shortened commit IDs and messages.

---

## Step 7 — View What a Specific Commit Contains

```bash
git show <COMMIT_ID>
```

* `<COMMIT_ID>` → copy the full or shortened commit hash from `git log`.

**Expected result:** Shows the exact content/changes that were part of that specific commit.

---

## Step 8 — Create a GitHub Account

**Step 1:** Open a browser and go to `https://github.com`.

**Step 2:** Click **Sign up**.

**Step 3:** Enter:
* Username: e.g. `your-chosen-username`
* Email: your real email address
* Password: a secure password

**Step 4:** Complete the human-verification puzzle (drag/slide as instructed).

**Step 5:** Choose the **Free plan** when prompted.

**Step 6:** Verify your email — GitHub sends a verification code to your email inbox; enter this code on GitHub to confirm your account.

**Expected result:** You are logged into your new GitHub account and see the GitHub dashboard.

---

## Step 9 — Create a Remote (Central) Repository on GitHub

**Step 1:** Click **Create repository** (or the **+** icon → **New repository**).

**Step 2:** Enter:
* Repository name: `central-git` (or any name you prefer)
* Description: optional
* Visibility: **Public** (free) or **Private** (may require payment for some older account types — public is fine for practice)

**Step 3:** Click **Create repository**.

**Expected result:** GitHub shows you the new (empty) repository page, along with its HTTPS URL (e.g. `https://github.com/<username>/central-git.git`).

---

## Step 10 — Connect Your Local Repository to the GitHub Remote

Back on your Mumbai EC2 terminal, inside the `mumbai-git` folder:

```bash
git remote add origin <REPO_URL>
```

* `<REPO_URL>` → the HTTPS URL copied from your GitHub repository page (e.g. `https://github.com/<username>/central-git.git`)
* `origin` → the conventional name Git uses to refer to your main remote repository.

**What this does:** Registers the GitHub repository as a remote destination your local repository can push to and pull from.

**Verify:**

```bash
git remote -v
```

**Expected result:** Shows `origin` listed with both fetch and push URLs pointing to your GitHub repo.

---

## Step 11 — Push Your Commit to GitHub

```bash
git push -u origin master
```

* `-u` → sets `origin master` as the default upstream, so future `git push`/`git pull` commands (without arguments) know where to go.
* `origin` → the remote you just added.
* `master` → the branch name (Git's default branch, created automatically when you ran `git init`).

**Expected result:** You will be prompted for your GitHub **username** and **password** (or a Personal Access Token, depending on GitHub's current authentication policy — check GitHub's docs if a plain password is rejected). After authenticating, the output shows the push completing (100%), confirming your commit reached GitHub.

**Verify:**
Refresh your GitHub repository page in the browser.

**Expected result:** `myfile1.txt` and your commit message ("My first commit from Mumbai") are now visible on GitHub, along with your commit history.

---

## Troubleshooting

### Error 1 — `fatal: not a git repository`
**Reason:** You ran a Git command outside a folder that has been initialized with `git init`.
**Fix:** `cd` into your repository folder, or run `git init` if you haven't yet.

### Error 2 — `remote origin already exists`
**Reason:** You tried to add a remote named `origin` when one is already configured.
**Fix:** Check the existing one with `git remote -v`; remove it with `git remote remove origin` and re-add if needed.

### Error 3 — Push fails with authentication error
**Reason:** GitHub may require a Personal Access Token instead of your account password for command-line pushes (policy varies over time).
**Fix:** ⚠️ Verification Required — check GitHub's current documentation for the accepted authentication method for HTTPS pushes at the time you're doing this, and generate a Personal Access Token if required.

---

## Final Result
By the end of this chapter:
* You have a local Git repository (`mumbai-git`) with at least one committed file
* You created a GitHub account and a remote repository (`central-git`)
* Your local repository is connected to GitHub via `origin`, and your commit has been successfully pushed and is visible on GitHub

## What's Next
**Chapter 5** covers **branches** — what they are, why they're used, how to create/switch/delete them, and how to merge a branch back into `master`.
