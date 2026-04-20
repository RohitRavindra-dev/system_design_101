# Section 1: Producer-Consumer (Queue-based systems)

## Base Scenario

At its core:

Some components generate work → some other components process that work
asynchronously

Instead of doing work inline (request → process → response), you
decouple production from consumption using a queue/stream.

------------------------------------------------------------------------

## Canonical Architecture

Producer → Queue (Kafka / SQS / RabbitMQ) → Consumer(s)

-   Producers push messages (events/tasks)
-   Queue buffers + persists
-   Consumers pull/process asynchronously

------------------------------------------------------------------------

## When / Why does this pattern exist?

### 1. Decoupling

Producer doesn't care who processes or how. Consumers evolve
independently.

Example: Order service → emits "order_placed" Payment, inventory,
notification consume independently

------------------------------------------------------------------------

### 2. Load leveling / burst handling

Traffic is spiky, processing capacity is not.

Example: 10k requests/sec spike Consumers handle 2k/sec → queue absorbs
backlog

------------------------------------------------------------------------

### 3. Async / latency optimization

User shouldn't wait for slow operations.

Example: Sending email, generating reports, ML inference

------------------------------------------------------------------------

### 4. Fan-out (1 → many)

One event triggers multiple downstream systems.

Example: Ride booked → driver allocation, billing, analytics,
notifications

------------------------------------------------------------------------

### 5. Fault isolation

If consumer is down → queue buffers

------------------------------------------------------------------------

## Real-world mental model

Producer = Swiggy app placing order\
Queue = Kitchen order screen\
Consumers = Chefs

------------------------------------------------------------------------

## Hidden in interviews

-   Uber / Ola
-   Swiggy / Zomato
-   Notification systems
-   Log ingestion
-   Payment processing
-   Video pipelines

------------------------------------------------------------------------

## Key knobs

-   Push vs Pull
-   Partitioning strategy
-   Consumer groups
-   Backpressure
-   Retry mechanisms
-   Dead Letter Queue

------------------------------------------------------------------------

## Candidate Exercise (User Answer + Feedback)

User identified pipelines:

1.  Fare pricing
2.  Ride updates
3.  Driver location updates
4.  Analytics events
5.  Ride creation

------------------------------------------------------------------------

### Refinements

#### Fare pricing

Not queue-first. Mostly sync/cached.

#### Ride updates

Strong ordering required (partition by ride_id). Event-driven state
machine.

#### Driver location

Not strictly ordered. Latest state matters. Use Redis/in-memory store.

#### Analytics

High throughput, no strict ordering. Kafka → batch systems.

#### Ride creation

Hybrid system (sync + async). Matching via queue.

------------------------------------------------------------------------

## Delivery Semantics Bridge

Scenario: Consumer processes message but crashes before committing
offset.

Result: Message gets reprocessed → duplicate processing.

This corresponds to at-least-once delivery.

------------------------------------------------------------------------

## What breaks under duplicates?

### Idempotent case

DB update → safe

### Non-idempotent case

Notifications → duplicates\
Payments → double charge (critical)

------------------------------------------------------------------------

## Key Insight

At-least-once is safe only if consumers are idempotent or side effects
are controlled.

------------------------------------------------------------------------

## Interview framing

"What happens if consumer crashes before committing offset?"

Answer: That results in duplicate processing, corresponding to
at-least-once delivery. We must design idempotent consumers or
deduplication mechanisms.

------------------------------------------------------------------------

## Final Mental Model

  Pipeline       Queue      Ordering   Latency     Sensitivity
  -------------- ---------- ---------- ----------- -------------
  Fare           No         No         High        Low
  Ride updates   Yes        Per ride   Medium      High
  Location       Optional   No         Very High   Medium
  Analytics      Yes        No         Low         Low
  Matching       Partial    No         High        High
