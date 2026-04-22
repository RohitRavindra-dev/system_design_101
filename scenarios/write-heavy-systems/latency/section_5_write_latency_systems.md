# Section 5: Real-world mappings (Write-heavy + Write-latency-sensitive systems)

---

## 🟢 1. WhatsApp / Chat systems (canonical example)

### What actually happens (simplified but accurate)

Sender → Chat server → Append to log (durable)  
                       → Ack to sender  
                       → Fanout to recipients  
                       → Store in DB / inbox  

### Which solution?

Mostly Solution A (WAL / log-first)

### Why this choice?

- Massive write volume (messages)
- Need fast ack (user UX)
- Can tolerate:
  - slight delay in delivery
  - eventual consistency

### Important nuance

“Ack” ≠ “delivered”

WhatsApp shows:
- Sent (server ack)
- Delivered (recipient got it)

Meaning:
- Ack is based on durable ingestion, not delivery

### Tradeoffs they accept

- Latency: Very low  
- Durability: High (log-backed)  
- Consistency: Eventual  
- Ordering: Per chat  

### Failure behavior

- If consumer lag:
  - Messages delayed
- But:
  - Not lost (log persists)

### Strong interview statement

Chat systems typically acknowledge after durable ingestion into a log, not after delivery, which allows low latency while ensuring no message loss.

---

## 🟡 2. Twitter / Post creation

### Write path

User → API → Log (Kafka)  
             → Ack  
             → Async:  
                  - DB write  
                  - Feed fanout  
                  - Indexing  

### Which solution?

Solution A (WAL-first)

### Why?

- Extremely high write throughput
- Fanout-heavy system (followers)
- Cannot block on:
  - DB writes
  - feed generation

### Key insight

Posting is cheap (log append),  
fanout is expensive (async)

### Tradeoffs

- Latency: Low  
- Durability: High  
- Read consistency: Eventual  
- Complexity: High async pipeline  

### Failure mode

- Feed delay
- Not post loss

### Strong interview statement

Social media systems decouple write acknowledgment from fanout using a log-based architecture.

---

## 🔵 3. Uber / Ride booking

### Write path (simplified)

User → API → Stateful service  
             → Write to DB (or replicated store)  
             → Ack  

### Which solution?

Closer to Solution B (replicated state system)

### Why NOT Kafka-first?

Because:
- This is state mutation, not event logging

You need:
- Current driver availability
- Matching logic
- Transactions

Requires:
- read-after-write consistency

### Tradeoffs

- Latency: Low  
- Consistency: Stronger  
- Durability: High  
- Throughput: Lower than Kafka systems  

### Failure mode

- If quorum not reached:
  - write fails
  - user retry needed

### Strong interview statement

Ride booking requires strong consistency and immediate state visibility, so a replicated state store is preferred over a log-first approach.

---

## 🔴 4. Payments systems

### Write path

User → API → Transaction service  
             → Write to DB (ACID / quorum)  
             → Ack  

### Which solution?

Solution B (or stricter: DB + consensus)

### Why?

- Cannot lose writes
- Cannot have duplicates
- Requires:
  - atomicity
  - consistency

### Tradeoffs

- Latency: Higher (acceptable)  
- Durability: Maximum  
- Consistency: Strong  
- Throughput: Lower  

### Key insight

Payments prioritize correctness over latency

### Strong interview statement

Financial systems prioritize strong consistency and durability, so they accept higher latency instead of using async log-based ingestion.

---

## 🟣 5. Metrics / Logging systems

### Write path

Agents → Ingestion service → Kafka/log  
                             → Ack  
                             → Async storage (TSDB)  

### Which solution?

Pure Solution A

### Why?

- Massive write volume
- Loss is tolerable (sometimes)
- No strict read-after-write

### Tradeoffs

- Latency: Very low  
- Durability: Medium–High  
- Consistency: Eventual  
- Accuracy: Sometimes approximate  

### Strong interview statement

Observability systems are optimized for ingestion throughput, often using log-based pipelines with relaxed consistency.

---

## Pattern Recognition

### Use Solution A (WAL/Kafka) when:

- Event-driven system
- Append-heavy
- Can tolerate eventual consistency
- Fanout / async processing needed

Examples:
- Chat
- Social media
- Logs

---

### Use Solution B (replicated state) when:

- Current state matters
- Read-after-write required
- Strong correctness needed

Examples:
- Payments
- Ride booking
- Inventory

---

## Final Mental Model

Event systems → Log-first (Solution A)  
State systems → Replicated DB (Solution B)

---

# Optional Deep Dive Prompts (with answers)

## Q1. Can WhatsApp use Solution B instead?

Yes, but it would significantly reduce scalability and increase complexity. Chat systems benefit from log-based ingestion because of high write throughput and relaxed consistency requirements. Using Solution B would introduce unnecessary coordination overhead.

---

## Q2. Can Uber use Kafka-first?

Not for the critical write path. Kafka can be used for event streaming (e.g., analytics, logs), but the core ride-booking flow requires immediate consistency and state correctness, which Kafka alone cannot provide.

---

## Q3. Hybrid systems: where both are used?

Many real-world systems combine both approaches:
- Kafka (Solution A) for ingestion, event streaming, async processing
- Replicated DB / state store (Solution B) for serving queries and maintaining current state

Example:
- Chat system:
  - Kafka for message ingestion
  - DB for message retrieval
