# CAP Theorem – HLD Interview Revision (Staff Engineer Level)

> This note is optimized for HLD/system design interviews (Meta/Google level) and is based on Evan King’s “CAP Theorem in System Design Interviews” plus additional supporting context and examples.[page:1][web:7]

---

## 1. What CAP Theorem Actually Says

CAP theorem (Brewer’s theorem) states that in a distributed data store you can guarantee at most two of the three properties: **Consistency (C)**, **Availability (A)**, and **Partition Tolerance (P)**.[page:1][web:7]  

In practice, when a network partition happens, you must choose between **consistency** and **availability**, because partition tolerance is non‑negotiable in real-world distributed systems.[page:1][web:4][web:6]

### Definitions (Interview-Ready)

- **Consistency (C, in CAP sense)**  
  Every read sees the most recent write or returns an error, so all clients observe the same state at a given logical time.[page:1][web:7][web:4]

- **Availability (A)**  
  Every non-failing node always returns a response (success or failure) to read/write requests, but the data may be stale.[page:1][web:7][web:4]

- **Partition Tolerance (P)**  
  The system continues to operate despite arbitrary message loss, delays, or network splits between nodes.[page:1][web:7][web:4]

> **Senior signaling:** Explicitly distinguish **CAP consistency** from **ACID consistency** and mention that CAP is about behavior under *network partitions* in *distributed* systems, not about single-node transaction semantics.[web:7][web:12]

---

## 2. Why Partition Tolerance Is Non‑Negotiable

In any realistic distributed system (multi-node, multi‑AZ, multi‑region), network failures are inevitable: links fail, zones go down, packets get dropped or delayed.[page:1][web:4][web:6]  

Because you can’t guarantee a perfect network, you must **assume partitions will happen** and design for partition tolerance if you want the system to remain useful.[page:1][web:7][web:6]

**Key point for interviews**

- In a system design interview, once you say “we’ll have replicas / sharding / multi‑region,” you’ve implicitly entered the CAP world; **P is a must**, so the real design choice is **CP vs AP**.[page:1][web:7]

> **Senior signaling:** Say out loud: “Given we’re building a distributed system, I’ll assume partitions are inevitable, so P is required. The interesting trade-off is CP vs AP during partitions.”[page:1][web:6]

---

## 3. Consistency vs Availability: The “Cost of Staleness”

Once P is fixed, your core question becomes:

> **“When there is a partition, do I prefer to serve *stale or possibly incorrect data* (AP) or to *fail/timeout* some requests to protect correctness (CP)?”**[page:1][web:7][web:10]

Evan frames this as: **Is stale data catastrophic or acceptable for this operation?**[page:1][web:15]

- If **stale data is catastrophic**, choose **CP**: you’re OK with errors/timeouts to preserve correctness.[page:1][web:9]
- If **stale data is acceptable for a short time**, choose **AP**: you’re OK with eventually consistent / slightly stale views to keep the system responsive.[page:1][web:9][web:6]

### Library Analogy (Professional)

Imagine a multi-branch library system:

- A user reserves the **last copy** of a rare book at Branch A.
- A network partition prevents Branch B from seeing that reservation.

Two possible policies:

1. **CP (Consistency First)**  
   If Branch B cannot confirm availability with Branch A, it refuses to reserve the book (shows “temporarily unavailable”).  
   No one gets incorrect confirmation; **but** some users get errors.[web:7][web:10]

2. **AP (Availability First)**  
   Branch B assumes there’s still a copy, lets the user reserve it, and resolves conflicts later.  
   Users always get a response, but occasionally **two people think they reserved the same copy**.[web:6][web:9]

> **Senior signaling:** Explicitly label scenarios as “failure = catastrophic vs failure = annoying” and tie that to CP vs AP choice per subsystem.[page:1][web:15]

---

## 4. Running Example: Two‑Region Profile Service

From the video: two servers, one in the USA and one in Europe, replicate user profile data between them.[page:1]

1. User A (US) updates their public profile (e.g., name) in the US region.  
2. The US server tries to replicate this update to the EU server.  
3. A **network partition** happens before replication completes.  
4. User B (EU) reads User A’s profile via the EU server.[page:1]

The system must choose:

- **Option A – CP behavior:** Return an error or block User B because the EU node can’t guarantee a fresh value (strong consistency).[page:1][web:7]
- **Option B – AP behavior:** Return the stale data (old name) to User B, prioritizing availability over strict correctness.[page:1][web:10]

For user profile names, stale reads are usually acceptable, so we pick **availability**, making this an **AP behavior**.[page:1][web:9]

**Pseudo-logic for the decision:**

```pseudo
if network_partition_detected_between(US, EU):
    if operation_requires_strong_consistency(request):
        return error("temporarily unavailable")  // CP
    else:
        return best_effort_stale_read()          // AP
else:
    return normal_read_or_write()
```

> **Senior signaling:** In an interview, walk through a concrete, time-ordered scenario like this and clearly articulate what each user sees under CP vs AP behavior.[page:1]

---

## 5. CP Systems (Consistency + Partition Tolerance)

### When to Choose CP

Choose CP when **incorrect or double-processed data is worse than being temporarily unavailable**.[page:1][web:6][web:15]

Examples (from the video and supporting sources):

- **Ticket booking / airline seats / event tickets**  
  Double-booking a seat is not acceptable; if two users think they own the same seat, you have a real-world conflict at the gate.[page:1][web:9][web:15]

- **Inventory systems (last item)**  
  If you have only one item left and two users both think they bought it, you now must cancel an order or compensate, which is costly.[page:1][web:9][web:15]

- **Financial systems / stock trading**  
  Using stale prices or stale order book state can lead to trades at incorrect prices or inconsistent balances.[page:1][web:3][web:9]

- **Banking / account balances**  
  You would rather show “Service unavailable” than allow an overdraft due to a stale version of the balance.[web:6][web:9]

### CP-Oriented Design Choices

If you decide the system or operation is CP in the presence of partitions:

1. **Distributed transactions / atomicity across components**  
   - Example: cache + DB must move together (or fail together), so you ensure write to both or neither.[page:1]  
   - You might adopt 2PC, saga patterns, or transactional outbox, but at least acknowledge that cross-resource consistency becomes a design focus.[web:6]

2. **Single primary / single logical writer**  
   - Use a **single database instance or a strongly consistent primary** for writes to avoid conflicting concurrent writes from multiple regions.[page:1][web:7]  
   - Others can be read replicas but with careful read-routing (e.g., read-your-own-writes from primary).[web:6]

3. **Higher latency acceptance**  
   - You may need to coordinate across regions or wait for quorum acknowledgements before confirming a write, which increases tail latency.[page:1][web:6]

4. **Technology examples**  
   - Traditional RDBMS with strong transactional guarantees: **PostgreSQL, MySQL, SQL Server**, etc.[page:1][web:6]  
   - **Google Spanner**: globally distributed relational DB with external consistency/true-time; often used where global strong consistency matters.[page:1][web:12]  
   - NoSQL with strong consistency modes, e.g. **DynamoDB with strongly consistent reads**, or strongly consistent configs of MongoDB.[page:1][web:5][web:9]

> **Senior signaling:** Don’t just say “I’ll use Postgres.” Explain **why**: “For this core transactional path, I want linearizable semantics and transactions; therefore I’ll centralize writes on a strongly consistent store and absorb the latency.”[page:1][web:6]

---

## 6. AP Systems (Availability + Partition Tolerance)

### When to Choose AP

Choose AP when **serving slightly stale data is acceptable and system responsiveness matters more than perfect freshness.**[page:1][web:6][web:9]

Examples from the video:

- **Social feeds / posts / likes**  
  If you see 99 likes and I see 100, nobody dies; the difference converges quickly.[page:1][web:3][web:9]

- **Profile photos / public profile metadata**  
  Seeing an old picture or previous name for a few seconds is fine as long as the system is fast and available.[page:1][web:3]

- **Business listings (Yelp-style) / menu data**  
  A slightly outdated menu item or business hours for a short window is acceptable compared to refusing to show the business at all.[page:1][web:3][web:9]

- **Content catalogs (e.g., Netflix movies list)**  
  If one region sees a new movie a bit later than another or sees an old description briefly, it does not break core product value.[page:1][web:9]

### AP-Oriented Design Choices

If you decide an operation is AP:

1. **Multiple replicas and scale-out reads**  
   - Serve reads from many replicas, possibly across regions, accepting that replicas may briefly diverge.[page:1][web:6]  
   - Use asynchronous replication; high availability under failures is prioritized over strict ordering.

2. **Eventual consistency and CDC**  
   - Use **change data capture (CDC)** or log-based replication to propagate updates asynchronously between services and regions.[page:1][web:6]  
   - Design clients and downstream systems to tolerate and reconcile eventual convergence.

3. **Conflict resolution strategies**  
   - Last-write-wins, version vectors, or domain-specific merge strategies for concurrent updates to the same logical entity (e.g., counters, preferences).[web:6]

4. **Technology examples**  
   - **DynamoDB** with eventually consistent reads and multi‑AZ or multi‑region setups.[page:1][web:9]  
   - **Cassandra** and similar AP-leaning databases optimized for high availability and horizontal scale.[page:1][web:6][web:9]  

> **Senior signaling:** When you say “I’ll make this AP,” immediately follow with: “That implies asynchronous replication and eventual consistency. Users may briefly see stale data, which is acceptable for this surface, and we’ll design UX to hide minor inconsistencies.”[page:1][web:6]

---

## 7. Consistency Spectrum (Beyond “Just Consistent”)

The video calls out that when people say “consistency” in CAP discussions, they usually mean **strong consistency**, but there’s a spectrum of consistency models you can apply to different parts of the system.[page:1][web:12]

### 7.1 Strong Consistency

- **Definition:** All reads reflect the latest write (or fail), giving the illusion of a single, up-to-date copy.[page:1][web:7][web:4]  
- **Use case:** Money, tickets, inventory, match decisions—where correctness is paramount.[page:1][web:9]

### 7.2 Causal Consistency

- **Definition:** All nodes observe **causally related events** in the same order, but unrelated events may be seen in different orders.[page:1][web:8][web:11]  
- Example: you must not show a **reply comment** before the **original comment**, even if both propagate asynchronously.[page:1][web:8]

### 7.3 Read-Your-Own-Writes (RYOW)

- **Definition:** A client that performs a write is guaranteed to see its own write on subsequent reads, even if other users may still see stale data.[page:1][web:14]  
- Example: If I update my profile photo, I should immediately see the new photo; others may see the old photo for a while.[page:1][web:14]

### 7.4 Eventual Consistency

- **Definition:** If no new updates are made, all replicas eventually converge to the same value; no guarantees about how long it takes, or what intermediate values are observed.[page:1][web:6][web:1]  
- This is what you implicitly choose when you prioritize availability over strong consistency under partitions.[page:1][web:6]

> **Senior signaling:** In a design, say: “Core balance updates are **strongly consistent**, comments are **causally consistent**, and profile metadata is **eventually consistent with read‑your‑own‑writes guarantees for the actor**.”[page:1][web:11]

---

## 8. Decomposed Design: Different CAP Priorities Per Subsystem

A key “staff-level” nuance from the video: **you do not pick CP *or* AP for the entire system once and for all.**[page:1][web:6]  

Instead, you **classify flows / subsystems** and assign different CAP priorities depending on their business risk and UX needs.[page:1][web:13]

### 8.1 Ticket Master Example

From the video:[page:1]

- **Booking tickets (seat assignment)**  
  - Must avoid double booking.  
  - **Behavior:** CP – prioritize strong consistency; show errors over stale availability.[page:1][web:9][web:15]

- **CRUD on events (event metadata)**  
  - Updating event descriptions, images, etc. can be eventually consistent; stale metadata for a bit is fine.  
  - **Behavior:** AP – prioritize availability so events are always viewable.[page:1][web:13]

How to say this in an interview:

> “For Ticket Master, I’ll treat **ticket booking** as CP—if we can’t confirm availability, I’d rather fail than oversell. For **event metadata CRUD**, I’ll make it AP—slightly stale descriptions are acceptable; system responsiveness is more important.”[page:1][web:15]

### 8.2 Tinder Example

From the video:[page:1]

- **Matching logic (swipes → match)**  
  - If User A swipes in Europe and User B swipes later in the US, we want the match to appear immediately and correctly.  
  - Needs a consistent view of “who swiped whom” to avoid duplicate or missing matches.  
  - **Behavior:** CP (or at least strongly consistent subset) for match state.[page:1]

- **Viewing / updating profile data**  
  - If someone updates their bio or picture, it’s fine if other users see the old version for a short time.  
  - **Behavior:** AP with eventual consistency and possibly RYOW for the owning user.[page:1][web:11]

How to say it in an interview:

> “In Tinder, **match creation** runs on a strongly consistent store, possibly centralizing writes or using a global index. Profile data, photos, and likes can be AP and eventually consistent across regions.”[page:1][web:6]

> **Senior signaling:** Proactively decompose the system into **“money/critical correctness paths”** (CP) and **“engagement/UX paths”** (AP) and articulate the trade-offs for each.[page:1][web:13][web:15]

---

## 9. Practical Interview Playbook: How To Use CAP

This section is about **how you talk through CAP** in a 45–60 minute system design interview.

### 9.1 Early in the Interview: Non‑Functional Requirements

After clarifying functional requirements, explicitly call out CAP when discussing non‑functional requirements.[page:1]

Script-like flow:

1. “Let’s talk about non-functional requirements: latency targets, data freshness, availability SLOs.”[page:1]
2. “Since this is a distributed system, we must assume partitions and therefore support partition tolerance.”[page:1][web:4]
3. “The main question is: during a partition, do we prioritize **consistency** or **availability** for this core operation?”[page:1][web:7]
4. “In this domain, the cost of stale data is [catastrophic/acceptable], so I’ll lean [CP/AP] for this path.”[page:1][web:15]

### 9.2 Per-Operation CAP Decision Template

For each major operation, mentally run this decision:

```pseudo
function choose_CAP_behavior(operation):
    if operation_failure_is_catastrophic(operation):
        return CP  // prefer errors/timeouts over stale data
    else:
        return AP  // prefer serving stale data over failing
```

Examples:

- **Book seat / charge card / transfer funds → CP**.[page:1][web:15]
- **Update profile photo / post a comment / add a review → AP**.[page:1][web:3][web:9]

### 9.3 Mapping CAP Decision to Concrete Design

Once you say “CP” or “AP,” show the interviewer you know what that implies:

- **For CP:**
  - Single-writer or strongly consistent store.  
  - Possible transactions or compensating logic.  
  - Willingness to accept increased latency and potential failures/timeouts during partitions.[page:1][web:6]

- **For AP:**
  - Async replication and eventual consistency.  
  - Multi-region / multi-AZ read replicas.  
  - UX patterns that tolerate and hide minor inconsistencies (e.g., local caching, optimistic UI, background refresh).[page:1][web:6][web:13]

> **Senior signaling:** Every time you make a CAP trade-off, connect it explicitly to **user experience** (“what might the user see”) and **business risk** (“what happens if we’re wrong”).[page:1][web:12]

---

## 10. Additional Analogies for Intuition

### 10.1 Restaurant Kitchen Analogy

- **Kitchen orders = writes**, **waiters delivering dishes = reads**.

1. **CP kitchen**  
   - If the communication between the head chef and line cooks is broken (partition), the restaurant **stops serving** certain dishes until they can synchronize.  
   - No wrong orders go out, but some customers wait or are turned away.[web:6][web:10]

2. **AP kitchen**  
   - Even if the connection to head chef is flaky, line cooks **guess** based on last known menu and serve dishes.  
   - Some dishes might be slightly wrong or missing modifications, but the restaurant stays busy.[web:6]

### 10.2 Trading Floor Analogy

- **CP mode:** Trading halts on a stock when price feeds are inconsistent; better to halt than trade on bad data.[web:9]  
- **AP mode:** News feed about the market keeps flowing even if some headlines are slightly delayed.[web:9]

Use these analogies when an interviewer appears non‑backend or when you want to simplify a complex trade-off.

---

## 11. Quick Reference: Examples by Category

### CP‑Leaning Operations (Strong Consistency)

- Banking: balance updates, transfers, loan disbursements.[page:1][web:6][web:9]  
- Trading: order book updates, trade execution, settlement.[page:1][web:3][web:9]  
- Ticketing: seat allocation, ticket issuance.[page:1][web:15]  
- Inventory: last item stock decrement.[page:1][web:15]  
- Matching: Tinder-style matches, ride assignment for ride-sharing.[page:1][web:6]

### AP‑Leaning Operations (High Availability + Eventual Consistency)

- Social feeds: posts, likes, comments (with causal constraints where needed).[page:1][web:3][web:11]  
- Profile metadata: names, bios, photos, preferences.[page:1][web:3]  
- Reviews and ratings: Yelp, Amazon product reviews.[page:1][web:3]  
- Content catalogs and recommendations: Netflix suggestions, “People you may know.”[page:1][web:9]

> **Senior signaling:** When asked for “examples of CP vs AP,” don’t just list domains—tie each to what goes wrong if data is stale vs unavailable.[page:1][web:15]

---

## 12. Senior-Level Notes & Pitfalls

- **CAP does not apply to non-distributed systems.**  
  If your design is single-node with no replication, CAP is not the right lens; focus instead on ACID, indexing, sharding, etc.[web:7][web:12]

- **CAP is about behavior *during partitions*, not normal operation.**  
  When the network is healthy, systems can often appear to satisfy C, A, and P simultaneously.[web:7][web:13]

- **CA systems in practice are very narrow.**  
  Once you introduce real replication over a non-zero-latency network, P becomes mandatory; pure CA systems are basically single-node or special environments.[web:7][web:6]

- **Overusing “strong consistency” everywhere is a scalability trap.**  
  It increases latency, reduces availability, and complicates global deployments; use it **surgically** where correctness really matters.[page:1][web:6]

- **Overusing AP without clear conflict semantics leads to data corruption.**  
  If you make balances or tickets AP and don’t define merge rules, you will get inconsistencies that can’t be reconciled.[web:6][web:9]

> **Senior signaling:** Acknowledge nuance: “CAP is an extreme model during partitions; in practice we combine replication, caching, retries, and consistency models to get ‘good enough’ behavior most of the time.”[web:6][web:13]

---

## 13. Interview Takeaways & Revision Checklist

Use this as your last‑minute revision list before HLD rounds.

### 13.1 Concept Checklist

- [ ] Can I define **C**, **A**, and **P** in one sentence each, in CAP terms? [page:1][web:7]  
- [ ] Can I explain why **P is non‑negotiable** in distributed systems? [page:1][web:4]  
- [ ] Can I walk through a **two‑region partition scenario** and explain CP vs AP behavior concretely? [page:1]  
- [ ] Can I give **at least three CP** and **three AP** real-world examples and justify them? [page:1][web:9][web:15]  
- [ ] Do I understand **strong, causal, RYOW, and eventual consistency**, with at least one example each? [page:1][web:11][web:14]

### 13.2 Communication Checklist (Senior Signaling)

- [ ] Early in the interview, explicitly call out: “We must be partition tolerant, so I’ll discuss CP vs AP trade-offs.”[page:1][web:6]  
- [ ] For each major operation, say whether stale data is **catastrophic vs annoying**, and choose CP or AP accordingly.[page:1][web:15]  
- [ ] Decompose the system into **CP vs AP subsystems** (e.g., bookings vs search, matches vs profiles).[page:1][web:13]  
- [ ] When naming a storage technology, explain **why its consistency model fits** the requirement (e.g., Spanner vs Cassandra vs DynamoDB mode).[page:1][web:6][web:9]  
- [ ] Tie CAP trade-offs back to **user experience** and **business risk**, not just theory.[page:1][web:12]

---

You can now save this entire block as `cap-theorem-hld-revision.md` and use it as your primary CAP cheat sheet before HLD/system design interviews.[page:1]