# Read-Heavy Systems + Hot Keys / Skew — HLD Deep Dive (V2)

---

# 0. How to use this doc
This is not a summary. This is a **thinking framework**.
- Read once slowly
- Revisit sections before interviews
- Practice explaining sections out loud

---

# 1. Base Scenario — Read Heavy Systems

## Core Definition
Read-heavy systems are those where:
- Reads dominate writes (often by orders of magnitude)
- Example ratio: 1 write → 10^3–10^7 reads

## Why it happens (First principles)
Human + system amplification:
- Humans converge on popular content
- Systems (ranking/recommendation) amplify it

## Mental Model
This is a **serving problem**, not a storage problem.

> Your job is NOT to store data — your job is to serve it fast repeatedly.

## Baseline Architecture
Client → CDN → Cache → DB

## Key Properties
- Latency-sensitive (user-facing)
- Cache-friendly
- Eventually consistent acceptable (usually)

---

# 2. Constraint — Hot Keys / Skew

## Definition
A small number of keys receive disproportionate traffic.

## Distribution
Zipf-like:
- Top 1% → majority traffic

## Example
- Celebrity post
- Viral reel
- IPL live score

## Why it breaks systems

### Normal assumption:
“Sharding distributes load”

### Reality:
Same key → same shard → bottleneck

## Failure Modes

### 1. Cache hotspot
One node overloaded

### 2. Thundering herd
Cache miss → DB storm

### 3. DB meltdown
Fallback cannot handle concentrated load

### 4. Uneven utilization
1 node hot, others idle

## Key Insight
> Hot keys convert horizontal systems into single-node bottlenecks

---

# 3. Good Solutions

## Solution 1 — Replication + Fan-out

### Idea
Duplicate hot data across multiple nodes.

### Flow
Instead of:
key → node A

Do:
key → node A, B, C
client distributes reads

### Why it works
Breaks single-node bottleneck

### Tradeoffs
- Write amplification
- Staleness
- Memory overhead
- Invalidation complexity

### Failure Modes
- Replica sync lag
- Uneven load
- All replicas expiring together → stampede

### Fixes
- TTL jitter
- Staggered refresh
- Coalescing

---

## Solution 2 — Request Coalescing (Single-flight)

### Idea
If request is in-flight → reuse it

### Flow
1000 requests → 1 backend call

### Where it helps
- Cache miss
- Cold start
- Burst traffic

### Tradeoffs
- Adds wait latency
- Doesn’t help steady load

### Failure Modes
- Lock contention
- cascading waits

---

## Combined Strategy (Interview Gold)
Replication → handles sustained load  
Coalescing → handles bursts

---

# 4. Bad / Trap Solutions

## 1. More Shards
Fails because same key → same shard

## 2. Vertical Scaling
Temporary fix, not scalable

## 3. Increase TTL
Reduces misses, doesn’t distribute load

## 4. Rate Limiting
Drops valid traffic → bad UX

## 5. DB Read Replicas
Too slow + expensive

## 6. Consistent Hashing
Balances keys, not traffic

---

# 5. Real World Systems

## Instagram / Twitter
- CDN for media
- Replicated caches for metadata
- Eventually consistent counters

## YouTube
- CDN handles majority traffic
- Pre-warming for trending videos

## Amazon
- Aggressive caching
- Strong consistency only where needed (inventory)

## IPL / Live Scores
- Redis in-memory
- Replication
- Push updates (avoid DB reads)

---

# 6. Constraint Combinations

## Hot Keys + Strong Consistency
- Replication weakens
- Use leader-based reads
- Tradeoff: scalability ↓

## Hot Keys + Low Latency
- CDN critical
- Avoid cache misses entirely

## Hot Keys + High Writes
- Batch writes
- Async processing (Kafka)
- Counters

## Hot Keys + Large Scale
- Multi-layer caching
- Geo-distribution

## Hot Keys + Fault Tolerance
- Over-replication
- Load rebalancing

## Hot Keys + Ordering
- Separate read/write paths
- Single writer, multiple readers

---

# 7. Advanced Concepts (High Signal)

## Hot Key vs Partition vs Shard
- Hot key → data problem
- Hot partition → grouping problem
- Hot shard → infra symptom

## Dynamic Replication
Increase replicas based on QPS

## Detection
- Top keys by QPS
- Node-level imbalance

---

# 8. Mental Models

## Model 1
“Same key → same shard → no scaling”

## Model 2
“Replication = horizontal scaling for one key”

## Model 3
“Coalescing = collapse duplicate work”

---

# 9. Interview Answer Template

“For hot keys in a read-heavy system, I’d:
1. Replicate data across multiple cache nodes to distribute sustained load
2. Use request coalescing to prevent thundering herd during cache misses
3. Add TTL jitter and possibly pre-warming to avoid synchronized expiry
4. Optionally use CDN if data is static

If additional constraints like strong consistency or high write frequency exist, I’d adjust by trading off replication or batching writes.”

---

# 10. Final Takeaway

Hot keys are not a caching problem.

They are a **distribution problem**.

And the solution is:
→ Break single-node ownership of that key

