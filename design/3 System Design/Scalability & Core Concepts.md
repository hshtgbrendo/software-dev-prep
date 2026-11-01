---
tags: [design, system-design, fundamentals]
aliases: [CAP, CAP theorem, PACELC, Consistency Models, Scalability, Latency vs Throughput]
---
# Scalability & Core Concepts

> [!summary] What you'll learn
> The foundational vocabulary and mental models behind every system design discussion: **vertical vs horizontal scaling**, why **stateless** services matter, **latency vs throughput** and percentiles, **availability and redundancy**, the **CAP theorem** and **PACELC**, **consistency models**, **ACID vs BASE**, and common failure dynamics (hot spots, thundering herds, backpressure).
>
> These concepts show up as the **reasons** behind design choices. You'll rarely be asked "define CAP" alone, but you'll constantly need it to justify decisions.

Back to [[Design Index]] · Prev: [[Back-of-the-Envelope Estimation]] · Next: [[Networking Essentials]]

---

## Scalability

**Scalability** is a system's ability to handle **more load** (users, requests, data) by adding resources, ideally without redesigning it and with cost growing roughly in proportion to the load.

### Vertical scaling ("scale up")
Give the existing machine **more power**: more CPU cores, more RAM, faster disks.
- ✅ **No code changes.** No distributed-systems complexity. Often the fastest fix.
- ✅ Modern machines are huge (hundreds of cores, terabytes of RAM). Many systems can go surprisingly far this way.
- ❌ **Hard ceiling**: there's a biggest machine you can buy.
- ❌ **Cost grows faster than capacity** at the high end.
- ❌ It's still a **single point of failure**, and upgrades may need downtime.

### Horizontal scaling ("scale out")
Add **more machines** and spread the load across them (behind a load balancer).
- ✅ Nearly **unlimited** scale; commodity hardware.
- ✅ **Redundancy**: if one machine dies, others continue.
- ✅ Can scale elastically up and down with demand.
- ❌ Requires the application to be designed for it: **stateless** services, partitioned data, and coordination between nodes.
- ❌ Brings distributed-systems problems: network failures, consistency, harder debugging.

> [!tip] In practice
> Scale the **stateless application tier horizontally** (easy), and scale the **data tier** with a combination of vertical scaling, caching, replication, and eventually sharding (hard). Databases are usually the bottleneck.

### Stateless services: the key to horizontal scaling
A service is **stateless** if it doesn't keep client-specific data in its own memory between requests. Any instance can handle any request.

**The problem with stateful servers:** if server A stores Brendan's session (login state, shopping cart) in memory:
- The load balancer must send all of Brendan's requests to A ("sticky sessions"), so load can't be balanced evenly.
- If A crashes or is removed during a deploy, Brendan is logged out and loses his cart.
- You can't freely add or remove servers.

**The fix:** move state **out** of the app servers into a shared store:
- Sessions → Redis or a database, or self-contained signed tokens (JWTs).
- Uploaded files → object storage (S3), not the local disk.
- Caches → a shared distributed cache, or accept per-instance caches as best effort.

Now app servers are interchangeable "cattle, not pets": add them, remove them, or replace them at will. The **state lives in the data tier**, which is scaled with [[Databases & Storage#Replication|replication]] and [[Databases & Storage#Sharding / Partitioning|sharding]].

### The typical scaling journey
Systems usually evolve through these stages as load grows. In an interview, you can narrate this progression:
1. **Single server** running the app and database.
2. **Separate the database** onto its own server (each can be sized independently).
3. **Load balancer + multiple stateless app servers** (scale the app tier; remove a SPOF).
4. **Cache** (Redis) for hot reads (takes load off the DB).
5. **CDN** for static assets and media (offload bandwidth, reduce latency globally).
6. **Read replicas** for the database (scale reads).
7. **Message queues + workers** for slow or non-critical work (keep requests fast).
8. **Sharding** or a horizontally scalable database for write scale and data size.
9. **Multiple data centers / regions** (availability and global latency).
10. **Split into services** as teams and complexity grow.

---

## Performance vocabulary

### Latency
**Latency** is the time to complete one operation, e.g., the time from sending a request to receiving the response (often measured in milliseconds).

### Throughput
**Throughput** is how much work the system completes per unit of time: requests per second (RPS/QPS), messages per second, MB/s.

### Latency vs throughput
They're related but distinct. A highway's **speed limit** is like latency (how fast one car gets through); the **number of lanes** is like throughput (how many cars per hour). You can often **trade one for the other**:
- **Batching** (writing 1,000 rows at once instead of one at a time) increases throughput but adds latency for each item, which waits for the batch.
- **Adding parallelism** increases throughput without necessarily improving per-request latency.

### Bandwidth
The **maximum** data transfer rate of a link (e.g., 1 Gbps). Throughput is how much you actually achieve.

### Percentiles: why averages lie
Latency varies from request to request. Report it as **percentiles**:
- **p50 (median)**: half of requests are faster than this.
- **p95, p99, p99.9**: 95%, 99%, 99.9% of requests are faster than this. These describe the **tail**.

**Why averages are misleading:** if 99 requests take 10 ms and 1 takes 5 seconds, the average is ~60 ms. That describes nobody's experience: most users see 10 ms, and one user waits 5 seconds.

**Why the tail matters so much at scale:**
- Your heaviest users (most data, most activity) often hit the slowest paths, and they're often your most valuable users.
- **Fan-out amplifies tail latency**: if rendering one page requires calls to 100 backend servers, and each has a 1% chance of being slow (its p99), then ~63% of page loads (1 − 0.99^100) will hit at least one slow call. The p99 of a component becomes the *typical* experience of the system.

SLAs and SLOs are therefore written in percentiles: "p99 latency under 300 ms."

### Little's Law
`L = λ × W`: the average number of items in a system (**L**) equals the arrival rate (**λ**) times the average time each spends in the system (**W**).

**Use it to size things:** if your service receives 1,000 requests/sec and each takes 200 ms, then on average 1,000 × 0.2 = **200 requests are in flight** at once. You need enough threads, connections, or event-loop capacity for ~200 concurrent requests (plus headroom). If latency doubles to 400 ms (a slow dependency), in-flight requests double to 400, which is how a slow database can exhaust your connection pool and cascade into an outage.

### Performance problem vs scalability problem
- **Performance problem**: the system is slow **even for a single user** (an inefficient algorithm, a missing index).
- **Scalability problem**: the system is fast for one user but **slows down under load** (contention, a saturated database, a single-threaded bottleneck).

---

## Availability, reliability, and redundancy

| Term | Meaning |
|---|---|
| **Availability** | The percentage of time the system is operational and responding (see [[Back-of-the-Envelope Estimation#Availability nines]]) |
| **Reliability** | The system works **correctly** over time: right results, no data corruption. A system can be available but unreliable (it responds quickly with wrong data) |
| **Fault tolerance** | Keeps working (possibly degraded) when components fail |
| **Resilience** | Recovers quickly from failures |
| **Durability** | Once data is acknowledged as written, it isn't lost (replication, write-ahead logs, backups) |
| **Single Point of Failure (SPOF)** | Any component whose failure takes the whole system down. Find and eliminate them with redundancy |

### Redundancy patterns
**Active–passive (failover):** a primary handles all traffic; a standby waits. A **heartbeat** detects primary failure, and the standby takes over (often by taking over its IP address or DNS name).
- ✅ Simple; no conflicts (only one active writer).
- ❌ Standby capacity sits idle; failover takes time (seconds to minutes); risk of **split brain** if both think they're primary.
- Typical for: relational database primaries.

**Active–active:** all nodes serve traffic simultaneously. If one fails, the others absorb its load.
- ✅ Uses all capacity; near-instant failover.
- ❌ For stateful systems with writes, you must handle **conflicts** between nodes; each node must have spare capacity to absorb a failed peer's load.
- Typical for: stateless app servers, multi-leader or leaderless databases, multi-region deployments.

---

## CAP theorem

### The statement
For a **distributed data store** (data replicated across multiple nodes), when a **network partition** occurs (some nodes can't communicate with others), you must choose between:
- **Consistency (C)**: every read receives the **most recent write** (or an error). All nodes appear to have the same, up-to-date data. (Technically, this is *linearizability*.)
- **Availability (A)**: every request to a non-failed node receives a **non-error response**, even though it might be stale.

You **can't guarantee both during a partition**. (Eric Brewer's conjecture, 2000; proved by Gilbert and Lynch, 2002.)

### Why "P" isn't optional
Networks *do* partition (switches fail, cables get cut, cloud zones lose connectivity, long garbage-collection pauses make nodes look dead). A distributed system must tolerate partitions, so the real choice is **what to do when one happens: CP or AP**. "CA" only exists for a single-node system, which isn't distributed.

### A concrete scenario
Two replicas, one in Seattle and one in Virginia, both holding your account balance of $100. The link between them breaks. A user in Virginia withdraws $80. Then a user in Seattle asks for the balance.
- **CP choice:** Seattle can't confirm it has the latest data, so it **refuses** (returns an error or blocks) until the partition heals. Correct, but unavailable.
- **AP choice:** Seattle **answers $100**, which is stale. Available, but inconsistent. The replicas reconcile later.

| Choice | Behavior during a partition | Good for | Examples |
|---|---|---|---|
| **CP** | Some requests fail or wait, to avoid returning stale or conflicting data | Bank balances, inventory counts, seat/ticket booking, distributed locks, leader election, configuration | ZooKeeper, etcd, Consul, HBase, Google Spanner, single-primary relational databases |
| **AP** | Always respond, possibly with stale data; reconcile afterward | Social feeds, like counts, product catalogs, shopping carts, DNS, user presence | Cassandra, DynamoDB (default settings), CouchDB, Riak, DNS |

> [!tip] How to use CAP in an interview
> Apply it to **specific data**, not the whole system: "For **seat inventory**, I need CP. Selling the same seat twice is unacceptable, so I'd rather show an error. For the **event listing and descriptions**, AP is fine. Slightly stale info is OK, and the page must load."

## PACELC

CAP only describes behavior **during partitions**, which are rare. PACELC (Daniel Abadi, 2010) extends it:

> **If** there's a **P**artition, choose **A**vailability or **C**onsistency; **E**lse (normal operation), choose **L**atency or **C**onsistency.

Even when the network is healthy, **strong consistency costs latency**: a write must wait until replicas confirm it, and those replicas might be in another region. Many systems choose low latency (and eventual consistency) even when everything is fine.

| Classification | Meaning | Examples |
|---|---|---|
| **PA/EL** | Favor availability during partitions and latency otherwise | Cassandra, DynamoDB (default), Riak |
| **PC/EC** | Favor consistency always (at the cost of latency/availability) | Google Spanner, VoltDB, traditional synchronously replicated databases |
| **PA/EC** or **PC/EL** | Mixed | Some configurable systems (e.g., MongoDB depending on settings) |

---

## Consistency models

"Consistency" in distributed systems describes **what values a read is allowed to return** given the writes that have happened. From strongest to weakest:

| Model | Guarantee | Example |
|---|---|---|
| **Linearizability (strong consistency)** | Every operation appears to happen instantaneously at a single point in time, in real-time order. Once a write completes, all subsequent reads (from anyone) see it | Reading from a single primary; Spanner; etcd |
| **Sequential consistency** | All nodes see all operations in the **same order**, but not necessarily in real-time order | Some multiprocessor memory models |
| **Causal consistency** | Operations that are **causally related** are seen in the same order by everyone (a reply is never seen before the message it replies to); unrelated operations may be seen in different orders | Some geo-distributed databases; collaboration apps |
| **Read-your-writes** | A user always sees **their own** updates immediately (others may see them later) | After you edit your profile, you see the change |
| **Monotonic reads** | Once you've seen a value, you never see an older one later (no "going back in time") | Pinning a user to the same replica |
| **Eventual consistency** | If no new writes occur, all replicas **eventually** converge to the same value. No guarantee about *when*, or what you read in the meantime | DNS, Cassandra at low consistency levels, async replicas |

### Why weaker models exist
Stronger consistency requires more coordination between nodes (waiting for acknowledgments, consensus rounds), which means **higher latency** and **lower availability**. Weaker models allow nodes to respond independently, which is faster and more available, but the **application must tolerate staleness and conflicts**.

### Read-your-writes in practice
This is one of the most common practical requirements. A user posts a comment and refreshes. If their read hits a lagging replica, the comment "disappears," which looks like a bug. Fixes:
- Read from the **primary** for a short window after a user writes (e.g., 10 seconds), or for that user's own data.
- Track the write's position in the replication log and only read from replicas that have caught up.
- Update the client's UI optimistically so it doesn't depend on the next read.

### Handling conflicts under eventual consistency
When two replicas accept conflicting writes, something must resolve them:
- **Last-write-wins (LWW)** by timestamp: simple, but can silently drop writes (and clocks drift).
- **Application-level merge**: e.g., union of shopping cart items.
- **Version vectors / vector clocks**: detect concurrent writes and surface them for resolution.
- **CRDTs**: data structures designed to merge automatically and deterministically (see [[Distributed Systems Patterns]]).

---

## ACID vs BASE

Two philosophies for data stores.

| ACID (traditional relational databases) | BASE (many NoSQL systems) |
|---|---|
| **Atomicity**: a transaction's operations all succeed or all fail; no partial results | **Basically Available**: the system responds even if some data is stale or some nodes are down |
| **Consistency**: transactions move the database from one valid state to another (constraints, foreign keys, invariants hold) | **Soft state**: state may change over time without new input (replicas catching up) |
| **Isolation**: concurrent transactions don't interfere (as if run one at a time, depending on isolation level) | **Eventual consistency**: replicas converge given time |
| **Durability**: once committed, data survives crashes | |

> [!warning] Classic gotcha
> The "C" in **ACID** (invariants and constraints hold) is **not** the "C" in **CAP** (every read sees the latest write / linearizability). They're different concepts that share a letter.

More detail on transactions and isolation levels: [[Databases & Storage#Transactions & isolation levels]].

---

## Failure dynamics and load concepts

### Elasticity and auto-scaling
**Elasticity** is automatically adding capacity when load rises and removing it when load falls (AWS Auto Scaling groups, Kubernetes Horizontal Pod Autoscaler), based on metrics like CPU, request rate, or queue depth. It saves cost and absorbs spikes, but scaling takes time (minutes to boot instances), so keep headroom for sudden spikes.

### Load shedding
When overloaded, **deliberately reject some requests** (fast 503s) so the system can keep serving the rest well, rather than trying to serve everyone and failing everyone. Prioritize: drop analytics before checkout.

### Backpressure
A mechanism for a **slow downstream component to tell upstream components to slow down**, instead of being overwhelmed. Examples: bounded queues that block or reject when full, TCP flow control, reactive streams, returning 429/503. Without backpressure, unbounded queues grow until memory runs out.

### Hot spots and hot keys
A **hot spot** is when one shard, node, or key receives a **disproportionate share of traffic**. Examples: a celebrity's profile, a viral post, a flash-sale product, or all new writes going to the latest time-range partition. Mitigations: caching the hot key, replicating it across nodes, splitting it into sub-keys (`post:123#1..#10`), or choosing a better partition key.

### Thundering herd
**Many clients hit a resource at the same moment**: a popular cache entry expires and thousands of requests hit the database simultaneously; all clients reconnect at once after an outage; a cron job fires at midnight on 1,000 servers. Mitigations: **jitter** (randomize timing), request coalescing (only one request recomputes), staggered retries with exponential backoff, rate limiting. (See [[Caching]] and [[Reliability & Observability]].)

### Cascading failure
One component fails or slows, its load shifts to others or its callers pile up waiting, those fail too, and so on. Defend with timeouts, circuit breakers, bulkheads, load shedding, and capacity headroom.

### Blast radius
**How much of the system (or how many users) is affected when something fails.** Reduce it with isolation: separate cells or shards, regional independence, bulkheads, gradual rollouts (canaries), and feature flags.

---

## Quick self-test
- [ ] Vertical vs horizontal scaling: pros, cons, and where each applies (app tier vs data tier).
- [ ] Why must app servers be stateless to scale horizontally? Where does the state go?
- [ ] Explain latency vs throughput with an example of trading one for the other.
- [ ] Why use p99 instead of averages? Explain how fan-out amplifies tail latency.
- [ ] Use Little's Law: 2,000 req/s at 50 ms average. How many requests are in flight?
- [ ] Explain CAP with a concrete example. Why isn't "CA" an option for distributed systems?
- [ ] What does PACELC add to CAP?
- [ ] Give a feature that needs strong consistency and one that's fine with eventual consistency.
- [ ] How do you provide read-your-writes on top of asynchronous replicas?
- [ ] What's the difference between the C in ACID and the C in CAP?
- [ ] Explain thundering herd, hot key, backpressure, and load shedding.

Next: [[Networking Essentials]]
