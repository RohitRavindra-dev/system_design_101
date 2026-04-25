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

### durability vs fault tolerance:

- **durability**: is my writes safe, even if db goes down, redis goes down etc

- **fault tolerance**: will i still be up even if faults occur internally

`variants:`

1. fault tolerance but not durable:
   system up but writes are not being persisted (egs: logs, bad design)

2. durable but not fault tolerant:
   all previous writes are safe, but if some component dies the whole system goes down.

### availability vs fault tolerance:

- availablility: a metric/something you measure n99 eetc
- fault tolerance: a design property

`variants:`

1. fault tolerant but low availability: system goes down but comes back in 1 min, it is fault tolerant since it does come back fast (failover is taking minimal time)

2. available but not strong fault tolerance: system doesnt fail easily, but when it does its ggs for a long time!

in practice availability is strongly coupled to fault tolerance.

(fault tolerance: bullt proof car with airless tires
availbility: car needs special fuel which isnt available so its not drivable sometimes.)

---

- Throughput → how many requests you finish per second
- Latency → how long one request takes
- Capacity → the maximum throughput your system can sustain

Simple analogy

Think of a restaurant kitchen:

Throughput = meals served per hour
Capacity = max meals kitchen can cook

Now:

- 50 orders/hour → smooth
- 100 orders/hour → maxed out
- 200 orders/hour →
  - orders pile up
  - customers wait longer
  - some leave (timeouts)

The kitchen doesn’t magically cook faster — it just gets overwhelmed.

---

Does low throughput = system failure?

Not necessarily.

Two scenarios:

1. Controlled system (good design)
   Rate limiting
   Backpressure
   Queueing

👉 Excess load is shed gracefully

Example:

You get HTTP 429 (Too Many Requests) 2. Uncontrolled system (bad design)
No limits
Everything tries to execute

👉 Result:

Cascading failures
DB crashes or becomes unresponsive

---
