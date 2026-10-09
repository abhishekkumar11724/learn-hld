# 1. Requirements and Estimation

**Rule:** never start drawing boxes. Spend the first 3-5 minutes asking questions and doing math. Interviewers score this part.

## Functional requirements (what the system does)

Must have:
1. Given a long URL, return a short, unique URL.
2. Opening the short URL redirects to the original long URL.

Nice to have (ask, then decide with the interviewer):
3. Custom alias (user picks `sho.rt/my-sale`).
4. Expiry (link dies after N days).
5. Click analytics (count, country, referrer).
6. Delete / edit a link (owner only).

## Non-functional requirements (how well it works)

| Requirement | What it means here | Why |
|-------------|--------------------|-----|
| High availability | Redirects must almost never fail | Dead links on a billboard or in an email are unfixable |
| Low latency | Redirect in tens of milliseconds | It is on the critical path of every click |
| Scalability | Millions of new links, billions of clicks | Read-heavy workload |
| Durability | Never lose a mapping | A lost mapping is a permanently broken link |
| Hard to guess | Codes should not be easily enumerable | Stops people scraping every link |

**CAP theorem link:** during a network partition you must choose consistency or availability. Here pick **availability (AP)**. If one replica is a few hundred milliseconds stale, nobody notices. If redirects fail, everybody notices. (Custom aliases are the one exception, see file 3.)

## Out of scope (say it out loud)

"I will not design the full user-account system, billing, or the web UI. I will assume an auth layer exists." Saying this shows you are scoping on purpose.

## Capacity estimation

State assumptions, then derive. The final numbers matter less than showing the steps.

**Assumptions**
- 100 million new URLs per month.
- Read:write ratio = 100:1 (people click far more than they create).
- Each record is about 500 bytes (long URL up to ~2 KB worst case, but average is much smaller).
- Keep data for 5 years.

**Traffic**

| Quantity | Calculation | Result |
|----------|-------------|--------|
| Seconds per month | 30 x 24 x 3600 | about 2.6 million |
| Write QPS (average) | 100M / 2.6M | about 40 per second |
| Read QPS (average) | 40 x 100 | about 4,000 per second |
| Read QPS (peak) | 4,000 x 4 (peak factor 3-5x) | about 16,000 per second |

**Storage**

| Quantity | Calculation | Result |
|----------|-------------|--------|
| Per month | 100M x 500 B | 50 GB |
| Total URLs in 5 years | 100M x 60 months | 6 billion |
| Total storage in 5 years | 6B x 500 B | about 3 TB |

3 TB is tiny. Storage is not the hard part of this problem. Read traffic and unique-code generation are.

**Bandwidth**
- Incoming (writes): 40/s x 500 B = 20 KB/s.
- Outgoing (reads): 4,000/s x 500 B = 2 MB/s.
- Both are negligible.

**Cache memory (80/20 rule: 20% of links cause 80% of traffic)**
- Reads per day: 4,000 x 86,400 = about 345 million.
- Cache the hot 20%: 0.2 x 345M x 500 B = about **35 GB**.
- That fits in a small Redis cluster. One big box could even hold it, but use a cluster for availability.

## How long should the short code be?

With Base62 (a-z, A-Z, 0-9):

| Length | Combinations | Enough for 6 billion URLs? |
|--------|--------------|----------------------------|
| 5 | 62^5 = 916 million | No |
| 6 | 62^6 = 56.8 billion | Yes, about 9x headroom |
| 7 | 62^7 = 3.5 trillion | Yes, about 580x headroom |
| 8 | 62^8 = 218 trillion | Overkill |

**Pick 7.** Six technically fits, but 7 leaves room to grow, and keeps random-collision odds tiny (see file 3). Why not Base64? Its `+` and `/` characters are special inside URLs and need escaping. Base62 avoids that.

## Say this in the interview

"I'm assuming 100M new links a month and a 100:1 read-to-write ratio. That's about 40 writes and 4,000 reads per second on average, maybe 16K reads at peak. Over five years that's 6 billion records, roughly 3 TB, so storage is easy. The real challenges are fast reads and generating unique codes without a bottleneck."
