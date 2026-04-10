# Cloud Build

## 1. Core Concept (Deep Explanation)

Cloud Build is Google's **fully managed CI/CD service** that builds, tests, and deploys applications. It:

- **Runs build steps**: Sequential or parallel Docker containers
- **Integrates with repos**: GitHub, Cloud Source Repositories, GitLab
- **Triggers deployments**: Git push/PR → Auto build → Deploy
- **Caches builds**: Layer caching reduces build time by 80%+
- **Artifact storage**: Push images to Artifact Registry

**Internal Architecture:**
- Build steps execute in containers (you control the image)
- Shared build cache across pipeline (layer reuse)
- Artifact storage in Container Registry (deprecated) or Artifact Registry
- Webhooks for GitHub/GitLab integration
- Cloud Build API for programmatic builds

## 2. Why This Exists

**Problems it solves:**
- No Jenkins server to manage
- Automated testing on every commit
- Consistent builds (same environment every time)
- Security: Build runs in isolated container
- Cost: Pay per build minute, zero idle costs

## 3. When To Use

**Every production system needs:**
- Automated unit/integration tests
- Container image builds
- Deployment to GKE/Cloud Run/Compute Engine
- Configuration deployment (Terraform, Helm)

## 4. When NOT To Use

**Avoid if:**
- Simple scripts (use Cloud Functions)
- Long builds (>2 hours, Cloud Build timeout)
- Need full VM access (use Jenkins on GCE)

## 5. Real-World Example: Docker Build → Push → Deploy

```yaml
# cloudbuild.yaml - Triggers on Git push
steps:

# Step 1: Run unit tests
- name: 'gcr.io/cloud-builders/docker'
  args: ['run', '--rm', '-v', '${_WORKSPACE}:/workspace', 
         'python:3.9', 'bash', '-c', 
         'cd /workspace && python -m pytest tests/']
  id: 'unit-tests'

# Step 2: Build Docker image
- name: 'gcr.io/cloud-builders/docker'
  args:
  - 'build'
  - '-t'
  - 'gcr.io/$PROJECT_ID/${_SERVICE_NAME}:${SHORT_SHA}'
  - '-t'
  - 'gcr.io/$PROJECT_ID/${_SERVICE_NAME}:latest'
  - '.'
  id: 'build-image'
  waitFor: ['unit-tests']

# Step 3: Push to Artifact Registry
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/${_SERVICE_NAME}:${SHORT_SHA}']
  id: 'push-image'

# Step 4: Deploy to Cloud Run
- name: 'gcr.io/cloud-builders/gke-deploy'
  args:
  - run
  - --filename=k8s/
  - --image=gcr.io/$PROJECT_ID/${_SERVICE_NAME}:${SHORT_SHA}
  - --location=us-central1
  - --cluster=prod-cluster
  id: 'deploy-gke'
  waitFor: ['push-image']

# Step 5: Run smoke tests
- name: 'gcr.io/cloud-builders/kubectl'
  args: ['rollout', 'status', 'deployment/${_SERVICE_NAME}', '-n', 'production']
  env: ['CLOUDSDK_COMPUTE_ZONE=us-central1-a', 'CLOUDSDK_CONTAINER_CLUSTER=prod-cluster']
  id: 'smoke-tests'
  waitFor: ['deploy-gke']

# On failure, automatic rollback
onFailure:
- rollbackToPreviousVersion

# Store images
images:
- 'gcr.io/$PROJECT_ID/${_SERVICE_NAME}:${SHORT_SHA}'
- 'gcr.io/$PROJECT_ID/${_SERVICE_NAME}:latest'

# Substitutions
substitutions:
  _SERVICE_NAME: 'api-service'
  _WORKSPACE: '.'

# Build settings
options:
  machineType: 'N1_HIGHCPU_8'
  logging: CLOUD_LOGGING_ONLY
  
timeout: '1800s'
tags: ['gke-deploy', '${_SERVICE_NAME}', '${SHORT_SHA}']
```

## 6. Architecture Thinking

**Cloud Build in a CI/CD Pipeline:**

```
┌──────────────────────┐
│ Developer Push       │
│ git push origin main │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────┐
│ GitHub / Cloud Repo      │
│ Webhook triggers build   │
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────────────┐
│ Cloud Build                      │
│ ├─ Step 1: Run unit tests       │
│ ├─ Step 2: Build Docker image   │
│ ├─ Step 3: Push to registry     │
│ ├─ Step 4: Deploy to GKE        │
│ └─ Step 5: Run smoke tests      │
└──────┬───────────────┬───────────┘
       │               │
       ▼               ▼
   SUCCESS         FAILURE
       │               │
    Store logs    Notify Slack
       │           Rollback
       ▼
   Production
```

**Build step execution model:**

```
Step 1: Unit Tests
  └─ Docker container with test runner
  └─ Tests run, generate report

Step 2, 3, 4 (Parallel)
  ├─ Build Docker (depends on Step 1 success)
  ├─ No dependency: Run in parallel if no dependency
  └─ Push to registry

Step 5: Deploy
  └─ Depends on Step 3 (push completes)
```

## 7. Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Building on every branch | Wasted build minutes/costs | Use branch filtering (`includedBranchNames`) |
| No build caching | Docker layer rebuild every time | Enable Docker cache, use buildpacks |
| Large Docker images | Slow push/pull, high storage | Multistage builds, distroless images |
| Building monolith on every commit | Unnecessary builds for unrelated changes | Use path filters (only build if src/ changes) |
| No timeout set | Hung build blocks pipeline | Always set timeout (e.g., 1800s) |
| No secret management | Credentials in code | Use Secret Manager, inject at build time |
| Pushing to public registry | All images visible to internet | Use Artifact Registry, private repos |
| No artifact retention | Old images accumulate, waste storage | Set retention policy (keep last 10 images) |
| Sequential builds | Slower pipeline | Parallelize non-dependent steps with `waitFor` |
| Not caching build dependencies | Python pip/npm reinstalls every time | Cache container layers with Kaniko |

## 8. Interview Questions (with Answers)

### Q1: Design a CI/CD pipeline that builds, tests, and deploys to multiple environments.

**Answer:**
```yaml
# cloudbuild.yaml - Multi-environment deployment
steps:

# Always run tests
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'test-image', '-f', 'Dockerfile.test', '.']
  id: 'run-tests'

- name: 'test-image'
  args: ['npm', 'test']
  id: 'tests'
  waitFor: ['run-tests']

# Build production image
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/app:${SHORT_SHA}', '.']
  id: 'build-prod'
  waitFor: ['tests']

# Push to staging
- name: 'gcr.io/cloud-builders/docker'
  args: ['tag', 'gcr.io/$PROJECT_ID/app:${SHORT_SHA}', 'gcr.io/$PROJECT_ID/app:staging']
  id: 'tag-staging'
  waitFor: ['build-prod']

# Deploy to staging
- name: 'gcr.io/cloud-builders/kubectl'
  args: ['set', 'image', 'deployment/app', 'app=gcr.io/$PROJECT_ID/app:staging']
  env: 
    - 'CLOUDSDK_CONTAINER_CLUSTER=staging-cluster'
    - 'CLOUDSDK_COMPUTE_ZONE=us-central1-a'
  id: 'deploy-staging'
  waitFor: ['tag-staging']

# Manual approval for prod (pause pipeline)
- name: 'gcr.io/cloud-builders/gke-deploy'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    echo "Waiting for approval..."
    # In reality, use Cloud Tasks for async approval
    # For now, just continue to prod if staging passed

# Deploy to production
- name: 'gcr.io/cloud-builders/kubectl'
  args: ['set', 'image', 'deployment/app', 'app=gcr.io/$PROJECT_ID/app:${SHORT_SHA}']
  env:
    - 'CLOUDSDK_CONTAINER_CLUSTER=prod-cluster'
    - 'CLOUDSDK_COMPUTE_ZONE=us-central1-a'
  id: 'deploy-prod'
  waitFor: ['deploy-staging']
```

### Q2: How would you implement blue-green deployment with Cloud Build?

**Answer:**
```yaml
# Blue-green deployment strategy
steps:

# Green deployment (new version)
- name: 'gcr.io/cloud-builders/kubectl'
  args:
  - 'set'
  - 'image'
  - 'deployment/app-green'
  - 'app=gcr.io/$PROJECT_ID/app:${SHORT_SHA}'
  env:
    - 'CLOUDSDK_CONTAINER_CLUSTER=prod-cluster'
    - 'CLOUDSDK_COMPUTE_ZONE=us-central1-a'
  id: 'deploy-green'

# Run smoke tests on green
- name: 'gcr.io/cloud-builders/apache-airflow'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    #!/bin/bash
    GREEN_LB=$(kubectl get svc app-green -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    for i in {1..30}; do
      HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://$GREEN_LB/health)
      if [[ $HTTP_CODE -eq 200 ]]; then
        echo "Green deployment healthy"
        exit 0
      fi
      sleep 10
    done
    echo "Green deployment failed health check"
    exit 1
  id: 'smoke-test-green'
  waitFor: ['deploy-green']

# Switch traffic from blue to green
- name: 'gcr.io/cloud-builders/kubectl'
  args:
  - 'patch'
  - 'service'
  - 'app-service'
  - '-p'
  - '{"spec":{"selector":{"deployment":"app-green"}}}'
  env:
    - 'CLOUDSDK_CONTAINER_CLUSTER=prod-cluster'
    - 'CLOUDSDK_COMPUTE_ZONE=us-central1-a'
  id: 'switch-traffic'
  waitFor: ['smoke-test-green']

# Keep blue ready for instant rollback
onFailure:
- name: 'gcr.io/cloud-builders/kubectl'
  args:
  - 'patch'
  - 'service'
  - 'app-service'
  - '-p'
  - '{"spec":{"selector":{"deployment":"app-blue"}}}'
  # Automatic rollback if green failed
```

### Q3: Optimize build time from 15 minutes to 5 minutes.

**Answer:**
```yaml
# Performance optimization techniques

# 1. Cache Docker layers
- name: 'gcr.io/cloud-builders/docker'
  args:
  - 'build'
  - '--cache-from=gcr.io/$PROJECT_ID/app:latest'
  - '-t'
  - 'gcr.io/$PROJECT_ID/app:${SHORT_SHA}'
  - '.'
  # Result: ~60% faster (layer reuse)

# 2. Use multistage build (Dockerfile)
# FROM python:3.9 as builder
# WORKDIR /app
# COPY requirements.txt .
# RUN pip install -r requirements.txt
#
# FROM python:3.9
# COPY --from=builder /app /app
# Result: ~40% smaller image

# 3. Parallel build steps
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'test-image', '.']
  id: 'tests'
  waitFor: ['-']  # Begin immediately (no dependencies)

- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'lint-image', '.']
  id: 'lint'
  waitFor: ['-']  # Begin immediately (parallel with tests)

# 4. Use Cloud Build cache
- name: 'gcr.io/cloud-builders/docker'
  args:
  - 'run'
  - '--rm'
  - '-v'
  - '/cache/pip:/root/.cache/pip'
  - 'python:3.9'
  - 'pip'
  - 'install'
  - '-r'
  - 'requirements.txt'
  # Reuses pip cache between builds

options:
  machineType: 'N1_HIGHCPU_8'  # Faster build machine
  logging: CLOUD_LOGGING_ONLY

# Result: 15min → 5min (multistage + cache + parallelization)
```

### Q4: Handle secrets securely in Cloud Build.

**Answer:**
```yaml
# Method 1: Cloud Secret Manager
steps:

# Get secret and inject
- name: 'gcr.io/cloud-builders/docker'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    gcloud secrets versions access latest --secret="database-password" > /workspace/db_pass.txt
  id: 'get-secret'

# Use secret in build
- name: 'gcr.io/cloud-builders/docker'
  args:
  - 'build'
  - '--build-arg'
  - 'DB_PASSWORD=$(cat /workspace/db_pass.txt)'
  - '-t'
  - 'gcr.io/$PROJECT_ID/app:latest'
  - '.'
  id: 'build'
  waitFor: ['get-secret']

# Method 2: Secure secret in cloudbuild.yaml
- name: 'gcr.io/cloud-builders/docker'
  args:
  - 'build'
  - '--secret'
  - 'id=db_password,src=/dev/stdin'
  - '-t'
  - 'gcr.io/$PROJECT_ID/app:latest'
  - '.'
  env:
  - 'DB_PASSWORD=$_DB_PASSWORD'  # From Cloud Build secret
  id: 'build-secure'

# Declare secret substitutions
availableSecrets:
  secretManager:
  - versionName: projects/$PROJECT_ID/secrets/database-password/versions/latest
    env: '_DB_PASSWORD'
  - versionName: projects/$PROJECT_ID/secrets/api-key/versions/latest
    env: '_API_KEY'

# Never log secrets
options:
  logging: CLOUD_LOGGING_ONLY  # Doesn't log build args/env
```

### Q5: Configure automatic builds on Git push (webhook).

**Answer:**
```bash
# Create build trigger
gcloud builds triggers create github \
  --name="auto-build-main" \
  --repo-name="my-repo" \
  --repo-owner="my-org" \
  --branch-pattern="^main$" \
  --build-config="cloudbuild.yaml" \
  --substitutions="_ENVIRONMENT=prod" \
  --include-logs-with-status

# Filter to only build on specific paths
gcloud builds triggers update auto-build-main \
  --included-files="src/**,Dockerfile" \
  --ignored-files="README.md,docs/**"

# Build on PR comments
gcloud builds triggers update auto-build-main \
  --pull-request-pattern=".*"

# Scheduled builds
gcloud builds create --config=cloudbuild.yaml \
  --substitutions="_SCHEDULE=daily-backup"
```

### Q6: Troubleshoot: Build fails with "Docker image not found" error.

**Answer:**
```bash
# 1. Check Artifact Registry exists
gcloud artifacts repositories list --location=us-central1

# 2. Check permissions (Cloud Build service account)
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --format='table(bindings.role)' \
  --filter="bindings.members:*@cloudbuild.gserviceaccount.com"

# 3. Grant permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:PROJECT_ID@cloudbuild.gserviceaccount.com \
  --role=roles/artifactregistry.writer

# 4. Verify image push
gcloud artifacts docker images list us-central1-docker.pkg.dev/PROJECT_ID/repository-id

# 5. Check Dockerfile path
cat cloudbuild.yaml | grep "dockerfile:"
```

### Q7: Implement canary deployment with Cloud Build.

**Answer:**
```yaml
# Canary Strategy: 10% → 50% → 100%
steps:

# Deploy canary (10% traffic)
- name: 'gcr.io/cloud-builders/kubectl'
  args:
  - 'patch'
  - 'deployment'
  - 'app-canary'
  - '-p'
  - '{"spec":{"replicas":1}}'
  env:
    - 'CLOUDSDK_CONTAINER_CLUSTER=prod-cluster'
  id: 'deploy-canary'

# Monitor canary for 5 minutes
- name: 'gcr.io/cloud-builders/gke-deploy'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    echo "Monitoring canary deployment..."
    sleep 300
    
    # Check error rates
    ERROR_RATE=$(gcloud monitoring time-series list \
      --filter 'metric.type="custom.googleapis.com/error_rate" AND metric.labels.deployment="app-canary"' \
      --format='value(points.value.double_val)' | head -1)
    
    if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
      echo "Canary error rate too high: $ERROR_RATE"
      exit 1
    fi
    
    echo "Canary healthy, proceeding with ramp-up"
  id: 'monitor-canary'
  waitFor: ['deploy-canary']

# Scale to 50%
- name: 'gcr.io/cloud-builders/kubectl'
  args:
  - 'patch'
  - 'deployment'
  - 'app-canary'
  - '-p'
  - '{"spec":{"replicas":5}}'
  id: 'scale-50'
  waitFor: ['monitor-canary']

# Final scale to 100%
- name: 'gcr.io/cloud-builders/kubectl'
  args:
  - 'patch'
  - 'deployment'
  - 'app-production'
  - '-p'
  - '{"spec":{"replicas":10}}'
  id: 'scale-100'
  waitFor: ['scale-50']
```

### Q8: Security best practices in Cloud Build.

**Answer:**
```yaml
# 1. Use specific base images (not 'latest')
- name: 'python:3.9-slim@sha256:...'  # Use digest, not tag

# 2. Don't run as root
dockerfile: 'Dockerfile'
# Dockerfile:
# USER nobody  # Non-root user

# 3. Scan images for vulnerabilities
- name: 'gcr.io/cloud-builders/gke-deploy'
  args: ['scan', 'image', 'gcr.io/$PROJECT_ID/app:${SHORT_SHA}']

# 4. Sign images
- name: 'gcr.io/cloud-builders/docker'
  args:
  - 'build'
  - '-t'
  - 'gcr.io/$PROJECT_ID/app:${SHORT_SHA}'
  - '.'

# 5. Enable Binary Authorization
- name: 'gcr.io/cloud-builders/gke-deploy'
  args:
  - 'run'
  - '--filename=k8s/'
  - '--image=gcr.io/$PROJECT_ID/app:${SHORT_SHA}'
  - '--binary-authorization=ENABLED'

# 6. Audit logs
options:
  logging: CLOUD_LOGGING_ONLY  # Send to Cloud Logging
  
# 7. Use separate service account
serviceAccount: 'projects/$PROJECT_ID/serviceAccounts/cloud-build-sa@$PROJECT_ID.iam.gserviceaccount.com'
```

### Q9: Cost optimization - reduce Cloud Build minutes.

**Answer:**
```
Current: 100 builds/day × 10 minutes = 1000 minutes = $3.34/day

Optimize:

1. Branch filtering (build main only, not every branch)
   → 50 builds/day = 500 minutes

2. Layer caching (skip rebuild)
   → 7 min per build = 350 minutes

3. Parallel steps (instead of sequential)
   → 5 min per build = 250 minutes

4. Only build on code changes (ignore docs)
   → 40 builds/day = 200 minutes

Result: 1000 min → 200 min = 40% cost reduction
Cost: $3.34/day → $0.67/day
```

### Q10: Compare Cloud Build vs local Jenkins.

**Answer:**

| Cloud Build | Jenkins |
|-----------|-----------|
| Fully managed | Self-managed VMs |
| $0.003/minute | Higher (VM + maintenance) |
| Scales automatically | Manual scaling |
| Integrated with GCP services | Requires plugins |
| GitHub/GitLab webhooks | More setup required |
| Audit trails built-in | Extra configuration |
| Ideal for GCP | More flexible for multi-cloud |

## 9. Advanced Insights (Senior-level)

### Performance Optimization

**Build time reduction matrix:**
- Docker layer caching: 50-80% savings
- Parallel execution: 30-40% savings
- Smaller base images: 20-30% savings
- Build machine type: e2-highcpu better than e2-standard

### Cost Optimization

**Monthly cost formula:**
- $0.0033/build minute
- 100 builds/day × 5 min = 500 min = $1.65/day = $50/month

### Security Framework

**Zero-trust build:**
1. Ephemeral VMs (no persistence)
2. Container isolation
3. Signed images
4. Secret Manager integration
5. Audit logging

