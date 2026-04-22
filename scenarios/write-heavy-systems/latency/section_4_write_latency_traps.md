# Section 4: Bad Solutions / Interviewer Traps

These are NOT stupid ideas — they’re tempting, and that’s why interviewers use them to test depth.

---

## ❌ Trap 1: “Just write directly to the database and scale it”

### Why it’s tempting
- Simple mental model
- DB gives:
  - durability
  - indexing
  - querying

Feels like:
“Why add Kafka/Redis complexity?”

### Why it fails here

#### ❌ 1. Write latency becomes unpredictable (p99 killer)
- DB writes involve:
  - index updates
  - locks
  - random I/O
  - transaction overhead

Tail latency spikes significantly.

#### ❌ 2. No burst absorption
- Write-heavy system = spiky traffic
- DB cannot absorb spikes like a buffer

Result:
- timeouts
- dropped requests

#### ❌ 3. Vertical scaling ceiling
- IOPS limits
- connection limits

### What if you try to force-fit it?
You’ll end up adding:
- write batching
- connection pooling
- sharding
- write queues → which essentially recreates a log/queue system

### Why it breaks assumptions
- Assumes DB can act as both ingestion + storage
- But DB is not optimized for ingestion spikes

### Key takeaway
Databases are poor write buffers — they don’t handle bursty ingestion well due to locking and random I/O.

---

## ❌ Trap 2: “Ack immediately after receiving request (before durability)”

### Why it’s tempting
- Ultra low latency
- Simple

### Why it fails

#### 💥 Data loss on crash

Flow:
Receive → Ack → Crash → Data gone

### Where candidates mess up
They say:
“We can retry”

But:
- Client thinks success
- No retry happens

### What it would take to force-fit
- Idempotency keys
- Client retries
- Reconciliation jobs

Adds significant complexity.

### Why it breaks assumptions
- Violates durability requirement

### Key takeaway
Acknowledging before durability risks permanent data loss, which is unacceptable in most write-critical systems.

---

## ❌ Trap 3: “Use Kafka but ack before replication (acks=1)”

### Why it’s tempting
- Faster than acks=all
- Still “kind of durable”

### Why it fails

#### 💥 Leader crash = data loss

Flow:
Write → leader ack → leader crashes → data not replicated → lost

### Where it might be acceptable
- Logs / analytics
- Non-critical data

### Why it’s dangerous in interviews
For systems like:
- chat
- payments
- user data

This is a red flag.

### Key takeaway
acks=1 improves latency but weakens durability guarantees — only acceptable for non-critical data.

---

## ❌ Trap 4: “Batch writes aggressively to improve throughput”

### Why it’s tempting
- Higher throughput
- Common optimization

### Why it fails here

#### ❌ Increases latency
Batching introduces:
- wait time
- queue delay

### Tradeoff
Throughput ↑  ↔  Latency ↑

### When it can work
- Micro-batching (very small windows)
- Not large batching

### Why it breaks assumptions
- Violates low latency requirement

### Key takeaway
Batching improves throughput but conflicts with strict latency SLAs.

---

## ❌ Trap 5: “Use cache (Redis) but without replication”

### Why it’s tempting
- Extremely fast
- Simple

### Why it fails

#### 💥 Single point of failure
Write → Redis → Ack → Redis crashes → data lost

### Why candidates fall for it
They assume speed = correctness

### What it takes to fix
- Replication
- Persistence (AOF)
- Failover

Which essentially converts it into a replicated system (Solution B).

### Key takeaway
A single-node cache sacrifices durability — replication or persistence is required.

---

## ❌ Trap 6: “Synchronous cross-region replication before ack”

### Why it’s tempting
- Maximum durability
- Geo redundancy

### Why it fails

#### 💥 Latency explosion
Cross-region RTT:
- 100–300 ms

Violates strict latency requirements (<50ms)

### Real-world approach
- Async cross-region replication
- Not in critical write path

### Key takeaway
Cross-region replication should not be in the critical path for low-latency systems.

---

## ❌ Trap 7: “Strong consistency everywhere (global ordering, transactions)”

### Why it’s tempting
- Clean mental model
- No edge cases

### Why it fails

#### 💥 Throughput and latency collapse
- Distributed transactions (2PC)
- Global locks
- Coordination overhead

### Reality
- Systems scope consistency:
  - per key
  - per partition
  - or relax it entirely

### Key takeaway
Global strong consistency increases coordination cost and hurts latency — avoid unless absolutely required.

---

## ⚔️ Meta Insight

All bad solutions fail because they violate one of:

Latency ↔ Durability ↔ Throughput ↔ Availability

You cannot optimize all four simultaneously.

---

# Optional Deep Dive Prompts (with answers)

## 1. What is fsync and why is it expensive?
fsync forces data from OS buffers to disk. It is expensive because it requires a physical disk flush, which involves latency from disk I/O and prevents batching optimizations.

## 2. How does Kafka guarantee durability internally?
Kafka writes messages to a commit log on disk and replicates them across brokers. With acks=all, data is acknowledged only after replication to in-sync replicas, ensuring durability.

## 3. What is quorum write mathematically?
A quorum requires majority agreement:
quorum = floor(N/2) + 1

This ensures that any two quorums overlap, preserving consistency and preventing split-brain issues.
