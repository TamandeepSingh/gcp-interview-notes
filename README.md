# 🚀 Complete DevOps Learning Repository

**Enterprise-grade DevOps learning system. Master Terraform, Kubernetes, Helm, Ansible, CI/CD, and GCP.**

This repository contains **advanced, production-ready content** designed for DevOps engineers preparing for technical interviews or building real-world infrastructure at scale.

---

## 📚 Complete Repository Structure

### **[gcp/](./gcp/)** - Google Cloud Platform
Production-grade GCP infrastructure and services guide.

- **[01-services.md](./gcp/01-services.md)** - Services landscape, compute decisions, networking, security, monitoring, multi-region HA

**Topics:** GCE vs GKE vs Cloud Run, Cloud SQL, Pub/Sub, BigQuery, Networking, IAM, Disaster Recovery, Cost Optimization


### **[terraform/](./terraform/)** - Infrastructure-as-Code
Master Terraform for managing cloud infrastructure at enterprise scale.

- **[01-fundamentals.md](./terraform/01-fundamentals.md)** - Execution flow, state machine, providers, lifecycle
- **[02-state_management.md](./terraform/02-state_management.md)** - **CRITICAL** - State files, locking, remote backends, recovery
- **[03-modules.md](./terraform/03-modules.md)** - Reusable patterns, composition, multi-cluster examples
- **[04-remote_backend.md](./terraform/04-remote_backend.md)** - GCS/S3 backends, encryption, disaster recovery
- **[05-terraform_gcp.md](./terraform/05-terraform_gcp.md)** - GCP provider, multi-project patterns, service accounts
- **[06-best_practices.md](./terraform/06-best_practices.md)** - Production patterns, CI/CD, validation, operational runbooks

**Topics:** State files (THE source of truth), remote backends, modules, multi-environment setups, disaster recovery, team workflows

---

### **[kubernetes/](./kubernetes/)** - Container Orchestration
Deep dive into Kubernetes architecture and production patterns.

- **[01-fundamentals.md](./kubernetes/01-fundamentals.md)** - Architecture, control plane, kubelet, pod lifecycle, scheduling, self-healing

**Topics:** Control plane components, controller reconciliation loops, pod scheduling, resource management, quality of service, self-healing

---

### **[helm/](./helm/)** - Kubernetes Package Manager
Package and deploy applications with templating and dependency management.

- **[01-fundamentals.md](./helm/01-fundamentals.md)** - Chart structure, template rendering, values, releases, hooks, GitOps

**Topics:** Helm charts, Go templating, values merging, dependency management, lifecycle hooks, release management

---

### **[ansible/](./ansible/)** - Configuration Management & Automation
Agentless automation for infrastructure configuration and deployment.

- **[01-fundamentals.md](./ansible/01-fundamentals.md)** - Playbooks, inventory, modules, handlers, roles, idempotency, rolling deployments

**Topics:** Agentless architecture, YAML playbooks, roles, variables, error handling, graceful shutdown, production deployments

---

### **[cicd/](./cicd/)** - Continuous Integration & Deployment
Automate your entire pipeline from code to production.

- **[01-fundamentals.md](./cicd/01-fundamentals.md)** - Pipeline architecture, infrastructure deployment, GitHub Actions, Cloud Build, GitOps

**Topics:** Build-test-deploy pipelines, infrastructure verification, automated rollback, security scanning, GitOps patterns, Flux/ArgoCD

---

### **[DEVOPS-GUIDE.md](./DEVOPS-GUIDE.md)** - Master Index
Comprehensive learning paths, interview scenarios, hands-on projects, and preparation checklist.

---

## 🎯 How to Use This Repository

### **Interview Prep (6-8 Weeks)**

**Week 1-2: Terraform Fundamentals**
- Read: `terraform/01-fundamentals.md`, `terraform/02-state_management.md`
- Focus: Why state matters, conflict prevention, disaster recovery
- Practice: Deploy with remote backend, test rollback
- Study: 10 interview scenarios provided

**Week 3-4: Kubernetes & Helm**
- Read: `kubernetes/01-fundamentals.md`, `helm/01-fundamentals.md`
- Focus: Pod scheduling, controller reconciliation, Helm templating
- Practice: Deploy Helm chart, implement auto-scaling
- Study: 15+ scenario-based questions with answers

**Week 5-6: CI/CD & Ansible**
- Read: `cicd/01-fundamentals.md`, `ansible/01-fundamentals.md`
- Focus: Infrastructure pipeline automation, configuration management
- Practice: GitHub Actions workflow, Ansible playbook
- Study: Real-world deployment patterns

**Week 7-8: GCP Integration & System Design**
- Read: `gcp/01-services.md`
- Practice: Design multi-region high-availability system
- Study: Architecture patterns and cost optimization
- Mock interview scenarios

### **Real-world Implementation**

**Deploy Microservices Platform End-to-End:**

```bash
# 1. Infrastructure (Terraform)
cd terraform/environments/prod
terraform init
terraform apply  # VPC, GKE cluster, Cloud SQL, Pub/Sub

# 2. Application Package (Helm)
cd ../../helm
helm create myapp
helm install myapp ./myapp -f values-prod.yaml --namespace production

# 3. CI/CD Pipeline (GitHub Actions + Terraform)
git push  # Triggers pipeline
# → Build Docker image → Push to registry
# → terraform validate → Deploy with Helm → Smoke tests

# 4. Configuration Management (Ansible)
cd ../../ansible
ansible-playbook -i inventory/prod production.yml

# 5. Monitor Everything (GCP Logging/Monitoring)
# Cloud Logging → Structured logs
# Cloud Monitoring → Metrics and alerts
```

---

## 🔥 Key Concepts You'll Master

### **Terraform**
- [ ] State file structure (JSON format, TF_STATE semantics)
- [ ] State locking (preventing concurrent modifications)
- [ ] Remote backends with encryption (GCS with CMK)
- [ ] Disaster recovery from state corruption
- [ ] Team workflows (remote state, shared backends)
- [ ] Modules for infrastructure reusability
- [ ] Multi-environment setups (dev/staging/prod)
- [ ] State import (existing cloud resources)

### **Kubernetes**
- [ ] Control plane components (API server, scheduler, kubelet)
- [ ] Controller reconciliation loop (desired vs actual state)
- [ ] Pod lifecycle (pending → running → succeeded/failed)
- [ ] Scheduling and node affinity
- [ ] Resource requests/limits and QoS classes
- [ ] Self-healing and auto-restart
- [ ] Health checks (readiness, liveness, startup probes)
- [ ] Graceful shutdown and PodDisruptionBudgets

### **Helm**
- [ ] Chart rendering with Go templates
- [ ] Values merging and variable precedence
- [ ] Helm dependencies and Chart.yaml
- [ ] Release management (install, upgrade, rollback)
- [ ] Lifecycle hooks (pre-install, post-upgrade, etc.)
- [ ] GitOps integration with Flux/ArgoCD

### **Ansible**
- [ ] Agentless architecture (SSH-based)
- [ ] Playbook idempotency
- [ ] Handlers and task ordering
- [ ] Roles and reusability
- [ ] Conditional execution and loops
- [ ] Error handling and graceful failures
- [ ] Rolling updates and blue-green deployments

### **CI/CD**
- [ ] Pipeline architecture (build → test → deploy)
- [ ] Infrastructure verification in pipelines
- [ ] Automated testing and security scanning
- [ ] Deployment safety (canary, blue-green, rolling)
- [ ] GitOps principles (Git as source of truth)
- [ ] Automated rollback strategies

### **GCP**
- [ ] Service landscape and when to use each
- [ ] Compute options (GCE, GKE, Cloud Run, App Engine)
- [ ] Networking (VPC, subnets, firewall, Cloud NAT)
- [ ] Identity (IAM, Service Accounts, Workload Identity)
- [ ] Storage options (Cloud Storage, Cloud SQL, Pub/Sub, BigQuery)
- [ ] Monitoring & observability stack
- [ ] Multi-region disaster recovery
- [ ] Cost optimization strategies

---

## 📊 Real-world Interview Scenarios

### Scenario 1: Production State Corruption
**Situation:** Terraform state file corrupted, need 15-minute RTO  
**File:** `terraform/02-state_management.md` (Disaster Recovery section)  
**Answer:** Rebuild from cloud resources via `terraform import`, use backend versioning for recovery

### Scenario 2: Safe Infrastructure Changes
**Situation:** Update production database safely with 2-person approval  
**Files:** `terraform/06-best_practices.md`, `cicd/01-fundamentals.md`  
**Answer:** Terraform plan in PR, manual approval, CI/CD auto-apply, health checks validate change

### Scenario 3: Kubernetes Scaling Under Load
**Situation:** Traffic spike 10x, need to handle scaling with 99.9% uptime  
**File:** `kubernetes/01-fundamentals.md` (Auto-scaling section)  
**Answer:** HPA + cluster autoscaler, pod disruption budgets, gradual rollout, monitor metrics

### Scenario 4: Multi-environment Consistency
**Situation:** Deploy same app to dev/staging/prod with different configs  
**Files:** `terraform/03-modules.md`, `helm/01-fundamentals.md`  
**Answer:** Same code with environment-specific tfvars and Helm values files

### Scenario 5: Git Workflow for Infrastructure
**Situation:** 10 engineers managing infrastructure safely  
**Files:** `terraform/06-best_practices.md`, `cicd/01-fundamentals.md`  
**Answer:** Feature branches, PR reviews, CI plan validation, auto-apply on merge

### Scenario 6: Multi-region Disaster Recovery
**Situation:** Primary region down, need failover in 5 minutes  
**File:** `gcp/01-services.md` (Disaster Recovery section)  
**Answer:** Multi-region setup, DNS failover, tested monthly, automated recovery

### Scenario 7: Pipeline Failure Analysis
**Situation:** Terraform apply failed mid-deployment  
**File:** `cicd/01-fundamentals.md` (Troubleshooting section)  
**Answer:** Check plan output, identify state mismatch, use partial apply, verify health

### Scenario 8: Cost Optimization from $5k to $2.5k/month
**Situation:** Cloud bill is too high, need to cut in half  
**File:** `gcp/01-services.md` (Cost Optimization section)  
**Answer:** Right-size instances, use reserved resources, optimize storage tiers, auto-shutdown

### Scenario 9: Helm Chart Deployment Strategy
**Situation:** Deploy chart updates without downtime  
**File:** `helm/01-fundamentals.md` (Production Patterns section)  
**Answer:** Rolling updates, health checks, hooks for data migrations, rollback capability

### Scenario 10: Application Configuration at Scale
**Situation:** Configure 50 servers consistently  
**File:** `ansible/01-fundamentals.md` (Playbooks section)  
**Answer:** Idempotent playbooks, roles for reusability, configuration management

---

## 💡 What Makes This Different

✅ **Intermediate to Advanced Content** - Production-grade, not "hello world"  
✅ **Real-world Scenarios** - Industry patterns, battle-tested approaches  
✅ **Common Mistakes** - What experienced engineers learn painfully  
✅ **Interview Ready** - 100+ scenario-based questions with answers  
✅ **Production Patterns** - Enterprise workflows, operational runbooks  
✅ **Internal Mechanics** - Pseudocode, architecture diagrams, request flows  
✅ **Debugging Guides** - How to solve common operational issues  
✅ **Best Practices** - Team workflows, security, cost optimization  

---

## 🚀 Practical Examples

### Example 1: Deploy Multi-region HA Application

```hcl
# terraform/main.tf
terraform {
  backend "gcs" {
    bucket         = "my-tf-state"
    prefix         = "prod"
    encryption_key = "..."  # Use Cloud KMS
  }
}

resource "google_container_cluster" "primary" {
  name     = "primary-${var.region}"
  region   = var.region
  
  # Enable cluster autoscaling
  autoscaling {
    min_node_count = 3
    max_node_count = 10
  }
  
  # Enable workload identity
  workload_identity_config {
    workload_pool = "${var.project}:googleapis.com"
  }
}
```

### Example 2: Helm Chart with Values Override

```yaml
# helm/values-prod.yaml
replicas: 5
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

nodeSelector:
  workload: production
```

### Example 3: CI/CD Pipeline with Terraform

```yaml
# .github/workflows/deploy.yml
name: Deploy Infrastructure
on:
  push:
    branches: [main]
    paths: ['terraform/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Terraform Plan
        run: terraform plan -out=tfplan
      - name: Terraform Apply
        run: terraform apply tfplan
      - name: Health Check
        run: ./scripts/health-check.sh
      - name: Rollback on Failure
        if: failure()
        run: terraform apply -var env=rollback
```

### Example 4: Ansible Playbook for Configuration

```yaml
# ansible/site.yml
- hosts: production
  serial: 1  # Rolling update
  
  pre_tasks:
    - name: Drain node
      shell: kubectl drain {{ ansible_hostname }}
  
  tasks:
    - name: Update docker
      apt:
        name: docker.io
        state: latest
    
    - name: Restart kubelet
      systemd:
        name: kubelet
        state: restarted
  
  post_tasks:
    - name: Uncordon node
      shell: kubectl uncordon {{ ansible_hostname }}
    
    - name: Wait for ready
      shell: kubectl wait --for=condition=ready node/{{ ansible_hostname }}
```

---

## 📈 Learning Path Progression

```
Foundation
    ↓
Google Cloud Basics (gcp/01-services.md)
    ↓
Infrastructure-as-Code (terraform/01-fundamentals.md)
    ↓
Container Orchestration (kubernetes/01-fundamentals.md)
    ↓
Package Management (helm/01-fundamentals.md)
    ↓
Configuration Management (ansible/01-fundamentals.md)
    ↓
Automation & Safety (cicd/01-fundamentals.md)
    ↓
Advanced: Multi-region HA, Cost Optimization, Security
    ↓
System Design (End-to-end projects)
    ↓
Interview Ready!
```

---

## ⏱️ Time Investment

| Component | Time | Value |
|-----------|------|-------|
| Read all core files | 8-10 hours | Deep technical understanding |
| Study interview Q&As | 2-3 hours | Interview readiness |
| Practice scenarios | 3-4 hours | Build confidence |
| Hands-on labs | 4-6 hours | Real implementation |
| Mock interviews | 2-3 hours | Simulation practice |
| **Total** | **19-26 hours** | **Interview ready** |

---

## 📋 Pre-Interview Checklist

- [ ] Can draw Kubernetes control plane from memory
- [ ] Understand Terraform state and why it's critical
- [ ] Know 3 deployment strategies (blue-green, canary, rolling)
- [ ] Can explain Helm chart rendering
- [ ] Know GCP service decision trees
- [ ] Understand IAM and least-privilege
- [ ] Can design multi-region HA system
- [ ] Know common mistakes for each technology
- [ ] Can troubleshoot pipeline failures
- [ ] Understand cost optimization techniques

---

## 🎓 Prerequisites

- Basic understanding of cloud platforms (AWS/GCP/Azure)
- Familiar with Linux/Unix command line
- Basic networking knowledge (ports, DNS, TCP/IP)
- Experience with Docker/containers
- Comfortable with Git and version control

**Don't need:** Advanced system administration or networking expertise

---

## 🔗 File Navigation Quick Links

**By Technology Type:**
- Terraform: [01-fundamentals](./terraform/01-fundamentals.md) → [02-state_management](./terraform/02-state_management.md) → [06-best_practices](./terraform/06-best_practices.md)
- Kubernetes: [01-fundamentals](./kubernetes/01-fundamentals.md) (foundation for everything)
- Helm: [01-fundamentals](./helm/01-fundamentals.md) (uses Kubernetes)
- Ansible: [01-fundamentals](./ansible/01-fundamentals.md) (configures infrastructure)
- CI/CD: [01-fundamentals](./cicd/01-fundamentals.md) (orchestrates everything)
- GCP: [01-services](./gcp/01-services.md) (integrates everything)

**By Interview Preparation:**
- Start with: `DEVOPS-GUIDE.md` (overview and learning paths)
- Core skills: `terraform/02-state_management.md` and `kubernetes/01-fundamentals.md`
- Advanced patterns: `terraform/06-best_practices.md` and `cicd/01-fundamentals.md`
- System design: Practice scenarios in each file

**By Real-world Use Case:**
- Deploy microservices: Kubernetes → Helm → CI/CD → GCP
- Manage infrastructure: Terraform → terraform/06-best_practices.md
- Configure servers: Ansible → ansible/01-fundamentals.md
- Monitor production: gcp/01-services.md (Monitoring section)

---

## 🌟 Key Insights

### **State is Everything (Terraform)**
The state file is your infrastructure's single source of truth. One corrupted byte can cause hours of pain. Understand remote backends, locking, and recovery.

### **Reconciliation is Magic (Kubernetes)**
Controllers continuously compare desired state (what you declared) vs actual state (what's running). This is how self-healing works.

### **Templates Enable Scale (Helm)**
Instead of managing hundreds of YAML files, use templating to generate them. This is production-grade infrastructure.

### **Idempotency Prevents Chaos (Ansible)**
Running a playbook 10 times should be safe. Idempotent operations let you retry safely without fear.

### **Pipelines Multiply Your Leverage (CI/CD)**
Automate everything you'd manually do. Every manual step is an opportunity for human error.

### **Services Are Building Blocks (GCP)**
Don't build what Google already built. Work with managed services; they're usually better than DIY solutions.

---

## 💼 Real Interview Questions You'll Face

1. **"Explain your experience with Terraform in production. How do you handle state?"**
2. **"Design a Kubernetes cluster for a SaaS platform with 1M users worldwide."**
3. **"How would you implement a zero-downtime deployment?"**
4. **"We have a state corruption issue. How do you recover?"**
5. **"Compare GKE vs Cloud Run. When would you use each?"**
6. **"Design a disaster recovery strategy for multi-region failover."**
7. **"How do you test infrastructure changes safely?"**
8. **"Explain your CI/CD pipeline. How do you handle failures?"**
9. **"You're on-call, production is down. Walk me through your debugging."**
10. **"How do you optimize cloud costs?**

Each of these is covered in detail with model answers in the respective files.

---

## 🚀 Ready to Get Started?

1. **Quick Start (2 hours):** Read `DEVOPS-GUIDE.md` then `terraform/02-state_management.md`
2. **Full Journey (2 weeks):** Follow the 8-week plan in `DEVOPS-GUIDE.md`
3. **Deep Dive (1 month):** Read all files, practice, build projects

---

## 📈 Content Roadmap

**Currently Complete:**
- ✅ Terraform (6 comprehensive files)
- ✅ Kubernetes fundamentals
- ✅ Helm fundamentals  
- ✅ Ansible fundamentals
- ✅ CI/CD fundamentals
- ✅ GCP services overview
- ✅ DEVOPS-GUIDE.md (master index)

**Planned Extensions:**
- [ ] Kubernetes advanced (storage, security, networking)
- [ ] Helm advanced (hooks, dependencies, testing)
- [ ] Ansible advanced (vault, testing, dynamic inventory)
- [ ] Cloud Build deep dive (GCP-specific)
- [ ] GCP networking deep dive
- [ ] GCP IAM and security advanced
- [ ] End-to-end project walkthroughs

---

## 🎓 Good Luck!

This guide represents production-grade DevOps knowledge. Master these concepts and you'll be well-prepared for intermediate to senior DevOps interviews.

**Last Updated:** April 2026  
**Total Content:** 8,000+ lines  
**Interview Questions:** 100+ with detailed answers

Happy studying! 🚀
