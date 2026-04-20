# Section 4: Bad Solutions / Anti-Patterns (Delivery Semantics)

These are NOT stupid ideas.  
These are the ones candidates confidently propose and get downgraded for.

---

## Anti-pattern 1: “Let’s just use exactly-once delivery (Kafka)”

### Why it’s tempting
- Sounds perfect: no duplicates, no loss
- Kafka advertises “exactly-once semantics”
- Feels like best engineering choice

### Why it’s wrong
Exactly-once at infra level does NOT solve end-to-end correctness

Because:
- Kafka can guarantee:
  - read → process → write to Kafka
- But NOT:
  - DB writes
  - external APIs (payments, email)

### Failure example
Read message  
→ charge user (external API)  
→ crash before Kafka commit  

Kafka retries → charge again  

Exactly-once broken at system level

### What breaks first at scale
- Cross-service coordination
- External side effects
- Operational complexity

### Key takeaway
Kafka exactly-once only applies within Kafka pipelines, not across external side effects.

---

## Anti-pattern 2: “Let’s prevent duplicates entirely”

### Why it’s tempting
- Feels clean: no duplicates → no idempotency needed

### Why it’s wrong
In distributed systems, duplicates are inevitable

Reasons:
- Retries
- Network failures
- Consumer crashes
- Rebalances

### What happens if you try
- Global locks
- Central coordination
- Serialized processing

### What breaks first
- Throughput drops
- Latency increases
- Availability suffers

### Key takeaway
We don’t prevent duplicates, we design systems to tolerate them.

---

## Anti-pattern 3: “Commit offset before processing”

### Why it’s tempting
Avoid duplicates

### Why it’s wrong
This becomes at-most-once delivery

### Failure scenario
Read message  
→ commit offset  
→ crash before processing  

Message is lost forever

### What breaks
- Orders lost
- Ride events missing
- Silent data corruption

### Key takeaway
This trades duplicates for data loss.

---

## Anti-pattern 4: “Use Redis TTL dedup and call it done”

### Why it’s tempting
- Fast
- Simple

### Why it’s dangerous
TTL introduces time-bound correctness

### Failure scenario
Event processed at T=0  
TTL = 1 hour  

Retry happens at T=2 hours  
Dedup entry expired → processed again

### What breaks
- Duplicate payments
- Re-triggered workflows
- Inconsistent state

### Key takeaway
Redis dedup is optimization, not source of truth.

---

## Anti-pattern 5: “Infinite retries until success”

### Why it’s tempting
At-least-once = never give up

### Why it’s wrong
Creates poison message loops

### Scenario
Message always fails  
→ retried forever

### What breaks
- Consumer stuck
- Queue backlog grows
- Throughput collapses

### Key takeaway
Use retry limits and DLQ.

---

## Anti-pattern 6: “Idempotency is easy”

### Why it’s tempting
Just store event_id

### Why it’s incomplete

#### Multi-step workflows
Step 1 done  
Step 2 fails  
Retry → Step 1 runs again?

#### External systems
- Payment gateway not idempotent
- Email APIs not idempotent

#### Race conditions
- Same message processed concurrently

### What breaks
- Partial duplication
- Inconsistent state
- Hard debugging

### Key takeaway
Idempotency must be carefully designed.

---

## Anti-pattern 7: “One semantic for entire system”

### Why it’s tempting
Simplicity

### Why it’s wrong
Different pipelines need different guarantees

- Analytics → loose
- Payments → strict
- Notifications → medium

### What breaks
- Over-engineering
- Latency increase
- Cost increase
- Complexity increase

### Key takeaway
Choose semantics per pipeline.

---

# Summary

| Anti-pattern | Issue |
|-------------|------|
| Infra exactly-once solves everything | Does not cover side effects |
| Prevent duplicates | Kills scalability |
| Commit before processing | Causes data loss |
| Redis TTL dedup only | Time-bounded correctness |
| Infinite retries | Poison message loops |
| Idempotency is trivial | Hard in distributed systems |
| One semantic everywhere | Overkill |

---

# Core Learning

You don’t control delivery. You control correctness.
