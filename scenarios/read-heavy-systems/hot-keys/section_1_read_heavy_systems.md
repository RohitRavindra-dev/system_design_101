# Section 1 — Base Scenario: Read-heavy Systems

## What is the base scenario?

A **read-heavy system** is one where:

> **Read QPS >> Write QPS**

Think:
- Millions of users consuming content
- Few users producing content

### Examples:
- Social media feeds
- Product catalog browsing (Amazon)
- News websites
- Video streaming metadata (YouTube homepage, recommendations)
- Public APIs with high fan-out reads

---

## Why does this happen? (Core intuition)

This is almost always due to **fan-out asymmetry**:

> One write → many reads

### Example:
- A celebrity posts once → millions of users read it
- A product is added → thousands of users view it
- A tweet is created → massive read amplification

So:

Write: 1  
Reads: 10^3 to 10^7

---

## Key characteristics of read-heavy systems

### 1. Read amplification dominates cost
- Database becomes bottleneck due to repeated reads of the same data
- Network and serialization overhead repeats for each request

---

### 2. Low tolerance for latency
- Users expect:
  - Feeds → <200ms
  - Product pages → <100ms
- Reads are user-facing, so latency matters more than writes

---

### 3. High cacheability

This is the biggest lever.

> Same data is read repeatedly → perfect for caching

---

### 4. Data access patterns matter more than data size

Even small datasets can break systems if:
- Access is skewed
- Reads are bursty

---

### 5. Eventually consistent is often acceptable

- Slight staleness is acceptable (e.g., like count off by a few)
- Strong consistency is usually overkill for read-heavy paths

---

## Mental model (important)

Think of read-heavy systems as:

> “Serving systems” rather than “state mutation systems”

Your job:
- Serve data fast
- Avoid hitting the database repeatedly
- Absorb traffic spikes

---

## Typical baseline architecture (before constraints)

Client → CDN → Cache (Redis/Memcached) → Database

Flow:
1. Try cache
2. If miss → database
3. Populate cache

---

## Where things start breaking (foreshadowing constraint)

Even in this simple system:
- Cache works well until access is uneven
- Some keys get hammered → hotspots form
- This leads to the “hot key / skew” problem

---

## What you should internalize before moving on

- Read-heavy = fan-out problem
- Cache = primary optimization tool
- Bottleneck = repeated access to same data
- Goal = serve fast without hitting DB repeatedly

---

## Optional Deep Dive Prompts (with answers)

### 1. Why is caching more effective in read-heavy vs write-heavy systems?

Because the same data is requested multiple times. In read-heavy systems:
- One write can be reused across thousands or millions of reads
- Cache hit rate can be very high (80–99%)

In write-heavy systems:
- Data changes frequently
- Cached data becomes stale quickly
- Cache invalidation overhead increases

---

### 2. What happens if cache hit rate drops from 95% to 80%?

This is catastrophic at scale.

Example:
- Assume 1M requests/sec
- At 95% hit rate → 50K requests hit DB
- At 80% hit rate → 200K requests hit DB

That’s a **4x increase in DB load**, which can:
- Overload DB
- Increase latency
- Cause cascading failures

---

### 3. Why are CDNs extremely powerful in read-heavy systems?

Because they:
- Cache data geographically closer to users
- Reduce latency (edge delivery)
- Offload traffic from origin servers

For static or semi-static content:
- CDN can serve majority of requests
- Backend systems see drastically reduced load

