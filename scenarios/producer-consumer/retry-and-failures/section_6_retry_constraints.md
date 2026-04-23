# Section 6: Constraint Combinations (Retry & Failure Handling)

## 6.1 Retry + Strict Ordering

### Constraint

Messages must be processed in order (e.g., account balance updates).

### Conflict

Retries introduce delays and reprocessing, breaking ordering.

### Failure Scenario

M1 → M2 → M3\
M1 fails → retried later\
M2, M3 processed first → order violated

### Solution

-   Partition by key (e.g., user_id)
-   Single consumer per partition
-   Block partition until retry succeeds

### Tradeoffs

-   Throughput drops (head-of-line blocking)
-   Poison messages stall entire partition

### Key Insight

You trade throughput for ordering correctness.

------------------------------------------------------------------------

## 6.2 Retry + Low Latency

### Constraint

Processing must happen in milliseconds.

### Conflict

Retries introduce delay.

### Solution

Split: - Sync path (fast) - Async path (retry-safe)

### Tradeoff

Eventual consistency.

### Key Insight

Retries must be off the critical path.

------------------------------------------------------------------------

## 6.3 Retry + High Throughput

### Constraint

Millions of messages/sec.

### Problem

Retry storms amplify load.

### Solutions

-   Exponential backoff + jitter
-   Rate limiting
-   Partitioned dedup stores

### Key Insight

Retries multiply system load.

------------------------------------------------------------------------

## 6.4 Retry + Exactly-Once

### Reality

Exactly-once is not achievable strictly.

### Practical Approach

-   At-least-once + idempotency

### Key Insight

Exactly-once = coordinated idempotency.

------------------------------------------------------------------------

## 6.5 Retry + External Side Effects

### Problem

External systems may not be idempotent.

### Solutions

-   Idempotency keys
-   Saga pattern
-   Compensation logic

### Key Insight

System correctness depends on weakest dependency.

------------------------------------------------------------------------

## 6.6 Retry + Multi-step Workflows

### Problem

Retries re-run partial workflows.

### Solution

Saga pattern: - Idempotent steps - Compensating actions

### Key Insight

Retries require stateful workflow design.

------------------------------------------------------------------------

## Impossible Constraints

You cannot guarantee all: 1. Exactly-once 2. Strict ordering 3. High
throughput

Must trade off.

------------------------------------------------------------------------

## Final Mental Model

Retries introduce duplicates.

Handling requires: - Idempotency - Controlled retries - State-aware
systems
