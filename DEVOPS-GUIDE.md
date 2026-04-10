# DevOps Interview Preparation - Complete Learning Guide

## Repository Structure

This is a **comprehensive DevOps learning system** designed for technical interview preparation, covering intermediate to advanced topics with real-world enterprise patterns.

---

## 📚 Learning Structure

### **[Terraform](./terraform/)**  - Infrastructure-as-Code
*Master Terraform for managing cloud infrastructure with enterprise patterns*

- **[01-fundamentals.md](./terraform/fundamentals.md)** - Core concepts, execution flow, state management
  - Terraform architecture, provider plugins, state file structure
  - When to use Terraform vs manual approaches
  - Interview questions on state management and disaster recovery

- **[02-state_management.md](./terraform/state_management.md)** - **CRITICAL** - State is the source of truth
  - State file internals, locking mechanisms, remote backends
  - Team collaboration patterns, fixing state corruption
  - Enterprise patterns for state isolation

- **[03-modules.md](./terraform/modules.md)** - Reusable infrastructure patterns
  - Module design principles, composition patterns, versioning
  - Real-world multi-cluster GKE setup with modules
  - Common mistakes: circular dependencies, tight coupling

- **[04-remote_backend.md](./terraform/remote_backend.md)** - Production state storage
  - Backend protocols, GCS/S3 backends, encryption
  - State locking deep dive, multi-environment backends
  - Disaster recovery and state corruption handling

- **[05-terraform_gcp.md](./terraform/terraform_gcp.md)** - GCP-specific patterns
  - Google provider internals, authentication, resource ID tracking
  - Multi-project infrastructure, service accounts, IAM
  - Cost optimization for different environments

- **[06-best_practices.md](./terraform/best_practices.md)** - Production-grade patterns
  - Project structure, naming conventions, code validation
  - Security scanning, documentation, operational runbooks
  - CI/CD integration for infrastructure changes

---

### **[Kubernetes](./kubernetes/)**  - Container Orchestration
*Deep dive into Kubernetes architecture and production patterns*

- **[01-fundamentals.md](./kubernetes/01-fundamentals.md)** - Architecture and operations
  - Control plane components, controller reconciliation loop
  - Pod lifecycle, resource scheduling, self-healing
  - When to use Kubernetes vs simpler alternatives

---

### **[Helm](./helm/)**  - Kubernetes Package Manager
*Deploy and manage Kubernetes applications at scale*

Files to create:
- `01-fundamentals.md` - Values, templates, releases
- `02-helm_charts.md` - Design patterns, dependencies, hooks
- `03-deployments.md` - Production deployments, GitOps integration

---

### **[Ansible](./ansible/)**  - Configuration Management & Automation
*Automate infrastructure configuration and deployment*

Files to create:
- `01-fundamentals.md` - Playbooks, inventory, modules
- `02-advanced.md` - Roles, templates, error handling
- `03-production.md` - Vault integration, test strategies

---

### **[CI/CD](./cicd/)**  - Continuous Integration & Deployment
*Build, test, and deploy automation pipelines*

Files to create:
- `01-fundamentals.md` - Pipeline concepts, GitOps workflows
- `02-github_actions.md` - Workflows, secrets, matrix builds
- `03-cloud_build.md` - GCP Cloud Build, infrastructure deployment

---

### **[GCP](./gcp/)**  - Google Cloud Platform Services
*Comprehensive GCP services for production infrastructure*

Files to create:
- `01-compute.md` - GCE, GKE, Cloud Run, App Engine
- `02-data.md` - Cloud SQL, Cloud Storage, BigQuery, Pub/Sub
- `03-networking.md` - VPC, Load Balancing, Cloud Interconnect
- `04-iam_security.md` - Service accounts, IAM roles, VPC Security
- `05-operations.md` - Logging, Monitoring, Cloud Trace

---

## 🎯 How to Use This Guide

### For Interview Preparation

**Week 1: Terraform Fundamentals**
- [ ] Read: fundamentals.md + state_management.md
- [ ] Practice: Set up local Terraform project with remote backend
- [ ] Interview prep: Know state file structure, common mistakes, disaster recovery

**Week 2: Terraform Advanced**
- [ ] Read: modules.md + best_practices.md + terraform_gcp.md
- [ ] Practice: Refactor code into modules, set up multi-environment structure
- [ ] Interview prep: Design patterns, module composition, team workflows

**Week 3: Kubernetes**
- [ ] Read: kubernetes fundamentals
- [ ] Practice: Deploy apps to GKE, set up health checks, scaling
- [ ] Interview prep: Pod scheduling, node failures, cluster architecture

**Week 4: Helm + Deployment**
- [ ] Read: Helm fundamentals + deployments
- [ ] Practice: Create Helm charts, deploy apps with Helm
- [ ] Interview prep: Template patterns, release management

**Week 5: CI/CD + Automation**
- [ ] Read: CI/CD fundamentals + platform-specific guides
- [ ] Practice: Build GitHub Actions or Cloud Build pipelines
- [ ] Interview prep: Infrastructure as code in pipelines, GitOps

**Week 6: GCP Integration**
- [ ] Read: GCP services guides
- [ ] Practice: Set up multi-service GCP infrastructure with Terraform
- [ ] Interview prep: GCP-specific patterns, cost optimization

**Week 7-8: Practice & Review**
- [ ] Complete end-to-end project: Terraform + Kubernetes + CI/CD
- [ ] Review all interview questions
- [ ] Practice explanations and design patterns

---

## 🔥 Common Interview Scenarios

### Scenario 1: "Design infrastructure for a 3-tier web application"

**Expected coverage:**
1. ✅ Network architecture (VPC, subnets, routing)
2. ✅ Compute layer (GKE cluster with auto-scaling)
3. ✅ Database layer (Cloud SQL with backups)
4. ✅ Infrastructure-as-code (Terraform)
5. ✅ Deployment automation (CI/CD)
6. ✅ Monitoring and logging
7. ✅ Security (IAM, network policies, secrets management)

**Learn from:** terraform_gcp.md + kubernetes fundamentals + cicd fundamentals

---

### Scenario 2: "Handle infrastructure state corruption in production"

**Expected coverage:**
1. ✅ Understand state file structure
2. ✅ Use remote backend versioning
3. ✅ Import/recover from cloud resources
4. ✅ Test recovery procedures regularly
5. ✅ Communicate with team about impact

**Learn from:** terraform/state_management.md (Issue #1)

---

### Scenario 3: "Scale Kubernetes deployment from 3 to 100 pods overnight"

**Expected coverage:**
1. ✅ Pod resource requests/limits
2. ✅ Horizontal Pod Autoscaling (HPA)
3. ✅ Cluster autoscaling (add nodes)
4. ✅ Rollout strategy (don't hammer with 97 new pods)
5. ✅ Monitoring for performance issues

**Learn from:** kubernetes fundamentals + helm deployments

---

### Scenario 4: "Team workflow for infrastructure changes"

**Expected coverage:**
1. ✅ Version control (Git workflow)
2. ✅ Infrastructure review (terraform plan in PR)
3. ✅ Automated testing (validation, linting)
4. ✅ Approval process (2-person rule for prod)
5. ✅ Automated apply (merge triggers deployment)
6. ✅ Rollback capability

**Learn from:** terraform/best_practices.md + cicd fundamentals

---

## 📊 Interview Question By Topic

### Terraform
- [ ] Explain state file internals and why it's critical
- [ ] How would you handle state file corruption?
- [ ] Design multi-environment infrastructure with modules
- [ ] How do you prevent concurrent apply disasters?
- [ ] What's your disaster recovery strategy?

### Kubernetes
- [ ] Explain pod scheduling and the scheduler algorithm
- [ ] What happens when a node fails?
- [ ] Design a production-grade deployment with HA
- [ ] How do you handle resource contention?
- [ ] Explain the reconciliation loop

### CI/CD
- [ ] Design an infrastructure deployment pipeline
- [ ] How do you prevent bad infrastructure changes?
- [ ] What's your strategy for rolling updates?
- [ ] How do you handle secrets in CI/CD?

### System Design
- [ ] Architecture for a distributed microservices system
- [ ] Multi-region setup with disaster recovery
- [ ] Internal tools infrastructure for 1000 engineers
- [ ] Infrastructure for real-time data processing

---

## 🛠️ Hands-on Projects

### Project 1: Deploy Web App from Scratch (Week 2-3)
**Objective:** End-to-end infrastructure deployment

Components:
1. Terraform: VPC + GKE cluster + Cloud SQL
2. Helm: Package web app
3. CI/CD: GitHub Actions to deploy on merge
4. Monitoring: Prometheus + Grafana
5. Backup: Test disaster recovery

Success criteria:
- [ ] Deploy using Terraform
- [ ] App runs on GKE
- [ ] Database backed up
- [ ] Monitoring shows metrics
- [ ] DR test succeeds

---

### Project 2: Multi-environment Setup (Week 4)
**Objective:** Manage dev, staging, prod consistently

Components:
1. Terraform: Same code, different vars for each environment
2. State files: Separate per environment
3. CI/CD: Different approval levels per environment
4. Team access: Different permissions per environment

Success criteria:
- [ ] Deploy to all 3 environments from one PR
- [ ] Prod requires 2-person approval
- [ ] Can easily add new environment
- [ ] No copy-paste code

---

### Project 3: Helm Chart Development (Week 5)
**Objective:** Package app as reusable Helm chart

Components:
1. Create chart with templates
2. Define values for different environments
3. Implement hooks for database migrations
4. Test with different configurations

Success criteria:
- [ ] Deploy multiple app versions with one chart
- [ ] Each version independently configurable
- [ ] Rollback works correctly
- [ ] Database migrations run automatically

---

## 🚀 Interview Preparation Checklist

### Knowledge
- [ ] Understand terraform state file structure completely
- [ ] Know Kubernetes architecture (control plane, nodes, pods)
- [ ] Explain HPA/VPA and cluster autoscaling
- [ ] Describe CI/CD pipeline from code to production
- [ ] Know GCP services and when to use each

### Practical
- [ ] Set up Terraform project with remote backend
- [ ] Deploy app to GKE with Helm
- [ ] Create GitHub Actions CI/CD pipeline
- [ ] Implement monitoring and logging
- [ ] Test disaster recovery scenarios

### Interview Ready
- [ ] Explain architecture decisions clearly
- [ ] Know tradeoffs (cost vs reliability, simplicity vs features)
- [ ] Handle follow-up questions about edge cases
- [ ] Discuss team workflows and operational concerns
- [ ] Ask clarifying questions about requirements

---

## 📖 Key Concepts Summary

### Terraform
- **State = Single source of truth** - Everything flows from state
- **Remote backend + locking** - Required for teams
- **Modules for reusability** - DRY principle applies
- **Workspace for environments** - One code, multiple states
- **Drift detection** - Manual changes break assumptions

### Kubernetes
- **Declarative model** - State -> desired state -> reconciliation
- **Controllers fix things** - Self-healing through continuous reconciliation
- **Pods are ephemeral** - Don't store anything locally
- **Resource isolation** - CPU/memory limits prevent noisy neighbors
- **Multi-replica = HA** - Survive node/pod failures

### CI/CD
- **Shift-left testing** - Fail fast, early in pipeline
- **GitOps** - Git as source of truth for infrastructure
- **Immutable infrastructure** - No manual changes, redeploy instead
- **Secrets management** - Never in code or logs
- **Rollback capability** - Always be able to go back

### GCP
- **Managed services** - Let Google operate the control plane
- **IAM everything** - Least privilege for all resources
- **Network isolation** - VPCs, VPC Service Controls
- **Observability built-in** - Cloud Logging, Cloud Monitoring
- **Cost optimization** - Reserved instances, preemptible nodes

---

## 💡 Pro Tips

1. **When explaining concepts:** Start simple, then dive deep
   - Bad: "etcd is a distributed KV store using Raft"
   - Good: "etcd stores K8s state. Like a database. Replicated for HA."

2. **Interview questions:** Often have multiple valid answers
   - Show tradeoffs
   - Explain why you chose your approach
   - Discuss when alternatives make sense

3. **Real-world context:** Always tie back to operations
   - "This is critical because in production..."
   - "We learned this when our database crashed..."

4. **Practice explaining:**
   - Record yourself explaining concepts
   - Explain to non-technical person
   - Watch for unclear parts

5. **System design focus:**
   - Start with requirements
   - Identify risks and mitigation
   - Scale as needed
   - Monitor and iterate

---

## 📞 Getting Help

Found an issue or want to contribute your learnings?

Each topic has:
- Comprehensive explanations with why, not just how
- Real-world DevOps scenarios
- Common mistakes and how to avoid them
- Interview questions with good answers
- Debugging techniques and solutions

---

## License & Usage

This is a learning resource for DevOps interview preparation. Feel free to:
- Clone and customize for your team
- Add your own scenarios and learnings
- Share with people preparing for interviews
- Contribute feedback and improvements

---

**Last Updated:** April 2026  
**Scope:** Intermediate to Advanced DevOps  
**Focus:** Production-grade infrastructure management
