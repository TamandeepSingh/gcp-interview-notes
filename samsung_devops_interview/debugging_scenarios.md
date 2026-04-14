# Debugging Scenarios - Samsung DevOps Interview

## Scenario 1: Intermittent 500 Errors - Where Do You Start?

### The Problem
Production API has intermittent 500 errors:
- Happens 0.5% of the time (affects 1 in 200 requests)
- No pattern (not time-based, not correlated with traffic)
- Affects 3 out of 15 pods inconsistently
- Kubernetes logs show nothing unusual
- Application team says "we didn't change anything"
- Last deploy was 2 weeks ago

**Walk through your debugging process. Be specific with commands.**

### Expected Answer (DETAILED)

#### Step 1: Triage & Rapid Assessment (First 5 minutes)

```bash
# Get baseline data immediately
$ kubectl get pods -l app=api -o wide
$ kubectl get events -A | grep -i error | tail -20
$ kubectl top nodes

# Check pod restart count (indicates crashes):
$ kubectl get pods -l app=api -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].restartCount}{"\n"}{end}'

# Check pod logs for recent errors:
$ for pod in $(kubectl get pods -l app=api -o name); do
    echo "=== $pod ==="
    kubectl logs --tail=50 $pod 2>&1 | grep -i "error\|exception\|panic" | tail -5
  done

# Check HTTP errors in application metrics:
$ kubectl exec -it <pod> -- curl -s http://localhost:9090/metrics | grep http_requests_total

# Immediate questions to ask:
- Did error rate change after a specific event?
- Which endpoints are affected?
- What's the error message?
- Memory/CPU spike before errors?
```

#### Step 2: Isolate the Problem

```bash
# 1. Check if it's application or infrastructure
# Test endpoint directly:
$ kubectl port-forward <pod> 8080:8080
$ for i in {1..100}; do
    curl -s http://localhost:8080/api/health | jq .status
    sleep 1
  done | sort | uniq -c

# 2. Identify which pods have issues:
$ kubectl logs <pod1> -n production | grep -i "500\|error" | tail -20
$ kubectl logs <pod2> -n production | grep -i "500\|error" | tail -20
$ kubectl logs <pod3> -n production | grep -i "500\|error" | tail -20

# 3. Get detailed symptoms:
$ kubectl exec <pod> -- ps aux | grep -i java  # Memory process?
$ kubectl exec <pod> -- free -h               # Memory available?
$ kubectl exec <pod> -- df -h                  # Disk full?
$ kubectl exec <pod> -- netstat -an | wc -l    # Connection count?

# 4. Check application logs more deeply:
$ kubectl logs <pod> --timestamps=true | tail -100 | \
  grep -B2 -A2 "500\|Exception\|Error"
```

#### Step 3: Root Cause Hypotheses

```
Based on symptoms, narrow down to most likely:

Hypothesis 1: Resource Exhaustion
├─ Memory leak: Error rate increases over pod lifetime
│  Fix: Check pod age vs error rate
│  $ kubectl get pods -o jsonpath='{.items[*].metadata.creationTimestamp}' | tr ' ' '\n' | sort
│
├─ File descriptor leak: netstat shows increasing connections
│  Fix: 'lsof' within pod, restart pod
│  $ kubectl exec <pod> -- lsof -p 1 | wc -l
│
└─ Disk full: df -h shows usage near 100%
   Fix: Check /tmp, /var/log, container ephemeral storage
   $ kubectl exec <pod> -- du -hs /tmp /var/log /app

Hypothesis 2: External Dependency Failure
├─ Database connection pool exhausted
│  Fix: Check upstream database health
│  $ kubectl logs <pod> | grep -i "connection pool\|max connections"
│
├─ Cache (Redis) timeouts
│  Fix: kubectl get svc redis-cache
│  $ redis-cli --latency
│
└─ External API timeout
   Fix: Check API logs for timeouts
   $ kubectl logs <pod> | grep -i "timeout\|connection refused"

Hypothesis 3: Configuration/State Corruption
├─ Bad ConfigMap mounted
├─ Stale cache
└─ Invalid state in pod storage

Hypothesis 4: Bug That Only Triggers Under Load
├─ Race condition in code
├─ Concurrency issue
└─ Specific input triggers crash
```

#### Step 4: Test Each Hypothesis

```bash
# Hypothesis 1: Resource Exhaustion
# Get resource metrics for affected pods
$ kubectl top pods -l app=api --containers
NAME     CPU  MEMORY
pod1     100m 256Mi
pod2     800m 2048Mi  ← Spike!

# Check pod history:
$ kubectl describe pod pod2 | grep -A5 "Last State"

# If OOMKilled, increase memory limit:
$ kubectl set resources deployment api \
  --limits=memory=4Gi,cpu=2000m \
  --requests=memory=2Gi,cpu=500m

---

# Hypothesis 2: External Dependency Failure
# Check if database is responsive:
$ kubectl run -it --rm debug --image=postgres:14 -- \
  psql -h postgres-service -U user -d mydb -c "SELECT 1"

# Check Redis reachability:
$ kubectl run -it --rm debug --image=redis:7 -- \
  redis-cli -h redis-cache ping

# Check resource limits on dependencies:
$ kubectl logs deployment/postgres | grep -i "max_connections\|connection\|pool"

---

# Hypothesis 3: Configuration Corruption
# Compare ConfigMaps between affected and healthy pods:
$ kubectl get configmap myapp-config -o yaml > /tmp/cm.yaml
$ kubectl exec pod1 -- cat /etc/config/app.yaml > /tmp/pod1-config.yaml
$ diff /tmp/cm.yaml /tmp/pod1-config.yaml

# Check for stale volumes:
$ kubectl exec pod1 -- ls -la /var/cache/app/
$ kubectl exec pod1 -- find /tmp -mtime +7 -type f  # Old files?

---

# Hypothesis 4: Bug Triggered Under Load
# Run load test against specific pod:
$ kubectl run -it --rm load-gen --image=busybox -- \
  sh -c "for i in {1..1000}; do wget -q -O- http://api:8080/api/test & done; wait"

# Monitor in real-time:
$ kubectl logs <pod> -f | grep -i "error\|exception"
$ kubectl top pod <pod> --containers --no-headers

# If error reproduces under load:
- Likely race condition or resource exhaustion
- May need code review of high-concurrency paths
```

#### Step 5: Deep Dive - Strace / Debug

```bash
# For mysterious errors, capture system calls:
$ kubectl exec <pod> -- apt-get update && apt-get install -y strace
$ kubectl exec <pod> -- strace -e trace=network,open,read,write -p 1 -f

# OR use nsenter to debug from node:
$ NODE=$(kubectl get pod <pod> --template='{{.spec.nodeName}}')
$ ssh root@$NODE
# Use nsenter to enter pod namespace:
$ nsenter -t $(docker inspect $(docker ps | grep <pod> | awk '{print $1}') --format '{{.State.Pid}}') -n netstat -an

# Check kernel logs on node:
$ journalctl -xn | grep -i "oom\|killed"
```

#### Step 6: Application-Level Debugging

```bash
# Enable debug logging:
$ kubectl set env deployment api LOG_LEVEL=DEBUG

# Add more detailed error handling:
$ kubectl exec <pod> -- curl -s http://localhost:8080/api/debug/profile | head -50

# Check application metrics:
$ kubectl port-forward <pod> 9090:9090
$ curl http://localhost:9090/metrics | grep error_rate

# Look for specific errors in logs:
$ kubectl logs <pod> | grep -E "NullPointerException|IndexOutOfBoundsException|FileNotFoundException"

# Parse logs for patterns:
$ kubectl logs <pod> --tail=1000 | \
  grep "ERROR" | \
  awk '{print $NF}' | \
  sort | uniq -c | sort -rn | head -10
# This shows most common errors
```

#### Step 7: Root Cause Resolution

```
Once root cause identified:

If Memory Leak:
- Fix code (close resources)
- Increase memory temporarily
- Set resource requests/limits

If Connection Pool Exhausted:
- Increase pool size
- Fix connection handling
- Add monitoring for pool utilization

If External Dependency:
- Investigate dependency
- Add circuit breaker
- Implement retry logic

If Bug:
- Fix in code
- Deploy via canary (5% → 25% → 50%)
- Monitor error rate decreases
```

#### Step 8: Prevention

```bash
# 1. Add monitoring to detect this early:
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-error-rate
spec:
  groups:
  - name: api
    rules:
    - alert: HighErrorRate
      expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.001
      for: 5m
      annotations:
        summary: "API error rate > 0.1% for 5 minutes"

# 2. Add health checks:
$ kubectl patch deployment api --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/livenessProbe",
    "value": {
      "httpGet": {"path": "/health", "port": 8080},
      "initialDelaySeconds": 30,
      "periodSeconds": 10
    }
  }
]'

# 3. Add resource limits to prevent OOM:
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"

# 4. Add structured logging:
- Every error should have:
  - Request ID (trace distributed requests)
  - Timestamp
  - Service name
  - Error message
  - Stack trace
```

### Follow-up Questions

**Q1: What if the 500 error only shows in production metrics but no logs?**
- Expected: Upstream reverse proxy might be generating error, check nginx/LB logs

**Q2: How would you determine if it's a specific input causing the issue?**
- Expected: Collect request parameters from error log, reproduce locally, trace through code

**Q3: What's your process if the issue disappears after 20 minutes every time?**
- Expected: Time-based issue, check scheduled tasks, cron jobs, batch processes

### Red Flags

❌ "I would just restart the pods"
- Doesn't diagnose underlying issue

❌ "The error rate is too low to worry about"
- 0.5% on 1000 req/sec = 5 fail/sec = customer impact

❌ "I'll just increase memory limits and call it a day"
- Doesn't fix the root cause

### Pro Tips

✅ "I'd immediately check CPU/memory metrics correlating with error spikes"

✅ "I'd use HTTP tracing (`strace -e trace=network`) to see what the app is doing"

✅ "I'd look at pod age - if errors correlate with pod uptime, it's a resource leak"

✅ "I'd check external services (database, cache) before blaming the application"

---

## Scenario 2: Networking Debug - Mysterious Connection Refused

### The Problem
App pods can't connect to a newly deployed database service:
```
Error: connection refused on service postgres-db:5432
Error: getaddrinfo failed for postgres-db
```

But:
- Other services can connect to postgres-db fine
- Same app connects fine to OTHER databases
- DNS works fine (nslookup succeeds)
- Service and endpoints exist
- Network policies allow traffic

**How do you systematically debug this?**

### Expected Answer (DETAILED)

#### Step 1: Verify Service Exists and is Healthy

```bash
# 1. Check service definition:
$ kubectl get svc postgres-db
$ kubectl describe svc postgres-db

# 2. Check endpoints:
$ kubectl get endpoints postgres-db
$ kubectl describe endpoints postgres-db

# CRITICAL: If endpoints show no IPs, pods aren't healthy
$ kubectl get pods -l app=postgres  # Are pods running?
$ kubectl logs -l app=postgres      # Do they have errors?

# 3. Check service cluster IP:
$ kubectl get svc postgres-db -o jsonpath='{.spec.clusterIP}'

# If shows "None", it's a headless service (different connection model)
```

#### Step 2: DNS Resolution Tests

```bash
# From inside a working pod:
$ kubectl exec -it <working-pod> -- nslookup postgres-db
$ kubectl exec -it <working-pod> -- nslookup postgres-db.default
$ kubectl exec -it <working-pod> -- nslookup postgres-db.default.svc.cluster.local

# From the failing pod:
$ kubectl exec -it <failing-pod> -- nslookup postgres-db
$ kubectl exec -it <failing-pod> -- nslookup postgres-db.default.svc.cluster.local

# If DNS fails:
$ kubectl get configmap coredns-config -n kube-system
$ kubectl logs -n kube-system deployment/coredns

# Check if service discovery is working:
$ kubectl exec <pod> -- curl -s http://kubernetes.default/api/v1/namespaces/default/services/postgres-db

# Direct IP test (if DNS fails):
SERVICE_IP=$(kubectl get svc postgres-db -o jsonpath='{.spec.clusterIP}')
$ kubectl exec <failing-pod> -- nc -zv $SERVICE_IP 5432
```

#### Step 3: Network Connectivity Tests

```bash
# From failing pod, test basic connectivity:
$ kubectl exec <failing-pod> -- \
  ap download-release sh curl wget nc \
  postgresql-client

# Test 1: Resolve DNS
$ kubectl exec <failing-pod> -- nslookup postgres-db

# Test 2: Ping service IP (ICMP)
$ kubectl exec <failing-pod> -- ping -c 1 <service-ip>
# Note: Some clusters disable ping, don't rely on this

# Test 3: Test exact connection
$ kubectl exec <failing-pod> -- \
  nc -zv postgres-db 5432
# -z: just check connection
# -v: verbose

# Test 4: Telnet to service port
$ kubectl exec <failing-pod> -- \
  telnet postgres-db 5432
# If connection successful, will show "Connected"

# Test 5: Curl to service (if it listens on HTTP)
$ kubectl exec <failing-pod> -- \
  curl -v telnet://postgres-db:5432

# If connection is refused at this point:
- Service IP is correct but no one listening
- OR NetworkPolicy is blocking
- OR pod's default gateway misconfigured
```

#### Step 4: Check NetworkPolicy

```bash
# List all network policies:
$ kubectl get networkpolicy -A

# Check if policy affects this pod:
$ kubectl get networkpolicy -o yaml | yaml query '.[] | select(.metadata.namespace == "default")'

# Check ingress rules:
$ kubectl describe networkpolicy <policy-name>

# Is the failing pod allowed to reach postgres-db?
Example policy that would BLOCK:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-postgres
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 3306  # MySQL port
    # Note: NO rule for 5432 (PostgreSQL)!

# Test if NetworkPolicy is the issue:
# Temporarily delete the policy
$ kubectl delete networkpolicy block-postgres

# Retry connection
$ kubectl exec <failing-pod> -- nc -zv postgres-db 5432

# If it works, reapply corrected policy
```

#### Step 5: Check Firewall & Security Groups (Cloud Level)

```bash
# If using GCP:
$ gcloud compute firewall-rules list --filter="targetTags:default"
$ gcloud compute firewall-rules describe <rule-name>

# Check if rule allows 5432:
$ gcloud compute firewall-rules create allow-postgres \
  --allow=tcp:5432 \
  --source-ranges=10.0.0.0/8 \
  --target-tags=postgres

# If using AWS:
$ aws ec2 describe-security-groups --group-names default
$ aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 5432 \
  --cidr 10.0.0.0/8

# Test with tcpdump (from node):
$ NODE=$(kubectl get pod <failing-pod> --template='{{.spec.nodeName}}')
$ ssh root@$NODE
# tcpdump to capture traffic:
$ tcpdump -i eth0 'port 5432'
# Then from pod, try connecting - should see traffic or blocked packets
```

#### Step 6: Check Pod Routing Configuration

```bash
# From failing pod, check routes:
$ kubectl exec <failing-pod> -- route -n
$ kubectl exec <failing-pod> -- ip route

# Check default gateway:
$ kubectl exec <failing-pod> -- ip route | grep default

# If gateway is not set correctly, pod can't reach service
# Usually Kubernetes CNI plugin sets this, but check:

# Check CNI configuration:
$ kubectl get daemonset -n kube-system  # CNI plugin
$ kubectl logs -n kube-system ds/weave-net  # or your CNI

# Common issue: IPTables rules not applied
$ kubectl get nodes -o wide
$ NODE=$(kubectl get pod <failing-pod> --template='{{.spec.nodeName}}')
$ ssh root@$NODE
$ iptables -t nat -L -n | grep postgres  # Check NAT rules
$ iptables -L -n | grep 5432
```

#### Step 7: Check Pod's /etc/resolv.conf and DNS Config

```bash
# Inside pod, check resolver config:
$ kubectl exec <failing-pod> -- cat /etc/resolv.conf
# Should show:
# nameserver 10.96.0.10  (CoreDNS)
# search default.svc.cluster.local svc.cluster.local cluster.local

# If wrong, pod might have:
# - Wrong nameserver (using host's DNS)
# - Missing search domains
# - Fix: Check deployment volumeMounts or dnsPolicy

# Check pod's dnsPolicy:
$ kubectl get pod <failing-pod> -o jsonpath='{.spec.dnsPolicy}'
# Should be ClusterFirst (not Default or Custom)

# If wrong, update:
$ kubectl patch pod <failing-pod> --type merge \
  -p '{"spec":{"dnsPolicy":"ClusterFirst"}}'
```

#### Step 8: Application-Level Debug

```bash
# Check if app is using correct hostname/IP:
$ kubectl exec <failing-pod> -- env | grep -i db
$ kubectl exec <failing-pod> -- cat /etc/hosts

# Check connection details in app config:
$ kubectl get configmap myapp-config -o yaml

# Parse the config:
$ kubectl exec <failing-pod> -- \
  grep -i "host\|db\|postgres" /app/config.yaml

# If app is using IP instead of hostname:
- Might bypass DNS
- Check if IP is correct
- If service IP is different than expected, config is stale

# Verify with strace:
$ kubectl exec <failing-pod> -- strace -e openat,connect -p 1 2>&1
# Then trigger connection from another process
# Watch for connect syscall to postgres-db
```

#### Step 9: Deep Dive - Packet Capture

```bash
# If all above checks pass, use tcpdump:

# From node where pod is running:
NODE=$(kubectl get pod <failing-pod> --template='{{.spec.nodeName}}')
ssh root@$NODE

# Find pod's network namespace:
POD_IP=$(kubectl get pod <failing-pod> -o jsonpath='{.status.podIP}')

# Capture traffic:
$ tcpdump -i any -nn 'host $POD_IP and port 5432' -w /tmp/pg-debug.pcap

# From another terminal, trigger the connection:
$ kubectl exec <failing-pod> -- psql -h postgres-db -U user -d mydb

# Analyze pcap:
$ wireshark /tmp/pg-debug.pcap
# Look for: SYN packets, RST packets (connection refused), timeout

# Common patterns:
- SRC: pod-ip:random DEST: svc-ip:5432 SYN → No RST = firewall drop
- SRC: pod-ip:random DEST: svc-ip:5432 SYN → SYN-ACK → RST = app refused
- SRC: pod-ip:random DEST: unknown:5432 = DNS resolution failed
```

### Follow-up Questions

**Q1: How would you debug if DNS resolves but connection times out?**
- Expected: Network policy or firewall blocking, check TCP handshake with tcpdump

**Q2: What if service works from one pod but not another?**
- Expected: NetworkPolicy might be label-based, check pod labels, RBAC

### Red Flags

❌ "Obviously network is down"
- Too vague, need systematic approach

❌ "DNS is fine, must be the application"
- Network might be blocking even if DNS works

### Pro Tips

✅ "I'd start with `nslookup` then `nc -zv` to isolate DNS vs network vs service"

✅ "I'd check NetworkPolicy first, it's a common and easy mistake"

✅ "I'd use tcpdump if all logical checks pass - physics don't lie"

---

## Scenario 3: Logs Not Appearing in Elasticsearch

### The Problem
Your microservices are generating logs (you can see them with `kubectl logs`), but they're not showing up in Elasticsearch. The Kibana dashboard shows 0 logs for the last 2 hours.

Last successful log ingestion was at 14:00 UTC. It's now 16:05 UTC (2+ hours of missing logs).

**Walk through how you'd systematically restore logging.**

### Expected Answer (DETAILED)

#### Step 1: Verify Logs Are Being Generated

```bash
# Confirm pods are actually writing logs:
$ kubectl logs <pod> --tail=50
# Should show recent logs

# Check multiple pods:
$ for pod in $(kubectl get pods -l app=myapp -o name | head -3); do
    echo "=== $pod ==="
    kubectl logs $pod --tail=5
  done

# If logs appear in kubectl but not in Elasticsearch = ingestion problem
# If logs DON'T appear in kubectl = application problem

# Check pod's actual log file:
$ kubectl exec <pod> -- cat /tmp/app.log | tail -20
$ kubectl exec <pod> -- tail -f /var/log/app.log | head -10
```

#### Step 2: Verify Fluent Bit is Running

```bash
# Check if fluent bit daemon set is running:
$ kubectl get daemonset -n kube-system | grep -i fluent
$ kubectl get pods -n kube-system | grep -i fluent

# Expected output:
# fluent-bit-5xpzk   1/1   Running
# fluent-bit-jkl8p   1/1   Running

# If pods are missing or not running:
$ kubectl describe daemonset fluent-bit -n kube-system
# Check why it's not deploying to nodes

# Check fluent bit pod logs:
$ kubectl logs -n kube-system -l app=fluent-bit --tail=50

# Common errors:
# ERROR: fluent_kube Get("service")
# ERROR: could not connect to Elasticsearch
# ERROR: Out of memory
```

#### Step 3: Check Fluent Bit Configuration

```bash
# Get the fluent bit configuration:
$ kubectl get configmap fluent-bit-config -n kube-system -o yaml

# Look for critical sections:
# 1. INPUT section (reads from container logs)
[INPUT]
    Name              tail
    Parser            docker
    Path              /var/log/containers/*.log
    Tag               kube.*
    Refresh_Interval  5

# 2. FILTER section (enriches logs)
[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# 3. OUTPUT section (sends to Elasticsearch)
[OUTPUT]
    Name            es
    Match           kube.*
    Host            elasticsearch.logging
    Port            9200
    HTTP_User       elastic
    HTTP_Passwd     ${ELASTICSEARCH_PASSWORD}

# Check if output is correct:
$ kubectl get service elasticsearch -n logging
$ kubectl get service -n logging | grep elasticsearch

# Verify service is accessible:
$ kubectl exec -it <fluent-bit-pod> -n kube-system -- \
  curl -s http://elasticsearch.logging:9200/_cluster/health
```

#### Step 4: Test Fluent Bit Connectivity to Elasticsearch

```bash
# From fluent bit pod, test Elasticsearch:
$ kubectl exec -it <fluent-bit-pod> -n kube-system -- bash

# Inside pod:
$ curl -v http://elasticsearch.logging:9200
# Should return 200 with cluster info

# If fails, check DNS:
$ nslookup elasticsearch.logging
$ nslookup elasticsearch.logging.svc.cluster.local

# Test from another pod in the cluster:
$ kubectl run -it --rm debug --image=curlimages/curl -- \
  curl http://elasticsearch.logging:9200/_cluster/health

# Check if authentication is required:
$ curl -u elastic:password http://elasticsearch:9200/_cluster/health

# Check Elasticsearch indices:
$ curl http://elasticsearch:9200/_cat/indices
# Look for kibana_sample_data_logs, logs-*

# Check if indices are receiving writes:
$ curl http://elasticsearch:9200/logstash-*/_stats/indexing
```

#### Step 5: Check Fluent Bit Filtering

```bash
# Get fluent bit logs to see if it's actually processing logs:
$ kubectl logs -n kube-system <fluent-bit-pod> -f | grep -i "error\|match\|dropped"

# Check if logs are being dropped due to filters:
$ kubectl logs -n kube-system <fluent-bit-pod> 2>&1 | grep -i "filtered\|drop\|skip"

# Common errors:
# [error] Could not match any record from tail
# [error] JSON corruption, invalid escape sequence

# View more detailed logs:
$ kubectl logs -n kube-system <fluent-bit-pod> -f --tail=100 | head -50

# Filter for warnings:
$ kubectl logs -n kube-system <fluent-bit-pod> | grep -i "warn\|retry\|reconnect"
```

#### Step 6: Check Pod Logs Path Configuration

```bash
# Verify tail plugin is reading from correct path:
$ kubectl exec -it <fluent-bit-pod> -n kube-system -- bash

# Check if log files exist:
$ ls -la /var/log/containers/

# Count log files:
$ ls /var/log/containers/ | wc -l

# Check a specific pod's log file:
$ POD_ID=$(kubectl get pod <pod-name> -o jsonpath='{.metadata.uid}')
$ ls -la /var/log/containers/*${POD_ID}*

# Parse one log file to see format:
$ head -5 /var/log/containers/default_myapp-xyz_myapp_abc123def456.log | jq .

# Expected format:
# {
#   "log": "INFO: Request received",
#   "stream": "stdout",
#   "time": "2024-04-13T14:30:00.123456789Z"
# }

# If format is wrong, Docker logging is misconfigured
```

#### Step 7: Verify Elasticsearch Credentials & Permissions

```bash
# Check if Elasticsearch password is correct:
$ kubectl get secret elasticsearch-credentials -n logging -o yaml

# Decode password:
$ kubectl get secret elasticsearch-credentials -n logging \
  -o jsonpath='{.data.password}' | base64 -d

# Test authentication:
$ kubectl exec -it <fluent-bit-pod> -n kube-system -- bash

# Inside pod:
$ curl -u elastic:password http://elasticsearch:9200/_security/user
# Should return user info

# Check if user has write permissions:
$ curl -u elastic:password http://elasticsearch:9200/_cat/security/privileges | grep indices

# If permission denied, grant write access:
$ curl -X PUT -u elastic:password \
  http://elasticsearch:9200/_security/role/fluent-writer \
  -H "Content-Type: application/json" \
  -d '{
    "indices": [{"names": ["logs-*"], "privileges": ["write", "create"]}]
  }'
```

#### Step 8: Check Elasticsearch Storage

```bash
# Check if Elasticsearch is out of disk space:
$ curl http://elasticsearch:9200/_cat/allocation
# Look for "disk.avail" - if near 0, that's the problem

# Check if writes are blocked:
$ curl http://elasticsearch:9200/_cat/indices?v | grep YELLOW
# YELLOW or RED status = issues

# Check cluster health:
$ curl http://elasticsearch:9200/_cluster/health
# Should show "status": "green"

# If RED/YELLOW, check watermark settings:
$ curl http://elasticsearch:9200/_cluster/settings | grep watermark

# Common error when out of space:
# "status": 429, "error": "index_template_missing_exception"
# Fix: Delete old indices or add disk space

# Delete old indices older than 7 days:
$ curl -X DELETE http://elasticsearch:9200/logs-2024.04.01
$ curl -X DELETE http://elasticsearch:9200/logs-2024.04.02
```

#### Step 9: Check Index Template & Mapping

```bash
# Verify index template exists:
$ curl http://elasticsearch:9200/_cat/templates
$ curl http://elasticsearch:9200/_cat/templates/logs

# Get template details:
$ curl http://elasticsearch:9200/_index_template/logs | jq .

# Check if template defines how logs should be indexed:
$ curl http://elasticsearch:9200/logs-*/_mapping | jq '.*.mappings.properties' | head -20

# If template is missing, create one:
$ curl -X PUT http://elasticsearch:9200/_index_template/logs-template \
  -H "Content-Type: application/json" \
  -d '{
    "index_patterns": ["logs-*"],
    "template": {
      "settings": {
        "number_of_shards": 1,
        "number_of_replicas": 0
      },
      "mappings": {
        "properties": {
          "timestamp": {"type": "date"},
          "message": {"type": "text"},
          "level": {"type": "keyword"}
        }
      }
    }
  }'
```

#### Step 10: Restart Fluent Bit and Monitor

```bash
# Force fluent bit restart:
$ kubectl rollout restart daemonset fluent-bit -n kube-system

# Monitor rollout:
$ kubectl rollout status daemonset fluent-bit -n kube-system

# Watch logs being processed:
$ kubectl logs -n kube-system -l app=fluent-bit -f | grep -i "output\|elasticsearch"

# Wait 2-3 minutes for fluent bit to catch up

# Verify logs in Elasticsearch:
$ curl http://elasticsearch:9200/_cat/indices | grep logs

# Query recent logs:
$ curl -X POST http://elasticsearch:9200/logs-*/_search \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "range": {
        "timestamp": {
          "gte": "now-5m"
        }
      }
    },
    "size": 10
  }' | jq '.hits.hits[0]._source'
```

### Root Cause (Most Common)
❌ Elasticsearch out of disk space / cluster RED status
❌ Authentication failed (wrong password, insufficient permissions)
❌ Elasticsearch index template misconfigured or missing
❌ Fluent Bit pod crashed or not enough replicas
❌ Network connectivity blocked between fluent-bit and Elasticsearch

### Resolution
✅ Check `curl elasticsearch:9200/_cluster/health` for cluster status
✅ Verify credentials: `curl -u elastic:pass elasticsearch:9200/_security/user`
✅ Check disk space: `curl elasticsearch:9200/_cat/allocation`
✅ Restart fluent bit: `kubectl rollout restart daemonset fluent-bit`
✅ Monitor ingestion with: `curl elasticsearch:9200/_cat/indices`

### Pro Tips

✅ "I'd check Elasticsearch cluster health first - most common failure point"

✅ "I'd test connectivity from fluent bit pod directly before debugging config"

✅ "I'd look at Elasticsearch disk watermark settings - often the silent killer"

---

## Scenario 4: Fluent Bit Pod Stuck in Crash Loop

### The Problem
Fluent Bit daemonset pod keeps restarting:

```bash
$ kubectl get pods -n kube-system -l app=fluent-bit
NAME                    READY   STATUS             RESTARTS   AGE
fluent-bit-5xpzk        0/1     CrashLoopBackOff   12         5m
fluent-bit-jkl8p        0/1     CrashLoopBackOff   10         5m
```

Logs were working fine until your team deployed a new fluent bit config 30 minutes ago.

**Debug what's causing the crash loop and fix it.**

### Expected Answer (DETAILED)

#### Step 1: Get Crash Logs

```bash
# Get pod logs with timestamps:
$ kubectl logs -n kube-system <fluent-bit-pod> \
  --previous --timestamps=true | tail -50

# Check for restart and crash messages:
$ kubectl logs -n kube-system <fluent-bit-pod> -p 2>&1 | tail -100

# Look for specific error patterns:
$ kubectl logs -n kube-system <fluent-bit-pod> -p 2>&1 | grep -i "error\|panic\|segfault"

# Common crash messages:
# "ERROR: [plugins] error loading config"
# "ERROR: json error"
# "Segmentation fault"
# "OOMKilled" (exit code 137)
```

#### Step 2: Check Pod Status Details

```bash
# Get exit code and reason:
$ kubectl describe pod <fluent-bit-pod> -n kube-system | grep -A10 "State:"

# Expected output shows:
# State:          Waiting
#   Reason:       CrashLoopBackOff
#   Message:      Back-off 5m0s restarting failed container fluent-bit

# Check last termination reason:
$ kubectl get pod <fluent-bit-pod> -n kube-system -o jsonpath='{.status.containerStatuses[0].lastState}'

# Exit code meanings:
# 1 = general error
# 2 = misuse of command
# 127 = command not found
# 137 = OOMKilled (out of memory)
# 139 = segmentation fault
```

#### Step 3: Validate Configuration Syntax

```bash
# Get the configmap that was recently changed:
$ kubectl get configmap fluent-bit-config -n kube-system -o yaml

# Save to file for editing:
$ kubectl get configmap fluent-bit-config -n kube-system -o yaml > fluent-bit-config.yaml

# Check syntax of the config file:
$ cat fluent-bit-config.yaml | grep -A100 "Corefile\|data:"

# Common config errors:
1. Missing quotes around values
   [OUTPUT]
       Name es
       Host elasticsearch (should be "elasticsearch")

2. Typo in plugin name
   [FILTER]
       Name kuberenetes (typo! should be "kubernetes")

3. Missing required fields
   [OUTPUT]
       Name es
       (missing Host and Port)

4. Invalid indentation (YAML-like format)
   [OUTPUT]
   Name es
    Host elasticsearch (mixed indentation)
```

#### Step 4: Test Config in Standalone Mode

```bash
# Create a test pod with fluent bit to validate config:
$ kubectl run -it --rm fluent-bit-test \
  --image=fluent/fluent-bit:latest \
  --restart=Never \
  -- fluent-bit -c /fluent-bit/etc/fluent-bit.conf -v

# Or validate config without running it:
$ fluent-bit -c config.conf -c /etc/fluent-bit/fluent-bit.conf --dry-run

# Run inside container:
$ kubectl exec -it <fluent-bit-pod> -n kube-system -- \
  fluent-bit -c /fluent-bit/etc/fluent-bit.conf --dry-run 2>&1 | head -50

# If config is valid, output will show:
# [2024/04/13 14:30:00] [ info] Configuration:
# [2024/04/13 14:30:00] [ info] flush time     : 1 sec.

# If invalid, you'll see:
# [2024/04/13 14:30:00] [error] Error reading config file: /fluent-bit/etc/fluent-bit.conf
```

#### Step 5: Check Resource Constraints

```bash
# Check if OOMKilled (exit code 137):
$ kubectl get pod <fluent-bit-pod> -n kube-system \
  -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}'
# If returns "OOMKilled", memory is the issue

# Check current resource limits:
$ kubectl get daemonset fluent-bit -n kube-system -o yaml | grep -A10 "resources:"

# Example:
# resources:
#   requests:
#     memory: "64Mi"
#     cpu: "100m"
#   limits:
#     memory: "128Mi"
#     cpu: "200m"

# Check actual memory usage before crash:
$ kubectl top pods -n kube-system -l app=fluent-bit

# If memory usage > limits, increase limits:
$ kubectl set resources daemonset fluent-bit \
  --limits=memory=512Mi,cpu=500m \
  --requests=memory=256Mi,cpu=250m \
  -n kube-system
```

#### Step 6: Revert Recent Changes

```bash
# Check what config was deployed last:
$ kubectl rollout history daemonset fluent-bit -n kube-system

# Expected output:
# REVISION  CHANGE-CAUSE
# 3         <none>
# 4         <none>

# Get the previous version:
$ kubectl rollout undo daemonset fluent-bit -n kube-system

# Monitor rollout:
$ kubectl rollout status daemonset fluent-bit -n kube-system

# Verify pods are now running:
$ kubectl get pods -n kube-system -l app=fluent-bit
# Should show Running, not CrashLoopBackOff

# If still crashing, undo further:
$ kubectl rollout undo daemonset fluent-bit -n kube-system \
  --to-revision=2
```

#### Step 7: Check Elasticsearch Connectivity During Startup

```bash
# Add debugging to fluent bit:
$ kubectl set env daemonset/fluent-bit \
  FLUENT_UID=0 \
  HTTP_DEBUG=1 \
  -n kube-system

# Restart daemonset:
$ kubectl rollout restart daemonset fluent-bit -n kube-system

# Check verbose logs:
$ kubectl logs -n kube-system <fluent-bit-pod> -f 2>&1 | grep -i "connect\|elasticsearch\|error"

# Expected during startup:
# Connecting to Elasticsearch at elasticsearch:9200
# Response: 200 OK

# If connection fails:
# Failed to create client
# Connection refused

# Verify Elasticsearch is running:
$ kubectl get pods -n logging | grep elasticsearch
$ kubectl get svc -n logging | grep elasticsearch
```

#### Step 8: Fix Specific Config Issues

```bash
# Example 1: Missing Elasticsearch password
# Error in logs: "401 Unauthorized"
# Fix:
$ kubectl get secret elasticsearch-credentials -n logging \
  -o jsonpath='{.data.password}' | base64 -d

# Add to configmap:
$ kubectl set env daemonset/fluent-bit \
  ELASTICSEARCH_PASSWD=<password> \
  -n kube-system

# Example 2: Wrong Elasticsearch service name
# Error in logs: "Failed to resolve host"
# Fix: Use FQDN
# Change: Host elasticsearch
# To: Host elasticsearch.logging.svc.cluster.local

# Example 3: TLS verification failure
# Error: "certificate verify failed"
# Fix:
$ kubectl set env daemonset/fluent-bit \
  FLUENT_BIT_SKIP_TLS_VERIFY=on \
  -n kube-system
```

#### Step 9: Validate Fix with Test Log

```bash
# Deploy a test pod that generates logs:
$ kubectl run test-logger --image=busybox -- \
  sh -c 'for i in {1..10}; do echo "TEST LOG $i"; sleep 1; done'

# Wait for logs to appear:
$ sleep 5

# Check if logs reached Elasticsearch:
$ curl http://elasticsearch.logging:9200/_cat/indices

# Query for test logs:
$ curl -X POST http://elasticsearch.logging:9200/logs-*/_search \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "match": {
        "kubernetes.pod_name": "test-logger"
      }
    }
  }' | jq '.hits.hits[].source.message'

# If logs appear, fluent bit is working!
```

### Root Cause (Most Common)
❌ Config syntax error (typo in plugin name, missing quotes)
❌ Memory limit too low (OOMKilled)
❌ Elasticsearch connectivity failure during initialization
❌ Missing credentials or authentication failure
❌ Pod image pull error or corrupted image

### Resolution
✅ Check pod events: `kubectl describe pod`
✅ Validate config syntax: `fluent-bit --dry-run`
✅ Check exit code: `kubectl get pod -o jsonpath='{.containerStatuses[0].lastState}'`
✅ Increase memory limits if OOMKilled  
✅ Rollback to previous working version

### Pro Tips

✅ "I'd check the exit code first - tells you if it's config, memory, or runtime crash"

✅ "I'd validate config with `--dry-run` before deploying to production"

✅ "I'd use `kubectl rollout undo` as first recovery step"

---

## Scenario 5: Kubernetes Logs Not Showing Event Metadata

### The Problem
Logs are reaching Elasticsearch, but they're missing critical metadata:

```
Missing fields:
- kubernetes.pod_name
- kubernetes.namespace_name
- kubernetes.container_name
- kubernetes.host
- kubernetes.labels.*
- kubernetes.annotations.*

Your Kibana dashboard shows only:
{
  "log": "[INFO] User login",
  "stream": "stdout",
  "time": "2024-04-13T14:30:00.123456789Z"
}

Expected (with enrichment):
{
  "log": "[INFO] User login",
  "stream": "stdout",
  "time": "2024-04-13T14:30:00.123456789Z",
  "kubernetes": {
    "pod_name": "api-5xpzk",
    "namespace_name": "production",
    "container_name": "api",
    "host": "gke-node-2",
    "labels": {
      "app": "api",
      "team": "backend"
    }
  }
}
```

**Debug why metadata is missing.**

### Expected Answer (DETAILED)

#### Step 1: Verify Kubernetes Filter is Configured

```bash
# Get fluent bit config:
$ kubectl get configmap fluent-bit-config -n kube-system -o yaml

# Look for FILTER section with kubernetes plugin:
$ kubectl get configmap fluent-bit-config -n kube-system -o yaml | \
  grep -A20 "\[FILTER\]" | grep -A20 "kubernetes"

# Expected config:
[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
    Merge_Log           On
    Keep_Log            Off

# If FILTER section is missing or wrong, add it:
$ kubectl edit configmap fluent-bit-config -n kube-system
```

#### Step 2: Check Fluent Bit Service Account Permissions

```bash
# Verify fluent bit has RBAC access to read pod metadata:
$ kubectl get serviceaccount fluent-bit -n kube-system

# Get the role:
$ kubectl get role -n kube-system | grep fluent

# Get role details:
$ kubectl get role fluent-bit -n kube-system -o yaml

# Should have permissions to get pods and namespaces:
rules:
- apiGroups: [""]
  resources: ["namespaces", "pods"]
  verbs: ["get", "watch", "list"]

# If permissions missing, add role:
$ kubectl create role fluent-bit \
  --verb=get,watch,list \
  --resource=pods,namespaces \
  -n kube-system

$ kubectl create rolebinding fluent-bit \
  --role=fluent-bit \
  --serviceaccount=kube-system:fluent-bit \
  -n kube-system
```

#### Step 3: Verify Fluent Bit Can Access Kubernetes API

```bash
# Test from fluent bit pod:
$ kubectl exec -it <fluent-bit-pod> -n kube-system -- bash

# Inside pod:
$ curl -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  https://kubernetes.default.svc:443/api/v1/pods

# Should return list of pods

# If fails with 403, check service account:
$ curl -I https://kubernetes.default.svc:443

# If you can't access k8s API, kubernetes plugin won't work
```

#### Step 4: Check Log Tag Matching

```bash
# Verify input logs have correct tag:
[INPUT]
    Name              tail
    Parser            docker
    Path              /var/log/containers/*.log
    Tag               kube.*  # <-- Important! Must start with kube
    Refresh_Interval  5

# Filter must match the tag:
[FILTER]
    Name                kubernetes
    Match               kube.*  # <-- Must match input tag

# If tag doesn't match, filter never runs

# Check actual log file format:
$ kubectl exec -it <fluent-bit-pod> -n kube-system -- \
  ls -la /var/log/containers/ | head -10

# Each file name has pattern:
# namespace_podname_containerid.log
# production_api-5xpzk_api-container-id.log

# Fluent bit parses this to extract namespace/pod/container
```

#### Step 5: Debug Kubernetes Plugin Enrichment

```bash
# Add debug logging for kubernetes filter:
$ kubectl set env daemonset/fluent-bit \
  FLUENT_BIT_LOG_LEVEL=debug \
  -n kube-system

# Restart:
$ kubectl rollout restart daemonset fluent-bit -n kube-system

# Check debug logs:
$ kubectl logs -n kube-system <fluent-bit-pod> -f 2>&1 | \
  grep -i "kubernetes\|enrichment\|merge"

# You should see:
# [debug] kubernetes filter enriching record
# [debug] added kubernetes.pod_name

# If not enriching, check:
# - Service account permissions
# - API connectivity
# - Tag matching
```

#### Step 6: Verify JSON Parsing

```bash
# Check if input JSON parser is correct:
# Look at a raw log file:
$ kubectl exec -it <fluent-bit-pod> -n kube-system -- \
  head -1 /var/log/containers/production_api-*_*.log | jq .

# Expected output:
# {
#   "log": "[INFO] message",
#   "stream": "stdout",
#   "time": "2024-04-13T14:30:00.123456789Z"
# }

# If jq fails, JSON is malformed

# Check parser config:
$ kubectl get configmap fluent-bit-parsers -n kube-system -o yaml | \
  grep -A10 "\[PARSER\] docker"

# Parser must be able to parse container log format
```

#### Step 7: Check Output Configuration

```bash
# Verify output section in fluent bit config:
$ kubectl get configmap fluent-bit-config -n kube-system -o yaml | \
  grep -A20 "\[OUTPUT\]"

# Should have:
[OUTPUT]
    Name            es
    Match           kube.*
    Host            elasticsearch.logging
    Port            9200
    Logstash_Format On  # <-- Important! Formats field names
    Logstash_Prefix logstash

# If Logstash_Format is Off, fields won't have proper naming
```

#### Step 8: Query Actual Elasticsearch Data

```bash
# Query actual log structure in Elasticsearch:
$ curl -s http://elasticsearch.logging:9200/logs-*/_search \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "match_all": {}
    },
    "size": 1
  }' | jq '.hits.hits[0]._source' | head -50

# Should show kubernetes fields:
# "kubernetes": {
#   "pod_name": "api-5xpzk",
#   "namespace_name": "production"
# }

# If kubernetes object is empty, enrichment didn't happen

# Check what fields exist:
$ curl -s http://elasticsearch.logging:9200/logs-*/_mapping | \
  jq '.*.mappings.properties' | grep -i kubernetes
```

#### Step 9: Fix & Verify

```bash
# If issue found, make sure fluent bit config has:

1. Correct INPUT tag:
[INPUT]
    ...
    Tag               kube.*

2. Kubernetes FILTER:
[FILTER]
    Name                kubernetes
    Match               kube.*
    Merge_Log           On

3. Proper RBAC:
$ kubectl get clusterrole system:aggregate-to-view -o yaml | grep kubernetes

# Restart with fixed config:
$ kubectl rollout restart daemonset fluent-bit -n kube-system

# Wait 2 minutes and check new logs:
$ sleep 120

# Query new logs:
$ curl -s http://elasticsearch.logging:9200/logs-*/_search \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "range": {
        "@timestamp": {
          "gte": "now-2m"
        }
      }
    }
  }' | jq '.hits.hits[0]._source.kubernetes'
```

### Root Cause (Most Common)
❌ Kubernetes filter not configured in fluent bit
❌ Fluent bit service account lacks permissions to read pods
❌ Log tag doesn't match filter pattern
❌ Input logs not in correct JSON format
❌ Logstash_Format disabled in output section

### Resolution
✅ Add kubernetes filter to fluent bit configmap
✅ Verify service account RBAC permissions
✅ Check input tag matches filter pattern (both must be kube.*)
✅ Enable Logstash_Format in ES output
✅ Test from fluent bit pod: `curl kubernetes.default.svc`

### Pro Tips

✅ "I'd verify k8s API access from fluent bit pod first"

✅ "I'd check that input TAG and filter MATCH both use kube.* pattern"

✅ "I'd query Elasticsearch to see the actual log structure before debugging further"

---

## Scenario 6: Pod Network Connectivity Timeout Issues

### The Problem
Your pods can sometimes connect to an external API (`api.external-service.com:443`), but other times requests timeout after exactly 30 seconds:

```
$ time curl https://api.external-service.com/health
curl: (7) Failed to connect to api.external-service.com port 443: Connection timed out
real    0m30.456s

$ time curl https://api.external-service.com/health
200 OK
real    0m0.234s
```

Intermittent failures suggest:
- Network instability
- Connection pool exhausted
- DNS issues
- Firewall rules

**Walk through debugging network connectivity issues systematically.**

### Expected Answer (DETAILED)

#### Step 1: Confirm the Issue Pattern

```bash
# Create a debugging pod:
$ kubectl run -it --rm network-debug \
  --image=nicolaka/netshoot:latest \
  -- bash

# Inside the pod, run repeated tests:
$ for i in {1..10}; do
    echo "=== Attempt $i ==="
    time curl -v https://api.external-service.com/health -w "\nStatus: %{http_code}\n"
    sleep 2
  done

# Track success/failure rate:
$ for i in {1..20}; do
    curl -s -o /dev/null -w "%{http_code}\n" https://api.external-service.com/health & done
$ wait

# Determine if failures are:
- Consistent (network issue)
- Random (transient)
- Pattern-based (time-based? load-based?)
```

#### Step 2: Check DNS Resolution

```bash
# Test DNS from pod:
$ nslookup api.external-service.com

# Should show IP address(es)
# If returns different IPs on different calls = DNS issues

# Test multiple times:
$ for i in {1..5}; do
    nslookup api.external-service.com | grep "Address:"
  done

# Test from node:
NODE=$(kubectl get node -o name | head -1)
gcloud compute ssh ${NODE##node/} --zone=<ZONE>

$ nslookup api.external-service.com
$ dig api.external-service.com +stats  # Shows query time

# If DNS response time spikes, that's likely the issue

# Check internal DNS:
$ kubectl exec <pod> -- nslookup kubernetes.default
$ kubectl exec <pod> -- nslookup kubernetes.default.svc.cluster.local
```

#### Step 3: Test TCP Connection Directly

```bash
# Test TCP handshake without HTTP:
$ kubectl exec -it <pod> -- nc -zv api.external-service.com 443

# Expected output:
# api.external-service.com [1.2.3.4] 443 (https) open

# If times out:
# nc -zv api.external-service.com 443
# Connection timed out

# Test from multiple pods:
$ for pod in $(kubectl get pods -o name | head -3); do
    echo "=== $pod ==="
    kubectl exec $pod -- nc -zv api.external-service.com 443
  done

# If some succeed and some fail = pod/node specific issue
```

#### Step 4: Check Firewall Rules

```bash
# List all firewall rules:
$ gcloud compute firewall-rules list

# Check if rule allows traffic to external services:
$ gcloud compute firewall-rules list --filter="sourceRanges:10.0.0.0/8"

# Look for rule that allows egress:
$ gcloud compute firewall-rules describe default-allow-egress

# Expected:
# Direction: EGRESS
# Allow: tcp:443,udp:443,tcp:80,udp:80

# If no egress rule, create one:
$ gcloud compute firewall-rules create allow-external-https \
  --allow=tcp:443,tcp:80 \
  --destination-ranges=0.0.0.0/0

# Check if rule applies to your VPC:
$ gcloud compute firewall-rules describe rule-name
# Look for "network" field
```

#### Step 5: Check Egress NAT Gateway

```bash
# If using Cloud NAT for pod egress:
$ gcloud compute routers list

# Check NAT status:
$ gcloud compute routers describe <ROUTER> --region us-central1

# Verify NAT is working:
$ kubectl exec <pod> -- curl ifconfig.io
# Should return consistent external IP

# If different IPs on each call = NAT pool exhausted

# Check NAT pool size:
$ gcloud compute routers nats describe <NAT> \
  --router=<ROUTER> \
  --region=us-central1 \
  --format='value(natIps[])'

# Increase NAT pool if needed:
$ gcloud compute routers nats update <NAT> \
  --router=<ROUTER> \
  --region=us-central1 \
  --auto-allocate-nat-external-ips
```

#### Step 6: Check Connection Tracking

```bash
# From pod, check established connections:
$ kubectl exec <pod> -- netstat -tan | grep :443

# Should show ESTABLISHED connections

# Check if connection table is full:
$ kubectl exec <pod> -- cat /proc/net/nf_conntrack | wc -l

# Compare to limit:
$ kubectl exec <pod> -- cat /proc/sys/net/nf_conntrack_max

# If count near max = connection tracking table exhausted

# Increase limit:
$ kubectl exec node-name -- sysctl -w net.nf_conntrack_max=1000000

# Or via sysctl config:
$ sudo tee /etc/sysctl.d/99-nf.conf > /dev/null << EOF
net.nf_conntrack_max = 1000000
net.netfilter.nf_conntrack_tcp_timeout_established = 600
EOF
$ sudo sysctl -p
```

#### Step 7: Test from Node vs Pod

```bash
# Test from pod:
$ kubectl exec <pod> -- \
  timeout 5 bash -c '</dev/tcp/api.external-service.com/443 && echo "Connected"'

# Test from node:
NODE=$(kubectl get pod <pod> --template='{{.spec.nodeName}}')
gcloud compute ssh $NODE --zone=<ZONE>

$ timeout 5 bash -c '</dev/tcp/api.external-service.com/443 && echo "Connected"'

# If node succeeds but pod fails:
- Pod network policy blocking
- Pod's networking stack issue
- Service mesh (Istio) blocking

# Check NetworkPolicy:
$ kubectl get networkpolicy
$ kubectl describe networkpolicy <POLICY>
```

#### Step 8: Packet Capture

```bash
# Capture traffic from pod to external service:
$ kubectl exec <pod> -- tcpdump -i eth0 \
  'dst api.external-service.com and port 443' -w /tmp/debug.pcap

# Trigger connection from another terminal:
$ kubectl exec <pod> -- curl -v https://api.external-service.com

# Copy pcap from pod:
$ kubectl cp <namespace>/<pod>:/tmp/debug.pcap ./debug.pcap

# Analyze with Wireshark locally:
$ wireshark debug.pcap

# Look for:
- SYN packets (initiation)
- SYN-ACK responses (server accepts)
- RST packets (connection reset)
- Timeout (no response to SYN)

# Or analyze with tshark:
$ tcpdump -r debug.pcap -nn 'tcp.flags.syn==1'
```

#### Step 9: Check Service Mesh (if using Istio)

```bash
# If using Istio, might be intercepting traffic:
$ kubectl get virtualservice
$ kubectl get destinationrule

# Check if rules affect external traffic:
$ kubectl get virtualservice -A -o yaml | grep -i "external\|egress"

# Check Envoy proxy logs:
$ kubectl logs <pod> -c istio-proxy | tail -50

# If seeing connection timeout in Envoy:
[ Outlier

 Detection: All healthy! all gates open ]
# This might indicate circuit breaker is tripping

# Check circuit breaker settings:
$ kubectl get destinationrule <service> -o yaml | \
  grep -A10 "outlierDetection:"
```

#### Step 10: Implement Retry & Debugging

```bash
# Add retry logic to application:
import socket
import time

def connect_with_retry(host, port, retries=3):
    for attempt in range(retries):
        try:
            sock = socket.create_connection((host, port), timeout=5)
            sock.close()
            return True
        except socket.timeout:
            print(f"Attempt {attempt+1}: Connection timeout")
            time.sleep(2)
        except Exception as e:
            print(f"Attempt {attempt+1}: {e}")
            time.sleep(2)
    return False

# Test:
connect_with_retry("api.external-service.com", 443)

# Also enable DNS debugging:
$ strace -e openat,connect,sendto -p 1 2>&1 | head -100
# Shows syscalls being made, helps identify bottleneck
```

### Root Cause (Most Common)
❌ DNS resolution latency or failures  
❌ NAT port pool exhausted
❌ Connection tracking table full
❌ Firewall rule missing for external traffic  
❌ Cloud NAT not configured or unhealthy

### Resolution
✅ Test DNS from pod: `nslookup api.external-service.com`
✅ Test TCP: `nc -zv api.external-service.com 443`
✅ Check NAT health: `gcloud compute routers describe`
✅ Check firewall: `gcloud compute firewall-rules list`
✅ Monitor connection table: `cat /proc/net/nf_conntrack | wc -l`

### Pro Tips

✅ "I'd start with DNS resolution - often the silent culprit in intermittent failures"

✅ "I'd test from multiple pods to isolate if it's pod-specific or cluster-wide"

✅ "I'd check NAT pool exhaustion - can't see in normal metrics"

✅ "I'd implement exponential backoff retry logic in application code"

