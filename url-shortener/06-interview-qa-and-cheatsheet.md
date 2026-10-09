# 6. Interview Q&A, Script, and Cheat Sheet

## A 2-minute spoken walkthrough

1. **Clarify:** "Core features are create short link and redirect. I'll add custom alias, expiry and basic analytics. Redirect latency and availability matter most, so I'll favour availability over consistency."
2. **Estimate:** "100M new links a month, 100:1 reads. About 40 writes/s and 4K reads/s, 16K at peak. 6 billion records over 5 years, around 3 TB. Seven Base62 characters give 3.5 trillion codes."
3. **API:** "`POST /urls` to create, `GET /{code}` to redirect with a 302."
4. **Data:** "Key-value store, `short_code` as partition key. The access pattern is pure lookup, so NoSQL fits."
5. **Code generation:** "Counter ranges plus Base62 with the ID scrambled, or random codes with a conditional write. KGS if we want zero collision handling."
6. **Architecture:** "Load balancer, stateless app servers, Redis cache-aside in front of the DB. Click events go to Kafka asynchronously."
7. **Deep dives:** "Consistent-hash partitioning with 3 replicas, TTL-based expiry, rate limiting and URL blocklists."
8. **Trade-offs:** "302 over 301 to keep analytics. AP over CP because a stale mapping is harmless but a failed redirect is not."

## Curveball questions

| Question | Answer |
|----------|--------|
| A link goes viral. What breaks? | The hot key overloads one cache shard. Add a short-TTL in-process cache on app servers, replicate the key across cache nodes, and let a CDN cache the redirect briefly. |
| Two servers generate the same code at the same time? | Counter ranges make this impossible (disjoint ranges). With random codes, the DB's conditional write lets only one win; the loser retries. |
| What if the code generator / KGS is down? | App servers hold a buffer of pre-fetched codes, and KGS is replicated. Reads are unaffected. |
| Why not auto-increment in a single SQL DB? | It is a bottleneck and a single point of failure, and IDs are sequential and guessable. |
| Why not use the hash as the primary key directly? | You can, but then collision handling becomes mandatory and every user shortening the same URL gets the same code. |
| How do you stop people guessing codes? | Random codes, or scramble sequential IDs with a reversible permutation, plus rate-limit repeated 404s. |
| Can the same long URL get multiple codes? | Yes by default, so each campaign or user has its own analytics. Dedupe only if saving space matters. |
| How would you support 10x the traffic? | App servers: add more. Cache: add shards. DB: add partitions (consistent hashing moves only about 1/N of keys). Everything is horizontally scalable. |
| How do you handle expired links? | DB TTL plus lazy check on read (410 Gone). Cache TTL never exceeds link expiry. |
| How do you delete or edit a link? | Update/delete in the DB, then invalidate the cache key. With 301 this can't be enforced (browser cached it), which is another reason to use 302. |
| What about an analytics dashboard? | Kafka to a stream processor to aggregated tables (clicks per link per hour). Never query the live URL table for it. |
| How do you go multi-region? | Geo-routing, multi-region replicated table, per-region code ranges, custom aliases routed to a home region. |
| What consistency model, and why? | Eventual consistency (AP). Mappings are write-once, so stale reads are harmless. Strong consistency only for custom alias creation. |
| What would you monitor? | Redirect p50/p95/p99 latency, error rate, cache hit ratio, DB latency, Kafka lag, codes remaining in buffers, creation-rate anomalies. |

## One-page cheat sheet

**Numbers**
- 100M new URLs/month, 100:1 read:write
- Writes: about 40/s. Reads: about 4,000/s average, about 16,000/s peak
- 500 B/record, 50 GB/month, 3 TB over 5 years, 6 billion records
- 62^6 = 56.8B, 62^7 = 3.5T, 62^8 = 218T
- Cache: about 35 GB (20% of daily reads)
- Random-code collision chance at 6B keys: about 0.17%

**Architecture:** LB -> stateless app servers -> Redis (cache-aside) -> NoSQL key-value DB. Kafka for async analytics. Code generator beside the app servers.

**Key choices**
- AP, eventual consistency
- NoSQL, partition key = `short_code`, consistent hashing, 3 replicas
- Base62, 7 chars
- 302 redirect
- Async analytics
- Native TTL + lazy expiry
- Rate limiting + URL blocklist

**Common mistakes to avoid**
1. Jumping to boxes without requirements or numbers.
2. Single SQL auto-increment counter with no mention of the bottleneck.
3. Blocking the redirect on analytics.
4. Saying 301 without discussing that it loses analytics.
5. Forgetting expiry, abuse, and enumeration.
6. Using Snowflake IDs and claiming a 7-character code (they encode to 10-11).
7. Saying "NoSQL because it scales" instead of reasoning from the access pattern.

## Practice checklist

- [ ] Explain Base62 encoding by hand with a small number
- [ ] Derive QPS and storage from the assumptions without notes
- [ ] Draw the full diagram from memory in under 2 minutes
- [ ] Explain all four code-generation options and pick one with reasons
- [ ] Explain 301 vs 302 and consistent hashing in 30 seconds each
- [ ] Walk through 3 failure scenarios
- [ ] Answer five curveball questions aloud
