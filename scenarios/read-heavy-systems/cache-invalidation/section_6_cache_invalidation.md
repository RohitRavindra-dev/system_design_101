# Section 6 --- Constraint Combinations (Read-heavy + Cache Invalidation)

## 1. + Strong consistency (read-after-write)

### What changes?

Requirement: "If I write something, I MUST see it immediately"

### Why this is hard

Cache-aside flow: Write → DB updated → cache deleted → next read There
is a window of inconsistency.

### Impact on solutions

-   Cache-aside (alone): NOT sufficient
-   Write-through + versioning: VALID
-   Hybrid: write-through for critical, cache-aside for rest

### Additional techniques

-   Session-level bypass (read from DB after write)
-   Sticky reads / read-your-writes

### Interview framing

For read-after-write guarantees, avoid pure cache-aside; use
write-through or session-level handling.

------------------------------------------------------------------------

## 2. + High write throughput

### What changes?

Writes are frequent → cache invalidation happens constantly.

### Why dangerous

Cache becomes unstable / low hit ratio.

### Impact

-   Write-through: high write amplification
-   Cache-aside: frequent invalidations, low benefit

### Better approach

Event-driven invalidation (async): Write → DB → emit event → consumers
update/invalidate cache

### Why works

Decouples write path, scales better, allows retries.

### New problems

Event lag, ordering, exactly-once complexity.

### Interview framing

Use async event pipeline for invalidation at high write throughput.

------------------------------------------------------------------------

## 3. + Low latency (strict SLA)

### What changes?

Every ms matters → cache must hit.

### Impact

-   Cache miss unacceptable
-   Sync invalidation increases latency

### Best approach

Precompute + aggressive caching Avoid misses entirely.

### Techniques

-   Background refresh
-   Cache warming
-   Multi-layer cache

### Tradeoff

Accept staleness for latency.

------------------------------------------------------------------------

## 4. + Hot keys / skewed traffic

### What changes?

Certain keys receive massive QPS.

### Problems

Cache node bottleneck, DB meltdown on miss.

### Solutions

-   Replicated cache
-   Request coalescing (single-flight)
-   Local (L1) caching

### Interview framing

Combine replication, coalescing, and local caching.

------------------------------------------------------------------------

## 5. + Distributed / multi-region

### What changes?

Multiple regions, multiple caches.

### Problem

Invalidation must propagate globally.

### Solution

Event-driven global propagation: Write → DB → global event → all regions
update cache

### Challenges

Latency, ordering, conflicts.

### Techniques

-   Versioning
-   Conflict resolution

------------------------------------------------------------------------

## 6. + Derived / aggregated data

### What changes?

Data depends on multiple sources.

### Problem

One write affects many cache entries.

### Solutions

-   Precomputed views
-   Event-driven recomputation

### Flow

Event → recompute → update cache

------------------------------------------------------------------------

# Summary / Takeaways

-   Caching strategy is constraint-driven, not one-size-fits-all.
-   Cache-aside is baseline; evolve based on requirements.
-   Strong consistency → write-through or session guarantees.
-   High write load → async/event-driven invalidation.
-   Low latency → precompute and maximize cache hits.
-   Hot keys → replication + coalescing.
-   Multi-region → global event propagation + versioning.
-   Derived data → recomputation pipelines, not simple invalidation.
