# 🔥 Tier 1 (MUST DO – covers 80% of Uber HLD)

If you prepare these well → you’re in a strong position.

---

## 1. 🚕 Ride Matching / Dispatch System (Uber Core)

### Base Problem
- Match riders with nearby drivers  
- Track location in real time  

### 🔄 Follow-up Constraints

#### 1. Spike Handling (e.g., IPL / concerts)

**Question:** What happens during spikes?

**What breaks:**
- Hotspot regions (geo skew)  
- Matching service overload  
- Need for surge pricing  

**Expected Evolution:**
- Geo-partitioning (region/grid based)  
- Dynamic load balancing  
- Surge pricing (feedback loop)  

---

#### 2. Real-time Requirement

**Constraint:** Location updates every 1–2 seconds  

**What breaks:**
- DB writes explode  
- Polling becomes infeasible  

**Expected:**
- WebSockets / streaming  
- Kafka / pub-sub  
- In-memory stores (Redis)  

---

#### 3. Consistency Constraint

**Constraint:** Driver should not be matched to multiple riders  

**What breaks:**
- Race conditions  
- Duplicate assignment  

**Expected:**
- Locking / reservation system  
- Idempotency  
- Atomic operations  

---

### 💡 What They’re Testing
- Real-time thinking  
- Geo sharding  
- Consistency vs latency trade-offs  

---

## 2. 🔔 Notification System

### Base Problem
- Send notifications to users  

### 🔄 Follow-ups

#### 1. High Fan-out

**Scenario:** 1 post → 10M followers  

**What breaks:**
- Synchronous push  
- DB writes  

**Expected:**
- Async queues  
- Fan-out on write vs read discussion  

---

#### 2. User Preferences

**Scenario:** Users can mute categories  

**Adds:**
- Filtering layer  
- Personalization  

---

#### 3. Delivery Guarantees

**Constraint:** No notification should be lost  

**Expected:**
- At-least-once delivery  
- Retry queues  
- Dead Letter Queue (DLQ)  

---

### 💡 What They’re Testing
- Async systems  
- Queues  
- Delivery semantics  

---

## 3. 📰 News Feed / Timeline System

### Base Problem
- Show posts from friends  

### 🔄 Follow-ups

#### 1. Celebrity Problem (Hot Keys)

**Scenario:** User with 100M followers posts  

**What breaks:**
- Fan-out on write explodes  

**Expected:**
- Hybrid model (fan-out on read for celebrities)  

---

#### 2. Ranking / Personalization

**Constraint:** Sort by relevance  

**Adds:**
- ML / ranking layer  
- Caching complexity  

---

#### 3. Real-time Feed

**Constraint:** Show updates instantly  

**Expected:**
- Push model  
- WebSockets  

---

### 💡 What They’re Testing
- Read-heavy systems  
- Caching strategies  
- Skew handling (**very important**)  

---

## 4. 📊 Rate Limiter

### Base Problem
- Limit requests per user  

### 🔄 Follow-ups

#### 1. Distributed System

**Scenario:** Multiple servers  

**What breaks:**
- Local counters become invalid  

**Expected:**
- Redis / centralized store  
- Token bucket / sliding window  

---

#### 2. Burst Handling

**Constraint:** Allow short spikes  

**Expected:**
- Token bucket algorithm  

---

#### 3. Global Limits

**Constraint:** Protect backend  

**Adds:**
- Coordination across nodes  

---

### 💡 What They’re Testing
- Distributed state  
- Correctness vs performance  

---

## 5. 💬 Chat System

### Base Problem
- Send / receive messages  

### 🔄 Follow-ups

#### 1. Ordering

**Constraint:** Messages must be in order  

**What breaks:**
- Multi-server processing  

**Expected:**
- Per-chat partitioning  
- Sequence numbers  

---

#### 2. Offline Users

**Scenario:** User returns later  

**Expected:**
- Persistence  
- Message queues  

---

#### 3. Read Receipts / Typing Indicators

**Adds:**
- Real-time signaling  
- Ephemeral data  

---

### 💡 What They’re Testing
- Ordering guarantees  
- Real-time vs storage separation  

---

# 🟡 Tier 2 (Do if time permits)

## 6. 📦 Food Delivery System (Uber Eats Variant)
- Similar to ride matching  
- Adds:
  - Order lifecycle  
  - Restaurant coordination  

---

## 7. 🔍 Search System
- Indexing vs querying  
- Eventual consistency  

---

## 8. 📈 Metrics / Logging System
- Write-heavy workload  
- Batch vs streaming  

---

# 🧠 Most Important: Constraint Patterns (This is Gold)

Across all systems, Uber reuses the same constraints:

| Constraint | What They’re Testing |
|----------|--------------------|
| Scale 10x | Horizontal scaling, partitioning |
| Real-time | Push vs pull, streaming |
| Hotspots / skew | Load imbalance, partitioning failure |
| Strict consistency | Locking, coordination |
| Low latency | Caching, in-memory systems |
| Failures | Retries, idempotency, DLQ |
| Multi-region | Replication, eventual consistency |

---

# 🔥 The Meta-Pattern (Internalize This)

Every follow-up question is essentially:

> “Your current design assumes X… what if that assumption breaks?”

### Example: IPL Spike

**Base Assumption:**
- Load is evenly distributed  

**Constraint:**
- Spike in one region  

**What breaks:**
- Partitioning  
- Load balancing  

**Fix:**
- Geo-aware sharding  
- Adaptive routing  

---

# ⚔️ Final Advice (Very Important)

Don’t prepare like:
- “Design Uber”  
- “Design Twitter”  

Instead, prepare like:
```
System → assumptions → constraint → what breaks → redesign
```