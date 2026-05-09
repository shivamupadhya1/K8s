# THURSDAY: Multi-Zone Kubernetes Architecture
### Study Notes for DevOps Engineers (2 Years Experience)

---

## WHY THIS MATTERS (Read This First)

Imagine your entire production app goes down because one AWS/GCP data center in `us-east1-a` had a power issue. That's a **zone failure**. If all your pods were in that one zone — game over.

Multi-Zone Kubernetes is the answer. It spreads your workloads across **multiple zones** (like `us-east1-a`, `us-east1-b`, `us-east1-c`) so that if one zone dies, the rest keep running.

As a DevOps engineer, this is your job: **design for failure before it happens.**

---

## SECTION 1: Core Concepts (Easy Language)

---

### 1.1 What is a Zone?

Think of a **Zone** as a separate physical building (data center) within the same city (region).

```
REGION: us-east1 (Virginia, USA)
  ├── Zone: us-east1-a  (Building A)
  ├── Zone: us-east1-b  (Building B)
  └── Zone: us-east1-c  (Building C)
```

- **Zone failure** = Building A catches fire. Only apps in Building A die.
- **Region failure** = All buildings in Virginia go down. Very rare.

> **Rule of thumb:** Always plan for zone failures. Almost never plan for full region failures (too expensive, rare).

---

### 1.2 Zonal Cluster vs Regional Cluster

This is the most important concept. Learn it cold.

#### Zonal Cluster (Non-HA)
- **Control Plane** (the brain of Kubernetes — API server, etcd, scheduler) lives in **ONE zone only**.
- Worker nodes can still be in multiple zones.
- If that one zone dies → **control plane goes down** → you can't deploy, scale, or manage anything.
- Existing pods may keep running (they don't need control plane to serve traffic), but you're flying blind.

```
ZONAL CLUSTER
─────────────
Control Plane → us-east1-a ONLY  ← single point of failure

Worker Nodes:
  us-east1-a: node-1, node-2
  us-east1-b: node-3, node-4
  us-east1-c: node-5, node-6
```

> **When to use:** Dev/Test environments. Never production for critical apps.

#### Regional Cluster (HA — High Availability)
- **Control Plane** is replicated across **all 3 zones** automatically.
- If one zone dies → 2 other control plane replicas take over. No downtime.
- GKE manages this for you (it's built-in, you just choose "regional").

```
REGIONAL CLUSTER (HA)
──────────────────────
Control Plane Replicas:
  us-east1-a: API Server + etcd  ✓
  us-east1-b: API Server + etcd  ✓
  us-east1-c: API Server + etcd  ✓

Worker Nodes:
  us-east1-a: node-1, node-2
  us-east1-b: node-3, node-4
  us-east1-c: node-5, node-6
```

> **When to use:** Always for production.

---

### 1.3 Node Spread

Node spread = how your **worker nodes** are distributed across zones.

In GKE, when you create a node pool with 6 nodes in a regional cluster:
- GKE automatically places 2 nodes in each zone (2+2+2).
- This is called **balanced node distribution**.

```bash
# Check how your nodes are spread
kubectl get nodes -o wide --show-labels | grep topology.kubernetes.io/zone
```

Example output:
```
node-1   Ready  us-east1-a
node-2   Ready  us-east1-a
node-3   Ready  us-east1-b
node-4   Ready  us-east1-b
node-5   Ready  us-east1-c
node-6   Ready  us-east1-c
```

---

### 1.4 Topology Spread Constraints (The Most Important Config)

Even if your nodes are spread across zones, **Kubernetes by default can still put all 10 pods on nodes in zone-a**.

Topology Spread Constraints tell Kubernetes: **"Spread my pods evenly, don't bunch them up."**

Think of it like assigning seats in a bus: you want people spread evenly, not all sitting at the back.

**Key fields:**
```yaml
topologySpreadConstraints:
  - maxSkew: 1                              # Max difference in pod count between zones
    topologyKey: topology.kubernetes.io/zone # What to spread across (zone)
    whenUnsatisfiable: DoNotSchedule        # What to do if can't spread (block scheduling)
    labelSelector:
      matchLabels:
        app: my-app                         # Which pods this rule applies to
```

**`maxSkew: 1`** means:
- Zone A: 3 pods, Zone B: 3 pods, Zone C: 2 pods → **OK** (max diff = 1)
- Zone A: 5 pods, Zone B: 1 pod, Zone C: 1 pod → **NOT OK** (diff = 4)

**`whenUnsatisfiable` options:**
| Option | Behavior |
|---|---|
| `DoNotSchedule` | Block new pod scheduling if constraint can't be met (strict) |
| `ScheduleAnyway` | Try to spread, but allow violation if no choice (lenient) |

---

### 1.5 Multi-Zone Autoscaling

When Cluster Autoscaler (CA) adds nodes, it must decide which zone to add them to.

**How it works with multi-zone:**
- CA tries to add a node in the zone where the pending pod is being scheduled.
- It respects the node pool's zone configuration.
- In GKE, you configure node pools to span multiple zones → CA adds nodes in the right zone.

**Balance Similar Node Groups:**
```
--balance-similar-node-groups=true
```
This tells CA to keep node counts balanced across zones. Without this, CA might add 10 nodes in zone-a and 0 in zone-b.

**Important:** Topology Spread Constraints + Autoscaler work together. The constraint says "spread pods", the autoscaler says "add nodes where pods need to go."

---

### 1.6 Control Plane High Availability (HA)

| Feature | Zonal (Non-HA) | Regional (HA) |
|---|---|---|
| Control Plane Zones | 1 | 3 |
| API Server uptime if 1 zone fails | DOWN | UP |
| etcd replicas | 1 | 3 (quorum) |
| Cost | Cheaper | ~3x CP cost |
| Use case | Dev/Test | Production |

**etcd quorum (why 3 matters):**
- etcd needs majority (quorum) to function.
- 3 replicas → 1 can die, 2 remain → quorum maintained → cluster keeps working.
- 2 replicas → 1 dies, only 1 remains → no quorum → cluster may break.

> **Always use odd numbers for etcd: 3, 5, 7.**

---

### 1.7 Designing Apps for Zone Outage

Your infrastructure can be multi-zone, but if your **app isn't designed for it**, it still fails.

**Checklist for zone-resilient apps:**

#### ✅ Stateless Services
- Pods don't store data locally.
- Any pod can handle any request.
- If zone-a pods die, zone-b pods just handle all traffic.

#### ✅ Use PodDisruptionBudgets (PDB)
Prevents too many pods from being down at once:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2          # Always keep at least 2 pods running
  selector:
    matchLabels:
      app: my-app
```

#### ✅ Use Readiness Probes
Zone-a pods going down → load balancer should automatically stop routing there:
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

#### ✅ Avoid Zone-Specific Dependencies
- Don't hard-code zone-specific endpoints.
- Use regional load balancers (not zonal).
- Use multi-zone persistent disks or databases (Cloud SQL HA, RDS Multi-AZ).

#### ✅ Test with Chaos Engineering
- Simulate zone outages regularly (drain all nodes in a zone).
- Validate that traffic shifts correctly.
- Measure how long recovery takes.

---

## SECTION 2: Full YAML Reference

---

### 2.1 Deployment with Topology Spread Constraints

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 6
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      # ──────────────────────────────────────────
      # TOPOLOGY SPREAD: Spread pods across zones
      # ──────────────────────────────────────────
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-app
        # Also spread across nodes within each zone
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: my-app

      # ──────────────────────────────────────────
      # ANTI-AFFINITY: Prefer different nodes
      # ──────────────────────────────────────────
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: my-app
                topologyKey: kubernetes.io/hostname

      containers:
        - name: my-app
          image: nginx:1.21
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
          # ──────────────────────────────────────
          # PROBES: Ensure traffic only goes to healthy pods
          # ──────────────────────────────────────
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 20
```

---

### 2.2 PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: default
spec:
  # Keep at least 2 pods running during voluntary disruptions (drains, upgrades)
  minAvailable: 2
  # Alternatively use maxUnavailable:
  # maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

---

## SECTION 3: LABS (Hands-On Practice)

---

### LAB 1: View Zone Topology — Check Node Labels & Zone Distribution

**Goal:** Understand how nodes are spread across zones in your cluster.

**Step 1: Get all nodes with their zone labels**
```bash
kubectl get nodes --show-labels
```

**Step 2: Extract zone info cleanly**
```bash
kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,ZONE:.metadata.labels.topology\.kubernetes\.io/zone,STATUS:.status.conditions[-1].type'
```

Expected output:
```
NAME              ZONE          STATUS
gke-pool-1-abc    us-east1-a    Ready
gke-pool-1-def    us-east1-a    Ready
gke-pool-1-ghi    us-east1-b    Ready
gke-pool-1-jkl    us-east1-b    Ready
gke-pool-1-mno    us-east1-c    Ready
gke-pool-1-pqr    us-east1-c    Ready
```

**Step 3: Count nodes per zone**
```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.labels.topology\.kubernetes\.io/zone}{"\n"}{end}' | sort | uniq -c
```

Expected output:
```
2 us-east1-a
2 us-east1-b
2 us-east1-c
```

**Step 4: Check pod distribution across zones (once you have pods running)**
```bash
# First deploy something
kubectl create deployment zone-test --image=nginx --replicas=6

# Check where pods landed
kubectl get pods -o wide
```

**Step 5: Map pods to zones**
```bash
# Get node for each pod, then get zone for that node
kubectl get pods -o wide | awk 'NR>1 {print $7}' | while read node; do
  kubectl get node $node -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}'
  echo " ($node)"
done
```

**What to observe:**
- Are pods spread evenly? Or did they all land in one zone?
- Without topology constraints, Kubernetes doesn't guarantee even spread.

**Cleanup:**
```bash
kubectl delete deployment zone-test
```

---

### LAB 2: Apply Topology Spread Constraints — Force Workload to Spread Evenly

**Goal:** Apply topology spread constraints and verify pods land evenly across zones.

**Step 1: Create a namespace for this lab**
```bash
kubectl create namespace tsc-lab
```

**Step 2: Deploy WITHOUT topology spread (see the problem first)**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-no-spread
  namespace: tsc-lab
spec:
  replicas: 6
  selector:
    matchLabels:
      app: app-no-spread
  template:
    metadata:
      labels:
        app: app-no-spread
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
EOF
```

**Step 3: Check the distribution (likely uneven)**
```bash
kubectl get pods -n tsc-lab -o wide
# Observe: pods may all be on same zone or random
```

**Step 4: Now deploy WITH topology spread**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-spread
  namespace: tsc-lab
spec:
  replicas: 6
  selector:
    matchLabels:
      app: app-with-spread
  template:
    metadata:
      labels:
        app: app-with-spread
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: app-with-spread
      containers:
        - name: nginx
          image: nginx:1.21
EOF
```

**Step 5: Check the distribution (should be even: 2-2-2)**
```bash
kubectl get pods -n tsc-lab -l app=app-with-spread -o wide
```

Expected:
```
NAME                             READY  NODE              ...
app-with-spread-xxx-aaa          1/1    node-us-east1-a   
app-with-spread-xxx-bbb          1/1    node-us-east1-a   
app-with-spread-xxx-ccc          1/1    node-us-east1-b   
app-with-spread-xxx-ddd          1/1    node-us-east1-b   
app-with-spread-xxx-eee          1/1    node-us-east1-c   
app-with-spread-xxx-fff          1/1    node-us-east1-c   
```

**Step 6: Test what happens if you scale to 7 (uneven by 1 is allowed with maxSkew=1)**
```bash
kubectl scale deployment app-with-spread -n tsc-lab --replicas=7
kubectl get pods -n tsc-lab -l app=app-with-spread -o wide
# Should be 3-2-2 or 2-3-2 or 2-2-3
```

**Step 7: Describe a pending pod (if scheduling fails)**
```bash
kubectl describe pod <pending-pod-name> -n tsc-lab
# Look for: "0/6 nodes are available: ... didn't match pod topology spread constraints"
```

**Cleanup:**
```bash
kubectl delete namespace tsc-lab
```

---

### LAB 3: Drain an Entire Zone — Observe How Deployments React

**Goal:** Simulate a zone failure by draining all nodes in one zone. See how Kubernetes redistributes workloads.

> ⚠️ **WARNING:** Only do this in a test/dev cluster. Draining removes pods from real nodes.

**Step 1: Set up your app with spread constraints and PDB**
```bash
kubectl create namespace drain-lab

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-app
  namespace: drain-lab
spec:
  replicas: 6
  selector:
    matchLabels:
      app: ha-app
  template:
    metadata:
      labels:
        app: ha-app
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: ha-app
      containers:
        - name: nginx
          image: nginx:1.21
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ha-app-pdb
  namespace: drain-lab
spec:
  minAvailable: 4
  selector:
    matchLabels:
      app: ha-app
EOF
```

**Step 2: Verify initial pod distribution**
```bash
kubectl get pods -n drain-lab -o wide
# Note which pods are on which nodes (and which zone)
```

**Step 3: Identify all nodes in zone-a**
```bash
kubectl get nodes -l topology.kubernetes.io/zone=us-east1-a
# Note the node names
```

**Step 4: Cordon all zone-a nodes (no new pods will be scheduled there)**
```bash
for node in $(kubectl get nodes -l topology.kubernetes.io/zone=us-east1-a -o name); do
  kubectl cordon $node
  echo "Cordoned: $node"
done
```

**Step 5: Watch pods in real-time (open a new terminal)**
```bash
watch kubectl get pods -n drain-lab -o wide
```

**Step 6: Drain all zone-a nodes (evict pods from them)**
```bash
for node in $(kubectl get nodes -l topology.kubernetes.io/zone=us-east1-a -o name); do
  kubectl drain $node \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --force \
    --grace-period=30
  echo "Drained: $node"
done
```

**Step 7: Observe what happens**
```bash
# Pods from zone-a should be rescheduled to zone-b and zone-c
kubectl get pods -n drain-lab -o wide

# Check events for what happened
kubectl get events -n drain-lab --sort-by='.lastTimestamp'
```

**Expected behavior:**
- Zone-a pods get evicted.
- Kubernetes tries to reschedule them in zone-b and zone-c.
- If `minAvailable: 4` in PDB, drain will pace itself to not drop below 4.
- Final state: all 6 pods in zone-b and zone-c (3+3).

**Step 8: Check if topology constraint causes pending pods**
```bash
kubectl get pods -n drain-lab
# Some pods may be PENDING because topology constraint says max 1 skew
# But with only 2 zones available, spread works differently
```

**Step 9: Describe pending pods to understand WHY**
```bash
kubectl describe pod <pending-pod> -n drain-lab | grep -A 10 "Events:"
```

**Step 10: Uncordon zone-a nodes (recovery simulation)**
```bash
for node in $(kubectl get nodes -l topology.kubernetes.io/zone=us-east1-a -o name); do
  kubectl uncordon $node
  echo "Uncordoned: $node"
done

# Watch pods redistribute back
watch kubectl get pods -n drain-lab -o wide
```

**Cleanup:**
```bash
kubectl delete namespace drain-lab
```

---

## SECTION 4: Quick Reference Cheat Sheet

```
CLUSTER TYPES
─────────────
Zonal (Non-HA)  → Control plane in 1 zone → Cheap, risky
Regional (HA)   → Control plane in 3 zones → Recommended for prod

ZONE LABELS ON NODES
─────────────────────
topology.kubernetes.io/zone   = us-east1-a
topology.kubernetes.io/region = us-east1
kubernetes.io/hostname        = node-name

TOPOLOGY SPREAD CONSTRAINT KEYS
────────────────────────────────
maxSkew              → Max allowed pod count difference between zones
topologyKey          → What to spread by (zone / hostname)
whenUnsatisfiable    → DoNotSchedule (strict) / ScheduleAnyway (lenient)
labelSelector        → Which pods this rule targets

USEFUL COMMANDS
────────────────
# View nodes with zone info
kubectl get nodes -o wide --show-labels

# Count pods per zone
kubectl get pods -o wide | grep -v NAME | awk '{print $7}' | sort | uniq -c

# Cordon a node (stop new pods)
kubectl cordon <node-name>

# Drain a node (evict pods)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Uncordon a node (bring back)
kubectl uncordon <node-name>

# Check PDB status
kubectl get pdb -A

# Describe events on pods
kubectl describe pod <pod-name> | grep -A 20 Events

RULES TO REMEMBER
──────────────────
1. Always use regional clusters for production.
2. Always add topology spread constraints for critical deployments.
3. Always set PodDisruptionBudgets for critical apps.
4. Always use readiness + liveness probes.
5. Use maxSkew: 1 with DoNotSchedule for strict zone spread.
6. Use maxSkew: 1 with ScheduleAnyway for lenient zone spread.
7. Zonal failures happen. Regional failures almost never happen.
8. Test your zone failure story BEFORE production incidents.
```

---

## SECTION 5: INFRAThrone Mindset — Senior Engineer Thinking

---

### Design for Failure (Not Hope for Uptime)

**Bad mindset:** "Our cloud provider is reliable, zones don't really fail."
**Good mindset:** "Zones WILL fail. When, not if. My design must tolerate it."

**Real incidents to know:**
- AWS us-east-1 has had multiple zone outages affecting major companies.
- GCP has had similar zone-level disruptions.
- Azure has had AZ failures affecting services.

---

### The Cost of Not Doing HA

| Scenario | Cost |
|---|---|
| Zone outage, non-HA cluster | 100% downtime, manual recovery, customer impact |
| Zone outage, HA cluster, no spread constraints | Partial impact, some pods die |
| Zone outage, HA cluster + spread constraints | Near-zero impact, automatic recovery |
| Implementing HA properly | ~20-30% extra infra cost → worth it |

---

### Interview-Ready Answers

**Q: What's the difference between a zonal and regional GKE cluster?**
> A: A zonal cluster has its control plane (API server, etcd) in a single zone. If that zone fails, the control plane goes down and you lose the ability to manage the cluster. A regional cluster replicates the control plane across 3 zones. Zone failures don't impact cluster management. I always use regional clusters for production workloads.

**Q: How do you ensure pods spread evenly across zones?**
> A: I use Kubernetes Topology Spread Constraints with `topologyKey: topology.kubernetes.io/zone` and `maxSkew: 1`. This tells the scheduler to keep pod counts within a difference of 1 across zones. I also use `DoNotSchedule` for critical apps so scheduling fails rather than violating the constraint.

**Q: What happens to pods if a zone goes down?**
> A: Nodes in that zone become NotReady. Kubernetes marks pods on those nodes for eviction after the node timeout (default 5 minutes). Pods are then rescheduled to healthy nodes in surviving zones. The key is having enough capacity in other zones to absorb the extra load — that's why I always over-provision slightly.

**Q: How does PodDisruptionBudget help with zone outages?**
> A: PDB controls voluntary disruptions like node drains and upgrades. If I drain zone-a nodes, PDB ensures I always maintain a minimum number of healthy pods. It prevents Kubernetes from evicting too many pods at once. For zone failures (involuntary), the node controller handles eviction, but PDB still guides the pace.

---

## SECTION 6: Study Links

| Resource | URL |
|---|---|
| GKE Regional Clusters | https://cloud.google.com/kubernetes-engine/docs/concepts/regional-clusters |
| AKS Availability Zones | https://learn.microsoft.com/en-us/azure/aks/availability-zones |
| Kubernetes Topology Spread Constraints | https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/ |
| PodDisruptionBudget | https://kubernetes.io/docs/tasks/run-application/configure-pdb/ |
| Cluster Autoscaler Multi-Zone | https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md |

---

## SECTION 7: Self-Check Questions

Answer these after studying. If you can't, re-read that section.

1. What breaks in a zonal cluster when a zone fails?
2. How many etcd replicas does a regional GKE cluster have?
3. What does `maxSkew: 1` mean in topology spread constraints?
4. What's the difference between `DoNotSchedule` and `ScheduleAnyway`?
5. When you drain a node, what happens to the pods on it?
6. What does a PodDisruptionBudget protect against?
7. Why is `minAvailable` important when simulating zone failure?
8. How would you verify pods are evenly spread across zones using kubectl?
9. What label key holds the zone information on a node?
10. If you have 6 replicas across 3 zones and zone-a goes down, what's the minimum pods running if PDB `minAvailable: 4`?

---

*Answer key: 1-Control plane goes down 2-3 replicas (one per zone) 3-max 1 pod difference between zones 4-block scheduling vs allow with best effort 5-evicted and rescheduled 6-minimum pod availability during voluntary disruptions 7-prevents all pods from being evicted at once 8-kubectl get pods -o wide, map nodes to zones 9-topology.kubernetes.io/zone 10-4 pods minimum (PDB holds)*
