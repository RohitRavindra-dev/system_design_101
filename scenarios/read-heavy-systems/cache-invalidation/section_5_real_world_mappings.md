# Section 5 — Real-world mappings

---

## 1. Social Media Feed (Instagram / Facebook)

### Scenario
- Massive read-heavy system  
- Feeds are fetched constantly  
- Writes = posts, likes, comments  

### Invalidation complexity
- When a user posts → millions of feeds affected  
- When someone likes → counters change everywhere  

### Likely approach
Hybrid:
- Precompute + cache-aside + async invalidation  

### Example
Post creation:
User posts →
→ Fan-out (push to followers OR mark for pull)
→ Feed cache invalidated or updated async

Feed read:
→ Check cache (precomputed feed)
→ If miss → compute → cache

### Key insight
They avoid recomputing feeds on every read

### Tradeoff
- Slightly stale feed  
- Ultra-fast reads  

---

## 2. E-commerce Product Page (Amazon)

### Scenario
- Product pages read millions of times  
- Writes = price, inventory, description updates  

### Invalidation complexity
- Price must reflect quickly  
- Inventory must not oversell  

### Likely approach
Cache-aside + selective write-through  

### Example
Update price →
→ DB update
→ Cache update (or delete)
→ Possibly event broadcast  

### Key insight
Not all fields have same consistency needs  

### Tradeoff
- Description → eventual consistency  
- Price → near real-time correctness  

---

## 3. Ride-hailing system (Uber)

### Scenario
- Ride state changes rapidly  
- Users constantly polling  

### Invalidation complexity
- Driver location changes every few seconds  
- Ride status must be accurate  

### Likely approach
Minimal caching + fast data stores + event-driven updates  

### Example
- Driver location stored in Redis/in-memory  
- Updates streamed via events  

### Key insight
Sometimes best strategy is not caching  

---

## 4. News / Content sites (Medium)

### Scenario
- Content written once  
- Read millions of times  

### Invalidation complexity
- Rare updates  

### Likely approach
Heavy caching + TTL + CDN  

### Example
Article published →
→ Cached at CDN + app cache
→ Rare invalidation on edit  

### Key insight
TTL works because writes are rare  

---

## 5. Financial / Payments systems (Stripe)

### Scenario
- Critical correctness  
- Reads frequent  
- Writes critical  

### Invalidation complexity
- Cannot show stale balance  

### Likely approach
Write-through + strong consistency  

### Example
Transaction →
→ DB update
→ Cache updated synchronously
→ Versioning / locking  

### Key insight
Correctness >> performance  

---

## Cross-system insight

Same problem, different solutions:

- Social feed → eventual consistency  
- E-commerce → hybrid  
- Ride-hailing → minimal cache  
- News → TTL  
- Payments → strong consistency  

No universal solution.

---

## Optional Deep Dive Q&A

### Q1: Feed fan-out push vs pull?
- Push: precompute feeds → faster reads, heavy writes  
- Pull: compute on read → simpler writes, slower reads  

---

### Q2: Inventory consistency tricks?
- Use write-through  
- Use DB as source of truth  
- Use reservations/locking for critical paths  

---

### Q3: CDC pipelines?
- DB changes → Kafka → consumers update cache  
- Decouples invalidation logic  

---

### Q4: Multi-layer caching?
- CDN → app cache → DB  
- Each layer reduces load progressively  

---

