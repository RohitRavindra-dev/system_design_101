# Section 5: Real-World Mappings

## 1. Social Media Feed (Facebook / Instagram / Twitter)

### Scenario
- Massive read-heavy (scrolling feeds)
- Writes are relatively rare (posting, liking)

### Consistency requirement
- Not strong
- Causal + Eventual

### Why not strong?
- Global sync across millions of users = impossible latency

### What consistency is needed?

#### Causal:
- If I post → I should see it
- If I comment → post must exist

#### Eventual:
- Likes count can lag
- Feed order may slightly shift

### Actual Design

Hybrid:

1. Precomputed feeds (fan-out on write)
   - push post → followers’ feed

2. Cache layer
   - serve feed quickly

3. Event stream (backbone)
   - updates propagate async

### Tech mapping
- Event stream: Apache Kafka
- Cache: Redis
- Storage: NoSQL / distributed DB

### Interview one-liner

“Feeds use CQRS-style precomputation + eventual consistency, with causal guarantees for user actions.”

---

## 2. E-commerce Product Catalog (Amazon / Flipkart)

### Scenario
- Read-heavy (browse/search)
- Writes: product updates, inventory changes

### Consistency requirement
Mixed:

#### Eventual:
- product description
- reviews
- ratings

#### Strong (locally):
- inventory during checkout

### Design

Reads:
- cache + replicas

Writes:
- primary DB

Inventory:
- strong consistency (critical path)

### Important nuance

Same system uses multiple consistency models

### Tech mapping
- Cache: Redis
- DB: MySQL
- Search: Elasticsearch

### Interview one-liner

“Catalog is eventual, but checkout path enforces strong consistency for inventory.”

---

## 3. Chat Systems (WhatsApp / Slack)

### Scenario
- Read-heavy (messages)
- Writes frequent but localized

### Consistency requirement
- Not strong globally
- Causal consistency is critical

### Why causal matters

User expectation:
- messages must appear in order
- replies must make sense

### What happens if violated?

- Reply appears before message
- Messages reordered

### Design

1. Partitioning by conversation
   - ensures ordering

2. Event stream / log
   - ordered append

3. Read replicas / caches
   - fast delivery

### Tech mapping
- Messaging backbone: Apache Kafka
- Storage: distributed log / DB

### Interview one-liner

“Chat systems prioritize causal ordering over global consistency.”

---

## 4. Analytics Dashboards (e.g., Metrics, Admin Panels)

### Scenario
- Heavy reads (dashboards refreshing)
- Writes: data ingestion pipelines

### Consistency requirement
- Eventual consistency is fine

### Why?
- Slight delay doesn’t matter
- Humans tolerate lag

### Design

Ingestion → Event Stream → Processing → Precomputed Views → Dashboard

### Key idea
- Never compute at read time
- Always read pre-aggregated data

### Tech mapping
- Stream: Apache Kafka
- Processing: Apache Spark
- Storage: OLAP / columnar DB

### Interview one-liner

“Analytics systems are fully eventual and rely heavily on precomputed aggregates.”

---

## Patterns Across All Systems

### 1. No system uses only one consistency model

| System | Consistency Mix |
|--------|----------------|
| Feed | Eventual + Causal |
| E-commerce | Eventual + Strong |
| Chat | Causal |
| Analytics | Eventual |

### 2. Strong consistency is always scoped
- only used where absolutely necessary

### 3. Read-heavy → always one of these
- cache + replicas  
- OR precomputed views  

### 4. Causal consistency shows up in UX-driven systems
- feeds
- chats
- collaborative tools

---

## Learnings

- Real systems mix consistency models, not pick one
- Strong consistency is expensive → use sparingly
- Eventual consistency dominates read-heavy systems
- Causal consistency is critical for user-facing correctness
- Precomputation and caching are the backbone of scalable reads
