# Section 1: Write-heavy Systems (Base Scenario)

---

## 1. What is a Write-Heavy System?

A write-heavy system is one where:

**Number of writes >> reads**, or writes are the primary business-critical operation.

Typical pattern:
- Continuous ingestion of data
- High throughput of incoming events
- Reads are secondary, often async or derived

---

## 2. When / Why Does This Scenario Occur?

This pattern appears when the system's core responsibility is capturing events or state transitions.

### Common categories:

#### a) Event Ingestion Systems
- Logs
- Telemetry
- Analytics pipelines

These systems continuously ingest large volumes of data.

#### b) Financial Systems
- Transactions
- Ledgers

Writes represent ground truth. Losing them is unacceptable.

#### c) Messaging Systems
- Chat systems
- Pub/Sub pipelines

Messages are written first and consumed later.

#### d) User Activity Streams
- Likes, clicks, impressions

These are write-dominant workloads with eventual read usage.

---

## 3. Core Characteristics of Write-Heavy Systems

### 3.1 Throughput Dominates Design

The system must handle extremely high write QPS:
- 100K to 1M+ writes/sec in large systems

Design decisions prioritize sustaining this throughput.

---

### 3.2 Writes Represent Source of Truth

Writes are not just operations — they define system state.

If a write is lost:
- State becomes incorrect
- Reconstruction is often impossible (e.g., payments)

---

### 3.3 Preference for Append-Only Patterns

Instead of updating existing data:

→ Systems prefer appending new records

Reasons:
- Sequential disk I/O (faster)
- Easier replication
- Better crash recovery
- Enables replayability

---

### 3.4 Backpressure Sensitivity

If the system slows down:
- Writes accumulate
- Queues fill up
- System may collapse

Hence buffering is critical:
- Queues (Kafka)
- In-memory buffers

---

### 3.5 Write Amplification Risks

Each logical write may trigger multiple physical writes:
- Replication
- Index updates
- WAL (Write Ahead Log)

This multiplies system load.

---

## 4. Typical Assumptions in Write-Heavy Systems

These assumptions are often made implicitly:

### Assumption A: Writes Can Be Buffered

Writes can be temporarily stored before persistence:
- Queues
- In-memory buffers

This smooths spikes.

---

### Assumption B: Eventual Persistence is Acceptable

A write may be acknowledged before it is fully durable.

Example:
- Async replication

---

### Assumption C: Some Data Loss is Acceptable

In systems like logs/metrics:
- Losing a small percentage (e.g., 0.01%) is acceptable

---

### Assumption D: Ordering Can Be Relaxed

Global ordering is not required.

Instead:
- Partition-level ordering is sufficient

---

### Assumption E: Latency vs Durability Tradeoff Exists

Faster writes often sacrifice durability guarantees.

---

## 5. Mental Model

Think of the system as a pipeline:

Producers → Load Balancer → Buffer/Queue → Storage → Replication → Consumers

This model highlights:
- Decoupling
- Backpressure handling
- Asynchronous processing

---

## 6. Common Mistakes in Interviews

### Mistake 1: Treating it like CRUD system

Incorrect:
- Designing like a standard DB system

Correct:
- Treat it as ingestion/streaming system

---

### Mistake 2: Ignoring Ingestion Pressure

Not planning for spikes leads to failure.

---

### Mistake 3: Over-Prioritizing Strong Consistency

Strong consistency reduces throughput drastically.

---

### Mistake 4: Avoiding Append-Only Design

Using update-heavy models hurts scalability.

---

## 7. Optional Deep Dive Q&A

### Q1: Why are append-only systems easier to scale?

Answer:
- Sequential writes are faster than random writes
- Reduce disk seek overhead
- Simplify replication (copy logs instead of states)
- Enable replay for recovery/debugging

---

### Q2: Why is Kafka log-based instead of DB-based?

Answer:
- Optimized for sequential writes
- Partitioning enables horizontal scaling
- Log retention allows replay
- Traditional DBs struggle with sustained high write throughput

---

### Q3: What breaks first in write-heavy systems?

Answer:
Typically:
1. Disk I/O (due to write saturation)
2. Network bandwidth (during replication)
3. Memory (if buffers grow due to backpressure)

---

## Key Takeaways

- Write-heavy systems are ingestion-first systems
- Writes define system truth
- Append-only patterns are dominant
- Throughput and durability trade-offs drive design decisions
- Buffering and backpressure handling are critical