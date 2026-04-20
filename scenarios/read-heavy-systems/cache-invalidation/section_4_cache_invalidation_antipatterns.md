# SECTION 4 — Bad Solutions / Anti-patterns

## 🚫 1. “Just use TTL everywhere”

### Why it’s tempting
- Dead simple
- No need to track writes
- Feels scalable (“eventually it fixes itself”)

### What it looks like
Cache set with TTL = 5 mins  
No explicit invalidation

### Why it’s wrong (core issue)
TTL ≠ correctness, it’s just delayed inconsistency

### Failure scenarios

#### 1. Stale data window is uncontrolled
- Data updated at T=0
- TTL expires at T=5 mins  
You serve wrong data for 5 minutes

#### 2. High-write systems break badly
- Price updates every 10 sec
- TTL = 5 mins  
Cache is almost always wrong

#### 3. Thundering herd on expiry
- All keys expire at same time
- Massive DB spike

### What it takes to “force-fit” this
- Very short TTL (kills cache benefit)
- Staggered expiry (jitter)
- Combine with invalidation anyway

### Interview answer
TTL alone is not sufficient for correctness. It works only when staleness is acceptable and data changes infrequently.

---

## 🚫 2. “Update cache first, DB later” (write-back caching without safeguards)

### Why it’s tempting
- Super fast writes
- Low latency

### What it looks like
Write → Update cache → Async write to DB

### Why it’s dangerous
Cache becomes source of truth without durability guarantees

### Failure scenarios
- Cache crash = data loss
- Write never reaches DB
- Multi-node inconsistency

### When it can work
- With Kafka / durable log
- Strong retry guarantees

### Interview answer
Write-back caching is risky unless durability and retry mechanisms are added.

---

## 🚫 3. “Delete cache and assume everything is fine”

### Why it’s tempting
- Common pattern
- Looks correct

### What people miss (race condition)
T1: Read miss → fetch old value  
T2: Write → DB updated → cache deleted  
T1: Writes old value into cache

### Why this is bad
Cache becomes stale again

### Fix
- Versioning
- Compare-and-set
- Request coalescing

### Interview answer
Naive invalidation can reintroduce stale data due to race conditions.

---

## 🚫 4. “Cache everything blindly”

### Why it’s tempting
- Feels like more cache = better

### Why it fails
- Low hit ratio
- Cache pollution
- Memory explosion

### Fix
Cache only hot / frequently accessed data

---

## 🚫 5. “Strong consistency via distributed locks everywhere”

### Why it’s tempting
- Prevents races

### Why it’s problematic
- Kills scalability
- Latency spikes
- Deadlocks

### When acceptable
- Small critical sections

---

## 🚫 6. “Synchronous fan-out invalidation everywhere”

### Why it’s tempting
- Immediate consistency

### Why it fails
- High latency
- Cascading failures
- Tight coupling

### Better approach
- Async invalidation (events, Kafka)

---

## 🚫 7. Invalid approaches

- Don’t use cache
- Use DB replicas only
- Always bypass cache

---

# Optional Deep Dive Q&A

### Q1: How to fix race in cache-aside?
Use versioning or compare-and-set so stale writes don’t overwrite fresh cache.

### Q2: What is single-flight / request coalescing?
Only one request fetches data on cache miss; others wait to prevent DB overload.

### Q3: How to prevent cache stampede?
- Use locks or single-flight
- Add jitter to TTL
- Warm cache proactively

### Q4: What is CDC-based invalidation?
Use Change Data Capture (e.g., Kafka) to propagate DB changes to cache asynchronously.
