# Section 2: Ordering Constraints (None, Per-Key, Global)

## What is Ordering?

In queue-based systems, ordering defines the sequence in which events must be processed relative to their production.

---

## Types of Ordering

### 1. No Ordering
- Events can be processed in any order.
- Example: logs, analytics, metrics.
- Maximum scalability and parallelism.

---

### 2. Per-Key Ordering
- Events must be ordered within a specific key (e.g., user_id).
- Across different keys → no ordering guarantee.
- Example: bank transactions per account, chat messages per user.

#### Key Insight:
Preserves correctness where needed while allowing parallelism.

---

### 3. Global Ordering
- All events must follow a single global sequence.
- Example: financial ledgers, blockchain systems.

#### Analogy:
A notebook where each event is written line by line. You must process exactly in that order.

#### Why Needed:
- Ensures correctness when operations depend on previous global state.

---

## Why Global Ordering is Hard

1. Requires a single logical sequence.
2. Introduces coordination overhead.
3. Limits parallelism (acts like single-threaded system).
4. Causes backpressure (one slow event blocks all).

---

## Implications of Ordering

| Type        | Parallelism | Complexity | Throughput |
|-------------|------------|-----------|-----------|
| None        | High       | Low       | High      |
| Per-Key     | Medium     | Medium    | High      |
| Global      | Low        | High      | Low       |

---

## Key Technical Requirements

- Deterministic routing (same key → same consumer)
- State awareness
- Retry handling
- Backpressure management

---

## Q&A Discussion

### Q1: Why is global ordering anti-scalable?

Answer:
Global ordering forces all events into a single logical sequence. Even with multiple consumers, events cannot be processed in parallel without strict coordination, effectively reducing the system to near-serial execution.

---

### Q2: What happens if one key becomes hot?

Answer:
A hot key creates a bottleneck because all events for that key must go to the same partition/consumer to preserve ordering. This leads to uneven load distribution and limits scalability.

---

## User Answers + Feedback

### User Answer:
- Per-key: Food delivery (userId)
- Global: Transactions

### Feedback:
- Per-key example is correct.
- Global example is directionally correct but should emphasize shared state across users (e.g., ledger).

---

### User Answer:
Pushback: "Is per-key enough?"

### Feedback:
Excellent. Strong interview move.

---

## Key Learnings

- Ordering is a tradeoff between correctness and scalability.
- Always question if strict ordering is required.
- Per-key ordering is the most practical real-world choice.
- Global ordering should be avoided unless absolutely necessary.
