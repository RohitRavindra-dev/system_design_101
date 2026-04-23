# Section 5 — Real-world mappings

(Verbatim reconstruction with added clarifications and optional Q&A)

## Banking Systems

Problem:
- Same account
- Multiple concurrent transactions
- Cannot allow incorrect balance

Architecture:
- Sharding by account
- Each shard has leader + replicas (consensus)

Flow:
1. Request → shard leader
2. Leader checks balance and appends log
3. Replicates to followers (quorum)
4. Commit
5. Respond

Why strong consistency:
Strict ordering required for correctness.

Not used in:
- Analytics
- Reports
- Notifications

---

## Booking Systems

Problem:
- Limited seats
- Multiple users racing

Architecture:
- Per show shard
- Leader + replicas

Flow:
- Leader serializes competing requests
- One succeeds, others fail

Nuance:
- Soft locks + expiry sometimes used

---

## Messaging Systems (WhatsApp-like)

Problem:
- Ordering per chat
- Delivery guarantees

Architecture:
- Per chat partition
- Leader assigns order

Insight:
- Not globally consistent
- Only per-chat consistency

---

## Distributed Coordination Systems

Examples:
- ZooKeeper
- etcd

Use cases:
- Leader election
- Config

Requirement:
- Absolute correctness

---

## E-commerce Inventory

Approaches:

Strong consistency:
- Strict updates

Relaxed:
- Allow oversell
- Fix later

---

# Key Pattern

Strong consistency is scoped:
- Bank → per account
- Booking → per show
- Messaging → per chat

---

# Optional Deep Dive Q&A

Q1: Cross-shard transactions?
A: Require 2PC or saga, complex and slow.

Q2: Spanner?
A: Uses TrueTime + global consensus for strong consistency.

Q3: Kafka?
A: Provides ordered log, not state consistency.

---

# Clarifications

- Sharding avoids coordination across unrelated data
- Replication introduces need for coordination
- Strong consistency applied only where correctness critical
