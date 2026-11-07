---
tags: [design, system-design, estimation]
aliases: [Capacity Estimation, Latency Numbers, Napkin Math]
---
# Back-of-the-Envelope Estimation

> [!summary]
> Quick estimates let you **justify design decisions** ("40K QPS → one DB can't handle it → add cache + replicas"). Interviewers care about **reasonable orders of magnitude and stated assumptions**, not exact arithmetic. Round aggressively.

Back to [[Design Index]] · Prev: [[System Design Interview Framework]]

---

## Powers of two / data units

| Power | Approx | Name |
|---|---|---|
| 2^10 | 1 thousand | 1 KB |
| 2^20 | 1 million | 1 MB |
| 2^30 | 1 billion | 1 GB |
| 2^40 | 1 trillion | 1 TB |
| 2^50 | 1 quadrillion | 1 PB |

Handy sizes:
- ASCII char = 1 byte; UUID = 16 bytes binary (36 as a string); int64/timestamp = 8 bytes
- A tweet/short text row ≈ 100s of bytes to 1 KB with metadata
- Image ≈ 200 KB–2 MB; 1 min of compressed HD video ≈ 50–100 MB

## Time conversions
- **1 day ≈ 86,400 s ≈ 10^5 s** (use 100K for napkin math)
- 1 month ≈ 2.5 million s; 1 year ≈ 3 × 10^7 s
- **1M requests/day ≈ 12 QPS** (1,000,000 / 86,400)
- 100M/day ≈ 1,200 QPS; 1B/day ≈ 12K QPS

---

## Latency numbers every engineer should know

(Originally from Jeff Dean / Peter Norvig; hardware has improved, but the **relative magnitudes** are what matter.)

| Operation | Approx latency |
|---|---|
| L1 cache reference | ~1 ns |
| Branch mispredict | ~3–5 ns |
| L2 cache reference | ~4–7 ns |
| Mutex lock/unlock | ~20–25 ns |
| **Main memory reference** | **~100 ns** |
| Compress 1 KB (fast compressor) | ~2–3 µs |
| Send 1 KB over 1 Gbps network | ~10 µs |
| **SSD random read (4 KB)** | **~16–150 µs** |
| Read 1 MB sequentially from memory | ~3–250 µs |
| **Round trip within same datacenter** | **~0.5 ms** |
| Read 1 MB sequentially from SSD | ~50 µs–1 ms |
| **HDD disk seek** | **~2–10 ms** |
| Read 1 MB sequentially from HDD | ~1–20 ms |
| **Round trip US ↔ Europe** | **~150 ms** |

> [!important] Takeaways
> - **Memory is ~100–1000× faster than disk/SSD** → caching matters.
> - **Network round trips are expensive** → batch calls, avoid chatty services, avoid N+1 queries.
> - **Sequential I/O is far faster than random I/O** → log-structured storage (Kafka, LSM trees) exploits this.
> - **Cross-region calls are slow (~100+ ms)** → keep data near users (CDN, geo-replication).
> - Compress before sending over the network.

---

## Availability nines

| Availability | Downtime / year | Downtime / month |
|---|---|---|
| 99% (two nines) | 3.65 days | ~7.2 hours |
| 99.9% (three nines) | 8.76 hours | ~43.8 min |
| 99.99% (four nines) | 52.6 min | ~4.4 min |
| 99.999% (five nines) | 5.26 min | ~26 s |

- Components **in series** multiply: two services at 99.9% each → 99.8% overall.
- Components **in parallel** (redundant): 1 − (1 − 0.999)² = 99.9999%.
See [[Reliability & Observability#SLI, SLO, SLA]].

---

## Rough capacity of single components (order of magnitude, 2020s hardware)

| Component | Ballpark |
|---|---|
| Single web/app server (simple API) | 1K–10K+ req/s (heavily workload-dependent) |
| PostgreSQL/MySQL single node | ~1K–10K writes/s; 10K+ simple indexed reads/s |
| Redis single node | ~100K+ ops/s, sub-ms latency |
| Kafka cluster | millions of messages/s |
| Server RAM | 64 GB – 1 TB+ |
| Server disk (SSD) | multiple TB |
| Network NIC | 10–100 Gbps |

Use these to decide **when** you need sharding, replicas, or caching. *State them as assumptions.*

---

## The estimation recipe

1. **Users**: DAU (daily active users).
2. **Traffic**: actions per user per day → requests/day → **QPS = requests/day ÷ 10^5**. **Peak ≈ 2–3× average**.
3. **Read:write ratio** → read QPS vs write QPS.
4. **Storage**: new records/day × size × retention (e.g., 5 years) × replication factor (often 3).
5. **Bandwidth**: QPS × payload size.
6. **Memory for cache**: often the 80/20 rule. Cache ~20% of daily hot data.

### Worked example: Twitter-like service
Assumptions: 300M MAU, **150M DAU**; each user posts 2 tweets/day and views the timeline 20 times/day; 10% of tweets have media.

| Metric | Calculation | Result |
|---|---|---|
| Write QPS | 150M × 2 / 10^5 | **~3K tweets/s** (peak ~6–9K) |
| Read QPS (timeline) | 150M × 20 / 10^5 | **~30K/s** (peak ~60–90K) → read-heavy 10:1 |
| Text storage/day | 300M tweets × ~300 B | ~90 GB/day → ~33 TB/year |
| Media storage/day | 30M × 1 MB | **~30 TB/day** → object storage + CDN |
| Cache | timelines of ~20% of DAU × 800 tweet IDs × 8 B | 30M × 6.4 KB ≈ 200 GB → Redis cluster |

**Decisions this drives:** read-heavy → precompute timelines + cache; media dominates storage → S3 + CDN; 3K writes/s is manageable but grows → shard tweets by ID.

### Worked example: URL shortener
100M new URLs/month, 100:1 read:write.
- Writes: 100M / (30 × 10^5) ≈ **~40/s**. Reads: **~4K/s**.
- Storage: 100M × 12 months × 5 years × 500 B ≈ **3 TB** → a single sharded DB or NoSQL is fine.
- Key space: base62 with 7 chars = 62^7 ≈ **3.5 trillion** keys → plenty.
See [[Design a URL Shortener]].

---

## Tips
- Say the assumptions out loud and write them down.
- Round: 86,400 → 100K; 2.5M → "a few million".
- Do only the math that **changes a decision**.
- Sanity-check against the component capacities above.

Next: [[Scalability & Core Concepts]]
