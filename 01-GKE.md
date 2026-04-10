# Google Kubernetes Engine (GKE)

## 1. Core Concept (Deep Explanation)

GKE is Google Cloud's managed Kubernetes service that abstracts away the control plane management while letting you manage worker nodes and workloads. Unlike self-managed Kubernetes, GKE handles:

- **Control Plane Management**: Master nodes, etcd database, API servers, scheduler, controller managers are fully managed by Google
- **Node Pool Architecture**: Flexible node pools with different machine types, custom configurations, and autoscaling policies
- **Workload Identity**: Native integration between Kubernetes service accounts and Google Cloud service accounts
- **Istio Integration**: Built-in service mesh support with Anthos service mesh for advanced traffic management

**Internal Working:**
- GKE uses Google's Compute Engine as the underlying infrastructure
- The control plane runs in a separate Google-managed project, isolated from your nodes
- Communication happens through secure TLS channels
- etcd stores cluster state with automatic backups and point-in-time recovery

## 2. Why This Exists

**Problems it solves:**
- **No Control Plane Overhead**: DevOps teams don't manage Kubernetes masters, etcd, API servers
- **Automatic Patching**: Control plane and nodes are patched automatically without manual intervention
- **Compliance Ready**: Built-in support for workload identity, pod network policies, binary authorization
- **Multi-Tenancy**: VPC-native networking makes isolation easier
- **Google Cloud Integration**: Native IAM, Cloud Logging, Cloud Monitoring, GCS, BigQuery integration

## 3. When To Use

**Best scenarios:**
- Containerized microservices requiring container orchestration
- Applications needing auto-scaling (both vertical and horizontal)
- Organizations requiring audit trails and compliance (HIPAA, PCI-DSS, SOC2)
- Multi-region/multi-zone deployments needing unified management
- Applications with complex networking requirements (Istio, Network Policies)
- Teams using GitOps for deployment automation
- Cost-optimized deployments using Preemptible and Spot VMs

## 4. When NOT To Use

**Avoid if:**
- Simple stateless web applications (use Cloud Run instead)
- Minimal DevOps expertise available (higher operational complexity)
- Need for very tight control over Kubernetes versions (use self-managed if version pinning is critical)
- Single-region, single-zone deployment with no scalability needs
- Legacy monolithic applications not containerized
- Extremely cost-sensitive with predictable traffic (Compute Engine more cost-effective)

**Tradeoffs:**
- More expensive than Compute Engine for simple workloads
- Requires Kubernetes knowledge (steeper learning curve)
- Container image management overhead
- Network overhead from service mesh (if using Istio)

## 5. Real-World Example: Production Microservices Platform

```yaml
# Production GKE Cluster Setup
gke-prod-cluster:
  name: prod-api-cluster
  zone: us-central1-a
  machine_type: n2-standard-4
  node_pools:
    - name: api-nodes
      autoscaling: 
        min: 3
        max: 50
      machine_type: n2-standard-4
      preemptible: false
    - name: batch-jobs
      autoscaling:
        min: 0
        max: 100
      machine_type: n2-highmem-8
      preemptible: true
      taints:
        - key: workload
          value: batch
          effect: NoSchedule
    - name: gpu-ml-nodes
      machine_type: n1-standard-8
      accelerators:
        gpu: nvidia-tesla-v100
        count: 2

networking:
  vpc_native: true
  secondary_ip_range: "pods-secondary-range"
  enable_network_policy: true
  enable_workload_identity: true

security:
  enable_rbac: true
  enable_pod_security_policy: true
  enable_shielded_nodes: true
  enable_binary_authorization: true
```

**Deployment Architecture:**
```
Internet Gateway
    ↓
Cloud Load Balancer (external)
    ↓
Ingress (internal GCP LB)
    ↓
Service (ClusterIP)
    ↓
Pods (across multiple node pools)
    ↓
Cloud Storage / Cloud SQL (external services)
```

## 6. Architecture Thinking

**How GKE Fits in a Production System:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
│  (Microservices deployed in GKE pods)                       │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│         GKE Cluster (Managed Kubernetes)                    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ Control Plane (Google Managed)                      │  │
│  │ - API Server, Scheduler, etcd                       │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌────────────────┬────────────────┬────────────────┐     │
│  │  Node Pool 1   │  Node Pool 2   │  Node Pool 3   │     │
│  │  (API Nodes)   │  (Batch Nodes) │  (GPU Nodes)   │     │
│  │  3-50 nodes    │  0-100 nodes   │  Preemptible   │     │
│  └────────────────┴────────────────┴────────────────┘     │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ Service Mesh (Istio) - Optional                    │  │
│  │ Traffic management, mTLS, observability            │  │
│  └─────────────────────────────────────────────────────┘  │
└──────────────┬──────────────────────────────────────────────┘
               │
    ┌──────────┴──────────┬──────────────┬──────────────┐
    ▼                     ▼              ▼              ▼
Cloud Storage      Cloud SQL        Firestore      Pub/Sub
Secret Manager     Redis (Memcached)  BigTable      Cloud Logging
                                                     Cloud Monitoring
```

**Critical Design Patterns:**
1. **Node Pool Separation**: Different workloads (API, batch, ML) in isolated node pools with taints/tolerations
2. **Workload Identity**: Pods authenticate to GCP services using Kubernetes service accounts
3. **Network Segmentation**: Network policies restrict traffic between namespaces
4. **Resource Limits**: Set CPU/memory requests and limits to prevent node resource exhaustion

## 7. Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| No resource requests/limits | Pod eviction, node pressure, OOMKill | Always set requests and limits based on load testing |
| Single node pool for all workloads | Resource contention, scheduling failures | Use multiple node pools with taints for workload isolation |
| Insufficient RBAC | Security risk, privilege escalation | Use least-privilege RBAC, service accounts per workload |
| No pod disruption budgets | Downtime during cluster upgrades | Define PodDisruptionBudgets for critical workloads |
| Running as root in containers | Container escape risk, compliance issues | Use non-root service accounts in Dockerfile |
| No network policies | Uncontrolled traffic between pods | Implement deny-all policies, whitelist required traffic |
| Not using node affinity | Uneven workload distribution | Use node affinity and pod anti-affinity for high availability |
| Storing secrets in ConfigMaps | Credential exposure risk | Use Secret Manager with Workload Identity |
| Not enabling workload identity | Service account key management nightmare | Always enable and use Workload Identity for pod authentication |
| Cluster autoscaling set too aggressive | Node thrashing, unstable deployments | Set reasonable thresholds (e.g., scale down after 10 minutes) |

## 8. Interview Questions (with Answers)

### Q1: Explain the difference between GKE and Compute Engine. When would you choose each?

**Answer:**
GKE is container orchestration (managed Kubernetes) while Compute Engine is IaaS VMs.
- **Choose GKE** for: Microservices, auto-scaling workloads, complex networking, CI/CD pipelines, applications needing rolling updates
- **Choose Compute Engine** for: Monolithic apps, legacy software, predictable traffic, cost-sensitive simple applications, full VM control needed

**Follow-up consideration:** GKE has ~10% overhead vs Compute Engine, so for simple workloads, Compute Engine is cheaper. For microservices needing orchestration, GKE saves operational overhead despite the cost.

### Q2: How does Workload Identity work and why is it important?

**Answer:**
Workload Identity maps Kubernetes service accounts to GCP service accounts. Instead of using service account keys (security risk):
1. Pod uses Kubernetes service account
2. GKE control plane intercepts access to GCP metadata service
3. Control plane exchanges the token for a short-lived GCP access token
4. Pod authenticates to GCP services (Cloud Storage, BigQuery) using the mapped GCP service account

**Why important:** No key management, automatic token rotation, audit trail through GCP IAM, compliance-friendly.

**Implementation:**
```bash
# Bind KSA to GSA
gcloud iam service-accounts add-iam-policy-binding GSA_NAME@PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]"

# Annotate KSA
kubectl annotate serviceaccount KSA_NAME \
  iam.gke.io/gcp-service-account=GSA_NAME@PROJECT_ID.iam.gserviceaccount.com
```

### Q3: You have a GKE cluster with 100 applications. One batch job is consuming all memory and crashing other pods. How do you prevent this?

**Answer:**
Multiple layers:
1. **Resource Requests/Limits**: Set memory requests and limits on deployments so scheduler knows capacity
   ```yaml
   resources:
     requests:
       memory: "128Mi"
       cpu: "100m"
     limits:
       memory: "256Mi"
       cpu: "500m"
   ```
2. **Node Pool Separation**: Move batch jobs to separate node pool with higher spec
3. **PodDisruptionBudget**: Prevent evictions of critical pods
4. **QoS Classes**: Use Guaranteed (requests=limits) for critical workloads
5. **LimitRange**: Set default limits per namespace
   ```yaml
   apiVersion: v1
   kind: LimitRange
   metadata:
     name: default-limits
   spec:
     limits:
     - default:
         memory: "256Mi"
         cpu: "100m"
       defaultRequest:
         memory: "128Mi"
         cpu: "50m"
       type: Container
   ```

### Q4: Your cluster is in us-central1-a, but you need multi-region redundancy. What's your strategy?

**Answer:**
Design multi-region GKE deployment:
1. **Regional Clusters**: Create GKE clusters in different regions (us-central1, us-east1, europe-west1)
2. **Istio Multi-Cluster**: Use Anthos service mesh for cross-cluster traffic management
3. **Cloud Load Balancer**: Frontend for routing traffic across regions
4. **Firestore/Cloud SQL**: Multi-region replication for data
5. **GitOps**: CI/CD pipeline deploys same manifests to all clusters

```
Traffic
  ↓
Cloud CDN (cache layer)
  ↓
Cloud Load Balancer
  ├─→ GKE Cluster (us-central1)
  ├─→ GKE Cluster (us-east1)
  └─→ GKE Cluster (europe-west1)
      ↓
   Multi-region database
```

### Q5: How does GKE cluster autoscaling differ from pod horizontal autoscaling (HPA)?

**Answer:**
- **Cluster Autoscaling**: Scales *nodes* based on resource requests. If pod can't schedule (no free CPU/memory), add nodes.
  - Based on: Node resource requests
  - Typical trigger: Pod pending for >10 seconds
  - Slow: Node creation takes 1-3 minutes

- **HPA**: Scales *pods* (replicas) based on metrics (CPU, memory, custom).
  - Based on: Observed metrics from Metrics Server
  - Typical trigger: CPU > 80% for 1 minute
  - Fast: New pod ready in seconds

**Interaction:**
```
Increased Traffic
    ↓
HPA detects CPU spike
    ↓
HPA creates new pods (fast)
    ↓
If no node capacity, cluster autoscaler adds nodes (slow)
```

**Best practice:** Configure both. HPA reacts to traffic changes; cluster autoscaler ensures capacity.

### Q6: Explain network policies in GKE. Write a simple policy to allow only traffic from pods labeled "frontend" to "backend" pods.

**Answer:**
Network policies are Kubernetes firewall rules at the pod level (not VM). Default is allow-all; you can restrict to deny-all then whitelist.

```yaml
# Step 1: Deny all traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Step 2: Allow ingress from frontend to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# Step 3: Allow backend to reach external databases
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: databases
    ports:
    - protocol: TCP
      port: 5432
```

**To enable:** GKE requires VPC-native cluster with network policy enabled:
```bash
gcloud container clusters create my-cluster --enable-network-policy
```

### Q7: Your GKE cluster experienced a control plane outage for 30 minutes. Worker nodes were fine. What failed and what was unavailable?

**Answer:**
Control plane outage means API server is down. Impact:

**What failed:**
- kubectl commands (no API communication)
- New deployments/updates
- Pod scheduling
- Configuration changes

**What continued working:**
- Existing pods kept running
- Services and Endpoints unchanged
- Traffic between pods flowed normally
- Node-local storage accessible

**Why:** Control plane manages cluster state but existing workloads run on nodes independently. Worker nodes can operate without master for hours; only scheduling and updates fail.

**Prevention:**
- Use GKE's high-availability control plane (handles failure automatically)
- Design applications for fault tolerance
- Monitor control plane health with Cloud Monitoring

### Q8: You need to roll out a canary deployment for a critical service. Design the strategy.

**Answer:**
```yaml
# Canary Deployment: 95% old version, 5% new version
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080

---
# Current production (95% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-stable
spec:
  replicas: 19  # 95% of 20
  selector:
    matchLabels:
      app: api
      version: stable
  template:
    metadata:
      labels:
        app: api
        version: stable
    spec:
      containers:
      - name: api
        image: gcr.io/project/api:v1.2.3

---
# Canary (5% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-canary
spec:
  replicas: 1  # 5% of 20
  selector:
    matchLabels:
      app: api
      version: canary
  template:
    metadata:
      labels:
        app: api
        version: canary
    spec:
      containers:
      - name: api
        image: gcr.io/project/api:v1.2.4
```

**Monitoring during canary:**
- Error rate increase in canary vs stable
- Latency increase
- Business metrics (conversion rate, revenue)

**If canary succeeds:**
1. Scale canary to 10 replicas (50%)
2. Scale canary to 20 replicas (100%)
3. Remove stable deployment

**If canary fails:**
1. Scale canary to 0 replicas
2. Root cause analysis
3. Fix and retry

**Better approach with Istio:**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: api
spec:
  hosts:
  - api
  http:
  - match:
    - uri:
        prefix: "/"
    route:
    - destination:
        host: api
        subset: stable
      weight: 95
    - destination:
        host: api
        subset: canary
      weight: 5
```

### Q9: Explain how GKE backup and disaster recovery works. What happens to persistent data?

**Answer:**
GKE backup uses GKE Backup (Backup for GKE):
- Backs up applications, namespaces, persistent volumes
- Uses storage backend (Cloud Storage)
- Point-in-time recovery capability

**Coverage:**
- ✓ Deployments, StatefulSets, RBAC, ConfigMaps, Secrets
- ✓ PersistentVolumes (GPD, Filestore)
- ✗ Control plane (managed by Google)
- ✗ Node data (irrelevant; nodes are cattle not pets)

**Restore process:**
```bash
# Create backup
gcloud container backup-restore backups create my-backup \
  --cluster=projects/PROJECT_ID/locations/us-central1/clusters/my-cluster

# Restore backup
gcloud container backup-restore restores create my-restore \
  --backup=projects/PROJECT_ID/locations/us-central1/backupPlans/my-plan/backups/my-backup \
  --cluster=projects/PROJECT_ID/locations/us-central1/clusters/restore-cluster
```

**For persistent data:**
- GKE backs up PersistentVolume specifications and data
- Cloud SQL backups independent of GKE
- Cloud Storage data in snapshots

### Q10: Your organization requires Pod Security Policy (PSP) to prevent privileged containers. Write a PSP that forbids root access and privileged escalation.

**Answer:**
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'runtime/default'
    apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'runtime/default'
    apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
  - ALL
  allowedCapabilities: []
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
    seLinuxOptions:
      level: "s0:c123,c456"
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
    - min: 1
      max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
    - min: 1
      max: 65535
  readOnlyRootFilesystem: false
```

**Apply the PSP:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: restricted-psp-user
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames:
  - restricted

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: restricted-psp-all-serviceaccounts
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: restricted-psp-user
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```

---

## 9. Advanced Insights (Senior-level)

### Cost Optimization

**Preemptible VMs for Non-Critical Workloads:**
- 60-80% cost saving vs on-demand
- 24-hour max lifetime, subject to interruption
- Ideal for batch jobs, ML training, CI/CD agents
- Not suitable for stateful workloads

**Node Pool Right-Sizing:**
```bash
# Analyze actual vs requested resources
kubectl top nodes
kubectl top pods -A

# Right-size: if 20% utilization, reduce machine type
gcloud container node-pools update batch-pool \
  --cluster my-cluster \
  --machine-type n2-standard-2 \
  --zone us-central1-a
```

**Committed Use Discounts (CUDs):**
- 25-30% discount for 1-year commitment
- 40-50% discount for 3-year commitment
- Commitment applies across regions
- Purchase when you have predictable baseline

### Performance Optimization

**Pod Density:**
- Small machines (n1-standard-1) ~30-40 pods
- Large machines (n2-standard-64) ~100+ pods
- Account for system pods (kube-proxy, kube-dns, container runtime)

**Network Performance:**
- Use GKE Dataplane V2 for better performance (VPC-native + eBPF)
- Enable network policy without performance overhead
- Use Maglev for load balancing (better connection establishment)

**Storage Performance:**
- Persistent Disk SSD (pd-ssd) for databases
- Balanced Disk (pd-balanced) for general purpose
- Regional persistent disks for HA across zones

### Security Hardening

**Defense in Depth:**
```
┌─────────────────────────────────────┐
│ Binary Authorization (image signing) │
├─────────────────────────────────────┤
│ Workload Identity (no VM keys)      │
├─────────────────────────────────────┤
│ Network Policies (pod firewall)     │
├─────────────────────────────────────┤
│ Pod Security Policy (container rules)│
├─────────────────────────────────────┤
│ RBAC (fine-grained access)          │
├─────────────────────────────────────┤
│ Secret Manager (no plaintext)       │
└─────────────────────────────────────┘
```

**Enable all:**
```bash
gcloud container clusters create prod-cluster \
  --enable-network-policy \
  --enable-workload-identity \
  --enable-shielded-nodes \
  --enable-binary-authorization \
  --enable-ip-alias \
  --enable-stackdriver-kubernetes
```

### Observability at Scale

**Three Pillars:**
1. **Metrics**: CPU, memory, network from Cloud Monitoring
2. **Logs**: Application & system logs to Cloud Logging
3. **Traces**: Distributed tracing with Cloud Trace + Istio

**Key dashboards:**
- Node utilization (CPU, memory, disk)
- Pod CPU throttling
- Network bandwidth utilization
- Persistent disk IOPS

**Alert strategy:**
- Control plane latency > 1s (indicates issues)
- Pod eviction rate > 0 (resource pressure)
- Node NotReady > 5 minutes (infrastructure issue)
- PVC usage > 80% (storage pressure)

