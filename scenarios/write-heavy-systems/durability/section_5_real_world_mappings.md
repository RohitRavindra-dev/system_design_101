# Section 5: Real-world mappings (Write-heavy systems + high durability)

## 1. Kafka (Event ingestion / pipelines)

### What problem it solves
- Logs, analytics, user events, clickstreams
- Massive write throughput
- Must not lose events (often)

### How it maps to our solution
Producer → Partition Leader  
→ Append to log (WAL)  
→ Replicate to ISR  
→ ACK (after ISR)  
→ Consumers read from log  

### Why Kafka works here
- Log = source of truth  
- Replication ensures durability  
- Replay gives recovery  

### Where it shows up
- Uber: trip events  
- LinkedIn: activity streams  
- Netflix: telemetry pipelines  

### Interview insight
“This is similar to Kafka’s replicated partition log model”

---

## 2. Payment systems (Stripe / banking systems)

### What problem it solves
- Money movement  
- Ledger updates  
- Absolute durability required  

### How it maps
Hybrid: log + DB  

API → Durable log (Kafka / internal log)  
→ Stream processor  
→ Ledger DB (quorum-based)  

### Why not just Kafka?
- Need strong consistency  
- Need constraints (no double spend)  
- Need transactional guarantees  

### Why not just DB?
- ingestion volume is high  
- need replay/audit  

### Final pattern
Log for ingestion + DB for correctness  

### Real-world nuance
- Every transaction is:
  - logged  
  - idempotent  
  - replayable  

### Interview insight
“For payments, I’d combine a durable log with a strongly consistent ledger DB.”

---

## 3. Distributed databases (CockroachDB / Spanner / etcd)

### What problem it solves
- Strongly consistent state  
- Metadata, configs, financial data  

### How it maps
Client → Leader  
→ WAL append  
→ Replicate to quorum  
→ Commit  
→ ACK  

### Why it works
- Majority agreement = durability  
- Strong consistency guarantees  

### Where it shows up
- Kubernetes (etcd)  
- Google Spanner  
- CockroachDB  

### Interview insight
“This is similar to how Raft-based systems like etcd ensure durability via quorum commits.”

---

## 4. WhatsApp / messaging systems

### What problem it solves
- Messages must not be lost  
- High write throughput  

### How it maps
Sender → Message broker (log)  
→ Replication  
→ ACK  
→ Delivery to receivers  

### Key idea
- Message is first durably logged  
- Delivery is secondary  

### Why this matters
Even if:
- user offline  
- server crashes  

message is still in log  

### Interview insight
“Messaging systems prioritize durable enqueue before delivery.”

---

## 5. CDC / Audit systems

### What problem it solves
- Track every change  
- No loss allowed (compliance)  

### How it maps
DB change → WAL / CDC log  
→ Replicated  
→ Consumers process  

### Why log is critical
- Full history preserved  
- replay = auditability  

### Where it shows up
- Debezium  
- database WAL streaming  
- audit pipelines  

---

## 6. Observability systems (logs/metrics)

### Interesting nuance
- Sometimes durability is relaxed  
- Sometimes strict (security logs)  

### Mapping
- High throughput ingestion → Kafka  
- Storage → long-term DB  

### Insight
Same architecture, different ACK semantics  

---

## Big pattern

Producers  
→ Durable, replicated log  
→ Processing layer  
→ State store  

### Why this pattern wins

| Concern | System |
|--------|--------|
| ingestion | log |
| durability | replication |
| processing | consumers |
| state | DB |

---

## What changes across systems?

| System | What’s strict? |
|------|---------------|
| Kafka | durability |
| Payments | durability + consistency |
| Messaging | durability + delivery |
| DB | consistency |

---

## Interview-level synthesis

“Most modern systems decouple ingestion and state by introducing a durable replicated log as the backbone, and then build processing and storage on top.”

---

# Optional deep-dive Q&A

### Q1. Why do payment systems need both Kafka and DB?
Kafka handles high-throughput ingestion and guarantees durability + replay. The DB enforces strong consistency, constraints, and transactional correctness (e.g., no double-spend).

---

### Q2. When can Kafka alone be enough?
When the system is append-only, does not require strong transactional constraints, and consumers can derive state from the log (e.g., analytics pipelines, event streaming).

---

### Q3. Why do logs scale better than DB writes?
Logs use sequential writes (append-only), which are disk-friendly and avoid random I/O. Databases often require index updates and random writes, which limits throughput.
