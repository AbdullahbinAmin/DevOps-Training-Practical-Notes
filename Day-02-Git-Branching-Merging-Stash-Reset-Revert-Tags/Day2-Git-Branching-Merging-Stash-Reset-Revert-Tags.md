# Day 2: Git & GitHub — Branching, Merging, Conflicts, Stashing, Reset, Revert & Tags

## Module 1: Branch Management Basic Commands

### Step 1: List Available Branches

```bash
git branch
```

**Why we run this:** To see a list of all available branches in your repository and verify which branch is active (marked with an asterisk `*`).

**Word Breakdown:**

- `git`: Invokes the Git version control tool.
- `branch`: Utility used to list, create, or delete branches.

### Step 2: Create a New Branch

```bash
git branch <branch-name>
```

**Why we run this:** Creates a new branch with the specified name without switching to it immediately.

**Word Breakdown:**

- `git branch`: Branch management command.
- `<branch-name>`: Name of the new branch you want to create (e.g., `branch1`).

### Step 3: Switch Branch

```bash
git checkout <branch-name>
```

**Why we run this:** Switches working directory context to the target branch.

**Word Breakdown:**

- `git checkout`: Changes working directory context to a target branch or commit.
- `<branch-name>`: Name of the branch you want to switch to.

### Step 4: Delete Branch (Safe vs Force Delete)

```bash
git branch -d <branch-name>
git branch -D <branch-name>
```

**Why we run this:** Deletes an existing branch. `-d` safely deletes a branch only if it has been merged, while `-D` force-deletes an unmerged branch.

**Word Breakdown:**

- `-d`: Safe delete flag option.
- `-D`: Force delete flag option.

---

## Module 2: Working with Branches Hands-On

### Step 5: System Environment Preparation

```bash
sudo su
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install git -y
git --version
cd <directory-you-created>
ls
git log --oneline
```

**Why we run this:** Switches to root user, updates Ubuntu package sources, installs Git software, navigates into project repository folder, and checks existing commit logs in single-line format.

### Step 6: Create and Switch to branch1

```bash
git branch
git branch branch1
git branch
git checkout branch1
git branch
ls
git log --oneline
```

**Why we run this:** Creates `branch1`, lists branches to verify its creation, switches working environment to `branch1`, and verifies branch status.

### Step 7: Create File and Commit inside branch1

```bash
nano branchfile
ls
git add .
git commit -m "branch first commit"
git log --oneline
```

**Why we run this:** Creates a file named `branchfile`, writes content inside it, stages changes, and commits it into `branch1` history.

### Step 8: Switch Back to master and Verify Branch Isolation

```bash
git checkout master
git log --oneline
ls
```

**Why we run this:** Demonstrates branch isolation — switching back to master hides `branchfile` because it only exists in `branch1`.

### Step 9: Add Second Commit inside branch1

```bash
git checkout branch1
git branch
nano secondbranchfile
ls
git checkout master
ls
git checkout branch1
git add .
git commit -m "second commit from branch1"
git log --oneline
git checkout master
ls
```

**Why we run this:** Creates a second file in `branch1`, stages and commits it, and verifies that master remains untouched.

---

## Module 3: Merging Branches

### Step 10: Merge branch1 into master

> **Note:** You cannot merge branches from different repositories directly. We use pull mechanisms for cross-repo operations. Within the same repository, we use `git merge`.

```bash
git branch
ls
git log --oneline
git merge branch1
ls
git log --oneline
git push origin master
```

**Why we run this:** Merges code from `branch1` into master, confirms files and logs from `branch1` are now present in master, and pushes updated master branch to remote GitHub repository.

**Word Breakdown:**

- `git merge`: Integrates history and commits from target branch into current active branch.
- `branch1`: Source branch being merged into active branch (master).
- `git push origin master`: Uploads updated local master branch commits to remote GitHub server (origin).

---

## Module 4: Handling Git Conflicts

### Step 11: Create and Resolve Conflict Scenario

**Concept:** Conflict occurs when the same file has different content in different branches during a merge operation.

```bash
git branch
nano conflictfile        # Write content: "Hello conflict"
git add .
git commit -m "first commit before conflict"

git checkout branch1
nano conflictfile        # Write content: "Hi conflict"
git add .
git commit -m "commit from branch1"

git branch
git checkout master
git merge branch1        # Triggers merge conflict!

vi conflictfile          # Edit file manually to set correct content
git status
git add .
git commit -m "Conflict resolve"
git log --oneline
```

**Why we run this:** Demonstrates how editing the same line in two different branches triggers a merge conflict, and how to edit, stage, and commit the file to resolve it.

---

## Module 5: Git Stashing

### Step 12: Temporarily Save Work in Progress

**Scenario:** Suppose you are implementing a new product feature. Work is in progress and suddenly a customer escalation comes. You must set aside your feature work for a few hours without committing partial code or discarding your changes.

```bash
touch demofile
git add .
git commit -m "Demofile"
git branch

vi demofile               # Add first code changes
git stash                 # Temporarily stores uncommitted modified files

git stash list            # View list of stashed items
vi demofile               # Add second code changes
git stash                 # Save second stash state

git stash list
git stash apply stash@{1} # Apply specific stash index from list
cat demofile
git add .
git commit -m "first code done"

git stash apply stash@{0} # Apply latest stash index
vi demofile
git add .
git commit -m "second code done"

git stash clear           # Clears all stashed entries from storage stack
```

**Why we run this:** Saves uncommitted modifications into a temporary stack, lists stored stashes, applies specific stash snapshots back to working directory, and clears stash memory stack.

**Word Breakdown:**

- `stash`: Stores uncommitted file changes in temporary memory.
- `stash list`: Displays saved stash index references (`stash@{0}`, `stash@{1}`).
- `stash apply`: Applies stashed changes back into active working directory.
- `stash clear`: Deletes all items from stash storage stack.

---

## Module 6: Git Reset (Before Commit)

### Step 13: Reset Staged Changes & Working Directory

```bash
git branch
nano testfile             # Add test content
git add .
git status
git reset .               # Unstages files from staging index
git status

git add .
git status
git reset --hard          # Wipes out changes from both staging index and working directory
git status
```

**Why we run this:** `git reset .` unstages files without losing local edits. `git reset --hard` completely wipes out all local uncommitted edits and restores workspace to last committed state.

**Word Breakdown:**

- `reset .`: Unstages changes from staging area back to working directory.
- `--hard`: Destructive flag that matches both index and working directory to target commit.

---

## Module 7: Git Revert (After Commit)

### Step 14: Safely Undo an Existing Commit

**Concept:** Revert helps undo an existing commit without deleting history. It creates a new commit containing inverted file changes, moving version history forward while restoring files to their previous state.

```bash
git status
ls
nano myfile               # Add code content
git add .
git commit -m "Code"
git log --oneline
git revert <commit-id>
```

**Why we run this:** Undoes modifications introduced by a specific commit ID by creating an inverse commit, ensuring full historical auditing accuracy.

**Word Breakdown:**

- `git revert`: Generates a new commit that reverses changes introduced by target commit SHA.
- `<commit-id>`: SHA hash identifier of target commit to undo.

---

## Module 8: Cleaning Untracked Files & Git Tags

### Step 15: Clean Untracked Files

```bash
git clean -n              # Dry run: shows which untracked files will be removed
git clean -f              # Forcefully deletes untracked files completely
```

**Why we run this:** Removes untracked temporary files from working directory safely.

**Word Breakdown:**

- `-n`: Perform dry run without deleting files.
- `-f`: Force deletion of untracked files.

### Step 16: Git Tag Operations

**Concept:** Tag operations allow assigning meaningful release names (such as `v1.0`, `v2.0`) to specific version commits in a repository.

```bash
git tag -a <tagname> -m "message" <commit-id>    # Applies tag to a specific commit
git tag                                          # Lists all created tags
git show <tagname>                               # Displays specific commit content linked to tag
git tag -d <tagname>                             # Deletes a tag
```

**Why we run this:** Labels specific release milestones in Git history for easy reference and deployment tracking.

**Word Breakdown:**

- `tag -a`: Creates an annotated tag.
- `-m`: Message flag describing release tag.
- `show`: Displays commit details associated with specified tag.
- `-d`: Deletes specified tag name.
