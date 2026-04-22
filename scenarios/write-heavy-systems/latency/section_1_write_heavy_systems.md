# Section 1: Base Scenario — Write-heavy systems

## 1. What is the base scenario?

A write-heavy system is one where:
- The dominant operation = writes (inserts/updates)
- Reads are either:
  - Less frequent OR
  - Can tolerate delay/staleness

Typical shape:
Write QPS >> Read QPS

---

## 2. When / why does this happen?

This shows up when systems are capturing events, not serving queries.

### (A) Event ingestion systems
- Logs, analytics, telemetry
- Example:
  - Clickstream tracking
  - Metrics ingestion (CPU, latency, etc.)

### (B) User-generated content systems
- Chat messages
- Tweets/posts
- Comments

### (C) State mutation systems
- Financial transactions
- Order placement
- Ride booking

---

## 3. Core intuition (this is key for interviews)

In write-heavy systems:
“The system’s primary job is to not lose writes and absorb spikes.”

Not optimize reads.

---

## 4. What assumptions do these systems normally rely on?

### Assumption 1: You can buffer writes
- Writes don’t need to hit DB instantly
- You can queue/batch them

---

### Assumption 2: Eventual consistency is acceptable
- Reads may lag behind writes
- Example:
  - Tweet appears after 1–2 sec
  - Metrics dashboard delayed

---

### Assumption 3: Write latency can be slightly relaxed
- User doesn’t need instant acknowledgment
- You can:
  - Accept → queue → process later

---

### Assumption 4: Writes are append-heavy
- Mostly inserts, not random updates
- Enables:
  - Log-based systems
  - Sequential disk writes

---

### Assumption 5: Ordering is often relaxed
- Strict ordering is not always required
- Or only required per key (e.g., per user)

---

## 5. What does a “default architecture” look like?

Client → Load Balancer → Write Service  
                      → Queue (Kafka)  
                      → Async consumers → DB  

Why this is the default:
- Absorbs spikes
- Decouples ingestion from storage
- Smooths load

---

## 6. Key mental model

Think of write-heavy systems as:

“Firehose → Buffer → Processing pipeline → Storage”

---

## 7. Where candidates go wrong

Common weak thinking:
- “Just write directly to DB”
- “Add more DB replicas”
- “Scale reads”

They are solving the wrong problem.

---

# Deep-dive Q&A (from optional prompts)

## Q1. Why is sequential write (log-based) faster than random write?

Sequential writes:
- Avoid disk seek time
- Are append-only → very cache/disk friendly
- Enable batching and OS-level optimizations

Random writes:
- Require disk head movement (seek)
- Cause fragmentation
- Much slower under high throughput

---

## Q2. Why does Kafka scale better than DB for ingestion?

- Kafka is log-based → sequential disk writes
- Partitioning allows horizontal scaling
- Pull-based consumers → avoids overload
- Designed for high-throughput append workloads

Databases:
- Handle indexing, constraints → overhead per write
- Often involve random I/O
- Harder to scale writes horizontally

---

## Q3. What happens if we remove the queue layer?

- DB becomes the immediate bottleneck
- Write spikes directly hit DB → overload/failure
- No buffering → poor resilience
- No decoupling → tight coupling between ingestion and storage
- Increased latency variance

Result:
System becomes fragile under load and hard to scale
