---
title: "Production Deployment Runbook"
author: "Platform Team"
tags: ["deployment", "kubernetes", "ci-cd", "rollback", "runbook", "production", "helm"]
last_updated: "2023-11-02"
version: "1.8.0"
---

# Production Deployment Runbook

> **Audience:** All backend and platform engineers who own a production service.
> **Purpose:** Define the standard, safe procedure for deploying changes to AetherFlow's production environment.
> **Related Docs:** `runbook-incident-response.md`, `onboarding-cicd-overview.md`, `policy-code-review.md`

---

## Table of Contents

1. [Pre-Deployment Checklist](#1-pre-deployment-checklist)
2. [Deployment Windows](#2-deployment-windows)
3. [Deployment Methods](#3-deployment-methods)
4. [Canary Deployment Procedure](#4-canary-deployment-procedure)
5. [Rollback Procedures](#5-rollback-procedures)
6. [Post-Deployment Validation](#6-post-deployment-validation)
7. [Database Migration Protocol](#7-database-migration-protocol)
8. [Feature Flag Management](#8-feature-flag-management)
9. [Tribal Knowledge & Gotchas](#9-tribal-knowledge--gotchas)

---

## 1. Pre-Deployment Checklist

Before any production deployment, the deploying engineer **must** complete every item in this checklist. If any item cannot be checked off, the deployment must be postponed.

```
PRE-DEPLOYMENT CHECKLIST — [SERVICE] v[VERSION] — [DATE]
Engineer: @username

Code Quality
[ ] PR has been approved by ≥2 engineers (see policy-code-review.md)
[ ] All CI checks are green on the release commit (build, unit, integration, lint)
[ ] No open CRITICAL or HIGH severity Snyk vulnerabilities introduced by this PR
[ ] CHANGELOG.md updated with this release's changes

Observability
[ ] New code paths have structured log statements at appropriate levels
[ ] Any new business-critical flows have metrics instrumented (Prometheus counters/histograms)
[ ] Alerts have been updated if SLO thresholds change with this release

Infrastructure
[ ] Helm values diff reviewed — no unintended resource limit changes
[ ] If DB migration included: migration has been dry-run against production schema snapshot (see Section 7)
[ ] Dependent service versions confirmed compatible (check service-catalog.md)

Communication
[ ] Deployment posted to #deployments Slack channel at least 30 min before go-live
[ ] On-call engineer for this service is aware and available for the next 2 hours
[ ] If this is a user-facing change: #product-updates notified
```

---

## 2. Deployment Windows

### Standard Windows

| Environment | Allowed Window | Rationale |
|-------------|---------------|-----------|
| **Production** | Tue–Thu, 10:00–16:00 UTC | Avoids Monday chaos and Friday risk; full team available |
| **Staging** | Any time | No restrictions |
| **Development** | Any time | No restrictions |

### Blackout Periods

Production deployments are **frozen** during the following periods. A VP of Engineering waiver is required to deploy during a freeze:

- Last 3 business days of each month (billing cycle close)
- 72 hours before and after a major product launch (announced in #product-all)
- Any active SEV-1 or SEV-2 incident (see `runbook-incident-response.md`)
- December 22 – January 2 (holiday freeze)

### Emergency Deployments (Hotfixes)

A deployment outside the standard window requires:
1. Explicit written approval from the on-call Engineering Manager (in `#deployments`)
2. Canary deployment only — no direct full rollouts for emergency hotfixes
3. IC-style monitoring in `#deployments` for 30 minutes post-deploy

---

## 3. Deployment Methods

All production services at AetherFlow are deployed via **Helm charts** managed in the `aetherflow/helm-charts` GitHub repository. Our CI/CD system (GitHub Actions + ArgoCD) handles the mechanical deployment. Engineers should almost never `kubectl apply` directly in production.

### Standard Deployment Flow

```
Developer merges PR to main
        │
        ▼
GitHub Actions runs CI pipeline
  - Build Docker image
  - Tag as gcr.io/aetherflow-prod/<service>:<git-sha>
  - Push to GCR
  - Run integration tests against staging
        │
        ▼
ArgoCD detects new image tag in Helm values
        │
        ▼
ArgoCD opens auto-PR against helm-charts/production/<service>/values.yaml
        │
        ▼
Engineer reviews Helm values diff and APPROVES the ArgoCD sync
        │
        ▼
ArgoCD applies rolling update to production namespace
        │
        ▼
Post-deploy validation (see Section 6)
```

### Triggering a Deployment Manually

In cases where the ArgoCD auto-PR doesn't fire (e.g., config-only changes):

```bash
# Ensure you have production kubectl context
kubectl config use-context aetherflow-prod-us-east-1

# Verify you're in the right context before touching anything
kubectl config current-context
# Expected output: aetherflow-prod-us-east-1

# Manually trigger a Helm upgrade (use with caution)
helm upgrade <service-name> ./helm-charts/services/<service-name> \
  --namespace production \
  --values ./helm-charts/services/<service-name>/values-prod.yaml \
  --set image.tag=<GIT_SHA> \
  --atomic \          # Auto-rollback if deployment fails
  --timeout 5m \
  --wait

# Watch the rollout
kubectl rollout status deployment/<service-name> -n production --timeout=5m
```

> ⚠️ **Never use `--force` in production.** It bypasses safety checks and can cause pod termination without a graceful drain. If you think you need `--force`, call the SRE on-call first.

---

## 4. Canary Deployment Procedure

All deployments for services that handle > 1,000 req/min in production **must** use a canary strategy. This applies to: `ingestion-api`, `stream-processor`, `query-api`, `auth-service`.

The other services (internal tooling, background workers with no user-facing SLAs) may use a standard rolling update.

### Canary Steps

```bash
# Step 1: Deploy the canary (10% traffic weight)
# Edit the Argo Rollout resource to set the new image
kubectl argo rollouts set image rollout/<service-name> \
  <service-name>=gcr.io/aetherflow-prod/<service-name>:<NEW_TAG> \
  -n production

# Step 2: Check canary health — watch for 10 minutes
kubectl argo rollouts get rollout <service-name> -n production --watch

# Step 3: Inspect the canary's error rate in Grafana
# Dashboard: https://grafana.aetherflow.internal/d/canary-analysis
# Threshold: canary error rate must be ≤ baseline error rate + 0.5%

# Step 4a: If healthy — promote to 100%
kubectl argo rollouts promote <service-name> -n production

# Step 4b: If unhealthy — abort and auto-rollback
kubectl argo rollouts abort <service-name> -n production
```

### Canary Analysis Template (Argo Rollouts)

```yaml
# k8s/rollouts/analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: aetherflow-canary-analysis
  namespace: production
spec:
  metrics:
    - name: error-rate
      interval: 60s
      successCondition: result[0] <= 0.01
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{
              job="{{ args.service-name }}",
              status=~"5..",
              pod=~".*canary.*"
            }[5m])) /
            sum(rate(http_requests_total{
              job="{{ args.service-name }}",
              pod=~".*canary.*"
            }[5m]))
```

---

## 5. Rollback Procedures

### Automatic Rollback (preferred)

If `--atomic` was used in the Helm upgrade, a failed rollout (pods crashing, failing health checks) will automatically roll back. Verify:

```bash
# Check rollout history
kubectl rollout history deployment/<service-name> -n production

# Confirm the current running revision
kubectl rollout status deployment/<service-name> -n production
```

### Manual Rollback

```bash
# Option A: Roll back to the immediately previous revision
kubectl rollout undo deployment/<service-name> -n production

# Option B: Roll back to a specific revision number
kubectl rollout undo deployment/<service-name> -n production --to-revision=<N>

# Option C: Pin to a known-good image tag via Helm (most explicit — preferred for SEV-1s)
helm upgrade <service-name> ./helm-charts/services/<service-name> \
  --namespace production \
  --values ./helm-charts/services/<service-name>/values-prod.yaml \
  --set image.tag=<LAST_KNOWN_GOOD_SHA> \
  --atomic \
  --timeout 5m

# After rollback, verify
kubectl get pods -n production -l app=<service-name>
```

> 📢 **Always post in `#deployments`** when performing a manual rollback. Format:
> `ROLLBACK: <service> from <bad-tag> → <good-tag>. Reason: [one sentence]. IC: @you`

---

## 6. Post-Deployment Validation

Run the following validation steps for **every** production deployment, canary or full:

```bash
#!/bin/bash
# scripts/post-deploy-validate.sh
# Usage: ./post-deploy-validate.sh <service-name> <expected-version>

SERVICE=$1
EXPECTED_VERSION=$2
NAMESPACE="production"

echo "=== Post-Deploy Validation: $SERVICE ==="

# 1. Confirm all pods are Running
echo "[1/4] Checking pod health..."
RUNNING=$(kubectl get pods -n $NAMESPACE -l app=$SERVICE \
  --field-selector=status.phase=Running --no-headers | wc -l)
TOTAL=$(kubectl get pods -n $NAMESPACE -l app=$SERVICE --no-headers | wc -l)

if [ "$RUNNING" -ne "$TOTAL" ]; then
  echo "❌ FAIL: Only $RUNNING/$TOTAL pods are Running"
  exit 1
fi
echo "✅ $RUNNING/$TOTAL pods Running"

# 2. Confirm the deployed image version
echo "[2/4] Checking image version..."
ACTUAL_VERSION=$(kubectl get deployment/$SERVICE -n $NAMESPACE \
  -o jsonpath='{.spec.template.spec.containers[0].image}' | cut -d: -f2)
if [ "$ACTUAL_VERSION" != "$EXPECTED_VERSION" ]; then
  echo "❌ FAIL: Expected $EXPECTED_VERSION, got $ACTUAL_VERSION"
  exit 1
fi
echo "✅ Image version: $ACTUAL_VERSION"

# 3. Check the /healthz endpoint
echo "[3/4] Checking health endpoint..."
HEALTH=$(kubectl run --rm -i --restart=Never health-check \
  --image=curlimages/curl -- curl -sf \
  "http://$SERVICE.$NAMESPACE.svc.cluster.local/healthz" 2>/dev/null)
if [ $? -ne 0 ]; then
  echo "❌ FAIL: /healthz returned non-200"
  exit 1
fi
echo "✅ /healthz OK"

# 4. Check error rate in Prometheus (last 5 min)
echo "[4/4] Checking error rate..."
echo "   → Open Grafana: https://grafana.aetherflow.internal/d/service-health"
echo "   → Confirm error rate for $SERVICE is within SLO bounds"
echo ""
echo "=== Validation complete. Manual Grafana check required. ==="
```

---

## 7. Database Migration Protocol

Schema migrations are the highest-risk part of any deployment. AetherFlow uses **golang-migrate** for all PostgreSQL migrations.

### Rules

1. **All migrations must be backward-compatible.** The new code must be able to run against the old schema (in case of rollback) and against the new schema.
2. **Never DROP a column in the same PR as the code that stops using it.** Drop columns in a separate PR at least one week after the code has been fully deployed.
3. **Always test migrations against the production schema snapshot** before deploying. The snapshot is refreshed daily to `postgres-snapshot.aetherflow.internal`.

### Running a Migration

```bash
# Dry-run against the production schema snapshot first
migrate -path ./db/migrations \
        -database "postgres://aetherflow_admin:${SNAPSHOT_PW}@postgres-snapshot.aetherflow.internal:5432/aetherflow?sslmode=require" \
        -verbose \
        up --dry-run

# If dry-run passes, apply to production (done by CI — do not run manually)
# Manual migration (emergency only, with SRE approval):
migrate -path ./db/migrations \
        -database "postgres://aetherflow_admin:${PROD_PW}@postgres-primary.aetherflow.internal:5432/aetherflow?sslmode=require" \
        -verbose \
        up 1   # Apply exactly ONE migration at a time in emergencies
```

### Large Table Migrations

For migrations on tables with > 5 million rows (currently: `events`, `metrics`, `audit_log`), use `pg_repack` or a background migration pattern. **Never run a full-table `ALTER TABLE` on these tables during business hours.** Consult the Platform Team before writing the migration.

---

## 8. Feature Flag Management

AetherFlow uses **LaunchDarkly** for feature flags. All new user-facing features must be gated behind a flag during initial rollout.

```go
// Standard feature flag check pattern
import "github.com/launchdarkly/go-server-sdk/v6/ldapi"

func (h *Handler) handleStreamReplay(ctx context.Context, tenantID string) error {
    flagValue := h.ldClient.BoolVariation(
        "stream-replay-enabled",
        ldapi.NewEvaluationContext(tenantID, nil),
        false, // safe default
    )
    if !flagValue {
        return ErrFeatureNotAvailable
    }
    // ... feature logic
}
```

Flag naming convention: `<feature-slug>-enabled` (e.g., `stream-replay-enabled`, `new-dashboard-enabled`).

Flag management lives in the LaunchDarkly project `aetherflow-production`. All engineers have read access; write access is restricted to senior engineers and team leads.

---

## 9. Tribal Knowledge & Gotchas

- **The ArgoCD sync button is dangerous.** ArgoCD's "Sync" button in the UI will apply the current Git state to production immediately, without a canary. Never click it without double-checking what the diff is. When in doubt, use the CLI with `--dry-run` first.

- **Helm `--atomic` does not protect against slow memory leaks.** A deployment might pass all health checks but introduce a memory leak that only becomes apparent after 4–6 hours of traffic. Always check memory trending in Grafana 2 hours post-deploy for any service touching the events pipeline.

- **`ingestion-api` has a 90-second graceful shutdown.** Our Kubernetes `terminationGracePeriodSeconds` for ingestion-api is set to 120s for this reason. If you lower it thinking you're speeding up deployments, you will drop in-flight events. This is configured in `helm-charts/services/ingestion-api/values-prod.yaml` and changing it requires a Platform Team review.

- **The staging environment is NOT a perfect mirror of production.** Staging runs with 20% of the replica count and uses a different (smaller) Kafka partition layout. A performance test passing on staging does not guarantee production performance. Load test against the production snapshot environment (`loadtest.aetherflow.internal`) for anything throughput-sensitive.

- **GitHub Actions cache poisoning:** We've had two incidents where a corrupt Go module cache in GitHub Actions caused a deployment with a subtly wrong dependency version. If you see inexplicable behavior after a deploy where the code looks correct, try forcing a cache-busted build by adding `[no-cache]` to your commit message.

---

*Document Owner: Platform Team | Review Cycle: Quarterly | Next Review: 2024-02-01*
*See also: `runbook-incident-response.md`, `onboarding-cicd-overview.md`, `policy-code-review.md`*
