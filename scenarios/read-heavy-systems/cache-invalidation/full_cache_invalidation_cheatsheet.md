# Master Cheat Sheet --- Read Heavy Systems + Cache Invalidation

------------------------------------------------------------------------

# SECTION 1 --- Read-heavy systems

## Definition

Read-heavy systems are those where reads dominate writes (100x--1000x).

## Why this happens

-   Humans consume more than they create
-   Fan-out effect (1 write → many reads)
-   Repeated access patterns

## Problems

-   DB becomes bottleneck
-   Latency increases
-   Cost increases

## Solution direction

Introduce cache: Client → Cache → DB fallback

## Key insight

Cache introduces duplicate state → leads to consistency issues

------------------------------------------------------------------------

# SECTION 2 --- Cache Invalidation Complexity

## Core problem

When DB updates, cache becomes stale.

## Types of inconsistency

-   Stale reads
-   Race conditions
-   Partial invalidation
-   Thundering herd
-   Eventual consistency window

## Tradeoff triangle

-   Correctness
-   Performance
-   Simplicity

## Key interview questions

-   Can we tolerate staleness?
-   Do we need read-after-write?
-   What is write frequency?

------------------------------------------------------------------------

# SECTION 3 --- Best Solutions

## 1. Cache-aside

### Flow

Read: cache → DB → cache set\
Write: DB → cache delete

### Pros

-   Simple
-   Efficient

### Cons

-   Stale window
-   Race conditions
-   Thundering herd

------------------------------------------------------------------------

## 2. Write-through + Versioning

### Flow

Write: DB + cache update

### Why versioning?

Prevents stale overwrite due to race conditions.

### Pros

-   Strong consistency
-   Handles concurrency

### Cons

-   Higher latency
-   Complexity

------------------------------------------------------------------------

# SECTION 4 --- Anti-patterns

## TTL-only

-   Uncontrolled staleness
-   Works only for static data

## Write-back

-   Risk of data loss

## Naive invalidation

-   Race conditions

## Blind caching

-   Low hit ratio

## Distributed locks everywhere

-   Poor scalability

## Sync invalidation

-   High latency, coupling

------------------------------------------------------------------------

# SECTION 5 --- Real-world mappings

## Social media

-   Precompute + async invalidation
-   Accept eventual consistency

## E-commerce

-   Hybrid approach
-   Strong consistency for price/inventory

## Ride-hailing

-   Minimal caching
-   Fast-changing data

## News

-   TTL + CDN

## Payments

-   Strong consistency
-   Write-through

------------------------------------------------------------------------

# SECTION 6 --- Constraint Combinations

## Strong consistency

→ Use write-through

## High write throughput

→ Async invalidation

## Low latency

→ Precompute

## Hot keys

→ Replication + coalescing

## Multi-region

→ Global event propagation

## Derived data

→ Event-driven recomputation

------------------------------------------------------------------------

# FINAL TAKEAWAYS

-   Cache strategy depends on constraints
-   No one-size-fits-all solution
-   Always reason about tradeoffs
-   Start simple → evolve based on needs
