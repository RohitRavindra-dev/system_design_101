# 🔥 HLD Deep Dive Cheat Sheet (V2)
## Read-Heavy Systems + Strict Read-After-Write Consistency

---

# SECTION 1: BASE SCENARIO — READ HEAVY SYSTEMS

## What does “read-heavy” REALLY mean?

Reads >> Writes (10x–1000x)

### Mental Model:
- Writes = rare, high-value events
- Reads = constant, massive, performance-critical

### Analogy:
Think YouTube:
- 1 upload → millions of views
- System stress is on reads, not writes

---

## Default Architecture (IMPORTANT)

Client
 → Cache (Redis/CDN)
 → Read Replicas
 → Primary DB

### Why this exists:
- Cache handles hot reads
- Replicas scale reads horizontally
- Primary handles writes

---

## Core Optimization Philosophy

Without constraints:
→ “Serve reads as fast as possible, even if slightly stale”

---

# SECTION 2: CONSTRAINT — READ-AFTER-WRITE CONSISTENCY

## Definition

After a write completes:
→ ALL subsequent reads must return latest value

---

## Example (Critical)

User updates profile:

Write: name = "Rohit v2"
Immediate Read → MUST return "Rohit v2"

---

## Where this matters

### 1. User-owned data
- Profile
- Settings

### 2. Financial systems
- Balance
- Transactions

### 3. State machines
- Orders
- Ride status

---

## Why this breaks default design

### Problem 1: Replica Lag

Primary → (async) → Replica

Replica is behind → stale reads

---

### Problem 2: Cache Staleness

Cache still has old value

---

## Core Tension

Performance vs Correctness

This is THE interview axis.

---

# SECTION 3: GOOD SOLUTIONS

---

## 🥇 Solution 1: Primary-Read Fallback

### Core Idea

Route reads to primary ONLY when needed

---

## How it works

### Write:
Client → Primary → ACK

### Read:
IF user recently wrote:
→ Read from PRIMARY

ELSE:
→ Read from Cache / Replica

---

## How to detect “recent write”

### Option 1: Sticky Session
Mark user for ~10 sec

### Option 2: Version Tracking
Client sends last-write version

### Option 3: Token-based
Server returns write version

---

## Why this is powerful

- Guarantees correctness locally
- Preserves scalability globally

---

## Tradeoffs

- Primary load increases
- Routing complexity
- Multi-device issues

---

## Failure Modes

### 1. Primary overload
Spike in fresh reads

### 2. Multi-device inconsistency
Phone writes → laptop reads stale

### 3. Routing bugs
Leads to silent inconsistency

---

## Real-world analogy

Restaurant kitchen:
- Fresh order → go to chef directly
- Old dishes → served from buffer

---

---

## 🥈 Solution 2: Synchronous Replication

### Core Idea

Write is committed ONLY after replicas updated

---

## How it works

Write:
Primary → Replicas → ACK

Read:
Any replica safe

---

## Quorum Variant

Write succeeds if W replicas ACK

Read succeeds if R replicas read

Condition:
R + W > N

---

## Tradeoffs

- High latency
- Reduced availability
- Expensive at scale

---

## Failure Modes

- Slow replica blocks system
- Network latency spikes
- Region failures hurt

---

## Analogy

Group decision:
Everyone must agree before proceeding

---

# SECTION 4: ANTI-PATTERNS

---

## ❌ Cache Invalidation Only

### Why tempting
“Just invalidate cache on write”

### Why wrong
Race condition:

Read → miss
Write → update
Read → old value cached again

---

## ❌ Reduce Replica Lag

Even 1ms lag = incorrect

Consistency is binary

---

## ❌ TTL Cache

TTL ≠ correctness

---

## ❌ Write-through Cache

Cache failure → inconsistency

---

## ❌ Client Retries

Shifts problem, doesn’t solve

---

## ❌ Global Strong Consistency

Kills performance, overkill

---

## Meta Insight

Bad solutions = “probably fresh”

Good solutions = “guaranteed correct”

---

# SECTION 5: REAL WORLD MAPPINGS

---

## Social Media

- User sees own post → strong
- Others → eventual

---

## Amazon

- Seller → strong
- Buyer → eventual

---

## Payments

- Strong consistency everywhere

---

## Ride Systems

- Critical paths → strong
- Others → eventual

---

## Google Docs

- Client-first consistency
- Backend sync

---

## Master Insight

Consistency is scoped:
- user-level
- operation-level
- role-level

---

# SECTION 6: COMBINED CONSTRAINTS

---

## + Low Latency

→ Prefer Solution 1
→ Avoid global sync

---

## + High Write Throughput

→ Shard primaries
→ Avoid single bottleneck

---

## + Geo Distribution

→ Route to local primary
→ Avoid cross-region sync

---

## + High Availability

→ Prefer fallback
→ Sync hurts availability

---

## + Ordering

→ Use versioning

---

## + Heavy Cache

→ Bypass cache for fresh reads

---

# FINAL MENTAL MODEL

1. Identify non-negotiables
2. Relax everything else
3. Apply selective guarantees

---

# GOLDEN INTERVIEW ANSWER

“Use primary-read fallback to guarantee read-after-write consistency for the writer, while serving most reads from cache/replicas. Scope consistency instead of enforcing it globally.”

---

# YOU ARE NOW AHEAD OF 90% CANDIDATES 🚀
