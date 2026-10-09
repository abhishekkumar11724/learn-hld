# HLD Interview Questions

Grouped by difficulty. Sources cross-checked Oct 2026 (Educative, system-design-primer, GeeksforGeeks).

## Easy (good starting point for 2+ YOE)
Scoped, single clear bottleneck, well-known pattern.

- [ ] Design a URL shortener (TinyURL/bit.ly)
- [ ] Design a rate limiter (like Firebase/GitHub's API limits)
- [ ] Design a distributed cache
- [ ] Design a pub/sub system (like Kafka, simplified)
- [ ] Design a web crawler
- [ ] Design a CDN
- [ ] Design an authentication / SSO service (like Auth0)
- [ ] Design a pastebin

## Medium (the bulk of real interviews, 1-2 YOE to 4-5 YOE level)
Multiple interacting components, real scaling trade-offs.

- [ ] Design a chat application (WhatsApp / Messenger)
- [ ] Design a social media feed/timeline (Twitter, Instagram, Facebook)
- [ ] Design a video streaming service (YouTube/Netflix)
- [ ] Design a ride-sharing service (Uber/Lyft)
- [ ] Design a proximity/nearby service (Yelp)
- [ ] Design a typeahead/autocomplete search service
- [ ] Design a file storage/sharing service (Google Drive/Dropbox)
- [ ] Design a discussion forum (Reddit/Quora)
- [ ] Design a notification system
- [ ] Design an e-commerce product/inventory system
- [ ] Design a recommendation engine
- [ ] Design a parking/ticket-booking system at scale (BookMyShow)

## Hard (senior / staff-leaning, or domain-specialized)
Heavy distributed-systems depth, consistency/consensus concerns, or massive scale.

- [ ] Design WhatsApp/ChatGPT-style AI chat service at scale
- [ ] Design a collaborative document editor (Google Docs — needs OT/CRDT)
- [ ] Design Google Maps / navigation system
- [ ] Design a payment gateway (Stripe) — correctness & idempotency critical
- [ ] Design a food-delivery system with real-time logistics (Uber Eats/DoorDash)
- [ ] Design a distributed lock service (Google Chubby)
- [ ] Design a coordination/consensus system (ZooKeeper/etcd)
- [ ] Design a distributed storage system (Bigtable/HDFS-like)
- [ ] Design a code deployment/CI-CD system
- [ ] Design a video conferencing system (Zoom)
- [ ] Design a stock trading / matching engine

## Fundamentals (asked as follow-ups inside any question above)

Know these cold before practicing full designs:

- CAP theorem
- Consistent hashing
- Load balancing strategies
- Sharding vs. partitioning
- Caching strategies & invalidation
- Database replication
- SQL vs NoSQL trade-offs
- Message queues
- Rate limiting algorithms
- Consensus (Raft/Paxos basics)
- Idempotency
- Circuit breaker pattern
- Back-of-envelope capacity estimation

## Suggested approach

1. Start with 2-3 Easy ones to nail the structure of your answer: requirements → estimation → API → high-level diagram → deep dive → trade-offs.
2. Spend most of your time in the Medium bucket — that's what actually shows up most often.
3. Pick Hard ones based on the companies you're targeting (e.g., payments-heavy companies will probe payment gateway / idempotency; infra-heavy companies will probe distributed storage / consensus).
