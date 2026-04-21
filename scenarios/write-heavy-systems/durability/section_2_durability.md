# Section 2: Failure Tolerance (High Durability, Zero Data Loss)

## What this constraint means

In a write-heavy system:

**Every accepted write must survive failures. No data loss.**

This means:
- Acknowledgement (ACK) must imply durability
- Writes cannot be acknowledged before being safely persisted and replicated

---

## System Guarantees Implied

### 1. ACK = Durability Guarantee
- ACK only after:
  - write is persisted to disk
  - and replicated (typically)

### 2. Survive Multiple Failures
System must tolerate:
- Node crash
- Machine crash
- AZ (datacenter) failure
- Network partition

### 3. Replayability Required
- Writes must be logged
- Ordered
- Recoverable

---

## Broken Assumptions from Write-heavy Systems

### Assumption A: Writes can be buffered
❌ Broken — in-memory buffering is unsafe  
✅ Need durable buffers (disk-backed logs)

---

### Assumption B: Eventual persistence is okay
❌ Broken — cannot delay persistence  
✅ Must persist synchronously before ACK

---

### Assumption C: Some data loss is acceptable
❌ Broken — zero tolerance

---

### Assumption D: Ordering can be relaxed
⚠️ Partially broken  
- Need ordering (at least per partition/key)

---

### Assumption E: Latency vs durability tradeoff
❌ Broken — must prioritize durability  
- Leads to higher latency and lower throughput

---

## Architectural Implications

### 1. Write-Ahead Logging (WAL)
- Write intent first to durable log
- Then apply to database/state

### 2. Replication before ACK
- Single node durability insufficient
- Need multi-replica writes

### 3. Append-only Systems
- Easier recovery
- Deterministic replay

### 4. Idempotency
- Required due to retries

### 5. Quorum / Consensus
- Required for correctness across replicas

---

## WAL (Write-Ahead Log)

### Definition
Write is first appended to a durable log before applying to database.

### Flow
producer → node → WAL (disk) → DB update → ACK

### Why it works
- Prevents data corruption on crash
- Enables deterministic recovery via replay

### Recovery
- Read WAL
- Replay operations

### Key properties
- Append-only
- Sequential writes
- Fast

---

## ISR (In-Sync Replicas in Kafka)

### Definition
Replicas that are fully caught up with the leader.

### Write Flow
1. Producer → leader
2. Leader writes to log
3. Followers replicate
4. ACK only after ISR replication

### Why ISR exists
- Some replicas lag
- Need balance between availability and durability

### Leader Failure
- New leader chosen only from ISR

### Guarantee
- No data loss if ACK was given

---

## Kafka Replication Model

- Each partition has multiple replicas
- Leader handles reads/writes
- Followers continuously replicate (not idle)
- Pull-based replication

---

## Multi-AZ Failure

### Definition
Entire datacenter failure

### Risk
If all replicas in one AZ → total data loss

### Solution
Distribute replicas across AZs

### Tradeoffs
- Increased latency
- Cross-AZ network cost

### Requirement
ACK after cross-AZ replication

---

## Key Tradeoffs

| Goal | Impact |
|------|-------|
| High throughput | prefers async |
| High durability | requires sync |
| Low latency | prefers early ACK |
| Zero data loss | forces late ACK |

---

## Optional Deep Dive Questions + Answers

### Q1: What happens if node crashes after ACK but before replication?
If ACK was given before replication, and node crashes, data is lost.  
Hence ACK must wait for replication.

---

### Q2: Why is WAL fundamental?
Because it ensures all writes are recorded durably before applying changes.  
Without it, recovery after crash is impossible or inconsistent.

---

### Q3: Why quorum is better than leader-only durability?
Leader-only durability fails if leader crashes before replication.  
Quorum ensures multiple copies exist before ACK → no data loss.

---

## Additional Clarifications (From Discussion)

### Where does WAL sit?

WAL is inside the node, before DB:

producer → node → WAL → DB → ACK

---

### Kafka replicas — what do followers do?

- Maintain full copy of partition log
- Continuously replicate from leader
- Stay in sync for failover
- Not idle — active replication

---

### Final Mental Model

System becomes:

producer → durable log → replication → ACK → consumers

---

## Summary

- Durability shifts system design fundamentally
- WAL + replication + quorum are core primitives
- ACK semantics define correctness
- Tradeoff: performance sacrificed for correctness

---

Generated on: 2026-04-21T13:09:48.564271 UTC
