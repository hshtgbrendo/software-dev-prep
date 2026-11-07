---
tags: [design, engineering-practices]
aliases: [Design Doc, RFC, Code Review, ADR]
---
# Design Docs, RFCs & Code Review

> [!summary]
> At FAANG-style companies, **design happens in writing** before code: design docs (Google), RFCs (Uber and many others), 6-pagers/PR-FAQs (Amazon), plus architecture decision records. And **every change is code reviewed**. Being good at both is how junior engineers get promoted, and how you'll talk about past work in behavioral interviews.

Back to [[Design Index]]

---

## Design documents

### When to write one
- New service/system, significant feature, cross-team change, new dependency or datastore, anything hard to reverse.
- Rough rule: if it's more than ~1–2 weeks of work or affects other teams, write a doc.

### Typical structure
```markdown
# <Title>  ·  Author · Reviewers · Status (Draft/In Review/Approved) · Last updated

## Context & Problem
What problem, for whom, why now. Link to the product requirements.

## Goals / Non-Goals
Explicit scope. Non-goals prevent scope creep.

## Requirements
Functional + non-functional (scale, latency, availability, security, privacy).

## Proposed Design
Architecture diagram, components, APIs, data model, key flows (sequence diagrams).

## Alternatives Considered
2–3 options with pros/cons and why they were rejected. (Reviewers care most about this.)

## Tradeoffs & Risks
What we're giving up. Failure modes. Mitigations.

## Cross-cutting concerns
Security, privacy, observability (metrics/alerts/dashboards), testing plan,
rollout plan (flags, phases), rollback plan, migration/backfill, cost.

## Open Questions
## Timeline / Milestones
```

> [!tip] The habit that makes doc reviews go well
> Always include **Alternatives Considered**. It shows you explored the space. It's exactly the "tradeoffs" skill graded in [[System Design Interview Framework|system design interviews]].

### Amazon-specific formats
- **6-pager narrative**: prose (no slides), read silently at the start of meetings.
- **PR-FAQ**: "Working Backwards." Write the **press release** and **FAQ** for the launched product *before* building it.
- Tie decisions to **Leadership Principles** (Customer Obsession, Dive Deep, Bias for Action...). They're also the behavioral interview rubric.

### Architecture Decision Records (ADRs)
Short, numbered docs capturing **one decision**: *Context → Decision → Status → Consequences.* Stored in the repo (`docs/adr/0007-use-postgres-for-orders.md`). They explain *why* things are the way they are to future engineers.

### Diagrams
- **C4 model**: Context → Containers → Components → Code (zoom levels).
- **Sequence diagrams** for flows, **ER diagrams** for data, **state diagrams** for lifecycles.
- Tools: Mermaid (works in Obsidian and GitHub!), Excalidraw, draw.io, Lucidchart.

---

## Code review

### As an author
- **Small PRs** (< ~400 lines changed). They get faster, better reviews. Split refactors from behavior changes.
- Clear description: **what, why, how tested**, screenshots for UI, linked ticket.
- Self-review your own diff first.
- Tests included; CI green before requesting review.
- Respond to every comment; don't take feedback personally.

### As a reviewer: what to look for (in priority order)
1. **Correctness**: does it do what it claims? Edge cases (null, empty, large inputs, concurrency, time zones)?
2. **Design**: does it fit the architecture? Right abstraction level? (See [[SOLID Principles]], [[Design Principles]].)
3. **Security**: injection, authZ checks, secrets, PII in logs. See [[Security Fundamentals]].
4. **Tests**: meaningful coverage of behavior and edge cases.
5. **Readability**: names, structure, comments explaining *why*.
6. **Performance**: N+1 queries, unbounded loops/queries, missing pagination or indexes.
7. **Operability**: logging, metrics, error handling, feature flag, backward compatibility of APIs/schemas.
8. Style: let **linters/formatters** handle it (ESLint, Prettier). Don't bikeshed.

### Comment etiquette
- Be kind and specific; critique code, not people.
- Prefix intent: **nit:** (optional), **question:**, **suggestion:**, **blocking:**.
- Explain why, and offer an alternative.
- Approve with minor nits rather than blocking.
- Google's guideline: approve once the change **definitely improves overall code health**, even if it isn't perfect.

---

## Other enterprise practices you'll encounter
- **Agile/Scrum**: sprints (1–2 weeks), standups, sprint planning, retros, story points, backlog grooming. **Kanban**: continuous flow with WIP limits.
- **Tickets**: Jira/Linear; write clear acceptance criteria.
- **Monorepo vs polyrepo**: Google and Meta use giant monorepos (Bazel/Buck builds); Amazon is mostly polyrepo.
- **Trunk-based development** with feature flags vs GitFlow. See [[CI-CD & Deployment]].
- **On-call** and **postmortems**. See [[Reliability & Observability#Incident management & on-call]].
- **Readability / style guides**: Google has per-language readability certification; follow team style guides.
- **Documentation**: READMEs, runbooks, API docs, onboarding docs.

## Self-test
- [ ] What sections does a design doc need, and why is "Alternatives Considered" important?
- [ ] What's an ADR?
- [ ] List your code review priorities in order.
- [ ] How would you break a 2,000-line PR into reviewable pieces?
