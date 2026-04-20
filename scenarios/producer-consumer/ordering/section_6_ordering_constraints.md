# Section 6: Constraint Combinations (Producer-Consumer + Ordering)

## Combo 1: Ordering + Delivery Semantics (at-least-once / exactly-once)

### What changes?
You must handle:
- Duplicates
- Retries
- Idempotency

### Problem
With per-key ordering:
E1 → E2 → E3

If E1 is retried:
- It may execute twice
- Must not break order or correctness

### What works
- Partitioned queue (still valid)
- Add idempotent consumers
- Deduplication using event_id or sequence numbers

### What breaks
- Skipping failed messages → breaks ordering
- Processing out of order → risky

### Insight
Ordering + retries = idempotency becomes mandatory


## Combo 2: Ordering + Low Latency

### What changes?
You care about:
- Queue delay
- Processing delay

### Problem
Queues introduce buffering and waiting

Example:
Event waits 100ms in queue → SLA breach

### What works
- Smaller partitions
- Faster consumers
- Careful prioritization

### What breaks
- Batch processing (adds latency)
- Deep queues (delay explosion)

### Insight
Ordering + low latency = tension with buffering


## Combo 3: Ordering + High Throughput

### What changes?
You want:
- Very high event rates

### Problem
Ordering restricts parallelism

### What works
- Partitioning (per-key)
- Smart key design

### What breaks
- Global ordering → not scalable
- Hot keys → bottleneck

### Insight
Throughput scales with number of independent keys


## Combo 4: Ordering + Fault Tolerance

### What changes?
You must handle:
- Consumer crashes
- Partial processing

### Problem
If consumer dies mid-processing:
E1 processed?
E2 started?

### What works
- Offset commits (Kafka-style)
- At-least-once + idempotency

### What breaks
- Exactly-once without system support
- Manual offset handling

### Insight
Fault tolerance + ordering = careful checkpointing


## Combo 5: Ordering + Multi-region / Distributed Systems

### What changes?
- Multiple producers across regions

### Problem
Clock skew and network delay:
Region A → E1
Region B → E2
Which came first?

### What works
- Per-key ordering
- Logical clocks / sequence numbers

### What breaks
- Global ordering → requires consensus (Raft/Paxos)

### Insight
Global ordering + distributed system = consensus problem


## Final Mental Model

### Hierarchy of difficulty
No ordering < Per-key ordering << Global ordering

### Scaling rule
Scale = number of independent keys

### Core tradeoff
More ordering → less parallelism
