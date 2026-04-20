# Section 4: Bad Solutions / Anti-Patterns

These are not stupid ideas — they can work — but are:
- hard to defend
- break at scale
- violate constraints subtly

---

## Anti-pattern 1: “Just scale the primary DB for reads”

### What it looks like
- Add bigger machines
- Vertical scaling
- Maybe some indexing

### Why it’s tempting
- Simple
- No architectural change
- Strong consistency preserved

### Why it’s wrong (in read-heavy systems)
You are treating a horizontal scaling problem with vertical scaling

### Problems:
1. Hard limit (CPU, RAM, IOPS)
2. Cost explosion
3. No isolation (reads compete with writes)
4. Single point of failure

### What breaks first
- CPU
- Disk I/O
- Connection limits

### Strong response
“Vertical scaling delays the problem but doesn’t solve it. Read-heavy systems require horizontal scaling via replicas or caching.”

---

## Anti-pattern 2: “Use read replicas everywhere” (blindly)

### What it looks like
- Add many replicas
- Route all reads to them

### Why it’s tempting
- Easy scaling
- Common practice

### Why it’s incomplete / dangerous
Replicas introduce replication lag

### Problems:
1. Stale reads
2. Read-your-writes broken
3. Causal violations

### What breaks first
- UX correctness
- User trust

### Strong response
“Replica lag causes stale reads. We need session-aware routing or fallback to primary.”

---

## Anti-pattern 3: “Cache everything with long TTL”

### What it looks like
- Aggressive caching
- Large TTL

### Why it’s tempting
- Massive performance boost
- Reduces DB load

### Why it’s dangerous
Trading correctness blindly for performance

### Problems:
1. Stale data
2. Write invisibility
3. Invalidation complexity ignored

### What breaks first
- Correctness-sensitive features
- User trust

### Strong response
“TTL-based caching is probabilistic. For correctness-sensitive paths, we need explicit invalidation or bypass strategies.”

---

## Anti-pattern 4: “Do joins/aggregation at read time”

### What it looks like
- Complex queries
- Multiple joins
- Sorting at query time

### Why it’s tempting
- Clean DB design
- Normalized schema

### Why it fails
You’re pushing computation into the hottest path

### Problems:
1. CPU heavy
2. Latency spikes
3. Poor horizontal scaling

### What breaks first
- Latency SLAs
- DB CPU

### Strong response
“At scale, we shift computation to write time via precomputed views.”

---

## Anti-pattern 5: “Force strong consistency everywhere”

### What it looks like
- Reads always from primary
- Sync replication
- Strict guarantees everywhere

### Why it’s tempting
- Easy correctness reasoning

### Why it’s bad
Kills scalability unnecessarily

### Problems:
1. High latency
2. Low throughput
3. Poor availability

### What breaks first
- Latency
- Throughput

### Strong response
“I’d scope strong consistency only to critical paths like payments.”

---

## Anti-pattern 6: “CQRS everywhere”

### What it looks like
- Event streams
- Multiple read models
- Heavy infra

### Why it’s tempting
- Sounds advanced

### Why it’s wrong
Unnecessary complexity

### Problems:
1. Operational overhead
2. Debugging difficulty
3. Eventual consistency everywhere

### What breaks first
- Team velocity
- Maintainability

### Strong response
“Start with cache + replicas. Move to CQRS only when needed.”

---

## Explicitly INVALID approaches

- Cache-only system (no DB authority)
- Fully async system for strong consistency use-case
- Random replica reads (no routing logic)

---

## Key Learning

Most bad solutions fail because they optimize one dimension while ignoring consistency implications.

There is no free lunch — only tradeoffs.
