# Section 2: Strong Consistency Constraint

## 1. What does strong consistency mean?

After a write completes, every future read must see that write.

System behaves like a single machine with one memory.

There exists a single correct order of operations and every node agrees
on that order.

## 2. Apply to scenario (Seat booking)

If a seat is booked, no other user should ever see it as available
again, even momentarily.

## 3. Implications

### Global ordering is required

All writes must be totally ordered.

### Coordination is mandatory

No independent local decisions.

### Reads cannot be stale

Cannot serve cached or lagging data.

### Latency increases

Coordination requires network round trips.

### Availability may be sacrificed

System may reject requests during partitions.

## 4. Broken assumptions

-   Eventual consistency is fine → Broken
-   Reads can be stale → Broken
-   Availability prioritized → Broken
-   Conflicts tolerable → Broken

## 5. When needed

-   Financial systems
-   Booking systems
-   Distributed locks
-   Inventory systems

## 6. Real systems

-   Spanner
-   CockroachDB
-   ZooKeeper / etcd
-   Primary DB systems

## 7. Core problem

Ensure one correct global state with strict ordering visible everywhere
instantly.

## 8. Why hard

Requires consensus, fault tolerance, handling network issues.

## 9. Key insight

Strong consistency is about agreement, not storage.

------------------------------------------------------------------------

## Optional Deep Dive Q&A

### Linearizability vs Sequential Consistency

Linearizability ensures real-time ordering. Sequential consistency only
ensures a consistent order but not real-time.

### Why leader-based systems?

Leader simplifies ordering by acting as a single writer.

### What is quorum?

Majority agreement among nodes before committing a write.

------------------------------------------------------------------------

## Clarification Recap

-   DB transactions work only in centralized setups
-   Distributed systems lack shared memory
-   Locks do not exist globally, must be implemented
-   Strong consistency requires coordination across nodes
