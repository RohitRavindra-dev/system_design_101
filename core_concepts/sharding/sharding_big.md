# Database Sharding – HLD Interview Revision (SE2/Senior)

## Overview

This document distills sharding concepts as explained by Evan King (ex‑Meta staff engineer) in the Hello Interview sharding video and write‑up, with additional depth for senior‑level HLD interviews. It focuses on how to talk about sharding in an interview: when you actually need it, how to design it, and how to handle the distributed pain it introduces.[1]

Use this as a revision sheet, not a tutorial from scratch. You are assumed to be comfortable with basic relational DBs, indexing, and caching.

***

## Scaling & Partitioning Taxonomy

### Vertical vs Horizontal Scaling

**Vertical scaling (scale‑up)** means buying a bigger box: more CPU, RAM, and storage for a single database instance. Cloud RDBMSes (Aurora, RDS, etc.) can reach hundreds of terabytes and very high write throughput before you truly need sharding.[1]

**Horizontal scaling (scale‑out)** means adding more machines and distributing load across them, typically by sharding. Sharding lets total storage and read/write throughput grow with the number of shards, at the cost of distributed systems complexity.[2][3][1]

**Interviewer pro‑tips**

- First show your math that a single DB (plus replicas) is insufficient before proposing sharding (storage, writes/sec, reads/sec).[1]
- Explicitly say when you *do not* need sharding yet; this shows mature judgment instead of sharding as a reflex.[1]

### Partitioning vs Sharding

- **Partitioning**: splitting tables into logical pieces *within one database instance* (e.g., Postgres table partitioning).[1]
- **Sharding**: horizontal partitioning of data *across multiple machines/DB instances*.[1]

Partitioning improves performance and maintainability on a single host; sharding adds capacity by distributing data and load across hosts.[1]

### Horizontal vs Vertical Partitioning

- **Horizontal partitioning**: rows are split; each partition has the same columns, fewer rows (e.g., orders by year).[1]
- **Vertical partitioning**: columns are split; each partition has fewer columns (e.g., hot columns vs cold blob columns).[1]

### Taxonomy Cheat‑Sheet Table

```text
Legend
- Node: physical DB instance (machine or managed instance)
- Partition: logical slice of a table
- Shard: DB instance holding a subset of the data
```

| Concept                    | Where it happens             | What changes                        | Pros                                              | Cons                                                       | Typical use case                          |
|---------------------------|------------------------------|--------------------------------------|---------------------------------------------------|------------------------------------------------------------|-------------------------------------------|
| Vertical scaling          | Single node                  | Bigger CPU/RAM/disk only             | Simple, no app changes                            | Hard limits on size/IO; expensive hardware                 | Early growth, small/medium products       |
| Horizontal partitioning   | Single node                  | Rows split into partitions           | Better index/maintenance, faster scans per range  | Still one node’s CPU/IO; single failure domain             | Very large tables, time‑series            |
| Vertical partitioning     | Single node                  | Columns split into partitions        | Smaller working set for hot columns               | Joins across partitions; more complex schema               | Separating hot vs cold data               |
| Sharding (horizontal)     | Multiple nodes (shards)      | Rows split across shards             | Scales storage and R/W throughput linearly-ish     | Cross‑shard joins, consistency, resharding, routing logic   | High‑scale OLTP (social, e‑com, SaaS)     |

**Interviewer pro‑tip**: When asked “sharding vs partitioning?”, answer: *“Partitioning is intra‑node; sharding is inter‑node. In interviews I care about sharding when one node’s storage or throughput ceiling is provably exceeded.”*[1]

***

## Shard Key Design

Sharding design is dominated by your **shard key**: the field you use to group data (e.g., `user_id`). A good shard key has three properties:[1]

1. **High cardinality** – many distinct values, so data can spread across shards (e.g., `user_id`, `order_id`).
2. **Even distribution** – values do not cluster heavily on a subset of shards (avoid `country` when 90% of users are in one country).[1]
3. **Query alignment** – common queries can be satisfied from a single shard (e.g., “get all posts for user X” ⇒ shard by `user_id`).[1]

**Good examples**[1]

- `user_id` for a social app: user‑centric reads/writes, millions of users.
- `order_id` for an orders table in e‑commerce: operations focused on individual orders.[1]

**Bad examples**[1]

- `is_premium` (boolean): only two values ⇒ effectively only two shards.
- `created_at` for a feed table: all new writes hammer the “latest” shard (time hotspot).

**Interviewer pro‑tips**

- Always state your shard key *explicitly* and justify it against these three properties.[1]
- If the interviewer grills your design, check for misalignment between shard key and hot query paths before anything else.[1]

***

## Sharding Strategies & Read/Write Trade‑offs

Once a shard key is chosen, you must decide **how** its values map to shards. Three canonical strategies:[3][1]

1. Range‑based sharding
2. Hash‑based sharding (usually with consistent hashing)
3. Directory‑based sharding

### Range‑Based Sharding

**Idea**: assign contiguous ranges of shard key values to each shard.[3][1]

```text
Shard 1: user_id 1 – 1,000,000
Shard 2: user_id 1,000,001 – 2,000,000
Shard 3: user_id 2,000,001 – 3,000,000
```

**Read characteristics**

- Point lookups: O(1) once you know the range; router maps key to shard by simple comparisons.[3]
- Range scans on shard key (e.g., `user_id` between X and Y or time ranges) are very efficient — often hit a single shard or a small subset.[2][3]
- Global queries (unbounded ranges or queries on non‑key attributes) require scatter‑gather across shards.

**Write characteristics**

- If key values increase monotonically (auto‑increment IDs, timestamps), all new writes go to the highest‑range shard, creating a hot shard and poor load balance.[2][1]
- Early in system life, only the first shard may hold real data and handle traffic; other shards sit idle.[1]

**When it shines**

- Tenants naturally fall into disjoint ranges (e.g., `tenant_id` ranges for B2B SaaS where each tenant’s traffic is relatively balanced).[1]
- Workloads rely heavily on range queries over the shard key.

**Diagram – Range Sharding**

```text
          +---------------------+
          |   Range Router      |
          +----------+----------+
                     |
        +------------+------------+
        |            |            |
   [1–1M]        [1M–2M]     [2M–3M]
   Shard 1       Shard 2      Shard 3
```

**Interviewer pro‑tips**

- If you choose range sharding, immediately acknowledge the “newest‑range hotspot” risk and talk about pre‑splitting / dynamic range splits.[2][1]
- Emphasize that most large production systems default to hash‑based sharding instead; range is usually for specialized range‑heavy workloads.[3][2]

### Hash‑Based Sharding (Default)

**Idea**: hash the shard key and take modulo by shard count (or use a hash ring) to pick a shard.[2][1]

```text
shard_id = hash(user_id) mod N
```

**Read characteristics**

- Point lookups: excellent, since data is evenly spread; a single shard handles each key.[3][2]
- Range queries on the shard key become expensive — values in a numeric or time range are spread across many shards, forcing scatter‑gather.[2]

**Write characteristics**

- Writes are naturally balanced across all shards (assuming good hash & key cardinality).[2]
- Naive modulo scheme makes resharding extremely painful: changing from `mod N` to `mod N+1` remaps almost every key.[2]

**Consistent hashing**

- Production systems mitigate resharding pain with **consistent hashing**: both keys and shards are placed on a hash ring, each shard owns a segment; adding/removing shards only moves keys for affected segments.[2][1]
- Frameworks like Cassandra’s partitioner and Vitess’s vindexes implement similar ideas with token ranges and virtual nodes.[4][1]

**When it shines**

- Default choice when you want even load distribution and primarily do key‑based lookups.[2][1]

**Diagram – Hash Sharding with Ring**

```text
        (hash space ring)

       [Shard A]
         /   \
        /     \
   key K1     key K2
 (first shard clockwise >= hash(K))
```

**Interviewer pro‑tips**

- If you say “I’ll shard by user_id”, most senior interviewers assume “hash‑based with consistent hashing” unless you say otherwise.[1]
- Be ready to explain resharding at a high level: add shards, stream data, update routing, minimize movement via consistent hashing.[2][1]

### Directory‑Based Sharding

**Idea**: maintain a mapping from shard key to shard in a central lookup table or service, instead of a formula.[3][1]

```text
user_to_shard
-------------
User 15  → Shard 1
User 87  → Shard 4
User 204 → Shard 2
```

**Read characteristics**

- Every request incurs an extra hop to the directory (unless cached), adding latency and creating a critical dependency.[1]
- Once the shard is known, reads behave like whatever sharding strategy is used per shard.

**Write characteristics**

- Flexible: individual hot keys can be moved between shards by updating the mapping; rebalancing doesn’t require changing the data’s primary key.[5][1]
- Directory becomes a write bottleneck if not carefully scaled; also adds complexity for keeping mapping consistent.

**When it shines**

- Handling exceptional cases like celebrity users or tenant migrations by overlaying directory logic on top of a hash‑based baseline.[5][1]

**Diagram – Directory‑Based Sharding**

```text
Client
  |
  v
+-----------+      +-------------+
| Directory | ---> |   Shard N   |
+-----------+      +-------------+
```

**Interviewer pro‑tips**

- In interviews, present directory‑based sharding as an **add‑on** for edge cases (like celebrity accounts), not your primary base strategy.[5][1]
- Call out the single‑point‑of‑failure and latency downside; mention caching and HA for the directory if pressed.[1]

### Strategy vs Read/Write Trade‑off Table

| Strategy        | Read profile                                         | Write profile                                             | Operational notes                                    |
|----------------|------------------------------------------------------|-----------------------------------------------------------|------------------------------------------------------|
| Range‑based    | Great for range queries on shard key; cheap point reads when key → range; global queries scatter‑gather | Writes skew to most recent range if key is monotonic; risk of hot shards | Need manual/dynamic range splits; good for tenant ranges |
| Hash‑based     | Great for point reads; range queries are scatter‑gather | Writes distributed evenly across shards                   | Resharding requires consistent hashing to avoid full shuffle |
| Directory‑based| Extra lookup per read; flexible per‑key routing       | Flexible key moves; directory can bottleneck writes       | Critical dependency; used for special cases (celebs, tenants) |

***

## The Hot Spot / Celebrity Problem

A **hot spot** occurs when one shard handles disproportionately high traffic compared to others, negating the benefits of sharding. The classic example is the **celebrity problem**: if users are sharded by `user_id` and a celebrity like Messi lives on shard 1, that shard gets hammered with profile views, likes, comments, and messages while others remain underutilized.[5][1]

### Strategies to Handle Hot Keys

1. **Compound/salted shard keys**
   - Instead of sharding purely on `user_id`, shard on a composite like `hash(user_id, bucket)` where `bucket` is a small integer (0…K‑1).[6][1]
   - This spreads a single user’s data across K physical shards while preserving logical grouping via a higher‑level user abstraction.[1]
   - Reads for that user’s entire history become scatter‑gather across their K shards; reads of recent windows can be routed to subset buckets (e.g., time‑bucketed keys).

2. **Dedicated celebrity shard(s)**
   - Maintain a directory mapping for high‑traffic users; move them onto dedicated shards with beefier hardware or tuned configuration.[5][1]
   - Routing layer checks “is this a celebrity?”; if yes, go to celebrity shard, otherwise default hash‑based routing.

3. **Write sharding for indexes / GSIs**
   - On systems like DynamoDB, hot keys in a **Global Secondary Index (GSI)** can be mitigated by prepending a random shard number to the index key and performing a scatter‑read over those shards.[7][6]
   - This spreads write load at the cost of more complex read paths.

**Diagram – Celebrity Shard Overlay**

```text
               +------------------+
Request (uid)  |  Routing Layer   |
-------------->|------------------|
               | if uid in celebs |
               |   -> Celebrity DB|
               | else             |
               |   -> Hash ring   |
               +---------+--------+
                         |
    +--------------------+-------------------+
    |                    |                   |
 Celebrity Shard     Shard A             Shard B
```

**Interviewer pro‑tips**

- Name the problem (“celebrity problem / hot key / hot partition”) and explicitly tie it to uneven load across shards.[5][1]
- Offer **two levels** of mitigation: (1) shard key design (compound keys, salts) and (2) routing‑layer exceptions (celebrity shard + directory mapping).[5][1]

***

## Distributed Complexity & Pain Points

Once data lives on multiple shards, three classes of complexity appear:[2][1]

1. Cross‑shard joins and queries
2. Distributed transactions
3. Indexing on fields that are not the shard key (global secondary indexes)

### Cross‑Shard Joins and Global Queries

If data for a query lives on multiple shards, the system must perform **scatter‑gather**: fan the query out to all relevant shards, wait for responses, merge results, and sometimes sort/aggregate on the coordinator.[2][1]

Typical examples:[1]

- “Top 10 posts globally” when sharding by `user_id`.
- Joins between entities sharded on different keys (e.g., `users` by `user_id` and `organizations` by `org_id`).

**Mitigation strategies**[2][1]

- **Align shard key with hot queries**: make sure the read path that must be fast hits a single shard whenever possible (user‑centric sharding for profile/feed reads).[1]
- **Caching heavy global queries**: compute “top N” or aggregates once (scatter‑gather), store them in Redis for a short TTL (e.g., 1–5 minutes), trade freshness for latency.[1]
- **Precomputation / background jobs**: offload global aggregates to periodic batch jobs; frontends read from a precomputed table keyed by something shard‑local.[1]
- **Denormalization**: duplicate data so that common join patterns can be satisfied from a single shard (e.g., store basic user info alongside posts).

**Interviewer pro‑tips**

- When you hear yourself say “we’ll just query all 64 shards every time”, immediately follow with caching/denormalization strategies and frame it as an **exception path**, not the hot path.[1]

### Distributed Transactions – 2PC vs Sagas

With one database, a transaction can atomically update multiple rows/tables with ACID guarantees. With sharding, participating rows may live on different shards, so a single local transaction is impossible.[1]

**Two‑phase commit (2PC)**[1]

- A coordinator asks all participating shards to **prepare** the transaction, then if all agree, issues a **commit**.
- Guarantees atomicity but is slow and fragile: if any shard or the coordinator fails mid‑protocol, participants may block, and recovery is complex.[1]

**Saga pattern (eventual consistency)**[1]

- Break the transaction into a sequence of local steps (each on a single shard), each with a compensating action.
- Example for cross‑shard bank transfer:
  1. Deduct amount from A on shard 1.
  2. Credit amount to B on shard 2.
  3. If step 2 fails, run compensating step: refund A.
- Provides eventual consistency and avoids 2PC’s blocking but complicates business logic.

**Design principles**[1]

- Prefer shard keys that keep transactional boundaries inside a single shard.
- Use sagas and eventual consistency only for unavoidable cross‑shard business flows.

**Interviewer pro‑tips**

- If asked about transactions across shards, outline: *“Avoid when possible via shard design; else, use sagas with compensating actions; 2PC is last resort due to latency and failure sensitivity.”*[1]

### Global Secondary Indexes (GSIs)

A **secondary index** lets you query by a non‑primary key field (e.g., `email`, `genre`, `status`). In a single‑node DB, indexes live on the same machine and can be updated transactionally with the base row.[8]

In sharded systems, you must decide how to shard the index itself:[9][10][8]

1. **Local secondary indexes (LSI‑style)**
   - Each shard maintains an index only for rows it owns.
   - Queries using the index must either:
     - Know which shard to hit (e.g., index includes shard key), or
     - Scatter‑read the index across shards and merge results.
   - Pros: fast writes, no cross‑shard index updates.
   - Cons: cross‑shard reads for lookups not aligned with shard key.

2. **Global secondary indexes (GSI‑style)**
   - Index is re‑partitioned by its own key, potentially on a different shard key than the base table.[10][9]
   - Pros: query by index key can hit a single index partition instead of all data shards; global view of attribute values.[9]
   - Cons: writes must update both base table and index, often across shards; more complex failure and consistency handling.[8]

3. **Write‑sharded GSIs**
   - To avoid hot partitions when indexing high‑cardinality but skewed attributes (e.g., “most recent events for a user”), systems like DynamoDB recommend adding a random shard prefix to the index partition key and scatter‑reading across these shards.[7][6]

**Interviewer pro‑tips**

- At senior levels, mentioning *“secondary indexes get tricky under sharding: either local (scatter reads) or global (scatter writes)”* shows depth.[8][9]
- Tie GSI design back to your hot paths: do you optimize write throughput or index read latency?

***

## Operational Realities – Resharding

Eventually some shards grow too large or too hot and you must **reshard**: change the mapping from keys to shards while the system is live.[2][1]

### Goals

- Add or remove shards with minimal or no downtime.
- Avoid moving more data than necessary (use consistent hashing, range splits, or chunk moves).
- Keep reads and writes serving during migration, or at worst accept a brief read‑only window.

### Typical Online Resharding Flow

Modern sharding frameworks (e.g., Vitess) provide operator‑driven **online resharding** workflows.[11][12][13][4]

High‑level phases:[11][4]

1. **Plan split/merge**
   - Decide source shards and target shard key ranges (e.g., split shard `0–FF` into `0–7F` and `80–FF`).[4][11]

2. **Provision new shards**
   - Start target shard instances but keep them out of serving traffic.[11]

3. **Bulk copy**
   - Copy data from source shards to target shards using snapshot + streaming (e.g., Vitess VReplication).[4][11]
   - Source shards continue serving reads and writes.

4. **Catch‑up replication**
   - Continuously replicate new changes (e.g., binlog) from source to targets until lag is near zero.[11][4]

5. **Traffic switch**
   - Gradually move read traffic to new shards, then perform a short coordinated cut‑over of writes (often seconds of read‑only while pointers are swapped).[4][11]

6. **Decommission old shards**
   - Once confident, stop replication and retire source shards.[11][4]

**Diagram – Split 2 shards → 4 shards**

```text
Before:                    After:

  Router                     Router
    |                           |
  [0–FF]                    [0–3F][40–7F][80–BF][C0–FF]
  Shard A                   S1    S2    S3    S4

Migration: copy+replicate A → S1..S4, then atomically switch routing.
```

**DIY resharding (homegrown systems)** generally follow the same pattern: dual‑write or replicate to new shards, validate, then flip routing, accepting a small window of risk or degraded mode.[2]

**Interviewer pro‑tips**

- You don’t need step‑by‑step Vitess commands; you do need to say “online resharding via background copy + incremental replication + cut‑over”.[4][11]
- Emphasize that “just change mod N to mod N+1” is not acceptable in production; you must minimize data movement and avoid global rewrites.[2]

***

## Routing Tier Design – Where Sharding Logic Lives

Sharding introduces a **routing problem**: given a request and a shard key, which shard should it hit? This logic can live in several layers.[14][1]

### Option 1 – Client‑Side Routing (Smart Clients)

- Each service uses a library that knows the shard topology (key ranges or hash ring) and sends queries directly to the right shard.[15][14]
- Topology metadata is pushed via config service or discovery and cached locally in the client.[14]

**Pros**

- No extra network hop beyond the DB connection.
- Enables data‑dependent routing close to business logic; connection pools can be per‑shard.[14]

**Cons**

- Every client must understand sharding; multi‑language environments require multiple client implementations.[16]
- Rolling out resharding changes requires coordinated client updates or robust config distribution.

**When to mention in interviews**

- When designing internal microservices with strong language standardization and a powerful shared data access layer.

### Option 2 – Proxy / Gateway Layer (e.g., Vitess)

- Applications talk to a logical “single” database endpoint (SQL or protocol‑compatible proxy).[16]
- The proxy parses queries and routes them to the right shards, handling scatter‑gather, resharding, and sometimes cross‑shard transactions.[16][4]

Examples: Vitess for MySQL, Citus for Postgres, Oracle sharding coordinators.[14][16]

**Pros**

- Applications see a mostly normal database; sharding is abstracted away.[16]
- Operator‑driven online resharding and routing changes don’t require app changes.[13][4]

**Cons**

- Proxy tier becomes critical infrastructure that must be scaled and made highly available.[16]
- Extra network hop adds latency (though often acceptable versus the complexity saved).

**Diagram – Proxy Routing**

```text
App → Proxy / VTGate → Shard A/B/C
```

**When to mention in interviews**

- When you want to show realistic production practice: *“We’d deploy Vitess in front of MySQL to handle sharding, routing, and online resharding.”*[4][16]

### Option 3 – Sidecar / Mid‑Tier Routing

- Each application pod has a local sidecar process responsible for routing to shards.[14]
- The sidecar may share logic with a central routing service but lives adjacent to the app in the same host or pod.

**Pros**

- Keeps app code simple (talk to localhost), while decoupling routing implementation.[14]
- Easier to roll out routing updates than touching every codebase.

**Cons**

- Operationally similar to managing many small proxies; must ensure consistent configuration and health.

**Interviewer pro‑tips**

- Frame routing tier choice as a trade‑off between latency, operational complexity, and how many teams touch the DB directly.[16][14]
- For a generic HLD answer, a **proxy/gateway** is often the cleanest story; client‑side routing is a good “advanced nuance” follow‑up.[16]

***

## Putting It Together in an Interview

### When to Introduce Sharding

Introduce sharding only after you show that a single database plus replicas cannot handle storage or throughput:[1]

- **Storage**: e.g., 500M users × 5KB ≈ 2.5 TB – still fits in a single strong RDBMS; sharding not yet mandatory.[1]
- **Write throughput**: if expected writes/sec exceed what a single instance can handle, even with tuning, propose sharding.[1]
- **Read throughput**: even with read replicas, massive DAU with many queries may require spreading reads across shards.[1]

### A 4‑Step Sharding Story (Social App Example)

Patterned on the Hello Interview guidance:[1]

1. **Propose shard key based on access pattern**  
   “Most operations are user‑centric (profile, feed, likes), so shard by `user_id` to keep a user’s data on one shard.”[1]

2. **Choose distribution strategy**  
   “Use hash‑based sharding with consistent hashing for even distribution and smooth resharding.”[2][1]

3. **Call out trade‑offs and mitigations**  
   “Global queries like trending posts require cross‑shard fan‑out. We’ll precompute/cache them and, where needed, denormalize data to minimize cross‑shard joins.”[1]

4. **Explain growth and resharding plan**  
   “Start with N shards; as we grow, use online resharding (background copy + binlog streaming + cut‑over) to add shards without downtime.”[11][4]

### Senior‑Level Signals to Sprinkle In

- Use capacity math to justify or reject sharding (orders of magnitude, not exactness).[1]
- Talk about **hotspots**, **cross‑shard operations**, and **consistency** as *inherent* trade‑offs, not bugs.[2][1]
- Mention **consistent hashing**, **sagas vs 2PC**, and **global vs local secondary indexes** briefly, tying each to your design choices.[9][2][1]
- Place sharding logic in a realistic routing tier (proxy/Vitess, managed sharded DB, or robust client library) instead of hand‑rolling everything in the app.[16][1]

If you can walk through this narrative crisply, you will sound like someone who has both implemented and operated sharded systems, not just memorized the buzzwords.