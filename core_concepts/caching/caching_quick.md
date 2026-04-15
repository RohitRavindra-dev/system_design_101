# HLD Interview Revision: The Definitive Guide to Distributed Caching
**Target Level:** Senior/Staff Software Engineer (System Design)
**Source Context:** Meta/Google Engineering Standards & Hello Interview Deep-Dive

---

## 1. The Staff Perspective: Why Cache?
In a High-Level Design (HLD) interview, caching is an **optimization**, not a requirement. Never lead with "I'll add Redis." Lead with the bottleneck.

* **The Latency Gap:** Accessing an SSD (Database) takes ~1ms. Accessing RAM (Cache) takes ~100ns. This is a **10,000x difference**.
* **The Library Analogy:** If the Database is the massive city library, the Cache is the book sitting on your nightstand. You only go to the city library when the book isn't on your nightstand.
* **The Kitchen Analogy:** A chef (App) doesn't run to the basement freezer (DB) for every pinch of salt. They keep a "Mise en place" (Cache) on the counter for high-frequency ingredients.

---

## 2. The Caching Taxonomy (Layers of the Stack)
We optimize for different constraints at different layers of the infrastructure.

| Layer | Technology | Primary Optimization | Senior Note |
| :--- | :--- | :--- | :--- |
| **Client/App** | HTTP Cache, LocalStorage | **Network Round Trips** | Hardest to invalidate (requires cache-busting URLs). Best for static assets. |
| **CDN (Edge)** | Cloudflare, Akamai | **Geographic Latency** | Solves the "Speed of Light" problem. Australia to Virginia is ~300ms; Edge is ~20ms. |
| **Load Balancer** | Nginx, Varnish | **Static Content** | Can cache "Public" API responses or HTML fragments to protect the App layer. |
| **In-Process** | Guava, Caffeine, Local Map | **Zero Network Hop** | Fastest possible. Use for "God-level" configs or small, static lookup tables. |
| **External (Dist.)** | Redis, Memcached | **Shared State** | The industry standard. Allows thousands of app nodes to share a global "source of truth." |

---

## 3. Read/Write Strategies: The "Reliability vs. Latency" Matrix
Choosing a strategy defines your system’s consistency model.

### **Read Patterns**
* **Cache-Aside (Lazy Loading):** App checks cache -> Miss -> DB -> Update Cache.
    * *Trade-off:* Lean (only requested data is cached), but first read is always slow.
* **Read-Through:** App treats Cache as the main store; Cache library handles DB fetching.
    * *Trade-off:* Simplifies app logic; acts as a proxy (standard for CDNs).

### **Write Patterns**
| Pattern | Flow | Reliability | Latency |
| :--- | :--- | :--- | :--- |
| **Write-Through** | Write to Cache & DB simultaneously | High (Synchronous) | Higher (Wait for both) |
| **Write-Around** | Write to DB only; bypass Cache | Medium | Low (Write), High (First Read) |
| **Write-Back** | Write to Cache; Async write to DB | Low (Risk of data loss) | Lowest (Ultra-fast) |

> **Staff Tip:** For analytics or high-volume logging, use **Write-Back**. For a bank ledger, use **Write-Through**. For 90% of interview cases, use **Cache-Aside**.

---

## 4. The "Three Hardest Failures" & Mitigations
Interviewers probe for these to see if you can handle "Day 2" operational issues.

### **A. Cache Penetration (Querying Non-existent Keys)**
* **Scenario:** An attacker requests `user_id: -999`. It’s not in the cache, so it slams the DB.
* **Staff Solution:** **Bloom Filters**. A probabilistic data structure that sits in front of the cache. It can say "This definitely isn't here" with 0 false negatives, saving the DB.

### **B. Cache Stampede / Breakdown (The "Taylor Swift" Hot-Key)**
* **Scenario:** A popular key (e.g., Top News Feed) expires. Suddenly, 100k requests see a "Miss" and all attempt to rebuild the cache from the DB simultaneously.
* **Staff Solution:** 1.  **Request Coalescing (Single Flight):** Use a mutex/lock so only the *first* thread queries the DB; others wait for the result.
    2.  **Cache Warming:** Proactively refresh the key (e.g., at 55s of a 60s TTL) so it never hits 0.

### **C. Cache Avalanche (Massive Simultaneous Expiry)**
* **Scenario:** You start a cluster and set all session keys to expire in exactly 2 hours. At the 2-hour mark, your DB collapses under the 100% miss rate.
* **Staff Solution:** **TTL Jitter**. Add a random offset (e.g., `TTL = 2hrs + rand(0, 300) seconds`) to stagger expirations.

---

## 5. Eviction & Expiry Policies
Memory is finite. When the cache is full, how do we decide what to kill?

* **LRU (Least Recently Used):** Most common. Evicts the "oldest" accessed data.
* **LFU (Least Frequently Used):** Best for "long-tail" data where some items are significantly more popular than others.
* **FIFO (First-In-First-Out):** Simplistic; rarely the right choice for HLD.
* **TTL vs. LRU:** Use TTL for **Freshness** (Data becomes stale). Use LRU for **Capacity** (Cache is full).

---

## 6. Consistency & Operational Scaling

### **The Invalidation Challenge**
"There are two hard things in CS: Cache Invalidation and Naming Things."
* **Invalidate on Write:** When the DB updates, delete the cache key. This ensures the next read is fresh.
* **Eventual Consistency:** Accept a 60-second "stale" window. Perfect for social feeds or view counts.

### **Consistent Hashing**
When scaling from 10 Redis nodes to 11, you don't want to reshuffle all your data. **Consistent Hashing** ensures that only `1/n` of keys need to be remapped, preventing a massive "Cold Start" cache miss event.

### **Hot-Keys & Sharding**
Even if you shard your cache by `user_id`, a single celebrity (e.g., Taylor Swift) can overwhelm a single shard.
* **Solution:** Replicate the hot-key across *all* shards or use a **Local Fallback Cache** (In-process) for 1 second.

---

## 7. Senior Signaling: How to Nail the Interview
When adding a cache, vocalize these five points to signal "Staff" level:

1.  **Quantify the Need:** "Our DB handles 5k QPS, but our peak is 50k. I’m introducing Redis to offload 90% of reads."
2.  **Define the Key Schema:** "I'll use a composite key `u:{user_id}:p:{page_id}` to keep lookups $O(1)$."
3.  **Choose Serialization Wisely:** Mention Protobuf or JSON. "I'll use Protobuf to save on network bandwidth and memory overhead."
4.  **Mention Observability:** "I will monitor the **Cache Hit Ratio**. If it drops below 80%, we need to revisit our eviction policy."
5.  **Address the "Single Point of Failure":** "I'll use Redis Sentinel or Cluster for high availability (Leader/Follower replication) so our system doesn't die if one cache node goes down."

---
### Summary Checklist
- [ ] **Bottleneck identified?** (Latency vs. Throughput)
- [ ] **Strategy chosen?** (Cache-Aside is the safe bet)
- [ ] **Failures addressed?** (Stampede, Penetration, Avalanche)
- [ ] **Invalidation plan?** (TTL vs. Manual Invalidation)
- [ ] **Scaling plan?** (Consistent Hashing)