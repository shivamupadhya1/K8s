# BattleOps Week 2 Complete Preparation Guide (War-Room Simulation Track)

## Positioning
BattleOps is where theory gets weaponized into incident response capability.
You already have 3 years of DevOps exposure, so this plan is intentionally high-pressure and execution-first.

Primary mission for this week:
- build rapid triage muscle,
- detect deception signals and false leads,
- isolate root cause under uncertainty,
- communicate with leadership while fixing production.

---

## BattleOps Success Criteria
You pass Week 2 BattleOps if you can:
1. Run a structured incident bridge as Incident Commander for 45+ minutes.
2. Produce evidence-based hypotheses and kill wrong hypotheses fast.
3. Resolve multi-layer incidents spanning app, infra, and Kubernetes.
4. Deliver post-incident corrective action plan with owners and deadlines.
5. Clear advanced interview scenarios that test practical incident ownership.

---

## BattleOps Core Mindset
- Evidence over intuition.
- Time-boxed hypotheses.
- Fix forward when safe, rollback when uncertain.
- Communication is part of incident response, not optional.
- Every incident must produce a measurable prevention action.

---

## Operating Protocol: VERDICT-7 (Battle Adaptation)
Use this every incident.

1. Verify impact
2. Establish timeline
3. Reproduce or simulate quickly
4. Detect layer where failure emerges
5. Isolate blast radius
6. Contain and mitigate
7. Track permanent corrective actions

Commandment:
- If evidence does not support your hypothesis in 7-10 min, kill it and pivot.

---

## Week 2 Battle Plan (Mon-Fri)

## Monday: OSI Outage Edition Drills
You train on outage localization at each layer.

Objectives:
- translate ambiguous symptoms into deterministic checks,
- avoid jumping directly to application blame.

Deliverables:
- outage-layer quick board,
- false-positive library.

## Tuesday: DNS/TLS Chaos and Recovery
You break name resolution and certificates in controlled environments.

Objectives:
- identify and separate DNS vs TLS vs app defects,
- avoid accidental broad rollback during cert incidents.

Deliverables:
- dns/tls incident cards,
- recovery sequence runbook.

## Wednesday: Kubernetes Split-Brain and Service Routing Failures
You focus on cluster-state confusion, stale endpoints, and control-plane/data-plane mismatch.

Objectives:
- detect cluster state inconsistency,
- restore service routing without panic changes.

Deliverables:
- split-brain response flow,
- k8s stabilization checklist.

## Thursday: Deployment War Game + CI/CD Breakpoints
You simulate bad release and dependency failures under time pressure.

Objectives:
- choose rollback/roll-forward with risk model,
- preserve forensic evidence while restoring service.

Deliverables:
- release war-room playbook,
- rollback decision matrix.

## Friday: Final Battle Simulation + Interview Fireline
You run end-to-end chaos scenario plus mock interviews.

Deliverables:
- complete incident report package,
- readiness scorecard with weak areas and fix plan.

---

## BattleOps Lab Stack

Tools:
- Docker, Kubernetes (Kind/Minikube), kubectl
- NGINX, sample microservices
- tc/netem (optional for latency injection)
- curl, dig, openssl, tcpdump, jq
- Git repo for runbooks and postmortems

Recommended team mode:
- If possible, run with 2-3 peers.
- Roles: Incident Commander, Investigator, Scribe.

---

## High-Pressure Labs (Detailed)

## B-Lab 01: 15-Minute Outage Triad
Duration: 60 min total (4 rounds)

Round design:
- 15 min diagnosis + 5 min debrief.

Injectable faults:
- wrong target port,
- DNS record typo,
- invalid cert,
- readiness probe misconfig.

Win condition:
- probable root cause within 10 min,
- verified root cause within 15 min.

Deliverable:
- triad-round-log.md

## B-Lab 02: DNS Storm Under Pressure
Duration: 90 min

Scenario:
- random services fail name resolution intermittently.

Tasks:
1. Validate resolver chain.
2. Compare pod DNS vs node DNS behavior.
3. Identify if issue is CoreDNS, upstream resolver, or network path.
4. Mitigate with minimal blast radius.

Deliverable:
- dns-storm-postmortem.md

## B-Lab 03: TLS Expiry Midnight Drill
Duration: 75 min

Scenario:
- certificate expires and client traffic drops.

Tasks:
1. Confirm expiry and affected endpoints.
2. Measure blast radius.
3. Execute safe certificate replacement sequence.
4. Verify recovery and residual risk.

Deliverable:
- tls-expiry-recovery.md

## B-Lab 04: Reverse Proxy 502 Maze
Duration: 90 min

Scenario:
- ingress/proxy returns intermittent 502 after deployment.

Tasks:
1. Differentiate backend crash vs timeout vs route mismatch.
2. Validate endpoint health and upstream keepalive settings.
3. Fix with minimal config drift.

Deliverable:
- proxy-502-analysis.md

## B-Lab 05: Kubernetes Endpoint Drift
Duration: 120 min

Scenario:
- service exists but routes to stale/unready pods.

Tasks:
1. Inspect endpoints and pod readiness.
2. Correlate with rolling update events.
3. Stabilize with controlled rollout strategy.

Deliverable:
- endpoint-drift-runbook.md

## B-Lab 06: Split-Brain Simulation (Safe)
Duration: 120 min

Scenario concept:
- cluster components show inconsistent state perception.

Tasks:
1. Identify conflicting state indicators.
2. Freeze risky actions.
3. Establish source-of-truth evidence.
4. Reconcile state with low-risk sequence.

Deliverable:
- split-brain-war-notes.md

## B-Lab 07: CrashLoop and Probe Meltdown
Duration: 90 min

Scenario:
- bad probe config causes cascading unavailability.

Tasks:
1. Prove whether app is healthy but marked unready.
2. Patch probe settings.
3. Validate traffic restoration.

Deliverable:
- probe-meltdown-fix.md

## B-Lab 08: Resource Saturation Combat
Duration: 90 min

Scenario:
- sudden latency spike due to CPU/memory pressure.

Tasks:
1. Detect noisy neighbor or bad limits.
2. Apply emergency scaling/tuning.
3. Validate throughput recovery.

Deliverable:
- saturation-response-report.md

## B-Lab 09: CI Pipeline Sabotage Drill
Duration: 90 min

Scenario:
- pipeline fails unpredictably across stages.

Tasks:
1. Isolate failing stage quickly.
2. Label failure class (dependency/test/env/artifact).
3. Implement guardrails.

Deliverable:
- ci-sabotage-playbook.md

## B-Lab 10: Rollback Roulette
Duration: 75 min

Scenario:
- release introduces partial failures; rollback has side effects.

Tasks:
1. Evaluate roll-forward vs rollback risk.
2. Execute chosen path.
3. Monitor canary signals.

Deliverable:
- rollback-decision-log.md

## B-Lab 11: Incident Commander Communication Drill
Duration: 60 min

Tasks:
1. Send 15-minute cadence updates.
2. Separate confirmed facts from assumptions.
3. Provide confidence-rated ETA.

Deliverable:
- commander-comms-transcript.md

## B-Lab 12: Blameless Postmortem Under Deadline
Duration: 90 min

Tasks:
1. Produce full postmortem in under 90 min.
2. Include preventive backlog with owner/date.
3. Add metrics for recurrence prevention.

Deliverable:
- postmortem-deadline-edition.md

---

## Battle Assignments (Capstone Quality)

## B-Assignment 1: Outage Card Deck (20 Cards)
Goal:
- Build reusable battle cards for common outages.

Each card must include:
1. Symptom
2. First 3 commands
3. Most likely causes
4. Fastest safe mitigation
5. Permanent fix ideas
6. Escalation trigger

Scoring (100):
- Utility in real incidents: 35
- Precision: 30
- Speed orientation: 20
- Clarity: 15

## B-Assignment 2: Split-Brain Investigation Dossier
Goal:
- Produce a forensic quality dossier for a state inconsistency incident.

Mandatory sections:
1. Signal timeline
2. Conflicting evidence map
3. Decision log
4. Why rejected hypotheses were wrong
5. Permanent stabilization architecture

Scoring (100):
- Forensic quality: 40
- Logic and rigor: 30
- Risk awareness: 20
- Presentation: 10

## B-Assignment 3: War-Room Leadership Simulation
Goal:
- Lead a simulated incident from detection to closure.

Expected outputs:
1. Incident timeline with timestamps
2. Communication artifacts
3. Root cause and fix
4. 30/60/90 prevention roadmap

Scoring (100):
- Leadership and control: 30
- Technical execution: 35
- Communication quality: 20
- Prevention depth: 15

## B-Assignment 4: Interview Fireline Packet
Goal:
- Build interview-ready incident stories with evidence.

Required:
- 5 incident stories using STAR + technical proof format.
- For each: architecture sketch, commands used, metrics changed, lessons learned.

Scoring (100):
- Authenticity and detail: 35
- Technical depth: 35
- Story structure: 20
- Reflection quality: 10

---

## Interview Fireline (Advanced DevOps)

## Core Scenarios to Practice
1. Intermittent 502 in Kubernetes right after deployment.
2. DNS resolution succeeds on node but fails in pod.
3. TLS handshake failures after certificate rotation.
4. High error rate with healthy CPU but poor latency.
5. CI/CD passes tests but production fails instantly.
6. Rollback worsens incident due to schema mismatch.
7. Multiple alerts from unrelated services during one outage.

## How to Answer Strongly
Use this structure:
1. Incident context and impact.
2. First 5 minutes actions.
3. Hypothesis tree and evidence.
4. Decision point and risk trade-off.
5. Final fix and prevention action.

## What Interviewers Look For at 3 Years Experience
- You can triage without panic.
- You know how to reject wrong assumptions quickly.
- You treat communication as a technical responsibility.
- You can convert incidents into engineering improvements.

---

## War-Room Templates

## Incident Channel Kickoff Message
```text
Incident Declared: <ID>
Impact: <who/what is affected>
Start Time: <UTC>
Current Severity: <SEV-1/2/3>
Incident Commander: <name>
Investigators: <names>
Next Update ETA: <time>
```

## 15-Minute Update Template
```text
Time: <UTC>
What we know (confirmed):
What we suspect (hypothesis):
What we are doing now:
Risks:
ETA confidence:
```

## Closure Template
```text
Incident resolved at: <UTC>
Root cause:
Mitigation applied:
Permanent fix plan:
Owner(s):
Follow-up review date:
```

---

## Battle Metrics to Track
Track these during and after drills:
- MTTD (Mean Time to Detect)
- MTTA (Mean Time to Acknowledge)
- MTTR (Mean Time to Recover)
- Hypothesis Accuracy Rate
- False Lead Time Wasted
- Communication Cadence Adherence
- Repeat Incident Rate

Target benchmark for this week:
- MTTR improvement >= 25% between first and final drill.

---

## Practical Troubleshooting Quick Paths

## Path A: 502/504 From Ingress
1. ingress logs
2. service endpoints
3. pod readiness
4. app timeouts
5. dependency reachability

## Path B: Pod Healthy but Traffic Fails
1. service selector match
2. endpoints populated
3. network policy rules
4. DNS resolution in-cluster
5. node-level connectivity

## Path C: Random Timeouts
1. packet loss/latency checks
2. retries and timeout configs
3. downstream saturation
4. resource throttling indicators
5. burst vs steady load behavior

---

## 5-Day Intensity Timetable (Suggested)

Daily block model (6-8 hours):
1. 90 min concept + recent incident reading
2. 120 min drill execution
3. 60 min debrief + metric capture
4. 90 min second drill
5. 60 min interview response practice
6. 30 min runbook/postmortem update

Recovery rule:
- Stop after 90 min blocks for 10 min reset.
- BattleOps requires quality focus, not fatigue-driven errors.

---

## Resource Pack (High Value)

Official:
- Kubernetes debugging docs: https://kubernetes.io/docs/tasks/debug/
- Kubernetes networking model: https://kubernetes.io/docs/concepts/cluster-administration/networking/
- NGINX troubleshooting docs: https://docs.nginx.com/
- OpenSSL reference: https://www.openssl.org/docs/

Operational Learning:
- Google SRE workbook
- Incident.io and PagerDuty incident communication examples
- CNCF case studies on production outages

---

## Final Readiness Scorecard
Score yourself 1-5 each day:
1. Speed of first actionable hypothesis
2. Evidence quality
3. Root cause precision
4. Mitigation safety
5. Communication clarity
6. Prevention action quality

Graduation threshold:
- Average >= 4.0 by Friday.

---

## Final Completion Checklist
- [ ] 12 battle labs executed with evidence
- [ ] 4 battle assignments completed
- [ ] 1 full war-room simulation led
- [ ] 1 split-brain dossier created
- [ ] 5 interview stories prepared with technical proof
- [ ] measurable MTTR improvement documented

You are ready for advanced DevOps ownership when you can both fix the incident and narrate the decision process with evidence, speed, and leadership confidence.
