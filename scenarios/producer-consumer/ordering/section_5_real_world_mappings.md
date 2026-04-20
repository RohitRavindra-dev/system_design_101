# Section 5: Real-world mappings / examples

## 1. Apache Kafka (distributed streaming platform)

Where it fits: - Log ingestion - Event streaming - Analytics pipelines

Mapping: - Topic → stream of events\
- Partition → ordered log\
- Key → determines partition

Ordering used: - Default: per-key ordering - Global ordering → only if 1
partition (rare)

Real use case: - User activity tracking - Clickstream processing

------------------------------------------------------------------------

## 2. Amazon Kinesis (AWS streaming service)

Same concept, different name

-   Partition = shard
-   Key = partition key

Real use case: - Real-time analytics dashboards - IoT data ingestion

------------------------------------------------------------------------

## 3. Payment / Ledger systems (important contrast)

Example: - Banking core system - Blockchain

Ordering: - Often global ordering (or very strict sequencing)

Why: - Balance correctness depends on order

Tradeoff: - Lower throughput - High coordination

------------------------------------------------------------------------

## 4. Food delivery (refined)

Flow: - Order lifecycle: - placed → accepted → prepared → picked →
delivered

Ordering: - Per-order (per-key)

Why: - Each order independent

Insight: This is why systems scale well here

------------------------------------------------------------------------

## 5. Chat systems (subtle but common)

Example: - WhatsApp / Slack

Ordering: - Per chat / conversation

Why: - Messages in same chat must be ordered - Different chats
independent

Design: - chat_id → partition key

------------------------------------------------------------------------

## 6. Logging / metrics systems

Example: - App logs - Metrics pipelines

Ordering: - No ordering required

Why: - Events independent

Benefit: - Max scalability

------------------------------------------------------------------------

## Pattern recognition

  System Type        Ordering Type
  ------------------ ---------------
  Logs / metrics     None
  User workflows     Per-key
  Financial ledger   Global

------------------------------------------------------------------------

## Interview trick

When given a system:

First identify what entity needs ordering

That gives you: - Partition key - Architecture direction

------------------------------------------------------------------------

## Q&A (Checkpoint)

Scenario: Ride sharing system (Uber)

Events: - Ride requested - Driver assigned - Ride started - Ride
completed

Answer:

1.  Ordering requirement: Per-ride ordering (per-key)

2.  Partition key: ride_id

Reason: Each ride is independent, but within a ride lifecycle, events
must be processed in order.

------------------------------------------------------------------------

## Key Learnings from Section 5

1.  Always map system → ordering type early
2.  Identify the entity that requires ordering → that becomes your
    partition key
3.  Most real-world systems use per-key ordering
4.  Global ordering is rare and expensive, used only when correctness
    strictly depends on sequence
5.  No-ordering systems scale the best
6.  Real-world systems often mix multiple ordering models across
    sub-systems
