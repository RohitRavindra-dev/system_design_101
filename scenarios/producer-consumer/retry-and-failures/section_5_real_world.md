
---

# Optional Deep-Dive Q&A

---

## Q1: Transactional Outbox Pattern

**What is it?**  
Write business data + event in same DB transaction, then asynchronously publish to queue.

**Why needed?**  
Prevents inconsistency between DB and queue.

**Key benefit:**  
Avoids “DB updated but message not sent” problem.

---

## Q2: Inbox Pattern

**What is it?**  
Consumer maintains a table of processed message IDs.

**Why?**  
Ensures idempotent consumption.

**Key benefit:**  
Duplicate messages become harmless.

---

## Q3: Saga Pattern (orchestration vs choreography)

**What is it?**  
Managing multi-step distributed workflows.

- Orchestration → central controller  
- Choreography → services react to events  

**Why needed?**  
No global transactions across services.

---

## Q4: Kafka Exactly-Once Semantics (Reality)

**What it claims:**  
Exactly-once processing

**What it actually does:**  
- Idempotent producer  
- Transactional writes  
- Offset + output coupling  

**Limitation:**  
Still requires idempotent consumers/sinks.

---

## Q5: Why idempotency is sometimes not enough

Because:
- External systems may not support it  
- Some operations are inherently non-idempotent (e.g., sending SMS)  

**Solution:**  
- Introduce dedup layer  
- Or accept eventual duplicates  

---

# End of Section 5