# Write-Heavy Systems — Section 6 (Lossless Version) + Clarifications

---

# SECTION 6: COMBINING CONSTRAINTS (DETAILED, NON-HANDWAVY)

This section explores how additional constraints interact with:
- Write-heavy systems
- Low write latency requirements

The key idea:
Real systems are NOT defined by one constraint — they are intersections of multiple constraints.
Each added constraint:
- Breaks assumptions
- Invalidates some solutions
- Forces new tradeoffs

---

## CONSTRAINT COMBO 1: + STRONG CONSISTENCY (READ-AFTER-WRITE)

### Requirement
User must immediately see their own write.

### What breaks

Solution A (log-first / Kafka) breaks because:
- Writes are acknowledged after log append
- DB update is async
- Reads typically hit DB
→ DB may not have the latest write

Thus:
- Violates read-after-write guarantee

### Why this happens (deep reasoning)

Log-based systems decouple:
- ingestion (fast)
- materialization (slow)

Strong consistency requires:
- both ingestion and visibility to be synchronous

This introduces coordination.

### What survives

Solution B (replicated state system with quorum)

Flow:
Write → leader → replicate to quorum → ack  
Read → leader or quorum → latest state

### Tradeoffs introduced

- Latency increases (waiting for quorum)
- Availability decreases:
  - If quorum not reachable → writes fail
- Throughput decreases due to coordination

### Failure mode

- Network partition:
  - Minority partition → rejects writes
  - System becomes unavailable for writes

### Impossible combination

Strong consistency + low latency + multi-region

Reason:
- Strong consistency requires cross-region coordination
- Network RTT makes latency high

You must sacrifice:
- latency OR consistency

---

## CONSTRAINT COMBO 2: + EXACTLY-ONCE SEMANTICS

### Requirement

No duplicates, no missing writes.

### What breaks

Kafka/log-based systems are:
- At-least-once delivery

Thus:
- Consumers may reprocess messages
- Duplicates can occur

### Why this happens

Failures lead to:
- retries
- replays

Kafka guarantees:
- durability
- ordering (per partition)

But NOT:
- exactly-once by default

### What works

Two approaches:

1. Idempotency
   - Each request has unique key
   - Duplicate writes ignored

2. Deduplication layer
   - Store processed IDs

3. Transactions (advanced)
   - Kafka transactions
   - DB transactional writes

### Tradeoffs

- Additional storage (dedup tables)
- Additional latency
- Complexity in pipelines

### Strong insight

Exactly-once is usually:
- implemented at application level
- not guaranteed by infrastructure

---

## CONSTRAINT COMBO 3: + ORDERING GUARANTEES

### Requirement

Writes must be processed in order.

### What breaks

Global ordering:
- Requires single serialization point
- Becomes bottleneck

### Why

Ordering implies:
- coordination
- sequencing

Global sequencing:
→ kills scalability

### What works

Partitioned ordering (per key)

Example:
- chat_id → partition
- ordering guaranteed per chat

### Tradeoffs

- Hot partitions:
  - popular keys overload single partition
- Uneven load distribution

### Failure mode

- Celebrity problem:
  - one key dominates traffic

---

## CONSTRAINT COMBO 4: + HIGH AVAILABILITY

### Requirement

System should accept writes even during failures.

### What breaks

Quorum-based systems:

- If quorum not reachable:
  → writes are rejected

Thus:
- availability drops

### What works

Log-first systems:

- Can accept writes locally
- Replicate asynchronously

### Tradeoff

Availability ↑  
Consistency ↓

### Why

No coordination required on critical path

---

## CONSTRAINT COMBO 5: + MULTI-REGION

### Requirement

Low latency for global users.

### What breaks

- Single-region architecture
- Synchronous cross-region writes

### Why

Network RTT between regions:
- 100–300 ms

Violates latency SLA.

### What works

Option 1:
- Region-local writes
- Async replication

Option 2:
- Partition users by region

### Tradeoffs

- Eventual consistency across regions
- Conflict resolution required

### Impossible combination

Low latency + strong consistency + multi-region

---

## CONSTRAINT COMBO 6: + ULTRA HIGH THROUGHPUT

### Requirement

Millions of writes per second.

### What breaks

- Quorum systems
- Coordination-heavy systems

### Why

Each write requires:
- network round trips
- synchronization

Limits throughput.

### What works

Log-based systems (Kafka)

### Why (deep reasoning)

Kafka aligns with hardware:

- Sequential disk writes
- Partition-level parallelism
- Efficient replication

---

# CLARIFICATIONS

---

## RELATIONSHIP BETWEEN CORE PROPERTIES

### 1. Latency vs Durability

Stronger durability requires:
- disk flush (fsync)
- replication

→ increases latency

---

### 2. Availability vs Consistency (CAP)

Under partition:

Choose:
- Availability → accept writes
- Consistency → reject writes

---

### 3. Latency vs Consistency

Consistency requires:
- coordination
- quorum

→ increases latency

---

### 4. Throughput vs Latency

Batching:
- increases throughput
- increases latency

---

### 5. Multi-region impact

Adds:
- network latency
- coordination complexity

---

### 6. Durability vs Availability

If durability requires quorum:
- failure → quorum unavailable
→ system rejects writes

---

# WHY KAFKA ENABLES HIGH THROUGHPUT (DETAILED)

---

## Key Principle

Sequential writes are faster than random writes.

---

## Disk Behavior

Random writes:
- require disk seek
- slow

Sequential writes:
- contiguous
- no seek
- fast

---

## Kafka Design Advantages

### 1. Append-only log

All writes:
append → append → append

---

### 2. Partitioning

Each partition:
- independent
- parallel

Total throughput:
sum of partition throughput

---

### 3. Batching

Producers:
- batch writes
- reduce overhead

---

### 4. OS Page Cache

- reduces disk I/O overhead
- improves performance

---

### 5. Sequential Replication

Followers:
- replicate log sequentially
- efficient

---

## Kafka Limits

- Disk bandwidth
- Network bandwidth
- Hot partitions
- Broker limits (CPU, memory)

---

## Scaling Kafka

- Increase partitions
- Add brokers
- Improve partitioning strategy
- Tune replication
- Multi-cluster setups

---

## Key Insight

Kafka does not remove limits.

It shifts bottlenecks to:
- hardware limits (disk + network)

Which are much higher than:
- DB limits (locks + random I/O)

---

# DEEP DIVE PROMPTS (Q&A)

---

## Q1: Why is sequential write faster than random write?

Sequential writes avoid disk seeks and operate on contiguous blocks, allowing much higher throughput compared to random writes which require repositioning the disk head.

---

## Q2: Why does partitioning increase throughput?

Each partition operates independently, enabling parallel processing across multiple brokers. Total throughput scales approximately linearly with the number of partitions until hardware limits are reached.

---

## Q3: Why does quorum replication hurt latency but not necessarily throughput?

Latency increases per request because each write must wait for multiple replicas. However, throughput remains high because multiple writes can be processed in parallel across different partitions or nodes.
