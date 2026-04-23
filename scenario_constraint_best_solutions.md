# Scenario + Constraint: Best Solutions Cheat Sheet

This is a consolidated gist of the best solution(s) from each generated scenario/constraint under `scenarios/`.

## Read-heavy systems

- **Constraint: strict read-after-write**
  - **Best solution:** consistency-aware routing (read-your-writes routing): route users who just wrote to primary; others to cache/replicas.
  - **Also good:** synchronous replication or quorum (`R + W > N`) when stronger guarantees are mandatory.
  - **Why:** gives immediate correctness only where needed while preserving read scalability and low latency for most traffic.

- **Constraint: consistency (eventual/causal/strong tradeoff)**
  - **Best solution:** cache + async replicas + consistency-aware read routing.
  - **Also good:** CQRS/precomputed read models when read complexity and scale are very high.
  - **Why:** cache+replicas is the practical default for read-heavy systems; CQRS shifts compute to write/stream side for very fast reads at scale.

- **Constraint: hot keys / skew**
  - **Best solution:** replicate hot keys and spread reads across replicas.
  - **Also good:** request coalescing/single-flight for bursty misses and stampede prevention.
  - **Why:** replication handles sustained skew load; coalescing removes duplicate concurrent backend work.

- **Constraint: cache invalidation complexity**
  - **Best solution:** cache-aside (read-through pattern at app layer) + explicit invalidation on writes.
  - **Also good:** write-through or write-around with versioning for critical data paths.
  - **Why:** cache-aside is simplest and most common; versioned write-through patterns reduce stale overwrite races when correctness requirements increase.

## Write-heavy systems

- **Constraint: write-latency sensitive**
  - **Best solution:** WAL/commit-log-first ingestion (append, ack fast, process asynchronously).
  - **Also good:** in-memory primary + replicated quorum ack path when you need even lower latency with stronger immediate durability via replicas.
  - **Why:** sequential append gives fast durable writes and high throughput; async downstream processing decouples ingestion from heavy work.

- **Constraint: high durability / no data loss**
  - **Best solution:** replicated commit log with strict ack rules (Kafka-style with ISR/min-insync-replicas).
  - **Also good:** quorum/consensus databases (Raft/Paxos) for stronger consistency semantics.
  - **Why:** acknowledged writes are persisted across replicas before success is returned, minimizing acknowledged-data-loss risk.

## Producer-consumer (queue systems)

- **Constraint: delivery semantics (at-least-once vs exactly-once effect)**
  - **Best solution:** at-least-once delivery + idempotent consumers.
  - **Also good:** exactly-once effect via transactional/idempotency patterns for critical flows (payments, inventory).
  - **Why:** duplicates are inevitable in distributed retries; idempotency is the most practical correctness mechanism.

- **Constraint: ordering**
  - **Best solution:** partitioned queue with per-key ordering (hash key to partition; single consumer per partition ordering lane).
  - **Also good:** single global partition/log only when strict global order is absolutely required.
  - **Why:** per-key ordering preserves correctness while still enabling horizontal parallelism.

- **Constraint: retries and failures**
  - **Best solution:** combine queue-level retry (visibility timeout + ACK model) with idempotent consumer logic.
  - **Also good:** dead-letter queues for poison messages and retry storm containment.
  - **Why:** retry guarantees redelivery, but only idempotency guarantees business correctness under duplicates.

## Distributed shared state

- **Constraint: strong consistency**
  - **Best solution:** leader-based replication with quorum commits.
  - **Also good:** full consensus protocols (Raft/Paxos), typically per shard (shard first, then replicate per shard).
  - **Why:** single ordering authority plus quorum overlap prevents conflicting committed states and ensures correctness under failures.

## Real-time systems

- **Constraint: hard latency SLA**
  - **Best solution:** precompute asynchronously and serve from in-memory hot path.
  - **Also good:** single-hop, co-located compute+state architecture when inline compute is unavoidable.
  - **Why:** removing heavy work and dependency fanout from request path is the most reliable way to keep p99/p999 latency bounded.
