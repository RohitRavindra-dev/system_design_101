# Section 3: Good Solutions (Delivery Semantics in Queue Systems)

## Solution 1: At-least-once + Idempotent Consumers

### Core Idea
Let the system retry freely and make consumers safe to run multiple times.

Process → then commit offset

Duplicates may happen but no incorrect side effects.

### Why it works
Shift from preventing duplicates (hard) to handling duplicates safely (easier).

### Implementation

#### Offset handling
poll message → process → commit offset

Crash before commit leads to retry and duplicate.

#### Idempotency strategies

A. Idempotent writes
UPDATE rides SET status = 'ARRIVED' WHERE ride_id = X

B. Idempotency key
if event_id already processed:
    skip
else:
    process + store event_id

Storage options:
- Redis (TTL-based)
- Database table

C. Unique constraints
INSERT INTO payments (payment_id UNIQUE)

### Tradeoffs
- Medium complexity
- Low latency
- High throughput
- Correctness depends on idempotency

### Gotchas
- Idempotency is hard with cross-service workflows
- Redis TTL expiry can reintroduce duplicates
- Ordering + idempotency conflicts

### Failure modes
- Duplicate storm due to crashes
- Idempotency store bottleneck
- Poison messages → need DLQ

### Tech stack
Kafka, SQS, RabbitMQ, Redis, relational DB, DLQ

### When to use
Default for most systems: ride updates, notifications, pipelines


---

## Solution 2: Exactly-once Effect (Transactions / Idempotency)

### Core Idea
Ensure processing + side effects + offset commit behave atomically.

### Approach A: Application-level (common)
BEGIN TRANSACTION
if payment_id not exists:
    insert
    charge
COMMIT

### Approach B: Kafka transactions (infra-level)
Read → Process → Write with atomic offset commit

### Tradeoffs
- High complexity
- Higher latency
- Lower throughput
- Strong correctness

### Gotchas
- Distributed transactions not truly atomic
- External APIs cannot be rolled back
- Lock contention

### Failure modes
- Partial success (charged but not recorded)
- DB contention
- Coordination overhead

### Tech stack
Relational DB, Kafka transactions, idempotency keys, outbox pattern

### When to use
Payments, financial systems, inventory


---

## Interview Answer

“I’ll use at-least-once delivery with idempotent consumers for most pipelines. For critical flows like payments, I’ll enforce exactly-once effect using idempotency keys and transactional guarantees.”


---

## Q&A

### Question
At-least-once system + Redis idempotency cache (TTL = 1 hour)

What bug appears?

### Answer
After TTL expires, duplicate messages arriving later will not be recognized as duplicates and will be processed again, causing unintended duplicate side effects (e.g., double charges, duplicate notifications).


---

## Key Learnings

- Exactly-once is usually implemented at application level
- Idempotency is central to correctness
- At-least-once is default in real systems
- Trade-offs exist between complexity and correctness
