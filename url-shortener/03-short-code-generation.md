# 3. Short Code Generation (the heart of the problem)

Expect most of your follow-up questions here. The goal is a **unique, short, hard-to-guess code**, generated **without a bottleneck** and **without two servers ever handing out the same code**.

## First, the Base62 building block

Base62 just writes a number using 62 symbols instead of 10.

```
Alphabet (index 0..61): 0-9, a-z, A-Z
```

Example: encode 1,000,000.

```
1,000,000 / 62 = 16129 remainder 2   -> '2'
   16,129 / 62 =   260 remainder 9   -> '9'
      260 / 62 =     4 remainder 12  -> 'c'
        4 / 62 =     0 remainder 4   -> '4'
Read the remainders bottom-up: "4c92"
```

Code:

```cpp
const string ALPHABET =
    "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

string encode(uint64_t id) {
    if (id == 0) return "0";
    string s;
    while (id > 0) {
        s += ALPHABET[id % 62];
        id /= 62;
    }
    reverse(s.begin(), s.end());
    return s;
}

uint64_t decode(const string& s) {
    uint64_t id = 0;
    for (char c : s) id = id * 62 + ALPHABET.find(c);
    return id;
}
```

Any number below 62^7 (3.5 trillion) encodes to at most 7 characters.

## The four approaches

### Option A: Unique ID, then Base62

Get a globally unique number, Base62-encode it, that is the code.

**Problem:** one database auto-increment counter is a single point of failure and a write bottleneck. Two ways around that:

**A1. Counter ranges (ticket server).**
- A small coordinator (ZooKeeper / etcd / a DB row) hands out blocks: server 1 gets IDs 1,000,000-1,999,999, server 2 gets 2,000,000-2,999,999.
- Each app server hands out IDs from its block locally, with no network call per request.
- When the block runs out, it asks for a new one.
- If a server crashes, the unused part of its block is wasted. That's fine, the space is 3.5 trillion.
- Analogy: a deli counter where each clerk takes a whole roll of numbered tickets instead of asking the manager for each one.

**A2. Snowflake-style IDs.**
- 64-bit ID = timestamp (41 bits) + machine ID (10 bits) + sequence (12 bits).
- Every machine generates IDs alone with no coordination, and IDs are roughly time-ordered.
- **Catch:** a 64-bit number is about 9.2 x 10^18, which Base62-encodes to **10-11 characters**, not 7. Snowflake is great as a general ID, but it does not fit a 7-character target. Use A1 if you want 7.

**Weakness of both:** IDs are sequential, so codes are guessable. Someone can walk `.../4c92`, `.../4c93`, `.../4c94` and scrape every link.

**Fix: scramble the number before encoding.** Apply a reversible permutation to the ID (multiply by a large number coprime to the keyspace modulo 62^7, or run a small Feistel network), then Base62-encode. The output looks random but is still collision-free, because the mapping is one-to-one.

### Option B: Hash the long URL

Take MD5 or SHA-256 of the long URL, Base62-encode the digest, keep the first 7 characters.

| Pros | Cons |
|------|------|
| Stateless, no coordination | Different URLs can produce the same 7 chars (collision) |
| Same URL gives the same code (free dedupe) | Same URL always gives the same code, which is bad if each user wants separate analytics |

On a collision, append a salt (like a counter or the user ID), re-hash, and retry. You detect a collision with a conditional write (see below).

### Option C: Random code plus conditional write

Pick 7 random Base62 characters and insert with **"put only if the key does not exist"** (DynamoDB conditional put, Cassandra `IF NOT EXISTS`, SQL unique constraint). If it fails, pick another.

Is the collision rate scary? Do the math:
- Keyspace = 3.5 trillion. Table at the end of 5 years = 6 billion.
- Chance that a fresh random code is already taken = 6B / 3.5T, about **0.17%**.
- So about 1 insert in 600 needs one retry. That is cheap, and the conditional write is a single DB call (no separate read first).

Honest take: at our scale this simple approach is perfectly fine. It also gives unguessable codes for free.

### Option D: Key Generation Service (KGS)

A separate service pre-generates millions of random unique codes offline and stores them.

```
unused_keys:  [aZ3k9Qp, Pq81xYz, ...]
used_keys:    [...]
```

- When an app server needs codes, KGS gives it a **batch** (say 1,000) and immediately marks them as used. No two servers ever get the same code.
- The app server keeps the batch in memory and hands them out locally. Writes never wait on collision checks.
- If an app server crashes, its leftover batch is lost. Fine, same reasoning as wasted ranges.
- KGS must not be a single point of failure: replicate it, and let app servers hold a buffer so a short KGS outage does not stop writes.

Use it when you want predictable write latency and zero collision handling. The price is one more service to run.

## Comparison

| Approach | Collisions? | Guessable? | Coordination | Code length | Complexity |
|----------|-------------|-----------|--------------|-------------|------------|
| A1 counter ranges + Base62 | None | Yes, unless scrambled | Rare (block fetch) | up to 7 | Low |
| A2 Snowflake + Base62 | None | Partly | None | 10-11 | Low |
| B hash of URL | Yes, handle | No | None | 7 | Medium |
| C random + conditional write | Rare, handle | No | None | 7 | Low |
| D KGS | None | No | Batch fetch | 7 | Highest |

## What I'd say in the room

"Simplest defensible answer: counter ranges plus Base62, with the ID scrambled so codes aren't sequential. If I'd rather avoid any coordination, I'd generate random 7-character codes with a conditional write, since at 6 billion keys only about 0.17% of inserts retry. If the interviewer wants zero write-path collision handling, I'd add a Key Generation Service that hands out batches."

## Custom aliases are different

A user-chosen alias must be unique, so the write needs a **strongly consistent** check: a conditional put that fails with 409 if taken. This is the one place you can't be fully "AP". Keep custom aliases in the same table (same key space) so they can never collide with generated codes. Optionally, give generated codes a fixed length (7) and require custom aliases to be different (for example, 4-6 or 8+ characters) so the two spaces cannot overlap.
