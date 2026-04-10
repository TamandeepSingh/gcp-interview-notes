# Google Cloud Identity & Access Management (IAM)

## 1. Core Concept (Deep Explanation)

Cloud IAM is **Google's identity and access control system** that determines who can do what on which resources. It implements:

- **Principals**: Who (user, service account, group)
- **Roles**: What permissions (Owner, Editor, Viewer, custom)
- **Resources**: What things (VMs, Cloud SQL, storage buckets)
- **Conditions**: When/where (time-based, IP-based access)

**Internal Model:**
- Policy is hierarchically inherited (Organization → Folder → Project → Resource)
- Service account is a "bot" identity for applications
- Workload Identity maps Kubernetes SA to GCP SA (no key management)
- IAM policy is attached to resource (bind role to principal)

## 2. Why This Exists

**Problems it solves:**
- Principle of least privilege (grant exact permissions needed)
- Compliance (audit trails, separation of duties)
- No shared credentials (each service gets unique identity)
- Fine-grained control (who can delete resources at midnight)
- Service-to-service authentication without passwords

## 3. When To Use

**Every production setup requires:**
- Different roles for dev/staging/prod
- Service accounts for applications (not human credentials)
- Workload Identity in Kubernetes (bind pods to GCP service accounts)
- Conditions for sensitive operations (MFA, IP restrictions)

## 4. When NOT To Use

**Don't:**
- Use default service account (too permissive)
- Store credentials in code
- Share service account keys between services
- Grant project-level Editor to everyone

## 5. Real-World Example: Multi-Environment IAM Setup

```yaml
# Organization structure
Organization (root)
├─ Folder: Production (owns resources)
│  ├─ Project: prod-platform
│  ├─ Project: prod-data
│  └─ Project: prod-network
├─ Folder: Staging
└─ Folder: Development

# Service Accounts per environment
prod-app-sa@prod-platform.iam.gserviceaccount.com
staging-app-sa@staging-platform.iam.gserviceaccount.com
dev-app-sa@dev-platform.iam.gserviceaccount.com

# Custom roles
- Developer: Can deploy to staging, view prod
- DBA: Can manage databases, read backups
- DevOps: Can manage infrastructure
- OnCall: Emergency access (MFA required)
```

**IAM Bindings:**

```bash
# Service account for production app
gcloud iam service-accounts create prod-api-sa \
  --display-name="Production API Service Account"

# Grant Cloud SQL client role
gcloud projects add-iam-policy-binding prod-platform \
  --member serviceAccount:prod-api-sa@prod-platform.iam.gserviceaccount.com \
  --role roles/cloudsql.client

# Grant Cloud Storage reader (for assets)
gcloud projects add-iam-policy-binding prod-platform \
  --member serviceAccount:prod-api-sa@prod-platform.iam.gserviceaccount.com \
  --role roles/storage.objectViewer

# Custom role for sensitive operations
gcloud iam roles create customDeletePolicy \
  --project prod-platform \
  --title "Custom Delete" \
  --permissions compute.instances.delete,storage.buckets.delete

# Conditional role binding (MFA required for deletion)
gcloud projects add-iam-policy-binding prod-platform \
  --member user:devops@company.com \
  --role roles/iam.securityReviewer \
  --condition 'resource.name.endsWith("prod") && request.auth.claims.acr=="urn:mace:incommon:iap:silver"'
```

## 6. Architecture Thinking

**IAM Hierarchy:**

```
┌─────────────────────────────────────────────┐
│  Organization                              │
│  - Org Policy: Require MFA                │
│  - Service Account Admin: all projects     │
└────┬──────────── ┬──────────────┬─────────┘
     │             │              │
┌────▼────┐  ┌─────▼─────┐  ┌────▼────┐
│ Folder  │  │  Folder   │  │ Folder  │
│ Prod    │  │ Staging   │  │ Dev     │
│ Policy: │  │ Policy:   │  │ Policy: │
│ Encrypt │  │ Audit log │  │ Any     │
└────┬────┘  └─────┬─────┘  └────┬────┘
     │             │             │
┌────▼┐       ┌────▼─┐       ┌───▼───┐
│Proj│       │Proj  │       │Proj   │
│IAM:│       │IAM:  │       │IAM:   │
│Prod│       │Test  │       │Dev    │
│SA  │       │Eng   │       │Any    │
└────┘       └──────┘       └───────┘
```

**Permission Flow:**

```
User makes API call to compute.instances.list
    ↓
Cloud IAM checks:
├─ Is principal authenticated? (has valid credentials/tokens)
├─ Is principal authorized? (has role with compute.instances.list permission)
├─ Are conditions met? (IP whitelist, time window, MFA verified)
└─ Is resource accessible? (firewall allows, API quota available)
    ↓
Return result or 403 Forbidden
```

## 7. Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Using default service account | Too permissive (has Editor role) | Create custom service accounts per workload |
| Storing service account keys | Keys can be compromised | Use Workload Identity (no keys needed) |
| Granting Editor to entire team | Principle of least privilege violated | Grant specific roles (Viewer, Developer) |
| No audit logging of IAM changes | Can't track who deleted resources | Enable Cloud Audit Logs for IAM |
| Service account keys in code | Credentials compromised if repo leaked | Use Secret Manager or Workload Identity |
| Single service account for all apps | If one app compromised, all affected | One service account per logical service |
| No conditions on sensitive roles | Anyone with role can perform any action | Add time-based, IP-based conditions |
| Not rotating service account keys | Old keys still active after compromise | Rotate keys every 90 days |
| Forgetting to revoke access | Former employees still have access | Clean up IAM bindings on departure |
| No resource-level IAM | Project-level control too coarse | Use resource-level IAM for buckets, databases |

## 8. Interview Questions (with Answers)

### Q1: Design a secure multi-project strategy with least privilege access.

**Answer:**
```
Organization
├─ Prod Platform Project
│  └─ Service Accounts:
│     ├─ prod-api-sa (API role: Cloud SQL client, Storage viewer)
│     ├─ prod-worker-sa (Worker role: Log writer only)
│     └─ prod-backup-sa (Backup role: Cloud SQL backup creator)
├─ Prod Data Project (separate)
│  └─ Service Accounts:
│     └─ prod-data-sa (Analytics role: BigQuery reader, Cloud Storage reader)
└─ Shared Services Project
   └─ DNS, VPC management service accounts

IAM Bindings:
- prod-api-sa: 
  - roles/cloudsql.client (Cloud SQL in prod-platform)
  - roles/storage.objectViewer (bucket in prod-platform)
- prod-data-sa:
  - roles/bigquery.dataViewer (datasets in prod-data)
  - roles/storage.objectViewer (data bucket in prod-data)

Humans:
- Dev@company.com: 
  - roles/viewer (Prod Platform) - can view but not modify
  - roles/editor (Dev Platform) - can develop
- DevOps@company.com:
  - roles/editor (Prod Platform) - can manage infrastructure
  - roles/admin (Shared Services) - can manage DNS/VPC
  - Condition: IP whitelist + MFA for sensitive operations
```

### Q2: Implement Workload Identity for GKE pods accessing Cloud SQL.

**Answer:**
```bash
# 1. Create Kubernetes service account
kubectl create serviceaccount app-ksa --namespace default

# 2. Create GCP service account
gcloud iam service-accounts create app-gsa \
  --display-name="App Service Account"

# 3. Grant role to GCP service account
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member serviceAccount:app-gsa@PROJECT_ID.iam.gserviceaccount.com \
  --role roles/cloudsql.client

# 4. Bind KSA to GSA
gcloud iam service-accounts add-iam-policy-binding \
  app-gsa@PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT_ID.svc.id.goog[default/app-ksa]"

# 5. Annotate KSA
kubectl annotate serviceaccount app-ksa \
  iam.gke.io/gcp-service-account=app-gsa@PROJECT_ID.iam.gserviceaccount.com

# 6. Deploy pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: default
spec:
  serviceAccountName: app-ksa
  containers:
  - name: app
    image: gcr.io/project/app:latest
    env:
    - name: GOOGLE_APPLICATION_CREDENTIALS
      value: /var/run/secrets/cloud.google.com/service_account/key.json
EOF

# 7. Verify (pod can now access Cloud SQL without keys)
kubectl exec -it app-pod -- gcloud sql instances list
```

### Q3: Troubleshoot: User gets "Permission denied" error. How do you investigate?

**Answer:**
```bash
# 1. Check what the user is trying to do
# Get error message - what resource and action?
# Error: "compute.instances.delete" denied for "user@company.com"

# 2. Check user's current IAM bindings
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --format='table(bindings.role)' \
  --filter="bindings.members:user@company.com"

# 3. Check if role has permission
# Find role that user has
USER_ROLE=$(gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --format='value(bindings.role)' \
  --filter="bindings.members:user@company.com" | head -1)

# Check if role contains compute.instances.delete
gcloud iam roles describe $USER_ROLE --format='value(includedPermissions)' | grep compute.instances.delete

# 4. Check resource-level IAM
# If project-level doesn't grant permission, check resource
gcloud compute instances get-iam-policy INSTANCE_NAME \
  --filter="bindings.members:user@company.com"

# 5. Check conditions
# Are there conditions blocking access (time, IP)?
gcloud projects get-iam-policy PROJECT_ID \
  --format='table(bindings.condition.expression)' \
  --filter="bindings.members:user@company.com and bindings.condition"
```

**Fix: Grant necessary role**
```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member user:user@company.com \
  --role roles/compute.admin  # Or more specific role
```

### Q4: Compare predefined roles vs custom roles. When to create custom?

**Answer:**

| Predefined | Custom |
|-----------|--------|
| Maintained by Google | You maintain |
| ~80 roles available | Tailored to your needs |
| Include new permissions auto | Must update manually |
| Use when possible (simpler) | Use for specific requirements |

**When to create custom role:**
```bash
# Scenario: Software engineer needs to:
# - Deploy to Cloud Run (manage revisions)
# - View logs and metrics
# - BUT cannot delete instances or projects

gcloud iam roles create developmentEngineer \
  --project PROJECT_ID \
  --title "Development Engineer" \
  --description "Deploy and debug apps" \
  --permissions \
    run.services.update,\
    run.services.create,\
    run.revisions.delete,\
    logging.logEntries.list,\
    monitoring.timeSeries.list

# Use custom role
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member user:dev@company.com \
  --role projects/PROJECT_ID/roles/developmentEngineer
```

### Q5: Apply conditions to role binding. Restrict deletion to Monday-Friday, 9am-5pm UTC.

**Answer:**
```bash
# Create condition for time-based access
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member user:devops@company.com \
  --role roles/compute.admin \
  --condition='resource.matchTag("env", "prod") && 
              request.time.getHours("America/New_York") >= 9 && 
              request.time.getHours("America/New_York") <= 17 && 
              request.time.getDayOfWeek("America/New_York") >= 1 && 
              request.time.getDayOfWeek("America/New_York") <= 5'

# Condition expression language supports:
# - request.time (datetime conditions)
# - request.auth (authentication attributes, MFA status)
# - resource.name (resource path matching)
# - resource.matchTag (resource labels)
```

### Q6: Implement MFA requirement for sensitive operations.

**Answer:**
```bash
# Add condition requiring MFA
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member user:admin@company.com \
  --role roles/resourcemanager.projectDeleter \
  --condition 'request.auth.claims.acr == "urn:mace:incommon:iap:silver"'

# "urn:mace:incommon:iap:silver" = MFA authentication required

# Alternative: IP-based restriction
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member user:admin@company.com \
  --role roles/compute.admin \
  --condition 'origin.ip in ["203.0.113.0/24"]'  # Your office IP range
```

### Q7: Implement audit logging for all IAM changes.

**Answer:**
```yaml
# Cloud Audit Logs capture all IAM changes
# Enable in Cloud Logging

apiVersion: logging.cnrm.cloud.google.com/v1beta1
kind: LoggingLogView
metadata:
  name: iam-audit-log
spec:
  projectRef:
    name: my-project
  filter: |
    resource.type="service_account"
    protoPayload.methodName="SetIamPolicy"
    severity="NOTICE"

# Query who deleted a role binding
gcloud logging read \
  'protoPayload.methodName="SetIamPolicy" AND 
   protoPayload.status.code=0' \
  --limit 50 \
  --format json

# Set up alert for privilege escalation
gcloud alpha monitoring policies create \
  --notification-channels CHANNEL_ID \
  --display-name "IAM Role Grant Alert" \
  --condition 'protoPayload.methodName="SetIamPolicy"'
```

### Q8: Design service account key rotation strategy.

**Answer:**
```bash
# Current keys
gcloud iam service-accounts keys list \
  --iam-account sa-name@project.iam.gserviceaccount.com

# Create new key
gcloud iam service-accounts keys create new-key.json \
  --iam-account sa-name@project.iam.gserviceaccount.com

# Update application with new key
# Then delete old key
gcloud iam service-accounts keys delete KEY_ID \
  --iam-account sa-name@project.iam.gserviceaccount.com

# Automate with Cloud Scheduler
# Schedule weekly key rotation script
gcloud scheduler jobs create app-engine rotate-keys \
  --schedule "0 0 * * 0" \
  --timezone UTC \
  --http-method POST \
  --uri https://key-rotation-function.run.app/rotate
```

### Q9: Explain service account impersonation and when to use.

**Answer:**
```bash
# Service account impersonation: User A temporarily assumes permissions of Service Account B

# Grant permission to impersonate
gcloud iam service-accounts add-iam-policy-binding \
  target-sa@project.iam.gserviceaccount.com \
  --role roles/iam.serviceAccountUser \
  --member user:user@company.com

# Now user can impersonate when running gcloud commands
gcloud compute instances list \
  --impersonate-service-account target-sa@project.iam.gserviceaccount.com

# Use in code
from google.auth import iam
from google.auth.transport import requests

credentials = iam.Credentials.from_service_account_info(
    info,
    target_scopes=['https://www.googleapis.com/auth/cloud-platform'],
    target_principal='target-sa@project-id.iam.gserviceaccount.com'
)

# Scenario: Dev impersonates prod SA to debug production issue
# Leaves audit trail of who accessed prod
```

### Q10: Implement resource-level IAM for Cloud Storage bucket.

**Answer:**
```bash
# Grant user access to specific bucket only
gsutil iam ch user:user@company.com:objectViewer gs://my-bucket

# Grant role at object level (more granular)
gsutil iam ch user:user@company.com:objectAdmin gs://my-bucket/sensitive/

# Verify permissions
gsutil iam get gs://my-bucket | grep user@company.com

# Deny a specific action (deny always wins)
gcloud resource-manager org-policies set-policy policy.yaml \
  --project PROJECT_ID
# policy.yaml:
# constraint: storage.uniformBucketLevelAccess
# listPolicy:
#   deniedValues:
#   - "storage.inherited"  # Bucket-level ACLs not allowed
```

## 9. Advanced Insights (Senior-level)

### Security Best Practices

**Defense in depth:**
1. Organizational policies (enforce MFA, disable service account key creation)
2. Project-level IAM (least privilege roles)
3. Resource-level IAM (specific buckets, databases)
4. Conditions (time, IP, MFA requirements)
5. Workload Identity (no keys for apps in Kubernetes)
6. Audit logging (track all changes)

### Compliance Framework

**For HIPAA/PCI-DSS/SOC2:**
- Enable Cloud Audit Logs
- Implement 4-eye principle (two admins approve deletions)
- Use conditions for sensitive operations
- Service account keys rotated every 90 days
- MFA for all humans accessing production

