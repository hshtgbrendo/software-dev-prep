---
tags: [design, system-design, estimation]
aliases: [Capacity Estimation, Latency Numbers, Napkin Math]
---
# Back-of-the-Envelope Estimation

> [!summary] What you'll learn
> How to quickly estimate traffic, storage, bandwidth, and memory for a system, and how to use those numbers to **justify design decisions**. You'll learn the reference numbers worth memorizing (latencies, time conversions, availability "nines," component capacities), a step-by-step recipe, and two fully worked examples.
>
> **Why it matters:** estimates turn vague claims into decisions. "We'll get a lot of traffic" says nothing. "~40K reads/sec at peak, and one Postgres node handles a few thousand, so we need caching and replicas" justifies architecture. Interviewers want **reasonable orders of magnitude with stated assumptions**, not precise arithmetic.

Back to [[Design Index]] · Prev: [[System Design Interview Framework]] · Next: [[Scalability & Core Concepts]]

---

## Why estimate at all?

Back-of-the-envelope math answers questions like:
- Does this fit on **one machine**, or do we need to distribute it?
- Is the system **read-heavy** or **write-heavy** (which drives caching vs write optimization)?
- Will the dataset fit **in memory** (cache everything?) or only on disk?
- Is **bandwidth** a concern (video), or is it negligible (text)?
- Roughly how many **servers** do we need?

The goal is **within a factor of ~10**. Whether it's 30K or 50K QPS rarely changes the design; whether it's 300 or 300K always does.

---

## Data sizes and powers of two

| Power of 2 | Approx. value | Name | Short |
|---|---|---|---|
| 2^10 | 1 thousand (10^3) | Kilobyte | KB |
| 2^20 | 1 million (10^6) | Megabyte | MB |
| 2^30 | 1 billion (10^9) | Gigabyte | GB |
| 2^40 | 1 trillion (10^12) | Terabyte | TB |
| 2^50 | 1 quadrillion (10^15) | Petabyte | PB |

**Typical sizes to assume:**
| Thing | Size |
|---|---|
| A character (ASCII) | 1 byte (UTF-8: 1–4 bytes) |
| Integer / timestamp / Snowflake ID | 4–8 bytes |
| UUID | 16 bytes binary, 36 bytes as a string |
| A short text record with metadata (tweet, chat message) | ~200 bytes – 1 KB |
| A user profile row | ~1 KB |
| A compressed photo | ~200 KB – 2 MB |
| 1 minute of HD video (compressed) | ~50–100 MB |
| A web page (HTML) | ~100 KB; with assets, a few MB |

---

## Time conversions (the most useful trick)

- **1 day = 86,400 seconds ≈ 10^5 seconds** (use 100K for napkin math)
- 1 month ≈ 2.5 × 10^6 seconds; 1 year ≈ 3 × 10^7 seconds

So converting "per day" to "per second" is **dividing by 100,000**:
| Per day | Per second (avg) |
|---|---|
| 1 million | ~12 |
| 10 million | ~120 |
| 100 million | ~1,200 |
| 1 billion | ~12,000 |

**Peak traffic** is usually **2–3× average** (more for event-driven spikes like sales or sports).

---

## Latency numbers every engineer should know

(Popularized by Jeff Dean and Peter Norvig. Absolute values improve with hardware, but the **relative magnitudes** are what you should internalize.)

| Operation | Approx. latency | In human terms (if L1 = 1 second) |
|---|---|---|
| L1 cache reference | ~1 ns | 1 second |
| Branch mispredict | ~3–5 ns | 4 seconds |
| L2 cache reference | ~4–7 ns | 6 seconds |
| Mutex lock/unlock | ~20–25 ns | 25 seconds |
| **Main memory (RAM) reference** | **~100 ns** | ~2 minutes |
| Compress 1 KB (fast compressor) | ~2–3 µs | ~45 minutes |
| Send 1 KB over a 1 Gbps network | ~10 µs | ~3 hours |
| **SSD random read (4 KB)** | **~16–150 µs** | ~1 day |
| Read 1 MB sequentially from memory | ~3–250 µs | |
| **Round trip within the same datacenter** | **~0.5 ms** | ~6 days |
| Read 1 MB sequentially from SSD | ~50 µs – 1 ms | |
| **HDD disk seek** | **~2–10 ms** | ~4 months |
| Read 1 MB sequentially from HDD | ~1–20 ms | |
| **Round trip California ↔ Netherlands** | **~150 ms** | ~5 years |

> [!important] What these numbers teach (memorize the lessons, not the digits)
> 1. **Memory is roughly 100–1000× faster than SSD/disk**, which is why caching works so well.
> 2. **Network round trips are expensive** relative to computation. Batch requests, avoid "chatty" designs where one user action triggers dozens of sequential calls, and avoid N+1 query patterns.
> 3. **Sequential I/O is much faster than random I/O.** Kafka and LSM-tree databases (Cassandra, RocksDB) are fast because they write sequentially to append-only logs.
> 4. **Crossing continents costs ~100–150 ms per round trip**, which is why CDNs and multi-region deployments exist. The speed of light is a hard limit.
> 5. **Compression is cheap** compared with sending bytes over a network.

---

## Availability nines

Availability is usually expressed as a percentage of time a system is up. Each extra "nine" is 10× less downtime and much harder (and more expensive) to achieve.

| Availability | Downtime per year | Per month | Per week |
|---|---|---|---|
| 99% ("two nines") | 3.65 days | ~7.3 hours | ~1.7 hours |
| 99.9% ("three nines") | ~8.8 hours | ~43.8 minutes | ~10 minutes |
| 99.95% | ~4.4 hours | ~22 minutes | ~5 minutes |
| 99.99% ("four nines") | ~52.6 minutes | ~4.4 minutes | ~1 minute |
| 99.999% ("five nines") | ~5.3 minutes | ~26 seconds | ~6 seconds |

**Combining components:**
- **In series** (a request needs A *and* B): availabilities **multiply**. Two services at 99.9% → 0.999 × 0.999 ≈ **99.8%**. Ten dependencies at 99.9% each → ~99.0%. That's why long synchronous call chains hurt reliability.
- **In parallel** (redundant: works if A *or* B is up): unavailability multiplies. Two replicas at 99.9% → 1 − (0.001 × 0.001) = **99.9999%**. That's why redundancy is so effective (assuming failures are independent).

More on targets and error budgets in [[Reliability & Observability#SLI, SLO, SLA]].

---

## Rough capacity of single components

Order-of-magnitude figures for modern hardware. Real numbers vary enormously with workload, so **state them as assumptions**.

| Component | Ballpark capacity |
|---|---|
| One app server (simple JSON API, Node/Go/Java) | ~1K–10K+ requests/sec |
| PostgreSQL/MySQL, single node | ~1K–10K writes/sec; 10K–50K+ simple indexed reads/sec |
| Redis, single node | ~100K+ operations/sec, sub-millisecond latency |
| Kafka cluster | Hundreds of thousands to millions of messages/sec |
| Cassandra node | ~10K+ writes/sec per node, scales roughly linearly with nodes |
| Server RAM | 64 GB – 1 TB+ |
| Server SSD | Several TB per server |
| Network card | 10–100 Gbps |
| One WebSocket server | ~10K–1M concurrent connections (depends on tuning and message rate) |

---

## The estimation recipe

1. **State assumptions about users**: DAU (daily active users). If given MAU (monthly), DAU is often ~25–50% of MAU.
2. **Traffic**: actions per user per day → total per day → **QPS ≈ daily total ÷ 10^5**. Multiply by ~2–3 for **peak**.
3. **Read vs write**: compute both. The ratio tells you whether to optimize reads (cache, replicas) or writes (sharding, batching, write-optimized stores).
4. **Storage**: new records per day × size per record × retention period (e.g., 5 years) × **replication factor** (often 3).
5. **Bandwidth**: QPS × average payload size (ingress for writes, egress for reads).
6. **Cache memory**: often the **80/20 rule**: ~20% of the data serves ~80% of the requests. Cache that hot 20% of daily read data.
7. **Servers** (if useful): peak QPS ÷ QPS per server, plus headroom.

**Then say what the numbers mean for the design.** The numbers by themselves don't count for anything.

---

## Worked example: Twitter-like service

**Assumptions** (state these out loud):
- 300M monthly active users, **150M daily active users**.
- Each user posts **2 tweets/day** on average and views their home timeline **20 times/day**.
- 10% of tweets include media averaging 1 MB.
- Average tweet with metadata: ~300 bytes. Retain 5 years.

| Metric | Calculation | Result | What it implies |
|---|---|---|---|
| Tweets per day | 150M × 2 | 300M/day | |
| **Write QPS** | 300M ÷ 10^5 | **~3,000/s** (peak ~6–9K) | Manageable, but a single DB node would be strained at peak; plan to shard |
| Timeline views/day | 150M × 20 | 3B/day | |
| **Read QPS** | 3B ÷ 10^5 | **~30,000/s** (peak ~60–90K) | **Read-heavy (10:1)**: precompute timelines, cache aggressively |
| Text storage/day | 300M × 300 B | ~90 GB/day | |
| Text storage, 5 years | 90 GB × 365 × 5 | ~165 TB (× 3 replicas ≈ 500 TB) | Needs a horizontally scalable store |
| **Media storage/day** | 30M × 1 MB | **~30 TB/day** | Media dominates: object storage + CDN, never the database |
| Timeline cache | 20% of DAU (30M) × 800 tweet IDs × 8 B | ~200 GB | Fits in a Redis cluster |
| Media egress | Read-heavy media: tens of GB/s at peak | | Must be served by a CDN |

**Decisions this drives:** precomputed timelines in a cache (reads dominate), sharded tweet storage, media in S3 behind a CDN, async fan-out for writes. See [[Design a News Feed]].

---

## Worked example: URL shortener

**Assumptions:** 100M new short URLs per month; 100:1 read:write ratio; each record ~500 bytes; retain 5 years.

| Metric | Calculation | Result |
|---|---|---|
| Write QPS | 100M ÷ (30 × 10^5) | **~40/s** |
| Read QPS | 40 × 100 | **~4,000/s** (peak ~10K) |
| Records over 5 years | 100M × 12 × 5 | 6 billion |
| Storage | 6B × 500 B | **~3 TB** |
| Key space needed | 6B unique codes | base62 with 7 characters = 62^7 ≈ **3.5 trillion**, which is plenty |
| Cache | Hot 20% of daily reads | A few GB of Redis |

**What it implies:** writes are trivial; reads benefit enormously from caching; 3 TB fits on a small sharded cluster or a managed NoSQL store. See [[Design a URL Shortener]].

---

## Tips for the interview
- **Say assumptions out loud and write them down.** The interviewer may adjust them, and that's fine.
- **Round aggressively**: 86,400 → 100K; 2.6M → "a few million"; 365 → 400 if it makes the math easier.
- **Use scientific notation** for big numbers: 3 × 10^8 × 3 × 10^2 = 9 × 10^10.
- **Only calculate what changes a decision.** Skip bandwidth for a text app if it's obviously small.
- **Sanity-check against the component capacities** above.
- **Spend 2–4 minutes**, not 15.

## Quick self-test
- [ ] Convert 500M requests/day to average and peak QPS.
- [ ] Why is memory access so much faster than SSD, and what design technique does that justify?
- [ ] How much downtime per month does 99.9% allow? 99.99%?
- [ ] Two services in series at 99.9% each: what's the combined availability? In parallel?
- [ ] Estimate storage for 1B photos at 500 KB each with 3× replication.
- [ ] How many 7-character base62 codes exist?

Next: [[Scalability & Core Concepts]]
