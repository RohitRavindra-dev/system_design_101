# Write-Heavy Systems + Low Latency Writes (HLD Cheat Sheet)

## Section 1: Base Scenario — Write-Heavy Systems

### Definition
A write-heavy system is one where write operations dominate reads.

Example ratio:
- Writes: 100K/sec
- Reads: 5K/sec

### Where it appears
- Chat systems (messages)
- Social media (posts)
- Logging/metrics pipelines
- Financial transactions

### Core goal
> Absorb large volumes of writes without losing data.

---

### Default Architecture

Client → API → Queue (Kafka) → Async Consumers → DB

### Why this works
- Queue absorbs spikes
- Async processing smooths load
- DB not overloaded

---

### Key Assumptions

1. Writes can be buffered
2. Eventual consistency is acceptable
3. Latency is not critical
4. Writes are append-heavy
5. Ordering can be relaxed

---

### Mental Model

Firehose → Buffer → Processing → Storage

---

## Section 2: Constraint — Low Write Latency

### Definition
Writes must be acknowledged quickly (typically <10–50 ms)

---

### Types of ACK

1. Weak: received in memory
2. Durable: persisted safely
3. Strong: fully processed

---

### Key Insight

> ACK must be fast AND durable → inherent tension

---

### What breaks

- Cannot rely purely on async queues
- Cannot delay writes
- DB now in critical path (or equivalent durability layer)

---

### Implications

- Need fast durability
- Must optimize p99 latency
- Must avoid slow storage in critical path

---

## Section 3: Good Solutions

---

### Solution A: WAL / Kafka-first

#### Flow
Client → API → Kafka → ACK → Async DB

#### Why it works
- Sequential writes → fast
- Durable log
- High throughput

#### Tradeoffs
- Eventual consistency
- Consumer lag
- Ordering only per partition

#### Failure modes
- Broker overload
- Consumer lag buildup

---

### Why not just Kafka?

Kafka cannot:
- Query efficiently
- Support indexes
- Enforce constraints
- Store data indefinitely

Hence:
> Kafka = ingestion layer  
> DB = query layer

---

### Solution B: In-Memory + Replication

#### Flow
Client → API → Leader → Replicate → ACK

#### Why it works
- Memory is fast
- Replication provides durability

#### Tradeoffs
- Network overhead
- Leader election complexity
- Not disk-first durability

---

## Section 4: Bad Solutions

---

### 1. Direct DB writes

Fails due to:
- Random I/O
- Locking
- Poor burst handling

---

### 2. ACK before durability

Leads to:
- Data loss on crash

---

### 3. Kafka with weak ACK (acks=1)

Risk:
- Leader crash = data loss

---

### 4. Aggressive batching

Tradeoff:
- Throughput ↑, Latency ↑

---

### 5. Single-node cache

Risk:
- Total data loss

---

### 6. Sync cross-region writes

Problem:
- Latency explodes (100–300ms)

---

### 7. Global ordering

Problem:
- Kills scalability

---

## Section 5: Real-world Systems

---

### Chat (WhatsApp)

- Log-first architecture
- ACK on durable write
- Async delivery

---

### Twitter

- Kafka ingestion
- Async fanout
- Eventual consistency

---

### Uber

- Stateful system
- Strong consistency
- DB/quorum writes

---

### Payments

- Strong consistency
- ACID writes
- Higher latency acceptable

---

### Metrics systems

- Kafka ingestion
- High throughput
- Approximate accuracy acceptable

---

## Section 6: Combining Constraints

---

### + Strong Consistency

→ Use Solution B  
→ Kafka becomes weak

---

### + Exactly-once

→ Need idempotency  
→ Deduplication

---

### + Ordering

→ Use partitioned ordering  
→ Avoid global ordering

---

### + High Availability

→ Favor Kafka (Solution A)

---

### + Multi-region

→ Avoid sync replication  
→ Use async replication

---

### + Extreme Throughput

→ Kafka dominates

---

## Tradeoff Summary

Latency ↔ Durability ↔ Consistency ↔ Availability

You cannot maximize all.

---

## Final Mental Model

Event Systems → Kafka (log-first)  
State Systems → Replicated DB

---

## Key Interview Lines

- "Kafka is optimized for sequential writes, not querying."
- "Durability and latency are inherently in tension."
- "Global ordering does not scale."
- "Exactly-once is achieved via idempotency."

