# GitOps with ArgoCD & k3s
## A Complete Guide to Your Self-Training Setup

**Author:** AJ  
**Date:** February 18, 2026  
**Version:** 1.0

---

## Table of Contents
1. [What We Built](#what-we-built)
2. [Understanding the Components](#understanding-the-components)
3. [What is GitOps?](#what-is-gitops)
4. [How It All Works Together](#how-it-all-works-together)
5. [Your Repository Structure](#your-repository-structure)
6. [The Deployment Flow](#the-deployment-flow)
7. [Advantages of GitOps](#advantages-of-gitops)
8. [Traditional vs GitOps Comparison](#traditional-vs-gitops-comparison)
9. [What the ArgoCD Dashboard Shows](#what-the-argocd-dashboard-shows)
10. [Common Commands Reference](#common-commands-reference)
11. [Next Steps for Learning](#next-steps-for-learning)

---

## What We Built

We created a **fully automated deployment pipeline** where:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   GitHub Repository ──────► ArgoCD ──────► Kubernetes (k3s)             │
│   (Source of Truth)        (Controller)    (Runtime)                    │
│                                                                          │
│   You push code here       Watches for     Runs your                    │
│                            changes         applications                  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**In simple terms:** You change a file in Git, and your application automatically updates in the cluster. No manual `kubectl apply` needed!

---

## Understanding the Components

### 1. k3s (Lightweight Kubernetes)

**What it is:** A certified Kubernetes distribution designed to be lightweight and easy to install.

**Why we chose it:**
- Single binary installation
- Low resource usage (perfect for laptops/VMs)
- Includes everything: container runtime, networking, load balancer
- Production-ready but simple enough for learning

**What it does in our setup:**
- Runs your containerized applications (pods)
- Manages networking between services
- Handles load balancing via Traefik (included)
- Provides the infrastructure layer

### 2. ArgoCD (GitOps Controller)

**What it is:** A declarative, GitOps continuous delivery tool for Kubernetes.

**Why we chose it:**
- Industry standard for GitOps
- Beautiful web UI for visualization
- Automatic sync capabilities
- Built-in health monitoring

**What it does in our setup:**
- Connects to your GitHub repository
- Watches for changes in the `main` branch
- Compares Git state vs Cluster state
- Automatically applies changes when detected
- Shows real-time status in the dashboard

### 3. Kustomize (Configuration Management)

**What it is:** A tool for customizing Kubernetes configurations without templates.

**Why we use it:**
- Native to Kubernetes (no extra installation)
- Supports overlays for different environments
- No complex templating syntax
- Easy to understand and maintain

**What it does in our setup:**
- Defines base configurations (deployment, service, ingress)
- Creates environment-specific overlays (dev, staging, prod)
- Manages differences between environments

---

## What is GitOps?

### Definition
GitOps is a way of implementing Continuous Deployment for cloud-native applications. It uses **Git as the single source of truth** for declarative infrastructure and applications.

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Declarative** | The entire system is described declaratively (YAML files) |
| **Versioned** | All changes are tracked in Git with full history |
| **Automated** | Changes are automatically applied to the cluster |
| **Self-healing** | The system corrects drift automatically |

### The GitOps Equation

```
Desired State (Git) = Actual State (Cluster)
```

If someone manually changes something in the cluster, ArgoCD will detect the **drift** and restore it to match Git.

---

## How It All Works Together

### Step-by-Step Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Step 1: Developer pushes code to GitHub                                   │
│          ↓                                                                  │
│  Step 2: ArgoCD detects the change (polls every 3 minutes or webhook)      │
│          ↓                                                                  │
│  Step 3: ArgoCD compares Git manifests vs Cluster state                    │
│          ↓                                                                  │
│  Step 4: If different → ArgoCD syncs (applies changes to k3s)              │
│          ↓                                                                  │
│  Step 5: k3s creates/updates pods, services, ingress                       │
│          ↓                                                                  │
│  Step 6: ArgoCD reports status (Synced ✓, Healthy ✓)                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### What Happens When You `git push`

1. **Git receives the commit** → stored in GitHub
2. **ArgoCD polls the repo** → detects new commit `cbc93ce`
3. **ArgoCD runs Kustomize** → generates final YAML
4. **Compares with cluster** → finds differences
5. **Applies changes** → `kubectl apply` internally
6. **k3s schedules pods** → pulls image, starts container
7. **Health checks pass** → status becomes "Healthy"

---

## Your Repository Structure

```
devops-pulse-gitops/
│
├── apps/
│   └── devops-pulse/
│       ├── base/                      # Shared configurations
│       │   ├── deployment.yaml        # Pod specification
│       │   ├── service.yaml           # Network exposure
│       │   ├── ingress.yaml           # External routing
│       │   └── kustomization.yaml     # Base resources list
│       │
│       └── overlays/                  # Environment-specific configs
│           ├── dev/
│           │   └── kustomization.yaml # Dev overrides (fewer resources)
│           ├── staging/
│           │   └── kustomization.yaml # Staging overrides
│           └── production/
│               └── kustomization.yaml # Prod overrides (more replicas)
│
├── argocd/
│   └── applications/
│       └── devops-pulse-dev.yaml      # ArgoCD Application definition
│
└── docs/
    └── GitOps-ArgoCD-Guide.md         # This document
```

### File Explanations

| File | Purpose |
|------|---------|
| `deployment.yaml` | Defines the pod: image, ports, resources, health checks |
| `service.yaml` | Creates internal DNS name and load balancing |
| `ingress.yaml` | Routes external traffic to the service |
| `kustomization.yaml` (base) | Lists all base resources |
| `kustomization.yaml` (overlay) | Customizes for specific environment |
| `devops-pulse-dev.yaml` | Tells ArgoCD what to deploy and where |

---

## The Deployment Flow

### Visual Representation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          YOUR LAPTOP (LT-0057)                             │
│                                                                             │
│  ┌──────────────┐     ┌──────────────────────────────────────────────────┐ │
│  │              │     │               k3s CLUSTER                        │ │
│  │   VS Code    │     │                                                  │ │
│  │   Editor     │     │  ┌─────────────────────────────────────────────┐ │ │
│  │              │     │  │            argocd namespace                 │ │ │
│  │  Edit YAML   │     │  │  ┌─────────────────────────────────────┐   │ │ │
│  │      │       │     │  │  │         ArgoCD Server               │   │ │ │
│  │      ▼       │     │  │  │   - Watches GitHub repo             │   │ │ │
│  │  git push ───┼─────┼──┼──┼─► - Compares state                  │   │ │ │
│  │              │     │  │  │   - Syncs changes                   │   │ │ │
│  └──────────────┘     │  │  └──────────────────┬──────────────────┘   │ │ │
│         │             │  └────────────────────────────────────────────┘ │ │
│         │             │                        │                        │ │
│         │             │                        ▼                        │ │
│         │             │  ┌─────────────────────────────────────────────┐ │ │
│         │             │  │        devops-pulse-dev namespace          │ │ │
│         │             │  │                                             │ │ │
│         │             │  │   ┌───────────┐    ┌───────────┐           │ │ │
│         │             │  │   │    Pod    │◄───│  Service  │           │ │ │
│         │             │  │   │   nginx   │    │  :80      │           │ │ │
│         │             │  │   └───────────┘    └─────┬─────┘           │ │ │
│         │             │  │                          │                  │ │ │
│         │             │  │                    ┌─────▼─────┐           │ │ │
│         │             │  │                    │  Ingress  │           │ │ │
│         │             │  │                    └───────────┘           │ │ │
│         │             │  └─────────────────────────────────────────────┘ │ │
│         │             └──────────────────────────────────────────────────┘ │
│         │                                                                   │
│         └─────────────────► GitHub (acg8326/devops-pulse-gitops)           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Advantages of GitOps

### 1. **Single Source of Truth**
- Everything is in Git
- No "what's actually running?" confusion
- Easy to audit and review

### 2. **Version Control for Infrastructure**
- Full history of all changes
- Easy rollback: `git revert` and push
- Blame/trace who changed what

### 3. **Automated Deployments**
- No manual `kubectl apply`
- Reduces human error
- Consistent deployments every time

### 4. **Self-Healing**
- Cluster always matches Git
- Manual changes are reverted
- Drift detection built-in

### 5. **Security**
- Developers don't need cluster access
- Only Git credentials required
- Audit trail in Git commits

### 6. **Easy Rollbacks**
```bash
# Traditional way (risky)
kubectl rollout undo deployment/app

# GitOps way (safe)
git revert <commit>
git push
# ArgoCD automatically reverts the cluster!
```

### 7. **Multi-Environment Support**
- Same base, different overlays
- Dev → Staging → Production
- Consistent promotion path

### 8. **Disaster Recovery**
- Cluster dies? No problem!
- Spin up new cluster
- Point ArgoCD to Git
- Everything restored automatically

---

## Traditional vs GitOps Comparison

| Aspect | Traditional CI/CD | GitOps |
|--------|-------------------|--------|
| **Deployment trigger** | CI pipeline pushes to cluster | Git commit triggers pull |
| **Cluster access** | CI needs cluster credentials | Only ArgoCD needs access |
| **State tracking** | Unknown until you check | Git IS the state |
| **Rollback** | Complex, manual | `git revert` + push |
| **Drift detection** | Manual audits | Automatic |
| **Multi-env** | Separate pipelines | Overlays in same repo |
| **Security** | Credentials in CI | Credentials only in cluster |

### Traditional Flow (Push-based)
```
Developer → Git → CI/CD Pipeline → kubectl apply → Cluster
                        ↑
                  Has cluster creds
                  (security risk)
```

### GitOps Flow (Pull-based)
```
Developer → Git ← ArgoCD (watches) → Cluster
                        ↑
                  Lives IN cluster
                  (secure)
```

---

## What the ArgoCD Dashboard Shows

Based on your screenshot:

### Application Health: ✅ Healthy
All resources are running correctly.

### Sync Status: ✅ Synced to main (cbc93ce)
The cluster matches the latest Git commit.

### Resource Tree Visualization

```
devops-pulse-dev (Application)
    │
    ├── devops-pulse-backend (Service)
    │       │
    │       ├── devops-pulse-backend (Endpoints)
    │       └── devops-pulse-backend-htbn2 (EndpointSlice)
    │
    ├── devops-pulse-backend (Deployment)
    │       │
    │       ├── devops-pulse-backend-c79bc... (ReplicaSet) ← Active, rev:2
    │       │       │
    │       │       └── devops-pulse-backend-c79bc... (Pod) ← Running 1/1
    │       │
    │       └── devops-pulse-backend-7d667... (ReplicaSet) ← Old, rev:1, scaled to 0
    │
    └── devops-pulse (Ingress)
```

### What the Colors Mean
- 💚 **Green heart** = Healthy
- 🟢 **Green checkmark** = Synced
- 🟡 **Yellow** = Progressing
- 🔴 **Red** = Degraded/Error

---

## Common Commands Reference

### kubectl Commands
```bash
# View all resources in namespace
kubectl get all -n devops-pulse-dev

# Watch pods in real-time
kubectl get pods -n devops-pulse-dev -w

# View pod logs
kubectl logs -n devops-pulse-dev <pod-name>

# Describe a resource
kubectl describe pod -n devops-pulse-dev <pod-name>

# Port-forward to access locally
kubectl port-forward svc/devops-pulse-backend -n devops-pulse-dev 8080:80
```

### ArgoCD CLI Commands
```bash
# Login to ArgoCD
argocd login localhost:8080 --insecure --username admin

# List applications
argocd app list

# Get application status
argocd app get devops-pulse-dev

# Force sync
argocd app sync devops-pulse-dev

# View app history
argocd app history devops-pulse-dev

# Rollback to previous version
argocd app rollback devops-pulse-dev <revision>
```

### Git Commands for GitOps
```bash
# Make changes and deploy
git add -A
git commit -m "feat: update deployment"
git push origin main

# Rollback a deployment
git revert HEAD
git push origin main
```

---

## Next Steps for Learning

### 1. **Add More Environments**
Deploy to staging and production using the existing overlays.

### 2. **Try a Rollback**
Make a breaking change, then `git revert` to practice recovery.

### 3. **Add Health Checks**
Customize readiness and liveness probes for real applications.

### 4. **Explore ArgoCD Features**
- Sync windows (deploy only during certain hours)
- Notifications (Slack/email on sync)
- RBAC (role-based access control)

### 5. **Add a Real Application**
Replace nginx with your own Docker image.

### 6. **Implement CI Pipeline**
Add GitHub Actions to build images on push.

---

## Summary

**What you accomplished:**

1. ✅ Installed k3s (lightweight Kubernetes) on your laptop
2. ✅ Deployed ArgoCD v2.13.3 (GitOps controller)
3. ✅ Created a GitOps repository with Kustomize structure
4. ✅ Connected ArgoCD to GitHub
5. ✅ Deployed your first application via GitOps
6. ✅ Witnessed automatic sync when you pushed changes

**The big picture:**

You now have a professional-grade deployment pipeline that mirrors how companies like Netflix, Spotify, and Intuit manage their Kubernetes deployments. The same principles apply whether you're deploying 1 app or 1,000.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                     GITOPS QUICK REFERENCE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Deploy changes:     git push origin main                       │
│  Check status:       argocd app get devops-pulse-dev            │
│  Force sync:         argocd app sync devops-pulse-dev           │
│  View pods:          kubectl get pods -n devops-pulse-dev       │
│  View logs:          kubectl logs -n devops-pulse-dev <pod>     │
│  Rollback:           git revert HEAD && git push                │
│  ArgoCD UI:          https://localhost:8080                     │
│  ArgoCD login:       admin / (get from secret)                  │
│                                                                 │
│  Repo: https://github.com/acg8326/devops-pulse-gitops           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

*Document created as part of self-training on GitOps, ArgoCD, and Kubernetes*
