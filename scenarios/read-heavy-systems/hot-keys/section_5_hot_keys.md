# Section 5 + Clarifications --- Read Heavy Systems with Hot Keys

------------------------------------------------------------------------

## Part 1 --- Follow-ups

### 1. What if hot key is also write-heavy?

High reads + high writes introduce contention and consistency issues.

Problems: - Write amplification across replicas - Race conditions -
Stale reads

Approaches: - Leader + read replicas (writes go to leader, reads to
replicas) - Write-through / write-around cache - Reduce write frequency
(batching, aggregation)

Key insight: Hot + write-heavy makes caching harder due to consistency
constraints.

------------------------------------------------------------------------

### 2. How to detect hot keys in production?

Techniques: - Cache metrics (Redis top keys, QPS, latency) - Application
logging (track key access frequency) - Sampling (analyze subset of
traffic) - Infra signals (one node overloaded, others idle)

Key signal: Top 1% keys contribute disproportionate traffic.

------------------------------------------------------------------------

### 3. How to dynamically adapt replication factor?

Steps: 1. Detect hot key via QPS threshold 2. Increase replicas
dynamically 3. Distribute load across replicas 4. Scale down when
traffic drops

Challenges: - Consistency across replicas - Lifecycle management -
Memory overhead

Key insight: Replication factor should be dynamic, not static.

------------------------------------------------------------------------

## Part 2 --- Mental Model Refinement

### Distributed Redis

Instead of single instance:

Client → multiple Redis nodes

Two concepts: - Sharding: different keys → different nodes -
Replication: same key → multiple nodes

Hot key solution uses replication, not sharding.

------------------------------------------------------------------------

### Routing

Requests distributed via: - Random - Round robin - Least loaded

------------------------------------------------------------------------

### Cons of replication

-   Synchronized expiry → DB stampede
-   Stale replicas
-   Memory overhead
-   Cache invalidation complexity
-   Load imbalance if routing poor

------------------------------------------------------------------------

### Request Coalescing

Not time-based, but overlap-based.

If request is already in-flight: → reuse it

Helps burst traffic, not steady load.

------------------------------------------------------------------------

### Pre-warming cache

Useful for predictable traffic. Not useful for sudden spikes.

------------------------------------------------------------------------

### Final Mental Model

Combine: - Replication (for sustained load) - Coalescing (for burst
control) - TTL jitter + prewarming (optional)

------------------------------------------------------------------------

## Section 5 --- Real World Mappings

### Instagram / Twitter

-   CDN for media
-   Cache replication for metadata
-   Fan-out reads
-   Buffered writes

------------------------------------------------------------------------

### YouTube

-   CDN dominant
-   Metadata caching
-   Pre-warming trending content

------------------------------------------------------------------------

### Amazon

-   Aggressive caching
-   Replication
-   DB rarely hit
-   Strong consistency for inventory

------------------------------------------------------------------------

### IPL / Live Score

-   In-memory cache
-   Replication
-   Push-based updates
-   Avoid DB reads

------------------------------------------------------------------------

### Google Docs

-   CRDT / operational transforms
-   Local buffering

------------------------------------------------------------------------

### Common Pattern

CDN → Cache → Replication → Coalescing → DB

------------------------------------------------------------------------

### Key Takeaway

Hot keys handled by: - Moving data closer to users - Distributing load
across replicas

------------------------------------------------------------------------

## Optional Prompts (Short Answers)

### When would you NOT replicate a hot key?

-   If data is highly dynamic and consistency critical
-   Memory constraints
-   Low QPS

------------------------------------------------------------------------

### How to choose replication factor?

-   Based on QPS per node capacity
-   Adjust dynamically
-   Consider memory tradeoff

------------------------------------------------------------------------

### How do CDNs handle hot objects?

-   Replicate across edge locations
-   Cache eviction policies
-   Load balancing at edge
