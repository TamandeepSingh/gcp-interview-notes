# Samsung Interview Overview (90-Min Format)

## Interview Structure

### 0–15 min: Deep Discussion
- "Explain your pipeline"
- "Explain your ELK architecture"
- They will dig deeper than the first round

### 15–60 min: Practical + Scenario
- Debugging problems
- System design
- Whiteboard or laptop task

### 60–90 min: Advanced + Behavioral
- Tradeoffs
- Security
- Decision making

---

## Core Topics to Expect

### 1. Kubernetes (Deep)
- Pod lifecycle
- Networking
- Ingress / Services
- Debugging

### 2. CI/CD Pipeline (GitHub Actions)
- Full end-to-end walkthrough
- Secrets handling
- Rollback strategy
- Failure handling

### 3. ELK + Logging (Strongest Area)
- Log flow end-to-end
- Failure scenarios
- Scaling issues

### 4. Terraform
- State management
- Drift handling
- Modules
- Real-world usage

### 5. GCP
- GKE
- IAM
- Networking

---

## Day 1 — Foundation + Depth

### Story 1: CI/CD Pipeline (GitHub Actions → GKE/EKS)
Must cover:
- Trigger (push / PR)
- Build Docker image
- Push to registry
- Deploy via Helm or kubectl
- ArgoCD role

Prepare for:
- What if deployment fails?
- How does rollback work?
- How are secrets managed?

### Story 2: ELK + Wazuh Pipeline
Explain like an architect:
- Fluent Bit → Logstash → Elasticsearch → Kibana
- Where parsing happens
- Where filtering happens
- PII masking logic

Prepare for:
- What if logs stop?
- How to scale?
- How to reduce cost?

### Story 3: Terraform Infra
Cover:
- How you structure code
- Modules
- State backend
- Drift handling

---

### Kubernetes Must-Know Concepts
- Pod lifecycle
- Liveness vs readiness probes
- Service types (ClusterIP, NodePort, LoadBalancer)
- Ingress flow

### Debugging Commands
```bash
kubectl get pods
kubectl describe pod <name>
kubectl logs <pod>
kubectl exec -it <pod> -- sh
```

---

## Key Scenarios to Prepare

### Scenario 1: Pod cannot access internet
Check:
- Network policy
- NAT gateway
- DNS

### Scenario 2: App deployed but not accessible
Check:
- Service
- Ingress
- Ports

### Scenario 3: Logs not reaching Elasticsearch
Check:
- Fluent Bit config
- Network
- Index issue

### Scenario 4: Terraform apply destroying resources
Reasons:
- State mismatch
- Config change

### Scenario 5: Pipeline passed but deployment failed
Reasons:
- Image tag mismatch
- ArgoCD sync issue

---

## Day 2 — Hands-On + Practice

### Hands-On in GCP
1. Create GKE cluster
2. Deploy app
3. Expose via service
4. Break it intentionally
5. Fix it

### Speaking Practice
Practice saying answers out loud:
> "First, I would check logs using `kubectl logs`..."

Structured spoken answers matter significantly in this round.

---

## How to Answer (Critical)

**Weak answer:**
> "I think the issue could be network…"

**Strong answer:**
> "First, I would verify pod status using `kubectl get pods`.
> Then check logs using `kubectl logs`.
> If it's a network issue, I'd verify service and DNS resolution..."

Structured thinking = selected.

---

## Samsung Health Department Context

Since this is a health-related department, emphasize:
- Security
- Data privacy (PII masking — highlight your experience)
- Reliability

Example line:
> "Since this involves sensitive data, I ensured PII masking at the Logstash layer…"

---

## Final Checklist

They will test:
- Depth
- Debugging
- Confidence

They don't care if you know everything. They care if:
- You think like an engineer
- You can handle production
