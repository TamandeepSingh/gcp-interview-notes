# Kubernetes Fundamentals

## 1. Core Concept

**Kubernetes (K8s)** is an **orchestration platform** that automates deployment, scaling, and management of containerized applications across clusters of machines. Instead of manually managing containers, you declare desired state (how many replicas, resources needed, network config) and Kubernetes figures out how to achieve and maintain it.

**Core value propositions:**
- **Container orchestration:** Deploy containers reliably across multiple machines
- **Self-healing:** Restarts failed containers, reschedules pods on failed nodes
- **Scaling:** Automatically scale replicas based on demand
- **Resource management:** Efficiently utilizes CPU/memory across cluster
- **Service discovery:** Applications find each other through DNS
- **Rolling updates:** Deploy new versions without downtime
- **Configuration management:** Decouple configuration from container images

---

## 2. Internal Working (VERY IMPORTANT)

### Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│            Kubernetes Cluster                        │
├─────────────────────────────────────────────────────┤
│                     CONTROL PLANE                    │
│  ┌────────────────┐  ┌──────────────┐  ┌──────────┐│
│  │ API Server     │  │ etcd         │  │Scheduler ││
│  │(REST API)      │  │(state store) │  │(assigns) ││
│  └────────────────┘  └──────────────┘  └──────────┘│
│  ┌────────────────────────────────────────────────┐ │
│  │Controller Manager (reconciliation loop)         │ │
│  │- Deployment controller                         │ │
│  │- StatefulSet controller                        │ │
│  │- DaemonSet controller                          │ │
│  └────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────┤
│                    WORKER NODES                      │
│  ┌──────────────────┐  ┌──────────────────┐       │
│  │ Node 1           │  │ Node 2           │  ...  │
│  ├──────────────────┤  ├──────────────────┤       │
│  │ kubelet          │  │ kubelet          │       │
│  │(node agent)      │  │(node agent)      │       │
│  │                  │  │                  │       │
│  │ ┌──────────────┐ │  │ ┌──────────────┐ │       │
│  │ │ Pod 1  │ Pod2│ │  │ │ Pod 3        │ │       │
│  │ │ (app)  │     │ │  │ │ (app)        │ │       │
│  │ └──────────────┘ │  │ └──────────────┘ │       │
│  │                  │  │                  │       │
│  │ kube-proxy       │  │ kube-proxy       │       │
│  │ (networking)     │  │ (networking)     │       │
│  └──────────────────┘  └──────────────────┘       │
└─────────────────────────────────────────────────────┘
```

### Request Flow: Creating a Deployment

```
1. User: kubectl apply -f deployment.yaml
   ↓ (HTTPS to API Server)

2. API Server receives request
   - Validates YAML syntax
   - Stores in etcd (state database)
   - Returns success to user
   ↓

3. Deployment Controller (running on control plane)
   - Watches etcd for new Deployments
   - Detects new deployment.yaml
   - Creates ReplicaSet (desired state: 3 replicas)
   ↓

4. ReplicaSet Controller
   - Watches for ReplicaSets
   - Detects need for 3 Pods
   - Creates 3 Pod definitions in etcd
   ↓

5. Scheduler
   - Watches API for unscheduled Pods
   - Gets node list with available resources
   - Algorithm: Selects Node 1 for Pod 1, Node 2 for Pod 2, etc
   - Updates Pod: "scheduledNode = Node 1"
   ↓

6. Kubelet on Node 1
   - Watches API for Pods assigned to it
   - Detects new Pod assignment
   - Calls container runtime (Docker/containerd)
   - "Pull image, create container, start container"
   - Reports status back to API Server
   ↓

7. API Server updates etad with status
   - Pod status: Running
   - User: kubectl get pods → Shows "Running"
```

### Controller Reconciliation Loop (Core of K8s)

```python
# Pseudocode of what controllers do
while true:
  desired_state = read_from_etcd()  # What user declared
  
  for resource in desired_state:
    current_state = read_from_cluster()  # What actually exists
    
    if current_state != desired_state:
      # Take corrective action
      if desired_replicas > current_replicas:
        create_new_pods()
      elif desired_replicas < current_replicas:
        delete_pods()
      elif pod_failed:
        restart_pod()
      elif node_failed:
        reschedule_pods()
      
    # Update status in etcd
    update_resource_status(resource)
  
  time.sleep(reconciliation_interval)  # ~10-30 seconds
```

**Key insight:** Kubernetes constantly compares desired vs actual, making corrections.

### Pod Lifecycle

```
┌──────────────────────────────────────────────────────┐
│                   POD LIFECYCLE                       │
├──────────────────────────────────────────────────────┤
│                                                       │
│  Pending ──> ContainerCreating ──> Running ──> Succeeded/Failed
│                                                       │
│  Pending: Pod created, waiting for container runtime │
│  ContainerCreating: Image pulling, container starting│
│  Running: Container executing                        │
│  Succeeded: Container exited 0 (normal)              │
│  Failed: Container exited non-zero                   │
│                                                       │
└──────────────────────────────────────────────────────┘

Phase transitions happen in etcd
Kubelet updates status continuously
Controllers watch for state changes, take action
```

---

## 3. Why Kubernetes is Needed

### Without Container Orchestration

**Scenario: Deploy 10 replicas of web app across 5 machines**

Manual approach:
```
1. SSH to machine1: docker run web:v1 -p 8080:8080
2. SSH to machine2: docker run web:v1 -p 8080:8080
   ... (repeat for all 5 machines)
3. Machine2 fails → oops, 2 replicas gone
   SSH to machine3, manually start container
4. New version released → update 10 containers manually
5. Need to scale to 20 replicas → manual SSH to 10 more machines
6. Developer leaves → no one knows what's actually running
```

Problems:
- Not reproducible
- No auto-healing
- Scaling is painful
- No configuration management
- Manual operational burden
- No audit trail

### What Kubernetes Enables

✅ **Single declaration of desired state:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 10
  template:
    spec:
      containers:
      - image: web:v1
```

✅ **Automatic execution:**
- Kubernetes creates 10 replicas
- Distributes across available nodes
- Auto-heals failed replicas
- Scales replicas by changing replicas: 10 → 20
- Rolls out new versions without downtime

✅ **Self-healing:**
- Pod crashes → restarted automatically
- Node fails → Pods rescheduled to other nodes
- Resource exhausted → Pods evicted

✅ **Operational clarity:**
- kubectl get pods shows exact state
- Git history shows infrastructure changes
- Audit logs show who made changes

---

## 4. When to Use vs Not Use Kubernetes

### Use Kubernetes When:

- ✅ Production microservices architecture
- ✅ Multiple services needing orchestration
- ✅ Need for scaling and high availability
- ✅ Team size justifies operational complexity
- ✅ Planned container workload lifetime: months+

### Don't Use Kubernetes When:

- ❌ Simple monolithic application (runs fine on VMs)
  - *Use: Docker + simple orchestration (Nomad, Docker Swarm)*
- ❌ Single-container applications
  - *Use: Cloud Run, Lambda, bare containers*
- ❌ Batch/cron jobs only
  - *Use: Cloud Functions, cron on VMs*
- ❌ Team very small, no DevOps expertise
  - *Use: Managed services (Cloud SQL), simpler deployment*
- ❌ No need for horizontal scaling
  - *Use: Just run on VMs, simpler to manage*

### Managed Kubernetes (Recommended for most)

Instead of self-hosted Kubernetes:
- Use GKE (Google), EKS (AWS), AKS (Azure)
- They manage control plane
- You manage node pools and workloads
- Simpler operations, less maintenance

---

## 5. Real-world DevOps Usage

### Production GKE Cluster Setup

**Typical enterprise deployment:**

```yaml
# gke-cluster.yaml
apiVersion: container.cnrm.cloud.google.com/v1beta1
kind: ContainerCluster
metadata:
  name: prod-gke
spec:
  # Network settings
  network: projects/prod-project/global/networks/prod-vpc
  subnetwork: prod-subnet
  clusterSecondaryRangeSpec:
    - rangeName: pods
      cidrBlock: 10.4.0.0/14
    - rangeName: services
      cidrBlock: 10.0.0.0/20
  
  # Workload Identity (secure pod-to-GCP auth)
  workloadIdentityConfig:
    workloadPool: prod-project.svc.id.goog
  
  # Security best practices
  masterAuth:
    clientCertificateConfig:
      issueClientCertificate: false
  
  # Logging and monitoring
  loggingService: logging.googleapis.com/kubernetes
  monitoringService: monitoring.googleapis.com/kubernetes
  
  # Cluster config
  initialNodeCount: 1
  nodeConfig:
    preemptible: false
    machineType: e2-standard-4
    
    # Security
    oauthScopes:
    - https://www.googleapis.com/auth/cloud-platform
    
    labels:
      environment: production
  
  # Maintenance
  maintenancePolicy:
    dailyMaintenanceWindow:
      startTime: "03:00"
      duration: 4h
```

**Applications deployed to cluster:**

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        version: v2
    spec:
      serviceAccountName: web-app-sa  # Workload Identity
      
      containers:
      - name: app
        image: gcr.io/prod-project/web-app:v2.1.0
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        
        # Config from ConfigMap
        env:
          - name: LOG_LEVEL
            valueFrom:
              configMapKeyRef:
                name: app-config
                key: log_level
          
          # Secrets
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: db-creds
                key: password
        
        # Volume mounts
        volumeMounts:
        - name: config
          mountPath: /etc/config
          readOnly: true
      
      # Pod disruption budget (prevents accidental deletion)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web
              topologyKey: kubernetes.io/hostname
      
      volumes:
      - name: config
        configMap:
          name: app-config

---
# Service (expose app)
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: production
spec:
  selector:
    app: web
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080

---
# HorizontalPodAutoscaler (auto-scale)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**Real operations:**

```bash
# Deploy application
kubectl apply -f deployment.yaml

# Check status
kubectl get deployment web-app
kubectl get pods -l app=web

# Monitor
kubectl logs -f deployment/web-app
kubectl top pods

# Scale manually
kubectl scale deployment web-app --replicas=5

# Rolling update (new version)
kubectl set image deployment/web-app \
  app=gcr.io/prod-project/web-app:v2.2.0

# Check rollout status
kubectl rollout status deployment/web-app

# Rollback if issues
kubectl rollout undo deployment/web-app
```

---

## 6. Architecture Thinking

### Design Principles

**Principle 1: Stateless Applications**
- Applications must run anywhere (any node)
- No local storage (pod can be killed anytime)
- Session state in external database (Redis/Cloud SQL)

**Principle 2: Resource Requests and Limits**
- Scheduler uses requests to bin-pack pods
- Limits protect node from resource exhaustion
- Both required for reliable clusters

**Principle 3: Health Checks**
- liveness = is the app responsive?
- readiness = is the app ready for traffic?
- graceful shutdown = handle SIGTERM

**Principle 4: Pod Anti-Affinity**
- Don't schedule all replicas on one node
- Survive node failures
- Automatic via pod disruption budgets

### Typical Cluster Architecture

```
┌─────────────────────────────────────────────────────┐
│           GKE Cluster (prod-gke)                     │
├─────────────────────────────────────────────────────┤
│                                                       │
│  ┌──────────────────────────────────────┐           │
│  │ kube-system (K8s internals)           │           │
│  │ - kube-dns                           │           │
│  │ - kube-proxy                         │           │
│  │ - metrics-server                     │           │
│  └──────────────────────────────────────┘           │
│                                                       │
│  ┌──────────────────────────────────────┐           │
│  │ monitoring (Prometheus, Grafana)      │           │
│  │ - Collects cluster and pod metrics    │           │
│  └──────────────────────────────────────┘           │
│                                                       │
│  ┌──────────────────────────────────────┐           │
│  │ production namespace                  │           │
│  │                                       │           │
│  │ ┌─────────────────────────────────┐  │           │
│  │ │ web-app Deployment (3 replicas) │  │           │
│  │ │ - Pod 1                         │  │           │
│  │ │ - Pod 2                         │  │           │
│  │ │ - Pod 3                         │  │           │
│  │ └─────────────────────────────────┘  │           │
│  │                                       │           │
│  │ ┌─────────────────────────────────┐  │           │
│  │ │ api-service Deployment (2)      │  │           │
│  │ └─────────────────────────────────┘  │           │
│  │                                       │           │
│  │ ┌─────────────────────────────────┐  │           │
│  │ │ worker Deployment (5)           │  │           │
│  │ └─────────────────────────────────┘  │           │
│  │                                       │           │
│  │ ┌─────────────────────────────────┐  │           │
│  │ │ Services (internal networking)  │  │           │
│  │ │ - web-app-service              │  │           │
│  │ │ - api-service                  │  │           │
│  │ └─────────────────────────────────┘  │           │
│  │                                       │           │
│  │ ┌─────────────────────────────────┐  │           │
│  │ │ Storage (PersistentVolumes)     │  │           │
│  │ │ - database storage              │  │           │
│  │ │ - cache storage                 │  │           │
│  │ └─────────────────────────────────┘  │           │
│  └──────────────────────────────────────┘           │
│                                                       │
│  ┌──────────────────────────────────────┐           │
│  │ ingress (external routing)            │           │
│  │ - Routes traffic to web-app-service   │           │
│  └──────────────────────────────────────┘           │
│                                                       │
└─────────────────────────────────────────────────────┘
```

---

## 7. Common Mistakes

### Mistake 1: Not Setting Resource Requests/Limits

❌ **Wrong:**
```yaml
containers:
- name: app
  image: myapp:v1
  # No requests/limits!
```

**Problem:**
- Scheduler doesn't know how much space needed
- Might schedule too many pods on one node
- Node runs out of memory
- Pod killed (OOMKilled)

✅ **Right:**
```yaml
containers:
- name: app
  image: myapp:v1
  resources:
    requests:  # What pod needs to run
      cpu: 100m
      memory: 128Mi
    limits:    # Maximum allowed
      cpu: 500m
      memory: 512Mi
```

### Mistake 2: Storing State in Pods

❌ **Wrong:**
```python
# App code
with open("/local/data.db", "w") as f:
  f.write(data)

# Pod gets deleted, data lost!
```

✅ **Right:**
```python
# Store in external database
db.save(data)

# Or use PersistentVolume
# But only for stateful services (databases, caches)
```

### Mistake 3: No Health Checks

❌ **Wrong:**
```yaml
containers:
- name: app
  image: myapp:v1
  # No liveness/readiness probes
```

**Problem:**
- Broken app still receives traffic
- Kubernetes doesn't know pod is unhealthy
- Users experience failures

✅ **Right:**
```yaml
containers:
- name: app
  image: myapp:v1
  livenessProbe:
    httpGet:
      path: /health
      port: 8080
    periodSeconds: 10
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    periodSeconds: 5
```

### Mistake 4: Single Replica (No HA)

❌ **Wrong:**
```yaml
spec:
  replicas: 1  # Single point of failure
```

✅ **Right:**
```yaml
spec:
  replicas: 3  # Survive node/pod failures
  
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - web
          topologyKey: kubernetes.io/hostname  # Spread across nodes
```

### Mistake 5: Storing Secrets in ConfigMap

❌ **Wrong:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config
data:
  db_password: "supersecret123"  # Exposed!
```

✅ **Right:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
stringData:
  password: "supersecret123"

# Or use Secret Manager
data:
  password_ref: "projects/prod/secrets/db-password"
```

### Mistake 6: No Resource Quotas

❌ **Problem:**
- One namespace uses all cluster resources
- Other services can't run
- Cluster becomes unavailable

✅ **Right:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"          # Max 10 CPU cores
    requests.memory: "20Gi"     # Max 20GB RAM
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "100"                 # Max 100 pods
```

### Mistake 7: No Pod Disruption Budget

❌ **Problem:**
```bash
kubectl drain node-1
# All 3 replicas on node-1 deleted
# Application down!
```

✅ **Right:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2  # Keep at least 2 running
  selector:
    matchLabels:
      app: web
```

---

## 8. Interview Questions (Scenario-based)

### Q1: Pod Scheduling Deep Dive

**Interviewer:** "Cluster has 3 nodes. You deploy 10 pods but only 8 appear. Where are the other 2? How do you debug?"

**Good Answer:**
```bash
# 1. Check pod status
kubectl get pods
# Status: Pending for 2 pods

# 2. Check why
kubectl describe pod <pending-pod>
# "0/3 nodes are available: 3 Insufficient cpu"

# 3. Resources problem
kubectl top nodes
# Node 1: 800m/1000m CPU (full)
# Node 2: 900m/1000m CPU (full)
# Node 3: 100m/1000m CPU (available, but not enough)

# 4. Solutions:
# - Add node pool
# - Reduce pod resource requests
# - Increase resource limits on cluster

kubectl scale deployment web --replicas=8  # Reduce replicas
# Now all pods schedule
```

### Q2: Node Failure Recovery

**Interviewer:** "A node suddenly fails. You have 3 replicas on it. What happens?"

**Good Answer:**
1. Kubelet stops reporting heartbeat
2. Control plane detects node NotReady
3. After 5 minutes (default), evicts pods
4. Pods rescheduled to other nodes
5. Deployment controller detects missing replicas
6. Creates new pods on available nodes
7. Result: Self-healed, application still runs
8. Total downtime: ~5 minutes worst case

### Q3: Helm vs Raw K8s

**Interviewer:** "Would you deploy directly with kubectl apply or use Helm? Why?"

**Good Answer:**
- **kubectl apply:** Good for learning, small projects
- **Helm:** 
  - Templating (same app, different values)
  - Version management
  - Dependency management (deploy app + supporting services)
  - Rollback capability
  - Better for production

For production: Always use Helm

### Q4: Scaling Strategy

**Interviewer:** "Traffic spikes 10x. Cluster can't handle it. What do you do?"

**Good Answer:**
1. **Immediate (HPA):** Already scaling if configured
   ```yaml
   HorizontalPodAutoscaler:
     maxReplicas: 100
   ```

2. **Cluster scaling:** Add nodes automatically
   ```bash
   # If cluster autoscaling enabled
   # New nodes added automatically
   ```

3. **If scaling not possible:**
   - Circuit breaker (reject some traffic gracefully)
   - Queue traffic (process later)
   - Cache responses (reduce load)

---

## 9. Debugging Techniques

### Debugging Pod Issues

```bash
# Status
kubectl get pods
kubectl describe pod <name>

# Logs
kubectl logs <pod>
kubectl logs <pod> -c <container>  # Specific container
kubectl logs <pod> --previous       # Logs before crash

# Execute command in pod
kubectl exec -it <pod> -- bash

# Port forward for debugging
kubectl port-forward <pod> 8080:8080
# Access localhost:8080

# Resources
kubectl top pod <name>
kubectl top nodes
```

### Common Pod Issues

```
ImagePullBackOff:
  - Image doesn't exist or bad credentials
  - Fix: Check image name, push to registry

CrashLoopBackOff:
  - App crashes immediately
  - Fix: Check logs with kubectl logs

OOMKilled:
  - Pod exceeded memory limit
  - Fix: Increase memory limit

Pending:
  - Can't schedule (not enough resources)
  - Fix: Reduce requests or add nodes
```

---

## 10. Advanced Insights

### Federation (Multi-cluster)

```
┌─────────────────────┐         ┌─────────────────────┐
│  GKE Cluster        │         │  GKE Cluster        │
│ (us-central1)       │         │ (eu-west1)          │
│                     │         │                     │
│ ┌───────────────┐   │         │ ┌───────────────┐   │
│ │ app-service   │   │         │ │ app-service   │   │
│ │ 5 replicas    │   │         │ │ 5 replicas    │   │
│ └───────────────┘   │         │ └───────────────┘   │
└─────────────────────┘         └─────────────────────┘
        ↑                               ↑
        └───────────────┬───────────────┘
                        │
                Global Load Balancer
                (Route based on geo)

Result:
- Survive entire cluster failure
- Low latency (geo-location)
- Global scaling
```

### Cost Optimization

```yaml
# Preemptible nodes (70% cheaper, risky)
nodePool:
  preemptible: true

# Pod Priority Classes
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production
value: 1000
globalDefault: false
---
# Spot instances (even cheaper)
nodePool:
  machineType: n2d-standard-8
  spot: true
```

---

## Key Takeaways

1. **Kubernetes automates container orchestration** through declarative configs
2. **Controllers continuously reconcile** actual vs desired state
3. **Always set resource requests/limits** for proper scheduling
4. **Health checks are mandatory** for reliable applications
5. **Design for failure** - assume nodes and pods will die
6. **Use managed Kubernetes** (GKE) to reduce operational burden
7. **External databases/storage** - don't store state in pods
8. **Monitoring and logging** - visibility is critical
