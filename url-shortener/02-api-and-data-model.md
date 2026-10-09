# 2. API and Data Model

## API design

### Create a short link

```
POST /api/v1/urls
Headers: Authorization: Bearer <token>   (or an API key)
Body:
{
  "longUrl": "https://example.com/very/long/path?x=1",
  "customAlias": "my-sale",          // optional
  "expiresAt": "2027-01-01T00:00:00Z" // optional
}

201 Created
{ "shortUrl": "https://sho.rt/aZ3k9Qp", "expiresAt": "..." }
```

### Redirect (the hot path)

```
GET /{shortCode}

302 Found
Location: https://example.com/very/long/path?x=1
```

### Optional endpoints

```
DELETE /api/v1/urls/{shortCode}     owner only
GET    /api/v1/urls/{shortCode}/stats   click analytics
```

### Status codes you should know

| Code | When |
|------|------|
| 201 Created | Short link made |
| 302 Found | Redirect (see file 4 for 301 vs 302) |
| 400 Bad Request | Invalid URL, bad alias characters |
| 401 / 403 | Not logged in, or not the owner |
| 404 Not Found | Code never existed |
| 409 Conflict | Custom alias already taken |
| 410 Gone | Link existed but has expired or been deleted |
| 429 Too Many Requests | Rate limit hit |

### Small details that impress

- **Idempotency:** if a client retries a create after a timeout, they should not get two different codes. Accept an optional `Idempotency-Key` header, or dedupe on (user, longUrl).
- **Same long URL submitted twice:** two valid choices.
  - Return the same code (needs an index on a hash of `long_url`; saves space).
  - Always make a new code (simpler; lets each user or campaign have separate analytics). Most real products do this one.
- **Validation:** only allow `http` and `https` schemes (reject `javascript:`, `data:`, `file:`), cap length (about 2,048 chars), reject URLs that point back to your own domain (redirect loops).
- **Custom alias rules:** 3-30 chars, `[a-zA-Z0-9-_]`, and a reserved-word list (`api`, `admin`, `login`, `static`...) so aliases cannot collide with real routes.

## Data model

One main table is enough.

```
urls
-----------------------------------------------
short_code     STRING   PRIMARY KEY (partition key)
long_url       STRING
user_id        STRING   nullable (who created it)
created_at     TIMESTAMP
expires_at     TIMESTAMP nullable
```

Notes:
- The lookup is always by `short_code`, so make it the partition key. That gives O(1) reads.
- Use the database's native TTL on `expires_at` where available (DynamoDB, Cassandra) so expired rows get cleaned automatically.
- Optional second table `users(user_id, email, ...)`, and `clicks` lives in the analytics store, not here.
- If you need "list all links of a user", add a secondary index on `user_id`. It is a rare query, so it is fine to be slower.

## SQL or NoSQL?

Don't say "NoSQL because it scales". Reason from the access pattern.

| Question | Answer for this system |
|----------|------------------------|
| What are the queries? | Only `get by short_code` and `put` |
| Any joins or transactions? | No |
| How much data / traffic? | Billions of rows, tens of thousands of reads per second |
| Consistency needed? | Eventual is fine |
| Does the schema change? | Barely |

This is a textbook **key-value** workload, which fits NoSQL (DynamoDB, Cassandra, Riak) naturally: they partition horizontally by key and favour availability.

**Sharded MySQL/Postgres is still valid.** At moderate scale it works fine and is easier to operate. Say so, then explain that you'd choose NoSQL because at billions of rows you would otherwise be hand-building the sharding and replication that NoSQL gives you for free.

## Say this in the interview

"The access pattern is a pure key lookup with no joins, so I'd use a key-value store like DynamoDB with `short_code` as the partition key. It scales horizontally and favours availability, which matches our CAP choice. A sharded relational DB would also work, but I'd rather not build sharding myself."
