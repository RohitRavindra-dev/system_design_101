# Section 3: Good Solutions -- Latency-aware Producer--Consumer Systems

## Core Distinction

Solution 1: Queue-first (buffer-first) - Queue is a shock absorber -
Backlog is expected - Goal: Eventually process everything

Solution 2: Stream-first (latency-first) - Queue acts as conduit
(partitioning layer) - Backlog is a failure condition - Goal: Process
within strict SLA

------------------------------------------------------------------------

## Solution 1: Aggressive Consumer Scaling + Queue Depth Control

### How it works

-   Producers enqueue work
-   Consumers poll queue
-   Autoscaling based on queue depth / lag
-   Backpressure if limits reached

### Why it works

-   Preserves decoupling
-   Smooths spikes
-   Easy to scale horizontally

### Tradeoffs

Pros: - Simple - Robust to spikes Cons: - Latency depends on queue
depth - Scaling lag causes spikes - Overprovisioning cost

### Failure modes

-   Sudden spike → queue grows
-   Cold start → delayed scaling
-   Downstream bottleneck → backlog

------------------------------------------------------------------------

## Solution 2: Stream Processing (Compute-first)

### How it works

-   Consumers are always active
-   Events processed as they arrive
-   Minimal buffering
-   Lag is treated as failure

### Why it works

-   Eliminates queue delay
-   Predictable latency

### Tradeoffs

Pros: - Very low latency - Predictable SLA Cons: - Fragile under
spikes - Requires tight capacity planning - Needs load shedding

### Failure modes

-   Consumer lag → system unhealthy
-   Backpressure cascades
-   Tight coupling risks

------------------------------------------------------------------------

## Key Comparison

  Aspect           Solution 1   Solution 2
  ---------------- ------------ -------------
  Latency          Variable     Predictable
  Buffering        Strong       Weak
  Spike handling   Good         Risky
  Goal             Throughput   Latency

------------------------------------------------------------------------

## Important Insights

-   Queue depth directly affects latency
-   Solution 1 trades latency for stability
-   Solution 2 trades availability for latency guarantees
-   Buffer vs Reject is the core tradeoff

------------------------------------------------------------------------

## Examples

Correct Solution 1 use cases: - Notifications - Order processing -
Analytics pipelines - Media processing

Correct Solution 2 use cases: - Fraud detection - Auctions - Ride
matching (core logic)

------------------------------------------------------------------------

## Common Mistake

Optimizing for: - "Don't reject requests"

Ignoring: - "Does delay break correctness?"

Rule: - If delay breaks correctness → Solution 2 - If delay acceptable →
Solution 1

------------------------------------------------------------------------

## Final Mental Model

Queue-first → protect system Stream-first → protect latency
