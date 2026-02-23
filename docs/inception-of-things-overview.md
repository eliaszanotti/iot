# Inception-of-Things (IoT) - Project Overview

> System Administration exercise - Introduction to Kubernetes (K3s, K3d), Vagrant, and Argo CD

## Project Goal

This project aims to deepen your knowledge by making you use **K3d** and **K3s** with **Vagrant**. You will learn how to:
- Set up a personal virtual machine with Vagrant
- Use K3s and its Ingress
- Discover K3d to simplify your life

---

## Project Structure

The project consists of **3 mandatory parts** + **1 bonus part**.

### Part 1: K3s and Vagrant

**Objective**: Set up a K3s cluster with 2 virtual machines.

**Requirements**:
- Create 2 VMs with **Vagrant** (latest stable distribution of your choice)
- Minimal resources: **1 CPU, 512-1024 MB RAM**

| Machine | Hostname | IP | Role |
|---------|----------|-----|------|
| 1 | `<login>S` | `192.168.56.110` | K3s Controller |
| 2 | `<login>SW` | `192.168.56.111` | K3s Agent |

- Install **K3s** in controller mode on the first machine
- Install **K3s** in agent mode on the second machine
- Use **kubectl** to manage the cluster
- SSH connection without password on both machines

---

### Part 2: K3s and Three Simple Applications

**Objective**: Deploy 3 web applications with Ingress-based routing.

**Requirements**:
- 1 VM with K3s in server mode
- 3 web applications accessible via `192.168.56.110`

**Routing by HOST header**:
- `app1.com` → **app1**
- `app2.com` → **app2** (must have **3 replicas**)
- Default (no HOST) → **app3**

```
Client Request → 192.168.56.110
                    │
                    ▼
              K3s Ingress
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
     app1        app2 (3x)    app3
```

---

### Part 3: K3d and Argo CD

**Objective**: Set up a GitOps continuous integration pipeline.

**Requirements**:
- Install **Docker** and **K3d** (K3s in Docker)
- Write an installation script
- Deploy **Argo CD** for GitOps

**Namespaces**:
- `argocd` - Dedicated to Argo CD
- `dev` - Contains the application

**Application**:
- Must have **2 versions** (tags: `v1` and `v2`)
- Options:
  - Use Wil's pre-made app: `wil42/playground` on Docker Hub
  - Create your own public Docker Hub image

**Workflow**:
1. Push configuration files to a **public GitHub repository**
2. Argo CD automatically deploys from GitHub
3. Update the version in GitHub → Argo CD syncs automatically
4. Application updates without manual intervention

**Verification**:
```bash
kubectl get ns  # Shows argocd and dev
kubectl get pods -n dev  # Shows the application pod
curl http://localhost:8888/  # Returns current version
```

---

### Bonus Part: GitLab Integration

**Objective**: Add GitLab to the Part 3 infrastructure.

**Requirements**:
- Use the latest version of GitLab
- GitLab instance runs **locally**
- Configure GitLab to work with your cluster
- Create a dedicated namespace: `gitlab`
- Everything from Part 3 must work with local GitLab

**Note**: You can use tools like **Helm** for this bonus.

---

## Submission Structure

```
/
├── p1/                   # Part 1: K3s + Vagrant
│   ├── Vagrantfile
│   ├── scripts/
│   └── confs/
├── p2/                   # Part 2: K3s + 3 Apps
│   ├── Vagrantfile
│   ├── scripts/
│   └── confs/
├── p3/                   # Part 3: K3d + Argo CD
│   ├── scripts/
│   └── confs/
└── bonus/                # Bonus: GitLab (optional)
    ├── Vagrantfile
    ├── scripts/
    └── confs/
```

- **scripts/**: All necessary scripts
- **confs/**: All configuration files

---

## Important Notes

- The whole project must be done in a **virtual machine**
- Read documentation extensively (K8s, K3s, K3d)
- This is a **minimal introduction** to Kubernetes
- The bonus will only be assessed if the mandatory part is flawless

---

## Tools Summary

| Tool | Purpose |
|------|---------|
| **Vagrant** | VM management and provisioning |
| **K3s** | Lightweight Kubernetes distribution |
| **K3d** | K3s wrapped in Docker (for local dev) |
| **kubectl** | Kubernetes CLI |
| **Argo CD** | GitOps continuous delivery |
| **Ingress** | HTTP/S routing for Kubernetes |
| **Docker Hub** | Container registry |
| **GitHub** | GitOps source of truth |
