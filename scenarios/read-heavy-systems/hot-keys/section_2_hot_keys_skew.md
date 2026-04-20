# Section 2 --- Constraint: Hot Keys / Skew

## What is this constraint?

In a read-heavy system:

Not all data is accessed uniformly.

Instead:

A small subset of keys receives a disproportionate amount of traffic.

This is called: - Hot keys - Access skew - Zipfian distribution

------------------------------------------------------------------------

## Concrete intuition

Instead of: All keys → equal traffic

Reality looks like: Top 1% keys → 50--90% traffic\
Remaining 99% → minimal traffic

------------------------------------------------------------------------

## Examples (real-world mapping)

-   Celebrity tweet → millions of reads
-   Viral Instagram reel → very high QPS on one object
-   Trending product → 100x normal traffic
-   IPL score page during final overs → massive spike on one key
-   Breaking news article → sudden hotspot

------------------------------------------------------------------------

## What exactly is a "key"?

Depends on system:

-   post_id → fetch post
-   video_id → fetch metadata
-   product_id → fetch product page
-   user_id → fetch profile
-   cache_key → Redis entry

Hot key = one of these gets hammered.

------------------------------------------------------------------------

## Why does this constraint exist? (Root cause)

Two forces:

### 1. Human behavior is not uniform

-   People converge on the same content
-   Trends, virality, herd behavior

### 2. System amplification

-   Recommendations push same content to millions
-   Ranking algorithms concentrate traffic

The system itself creates hot keys.

------------------------------------------------------------------------

## What does this imply for the system?

### 1. Cache alone is NOT enough

Hot key still hits one cache node → bottleneck.

------------------------------------------------------------------------

### 2. Single-node saturation happens early

Even if Redis handles \~100k QPS, Hot key at 1M QPS → node overload.

------------------------------------------------------------------------

### 3. Load is NOT horizontally distributed

hash(key) → node\
Same key → same shard → no distribution.

------------------------------------------------------------------------

### 4. Tail latency explodes

Queueing delays, CPU saturation → p99 latency spikes.

------------------------------------------------------------------------

### 5. Cache stampede risk increases

Hot key expiry → many requests hit DB simultaneously.

------------------------------------------------------------------------

## Failure modes

### 1. Cache node hotspot

Single node overloaded, others idle.

### 2. Thundering herd

Cache miss → burst traffic to DB.

### 3. DB fallback overload

Cache failure → DB cannot handle concentrated load.

### 4. Uneven scaling

Adding nodes does not help.

------------------------------------------------------------------------

## Why this constraint is tricky

  Assumption                  Broken by hot keys
  --------------------------- --------------------
  Add nodes → scale           No
  Cache solves reads          Not fully
  Sharding distributes load   Not for hot keys

------------------------------------------------------------------------

## Core insight

Hot keys convert a horizontally scalable system into a single-node
bottleneck.

------------------------------------------------------------------------

## Mental model shift

Without skew: Scale by adding machines

With skew: One machine is overloaded while others are idle

------------------------------------------------------------------------

## Interview trap

Question: "We already have Redis cache, is that enough?"

Weak: Yes, cache handles reads\
Strong: No --- hot keys concentrate load on one shard causing hotspots

------------------------------------------------------------------------

# Optional Deep Dive Prompts + Answers

## 1. Why does consistent hashing make hot keys worse?

Because consistent hashing ensures the same key always maps to the same
node.\
This means a hot key cannot benefit from horizontal scaling --- it is
stuck on one node.

------------------------------------------------------------------------

## 2. Can replication solve this?

Partially.

If you replicate data across multiple nodes and load-balance reads, you
can spread traffic.\
However: - Requires careful routing logic - Risk of stale reads - Still
limited by number of replicas

------------------------------------------------------------------------

## 3. What happens if hot key QPS exceeds network bandwidth?

The node saturates network first: - Increased latency - Packet drops -
Requests fail even before CPU becomes bottleneck

------------------------------------------------------------------------

## 4. Difference between hot key vs hot partition vs hot shard

-   Hot key → single key overloaded
-   Hot partition → range/group of keys overloaded
-   Hot shard → entire node overloaded due to multiple hot keys

Hot key is most extreme and hardest to scale.
