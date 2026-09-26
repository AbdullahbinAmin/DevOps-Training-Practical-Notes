# Practical 2 — Git's Internal Architecture (Working Directory, Staging Area, Local Repository)

## Objective
Understand the three logical areas Git creates on your machine, and how code moves between them using `add` and `commit`, before running any real commands.

## Prerequisites
* OS: Any (conceptual chapter)
* Required software: None yet — Git installation happens in Chapter 3

## Pre-Check
No commands yet — this chapter builds the mental model needed before hands-on work.

---

## Concept — The Three Areas Git Creates

When you run `git init` inside a folder, that folder becomes a **Git repository**. Internally, Git manages your work through **three logical areas**:

```text
1. Working Directory (also called Working Tree / Workspace)
2. Staging Area
3. Local Repository
```

```text
Working Directory  --(git add)-->  Staging Area  --(git commit)-->  Local Repository
```

### 1. Working Directory (Workspace)
This is where you actually **write and edit your code** — create files, modify them, delete them. Think of it as your rough workspace. Whatever you're actively working on lives here first.

### 2. Staging Area
Once you've written some code and decided "yes, I want to save this," you move it to the Staging Area using `git add`. This is like laying out clothes you've shortlisted before finally deciding which ones to buy — you're not committing to it yet, but you're marking it as ready.

**Analogy:** Imagine you're cooking Shahi Paneer. Before you start cooking, you gather your ingredients — paneer, onions, spices — onto the kitchen counter. That gathering step is the Staging Area: you've selected what you plan to use, but you haven't cooked (committed) it yet. You can still remove something from the counter before cooking.

### 3. Local Repository
Once you're sure about what you staged, you run `git commit`. This takes everything in the Staging Area and permanently records it (takes a "snapshot") in your Local Repository. Continuing the cooking analogy — this is the point where you actually start cooking with the ingredients you gathered; the dish is now made and saved.

**Once committed, that snapshot is permanent and gets a unique Commit ID.**

---

## Concept — Why Two Steps (Add, then Commit) Instead of One?

You might wonder: why not just save directly from Working Directory to Local Repository? The two-step process gives you control: you can stage some files and not others, review what's staged before finalizing, and avoid accidentally committing something incomplete or wrong (like forgetting an ingredient before cooking).

---

## Concept — What is a "Commit"?

A **commit** is a **snapshot** of your work at a specific point in time — much like a photograph captures a specific moment. When you commit:
* Git records exactly what your files looked like at that moment
* You get a unique **Commit ID** (a long alphanumeric string, e.g. 40 characters, based on SHA-1 hashing)
* This Commit ID lets you go back and view or restore that exact version later

### Commits Are Incremental
A common misconception: people think each commit stores a full new copy of every file. **This is false.** Git's snapshots are **incremental** — if you have a 5000-line file and you only changed 4 lines, Git only stores information about those 4 changed lines (referencing the rest from the previous snapshot). This is why Git is efficient with storage, even with thousands of commits.

**Important:** Git generally does not delete old data when you make a new commit — it typically **appends** new snapshots. Old history remains accessible.

---

## Concept — Key Terminology Recap

| Term | Meaning |
|---|---|
| Repository (Repo) | A storage location — think of it as a folder — that holds all your project's files and their full history |
| Local Repository | The repository stored on your own machine, created when you run `git init` |
| Remote / Central Repository | A repository stored elsewhere (a server, or a service like GitHub/GitLab) |
| Commit | A saved, permanent snapshot of your work with a unique ID |
| Commit ID | A unique 40-character alphanumeric hash (SHA-1) identifying a specific commit |
| Working Directory | Where you actively edit files |
| Staging Area | Where you mark files as ready to be committed |
| Push | Sending your local repository's committed changes to a remote repository |
| Pull | Fetching and merging changes from a remote repository into your local repository |

---

## Concept — Integrity Checking (Why Commit IDs Matter Beyond Naming)

Git uses a hashing technique (SHA-1) to generate each commit's ID. This hash is calculated from the actual content of what's being committed. If even a single character in a file changes, the resulting hash will be **completely different**.

**Why this matters:** This gives Git a built-in integrity check. If code is sent from one place to another (e.g. pushed to a remote and then pulled by someone else), both sides can verify the hash matches — confirming the data was not tampered with or corrupted in transit. This is one reason Git is considered secure and reliable.

---

## Final Result
By the end of this chapter you understand:
* The three areas Git works with: Working Directory → Staging Area → Local Repository
* What `git add` and `git commit` conceptually do
* That commits are permanent, incremental snapshots identified by a unique hash-based Commit ID
* Why this design makes Git both efficient (incremental storage) and secure (hash-based integrity)

## What's Next
**Chapter 3** covers setting up two real Linux (Ubuntu) EC2 instances on AWS, installing Git on them, and running your first real Git commands.
