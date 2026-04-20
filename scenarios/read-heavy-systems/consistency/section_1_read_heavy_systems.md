# Section 1: Base Scenario --- Read-Heavy Systems

## What is the scenario?

A system where:

**Number of reads \>\> number of writes**

Quantitatively (interview-safe framing): - Read:Write ratio = **10:1,
100:1, even 1000:1**

------------------------------------------------------------------------

## Why does this happen?

Because **consumption massively outweighs creation** in most real
systems.

### Mental model:

-   Writes = *state changes*
-   Reads = *state consumption*

Most users: - **consume far more than they create**

------------------------------------------------------------------------

## Where does this show up (intuitively)?\*\*

Think in terms of **user behavior patterns**:

### 1. Social / Content Platforms

-   1 user posts
-   1M users read

Classic fan-out pattern

------------------------------------------------------------------------

### 2. Product Catalog / Search

-   Products updated occasionally
-   Millions of users browsing/searching

------------------------------------------------------------------------

### 3. News / Feeds

-   Few publishers
-   Massive readership

------------------------------------------------------------------------

### 4. Configuration / Metadata Systems

-   Config changes rarely
-   Services read constantly

------------------------------------------------------------------------

### 5. Analytics Dashboards

-   Data ingested periodically
-   Dashboards queried frequently

------------------------------------------------------------------------

## Core Characteristics of Read-Heavy Systems

### 1. Reads dominate system load

-   CPU, DB, network → mostly consumed by reads

Bottleneck shifts from writes → reads

------------------------------------------------------------------------

### 2. Latency sensitivity is high

Users tolerate: - Slow writes ❌ - Slow reads ❌❌❌ (worse)

Reads are often **user-facing critical path**

------------------------------------------------------------------------

### 3. Caching becomes first-class

-   You *cannot* hit DB for every read
-   System design becomes: "How do we avoid touching the database?"

------------------------------------------------------------------------

### 4. Replication is natural

-   Scale reads via replicas
-   Tradeoff: consistency vs availability

------------------------------------------------------------------------

### 5. Staleness becomes negotiable

In read-heavy systems: You often accept stale data to scale

------------------------------------------------------------------------

## Hidden Insight

A weak candidate says: "Use cache"

A strong candidate understands: "Read-heavy systems are fundamentally
about decoupling read path from write path"

Meaning: - Writes go one way (authoritative store) - Reads come from: -
caches - replicas - precomputed views

------------------------------------------------------------------------

## Where people mess this up in interviews

### Mistake 1:

Treating reads and writes symmetrically

Wrong. They require different architectures

------------------------------------------------------------------------

### Mistake 2:

Thinking DB scaling alone is enough

Even 100 replicas won't save you if: - cache miss rate is high - hot
keys exist

------------------------------------------------------------------------

### Mistake 3:

Ignoring access patterns

Not all reads are equal: - hot data vs cold data - fan-out vs point
lookup

------------------------------------------------------------------------

## One clean abstraction to lock in

Write Path: Client → DB (authoritative, slower, controlled)

Read Path: Client → Cache/Replica/View (fast, scalable, possibly stale)

------------------------------------------------------------------------

## Learnings / Takeaways

1.  Read-heavy systems are driven by user behavior asymmetry (consume
    \>\> create)
2.  Reads become the dominant bottleneck, not writes
3.  Caching and replication are not optimizations---they are necessities
4.  Decoupling read and write paths is the core architectural principle
5.  Accepting staleness is often required to achieve scale
