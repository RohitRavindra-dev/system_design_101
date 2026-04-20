# Section 2: Consistency Constraints (Strong vs Eventual vs Causal)

## What is consistency?

Consistency = what a read is allowed to return after a write.

This is about client-observable behavior, not ACID or isolation levels.

---

## Why consistency matters in read-heavy systems

Reads come from:
- caches
- replicas
- precomputed views

These are not instantly in sync with writes.

So the key question becomes:
Which version of data will the user see?

---

## Types of Consistency

### 1. Strong Consistency

Definition:
After a write completes, all subsequent reads see that write immediately.

Implications:
- Reads must go to primary or perfectly synced replicas
- No stale reads allowed

Meaning:
Read path is tightly coupled with write path

Requirements:
- synchronous replication
- strict cache invalidation

Use cases:
- banking
- payments
- inventory

Key idea:
latest truth always

---

### 2. Eventual Consistency

Definition:
If no new writes happen, all replicas eventually converge.

Implications:
- Reads may return stale data
- Reads can hit caches, replicas, CDN

Meaning:
Read path decoupled from write path

Mechanisms:
- async replication
- delayed cache updates

Use cases:
- feeds
- product catalogs
- likes/counters

Key idea:
fast reads, possibly stale

---

### 3. Causal Consistency

Definition:
If one operation depends on another, all clients see them in that order.

Implications:
- ensures logical correctness
- no guarantee of global freshness

Example:
Post A -> Comment B

System must not show B without A.

Use cases:
- social feeds
- chat systems

Key idea:
things should make sense

---

## Comparison

| Property | Strong | Causal | Eventual |
|----------|--------|--------|----------|
| Freshness | Always latest | Depends | May be stale |
| Latency | High | Medium | Low |
| Scalability | Low | Medium | High |
| UX correctness | Perfect | Logical | Sometimes inconsistent |

---

## Key Insight

Consistency is a design dial.

Different parts of system can use:
- strong for critical data
- causal for UX flows
- eventual for scale

---

## Design Impact

Consistency affects:

1. Caching
   - strong: hard
   - eventual: easy

2. Replication
   - strong: sync
   - eventual: async

3. Read routing
   - strong: primary
   - eventual: replicas

4. Precomputation
   - strong: difficult
   - eventual: natural

---

## Interview Trap

Question: Can we use cache?

Answer depends:
- strong: only with strict invalidation
- eventual: yes
- causal: depends on session guarantees

---

## Mental Anchors

Strong → latest or nothing  
Eventual → fast but maybe stale  
Causal → logically consistent  

---

## Learnings

- Read-heavy systems naturally push toward eventual consistency
- Strong consistency limits scaling significantly
- Causal consistency is key for user experience correctness
