# WEDNESDAY: Chaos Engineering Fundamentals
## Study Notes + Practice Guide

---

## 1. WHAT CHAOS ENGINEERING IS (AND IS NOT)

### What It IS
Chaos Engineering is the **discipline of experimenting on a system** in order to build confidence in the system's capability to withstand turbulent and unexpected conditions in production.

It follows the **scientific method**:
1. Define a steady state (normal behavior baseline)
2. Hypothesize that steady state will continue in both control and experimental groups
3. Introduce variables that reflect real-world events (pod crashes, network latency, disk full, etc.)
4. Try to disprove the hypothesis

### What It IS NOT
| NOT Chaos Engineering | Why |
|---|---|
| Random destruction ("break things for fun") | Must be hypothesis-driven and observable |
| A blame game | It's a learning exercise, not a failure hunt |
| Done only in production | Start in staging/dev, earn the right to prod |
| A one-time event | Continuous practice, automated over time |
| Penetration testing | Chaos targets resilience, not security vulnerabilities |

### Core Principles (from Principlesofchaos.org)
- **Build a Hypothesis around Steady State Behavior** — not about system internals
- **Vary Real-world Events** — simulate realistic failures
- **Run Experiments in Production** — only after validating in lower environments
- **Automate Experiments to Run Continuously**
- **Minimize Blast Radius** — limit the impact scope, always have a kill switch

---

## 2. FAILURE HYPOTHESIS

A failure hypothesis is a **structured prediction** about what your system will do when a specific failure is injected.

### Template
```
GIVEN:  [system is in steady state, e.g., all pods Running, p99 latency < 200ms]
WHEN:   [failure is injected, e.g., one pod in Deployment X is killed]
THEN:   [expected outcome, e.g., Kubernetes reschedules within 30s, no user-visible error]
BECAUSE: [reason, e.g., Deployment has minAvailable=1 PodDisruptionBudget]
```

### Example Hypotheses
```
GIVEN:  mercedes-integrations has 3 replicas, HPA enabled, readinessProbe configured
WHEN:   1 pod is killed randomly
THEN:   Traffic shifts to remaining 2 pods within 10s, new pod starts within 60s
BECAUSE: Service load balances, Deployment controller restarts the pod

GIVEN:  Downstream service call with 5s timeout configured
WHEN:   200ms network latency is injected on egress
THEN:   Calls succeed, circuit breaker NOT triggered (threshold not crossed)
BECAUSE: 200ms < 5s timeout, retry logic absorbs transient delay

GIVEN:  A worker node hosts 3 application pods
WHEN:   Node is cordoned and drained (node kill simulation)
THEN:   All 3 pods are rescheduled to other nodes within 120s
BECAUSE: Kubernetes scheduler finds capacity on remaining nodes
```

---

## 3. KUBERNETES CHAOS TOOLING

### Tool Comparison
| Tool | Type | Strength | Best For |
|---|---|---|---|
| **Chaos Mesh** | Kubernetes-native CRD | Rich fault types, Grafana integration | Production chaos at scale |
| **LitmusChaos** | Workflow-based | GitOps native, Hub of experiments | CI/CD integrated chaos |
| **Gremlin** | SaaS + Agent | Enterprise support, safe mode | Teams new to chaos |
| **Chaos Toolkit** | CLI + Python | Lightweight, scriptable | Custom scenarios |
| `kubectl delete pod` | Manual | Zero setup | Learning, quick tests |

### Chaos Mesh Architecture
```
                    ┌─────────────────────────┐
                    │    Chaos Mesh Dashboard  │
                    └────────────┬────────────┘
                                 │ REST API
                    ┌────────────▼────────────┐
                    │   Chaos Controller Mgr   │  ← Watches CRDs
                    └────────────┬────────────┘
                    ┌────────────▼────────────┐
                    │    Chaos Daemon (DaemonSet)│ ← Runs on each node
                    └─────────────────────────┘
                           Injects faults via
                    ├── tc (traffic control)
                    ├── iptables (network)
                    └── kill (process)
```

### LitmusChaos Architecture
```
LitmusChaos Hub (experiment library)
        │
        ▼
ChaosEngine (CR)  →  ChaosExperiment (CR)  →  ChaosResult (CR)
        │
        ▼
  Chaos Runner Pod
        │
        ▼
  Target Pod / Node
```

---

## 4. FAULT TYPES: POD KILL, NODE KILL, NETWORK LATENCY

### 4.1 Pod Kill

**What it tests:** Deployment controller recovery, readiness probe correctness, PDB (PodDisruptionBudget) enforcement, service traffic rerouting

**Manual (no tooling needed):**
```bash
# Kill a specific pod
kubectl delete pod <pod-name> -n <namespace>

# Kill a random pod from a deployment
kubectl delete pod $(kubectl get pods -n mercedes -l app=mercedes-integrations \
  -o jsonpath='{.items[0].metadata.name}') -n mercedes

# Watch recovery in real time
kubectl get pods -n mercedes -w
```

**Chaos Mesh YAML:**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-mercedes
  namespace: chaos-testing
spec:
  action: pod-kill
  mode: one                      # kill one pod at a time
  selector:
    namespaces:
      - mercedes
    labelSelectors:
      app: mercedes-integrations
  scheduler:
    cron: "@every 5m"            # repeat every 5 min
```

**LitmusChaos YAML:**
```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: pod-delete-engine
  namespace: mercedes
spec:
  appinfo:
    appns: mercedes
    applabel: "app=mercedes-integrations"
    appkind: deployment
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"          # inject for 60 seconds
            - name: CHAOS_INTERVAL
              value: "10"          # kill every 10 seconds
            - name: FORCE
              value: "false"
```

**What to observe:**
- Pod count drops then recovers
- No 5xx errors in application logs during recovery
- New pod passes readiness probe before receiving traffic
- HPA does not scale unnecessarily

---

### 4.2 Node Kill (Drain / Cordon Simulation)

**What it tests:** Pod rescheduling across nodes, PDB enforcement, cluster capacity, PersistentVolume reattachment

**Manual simulation:**
```bash
# Step 1: Cordon the node (no new pods scheduled here)
kubectl cordon <node-name>

# Step 2: Drain the node (evict all pods safely)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --force

# Step 3: Watch pods reschedule
kubectl get pods -A -o wide -w

# Step 4: Uncordon to restore
kubectl uncordon <node-name>
```

**Chaos Mesh YAML:**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PhysicalMachineChaos
metadata:
  name: node-kill-experiment
spec:
  action: shutdown
  address:
    - "192.168.1.10:31768"   # physical machine chaos daemon address
  duration: "30s"
```

**What to observe:**
- `kubectl get nodes` shows NotReady
- Pods on that node show Terminating → appear on other nodes
- Application traffic reroutes (check ingress logs)
- No data loss if StatefulSets use proper PVCs

---

### 4.3 Network Latency Injection

**What it tests:** Timeout configurations, circuit breaker thresholds, retry logic, user experience degradation

**Manual using `tc` (Linux traffic control):**
```bash
# SSH into pod and add 500ms delay
kubectl exec -it <pod-name> -n mercedes -- sh
tc qdisc add dev eth0 root netem delay 500ms

# Verify it's applied
tc qdisc show dev eth0

# Remove the delay
tc qdisc del dev eth0 root
```

**Chaos Mesh YAML:**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-mercedes
  namespace: chaos-testing
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - mercedes
    labelSelectors:
      app: mercedes-integrations
  delay:
    latency: "200ms"
    correlation: "25"            # 25% correlation between delays
    jitter: "50ms"               # ±50ms variation
  direction: to                  # inject on ingress traffic
  duration: "5m"
```

**LitmusChaos YAML:**
```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: pod-network-latency-engine
  namespace: mercedes
spec:
  appinfo:
    appns: mercedes
    applabel: "app=mercedes-integrations"
    appkind: deployment
  experiments:
    - name: pod-network-latency
      spec:
        components:
          env:
            - name: NETWORK_LATENCY
              value: "2000"        # 2000ms = 2s latency
            - name: TOTAL_CHAOS_DURATION
              value: "120"
            - name: TARGET_CONTAINER
              value: "mercedes-integrations"
```

**What to observe:**
- Response times increase in Prometheus/Grafana
- Circuit breaker opens if latency exceeds threshold
- Timeout errors appear in logs (expect them — validate they are handled gracefully)
- Retries are logged and eventually succeed or fail fast

---

## 5. OBSERVABILITY VALIDATION DURING CHAOS

Chaos without observability is just destruction. You must instrument your system to **see** what is happening.

### The Observability Trinity During Chaos

```
           ┌──────────┐
           │  METRICS  │  ← Prometheus: p99 latency, error rate, pod count
           └────┬─────┘
                │
   ┌────────────┼────────────┐
   │                         │
┌──▼──────┐           ┌──────▼────┐
│  LOGS   │           │  TRACES   │
│(Loki/   │           │(Jaeger/   │
│ EFK)    │           │ Zipkin)   │
└─────────┘           └───────────┘
```

### Key Metrics to Watch (Prometheus Queries)

```promql
# Pod availability during chaos
kube_deployment_status_replicas_available{deployment="mercedes-integrations"}

# Error rate spike detection
rate(http_requests_total{status=~"5.."}[1m])

# P99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Pod restart count
kube_pod_container_status_restarts_total{namespace="mercedes"}

# CPU/Memory pressure during chaos
container_cpu_usage_seconds_total{namespace="mercedes"}
```

### Log Signals to Watch
```bash
# Watch live logs during chaos
kubectl logs -f deployment/mercedes-integrations -n mercedes --all-containers

# Search for error patterns post-chaos
kubectl logs deployment/mercedes-integrations -n mercedes | grep -E "ERROR|WARN|timeout|circuit"

# Check readiness probe failures
kubectl describe pod <pod-name> -n mercedes | grep -A5 "Readiness"
kubectl describe pod <pod-name> -n mercedes | grep "Unhealthy\|BackOff"
```

### Events to Monitor
```bash
# Watch Kubernetes events during chaos
kubectl get events -n mercedes --sort-by='.lastTimestamp' -w

# Node events
kubectl describe node <node-name> | grep -A20 Events
```

### Grafana Dashboard Checklist During Chaos
- [ ] Error rate stays below SLO threshold
- [ ] P99 latency does not exceed timeout config
- [ ] Pod count recovers to desired replica count
- [ ] No persistent alert firing after chaos ends
- [ ] Memory/CPU does not spike permanently

---

## 6. RESILIENCE PATTERNS TO UNDERSTAND

### Circuit Breaker
```
Normal → [Request] → Service
Degraded → [Request] → OPEN circuit → Fallback response (no cascading failure)
Recovered → Circuit closes after probe succeeds
```
**In Spring Boot (Resilience4j):**
```java
@CircuitBreaker(name = "mercedesBackend", fallbackMethod = "fallback")
public ResponseEntity<?> callDownstream() { ... }
```

### Retry with Exponential Backoff
```
Attempt 1 → fail → wait 1s
Attempt 2 → fail → wait 2s
Attempt 3 → fail → wait 4s → give up → return error
```

### Bulkhead
Limit concurrent calls to a downstream service to prevent one slow service from consuming all threads.

### Readiness vs Liveness Probes (Critical for Chaos)
```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3        # 3 failures → remove from Service endpoints

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3        # 3 failures → restart the container
```

### Pod Disruption Budget (PDB)
Ensures minimum availability during voluntary disruptions (node drains, chaos):
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mercedes-pdb
  namespace: mercedes
spec:
  minAvailable: 2            # at least 2 pods must be running
  selector:
    matchLabels:
      app: mercedes-integrations
```

---

## LAB 1: Pod Kill Experiment

### Objective
Delete random pods and observe automatic recovery, readiness validation, and traffic continuity.

### Prerequisites
```bash
# Ensure you have at least 2 replicas
kubectl scale deployment mercedes-integrations --replicas=3 -n mercedes

# Confirm all pods are Running and Ready
kubectl get pods -n mercedes -l app=mercedes-integrations
```

### Experiment Steps

**Terminal 1 — Continuous monitoring:**
```bash
watch -n 2 kubectl get pods -n mercedes -l app=mercedes-integrations
```

**Terminal 2 — Kill a pod:**
```bash
# Get pod name
POD=$(kubectl get pods -n mercedes -l app=mercedes-integrations \
  -o jsonpath='{.items[0].metadata.name}')

echo "Killing pod: $POD"
kubectl delete pod $POD -n mercedes

# Record the time
echo "Pod killed at: $(date)"
```

**Terminal 3 — Watch Kubernetes events:**
```bash
kubectl get events -n mercedes --sort-by='.lastTimestamp' -w
```

**Terminal 4 — Simulate load during chaos:**
```bash
# Basic load test using curl loop
while true; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://<app-url>/health)
  echo "$(date +%H:%M:%S) - HTTP $STATUS"
  sleep 1
done
```

### Validation Checklist
- [ ] Time from pod deletion to new pod Running: _____ seconds
- [ ] Time from pod deletion to new pod Ready: _____ seconds
- [ ] Any 5xx errors observed in Terminal 4: Y / N
- [ ] Kubernetes Events show: Killing → Pulling → Started → Ready
- [ ] Final pod count matches desired replicas: Y / N

### Expected Timeline
```
T+0s:   Pod deleted
T+1s:   Endpoint removed from Service (readiness fails → traffic shifts)
T+5s:   Controller creates replacement pod
T+20s:  New pod's container starts
T+30s:  Readiness probe passes → pod added back to Service
T+60s:  All metrics return to steady state
```

---

## LAB 2: Network Latency Injection (Controlled)

### Objective
Add artificial network delay and observe how timeout configurations and circuit breakers respond.

### Option A: Manual with kubectl exec + tc

```bash
# Step 1: Get a target pod
POD=$(kubectl get pods -n mercedes -l app=mercedes-integrations \
  -o jsonpath='{.items[0].metadata.name}')

# Step 2: Inject 500ms latency on all outbound traffic
kubectl exec -it $POD -n mercedes -- sh -c \
  "tc qdisc add dev eth0 root netem delay 500ms 50ms"

# Step 3: Immediately start monitoring
kubectl logs -f $POD -n mercedes &

# Step 4: Send test requests
for i in {1..10}; do
  time curl -s http://<app-url>/api/endpoint
  sleep 2
done

# Step 5: Remove latency (cleanup)
kubectl exec -it $POD -n mercedes -- sh -c \
  "tc qdisc del dev eth0 root"
```

### Option B: Chaos Mesh NetworkChaos

```bash
# Install Chaos Mesh (dev cluster)
curl -sSL https://mirrors.chaos-mesh.org/v2.6.2/install.sh | bash -s -- --local kind

# Apply the experiment
cat <<EOF | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: lab2-latency
  namespace: mercedes
spec:
  action: delay
  mode: one
  selector:
    namespaces: [mercedes]
    labelSelectors:
      app: mercedes-integrations
  delay:
    latency: "500ms"
    jitter: "100ms"
  duration: "3m"
EOF

# Monitor
kubectl get networkchaos -n mercedes
kubectl describe networkchaos lab2-latency -n mercedes
```

### Validation Checklist
- [ ] Application response time increased to ~500ms+: Y / N
- [ ] Circuit breaker triggered (if configured): Y / N
- [ ] Timeout errors in logs match expected behavior: Y / N
- [ ] After chaos removed, latency returned to normal: Y / N
- [ ] No permanent degradation after experiment: Y / N

### Timeout Behavior Matrix
| Injected Latency | App Timeout | Expected Behavior |
|---|---|---|
| 200ms | 5s | ✅ Success — under threshold |
| 1s | 5s | ✅ Success — slow but works |
| 6s | 5s | ❌ Timeout — circuit breaker should activate |
| 6s | 5s (with retry) | ⚠️ Retry → fail → fallback |

---

## LAB 3: Validate Recovery Signals

### Objective
After chaos, validate that the system fully recovers by checking readiness, logs, and scaling behavior.

### Recovery Validation Script
```bash
#!/bin/bash
NAMESPACE="mercedes"
DEPLOYMENT="mercedes-integrations"
DESIRED_REPLICAS=3

echo "=== RECOVERY VALIDATION ==="
echo ""

# 1. Check pod readiness
echo "--- Pod Readiness ---"
kubectl get pods -n $NAMESPACE -l app=$DEPLOYMENT \
  -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,READY:.status.containerStatuses[0].ready'
echo ""

# 2. Check deployment rollout status
echo "--- Deployment Status ---"
kubectl rollout status deployment/$DEPLOYMENT -n $NAMESPACE
echo ""

# 3. Check replica count
AVAILABLE=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
  -o jsonpath='{.status.availableReplicas}')
echo "Available replicas: $AVAILABLE / $DESIRED_REPLICAS"
if [ "$AVAILABLE" -eq "$DESIRED_REPLICAS" ]; then
  echo "✅ Replica count: RECOVERED"
else
  echo "❌ Replica count: NOT YET RECOVERED"
fi
echo ""

# 4. Check for recent restart loops
echo "--- Restart Counts ---"
kubectl get pods -n $NAMESPACE -l app=$DEPLOYMENT \
  -o custom-columns='NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount'
echo ""

# 5. Check recent events for warnings
echo "--- Recent Warning Events ---"
kubectl get events -n $NAMESPACE --field-selector type=Warning \
  --sort-by='.lastTimestamp' | tail -10
echo ""

# 6. Check HPA (if configured)
echo "--- HPA Status ---"
kubectl get hpa -n $NAMESPACE 2>/dev/null || echo "No HPA configured"
echo ""

echo "=== VALIDATION COMPLETE ==="
```

### Recovery Signal Definitions
| Signal | Command | Healthy State |
|---|---|---|
| Pod readiness | `kubectl get pods` | All pods `Running` + `1/1 READY` |
| Endpoint registered | `kubectl get endpoints` | Pod IPs listed under service |
| No restart loop | `kubectl get pods` | `RESTARTS` count = 0 or stable |
| HPA not scaling | `kubectl get hpa` | `CURRENT` replicas = `DESIRED` |
| No Warning events | `kubectl get events` | No `BackOff` or `Unhealthy` |
| App health endpoint | `curl /actuator/health` | `{"status": "UP"}` |

### Log Recovery Indicators
```bash
# Good: Startup complete
kubectl logs -n mercedes -l app=mercedes-integrations | \
  grep -E "Started|Initialized|Ready to serve"

# Bad: Still in error state
kubectl logs -n mercedes -l app=mercedes-integrations | \
  grep -E "ERROR|Exception|refused|timeout" | tail -20

# Check readiness probe passes
kubectl describe pod <pod> -n mercedes | grep -E "Readiness|Liveness" -A5
```

---

## 7. INFRA THRONE MINDSET NOTES

### Chaos Must Be: Safe, Predictable, Observable

| Principle | Implementation |
|---|---|
| **Safe** | Always have a kill switch. Use `kubectl delete` on the chaos CRD. Set `duration` field. |
| **Predictable** | Write hypothesis BEFORE running. Never run chaos without a defined expected outcome. |
| **Observable** | Open Grafana BEFORE injecting. Have `kubectl logs -f` running. Set up alerts. |

### Blast Radius Strategy
```
Start Small ──────────────────────────────────────────► Expand Gradually

Single Pod      →    Deployment    →    Namespace    →    Cluster-wide
One API route   →    One service   →    Entire stack →    Multi-region
Dev env         →    Staging       →    Canary prod  →    Full prod
```

**Never skip levels.** If a pod kill breaks staging, fix it before going to prod.

### Chaos Runbook Template
```markdown
## Chaos Experiment: [Name]
**Date:** 
**Engineer:**
**Environment:** Dev / Staging / Prod

### Hypothesis
GIVEN:
WHEN:
THEN:

### Abort Criteria (stop if):
- Error rate > 5% for > 2 minutes
- Pods not recovering within 5 minutes
- On-call engineer not available to monitor

### Steps
1. Open monitoring (Grafana dashboard URL: ___)
2. Notify team in Slack: #chaos-experiments
3. Apply chaos manifest
4. Observe for ___ minutes
5. Remove chaos
6. Validate recovery using Lab 3 script

### Results
- Hypothesis confirmed: Y / N
- Unexpected findings:
- Action items:
```

---

## 8. QUICK REFERENCE CHEAT SHEET

### Install Chaos Mesh (local/kind cluster)
```bash
kubectl create ns chaos-testing
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-testing \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

### Install LitmusChaos
```bash
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-operator-v3.0.0.yaml
kubectl get pods -n litmus
```

### Stop/Pause Chaos (Emergency Kill Switch)
```bash
# Chaos Mesh - pause experiment
kubectl annotate podchaos pod-kill-mercedes \
  chaos-mesh.org/pause=true -n chaos-testing

# Delete experiment entirely
kubectl delete podchaos pod-kill-mercedes -n chaos-testing

# LitmusChaos - stop engine
kubectl patch chaosengine pod-delete-engine -n mercedes \
  -p '{"spec":{"engineState":"stop"}}' --type merge
```

### Verify No Chaos Running
```bash
kubectl get podchaos,networkchaos,iochaos -A
kubectl get chaosengine -A
```

---

## 9. KEY TERMS GLOSSARY

| Term | Definition |
|---|---|
| **Steady State** | The normal, measurable behavior of a system (e.g., p99 < 200ms, 0 errors) |
| **Blast Radius** | The scope of impact of a chaos experiment |
| **Game Day** | A scheduled chaos event where a team runs experiments together |
| **Fault Injection** | Deliberately introducing a failure (latency, packet loss, pod kill) |
| **Observability** | Ability to understand system state from its outputs (logs, metrics, traces) |
| **PDB** | PodDisruptionBudget — limits voluntary pod disruptions |
| **Circuit Breaker** | Pattern that stops calling a failing service to prevent cascade failure |
| **Bulkhead** | Isolates failures to one component, like compartments in a ship |
| **MTTR** | Mean Time To Recovery — key metric improved by chaos engineering |
| **SLO** | Service Level Objective — the target your system must meet during chaos |

---

## 10. STUDY RESOURCES

| Resource | URL / Location |
|---|---|
| Principles of Chaos Engineering | https://principlesofchaos.org |
| Chaos Mesh Docs | https://chaos-mesh.org/docs |
| LitmusChaos Docs | https://docs.litmuschaos.io |
| Gremlin Chaos Guide | https://www.gremlin.com/community/tutorials |
| Resilience4j Docs | https://resilience4j.readme.io |
| Netflix Chaos Monkey | https://netflix.github.io/chaosmonkey |

---

## SELF-TEST QUESTIONS

1. What is the difference between a **liveness** probe and a **readiness** probe during a pod kill experiment?
2. What happens if a node is drained but no PDB is configured?
3. At what injected latency would a 3-second configured timeout circuit breaker open?
4. Why should chaos experiments define an **abort criterion** before starting?
5. What Prometheus query would detect a spike in pod restarts?
6. How does a **PodDisruptionBudget** protect against accidental downtime during node drains?
7. What is the difference between `action: pod-kill` and `action: pod-failure` in Chaos Mesh?
8. Name three **recovery signals** you check after a chaos experiment ends.
9. What does `correlation: "25"` mean in a NetworkChaos latency spec?
10. Why is chaos in production only appropriate after passing staging experiments?

---
*Study Notes — Wednesday: Chaos Engineering Fundamentals*
*InfraThrone Study Plan*
