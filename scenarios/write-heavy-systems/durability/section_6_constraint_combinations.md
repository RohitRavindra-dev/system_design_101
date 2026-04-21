# Section 6: Constraint Combinations (Write-heavy systems + High Durability)

------------------------------------------------------------------------

## Combination 1: Durability + Low Latency

### Conflict

-   Durability → wait for replication\
-   Low latency → ACK early

Direct conflict.

------------------------------------------------------------------------

### What happens if you try both?

#### Option A: ACK after replication (safe)

-   Durable\
-   Higher latency

#### Option B: ACK before replication

-   Low latency\
-   Data loss possible

------------------------------------------------------------------------

### Real-world solutions

1.  Batching
    -   Group multiple writes\
    -   Replicate in batches
2.  Pipelining
    -   Overlap replication
3.  Tunable durability
    -   acks=1 (fast, unsafe)\
    -   acks=all (safe, slower)

------------------------------------------------------------------------

### Which solutions survive?

-   Kafka/log → best fit\
-   Quorum DB → slower

------------------------------------------------------------------------

### Key insight

You cannot get both strict durability and ultra-low latency. Mitigation
techniques include batching and pipelining.

------------------------------------------------------------------------

## Combination 2: Durability + High Throughput

### Conflict

-   Durability → replication overhead\
-   Throughput → minimal coordination

------------------------------------------------------------------------

### What breaks first?

Disk I/O and network bandwidth.

------------------------------------------------------------------------

### Solutions

1.  Partitioning\
2.  Sequential writes (WAL)\
3.  Compression and batching

------------------------------------------------------------------------

### Which solutions survive?

-   Kafka/log → excellent\
-   Quorum DB → bottlenecked

------------------------------------------------------------------------

### Key insight

Partitioned logs scale horizontally; quorum systems scale per shard.

------------------------------------------------------------------------

## Combination 3: Durability + Ordering

### Conflict

Global ordering is expensive and limits scalability.

------------------------------------------------------------------------

### Solution

Partition-level ordering.

------------------------------------------------------------------------

### Tradeoff

-   Scalable\
-   No global ordering

------------------------------------------------------------------------

### Which solutions survive?

-   Kafka/log → partition ordering\
-   Quorum DB → global ordering possible but slower

------------------------------------------------------------------------

### Key insight

Ordering is scoped to partition/key to maintain scalability.

------------------------------------------------------------------------

## Combination 4: Durability + Exactly-once Semantics

### Problem

Retries cause duplicates.

------------------------------------------------------------------------

### Why harder with durability

Retries are unavoidable and replication may duplicate work.

------------------------------------------------------------------------

### Solutions

1.  Idempotency keys\
2.  Transactional writes\
3.  Exactly-once pipelines

------------------------------------------------------------------------

### Which solutions survive?

-   Kafka/log → possible with complexity\
-   Quorum DB → easier

------------------------------------------------------------------------

### Key insight

Exactly-once is an end-to-end problem.

------------------------------------------------------------------------

## Combination 5: Durability + Availability (CAP)

### Conflict

Network partition forces a tradeoff.

------------------------------------------------------------------------

### Choices

-   CP → reject writes\
-   AP → risk inconsistency

------------------------------------------------------------------------

### Mapping

-   Kafka (strict ISR) → CP-ish\
-   Raft DB → CP\
-   Async systems → AP

------------------------------------------------------------------------

### Key insight

Zero data loss requires sacrificing availability under partition.

------------------------------------------------------------------------

## Combination 6: Durability + Multi-region

### Problem

Cross-region latency vs durability.

------------------------------------------------------------------------

### Options

1.  Sync replication → safe but slow\
2.  Async replication → fast but risky\
3.  Hybrid → local quorum + async global

------------------------------------------------------------------------

### Which solutions survive?

-   Kafka → complex\
-   Spanner-like DB → designed for this

------------------------------------------------------------------------

### Key insight

Multi-region durability introduces latency vs safety tradeoff.

------------------------------------------------------------------------

## Final Synthesis

Once zero data loss is required, systems must use synchronous durability
mechanisms like WAL and replication. Every additional constraint
introduces tradeoffs.

------------------------------------------------------------------------

## Impossible Triangle

-   Zero data loss\
-   Ultra-low latency\
-   High availability under partition

All three cannot be achieved simultaneously.

------------------------------------------------------------------------

## Mental Model

1.  Identify workload (write-heavy)\
2.  Add constraint (durability)\
3.  Identify broken assumptions\
4.  Choose architecture (log vs quorum vs hybrid)
