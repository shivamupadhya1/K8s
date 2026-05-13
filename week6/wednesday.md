# WEDNESDAY: ArgoCD GitOps, Sync Strategies & Drift Detection
### Complete Study Material — 3-Year DevOps Engineer Track

---

## TABLE OF CONTENTS

1. [What is GitOps?](#1-what-is-gitops)
2. [Why GitOps? — The Problem It Solves](#2-why-gitops)
3. [ArgoCD — What It Is & How It Works](#3-argocd-deep-dive)
4. [ArgoCD Application CRD — Anatomy](#4-argocd-application-crd)
5. [Pull-Based vs Push-Based Deployments](#5-pull-vs-push)
6. [Sync Strategies — Auto vs Manual](#6-sync-strategies)
7. [Drift Detection — Git vs Cluster Mismatch](#7-drift-detection)
8. [Self-Healing — The Crown Jewel](#8-self-healing)
9. [ArgoCD Architecture — Under the Hood](#9-argocd-architecture)
10. [Real-World Scenarios](#10-real-world-scenarios)
11. [Lab 1 — Deploy an App with ArgoCD](#lab-1-deploy-an-app-with-argocd)
12. [Lab 2 — Induce Drift & Observe](#lab-2-induce-drift--observe)
13. [Lab 3 — Auto-Sync & Self-Healing](#lab-3-auto-sync--self-healing)
14. [Bonus Lab — Sync Waves & Advanced Options](#bonus-lab--sync-waves--advanced-options)
15. [Common Interview Questions](#common-interview-questions)
16. [Cheat Sheet](#cheat-sheet)
17. [Reading Resources](#reading-resources)

---

## 1. WHAT IS GitOps?

### The Simple Explanation

Imagine your entire infrastructure — every deployment, every config, every secret reference — is written down in a notebook. That notebook is **Git**. GitOps says: **whatever is in that notebook IS what should be running in production. Always.**

If someone changes something in production without writing it in the notebook first → that change is **wrong** and must be corrected.

### The Formal Definition

> **GitOps** is an operational framework where Git repositories serve as the single source of truth for declarative infrastructure and application definitions.

### The 4 Core Principles (CNCF OpenGitOps)

| # | Principle | Plain English |
|---|-----------|---------------|
| 1 | **Declarative** | You describe *what* you want, not *how* to get there |
| 2 | **Versioned & Immutable** | Every change has a Git commit hash — full history |
| 3 | **Pulled Automatically** | A software agent pulls changes from Git (not pushed in) |
| 4 | **Continuously Reconciled** | The agent constantly checks Git == Cluster and corrects drift |

### The Mental Model

```
Developer → Git Commit → Git Repo ← ArgoCD watches
                                          ↓
                              Compares Git state vs Cluster state
                                          ↓
                         If different: SYNC (apply Git state to cluster)
```

---

## 2. WHY GitOps? — THE PROBLEM IT SOLVES

### The Old World (Without GitOps)

Picture this: It's 3 AM. Production is down. Your team is asking:

- "Who ran `kubectl apply` last?"
- "Which version is actually deployed right now?"
- "Why does staging work but prod doesn't?"
- "Did someone manually edit that ConfigMap?"

Nobody knows. You're flying blind.

### The Root Causes

```
Problem 1: Manual kubectl apply
  → No audit trail
  → Anyone can change anything
  → State drifts silently

Problem 2: No single source of truth
  → Jenkins pipeline says v1.2
  → Cluster is running v1.1
  → Git has v1.3
  → Which one is real?

Problem 3: Environment inconsistency
  → "Works on staging" but prod was edited manually
  → Drift happens over weeks/months silently
```

### How GitOps Fixes This

| Old Problem | GitOps Solution |
|-------------|----------------|
| "Who changed this?" | Every change = Git commit with author |
| "What version is running?" | Git tag = deployed version, always |
| "My prod config drifted" | ArgoCD detects & auto-corrects |
| "How do I rollback?" | `git revert` = instant rollback |
| "Disaster recovery" | Clone the repo → deploy → done |

### The INFRAThrone Rule

> **"kubectl apply is NOT GitOps."**
>
> Why? Because running `kubectl apply` from your laptop leaves no trace in Git. The cluster state is now ahead of (or behind) Git. You've broken the single source of truth.

---

## 3. ARGOCD DEEP DIVE

### What is ArgoCD?

ArgoCD is a **declarative, GitOps continuous delivery tool for Kubernetes**. It runs *inside* your cluster, watches your Git repo, and makes sure your cluster always matches what Git says.

Think of ArgoCD as a **security guard** that:
1. Constantly checks your Git repo
2. Looks at what's actually running in Kubernetes
3. If they don't match → raises an alarm (drift detected)
4. If auto-sync is on → fixes it automatically

### ArgoCD in One Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Git Repository                        │
│   /apps/myapp/deployment.yaml  (image: myapp:v2)        │
└─────────────────────┬───────────────────────────────────┘
                      │  ArgoCD watches every 3 minutes
                      │  (or via webhook = instant)
                      ▼
┌─────────────────────────────────────────────────────────┐
│                   ArgoCD (in cluster)                    │
│                                                          │
│  Compares:  Git state  ←────────→  Cluster state        │
│             image:v2              image:v1 ← DRIFT!     │
│                                                          │
│  Decision:  AUTO-SYNC ON → apply v2 to cluster          │
│             AUTO-SYNC OFF → mark as "OutOfSync", alert  │
└─────────────────────────────────────────────────────────┘
```

### Key Terminology

| Term | Meaning |
|------|---------|
| **Application** | An ArgoCD CRD that defines what Git repo/path to watch and where to deploy |
| **Source** | The Git repo + path + branch/tag |
| **Destination** | The Kubernetes cluster + namespace to deploy into |
| **Sync** | The act of making cluster state match Git state |
| **Sync Status** | `Synced` (matches Git) or `OutOfSync` (differs from Git) |
| **Health Status** | `Healthy`, `Progressing`, `Degraded`, `Missing` |
| **Drift** | When cluster state differs from Git state |
| **Reconciliation** | ArgoCD's process of comparing and correcting state |

---

## 4. ARGOCD APPLICATION CRD — ANATOMY

### What is a CRD?

CRD = **Custom Resource Definition**. Kubernetes lets you create your own "types" of objects. ArgoCD registers an `Application` type with Kubernetes. When you create an `Application` object, ArgoCD sees it and starts managing that deployment.

### Full Application YAML with Explanation

```yaml
apiVersion: argoproj.io/v1alpha1   # ArgoCD's API group
kind: Application                   # This is the ArgoCD Application type
metadata:
  name: myapp                       # Name of this ArgoCD application
  namespace: argocd                 # MUST be in the argocd namespace

spec:
  # ─── PROJECT ──────────────────────────────────────────
  project: default                  # ArgoCD project (for RBAC/grouping)

  # ─── SOURCE: Where is the desired state? ──────────────
  source:
    repoURL: https://github.com/yourorg/k8s-manifests.git
    targetRevision: HEAD            # Which branch/tag/commit to track
    path: apps/myapp                # Which folder in the repo

  # ─── DESTINATION: Where to deploy? ───────────────────
  destination:
    server: https://kubernetes.default.svc  # In-cluster (this cluster)
    namespace: production                    # Target namespace

  # ─── SYNC POLICY ──────────────────────────────────────
  syncPolicy:
    automated:                      # Enable auto-sync
      prune: true                   # Delete resources removed from Git
      selfHeal: true                # Correct manual cluster changes
    syncOptions:
      - CreateNamespace=true        # Create namespace if it doesn't exist
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true     # Only sync changed resources
```

### Breaking Down Each Section

#### `source` — The Git Side
```yaml
source:
  repoURL: https://github.com/org/repo.git
  #         ↑ Can be: GitHub, GitLab, Bitbucket, any git server

  targetRevision: HEAD
  #               ↑ Options:
  #                 HEAD          = latest commit on default branch
  #                 main          = tip of main branch
  #                 v1.2.3        = specific tag
  #                 abc1234       = specific commit hash

  path: apps/myapp
  #     ↑ Folder inside the repo containing your K8s YAML files
```

#### `destination` — The Cluster Side
```yaml
destination:
  server: https://kubernetes.default.svc
  #        ↑ This special URL means "the same cluster ArgoCD runs in"
  #          For external clusters: https://10.0.0.100:6443

  namespace: production
  #          ↑ The K8s namespace to deploy into
```

#### `syncPolicy` — The Behavior
```yaml
syncPolicy:
  automated:          # Without this block = MANUAL sync only
    prune: true       # If you DELETE a file from Git → delete from cluster
    selfHeal: true    # If someone edits cluster manually → revert to Git

  # No automated block = manual sync required via UI or CLI
```

---

## 5. PULL-BASED vs PUSH-BASED DEPLOYMENTS

This is **critical to understand** for interviews and real-world work.

### Push-Based (Old Way — CI/CD pipelines like Jenkins/GitHub Actions)

```
Developer pushes code
       ↓
CI pipeline builds image
       ↓
CI pipeline runs: kubectl apply -f manifests/
  ← CI server needs cluster credentials!
       ↓
Kubernetes receives the update
```

**Problems with Push:**
- CI server holds powerful cluster credentials (security risk)
- No ongoing reconciliation — drift happens silently
- If someone manually changes the cluster, CI won't fix it
- Hard to know "what was the last deployment?"

### Pull-Based (GitOps Way — ArgoCD)

```
Developer pushes code to Git
       ↓
Git repo is updated
       ↓
ArgoCD (running IN cluster) detects change
       ↓
ArgoCD PULLS manifests from Git
       ↓
ArgoCD applies changes to cluster from inside
```

**Benefits of Pull:**
- No external credentials needed — ArgoCD runs inside the cluster
- Continuous reconciliation — drift is always caught
- Full audit trail in Git
- Works even if CI pipeline is down

### Side-by-Side Comparison

| Feature | Push-Based (Jenkins) | Pull-Based (ArgoCD) |
|---------|---------------------|---------------------|
| Cluster credentials | Stored in CI server | ArgoCD runs inside cluster |
| Drift detection | None | Continuous |
| Audit trail | CI logs | Git commits |
| Manual change protection | None | Auto-corrected |
| Security model | External → Internal | Internal only |

---

## 6. SYNC STRATEGIES — AUTO vs MANUAL

### Manual Sync

Without `syncPolicy.automated`, ArgoCD will:
1. Detect drift (show `OutOfSync` in UI)
2. Wait for you to click "Sync" or run `argocd app sync myapp`
3. Only then apply changes

**When to use Manual Sync:**
- Production environments where you want human approval
- Teams with change management processes
- When combined with ArgoCD `Sync Waves` for ordered deployments

### Auto Sync

With `syncPolicy.automated: {}`, ArgoCD will:
1. Detect drift
2. **Automatically apply** Git state to cluster
3. No human intervention needed

**When to use Auto Sync:**
- Development/staging environments
- Fully automated GitOps pipelines
- When `selfHeal: true` for security

### The `prune` Flag — IMPORTANT

```yaml
syncPolicy:
  automated:
    prune: false  # DEFAULT behavior
    # → If you delete a file from Git, ArgoCD will NOT delete the resource
    # → Orphaned resources pile up in the cluster

    prune: true   # RECOMMENDED
    # → If you delete a file from Git, ArgoCD WILL delete the resource
    # → Cluster stays clean
```

**Real-world example:**
- You have `deployment-v1.yaml` in Git
- You rename it to `deployment-v2.yaml`
- With `prune: false` → you now have TWO deployments running!
- With `prune: true` → old deployment is cleaned up automatically

### The `selfHeal` Flag — CRITICAL

```yaml
syncPolicy:
  automated:
    selfHeal: false  # DEFAULT
    # → If someone runs kubectl edit, ArgoCD notices but does NOT fix it
    # → Drift stays until someone syncs manually

    selfHeal: true   # RECOMMENDED for true GitOps
    # → If someone runs kubectl edit, ArgoCD detects it within ~3 minutes
    # → ArgoCD automatically reverts the change back to Git state
```

### Sync Waves — Advanced (Order of Operations)

Sometimes you need resources deployed in a specific order (e.g., CRDs before apps, namespaces before deployments).

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # Deploy first
    # Wave 0 → Wave 1 → Wave 2 → etc.
```

Example ordering:
```
Wave 0: Namespaces, CRDs
Wave 1: Secrets, ConfigMaps
Wave 2: Deployments, Services
Wave 3: Ingress
```

---

## 7. DRIFT DETECTION — GIT VS CLUSTER MISMATCH

### What is Drift?

**Drift** = when what's running in Kubernetes doesn't match what's defined in Git.

```
Git says:    replicas: 3, image: myapp:v2
Cluster has: replicas: 5, image: myapp:v1
              ↑ DRIFT!     ↑ DRIFT!
```

### How Does Drift Happen?

```
Cause 1: Manual kubectl commands
  kubectl edit deployment myapp
  kubectl scale deployment myapp --replicas=5
  kubectl set image deployment/myapp myapp=myapp:v3

Cause 2: HPA (Horizontal Pod Autoscaler) changes replicas
  ArgoCD is smart about this — it can ignore HPA-managed fields

Cause 3: Admission webhooks mutate resources
  Webhooks can add annotations/labels on admission

Cause 4: Operators modify resources
  Some operators update status/spec fields
```

### How ArgoCD Detects Drift — The 3-Way Diff

ArgoCD doesn't just compare Git vs Cluster. It does a 3-way comparison:
1. **Git state** (desired)
2. **Last applied state** (what ArgoCD last applied)
3. **Live cluster state** (what's actually running)

This lets ArgoCD ignore Kubernetes-internal fields (`resourceVersion`, `uid`, etc.) and only flag real drift.

```
              ┌─────────────────┐
              │   Git State     │ (desired)
              │  image: v2      │
              │  replicas: 3    │
              └────────┬────────┘
                       │
         ArgoCD compares using 3-way diff
                       │
              ┌────────▼────────┐
              │  Cluster State  │ (actual/live)
              │  image: v1      │ ← DIFFERENT
              │  replicas: 5    │ ← DIFFERENT
              └─────────────────┘
                       ↓
              Status: OutOfSync
              Diff shown in ArgoCD UI
```

### Viewing Drift via CLI

```bash
# Check sync status
argocd app get myapp

# See the actual diff
argocd app diff myapp

# Example output:
# ===== apps/Deployment default/myapp ======
# 16c16
# <   image: myapp:v2    ← Git says this
# >   image: myapp:v1    ← Cluster has this
```

---

## 8. SELF-HEALING — THE CROWN JEWEL

### The Self-Healing Loop

```
1. Someone runs: kubectl set image deployment/myapp myapp=hotfix:debug
                          ↓
2. Cluster state changes
                          ↓
3. ArgoCD reconciliation loop runs (every 3 minutes by default)
                          ↓
4. ArgoCD detects: "Cluster != Git"
                          ↓
5. selfHeal: true → ArgoCD automatically syncs
                          ↓
6. Cluster reverts to: image: myapp:v2 (what Git says)
                          ↓
7. ArgoCD sends notification: "App myapp was auto-synced due to drift"
```

### Why Self-Healing Matters (Real Incident Story)

- Junior engineer gets paged at 2 AM — production is erroring
- They `kubectl edit` the deployment to add a debug env var
- They go to sleep, forgetting to revert it
- Next morning: everyone confused why prod has debug settings
- **With self-healing:** that edit would have been reverted in 3 minutes automatically

### The Correct Way to Make Changes

```
WRONG:
kubectl edit deployment myapp       ← Will be reverted by ArgoCD in ~3 min

CORRECT:
1. Edit apps/myapp/deployment.yaml in Git
2. Create PR → Get review → Merge to main
3. ArgoCD detects new commit
4. ArgoCD syncs automatically
5. Cluster updated with full audit trail
```

---

## 9. ARGOCD ARCHITECTURE — UNDER THE HOOD

### Components

```
┌─────────────────────────────────────────────────────────┐
│                    ArgoCD Namespace                      │
│                                                          │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ API Server  │  │  Repo Server │  │  Controller  │   │
│  │             │  │              │  │              │   │
│  │ Handles UI  │  │ Clones Git   │  │ Reconciles   │   │
│  │ CLI & REST  │  │ repos        │  │ Git vs       │   │
│  │ requests    │  │ Renders      │  │ Cluster      │   │
│  │             │  │ manifests    │  │ state        │   │
│  └─────────────┘  └──────────────┘  └──────────────┘   │
│                                                          │
│  ┌─────────────┐  ┌──────────────┐                      │
│  │  Redis      │  │  Dex (OIDC)  │                      │
│  │  Cache      │  │  Auth/SSO    │                      │
│  └─────────────┘  └──────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

### Component Roles

| Component | What It Does |
|-----------|-------------|
| **API Server** | Handles all UI, CLI, and API requests. Manages app CRUD operations |
| **Repo Server** | Clones Git repos, renders manifests (Helm, Kustomize, plain YAML) |
| **Application Controller** | The heart — watches K8s resources, compares with Git, triggers syncs |
| **Redis** | Caches app state and repo data for performance |
| **Dex** | Optional OIDC provider for SSO integration (GitHub, Google, LDAP) |

### The Reconciliation Loop

```
Every ~3 minutes (configurable), OR via webhook:
  1. Fetch desired state from Repo Server (Git)
  2. Fetch live state from Kubernetes API
  3. Compare them (3-way diff)
  4. If same → mark Synced, continue
  5. If different:
       → Mark OutOfSync
       → If autoSync enabled → trigger sync
       → If selfHeal enabled → sync even for manual cluster changes
```

### Webhook Integration (for instant sync)

By default, ArgoCD polls Git every 3 minutes. With webhooks, changes are detected **instantly**:

```
Git push → GitHub sends webhook POST to ArgoCD API
         → ArgoCD immediately refreshes
         → Sync triggered (if auto-sync on)
         → Total time from push to deployed: ~30 seconds
```

---

## 10. REAL-WORLD SCENARIOS

### Scenario 1: The Midnight Hotfix
```
Problem: Critical bug in production
Bad:  kubectl set image deployment/api api=api:hotfix   ← No audit, will revert
Good: Push hotfix commit to Git → ArgoCD auto-syncs → Full audit trail
```

### Scenario 2: Rollback
```
Problem: New deployment broke production
Bad:  kubectl rollout undo deployment/api
Good: git revert HEAD && git push
      OR: git checkout HEAD~1 -- apps/api/deployment.yaml && git push
      → ArgoCD detects, deploys previous version automatically
```

### Scenario 3: Multi-Environment Promotion
```
k8s-manifests/
├── environments/
│   ├── dev/         ← ArgoCD App "myapp-dev" watches this
│   ├── staging/     ← ArgoCD App "myapp-staging" watches this
│   └── prod/        ← ArgoCD App "myapp-prod" watches this

Promotion: update environments/prod/deployment.yaml → push → ArgoCD deploys
```

### Scenario 4: Handling HPA Drift (Advanced)
```
Problem: HPA changes replicas from 3 to 7 (correct!)
         But ArgoCD sees "replicas: 7" vs Git "replicas: 3" and reverts!

Solution: Tell ArgoCD to ignore the replicas field:

spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
```

---

---

# LAB 1: Deploy an App with ArgoCD
## Full Step-by-Step from Zero to Running

**Time:** 45–60 minutes | **Difficulty:** Beginner

---

### Prerequisites

| Tool | Purpose | Notes |
|------|---------|-------|
| `kubectl` | Interact with Kubernetes | https://kubernetes.io/docs/tasks/tools/ |
| `minikube` or `kind` | Local Kubernetes cluster | See Step 1 |
| `argocd` CLI | Control ArgoCD from terminal | Installed in Step 3 |
| GitHub account | Host your GitOps repo | https://github.com |

---

### STEP 1 — Set Up a Local Kubernetes Cluster

**Option A: Minikube (Recommended)**

```bash
# Start minikube with enough resources for ArgoCD
minikube start --cpus=4 --memory=4096 --driver=docker

# Verify
kubectl get nodes
# NAME       STATUS   ROLES           AGE
# minikube   Ready    control-plane   1m
```

**Option B: kind (Kubernetes IN Docker)**

```bash
kind create cluster --name gitops-lab
kubectl cluster-info --context kind-gitops-lab
```

---

### STEP 2 — Install ArgoCD on Kubernetes

ArgoCD runs **inside** your cluster. This is what makes it pull-based.

```bash
# Create a dedicated namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be Running (takes 1-2 minutes)
kubectl get pods -n argocd --watch

# Wait until you see all these Running:
# argocd-application-controller-0        1/1   Running
# argocd-dex-server-xxxx                 1/1   Running
# argocd-notifications-controller-xxxx   1/1   Running
# argocd-redis-xxxx                      1/1   Running
# argocd-repo-server-xxxx                1/1   Running
# argocd-server-xxxx                     1/1   Running
# Press Ctrl+C when done
```

**What just got installed?**
- `argocd-application-controller` — the brain that reconciles Git vs cluster
- `argocd-server` — the UI and API you'll interact with
- `argocd-repo-server` — clones your Git repos
- `argocd-redis` — caching for performance
- `argocd-dex-server` — SSO/auth provider

---

### STEP 3 — Access the ArgoCD UI

```bash
# Get the auto-generated admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
# Example output: xK9mN2pQ8rT1vW3y  ← SAVE THIS

# Forward ArgoCD UI to your machine (keep this terminal open!)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open browser: **https://localhost:8080**
- Username: `admin`
- Password: *(the one you saved above)*

Accept the certificate warning (self-signed, safe for local dev).

**Install ArgoCD CLI** (open a new terminal):

```bash
# Linux/Mac
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Windows (PowerShell)
$version = (Invoke-RestMethod https://api.github.com/repos/argoproj/argo-cd/releases/latest).tag_name
Invoke-WebRequest -Uri "https://github.com/argoproj/argo-cd/releases/download/$version/argocd-windows-amd64.exe" -OutFile argocd.exe

# Login
argocd login localhost:8080 --username admin --password <YOUR-PASSWORD> --insecure
# Output: 'admin:login' logged in successfully
```

---

### STEP 4 — Create Your GitOps Repository

This is the "Git = source of truth" part.

**On GitHub:**
1. Go to https://github.com/new
2. Name it: `gitops-lab-manifests`
3. Make it **Public**
4. Initialize with README: Yes
5. Click "Create repository"

**Clone and create the structure:**

```bash
git clone https://github.com/YOUR-USERNAME/gitops-lab-manifests.git
cd gitops-lab-manifests
mkdir -p apps/myapp
```

**Create `apps/myapp/namespace.yaml`:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
```

**Create `apps/myapp/deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
  labels:
    app: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
```

**Create `apps/myapp/service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

**Push to GitHub:**

```bash
git add .
git commit -m "feat: add myapp kubernetes manifests"
git push origin main
```

---

### STEP 5 — Create the ArgoCD Application

Create file `argocd-app.yaml` (anywhere on your machine, NOT in the manifests repo):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR-USERNAME/gitops-lab-manifests.git
    targetRevision: HEAD
    path: apps/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

Apply it:
```bash
# REPLACE YOUR-USERNAME first!
kubectl apply -f argocd-app.yaml
# application.argoproj.io/myapp created
```

---

### STEP 6 — Sync the Application

```bash
# Check current status
argocd app get myapp
# Sync Status:  OutOfSync  ← Resources don't exist yet
# Health Status: Missing

# Trigger the first sync
argocd app sync myapp

# Verify
kubectl get all -n myapp
# NAME                       READY   STATUS    RESTARTS
# pod/myapp-xxxx-xxxx        1/1     Running   0
# pod/myapp-yyyy-yyyy        1/1     Running   0
#
# NAME            TYPE        CLUSTER-IP
# service/myapp   ClusterIP   10.96.x.x
#
# NAME                    READY   UP-TO-DATE
# deployment.apps/myapp   2/2     2
```

---

### STEP 7 — Update via Git (The GitOps Loop)

**Edit `apps/myapp/deployment.yaml`** — change replicas and image:

```yaml
spec:
  replicas: 3          # was 2
  ...
        image: nginx:1.26  # was 1.25
```

**Push the change:**
```bash
git add apps/myapp/deployment.yaml
git commit -m "feat: scale to 3 replicas and upgrade to nginx 1.26"
git push origin main
```

**Watch ArgoCD detect it:**
```bash
argocd app get myapp --refresh
# Sync Status: OutOfSync  ← Detects the difference

argocd app diff myapp
# Shows: replicas 2→3, image nginx:1.25→1.26

argocd app sync myapp

kubectl get pods -n myapp
# Now 3 pods running with nginx:1.26
```

**Lab 1 Complete.** You have a working GitOps pipeline.

---

---

# LAB 2: Induce Drift & Observe

**Time:** 20–30 minutes | **Prereq:** Lab 1 completed

**Goal:** Manually change something in the cluster and watch ArgoCD catch it. Understand what drift looks like from both the UI and CLI.

---

### STEP 1 — Check the Baseline (Before Inducing Drift)

```bash
# Confirm app is healthy and synced
argocd app get myapp
# Sync Status:   Synced    ← All good
# Health Status: Healthy

# Note current state
kubectl get deployment myapp -n myapp -o yaml | grep -E "replicas:|image:"
# replicas: 3
# image: nginx:1.26
```

---

### STEP 2 — Induce Drift: Change Replicas

Manually scale the deployment **without touching Git**:

```bash
kubectl scale deployment myapp -n myapp --replicas=5

# Verify the cluster change
kubectl get deployment myapp -n myapp
# DESIRED   CURRENT   READY
# 5         5         5     ← Cluster now has 5 replicas
# But Git still says 3!
```

---

### STEP 3 — Induce More Drift: Change the Image

```bash
kubectl set image deployment/myapp myapp=nginx:1.24 -n myapp
# deployment.apps/myapp image updated

# This is exactly what a developer might do in a panic situation
# "Let me just roll back quickly with kubectl"
```

---

### STEP 4 — Induce Drift via kubectl edit

```bash
# This opens the live resource in your editor
kubectl edit deployment myapp -n myapp

# Add this env var under spec.template.spec.containers[0]:
# env:
# - name: DEBUG_MODE
#   value: "true"

# Save and exit
# This simulates a developer adding a debug setting manually
```

---

### STEP 5 — Observe the Drift in ArgoCD

**Wait ~3 minutes** OR force a refresh:

```bash
# Force refresh (makes ArgoCD re-check immediately)
argocd app get myapp --refresh

# Output will show:
# Sync Status:  OutOfSync   ← DRIFT DETECTED
# Health Status: Healthy    ← App still works, but config differs

# See exactly what drifted
argocd app diff myapp
```

**Expected diff output:**
```
===== apps/Deployment myapp/myapp ======
spec:
  replicas:
-   3         ← Git says 3
+   5         ← Cluster has 5

  template.spec.containers[0]:
    image:
-     nginx:1.26   ← Git says 1.26
+     nginx:1.24   ← Cluster has 1.24

    env:
+     - name: DEBUG_MODE    ← Not in Git at all
+       value: "true"
```

---

### STEP 6 — Observe Drift in the ArgoCD UI

Open https://localhost:8080:

1. Your app `myapp` shows a **yellow/orange status** indicating `OutOfSync`
2. Click on the app → you'll see the resource tree
3. Resources with drift show a **yellow triangle** warning icon
4. Click on the `Deployment` resource → click "Diff" tab
5. You'll see a color-coded diff: red = cluster has this, green = Git says this

---

### STEP 7 — Manually Reconcile (Fix the Drift)

Since we don't have auto-sync enabled yet, fix it manually:

```bash
argocd app sync myapp

# ArgoCD will:
# 1. Apply replicas: 3 (overwriting the 5 you set)
# 2. Apply image: nginx:1.26 (overwriting 1.24)
# 3. Remove the DEBUG_MODE env var (not in Git)
```

Verify:
```bash
kubectl get deployment myapp -n myapp -o yaml | grep -E "replicas:|image:|DEBUG"
# replicas: 3       ← Back to Git value
# image: nginx:1.26 ← Back to Git value
# (no DEBUG_MODE)   ← Removed
```

---

### STEP 8 — Induce Drift Again: Delete a Resource

```bash
# Delete the service (simulate someone accidentally deleting it)
kubectl delete service myapp -n myapp
# service "myapp" deleted

# Check ArgoCD
argocd app get myapp --refresh
# The Service resource will show as "Missing" in the diff
# ArgoCD detected that a resource that should exist is gone

# Fix it
argocd app sync myapp
# ArgoCD re-creates the service from Git

kubectl get service myapp -n myapp
# Service is back!
```

---

### STEP 9 — Understanding What ArgoCD Does and Doesn't Catch

```bash
# ✅ ArgoCD WILL catch these drifts:
kubectl scale deployment myapp -n myapp --replicas=10
kubectl set image deployment/myapp myapp=nginx:latest -n myapp
kubectl edit deployment myapp -n myapp    # any spec change
kubectl delete service myapp -n myapp

# ⚠️  ArgoCD ignores these by default (Kubernetes-internal fields):
# - resourceVersion
# - creationTimestamp  
# - uid
# - generation
# These change automatically and ArgoCD knows to ignore them
```

---

### Lab 2 Summary — What You Learned

| Action | ArgoCD Response |
|--------|----------------|
| `kubectl scale` | Detects replica count mismatch → OutOfSync |
| `kubectl set image` | Detects image mismatch → OutOfSync |
| `kubectl edit` | Detects any spec change → OutOfSync |
| `kubectl delete` resource | Detects missing resource → OutOfSync |
| Kubernetes internal fields change | Ignored (correct behavior) |

**The key insight:** ArgoCD is always watching. You cannot "sneak" a cluster change past it. Every drift is caught, logged, and correctable.

---

---

# LAB 3: Auto-Sync & Self-Healing

**Time:** 30–40 minutes | **Prereq:** Lab 2 completed

**Goal:** Enable auto-sync and self-healing. Watch ArgoCD automatically correct drift without any human intervention.

---

### STEP 1 — Enable Auto-Sync via YAML

Update your ArgoCD Application to add auto-sync:

```bash
kubectl edit application myapp -n argocd
```

Find the `spec` section and update it to:

```yaml
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR-USERNAME/gitops-lab-manifests.git
    targetRevision: HEAD
    path: apps/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:           # ← ADD THIS BLOCK
      prune: true        # ← Delete resources removed from Git
      selfHeal: true     # ← Auto-revert manual cluster changes
    syncOptions:
      - CreateNamespace=true
```

Save and exit.

**Or patch it with CLI:**

```bash
argocd app set myapp --sync-policy automated
argocd app set myapp --auto-prune
argocd app set myapp --self-heal
```

---

### STEP 2 — Verify Auto-Sync is Enabled

```bash
argocd app get myapp

# Look for:
# Sync Policy:  Automated (Prune)
# This confirms auto-sync with pruning is active
```

---

### STEP 3 — Test Self-Healing: Scale Replicas

```bash
# Record the time before inducing drift
date
# Wed May 13 10:00:00 UTC 2026

# Induce drift
kubectl scale deployment myapp -n myapp --replicas=10

# Immediately check
kubectl get deployment myapp -n myapp
# DESIRED: 10  ← Drift is there

# NOW WAIT... watch what happens (check every 30 seconds)
watch kubectl get deployment myapp -n myapp
```

Within 2-3 minutes you'll see replicas snap back to 3 automatically!

```bash
# You'll see the ArgoCD sync event
argocd app get myapp
# Look at the "Last Sync" timestamp — it will have just synced

# Check the sync history
argocd app history myapp
# You'll see a new sync entry with reason "self-heal"
```

---

### STEP 4 — Test Self-Healing: Change the Image

```bash
# Change image to something not in Git
kubectl set image deployment/myapp myapp=nginx:WRONG-VERSION -n myapp

# Watch the pods (they'll fail to pull this image)
kubectl get pods -n myapp
# Some pods show ImagePullBackOff or ErrImagePull

# Wait for ArgoCD to self-heal (~1-3 minutes)
# ArgoCD will detect: cluster has nginx:WRONG-VERSION, Git says nginx:1.26
# ArgoCD applies Git state → image reverts to nginx:1.26
# Pods recover automatically

kubectl get pods -n myapp
# All pods Running again with correct image
```

---

### STEP 5 — Test Self-Healing: Delete a Resource

```bash
# Delete the service
kubectl delete service myapp -n myapp
# service "myapp" deleted

# Start watching
kubectl get service myapp -n myapp --watch

# Within ~3 minutes, ArgoCD will recreate it!
# Output:
# NAME    TYPE        CLUSTER-IP
# myapp   ClusterIP   10.96.x.x    ← Service is back!
```

---

### STEP 6 — Test Prune: Delete a Resource from Git

This tests what happens when you **intentionally** remove a resource.

```bash
# In your git repo, delete the service.yaml
cd gitops-lab-manifests
rm apps/myapp/service.yaml
git add .
git commit -m "feat: remove service (switching to headless)"
git push origin main
```

Watch ArgoCD:
```bash
argocd app get myapp --refresh
# In the diff, you'll see service marked for deletion

# Since prune: true, ArgoCD will automatically delete the service
# No manual sync needed!
kubectl get service myapp -n myapp
# Error from server (NotFound): services "myapp" not found
# ← ArgoCD pruned it as expected
```

**Restore the service.yaml before continuing:**
```bash
# Re-add the service.yaml with the content from Lab 1
git add apps/myapp/service.yaml
git commit -m "feat: restore service"
git push origin main
# ArgoCD auto-syncs and recreates the service
```

---

### STEP 7 — Observe the Full GitOps Loop in Real Time

Open two terminal windows side-by-side:

**Terminal 1** — Watch the deployment:
```bash
watch kubectl get deployment myapp -n myapp
```

**Terminal 2** — Induce continuous drift and watch it heal:
```bash
# Drift #1
kubectl scale deployment myapp -n myapp --replicas=99
# Watch Terminal 1 — replicas will jump to 99, then snap back to 3

sleep 30

# Drift #2
kubectl set image deployment/myapp myapp=nginx:1.21 -n myapp
# Watch Terminal 1 — image changes, then reverts to 1.26

sleep 30

# Drift #3
kubectl delete service myapp -n myapp
# Watch ArgoCD UI — service shows Missing, then gets recreated
```

This is **the self-healing GitOps loop in action.**

---

### STEP 8 — Check Sync History and Audit Trail

```bash
# View all sync events
argocd app history myapp

# Output:
# ID   DATE                           REVISION
# 1    2026-05-13 10:00:00 +0000      abc1234 (initial deploy)
# 2    2026-05-13 10:05:00 +0000      def5678 (scaled to 3)
# 3    2026-05-13 10:08:00 +0000      def5678 (self-heal: replicas drift)
# 4    2026-05-13 10:11:00 +0000      def5678 (self-heal: image drift)
```

Every self-heal is logged with a timestamp. This is your full audit trail.

---

### STEP 9 — Auto-Sync with a New Git Commit

Push a real change to Git and watch it deploy with zero manual intervention:

```bash
# Update replicas from 3 to 4 in Git
# Edit apps/myapp/deployment.yaml: replicas: 4
git add .
git commit -m "feat: scale myapp to 4 replicas for load"
git push origin main

# Wait ~3 minutes OR use webhook
# Watch:
watch kubectl get deployment myapp -n myapp
# replicas will change from 3 to 4 automatically!
```

---

### Lab 3 Summary — Self-Healing Results

| Drift Induced | Auto-Healed? | Time to Heal |
|--------------|-------------|-------------|
| kubectl scale replicas | ✅ Yes | ~1-3 minutes |
| kubectl set image | ✅ Yes | ~1-3 minutes |
| kubectl delete service | ✅ Yes | ~1-3 minutes |
| kubectl edit (env vars) | ✅ Yes | ~1-3 minutes |
| Resource deleted from Git | ✅ Yes (pruned) | ~1-3 minutes |
| Git commit pushed | ✅ Auto-deployed | ~1-3 minutes |

---

---

# BONUS LAB: Sync Waves & Advanced Options

**Time:** 20–30 minutes | **Prereq:** Lab 3 completed

---

### Part A — Sync Waves (Deployment Order)

Problem: You need your database to be ready before your app starts.

Create `apps/ordered-app/` in your Git repo:

**`apps/ordered-app/01-namespace.yaml`** (Wave 0 — first):
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ordered-app
  annotations:
    argocd.argoproj.io/sync-wave: "0"
```

**`apps/ordered-app/02-configmap.yaml`** (Wave 1 — second):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: ordered-app
  annotations:
    argocd.argoproj.io/sync-wave: "1"
data:
  DB_HOST: "mydb"
  APP_ENV: "production"
```

**`apps/ordered-app/03-deployment.yaml`** (Wave 2 — last):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ordered-app
  namespace: ordered-app
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ordered-app
  template:
    metadata:
      labels:
        app: ordered-app
    spec:
      containers:
      - name: ordered-app
        image: nginx:1.26
        envFrom:
        - configMapRef:
            name: app-config
```

Create the ArgoCD app and sync:
```bash
argocd app create ordered-app \
  --repo https://github.com/YOUR-USERNAME/gitops-lab-manifests.git \
  --path apps/ordered-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace ordered-app \
  --sync-option CreateNamespace=true

argocd app sync ordered-app
# Watch in UI: Wave 0 deploys → Wave 1 deploys → Wave 2 deploys
# Each wave waits for the previous to be Healthy before starting
```

---

### Part B — Ignore Differences (HPA Use Case)

When HPA controls replicas, tell ArgoCD to ignore that field:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  # ... source and destination ...

  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas          # Ignore replica count (HPA manages this)

  - group: ""
    kind: Service
    jsonPointers:
    - /spec/clusterIP         # Ignore auto-assigned cluster IP
```

Apply:
```bash
kubectl apply -f argocd-app-with-ignore.yaml

# Now test: HPA or manual scale won't trigger OutOfSync for replicas
kubectl scale deployment myapp -n myapp --replicas=99
argocd app get myapp
# Status: Synced  ← Not OutOfSync because replicas is ignored!
```

---

### Part C — Sync Windows (Maintenance Windows)

Prevent ArgoCD from syncing during business hours:

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

# Add to the Application spec:
  syncWindows:
  - kind: deny
    schedule: "0 9 * * 1-5"     # 9 AM on weekdays
    duration: 8h                  # for 8 hours
    applications:
    - "myapp"
    namespaces:
    - "myapp"
    # During business hours 9AM-5PM weekdays: no auto-sync
    # After 5PM and weekends: auto-sync resumes
```

---

---

# COMMON INTERVIEW QUESTIONS

**Q: What is the difference between GitOps and CI/CD?**
> CI/CD automates building and testing. GitOps is a deployment *model* where Git is the source of truth and a controller continuously reconciles cluster state with Git. They complement each other: CI builds the image and updates the Git manifest; GitOps (ArgoCD) deploys what's in Git.

**Q: What happens if someone runs kubectl apply in a GitOps environment?**
> ArgoCD detects the drift within 3 minutes (or instantly via webhook). If `selfHeal: true`, it automatically reverts the change to what Git says. If manual sync, it shows `OutOfSync` status and waits.

**Q: How is ArgoCD different from Flux?**
> Both are GitOps tools. ArgoCD has a UI, multi-cluster management, and a rich Application CRD. Flux is more lightweight and CLI/API-driven. ArgoCD is generally preferred for larger teams needing a UI and multi-tenancy. Flux is preferred for programmatic/operator-based workflows.

**Q: What is drift detection in ArgoCD?**
> Drift detection is ArgoCD's continuous comparison of desired state (Git) vs live state (Kubernetes). It uses a 3-way diff. When they differ, ArgoCD marks the app `OutOfSync` and optionally self-heals.

**Q: What is the purpose of `prune: true`?**
> When resources are removed from Git, `prune: true` tells ArgoCD to also delete them from the cluster. Without it, deleted manifests leave orphaned resources silently running.

**Q: How do you handle secrets in GitOps?**
> Never store plaintext secrets in Git. Use: **Sealed Secrets** (encrypt before committing), **External Secrets Operator** (pull from Vault/AWS Secrets Manager at runtime), **SOPS** (Mozilla's encrypt-in-Git solution), or **HashiCorp Vault** agent injection.

**Q: What is a Sync Wave?**
> Sync waves control deployment order. Lower wave numbers deploy first. Wave 0 completes and is Healthy before Wave 1 starts. Used for ordering CRDs before apps, databases before migrations, namespaces before workloads.

**Q: What does ArgoCD do when a resource not in Git exists in the cluster?**
> With `prune: false` (default): ArgoCD ignores it — the extra resource stays. With `prune: true`: ArgoCD deletes it during the next sync to keep the cluster clean.

**Q: What is the reconciliation interval in ArgoCD?**
> Default is every 3 minutes. Can be tuned via `timeout.reconciliation` in the `argocd-cm` ConfigMap. With webhook integration, reconciliation triggers immediately on Git push.

---

---

# CHEAT SHEET

## ArgoCD CLI Quick Reference

```bash
# ── AUTHENTICATION ─────────────────────────────────────────────────
argocd login localhost:8080 --username admin --password <pw> --insecure
argocd logout localhost:8080

# ── APP MANAGEMENT ─────────────────────────────────────────────────
argocd app list                              # List all apps
argocd app get myapp                         # Get app details
argocd app get myapp --refresh              # Force re-check of Git

# ── SYNC ───────────────────────────────────────────────────────────
argocd app sync myapp                        # Manual sync
argocd app sync myapp --dry-run             # Preview what would change
argocd app sync myapp --prune               # Sync and delete orphans
argocd app sync myapp --force               # Force replace resources

# ── DIFF & HISTORY ─────────────────────────────────────────────────
argocd app diff myapp                        # Show Git vs cluster diff
argocd app history myapp                     # Show sync history
argocd app rollback myapp 3                  # Rollback to history ID 3

# ── CREATE & CONFIGURE ─────────────────────────────────────────────
argocd app create myapp \
  --repo https://github.com/org/repo.git \
  --path apps/myapp \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace myapp

argocd app set myapp --sync-policy automated # Enable auto-sync
argocd app set myapp --auto-prune            # Enable pruning
argocd app set myapp --self-heal             # Enable self-heal
argocd app set myapp --sync-policy none      # Disable auto-sync

# ── CLEANUP ────────────────────────────────────────────────────────
argocd app delete myapp                      # Delete app (not resources)
argocd app delete myapp --cascade            # Delete app AND resources
```

## Application YAML Templates

**Minimal (Manual Sync):**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: HEAD
    path: apps/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
```

**Full Auto-Sync with Self-Healing:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: HEAD
    path: apps/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
```

**With Ignored Fields (HPA-friendly):**
```yaml
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
```

## Status Reference

| Sync Status | Meaning |
|-------------|---------|
| `Synced` | Cluster matches Git — all good |
| `OutOfSync` | Cluster differs from Git — action needed |
| `Unknown` | ArgoCD can't determine status |

| Health Status | Meaning |
|--------------|---------|
| `Healthy` | All resources are running correctly |
| `Progressing` | Resources are updating/starting |
| `Degraded` | Resources have errors (CrashLoopBackOff, etc.) |
| `Missing` | Resources defined in Git don't exist in cluster |
| `Suspended` | Resources are paused |

## Key Flags Reference

| Flag | What it does |
|------|-------------|
| `automated` | Enables auto-sync on Git changes |
| `prune: true` | Deletes cluster resources removed from Git |
| `selfHeal: true` | Auto-reverts manual cluster changes |
| `CreateNamespace=true` | Creates namespace if missing |
| `ApplyOutOfSyncOnly=true` | Only applies changed resources (faster) |

## Sync Wave Annotation

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"   # Lower = earlier
```

## INFRAThrone Rules (Pin These)

```
1. "kubectl apply" is NOT GitOps — it bypasses the audit trail
2. Drift is inevitable — the goal is automatic correction
3. GitOps reduces outages caused by "who changed this?"
4. Every production change should be a Git commit first
5. Self-healing is not a bug — it's the feature
6. If it's not in Git, it doesn't exist
```

---

---

# READING RESOURCES

## Official Documentation
- **ArgoCD Docs:** https://argo-cd.readthedocs.io/en/stable/
- **ArgoCD Best Practices:** https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/
- **ArgoCD Architecture:** https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/
- **Sync Options Reference:** https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/
- **CNCF OpenGitOps Principles:** https://opengitops.dev/

## Deep Dive Articles
- GitOps Overview: https://www.gitops.tech/
- ArgoCD vs Flux Comparison: https://thenewstack.io/gitops-argocd-vs-flux/

## Video Resources (search on YouTube)
- "ArgoCD Tutorial for Beginners" — TechWorld with Nana
- "GitOps with ArgoCD" — KubeCon talks on CNCF YouTube
- "CNCF ArgoCD Deep Dive" — CNCF YouTube channel

## Hands-on Practice Environments
- **Killercoda ArgoCD scenarios:** https://killercoda.com/search?query=argocd
- **Play with Kubernetes:** https://labs.play-with-k8s.com/

---

*INFRAThrone DevOps Track | Wednesday Session | Level: 3-Year Engineer*
