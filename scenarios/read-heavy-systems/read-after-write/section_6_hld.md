# Section 6 — Constraint Combinations & Their Impact

We start with the base constraint:
Read-heavy systems + Read-after-write consistency

This section explores how additional constraints change the design and which solutions remain valid.

---

## 1. + Low Latency (e.g., <100ms SLA)

### What changes?
Now the system must provide:
- Strict consistency
- Low latency responses

### Conflict
- Primary reads (Solution 1) can be slower
- Synchronous replication (Solution 2) increases write latency

### What survives?

**Solution 1 (Primary-read fallback) — still best**
Optimizations:
- Use in-memory primary (Redis + write-through DB)
- Keep primary in same region
- Reduce fresh-read window duration

**Solution 2 becomes risky**
- Sync replication increases latency beyond SLA

### Insight
Low latency pushes toward localized correctness, not global correctness

---

## 2. + High Write Throughput

### What changes?
- Many concurrent writers
- Many users require fresh reads

### Problem
Primary-read fallback starts breaking:
- Too many reads hit primary
- Primary becomes bottleneck

### What survives?

**Hybrid approach**
- Shard data
- Each shard has its own primary
- Route users to shard primary

**Partial quorum usage**
- Use for critical/high contention data

### Insight
Scaling writes requires scaling primaries

---

## 3. + Geo-distribution (multi-region)

### What changes?
- Writes happen in one region
- Reads may happen in another

### Problem
- Replication lag across regions
- Sync replication causes high latency

### What survives?

**Solution 1 with geo-awareness**
- Route user to home region
- Sticky sessions / region affinity

**Solution 2 becomes expensive**
- Global sync replication is slow and fragile

### Real-world pattern
Write local, read local

### Insight
Global strong consistency is impractical at scale

---

## 4. + High Availability (99.99%)

### What changes?
- System must survive failures

### Conflict
- Sync replication reduces availability (needs quorum)

### What survives?

**Solution 1 preferred**
- Failover to replica
- Routing adapts

**Solution 2 tradeoff**
- Must choose between availability and consistency (CAP theorem)

### Insight
Strong consistency reduces availability

---

## 5. + Ordering guarantees

### What changes?
System must preserve write order

### Problem
Replica lag may violate order:
Write1 → Write2 → Read sees Write2 but not Write1

### What survives?

**Solution 1 with versioning**
- Use version numbers or vector clocks
- Ensure reads respect ordering

**Solution 2 naturally supports ordering**

### Insight
Ordering requires version-aware systems

---

## 6. + Heavy caching requirement

### What changes?
Cache must be used aggressively

### Problem
Cache introduces inconsistency

### What survives?

**Solution 1 + selective cache bypass**
- Fresh reads bypass cache
- Normal reads use cache

**Advanced patterns**
- Versioned cache
- Cache validation
- Stampede protection

### Insight
Cache becomes tiered, not universal

---

## Final Meta Model

### Step 1: Identify non-negotiables
- Read-after-write consistency

### Step 2: Relax other constraints strategically

### Step 3: Apply selective guarantees

- Writer → strong consistency
- Others → eventual consistency
- Critical paths → strong
- Non-critical → cached

---

## Strong Hire Answer Template

Given a read-heavy system with read-after-write consistency:

Use primary-read fallback to guarantee correctness for the writer while serving most traffic from replicas and cache.

With additional constraints like geo-distribution or latency:
- Scope consistency to user/session
- Route reads to local primary
- Avoid global synchronization

---

## Key Takeaway

Systems are not designed with one solution.

They are designed using:
- Controlled inconsistency
- Targeted correctness

---

## Learnings

- Constraints interact, not exist in isolation
- Global guarantees are expensive
- Scope consistency wherever possible
- Hybrid systems dominate real-world design
