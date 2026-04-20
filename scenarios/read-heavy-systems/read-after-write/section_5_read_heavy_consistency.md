# Section 5 — Real-world mappings / examples

This section focuses on mapping theoretical solutions to real-world systems and understanding why specific design choices are made.

---

## Example 1: Social media (Instagram / Twitter feeds)

### Scenario
- Extremely read-heavy  
- Users post rarely, consume constantly  

### Consistency requirement
User must see their own post immediately after posting.  
Other users can tolerate slight delay.

### Approach
Primary-read fallback (scoped consistency):
- After posting:
  - User reads → routed to primary / fresh data
- Other users:
  - Read from cache / replicas

### Key insight
Consistency is user-scoped, not global.

---

## Example 2: Amazon product catalog

### Scenario
- Read-heavy (browsing)
- Writes are infrequent (price updates, stock)

### Consistency requirement
- Seller must see updates immediately  
- Customers can tolerate slight delay

### Approach
Hybrid:
- Seller dashboard → primary (strict)
- Customer-facing → cache/CDN (eventual)

### Insight
Same data, different consistency guarantees per user type.

---

## Example 3: Payments (Stripe / banking systems)

### Scenario
- Moderate reads
- Critical correctness

### Consistency requirement
Absolute correctness (no stale reads allowed)

### Approach
Strong consistency (sync replication / quorum):
- Writes commit across replicas
- Reads can hit any node safely

### Insight
Correctness > latency

---

## Example 4: Ride-hailing (Uber/Ola booking state)

### Scenario
- Medium reads + writes  
- State machine transitions

### Consistency requirement
User must see correct ride status immediately

### Approach
Hybrid:
- Critical transitions → strong consistency / primary
- Non-critical reads → cached

### Insight
Consistency is operation-scoped

---

## Example 5: Google Docs

### Scenario
- Collaborative editing

### Consistency requirement
User must see their own edits immediately

### Approach
- Local writes applied instantly (client-side)
- Synced with backend
- Conflict resolution via OT/CRDT

### Insight
Consistency can be achieved via client-side guarantees

---

# Cross-example patterns

## Pattern 1: Consistency is rarely global
- User-scoped  
- Operation-scoped  
- Role-scoped  

## Pattern 2: Hybrid systems dominate
Critical path → strong consistency  
Non-critical → eventual consistency + caching  

## Pattern 3: Tiered read paths
Fresh reads → primary / strong path  
Normal reads → cache / replicas  

## Pattern 4: Business requirements drive design
Decisions depend on tolerance for staleness

---

# Interview framing

A strong answer example:

“Systems like social media use user-scoped consistency where the writer reads from primary while others read from replicas, whereas financial systems like payments use quorum-based strong consistency because correctness is critical.”

---

# Key learnings

- Consistency decisions are contextual, not universal  
- Hybrid architectures are the norm  
- Strong consistency is expensive and used selectively  
- Understanding user expectations is critical to system design  
