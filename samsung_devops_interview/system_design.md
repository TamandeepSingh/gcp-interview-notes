# System Design & Architecture Scenarios - Samsung DevOps Interview

## Scenario 1: Design Log Aggregation Pipeline for 200 Microservices

### The Problem
Samsung has:
- 200+ microservices across 5 regions
- 50 million log events PER DAY
- Need to search logs within 10 seconds
- Retain logs for 30 days
- Must extract PII before storage
- Compliance requirement: audit trail of all data access

**Design a complete log aggregation pipeline. Consider cost, performance, and compliance.**

### Solution Architecture (DETAILED)

#### Component 1: Log Collection (Edge)
```
Trade-off Analysis:

Option A: Fluentd (lightweight, flexible)
Option B: Fluent Bit (smaller, faster)
Option C: Logstash (heavy, powerful)
Option D: Vector (new, efficient)

CHOSEN: Fluent Bit
Why: 
- 10MB vs 100MB (Fluentd) memory footprint
- Comparable processing speed
- Lower resource overhead on microservices
- Good GCP/cloud integration

Architecture:
├─ Fluent Bit Agent (on each pod) - 5MB per pod
│  ├─ Parse application logs
│  ├─ Add metadata (service, pod, node)
│  ├─ Mask PII (regex patterns)
│  └─ Buffer locally (2GB)
│
├─ Kafka broker (local region)
│  ├─ Topic: raw-logs
│  ├─ Retention: 3 hours (replay window)
│  ├─ Partitions: 20 (parallel processing)
│  └─ Replication: 3
│
└─ Fluent Forward protocol (to central logging)
   └─ TLS encryption in transit
```

#### Component 2: PII Masking (Processing)
```
Location: Between Kafka and Storage

Two approaches:

Option A: Fluent Processor (lightweight)
- Pro: Simple, in-pipeline
- Con: Limited regex accuracy
- Risk: false negatives

Option B: Dedicated PII Service (safer)
- Pro: Accurate, audit trail, testable
- Con: Extra service, latency hit
- Risk: bottleneck

CHOSEN: Hybrid approach
- Cheap PII masking in Fluent Bit (obvious patterns)
- Deeper scanning in central service (probabilistic)
- Audit log: before/after comparison

PII Patterns to mask:
- Credit cards: \d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}
- Social security: \d{3}-\d{2}-\d{4}
- Email: (?i)(?<=[^\w])[\w.-]+@[\w.-]+\.\w+
- API keys: (?i)(api[_-]?key|secret|token)["\s]*[:=]["\s]*[\w\-]+

Implementation:
<filter **>
  @type record_transformer
  <record>
    message ${record["message"].gsub(/\d{3}-\d{2}-\d{4}/, "***-**-****")}
    message ${record["message"].gsub(/\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}/, "****-****-****-****")}
  </record>
</filter>
```

#### Component 3: Aggregation & Transformation
```
Kafka Streams vs Spark Streaming vs Flink

Trade-offs:
- Kafka Streams: Simple, state management, 500ms latency
- Spark: Heavy, excellent analytics, 1-2s latency
- Flink: Complex setup, real-time, 100ms latency

CHOSEN: Kafka Streams
Why: Perfect for this workload
- Stateless transformations
- Enrichment (add node topology, datacenter)
- No complex state management
- Low operational overhead

Architecture:
Kafka raw-logs
  ↓
Kafka Streams Application
  ├─ Extract timestamp
  ├─ Enrich with metadata (service tier, environment)
  ├─ Deduplicate (by requestId within 1 minute window)
  └─ Route to appropriate topic
    ├─ logs-persistent (for everything)
    ├─ logs-errors (only severity >= ERROR)
    ├─ logs-critical (only critical alerts)
    └─ logs-debug (short retention, 7 days)

Code Example:
```java
public class LogAggregation {
    public static void main(String[] args) {
        StreamsBuilder builder = new StreamsBuilder();
        
        builder.stream("raw-logs")
            .map((k, v) -> {
                JsonNode log = objectMapper.readTree(v);
                return new KeyValue<>(
                    log.get("requestId").asText(),
                    log
                );
            })
            .selectKey((k, v) -> v.get("service").asText())
            .map((service, log) -> {
                // Enrich with metadata
                ((ObjectNode) log).put("region", "us-central");
                ((ObjectNode) log).put("ingested_at", System.currentTimeMillis());
                return new KeyValue<>(service, log.toString());
            })
            .to("logs-persistent");
        
        // Also route errors to separate topic
        builder.stream("raw-logs")
            .filter((k, v) -> {
                String level = v.get("level").asText();
                return level.equals("ERROR") || level.equals("CRITICAL");
            })
            .to("logs-errors");
        
        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
    }
}
```

#### Component 4: Storage Layer
```
Decision Tree:

For Real-time Search (50M logs/day):
ElasticSearch vs OpenSearch vs BigQuery vs ClickHouse?

Trade-offs:
┌─ ElasticSearch
├─ Pro: Best UX, mature, great for metrics
├─ Con: Expensive ($$$), needs tuning
└─ Scale: ~100GB/day storage

┌─ OpenSearch
├─ Pro: Cheaper ElasticSearch fork
├─ Con: Less mature than ES
└─ Scale: ~100GB/day

┌─ BigQuery
├─ Pro: Unlimited scale, SQL queries
├─ Con: 30-50s query latency, expensive
└─ Best for: analytics, compliance

┌─ ClickHouse
├─ Pro: Cheap, incredibly fast analytics
├─ Con: Complex ops, different paradigm
└─ Best for: metrics, TimeSeries

CHOSEN: Heterogeneous approach
- Real-time (24h): OpenSearch (5 node cluster)
- Archive (30d): Google Cloud Storage
- Analytics (compliance): BigQuery
- Metrics: Prometheus + M3DB

Cost Breakdown:
- 50M logs/day × 1KB avg = 50GB/day = 1.5TB/month
- OpenSearch 5-node cluster: $5K/month
- Storage (cold): $100/month (GCS)
- BigQuery sync: $500/month
- Total: ~$6K/month
```

#### Component 5: Query & Compliance
```
Requirements:
1. Real-time search (< 10 seconds)
2. 30-day retention
3. Audit trail of accesses
4. PII access restrictions
5. Data retention compliance

Design:
┌─ OpenSearch (Hot - 24h)
│  └─ Full-text search, aggregations
├─ Cloud Storage (Warm - 7d)
│  └─ Parquet files, compressed
├─ Cloud Storage (Cold - 30d)
│  └─ Archived, infrequent access tier
└─ BigQuery (Archive - 90d)
   └─ Query compliance data

Access Control:
- ServiceAccount: only read logs from their service
- PII Supervisor: can read masked PII
- Auditor: read-only to compliance queries
- DevOps: full access to debug

RBAC Implementation:
OpenSearch Security Plugin:
{
  "roles": {
    "service_reader": {
      "index_permissions": [{
        "index_patterns": ["logs-service-*"],
        "allowed_actions": ["read"]
      }],
      "tenant_permissions": [{
        "tenant_patterns": ["service_*"],
        "allowed_actions": ["kibana_all_read"]
      }]
    }
  }
}

Audit Trail:
┌─ Every log access logged to separate topic
├─ Kafka → Audit Service → Immutable DB
├─ Immutable (Append-only rules):
│  - Who accessed what
│  - When
│  - How many records
│  - From which source IP
└─ 7-year retention for compliance
```

#### Component 6: Scalability Plan
```
Capacity Planning for 50M → 200M logs/day (4x growth):

Current (50M/day):
- Fluent Bit: 200 pods × 50MB = 10GB memory
- Kafka: 3 brokers, 100GB disk each
- OpenSearch: 5 nodes, 100GB each
- Cost: $6K/month

At 4x growth (200M/day):
- Fluent Bit: Same (stateless)
- Kafka: Add 2 more brokers, partition expansion
- OpenSearch: 8 nodes (double), 200GB each
- Cost: $15K/month

Bottleneck Timeline:
- Week 10: ElasticSearch needs more nodes (add when CPU > 70%)
- Week 15: Kafka needs more brokers (add when disk > 80%)
- Week 20: Need second region deployment

Auto-scaling Strategy:
- Fluent Bit: Already scaled with pods
- Kafka: Manual review, alert at 60% disk
- OpenSearch: KubeDB operator for auto-scaling
```

### Follow-up Questions

**Q1: How would you handle log loss scenarios (broker down)?**
- Expected: Discuss Kafka replication, durability settings, recovery guarantees

**Q2: What if compliance requires 10-year retention?**
- Expected: Tiered storage strategy, archive to glacier, data format compatibility

**Q3: How would you implement regional isolation for sensitive logs?**
- Expected: Separate topics per region, encryption keys per region, cross-region replication rules

### Red Flags

❌ "Just use ELK stack, it handles everything"
- Doesn't address PII masking, compliance, cost optimization

❌ "Stream everything to ElasticSearch real-time"
- Wasteful, expensive, not needed for all logs

❌ "We don't need separate storage tiers"
- Misses cost optimization opportunities

### Pro Tips

✅ "I'd implement aggressive sampling for non-critical logs to reduce cost"

✅ "I'd use feature flags to enable detailed logging only when debugging"

✅ "I'd implement log correlation with trace IDs for distributed tracing"

✅ "I'd set up automated alerts for unusual log volume spikes (indicates failures)"

---

## Scenario 2: Design Zero-Downtime Database Migration (MySQL → PostgreSQL)

### The Problem
You need to migrate:
- 500GB MySQL database
- 100,000 queries/sec read traffic
- 10,000 queries/sec write traffic
- Zero downtime required
- Application code uses MySQL features (specific syntax)
- 3-day migration window before cost doubles

**Design a migration strategy. Consider data consistency, performance, rollback.**

### Solution Architecture (DETAILED)

#### Phase 1: Planning & Validation (Day 0)

```
Preparation:
1. Create MySQL read replica in new region (PostgreSQL region)
2. Set up PostgreSQL standby (same spec as target)
3. Database schema compatibility audit
4. Application code review for MySQL-isms
5. Performance baseline testing

MySQL-specific features to handle:
- AUTO_INCREMENT vs SERIAL/SEQUENCES in PostgreSQL
- TINYINT/SMALLINT vs INTEGER
- VARCHAR(255) vs VARCHAR with constraints
- JSON support (MySQL JSON vs PostgreSQL jsonb)
- FULLTEXT search (migrate to PostGIS)
- Triggers (PostgreSQL has different syntax)

Tool: aws dms (Database Migration Service)
- Schema conversion
- Continuous replication
- Validation and reconciliation
```

#### Phase 2: Parallel Running (Day 1-2)

```
Setup:
PostgreSQL ← CDC (Change Data Capture) ← MySQL
   ↑
   └─── Validation layer matches reads

Architecture:

MySQL Production
  ├─ Write goes to MySQL directly
  ├─ CDC captures changes (binlog)
  │  └─ AWS DMS reads binlog
  │     └─ Replicates to PostgreSQL in real-time
  ├─ Read goes to MySQL (or PostgreSQL with lag)
  └─ Validation service compares queries
     ├─ Executes sample queries on both
     ├─ Compares results
     └─ Alerts if divergence

Key Metrics:
- Replication lag (should be < 100ms)
- Validation drift (should be 0 rows)
- Write consistency checks

Timeline:
T+0h: DMS CDC starts
T+4h: Initial full sync complete
T+8h: Continuous replication started
T+20h: Monitoring, gap detection
T+40h: Ready for cutover
```

#### Phase 3: Pre-cutover Validation

```
Checklist:

1. Schema Verification
   $ SELECT COUNT(*) FROM mysql_db.table1;
   $ SELECT COUNT(*) FROM postgres_db.table1;
   # Must match exactly

2. Data Integrity Checks
   $ SELECT MD5(GROUP_CONCAT(DISTINCT id ORDER BY id)) 
     FROM mysql_db.large_table;
   # Compare against PostgreSQL equivalent

3. Sequence Validation
   $ SELECT MAX(id) FROM mysql_db.auto_inc_table;
   # Sync PostgreSQL sequence to this value + 1

4. Constraint Validation
   $ SELECT CONSTRAINT_NAME 
     FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS
     WHERE TABLE_NAME = 'users';
   # Verify all constraints exist in PostgreSQL

5. Index Validation
   $ EXPLAIN SELECT * FROM mysql_db.users WHERE id = 100;
   $ EXPLAIN SELECT * FROM postgres_db.users WHERE id = 100;
   # Compare query plans

6. Performance Baseline
   $ ab -n 10000 -c 100 http://app:8080/api/users
   # Run against both MySQL and PostgreSQL
   # p95 latency should be within 10%

7. Replication Lag Check
   $ aws dms describe-replication-tasks \
     --filters Name=replication-task-arn,Values=arn:aws:dms:* \
     --query 'ReplicationTasks[0].{CDCLatencySource,CDCLatencyTarget}'
   # Should be < 100ms
```

#### Phase 4: Cutover Strategy

```
Option A: Scheduled Maintenance Window (RISKY)
- Stop all writes to MySQL at 2 AM
- Wait for CDC to catch up (0ms lag)
- Point application to PostgreSQL
- Verify all queries work
- Con: Users experience 30min downtime

Option B: Blue-Green (BETTER)
- Create new app cluster
- Point to PostgreSQL
- Gradually shift traffic (10% → 25% → 50% → 100%)
- Monitor error rates at each step
- Rollback if issues detected

Option C: Feature Flag + Gradual Migration (BEST)
- Deploy feature flag: USE_POSTGRES=false
- Set flag to "random-10-percent" for 10% of requests
- Monitor for 1 hour
- Gradually increase: 10% → 25% → 50% → 100%
- Can rollback in seconds

CHOSEN: Option C (Feature Flag approach)

Implementation:

// In application code
Config config = getConfig();
boolean usePostgres = config.requestAffinity("use_postgres", 0.0);

Connection conn;
if (usePostgres) {
    conn = postgresPool.getConnection();
} else {
    conn = mysqlPool.getConnection();
}

// Canary deployment:
Time        Postgres%   Strategy
T+0         0%         MySQL only (baseline)
T+0:30      5%         Random 5%
T+1:00      10%         Random 10%
T+2:00      25%         Random 25%
T+4:00      50%         Random 50%
T+6:00      75%         Random 75%
T+8:00      100%        Full PostgreSQL (MySQL remains as backup)
```

#### Phase 5: Monitoring During Cutover

```
Key Metrics to Watch:

1. Error Rate
   - Errors/sec on both databases
   - Alert if > 2x increase
   - Query-level error categorization

2. Latency
   - p50, p95, p99 latency
   - Should stay within ±10%
   - Per-endpoint latency breakdown

3. Data Consistency
   - Random row verification every 100k queries
   - Compare query results between DB
   - MD5 checksums of aggregates

4. Connection Count
   - Connection pool utilization
   - Max connections not falling below 20%

5. Replication Lag
   - CDC lag (should be ~0ms)
   - Write throughput

Architecture:
MySQL (8000 conn)
 ↓ [Feature Flag 10%]
PostgreSQL (check)
 ↓
Metrics Aggregator
 ↓
Prometheus + Grafana
```

#### Phase 6: Validation After Cutover

```
Post-Migration Checks:

Day 1:
- Monitor error logs (every 5 min)
- Check query performance (every 10 min)
- Validate no data anomalies
- Check application response times
- Database connection pool health

Day 2:
- Confirm MySQL replication is complete
- Keep MySQL as read-only replica for 24h
- Verify no missed writes
- Check for slow queries building up

Day 3:
- Prepare MySQL decommission (do NOT do this yet)
- Archive MySQL backups
- Cleanup old application code using MySQL features
- Final audit

Week 1:
- Monitor for any intermittent issues
- Performance tuning (if needed)
- Create runbook for unexpected issues

Month 1:
- Can safely decommission MySQL (likely still running)
- Final cost/performance report

Rollback Window:
Keep MySQL online and in sync for 2 weeks
- If critical bug found, can fail-back immediately
- After 2 weeks, decommission MySQL
```

#### Phase 7: Post-Migration Optimizations

```
PostgreSQL-specific tuning:

1. Connection pooling
   # Switch from app-level to pgBouncer
   # Better connection management

2. Index optimization
   # PostgreSQL handles indexes differently
   # May need to rebuild indexes for performance

3. Query optimization
   # PostgreSQL query planner may differ
   # Use EXPLAIN to review slow queries

4. Full-text search
   # If using MySQL FULLTEXT:
   CREATE INDEX ft_idx ON documents 
     USING GIN(to_tsvector('english', body));

5. Partitioning
   # PostgreSQL supports better partitioning
   # Consider range/hash partitioning for large tables

6. Replication
   # If replication needed, use PostgreSQL replication
   # Built-in, not like MySQL master-master

Configuration Tuning:
shared_buffers = max(25% RAM, 64MB) = 128GB
effective_cache_size = 75% RAM = 384GB
work_mem = (total RAM * 3 / 4) / max_connections
maintenance_work_mem = total_RAM * 0.05
random_page_cost = 1.1 (SSD) vs 4.0 (HDD)
```

### Follow-up Questions

**Q1: How would you handle application code changes needed for PostgreSQL?**
- Expected: Feature flags, database abstraction layer, compatibility testing

**Q2: What if replication lag spikes during cutover?**
- Expected: Pause traffic shift, investigate bottleneck, resume when healthy

**Q3: How would you handle rollback if PostgreSQL has serious issues?**
- Expected: Keep both DBs in sync, quick flip back to MySQL, RTO < 15min

### Red Flags

❌ "Just run mysqldump and restore to PostgreSQL"
- Doesn't handle zero-downtime requirement

❌ "We don't need validation, replication will handle it"
- Schema differences could cause silent data loss

❌ "Feature flags are too complex, let's just do maintenance"
- Shows risk aversion over engineering

### Pro Tips

✅ "I'd use AWS DMS or Debezium for CDC, not custom scripts"

✅ "I'd implement feature flags for gradual traffic shift"

✅ "I'd keep MySQL running for 2 weeks as safety net"

✅ "I'd monitor every metric and have automated rollback"

---

## Scenario 3: Design Disaster Recovery for Multi-Region Setup

### The Problem
Design DR for Samsung's production:
- Data at rest in us-east1 (primary), us-west1 (secondary)
- RTO (Recovery Time Objective): < 30 minutes
- RPO (Recovery Point Objective): < 5 minutes
- Failover must be automatic on primary region failure
- Cannot afford downtime during disaster test

**Design complete DR strategy. Consider budget, data sync, network, automation.**

### Solution Architecture (DETAILED)

#### Layer 1: Data Replication

```
Tier 1: Synchronous Data Replication
├─ Database: PostgreSQL multi-region replication
│  ├─ Primary (us-east): streaming replication
│  ├─ Standby (us-west): receives replication stream
│  ├─ Lag: ~50ms (must stay < 100ms)
│  └─ Failover: Manual (prevents split-brain)
│
├─ Storage: Cloud Storage GCS
│  ├─ Dual-region bucket (us and eu)
│  ├─ Cross-region replication: < 15 minutes
│  ├─ Versioning: enabled
│  └─ Backup: daily snapshots to separate project
│
└─ Cache: Redis
   ├─ Primary: ElastiCache in us-east
   ├─ Read replica: us-west
   └─ Failover: DNS switch (30 second DNS TTL)

Tier 2: Application Data Consistency
├─ Event sourcing for state changes
│  ├─ All writes to Kafka → database
│  ├─ Kafka multi-region cluster
│  └─ Secondary region subscribes to topic
│
├─ Eventual consistency guarantees
│  ├─ Max 5 minutes lag between regions
│  ├─ Compensating transactions if needed
│  └─ Conflict resolution strategy
│
└─ Retry logic
   ├─ Failed writes queued locally
   ├─ Retried with exponential backoff
   └─ Dashboard for stuck transactions
```

#### Layer 2: Traffic Failover

```
Scenario 1: us-east primary fails

Current State:
Users → Route53 → ALB (us-east) → Apps → Database (us-east)
                              ↓ replication
                         Standby (us-west)

Failover Mechanism:
1. Health Check Detects Failure (< 30 seconds)
   - Route53 health check pings ALB every 10 seconds
   - After 3 consecutive failures (30s), mark unhealthy

2. Automatic Failover Trigger
   - Route53 automatic failover switches to us-west
   - New traffic: Users → Route53 → ALB (us-west)

3. Database Failover
   - Automated: PostgreSQL monitoring detects primary down
   - Promotes standby to primary (< 1 minute)
   - OR manual: for critical operations

4. Application Failover
   - Apps already running in us-west
   - Just need traffic rerouted (happens via DNS)
   - DNS TTL: 30 seconds (old connections timeout)

Timeline:
T+0s: Primary region failure
T+0-30s: Route53 detects (health check interval)
T+30-60s: DNS propagates to end users
T+60-90s: Database failover completes
T+90-120s: Applications resume (old connections close)
Total RTO: ~2 minutes (within 30min SLA)
```

#### Layer 3: Automation & Orchestration

```
Disaster Recovery Orchestration:

Component: Failover Orchestrator (Lambda + EventBridge)

{
  "source": "aws.route53",
  "detail-type": "HealthCheckStatusChange",
  "detail": {
    "NewState": "ALARM"
  }
}
↓
Lambda: check-dr-prerequisites
├─ Is secondary region healthy?
├─ Is database replication lag < 1 min?
├─ Are applications ready?
└─ Proceed: YES → next step, NO → alert human

→ Lambda: execute-database-failover
├─ Promote PostgreSQL standby
├─ Update DNS to point to new primary
├─ Verify connections working
└─ Alert operations team

→ Lambda: verify-application-health
├─ Hit health-check endpoints
├─ Verify error rates normal
├─ Check database connections
└─ Send success notifications

Full workflow in CloudFormation stack:
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Resources": {
    "FailoverLambda": {
      "Type": "AWS::Lambda::Function",
      "Properties": {
        "Code": {
          "S3Bucket": "lambda-code",
          "S3Key": "failover.zip"
        },
        "Handler": "index.handler",
        "Role": "arn:aws:iam::123:role/lambda-dr-role",
        "Environment": {
          "Variables": {
            "PRIMARY_DB": "prod-db-primary.us-east-1.rds.amazonaws.com",
            "SECONDARY_DB": "prod-db-standby.us-west-1.rds.amazonaws.com",
            "ROUTE53_ZONE_ID": "Z1234567890ABC"
          }
        }
      }
    },
    "HealthCheckAlarm": {
      "Type": "AWS::CloudWatch::Alarm",
      "Properties": {
        "AlarmActions": [{
          "Arn": "arn:aws:lambda:us-east-1:123:function:FailoverLambda"
        }]
      }
    }
  }
}
```

#### Layer 4: Network Resilience

```
Cross-region networking:

Primary (us-east)                  Secondary (us-west)
VPC: 10.0.0.0/16                  VPC: 10.1.0.0/16
├─ ALB: 10.0.1.0/24              ├─ ALB: 10.1.1.0/24
├─ Apps: 10.0.2.0/24             ├─ Apps: 10.1.2.0/24
└─ RDS: 10.0.3.0/24              └─ RDS: 10.1.3.0/24

VPC Peering:
us-east ↔ us-west (always on)
├─ Route table: 10.1.0.0/16 via pcx-xxxxx
├─ Bandwidth: 100 Gbps (regional peering)
└─ Latency: ~20ms

Cross-region backup network:
Primary → S3 → Secondary
├─ S3 replication: 15 min lag
├─ NLB backup route: < 50ms
└─ Fallback: connect via public IPs (encrypted)

Network Failover:
┌─ Primary failure
├─ Route53 health check fails
├─ DNS switches to us-west ALB
├─ Secondary apps already running
└─ Traffic starts flowing to secondary

Critical: Make sure apps in secondary are:
- Already running (not cold-start)
- Pre-warmed with caches
- Connected to secondary database
```

#### Layer 5: Testing & Validation

```
Disaster Recovery Drills (quarterly):

Type 1: Failover Simulation (No-Impact Test)
- Run in staging environment
- Simulate primary region failure
- Automated failover executed
- Verify all components come up
- Measure actual failover time
- Check for data loss

Type 2: Chaos Engineering (Controlled)
- Randomly kill resources in primary
- Observe monitoring and alerting
- Test detection and response
- Measure MTTR (Mean Time To Recovery)

Type 3: Full Regional Failover (Planned Maintenance)
- Announced in advance
- Temporary downtime expected
- Never during peak hours
- Full validation of secondary region
- Example: switch to us-west for 1 hour
- Measure performance differences

Type 4: Backup Restore Test (Monthly)
- Take backup from 30 days ago
- Restore to separate environment
- Query data to verify integrity
- Verify no corruption

Automation:
```bash
#!/bin/bash
# dr-drill.sh - Run disaster recovery drill

set -e

ENVIRONMENT="staging"
DRILL_ID="dr-$(date +%Y%m%d-%H%M%S)"

echo "Starting DR drill: $DRILL_ID"

# 1. Pre-flight checks
./scripts/preflight-checks.sh

# 2. Stop primary region
./scripts/stop-primary-region.sh "$ENVIRONMENT"

# 3. Wait for health checks to detect failure
echo "Waiting for detection..."
sleep 40

# 4. Verify automatic failover
./scripts/verify-failover.sh "$ENVIRONMENT"

# 5. Run smoke tests
./scripts/smoke-tests.sh "secondary"

# 6. Measure failover time
FAILOVER_TIME=$(date +%s%N)
echo "Failover completed in: ${FAILOVER_TIME}ms"

# 7. Report results
./scripts/generate-dr-report.sh "$DRILL_ID"

# 8. Restore primary
./scripts/restore-primary.sh "$ENVIRONMENT"

echo "DR drill completed: $DRILL_ID"
```

Drill Frequency:
- Failover simulation: Monthly
- Chaos engineering: Weekly
- Full regional switch: Quarterly
- Backup restore: Monthly
```

#### Layer 6: Monitoring & Alerting

```
Replication Lag Monitoring:
Alert if:
- Database replication lag > 5 minutes (soft alert)
- Database replication lag > 10 minutes (hard alert + page)
- Replication broken (lag increasing indefinitely)

S3 Replication Status:
Alert if:
- Replication failures detected
- Replication latency > 30 minutes
- Failed object count > 100

Application Health:
Alert if:
- Error rate in secondary > 1%
- Latency in secondary > 1.5x primary
- Database connection failures

Network Health:
Alert if:
- VPC peering connection down
- Cross-region latency > 100ms
- Packet loss > 0.1%

Prometheus Metrics:
```python
# In application
replication_lag_seconds = Gauge('replication_lag_seconds')
failover_time_seconds = Gauge('failover_time_seconds')
data_inconsistency_rows = Gauge('data_inconsistency_rows')

# Scrape every 10 seconds
@app.route('/metrics')
def metrics():
    # Query database replication lag
    lag = get_replication_lag()
    replication_lag_seconds.set(lag)
    return render_prometheus_metrics()
```

Dashboard:
┌─ Primary Region Status
│  ├─ Availability
│  ├─ Error Rate
│  └─ Latency
├─ Secondary Region Status
│  ├─ Availability
│  ├─ Error Rate
│  └─ Latency
├─ Data Replication
│  ├─ Lag (ms)
│  ├─ Throughput (MB/s)
│  └─ Errors
└─ Last Successful Drill
   ├─ Date
   ├─ Failover time
   └─ Results
```

#### Layer 7: Cost Optimization

```
DR Infrastructure Cost:

Option A: Full Active-Active (Expensive)
- Both regions fully staffed
- $100K/month infrastructure
- RPO: 0 (real-time)
- RTO: 0 (instant)
- Best for: critical financial systems

Option B: Active-Passive (Recommended)
- Primary: fully scaled (80% capacity)
- Secondary: minimal (20% capacity, auto-scale up)
- $60K/month infrastructure
- RPO: 5 minutes
- RTO: 10-15 minutes
- Best for: most applications

Option C: Backup-Only (Budget)
- Primary: fully scaled
- Secondary: backups only, restore if needed
- $30K/month infrastructure
- RPO: 1 hour (backup interval)
- RTO: 1-2 hours (restore time)
- Best for: non-critical systems

CHOSEN: Option B (Active-Passive)

Cost Breakdown:
Primary region:
- EKS cluster: $20K (80 nodes)
- RDS Multi-AZ: $15K
- Data synch: $5K
- Total: $40K

Secondary region:
- EKS cluster: $10K (40 nodes) → auto-scale to 80
- RDS read replica: $10K
- Data synch: $3K
- Total: $23K

Shared costs:
- Data transfer: $2K
- Monitoring: $2K
- Total: $67K/month
```

### Follow-up Questions

**Q1: How would you test failover without impacting production?**
- Expected: Staging environment, chaos engineering, scheduled drills

**Q2: What if data corruption is detected after failover?**
- Expected: Rollback to previous snapshot, point-in-time recovery, data reconciliation

**Q3: How do you handle database consistency during network partition?**
- Expected: Fencing, split-brain prevention, quorum-based decisions

### Red Flags

❌ "We have backups, so we don't need active-passive replication"
- Backups are for recovery, not for uptime

❌ "We'll test failover during maintenance window only"
- Need regular unannounced drills

❌ "Secondary region is always off to save cost"
- Can't meet RTO < 30min if secondary is cold

### Pro Tips

✅ "I'd implement automatic failover with human verification gates"

✅ "I'd separate read replicas for analytics from DR replicas"

✅ "I'd use DNS failover with short TTLs for instant switching"

✅ "I'd conduct monthly DR drills with actual failover"

---

## Scenario 4: Design CI/CD Pipeline for Microservices (200+ Services)

### Requirements

**Scale:**
- 200+ independent microservices (Java, Python, Go, Node.js)
- 500+ developers across 5 teams
- 10,000+ commits per day
- Deployment targets: GKE (3 regions), Cloud Run, Cloud Functions
- Production SLA: 99.99% uptime

**Constraints:**
- Build time: < 5 minutes (including tests, security scan, registry push)
- Deployment time: < 10 minutes (rolling update)
- Rollback: < 2 minutes
- Cost: < $50K/month infrastructure

**Quality Gates:**
- Unit tests: 80%+ coverage required
- Integration tests: 100% passing
- Security scan: No critical/high vulnerabilities
- Performance: P95 latency < 500ms

### Architecture

#### Layer 1: Version Control & Triggering

```
GitHub Repository Structure:
├─ microservice-1/
│  ├─ src/
│  ├─ tests/
│  ├─ Dockerfile
│  ├─ helm/
│  │  ├─ values.yaml
│  │  └─ templates/
│  └─ .github/workflows/  # GitHub Actions
│
├─ microservice-2/
└─ shared-libs/

Trigger Model:
- Commit to feature branch → Run tests, lint, security scan (no deploy)
- PR created → Run full test suite, post results as comments
- Merge to main → Deploy to staging, run smoke tests
- Tag release (v1.2.3) → Deploy to production
- Manual trigger → Deploy any version anywhere (with approval)

Webhook Flow:
GitHub Push → GitHub Actions Runner
         ↓
    Parse git event
    Determine affected services (monorepo aware)
    Queue builds for changed services only
    (Avoids rebuilding everything)
```

#### Layer 2: Build Pipeline

```
Stage 1: Test (Parallel, ~2 min)
├─ Unit tests: go test, pytest, npm test
├─ Integration tests: docker-compose up + API tests
├─ Linting: golangci-lint, pylint, ESLint
└─ SAST: SonarQube or Snyk scan

Stage 2: Build Image (Parallel, ~1.5 min)
├─ Multi-stage Dockerfile (optimize for size)
├─ From base image (cached layer)
├─ Mount build cache in Docker (layer caching)
└─ Tag: us-central1-docker.pkg.dev/PROJECT/images/service:COMMIT_SHA

Stage 3: Registry Push (~0.5 min)
├─ Authenticate to Artifact Registry
├─ Push image layers (incremental)
├─ Push image index (manifest)
└─ Tag with branch/version

Stage 4: Security Scan (~1 min)
├─ Trivy scan for CVEs in image
├─ Policy check: fail if critical vulnerability
├─ SBOM generation
└─ Sign image with Binary Authorization

Total Build Time: ~5 minutes

GitHub Actions Workflow Example:
name: Build and Deploy

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/setup-buildx-action@v2
      - run: npm test
      - run: npm run lint
      - uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: us-central1-docker.pkg.dev/PROJECT/images/${{ github.event.repository.name }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  security-scan:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE }}
          severity: CRITICAL,HIGH
```

#### Layer 3: Deploy Pipeline

```
Staging Deployment (Automatic on merge to main):
1. Pull Helm chart from repo
2. Generate values-staging.yaml (secrets from Secrets Manager)
3. Helm upgrade --install on staging cluster
4. Run smoke tests (health check, critical paths)
5. If tests pass: promote to Canary
6. If tests fail: Slack alert, pause, manual investigation

Production Deployment (Canary, requires approval):
1. Create canary deployment (5% of traffic)
2. Monitor metrics for 10 minutes:
   - Error rate < 1%
   - P95 latency increase < 10%
   - Pod restarts = 0
3. If metrics OK: promote to 25% → 50% → 100%
4. If degradation detected: auto-rollback to previous version

Deployment Architecture:
GitOps Approach (ArgoCD):
├─ Git repo contains desired state (Helm values)
├─ ArgoCD watches git repo
├─ On commit, ArgoCD syncs to Kubernetes
├─ Helm values determined by:
│  ├─ Service version (from tag)
│  ├─ Environment overrides (prod values file)
│  └─ Feature flags (ConfigMap)
└─ All deployments traceable to git commit

Benefits:
- Declarative (desired state in git)
- Audit trail (git history)
- Rollback (revert commit)
- Disaster recovery (redeploy from git)
```

#### Layer 4: Monitoring & Feedback

```
Metrics Collected During Deploy:
- Build time (should be < 5min)
- Test coverage (should be >= 80%)
- Security scan results (critical vulns = fail)
- Deploy time (should be < 10min)
- Rollback count (spike = issue)

Post-Deploy Monitoring:
- Error rate spike > 2x baseline (auto-rollback)
- P95 latency increase > 20% (alert)
- Pod restart rate > 5%/hour (alert)
- CPU/Memory spike > 50% (alert)

Feedback Loop:
Build fails → Slack alert + PR comment
Tests fail → Email to team lead
Security scan finds CVE → Jira ticket, block deployment
Deploy succeeds → Slack notification with metrics
Rollback triggered → Page on-call engineer
```

### Tradeoffs

**Monorepo vs Polyrepo:**
- Monorepo: Atomic commits, shared-lib changes visible immediately
- Polyrepo: Independent versioning, easier to parallelize
- Choose: Monorepo for tightly coupled services, Polyrepo for independent teams

**Fast Builds vs Image Size:**
- Fast: Layer caching, parallel builds, shared layers
- Small: Multi-stage builds, Alpine base image
- Choose: Balance with caching > size

**Automatic vs Manual Approval:**
- Automatic: Fast deploys, risk of bad deployments
- Manual: Safety but bottleneck
- Choose: Automatic to staging, manual approval to production

**Rolling Update vs Blue-Green:**
- Rolling: Less infrastructure, gradual rollout
- Blue-Green: Instant switch, requires 2x infrastructure
- Choose: Rolling with canary phase

### Scaling Considerations

**Build Capacity:**
- Peak: 500 commits/hour × 5 services/commit = 2500 builds/hour
- Need: 20 concurrent runners to handle
- GitHub Actions: 5K/month = $10/month per runner
- Or: Self-hosted runners on GCE (cheaper at scale)

**Deploy Capacity:**
- Peak: 200 services × 10 deploys/day = 2000 deploys/day
- Each deploy: 10 minutes = 334 hours/day needed
- Can parallelize to 30+ simultaneous with ArgoCD
- Need: 334 hours / 24 hours / 30 parallel = 0.46 clusters (1 cluster enough)

**Registry Storage:**
- 200 services × 10 versions × 500MB/image = 1TB
- Artifact Registry cost: $0.26/GB/month = ~$260/month
- Cleanup: Delete images older than 30 days

**Monitoring Data:**
- 200 services × 100 metrics × 60 samples/hour = 1.2M time series
- Cloud Monitoring: $0.26 per 1M Sampling, $20K/month
- Alternative: Prometheus (self-hosted, cheaper)

### Cost Optimization

```
Scenario: 200 services, 50 deploys/day

Costs:
1. GitHub Actions:
   - 2500 builds/hour × 2 hours peak
   - Standard runner: 10 min × 2500 = 416 hours/month
   - $0.035/min = $870/month

2. Artifact Registry:
   - 1000 images pushed/day
   - Layer size: ~100MB average
   - Ingress: free
   - Egress: $0.15/GB = 150GB × $0.15 = $22/month
   - Storage: 5000 images × 500MB × 6 months retention
   - Cost: ~$400/month

3. GKE Staging/Canary:
   - 3 node pools (staging, canary prod-a, prod-b)
   - 2 nodes each × $50/month = $300/month
   - (Use preemptible for staging: $10/month)

4. Cloud Build alternative:
   - 120 free build-minutes/day (too small)
   - $0.006/min after free = $270/month

5. Security scanning:
   - Trivy: free (self-hosted)
   - Or Snyk: $3K/team/month

Total: ~$1,500/month
Target achieved: < $2K/month infrastructure

Optimizations:
1. Use GitHub Actions (cheaper than Cloud Build at scale)
2. Artifact Registry staging cleanup policy (delete old images)
3. Preemptible nodes for non-production (10x cheaper)
4. Layer caching (dramatically reduces build time + bandwidth)
5. Self-hosted runners for peak loads
```

### Follow-up Questions

**Q1: How would you handle database schema migrations in this pipeline?**
- Expected: Flyway/Liquibase for declarative migrations, coordination between services

**Q2: What if you need to deploy a hotfix in 2 minutes?**
- Expected: Bypass some tests, skip canary, direct production deploy (with approval), immediate monitoring

**Q3: How do you handle backward compatibility for API changes?**
- Expected: API versioning, deprecation periods, feature flags, coordinated deployment windows

**Q4: What happens if the canary detects bad metrics?**
- Expected: Auto-rollback to previous version, page secondary-channel engineer

### Red Flags

❌ "We deploy every service independently with no coordination"
- Leads to cascading failures

❌ "Tests are optional to speed up deployments"
- Costs more in production firefighting

❌ "We don't have rollback capability"
- Essential for production resilience

### Pro Tips

✅ "I'd use feature flags instead of fast rollbeacks for safer deployments"

✅ "I'd implement canary deployments to detect issues before affecting all users"

✅ "I'd cache Docker layers aggressively to keep builds under 5 minutes"

✅ "I'd use GitOps (ArgoCD) for declarative, auditable deployments"

---

## Scenario 5: Design Scalable GCP Architecture for 100M Users Globally

### Requirements

**Scale:**
- 100 million monthly active users
- 50,000 requests per second peak
- 500 TB data, growing 10% per month
- 5 global regions (Asia, Europe, Americas)
- SLA: 99.99% uptime, P95 latency < 500ms

**Services:**
- API servers (stateless microservices)
- Real-time analytics (click streams, recommendations)
- File storage (videos, user uploads)
- Message queues (order processing, notifications)
- Data warehouse (business intelligence)

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Edge Layer (CDN)                             │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐   │
│  │Cloud CDN  │  │Cloud CDN  │  │Cloud CDN  │  │Cloud CDN  │   │
│  │ (Global)  │  │ (Global)  │  │ (Global)  │  │ (Global)  │   │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
└────────┬──────────────────────────────────────────────┬──────────┘
         │                                             │
    ┌────────────────────────────────────────────────────────────┐
    │         Global HTTP(S) Load Balancer                       │
    │   (Anycast IP, routes to nearest region)                   │
    └───┬──────────────────────────────────────────────────────┬─┘
        │                                                    │
   ┌────────────────────────────┐              ┌─────────────────────┐
   │   REGION 1: us-central1    │              │ REGION 2: eu-west1  │
   ├────────────────────────────┤              ├─────────────────────┤
   │                            │              │                     │
   │ ┌──────────────────────┐   │              │ ┌──────────────────┐ │
   │ │Regional LB (internal)│   │              │ │Regional LB       │ │
   │ └──────────┬───────────┘   │              │ └────────┬─────────┘ │
   │            │               │              │          │           │
   │ ┌──────────────────────┐   │              │ ┌─────────────────┐  │
   │ │  GKE Autopilot      │   │              │ │GKE Autopilot    │  │
   │ │  (API Services)     │   │              │ │(API Services)   │  │
   │ │  ├─ 100 pods API    │   │              │ ├─ 50 pods        │  │
   │ │  ├─ Auth service    │   │              │ └─────────────────┘  │
   │ │  └─ Payment service │   │              │                     │
   │ └─────────┬───────────┘   │              │ ┌─────────────────┐  │
   │           │               │              │ │Cloud Memorystore│  │
   │ ┌──────────────────────┐   │              │ │ (Redis Cache)   │  │
   │ │   Cloud Memorystore  │   │              │ └─────────────────┘  │
   │ │   (Session cache)    │   │              │                     │
   │ │   64GB             │   │              │ ┌─────────────────┐  │
   │ │                    │   │              │ │Cloud SQL Replica│  │
   │ │                    │   │              │ │(Read-only)      │  │
   │ └────────────────────┘   │              │ └─────────────────┘  │
   │                            │              │                     │
   │ ┌──────────────────────┐   │              └─────────────────────┘
   │ │   Cloud SQL          │   │
   │ │   (Primary DB)       │   │  Replication Delay: < 100ms
   │ │   ├─ 500GB           │   │  Replication Type: Synchronous
   │ │   ├─ HA config       │   │
   │ │   └─ Failover        │   │
   │ └────────┬─────────────┘   │
   │          │                │
   │ ┌────────────────────────┐ │
   │ │   Cloud Storage        │ │              ┌─────────────────────┐
   │ │   (Object Storage)     │ │              │  REGION 3+4+5       │
   │ │   ├─ Hot: 10TB         │ │              │  (Similar setup)    │
   │ │   ├─ Warm: 50TB        │ │              │  (Replicated data)  │
   │ │   ├─ Cold: 200TB       │ │              └─────────────────────┘
   │ │   └─ Archive: 240TB    │ │
   │ └────────┬────────────────┘ │
   │          │                  │
   │ ┌────────────────────────┐  │
   │ │   Pub/Sub Topic        │  │
   │ │   (order-events)       │  │
   │ │   ├─ Subscription: ML  │  │
   │ │   ├─ Subscription: DW  │  │
   │ │   └─ Subscription: Email│ │
   │ └────────────────────────┘  │
   │                              │
   │ ┌────────────────────────┐   │
   │ │   Cloud Tasks          │   │
   │ │   (Delayed execution)  │   │
   │ └────────────────────────┘   │
   └────────────────────────────────┘

Central Hub (for shared services):
┌────────────────────────────────────┐
│   BigQuery (Data Warehouse)        │
│   ├─ 500TB data                    │
│   ├─ Raw dataset (daily snapshots) │
│   ├─ Analytics dataset (processed) │
│   └─ ML dataset (feature store)    │
├────────────────────────────────────┤
│   Dataflow (Real-time ETL)         │
│   ├─ Pub/Sub → BigQuery            │
│   ├─ Stream processing             │
│   └─ Enrichment pipelines          │
├────────────────────────────────────┤
│   Vertex AI (ML Platform)          │
│   ├─ Recommendation model          │
│   ├─ Anomaly detection             │
│   └─ Prediction service            │
├────────────────────────────────────┤
│   Cloud Logging & Monitoring       │
│   ├─ Centralized logs              │
│   ├─ 100B events/day               │
│   └─ Cross-region dashboards       │
└────────────────────────────────────┘
```

### Tradeoffs

**Regional vs Global Storage:**
- Global: Fast reads everywhere, eventual consistency
- Regional: Strong consistency, slower reads in far regions
- Choose: Regional primary + global replicas with eventual consistency

**Cache Size vs Cost:**
- Large cache: Hit rate > 95%, expensive
- Small cache: Hit rate 70%, but very cheap
- Choose: 64GB per region for 80%+ hit rate

**Read Replicas vs Single Primary:**
- Multiple primaries: Complex, replication lag
- Single primary + read replicas: Simple, but bottleneck
- Choose: Single primary in primary region, read replicas globally

**Synchronous vs Async Replication:**
- Sync: Guaranteed consistency, slower writes
- Async: Fast writes, eventual consistency
- Choose: Async for non-critical data, Sync for payments

### Scaling Considerations

```
Capacity Analysis:

API Traffic:
- 50K req/sec × 100ms avg = 5000 concurrent requests
- GKE: 1 pod = 100 req/sec → Need 500 pods
- Replicas across 5 regions: 500 / 5 = 100 pods per region
- Actual: <100 pods (many requests complete faster)
- Autoscaling: Min=20, Max=200 per region

Database Capacity:
- 50K orders/sec × 1KB = 50MB/sec ingestion
- Storage: 500TB over 1 year
- Read replicas: Same storage, additional network cost

Cache Capacity:
- Session data: 100M users × 1KB = 100GB
- Distributed: 100GB / 5 regions × 2 replicas = 40GB per region
- Hit rate target: > 85%

Message Backlog:
- 50K events/sec × 3600 sec = 180M events/hour
- Retention: 24 hours = 4.3B events
- Pub/Sub will auto-scale, but monitor for lag

CDN Cache:
- Cacheable responses: ~20% of traffic
- Cache hit rate: >95% for cacheable items
```

### Cost Optimization

```
Monthly Cost Breakdown for 100M users:

1. GKE Autopilot (5 regions):
   - 500 pods × $5/month = $2,500
   - Network egress: 50K req/sec × 1KB × 3600 × 24 × $0.14/GB
   - = 6TB/day × $0.14 = $840/day = $25K/month
   - Total: $27,500

2. Cloud SQL (HA, 500GB):
   - Primary: db-custom-4-15GB = $2,000/month
   - HA failover: +50% = $1,000
   - Replicas (3 more regions): × 3 = $9,000
   - Backups: 500GB × 3 snapshots × $0.026 = $40
   - Total: $12,040

3. Cloud Storage (500TB):
   - Hot (10TB): $0.020/GB = $200
   - Warm (50TB): $0.010/GB = $500
   - Cold (200TB): $0.004/GB = $800
   - Archive (240TB): $0.0025/GB = $600
   - Total: $2,100

4. Pub/Sub (50K msg/sec):
   - 50K/sec × 86400 sec = 4.32B msg/day
   - $6 per 1B messages = $26/day = $780
   - Storage: 4.3B messages × 1KB × $0.026 = $100
   - Total: $880

5. BigQuery (Data Warehouse):
   - Storage: 500TB × $6.25/TB = $3,125
   - Queries: 100 queries/day × 1TB each × $6.25/TB
   - = 100 × $6.25 = $625
   - Total: $3,750

6. Cloud CDN:
   - Cache fraction: ~20% of traffic × average objects
   - Cache size: 1TB of frequently accessed data
   - Cost: $15/month (mostly covered by egress savings)

7. Monitoring & Logging:
   - API metrics: 1M time series × $1/month = $1,000
   - Log ingestion: 100B logs/day × $0.50/GB = $50/day = $1,500
   - Total monitoring: $2,500

8. Dataflow (ETL):
   - 10K vCPU-hours/day × $0.25/hour = $2,500

TOTAL MONTHLY: ~$50,000

Cost Optimization Strategies:
1. Use Committed Use Discounts (CUD): -25% on GKE/GCE
2. Use Preemptible VMs for non-critical batch jobs: -70%
3. Archive data > 90 days old: Cold storage vs Hot
4. Set log retention to 30 days (not 365)
5. Use BigQuery slot reservations for predictable queries
6. Optimize DB queries to reduce BigQuery scans
7. Use Cloud CDN aggressively for static assets
8. Implement request rate limiting to reduce API abuse

With optimizations: ~$35,000/month
```

### Follow-up Questions

**Q1: What happens if a region goes down?**
- Expected: Traffic reroutes to other regions via Load Balancer, within 10 seconds

**Q2: How do you handle data consistency across regions?**
- Expected: Primary-replica + eventual consistency, separate consistency models for different data types

**Q3: What if there's a database failover?**
- Expected: Automatic failover to HA replica (same region), manual failover to other regions

### Pro Tips

✅ "I'd use Cloud CDN to offload 90% of static traffic cost"

✅ "I'd implement regional caching (Memorystore) before hitting database"

✅ "I'd use eventual consistency for non-critical data to scale infinitely"

✅ "I'd monitor inter-region latency and use traffic splitting to optimize"

---

## Scenario 6: Design Real-time Analytics Platform (50 Billion Events/Day)

### Requirements

**Scale:**
- 50 billion events per day (580K events/sec peak)
- Raw data: 50TB/day ingestion
- Query response time: < 5 seconds for 90th percentile
- Real-time dashboards: Update every 10 seconds
- Data retention: Hot (1 week), Warm (1 month), Cold (1 year)
- Cost: < $100K/month

**Event Types:**
- User clicks (60% of traffic): ~350K/sec
- Product views: ~100K/sec  
- Transactions: ~50K/sec
- Server metrics: ~80K/sec

**Use Cases:**
1. Real-time dashboards (execs, marketing)
2. Ad hoc SQL queries (analysts)
3. Machine learning training (recommendation engine)
4. Alerting on anomalies

### Architecture

```
Data Flow Pipeline:

Client App → Events (JSON)
    ↓ (HTTPS)
┌─────────────────────────────────────┐
│      Cloud Load Balancer            │  Global ingestion endpoint
│      (any-cast IP)                  │
└────────────────┬────────────────────┘
                 ↓
    ┌────────────────────────────┐
    │   Cloud Pub/Sub            │  50K events/sec - acts as buffering
    │   Topic: events-raw        │
    │   ├─ Sub: realtime         │  → Dataflow (stream processing)
    │   ├─ Sub: batch            │  → Cloud Storage (raw backup)
    │   └─ Sub: ml               │  → Vertex AI (feature store)
    └────────┬───────────────────┘
             │
    ┌────────────┴─────────────────────────────────┐
    │                                              │
┌───────────────────────────┐        ┌───────────────────────────┐
│   Dataflow (Real-time)    │        │  Cloud Storage (Backup)   │
│   ├─ Window: 10 sec       │        │  ├─ Raw JSON (parquet)    │
│   ├─ Aggregations:        │        │  ├─ Retention: 30 days    │
│   │  ├─ Count by page     │        │  └─ Storage class: COLD   │
│   │  ├─ Count by time     │        └───────────────────────────┘
│   │  ├─ Revenue by hour   │
│   │  └─ Top 100 items   │
│   ├─ Join enrichment     │
│   │  (User profile API)  │
│   └─ Publish results →  │
│      Pub/Sub:          │
│      results-10sec     │
└─────────────┬───────────┘
              │
   ┌──────────┴──────────┐
   │                     │
┌────────────────┐  ┌────────────────────┐
│ BigQuery       │  │ Cloud Firestore    │
│ streaming      │  │ (RTDB)             │
│ inserts        │  │ ├─ Latest metrics  │
│ ├─ Raw table   │  │ ├─ Top products    │
│ │ (50TB/day)   │  │ └─ User segments   │
│ │ SLA: 1sec    │  │                    │
│ ├─ Aggregated  │  │ Update freq: 10sec │
│ │ hourly       │  │ Read format: JSON  │
│ └─ Optimized   │  └────────────────────┘
│   for queries  │
└────────┬───────┘
         │
    ┌────────────────────────────────┐
    │   Cloud Firestore / Memcache   │  Real-time dashboard cache
    │   ├─ Top 100 products          │  Serves metrics to UI
    │   ├─ Revenue so far today      │  Avg latency: < 50ms
    │   ├─ User segments             │
    │   └─ Anomaly alerts            │
    └────────────────────────────────┘

Query Path (for analysts):
BigQuery SQL Query Script
    ↓
BigQuery Scheduler (daily reports)
├─ Extract schema from Pub/Sub events (DDL generator)
├─ Schema migration (add new fields)
└─ Publish results as table

Dashboard Refresh:
Looker / Data Studio
    ├─ Source: BigQuery aggregated tables
    ├─ Refresh: 10 minutes for aggregate dashboards
    ├─ Source: Firestore for real-time dashboards
    └─ Refresh: Every 10 seconds
```

### Schema & Data Layout

```
BigQuery Raw Table:
CREATE TABLE `project.analytics.events_raw` (
  timestamp TIMESTAMP,
  event_type STRING,
  user_id STRING,
  session_id STRING,
  page_path STRING,
  product_id STRING,
  product_category STRING,
  product_price FLOAT64,
  action STRING,
  referrer STRING,
  device_type STRING,
  geo_country STRING,
  geo_region STRING,
  client_version STRING,
  raw_payload STRING  -- Full JSON
) 
PARTITION BY DATE(timestamp)
CLUSTER BY user_id, product_id;

Partitioning Strategy:
- By DATE: Query only relevant days
- Clustering by user_id + product_id: Pre-filters before full scan

BigQuery Aggregated Table (Materialized View):
CREATE MATERIALIZED VIEW monthly_product_revenue AS
SELECT
  DATE(timestamp) as date,
  product_category,
  product_id,
  COUNT(*) as view_count,
  SUM(product_price) as total_revenue,
  COUNT(DISTINCT user_id) as unique_users
FROM `project.analytics.events_raw`
WHERE DATE(timestamp) > DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY 1, 2, 3;

Refresh: Daily (can query immediately, updates in background)
```

### Tradeoffs

**Streaming vs Batch Processing:**
- Streaming: Real-time results (10-60 sec), complex logic
- Batch: 1-24 hour latency, simpler processing
- Choose: Streaming for dashboard, Batch for deep analytics

**Hot vs Warm vs Cold Data:**
- Hot (1 week): Fast queries, expensive ($20/TB)
- Warm (1 month): Medium queries, cheaper ($10/TB)
- Cold (1 year): Slow queries, very cheap ($3/TB)
- Choose: Tiered - query recent data from hot, archive to cold

**Stateful vs Stateless Processing:**
- Stateful: Windowing, session tracking (harder to scale)
- Stateless: Event-by-event (easy to scale)
- Choose: Mostly stateless, state only for windowing

### Scaling Considerations

```
Traffic Scaling:

Current: 580K events/sec
Projected growth: 3x in 2 years = 1.7M events/sec

Pub/Sub Scaling:
- Auto-scales to millions of events/sec
- Cost: $6 per 1B messages
- 50B events/day = $300

Dataflow Scaling:
- 580K events/sec × 100ms processing = 58K concurrent events
- Need: 580 vCPU (1 event per 10µs per vCPU)
- Autoscale worker pool: Min 100, Max 500 workers
- Cost: $0.25/vCPU-hour × 500 × 24 × 30 = $90K/month
- Optimization: Batch processing reduces to $20K/month

BigQuery Scaling:
- Streaming inserts: 1M rows/sec max per table
- Solution: Multiple tables with federation
- Query capacity: Automatically scales (slot-based pricing)
- Storage: Auto-scales (pay per TB)

Firestore Scaling:
- Document writes: 25K/sec for small documents
- Pub/Sub → Firestore: 10 sec aggregation → 5K writes/sec
- Document reads: 100K/sec (easily handles)
```

### Cost Optimization

```
Monthly Cost (50B events/day):

1. Cloud Pub/Sub:
   - 50B messages × $6/1B = $300
   - Storage: 50B × 1KB × $0.026/GB/month = $1.3K
   - Total: $1.6K

2. Dataflow (Real-time Processing):
   - 580K events/sec × 100ms = 58K concurrent
   - 100 workers base, 500 max
   - Worker: n1-standard-4 = $10/day
   - 300 workers × 24 hours × 30 days × $0.19/hour = $41K
   - But with scaling down: ~$15K
   - Shuffle storage: 10% overhead = $2K

3. Cloud Storage (Raw backup):
   - 50TB/day ingestion
   - 30 days = 1.5PB
   - Cold storage: 1.5PB × $3/TB = $4.5K
   - Lifecycle: Auto-archive after 7 days

4. BigQuery:
   - Streaming inserts: 50B rows/day
   - Approximate cost: 50B × $0.06/1M rows = $3K
   - Query analysis: 100 queries/day × 1TB = 100TB scanned
   - $6.25 per TB = $625
   - BI Engine cache (2TB): $3K (speeds up dashboards)
   - Total: $6.6K

5. Firestore (Real-time metrics):
   - Writes: 5K/sec × 60 × 60 × 24 * 30 = 12.96B documents/month
   - Write: 12.96B × $0.06/1M = $776
   - Reads: 100K/sec (dashboard hits)
   - Read: Minimal with cache

6. Cloud Monitoring:
   - Metrics: 10K time series × $1/month = $10K
   - Log storage: Minimal
   - Total: $10K

7. Looker / Data Studio:
   - Dashboards: $500-1.5K/month

TOTAL: ~$40K/month

Optimization strategies:
1. Batch events: Instead of per-event ingestion, batch 100 events/request
2. Aggregate early: Pre-aggregate in Pub/Sub messages (not raw events)
3. Archive cold data: BigQuery lifecycle policies (auto-move to Cold)
4. Use BI Engine: 2TB cache reduces query cost 5-10x
5. Snapshot tables: Create hourly snapshots (cheaper than full query)
6. Use Parquet format: Compress better than JSON (40% reduction)
```

### Follow-up Questions

**Q1: How do you handle late-arriving data (event from 1 hour ago)?**
- Expected: Windowing strategy, grace period, backfill capability

**Q2: What if a query suddenly scans 100TB of data?**
- Expected: Query cost limits, slot reservations, cancellation thresholds

### Pro Tips

✅ "I'd use BI Engine cache to avoid full BigQuery scans"

✅ "I'd batch events before ingestion to reduce API calls"

✅ "I'd implement early aggregation in Dataflow to reduce downstream data"

✅ "I'd use Firestore for hot metrics, BigQuery for historical analysis"

---

## Scenario 7: Design Enterprise Kubernetes Cluster for Multi-Team Platform

### Requirements

**Scale:**
- 5 teams (Backend, ML, Frontend, DevOps, Data)
- 500+ microservices deployed
- 1000+ pods running concurrently
- 5 month upgrade cycle (no downtime requirement)

**Constraints:**
- Resource isolation (team A can't affect team B)
- Security: RBAC, network policies, pod security
- Cost tracking per team
- Self-service deployment with guardrails
- Compliance: PCI-DSS, HIPAA readiness

### Architecture

```
                    ┌─────────────────────────────┐
                    │  Cluster Control Plane      │
                    │  (Google Kubernetes Engine) │
                    │  ├─ API Server              │
                    │  ├─ etcd (HA)               │
                    │  └─ Controller Manager      │
                    └─────────────┬───────────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         │                        │                        │
   ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
   │  Node Pool 1 │        │  Node Pool 2 │        │  Node Pool 3 │
   │ (Gen purpose)│        │  (Memory)    │        │   (GPU)      │
   │  ├─ 10 nodes │        │  ├─ 5 nodes  │        │  ├─ 2 nodes  │
   │  ├─ e2-std-4 │        │  ├─ n1-mem-8 │        │  ├─ n1-std-4 │
   │  └─ Auto: 5-50│        │  └─ Auto: 2-10│       │  │ (A100 GPU) │
   └──────┬───────┘        └──────┬───────┘        └──────┬───────┘
          │                       │                       │
          │ Runs namespaced      │ For data-heavy       │ ML training jobs
          │ workloads            │ services             │
          │                      │                       │
          └──────────────────────┼───────────────────────┘
                                 │
     ┌───────────────────────────┼───────────────────────────┐
     │                           │                           │
  NAMESPACE 1              NAMESPACE 2                NAMESPACE 3
"backend-team"           "ml-team"                  "data-team"
┌────────────────────┐  ┌─────────────────┐      ┌─────────────────┐
│ Resources Quota:   │  │ Resources Quota:│      │ Resources Quota:│
│ CPU: 20 cores     │  │ CPU: 50 cores   │      │ CPU: 30 cores   │
│ Memory: 50GB      │  │ Memory: 200GB   │      │ Memory: 100GB   │
│ Storage: 500GB    │  │ GPU: 4 A100s    │      │ Storage: 2TB    │
│ PV count: 10      │  │ PV count: 50    │      │ PV count: 100   │
└────────┬───────────┘  └────────┬────────┘      └────────┬────────┘
         │                       │                       │
   ┌─────────────┐         ┌──────────────┐        ┌─────────────┐
   │ 50 pods max │         │ 150 pods max │        │ 200 pods max│
   └─────────────┘         └──────────────┘        └─────────────┘
```

**Namespace Design:**

```yaml
# 1. Team Namespaces with ResourceQuota
apiVersion: v1
kind: Namespace
metadata:
  name: backend-team
  labels:
    team: backend
    cost-center: "12345"

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: backend-quota
  namespace: backend-team
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "50Gi"
    limits.cpu: "40"
    limits.memory: "100Gi"
    pods: 50
    persistentvolumeclaims: 10
    services.nodeports: 2

---
# 2. Network Policy - Team isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-isolation
  namespace: backend-team
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow from same namespace
  - from:
    - namespaceSelector:
        matchLabels:
          name: backend-team
  # Allow from ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  # Allow DNS
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Allow to other services (explicit whitelist)
  - to:
    - namespaceSelector:
        matchLabels:
          name: backend-team
    ports:
    - protocol: TCP
      port: 5432

---
# 3. RBAC - Team role separation
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: backend-developer
  namespace: backend-team
rules:
- apiGroups: [""]
  resources: ["pods", "pods/logs", "pods/exec"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "create", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "update"]  # Can't delete (lead dev only)

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: backend-developer-binding
  namespace: backend-team
subjects:
- kind: Group
  name: "backend-team@company.com"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: backend-developer
  apiGroup: rbac.authorization.k8s.io

---
# 4. Pod Security Policy
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'MustRunAs'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

**Team Self-Service Portal:**

```
├─ Dashboard
│  ├─ Resource usage (CPU, Memory, Storage)
│  ├─ Cost breakdown ($/day)
│  ├─ Pod count, status
│  └─ Network I/O
│
├─ Deployment
│  ├─ Git repo → auto-build & deploy (ArgoCD)
│  ├─ Helm chart templating
│  ├─ Environment-specific values (dev/staging/prod)
│  └─ Automatic rolling updates
│
├─ Debugging Tools
│  ├─ Pod logs (streaming)
│  ├─ Pod exec (terminal)
│  ├─ Metrics (CPU, memory real-time)
│  └─ Events (pod lifecycle)
│
└─ Requests
   ├─ Increase resource quota (for lead dev approval)
   ├─ Request PV (with size/type)
   ├─ Request network policy exception
   └─ Request GPU access
```

**Monitoring & Cost Allocation:**

```
Prometheus Metrics (per team namespace):
- container_cpu_usage_seconds_total (by namespace)
- container_memory_usage_bytes (by namespace)
- kubelet_volume_stats_used_bytes (by namespace)
- network_io_bytes (by namespace)

Cost Calculation:
- Compute: vCPU-hours × hourly rate
- Memory: GiB-hours × hourly rate
- Storage: GiB-month × rate per tier
- Network: GiB transferred × rate
- Workload: Per-pod cost

Monthly Bill:
backend-team:
  ├─ CPU: 450 vCPU-hours × $0.045 = $20.25
  ├─ Memory: 750 GiB-hours × $0.006 = $4.50
  ├─ Storage: 50 GiB × $0.17/GiB-month = $8.50
  ├─ Network: 50 GiB egress × $0.14/GiB = $7
  └─ Total: $40.25/day = $1,208/month

Shared costs allocated proportionally:
- API Server: 5% (small)
- etcd: 5% share (small)
- Monitoring: 10% (infrastructure)
- Ingress controller: 20% (allocated by ingress rules)
```

### Tradeoffs

**Shared vs Isolated Cluster:**
- Shared: Cost efficient, but blast radius exists
- Isolated: Safe, but expensive
- Choose: Shared with strong isolation policies (namespace, network policy, resource quota)

**Node pool per team vs shared:**
- Per team: Guaranteed resources, higher cost
- Shared: Cost efficient, but competition for resources
- Choose: Shared pools with autoscaling and QoS

**Manual vs Auto-scaling:**
- Manual: Predictable, but requires planning
- Auto: Responsive, but can overshoot
- Choose: Auto horizontal pod scaling (HPA) + node pool autoscaling

### Scaling Considerations

```
1000 pods across cluster:
- etcd size: Each pod object ≈ 5KB
- etcd capacity: 1000 pods × 5KB = 5MB (easily fits)
- API query rate: 100 requests/sec per team × 5 teams = 500 req/sec
- API capacity: Google manages (no concern)

Node scaling:
- Total requests: 1000 pods × avg 1 core = 1000 cores
- Node capacity: n1-standard-4 = 4 cores usable (3.8 after kubelet)
- Nodes needed: 1000 / 3.8 = 263 nodes
- Configured: Auto-scale 50-300 nodes

Etcd scaling:
- Request rate: 100 ops/sec per team = 500 ops/sec total
- Etcd capacity: ~1000 ops/sec built-in
- Before issues: Would need >2000 pods or >1000 ops/sec
```

### Cost Optimization

```
Annual Cost (100-300 nodes, 1000 avg pods):

1. GKE Cluster (Autopilot or auto-managed):
   - 100 nodes baseline × $50/month = $5K
   - Autoscaling 100-300 avg 180 nodes × $50 = $9K
   - Total compute: $14K/month

2. Cluster monitoring:
   - Metrics: 5 teams × 1000 metrics = 5K series
   - Cost: 5K × $0.26/1K = $1.3K/month

3. Storage (all teams):
   - PVs: 200 PVs avg × 10GB = 2TB
   - SSD: 2TB × $0.17 = $340
   - Standard: (if mixed) $200
   - Snapshots: $100
   - Total: $640/month

4. Network:
   - Ingress: $16/month (fixed)
   - Inter-region: ~5% of workload traffic = $1K/month

Cost allocation to teams:
backend-team (40% of compute):
   - Compute: $5.6K
   - Storage: $256
   - Monitoring: $1.3K (shared)
   - Total: ~$7K/month

ml-team (40% of compute, GPU):
   - Compute: $5.6K
   - GPU: 4 × A100 = $2.5K each = $10K
   - Storage: $256
   - Total: ~$15.5K/month

Other teams: Proportional allocation

TOTAL: ~$35K-40K/month
```

### Follow-up Questions

**Q1: How do you prevent one team from starving others?**
- Expected: Resource quotas, Pod Priority and Preemption, node pool separate for critical workloads

**Q2: What if a pod is consuming memory but not requesting it?**
- Expected: Memory limits (OOMKill), memory requests to reserve capacity, vertical pod autoscaler

### Pro Tips

✅ "I'd use ResourceQuota + NetworkPolicy for hard isolation"

✅ "I'd implement Pod Disruption Budget for critical services"

✅ "I'd track costs per namespace and show teams their bill"

✅ "I'd use separate node pools for GPU/memory to optimize scheduling"

---

## Scenario 8: Design Multi-Region Active-Active System (Zero RPO, < 1s RTO)

### Requirements

**Uptime Guarantee:**
- Zero RPO (no data loss)
- RTO < 1 second (automatic failover)
- Active-active (both regions serve traffic, no "standby")
- Consistency: Strong for critical data, eventual for cache

**Geography:**
- US (primary) - us-central1
- EU (secondary) - eu-west1
- Japan (tertiary) - asia-northeast1

**Data:**
- User accounts: 100M users, ~2GB data (must be replicated)
- Session data: 1M active sessions, ~200GB (can tolerate brief loss)
- Product catalog: 1M products, ~1GB (replicate 2/3 of world)
- Orders: 50K/day, all replicated

### Architecture

```
User Traffic:
┌─ DNS (Anycast) → Route to nearest region
│
├─ Requests to US
│  └─ Load Balancer → us-central1-ilb
│     ├─ Pods (API layer)
│     ├─ Cache (Memorystore)
│     └─ Database (Cloud SQL)
│
├─ Requests to EU
│  └─ Load Balancer → eu-west1-ilb
│     ├─ Pods (API layer)
│     ├─ Cache (Memorystore)
│     └─ Database (Cloud SQL)
│
└─ Requests to Japan
   └─ Load Balancer → asia-northeast1-ilb
      ├─ Pods (API layer)
      ├─ Cache read-only (Memorystore)
      └─ Database read-only replica


Database Replication (MySQL):

Primary (US)
├─ Write  master.sql.internal:3306
├─ Binlog stream → 
├─ Cloud SQL Auth Proxy in deployment
└─ Read replicas:
   ├─ EU secondary (can be promoted to primary)
   ├─ JP read-only (disaster failover)

Replication setup:

CREATE USER 'replication'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';

SHOW MASTER STATUS;  -- Get binlog file + position
Binary_log_file | Position
mysql-bin.000015 | 1234567

-- On EU replica:
CHANGE MASTER TO
  MASTER_HOST='us-central1-...:3306',
  MASTER_USER='replication',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000015',
  MASTER_LOG_POS=1234567;

START SLAVE;
SHOW SLAVE STATUS;

Replication lag target: < 500ms RTO requirement
```

**Conflict Resolution (Write-Write Conflicts):**

```
What if both US and EU write at same time to different replicas?

Strategy 1: Single Writer (Primary)
├─ All writes go to US (primary)
├─ EU/JP can only read
├─ Failover scenario: EU becomes new primary
├─ Complexity: Routing changes during failover
└─ RPO: 0, RTO: 10+ seconds (DNS propagation)

Strategy 2: Multi-Master (Circular)
├─ US can write → replicates to EU → replicates to JP
├─ EU can write → replicates to JP → replicates to US
├─ Conflicts: LAST-WRITE-WINS based on timestamp
├─ Complexity: Version tracking, conflict resolution logic
├─ Risk: Data anomalies possible
└─ RPO: 0, RTO: <1 second

Strategy 3: Logical Partitioning
├─ By data center: US writes US data, EU writes EU data
├─ Cross-region read replicas only
├─ Conflicts: None (partitioned domain)
├─ Trade-off: Limited to domain model
└─ Best for: User accounts (by region), metadata

CHOSEN: Hybrid
├─ User accounts: Logical partitioning (by region)
├─ Session data: Multi-master (eventual consistency)
└─ Orders: Write to primary, read from replicas
```

**Application Layer:**

```
Global Load Balancer (Anycast IP)
Let's say IP: 203.0.113.1

Client DNS: api.example.com → 203.0.113.1

GCP routing (automatic):
├─ US client → 203.0.113.1 routes to US backend
├─ EU client → 203.0.113.1 routes to EU backend
└─ JP client → 203.0.113.1 routes to JP backend

No explicit routing logic needed (GCP handles)

Failover scenario:
If US region goes down:
├─ Global LB reroutes traffic to EU (within 10 seconds)
├─ EU becomes primary for all users
├─ Schema sync: EU → JP (asynchronous)
│  └─ Lag: ~1 minute before JP is current
└─ Users experience: 10-second delay, then recovery

Client perspective:
├─ Browser request times out after 30 seconds
├─ Browser retries
├─ 2nd attempt hits EU (now available)
└─ Connection works
```

**Consistency Model:**

```
Critical data (User accounts):
├─ Write to US master (0.1ms)
├─ Acknowledge to client (immediate)
├─ Replicate to EU (200ms latency)
├─ Replicate to JP (300ms latency)
├─ Consistency: Strong (all writes are serialized)

Session data (Temporary):
├─ Write to local region's Memorystore
├─ Replicate asynchronously to other regions
├─ If region dies: Session replication lag possible
├─ Trade-off: Session loss acceptable (re-login)

Product catalog:
├─ Replicate full catalog to US + EU
├─ Read-only replica in JP (popular products only)
├─ Consistency: Eventual (JP lags by 1 hour)
└─ Trade-off: Saves replication bandwidth
```

### Tradeoffs

**Binary Replication vs Row-based:**
- Binary: Small log size, non-deterministic functions
- Row-based: Deterministic, larger log, version-specific
- Choose: Row-based (safer for active-active)

**Synchronous vs Asynchronous:**
- Sync: Guaranteed replication, slow writes
- Async: Fast writes, risk of data loss on failover
- Choose: Sync for critical (user accounts), async for cache

**Geo-distribution vs Consolidation:**
- Geo: Low latency, high complexity, regulatory compliance
- Consolidated: Simple, high latency
- Choose: 2 regions minimum, 3 for critical services

### Scaling Considerations

```
Replication traffic between regions:
- 100M users × write rate 10% daily = 10M writes/day
- Write size: 1KB = 10GB/day traffic
- Peak: 10GB / 86400 sec = ~120KB/sec = 0.96 Mbps
- Cost: Negligible

Inter-region network cost:
- 0.96 Mbps * 30 days = 3.6 TB/month
- Cross-region egress: 3.6TB × $0.20/GB = $720/month
- For 3 regions: $1.4K/month

Replication lag impact:
- At 500ms lag, max writes in-flight: 500ms × write_rate
- At 10M writes/day: 10M / (86400 * 1000 / 500) = 58 writes in flight
- Buffering: No special handling needed (so small)
```

### Cost Optimization

```
Multi-region active-active setup:

1. Compute (GKE, 3 regions):
   - 50K req/sec / 3 regions = 16.7K per region
   - Per region 200 pods × $5/month = $1K per region
   - Total: 3 × $1K = $3K computes.

2. Database (Cloud SQL HA + replicas):
   - US primary (HA): db-custom-4-15GB = $2K
   - EU secondary (HA): $2K
   - JP read-only: $1K
   - Replication: ~$500
   - Total: $5.5K

3. Managed Memorystore (Session caching):
   - US: 64GB = $400
   - EU: 64GB = $400
   - JP: 16GB (read-only) = $100
   - Total: $900

4. Network costs:
   - Cross-region egress (replication): $1.4K
   - Global LB: $16
   - Cloud Interconnect (optional, for low-latency): Skip (use internet)

5. Monitoring:
   - Multi-region metrics: $3K

Total: ~$13.8K/month

Optimizations:
1. Use GKE Autopilot (no node management overhead)
2. Use read replicas; only promote on failover
3. Cache aggressively to reduce DB replication load
4. Use auto-scaling for compute
5. Use Cloud CDN for static content (reduces region-to-region traffic)
```

### Follow-up Questions

**Q1: What if US and EU write the same record at same time?**
- Expected: Timestamp-based conflict resolution, write-ahead logs, versioning

**Q2: How do you test active-active failover?**
- Expected: Simulate region failure, monitor metrics, verify no data loss

### Pro Tips

✅ "I'd use row-based replication for active-active (safer than binary)"

✅ "I'd keep session data local, replicate user accounts synchronously"

✅ "I'd use logical partitioning to avoid write conflicts"

✅ "I'd test failover monthly to ensure RTO < 1 second"

