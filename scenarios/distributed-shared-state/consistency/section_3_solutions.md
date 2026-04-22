# Section 3 --- Strong Consistency (Gold Version, High-Fidelity)

## 1. Core Solutions

### Solution 1: Single Leader (Primary)

Core Idea: One node (leader) decides order of writes. All writes go
through leader.

Flow: 1. Client → Leader 2. Leader assigns log order 3. Replicates to
followers 4. Waits for quorum 5. Commits and responds

Why it works: Single ordering authority eliminates conflicts.

Tradeoffs: - Leader bottleneck - Increased latency due to replication -
Failover complexity

Failure Modes: - Leader crash before quorum → write lost - Network
partition → must stop writes to avoid split brain

------------------------------------------------------------------------

### Solution 2: Consensus (Raft/Paxos)

Core Idea: Nodes agree on writes using quorum.

Flow: 1. Client → Leader 2. Leader appends log 3. Sends to followers 4.
Majority ACK 5. Commit

Why it works: Majority overlap guarantees correctness.

Tradeoffs: - Higher latency - Complexity - Lower throughput

------------------------------------------------------------------------

## 2. Detailed Request Flow (Concrete)

Initial: balance = 100

Requests: - Withdraw 30 (W1) - Withdraw 50 (W2) - Read (R1)

Execution: 1. Leader orders W1 then W2 2. W1 replicated → quorum →
commit → balance = 70 3. W2 replicated → quorum → commit → balance = 20

Reads: - Leader → 20 (correct) - Synced follower → 20 (correct) -
Lagging follower → 70 (incorrect)

Insight: Writes = agreement\
Reads = freshness guarantee

------------------------------------------------------------------------

## 3. Critical Clarifications (Your Doubts Resolved)

### Doubt 1: "If leader handles everything, why replicas?"

Clarification:

Replicas exist NOT just for durability, but because:

1.  Fault tolerance: Leader crash → replica takes over

2.  Consensus: Replicas vote → ensure correctness

3.  Read scaling (optional)

Key shift: Replication (copying) ≠ Consensus (agreement)

------------------------------------------------------------------------

### Doubt 2: "Doesn't strong consistency kill latency/throughput?"

YES. This is fundamental.

Strong consistency requires coordination → slower writes.

This is NOT a bug, it's a tradeoff.

------------------------------------------------------------------------

### Doubt 3: "Why distribute state if we must coordinate anyway?"

Because distribution is required for:

-   Scale (reads/writes)
-   Latency (geo)
-   Fault tolerance

But distribution introduces inconsistency → must be fixed via
coordination.

------------------------------------------------------------------------

### Doubt 4: "Sharding vs Replication confusion"

Important distinction:

Replication: Same data on multiple nodes → needs coordination

Sharding: Different data on different nodes → avoids coordination

------------------------------------------------------------------------

### Correct Architecture Pattern:

Shard first → then replicate per shard

Example:

Shard1: Leader + Followers (Raft)

Shard2: Leader + Followers (Raft)

No global coordination across shards.

------------------------------------------------------------------------

### Doubt 5: "Why not consensus across shards?"

Because: - Extremely high latency - Unnecessary (data independent)

Only needed for cross-shard transactions.

------------------------------------------------------------------------

## 4. Tradeoff (Core Insight)

Strong Consistency vs Performance:

-   Strong consistency → coordination → latency ↑ throughput ↓
-   High throughput → avoid coordination
-   Low latency → avoid cross-node sync

Golden rule:

Shard to avoid coordination\
Replicate to survive failures\
Consensus to enforce correctness

------------------------------------------------------------------------

## 5. Optional Deep Dive Q&A

### Why quorum = majority?

Ensures overlap between any two quorums → prevents conflicting commits.

### How leader election works?

Nodes vote using term numbers. Majority elects leader. Old leaders step
down.

### Can we scale writes?

Not within a shard. Use sharding.

------------------------------------------------------------------------

## 6. Final Mental Model

Leader = ordering authority\
Followers = replicas + voters\
Log = source of truth\
Quorum = correctness guarantee

System goal: Behave like ONE machine despite being distributed

------------------------------------------------------------------------

## 7. One-line Interview Answer

"We shard data to scale, replicate for fault tolerance, and use
leader-based or consensus protocols within each shard to enforce strong
consistency, accepting higher latency as a tradeoff for correctness."
