# Read-Heavy Systems + Consistency Constraints (No-Compromise Cheat Sheet)

------------------------------------------------------------------------

# SECTION 1 --- READ-HEAVY SYSTEMS

## Definition

A read-heavy system is one where: Read:Write ratio \>\> 1 (10x, 100x,
1000x)

## Why it happens (behavioral asymmetry)

-   Creation is rare, consumption is massive
-   One write fans out to millions of reads

## Core mental model

Writes = truth mutation\
Reads = truth consumption

## Architectural consequence

You MUST decouple:

Write Path → correctness\
Read Path → scalability

------------------------------------------------------------------------

## System properties

### 1. Read path dominates infra cost

-   CPU (query execution)
-   Network (fan-out)
-   DB connections

### 2. Latency sensitivity

Users tolerate: - slow writes (sometimes) - NEVER slow reads

### 3. Caching becomes mandatory

Without cache → DB collapse

### 4. Replication becomes default

Scale reads horizontally

### 5. Staleness becomes negotiable

Trade correctness for performance

------------------------------------------------------------------------

## Strong insight

You are not scaling the DB. You are avoiding touching the DB.

------------------------------------------------------------------------

# SECTION 2 --- CONSISTENCY MODELS

## Definition

Consistency = what a read returns after a write

------------------------------------------------------------------------

## STRONG CONSISTENCY

Guarantee: After write → all reads see latest value

### Implications

-   Read from primary OR sync replicas
-   No stale data allowed

### Cost

-   High latency
-   Low availability
-   Low scalability

### Use cases

-   Payments
-   Inventory locking
-   Banking

------------------------------------------------------------------------

## EVENTUAL CONSISTENCY

Guarantee: System converges over time

### Implications

-   Reads can be stale
-   Async replication allowed

### Benefits

-   Low latency
-   High scalability

### Use cases

-   Feeds
-   Product catalog
-   Analytics

------------------------------------------------------------------------

## CAUSAL CONSISTENCY

Guarantee: Dependent operations appear in order

Example: Post → Comment must be seen in order

### Implications

-   Not globally consistent
-   But logically consistent

### Use cases

-   Chat systems
-   Social feeds

------------------------------------------------------------------------

## Key insight

Consistency is NOT global. It is scoped per feature.

------------------------------------------------------------------------

# SECTION 3 --- BEST SOLUTIONS

------------------------------------------------------------------------

## SOLUTION 1: CACHE + REPLICAS

### Architecture

Write: Client → Primary → async replicas + cache invalidation

Read: Client → Cache → Replica → Primary fallback

------------------------------------------------------------------------

### Why it works

-   Cache absorbs most traffic
-   Replicas scale horizontally
-   Async propagation enables throughput

------------------------------------------------------------------------

### Consistency handling

Strong: - bypass cache / read primary

Eventual: - cache + replicas freely

Causal: - session-aware routing - read-your-writes guarantees

------------------------------------------------------------------------

### Failure modes

1.  Cache stampede
2.  Replica lag
3.  Stale cache
4.  Hot keys

------------------------------------------------------------------------

### Tradeoffs

Pros: - simple - scalable - flexible

Cons: - invalidation complexity - inconsistency windows

------------------------------------------------------------------------

## SOLUTION 2: CQRS + PRECOMPUTED VIEWS

------------------------------------------------------------------------

## Core idea

Compute during write, not read

------------------------------------------------------------------------

## Example: Feed system

Instead of: Compute feed on request

Do: Push posts into prebuilt feed

------------------------------------------------------------------------

## Flow

Write: Client → DB → Event

Process: Event → Compute → Read model

Read: Client → Read model (fast)

------------------------------------------------------------------------

## Why it works

-   removes read-time computation
-   enables ultra-fast reads

------------------------------------------------------------------------

## Consistency

Eventual: default

Causal: enforced via ordered event streams

Strong: very hard

------------------------------------------------------------------------

## Tradeoffs

Pros: - extremely scalable - fast reads

Cons: - complex - debugging hard - eventual consistency inherent

------------------------------------------------------------------------

# SECTION 4 --- ANTI-PATTERNS

------------------------------------------------------------------------

## 1. Vertical DB scaling

Temporary fix, not scalable

------------------------------------------------------------------------

## 2. Blind replica usage

Causes stale reads

------------------------------------------------------------------------

## 3. Long TTL caching

Leads to correctness bugs

------------------------------------------------------------------------

## 4. Compute on read

Kills latency at scale

------------------------------------------------------------------------

## 5. Strong consistency everywhere

Destroys performance

------------------------------------------------------------------------

## 6. CQRS everywhere

Overengineering

------------------------------------------------------------------------

## Meta insight

Every bad solution ignores consistency implications

------------------------------------------------------------------------

# SECTION 5 --- REAL WORLD SYSTEMS

------------------------------------------------------------------------

## Social Feed

Consistency: Eventual + Causal

Design: - fan-out on write - precomputed feeds - cache

------------------------------------------------------------------------

## E-commerce

Consistency: - Eventual (catalog) - Strong (checkout)

------------------------------------------------------------------------

## Chat

Consistency: Causal

Design: - partition by conversation - ordered logs

------------------------------------------------------------------------

## Analytics

Consistency: Eventual

Design: - full CQRS - precomputed aggregates

------------------------------------------------------------------------

## Insight

Different parts use different consistency

------------------------------------------------------------------------

# SECTION 6 --- CONSTRAINT COMBINATIONS

------------------------------------------------------------------------

## Consistency + Latency

→ cache mandatory

------------------------------------------------------------------------

## Consistency + High writes

→ CQRS preferred

------------------------------------------------------------------------

## Consistency + Ordering

→ event streams

------------------------------------------------------------------------

## Consistency + Geo scale

→ eventual consistency

------------------------------------------------------------------------

## Consistency + Availability

→ CAP tradeoff

------------------------------------------------------------------------

## Final Insight

System design = constraint negotiation

No perfect solution. Only tradeoffs.

------------------------------------------------------------------------

# FINAL STAFF-LEVEL SUMMARY

Start: Cache + replicas

Evolve: CQRS when: - complexity ↑ - scale ↑ - ordering required

Restrict: Strong consistency to critical paths only
