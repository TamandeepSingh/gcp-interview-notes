# Cloud Pub/Sub

## 1. Core Concept (Deep Explanation)

Cloud Pub/Sub is Google Cloud's **fully managed, globally distributed message broker** that decouples producers from consumers. It provides:

- **Publish-Subscribe Model**: Producers publish messages without knowing consumers; consumers subscribe to topics
- **At-least-once Delivery**: Messages delivered at least once per subscription (exactly-once with ordering enabled)
- **Global Scale**: Automatically handles millions of messages/second across regions
- **Message Ordering**: Optional ordered delivery per partition key
- **Dead-letter Topics**: Automatic handling of failed messages

**Internal Architecture:**
- Topics store published messages in distributed storage
- Subscriptions are separate consumers of a topic
- Messages retained for 7 days by default (configurable)
- Pull subscriptions use streaming pull (efficient)
- Push subscriptions automatically forward to webhooks (HTTP endpoints)

## 2. Why This Exists

**Problems it solves:**
- **Decoupling**: Services don't need to know about each other
- **Asynchronous Processing**: Fire-and-forget message patterns
- **Scalability**: Handle temporary traffic spikes without losing messages
- **Reliability**: Retry failed messages automatically
- **Event-Driven Architecture**: Build reactive systems

## 3. When To Use

**Best scenarios:**
- Event-driven microservices (order placed → email sent → inventory updated)
- Asynchronous job processing (file upload → transcoding → storage)
- Real-time data streaming (IoT sensors, click streams to BigQuery)
- Cross-service communication (decouple API from workers)
- Webhook delivery (retry logic, ordering)

## 4. When NOT To Use

**Avoid if:**
- Need sub-second latency (use Cloud Tasks for better guarantees)
- Transactional guarantees needed (message atomicity across services)
- Simple request-response pattern (use HTTP APIs directly)
- Very small throughput (<1 msg/sec waste of resources)

## 5. Real-World Example: E-commerce Order Processing

```python
# Order Service (Producer) - receives order from API
from google.cloud import pubsub_v1
import json

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path('project-id', 'orders')

def create_order(order_data):
    """Publish order event"""
    message_json = json.dumps({
        'order_id': order_data['id'],
        'customer_id': order_data['customer_id'],
        'total': order_data['total'],
        'items': order_data['items'],
        'timestamp': datetime.now().isoformat()
    })
    
    future = publisher.publish(
        topic_path,
        message_json.encode('utf-8'),
        ordering_key=str(order_data['customer_id'])  # Ensures ordering per customer
    )
    
    message_id = future.result()
    print(f'Message published: {message_id}')
    return {'order_id': order_data['id']}

# Payment Service (Subscriber) - processes payment
from google.cloud import pubsub_v1
import base64

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path('project-id', 'payment-processing')

def callback(message):
    """Called when message received"""
    try:
        order = json.loads(message.data.decode('utf-8'))
        
        # Process payment
        stripe_charge = stripe.Charge.create(
            amount=int(order['total'] * 100),
            currency='usd',
            customer=order['customer_id'],
            idempotency_key=order['order_id']  # Prevent duplicate charges
        )
        
        # Publish payment success event
        publisher.publish(
            publisher.topic_path('project-id', 'payment-completed'),
            json.dumps({
                'order_id': order['order_id'],
                'stripe_charge_id': stripe_charge.id,
                'status': 'completed'
            }).encode('utf-8')
        )
        
        # Acknowledge message (remove from subscription)
        message.ack()
        print(f'Processed order {order["order_id"]}')
    
    except Exception as e:
        print(f'Error processing order: {e}')
        # Don't ack - message will be redelivered

# Start subscriber
streaming_pull_future = subscriber.subscribe(
    subscription_path,
    callback=callback,
    flow_control=pubsub_v1.types.FlowControl(
        max_messages=100,
        max_lease_duration=3600
    )
)

streaming_pull_future.result()
```

**Event Flow:**
```
Customer places order
    ↓
Order Service publishes to 'orders' topic
    ├─ Payment Service subscribes → processes payment
    ├─ Email Service subscribes → sends confirmation
    ├─ Inventory Service subscribes → updates stock
    └─ Analytics Service subscribes → records metric
         ↓
    All services process in parallel (async)
```

## 6. Architecture Thinking

**Pub/Sub in a Microservices System:**

```
┌─────────────────────────────────────────────────────────────┐
│         API Gateway / Frontend                              │
└──────┬──────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────┐
│  Order Service   │ (REST API)
│                  │
│ POST /orders     │
│ → Publish to     │
│   Pub/Sub        │
│ → Return 202     │
└────────┬─────────┘
         │
         ▼ Pub/Sub Topic: 'orders'
         │
    ┌────┴────────────────────────────────┐
    │                                     │
    ▼ Subscription: payment               ▼ Subscription: email
┌──────────────────┐              ┌──────────────────┐
│ Payment Service  │              │ Email Service    │
└──────────────────┘              └──────────────────┘
    │                                 │
    ▼                                 ▼ (publishes)
Deep Link Events (retry logic)     email-sent events
    │                                 │
    ▼                                 ▼
Cloud Dataflow (batching)          Cloud Logging
    ↓
BigQuery (analytics)
```

**Key Components:**

```
Publisher (Producer)
    ├─ Creates message
    ├─ Optional: ordering_key (ensures ordering)
    └─ Publishes to Topic

Topic
    ├─ Durability: Stores messages 7 days
    ├─ Partitioning: 4000 partitions (scales throughput)
    └─ Global: Replicated across regions

Subscription
    ├─ Consume mode: Push or Pull
    ├─ Flow control: Max messages/acks per second
    └─ Ack deadline: 10-600 seconds (time to process before redelivery)

Subscriber (Consumer)
    ├─ Receives messages via Pull or Push
    ├─ Process message
    └─ Acknowledge (ack) on success or Leave unacked to retry
```

## 7. Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Not acking messages | Messages redelivered indefinitely | Always ack on success, nack on failure |
| Ack before processing | Message lost on failure | Ack AFTER processing completes |
| No error handling | Crashing subscriber stops handling | Wrap in try-catch, handle errors gracefully |
| Not using ordering_key | Messages out of order | Set ordering_key for ordering requirement |
| Pushing to non-responsive endpoint | Failed deliveries lose messages | Implement robust push endpoint webhook |
| No dead-letter topic | Failed messages discarded after retries | Create DLT for manual investigation |
| Single subscription | All consumers race for messages | Create multiple subscriptions for broadcasting |
| No flow control | Out of memory (too many messages pulled) | Set max_messages in FlowControl |
| Ignoring ack_deadline | Messages redelivered while processing | Increase deadline or use modifyAckDeadline |
| No exponential backoff | Retry storms hammer service | Implement exponential backoff on subscriber |

## 8. Interview Questions (with Answers)

### Q1: Design a system where orders must be processed in FIFO order per customer but parallelized across customers.

**Answer:**
Use `ordering_key` set to `customer_id`:

```python
# Publisher
publisher.publish(
    topic_path,
    message_data,
    ordering_key=str(order['customer_id'])  # Groups messages per customer
)

# Result: Messages for customer 123 processed in order, but customer 456
# messages processed in parallel on different workers
```

**Architecture:**
```
order: {id: 1, customer: A}  
order: {id: 2, customer: B}  
order: {id: 3, customer: A}  
order: {id: 4, customer: C}  

With ordering_key=customer:
├─ Customer A: order 1 → order 3 (sequential)
├─ Customer B: order 2 (parallel)
└─ Customer C: order 4 (parallel)

Result: Within-customer FIFO + cross-customer parallelism
```

### Q2: Compare Pub/Sub with Cloud Tasks. When would you use each?

**Answer:**

| Dimension | Pub/Sub | Cloud Tasks |
|-----------|---------|------------|
| **Throughput** | Millions msg/sec | 500 tasks/sec |
| **Parallelism** | Parallel per partition | Sequential/rate-limited |
| **Use case** | Broadcast/events | Job queue, webhooks |
| **Fanout** | Multiple subscribers | Single target URL |
| **Retry** | Automatic (configurable) | Exponential backoff |
| **Ordering** | With ordering_key | With task routing |

**Decision:**
- **Pub/Sub**: "When something happens, notify ALL interested services" (email, analytics, inventory)
- **Cloud Tasks**: "Execute this specific task reliably at this time" (send email in 1 hour)

### Q3: Your subscriber is processing 1000 messages/second. Memory usage is 80%. How do you optimize?

**Answer:**
Reduce messages outstanding with flow control:

```python
# Before: Memory pressure
streaming_pull_future = subscriber.subscribe(
    subscription_path,
    callback=callback
)  # Default: pulls max messages aggressively

# After: Controlled backpressure
streaming_pull_future = subscriber.subscribe(
    subscription_path,
    callback=callback,
    flow_control=pubsub_v1.types.FlowControl(
        max_messages=10,      # Reduce outstanding messages
        max_lease_duration=60 # Seconds before redelivery
    )
)
```

**Memory reduction mechanism:**
- Before: 1000 messages in memory, processing 100/sec = 10s lag
- After: 10 messages in memory, processing 100/sec = 0.1s lag

**Alternative optimization:**
```python
# Batch processing (efficient for large datasets)
def callback(message):
    batch_queue.append(message)
    
    # Process batch every 100 messages or 5 seconds
    if len(batch_queue) >= 100 or time.time() - last_batch > 5:
        process_batch(batch_queue)
        for msg in batch_queue:
            msg.ack()
        batch_queue.clear()
```

### Q4: You have a Pub/Sub topic with 3 subscriptions. You publish 1000 messages. How many messages does each subscriber receive?

**Answer:**
Each subscription receives **1000 messages independently**. Subscriptions don't share messages.

```
Topic: 'orders' (publishers send 1000 messages)
    │
    ├─ Subscription: 'payment-processing'
    │  └─ Receives 1000 messages (independent copy)
    │
    ├─ Subscription: 'email-notifications'
    │  └─ Receives 1000 messages (independent copy)
    │
    └─ Subscription: 'analytics'
       └─ Receives 1000 messages (independent copy)

Total messages: 3000 (1000 per subscriber)
```

**Key difference from Kafka:**
- **Kafka topic partition**: All consumers of topic share messages (only one consumer per partition in consumer group)
- **Pub/Sub topic**: Each subscription gets full copy of all messages

### Q5: Implement dead-letter topic handling. What happens to messages that fail 5 times?

**Answer:**
```python
from google.cloud import pubsub_v1

# Create DLT as separate topic
admin_client = pubsub_v1.PublisherClient()
subscriber_client = pubsub_v1.SubscriberClient()

# Subscription with dead-letter policy
subscription_config = pubsub_v1.types.Subscription(
    name=f'projects/PROJECT_ID/subscriptions/main-sub',
    topic=f'projects/PROJECT_ID/topics/main-topic',
    dead_letter_policy=pubsub_v1.types.DeadLetterPolicy(
        dead_letter_topic=f'projects/PROJECT_ID/topics/dead-letter',
        max_delivery_attempts=5  # Move to DLT after 5 failed attempts
    ),
    ack_deadline_seconds=60
)

# Subscribe to dead-letter topic for manual inspection
def handle_dlq(message):
    """Process failed messages"""
    print(f'Dead-letter message: {message.data.decode()}')
    
    # Manually inspect and decide:
    # 1. Fix and republish to main topic
    # 2. Log to monitoring system
    # 3. Alert ops team
    
    message.ack()

response = subscriber_client.create_subscription(
    parent=f'projects/PROJECT_ID',
    subscription=subscription_config
)
```

### Q6: Your Cloud Run service gets 100 events/second but only processes 50/second. Design auto-scaling.

**Answer:**
**Problem:** Messages queue up in subscription. Need more consumer instances.

```bash
# 1. Create minimum number of Cloud Run instances
gcloud run deploy event-processor \
  --image gcr.io/project/processor \
  --min-instances 2 \
  --max-instances 100 \
  --region us-central1

# 2. Cloud Scheduler triggers Cloud Run to check Pub/Sub backlog
gcloud scheduler jobs create app-engine check-backlog \
  --schedule "* * * * *" \
  --timezone UTC \
  --http-method POST \
  --uri https://event-processor-...run.app/health
```

**Subscriber-side auto-scaling:**
```python
from google.cloud import pubsub_v1, monitoring_v3

def check_and_scale():
    """Monitor subscription backlog"""
    query_client = monitoring_v3.QueryServiceClient()
    
    # Query: oldest unacked message age
    results = query_client.query_time_series(
        name='projects/PROJECT_ID',
        query='''
        fetch pubsub_subscription
        | metric 'pubsub.googleapis.com/subscription/oldest_unacked_message_age'
        | align mean(1m)
        '''
    )
    
    for result in results.time_series_list:
        age_seconds = result.points[0].values[0].double_value
        
        if age_seconds > 60:  # Messages queuing > 1 minute
            scale_workers(up=True)
        elif age_seconds < 5:  # Processing ahead
            scale_workers(up=False)
```

### Q7: Deploy Pub/Sub subscriber as GKE deployment. How do you handle graceful shutdown?

**Answer:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-subscriber
spec:
  replicas: 5
  selector:
    matchLabels:
      app: event-subscriber
  template:
    metadata:
      labels:
        app: event-subscriber
    spec:
      serviceAccountName: event-subscriber-sa
      
      # Graceful shutdown configuration
      terminationGracePeriodSeconds: 120  # Give 120s to finish processing
      
      containers:
      - name: subscriber
        image: gcr.io/project/subscriber:latest
        
        # Health check
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        
        # Readiness check (stop accepting new messages)
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        
        # Preemption signal handler
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15 && kill -TERM 1"]  # Give time for graceful shutdown

---
# Pod disruption budget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: event-subscriber-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: event-subscriber
```

**Subscriber implementation with graceful shutdown:**
```python
import signal
import sys
from google.cloud import pubsub_v1

streaming_pull_future = None
should_exit = False

def signal_handler(sig, frame):
    """Handle SIGTERM for graceful shutdown"""
    global should_exit, streaming_pull_future
    print('Received SIGTERM, shutting down gracefully...')
    should_exit = True
    
    if streaming_pull_future:
        # Stop pulling new messages
        streaming_pull_future.cancel()
        
        # Wait for existing messages to finish
        try:
            streaming_pull_future.result(timeout=120)
        except Exception as e:
            print(f'Error during shutdown: {e}')

signal.signal(signal.SIGTERM, signal_handler)

def callback(message):
    """Process message"""
    try:
        process_message(message.data.decode())
        message.ack()
    except Exception as e:
        print(f'Error: {e}')
        # Don't ack - redelivery enables recovery

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path('project', 'subscription')

streaming_pull_future = subscriber.subscribe(
    subscription_path,
    callback=callback,
    flow_control=pubsub_v1.types.FlowControl(
        max_messages=100,
        max_lease_duration=600
    )
)

try:
    streaming_pull_future.result()
except KeyboardInterrupt:
    pass
```

### Q8: You need exactly-once processing semantics. How do you implement?

**Answer:**
Exactly-once is hard in distributed systems. Pub/Sub guarantees **at-least-once**. Implement idempotency:

```python
import hashlib
from google.cloud import datastore

# Store processed message IDs in datastore
datastore_client = datastore.Client()

def process_message_idempotent(message_data):
    """Process with idempotency check"""
    
    # Generate unique message ID
    message_id = hashlib.sha256(
        (message_data + datetime.now().isoformat()).encode()
    ).hexdigest()
    
    # Check if already processed
    key = datastore_client.key('ProcessedMessages', message_id)
    existing = datastore_client.get(key)
    
    if existing:
        print(f'Message {message_id} already processed, skipping')
        return False
    
    # Process message
    result = do_business_logic(message_data)
    
    # Store as processed (atomic with result)
    entity = datastore.Entity(key=key)
    entity['result'] = result
    entity['processed_at'] = datetime.now()
    
    datastore_client.put(entity)
    return True
```

**Better approach: Use message attributes unique per producer:**
```python
# Publisher includes unique ID
publisher.publish(
    topic_path,
    message_data,
    message_id=str(order_id),  # Unique per order
    timestamp=datetime.now().isoformat()
)

# Subscriber deduplicates
seen_ids = {}

def callback(message):
    msg_id = message.attributes.get('message_id')
    
    if msg_id in seen_ids:
        print(f'Duplicate message {msg_id}, skipping')
        message.ack()
        return
    
    process(message)
    seen_ids[msg_id] = True
    message.ack()
```

### Q9: Explain Pub/Sub message delivery guarantee. Why not exactly-once?

**Answer:**
Delivery guarantees:

1. **Best effort** (default): Messages may be lost if broker crashes
2. **At-least-once** (Pub/Sub): Every message delivered at least once
   - Message stored durably on Pub/Sub
   - Subscription tracks acks per message
   - Unacked messages redelivered after deadline
3. **Exactly-once**: Impossible to guarantee in distributed systems (no network is reliable)

**Why not exactly-once?**
```
Network partition between subscriber and broker:
├─ Subscriber processes message
├─ Subscriber sends ack
├─ Network breaks before ack reaches broker
├─ Broker resends message
└─ Duplicate processing occurs
```

**Solution: Idempotent processing** (business logic handles duplicates)

### Q10: Design high-availability Pub/Sub setup for critical payment processing.

**Answer:**
```yaml
# Multi-region topic replication
Topics:
  payment-events-primary:
    region: us-central1
  payment-events-replica:
    region: us-east1

# Primary subscription
Subscription:
  payment-processing:
    topic: payment-events-primary
    dead_letter_topic: payment-dlq
    max_delivery_attempts: 10
    ack_deadline: 600s  # 10 minutes for payment processing

# Replica subscription for failover
Subscription:
  payment-processing-failover:
    topic: payment-events-replica
    enabled: false
    (activated if primary region fails)

# Duplicate detection
Datastore:
  ProcessedPayments:
    - payment_id: "123"
    - stripe_charge_id: "ch_4321"
    - processed_at: timestamp
    (idempotency key)

# Monitoring and alerting
Alerts:
  - Oldest unacked message > 5 minutes
  - Dead letters > 0
  - Subscription lag increasing
  - Pub/Sub quota exceeded
```

## 9. Advanced Insights (Senior-level)

### High-Throughput Optimization

**Batching for efficiency:**
- Instead of processing 1 message at a time
- Batch 1000 messages, process together
- Reduce database round-trips by 1000x

### Cost Optimization

**For high volume (millions of messages/month):**
- Committed use discounts available
- Estimate costs: $0.40 per GB ingested

### Monitoring Strategy

**Key metrics:**
- Push latency (time from publish to subscriber delivery)
- Ack latency (time to process and acknowledge)
- Oldest unacked message age (subscription lag)
- Dead-letter rate

