# Practical 6 — git stash, and the Difference Between git reset and git revert

## Objective
Learn how to temporarily shelve uncommitted work using `git stash` when an urgent task interrupts your current work, and understand the critical difference between `git reset` (undoing before a commit) and `git revert` (safely undoing after a commit).

## Prerequisites
* OS: Ubuntu (either instance)
* Required software: Git installed
* Required files: An existing local Git repository (from previous chapters)

## Pre-Check

```bash
cd ~/mumbai-git
git status
```

**Expected result:** Working tree is clean (no pending changes), confirming a fresh starting point.

---

## Part A — git stash

### Concept — Why Do We Need Stash?

Imagine you're in the middle of writing code — you've made changes in your Working Directory but haven't staged or committed them yet. Suddenly, someone tells you: "Stop what you're doing, fix this other urgent thing first." You don't want to lose your half-finished work, but you also can't commit incomplete code. `git stash` solves this: it temporarily "shelves" your uncommitted changes, giving you a clean Working Directory to handle the urgent task, and lets you bring your original work back later exactly as it was.

> **Important:** Stash only works on changes that are **already tracked** (i.e. modified files that Git already knows about, or staged changes) — not on brand-new untracked files, unless you explicitly include them.

---

### Step 1 — Make Some Uncommitted Changes

```bash
cd ~/mumbai-git
git checkout master
echo "First code, in progress..." > demofile.txt
git add .
git commit -m "Base commit for demo file"
echo "Some more work in progress, not ready to commit" >> demofile.txt
```

**Verify:**

```bash
git status
```

**Expected result:** `demofile.txt` shows as modified (not yet staged/committed).

---

### Step 2 — Stash the Changes

```bash
git stash
```

**What this does:** Saves your uncommitted changes into a temporary storage area and reverts your Working Directory back to the last commit — so it looks clean again.

**Verify:**

```bash
git status
cat demofile.txt
```

**Expected result:** `git status` shows a clean working tree, and `demofile.txt` shows only the originally committed content — your in-progress line is gone from view (but not lost).

---

### Step 3 — Do the Urgent Work

At this point, your Working Directory is clean. You can now switch tasks, edit other files, make a separate commit, etc., without your unfinished work getting in the way.

```bash
echo "Urgent fix content" > urgentfile.txt
git add .
git commit -m "Urgent fix committed"
```

---

### Step 4 — View What's in the Stash

```bash
git stash list
```

**Expected result:** Shows one or more stashed entries, numbered like `stash@{0}`, `stash@{1}`, etc. — the most recent stash is `stash@{0}`.

---

### Step 5 — Bring Your Stashed Work Back

```bash
git stash apply stash@{0}
```

**What this does:** Restores the stashed changes back into your Working Directory, **without removing them from the stash list**.

**Verify:**

```bash
cat demofile.txt
```

**Expected result:** Your in-progress line is back.

Continue your work, then commit as normal:

```bash
git add .
git commit -m "Completed the earlier task after stash"
```

---

### Step 6 — Clear the Stash

Since `apply` doesn't remove the entry from the stash list, clean it up once you're done:

```bash
git stash clear
```

**Verify:**

```bash
git stash list
```

**Expected result:** Empty — no stashed entries remain.

> **Tip:** `git stash pop` is an alternative to `apply` that both restores AND removes that stash entry in one step.

---

## Part B — git reset vs git revert

### Concept — The Key Difference

| | git reset | git revert |
|---|---|---|
| When to use | To undo changes **before** they are committed (still in Staging Area) | To undo changes **after** they have already been committed |
| Effect | Removes the change from the Staging Area (and optionally the Working Directory) | Does NOT delete the old commit — creates a **new** commit that reverses it |
| Safety for shared history | Safe, since nothing was committed/shared yet | Safe even for already-pushed, publicly shared commits |

**Why this distinction matters:** Once a commit is made — and especially once it's pushed to a shared remote repository — **you cannot delete it**. Other people may already see it, and rewriting shared history causes major problems for collaborators. Think of a commit like a bullet fired from a gun: once it's gone, you can't call it back. The safe way to "undo" a bad commit that's already out there is not to erase it, but to make a **new commit that reverses it**.

---

### Step 1 — Using git reset (Before Commit)

Stage a file, then decide you don't want it staged after all:

```bash
echo "Accidental content" > mistake.txt
git add mistake.txt
git status
```

**Expected result:** `mistake.txt` shows in green (staged).

**Unstage it (remove from Staging Area only, keep the file):**

```bash
git reset mistake.txt
```

**Verify:**

```bash
git status
```

**Expected result:** `mistake.txt` is back to being untracked/unstaged — it's still in your Working Directory, just no longer staged.

**Fully remove from both Staging Area AND Working Directory (use carefully — deletes the actual change):**

```bash
git reset --hard mistake.txt
```

> ⚠️ `--hard` discards changes in both the Staging Area and Working Directory for the given path. Use this only when you are certain you want to fully discard the uncommitted change.

---

### Step 2 — Using git revert (After Commit)

Suppose you already committed something incorrect:

```bash
echo "Wrong content committed by mistake" > important.txt
git add .
git commit -m "Bad commit that needs to be undone"
```

**Get the Commit ID to revert:**

```bash
git log --oneline
```

Copy the commit hash of the bad commit (e.g. `a1b2c3d`).

**Revert it:**

```bash
git revert a1b2c3d
```

**What this does:** Opens a commit message editor (or uses a default message) and creates a **brand new commit** that undoes exactly what the bad commit did — the old file's previous content is restored, and the bad commit's changes are effectively cancelled out.

**Confirm/save the revert commit message** (in the editor that opens, save and exit — typically `Esc :wq Enter` if using `vim`), or if prompted directly, confirm the revert.

**Verify:**

```bash
git log --oneline
```

**Expected result:** A NEW commit appears at the top (e.g. "Revert 'Bad commit that needs to be undone'"), while the original bad commit is STILL VISIBLE in history below it — it was never deleted, just cancelled out by the new commit. It's good practice to also add a message like "Please ignore previous commit" for clarity to collaborators.

---

## Troubleshooting

### Error 1 — `git stash apply` shows a conflict
**Reason:** Changes made after stashing conflict with the stashed changes on the same lines.
**Fix:** Resolve the conflict manually in the affected file (same process as a merge conflict — edit, remove markers, `git add`, then continue), then optionally `git stash drop` that entry.

### Error 2 — `git reset --hard` removed work I actually needed
**Reason:** `--hard` permanently discards uncommitted changes with no easy recovery.
**Fix:** ⚠️ Prevention is key — always double check with `git status` and `git diff` before running `--hard`. Recovery after the fact is not guaranteed for uncommitted work.

### Error 3 — `git revert` opens an editor and you're unsure what to do
**Reason:** Git wants a commit message for the revert commit.
**Fix:** If the editor is `vim`, press `Esc`, then type `:wq`, then press `Enter` to save and exit with the default message, or edit the message first if you want to add more context.

---

## Final Result
By the end of this chapter you can:
* Temporarily stash uncommitted work, do other tasks, then safely restore it
* Understand and correctly choose between `git reset` (pre-commit undo) and `git revert` (post-commit, safe undo via a new commit)

## What's Next
**Chapter 7** covers `git tag`, `git clean`, `git clone`, and doing merges via the GitHub web interface (Pull Requests).
