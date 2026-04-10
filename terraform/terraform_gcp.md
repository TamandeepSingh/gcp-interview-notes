# Terraform + GCP

## 1. Core Concept

**Terraform on GCP** means using Terraform to manage Google Cloud Platform infrastructure through the `google` provider. This enables complete infrastructure-as-code for GCP resources: Compute, Networking, Storage, Databases, IAM, and more.

**What Terraform manages:**
- Compute Engine (VMs, Images, Disks, Snapshots)
- GKE (Kubernetes clusters, node pools)
- Cloud SQL (MySQL, PostgreSQL, SQL Server)
- Cloud Storage (Buckets, objects, IAM)
- Networking (VPCs, subnets, routes, firewalls)
- IAM (Service Accounts, roles, permissions)
- Cloud Run
- Pub/Sub
- Cloud Functions
- BigQuery
- And 300+ more resources

---

## 2. Internal Working (VERY IMPORTANT)

### GCP Provider Plugin Architecture

```
terraform init
  ↓
Downloads google provider binary from HashiCorp Registry
  ├─ e.g., terraform-provider-google-5.0.0_linux_amd64
  └─ Stores in .terraform/providers/registry.terraform.io/hashicorp/google/5.0.0/
  
Provider Plugin (separate process)
  ├─ Starts gRPC server on random port
  ├─ Terraform core communicates via gRPC
  ├─ Calls Google Cloud APIs when resources created/updated
  └─ Returns state to terraform core
```

### Resource Creation Workflow

```hcl
resource "google_compute_instance" "web" {
  name         = "web-1"
  machine_type = "e2-medium"
  zone         = "us-central1-a"
  
  boot_disk {
    initialize_params {
      image = "debian-11"
    }
  }
}
```

**When `terraform apply` runs:**

```
1. Terraform core calls provider: CreateResource()
2. Provider receives resource config:
{
  "type": "google_compute_instance",
  "values": {
    "name": "web-1",
    "machine_type": "e2-medium",
    "zone": "us-central1-a"
  }
}

3. Provider translates to GCP API call:
POST https://compute.googleapis.com/compute/v1/projects/PROJECT/zones/us-central1-a/instances
Body: {
  "name": "web-1",
  "machineType": "zones/us-central1-a/machineTypes/e2-medium",
  "disks": [{...}]
}

4. GCP API creates instance, returns:
{
  "id": "1234567890",
  "name": "web-1",
  "status": "RUNNING",
  "selfLink": "https://...",
  "publicIp": "35.192.1.1"
}

5. Provider stores response as resource state
6. Terraform saves to tfstate:
{
  "type": "google_compute_instance",
  "instances": [{
    "attributes": {
      "id": "web-1",              // ID for tracking
      "name": "web-1",
      "zone": "us-central1-a",
      "self_link": "https://...",
      "network_interface": [{
        "network_ip": "10.0.0.2",
        "access_config": [{
          "nat_ip": "35.192.1.1"
        }]
      }]
    }
  }]
}
```

### Provider Authentication

```hcl
provider "google" {
  project = "my-project"
  region  = "us-central1"
  
  # Optional: explicitly specify credentials
  # credentials = file("service-account-key.json")
}
```

**Authentication resolution (in order):**

1. Explicit `credentials` in provider block
2. Environment variable: `GOOGLE_APPLICATION_CREDENTIALS`
3. Application Default Credentials (SDK)
   ```bash
   gcloud auth application-default login
   # Creates ~/.config/gcloud/application_default_credentials.json
   ```
4. Cloud metadata server (if running on GCE/GKE)

**Production pattern (CI/CD):**

```bash
# Authenticate with service account
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/sa-key.json

# Or using gcloud
gcloud auth activate-service-account --key-file=sa-key.json
gcloud config set project my-project

# Or in GitHub Actions
- uses: google-github-actions/auth@v1
  with:
    credentials_json: ${{ secrets.GCP_SA_KEY }}

# Terraform automatically picks up credentials
terraform init
terraform plan
```

### Resource ID Tracking

```
GCP resources have different ID patterns:

google_compute_instance:
  id = "instance-name" (or numeric ID)
  Terraform state tracks: "web-1"
  
google_compute_network:
  id = "network-id" 
  Terraform state tracks: "projects/my-project/global/networks/default"
  
google_container_cluster:
  id = "cluster-name"
  Retrieved via: 
    projects/{project}/locations/{zone}/clusters/{name}

google_sql_instance:
  id = "instance-name"
  Retrieved via:
    projects/{project}/instances/{name}

When Terraform needs to read resource:
  → Uses stored ID
  → Calls GCP API with ID
  → Gets current attributes
  → Detects drift
```

---

## 3. Why Terraform + GCP is Needed

### Without Terraform (Manual Approaches)

**Console clicking:**
```
Problem: 
- Infrastructure not reproducible
- New employee doesn't know what's deployed
- Disaster recovery takes hours
- No version control
- No change review process
```

**gcloud CLI scripts:**
```bash
gcloud compute instances create web-1 --machine-type=e2-medium --zone=us-central1-a
gcloud compute instances create web-2 --machine-type=e2-medium --zone=us-central1-b
# etc

Problem:
- Imperative (how to build), not declarative (desired state)
- No idempotency (script fails if resource exists)
- Hard to maintain consistency
- Difficult to track desired state
- Complex dependencies between resources
```

### Problem Solved by Terraform

✅ **Idempotent Infrastructure**
```
terraform apply → creates or updates as needed
terraform apply (2nd time) → recognizes everything correct, no changes
```

✅ **Complete Documentation**
```
.tf files describe entire infrastructure
New teammate: `terraform state list` shows exactly what exists
New environment: `terraform apply` recreates everything
```

✅ **Change Review**
```
git PR → terraform plan shows changes
code review → team approves
auto-merge → terraform apply executes
audit trail: Git history
```

✅ **Multi-region/Multi-project**
```
Same code, different variables → deploy to dev, staging, prod
Other project → copy variables.tfvars, terraform apply
Provider: GCP or AWS or Azure, same workflow
```

---

## 4. When to Use vs Not Use Terraform + GCP

### Use Terraform When:

- ✅ Any production infrastructure
- ✅ Infrastructure lasting > 1 week
- ✅ Multiple environments (dev/staging/prod)
- ✅ Team needs to manage infrastructure together
- ✅ Need disaster recovery capability
- ✅ Compliance requires audit trails

### DON'T Use Terraform When:

- ❌ Temporary experiments (< 4 hours)
- ❌ One-time infrastructure with no future changes
- ❌ No version control available
- ❌ Team unfamiliar with infrastructure-as-code

### Hybrid Approach (Common Pattern)

```hcl
# Use Terraform for static infrastructure
terraform {
  # VPCs, databases, GKE cluster
}

# Use Helm/kubectl for dynamic app deployment
# (Apps change frequently, cluster stable)
```

---

## 5. Real-world DevOps Usage

### Complete GCP Infrastructure Setup

**Directory Structure:**
```
terraform/
├── versions.tf
├── variables.tf
├── main.tf
├── outputs.tf
├── terraform.tfvars
└── modules/
    ├── vpc/
    ├── gke/
    └── cloud_sql/
```

**versions.tf:**
```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
  
  backend "gcs" {
    bucket = "tf-state-prod"
    prefix = "infrastructure"
  }
}

provider "google" {
  project = var.gcp_project
  region  = var.region
}
```

**variables.tf:**
```hcl
variable "gcp_project" {
  type = string
  description = "GCP Project ID"
}

variable "region" {
  type = string
  default = "us-central1"
}

variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod"
  }
}
```

**main.tf:**
```hcl
# VPC Network
module "vpc" {
  source = "./modules/vpc"
  
  name    = "prod-vpc"
  project = var.gcp_project
  region  = var.region
}

# GKE Cluster
module "gke" {
  source = "./modules/gke"
  
  cluster_name = "prod-gke"
  network_id   = module.vpc.network_id
  subnetwork   = module.vpc.subnetwork_id
  
  depends_on = [module.vpc]
}

# Cloud SQL Database
module "cloudsql" {
  source = "./modules/cloud_sql"
  
  instance_name = "prod-database"
  database_name = "production"
  region        = var.region
  
  authorized_networks = [
    {
      name  = "gke-nodes"
      value = module.gke.cluster_primary_cidr
    }
  ]
}

# IAM - Service Accounts and Roles
resource "google_service_account" "k8s_workload" {
  account_id   = "k8s-workload"
  display_name = "Kubernetes workload service account"
}

resource "google_project_iam_member" "k8s_permissions" {
  project = var.gcp_project
  role    = "roles/container.developer"
  member  = "serviceAccount:${google_service_account.k8s_workload.email}"
}
```

**outputs.tf:**
```hcl
output "kubernetes_endpoint" {
  value       = module.gke.endpoint
  description = "Kubernetes API endpoint"
  sensitive   = true
}

output "cloudsql_connection_name" {
  value = module.cloudsql.connection_name
  description = "Cloud SQL connection string for applications"
}

output "vpc_network_id" {
  value = module.vpc.network_id
}
```

**Deployment Workflow:**

```bash
# 1. Initialize
terraform init

# 2. Validate
terraform fmt
terraform validate

# 3. Plan
terraform plan -out=tfplan

# 4. Review plan, commit to Git
git add tfplan
git commit -m "Add prod infrastructure"

# 5. Apply (manual or via CI/CD)
terraform apply tfplan

# 6. Verify
terraform state list
gcloud compute instances list
gcloud container clusters list
gcloud sql instances list
```

---

## 6. Architecture Thinking

### GCP Resources in Terraform

**Basic Resource Patterns:**

```hcl
# 1. Simple resource (Create once, rarely change)
resource "google_compute_network" "main" {
  name = "prod-vpc"
}

# Changes: Easy, just terraform apply

# 2. Resource with dependencies
resource "google_compute_subnetwork" "prod" {
  name          = "prod-subnet"
  ip_cidr_range = "10.0.0.0/24"
  network       = google_compute_network.main.id  # Depends on VPC
}

# Terraform automatically waits for VPC creation first

# 3. Many resources (Dynamic creation)
resource "google_compute_instance" "web" {
  for_each = var.web_servers
  
  name         = each.key
  machine_type = each.value.machine_type
  zone         = each.value.zone
}

# Create/destroy servers by modifying var.web_servers

# 4. Resource with local modifications that Terraform should ignore
resource "google_compute_instance" "lifecycle_example" {
  name         = "web-1"
  machine_type = "e2-medium"
  
  lifecycle {
    # Ignore if someone manually adds labels
    ignore_changes = [labels]
  }
}
```

### Multi-environment GCP Architecture

**Scenario: Deploy same infrastructure to dev, staging, prod**

```
gcp-infrastructure/
├── modules/
│   ├── compute/
│   ├── networking/
│   └── databases/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
│   │   └── backend-config.gcs
│   ├── staging/
│   └── prod/
│       ├── main.tf
│       ├── terraform.tfvars
│       └── backend-config.gcs
└── global/
    └── shared_resources.tf
```

**environments/prod/terraform.tfvars:**
```hcl
gcp_project    = "company-prod"
region         = "us-central1"
environment    = "production"

# Infrastructure sizing
gke_node_count = 10
node_machine   = "e2-standard-8"
cloudsql_tier  = "db-custom-8-32gb"

# High availability for prod
enable_backups    = true
enable_monitoring = true
multi_zone        = true
```

**environments/dev/terraform.tfvars:**
```hcl
gcp_project    = "company-dev"
region         = "us-central1"
environment    = "development"

# Cost-optimized
gke_node_count = 1
node_machine   = "e2-medium"
cloudsql_tier  = "db-f1-micro"

# Disable expensive features
enable_backups    = false
enable_monitoring = false
multi_zone        = false
```

**Deployment process:**
```bash
# Setup prod
cd environments/prod
terraform init -backend-config=backend-config.gcs
terraform plan
terraform apply

# Setup dev (independent, same pattern)
cd ../dev
terraform init -backend-config=backend-config.gcs
terraform plan
terraform apply

# Result: prod and dev use completely separate GCP projects
# Risk isolation, independent scaling
```

---

## 7. Common Mistakes

### Mistake 1: Hardcoded Project ID

❌ **Wrong:**
```hcl
resource "google_compute_instance" "web" {
  project = "my-company-prod-123"  # Hardcoded!
  name    = "web-1"
}

# Problem:
# - Can't reuse code for other projects
# - Have to edit for each environment
# - Copy-paste errors likely
```

✅ **Right:**
```hcl
variable "gcp_project" { type = string }

provider "google" {
  project = var.gcp_project
}

resource "google_compute_instance" "web" {
  name = "web-1"
  # Project inherited from provider
}

# environments/prod/terraform.tfvars
gcp_project = "company-prod-123"

# environments/dev/terraform.tfvars
gcp_project = "company-dev-456"
```

### Mistake 2: Not Using Service Accounts

❌ **Wrong:**
```bash
# Using personal credentials for Terraform in CI/CD
gcloud auth login

# Problems:
# - Credentials tied to person
# - Person leaves company → credentials expire
# - No audit trail
# - CI/CD box has personal creds (security risk)
```

✅ **Right:**
```bash
# Create dedicated service account
gcloud iam service-accounts create terraform \
  --display-name="Terraform automation"

# Grant minimal required permissions
gcloud projects add-iam-policy-binding my-project \
  --member=serviceAccount:terraform@my-project.iam.gserviceaccount.com \
  --role=roles/compute.admin \
  --role=roles/container.admin \
  --role=roles/cloudsql.admin

# Use service account in CI/CD
export GOOGLE_APPLICATION_CREDENTIALS=./terraform-sa-key.json
terraform init
```

### Mistake 3: Improper IAM Configuration

❌ **Wrong:**
```hcl
resource "google_project_iam_member" "admin" {
  project = var.gcp_project
  role    = "roles/editor"  # Too permissive
  member  = "user:dev@company.com"
}

# Problem: Developer has full edit access to everything
```

✅ **Right:**
```hcl
resource "google_project_iam_member" "compute" {
  project = var.gcp_project
  role    = "roles/compute.admin"      # Minimal required
  member  = "serviceAccount:${var.app_sa}"
}

resource "google_project_iam_member" "logging" {
  project = var.gcp_project
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${var.app_sa}"
}

# Least privilege: only Compute and Logging access
```

### Mistake 4: Creating Firewall Rules Wrong

❌ **Wrong:**
```hcl
resource "google_compute_firewall" "allow_all" {
  name    = "allow-all"
  network = "default"
  
  allow {
    protocol = "tcp"
    ports    = ["0-65535"]  # All ports!
  }
  
  source_ranges = ["0.0.0.0/0"]  # All IPs!
}

# Disaster: Entire infrastructure exposed to internet
```

✅ **Right:**
```hcl
resource "google_compute_firewall" "allow_https" {
  name    = "allow-https"
  network = google_compute_network.main.name
  
  allow {
    protocol = "tcp"
    ports    = ["443"]  # Only HTTPS
  }
  
  source_ranges = ["0.0.0.0/0"]  # OK for HTTPS
}

resource "google_compute_firewall" "allow_internal" {
  name    = "allow-internal-communication"
  network = google_compute_network.main.name
  
  allow {
    protocol = "tcp"
    ports    = ["5432"]  # PostgreSQL
  }
  
  source_ranges = ["10.0.0.0/8"]  # Only internal IPs
}
```

### Mistake 5: Not Using Variable Validation

❌ **Wrong:**
```hcl
variable "gce_instance_count" { type = number }

resource "google_compute_instance" "web" {
  count        = var.gce_instance_count
  name         = "web-${count.index}"
  machine_type = "e2-medium"
}

# If someone passes count=1000, creates 1000 instances!
terraform apply -var="gce_instance_count=1000"
# Surprise $10k bill
```

✅ **Right:**
```hcl
variable "gce_instance_count" {
  type = number
  
  validation {
    condition     = var.gce_instance_count >= 1 && var.gce_instance_count <= 10
    error_message = "Instance count must be between 1 and 10"
  }
}

# terraform sum apply -var="gce_instance_count=1000"
# Error: Instance count must be between 1 and 10
```

### Mistake 6: Forgetting About Costs

❌ **Problem:**
```hcl
resource "google_compute_instance" "debug" {
  name         = "debug-instance"
  machine_type = "n2-highmem-96"  # 96 CPUs, very expensive
  zone         = "us-central1-a"
  
  # Left running in dev environment
  # Cost: $1000/month!
}

# terraform destroy forgot about the debug instance
# Instance keeps running, keeps accumulating costs
```

✅ **Right:**
```hcl
resource "google_compute_instance" "debug" {
  count        = var.create_debug_instance ? 1 : 0
  name         = "debug-instance"
  machine_type = "e2-medium"  # Smaller for dev
  zone         = "us-central1-a"
  
  lifecycle {
    prevent_destroy = false  # Easy to destroy when needed
  }
}

# In dev/terraform.tfvars
create_debug_instance = true

# When no longer needed:
create_debug_instance = false
terraform apply  # Destroys debug instance
```

---

## 8. Interview Questions (Scenario-based)

### Q1: Multi-project Setup

**Interviewer:** "You have dev, staging, prod projects in GCP. Design Terraform structure."

**Good Answer:**
```
terraform/
├── modules/
│   └── (shared code)
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   └── terraform.tfvars (project = dev-project)
│   ├── staging/
│   │   └── terraform.tfvars (project = staging-project)
│   └── prod/
│       └── terraform.tfvars (project = prod-project)

Usage:
  cd environments/prod && terraform apply
  
Result:
  - Same code, different outputs per project
  - States isolated per project
  - Each project independent
```

### Q2: GKE and Cloud SQL

**Interviewer:** "Design Terraform for GKE cluster + Cloud SQL. K8s needs to connect to database."

**Good Answer:**
1. Create Cloud SQL instance in private network
2. Create GKE cluster with IP range that can access database
3. Configure Cloud SQL user and password
4. Create Kubernetes Secret with connection credentials
5. Pods connect using secret credentials

### Q3: Cost Optimization

**Interviewer:** "Test environment doesn't need to run 24/7. How use Terraform to save costs?"

**Good Answer:**
```hcl
variable "environment" { type = string }
variable "auto_shutdown_time" { type = string; default = "22:00" }

resource "google_compute_instance" "web" {
  machine_type = var.environment == "production" ? "e2-standard-8" : "e2-medium"
  
  scheduling {
    preemptible        = var.environment != "production"  # Save 70% in dev
    automatic_restart  = false
  }
}

resource "google_cloud_scheduler_job" "shutdown_dev" {
  count = var.environment == "development" ? 1 : 0
  
  schedule  = "0 22 * * *"  # Shutdown at 10 PM daily
  // Terraform destroys resources at scheduled time
}
```

---

## 9. Debugging Issues

### Issue 1: "provider.google: Invalid location"

```
Error: Error creating Compute Instance:
location "us-central1" is not a valid GCP location
```

**Diagnosis:**
```bash
# Check valid zones
gcloud compute zones list

# Check if zone vs region confusion
# Zones: us-central1-a, us-central1-b (specific)
# Regions: us-central1 (group of zones)
```

**Solution:**
```hcl
# Instance requires zone (not region)
resource "google_compute_instance" "web" {
  zone = "us-central1-a"  # ZONE, not region
}

# Cloud SQL requires region
resource "google_sql_instance" "db" {
  region = "us-central1"  # REGION, not zone
}
```

### Issue 2: "Insufficient project permissions"

```
Error: Error reading project IAM policy:
googleapi: Error 403: Insufficient Permission
```

**Diagnosis:**
```bash
# Check current permissions
gcloud projects get-iam-policy $(gcloud config get-value project)
```

**Solution:**
```bash
# Grant necessary roles
gcloud projects add-iam-policy-binding my-project \
  --member=serviceAccount:terraform@my-project.iam.gserviceaccount.com \
  --role=roles/compute.admin \
  --role=roles/container.admin
```

### Issue 3: "Network already exists"

```
Error: Error creating Network:
resource "google_compute_network" "main" already exists
```

**Cause:** Network created manually or by previous Terraform

**Solution:**
```bash
# Import existing network
terraform import google_compute_network.main projects/my-project/global/networks/default

# Now Terraform knows about it
terraform plan  # Should show no changes
```

---

## 10. Advanced Insights

### Terraform Cloud Integration with GCP

**Pattern: Terraform Cloud for multi-team coordination**

```hcl
# terraform.tf
terraform {
  cloud {
    organization = "my-company"
    
    workspaces {
      name = "prod-gcp"
    }
  }
}

# Benefits:
# - Remote state management (Terraform Cloud instead of GCS)
# - Team collaboration features
# - Cost estimation before apply
# - Policy as code (prevent dangerous changes)
# - State locking automatic
```

### GCP + Terraform for Disaster Recovery

**Pattern: Generate state from GCP resources**

```bash
# If local state corrupted, recover from GCP:

# 1. Remove all resources from state
terraform state list | xargs -I {} terraform state rm {}

# 2. Import all resources from GCP
for resource_id in $(gcloud compute instances list --format='value(NAME)'); do
  terraform import google_compute_instance.web_* projects/PROJECT/zones/ZONE/instances/$resource_id
done

# 3. Verify
terraform plan  # Should show no changes

# 4. Infrastructure recovered, state rebuilt
```

---

## Key Takeaways

1. **Terraform is the standard** for managing GCP infrastructure at scale
2. **Use service accounts** in production, never personal credentials
3. **Separate environments** by project using same Terraform code + variables
4. **Design for disaster recovery** - state can always be rebuilt from GCP APIs
5. **Cost optimization** - leverage variables for environment-specific sizing
6. **Automate everything** - infrastructure changes via  Git PR not console clicks
