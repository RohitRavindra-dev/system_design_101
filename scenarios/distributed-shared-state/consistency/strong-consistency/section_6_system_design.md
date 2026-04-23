# Section 6 — Constraint Combinations (Comprehensive Notes)

## First Principle
Strong consistency already forces coordination. Adding more constraints increases system complexity and tradeoffs.

---

## 1. Strong Consistency + Low Latency (Geo-distributed)

### Problem
Users are globally distributed and require low latency (<50ms) while maintaining strong consistency.

### Why difficult
Strong consistency requires coordination (quorum, leader), which introduces cross-region network latency.

### Valid Solutions
- Single region leader: Simple but high latency for distant users.
- Spanner-like systems: Use tightly synchronized clocks and global consensus.

### Insight
Latency must be sacrificed for correctness.

---

## 2. Strong Consistency + High Throughput

### Problem
System must handle large number of writes per second.

### Why difficult
Each write requires coordination → limits throughput.

### Solution
Shard aggressively:
- Each shard runs its own consensus group.
- Parallelize writes across shards.

### Insight
Scale via partitioning, not vertical scaling.

---

## 3. Strong Consistency + High Availability (Partitions)

### CAP Theorem
Cannot have Consistency, Availability, and Partition tolerance simultaneously.

### Behavior
- Majority partition continues.
- Minority partition rejects requests.

### Insight
Correctness > Availability.

---

## 4. Strong Consistency + Multi-Region Writes

### Problem
Users write from multiple regions.

### Why difficult
Concurrent writes across regions require global agreement.

### Solutions
- Route all writes to a single leader.
- Use global consensus (Spanner).

### Insight
Either accept latency or complexity.

---

## Incompatible Constraints

- Strong consistency + independent local writes → impossible
- Strong consistency + async replication → impossible
- Strong consistency + zero latency → impossible

---

## Solution Validity

### Single Leader
- Poor latency globally
- Bottleneck under high throughput

### Consensus (Raft)
- Correct under partitions
- Needs sharding for scale

---

## Final Synthesis

Strong consistency introduces coordination overhead affecting latency, throughput, and availability. Systems minimize scope by applying consistency within shards.

---

## Ultimate Design Approach

1. Define consistency scope
2. Shard accordingly
3. Apply consensus per shard
4. Avoid cross-shard coordination

---

## Optional Deep Dive Answers

### Why quorum = majority?
Ensures overlap between any two quorums, preventing conflicting decisions.

### How does leader election work?
Nodes vote; majority wins; ensures single leader.

### Can we scale writes?
Yes, via sharding. Each shard handles its own writes.

---

## Clarifications

- Strong consistency and low latency conflict fundamentally.
- Replication is required not just for durability but for consensus and failover.
- Kafka/WAL cannot replace consensus for correctness-critical systems.

