# 🚀 Complete Deployment Notes (DevOps & Kubernetes)

> **Comprehensive study notes covering everything discussed so far — interview-ready and production-focused.**

---

# 📘 TABLE OF CONTENTS

1. Deployment Basics
2. Kubernetes Deployment Strategies
3. Rolling Update (Detailed)
4. Blue–Green Deployment
5. Canary Deployment (Percentage Traffic)
6. Strategy Comparison Table
7. Deployment Decision Flowchart
8. Production Deployment Checklist
9. Real Production Outage Case Study
10. Common Deployment Failures
11. Debugging & Rollback Commands
12. Interview Power Statements

---

# 1️⃣ DEPLOYMENT BASICS

## What is Deployment?

Deployment is the process of:
- Releasing application code
- Running it in production
- Managing updates safely
- Ensuring availability and rollback

---

## Kubernetes Deployment Responsibilities

- Pod lifecycle management
- Scaling replicas
- Rolling updates
- Self-healing
- Version history

---

# 2️⃣ KUBERNETES DEPLOYMENT STRATEGIES

| Strategy | Description |
|------|------|
| Rolling Update | Gradually replaces old pods |
| Blue–Green | Two environments with instant switch |
| Canary | Gradual percentage-based rollout |

---

# 3️⃣ ROLLING UPDATE DEPLOYMENT (DETAILED)

## Default Behavior

```yaml
strategy:
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%
```

Meaning:
- Some old pods removed
- Some new pods added
- Both versions run together

---

## Rolling Update Flow

```
Old Pods (v1)
   ↓
Create New Pod (v2)
   ↓
Delete Old Pod (v1)
   ↓
Repeat until complete
```

---

## Common Problem

- New pods receive traffic immediately
- Application not ready
- Users face intermittent failures

Reason:

> **Running ≠ Ready**

---

## Critical Fixes

### Readiness Probe

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 20
```

### Liveness Probe

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 60
```

---

## Safe Rolling Strategy

```yaml
strategy:
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

---

# 4️⃣ BLUE–GREEN DEPLOYMENT

## Concept

Two identical environments:

- **Blue** → current production
- **Green** → new version

Traffic is switched using Service selector.

---

## Flow

```
User → Service → Blue Pods (v1)

After switch:

User → Service → Green Pods (v2)
```

---

## Advantages

- Zero downtime
- Instant rollback
- Very simple

## Disadvantages

- Double infrastructure cost
- No percentage rollout

---

# 5️⃣ CANARY DEPLOYMENT (PERCENTAGE TRAFFIC)

## Concept

Deploy new version to small percentage of users first.

Example:

```
90% → v1
10% → v2
```

---

## Canary Flow

```
5% → 15% → 30% → 50% → 100%
```

---

## Traffic Control Tools

- NGINX Ingress
- Istio / Service Mesh
- AWS ALB

(Kubernetes Service alone cannot split traffic.)

---

## Canary Advantages

- Lowest risk
- Real-user testing
- Metrics-driven rollout
- Ideal for fintech & payments

---

# 6️⃣ DEPLOYMENT STRATEGY COMPARISON

| Feature | Rolling | Blue–Green | Canary |
|------|------|------|------|
| Downtime | Possible | None | None |
| Rollback | Medium | Instant | Instant |
| Traffic Control | ❌ | 100% | % based |
| Risk | Medium | Low | Lowest |
| Cost | Low | High | Medium |
| Complexity | Easy | Medium | High |

---

# 7️⃣ DEPLOYMENT DECISION FLOWCHART

```
START
 │
 ├── Is Production?
 │     ├── NO → Rolling
 │     └── YES
 │          ├── Downtime allowed?
 │          │      ├── YES → Rolling
 │          │      └── NO
 │          │            ├── Instant rollback needed?
 │          │            │      ├── YES → Blue–Green
 │          │            │      └── NO
 │          │            │            ├── High risk release?
 │          │            │            │      ├── YES → Canary
 │          │            │            │      └── NO → Rolling
```

---

# 8️⃣ PRODUCTION DEPLOYMENT CHECKLIST

## ✅ Code & Build
- CI pipeline green
- Version tagged
- No hardcoded secrets

---

## ✅ Docker Image
- No `latest` tag
- Non-root user
- Vulnerability scanned

---

## ✅ Kubernetes YAML
- Resource requests defined
- Resource limits defined
- Correct namespace
- Labels & selectors match

---

## ✅ Health Probes

- Readiness probe
- Liveness probe
- Startup probe (if slow app)

---

## ✅ Configuration

- ConfigMaps exist
- Secrets exist
- Environment variables verified

---

## ✅ Deployment Strategy

- Rolling / Blue–Green / Canary selected
- Rollback plan ready

---

## ✅ Capacity Planning

- Node resources available
- HPA configured

---

## ✅ Monitoring

- Prometheus
- Grafana dashboards
- Alerts enabled

---

## ✅ Rollback Readiness

```bash
kubectl rollout undo deployment app
```

---

# 9️⃣ REAL PRODUCTION OUTAGE CASE STUDY

## Incident Summary

- FinTech company
- Salary-day traffic
- Rolling update used

---

## What Changed

- DB connection pool reduced
- Health endpoint still returned 200

---

## What Happened

- Gradual rollout replaced all pods
- DB exhausted
- APIs slowed to 20 seconds
- Payments failed

---

## Impact

- 37 minutes outage
- ₹8 crore transaction failure

---

## Root Cause

- No canary deployment
- Health check not business-aware
- No error-rate gating

---

## Fix Implemented

- Mandatory canary rollout
- Metrics-based promotion
- Deep health checks

---

# 🔟 COMMON DEPLOYMENT FAILURES

| Issue | Cause |
|------|------|
| CrashLoopBackOff | Missing env / wrong command |
| Random errors | No readiness probe |
| OOMKilled | No memory limit |
| Full outage | Rolling update misuse |
| Rollback failed | Image overwritten |

---

# 1️⃣1️⃣ DEBUGGING COMMANDS

```bash
kubectl get deployments
kubectl rollout status deployment app
kubectl rollout undo deployment app
kubectl describe pod pod-name
kubectl logs pod-name
kubectl logs pod-name --previous
kubectl exec -it pod-name -- sh
```

---

# 1️⃣2️⃣ INTERVIEW POWER STATEMENTS

> “Running does not mean ready. Readiness probes protect users.”

> “If rollback cannot be done in 30 seconds, deployment is unsafe.”

> “Rolling ensures availability, blue-green ensures rollback, canary ensures production safety.”

---

# ✅ FINAL SUMMARY

You now understand:

✔ All deployment strategies
✔ Real YAML patterns
✔ When to use which strategy
✔ Why outages happen
✔ How senior DevOps engineers design deployments

---

📌 **This markdown file can be directly used as DevOps interview notes or converted into PDF.**

---

🚀 **End of Complete Deployment Notes**

