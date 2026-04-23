# Section 1: Base Scenario — Producer-Consumer Systems

## 1.1 What is the base scenario?

At its core:

Producers generate work → Consumers process that work asynchronously

The system is decoupled via a buffer/queue/log.

### Mental model

- Producer = “I create tasks/events”
- Consumer = “I execute tasks”
- Queue = “shock absorber”

---

## 1.2 Canonical Architecture

[ Producers ] → [ Queue / Log ] → [ Consumers ]

Typical implementations:
- Queue → RabbitMQ, SQS
- Log → Kafka
- Lightweight → Redis streams / lists

---

## 1.3 When/Why does this pattern exist?

### Core reason: Decoupling + Load smoothing

---

### 1.3.1 Decoupling (Time + Ownership)

Producer doesn’t care:
- Who processes
- When it gets processed
- How many consumers exist

Example:
- User uploads image → producer emits “process image”
- Image processing service picks it later

Without this:
- User request waits for processing → bad latency

---

### 1.3.2 Load smoothing (shock absorber)

Traffic is spiky. Processing capacity is not.

Example:
- 10K requests/sec spike
- Consumers can only process 2K/sec

Queue absorbs the difference:
- No data loss
- System doesn’t crash immediately

---

### 1.3.3 Async processing = latency isolation

Critical path:
- User request → fast response

Non-critical:
- Emails
- Notifications
- Analytics
- ML pipelines

---

## 1.4 Where does this show up (real-world)?

- Notifications (push/email/SMS)
- Payment processing pipelines
- Order fulfillment systems
- Log ingestion pipelines
- Video processing systems
- Ride matching systems
- ML feature pipelines

---

## 1.5 Core assumptions in the BASE scenario

### Assumption A: Consumers are reliable

- Consumers don’t crash mid-processing
- Or failures are rare and ignorable

---

### Assumption B: Processing is idempotent OR failures don’t matter

- If a message is processed twice → no harm  
OR  
- Duplicate processing is acceptable

---

### Assumption C: Message delivery is “good enough”

- At-most-once OR at-least-once (not strictly defined yet)

---

### Assumption D: Queue is durable enough

- Messages don’t disappear randomly
- Or occasional loss is acceptable

---

### Assumption E: Ordering is not strict (often)

- Messages can be processed out of order
- Or order doesn’t matter

---

### Assumption F: Processing time is bounded

- Consumers don’t hang forever
- No infinite retries

---

## 1.6 What is NOT part of the base scenario

We are NOT yet handling:

- Consumer crashes mid-processing
- Duplicate processing issues
- Retry semantics
- Dead letter queues
- Poison messages
- Exactly-once guarantees

These are future constraints, not base assumptions.

---

## 1.7 Analogy

Restaurant kitchen:

- Waiter = Producer
- Order slip = Message
- Kitchen queue = Queue
- Chef = Consumer

Base assumptions:
- Chef doesn’t forget orders
- Chef doesn’t cook same order twice accidentally
- Orders don’t get lost
- Chef doesn’t die mid-cooking

---

## 1.8 Key takeaway

A producer-consumer system without failure handling is an idealized model.

It assumes:
“Processing eventually happens correctly”

---

# Optional Deep Dive (Short Answers)

## 1. Queue vs Log (RabbitMQ vs Kafka mental model)

- Queue: message removed once consumed (point-to-point)
- Log: message retained, consumers track offsets (replay possible)

## 2. Where does backpressure show up?

- When producer rate > consumer throughput
- Queue length grows
- Eventually memory/storage or latency becomes a bottleneck

## 3. Push vs Pull consumption

- Push: broker sends messages → lower latency but less control
- Pull: consumer fetches → better control, batching, backpressure handling

## 4. Consumer scaling interaction

- More consumers → higher throughput
- Limited by partitioning (Kafka) or queue contention
- Over-scaling can cause coordination overhead
