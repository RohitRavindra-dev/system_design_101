# Section 6: Consistency + Other Constraints (Deep Dive)

## Overview

Real-world systems are never just:
read-heavy + consistency

They are:
read-heavy + consistency + latency + scale + ordering + availability + delivery semantics

This section explores how additional constraints fundamentally change system design.

---

## 1. Consistency + Low Latency (10–50ms SLO)

### What changes
Need fast reads AND correctness.

### Impact
Strong consistency + low latency is very hard:
- primary DB adds latency
- sync replication increases latency further

### Solutions

Cache + Replicas (with tweaks):
- mandatory caching
- write-through / write-around
- selective bypass

CQRS:
- works for eventual
- breaks for strong consistency due to lag

### Techniques
- read-through cache
- local (in-process) cache
- quorum reads

### Key Insight
Strong consistency must be scoped down to critical paths.

---

## 2. Consistency + High Write Throughput

### What changes
Writes are no longer rare.

### Impact
- cache invalidation frequency increases
- replica lag increases
- event backlog increases

### Solutions

Cache + Replicas:
- works but suffers from churn and invalidation overhead

CQRS:
- better fit
- async processing absorbs spikes

### Failure Shift
From read bottleneck → write propagation bottleneck

### Key Insight
CQRS becomes more attractive as write throughput increases.

---

## 3. Consistency + Ordering

### What changes
Need correct sequence, not just correct data.

### Impact
- replication alone insufficient
- caching can break ordering
- parallelism introduces issues

### Solutions

Cache + Replicas:
- requires session stickiness and version checks

CQRS:
- natural fit with partitioned event streams

### Techniques
- partition keys
- per-key ordering
- idempotency

### Key Insight
Ordering constraints push design toward event streams.

---

## 4. Consistency + High Availability

### What changes
System must survive failures.

### Impact
CAP tradeoff:
- cannot have full consistency and availability

### Solutions

Eventual consistency:
- highly available

Strong consistency:
- may reject requests during failure

### Design Shift
- multi-region replication
- async propagation

### Key Insight
Availability often requires relaxing consistency.

---

## 5. Consistency + Geo-distribution

### What changes
Cross-region latency (100–300ms).

### Impact
Strong consistency becomes very slow.

### Solutions

Eventual consistency:
- local reads from region

Strong consistency:
- limited to critical paths

CQRS:
- strong fit
- local read models per region

### Key Insight
Geo-distribution almost forces eventual consistency.

---

## 6. Consistency + Hot Keys / Skew

### What changes
Some data is disproportionately accessed.

### Impact
- cache hotspots
- replica overload
- partition imbalance

### Solutions

Cache + Replicas:
- replication of hot keys
- request coalescing

CQRS:
- partitioning challenges

### Techniques
- hot key sharding
- fan-out caching
- CDN

### Key Insight
Hot keys break naive scaling strategies.

---

## 7. Consistency + Delivery Semantics

### What changes
Events may duplicate or reorder.

### Impact
- CQRS pipelines become complex
- duplicate processing possible

### Solutions

CQRS:
- requires idempotency
- deduplication

Cache systems:
- less affected

### Key Insight
At-least-once delivery requires idempotent processing.

---

## Big Picture Synthesis

### Base Solution Choice

General read-heavy:
- Cache + replicas

Complex reads / massive scale:
- CQRS

---

### Constraint → Design Direction

Low latency → Cache  
High write throughput → CQRS  
Ordering → CQRS  
Geo-scale → Eventual + CQRS  
Strong consistency → Primary DB (scoped)

---

### Failure Mapping

Cache + Replicas breaks:
- strict consistency
- ordering

CQRS breaks:
- strong consistency
- low latency

---

## Final Insight

System design is about understanding:
which constraint invalidates which solution.

---

## Key Interview Answer

“I’ll start with cache + replicas for read-heavy access.  
If we introduce ordering or very high write throughput, I’d move toward a CQRS-style architecture.  
If strong consistency is required, I’ll scope it narrowly and avoid applying it to the entire system.”

---

## Learnings

- Constraints reshape architecture, not just optimize it
- Strong consistency is expensive and must be scoped
- CQRS is powerful but not default
- Ordering is a hidden constraint that changes everything
- There is no universal solution, only tradeoffs
