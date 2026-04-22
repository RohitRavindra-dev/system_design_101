# Section 3: Good Solutions (Write-heavy + Write-latency-sensitive)

---

## 🥇 Solution 1: Write-Ahead Log (WAL) / Commit Log as First-Class Storage

### Idea
Acknowledge the write after appending it to a durable, sequential log (fast), not after full DB processing.

### Architecture
Client → API Server → Append to WAL (Kafka / log / disk)  
→ Ack immediately  
→ Async consumers → DB / indexing / fanout

### Why this works
Sequential append to a log is the fastest durable write:
- Disk is slow for random writes
- Disk is fast for sequential writes

We avoid:
- Index updates
- Random IO
- Transaction overhead

### What you achieve
- Low latency (fast append)
- Durability (persisted log)
- High throughput (sequential writes)
- Decoupling (async consumers)

### Technologies
- Apache Kafka
- Apache Pulsar
- Database WALs (PostgreSQL, MySQL)

### Ack semantics
Ack after:
- Write to log
- (Optionally) replication

### Tradeoffs / gotchas
1. Not fully processed → data not yet in DB  
2. Read-after-write inconsistency  
3. Log is critical infrastructure  
4. Ordering only within partitions  

### Failure modes
- Log saturation (disk/network bottleneck)
- Consumer lag → eventual consistency delay
- Replication latency spikes

### Assumptions
- Buffering restored
- Durability satisfied
- Low latency satisfied
- Immediate consistency relaxed

---

## 🥈 Solution 2: In-Memory + Replicated Write Path (Primary + Replicas)

### Idea
Write to in-memory system and replicate synchronously before ack.

### Architecture
Client → API → Primary (in-memory)  
→ Sync replicate to replicas  
→ Ack after quorum  
→ Async persist to DB (optional)

### Why this works
Memory is fast; replication gives durability.

### What you achieve
- Low latency (memory write)
- Durability (replication)
- Availability (multiple replicas)

### Technologies
- Redis (with replication)
- DynamoDB
- Cassandra

### Ack semantics
Ack after quorum replication (e.g., 2/3 nodes)

### Tradeoffs / gotchas
1. Memory not inherently durable  
2. Network bottleneck  
3. Write amplification  
4. Leader election complexity  

### Failure modes
- Network partition → quorum issues
- Hot partitions
- Replica lag (if async)

### Persistence options
1. Async flush to DB  
2. Use system as DB (Cassandra/Dynamo)  
3. Hybrid (Redis + DB)

### Assumptions
- Low latency satisfied
- Durability via replication
- Buffering reduced
- Consistency stronger (config dependent)

---

## ⚔️ Comparison

| Dimension | WAL / Kafka | In-Memory + Replication |
|----------|------------|------------------------|
| Latency | Low | Very low (depends on quorum) |
| Durability | Strong (disk) | Medium (replication) |
| Throughput | Very high | High |
| Complexity | Medium | High |
| Consistency | Eventual | Stronger |

---

## 🔥 Key Insight

Kafka/WAL is not a query system:
- No indexing
- No efficient querying
- Retention limits
- No relational constraints

Correct model:
Kafka = ingestion layer  
DB = query layer

---

## 🧠 Optional Deep Dive Q&A

### Q1: When would you NOT use Kafka?
- When strong consistency required
- When low latency + read-after-write is critical
- When query complexity is high

### Q2: Can we combine both solutions?
Yes:
- Kafka for ingestion
- In-memory store for fast reads

### Q3: How does WhatsApp likely do this?
- Write to log (durable)
- Fanout via queues
- Store in DB for retrieval

### Q4: What does “acks=all” mean in Kafka?
- Leader waits for all replicas before ack
- Increases durability, increases latency

### Q5: Why can Kafka be system of record?
- When reads are simple or replay-based
- When data can be derived by sequential processing
