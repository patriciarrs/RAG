---
title: "Code Review Standards & Expectations"
author: "Engineering Leadership"
tags: ["code-review", "policy", "pull-request", "standards", "collaboration", "quality", "codeowners"]
last_updated: "2023-10-18"
version: "2.2.0"
---

# Code Review Standards & Expectations

> **Audience:** All engineers at AetherFlow.
> **Purpose:** Define what a good pull request looks like, what reviewers are responsible for, and how we maintain code quality without creating bottlenecks.
> **Related Docs:** `runbook-production-deployment.md`, `onboarding-cicd-overview.md`, `onboarding-first-30-days.md`

This document is one of the most important in the wiki. Code review is not a gate — it is the primary mechanism by which we transfer knowledge, maintain system coherence, and catch defects before they reach customers. Treat it accordingly.

---

## Table of Contents

1. [Core Principles](#1-core-principles)
2. [Pull Request Requirements](#2-pull-request-requirements)
3. [PR Size Guidelines](#3-pr-size-guidelines)
4. [Reviewer Responsibilities](#4-reviewer-responsibilities)
5. [Author Responsibilities](#5-author-responsibilities)
6. [Review Comment Conventions](#6-review-comment-conventions)
7. [Approval & Merge Rules](#7-approval--merge-rules)
8. [CODEOWNERS & Mandatory Reviewers](#8-codeowners--mandatory-reviewers)
9. [Review SLAs](#9-review-slas)
10. [Tribal Knowledge & Gotchas](#10-tribal-knowledge--gotchas)

---

## 1. Core Principles

These principles take precedence over every specific rule in this document. When a rule conflicts with a principle, re-examine the rule.

**Principle 1: The author is responsible for the change; the reviewer is responsible for understanding it.**
Approving a PR means you have read and understood the code well enough to defend it in a postmortem. "I just clicked approve" is not acceptable.

**Principle 2: Comments are about code, never about people.**
Write "this function has a race condition" not "you introduced a race condition." The distinction matters more than it sounds over months of working together.

**Principle 3: Slow reviews are a team failure, not an individual failure.**
If PRs are sitting unreviewed for more than a day, the whole team owns that. It is everyone's responsibility to flag this in the `#eng-prs` channel.

**Principle 4: Merge small, merge often.**
A 50-line PR gets reviewed in 10 minutes and merged in an hour. A 2,000-line PR takes three days, blocks the author, breeds conflicts, and receives worse feedback because reviewers get fatigued. Prefer the former.

---

## 2. Pull Request Requirements

Every PR merged to `main` (or a release branch) MUST satisfy all of the following. CI enforces most of these automatically; the rest are human-checked.

### Required Elements

```markdown
## What does this PR do?
<!-- One paragraph. What is the problem and what is the solution? -->

## Why is this the right approach?
<!-- What alternatives were considered? Why was this chosen? -->

## How was this tested?
<!-- Unit tests? Integration tests? Manual testing steps? -->

## Checklist
- [ ] Unit tests added or updated
- [ ] Integration tests added or updated (if behaviour changes)
- [ ] CHANGELOG.md updated
- [ ] No new Snyk CRITICAL/HIGH vulnerabilities
- [ ] Relevant documentation updated (wiki, OpenAPI spec, README)
- [ ] If this changes the DB schema: migration is backward-compatible
- [ ] If this touches ingestion or query hot paths: benchmarks run
```

The PR template is pre-populated automatically when you open a PR in GitHub. **Do not delete the template sections** — fill them in.

### PR Title Format

We follow **Conventional Commits** for PR titles. GitHub Actions enforces this on the PR title (which becomes the squash-merge commit message).

```
<type>(<scope>): <short summary>

Types: feat | fix | chore | refactor | perf | test | docs | ci | build
Scope: ingestion | auth | billing | query | stream-processor | platform | all

Examples:
feat(ingestion): add idempotency key support for event deduplication
fix(auth): correct token expiry comparison to use UTC consistently
perf(query): replace linear scan with indexed lookup in tenant resolution
chore(deps): bump sarama to v1.42.0
docs(wiki): update onboarding-env-setup.md with grpcurl instructions
```

---

## 3. PR Size Guidelines

| Size | Lines Changed | Target Review Time | Notes |
|------|--------------|-------------------|-------|
| **XS** | < 20 | < 15 minutes | Typo fix, config tweak — can be merged with 1 approval |
| **S** | 20–150 | 30–60 minutes | Ideal size for most feature work |
| **M** | 150–500 | 1–3 hours | Acceptable; describe context thoroughly in the PR body |
| **L** | 500–1,000 | 3–6 hours | Requires justification; consider splitting |
| **XL** | > 1,000 | > 1 day | **Requires prior team agreement.** Almost always splittable. |

### How to Split a Large PR

If you find yourself writing an XL PR, common split strategies:

1. **Infrastructure first, logic second:** Merge the new types, interfaces, and data structures in one PR; then the implementation; then the callers.
2. **Feature-flagged:** Merge the full implementation behind a `false`-defaulted feature flag, then merge the flag enablement separately.
3. **Refactor then change:** If you're refactoring and adding features simultaneously, split them. Refactors with no behavior change are trivially reviewable.
4. **Data migration separate from code change:** Never bundle a schema migration with a large application change.

> If you genuinely cannot split a PR (e.g., a large atomic refactor), post in `#eng-prs` explaining why and ask for a dedicated review session rather than async review.

---

## 4. Reviewer Responsibilities

### What You Are Reviewing For

Reviewers are expected to evaluate PRs across all of the following dimensions. Not every dimension applies to every PR, but you should consciously consider each one.

**Correctness**
- Does the code do what the PR description says?
- Are there edge cases the author hasn't handled? (empty inputs, nil pointers, concurrent access, large inputs)
- Are error paths handled correctly and propagated to callers?

**Safety & Security**
- Does the change introduce SQL injection vectors, path traversal, unvalidated input handling?
- Are secrets handled correctly? (Not logged, not returned in API responses, not hardcoded)
- Does the change alter authorization logic? If so, has the security team been looped in?

**Performance**
- Are there N+1 query patterns?
- Is anything happening synchronously that should be async?
- Are new hot-path allocations avoidable?

**Observability**
- Are new code paths covered by structured log statements at appropriate levels?
- Are new business-critical operations instrumented with metrics?
- Are new errors distinguishable in logs and traces?

**Maintainability**
- Is the code readable by someone unfamiliar with this area in 6 months?
- Are complex sections commented with *why*, not just *what*?
- Does the naming (variables, functions, packages) convey intent?

**Test Quality**
- Do the tests actually test the behavior, or do they test the implementation?
- Are test cases comprehensive? (Happy path + at least two failure modes)
- Do the tests run deterministically? (No time-dependent logic, no random seeds without fixing)

### What You Are NOT Reviewing For

- **Style preferences without a lint rule.** If you prefer `err != nil` blocks written a certain way but `golangci-lint` doesn't enforce it, do not block a PR over it. File a lint rule proposal instead.
- **Perfection.** "This could be written more cleverly" is not a blocker. "This is incorrect" is.

---

## 5. Author Responsibilities

### Before Opening the PR

- Run the full test suite locally: `go test ./... -race`
- Run the linter: `golangci-lint run ./...`
- Review your own diff in GitHub before requesting review — you will always catch something
- Ensure the PR description fully answers "why" — reviewers don't have the context you have

### During Review

- Respond to all review comments within **1 business day**
- For comments you disagree with: engage, explain, and if still deadlocked, escalate to a third engineer — do not silently ignore
- Mark addressed comments as **Resolved** so reviewers know what's been acted on
- If you make significant changes based on review feedback, leave a top-level comment summarizing what changed so reviewers don't have to re-read the entire diff

### After Merge

- Delete the feature branch immediately after merge
- Monitor the deployment for at least 30 minutes if the change is on a hot path (see `runbook-production-deployment.md`)

---

## 6. Review Comment Conventions

To reduce ambiguity, all review comments should be prefixed with a label indicating the expected action. This convention was adopted from the [Conventional Comments](https://conventionalcomments.org) standard.

| Prefix | Meaning | Blocks merge? |
|--------|---------|---------------|
| `nit:` | Minor style/preference; author can take or leave it | ❌ No |
| `question:` | Genuine curiosity; not necessarily a change request | ❌ No |
| `suggestion:` | A potentially better approach; author decides | ❌ No |
| `issue:` | A correctness or safety problem that must be fixed | ✅ Yes |
| `blocking:` | Explicit blocker — equivalent to "Request Changes" | ✅ Yes |
| `praise:` | Positive feedback — use it; it matters | ❌ No |
| `thought:` | An idea for future work, not this PR | ❌ No |

### Examples

```
nit: Variable name `x` is a bit terse — `tenantID` would be clearer.

issue: This creates a goroutine without a corresponding WaitGroup or context
cancellation path. If the parent handler returns before this goroutine
finishes, we'll have a goroutine leak. See the pattern in pkg/worker/pool.go.

question: Why do we validate the schema here rather than at the ingestion
boundary? I might be missing context — just want to understand the intent.

praise: Really clean separation of concerns here. The interface boundary makes
this very testable.
```

---

## 7. Approval & Merge Rules

| PR Size / Type | Required Approvals | Additional Rules |
|---------------|-------------------|-----------------|
| XS (< 20 lines, non-critical path) | 1 | —  |
| S, M (standard feature/fix) | 2 | At least 1 approver must be from the owning team |
| L, XL | 2 + team lead sign-off | Synchronous review session recommended |
| Security-sensitive change | 2 + `@security-review` team approval | Security changes tagged in GitHub labels |
| DB schema migration | 2 + Platform Team approval | See `runbook-production-deployment.md` §7 |
| Changes to `policy-*.md` or `adr-*.md` wiki files | 2 + Engineering Manager approval | — |

### Merge Method

**All PRs are merged via squash-merge.** We do not use merge commits or rebase-merge. This keeps the `main` branch history clean and ensures every commit on `main` corresponds to a deployable unit.

The squash-merge commit message is automatically generated from the PR title (hence the Conventional Commits requirement in Section 2).

### Stale PRs

PRs with no activity (no comments, no commits) for **5 business days** are automatically labeled `stale` by our GitHub bot. PRs stale for **10 business days** are automatically closed. If a PR is closed due to staleness, you can re-open it — the history is not lost.

---

## 8. CODEOWNERS & Mandatory Reviewers

The `CODEOWNERS` file in the root of each repository defines which teams must approve changes to which paths. GitHub enforces these as required reviewers.

Key ownership areas in the main `aetherflow` repo:

```
# CODEOWNERS excerpt — aetherflow/aetherflow
/cmd/ingestion-api/         @aetherflow/backend-ingestion
/cmd/auth-service/          @aetherflow/backend-platform
/cmd/billing-service/       @aetherflow/backend-billing
/internal/pkg/messaging/    @aetherflow/platform          # Kafka client code — Platform Team must approve
/db/migrations/             @aetherflow/platform          # All DB migrations — Platform Team must approve
/helm-charts/               @aetherflow/sre               # Infra changes — SRE must approve
/.github/                   @aetherflow/eng-leads         # CI config — Engineering Leadership must approve
/docs/policy-*.md           @aetherflow/eng-leads
/docs/adr-*.md              @aetherflow/eng-leads
```

> ⚠️ **Do not attempt to bypass CODEOWNERS** by squash-force-pushing to `main`. Branch protection rules on `main` prevent direct pushes entirely. If a CODEOWNERS review is blocking you unreasonably, escalate to an engineering manager — do not look for workarounds.

---

## 9. Review SLAs

Slow reviews are a tax on everyone's productivity. These SLAs are not aspirational — they are expected.

| Situation | SLA |
|-----------|-----|
| Initial review of a new PR | Within **1 business day** of review request |
| Re-review after author addresses comments | Within **1 business day** |
| Review of a SEV-1 hotfix PR | Within **30 minutes** (ping the reviewer directly in Slack) |
| XL PR requiring a review session | Schedule within **2 business days** |

### When You're Overloaded

If you have more review requests than you can handle within the SLA:

1. Post in `#eng-prs`: "I'm at capacity today, can someone cover @[author]'s PR?"
2. Reassign the review to another team member with context
3. Do NOT silently let the SLA expire

### Requesting Expedited Review

If your PR is blocking critical work:

1. Post in `#eng-prs` with the PR link and a one-sentence reason: "This unblocks the Kafka migration — would appreciate a same-day review."
2. Do not mass-tag — ping 1–2 specific engineers who have the relevant context.

---

## 10. Tribal Knowledge & Gotchas

- **The `#eng-prs` channel is the review coordination hub.** When you open a PR, paste the link in `#eng-prs` with a one-line description. When you approve and merge, react with ✅. This gives the whole team visibility into what's moving and what's stuck.

- **"Approved with comments" is not a thing in GitHub.** If you leave comments and click Approve, the author may merge without seeing your comments. If your comments matter, use **Request Changes** until they're addressed, then switch to Approve.

- **The `auto-merge` label is available but should be used sparingly.** Auto-merge merges a PR the moment required approvals are satisfied and CI passes. It's appropriate for dependency bumps and documentation fixes. For feature code, prefer a human moment of deliberate merge.

- **Dependabot PRs need a human eye.** Our Dependabot configuration auto-opens PRs for dependency updates. These are NOT auto-merged. They still require a 1-approval review. Before approving, check the release notes for the dependency — we've been caught twice by minor version bumps with breaking behavior changes.

- **Review comments on old code you're touching.** If you're modifying a function and you notice adjacent code with a clear bug or smell, you are encouraged (but not required) to fix it in your PR or open a follow-up ticket. Don't add a review comment telling the author to fix something they didn't write — that's demotivating. Fix it yourself or file a ticket.

- **Rebasing during active review is disorienting for reviewers.** If you rebase onto `main` while a review is in progress, the reviewer's inline comments may become orphaned. Prefer merging `main` into your branch (a merge commit) during active review, and rebase only before final merge if the branch history matters.

---

*Document Owner: Engineering Leadership | Review Cycle: Bi-annual*
*Next Review: 2024-04-18 | Feedback: Open a PR against this file or post in `#eng-culture`*
*See also: `runbook-production-deployment.md`, `onboarding-first-30-days.md`, `onboarding-cicd-overview.md`*
