# HLD Revision: Database Sharding for High-Scale Systems
**Role Context:** Staff Engineer Perspective (Meta/Uber)
**Target Level:** Senior (3+ YOE) / SE II
**Source Analysis:** Hello Interview (Evan, fmr. Meta Staff)

---

## 1. Core Sharding Taxonomy
Before jumping into sharding, a Senior engineer demonstrates maturity by acknowledging that sharding is the "complexity tax" we pay when vertical scaling hits a physical or financial ceiling.

### Horizontal vs. Vertical Scaling
| Feature | Vertical Scaling (Scale-Up) | Horizontal Scaling (Sharding) |
| :--- | :--- | :--- |
| **Action** | Upgrade to a larger RDS instance (e.g., 70TB -> 140TB). | Partition data across multiple standalone nodes. |
| **Analogy** | Buying a bigger truck to carry more cargo. | Buying a fleet of small vans. |
| **Complexity** | Low. Standard DB operations apply. | High. Requires routing and distributed logic. |
| **Limit** | Hard ceiling (Hardware limits of cloud providers). | Theoretically infinite scaling. |

### Partitioning vs. Sharding
* **Partitioning (Logical):** Breaking a table into smaller chunks *within* one database. Helps with query performance on a single disk.
* **Sharding (Physical):** Distributing those chunks across *multiple separate machines*, each with its own CPU, memory, and connection pool.

---

## 2. Sharding Strategies & Trade-offs
The "best" strategy is always derived from the **Access Pattern**.

### A. Hash-Based Sharding
* **Logic:** `Shard_ID = Hash(Shard_Key) % Number_of_Shards`.
* **Read vs. Write:**
    * **Write:** Extremely uniform. Prevents hotspots for sequential IDs.
    * **Read:** Great for point lookups (`WHERE id=123`). Terrible for range queries (forces "Scatter-Gather" across all shards).
* **Operational Trade-off:** Re-sharding is a nightmare (moving 90% of data). 
* **Senior Signal:** Use **Consistent Hashing** to minimize data movement to $1/n$ during scaling.

### B. Range-Based Sharding
* **Logic:** Shard 1: [A-M], Shard 2: [N-Z].
* **Read vs. Write:**
    * **Write:** High risk of "Hot Shards" (e.g., if sharding by timestamp, the most recent shard takes 100% of the load).
    * **Read:** Optimized for range scans within a shard.
* **Operational Trade-off:** Requires constant monitoring and manual "splits" as certain ranges grow faster than others.

### C. Directory-Based (Lookup) Sharding
* **Logic:** A centralized "Lookup Table" stores the mapping of `Entity_ID -> Shard_ID`.
* **Read vs. Write:**
    * **Performance:** Adds an extra network hop (Latency +1) for every request.
* **Operational Trade-off:** Ultimate flexibility (move one user to their own shard). However, the directory becomes a **Single Point of Failure (SPOF)**.

---

## 3. The 'Hot Spot' (Celebrity) Problem
**The Scenario:** You shard Twitter by `user_id`. Lionel Messi lands on Shard 1. That shard collapses under the weight of his 500M followers while Shard 2 is idle.



### Mitigation Strategies:
1.  **Compound Shard Keys / Salting:** Append a random suffix to the key for celebrities.
    * *Key:* `user_id : salt_index(0-N)`
    * *Effect:* Messi's writes are spread across $N$ shards.
    * *Cost:* Reads must now query $N$ shards and aggregate the results.
2.  **Dedicated Celebrity Shards:** Use the Directory Tier to identify "Whales" and move them to dedicated high-performance hardware (the "VIP Lounge" for data).

---

## 4. Distributed Complexity: The 'Pain Points'

### Cross-Shard Joins
* **Standard Answer:** Joins don't work across shards.
* **Senior Solution:** Denormalize data (redundancy) to keep related data together, or perform the join at the **Application Layer**. If you are doing constant "Scatter-Gather," your shard key is wrong.

### Distributed Transactions
* **2-Phase Commit (2PC):** Ensures ACID but is a performance killer due to synchronous locking.
* **Saga Pattern:** A sequence of local transactions. If one fails, you trigger "Compensating Transactions" (undos). This favors **Eventual Consistency** over strict locking.

### Global Secondary Indexes (GSI)
* Sharding by `user_id` but needing to query by `email` requires a GSI. This is essentially a second table sharded by `email` that points back to the `user_id`.

---

## 5. Operational Realities: Re-sharding
How do we scale from 10 to 20 shards without a total outage?



### The "Live Migration" Pattern:
1.  **Shadow Migration:** Start a background process to copy data from old shards to new ones.
2.  **Dual-Write:** Modify the app to write to both the old and new locations.
3.  **The Pivot:** Once synchronized, update the routing tier to point to the new shards.
4.  **Cleanup:** Drop the data from the old shards.

---

## 6. Routing Tier Design
Where should the sharding logic live?

1.  **Client-side (Library):** Fast (no extra hop), but hard to update/patch without a full app deploy.
2.  **Proxy/Gateway (e.g., Vitess):** Sits between the App and DB. Centralizes logic and connection pooling. Decouples app logic from DB topology.
3.  **Sidecar (Service Mesh):** A local proxy on the same pod/host. Combines the benefits of both (localized speed + centralized control).

---

## 💡 Interviewer Pro-Tips (Senior/Staff Signaling)

* **Pro-Tip 1: The "No-Shard" Defense.** Don't propose sharding immediately. Start with: *"Based on the math (500M users * 5KB = 2.5TB), a single high-memory instance can handle this. I wouldn't shard yet to avoid complexity."* This shows you care about operational costs.
* **Pro-Tip 2: High Cardinality.** Always pick a key with millions of unique values. Sharding by `country` or `is_premium` is a red flag.
* **Pro-Tip 3: Query Alignment.** Explicitly state: *"I'm sharding by User_ID because 95% of our queries are user-scoped (loading a profile or feed)."*
* **Pro-Tip 4: The Blast Radius.** Acknowledge that while sharding helps scale, it increases the blast radius of failures (e.g., if your GSI or Routing Tier fails, the whole system is crippled).