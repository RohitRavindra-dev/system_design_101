# Section 3: Best Solutions for Read-Heavy Systems + Consistency Constraints

------------------------------------------------------------------------

## 🥇 Solution 1: Cache + Async Replication + Consistency-Aware Read Routing

### Core Idea

Write Path: Client → Primary DB → (async) replicas + cache invalidation

Read Path: Client → Cache → Replica → (fallback) Primary

------------------------------------------------------------------------

### Why this works

-   Cache → ultra low latency
-   Replicas → horizontal read scaling
-   Async propagation → high throughput

------------------------------------------------------------------------

### Consistency Handling

#### Eventual Consistency

-   Cache TTL / invalidation
-   Replica lag acceptable

#### Causal Consistency

-   Session awareness
-   Read-your-writes via:
    -   sticky sessions
    -   version tokens

#### Strong Consistency

-   Reads → primary only
-   Sync replication (expensive)

------------------------------------------------------------------------

### Tech Stack

-   Redis (cache)
-   PostgreSQL / MySQL (DB)
-   Kafka (optional for invalidation/events)

------------------------------------------------------------------------

### Tradeoffs

Pros: - Massive read scalability - Low latency - Flexible consistency

Cons: - Cache invalidation complexity - Replica lag - Hot key issues

------------------------------------------------------------------------

### Failure Modes

-   Cache stampede
-   Replica lag spike
-   Invalidation bugs
-   Hot partitions

------------------------------------------------------------------------

## 🥈 Solution 2: Precomputed Views / CQRS

### Core Idea

Write Path: Client → DB → Event Stream

Read Model: Event Stream → Precomputed Views → Fast Reads

------------------------------------------------------------------------

### What are Precomputed Views?

Instead of computing on read, compute ahead of time.

Example: user_feed: \[user_id → precomputed list of posts\]

------------------------------------------------------------------------

### What is CQRS?

Command (write model): - Handles writes - Source of truth

Query (read model): - Handles reads - Optimized for access patterns

------------------------------------------------------------------------

### Why this works

-   Eliminates heavy read-time computation
-   Enables extremely fast reads
-   Scales well

------------------------------------------------------------------------

### Consistency Handling

-   Eventual consistency (default)
-   Causal via ordered event streams
-   Strong consistency is hard

------------------------------------------------------------------------

### Ordering & Causality

-   Use partitioned event streams
-   Same key → same partition → ordered processing

Example: Post A → Comment B Ensures A is processed before B

------------------------------------------------------------------------

### Tech Stack

-   Kafka (event streaming)
-   Flink / Spark (processing)
-   NoSQL / Redis (read models)

------------------------------------------------------------------------

### Tradeoffs

Pros: - Extremely fast reads - Handles complex queries - Scales
massively

Cons: - High complexity - Data duplication - Processing lag

------------------------------------------------------------------------

### Failure Modes

-   Event backlog
-   Ordering bugs
-   Reprocessing challenges
-   Partial updates

------------------------------------------------------------------------

## Key Comparison

  Dimension             Cache + Replicas   Precomputed Views
  --------------------- ------------------ -------------------
  Complexity            Medium             High
  Latency               Very low           Extremely low
  Flexibility           High               Medium
  Consistency control   Fine-grained       Harder
  Best use case         General systems    Feeds, analytics

------------------------------------------------------------------------

## Key Learnings

-   Cache + replicas is the default starting point
-   CQRS is an evolution when scale/complexity increases
-   Consistency requirements dictate architecture
-   Precomputation trades write complexity for read performance

------------------------------------------------------------------------

## Intuition

-   Cache approach → avoid DB hits
-   CQRS approach → avoid computation entirely
