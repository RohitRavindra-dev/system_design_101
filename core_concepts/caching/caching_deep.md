# Caching for System Design Interviews (HLD Revision)

> Based heavily on “Caching in System Design Interviews w/ Meta Staff Engineer (Hello Interview)” plus supporting material.[page:1][web:32]

---

## How to Use This Sheet in Interviews

- Use this as a **deep-dive module** when you hit scaling/latency during your HLD.  
- For each system, articulate:
  - **Why** you need a cache (numbers, bottleneck).
  - **Where** you put it (layer, product choice).
  - **How** reads and writes flow (pattern).
  - **What** can go wrong (stampede / hot key / penetration / avalanche).
  - **How** you keep it consistent enough (TTL vs invalidation vs write-through).  

**Senior signal:** Never say “I’ll add Redis” without: approximate QPS, expected hit rate, and which failure mode worries you most (and how you’d mitigate).  

---

## 1. Caching Basics

A cache is a **temporary storage layer** that keeps recently or frequently accessed data closer to the CPU or user so that future reads are faster than going to the system of record (DB, object store, service).[page:1]  

In typical backend systems, reading from SSD-backed DBs is around 1 ms, while reading from RAM is on the order of 100 ns, roughly a 10,000× difference in latency.[page:1]  

Caching trades:
- **Extra memory + complexity**  
- For **lower latency + reduced load** on downstream systems (DB, microservices, storage).[page:1][web:13]  

### Library analogy

- DB = **central library archive in the basement** (complete but slow).  
- Cache = **“featured shelf” near the entrance** with the most popular books; you walk less, so you serve more patrons per minute.[page:1]  

### Kitchen analogy

- DB = **pantry** in another room.  
- Cache = **countertop / fridge**: items you touch many times per day live here.  
- Eviction = when the fridge is full, you throw out or move the least useful/oldest stuff.  

**Senior signal:** Always anchor caching to **latency SLOs** (“we need P99 ≤ 100 ms”) and **DB CPU/QPS constraints** (“DB tops out around 50k QPS; traffic model says 200k QPS; we need a cache for read scaling”).[page:1][web:11]  

---

## 2. Caching Taxonomy: Layers in the Path

High-level flow:  

`Client (browser/app) → Client cache → CDN → Load balancer / reverse proxy cache → Application cache (local + distributed) → Database / storage`[page:1][web:18][web:20]  

### 2.1 Client-side caching (Browser / Mobile)

- **Browser HTTP cache**: keyed by URL + headers, controlled via `Cache-Control`, `ETag`, `Last-Modified` etc.[web:17][web:21][web:25]  
- **Local storage / IndexedDB / on-device DB**: apps persist data for offline reads or fast reopens (e.g., Strava caching runs locally, then syncing later).[page:1]  

Pros:
- Lowest latency: no network hop if cache hits.[page:1][web:21]  
- Can significantly reduce backend QPS if resources are cacheable with long TTLs.[web:21][web:25]  

Cons:
- **Harder to control**; data goes stale unless you use validation (ETag/Last-Modified) or short TTLs.[web:17][web:25]  
- Mostly relevant in interviews when you mention **offline mode** or **client-heavy apps** (maps, fitness apps).  

**Senior signal:** Mention **HTTP caching headers** (strong vs weak validators) and when you’d prefer `Cache-Control: max-age` vs `ETag`-based revalidation for static assets vs APIs.[web:17][web:27]  

---

### 2.2 CDN / Edge cache

A CDN is a geographically distributed network of edge servers that cache content close to users, primarily to reduce **network latency**, not just disk vs memory latency.[page:1]  

Typical use:
- Static media: images, videos, CSS/JS, downloads.[page:1]  
- Increasingly: **public APIs and HTML** for anonymous traffic.[page:1][web:18]  

Pros:
- Reduces round trip from hundreds of ms to a few tens of ms for distant users.[page:1][web:18]  
- Offloads huge volumes from origin web/app servers and storage.[page:1][web:18]  

Cons:
- Harder invalidation; often rely on **URL versioning** or purge APIs.  
- Need to deal with **cache keys** (path, query params, cookies) very carefully.[web:18][web:27]  

**In interviews:**  
- Mention CDN caching for **global user bases** hitting media or static assets and tie it explicitly to **latency** and **egress cost reduction**.[page:1][web:18]  

---

### 2.3 Load balancer / reverse-proxy cache

Many L7 load balancers or reverse proxies (NGINX, Varnish, Envoy, cloud LBs) can cache responses and serve them directly, reducing hits to app servers.[web:16][web:18][web:22]  

Examples:
- NGINX `proxy_cache`: caches static assets or selected API responses, supports TTL, stale-while-revalidate, etc.[web:18]  
- Varnish cache in front of origin servers with custom VCL rules (strip cookies, normalize URLs, control TTL).[web:18]  

Pros:
- Central point where you can apply caching logic for many backends.[web:18]  
- Shortens path: LB can serve from disk/memory instead of hitting app tier.  

Cons:
- Extra moving part and configuration complexity (cache keys, bypass rules, purge).[web:18][web:22]  
- For personalized / authenticated traffic, often disabled or heavily constrained.[web:18]  

**Senior signal:** When user traffic is **read-heavy and mostly anonymous** (e.g., landing pages, blogs), propose **reverse-proxy caching** in front of app servers before adding a heavy DB cache.  

---

### 2.4 Application-level caching

#### 2.4.1 External distributed cache (typical Redis / Memcached)

**External cache** = network service (e.g., Redis, Memcached) shared by multiple app servers, often the primary thing interviewers expect you to mention.[page:1][web:2][web:5][web:11]  

Pros:
- **Shared global view**: multiple app servers reuse the same cached values.[page:1]  
- Easily sharded or clustered for capacity and throughput.[web:2][web:14]  
- Tooling/ecosystem (monitoring, eviction policies, clustering).  

Cons:
- Network hop cost (still much lower than DB but non-zero).[page:1]  
- Extra infra: high availability, failover, cluster resizing, security.[web:2][web:14]  

**Redis vs Memcached quick distinctions (for interviews):**

- **Redis**: rich data structures (lists, hashes, sets, sorted sets, streams), persistence options, replication and Redis Cluster, pub/sub, scripting; often used beyond pure caching (sessions, rate limiting, leaderboards, queues).[web:2][web:5][web:14]  
- **Memcached**: simple, multi-threaded, in-memory key–value store with strings only; no persistence or built-in replication; great as a **simple, very fast cache-only layer**.[web:2][web:5][web:8][web:11]  

**Senior signal:**  
- Say: “For simple key–value caching in front of a DB with high read QPS and no need for advanced data structures, Memcached is fine; for richer features (sorted sets for leaderboards, TTL per hash field, persistence), I’d use Redis/Valkey.”[web:2][web:5][web:14]  

#### 2.4.2 In-process / local cache (L1)

In-process caching uses **app server memory** (e.g., in-JVM cache, in-process LRU map) to avoid even the network hop to Redis.[page:1]  

Pros:
- Fastest possible: memory lookup within same process; no network.[page:1]  
- Useful for tiny, super-hot data: config flags, feature toggles, small lookup tables needed on every request.[page:1]  

Cons:
- Each node has its own view; no global synchronization, leading to **inconsistencies** between app instances and wasted memory due to duplicated entries.[page:1]  
- Difficult invalidation across many app instances.  

**Senior signal:** Use this as an **L1 cache** in front of Redis (L2). You can explicitly say “I’d use a two-level cache: in-process LRU for ultra-hot keys, backed by a Redis cluster as L2.”[page:1][web:13]  

---

### 2.5 Database-level caching techniques

Even at the DB/storage layer, there are forms of caching:
- DB buffer cache / page cache: managed by the DB (e.g., Postgres, MySQL) – usually not something you design but you should acknowledge it exists.  
- Query result caches inside some DBs or application frameworks.[web:13]  

**Senior signal:** Show awareness: “First I’d try schema/indexes and ensure DB buffer cache is effective; only when DB is still bottlenecked do I place an app-level cache in front.”  

---

## 3. Read / Write Caching Strategies

Below are the classic patterns interviewer expect (names not required, **behavior is**).[page:1][web:32][web:41]  

### 3.1 Overview table: latency vs reliability

| Strategy        | Read path (simplified)                                  | Write path (simplified)                                   | Latency profile                             | Reliability / consistency profile                                                    | Typical use cases |
|----------------|----------------------------------------------------------|-----------------------------------------------------------|---------------------------------------------|--------------------------------------------------------------------------------------|-------------------|
| **Cache-aside**| App ⇒ cache; on miss ⇒ DB ⇒ cache ⇒ app                 | App ⇒ DB; optionally invalidate or update cache           | Reads: low after warmup; writes: normal DB  | Eventual consistency; short stale windows; simple mental model                       | Default for app caches |
| **Read-through**| App ⇒ cache; cache auto-fills from DB on miss          | Usually paired with write-through or DB writes + invalidation | Similar to cache-aside after warmup        | Similar to cache-aside; centralizes logic in cache component                         | CDNs, managed caching libs |
| **Write-through**| Reads like cache-aside / read-through                  | App ⇒ cache ⇒ (sync) DB ⇒ success                        | Writes: higher latency (must update both)   | Stronger read consistency (cache and DB in sync); risk of dual-write failures        | Financial data, critical configs |
| **Write-around**| App reads from cache+DB; cache filled only on read     | App ⇒ DB; does **not** update cache                      | Writes: low; first reads: higher latency    | Eventual consistency; cache can temporarily serve old value unless invalidated       | Write-heavy, rarely re-read data |
| **Write-back / write-behind**| Reads from cache; DB catches up later      | App ⇒ cache; cache ⇒ (async batch) DB                    | Writes: very low latency, high throughput   | Eventual consistency; risk of data loss on cache crash; complexity in failure cases  | Metrics/analytics, logs, non-critical data |

[page:1][web:31][web:32][web:33][web:34][web:35][web:41]  

---

### 3.2 Cache-aside (lazy loading) – your default

**Idea:** Application owns logic. On read, check cache; on miss, read DB, then populate cache. Writes go directly to DB and optionally invalidate cache.[page:1][web:41]  

Pseudo-logic:

```pseudo
function getUserProfile(userId):
    key = "user:" + userId
    value = cache.get(key)
    if value != null:
        return value   # cache hit

    # miss → DB
    row = db.query("SELECT ... FROM users WHERE id = ?", userId)
    if row == null:
        return null    # or 404

    cache.set(key, row, ttl = 5 minutes)
    return row
```

```pseudo
function updateUserProfile(userId, newData):
    db.update("UPDATE users SET ... WHERE id = ?", newData, userId)
    cache.delete("user:" + userId)  # invalidate on write
```

Pros:
- Simple to implement; works with any cache (Redis/Memcached).[page:1][web:42]  
- Cache only stores **hot data actually requested**, so memory usage is efficient.[page:1][web:42]  

Cons:
- First request after key eviction hits DB (cold-start / warmup cost).[page:1][web:42]  
- Stale cache if you forget to invalidate or if invalidation races with reads.  

**Senior signal:**  
- Mention **idempotency**: repeated misses for same key during a stampede can overload DB → leads into stampede mitigation (mutex, coalescing).[page:1][web:13]  

---

### 3.3 Read-through

**Idea:** Cache acts like a **proxy**; app only talks to cache, which on a miss fetches from DB and stores result.[page:1][web:32][web:41]  

This is basically how **CDNs** behave: on miss, they fetch from origin, cache, and return.[page:1][web:18]  

Pros:
- Centralizes logic; app is simpler (DB access embedded in cache layer).  
- Easy to reuse for multiple services.[web:32]  

Cons:
- Requires a cache implementation or framework that knows how to load from DB, not just dumb key–value.[web:32]  
- Harder to debug sometimes since DB access is “hidden” behind cache layer.  

**Interview usage:**  
- Mention for CDNs, edge caches, or when your platform team provides a **read-through caching library**. Otherwise fall back to cache-aside.  

---

### 3.4 Write-through

**Idea:** On mutating writes, application writes to cache, and cache synchronously writes to DB; only after **both** succeed is the write acknowledged.[page:1][web:32][web:41][web:45]  

Pros:
- Stronger read consistency: cache is always up to date with DB (no stale reads from cache as long as writes succeed).[web:32][web:41]  
- High hit ratio because all written data ends up in cache.[web:32][web:45]  

Cons:
- Higher write latency: every write hits both cache and DB.[page:1][web:32][web:41]  
- **Dual-write problem**: cache write succeeds, DB write fails (or vice versa) → inconsistency unless you build robust retry and compensation logic.[page:1][web:32]  
- Cache pollution: data that is never read still occupies cache memory.[page:1][web:32][web:45]  

**Senior signal:** Explicitly mention **dual-write risk** and that distributed transactions are often overkill; you typically accept small windows of inconsistency or wrap writes in **idempotent retry pipelines** with DLQs.[page:1][web:3][web:32]  

---

### 3.5 Write-back / write-behind

**Idea:** Application writes only to cache; cache asynchronously flushes data to DB later (batch or periodic).[page:1][web:32][web:34][web:41][web:42]  

Pros:
- Very fast writes; good for write-heavy workloads.[page:1][web:32][web:34][web:41]  
- DB load is smoothed via batching/coalescing (e.g., N writes compressed into one update).[web:34]  

Cons:
- Data loss risk: if cache fails before flush, writes are lost.[page:1][web:31][web:34][web:41]  
- DB may lag behind cache; you now have **cache as primary** for some period → tricky for other systems that read directly from DB.[web:34][web:41]  
- More complex failure handling, retries, ordering guarantees.  

Typical use cases:
- Analytics counters, metrics, logging, click-stream data, places where losing a small percentage of events is acceptable.[page:1][web:34][web:41]  

**Senior signal:** Clearly say: “I would not use write-back for user balances or orders; only for workloads where **eventual consistency and some data loss are acceptable**.”[page:1][web:34][web:41]  

---

### 3.6 Write-around

**Idea:** Writes go directly to DB; cache is only updated on subsequent reads (often via cache-aside), avoiding polluting cache with write-only data.[web:31][web:33][web:35][web:41]  

Pros:
- Cache stays focused on **read-heavy hot data**; write-only or rarely-read data doesn’t evict useful entries.[web:35][web:41]  
- Write latency is lower than write-through since you avoid cache write overhead.[web:33][web:41]  

Cons:
- First read after a write always misses cache, causing extra DB latency.[web:31][web:35][web:41]  
- If you forget to invalidate, cache can serve stale data (same as cache-aside).  

**Senior signal:** Use write-around in discussion when you have **huge volumes of data being written (like logs, time-series)** where only a small subset is read, and you want to keep the cache focused on hot read paths.[web:35][web:41]  

---

## 4. Eviction & Expiry Policies

Memory is finite; you can’t cache everything. Eviction policies decide **which keys to remove** when the cache is full; expiry defines **how long** a key lives.[page:1][web:13]  

### 4.1 LRU, LFU, FIFO

**LRU (Least Recently Used)**  
- Evicts the item that hasn’t been accessed for the longest time.[page:1]  
- Good default for general workloads: recently used data likely to be used again soon (temporal locality).[page:1][web:13]  

**LFU (Least Frequently Used)**  
- Evicts item with lowest access count; useful when a few keys are consistently hot.[page:1][web:13]  
- Works well for **highly skewed traffic** (Zipf-like distribution – top N keys dominate).[page:1][web:13]  

**FIFO (First In, First Out)**  
- Evicts oldest item regardless of usage.[page:1]  
- Simple but rarely ideal; can evict hot items that arrived early.[page:1]  

**In Redis / Memcached:** you can choose policies like `allkeys-lru`, `allkeys-lfu`, `volatile-lru`, etc., to apply LRU/LFU across all or TTL-only keys.[web:14]  

**Senior signal:** For a **hot feed or trending list**, mention LFU plus TTL; for more uniform workloads, LRU is fine as default.  

---

### 4.2 TTL-based expiry

**TTL (Time To Live):** Each key has an expiry timestamp; when it expires, cache evicts it or treats it as invalid.[page:1][web:13]  

Pros:
- Easy to reason about freshness (“user feed is at most 60 seconds stale”).[page:1][web:13]  
- Provides a safety net against **permanently stale** entries if invalidation breaks.[web:6][web:13]  

Cons:
- TTL expirations can cause **stampedes** when many clients request the same key right after expiry.[page:1][web:1][web:13]  
- Hard to pick optimal TTL – too short → low hit rate; too long → stale data.[web:6][web:13]  

**Senior signal:** Combine TTL with **event-driven invalidation**: “We invalidate keys on writes via pub/sub, and TTL acts as a backstop in case invalidation messages are delayed or dropped.”[web:3][web:6][web:15]  

---

## 5. Three “Hard” Failure Modes

In practice, caching creates its own failure zoo. Common categories:[page:1][web:7][web:10][web:13]  

1. **Cache penetration** – requests for data that doesn’t exist anywhere.  
2. **Cache breakdown / stampede** – one hot key expires and many requests rebuild it.  
3. **Cache avalanche** – many keys expire or cache cluster fails at once.  

### 5.1 Cache penetration (non-existent keys)

**Problem:** Requests for keys that are not present in cache **or** DB. Every request misses and goes to DB, which returns nothing and doesn’t cache the null value, so the next request repeats the pattern.[web:7][web:10][web:13]  

Typical causes:
- Malicious scans of random IDs.  
- Bugs requesting invalid IDs (e.g., negative or nonsense IDs).  

**Mitigations:**

1. **Input validation / sanity checks**  
   - Drop obviously invalid IDs at API layer (negative IDs, malformed UUIDs, out-of-range values).[web:7]  

2. **Cache negative results**  
   - When DB returns “no row”, cache a **null marker** (e.g., special value) with short TTL (30–60s).[web:7][web:13]  
   - Subsequent requests hit cache and never reach DB during that TTL.[web:7][web:13]  

3. **Bloom filter in front of cache**  
   - Maintain Bloom filter of all valid IDs; if filter says “definitely not present”, reject or short-circuit before hitting cache/DB.[web:4][web:7]  
   - Bloom filter is memory efficient, trades space for small false positive rate.[web:4][web:7]  

Pseudo-logic with Bloom filter:

```pseudo
function getProduct(productId):
    if !bloomFilter.mightContain(productId):
        return null   # short-circuit → avoid DB entirely

    value = cache.get("product:" + productId)
    if value != null:
        return value

    row = db.query("SELECT ... FROM products WHERE id = ?", productId)
    if row == null:
        cache.set("product:" + productId, NULL_MARKER, ttl = 30s)
        return null

    cache.set("product:" + productId, row, ttl = 5 minutes)
    return row
```

**Senior signal:** When discussing penetration, explicitly call out **Bloom filters vs negative caching**, and mention the **false positive trade-off** for Bloom filters but “never false negatives”.[web:4][web:7]  

---

### 5.2 Cache breakdown / stampede (hot key expiry)

This is exactly the “thundering herd” scenario described in the video.[page:1][web:1][web:10][web:13]  

**Problem:** A hot key (e.g., homepage feed) expires. Suddenly N concurrent requests all see a miss and issue expensive DB queries to rebuild the key, overwhelming DB.[page:1][web:1][web:10][web:13]  

Mitigation strategies:

1. **Per-key mutex / request coalescing**  
   - Only **one** request is allowed to rebuild the key; others wait and then read from cache.  

```pseudo
function getHomeFeed(userId):
    key = "home_feed:" + userId
    value = cache.get(key)
    if value != null:
        return value

    # Try to acquire per-key lock
    if !lock.acquire(key, ttl = 5s):
        # Another worker is rebuilding – wait and retry
        sleep(50ms)
        return cache.get(key)  # may still be null; handle fallback

    try:
        # Double-check after acquiring lock
        value = cache.get(key)
        if value != null:
            return value

        value = computeHomeFeed(userId)   # expensive
        cache.set(key, value, ttl = 60s)
        return value
    finally:
        lock.release(key)
```

[page:1][web:13]  

2. **Cache warming / refresh-ahead**  
   - Background job refreshes hot keys **before** TTL expiry (e.g., at 55s of a 60s TTL), so they never all expire under load.[page:1][web:1][web:4][web:13]  

3. **Stale-while-revalidate**  
   - Serve stale value if TTL is expired but refresh in the background; users get slightly old but fast data instead of hitting DB thundering herd.[web:4][web:13][web:18]  

**Senior signal:** Mention **double-checked locking pattern** plus **server-side metrics** like lock contention, rebuild latency, and DB QPS spikes during hot-key updates.  

---

### 5.3 Cache avalanche (mass expiry or cache failure)

**Problem:** Many keys expire at roughly the same time (e.g., you set TTL=1 hour for millions of keys, or the cache cluster restarts), causing almost all requests to miss and slam the DB simultaneously.[web:1][web:4][web:7][web:10][web:13]  

Mitigations:

1. **TTL jitter / randomization**  
   - Instead of TTL = 3600s for all keys, use `TTL = 3600 + random(0, 300)`; expirations are spread over 5 minutes instead of synchronized.[web:1][web:4][web:13]  

2. **Staggered cache population / warmup**  
   - During deployment or at startup, pre-warm hot keys gradually to avoid single big spike.[web:1][web:4]  

3. **High-availability cache cluster**  
   - Use Redis Cluster or equivalent with replication; avoid entire cluster going down as a SPoF.[web:4][web:7][web:14]  

4. **Graceful degradation & circuit breakers**  
   - If DB is overloaded, serve degraded responses (partial data, older snapshots) or shed low-priority traffic until cache recovers.[web:7]  

**Senior signal:** Explicitly differentiate **stampede (one key)** vs **avalanche (many keys)** in your explanation.  

---

## 6. Consistency & Cache Invalidation

Famous quote: “There are only two hard problems in CS: cache invalidation and naming things.”[page:1][web:3][web:6]  

### 6.1 Inconsistency example (profile picture)

Scenario from the video:  
- User updates profile picture; DB now has new image, but cache still has old image; readers hit stale cache.[page:1]  

Causes:
- Reads go to cache but writes go to DB.  
- Cache is not invalidated or updated on write, or invalidation races with concurrent reads.[page:1][web:3]  

### 6.2 Strategies for keeping cache “fresh enough”

1. **Invalidate on write (cache-aside variant)**  
   - After DB write, delete or update corresponding cache key(s).[page:1][web:6][web:15]  

2. **Short TTLs**  
   - If approximate freshness is acceptable, keep TTL small (e.g., 60s for feeds, 5–15 min for profile details).[page:1][web:6][web:13]  

3. **Event-driven invalidation / pub-sub**  
   - Write to DB → emit change event → subscribers invalidate or update cache across all nodes/clusters.[web:3][web:6][web:15]  

4. **Write-through / write-back**  
   - For some critical subsets, you may use write-through so cache and DB are updated together at write-time.[page:1][web:32][web:41]  

**Senior signal:**  
- Mention **ordering issues**: invalidation must usually arrive **before or at most concurrently** with cache fill responses to avoid re-caching stale values.[web:3][web:6]  
- Reference that large systems (e.g., Meta’s TAO) optimize **invalidation latency** relative to DB replication lag to keep caches consistent.[web:3][web:6]  

---

### 6.3 Strong vs eventual consistency in caches

| Aspect              | Strong consistency via caching (e.g., write-through) | Eventual consistency via TTL/invalidation |
|---------------------|------------------------------------------------------|-------------------------------------------|
| Reads               | See latest committed write                          | May see stale data briefly                |
| Latency             | Higher (synchronous writes / invalidations)         | Lower; async invalidation / TTL           |
| Availability        | Lower during failures; may reject writes            | Higher; system can serve slightly stale   |
| Typical use cases   | Financial balances, inventory counts, critical configs | Social feeds, counters, analytics, recommendation lists |

[web:6][web:9][web:12]  

**Interview framing:**  
- “For user balances, I would either **not cache** or use write-through with small, strongly-consistent subset and accept slower writes.”[web:6][web:9]  
- “For feeds and metrics, I accept **eventual consistency** with bounded staleness via TTL and asynchronous invalidation.”[page:1][web:6][web:9][web:13]  

---

## 7. Operational Scaling: Redis Cluster, Memcached & Consistent Hashing

### 7.1 Redis vs Memcached at scale

- **Redis Cluster** uses **hash slots (0–16383)** and maps slots to nodes; client library knows which node to hit, and cluster can rebalance slots when nodes are added/removed.[web:2][web:14]  
- **Memcached** typically relies on **client-side consistent hashing** to map keys to nodes; adding/removing a node changes the ring, but consistent hashing reduces the fraction of keys that move.[web:2][web:8][web:11]  

Pros of Redis Cluster:
- Built-in sharding and replication; many client libraries support transparent routing.[web:2][web:14]  
- Advanced features: persistence, scripts, streams, sorted sets, etc. still available in cluster mode.[web:2][web:14]  

Pros of Memcached client sharding:
- Very simple architecture (no cluster logic on server); each node is independent.[web:2][web:8]  
- High throughput for simple key–value workloads, multi-threaded from day one.[web:2][web:11]  

**Senior signal:**  
- Say: “At ~hundreds of GB of cached data, I’d use Redis Cluster with hash slots or a Memcached pool with consistent hashing; re-sharding impacts hit rate, so we scale cautiously and may temporarily overprovision.”[web:2][web:8][web:11][web:14]  

---

### 7.2 Consistent hashing basics (for cache clusters)

**Problem:** If you naively map keys to `hash(key) % N`, adding/removing a node changes N and remaps **almost all keys**, destroying cache warmness.[web:13][web:41]  

**Consistent hashing solution:**
- Arrange nodes on a hash ring.  
- Map each key to position `hash(key)` on ring; walk clockwise to find owner node.  
- When adding/removing a node, only keys that fall between ring segments around that node move, not all keys.[web:13][web:41]  

At scale:
- Use **virtual nodes** (many hash points per physical node) to spread load evenly and avoid hotspots.[web:13][web:41]  

**Senior signal:** When asked about “adding cache nodes”, mention **temporary hit-rate drop** and mitigating strategies:
- Gradual redistribution (migrate slots/vnodes in small batches).  
- Overlapping warmup: new nodes pre-fetch some hot data before they start taking full traffic.[web:2][web:14]  

---

## 8. Guided Interview Walkthrough: Example Narrative

Imagine designing **Twitter-like feed service** with 100M DAUs and strict latency requirements.[page:1][web:13]  

### 8.1 When to introduce caching

- You estimate: 100M DAUs × 20 feed fetches/day ≈ 2B feed reads/day (~23k QPS average, much higher at peak).[page:1]  
- Computing feed is expensive (joins across posts, follow graph, likes), hitting DB directly per request is too slow and costly.[page:1][web:13]  

So you say:

> “To meet our P99 latency target of 200 ms and avoid overwhelming the DB, I’ll cache each user’s precomputed feed in Redis with TTL ~60 seconds.”[page:1][web:13]  

### 8.2 How you describe it (end-to-end)

1. **Where to cache**  
   - CDN for static assets and public timelines (if any).  
   - Redis Cluster as external cache in front of feed DB.  
   - Optional in-process L1 cache in app for extremely hot celebrity feeds.[page:1][web:13][web:14]  

2. **Read strategy**  
   - Cache-aside: app checks Redis `feed:{userId}`; if hit, return; if miss, compute feed from DB/graph, write back to Redis with TTL 60s.[page:1][web:42]  

3. **Write strategy**  
   - On new post: write to posts DB; update fan-out pipelines; optionally invalidate or update feed cache for followers or let it be stale until TTL expires (depending on consistency needs).[page:1][web:6][web:13]  

4. **Eviction & expiry**  
   - Use LRU + TTL 60s; high-churn feeds will stay hot naturally; long-tail inactive users will eventually be evicted.[page:1][web:13][web:14]  

5. **Failure modes & mitigations**  
   - Stampede on popular feed: per-key distributed lock + refresh-ahead worker for top N feeds.[page:1][web:1][web:13]  
   - Hot key (Taylor Swift): replicate her feed on multiple Redis nodes or L1 caches in each app instance.[page:1][web:13]  
   - Avalanche: add TTL jitter ±10 seconds to avoid synchronized expiry of many feeds at once.[web:1][web:4][web:13]  

6. **Consistency stance**  
   - Accept eventual consistency for feeds (users may see slightly stale posts for up to ~60s), but ensure profile changes or privacy settings use shorter TTLs and/or invalidation-on-write.[page:1][web:6][web:9]  

**Senior signal:** You explicitly talk **numbers (QPS, target P99)**, **trade-offs (staleness vs latency)**, and **operational risks (DB overload, cluster failover)** rather than just “add Redis”.  

---

## 9. Extra Analogies & Mental Models

### 9.1 Caching as a multi-level memory hierarchy

- L1 in-process cache → L2 Redis/Memcached → L3 DB → L4 cold storage (S3) mirrors CPU cache L1/L2/L3 → RAM → disk.[web:13][web:14]  

Consider:
- **Hit ratio** at each level.  
- **Latency** and **capacity** per level.  
- **Promotion / demotion** rules between levels.  

### 9.2 Caching as “front-of-house vs back-of-house”

- Front-of-house (cache): fast, small menu (only popular items).  
- Back-of-house (DB): full menu, slower; customers don’t interact directly.  
- Running out of front-of-house items = cache miss; you go to kitchen, prepare, then stock front-of-house again.  

**Senior signal:** Reuse analogies only to clarify, then immediately translate them back into **numbers and patterns**.  

---

## 10. Senior-Signaling Checklist for Caching Questions

When caching comes up, try to hit the following bullets naturally:

1. **Justification with data**
   - “Reads are N× writes, DB can’t handle M QPS, target latency is L; cache helps by cutting DB read QPS by ~hit_rate.”[page:1][web:11][web:13]  

2. **Layer choice**
   - “For global static assets I’d use CDN; for dynamic per-user data I’d use Redis; for ultra-hot config I’d use in-process cache.”[page:1][web:14]  

3. **Explicit read/write path**
   - Describe step-by-step what happens on hit vs miss, including DB and cache changes (even if you forget the pattern names).[page:1][web:32][web:41]  

4. **Eviction & TTL reasoning**
   - Name LRU/LFU/TTL and why you picked them for your access pattern.[page:1][web:13][web:14]  

5. **Failure modes & mitigation**
   - Penetration (Bloom filter / negative caching).[web:4][web:7][web:13]  
   - Stampede (per-key lock / refresh-ahead / stale-while-revalidate).[page:1][web:1][web:4][web:13]  
   - Avalanche (TTL jitter / HA cluster / degradation).[web:1][web:4][web:7][web:10][web:13]  
   - Hot keys (replication / local L1 cache).[page:1][web:13]  

6. **Consistency stance**
   - Clearly choose strong vs eventual consistency per data type and tie it to business risk.[page:1][web:6][web:9][web:12]  

7. **Operational story**
   - Mention monitoring (hit rate, latency, DB QPS), scaling (adding nodes, impact on hash ring), and what happens on cache failures.[web:11][web:13][web:14]  

---

## 11. Key Takeaways for Quick Revision

- **Default mental model:** cache-aside Redis in front of DB, LRU + TTL, invalidate on write, and be explicit about stale-read tolerance.[page:1][web:32][web:41]  
- **Think in layers:** client / CDN / LB / app cache / DB; pick the cheapest layer that solves your latency and load problem.[page:1][web:18][web:20]  
- **Name and handle the “three failures”:** penetration (Bloom filter, negative cache), breakdown/stampede (locks, warmup), avalanche (jitter, HA cluster).[page:1][web:1][web:4][web:7][web:10][web:13]  
- **Don’t over-optimize names:** interviewers care more that you can **describe the behavior** correctly than whether you recall “write-around” vs “write-through”.[page:1][web:32][web:41]  
- **Senior signal:** tie caching decisions to **numbers, failure modes, and operations**, not just “we’ll add a cache”.[page:1][web:11][web:13]  
