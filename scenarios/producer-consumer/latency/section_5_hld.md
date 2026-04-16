# Section 5: Real-world mappings (Producer–Consumer with Latency Constraints)

## Category 1: Strict Low Latency (Solution 2 dominant)

### 1. Fraud Detection (Payments)
Flow:
Payment request → Fraud service → decision → continue/block

Why latency matters:
- Must respond in <50–100ms
- Cannot queue → payment flow blocks

Design:
- Stream processing (Kafka / in-memory)
- Pre-warmed consumers
- No batching

Failure strategy:
- Fail-open (allow transaction) OR
- Degrade model (simpler checks)

---

### 2. Ride Matching (Uber/Ola)
Flow:
User request → matching service → driver assigned

Why latency matters:
- Drivers move → stale data invalid
- User expects quick response

Design:
- Real-time event streams
- In-memory state (Redis, etc.)
- Continuous processing

Hybrid usage:
- Matching → Solution 2
- Notifications/logging → Solution 1

---

### 3. Real-time Bidding (Ads)
Flow:
Ad request → bidders → winner in ~100ms

Why latency matters:
- Auction has strict timeout
- Late bids ignored

Design:
- Streaming / direct calls
- No queue buffering

---

## Category 2: Near Real-Time (Hybrid systems)

### 4. Notifications (Push / Email / SMS)
Flow:
User action → queue → workers → send notification

Why latency is flexible:
- Few seconds delay acceptable
- Delivery > immediacy

Design:
- Queue-based (Kafka/SQS)
- Autoscaling consumers

---

### 5. Live Activity Feeds (Instagram/Twitter)
Flow:
User post → fanout → followers’ feed

Why hybrid:
- Posting → needs to feel fast
- Fanout → can be slightly delayed

Design:
- Write path → fast
- Fanout → queue + workers

---

## Category 3: Batch / Throughput-first (Solution 1 dominant)

### 6. Analytics Pipelines
Flow:
Events → Kafka → batch processing → data warehouse

Why latency irrelevant:
- Minutes delay acceptable

Design:
- Heavy batching
- Large queues

---

### 7. Video Processing (YouTube)
Flow:
Upload → queue → transcoding workers

Why:
- Processing takes time anyway
- No expectation of instant completion

---

## Key Pattern Across Real Systems

Most real systems are hybrid:

- Critical path → Solution 2 (latency-first)
- Async work → Solution 1 (buffer-first)

---

## Example: Payment system (end-to-end)

User pays →
    Fraud check (Solution 2)
    ↓
    Payment success →
        Email receipt (Solution 1)
        Analytics logging (Solution 1)

---

## Pattern for Food Delivery Systems

### Solution 2 (latency-critical path)
- Order placement (user → system)
- Payment processing
- Order acceptance/rejection by restaurant
- Driver assignment / matching
- Cancellations (must reflect immediately)

Notes:
- Initial dispatch to restaurant/driver is latency-sensitive
- Retries can be handled asynchronously

---

### Solution 1 (async / buffer-friendly)
- Email receipt
- Analytics / logging
- Menu updates propagation
- Notification fanout retries
- Recommendation updates
- Billing reconciliation
- Delivery tracking history storage

---

## Important Nuance

Same feature can use both models:

Example: Driver assignment
Initial match → Solution 2 (fast)
Retry if failed → Solution 1 (queue + retry workers)

Example: Notifications
Send push → Solution 2
Retry failures → Solution 1

---

## Core Design Principle

Split systems into:
- Critical path (latency-sensitive)
- Background work (async, buffer-friendly)

---

## Final Mental Model

User is waiting → Solution 2  
User is NOT waiting → Solution 1
