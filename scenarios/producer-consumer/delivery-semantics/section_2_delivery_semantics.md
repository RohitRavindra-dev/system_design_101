
# Section 2: Delivery Semantics (Producer-Consumer Systems)

## What is Delivery Semantics?

Delivery semantics define what guarantees a system provides about message delivery under failures (crashes, retries, rebalances), not during normal operation.

In a system:
Producer → Queue → Consumer

Failures can occur at:
1. Producer
2. Queue
3. Consumer

Delivery semantics answer:
- Can messages be lost?
- Can messages be duplicated?
- Will messages always be delivered?

---

## Types of Delivery Semantics

### 1. At-most-once
- Delivered 0 or 1 times
- No retries
- Messages can be lost
- No duplicates

Failure example:
Consumer reads → commits offset → crashes before processing → message lost

Use cases:
- Metrics
- Non-critical logs

---

### 2. At-least-once
- Delivered one or more times
- No data loss
- Duplicates possible

Failure example:
Consumer processes → crashes before committing offset → message reprocessed

Use cases:
- Most real-world systems (default)
- Ride updates, order processing

Key requirement:
- Idempotent consumers

---

### 3. Exactly-once

- No data loss
- No duplicates

Reality:
- Hard and expensive
- Often implemented at application level

Two approaches:
1. Infrastructure-level (Kafka transactions)
2. Application-level (idempotency keys, dedup tables)

---

## Core Insight

At-least-once + Idempotency = Exactly-once EFFECT (not guarantee)

---

## Key Clarifications

- Exactly-once is NOT about partial progress or checkpointing
- It is about side effects happening only once

---

## Delivery Guarantees Spectrum

At-most-once < At-least-once < Exactly-once (application level) < Exactly-once (infra level)

Stronger guarantees → more complexity and latency

---

## Interview Framing

“I prefer at-least-once delivery with idempotent consumers, and only use stronger guarantees where business-critical invariants require it.”

---

## Q&A Practice

Q1: Promotional notifications  
A: At-least-once (missing is worse than duplicates)

Q2: Charging user  
A: Exactly-once effect via idempotency

Q3: Analytics  
A: At-least-once (duplicates acceptable)

Q4: Ride updates  
A: At-least-once with idempotent consumers

---

## Implementation Requirements for At-least-once

1. Process before committing offset
2. Messaging system ensures exclusive consumption
3. Idempotent consumers
4. Retry + Dead Letter Queue (DLQ)

---

## Key Learning

Delivery semantics define failure behavior. Strong systems choose the weakest guarantee that satisfies business correctness.
