# Terraform Infrastructure Story (GCP + AWS)

## Opening

I used Terraform extensively to provision and manage infrastructure across both GCP and AWS. My focus was on creating reusable, modular, and environment-specific infrastructure that could be consistently deployed across dev, staging, and production.

I worked on Terraform state management, detecting and resolving drift, and ensuring infrastructure changes were controlled and predictable.

---

## Code Structure

```
terraform/
│
├── modules/
│   ├── gke/
│   ├── vpc/
│   ├── eks/
│   ├── iam/
│   └── logging/
│
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── backend.tf
│   │
│   ├── staging/
│   └── prod/
│
└── shared/
    └── variables.tf
```

---

## Core Concepts

### 1. Modular Design

Terraform code structured using reusable modules. Each module represents a logical infrastructure component — VPC, GKE cluster, IAM roles, logging setup.

This allows reuse across multiple environments while passing different configurations (region, instance sizes, scaling parameters).

**Examples:**
- `modules/gke`
- `modules/vpc`
- `modules/iam`

**Benefit:** Reduced duplication, consistent infrastructure across environments.

---

### 2. Environment Separation

Each environment (dev, staging, prod) has its own Terraform configuration referencing shared modules. Changes can be applied independently per environment — no accidental impact on production.

---

### 3. Remote State Backend

Terraform state stored centrally:
- **AWS:** S3 bucket + DynamoDB for state locking
- **GCP:** GCS bucket with locking

**Why remote state:**
- Team collaboration without conflicts
- Consistent state across CI/CD and local runs
- Prevents local state drift

**State locking:** Prevents two people from running `apply` simultaneously, which would corrupt infrastructure state.

---

### 4. Standard Workflow

```bash
terraform init      # Initialize providers and backend
terraform plan      # Preview changes (always review before apply)
terraform apply     # Execute changes
```

Always review `terraform plan` output before applying, especially in production.

---

## Drift Handling

### What is drift?
Drift occurs when actual infrastructure differs from the Terraform state — typically due to manual changes or external modifications.

### How to detect:
```bash
terraform plan
# Shows differences between desired state and actual infrastructure
```

### How to resolve:
1. **Update Terraform code** to match real infra (when manual change was intentional)
2. **Reapply Terraform** to enforce desired state (when change was unauthorized)

**Strong interview line:**
> "I was involved in Terraform state reconciliation where we identified mismatches between infrastructure and configuration, and resolved them to ensure consistency across environments."

---

## State Management Deep Dive

### What is in the state file?
- Resource IDs (links Terraform config to real cloud resources)
- Metadata
- Dependency mapping

### Why state is critical:
Terraform relies on state to determine what changes need to be applied. Without accurate state, Terraform cannot safely manage infrastructure — it may create duplicates or attempt to destroy resources it doesn't recognize.

### If state is corrupted:
1. Check backup — remote backends support versioning
2. Restore previous version from backend
3. Use `terraform refresh` carefully to resync state with reality
4. Avoid manual state file editing unless absolutely necessary

---

## Interview Questions + Answers

**Q1: How do you structure Terraform code?**

Modular structure where reusable components (VPC, compute, IAM, Kubernetes clusters) are defined as modules. Environment-specific configurations reference these modules with different inputs — consistent infrastructure with environment-level flexibility.

---

**Q2: Why use modules?**

Modules promote reuse, reduce duplication, and enforce standardization. They make infrastructure easier to maintain and scale across multiple environments.

---

**Q3: What is Terraform state?**

A file that keeps track of infrastructure resources managed by Terraform. It maps configuration to real-world resources and helps Terraform determine what changes need to be applied.

---

**Q4: Why use a remote backend?**

Remote backends allow centralized state storage, enable team collaboration, provide state locking, and prevent conflicts when multiple engineers work on the same infrastructure.

---

**Q5: What is drift and how do you handle it?**

Drift happens when infrastructure changes outside Terraform. Detect with `terraform plan` and either update the code to reflect actual changes, or reapply Terraform to enforce the desired state.

---

**Q6: What if two people run apply at the same time?**

Without locking, state gets corrupted. That's why backends with locking (DynamoDB for AWS, GCS locking for GCP) ensure only one apply runs at a time.

---

**Q7: How do you manage secrets in Terraform?**

Never hardcode secrets in `.tf` files. Use:
- Environment variables
- Cloud secret managers (GCP Secret Manager, AWS Secrets Manager)
- CI/CD secrets injection at pipeline runtime
- Mark sensitive variables with `sensitive = true` in variable definitions

---

**Q8: `terraform plan` vs `apply`?**

`plan` shows what changes Terraform will make. `apply` executes those changes. Reviewing plan output is critical to avoid unintended modifications, especially resource destruction.

---

**Q9: How do you prevent accidental resource deletion?**

```hcl
resource "google_container_cluster" "main" {
  lifecycle {
    prevent_destroy = true
  }
}
```

Also: review plans carefully, enforce approval workflows in CI/CD for production applies.

---

**Q10: How do you scale Terraform for large infra?**

- Modularize infrastructure by component
- Split state per environment or service (separate state files)
- Use remote backends
- Organize code with clear environment directories
- Use Terragrunt for DRY configuration at scale

---

## Debugging Scenarios

**Scenario 1: Terraform wants to destroy resources unexpectedly**

Possible reasons:
- State mismatch (resource created outside Terraform)
- Resource config changed (forcing recreation)
- Module updated
- Manual drift

```bash
terraform plan               # Review what will be destroyed
terraform state list         # Compare state vs config
# DO NOT apply immediately — investigate first
```

---

**Scenario 2: Resource exists but Terraform wants to recreate it**

Likely causes:
- Missing state entry (resource not imported)
- Config mismatch
- Imported incorrectly

Solution:
```bash
terraform import <resource_type.name> <resource_id>
# Then align config with actual resource attributes
```

---

**Scenario 3: Apply fails halfway**

```bash
# 1. Check partial state updates
terraform state list

# 2. Fix the failed resource (manually if needed)
# 3. Re-run apply — Terraform is idempotent and will continue from where it left off
terraform apply
```

---

## Closing Statement

> Using Terraform allowed us to manage infrastructure in a consistent, version-controlled, and automated way. With proper module design, remote state management, and drift handling, we ensured reliability and scalability across environments while reducing manual errors.

---

## Interview Tip

| Don't say | Say instead |
|---|---|
| "I used Terraform" | "I designed modular Terraform architecture with remote state and handled drift reconciliation in production" |
| "I ran terraform apply" | "I reviewed plan output, validated changes, and applied with appropriate approval gates" |
