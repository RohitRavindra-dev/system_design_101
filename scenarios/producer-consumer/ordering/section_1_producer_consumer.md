# Section 1: Base Scenario — Producer–Consumer (Queue-based systems)

## What is the core problem?

At its heart:

You have systems that produce work faster or differently than others can process it.

So you decouple:
- Producers → generate events/tasks
- Consumers → process them
- Queue → buffer in between

---

## Mental model (this is what interviewers expect you to “see”)

Think in 3 blocks:

[Producers] → [Queue / Stream] → [Consumers]

- Producers don’t care who processes
- Consumers don’t care who produced
- Queue absorbs mismatch (rate, failures, spikes)

---

## When / Why does this pattern show up?

### 1. Rate mismatch (MOST IMPORTANT)

Producer rate ≠ Consumer rate

Example:
- 10k requests/sec incoming
- DB can only handle 2k writes/sec

Without a queue → system collapses  
With a queue → backlog builds, system survives

---

### 2. Async processing (latency decoupling)

User request should not wait for heavy work.

Example:
- Upload image → resize, compress, store variants
- You ack immediately, process later

---

### 3. Fan-out / multiple consumers

One event → multiple downstream systems

Example:
- Order placed:
  - Payment service
  - Notification service
  - Analytics pipeline

Queue allows multiple consumers independently.

---

### 4. Reliability / retries

If consumer fails:
- Queue retains message
- Retry later

Without queue → lost work

---

### 5. Traffic spikes (burst handling)

Examples:
- Black Friday
- IPL traffic
- Ticket booking spikes

Queue absorbs sudden bursts

---

## Real-world intuition (important for interviews)

You can map it to:

- Kafka / Pulsar → streaming queues
- RabbitMQ / SQS → task queues
- Redis queues → lightweight buffering

---

## Concrete examples (anchor this in your head)

### Example 1: Food delivery system
- Producer: Order Service
- Queue: Order Events
- Consumers:
  - Restaurant service
  - Delivery assignment
  - Notification

---

### Example 2: Logging system
- Producers: App servers
- Queue: Kafka topic
- Consumers:
  - Storage pipeline
  - Monitoring/alerts
  - Analytics

---

### Example 3: Email sending
- Producer: Signup service
- Queue: Email jobs
- Consumer: Email workers

---

## Key properties of this base scenario

Before constraints, assume:

- No strict ordering guarantees
- At-least-once delivery (usually)
- Consumers are scalable horizontally
- Queue acts as buffer + durability layer

---

## Where candidates mess up (important)

Weak candidates:
- Jump to Kafka/SQS without explaining why queue is needed
- Ignore rate mismatch
- Don’t articulate decoupling clearly

Strong candidates:
- Start with:
  “We decouple producers and consumers via a queue to handle rate mismatch, failures, and async processing.”

---

## One-line takeaway (burn this in memory)

Producer–Consumer exists to decouple systems in time, scale, and failure domains using a buffer (queue).
