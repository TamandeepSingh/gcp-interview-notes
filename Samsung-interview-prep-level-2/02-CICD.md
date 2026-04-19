# CI/CD Pipeline Story: GitHub Actions → GKE/EKS

## High-Level Intro

I built and managed CI/CD pipelines using GitHub Actions for containerized applications. The goal was to automate the full delivery flow — from code commit to deployment on Kubernetes clusters (GKE and EKS) — while keeping the process reliable, secure, and easy to roll back.

The pipeline handled image build, registry push, deployment manifest updates, and cluster deployment. In some environments, ArgoCD managed the final Kubernetes deployment state in a GitOps model.

---

## Problem Statement

Before the pipeline was standardized, deployments were slower and more error-prone due to too many manual steps. The team needed a repeatable process to build images consistently, push them to the correct registry, and deploy safely to Kubernetes environments.

**My role:** Create a pipeline that reduced manual work, improved deployment consistency, and supported production-style controls — environment separation, rollback capability, and secret handling.

---

## Architecture

```
Developer
   │
   │ git push / pull request
   ▼
GitHub Repository
   │
   ▼
GitHub Actions Workflow
   │
   ├── Checkout source code
   ├── Run validation / lint / tests
   ├── Build Docker image
   ├── Tag image (commit SHA / version / env tag)
   ├── Authenticate to registry
   ├── Push image to Artifact Registry / ECR
   │
   ├── Option A (GitOps with ArgoCD):
   │     Update Helm values or Kubernetes manifests in deployment repo
   │     Commit new image tag
   │     ArgoCD detects Git change
   │     ArgoCD syncs to GKE / EKS
   │
   └── Option B (Direct deploy):
         Helm upgrade or kubectl apply
         Kubernetes cluster pulls latest image

Kubernetes Cluster (GKE / EKS)
   │
   ├── Deployment rollout
   ├── Readiness/liveness checks
   ├── Service / Ingress exposure
   └── Observability through logs / metrics
```

---

## Pipeline Stages

### Trigger
The pipeline triggered on push to `main`, `develop`, or release branches. Pull requests triggered validation-only workflows; merges to main triggered full deployment.

**Interview line:**
> "I separated CI and CD concerns. PRs ran checks like linting, tests, and build validation, while merges to protected branches triggered image publishing and deployment."

---

### Build Stage
After checkout, the workflow built the Docker image using the application Dockerfile. Image tags used the Git commit SHA — this gives traceability between deployed code and source history.

Tagging convention:
- `app:<commit-sha>` — source of truth
- `latest-dev` / `latest-prod` — convenience tags only

---

### Registry Push
The pipeline authenticated to the registry and pushed the image.
- GCP: Artifact Registry via service account / workload identity
- AWS: OIDC-based role assumption for GitHub Actions
- Avoid long-lived static credentials

---

### Deployment — GitOps with ArgoCD (Version A)

GitHub Actions did not directly change cluster state. Instead, it updated the deployment repo or Helm values file with the new image tag.

ArgoCD continuously watched that Git repository. When it detected a new commit, it synchronized the cluster state to match the desired configuration in Git.

**Why interviewers like this:**
- Declarative
- Auditable
- Safer than direct manual deploys
- Desired state visible in Git

---

### Deployment — Direct Deploy (Version B)

In simpler environments, GitHub Actions deployed directly using Helm or kubectl. Faster to implement but less GitOps visibility and separation of concerns.

---

## ArgoCD Role

ArgoCD continuously reconciled the cluster with the desired state stored in Git. If the pipeline updated the image tag in a Helm values file, ArgoCD detected that change and applied it to the cluster.

It also detected drift — if someone made a manual cluster change, ArgoCD flagged the application as out of sync.

**Strong line:**
> "I see ArgoCD as the deployment controller with Git as the source of truth, while GitHub Actions is the build-and-release automation layer."

---

## Helm Role

Helm packaged Kubernetes resources in a reusable, configurable way. Instead of hardcoding raw YAML per environment, chart templates and values files managed differences across dev, staging, and production — image tag, replica count, ingress settings, env vars.

---

## Secrets Management

- Pipeline secrets stored in GitHub Secrets or environment-level secrets
- Cloud access used short-lived identity-based auth (workload identity / OIDC) where possible
- Runtime secrets retrieved via Kubernetes Secrets or external secret manager
- No secrets baked into images or repos

**Strong line:**
> "My principle is to minimize secret exposure across the pipeline. The CI system should only access what it needs, and the cluster workload should retrieve runtime secrets securely."

---

## Rollback Strategy

| Deployment Type | Rollback Method |
|---|---|
| GitOps (ArgoCD) | Revert the Git commit; ArgoCD resyncs |
| Helm direct | `helm history` + `helm rollback` |

Immutable tags and versioned manifests make rollback safe and predictable.

---

## Full Polished Answer

> I built CI/CD pipelines in GitHub Actions for containerized applications deploying to GKE and EKS. The flow started when developers pushed code or merged pull requests into controlled branches.
>
> For pull requests, the workflow focused on validation — linting, tests, and build checks. For deployment branches, the pipeline built a Docker image, tagged it with the commit SHA, authenticated to the cloud registry, and pushed the image.
>
> In environments using ArgoCD, GitHub Actions updated the image tag in Helm values in a deployment repository. ArgoCD watched that repo and synchronized the new desired state into the Kubernetes cluster.
>
> This design gave us: consistent builds, repeatable deployments, auditability through Git, easier rollback, and reduced manual intervention. Helm standardized deployment templates; ArgoCD provided reconciliation and drift detection.
>
> If a deployment failed, I checked rollout status, pod logs, service and ingress mappings, and whether the correct image tag was deployed. For rollback, I preferred reverting the Git change in GitOps environments or using Helm rollback for direct deployments.

---

## Design Decision Q&A

| Question | Answer |
|---|---|
| Why GitHub Actions? | Close to source repo, simple workflow management, easy PR integration |
| Why immutable tags? | Traceability, safer rollback, avoids `latest` ambiguity |
| Why Helm? | Reusable templates, environment-specific values, easier upgrades |
| Why ArgoCD? | GitOps model, desired state in Git, drift detection, controlled deployments |
| Why not always deploy directly? | Less auditable, less control, harder CI/CD separation at scale |

---

## Interview Questions + Answers

**Q1: Walk me through your pipeline end to end.**

The pipeline starts on code push or merge. GitHub Actions checks out code, runs validations, builds the Docker image, tags it with commit SHA, authenticates to the registry, and pushes the image. Then it either deploys directly via Helm/kubectl or updates deployment manifests in Git. If ArgoCD is used, it detects the Git change and syncs the cluster automatically.

---

**Q2: Why use commit SHA tags instead of latest?**

Commit SHA tags are immutable and traceable. If a deployment fails, I know exactly which source version is running. Using only `latest` creates ambiguity because it can point to different content over time.

---

**Q3: What exactly does ArgoCD do in your architecture?**

ArgoCD watches the Git repository containing Kubernetes manifests or Helm values and continuously reconciles the cluster to match that desired state. It acts as the deployment controller and also detects drift when the actual cluster state diverges from Git.

---

**Q4: How do you manage secrets in the pipeline?**

Pipeline secrets stored in GitHub Secrets or environment-specific scopes. I prefer identity-based auth (workload identity / OIDC) to reduce reliance on long-lived static secrets. Runtime application secrets come from Kubernetes Secrets or an external secret manager.

---

**Q5: What happens if deployment fails after image push?**

1. Identify whether failure is in manifest sync, Kubernetes rollout, or application startup
2. Check ArgoCD sync status or Helm output
3. Check `kubectl rollout status`, pod events, and logs
4. Common issues: wrong image tag, failed readiness probe, missing secret/config, service misconfiguration
5. Fix forward or roll back to last stable version

---

**Q6: How do you roll back?**

In GitOps: revert the Git change that introduced the bad version; ArgoCD resyncs. With Helm directly: `helm history` + `helm rollback`. Immutable tags and versioned manifests make rollback safe.

---

**Q7: How do you handle different environments (dev, stage, prod)?**

Separate values files, separate overlays, or separate deployment repos. Keep shared templates reusable while isolating environment-specific config — image tags, ingress URLs, secrets references, replica counts.

---

**Q8: What are common failure points?**

- Registry authentication problems
- Image tag mismatch
- Bad Dockerfile layers
- Missing secrets
- Broken Helm values
- Readiness probe failures
- Wrong service selectors
- Ingress misconfiguration
- ArgoCD sync failures due to invalid manifests

---

**Q9: Pipeline succeeded but app is down. What do you check?**

1. Confirm correct image was pushed
2. Confirm deployment actually updated
3. Check rollout status
4. Inspect pod logs and events
5. Verify service endpoints
6. Verify ingress/load balancer routing
7. Check readiness/liveness probes and application startup config

---

**Q10: ArgoCD shows OutOfSync. What does that mean?**

Actual cluster state does not match desired state in Git. Could be a manual cluster change, failed sync, or difference between live and committed manifests. Inspect the diff and determine whether to reconcile or investigate as drift.

---

**Q11: What is the difference between CI and CD in your setup?**

CI covers code validation, testing, and image build. CD covers controlled release of that artifact into the target environment. In a GitOps setup, GitHub Actions handles CI and artifact publication; ArgoCD handles deployment reconciliation.

---

**Q12: What if the registry push succeeds but ArgoCD deploys an old image?**

Verify the manifest or values file was actually updated with the new tag, confirm ArgoCD synced that revision, and check whether `imagePullPolicy`, caching, or tag reuse caused confusion. This is another reason immutable tags are essential.

---

**Q13: How would you explain GitOps in one line?**

GitOps means Git is the source of truth for infrastructure or application configuration, and an automated controller continuously enforces that desired state in the target environment.

---

## Whiteboard Answer

> Developer pushes code to GitHub. GitHub Actions starts the CI pipeline, runs validations, builds the Docker image, and pushes it to the cloud registry. Then the workflow updates the deployment configuration — usually a Helm values file with the new image tag. ArgoCD monitors that deployment repo and syncs the desired state to the Kubernetes cluster. The cluster performs a rolling update, and health probes ensure only healthy pods receive traffic.

---

## Red Flags to Avoid

| Don't say | Say instead |
|---|---|
| "We just used latest tag" | Immutable tags with commit SHA |
| "I'm not sure what ArgoCD does" | Deployment controller, desired state in Git |
| "Secrets were mostly hardcoded in YAML" | Least privilege, external secret manager |
| "If deployment failed, we reran pipeline" | Rollback via Git revert or Helm history |
| "GitHub Actions and ArgoCD both deployed" | Clear CI/CD separation of responsibilities |
