---
title: "Secrets & Credentials Management Policy"
author: "Security Team"
tags: ["secrets", "security", "policy", "vault", "credentials", "iam", "1password", "compliance"]
last_updated: "2023-07-22"
version: "1.3.0"
---

# Secrets & Credentials Management Policy

> **Audience:** All engineers, SREs, and anyone who handles credentials at AetherFlow.
> **Purpose:** Define how secrets are stored, rotated, injected into services, and audited. Non-compliance with this policy is a security incident.
> **Related Docs:** `onboarding-env-setup.md`, `reference-infra-terraform.md`, `policy-code-review.md`, `onboarding-cicd-overview.md`

If you commit a secret to Git — even for one minute, even to a private repository — treat it as fully compromised and rotate it immediately. GitHub's push scanning cannot unwrite history, and automated scrapers index new commits within seconds.

---

## Table of Contents

1. [Secret Classification](#1-secret-classification)
2. [Prohibited Practices](#2-prohibited-practices)
3. [Secret Storage by Context](#3-secret-storage-by-context)
4. [Runtime Secret Injection](#4-runtime-secret-injection)
5. [Vault: Structure & Access Patterns](#5-vault-structure--access-patterns)
6. [Secret Rotation Policy](#6-secret-rotation-policy)
7. [Emergency: Secret Leaked to Git](#7-emergency-secret-leaked-to-git)
8. [Audit & Compliance](#8-audit--compliance)
9. [Tribal Knowledge & Gotchas](#9-tribal-knowledge--gotchas)

---

## 1. Secret Classification

Not all secrets carry equal risk. Classification determines storage location, rotation frequency, and access controls.

| Class | Examples | Storage | Rotation |
|-------|---------|---------|---------|
| **CRITICAL** | Database root credentials, Stripe live API key, TLS private keys, Vault root token | AWS Secrets Manager + Vault | 90 days or immediately on suspected compromise |
| **HIGH** | Service-to-service mTLS certificates, GitHub Actions deploy tokens, LaunchDarkly server SDK key, internal API keys | Vault (service secret path) | 180 days |
| **MEDIUM** | Third-party webhook signing secrets, SendGrid API key, Datadog API key | AWS Secrets Manager | 365 days |
| **LOW** | Non-production API keys, local dev credentials, staging database passwords | 1Password (Engineering vault) | Annual review |

> ⚠️ **When in doubt, classify higher.** The cost of over-rotating a LOW secret is a few minutes of work. The cost of under-protecting a CRITICAL secret can be existential for the company.

---

## 2. Prohibited Practices

The following are **never acceptable** at AetherFlow. Each prohibition is backed by a real incident or near-miss:

```
❌ Hardcoding a secret in source code (any language, any file type)
❌ Committing a secret to any Git repository (public or private)
❌ Storing a secret in a Kubernetes ConfigMap (ConfigMaps are not encrypted at rest)
❌ Storing a secret in a Helm values file committed to the helm-charts repo
❌ Logging a secret (structured or unstructured — check your debug log statements)
❌ Sending a secret over Slack, email, or any messaging platform
❌ Storing production secrets in a local .env or .envrc file committed to Git
❌ Sharing a production secret between two engineers via any channel other than
   Vault or 1Password (direct handoff in a secure vault only)
❌ Using a personal AWS IAM user key in a CI/CD system (use OIDC/workload identity)
❌ Reusing secrets across environments (staging and production must have distinct credentials)
```

### What the `pre-commit` Hook Catches

The `gitleaks` scanner in our pre-commit hook (installed via `make install-hooks` — see `onboarding-env-setup.md`) detects patterns for:

- AWS access keys (`AKIA*`)
- Generic high-entropy strings > 40 chars in quoted contexts
- Private key PEM headers (`-----BEGIN * PRIVATE KEY-----`)
- Common API key prefixes (Stripe `sk_live_*`, SendGrid `SG.*`, etc.)

This hook is **not a complete safety net** — it catches common patterns but is not exhaustive. Human vigilance is required.

---

## 3. Secret Storage by Context

### Production Services

All production secrets are stored in **AWS Secrets Manager** and/or **HashiCorp Vault**, and injected at runtime. No production secret exists on any engineer's laptop or in any Git repository.

```
Production Secret Locations:
├── AWS Secrets Manager
│   ├── prod/postgres/aetherflow-admin        (DB master credentials)
│   ├── prod/postgres/service-accounts/       (per-service DB users)
│   ├── prod/stripe/live-api-key
│   ├── prod/sendgrid/api-key
│   ├── prod/datadog/api-key
│   └── prod/launchdarkly/sdk-key
│
└── HashiCorp Vault (vault.aetherflow.internal)
    ├── secret/production/<service-name>/     (service-specific secrets)
    ├── pki/                                  (mTLS certificate authority)
    └── aws/                                  (dynamic AWS credentials — advanced use only)
```

### CI/CD Secrets

Pipeline secrets live in **GitHub Actions organization secrets** (not repository secrets) and are managed exclusively by the Platform team. See `onboarding-cicd-overview.md` §9 for the full list.

**Engineers must never add secrets directly to GitHub repository settings** — they must be added at the organization level by the Platform team after a security review.

### Local Development

Local dev secrets are stored in **1Password** (AetherFlow Engineering vault). All engineers have read access from their first day. The local dev Vault instance (`http://localhost:8200`) is pre-seeded with non-production credentials via `make db-seed-local`.

```bash
# Pull a local dev secret from 1Password CLI
op item get "local-dev/ingestion-api" --fields "DATABASE_URL"

# Or use the 1Password shell plugin (if configured)
# Secrets are injected into your shell session, not persisted to disk
op run --env-file=.env.1password -- go run ./cmd/ingestion-api/main.go
```

---

## 4. Runtime Secret Injection

AetherFlow uses two patterns for getting secrets into running pods. **Never use environment variables set from a plain Kubernetes ConfigMap** for secret values.

### Pattern A: Vault Agent Sidecar (Preferred for CRITICAL and HIGH)

The Vault Agent runs as a sidecar container and writes secrets to a shared in-memory volume (`/vault/secrets/`). The application reads from the file. Secrets are automatically rotated by Vault Agent without a pod restart.

```yaml
# k8s/services/ingestion-api/deployment.yaml (excerpt)
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "ingestion-api-production"
        vault.hashicorp.com/agent-inject-secret-db-credentials: |
          secret/production/ingestion-api/db-credentials
        vault.hashicorp.com/agent-inject-template-db-credentials: |
          {{- with secret "secret/production/ingestion-api/db-credentials" -}}
          DATABASE_URL=postgres://{{ .Data.data.username }}:{{ .Data.data.password }}@pgbouncer.aetherflow.internal:5432/aetherflow
          {{- end }}
    spec:
      serviceAccountName: ingestion-api  # IRSA-linked SA with Vault auth
      containers:
        - name: ingestion-api
          image: gcr.io/aetherflow-prod/ingestion-api:latest
          command: ["/bin/sh", "-c"]
          args:
            # Source the secret file injected by Vault Agent
            - "source /vault/secrets/db-credentials && exec /app/ingestion-api"
```

```go
// Application code — reads from env var populated by the sourced secret file
// No Vault SDK dependency in application code; secrets arrive as env vars
dbURL := os.Getenv("DATABASE_URL")
if dbURL == "" {
    log.Fatal("DATABASE_URL not set — Vault Agent may not have initialized")
}
```

### Pattern B: Kubernetes Secrets from AWS Secrets Manager (for MEDIUM)

For lower-classification secrets, we use the **AWS Secrets Store CSI Driver** to sync AWS Secrets Manager values into Kubernetes Secrets, which are then mounted as environment variables.

```yaml
# k8s/services/notification-service/secret-provider-class.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: notification-service-secrets
  namespace: production
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/sendgrid/api-key"
        objectType: "secretsmanager"
        objectAlias: "SENDGRID_API_KEY"
  secretObjects:
    - secretName: notification-service-env
      type: Opaque
      data:
        - objectName: SENDGRID_API_KEY
          key: sendgrid-api-key
```

```yaml
# Deployment spec
envFrom:
  - secretRef:
      name: notification-service-env
```

### Anti-Pattern: What NOT to Do

```yaml
# ❌ NEVER — hardcoded secret in Deployment manifest
env:
  - name: DATABASE_PASSWORD
    value: "supersecretpassword123"

# ❌ NEVER — secret in ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_PASSWORD: "supersecretpassword123"  # ConfigMaps are plaintext in etcd
```

---

## 5. Vault: Structure & Access Patterns

### Vault Authentication

Services authenticate to Vault using **Kubernetes Service Account tokens** (the `kubernetes` auth method). Engineers authenticate using **OIDC via Google Workspace** (SSO).

```bash
# Engineer login (interactive)
vault login -method=oidc -path=gsuite

# Verify your token and policies
vault token lookup

# List secrets you have access to
vault kv list secret/production/ingestion-api/
```

### Vault Policy Structure

Each service has a dedicated Vault policy granting read-only access to its own secret path and no other:

```hcl
# vault/policies/ingestion-api-production.hcl
# Policy: ingestion-api-production
# Scope: read-only access to this service's secrets only

path "secret/data/production/ingestion-api/*" {
  capabilities = ["read"]
}

# Allow the service to renew its own token
path "auth/token/renew-self" {
  capabilities = ["update"]
}

# Deny everything else explicitly
path "*" {
  capabilities = ["deny"]
}
```

Human access policies are role-based:

| Role | Access |
|------|--------|
| `engineer` | Read own service secrets, read staging secrets, no prod write |
| `senior-engineer` | Read all service secrets, create staging secrets |
| `sre` | Read/write all secrets, manage rotation |
| `security-lead` | Full Vault admin |

### Creating a New Secret

```bash
# Only SRE or security-lead role can create production secrets
vault kv put secret/production/my-new-service/api-credentials \
  api_key="$(openssl rand -base64 32)" \
  api_secret="$(openssl rand -base64 64)"

# Verify it was written
vault kv get secret/production/my-new-service/api-credentials

# Add the corresponding Vault policy (in infra/vault/policies/ — Terraform managed)
# File a PR against infra repo to add the policy and IRSA binding
```

---

## 6. Secret Rotation Policy

### Rotation Schedule

Rotation is automated where possible. The following table shows what is automated vs. manual:

| Secret | Rotation Frequency | Method | Automated? |
|--------|-------------------|--------|-----------|
| mTLS certificates (internal services) | 30 days | Vault PKI auto-rotation | ✅ Yes |
| PostgreSQL service account passwords | 90 days | Vault dynamic credentials | ✅ Yes |
| GitHub Actions OIDC tokens | Per-workflow | GitHub OIDC (ephemeral) | ✅ Yes |
| Stripe live API key | 365 days | Manual — Security team | ❌ No |
| SendGrid API key | 365 days | Manual — Security team | ❌ No |
| Datadog API key | 365 days | Manual — Security team | ❌ No |
| LaunchDarkly SDK key | 180 days | Manual — Platform team | ❌ No |
| PostgreSQL master password | 90 days | Manual — SRE team | ❌ No |

### Manual Rotation Procedure

For manual rotations, the responsible team follows this procedure:

```bash
# Example: Rotating the SendGrid API key

# 1. Generate a new API key in the SendGrid dashboard
#    (do this BEFORE updating Vault — have both keys active simultaneously)

# 2. Update the secret in AWS Secrets Manager
aws secretsmanager put-secret-value \
  --secret-id "prod/sendgrid/api-key" \
  --secret-string '{"api_key":"SG.NEW_KEY_HERE"}' \
  --region us-east-1

# 3. Force pod restarts to pick up the new secret
#    (AWS Secrets Store CSI Driver syncs automatically within 2 minutes,
#     but a rolling restart ensures immediate propagation)
kubectl rollout restart deployment/notification-service -n production

# 4. Verify the service is using the new key (check logs for auth errors)
kubectl logs -n production deploy/notification-service --tail=50

# 5. Once confirmed working, revoke the OLD key in the SendGrid dashboard
#    (do NOT revoke the old key before confirming the new one works)

# 6. Update the rotation log:
#    https://wiki.aetherflow.internal/secret-rotation-log
```

> 📋 **The rotation log** (`secret-rotation-log` in Confluence) must be updated within 24 hours of every manual rotation. It records: secret name, rotation date, engineer who performed it, and the next scheduled rotation date. This log is reviewed during SOC 2 audits.

### Emergency Rotation (Suspected Compromise)

If a secret is suspected of being compromised (committed to Git, seen in logs, shared insecurely):

1. **Rotate immediately** — do not wait for confirmation of misuse
2. Post in `#security-incidents` with: secret type, suspected exposure vector, time of exposure
3. Check Vault audit logs for unexpected access to the secret in the past 30 days:
   ```bash
   vault audit list
   # Audit logs are also exported to Datadog — search for:
   # operation="read" path="secret/production/<service>/<secret-name>"
   ```
4. File a security incident ticket in Jira (`SEC` project) within 1 hour

---

## 7. Emergency: Secret Leaked to Git

This is the most common secret incident at technology companies. The procedure is time-critical.

### Immediate Response (< 5 minutes)

```bash
# Step 1: Rotate the secret RIGHT NOW — before doing anything else
# The commit is already public from GitHub's perspective. Rotation first.

# Step 2: Post in #security-incidents
# "SECRET LEAK: [type of secret] committed to [repo] in commit [SHA].
#  Secret rotated at [TIME UTC]. Investigating exposure window."

# Step 3: Do NOT force-push to rewrite Git history yet
# (It doesn't help — GitHub and any clones already have the commit)
# History rewrite is a cleanup step, not a security step
```

### Investigation (< 1 hour)

```bash
# Find when the secret was first committed
git log -p --all -S "EXPOSED_SECRET_VALUE" -- .

# Check if the commit was ever pushed to a remote (it usually was)
git log --remotes -1 --format="%H %ai %ae" <commit-sha>

# Check GitHub's push log for the repository (Security tab in GitHub)
# Determine: was this repo public at any point? Was the branch ever public?

# Check for evidence of secret use in AWS CloudTrail (for AWS keys)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=EXPOSED_KEY_ID \
  --start-time $(date -d '48 hours ago' -u +%Y-%m-%dT%H:%M:%SZ) \
  --region us-east-1
```

### Cleanup (after rotation and investigation)

```bash
# Remove the secret from Git history using git-filter-repo
# Install: pip install git-filter-repo
git filter-repo --replace-text <(echo "EXPOSED_SECRET_VALUE==>REDACTED")

# Force-push the cleaned history (requires branch protection override from SRE)
git push origin main --force-with-lease

# Request GitHub support to clear cached views of the old commits:
# https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository
```

> ⚠️ **Force-pushing `main` requires temporary bypass of branch protection.** Contact an SRE lead or Engineering Manager to enable this. Re-enable branch protection immediately after the force-push.

---

## 8. Audit & Compliance

AetherFlow is SOC 2 Type II compliant. Secret management practices are audited annually. Key audit evidence includes:

| Evidence | Source | Retention |
|----------|--------|-----------|
| Vault audit logs (all secret access) | Vault → Datadog → S3 | 2 years |
| AWS Secrets Manager access logs | CloudTrail → S3 | 2 years |
| Secret rotation log | Confluence | Indefinite |
| GitHub security scan results | GitHub Advanced Security → S3 | 1 year |
| Incident tickets for secret leaks | Jira (`SEC` project) | Indefinite |

### Quarterly Access Review

Every quarter, the Security team runs an access review:

1. Export the list of all Vault policies and their assignments (`vault auth list`, `vault policy list`)
2. Compare against the current employee list in Rippling
3. Revoke access for departed employees and engineers who changed teams
4. Generate a report for the compliance record

Engineers who are found to have excessive secret access (e.g., read access to services they no longer own) will have their access scoped down without notice.

---

## 9. Tribal Knowledge & Gotchas

- **The Vault Agent sidecar has a startup race condition.** Occasionally, the application container starts before the Vault Agent has finished writing secrets to the shared volume. The application will crash on startup with a "DATABASE_URL not set" error. The fix is in our `initContainers` — we added a 5-second sleep in the app container entrypoint as a mitigation. The proper fix (a readiness gate checking for the secret file) is tracked in `PLAT-2991`.

- **AWS Secrets Manager has a propagation delay with the CSI driver.** After updating a secret in Secrets Manager, the CSI driver syncs within **2 minutes** by default (configurable via `syncIntervalHours`). During that window, the old secret is still active in running pods. For immediate propagation, trigger a rolling restart.

- **Kubernetes Secrets are base64-encoded, NOT encrypted** by default in etcd. We have envelope encryption enabled on our EKS cluster (`--encryption-provider-config`), which encrypts secrets at rest in etcd using a KMS key. However, secrets are still visible in plaintext via `kubectl get secret -o yaml` to anyone with RBAC permission. **Treat Kubernetes Secrets as sensitive.** The RBAC policy restricts `get`/`list` on Secrets in the `production` namespace to SRE and service accounts only.

- **`vault kv get` caches credentials locally.** The Vault CLI caches your token in `~/.vault-token`. If you share your laptop or use a shared bastion host, log out of Vault explicitly: `vault token revoke -self`. Do not leave a privileged Vault session open on a shared machine.

- **GitHub's secret scanning only covers the default branch in free tier.** Our GitHub Advanced Security license covers all branches and forks. However, the push scanning hook runs asynchronously — it does not block the push in real time. The `gitleaks` pre-commit hook is your real-time protection.

- **Stripe keys have different prefixes for test vs. live.** `sk_test_*` = test mode (safe to use in staging). `sk_live_*` = live mode (real money, CRITICAL classification). A `sk_live_*` key committed to Git should be treated as a P0 security incident regardless of the repository's visibility.

- **Rotating the PostgreSQL master password requires a Terraform apply.** The master password is set in RDS via Terraform. Rotating it requires updating the secret in Secrets Manager and running `terraform apply` with the new password variable. This does NOT cause downtime (RDS updates the password online), but it does require the standard Terraform review and apply workflow.

---

*Document Owner: Security Team | Review Cycle: Bi-annual | Next Review: 2024-01-22*
*Compliance contact: security@aetherflow.com*
*See also: `onboarding-env-setup.md`, `reference-infra-terraform.md`, `onboarding-cicd-overview.md`*
