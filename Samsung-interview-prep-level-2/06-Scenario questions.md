# Scenario Questions + Structured Answers

## Answer Framework (Use Every Time)

1. **Identify the layer** — "First I confirm where the issue is..."
2. **Step-by-step debugging** — "Then I check pods, services..."
3. **Show real commands**
4. **Root cause thinking** — "Possible reasons include..."
5. **Resolution** — "Then I fix by..."

---

## Scenario 1: Pod Cannot Access Internet

**Question:** Your pod is running but cannot access external APIs (internet). How do you debug?

**Answer:**

First, verify whether the issue is inside the pod or at the network level.

```bash
# Step 1: Exec into pod
kubectl exec -it <pod> -- sh

# Step 2: Test connectivity
ping google.com
curl https://google.com

# Step 3: Check DNS
nslookup google.com
# If DNS fails → CoreDNS issue
```

```bash
# Step 4: Check Network Policies
kubectl get networkpolicy
# Look for egress rules that may be blocking outbound traffic
```

**Step 5: Check NAT configuration**
In GKE private clusters, pods need Cloud NAT configured for internet access. Verify NAT gateway exists and covers the subnet.

**Closing line:**
> "I isolate whether it's DNS, network policy, or NAT configuration, and verify each layer step-by-step from inside the pod to cluster networking."

---

## Scenario 2: App Deployed But Not Accessible

**Question:** App is deployed but not reachable externally. What do you check?

**Answer:**

Debug layer by layer: Pod → Service → Ingress → Load Balancer.

```bash
# Step 1: Check pods
kubectl get pods
# Are they Running AND Ready?

# Step 2: Check service
kubectl get svc
kubectl describe svc <service>
# Verify: selector labels, ports, type

# Step 3: Check endpoints
kubectl get endpoints
# If empty → label mismatch between service selector and pod labels

# Step 4: Check ingress
kubectl get ingress
kubectl describe ingress <name>
# Verify: host rules, backend service name, port

# Step 5: Check external IP
kubectl get svc
# Verify LoadBalancer IP is assigned and accessible
```

**Closing line:**
> "I systematically validate each layer from pods to ingress to identify where the traffic is breaking."

---

## Scenario 3: Logs Not Reaching Elasticsearch

**Question:** Logs are not appearing in Elasticsearch. How do you debug?

**Answer:**

Check the log pipeline end-to-end from source to destination.

```bash
# Step 1: Check Fluent Bit pods
kubectl get pods -n logging
kubectl logs <fluent-bit-pod>

# Step 2: Verify Fluent Bit config
# - output plugin pointing to correct Logstash/ES endpoint?
# - index name format correct?
# - auth configured?

# Step 3: Network check
curl http://elasticsearch:9200

# Step 4: Check Elasticsearch indices
curl http://elasticsearch:9200/_cat/indices
```

**Common issues:**
- Wrong index name format (date suffix mismatch)
- Auth failure (API key or basic auth changed)
- DNS failure (ES hostname not resolving)
- Network policy blocking Fluent Bit → Elasticsearch traffic

**Closing line:**
> "I trace logs from Fluent Bit to Elasticsearch, validating configuration, connectivity, and indexing issues."

---

## Scenario 4: Terraform Apply Destroying Resources

**Question:** Terraform plan shows resource destruction unexpectedly. What do you do?

**Answer:**

Do NOT apply immediately. First analyze why Terraform is planning destruction.

```bash
# Step 1: Review plan output carefully
terraform plan

# Step 2: Check state
terraform state list

# Step 3: Compare state vs config
terraform state show <resource>
```

**Possible reasons:**
- State mismatch (resource exists in infra but not in state)
- Manual drift (resource changed outside Terraform)
- Config change (attribute that forces recreation)
- Resource renamed in code

**Fix options:**
```bash
# Import existing resource into state
terraform import <resource_type.name> <resource_id>

# Or update config to match actual resource
# Then re-run terraform plan to verify no destruction
```

**Closing line:**
> "I always validate the plan, identify drift or config mismatch, and reconcile state before applying changes."

---

## Scenario 5: Pipeline Passed But Deployment Failed

**Question:** CI pipeline passed, but deployment is broken. Why?

**Answer:**

CI success doesn't guarantee deployment success. Check runtime issues.

```bash
# Step 1: Check what image tag is actually deployed
kubectl describe deployment <name>
# Compare image tag with what the pipeline built

# Step 2: Check deployment status
kubectl get pods
kubectl describe deployment <name>

# Step 3: Check pod logs
kubectl logs <pod>

# Step 4: Check ArgoCD (if used)
# - sync status: is it OutOfSync?
# - health status: degraded?
```

**Common issues:**
- Image tag not updated in manifest
- Wrong environment variables in deployment
- Missing secrets (Secret not created in target namespace)
- Config mismatch between environments
- ArgoCD not synced to latest commit

**Closing line:**
> "I validate that the deployed image and configuration match the pipeline output, then debug runtime issues in Kubernetes."

---

## Quick Reference: Debugging Commands

```bash
# Pod status
kubectl get pods
kubectl get pods -o wide
kubectl describe pod <name>
kubectl logs <pod>
kubectl logs <pod> --previous    # previous container instance
kubectl exec -it <pod> -- sh

# Service / Network
kubectl get svc
kubectl get endpoints
kubectl describe svc <name>
kubectl get networkpolicy

# Deployment
kubectl get deployment
kubectl describe deployment <name>
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>

# Ingress
kubectl get ingress
kubectl describe ingress <name>

# Terraform
terraform plan
terraform state list
terraform state show <resource>
terraform import <resource> <id>
```
