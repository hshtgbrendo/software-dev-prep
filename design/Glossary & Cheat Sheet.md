---
tags: [design, glossary, cheat-sheet]
aliases: [Design Glossary, Design Cheat Sheet]
---
# Glossary & Cheat Sheet

> [!summary]
> Quick definitions to review before an interview. Each links to its deeper note. Use Obsidian's search (Ctrl+Shift+F) or the outline pane to jump.

Back to [[Design Index]]

---

## 🔟 The 10 things to say in almost every system design
1. Clarify **functional + non-functional** requirements first.
2. **Stateless app servers** behind a **load balancer**.
3. **Cache** hot reads (Redis, cache-aside, TTL).
4. **CDN + object storage** for static/media.
5. **Read replicas** for read-heavy; **sharding** (with a good shard key) for write-heavy/huge data.
6. **Async queues** for slow or non-critical work.
7. **Idempotency** for retries (keys, dedup).
8. **No single points of failure**: redundancy across AZs.
9. **Rate limiting**, auth, and monitoring/alerting.
10. State **tradeoffs** explicitly (consistency vs availability, latency vs cost, complexity vs need).

---

## A–Z glossary

| Term | Meaning | Note |
|---|---|---|
| **ACID** | Atomicity, Consistency, Isolation, Durability | [[Databases & Storage]] |
| **Aggregate** | DDD consistency boundary with a root entity | [[Domain-Driven Design]] |
| **API Gateway** | Single entry point: routing, auth, rate limiting | [[Networking Essentials]] |
| **Availability** | % of time the system is usable (nines) | [[Scalability & Core Concepts]] |
| **Backpressure** | Downstream signals upstream to slow down | [[Scalability & Core Concepts]] |
| **BASE** | Basically Available, Soft state, Eventually consistent | [[Scalability & Core Concepts]] |
| **Bloom filter** | Probabilistic set: "definitely not" or "maybe" | [[Distributed Systems Patterns]] |
| **Blue-green deploy** | Two environments, switch traffic at once | [[CI-CD & Deployment]] |
| **Bounded context** | Boundary within which a domain model is consistent | [[Domain-Driven Design]] |
| **Bulkhead** | Isolate resources so one failure doesn't sink all | [[Reliability & Observability]] |
| **Cache-aside** | App reads cache, on miss loads DB and fills cache | [[Caching]] |
| **Canary release** | Roll out to a small % first, watch metrics | [[CI-CD & Deployment]] |
| **CAP theorem** | During partition: choose consistency or availability | [[Scalability & Core Concepts]] |
| **CDC** | Change Data Capture: stream DB changes | [[Databases & Storage]] |
| **CDN** | Edge caches near users | [[Networking Essentials]] |
| **Circuit breaker** | Stop calling a failing dependency; fail fast | [[Reliability & Observability]] |
| **Cohesion / Coupling** | Focus within a module / dependency between modules | [[OOP Fundamentals]] |
| **Consistent hashing** | Hash ring; adding a node moves only ~K/N keys | [[Distributed Systems Patterns]] |
| **CQRS** | Separate write and read models | [[Architectural Styles]] |
| **CRDT** | Data types that merge concurrent edits automatically | [[Distributed Systems Patterns]] |
| **Denormalization** | Duplicate data for faster reads | [[Databases & Storage]] |
| **Dependency Injection** | Pass dependencies in rather than creating them | [[OOP Fundamentals]] |
| **DLQ** | Dead letter queue for repeatedly failing messages | [[Messaging & Event Streaming]] |
| **Error budget** | 100% − SLO; allowed unreliability | [[Reliability & Observability]] |
| **Event sourcing** | Store events as the source of truth | [[Architectural Styles]] |
| **Eventual consistency** | Replicas converge if writes stop | [[Scalability & Core Concepts]] |
| **Fan-out on write/read** | Push to followers' feeds vs pull at read time | [[Design a News Feed]] |
| **Feature flag** | Toggle features without deploying | [[CI-CD & Deployment]] |
| **Gossip protocol** | Peer-to-peer state dissemination | [[Distributed Systems Patterns]] |
| **Horizontal scaling** | Add more machines | [[Scalability & Core Concepts]] |
| **Hot spot / hot key** | Disproportionate load on one shard/key | [[Scalability & Core Concepts]] |
| **Idempotency** | Repeating an operation has the same effect | [[Distributed Systems Patterns]] |
| **Isolation levels** | Read committed, repeatable read, serializable... | [[Databases & Storage]] |
| **JWT** | Signed, stateless token with claims | [[Security Fundamentals]] |
| **Kafka** | Partitioned, replicated, replayable log | [[Messaging & Event Streaming]] |
| **L4 / L7 LB** | Transport-level vs HTTP-aware load balancing | [[Networking Essentials]] |
| **Leader election** | Choosing a single coordinator node | [[Distributed Systems Patterns]] |
| **Linearizability** | Strongest consistency; reads see the latest write | [[Scalability & Core Concepts]] |
| **Little's Law** | L = λW | [[Scalability & Core Concepts]] |
| **LSM tree** | Write-optimized storage (memtable + SSTables) | [[Databases & Storage]] |
| **MVCC** | Multi-version concurrency; readers don't block writers | [[Databases & Storage]] |
| **N+1 problem** | 1 query for a list + N queries for related items | [[Databases & Storage]] |
| **OAuth 2.0 / OIDC** | Delegated authorization / authentication on top | [[Security Fundamentals]] |
| **Optimistic locking** | Version check on write; retry on conflict | [[Databases & Storage]] |
| **Outbox pattern** | Write event to DB table in same txn, relay to broker | [[Messaging & Event Streaming]] |
| **p99 latency** | 99th percentile response time | [[Scalability & Core Concepts]] |
| **PACELC** | CAP + latency vs consistency when no partition | [[Scalability & Core Concepts]] |
| **Presigned URL** | Temporary URL for direct upload/download to object storage | [[Databases & Storage]] |
| **Quorum** | W + R > N ensures read/write overlap | [[Databases & Storage]] |
| **Raft / Paxos** | Consensus algorithms | [[Distributed Systems Patterns]] |
| **Rate limiter** | Caps requests per client per time | [[Design a Rate Limiter]] |
| **Read replica** | Follower DB serving reads | [[Databases & Storage]] |
| **Reverse proxy** | Server-side intermediary (Nginx) | [[Networking Essentials]] |
| **RPO / RTO** | Acceptable data loss / downtime in disaster | [[Reliability & Observability]] |
| **Saga** | Sequence of local txns with compensations | [[Distributed Systems Patterns]] |
| **Sharding** | Horizontal partitioning of data across nodes | [[Databases & Storage]] |
| **SLI / SLO / SLA** | Metric / target / contract | [[Reliability & Observability]] |
| **Snowflake ID** | 64-bit time-sortable distributed ID | [[Distributed Systems Patterns]] |
| **SOLID** | SRP, OCP, LSP, ISP, DIP | [[SOLID Principles]] |
| **SPOF** | Single point of failure | [[Scalability & Core Concepts]] |
| **Thundering herd / stampede** | Many requests at once after expiry/restart | [[Caching]] |
| **Token bucket** | Rate-limit algorithm allowing bursts | [[Design a Rate Limiter]] |
| **Two-phase commit** | Blocking atomic commit across nodes | [[Distributed Systems Patterns]] |
| **Vertical scaling** | Bigger machine | [[Scalability & Core Concepts]] |
| **WebSocket** | Persistent full-duplex connection | [[API Design]] |
| **Write-through / write-behind** | Sync vs async cache+DB writes | [[Caching]] |

---

## Numbers to memorize
- 1 day ≈ **10^5 s** → **1M req/day ≈ 12 QPS**
- Memory read **~100 ns**; SSD random read **~100 µs**; same-DC round trip **~0.5 ms**; cross-continent **~150 ms**
- 99.9% = **~8.8 h/yr** (~43 min/mo) downtime; 99.99% = **~53 min/yr**
- Redis **~100K ops/s** per node; a single SQL node **~thousands of writes/s**
- base62, 7 chars ≈ **3.5 trillion** combinations
- Snowflake: **41** bits time | **10** machine | **12** sequence

## Pattern one-liners
**Strategy** swap algorithms · **Observer** notify subscribers · **Command** request as object (undo/queue) · **State** behavior by state · **Template Method** fixed skeleton, variable steps · **Chain of Responsibility** pipeline of handlers · **Adapter** convert interface · **Decorator** add behavior by wrapping · **Facade** simple front for complex subsystem · **Proxy** control access · **Composite** tree of uniform nodes · **Singleton** one instance · **Factory** create without naming class · **Builder** step-by-step construction
