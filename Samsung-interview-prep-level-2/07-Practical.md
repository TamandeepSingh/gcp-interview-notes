# Practical Lab: GitHub Actions → GKE Deployment

## What You Will Build

A Hello World Node.js app deployed via GitHub Actions to GKE.

**Pipeline flow:**
1. Build Docker image
2. Push to Artifact Registry
3. Deploy to GKE via `kubectl apply`

---

## Architecture

```
Developer push to GitHub
        │
        ▼
GitHub Actions
  ├─ checkout code
  ├─ authenticate to Google Cloud (Workload Identity Federation)
  ├─ build Docker image
  ├─ push to Artifact Registry
  ├─ get GKE credentials
  └─ kubectl apply deployment + service
        │
        ▼
GKE Cluster
  ├─ Deployment
  ├─ Pods
  └─ Service type LoadBalancer
        │
        ▼
External IP
```

---

## Repo Structure

```
hello-gke-github-actions/
├── app/
│   ├── package.json
│   ├── server.js
│   └── Dockerfile
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
└── .github/
    └── workflows/
        └── deploy.yml
```

---

## 1. Application Code

### `app/package.json`
```json
{
  "name": "hello-gke",
  "version": "1.0.0",
  "description": "Simple hello world app for GKE practice",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.19.2"
  }
}
```

### `app/server.js`
```javascript
const express = require("express");
const app = express();

const port = process.env.PORT || 8080;

app.get("/", (req, res) => {
  res.send("Hello from GKE via GitHub Actions!");
});

app.get("/health", (req, res) => {
  res.status(200).send("ok");
});

app.listen(port, () => {
  console.log(`App listening on port ${port}`);
});
```

### `app/Dockerfile`
```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package.json ./
RUN npm install --omit=dev

COPY server.js ./

EXPOSE 8080

CMD ["npm", "start"]
```

---

## 2. Kubernetes Manifests

### `k8s/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-gke
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-gke
  template:
    metadata:
      labels:
        app: hello-gke
    spec:
      containers:
        - name: hello-gke
          image: IMAGE_PLACEHOLDER
          ports:
            - containerPort: 8080
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

### `k8s/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-gke-service
spec:
  type: LoadBalancer
  selector:
    app: hello-gke
  ports:
    - port: 80
      targetPort: 8080
```

---

## 3. GCP Setup (Run Once)

```bash
export PROJECT_ID="your-gcp-project-id"
export REGION="us-central1"
export ZONE="us-central1-a"
export CLUSTER_NAME="hello-gke-cluster"
export REPO_NAME="hello-gke-repo"

# Enable required APIs
gcloud services enable \
  container.googleapis.com \
  artifactregistry.googleapis.com \
  cloudresourcemanager.googleapis.com \
  iamcredentials.googleapis.com \
  sts.googleapis.com

# Create Artifact Registry repo
gcloud artifacts repositories create $REPO_NAME \
  --repository-format=docker \
  --location=$REGION \
  --description="Docker repo for hello-gke practice"

# Create GKE cluster
gcloud container clusters create $CLUSTER_NAME \
  --zone $ZONE \
  --num-nodes 2

# Connect kubectl
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE
```

---

## 4. GitHub Actions Auth Setup (Workload Identity Federation)

WIF allows GitHub Actions to authenticate to GCP without storing long-lived JSON keys. GitHub issues an OIDC token; GCP trusts that identity provider and allows impersonation of a service account.

```bash
# Create service account
gcloud iam service-accounts create github-actions-gke \
  --display-name="GitHub Actions GKE deployer"

# Grant roles
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-actions-gke@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/container.developer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-actions-gke@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-actions-gke@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/container.clusterViewer"
```

### GitHub Repository Secrets/Variables to add:
- `GCP_PROJECT_ID`
- `GCP_WIF_PROVIDER`
- `GCP_SERVICE_ACCOUNT`
- `GKE_CLUSTER`
- `GKE_ZONE`
- `ARTIFACT_REGISTRY_REGION`
- `ARTIFACT_REGISTRY_REPO`

---

## 5. GitHub Actions Workflow

### `.github/workflows/deploy.yml`
```yaml
name: Build and Deploy to GKE

on:
  push:
    branches: ["main"]

permissions:
  contents: read
  id-token: write

env:
  PROJECT_ID: ${{ vars.GCP_PROJECT_ID }}
  GAR_REGION: ${{ vars.ARTIFACT_REGISTRY_REGION }}
  GAR_REPO: ${{ vars.ARTIFACT_REGISTRY_REPO }}
  CLUSTER_NAME: ${{ vars.GKE_CLUSTER }}
  CLUSTER_ZONE: ${{ vars.GKE_ZONE }}
  IMAGE_NAME: hello-gke

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WIF_PROVIDER }}
          service_account: ${{ vars.GCP_SERVICE_ACCOUNT }}

      - name: Set up gcloud
        uses: google-github-actions/setup-gcloud@v2

      - name: Configure Docker auth for Artifact Registry
        run: gcloud auth configure-docker $GAR_REGION-docker.pkg.dev --quiet

      - name: Build image
        run: |
          docker build -t $GAR_REGION-docker.pkg.dev/$PROJECT_ID/$GAR_REPO/$IMAGE_NAME:${{ github.sha }} ./app

      - name: Push image
        run: |
          docker push $GAR_REGION-docker.pkg.dev/$PROJECT_ID/$GAR_REPO/$IMAGE_NAME:${{ github.sha }}

      - name: Get GKE credentials
        run: |
          gcloud container clusters get-credentials $CLUSTER_NAME --zone $CLUSTER_ZONE --project $PROJECT_ID

      - name: Render deployment image
        run: |
          sed "s|IMAGE_PLACEHOLDER|$GAR_REGION-docker.pkg.dev/$PROJECT_ID/$GAR_REPO/$IMAGE_NAME:${{ github.sha }}|g" \
          k8s/deployment.yaml > k8s/deployment-rendered.yaml

      - name: Deploy to GKE
        run: |
          kubectl apply -f k8s/deployment-rendered.yaml
          kubectl apply -f k8s/service.yaml
          kubectl rollout status deployment/hello-gke --timeout=120s

      - name: Show service
        run: kubectl get svc hello-gke-service
```

---

## 6. Verify Deployment

```bash
kubectl get pods
kubectl get svc hello-gke-service
# Wait for EXTERNAL-IP, then open in browser
```

---

## 7. Break-and-Fix Drills

### Break 1: Wrong targetPort
Change `service.yaml`:
```yaml
targetPort: 3000  # Container actually runs on 8080
```

**Symptom:** Pods healthy, app unreachable.

```bash
kubectl describe svc hello-gke-service
kubectl get endpoints hello-gke-service
kubectl logs deploy/hello-gke
```

**Interview answer:**
> "I validate the traffic path layer by layer: pod health, container port, service targetPort, endpoints, and external exposure."

---

### Break 2: Wrong selector labels
Change `service.yaml`:
```yaml
selector:
  app: wrong-label
```

**Symptom:** Service has no endpoints.

```bash
kubectl get endpoints hello-gke-service  # Will be empty
kubectl get pods --show-labels
kubectl describe svc hello-gke-service
```

**Interview answer:**
> "If endpoints are empty, I immediately suspect a label mismatch between the service selector and the pod template labels."

---

### Break 3: Bad image
Change the image to a non-existent tag.

**Symptom:** Pods go `ImagePullBackOff`.

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl rollout status deployment/hello-gke
```

**Interview answer:**
> "`describe pod` shows image pull failures, registry auth issues, or bad tags very clearly in the Events section."

---

### Break 4: Failing health endpoint
Change `server.js` so `/health` returns 500.

**Symptom:** Pods restart repeatedly (liveness probe failure) or never become Ready (readiness probe failure).

```bash
kubectl get pods    # Shows RESTARTS count increasing
kubectl describe pod <pod-name>   # Shows probe failures in Events
kubectl logs <pod-name>
```

**Interview answer:**
> "A running pod is not enough. Readiness controls traffic routing and liveness controls restarts — I always verify probes independently."

---

## Interview Talking Points

| Question | Answer |
|---|---|
| Why GitHub Actions? | Integrates closely with source repo, automates build/test/deploy on pushes and PRs |
| Why Artifact Registry? | GCP's native container store, integrates cleanly with GKE image pulls |
| Why Workload Identity Federation? | Avoids storing long-lived service account keys; uses short-lived identity-based auth |
| Why LoadBalancer service? | Simple external entry point backed by Google Cloud load balancing |
| Why `github.sha` for image tags? | Traceability from deployed image back to the exact commit that produced it |

---

## Practice Plan

**Day 1:**
1. Create cluster
2. Create Artifact Registry repo
3. Set up GitHub Actions auth
4. Push app and verify external IP
5. Note every command used

**Day 2:**
Run the 4 failure drills. For each, write:
- Symptom observed
- First command you ran
- Root cause identified
- Fix applied

This turns "I know Kubernetes" into "I debug Kubernetes systematically."
