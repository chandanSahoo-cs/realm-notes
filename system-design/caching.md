# Caching: Architecture & Failure Modes

Caching stores copies of data in high-speed, volatile storage (typically RAM) to serve repetitive read operations orders of magnitude faster than querying persistent disks or re-executing expensive computations.

---

## Latency Hierarchy & Hit Ratio Mathematics

```text
Hardware Storage Latency:
L1/L2 CPU Cache   | 0.5 - 5 ns
System RAM        | ~100 ns         <-- In-Memory Cache (Redis, Caffeine)
NVMe SSD Storage  | 10 - 50 us
Rotational Disk   | 5 - 10 ms       <-- Relational Database Disk Storage
Cross-Region Net  | 50 - 150 ms
```

### The Leverage of the Cache Hit Ratio
The real-world performance of a cache is dictated by its **Effective Access Time (EAT)**:

```text
Effective Latency = (H * L_cache) + ((1 - H) * L_db)
```

Where `H` is the Cache Hit Ratio, `L_cache` is cache latency (~1 ms), and `L_db` is database query latency (~50 ms).

```text
Hit Ratio (H)     Effective Latency     Database Load
------------------------------------------------------
0% (No Cache)     50.0 ms               100% (Baseline)
80%               10.8 ms               20%
90%                5.9 ms               10%
99%                1.49 ms               1% (10x database load drop vs 90%!)
```

Improving your hit ratio from **90% to 99%** does not just speed up user responses—it reduces the read throughput hitting your database by an entire order of magnitude.

---

## Multi-Tier Caching Topologies

Scalable architectures distribute caching responsibilities across multiple layers:

```text
User Device  -->  Edge / CDN  -->  API Gateway / App  -->  Distributed Cache  -->  Primary DB
(Browser)       (Cloudflare)      (In-Process RAM)        (Redis Cluster)         (Postgres)
 < 1 ms           20 - 40 ms           < 1 us                 ~1 ms                 10 - 50 ms
```

| Cache Layer | Technology | State Shared? | Primary Use Case | Trade-offs |
|---|---|---|---|---|
| **Client / Mobile** | HTTP Cache, SQLite, Disk | No (Per Device) | Static assets, offline tokens | Lowest latency; hardest to forcibly invalidate |
| **CDN / Edge** | Cloudflare, Fastly, Akamai | Yes (Per Region) | Media assets, public GET endpoints | Shields origin; unsuitable for personalized data |
| **In-Process Cache** | Caffeine, Go `sync.Map` | No (Per Instance) | Config flags, permission sets, hot keys | Zero network overhead; creates intra-cluster inconsistency |
| **Distributed Cache**| Redis, Memcached, KeyDB | Yes (Cluster-wide) | User sessions, cart data, query caches | Shared state; incurs network serialization hop (~1 ms) |

> **Client-Side Cluster Topology Caching**
> Advanced client SDKs also cache infrastructure topology. For example, Redis Cluster clients maintain an in-memory map of which node owns each of the 16,384 hash slots. When a client issues a command, it routes directly to the correct shard. If resharding occurs, Redis returns a `-MOVED <slot> <ip>` response, prompting the client to refresh its local topology cache and retry transparently.

---

## Cache Access & Mutation Topologies

How your application interacts with the cache during reads and writes determines consistency, latency, and data safety:

```text
1. Cache-Aside (Lazy Loading) - Most Common
   Read Path:
   1. Check Cache. If HIT -> Return data.
   2. If MISS -> Query Database -> Store in Cache -> Return data.

   Write Path:
   1. Write changes directly to Database.
   2. INVALIDATE (delete) the corresponding key in Cache.
```

> **Why delete rather than update the cache on writes?**
> If two concurrent writes execute (Thread A writes 1, Thread B writes 2), network jitter can cause Thread A's cache update to arrive *after* Thread B's. This leaves the cache holding value 1 while the database holds value 2. Deleting the key avoids this race condition by forcing the next read to fetch the latest state from the database.

```text
2. Write-Through Caching
   App writes only to Cache -> Cache synchronously writes to Database -> ACK.
   - Pros: Reads never miss; data is immediately warm.
   - Cons: Higher write latency; populates cache with data that might never be read again.

3. Write-Behind (Write-Back) Caching
   App writes to Cache -> ACK immediately -> Cache asynchronously flushes to DB in batches.
   - Pros: Maximum write throughput; collapses multiple updates to the same key into one DB write.
   - Cons: RISK OF DATA LOSS. If the cache node crashes before dirty data flushes to disk,
     writes are lost forever. Ideal for analytics counters, telemetry, and view counts.

4. Read-Through Caching
   App reads exclusively from Cache. On a miss, the cache infrastructure itself fetches 
   from the backing store and populates itself before responding to the app.
   - Standard pattern for CDNs; less common in custom application layers.
```

---

## Eviction Policies & Memory Management

When a cache reaches its memory threshold, eviction algorithms determine which keys to drop:

- **Least Recently Used (LRU):** Tracks access recency using a doubly linked list combined with a hash map. Constant-time `O(1)` reads and evictions. Safest default for workloads where recently accessed items are likely to be accessed again.
- **Least Frequently Used (LFU):** Maintains an access frequency counter per key. Ideal for stable, long-term popular items (e.g., top-selling products). Vulnerable to "historical bias" where items that experienced a past traffic spike remain stuck in cache long after traffic subsides.
- **First In, First Out (FIFO):** Evicts the oldest inserted item based on creation time, completely ignoring access recency and frequency. Rarely used in production because it routinely evicts frequently accessed "hot" keys simply because they were populated earlier.
- **Time-To-Live (TTL):** Hard expiration timestamp. Redis implements this via a combination of passive checks (purging an expired key upon access) and active checks (periodically sampling random keys and evicting expired entries).

---

## The Four Classic Caching Failure Modes

```text
1. Cache Stampede (Thundering Herd)
   Problem: A viral product key with high concurrency expires. Thousands of simultaneous 
            requests miss the cache and hit the database at the exact same millisecond.
   Fix 1: Request Coalescing (Singleflight). Lock the key so only ONE worker executes the 
          database query while all other requests block and wait for the result.
   Fix 2: Probabilistic Early Expiration (XFetch algorithm). Background workers calculate 
          the likelihood of expiry and asynchronously warm the key slightly before it dies.

2. Cache Penetration
   Problem: Malicious or buggy clients query non-existent keys (e.g., GET /products/-99999).
            Every request misses the cache and directly queries the database.
   Fix 1: Bloom Filters. A probabilistic data structure at the cache ingress checks if a key
          could possibly exist before querying the database.
   Fix 2: Cache Empty Results. Store key = null with a short TTL (e.g., 60 seconds).

3. Cache Avalanche
   Problem: Hundreds of thousands of keys are written at once with identical 1-hour TTLs.
            Exactly 60 minutes later, all keys expire simultaneously, causing a massive DB spike.
   Fix: TTL Jitter. Add uniform random noise to expiration windows:
        TTL = base_duration + random(0, jitter_seconds)

4. Hot-Key Node Saturation
   Problem: A single viral SKU receives 200,000 requests per second, maximizing the CPU 
            bandwidth of the single Redis node owning that specific hash slot.
   Fix 1: Add a local in-process L1 cache (e.g., Caffeine) directly on the API gateway instances.
   Fix 2: Key Salting. Spread writes across multiple keys (product:101#0 through product:101#9)
          to distribute load across multiple Redis shards.
```

---

## Invalidation Architecture: Solving Dual-Write Inconsistencies

Having application code update the database and then call `cache.delete()` is inherently vulnerable to network failures or process crashes between the two operations.

A more resilient pattern uses **Change Data Capture (CDC)** for cache invalidation:

```text
Application  -->  Writes to Primary Database (Committed Transaction)
                          │
                     Database WAL
                          │
                  CDC Tailer (Debezium)
                          │ Streams committed row changes
                          ▼
                  Message Bus (Kafka)
                          │
                 Invalidation Service  -->  DEL cache:product:42
```

- **Guaranteed Consistency:** Invalidation events are emitted only after transactions have been committed to the database log.
- **Decoupled Architecture:** Application services are freed from maintaining distributed cache invalidation hooks.

---

## Redis Data Structures & Architectural Patterns

Redis is more than a simple string key-value store; it provides optimized in-memory data structures tailored for specific distributed patterns:

| Data Structure | Key Commands | Complexity | Production Use Case |
|---|---|---|---|
| **Strings** | `SET`, `GET`, `SET key val NX EX 60`, `MGET` | `O(1)` | Serialized JSON caching, distributed locking (`SET NX`), rate limiting counters (`INCR`). |
| **Lists** | `LPUSH`, `RPUSH`, `LPOP`, `RPOP`, `LLEN` | `O(1)` push/pop | FIFO Task Queues (`LPUSH` + `RPOP`) or LIFO Stacks (`LPUSH` + `LPOP`), recent activity feeds. |
| **Hashes** | `HSET`, `HGET`, `HGETALL`, `HINCRBY` | `O(1)` per field | Modeling entity objects (`user:101` -> fields: `name`, `email`); allows updating single fields without re-serializing whole JSON. |
| **Sets** | `SADD`, `SREM`, `SISMEMBER`, `SINTER` | `O(1)` membership | Unique item collections, tag associations, social mutual friend intersections (`SINTER`). |
| **Sorted Sets (ZSET)**| `ZADD`, `ZRANGEBYSCORE`, `ZREVRANK` | `O(log N)` | Gaming leaderboards, real-time rank tracking, sliding-window rate limiters (score = timestamp). |
| **Streams** | `XADD`, `XREADGROUP`, `XACK` | `O(1)` append | Distributed append-only event stream with consumer group semantics (lightweight Kafka alternative). |
| **Bitmaps / HyperLogLog**| `SETBIT`, `PFADD`, `PFCOUNT` | `O(1)` memory | Tracking daily active users (DAU) and cardinality estimation of unique visitors using ~12 KB RAM. |

### Redis Key Naming Best Practices
Use hierarchical namespacing delimited by colons:
```text
object_type:primary_id:attribute
- user:441:profile
- order:9921:items
- rate_limit:ip_203.0.113.45:minute
```

