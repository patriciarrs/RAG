---
title: "Local Development Environment Setup"
author: "Engineering Enablement Team"
tags: ["onboarding", "local-dev", "docker", "go", "environment", "setup", "toolchain", "new-engineer"]
last_updated: "2024-01-22"
version: "3.1.2"
---

# Local Development Environment Setup

> **Audience:** New engineers joining AetherFlow, or anyone rebuilding their dev environment from scratch.
> **Time to complete:** ~2 hours for a clean machine; ~45 minutes if you already have Go and Docker installed.
> **Related Docs:** `onboarding-cicd-overview.md`, `onboarding-first-30-days.md`, `adr-009-api-grpc.md`

If you hit a problem not covered here, post in `#eng-onboarding` on Slack. Do NOT suffer in silence — if something is broken for you, it's probably broken for the next person too, and fixing the doc is a gift to the whole team.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Repository Setup](#2-repository-setup)
3. [Core Services: Docker Compose](#3-core-services-docker-compose)
4. [Go Toolchain](#4-go-toolchain)
5. [Proto Toolchain Setup (gRPC)](#5-proto-toolchain-setup-grpc)
6. [Environment Variables](#6-environment-variables)
7. [Running Services Locally](#7-running-services-locally)
8. [Running Tests](#8-running-tests)
9. [IDE Setup](#9-ide-setup)
10. [Tribal Knowledge & Common Pitfalls](#10-tribal-knowledge--common-pitfalls)

---

## 1. Prerequisites

Install the following before anything else. Version requirements are strict — mismatched versions are the #1 source of "it works on my machine" problems.

| Tool | Required Version | Install |
|------|-----------------|---------|
| **Go** | 1.21.x | `brew install go@1.21` or [go.dev/dl](https://go.dev/dl) |
| **Docker Desktop** | ≥ 4.25 | [docker.com](https://www.docker.com/products/docker-desktop) |
| **kubectl** | ≥ 1.28 | `brew install kubectl` |
| **Helm** | ≥ 3.13 | `brew install helm` |
| **buf** | ≥ 1.28 | `brew install bufbuild/buf/buf` |
| **grpcurl** | Latest | `brew install grpcurl` |
| **golang-migrate** | ≥ 4.17 | `brew install golang-migrate` |
| **jq** | Latest | `brew install jq` |
| **direnv** | Latest | `brew install direnv` |

```bash
# Verify all versions at once
go version       # Expected: go1.21.x
docker --version # Expected: Docker version 24.x or higher
buf --version    # Expected: 1.28.x or higher
migrate --version
```

> **Linux users:** We don't have a formal Linux setup guide yet (most engineers use macOS). The Docker Compose steps and Go toolchain work identically. For `buf`, use the [official install script](https://buf.build/docs/installation). Post in `#eng-onboarding` if you need help.

---

## 2. Repository Setup

AetherFlow uses a multi-repo structure. You'll need at minimum the main application repo and the proto repo.

```bash
# Create your workspace directory
mkdir -p ~/code/aetherflow && cd ~/code/aetherflow

# Clone the main application repo
git clone git@github.com:aetherflow/aetherflow.git
cd aetherflow

# Clone the proto definitions repo (required for gRPC code generation)
cd ~/code/aetherflow
git clone git@github.com:aetherflow/proto.git

# Optional but recommended: clone the helm-charts repo for infra work
git clone git@github.com:aetherflow/helm-charts.git
```

### Git Configuration

```bash
# Ensure your commit email matches your GitHub account (required for CODEOWNERS enforcement)
git config --global user.email "your.name@aetherflow.com"
git config --global user.name "Your Name"

# Install our git hooks (lint, secret scanning, conventional commits check)
cd ~/code/aetherflow/aetherflow
make install-hooks
```

The `make install-hooks` command installs:
- **pre-commit:** runs `golangci-lint` and `gofmt` on staged `.go` files
- **commit-msg:** enforces [Conventional Commits](https://www.conventionalcommits.org/) format (`feat:`, `fix:`, `chore:`, etc.)
- **pre-push:** runs `gitleaks` to scan for accidentally committed secrets

---

## 3. Core Services: Docker Compose

The fastest way to get all external dependencies running locally is our Docker Compose stack. This brings up Kafka, PostgreSQL, Redis, the Schema Registry, and local versions of our observability stack.

```bash
cd ~/code/aetherflow/aetherflow

# Start the full local dependency stack
# First run will pull ~2GB of images — grab a coffee
docker compose -f docker/docker-compose.local.yml up -d

# Verify everything came up healthy
docker compose -f docker/docker-compose.local.yml ps
```

Expected output (all should be `healthy` or `running`):

```
NAME                        STATUS          PORTS
aetherflow-postgres         healthy         0.0.0.0:5432->5432/tcp
aetherflow-kafka-broker-1   running         0.0.0.0:9092->9092/tcp
aetherflow-kafka-broker-2   running         0.0.0.0:9093->9092/tcp
aetherflow-zookeeper        running         0.0.0.0:2181->2181/tcp
aetherflow-schema-registry  healthy         0.0.0.0:8081->8081/tcp
aetherflow-redis            running         0.0.0.0:6379->6379/tcp
aetherflow-prometheus       running         0.0.0.0:9090->9090/tcp
aetherflow-grafana          running         0.0.0.0:3000->3000/tcp
```

### Seeding the Database

```bash
# Run all pending migrations against local Postgres
make db-migrate-local

# Seed with development fixture data (fake tenants, sample events)
make db-seed-local
```

This creates a `dev` tenant (`tenant_id: dev-tenant-001`) and seeds 7 days of synthetic event data, which is useful for testing the query and analytics services.

### Useful Local Service URLs

| Service | URL | Credentials |
|---------|-----|-------------|
| Grafana | http://localhost:3000 | admin / aetherflow-dev |
| Prometheus | http://localhost:9090 | — |
| Kafka UI | http://localhost:8080 | — |
| Schema Registry | http://localhost:8081 | — |
| PostgreSQL | localhost:5432 | aetherflow / aetherflow-dev |

---

## 4. Go Toolchain

```bash
cd ~/code/aetherflow/aetherflow

# Download all Go module dependencies
go mod download

# Verify the build
go build ./...

# Run the linter (same configuration as CI)
golangci-lint run ./...

# If golangci-lint is not installed:
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh \
  | sh -s -- -b $(go env GOPATH)/bin v1.55.2
```

### Key Go Module Conventions

- We use Go workspaces (`go.work`) to manage the multi-module local setup. The `go.work` file at the root references both `aetherflow/` and `proto/gen/go/`.
- **Do not run `go get` directly** to add dependencies. Use `go get -u <module>@<version>` and then run `make vendor` to update the vendor directory. We vendor all dependencies.
- The `internal/` package structure is intentional — packages under `internal/` cannot be imported by external tools. If you need to share code between services, it goes in `pkg/`.

---

## 5. Proto Toolchain Setup (gRPC)

> This section is required if you work on any service with gRPC interfaces. If you're purely working on the frontend or data pipelines, you can skip this.
> See `adr-009-api-grpc.md` for the architectural context.

```bash
cd ~/code/aetherflow/proto

# Install buf (if not already done in Section 1)
brew install bufbuild/buf/buf

# Verify buf can see the config
buf config ls-files

# Generate Go and Python stubs from .proto files
# Output goes to proto/gen/go/ and proto/gen/python/
buf generate

# The generated Go stubs are automatically picked up by the
# Go workspace (go.work) in the main aetherflow repo.
# You should NOT commit generated files — they're regenerated in CI.
```

### Adding a New RPC Method

1. Edit the relevant `.proto` file in `proto/aetherflow/<domain>/v1/`
2. Run `buf lint` — fix any linting errors before proceeding
3. Run `buf breaking --against "https://buf.build/aetherflow/proto"` — **if this reports a breaking change, you need to create a new message version (e.g., `v2`), not modify the existing one**
4. Run `buf generate` to regenerate stubs
5. Implement the server-side handler in the service repository
6. Write a unit test using the generated mock client (see `go/pkg/testutil/grpc_mock.go`)

---

## 6. Environment Variables

We use `direnv` to manage per-directory environment variables. Each service has a `.envrc.example` file you copy and populate.

```bash
cd ~/code/aetherflow/aetherflow

# Copy the example env file
cp .envrc.example .envrc

# Allow direnv to load it
direnv allow .

# View the populated env
direnv exec . env | grep AETHERFLOW
```

### Critical Environment Variables

| Variable | Description | Local Default |
|----------|-------------|---------------|
| `AETHERFLOW_ENV` | Environment name | `development` |
| `DATABASE_URL` | PostgreSQL connection string | `postgres://aetherflow:aetherflow-dev@localhost:5432/aetherflow` |
| `KAFKA_BROKERS` | Kafka broker list | `localhost:9092,localhost:9093` |
| `SCHEMA_REGISTRY_URL` | Confluent Schema Registry | `http://localhost:8081` |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379` |
| `VAULT_ADDR` | HashiCorp Vault address | `http://localhost:8200` (local dev Vault) |
| `LOG_LEVEL` | Logging verbosity | `debug` |
| `LAUNCHDARKLY_SDK_KEY` | LaunchDarkly SDK key | Use the `dev` key from 1Password |

> ⚠️ **Never commit a populated `.envrc` file.** It is in `.gitignore`, but double-check with `git status` before committing anything. The `pre-commit` hook also runs `gitleaks` to catch accidental secret commits.

### Secrets in 1Password

All secrets for local development are stored in the **AetherFlow Engineering** vault in 1Password. Request access on your first day — your manager will send the invite. The vault is organized by service (e.g., `local-dev/ingestion-api`, `local-dev/auth-service`).

---

## 7. Running Services Locally

Each service can be run individually or you can use the top-level Makefile targets:

```bash
# Run a single service (e.g., ingestion-api)
cd ~/code/aetherflow/aetherflow
go run ./cmd/ingestion-api/main.go

# Run with hot-reload (uses Air — install with: go install github.com/cosmtrek/air@latest)
air -c cmd/ingestion-api/.air.toml

# Run all services simultaneously (uses Procfile + goreman)
# Install goreman: go install github.com/mattn/goreman@latest
goreman start

# Run a quick integration smoke test against local services
make smoke-test-local
```

### Service Port Map (Local)

| Service | gRPC Port | HTTP Port (legacy / health) |
|---------|-----------|----------------------------|
| `ingestion-api` | 50051 | 8080 |
| `auth-service` | 50052 | 8081 |
| `query-api` | 50053 | 8082 |
| `tenant-service` | 50054 | 8083 |
| `stream-processor` | — (consumer only) | 8084 (health) |
| `billing-service` | 50055 | 8085 |

---

## 8. Running Tests

```bash
# Run all unit tests
go test ./... -short

# Run with race detector (always enabled in CI — run locally before pushing)
go test ./... -race -short

# Run integration tests (requires Docker Compose stack to be running)
go test ./... -tags=integration -count=1

# Run tests for a single package with verbose output
go test -v -run TestIngestionHandler ./internal/ingestion/...

# Generate test coverage report
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out -o coverage.html
open coverage.html
```

### Test Conventions

- Unit tests live next to the code they test (`foo.go` → `foo_test.go`)
- Integration tests are in `tests/integration/` and tagged with `//go:build integration`
- Use `testify` for assertions (`github.com/stretchr/testify/assert`)
- Database tests use `testcontainers-go` to spin up a real Postgres instance — do not mock the database layer in integration tests

---

## 9. IDE Setup

### VS Code (recommended for Go)

Install the [Go extension](https://marketplace.visualstudio.com/items?itemName=golang.Go) by the Go team. The repo includes a `.vscode/settings.json` with the recommended configuration for `gopls`, format-on-save, and the golangci-lint integration.

Recommended additional extensions:
- **Buf (Protobuf):** `bufbuild.vscode-buf` — syntax highlighting and `buf lint` integration
- **vscode-proto3:** `zxh404.vscode-proto3` — Protobuf syntax highlighting
- **GitLens:** for blame and history

### GoLand / IntelliJ

Works well out of the box. Install the **Buf** plugin from the JetBrains marketplace for `.proto` file support. Set the Go SDK to 1.21 explicitly — GoLand sometimes picks up a system Go version.

---

## 10. Tribal Knowledge & Common Pitfalls

- **Docker Desktop memory:** The local stack is memory-hungry — Kafka alone wants 1.5GB JVM heap. Set Docker Desktop's memory limit to at least **8GB** in Preferences → Resources. Engineers have lost hours to mysterious Kafka startup failures that were just OOM kills.

- **The `go.work` file confusion:** If you see `go: updates to go.work needed; to update it: go work sync` errors, run `go work sync` from the repo root. This happens when `proto/gen/go/` is regenerated.

- **Kafka local vs. production topic names:** The local Docker Compose Kafka uses the same topic names as production (e.g., `aetherflow.ingestion.raw-event.v1`). If you run a consumer with `--from-beginning` against local Kafka after seeding, you'll replay all the seed data. This is usually what you want, but can be confusing when debugging consumer offset behavior.

- **`direnv allow` must be re-run after `.envrc` changes:** If you update your `.envrc` file and don't re-run `direnv allow`, the old values will still be in your shell. If your local service is behaving strangely after an env change, this is the first thing to check.

- **Port conflicts with other local services:** The `ingestion-api` health port (8080) conflicts with many other development tools (Caddy, local web servers). If `goreman start` fails immediately, run `lsof -i :8080` to find the conflicting process.

- **The `dev-tenant-001` seed tenant:** The seed data's tenant has API keys stored in `db/seeds/dev-tenant.sql`. If you wipe and re-seed the database, the API key changes. The current key is always in 1Password under `local-dev/dev-tenant`.

- **gRPC reflection is disabled in production** but enabled locally. `grpcurl -plaintext localhost:50051 list` will show all services locally but will not work in staging or production. In those environments, you need to specify the full service and method name explicitly.

---

*Document Owner: Engineering Enablement Team | Review Cycle: Monthly*
*Next Review: 2024-02-22 | Feedback: `#eng-onboarding` or open a PR against this file*
*See also: `onboarding-cicd-overview.md`, `onboarding-first-30-days.md`, `policy-secret-management.md`*
