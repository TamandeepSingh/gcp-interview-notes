# Terraform State Management

## 1. Core Concept

**Terraform state is the bridge between desired infrastructure (code) and actual infrastructure (cloud provider).** It's a JSON file that tracks:
- Resource IDs (to know which cloud resources Terraform manages)
- Resource attributes (IP addresses, database endpoints)
- Resource metadata (creation date, last modified)
- Dependency relationships
- Secrets (credentials, API keys)

State enables Terraform's core magic: **detecting what changed and only updating that.**

Without state:
- Terraform couldn't know which resources it created
- Couldn't distinguish between resources it should manage vs manually created
- Couldn't track dependencies
- Would destroy and recreate everything on each apply

---

## 2. Internal Working (VERY IMPORTANT)

### State File Lifecycle

**Read Phase (terraform plan):**
```
1. Terraform reads local/.tf files
2. Loads current state from backend (disk/S3/GCS/etc)
3. Calls provider ReadResource API for each resource in state
   - "What does this resource look like NOW in AWS?"
4. Compares provider response with state file
   - "Did someone manually change it? (drift detection)"
5. Compares provider response with desired code
   - "What needs to change to match code?"
6. Generates plan showing the diff
```

**Detailed State Read Process:**
```hcl
# terraform.tfstate (simplified)
{
  "resources": [
    {
      "type": "aws_instance",
      "name": "web",
      "instances": [
        {
          "attributes": {
            "id": "i-0f1234567890abcde",     // AWS resource ID
            "private_ip": "10.0.1.100",      // Last known IP
            "instance_type": "t3.medium",    // Last known type
            "__meta": {
              "schema_version": 1,
              "e_tag": "12345"               // Checksum to detect corruption
            }
          }
        }
      ]
    }
  ]
}
```

**When you run `terraform plan`:**

```python
# Pseudocode of Terraform's comparison logic
for resource_in_state in state.resources:
    actual_resource = provider.read(resource_in_state.id)
    
    # Detect drift
    if actual_resource.attributes != resource_in_state.attributes:
        print(f"DRIFT: {resource_in_state.name} changed outside Terraform")
    
    # Compare with code
    desired = parse_terraform_code()
    if actual_resource.attributes != desired.attributes:
        print(f"CHANGE NEEDED: Update {resource_in_state.name}")
```

### State Locking Mechanism

**Problem State Locking Solves:**
```
Scenario without locking:
1. Developer A runs: terraform plan
2. Developer B runs: terraform apply  (creates resources)
3. Developer A runs: terraform apply  (based on old state, might delete B's resources)
```

**Solution: Distributed State Lock**

State backends like GCS, S3, Terraform Cloud support advisory locks:

```
1. Developer A runs: terraform plan
   → Acquires lock: s3://bucket/.terraform.lock.db
   → Lock contains: timestamp, developer ID, terraform version
2. Developer B runs: terraform plan
   → Tries to acquire lock
   → Lock already held
   → Waits (default 0.3s between retries)
   → After configured timeout, fails with "resource locked" error
3. Developer A: terraform apply, then terraform completes
   → Lock automatically released
4. Developer B: Can now acquire lock and proceed
```

**Lock file structure (example):**
```json
{
  "ID": "gcp-tf-state-prod",
  "Operation": "OperationTypeApply",
  "Info": "",
  "Who": "user@company.com",
  "Version": "1.5.0",
  "Created": "2024-04-01T10:30:45Z",
  "Path": "terraform.tfstate"
}
```

### State Merging and Conflicts

**Real scenario: Two teams modify separate infrastructure components**

```
Team Infra:
  state: prod.tfstate
  manages: networking, databases, IAM roles

Team Apps:
  state: prod.tfstate (same file!)
  manages: GKE clusters, deployments, services
```

**What happens:**
```
1. Infra team: terraform apply → increments serial from 41 → 42
2. Apps team: terraform apply → increments serial from 41 → 42
   → Terraform detects serial mismatch!
   → ERROR: "Another apply is in progress"
```

**Solution: Separate state files**
```hcl
# Infra team: infra/terraform.tf
terraform {
  backend "gcs" {
    bucket = "tf-state-prod-infra"
    prefix = "v1"
  }
}

# Apps team: apps/terraform.tf
terraform {
  backend "gcs" {
    bucket = "tf-state-prod-apps"
    prefix = "v1"
  }
}

# Apps team references infra state
data "terraform_remote_state" "infra" {
  backend = "gcs"
  config = {
    bucket = "tf-state-prod-infra"
    prefix = "v1"
  }
}

resource "kubernetes_deployment" "app" {
  spec {
    # Access infra team's outputs
    env = {
      DB_ENDPOINT = data.terraform_remote_state.infra.outputs.cloudsql_endpoint
    }
  }
}
```

### State Refresh vs File Read

**Refresh: Reading actual state from provider**
```bash
terraform refresh
# Behind the scenes:
# - Calls AWS API: "What's the current state of i-0f1234567890abcde?"
# - Updates tfstate with actual values
# - Does NOT create/modify/delete anything
```

**When refresh happens:**
1. Automatic during `terraform plan`
2. Automatic during `terraform apply` (before creating changes)
3. Manual with `terraform refresh`
4. Can be skipped with `terraform plan -refresh=false` (dangerous!)

**Why refresh is important:**
```
Real incident:
1. Someone manually deleted security group via console
2. Terraform state still shows it exists
3. Next terraform apply tries to add rule to deleted group
4. API error: "resource not found"
5. Terraform panics
```

**With refresh:**
```
1. terraform plan starts
2. Refresh happens: calls "describe-security-group" API
3. API returns: "not found"
4. Terraform plan shows: "will need to recreate security group"
5. terraform apply fixes it
```

---

## 3. Why State Management is Critical

### State is Your Insurance Policy

**Scenario 1: Disaster Recovery**
- Database accidentally deleted in console
- Without state: manually recreate infrastructure (1-2 hours)
- With state: run `terraform apply`, full infra in 5 minutes

**Scenario 2: Team Coordination**
- Without state: "What resources exist? No idea, let me check console"
- With state: `terraform state list` shows everything

**Scenario 3: Audit Trail**
- Without state: no record of infrastructure changes
- With state in Git: each change has commit, reviewer, timestamp

**Scenario 4: Safety**
- Without state: `terraform destroy` can't know what NOT to destroy
- With state: only destroys resources it created

---

## 4. When to Use vs Not Use State Management

### Use Comprehensive State Management When:

- ✅ Production infrastructure (must be in state)
- ✅ Multiple team members accessing infrastructure
- ✅ Infrastructure changes need audit trail
- ✅ Disaster recovery is critical
- ✅ Infrastructure worth $ (everything!)

### Local State Only When:

- ❌ Solo developer in temporary environment
- ❌ Lab/proof-of-concept (< 24 hours)
- ❌ No sensitive resources
- ❌ No disaster recovery needed

### When to Split State Files:

- ✅ Different teams managing different concerns
- ✅ Infrastructure has natural boundaries
- ✅ One state file > 500 resources
- ✅ Different change frequencies (infra rarely changes, apps frequently)
- ✅ Need to apply changes independently

---

## 5. Real-world DevOps Usage

### Enterprise Pattern: GCP with Remote State

**Directory structure:**
```
terraform/
├── backends.tf
├── environments/
│   ├── dev/
│   │   ├── terraform.tfvars
│   │   ├── main.tf
│   │   └── .gitignore
│   ├── staging/
│   └── prod/
└── modules/
```

**backends.tf (defines state storage):**
```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.gcp_project
  region  = var.region
}
```

**environments/prod/terraform.tf (state backend):**
```hcl
terraform {
  backend "gcs" {
    bucket         = "tf-state-prod-company"
    prefix         = "infrastructure"
    encryption_key = "AEdPq3peHBGCSLgcz63PEe9G5J7BP...=="  # CMEK key
  }
}

terraform {
  required_version = "~> 1.5"
}
```

**environments/prod/terraform.tfvars:**
```hcl
gcp_project = "company-prod"
region      = "us-central1"
environment = "production"

instance_count = 5
machine_type   = "e2-standard-4"
enable_backups = true
```

**Actual workflow:**
```bash
cd environments/prod
terraform init
# Initialize remote state with encryption

terraform plan
# Read from gs://tf-state-prod-company/infrastructure/.terraform.lock.json
# Lock acquired, plan compares state with code
# Shows changes

terraform apply
# Acquire lock again
# Apply changes
# Write new state back to GCS
# Release lock
```

### Multi-environment State Management

**Scenario: Deploy same infrastructure to dev, staging, prod**

```hcl
# environments/common/variables.tf
variable "environment" {
  type = string
}

variable "instance_count" {
  type = number
}

# environments/dev/terraform.tfvars
environment    = "dev"
instance_count = 1

# environments/staging/terraform.tfvars
environment    = "staging"
instance_count = 3

# environments/prod/terraform.tfvars
environment    = "production"
instance_count = 10
```

**State files are isolated per environment:**
```
gs://tf-state-dev/infrastructure/
gs://tf-state-staging/infrastructure/
gs://tf-state-prod/infrastructure/
```

**Verify isolation:**
```bash
# Run in dev
cd environments/dev
terraform apply
# Updates gs://tf-state-dev only

# Prod is unaffected
cd environments/prod
terraform state list
# Still shows old resources, unmodified
```

### Secrets in State

**Problem: Passwords end up in state files**

```bash
# Example: RDS password in state
terraform state show aws_db_instance.prod
{
  "password": "SuperSecretPassword123!",  # EXPOSED!
  "username": "admin"
}

# If state goes to Git or exposed log
git log --all -p | grep PASSWORD
# PASSWORD LEAKED!
```

**Solution 1: Mark sensitive outputs**
```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

output "connection_string" {
  value     = "postgres://user:${var.db_password}@host"
  sensitive = true  # Terraform redacts from logs
}
```

**Solution 2: Use secret management provider**
```hcl
data "google_secret_manager_secret_version" "db_password" {
  secret = "prod-db-password"
}

resource "google_sql_instance" "prod" {
  root_password = data.google_secret_manager_secret_version.db_password.secret_data
}

# Password isn't in tfstate, only reference to secret
terraform state show google_sql_instance.prod
{
  "root_password": "",  # Not stored!
}
```

**Solution 3: Encrypt state file**
```hcl
terraform {
  backend "gcs" {
    bucket              = "tf-state-prod"
    prefix              = "infrastructure"
    encryption_key      = "AEdPq3peHBGCSLgcz63PEe9G5J7BP...=="  # CMEK
  }
}

# GCS now encrypts tfstate automatically with your KMS key
```

---

## 6. Architecture Thinking

### State Architecture Decision Tree

**Question 1: Solo Developer or Team?**
```
Solo + Temporary
  └─> Local state is fine
      If team later accesses: terraform state push to remote

Team + Permanent
  └─> Remote state mandatory
      (Prevents concurrent apply disasters)
```

**Question 2: How Many Resources?**
```
< 50 resources
  └─> Single state file OK

50-500 resources
  └─> Consider splitting by logical concern

> 500 resources
  └─> Must split
      Example: prod-infra, prod-database, prod-apps
      Each team gets own state
```

**Question 3: How Frequently Changes Occur?**
```
Infrastructure (rarely changes)
  └─> State file A
      VPCs, subnets, security groups
      Update: maybe monthly

Applications (frequently changes)
  └─> State file B
      Pod replicas, deployments, services
      Update: many times per day

Different teams can work independently
No blocking between infra and app changes
```

### State file Boundaries - Real Example

**Company Infrastructure:**

```
State 1: networking
  Resources: VPCs, subnets, routes, firewalls
  Owner: Platform team
  Change frequency: Monthly
  Why separate: Fundamental to everything, rarely changes

State 2: databases
  Resources: Cloud SQL, CloudSQL backups, database users
  Owner: DBA team
  Change frequency: Weekly
  Why separate: Different team, different expertise

State 3: kubernetes
  Resources: GKE cluster, node pools, storage classes
  Owner: Platform team
  Change frequency: Monthly
  Why separate: Infrastructure component

State 4: applications
  Resources: Kubernetes deployments, services, configmaps
  Owner: App development team
  Change frequency: Daily
  Why separate: Different team, very high change volume
```

**Dependency Flow:**
```
App state imports from Kubernetes state
      ↓
App state imports from Database state
      ↓
K8s state imports from Networking state
      ↓
All states reference Networking state

Result: Clear boundaries, parallel work possible
```

---

## 7. Common Mistakes

### Mistake 1: Storing Secrets in State

❌ **Wrong:**
```hcl
resource "google_sql_instance" "db" {
  root_password = "SuperSecretPassword123"  # HARDCODED!
}
gtfstate
{
  "password": "SuperSecretPassword123"  # EXPOSED IN STATE!
}
```

**Why it's bad:**
- State file might be in logs (terraform apply output)
- State file might go to Git if .gitignore is wrong
- Developers might accidentally print state
- Recovery from compromised password is complex

✅ **Right:**
```hcl
# Option 1: Generate and store in secret manager
resource "random_password" "db" {
  length = 32
}

resource "google_secret_manager_secret" "db_password" {
  secret_id = "prod-db-password"
}

resource "google_secret_manager_secret_version" "db_password" {
  secret      = google_secret_manager_secret.db_password.id
  secret_data = random_password.db.result
}

resource "google_sql_instance" "db" {
  root_password = random_password.db.result
}

# State shows:
{
  "root_password": ""  # Omitted from state logs
}
```

### Mistake 2: Committing State Files to Git

❌ **Wrong:**
```bash
git add terraform.tfstate
git commit -m "backup"

# Later discovered:
git log -p | grep password
# PASSWORD EXPOSED IN HISTORY!
# Even if deleted, it's in git history forever
```

✅ **Right:**
```bash
# .gitignore
*.tfstate
*.tfstate.*
terraform.tfstate.d/  # Workspaces

# Use remote backend instead
terraform {
  backend "gcs" {
    bucket = "tf-state"
  }
}
```

### Mistake 3: Different Teams Using Same State File

❌ **Scenario:**
```
Team infra: terraform apply to prod.tfstate
  Creates: VPC, databases, monitoring

Team apps: terraform apply to prod.tfstate
  Creates: GKE, deployments, services
```

**Problem:**
```
Both teams lock same state file
Terra apply from infra team locks state
Apps team must wait
One small infra change blocks all app deployments

Worst case:
Infra team destoys old version
State file deletes in wrong order
Apps team's resources now orphaned
```

✅ **Right:**
```
# Separate state files
terraform {
  backend "gcs" {
    bucket = "tf-state-infra"  # Team A only
  }
}

# Team B uses different state
terraform {
  backend "gcs" {
    bucket = "tf-state-apps"  # Team B only
  }
}

# Team B references Team A's outputs
data "terraform_remote_state" "infra" {
  backend = "gcs"
  config = {
    bucket = "tf-state-infra"
  }
}
```

### Mistake 4: Manual State File Editing

❌ **Tempting but dangerous:**
```bash
# Developer thinks it's faster to edit state directly
terraform state pull > state.json
# Edit state.json with text editor
terraform state push state.json

# Problems:
# - Breaks internal consistency
# - State checksums become invalid
# - Next apply may corrupt state
# - Breaks locks
```

✅ **Right:**
```bash
# Use official state commands
terraform state mv old_name new_name
terraform state rm resource_to_delete
terraform state list
terraform state show resource_name

# For importing unmanaged resources:
terraform import google_compute_instance.web \
  projects/my-project/zones/us-central1-a/instances/web-1
```

### Mistake 5: Not Backing Up State

❌ **Problem:**
```
State file on local laptop
Team member leaves company
State file lost
Must manually recreate all infrastructure or find ancient backup
```

✅ **Solution:**
```hcl
# Version-controlled backend with built-in history
terraform {
  backend "gcs" {
    bucket              = "tf-state-prod"
    prefix              = "infrastructure"
  }
}

# GCS features:
# - Object versioning enabled
# - Can restore old versions
# - Encryption at rest
# - Access logs for audit trail

# Verify
gsutil versioning set on gs://tf-state-prod
# Now can restore previous state versions
```

### Mistake 6: Losing Track of What State Manages

❌ **Scenario:**
```
Infra team creates resources via Terraform
App team manually creates some resources
Weeks later, app team forgets what was manual vs terraform-managed
Disaster when manual resources are deleted
```

✅ **Solution:**
```bash
# Always know what's managed
terraform state list
# Output shows all terraform-managed resources

terraform state show google_compute_instance.web
# Shows attributes and when created

terraform import
# Bring manual resources under management
# Now terraform knows about them

# Regular audits
terraform plan -refresh-only
# Shows any drift (manual changes)
```

### Mistake 7: Not Testing State Recovery

❌ **Problem:**
```
Assume state backups exist
When actually needed, backup is corrupted
Can't recover infrastructure
```

✅ **Solution:**
```bash
# Monthly: Test recovery process
# 1. List all available state versions
gsutil ls -L gs://tf-state-prod/infrastructure.tfstate

# 2. Copy old version
gsutil cp "gs://tf-state-prod/infrastructure.tfstate#VERSION" \
  gs://tf-state-prod/infrastructure.tfstate-backup

# 3. Test: Can terraform still plan/apply?
terraform plan

# 4. Restore
gsutil cp gs://tf-state-prod/infrastructure.tfstate-backup \
  gs://tf-state-prod/infrastructure.tfstate

# Regular testing prevents surprises
```

---

## 8. Interview Questions (Scenario-based)

### Q1: State File Corruption Under Pressure

**Interviewer:** "Your state file is corrupted (JSON syntax error). Production team needs to deploy NOW. Walk me through your fix."

**Good Answer:**
1. Don't panic, identify what's corrupted:
   ```bash
   terraform state pull | jq .
   # Returns JSON parse error with line number
   ```

2. Check backend history:
   ```bash
   # GCS versioning shows previous good state
   gsutil ls -L gs://tf-state-prod/
   ```

3. Restore previous version:
   ```bash
   gsutil cp "gs://tf-state-prod/tf.state#VERSION_BEFORE_CORRUPTION" \
     gs://tf-state-prod/tf.state
   ```

4. Verify:
   ```bash
   terraform plan -refresh-only
   # If works, state is recovered
   ```

5. If no good backup exists, rebuild from cloud:
   ```bash
   # Remove all resources from state
   terraform state list | xargs -I {} terraform state rm {}
   
   # Import resources back from cloud
   for resource in $(list_actual_cloud_resources); do
     terraform import $resource
   done
   ```

### Q2: Concurrent Team Applies

**Interviewer:** "Your infra team and app team both need to deploy at same time to prod. Same state file. What's the risk?"

**Good Answer:**
- **Risk:** Without state locking, both teams could apply conflicting changes
- Scenario:
  ```
  Infra team: terraform apply (locks state)
  App team: terraform apply 
    → Tries to acquire lock
    → Waits for lock release
    → Must wait 10-20 minutes for infra team
  ```
- **Better solution:** Separate state files
  ```
  Infra: gs://tf-state-infra/
  Apps: gs://tf-state-apps/
  
  Both can apply simultaneously
  Apps imports infra outputs via data source
  ```

### Q3: Disaster Recovery Scenario

**Interviewer:** "Production database was deleted. Your backup is fresh (1 hour old). Using Terraform, how long to restore?"

**Good Answer:**
1. Check if it's in state:
   ```bash
   terraform state show google_sql_instance.prod
   # (Returns database attributes)
   ```

2. Restore from backup and reapply:
   ```bash
   # In GCP console: restore from backup (takes 5-10 mins)
   
   # In Terraform, verify plan shows correct state:
   terraform plan
   # No changes needed (DB already restored outside terraform)
   ```

3. If needed to recreate:
   ```bash
   # Remove from state if in bad state
   terraform state rm google_sql_instance.prod
   
   # Run terraform apply, detects need to create
   terraform apply  # Recreates from code
   ```

**Timeline:** 5-15 minutes total (most is restore, not terraform)

### Q4: State Import Complexity

**Interviewer:** "A critical service was manually deployed (not managed by Terraform). We need to bring it under Terraform management safely. What's your process?"

**Good Answer:**
1. Generate import resource in code first:
   ```hcl
   resource "google_compute_instance" "manual_server" {
     # Empty, will be filled by import
   }
   ```

2. Import the resource:
   ```bash
   terraform import google_compute_instance.manual_server \
     projects/my-project/zones/us-central1-a/instances/prod-web-1
   ```

3. Generate the resource attributes:
   ```bash
   # Terraform now knows the resource, see what it imported
   terraform state show google_compute_instance.manual_server
   # Copy these attributes into .tf file
   ```

4. Verify no changes on apply:
   ```bash
   terraform plan
   # Should show: No changes, infrastructure up-to-date
   ```

5. Now it's managed:
   ```bash
   # Safe to modify via terraform going forward
   terraform apply
   ```

### Q5: State Splitting Decision

**Interviewer:** "Your Terraform has grown to 2000+ resources in one state file. Plan takes 15 minutes. What do you do?"

**Good Answer:**
- **Identify natural boundaries:**
  ```
  1. By team: infra team manages VPCs/networks
  2. By lifecycle: databases rarely change, apps change hourly
  3. By blast radius: each can be applied independently
  ```

- **Create separate state files:**
  ```
  tf-state-networking/
  tf-state-databases/
  tf-state-kubernetes/
  tf-state-applications/
  ```

- **Link them with data sources:**
  ```hcl
  data "terraform_remote_state" "networking" {
    backend = "gcs"
    config = {
      bucket = "tf-state-networking"
    }
  }
  
  resource "kubernetes_namespace" "apps" {
    metadata {
      labels = {
        network = data.terraform_remote_state.networking.outputs.network_id
      }
    }
  }
  ```

- **Result:** 15-minute plan becomes 3-4 minute plans per service

---

## 9. Debugging Issues

### Issue 1: "Failed to acquire lock" Error

```
Error: Error acquiring the state lock

Reason: "timeout waiting for HTTP 200 response"
```

**Diagnosis:**
```bash
# Who has the lock?
terraform force-unlock <LOCK_ID>
# (Shows who locked it)

# Or check backend lock file
gsutil cat gs://tf-state-prod/.terraform.lock.d/lock
{
  "ID": "...",
  "Who": "john@company.com",
  "Created": "2024-04-01T10:30:45Z"
}
```

**Solution:**
```bash
# If lock is stale (> 30 minutes old)
terraform force-unlock <LOCK_ID>

# Then retry
terraform apply
```

### Issue 2: "State Serial Mismatch"

```
Error: Error fetching state: 
Serial number mismatch, newer version exists in backend
```

**Cause:** Two applies happened from different states

**Diagnosis:**
```bash
# Check serial number
terraform state pull | jq .serial
# Output: 42

# But backend has serial 43
# Someone else applied between your plan and apply
```

**Solution:**
```bash
# Reload state and re-plan
terraform init  # Picks up new state from backend
terraform plan  # Re-plan based on new state
terraform apply
```

### Issue 3: "Resource Not Found" During Apply

```
Error applying plan:

google_compute_instance.web: Error creating instance:
Resource not found: projects/my-project/zones/us-central1-a/instances/web
```

**Cause:** Resource exists in state but not in cloud (manual deletion)

**Diagnosis:**
```bash
# Check state still shows it
terraform state show google_compute_instance.web
{
  "id": "web",
  "zone": "us-central1-a"
}

# But cloud doesn't have it
gcloud compute instances describe web --zone=us-central1-a
# ERROR 404: Instance not found
```

**Solution:**
```bash
# Re-import or remove from state
# Option 1: Let terraform recreate it
terraform apply
# Terraform notices it missing, recreates

# Option 2: Remove from state
terraform state rm google_compute_instance.web
terraform plan  # Shows it will create
terraform apply
```

### Issue 4: Infinite "Plan Shows Changes"

```bash
terraform plan
# Outputs: 1 to add, 0 to modify, 0 to destroy

terraform plan
# Same output again! (shouldn't happen)
```

**Diagnosis:**
```bash
# Check if computed value changes each read
terraform state show resource_name
# Look at attributes that change each refresh
```

**Solution:**
```hcl
# Ignore the computed fields
lifecycle {
  ignore_changes = [
    labels["auto_managed"],  # Changes automatically
    annotations["last_updated"]  # Computed by provider
  ]
}
```

### Issue 5: Remote State Not Updating

```
terraform apply
# Success message shows
# But state on backend didn't update
```

**Diagnosis:**
```bash
# Pull latest state
terraform state pull > state.json

# Compare with local
diff state.json .terraform/tfstate
```

**Cause:** Network timeout, write failed silently

**Solution:**
```bash
# Force push state
terraform state push state.json

# Verify
terraform state pull | grep serial
# Should reflect recent changes
```

---

## 10. Advanced Insights

### State Refactoring at Scale

**Challenge:** Reorganizing 500+ resources across teams without losing state

**Process:**
```bash
# 1. Create new state file for new team
$ mkdir team-a-new
$ cd team-a-new
$ terraform init -backend-config="bucket=tf-state-team-a-new"

# 2. Move resources from old to new state
$ terraform state pull gs://tf-state-old/tf.state | jq '.resources |= map(select(.type=="type_managed_by_team_a"))' > team-a.json

# 3. Push to new state
$ terraform state push team-a.json

# 4. Remove from old state
$ cd ../old
$ terraform state rm resource1 resource2 resource3...

# 5. Verify both maintain same actual cloud state
$ terraform plan -target=moved_resources
# Should show: No changes

# 6. Switch teams to new state
# Update backend configs
# Run terraform init
# Verify everything works
```

### State Optimization for Enterprise Scale

**Pattern: Federated State Management**

```
Central Team:
  State: tf-state-core (networking, IAM, security)
  
Service Team A:
  State: tf-state-svc-a
  Imports: tf-state-core outputs
  
Service Team B:
  State: tf-state-svc-b
  Imports: tf-state-core outputs
```

**Advantages:**
- Core infrastructure changes don't affect service teams
- Service teams can deploy independently
- Shared infrastructure versioned together
- Clear ownership boundaries

### State Encryption and Compliance

**Pattern: Encrypted State with Key Management**

```hcl
# Use CMEK (Customer Managed Encryption Keys)
terraform {
  backend "gcs" {
    bucket              = "tf-state-prod"
    encryption_key      = "projects/my-project/locations/us/keyRings/terraform/cryptoKeys/prod-state"
  }
}

# Terraform automatically encrypts state with your KMS key
# Key audit trail in Cloud KMS
# Compliance: meets data residency requirements

# Access control
resource "google_kms_crypto_key_iam_member" "terraform_sa" {
  crypto_key_id = google_kms_crypto_key.prod_state.id
  role          = "roles/cloudkms.users"
  member        = "serviceAccount:terraform-prod@${var.project}.iam.gserviceaccount.com"
}
```

---

## Key Takeaways

1. **State is the source of truth** - everything flows from correct state management
2. **Remote state with locking is mandatory** for teams
3. **Secrets should not be in state** - use secret management services
4. **Separate state files by team/concern** - enables parallel work
5. **State corruption recovery requires procedures** - test monthly
6. **State architecture impacts team velocity** - plan for growth
