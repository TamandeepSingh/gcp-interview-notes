# CI/CD Scenarios - Samsung DevOps Interview

## Scenario 1: GitHub Actions Pipeline Flakiness

### The Problem
Your GitHub Actions CI/CD pipeline passes 70% of the time, fails 30%. The failures appear random:
- Sometimes fails at "Build Docker Image" (timeout)
- Sometimes fails at "Push to Registry" (random errors)
- Sometimes passes fine
- When you re-run, often passes on second try

Your pipeline runs 50 times per day. `main` branch is blocked half the time.

**How do you identify the root cause and fix it?**

### Expected Answer (DETAILED)

#### Step 1: Collect Failure Data
```bash
# Export workflow runs to analyze:
$ gh run list --workflow=main.yml --json status,conclusion,duration,createdAt

# Export last 100 runs:
$ gh run list --limit 100 --json status,conclusion,name,createdAt \
  > workflow_runs.json

# Analyze failure patterns:
$ cat workflow_runs.json | jq '.[] | select(.conclusion == "failure") | .name' | sort | uniq -c

# Get detailed logs from failed run:
$ gh run view <run_id> --log

# Search for error patterns:
$ gh run view <run_id> --log | grep -i "error\|timeout\|connection" | head -20
```

#### Step 2: Identify Flaky Step
```bash
# Check which steps fail most often:
$ for run_id in $(gh run list --limit 50 --json databaseId --jq '.[].databaseId'); do
    gh run view $run_id --json jobs | jq '.[] | select(.conclusion == "failure") | .name'
  done | sort | uniq -c | sort -rn

# Most common failures indicate the problem
```

#### Step 3: Common Causes of Flakiness

##### Cause 1: Intermittent Network Issues (Most Common)
```yaml
# Problem: GitHub Actions runners have varying network quality
# Solution: Add retries and exponential backoff

- name: Build Docker Image
  uses: docker/build-push-action@v4
  with:
    context: .
    push: false
    tags: myapp:latest
  # No retry built-in, need to implement

# BETTER: Wrap with custom retry logic
- name: Build Docker Image (with retry)
  uses: nick-invision/retry@v2  # Third-party action
  with:
    timeout_minutes: 15
    max_attempts: 3
    retry_wait_seconds: 30
    command: |
      docker build -t myapp:latest .

# Or implement retry in shell:
- name: Build Docker Image
  run: |
    set -e
    for attempt in 1 2 3; do
      echo "Attempt $attempt..."
      if docker build -t myapp:latest .; then
        break
      fi
      if [ $attempt -lt 3 ]; then
        echo "Build failed, retrying after 30 seconds..."
        sleep 30
      else
        exit 1
      fi
    done
```

##### Cause 2: Resource Exhaustion on Runner
```yaml
# Problem: Runner CPU/memory gets full
# Solution: Add resource monitoring and parallel job limiting

jobs:
  build-matrix:
    strategy:
      max-parallel: 2  # Limit concurrent jobs
      matrix:
        node-version: [16, 18]
        
- name: Check runner resources
  run: |
    echo "Disk space:"
    df -h /
    echo "Memory:"
    free -h
    echo "CPU:"
    nproc

# Clean up before main tasks:
- name: Cleanup previous builds
  run: |
    docker system prune -f
    docker volume prune -f
    docker builder prune -f
```

##### Cause 3: Timeout Too Aggressive
```yaml
# Problem: Operations like tests or builds take variable time
# Solution: Increase timeout incrementally

- name: Run Integration Tests (with timeout)
  timeout-minutes: 45  # Increase if consistently timing out
  run: |
    pytest --timeout=40 tests/integration

# Monitor actual execution times:
- name: Report execution time
  run: |
    echo "Total step duration: $SECONDS seconds"
```

##### Cause 4: Dependent Service Flakiness
```yaml
# Problem: Services started in CI not ready
# Solution: Add health checks and retry logic

- name: Start Docker services
  run: |
    docker-compose -f docker-compose.test.yml up -d
    
    # Health check: postgres ready?
    for i in {1..30}; do
      if docker exec postgres_test pg_isready -U test; then
        echo "Postgres ready"
        break
      fi
      if [ $i -eq 30 ]; then
        echo "Postgres failed to start"
        exit 1
      fi
      echo "Waiting for postgres ($i/30)..."
      sleep 2
    done

- name: Run tests
  run: pytest tests/

- name: Stop services
  run: docker-compose -f docker-compose.test.yml down
```

##### Cause 5: Cache Corruption
```yaml
# Problem: GitHub Actions cache becomes invalid/corrupted
# Solution: Version your cache and implement cache invalidation

- uses: actions/setup-node@v3
  with:
    node-version: '18'
    cache: 'npm'
    cache-dependency-path: 'package-lock.json'  # Ensure exact match

# Explicit cache with version:
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-v1-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-v1-

# Add cache cleanup:
- name: Clean npm cache
  run: npm cache clean --force
```

#### Step 4: Implement Comprehensive Fixes
```yaml
name: CI/CD Pipeline - Production

on: [push, pull_request]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest-8-cores  # Premium runner for stability
    timeout-minutes: 45
    
    steps:
      # Pre-checks
      - uses: actions/checkout@v3
      
      - name: Check runner health
        run: |
          echo "Available disk: $(df -h / | tail -1 | awk '{print $4}')"
          echo "Available memory: $(free -h | grep Mem | awk '{print $7}')"
      
      - name: Clean Docker resources
        run: docker system prune -af --volumes
      
      # Caching
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: npm
          cache-dependency-path: package-lock.json
      
      # Build with retry
      - name: Install dependencies
        uses: nick-invision/retry@v2
        with:
          max_attempts: 3
          timeout_minutes: 10
          command: npm ci --prefer-offline
      
      - name: Run linter
        uses: nick-invision/retry@v2
        with:
          max_attempts: 2
          timeout_minutes: 10
          command: npm run lint
      
      # Build
      - name: Build application
        uses: nick-invision/retry@v2
        with:
          max_attempts: 3
          timeout_minutes: 15
          retry_wait_seconds: 30
          command: npm run build
      
      # Test
      - name: Run unit tests
        uses: nick-invision/retry@v2
        with:
          max_attempts: 2
          timeout_minutes: 20
          command: npm run test:unit
      
      # Docker build
      - name: Build Docker image
        uses: nick-invision/retry@v2
        with:
          max_attempts: 3
          timeout_minutes: 20
          retry_wait_seconds: 60
          command: docker build -t myapp:${{ github.sha }} .
      
      # Report
      - name: Report metrics
        if: always()
        run: |
          echo "Job duration: ${{ job.duration }}"
          docker ps -a
          df -h /
```

#### Step 5: Monitor and Alert
```bash
# Set up GitHub API monitoring:
$ cat << 'EOF' > monitor_workflow.sh
#!/bin/bash
# Monitor workflow success rate hourly

REPO="owner/repo"
WORKFLOW_ID="main.yml"

SUCCESS=0
TOTAL=0

for i in {1..100}; do
  RESULT=$(gh run list --repo $REPO --workflow $WORKFLOW_ID \
    --limit 1 --json conclusion --jq '.[0].conclusion')
  
  [[ $RESULT == "success" ]] && ((SUCCESS++))
  ((TOTAL++))
done

RATE=$((SUCCESS * 100 / TOTAL))
echo "Success rate: $RATE%"

# Alert if below 85%
if [ $RATE -lt 85 ]; then
  echo "ALERT: Success rate dropped to $RATE%" | mail -s "CI/CD Alert" ops@example.com
fi
EOF

# Run hourly via cron
0 * * * * /path/to/monitor_workflow.sh
```

### Follow-up Questions

**Q1: How do you handle flakiness in production deployment pipelines?**
- Expected: Blue-green, canary, rollback strategies, circuit breakers

**Q2: What metrics would you monitor for a healthy pipeline?**
- Expected: Success rate, duration, flakiness per step, resource usage

**Q3: How often should you test your disaster recovery?**
- Expected: Regularly, automated, integrated into pipeline

### Red Flags

❌ "We just re-run failed jobs manually"
- Doesn't scale, band-aid solution

❌ "Flakiness is normal in CI/CD"
- Sign of deeper infrastructure problems

❌ "I'll ignore failures that pass on retry"
- These are real problems masked by retries

### Pro Tips

✅ "I'd use premium runners (more CPU/memory) for stability"

✅ "I'd implement structured retry with exponential backoff"

✅ "I'd monitor success rate and alert on decline"

✅ "I'd separate flaky tests to a separate job with retries"

---

## Scenario 2: Secrets Management Breach Detection

### The Problem
Your team uses GitHub Actions to deploy to production. A junior engineer accidentally commits an AWS access key to a public repository. The key:
- Has full AWS permissions
- Has been in the repo for 3 hours before noticed
- Could have been in GitHub's public cache

**What's your incident response?**

### Expected Answer (DETAILED)

#### Step 1: Immediate Containment (First 5 minutes)
```bash
# STOP: Immediately revoke the key
$ aws iam list-access-keys --user-name jenkins-ci

# Check which key was exposed:
$ aws iam get-access-key-last-used --access-key-id AKIAIOSFODNN7EXAMPLE

# Deactivate immediately:
$ aws iam update-access-key-status \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --user-name jenkins-ci \
  --status Inactive

# Delete the key:
$ aws iam delete-access-key \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --user-name jenkins-ci

# Generate new key:
$ aws iam create-access-key --user-name jenkins-ci
```

#### Step 2: Audit for Unauthorized Access
```bash
# Check CloudTrail for API calls from this key:
$ aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAIOSFODNN7EXAMPLE \
  --max-results 50 | jq '.Events[] | {Time: .EventTime, EventName, ErrorCode, ErrorMessage}'

# Check for suspicious activities:
$ aws s3 ls # Did attacker access S3?
$ aws ec2 describe-instances # Were instances launched?
$ aws iam get-user # Was account modified?

# Check for resource creation:
$ aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId, LaunchTime]' | grep -i "now\|recent"
```

#### Step 3: Remove from Git History
```bash
# Option 1: If still in unpushed commits
$ git log --oneline | head -5
$ git reset --soft HEAD~1
$ git add .
$ git commit -m "Remove secrets"

# Option 2: If already pushed (use BFG or git-filter-branch)
$ git clone --mirror <repo-url> <repo-mirror>
$ cd <repo-mirror>

# Use BFG to remove secrets:
$ bfg --delete-files <your-secret-file>

# Or use git-filter-repo:
$ git filter-repo --path secrets.txt --invert-paths

# Push force push clean history:
$ git push --force --all
$ git push --force --tags
```

#### Step 4: Check GitHub for Exposure
```bash
# Check if secret was indexed by GitHub:
$ curl -s https://api.github.com/search/code?q=AKIAIOSFODNN7EXAMPLE

# If found in multiple places:
# - Report to GitHub security team
# - Check all forks and mirrors

# Revoke GitHub token too:
$ gh auth logout
$ gh auth logout --all
```

#### Step 5: Implement Prevention Measures
```yaml
# Add to GitHub Actions workflow:

- name: Secret scanning
  uses: gitleaks/gitleaks-action@v2

# Or use TruffleHog:
- name: TruffleHog Secret Scan
  uses: trufflesecurity/trufflehog@main
  with:
    path: ./
    base: ${{ github.event.repository.default_branch }}
    head: HEAD
    extra_args: --json --only-verified

# Pre-commit hook:
$ cat << 'EOF' > .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.1.0
    hooks:
      - id: detect-secrets
EOF
```

#### Step 6: Implement Secrets Management Best Practices
```yaml
# GitHub Secrets - never commit
- name: Deploy with GitHub Secrets
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  run: |
    # Never print secrets
    echo "Deploying..."
    terraform apply -auto-approve

# Better: Use OIDC federation (no secrets needed)
- uses: aws-actions/configure-aws-credentials@v2
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions
    aws-region: us-east-1

# Even better: Use HashiCorp Vault or AWS Secrets Manager
- name: Fetch secret from AWS Secrets Manager
  run: |
    SECRET=$(aws secretsmanager get-secret-value \
      --secret-id prod/db-password \
      --query SecretString \
      --output text)
    # Use $SECRET but don't print
```

#### Step 7: Communication and Post-Mortem
```bash
# Notify affected teams immediately:
# - Security team
# - DevOps team
# - Engineering leads
# - Customers (if applicable)

# Create incident report:
$ cat << 'EOF' > incident_report.md
# Security Incident: Exposed AWS Key

## Timeline
- 2024-01-15 10:30: Key committed to public repo
- 2024-01-15 13:27: Incident discovered
- 2024-01-15 13:30: Key revoked
- 2024-01-15 13:35: Unauthorized access checked (NONE FOUND)
- 2024-01-15 14:00: Git history cleaned

## Root Cause
Junior engineer didn't use `.gitignore` properly

## Actions Taken
1. Revoked AWS access key
2. Generated new key
3. Cleaned git history
4. Implemented pre-commit hooks
5. Added secret scanning to GitHub Actions

## Prevention
- Require pre-commit hooks
- Automated secret scanning in CI/CD
- Enforce OIDC federation instead of long-lived keys
- Team security training on secrets management

## Lessons Learned
- No secrets should ever be committed
- Detection time was 2h 57m - need faster detection
- Audit trail showed no unauthorized access
EOF
```

### Follow-up Questions

**Q1: How would you verify that the key wasn't used?**
- Expected: CloudTrail analysis, AWS CloudWatch logs, checking resource creation times

**Q2: What's the different approach if this was a staging key?**
- Expected: Still revoke, but lower urgency; might not need incident response

**Q3: How do you train engineers to prevent this?**
- Expected: Automated tooling > manual training; make it impossible before educating

### Red Flags

❌ "We don't have secret scanning enabled"
- This is now table stakes

❌ "The key only had limited permissions anyway"
- No such thing as "limited" when exposed

❌ "We'll rotate keys later"
- Every second counts in a breach

### Pro Tips

✅ "I'd use GitHub's secret scanning with custom patterns"

✅ "I'd implement OIDC federation to eliminate long-lived secrets"

✅ "I'd add pre-commit hooks with detect-secrets to block commits"

✅ "I'd require all teams to complete security training"

---

## Scenario 3: Deployment Fails at 2 AM - You're On-Call

### The Problem
Your production deployment started via GitHub Actions. It's now 2:06 AM and your on-call alert fires:
```
DEPLOYMENT FAILED: prod-api deployment failed to reach stable state
Container image push succeeded
Kubernetes rollout failed: ImagePullBackOff
```

The problem:
- New container image exists and was pushed successfully
- But pods in Kubernetes can't pull the image
- You have 5 minutes to decide: rollback or investigate?

**What's your incident response?**

### Expected Answer (DETAILED)

#### Step 1: Quick Diagnosis (First 90 seconds)
```bash
# Check the pod status IMMEDIATELY:
$ kubectl get deployment prod-api -o yaml | grep image

# Get full pod events:
$ kubectl describe pod <prod-api-pod> | grep -A5 "Events:"

# Check image pull secrets:
$ kubectl get secret -n prod | grep docker

# Check if image exists in registry:
$ gcloud container images describe \
  gcr.io/my-project/prod-api:abc123def456
```

#### Step 2: Immediate Decision
```
Is production currently broken?
- YES: ROLLBACK IMMEDIATELY (restore previous deployment)
- NO (older pods still running): Investigate while live

If old pods still serving traffic, buy yourself time to investigate
If COMPLETE outage, rollback first, diagnose later
```

#### Step 3: Quick Rollback Option
```bash
# IF DECISION IS ROLLBACK:
$ kubectl rollout undo deployment/prod-api -n prod

# Verify rollback:
$ kubectl rollout status deployment/prod-api -n prod

# Confirm traffic restored:
$ curl -s https://api.prod.example.com/health | jq .

# Now you have time to investigate what went wrong
```

#### Step 4: Parallel Investigation
```bash
# While rollback is happening:

# Check Docker Hub / Registry API:
$ docker manifest inspect \
  gcr.io/my-project/prod-api:abc123def456

# Check if image layer exists:
$ gcloud container images describe \
  gcr.io/my-project/prod-api:abc123def456 \
  --show-package-vulnerability

# Check image size - did it build correctly?
$ gcloud container images describe \
  gcr.io/my-project/prod-api:abc123def456 \
  --format="value(image_summary.image_size_bytes)"
```

#### Step 5: Root Cause Analysis
```
Common causes of ImagePullBackOff:

1. Image tag doesn't exist
   -> Check tag in deployment manifest
   -> Verify Docker build actually created image

2. ImagePullSecrets missing or wrong
   -> Check namespace: kubectl get secrets -n prod
   -> Check secret content: kubectl describe secret docker-creds -n prod
   -> Verify registry credentials are valid

3. Image registry access blocked
   -> Network policy blocking registry access
   -> Firewall blocking GCR/ECR
   -> Registry authentication failed

4. Image pulled from wrong registry
   -> Check imagePullPolicy in deployment
   -> Verify image URL is correct

5. Registry outage
   -> Check status page: status.docker.com or status.cloud.google.com
```

#### Step 6: Prevent Next Time - CD Pipeline Review
```yaml
# Add validation BEFORE deploying

name: Deploy Production

on: [workflow_dispatch]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      # 1. Build image
      - name: Build Docker image
        run: |
          docker build -t gcr.io/my-project/prod-api:$GITHUB_SHA .
          docker tag gcr.io/my-project/prod-api:$GITHUB_SHA \
                     gcr.io/my-project/prod-api:latest
      
      # 2. TEST the image before pushing
      - name: Test image locally
        run: |
          docker run --rm gcr.io/my-project/prod-api:$GITHUB_SHA \
            /app/bin/health-check
      
      # 3. Push image
      - name: Push to registry
        run: |
          docker push gcr.io/my-project/prod-api:$GITHUB_SHA
          docker push gcr.io/my-project/prod-api:latest
      
      # 4. Verify image in registry
      - name: Verify image pushed
        run: |
          gcloud container images describe \
            gcr.io/my-project/prod-api:$GITHUB_SHA
      
      # 5. Update deployment
      - name: Update Kubernetes deployment
        run: |
          kubectl set image deployment/prod-api \
            prod-api=gcr.io/my-project/prod-api:$GITHUB_SHA \
            -n prod
      
      # 6. Wait for rollout
      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/prod-api -n prod \
            --timeout=5m
      
      # 7. Health check
      - name: Health check
        uses: nick-invision/retry@v2
        with:
          max_attempts: 5
          timeout_minutes: 2
          command: |
            curl -f https://api.prod.example.com/health || exit 1
      
      # 8. Alert if everything OK
      - name: Success notification
        if: success()
        run: echo "Deployment successful"
      
      # 9. If failed, rollback automatically
      - name: Automatic rollback on failure
        if: failure()
        run: |
          kubectl rollout undo deployment/prod-api -n prod
          kubectl rollout status deployment/prod-api -n prod
```

#### Step 7: Documentation and Learnings
```bash
# Create post-incident documentation:
$ cat << 'EOF' > incidents/2024-01-15-imagepullbackoff.md
# Incident: ImagePullBackOff During Deployment

## Timeline
- 02:05 AM: Deployment started
- 02:06 AM: Alert triggered
- 02:07 AM: Rollback initiated
- 02:08 AM: Service recovered

## Root Cause
Image push to registry succeeded but service account in K8s didn't have pull permissions

## Why Not Caught Earlier
- No image verification step in CD pipeline
- No pre-deployment validation

## Fixes Implemented
1. Added image verification step to CD pipeline
2. Added health check after deployment
3. Implemented automatic rollback on failure
4. Added alert for ImagePullBackOff status

## Prevention
- All image validation must pass before deployment
- Health check required post-deployment
- Deployment timeout set to 5 minutes

## Action Items
- [ ] Review all ServiceAccounts for proper RBAC
- [ ] Add image scanning to build process
- [ ] Update runbook for this scenario
- [ ] Team training on Kubernetes image pull errors
EOF
```

### Follow-up Questions

**Q1: How would you prevent this specific issue from happening?**
- Expected: Image verification, RBAC validation, health checks

**Q2: When should you rollback vs. investigate?**
- Expected: Rollback if production is down; investigate if serving stale version

**Q3: How do you balance speed with safety in incident response?**
- Expected: Rollback fast, investigate thoroughly post-incident

### Red Flags

❌ "I would investigate first, then rollback if needed"
- Could extend user impact significantly

❌ "This never happens to us"
- Familiarity breeds complacency

❌ "I'll restart the pod and see if it helps"
- No understanding of root cause

### Pro Tips

✅ "I'd check if old pods still running -> decides immediate rollback"

✅ "I'd have automated rollback on deployment failure"

✅ "I'd implement image verification before pushing to registry"

✅ "I'd have post-deployment health checks that block rollout"

