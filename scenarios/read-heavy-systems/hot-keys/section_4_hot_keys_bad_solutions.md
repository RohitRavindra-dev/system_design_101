# Section 4 — Bad / Tempting Solutions (Interview Traps)

These are not stupid ideas — they sound reasonable, which is why interviewers use them to probe depth.

---

## Bad Solution 1: “Just shard more / increase partitions”

### Why it’s tempting:
- Horizontal scaling is the default instinct
- Works for most distributed systems problems

### Why it fails here:
hash(key) → same shard

Hot key:
- Always maps to same shard
- No distribution

You just made:
- 1 hot shard
- N idle shards

### If you try to force it:
- Artificially split the key:
  post:123 → post:123:1, post:123:2...
- Reads become complex
- Writes must update all splits
- You’ve reinvented replication (badly)

### Interview answer:
“Sharding doesn’t help for hot keys because the same key hashes to the same shard, so it remains a bottleneck”

---

## Bad Solution 2: “Scale the cache node vertically”

### Why it’s tempting:
- Easy
- Immediate relief

### Why it’s weak:
- Not scalable
- Hardware limits exist
- Cost explodes quickly

### Failure mode:
- CPU max
- Network bandwidth max
- Single point of failure

### If forced:
- You delay the problem, don’t solve it

### Interview answer:
“Vertical scaling is a short-term mitigation, but doesn’t solve the fundamental hotspot problem”

---

## Bad Solution 3: “Increase cache TTL aggressively”

### Why it’s tempting:
- Fewer cache misses → fewer DB hits

### Why it’s incomplete:
- Doesn’t solve hot node saturation
- Only reduces stampede frequency

### Hidden tradeoff:
- Stale data increases
- Not acceptable for dynamic content

### Interview answer:
“TTL tuning helps with stampede but doesn’t distribute load across nodes”

---

## Bad Solution 4: “Rate limit requests for hot key”

### Why it’s tempting:
- Protects system
- Simple to implement

### Why it’s dangerous:
- Drops valid user requests
- Bad UX for important content

### When it’s acceptable:
- Extreme overload protection (last resort)

### Interview answer:
“Rate limiting protects the system but sacrifices correctness and UX, so it’s not a primary solution”

---

## Bad Solution 5: “Rely only on DB read replicas”

### Why it’s tempting:
- Classic scaling approach
- Works for moderate read load

### Why it fails:
- DB replicas are slower than cache
- Expensive
- Hot key still repeatedly accessed

### Failure mode:
- Replication lag → stale reads
- DB bottleneck

### Interview answer:
“DB replicas help general read scaling but are too heavy and slow for hot key scenarios compared to cache-level solutions”

---

## Bad Solution 6: “Consistent hashing will balance load”

### Why it’s tempting:
- It balances keys evenly

### Why it fails:
- Balances number of keys, not traffic per key

Core flaw:
1 hot key ≠ 1 normal key

### Interview answer:
“Consistent hashing assumes uniform access, which breaks under skewed workloads”

---

## Explicitly INVALID approaches

- Just add more shards
- Cache solves everything
- Consistent hashing ensures balance

---

## What a strong candidate says

“Sharding and consistent hashing assume uniform access distribution, but hot keys violate that assumption. So we need replication-based strategies instead.”

---

# Optional Follow-ups (Short Answers)

## Q1: What if hot key is also write-heavy?
- Replication becomes harder due to write amplification
- Need techniques like write-through cache, batching, or partitioned writes

## Q2: How to detect hot keys in production?
- Monitor per-key access frequency
- Use Redis hot key detection, logs, or metrics (top-K keys)

## Q3: How to dynamically adapt replication factor?
- Detect hotness threshold
- Increase replicas for hot keys
- Reduce replicas when traffic drops
