# SECTION 2 — Constraint: Hard Latency SLAs

## What is this constraint?

You are given a strict upper bound on response time — and you must always stay within it.

Not “try your best”.

It’s:
- 100ms max (ads bidding)
- 10ms max (trading)
- <50ms tick updates (gaming)

If you cross it:
- Response is discarded
- System is considered failed
- You lose money / correctness

---

## Key shift from Section 1

Earlier:
Latency matters

Now:
Latency is a contractual boundary

---

## Important nuance

It’s not about average latency

You are designing for:
Worst-case latency under load (p99 / p999)

Real constraint:
“Every request must complete within X ms, even under peak load, failures, and contention.”

---

## What does this imply?

### 1. You cannot “wait”

Anything that introduces unbounded delay is dangerous:
- Queue buildup  
- Retry loops  
- Blocking I/O  
- Lock contention  

Because:
You don’t have time to wait your turn

---

### 2. You cannot rely on slow dependencies

If your request depends on:
- DB query (10–50ms)
- Network hop (5–20ms each)
- External service

You’ve already blown your budget

---

### 3. You must control tail latency

Even rare events matter:
- GC pause = 100ms spike
- Cache miss → DB fallback = 200ms

These violate SLA

---

### 4. You must prefer predictability over efficiency

Tradeoffs:

- Fast on average vs predictable → Predictable
- Efficient vs consistent latency → Consistent latency
- High throughput vs bounded latency → Bounded latency

---

### 5. You often drop work instead of delaying

Better to fail fast than respond late

Examples:
- Ad system → drop bid if timeout
- Trading → cancel order if not processed in time

---

## Where/when does this constraint appear?

### 1. Competitive environments
- Ad bidding
- Trading
- Gaming

Missing deadline = losing to competitor

---

### 2. External protocol deadlines
- Exchanges (trading APIs)
- RTB protocols

Hard timeout enforced externally

---

### 3. User-perceived fairness
- Multiplayer games
- Auctions

Delay = unfair advantage

---

## Mental model

You have a deadline timer ticking from the moment request arrives

Example:

Total budget = 100ms

- Network in: 10ms
- Processing: 60ms
- Network out: 10ms
- Buffer: 20ms

This is latency budgeting

---

## Hidden implication

You are not designing for “correctness eventually”  
You are designing for correctness within time bounds

This kills:
- Strong consistency (sometimes)
- Complex joins
- Heavy computations

---

## Common mistakes candidates make

1. “We’ll just add caching”
   → Doesn’t address tail latency or misses

2. “We’ll scale horizontally”
   → Helps throughput but not latency guarantees

3. “We’ll retry on failure”
   → Retry = too late

4. “We’ll use async queues”
   → Queue = waiting = SLA violation

---

## Interview framing

Given hard latency SLAs, we must design for deterministic, bounded latency. This means eliminating unbounded operations like queues, minimizing network hops, avoiding synchronous dependencies, and optimizing for p99 latency rather than average.

---

# Optional Deep Dive — Answers

## 1. Latency budgeting per component
Break total SLA into sub-budgets per component. Each layer (network, compute, storage) must fit within its allocated slice.

## 2. Hedged requests vs retries
Hedged requests: send duplicate request after small delay to another replica → reduces tail latency.  
Retries: wait for failure → too late in hard real-time.

## 3. Why GC tuning matters
GC pauses introduce unpredictable latency spikes (100ms+), violating p99 SLAs.

## 4. Kernel bypass / RDMA
Used in ultra-low latency systems to avoid OS/network stack overhead and reduce latency jitter.
