# SECTION 1 --- Base Scenario: Read-heavy systems

## What is the scenario?

A read-heavy system is one where:

Reads \>\> Writes (often 100:1, 1000:1, or more)

That means: - Most requests are fetching data - Very few requests are
modifying data

------------------------------------------------------------------------

## Concrete examples (anchor this properly)

### 1. Social media feeds

-   Users scroll endlessly
-   Writes (posts, likes) are relatively rare
-   Reads (feed loads, refresh, scroll) are massive

### 2. E-commerce product pages

-   Millions browsing
-   Few actually updating product info

### 3. News / content platforms

-   Articles written once
-   Read millions of times

### 4. Profile lookups (LinkedIn, etc.)

-   Profiles updated occasionally
-   Viewed constantly

------------------------------------------------------------------------

## Why does this scenario naturally happen?

Three structural reasons:

### 1. Human behavior asymmetry

Users consume far more than they create

### 2. Fan-out effect

One write → many reads

Example: - You post once - 10,000 users read it

### 3. System amplification

-   APIs get hit repeatedly
-   Mobile retries, refreshes, polling
-   Same data requested again and again

------------------------------------------------------------------------

## What problem does this create?

If you naively hit DB for every read, you get:

### 1. Database becomes bottleneck

-   Reads dominate CPU, IO
-   Even read replicas start choking

### 2. Latency increases

-   Disk access / network hops
-   P99 gets ugly

### 3. Cost explodes

-   Scaling DB horizontally is expensive
-   You're using DB for something inefficient (repeated reads)

------------------------------------------------------------------------

## Core intuition (this is what interviewers look for)

"Why are we recomputing or refetching the same data repeatedly?"

This leads to introducing caching.

------------------------------------------------------------------------

## Mental model

-   DB = source of truth (slow, expensive)
-   Cache = fast, temporary copy (cheap, fast)

------------------------------------------------------------------------

## Key design shift in read-heavy systems

Instead of:

Client → DB

We move to:

Client → Cache → DB (fallback)

------------------------------------------------------------------------

## What makes this non-trivial?

At first glance: "Just add cache"

But the real problem begins here:

Now you have 2 copies of data: - DB (truth) - Cache (possibly stale)

This introduces consistency problems.

------------------------------------------------------------------------

## Summary

-   Read-heavy = reads dominate writes (100x+)
-   Happens due to consumption patterns + fan-out
-   DB becomes bottleneck → latency + cost issues
-   Solution direction = caching
-   But introduces data duplication → consistency problems

------------------------------------------------------------------------

# Optional Prompts --- Short Answers

## 1. What is cache hit ratio and why does it matter?

Cache hit ratio = (number of cache hits) / (total requests)

-   High hit ratio → most requests served from cache → low latency, low
    DB load
-   Low hit ratio → cache is ineffective → DB still gets hammered

------------------------------------------------------------------------

## 2. Types of caching?

-   Client-side (browser cache)
-   CDN (edge caching)
-   Application-level cache (Redis, Memcached)
-   Database-level cache (buffer pools)

Each layer reduces load closer to the user.

------------------------------------------------------------------------

## 3. Read-through vs write-through?

-   Read-through: cache fetches from DB on miss automatically
-   Write-through: writes go to cache and DB together

Read-through is common for read-heavy systems.

------------------------------------------------------------------------

## 4. When is caching NOT helpful?

-   Highly dynamic data (changes every second)
-   Low read frequency
-   Strong consistency requirements (no stale reads allowed)
