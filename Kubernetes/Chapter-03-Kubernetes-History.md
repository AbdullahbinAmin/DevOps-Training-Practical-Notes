# Chapter 3 — History of Kubernetes

## Objective
Explain where Kubernetes came from, why it was created, and what its name/logo mean. This is a theory/story chapter, useful for interviews.

## Story: Why Kubernetes Was Created

- In the early days, **Google.com** used to crash after running for some time under heavy traffic.
- Because the system was not "robust" (strong/reliable), Google engineers had to manually restart services again and again.
- In **2014**, Google engineers built an internal system called **Borg** to solve this problem.
- Borg was a very advanced system: applications running on it could **self-heal** (fix themselves automatically) if they crashed, and could **auto-scale** (increase capacity automatically) when traffic increased.
- Google then **open-sourced** this idea, and it became known as **Kubernetes**.
- As of the time of this recording (around 2025), Kubernetes is close to **10 years old**.

## Why It Is Called "K8s"

Kubernetes has the letters: K - u - b - e - r - n - e - t - e - s

- Between "K" and "s", there are exactly **8 letters** (u-b-e-r-n-e-t-e).
- That is why it is short-formed as **K8s** (K + 8 letters + s).

## CNCF (Cloud Native Computing Foundation)

- Kubernetes is a **graduated project** under CNCF.
- CNCF is an organization that supports strong, high-value open-source projects like:
  - Kubernetes
  - ArgoCD
  - Helm
- CNCF gives these projects developers, resources, and support so that the open-source community can grow around them.

## What Kubernetes Actually Is (Simple Definition)

Kubernetes is basically an **Orchestration Tool**.

**Orchestration** = coordinating and managing multiple things (in this case, Docker containers) so they work together properly — deciding when to scale a container, when to heal a crashed container, and how containers communicate with each other.

### Example
Imagine you have multiple Docker containers:
- One for Frontend
- One for Backend
- One for Database

These containers need to:
- Talk to each other
- Scale automatically when needed
- Heal automatically if one crashes

Kubernetes is the tool that manages all of this in a smooth, reliable way.

## Final Result
Students understand:
- Kubernetes started as Google's internal tool called Borg (2014)
- It was later open-sourced and renamed Kubernetes
- The name "K8s" comes from counting letters
- Kubernetes is a CNCF graduated project
- At its core, Kubernetes is a container orchestration tool

---
**Prerequisite for Chapter 4:** Basic understanding of what Kubernetes is (from this chapter).
