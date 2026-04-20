# SECTION 3 — Good Solutions (Ranked)

---

## Solution 1 (BEST): Precompute + In-Memory Serving (Hot Path Only)

Core idea:
Do all heavy work beforehand. At request time → only do O(1) in-memory lookup + minimal logic.

### Architecture

[Async pipeline]
Kafka → Stream processing → Precompute → Store in Redis / memory

[Serving path]
Request → In-memory lookup → Response

---

### Example (Ad Bidding)

Instead of:
- fetching user profile
- computing features
- running heavy models

You precompute:
- user segments
- predicted CTR
- bidding scores

Store in:
- Redis / in-memory KV

At request:
- lookup → simple math → respond in <10ms

---

### Why it works

Because you:
- eliminate runtime computation
- eliminate DB queries
- eliminate network hops (ideally)

You’ve turned:
“compute-heavy problem” → “lookup problem”

---

### Key techniques

- Precomputed views / materialized data
- Feature stores (for ML systems)
- In-memory KV stores:
  - Redis
  - Aerospike
  - in-process cache

---

### Tradeoffs

1. Staleness
- Data is not real-time
- You’re serving slightly outdated info

2. Pipeline complexity
- Need async pipelines (Kafka, Flink, Spark)

3. Memory cost
- Storing everything in RAM is expensive

---

### Failure modes at scale

1. Cache miss
- Falls back to DB → SLA blown instantly

Solution:
- NEVER fallback synchronously  
- Return default / drop request

2. Hot keys
- Popular users / items overload single shard

Fix:
- replication
- sharding
- request spreading

3. Pipeline lag
- Precomputed data becomes too stale

---

### When to use

- Ad systems
- Recommendation systems (real-time serving)
- Gaming state snapshots
- Trading signals (partially)

---

### Interview line

“For hard latency SLAs, I’d move all heavy computation out of the request path and serve from precomputed in-memory data structures to guarantee bounded latency.”

---

## Solution 2: Single-Hop, Minimal Dependency Architecture (Inline Compute, but Tight)

Core idea:
If you must compute at request time → keep the path extremely short and predictable

### Architecture

Client → Stateless service (co-located compute + data) → Response

Constraints:
- ≤ 1 network hop
- ≤ 1 data access (preferably memory)
- no fanout

---

### Example (Trading Order Matching)

Order book:

BUY orders (bids)       SELL orders (asks)
₹100 → 10 shares        ₹101 → 5 shares
₹99  → 20 shares        ₹102 → 10 shares

Order book is live in-memory state.

When order comes:
- system matches buy/sell instantly
- executes trade

---

### System design

Partitioning:
- Each stock handled by one node

Inside node:
- Order book in memory
- Matching engine in-process

Request flow:
Client → correct node → in-memory match → response

---

### Why it works

You minimize:
- network latency
- variability
- coordination overhead

System becomes deterministic and predictable

---

### Key techniques

- In-memory state
- Co-location of compute + data
- Partitioning (by key)

---

### Tradeoffs

1. Limited scalability
- Each node handles partition

2. Fault tolerance complexity
- Node failure risks state loss

3. Data consistency challenges
- No global easy view

---

### Failure modes

1. Uneven partitioning
- Hot shard → latency spikes

2. Node overload
- Requests dropped

3. GC / CPU spikes
- Direct latency impact

---

### When to use

- Trading engines
- Game servers
- Real-time control systems

---

### Interview line

“If computation must happen inline, I’d ensure a single-hop architecture with co-located in-memory state, avoiding fanout and external dependencies to keep latency predictable.”

---

## Key Comparison

| Aspect | Precompute + Cache | Inline Minimal Compute |
|------|------------------|----------------------|
| Latency | Lowest | Low but variable |
| Freshness | Stale | Real-time |
| Complexity | High (pipeline) | High (infra/partitioning) |
| Use case | Ads, recs | Trading, gaming |

---

## Meta Insights

- No unbounded operations
- Minimized network hops
- In-memory first
- Fail fast instead of waiting

---

## Optional Prompt Q&A

Q1: Why fallback-to-DB is dangerous?
A: DB latency is unpredictable and often 10–100ms+, immediately violating strict SLAs.

Q2: Why fanout kills SLAs?
A: Multiple downstream calls increase latency variance and tail latency (p99 grows significantly).

Q3: Why queues are incompatible?
A: Queues introduce waiting time, which is unbounded and violates hard latency guarantees.

Q4: Why partitioning works?
A: It ensures all required state is local to one node, avoiding coordination delays.

Q5: Why pipelines exist in solution 1?
A: To move heavy computation outside the request path, ensuring fast serving.
