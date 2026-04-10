# Helm - Kubernetes Package Manager & Templating

## 1. Core Concept

**Helm** is a package manager for Kubernetes that **templatesand deploys applications**. Instead of writing raw YAML with duplicated config, Helm packages your Kubernetes manifests into reusable, configurable charts.

**Analogy:** Helm is to Kubernetes what `apt` is to Linux or `npm` is to Node.js - dependency management + easy deployment.

**Core value:**
- **Templating:** Same chart, different values = different environments
- **Dependency management:** Deploy apps + their dependencies together
- **Version management:** Track what version is deployed
- **Rollback:** Easy version rollback
- **Package sharing:** Share charts across teams/public registry

---

## 2. Internal Working (VERY IMPORTANT)

### Chart Structure

```
my-app-chart/
├── Chart.yaml           # Metadata (name, version, dependencies)
├── values.yaml          # Default values (override-able)
├── values-prod.yaml     # Production-specific overrides
├── templates/
│   ├── deployment.yaml  # K8s manifests with {{.Values}} templating
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl     # Template helpers (reusable snippets)
│   ├── NOTES.txt        # Post-install notes
│   └── tests/           # Helm test files
├── Chart.lock           # Dependency versions (like package-lock)
├── charts/              # Dependency charts
│   ├── postgresql/
│   └── redis/
└── README.md
```

### Rendering Pipeline

```
1. User runs: helm install my-release my-app-chart -f values-prod.yaml
   ↓

2. Helm merges values (in order of precedence):
   - values.yaml (defaults)
   - values-prod.yaml (override)
   - -f flags (final override)
   - --set flags (highest priority)
   
   Result: Merged configuration object (.Values)
   ↓

3. Helm processes templates/
   For each file in templates/:
   a. Read file as Go template
   b. Replace {{.Chart.Name}} → my-app-chart
   c. Replace {{.Values.replicas}} → 3
   d. Replace conditionals ({{if}})
   e. Generate Kubernetes manifest YAML
   ↓

4. Helm renders complete YAML
   spec:
     replicas: 3              # From values
     image: my-app:v2.1.0    # From values
   ↓

5. Helm either:
   - Displays (helm template / helm diff)
   - Sends to Kubernetes API (helm install / helm upgrade)

6. Kubernetes applies manifests, runs deployment
```

### Release Management

```
Release = Deployed instance of a chart

helm install my-release my-app-chart
  → Creates release named "my-release"
  → Stores release info in configmap in kube-system namespace
  → Renders templates and creates K8s resources

helm upgrade my-release my-app-chart
  → Updates existing release
  → Compares old vs new manifests
  → Only changes what's different
  → Old release preserved (can rollback)

helm history my-release
  REVISION  UPDATED                    STATUS      DESCRIPTION
  1        Mon Mar 1 12:00:00 2024    Deployed    install complete
  2        Mon Mar 8 15:30:00 2024    Deployed    upgraded to v2.1.0
  3        Mon Mar 9 10:00:00 2024    Superseded  rolled back to v1.9.0

helm rollback my-release 1
  → Reverts to revision 1
  → Redeploys all resources as they were
```

### Hooks (Lifecycle Events)

```yaml
# templates/db-migration.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    "helm.sh/hook": "pre-upgrade"           # Run before upgrading release
    "helm.sh/hook-weight": "-5"             # Order (lower = earlier)
    "helm.sh/hook-delete-policy": "before-hook-creation"
spec:
  template:
    spec:
      containers:
      - name: db-migration
        image: my-app-migration:latest
        command:
          - sh
          - -c
          - |
            python manage.py migrate
      restartPolicy: Never

---
# Hook types:
# pre-install       - Before install
# post-install      - After install
# pre-upgrade       - Before upgrade
# post-upgrade      - After upgrade
# pre-rollback      - Before rollback
# post-rollback     - After rollback
# test              - helm test command
```

---

## 3. Why Helm is Essential

### Problem Without Helm

**Scenario: Deploy same app to dev, staging, prod**

```bash
# Manual approach
kubectl apply -f deployment-dev.yaml
kubectl apply -f deployment-staging.yaml  
kubectl apply -f deployment-prod.yaml

# Problems:
# Files 90% identical (only replicas, resources differ)
# Update one, must update all - easy to miss one
# Consistency hard to maintain
# No versioning of deployment

# Rollout new version:
# Must update 3 files manually
# Risk of inconsistency
```

### What Helm Enables

✅ **Single chart, multiple environments:**
```bash
helm install dev-release my-app-chart -f values-dev.yaml
helm install staging-release my-app-chart -f values-staging.yaml
helm install prod-release my-app-chart -f values-prod.yaml

# Same chart, different configuration
# Update chart once, all environments get fix
```

✅ **Version tracking:**
```bash
helm history my-release
# See exactly what version is deployed where
```

✅ **Rollback:**
```bash
helm rollback my-release 1
# Previous version deployed immediately
```

✅ **Dependency management:**
```yaml
dependencies:
  - name: postgresql
    version: "12.1.0"
    repository: https://charts.bitnami.com/bitnami
    
# helm dependency update downloads postgresql chart
# helm install brings up app + database together
```

---

## 4. When to Use vs Not Use Helm

### Use Helm When:

- ✅ Production Kubernetes deployments
- ✅ Multiple environments with same app structure
- ✅ Apps with external dependencies (database, cache)
- ✅ Need to share charts across teams
- ✅ require version/release management

### Don't Use Helm When:

- ❌ Single simple pod (just use kubectl apply)
- ❌ Learning Kubernetes basics
- ❌ Helm adds complexity than value
- ❌ Heavy customization needed (evaluate templating language)

---

## 5. Real-world Helm Usage

### Production Chart Structure

```yaml
# Chart.yaml
apiVersion: v2
name: my-web-app
description: Production web application
type: application
version: 2.3.0
appVersion: "1.5.2"

dependencies:
  - name: postgresql
    version: "11.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "16.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
maintainers:
  - name: Platform Team
    email: platform@company.com
```

**values.yaml (defaults):**
```yaml
replicaCount: 2

image:
  repository: gcr.io/company/my-app
  pullPolicy: IfNotPresent
  tag: ""  # Empty = use Chart appVersion

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Dependencies
postgresql:
  enabled: true
  auth:
    username: appuser
    password: changeme
    database: appdb

redis:
  enabled: true
  auth:
    enabled: false
```

**values-prod.yaml (overrides):**
```yaml
replicaCount: 5  # More replicas in prod

image:
  tag: "1.5.2"  # Explicit version (not "latest")

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2
    memory: 2Gi

postgresql:
  enabled: true
  auth:
    username: produser
    # Password from Secret, not in values
    existingSecret: prod-db-secret
    database: prod_appdb
  primary:
    persistence:
      enabled: true
      size: 100Gi  # Prod needs storage
    resources:
      requests:
        memory: 2Gi

redis:
  enabled: true
  auth:
    enabled: true
    existingSecret: prod-redis-secret

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
                  - my-app
          topologyKey: kubernetes.io/hostname
```

**templates/deployment.yaml (with templating):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 10
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "my-app.fullname" . }}-db
                  key: url
            - name: REDIS_URL
              {{- if .Values.redis.enabled }}
              value: "redis://{{ include "my-app.fullname" . }}-redis:6379"
              {{- else }}
              value: {{ .Values.externalRedis.url | quote }}
              {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### Helm Deployment Workflow

```bash
# 1. Create chart
helm create my-web-app
# Creates scaffolding

# 2. Customize templates and values
# Edit templates/, values.yaml, values-prod.yaml

# 3. Fetch dependencies
helm dependency update
# Downloads postgres, redis charts to charts/

# 4. Template check (dry-run)
helm template my-release . -f values.yaml
# Shows what will be deployed (doesn't deploy)

# 5. Lint (catch errors)
helm lint
# Validates chart structure

# 6. Install to dev
helm install dev-release . -f values.yaml -n dev --create-namespace

# 7. Check deployment
helm status dev-release
kubectl logs -f deployment/my-web-app -n dev

# 8. Upgrade app version
# Edit values.yaml image.tag
helm upgrade dev-release . -f values.yaml -n dev

# 9. Verify
helm history dev-release
# Shows all revisions

# 10. Rollback if needed
helm rollback dev-release 1

# 11. Install to prod
helm install prod-release . -f values-prod.yaml -n prod --create-namespace
```

---

## 6. Common Mistakes

### Mistake 1: Storing Secrets in values.yaml

❌ **Wrong:**
```yaml
values-prod.yaml:
postgresql:
  password: "prod-database-password-123"
  
# This goes to Git!
```

✅ **Right:**
```yaml
values-prod.yaml:
postgresql:
  existingSecret: prod-db-secret
  
# Secret created separately with kubectl
kubectl create secret generic prod-db-secret \
  --from-literal=password=... -n prod
```

### Mistake 2: Hardcoding Environment

❌ **Wrong:**
```yaml
# deployment.yaml
env:
  - name: ENVIRONMENT
    value: "production"  # Hardcoded!
```

✅ **Right:**
```yaml
# values.yaml
environment: production

# templates/deployment.yaml
env:
  - name: ENVIRONMENT
    value: "{{ .Values.environment }}"
```

### Mistake 3: No Checksums for ConfigMaps

❌ **Problem:**
```bash
# Update configmap values
helm upgrade my-release ...

# But pods don't reload (ConfigMap not changed)
# Old config still running
```

✅ **Right:**
```yaml
# templates/deployment.yaml
annotations:
  checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
  
# Hash of configmap included in pod annotation
# ConfigMap changes → hash changes → pods recreated
```

---

## 7. GitOps Integration

### Helm + Flux (Modern GitOps Pattern)

```yaml
# infra/helm-release.yaml (stored in Git)
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: my-app
  namespace: production
spec:
  chart:
    repository: https://company.com/helm-charts
    name: my-app
    version: 2.3.0
  
  values:
    image:
      tag: abc123def456
  
  # Flux watches this file in Git
  # When changed, automatically runs: helm upgrade
  # Creates audit trail (who changed what)
  # Automatic rollback on failure

---
# stages/prod-app-values.yaml
replicas: 5
resources:
  limits:
    memory: 2Gi
```

**Workflow:**
```bash
1. Developer: Update app version in Git
2. CI builds: Pushes Docker image with tag v1.2.0
3. Developer: Creates PR updating:
   infra/helm-release.yaml image.tag: v1.2.0
4. Code review: Team approves
5. Merge: Commits to main
6. Flux watches: Detects change in Git
7. Auto-apply: Runs helm upgrade automatically
8. GitOps principle: Git is source of truth
```

---

## 8. Interview Questions

### Q1: Templating Strategy

**Interviewer:** "You have 50 microservices using same Helm chart. Each needs slightly different config. Design the values structure."

**Good Answer:**
```yaml
# Single chart for all services
# Each service has unique values file

services:
  - name: api-service
    replicas: 5
    image: api:v2.1.0
    
  - name: worker-service
    replicas: 10
    image: worker:v1.9.0
    
  - name: scheduler-service
    replicas: 2
    image: scheduler:v3.0.0

# Or use Helm subcharts:
charts/
├── shared-config/    # Common values
├── api-service/      # API-specific chart
├── worker-service/   # Worker-specific chart
```

### Q2: Helm vs Direct K8s

**Interviewer:** "When would you use Helm vs kubectl apply?"

**Good Answer:**
- **kubectl apply:** Simple configs, learning
- **Helm:** Production, multiple environments, dependencies
- Helm adds value when: templating, dependencies, version management matter

---

## 9. Key Takeaways

1. **Helm = package manager for Kubernetes** - Think `apt` for K8s
2. **Charts are templates** - Same chart, different values = different environments
3. **Dependency management** - Deploy apps with their services together
4. **Release tracking** - Know what version is deployed
5. **Hooks for lifecycle events** - Database migrations, etc.
6. **GitOps ready** - Integrates with Flux/ArgoCD for automation
7. **Production standard** - Use Helm for all production deployments
