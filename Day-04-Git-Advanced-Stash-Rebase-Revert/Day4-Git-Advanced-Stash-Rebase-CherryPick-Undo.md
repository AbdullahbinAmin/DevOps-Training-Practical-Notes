# Day 3: Git & GitHub — Advanced Operations (Stash, Rebase, Cherry-Pick & Undoing Commits)

IT industry mein daily production environments ke andar **Emergency Hotfixes, Clean Commit History (Rebase), Specific Code Isolation (Cherry-Pick)** aur **Production Issue Rollbacks (Revert/Reset)** bohat critical operations hote hain.

---

## 🏢 Real-World Company Scenario Brief

- **Scenario 1 (Urgent Production Hotfix):** Aap `feature-cart` par kaam kar rahe the ke achaanak production live server par critical bug agaya. Aap ko working code adhoora commit kiye baghair side par save (Stash) karna hai, bug fix karna hai, aur phir wapis feature par aana hai.
- **Scenario 2 (Clean History Requirement):** Company policy hai ke commit history clean horizontal line mein ho, unnecessary merge commits eliminate kiye jayen. Is ke liye `git rebase` use hoga.
- **Scenario 3 (Selective Hotfix Deployment):** Doosri testing branch par aik specific commit bug fix ka hai, pooray branch ko merge kiye bina sirf us 1 commit ko master par lana hai.
- **Scenario 4 (Rollback Bad Code):** Wrong commit push ho chuka hai, usko safely revert karna hai.

---

## 📌 Module 1: Emergency Stashing (Uncommitted Work Ko Hide / Save Karna)

### Step 1: Uncommitted Work Ko Temporarily Save Karein

```bash
git stash save "Work in progress: Cart page UI"
```

**Why we run this:** Uncommitted changes aur modified files ko temporarly internal stack memory mein save kar deta hai aur working workspace ko clean HEAD state par le aata hai.

**Word Breakdown:**

- `git`: Invokes Git command-line tool.
- `stash`: Stores modified uncommitted files in a temporary safety stack.
- `save "..."`: Adds a descriptive custom message label to the stash entry.

### Step 2: Saved Stashes Ki List Dekhein

```bash
git stash list
```

**Why we run this:** Temporary memory stack mein jitne bhi stashes saved hain unki IDs (`stash@{0}`, `stash@{1}`) aur custom labels show karta hai.

**Word Breakdown:**

- `git stash`: Stash command operations suite.
- `list`: Prints stored stash list references.

### Step 3: Stash Work Ko Wapis Workspace Mein Restore Karein

```bash
git stash pop
```

**Why we run this:** Sab se latest saved stash (`stash@{0}`) ko wapis working directory mein restore karta hai aur stack memory se delete kar deta hai.

**Word Breakdown:**

- `pop`: Restores the latest stash entry and removes it from the stash stack.

---

## 📌 Module 2: Git Rebase (Clean Commit History Maintain Karna)

### Step 4: Master Branch Changes Ko Rebase Karein

```bash
git checkout feature-cart
git rebase master
```

**Why we run this:** Feature branch ke commits ko master ke sab se latest commit ke upar re-apply karta hai. Is se unnecessary merge commits create nahi hote aur history clean linear line rehti hai.

**Word Breakdown:**

- `git`: Invokes Git CLI.
- `rebase`: Re-applies local commits on top of another base tip.
- `master`: Target base branch.

---

## 📌 Module 3: Git Cherry-Pick (Selective Single Commit Merge)

### Step 5: Kisi Doosri Branch Se Selective Commit Pick Karein

```bash
git checkout master
git cherry-pick <commit-id>
```

**Why we run this:** Kisi doosri branch (e.g. experimental/bugfix branch) se sirf aik specific commit SHA hash pick kar ke target branch par apply kar deta hai, baghair poora branch merge kiye.

**Word Breakdown:**

- `git`: Invokes Git CLI tool.
- `cherry-pick`: Applies the exact changes introduced by a single target commit.
- `<commit-id>`: Unique SHA hash identifier of the target commit.

---

## 📌 Module 4: Undoing Changes & Rollbacks (Revert vs Reset)

### Step 6: Pushed Bad Commit Ko Safely Undo / Revert Karein

```bash
git revert <commit-id>
```

**Why we run this:** Revert aik new commit create karta hai jo specific old commit ke changes ko reverse karta hai. Shared/Remote production branches ke liye yeh safe aur standard practice hai kyun ke history delete nahi hoti.

**Word Breakdown:**

- `git revert`: Creates a new reversing commit for the target commit ID.

### Step 7: Local Mistakes Ko Hard Reset Karein (Local Undo)

```bash
git reset --hard <commit-id>
```

**Why we run this:** Current local branch ko target commit ID par hard force move kar deta hai aur beech ke saare commits aur uncommitted file changes completely discard kar deta hai.

**Word Breakdown:**

- `reset`: Moves current HEAD branch point to target commit.
- `--hard`: Destructive flag that clears working index and discards all local uncommitted changes.
