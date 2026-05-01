---
layout: default
title: "2.3 Back-of-Envelope Estimation"
parent: "Phase 2: Core System Design"
nav_order: 3
---

# Back-of-Envelope Estimation
{: .no_toc }

<details open markdown="block">
  <summary>Table of Contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

Back-of-envelope estimation is a **first-class interview skill**. Interviewers use it to verify whether you understand the scale of the system you're designing. An estimate that's off by 10× signals you don't understand the domain. Done well, it drives architectural decisions: "we need caching because the DB can't handle 50,000 QPS."

---

## Powers of 2 Reference

Memorize this table. Every storage/traffic estimate traces back to it.

| Power | Approximate value | Common unit |
|:------|:-----------------|:------------|
| 2¹⁰ | 1,024 | 1 KB |
| 2²⁰ | 1,048,576 | 1 MB |
| 2³⁰ | 1,073,741,824 | 1 GB |
| 2⁴⁰ | ~1 trillion | 1 TB |
| 2⁵⁰ | ~1 quadrillion | 1 PB |

**Quick conversion tricks:**
- 1 million = 10⁶ ≈ 2²⁰ (1 MB in bytes)
- 1 billion = 10⁹ ≈ 2³⁰ (1 GB in bytes)
- 1 trillion = 10¹² ≈ 2⁴⁰ (1 TB in bytes)

---

## Latency Numbers Every Engineer Must Know

These numbers are the foundation of performance intuition. If your design requires 100 DB reads per user request, and each DB read is 1ms, your P99 is at least 100ms — before any application logic.

| Operation | Latency | Notes |
|:----------|:--------|:------|
| L1 cache reference | 0.5 ns | ~2 GHz clock cycle |
| Branch mispredict | 5 ns | CPU pipeline flush |
| L2 cache reference | 7 ns | |
| Mutex lock/unlock | 25 ns | |
| Main memory reference | 100 ns | 200× L1 cache |
| Compress 1 KB with Zippy | 3,000 ns = 3 µs | |
| Send 2 KB over 1 Gbps network | 20,000 ns = 20 µs | |
| Read 1 MB sequentially from memory | 250,000 ns = 250 µs | |
| RTT within same datacenter | 500,000 ns = 0.5 ms | |
| **SSD random read** | 16,000 ns = **16 µs** | |
| Read 1 MB sequentially from SSD | 250,000 ns = 250 µs | |
| **HDD seek** | 10,000,000 ns = **10 ms** | 20,000× SSD random read |
| Read 1 MB sequentially from HDD | 30,000,000 ns = 30 ms | |
| Packet RTT US to Europe | 150,000,000 ns = **150 ms** | |

### Intuitions from the Latency Table

- **Memory is 200× faster than SSD, 10,000× faster than HDD.** This is why databases buffer in memory (InnoDB buffer pool).
- **SSD is 625× faster than HDD for random reads.** Never store hot data on spinning disk.
- **Cross-datacenter RTT is 150ms.** Synchronous calls between US and EU add 150ms minimum — never do this in a critical path.
- **L1 cache → RAM is a 200× jump.** Cache misses are expensive even at the hardware level.

---

## Traffic Estimation

### DAU → QPS Conversion

Most systems are defined by **DAU (Daily Active Users)** and the expected requests per user per day.

```
QPS = (DAU × requests_per_user_per_day) / 86,400 seconds

Peak QPS ≈ 2–3× average QPS
```

**Examples:**

| System | DAU | Req/user/day | Avg QPS | Peak QPS |
|:-------|:----|:-------------|:--------|:---------|
| Twitter-like feed | 100M | 10 reads | 11,574 | ~35,000 |
| URL shortener | 10M | 2 reads | 231 | ~700 |
| WhatsApp messages | 2B | 40 messages | 925,926 | ~3M |
| Google Search | 8B searches/day | — | 92,593 | ~300,000 |

### Read:Write Ratio

Most systems are read-heavy. The ratio drives caching decisions.

| System | Read:Write |
|:-------|:-----------|
| Twitter timeline | 100:1 |
| URL shortener | 100:1 |
| Photo sharing | 50:1 |
| Payment processing | 1:1 |
| Logging/metrics | 1:100 (write-heavy) |

---

## Storage Estimation

### Data Size Reference

| Data | Size |
|:-----|:-----|
| ASCII character | 1 byte |
| Unicode character | 2–4 bytes (UTF-8) |
| Integer (int32) | 4 bytes |
| Long (int64) | 8 bytes |
| UUID | 16 bytes |
| IPv4 address | 4 bytes |
| IPv6 address | 16 bytes |
| Timestamp | 8 bytes |
| URL (average) | 100 bytes |
| Tweet (text only) | ~200 bytes |
| Small image thumbnail | 200 KB |
| Full HD image | 3 MB |
| 1 minute of MP4 video (720p) | ~100 MB |

### Storage Formula

```
Daily storage = writes_per_day × size_per_record
Annual storage = daily_storage × 365
With replication: × replication_factor (typically 3)
```

**Example — Twitter-like service:**
```
New tweets per day:
  100M DAU × 5% who tweet × 1 tweet = 5M tweets/day

Per tweet storage:
  tweet_id: 8 bytes
  user_id: 8 bytes
  text: 200 bytes
  timestamp: 8 bytes
  metadata: 30 bytes
  Total: ~250 bytes per tweet

Daily storage: 5M × 250 bytes = 1.25 GB/day
Annual (text only): 1.25 × 365 = ~456 GB/year
With replication (3×): ~1.4 TB/year
```

**Media storage:**
```
Assume 10% of tweets have an image (500K/day)
Average image: 200 KB thumbnail + 3 MB original = 3.2 MB/tweet

Daily media storage: 500K × 3.2 MB = 1.6 TB/day
Annual media: 1.6 × 365 = ~584 TB/year
```

---

## Memory Estimation

### JVM Object Sizing

In Java, objects have overhead beyond the raw field sizes.

```
Object header:                  16 bytes (mark word + class pointer)
int field:                       4 bytes
long field:                      8 bytes
Object reference:                8 bytes (64-bit JVM)
String (empty):                 ~40 bytes
String "hello":                 ~80 bytes (40 + char array overhead)
ArrayList (empty):              ~48 bytes
ArrayList (100 Integer refs):  ~1,248 bytes (48 + 100×8 + 100×16)
```

**Rule of thumb:** Java objects consume 3–5× their raw data size due to headers, padding, and boxed types. A million "simple" Java objects with 3 int fields ≈ 50–80 MB.

### Redis Memory

Redis stores keys and values with overhead:

```
Redis overhead per key: ~64–100 bytes (hash map entry, sds string struct)
Redis String "hello": ~50 bytes key overhead + 5 bytes value = ~55 bytes
Redis Hash (small, <128 fields): stored as ziplist = compact, ~50 bytes overhead
```

**Rule of thumb for sizing Redis:** raw data size × 2–3 for typical string/hash data.

### How Much to Cache

```
Target: serve top 20% of data from cache (80/20 rule)
Cache size = 0.20 × daily_active_data_size

Or: cache size = peak_QPS × P99_response_size × cache_TTL_seconds
```

---

## Bandwidth Estimation

```
Inbound bandwidth = writes_per_second × avg_write_size
Outbound bandwidth = reads_per_second × avg_response_size

Outbound usually dominates for read-heavy systems.
```

**Example — video streaming service:**
```
DAU: 10M  
Average session: 30 min, streams at 5 Mbps (HD)

Total simultaneous viewers (peak):
  Assume 5% concurrent at peak = 500,000 users

Peak outbound bandwidth:
  500,000 × 5 Mbps = 2,500,000 Mbps = 2.5 Tbps

This is why Netflix uses CDN — no single origin can serve 2.5 Tbps.
```

---

## Worked Example: URL Shortener

This is the canonical "first" system design problem. Walk through it fully.

### Clarify Requirements

```
Functional:
- Given a long URL, return a short URL (e.g., bit.ly/abc123)
- Given a short URL, redirect to original URL
- Custom aliases optional

Non-functional:
- 100M shortened URLs created per day
- 10:1 read-to-write ratio (1B redirects/day)
- URLs stored for 5 years
- High availability on reads
- Latency: redirect < 100ms
```

### Step 1: QPS Estimation

```
Writes (new URLs):
  100M / 86,400 = ~1,157 writes/sec
  Peak writes: ~3,500 writes/sec

Reads (redirects):
  1B / 86,400 = ~11,574 reads/sec
  Peak reads: ~35,000 reads/sec
```

### Step 2: Storage Estimation

```
Per URL record:
  short_code:  7 bytes ("abc1234")
  long_url:  200 bytes (average)
  created_at:  8 bytes
  expires_at:  8 bytes
  user_id:     8 bytes
  Total: ~230 bytes

Per day:  100M × 230 bytes = 23 GB/day
5 years:  23 × 365 × 5 = ~42 TB (before replication)
With replication (3×): ~126 TB
```

### Step 3: Memory Estimation (Cache)

```
Cache top 20% of hot URLs:
  Daily active URLs ≈ 100M (assume all are active)
  Cache 20% = 20M URLs × 300 bytes ≈ 6 GB

Hot URLs cache: 6 GB Redis → fits on a single Redis node
```

### Step 4: Bandwidth Estimation

```
Inbound:  3,500 writes/sec × 300 bytes = ~1 MB/s
Outbound: 35,000 reads/sec × 200 bytes = ~7 MB/s
(Redirects are small: just the long URL in a 301/302 response)
```

### Step 5: Architecture Decisions from Numbers

```
✓ Reads dominate (10:1) → caching is critical
✓ 35K reads/sec → Redis cluster handles this easily (100K+/sec)
✓ 3,500 writes/sec → any SQL DB handles this
✓ 42 TB total → need a DB that handles this (MySQL with sharding, Cassandra)
✓ < 100ms redirect → cache hit in Redis (1-2ms) → 301 redirect, done
```

---

## Interview Tips

1. **Always state your assumptions.** "I'll assume 100M DAU and 1% post rate."
2. **Round aggressively.** `86,400 ≈ 100,000` is fine. Precision wastes time.
3. **Know the 5% concurrent rule.** At any given moment, ~5% of DAU are active simultaneously.
4. **Derive architecture from numbers.** Don't just compute — say what it means. "100K QPS means we need a cache; the database can't handle this cold."
5. **Know rough system capacities:**
   - Single MySQL: ~5,000 writes/sec, ~50,000 reads/sec
   - Redis single node: ~100,000 operations/sec
   - Kafka single partition: ~50 MB/s throughput
   - 1 Gbps NIC: ~125 MB/s

---

## References

- [Latency Numbers Every Programmer Should Know](https://gist.github.com/jboner/2841832) — Jonas Bonér
- *System Design Interview* — Alex Xu, Chapter 2 (Back-of-Envelope Estimation)
- [Google Jeff Dean's "Numbers Every Engineer Should Know"](http://highscalability.com/numbers-everyone-should-know)
