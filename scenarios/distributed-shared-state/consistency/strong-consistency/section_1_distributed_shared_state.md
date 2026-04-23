# Section 1: Distributed Shared State — Comprehensive Notes

## 1. What is the base scenario?

**Definition (simple):**  
Multiple machines/services need to **read + write the same data**, and that data must feel like it’s a **single shared truth**.

---

### Think of it like this:

You and your friends are editing a **shared Google Doc**.

- Everyone can read/write
- Everyone expects the doc to reflect “current truth”
- But you're all sitting in different cities → **distributed system**

That doc = **shared state**

---

## 2. Where does this show up (real systems)?

This is *everywhere*. The moment you have more than one server touching the same data.

### Classic examples:

- **Seat booking system**
  - Same seat visible to multiple users
- **Bank account balance**
  - Multiple transactions hitting same account
- **Inventory system**
  - Multiple buyers reducing same stock
- **Distributed cache**
  - Many services updating same key
- **Leader election / config store**
  - Services agreeing on “who is leader”

---

## 3. Why does this problem even exist?

Because:

Scaling = multiple machines  
Multiple machines = no shared memory  
⇒ We need to simulate “shared memory” over a network

---

## 4. Core assumptions in the base scenario

### Assumption A — Eventual convergence is fine

- Not all nodes need to see the same value immediately
- Temporary inconsistency is acceptable

Example:
- Social media likes count → slightly off is fine

---

### Assumption B — Writes don’t conflict heavily

- Either:
  - Writes are rare, OR
  - Conflicts are tolerable

---

### Assumption C — Network is “mostly reliable”

- Messages may be delayed, but system will recover

---

### Assumption D — Reads can be stale

- Serving slightly outdated data is acceptable

---

### Assumption E — System prioritizes availability

- If some nodes are down, system still works (even if inconsistent)

---

## 5. What makes this hard (core tension)?

There is **no global clock** and **no shared memory**.

So:

- Two users can see different states
- Two writes can happen “at the same time”
- Messages can arrive out of order

---

## 6. The core problem boiled down

When multiple nodes update shared state:

You must answer:

1. Who wins if 2 writes conflict?
2. What does a read return?
3. How do all nodes converge?
4. What happens during network issues?

---

## 7. Mental model (important)

Whenever you hear:

“Distributed shared state”

Translate it to:

“Multiple writers + multiple readers + no shared memory + network uncertainty”

---

## 8. Concrete example

### Movie seat booking:

- Seat A1 is free
- User1 (Server1) reads → free
- User2 (Server2) reads → free
- Both try to book

Now:

- Who wins?
- How do we prevent double booking?

---

## 9. Where weak candidates fail

- Jump to tools (Redis/Kafka/etc)
- Don’t articulate assumptions
- Don’t identify conflict + ordering problem

---

## 10. Clarifications: Where is shared state maintained?

### Case A — Single database (centralized state)

- All services talk to one DB
- DB enforces transactions, locks

This works because:

- ACID guarantees
- Row-level locks
- Transactions

But introduces:

- Scalability bottleneck
- Single point of failure
- Latency

Mental model:

DB = global lock manager + truth holder

---

### Case B — Distributed state

- Multiple DB replicas
- Multiple caches
- Multiple services

Now:

- No single authority
- Conflicts emerge
- Ordering is unclear

---

## 11. Why Java read-write locks don’t apply

They only work because:

- Shared memory
- Same process
- Same scheduler

In distributed systems:

- No shared memory
- No global clock
- Network delays

Analogy:

- Java lock = same room
- Distributed system = different continents

---

## 12. Key shift in thinking

From:

Memory synchronization problem

To:

Coordination over unreliable network

---

## 13. Refined understanding

Multiple independent nodes must coordinate updates over a network while handling:

- Delays
- Failures
- Concurrency

---

## 14. Ways shared state is handled

1. Centralized (DB)
2. Partitioned (sharding)
3. Fully distributed (replicated)

---

## 15. Brutal truth

Strong consistency in distributed systems = re-implementing database/consensus system

---

## Optional Deep Dive Prompts — Answers

### 1. What are different types of conflicts?

- Write-write conflict (two users updating same value)
- Read-write conflict (stale read followed by write)
- Ordering conflict (events arrive in different orders)
- Duplicate operations (retries causing double updates)

---

### 2. How do systems resolve conflicts without strong consistency?

- Last write wins (timestamp based)
- Versioning (vector clocks)
- Application-level reconciliation
- CRDTs (conflict-free data types)

---

### 3. Shared state vs event-driven systems?

- Shared state → current value matters
- Event-driven → sequence of events matters
- Event systems avoid overwrites by storing history instead of state

---

## Final Summary

Distributed shared state means:

- Same data
- Multiple writers/readers
- Across machines
- No shared memory
- Requires coordination
