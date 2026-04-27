# Write-Ahead Log (WAL), Commit, and Durability — Deep Dive Notes

## 1. What is a Write-Ahead Log (WAL)?

A Write-Ahead Log (WAL) is a mechanism where every change to the system is first recorded in an append-only log before being applied to the actual database structures.

This log is:

- Sequential (fast to write)
- Append-only (no in-place updates)
- Durable (once flushed to disk)

### Why it exists

Writing directly to complex data structures (like indexes or pages) is:

- slower (random I/O)
- harder to recover if interrupted

Instead, systems:

1. Record intent in WAL
2. Make that durable
3. Apply changes later

---

## 2. What is `fsync`?

`fsync` is an operating system call that forces buffered writes to be flushed from memory to stable storage (disk).

Without fsync:

- Data may sit in memory (OS page cache)
- A crash can lose it

With fsync:

- Data is physically persisted
- It survives crashes or power failures (assuming reliable hardware)

In WAL terms:

- Write = append to memory buffer
- fsync = guarantee persistence

---

## 3. What does “Commit” mean?

Commit is a logical guarantee, not a physical operation.

### Definition:

A transaction is considered **committed** when the system guarantees that it will not be lost, even in the event of failures.

How this guarantee is achieved depends on system design:

- Single-node system:
  - WAL must be fsynced locally

- Distributed system (synchronous replication):
  - WAL must be fsynced on a quorum of nodes

- Distributed system (asynchronous replication):
  - WAL is fsynced only on the leader

---

## 4. What does “Apply to DB” mean?

Applying means taking the WAL entries and updating:

- tables
- indexes
- in-memory structures

This step:

- happens after commit
- is replayable from WAL
- may lag behind commit

---

## 5. Key Separation

| Concept   | Meaning                             |
| --------- | ----------------------------------- |
| WAL Write | Recording the intent                |
| Commit    | Guarantee of durability             |
| Apply     | Updating actual database structures |

---

## 6. Analogy: Bank Ledger vs Account Display

- WAL = official ledger book
- DB = account balance display

### Flow:

1. Transaction recorded in ledger
2. Ledger is secured (fsync/quorum)
3. Transaction is committed
4. Account balances updated later

If display fails:

- rebuild from ledger

Ledger is the source of truth.

---

## 7. Distributed Systems: Step-by-Step (Synchronous Replication)

### Flow:

1. Leader receives client request
2. Leader writes entry to WAL (append)
3. Leader calls fsync → entry is durable locally
4. Leader sends entry to followers
5. Followers append to their WAL
6. Followers call fsync → durable on each follower
7. Followers send acknowledgment to leader
8. Once a quorum acknowledges → entry is **committed**
9. Leader advances commit index
10. Leader applies entry to database
11. Followers learn commit index and apply entry

### Important:

- Commit happens after quorum WAL writes
- Apply happens after commit
- Apply can lag, but must be ordered

---

## 8. Distributed Systems: Asynchronous Replication

### Flow:

1. Leader receives client request
2. Leader writes entry to WAL
3. Leader calls fsync
4. Leader immediately marks entry as committed
5. Leader applies entry to database
6. Leader responds success to client
7. Replication to followers happens later

### Risk:

If leader crashes before replication:

- Followers may not have the entry
- A new leader may be elected without it
- The committed (to client) data is lost

---

## 9. Why WAL is the Source of Truth

The database state is derived from WAL.

If:

- DB is inconsistent
- node crashes
- partial writes occurred

System:

- replays WAL
- reconstructs state deterministically

WAL is authoritative. Database state is a materialized view.

---

## 10. Read Consistency and Visibility

Even if WAL is committed but not yet applied:

Strongly consistent systems ensure clients do NOT see stale data.

Mechanisms include:

- Serving reads from leader only
- Ensuring apply has caught up before serving reads
- Using commit index checks (e.g., in Raft)
- Read barriers or linearizable reads

If stale reads are possible:

- system is operating under eventual consistency

---

## 11. Is WAL Optional?

In most modern databases, WAL is fundamental and always enabled.

Examples:

- PostgreSQL (WAL)
- MySQL InnoDB (redo log)
- Apache Kafka (log-based storage)

You typically configure:

- fsync frequency (durability vs performance)
- replication mode (sync vs async)

---

## 12. Final Mental Model

Think in layers:

### Layer 1 — WAL (truth)

- append-only
- durable
- replicated

### Layer 2 — Commit (guarantee)

- defines what is safe
- based on fsync / quorum rules

### Layer 3 — Apply (visibility)

- updates actual database state
- derived from WAL

Durability comes from WAL + commit rules, not from when data appears in tables.

---

## 13. Subtle but Important Truth

There is always a window where data can be lost:

- Before WAL append
- Before fsync completes
- Before quorum replication (in distributed systems)

Systems define “commit” as the boundary after which data loss is no longer acceptable.

---

## 14. One-line Summary (Precise)

A system is durable when:

"A committed operation has been recorded in WAL according to the system’s durability rule (local fsync or quorum fsync), such that it can always be recovered, even if not yet applied to the database."
