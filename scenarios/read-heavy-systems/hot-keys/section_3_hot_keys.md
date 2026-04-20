# Section 3 --- Good Solutions for Hot Keys / Skew

## Solution 1 (Best): Replication + Request Fan-out (a.k.a. "scatter reads")

### Idea

Instead of 1 copy of hot data → create multiple replicas and distribute
read load across them

### How it works

Normally: key → 1 cache node → bottleneck

Now: key → N replicas (same data) ↓ load balanced reads

### Implementation patterns

1.  Cache-level replication (most common)

-   Store same key in multiple Redis nodes: post:123 → Redis A post:123
    → Redis B post:123 → Redis C

-   Client (or proxy) randomly picks one

2.  Application-level fan-out

-   App knows replicas:
    -   Picks one using:
        -   random
        -   round robin
        -   least loaded

3.  CDN (for static-ish content)

-   Global replication across edge nodes

### Why it works

-   Breaks single-node bottleneck
-   Converts: 1 node handling 1M QPS

into: 10 nodes handling 100k QPS each

### Tradeoffs

Pros: - Simple mental model - Works immediately for hot keys - Linear
scaling with replicas

Cons: - Write amplification (need to update N replicas) - Consistency
lag (replicas may be stale) - More memory usage

### Failure modes

-   Async replication → stale reads
-   Sync replication → write latency increases
-   Uneven load distribution → some replicas still hot

### Tech choices

-   Redis (manual replication or duplication)
-   Memcached clusters
-   CDN (Cloudflare, Akamai)
-   Kafka-backed cache warmers

------------------------------------------------------------------------

## Solution 2: Request Coalescing / Single-flight

### Idea

If many requests for same key arrive simultaneously → collapse them into
one backend request

### How it works

Instead of: 1000 requests → DB/cache

Do: 1 request → DB 999 wait → reuse result

### Where this applies

-   Cache miss
-   Cache expiry
-   Cold start

### Implementation

1.  In-process

-   Use a map: key → in-flight request

-   If request exists: → wait on same promise/future

2.  Distributed lock (Redis)

-   First request acquires lock
-   Others wait or retry

3.  Libraries

-   Go: singleflight
-   Java: Futures
-   Node: Promise deduping

### Why it works

-   Prevents thundering herd
-   Protects DB from spikes
-   Reduces duplicate work

### Tradeoffs

Pros: - Huge reduction in backend load - Easy to justify

Cons: - Adds latency for waiting requests - Needs timeout handling -
Doesn't solve sustained load

### Failure modes

-   Lock contention
-   Deadlocks (bad implementation)
-   Cascading waits if backend slow

### Tech choices

-   Redis locks (SETNX)
-   In-memory locks
-   Middleware layer

------------------------------------------------------------------------

## Combined Interview Answer

"For hot keys, I'd combine: 1. Replication to distribute sustained load
2. Request coalescing to handle burst traffic and cache misses"

------------------------------------------------------------------------

# Optional Deep Dive Q&A

Q1. Why replication alone doesn't solve cache stampede? A: Because on
cache expiry, all replicas miss simultaneously → multiple backend hits
still occur.

Q2. Why coalescing alone doesn't solve sustained load? A: It only
reduces duplicate concurrent requests, but sustained high QPS still hits
a single node.

Q3. When would CDN completely eliminate hot key issue? A: When data is
static/rarely changing and can be globally cached at edge nodes,
distributing load geographically.
