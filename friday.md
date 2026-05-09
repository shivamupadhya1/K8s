# FRIDAY: Runbooks, Escalation Chains & Failover Drills
### Study Notes for DevOps Engineers (2 Years Experience)

---

## WHY THIS MATTERS (Read This First)

It's 3 AM. Production is down. Your lead is asleep. The customer is screaming.

What do you do?

If you have a **runbook** → you follow it step by step and fix it.
If you don't → you panic, you guess, you make it worse.

A runbook is your **emergency instruction manual**. An escalation chain tells you **who to call when you're stuck**. Failover drills make sure **your plan actually works before the crisis hits**.

This is the difference between an SRE and someone who just "keeps the lights on."

---

## SECTION 1: Core Concepts (Easy Language)

---

### 1.1 What is a Runbook?

A **runbook** is a written, step-by-step guide that tells you exactly what to do in a specific situation — like an incident, a deployment, or a recovery procedure.

Think of it like a **pilot's checklist**. Pilots don't rely on memory during an emergency. They follow a checklist. You should too.

**Two types of runbooks:**

| Type | Purpose | Example |
|---|---|---|
| **Operational Runbook** | Routine tasks | How to deploy, how to rotate secrets |
| **Incident Runbook** | Emergency response | What to do when the DB is down, pods are crashing |

**A good runbook has:**
1. **Trigger/Symptoms** — How do I know this runbook applies?
2. **Severity** — How bad is this? P1 / P2 / P3?
3. **Diagnostic Steps** — How do I confirm what's wrong?
4. **Remediation Steps** — How do I fix it?
5. **Verification Steps** — How do I confirm it's fixed?
6. **Escalation** — Who do I call if I'm stuck?
7. **Rollback** — How do I undo if my fix makes it worse?

**A bad runbook:**
- "Restart the service if it's down." ← Too vague. Which service? How? What if restart fails?

**A good runbook:**
- "If alert `pod-crashloopbackoff` fires on namespace `payments`, run Step 1 → Step 2 → Step 3..."

---

### 1.2 Why SREs Live by Runbooks

Google's SRE philosophy: **"Hope is not a strategy."**

When an incident happens:
- You're under stress → memory fails you.
- Multiple people are involved → confusion without a common procedure.
- Every second of downtime = money lost.

A tested runbook means:
- **Any team member** (not just the expert) can handle it.
- **Recovery time is predictable** (you've measured it before).
- **Nothing gets missed** (no "oh I forgot to check that").

> SRE Rule: **If you solve the same incident twice without writing a runbook, that's a process failure.**

---

### 1.3 RTO vs RPO (Critical Concepts)

These two terms come up in every DR (Disaster Recovery) conversation. Know them cold.

#### RTO — Recovery Time Objective
**"How long can we be down?"**

> Example: RTO = 4 hours → If the system goes down, you have 4 hours to bring it back before it violates SLA.

#### RPO — Recovery Point Objective
**"How much data can we lose?"**

> Example: RPO = 1 hour → If the system crashes, you can afford to lose at most 1 hour of data. So you need backups every hour minimum.

```
TIMELINE VISUALIZATION:
────────────────────────────────────────────────────────
Last Backup     Incident Occurs     System Restored
    |                |                    |
    |<── RPO ───────>|<────── RTO ───────>|
    |  (data loss    |  (downtime window) |
    |   tolerance)   |                    |
────────────────────────────────────────────────────────
```

**Setting RTO and RPO:**

| Tier | Example Service | RTO | RPO |
|---|---|---|---|
| Tier 0 (Critical) | Payment processing | < 15 min | 0 (no data loss) |
| Tier 1 (High) | User auth service | < 1 hour | < 5 min |
| Tier 2 (Medium) | Reporting dashboard | < 4 hours | < 1 hour |
| Tier 3 (Low) | Internal tools | < 24 hours | < 24 hours |

> **Key insight:** The tighter your RTO/RPO, the more expensive your infrastructure (more replicas, more frequent backups, active-active setup). Business decides the tolerance, you build the architecture to meet it.

---

### 1.4 Escalation Chains

An **escalation chain** (also called escalation tree or escalation path) is the defined sequence of **who to contact** and **when** during an incident.

**Why you need it:**
- Without it: everyone calls the same person (usually the most senior), burning them out.
- With it: structured path → right person gets paged at the right time → faster resolution.

**L1 → L2 → L3 Structure:**

```
ESCALATION LEVELS
──────────────────
L1 — On-Call Engineer (First Responder)
  → Handles known issues using runbooks
  → Escalates if not resolved in 30 min

L2 — Senior Engineer / Tech Lead
  → Deep technical expertise
  → Handles unknown or complex issues
  → Escalates if not resolved in 60 min

L3 — Principal Engineer / Architect / Vendor Support
  → System design-level issues
  → External vendor escalation (AWS/GCP support)
  → Escalates to management if business impact is critical

L4 — Management / CTO / War Room
  → Customer communication, business decisions
  → Only for P0/P1 incidents with major business impact
```

**Rules of good escalation:**
1. **Time-boxed** — Define exactly when to escalate (e.g., "if unresolved in 30 min, page L2").
2. **No skipping levels** — L1 must attempt before paging L2. Avoid always going straight to the expert.
3. **Parallel escalation for P0** — For critical incidents, page L1 + L2 simultaneously.
4. **Single incident commander** — One person owns the incident. Everyone else supports.
5. **Communicate status** — Update a shared channel every 15-30 min even if no resolution yet.

---

### 1.5 Severity Levels (P0 to P3)

Before escalating, you must know how bad the incident is.

| Severity | Definition | Example | Response Time | Escalation |
|---|---|---|---|---|
| **P0** | Complete outage, all users affected | Entire app down | Immediate (< 5 min) | L1 + L2 simultaneously |
| **P1** | Major feature broken, many users affected | Payments failing | < 15 min | L1, escalate to L2 in 30 min |
| **P2** | Partial degradation, some users affected | Slow response times | < 1 hour | L1, escalate if needed |
| **P3** | Minor issue, workaround exists | UI bug | < 4 hours | L1 during business hours |

---

### 1.6 Failover Drills (Controlled)

A **failover drill** is a planned, controlled test where you intentionally break something to verify your recovery process works.

**Why do it?**
- "Our DR plan works" is a hypothesis. A drill tests the hypothesis.
- Discovering gaps during a drill (planned) is infinitely better than during a real incident (unplanned).
- Builds **muscle memory** — your team gets comfortable with emergency procedures.

**Types of drills:**

| Drill Type | What You Test | Example |
|---|---|---|
| **Tabletop Exercise** | Walk through the runbook verbally, no real action | "What would we do if zone-a failed?" |
| **Failover Drill** | Actually fail over to backup/DR environment | Switch traffic from primary to DR cluster |
| **Chaos Engineering** | Randomly inject failures in production/staging | Kill pods, block network, spike CPU |
| **Game Day** | Full team simulation of a major incident | "Pretend it's 3 AM and the DB is down" |

**Drill checklist:**
- [ ] Define scope: what are you failing over? What's out of scope?
- [ ] Notify all stakeholders before the drill.
- [ ] Have a rollback plan ready before you start.
- [ ] Measure actual RTO — does it meet your target?
- [ ] Document what broke, what was slow, what was confusing.
- [ ] Write a post-drill report with action items.

---

## SECTION 2: Full Runbook Templates

---

### 2.1 Incident Runbook Template

```markdown
# RUNBOOK: [Incident Name]
**ID:** RB-001
**Service:** [Service Name]
**Severity:** P1
**Last Updated:** 2026-05-09
**Owner:** Platform Team

---

## 1. SYMPTOMS / TRIGGER
- Alert: `<alert-name>` fires in PagerDuty/Grafana
- User reports: [describe user-facing symptom]
- Monitoring shows: [describe what metrics look abnormal]

Example:
- Alert `pod-crashloopbackoff` fires for namespace `payments`
- Users report "Payment failed" errors
- Error rate on `/api/payment` > 5% for > 5 minutes

---

## 2. SEVERITY ASSESSMENT
Answer these to confirm severity:
- [ ] Is the entire service down? → P0
- [ ] Are payments/auth/core flows broken? → P1
- [ ] Is it partial degradation? → P2
- [ ] Is there a workaround available? → P3

---

## 3. INITIAL RESPONSE (Do in the first 5 minutes)
1. Acknowledge the alert in PagerDuty.
2. Post in #incidents Slack channel:
   "P[X] incident declared for [service]. I am investigating. Next update in 15 min."
3. Start a war room: create Zoom link, share in #incidents.
4. Assign roles:
   - Incident Commander: [you or most senior available]
   - Comms Lead: [person posting updates]
   - Tech Lead: [person doing the fixing]

---

## 4. DIAGNOSTIC STEPS

### Step 1: Check pod health
```bash
kubectl get pods -n payments
kubectl describe pod <crashing-pod-name> -n payments
kubectl logs <crashing-pod-name> -n payments --previous
```
→ Look for: OOMKilled, CrashLoopBackOff, ImagePullBackOff, config errors

### Step 2: Check recent deployments
```bash
kubectl rollout history deployment/payment-service -n payments
# Was there a recent deploy?
```

### Step 3: Check resource usage
```bash
kubectl top pods -n payments
kubectl top nodes
```
→ Look for: CPU/Memory throttling, node pressure

### Step 4: Check upstream dependencies
```bash
# Database connectivity
kubectl exec -it <pod> -n payments -- nc -zv db-host 5432

# Check if external API is reachable
kubectl exec -it <pod> -n payments -- curl -v https://external-api.example.com/health
```

### Step 5: Check Kubernetes events
```bash
kubectl get events -n payments --sort-by='.lastTimestamp' | tail -20
```

---

## 5. REMEDIATION STEPS

### Scenario A: Bad deployment (most common)
```bash
# Roll back to last known good version
kubectl rollout undo deployment/payment-service -n payments

# Verify rollback
kubectl rollout status deployment/payment-service -n payments
```

### Scenario B: OOMKilled (memory limit too low)
```bash
# Temporarily increase memory limit
kubectl set resources deployment/payment-service \
  -n payments \
  --limits=memory=512Mi

# Then update the deployment YAML in git for permanent fix
```

### Scenario C: Database connection failure
```bash
# Check DB pod/service
kubectl get svc -n database
kubectl get pods -n database

# Check connection secret is correct
kubectl get secret db-credentials -n payments -o yaml
```

### Scenario D: Node failure causing pod evictions
```bash
# Check node health
kubectl get nodes
kubectl describe node <unhealthy-node>

# Cordon bad node
kubectl cordon <node-name>

# Check if pods rescheduled successfully
kubectl get pods -n payments -o wide
```

---

## 6. VERIFICATION STEPS
After applying a fix, confirm it actually worked:

```bash
# 1. All pods are Running
kubectl get pods -n payments

# 2. No error logs in last 5 minutes
kubectl logs deployment/payment-service -n payments --since=5m | grep -i error

# 3. Health endpoint responds
kubectl exec -it <pod> -n payments -- curl -s http://localhost:8080/health

# 4. Error rate dropped in Grafana
# → Navigate to: Grafana → Payments Dashboard → Error Rate panel
# → Confirm it's back below 1%
```

Declare incident resolved when:
- [ ] All pods are Running and Ready
- [ ] Error rate is back to baseline
- [ ] No new alerts firing
- [ ] Smoke test passes (manual check of key user flow)

---

## 7. ESCALATION PATH
| Time | Action |
|---|---|
| 0 min | L1 (On-call) starts investigation |
| 30 min | If unresolved, page L2 (Senior Engineer) |
| 60 min | If unresolved, page L3 (Principal/Architect) |
| 90 min | If P0/P1 still unresolved, loop in Engineering Manager |

**L2 Contact:** [Name] — PagerDuty / Slack: @name
**L3 Contact:** [Name] — PagerDuty / Slack: @name
**Manager:** [Name] — Phone: xxx-xxxx (P0 only)

---

## 8. ROLLBACK PROCEDURE
If your fix made things worse:
```bash
# Rollback deployment
kubectl rollout undo deployment/payment-service -n payments

# Restore previous config from git
git checkout HEAD~1 -- k8s/payments/deployment.yaml
kubectl apply -f k8s/payments/deployment.yaml
```

---

## 9. POST-INCIDENT ACTION
- [ ] Write postmortem within 48 hours
- [ ] Add new failure scenario to runbook if not covered
- [ ] Create Jira ticket for permanent fix
- [ ] Update monitoring/alerts if gap found
```

---

### 2.2 Escalation Matrix Template

```markdown
# ESCALATION MATRIX: Platform Team

## Severity → Response Time → Who to Contact

| Severity | Max Response | L1 (On-Call) | L2 (Senior) | L3 (Principal) | Manager |
|---|---|---|---|---|---|
| P0 | 5 min | Page immediately | Page simultaneously | Page simultaneously | Notify |
| P1 | 15 min | Page | Escalate at 30 min | Escalate at 60 min | Notify at 60 min |
| P2 | 1 hour | Page | Escalate at 2 hours | Only if needed | Only if needed |
| P3 | 4 hours | Ticket | — | — | — |

## On-Call Schedule
| Week | L1 On-Call | L2 Backup |
|---|---|---|
| Week 1 | Engineer A | Senior B |
| Week 2 | Engineer C | Senior D |

## Vendor Escalation
| Vendor | Support Link | Severity to Escalate |
|---|---|---|
| GCP/GKE | console.cloud.google.com/support | P0 node/zone issues |
| AWS | aws.amazon.com/support | P0 infra issues |
| PagerDuty | support.pagerduty.com | Alerting failures |
```

---

## SECTION 3: LABS (Hands-On Practice)

---

### LAB 1: Create a Complete Runbook

**Goal:** Write a real runbook for a `CrashLoopBackOff` incident. This is one of the most common Kubernetes issues you'll face.

**Step 1: Set up a broken app to practice with**
```bash
kubectl create namespace runbook-lab

# Deploy an app that will crash (bad command)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
  namespace: runbook-lab
spec:
  replicas: 3
  selector:
    matchLabels:
      app: broken-app
  template:
    metadata:
      labels:
        app: broken-app
    spec:
      containers:
        - name: broken-app
          image: nginx:1.21
          command: ["/bin/sh", "-c", "echo starting; sleep 5; exit 1"]
EOF
```

**Step 2: Observe symptoms (Diagnostic section of runbook)**
```bash
# Observe CrashLoopBackOff
kubectl get pods -n runbook-lab -w

# Check restart count
kubectl get pods -n runbook-lab
# RESTARTS column will keep growing
```

**Step 3: Practice your diagnostic steps**
```bash
# Step A: Describe the pod
kubectl describe pod <pod-name> -n runbook-lab
# Look for: Last State, Reason, Exit Code

# Step B: Read crash logs
kubectl logs <pod-name> -n runbook-lab --previous
# --previous = logs from the LAST crashed container instance

# Step C: Check events
kubectl get events -n runbook-lab --sort-by='.lastTimestamp'
```

**What you should see in describe:**
```
State:          Waiting
  Reason:       CrashLoopBackOff
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
```

**Step 4: Practice remediation**
```bash
# Fix: update the deployment with a working command
kubectl set env deployment/broken-app -n runbook-lab FIXED=true

# Or patch the command:
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
  namespace: runbook-lab
spec:
  replicas: 3
  selector:
    matchLabels:
      app: broken-app
  template:
    metadata:
      labels:
        app: broken-app
    spec:
      containers:
        - name: broken-app
          image: nginx:1.21
          # Remove the bad command — nginx will start normally
EOF
```

**Step 5: Verify**
```bash
# All pods should be Running
kubectl get pods -n runbook-lab

# No error logs
kubectl logs deployment/broken-app -n runbook-lab --since=2m

# Check rollout status
kubectl rollout status deployment/broken-app -n runbook-lab
```

**Step 6: Practice rollback**
```bash
# View rollout history
kubectl rollout history deployment/broken-app -n runbook-lab

# Rollback to previous version
kubectl rollout undo deployment/broken-app -n runbook-lab

# Verify
kubectl get pods -n runbook-lab
```

**Now write your runbook sections based on what you observed:**
- Symptoms: what did `kubectl get pods` show?
- Diagnostic: which commands identified the root cause?
- Remediation: what fixed it?
- Verification: how did you confirm it was fixed?

**Cleanup:**
```bash
kubectl delete namespace runbook-lab
```

---

### LAB 2: Build an Escalation Tree

**Goal:** Design and document a realistic L1 → L2 → L3 escalation chain.

**Step 1: Define your service tiers**

On paper or in a doc, map out:
```
SERVICE INVENTORY
──────────────────
Tier 0 (P0 candidate):
  - payment-service (revenue critical)
  - auth-service (all users blocked if down)

Tier 1 (P1 candidate):
  - order-service
  - notification-service

Tier 2 (P2 candidate):
  - reporting-service
  - admin-dashboard
```

**Step 2: Build the escalation tree for each tier**

```
PAYMENT-SERVICE ESCALATION TREE
─────────────────────────────────
Alert fires in PagerDuty
        │
        ▼
   L1: On-Call Engineer
   ● Checks runbook RB-001
   ● Tries to resolve in 30 min
        │
        ├── Resolved? → Post resolution, write postmortem
        │
        └── Not resolved in 30 min?
                │
                ▼
           L2: Senior Platform Engineer
           ● Takes over, L1 supports
           ● Has deeper system access
                │
                ├── Resolved? → Post resolution, write postmortem
                │
                └── Not resolved in 60 min?
                        │
                        ▼
                   L3: Principal Engineer / GCP Support
                   ● Vendor escalation if infra issue
                   ● Architecture-level decisions
                        │
                        └── P0 still active at 90 min?
                                │
                                ▼
                           Management notified
                           Customer comms drafted
                           War room continues
```

**Step 3: Create a PagerDuty-style escalation policy (even if on paper)**
```
ESCALATION POLICY: payment-service
────────────────────────────────────
Step 1: Notify on-call L1 engineer immediately
  → Wait 30 minutes for acknowledgement
  → If no ack: also notify L2

Step 2: Notify L2 senior engineer at 30 min mark
  → If unresolved at 60 min: notify L3

Step 3: Notify L3 principal engineer at 60 min mark
  → If P0 still unresolved at 90 min: notify manager

RULES:
  ● Anyone can escalate earlier if they judge it necessary
  ● P0 incidents skip the wait — L1 + L2 paged simultaneously
  ● All escalations posted in #incidents Slack channel
```

**Step 4: Simulate a handoff conversation**

Practice saying (out loud or in writing):
```
"Hi [L2 name], escalating payment-service P1 incident to you.
Summary: payment-service pods are in CrashLoopBackOff since 14:30 UTC.
I tried: rollback (failed), pod restart (no change), checked DB connectivity (ok).
Current state: 0/3 pods running, error rate 100%.
What I need: your review of the deployment config and possible DB schema issue.
Zoom link: [link]. Incident channel: #incident-20260509"
```

This is called a **SBAR handoff**: Situation, Background, Assessment, Recommendation.

---

### LAB 3: Failover Simulation

**Goal:** Simulate a workload outage on Kubernetes, then follow your runbook to recover. Measure your actual RTO.

**Step 1: Set up a "production-like" app**
```bash
kubectl create namespace failover-lab

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app
  namespace: failover-lab
spec:
  replicas: 4
  selector:
    matchLabels:
      app: prod-app
  template:
    metadata:
      labels:
        app: prod-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: prod-app-svc
  namespace: failover-lab
spec:
  selector:
    app: prod-app
  ports:
    - port: 80
      targetPort: 80
EOF
```

**Step 2: Verify healthy state (baseline)**
```bash
kubectl get pods -n failover-lab
kubectl get svc -n failover-lab

# Confirm traffic works
kubectl port-forward svc/prod-app-svc 8080:80 -n failover-lab &
curl http://localhost:8080   # Should return nginx page
```

**Step 3: Start the clock — BEGIN DRILL**
```bash
echo "DRILL STARTED AT: $(date)"
# Note the exact time — you're measuring RTO from here
```

**Step 4: Inject failure (simulate the incident)**
```bash
# Simulate: bad image pushed to production
kubectl set image deployment/prod-app nginx=nginx:broken-image-that-doesnt-exist -n failover-lab

# Or simulate: resource exhaustion (set impossible memory request)
# kubectl set resources deployment/prod-app --requests=memory=999Gi -n failover-lab
```

**Step 5: Observe — you are now "L1 on-call"**
```bash
# You get paged. First thing you do:
kubectl get pods -n failover-lab
# Expected: ImagePullBackOff or Pending

kubectl get events -n failover-lab --sort-by='.lastTimestamp'
# Expected: Failed to pull image "nginx:broken-image-that-doesnt-exist"
```

**Step 6: Follow the diagnostic runbook**
```bash
# Describe a bad pod
kubectl describe pod <bad-pod-name> -n failover-lab
# Identify root cause: bad image tag

# Check rollout history
kubectl rollout history deployment/prod-app -n failover-lab
```

**Step 7: Remediate — follow the runbook**
```bash
# REMEDIATION: Roll back to last known good version
kubectl rollout undo deployment/prod-app -n failover-lab

# Watch recovery
kubectl rollout status deployment/prod-app -n failover-lab -w
```

**Step 8: Verify**
```bash
kubectl get pods -n failover-lab
# All 4 pods should be Running

# Test traffic again
curl http://localhost:8080
```

**Step 9: Stop the clock — MEASURE RTO**
```bash
echo "DRILL ENDED AT: $(date)"
# Calculate: end_time - start_time = your actual RTO
# Compare against your target RTO
```

**Step 10: Post-drill debrief (do this immediately)**
Answer these questions:
```
1. What was the actual RTO? Did it meet the target?
2. Was the runbook easy to follow, or confusing?
3. Were any steps missing from the runbook?
4. Did you have all the access/permissions you needed?
5. Were alerts fired correctly? Were they too late/early?
6. What would have made recovery faster?
```

**Cleanup:**
```bash
kill %1  # Stop port-forward
kubectl delete namespace failover-lab
```

---

## SECTION 4: Postmortem Template

A postmortem is written **after** every significant incident. It's blame-free — the goal is to fix systems, not punish people.

```markdown
# POSTMORTEM: [Incident Title]
**Date:** 2026-05-09
**Severity:** P1
**Duration:** 47 minutes
**Author:** [Your Name]
**Reviewers:** [Team]

---

## INCIDENT SUMMARY
[2-3 sentences. What happened, who was affected, how long it lasted.]

Example: "The payment-service experienced 100% error rate from 14:30 to 15:17 UTC
on 2026-05-09, affecting all users attempting checkout. Root cause was a bad
Docker image deployed via CI/CD pipeline. Incident was resolved by rolling back
to the previous deployment."

---

## TIMELINE
| Time (UTC) | Event |
|---|---|
| 14:30 | Deployment v2.3.1 triggered via CI/CD |
| 14:32 | Error rate began rising |
| 14:35 | PagerDuty alert fired (2 min alert delay) |
| 14:37 | L1 acknowledged alert |
| 14:45 | Root cause identified (bad image tag) |
| 14:52 | Rollback initiated |
| 15:17 | All pods healthy, incident resolved |

---

## ROOT CAUSE
[Be specific. What exactly caused the incident?]

"The CI/CD pipeline deployed image tag `nginx:v2.3.1-broken` which referenced
a non-existent image in the container registry. The image had been accidentally
deleted during a registry cleanup job."

---

## CONTRIBUTING FACTORS
- No image validation step in CI/CD pipeline
- Alert threshold was 2 minutes — could be reduced to 30 seconds
- Runbook did not cover ImagePullBackOff scenario

---

## WHAT WENT WELL
- On-call engineer acknowledged within 2 minutes
- Rollback procedure worked correctly and quickly
- Incident channel communication was clear

---

## WHAT WENT POORLY
- Alert delay of 2 minutes was too long for a P1
- No pre-deployment image validation
- Runbook missing ImagePullBackOff diagnostic steps

---

## ACTION ITEMS
| Action | Owner | Due Date | Priority |
|---|---|---|---|
| Add image existence check to CI/CD pipeline | DevOps Team | 2026-05-16 | P1 |
| Reduce alert threshold from 2 min to 30 sec | SRE Team | 2026-05-14 | P2 |
| Update runbook RB-001 with ImagePullBackOff steps | On-call L1 | 2026-05-12 | P2 |
| Add registry cleanup job safeguards | Platform Team | 2026-05-23 | P2 |

---

## LESSONS LEARNED
"Automated image validation before deployment would have prevented this
incident entirely. We will treat image registry integrity as part of
our deployment pipeline, not an assumption."
```

---

## SECTION 5: Quick Reference Cheat Sheet

```
RUNBOOK SECTIONS (in order)
─────────────────────────────
1. Symptoms / Trigger
2. Severity Assessment
3. Initial Response (first 5 min)
4. Diagnostic Steps
5. Remediation Steps
6. Verification Steps
7. Escalation Path
8. Rollback Procedure
9. Post-Incident Actions

RTO vs RPO
───────────
RTO = How long can we be DOWN?    (time to recover)
RPO = How much DATA can we LOSE?  (backup frequency)
Tighter = more expensive infrastructure

SEVERITY LEVELS
────────────────
P0 → Full outage, immediate page L1+L2
P1 → Major feature broken, < 15 min response
P2 → Partial degradation, < 1 hour response
P3 → Minor issue, next business day OK

ESCALATION RULE
────────────────
L1: Handles known issues with runbooks
L2: Escalate if unresolved in 30 min
L3: Escalate if unresolved in 60 min
Manager: Escalate P0/P1 at 90 min

FAILOVER DRILL CHECKLIST
──────────────────────────
□ Define scope before starting
□ Notify stakeholders
□ Have rollback ready
□ Start the clock (measure RTO)
□ Follow runbook exactly
□ Stop the clock
□ Debrief immediately
□ Write action items

USEFUL KUBECTL COMMANDS FOR INCIDENTS
───────────────────────────────────────
# Check all pod statuses
kubectl get pods -A

# Crash logs from previous container
kubectl logs <pod> --previous

# Describe pod (events, resource usage)
kubectl describe pod <pod>

# Rollback a deployment
kubectl rollout undo deployment/<name>

# Check rollout history
kubectl rollout history deployment/<name>

# Watch pods in real time
kubectl get pods -w

# Check all events in a namespace
kubectl get events -n <ns> --sort-by='.lastTimestamp'

# Force restart all pods
kubectl rollout restart deployment/<name>
```

---

## SECTION 6: INFRAThrone Mindset — Senior Engineer Thinking

---

### A Runbook Is Worthless Unless Tested

**Bad:** Write runbook, file it away, never look at it again.
**Good:** Run a drill every quarter. Update the runbook after every incident. Rotate who follows it so the whole team is practiced.

Test your runbook like you test your code:
- Can a junior engineer follow it without asking questions?
- Does every command actually work?
- Is every assumption documented?

---

### Escalation Must Follow Path of Least Confusion

The worst thing during an incident is confusion about who's in charge and what's happening.

**Golden rules:**
1. **One incident commander.** Everyone else supports.
2. **Over-communicate.** Post in the incident channel every 15 minutes, even if it says "still investigating, no update."
3. **Never page someone without context.** Give them: what's broken, what you tried, what you need.
4. **Document in real time.** Use the incident channel as a live log. You'll need it for the postmortem.

---

### DR is Successful Only When... "The Team Can Execute It Half-Asleep at 3 AM"

This is the real test. If your DR plan requires:
- Someone to remember 20 steps from memory → Fail.
- Deep expert knowledge to execute → Fail.
- 3 different logins and 2 VPN connections → Fail.

Good DR means:
- Runbook is in an always-available location (not on a VPN-gated internal wiki at 3 AM).
- Every step is a copy-paste command, not "figure it out."
- Any team member can execute it, not just the expert.
- It's been run before. Multiple times.

---

### Interview-Ready Answers

**Q: What's the difference between RTO and RPO?**
> A: RTO is the maximum acceptable downtime — how long you can be offline before it violates your SLA. RPO is the maximum acceptable data loss — how much data you can afford to lose, which drives your backup frequency. For payment services, I've worked with RTO of 15 minutes and RPO near-zero using synchronous DB replication.

**Q: Walk me through how you handle a P1 incident.**
> A: First, I acknowledge the alert immediately and post in the incident channel. I follow the runbook for that service — starting with diagnostic steps to confirm root cause. I don't jump straight to remediation. If I can't resolve in 30 minutes, I escalate to L2 with a full SBAR handoff. After resolution, I write a postmortem within 48 hours with concrete action items.

**Q: Why do you write postmortems? What's in them?**
> A: Postmortems are how organizations learn from incidents systemically. They're blameless — focused on system failures, not human errors. A good postmortem has: incident summary, timeline, root cause, what went well, what went poorly, and action items with owners and due dates. Without postmortems, you fix the same incidents repeatedly.

**Q: How do you approach a failover drill?**
> A: I define the scope and target RTO upfront, notify stakeholders, and ensure a rollback plan exists before starting. I inject the failure, follow the runbook exactly as written (no shortcuts — that defeats the purpose), and measure the actual RTO. Then I debrief immediately with the team and capture gaps as action items. The goal is to make the runbook better, not to show that everything went perfectly.

---

## SECTION 7: Study Links

| Resource | URL |
|---|---|
| Google SRE Book (Free) | https://sre.google/sre-book/table-of-contents/ |
| Google SRE Workbook (Free) | https://sre.google/workbook/table-of-contents/ |
| PagerDuty Incident Response Guide | https://response.pagerduty.com/ |
| PagerDuty Postmortem Templates | https://postmortems.pagerduty.com/ |
| Atlassian Incident Management | https://www.atlassian.com/incident-management |
| Kubernetes Troubleshooting | https://kubernetes.io/docs/tasks/debug/debug-application/ |

---

## SECTION 8: Self-Check Questions

Answer these after studying. If you can't, re-read that section.

1. What are the 4 main sections every incident runbook must have?
2. What does RTO stand for, and what does it measure?
3. What does RPO stand for, and what does it measure?
4. If RTO = 2 hours, what does that mean in plain English?
5. When do you escalate from L1 to L2?
6. What is a PDB (PodDisruptionBudget) and when would you use it in a runbook? (Review from Thursday)
7. What is a postmortem and why is it blameless?
8. Name 3 types of failover drills from easiest to most realistic.
9. What is SBAR and when do you use it?
10. If you run a failover drill and RTO is 3 hours but your target is 1 hour, what do you do next?

---

*Answer key: 1-Symptoms, Diagnostic, Remediation, Verification 2-Recovery Time Objective; max time system can be down 3-Recovery Point Objective; max acceptable data loss 4-You have 2 hours to restore service before SLA is breached 5-If unresolved after 30 min (or immediately for P0) 6-Ensures min pods stay running during voluntary disruptions like drains 7-Blame-free review of incident to improve systems not blame people 8-Tabletop exercise → Failover drill → Game Day/Chaos Engineering 9-Situation Background Assessment Recommendation; used when handing off incident to next responder 10-Document the gap, identify what slowed you down, improve runbook and infrastructure, run drill again*
