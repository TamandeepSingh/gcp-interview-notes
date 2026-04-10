# Terraform Best Practices

## 1. Core Concept

**Terraform best practices** are patterns and strategies that production-grade infrastructure teams use to maintain scalable, secure, and maintainable infrastructure-as-code. They're not required syntax, but the difference between "works" and "production-ready."

---

## 2. Project Structure

### Best Practice: Modular Organization

```
terraform/
├── versions.tf                    # Required providers, tf version
├── variables.tf                   # Root input variables
├── locals.tf                      # Local values (debugging aids)
├── main.tf                        # Primary resources
├── outputs.tf                     # Root module outputs
├── terraform.tf                   # Backend configuration
├── terraform.tfvars               # Actual values (git ignored)
├── terraform.tfvars.example       # Example values (git tracked)
│
├── modules/                       # Reusable modules
│   ├── vpc/
│   │   ├── versions.tf
│   │   ├── variables.tf
│   │   ├── main.tf
│   │   └── outputs.tf
│   │
│   ├── gke-cluster/
│   │   ├── versions.tf
│   │   ├── variables.tf
│   │   ├── main.tf
│   │   └── outputs.tf
│   │
│   └── cloud-sql/
│       ├── versions.tf
│       ├── variables.tf
│       ├── main.tf
│       └── outputs.tf
│
├── environments/                  # Different configs per environment
│   ├── dev/
│   │   ├── terraform.tf
│   │   ├── terraform.tfvars
│   │   └── backend-config.gcs
│   │
│   ├── staging/
│   │   ├── terraform.tf
│   │   ├── terraform.tfvars
│   │   └── backend-config.gcs
│   │
│   └── prod/
│       ├── terraform.tf
│       ├── terraform.tfvars
│       └── backend-config.gcs
│
├── .gitignore                    # Prevent state, keys from Git
├── .terraform/                   # Local cache (git ignored)
├── README.md                     # Documentation
└── CONTRIBUTING.md              # How to contribute

.gitignore content:
**/terraform.tfvars              # Secrets
**/terraform.tfvars.*.json
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl
crash.log
```

### Why This Structure Works

```
✅ clarity: Everyone knows where to look
✅ Scalability: Easy to add new modules
✅ Team collaboration: Clear boundaries
✅ Environment parity: Same code, different variables
✅ Testing: Each module can be tested independently
```

---

## 3. Naming Conventions

### Best Practice: Consistent Naming

```hcl
# ✅ Good: Descriptive, consistent
resource "google_compute_instance" "web_server_primary" {
  name = "prod-web-primary"
}

resource "google_compute_instance" "web_server_secondary" {
  name = "prod-web-secondary"
}

# ❌ Bad: Unclear naming
resource "google_compute_instance" "machine1" {
  name = "machine"
}

resource "google_compute_instance" "machine2" {
  name = "vm"
}
```

**Naming Pattern:**

```
resource_type_purpose_identifier

Examples:
  google_compute_instance.web_server_primary
  google_cloudsql_instance.prod_database
  google_container_cluster.gke_prod_us
  google_compute_firewall.allow_internal_http
```

**Naming Variables:**

```hcl
# ✅ Clear, hierarchical names
variable "environment_name" { type = string }
variable "project_id" { type = string }
variable "region" { type = string }
variable "gke_cluster_name" { type = string }
variable "gke_node_count" { type = number }
variable "enable_autoscaling" { type = bool }

# vs

# ❌ Generic names
variable "env" { type = string }
variable "proj" { type = string }
variable "cfg" { type = object(...) }
```

---

## 4. State Management

### Best Practice: Remote State + Locking

✅ **Enforce remote state:**

```hcl
terraform {
  backend "gcs" {
    bucket              = "tf-state-prod"
    prefix              = "infrastructure"
    encryption_key      = "projects/company-prod/locations/us/keyRings/terraform/cryptoKeys/prod"
  }
}

# Non-negotiable for production
# Prevents: Lost state, concurrent corruptions, credential exposure
```

✅ **Encrypt sensitive data:**

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

output "db_connection_string" {
  value     = "postgres://user:${var.db_password}@postgresql:5432/mydb"
  sensitive = true  # Redacts from terraform apply output
}
```

✅ **State file backup/versioning:**

```bash
# Enable versioning on backend bucket
gsutil versioning set on gs://tf-state-prod

# Now automatic history
gsutil ls -L gs://tf-state-prod/infrastructure/default.tfstate
# Lists all versions with timestamps
```

---

## 5. Variables and Locals

### Best Practice: Use Variables Strategically

```hcl
# ✅ Good: Parameterized for reusability
variable "environment" {
  type = string
  description = "Environment name (dev, staging, prod)"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod"
  }
}

variable "region" {
  type    = string
  default = "us-central1"
  description = "GCP region for resources"
}

variable "instance_count" {
  type    = number
  default = 3
  
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10"
  }
}
```

### Best Practice: Locals for Derived Values

```hcl
# ✅ Use locals to reduce repetition
locals {
  common_tags = {
    environment = var.environment
    managed_by  = "terraform"
    timestamp   = timestamp()
  }
  
  # Derived values (don't want users changing)
  cluster_name = "${var.environment}-gke-${var.region}"
  
  # Environment-specific values
  node_config = {
    dev = {
      node_count   = 1
      machine_type = "e2-medium"
      autoscale_min = 1
      autoscale_max = 3
    }
    prod = {
      node_count   = 5
      machine_type = "e2-standard-8"
      autoscale_min = 3
      autoscale_max = 20
    }
  }
}

resource "google_container_cluster" "primary" {
  name = local.cluster_name
  
  initial_node_count = local.node_config[var.environment].node_count
  
  resource_labels = local.common_tags
}
```

---

## 6. Code Quality and Validation

### Best Practice: Pre-commit Checks

```bash
# .git/hooks/pre-commit
#!/bin/bash

# Format check
terraform fmt -check -recursive

# Syntax validation
terraform validate

# Linting
tflint

# These prevent bad code from entering Git
```

### Best Practice: Automated Linting (CI/CD)

```yaml
# .github/workflows/terraform.yml
name: Terraform Quality Checks

on: [pull_request]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      # Format
      - run: terraform fmt -check -recursive
      
      # Validate
      - run: terraform validate
      
      # Lint
      - uses: terraform-linters/setup-tflint@v3
      - run: tflint --recursive
      
      # Security scan
      - uses: aquasecurity/tfsec-action@v1
      
      # Plan and post to PR
      - uses: hashicorp/setup-terraform@v2
      - run: terraform plan
      - uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '${{ steps.plan.outputs.stdout }}'
            })
```

### Best Practice: Security Scanning

```bash
# Scan for security issues
tfsec .

# Output:
# ⚠️  google_compute_firewall.allow_all: 
#     Firewall rule allows 0.0.0.0/0 - should restrict
#     
# ✓ 45 checks passed
# ⚠️  3 warnings
```

---

## 7. Module Design

### Best Practice: Clear Module Boundaries

```hcl
# ✅ Good: Module has single responsibility
module "network_infrastructure" {
  source = "./modules/vpc"
  
  # Clear inputs
  network_name = var.vpc_name
  cidr_block   = var.vpc_cidr
  
  # Creates: VPC, subnets, routes, routing rules
}

# vs

# ❌ Bad: Module does too much
module "everything" {
  source = "./modules/platform"
  
  # Creates: VPC, servers, databases, monitoring, users
  # Too broad, hard to reuse, hard to understand
}
```

### Best Practice: Explicit Outputs

```hcl
# ✅ Good: Document outputs clearly
output "vpc_id" {
  value       = google_compute_network.main.id
  description = "VPC network ID for reference by other modules"
}

output "subnet_ids" {
  value       = [for subnet in google_compute_subnetwork.main : subnet.id]
  description = "List of subnet IDs created"
}

# vs

# ❌ Bad: No documentation
output "id" {
  value = google_compute_network.main.id
}

output "stuff" {
  value = { vpc = google_compute_network.main, subnets = [...] }
}
```

### Best Practice: Module Versioning

```hcl
# ✅ Pin module versions
module "gke" {
  source = "git::https://github.com/company/modules.git//gke?ref=v2.1.0"
  
  # Deliberately pin to version
  # Updates require explicit action
}

# vs

# ❌ Float to latest (dangerous)
module "gke" {
  source = "git::https://github.com/company/modules.git//gke"
  # Always uses main branch
  # Breaking changes can surprise you
}
```

---

## 8. Workspace Strategy

### Best Practice: Workspaces for Environment Parity

```bash
# ✅ Use workspaces for identical environments with different configs
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Apply same code with different variables
terraform workspace select dev
terraform apply -var-file="dev.tfvars"

terraform workspace select prod
terraform apply -var-file="prod.tfvars"

# Result: Same code, separate state files per workspace
```

### Workspace Best Practices

```bash
# Each workspace has separate state
.terraform.tfstate.d/
├── dev/
│   └── terraform.tfstate
├── staging/
│   └── terraform.tfstate
└── prod/
    └── terraform.tfstate

# Can safely manage different environments simultaneously
# Prevents accidental production changes

# ❌ Don't do this:
terraform workspace select prod
terraform destroy
# Won't happen if you're conscious of workspace

# Use in conditional
resource "google_compute_instance" "web" {
  count        = terraform.workspace == "prod" ? 3 : 1
  name         = "web-${count.index}"
  machine_type = "e2-medium"
}
```

---

## 9. Error Handling

### Best Practice: Validation and Assertions

```hcl
# ✅ Validate inputs early
variable "instance_count" {
  type = number
  
  validation {
    condition     = var.instance_count > 0 && var.instance_count < 50
    error_message = "Instance count must be between 1 and 49"
  }
}

# ✅ Assert expected conditions
resource "google_compute_instance" "web" {
  name = var.instance_name
  
  # Fail if required input missing
  preconditions = [{
    condition     = length(var.instance_name) > 0
    error_message = "Instance name is required"
  }]
  
  # Verify expected state after creation
  postconditions = [{
    condition     = self.status == "RUNNING"
    error_message = "Instance failed to start"
  }]
}
```

### Best Practice: Graceful Degradation

```hcl
# ✅ Optional features
variable "enable_monitoring" {
  type    = bool
  default = true
}

resource "google_monitoring_alert_policy" "cpu" {
  count = var.enable_monitoring ? 1 : 0
  
  display_name = "High CPU Alert"
  # ...
}

# If monitoring not enabled, don't fail, just skip
# vs

# ❌ Required dependency
resource "google_monitoring_alert_policy" "cpu" {
  display_name = "High CPU Alert"
  # This always runs, fails if inputs missing
}
```

---

## 10. Documentation

### Best Practice: Document Everything

```hcl
# ✅ Comprehensive documentation
variable "cluster_name" {
  type        = string
  description = <<-EOT
    Name of the GKE cluster. Must be unique within the project.
    
    Examples:
      - prod-gke-us-central1
      - dev-gke-us-east1
      
    Naming convention: {environment}-gke-{region}
  EOT
}

output "kubernetes_endpoint" {
  value       = google_container_cluster.primary.endpoint
  description = "Kubernetes API endpoint. Used to configure kubectl."
  sensitive   = true
}
```

### Best Practice: Module README

```markdown
# GKE Module

Provisions a production-grade GKE cluster with best practices.

## Usage

```hcl
module "gke" {
  source = "./modules/gke"
  
  cluster_name = "prod-cluster"
  region       = "us-central1"
  node_count   = 3
}
```

## Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| cluster_name | string | - | Name of cluster |
| region | string | - | GCP region |
| node_count | number | 3 | Initial nodes |
| enable_autoscaling | bool | true | Auto-scale nodes |

## Outputs

| Name | Description |
|------|-------------|
| cluster_endpoint | Kubernetes API endpoint |
| cluster_ca_certificate | CA certificate for authentication |

## Examples

See [examples/](./examples/) directory for complete examples.

## Features

- OSS Kubernetes (not GKE Enterprise)
- Workload Identity enabled
- Logging and monitoring integrated
- Network policies enforced
```

---

## 11. Common Mistakes Summary

### Mistake: Not Destroying Test Resources

```bash
# Create test infrastructure
terraform apply -var="environment=test"

# Forget to destroy
terraform destroy

# $500/month bill for unused test resources
```

✅ **Prevention:**
```hcl
resource "google_compute_instance" "test" {
  count = var.is_test ? 1 : 0
  # Only created when explicitly testing
  
  lifecycle {
    prevent_destroy = false  # Easy to clean up
  }
}
```

### Mistake: Large State Files

```
❌ 1000 resources in single state
  → 10 minute plans
  → Slow API calls
  → Risky changes

✅ Split into multiple smaller states
  → 2-3 minute plans
  → Fast, predictable
  → Lower transaction cost
```

### Mistake: No Drift Detection

```bash
❌ Assume Terraform has full control
   Manual changes outside Terraform
   State becomes stale

✅ Regular drift detection
   terraform plan -refresh-only
   Daily job: Report drift
   Alert if manual changes detected
```

---

## 12. Operations and Maintenance

### Best Practice: Regular Terraform Audits

```bash
# Monthly checklist

# 1. Check for unused resources
terraform state list | wc -l  # Growing without reason?

# 2. Test disaster recovery
# Simulate state file loss, rebuild from cloud
terraform state list | xargs -I {} terraform state rm {}

for resource_id in $(gcloud compute instances list --format='value(NAME)'); do
  terraform import ...
done

# 3. Update Terraform versions
terraform version
# Check for updates, test in non-prod first

# 4. Review state size
terraform state pull | jq '.resources | length'  # Should grow linearly

# 5. Validate all modules still work
terraform validate
terraform plan -refresh-only
```

### Best Practice: Runbook for Common Operations

```markdown
# Terraform Operations Runbook

## Adding New Environment

1. Create new directory: `environments/new-env/`
2. Copy from `environments/prod/`
3. Update terraform.tfvars:
   - GCP project ID
   - Environment name
   - Resource counts
4. Create new backend bucket
5. Run terraform init
6. Run terraform plan
7. Review, then apply

## Scaling Cluster

1. Update `gke_node_count` in tfvars
2. terraform plan (review node changes)
3. terraform apply (scales cluster)
4. Monitor for pod rescheduling

## Rolling Terraform Version

1. Test in dev environment
2. Update versions.tf
3. Run terraform init
4. Run terraform plan (migration checks)
5. Verify behavior unchanged
6. Roll to prod

## Emergency Rollback

If infrastructure corrupted:
1. Stop new terraform applies: `terraform force-unlock`
2. Restore state from backup: `gsutil cp gs://...#VERSION`
3. Inspect restored state: `terraform state list`
4. Apply to rebuild: `terraform apply`
```

---

## Key Best Practices Summary

| Practice | Why | How |
|----------|-----|-----|
| Remote state | Collaboration + safety | GCS/S3 backend |
| State locking | Prevent corruption | Built-in to cloud backends |
| Modular code | Reusability + maintenance | modules/ directory |
| Clear naming | Understandability | Descriptive names |
| Validation | Catch errors early | variable validation blocks |
| Versioning | Control changes | Pin module versions |
| Documentation | Onboarding + reference | README + comments |
| Testing | Reliability | CI/CD validation |
| Security scanning | Prevent vulnerabilities | tfsec + policy checks |
| Monitoring drift | Keep reality in sync | Regular terraform plan |

---

## Production Terraform Stack

**Complete production setup:**

```
✅ Version control (Git)
   - Code reviews (PRs)
   - History and audit trail

✅ CI/CD (GitHub Actions, CloudBuild)
   - terraform fmt
   - terraform validate
   - terraform plan → post to PR
   - terraform apply on merge

✅ Remote state (GCS with encryption)
   - State locking
   - Versioning

✅ Security (IAM + Service Accounts)
   - Least privilege
   - Separate SA per environment

✅ Monitoring (Cloud Logging)
   - All apply actions logged
   - State changes tracked

✅ Documentation (README + examples)
   - New developers onboarded quickly
   - Clear patterns

✅ Backup/Recovery (tested monthly)
   - State restores
   - Disaster recovery validated

Result: Enterprise-grade infrastructure management
```

---

## Final Principles

1. **Treat infrastructure like code:** Version control, code reviews, testing
2. **Automate everything:** No manual infrastructure changes
3. **Keep it simple:** Understand what you're deploying
4. **Plan for failure:** Regular disaster recovery drills
5. **Document decisions:** Future you will be grateful
6. **Security first:** Encrypt state, use service accounts, audit access
7. **Monitor drift:** Infrastructure should match code
