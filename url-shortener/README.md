# Design a URL Shortener (made easy)

A URL shortener turns a long, ugly link into a short one. Anyone who opens the short link gets sent to the original.

```
https://example.com/blog/2026/10/some-very-long-title?utm_source=x&ref=y
                         |  shorten
                         v
                 https://sho.rt/aZ3k9Qp
```

Think of it like a **coat-check ticket**. You hand over a bulky coat (long URL) and get a tiny ticket (short code). Later, showing the ticket gets the coat back. The whole system is a giant, fast lookup table: `short_code -> long_url`.

## Reading order

| # | File | What you learn |
|---|------|----------------|
| 1 | [01-requirements-and-estimation.md](01-requirements-and-estimation.md) | What to ask first, and the back-of-envelope math |
| 2 | [02-api-and-data-model.md](02-api-and-data-model.md) | The endpoints, the table, SQL vs NoSQL |
| 3 | [03-short-code-generation.md](03-short-code-generation.md) | The heart of the problem: how to make unique short codes |
| 4 | [04-read-write-flow-and-caching.md](04-read-write-flow-and-caching.md) | Step-by-step flows, Redis cache, 301 vs 302 |
| 5 | [05-scaling-reliability-security.md](05-scaling-reliability-security.md) | Sharding, replication, failures, expiry, analytics, abuse |
| 6 | [06-interview-qa-and-cheatsheet.md](06-interview-qa-and-cheatsheet.md) | Curveball questions, a 2-minute script, one-page cheat sheet |

## The whole system on one page

```
                         +-----------------+
  Client --> Load -----> |  App servers    | --hit--> Cache (Redis)
             Balancer    |  (stateless)    | <--miss--+     |
                         +--------+--------+                | miss
                                  |                         v
                  write path      |                  Database (NoSQL)
                                  v                   short_code -> long_url
                     Code generator
                     (counter / KGS / random)

                  redirect path also sends a click event (async)
                                  |
                                  v
                               Kafka --> Analytics store
```

## The 6 big decisions (memorise this table)

| Decision | Pick | One-line reason |
|----------|------|-----------------|
| Consistency vs availability | Availability (AP) | A broken redirect is worse than a slightly stale one |
| Database | NoSQL key-value (DynamoDB / Cassandra) | Pure key lookup, no joins, huge scale |
| Code length / alphabet | 7 chars of Base62 | 62^7 = 3.5 trillion, enough for decades |
| Code generation | Counter ranges + Base62, or random with conditional write | No per-request coordination |
| Redirect status | 302 | Keeps analytics and expiry under your control |
| Analytics | Async via Kafka | Never slow down the redirect |

## Mini glossary

- **Short code / alias**: the tail of the short URL (`aZ3k9Qp`).
- **Base62**: digits + lowercase + uppercase letters (10 + 26 + 26 = 62 symbols).
- **QPS**: queries per second.
- **Hot key**: one key getting a huge share of traffic (a viral link).
- **Cache-aside**: app checks the cache first, and on a miss reads the DB and fills the cache.
- **KGS**: Key Generation Service, a service that pre-creates unique codes.
- **TTL**: time to live, when a cache entry or record expires.
