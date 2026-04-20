# Section 4 — Bad Solutions / Anti-patterns

This section focuses on realistic traps that candidates often fall into during system design interviews when dealing with read-heavy systems requiring strict read-after-write consistency.

---

## ❌ Anti-pattern 1: “Just use cache + invalidate on write”

### Why it’s tempting

Feels clean and commonly used:

Write → DB → Invalidate cache  
Read → Cache (fresh)

You might say:
“We’ll just invalidate Redis on every write, so reads are always fresh.”

---

### Why this breaks (subtle but critical)

Race condition:

T1: Read → cache miss  
T2: Write → DB updated → cache invalidated  
T1: Reads old DB value → sets stale cache  

Result: stale data is back in cache.

Also:
- Cache invalidation is not atomic with DB write
- Distributed systems introduce network delays and retries
- Multiple caches can lead to inconsistent invalidation

---

### Key takeaway

Cache invalidation alone cannot guarantee read-after-write consistency due to race conditions and lack of atomicity with the write.

---

### If forced to work

You would need:
- Distributed locks
- Write-through cache
- Strict ordering guarantees

This increases complexity significantly and effectively recreates a more complex version of better solutions.

---

## ❌ Anti-pattern 2: “Just reduce replica lag”

### Why it’s tempting

Sounds reasonable:
“We’ll make replication faster so staleness is negligible.”

---

### Why it’s wrong

Even 1ms lag violates strict consistency.

Example:
Write → primary  
Read → replica (1ms behind)  

Still stale → still incorrect.

---

### Key takeaway

Reducing replication lag improves freshness but does not provide correctness guarantees.

---

### If forced to work

You would need synchronous replication, which becomes a different solution entirely.

---

## ❌ Anti-pattern 3: “Use TTL-based cache (short expiry)”

### Why it’s tempting

“Let’s keep cache TTL very low so data stays fresh.”

---

### Why it’s wrong

TTL provides probabilistic freshness, not guaranteed correctness.

Example:
Write at T=0  
Read at T=0.1 → cache still valid → stale data  

---

### Key takeaway

TTL reduces staleness window but does not eliminate it, so it cannot satisfy strict read-after-write guarantees.

---

## ❌ Anti-pattern 4: “Always read from cache + update cache on write” (write-through)

### Why it’s tempting

Feels strong and consistent:

Write → DB + Cache (synchronously)  
Read → Cache  

---

### Why it still breaks

Problem 1: Multiple writers  
- Concurrent writes can cause ordering issues

Problem 2: Cache failure  
Write → DB success  
Write → Cache fails  
Now cache is stale

Problem 3: Distributed cache  
- Multiple nodes → inconsistency

---

### Key takeaway

Cache is not a source of truth and cannot guarantee strict consistency.

---

## ❌ Anti-pattern 5: “Client retries until fresh data appears”

### Why it’s tempting

“We’ll just retry until we get the updated value.”

---

### Why it’s bad

- Unbounded retries → poor user experience
- Increased load → thundering herd problem
- Still no guarantee if replica is stuck

---

### Key takeaway

Retry-based approaches shift inconsistency to the client but do not eliminate it.

---

## ❌ Anti-pattern 6: “Global strong consistency everywhere”

### Why it’s tempting

“Let’s enforce strong consistency everywhere to be safe.”

---

### Why it’s a trap

- Increased latency
- Reduced availability
- Poor scalability for read-heavy systems

---

### Key takeaway

Applying global strong consistency unnecessarily sacrifices scalability.

---

## 🚫 Explicitly invalid approaches

These should be quickly rejected in interviews:

- Async replication with blind replica reads  
- Cache-aside without routing logic  
- TTL-based freshness approaches  
- Assuming replication is “fast enough”  

---

## 🧠 Meta-pattern

All bad solutions share the same flaw:

They optimize for freshness instead of guaranteed correctness.

---

## ✅ Learnings from Section 4

- Strict consistency is binary — not probabilistic
- Cache and replicas are fundamentally eventually consistent
- Race conditions are the biggest hidden risk
- Correctness must be enforced through architecture, not assumptions
- Strong candidates explicitly reject tempting but flawed solutions
