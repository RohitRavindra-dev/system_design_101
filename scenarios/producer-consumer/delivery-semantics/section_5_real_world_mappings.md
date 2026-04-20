# Section 5: Real-world mappings

---

## 1. Uber / Ola — Ride Lifecycle

Pipeline:
Ride Service → Kafka → Consumers
(driver service, billing, notifications, analytics)

Delivery semantic:
At-least-once + idempotency

Why:
- Losing events = catastrophic
- Duplicates manageable via idempotent updates

How idempotency is achieved:
State overwrite:
SET ride_status = "ARRIVED"

Event versioning:
event_version = 5
only apply if version > current_version

Key takeaway:
State machine systems → at-least-once + idempotent transitions

---

## 2. Payments (Stripe / Razorpay / UPI)

Pipeline:
Payment Request → Queue → Payment Processor → Bank API

Delivery semantic:
Exactly-once EFFECT (not infra exactly-once)

Why:
- Double charge = unacceptable
- Missed charge = bad

How it's actually done:
Idempotency key:
payment_id = unique
if payment_id exists:
    return previous result
else:
    process

DB constraint:
UNIQUE(payment_id)

Key takeaway:
Critical money flows → exactly-once EFFECT via idempotency + DB guarantees

---

## 3. Analytics / Event Tracking

Pipeline:
Frontend → Kafka → Stream Processor → Data Warehouse

Delivery semantic:
At-least-once (sometimes at-most-once tolerated)

Why:
- Losing some events acceptable
- Duplicates negligible

Design philosophy:
Approximate correctness > system complexity

Key takeaway:
High-scale analytics → favor throughput over strict correctness

---

## 4. Notification Systems

Pipeline:
Event → Queue → Notification Workers → External Provider

Delivery semantic:
At-least-once

Why:
- Missing notification = bad UX
- Duplicate = acceptable

Enhancements:
- Dedup at client or cache

Key takeaway:
User communication → at-least-once with optional dedup

---

## 5. Driver Location Updates

Pipeline:
Driver App → Ingestion → Stream → Redis / In-memory store

Delivery semantic:
At-most-once / latest-state-wins

Why:
- Only latest location matters

Design:
SET driver_location = latest

Key takeaway:
State overwrite systems → don’t over-engineer semantics

---

## Pattern Recognition

| Use-case type | Semantic |
|--------------|---------|
| State machines | At-least-once + idempotent |
| Money | Exactly-once effect |
| Analytics | At-least-once |
| Notifications | At-least-once |
| Real-time state | At-most-once |

---

## Learnings

- Always map business criticality → delivery semantics
- Default: at-least-once + idempotency
- Exactly-once is rare and expensive
- Different pipelines require different guarantees
