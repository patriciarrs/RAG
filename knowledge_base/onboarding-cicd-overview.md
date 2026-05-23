---
title: "CI/CD Pipeline Overview"
author: "Engineering Enablement Team"
tags: ["ci-cd", "github-actions", "argocd", "onboarding", "deployment", "pipeline", "testing", "automation"]
last_updated: "2023-12-05"
version: "2.0.1"
---

# CI/CD Pipeline Overview

> **Audience:** All engineers at AetherFlow, especially those new to the team.
> **Purpose:** Explain how code moves from a developer's laptop to production, covering every automated step in between.
> **Related Docs:** `onboarding-env-setup.md`, `runbook-production-deployment.md`, `policy-code-review.md`, `reference-infra-terraform.md`

Understanding the CI/CD pipeline is non-negotiable for contributing to AetherFlow. This document explains not just *what* the pipeline does, but *why* each step exists and what to do when it breaks.

---

## Table of Contents

1. [Pipeline Philosophy](#1-pipeline-philosophy)
2. [High-Level Flow](#2-high-level-flow)
3. [CI Pipeline: Pull Requests](#3-ci-pipeline-pull-requests)
4. [CD Pipeline: Staging Deployment](#4-cd-pipeline-staging-deployment)
5. [CD Pipeline: Production Deployment](#5-cd-pipeline-production-deployment)
6. [ArgoCD & GitOps](#6-argocd--gitops)
7. [Pipeline for Special Cases](#7-pipeline-for-special-cases)
8. [Debugging Failed Pipelines](#8-debugging-failed-pipelines)
9. [Pipeline Configuration Reference](#9-pipeline-configuration-reference)
10. [Tribal Knowledge & Gotchas](#10-tribal-knowledge--gotchas)

---

## 1. Pipeline Philosophy

AetherFlow's pipeline is built on three principles:

**Principle 1: `main` is always deployable.**
Every commit on `main` must pass all CI checks and be safe to deploy to production at any moment. We never commit directly to `main` — all changes go through a PR. If CI is red on `main`, that is a P0 issue for the team that broke it, not a background task.

**Principle 2: Staging is a required gate, not an optional check.**
All code is deployed to staging before production. Staging uses real Kafka, real PostgreSQL (a separate instance), and runs integration tests. It is not a toy environment.

**Principle 3: Production deployments are intentional human decisions.**
No code is automatically deployed to production. The ArgoCD sync that pushes to production always requires a human to review the diff and approve it. Automation prepares; humans decide.

---

## 2. High-Level Flow

```
Developer pushes branch
        │
        ▼
┌───────────────────────┐
│   PR Opened / Updated  │
│   CI Pipeline runs     │◄────── See Section 3
│   (~8-12 minutes)      │
└───────────┬───────────┘
            │ All checks green
            ▼
┌───────────────────────┐
│   PR Reviewed          │
│   & Merged to main     │◄────── See policy-code-review.md
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│   Staging CD runs      │
│   - Build image        │◄────── See Section 4
│   - Deploy to staging  │
│   - Run E2E tests      │
│   (~15-20 minutes)     │
└───────────┬───────────┘
            │ Staging deployment successful
            ▼
┌───────────────────────┐
│   ArgoCD PR opened     │
│   for production       │◄────── See Section 5 & 6
│   (human reviews diff) │
└───────────┬───────────┘
            │ Engineer approves & triggers sync
            ▼
┌───────────────────────┐
│   Production deployment│
│   (canary or rolling)  │
│   Post-deploy validate │
└───────────────────────┘
```

---

## 3. CI Pipeline: Pull Requests

Every push to a pull request branch triggers the CI pipeline defined in `.github/workflows/ci.yml`. The pipeline is **required to pass** before a PR can be merged.

### Pipeline Steps (in order)

```yaml
# .github/workflows/ci.yml — annotated excerpt
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  # ─── Stage 1: Fast feedback (run in parallel) ──────────────────────────────
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version-file: '.go-version' }
      - name: golangci-lint
        uses: golangci/golangci-lint-action@v4
        with:
          version: v1.55.2
          args: --timeout=5m
      # Also runs: gofmt check, go vet, shadow variable check, staticcheck

  proto-check:
    name: Proto Lint & Breaking Change Detection
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: bufbuild/buf-setup-action@v1
      - name: Buf Lint
        run: buf lint
        working-directory: ../proto  # Checked out separately
      - name: Buf Breaking
        run: buf breaking --against "https://buf.build/aetherflow/proto"
        working-directory: ../proto

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Snyk Go Scan
        uses: snyk/actions/golang@master
        env: { SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }} }
        with:
          args: --severity-threshold=high  # CRITICAL and HIGH block merge
      - name: Secret Scan (gitleaks)
        uses: gitleaks/gitleaks-action@v2

  # ─── Stage 2: Tests (depend on lint passing) ───────────────────────────────
  unit-tests:
    name: Unit Tests
    needs: [lint]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version-file: '.go-version' }
      - name: Run unit tests with race detector
        run: go test ./... -short -race -count=1 -coverprofile=coverage.out
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.out
          fail_ci_if_error: false  # Coverage upload failure doesn't block merge

  integration-tests:
    name: Integration Tests
    needs: [lint]
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15.5
        env:
          POSTGRES_DB: aetherflow_test
          POSTGRES_USER: aetherflow
          POSTGRES_PASSWORD: testpassword
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10
      redis:
        image: redis:7.2-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version-file: '.go-version' }
      - name: Run integration tests
        run: go test ./... -tags=integration -count=1 -timeout=10m
        env:
          DATABASE_URL: postgres://aetherflow:testpassword@localhost:5432/aetherflow_test
          REDIS_URL: redis://localhost:6379
          # Kafka is NOT spun up in CI integration tests — Kafka interactions are mocked
          # Real Kafka integration is tested in the staging E2E suite (Section 4)

  # ─── Stage 3: Build (depends on all tests passing) ─────────────────────────
  build:
    name: Build & Push Image
    needs: [unit-tests, integration-tests, security-scan]
    runs-on: ubuntu-latest
    # Only build on pushes to main — not on every PR commit
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Authenticate to GCR
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            gcr.io/aetherflow-prod/${{ env.SERVICE_NAME }}:${{ github.sha }}
            gcr.io/aetherflow-prod/${{ env.SERVICE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_VERSION=${{ github.sha }}
            BUILD_TIME=${{ github.run_started_at }}
```

### CI Duration Targets

| Stage | Target Duration | If Over Target |
|-------|----------------|----------------|
| Lint | < 2 minutes | Investigate new lint rules or slow static analysis |
| Unit tests | < 3 minutes | Check for missing `-short` tags or slow test setup |
| Integration tests | < 5 minutes | Check for unindexed test queries or missing test parallelism |
| Build | < 4 minutes | Check layer caching; flag to Platform if persistent |

**Total CI target: < 12 minutes.** Pipeline times are tracked in Datadog CI Visibility. If the pipeline consistently exceeds 12 minutes, the Platform team will investigate.

---

## 4. CD Pipeline: Staging Deployment

Triggered automatically on every merge to `main`. Defined in `.github/workflows/deploy-staging.yml`.

### Steps

```
1. Trigger: merge to main
2. Pull the image built in the CI build step (same SHA)
3. Update helm-charts/staging/<service>/values.yaml with new image tag
4. Commit the values change to helm-charts repo
5. ArgoCD detects the change and syncs to staging automatically
6. Run E2E test suite against staging (see below)
7. If E2E passes: open ArgoCD PR for production values update
8. If E2E fails: post failure to #deployments; block production PR
```

### E2E Test Suite

The E2E suite lives in `tests/e2e/` and runs against the live staging environment after each deployment. It is written in Go using `testify` and exercises full customer-facing flows:

```go
// tests/e2e/ingestion_test.go — example
func TestEventIngestionEndToEnd(t *testing.T) {
    ctx := context.Background()
    client := newStagingClient(t)

    // 1. Ingest an event
    eventID := uuid.New().String()
    resp, err := client.IngestEvent(ctx, &api.IngestEventRequest{
        TenantID:  stagingTenantID,
        EventID:   eventID,
        EventType: "user.action",
        Payload:   []byte(`{"action": "click", "element": "cta-button"}`),
    })
    require.NoError(t, err)
    require.Equal(t, http.StatusAccepted, resp.StatusCode)

    // 2. Wait for the event to propagate through the pipeline
    // (ingestion-api → Kafka → stream-processor → PostgreSQL)
    require.Eventually(t, func() bool {
        queryResp, err := client.QueryEvents(ctx, &api.QueryEventsRequest{
            TenantID: stagingTenantID,
            Filter:   fmt.Sprintf("event_id = '%s'", eventID),
        })
        return err == nil && len(queryResp.Events) == 1
    }, 30*time.Second, 2*time.Second, "event did not appear in query results within 30s")
}
```

**E2E suite target:** < 8 minutes. The suite covers the 10 most critical user flows. It is NOT exhaustive — it is a confidence check, not a replacement for unit and integration tests.

---

## 5. CD Pipeline: Production Deployment

Production deployments are **never automatic**. The CD pipeline opens a PR against `helm-charts/production/<service>/values.yaml` with the new image tag. An engineer must review and approve this PR to trigger the production sync.

### The Production ArgoCD PR

The auto-generated PR looks like this:

```
Title: chore(deploy): ingestion-api → a3f8c21 [staging: ✅]

Changes:
  helm-charts/production/ingestion-api/values.yaml

  - image:
  -   tag: "b2d7f14"
  + image:
  +   tag: "a3f8c21"

---
Commit: a3f8c21
Author: Priya Nair <priya.nair@aetherflow.com>
PR: https://github.com/aetherflow/aetherflow/pull/2847
Staging E2E: ✅ Passed (2m 43s)
Deploy diff: https://argocd.aetherflow.internal/app/ingestion-api-prod/diff
```

**Before approving**, the reviewer must:
1. Click the "Deploy diff" link and review what ArgoCD will change
2. Verify the staging E2E badge is ✅
3. Confirm the linked commit PR was merged via the standard code review process
4. Check that the deployment is within the [allowed deployment window](runbook-production-deployment.md#2-deployment-windows)

### Triggering the Production Sync

After approving and merging the ArgoCD PR:

```bash
# Option A: Let ArgoCD auto-sync (default — syncs within ~2 minutes of merge)

# Option B: Manually trigger sync immediately via CLI
argocd app sync ingestion-api-prod \
  --server argocd.aetherflow.internal \
  --auth-token $ARGOCD_TOKEN

# Watch the sync
argocd app wait ingestion-api-prod \
  --server argocd.aetherflow.internal \
  --health --timeout 300
```

---

## 6. ArgoCD & GitOps

AetherFlow uses **ArgoCD** as the GitOps engine for all Kubernetes deployments. The golden rule: **the `helm-charts` repository is the source of truth for what is running in each environment.** If a resource in Kubernetes differs from what's in `helm-charts`, ArgoCD will detect it as drift and either auto-sync (staging) or alert (production).

### ArgoCD Application Structure

```
ArgoCD Applications:
├── ingestion-api-staging    → helm-charts/staging/ingestion-api/
├── ingestion-api-prod       → helm-charts/production/ingestion-api/
├── auth-service-staging     → helm-charts/staging/auth-service/
├── auth-service-prod        → helm-charts/production/auth-service/
├── ...
└── infrastructure-prod      → helm-charts/production/infrastructure/
    (Kafka operator, cert-manager, ingress-nginx, etc.)
```

### Sync Policies

| Environment | Auto-Sync | Prune | Self-Heal |
|-------------|-----------|-------|-----------|
| Staging | ✅ Yes | ✅ Yes | ✅ Yes |
| Production | ❌ No | ✅ Yes (after manual sync) | ❌ No |

**Self-heal** means ArgoCD will revert manual `kubectl apply` or `kubectl edit` changes made outside of Git. In production, self-heal is disabled — a manually applied emergency patch will survive until the next sync. In staging, it is enabled, which means *any* manual change to staging Kubernetes resources will be reverted within 3 minutes.

### ArgoCD Access

- UI: `https://argocd.aetherflow.internal` (SSO via Google Workspace)
- All engineers have read access; write access (sync, rollback) requires the `argocd-deployer` role
- SRE engineers have admin access

---

## 7. Pipeline for Special Cases

### Hotfix Deployment

A hotfix that needs to bypass staging E2E (e.g., a one-line critical fix with an already-green CI) can be deployed directly by:

1. Getting explicit written approval from the Engineering Manager on-call in `#deployments`
2. Manually triggering the ArgoCD production sync with the hotfix image tag
3. Following the monitoring procedure in `runbook-production-deployment.md` §6

Document the bypass in the PR and file a follow-up ticket to ensure the fix goes through E2E in the next regular deploy.

### Infrastructure-Only Changes (Terraform)

Terraform changes follow a separate pipeline — see `reference-infra-terraform.md` §3. They do not go through the application CI/CD pipeline.

### Proto / Schema Changes

Changes to `.proto` files in the `aetherflow/proto` repository trigger a separate pipeline:

```
Proto PR merged
    │
    ▼
buf generate runs (generates Go + Python stubs)
    │
    ▼
Generated stubs committed to proto/gen/ branch
    │
    ▼
Dependent service repos pick up new stubs via go.work sync
    │
    ▼
Service CIs run against new stubs — build failures surface broken callers
```

### Database Migrations

Migration files in `db/migrations/` are detected by the CI pipeline. When present, an additional `migration-dry-run` job executes automatically:

```yaml
migration-dry-run:
  name: Migration Dry Run
  needs: [lint]
  if: contains(github.event.pull_request.changed_files, 'db/migrations/')
  runs-on: ubuntu-latest
  steps:
    - name: Dry-run migration against prod schema snapshot
      run: |
        migrate -path ./db/migrations \
          -database "$SNAPSHOT_DATABASE_URL" \
          -verbose up --dry-run
      env:
        SNAPSHOT_DATABASE_URL: ${{ secrets.SNAPSHOT_DATABASE_URL }}
```

The snapshot is a daily copy of the production schema (no data) updated by a nightly job. If the dry-run fails, the PR cannot be merged.

---

## 8. Debugging Failed Pipelines

### Common CI Failures and Fixes

**`golangci-lint: typecheck failed`**
```bash
# Usually a compilation error or missing import
# Run locally first:
golangci-lint run ./...
go build ./...
```

**`go test: race condition detected`**
```bash
# Reproduce locally with the race detector:
go test ./... -race -run TestNameOfFailingTest -count=5
# Fix: add proper mutex/channel synchronization or mark test with t.Parallel() correctly
```

**`buf breaking: field deleted`**
```bash
# You removed or renamed a Protobuf field — breaking change
# Fix: add a new field with a new field number; deprecate the old one
# Never reuse field numbers
```

**`Snyk: HIGH severity vulnerability`**
```bash
# Find which dependency introduced it:
snyk test --all-projects --severity-threshold=high

# Options:
# 1. Upgrade the dependency to a patched version
# 2. If no patch available: apply a Snyk ignore with a JIRA ticket reference
#    (requires security team approval for CRITICAL severity)
```

**`integration-tests: connection refused` (Postgres service)**
```bash
# The service container health check failed — check the GitHub Actions log for
# "health_check" step. Usually means the postgres:15.5 image took > 30s to start.
# Workaround: re-run the job. If persistent, file a ticket with Platform.
```

### Viewing Pipeline Logs

```bash
# Install GitHub CLI if not already available
brew install gh
gh auth login

# View the most recent run for your branch
gh run list --branch $(git branch --show-current) --limit 5

# View logs for a specific run
gh run view <RUN_ID> --log

# Re-run a failed job
gh run rerun <RUN_ID> --failed
```

### Pipeline Runbook Slack Channel

For persistent pipeline failures that block multiple engineers, post in `#eng-ci`. The Platform team monitors this channel and has a 4-hour SLA for pipeline infrastructure issues during business hours.

---

## 9. Pipeline Configuration Reference

### Key Files

| File | Purpose |
|------|---------|
| `.github/workflows/ci.yml` | PR and main branch CI |
| `.github/workflows/deploy-staging.yml` | Staging CD after merge |
| `.github/workflows/deploy-production.yml` | Production ArgoCD PR opener |
| `.github/workflows/drift-detection.yml` | Terraform drift detection (cron) |
| `.golangci.yml` | golangci-lint rule configuration |
| `buf.yaml` | Buf linting rules |
| `.snyk` | Snyk vulnerability ignore list |
| `Dockerfile` | Multi-stage Docker build |
| `.go-version` | Pinned Go version (read by `setup-go@v5`) |

### Environment Secrets (GitHub Actions)

These secrets are configured at the organization level and injected into all repo pipelines:

| Secret | Used For |
|--------|---------|
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | Keyless GCP auth for pushing Docker images |
| `GCP_SERVICE_ACCOUNT` | GCP service account for image push |
| `SNYK_TOKEN` | Snyk vulnerability scanning |
| `SNAPSHOT_DATABASE_URL` | Migration dry-run against prod schema snapshot |
| `ARGOCD_TOKEN` | ArgoCD sync trigger |
| `SLACK_WEBHOOK_URL` | Pipeline failure notifications to `#deployments` |

All secrets are managed by the Platform team. If a secret needs rotation, file a ticket in the `PLAT` Jira project.

---

## 10. Tribal Knowledge & Gotchas

- **The staging E2E suite is only as good as its last update.** We have had cases where a critical code path was changed but the corresponding E2E test was not updated, allowing a regression to reach production. Treat the E2E suite like production code — if you change a user-facing flow, check `tests/e2e/` and update the relevant test.

- **ArgoCD's "Sync" button bypasses the PR review step.** Any engineer with the `argocd-deployer` role can click Sync in the ArgoCD UI and push any Git state to production immediately, without the ArgoCD PR review. This is intentional for emergency situations but should not be used routinely. Every manual sync should be logged in `#deployments`.

- **GitHub Actions caches can cause phantom test passes.** The Go module cache is persisted across runs using `actions/cache`. On very rare occasions a corrupt cache causes a build to succeed with stale dependencies. If a test passes in CI but fails locally on a clean machine, try running CI with `[no-cache]` in the commit message to bust the cache.

- **The `latest` Docker image tag is a trap.** We tag images with both the Git SHA and `latest`. Never reference `latest` in a Helm values file — it is non-deterministic and defeats the purpose of GitOps. The `latest` tag exists only for external tooling that requires it.

- **Staging auto-sync has a 3-minute polling interval.** After you merge to main, ArgoCD won't pick up the staging deployment for up to 3 minutes. If you're waiting for staging to show your change, check the ArgoCD UI rather than repeatedly reloading the staging app.

- **The production pipeline deliberately adds friction.** New engineers sometimes ask why production deployments require an extra PR review step when staging already passed. The answer: staging is 50% scale with synthetic traffic. The ArgoCD PR review is the moment where a human confirms "yes, I want this specific SHA in production right now" — not just "yes, the tests passed."

- **E2E test flakiness is tracked in `#eng-ci`.** We have two known flaky E2E tests (`TestStreamReplayLargeWindow` and `TestBillingInvoiceWebhook`) that fail roughly 1 in 20 runs due to timing dependencies. These are tracked in JIRA tickets `PLAT-3201` and `PLAT-3244`. If CI fails only on these tests, re-run the E2E suite before escalating.

---

*Document Owner: Engineering Enablement Team | Review Cycle: Quarterly | Next Review: 2024-03-05*
*See also: `onboarding-env-setup.md`, `runbook-production-deployment.md`, `policy-code-review.md`*
