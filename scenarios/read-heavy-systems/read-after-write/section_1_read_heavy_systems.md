# Section 1 --- Base Scenario: Read-heavy systems

## What is the base scenario?

A **read-heavy system** is one where:

> **Reads \>\> Writes (often 10x--1000x more reads than writes)**

Typical ratio examples: - Twitter timeline: \~1000 reads per tweet
write\
- Product catalog: \~100 reads per update\
- News feed: massive fan-out reads, few writes

------------------------------------------------------------------------

## Why does this happen?

Because most systems follow this pattern:

> **Few users create/update data → Many users consume it repeatedly**

Think: - 1 user posts → 1M users read\
- 1 product updated → thousands browse it\
- 1 config change → millions depend on it

------------------------------------------------------------------------

## Core problem in read-heavy systems

The system bottleneck is almost always:

> **Serving reads efficiently (low latency + high throughput)**

So naturally, systems optimize for: - Caching (Redis, CDN) - Replication
(read replicas) - Precomputation (materialized views)

------------------------------------------------------------------------

## Canonical mental model

Write path: Rare but important\
Read path: Constant, massive, performance-critical

------------------------------------------------------------------------

## What does a default (no constraints yet) architecture look like?

Clients\
↓\
Cache (Redis/CDN)\
↓\
Read replicas (many)\
↓\
Primary DB (handles writes)

Key ideas: - Reads go to cache or replicas\
- Writes go to primary\
- Replication is usually async

------------------------------------------------------------------------

## Why this works (without constraints)

Because: - We prioritize latency + scale\
- Slight staleness is acceptable in many systems

------------------------------------------------------------------------

## Examples of pure read-heavy systems (no strict consistency yet)

-   Social feeds (Instagram, Twitter)
-   Product listings (Amazon)
-   Video metadata (YouTube)
-   News websites

These systems tolerate: \> Seeing slightly stale data is fine

------------------------------------------------------------------------

## Interview baseline

When interviewer says:

> Design a read-heavy system

Default instinct: - Cache aggressively\
- Use read replicas\
- Accept eventual consistency

------------------------------------------------------------------------

## Twist (what makes this problem interesting)

Constraint introduced:

> Strict consistency for reads after writes

Meaning:

> User must ALWAYS see their latest write immediately

------------------------------------------------------------------------

## Q&A / Clarifications

Q: Why are replicas typically async?\
A: To maximize write throughput and reduce latency. Sync replication
slows down writes.

Q: Why is cache placed before replicas?\
A: To absorb maximum read load and reduce pressure on databases.

Q: What breaks if reads are too high?\
A: Database becomes bottleneck → need caching + horizontal scaling.

------------------------------------------------------------------------

## Key Learnings

-   Read-heavy systems optimize for read scalability and latency
-   Default design uses cache + async replicas
-   Eventual consistency is usually acceptable
-   Introducing strict consistency will break default assumptions and
    require redesign
