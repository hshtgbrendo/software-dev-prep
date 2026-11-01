---
tags: [design, engineering-practices]
aliases: [Design Doc, RFC, Code Review, ADR, Agile]
---
# Design Docs, RFCs & Code Review

> [!summary] What you'll learn
> At large tech companies, **design happens in writing before code**, and **every change is reviewed by another engineer**. This note explains why, what a good design document contains (with a template), company-specific formats (Google design docs, Amazon 6-pagers and PR-FAQs, RFCs), Architecture Decision Records, diagramming, how to **write** and **review** pull requests well, and the surrounding team practices (agile, tickets, monorepos).
>
> **Why it matters for your career:** writing clear design docs and giving useful reviews are among the most visible signals of growth from junior to mid-level and senior. They're also what you'll describe in behavioral interviews ("tell me about a technical decision you drove").

Back to [[Design Index]]

---

## Why write design docs at all?

Code is expensive to change once built, and a design flaw found in review costs minutes to fix, versus weeks after implementation. A design doc:
1. **Forces clear thinking**: writing exposes gaps ("wait, what happens when the payment succeeds but the order write fails?").
2. **Gets feedback early** from people with context you lack (security, the database team, the team that owns a dependency).
3. **Builds consensus** across teams before work starts, so nobody is surprised.
4. **Records decisions and their reasons** for future engineers: "why on earth did they use DynamoDB here?" has an answer.
5. **Scales communication**: one doc can be read by 20 people asynchronously, across time zones.

### When to write one
- A new service, system, or significant feature.
- Changes that cross team boundaries or change a public API or data model.
- New infrastructure dependencies (a new database, queue, or vendor).
- Anything **hard to reverse** (data migrations, storage format choices).
- Rule of thumb: more than ~2 weeks of work, or affects other teams → write a doc. A one-day bug fix → a good PR description is enough.

---

## Anatomy of a good design doc

```markdown
# <Title>
Author(s) · Reviewers · Status: Draft / In Review / Approved / Superseded · Last updated

## 1. Context and problem
What's the problem, who has it, and why solve it now? Link to product requirements and data.
Someone outside your team should understand this section.

## 2. Goals and non-goals
Goals: measurable outcomes ("p99 checkout latency < 300 ms", "support 10× current volume").
Non-goals: explicitly out of scope, which prevents scope creep and endless debate.

## 3. Requirements
Functional and non-functional: scale, latency, availability, consistency,
security, privacy, compliance, cost.

## 4. Proposed design
- Architecture diagram (components and data flow)
- APIs (endpoints or RPCs, request/response shapes)
- Data model (tables, keys, indexes) and storage choices with reasons
- Key flows (sequence diagrams for the main use cases and failure cases)
- How it meets each requirement

## 5. Alternatives considered
2–3 realistic options, each with pros and cons and why it was rejected.
(Reviewers often care about this section most.)

## 6. Tradeoffs and risks
What we're giving up. Failure modes and mitigations. Open risks.

## 7. Cross-cutting concerns
- Security and privacy (authN/Z, data classification, threat model)
- Observability (metrics, dashboards, alerts, SLOs)
- Testing strategy
- Rollout plan (feature flags, phases, canary) and rollback plan
- Data migration / backfill plan
- Cost estimate
- Operational ownership (on-call, runbooks)

## 8. Open questions
## 9. Milestones and timeline
## Appendix: references, benchmarks, prototypes
```

### Tips for writing one well
- **Lead with the problem**, not the solution. Reviewers who don't agree on the problem will never agree on the solution.
- **Be concrete**: numbers, example payloads, diagrams. "It should be fast" isn't a requirement.
- **Show alternatives honestly**, including "do nothing" and the simplest option. A doc presenting only one option looks like a decision already made.
- **Keep it as short as the problem allows.** Put details in appendices.
- **Share early as a draft** with one or two trusted reviewers before a wide review.
- **Resolve every comment** (agree, change the doc, or explain why not), and record decisions in the doc.
- **Update the status** when the design changes during implementation, or mark it superseded.

> [!tip] The habit that makes doc reviews go well
> Always include **Alternatives Considered**. It shows you explored the design space, and it's exactly the **tradeoff reasoning** that system design interviews grade ([[System Design Interview Framework]]).

---

## Company-specific formats

### Google-style design docs
Informal but thorough documents reviewed by the team and relevant experts, often followed by a design review meeting. Strong emphasis on context, goals and non-goals, alternatives, and cross-cutting concerns.

### Amazon: 6-pagers and PR-FAQs
- **Six-page narrative memos**: written in **full prose** (no bullet-point slide decks). Meetings start with everyone **silently reading** the document for ~20–30 minutes, then discussing. The idea is that prose forces complete thinking and everyone discusses the same information.
- **PR-FAQ ("Working Backwards")**: before building a product, write the **press release** announcing it (from the customer's point of view: what problem it solves and why they'd care) and an **FAQ** answering hard customer and internal questions. If the press release isn't compelling, rethink the product.
- Decisions are framed with the **Leadership Principles** (Customer Obsession, Ownership, Dive Deep, Bias for Action, Invent and Simplify...). The same principles are the rubric for Amazon's behavioral interviews.

### RFCs (Requests for Comments)
Many companies (Uber, Stripe, HashiCorp, and much of open source) use an RFC process: a proposal is posted publicly within the company with a comment period, often in a dedicated repo or doc system, then accepted or rejected.

---

## Architecture Decision Records (ADRs)

**The problem:** months later, nobody remembers *why* a choice was made, so people either cargo-cult it or reverse it without knowing the original reasons.

**The solution:** a short, numbered document per significant decision, stored **in the repository** next to the code:
```markdown
# ADR 0007: Use PostgreSQL for the orders service

Status: Accepted (2026-10-12)

## Context
Orders need multi-row transactions (order + line items + inventory reservation) and ad-hoc reporting.
Expected volume: ~200 writes/sec at peak within 2 years.

## Decision
Use PostgreSQL (managed RDS) with read replicas. Revisit sharding if writes exceed ~5K/sec.

## Consequences
+ ACID transactions, familiar tooling, flexible queries
- Vertical write limit; sharding later would be significant work
- Reporting queries must go to a replica to avoid impacting checkout
```
ADRs are **immutable**: when a decision changes, write a new ADR that **supersedes** the old one, so the history is preserved.

---

## Diagrams

Good diagrams communicate a design faster than pages of text.
- **C4 model** (Simon Brown): diagrams at four zoom levels: **Context** (the system and its users/external systems) → **Containers** (apps, databases, queues) → **Components** (modules inside a container) → **Code** (classes, rarely needed). Pick the level that fits the audience.
- **Sequence diagrams**: for request flows, especially failure paths and async interactions.
- **Entity-relationship (ER) diagrams**: for data models.
- **State diagrams**: for lifecycles (order states, connection states).
- **Tools**: **Mermaid** (text-based; renders in **Obsidian**, GitHub, GitLab, and Notion, which makes it great for docs in repos), Excalidraw (hand-drawn style), draw.io/diagrams.net, Lucidchart, PlantUML.
- Label arrows (what flows: "HTTP POST /orders," "OrderPlaced event"), include a legend if using colors, and keep each diagram focused on one idea.

---

## Code review

### Why code review exists
1. **Catch bugs** and design problems before they reach users.
2. **Share knowledge**: more than one person understands each part of the code (the "bus factor"), and juniors learn from seniors (and vice versa).
3. **Keep the codebase consistent** (style, patterns, architecture).
4. **Security and compliance**: many companies require at least one approval for every production change.

### As an author: making your PRs easy to review
- **Keep PRs small**, ideally under ~200–400 changed lines. Small PRs get reviewed faster and more thoroughly; large ones get rubber-stamped. Split work into a sequence of PRs (stacked PRs), and separate **refactors** from **behavior changes**.
- **Write a clear description**:
  ```markdown
  ## What
  Adds idempotency keys to POST /payments.

  ## Why
  Mobile clients retry on timeouts; we saw 37 duplicate charges last month (link to ticket).

  ## How
  - New idempotency_keys table (migration included) with a unique key and 24h TTL cleanup job
  - Middleware stores the response on success and replays it on repeat keys
  - Returns 409 if a request with the same key is still in progress

  ## Testing
  Unit tests for the middleware; integration test with concurrent duplicate requests; tested on staging.

  ## Rollout
  Behind the `payments_idempotency` flag, enabled for 5% of traffic first.
  ```
- **Review your own diff first**: you'll catch leftover debug logs, commented-out code, and typos.
- Make sure **CI is green** and tests are included before requesting review.
- **Annotate tricky parts** with PR comments explaining non-obvious decisions.
- **Respond to every comment**: fix it, or explain your reasoning. Assume good intent; feedback on code isn't feedback on you.

### As a reviewer: what to look for (rough priority order)
1. **Correctness**: does it do what it claims? Edge cases (null/undefined, empty lists, very large inputs, unicode, time zones, concurrent requests, retries, partial failures)?
2. **Design**: does it fit the existing architecture? Is the logic in the right layer? Is it over- or under-engineered? (See [[SOLID Principles]] and [[Design Principles]].)
3. **Security**: input validation, authorization checks (IDOR!), injection, secrets, sensitive data in logs. See [[Security Fundamentals]].
4. **Tests**: are the important behaviors and edge cases tested? Would the tests catch a regression? Are they readable?
5. **Readability and maintainability**: clear names, reasonable function sizes, comments explaining *why*.
6. **Performance**: N+1 queries, missing indexes, unbounded queries or loops, missing pagination, unnecessary work in hot paths.
7. **Operability**: logging and metrics for new paths, error handling, feature flags for risky changes, backward compatibility of APIs, events, and database schemas (can this be rolled back safely?).
8. **Style**: leave formatting to automated tools (Prettier, ESLint, gofmt, Black). Don't spend human attention on things linters can enforce.

### Comment etiquette
- **Be kind and specific.** Critique the code, not the person: "This loop queries the DB per item; could we batch these into one `IN` query?" rather than "This is slow."
- **Explain why**, and suggest an alternative when you can.
- **Label the weight** of each comment so the author knows what's required:
  - `nit:` minor and optional (naming, small style points)
  - `question:` you want to understand, not necessarily change anything
  - `suggestion:` a possible improvement; the author decides
  - `blocking:` must be addressed before merging (bugs, security issues)
- **Praise good things** too ("nice test coverage on the edge cases").
- **Approve with minor nits** rather than blocking on them. **Google's guideline:** approve a change once it **definitely improves the overall health of the codebase**, even if it isn't perfect.
- **Review promptly** (within a business day). Slow reviews block teammates and encourage giant PRs.
- If a discussion goes back and forth more than twice, **talk synchronously** (a quick call), then record the outcome in the PR.

---

## Other team practices you'll encounter

### Agile, Scrum, and Kanban
- **Scrum**: work in **sprints** (usually 1–2 weeks). Ceremonies: **sprint planning** (choose work), **daily standup** (what I did, what I'll do, blockers), **sprint review/demo**, and **retrospective** (what to improve). Work items are estimated with **story points** (relative effort, not hours).
- **Kanban**: continuous flow on a board (To Do → In Progress → Review → Done) with **work-in-progress (WIP) limits**, no fixed sprints.
- **Backlog grooming/refinement**: clarifying and sizing upcoming tickets.
- Most companies run a pragmatic mix.

### Tickets
Jira, Linear, GitHub Issues. A good ticket has: context (why), a clear description of what's needed, **acceptance criteria** ("Given a user with an expired card, when they check out, then they see an 'update payment method' prompt"), and links to designs or docs.

### Repositories and builds
- **Monorepo**: all (or most) code in one giant repository with shared tooling. **Google and Meta** run enormous monorepos with custom build systems (Bazel, Buck2). Pros: atomic cross-project changes, shared code, consistent tooling. Cons: needs heavy tooling at scale.
- **Polyrepo**: one repository per service or library (common at **Amazon** and many startups). Pros: independence. Cons: cross-repo changes and dependency versioning are harder.
- **Trunk-based development** with short-lived branches and feature flags is standard at large companies. See [[CI-CD & Deployment]].

### Style guides and readability
Teams follow language style guides (Google publishes theirs; Airbnb's JavaScript guide is well known). Google even has per-language **"readability" certification**: someone with readability in a language must approve code written in it.

### Documentation
READMEs (how to run, test, deploy), **runbooks** (how to respond to each alert), API docs, onboarding guides, and ADRs. Good engineers keep docs next to the code and update them in the same PR as the code change.

### On-call and incidents
See [[Reliability & Observability#Incident management & on-call]].

---

## Quick self-test
- [ ] Why do companies write design docs before coding? List four benefits.
- [ ] What sections does a design doc need? Why is "Alternatives Considered" so important?
- [ ] What's the difference between goals and non-goals, and why write non-goals?
- [ ] What's a PR-FAQ? What's an ADR, and why are ADRs immutable?
- [ ] What makes a PR easy to review? Write a PR description for a recent change you made.
- [ ] List your code review priorities in order. What should be left to linters?
- [ ] How do you label comment severity, and when should you approve with nits instead of blocking?
- [ ] How would you split a 2,000-line PR into reviewable pieces?

Next: [[Testing Strategy]]
