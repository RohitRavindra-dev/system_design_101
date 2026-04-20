# Section 6: Constraint Combinations — Delivery Semantics

This section focuses on how delivery semantics interact with other system constraints.
The goal is to understand what breaks, what survives, and how designs evolve under pressure.

---

## 1. Delivery Semantics + Ordering

### Scenario
Events must be processed:
- In order
- Without loss (or controlled duplicates)

Example:
- Ride lifecycle
- Order state machine

### Problem
At-least-once + retries can break ordering.

Example:
Event 1 → processed  
Event 2 → processed  
Crash  
Event 2 → replayed  

Seems fine until:

Event 3 processed  
Event 2 retried late  

Result:
1 → 2 → 3 → 2 (state regression)

### Solution
- Partition by key (ride_id)
- Use monotonic versioning

Apply only if:
incoming_version > current_version

### Key Insight
Ordering + at-least-once requires protection against out-of-order retries.

---

## 2. Delivery Semantics + Low Latency

### Scenario
- Sub-100ms response
- Cannot afford retries or blocking

Example:
- Driver matching
- Real-time bidding

### Problem
Exactly-once introduces:
- Coordination
- Locking
- Increased latency

### Solution
- At-most-once OR relaxed at-least-once
- Accept trade-offs:
  - Missed events
  - Occasional duplicates

### Key Insight
Low latency systems sacrifice guarantees.

---

## 3. Delivery Semantics + High Throughput

### Scenario
- Millions of events per second

Example:
- Analytics
- Clickstream

### Problem
Exactly-once causes:
- Coordination overhead
- DB contention
- Throughput collapse

### Solution
- At-least-once
- Batch processing
- Eventual consistency

### Key Insight
Throughput and strict guarantees are natural enemies.

---

## 4. Delivery Semantics + External Side Effects

### Scenario
- External APIs (payments, email, SMS)

### Problem
Infra-level guarantees don’t extend to external systems.

### Solution
- Idempotency keys
- Deduplication
- Reconciliation jobs

### Key Insight
Exactly-once must be enforced at system boundaries.

---

## 5. Delivery Semantics + Failures / Retries

### Scenario
- Frequent crashes
- Repeated retries

### Problem
Infinite retries create poison message loops.

### Solution
- Retry with exponential backoff
- Dead Letter Queue (DLQ)

Flow:
Retry → Retry → Retry → DLQ

### Key Insight
Not all failures are recoverable.

---

## 6. Delivery Semantics + Scale (Partitioning)

### Scenario
- Horizontal scaling required

### Problem
Global ordering becomes impossible.

### Solution
- Partition by key
- Maintain ordering per key

Tradeoff:
More partitions → more parallelism  
Fewer partitions → better ordering  

### Key Insight
Ordering is local, not global.

---

## 7. Delivery Semantics + Consistency

### Scenario
- Multiple services updating state

### Problem
Strong consistency conflicts with async queues.

### Solution
- Eventual consistency
- Event-driven architecture

### Key Insight
Queues push systems toward eventual consistency.

---

# Final Synthesis

When constraints combine:

- Use partitioning + versioning for ordering
- Prefer at-least-once for throughput
- Use idempotency for correctness
- Use exactly-once effect only for critical paths

---

# Learnings

- You don’t choose semantics in isolation.
- Every additional constraint invalidates some solutions.
- At-least-once + idempotency is the default backbone.
- Exactly-once is a business decision, not a technical default.
