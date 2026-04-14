# Kubernetes Scenarios - Samsung DevOps Interview

## Scenario 1: Silent Pod Memory Leak

### The Problem
Your production API service has 50 pods running. After 7 days, memory consumption per pod goes from 200MB to 800MB. Pods restart daily via Kubernetes, but immediately start the leak cycle again. Your team says "it's probably garbage collection overhead," which doesn't match the symptoms.

**How do you diagnose and fix this?**

### Expected Answer (DETAILED)

#### Step 1: Confirm Memory Leak Pattern
```bash
# Get memory metrics over time:
$ kubectl top pods -n production --sort-by=memory

# Check if it's CPU or memory:
$ kubectl top nodes
$ kubectl get nodes -o json | jq '.items[] | {name:.metadata.name, memory:.status.allocatable.memory}'

# Check pod age distribution:
$ kubectl get pods -n production -o json | \
  jq -r '.items[] | .metadata.name + " " + (.metadata.creationTimestamp | fromdateiso8601 | now - . | floor / 86400 | "\(.)d")' | \
  sort -k2 -n | tail -20
```

#### Step 2: Identify Culprit Container
```bash
# Get detailed memory stats:
$ for pod in $(kubectl get pods -n production -o name); do
    kubectl exec $pod -n production -- ps aux | sort -k6 -rn | head -5
    kubectl exec $pod -n production -- free -h
  done

# Check if memory is in container or kernel cache:
$ kubectl exec <pod> -n production -- cat /proc/meminfo | grep -E "^Mem|^Cached"

# Use container metrics API:
$ kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/production/pods
```

#### Step 3: Deep Dive - Where is Memory Going?
```bash
# If Node: container runtime cache
$ kubectl debug node/<node-name> -it --image=ubuntu

# Inside node:
# Check container memory:
$ docker ps | grep <pod-name>
$ docker stats <container-id>

# Check page cache:
$ grep -E "Cached|Buffers" /proc/meminfo

# Check kernel memory:
$ grep MemTotal /proc/meminfo
```

#### Step 4: Application-Level Investigation
```bash
# Common culprits:

# 1. Connection pool leaking connections
$ kubectl exec <pod> -- lsof -i -P -n | grep ESTABLISHED | wc -l

# 2. Unbounded cache
$ kubectl exec <pod> -- du -h /app/cache 2>/dev/null || echo "No cache dir"

# 3. Log accumulation (if logging to disk)
$ kubectl exec <pod> -- du -sh /var/log

# 4. Thread explosion (goroutines, threads)
$ kubectl exec <pod> -- ps -eLf | wc -l

# 5. Java Heap (if Java app)
$ kubectl exec <pod> -- jps -v | grep -i java
$ kubectl exec <pod> -- jmap -heap PID
```

#### Step 5: Capture Heap Dump (if applicable)
```bash
# For Java applications:
$ kubectl exec <pod> -- jmap -dump:live,format=b,file=/tmp/heap.bin <PID>
$ kubectl cp production/<pod>:/tmp/heap.bin ./heap.bin
# Analyze with MAT (Memory Analyzer Tool) or jhat

# For Go applications:
$ kubectl exec <pod> -- curl http://localhost:6060/debug/pprof/heap > heap.prof
$ go tool pprof heap.prof

# For Python:
$ kubectl exec <pod> -- pip install memory-profiler
$ kubectl exec <pod> -- python -m memory_profiler <script>
```

#### Step 6: Prevention and Monitoring
```bash
# Set memory limits to kill leaky pods:
$ kubectl patch deployment api-service -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "api",
          "resources": {
            "limits": {"memory": "500Mi"},
            "requests": {"memory": "200Mi"}
          }
        }]
      }
    }
  }
}'

# Monitor with Prometheus:
$ cat << 'EOF' > memory-leak-alert.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: memory-leak-detection
spec:
  groups:
  - name: memory
    interval: 1m
    rules:
    - alert: PodMemoryLeakDetected
      expr: |
        (container_memory_usage_bytes / container_spec_memory_limit_bytes) > 0.9
        AND
        (rate(container_memory_usage_bytes[24h]) > 0)
      for: 30m
      annotations:
        summary: "Pod {{ $labels.pod }} showing memory leak"
EOF

# Set up live monitoring:
$ kubectl top pods -n production --containers --sort-by=memory &
$ watch -n 5 'kubectl top pods -n production'
```

#### Step 7: Code-Level Fix
```go
// Example: Connection Pool Leak in Go
// BEFORE: Connections not closed
resp, _ := http.Get(url)
// Missing: resp.Body.Close()

// AFTER: Defer closing
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()

// Or use connection pooling with limits:
client := &http.Client{
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        MaxConnsPerHost:     100,
    },
    Timeout: 30 * time.Second,
}
```

### Follow-up Questions

**Q1: Your pod restarted 50 times this week. Why?**
- Expected: Mention Kubernetes restart policy, look at event history, check for OOMKilled status

**Q2: How do you prevent this from reaching production?**
- Expected: Load testing, heap monitoring in staging, automated memory profiling in CI/CD

**Q3: What if the leak is in a system library, not your code?**
- Expected: Discuss patching, vendor updates, workarounds with resource limits

### Red Flags

❌ "The memory usage is fine, it's just Kubernetes overhead"
- Shows ignorance of memory metrics

❌ "I'll increase the memory limit so pods don't die"
- Masking the leak, not fixing it

❌ "Java garbage collection causes this"
- Every garbage collector doesn't cause monotonic growth

❌ "I'll just assume the framework has a leak"
- No evidence gathering

### Pro Tips

✅ "I'd compare memory growth across pod age: if new pods have same pattern, it's a leak"

✅ "I'd use `kubectl debug` to inspect the node's container runtime stats"

✅ "I'd set up continuous memory profiling with Prometheus + automated alerts for growth rate"

✅ "I'd test the fix in staging with realistic load before deploying"

---

## Scenario 2: NetworkPolicy Broke Your Microservices

### The Problem
You deployed a NetworkPolicy to restrict traffic between namespaces. Suddenly, 30% of your services are failing with connection timeouts. The errors are: "mongo-db-service:27017: connection refused."

Your services are in:
- `apps` namespace (frontend, backend)
- `database` namespace (mongodb, redis)
- `monitoring` namespace (prometheus)

**How do you debug and fix without disabling the entire NetworkPolicy?**

### Expected Answer (DETAILED)

#### Step 1: Immediate Diagnosis
```bash
# Check if DNS works:
$ kubectl run debug --image=busybox -it --restart=Never -- \
  nslookup mongo-db-service.database.svc.cluster.local

# Check if connectivity works:
$ kubectl run debug --image=busybox -it --restart=Never -n apps -- \
  wget -O- http://mongo-db-service.database.svc.cluster.local:27017

# List all NetworkPolicies:
$ kubectl get networkpolicy -A

# Describe the restrictive policy:
$ kubectl describe networkpolicy <policy-name> -n apps
```

#### Step 2: Understand the Policy Being Blocked
```yaml
# Current problematic policy might look like:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: apps
spec:
  podSelector: {}  # Applies to ALL pods in this namespace
  policyTypes:
  - Ingress
  - Egress
  # Effectively blocks ALL outbound traffic

# This is the problem: it blocks ALL egress, including to other namespaces
```

#### Step 3: Check Actual Traffic Flow
```bash
# See which pods are trying to connect to what:
$ kubectl logs -n apps -l app=backend --tail=50 | grep -i "connection\|error\|failed"

# Use Cilium if installed:
$ kubectl get ciliumpolicy -A
$ cilium monitor

# Or check with tcpdump:
$ kubectl exec -it <pod> -n apps -- tcpdump -i eth0 -n 'tcp port 27017'
```

#### Step 4: Determine Required Connections
```
Map out what needs to talk to what:
- Frontend (apps) → Backend (apps)
- Backend (apps) → MongoDB (database)
- Backend (apps) → Redis (database)
- Prometheus (monitoring) → All pods (for scraping)
- CoreDNS (kube-system) → All pods (for DNS)
- Kubelet (kube-system) → All pods (for probes)
```

#### Step 5: Create Targeted NetworkPolicy
```yaml
# CORRECT: Allow specific traffic, deny rest
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: mongodb
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: apps
      podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 27017

---
# Allow DNS (critical!)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: apps
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# Allow egress to database namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-to-database
  namespace: apps
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
    ports:
    - protocol: TCP
      port: 27017
    - protocol: TCP
      port: 6379

---
# Allow Prometheus to scrape metrics (ingress on monitored pods)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: apps
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090
```

#### Step 6: Apply and Validate
```bash
# Backup current policies:
$ kubectl get networkpolicy -A -o yaml > networkpolicies-backup.yaml

# Apply fixes:
$ kubectl apply -f corrected-networkpolicies.yaml

# Test connectivity:
$ kubectl run debug --image=busybox -it --rm -n apps -- \
  wget -O- http://mongo-db-service.database.svc.cluster.local:27017 --timeout=5

# Check pod connectivity:
$ kubectl exec -it <backend-pod> -n apps -- \
  nc -zv mongo-db-service.database.svc.cluster.local 27017

# Verify with logs:
$ kubectl logs -n apps -l app=backend --tail=20
```

### Follow-up Questions

**Q1: How do you test NetworkPolicy changes safely?**
- Expected: Staging cluster, canary rollout, monitoring during rollout

**Q2: What if you need cross-namespace communication from monitoring?**
- Expected: Use namespace selectors with labels, service mesh alternatives

**Q3: How do you document this for the team?**
- Expected: Comments in YAML, diagram of allowed traffic, runbook

### Red Flags

❌ "I'll just delete all NetworkPolicies to make it work"
- Security risk, no learning

❌ "NetworkPolicy policing is for security teams"
- You should understand network traffic flow

❌ "I'll use a service mesh to bypass NetworkPolicy"
- Service mesh is for traffic management, not avoiding policies

### Pro Tips

✅ "I'd label namespaces (`kubectl label namespace database policy=restricted`) to make policies more maintainable"

✅ "I'd implement a policy audit mode first to see what would break"

✅ "I'd use network policy visualization tools to map traffic before applying"

✅ "I'd ensure DNS and kubelet ports are always allowed to prevent cluster degradation"

---

## Scenario 3: PVC Stuck in Pending Forever

### The Problem
You deployed a database pod that claims a 500GB PVC. It's been pending for 4 hours. Kubectl describe shows:

```
Events:
  Warning  ProvisioningFailed  2m ago    persistentvolumeclaim-controller
  Failed to provision volume: rpc error: code = ResourceExhausted
```

Meanwhile, your other PVCs (1GB to 100GB) all claim successfully.

**What's happening and how do you fix it?**

### Expected Answer (DETAILED)

#### Step 1: Gather Information
```bash
# Check PVC status in detail:
$ kubectl describe pvc large-db-pvc
$ kubectl get pvc -o wide
$ kubectl get pv -o wide

# Check storage class details:
$ kubectl get storageclass -o yaml
$ kubectl describe storageclass default

# Check current disk usage:
$ kubectl get nodes -o json | jq '.items[] | {
  name: .metadata.name,
  allocatable: .status.allocatable.storage,
  capacity: .status.capacity.storage
}'
```

#### Step 2: Identify the Root Cause
```
Possible causes for 500GB failure:

1. Storage quota exceeded
   - Check GCP project quota: gcloud compute project-info describe --project=PROJECT_ID | grep quotas
   - Check persistent disk quota limit
   
2. Zone has no available storage
   - gcloud compute disks list --project=PROJECT_ID
   - Check in different zones

3. StorageClass not properly configured
   - Check StorageClass type: fast, standard, regional?
   - Check if regional availability

4. Insufficient cluster resources
   - Nodes can't handle the PVC binding
   - Check node statuses: kubectl get nodes
   
5. Storage provisioner is stuck/missing
   - kubectl get pods -n kube-system | grep provisioner
   - kubectl logs <provisioner-pod> -n kube-system | tail -50
```

#### Step 3: Check Storage Quota
```bash
# For GCP/GKE:
$ gcloud compute project-info describe --project=PROJECT_ID | \
  grep -A5 "PERSISTENT_DISKS"

# For AWS/EKS:
$ aws ec2 describe-account-attributes --attribute-names max-elastic-ips

# View StorageClass provisioner:
$ kubectl get storageclass standard -o yaml | grep provisioner

# Check provisioner pod logs:
$ kubectl logs -n kube-system deployment/ebs-csi-controller -c provisioner | tail -100
```

#### Step 4: Common Fixes
```bash
# Fix 1: Use a smaller disk size
$ kubectl patch pvc large-db-pvc -p '{
  "spec": {
    "resources": {
      "requests": {
        "storage": "250Gi"
      }
    }
  }
}'

# Fix 2: Use a different StorageClass (faster, different zone)
$ kubectl get storageclass

# Create new SC for large volumes:
$ cat << 'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-large-volumes
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional
  zones: us-central1-a,us-central1-b
allowVolumeExpansion: true
EOF

# Fix 3: Use a different AZ/region if available
$ cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: large-db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-large-volumes
  resources:
    requests:
      storage: 500Gi
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - us-central1-a
EOF

# Fix 4: Increase project quota (if quota is the issue)
# Contact cloud provider support or use Cloud Console
```

#### Step 5: Verify Provisioner Health
```bash
# Check if provisioner is running:
$ kubectl get pods -n kube-system | grep -i provisioner
$ kubectl logs -n kube-system <provisioner-pod> | grep -i error | head -20

# Check if provisioner has RBAC access:
$ kubectl get clusterrole | grep provisioner
$ kubectl get rolebindings -A | grep provisioner

# Restart provisioner if stuck:
$ kubectl rollout restart deployment <provisioner> -n kube-system
$ kubectl rollout status deployment <provisioner> -n kube-system
```

#### Step 6: Implement Best Practices Going Forward
```yaml
# Use storage quotas to prevent runaway claims:
apiVersion: v1
kind: Namespace
metadata:
  name: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: production
spec:
  hard:
    requests.storage: "2Ti"
    persistentvolumeclaims: "50"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high", "medium"]

---
# LimitRange for PVC sizes:
apiVersion: v1
kind: LimitRange
metadata:
  name: pvc-limits
  namespace: production
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 1Ti
    min:
      storage: 1Gi
```

### Follow-up Questions

**Q1: How do you prevent large PVC claims from failing?**
- Expected: Capacity planning, quota enforcement, progressive allocation

**Q2: What's the difference between provisioning failure and binding failure?**
- Expected: Provisioning = disk creation, Binding = attaching to node

**Q3: Can you use regional PDs instead of zonal?**
- Expected: Yes, but more expensive, better for HA

### Red Flags

❌ "Just increase the storage quota limit"
- Doesn't solve the underlying problem

❌ "Delete other PVCs to free up space"
- Shows no understanding of how provisioning works

❌ "The cloud provider is broken"
- Always exhaust your own options first

### Pro Tips

✅ "I'd check the provisioner pod logs BEFORE trying anything else"

✅ "I'd implement ResourceQuota to catch misconfigured PVC claims early"

✅ "I'd use regional PDs for critical workloads to ensure HA"

✅ "I'd monitor PVC usage with Prometheus alerts for 80% capacity"

