# Section 4 — Bad Solutions / Interview Traps

## ❌ 1. “Let’s add a queue (Kafka / SQS) to buffer requests”

### Why it’s tempting
- Decouples producers and consumers  
- Handles spikes  
- Industry standard pattern  

### Why it breaks here
A queue introduces unbounded waiting time. Under load, queue grows → latency increases unpredictably.

### What breaks first
- Queue depth increases  
- p99 latency explodes  

### If you try to fix it
- More consumers → improves throughput, not latency guarantees  
- Priority queues → still waiting  
- Dropping messages → equivalent to fail-fast  

### Interview answer
Queues introduce unbounded delay → violates hard latency SLAs.

---

## ❌ 2. “We’ll retry on failure/timeouts”

### Why it’s tempting
- Improves reliability  

### Why it breaks here
Retry adds extra time → response already too late.

### What breaks first
- Retry storms  
- Amplified load → worse tail latency  

### Fix attempts
- Faster retry → still too late  
- Parallel retry → increases load  

### Interview answer
Retries are ineffective because success after retry still violates SLA.

---

## ❌ 3. “We’ll call multiple services (fanout) and aggregate”

### Why it’s tempting
- Clean microservices design  

### Why it breaks here
Latency becomes max of all downstream services → tail latency worsens.

### What breaks first
- One slow dependency ruins request  

### Fix attempts
- Parallel calls → still bound by slowest  
- Timeouts → incomplete responses  

### Interview answer
Fanout increases tail latency and dependency risk.

---

## ❌ 4. “Fallback to DB on cache miss”

### Why it’s tempting
- Ensures correctness  

### Why it breaks here
DB latency unpredictable → violates SLA.

### What breaks first
- Cache misses  
- Cold starts  

### Fix attempts
- Bigger cache → reduces misses, not eliminates  

### Interview answer
Never fallback synchronously; return default or drop.

---

## ❌ 5. “Scale horizontally / autoscale”

### Why it’s tempting
- Handles load  

### Why it breaks here
Improves throughput, not latency determinism.

### What breaks first
- Load imbalance  
- Cold starts  

### Interview answer
Scaling does not guarantee bounded latency.

---

## ❌ 6. “Batch requests”

### Why it’s tempting
- Improves efficiency  

### Why it breaks here
Batching introduces delay.

### Interview answer
Batching trades latency for throughput → opposite goal.

---

## ❌ 7. “Strong consistency DB in hot path”

### Why it’s tempting
- Correctness  

### Why it breaks here
Coordination, locks, replication → latency spikes.

### Interview answer
Strong consistency adds unpredictable delay.

---

# Meta Pattern
All bad solutions introduce:
- Waiting
- Extra work
- Dependencies
- Coordination

---

# Optional Deep Dive Answers

## Circuit breakers
They prevent cascading failures but don’t eliminate latency spikes from slow dependencies.

## Tail latency amplification
Multiple services multiply p99 latency due to independent delays.

## Coordinated omission
Latency measurements can be misleading if slow requests are under-sampled.

