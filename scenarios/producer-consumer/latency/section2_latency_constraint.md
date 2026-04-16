# Section 2: Latency Constraint in Producer-Consumer Systems

## What does "latency constraint" mean here?

In a queue system:

Latency = time from when a producer emits an event to when it is
processed.

Total latency = queue wait time + processing time

------------------------------------------------------------------------

## Types of latency constraints

### 1. Low latency (synchronous-like)

-   Target: milliseconds
-   Example:
    -   Fraud check before payment
    -   Ride matching
-   Expectation:
    -   Queue should not introduce noticeable delay

### 2. Near real-time

-   Target: sub-second to a few seconds
-   Example:
    -   Notifications
    -   Live dashboards
-   Expectation:
    -   Small buffering is acceptable
    -   Slight delay OK, but not minutes

### 3. Batch latency

-   Target: seconds → minutes → hours
-   Example:
    -   Analytics pipelines
    -   Email digests
-   Expectation:
    -   Throughput \> latency
    -   Intentional delay (batching)

------------------------------------------------------------------------

## What does this constraint do to the system?

### (A) Queue behavior

-   Low latency → Queue must be almost empty
-   Near real-time → Small backlog allowed
-   Batch → Large backlog acceptable

Key insight: Queue depth == latency

------------------------------------------------------------------------

### (B) Consumer scaling strategy

-   Low latency → over-provision consumers
-   Batch → optimize for efficiency, not speed

------------------------------------------------------------------------

### (C) Processing strategy

-   Low latency → process immediately (no batching)
-   Batch → aggregate and process in chunks

------------------------------------------------------------------------

### (D) Technology choice changes

-   Low latency → Kafka, Redis streams
-   Batch → S3 + Spark / batch jobs

------------------------------------------------------------------------

## Why does this constraint arise?

Because different products have different user expectations:

-   Payments → instant
-   Notifications → soon
-   Analytics → eventually

Also influenced by: - Cost vs speed tradeoff - Infrastructure limits -
Downstream system capacity

------------------------------------------------------------------------

## What tension does this create?

Queue wants to buffer\
Latency constraint wants no waiting

Throughput vs Latency tradeoff:

Throughput ↑ vs Latency ↓

------------------------------------------------------------------------

## Concrete intuition

Queue like a restaurant line:

-   Empty line → low latency
-   Long line → high latency
-   Batching → serving multiple orders together (efficient but slower
    per person)

------------------------------------------------------------------------

## What happens under imbalance?

If producer rate \> consumer rate:

-   Backlog grows continuously
-   Queue wait time increases linearly
-   Latency becomes unbounded over time

------------------------------------------------------------------------

## Fix strategies

1.  Scale consumers
    -   Ensure consumer throughput ≥ producer rate
2.  Introduce backpressure
    -   Rate limiting
    -   Reject requests when queue too large
3.  Prioritization
    -   Critical vs non-critical tasks
4.  Degradation strategies
    -   Drop low-priority work

------------------------------------------------------------------------

## Strong candidate framing

"In a low latency system, queue depth must be tightly controlled, since
queueing delay directly impacts latency. We should ensure consumer
throughput matches or exceeds producer rate, and introduce backpressure
when necessary."

------------------------------------------------------------------------

## Key takeaway

A growing queue is not just a backlog --- it is a latency time bomb.
