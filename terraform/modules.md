# Terraform Modules

## 1. Core Concept

A **Terraform module** is a reusable collection of resources (`.tf` files) packaged together for reuse. It's analogous to a function in programming:
- **Input:** Variables (function parameters)
- **Processing:** Resource creation logic
- **Output:** Outputs (return values)

Modules enable:
- **Reusability:** Write once, use 50 times
- **Abstraction:** Hide complexity behind simple interface
- **Consistency:** Same pattern deployed identically each time
- **Team collaboration:** Different teams extend same modules

---

## 2. Internal Working (VERY IMPORTANT)

### Module Invocation Flow

```hcl
# Root module (main infrastructure)
module "networking" {
  source = "./modules/vpc"
  
  vpc_name          = "prod-network"
  subnet_count      = 3
  enable_nat        = true
}

# Terraform executes:
# 1. Load ./modules/vpc/* files
# 2. Instantiate variables from module block (vpc_name, subnet_count, etc)
# 3. Execute resource creation logic
# 4. Return outputs back to root module
# 5. Store in state as: module.networking.google_compute_network.main
```

### Module State Storage

```json
{
  "resources": [
    {
      "type": "module",
      "name": "networking",
      "instances": [
        {
          "attributes": {
            "vpc_id": "prod-network",
            "subnet_ids": ["subnet-1", "subnet-2"]
          }
        }
      ],
      // Note: Child resources stored as:
      "module.networking.google_compute_network.main": {...}
    }
  ]
}
```

### Module Resolution Process

```
1. Parser sees: module "name" { source = "..." }
2. Resolver fetches source:
   - Local path: ./modules/vpc (reads disk)
   - Git: git::https://github.com/...//modules/vpc
   - Registry: hashicorp/aws//modules/vpc
   - S3/GCS: s3::https://bucket.s3.amazonaws.com/modules/...
3. Downloads/clones source to .terraform/modules/
4. Loads module's variables.tf, main.tf, outputs.tf
5. Variables passed by caller are assigned to module's var.*
6. Resources inherit module namespace in state
```

### Module Dependency Resolution

```hcl
# Root module
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}

module "gke" {
  source        = "./modules/gke"
  network_id    = module.vpc.network_id    # Reference module output
  subnet_ids    = module.vpc.subnet_ids
}

# Terraform dependency graph:
vpc_module → gke_module
# Terraform ensures VPC created before GKE
# Even though both modules are defined sequentially
```

### How Reusability Works

```hcl
# Module definition: modules/microservice/main.tf
variable "name" { type = string }
variable "image" { type = string }
variable "replicas" { type = number; default = 3 }

resource "kubernetes_deployment" "app" {
  metadata {
    name = var.name
  }
  spec {
    replicas = var.replicas
    template {
      spec {
        container {
          image = var.image
        }
      }
    }
  }
}

output "deployment_name" {
  value = kubernetes_deployment.app.metadata[0].name
}

# Usage 1: Deploy microservice A
module "svc_a" {
  source   = "./modules/microservice"
  name     = "service-a"
  image    = "gcr.io/service-a:latest"
  replicas = 5
}

# Usage 2: Deploy microservice B (same module, different inputs)
module "svc_b" {
  source   = "./modules/microservice"
  name     = "service-b"
  image    = "gcr.io/service-b:latest"
  replicas = 3
}

# Result: Identical deployment pattern, different config
# Maintenance: Update once in module, both services benefit
```

---

## 3. Why Modules are Essential

### Problem Without Modules

**Scenario: Deploy 50 Kubernetes deployments with same pattern**

```hcl
# Without modules - extreme duplication
resource "kubernetes_deployment" "svc_1" {...}
resource "kubernetes_deployment" "svc_2" {...}
resource "kubernetes_deployment" "svc_3" {...}
# ... repeat 47 more times ...
resource "kubernetes_deployment" "svc_50" {...}

# Problem: 2000+ lines of near-identical code
# Maintenance nightmare: Change pattern = edit 50 resources
# Bug fix: If one is configured wrong, all might be
# Testing: No way to validate pattern
```

### Problem Solved by Modules

```hcl
# With modules - DRY principle
module "microservices" {
  for_each = {
    svc_1 = { image = "gcr.io/svc-1:latest" }
    svc_2 = { image = "gcr.io/svc-2:latest" }
    # ... 48 more ...
    svc_50 = { image = "gcr.io/svc-50:latest" }
  }
  
  source   = "./modules/microservice"
  name     = each.key
  image    = each.value.image
}

# Result: 20 lines of code, same outcome
# Maintenance: Update pattern once in module
# Testing: Validate pattern in one place
```

### DRY Principle in Action

**Change Requirement:** "All deployments need resource limits"

Without modules:
```
Edit 50 resource definitions manually
Run Terraform
Risk of inconsistency (forgot one?)
```

With modules:
```
Edit modules/microservice/main.tf once
Run Terraform
All 50 deployments updated consistently
```

---

## 4. When to Use vs Not Use Modules

### Use Modules When:

- ✅ Same resource pattern deployed multiple times
- ✅ Pattern used across environments (dev, staging, prod)
- ✅ Team wants to enforce standards/best practices
- ✅ Complex resource groups need abstraction
- ✅ Planning to share code across teams/projects
- ✅ Versioning infrastructure patterns needed

### Don't Use Modules When:

- ❌ One-time infrastructure deployments
  - *Use: Direct resource definition in root*
- ❌ Simple infrastructure (1-2 resources)
  - *Use: Inline in root module*
- ❌ Module adds more complexity than value
  - *Use: Copy/paste, keep simple*

### Gray Area: When to Modularize Complex Resources

**Question:** Should I create a module for GKE cluster?

✅ **Yes, if:**
- You have multiple GKE clusters (prod, staging, dev)
- Team standardizes on specific GKE configuration
- Need consistent node pool sizing and policies

❌ **No, if:**
- Single cluster for entire company
- Cluster config is unique
- Adding module layer slows deployment

---

## 5. Real-world DevOps Usage

### Enterprise Pattern: Multi-Cluster GKE Management

**Requirement:** Deploy 3 GKE clusters (dev, staging, prod) with consistent best practices.

```
modules/
├── gke-cluster/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf
└── networking/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf

environments/
├── dev/
│   ├── main.tf
│   └── terraform.tfvars
├── staging/
│   ├── main.tf
│   └── terraform.tfvars
└── prod/
    ├── main.tf
    └── terraform.tfvars
```

**modules/gke-cluster/main.tf:**
```hcl
variable "cluster_name" { type = string }
variable "region" { type = string }
variable "node_count" { type = number; default = 3 }
variable "machine_type" { type = string; default = "e2-standard-4" }
variable "enable_autoscale" { type = bool; default = true }
variable "min_nodes" { type = number; default = 1 }
variable "max_nodes" { type = number; default = 10 }
variable "enable_logging" { type = bool; default = true }
variable "network_id" { type = string }
variable "subnetwork_id" { type = string }

# Best practices baked into module
resource "google_container_cluster" "primary" {
  name             = var.cluster_name
  location         = var.region
  initial_node_count = 1
  
  # Security: disable basic auth
  master_auth {
    client_certificate_config {
      issue_client_certificate = false
    }
  }
  
  # Networking: use custom network (prevent IP conflicts)
  network    = var.network_id
  subnetwork = var.subnetwork_id
  
  # Auto-scaling configuration
  autoscaling_config {
    min_node_count = var.min_nodes
    max_node_count = var.max_nodes
  }
  
  # Logging
  logging_service = var.enable_logging ? "logging.googleapis.com" : "none"
  
  # Monitoring
  monitoring_service = var.enable_logging ? "monitoring.googleapis.com" : "none"
  
  addons_config {
    http_load_balancing {
      disabled = false
    }
    kubernetes_dashboard {
      disabled = true  # Best practice: disable unused
    }
  }
}

# Node pool with best practices
resource "google_container_node_pool" "primary_nodes" {
  name       = "${var.cluster_name}-node-pool"
  cluster    = google_container_cluster.primary.id
  node_count = var.node_count

  autoscaling {
    min_node_count = var.min_nodes
    max_node_count = var.max_nodes
  }

  node_config {
    preemptible  = false
    machine_type = var.machine_type
    
    # Security: use minimal scopes
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
    
    # Best practice: use labels for cluster management
    labels = {
      environment = var.cluster_name
      managed_by  = "terraform"
    }
    
    # Taints for specialized workloads
    taint {
      key    = "managed-pool"
      value  = "true"
      effect = "NO_SCHEDULE"
    }
  }
}

output "cluster_name" {
  value = google_container_cluster.primary.name
}

output "kubernetes_cluster_host" {
  value       = google_container_cluster.primary.endpoint
  sensitive   = true
}

output "kubernetes_cluster_ca_certificate" {
  value       = base64decode(google_container_cluster.primary.master_auth[0].cluster_ca_certificate)
  sensitive   = true
}
```

**environments/prod/main.tf:**
```hcl
module "prod_network" {
  source = "../../modules/networking"
  
  vpc_name = "prod-vpc"
  region   = "us-central1"
}

# Use module to create cluster - all best practices applied automatically
module "prod_gke" {
  source = "../../modules/gke-cluster"
  
  cluster_name      = "prod-gke-cluster"
  region            = "us-central1"
  node_count        = 5
  machine_type      = "e2-standard-8"
  enable_autoscale  = true
  min_nodes         = 3
  max_nodes         = 20
  enable_logging    = true
  network_id        = module.prod_network.network_id
  subnetwork_id     = module.prod_network.subnetwork_id
}

# Prod outputs for other teams
output "cluster_host" {
  value = module.prod_gke.kubernetes_cluster_host
}
```

**environments/dev/main.tf:**
```hcl
# Same module, different config (cost-optimized for dev)
module "dev_network" {
  source = "../../modules/networking"
  
  vpc_name = "dev-vpc"
  region   = "us-central1"
}

module "dev_gke" {
  source = "../../modules/gke-cluster"
  
  cluster_name      = "dev-gke-cluster"
  region            = "us-central1"
  node_count        = 1                    # Minimal for dev
  machine_type      = "e2-medium"          # Smaller for cost
  enable_autoscale  = true
  min_nodes         = 1
  max_nodes         = 3                    # Limited autoscale
  enable_logging    = false                # No logging in dev
  network_id        = module.dev_network.network_id
  subnetwork_id     = module.dev_network.subnetwork_id
}
```

**Real Benefits:**
```
- Prod cluster: All best practices enforced
- Dev cluster: Cost-optimized but secure
- New cluster needed: Just copy block and change variables
- Update best practices: Edit module once, all clusters updated
```

---

## 6. Architecture Thinking

### Module Design Principles

**Principle 1: Single Responsibility**
```hcl
# Good module: Does one thing well
module "gke_cluster" {
  # Creates GKE cluster, node pools, networking
  # Clear scope: Kubernetes infrastructure
}

# Bad module: Does too much
module "platform" {
  # Creates GKE, databases, monitoring, IAM, load balancers
  # Unclear scope, hard to reuse, tight coupling
}
```

**Principle 2: Clear Interface**
```hcl
# Module inputs should be obvious
variable "cluster_name" { type = string }
variable "region" { type = string }
variable "node_count" { type = number }

# Not: variable "config" { type = object(...) }
# Too broad, forces users to understand complex structure
```

**Principle 3: Opinionated Defaults**
```hcl
# Good: Sensible defaults with escape hatch
variable "enable_logging" {
  type = bool
  default = true  # Secure by default
}

variable "logging_exclusion_filter" {
  type = string
  default = ""  # Users can override if needed
}

# Bad: All optional, requires remembering every config
variable "logging_enabled" { type = bool }  # Required!
```

**Principle 4: Explicit vs Implicit**
```hcl
# Good: Explicit dependencies
module "gke" {
  network_id = module.networking.network_id  # Clear dependency
}

# Bad: Implicit via data source
module "gke" {
  network_id = data.google_compute_network.default.id  # Hidden dependency
}
```

### Module Composition Patterns

**Pattern 1: Simple Module Wrapper**
```hcl
# module/gke/main.tf wraps provider resource
module "gke" {
  source = "./modules/gke"
  name   = "prod"
}

# Benefit: Can add policy, validation, defaults
# Drawback: Not much abstraction value if just wrapping
```

**Pattern 2: Composed Modules (Recommended)**
```hcl
# module/gke-complete/main.tf uses other modules
module "networking" {
  source = "../networking"
  name   = var.cluster_name
}

module "cluster" {
  source   = "../gke"
  network  = module.networking.network_id
  name     = var.cluster_name
}

module "monitoring" {
  source   = "../monitoring"
  cluster  = module.cluster.cluster_name
}

# User calls one module, gets complete solution
module "prod_platform" {
  source = "./modules/gke-complete"
  name   = "production"
}
```

**Pattern 3: Terraform Registry Modules (Reuse Community)**
```hcl
# Use official modules from registry
module "gke_auth" {
  source  = "terraform-google-modules/kubernetes-engine/google//modules/auth"
  version = "~> 26.0"
  
  depends_on = [google_container_cluster.primary]
  
  project_id   = var.project_id
  cluster_name = google_container_cluster.primary.name
}

# Benefit: Community-maintained, battle-tested patterns
# Drawback: Less control, must follow module's design
```

---

## 7. Common Mistakes

### Mistake 1: Module with Too Many Variables

❌ **Wrong:**
```hcl
variable "cluster_config" {
  type = object({
    name              = string
    region            = string
    node_count        = number
    machine_type      = string
    enable_logging    = bool
    logging_level     = string
    enable_monitoring = bool
    monitor_interval  = number
    # ... 50 more variables in object
  })
}

# Problem: Module interface too complex
# Users must understand entire cluster config
# Maintenance: Hard to add features without breaking users
```

✅ **Right:**
```hcl
variable "cluster_name" { type = string }
variable "region" { type = string }
variable "node_count" { type = number; default = 3 }
variable "machine_type" { type = string; default = "e2-standard-4" }

# Optional nested config for advanced users
variable "advanced_config" {
  type = object({
    logging_level      = optional(string, "INFO")
    monitor_interval   = optional(number, 60)
  })
  default = {}
}

# Or just provide escape hatch:
variable "additional_annotations" {
  type    = map(string)
  default = {}
}

resource "kubernetes_deployment" "app" {
  metadata {
    name = var.cluster_name
    annotations = merge(
      { managed_by = "terraform" },
      var.additional_annotations  # Users can override
    )
  }
}
```

### Mistake 2: Module Too Generic

❌ **Problem:**
```hcl
# Generic "k8s app" module
module "app" {
  source = "./modules/generic-app"
  # Tries to handle web, worker, batch, stateful all the same
  # Each app has different requirements
  # Module must have 100 variables
}

# vs

# Specific domain module
module "web_app" {
  source = "./modules/web-app"
  # Optimized for web patterns
  # Sensible security defaults
  # Load balancing built-in
  # Fewer variables to learn
}
```

✅ **Right:** Create specialized modules for domain patterns:
- `modules/web-app` (replicas, load balancing, health checks)
- `modules/batch-job` (parallelism, resource cleanup)
- `modules/stateful-app` (storage management, upgrades)

### Mistake 3: Circular Dependencies Between Modules

❌ **Wrong:**
```hcl
# Module A
module "networking" {
  source = "./modules/networking"
  # References module.database output
  database_sg_id = module.database.security_group_id
}

# Module B
module "database" {
  source = "./modules/database"
  # References module.networking output
  network_id = module.networking.network_id
}

# Terraform error: Cycle detected
```

✅ **Right:** Break circular dependencies with data sources:
```hcl
# Module A (doesn't create circular dep)
module "networking" {
  source = "./modules/networking"
}

# Module B (uses networking output)
module "database" {
  source = "./modules/database"
  network_id = module.networking.network_id
}

# Or if truly circular, split into separate root modules
```

### Mistake 4: Module Variables Not Documented

❌ **Problem:**
```hcl
variable "cluster_name" { type = string }
variable "node_count" { type = number }
variable "extra_config" { type = map(any) }
# User has no idea what input is

# Later in code:
var.extra_config["max_concurrent_deployments"]
# Is this expected? Documented? Required?
```

✅ **Right:**
```hcl
variable "cluster_name" {
  type        = string
  description = "Name of the GKE cluster"
  
  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.cluster_name))
    error_message = "Cluster name must be lowercase alphanumeric with hyphens"
  }
}

variable "node_count" {
  type        = number
  description = "Initial number of nodes in cluster. Use autoscaling for prod"
  
  validation {
    condition     = var.node_count >= 1 && var.node_count <= 1000
    error_message = "Node count must be between 1 and 1000"
  }
}

variable "extra_config" {
  type        = map(any)
  description = <<-EOT
    Additional configuration passed to GKE:
      - max_concurrent_deployments: (int) parallel deployments
      - enable_experimental_features: (bool) enable beta features
  EOT
  default     = {}
}
```

### Mistake 5: Not Versioning Modules

❌ **Problem:**
```hcl
module "gke" {
  source = "./modules/gke"  # No version specified
  # If someone updates module, breaks your code
}

# vs

module "gke" {
  source = "git::https://github.com/company/tf-modules.git//gke?ref=main"
  # Reference branch, always gets latest
  # Might break if latest incompatible
}
```

✅ **Right:**
```hcl
module "gke" {
  source = "git::https://github.com/company/tf-modules.git//gke?ref=v2.1.0"
  # Pinned to specific version
  # Changes only by deliberate commit
  
  # For local modules, use version numbers in comments
  # modules/gke/v2.1.0/main.tf
}

# Document breaking changes
# When update needed, make it explicit
```

### Mistake 6: Outputs Don't Match Interface

❌ **Problem:**
```hcl
# Module users expect these outputs
output "cluster_name" { value = google_container_cluster.primary.name }
output "network_id" { value = google_compute_network.main.id }

# But they also import module and try:
${module.gke.kubernetes_access_config}  # Doesn't exist!
# Documentation missing, guess outputs, fail
```

✅ **Right:**
```hcl
# Export what users need, clearly documented
output "cluster_name" {
  description = "Name of the GKE cluster"
  value       = google_container_cluster.primary.name
}

output "cluster_endpoint" {
  description = "Kubernetes api endpoint"
  value       = google_container_cluster.primary.endpoint
  sensitive   = true
}

output "network_id" {
  description = "VPC network ID"
  value       = google_compute_network.main.id
}

# In README.md, document all outputs
# Users know exactly what they get from module
```

---

## 8. Interview Questions (Scenario-based)

### Q1: Module Design Decision

**Interviewer:** "You need to deploy 20 microservices in GKE. Each has different images, configs, resource limits. Design a module solution."

**Good Answer:**
1. Create module for single microservice:
   ```hcl
   module "app" {
     for_each = var.microservices
     source   = "./modules/microservice"
     
     name            = each.key
     image           = each.value.image
     replicas        = each.value.replicas
     resource_limits = each.value.limits
   }
   ```

2. Module has clear inputs/outputs
3. Easy to add 21st service (just add to map)
4. Consistency enforced at module level

### Q2: Circular Dependency

**Interviewer:** "You have networking depending on database for security groups, and database depending on networking for VPC. How do you structure modules?"

**Good Answer:**
- Root module creates networking first
- Database module references networking output
- No circular dependency, clear execution order
- Both modules available for other code

### Q3: Module Reuse Across Projects

**Interviewer:** "Your GKE module works great. Team B wants to use it. They want X feature you don't have. How do you handle?"

**Good Answer:**
1. **Don't hard-code Team B's requirement**
2. **Add optional feature to module:**
   ```hcl
   variable "enable_custom_feature" {
     type    = bool
     default = false
   }
   
   resource "..." {
     count = var.enable_custom_feature ? 1 : 0
   }
   ```
3. **Team B enables in their code**
4. **Module stays flexible**

### Q4: Module Output Dependency

**Interviewer:** "You have module.networking.outputs.network_id needed by 3 other modules. What if networking gets replaced? How do you prevent cascading failures?"

**Good Answer:**
- Network ID remains same if subnet config unchanged
- Use `lifecycle.ignore_changes` on network ID if needed
- Better: Isolate changes to internal network structure
- Reference network by name if possible (more stable)

### Q5: Testing Module Changes

**Interviewer:** "You need to update GKE module (add new feature). How do you ensure it doesn't break 5 environments currently using it?"

**Good Answer:**
1. **Create test environment:**
   ```bash
   terraform workspace new module-test
   ```

2. **Update module version:**
   ```hcl
   source = "git::...//modules/gke?ref=feature-branch"
   ```

3. **Plan to see impact:**
   ```bash
   terraform plan
   # Review if any breaking changes
   ```

4. **Update in stages:**
   - Dev environment (low risk)
   - Staging (pre-validated)
   - Prod (after approval)

---

## 9. Debugging Issues

### Issue 1: "Module Source Not Found"

```
Error: Invalid source address

module.gke: Failed to get module registry metadata
```

**Diagnosis:**
```bash
# Check path is correct
ls -la ./modules/gke/
# If doesn't exist, path wrong
```

**Solution:**
```hcl
# Local module
source = "./modules/gke"

# Git module
source = "git::https://github.com/org/repo.git//modules/gke?ref=main"

# Registry
source = "terraform-google-modules/kubernetes-engine/google"
```

### Issue 2: "Variable Not Set in Module Call"

```
Error: Insufficient args

on modules/gke/main.tf line 10, in variable "cluster_name":
Unsatisfied requirement "cluster_name"
```

**Diagnosis:**
```bash
# Check what variables module requires
grep "variable" modules/gke/variables.tf | grep -v default
# Shows required variables
```

**Solution:**
```hcl
module "gke" {
  source = "./modules/gke"
  
  cluster_name = "prod-cluster"  # MUST provide
}
```

### Issue 3: "Module Output Not Available"

```
Error: Unsupported attribute

${module.gke.some_output}: Available outputs are: cluster_name, network_id
```

**Diagnosis:**
```bash
# Check what outputs module provides
grep "output" modules/gke/outputs.tf
```

**Solution:**
```hcl
module "gke" {
  source = "./modules/gke"
  name   = "prod"
}

# Use available outputs
kubernetes_endpoint = module.gke.cluster_endpoint
network_id          = module.gke.network_id

# If output is missing, check:
# 1. Output defined in module?
# 2. Terraform applied module?
```

### Issue 4: "State Referencing Old Module Location"

```
Error: Module not found

module.gke: source "./modules/gke-old" not found
```

**Cause:** State file has reference to old location

**Solution:**
```bash
# Move modules
mv ./modules/gke-old ./modules/gke

# Update code
# source = "./modules/gke-old" → "./modules/gke"

# Refresh state
terraform state show module.gke
# Shouldn't error

# Or move in state
terraform state mv 'module.gke' 'module.gke_v2'
```

---

## 10. Advanced Insights

### Module Versioning Strategies

**Strategy 1: Semantic Versioning**
```
Modules stored as:
  modules/gke/v1.0.0/main.tf
  modules/gke/v1.1.0/main.tf
  modules/gke/v2.0.0/main.tf

Backward compatible improvements: v1.1.0 (minor bump)
Breaking changes: v2.0.0 (major bump)

Code references:
  source = "./modules/gke/v1.1.0"
  source = "./modules/gke/v2.0.0"
```

**Strategy 2: Git Tags with Module Registry**
```
terraform {
  required_providers {
    hashicorp/google = "~> 5.0"
  }
}

module "gke" {
  source  = "git::ssh://git@github.com/org/modules.git//gke?ref=v1.5.3"
  version = "~> 1.5"
}
```

**Strategy 3: Internal Module Registry**
```
Company hosts internal registry:
  https://registry.company.com/terraform/google/gke/1.5.3

Then use:
source = "company/google/gke"
version = "~> 1.5"

# Terraform Cloud/Enterprise feature
```

### Module Testing Framework

**Pattern: Terraform Testing**
```bash
# Run module in test environment
$ cd modules/gke/test

$ terraform init
$ terraform plan

# Verify resources created correctly
$ terraform apply

$ gcloud container clusters describe test-cluster
# Validate cluster actually exists and configured right

# Cleanup
$ terraform destroy
```

**Pattern: Terratest (Go testing framework)**
```go
// Similar to unit testing for Terraform modules
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
)

func TestGKEModule(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../",
    }

    terraform.InitAndApply(t, terraformOptions)
    defer terraform.Destroy(t, terraformOptions)

    clusterName := terraform.Output(t, terraformOptions, "cluster_name")
    // Verify cluster was created
}
```

---

## Key Takeaways

1. **Modules enable DRY infrastructure** - write once, use many times
2. **Single responsibility principle** - one module = one concern
3. **Clear interfaces** - obvious inputs/outputs matter
4. **Composition over inheritance** - combine simple modules
5. **Version your modules** - prevent surprise breaking changes
6. **Document everything** - variables, outputs, usage patterns
