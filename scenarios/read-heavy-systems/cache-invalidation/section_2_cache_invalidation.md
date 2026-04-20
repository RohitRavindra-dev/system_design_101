# Section 2 — Cache Invalidation Complexity

## What is the constraint (in one line)?

When underlying data changes, how do we ensure cache is still correct?

---

## Why this becomes a problem (connect to Section 1)

From Section 1:

We introduced:
- DB = source of truth  
- Cache = fast copy  

So now:

DB → correct  
Cache → may be stale  

---

## What does “stale” actually mean?

Cache contains:
- Old value  
- Missing updates  
- Incorrect ordering  
- Partially updated state  

Example:

Product price updated: ₹100 → ₹120  
Cache still serves ₹100  

That is stale data.

---

## Why does this happen?

Because:

Cache does NOT automatically know when DB changes.

There is no built-in synchronization.

---

## Where does invalidation come in?

Whenever a write happens, we must:

Either:
1. Update cache  
2. Delete cache  
3. Let it expire  

---

## The core difficulty

Cache invalidation is hard because of 3 competing goals:

1. Correctness  
   Data must be fresh. No stale reads.

2. Performance  
   Cache must reduce DB load. Avoid frequent invalidations.

3. Simplicity  
   System must be maintainable. Avoid distributed coordination complexity.

You cannot fully optimize all three. You always trade one for another.

---

## Types of cache inconsistency

### 1. Stale reads
Cache not updated after write.

---

### 2. Write-write race conditions

Example:

Write1 → updates DB  
Write2 → updates DB  
Cache updated out of order  

Cache ends up with wrong value.

---

### 3. Partial invalidation

Example:
User profile updated, but only some cache keys cleared.

Leads to inconsistent views across system.

---

### 4. Thundering herd after invalidation

Cache entry deleted.  
Thousands of requests hit DB simultaneously.

Leads to DB overload.

---

### 5. Eventual inconsistency windows

Update happens.  
Cache takes time to sync.

Leads to temporary stale data window.

---

## When does this constraint matter MOST?

1. High read + frequent writes  
   Example: stock prices, ride tracking, order tracking

2. Strong correctness expectations  
   Example: payments, inventory, pricing

3. Derived or aggregated data  
   Example: feeds, counters (likes, views)

These are harder to invalidate correctly.

---

## When is it LESS critical?

When:
- Slight staleness is acceptable  
- Data changes infrequently  

Example:
- Blog posts  
- Static profiles  

---

## Key insight

Cache invalidation introduces distributed consistency problems without strong guarantees.

---

## Mental model

Cache is a lazy, eventually correct replica of your database.

Your job is to:
- Decide how quickly it converges  
- Decide how much staleness is acceptable  

---

## Key design questions (must ask in interviews)

1. Can we tolerate stale data? If yes, how much?  
2. What is the write frequency?  
3. What happens if stale data is shown?  
4. Do we need read-after-write consistency?  

---

## Summary

- Cache introduces duplicate state  
- Writes make cache stale  
- No automatic sync → invalidation required  
- Tradeoffs between correctness, performance, simplicity  
- Leads to stale reads, race conditions, thundering herd  
- Define acceptable staleness before designing  

---

# Optional Deep Dive — Questions & Answers

## 1. What is read-after-write consistency?

It means:

If a user writes data, they must immediately see their own update in subsequent reads.

Short answer:
Guarantee that the latest write is visible to the same user immediately.

---

## 2. Why is “just delete cache” not enough?

Because:
- Causes thundering herd problem  
- Many requests hit DB simultaneously  
- Leads to latency spikes and possible outages  

Short answer:
Cache miss storms overwhelm DB after invalidation.

---

## 3. Why is TTL-only caching dangerous?

Because:
- Cache may stay stale until TTL expires  
- No guarantee of freshness  
- Critical updates may not reflect immediately  

Short answer:
TTL introduces uncontrolled staleness windows.

---

## 4. Why do we need versioning or timestamps?

Because:
- Helps resolve race conditions  
- Ensures newer updates do not get overwritten by older ones  

Short answer:
Allows ordering of writes to maintain correctness.
