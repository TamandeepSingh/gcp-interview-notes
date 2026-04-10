# GCP Services - Production-Grade Infrastructure

## 1. Core GCP Services Landscape

```
┌─────────────────────────────────────────────────────────────┐
│             Google Cloud Platform Services                   │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│ COMPUTE         NETWORKING        STORAGE        DATA        │
│ ─────────────    ──────────────    ────────────   ──────    │
│ • GCE (VMs)     • VPC              • Cloud       • SQL    │
│ • GKE (K8s)     • Cloud CDN        Storage     • BigQuery│
│ • Cloud Run     • Cloud Load       • Firestore • Pub/Sub  │
│ • App Engine      Balancer         • Bigtable   • Dataflow│
│                 • Cloud           • Cloud               │
│ MANAGEMENT        Interconnect      Datastore         │
│ ──────────────                                       │
│ • GCP Compute  SECURITY & IAM    MONITORING          │
│ • Cloud Build  ──────────────────  ──────────────    │
│ • Cloud Ops    • IAM Roles         • Cloud Logging    │
│                • VPC Service       • Cloud Monitoring │
│                  Controls          • Cloud Trace      │
│                • Cloud KMS         • Cloud Profiler   │
│                • Secret Manager                       │
│                                                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Compute Services Decision Tree

### When to Use Each Service

```
┌─────────────────────────────────────────────┐
│ Need to run code?                           │
├─────────────────────────────────────────────┤
│                                             │
├─ Serverless                                │
│  ├─ No infrastructure management           │
│  ├─ Cloud Run (containers, any language)  │
│  ├─ Cloud Functions (lightweight, events) │
│  └─ App Engine (Python/Node/Go/Java)      │
│                                             │
├─ Kubernetes (container orchestration)     │
│  ├─ Multiple microservices                │
│  ├─ GKE (managed Kubernetes)              │
│  └─ Long-running services (months+)       │
│                                             │
├─ Traditional VMs (full control)           │
│  ├─ Compute Engine (GCE)                  │
│  ├─ Full OS control                       │
│  └─ Legacy/monolithic apps                │
│                                             │
└─────────────────────────────────────────────┘
```

---

## 3. Data & Storage Decisions

### Real-world Scenario: E-commerce Platform

```
Requirements:
- User accounts (structured, ACID) → Cloud SQL (PostgreSQL)
- Product catalog (large, searchable) → BigQuery
- Real-time inventory (fast reads) → Firestore
- Session data (temporary, fast) → Cloud Memorystore (Redis)
- Image files → Cloud Storage
- Events (logging) → Pub/Sub → BigQuery
- User analytics → Cloud Logging → BigQuery

Architecture:
┌────────────────────────────────────────────────────┐
│           E-commerce Platform                       │
├────────────────────────────────────────────────────┤
│                                                     │
│ Web App (GKE) → Cloud SQL (accounts)              │
│              → Firestore (inventory)              │
│              → Cloud Storage (images)             │
│              → Pub/Sub (events)                   │
│                    │                              │
│                    └─→ BigQuery (analytics)      │
│                                                     │
│ Redis Cache (Memorystore) ← Web App              │
│                                                     │
└────────────────────────────────────────────────────┘
```

---

## 4. Networking Architecture

### Multi-environment VPC Setup

```
GCP Project: company-prod
│
├─ VPC: prod-vpc
│  │
│  ├─ Subnet: us-central1/prod-subnet
│  │  ├─ GKE Cluster (pods: 10.4.0.0/14)
│  │  ├─ Cloud SQL Private IP
│  │  ├─ Cloud Memorystore (Redis)
│  │  └─ Firewall rules (internal only)
│  │
│  ├─ Subnet: us-east1/prod-subnet-dr
│  │  └─ Disaster recovery cluster
│  │
│  ├─ Cloud Router (NAT for private subnets)
│  │
│  └─ VPC Peering to on-prem network
│
├─ Cloud Load Balancer (external)
│  ├─ Public IP: 35.192.1.100
│  ├─ Routes to Cloud Armor (DDoS protection)
│  ├─ Routes to GKE ingress
│  └─ SSL/TLS termination
│
└─ Cloud Interconnect (dedicated connection to on-prem)
```

---

## 5. IAM & Security Pattern

### Least Privilege Setup

```yaml
# Service Account for Applications
apiVersion: iam.cnrm.cloud.google.com/v1beta1
kind: IAMServiceAccount
metadata:
  name: web-app-sa
spec:
  displayName: Web App Service Account

---
# Service Account only gets required permissions
apiVersion: iam.cnrm.cloud.google.com/v1beta1
kind: IAMPolicyMember
metadata:
  name: web-app-cloud-sql
spec:
  member: serviceAccount:web-app-sa@company-prod.iam.gserviceaccount.com
  role: roles/cloudsql.client  # Connect to Cloud SQL only
  resourceRef:
    apiVersion: sqladmin.cnrm.cloud.google.com/v1beta1
    kind: SQLInstance
    name: prod-database

---
# Web App can read secrets, but only specific ones
apiVersion: secretmanager.cnrm.cloud.google.com/v1beta1
kind: SecretIAMPolicy
metadata:
  name: db-password-access
spec:
  bindings:
    - members:
        - serviceAccount:web-app-sa@company-prod.iam.gserviceaccount.com
      role: roles/secretmanager.secretAccessor
  resourceRef:
    name: db-password  # Only this secret

---
# GKE pods use Workload Identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app
  namespace: production
  annotations:
    iam.gke.io/gcp-service-account: web-app-sa@company-prod.iam.gserviceaccount.com
```

**Result:**
- Web app runs under web-app-sa service account
- Only has permissions to: Cloud SQL, specific secrets
- Cannot access other services or projects
- All access is logged and auditable

---

## 6. Monitoring & Observability

### Production Monitoring Stack

```yaml
# Cloud Monitoring (Prometheus-compatible)
apiVersion: monitoring.cnrm.cloud.google.com/v1beta1
kind: MonitoringAlertPolicy
metadata:
  name: high-cpu-alert
spec:
  displayName: High CPU Usage
  conditions:
    - displayName: CPU > 80%
      conditionThreshold:
        filter: |
          resource.type="gke_container"
          AND metric.type="kubernetes.io/container/cpu/core_usage_time"
        comparison: COMPARISON_GT
        thresholdValue: 0.8
        duration: 300s  # 5 minutes
  notificationChannels:
    - /projects/company-prod/notificationChannels/123
  alertStrategy:
    autoClose: 1800s  # Auto-resolve after 30 min

---
# Cloud Logging (ELK alternative)
apiVersion: logging.cnrm.cloud.google.com/v1beta1
kind: LoggingLogView
metadata:
  name: app-errors
spec:
  filter: |
    resource.type="k8s_container"
    severity="ERROR"
    resource.labels.container_name="myapp"
```

**Observability:** 
- Logs: All pod logs automatically collected
- Metrics: Prometheus scraping built-in
- Traces: Cloud Trace for distributed tracing
- Dashboards: Custom dashboards in Cloud Console
- Alerts: Automated notifications on anomalies

---

## 7. Cost Optimization

### Real Cost Scenario

```
Compute (GKE):
  5 n2-standard-4 nodes × 24h × 30 days × $0.20 = $1,440/month
  ├─ Preemptible would be: $433/month (70% savings)
  └─ Reserved instances would be: $864/month (40% savings)

  Optimization:
  - Use Autopilot (Google manages nodes) → pay per pod
  - Mix committed use discounts (1-year) + Spot instances
  - Scale down off-hours (dev/staging only)

Database:
  Cloud SQL db-custom-4-15GB × 24h × 30 days = $780/month
  ├─ Dev should be: db-f1-micro = $9/month
  └─ Staging should be: db-custom-2-7GB = $195/month

  Optimization:
  - Separate instances per environment (dev/staging/prod)
  - Shared MySQL instance (cheaper than PostgreSQL)
  - Turn off backups in dev

Network:
  Data egress: $0.12/GB
  (Biggest cost driver if not optimized)

  Optimization:
  - Cloud CDN reduces egress
  - VPC Service Controls reduce inter-region traffic
  - Caching strategies (aggressive TTLs)

Total Optimization:
Before: ~$5,000/month
After:  ~$2,500/month (50% savings, same functionality)
```

---

## 8. Disaster Recovery

### Multi-region HA Setup

```yaml
# Primary Region: us-central1
# DR Region: us-east1

---
# GKE Cluster 1 (Primary)
apiVersion: container.cnrm.cloud.google.com/v1beta1
kind: ContainerCluster
metadata:
  name: prod-gke-primary
spec:
  location: us-central1-a
  networkRef:
    name: prod-vpc
  
  # High availability
  addonsConfig:
    networkPolicyConfig:
      disabled: false
    horizontalPodAutoscaling:
      disabled: false
  
  # Logging/monitoring
  loggingService: logging.googleapis.com/kubernetes
  monitoringService: monitoring.googleapis.com/kubernetes

---
# GKE Cluster 2 (DR)
apiVersion: container.cnrm.cloud.google.com/v1beta1
kind: ContainerCluster
metadata:
  name: prod-gke-dr
spec:
  location: us-east1-b
  networkRef:
    name: prod-vpc
  # Same config as primary

---
# Cloud SQL: Automatic failover to replica
apiVersion: sqladmin.cnrm.cloud.google.com/v1beta1
kind: SQLInstance
metadata:
  name: prod-database-primary
spec:
  region: us-central1
  failoverReplica:
    name: prod-database-replica
    # Automatic failover on primary failure
    configuration:
      region: us-east1

---
# Cloud Storage: Geo-redundant
apiVersion: storage.cnrm.cloud.google.com/v1beta1
kind: StorageBucket
metadata:
  name: prod-data-bucket
spec:
  location: US  # Multi-region (not single region)
  storageClass: STANDARD  # Replicates across US regions

---
# Global Load Balancer routes between regions
apiVersion: compute.cnrm.cloud.google.com/v1beta1
kind: ComputeBackendService
metadata:
  name: global-backend
spec:
  backends:
    - group: prod-gke-primary-backends
      balancingMode: RATE
      maxRatePerEndpoint: 100
    - group: prod-gke-dr-backends
      balancingMode: RATE
      maxRatePerEndpoint: 100
  
  # Health checks determine routing
  healthChecks:
    - /projects/company-prod/global/healthChecks/app-health

---
# Failover Test (monthly)
# 1. Verify DR cluster can access data
# 2. Shift traffic to DR temporarily
# 3. Verify functionality
# 4. Fail back to primary
# 5. Document results
```

**RTO/RPO:**
- RTO (Recovery Time Objective): 5-10 minutes
- RPO (Recovery Point Objective): 1 hour
- Real failover tested monthly

---

## 9. Common GCP Mistakes

### Mistake 1: Default VPC (Public)

❌ **Problem:**
```
All resources on default VPC
Default VPC allows communication between all instances
No network isolation
```

✅ **Right:**
```yaml
apiVersion: compute.cnrm.cloud.google.com/v1beta1
kind: ComputeNetwork
metadata:
  name: prod-vpc
spec:
  autoCreateSubnetworks: false  # Custom subnets only

---
# Private subnet (no default route to internet)
apiVersion: compute.cnrm.cloud.google.com/v1beta1
kind: ComputeSubnetwork
metadata:
  name: prod-subnet
spec:
  networkRef:
    name: prod-vpc
  region: us-central1
  ipCidrRange: "10.0.0.0/24"
  privateIpGoogleAccess: true  # Access Google APIs privately

---
# Cloud NAT for outbound internet (from private subnet)
apiVersion: compute.cnrm.cloud.google.com/v1beta1
kind: ComputeRouter
metadata:
  name: prod-router
spec:
  networkRef:
    name: prod-vpc
  region: us-central1

---
apiVersion: compute.cnrm.cloud.google.com/v1beta1
kind: ComputeRouterNat
metadata:
  name: prod-nat
spec:
  routerRef:
    name: prod-router
  region: us-central1
  sourceSubnetworkIpRangesToNat: LIST_OF_SUBNETWORKS
  natIps:
    - prod-nat-ip-1
    - prod-nat-ip-2
```

### Mistake 2: Storing Secrets in Code or Environment

❌ **Wrong:**
```python
# app.py
DB_PASSWORD = "production-db-123"  # In code!
```

✅ **Right:**
```python
import secret_manager

# Secret Manager (centralized, rotatable)
client = secret_manager.SecretManagerServiceClient()
password = client.access_secret_version(
    request={"name": "projects/prod/secrets/db-password/versions/latest"}
)

# Or in Terraform
resource "google_secret_manager_secret_version" "db_password" {
  secret      = google_secret_manager_secret.db_password.id
  secret_data = var.db_password  # From Terraform Cloud, encrypted
}
```

### Mistake 3: No IAM Audit Logging

❌ **Problem:**
```
Who changed IAM role of critical service account?
Who accessed sensitive data?
No audit trail
```

✅ **Right:**
```yaml
apiVersion: cloudaudit.cnrm.cloud.google.com/v1beta1
kind: CloudAuditConfig
metadata:
  name: enable-audit-logs
spec:
  service: allServices
  auditLogs:
    - service: allServices
      logType: ADMIN_WRITE  # Who changed resources
    - service: allServices
      logType: DATA_READ    # Who accessed data
    - service: allServices
      logType: DATA_WRITE   # Who modified data
```

Result: All activity logged in Cloud Logging, searchable, retained

---

## 10. Interview Scenario: Build Complete Platform

**Interviewer:** "Design GCP infrastructure for SaaS platform: 1,000 customers, 10 million requests/day, global deployment."

**Good Answer Structure:**

1. **Compute:**
   - GKE (multi-region for geo-proximity)
   - Workload Identity for pod-to-GCP auth
   - Autopilot for simplified management

2. **Data:**
   - Cloud SQL (PostgreSQL) per customer region
   - Firestore (real-time, scalable)
   - Cloud Storage(user uploads, geo-redundant)
   - BigQuery (analytics on logs)

3. **Networking:**
   - Private VPCs per customer (isolation)
   - VPC Service Controls (access boundaries)
   - Cloud Interconnect to on-prem

4. **Security:**
   - Service accounts (least privilege)
   - Secret Manager (secrets rotation)
   - Cloud Armor (DDoS protection)
   - VPC Security Controls

5. **Monitoring:**
   - Cloud Logging (all activity)
   - Cloud Monitoring (metrics/alerts)
   - Cloud Trace (debugging)
   - Custom dashboards

6. **HA/DR:**
   - Multi-region deployment
   - Automatic failover
   - RTO: 5 min, RPO: 1 hour

7. **Cost:**
   - ~ $50k/month infrastructure
   - Committed use discounts
   - Mix Spot + On-demand

---

## Key GCP Principles

1. **IAM First** - Principle of least privilege
2. **Encryption Everywhere** - At rest and in transit
3. **Private by Default** - Private networks, VPCs, secret management
4. **Observability Built-in** - Logging, monitoring, tracing
5. **Managed Services** - Let Google operate complexity
6. **Multi-region HA** - Survive regional failures
7. **Audit Everything** - Compliance and security
