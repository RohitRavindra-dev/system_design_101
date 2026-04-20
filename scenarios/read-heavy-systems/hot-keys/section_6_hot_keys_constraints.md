# Section 6 — Constraint Combinations (Read-heavy systems + Hot Keys)

This section explores how **hot key problems interact with other constraints** in real-world systems. The goal is to understand how adding constraints changes system design decisions and invalidates or weakens certain solutions.

---

## 1. Hot Keys + Strong Consistency

### What changes?
Earlier:
- Eventual consistency was acceptable

Now:
- Reads must reflect the latest writes

### Why this is hard
Replication introduces:
- Multiple copies
- Sync/async propagation

Strong consistency conflicts with replication.

### Impact on solutions

**Replication weakens**
- Async replication → stale reads
- Sync replication → high write latency

**CDN becomes mostly unusable**
- Edge caches inherently serve stale data

### What works instead

**Leader-based reads**
- All reads and writes go to a single leader
- Ensures correctness but creates a bottleneck

**Strict cache invalidation**
- Writes invalidate all caches immediately
- Reads fetch fresh data

**Quorum systems**
- Read/write from majority replicas

### Tradeoffs

| Property | Impact |
|---------|--------|
| Consistency | Strong |
| Latency | Higher |
| Scalability | Limited |

### Key insight
Hot keys + strong consistency fundamentally limit horizontal scaling. Systems often relax consistency to scale.

---

## 2. Hot Keys + Low Latency (Strict SLA)

### What changes?
- p99 latency becomes critical (<100ms or tighter)

### Impact

**CDN becomes critical**
- Moves data closer to users

**Replication becomes mandatory**
- Avoids queueing delays

**Coalescing tradeoff**
- Waiting introduces latency

### Additional techniques

**Pre-warming**
- Keep cache hot for known hot keys

**TTL jitter**
- Avoid simultaneous expiry

**Edge computation**
- Move logic closer to CDN

### Key insight
Avoid cache misses entirely for strict latency systems.

---

## 3. Hot Keys + High Write Frequency

### What changes?
- Same key receives high reads and high writes

### Impact

**Replication becomes complex**
- Multiple replicas must be updated

**Cache becomes unstable**
- Frequent invalidations

### What works

**Write aggregation**
- Buffer writes and batch updates

**Asynchronous processing**
- Use queues (Kafka) to process writes

**CRDTs / counters**
- Eventually consistent updates

### Key insight
Reduce write frequency instead of scaling writes directly.

---

## 4. Hot Keys + Large Scale (Millions QPS)

### What changes?
- Even cache nodes become bottlenecks

### Impact

**Multi-layer caching**

CDN → Regional cache → Application cache → DB

**Massive replication**
- Same key stored across many nodes

**Geo-distribution**
- Serve from nearest region

### Failure mode
- Cross-region consistency lag

### Key insight
At extreme scale, multiple cache layers and geo-replication are necessary.

---

## 5. Hot Keys + Fault Tolerance

### What changes?
- System must survive node failures

### Impact

**Replication becomes mandatory**
- Needed for both load and resilience

### New problem
- Node failure shifts traffic to remaining nodes, creating new hotspots

### Solutions

**Over-replication**
- Maintain extra replicas

**Smart load balancing**
- Prevent sudden overload

**Graceful degradation**
- Serve stale data if needed

### Key insight
Replication helps but requires careful traffic redistribution during failures.

---

## 6. Hot Keys + Ordering Requirements

### What changes?
- Data must be processed or read in order

### Impact

**Replication can break ordering**
- Different replicas may have different states

**Parallel reads can cause inconsistency**

### What works

**Partition by key**
- Ensures ordering but creates hotspot

**Hybrid approach**
- Writes: single partition (ordered)
- Reads: replicated

### Key insight
Ordering often forces single-writer systems, so read/write paths must be separated.

---

# Meta Pattern

All constraint combinations revolve around balancing:

Consistency vs Latency vs Scalability

- Hot keys stress scalability
- Additional constraints push toward consistency or latency guarantees
- Tradeoffs are unavoidable

---

# Final Mental Model

Hot Key Problem →
    Solve using Replication + Coalescing

Add Constraints →
    Break assumptions →
        Introduce tradeoffs →
            Choose based on system goals

---

# Takeaways (Concise)

- Hot keys are easy with eventual consistency but hard with strong consistency.
- Replication improves scalability but conflicts with consistency and ordering.
- Coalescing helps bursts but not sustained load.
- Multi-layer caching and CDN are essential at extreme scale.
- Write-heavy hot keys require reducing write frequency, not scaling writes.
- Every added constraint forces tradeoffs between consistency, latency, and scalability.
