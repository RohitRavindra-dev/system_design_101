# Write-heavy Systems + High Durability (Zero Data Loss) — HLD Cheat Sheet

---

# SECTION 1: Base Scenario — Write-heavy Systems

## Definition
A write-heavy system is one where writes dominate reads and are the primary source of truth.

Examples:
- Log ingestion (Kafka)
- Payments (ledger updates)
- Messaging systems
- Analytics pipelines

## Core Characteristics
- High write throughput (100K–1M writes/sec)
- Append-heavy (sequential writes)
- Writes = source of truth
- Backpressure sensitive

## Mental Model
Producer → Buffer → Storage → Consumers

## Key Assumptions
- Writes can be buffered
- Eventual persistence is okay
- Some data loss acceptable
- Ordering relaxed
- Latency prioritized over durability

---

# SECTION 2: Constraint — Failure Tolerance (Zero Data Loss)

## Definition
Every acknowledged write must survive failures.

## Guarantees
- ACK only after durability
- Survive node, disk, AZ failures
- Replayability required

## Broken Assumptions
- No async persistence
- No data loss tolerance
- Buffer must be durable
- Ordering often stricter

## Architectural Implications
- WAL required
- Replication required
- Quorum/ISR needed
- Idempotency required

---

# WAL (Deep Dive)

## What
Append-only log written BEFORE applying changes.

## Why
Ensures recovery after crashes.

## Analogy
Flight recorder — reconstruct everything after crash.

## Key Property
WAL-first → DB-second

---

# ISR (Kafka)

## Definition
Set of replicas fully caught up with leader.

## Why
Avoid waiting for slow replicas while preserving durability.

## Guarantee
Only ISR replicas eligible for leadership.

---

# Multi-AZ Failure

## Definition
Entire datacenter failure.

## Solution
Replicate across AZs.

## Tradeoff
Higher latency due to cross-AZ replication.

---

# SECTION 3: Strong Solutions

---

## Solution 1: Replicated Commit Log (Kafka)

### Flow
Producer → Leader → Append to log → Replicate → ACK → Consumers

### Why it works
- Log = WAL
- Sequential writes
- Replication ensures durability

### Tradeoffs
- Higher latency
- Operational complexity

### Failure Modes
- ISR shrink
- Disk bottlenecks

---

## Solution 2: Quorum-based DB (Raft)

### Flow
Client → Leader → WAL → Replicate → Quorum → Commit → ACK

### Why it works
- Majority guarantees durability
- Strong consistency

### Tradeoffs
- Lower throughput
- Leader bottleneck

---

# SECTION 4: Bad Solutions / Traps

## Trap 1: Single DB + WAL
Fails on machine/AZ failure.

## Trap 2: Async replication
ACK before replication → data loss.

## Trap 3: Queue only
In-memory → data loss.

## Trap 4: Fan-out writes
No coordination → inconsistency.

## Trap 5: Retry-based durability
Retries don’t fix post-ACK loss.

## Trap 6: Cache as primary store
Redis may lose data.

## Trap 7: Increase replicas only
Without correct ACK semantics → useless.

---

# SECTION 5: Real-world Mapping

## Kafka
Event ingestion, logs

## Payments
Kafka + DB hybrid

## Distributed DBs
Raft-based systems (etcd, Spanner)

## Messaging
Durable enqueue before delivery

## CDC
Logs for audit and replay

---

# SECTION 6: Constraint Combinations

## Durability + Low Latency
Conflict → solved via batching, pipelining

## Durability + Throughput
Partitioning required

## Durability + Ordering
Partition-level ordering

## Durability + Exactly-once
Idempotency + transactions

## Durability + Availability
CAP tradeoff → choose CP

## Durability + Multi-region
Latency vs safety tradeoff

---

# FINAL TAKEAWAY

All systems converge to:

Producers → Durable Log → Processing → State Store

Key principle:
ACK must imply durability.

If ACK lies → system is broken.
