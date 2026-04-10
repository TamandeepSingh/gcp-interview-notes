# Cloud Storage

## 1. Core Concept (Deep Explanation)

Cloud Storage is Google's **object storage service** for unstructured data. Unlike block storage (Persistent Disk) or relational databases:

- **Object-based**: Store files as individual objects (not hierarchical filesystem)
- **Unlimited scale**: Petabytes of data with no performance degradation
- **Durability**: 11 nines (99.999999999%) replication
- **Global distribution**: Serve from nearest edge location via Cloud CDN
- **Versioning**: Keep object history for accidental deletion recovery

**Internal Architecture:**
- Objects stored in buckets (like AWS S3 buckets)
- Region/multi-region: Geographic replication
- Storage classes: Hot (Standard) → Cold → Archived based on access patterns
- Lifecycle policies: Auto-transition/delete old objects
- Signed URLs: Temporary access without authentication

## 2. Why This Exists

**Problems it solves:**
- Store unstructured data (images, videos, backups)
- Serve static content globally (CDN integration)
- Compliance: Immutable backups (retention policies)
- Cost: Archive old data (pay-per-access)

## 3. When To Use

**Best scenarios:**
- Application assets (images, videos, CSS, JS)
- Backups and archives (long-term retention)
- Machine learning training data
- Data lakes and data warehousing
- Log storage and analysis

## 4. When NOT To Use

**Avoid if:**
- Need filesystem operations (use Persistent Disk)
- Frequent object updates (versioning overhead)
- Structured queries (use BigQuery, Cloud SQL)
- Block storage requirements (use Persistent Disk)

## 5. Real-World Example: Multi-Region Media Application

```yaml
# Production setup: Global distribution of media
Bucket Configuration:
  name: company-media-prod
  location: US (multi-region)  # Replicated across 2+ US regions
  
  storage_class: STANDARD
  
  versioning:
    enabled: true  # Keep 30 days of versions
  
  lifecycle_rules:
  - action: SetStorageClass
    condition:
      age_days: 30
      storage_class: STANDARD
    new_storage_class: NEARLINE  # Move to cold storage
  
  - action: Delete
    condition:
      age_days: 365  # Delete after 1 year
  
  # Signed URLs for temporary access
  signed_urls:
    ttl: 1h
  
  # Immutable backups
  retention_policy:
    retention_days: 30  # Can't delete for 30 days
  
  # CDN for global delivery
  cdn_enabled: true
```

**Application code:**
```python
from google.cloud import storage
import json
from datetime import datetime, timedelta

# Upload with metadata
def upload_media(bucket_name, file_path, object_name):
    """Upload file to Cloud Storage"""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(object_name)
    
    # Set metadata
    blob.metadata = {
        'uploaded_by': 'media-processor',
        'timestamp': datetime.now().isoformat(),
        'source': 'user-upload'
    }
    
    # Upload with resumable upload (for large files)
    blob.upload_from_filename(
        file_path,
        content_type='application/octet-stream',
        timeout=600  # 10 minute timeout
    )
    
    return blob.public_url

# Generate signed URL for temporary access
def generate_signed_url(bucket_name, object_name, duration=3600):
    """Create signed URL valid for 1 hour"""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(object_name)
    
    url = blob.generate_signed_url(
        version="v4",
        expiration=timedelta(seconds=duration),
        method="GET"
    )
    
    return url

# Batch operations
def batch_delete_old_objects(bucket_name, prefix, days=90):
    """Delete objects older than N days"""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    
    cutoff_time = datetime.now() - timedelta(days=days)
    
    for blob in bucket.list_blobs(prefix=prefix):
        if blob.time_created < cutoff_time:
            blob.delete()
            print(f'Deleted {blob.name}')

# Stream large file processing
def process_log_files(bucket_name):
    """Process logs without downloading full file"""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    
    for blob in bucket.list_blobs(prefix='logs/'):
        # Download line by line for memory efficiency
        with blob.open('r') as f:
            for line in f:
                log_entry = json.loads(line)
                process_log(log_entry)
```

## 6. Architecture Thinking

**Cloud Storage in a Data Pipeline:**

```
┌──────────────┐
│ User upload  │
└──────┬───────┘
       │
       ▼
┌─────────────────────────────────────┐
│ Cloud Storage (standard, US region) │
│ Auto-replicated across US regions   │
└────────────┬─────────┬──────────────┘
             │         │
             │         └─→ Cloud CDN
             │            └─→ Global cache edges
             │
    ┌────────▼──────┐
    │ Pub/Sub trigger
    │ (new object)
    └────────┬──────┘
             │
    ┌────────▼──────────────┐
    │ Cloud Run Processor   │
    │ (transcoding, resize) │
    └────────┬──────────────┘
             │
    ┌────────▼──────────┐
    │ Cloud Storage     │
    │ (processed files) │
    └───────────────────┘
             │
    ┌────────▼──────────────┐
    │ BigQuery             │
    │ (analytics)          │
    └──────────────────────┘
```

**Storage classes and lifecycle:**

```
STANDARD (hot): Day 1-30
├─ Pay: $0.020/GB/month + API calls
├─ Use: Active data, frequently accessed
└─ Retrieval: Instant

NEARLINE (warm): Day 31-90
├─ Pay: $0.010/GB/month + Access charges ($0.010/GB)
├─ Use: Backed up instantly, infrequent access
└─ Retrieval: Instant (but charged)

COLDLINE (cold): Day 91-365
├─ Pay: $0.004/GB/month + Access charges ($0.004/GB)
├─ Use: Archive, compliance, rare access
└─ Retrieval: Instant (but charged)

ARCHIVE (very cold): Year+ storage
├─ Pay: $0.0012/GB/month + Access charges ($0.05/GB)
├─ Use: Long-term retention, compliance holds
└─ Retrieval: Instant but very expensive
```

## 7. Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Storing in single region | Data center failure = data unavailable | Use multi-region (US, EU, ASIA) |
| No lifecycle policy | Old data stored forever at high cost | Auto-transition to NEARLINE/COLDLINE |
| Public read access on sensitive data | Data exposed to internet | Use signed URLs or IAM authentication only |
| Not versioning production backups | Can't recover from accidental deletion | Enable versioning on backup buckets |
| No retention policy | Compliance data can be deleted | Set retention_days for regulatory compliance |
| Large object operations sync | App freezes during upload | Use resumable uploads, multipart for >100MB |
| No disaster recovery plan | Can't access data if primary region fails | Replicate to secondary region |
| Not using CDN for static assets | Slow for geographically distributed users | Enable Cloud CDN for images/CSS/JS |
| Overprovisioning storage class | Paying for hot storage on archive data | Use lifecycle policies for transitions |
| Not monitoring access patterns | Can't optimize storage class | Use Cloud Monitoring to track by storage class |

## 8. Interview Questions (with Answers)

### Q1: Design a cost-optimized storage strategy for 100GB of logs.

**Answer:**
```
Logs written daily (1-10GB/day)
├─ Days 1-7 (STANDARD): $0.020/GB = $0.14/month
│  └─ Fast retrieval for debugging crashes
├─ Days 8-30 (NEARLINE): $0.010/GB = $0.10/month
│  └─ Infrequent queries, still fast
├─ Days 31-365 (COLDLINE): $0.004/GB = $0.04/month
│  └─ Compliance retention, rarely accessed
└─ After 1 year (ARCHIVE): $0.0012/GB paid once then stored
   └─ Legal hold for 7 years

Total monthly: ~$0.28 for active logs
After 1 year: $0.012 * 365 = $4.38 annual storage
```

**Implementation:**
```python
from google.cloud import storage

# Create lifecycle policy
bucket = storage_client.bucket('logs-bucket')

rule1 = storage.bucket.LifecycleRuleDelete(age=365)
rule2 = storage.bucket.LifecycleRuleSetStorageClass(
    storage_class='NEARLINE',
    age=30
)
rule3 = storage.bucket.LifecycleRuleSetStorageClass(
    storage_class='COLDLINE',
    age=365
)

bucket.lifecycle_rules = [rule2, rule3]
bucket.patch()

# Set immutable retention for compliance
bucket.retention_period = 31536000  # 1 year in seconds
bucket.patch()
```

### Q2: Implement secure file sharing with temporary URLs.

**Answer:**
```python
from google.cloud import storage
from datetime import datetime, timedelta
import hashlib
import hmac
import base64

def create_signed_url(bucket_name, object_name, expiration_seconds=3600):
    """Generate signed URL valid for 1 hour"""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(object_name)
    
    url = blob.generate_signed_url(
        version="v4",
        expiration=timedelta(seconds=expiration_seconds),
        method="GET",
        content_type="application/octet-stream"
    )
    
    return url

def share_file_temporarily(receiver_email, file_path, duration=3600):
    """
    Share file secure way:
    1. Generate signed URL valid for 1 hour
    2. Send link via email
    3. Receiver downloads without authentication
    4. Link expires after 1 hour
    """
    
    signed_url = create_signed_url('company-files', file_path, duration)
    
    # Send email with link
    send_email(
        to=receiver_email,
        subject="File Download Link",
        body=f"Download link (expires in 1 hour): {signed_url}"
    )
    
    return signed_url

# Usage
share_file_temporarily('user@company.com', 'reports/q4-2024.pdf', duration=86400)
# Link valid for 24 hours
```

### Q3: How would you replicate Cloud Storage data to a secondary region?

**Answer:**
```bash
# Option 1: Multi-region bucket (automatic replication, same region choice)
gsutil mb -l US -b on gs://my-bucket/

# Option 2: Cross-region replication with Transfer Service
# Periodically sync to secondary region

gsutil -m cp -r gs://primary-bucket/* gs://secondary-bucket/

# Option 3: Application-level replication
# Every write to primary is replicated to secondary
```

**Python implementation:**
```python
from google.cloud import storage

def replicate_object(source_bucket, dest_bucket, object_name):
    """Replicate object to secondary region"""
    source = storage_client.bucket(source_bucket).blob(object_name)
    dest = storage_client.bucket(dest_bucket).blob(object_name)
    
    # Copy with metadata
    dest.upload_from_string(
        source.download_as_bytes(),
        content_type=source.content_type,
        metadata=source.metadata
    )

# Trigger on every write (in application code)
def save_file_replicated(bucket, object_name, data):
    """Save to primary and replicate to secondary"""
    
    # Write to primary
    dest = storage_client.bucket('primary-bucket').blob(object_name)
    dest.upload_from_string(data)
    
    # Replicate to secondary
    replicate_object('primary-bucket', 'secondary-bucket', object_name)
    
    # Return success
    return f"File saved and replicated: {object_name}"
```

### Q4: Explain storage classes and when to use each.

**Answer:**

| Class | Cost/GB/month | Retrieval | Use Case |
|-------|-----------|-----------|----------|
| STANDARD | $0.020 | Instant | Active data, frequently accessed |
| NEARLINE | $0.010 | Instant +$0.01 | Monthly backup access |
| COLDLINE | $0.004 | Instant +$0.004 | Yearly compliance archive |
| ARCHIVE | $0.0012 | Instant +$0.05 | 7-year legal hold, rarely accessed |

### Q5: Disable public access to a bucket for security.

**Answer:**
```bash
# Disable "public" role from bucket ACL
gsutil iam ch -d allUsers:objectViewer gs://my-bucket/

# Set uniform bucket-level access (no object-level ACLs)
gsutil uniformbucketlevelaccess set on gs://my-bucket/

# Enforce via Org Policy (project-wide)
gcloud resource-manager org-policies set-policy policy.yaml
# policy.yaml:
# constraint: storage.uniformBucketLevelAccess
# booleanPolicy:
#   enforced: true

# Now all access must go through IAM
gsutil iam ch serviceAccount:app@project.iam.gserviceaccount.com:objectAdmin gs://my-bucket/
```

### Q6: Upload large file (5GB) efficiently.

**Answer:**
```python
from google.cloud import storage
import os

def upload_large_file(bucket_name, file_path, object_name):
    """Upload 5GB file efficiently"""
    
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(object_name)
    
    # Resumable upload for large files
    # If connection breaks, resume from checkpoint
    blob.upload_from_filename(
        file_path,
        timeout=3600,  # 1 hour timeout
        num_retries=5
    )
    
    print(f'Uploaded {file_path} to {object_name}')

# Alternative: Streaming upload (memory efficient)
def stream_upload(bucket_name, object_name, file_size_gb):
    """Stream data without loading full file into memory"""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(object_name)
    
    # Write in chunks
    with open('large_file.bin', 'rb') as f:
        blob.upload_from_file(f, chunk_size=10 * 1024 * 1024)  # 10MB chunks

# Parallel upload with client libraries
from concurrent.futures import ThreadPoolExecutor

def parallel_upload(bucket_name, files):
    """Upload multiple files in parallel"""
    storage_client = storage.Client()
    
    def upload_file(file_path):
        bucket = storage_client.bucket(bucket_name)
        blob = bucket.blob(file_path.split('/')[-1])
        blob.upload_from_filename(file_path)
        return file_path
    
    with ThreadPoolExecutor(max_workers=5) as executor:
        futures = [executor.submit(upload_file, f) for f in files]
        results = [f.result() for f in futures]
    
    return results
```

### Q7: Track bucket costs and usage.

**Answer:**
```bash
# Enable usage logs
gsutil logging set on -b gs://logs-bucket/ -o usage-logs gs://my-bucket/

# Query usage in BigQuery
bq query --use_legacy_sql=false '
SELECT
  DATE(time_micros) as date,
  COUNT(*) as total_requests,
  SUM(bytes_received) as bytes_received,
  SUM(bytes_sent) as bytes_sent
FROM
  `project.dataset.usage_logs_*`
WHERE
  bucket_name = "my-bucket"
GROUP BY date
ORDER BY date DESC
'

# Set up billing alert
gcloud billing budgets create \
  --billing-account=BILLING_ACCOUNT_ID \
  --display-name="Storage Budget Alert" \
  --budget-amount=1000 \
  --threshold-rule=percent=80,percent=100 \
  --filter='service.description="Cloud Storage"'
```

### Q8: Implement immutable backup bucket for compliance.

**Answer:**
```bash
# Create bucket with retention policy
gsutil mb -l us-central1 gs://immutable-backups-prod/

# Set retention: can't delete for 30 days
gsutil retentionpolicy set 2592000 gs://immutable-backups-prod/

# Lock retention (irreversible!)
gsutil retentionpolicy lock gs://immutable-backups-prod/

# Now objects can't be deleted before 30 days
gsutil -h "x-goog-if-generation-match:0" cp backup.tar.gz gs://immutable-backups-prod/

# Verify retention
gsutil retentionpolicy get gs://immutable-backups-prod/
```

### Q9: Design disaster recovery with Cloud Storage.

**Answer:**
```
Primary Region (us-central1)
├─ Cloud Storage (multi-region US)
├─ Backup: daily snapshots
└─ Replication: object-level to secondary region

Secondary Region (us-east1)
├─ Cloud Storage (standby)
└─ Read replica of primary

Failure scenario:
1. Primary region fails
2. Switch DNS to secondary
3. Restore from most recent snapshot
4. Application continues
```

### Q10: Implement immutable versioning for disaster recovery.

**Answer:**
```python
from google.cloud import storage

def setup_versioned_backup(bucket_name):
    """Enable versioning for disaster recovery"""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    
    # Enable versioning
    bucket.versioning_enabled = True
    bucket.patch()
    
    # Set lifecycle: keep versions for 90 days
    rule = storage.bucket.LifecycleRuleDelete(
        num_newer_versions=3,  # Keep at least 3 versions
        age=90  # Delete versions older than 90 days
    )
    
    bucket.lifecycle_rules = [rule]
    bucket.patch()

def restore_from_version(bucket_name, object_name, version_id):
    """Restore object from specific version"""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    
    # Get specific version
    blob = bucket.blob(object_name, generation=version_id)
    
    # Restore (copy to current version)
    current_blob = bucket.blob(object_name)
    current_blob.upload_from_string(blob.download_as_bytes())
    
    return f"Restored {object_name} to version {version_id}"
```

## 9. Advanced Insights (Senior-level)

### Performance Optimization

**Multipart uploads:**
- Downloads/uploads >100MB benefit from parallel transfers
- 32 parallel streams recommended for optimal throughput

**Signed URLs:**
- Use for temporary secure access
- Include custom metadata for audit trail
- Supports HTTP methods: GET, PUT, DELETE, HEAD

### Cost Optimization

**Storage formula:**
```
Monthly cost = (Storage capacity * $/GB/month) + (API calls * $/1M requests) + (Egress GB * $/GB)
```

**Save 60%+ by:**
- Auto-transitioning to COLDLINE after 90 days
- Deleting data older than required retention
- Using ARCHIVE for compliance holds

### Security Hardening

**Defense in depth:**
- Uniform bucket-level access (no per-object ACLs)
- Signed URLs with expiration
- Encryption: Customer-managed keys (CMEK)
- VPC Service Controls for organizational boundary

