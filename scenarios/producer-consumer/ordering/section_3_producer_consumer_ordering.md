# Section 3: Good Solutions (Producer-Consumer with Ordering Constraints)

## 🏆 Solution 1: Partitioned Queue (Per-key Ordering)

### Core Idea
Partition the queue using a key (e.g., user_id). Each partition maintains order and is consumed by one consumer.

### Architecture
Producers → Partitioner → [P1 | P2 | P3 | ... | Pn] → Consumers

- Key → hash → partition  
- Partition → single consumer → ordered processing  

### Example (Kafka-style)
- Topic with 10 partitions  
- user_id % 10 → partition  

All events for a given user go to the same partition, preserving order.

### Why this works
- Same key → same partition → order preserved  
- Different keys → different partitions → parallelism  

### Tradeoffs

#### Pros
- Scales horizontally  
- Maintains correctness (per-key ordering)  
- Industry standard  

#### Cons
- Hot key problem  
- Partition skew  
- Repartitioning is difficult and risky  

### Failure Modes

1. Consumer crash  
   - Partition stalls until rebalance  

2. Slow consumer  
   - Lag increases for that partition  

3. Poison message  
   - Blocks entire partition  

### What breaks first at scale
- Hot partitions → lag → SLA issues  

### Technologies
- Kafka  
- Kinesis  
- Pub/Sub  

---

## 🥈 Solution 2: Single Partition / Global Log (Global Ordering)

### Core Idea
All events go to a single partition/log to preserve total ordering.

### Architecture
Producers → [Single Queue / Log] → Consumer(s)

### Why this works
- One queue = one global order  
- No ambiguity in processing  

### Tradeoffs

#### Pros
- Simple mental model  
- Strong correctness guarantees  

#### Cons
- Throughput bottleneck  
- No real parallelism  
- High latency under load  
- Backpressure affects entire system  

### Failure Modes

1. Consumer slowdown  
   - Queue builds up → latency spikes  

2. Retry loops  
   - One failing event blocks all subsequent events  

3. System-wide impact  
   - Entire system affected, not localized  

### What breaks first
- Throughput ceiling and latency  

### Technologies
- Kafka (single partition)  
- SQS FIFO (single group)  
- DB-backed log (rare)

---

## Key Comparison

| Aspect            | Partitioned Queue        | Global Log              |
|------------------|------------------------|------------------------|
| Ordering         | Per-key                | Global                 |
| Scalability      | High                   | Very low               |
| Parallelism      | Yes                    | No                     |
| Failure impact   | Localized              | System-wide            |
| Use case         | Most real systems      | Rare, critical systems |

---

## Key Takeaways

- Partition = unit of ordering + unit of parallelism  
- Increasing partitions improves throughput, not per-key performance  
- Global ordering forces a single logical processing lane  
- Hot keys are fundamental bottlenecks under strict ordering  
- Always start with per-key partitioning and only move to global ordering if absolutely required
