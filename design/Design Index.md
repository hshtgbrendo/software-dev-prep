---
tags: [design, index, moc]
aliases: [Design MOC, Software Design Study Guide, System Design Study Guide]
created: 2026-10-16
---
# 🏗️ Software Design Study Guide: Index

> [!abstract] What this is
> A complete refresher on **software design, architecture, design patterns, and system design** for entry- to mid-level software engineering roles, written with FAANG-style interviews **and** day-to-day enterprise work in mind. Code examples are in JavaScript/TypeScript.
>
> Every note has: a summary callout → core content with examples and diagrams → tradeoff tables → a **self-test checklist** at the bottom.

---

## 🎯 What interviews actually test (by level)

| Level | Design-related rounds | Focus your prep on |
|---|---|---|
| **Entry / new grad** (Google L3, Meta E3, Amazon SDE I, Microsoft 59–60) | Usually **no** large-scale system design. **Amazon SDE I includes an OOD/low-level design round.** Design also comes up in project deep dives. | [[OOP Fundamentals]] · [[SOLID Principles]] · design patterns · [[Low-Level Design Interview]] · basics of system design |
| **Mid-level** (Google L4, Meta E4, Amazon SDE II, Microsoft 61–62) | **System design is now standard in most loops** (reported shift during 2024–2025). Bar: sensible components, ~2 levels of depth, clear tradeoffs, good communication. | Everything in Part 3 + case studies. Don't over-scope. |
| **On the job (any level)** | Design docs, code review, on-call, deployments | Parts 2, 3, 5 |

> [!tip] Suggested order if you're starting from zero
> **Week 1:** Part 1 (OOP → SOLID → principles → patterns → LLD)
> **Week 2:** Part 2 (architecture, APIs) + start Part 3 (framework, estimation, core concepts, networking)
> **Week 3:** Rest of Part 3 (databases, caching, messaging, distributed systems, reliability, security)
> **Week 4:** Part 4 case studies (practice out loud, 45-min timer) + Part 5
> Review the [[Glossary & Cheat Sheet]] before every interview.

---

## Part 1: Software Design (code level)
| # | Note | Covers |
|---|---|---|
| 1.1 | [[OOP Fundamentals]] | 4 pillars, composition vs inheritance, class relationships/UML, coupling & cohesion, DI |
| 1.2 | [[SOLID Principles]] | SRP, OCP, LSP, ISP, DIP with violation → fix examples |
| 1.3 | [[Design Principles]] | DRY, KISS, YAGNI, Law of Demeter, clean code, code smells, tech debt |
| 1.4 | [[Creational Patterns]] | Singleton, Factory, Abstract Factory, Builder, Prototype |
| 1.5 | [[Structural Patterns]] | Adapter, Decorator, Facade, Proxy, Composite, Bridge, Flyweight |
| 1.6 | [[Behavioral Patterns]] | Strategy, Observer, Command, State, Template, Iterator, Chain of Resp., etc. + pattern picker table |
| 1.7 | [[Low-Level Design Interview]] | LLD framework, **worked parking lot example**, practice list, checklist |

## Part 2: Architecture
| # | Note | Covers |
|---|---|---|
| 2.1 | [[Architectural Styles]] | Layered, MVC/MVVM, monolith vs microservices, EDA, CQRS, hexagonal/clean, serverless |
| 2.2 | [[API Design]] | REST (methods, status codes, pagination, idempotency, versioning), GraphQL, gRPC, WebSockets/SSE/webhooks |
| 2.3 | [[Domain-Driven Design]] | Bounded contexts, entities vs value objects, aggregates, domain events |

## Part 3: System Design (distributed systems)
| # | Note | Covers |
|---|---|---|
| 3.1 | [[System Design Interview Framework]] | Who gets the round, 4-step framework, building blocks, practice list, resources |
| 3.2 | [[Back-of-the-Envelope Estimation]] | Latency numbers, nines, capacity math, worked examples |
| 3.3 | [[Scalability & Core Concepts]] | Vertical/horizontal, latency/throughput, CAP, PACELC, consistency models |
| 3.4 | [[Networking Essentials]] | TCP/UDP, HTTP/1.1–3, DNS, "type a URL", load balancers, proxies, CDN |
| 3.5 | [[Databases & Storage]] | SQL vs NoSQL, indexes, isolation levels, locking, replication, sharding, object storage |
| 3.6 | [[Caching]] | Cache layers, cache-aside/write-through/write-behind, eviction, stampede, Redis |
| 3.7 | [[Messaging & Event Streaming]] | Queues vs pub/sub vs Kafka, delivery semantics, DLQ, outbox |
| 3.8 | [[Distributed Systems Patterns]] | Idempotency, consistent hashing, IDs, 2PC vs saga, Raft, locks, clocks, Bloom filters |
| 3.9 | [[Reliability & Observability]] | Timeouts/retries/circuit breakers, SLOs & error budgets, logs/metrics/traces, incidents |
| 3.10 | [[Security Fundamentals]] | AuthN/AuthZ, sessions vs JWT, OAuth/OIDC, passwords, encryption, OWASP Top 10 |

## Part 4: Case Studies (practice these out loud)
| Problem | Key concepts |
|---|---|
| [[Design a URL Shortener]] | Key generation, base62, read-heavy caching, 301 vs 302 |
| [[Design a Rate Limiter]] | Token bucket & other algorithms, Redis + Lua, distributed counters |
| [[Design a News Feed]] | Fan-out on write vs read, celebrity problem, feed cache, ranking |
| [[Design a Chat System]] | WebSockets, session registry, message ordering, presence, offline delivery |

> [!todo] Next case studies to add
> Notification system · Typeahead/autocomplete · YouTube/Netflix · Dropbox/Google Drive · Uber/Yelp (geo) · Ticketmaster · Web crawler · Distributed key-value store · Payment system

## Part 5: Engineering Practices (enterprise reality)
| Note | Covers |
|---|---|
| [[Design Docs & Code Review]] | Design doc template, Amazon 6-pager/PR-FAQ, ADRs, code review checklist, agile |
| [[Testing Strategy]] | Test pyramid, test doubles, TDD, contract tests, coverage |
| [[CI-CD & Deployment]] | Git workflows, CI/CD, canary/blue-green, feature flags, zero-downtime migrations, Docker, Kubernetes, IaC, cloud services map |

## Reference
- [[Glossary & Cheat Sheet]]: A–Z terms, numbers to memorize, pattern one-liners

---

## 📚 Best external resources
- **Books:** *System Design Interview* Vol 1 & 2 (Alex Xu) · *Designing Data-Intensive Applications* (Kleppmann) · *Head First Design Patterns* · *Clean Code* / *Clean Architecture* (Martin) · *Refactoring* (Fowler) · *The Pragmatic Programmer* · Google *SRE Book* (free online)
- **Sites:** Hello Interview · ByteByteGo · System Design Primer (GitHub) · refactoring.guru (design patterns with diagrams) · Martin Fowler's blog
- **Engineering blogs:** Netflix, Uber, Meta, Discord, Stripe, Cloudflare, Airbnb, Slack

## ✅ Progress tracker
- [ ] Part 1: Software design
- [ ] Part 2: Architecture
- [ ] Part 3: System design
- [ ] Part 4: 4 case studies practiced out loud
- [ ] Part 5: Engineering practices
- [ ] Mock interview: 1 LLD + 1 system design
