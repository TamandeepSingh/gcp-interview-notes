# Terraform Fundamentals

## 1. Core Concept

Terraform is an **Infrastructure-as-Code (IaC) tool** that enables declarative provisioning of cloud resources across multiple providers. Unlike imperative scripts that describe *how* to build infrastructure, Terraform uses a **declarative model** where you describe *what* your infrastructure should look like, and Terraform figures out the steps needed to achieve that state.

**Key Principles:**
- **Declarative language**: Write desired state, not steps
- **Provider-agnostic**: Works with AWS, GCP, Azure, Kubernetes, etc.
- **State management**: Tracks actual vs desired state
- **Idempotent**: Running twice produces same result
- **Plan before apply**: See changes before they happen

---

## 2. Internal Working (VERY IMPORTANT)

### Terraform Execution Flow

```
1. WRITE: Developer writes .tf files with resource definitions
2. PARSE: HCL parser converts files into AST (Abstract Syntax Tree)
3. VALIDATE: Schema validation against provider schemas
4. GRAPH: Creates dependency graph of all resources
5. PLAN: 
   - Reads current state from tfstate
   - Calls provider APIs to get actual resource state
   - Calculates diff (what needs to change)
   - Generates execution plan
6. APPLY:
   - Executes changes in dependency order (respecting graph)
   - Updates tfstate file with new resource IDs and attributes
   - Handles state locking (distributed lock mechanism)
7. STATE UPDATE: tfstate file reflects actual infrastructure
```

### State File Structure (Low-level)

```json
{
  "version": 4,
  "terraform_version": "1.5.0",
  "serial": 42,
  "lineage": "unique-id",
  "outputs": {...},
  "resources": [
    {
      "type": "google_compute_instance",
      "name": "web_server",
      "provider": "provider[\"registry.terraform.io/-/google\"]",
      "instances": [
        {
          "schema_version": 6,
          "attributes": {
            "id": "projects/my-project/zones/us-central1-a/instances/web-1",
            "name": "web-1",
            "zone": "us-central1-a",
            "machine_type": "e2-medium",
            ...computed attributes from API...
          }
        }
      ]
    }
  ]
}
```

### Provider Plugin Architecture

- Terraform core is a **state machine** that orchestrates providers
- Providers are **gRPC plugins** that communicate with Terraform core
- When you run `terraform init`, it downloads provider binaries for your OS
- Each provider implements CRUD operations for resources it manages
- Terraform core calls provider's `Read`, `Create`, `Update`, `Delete` operations

### Critical Internal Mechanisms

**Dependency Resolution:**
```
1. Terraform builds a directed acyclic graph (DAG)
2. Nodes = resources, edges = dependencies
3. Uses topological sort to determine execution order
4. Parallelizes resources with no dependencies (default -parallelism=10)
```

**State Refresh:** Before planning, Terraform reads actual resource state from cloud provider APIs to detect drift:
```
For each resource in state:
  → Call provider's ReadResource API
  → Compare returned attributes with state file
  → Identify any manual changes (drift)
```

---

## 3. Why Terraform is Needed

### Problem It Solves

1. **Manual Infrastructure is Unmaintainable**
   - Documented disasters of "someone clicked console and nobody knows what they changed"
   - No audit trail
   - Impossible to replicate environment for testing or DR

2. **Cloud Console Doesn't Scale**
   - Can't deploy 50 instances across 3 regions using UI
   - No version control
   - Impossible to review changes before deployment
   - Can't automate complex multi-resource deployments

3. **Imperative Scripts Lack Idempotency**
   - Bash script that creates instance fails on second run
   - PowerShell script doesn't detect existing resources
   - Hard to know current state without manual inspection

4. **Multi-cloud Consistency**
   - Different CLI commands per cloud provider
   - No unified workflow across AWS, GCP, Azure
   - Terraform uses same workflow across all

5. **Infrastructure Versioning**
   - Git becomes source of truth for infrastructure
   - Code review process for infrastructure changes
   - Rollback capability (destroy and recreate)
   - Audit trail of who changed what when

### Real-world Impact

**Before Terraform:**
- Infrastructure deployment took 2 weeks (manual steps, documentation)
- Disaster recovery took 4+ hours (manual recreation)
- No-one really knew what was in production

**After Terraform:**
- New environment deployed in 15 minutes
- DR environment up in 5 minutes
- Complete audit trail in Git
- Team consensus on infrastructure changes via PRs

---

## 4. When to Use vs Not Use

### Use Terraform When:

- ✅ Managing infrastructure lasting months or longer
- ✅ Multiple environments (dev, staging, prod)
- ✅ Team needs infrastructure versioning/audit trail
- ✅ Multi-cloud or multi-provider scenarios
- ✅ Complex interdependencies between resources
- ✅ Infrastructure changes need review/approval process
- ✅ Need to implement GitOps workflow

### Don't Use Terraform When:

- ❌ Temporary lab/proof-of-concept (< 1 day)
  - *Use: Cloud console or provider CLI directly*
- ❌ One-time batch operations
  - *Use: Cloud SDK scripts*
- ❌ No version control available
  - *Use: Cloud console*
- ❌ Heavy dynamic resource creation (e.g., autoscaling creating 1000s of instances)
  - *Use: Provider's auto-scaling service directly*
- ❌ Teams unfamiliar with version control
  - *Use: Ansible or imperative scripts*

### Gray Area Scenarios

**Partially IaC with Terraform + Scripts:**
- Use Terraform for static infrastructure core
- Use cloud provider APIs/scripts for dynamic operations
- Example: Terraform creates GKE cluster, separate Helm/kubectl for adding workloads

---

## 5. Real-world DevOps Usage

### Enterprise Scenario: Multi-region Deployment

**Requirement:** Deploy application across US and EU with separate databases, load balancers, and monitoring.

```hcl
# Root module
terraform {
  required_version = ">= 1.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# Input variables for flexibility
variable "regions" {
  type = list(object({
    name          = string
    zone          = string
    instance_type = string
  }))
}

# Deploy infrastructure for each region
module "regional_deployment" {
  for_each = { for r in var.regions : r.name => r }
  
  source = "./modules/regional"
  
  region        = each.value.name
  zone          = each.value.zone
  instance_type = each.value.instance_type
  app_version   = var.app_version
}

# Global load balancer
resource "google_compute_global_forwarding_rule" "global_lb" {
  name                  = "global-lb"
  load_balancing_scheme = "EXTERNAL"
  port_range            = "443"
  target_https_proxy    = google_compute_target_https_proxy.proxy.id
}

output "deployment_status" {
  value = {
    for region, deployment in module.regional_deployment :
    region => {
      instances = deployment.instance_ids
      lb_ip     = deployment.load_balancer_ip
    }
  }
}
```

**Actual execution:**
```bash
# Plan shows all changes before applying
terraform plan -out=tfplan

# Review and approve plan
terraform apply tfplan

# Check actual state
terraform state show google_compute_instance.app
```

### DevOps Workflow Integration

1. **Developer:** Commits Terraform code to Git
2. **CI Pipeline:** Runs `terraform plan`, posts plan as PR comment
3. **Team Lead:** Reviews changes, approves PR
4. **CI Pipeline:** On PR merge, runs `terraform apply`
5. **Monitoring:** Detects drift, alerts if manual changes made

---

## 6. Architecture Thinking

### Design Principles

**Principle 1: Environment Parity**
- Same Terraform code for dev, staging, prod
- Use variables/workspaces to vary only what differs
- Reduces surprises when promoting to production

**Principle 2: Modularity**
- Don't put everything in root/main.tf
- Create reusable modules for common patterns
- Example: `modules/gke-cluster`, `modules/cloud-sql`, `modules/network`

**Principle 3: State Isolation**
- Separate state files by environment (state file per project)
- Don't manage 100 resources in one state file
- Split: `prod-infra`, `prod-k8s`, `prod-databases`

**Principle 4: Single Responsibility**
- One root module manages one logical system
- GKE cluster root module manages: cluster, node pools, networking
- App deployment root module manages: deployments, services, ingress
- Keep concerns separated for easier reasoning

### Typical Enterprise Structure

```
terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf          (calls modules)
│   │   ├── variables.tf      (dev-specific values)
│   │   ├── terraform.tfvars  (dev config)
│   │   └── state/            (state files, gitignored)
│   ├── staging/
│   └── prod/
├── modules/
│   ├── gke-cluster/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── versions.tf
│   ├── cloud-sql/
│   ├── vpc-network/
│   └── monitoring/
└── global/
    ├── terraform.tf         (provider config)
    └── versions.tf
```

---

## 7. Common Mistakes

### Mistake 1: State File in Git (CRITICAL)

❌ **Wrong:**
```bash
git add terraform.tfstate
git commit -m "backup state"
```

**Why:** State files contain secrets (DB passwords, API keys), commit history is permanent

✅ **Right:**
```bash
# .gitignore
*.tfstate
*.tfstate.*
*.tfstate.backup
```

### Mistake 2: Not Using Remote State

❌ **Problem:**
- Local state file on developer's laptop
- Another developer can't plan/apply
- When developer leaves, state is lost
- No audit trail of who changed what

✅ **Solution:**
```hcl
terraform {
  backend "gcs" {
    bucket = "tf-state-prod"
    prefix = "infrastructure"
  }
}
```

### Mistake 3: Manual Resource Creation and Terraform

❌ **Scenario:**
```
1. Create VPC via Terraform
2. Developer manually adds firewall rule via console
3. Run terraform plan → doesn't show the firewall rule
4. Disaster when teammate runs terraform destroy
```

**Why:** Terraform only knows what it created. Manual changes become "drift".

✅ **Solution:** Use `terraform import` to bring manual resources under management:
```bash
terraform import google_compute_firewall.allow_ssh \
  projects/my-project/global/firewalls/allow-ssh
```

### Mistake 4: Modifying Tfstate Directly

❌ **Wrong:**
```bash
# Never do this
cat terraform.tfstate | jq '.resources[0].instances[0].attributes.id = "new-id"' > terraform.tfstate
```

**Why:** Breaks internal consistency, breaks state locking, corrupts state

✅ **Right:**
```bash
# Use terraform state command
terraform state mv google_compute_instance.old \
  google_compute_instance.new
terraform state rm google_compute_instance_to_delete
```

### Mistake 5: Destroying Production without Confirmation

❌ **Wrong:**
```bash
terraform destroy -auto-approve
# Oops, deleted production database
```

✅ **Right:**
```hcl
# Add lifecycle rule for critical resources
resource "google_cloudsql_instance" "prod_db" {
  name = "prod-database"
  
  lifecycle {
    prevent_destroy = true  # Blocks terraform destroy
  }
}

# Override in CI only with explicit flag
terraform destroy -replace=resource_id
```

### Mistake 6: Giant Monolithic State Files

❌ **Problem:**
- 500 resources in one terraform state file
- Entire prod infrastructure in one plan
- Small mistake = large blast radius
- Plan takes 5 minutes

✅ **Solution:**
```
State boundary principles:
- One state per logical boundary
- Example: prod-network (VPCs, subnets, routing)
- Example: prod-gke (cluster, node pools, storage)
- Reference outputs from other states using data sources
```

### Mistake 7: Not Handling Sensitive Data

❌ **Wrong:**
```hcl
variable "db_password" {
  type = string
}

output "database_connection_string" {
  value = "user:${var.db_password}@host"  # Logs show password!
}
```

✅ **Right:**
```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

output "database_connection_string" {
  value     = "user:${var.db_password}@host"
  sensitive = true  # Blocks output from terraform apply logs
}
```

---

## 8. Interview Questions (Scenario-based)

### Q1: State Corruption Scenario
**Interviewer:** "During maintenance, your state file got corrupted (JSON syntax error). Your team needs to deploy changes NOW. Walk through your resolution."

**Good Answer:**
1. Identify recent good state backup (remote backend has history)
2. Use `terraform state pull` to get current state
3. If backup exists: `terraform state push backup.tfstate`
4. Run `terraform plan` to verify state is recoverable
5. If unrecoverable, use `terraform state rm` on each resource, then `terraform import` to rebuild from actual cloud state
6. Post-incident: Implement backup lock and validation

### Q2: Team Workflow Under Pressure
**Interviewer:** "Two teammates are modifying same root Terraform module. Without coordination, what goes wrong? How do you prevent it?"

**Good Answer:**
- **Problem:** Without state locking, both run apply → second one wins, first one's changes overwritten
- **Prevention:** Remote state backend with state locking (default with GCS, S3)
- **Team workflow:** PR review + CI pipeline running `terraform plan` prevents concurrent applies
- **Verification:** `terraform state list` shows current resources, `terraform state lock` shows active locks

### Q3: Module Design Decision
**Interviewer:** "You need to create 20 microservices in a GKE cluster. Each is identical except config. Do you create 20 separate resource definitions or one module? Why?"

**Good Answer:**
- **One reusable module** with input variables
- Reduces code duplication
- One place to update if best practices change
- Module versioning possible
- Example:
  ```hcl
  module "microservice" {
    for_each = toset(var.services)
    source = "./modules/microservice"
    
    name    = each.key
    image   = "gcr.io/${each.key}:latest"
    replicas = var.replicas[each.key]
  }
  ```

### Q4: Disaster Recovery Timing
**Interviewer:** "Your prod database was accidentally deleted. Using backups, you need to restore DB and all associated infrastructure. Estimate time with Terraform vs manual."

**Good Answer:**
- **With Terraform:** 5-10 minutes
  - Restore DB backup from cloud backup
  - Run `terraform apply` to recreate compute/networking
  - Full infrastructure online
- **Manual:** 1-2 hours
  - Restore DB manually
  - Manually recreate compute, networking, security rules, monitoring
  - Error-prone, forget something
- **Terraform enables 10x faster DR**

### Q5: Drift Detection
**Interviewer:** "Your monitoring system detects CPU spike on prod. You suspect someone manually scaled instances via console. How do you detect and fix drift in Terraform?"

**Good Answer:**
1. Run `terraform plan` → shows manual instance
2. Two options:
   - **Option A (Keep manual):** `terraform import` to bring under management
   - **Option B (Revert):** Run `terraform apply` to destroy and recreate per code
3. Implement drift detection job:
   - Run `terraform plan` daily
   - Alert if resources not in plan (manual changes)
4. Use `prevent_manual_changes` policy if provider supports

---

## 9. Debugging Issues

### Issue 1: "State lock timeout waiting for lock"

```
Error: Error acquiring the state lock

Error message includes:
"Failed to acquire lock: timeout waiting for HTTP response"
```

**Diagnosis:**
```bash
# List current locks (in GCS backend)
gsutil ls gs://tf-state-bucket/.terraform.lock.d/

# Check age - if > 1 hour, likely stale
stat .terraform.lock.d/lock
```

**Causes:**
1. Previous `terraform apply` crashed without releasing lock
2. Network timeout left lock hanging
3. Multiple people running apply simultaneously

**Solution:**
```bash
# Force unlock (use carefully!)
terraform force-unlock <LOCK_ID>

# Or if really stuck, remove from backend manually
gsutil rm gs://tf-state-bucket/.terraform.lock.d/lock
```

### Issue 2: "Resource already exists"

```
Error: google_compute_instance.web: Error creating instance: 
resource instance already exists
```

**Diagnosis:**
```bash
# Check if it's in state
terraform state list | grep web

# If not in state but exists in cloud
terraform state show google_compute_instance.web
# (returns nothing)
```

**Cause:** Resource created manually or by another Terraform config

**Solution:**
```bash
# Import existing resource
terraform import google_compute_instance.web \
  projects/my-project/zones/us-central1-a/instances/web

# Now terraform knows about it
terraform plan  # Should show no changes
```

### Issue 3: "Plan shows changes but nothing actually changed"

```bash
terraform plan
# Output: 1 to add, 0 to modify, 0 to destroy
# But running again shows same changes (infinite loop)
```

**Causes:**
1. Dynamic fields computed each time (timestamps, random values)
2. API returning different values on each read

**Diagnosis:**
```bash
# Run plan twice, compare
terraform plan -out=plan1
terraform plan -out=plan2
diff plan1 plan2

# Check state
terraform state show resource_name
# Look for attributes that change each read
```

**Solution:**
```hcl
# Option 1: Ignore computed changes
resource "google_compute_address" "static" {
  name = "static-ip"
  
  lifecycle {
    ignore_changes = [labels["last_updated"]]  # Ignore if it changes on read
  }
}

# Option 2: Mark field as computed (for custom providers)
variable "config" {
  type = object({
    id = string  # Computed by provider
  })
}
```

### Issue 4: "Terraform apply partially failed"

```
Error: Error applying plan:

google_compute_instance.web: Creating... (0s elapsed)
google_compute_instance.worker: Creating... (2s elapsed)
google_compute_instance.worker: Creation failed: quota exceeded
google_compute_instance.web: Still creating... (5s elapsed)
```

**Diagnosis:**
```bash
terraform state list
# web is in state (was created successfully before error)
# worker is NOT in state (failed before being added)
```

**Understand:** Terraform already created `web` but failed on `worker`

**Solution:**
1. Fix quota issue in cloud console
2. Run `terraform apply` again
3. Terraform detects `web` already exists, focuses on `worker`
4. Completes partial application

**Prevention:** Use `parallelism` flag to apply serially:
```bash
terraform apply -parallelism=1  # One resource at a time, easier to debug
```

### Issue 5: "State corruption - invalid JSON"

```
Error: Unable to read state: error decoding state file: invalid character
```

**Diagnosis:**
```bash
# Check if state file parseable
terraform state pull | jq .
# Returns JSON parse error
```

**Recovery:**
```bash
# Restore from backend history (GCS makes versions)
# List versions
gsutil ls -L gs://tf-state-bucket/default.tfstate

# Copy old version
gsutil cp gs://tf-state-bucket/default.tfstate#GENERATION \
  gs://tf-state-bucket/default.tfstate

# Or rebuild from cloud resources
$ for resource in $(terraform state list); do
  terraform state rm $resource
done

# Then import all resources back
terraform import ...  # One by one
```

---

## 10. Advanced Insights

### Scaling Considerations for Large Infrastructure

**Challenge 1: State File Size**
- As infrastructure grows, state file becomes large (50MB+)
- Slow to read/write
- Network transfers become bottleneck

**Solution: Terraform Cloud/Enterprise State Versioning**
```
- Zipped state uploads
- Automatic cleanup of old versions
- Concurrent-safe operations
- State retention policies
```

**Challenge 2: Plan Time Explosion**
- 1000+ resources = 10+ minute plans
- Parallelism limits due to API rate limiting
- Developer experience suffers

**Solution: Targeted Planning**
```bash
# Don't plan 1000 resources, target specific ones
terraform plan -target=module.apps -target=module.networking

# Or use workspace to split concerns
terraform workspace new prod-apps
terraform workspace new prod-infra
# Each uses own state file, faster plans
```

### Team Workflow at Scale

**Enterprise Pattern: State File Per Team**
```
Platform Team:
  - State: infrastructure/networking/cloud-sql
  - Manages: VPCs, FirewallRules, Databases
  - Approval: 2-person approval required

App Team A:
  - State: apps/team-a
  - Manages: GKE deployments, services, configmaps
  - References: database_endpoint from platform state via data source

App Team B:
  - State: apps/team-b
  - Same setup independently
```

**Implementation:**
```hcl
# App team's code references platform team's state
data "terraform_remote_state" "platform" {
  backend = "gcs"
  config = {
    bucket = "tf-state-platform"
    prefix = "database"
  }
}

resource "kubernetes_secret" "db_creds" {
  metadata {
    name = "database"
  }
  data = {
    connection_string = data.terraform_remote_state.platform.outputs.cloudsql_endpoint
  }
}
```

**Advantages:**
- Different teams can apply independently
- No waiting for other teams
- Failure in app deployment doesn't affect infrastructure
- Clear separation of concerns

### Cost Optimization Patterns

**Pattern 1: Schedule-Based Resource Scaling**
```hcl
variable "environment_schedule" {
  type = object({
    production  = bool
    development = bool
  })
  
  default = {
    production  = true   # Always on
    development = false  # Off after hours
  }
}

resource "google_compute_instance" "dev" {
  count = var.environment_schedule.development ? 1 : 0
  
  # Turn development env on/off with terraform apply
  name = "dev-instance"
  # ...
}
```

**Pattern 2: Alternative Resources Based on Cost**
```hcl
variable "is_production" {
  type = bool
}

resource "google_cloudsql_instance" "database" {
  database_version = "POSTGRES_15"
  tier             = var.is_production ? "db-custom-8-32768" : "db-f1-micro"
  
  backup_configuration {
    enabled = var.is_production  # Backups in prod only
  }
}
```

---

## Key Takeaways

1. **Terraform is about controlling infrastructure lifecycle** through code and state management
2. **State file is the source of truth** - protect it, version it, audit it
3. **Drift detection is critical** - manual changes will bite you
4. **Team workflows require remote state + locking**
5. **Modular design enables scaling** to enterprise infrastructure
6. **IaC enables disaster recovery** - full rebuild possible in minutes
