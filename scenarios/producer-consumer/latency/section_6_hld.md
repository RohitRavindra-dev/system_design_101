# Section 6: Constraint Combinations (Latency-focused Systems)

## Constraint 1: Latency + Ordering

Strict ordering reduces parallelism. To preserve order, systems often serialize processing per key (e.g., user_id, order_id), which directly conflicts with low-latency and high-throughput goals.

### Challenges:
- Reduced parallelism
- Head-of-line blocking
- Increased latency due to enforced ordering
- Partitioning complexity

### Solutions:
- Partitioned streams (e.g., Kafka partitions)
- Key-based routing (same key → same partition)

### Tradeoff:
Scalability is sacrificed for correctness.

---

## Constraint 2: Latency + Exactly-once processing

Systems must avoid duplicates and missed events.

### Challenges:
- Coordination overhead
- Increased latency

### Solutions:
- Idempotent consumers
- Deduplication via event IDs
- Transactional processing (Kafka EOS)

### Tradeoff:
Higher latency due to coordination.

---

## Constraint 3: Latency + High load (spiky traffic)

### Challenges:
- Sudden spikes overwhelm system
- Autoscaling too slow

### Solutions:
- Pre-warmed capacity
- Load shedding (fail fast)
- Priority handling

### Tradeoff:
Must reject or degrade instead of buffering.

---

## Constraint 4: Latency + Fault tolerance

### Challenges:
- Retries can amplify load
- Risk of cascading failures

### Solutions:
- Bounded retries
- Dead-letter queues
- Circuit breakers

### Tradeoff:
Reliability mechanisms can hurt latency.

---

## Constraint 5: Latency + Multi-region

### Challenges:
- Network latency dominates
- Consistency issues

### Solutions:
- Region-local processing
- Async replication

### Tradeoff:
Consistency sacrificed for latency.

---

## Final Insight

Every additional constraint reduces degrees of freedom in the system. Latency conflicts with buffering, retries, ordering, and coordination. System design becomes a balance of opposing forces.
