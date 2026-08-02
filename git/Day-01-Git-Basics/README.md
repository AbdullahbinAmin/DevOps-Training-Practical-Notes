# Day 1: Complete Git & GitHub Hands-On Practical Guide

## Prerequisites & Setup

**Create 2 Linux EC2 Instances on AWS:**

- **Instance 1:** Mumbai Region
- **Instance 2:** Singapore Region

**Security Group Settings:** Allow SSH and HTTP traffic from Anywhere (`0.0.0.0/0`).

---

## Part 1: Initial System Setup (Run on BOTH Instances)

### Step 1: Switch to Superuser (Root)

```bash
sudo su
```

**Why we run this:** To get full administrative (root) permissions so we can install software smoothly.

**Word Breakdown:**

- `sudo`: SuperUser DO — Run command with admin privileges.
- `su`: Switch User — Switches the current user to the superuser (root).

### Step 2: Update Package List

```bash
sudo apt-get update
```

**Why we run this:** Updates the local index of available software packages from server repositories.

**Word Breakdown:**

- `sudo`: Admin privilege execution.
- `apt-get`: Advanced Package Tool for Debian/Ubuntu Linux.
- `update`: Refreshes package lists without upgrading installed programs.

### Step 3: Upgrade Packages & Install Git

```bash
sudo apt-get upgrade -y && sudo apt-get install git -y
```

**Why we run this:** Upgrades installed software to the newest versions and installs Git version control software.

**Word Breakdown:**

- `sudo apt-get upgrade`: Upgrades installed packages.
- `-y`: Automatically answers "yes" to installation prompts.
- `&&`: Runs the second command only if the first command succeeds.
- `install git`: Downloads and installs the Git tool.

### Step 4: Verify Git Installation

```bash
git --version
```

**Why we run this:** Checks if Git is installed correctly and displays its installed version.

**Word Breakdown:**

- `git`: Invokes the Git tool.
- `--version`: Flag requesting version information.

### Step 5: Configure Username

```bash
git config --global user.name "Your Name"
```

**Why we run this:** Connects your commits to your identity in Git history.

**Word Breakdown:**

- `git`: Invokes Git.
- `config`: Modifies configuration settings.
- `--global`: Applies setting across all Git repositories on this machine.
- `user.name`: Variable storing author name.
- `"Your Name"`: Value assigned to the variable.

### Step 6: Configure Email

```bash
git config --global user.email "your-email@example.com"
```

**Why we run this:** Attaches your email address to your commits so GitHub can match them to your user profile.

**Word Breakdown:**

- `git config --global`: Sets global system settings.
- `user.email`: Variable storing author email.
- `"your-email@example.com"`: Value assigned to email.

### Step 7: Verify Configuration Settings

```bash
git config --list
```

**Why we run this:** Displays active global configuration settings to confirm email and username were saved correctly.

**Word Breakdown:**

- `git config`: Configuration command.
- `--list`: Options list output flag.

---

## Part 2: Mumbai Region Operations

### Step 1: Create Directory and Go Inside

```bash
mkdir my-repo && cd my-repo
```

**Why we run this:** Creates a dedicated project workspace directory and navigates into it.

**Word Breakdown:**

- `mkdir`: Make Directory.
- `my-repo`: Directory name.
- `cd`: Change Directory.

### Step 2: Initialize Git Repository

```bash
git init
```

**Why we run this:** Converts the current directory into a tracked Git repository by creating a hidden `.git` folder.

**Word Breakdown:**

- `git`: Git command line tool.
- `init`: Short for initialize.

### Step 3: Create Sample File & Add Content

```bash
touch myfile && echo "Hello from Mumbai" > myfile
```

**Why we run this:** Creates a file named `myfile` and populates it with test text.

**Word Breakdown:**

- `touch`: Creates an empty file.
- `echo`: Prints text.
- `>`: Redirects printed text into `myfile`.

### Step 4: Check Working Directory Status

```bash
git status
```

**Why we run this:** Checks which files are modified, untracked, or staged for commit.

**Word Breakdown:**

- `git status`: Displays the repository's current state.

### Step 5: Stage Changes

```bash
git add .
```

**Why we run this:** Stages all new or modified files in the working directory so they are ready to be committed.

**Word Breakdown:**

- `git add`: Adds files to the staging index.
- `.`: Selects all files in current working folder.

### Step 6: Commit Staged Changes

```bash
git commit -m "1st commit from mumbai"
```

**Why we run this:** Saves staged changes into repository version history with a descriptive message.

**Word Breakdown:**

- `git commit`: Saves staged snapshot to history.
- `-m`: Message flag.
- `"1st commit from mumbai"`: Log description string.

### Step 7: View Commit Logs & Details

```bash
git log
git show <commit-id>
```

**Why we run this:** `git log` displays commit history; `git show <commit-id>` inspects precise line-by-line changes made in a specific commit.

**Word Breakdown:**

- `log`: Shows commit records (hash, author, date, message).
- `show`: Details exact file modifications.
- `<commit-id>`: Unique commit hash identifier.

### Step 8: Link Remote GitHub Repository

```bash
git remote add origin <central-git-repo-url>
```

**Why we run this:** Connects local repository to remote GitHub repository target using reference alias `origin`.

**Word Breakdown:**

- `git remote add`: Links local repository to remote location.
- `origin`: Default alias name assigned to remote server URL.
- `<central-git-repo-url>`: HTTPS/SSH target web address.

### Step 9: Push Local Commits to GitHub

```bash
git push -u origin master
```

**Why we run this:** Uploads local commits to remote GitHub master branch and sets up default upstream tracking.

**Word Breakdown:**

- `git push`: Sends local commits upstream to remote server.
- `-u`: Sets default upstream branch.
- `origin`: Target remote alias.
- `master`: Destination branch name.

> **Note:** Authenticate using your GitHub Username and Personal Access Token (PAT) when prompted.

---

## Part 3: Singapore Region Operations

### Step 1: Navigate to Project Directory & Initialize

```bash
mkdir singapore-repo && cd singapore-repo
git init
```

**Why we run this:** Prepares local working environment on Singapore instance.

### Step 2: Connect Remote GitHub Repository

```bash
git remote add origin <github-repo-url>
```

**Why we run this:** Connects Singapore instance to central GitHub repository.

### Step 3: Pull Central Code Base

```bash
git pull -u origin master
```

**Why we run this:** Downloads and merges latest remote commits from GitHub master branch into local workspace.

**Word Breakdown:**

- `git pull`: Fetches and integrates changes from remote repository into current working branch.

### Step 4: Verify Repository History

```bash
git log
git show <commit-id>
```

**Why we run this:** Verifies commits created in Mumbai are visible on Singapore machine.

### Step 5: Edit Files & Add Code Updates

```bash
echo "Singapore update content" >> myfile
```

**Why we run this:** Simulates code changes inside existing file.

### Step 6: Stage, Commit, and Push Changes

```bash
git status
git add .
git commit -m "Singapore update 1"
git status
git log
git push -u origin master
```

**Why we run this:** Packages new local updates, saves commit record, and syncs changes back to shared GitHub repository.

---

## Part 4: Ignoring Files using .gitignore

### Step 1: Create .gitignore File

```bash
vi .gitignore
```

Add rules specifying file patterns to ignore:

```plaintext
*.css
*.java
```

**Why we run this:** Instructs Git to ignore specific file extensions (e.g., `.css`, `.java`) so temporary or generated build files are not tracked.

**Word Breakdown:**

- `vi`: Text editor tool.
- `.gitignore`: Special configuration file processed by Git.
- `*.css`: Wildcard matching all files ending in `.css`.

### Step 2: Commit .gitignore File

```bash
git add .gitignore
git commit -m "latest update exclude css"
git status
```

**Why we run this:** Saves ignore rules into Git repository history.

### Step 3: Test .gitignore Behavior

```bash
touch index.css App.java notes.txt
ls
git status
```

**Why we run this:** Demonstrates that `index.css` and `App.java` are ignored by `git status`, while `notes.txt` shows as an untracked file.

**Word Breakdown:**

- `touch`: Creates files.
- `ls`: Lists directory contents.

### Step 4: Add and Commit Untracked Text File

```bash
git add .
git commit -m "my text files only"
```

**Why we run this:** Confirms Git stages only non-ignored files (`notes.txt`).

### Step 5: Verify Compact History Log

```bash
git log --oneline
```

**Why we run this:** Displays clean, single-line commit summary listing commit hashes and messages.

**Word Breakdown:**

- `--oneline`: Formats output log into single-line entries for fast scanning.
