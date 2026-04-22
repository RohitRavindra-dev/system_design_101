# Section 2: Constraint --- Write Latency Sensitive

## 1. What does this constraint actually mean?

Writes must be acknowledged to the client very quickly (low p99
latency).

Not: - "eventually succeed" - "queued for later"

But:

Client sends write → system must respond FAST (typically \<10--50 ms)

------------------------------------------------------------------------

## 2. Important clarification

Acknowledgement ≠ fully processed

There are 3 possible meanings of "ack":

-   Weak ack → "I received it"
-   Medium ack → "It's durably stored somewhere"
-   Strong ack → "Fully processed & committed"

Interview expectation: At least durably accepted, not just received in
memory.

------------------------------------------------------------------------

## 3. Why does this constraint exist?

(A) User-facing interactions

-   Chat apps
-   Posting tweets
-   Sending payments → Expect instant feedback

(B) High-frequency systems

-   Trading systems
-   Gaming events → Delay = lost opportunity

(C) API SLAs

-   Backend systems with strict latency contracts

------------------------------------------------------------------------

## 4. What does this constraint imply?

Core tension: Need fast acknowledgment AND durability

Implications:

1.  Cannot block on slow storage

-   Disk and cross-region replication are slow

2.  Cannot rely purely on async pipelines

-   Queue + async consumer breaks latency expectations

3.  Need a fast durability layer

-   Low latency + reliable + write-optimized

4.  Must optimize for p99 latency

-   Tail latency matters

------------------------------------------------------------------------

## 5. Which assumptions from Section 1 break?

Broken:

-   Cannot buffer writes freely in queue
-   Cannot relax write latency
-   DB cannot be purely async sink

Partially broken:

-   Eventual consistency (depends on UX expectations)
-   Batching (helps throughput but hurts latency)

------------------------------------------------------------------------

## 6. Real-world systems

-   Chat systems (instant send expectation)
-   Financial systems (fast confirmation + durability)
-   Ride booking (instant confirmation)
-   Social media posting (instant feedback)

------------------------------------------------------------------------

## 7. Core tension (interview framing)

In write-heavy systems, we decouple writes via queues.

But with write-latency-sensitive constraint: - Must acknowledge
quickly - Must include durability in critical path

Tradeoff: Latency vs Durability vs Throughput

------------------------------------------------------------------------

## 8. What interviewer is testing

-   Why Kafka alone is not enough
-   Why DB alone is too slow
-   Why in-memory ack is dangerous
-   How to balance latency vs durability

------------------------------------------------------------------------

# Optional Deep Dive Prompts (with answers)

## Q1: What happens if we ack before durability?

Answer: You risk data loss. If the process crashes after ack but before
persistence, the client believes the write succeeded but it is lost
permanently. This violates correctness guarantees.

------------------------------------------------------------------------

## Q2: Can Kafka itself act as durable storage for ack?

Answer: Yes. Kafka can act as a durable log. If configured with
replication (acks=all), a write is considered durable once replicated to
multiple brokers. This is a common pattern for fast durable
acknowledgment.

------------------------------------------------------------------------

## Q3: Why is fsync expensive?

Answer: fsync forces data from OS page cache to disk. It introduces
latency due to: - Disk I/O (especially random writes) - Blocking
behavior - Coordination with filesystem and hardware

------------------------------------------------------------------------

## Q4: What is the fastest durable write path in distributed systems?

Answer: Typically: - Append-only log (sequential write) - Replicated
in-memory + disk-backed systems (e.g., Kafka, WAL) - Avoid random disk
I/O - Use batching + replication for durability with low latency

------------------------------------------------------------------------

Generated on: 2026-04-22T06:35:01.203534
