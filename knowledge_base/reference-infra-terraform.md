---
title: "Infrastructure-as-Code Reference (Terraform)"
author: "SRE Team"
tags: ["terraform", "infrastructure", "iac", "aws", "eks", "rds", "reference", "platform", "sre"]
last_updated: "2024-02-14"
version: "2.1.0"
---

# Infrastructure-as-Code Reference (Terraform)

> **Audience:** SRE engineers and backend engineers who need to provision or modify cloud infrastructure.
> **Purpose:** Document AetherFlow's Terraform conventions, module structure, workflow, and the most commonly used patterns.
> **Related Docs:** `runbook-production-deployment.md`, `runbook-database-failover.md`, `policy-secret-management.md`, `reference-service-catalog.md`

All AetherFlow cloud infrastructure is managed as code. If it's not in Terraform, it doesn't exist — and if it exists and isn't in Terraform, it will be destroyed or drift-corrected without warning during the next `terraform apply`. There are no exceptions.

---

## Table of Contents

1. [Repository Structure](#1-repository-structure)
2. [Environment Layout](#2-environment-layout)
3. [Terraform Workflow](#3-terraform-workflow)
4. [Module Catalog](#4-module-catalog)
5. [Key Resource Configurations](#5-key-resource-configurations)
6. [State Management](#6-state-management)
7. [Secret Injection Pattern](#7-secret-injection-pattern)
8. [Drift Detection](#8-drift-detection)
9. [Common Patterns & Recipes](#9-common-patterns--recipes)
10. [Tribal Knowledge & Gotchas](#10-tribal-knowledge--gotchas)

---

## 1. Repository Structure

All Terraform lives in the `aetherflow/infra` repository:

```
infra/
├── modules/                    # Reusable internal modules
│   ├── eks-cluster/            # EKS cluster + node groups
│   ├── rds-postgres/           # RDS PostgreSQL instance + parameter group
│   ├── kafka-strimzi/          # Strimzi Kafka Operator installation
│   ├── vpc/                    # VPC, subnets, route tables, NAT gateways
│   ├── iam-service-account/    # IRSA (IAM Roles for Service Accounts)
│   └── acm-certificate/        # ACM certificate + Route53 DNS validation
│
├── environments/
│   ├── production/
│   │   ├── main.tf             # Root module wiring all child modules
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── backend.tf          # S3 + DynamoDB remote state config
│   │   └── terraform.tfvars    # Production variable values (NOT secrets)
│   ├── staging/
│   │   └── ...
│   └── development/
│       └── ...
│
├── .terraform-version          # Pinned Terraform version (via tfenv)
└── Makefile                    # Standard targets: fmt, validate, plan, apply
```

### Tooling Requirements

| Tool | Version | Install |
|------|---------|---------|
| `terraform` | 1.7.x (pinned via `.terraform-version`) | `brew install tfenv && tfenv install` |
| `tfenv` | Latest | `brew install tfenv` |
| `tflint` | ≥ 0.50 | `brew install tflint` |
| `terraform-docs` | ≥ 0.17 | `brew install terraform-docs` |
| `infracost` | Latest | `brew install infracost` |

---

## 2. Environment Layout

AetherFlow runs three environments on AWS. Each has its own Terraform state, AWS account, and VPC.

| Environment | AWS Account ID | Primary Region | Purpose |
|-------------|---------------|----------------|---------|
| `production` | `123456789012` | `us-east-1` | Customer-facing production |
| `staging` | `234567890123` | `us-east-1` | Pre-production testing; mirrors prod at 50% scale |
| `development` | `345678901234` | `us-east-1` | Shared dev environment; no SLA |

> Production and staging are in **separate AWS accounts** — this is a hard isolation boundary. IAM roles, VPCs, and security groups cannot cross account boundaries by default. Engineers do not have console access to the production account; all production changes go through Terraform and GitHub Actions.

---

## 3. Terraform Workflow

### Day-to-Day Changes

```bash
cd infra/environments/production

# 1. Format your code
terraform fmt -recursive

# 2. Validate syntax and provider schema
terraform validate

# 3. Run tflint for module-level checks
tflint --recursive

# 4. Generate a plan and review it carefully
terraform plan -out=tfplan

# 5. Estimate cost impact (required for changes that add resources)
infracost breakdown --path . --format table

# 6. Review the plan output — every resource change must be intentional
# Look for: unexpected destroys, replacement triggers, unexpected count changes

# 7. Apply (requires SRE approval for production; see Section 7 of this doc)
terraform apply tfplan
```

### CI/CD Integration

Terraform changes go through a mandatory CI pipeline in GitHub Actions:

```yaml
# .github/workflows/terraform.yml (excerpt)
on:
  pull_request:
    paths:
      - 'infra/**'

jobs:
  terraform-plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.3"

      - name: Terraform Format Check
        run: terraform fmt -check -recursive
        working-directory: infra/

      - name: Terraform Validate
        run: terraform validate
        working-directory: infra/environments/production/

      - name: Terraform Plan
        run: terraform plan -no-color -out=tfplan 2>&1 | tee plan-output.txt
        working-directory: infra/environments/production/
        env:
          AWS_ROLE_ARN: ${{ secrets.PROD_TERRAFORM_ROLE_ARN }}

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const plan = require('fs').readFileSync('infra/environments/production/plan-output.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Terraform Plan — Production\n\`\`\`\n${plan.slice(0, 65000)}\n\`\`\``
            });
```

`terraform apply` in production is **never automated** — it always requires a manual step by an SRE engineer after the PR is reviewed and merged.

---

## 4. Module Catalog

### `modules/eks-cluster`

Provisions an EKS cluster with managed node groups. Used by all three environments.

**Inputs:**

| Variable | Type | Description |
|----------|------|-------------|
| `cluster_name` | string | EKS cluster name |
| `cluster_version` | string | Kubernetes version (e.g., `"1.29"`) |
| `vpc_id` | string | VPC to deploy into |
| `subnet_ids` | list(string) | Private subnet IDs for nodes |
| `node_groups` | map(object) | Node group definitions (instance type, min/max/desired counts) |
| `enable_irsa` | bool | Enable IAM Roles for Service Accounts (always `true` in prod) |

**Usage (production):**

```hcl
# environments/production/main.tf
module "eks" {
  source = "../../modules/eks-cluster"

  cluster_name    = "aetherflow-prod"
  cluster_version = "1.29"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids
  enable_irsa     = true

  node_groups = {
    general = {
      instance_types = ["m6i.2xlarge"]
      min_size       = 6
      max_size       = 20
      desired_size   = 10
      disk_size_gb   = 100
      labels = {
        "node-group" = "general"
      }
    }
    kafka = {
      instance_types = ["m6i.2xlarge"]
      min_size       = 3
      max_size       = 3   # Kafka nodes are not autoscaled
      desired_size   = 3
      disk_size_gb   = 500  # Kafka needs large local disk
      labels = {
        "node-group"        = "kafka"
        "kafka-broker"      = "true"
      }
      taints = [{
        key    = "kafka-dedicated"
        value  = "true"
        effect = "NO_SCHEDULE"
      }]
    }
  }
}
```

---

### `modules/rds-postgres`

Provisions an RDS PostgreSQL instance with a Multi-AZ standby, automated backups, and a dedicated parameter group.

**Key Configuration (production):**

```hcl
module "postgres_primary" {
  source = "../../modules/rds-postgres"

  identifier        = "aetherflow-prod-primary"
  engine_version    = "15.5"
  instance_class    = "db.r6g.2xlarge"
  allocated_storage = 1000   # GB
  max_allocated_storage = 4000  # Autoscaling ceiling

  # High Availability
  multi_az               = true
  deletion_protection    = true   # Must be disabled manually before destroy
  backup_retention_days  = 30
  backup_window          = "03:00-04:00"  # UTC
  maintenance_window     = "sun:04:00-sun:05:00"  # UTC

  # Performance
  performance_insights_enabled = true
  monitoring_interval          = 60  # Enhanced monitoring every 60s

  # Connectivity
  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.database_subnet_ids
  security_group_ids = [aws_security_group.postgres.id]

  # Credentials — sourced from AWS Secrets Manager, NOT hardcoded
  username = "aetherflow_admin"
  # Password injected at apply time via: -var="db_password=$(aws secretsmanager get-secret-value ...)"

  tags = local.common_tags
}
```

**Read Replica (for query-api):**

```hcl
module "postgres_replica_query" {
  source = "../../modules/rds-postgres"

  identifier          = "aetherflow-prod-replica-query"
  replicate_source_db = module.postgres_primary.db_instance_id
  instance_class      = "db.r6g.xlarge"

  # Read replicas: no backups, no multi-az
  backup_retention_days = 0
  multi_az              = false

  tags = merge(local.common_tags, { "purpose" = "query-api-read-replica" })
}
```

---

### `modules/iam-service-account`

Creates an IAM role trusted by a specific Kubernetes service account (IRSA pattern). Services use this to access AWS resources without static credentials.

```hcl
module "ingestion_api_irsa" {
  source = "../../modules/iam-service-account"

  role_name             = "aetherflow-prod-ingestion-api"
  eks_cluster_oidc_url  = module.eks.cluster_oidc_issuer_url
  kubernetes_namespace  = "production"
  kubernetes_service_account = "ingestion-api"

  policy_arns = [
    aws_iam_policy.ingestion_api_secrets.arn,  # Access to Secrets Manager
    aws_iam_policy.ingestion_api_s3.arn,       # Write access to raw event archive bucket
  ]
}
```

---

## 5. Key Resource Configurations

### VPC Layout

```hcl
# Production VPC: 10.0.0.0/16
# Subnets:
#   Public:   10.0.0.0/20,  10.0.16.0/20,  10.0.32.0/20   (3 AZs — Load balancers only)
#   Private:  10.0.48.0/20, 10.0.64.0/20,  10.0.80.0/20   (3 AZs — EKS nodes, app traffic)
#   Database: 10.0.96.0/24, 10.0.97.0/24,  10.0.98.0/24   (3 AZs — RDS, ElastiCache)

module "vpc" {
  source = "../../modules/vpc"

  name = "aetherflow-prod"
  cidr = "10.0.0.0/16"
  azs  = ["us-east-1a", "us-east-1b", "us-east-1c"]

  public_subnets   = ["10.0.0.0/20",  "10.0.16.0/20", "10.0.32.0/20"]
  private_subnets  = ["10.0.48.0/20", "10.0.64.0/20", "10.0.80.0/20"]
  database_subnets = ["10.0.96.0/24", "10.0.97.0/24", "10.0.98.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = false  # One NAT GW per AZ for HA
  enable_dns_hostnames   = true
  enable_dns_support     = true
}
```

### S3 Buckets

All S3 buckets follow a strict naming and configuration standard:

```hcl
resource "aws_s3_bucket" "raw_event_archive" {
  bucket = "aetherflow-prod-raw-event-archive"
  tags   = local.common_tags
}

resource "aws_s3_bucket_versioning" "raw_event_archive" {
  bucket = aws_s3_bucket.raw_event_archive.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "raw_event_archive" {
  bucket = aws_s3_bucket.raw_event_archive.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3_encryption.arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "raw_event_archive" {
  bucket                  = aws_s3_bucket.raw_event_archive.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

> ⚠️ **S3 public access block is non-negotiable.** Every S3 bucket must have all four `block_public_*` settings set to `true`. The CI pipeline runs `tflint` with a rule that fails the plan if this is missing.

---

## 6. State Management

Terraform state is stored in S3 with DynamoDB locking.

```hcl
# environments/production/backend.tf
terraform {
  backend "s3" {
    bucket         = "aetherflow-terraform-state-prod"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123..."

    # DynamoDB for state locking — prevents concurrent applies
    dynamodb_table = "aetherflow-terraform-locks"
  }

  required_version = "~> 1.7"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
  }
}
```

**State access** is restricted to:
- The GitHub Actions OIDC role (`arn:aws:iam::123456789012:role/github-actions-terraform`)
- SRE engineers (via individual IAM user policies in the `aetherflow-iam` Terraform module)

**Never run `terraform state` commands without SRE lead sign-off.** State manipulation can cause Terraform to lose track of real resources, resulting in duplicate infrastructure or orphaned resources.

---

## 7. Secret Injection Pattern

Secrets are never stored in `.tfvars` files or in the Terraform state as plaintext. We use two patterns:

### Pattern A: AWS Secrets Manager (Preferred)

Sensitive values (DB passwords, API keys) are stored in AWS Secrets Manager and referenced at `apply` time:

```bash
# Fetch secret and inject as Terraform variable
DB_PASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id "prod/postgres/aetherflow-admin" \
  --query 'SecretString' --output text | jq -r '.password')

terraform apply -var="db_password=${DB_PASSWORD}" tfplan
```

In CI, this is done via the GitHub Actions OIDC role — no static AWS credentials are stored in GitHub.

### Pattern B: `data` source lookup (for non-sensitive config)

```hcl
# Look up an existing secret ARN to pass to an ECS task or Kubernetes secret
data "aws_secretsmanager_secret" "stripe_api_key" {
  name = "prod/billing-service/stripe-api-key"
}

resource "kubernetes_secret" "billing_service" {
  metadata {
    name      = "billing-service-secrets"
    namespace = "production"
  }
  data = {
    stripe_api_key_arn = data.aws_secretsmanager_secret.stripe_api_key.arn
  }
}
```

See `policy-secret-management.md` for the full secrets handling policy.

---

## 8. Drift Detection

AetherFlow runs automated Terraform drift detection every 6 hours via a GitHub Actions scheduled workflow:

```yaml
# .github/workflows/drift-detection.yml
on:
  schedule:
    - cron: '0 */6 * * *'

jobs:
  drift-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Terraform Plan (drift check)
        run: |
          terraform init
          terraform plan -detailed-exitcode -no-color 2>&1 | tee drift-output.txt
          EXIT_CODE=${PIPESTATUS[0]}
          if [ $EXIT_CODE -eq 2 ]; then
            echo "DRIFT DETECTED"
            # Post to #infra-alerts Slack channel
            curl -X POST $SLACK_WEBHOOK_URL \
              -H 'Content-type: application/json' \
              --data "{\"text\":\"⚠️ Terraform drift detected in production. See GitHub Actions run: $GITHUB_SERVER_URL/$GITHUB_REPOSITORY/actions/runs/$GITHUB_RUN_ID\"}"
          fi
        working-directory: infra/environments/production/
```

Drift alerts appear in `#infra-alerts`. When drift is detected:

1. Identify the drifted resource from the plan output
2. Determine if it was a manual change (investigate why) or an external event (AWS maintenance, etc.)
3. Either reconcile the manual change into Terraform code, or run `terraform apply` to revert the drift
4. Document the incident in the drift log: `infra/docs/drift-log.md`

---

## 9. Common Patterns & Recipes

### Adding a New Security Group Rule

```hcl
# Always use specific CIDR blocks or security group references — never 0.0.0.0/0
resource "aws_security_group_rule" "allow_ingestion_api_to_postgres" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.ingestion_api.id
  security_group_id        = aws_security_group.postgres.id
  description              = "Allow ingestion-api pods to connect to PostgreSQL"
}
```

### Tagging Standard

All resources must carry the `local.common_tags` map. Define it once per environment:

```hcl
# environments/production/main.tf
locals {
  common_tags = {
    Environment = "production"
    ManagedBy   = "terraform"
    Repository  = "github.com/aetherflow/infra"
    Team        = "sre"
    CostCenter  = "engineering"
  }
}
```

Cost Explorer reports are filtered by `Environment` and `CostCenter` tags. Resources without these tags will be flagged in our monthly FinOps review.

### Creating a New IAM Policy

```hcl
resource "aws_iam_policy" "my_service_s3_read" {
  name        = "aetherflow-prod-my-service-s3-read"
  description = "Allows my-service to read from the event archive bucket"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.raw_event_archive.arn,
          "${aws_s3_bucket.raw_event_archive.arn}/*"
        ]
      }
    ]
  })
}
```

Use `jsonencode()` for inline policies — never raw JSON strings. It allows Terraform to validate the structure.

---

## 10. Tribal Knowledge & Gotchas

- **The `deletion_protection = true` trap:** RDS and several other resources have `deletion_protection`. Terraform cannot destroy them while this is set. You will get a confusing error during `terraform destroy` or resource replacement. You must first apply a change that sets `deletion_protection = false`, then destroy. This is intentional — it's saved us from accidental database destruction twice.

- **EKS node group replacements are destructive.** Changing `instance_type` in a node group forces replacement — Terraform will terminate old nodes and provision new ones. For the Kafka node group (which has `min_size = max_size = 3`), this causes a temporary Kafka cluster degradation. Always coordinate node group changes with the SRE team and schedule during low-traffic windows.

- **AWS provider version upgrades need careful review.** The `hashicorp/aws` provider has broken resource schemas between minor versions (e.g., `4.x → 5.x`). We pin to `~> 5.30` and bump versions intentionally, with a full staging plan review before production.

- **Terraform plan output can be misleading for `ignore_changes`.** Several of our resources use `lifecycle { ignore_changes = [...] }` to ignore fields managed externally (e.g., EKS node group desired count managed by the cluster autoscaler). The plan will say `No changes` for these fields even if they differ from state. This is intentional — do not "fix" it by removing `ignore_changes`.

- **DynamoDB state locking will block concurrent applies.** If a `terraform apply` process is killed mid-run, it may leave the DynamoDB lock in place. You'll see: `Error: Error acquiring the state lock`. Check the DynamoDB `aetherflow-terraform-locks` table for the lock entry and confirm no apply is actually running before manually deleting it. Confirm with the SRE lead first.

- **`terraform import` is a last resort.** If a resource was created manually outside Terraform, use `terraform import` to bring it under management. But: import only captures the resource itself, not its dependencies or associated resources. Always run `terraform plan` immediately after import and reconcile any diffs before merging.

- **The `infra` repository requires SRE review on all PRs.** This is enforced via CODEOWNERS (`@aetherflow/sre`). Do not try to route around it — infrastructure changes without SRE review are a primary cause of production incidents in this industry.

---

*Document Owner: SRE Team | Review Cycle: Quarterly | Next Review: 2024-05-14*
*See also: `runbook-production-deployment.md`, `runbook-database-failover.md`, `policy-secret-management.md`*
*Terraform state console: https://s3.console.aws.amazon.com/s3/buckets/aetherflow-terraform-state-prod*
