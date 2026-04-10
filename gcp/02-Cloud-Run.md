# Cloud Run

## 1. Core Concept (Deep Explanation)

Cloud Run is a **fully managed serverless compute platform** that executes containerized code without managing infrastructure. Unlike GKE where you manage nodes, Cloud Run:

- **Auto-scales to zero**: No running containers = no charges. Perfect consumption-based pricing.
- **Stateless execution**: Each request gets a fresh container instance
- **Built-in observability**: Automatic request logging, error reporting, profiling
- **Request-driven**: Scales based on incoming HTTP requests in real-time

**Internal Working:**
- Your container receives HTTP requests on port `$PORT` (default 8080)
- GVisor sandbox isolation (each container runs in lightweight VM)
- Request routed through Cloud Load Balancer globally
- Container lifecycle: Start on first request → Execute → Idle for 15 minutes → Terminate

## 2. Why This Exists

**Problems it solves:**
- **Zero Operations**: No node management, patching, scaling configuration
- **Cost Efficiency**: Pay exact milliseconds of execution (rounded to 100ms). Idle time costs nothing.
- **Rapid Deployment**: Deploy from container image or source code in seconds
- **Perfect for Microservices**: Each service is independent, scales independently
- **Webhook/Event Handler**: Ideal for Pub/Sub subscribers, Cloud Tasks consumers

## 3. When To Use

**Best scenarios:**
- APIs and webhooks (high traffic variability)
- Event-driven workflows (Pub/Sub subscribers)
- Background jobs triggered by Cloud Tasks
- Microservices with bursty traffic patterns
- Rapid prototyping and MVPs
- Continuous deployment pipelines
- Scheduled jobs (Cloud Scheduler → Cloud Run)

## 4. When NOT To Use

**Avoid if:**
- Application requires persistent state (Cloud Run is stateless)
- GPU/TPU processing (not supported)
- Long-running jobs >60 minutes (Cloud Run timeout)
- Needs direct access to VPC resources (use Cloud Run for VPC, but adds complexity)
- Requires TCP connections (Cloud Run uses HTTP/gRPC only)
- Complex networking needed (VPC needs additional setup)

**Tradeoffs:**
- Stateless only (no local storage between requests)
- Max request timeout 60 minutes
- Cold start latency (100-500ms first request)
- Image size limit 4GB per container
- Limited to HTTP/gRPC protocols

## 5. Real-World Example: E-commerce Order Processing Pipeline

```yaml
# Cloud Run service: API endpoint for orders
name: order-api
runtime: python39
serviceAccountEmail: order-service-sa@project-id.iam.gserviceaccount.com

automapping: true
port: 8080

# Auto-scales based on traffic
scaling:
  minInstances: 0      # Scale to zero on idle
  maxInstances: 1000   # Max concurrent instances
  
# Timeout for long operations
timeout: 60s

# Memory configuration
memory: 512Mi
cpu: 2

# Environment variables from Secret Manager
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: db-connection-string
  - name: STRIPE_API_KEY
    valueFrom:
      secretKeyRef:
        name: stripe-key

# Health check
livenessProbe:
  httpGet:
    path: /health
    port: 8080
```

**Workflow Implementation:**
```python
from flask import Flask, request
import cloud_sql_python_connector
import google.cloud.storage
from google.cloud import secretmanager

app = Flask(__name__)

@app.route('/orders', methods=['POST'])
def create_order():
    """
    1. Accept order from client
    2. Store in Cloud SQL
    3. Publish to Pub/Sub
    4. Return immediately
    """
    order_data = request.json
    
    # Validate
    if not validate_order(order_data):
        return {"error": "Invalid order"}, 400
    
    # Store in Cloud SQL
    db_connection = get_db_connection()
    order_id = db_connection.insert_order(order_data)
    
    # Publish async event
    publisher.publish(
        topic_path="projects/PROJECT_ID/topics/order-events",
        data=json.dumps({"order_id": order_id, "status": "created"}).encode()
    )
    
    # Return immediately - processing continues async
    return {"order_id": order_id}, 201

@app.route('/health')
def health():
    return {"status": "healthy"}, 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=os.environ.get('PORT', 8080))
```

**Processing Pipeline:**
```
Client Request
    ↓
Cloud Run API (order-api)
    ├─ Store in Cloud SQL
    └─ Publish to Pub/Sub (order-events)
        ↓
    Cloud Run Subscriber (payment-processor)
    ├─ Call Stripe API
    ├─ Update Cloud SQL
    └─ Publish to Pub/Sub (payment-complete)
        ↓
    Cloud Run Subscriber (notification-service)
    └─ Send email via SendGrid
```

## 6. Architecture Thinking

**Cloud Run in a Microservices Architecture:**

```
┌────────────────────────────────────────────────────┐
│              Client / Browser                       │
└──────────┬─────────────────────────────────────────┘
           │ HTTPS
           ▼
┌────────────────────────────────────────────────────┐
│    Cloud Load Balancer (Global)                    │
│    - TLS termination                               │
│    - Rate limiting                                 │
│    - DDoS protection                               │
└───┬──────────────────────────────┬────────────────┘
    │                              │
    ▼                              ▼
┌──────────────────────┐    ┌──────────────────────┐
│  Cloud Run Service   │    │  Cloud Run Service   │
│  (order-api)         │    │  (inventory-api)     │
│  Min: 0, Max: 1000   │    │  Min: 1, Max: 500    │
│  (Auto-scale)        │    │  (Always warm)       │
└──────────┬───────────┘    └─────────┬────────────┘
           │                          │
           └──────────┬───────────────┘
                      │
    ┌─────────────────┼─────────────────┐
    │                 │                 │
    ▼                 ▼                 ▼
Cloud SQL (Regional) Firestore       Cloud Storage
(Transactional)      (Documents)     (Images)
    │
    ├─ Replica in another region
    └─ Automated backups

    ▼
Pub/Sub (Topics)
    ├─ order-events
    ├─ payment-events
    └─ notification-events
        │
        ▼
    Cloud Run Subscribers
    ├─ payment-processor
    ├─ notification-service
    └─ analytics-ingestion
```

**Request Flow with Cold Starts:**

```
Request 1 (Cold start)
├─ Container image pulled & instantiated (500ms)
├─ Application startup (application code)
├─ Request executed (user code)
└─ Total: ~1000ms

Request 2 (Warm instance exists)
├─ Routed to warm instance
├─ Request executed
└─ Total: ~50-100ms

Idle for 15 minutes
└─ Instance terminated

Request 3 (New cold start)
└─ Same as Request 1 (~1000ms)
```

## 7. Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Not setting minInstances | Cold starts on all requests | Set minInstances=1 for critical APIs; costs ~$3/month |
| Trying to store state locally | Data lost between requests | Use Cloud SQL, Firestore, Cloud Storage for persistence |
| Not using Secret Manager | Credentials in environment variables | Inject secrets from Secret Manager via service account |
| Timeout set too low | Legit requests fail | Use Cloud Tasks + Cloud Run for long operations |
| Single region deployment | No redundancy | Deploy same service to multiple regions + Cloud LB |
| Not using Pub/Sub for async | Blocking requests timeout | Publish event, return immediately, process async |
| Oversizing container | High cold start time | Keep images <500MB; remove unnecessary dependencies |
| No health checks | Invalid instances serve traffic | Implement `/health` endpoint returning 200 on startup |
| Using large batch processing | OOM kills containers (512MB default) | Use Cloud Dataflow or GCE for batch; split into smaller tasks |
| Not monitoring cold starts | Performance degradation unnoticed | Monitor p95 latency; set up alerts for %cold_starts > 5% |

## 8. Interview Questions (with Answers)

### Q1: You have an order processing API. Sometimes it takes 45 seconds to call an external payment processor. Cloud Run has a 60-second timeout. Design a robust solution.

**Answer:**
Don't block the request for 45 seconds. Use asynchronous processing:

```python
@app.route('/orders', methods=['POST'])
def create_order():
    """
    Synchronous: Accept order, return immediately
    Asynchronous: Process payment in background
    """
    order = {
        'id': generate_order_id(),
        'items': request.json['items'],
        'total': calculate_total(request.json['items']),
        'status': 'pending'  # Not charged yet
    }
    
    # Store in Cloud SQL immediately
    db.insert_order(order)
    
    # Publish to Pub/Sub - payment processing happens async
    publisher.publish(
        topic_path=PAYMENT_TOPIC,
        data=json.dumps(order).encode()
    )
    
    # Return immediately - still within timeout
    return {'order_id': order['id'], 'status': 'pending'}, 202

# Separate Cloud Run service as Pub/Sub subscriber
@app.route('/process-payment', methods=['POST'])
def process_payment():
    """
    Called by Pub/Sub when message pushed
    Has full 60 seconds to:
    1. Call Stripe API
    2. Retry on failure
    3. Update order status
    """
    envelope = request.get_json()
    payload = base64.b64decode(envelope['message']['data'])
    order = json.loads(payload)
    
    try:
        # Call external API (can take 45 seconds)
        charge = stripe.charge.create(
            amount=order['total'],
            currency='usd',
            customer=order['customer_id'],
            idempotency_key=order['id']  # Idempotent
        )
        
        # Update order status
        db.update_order(order['id'], {
            'status': 'charged',
            'stripe_charge_id': charge.id
        })
        
        # Publish success event
        publisher.publish(COMPLETION_TOPIC, ...)
        
        return {}, 200
    
    except stripe.error.CardError as e:
        # Handle card decline
        db.update_order(order['id'], {'status': 'payment_failed'})
        return {}, 400
    except Exception as e:
        # Nack message so Pub/Sub retries
        return {}, 500  # Signals Pub/Sub to retry
```

**Architecture Summary:**
- API returns immediately → User sees "Processing order" status
- Pub/Sub delivery to payment processor is retried automatically
- Idempotency key prevents duplicate charges on retry
- Client polls `/orders/{id}` to check status

### Q2: Your Cloud Run service costs $500/month. You notice 60% of your traffic is during 8am-6pm (US timezone) and minimal at night. How do you optimize costs?

**Answer:**
Implement intelligent scaling with scheduled scaling:

```bash
# Use gcloud to set scaling based on time

# During peak hours (8am-6pm)
gcloud run services update my-service \
  --min-instances 10 \
  --max-instances 1000

# During off-peak hours (6pm-8am)
# Use Cloud Scheduler to trigger scale-down
gcloud scheduler jobs create app-engine scale-down \
  --schedule "0 18 * * *" \
  --timezone America/Los_Angeles \
  --http-method POST \
  --uri https://example.com/scale-down
```

**Alternative: Intelligent Scaling Script**

```python
from google.cloud import run_v2

def scale_service(min_instances):
    """Scale service based on time of day"""
    client = run_v2.ServicesClient()
    
    service = run_v2.Service(
        name='projects/PROJECT_ID/locations/us-central1/services/my-service',
        scaling=run_v2.ServiceScaling(
            min_instances=min_instances,
            max_instances=1000
        )
    )
    
    request = run_v2.UpdateServiceRequest(service=service)
    operation = client.update_service(request=request)
    return operation.result()

# Cloud Scheduler triggers this
def scale_endpoint(request):
    """HTTP endpoint for Cloud Scheduler"""
    from datetime import datetime
    hour = datetime.now().hour
    
    if 8 <= hour < 18:  # Peak hours
        min_instances = 10
    else:
        min_instances = 0
    
    scale_service(min_instances)
    return {'min_instances': min_instances}, 200
```

**Cost Analysis:**
- Peak: 10 instances × $0.00002400/vCPU-sec × 3600s × 10hrs/day × 22 days/month ≈ $190/month
- Off-peak: 0 instances × 1-2 hrs/month still serving = $5/month
- **Total: ~$195/month (61% reduction from $500)**

**Additional optimization:**
- Increase request size before scaling out (reduce cold starts)
- Use regional endpoints vs global (reduce distance)
- Cache responses in Cloud CDN

### Q3: Compare Cloud Run vs GKE vs App Engine. When would you use each?

**Answer:**

| Aspect | Cloud Run | GKE | App Engine |
|--------|-----------|-----|-----------|
| **Scaling** | Auto to 1000s | Manage node pools | Auto-managed |
| **Startup time** | 100-500ms | Seconds (pod scheduling) | Seconds |
| **Usage pattern** | Request-driven | Always running | Request-driven (legacy) |
| **Cost for bursty** | Very cheap (scale to zero) | Higher (minimum nodes) | Cheaper than GKE |
| **Control** | Limited (stateless) | Full control | Limited |
| **Cold starts** | Possible | Rare (always warm) | Possible |
| **Skills required** | Docker | Kubernetes expertise | App Engine specific |

**Decision tree:**
```
Need to scale to zero automatically?
├─ Yes → Cloud Run ✓
├─ No → GKE or App Engine?
    ├─ Need full orchestration control?
    │  ├─ Yes → GKE ✓
    │  └─ No → App Engine (legacy, use Cloud Run instead)
```

**Real-world example:**
- **Cloud Run**: API endpoints, webhooks, event handlers
- **GKE**: Monolithic application, complex networking, stateful workloads
- **App Engine**: Legacy Python/Java apps already on GAE

### Q4: Design a multi-region Cloud Run deployment with zero downtime failover.

**Answer:**
```yaml
# Primary region: us-central1
apiVersion: cloud.google.com/v1
kind: Service
metadata:
  name: api-service-primary
  region: us-central1
spec:
  template:
    spec:
      containers:
      - image: gcr.io/project/api:latest
        env:
        - name: REGION
          value: primary
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-connection-primary

---
# Secondary region: us-east1
apiVersion: cloud.google.com/v1
kind: Service
metadata:
  name: api-service-secondary
  region: us-east1
spec:
  template:
    spec:
      containers:
      - image: gcr.io/project/api:latest
        env:
        - name: REGION
          value: secondary
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-connection-secondary

---
# Global Load Balancer for failover
apiVersion: compute.cnrm.cloud.google.com/v1beta1
kind: ComputeBackendService
metadata:
  name: multi-region-backend
spec:
  loadBalancingScheme: EXTERNAL
  protocol: HTTPS
  sessionAffinity: CLIENT_IP
  
  backends:
  - group: cloud-run-neg-us-central1  # NEG = Network Endpoint Group
    balancingMode: RATE
    maxRatePerEndpoint: 1000
  - group: cloud-run-neg-us-east1
    balancingMode: RATE
    maxRatePerEndpoint: 1000
  
  healthChecks:
  - name: api-health-check
    checkIntervalSec: 10
    timeoutSec: 5
    healthyThreshold: 1
    unhealthyThreshold: 3
    httpHealthCheck:
      port: 443
      requestPath: /health
      proxyHeader: NONE
```

**Traffic routing logic:**
1. Cloud Load Balancer receives request
2. Route to primary (us-central1)
3. If primary health check fails → Route to secondary (us-east1)
4. No DNS propagation delay (health check-based switch)

**Testing failover:**
```bash
# Kill primary service
gcloud run services delete api-service-primary --region us-central1

# Traffic automatically switches to secondary (measured in seconds)
curl https://api.example.com  # Still works!

# Re-enable primary with new version
gcloud run deploy api-service-primary \
  --region us-central1 \
  --image gcr.io/project/api:new-version
  
# Traffic gradual returns to primary (depends on backend distribution)
```

### Q5: Explain Cloud Run's execution model. What happens during startup?

**Answer:**
```
Request arrives at global load balancer
    ↓
Route to nearest Cloud Run instance
    ↓
Instance exists and ready?
├─ Yes → Route request immediately
└─ No → Create new instance
    ├─ Pull container image from Artifact Registry
    ├─ Create gVisor sandbox (lightweight VM isolation)
    ├─ Start container runtime (runc)
    ├─ Execute container entrypoint
    ├─ Container starts listening on $PORT
    └─ (All previous steps: 100-500ms - "cold start")
         ↓
    Route request to instance
         ↓
    Container executes request handler
         ↓
    Return response (JSON, HTML, etc)
         ↓
    Instance stays alive in memory (instanceReuse)
    └─ Handles 1000s of concurrent requests
         ↓
    After 15 minutes of idle
    └─ Instance terminated, memory released
```

**Key design decisions:**
- **Per-request lifecycle**: Each HTTP request gets CPU time
- **Concurrent requests**: Single instance handles multiple requests via async event loop
- **gVisor isolation**: Each container in lightweight VM (security boundary)
- **Automatic scaling**: Incoming requests trigger instance creation
- **Graceful shutdown**: Container receives SIGTERM signal before termination

### Q6: How does Cloud Run handle networking? Can it access VPC resources?

**Answer:**
**Before Cloud Run VPC Connector (legacy):**
- Cloud Run could only reach external internet
- Could NOT access internal VPC resources (Cloud SQL, private instances)

**With VPC Connector (modern solution):**
```bash
# Create VPC connector (runs in VPC, forwards traffic)
gcloud compute networks vpc-access connectors create my-connector \
  --network default \
  --region us-central1 \
  --range 10.8.0.0/28

# Deploy Cloud Run with VPC connector
gcloud run deploy my-service \
  --image gcr.io/project/app \
  --vpc-connector my-connector \
  --region us-central1
```

**Architecture:**
```
Cloud Run Container
    ↓
Wants to connect to Cloud SQL (10.0.1.5)
    ↓
VPC Connector (in VPC)
    ├─ acts as sidecar proxy
    └─ forwards traffic through VPC
         ↓
    Cloud SQL Private IP (10.0.1.5)
         ✓ Connection established
```

**Limitations:**
- Connector adds ~50ms latency
- Connector in specific region (ensure same region as service)
- Max throughput dependent on connector machine type

**Better alternative (Cloud Run for VPC):**
```bash
gcloud run deploy my-service \
  --network my-vpc \
  --subnet my-subnet \
  --image gcr.io/project/app
```
- Cloud Run instance runs directly in VPC
- No connector needed
- Lower latency
- Still can reach external internet

### Q7: Your Cloud Run service suddenly has increased error rates. How do you diagnose?

**Answer:**
```bash
# 1. Check recent deployments
gcloud run revisions list --service my-service --region us-central1

# 2. View Cloud Logging for errors
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=my-service" \
  --limit 100 \
  --format json

# 3. Check Cloud Trace for latency increase
gcloud trace list --filter 'labels.http_status!=200' --limit 50

# 4. View Cloud Monitoring metrics
gcloud monitoring metrics-descriptors list --filter 'metric.type:run'

# 5. Check container logs directly
gcloud run logs read my-service --region us-central1 --limit 100

# 6. Compare with previous revision
gcloud run revisions update-traffic my-service \
  --to-revisions REVISION_NAME=0 \
  --region us-central1
  # Switch back to previous version to test
```

**Common causes:**

1. **Memory issues**: Container OOMKilled
   - Check: `gcloud run services describe my-service` → memory limit
   - Fix: `gcloud run deploy ... --memory 1Gi`

2. **Timeout issues**: Requests taking >60s
   - Check: `gcloud logging read ... | grep timeout`
   - Fix: Reduce timeout or use async with Pub/Sub

3. **Dependency failure**: External API down
   - Check: `gcloud logging read ... | grep "connection refused"`
   - Fix: Implement circuit breaker, fallback

4. **Resource exhaustion**: Too many concurrent requests, containers unable to scale
   - Check: `gcloud monitoring read "metric.type=run.googleapis.com/request_count" --filter 'resource.labels.service_name=my-service'`
   - Fix: Increase max instances or add rate limiting

### Q8: Compare Cloud Run with AWS Lambda. What's different?

**Answer:**

| Dimension | Cloud Run | Lambda |
|-----------|-----------|--------|
| **Container** | Full container image (4GB max) | Zip file or container (250MB unzipped) |
| **Runtime** | Any language in container | Specific runtimes (Python, Node, Go, etc.) |
| **Execution model** | Concurrent requests per instance | One request per Lambda (new instance each time) |
| **Networking** | HTTP/gRPC via load balancer | Synchronous HTTP, event-driven |
| **Cold start** | 100-500ms | 1-10 seconds (Lambda slower) |
| **Pricing** | Per 100ms | Per 100ms + GB-second |
| **State** | Can use local temp storage | No local state between invocations |

**Cloud Run advantages:**
- Faster cold starts
- True concurrency (multiple requests per instance)
- Standard containerization (not AWS-specific)
- Works with any language

### Q9: You need to process files from Cloud Storage, transform them, and store results. Design a Cloud Run solution.

**Answer:**
```yaml
# Architecture: Storage Trigger → Pub/Sub → Cloud Run Processor

# Step 1: Cloud Storage bucket
# Create topic for object notifications
gsutil notification create -t projects/PROJECT_ID/topics/gcs-uploads gs://my-bucket

# Step 2: Cloud Run service as Pub/Sub subscriber
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: file-processor
spec:
  template:
    spec:
      serviceAccountName: file-processor-sa
      containers:
      - image: gcr.io/project/file-processor:latest
        env:
        - name: OUTPUT_BUCKET
          value: gs://output-bucket
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: 2Gi

# Step 3: Pub/Sub subscription
gcloud pubsub subscriptions create gcs-uploads-sub \
  --topic gcs-uploads \
  --push-endpoint https://file-processor-REGION-PROJECT_ID.run.app/process \
  --push-auth-service-account file-processor-sa@PROJECT_ID.iam.gserviceaccount.com
```

**Cloud Run implementation:**
```python
from flask import Flask, request, jsonify
from google.cloud import storage
import json
import base64
import logging

app = Flask(__name__)
storage_client = storage.Client()

@app.route('/process', methods=['POST'])
def process_file():
    """
    Triggered by Pub/Sub when file uploaded to Cloud Storage
    """
    envelope = request.get_json()
    
    if not envelope:
        return jsonify({'error': 'No Pub/Sub message received'}), 400
    
    if 'message' not in envelope:
        return jsonify({'error': 'Invalid Pub/Sub message'}), 400
    
    # Decode Pub/Sub message
    payload = base64.b64decode(envelope['message']['data']).decode('utf-8')
    gcs_event = json.loads(payload)
    
    bucket_name = gcs_event['bucket']
    file_name = gcs_event['name']
    
    logging.info(f'Processing {bucket_name}/{file_name}')
    
    try:
        # Download file from Cloud Storage
        bucket = storage_client.bucket(bucket_name)
        blob = bucket.blob(file_name)
        file_contents = blob.download_as_bytes()
        
        # Process (e.g., image resize, JSON transformation)
        processed_data = transform_file(file_contents, file_name)
        
        # Upload result to output bucket
        output_bucket = storage_client.bucket('output-bucket')
        output_blob = output_bucket.blob(f'processed/{file_name}')
        output_blob.upload_from_string(
            processed_data,
            content_type=blob.content_type
        )
        
        logging.info(f'Successfully processed {file_name}')
        
        # Return 200 to acknowledge message (Pub/Sub acks)
        return jsonify({'status': 'processed'}), 200
    
    except Exception as e:
        logging.error(f'Error processing {file_name}: {str(e)}')
        # Return 500 so Pub/Sub retries the message
        return jsonify({'error': str(e)}), 500

def transform_file(data, filename):
    """
    Transform file (e.g., compress, convert format, extract metadata)
    """
    if filename.endswith('.json'):
        obj = json.loads(data)
        obj['processed_at'] = datetime.now().isoformat()
        return json.dumps(obj).encode()
    else:
        return data

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 8080)))
```

### Q10: Performance optimization - your Cloud Run service has 200ms p95 latency. How do you reduce it to 50ms?

**Answer:**

```
Root Cause Analysis:
    ↓
1. Cold starts? (100-500ms)
   → Set minInstances=1 to keep instance warm
   
2. Dependency latency? (database queries)
   → Add cloud SQL Auth proxy caching
   → Use connection pooling
   
3. Network latency to VPC? 
   → Use direct VPC integration (cloud-run-network)
   → Remove VPC connector
   
4. Serialization overhead?
   → Use Protocol Buffers instead of JSON
   → Compress response bodies
   
5. Single instance slower than expected?
   → Profile with Cloud Profiler
   → Optimize hot code paths
```

**Implementation:**

```python
# 1. Connection pooling
from google.cloud.sql.connector import Connector

connector = Connector()

def getconn():
    return connector.connect(
        "project:region:db-instance",
        "pymysql",
        user="user",
        password="password",
        db="database",
    )

pool = create_engine(
    "mysql+pymysql://",
    creator=getconn,
    pool_size=10,
    max_overflow=20,
    pool_recycle=3600,
)

# 2. Response caching with Cloud CDN
from flask import make_response

@app.route('/api/data/<id>', methods=['GET'])
def get_data(id):
    # Query (expensive)
    data = db_query(id)
    
    response = make_response(json.dumps(data))
    response.headers['Cache-Control'] = 'public, max-age=3600'  # 1 hour
    return response

# 3. Protocol Buffers serialization (faster than JSON)
import my_data_pb2

@app.route('/api/pb-data/<id>', methods=['GET'])
def get_data_pb(id):
    proto_msg = my_data_pb2.DataMessage()
    proto_msg.ParseFromString(db_query(id))
    return proto_msg.SerializeToString()

# 4. Async I/O for multiple operations
@app.route('/api/composite', methods=['GET'])
async def get_composite():
    # Parallel requests instead of sequential
    user, orders, profile = await asyncio.gather(
        fetch_user(),
        fetch_orders(),
        fetch_profile()
    )
    return {'user': user, 'orders': orders, 'profile': profile}
```

**Monitoring improvements:**
```bash
# Create dashboard showing p95 latency
gcloud monitoring dashboards create \
  --config-from-file dashboard.yaml

# Alert on p95 > 100ms
gcloud alpha monitoring policies create \
  --notification-channels CHANNEL_ID \
  --display-name "Cloud Run P95 Latency" \
  --condition "run.googleapis.com/request_latencies | p95 > 100ms"
```

---

## 9. Advanced Insights (Senior-level)

### Cost Optimization

**Scaling strategy for cost:**
```
High traffic (9am-5pm): 1000 max instances, 5 min instances
Medium traffic (6pm-9pm): 500 max instances, 2 min instances
Low traffic (9pm-9am): 100 max instances, 0 min instances
```

**Memory tuning:**
- 256MB: Simple APIs, I/O passthrough
- 512MB: Database queries, small data processing
- 1GB: Image processing, machine learning inference
- 2GB+: Large data transformation

### Performance Optimization

**Cold start reduction techniques:**
1. Optimize Dockerfile (multistage build)
2. Lazy load heavy dependencies
3. Keep container image <200MB
4. Use minInstances strategically

**Concurrency optimization:**
- Single instance handles 1000+ concurrent requests
- Use async I/O (not threading)
- Connection pooling to databases

### Security Hardening

**Defense in depth:**
- Run as non-root user (Dockerfile: `USER nobody`)
- Use distroless images (no shell, smaller attack surface)
- Implement request signing (request authentication)
- Use service accounts with minimal IAM permissions

### Advanced Patterns

**Fan-out pattern for parallel processing:**
```
Single request
    ↓
Cloud Run API
    ├─ Publish 100 tasks to Pub/Sub (each ~1KB)
    └─ Return task_id immediately
         ↓
    100 Cloud Run workers subscribe to topic
    ├─ Process in parallel (100 tasks x 10s = 100 seconds total vs 1000 seconds sequential)
    └─ Store results in Firestore
         ↓
    Client polls `/tasks/{task_id}` for completion
```

