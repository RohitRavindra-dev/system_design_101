# Section 2 --- Constraint: Strict Consistency (Read-after-write)

## What exactly is the constraint?

After a write completes, any subsequent read MUST return the latest
value.

This is typically called: - Read-after-write consistency - A form of
strong consistency

### Translate this into a real guarantee

If I do:

PUT /profile → name = "Rohit v2"

Then immediately:

GET /profile

I must see "Rohit v2" ---not "Rohit v1" ---not "eventually updated"
---not "depends which replica I hit"

No staleness allowed. Period.

------------------------------------------------------------------------

## Why/when does this constraint exist?

This shows up when user trust or correctness depends on it.

### Common cases:

1.  User-owned data (most important)

-   Profile updates
-   Settings
-   Preferences

User expectation: "I just changed it. Why am I seeing old data?"

------------------------------------------------------------------------

2.  Financial / transactional systems

-   Bank balance
-   Payments
-   Wallets

Here it's not UX---it's correctness: Showing stale data = wrong money

------------------------------------------------------------------------

3.  Critical state machines

-   Order status (placed → shipped)
-   Ride booking status
-   Ticket booking

------------------------------------------------------------------------

4.  Configuration systems

-   Feature flags
-   Permissions

------------------------------------------------------------------------

## Why this constraint is painful in read-heavy systems

Because it directly conflicts with the default optimizations.

------------------------------------------------------------------------

### Conflict 1: Read replicas (async replication)

Default behavior:

Primary → (async) → Replica

Problem: - Write goes to primary - Replica is lagging - Read hits
replica → stale data

Violates read-after-write consistency

------------------------------------------------------------------------

### Conflict 2: Caching (Redis/CDN)

Default:

Client → Cache → DB

Problem: - Cache may still have old value - Even if DB is updated, cache
is stale

Violates consistency

------------------------------------------------------------------------

### Conflict 3: Geo-distribution

If user writes in region A: - Reads from region B may not have the
update yet

Again, stale read

------------------------------------------------------------------------

## Core tension

Read-heavy systems typically rely on caching and async replication for
scale, but strict read-after-write consistency breaks those
optimizations because both can serve stale data.

------------------------------------------------------------------------

## What does this constraint force you to think about?

1.  Read path must "see" the write
    -   Either by reading from source of truth
    -   Or ensuring replicas/cache are updated synchronously
2.  You can no longer blindly use:
    -   Read replicas
    -   Cache-aside patterns
    -   CDN caching
3.  You must reason about:
    -   Replication lag
    -   Cache invalidation timing
    -   Routing reads intelligently

------------------------------------------------------------------------

## Subtle but important nuance

This constraint is often scoped, not global.

Meaning: - Only the user who made the write needs strict consistency -
Others can tolerate eventual consistency

Example: - You update your profile → YOU must see latest - Others seeing
old data briefly → often fine

------------------------------------------------------------------------

## Mental model upgrade

Without constraint: Optimize for read performance

With constraint: Optimize for correctness FIRST, then recover
performance

------------------------------------------------------------------------

## Q&A / Clarifications

Q: Why exactly does replica lag happen? A: Because replication is
asynchronous. The primary commits the write first and then ships it to
replicas. Network delay, batching, and load can delay propagation.

Q: Can cache ever be safe? A: Yes, but only if: - You synchronously
update/invalidate it on writes - Or you bypass it for critical reads
Otherwise it will serve stale data.

Q: Difference between strong consistency vs read-after-write? A: -
Strong consistency: All reads see latest globally - Read-after-write:
Only guarantees the writer sees their own latest write

------------------------------------------------------------------------

## Key Learnings

-   Read-after-write consistency is about guaranteeing no stale reads
    immediately after a write
-   Default read-heavy optimizations (cache + replicas) break this
    guarantee
-   This creates a fundamental tradeoff between latency/scale vs
    correctness
-   The constraint is often user-scoped, which is a critical design
    lever
