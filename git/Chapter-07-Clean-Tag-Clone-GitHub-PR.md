# Practical 7 — git clean, git tag, git clone, and Merging via GitHub Pull Requests

## Objective
Learn to bulk-delete untracked files with `git clean`, label important commits with `git tag`, clone a full remote repository to create a fresh local copy, and perform a branch merge visually using GitHub's Pull Request feature.

## Prerequisites
* OS: Ubuntu (either instance)
* Required software: Git installed
* Required account: GitHub account with a remote repository (from Chapter 4)
* Required files: An existing local Git repository connected to a GitHub remote

## Pre-Check

```bash
cd ~/mumbai-git
git remote -v
```

**Expected result:** Shows `origin` pointing to your GitHub repository URL, confirming the remote connection from Chapter 4 is intact.

---

## Part A — git clean (Bulk-Deleting Untracked Files)

### Concept
Sometimes you accumulate several files that were created but never staged or committed — clutter you no longer need. Deleting them one by one is tedious. `git clean` removes all untracked files in one go.

### Step 1 — Create Some Untracked Test Files

```bash
touch file1.txt file2.txt file3.txt
git status
```

**Expected result:** All three files appear in red, listed as untracked.

### Step 2 — Preview What Would Be Deleted (Dry Run)

```bash
git clean -n
```

* `-n` → "dry run" — shows what WOULD be deleted, without actually deleting anything.

**Expected result:** Lists `file1.txt`, `file2.txt`, `file3.txt` as candidates for removal.

### Step 3 — Actually Delete the Untracked Files

```bash
git clean -f
```

* `-f` → "force" — required by Git as a safety confirmation before actually deleting files.

**Expected result:** Output confirms each file was removed (e.g. `Removing file1.txt`).

**Verify:**

```bash
git status
```

**Expected result:** Working tree is clean — no untracked files remain.

> ⚠️ **Warning:** `git clean -f` **permanently deletes** files — this does not go through any recycle bin/trash. Use `-n` first to preview, always.

> **To also remove untracked directories**, add `-d`: `git clean -fd`.

---

## Part B — git tag (Labeling Important Commits)

### Concept
A commit ID is a long, hard-to-remember hash (e.g. `a1b2c3d4...`). If a particular commit is especially important (e.g. a release, or a milestone), you can give it a memorable, human-readable label called a **tag** — similar to labeling spice containers in a kitchen instead of guessing by appearance.

### Step 1 — Make a Commit to Tag

```bash
echo "Important milestone code" > milestone.txt
git add .
git commit -m "Very important commit"
```

### Step 2 — Create a Tag Pointing to This Commit

```bash
git tag -a important -m "This is a very important commit"
```

* `-a important` → the tag's name (`important`).
* `-m "..."` → a message describing the tag.

**By default, this tags the most recent (HEAD) commit.** To tag a specific older commit, append its commit ID at the end:

```bash
git log --oneline
git tag -a important -m "This is a very important commit" <COMMIT_ID>
```

### Step 3 — List All Tags

```bash
git tag
```

**Expected result:** Shows `important` in the list.

### Step 4 — View Details of a Tagged Commit

```bash
git show important
```

**Expected result:** Shows full commit details (author, date, message, content) — exactly as if you had looked it up by commit ID, but referenced by the easier-to-remember tag name instead.

### Step 5 — Delete a Tag

```bash
git tag -d important
```

**Verify:**

```bash
git tag
```

**Expected result:** `important` no longer appears.

---

## Part C — git clone (Getting a Full Copy of a Remote Repository)

### Concept
`git clone` is different from `git remote add` + `git pull`. When you **clone** a repository:
* Git automatically creates a new local folder, named after the remote repository.
* That folder is automatically initialized as a Git repository (you don't need to run `git init` yourself).
* All files and the full commit history from the remote are copied down immediately.

This is the standard way to start working on a project that already exists on GitHub (rather than starting from scratch with `git init`).

### Step 1 — Get the Repository's Clone URL

On GitHub, go to your repository page, click the green **Code** button, and copy the HTTPS URL (e.g. `https://github.com/<username>/central-git.git`).

### Step 2 — Clone It

From your home directory (or anywhere you want the copy):

```bash
git clone https://github.com/<username>/central-git.git
```

**Expected result:** Output shows `Cloning into 'central-git'...` followed by a success message.

**Verify:**

```bash
ls
cd central-git
ls
```

**Expected result:** A new folder named `central-git` (matching the repository's name) exists, and it contains all the files that were present on GitHub.

**Verify it's already a Git repository (no `git init` needed):**

```bash
git status
git remote -v
```

**Expected result:** `git status` works without error, and `git remote -v` already shows `origin` pointing to the GitHub URL automatically — this was configured for you by `git clone`.

---

## Part D — Merging via GitHub's Web Interface (Pull Requests)

Everything you've done so far via command line (`git branch`, `git merge`) can also be done directly on GitHub's website — useful for team collaboration and code review.

### Step 1 — Create a New Branch on GitHub

On your repository's GitHub page, click the branch dropdown (usually shows `master`), type a new branch name (e.g. `branch-1`), and select **Create branch: branch-1 from 'master'**.

**Expected result:** A new branch appears, containing an exact copy of everything from `master` at that moment (same rule as command-line branching).

### Step 2 — Add a New File on That Branch (via GitHub UI)

While on `branch-1`, click **Add file → Create new file**, name it (e.g. `newfile.md`), add some content, and click **Commit changes** — making sure "Commit directly to the `branch-1` branch" is selected.

### Step 3 — Open a Pull Request

Go back to the repository's main page. GitHub usually shows a banner suggesting **Compare & pull request** for your recently pushed branch. Click it (or go to the **Pull requests** tab → **New pull request**).

**Step 4:** Confirm the merge direction shown — e.g. merging `branch-1` **into** `master`.

**Step 5:** GitHub automatically checks for conflicts and shows a message like "This branch has no conflicts with the base branch" if it's safe to merge — this check is the "Pull Request" mechanism doing its job before any actual merge happens.

**Step 6:** Click **Create pull request**, then on the next screen click **Merge pull request**, and confirm with **Confirm merge**.

**Expected result:** GitHub shows "Pull request successfully merged and closed." The content from `branch-1` is now part of `master`.

**Step 7 (Optional cleanup):** GitHub offers a **Delete branch** button after a successful merge — click it to remove `branch-1` since its work is now safely merged into `master`.

### Step 8 — Verify From the Command Line

Back on your terminal (in a repository connected to this same remote):

```bash
git checkout master
git pull origin master
ls
```

**Expected result:** The file added via GitHub's UI on `branch-1` is now present locally on `master` too, confirming the GitHub-side merge synced down correctly.

---

## Troubleshooting

### Error 1 — `git clean -f` deleted a file I still needed
**Reason:** `git clean` is permanent and does not go to trash.
**Fix:** ⚠️ No guaranteed recovery — always run `git clean -n` (dry run) first to review what will be deleted before adding `-f`.

### Error 2 — `fatal: tag 'important' already exists`
**Reason:** You tried to create a tag with a name that's already in use.
**Fix:** Delete the existing tag first (`git tag -d important`) or choose a different tag name.

### Error 3 — GitHub shows a conflict when trying to merge a Pull Request
**Reason:** The same file has different content on both branches, similar to a command-line merge conflict.
**Fix:** GitHub provides a **Resolve conflicts** button in the Pull Request UI — click it, manually edit the conflict markers shown (same `<<<<<<<`, `=======`, `>>>>>>>` format), then mark as resolved and complete the merge.

### Error 4 — `git clone` fails with a permission or URL error
**Reason:** Wrong URL copied, or the repository is private and you're not authenticated.
**Fix:** Double-check the exact URL from GitHub's green **Code** button, and ensure you're logged in / have access if the repository is private.

---

## Final Result
By the end of this chapter you can:
* Bulk-remove untracked files safely with `git clean -n` (preview) and `git clean -f` (delete)
* Label important commits with memorable tags using `git tag`
* Clone an existing GitHub repository to instantly get a full working local copy with history
* Create branches, add files, and merge branches entirely through GitHub's web interface using Pull Requests, and confirm the results locally

## Course Wrap-Up
Across these 7 chapters, you have covered: why version control exists, Git's internal architecture, installing Git, the core commit workflow (`init`, `add`, `commit`, `status`, `log`), connecting to GitHub and pushing code, branching and merging (including conflict resolution), stashing, the reset-vs-revert distinction, and advanced housekeeping (`clean`, `tag`, `clone`, GitHub Pull Requests). This covers the practical Git skillset needed for day-to-day DevOps and development work.
