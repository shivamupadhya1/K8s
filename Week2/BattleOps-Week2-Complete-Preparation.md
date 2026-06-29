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

## How To Use This Guide Like An Incident Engineer

This file should not be read as a motivational outline. It should be used like an operator manual.

Daily execution order:
1. Study the concepts for the day.
2. Build a hypothesis sheet before running any drill.
3. Run the drill under a timer.
4. Capture every command and observation.
5. Write the postmortem immediately while details are fresh.

For a DevOps engineer with 3 years of experience, BattleOps study is about strengthening three things:
1. Signal interpretation under pressure.
2. Safe decision making during uncertainty.
3. Incident communication with technical credibility.

---

## Detailed Day-Wise Battle Study Material

## Monday Deep Study: Outage Anatomy, Signal Discipline, and OSI Combat Thinking

### What to understand deeply
An outage usually presents as noise first, then symptoms, then confirmed facts. Weak operators chase noise. Strong operators structure uncertainty.

### Core concepts
1. Symptom vs signal vs root cause.
2. Leading indicators vs lagging indicators.
3. Blast radius assessment.
4. Why dashboards can disagree.
5. Why the loudest alert is often not the root cause.

### Practical outage interpretation model
Use a three-column model:
1. What is broken for users?
2. What systems show abnormal signals?
3. What evidence actually proves causal linkage?

### OSI combat thinking
You are not studying OSI academically. You are using it as a way to kill bad hypotheses faster.

Examples:
1. `502` at ingress does not automatically mean ingress is broken.
2. Successful `ping` does not prove TCP reachability.
3. Open port does not prove application correctness.
4. Healthy pod count does not prove service availability.

### Resources to study
1. SRE incident fundamentals:
- https://sre.google/sre-book/

2. Google SRE workbook:
- https://sre.google/workbook/

3. HTTP response and protocol basics:
- https://developer.mozilla.org/en-US/docs/Web/HTTP

4. Linux networking man pages:
- https://man7.org/linux/man-pages/

### Monday exercises
1. Write 15 outage symptoms and classify likely layer.
2. For each symptom, list the fastest disconfirming command.
3. Practice saying, "This is a symptom, not yet root cause."

## Tuesday Deep Study: DNS Failure Warfare, TLS Failure Warfare

### DNS battle concepts
DNS incidents are dangerous because they create inconsistent reality. One client may fail, another may work, and the system may look half-recovered.

#### Concepts to study
1. Resolver path and caching layers.
2. TTL impact on recovery speed.
3. Stub resolver vs recursive resolver.
4. Split-horizon and environment-specific records.
5. Why stale DNS can outlive the actual fix.

### TLS battle concepts
TLS incidents feel like network or app failures to many teams, but the root issue is often trust-chain or hostname validation.

#### Concepts to study
1. Certificate chain and trust store.
2. SAN, CN, SNI.
3. Expiry and rotation failure modes.
4. Cipher and protocol compatibility.
5. Reverse proxy termination vs passthrough.

### Operational lessons to internalize
1. A DNS incident can survive the record correction because cache still exists elsewhere.
2. A certificate can be technically present but unusable due to wrong hostname or missing intermediate.
3. Broad emergency changes during cert incidents can make recovery slower.

### Resources to study
1. DNS learning resources:
- https://www.cloudflare.com/learning/dns/

2. TLS handshake explanation:
- https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/

3. Web PKI and HTTPS basics:
- https://developer.mozilla.org/en-US/docs/Web/Security

4. OpenSSL docs:
- https://www.openssl.org/docs/

### Tuesday exercises
1. Explain why one region can recover before another in a DNS outage.
2. Explain why `curl -k` is useful for testing but dangerous for diagnosis if overused.
3. Write a four-step certificate-incident recovery sequence.

## Wednesday Deep Study: Kubernetes Control Plane vs Data Plane, Split-Brain Style Reasoning

### Concepts to understand deeply
Kubernetes incidents are hard because the API state, the node state, and the real traffic state can disagree.

### Core ideas
1. Control plane tells you desired and observed state, not always end-user truth.
2. Data plane carries real packets and real failures.
3. Endpoint health, readiness, and DNS must align.
4. CNI issues can break traffic even when pods are running.
5. NetworkPolicies can cause selective failure that looks random.

### Split-brain style reasoning
In BattleOps language, split-brain does not only mean a formal cluster quorum problem. It also means different components are operating on inconsistent assumptions or stale views.

Examples:
1. Service selector changed but rollout incomplete.
2. Endpoints object stale or not matching ready pods.
3. Ingress routes to a service that exists but has no healthy backend.
4. DNS resolves but pods cannot reach endpoint due to policy.

### Resources to study
1. Kubernetes debugging and networking docs:
- https://kubernetes.io/docs/tasks/debug/
- https://kubernetes.io/docs/concepts/cluster-administration/networking/

2. Services, endpoints, ingress, and DNS:
- https://kubernetes.io/docs/concepts/services-networking/

3. Probes:
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

### Wednesday exercises
1. Explain why a deployment can be healthy but the service broken.
2. Write the exact order of checks for pod -> service -> ingress failure.
3. Create a short list of control-plane truths vs data-plane truths.

## Thursday Deep Study: Release Incidents, Rollback Logic, and Observability Under Stress

### Release incident concepts
Many outages are not pure infrastructure failures. They are change failures revealed by traffic.

### Core ideas to study
1. Deployment completed vs release succeeded.
2. Canary signals and rollback triggers.
3. Safe rollback vs dangerous rollback.
4. Schema, cache, and config compatibility problems.
5. Dependency version drift and environment parity issues.

### Observability under pressure
During incident response, observability is not dashboard tourism. It is evidence extraction.

Ask these questions:
1. What changed first?
2. What user path broke first?
3. What signals confirm the blast radius?
4. Which graph proves mitigation worked?

### Resources to study
1. Deployment patterns and strategy:
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

2. Prometheus docs:
- https://prometheus.io/docs/introduction/overview/

3. OpenTelemetry docs:
- https://opentelemetry.io/docs/

4. Incident command and communication material from SRE workbook:
- https://sre.google/workbook/

### Thursday exercises
1. Define a rollback trigger based on latency, not only errors.
2. Define a change-failure scenario where rollback is unsafe.
3. Write one leader update that separates facts from assumptions.

## Friday Deep Study: Incident Command, Leadership, and Interview-Ready Storytelling

### Concepts to master
1. Owning the timeline.
2. Separating facts, assumptions, and decisions.
3. Avoiding false recovery.
4. Turning incident lessons into preventive engineering work.
5. Explaining incidents in interviews with clarity and honesty.

### Storytelling model for your experience level
Your answer should sound like an engineer who has seen production, not like a student repeating notes.

Use this sequence:
1. Business impact.
2. First technical signal.
3. Hypothesis path.
4. Most important command or dashboard used.
5. Decision made and why.
6. Fix, validation, prevention.

### Resources to study
1. SRE book and workbook:
- https://sre.google/sre-book/
- https://sre.google/workbook/

2. Pager and incident communication examples:
- Study public incident writeups from major providers for structure and clarity.

### Friday exercises
1. Deliver one five-minute incident summary aloud.
2. Write one postmortem with exact timestamps.
3. Practice one mock stakeholder update and one mock technical deep dive.

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

## BattleOps Resource Bank

## Study these first because they map directly to drills
1. Kubernetes official debugging docs:
- https://kubernetes.io/docs/tasks/debug/

2. Kubernetes networking model:
- https://kubernetes.io/docs/concepts/cluster-administration/networking/

3. Services, ingress, and DNS:
- https://kubernetes.io/docs/concepts/services-networking/

4. NGINX documentation:
- https://nginx.org/en/docs/

5. Prometheus documentation:
- https://prometheus.io/docs/

6. OpenTelemetry documentation:
- https://opentelemetry.io/docs/

7. OpenSSL docs:
- https://www.openssl.org/docs/

8. Cloudflare learning resources for DNS and TLS:
- https://www.cloudflare.com/learning/

## Books and long-form material worth your time
1. Site Reliability Engineering
Focus on:
- incident response
- monitoring philosophy
- postmortem discipline

2. The Site Reliability Workbook
Focus on:
- practical operations
- alerting and on-call practice
- incident learning loops

3. The DevOps Handbook
Focus on:
- system thinking
- deployment flow
- feedback loops

## What not to do while preparing
1. Do not only read runbooks without simulating failure.
2. Do not trust a single dashboard during drills.
3. Do not mark recovery complete without user-path validation.
4. Do not practice only technical fixes and ignore communication.

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

What you are really learning:
- how partial recovery creates dangerous false confidence,
- how to distinguish resolver inconsistency from application instability,
- how to validate recovery segment by segment rather than globally guessing.

Why this matters in incidents:
DNS failures rarely fail cleanly. They fail unevenly. That is exactly why teams misread them and close incidents too early.

Tasks:
1. Validate resolver chain.
2. Compare pod DNS vs node DNS behavior.
3. Identify if issue is CoreDNS, upstream resolver, or network path.
4. Mitigate with minimal blast radius.

Deliverable:
- dns-storm-postmortem.md

Expected observations:
1. One resolver may return stale values while another is correct.
2. Pod failures with healthy node DNS often suggest in-cluster DNS path issues.
3. Recovery can appear complete from one test point while failing elsewhere.

Reflection questions:
1. What exact evidence would let you declare segment-wise recovery?
2. Which clients or environments should be tested before closing the incident?
3. What would make you suspect cache persistence instead of active DNS failure?

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

What you are really learning:
- how service health can diverge from pod health,
- how control-plane objects can look normal while user traffic still fails,
- how rollout timing and readiness interact under pressure.

Why this matters in incidents:
This is one of the most common production failure patterns in Kubernetes-based systems. If you cannot diagnose endpoint drift confidently, you will lose time chasing logs from the wrong component.

Tasks:
1. Inspect endpoints and pod readiness.
2. Correlate with rolling update events.
3. Stabilize with controlled rollout strategy.

Deliverable:
- endpoint-drift-runbook.md

Expected observations:
1. Service object may remain unchanged while backend eligibility changes.
2. Readiness failures remove pods from traffic even if containers are still running.
3. Rolling deployments can create short windows of inconsistent routing.

Reflection questions:
1. Which object gives you the most direct truth about traffic eligibility?
2. How do you prove the problem is endpoint selection rather than ingress itself?
3. What deployment strategy would reduce this class of failure?

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

What you are really learning:
- how to evaluate rollback as an engineering decision instead of a reflex,
- how database, schema, cache, and config dependencies influence rollback safety,
- how to preserve stability while still gathering enough evidence for RCA.

Why this matters in incidents:
Rollback is often treated as automatically safe. In real systems, rollback can deepen outages when state has already changed. Mature operators evaluate rollback cost before executing it.

Tasks:
1. Evaluate roll-forward vs rollback risk.
2. Execute chosen path.
3. Monitor canary signals.

Deliverable:
- rollback-decision-log.md

Expected observations:
1. Partial failure can still justify roll-forward if rollback risk is higher.
2. Metrics must be read together, not alone.
3. Fast technical recovery may still leave hidden data inconsistency risk.

Reflection questions:
1. What system state makes rollback unsafe?
2. Which metrics tell you whether mitigation actually reduced user pain?
3. What evidence would justify controlled roll-forward instead of immediate rollback?

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

What you are really learning:
- how to convert incident chaos into reusable organizational learning,
- how to write a postmortem that engineers trust and leaders can act on,
- how to avoid blame language while still preserving accountability.

Why this matters in incidents:
An incident without a strong postmortem becomes a future repeat incident. High-quality postmortems improve systems, runbooks, and team behavior.

Expected observations:
1. Weak timelines hide causal ordering.
2. Blame language reduces learning quality.
3. Preventive actions without owners or due dates usually do not happen.

Reflection questions:
1. Did your postmortem identify primary root cause and contributing factors separately?
2. Did your actions reduce recurrence probability measurably?
3. Would another engineer be able to use your report during the next similar outage?

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
