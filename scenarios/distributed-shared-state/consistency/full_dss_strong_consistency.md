# Distributed Shared State + Strong Consistency — HLD Master Cheat Sheet

---

# Section 1: Base Scenario — Distributed Shared State

## Definition
Multiple nodes/services need to read and write the same logical data, but the data is physically distributed across machines.

## Core Problem
- No shared memory
- No global clock
- Network is unreliable

## Why this exists
- Scale (too many users)
- Latency (geo-distribution)
- Fault tolerance

## Example
Seat booking:
Two users see same seat available → both try booking → race condition

## Assumptions (before constraints)
- Eventual consistency is acceptable
- Reads can be stale
- Conflicts are rare
- Availability preferred over correctness

---

# Section 2: Strong Consistency Constraint

## Definition
After a write completes, all future reads must see that write.

## Implications
- Global ordering required
- Coordination mandatory
- No stale reads
- Higher latency
- Reduced availability under partitions

## Broken assumptions
- No eventual consistency
- No stale reads
- Availability sacrificed

## Example
Bank:
Balance must reflect latest transaction always

---

# Section 3: Good Solutions

## 1. Leader-based (Primary Replica)

### How it works
- Single leader handles all writes
- Replicates to followers
- Waits for quorum before commit

### Pros
- Simple
- Strong ordering

### Cons
- Bottleneck
- Failover complexity

---

## 2. Consensus (Raft/Paxos)

### How it works
- Leader proposes
- Followers vote
- Majority decides

### Pros
- No single point of failure
- Strong correctness

### Cons
- Complex
- Higher latency

---

# Section 4: Bad Solutions

## 1. Just DB transactions
Fails across multiple nodes

## 2. Distributed locks
Locks require strong consistency themselves

## 3. Multi-leader writes
No global ordering

## 4. Async replication
Leads to data loss

## 5. Cache-based
Stale reads

---

# Section 5: Real-world mappings

## Banking
Shard per account + strong consistency per shard

## Booking
Shard per show + strict ordering

## Messaging
Strong consistency per chat only

## Config systems
Global strong consistency (small scale)

---

# Section 6: Constraint combinations

## Strong + Low latency
Impossible without tradeoffs

## Strong + High throughput
Solved via sharding

## Strong + Availability
Partition → availability sacrificed

## Strong + Multi-region
Requires global consensus (expensive)

---

# Final Principles

- Shard to scale
- Replicate to survive
- Consensus to ensure correctness

---
