# Section 3 — Best Solutions (Read-heavy systems + Cache invalidation)

---

# 🥇 Solution 1: Cache-aside (Lazy loading) + Explicit Invalidation

## Core idea
Cache is updated on reads, and invalidated on writes

---

## Flow

### Read path:
1. Check cache  
2. If hit → return  
3. If miss → read DB → populate cache → return  

### Write path:
1. Write to DB  
2. Delete (invalidate) cache  

---

## Why this works
- Cache is only populated when needed → efficient  
- DB remains source of truth  
- Invalidation ensures stale data eventually disappears  

---

## Example

GET /user/123  
→ cache miss → DB → cache set  

UPDATE /user/123  
→ DB updated  
→ cache key deleted  

---

## Technologies
- Cache: Redis, Memcached  
- DB: SQL/NoSQL  
- App layer handles logic  

---

## Tradeoffs

### Pros
- Simple, widely used, easy to reason about  
- Works well for most read-heavy systems  
- No tight coupling between cache and DB  

### Cons

1. Stale window exists  
Between: DB write → cache delete → next read → cache refill  

2. Race condition  
T1: read (miss) → fetch old value  
T2: write → DB updated → cache deleted  
T1: writes old value back into cache  

3. Thundering herd  
- Cache invalidated  
- Many requests hit DB simultaneously  

---

## Failure modes at scale

1. DB overload on cache miss storms  
2. Hot keys causing DB meltdown  
3. Inconsistent multi-key invalidation  

---

## When to use this
Default choice unless proven otherwise  

---

# 🥇 Solution 2: Write-through / Write-around with Versioning

## Core idea
Writes update both DB and cache, often with versioning to prevent stale overwrites  

---

## Variants

### Write-through
Write → Update DB → Update cache  

### Write-around
Write → Update DB → Skip cache → Cache filled on next read  

---

## Versioning

cache[user:123] = { value, version }

Only update if incoming_version > current_version  

---

## Why this works
- Prevents stale overwrites  
- Ensures monotonic correctness  
- Reduces race condition issues  

---

## Example

T1: write version 1  
T2: write version 2  

Cache only accepts version 2  

---

## Technologies
- Redis (atomic ops, Lua scripts)  
- Kafka (optional async updates)  
- DB with version/timestamp  

---

## Tradeoffs

### Pros
- Stronger consistency guarantees  
- Handles race conditions  
- Good for critical data  

### Cons

1. Higher write latency  
2. Increased coupling  
3. Cache pollution (write-through)  
4. Added complexity  

---

## Failure modes at scale

1. Partial failure  
DB write succeeds, cache update fails  

2. Distributed ordering issues  

3. Write amplification  

---

## When to use this
- Strong correctness required  
- Read-after-write consistency matters  
- Critical data (money, inventory)  

---

# Comparison

| Aspect | Cache-aside | Write-through + Versioning |
|--------|------------|----------------------------|
| Simplicity | Simple | Complex |
| Read performance | Excellent | Excellent |
| Write latency | Low | Higher |
| Consistency | Eventual | Stronger |
| Race handling | Weak | Strong |
| Default choice | Yes | Only when needed |

---

# Senior-level framing

Start with cache-aside for simplicity.  
If correctness requirements tighten, move to write-through with versioning or hybrid approaches.

---

# Hybrid insight

Real systems:
- Cache-aside for most data  
- Write-through for critical paths  

---

# Optional Deep Dive Q&A

## 1. How to fix race condition in cache-aside?
Use techniques like:
- Delayed double delete  
- Write-through fallback  
- Versioning  
- Distributed locks  

---

## 2. How to prevent thundering herd?
- Request coalescing (single flight)  
- Mutex/locks per key  
- Staggered TTLs  
- Background refresh  

---

## 3. Hybrid patterns?
- Cache-aside + TTL  
- Write-through for critical data  
- Async invalidation via Kafka  

---

## 4. Event-driven invalidation?
- Use CDC (Change Data Capture)  
- Publish DB changes to Kafka  
- Consumers update/invalidate cache  

---

# Summary

- Cache-aside = default, simple, eventual consistency  
- Write-through + versioning = stronger consistency, more complex  
- Real systems mix both approaches  
