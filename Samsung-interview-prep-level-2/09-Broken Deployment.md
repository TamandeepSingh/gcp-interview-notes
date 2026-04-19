# Broken Deployment — Mock Interview Scenarios

## Debugging Framework (Use Every Time)

```
1. Identify the layer    → pod / service / network / pipeline
2. Step-by-step checks   → pods → service → endpoints → external access
3. Show commands         → always state what you'd run
4. Root cause thinking   → "Possible reasons are..."
5. Resolution            → "I fix it by..."
```

---

## Scenario 1: Pods Running But App Not Accessible

**Interviewer:**
> Your application is deployed on GKE. Pods are running, but the application is not accessible from the browser. Walk me through how you debug this.

**Ideal Answer:**

First, I identify at which layer the issue is occurring — pod level, service level, or ingress/load balancer level.

```bash
# Step 1: Verify pods are Running AND Ready
kubectl get pods

# Step 2: Check application logs for errors
kubectl logs <pod>

# Step 3: Inspect service configuration
kubectl get svc
kubectl describe svc <service>
# Verify: selector labels, ports, type (LoadBalancer)

# Step 4: Check endpoints
kubectl get endpoints
# If empty → label mismatch between selector and pod labels

# Step 5: Verify external IP assigned
kubectl get svc
# Confirm EXTERNAL-IP is not <pending>
```

**Closing line:**
> "I debug layer by layer — pod → service → endpoints → external access — to isolate where traffic is breaking."

---

## Scenario 2: Pipeline Passed But New Version Not Deployed

**Interviewer:**
> Your GitHub Actions pipeline passed successfully, but the new version of the app is not reflected in Kubernetes. What could be wrong?

**Ideal Answer:**

First, I verify whether the new image was actually deployed to the cluster.

```bash
# Step 1: Check what image tag is running
kubectl describe deployment <name>
# Compare "Image:" field to the tag the pipeline built

# Step 2: Check rollout status
kubectl rollout status deployment/my-app

# Step 3: Look at deployment events
kubectl describe deployment <name>
```

**Possible issues:**
- Image tag not updated in deployment manifest (pipeline didn't update YAML)
- Deployment YAML not re-applied by pipeline
- ArgoCD not synced to latest commit
- `imagePullPolicy: IfNotPresent` serving a cached image
- Wrong namespace — pipeline deployed to different namespace

**Closing line:**
> "CI success does not guarantee deployment success — I always validate what is actually running in the cluster."

---

## Scenario 3: CrashLoopBackOff

**Interviewer:**
> A pod is in CrashLoopBackOff. How do you debug?

**Ideal Answer:**

```bash
# Step 1: Get events and container state
kubectl describe pod <pod>
# Events section shows: exit codes, last restart reason

# Step 2: Get current logs
kubectl logs <pod>

# Step 3: Get logs from previous crashed container
kubectl logs <pod> --previous
```

**Common reasons:**
- App crash (unhandled exception, missing library)
- Bad configuration (wrong env var, missing config file)
- Missing environment variable or secret not mounted
- Dependency not reachable (DB, external service)
- Health probe too aggressive causing premature restart

**Closing line:**
> "Logs and events are the fastest path to root cause — I always check both the current and previous container logs."

---

## Scenario 4: Logs Not Reaching ELK

**Interviewer:**
> Logs are not reaching Elasticsearch. What do you do?

**Ideal Answer:**

```bash
# Step 1: Check Fluent Bit pod health
kubectl get pods -n logging
kubectl logs <fluent-bit-pod>

# Step 2: Verify Fluent Bit output config
# - output plugin type (http / es / loki)
# - Logstash or ES endpoint correct?
# - auth configured?

# Step 3: Test network connectivity
curl http://elasticsearch:9200

# Step 4: Check Elasticsearch indices
curl http://elasticsearch:9200/_cat/indices
# If index missing → index template or name format issue
```

**Common issues:**
- Fluent Bit output plugin misconfigured
- Network policy blocking egress from logging namespace
- Elasticsearch cluster unhealthy (red/yellow status)
- Index mapping conflict (field type changed between log versions)
- Auth failure (changed API key not updated in Fluent Bit config)

**Closing line:**
> "I trace logs end-to-end from source to destination to identify exactly where the pipeline is breaking."

---

## Scenario 5: Pod Running But Service Not Working

**Interviewer:**
> Pod is running but service is not working. What do you check?

**Ideal Answer:**

```bash
# Step 1: Check service selector matches pod labels
kubectl get pods --show-labels
kubectl describe svc <service>
# Compare "Selector:" with pod labels

# Step 2: Check endpoints
kubectl get endpoints <service>
# If empty → label mismatch

# Step 3: Verify port configuration
kubectl describe svc <service>
# port (service port) vs targetPort (container port)

# Step 4: Test from within cluster
kubectl exec -it <another-pod> -- curl http://<service-name>:<port>

# Step 5: Check readiness probe
kubectl describe pod <pod>
# Is pod actually Ready? (1/1 vs 0/1)
```

**Root causes to consider:**
- Label mismatch → empty endpoints
- Wrong targetPort → connection refused
- Pod failing readiness probe → not added to endpoints
- Service in wrong namespace

**Closing line:**
> "A running pod doesn't mean a reachable service — I verify the full chain: pod readiness, label selectors, port mapping, and endpoints."

---

## Quick Command Reference

```bash
# Pod debugging
kubectl get pods
kubectl describe pod <name>
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl exec -it <pod> -- sh

# Service / Connectivity
kubectl get svc
kubectl get endpoints
kubectl describe svc <name>
kubectl get pods --show-labels

# Deployment
kubectl describe deployment <name>
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>

# Ingress
kubectl get ingress
kubectl describe ingress <name>

# Namespace-specific
kubectl get pods -n <namespace>
kubectl logs <pod> -n <namespace>
```
