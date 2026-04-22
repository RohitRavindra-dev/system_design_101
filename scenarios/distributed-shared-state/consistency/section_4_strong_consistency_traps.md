# Section 4 — Bad Solutions / Interview Traps (Strong Consistency)

---

## Trap 1 — “Just use DB transactions”

### Why it’s tempting
You already know:
- ACID
- Row locks
- Serializable isolation

So you think:
“Transactions prevent race conditions → done”

### Why it fails
Transactions only work **within a single database instance**

They do NOT handle:
- Multiple replicas accepting writes
- Cross-region databases
- Network partitions

### Failure scenario
Region A DB → withdraw ₹30  
Region B DB → withdraw ₹50  

Both succeed locally.

Final state diverges ❌

### What you'd need to force-fit this
- Distributed transactions (2PC)

### Why 2PC is bad
- Blocking (coordinator failure = stuck system)
- High latency
- Poor fault tolerance

### Broken assumption
“There is a single source of truth”

### Strong answer
“Transactions work within a single node, but distributed systems require coordination across nodes, which transactions alone don’t provide.”

---

## Trap 2 — “Use distributed locks (Redis/ZooKeeper lock)”

### Why it’s tempting
Feels like Java:
“Just lock before writing”

### Why it fails
Locks themselves require:
**Strong consistency to be correct**

### Failure scenario
- Redis lock acquired in region A
- Network partition
- Region B doesn’t see lock

Two writers proceed ❌

### What you'd need to fix it
- Strongly consistent lock service (ZooKeeper/etcd)

Which internally uses consensus

### Key insight
You didn’t solve the problem  
You just moved it

### Broken assumption
“Lock service is always correct and visible”

### Strong answer
“Distributed locks themselves require strong consistency guarantees, so they don’t eliminate the need for consensus—they rely on it.”

---

## Trap 3 — “Allow multiple leaders (multi-master)”

### Why it’s tempting
- Better latency
- Better write throughput
- Each region writes locally

### Why it fails
Concurrent writes → no global ordering

### Failure scenario
Region A: withdraw 30 → balance = 70  
Region B: withdraw 50 → balance = 50  

Conflict with no deterministic resolution ❌

### What you'd need to fix it
- Conflict resolution

But for banking/booking → not resolvable

### Broken assumption
“Writes are independent / mergeable”

### Strong answer
“Multi-leader setups improve latency but introduce conflicting writes, which cannot be resolved for strict consistency use cases like financial transactions.”

---

## Trap 4 — “Async replication is fine”

### Why it’s tempting
- Fast writes
- Common in MySQL/Postgres setups

### Why it fails
Leader acknowledges BEFORE replicas are updated

### Failure scenario
T1: Write success (leader only)  
T2: Leader crashes  
T3: Replica promoted (missing write)  

Data loss ❌

### Broken assumption
“Replication will eventually catch up safely”

### Strong answer
“Async replication sacrifices consistency because writes may be acknowledged before being safely replicated.”

---

## Trap 5 — “Cache + DB will handle it”

### Why it’s tempting
- Redis is fast
- Widely used

### Why it fails
Cache introduces:
- stale reads
- race conditions

### Failure scenario
T1: DB updated → A1 booked  
T2: Cache still shows AVAILABLE  
T3: Another user books → double booking ❌

### What you'd need
- Cache invalidation + coordination

Leads back to consensus

### Broken assumption
“Cache is always in sync with DB”

### Strong answer
“Caches are eventually consistent and cannot guarantee correctness for strong consistency requirements.”

---

## Trap 6 — “Just retry on conflict”

### Why it’s tempting
- Optimistic concurrency
- Works in some systems

### Why it fails
If both succeed initially:
Damage already done

Example:
- 2 bookings confirmed

### Broken assumption
“Conflicts are reversible”

### Strong answer
“Retries help with transient failures but don’t prevent conflicting commits, which violate strong consistency.”

---

## Trap 7 — “Read from replicas for performance”

### Why it’s tempting
- Improves read throughput
- Common practice

### Why it fails
Replicas can be stale

### Failure scenario
Follower lag → stale read → wrong decision ❌

### Broken assumption
“Replicas are always up-to-date”

### Strong answer
“For strong consistency, reads must either go through the leader or ensure the replica is up-to-date.”

---

# Final Synthesis

All bad solutions fail because they violate:

- No global ordering → multi-leader
- No agreement → async replication
- No freshness guarantee → cache / replica reads
- No fault-safe coordination → weak distributed locks

---

# Optional Deep Dive Q&A

### Q1. Why quorum = majority?
A majority ensures overlap between any two quorums, preventing conflicting commits.

### Q2. What if leader lies?
Consensus assumes non-Byzantine faults. Byzantine systems require more complex protocols (PBFT).

### Q3. Can we scale writes?
Yes, via sharding. Each shard has its own consensus group.

---

# Clarifications from Discussion

- Replicas are not just for durability; they participate in consensus.
- Strong consistency requires coordination, which increases latency.
- Sharding reduces need for coordination by partitioning data.
- Strong consistency and high availability/low latency are fundamentally in tension.
