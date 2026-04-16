1. most decisions have a con...even if the pro is soo good, the con always exists and can be a major con for your usecase, be aware of cons

2. things to think about if you add auxilary stuff to enforce correctness or solve some problems:
i. latency might/will mostly go up
ii. single points of failures might go up!

Eg: directory based sharding solves some problems and evades consistent hashing complexity but adds latency and becomes a single point of failure!


### Every system design problem can be reduced to these:

1. Ordering
none
per key
global
2. Consistency
strong
eventual
causal (often ignored but shows seniority)
3. Delivery semantics
at-most-once
at-least-once
exactly-once
4. Load shape
read-heavy
write-heavy
bursty
steady
5. Latency requirements
batch (seconds/minutes OK)
near real-time
hard real-time (strict SLA)
6. Data characteristics
mutable vs append-only
size (GB vs PB)
hot vs cold
7. Failure tolerance
can drop data?
can retry?
must be durable?
8. Scale dimension
users
data
geography