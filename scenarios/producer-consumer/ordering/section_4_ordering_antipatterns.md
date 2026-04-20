# Section 4: Bad Solutions / Anti-Patterns (Ordering Constraints)

## User Understanding / Position
User takeaway:
- Choose a good key
- Increase partitions
- Accept that hot keys fundamentally limit throughput if ordering is strict

---

## Anti-pattern 1: Shared Queue with Multiple Consumers

### Why it’s tempting
- Simple
- Scales easily
- Works for general queue problems

### What goes wrong
Queue: [E1(user_123), E2(user_123)]

C1 picks E1  
C2 picks E2  

E2 may finish before E1 → Ordering broken

### Root problem
Queue does not enforce per-key routing

### Interview defense
A shared queue with competing consumers cannot guarantee ordering because messages for the same key can be processed concurrently by different consumers.

### What it would take to fix
- Add per-key locking or routing
- Essentially reinvent partitioning

---

## Anti-pattern 2: Distributed Locks per Key

### Why it’s tempting
- Ensures one consumer per key

### What goes wrong
- High coordination overhead
- Lock contention for hot keys
- Failure complexity (deadlocks, expiry issues)
- Throughput collapse

### Interview defense
Distributed locking introduces high coordination overhead and effectively serializes processing with additional latency.

---

## Anti-pattern 3: Consumer-side Reordering

### Why it’s tempting
- Buffer and sort events

### What goes wrong
- Need sequence numbers and coordination
- Buffer can grow unbounded
- Missing events block processing indefinitely

### Interview defense
Consumer-side reordering shifts complexity downstream and introduces memory and failure handling issues.

---

## Anti-pattern 4: Skip Failed Messages and Retry Later

### Why it’s tempting
- Improves throughput

### What goes wrong
E1 fails → skip  
E2 processed → ordering violated

### Interview defense
Skipping failed messages breaks ordering guarantees; ordered systems must process sequentially.

---

## Anti-pattern 5: Dynamic Rebalancing of Hot Keys

### Why it’s tempting
- Feels like load balancing

### What goes wrong
user_123 split across consumers → parallel execution → ordering broken

### Interview defense
Rebalancing a hot key violates ordering since events may execute concurrently.

---

## Anti-pattern 6: Using DB as Queue with ORDER BY

### Why it’s tempting
- SQL ordering is easy

### What goes wrong
- DB bottleneck
- High contention
- Poor scalability

### Interview defense
Database queues do not scale well and introduce contention despite ordering guarantees.

---

## Explicitly Invalid Approaches

- Increasing consumers to fix ordering
- Random distribution of events
- Parallel batch processing

---

## Core Insight

All bad solutions try to increase parallelism without respecting ordering constraints.

Golden rule:
If ordering is required → concurrency is constrained

---

## Final Interview Insight

If strict ordering + high throughput for same key:
- You must either optimize processing
- Or relax constraints

There is no free scaling.
