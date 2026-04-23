# Section 3: Good Solutions — Retry & Failure Handling in Producer-Consumer Systems

---

## 🥇 Solution 1 (Most Important): At-least-once Delivery + Idempotent Consumers

### Core Idea

Accept duplicates and make processing safe.

Instead of preventing retries:

* We allow retries freely
* We design consumers so duplicates don’t break correctness

---

### Architecture

```
Producer → Queue → Consumer → Idempotent Processing Layer → DB / External Systems
```

Key addition:

* Idempotency layer

---

### What is Idempotency?

Processing the same message multiple times results in the same final state.

Examples:

* ❌ Charge card → NOT idempotent
* ✅ “Charge if not already charged” → idempotent

---

### How to Implement Idempotency

#### Approach A: Idempotency Key + Dedup Store

Each message includes a unique identifier:

```
message_id (UUID)
```

Consumer logic:

```
if message_id already processed:
    skip
else:
    process
    mark as processed
```

---

### Storage Options

* Redis (fast, supports TTL)
* Relational DB (with unique constraints)
* Kafka compacted topics

---

### Example (Payments)

Message:

```
{ payment_id: 123 }
```

Database table:

```
processed_payments(payment_id PRIMARY KEY)
```

Flow:

* Try inserting `payment_id`
* If success → process payment
* If duplicate key error → skip

---

### Why This Works

* Consumers will fail
* Messages will be retried
* Duplicates will occur

This solution embraces that reality and ensures duplicates do not corrupt state.

---

### Tradeoffs / Gotchas

#### 1. Storage Overhead

* Need to maintain processed IDs
* Requires TTL or cleanup strategy

#### 2. Domain-Specific Idempotency

* Not all operations are naturally idempotent

Examples:

* Increment → not idempotent
* Use SET / UPSERT instead

#### 3. External Side Effects

* Third-party APIs may not be idempotent
* Must pass idempotency keys downstream or handle duplication there

---

### Failure Modes at Scale

#### 1. Dedup Store Bottleneck

* High read/write load

Mitigation:

* Partitioning
* Sharding
* Redis cluster

---

#### 2. TTL Expiry Issues

* Old message retried after TTL expiry
* Dedup record missing → duplicate execution

---

#### 3. Non-Atomic Processing

Sequence:

1. Check dedup
2. Process
3. Mark done

Crash between (2) and (3) → duplicate

Mitigation:

* Use atomic DB constraints
* Transactional outbox pattern

---

### Technologies

* Queue: Kafka, SQS, RabbitMQ
* Dedup store: Redis, DynamoDB, PostgreSQL
* Techniques:

  * UPSERT
  * Unique constraints
  * Redis SETNX

---

### Assumptions

Restored:

* Duplicate processing is safe

Relaxed:

* Exactly-once execution

---

---

## 🥈 Solution 2: Visibility Timeout + ACK-Based Retry (Queue-Level)

### Core Idea

Message is hidden during processing and retried if not acknowledged.

---

### Flow

1. Consumer pulls message
2. Queue marks message invisible
3. Consumer processes
4. Consumer sends ACK
5. If no ACK → message becomes visible again

---

### Architecture

```
Queue (with visibility timeout)
   ↓
Consumer (process + ACK)
```

---

### Why This Works

* Messages are only removed on ACK
* Failure → no ACK → message retried
* Guarantees no message loss

---

### Tradeoffs / Gotchas

#### 1. Duplicate Processing Still Happens

If:

* Processing completes
* Crash occurs before ACK

Result:

* Message retried → duplicate execution

---

#### 2. Timeout Tuning Complexity

* Too short → premature retries
* Too long → delayed recovery

---

#### 3. Long-Running Tasks

If processing time exceeds timeout:

* Message becomes visible again
* Another consumer picks it up

Mitigation:

* Extend visibility timeout dynamically (heartbeat)

---

### Failure Modes at Scale

#### 1. Retry Storms

* Large number of failures
* Messages repeatedly retried
* System overload

---

#### 2. Hot Partitions

* Certain messages retry more often
* Uneven load distribution

---

#### 3. Poison Messages

* Messages that always fail
* Infinite retry loops

Mitigation:

* Dead Letter Queue (DLQ)

---

### Technologies

* AWS SQS (visibility timeout)
* Kafka (offset commit model)
* RabbitMQ (ACK/NACK)

---

### Assumptions

Restored:

* Messages are not lost

Not restored:

* No duplicate processing

---

---

## 🔥 Critical Insight

Retry mechanism does not guarantee correctness.

* Queue-level retry → ensures delivery
* Idempotent consumer → ensures correctness

Real systems combine both:

* Retry + Idempotency

---

## Final Mental Model

| Concern          | Solution                 |
| ---------------- | ------------------------ |
| Message loss     | Visibility timeout / ACK |
| Duplicate safety | Idempotent consumer      |

---

# Deep-Dive Q&A (Short Answers)

---

### 1. Why is exactly-once delivery mostly a myth?

Because:

* Network failures, crashes, and retries are unavoidable
* Systems cannot atomically guarantee both:

  * Message delivery
  * Side-effect execution

Most systems provide:

* At-least-once + idempotency

---

### 2. What is the Transactional Outbox Pattern?

Pattern to ensure atomicity between:

* DB write
* Message publish

Flow:

1. Write business data + event to DB in same transaction
2. Separate process reads outbox table
3. Publishes event to queue

Prevents:

* DB updated but message not sent
* Message sent but DB not updated

---

### 3. Kafka Exactly-Once Semantics — what does it really mean?

Kafka EOS ensures:

* No duplicate messages in Kafka logs
* Exactly-once processing within Kafka streams

BUT:

* External side effects (DB, APIs) are NOT covered

So:

* Still need idempotency outside Kafka

---

### 4. When is idempotency NOT enough?

Cases:

* Non-idempotent external systems (e.g., payment gateways without keys)
* Side effects that cannot be reversed
* Operations requiring strict ordering

In such cases:

* Need stronger coordination (transactions, locking, or redesign)

---
