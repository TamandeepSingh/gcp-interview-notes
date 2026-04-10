# Cloud SQL

## 1. Core Concept (Deep Explanation)

Cloud SQL is a **fully managed relational database service** running MySQL, PostgreSQL, or SQL Server. Unlike self-managed databases:

- **Automated Backups**: Daily backups with point-in-time recovery
- **High Availability (HA)**: Automatic failover with replica in different zone
- **Replication**: Read replicas for scaling reads
- **Maintenance Windows**: Google applies patches without downtime (HA mode)
- **Connection Security**: Private IP (VPC) and SSL encryption

**Internal Architecture:**
- Primary instance + standby replica (synchronous replication)
- Automatic failover if primary fails (<60 seconds)
- Storage: Persistent disks with snapshots
- Networking: VPC-native with private IP support
- Backups stored in Cloud Storage (encrypted)

## 2. Why This Exists

**Problems it solves:**
- No database administration overhead
- Automatic redundancy and failover
- Compliance-ready (encryption, audit logs, VPC isolation)
- Simplified scaling (read replicas for reads, primary for writes)

## 3. When To Use

**Best scenarios:**
- Transactional workloads (orders, payments)
- Structured data requiring ACID transactions
- Existing MySQL/PostgreSQL applications
- Multi-region read replicas needed
- Compliance requires encryption and audit trails

## 4. When NOT To Use

**Avoid if:**
- Unstructured data (use Cloud Storage, Firestore)
- Time-series data (use BigTable instead)
- NoSQL access patterns (use Firestore, Datastore)
- Very high write throughput (Cloud SQL limited to ~64k IOPS)

## 5. Real-World Example: E-commerce Database

```yaml
# High Availability Cloud SQL setup
apiVersion: cloudsql.cnrm.cloud.google.com/v1beta1
kind: CloudSQLInstance
metadata:
  name: prod-db-instance
spec:
  databaseVersion: POSTGRES_13
  region: us-central1
  
  # High Availability
  settings:
    availabilityType: REGIONAL  # HA mode
    backupConfiguration:
      enabled: true
      startTime: "03:00"  # UTC
      pointInTimeRecoveryEnabled: true
      transactionLogRetentionDays: 7
      backupRetentionSettings:
        retentionUnit: COUNT
        retainedBackups: 30
    
    # Database flags
    databaseFlags:
    - name: max_connections
      value: "500"
    - name: shared_buffers
      value: "262144"  # 2GB
    
    # Replication
    replicationConfiguration:
      kind: ASYNCHRONOUS
    
    # Backup location
    location_preference:
      zone: us-central1-a
    
    # IP management
    ipConfiguration:
      privateNetwork: "default"  # VPC
      enablePublicIp: false
    
    # Machine type
    tier: db-custom-8-32768  # 8 vCPU, 32GB RAM
```

**Application connection (with retry logic):**
```python
from sqlalchemy import create_engine, pool
from sqlalchemy.exc import SQLAlchemyError
import time

# Use Cloud SQL Auth proxy for secure connection
engine = create_engine(
    'postgresql://user:password@127.0.0.1/dbname',
    poolclass=pool.QueuePool,
    pool_size=20,
    max_overflow=40,
    pool_recycle=3600,  # Recycle connections after 1 hour
    pool_pre_ping=True,  # Test connection before using
    echo_pool=True  # Log pool operations
)

def insert_order_with_retry(order_data, max_retries=3):
    """Insert order with exponential backoff"""
    for attempt in range(max_retries):
        try:
            with engine.connect() as conn:
                result = conn.execute(
                    '''INSERT INTO orders (customer_id, total, status)
                       VALUES (%s, %s, %s)
                       RETURNING id''',
                    (order_data['customer_id'], 
                     order_data['total'], 
                     'pending')
                )
                conn.commit()
                return result.fetchone()[0]
        
        except SQLAlchemyError as e:
            if attempt < max_retries - 1:
                # Exponential backoff: 1s, 2s, 4s
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
```

## 6. Architecture Thinking

**Cloud SQL HA Architecture:**

```
┌────────────────────────────────────────────────────────┐
│              Application (Cloud Run / GKE)             │
└────────────────┬─────────────────────────────────────┘
                 │ SQL queries (connection pooling)
                 │
         Cloud SQL Proxy (on VM or via cloud_sql_python_connector)
                 │
    ┌────────────▼──────────────┐
    │   Cloud SQL Instance       │
    │   (HA enabled)            │
    │                           │
    │ Primary (us-central1-a)   │
    │ - Accepts reads & writes  │
    │ - Synchronous replication │
    │                           │
    ├─→ Standby (us-central1-b) │
    │   - Hot standby           │
    │   - Automatic failover    │
    └────────┬──────────────────┘
             │
    ┌────────▼──────────────┐
    │  Read Replicas        │
    │ (Asynchronous)        │
    │                       │
    ├─ us-east1 (read-only) │
    ├─ eu-west1 (read-only) │
    └───────────────────────┘
             │
    ┌────────▼──────────────┐
    │  Backup Storage       │
    │  (Cloud Storage)      │
    │                       │
    │ - Daily backups       │
    │ - 30-day retention    │
    │ - Point-in-time REST  │
    └───────────────────────┘
```

**Connection flow:**
```
1. App → Cloud SQL Proxy (or cloud_sql_python_connector)
   └─ Proxy handles IAM authentication, encryption
2. Proxy → Cloud SQL Private IP
3. Primary processes query
4. Standby receives synchronous replication
5. Return result to app
```

## 7. Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| No connection pooling | Too many connections, slow queries | Use pg_bouncer, SQLAlchemy pool |
| Public IP with SQL Auth | Security risk | Use Private IP (VPC) + IAM auth |
| No backups configured | Data loss on failure | Enable automated backups + PITR |
| HA disabled | Single point of failure | Always enable REGIONAL for production |
| Running as root user | Security risk, privilege escalation | Create limited service accounts |
| No SSL encryption | Data in transit unencrypted | Enforce SSL connections |
| Undersized machine | Slow queries, CPU throttling | Monitor CPU/memory, right-size |
| No read replicas | Reads contend with writes | Create read replicas for reporting |
| Not monitoring slow queries | Unknown bottlenecks | Enable query logging, set slow_query_log |
| Blocking schema changes | Zero downtime deployments impossible | Use online operations (PostgreSQL 11+) |

## 8. Interview Questions (with Answers)

### Q1: Your Cloud SQL instance has 80% CPU utilization. How do you diagnose and fix?

**Answer:**
```bash
# Check slow query log
gcloud sql operations list --instance=INSTANCE_ID | head -20

# View slow queries
gcloud logging read "resource.type=cloudsql_database AND severity=WARNING" --limit 50

# Check active connections
gcloud sql connect INSTANCE_ID -- \
  -c "SELECT datname, usename, COUNT(*) FROM pg_stat_activity GROUP BY datname, usename;"
```

**Diagnosis steps:**
1. **Check query patterns**: Are queries inefficient?
   ```sql
   -- PostgreSQL: Find slow queries
   SELECT query, calls, mean_time FROM pg_stat_statements
   ORDER BY mean_time DESC LIMIT 10;
   ```

2. **Add indexes**: Missing indexes cause full table scans
   ```sql
   CREATE INDEX idx_orders_customer ON orders(customer_id);
   ```

3. **Right-size machine**: If CPU still high after optimization
   ```bash
   gcloud sql instances patch INSTANCE_ID \
     --tier db-custom-16-65536  # Upgrade from 8 to 16 vCPU
   ```

4. **Add read replicas**: Distribute read traffic
   ```bash
   gcloud sql instances create INSTANCE_ID-replica \
     --master-instance-name INSTANCE_ID \
     --region us-east1
   ```

### Q2: Design a read-scaling strategy for 10M queries/day to Cloud SQL.

**Answer:**
```
Primary instance (writes): us-central1
    │
    ├─ Read Replica 1 (us-east1): 30% of read traffic
    ├─ Read Replica 2 (eu-west1): 30% of read traffic
    └─ Read Replica 3 (ap-south1): 30% of read traffic
    10% traffic to primary (failover)
```

**Application routing:**
```python
from google.cloud.sql.connector import Connector
import random

# Read-only replicas
read_replicas = [
    'project:us-east1:replica-1',
    'project:eu-west1:replica-2',
    'project:ap-south1:replica-3'
]

def get_connection(is_read=True):
    """Route queries efficiently"""
    if is_read:
        instance_id = random.choice(read_replicas)
    else:
        instance_id = 'project:us-central1:primary'  # Writes to primary
    
    connector = Connector()
    return connector.connect(instance_id, 'pymysql')

# Usage
write_conn = get_connection(is_read=False)
insert_order(write_conn, order_data)

read_conn = get_connection(is_read=True)
get_user_orders(read_conn, user_id)
```

**Load distribution:**
- Primary: ~50 QPS (writes + local reads)
- Each replica: ~3000 QPS (10M/86400 * 30%)
- Replication lag: 100-500ms (async)

### Q3: You need to restore database from 3 days ago. What's the process?

**Answer:**
```bash
# Option 1: Restore from backup (if available)
# List backups
gcloud sql backups list --instance INSTANCE_ID

# Restore to new instance
gcloud sql backups restore BACKUP_ID \
  --backup-instance INSTANCE_ID \
  --backup-configuration=default \
  --target-instance NEW_INSTANCE_ID

# Option 2: Point-in-time recovery (requires binary logging)
gcloud sql instances clone INSTANCE_ID restored-instance \
  --point-in-time='2024-04-07T12:00:00Z'

# Verify restoration
gcloud sql connect restored-instance -- \
  -c "SELECT COUNT(*) FROM orders;"
```

**Time comparison:**
- Full backup restore: 30-60 minutes (for 1TB)
- PITR: 15-30 minutes (uses binary logs)

### Q4: Design a green-blue deployment for schema migration.

**Answer:**
```bash
# Current: Blue instance (v1)
# New: Green instance (v1 + schema changes)

# Step 1: Create green replica
gcloud sql instances create prod-db-green \
  --master-instance-name prod-db-blue

# Step 2: Apply schema migrations on green
gcloud sql connect prod-db-green -- \
  -f schema_changes.sql  # Add new columns, indexes, etc.

# Step 3: Wait for replication to fully catch up
# Monitor: gcloud sql operations list --instance prod-db-green

# Step 4: Verify green replica is healthy
gcloud sql connect prod-db-green -- \
  -c "SELECT * FROM pg_stat_replication;"

# Step 5: Promote green to primary
gcloud sql instances promote-replica prod-db-green \
  --async

# Step 6: Update application connection string
# Blue (old) → Green (new)

# Step 7: Keep blue as standby for quick rollback
# If issues: gcloud sql instances failover prod-db-green
```

### Q5: Implement custom IAM authentication to Cloud SQL from GKE.

**Answer:**
```yaml
# Service Account for Pod
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default

---
# Bind KSA to GSA
apiVersion: iam.cnrm.cloud.google.com/v1beta1
kind: IAMServiceAccount
metadata:
  name: cloud-sql-gsa
spec:
  displayName: Cloud SQL Service Account

---
# Grant Cloud SQL Client role
apiVersion: iam.cnrm.cloud.google.com/v1beta1
kind: IAMPolicy
metadata:
  name: cloud-sql-iam-policy
spec:
  resourceRef:
    apiVersion: iam.cnrm.cloud.google.com/v1beta1
    kind: IAMServiceAccount
    name: cloud-sql-gsa
  bindings:
  - role: roles/cloudsql.client
    members:
    - serviceAccount:app-sa@project-id.iam.gserviceaccount.com

---
# Pod with workload identity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      serviceAccountName: app-sa
      annotations:
        iam.gke.io/gcp-service-account: cloud-sql-gsa@project-id.iam.gserviceaccount.com
      containers:
      - name: app
        image: gcr.io/project/app:latest
        env:
        - name: DB_HOST
          value: "127.0.0.1"
        - name: DB_USER
          value: "app@project-id.iam"
        volumeMounts:
        - name: cloud-sql-proxy
          mountPath: /cloudsql
      
      - name: cloud-sql-proxy
        image: gcr.io/cloudsql-docker/gce-proxy:1.35.2
        args:
        - "-instances=project:region:instance=tcp:5432"
        - "-credential_file=/secrets/service_account.json"
```

**Python app connection:**
```python
from google.cloud.sql.connector import Connector

connector = Connector()
conn = connector.connect(
    "project:region:instance",
    "pymysql",
    user="app@project-id.iam",
    db="mydb",
    enable_iam_auth=True  # Use IAM authentication
)
```

### Q6: Troubleshoot: "too many connections" error.

**Answer:**
```bash
# Check current connections
gcloud sql connect INSTANCE_ID -- \
  -c "SHOW max_connections;" 
  # Default: 100 for small instances

# Increase max connections
gcloud sql instances patch INSTANCE_ID \
  --database-flags max_connections=500

# Check connections per user
gcloud sql connect INSTANCE_ID -- \
  -c "SELECT usename, COUNT(*) FROM pg_stat_activity GROUP BY usename;"

# Kill idle connections
gcloud sql connect INSTANCE_ID -- \
  -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity 
      WHERE state = 'idle' AND state_change < now() - interval '30 minutes';"

# Implement connection pooling (recommended)
# Use PgBouncer as sidecar in GKE
```

### Q7: Compare Cloud SQL vs Firestore vs BigTable.

**Answer:**

| Dimension | Cloud SQL | Firestore | BigTable |
|-----------|-----------|-----------|----------|
| **Type** | Relational | Document/NoSQL | Wide-column |
| **Throughput** | 64k IOPS | 100k QPS | Millions QPS |
| **Queries** | SQL (complex joins) | Document queries | Key lookups |
| **Transactions** | ACID | Multi-doc transactions | Row-level atomic |
| **Scaling** | Vertical (mostly) | Automatic horizontal | Automatic horizontal |
| **Cost** | Hourly + storage | Request-based | Hourly + storage |
| **Use case** | Transactional apps | Mobile, web apps | Analytics, timeseries |

### Q8: Design high-availability database with global reads.

**Answer:**
```
Primary: us-central1 (accepts all writes)
    │
    ├─ Read Replica: us-east1
    ├─ Read Replica: eu-west1
    ├─ Read Replica: ap-southeast1
    └─ Read Replica: au-southeast1

Replication pattern:
Primary → Replica (async, ~100-500ms lag)

Application logic:
- Writes: Always to primary (strong consistency)
- Reads: Route to nearest replica (eventual consistency)
```

### Q9: Explain binary logging and PITR.

**Answer:**
```
Binary logging: Records all database changes
├─ Used for: Replication, PITR, audit trails
├─ Storage: On primary instance
└─ Retention: 7 days (configurable)

Point-in-time recovery:
├─ Take backup at day 1
├─ Replay binary logs from days 2-7
├─ Restore to exact timestamp (e.g., 3 days ago)
└─ Create new instance from restored state

Example:
gcloud sql instances clone prod prod-restored \
  --point-in-time='2024-04-07T12:00:00Z'
```

### Q10: Troubleshoot replication lag on read replicas.

**Answer:**
```bash
# Check replication lag (in seconds)
gcloud sql connect PRIMARY_ID -- \
  -c "SELECT slot_name, restart_lsn, 
      (pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) / 1024 / 1024 / 1024) as lag_gb,
      backend_start, state 
      FROM pg_replication_slots;"

# If lag is high (>1GB):
# 1. Check replica resource utilization
#    High CPU/disk I/O slows replication

# 2. Increase replica machine size
gcloud sql instances patch REPLICA_ID \
  --tier db-custom-16-65536

# 3. Monitor replica performance
gcloud sql instances describe REPLICA_ID | grep -i cpu

# 4. Check network latency between regions
#    High latency increases replication time
```

## 9. Advanced Insights (Senior-level)

### Performance Optimization

**Query optimization tips:**
- Add indexes on WHERE/JOIN columns
- Use EXPLAIN ANALYZE to understand query plans
- Partition large tables (billions of rows)
- Archive old data to Cloud Storage

**Connection pooling:**
- CloudSQL Auth Proxy: Handles security + connection pooling
- PgBouncer: Application-level connection pooling
- Pool size = (core count * 2) + effective_cache_size

### Cost Optimization

**Machine sizing:**
- db-custom-2-7680: ~$0.50/hour (small)
- db-custom-32-122880: ~$8/hour (large)
- Reserved instances: ~40% discount

### Security Hardening

**Defense in depth:**
- Private IP (VPC) + no public IP
- IAM authentication (no passwords)
- SSL encryption in transit
- Encrypted backups at rest
- Audit logging enabled

