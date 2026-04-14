# Ansible & Helm Scenarios - Samsung DevOps Interview

## Scenario 1: Ansible Playbook Idempotency Breaks After 5 Runs

### The Problem
Your Ansible playbook deploys a microservice across 100 servers. It works fine for the first 5 runs but fails on the 6th run with:
```
ERROR! unexpected failure during handler execution: 'str' object is not subscriptable
```

Running the same playbook on affected servers manually works fine. Re-running the complete cluster fails.

**How do you diagnose and fix this without disrupting production?**

### Expected Answer (DETAILED)

#### Step 1: Enable Debug Logging
```bash
# Run with maximum verbosity:
$ ansible-playbook deploy.yml -vvvv 2>&1 | tee debug-run.log

# Look for the exact failure point:
$ grep -i "fatal\|error\|traceback" debug-run.log | head -20

# Check which host failed first:
$ grep -B5 "unexpected failure" debug-run.log

# Get the exact line of playbook causing issue:
$ grep -A5 "handler execution" debug-run.log
```

#### Step 2: Understand the Error
```
Error: 'str' object is not subscriptable
This typically means:
- Trying to access item[key] on a string instead of dict
- Variable type changed (was dict, now string)
- Jinja2 template returning wrong type

Common causes:
1. Register variable being re-used incorrectly
2. Handler accessing task result with wrong assumptions
3. State file created by previous run causing issues
4. Variable namespacing collision
```

#### Step 3: Common Playbook Patterns That Break
```yaml
# PROBLEM 1: Handler accessing task result incorrectly
tasks:
  - name: Get service status
    shell: systemctl status my-service
    register: service_status
    # After run 5, this returns string not dict!

handlers:
  - name: Restart service
    systemd:
      name: my-service
      state: restarted
    # If this handler tries to access service_status['stdout']
    # but service_status is now a string, it fails

# FIX:
handlers:
  - name: Restart service
    systemd:
      name: my-service
      state: restarted
    # Handler shouldn't access register vars at all
    # Keep handlers simple and independent


# PROBLEM 2: Variable accumulation across runs
tasks:
  - name: Gather facts about servers
    shell: hostname
    register: hostnames  # Gets appended?
    # After multiple runs, this might be list instead of string

# FIX: Always set to fresh value
tasks:
  - name: Gather facts
    shell: hostname
    register: hostnames
    changed_when: false  # Don't trigger handlers
    check_mode: no
  
  - name: Debug hostnames
    debug:
      msg: "{{ hostnames.stdout }}"


# PROBLEM 3: Reusing register variable with different values
tasks:
  - name: First command
    shell: echo "test1"
    register: result

  - name: Second command  
    shell: echo "test2"
    register: result
    # Same var, different content type
    
  - name: Use result
    debug:
      msg: "{{ result.stdout }}"
    # Ambiguous - which result are we using?

# FIX: Use different variable names
tasks:
  - name: First command
    shell: echo "test1"
    register: result_first

  - name: Second command
    shell: echo "test2"
    register: result_second
```

#### Step 4: Diagnose State Accumulation
```bash
# Check if Ansible is caching data between runs:
$ ansible-inventory --list | jq '.all.hosts | length'

# Check if group_vars are being accumulated:
$ cat group_vars/all/*.yml | grep -E "register|var:" | wc -l

# Look for dangling state files:
$ find /etc/ansible -name "*.json" -o -name "*.yaml" -o -name ".cache" | head -20

# Check if variables persist across plays:
$ grep -r "set_fact" deploy.yml
# These might accumulate if not namespaced properly
```

#### Step 5: Add Explicit Variable Cleanup
```yaml
# Ensure clean state at start of each play:
---
- name: Deploy Service
  hosts: all
  gather_facts: yes
  
  vars:
    # Explicitly set once at play level
    service_name: my-service
    deployment_version: "{{ lookup('env','DEPLOY_VERSION') }}"
  
  pre_tasks:
    # Clear any dangling registers from previous runs
    - name: Initialize empty variables
      set_fact:
        result_check: {}
        service_status: {}
        deployment_log: []
        temp_files: []
      run_once: true
  
  tasks:
    - name: Pre-flight checks
      block:
        - name: Check service exists
          command: systemctl is-enabled {{ service_name }}
          register: result_check
          ignore_errors: yes
          changed_when: false
        
        - name: Validate registration result
          assert:
            that:
              - result_check is defined
              - result_check.rc is defined
            fail_msg: "Service check returned unexpected format"
  
  handlers:
    - name: Restart service
      systemd:
        name: "{{ service_name }}"
        state: restarted
      listen: "restart service"
  
  post_tasks:
    - name: Cleanup temporary state
      set_fact:
        result_check: null
        service_status: null
      # Clean up to prevent accumulation
```

#### Step 6: Implement Proper Testing
```bash
# Test playbook multiple times:
$ for i in {1..10}; do
    echo "Run $i:"
    ansible-playbook deploy.yml -e "run_number=$i"
    if [ $? -ne 0 ]; then
      echo "FAILED on run $i"
      break
    fi
  done

# Test with different hostsets:
$ ansible-playbook deploy.yml --limit "50%"  # First half
$ ansible-playbook deploy.yml --limit "50%[1]"  # Second half

# Dry-run first:
$ ansible-playbook deploy.yml --check

# Diff mode to see what will change:
$ ansible-playbook deploy.yml --diff --check
```

#### Step 7: Production Safeguards
```yaml
# Add to your playbook:

- name: Safety Checks Before Deployment
  hosts: all
  gather_facts: no
  
  tasks:
    - name: Check for stale Ansible caches
      stat:
        path: /tmp/ansible_*
      register: stale_caches
    
    - name: Fail if stale caches exist
      assert:
        that:
          - stale_caches.stat.exists == false
        fail_msg: "Stale Ansible caches detected. Clean before running."
    
    - name: Validate JSON fact files
      shell: jq empty /etc/ansible/facts.d/*.json 2>/dev/null || true
      changed_when: false
    
    - name: Ensure register variables are isolated
      debug:
        msg: |
          Verify all register variables are:
          1. Namespaced uniquely
          2. Not accumulated from previous runs
          3. Have explicit handlers, not shared
```

### Follow-up Questions

**Q1: How would you make this playbook idempotent by design?**
- Expected: Avoid shell commands, use modules that are idempotent, state declarations

**Q2: What's the difference between idempotent and deterministic?**
- Expected: Idempotent = same result running 1 or N times; Deterministic = predictable

**Q3: How would you test idempotency in CI/CD?**
- Expected: Run playbook 5+ times, assert no changes after first run

### Red Flags

❌ "I'll just restart Ansible for each run"
- Doesn't fix underlying design issue

❌ "The error message is unclear, must be Ansible bug"
- Always look for root cause in own code first

❌ "Let me add ignore_errors to skip this"
- Masking the problem

### Pro Tips

✅ "I'd use ansible-test to validate playbook syntax before running"

✅ "I'd implement idempotency checks: run twice, assert no changes on run 2"

✅ "I'd use check mode to debug without making changes"

✅ "I'd namespace all register variables with task name prefix"

---

## Scenario 2: Helm Chart Upgrade Fails, Values.yaml Confusion

### The Problem
You're upgrading a Helm chart from v1.2.0 to v2.0.0. The upgrade command succeeds, but your pods are stuck in `CrashLoopBackOff`.

New version has changed values structure:
```
v1.2.0: replicas: 3
v2.0.0: replicaSet: { count: 3 }
```

Your values file uses the old structure. Application containers are crashing because environment variables don't match what the new chart expects.

**How do you fix this without data loss?**

### Expected Answer (DETAILED)

#### Step 1: Quick Diagnosis
```bash
# Check pod status:
$ helm list -a
$ helm status <release-name>

# Check chart version actually upgraded:
$ helm get values <release-name> --all
$ helm get manifest <release-name> | head -30

# Check pod logs to see actual error:
$ kubectl logs <pod-name> --tail=50
$ kubectl describe pod <pod-name> | grep -A10 "Events:"

# Check if it's a config issue:
$ kubectl get configmap <release-name> -o yaml | grep -E "config|err"
```

#### Step 2: Identify the Schema Change
```bash
# Get old values:
$ helm get values <release-name> > old-values.yaml

# Get the chart defaults:
$ helm show values <chart-name> --version 2.0.0 > new-defaults.yaml

# Compare the two:
$ diff -u old-values.yaml new-defaults.yaml | head -50

# Key changes to look for:
$ grep -E "replicas|replicaSet|replica" *-values.yaml old-defaults.yaml new-defaults.yaml
```

#### Step 3: Immediate Remediation - Rollback
```bash
# If urgent, rollback to last working version:
$ helm rollback <release-name> <revision-number>

# Check previous revisions:
$ helm history <release-name>

# Verify rollback successful:
$ kubectl rollout status deployment/<release-name>
$ kubectl get pods
```

#### Step 4: Create Migration Path
```yaml
# new-values.yaml - mapping old to new structure

# Old structure still valid for backward compat in v2.0.0:
replicas: 3  # Charts should support both

# OR use new structure:
replicaSet:
  count: 3

# Keep everything else:
image:
  repository: myapp
  tag: latest

resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# Environment is usually in values:
env:
  LOG_LEVEL: info
  DB_HOST: postgres.default.svc.cluster.local
```

#### Step 5: Dry-Run Before Upgrade
```bash
# Dry-run the upgrade with new values:
$ helm upgrade <release-name> <chart-name> \
  --version 2.0.0 \
  -f new-values.yaml \
  --dry-run \
  --debug

# Review the output manifests:
$ helm upgrade <release-name> <chart-name> \
  --version 2.0.0 \
  -f new-values.yaml \
  --dry-run \
  --debug | kubectl apply -f - --dry-run=client

# Check if generates warnings:
$ helm template <release-name> <chart-name> \
  --version 2.0.0 \
  -f new-values.yaml | kubectl apply -f - --dry-run=server
```

#### Step 6: Staged Upgrade Strategy
```bash
# Strategy: Create new release, traffic shift, then delete old one

# 1. Deploy new version side-by-side:
$ helm install <release-name>-v2 <chart-name> \
  --version 2.0.0 \
  -f new-values.yaml

# 2. Verify new deployment works:
$ kubectl get pods -l app=<release-name>-v2
$ kubectl logs -l app=<release-name>-v2 -f

# 3. Update ingress/service to point to both (canary):
$ kubectl patch service <release-name> -p '{
  "spec": {
    "selector": {
      "app": "<release-name>-v2"  # Switch traffic
    }
  }
}'

# 4. Wait and monitor for errors:
$ kubectl get events -A --field-selector involvedObject.name=<release-name>-v2

# 5. If successful, delete old release:
$ helm uninstall <release-name>

# 6. Rename new release:
$ helm get values <release-name>-v2 > /tmp/vals.yaml
$ helm uninstall <release-name>-v2
$ helm install <release-name> <chart-name> --version 2.0.0 -f /tmp/vals.yaml
```

#### Step 7: Proper Upgrade Path
```bash
# Once values are fixed, do normal upgrade:

# 1. Backup current state:
$ helm get values <release-name> > backup-values.yaml
$ kubectl get all -o yaml > backup-resources.yaml

# 2. Perform upgrade carefully:
$ helm upgrade <release-name> <chart-name> \
  --version 2.0.0 \
  -f new-values.yaml \
  --timeout 5m \
  --wait \
  --atomic  # Rollback if timeout

# 3. Monitor the rollout:
$ kubectl rollout status deployment/<release-name> --timeout=5m

# 4. Health checks:
$ kubectl get pods
$ kubectl get events | grep -i error
```

#### Step 8: Prevent This in Future Releases
```yaml
# Add migration guide to chart

# Chart.yaml
apiVersion: v2
name: myapp
version: 2.0.0
description: Updated version with new values structure
annotations:
  # Add migration information
  migration-required: "true"
  migration-guide: "See MIGRATION.md"

---
# values.yaml - Support both old and new format

# NEW RECOMMENDED FORMAT:
replicaSet:
  count: 3

# OLD FORMAT KEPT FOR BACKWARD COMPATIBILITY:
replicas: 3

# In templates, handle both:
# {{ .Values.replicaSet.count | default .Values.replicas | default 1 }}

---
# Create MIGRATION.md
# Upgrading from v1.2.0 to v2.0.0

## Breaking Changes

1. **replicas structure changed**
   - Old: `replicas: 3`
   - New: `replicaSet: { count: 3 }`
   - Both formats supported for transition period

2. **Environment variable mapping**
   - Old: ENV vars in deployment
   - New: ConfigMap preferred

## Migration Steps

1. Update values.yaml before upgrading
2. Run: `helm upgrade myapp ./myapp -f values.yaml --dry-run`
3. Review changes
4. Execute: `helm upgrade myapp ./myapp -f values.yaml`
5. Monitor: `kubectl get pods -w`
```

#### Step 9: Version Gating in CI/CD
```yaml
# test-upgrade.yaml in CI pipeline
name: Test Helm Upgrade

on: [pull_request]

jobs:
  upgrade-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Kind cluster setup
        uses: helm/kind-action@v1.9.0
      
      - name: Install previous chart version
        run: |
          helm repo add myrepo https://charts.example.com
          helm install myapp myrepo/myapp --version 1.2.0
      
      - name: Wait for deployment
        run: kubectl rollout status deployment/myapp --timeout=5m
      
      - name: Test upgrade to new version
        run: |
          helm upgrade myapp ./chart \
            -f test-values.yaml \
            --wait \
            --timeout=5m
      
      - name: Verify pods running
        run: |
          kubectl get pods
          kubectl wait --for=condition=Ready pod \
            -l app=myapp \
            --timeout=300s
      
      - name: Test application
        run: |
          kubectl port-forward svc/myapp 8080:80 &
          sleep 2
          curl http://localhost:8080/health
```

### Follow-up Questions

**Q1: How do you handle Helm secrets that might be affected by schema changes?**
- Expected: Discuss sealed-secrets, external vault integration, rotation strategies

**Q2: What would you include in upgrade runbook?**
- Expected: Backup steps, rollback procedure, health checks, communication plan

**Q3: How would you test this with real data?**
- Expected: Staging environment, production data restoration, load testing

### Red Flags

❌ "I'll just revert the values back to old format"
- Doesn't take advantage of new features

❌ "No need for dry-run, just do it"
- Shows recklessness with production

❌ "The chart version should handle backward compatibility"
- Sometimes it doesn't, need to verify

### Pro Tips

✅ "I'd use `helm upgrade --dry-run --debug` first"

✅ "I'd implement canary deployment with traffic shifting"

✅ "I'd automate upgrade testing in CI/CD pipeline"

✅ "I'd document migration path in MIGRATION.md"

---

## Scenario 3: ConfigMap Secret Rotation Fails in 50 Pods

### The Problem
You update a ConfigMap that 50 pods use as environment variables. Most pods pick up the change, but 10% are still using old values after 30 minutes. Some pods show:

```
Warning: ConfigMap prod-config was updated, but pods still using cached version
```

Your pod doesn't have a sidecar watching for ConfigMap changes. The application doesn't auto-reload config.

**How do you force pod refresh without downtime?**

### Expected Answer (DETAILED)

#### Step 1: Verify ConfigMap Update
```bash
# Check if ConfigMap actually updated:
$ kubectl get configmap prod-config -o yaml | grep -A20 data:

# Check when it was updated:
$ kubectl get configmap prod-config -o jsonpath='{.metadata.managedFields}' | jq '.[] | .time'

# Compare with pod mounts:
$ kubectl get pods -o yaml | grep -A3 "configMap:" | head -20
```

#### Step 2: Understand Pod Mounting
```bash
# Check how pods mount the configmap:
$ kubectl get deployment myapp -o yaml | grep -A10 "volumes:"

# Check actual mount in running pod:
$ kubectl exec <pod-name> -- ls -la /etc/config/

# Compare with ConfigMap:
$ kubectl get configmap prod-config -o jsonpath='{.data}'

# Check if mounted as volume or env:
$ kubectl exec <pod-name> -- env | grep CONFIG
```

#### Step 3: Identify Non-Updating Pods
```bash
# Get pods that haven't refreshed:
# ConfigMap stored as /etc/config/... gets updated immediately
# BUT env vars from configmap DON'T get updated automatically

# For env-based config:
$ kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | while read pod; do
    kubectl exec $pod -- env | grep APP_CONFIG_VERSION
  done | sort | uniq -c

# Pods with old version haven't restarted
```

#### Step 4: Force Pod Refresh - Strategy 1: Rolling Restart
```bash
# Restart pods gracefully with RollingUpdate:

# Add annotation to trigger restart:
$ kubectl patch deployment myapp \
  -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"restart-trigger\":\"$(date +%s)\"}}}}}}"

# This forces all pods to restart without stopping service
# Verify restart:
$ kubectl rollout status deployment/myapp --timeout=10m

# Alternative: Use kubectl rollout restart
$ kubectl rollout restart deployment/myapp
$ kubectl rollout status deployment/myapp
```

#### Step 5: Force Pod Refresh - Strategy 2: Replace Pods
```bash
# If rolling restart too slow:

# Option A: Delete pods one-by-one (let deployment redeploy)
$ for pod in $(kubectl get pods -l app=myapp -o name); do
    echo "Deleting $pod..."
    kubectl delete $pod --grace-period=30
    kubectl wait --for=condition=ready pod -l app=myapp --timeout=2m
  done

# Option B: Scale down and up
$ kubectl scale deployment myapp --replicas=0
$ kubectl scale deployment myapp --replicas=50

# Option C: Blue-green (safest)
$ kubectl patch deployment myapp -p '{"spec":{"selector":{"matchLabels":{"version":"blue"}}}}' 
$ # (Wait for new green pods to start)
$ kubectl delete pods -l version=blue
```

#### Step 6: Verify All Pods Refreshed
```bash
# Confirm all pods have new config:
$ kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].restartCount}{"\n"}{end}'

# Check env values in all pods:
$ kubectl exec -it deployment/myapp -- env | grep CONFIG

# Or use bash loop:
$ kubectl get pods -l app=myapp -o name | while read pod; do
    echo "=== $pod ==="
    kubectl exec $pod -- cat /etc/config/app.yaml
  done | sort | uniq -c
```

#### Step 7: Fix: Implement Auto-Reload
```yaml
# Option 1: Add ConfigMap watcher sidecar

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: config
          mountPath: /etc/config
          readOnly: true
      
      # Add watcher sidecar
      - name: config-reloader
        image: jimmidyson/configmap-reload:v0.5.0
        volumes:
        - name: config
          configMap:
            name: prod-config
        volumeMounts:
        - name: config
          mountPath: /etc/config
        args:
          - --volume-dir=/etc/config
          - --volume-dir-original=/etc/config
          - --webhook-url=http://localhost:8080/reload
          - --webhook-method=POST
      
      volumes:
      - name: config
        configMap:
          name: prod-config
          defaultMode: 0644

---
# Option 2: Use Operator with auto-refresh

apiVersion: v1
kind: ConfigMap
metadata:
  name: prod-config
  labels:
    app.kubernetes.io/name: config-auto-reload
    app.kubernetes.io/version: "1"
data:
  config.yaml: |
    log_level: info
    database: postgres://db:5432

---
# Option 3: Implement app-level reload endpoint

# In your application:
# GET /admin/reload-config -> reload from ConfigMap
# Or watch ConfigMap via app code

# In deployment:
lifecycle:
  postStart:
    exec:
      command: ["/bin/sh", "-c", "while true; do inotifywait -e modify /etc/config && curl -X POST http://localhost:8080/reload; done"]
```

#### Step 8: Prevent ConfigMap Issues
```yaml
# Use Kustomize or Helm to manage ConfigMap versioning

# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

configMapGenerator:
- name: prod-config
  behavior: create
  literals:
  - log_level=info
  - db_host=postgres
  files:
  - app.yaml=config/app.yaml
  options:
    disableNameSuffixHash: false  # This adds hash to trigger pod refresh

---
# In deployment reference:
configMap:
  name: prod-config  # Actually becomes prod-config-[hash]
  # Pods automatically restart when ConfigMap changes!
```

#### Step 9: Better Application Design
```go
// Watch ConfigMap in application

package main

import (
    "log"
    "os"
    "sync"
    "time"
)

type Config struct {
    mu    sync.RWMutex
    data  map[string]string
}

func (c *Config) Watch(configPath string) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    
    for range ticker.C {
        // Re-read /etc/config/ every 5 seconds
        newConfig := loadConfig(configPath)
        c.mu.Lock()
        c.data = newConfig
        c.mu.Unlock()
        log.Println("Config reloaded")
    }
}

func (c *Config) Get(key string) string {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.data[key]
}
```

### Follow-up Questions

**Q1: How would you automate ConfigMap rotation for secrets?**
- Expected: Operators, External Secrets, periodic pull from Vault

**Q2: How do you ensure zero-downtime config updates?**
- Expected: Canary rollout, gradual propagation, health checks

**Q3: What if you need to rollback a bad ConfigMap update?**
- Expected: kubectl rollout history, restore from backup

### Red Flags

❌ "I'll just manually restart pods"
- Doesn't scale to 50+ pods

❌ "ConfigMap updates should just work automatically"
- They don't update running pod environment

❌ "I'll delete all ConfigMaps and recreate"
- Risky, could break running services

### Pro Tips

✅ "I'd use kustomize configMapGenerator with hashing for auto-refresh"

✅ "I'd add configmap-reload sidecar for applications that don't support reload"

✅ "I'd implement gradual ConfigMap rollouts using canary deployments"

✅ "I'd use External Secrets Operator for secrets that need frequent rotation"

