# Section 1: Base Scenario — Producer–Consumer (Queue-based systems)

## What is the core problem?

At its simplest:

You have producers generating work and consumers processing that work, and you decouple them using a queue.

---

## Mental Model

Think in terms of decoupling + buffering + async execution:

- Producers → generate events/tasks
- Queue → absorbs bursts, smooths load
- Consumers → process at their own pace

---

## Why does this pattern exist?

Because in real systems:

1. Producers and consumers operate at different speeds  
   Example: API receives 10k requests/sec, DB can only handle 2k writes/sec

2. You don’t want producers blocked by slow processing  
   Otherwise → latency spikes + timeouts

3. You need resilience to spikes  
   Traffic is bursty, not uniform

4. You want horizontal scalability  
   Add more consumers instead of scaling a monolith

---

## Canonical Flow

Client → Producer (API/service) → Queue → Consumer workers → DB / downstream systems

---

## Where does this show up? (Real systems)

This is everywhere. If you don’t use this, you usually don’t scale.

- Order processing  
  Place order → enqueue → process payment, inventory, shipping

- Notifications  
  User action → enqueue → email/SMS/push workers

- Logging/analytics pipelines  
  Events → Kafka → stream processors

- Video/image processing  
  Upload → queue → async processing

- Ride matching (Uber-like)  
  Requests/events flowing through async pipelines

---

## What problem does it fundamentally solve?

Three big things:

### 1. Decoupling
Producer doesn’t care:
- who processes
- when it gets processed

### 2. Backpressure handling
Queue acts as a buffer:
- absorbs spikes
- prevents system collapse

### 3. Async execution
You move work out of the critical path

---

## Key properties of this base scenario

Before adding constraints, assume:

- Processing is eventually done
- Ordering may or may not matter
- Delivery semantics undefined (at-least-once, etc.)
- Latency is flexible
- Queue is reliable

---

## Where candidates usually mess this up

- Treating queue as just “a pipe” instead of a control mechanism
- Not recognizing that:
  queue = rate limiter + buffer + decoupler

---

## Interview signal

A strong candidate will say something like:

“We’ll decouple producers and consumers via a durable queue like Kafka/SQS so we can absorb burst traffic, scale consumers independently, and keep the write path fast.”
