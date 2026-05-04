
# MONDAY: PDBs, preStop Hooks & Graceful Shutdown
### InfraThrone Study Notes — Complete with Hands-On Labs

---

## SECTION 1: WHAT HAPPENS WHEN A POD IS TERMINATED

### The Full Termination Lifecycle

When Kubernetes decides to terminate a pod (rolling update, node drain, scale-down, eviction), this is the exact sequence:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    POD TERMINATION SEQUENCE                         │
│                                                                     │
│  1. Pod marked "Terminating" in etcd                                │
│  2. Endpoint controller removes pod from Service endpoints          │
│     → new traffic stops routing to this pod                         │
│  3. preStop hook executes (if defined) — blocks container stop      │
│  4. SIGTERM sent to PID 1 inside container                          │
│  5. terminationGracePeriodSeconds countdown begins (default: 30s)   │
│  6. App should finish in-flight requests, close DB connections,     │
│     flush buffers, etc.                                             │
│  7. If still running after grace period → SIGKILL (forced kill)     │
│                                                                     │
│  TIMELINE:                                                          │
│  t=0s   → preStop starts                                            │
│  t=0s   → SIGTERM sent                                              │
│  t=30s  → SIGKILL if not exited (default grace period)             │
└─────────────────────────────────────────────────────────────────────┘
```

### Critical Race Condition You Must Know

There is a **race condition** between:
- The endpoint controller removing the pod from Service endpoints
- The kubelet sending SIGTERM to the container

These happen **in parallel**, not sequentially. This means:

```
Timeline with race condition:
  t=0: kubelet starts termination
  t=0: kubelet sends SIGTERM to container    ← both happen
  t=0: endpoint controller notified          ← simultaneously
  t=??: kube-proxy updates iptables rules    ← takes a few seconds
  
PROBLEM: Container gets SIGTERM at t=0, but kube-proxy may still
         be routing traffic to it until t=2s or t=3s.
         
SOLUTION: Use preStop hook with a sleep to allow iptables to propagate
          BEFORE the container starts shutting down.
```

---

## SECTION 2: SIGTERM → GRACEFUL SHUTDOWN → SIGKILL TIMELINE

### The Three Signals

| Signal | What it means | App behavior expected |
|--------|---------------|----------------------|
| `SIGTERM` (15) | "Please shut down gracefully" | Stop accepting new work, finish in-flight, exit cleanly |
| `SIGKILL` (9) | "Die immediately, no choice" | Forced kill, no cleanup possible |
| `SIGINT` (2) | Ctrl+C interrupt | Similar to SIGTERM, depends on app |

### terminationGracePeriodSeconds

```yaml
spec:
  terminationGracePeriodSeconds: 60   # default is 30
  containers:
  - name: app
    ...
```

**Rules:**
- Total budget = `terminationGracePeriodSeconds`
- If `preStop` hook runs for 20s and grace period is 30s → app only gets 10s after SIGTERM
- If `preStop` hook exceeds grace period → Kubernetes adds 2 extra seconds, then SIGKILL
- Set this to: `max(preStop duration) + max(request drain time) + buffer`

### What SIGTERM Actually Does (Common App Behaviors)

```
nginx:         Closes listening socket, waits for active connections to finish
Java Spring:   Triggers ApplicationContext shutdown hooks, Tomcat graceful shutdown
Node.js:       Must manually handle process.on('SIGTERM', ...) 
Python Flask:  Needs gunicorn with --timeout flag or manual signal handler
Go:            Must use signal.Notify(c, syscall.SIGTERM) pattern
```

### Example: Node.js Graceful Shutdown Handler

```javascript
// MUST implement this — Node.js won't gracefully shut down otherwise
const server = app.listen(3000);

process.on('SIGTERM', () => {
  console.log('SIGTERM received, starting graceful shutdown');
  
  server.close(() => {
    console.log('HTTP server closed');
    // close DB connections, flush caches, etc.
    process.exit(0);
  });
  
  // Force exit after 25s if something hangs (below 30s grace period)
  setTimeout(() => {
    console.error('Forced exit after timeout');
    process.exit(1);
  }, 25000);
});
```

---

## SECTION 3: HOW READINESS PROBES PROTECT TRAFFIC DURING SHUTDOWN

### Readiness vs Liveness Probes

```
┌─────────────────────────────────────────────────────┐
│  LIVENESS probe  → "Is the app alive/healthy?"      │
│  Fails → pod is RESTARTED                           │
│                                                     │
│  READINESS probe → "Is the app ready for traffic?"  │
│  Fails → pod is REMOVED from Service endpoints      │
│  (pod keeps running, just no new traffic)           │
└─────────────────────────────────────────────────────┘
```

### Readiness During Shutdown

When your app starts shutting down (receives SIGTERM), it should:

1. **Immediately fail the readiness probe** → tells kube-proxy to stop routing traffic
2. **Drain existing connections** → wait for in-flight requests to complete
3. **Exit cleanly**

```go
// Example: Go app failing readiness on SIGTERM
var isReady = true

http.HandleFunc("/healthz/ready", func(w http.ResponseWriter, r *http.Request) {
    if !isReady {
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})

signal.Notify(sigChan, syscall.SIGTERM)
go func() {
    <-sigChan
    isReady = false  // fail readiness immediately
    time.Sleep(10 * time.Second)  // drain connections
    os.Exit(0)
}()
```

### Probe Configuration That Works With Shutdown

```yaml
readinessProbe:
  httpGet:
    path: /healthz/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 1    # fail fast — stop traffic immediately on first failure
  successThreshold: 1

livenessProbe:
  httpGet:
    path: /healthz/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3    # more tolerant — don't restart unless truly dead
```

---

## SECTION 4: PODDISRUPTIONBUDGET (PDB)

### What is a PDB?

A PDB is a **cluster-level policy** that limits how many pods of a deployment/statefulset can be simultaneously unavailable during **voluntary disruptions**.

```
Voluntary disruptions (PDB applies):     Involuntary disruptions (PDB does NOT apply):
  - kubectl drain node                     - Node hardware failure
  - Cluster upgrades                       - OOM kill
  - Manual pod deletion                    - Kernel panic
  - Cluster autoscaler scale-down          - Node network partition
```

### PDB Spec: minAvailable vs maxUnavailable

```yaml
# Option 1: minAvailable — at least N pods must stay up
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2          # can be a number OR percentage "50%"
  selector:
    matchLabels:
      app: myapp

# Option 2: maxUnavailable — at most N pods can be down at once
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  maxUnavailable: 1        # can be a number OR percentage "25%"
  selector:
    matchLabels:
      app: myapp
```

### PDB Decision Logic (How Kubernetes Enforces It)

```
SCENARIO: You have 3 replicas, PDB says minAvailable: 2

   Pod-A [Running] ──┐
   Pod-B [Running] ──┼── All 3 healthy. 3 available.
   Pod-C [Running] ──┘

   kubectl drain node-1  (Pod-A is on node-1)
   
   Kubernetes checks: "If I evict Pod-A, will 2 pods remain?"
   3 - 1 = 2 ✓  → eviction ALLOWED

   Pod-A [Terminating]
   Pod-B [Running]   ──┬── 2 available = minAvailable satisfied
   Pod-C [Running]   ──┘

   Now try to drain node-2 (Pod-B is on node-2) at the SAME time:
   
   Kubernetes checks: "If I evict Pod-B, will 2 pods remain?"
   2 - 1 = 1 ✗  → eviction BLOCKED (returns 429 Too Many Requests)
```

### PDB with Percentage (Best for Large Deployments)

```yaml
spec:
  minAvailable: "80%"   # 10 replicas → at least 8 must be available
  maxUnavailable: "20%" # 10 replicas → at most 2 can be down
```

### Unhealthy Pod Eviction Policy (Kubernetes 1.26+)

```yaml
spec:
  minAvailable: 2
  unhealthyPodEvictionPolicy: AlwaysAllow  # default: IfHealthyBudget
  # AlwaysAllow: allows evicting unhealthy pods even if it violates budget
  # IfHealthyBudget: only evict unhealthy pods if budget is satisfied
```

---

## SECTION 5: PRESTOP HOOKS

### What is a preStop Hook?

A `preStop` hook runs **inside the container** before SIGTERM is sent. It blocks the SIGTERM until it completes (or grace period expires).

Two types:
1. **exec** — runs a command in the container
2. **httpGet** — makes an HTTP request (useful for service mesh draining)

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 15"]  # allow iptables to propagate

    # OR

    httpGet:
      path: /drain
      port: 8080
```

### The Classic preStop Sleep Pattern

```yaml
# This is the most common real-world pattern
lifecycle:
  preStop:
    exec:
      command:
        - /bin/sh
        - -c
        - sleep 15

# Why sleep 15?
# → kube-proxy takes ~2-5s to update iptables rules
# → sleep 15 ensures ALL proxies have removed this pod from load balancer
# → THEN container starts shutting down (SIGTERM arrives after preStop)
# → With terminationGracePeriodSeconds: 60, app gets 45s after preStop
```

### nginx Graceful Drain with preStop

```yaml
lifecycle:
  preStop:
    exec:
      command:
        - /bin/sh
        - -c
        - |
          sleep 5
          nginx -s quit
          while kill -0 $(cat /run/nginx.pid 2>/dev/null) 2>/dev/null; do
            sleep 1
          done
```

### Combining preStop + terminationGracePeriodSeconds

```yaml
spec:
  terminationGracePeriodSeconds: 90   # total budget

  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 20"]  # 20s for iptables drain

    # After preStop (20s), app gets SIGTERM
    # App has 90 - 20 = 70s remaining to finish in-flight requests
    # At t=90s from start: SIGKILL
```

---

## SECTION 6: DESIGNING ZERO-DOWNTIME ROLLOUTS

### The Complete Zero-Downtime Recipe

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # can create 1 extra pod above desired (4 total)
      maxUnavailable: 0    # NEVER reduce below desired count during rollout

  template:
    spec:
      terminationGracePeriodSeconds: 60

      containers:
      - name: app
        image: myapp:v2

        # Readiness probe — gates traffic
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3
          failureThreshold: 2

        # Liveness probe — detects deadlocks
        livenessProbe:
          httpGet:
            path: /live
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10

        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]

---
# Companion PDB
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

---

## SECTION 7: PROTECTING STATEFUL APPLICATIONS

### StatefulSet + PDB Pattern

```yaml
# StatefulSet with proper shutdown handling
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 3
  podManagementPolicy: OrderedReady   # shutdown one at a time (default)
  
  template:
    spec:
      terminationGracePeriodSeconds: 120  # DB needs more time to flush WAL

      containers:
      - name: postgres
        lifecycle:
          preStop:
            exec:
              command:
                - /bin/sh
                - -c
                - |
                  # Gracefully stop accepting new connections
                  psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity 
                           WHERE datname = current_database() 
                           AND pid <> pg_backend_pid();"
                  pg_ctl stop -m fast

---
# PDB for StatefulSet — CRITICAL for stateful apps
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
spec:
  minAvailable: 2    # For 3-node cluster, always keep 2 up (quorum)
  selector:
    matchLabels:
      app: postgres
```

---

## HANDS-ON LAB 1: Apply a PodDisruptionBudget

**Goal:** Ensure at least 1 pod stays available during node drain.

### Step 1: Create the Deployment

```bash
kubectl create namespace lab-pdb
kubectl config set-context --current --namespace=lab-pdb
```

```yaml
# File: lab1-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: lab-pdb
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.25
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 3
```

```bash
kubectl apply -f lab1-deployment.yaml
kubectl get pods -w   # wait for all 3 to be Running
```

### Step 2: Apply the PDB

```yaml
# File: lab1-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webapp-pdb
  namespace: lab-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: webapp
```

```bash
kubectl apply -f lab1-pdb.yaml
kubectl get pdb webapp-pdb
```

**Expected output:**
```
NAME         MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
webapp-pdb   1               N/A               2                     10s
```
`ALLOWED DISRUPTIONS = 2` means: you have 3 pods, minimum 1 must stay, so 2 can be disrupted.

### Step 3: Test the PDB with Eviction

```bash
# Get pod names
kubectl get pods -o wide

# Try to manually evict a pod (simulates what drain does)
POD1=$(kubectl get pods -o name | head -1 | cut -d/ -f2)
POD2=$(kubectl get pods -o name | sed -n '2p' | cut -d/ -f2)
POD3=$(kubectl get pods -o name | sed -n '3p' | cut -d/ -f2)

echo "Pod names: $POD1, $POD2, $POD3"
```

```bash
# Evict first pod — should succeed
kubectl delete pod $POD1

# Immediately try to evict second pod while first is terminating
kubectl delete pod $POD2

# Watch PDB status in another terminal
kubectl get pdb webapp-pdb -w
```

### Step 4: Simulate Node Drain (if using kind or minikube with multiple nodes)

```bash
# List nodes
kubectl get nodes

# Cordon + drain a node (simulates maintenance)
NODE=$(kubectl get pods $POD1 -o jsonpath='{.spec.nodeName}')
kubectl cordon $NODE
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data

# Observe: drain will evict pods respecting PDB
# If PDB would be violated, drain blocks and retries
kubectl get pods -w   # watch pods reschedule

# Uncordon when done
kubectl uncordon $NODE
```

### Step 5: Test PDB Blocking

```yaml
# Set aggressive PDB — minAvailable equals replicas (no disruption allowed)
# File: lab1-pdb-strict.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webapp-pdb
  namespace: lab-pdb
spec:
  minAvailable: 3   # ALL pods must be available
  selector:
    matchLabels:
      app: webapp
```

```bash
kubectl apply -f lab1-pdb-strict.yaml
kubectl get pdb webapp-pdb
# ALLOWED DISRUPTIONS should now be 0

# Try to delete any pod — it will be blocked by eviction API
kubectl delete pod $POD1   # This bypasses PDB! (delete != eviction)

# The correct way to test blocking is via eviction API:
cat <<EOF | kubectl create -f -
apiVersion: policy/v1
kind: Eviction
metadata:
  name: $POD1
  namespace: lab-pdb
EOF
# Should return: 429 Too Many Requests (PDB violated)
```

### Step 6: Verify and Clean Up

```bash
kubectl describe pdb webapp-pdb
# Look for: "DisruptionsAllowed", "ExpectedPods", "CurrentHealthy"

kubectl delete namespace lab-pdb
```

---

## HANDS-ON LAB 2: Add a preStop Hook

**Goal:** Gracefully drain traffic before shutting down the container.

### Step 1: Setup

```bash
kubectl create namespace lab-prestop
kubectl config set-context --current --namespace=lab-prestop
```

### Step 2: Deploy App WITHOUT preStop (observe the problem)

```yaml
# File: lab2-no-prestop.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-noprestop
  namespace: lab-prestop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp-noprestop
  template:
    metadata:
      labels:
        app: webapp-noprestop
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-noprestop
  namespace: lab-prestop
spec:
  selector:
    app: webapp-noprestop
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f lab2-no-prestop.yaml
```

### Step 3: Deploy App WITH preStop

```yaml
# File: lab2-with-prestop.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-prestop
  namespace: lab-prestop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp-prestop
  template:
    metadata:
      labels:
        app: webapp-prestop
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 3
          failureThreshold: 1
        lifecycle:
          preStop:
            exec:
              command:
                - /bin/sh
                - -c
                - |
                  echo "preStop: sleeping 15s for iptables propagation"
                  sleep 15
                  echo "preStop: sending nginx quit signal"
                  nginx -s quit
                  echo "preStop: complete"
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-prestop
  namespace: lab-prestop
spec:
  selector:
    app: webapp-prestop
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f lab2-with-prestop.yaml
```

### Step 4: Observe preStop Execution

```bash
# Get pod name
POD=$(kubectl get pods -l app=webapp-prestop -o name | head -1 | cut -d/ -f2)

# Follow logs in one terminal
kubectl logs -f $POD

# In another terminal, delete the pod and watch timing
time kubectl delete pod $POD

# The deletion will take ~15s+ (preStop sleep) before pod is gone
# You'll see the sleep message in logs before termination
```

### Step 5: Measure Termination Time Difference

```bash
# Time pod deletion WITHOUT preStop
POD_NO=$(kubectl get pods -l app=webapp-noprestop -o name | head -1 | cut -d/ -f2)
echo "Deleting pod without preStop..."
time kubectl delete pod $POD_NO
# Expected: ~1-2 seconds

# Time pod deletion WITH preStop
POD_YES=$(kubectl get pods -l app=webapp-prestop -o name | head -1 | cut -d/ -f2)
echo "Deleting pod with preStop..."
time kubectl delete pod $POD_YES
# Expected: ~15-20 seconds (preStop sleep + shutdown time)
```

### Step 6: HTTP-based preStop (Advanced)

```yaml
# For apps that expose a /drain endpoint
lifecycle:
  preStop:
    httpGet:
      path: /admin/drain       # tells app to stop accepting new requests
      port: 9090               # admin port
      scheme: HTTP
```

```bash
# Clean up
kubectl delete namespace lab-prestop
```

---

## HANDS-ON LAB 3: Verify Graceful Termination Sequence

**Goal:** Trigger termination → watch logs → observe probe behavior.

### Step 1: Build a Custom App That Logs Termination Events

```yaml
# File: lab3-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graceful-app
  namespace: lab-grace
spec:
  replicas: 2
  selector:
    matchLabels:
      app: graceful-app
  template:
    metadata:
      labels:
        app: graceful-app
    spec:
      terminationGracePeriodSeconds: 60

      containers:
      - name: app
        # Using a shell script via busybox to simulate a well-behaved app
        image: busybox:1.36
        command:
          - /bin/sh
          - -c
          - |
            echo "[$(date)] Container STARTED"
            
            # Handle SIGTERM
            trap 'echo "[$(date)] SIGTERM received! Starting graceful shutdown..."; \
                  sleep 10; \
                  echo "[$(date)] Graceful shutdown complete. Exiting 0."; \
                  exit 0' TERM
            
            # Main loop
            i=0
            while true; do
              i=$((i+1))
              echo "[$(date)] Heartbeat #$i - still running"
              sleep 5 &
              wait $!
            done

        lifecycle:
          preStop:
            exec:
              command:
                - /bin/sh
                - -c
                - |
                  echo "[$(date)] preStop hook STARTED"
                  sleep 10
                  echo "[$(date)] preStop hook COMPLETE"

        readinessProbe:
          exec:
            command:
              - /bin/sh
              - -c
              - "test -f /tmp/ready || exit 1"
          initialDelaySeconds: 2
          periodSeconds: 3
          failureThreshold: 1

        livenessProbe:
          exec:
            command: ["sh", "-c", "exit 0"]
          initialDelaySeconds: 5
          periodSeconds: 10
```

> **Note:** The above readiness probe uses a file `/tmp/ready`. In a real scenario, create this file at startup: add `touch /tmp/ready` in your startup command.

### Step 2: Updated App With Proper Readiness

```yaml
# File: lab3-full.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab-grace
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graceful-app
  namespace: lab-grace
spec:
  replicas: 3
  selector:
    matchLabels:
      app: graceful-app
  template:
    metadata:
      labels:
        app: graceful-app
    spec:
      terminationGracePeriodSeconds: 60

      containers:
      - name: app
        image: busybox:1.36
        command:
          - /bin/sh
          - -c
          - |
            echo "[$(date '+%H:%M:%S')] === CONTAINER STARTED ==="
            
            # Create readiness flag
            touch /tmp/ready
            echo "[$(date '+%H:%M:%S')] Readiness flag set — pod is READY for traffic"
            
            # SIGTERM handler
            _term() {
              echo "[$(date '+%H:%M:%S')] === SIGTERM RECEIVED ==="
              echo "[$(date '+%H:%M:%S')] Removing readiness flag (failing readiness probe)"
              rm -f /tmp/ready
              echo "[$(date '+%H:%M:%S')] Draining in-flight requests (simulated 15s drain)"
              sleep 15
              echo "[$(date '+%H:%M:%S')] === GRACEFUL SHUTDOWN COMPLETE ==="
              exit 0
            }
            trap _term TERM
            
            # Heartbeat loop
            count=0
            while true; do
              count=$((count + 1))
              echo "[$(date '+%H:%M:%S')] [heartbeat-$count] serving traffic"
              sleep 5 &
              wait $!
            done

        lifecycle:
          preStop:
            exec:
              command:
                - /bin/sh
                - -c
                - |
                  echo "[$(date '+%H:%M:%S')] === PRESTOP HOOK STARTED ==="
                  echo "[$(date '+%H:%M:%S')] Sleeping 10s for iptables propagation"
                  sleep 10
                  echo "[$(date '+%H:%M:%S')] === PRESTOP HOOK COMPLETE ==="

        readinessProbe:
          exec:
            command: ["test", "-f", "/tmp/ready"]
          initialDelaySeconds: 2
          periodSeconds: 3
          failureThreshold: 1

        livenessProbe:
          exec:
            command: ["sh", "-c", "exit 0"]
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: graceful-app-pdb
  namespace: lab-grace
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: graceful-app
```

```bash
kubectl apply -f lab3-full.yaml
kubectl get pods -n lab-grace -w   # wait for Running/Ready
```

### Step 3: Open Multiple Terminals to Observe Everything

```bash
# Terminal 1: Watch pod status
watch kubectl get pods -n lab-grace -o wide

# Terminal 2: Stream logs from all pods
kubectl logs -n lab-grace -l app=graceful-app -f --prefix

# Terminal 3: Watch PDB status
watch kubectl get pdb -n lab-grace

# Terminal 4: Watch endpoints (shows when pod is removed from service)
watch kubectl get endpoints -n lab-grace
```

### Step 4: Trigger Termination and Observe

```bash
# Terminal 5: Trigger pod deletion
POD=$(kubectl get pods -n lab-grace -o name | head -1 | cut -d/ -f2)
echo "Terminating pod: $POD"
kubectl delete pod -n lab-grace $POD
```

### Step 5: What You Should Observe in Logs

```
# Expected log sequence (timestamps will be actual times):
[10:00:00] === CONTAINER STARTED ===
[10:00:00] Readiness flag set — pod is READY for traffic
[10:00:01] [heartbeat-1] serving traffic
...
[10:00:30] === PRESTOP HOOK STARTED ===        ← preStop fires first
[10:00:30] Sleeping 10s for iptables propagation
[10:00:40] === PRESTOP HOOK COMPLETE ===       ← preStop done
[10:00:40] === SIGTERM RECEIVED ===            ← SIGTERM arrives after preStop
[10:00:40] Removing readiness flag (failing readiness probe)
[10:00:40] Draining in-flight requests (simulated 15s drain)
[10:00:55] === GRACEFUL SHUTDOWN COMPLETE ===  ← clean exit

# Total termination time: ~25s (10s preStop + 15s drain)
# Pod's terminationGracePeriodSeconds: 60 → no SIGKILL needed ✓
```

### Step 6: Verify Endpoint Removal Timing

```bash
# Before deletion: pod IP should appear in endpoints
kubectl describe endpoints graceful-app -n lab-grace

# After deletion starts: within ~3-5s, IP should disappear from endpoints
# This confirms kube-proxy stops routing traffic to the terminating pod

# Run a quick endpoint check loop
for i in $(seq 1 30); do
  echo "=== Second $i ==="
  kubectl get endpoints graceful-app -n lab-grace 2>/dev/null || echo "no service"
  sleep 1
done
```

### Step 7: Force SIGKILL Scenario (observing ungraceful shutdown)

```bash
# Delete with grace period of 0 — immediate SIGKILL
kubectl delete pod -n lab-grace $POD --grace-period=0 --force

# Logs will NOT show graceful shutdown message — container is killed immediately
# This is what happens in practice during node failure (involuntary disruption)
```

### Step 8: Clean Up All Labs

```bash
kubectl delete namespace lab-pdb lab-prestop lab-grace 2>/dev/null; echo "Cleaned up"
```

---

## SECTION 8: QUICK REFERENCE CHEAT SHEET

### PDB Cheat Sheet

```bash
# Create PDB
kubectl apply -f pdb.yaml

# View PDB status
kubectl get pdb
kubectl describe pdb <name>

# Key fields to watch:
# - ALLOWED DISRUPTIONS: how many pods can be disrupted right now
# - Expected Pods: how many pods the selector matches
# - Current Healthy: how many are currently healthy

# Force eviction (respects PDB)
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# Check if eviction is blocked
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --dry-run
```

### preStop Cheat Sheet

```yaml
# Minimum viable preStop (prevents connection drops)
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 15"]

# With terminationGracePeriodSeconds
spec:
  terminationGracePeriodSeconds: 60  # total budget
  # preStop uses 15s → SIGTERM fires at t=15s → app has 45s to finish
```

### Graceful Shutdown Cheat Sheet

```bash
# Watch pod termination live
kubectl get pods -w

# Check pod's termination grace period
kubectl get pod <name> -o jsonpath='{.spec.terminationGracePeriodSeconds}'

# Force immediate kill (no grace period)
kubectl delete pod <name> --grace-period=0 --force

# View termination events
kubectl describe pod <name> | grep -A 20 Events
```

---

## SECTION 9: COMMON MISTAKES AND FIXES

| Mistake | Symptom | Fix |
|---------|---------|-----|
| No preStop sleep | Dropped connections during rollout | Add `sleep 15` preStop |
| terminationGracePeriodSeconds too low | SIGKILL before drain completes | Increase to 60-120s |
| App doesn't handle SIGTERM | Pod killed after full grace period | Add SIGTERM handler in app code |
| PDB minAvailable = replicas | node drain hangs forever | Use `minAvailable: replicas - 1` |
| maxUnavailable: 0 without surge | Rolling update hangs | Add `maxSurge: 1` with `maxUnavailable: 0` |
| Liveness probe kills starting pod | CrashLoopBackOff on slow startup | Use `startupProbe` or increase `initialDelaySeconds` |
| preStop exceeds grace period | SIGKILL sent after grace + 2s | Ensure `preStop_duration + drain_time < terminationGracePeriodSeconds` |

---

## SECTION 10: INFRATHRONE MINDSET NOTES

```
╔══════════════════════════════════════════════════════════════════╗
║  INFRATHRONE MINDSET                                             ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  PDB protects AVAILABILITY, not PERFORMANCE.                     ║
║  → A pod behind a slow node still counts as "available"          ║
║  → Use HPA + resource limits to protect performance              ║
║                                                                  ║
║  Graceful shutdown is part of SRE GOLDEN SIGNALS.               ║
║  → Latency: shutdown should NOT spike p99 latency               ║
║  → Errors: zero 5xx during pod termination is the goal           ║
║  → Traffic: readiness probe removal is your traffic gate         ║
║  → Saturation: drain period protects in-flight request budget    ║
║                                                                  ║
║  Zero-downtime is a CONTRACT between:                            ║
║  → Your app (handles SIGTERM, fails readiness on shutdown)       ║
║  → Your deployment (maxUnavailable: 0, maxSurge: 1)             ║
║  → Your infrastructure (PDB, terminationGracePeriodSeconds)      ║
║                                                                  ║
║  THINK IN LAYERS:                                                ║
║  Layer 1: App code      → SIGTERM handler                        ║
║  Layer 2: Container     → preStop hook                           ║
║  Layer 3: Pod spec      → terminationGracePeriodSeconds          ║
║  Layer 4: Deployment    → maxUnavailable: 0                      ║
║  Layer 5: Cluster       → PodDisruptionBudget                    ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## KNOWLEDGE CHECK

Answer these before moving on:

1. **What is the race condition during pod termination and how do you fix it?**
   > SIGTERM and endpoint removal happen in parallel. Fix: preStop sleep ≥ 5-15s.

2. **If your app has a preStop hook that takes 25s and terminationGracePeriodSeconds is 30s, how long does the app have after SIGTERM?**
   > Only 5 seconds. Set terminationGracePeriodSeconds higher, e.g., 60s.

3. **Does a PDB protect against a node hardware failure?**
   > No. PDB only applies to voluntary disruptions (drain, eviction, upgrades).

4. **What does `ALLOWED DISRUPTIONS: 0` in kubectl get pdb output mean?**
   > No pods can be evicted right now without violating the budget (e.g., one is already down).

5. **Why should the readiness probe fail immediately on SIGTERM?**
   > So kube-proxy stops routing new requests to the pod while it finishes existing ones.

6. **What is the difference between kubectl delete pod and eviction?**
   > `delete pod` bypasses PDB. Eviction API (used by kubectl drain) respects PDB.

---

*InfraThrone Study Notes | Monday: PDBs, preStop Hooks & Graceful Shutdown*

