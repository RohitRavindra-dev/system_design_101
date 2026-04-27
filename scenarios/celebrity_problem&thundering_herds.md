# Distributed Systems: Hot Keys, Thundering Herd, and the “Celebrity Event” Pattern

## 0. What this document is trying to do

This is not a list of buzzwords or isolated tricks.

The goal is to build a **mechanical understanding** of:

- why hot keys happen
- why thundering herds happen
- why systems like ticket booking fail under celebrity-level demand
- how the same few principles solve all of them

If you understand the _shape of the problem_, the solutions become obvious instead of memorized.

---

# 1. The Core Idea: Load Concentration

Most distributed systems assume:

> traffic is spread somewhat evenly across keys, users, or resources

Reality breaks this assumption in two ways:

### A. Hot Key Problem

A small number of keys receive a disproportionate amount of traffic.

Example:

- `product:iphone-15`
- `user:elonmusk`
- `event:taylor-swift-bangalore`

Instead of:

```
1M requests → 1M keys
```

You get:

```
1M requests → 3 keys
```

Now:

- one shard melts
- others are idle

---

### B. Thundering Herd Problem

Many clients act **at the same time**.

Example:

- cache expires at 10:00:00
- 50k clients request the same data at 10:00:01

Instead of:

```
steady traffic
```

You get:

```
sharp spike → backend collapse
```

---

### C. When Both Combine

This is where systems actually break.

Example:

- a hot key expires
- all users request it simultaneously

Now you have:

- concentrated load (hot key)
- synchronized timing (herd)

This is not additive — it’s multiplicative.

---

# 2. The “Celebrity Event” Pattern

This is the real-world version of both problems combined.

Think:

- major concert ticket drops
- sneaker releases
- IPL finals ticket sales
- product launches

### What makes it different?

It’s not just reads.

Users are trying to:

- **read the same data**
- **and modify it at the same time**
- **with strict correctness**

---

## Example: Concert Ticket Booking

At time T = 0 (tickets go live):

### Everyone hits:

- same event page
- same seat inventory
- same booking endpoint

### That creates:

#### 1. Hot Keys

```
inventory:event123
seat:A12
seat:A13
```

#### 2. Thundering Herd

- millions of users refreshing simultaneously

#### 3. Write Contention (this is the hard part)

- everyone tries to reserve seats
- only one can succeed per seat

---

## Why this is harder than normal caching problems

For something like a blog post:

- you can cache aggressively
- stale data is acceptable

For ticket booking:

- stale data = double booking
- correctness matters more than speed

So:

> you cannot rely purely on caching

You must coordinate writes.

---

# 3. First Principles for Solving These Problems

Every working system uses some combination of these:

---

## Principle 1: Spread Load (Don’t let one thing take all traffic)

### Example: Key Sharding

Instead of:

```
inventory:event123
```

Do:

```
inventory:event123:1
inventory:event123:2
inventory:event123:3
```

Requests are distributed across shards.

Analogy:

> Instead of one billing counter at a supermarket, you open 10 counters.

---

## Principle 2: Break Synchronization (Avoid everyone acting at the same time)

### Example: TTL Jitter

Instead of:

```
cache expires at exactly 60s
```

Do:

```
cache expires at 60s ± random(0–10s)
```

Now:

- requests spread over time
- no sudden spike

Analogy:

> If a stadium empties through one gate at once → chaos
> If people leave gradually → smooth flow

---

## Principle 3: Eliminate Duplicate Work

### Example: Request Coalescing

If 1000 requests ask for the same thing:

- only 1 actually computes it
- others wait for the result

Analogy:

> One person goes to the kitchen to check if food is ready, instead of 100 people crowding it.

---

## Principle 4: Control Entry Rate

### Example: Virtual Queue

Instead of letting everyone hit backend:

- users are queued
- processed at controlled rate

Analogy:

> Airport security: you don’t let 10,000 people rush the scanner.

---

## Principle 5: Isolate Contention

### Example: Partitioned Ownership

Instead of:

- one system handling all seats

Do:

- each section handled independently

```
section A → node 1
section B → node 2
```

Analogy:

> Different ticket counters for different sections of a stadium.

---

## Principle 6: Enforce Correctness Locally

### Example: Seat Locking

When user selects a seat:

- it is locked temporarily (e.g., 5 minutes)
- stored in fast storage

This ensures:

- no double booking
- controlled contention

Analogy:

> Holding a seat in a movie theater while you go buy popcorn.

---

## Principle 7: Fail Early Instead of Failing Late

### Techniques:

- rate limiting
- circuit breakers
- rejecting excess traffic

Analogy:

> A nightclub that stops letting people in instead of letting overcrowding cause a disaster inside.

---

# 4. Putting It Together: Full System Flow

Let’s walk through a realistic flow for a high-demand ticket sale.

---

## Step 1: Traffic Arrival

Millions of users open the app.

### Protection:

- CDN caches static assets
- virtual queue limits entry rate

---

## Step 2: Browsing Event Page

### Safe to cache:

- event info
- seating layout

### Not safe:

- live seat availability

---

## Step 3: Seat Selection

User clicks seat A12.

### System does:

- check availability
- acquire lock

If successful:

```
seat:A12 → locked for user123 (5 mins)
```

If not:

- fail immediately

---

## Step 4: Payment Phase

User completes payment while lock is active.

If payment succeeds:

```
seat:A12 → permanently booked
```

If timeout:

```
lock expires → seat becomes available
```

---

## Step 5: Handling Load

Throughout this:

### Hot keys handled by:

- partitioning seats
- distributing ownership

### Herd handled by:

- queues
- jitter
- rate limiting

### Correctness handled by:

- locks
- atomic operations

---

# 5. Failure Modes (What Actually Breaks Systems)

Understanding failures is more useful than memorizing solutions.

---

## Failure 1: No Queue

All users hit backend at once.

Result:

- database overload
- cascading failure

---

## Failure 2: No Jitter

All cache entries expire simultaneously.

Result:

- sudden traffic spike
- backend collapse

---

## Failure 3: No Partitioning

All requests go to one node.

Result:

- hotspot
- uneven utilization

---

## Failure 4: Weak Locking

Multiple users book same seat.

Result:

- inconsistency
- refunds and chaos

---

## Failure 5: Slow Failure Handling

System keeps retrying failing dependency.

Result:

- retry storm
- total outage

---

# 6. Mental Model to Keep

Think in three layers:

---

## Layer 1: Entry (Control the crowd)

- queues
- rate limits
- CDNs

---

## Layer 2: Distribution (Spread the load)

- sharding
- replication
- partitioning

---

## Layer 3: Coordination (Protect correctness)

- locks
- atomic operations
- transactions

---

If any one layer is weak:

- the system will fail under extreme load

---

# 7. The One-Line Summary (Not a slogan, a mechanism)

These problems are not separate.

They are all variations of:

> Too many requests, targeting the same resource, at the same time, with insufficient coordination.

Solve that, and you solve:

- hot keys
- thundering herd
- celebrity-scale events

---

# 8. If You Want to Go Deeper Next

Natural next steps:

- designing a Redis-based locking system (with edge cases)
- how consistent hashing actually distributes hot keys
- how to model this in an interview (step-by-step answer)
- how real systems degrade gracefully instead of crashing

---

End of document.
