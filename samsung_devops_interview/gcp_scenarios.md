# GCP Scenarios - Samsung DevOps Interview

## Scenario 1: GKE Cluster Cost Explosion

### The Problem
You discover your GKE cluster bill has tripled overnight. The team deployed a new microservice 4 hours ago. Your quota allows for max 100 nodes, but now you have exactly 100 nodes running with high CPU utilization across all of them. Previous maximum was 25 nodes.

**What's your immediate action plan?**

### Expected Answer (DETAILED)

#### Step 1: Root Cause Analysis (First 10 minutes)
```
1. Check Kubernetes metrics immediately:
   - kubectl top nodes
   - kubectl top pods --all-namespaces --sort-by=memory
   - kubectl get nodes -o wide | grep NotReady

2. Identify the culprit deployment:
   - kubectl get deployment -A --sort-by=.metadata.creationTimestamp
   - kubectl describe pod <problematic-pod>
   - Check resource requests/limits: kubectl get deployment <name> -A -o yaml | grep -i resources
   
3. Check HPA status:
   - kubectl get hpa -A
   - kubectl describe hpa <hpa-name>
   - Look at current replicas vs desired replicas
```

#### Step 2: Deep Dive - Why 100 Nodes?
```
Questions to ask yourself:
- Is there a resource leak? (Pod requests 10Gi memory but only uses 100Mi)
- Is HPA misbehaving? (Is it scaling up indefinitely?)
- Are node pool autoscaling limits reached?
- Did someone change resource requests? (e.g., 10Gi to 100Gi)

Check GKE Workloads:
- Cloud Console → GKE → Workloads → Filter by newest
- Cloud Monitor → Custom metrics for the service
```

#### Step 3: Immediate Mitigation (Emergency Mode)
```
Option A - Quick Kill (If it's clearly a bad deploy):
$ kubectl delete deployment <problematic-deployment> -n <namespace>
$ kubectl delete hpa <hpa-name> -n <namespace>

Option B - Resource Limit (If you need the app):
$ kubectl patch deployment <name> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container>","resources":{"requests":{"memory":"256Mi","cpu":"100m"},"limits":{"memory":"512Mi","cpu":"500m"}}}]}}}}'

Option C - Scale Down Nodes (Manual):
$ gcloud container clusters update <CLUSTER_NAME> --zone <ZONE> --num-nodes 30
$ gcloud container node-pools update <NODE_POOL> --cluster <CLUSTER> --zone <ZONE> --enable-autoscaling --min-nodes 1 --max-nodes 50
```

#### Step 4: Investigation Depth
```
Check the new service's code:
1. Memory leaks in loops?
   kubectl logs <pod-name> -n <namespace> --tail=100

2. Database queries spawning connections?
   kubectl exec <pod> -- netstat -an | grep ESTABLISHED | wc -l

3. Infinite worker threads?
   kubectl exec <pod> -- ps aux | wc -l

4. Container image size?
   docker inspect <image> | jq .Size
```

#### Step 5: Long-term Fix
```
1. Add resource requests/limits to the deployment YAML
2. Implement HPA with proper metrics and thresholds
3. Add Pod Disruption Budgets (PDB)
4. Implement resource quotas per namespace
5. Set up alerts on node count spikes
```

### Follow-up Questions

**Q1: How would you prevent this from happening again?**
- Expected: Talk about admission webhooks, resource quotas, webhook validation, CI/CD checks

**Q2: What if the deployment is critical and can't be deleted?**
- Expected: Discuss cordon/drain, traffic shifting, blue-green deployment, gradual rollback

**Q3: How do you handle the billing implications?**
- Expected: Discuss committed use discounts (CUDs), preemptible nodes, cost anomaly detection alerts

**Q4: What metrics should you monitor?**
- Expected: Node count, pod density per node, CPU/memory utilization, failed pod scheduling, restart counts

### Red Flags (What NOT to Say)

❌ "I would just delete all pods and restart"
- Shows lack of understanding of production impact

❌ "Wait for it to auto-scale down"
- Auto-scaling DOWN takes 10+ minutes; meanwhile costs are accumulating

❌ "I don't know, let me ask the app team"
- You should have diagnostic tools knowledge

❌ "Probably a configuration error, I'll re-deploy everything"
- Shows no systematic debugging approach

❌ "The quota limit is the problem, increase it"
- Treating symptom, not root cause

### Pro Tips (How to Impress)

✅ **Mention GKE-specific monitoring:**
- "I'd check GKE Cost Monitor and look at resource requests vs actual usage to spot misconfigurations"

✅ **Talk about preventive mechanisms:**
- "I'd implement a mutating webhook to set default resource requests if missing"
- "I'd enable ResourceQuota per namespace to prevent runaway scaling"

✅ **Show production mentality:**
- "I'd check Pod events, CrashLoopBackOff patterns, and look at application logs for errors"
- "While investigating, I'd also alert the team and prepare a rollback plan"

✅ **Mention cost optimization:**
- "I'd switch to preemptible nodes for non-critical workloads, use committed discounts for prod"

---

## Scenario 2: GCP IAM Permission Hell

### The Problem
Your CI/CD pipeline is failing with: `Error: Permission denied on gs://prod-bucket/deploy. Caller is user: projects/-/serviceAccounts/builder@myproject.iam.gserviceaccount.com`

The pipeline needs to:
1. Read from GCS bucket (prod-bucket)
2. Deploy to Cloud Run
3. Update Cloud SQL passwords
4. Access Secrets Manager

**How do you minimize permissions (principle of least privilege) while keeping the pipeline working?**

### Expected Answer (DETAILED)

#### Step 1: Current Permission Audit
```bash
# Check current service account permissions:
$ gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccounts/builder@*"

# Check inherited permissions:
$ gcloud resource-manager folders get-iam-policy FOLDER_ID

# Check custom roles:
$ gcloud iam roles list --project=PROJECT_ID --format=table(name)
```

#### Step 2: Map Pipeline Requirements to Exact Permissions
```yaml
Task: Read from GCS bucket
Permission: storage.buckets.get, storage.objects.get
NOT: storage.*.* (too broad)
Role: roles/storage.objectViewer (built-in)
Better: Custom role with scoped permissions

Task: Deploy to Cloud Run
Permissions: 
  - run.services.create, run.services.update, run.services.get
  - iam.serviceAccountsActAs (to assume run.iam.gserviceaccount.com)
Role: roles/run.developer
WARNING: roles/run.admin is too broad

Task: Update Cloud SQL
Permissions:
  - cloudsql.instances.update
  - cloudsql.instances.get
NOT: cloudsql.*.* 
Role: Custom role (NO built-in matches exactly)

Task: Access Secrets Manager
Permissions:
  - secretmanager.secrets.get
  - secretmanager.versions.access
Role: roles/secretmanager.secretAccessor
```

#### Step 3: Create Custom IAM Role (if needed)
```bash
# Create YAML for custom role:
$ cat > custom_role.yaml << EOF
title: "CI/CD Pipeline Role"
description: "Minimal permissions for GitHub Actions deployment"
includedPermissions:
  # GCS permissions
  - storage.buckets.get
  - storage.objects.get
  
  # Cloud Run permissions
  - run.services.create
  - run.services.update
  - run.services.get
  - iam.serviceAccountsActAs (for run service account)
  
  # Cloud SQL permissions
  - cloudsql.instances.update
  - cloudsql.instances.get
  
  # Secrets Manager
  - secretmanager.secrets.get
  - secretmanager.versions.access
  
  # Logging
  - logging.logEntries.create
EOF

# Create the role:
$ gcloud iam roles create cicdPipelineRole \
  --project=PROJECT_ID \
  --file=custom_role.yaml
```

#### Step 4: Apply Permissions with Conditions
```bash
# Bind with condition: only production deployments during work hours
$ gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:builder@PROJECT_ID.iam.gserviceaccount.com \
  --role=projects/PROJECT_ID/roles/cicdPipelineRole \
  --condition='resource.matchTag("env","prod") AND request.time.getHours("America/Vancouver") >= 8 && request.time.getHours("America/Vancouver") <= 18'

# Add Service Account User role (needed for Cloud Run)
$ gcloud iam service-accounts add-iam-policy-binding \
  run@PROJECT_ID.iam.gserviceaccount.com \
  --member=serviceAccount:builder@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.serviceAccountUser
```

#### Step 5: Verify and Test
```bash
# Test each permission:
$ gcloud auth activate-service-account --key-file=builder-key.json
$ gsutil ls gs://prod-bucket/  # Test GCS
$ gcloud run services list --project=PROJECT_ID  # Test Cloud Run
$ gcloud secrets versions access latest --secret=db-password  # Test Secrets
```

### Follow-up Questions

**Q1: What if you need temporary elevated permissions for a one-time migration?**
- Expected: Discuss temporary access via gcloud compute ssh, audit logging, JIT access patterns

**Q2: How would you rotate service account keys after this is set up?**
- Expected: Workload Identity Federation, short-lived tokens, automated key rotation

**Q3: What audit trail would you set up?**
- Expected: Cloud Audit Logs, log sinks to BigQuery, alerts on permission changes

### Red Flags

❌ "Just give it Editor role, it's a CI/CD service account"
- This is a massive security risk

❌ "Use --condition flag later, let's just get it working"
- Shows you don't prioritize security

❌ "The error is about permissions, so add all storage.* permissions"
- Doesn't understand role hierarchy

### Pro Tips

✅ "I'd use Workload Identity Federation instead of long-lived service account keys"

✅ "I'd set up conditional IAM bindings to restrict access by time and environment"

✅ "I'd enable Cloud Audit Logs and set up BigQuery sink to track ALL permission usage"

✅ "I'd create a custom role instead of using built-in roles to follow least privilege"

---

## Scenario 3: Cross-Region Cloud SQL Failover Gone Wrong

### The Problem
Your primary Cloud SQL instance in us-central1 failed. You have a replica in us-east1 with read-only replica in asia-southeast1. The read replica is 2 hours behind and now serving critical production traffic because the failover promoted it to primary.

**How do you fix this without losing data and minimizing downtime?**

### Expected Answer (DETAILED)

#### Step 1: Assess the Situation (5 minutes)
```bash
# Check instance status:
$ gcloud sql instances describe prod-db-primary

# Check replica status:
$ gcloud sql instances describe prod-db-replica

# Check replication lag:
$ gcloud sql operations list --instance=prod-db-replica | head -5

# Check binary logs:
$ gcloud sql backups list --instance=prod-db-primary
```

#### Step 2: Data Loss Assessment
```
Critical question: How did failover occur?
- Manual failover? (Less data loss, easier rollback)
- Automatic failover? (Check if using HA, which has lower RTO/RPO)

If 2 hours behind:
- Lost data between T-120min and T0
- Check what transactions were lost
- Determine business impact

If using HA (Better situation):
- Less lag, likely minutes not hours
- Cleaner failover, better automation
```

#### Step 3: The Fix Strategy
```
Scenario A: If you have automated backups within last 2 hours
1. Create new instance from backup at T-120min
2. Use Cloud SQL Proxy to forward traffic to this instance
3. Restore missing data from binlog (if available)
4. Point application back to original name (using Cloud SQL Auth Proxy)

Scenario B: If data loss is acceptable (business decision)
1. Promote the replica to primary immediately
2. Create new replica in asia-southeast1
3. Document lost transactions
4. Plan data reconciliation

Scenario C: Live failover recovery (Complex but best)
1. Don't promote the replica yet
2. Create on-demand backup of current replica (2-hour slow state)
3. Repair primary instance in-place if possible
4. Resync replica from primary
```

#### Step 4: Promote Replica Carefully
```bash
# If you must promote, do it explicitly:
$ gcloud sql instances promote-replica prod-db-replica \
  --region=us-east1

# Verify the promotion:
$ gcloud sql instances describe prod-db-replica | grep instanceType

# Create immediate backup after promotion:
$ gcloud sql backups create \
  --instance=prod-db-replica \
  --description="Post-promotion backup"
```

#### Step 5: Restore from Backup
```bash
# If you have point-in-time recovery (PITR):
$ gcloud sql backups restore BACKUP_ID \
  --backup-instance=prod-db-primary \
  --backup-configuration=backup-config-id \
  --point-in-time=2024-01-15T10:30:00Z

# Or clone to new instance and sync:
$ gcloud sql instances clone prod-db-replica prod-db-restore \
  --point-in-time=2024-01-15T10:30:00Z
```

#### Step 6: Application DNS/Connection Cleanup
```
Important: How are applications connecting?

If using Cloud SQL Auth Proxy (RECOMMENDED):
- Update Cloud SQL Auth Proxy config to point to new primary
- Restart proxy on all servers
- Apps don't need changes

If using direct IP:
- Applications need connection string updates
- Use migration window with traffic rerouting

If using Cloud SQL Connector (RECOMMENDED):
- Connection info fetched dynamically
- Just update instance name in config
```

### Follow-up Questions

**Q1: How would you prevent this situation?**
- Expected: Discuss HA replicas, automated failover, proper backup strategy, RPO/RTO SLAs

**Q2: What if the primary instance is corrupted, not down?**
- Expected: Discuss backup restoration, data validation, incremental recovery

**Q3: How do you test failover without causing incidents?**
- Expected: Test replicas, staging environments, chaos engineering practices

### Red Flags

❌ "We don't have backups, let me restore from the replica"
- Shows zero backup strategy

❌ "Just delete the old primary and promote the replica"
- Immediate data loss, no rollback

❌ "We'll wait for the 2-hour replica to catch up"
- Bad production response, extended downtime

### Pro Tips

✅ "I'd check if we have BACKUP and BINLOG enabled for point-in-time recovery"

✅ "I'd use Cloud SQL Auth Proxy to abstract away the instance change"

✅ "I'd implement HA replicas with automatic failover for RPO < 5min"

✅ "I'd set up monitoring for replication lag > 1 minute to catch this early"

---

## Scenario 4: GKE Pod Cannot Access Internet

### The Problem
Your microservice pods deployed in GKE can access internal services fine, but cannot reach external APIs (e.g., `https://api.external-service.com`). Error: `connection timeout` after 30 seconds.

The cluster works fine in dev environment. Production just started failing 2 hours ago.

**Walk through systematic debugging approach.**

### Expected Answer (DETAILED)

#### Step 1: Verify Pod-Level Connectivity
```bash
# Check if pod can resolve DNS:
$ kubectl exec -it <pod-name> -- nslookup api.external-service.com
# If fails: DNS issue

# Check if pod has outbound internet access:
$ kubectl exec -it <pod-name> -- curl -v https://api.external-service.com --max-time 5
# If timeout: routing/firewall issue

# Check pod's default route:
$ kubectl exec <pod> -- ip route
# Should show: default via <gateway-ip>

# Check if DNS server is reachable:
$ kubectl exec <pod> -- cat /etc/resolv.conf
$ kubectl exec <pod> -- nslookup api.external-service.com 8.8.8.8  # Google DNS
```

#### Step 2: Check GKE Cluster Network Configuration
```bash
# Verify cluster has internet access enabled:
$ gcloud container clusters describe <CLUSTER_NAME> \
  --zone=<ZONE> \
  --format='value(networkPolicy.enabled)'

# Check if cluster has external endpoint:
$ gcloud container clusters describe <CLUSTER_NAME> \
  --zone=<ZONE> \
  --format='value(endpoint)'

# Check VPC network:
$ gcloud container clusters describe <CLUSTER_NAME> \
  --zone=<ZONE> \
  --format='value(network)'

# Verify internet gateway is configured:
$ gcloud compute networks subnets describe <SUBNET_NAME> \
  --network=<NETWORK_NAME> \
  --region=<REGION> \
  --format='value(privateIpGoogleAccess)'
```

#### Step 3: Check Firewall Rules
```bash
# List all egress firewall rules:
$ gcloud compute firewall-rules list \
  --filter="network:<NETWORK_NAME> AND sourceRanges=*"

# Check if default-allow-egress rule exists:
$ gcloud compute firewall-rules describe default-allow-egress

# If blocking rule found, check priority:
$ gcloud compute firewall-rules list --sort-by priority

# Check pod's source IP (from ingress rules perspective):
$ kubectl get pods -o wide  # Get pod IP

# Verify firewall allows traffic from pod IP to destination:
$ gcloud compute firewall-rules list \
  --filter="sourceRanges:10.0.0.0/8"  # Cluster CIDR
```

#### Step 4: Check NAT Gateway Configuration
```bash
# If using Cloud NAT (recommended for outbound):
$ gcloud compute routers list --project=<PROJECT_ID>
$ gcloud compute routers describe <ROUTER_NAME> \
  --region=<REGION> \
  --format='value(nats[].name)'

# Check NAT gateway status:
$ gcloud compute routers nats describe <NAT_NAME> \
  --router=<ROUTER_NAME> \
  --region=<REGION>

# If Cloud NAT is down:
# - Check if router is healthy
# - Check if NAT gateway has available IP addresses
# - Check if traffic is being routed through NAT

# Recreate NAT if needed:
$ gcloud compute routers nats create <NAT_NAME> \
  --router=<ROUTER_NAME> \
  --region=<REGION> \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips
```

#### Step 5: Check Network Policy
```bash
# Network policies might block external traffic:
$ kubectl get networkpolicy -A

# Check if any policy denies egress:
$ kubectl describe networkpolicy <policy-name>

# Check pod labels match the policy selector:
$ kubectl get pods <pod-name> --show-labels

# If policy is blocking, add egress rule:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32  # Block metadata service
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 80
```

#### Step 6: Check DNS Resolution
```bash
# If DNS is slow (might appear as timeout):
$ kubectl exec <pod> -- time nslookup api.external-service.com

# Check CoreDNS logs:
$ kubectl logs -n kube-system deployment/coredns | tail -50

# If CoreDNS is slow, check:
$ kubectl top pods -n kube-system  # Memory/CPU

# For persistent DNS issues:
$ kubectl get configmap coredns -n kube-system -o yaml
# Check if forward or loop plugins are misconfigured
```

#### Step 7: Route Tracing
```bash
# From node, trace route to external service:
NODE=$(kubectl get pod <pod> --template='{{.spec.nodeName}}')
gcloud compute ssh $NODE --zone=<ZONE>

# From node, check routes:
$ route -n
$ ip route

# Check if default route points to internet gateway:
$ ip route | grep default

# If missing, might need to add route:
$ gcloud compute routes list --filter network:<NETWORK_NAME>
```

#### Step 8: Test & Verify Fix
```bash
# After fix, verify connectivity:
$ kubectl exec <pod> -- curl -v https://api.external-service.com

# Test with timeout and retry:
for i in {1..5}; do
  kubectl exec <pod> -- curl -I https://api.external-service.com && break || sleep 5
done

# Monitor container logs:
$ kubectl logs <pod> -f | grep -i "connection\|error"
```

### Root Cause (Most Common)
❌ Missing Cloud NAT for pod egress traffic
❌ Firewall rule blocking 0.0.0.0/0 destination
❌ Pod's VPC subnet doesn't have internet gateway
❌ NetworkPolicy denying all egress

### Resolution
✅ Create Cloud NAT on router
✅ Add firewall egress rule allowing HTTPS (443) and HTTP (80)
✅ Configure route with internet gateway
✅ Update NetworkPolicy to allow external egress

### Follow-up Questions

**Q1: How would you manage this at scale (1000s of pods)?**
- Expected: Cloud NAT best practices, shared nat vs per-namespace, cost optimization

**Q2: What if you need to restrict outbound to specific IPs?**
- Expected: Egress firewall rules, Cloud NAT per-subnet, proxy solutions

**Q3: How would you monitor external API connectivity?**
- Expected: Synthetic monitoring, Prometheus exporters, alerting on connection failures

### Pro Tips

✅ "I'd start with Cloud NAT - it's the standard solution for GKE egress"

✅ "I'd check firewall rules before assuming pod misconfiguration"

✅ "I'd test with curl inside pod first, then trace through layers"

✅ "I'd implement NetworkPolicy egress rules before making connectivity work"

---

## Scenario 5: IAM Service Account Permission Denied

### The Problem
Your deployment uses a Kubernetes service account that's bound to a Google service account. It can read from Cloud Storage but fails with `Permission denied` when trying to write.

Error: `User: projects/-/serviceAccounts/app-sa@project.iam.gserviceaccount.com does not have storage.objects.create permission on resource gs://my-bucket/uploads/...`

The service account was working fine for writes 1 week ago.

**Trace the root cause and fix it.**

### Expected Answer (DETAILED)

#### Step 1: Verify Kubernetes Service Account Binding
```bash
# Check if workload identity is configured:
$ kubectl describe sa <SERVICE_ACCOUNT_NAME> -n <NAMESPACE>
# Should see: Annotations: iam.gke.io/gcp-service-account=...

# Get annotations:
$ kubectl get sa <SERVICE_ACCOUNT_NAME> -n <NAMESPACE> \
  -o jsonpath='{.metadata.annotations.iam\.gke\.io/gcp-service-account}'

# This should match your Google service account email
```

#### Step 2: Verify IAM Binding
```bash
# Check if Kubernetes SA is bound to Google SA:
$ gcloud iam service-accounts get-iam-policy \
  $(gcloud iam service-accounts describe app-sa@PROJECT.iam.gserviceaccount.com \
    --format='value(email)')

# Should see workload identity binding:
$ gcloud iam service-accounts get-iam-policy \
  app-sa@PROJECT.iam.gserviceaccount.com \
  --format='table(bindings[].principals[])' | grep kubernetes

# If binding missing, add it:
$ gcloud iam service-accounts add-iam-policy-binding \
  app-sa@PROJECT.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member=serviceAccount:PROJECT.svc.id.goog[NAMESPACE/POD_NAME]
```

#### Step 3: Check Google Service Account Permissions
```bash
# List all roles for the service account:
$ gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:app-sa@PROJECT.iam.gserviceaccount.com" \
  --format='table(bindings.role)'

# Check specific permissions:
$ gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:app-sa@PROJECT.iam.gserviceaccount.com" \
  --format='json' > sa_roles.json

# Verify storage.objects.create permission exists:
$ grep "storage.objects.create" sa_roles.json
# If missing, need to add role

# Add Storage Object Creator role:
$ gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:app-sa@PROJECT.iam.gserviceaccount.com \
  --role=roles/storage.objectCreator
```

#### Step 4: Check Bucket-Level Permissions
```bash
# List bucket IAM policy:
$ gsutil iam get gs://my-bucket

# Check if service account has explicit deny:
$ gsutil iam get gs://my-bucket | grep "app-sa"

# List effective permissions on folder:
$ gsutil iam ch serviceAccount:app-sa@PROJECT.iam.gserviceaccount.com:objectCreator gs://my-bucket/uploads

# If bucket has custom roles, verify they include storage.objects.create:
$ gcloud iam roles describe <CUSTOM_ROLE_NAME> \
  --format='value(includedPermissions[])'
```

#### Step 5: Check for Deny Policies
```bash
# Check if organization has deny policies blocking writes:
$ gcloud resource-manager org-policies list --project=PROJECT_ID

# Check for resource-level or folder-level deny policies:
$ gcloud resource-manager org-policies describe \
  constraints/storage.restrictObjectCreation \
  --project=PROJECT_ID

# If deny policy exists:
$ gcloud resource-manager org-policies delete \
  constraints/storage.restrictObjectCreation \
  --project=PROJECT_ID
```

#### Step 6: Verify Token Generation
```bash
# Inside pod, verify it can get Google credentials:
$ kubectl exec <POD> -- \
  curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://iam.googleapis.com/

# If fails, workload identity binding is broken

# Check if pod has correct namespace and SA:
$ kubectl get pod <POD> -n <NAMESPACE> -o yaml | grep -A5 serviceAccount

# Verify token is valid:
$ kubectl exec <POD> -- \
  gcloud auth list
```

#### Step 7: Test Permissions
```bash
# From pod, test write permission:
$ kubectl exec <POD> -- \
  gsutil -m cp /tmp/test.txt gs://my-bucket/uploads/test.txt

# If fails, get detailed error:
$ kubectl exec <POD> -- \
  gsutil -D cp /tmp/test.txt gs://my-bucket/uploads/test.txt 2>&1 | grep -i "permission\|denied"

# Test with explicit service account:
$ kubectl exec <POD> -- \
  gcloud config get-value account
```

#### Step 8: Diagnose Recent Changes
```bash
# Check if permissions were recently revoked:
$ gcloud resource-manager org-policies list --project=PROJECT_ID

# Check CloudAudit logs for policy changes:
$ gcloud logging read "resource.type=gce_instance AND \
  protoPayload.methodName=SetIamPolicy AND \
  protoPayload.resourceName=*app-sa*" \
  --limit 10 \
  --format json

# Look for "DeleteBinding" or "RevokeRole" in last 7 days
```

### Root Cause (Most Common)
❌ Service account role removed (most likely after permission audit)
❌ Bucket-level deny policy added
❌ Workload identity binding missing or broken
❌ Custom role permissions updated without storage.objects.create
❌ Organization policy restricting storage operations

### Resolution
✅ Re-add Storage Object Creator role to service account
✅ Verify workload identity binding exists
✅ Check and remove deny policies if applicable
✅ Test permissions after fix

### Follow-up Questions

**Q1: How would you set up proper bucket permissions using service accounts?**
- Expected: Least privilege, role separation, bucket vs organization policies

**Q2: How would you audit service account usage for over-privileged accounts?**
- Expected: Cloud Audit Logs, IAM recommender, periodic permission reviews

**Q3: What's the difference between organization policy and IAM permissions?**
- Expected: Org policy is org-wide constraint, IAM is per-resource access control

### Pro Tips

✅ "I'd check org policies BEFORE assuming service account permissions are the issue"

✅ "I'd verify workload identity binding with `kubectl exec ... gcloud auth list`"

✅ "I'd use Cloud Audit Logs to see when permissions changed"

✅ "I'd test with `gsutil -D` for detailed error debugging"

---

## Scenario 6: Load Balancer Not Routing Traffic to Pods

### The Problem
You deployed a service using Google Cloud Load Balancer (Ingress). The load balancer shows healthy status, but requests timeout or get 502 Bad Gateway.

Health checks show "Healthy" but actual requests fail. The Ingress looks correct.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 8080
```

**How do you diagnose this?**

### Expected Answer (DETAILED)

#### Step 1: Verify Service & Backend Configuration
```bash
# Check service endpoints are healthy:
$ kubectl get endpoints web-service
# Should show pod IPs

# Verify pods are actually running:
$ kubectl get pods -l app=web

# Check pod status:
$ kubectl describe pod <POD_NAME> | grep -A5 "Status:"

# Check if service has correct ports:
$ kubectl get svc web-service -o yaml | grep -A5 "ports:"

# Get pod IP and service IP:
$ kubectl get svc web-service -o jsonpath='{.spec.clusterIP}'
$ kubectl get pods -o wide
```

#### Step 2: Check Ingress Status
```bash
# Describe ingress to see status:
$ kubectl describe ingress web-ingress

# Check if ingress controller is running:
$ kubectl get pods -n ingress-nginx
# OR
$ kubectl get pods -n gke-system | grep ingress

# Get ingress IP:
$ kubectl get ingress web-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Check if IP is provisioned:
# If empty, ingress controller might be stuck
```

#### Step 3: Check Load Balancer Configuration in GCP
```bash
# List load balancers:
$ gcloud compute forwarding-rules list

# Find the one for your cluster:
$ gcloud compute forwarding-rules describe <FORWARDING_RULE_NAME> \
  --global

# Check backend services:
$ gcloud compute backend-services list

# Describe the backend service:
$ gcloud compute backend-services describe <BACKEND_SERVICE_NAME> \
  --global \
  --format='table(healthChecks[].name, backends[].group)'

# Check backend health:
$ gcloud compute backend-services get-health <BACKEND_SERVICE_NAME> \
  --global
```

#### Step 4: Check Health Checks
```bash
# List health checks:
$ gcloud compute health-checks list

# Describe the one used by load balancer:
$ gcloud compute health-checks describe <HEALTH_CHECK_NAME>

# Check health check configuration:
# - Port number (must match pod port)
# - Path (must exist and return 200)
# - Timeout (must be reasonable)

# Manually test health check:
$ kubectl exec <POD> -- curl -v http://localhost:8080/healthz

# If fails, that's why LB thinks backend is unhealthy
```

#### Step 5: Check Firewall Rules
```bash
# Load balancer needs firewall access to pods:
$ gcloud compute firewall-rules list \
  --filter="network=$(gcloud container clusters describe <CLUSTER> --zone <ZONE> --format='value(network)')"

# Check if rule allows 35.191.0.0/16 (GCP health checks):
$ gcloud compute firewall-rules describe <RULE_NAME>

# If missing, add it:
$ gcloud compute firewall-rules create allow-health-checks \
  --allow=tcp \
  --source-ranges=35.191.0.0/16,130.211.0.0/22 \
  --target-tags=gke-node
```

#### Step 6: Check Network Policy
```bash
# If using network policy, it might block LB -> pod traffic:
$ kubectl get networkpolicy -A

# Check for restrictive ingress policies:
$ kubectl describe networkpolicy <POLICY_NAME>

# LB communicates via node IPs, so need ingress rule:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-lb
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 35.191.0.0/16  # GCP health checks
    - ipBlock:
        cidr: 130.211.0.0/22  # GCP load balancer
    ports:
    - protocol: TCP
      port: 8080
```

#### Step 7: Check NEG Configuration
```bash
# Network Endpoint Groups translate LB traffic to pods:
$ gcloud compute network-endpoint-groups list --zone=<ZONE>

# Check NEG health:
$ gcloud compute network-endpoint-groups describe <NEG_NAME> \
  --zone=<ZONE> \
  --format='table(networkEndpoints[].status)'

# If unhealthy, check pod logs:
$ kubectl logs <POD> -c main
```

#### Step 8: Test Connectivity
```bash
# Get LB IP and test directly:
LB_IP=$(kubectl get ingress web-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# From outside cluster:
$ curl -v http://$LB_IP
$ curl -v --resolve app.example.com:80:$LB_IP http://app.example.com

# If fails with 502, backend isn't handling requests

# Check pod logs directly:
$ kubectl logs <POD_NAME> -f

# Check if pod is accepting connections:
$ kubectl exec <POD> -- netstat -tlnp | grep 8080
```

### Root Cause (Most Common)
❌ Health check failing (wrong port, wrong path)
❌ Pod not actually listening on specified port
❌ Firewall rules blocking 35.191.0.0/16
❌ NetworkPolicy blocking ingress traffic
❌ NEG not configured (pods shown unhealthy)

### Resolution
✅ Verify pod is listening: `kubectl exec <pod> -- netstat -tlnp`
✅ Test health check: `kubectl exec <pod> -- curl http://localhost:8080/healthz`
✅ Add firewall rules for GCP health checks
✅ Verify NetworkPolicy allows ingress

### Follow-up Questions

**Q1: How would you debug if only specific endpoints fail?**
- Expected: Service routing, traffic splitting, canary deployments

**Q2: What's the difference between a 502 and a timeout?**
- Expected: 502 = backend responding but erroring, timeout = LB can't reach backend

### Pro Tips

✅ "I'd verify the health check works from inside the pod first"

✅ "I'd check firewall rules for 35.191.0.0/16 and 130.211.0.0/22"

✅ "I'd use `gcloud compute backend-services get-health` to see NEG status"

---

## Scenario 7: Logs Not Reaching Cloud Logging (Stackdriver)

### The Problem
Your pods write logs to stdout, but they don't appear in Cloud Logging. The application is definitely running and writing logs (you can see them with `kubectl logs`), but nowhere in Cloud Logging UI.

Other clusters' logs work fine. This cluster is only 2 days old.

**Why are logs disappearing?**

### Expected Answer (DETAILED)

#### Step 1: Verify Logs Are Being Generated
```bash
# Check if kubectl logs shows anything:
$ kubectl logs <POD_NAME>
# If empty, app might not be logging

# Check pod logs with timestamps:
$ kubectl logs --timestamps=true <POD_NAME> | head -20

# Stream logs in real-time:
$ kubectl logs -f <POD_NAME>
```

#### Step 2: Check fluent-bit/fluentd Logging Agent
```bash
# Cloud Logging uses a logging agent (usually fluent-bit):
$ kubectl get daemonset -n kube-system | grep -i fluent

# If not found, agent might not be installed:
$ kubectl get pods -n kube-system | grep -i logging

# Check if agent is running on nodes:
$ kubectl get pods -n kube-system -o wide | grep -i fluent

# Check agent logs:
$ kubectl logs -n kube-system deployment/fluentd-gcp-v3.1.1 | tail -50

# Common error: agent can't access Cloud Logging API
$ kubectl logs -n kube-system ds/fluent-bit | grep -i "error\|failed\|permission"
```

#### Step 3: Verify Service Account Permissions
```bash
# Logging agent needs logging.logEntries.create permission:
$ gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:*compute@developer.gserviceaccount.com"

# The following role should exist:
$ gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:*compute*"

# Should have one of:
# - roles/logging.logWriter
# - roles/monitoring.metricWriter (includes logging)
# - roles/editor (has everything)

# If missing, add it:
$ gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --role=roles/logging.logWriter
```

#### Step 4: Check GKE Cluster Logging Configuration
```bash
# Verify cluster logging is enabled:
$ gcloud container clusters describe <CLUSTER_NAME> \
  --zone=<ZONE> \
  --format='value(loggingService)'
# Should return: logging.googleapis.com/kubernetes OR logging.googleapis.com

# If returns "none", logging is disabled:
$ gcloud container clusters update <CLUSTER_NAME> \
  --zone=<ZONE> \
  --logging=LOGGING,API_SERVER,SCHEDULER,CONTROLLER_MANAGER
```

#### Step 5: Check Network Connectivity to Logging Service
```bash
# Verify DNS can reach logging service:
$ kubectl exec -it <POD> -- nslookup logging.googleapis.com

# From node, check if can reach logging API:
NODE=$(kubectl get pod <POD> --template='{{.spec.nodeName}}')
gcloud compute ssh $NODE --zone=<ZONE>
$ curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://iam.googleapis.com/

# Check if firewall allows egress to logging.googleapis.com:
$ gcloud compute firewall-rules list \
  --filter="sourceRanges=10.0.0.0/8 AND name=default-allow-egress"
```

#### Step 6: Check if Logs Exist but Are Filtered
```bash
# Cloud Logging might have logs but they're not visible:
$ gcloud logging read "resource.type=k8s_container AND resource.labels.container_name=<POD>" \
  --limit=10

# Check if they appeared (might have delay):
$ gcloud logging read "resource.type=k8s_pod" --limit=5 --format=json

# If logs exist but not showing in UI, might be:
# - Filtering on resource type
# - Incorrect label matching
# - UI refresh issue (wait 1-2 minutes)
```

#### Step 7: Deploy Logging Agent if Missing
```bash
# If logging agent isn't running, manually deploy one:
# Option 1: Use Cloud Logging's fluent-bit container
$ kubectl apply -f - << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: kube-system
data:
  fluent-bit.conf: |
    [SERVICE]
        Daemon Off
        Flush 1
        Log_Level info

    [INPUT]
        Name tail
        Parser docker
        Path /var/log/containers/*.log
        Tag kube.*

    [OUTPUT]
        Name stackdriver
        Match *
        google_service_credentials /var/secrets/google/key.json
        resource k8s_container
EOF

# Option 2: Enable logging on cluster (easier):
$ gcloud container clusters update <CLUSTER_NAME> \
  --zone=<ZONE> \
  --logging=LOGGING --enable-ip-alias
```

#### Step 8: Test Logging Flow
```bash
# Create test log:
$ kubectl run test-logger --image=busybox -- \
  sh -c 'echo "TEST LOG $(date)" && sleep 300'

# Wait 30 seconds for pipeline:
$ sleep 30

# Query test log:
$ gcloud logging read "resource.type=k8s_container AND resource.labels.pod_name=test-logger" \
  --limit=5

# If appears, logging is working

# Cleanup:
$ kubectl delete pod test-logger
```

### Root Cause (Most Common)
❌ Logging disabled on cluster (loggingService: "none")
❌ Logging agent service account lacks permissions
❌ Logging agent pod not running
❌ Node service account doesn't have logging.logWriter role
❌ Firewall blocking googleapis.com egress

### Resolution
✅ Enable logging: `gcloud container clusters update ... --logging=LOGGING`
✅ Add IAM role: `gcloud projects add-iam-policy-binding ... --role=roles/logging.logWriter`
✅ Verify agent running: `kubectl get pods -n kube-system | grep fluent`
✅ Check egress firewall rules

### Follow-up Questions

**Q1: How would you set up structured logging in Cloud Logging?**
- Expected: JSON logging, log structuring, severity levels

**Q2: What if logs have PII and need to be excluded?**
- Expected: Log exclusion filters, structured logs, PII redaction

### Pro Tips

✅ "First check if logging is enabled on cluster, not the pod"

✅ "Use `gcloud logging read` to verify logs are being ingested"

✅ "Check node service account has logging.logWriter, not just pod SA"

---

## Scenario 8: VPC Peering Traffic Intermittently Fails

### The Problem
Your GKE cluster (VPC A) communicates with a managed database in another VPC (VPC B) using VPC peering. Most traffic works fine, but occasionally (2-3 times per hour) you get:

```
Error: connection timeout to database service
```

Then it works again. The pattern seems random.

**Walk through what could cause intermittent failures.**

### Expected Answer (DETAILED)

#### Step 1: Verify Peering Connection
```bash
# Check if peering exists:
$ gcloud compute networks peerings list --network <VPC_A>
$ gcloud compute networks peerings describe <PEERING_NAME> \
  --network=<VPC_A>

# Verify peering state is ACTIVE:
$ gcloud compute networks peerings describe <PEERING_NAME> \
  --network=<VPC_A> \
  --format='value(state)'

# Check both sides are connected:
$ gcloud compute networks peerings describe <PEERING_NAME> \
  --network=<VPC_A> \
  --format='table(state, auto_create_routes, export_custom_routes,import_custom_routes)'
```

#### Step 2: Check Routes
```bash
# Verify route to remote VPC exists:
$ gcloud compute routes list --filter="network=<VPC_A> AND destination_range=<VPC_B_CIDR>"

# Check all routes for missing routes to VPC B:
$ gcloud compute routes list --network=<VPC_A> | grep <VPC_B_CIDR>

# If missing, need to create route:
$ gcloud compute routes create <ROUTE_NAME> \
  --destination-range=<VPC_B_CIDR> \
  --network=<VPC_A> \
  --next-hop-gateway=peering

# Check if auto-create-routes is enabled:
$ gcloud compute networks peerings describe <PEERING_NAME> \
  --network=<VPC_A> \
  --format='value(auto_create_routes)'
```

#### Step 3: Check Firewall Rules
```bash
# Both VPCs need ingress rules allowing traffic:
# From VPC A:
$ gcloud compute firewall-rules list \
  --filter="network=<VPC_B> AND sourceRanges=<VPC_A_CIDR>"

# From VPC B:
$ gcloud compute firewall-rules list \
  --filter="network=<VPC_A> AND sourceRanges=<VPC_B_CIDR>"

# If missing, create:
$ gcloud compute firewall-rules create allow-from-vpc-a \
  --network=<VPC_B> \
  --allow=tcp,udp \
  --source-ranges=<VPC_A_CIDR>

$ gcloud compute firewall-rules create allow-from-vpc-b \
  --network=<VPC_A> \
  --allow=tcp,udp \
  --source-ranges=<VPC_B_CIDR>
```

#### Step 4: Check MTU Settings
```bash
# MTU mismatch can cause intermittent failures:
$ gcloud compute networks describe <VPC_A> \
  --format='value(mtu)'

$ gcloud compute networks describe <VPC_B> \
  --format='value(mtu)'

# Both should be 1460 or higher
# If different, packets get fragmented

# Check if issue is MTU-related by testing:
$ kubectl exec <POD> -- ping -s 1400 <REMOTE_VPC_IP>
# If fails but ping -s 100 works, MTU issue
```

#### Step 5: Check Connection State
```bash
# Monitor connections during failure:
$ kubectl exec <POD> -- netstat -tlnp | grep <DB_PORT>

# During failure:
$ kubectl exec <POD> -- \
  watch -n 0.1 'netstat -tlnp | grep <DB_PORT>'

# Look for TIME_WAIT or CLOSE_WAIT states
# These indicate connection issues

# Check connection backlog:
$ kubectl exec <POD> -- cat /proc/net/tcp | grep <DB_PORT> | awk '{print $4}' | sort | uniq -c
```

#### Step 6: Analyze Packet Loss
```bash
# Run mtr to TCP path analysis:
$ kubectl run -it --rm mtr --image=networkstatic/mtr -- \
  mtr -tcp -P 5432 <REMOTE_DATABASE_IP>

# Look for packet loss at any hop

# Use tcpdump on pod side:
$ kubectl exec <POD> -- \
  tcpdump -i eth0 -nn 'dst port 5432' -c 100

# On database side (if accessible):
$ tcpdump -i eth0 -nn 'src 10.0.0.0/8' -c 100
```

#### Step 7: Check Database Side
```bash
# If database is in another project's VPC:
# Ask database team to check:
# - Is database accepting connections?
# - Any resource limits on connection count?
# - Any automatic failover happening?

# Query database logs:
$ gcloud sql instances describe <DB_INSTANCE> \
  --format='value(state)'

# Check for connection pool issues on database side
```

#### Step 8: Implement Connection Retry
```
While investigating, implement retry logic to reduce impact:

Python Example:
import sqlalchemy
from sqlalchemy import pool

# Use connection pooling with retry:
engine = sqlalchemy.create_engine(
    'postgresql://...',
    poolclass=pool.QueuePool,
    pool_size=5,
    max_overflow=10,
    pool_recycle=3600,
    connect_args={
        'connect_timeout': 5,
        'keepalives': 1,
        'keepalives_idle': 30,
        'keepalives_interval': 10,
        'keepalives_count': 3,
    }
)
```

### Root Cause (Most Common)
❌ MTU mismatch causing packet fragmentation
❌ Firewall rule missing or order-dependent
❌ Database connection limit reached
❌ Connection pool exhaustion on application
❌ Peering route missing or disabled

### Resolution
✅ Set MTU to 1460 minimal
✅ Ensure explicit firewall rules both ways
✅ Increase database connection limit
✅ Implement connection pooling on app side
✅ Monitor and alert on connection failures

### Pro Tips

✅ "I'd check MTU first - common cause of intermittent failures"

✅ "I'd test with different packet sizes to confirm MTU issue"

✅ "I'd implement exponential backoff retry logic to reduce user impact"

---

## Scenario 9: Pod Gets Wrong IAM Permissions via Workload Identity

### The Problem
You have two apps in same namespace, each with separate Kubernetes service accounts:
- `app-read-sa` - should only read from storage
- `app-write-sa` - should read AND write to storage

Both pods start with credentials "working" but after 5 minutes, both have identical permissions (both read+write).

**What could cause service accounts to inherit each other's permissions?**

### Expected Answer (DETAILED)

#### Step 1: Verify Service Account Separation
```bash
# Check both KSAs exist:
$ kubectl get sa -n production
app-read-sa    
app-write-sa

# Check their annotations:
$ kubectl get sa app-read-sa -o yaml | grep gcp-service-account
$ kubectl get sa app-write-sa -o yaml | grep gcp-service-account

# They should point to different Google service accounts
# If both point to same GOOGLE_SA, that's the problem!
```

#### Step 2: Check Workload Identity Bindings
```bash
# For read-only service account:
$ gcloud iam service-accounts get-iam-policy \
  app-read-sa@PROJECT.iam.gserviceaccount.com \
  --format='table(bindings[].principals[])'

# Should only have one binding for app-read-sa:
# serviceAccount:PROJECT.svc.id.goog[production/app-read-sa]

# For write service account:
$ gcloud iam service-accounts get-iam-policy \
  app-write-sa@PROJECT.iam.gserviceaccount.com

# If both have binding to same KSA, permissions will merge!
```

#### Step 3: Check Pod Token
```bash
# Inside pod, check which service account credentials are being used:
$ kubectl exec -it <APP_READ_POD> -- \
  cat /var/run/secrets/workload.google.com/identity/namespace

$ kubectl exec -it <APP_READ_POD> -- \
  cat /var/run/secrets/workload.google.com/identity/service-account

# Should show:
# namespace: production
# service-account: app-read-sa

# If shows app-write-sa, pod is using wrong service account!
```

#### Step 4: Check Deployment Service Account Assignment
```bash
# Verify deployment uses correct service account:
$ kubectl get deployment app-read -o yaml | grep serviceAccountName

# Should be: serviceAccountName: app-read-sa

# If missing or wrong, deployment will use default service account

# If missing, add it:
$ kubectl patch deployment app-read -p '{"spec":{"template":{"spec":{"serviceAccountName":"app-read-sa"}}}}'
```

#### Step 5: Check GCP Service Account Roles
```bash
# List roles on each Google service account:
$ gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:app-read-sa@*"

$ gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:app-write-sa@*"

# If both have storage.objectCreator role, permissions fully merged

# Correct permissions:
# app-read-sa:  roles/storage.objectViewer (read-only)
# app-write-sa: roles/storage.objectEditor (read+write)

# Remove over-privileged role:
$ gcloud projects remove-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:app-read-sa@PROJECT.iam.gserviceaccount.com \
  --role=roles/storage.objectEditor
```

#### Step 6: Check Bucket-Level Permissions
```bash
# Bucket might grant same permissions to both:
$ gsutil iam get gs://my-bucket | grep -A5 -B5 "app-read"
$ gsutil iam get gs://my-bucket | grep -A5 -B5 "app-write"

# If same permissions granted, remove them:
$ gsutil iam ch -d serviceAccount:app-read-sa@PROJECT.iam.gserviceaccount.com:objectCreator gs://my-bucket
```

#### Step 7: Check Default Service Account
```bash
# If deployment doesn't specify serviceAccountName,
# all pods use "default" service account

# Check default SA:
$ kubectl get sa default -o yaml | grep gcp-service-account

# If default is bound to write service account,
# ALL pods get write permissions!

# Never bind default SA to privileged Google SA:
$ kubectl get sa default -o yaml
# Make sure it's either unbound or bound to minimal SA
```

#### Step 8: Test and Verify
```bash
# Force pod restart to pick up new workload identity:
$ kubectl rollout restart deployment app-read
$ kubectl rollout status deployment app-read

# Verify correct permissions:
$ kubectl exec <NEW_APP_READ_POD> -- \
  gsutil ls gs://my-bucket  # Should work (read only)

$ kubectl exec <NEW_APP_READ_POD> -- \
  gsutil cp /tmp/test.txt gs://my-bucket  # Should fail

$ kubectl exec <NEW_APP_WRITE_POD> -- \
  gsutil cp /tmp/test.txt gs://my-bucket  # Should work
```

### Root Cause (Most Common)
❌ Both KSAs bound to same Google SA
❌ Deployment uses default service account instead of named one
❌ Google SA has permissions at project level (granted to both)
❌ Bucket-level IAM grants same role to both accounts

### Resolution
✅ Use distinct Google service accounts for each KSA
✅ Bind KSA to Google SA properly (one-to-one)
✅ Use deployment-specific service accounts (not default)
✅ Apply least privilege at both project and bucket level

### Pro Tips

✅ "I'd check both the KSA annotation and actual pod token to verify"

✅ "I'd avoid using default service account - always specify one"

✅ "I'd test permissions from inside pod: `gcloud auth list`"

---

## Scenario 10: DNS Resolution Fails Intermittently in GKE

### The Problem
Pods can usually resolve services, but occasionally (5-10 times per hour):

```
error: getaddrinfo failed: Temporary failure in name resolution
```

Then DNS works again. The error is intermittent and doesn't correlate with pod changes.

**How do you debug intermittent DNS issues?**

### Expected Answer (DETAILED)

#### Step 1: Verify CoreDNS Is Running
```bash
# Check all CoreDNS replicas:
$ kubectl get pods -n kube-system -l k8s-app=kube-dns

# All should be Running and Ready:
$ kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide

# Check if any are restarting:
$ kubectl get pods -n kube-system -l k8s-app=kube-dns \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].restartCount}{"\n"}{end}'

# High restartCount indicates CoreDNS is crashing
```

#### Step 2: Check CoreDNS Resource Usage
```bash
# Monitor CoreDNS CPU/memory during failures:
$ kubectl top pods -n kube-system -l k8s-app=kube-dns

# If memory/CPU maxed out, that's the problem

# Check CoreDNS resource limits:
$ kubectl get pods -n kube-system -l k8s-app=kube-dns -o yaml | grep -A5 resources:

# If limits too low, increase them:
$ kubectl patch deployment coredns -n kube-system -p '{"spec":{"template":{"spec":{"containers":[{"name":"coredns","resources":{"requests":{"memory":"128Mi","cpu":"100m"},"limits":{"memory":"256Mi","cpu":"200m"}}}]}}}}'
```

#### Step 3: Check CoreDNS Cache
```bash
# CoreDNS might have cache issues:
$ kubectl exec <COREDNS_POD> -n kube-system -- \
  cat /etc/coredns/Corefile

# Look for cache parameters:
# cache 180
```

#### Step 4: Monitor DNS Queries
```bash
# From pod, test DNS resolution:
$ kubectl exec <POD> -- nslookup kubernetes.default
$ kubectl exec <POD> -- nslookup kubernetes.default.svc
$ kubectl exec <POD> -- nslookup kubernetes.default.svc.cluster.local

# Check resolution times:
$ kubectl exec <POD> -- time nslookup kubernetes.default

# If >1 second, performance issue

# Use dig for more details:
$ kubectl run -it --rm dnsutil --image=nsenter-dns -- \
  sh -c 'dig kubernetes.default +short'
```

#### Step 5: Check DNS Logging
```bash
# Enable debug logging in CoreDNS:
$ kubectl edit configmap coredns -n kube-system

# Add log directive:
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        log
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        cache 30
        forward . /etc/resolv.conf
    }

# Restart CoreDNS:
$ kubectl rollout restart deployment coredns -n kube-system

# Check logs:
$ kubectl logs -f deployment/coredns -n kube-system | grep -i error
```

#### Step 6: Check DNS Chaining
```bash
# Verify upstream DNS configuration:
$ kubectl exec <POD> -- cat /etc/resolv.conf

# Should show:
# nameserver 10.0.0.10  (CoreDNS)
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5 single-request-reopen

# If upstream nameservers are external IPs that timeout,
# that causes intermittent failures

# Check if external DNS is reachable:
$ kubectl exec <POD> -- dig @8.8.8.8 google.com +short
```

#### Step 7: Check for Connection Issues
```bash
# Sometimes DNS port (53) gets saturated:
$ kubectl exec <COREDNS_POD> -n kube-system -- \
  netstat -tlnp | grep 53

# Check connections:
$ kubectl exec <COREDNS_POD> -n kube-system -- \
  ss -s | grep UDP

# If UDP receive buffer full, add:
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        # Reduce verbosity to save resources
        log @NOT_ENABLED
        errors
        ...
    }
```

#### Step 8: Check DNS Service Endpoints
```bash
# Verify CoreDNS service has endpoints:
$ kubectl get endpoints kube-dns -n kube-system
$ kubectl get svc kube-dns -n kube-system

# If endpoints empty, no CoreDNS available

# Check pod selector matches:
$ kubectl get pods -n kube-system -l k8s-app=kube-dns
```

#### Step 9: Network Policy Check
```bash
# If using network policy, ensure DNS traffic allowed:
$ kubectl get networkpolicy -n kube-system

# Need to allow traffic to port 53:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: kube-system
spec:
  podSelector:
    matchLabels:
      k8s-app: kube-dns
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

#### Step 10: Test Resolution Under Load
```bash
# Run many parallel DNS queries to trigger issue:
$ for i in {1..100}; do
    kubectl exec <POD> -- nslookup kubernetes.default &
  done
$ wait

# If fails during load, resource issue

# Monitor CoreDNS during test:
$ kubectl top pods -n kube-system -l k8s-app=kube-dns
```

### Root Cause (Most Common)
❌ CoreDNS pod OOMKilled (memory pressure)
❌ CoreDNS resource limits too low
❌ External DNS upstream timeout
❌ Connection pool exhausted
❌ High cardinality DNS queries

### Resolution
✅ Increase CoreDNS resource limits
✅ Add caching for frequently queried names
✅ Use internal DNS only, avoid external lookups
✅ Monitor CoreDNS health and restart on failure

### Pro Tips

✅ "I'd check CoreDNS memory usage first - common cause"

✅ "I'd capture DNS logs during failure for analysis"

✅ "I'd increase DNS cache size to reduce CoreDNS load"

---

## Scenario 11: PersistentVolumeClaim Stuck in Pending State

### The Problem
Your deployment requested a PVC but it never becomes Bound:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: ssd
```

The PVC has been pending for 10 minutes. Your pod is waiting for the PVC before it can start.

**How do you diagnose and fix the pending PVC?**

### Expected Answer (DETAILED)

#### Step 1: Check PVC Status
```bash
# Get PVC status:
$ kubectl describe pvc app-storage
# Look for Status: Pending
# Check Events section for clues

# List PVCs:
$ kubectl get pvc -o wide
# Status should be Bound, if Pending there's problem

# Check if PV exists:
$ kubectl get pv
# Should show a PV with same storage size
```

#### Step 2: Verify StorageClass Exists
```bash
# Check if storageClass 'ssd' exists:
$ kubectl get storageclass

# Describe the storage class:
$ kubectl describe storageclass ssd

# Check provisioner:
$ kubectl get storageclass ssd -o jsonpath='{.provisioner}'
# Should be pd.csi.storage.gke.io for GCP persistent disks

# If SSD storage class doesn't exist, create:
$ kubectl apply -f - << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
EOF
```

#### Step 3: Check Storage Quota
```bash
# GCP has storage limits per region:
$ gcloud compute disks list --filter="zone:us-central1-a"

# Check quota in region:
$ gcloud compute project-info describe --project=PROJECT_ID \
  --format='value(quotas[name="DISKS_TOTAL_GB"].usage)'

$ gcloud compute project-info describe --project=PROJECT_ID \
  --format='value(quotas[name="DISKS_TOTAL_GB"].limit)'

# If usage >= limit, need to increase quota:
# Go to IAM & Admin > Quotas > Edit Quotas

# Or increase via API:
$ gcloud compute project-info describe --project=PROJECT_ID \
  --format='table(quotas[].name, quotas[].usage, quotas[].limit)' | grep DISK
```

#### Step 4: Check Namespace ResourceQuota
```bash
# Check if namespace has storage limit:
$ kubectl get resourcequota -n <NAMESPACE>

# Describe the quota:
$ kubectl describe resourcequota <QUOTA_NAME>

# Look for:
# requests.storage: 50Gi / 100Gi
# If at limit, can't create new PVC

# If quota blocking, either:
# 1. Increase quota
$ kubectl patch resourcequota <QUOTA_NAME> \
  -p '{"spec":{"hard":{"requests.storage":"200Gi"}}}'

# 2. Or delete unused PVCs to free space
$ kubectl delete pvc <UNUSED_PVC>
```

#### Step 5: Check PV Availability
```bash
# Is there an available PV?
$ kubectl get pv -o wide

# Check if available PV matches storage request:
$ kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.capacity.storage}{"\t"}{.status.phase}{"\n"}{end}'

# For dynamic provisioning (which 'ssd' does):
# should auto-create PV when PVC requested

# Check if CSI driver is running:
$ kubectl get pods -n kube-system | grep csi

# Should see:
# gke-gce-pd-csi-driver-controller-...
# gke-gce-pd-csi-plugin-...
```

#### Step 6: Check CSI Driver Configuration
```bash
# If CSI driver having issues:
$ kubectl describe daemonset -n gke-system gke-gce-pd-csi-plugin

# Check CSI driver logs:
$ kubectl logs -n gke-system -l app=gke-gce-pd-csi-plugin \
  --tail=50 | grep -i error

# If driver stuck, restart:
$ kubectl rollout restart daemonset gke-gce-pd-csi-plugin -n gke-system
```

#### Step 7: Check Availability Zone
```bash
# Pods must be in same AZ as PV:
$ kubectl get pod -o wide
# Note the NODE and zone

# Get node's zone:
$ kubectl get nodes -L topology.kubernetes.io/zone

# PVC must be provisioned in accessible zone:
$ gcloud compute disks list --filter="zone:us-central1-a"

# If PVC stuck, might be multi-zone cluster issue:
# Check PVC node selector:
$ kubectl describe pvc app-storage | grep -i "node\|selector"
```

#### Step 8: Test and Verify
```bash
# After applying fix, check PVC status:
$ kubectl describe pvc app-storage | grep Status

# Monitor PVC provisioning:
$ watch kubectl get pvc app-storage

# Once Bound, verify pod can use it:
$ kubectl get pod
# Pod should now be Running

# Mount and test:
$ kubectl exec <POD> -- ls -la /mnt/app-storage
```

### Root Cause (Most Common)
❌ StorageClass doesn't exist
❌ Reached GCP storage quota in region
❌ Namespace ResourceQuota at limit
❌ CSI driver not running
❌ Availability zone mismatch

### Resolution
✅ Verify StorageClass exists and uses correct provisioner
✅ Check GCP project quota for persistent disks
✅ Check namespace ResourceQuota and increase if needed
✅ Verify CSI driver pod is running
✅ Ensure pod and PV in same availability zone

### Pro Tips

✅ "I'd check StorageClass first - easiest issue to spot"

✅ "I'd use `kubectl describe pvc` for detailed error messages"

✅ "I'd verify CSI driver pods are running with `kubectl get pods -n gke-system | grep csi`"

---

## Scenario 12: Secret Not Mounted in Container

### The Problem
You created a Kubernetes Secret and mounted it in your deployment, but the container can't find the secret files.

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password  # This key doesn't exist in secret
```

The pod starts but fails on startup: `Key "password" not found in secret`.

**Walk through verifying the secret and fixing the mounting issue.**

### Expected Answer (DETAILED)

#### Step 1: Verify Secret Exists
```bash
# Check if secret exists:
$ kubectl get secret db-secret
$ kubectl get secret db-secret -o yaml

# List all secrets in namespace:
$ kubectl get secret

# Describe the secret to see keys:
$ kubectl describe secret db-secret

# Get secret value (base64 encoded):
$ kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

#### Step 2: Check Secret Keys
```bash
# Get all available keys in secret:
$ kubectl get secret db-secret -o jsonpath='{.data}' | jq 'keys[]'

# Should output available keys:
# "password"
# "username"
# "host"

# If "password" not in list, need to add it:
# Option 1: Update secret directly
$ kubectl create secret generic db-secret \
  --from-literal=password=newpassword \
  --from-literal=username=admin \
  -o yaml | kubectl apply -f -

# Option 2: Edit secret YAML:
$ kubectl edit secret db-secret
# Add correct key in data section
```

#### Step 3: Check Pod Environment Variables
```bash
# Verify pod CA see the secret:
$ kubectl exec <POD> -- env | grep DB_PASSWORD

# If empty, secret not mounted

# Check if pod spec references correct secret:
$ kubectl get deployment app -o yaml | grep -A5 secretKeyRef

# Should match:
# - name: db-secret
#   key: password
```

#### Step 4: Check Volume Mounts
```bash
# If using volume mount (not env variable):
$ kubectl describe pod <POD> | grep -A5 "Mounts:"

# Should show:
# Mounts:
#   /var/run/secrets/db-secret from db-secret (ro)

# If missing, secret not mounted as volume

# Check pod spec:
$ kubectl get pod <POD> -o yaml | grep -A10 "volumes:"

# Should have:
# - name: db-secret
#   secret:
#     secretName: db-secret
```

#### Step 5: Check File Path
```bash
# Inside pod, check if secret files exist:
$ kubectl exec <POD> -- ls -la /var/run/secrets/db-secret/

# Should show files matching secret keys:
# password
# username
# host

# If files missing, secret volume not mounted

# Check exact paths:
$ kubectl exec <POD> -- cat /var/run/secrets/db-secret/password
```

#### Step 6: Check Secret Access Permissions
```bash
# Pods need permission to read secrets:
$ kubectl get rolebinding -A | grep secret

# If none, create role binding:
$ kubectl create role secret-reader \
  --verb=get,list,watch \
  --resource=secrets

$ kubectl create rolebinding secret-reader \
  --role=secret-reader \
  --serviceaccount=default:default
```

#### Step 7: Check Secret Type
```bash
# Get secret type:
$ kubectl get secret db-secret -o jsonpath='{.type}'

# Common types:
# - Opaque (generic secret, most common)
# - kubernetes.io/service-account-token (auto-created)
# - kubernetes.io/dockercfg (Docker registry)
# - kubernetes.io/basic-auth

# If wrong type, secret might not mount:
$ kubectl patch secret db-secret -p '{"type":"Opaque"}'
```

#### Step 8: Verify After Fix
```bash
# Redeploy pod to use updated secret:
$ kubectl rollout restart deployment app

# Check pod status:
$ kubectl get pod -w

# Once running, verify secret available:
$ kubectl exec <POD> -- env | grep DB_PASSWORD
# Should show the secret value

# Or check file:
$ kubectl exec <POD> -- cat /var/run/secrets/db-secret/password
```

### Root Cause (Most Common)
❌ Secret key name mismatch (env var references non-existent key)
❌ Secret not mounted as volume in pod spec
❌ Wrong secret name specified in pod spec
❌ Secret doesn't exist at all
❌ Pod using different namespace than secret

### Resolution
✅ Verify secret exists with correct keys
✅ Ensure pod spec correctly references secret name and keys
✅ Check RBAC permissions for secret access
✅ Verify volume mounts are configured

### Pro Tips

✅ "I'd use `kubectl get secret -o jsonpath` to check available keys"

✅ "I'd verify pod spec uses exact secret name with grep"

✅ "I'd test secret access from pod with `cat` command directly"

---

## Scenario 13: GKE Control Plane Upgrade Causes Pod Eviction

### The Problem
Your cluster started a routine GKE control plane upgrade (worker node upgrade went fine). During upgrade, some pods got evicted and didn't reschedule for 10+ minutes.

Error in your monitoring:
```
Pod evicted: Nodepressure - MemoryPressure
Pod: app-prod pending for 10 minutes
```

But your cluster has plenty of memory available. Why didn't pods reschedule?

**Debug why pod eviction happened and how to prevent it.**

### Expected Answer (DETAILED)

#### Step 1: Check Cluster Upgrade Status
```bash
# Verify upgrade is happening:
$ gcloud container clusters describe <CLUSTER_NAME> \
  --zone=<ZONE> \
  --format='value(status)'
# Should be: RUNNING or UPDATING

# Check node pool status:
$ gcloud container node-pools describe default-pool \
  --cluster=<CLUSTER_NAME> \
  --zone=<ZONE> \
  --format='value(status)'

# Check if nodes are being cordoned/drained:
$ kubectl get nodes
# Should show some nodes with "NotReady" or "SchedulingDisabled"
```

#### Step 2: Check Pod Disruption Budget
```bash
# Pod Disruption Budgets determine if pod can be evicted:
$ kubectl get pdb -A

# Describe the PDB:
$ kubectl describe pdb <PDB_NAME>

# Should show:
# Disruptions allowed: 1
# Current health: 5

# If PDB too restrictive, pods can't be evicted:
$ kubectl patch pdb myapp \
  -p '{"spec":{"minAvailable":1}}'  # Relax constraint

# Best practice:
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 1  # Keep 1 available during disruption
  selector:
    matchLabels:
      app: myapp
```

#### Step 3: Check Pod Resource Requests
```bash
# Pods without resource requests are evicted first:
$ kubectl get deployment app -o yaml | grep -A10 resources:

# Should have:
# resources:
#   requests:
#     memory: "256Mi"
#     cpu: "100m"

# If missing requests, add them:
$ kubectl patch deployment app --patch='{"spec":{"template":{"spec":{"containers":[{"name":"app","resources":{"requests":{"memory":"256Mi","cpu":"100m"}}}]}}}}'
```

#### Step 4: Check Node Affinity
```bash
# Pods might have restrictive node selectors blocking rescheduling:
$ kubectl get pods -o yaml | grep -A5 "affinity:"
$ kubectl get pods -o yaml | grep -A5 "nodeSelector:"

# Pod can't reschedule if:
# - Requires specific node
# - Node is cordoned
# - Pod stuck in Pending

# Check pod events:
$ kubectl describe pod <POD_NAME>
# Look for "Unschedulable" reason

# Common reasons:
# - Insufficient memory/CPU
# - Node selector not found
# - Affinity violations
```

#### Step 5: Check Node Drain
```bash
# At end of control plane upgrade, nodes might drain slowly:
$ kubectl get nodes -o wide | grep -i "notready\|schedulingdisabled"

# Check pod eviction deadline:
$ kubectl get events -n <NAMESPACE> | grep -i "evicted\|drain"

# List recently evicted pods:
$ kubectl get pods --field-selector=status.phase=Failed | grep -i evicted
```

#### Step 6: Check Cluster Autoscaler
```bash
# Autoscaler might not scale up quickly enough:
$ kubectl describe cm cluster-autoscaler-status -n kube-system

# Check if autoscaler is responding:
$ kubectl logs -n kube-system deployment/cluster-autoscaler | tail -50

# If autoscaler stuck, might block rescheduling:
$ kubectl rollout restart deployment cluster-autoscaler -n kube-system
```

#### Step 7: Monitor Node Cordoning
```bash
# During upgrade, nodes get cordoned (can't accept new pods):
$ kubectl get nodes --show-labels | grep -i cordoned

# Force uncordon if stuck:
$ kubectl uncordon <NODE_NAME>

# Check node status:
$ kubectl describe node <NODE_NAME> | grep -A5 "Conditions:"
```

#### Step 8: Prevent Future Issues
```bash
# Set up Pod Disruption Budget for all services:
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2  # Always keep 2 copies during disruption
  selector:
    matchLabels:
      app: myapp

# Ensure replicas > minAvailable (at least 3 replicas recommended)

# Set up node affinity to spread pods:
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
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
              - myapp
          topologyKey: kubernetes.io/hostname
```

### Root Cause (Most Common)
❌ No Pod Disruption Budget defined
❌ Pod resource requests too low or missing
❌ Pod stuck in Pending due to PDB constraints
❌ Cluster Autoscaler not scaling up fast enough
❌ Node pool not configured with enough capacity

### Resolution
✅ Define Pod Disruption Budget for production services
✅ Set clear resource requests for all pods
✅ Ensure pod replicas >= PDB minAvailable + 1
✅ Monitor cluster autoscaler during upgrades
✅ Configure multiple node pools for resilience

### Pro Tips

✅ "I'd always define PDB for production workloads - it's critical for upgrades"

✅ "I'd check `kubectl describe pod` for Unschedulable events"

✅ "I'd monitor node drain speed during control plane upgrades"

---

## Scenario 14: Google Cloud Secrets Manager Integration Fails in Pods

### The Problem
Your pod tries to access secrets from Google Cloud Secrets Manager using the Secrets Store CSI Driver. It works in dev, but in production always fails:

```
Error: Failed to mount secret: rpc error: code = NotFound desc = secret "prod-db-password" not found
```

But the secret definitely exists in Cloud Secrets Manager (you can read it with gcloud).

**How do you diagnose secret access issues?**

### Expected Answer (DETAILED)

#### Step 1: Verify Secrets Store CSI Driver is Running
```bash
# Check if driver installed:
$ kubectl get pods -n kube-system | grep secrets-store-csi

# Should show:
# secrets-store-csi-driver-controller-...
# secrets-store-csi-driver-node-...

# If missing, install:
$ kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/secrets-store-csi-driver-provider-gcp/main/deploy/google-provider-installer.yaml
```

#### Step 2: Verify Google Cloud Secrets Provider
```bash
# Check if Google provider pod running:
$ kubectl get pods -n kube-system | grep gcp-provider

# Should show:
# secrets-store-csi-driver-provider-gcp-...

# Check provider logs:
$ kubectl logs -n kube-system -l \
  app=secrets-store-csi-driver-provider-gcp | tail -50
```

#### Step 3: Check Secret Exists in Cloud Secrets Manager
```bash
# List secrets in Cloud Secrets Manager:
$ gcloud secrets list

# Verify specific secret:
$ gcloud secrets versions access latest --secret="prod-db-password"

# If not found:
$ gcloud secrets create prod-db-password \
  --replication-policy="automatic"

# Add secret version:
$ echo -n "mysecretvalue" | \
  gcloud secrets versions add prod-db-password --data-file=-
```

#### Step 4: Check Pod Service Account Permissions
```bash
# Pod needs secretmanager.secretAccessor role:
$ gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:*compute*" \
  --format='table(bindings.role)'

# Should include roles/secretmanager.secretAccessor

# If missing, add:
$ gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor

# Or for specific service account:
$ gcloud secrets add-iam-policy-binding prod-db-password \
  --member=serviceAccount:app-sa@PROJECT.iam.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor
```

#### Step 5: Verify SecretProviderClass Configuration
```bash
# Check if SecretProviderClass exists:
$ kubectl get secretproviderclass

# Describe it:
$ kubectl describe secretproviderclass <CLASS_NAME>

# Should show:
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: prod-secrets
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/PROJECT_ID/secrets/prod-db-password/versions/latest"
        path: "db-password"

# If path wrong or secret name wrong, will fail
```

#### Step 6: Check Pod Spec Volume Mount
```bash
# Verify pod mounts the secret correctly:
$ kubectl get pod <POD> -o yaml | grep -A10 "volumes:"

# Should have:
# - name: secrets-store
#   csi:
#     driver: secrets-store.csi.k8s.io
#     readOnly: true
#     volumeAttributes:
#       secretProviderClass: "prod-secrets"

# Check volume mount path:
# - name: secrets-store
#   mountPath: /mnt/secrets
#   readOnly: true
```

#### Step 7: Test Secret Access from Pod
```bash
# Inside pod, check if secret file exists:
$ kubectl exec <POD> -- ls -la /mnt/secrets/

# Should show:
# db-password

# Read the secret:
$ kubectl exec <POD> -- cat /mnt/secrets/db-password

# If permission denied, check pod service account
$ kubectl exec <POD> -- gcloud secrets versions access latest \
  --secret="prod-db-password"
```

#### Step 8: Debug With Enhanced Logging
```bash
# Check provider logs during pod startup:
$ kubectl logs -n kube-system -l \
  app=secrets-store-csi-driver-provider-gcp --tail=100

# Look for:
# - Authentication errors
# - "secret not found"
# - Permission denied

# Check CSI driver logs:
$ kubectl logs -n kube-system -l app=secrets-store-csi-driver --tail=100

# Test with debug pod:
$ kubectl run -it --rm debug-pod \
  --image=gcr.io/google.com/cloudsdktool/cloud-sdk:latest -- \
  sh -c 'gcloud secrets versions access latest --secret="prod-db-password"'
```

### Root Cause (Most Common)
❌ Service account lacks secretmanager.secretAccessor role
❌ Secret doesn't exist or wrong secret name
❌ SecretProviderClass resourceName path wrong
❌ CSI driver or Google provider not installed
❌ Pod spec doesn't mount the secrets volume

### Resolution
✅ Grant secretmanager.secretAccessor to service account
✅ Verify secret exists: `gcloud secrets describe prod-db-password`
✅ Verify SecretProviderClass uses correct resourceName
✅ Ensure CSI driver pods running: `kubectl get pods -n kube-system | grep secrets`
✅ Test secret access from pod: `kubectl exec pod -- cat /mnt/secrets/db-password`

### Pro Tips

✅ "I'd verify CSI driver and Google provider running first"

✅ "I'd check service account IAM permissions BEFORE pod specs"

✅ "I'd test secret access from pod directly to isolate the layer"

---

## Scenario 15: GKE Autopilot Cluster Pods Not Scheduling Due to Missing Node Pools

### The Problem
You have a GKE Autopilot cluster. You deployed a pod with specific resource requirements (3000m CPU, 8Gi memory). The pod stays in Pending state forever:

```
kubectl describe pod <POD>
Events:
  Type     Reason             Age          Message
  ----     ------             ----         -------
  Warning  FailedScheduling   5m (x20)     no nodes available to schedule pods
```

Cluster status shows healthy. Why can't the pod schedule?

**Debug autopilot cluster scheduling issues.**

### Expected Answer (DETAILED)

#### Step 1: Verify Cluster is Autopilot
```bash
# Check cluster type:
$ gcloud container clusters describe <CLUSTER_NAME> \
  --zone=<ZONE> \
  --format='value(enableAutopilot)'
# Should return: True

# Check Autopilot configuration:
$ gcloud container clusters describe <CLUSTER_NAME> \
  --zone=<ZONE> \
  --format='value(autopilot.*)'
```

#### Step 2: Check Node Pools
```bash
# Autopilot manages node pools automatically:
$ gcloud container node-pools list \
  --cluster=<CLUSTER_NAME> \
  --zone=<ZONE>

# Should show at least one default node pool

# Get more details:
$ gcloud container node-pools describe default-pool \
  --cluster=<CLUSTER_NAME> \
  --zone=<ZONE> \
  --format='table(name, machineType, initialNodeCount, status)'
```

#### Step 3: Check Actual Nodes
```bash
# List kubernetes nodes:
$ kubectl get nodes

# Check node resources:
$ kubectl describe node <NODE_NAME> | grep -A10 "Allocated resources:"

# If no nodes, Autopilot hasn't created any yet

# Check node status:
$ kubectl get nodes -o wide
# All should be Ready
```

#### Step 4: Review Pod Resource Requirements
```bash
# Check what resources pod is requesting:
$ kubectl describe pod <POD> | grep -A5 "Requests"

# Example:
# Requests:
#   cpu: 3
#   memory: 8Gi

# These requirements must match available machine type:
# Autopilot has limited machine types

# List available machine types for Autopilot:
$ gcloud compute machine-types list --filter="zone:us-central1-a" \
  --format='table(name, memoryMb, guestCpus)' | head -20
```

#### Step 5: Check Autopilot Constraints
```bash
# Autopilot has specific constraints:
# - Minimum CPU per node: 250m
# - Maximum memory: Memory of largest machine type

# Check if pod fits standard machine:
# Standard machine types:
# - e2-standard-2: 2 vCPU, 8GB RAM
# - e2-standard-4: 4 vCPU, 16GB RAM
# - n1-standard-4: 4 vCPU, 15GB RAM

# If pod needs > 4 vCPU, might need special machine type

# Pod with 3 vCPU + 8GB = fits e2-standard-4

# Check if issue is machine type conflict:
$ kubectl describe pod <POD> | grep -i "machineType\|node-selector"
```

#### Step 6: Check Workload Placement
```bash
# Pod might have constraints blocking it from scheduling:
$ kubectl get pod <POD> -o yaml | grep -A10 "affinity:"
$ kubectl get pod <POD> -o yaml | grep -A10 "nodeSelector:"
$ kubectl get pod <POD> -o yaml | grep -A10 "tolerations:"

# If affinity requires specific zone autopilot might not support:
# Should work as is for standard placement
```

#### Step 7: Force Node Provisioning
```bash
# Autopilot should auto-provision nodes, but might be slow:
# - First node might take 5-10 minutes
# - Subsequent nodes 1-3 minutes

# If stuck, force by adding more replicas:
$ kubectl scale deployment <DEPLOYMENT> --replicas=3

# This triggers Autopilot to create more nodes

# Monitor node creation:
$ watch kubectl get nodes
$ watch gcloud container node-pools describe default-pool \
  --cluster=<CLUSTER_NAME> \
  --zone=<ZONE> \
  --format='value(currentNodeCount)'
```

#### Step 8: Check Compute Quota
```bash
# GCP quota might block node creation:
$ gcloud compute project-info describe --project=PROJECT_ID \
  --format='value(quotas[name="N1_CPUS"].usage)'

$ gcloud compute project-info describe --project=PROJECT_ID \
  --format='value(quotas[name="N1_CPUS"].limit)'

# Compare usage to limit

# If near limit, increase quota:
# Go to IAM Admin > Quotas > Edit Quotas
```

#### Step 9: Test with Smaller Resource Request
```bash
# Temporarily reduce resource request to test scheduling:
$ kubectl set resources deployment <DEPLOYMENT> \
  --requests cpu=500m,memory=512Mi

# Pod should schedule immediately

# Once verified issue is resource-related:
# - Increase cluster resources
# - Upgrade machine type
# - Switch to powerful machine pool

# Create high-memory node pool:
$ gcloud container node-pools create high-memory \
  --cluster=<CLUSTER_NAME> \
  --zone=<ZONE> \
  --machine-type=n1-memory-64 \
  --num-nodes=0 \
  --enable-autoscaling --min-nodes=0 --max-nodes=10
```

#### Step 10: Verify After Fix
```bash
# Once nodes provisioned:
$ kubectl get pods
# Pod should transition from Pending to Running

# Check node resources:
$ kubectl describe node | grep -A10 "Allocated resources:"

# Verify pod is actually scheduled:
$ kubectl get pod <POD> -o wide
# Should show node name
```

### Root Cause (Most Common)
❌ Pod resource request too large for Autopilot machine types
❌ Compute quota reached in project
❌ Autopilot slow provisioning (first node takes 10 min)
❌ Pod has node affinity constraints incompatible with Autopilot pool
❌ Insufficient replicas to trigger node auto-provisioning

### Resolution
✅ Verify pod resources fit standard machine types
✅ Check GCP compute quota hasn't been exceeded
✅ Wait for Autopilot to provision nodes (10+ min first time)
✅ Remove node affinity constraints or create matching node pool
✅ Monitor node-pool status and node creation progress

### Pro Tips

✅ "I'd check if issue is resource size vs quota vs provisioning time"

✅ "I'd test with scaled-down resources to verify scheduling works"

✅ "I'd watch `kubectl get nodes` during pod deployment to see provisioning"

✅ "I'd create high-memory node pools only when needed to save costs"

