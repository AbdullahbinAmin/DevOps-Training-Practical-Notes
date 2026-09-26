# Practical 1 — Introduction to Source Code Management (SCM) and Version Control Systems

## Objective
Understand why Source Code Management exists, what problems it solves, and the difference between Centralized and Distributed Version Control Systems — before touching any Git commands.

## Prerequisites
* OS: Not required for this theory session
* Required software: None yet
* Required account: None yet

## Pre-Check
No commands in this chapter. This is a concept-building session required before any hands-on Git work.

---

## Concept 1 — Why Do We Need Source Code Management?

In a software company, many developers work on the same application at the same time. For example, in a large app like Flipkart:
* One developer works on the login page
* Another works on the product listing/scroll behavior
* Another works on the payment page
* Another works on the cart page
* Another works on how products are displayed

Around 15–20 years ago, before proper tools existed, developers had to manually share their code with each other, check compatibility by hand, and merge things manually. This was slow, error-prone, and hard to track — especially figuring out who changed what and when.

This is the exact problem that **Source Code Management (SCM)** — also called **Version Control** — solves.

**SCM keeps track of every version of your source code**, so that:
* Multiple developers can work on the same codebase without overwriting each other's work
* Every change is recorded and traceable
* Old versions can be recovered at any time

### What is a "version"?
If you write a file today and add new code tomorrow, that updated file is a new **version** of the original file (e.g. `myfile v1.0` becomes `myfile v1.1` after an update). SCM tools manage these versions systematically.

---

## Concept 2 — Two Types of Version Control Systems

There are two broad categories of version control systems:

```text
1. Centralized Version Control System (CVCS)
2. Distributed Version Control System (DVCS)
```

---

## Concept 3 — Centralized Version Control System (CVCS)

**Example tool:** SVN (Subversion) — very popular before Git existed.

**How it worked:**
* There is one **central repository** (a remote storage location on a server) that holds everyone's code.
* Each developer works locally, then uploads (**commits**) their code directly to this central repository.
* Everyone can see everyone else's code because it all lives in one central place, letting developers check each other's work, avoid duplicate work, and pick up where a colleague left off (useful across time zones).

**Problems with CVCS:**
* **Single point of failure:** If the central server goes down or gets corrupted, nobody has a full local copy — work can be completely lost or inaccessible.
* **Requires constant internet/network connection:** Since everything lives centrally, you cannot commit, update, or do almost any meaningful action without being connected to the central server.
* Every action (update, commit) has to travel over the network, so operations are **slower**, especially with a poor internet connection.

These drawbacks led to the rise of Distributed Version Control Systems.

---

## Concept 4 — Distributed Version Control System (DVCS)

**Example tools:** Git (most popular), Mercurial.

**How it's different:**
* Every contributor maintains a **full local copy (local repository)** of the entire project — not just their working files, but the complete history.
* You can commit, view history, create branches, and work entirely **offline**, since your local repository has everything.
* You only need an internet connection when you want to **synchronize** with a remote/central repository (to push your changes out or pull others' changes in).
* Because everyone has a full copy of the repository, if the central server goes down, **no data is lost** — anyone's local copy can be used to restore it. This is what "distributed" means: copies of the code exist in multiple places, not just one central location.

**Additional benefit:** Even a tiny change gets tracked with a **commit ID**, recording exactly who changed what and when. Nobody can make an untracked change — everything is traceable.

---

## Concept 5 — Comparison Table (Frequently Asked in Interviews)

| Point | Centralized VCS (e.g. SVN) | Distributed VCS (e.g. Git) |
|---|---|---|
| Local copy | Client must fetch a copy from server every time; no full local history | Client has a full local copy including complete history |
| Committing | Every commit goes directly to the central server | Commit first goes to your local repository; pushed to remote separately |
| Learning curve | Simpler to learn | Slightly more concepts (branches etc.) but not difficult |
| Branching | Difficult to work with branches | Very easy to create and work with branches |
| Merge conflicts | More frequent | Minimized |
| Offline access | Not possible — needs constant server connection | Fully possible — work offline, sync later |
| Server dependency | If server fails, work can be blocked/lost | If server fails, every contributor still has a full copy |

---

## Concept 6 — Why Git Specifically?

* Git was created by **Linus Torvalds** — the same person who created the Linux kernel.
* Git's first version was released in **2005**.
* Before Git, Linux kernel development used a tool called **BitKeeper** (a third-party paid tool). A licensing dispute led BitKeeper's company to stop offering free access, so Linus Torvalds built Git — a free, open-source alternative designed to work efficiently with Linux-style development.
* Git works especially well on Linux because it was designed by the same person who designed Linux.

> **Common confusion — Git vs GitHub:** Git and GitHub are NOT the same thing.
> * **Git** operates at the local machine level — it's the version control software you install and run.
> * **GitHub** (or GitLab, Bitbucket, etc.) is a **service** that hosts a central/remote repository online (similar to cloud storage for code), letting you back up and share your Git repositories.
> Git works with or without GitHub; GitHub is just one popular place to host your central repository.

> **Trivia:** Microsoft acquired GitHub for a very large sum (reported around $7.5 billion). This concerned some open-source developers, which is part of why alternative platforms like **GitLab** also became popular — as an independent alternative.

---

## Final Result
By the end of this chapter you understand:
* Why Source Code Management exists and what problem it solves
* The difference between Centralized VCS (SVN-style) and Distributed VCS (Git-style)
* Why Git specifically became the industry standard
* That Git (the tool) and GitHub (the hosting service) are different things

## What's Next
**Chapter 2** covers Git's internal architecture — the Working Directory, Staging Area, and Local Repository — and how data moves between them.
