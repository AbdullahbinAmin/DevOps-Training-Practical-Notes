# Day 2: Git & GitHub — Feature Branching, Pull Requests & Merge Conflicts

IT companies mein daily work bilkul isi **Branching, Pull Request (PR), and Merge Conflict Resolution** workflow par chalta hai. Production/Main branch par koi direct commit nahi karta, bilkul isi scenario ko follow kiya jata hai.

---

## 🏢 Scenario Brief (Company Standard Workflow)

- **Main/Master Branch Protection:** Company mein master branch live status ko target karti hai. Is par direct code edit ya push blocked hota hai.
- **Feature Branch:** Har new ticket/feature ke liye developer alag branch banata hai (e.g., `feature-login`).
- **Pull Request (PR) & Code Review:** Code complete hone par GitHub par PR create kiya jata hai jahan Team Lead code review karta hai.
- **Merge & Conflict Resolution:** Code approve hone par master mein merge kiya jata hai. Agar do developers same file edit kar lein toh conflict resolve karna hota hai.

---

## 📌 Module 1: Branch Management & Feature Isolation

### Step 1: Check Existing Branches

```bash
git branch
```

**Why we run this:** To list all local branches in your repository and check which branch is currently active (marked with an asterisk `*`).

**Word Breakdown:**

- `git`: Invokes the Git version control tool.
- `branch`: Lists, creates, or manages branches.

### Step 2: Create and Switch to a Feature Branch

```bash
git checkout -b feature-login
```

**Why we run this:** Creates a new isolated branch named `feature-login` and immediately switches your workspace to it.

**Word Breakdown:**

- `git`: Calls Git command line interface.
- `checkout`: Switches working directory context to a specific branch or commit.
- `-b`: Flag option to create a new branch before switching.
- `feature-login`: Name assigned to the newly created branch.

### Step 3: Write Feature Code and Commit

```bash
echo "<h1>Login Page Code</h1>" > login.html
git add login.html
git commit -m "Add initial layout for login page"
```

**Why we run this:** Simulates writing feature code in isolation without affecting the main code on master.

**Word Breakdown:**

- `echo`: Prints specified text string.
- `> login.html`: Redirects output stream into file named `login.html`.
- `git add`: Adds `login.html` to the staging area.
- `git commit`: Takes snapshot of staged files.
- `-m`: Message parameter for describing commit changes.

### Step 4: Push Feature Branch to Remote GitHub

```bash
git push -u origin feature-login
```

**Why we run this:** Uploads the local feature branch to GitHub so peers and leads can review it.

**Word Breakdown:**

- `git push`: Transmits local commits upstream to remote server.
- `-u`: Sets default upstream branch tracking link.
- `origin`: Default alias for remote GitHub server URL.
- `feature-login`: Target branch name on GitHub remote.

---

## 📌 Module 2: GitHub Pull Request (PR) Workflow

### Step 5: Create Pull Request (PR) on GitHub UI

**Steps on GitHub Web Portal:**

1. Open your GitHub Repository in your web browser.
2. Click **Compare & pull request** banner next to `feature-login`.
3. Enter PR Title: `Feat: Added Login Page UI`.
4. Enter Description: Details of code changes and test status.
5. Assign Team Lead as Reviewer and click **Create Pull Request**.

**Why we do this:** Allows team members to inspect code changes, run automated CI/CD checks, and discuss improvements before merging to production.

### Step 6: Review, Approve and Merge PR

**Steps on GitHub UI (Performed by Lead/Reviewer):**

1. Reviewer inspects diff changes under **Files changed** tab.
2. Click **Approve** → **Merge pull request** → **Confirm merge**.
3. Click **Delete branch** button to clean up remote `feature-login` branch.

---

## 📌 Module 3: Syncing Local Master & Branch Cleanup

### Step 7: Switch Back to Local Master and Pull Merged Code

```bash
git checkout master
git pull origin master
```

**Why we run this:** Switches back to your local master branch and downloads the newly merged code from GitHub so your local workspace stays up to date.

**Word Breakdown:**

- `git checkout master`: Changes active working environment to local master.
- `git pull`: Downloads updates from remote branch and merges them into current branch.
- `origin master`: Specifies remote server alias (`origin`) and branch (`master`).

### Step 8: Delete Merged Local Feature Branch

```bash
git branch -d feature-login
```

**Why we run this:** Cleans up local branch list by deleting feature branch after its changes are merged into master.

**Word Breakdown:**

- `git branch`: Branch management utility.
- `-d`: Safe delete flag (prevents deletion if changes are unmerged).
- `feature-login`: Target branch name to delete.

---

## 📌 Module 4: Real-World Merge Conflict Resolution

### Step 9: Simulate Conflict Scenario

Suppose two developers update the same file (`config.txt`) line at the same time:

```bash
# Developer 1 creates a payment feature branch:
git checkout -b feature-payment
echo "Payment System: Stripe" > config.txt
git add config.txt
git commit -m "Set payment engine to Stripe"

# Meanwhile, Developer 2 updates master directly:
git checkout master
echo "Payment System: PayPal" > config.txt
git add config.txt
git commit -m "Set payment engine to PayPal"
```

### Step 10: Trigger Merge Conflict

```bash
git merge feature-payment
```

**Why we run this:** Tries to merge `feature-payment` into master. Git stops automatically and highlights a conflict because both branches edited line 1 of `config.txt`.

**Output Warning:**

```plaintext
CONFLICT (content): Merge conflict in config.txt.
Automatic merge failed; fix conflicts and then commit the result.
```

### Step 11: Inspect and Resolve Conflict Manually

Open `config.txt` using text editor (e.g., `vi config.txt`). Git inserts conflict marker tags:

```plaintext
<<<<<<< HEAD
Payment System: PayPal
=======
Payment System: Stripe
>>>>>>> feature-payment
```

**Resolution Steps:**

1. Manually edit file to remove conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`).
2. Keep the intended correct line (or combine both if required).
3. Save file with clean content:

```plaintext
Payment System: Stripe
```

### Step 12: Stage and Complete Merge

```bash
git status
git add config.txt
git commit -m "Resolved merge conflict in config.txt by retaining Stripe engine"
```

**Why we run this:** Informs Git that conflict has been resolved, stages corrected file, and records final merge commit.

**Word Breakdown:**

- `git status`: Confirms conflict state is cleared.
- `git add config.txt`: Marks file as resolved and staged.
- `git commit`: Finalizes merge process.

### Step 13: Push Resolved Master Branch to GitHub

```bash
git push origin master
```

**Why we run this:** Syncs resolved master branch up to shared central GitHub repository.
