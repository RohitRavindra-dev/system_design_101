# Section 3 --- Good Solutions (Read-heavy systems with strict read-after-write consistency)

------------------------------------------------------------------------

## 🥇 Solution 1: Primary-read fallback (Read-your-writes routing)

### Core idea

Route reads to primary (or guaranteed-fresh source) when consistency is
required; use replicas/cache otherwise.

------------------------------------------------------------------------

### How it works

#### Write flow:

Client → Primary DB → ACK

#### Read flow:

If user just wrote: Client → Primary DB (bypass replicas/cache)

Otherwise: Client → Cache / Read replicas

------------------------------------------------------------------------

### How do we know "user just wrote"?

1.  Sticky session / short-lived flag\
    After write → mark user as "dirty" for \~5--30 seconds → route reads
    to primary

2.  Version / timestamp tracking\
    Client sends last_write_timestamp\
    If replica is behind → fallback to primary

3.  Read-your-write token\
    Server returns a write version\
    Client sends it in next read\
    System ensures read \>= version

------------------------------------------------------------------------

### Why this works

Guarantees correctness only where needed while preserving read
scalability.

------------------------------------------------------------------------

### Tradeoffs

Pros: - Scales well - Minimal disruption - Works with async replication

Cons: - Routing complexity - Higher latency for fresh reads - Needs
session/metadata

------------------------------------------------------------------------

### Failure modes

1.  Primary overload\
    Too many users in fresh-read state

2.  Incorrect routing logic\
    Bugs → stale reads

3.  Multi-device inconsistency\
    Write on phone, read on laptop (no context)

------------------------------------------------------------------------

### Tech choices

-   MySQL/Postgres (primary + replicas)
-   Redis (optional cache)
-   API layer routing logic
-   Redis/in-memory for recent write tracking

------------------------------------------------------------------------

## 🥈 Solution 2: Synchronous replication / quorum

### Core idea

Ensure replicas are up-to-date before acknowledging writes.

------------------------------------------------------------------------

### How it works

#### Write flow:

Client → Primary → Replicate to replicas → ACK after replication

#### Read flow:

Client → Any replica (safe)

------------------------------------------------------------------------

### Variants

1.  Fully synchronous replication\
2.  Quorum-based:
    -   Write when W replicas ack
    -   Read from R replicas
    -   Condition: R + W \> N

------------------------------------------------------------------------

### Why this works

Replicas are never stale → safe reads everywhere.

------------------------------------------------------------------------

### Tradeoffs

Pros: - Clean model - No routing logic - Strong guarantees

Cons: - High write latency - Lower availability - Poor for high write
systems

------------------------------------------------------------------------

### Failure modes

1.  Cross-region latency\
2.  Slow replica blocks writes\
3.  Network partitions (CAP tradeoff)

------------------------------------------------------------------------

### Tech choices

-   Cassandra (quorum)
-   Google Spanner
-   CockroachDB
-   Amazon Aurora

------------------------------------------------------------------------

## ⚖️ When to choose what

Use Primary-read fallback: - Read-heavy systems - Performance matters -
Example: social feeds, profiles

Use Sync replication: - Correctness critical - Example: payments,
banking

------------------------------------------------------------------------

## 🔥 Key insight

Most real systems: Use primary-read fallback by default and sync
replication only for critical paths.

------------------------------------------------------------------------

## Q&A / Reinforcement

Q: Why not always use sync replication?\
A: Write latency + availability hit makes it impractical for large-scale
read-heavy systems.

Q: Why not always route reads to primary?\
A: Primary becomes bottleneck → defeats purpose of read scaling.

Q: What breaks first at scale?\
A: Primary overload (Solution 1), write latency (Solution 2).

------------------------------------------------------------------------

## Learnings

-   Consistency vs performance is the core tradeoff
-   Read-heavy systems default to async + cache
-   Constraint forces selective correctness guarantees
-   Routing logic is often the practical solution in real systems
