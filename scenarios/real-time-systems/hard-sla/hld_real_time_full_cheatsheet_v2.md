# REAL-TIME SYSTEMS WITH HARD LATENCY SLAs — STAFF LEVEL CHEAT SHEET (V2)

---

# SECTION 1 — REAL-TIME SYSTEMS (DEEP UNDERSTANDING)

## Definition
A real-time system is one where:
> The correctness or usefulness of a response depends on it arriving within a strict time bound.

If the system responds late:
- The response is useless (ad bidding → bid ignored)
- Or harmful (trading → wrong price executed)

---

## Core Insight (CRITICAL)
Latency is not a performance metric.

Latency = correctness.

This is the biggest mental shift.

---

## System Categories

### 1. Offline Systems
- Analytics, reports
- Latency tolerance: minutes to hours
- Optimization goal: throughput

---

### 2. Near Real-Time Systems
- Notifications, feeds
- Latency tolerance: seconds
- Optimization goal: user experience

---

### 3. Real-Time Systems
- Trading, gaming, bidding
- Latency tolerance: milliseconds or less
- Optimization goal: deterministic latency

---

## Tail Latency (WHY IT MATTERS)

Latency is not uniform.

Example:
- p50 → 10ms
- p95 → 30ms
- p99 → 200ms

If SLA = 100ms:
→ p99 already violates SLA

---

## Key Insight

> Users experience tail latency, not average latency.

One slow request:
- drops bid
- causes wrong trade
- breaks game fairness

---

## Hard vs Soft Real-Time

### Soft Real-Time
- Missing deadline = degraded experience
- Example: video buffering

---

### Hard Real-Time
- Missing deadline = system failure
- Example: ad bidding, trading

---

## Mental Model

Driving analogy:

- Offline system → writing a report
- Real-time system → driving at 120 km/h

Late reaction = crash

---

# SECTION 2 — HARD LATENCY SLAs

## Definition

A strict upper bound on response time.

Example:
- Ad bidding: ~100ms
- Trading: ~1–10ms

---

## What This Really Means

You are not optimizing for:
- average latency
- throughput

You are optimizing for:
> guaranteed worst-case latency

---

## Implications

### 1. No Waiting

Anything that introduces waiting is dangerous:
- queues
- retries
- locks

---

### 2. No Slow Dependencies

Each dependency adds risk:
- DB calls
- network hops
- external services

---

### 3. Tail Latency Dominates

Even rare events matter:
- GC pauses
- cache misses
- network jitter

---

### 4. Predictability > Efficiency

You prefer:
- consistent 20ms
over:
- 5ms average + 200ms spikes

---

### 5. Fail Fast

Better to drop request than return late.

---

## Latency Budgeting

Example:

Total SLA = 100ms

Breakdown:
- network in: 10ms
- processing: 60ms
- network out: 10ms
- buffer: 20ms

Every component must fit budget.

---

# SECTION 3 — GOOD SOLUTIONS

---

## SOLUTION 1 — PRECOMPUTE + IN-MEMORY SERVING

---

## Core Idea

Move all heavy computation OUT of request path.

Convert:
“compute problem” → “lookup problem”

---

## Architecture

Offline:
Logs → stream processing → compute features → store in Redis

Online:
Request → lookup → simple computation → response

---

## Ad Bidding Example (FULL FLOW)

User opens app.

System receives:
- user_id
- context

Steps:
1. Lookup features from Redis
2. Run simple model
3. Compute bid
4. Return

Total time: ~5–10ms

---

## What is “Precompute”?

Instead of computing:
- “user clicked 5 ads in last 7 days”

At runtime ❌

You compute offline:
- click_rate_7d = 0.32

And store it.

---

## Why This Works

- Eliminates DB calls
- Eliminates joins
- Eliminates heavy computation
- Ensures predictable latency

---

## Tradeoffs

1. Staleness
2. Memory cost
3. Pipeline complexity

---

## Failure Modes

### Cache Miss
Fallback to DB → SLA violated

Solution:
→ No fallback, return default

---

### Hot Keys
Uneven load

Solution:
→ replication, sharding

---

### Pipeline Lag
Data becomes stale

---

## Interview Line

“I would move all heavy computation offline and serve requests using precomputed data from in-memory storage.”

---

---

## SOLUTION 2 — PARTITION + LOCAL IN-MEMORY COMPUTE

---

## Core Idea

Route request to node owning the data.
Process locally with no external calls.

---

## Architecture

Client → consistent hashing → node → process → respond

---

## Example: Trading

- Order book in memory
- Matching happens in same process

---

## Why It Works

- No network hops
- No coordination
- Deterministic latency

---

## Key Techniques

- Partitioning by key
- Co-locating compute + data
- In-memory state

---

## Tradeoffs

1. Hard failover
2. Uneven partitions
3. Limited global visibility

---

## Failure Modes

- Hot partitions
- Node overload
- GC pauses

---

## Interview Line

“I would colocate state and compute per partition and avoid cross-node coordination.”

---

# SECTION 4 — BAD SOLUTIONS

---

## Pattern Recognition

Bad solutions introduce:
- waiting
- dependencies
- coordination

---

## 1. Queues

Problem:
- introduces waiting
- latency becomes unbounded

---

## 2. Retries

Problem:
- adds time
- response becomes useless

---

## 3. Fanout

Problem:
- latency = slowest dependency

---

## 4. DB Fallback

Problem:
- unpredictable latency

---

## 5. Horizontal Scaling

Problem:
- improves throughput, not latency guarantees

---

## 6. Batching

Problem:
- introduces delay intentionally

---

## 7. Strong Consistency

Problem:
- coordination overhead

---

## Golden Rule

If something introduces unpredictable delay → reject.

---

# SECTION 5 — REAL WORLD SYSTEMS

---

## Ad Bidding

- SLA: ~100ms
- Uses precompute + Redis

Flow:
- request → lookup → model → bid

---

## Trading

- SLA: microseconds
- Uses in-memory matching engine

Partition:
- per stock

---

## Gaming

- SLA: ~16–50ms
- Partition per game room

State:
- in memory

---

# SECTION 6 — CONSTRAINT COMBINATIONS

---

## Latency + Consistency

Conflict:
- latency → no coordination
- consistency → coordination

Solution:
→ bounded staleness

---

## Latency + Ordering

Solution:
→ partitioned ordering

No global ordering

---

## Latency + Scale

Solution:
- no queues
- load shedding
- over-provisioning
- graceful degradation

---

# FINAL TAKEAWAYS (CRISP)

1. Optimize for p99, not average
2. Remove work from hot path
3. Avoid coordination
4. Prefer in-memory over disk
5. Partition aggressively
6. Fail fast instead of waiting
7. Accept tradeoffs consciously

---

# INTERVIEW MASTER FRAMEWORK

When solving:

1. Identify primary constraint
2. Eliminate violating patterns
3. Choose architecture
4. Explain tradeoffs clearly

---

This is the mental model of a strong candidate.
