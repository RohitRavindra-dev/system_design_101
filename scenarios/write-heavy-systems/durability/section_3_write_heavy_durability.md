# Section 3: Strong Solutions — Write-heavy systems with high durability (no data loss)

---

## Solution 1 (Best): Replicated Commit Log (Kafka-style)

### Core Idea
Make the system itself a durable, replicated, append-only log and treat it as the source of truth.

---

## Correct Mental Model

Kafka IS the WAL.

There is no:
Kafka → WAL → consumers

Correct:
Kafka partition log = WAL = source of truth

Consumers read directly from this log.

---

## Architecture

Producers → Leader Broker → Append to partition log (WAL)
           → Replicate to ISR
           → ACK
           → Consumers read from log

---

## How it works

### Partitioning
- Data split into partitions
- Each partition = independent log
- Enables horizontal scaling

---

### Write Flow

1. Producer sends request to leader
2. Leader appends to log (WAL)
3. Followers replicate (pull model)
4. Once replication condition is satisfied (ISR/quorum)
5. ACK sent

---

### Read Flow

- Consumers read from log
- Offset-based consumption
- Replayable

---

## Why this works

- WAL ensures durability
- Replication ensures fault tolerance
- Sequential writes ensure high throughput
- Log enables replay and recovery

---

## Failure Scenarios

### Leader crashes after ACK
- Data present in ISR
- New leader elected
- No loss

### Leader crashes before ACK
- No ACK sent
- Producer retries
- No inconsistency

### Leader crashes after local write but before replication and ACK sent incorrectly
- Data loss possible
- Prevented by waiting for ISR before ACK

### All replicas crash before replication
- Data lost
- No ACK should have been sent

---

## ISR Shrinking Problem

- ISR can shrink due to lagging replicas
- If ISR = 1 → single point of failure

### Kafka Protection

min.insync.replicas

Example:
replication.factor = 3
min.insync.replicas = 2

- If ISR < 2 → writes rejected
- Availability sacrificed for durability

---

## Catch-up Mechanism

- Followers pull from leader
- Fetch missing offsets
- Append to log
- Rejoin ISR when caught up

---

## Tradeoffs

- Higher latency due to replication
- Operational complexity
- Storage growth
- Consumer offset management

---

## Solution 2: Quorum-based Distributed DB (Raft/Paxos)

### Core Idea

Use consensus-based replication with quorum commits.

---

## Architecture

Client → Leader
       → WAL write
       → Replication to followers
       → Quorum ACK
       → Commit
       → Apply to DB
       → ACK

---

## Key Flow

- WAL is written first
- WAL replicated
- Quorum achieved
- Then state applied

---

## Why this works

- Strong consistency
- Majority durability
- Safe leader election

---

## Tradeoffs

- Lower throughput
- Higher latency
- Leader bottleneck
- Less horizontally scalable

---

## Failure Modes

- Leader failure → re-election
- Network partition → system unavailable if quorum lost
- Slow followers → impact commit latency

---

## Key Insight

Both systems follow:

Replicated log → derive state

Kafka:
- Log is exposed

Raft DB:
- Log is internal

---

# Optional Deep Dive Answers

### Why Kafka avoids consensus per write but still gives durability?
Kafka avoids full consensus (like Raft) per write to reduce latency and increase throughput. Instead, it uses leader + ISR replication. Durability is ensured by requiring replication to in-sync replicas before ACK. It trades strict consensus guarantees for performance while still preventing acknowledged data loss.

---

### Why quorum = N/2 + 1?
To ensure overlap between any two majorities. This guarantees that any committed write is present in at least one node in the next quorum, preventing data loss and ensuring consistency.

---

### When does Kafka become unsafe compared to Raft?
Kafka becomes unsafe when:
- ACK is sent before sufficient replication
- ISR shrinks too much
- min.insync.replicas is misconfigured
- Multiple failures happen before replication

Raft provides stronger guarantees due to strict quorum-based commits.
