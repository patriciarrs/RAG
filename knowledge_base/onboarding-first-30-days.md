---
title: "First 30 Days — Engineer Checklist"
author: "Engineering Enablement Team"
tags: ["onboarding", "new-engineer", "checklist", "first-30-days", "culture", "setup", "expectations"]
last_updated: "2024-01-30"
version: "1.1.1"
---

# First 30 Days — Engineer Checklist

> **Audience:** Engineers in their first month at AetherFlow, and their managers.
> **Purpose:** Make the first 30 days navigable, productive, and culturally grounding.
> **Related Docs:** `onboarding-env-setup.md`, `onboarding-cicd-overview.md`, `policy-code-review.md`, `policy-oncall-rotations.md`

Welcome to AetherFlow. This document is your map for the first 30 days. It is organized by week, with a mix of practical setup tasks, reading assignments, and human touchpoints. The goal is not to do everything on this list — it's to ensure you have enough context to contribute meaningfully and feel like you belong here by the end of week four.

Your manager owns the first column of the week-one table. You own everything else. If something on this list is blocked or unclear, post in `#eng-onboarding` immediately.

---

## How to Use This Document

- Work through tasks roughly in order, but don't treat it as a waterfall — some things can and should happen in parallel
- Check items off as you complete them; this document is yours
- If you find something missing, outdated, or confusing: open a PR to fix it. Your first PR can be a documentation fix

---

## Week 1 — Orientation & Access

### Day 1 (Manager owns this)

- [ ] Laptop provisioned with Jamf MDM profile enrolled
- [ ] Google Workspace account active (`yourname@aetherflow.com`)
- [ ] Slack added to `aetherflow` workspace; invited to team channels
- [ ] 1Password invited to **AetherFlow Engineering** vault
- [ ] GitHub organization invite sent (`github.com/aetherflow`)
- [ ] Jira access granted (project spaces: `PLAT`, `BACK`, `SRE`, `INFRA`)
- [ ] PagerDuty account created; added to your team's escalation policy
- [ ] Introduced to your onboarding buddy (a peer engineer, separate from your manager)
- [ ] 30-minute welcome call with your team lead

### Day 1–2 (You own this)

- [ ] Accept all tool invites (Slack, GitHub, Jira, PagerDuty, 1Password)
- [ ] Enable **critical alerts** in PagerDuty mobile app (iOS: bypass DND; Android: priority notifications)
- [ ] Set up your development environment — follow `onboarding-env-setup.md` completely
- [ ] Confirm your first commit compiles: `go build ./...` in the `aetherflow` repo
- [ ] Read `policy-code-review.md` — understand how PRs work before you open your first one

### Day 2–5

- [ ] **Codebase tour:** Read `onboarding-codebase-tour.md` and open the key service directories (`cmd/ingestion-api`, `cmd/auth-service`, `cmd/query-api`) to get familiar with the structure
- [ ] **CI/CD tour:** Read `onboarding-cicd-overview.md` and find your first merged PR in the `aetherflow` repo — trace it from GitHub Actions to ArgoCD to production
- [ ] **Architecture overview:** Read `reference-service-catalog.md` — know what every service does before you touch any of them
- [ ] **ADR reading:** Read the four current ADRs (`adr-007-messaging-kafka.md`, `adr-009-api-grpc.md`, `adr-005-database-postgres.md`, `adr-010-observability-otel.md`). Also skim the two superseded ones (`adr-001-messaging-rabbitmq.md`, `adr-003-api-rest.md`) so you understand why we made changes
- [ ] **Incident response:** Read `runbook-incident-response.md`. You won't be primary on-call for 8 weeks, but you need to know what `#war-room` means
- [ ] **Security baseline:** Read `policy-secret-management.md`. Non-negotiable before you touch any credentials
- [ ] Shadow your onboarding buddy through a normal working day (stand-up, Slack patterns, how they triage work)

### Week 1 Human Touchpoints

| Meeting | With | Purpose |
|---------|------|---------|
| Welcome 1:1 | Engineering Manager | Goals, team context, first project scoping |
| Team stand-up | Your team | Begin understanding team rhythm |
| Company all-hands (if scheduled) | Whole company | Company context |
| Lunch/coffee with onboarding buddy | Buddy | Informal culture orientation |

---

## Week 2 — First Contribution

By end of week 2, you should have **at least one PR merged** to the main `aetherflow` repository. It doesn't need to be a feature — it can be a documentation fix, a test improvement, a linting clean-up, or a small bug. The goal is to run the full PR cycle (open, review, revision, merge) once under low pressure.

### Tasks

- [ ] **Find your first task:** Work with your team lead to identify a `good-first-issue` tagged ticket in Jira (these are pre-scoped by the team for new joiners). If none exist, ask your team lead — they should create one.
- [ ] **Open your first PR:** Follow the PR template faithfully. Over-communicate in the PR description — more context is always better for your first few PRs
- [ ] **Experience code review from the reviewer side:** Review at least one PR from a teammate this week. Leave at least three comments using the `nit:` / `question:` / `suggestion:` format from `policy-code-review.md`
- [ ] **Run the E2E test suite locally:** `make smoke-test-local` against your local Docker Compose stack — make sure it passes on your machine
- [ ] **Explore Grafana:** Open `https://grafana.aetherflow.internal` and find the ingestion-api service health dashboard. Understand what the P99 latency and error rate panels are measuring
- [ ] **Attend a production deployment:** Ask your team lead if a deployment is happening this week and shadow it. Watch the ArgoCD sync, the post-deploy validation steps, and the `#deployments` channel activity
- [ ] **Read your team's recent PIRs:** Find the last 2 post-incident reviews filed by your team in Confluence. These are the most concentrated source of system knowledge you'll find

### Key Slack Channels to Join by End of Week 2

| Channel | Purpose |
|---------|---------|
| `#deployments` | All production deployments — read-only observation initially |
| `#eng-prs` | PR coordination for your team |
| `#alerts-prod` | Production alerts — high volume; skim for awareness |
| `#incidents` | Active incident coordination |
| `#eng-onboarding` | Your lifeline for questions |
| `#team-<yourteam>` | Your immediate team channel |
| `#product-all` | Company-wide product updates |
| `#reliability` | SRE and reliability engineering discussions |

---

## Week 3 — Deeper Context

By week 3 you should be operating more independently. Tasks are less prescriptive — use your judgment about what needs more depth.

### Tasks

- [ ] **Own a ticket end-to-end:** Take a Jira ticket from triage to deployed-in-production without your buddy holding your hand. Post each step in your 1:1 notes so your manager can see the shape of the work
- [ ] **Understand your team's SLOs:** Find your service's SLO dashboard in Grafana and answer: what is our error budget burn rate this month? Are we on track?
- [ ] **Shadow an on-call shift:** Spend one day watching the on-call engineer for your team. How many alerts fire? How do they triage? What's in their queue at end-of-day?
- [ ] **Kafka deep-dive (if on Ingestion or Platform):** Read `runbook-kafka-operations.md` and `adr-007-messaging-kafka.md`. Run `kafka-consumer-groups.sh` against local Kafka and understand what consumer lag looks like
- [ ] **Database fundamentals:** Connect directly to the local PostgreSQL instance (credentials in `onboarding-env-setup.md`) and run `EXPLAIN ANALYZE` on two queries from the query-api service. Understand the difference between sequential scan and index scan on the `events` table
- [ ] **Meet two engineers outside your immediate team:** AetherFlow has an `#eng-coffees` Slack app that randomly pairs engineers for 30-minute virtual coffees. Sign up if you haven't already. Alternatively, reach out directly — everyone expects and welcomes this from new joiners

### Questions to Be Able to Answer by End of Week 3

These aren't a test — use them as a self-check:

1. What happens to a customer event between the time it hits `ingest.aetherflow.com` and the time it appears in the query results? Name every service and queue it passes through.
2. If `auth-service` went down right now, what would break and what would keep working?
3. Where would you look first if a customer reported that their events from 10 minutes ago aren't showing in their dashboard?
4. What is the difference between a SEV-1 and a SEV-2, and who gets paged for each?
5. What is the current messaging system used for inter-service async communication, and what did we use before?

---

## Week 4 — Independence & On-Call Readiness

By week 4, you should be contributing at close to full velocity and preparing to join the on-call rotation (which begins after week 8, per policy).

### Tasks

- [ ] **Complete your on-call pre-qualification checklist** (below)
- [ ] **Present your work:** Most teams do a weekly engineering sync where engineers share what they shipped. Volunteer to present this week — even a small thing
- [ ] **File your first Jira ticket from scratch:** You've noticed something that needs fixing or improving. Write a well-scoped ticket: clear problem statement, reproduction steps or evidence, proposed approach, acceptance criteria
- [ ] **Infrastructure familiarity:** Browse the `infra/environments/production/` directory in the `infra` repo. Understand where the EKS cluster, RDS instances, and Kafka nodes are defined. You don't need to modify anything — just understand the shape
- [ ] **Meet with your manager for a 30-day retrospective:** This is a two-way conversation. Come with: what's going well, what's confusing, what you wish you'd known on day 1

### On-Call Pre-Qualification Checklist

You will join the on-call rotation after **8 weeks**, not before. Before going live as primary, you must have checked off every item:

```
On-Call Pre-Qualification — [YOUR NAME]
Manager sign-off required before on-call rotation begins.

Runbook Familiarity
[ ] Can explain the SEV-1 response procedure from memory (rough outline is fine)
[ ] Has read runbook-incident-response.md in full
[ ] Has read runbook-database-failover.md in full
[ ] Has read runbook-kafka-operations.md in full (if on Ingestion/Platform)
[ ] Knows how to check Kafka consumer lag and identify which consumer group is behind
[ ] Knows how to trigger a Kubernetes rollback and confirm it completed

Tooling Access
[ ] PagerDuty account active with critical alert notifications enabled
[ ] Can acknowledge a PagerDuty alert from mobile
[ ] Has kubectl access to the production namespace (read + limited write)
[ ] Can open Grafana and navigate to the service health dashboard
[ ] Has joined #war-room and understands the war-room protocol

Incident Simulation
[ ] Has shadowed at least one on-call shift for their rotation
[ ] Has participated in a game-day or incident simulation (scheduled by SRE quarterly)
  OR: manager has explicitly waived this requirement with documented rationale

Human Touchpoints
[ ] Introductory call with SRE lead about infrastructure dependencies
[ ] Confirmed handoff buddy for first on-call week (a more experienced engineer
    who agrees to be reachable for escalations during your first rotation)
```

---

## Ongoing — Culture & Career

These aren't week-specific but are important to establish early.

### How We Work

- **Default to async:** Most communication at AetherFlow happens asynchronously via Slack and Jira. Real-time meetings are for decisions, not status updates. If a message can wait 4 hours, it belongs in Slack, not a calendar invite.
- **Write things down:** Engineering decisions belong in ADRs. Operational knowledge belongs in runbooks. If you solved a problem that took you more than an hour to figure out, write down what you learned somewhere durable.
- **PRs are conversations:** Code review is the primary way we share knowledge and build shared ownership of the codebase. Review others' PRs actively — it's not a distraction from your work; it is your work.
- **Blameless incident culture:** When things break, we fix the system, not the person. Postmortems are not performance reviews. Admitting you made a mistake is not only safe — it's expected and respected.

### Engineering Values (AetherFlow Engineering Principles)

These are not slogans — they are the criteria by which we make ambiguous decisions:

1. **Reliability over velocity.** A feature that works 99% of the time is worse than no feature if the 1% failure corrupts customer data.
2. **Observable systems.** If you can't measure it, you can't improve it. Instrument everything.
3. **Small, reversible changes.** Big-bang deploys are how SEV-1s are born. Prefer feature flags and incremental rollouts.
4. **Explicit over implicit.** Configuration should be in code. Behavior should be predictable from reading the code. Surprise is a bug.
5. **Systems thinking.** Every change has downstream effects. Think two services ahead before you merge.

### 1:1 with Manager — Suggested Topics by Week

| Week | Suggested Topics |
|------|-----------------|
| 1 | Goals for the first 90 days; how your team operates; what success looks like |
| 2 | How did your first PR feel? What was harder than expected? |
| 3 | SLO and reliability context for your team's services; career growth interests |
| 4 | 30-day retrospective; on-call readiness timeline; upcoming projects |

---

## Quick Reference: Who to Ask for What

| Question | Ask |
|----------|-----|
| My local environment is broken | `#eng-onboarding` |
| I don't understand a piece of code | Your onboarding buddy first, then the service owner |
| I think I found a bug in production | Post in your team channel; file a Jira ticket |
| I have a security concern | `#security-incidents` or security@aetherflow.com |
| I disagree with a code review comment | The reviewer directly; escalate to a third engineer if needed |
| Something in this wiki is wrong | Open a PR to fix it and post in `#eng-docs` |
| I'm overwhelmed | Your manager, your onboarding buddy, or `#eng-wellbeing` |
| I need access to something | File a Jira ticket in the `IT` project |
| I don't know what to work on | Your team lead in your next 1:1 or your team Slack channel |

---

## Further Reading (No Deadline)

These are not required reading for week 1–4, but are high-value for long-term context at AetherFlow:

- **ADR-005:** `adr-005-database-postgres.md` — Why PostgreSQL, and the constraints we operate under
- **ADR-010:** `adr-010-observability-otel.md` — How we think about observability and why we moved to OpenTelemetry
- **Data Retention Policy:** `policy-data-retention.md` — Critical for understanding what we can and cannot do with customer data
- **Terraform Reference:** `reference-infra-terraform.md` — How our cloud infrastructure is defined and managed
- **[Google SRE Book](https://sre.google/sre-book/table-of-contents/)** — The conceptual foundation for how our SRE team thinks. Chapters 3 (Embracing Risk), 6 (Monitoring), and 13 (Emergency Response) are directly applicable.
- **[The Pragmatic Engineer Newsletter](https://newsletter.pragmaticengineer.com/)** — Gergely Orosz covers real-world engineering practices at companies similar to AetherFlow's scale.

---

*Document Owner: Engineering Enablement Team | Review Cycle: Quarterly | Next Review: 2024-04-30*
*Feedback: `#eng-onboarding` or open a PR — new-joiner feedback on this document is especially valuable*
*See also: `onboarding-env-setup.md`, `onboarding-cicd-overview.md`, `policy-code-review.md`*
