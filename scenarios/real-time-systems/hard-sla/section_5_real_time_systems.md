# Section 5 — Real-world mappings (Real-time systems with hard latency SLAs)

---

## 1. Ad Bidding (RTB — Real-Time Bidding)

### What is happening?
When a user opens an app or website, there is an empty ad slot. Multiple advertisers compete in real-time to show their ad. Each advertiser sends a “bid” — how much they are willing to pay to show their ad to that specific user.

### What is a bid?
A bid is simply:
“I am willing to pay ₹X to show my ad to THIS user.”

### Example:
User opens Instagram:
- user = Rohit
- location = Bangalore
- interests = tech, fitness

Advertisers respond:
- Nike → ₹5
- Amazon → ₹8
- Zomato → ₹3

Amazon wins → their ad is shown.

---

## What does our system do?
We are building the advertiser side (e.g., Amazon).

Goal:
Given a user request → decide how much to bid

Constraint:
~100ms total time

---

## Concepts explained

### Features (user features)
Features = any signal about the user

Examples:
- age
- location
- interests
- past purchases
- device type

Example:
user_123 →
{
  age: 25,
  city: Bangalore,
  interest: fitness,
  last_purchase: shoes,
  avg_spend: ₹2000
}

---

### Feature Store
A fast storage system (usually in-memory like Redis):
- key = user_id
- value = features

Used because computing features at request time is too slow.

---

### Precompute
Instead of computing data at request time, we compute it beforehand.

Bad (runtime):
Fetch data → compute → slow

Good (precompute):
Logs → process → store results → fast lookup

Example:
Instead of calculating “click rate in last 7 days” during request:
Store:
user_123 → click_rate_7d = 0.32

---

### Model
A function that takes features → outputs a score (probability user will click)

Example:
score =
  0.5 * interested_in_fitness +
  0.3 * recent_shoe_purchase +
  0.2 * high_spender

---

### Compute bid
bid = score × value_per_click

Example:
probability = 0.4
value per click = ₹20
bid = ₹8

---

## Full System Flow

### Offline (precompute phase)
User activity → pipeline → compute features → store in Redis

### Online (real-time phase)
1. Receive request
2. Lookup features (~1–2ms)
3. Run model (~1ms)
4. Compute bid
5. Return response

Total latency: ~5–10ms

---

## Why this works
- No DB calls
- No heavy computation
- No fanout
- Everything is precomputed

---

## Failure handling
- Missing feature → default bid or skip
- No retries

---

## Key insight
Ad bidding is not about computing in real-time.
It is about deciding instantly using precomputed data.

---

## Interview explanation
“In ad bidding, we receive a request and must respond within ~100ms. So we precompute user features offline and store them in an in-memory store like Redis. At request time, we fetch these features, run a lightweight model, and compute a bid.”

---

## Optional Deep Dive (Short Answers)

### 1. How is freshness handled?
Pipelines continuously update features (stream processing). Data is slightly stale but acceptable.

### 2. What happens if Redis is down?
System fails fast — skip bidding or fallback to default. No blocking.

### 3. Why not compute features on the fly?
Too slow (DB queries, joins, aggregations). Violates SLA.

### 4. How are models kept fast?
Use lightweight models (linear/logistic regression or optimized ML models). Avoid heavy deep learning in hot path.

### 5. How do we scale?
Shard feature store (Redis cluster), partition by user_id, replicate hot keys.

---

# 2. Trading Systems (Order Matching)

### Constraint
Latency = microseconds to milliseconds

### System
Client → Matching Engine (in-memory)

### Key idea
- Order book in memory
- Matching happens in-process

### Partitioning
Stock → node

Example:
AAPL → node 1
GOOG → node 2

### Flow
- order arrives
- matched instantly
- response returned

### Why it works
- no DB
- no network hops
- no coordination

### Tradeoffs
- failover complexity
- replication complexity

### Interview explanation
“Trading systems colocate compute and state in memory per partition to eliminate coordination and guarantee ultra-low latency.”

---

# 3. Multiplayer Gaming

### Constraint
~16–50ms updates

### System
Game room → assigned to one server

### Flow
- server holds game state
- processes actions
- sends updates

### Why it works
- no cross-node coordination
- consistent state per room

### Techniques
- client-side prediction
- interpolation
- lag compensation

### Tradeoffs
- slight inconsistencies
- eventual correction

### Interview explanation
“Gaming systems partition by session and process everything in-memory on one node to avoid coordination and keep latency low.”

---

# Cross-system Summary

| System | Approach |
|------|--------|
| Ad bidding | Precompute + in-memory |
| Trading | In-memory inline compute |
| Gaming | Partition + in-memory |

---

# Key Patterns

1. Move work out of request path
2. Keep hot path minimal
3. Use in-memory data
4. Partition aggressively
5. Accept tradeoffs

---

# Final Insight

Real-time systems are not just fast systems.
They eliminate sources of latency variability.
