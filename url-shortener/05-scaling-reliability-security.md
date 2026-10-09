# 5. Scaling, Reliability, Expiry, Analytics, Security

## Scaling the database

### Partitioning (sharding) by short_code

Split the data across many machines by `hash(short_code)`. Every lookup goes to exactly one partition, so reads and writes scale with the number of machines.

**Why consistent hashing, not `hash % N`?**
- With `hash % N`, adding a server changes N, so almost every key moves to a different machine. Huge data reshuffle.
- Consistent hashing puts servers and keys on a ring (like a clock face). A key belongs to the first server clockwise from it. Adding a server only takes over keys from its neighbour, roughly 1/N of the data.
- **Virtual nodes** (each server appears at many ring positions) spread load evenly.
- DynamoDB and Cassandra do this internally. You don't build it, but you must be able to explain it.

### Replication

- Keep 3 copies of each partition in different machines or availability zones.
- Reads can go to any replica (scales reads).
- Writes: either leader-follower, or leaderless with quorums (write to W replicas, read from R, with W + R > N for stronger reads).
- Async replication is acceptable here. Mappings are write-once, so there are no real conflicts. This is the AP choice paying off.

### Stateless app servers

App servers hold no state (just a small buffer of codes at most), so you scale them by adding more behind the load balancer, and auto-scale on CPU or QPS.

## Expiration and cleanup

Three ways, usually combined:
1. **Native TTL** on the `expires_at` column (DynamoDB / Cassandra delete expired rows automatically).
2. **Lazy expiry:** on read, if `expires_at < now`, return 410 Gone and delete in the background. Guarantees expired links are never served, even if cleanup lags.
3. **Background sweeper** if your DB has no TTL: a low-priority job that scans and deletes.

After deletion, can the code be reused? Safer to **not** reuse for a long time (old printed links would suddenly point somewhere else). With 3.5 trillion codes there is no need to recycle.

## Analytics (kept off the hot path)

```
Redirect handler --event--> Kafka --> stream consumer --> aggregates --> analytics DB
(code, time, country, referrer, user-agent)           (counts per link per hour/day)
```

- Fire-and-forget. If it fails, drop or buffer the event, never fail the redirect.
- Aggregate in the consumer (counts per hour) instead of storing every click forever, unless raw data is required.
- Store raw events in cheap storage (object store) if you need to reprocess later.
- Privacy: be careful with IP addresses (GDPR). Hash or truncate them if you don't need the full IP.

## Failure scenarios (what if X dies?)

| Component dies | What happens | How we survive |
|----------------|--------------|----------------|
| One app server | Requests to it fail | Stateless; the load balancer health-checks and routes around it |
| One Redis shard | Its keys miss and go to the DB | Replica takes over; DB has headroom; request coalescing |
| Entire Redis | All reads hit the DB | Slower, DB spike; rate-limit/shed load, rebuild cache gradually. This is why the DB must still be able to survive some load |
| One DB node | One replica gone | Other replicas serve; re-replicate in the background |
| Whole availability zone | A third of capacity gone | Replicas span zones; auto-scaling replaces capacity |
| Code generator / KGS | Cannot make new codes | Local buffer of pre-fetched codes; KGS itself replicated. Reads are unaffected |
| Kafka | Analytics events lost or delayed | Redirects continue; local buffer or accept loss |

Reads (redirects) keep working in nearly every scenario. That's the design goal.

## Multi-region

- Deploy in several regions; geo-DNS or anycast routes users to the closest one.
- Replicate the table across regions (DynamoDB Global Tables / Cassandra multi-DC). Eventual consistency is fine for redirects.
- Code uniqueness across regions: give each region its own code range or prefix (or use random codes with a conditional write), so two regions never mint the same code.
- Custom aliases need a global uniqueness check. Options: route alias creation to one home region, or accept first-write-wins conflict resolution.

## Security and abuse

| Threat | Defence |
|--------|---------|
| Spam: someone creates millions of links | Rate-limit creation per user, IP, and API key (token bucket) |
| Phishing / malware links | Check new URLs against a blocklist / safe-browsing service; re-scan periodically; provide a report-abuse flow and takedown |
| Scraping all links | Unguessable codes (random or scrambled), rate-limit repeated 404s from one client |
| Bad schemes (`javascript:`, `data:`) | Allow only `http` / `https` |
| Redirect loops | Reject URLs that point back to your own domain |
| Alias squatting / route clashes | Reserved-word list; length and character rules |
| Traffic interception | HTTPS everywhere, HSTS |
| DDoS | CDN/WAF in front, per-IP rate limits |

Important wording: short links are "**unlisted**, not private". If a link must be private, it needs authentication, not just an unguessable code.
