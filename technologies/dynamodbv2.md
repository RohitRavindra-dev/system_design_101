# 📘 DynamoDB Deep Dive — Detailed Notes

---

## 1. Mental Model First (What DynamoDB REALLY is)

DynamoDB is not just “NoSQL”.

Think of it as:

> A distributed key-value store optimized for predictable, high-scale access patterns

### Key Implications

- You don’t query arbitrarily
- You design your data model around access patterns
- Performance is predictable because:
  - You always hit a known partition

---

## 2. Data Model (Core Foundation)

### 2.1 Table, Item, Attributes

- **Table** → collection of data
- **Item** → one record
- **Attributes** → fields

👉 Schema is flexible:

- Items in the same table can have different attributes

---

### 2.2 Primary Key (CRITICAL)

#### (A) Partition Key

- Single key
- Determines:
  - Where data is stored
  - Which machine handles the request

👉 Analogy:

- Like hashing a key into a distributed hashmap

---

#### (B) Composite Key (Partition + Sort Key)

- **Partition key** → grouping
- **Sort key** → ordering within group

**Example:**

```
UserID = 123
OrderID = 1, 2, 3...
```

This enables:

- Storing related data together
- Efficient querying within a group

---

### 🔑 Key Insight

> The partition key is your scaling strategy

If designed poorly:

- Hot partitions occur
- System gets throttled

---

## 3. Indexing (How You Query Differently)

### 3.1 Why Indexes Exist

You can ONLY query efficiently by key.

If your query doesn’t match your primary key → you need an index.

---

### 3.2 Global Secondary Index (GSI)

- Different partition key
- Enables new query patterns

**Example:**

Main table:

```
PK: userId
SK: orderId
```

GSI:

```
PK: orderStatus
```

Now you can:

- Fetch all "pending" orders

---

### 3.3 Local Secondary Index (LSI)

- Same partition key
- Different sort key

Used for:

- Alternate sorting within the same group

---

### 🔑 Key Insight

> Every index is essentially a copy of your data organized differently

**Tradeoff:**

- More indexes → higher write cost

---

## 4. How to Use DynamoDB (Design Thinking Shift)

### 4.1 SQL Mindset (Incorrect Here)

- Normalize data
- Use joins
- Query flexibly later

---

### 4.2 DynamoDB Mindset (Correct)

Start with access patterns first.

Ask:

- What queries will I run?
- What data do I need together?

Then:

- Design keys accordingly

---

### Example

If you need:

- Get user profile
- Get user orders

Design:

```
PK = userId
SK = orderId / metadata
```

---

### 🔑 Big Idea

> You denormalize intentionally

- Duplicate data if needed
- Optimize reads over storage efficiency

---

## 5. Architecture (Under the Hood)

### 5.1 Partitioning

- Data is split into partitions
- Each partition:
  - Handles part of traffic
  - Has limits

Partition key → hashed → determines location

---

### 5.2 Replication

Each partition:

- Stored across multiple nodes

Ensures:

- High availability
- Fault tolerance

---

### 5.3 Request Flow (Write Path)

1. Request comes in
2. Partition key is hashed
3. Routed to correct partition
4. Written to leader node
5. Replicated to other nodes

---

### 🔑 Insight

> No query planner, no joins — just direct key lookup

Result:

- Predictable latency
- Linear scalability

---

## 6. Advanced Concepts

### 6.1 Hot Partitions

**Problem:**

- Too many requests hit the same partition

**Cause:**

- Low-cardinality partition key

**Example:**

```
PK = "USA"   ❌
```

**Fix:**

```
PK = userId  ✅
```

---

### 6.2 Throughput & Scaling

- DynamoDB scales automatically
- But each partition has limits

If exceeded:

- Requests get throttled

---

### 6.3 Denormalization

- Store the same data in multiple places

**Why:**

- Avoid joins
- Enable fast reads

---

### 6.4 Single Table Design

Instead of:

- Users table
- Orders table

Use:

- One table with mixed item types

Distinguish using:

- PK + SK patterns

---

### 🔑 Insight

> Table is not your boundary — access pattern is

---

## 7. When to Use DynamoDB

### Good Fit

- High scale systems
- Known access patterns
- Simple key-based queries

**Examples:**

- User sessions
- Shopping carts
- Event logs

---

### Bad Fit

- Ad-hoc queries
- Complex joins
- Analytics-heavy workloads

---

## 8. Tradeoffs

### Pros

- Massive scalability
- Predictable performance
- Fully managed

---

### Cons

- Rigid access patterns
- Hard to redesign later
- Requires upfront thinking

---

## 9. System Design Interview Angle

### What Interviewers Look For

You choose DynamoDB when:

- Scale is huge
- Queries are predictable

You demonstrate:

- Correct partition key design
- Efficient access patterns

You avoid:

- Table scans
- Hot partitions

---

## 10. Final Mental Model (Condensed)

> DynamoDB = Distributed hashmap + sorted buckets

- **Partition key** → which bucket
- **Sort key** → order inside bucket
- **Index** → alternate bucket arrangement

---
