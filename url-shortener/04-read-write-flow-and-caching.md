# 4. Read/Write Flow, Caching, and 301 vs 302

## Write path: creating a short link

```
Client -> LB -> App server
                  1. validate URL (scheme, length, blocklist)
                  2. rate-limit check (per user / IP)
                  3a. custom alias?  -> conditional put; 409 if taken
                  3b. otherwise get a code (counter / random / KGS)
                  4. write {code, long_url, user, created, expires} to DB
                  5. (optional) pre-warm the cache
                  6. return https://sho.rt/<code>
```

Write volume is tiny (about 40/s), so this path does not need heroic optimisation. Correctness matters more: unique codes, valid URLs, no abuse.

## Read path: following a short link (the hot path)

```
Browser -> LB -> App server
                   1. parse short_code from path
                   2. look in Redis
                        hit  -> go to 5
                        miss -> 3
                   3. read DB by partition key (O(1))
                        not found -> 404
                        expired   -> 410
                   4. put in Redis (TTL <= time until expiry)
                   5. respond 302 with Location: long_url
                   6. fire click event to Kafka (async, don't wait)
```

**Golden rule:** step 6 must never block step 5. If Kafka is down or slow, the redirect still goes out. Analytics is best-effort.

With the 80/20 rule and a good cache, most reads end at step 2. The DB only sees misses, so DB load is a small fraction of 4,000 to 16,000 QPS.

## Caching in detail

**Pattern: cache-aside.** The app owns the logic: check cache, on a miss read DB and fill cache. Simple and robust (if Redis dies, the app still works from the DB).

| Setting | Choice | Reason |
|---------|--------|--------|
| Store | Redis cluster, sharded by `short_code` hash | Fast, supports TTL, easy to scale |
| Eviction | LRU | Keeps the popular links, drops the cold ones |
| TTL | min(a few hours, time until link expiry) | Never serve an expired link from cache |
| Size | about 35 GB (from file 1) | Hot 20% of daily reads |
| Replication | Replica per shard | A node failure should not empty a shard |

Links are almost never updated, so **invalidation is easy**: only on delete or edit, remove the key from the cache. This is a big reason the problem is friendly to caching.

### Three cache problems and their fixes

| Problem | What happens | Fix |
|---------|--------------|-----|
| **Hot key** | A viral link hammers one Redis shard | Tiny in-process cache on each app server (a few seconds TTL), replicate the hot key across nodes |
| **Cache stampede** | A popular entry expires and thousands of requests hit the DB at once | Request coalescing (only one request fetches, others wait), add random jitter to TTLs |
| **Cache penetration** | Requests for codes that don't exist always miss the cache and hit the DB (also an attack vector) | Cache "not found" results for a short time (negative caching), or use a Bloom filter of valid codes |

## 301 vs 302 (favourite trick question)

| | 301 Moved Permanently | 302 Found (temporary) |
|--|----------------------|----------------------|
| Browser behaviour | Caches the redirect, never asks you again | Asks you every click |
| Server load | Lower | Higher |
| Analytics | Lost after the first click per browser | Every click is seen |
| Expiry / editing | Can't enforce, browser has the old target | Fully enforceable |
| Search engines | Pass link equity to the target | Don't |

**Answer:** use **302** when you need analytics, expiry, or the ability to change or delete a link, which is almost always the case for a real shortener. Use 301 only if the mapping is truly permanent and you want to shed load.

Related details:
- 307/308 are the "method-preserving" versions of 302/301. Irrelevant for GET redirects, but nice to know.
- A CDN can cache a 302 if you add `Cache-Control: max-age=...`. This offloads huge traffic for viral links, but those clicks won't reach you, so analytics must come from edge logs. A short max-age (say 60 s) is a common compromise.

## Say this in the interview

"On a redirect I check Redis first, fall back to the DB by partition key on a miss, and fill the cache. I respond with a 302 so I keep analytics and expiry control. The click event goes to Kafka asynchronously so analytics can never slow down or break the redirect."
