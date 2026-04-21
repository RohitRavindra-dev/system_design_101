# Section 4: Bad / Tempting Solutions (Interviewer Traps)

---

## Trap 1: “Just use a single database with WAL — that gives durability”

### Why it’s tempting

- WAL = durability (true)
- Databases are “reliable” (sounds safe)
- Simple mental model

---

### Why it’s wrong here

WAL only protects against **process/node crash**, not **machine/AZ failure**

---

### Failure scenario

Write → WAL (disk) → ACK  
Machine dies → disk gone → data lost

---

### What would it take to force-fit?

- Add replication  
- Add failover  
- Add quorum  

You basically reinvent:
- Kafka OR  
- Raft DB  

---

### Why it fails under constraint

Constraint requires surviving multi-node / AZ failure.  
Single-node WAL only provides local durability.

---

### Clean interview response

“WAL alone only gives crash recovery, not fault tolerance. We need replication before ACK.”

---

## Trap 2: “Async replication is fine — we’ll replicate after ACK”

### Why it’s tempting

- Low latency  
- High throughput  
- Common in eventually consistent systems  

---

### Why it breaks

Write → local write → ACK → replication pending  
Leader crashes → followers don’t have data → data lost after ACK

---

### Why this violates constraint

You acknowledged something that is not durable.

---

### What it would take to fix

- Delay ACK until replication completes  
→ becomes quorum/ISR model

---

### Clean interview response

“Async replication violates durability guarantees because a leader crash can lose acknowledged writes.”

---

## Trap 3: “We’ll use a queue (like RabbitMQ / in-memory buffer)”

### Why it’s tempting

- Natural for write-heavy systems  
- Buffers absorb spikes  

---

### Why it’s dangerous

Queues may:
- store data in memory  
- lack replication  

---

### Failure scenario

Write → in-memory queue → ACK → node crashes → queue gone → data lost

---

### What it would take to fix

- persistent log  
- replication  
- replay  

→ effectively Kafka

---

### Clean interview response

“Queues help with buffering but without replication + durable storage they violate no-data-loss guarantees.”

---

## Trap 4: “Write to multiple DB replicas in parallel (fan-out writes)”

### Why it’s tempting

- “Just write to 3 DBs”  
- sounds like replication  

---

### Why it breaks

- No coordination  
- partial writes  
- inconsistent state  

---

### Failure scenario

DB1 success, DB2 fail, DB3 success → ACK → inconsistent system

---

### What it would take to fix

- 2PC (complex, slow)  
- consensus (Raft)

---

### Clean interview response

“Fan-out writes without a commit protocol lead to inconsistency and can’t guarantee durability.”

---

## Trap 5: “Rely on retries to ensure durability”

### Why it’s tempting

- “Client will retry”  

---

### Why it’s wrong

Retries only help before ACK, not after ACK.

---

### Failure scenario

Write → ACK → data lost internally → client unaware → no retry

---

### Clean interview response

“Retries handle failures before ACK, not after. Durability must be guaranteed before ACK.”

---

## Trap 6: “Use cache (Redis) as primary write store”

### Why it’s tempting

- Very fast  
- widely used  

---

### Why it fails

- async persistence  
- possible data loss  

---

### Failure scenario

Write → Redis → ACK → crash before persistence → data lost

---

### What it would take

- AOF with fsync=always  
- replication  
- still complex

---

### Clean interview response

“Caches optimize latency but are not suitable as primary durable storage under strict durability constraints.”

---

## Trap 7: “We’ll just increase replication factor”

### Why it’s tempting

- more replicas = safer  

---

### Why it’s incomplete

If ACK is before replication, replication factor doesn’t matter.

---

### Core issue

ACK semantics matter more than number of replicas.

---

### Clean interview response

“Replication factor only helps if ACK is tied to replication completion.”

---

# Meta Insight

All bad solutions fail because they violate:

“ACK must imply durability under failure”

---

# Optional Deep-Dive Questions + Answers

### 1. When is async replication acceptable?

- When slight data loss is acceptable  
- Example: analytics, logs, metrics  
- Not acceptable for payments, ledgers  

---

### 2. Why is 2PC avoided in high-scale systems?

- High latency (multiple round trips)  
- Blocking nature  
- Poor fault tolerance under coordinator failure  

---

### 3. How does Kafka avoid full consensus per write?

- Uses leader-based replication  
- No global agreement required  
- Relies on ISR + leader election for safety  
