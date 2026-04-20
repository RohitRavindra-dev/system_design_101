# Section 1 — Real-time Systems (Base Scenario)

## What is the scenario?

A real-time system is one where:
The value of the response is tightly coupled to how fast it is returned.

Not “fast is nice to have” — but:
If you’re late → response is useless or harmful

---

## Core intuition

There are 2 kinds of systems:

Offline / async → Analytics, reports → Minutes–hours OK  
Near real-time → Notifications, feeds → Seconds OK  
Real-time → Trading, gaming, bidding → Milliseconds matter  

In real-time systems:
Latency is correctness, not just performance

---

## When does this scenario appear?

1. External world is moving fast  
Markets changing, players moving, auctions progressing  

2. You are competing with others in time  
Ad bidding, HFT trading, multiplayer games  

3. User perception depends on immediacy  
Lag = bad UX, Delay = unfairness  

---

## Examples

Trading systems  
Order placement must happen in microseconds–milliseconds  
Late execution = financial loss  

Multiplayer gaming  
Delayed updates = unfair outcomes  

Real-time bidding  
~100ms to respond or you are dropped  

Ride matching  
Needs to feel instant (~100–300ms)  

---

## Key characteristics

1. Tight feedback loop  
2. Continuous stream, not batch  
3. Rapidly changing state  
4. Always hot system  

---

## Mental model

Driving at 120 km/h → late reaction is dangerous  
vs  
Writing a report → delay is fine  

---

## What not to confuse

Low latency API ≠ Real-time system  
Fast response ≠ Real-time system  

Real-time = deadline beyond which response is useless  

---

## Tail Latency

Tail latency = slowest few % of requests (p95, p99)

Why it happens:
GC pauses, I/O spikes, network jitter, contention, cache misses

Why it matters:
In real-time systems, one slow request can break correctness

Examples:
Trading → bad execution  
Bidding → dropped requests  
Gaming → lag spikes  

Key idea:
Optimize for p99, not average  

Mental model:
Average = speed  
Tail = brake failure  

---

## Hard vs Soft Real-time

Soft real-time:
Missing deadline → degraded quality  
Examples: streaming, notifications  

Hard real-time:
Missing deadline → invalid result  
Examples: trading, bidding, gaming  

Key difference:
Soft → quality drop  
Hard → correctness breaks  

Design implications:
Hard RT prioritizes predictability over throughput  

---

## Optional Prompts (with answers)

Q1. Why is p99 more important than average in bidding?  
Because missing the deadline means the bid is discarded entirely, making those requests useless despite good averages.

Q2. Why are retries often useless in hard real-time systems?  
Because retries increase latency further and violate deadlines; by the time retry succeeds, the result is already stale or irrelevant.
