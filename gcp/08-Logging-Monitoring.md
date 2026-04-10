# Cloud Logging & Cloud Monitoring

## 1. Core Concept (Deep Explanation)

**Cloud Logging** and **Cloud Monitoring** are Google's observability platform:

- **Cloud Logging**: Centralized log storage and analysis (all GCP services, applications, VMs)
- **Cloud Monitoring**: Metrics collection, dashboards, alerts (CPU, memory, custom metrics)
- **Cloud Trace**: Distributed tracing for latency analysis
- **Cloud Profiler**: Continuous application profiling (CPU, memory)

**Internal Architecture:**
- Logs shipped to Cloud Logging via Fluent Bit/Logstash
- Metrics collected by Prometheus-compatible agent
- Data stored in BigQuery for long-term analysis
- Logs retained 30-400+ days (configurable)
- Metrics retained 246 days (standard)

## 2. Why This Exists

**Problems it solves:**
- No log server to manage (fully managed)
- Unified logging across entire stack (VMs, containers, applications)
- Search and filter terabytes of logs instantly
- Correlate logs with metrics for troubleshooting
- Compliance: Audit trails for all access

## 3. When To Use

**Production systems require:**
- All application logs centralized
- Metrics from infrastructure
- Alerts on anomalies
- Dashboards for operations
- Distributed tracing for microservices

## 4. When NOT To Use

**Avoid:**
- Exporting all debug logs (too noisy, expensive)
- Storing personally identifiable information (PII)
- Real-time streaming for <100ms latency (use pub/sub instead)

## 5. Real-World Example: E-commerce Platform Observability

```yaml
# Application logging
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [SERVICE]
      Plugins_File /fluent-bit/plugins.conf
    
    [INPUT]
      Name systemd
      Tag host.*
      Read_From_Tail On
    
    [FILTER]
      Name kubernetes
      Match kube.*
      Keep_Log On
      K8S-Logging.Parser On
      K8S-Logging.Exclude On
    
    [OUTPUT]
      Name stackdriver
      Match *
      google_service_credentials /var/secrets/google/key.json
      k8s_cluster_name prod-cluster
      k8s_cluster_location us-central1

---
# Prometheus for metrics collection
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    
    scrape_configs:
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: keep
        regex: true
        source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]

---
# Cloud Monitoring dashboard
apiVersion: monitoring.cnrm.cloud.google.com/v1beta1
kind: MonitoringDashboard
metadata:
  name: ecommerce-dashboard
spec:
  displayName: E-commerce Platform
  dashboardFilters:
  - labelKey: resource.label.cluster_name
    templateVariable: CLUSTER
  
  gridLayout:
    widgets:
    - title: "API Latency (p95)"
      xyChart:
        dataSets:
        - timeSeriesQuery:
            timeSeriesFilter:
              filter: |
                metric.type="custom.googleapis.com/api_latency"
                AND metric.labels.percentile="p95"

    - title: "Error Rate %"
      xyChart:
        dataSets:
        - timeSeriesQuery:
            timeSeriesFilter:
              filter: |
                metric.type="custom.googleapis.com/api_errors"
                AND metric.labels.status="5xx"
```

## 6. Architecture Thinking

**Observability Stack:**

```
┌────────────────────────────────────────┐
│       Application / Infrastructure     │
│                                        │
│ ├─ Cloud Run (stderr/stdout)          │
│ ├─ GKE Pods (container logs)          │
│ ├─ Compute Engine (syslog)            │
│ ├─ Cloud SQL (query logs)             │
│ └─ Custom metrics                     │
└────────┬───────────────────────────────┘
         │
    ┌────┴─────┐
    │           │
    ▼           ▼
Cloud Logging  Cloud Monitoring
├─ Logs       ├─ Metrics
├─ Traces     ├─ Dashboards
├─ Audit logs ├─ Alerts
└─ Exports    └─ SLO tracking
    │           │
    └─────┬─────┘
          │
    ┌─────▼──────────┐
    │   BigQuery     │
    │   (analysis)   │
    └────────────────┘
```

**Log Flow:**

```
Application emits log line
    ▼
Fluent Bit (agent on node) collects
    ▼
Cloud Logging (fully managed backend)
    ├─ Stored in Stackdriver
    ├─ Searchable in UI
    ├─ Exportable to BigQuery
    └─ Retained 30-400+ days (configurable)
```

## 7. Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Logging everything | Expensive, noisy, hard to find issues | Log at WARN/ERROR level, use DEBUG selectively |
| No structured logging | Can't query logs effectively | Use JSON logging (severity, service, error_code) |
| No metrics | Can't see performance trends | Add custom metrics (latency, errors, business metrics) |
| Alerts on every metric spike | Alert fatigue, ignored alerts | Use thresholds, anomaly detection, composite alerts |
| No log retention policy | Old logs fill storage indefinitely | Set 30-90 day retention for cost control |
| Unrelated logs mixed | Can't correlate issues | Use different log names (app.log, access.log, error.log) |
| No request ID correlation | Can't trace request through services | Use `x-request-id` header across services |
| Not sampling high-volume logs | Expensive (millions of logs/second) | Sample 1% of logs for non-critical paths |
| Storing sensitive data in logs | PII exposed in logs | Redact SSN, credit cards, API keys from logs |
| No log aggregation | Can't compare metrics across services | Use Cloud Monitoring for aggregation |

## 8. Interview Questions (with Answers)

### Q1: Your app is slow. Design an investigation strategy using Cloud Logging and Monitoring.

**Answer:**
```bash
# Step 1: Check application latency metrics
gcloud monitoring time-series list \
  --filter 'metric.type="custom.googleapis.com/request_latency" AND resource.labels.service="api"' \
  --format='table(metric.labels.endpoint, points.value.double_val)' | head -10

# Step 2: Query Cloud Logging for slow requests
gcloud logging read \
  'jsonPayload.request_time_ms > 5000 AND severity="WARNING"' \
  --limit 50 \
  --format json

# Step 3: Check if specific endpoint is slow
gcloud logging read \
  'jsonPayload.endpoint="/api/orders" AND jsonPayload.status="200"' \
  --format='table(timestamp, jsonPayload.request_time_ms, jsonPayload.database_time_ms)' \
  --limit 20

# Step 4: Identify root cause
# - Database queries slow? Check Cloud SQL metrics
# - External API calls? Check latency to that service
# - CPU-bound? Check GKE node CPU

gcloud sql operations list --instance my-instance --limit 10
# Check if there are schema locks or long queries

# Step 5: Create dashboard to monitor
gcloud monitoring dashboards create \
  --config-from-file slow-endpoint-dashboard.json
```

**Diagnosis checklist:**
1. **Database latency**: Check `database_time_ms` in logs
2. **External API calls**: Check service-to-service latency
3. **Resource exhaustion**: Check CPU/memory on nodes
4. **Connection pool**: Check if connections are open
5. **Lock contention**: Check database locks

### Q2: Create a custom metric to track business events (order completed).

**Answer:**
```python
from google.cloud import monitoring_v3
from google.api_core import gapic_v1
import time

# Create metric descriptor
def create_metric(project_id):
    client = monitoring_v3.MetricServiceClient()
    project_name = f"projects/{project_id}"
    
    descriptor = monitoring_v3.MetricDescriptor()
    descriptor.type = "custom.googleapis.com/orders_completed"
    descriptor.metric_kind = monitoring_v3.MetricDescriptor.MetricKind.CUMULATIVE
    descriptor.value_type = monitoring_v3.MetricDescriptor.ValueType.INT64
    
    descriptor.description = "Total orders completed"
    descriptor.display_name = "Orders Completed"
    
    # Add custom label
    labels = []
    label = monitoring_v3.LabelDescriptor()
    label.key = "region"
    label.value_type = monitoring_v3.LabelDescriptor.ValueType.STRING
    labels.append(label)
    descriptor.labels = labels
    
    descriptor = client.create_metric_descriptor(
        name=project_name,
        metric_descriptor=descriptor
    )
    print(f"Created {descriptor.name}")

# Record metric value
def record_order_completed(project_id, region):
    """Increment order completed counter"""
    client = monitoring_v3.MetricsServiceClient()
    project_name = f"projects/{project_id}"
    
    series = monitoring_v3.TimeSeries()
    series.metric.type = "custom.googleapis.com/orders_completed"
    series.metric.labels["region"] = region
    
    # Resource labels
    series.resource.type = "global"
    series.resource.labels["project_id"] = project_id
    
    # Data point
    now = time.time()
    seconds = int(now)
    nanos = int((now - seconds) * 10 ** 9)
    interval = monitoring_v3.TimeInterval(
        {"seconds": seconds, "nanos": nanos}
    )
    point = monitoring_v3.Point(
        {"interval": interval, "value": {"int64_value": 1}}
    )
    series.points = [point]
    
    client.create_time_series(
        name=project_name,
        time_series=[series]
    )

# Usage
create_metric("my-project")
record_order_completed("my-project", "us-central1")
```

### Q3: Set up alert for high error rate (>5% 5xx errors).

**Answer:**
```yaml
# Cloud Monitoring Alert Policy
apiVersion: monitoring.cnrm.cloud.google.com/v1beta1
kind: MonitoringAlertPolicy
metadata:
  name: high-error-rate-alert
spec:
  displayName: "High Error Rate (>5%)"
  
  conditions:
  - displayName: "5xx error rate > 5%"
    conditionThreshold:
      filter: |
        metric.type="logging.googleapis.com/user/api_errors"
        AND metric.labels.status="5xx"
      
      aggregations:
      - alignmentPeriod: 60s  # 1 minute window
        perSeriesAligner: ALIGN_RATE
      
      comparison: COMPARISON_GT
      thresholdValue: 0.05  # > 5%
      
      duration: 300s  # Alert if sustained for 5 minutes

  notificationChannels:
  - "projects/my-project/notificationChannels/slack-channel"
  - "projects/my-project/notificationChannels/pagerduty-channel"
  
  alertStrategy:
    autoClose: 604800s  # Auto close after 7 days
    
  enabled: true
```

**Alternative using gcloud:**
```bash
# Create notification channel (Slack)
gcloud alpha monitoring channels create \
  --display-name="Slack Alerts" \
  --type=slack \
  --channel-labels=channel_name="#alerts"

# Create alert policy
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="High Error Rate" \
  --condition='
    metric.type="logging.googleapis.com/user/api_errors"
    AND metric.labels.status="5xx"
  ' \
  --condition-threshold=5 \
  --condition-duration=300s
```

### Q4: Design structured logging for an order API.

**Answer:**
```python
import json
import logging
import uuid
from datetime import datetime

# Configure structured logging
class StructuredLogFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'exception': None,
            'context': {}
        }
        
        # Add custom context if available
        if hasattr(record, 'request_id'):
            log_data['request_id'] = record.request_id
        if hasattr(record, 'service'):
            log_data['service'] = record.service
        
        # Add exception info
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)
        
        return json.dumps(log_data)

# Set up logging
handler = logging.StreamHandler()
handler.setFormatter(StructuredLogFormatter())
logger = logging.getLogger(__name__)
logger.addHandler(handler)

# Usage in API
from flask import Flask, request, g
import uuid

app = Flask(__name__)

@app.before_request
def setup_logging():
    g.request_id = str(uuid.uuid4())
    g.start_time = time.time()

@app.route('/orders', methods=['POST'])
def create_order():
    """Create order with structured logging"""
    
    # Create logger with context
    logger = logging.getLogger(__name__)
    logger.extra = {'request_id': g.request_id}
    
    try:
        logger.info(
            "Order created",
            extra={
                'request_id': g.request_id,
                'service': 'orders-api',
                'customer_id': request.json.get('customer_id'),
                'event': 'order.created'
            }
        )
        
        # Process order
        order = save_order(request.json)
        
        # Log with metrics
        logger.info(
            "Order saved successfully",
            extra={
                'request_id': g.request_id,
                'order_id': order['id'],
                'total': order['total'],
                'duration_ms': (time.time() - g.start_time) * 1000
            }
        )
        
        return {'order_id': order['id']}, 201
    
    except Exception as e:
        logger.error(
            "Order creation failed",
            extra={
                'request_id': g.request_id,
                'error': str(e),
                'error_type': type(e).__name__
            },
            exc_info=True
        )
        return {'error': 'Internal server error'}, 500

# Query structured logs
# gcloud logging read 'jsonPayload.event="order.created" AND jsonPayload.service="orders-api"'
```

### Q5: Correlate logs across microservices using request ID.

**Answer:**
```python
# Service A (Orders API)
from flask import Flask, request
import requests
import uuid

app = Flask(__name__)

@app.route('/orders', methods=['POST'])
def create_order():
    # Generate or extract request ID
    request_id = request.headers.get('X-Request-ID', str(uuid.uuid4()))
    
    logger.info(
        "Order creation started",
        extra={'request_id': request_id}
    )
    
    # Pass request ID to downstream service
    headers = {'X-Request-ID': request_id}
    
    # Call Payment Service
    response = requests.post(
        'http://payment-service/process-payment',
        json={'order_id': order['id'], 'total': order['total']},
        headers=headers
    )
    
    # Correlated logs example:
    # Service A: "Order created" request_id=abc123
    # Service B: "Payment processing" request_id=abc123
    # Service C: "Email sent" request_id=abc123
    # Now can search: gcloud logging read 'jsonPayload.request_id="abc123"'
    # And see entire flow across services

# Service B (Payment Service)
@app.route('/process-payment', methods=['POST'])
def process_payment():
    request_id = request.headers.get('X-Request-ID')
    
    logger.info(
        "Payment started",
        extra={'request_id': request_id}
    )
    
    # ... process payment ...
    
    logger.info(
        "Payment completed",
        extra={'request_id': request_id, 'charge_id': charge['id']}
    )
```

**Query correlated logs:**
```bash
# Find all logs for single request
gcloud logging read 'jsonPayload.request_id="abc123-def456"' \
  --format='table(timestamp, jsonPayload.service, jsonPayload.message)' \
  --limit 1000

# Result:
# timestamp                 service            message
# 2024-04-10T12:00:01Z     orders-api         Order creation started
# 2024-04-10T12:00:01Z     payment-api        Payment processing started
# 2024-04-10T12:00:02Z     payment-api        Stripe call completed (400ms)
# 2024-04-10T12:00:02Z     email-api          Email sent
```

### Q6: Implement SLO monitoring with error budget.

**Answer:**
```yaml
# SLO: 99.9% availability (max 43 seconds downtime per day)
apiVersion: monitoring.cnrm.cloud.google.com/v1beta1
kind: MonitoringServiceLevelObjective
metadata:
  name: api-availability-slo
spec:
  serviceLevelIndicator:
    indicator:
      requestBased:
        goodFilter: |
          metric.type="logging.googleapis.com/user/http_requests"
          AND metric.labels.status=~"2.."  # 200-299 status codes
        totalFilter: |
          metric.type="logging.googleapis.com/user/http_requests"
  
  goal: 0.999  # 99.9% uptime
  rollingPeriod: 2592000s  # 30 days
  
  displayName: "API Availability SLO"
  
  # If 99.9% SLO:
  # - 30-day window
  # - Error budget: 100% - 99.9% = 0.1%
  # - Errors allowed: 30 days * 0.1% = 43 seconds
```

**Monitor error budget:**
```bash
# Check current SLO status
gcloud monitoring slo list --service=my-service

# Query error budget consumption
gcloud logging read \
  'resource.type="api" AND protoPayload.status.code != 0' \
  --format='table(timestamp, protoPayload.status.code)' \
  --limit 100

# Alert when error budget exhausted
# Create alert if errors > 0.1% over 7-day rolling window
```

### Q7: Troubleshoot: Logs not appearing in Cloud Logging.

**Answer:**
```bash
# Check Fluent Bit configuration on node
kubectl logs -n kube-system -l app=fluent-bit --tail=50

# Check logging agent is running
gcloud compute instances describe INSTANCE \
  --zone ZONE \
  --format='table(metadata.items[key=google-logging-enabled])'

# Enable logging agent
gcloud compute instances create INSTANCE \
  --logging-enabled

# Check service account permissions
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:*@cloudservices.gserviceaccount.com"

# Grant logging writer role
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:gke-pod@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/logging.logWriter

# Test with direct write
gcloud logging write test-log "Test message" --severity=WARNING

# Check quota
gcloud compute project-info describe --project=PROJECT_ID | grep logging
```

### Q8: Compare Cloud Logging with Splunk/ELK.

**Answer:**

| Cloud Logging | Splunk | ELK Stack |
|-----------|--------|----------|
| Fully managed | Self-managed | Self-managed |
| GCP integrated | Multi-cloud | Multi-cloud |
| ~$0.50/GB | $10-15/GB | $3-5/GB (self-hosted) |
| Query language: SQL-like | SPL (Splunk) | Lucene/KQL |
| Real-time search | Real-time | Real-time |
| No scaling needed | Scaling complex | Requires management |
| BigQuery integration | Limited | Limited |

**Choose Cloud Logging if:**
- 100% GCP stack
- Preferring managed service
- Want BigQuery analysis

### Q9: Stream logs to BigQuery for analysis.

**Answer:**
```bash
# Create BigQuery dataset
bq mk --dataset \
  --description="Application logs analytics" \
  logs_analytics

# Create log sink (export logs)
gcloud logging sinks create logs-to-bigquery \
  bigquery.googleapis.com/projects/PROJECT_ID/datasets/logs_analytics \
  --log-filter='resource.type="cloud_run_revision"'

# Grant permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:service-PROJECT_NUMBER@gcp-sa-logging.iam.gserviceaccount.com \
  --role=roles/bigquery.dataEditor

# Query logs in BigQuery
bq query --use_legacy_sql=false '
SELECT
  timestamp,
  jsonPayload.request_id,
  jsonPayload.status,
  jsonPayload.latency_ms,
  COUNT(*) as count
FROM
  `project-id.logs_analytics.cloud_run_revision*`
WHERE
  DATE(timestamp) = CURRENT_DATE()
  AND jsonPayload.status != 200
GROUP BY
  timestamp, jsonPayload.request_id, jsonPayload.status, jsonPayload.latency_ms
ORDER BY
  jsonPayload.latency_ms DESC
'
```

### Q10: Set up log sampling for efficiency.

**Answer:**
```python
# Only log 1% of requests to reduce cost
import random

def should_log_request(request, sample_rate=0.01):
    """Sample based on request ID for consistency"""
    request_id = request.headers.get('X-Request-ID')
    
    # Deterministic sampling:
    # Same request_id always sampled same way
    hash_value = int(request_id.encode().hex(), 16) % 100
    return (hash_value / 100) < sample_rate

@app.route('/api/endpoint')
def endpoint():
    if should_log_request(request, sample_rate=0.01):  # 1%
        logger.info("Request processed", extra={'request_id': request_id})
    else:
        logger.debug("Request processed (not sampled)")
    
    # Alternative: Log only errors (100% sampling)
    try:
        result = process()
        return result
    except Exception as e:
        logger.error("Error processing", exc_info=True)  # Always log errors
        raise
```

## 9. Advanced Insights (Senior-level)

### Observability Best Practices

**Three Pillars:**
1. **Metrics**: CPU, memory, requests/sec, latency (p50, p95, p99)
2. **Logs**: Structured, JSON, searchable, correlated
3. **Traces**: end-to-end request flow across services

### Cost Optimization

**Reduce Cloud Logging costs:**
- Sample non-critical logs (1-10%)
- Exclude debug logs in production
- Archive to BigQuery only important logs
- Set 30-day retention limit

### SLO/SLI Strategy

**Define SLIs (Service Level Indicators):**
- Availability: % requests returning 2xx
- Latency: p99 < 500ms
- Error rate: < 0.1%

**SLO (Service Level Objective):**
- Availability: 99.9%
- Latency: p99 < 500ms 99% of time
- Error budget: 43 minutes downtime/month

