# Chapter 1 — Introduction & Overview of Kubernetes Concept

## Objective
Give students a quick roadmap of what this whole Kubernetes course will cover, so they know what is coming (basics, advanced topics, networking, scaling, security, monitoring, and a final real project).

## What This Course Will Cover (Roadmap)

### Core Concepts
- Difference between Monolithic and Microservices architecture
- Kubernetes architecture (how it works internally)
- Hands-on cluster setup (Local machine and AWS EC2)
- `kubectl` command line tool
- Pods
- Namespaces

### Intermediate / Advanced Concepts
- Deployments
- StatefulSets
- DaemonSets
- ReplicaSets
- CronJobs

### Networking Concepts
- Services
- Ingress

### Storage Concepts
- Persistent Volumes (PV)
- Storage Classes
- Persistent Volume Claims (PVC)

### Production / Scaling Concepts
- Horizontal Pod Autoscaling (HPA)
- Vertical Pod Autoscaling (VPA)
- Node Affinity, Limits, Probes (readiness/liveness)

### Security
- Role Based Access Control (RBAC)

### Monitoring & Deployment
- Monitoring using Prometheus and Grafana
- Helm
- CI/CD with Jenkins
- GitOps using ArgoCD

## Final Project (End Goal of the Course)
By the end, students will deploy a **3-tier application** on an **AWS EKS cluster**, using:
- CI/CD pipeline (Jenkins)
- GitOps deployment (ArgoCD)
- MongoDB as database
- A Python application
- Monitoring with Prometheus and Grafana

## Teaching Note
This is one long, single video/course covering everything from basics to production-level Kubernetes. Students should keep their laptop and notes ready before starting, since the content is dense and continuous.

## Final Result
Students should walk away understanding the complete learning path of this course before jumping into technical setup, so they know what depth to expect in each topic.

---
**Prerequisite for Chapter 2:** None. This is just an orientation chapter.
