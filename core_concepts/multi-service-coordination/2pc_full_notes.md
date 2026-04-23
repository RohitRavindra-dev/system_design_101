
# Two-Phase Commit (2PC) — Comprehensive Notes

---

## 1. Concept (simple, no fluff)

Problem:
You want multiple systems to behave like one atomic transaction.

Example:
DB1 (Orders) + DB2 (Payments)

You want:
Either BOTH succeed OR BOTH fail

---

## Core Idea

Use a coordinator to force all systems to agree before committing.

---

## How it works (2 phases)

### Phase 1: Prepare (Voting phase)

Coordinator asks:
“Can you commit?”

Each participant:
- Executes transaction locally (but does not commit)
- Locks resources
- Replies YES or NO

---

### Phase 2: Commit / Abort

If ALL say YES:
Coordinator sends COMMIT

If ANY says NO:
Coordinator sends ROLLBACK

---

## Flow

Coordinator:
→ Ask all: prepare?

Participants:
→ Lock + say YES/NO

Coordinator:
→ If all YES → COMMIT
→ Else → ABORT

---

## Analogy

Group splitting a bill:

1. Everyone confirms they can pay (prepare)
2. Then everyone pays (commit)

If one cannot:
Nobody pays

---

## 2. Example (real system)

Scenario: Order + Payment

Goal:
Order created ↔ Payment charged atomically

---

### 2PC Flow

1. Order DB: “I can create order”
2. Payment DB: “I can charge”

Both say YES

Coordinator → COMMIT

---

## 3. Where / Why it helps

### Guarantees

- Strong consistency
- True atomicity across systems
- No partial failures

---

### Where used

- Traditional banking systems
- Legacy enterprise systems
- Distributed databases (internally)

---

## Key Insight

2PC gives strong consistency, but at very high cost

---

## 4. Cons / Why avoided

### 1. Blocking problem

If coordinator crashes:
- Participants stuck in prepared state
- Locks held
- System stalls

---

### 2. High latency

- Multiple network round trips
- Coordination overhead

---

### 3. Poor scalability

- Global coordination required
- Locks across systems

---

### 4. Fragility

- Network failures create uncertainty

---

### 5. Tight coupling

- All services must participate
- Hard to evolve independently

---

## 2PC vs Saga

| Aspect | 2PC | Saga |
|--------|-----|------|
| Consistency | Strong | Eventual |
| Performance | Slow | Fast |
| Scalability | Poor | High |
| Failure handling | Blocking | Compensation |
| Coupling | Tight | Loose |

---

## Core Tradeoff

2PC:
Prevent inconsistency upfront

Saga:
Allow inconsistency and fix later

---

## When to use 2PC

- Strong consistency is mandatory
- Systems tightly controlled
- Lower scale environments

Examples:
- Financial ledgers
- Core banking systems

---

## When NOT to use

- Microservices
- High-scale distributed systems
- Event-driven architectures

---

## Final Mental Model

2PC:
“Do not allow inconsistency”

Saga:
“Allow inconsistency, fix later”

---

## Interview One-liner

2PC provides strong consistency via coordinated commits but suffers from blocking, latency, and poor scalability, so modern systems prefer Saga + idempotency.

---

# Deep Dive Q&A

## Q1: Why does 2PC block?

Because participants lock resources during prepare phase and wait for coordinator decision. If coordinator crashes, they cannot proceed safely.

---

## Q2: Can 2PC be made non-blocking?

Not fully. Variants like 3PC try, but introduce more complexity and still have edge cases.

---

## Q3: Why is latency high?

Two-phase communication:
- Prepare round
- Commit round
Each involves network calls + disk writes.

---

## Q4: Why does it not scale?

Global coordination creates bottlenecks and prevents independent scaling of services.

---

## Q5: Why not use it in microservices?

Because microservices are designed for autonomy, and 2PC introduces tight coupling and shared fate.

---

## Q6: Is 2PC used inside databases?

Yes. Many distributed databases use variants internally where environment is controlled.

---

## Q7: Why Saga preferred?

Because it trades strict consistency for availability and scalability, which is better for most modern systems.

---

