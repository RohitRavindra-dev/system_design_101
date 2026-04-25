# System Design Cheat Sheet

## 1. Throughput (handle more work/sec)

- Horizontal scaling (stateless services + load balancer)
- Caching (CDN, Redis)
- Batching (process many items together)
- Async processing (queues like Kafka)
- Partitioning/sharding (split load across nodes)
- Connection pooling

## 2. Latency (make individual requests fast)

- Caching (especially near user, CDN)
- Reduce network hops (co-locate services)
- Indexing in DB
- Denormalization / precomputed views
- Use async only where needed (avoid unnecessary hops)
- Efficient protocols (gRPC vs HTTP/1 when needed)

## 3. Availability / Fault Tolerance

- Replication (multi-AZ / multi-region)
- Failover (active-active or active-passive)
- Health checks + auto-restart
- Load balancing
- Circuit breakers
- Graceful degradation (serve partial results)

## 4. Consistency

- Strong consistency:
  - Leader-based writes
  - Consensus (Raft/Paxos)
- Transactions (ACID)
- Distributed locks (careful usage)
- Versioning / optimistic concurrency control
- Eventual consistency:
  - Async replication
  - Conflict resolution

## 5. Ordering (event/message order guarantees)

- Partitioning with keys (same user → same partition)
- Single-writer principle per key
- Sequence numbers / timestamps
- FIFO queues
- Idempotent consumers (to handle reordering safely)

## 6. Reliability (correctness over time)

- Retries (with backoff, not blind retries)
- Idempotency keys (critical for payments, APIs)
- Exactly-once effect (via idempotency + dedup)
- Inbox/Outbox pattern
- Saga pattern (for distributed transactions)
- Validation + invariants (don’t accept bad state)
- Monitoring + alerting

## 7. Durability (data not lost)

- Write-Ahead Logging (WAL)
- fsync / synchronous disk writes
- Replication (sync or async depending on guarantees)
- Backups + restore strategy
- Multi-region storage (for critical data)

## 8. Scalability

- Stateless services (easy horizontal scaling)
- Sharding (DB, cache)
- Service decomposition (microservices when needed)
- Auto-scaling policies
- Avoid global locks / bottlenecks

## 9. Elasticity (auto adapt to load)

- Auto-scaling groups (CPU/QPS based)
- Queue-based buffering (absorbs spikes)
- Serverless / on-demand compute
- Load prediction (advanced setups)

## 10. Backpressure / Load Shedding

- Rate limiting (token bucket, leaky bucket)
- Bounded queues (don’t let queues grow forever)
- Reject early (HTTP 429)
- Priority queues (drop low-priority work first)
- Circuit breakers (stop calling failing services)

## 11. Cost Efficiency

- Caching (reduces expensive DB hits)
- Tiered storage (hot vs cold data)
- Spot/preemptible instances
- Right-sizing infra
- Async/batch processing instead of real-time when possible

## 12. Security

- Authentication (OAuth, JWT)
- Authorization (RBAC, ABAC)
- Encryption (TLS, at rest)
- Rate limiting (also prevents abuse)
- Auditing/logging

## 13. Observability

- Metrics (latency, throughput, error rate)
- Logging (structured logs)
- Distributed tracing (request flow across services)
- Dashboards + alerts

## 14. Maintainability / Operability

- CI/CD pipelines
- Feature flags
- Blue-green / canary deployments
- Rollbacks
- Clean abstractions (don’t overcomplicate)

## 15. Data Freshness

- TTLs on cache
- Cache invalidation strategies
- Streaming (real-time updates)
- Polling vs push
- Event-driven architecture
