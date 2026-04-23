# Section 2: Constraint — Retry & Failure Handling

## Consumers may fail → messages must be retried safely

This sounds simple. It is not. This is where most candidates collapse.

---

## 2.1 What exactly is the constraint?

In the base model we assumed:

> “Consumer processes message successfully”

Now we introduce reality:

> Consumers can fail at ANY point during processing

And because of that:

> The system must ensure the message is not lost and is retried correctly

---

## 2.2 Where exactly can failure happen?

A consumer doesn’t just “fail”. It fails at specific points:

---

### Case 1: Before processing starts
- Consumer pulls message
- Crashes before doing anything

Safe to retry (no side effects yet)

---

### Case 2: During processing (most dangerous)
- Consumer starts processing
- Does partial work (e.g., charged card)
- Crashes

Now we don’t know if work happened or not

---

### Case 3: After processing, before ACK
- Work is done
- Consumer crashes before acknowledging

Message will be retried → duplicate execution risk

---

### Case 4: After ACK (rare but possible)
- ACK sent
- Crash before downstream consistency

Message is gone → possible data inconsistency

---

## 2.3 What does “retry safely” actually mean?

Retry safely = No data loss + No harmful duplication

---

### Requirement 1: No message loss
- Every message must eventually be processed
- Even if consumer crashes

---

### Requirement 2: No incorrect duplicates
- Retrying should NOT cause:
  - Double payment
  - Double email
  - Double order

---

### Requirement 3: Eventual completion
- System should not get stuck retrying forever

---

## 2.4 Which base assumptions are now broken?

---

### Assumption A broken: Consumers are reliable

Now:
- Consumers crash frequently
- Network partitions happen
- Timeouts happen

---

### Assumption B broken: Processing is idempotent OR duplicates are fine

Now:
- Duplicate processing is dangerous
- We MUST control it

---

### Assumption C becomes explicit: Delivery semantics matter

We now care about:
- At-most-once
- At-least-once
- Exactly-once (hard)

---

### Assumption F broken: Processing is bounded

Now:
- Messages can retry indefinitely
- Poison messages appear

---

## 2.5 What new problems are introduced?

---

### Problem 1: Duplicate processing

Because of retries:
- Same message may be processed multiple times

Example:
- Payment charged twice

---

### Problem 2: Lost progress

If consumer crashes mid-task:
- Work may be partially done
- System doesn’t know state

---

### Problem 3: Poison messages

Some messages:
- Always fail
- Cause infinite retry loops

Example:
- Invalid data format
- Downstream dependency always failing

---

### Problem 4: Retry storms

If many consumers fail:
- Same messages get retried aggressively
- System overloads itself

---

### Problem 5: Ordering violations

Retries can:
- Reorder messages
- Break sequential logic

---

## 2.6 Where does this show up in real systems?

---

### Payments systems (Stripe-like)

- Charge request sent
- Consumer crashes after charging card
- Retry happens → double charge

---

### Email systems

- Email sent successfully
- ACK lost
- Retry → duplicate email

---

### Order processing (Amazon-like)

- Order placed
- Inventory updated
- Crash before marking “processed”
- Retry → inventory deducted twice

---

### Kafka-based pipelines

- Consumer reads message
- Fails before committing offset
- Message reprocessed

---

### ML pipelines

- Job partially processed
- Retry → duplicated training data

---

## 2.7 Key insight

Retries convert a reliability problem into a correctness problem

Without retries:
- You lose data (bad)

With retries:
- You risk incorrect data (worse)

The real problem is not retrying  
The real problem is retrying without corrupting state

---

## 2.8 Core tension introduced by this constraint

| Goal | Conflict |
|------|--------|
| No data loss | Requires retries |
| No duplicates | Requires control/idempotency |
| High throughput | Retries increase load |
| Low latency | Retries add delay |

You cannot optimize all simultaneously

---

## 2.9 Mental model upgrade

Earlier:
Queue = buffer

Now:
Queue = source of truth + replay mechanism

---

## 2.10 Analogy

Restaurant system:

- Chef starts cooking
- Gas goes off mid-way
- New chef restarts order

Questions:
- Did previous chef already cook partially?
- Will customer get two dishes?
- Will kitchen keep retrying forever?

This is exactly your system now

---

# Optional Deep Dive Q&A

## 1. Exactly-once vs At-least-once — what’s actually achievable?

- At-least-once: Easy → retry until success → duplicates possible  
- At-most-once: Easy → no retries → data loss possible  
- Exactly-once: Very hard → requires coordination between:
  - message system
  - processing logic
  - storage layer

In practice:
- Most systems implement **at-least-once + idempotency**
- “Exactly-once” is usually simulated, not truly guaranteed

---

## 2. Why ACK timing is everything

ACK determines when the system considers a message “done”.

- ACK before processing → data loss risk
- ACK after processing → duplicate risk

Correct approach:
- Process → commit result → then ACK

But even then:
- Crash before ACK → duplicate still possible

So ACK timing reduces risk, but does not eliminate it

---

## 3. Visibility timeout (SQS-style)

- When a consumer picks a message:
  - Message becomes invisible to others temporarily

If consumer:
- Finishes → deletes message
- Crashes → timeout expires → message becomes visible again

This enables retry without duplication during processing window

---

## 4. Offset commit (Kafka mental model)

Kafka doesn’t delete messages. It tracks progress via offsets.

- Consumer reads message
- Processes it
- Commits offset

If:
- Crash before commit → message reprocessed
- Commit before processing → data loss

So:
- Same tradeoff as ACK, just different abstraction

---

# End of Section 2