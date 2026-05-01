---
layout: default
title: "2.2 Caching"
parent: "Phase 2: Core System Design"
nav_order: 2
---

# Caching
{: .no_toc }

<details open markdown="block">
  <summary>Table of Contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

Caching is the single most impactful tool for improving system performance. It exploits the **principle of locality**: recently accessed data is likely to be accessed again. A well-designed cache layer can reduce database load by 95%+ and cut P99 latency from hundreds of milliseconds to single digits.

---

## The Cache Hierarchy

Every layer of a system can cache. Understanding where to cache what is the architectural decision.

```
Browser Cache
    ↓ (miss)
CDN Edge Cache
    ↓ (miss)
API Gateway Cache
    ↓ (miss)
Application L1 Cache (in-process: Caffeine)
    ↓ (miss)
Application L2 Cache (distributed: Redis)
    ↓ (miss)
Database Query Cache (deprecated in MySQL 8.0+)
    ↓ (miss)
Database → Disk
```

| Layer | Technology | Latency | Scope |
|:------|:-----------|:--------|:------|
| Browser | HTTP Cache-Control headers | 0ms | Per user |
| CDN | Cloudflare, Fastly, CloudFront | 5–50ms | Global |
| API Gateway | Kong, AWS API Gateway | 1–5ms | All users |
| Application L1 | Caffeine (in-JVM) | 0.1–1ms | Per JVM instance |
| Application L2 | Redis, Memcached | 1–5ms | All instances |
| DB cache | Buffer pool (InnoDB) | 0.1ms | Per DB instance |

{: .note }
In interviews, when asked "how do you improve read performance?", walk through this hierarchy. Starting at the closest layer and working outward is the correct framing.

---

## Cache Strategies

### Cache-Aside (Lazy Loading)

The application is responsible for loading data into the cache on a miss. Most common pattern.

```
Read path:
  1. Check cache for key K
  2. Cache hit  → return value
  3. Cache miss → query DB → write result to cache → return value

Write path:
  1. Write to DB
  2. Invalidate (delete) the cache key
  (NOT update — avoids race conditions)
```

```java
@Service
public class UserService {
    @Autowired private UserRepository db;
    @Autowired private RedisTemplate<String, User> redis;
    
    public User getUser(Long userId) {
        String key = "user:" + userId;
        
        // 1. Check cache
        User cached = redis.opsForValue().get(key);
        if (cached != null) return cached;
        
        // 2. Cache miss: load from DB
        User user = db.findById(userId).orElseThrow();
        
        // 3. Populate cache (TTL: 10 minutes)
        redis.opsForValue().set(key, user, Duration.ofMinutes(10));
        
        return user;
    }
    
    public void updateUser(Long userId, User updated) {
        db.save(updated);
        // Invalidate, don't update — avoids race conditions
        redis.delete("user:" + userId);
    }
}
```

**Pros:**
- Cache only contains what's actually read (no pollution)
- Resilient to cache failure: fall back to DB
- Works for read-heavy workloads

**Cons:**
- **Cache miss penalty:** First request always hits DB (3 round trips)
- **Stale data:** Window between write and cache invalidation
- **Race condition risk:** Two concurrent cache-misses can both populate with stale data

### Write-Through

On every write to the DB, also write to the cache synchronously.

```
Write path:
  1. Write to cache (with key K)
  2. Write to DB
  Both succeed or neither succeeds (application logic)

Read path:
  1. Check cache — always a hit (if data exists)
```

```java
public void updateUser(Long userId, User updated) {
    // Write to both — cache is always in sync
    redis.opsForValue().set("user:" + userId, updated, Duration.ofMinutes(10));
    db.save(updated);
}
```

**Pros:** Cache is always consistent with DB. No stale reads.

**Cons:**
- **Cache churn:** Data written to cache may never be read (cold path writes pollute the cache)
- **Write latency:** Every write has the overhead of a cache write
- **Cold start:** Cache is empty on restart — no benefit until warmed up

**Use case:** Caching user preferences, settings, or reference data that is read frequently after every write.

### Write-Back (Write-Behind)

Write to cache first, then asynchronously flush to DB. Fastest writes.

```
Write path:
  1. Write to cache immediately (ack to client)
  2. Background process: flush dirty cache entries to DB

Read path:
  1. Check cache — reads the buffered (not yet persisted) value
```

**Pros:** Extremely fast writes. DB sees batched writes (better throughput).

**Cons:**
- **Data loss risk:** If cache crashes before flush, writes are lost
- **Complexity:** Need to implement flush queue and error handling

**Use case:** Shopping cart updates, counters (view counts, likes), time-series metrics ingestion.

### Write-Around

Write directly to DB, bypass cache. Cache is only populated on read miss.

```
Write path:  → DB only (cache not touched)
Read path:   → Cache miss → DB → populate cache → return
```

**Use case:** Write-once / read-rarely data (log records, audit trails). Avoids polluting cache with data that won't be re-read.

### Choosing the Right Strategy

| Pattern | Best For | Avoid When |
|:--------|:---------|:-----------|
| Cache-aside | General reads, diverse data access | Write-heavy with frequent reads after writes |
| Write-through | Frequent reads right after writes | Write-heavy workloads (writes without reads) |
| Write-back | High-throughput write-heavy | Cannot tolerate any data loss |
| Write-around | Infrequent re-reads | Frequently read data |

---

## Cache Eviction Policies

When a cache fills up, old entries must be evicted to make room. The eviction policy determines what gets removed.

### LRU (Least Recently Used)

Evict the entry that was **accessed least recently**. Assumption: recently accessed data will be needed again.

```
Access order: A B C D E (cache full)
Access A again: A B C D E → A moves to front
Add F (cache full): evict B (LRU)
Final: F A C D E
```

**Implementation:** Doubly linked list + HashMap. O(1) get and put.

**Good for:** Most workloads with temporal locality.  
**Bad for:** "Scan" workloads — iterating over large datasets evicts the working set.

### LFU (Least Frequently Used)

Evict the entry that was accessed the **fewest times**.

```
A accessed 5 times, B accessed 1 time, C accessed 3 times
Cache full, add D → evict B (LFU)
```

**Good for:** Stable working sets where popularity is a reliable predictor.  
**Bad for:** New items — they start with count 0 and are immediately vulnerable to eviction. A brand-new viral post gets evicted before an old post that was popular years ago.

### ARC (Adaptive Replacement Cache)

Maintains **four lists**: recently used (T1), frequently used (T2), ghost entries for T1 (B1), ghost entries for T2 (B2). Dynamically adjusts the balance between recency and frequency.

**Good for:** Workloads where access patterns shift over time.  
**Used by:** ZFS, IBM storage systems.

### W-TinyLFU (Window TinyLFU)

Used by **Caffeine** (the Java cache library). Combines a small LRU window cache with a main segmented LRU protected by a frequency filter (TinyLFU). Uses a Count-Min Sketch (see Phase 1) to approximate frequency with tiny memory overhead.

**Why it's better than LRU:** Resists scan pollution while still admitting new items based on access frequency. Caffeine consistently outperforms LRU in benchmarks by 15–40%.

### Redis Eviction Policies

Set in `redis.conf`:

```bash
maxmemory 4gb
maxmemory-policy allkeys-lru  # evict any key, LRU order

# Options:
# noeviction       - return error when full (default for caches that must not lose data)
# allkeys-lru      - evict any key, least recently used (most common for cache)
# allkeys-lfu      - evict any key, least frequently used
# volatile-lru     - evict only keys with TTL set, LRU order
# volatile-ttl     - evict keys with shortest TTL first
# allkeys-random   - evict random keys (rarely a good choice)
```

---

## Cache Pitfalls

### Cache Stampede (Thundering Herd)

When a popular cache entry expires, **hundreds of requests simultaneously miss** and all hit the database.

```
T=0:  Cache for "trending:posts" expires
T=1:  500 requests check cache → all miss
T=2:  500 requests all query DB simultaneously
T=3:  DB is overwhelmed, returns errors
T=4:  500 cache write attempts, only one value wins
```

**Solutions:**

**Mutex Lock (request coalescing):**
```java
// Only one thread rebuilds the cache; others wait
public String getTrending(String key) {
    String cached = redis.get(key);
    if (cached != null) return cached;
    
    String lockKey = "lock:" + key;
    try {
        boolean locked = redis.setIfAbsent(lockKey, "1", Duration.ofSeconds(5));
        if (locked) {
            String value = db.computeTrending();
            redis.set(key, value, Duration.ofMinutes(5));
            return value;
        } else {
            // Wait and retry — another thread is rebuilding
            Thread.sleep(100);
            return redis.get(key); // likely populated now
        }
    } finally {
        redis.delete(lockKey);
    }
}
```

**Probabilistic Early Expiration:**
```java
// Randomly start rebuilding BEFORE expiry, so stampede window narrows
// P(rebuild) increases as TTL approaches 0
double ttlFraction = (double) remainingTtl / originalTtl;
if (Math.random() > ttlFraction) {
    // Proactively refresh — only a few threads hit this
    refreshInBackground(key);
}
```

**Background refresh:**
A scheduled job refreshes popular keys before they expire, ensuring the cache is never empty for hot data.

### Cache Penetration

Requesting keys that **do not exist in the DB**. Every request bypasses cache and hits the database.

```
Attacker sends: GET /user/9999999999
→ Cache miss (doesn't exist)
→ DB query: no row found
→ Cache is NOT populated (nothing to cache)
→ Next request: same miss, same DB hit
```

**Solutions:**
- **Cache null values:** Store a sentinel (`NULL`) in cache with a short TTL (30s). Subsequent requests return the sentinel without hitting DB.
- **Bloom filter:** Before checking cache or DB, check a Bloom filter of valid IDs. Invalid IDs are rejected immediately.

```java
// Caching null values
public User getUser(Long userId) {
    String key = "user:" + userId;
    Object cached = redis.opsForValue().get(key);
    
    if (cached == NULL_SENTINEL) return null; // cached miss
    if (cached != null) return (User) cached;
    
    User user = db.findById(userId).orElse(null);
    if (user == null) {
        redis.opsForValue().set(key, NULL_SENTINEL, Duration.ofSeconds(30));
        return null;
    }
    
    redis.opsForValue().set(key, user, Duration.ofMinutes(10));
    return user;
}
```

### Cache Avalanche

**Many cache entries expire at the same time** — overwhelming the DB.

**Cause:** You populated the cache all at once (warm-up) with the same TTL. They all expire simultaneously.

**Solutions:**
- **Stagger TTLs:** Add random jitter: `TTL = base_ttl + random(0, base_ttl * 0.25)`
- **Persistent cache:** Use RDB/AOF in Redis so cache survives restarts
- **Circuit breaker on DB:** If DB is overwhelmed, return degraded responses rather than cascading failures

```java
// Staggered TTL
long ttlSeconds = 600 + (long)(Math.random() * 120); // 10-12 minutes
redis.opsForValue().set(key, value, Duration.ofSeconds(ttlSeconds));
```

### Hot Key / Hotspot

A single cache key receives extreme traffic — the Redis shard hosting it becomes the bottleneck.

**Example:** A viral tweet being read 1 million times/second — all traffic hits one Redis node.

**Solutions:**
- **Local cache + Redis:** Cache the hot key in L1 (in-JVM Caffeine) for 100ms. Most requests never leave the JVM.
- **Key splitting:** Replicate the hot value across N keys: `trending:post:shard:1`, `trending:post:shard:2`... Randomly pick one on read.
- **Read replicas:** Route hot key reads to Redis replicas.

---

## Redis Internals

### Data Structures

| Type | Use Case | Key Commands |
|:-----|:---------|:-------------|
| **String** | Simple cache, counters | `GET/SET`, `INCR`, `EXPIRE` |
| **Hash** | Object fields (user profile) | `HGET/HSET`, `HGETALL` |
| **List** | Queue, timeline (ordered) | `LPUSH/RPOP`, `LRANGE` |
| **Set** | Unique members (tags, friends) | `SADD`, `SISMEMBER`, `SUNION` |
| **Sorted Set** | Leaderboards, rate limiting | `ZADD`, `ZRANGEBYSCORE`, `ZRANK` |
| **HyperLogLog** | Cardinality estimation | `PFADD`, `PFCOUNT` |
| **Stream** | Append-only log, Kafka-like | `XADD`, `XREAD`, `XGROUP` |
| **Bitmap** | Feature flags, daily active users | `SETBIT`, `BITCOUNT` |

### Persistence: RDB vs AOF

```
RDB (Redis Database Snapshot):
  - Point-in-time snapshot of the entire dataset
  - Written to disk every N seconds or after M writes
  - Compact binary format, fast restart
  - Data loss: up to N seconds of writes
  
AOF (Append-Only File):
  - Every write command is appended to a log
  - fsync options: always (safe), everysec (default), no (fast)
  - Slower restart (replay all commands)
  - Data loss: at most 1 second (with everysec)
  
Both:
  - Use both for production (RDB for fast restarts, AOF for durability)
  - redis.conf: save "900 1" + appendonly yes
```

### Redis High Availability

**Redis Sentinel** — HA for a single primary with replicas:
- 3+ Sentinel nodes monitor the primary
- On primary failure, Sentinels vote, elect new primary, reconfigure replicas
- Clients connect to Sentinel first, discover current primary

**Redis Cluster** — horizontal sharding:
- Data split into **16,384 hash slots** across nodes
- Each node owns a range of slots
- Client hashes key to slot: `CRC16(key) % 16384`
- `CLUSTER KEYSLOT mykey` returns the slot number
- Each shard has 1 primary + 1-2 replicas
- Supports up to ~1,000 nodes

```bash
# Redis Cluster: hash tags for multi-key operations
# Keys {user:123}:profile and {user:123}:sessions map to same slot
# because hash tag is "user:123"
MGET {user:123}:profile {user:123}:sessions   # OK — same slot
MGET user:123:profile user:456:sessions       # ERROR — different slots
```

### Redis vs Memcached

| | Redis | Memcached |
|:-|:------|:---------|
| Data types | 8+ types | Strings only |
| Persistence | RDB + AOF | None |
| Replication | Yes | Third-party only |
| Pub/Sub | Yes | No |
| Scripting | Lua | No |
| Memory efficiency | Slightly lower | Slightly higher |
| Clustering | Native cluster | Client-side sharding |
| **Choose when** | Rich data types, persistence, pub/sub needed | Simple string cache, maximum memory efficiency |

**Default choice today is Redis.** Memcached is only worth considering when memory efficiency is critical and you don't need any Redis features.

---

## Local Cache (Caffeine)

Caffeine is the state-of-the-art in-process Java cache. It replaced Guava's cache and is used internally by Spring Boot's caching abstraction.

### Why Local Cache (L1)?

- **Redis latency:** Even a fast Redis call is 1–3ms. An L1 cache hit is 0.01–0.1ms (100× faster).
- **Hot keys:** Extremely popular data can be served from JVM heap without network I/O.
- **Resiliency:** If Redis is down, L1 continues to serve recently cached data.

### Caffeine Configuration

```java
// Spring Boot: enable Caffeine as L1 cache
// pom.xml: spring-boot-starter-cache + caffeine

@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(
            Caffeine.newBuilder()
                .maximumSize(10_000)            // max entries
                .expireAfterWrite(5, MINUTES)   // TTL after write
                .expireAfterAccess(2, MINUTES)  // TTL after last access
                .recordStats()                  // expose hit/miss metrics
        );
        return manager;
    }
}

// Use declaratively
@Service
public class ProductService {
    @Cacheable(value = "products", key = "#productId")
    public Product getProduct(Long productId) {
        return db.findById(productId).orElseThrow(); // only called on cache miss
    }
    
    @CacheEvict(value = "products", key = "#product.id")
    public void updateProduct(Product product) {
        db.save(product);
    }
}
```

### Two-Tier Caching: L1 (Caffeine) + L2 (Redis)

```
Request → L1 (Caffeine, in-JVM)
           ↓ miss
         L2 (Redis, distributed)
           ↓ miss
         Database
```

```java
public Product getProduct(Long productId) {
    String key = "product:" + productId;
    
    // L1: in-process Caffeine
    Product l1 = caffeineCache.getIfPresent(key);
    if (l1 != null) return l1;
    
    // L2: Redis
    Product l2 = redis.opsForValue().get(key);
    if (l2 != null) {
        caffeineCache.put(key, l2); // backfill L1
        return l2;
    }
    
    // DB
    Product product = db.findById(productId).orElseThrow();
    redis.opsForValue().set(key, product, Duration.ofMinutes(30));
    caffeineCache.put(key, product);
    return product;
}
```

**Limitation:** L1 caches are per-JVM. When you update a product in one instance, other instances' L1 caches still hold the stale value. Use short TTLs for L1 (30–60 seconds) or invalidate via pub/sub.

---

## Key Takeaways for Interviews

1. **Cache-aside is the default pattern.** Write-through for write-then-immediately-read; write-back only when you can tolerate data loss.
2. **LRU is the default eviction policy** (`allkeys-lru` in Redis). W-TinyLFU (Caffeine) is better but requires a library.
3. **Thunder Herd has three fixes:** mutex, probabilistic early expiry, background refresh. Know all three.
4. **Cache penetration is an attack surface.** Cache null values or use a Bloom filter.
5. **Hot keys don't scale on Redis alone.** Add an in-process L1 Caffeine cache for ultra-hot keys.
6. **Redis Cluster uses 16,384 hash slots.** Use hash tags `{key}` to keep related keys on the same slot.

---

## References

- [Redis documentation](https://redis.io/docs/)
- [Caffeine GitHub wiki](https://github.com/ben-manes/caffeine/wiki)
- [TinyLFU paper](https://dl.acm.org/doi/10.1145/3149371) — Einziger et al.
- *Designing Data-Intensive Applications* — Chapter 5 (Replication) and Chapter 11 (Stream Processing)
- [Understanding ARC](https://www.usenix.org/conference/fast-03/arc-self-tuning-low-overhead-replacement-cache)
