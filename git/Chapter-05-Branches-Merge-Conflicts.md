# Practical 5 — Git Branches (Create, Switch, Delete, Merge, and Conflicts)

## Objective
Understand what a branch is and why it's used, create and switch between branches, merge a branch into `master`, and deliberately create and resolve a merge conflict.

## Prerequisites
* OS: Ubuntu (Mumbai or Singapore instance)
* Required software: Git installed
* Required files: An existing local Git repository with at least one commit (from Chapter 4)
* Required account: GitHub account with a remote configured (optional for this chapter, used at the end)

## Pre-Check

```bash
cd ~/mumbai-git
git log --oneline
```

**Expected result:** Shows at least one prior commit, confirming the repository is ready.

---

## Concept — What is a Branch?

Every Git repository has a **default branch**, created automatically the moment you run `git init` — conventionally called `master` (or `main` on newer setups). Think of `master` as your main, stable line of work.

**Why branches matter:** You should generally **never** work directly on `master`, because it may be your live/production/stable code. Instead:
* You create a **new branch** for any new piece of work (a new feature, an experiment, a fix).
* You do all your work — writing code, committing — inside that branch.
* Once the work is complete and tested, you **merge** it back into `master`.

**Key rule:** When a new branch is created, it starts as an **exact copy** of whatever branch it was created from, at that exact moment. After that, the two branches are independent — a commit made in one branch does **not** automatically appear in the other, until you explicitly **merge** them.

**Real-world benefit — Parallel Development:** Multiple developers can work on the same codebase at the same time without interfering with each other. For example, building a WhatsApp-like app: one developer builds the login page on `branch-login`, another builds the contacts screen on `branch-contacts`, another builds the chat screen on `branch-chat`. Each works independently; later, all branches are merged together into one final application.

---

## Directory / File Structure (for this practical)

```text
~/mumbai-git/
├── myfile1.txt        (created in Chapter 4, on master)
├── branch1file.txt     (created on branch-1)
└── secondpart.txt      (also created on branch-1)
```

---

## Step 1 — Check Which Branch You're Currently On

```bash
git branch
```

**What this does:** Lists all branches in your local repository. The branch you're currently on is marked with a `*` (asterisk) next to its name.

**Expected result:** Only `master` is listed (with `*`), since no other branch has been created yet.

---

## Step 2 — Create a New Branch

```bash
git branch branch-1
```

**What this does:** Creates a new branch named `branch-1`. At this point, you have NOT switched to it yet — you are still on `master`.

**Verify:**

```bash
git branch
```

**Expected result:** Both `master` (with `*`) and `branch-1` are listed.

---

## Step 3 — Switch to the New Branch

```bash
git checkout branch-1
```

**What this does:** Switches your active branch to `branch-1`. Any files you create or edit now happen inside this branch's context.

**Verify:**

```bash
git branch
```

**Expected result:** `branch-1` now shows the `*`, confirming you are inside it.

**Shortcut — create and switch in one command:**

```bash
git checkout -b branch-2
```

**What this does:** Creates `branch-2` AND switches to it in a single step.

---

## Step 4 — Confirm Data Was Copied From the Source Branch

```bash
ls
```

**Expected result:** `myfile1.txt` (created back in Chapter 4 on `master`) is visible here too — proof that a new branch starts with an exact copy of the source branch's content at creation time.

---

## Step 5 — Work Inside the New Branch

Switch back to `branch-1` and create a new file there:

```bash
git checkout branch-1
cat > branch1file.txt
```

Type some content, e.g.:

```text
Hello to all
```

Press `Ctrl+D` to save.

```bash
git add .
git commit -m "First commit in branch-1"
```

**Verify this file is NOT visible in `master`:**

```bash
git checkout master
ls
```

**Expected result:** `branch1file.txt` is **not** present here — confirming that changes made in a branch stay isolated to that branch until merged.

**Switch back and confirm it's there again:**

```bash
git checkout branch-1
ls
```

**Expected result:** `branch1file.txt` is visible again.

---

## Step 6 — Merge a Branch Into Master

Once work in `branch-1` is complete and tested, merge it into `master`:

```bash
git checkout master
git merge branch-1
```

**What this does:**
* First, switch to the branch you want to merge **into** (the destination — here, `master`).
* `git merge branch-1` → pulls all the changes made in `branch-1` and copies them into your current branch (`master`).

**Expected result:** Git reports the merge as successful (e.g. "Fast-forward" or a merge commit summary).

**Verify:**

```bash
ls
```

**Expected result:** `branch1file.txt` now appears in `master` too, since the merge copied it over.

---

## Step 7 — Delete a Branch You No Longer Need

Once a branch's work has been merged, you can safely delete it:

```bash
git branch -d branch-1
```

* `-d` → deletes the branch (Git will refuse if the branch has unmerged changes, as a safety check).

**Force delete** (even if not merged — use with caution):

```bash
git branch -D branch-1
```

**Verify:**

```bash
git branch
```

**Expected result:** `branch-1` no longer appears in the list.

---

## Step 8 — Deliberately Create a Merge Conflict (For Learning)

A **merge conflict** happens when two branches have a file with the **same name**, but **different content** inside it, and Git cannot automatically decide which version to keep.

**On `master`:**

```bash
git checkout master
cat > conflictfile.txt
```
Type: `Hello Master Version`
Press `Ctrl+D`, then:
```bash
git add .
git commit -m "Master version of conflict file"
```

**On a new branch:**

```bash
git checkout -b branch-conflict
cat > conflictfile.txt
```
Type: `Hello Branch Version`
Press `Ctrl+D`, then:
```bash
git add .
git commit -m "Branch version of conflict file"
```

**Now attempt the merge:**

```bash
git checkout master
git merge branch-conflict
```

**Expected result:**
```text
CONFLICT (add/add): Merge conflict in conflictfile.txt
Automatic merge failed; fix conflicts and then commit the result.
```

---

## Step 9 — Resolve the Conflict

Open the conflicted file:

```bash
cat > conflictfile.txt
```
(or use `nano conflictfile.txt` to edit properly)

**Expected content inside the file (Git marks both versions):**

```text
<<<<<<< HEAD
Hello Master Version
=======
Hello Branch Version
>>>>>>> branch-conflict
```

**What to do:** Manually edit the file to keep whichever content you want (or combine both), and remove the `<<<<<<<`, `=======`, and `>>>>>>>` marker lines completely. For example, decide the final content should be:

```text
Hello Branch Version
```

Save the file, then:

```bash
git add conflictfile.txt
git commit -m "Resolved merge conflict"
```

**Verify:**

```bash
git log --oneline
```

**Expected result:** A new commit confirms the conflict was resolved and merged successfully.

---

## Troubleshooting

### Error 1 — `error: The branch 'branch-1' is not fully merged`
**Reason:** You tried `git branch -d` on a branch with unmerged commits.
**Fix:** Either merge it first (`git merge branch-1` from the target branch), or use `git branch -D branch-1` to force-delete if you're sure you don't need those changes.

### Error 2 — `CONFLICT (content): Merge conflict in <file>`
**Reason:** The same file has different content in both branches being merged.
**Fix:** Open the file, manually resolve by choosing/combining content, remove the conflict markers, then `git add` and `git commit` to finalize.

### Error 3 — Accidentally editing on `master` directly
**Reason:** Forgot to create a new branch before starting work.
**Fix:** As a habit, always run `git checkout -b <new-branch-name>` before starting any new piece of work.

---

## Final Result
By the end of this chapter you can:
* Create, switch to, and delete branches
* Understand that a new branch is an exact copy of its source branch at creation time, and stays isolated afterward
* Merge a completed branch back into `master`
* Recognize, and manually resolve, a merge conflict

## What's Next
**Chapter 6** covers `git stash` (temporarily shelving work), and the difference between `git reset` and `git revert`.
