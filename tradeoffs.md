# System Design Trade-offs

## 1. Throughput vs Latency

- Increase throughput → batching, queuing, async processing
- Trade-off → latency increases

**Examples:**

- Kafka → high throughput, higher latency
- Trading systems → low latency, lower throughput

---

## 2. Availability vs Consistency (CAP Theorem)

- You can’t have strong consistency and availability during network partitions

**Examples:**

- Banking systems → favor consistency over availability
- Social media → favor availability over consistency

---

## 3. Throughput vs Consistency

- Strong consistency → requires coordination (locks, consensus)
- Coordination → slows system → reduces throughput

**Example:**

- Distributed transactions severely reduce throughput

---

## 4. Latency vs Consistency

- Strong consistency → requires waiting for replicas
- Waiting → increases latency

---

## 5. Durability vs Latency

- Synchronous disk writes (`fsync`) → durable but slow
- Asynchronous writes → fast but risk data loss

---

## 6. Scalability vs Consistency

- Scaling → requires distribution
- Distribution → introduces coordination problems
- Coordination → leads to consistency trade-offs

---

## 7. Availability vs Cost

- Multi-region replication → improves availability
- Trade-off → higher cost ($$$)

---

## 8. Fault Tolerance vs Complexity

- More redundancy → more moving parts
- More moving parts → harder to reason about and debug

---

## Mental Model to Internalize

Every system operates along these axes:

- **Speed** → latency
- **Volume** → throughput
- **Correctness** → consistency
- **Survival** → availability, fault tolerance, durability
- **Cost**

> You cannot maximize all of them simultaneously.
