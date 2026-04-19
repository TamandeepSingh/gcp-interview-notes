# Fix Broken YAML — Interview Practice

## Round 1: Wrong targetPort

**Question:** Your app is deployed but not accessible. What is wrong with this YAML?

**Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 3000
```

**Deployment:**
```yaml
containers:
  - name: app
    image: nginx
    ports:
      - containerPort: 80
```

**What is wrong?**

The service is forwarding traffic to port `3000`, but the container is running on port `80`. Traffic never reaches the application.

**Fix:**
```yaml
targetPort: 80
```

**Strong interview line:**
> "I always ensure the service `targetPort` matches the `containerPort` defined in the deployment."

---

## Round 2: Label Mismatch

**Question:** Why is the service not working?

**Service selector:**
```yaml
selector:
  app: my-app
```

**Pod labels:**
```yaml
labels:
  app: hello-app
```

**What is wrong?**

The service selector (`app: my-app`) does not match the pod labels (`app: hello-app`). Kubernetes cannot link the service to any pods — no traffic is routed.

**Fix:**

Either update the service selector:
```yaml
selector:
  app: hello-app
```

Or update the pod label:
```yaml
labels:
  app: my-app
```

**Verify fix:**
```bash
kubectl get endpoints my-service
# Should now show pod IPs instead of being empty
```

**Strong interview line:**
> "Services rely on label selectors for pod discovery — a label mismatch results in empty endpoints and no traffic routing."

---

## Round 3: Missing Readiness Probe

**Question:** What is missing from this deployment?

```yaml
containers:
  - name: app
    image: my-app
```

**What is missing?**

No readiness probe is defined. Without it, Kubernetes may send traffic to a pod before the application has finished starting up.

**Fix:**
```yaml
containers:
  - name: app
    image: my-app
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10
```

**Strong interview line:**
> "Readiness probes ensure traffic is only sent to pods that have successfully initialized. Without them, you get traffic routing failures during startup."

---

## Round 4: Pipeline Debug

**Question:** Pipeline builds successfully but the app is not updated in the cluster. What do you check?

**Step-by-step:**

```bash
# Step 1: Check what image tag is actually deployed
kubectl describe deployment <name>
# Look for "Image:" field — compare to pipeline build tag

# Step 2: Check rollout status
kubectl rollout status deployment/my-app

# Step 3: Check pod status
kubectl get pods
kubectl describe pod <pod>

# Step 4: Check deployment events
kubectl describe deployment <name>
```

**Possible causes:**
- Image tag not updated in deployment manifest
- Deployment YAML not re-applied by pipeline
- ArgoCD not synced to latest commit
- `imagePullPolicy: IfNotPresent` using cached image

**Strong interview line:**
> "CI success doesn't guarantee deployment success — I always verify what image tag is actually running in the cluster."

---

## Round 5: CrashLoopBackOff

**Question:** A pod is in `CrashLoopBackOff`. How do you debug?

```bash
# Step 1: Get pod events
kubectl describe pod <pod>
# Look at Events section at the bottom

# Step 2: Check container logs
kubectl logs <pod>

# Step 3: Check previous container logs (if pod restarted)
kubectl logs <pod> --previous
```

**Possible reasons:**
- Application crash (unhandled exception, missing dependency)
- Wrong environment variables
- Bad config (missing required config file or secret)
- Health probe misconfiguration causing restart loop

**Strong interview line:**
> "Pod logs and events are the fastest way to identify the root cause of a crash — I always check both."

---

## Round 6: Write a Deployment from Scratch

**Question:** Write a deployment with 2 replicas, nginx, port 80.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
```

---

## Answer Structure to Use Every Time

```
"First I check..."    → identify the layer
"Then I verify..."    → step-by-step checks with commands
"Possible reasons are..."  → root cause analysis
"I fix it by..."      → specific resolution
```
