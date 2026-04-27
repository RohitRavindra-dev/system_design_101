# Redis Deep Dive — From First Principles to Real Systems

---

# 1. What Redis Actually Is

Redis is best understood as:

> An in-memory data structure server that executes operations atomically on rich data types.

This definition matters because most people incorrectly reduce Redis to “just a cache” or “just a key-value store”.

## 1.1 Not Just Key-Value

A traditional key-value store looks like:

```
"user:1" → "Rohit"
```

Redis allows structured values:

```
"user:1" → HASH
"leaderboard" → SORTED SET
"tasks" → LIST
```

So the more accurate mental model is:

> A top-level hashmap where each key maps to a specialized data structure.

---

## 1.2 Why In-Memory Matters

Redis stores data in RAM.

Implications:

- Extremely fast (microsecond-level operations)
- Limited by memory size
- Needs additional mechanisms for durability

### Analogy

- Disk database → warehouse (large, slower)
- Redis → kitchen counter (small, extremely fast)

---

## 1.3 Single-Threaded Execution

Redis processes commands on a single main thread.

This avoids:

- Locks
- Race conditions
- Complex concurrency bugs

### Analogy

Instead of multiple workers fighting over shared resources, Redis has one extremely fast worker executing tasks sequentially with no contention.

---

# 2. Keyspace and Data Modeling

## 2.1 Flat Keyspace

Redis does not support hierarchy.

```
user:1
user:1:posts
user:1:followers
```

These look hierarchical but are just strings.

Redis treats all keys as:

```
"user:1" == "banana"
```

There is no structural meaning in `:`.

---

## 2.2 Why Use `:` Then?

Because humans need structure.

It enables:

- Pattern matching (`SCAN user:*`)
- Logical grouping
- Easier debugging

---

## 2.3 Important Limitation

Redis does NOT:

- Enforce relationships
- Cascade deletes
- Support joins

If you delete:

```
DEL user:1
```

Then:

```
user:1:posts
```

still exists.

So:

> Data modeling responsibility is pushed to the application.

---

# 3. Core Data Structures

## 3.1 String

Simple value storage.

Use cases:

- Caching
- Counters

---

## 3.2 List

Ordered sequence.

Use cases:

- Queues
- Background jobs

---

## 3.3 Set

Unordered collection of unique elements.

Use cases:

- Unique users
- Membership tracking

---

## 3.4 Hash

Object-like structure.

```
HSET user:1 name Rohit age 25
```

Use cases:

- User profiles
- Metadata

---

## 3.5 Sorted Set (Critical)

Each element has a score and is automatically ordered.

```
ZADD leaderboard 100 user:1
```

Use cases:

- Leaderboards
- Ranking systems
- Time-based ordering

---

# 4. Real System: Leaderboard

## 4.1 Data Model

```
leaderboard → SORTED SET
user:1 → HASH
user:2 → HASH
```

---

## 4.2 Operations

### Add / Update Score

```
ZINCRBY leaderboard 50 user:1
```

This:

- Updates score
- Reorders automatically
- Happens atomically

---

### Get Top Players

```
ZREVRANGE leaderboard 0 2 WITHSCORES
```

---

### Get Rank

```
ZREVRANK leaderboard user:1
```

---

### Fetch Metadata

```
HGETALL user:1
```

---

## 4.3 Key Insight

Redis does not join data.

You:

1. Fetch IDs
2. Fetch metadata
3. Combine in application

---

# 5. Pub/Sub (Real-Time Messaging)

## 5.1 What It Is

Pub/Sub is a real-time messaging system inside Redis.

It is NOT:

- HTTP
- Webhooks
- Persistent storage

It works over open TCP connections.

---

## 5.2 How It Works

Subscriber:

```
SUBSCRIBE leaderboard_updates
```

Publisher:

```
PUBLISH leaderboard_updates "user:1 score=200"
```

Redis pushes messages to all subscribers.

---

## 5.3 Example Architecture

- Game service → updates score
- Redis → broadcasts event
- WebSocket server → receives and forwards to users

---

## 5.4 Critical Limitations

- No persistence
- No replay
- Messages lost if subscriber is offline

---

## 5.5 Analogy

Like a live group chat:

- If you’re online → you see messages
- If offline → you miss them forever

---

# 6. AOF (Append Only File)

## 6.1 Problem It Solves

Redis is in-memory.

Crash = data loss.

---

## 6.2 What AOF Does

Logs every write operation:

```
ZADD leaderboard 100 user:1
ZINCRBY leaderboard 50 user:1
```

---

## 6.3 Recovery

On restart:

1. Read AOF
2. Replay commands
3. Rebuild memory

---

## 6.4 fsync Strategies

### Always

Every write flushed to disk.

Pros:

- Minimal data loss

Cons:

- High latency

---

### Every Second (Common)

Flush every second.

Pros:

- Good balance

Cons:

- Up to 1 second data loss

---

### No

OS decides.

Pros:

- Fastest

Cons:

- Unpredictable loss

---

## 6.5 AOF Rewrite

Problem:

```
ZINCRBY +50
ZINCRBY +10
ZINCRBY -10
```

Becomes:

```
ZADD 150
```

This compacts history into current state.

---

# 7. Redis vs Database (With AOF Always)

## 7.1 Redis Write Path

1. Execute in memory
2. Append to AOF
3. fsync

---

## 7.2 Database Write Path

1. Parse query
2. Plan execution
3. Locate row
4. Update data
5. Write WAL
6. Update indexes
7. fsync

---

## 7.3 Key Differences

Redis:

- Simpler execution
- No query planning
- No index maintenance

Database:

- Flexible queries
- Strong consistency
- Complex execution

---

## 7.4 Critical Insight

When fsync is required:

> Disk latency dominates both systems.

---

## 7.5 Why Redis Still Often Wins

- Lower overhead
- Simpler operations
- In-memory execution

---

## 7.6 Why Databases Still Matter

- Complex queries
- Relationships
- Strong guarantees

---

# Final Mental Model

Redis =

- Flat keyspace
- Rich data structures
- Atomic operations
- Optional durability via logs

Database =

- Query engine
- Relationship management
- Strong consistency

---

# Core Takeaways

1. Redis is not just a cache; it is a data structure engine.
2. Keys are flat; hierarchy is simulated.
3. Sorted sets are the most powerful primitive for ranking systems.
4. Pub/Sub is real-time but unreliable.
5. AOF provides durability by logging commands, not state.
6. Disk fsync is the true bottleneck in durable systems.
7. Redis optimizes known access patterns; databases optimize unknown queries.

---

This document captures the foundational mental models required to reason about Redis in system design interviews and real-world systems.
