# Terraform Scenarios - Samsung DevOps Interview

## Scenario 1: Terraform State Corruption in Production

### The Problem
Your CI/CD pipeline ran `terraform apply` but crashed mid-execution. The `.tfstate` file in your remote backend is incomplete, and terraform commands now hang indefinitely.

You have:
- 200+ resources currently in AWS
- Team of 5 engineers who might be running terraform
- Production database, load balancers, and compute instances

**How do you recover without destroying infrastructure?**

### Expected Answer (DETAILED)

#### Step 1: Assess the Damage
```bash
# DO NOT run terraform apply or destroy yet
# First, understand what's corrupted:

# Check if terraform can even read state:
$ terraform validate
$ terraform state list 2>&1 | head -20

# Try showing state (but don't lock it):
$ terraform show 2>&1

# Check the remote state file directly:
$ aws s3 cp s3://my-terraform-bucket/prod.tfstate ./prod.tfstate.bak
$ head -100 prod.tfstate.bak  # Check for valid JSON

# Check for locks:
$ terraform force-unlock <LOCK_ID>  # If dynamodb lock exists

# View lock status:
$ aws dynamodb scan --table-name terraform-locks --region us-east-1
```

#### Step 2: Validate State File
```bash
# Check if state is valid JSON:
$ jq . prod.tfstate > /dev/null && echo "Valid JSON" || echo "Invalid JSON"

# Check terraform version in state:
$ cat prod.tfstate | jq '.terraform_version'

# Compare with current terraform version:
$ terraform version

# Check state version:
$ cat prod.tfstate | jq '.version'
```

#### Step 3: Recovery Options

##### Option A: Restore from Backup (Best Case)
```bash
# If you have versioning enabled on S3:
$ aws s3api list-object-versions --bucket my-terraform-bucket --prefix prod.tfstate

# Restore previous version:
$ aws s3api get-object \
  --bucket my-terraform-bucket \
  --key prod.tfstate \
  --version-id <PREVIOUS_VERSION_ID> \
  prod.tfstate.restored

# Verify it's valid:
$ jq . prod.tfstate.restored | head -50

# Point terraform to restored state:
$ mv prod.tfstate prod.tfstate.corrupted
$ mv prod.tfstate.restored prod.tfstate

# Push to remote:
$ aws s3 cp prod.tfstate s3://my-terraform-bucket/prod.tfstate
```

##### Option B: Manual State Reconstruction (Complex Recovery)
```bash
# If backup is lost, reconstruct from actual AWS state
# This is risky but doable

# 1. Create new completely fresh terraform.tfstate:
$ terraform state list > actual_resources.txt

# 2. Manually import each resource (if terraform is completely broken):
$ terraform import aws_instance.web i-1234567890abcdef0
$ terraform import aws_security_group.main sg-12345678
# This is tedious but might be necessary

# 3. Or, use terraform provider to refresh:
$ terraform refresh  # Re-read actual state from AWS
```

##### Option C: Break Everything and Rebuild (Nuclear Option)
```bash
# Only if:
# - State is completely unrecoverable
# - You have IaC for everything
# - You can afford 1-2 hour downtime

# 1. Document current infrastructure:
$ aws ec2 describe-instances --output json > instances.json
$ aws rds describe-db-instances --output json > databases.json

# 2. Backup DNS/routes:
$ aws route53 list-resource-record-sets --hosted-zone-id Z123 > route53_backup.json

# 3. Start fresh:
$ rm -rf terraform.tfstate*
$ terraform init -reconfigure
$ terraform apply  # Builds everything from scratch

# 4. Point traffic back manually if needed
```

#### Step 4: Prevent This Going Forward
```bash
# 1. Enable S3 versioning:
$ aws s3api put-bucket-versioning \
  --bucket my-terraform-bucket \
  --versioning-configuration Status=Enabled

# 2. Enable MFA delete (optional, requires root):
$ aws s3api put-bucket-versioning \
  --bucket my-terraform-bucket \
  --versioning-configuration Status=Enabled,MFADelete=Enabled

# 3. Enable DynamoDB locking:
resource "aws_dynamodb_table" "terraform_locks" {
  name           = "terraform-locks"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    purpose = "terraform-state-locking"
  }
}

# In terraform backend config:
terraform {
  backend "s3" {
    bucket         = "my-terraform-bucket"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# 4. Regular state backups:
$ cat << 'EOF' > backup_state.sh
#!/bin/bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
aws s3 cp s3://my-terraform-bucket/prod.tfstate \
  s3://my-terraform-backup-bucket/prod_$TIMESTAMP.tfstate
EOF

# Run on schedule (cron/CloudScheduler):
*/6 * * * * /path/to/backup_state.sh
```

#### Step 5: CI/CD Pipeline Safeguards
```yaml
# GitHub Actions example with safeguards:
name: Terraform Apply with Safety

on: [push]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      # 1. Backup state before apply
      - name: Backup state
        run: |
          aws s3 cp s3://my-bucket/prod.tfstate \
            s3://my-backup-bucket/pre-apply-$(date +%s).tfstate
      
      # 2. Plan first
      - name: Terraform plan
        run: |
          terraform plan -out=tfplan
          terraform show tfplan > plan_output.txt
      
      # 3. Fail if plan is too big (catch runaway changes)
      - name: Validate plan
        run: |
          RESOURCE_COUNT=$(grep -c "will be" plan_output.txt)
          if [ $RESOURCE_COUNT -gt 50 ]; then
            echo "ERROR: Plan modifies >50 resources. Requires manual approval."
            exit 1
          fi
      
      # 4. Apply with timeout
      - name: Terraform apply
        timeout-minutes: 30
        run: terraform apply tfplan
      
      # 5. If apply fails, restore state
      - name: Restore state on failure
        if: failure()
        run: |
          aws s3 cp s3://my-backup-bucket/pre-apply-latest.tfstate \
            s3://my-bucket/prod.tfstate
```

### Follow-up Questions

**Q1: How do you prevent race conditions when multiple engineers are running terraform?**
- Expected: Discuss locking, state locking, CI/CD as single source of truth

**Q2: What's the difference between remote state and local state?**
- Expected: Explain sharing, versioning, locking capabilities

**Q3: How do you handle terraform drift (manual AWS changes)?**
- Expected: `terraform refresh` and `import`, monitoring for drift

### Red Flags

❌ "I'll just re-apply and hope it works"
- Could destroy resources or create duplicates

❌ "State is complicated, we don't need to back it up"
- State is the most critical artifact in infrastructure

❌ "Lock files are optional"
- Locks prevent simultaneous modifications that corrupt state

### Pro Tips

✅ "I'd check state version and terraform version match before any operations"

✅ "I'd implement a backup strategy: daily snapshots to S3 with versioning"

✅ "I'd use terraform plan review as a mandatory step before apply"

✅ "I'd monitor state file size; sudden changes can indicate corruption"

---

## Scenario 2: Terraform Module Dependency Hell

### The Problem
Your infrastructure has 15 terraform modules:
- `vpc` (no dependencies)
- `eks` (depends on `vpc`, `networking`)
- `networking` (depends on `vpc`)
- `rds` (depends on `vpc`)
- `helm-releases` (depends on `eks`)
- etc.

When you run `terraform apply`, sometimes it works, sometimes:
- `helm-releases` fails because `eks` isn't fully ready
- `rds` is created before security groups
- You're running `terraform apply` 3-4 times in a row until all modules succeed

**How do you fix the dependency resolution?**

### Expected Answer (DETAILED)

#### Step 1: Map Dependencies Explicitly
```bash
# First, visualize current state:
$ terraform graph | grep -E "module\.|depends_on" > dependency_map.txt

# Use terragrunt to visualize better:
$ terragrunt graph > graph.json

# Or manually document:
# vpc
#   ├── networking
#   │   └── eks
#   │       └── helm-releases
#   ├── rds
#   └── elasticache
```

#### Step 2: Identify Hidden Dependencies
```bash
# These are often implicit and cause issues:

# In eks module, check for actual references to vpc:
$ grep -r "aws_vpc" terraform/modules/eks/

# Check for data source dependencies:
$ grep -r "data.aws_" terraform/modules/*/

# Example problem:
# In helm-releases/main.tf:
# - Waiting for EKS node group to exist
# - Waiting for karpenter controller to be ready
# - Waiting for specific CRDs to be available
```

#### Step 3: Add Explicit Dependencies
```hcl
# BAD: Implicit dependencies through data sources
module "helm_releases" {
  source = "./modules/helm-releases"
  cluster_name = module.eks.cluster_name
  
  # Problem: No guarantee EKS cluster is READY, just created
}

# BETTER: Explicit depends_on
module "helm_releases" {
  source = "./modules/helm-releases"
  cluster_name = module.eks.cluster_name
  
  depends_on = [
    module.eks.cluster_ready,  # Custom output indicating readiness
    aws_eks_node_group.main,   # Direct dependency on node group
  ]
}

# BEST: Wait for actual readiness conditions
resource "null_resource" "eks_ready" {
  triggers = {
    cluster_id = module.eks.cluster_id
  }

  provisioner "local-exec" {
    command = <<-EOT
      set -e
      
      # Wait for kubeapi to respond
      for i in {1..60}; do
        kubectl cluster-info && break || sleep 10
      done
      
      # Wait for nodes to be ready
      kubectl wait --for=condition=Ready node --all --timeout=300s
      
      # Wait for system pods to be running
      kubectl wait --for=condition=ready pod \
        -l k8s-app=aws-node -n kube-system \
        --timeout=300s
    EOT
  }
}

module "helm_releases" {
  depends_on = [null_resource.eks_ready]
}
```

#### Step 4: Reduce Dependencies with Data Source Approach
```hcl
# Instead of depending on module outputs (which are created sequentially):
# Use data sources to reference existing resources

# BAD: Forces sequential creation
module "networking" {
  source = "./modules/networking"
  vpc_id = module.vpc.id
}

module "rds" {
  source = "./modules/rds"
  security_group_id = module.networking.rds_sg_id
}

# BETTER: Use data sources for parallel creation where possible
data "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "main-vpc"
  }
}

# RDS can reference data source directly
module "rds" {
  source = "./modules/rds"
  vpc_id = data.aws_vpc.main.id
}
```

#### Step 5: Use Terragrunt for Dependency Management
```hcl
# Terragrunt: terragrunt.hcl in each module directory

# vpc/terragrunt.hcl
terraform {
  source = "./"
}

# networking/terragrunt.hcl
terraform {
  source = "./"
}
dependency "vpc" {
  config_path = "../vpc"
  mock_outputs = {
    vpc_id = "vpc-12345"
  }
}

inputs = {
  vpc_id = dependency.vpc.outputs.vpc_id
}

# eks/terragrunt.hcl
terraform {
  source = "./"
}
dependency "vpc" {
  config_path = "../vpc"
}
dependency "networking" {
  config_path = "../networking"
}

inputs = {
  vpc_id = dependency.vpc.outputs.vpc_id
  subnets = dependency.networking.outputs.private_subnets
}

# helm/terragrunt.hcl
terraform {
  source = "./"
}
dependency "eks" {
  config_path = "../eks"
  fetch_output_from_state = true
  
  # Wait for output to be available
  skip_outputs = false
}

inputs = {
  cluster_name = dependency.eks.outputs.cluster_name
}

# Apply with dependency order:
$ terragrunt run-all apply --terragrunt-non-interactive
```

#### Step 6: Async Operations and Waits
```hcl
# Handle resources that take time to become ready

# EKS cluster creation is fast, but node group takes 10+ minutes
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "main"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = aws_subnet.private[*].id

  tags = { Name = "main" }
}

# Add a wait condition
resource "time_sleep" "wait_eks_ready" {
  depends_on      = [aws_eks_node_group.main]
  create_duration = "120s"  # Wait 2 minutes for nodes to be ready
}

# Reference this in helm module
module "helm_releases" {
  depends_on = [time_sleep.wait_eks_ready]
}
```

#### Step 7: Testing Dependencies
```bash
# Destroy and recreate to test dependency ordering:
$ terraform destroy -auto-approve

# Test with fresh apply:
$ time terraform apply -auto-approve

# Should complete in single pass without errors
# Time should be minimal (only sequential waits, not retries)
```

### Follow-up Questions

**Q1: What if two modules have circular dependencies?**
- Expected: Refactor one to use data sources, split module responsibilities

**Q2: How do you test dependency changes safely?**
- Expected: Terraform plan, staging environment, destroy/rebuild testing

**Q3: What if a resource takes 30 minutes to become ready?**
- Expected: Use `time_sleep`, custom conditions, health checks

### Red Flags

❌ "We just run terraform apply until it works"
- Manual, error-prone, shows no architectural understanding

❌ "Dependencies are terraform's job, not ours"
- You need to understand and manage them explicitly

❌ "We use implicit dependencies through module outputs"
- Creates tight coupling and sequential creation

### Pro Tips

✅ "I'd use Terragrunt for explicit dependency management and cleaner separation"

✅ "I'd implement health checks with `null_resource` and local-exec"

✅ "I'd parallelize where possible using data sources for existing resources"

✅ "I'd add explicit depends_on even when not strictly necessary for readability"

---

## Scenario 3: Terraform Apply Hangs - What's Wrong?

### The Problem
`terraform apply` has been running for 45 minutes without any output. It's not consuming CPU/memory, not making API calls (checked CloudTrail). Just... stuck.

Ctrl+C shows:
```
resource "aws_rds_cluster_instance" "main": Still creating...
```

**How do you diagnosed and fix this without losing progress?**

### Expected Answer (DETAILED)

#### Step 1: Understand Terraform Process
```bash
# Check terraform background processes:
$ ps aux | grep terraform
$ lsof -p <terraform_pid> | grep -E "network|socket"

# Check network connections:
$ netstat -an | grep ESTABLISHED | grep -E "443|amazonaws"

# Check if API calls are happening:
$ aws cloudtrail lookup-events --max-results 10 \
  --query 'Events[] | [].{EventName, EventTime, Resources, ErrorCode}'
```

#### Step 2: Check AWS API Rate Limits
```bash
# RDS creation is often slow:
# - Cluster creation: 5-10 minutes
# - Instance creation: 10-15 minutes
# - Total with multi-AZ: 20-30 minutes

# Check if it's just slow, not stuck:
$ aws rds describe-db-clusters --query 'DBClusters[].{
  DBClusterIdentifier,
  Status,
  PercentProgress
}'

# If stuck at 0%:
$ aws rds describe-db-cluster-instances --query 'DBClusterMembers[].{
  DBInstanceIdentifier,
  IsClusterWriter
}'
```

#### Step 3: Check terraform.log
```bash
# Enable debug logging:
$ TF_LOG=DEBUG terraform apply -auto-approve 2>&1 | tee terraform-debug.log

# Search for hanging points:
$ grep -i "error\|timeout\|pending" terraform-debug.log | tail -20

# Look for last successful API call:
$ grep "aws_" terraform-debug.log | tail -20
```

#### Step 4: Common Causes and Fixes

##### Cause 1: RDS Creation Legitimately Slow
```bash
# This is normal - RDS Multi-AZ can take 30+ minutes
# Monitor progress:
$ watch -n 30 'aws rds describe-db-clusters \
  --query "DBClusters[].{ID:DBClusterIdentifier,Status:Status}"'

# If status changes, it's working
# Can safely Ctrl+C and check if terraform recovers
```

##### Cause 2: Security Group Rule Missing (Most Common)
```bash
# RDS creation hangs when VPC connectivity issues exist
# Check if RDS is trying to access something:
$ aws rds describe-db-clusters --query 'DBClusters[] | [0].{
  DBClusterIdentifier,
  Status,
  VpcSecurityGroups
}'

# Check if security group has required rules:
$ aws ec2 describe-security-groups --group-ids sg-12345 \
  --query 'SecurityGroups[0].IpPermissions'

# Fix: Add egress or modify security group rules
resource "aws_security_group_rule" "rds_egress" {
  type              = "egress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.rds.id
}
```

##### Cause 3: Terraform Provider/API Bug
```bash
# Check if AWS API is responding:
$ aws ec2 describe-instances --max-results 1 \
  --region us-east-1

# If this hangs too, it's AWS API issue, not terraform

# Force interrupt and remove lock:
$ terraform force-unlock <LOCK_ID>

# Check terraform cloud logs:
$ terraform show -json | jq -r '.values.root_module.resources[] | .address' | head -5
```

##### Cause 4: Resource Quota Exceeded
```bash
# Check if you've hit AWS resource limits:
$ aws service-quotas get-service-quota \
  --service-code rds \
  --quota-code L-7B6409FD  # RDS instances

# If at limit:
# - Request quota increase
# - Or destroy unused resources
```

#### Step 5: Emergency Recovery
```bash
# If truly stuck and urgent:
# Option 1: Manual interrupt (terraform will retry on next apply)
$ Ctrl+C

# Wait 5 seconds
$ terraform apply -auto-approve

# Option 2: Check if resource was actually created
$ aws rds describe-db-clusters \
  --query 'DBClusters[].DBClusterIdentifier'

# If yes, remove from state and re-import:
$ terraform state rm aws_rds_cluster.main
$ terraform import aws_rds_cluster.main <cluster-arn>

# Option 3: Increase timeout for long-running resources
resource "aws_rds_cluster_instance" "main" {
  # ... other config ...

  timeouts {
    create = "2h"
    delete = "1h"
  }
}
```

### Follow-up Questions

**Q1: How do you set timeout expectations for different resources?**
- Expected: Discuss different resource types, add documentation

**Q2: What monitoring would catch this before it hangs?**
- Expected: CloudWatch, API metrics, terraform state monitoring

### Red Flags

❌ "I'll just kill the process and rerun"
- Risk of orphaned resources or duplicate creation

❌ "The provider is broken"
- Investigate thoroughly before concluding that

### Pro Tips

✅ "I'd add explicit timeouts to slow resources like RDS"

✅ "I'd monitor terraform apply runs with CloudWatch"

✅ "I'd log all terraform runs with TF_LOG=DEBUG to Cloudwatch"

