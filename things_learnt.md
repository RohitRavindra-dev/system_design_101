1. most decisions have a con...even if the pro is soo good, the con always exists and can be a major con for your usecase, be aware of cons

2. things to think about if you add auxilary stuff to enforce correctness or solve some problems:
i. latency might/will mostly go up
ii. single points of failures might go up!

Eg: directory based sharding solves some problems and evades consistent hashing complexity but adds latency and becomes a single point of failure!