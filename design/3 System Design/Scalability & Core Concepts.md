---
tags: [design, system-design, fundamentals]
aliases: [CAP, CAP theorem, PACELC, Consistency Models, Scalability]
---
# Scalability & Core Concepts

> [!summary]
> The vocabulary and mental models behind every system design discussion: scaling, latency vs throughput, availability, CAP/PACELC, and consistency models.

Back to [[Design Index]] · Prev: [[Back-of-the-Envelope Estimation]]

---

## Scalability

**Scalability** = the ability to handle increased load by adding resources.

### Vertical scaling (scale up)
Bigger machine: more CPU, RAM, faster disk.
- ✅ Simple; no code changes; no distributed-systems problems.
- ❌ Hardware ceiling; expensive at the top end; still a **single point of failure**.

### Horizontal scaling (scale out)
More machines behind a load balancer.
- ✅ Near-unlimited scale; redundancy and fault tolerance; commodity hardware.
- ❌ Requires **stateless** app servers (or shared state); data partitioning; network complexity.

> [!important] Stateless services
> To scale horizontally, app servers must not keep user state in local memory. Move sessions to Redis or a DB, or use stateless tokens (JWT), so **any server can handle any request**. State lives in the data tier, which is scaled with [[Databases & Storage#Replication|replication]] and [[Databases & Storage#Sharding / Partitioning|sharding]].

### The typical scaling journey
1. Single server (app + DB)
2. Separate DB server
3. Load balancer + multiple stateless app servers
4. Cache layer (Redis) for hot reads
5. CDN for static assets/media
6. DB read replicas
7. Async work via message queues + workers
8. DB sharding / NoSQL for write scale
9. Multiple data centers / regions
10. Split into services as teams and complexity grow

---

## Performance vocabulary

- **Latency**: time for one request (ms). Measure **percentiles**:
  - **p50** (median), **p95**, **p99**, **p99.9**: "99% of requests are faster than X."
  - **Tail latency** (p99+) matters: with fan-out to 100 backends, almost every user request hits some backend's p99.
  - Averages hide outliers; don't use them for SLAs.
- **Throughput**: work per unit time (requests/s, MB/s).
- **Bandwidth**: maximum data transfer capacity of a link.
- **Latency vs throughput**: you can often trade one for the other (batching ↑throughput, ↑latency).
- **Little's Law**: `L = λ × W` (items in system = arrival rate × time in system). E.g., 1,000 req/s × 0.2 s = 200 concurrent requests in flight → size thread pools and connection pools accordingly.
- **Performance problem** = slow for one user. **Scalability problem** = fast for one user, slow under load.

---

## Availability & reliability

- **Availability**: % of time the system is operational (see the nines table in [[Back-of-the-Envelope Estimation#Availability nines]]).
- **Reliability**: the system works *correctly* over time (no data corruption, correct results).
- **Fault tolerance**: keeps working despite component failures.
- **Durability**: once data is written, it's not lost (replication, backups, WAL).
- **Single Point of Failure (SPOF)**: any component whose failure takes the system down. Eliminate with **redundancy**.

**Redundancy patterns:**
- **Active–passive (failover)**: a standby takes over on failure (heartbeat). Some downtime during failover; standby capacity is idle.
- **Active–active**: all nodes serve traffic; on failure, the others absorb its load. Better utilization; harder with writes (conflicts).

---

## CAP theorem

In a distributed data store, when a **network Partition** happens, you must choose between:
- **Consistency (C)**: every read sees the most recent write (linearizability).
- **Availability (A)**: every request gets a (non-error) response, possibly stale.

**P is not optional.** Networks *do* partition. So the real choice is **CP vs AP during a partition**.

| Choice | Behavior during partition | Use when | Examples |
|---|---|---|---|
| **CP** | Refuse/err some requests to stay consistent | Money, inventory counts, bookings, locks, leader election | ZooKeeper, etcd, HBase, Spanner, traditional RDBMS (single primary) |
| **AP** | Serve possibly stale data; reconcile later | Social feeds, likes, product catalogs, DNS, shopping carts | Cassandra, DynamoDB (default), CouchDB, DNS |

> [!tip] Interview usage
> Tie CAP to **specific data**, not the whole system: "For the **seat booking** I need CP. Double-selling a seat is unacceptable. For the **event listing page** AP is fine; slightly stale is OK."

## PACELC (the more complete version)
**If Partition → choose A or C; Else (normal operation) → choose Latency or Consistency.**
Even without failures, strong consistency costs latency (you wait for replicas to agree).
- **PA/EL**: Cassandra, DynamoDB (favor availability and low latency)
- **PC/EC**: Spanner, traditional sync-replicated RDBMS (favor consistency)

---

## Consistency models (strongest → weakest)

| Model | Guarantee | Example |
|---|---|---|
| **Linearizability (strong)** | Ops appear instantaneous, in real-time order; reads see the latest write | Single-leader DB reading from the leader; Spanner; etcd |
| **Sequential** | All nodes see ops in the same order (not necessarily real-time) | |
| **Causal** | Causally related ops seen in order (reply after the original post) | Some geo-distributed DBs |
| **Read-your-writes** | A user always sees their own updates | Read from the leader after your own write; sticky sessions |
| **Monotonic reads** | Never see older data after seeing newer | Pin a user to one replica |
| **Eventual** | If writes stop, all replicas *eventually* converge | DNS, Cassandra (at low consistency levels), async replicas |

> **Read-your-writes** is a common practical requirement: after a user edits their profile, *they* must see the change immediately even if others see it a bit later.

### Strong vs eventual consistency in practice
- Strong: simpler to reason about; higher latency; lower availability during failures.
- Eventual: fast and available; app must handle staleness and conflicts (last-write-wins, version vectors, CRDTs).

---

## ACID vs BASE

| ACID (traditional RDBMS) | BASE (many NoSQL) |
|---|---|
| **Atomicity**: all or nothing | **Basically Available** |
| **Consistency**: constraints/invariants hold | **Soft state**: state may change without input (replication catching up) |
| **Isolation**: concurrent txns don't interfere | **Eventually consistent** |
| **Durability**: committed = persisted | |

Note: the "C" in ACID (invariants) ≠ the "C" in CAP (linearizability). A classic gotcha.
More: [[Databases & Storage#Transactions & isolation levels]]

---

## Other key concepts
- **Elasticity**: automatically add/remove capacity with load (auto-scaling groups, Kubernetes HPA).
- **Load shedding**: deliberately drop low-priority requests under overload to protect the system.
- **Backpressure**: downstream signals upstream to slow down (bounded queues, TCP flow control, reactive streams).
- **Hot spot / hot key**: one shard or key gets disproportionate traffic (a celebrity's tweet). Mitigate with caching, key splitting, replication.
- **Thundering herd**: many clients hit a resource at once (cache expiry, service restart). Mitigate with jitter, request coalescing, warm-up.
- **Blast radius**: how much breaks when one thing fails. Reduce with cells, bulkheads, regional isolation.

## Self-test
- [ ] Why must app servers be stateless to scale horizontally?
- [ ] Explain CAP. Why isn't "CA" a real option for distributed systems?
- [ ] What does PACELC add?
- [ ] Give a feature that needs strong consistency and one that's fine with eventual.
- [ ] Why use p99 instead of averages?

Next: [[Networking Essentials]]
