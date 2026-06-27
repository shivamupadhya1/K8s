# CoreOps Week 2 Complete Preparation Guide (Advanced Practitioner Track)

## Who This Is For
This guide is designed for you as a DevOps engineer with around 3 years of hands-on experience. You are no longer in beginner mode. The goal here is to sharpen your fundamentals into production-grade operational judgment.

Week 2 should convert your existing knowledge into:
- faster debugging under pressure,
- stronger networking + Linux + container depth,
- cleaner operational communication,
- interview-ready explanations with practical examples.

---

## Week 2 Outcome Targets
By the end of this week, you should be able to:
1. Diagnose service outages using an OSI-layered method in less than 20 minutes for common faults.
2. Explain packet flow from client to pod/service/ingress and where failures can happen.
3. Build reproducible troubleshooting runbooks for network, process, and container-level issues.
4. Execute incident communication updates that are crisp, accurate, and leadership-ready.
5. Pass mid-level DevOps interview rounds on networking, Linux, containers, CI/CD reliability, and incident handling.

---

## Required Setup (Do This First)

## Local Environment
- OS: Windows with WSL2 Ubuntu (recommended)
- Terminal: PowerShell + WSL bash
- Git latest
- Docker Desktop + Kubernetes enabled (or Kind/Minikube)
- kubectl, helm, jq, yq, curl, wget, dig, nslookup, netstat/ss, tcpdump
- VS Code + extensions:
  - Docker
  - Kubernetes
  - YAML
  - GitLens
  - Markdown All in One

## Cloud Sandbox (Optional but Strongly Recommended)
- AWS free tier account
- IAM user with least-privilege lab policy
- 1 Linux VM (or EC2) for remote ops drills

## Folder Structure
Create this once and reuse all week:

```text
week2/
  notes/
  labs/
    lab-01-osi/
    lab-02-dns/
    lab-03-http/
    ...
  assignments/
  runbooks/
  postmortems/
  scripts/
```

## Baseline Validation Checklist
- docker run hello-world works
- kubectl version and cluster-info work
- kind/minikube cluster can schedule nginx pod
- dig google.com resolves
- curl https://example.com works

---

## CoreOps Week 2 Schedule (Mon-Fri)

## Monday: OSI + Linux Network Primitives
Focus:
- OSI model mapped to real tools
- Linux networking commands and packet path

Time Blocks:
1. 90 min concept refresh
2. 120 min guided labs
3. 90 min debug challenge
4. 60 min note consolidation

Deliverables:
- OSI quick decision tree
- command cheat sheet with when-to-use columns

## Tuesday: DNS + HTTP + TLS + Reverse Proxy
Focus:
- DNS resolution lifecycle
- HTTP request path and failure signatures
- TLS handshake debugging basics

Deliverables:
- DNS troubleshooting runbook
- TLS failure matrix

## Wednesday: Container + Kubernetes Network Fundamentals
Focus:
- container networking modes
- k8s service types, kube-proxy concepts, CoreDNS
- ingress routing basics

Deliverables:
- pod-to-service packet walk diagram
- k8s network triage checklist

## Thursday: CI/CD Reliability + Observability Basics
Focus:
- common pipeline failures and root causes
- logs/metrics/traces baseline strategy
- deployment verification gates

Deliverables:
- pipeline failure playbook
- deploy validation checklist

## Friday: Integrated Operations Drill + Interview Simulation
Focus:
- timed troubleshooting scenarios
- communication and postmortem quality
- interview answers with practical framing

Deliverables:
- one full postmortem
- one mock interview transcript

---

## Core Concept Framework: Practical OSI for DevOps
Use this order during incidents:
1. Layer 7 (Application): Is app healthy? Status endpoint? Error logs?
2. Layer 6/5 (Session/Presentation): TLS/cert/session mismatch?
3. Layer 4 (Transport): Port open? SYN/SYN-ACK? Connection reset/timeouts?
4. Layer 3 (Network): Routing, subnet ACL, SG/NACL, CNI path.
5. Layer 2 (Data Link): ARP/neighbor issues in local segment.
6. Layer 1 (Physical/Host): Interface down, VM/node host issue.

Golden rule:
- Start where symptoms appear.
- Prove each lower layer before moving up.

---

## Labs (Complete and Detailed)

## Lab 01: OSI-Layer Outage Mapping
Duration: 60-90 min

Objective:
- Build reflexes to map symptoms to layers and tools.

Tasks:
1. Create a table with columns: symptom, probable layer, verification command, fix.
2. Inject simple failures (wrong port, DNS typo, firewall block).
3. Time your diagnosis and annotate your decision path.

Expected Deliverable:
- osi-outage-map.md in labs/lab-01-osi/

Evaluation:
- Correct layer mapping >= 80%
- Root cause found in <= 15 min per scenario

## Lab 02: DNS Deep Dive
Duration: 90 min

Objective:
- Understand recursion, authoritative responses, and caching effects.

Tasks:
1. Use dig/nslookup for A, CNAME, MX records.
2. Compare responses from system resolver vs public resolver.
3. Simulate bad DNS entry in local hosts and verify behavior.
4. Capture a failure mode: NXDOMAIN, SERVFAIL, timeout.

Commands:
```bash
dig example.com
dig +trace example.com
nslookup example.com
cat /etc/resolv.conf
```

Deliverable:
- dns-runbook.md including failure signatures and fixes.

## Lab 03: HTTP Status + Header Path
Duration: 75 min

Objective:
- Debug app-level and proxy-level HTTP failures.

Tasks:
1. Start nginx container and curl from local host.
2. Use wrong upstream port and observe 502/504.
3. Use curl -v and inspect headers and connection behavior.

Commands:
```bash
curl -I http://localhost
curl -v http://localhost
```

Deliverable:
- http-debug-cheatsheet.md with status code to probable cause mapping.

## Lab 04: TLS Handshake Diagnosis
Duration: 90 min

Objective:
- Identify cert expiry, CN/SAN mismatch, and protocol mismatch quickly.

Tasks:
1. Hit a known TLS endpoint using openssl.
2. Observe certificate chain and expiry.
3. Simulate handshake failure with invalid cert setup in local reverse proxy.

Commands:
```bash
openssl s_client -connect example.com:443 -servername example.com
```

Deliverable:
- tls-failure-matrix.md

## Lab 05: Linux Process + Port Correlation
Duration: 60 min

Objective:
- Map app process to socket state and troubleshoot port conflicts.

Tasks:
1. Run sample process listening on custom port.
2. Identify process by port.
3. Kill stale process and restart service cleanly.

Commands:
```bash
ss -tulnp
lsof -i :8080
ps aux | grep <process>
```

Deliverable:
- process-port-runbook.md

## Lab 06: Network Packet Capture Basics
Duration: 90 min

Objective:
- Use tcpdump to validate whether traffic reaches target.

Tasks:
1. Capture packets for one app port.
2. Compare successful vs failed request trace.
3. Identify if failure is before app or inside app.

Commands:
```bash
sudo tcpdump -i any port 8080 -nn
```

Deliverable:
- pcap-analysis-notes.md

## Lab 07: Container Networking Modes
Duration: 90 min

Objective:
- Compare bridge vs host modes and exposure implications.

Tasks:
1. Run same app in bridge and host network mode.
2. Validate host reachability differences.
3. Document security implications.

Deliverable:
- container-networking-comparison.md

## Lab 08: Docker Compose Service Discovery
Duration: 75 min

Objective:
- Verify service name resolution and internal communication.

Tasks:
1. Create two services in compose.
2. Access one service from another by DNS name.
3. Break service name intentionally and debug.

Deliverable:
- compose-dns-debug.md

## Lab 09: Kubernetes Pod Networking Basics
Duration: 120 min

Objective:
- Understand pod IPs, service abstraction, and DNS in k8s.

Tasks:
1. Deploy two pods and one ClusterIP service.
2. Exec into pod and curl service DNS.
3. Delete and recreate pod; observe service stability.

Commands:
```bash
kubectl get pods -o wide
kubectl get svc
kubectl exec -it <pod> -- sh
```

Deliverable:
- k8s-pod-service-lab.md

## Lab 10: CoreDNS Failure Drill
Duration: 90 min

Objective:
- Detect DNS-related in-cluster failure quickly.

Tasks:
1. Check CoreDNS health.
2. Run nslookup inside pod.
3. Simulate broken DNS config and recover.

Deliverable:
- k8s-coredns-runbook.md

## Lab 11: Service Types and Traffic Entry
Duration: 90 min

Objective:
- Compare ClusterIP, NodePort, LoadBalancer behavior.

Tasks:
1. Expose same deployment with different service types.
2. Access from inside/outside cluster accordingly.
3. Document practical use-cases.

Deliverable:
- service-type-decision-guide.md

## Lab 12: Ingress Routing and 404/502 Analysis
Duration: 120 min

Objective:
- Troubleshoot ingress route/path/backend failures.

Tasks:
1. Deploy ingress with two path rules.
2. Break backend service name.
3. Trace 404 vs 502 cause.

Deliverable:
- ingress-failure-playbook.md

## Lab 13: Readiness/Liveness Impact
Duration: 75 min

Objective:
- Understand how bad probes create perceived outages.

Tasks:
1. Configure readiness probe with wrong endpoint.
2. Observe service not routing to pod.
3. Fix and verify restoration.

Deliverable:
- probe-debug-notes.md

## Lab 14: Resource Pressure and OOMKill Signals
Duration: 90 min

Objective:
- Diagnose memory/cpu pressure impact.

Tasks:
1. Deploy stress container.
2. Observe pod restart reason.
3. Tune requests/limits.

Deliverable:
- resource-tuning-guide.md

## Lab 15: CI Pipeline Failure Taxonomy
Duration: 90 min

Objective:
- Categorize failures by dependency, environment, test, and artifact steps.

Tasks:
1. Build a sample pipeline with 4 stages.
2. Introduce failures in each stage.
3. Write one-line symptom and one-line root cause per failure.

Deliverable:
- ci-failure-taxonomy.md

## Lab 16: Deployment Safety Gates
Duration: 75 min

Objective:
- Add quick quality gates before production deployment.

Tasks:
1. Add smoke checks post-deploy.
2. Add rollback criteria.
3. Simulate failed rollout and rollback.

Deliverable:
- deploy-gates-checklist.md

## Lab 17: Logging Correlation Drill
Duration: 75 min

Objective:
- Correlate app logs, infra logs, and deployment timeline.

Tasks:
1. Trigger a controlled failure.
2. Collect relevant logs by timestamp window.
3. Identify first bad event.

Deliverable:
- timeline-correlation.md

## Lab 18: Incident Communication Simulation
Duration: 60 min

Objective:
- Write updates for engineer, manager, and stakeholder audiences.

Tasks:
1. Draft 15-min update template.
2. Draft ETA with confidence level.
3. Draft closure update with prevention action.

Deliverable:
- incident-comms-template.md

## Lab 19: Postmortem Writing
Duration: 90 min

Objective:
- Produce blameless, high-value postmortem.

Template fields:
- Summary
- Impact
- Timeline
- Root cause
- Contributing factors
- What worked
- What failed
- Corrective actions (owner + due date)

Deliverable:
- pm-incident-01.md

## Lab 20: Timed End-to-End Triage Challenge
Duration: 120 min

Objective:
- Solve a full-stack outage scenario under time pressure.

Scenario:
- Users report intermittent 502.
- Recent deploy done 40 min ago.
- Pod restarts increased.
- DNS latency spikes in cluster.

Success Criteria:
- Triage narrative
- verified root cause
- mitigation + permanent fix plan

Deliverable:
- final-triage-report.md

---

## Assignments (Graded)

## Assignment A1: OSI Outage Atlas
Goal:
- Build a practical outage atlas with at least 30 symptoms.

Requirements:
1. 30 real symptoms mapped to layer and commands.
2. At least 10 symptoms from Kubernetes environments.
3. At least 5 symptoms from TLS/Cert issues.
4. Include false-positive traps.

Rubric (100):
- Accuracy: 40
- Practical command quality: 25
- Clarity and speed orientation: 20
- Preventive recommendations: 15

## Assignment A2: DNS and Ingress Blackout Investigation
Goal:
- Reproduce and fix a compounded outage.

Required outputs:
1. Reproduction steps.
2. Timeline with exact command evidence.
3. Root cause tree (primary + contributing).
4. Runbook update to prevent recurrence.

Rubric (100):
- Reproducibility: 20
- Depth of analysis: 35
- Fix quality: 25
- Documentation quality: 20

## Assignment A3: CI/CD Reliability Hardening
Goal:
- Improve an unstable pipeline and quantify improvement.

Deliverables:
1. Before vs after failure rate.
2. Added gates and rollback strategy.
3. Standardized failure labels.
4. MTTR improvement evidence.

Rubric (100):
- Reliability improvement: 30
- Engineering rigor: 30
- Metrics quality: 20
- Communication: 20

## Assignment A4: Week 2 Capstone Incident Report
Goal:
- Create complete incident package as if presenting to senior leadership.

Package includes:
1. 1-page executive summary
2. 2-page technical deep dive
3. prevention roadmap (30/60/90 days)

Rubric (100):
- Executive clarity: 20
- Technical correctness: 35
- Actionability: 30
- Ownership model: 15

---

## Interview Preparation (Week 2 Focus)

## Must-Answer Questions with Practical Structure
Use this response structure for every interview answer:
1. Real scenario context
2. Signal observed
3. Commands/tools used
4. Root cause
5. Fix and preventive action

Questions to drill:
1. How do you debug intermittent 502 in Kubernetes?
2. Difference between readiness and liveness in outage context?
3. How does DNS failure manifest differently at app and pod level?
4. How do you identify if issue is network vs application?
5. How do you reduce MTTR in CI/CD-related incidents?
6. Explain one incident where communication mattered as much as technical fix.
7. How do you design rollback criteria?
8. What metrics do you watch during rollout?
9. How do TLS cert issues break service communication?
10. What should be in a production-grade postmortem?

## Mock Interview Drill
- 45 min technical round
- 15 min scenario design round
- 10 min incident communication round

Scoring dimensions:
- depth
- speed
- structure
- practical command literacy

---

## Operational Cheat Sheets

## High-Value Linux Commands
```bash
ip a
ip route
ss -tulnp
netstat -plant
lsof -i :<port>
dig <domain>
curl -v <url>
traceroute <host>
sudo tcpdump -i any host <ip> and port <port>
```

## High-Value Kubernetes Commands
```bash
kubectl get pods -A -o wide
kubectl describe pod <pod>
kubectl logs <pod> --previous
kubectl get svc,ep
kubectl get ingress
kubectl get events --sort-by=.lastTimestamp
kubectl exec -it <pod> -- nslookup kubernetes.default
kubectl top pod
kubectl rollout status deploy/<name>
```

---

## Practical Troubleshooting Decision Trees

## If Users See 5xx Errors
1. Check ingress/controller logs.
2. Verify service endpoints exist.
3. Verify pod readiness.
4. Check app logs for exceptions/timeouts.
5. Validate upstream dependencies.

## If Service Is Reachable Internally But Not Externally
1. Check ingress/ALB/NLB/NodePort exposure.
2. Check firewall/security groups/network ACLs.
3. Validate DNS record points to correct endpoint.
4. Validate TLS cert and SNI config.

## If Pods Keep Restarting
1. Check events for OOMKilled/CrashLoopBackOff.
2. Inspect container logs including previous instance.
3. Validate env vars/secrets/configmaps.
4. Validate resource requests/limits.
5. Confirm startup and readiness probes.

---

## Daily Reflection Template (Use End of Day)
Create one file per day in notes/:

```markdown
# Day X Reflection

## What failed and why?

## What signal did I miss initially?

## Which command gave the fastest clarity?

## What runbook did I improve today?

## Interview question I can now answer strongly:
```

---

## 30/60/90 Improvement Plan (Post Week 2)

## Next 30 Days
- Build 10 additional outage simulations.
- Standardize 5 runbooks in your current project.
- Improve deployment verification checklist.

## Next 60 Days
- Introduce SLO-based alert tuning in one service.
- Build incident metrics dashboard: MTTD, MTTR, repeat incidents.

## Next 90 Days
- Lead one game day exercise end-to-end.
- Mentor juniors on structured triage and communication.

---

## Resources (High Signal)

Official Docs:
- Kubernetes docs: https://kubernetes.io/docs/
- Docker docs: https://docs.docker.com/
- Linux networking guide (Red Hat): https://access.redhat.com/documentation/
- NGINX docs: https://nginx.org/en/docs/
- OpenSSL docs: https://www.openssl.org/docs/

Practical Learning:
- Play with packet capture and protocol analysis using Wireshark docs
- CNCF resources for production Kubernetes operations

Books:
- Site Reliability Engineering (Google)
- The DevOps Handbook
- Kubernetes Up and Running

---

## Final Week 2 Completion Checklist
- [ ] 20 labs completed with notes and evidence
- [ ] 4 assignments submitted with rubric score self-assessment
- [ ] 1 full postmortem written
- [ ] 1 mock interview recorded and reviewed
- [ ] 1 personal runbook pack created for reuse at work

If you complete this seriously, you will not just know tools; you will operate like a production-focused DevOps engineer who can lead incidents and communicate with confidence.
