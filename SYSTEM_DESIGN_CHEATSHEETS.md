# System Design Interview Cheat Sheets

---

## 1. Relational vs NoSQL
**Relational:** structured data, clear relationships, ACID/multi-row transactions needed, ad hoc queries/joins → e-commerce orders/customers, banking ledgers.
**NoSQL:** schema unstructured/changes often, massive scale, eventual consistency OK, access pattern known upfront so data can be denormalized for it.
Decision test: do you need joins/multi-row ACID across entities (→ relational), or do you know your query pattern in advance and need to scale past one machine (→ pick a NoSQL type below)?

---

## 2. NoSQL Database Types
Pick by dominant access pattern — each trades relational flexibility (joins, ad hoc queries) for one specific strength.
- **Document (MongoDB, Firestore, CouchDB)**: schema flexible/varies per record (JSON/BSON/XML), nested data fetched in a single read, no joins needed → user profiles, product catalogs
- **Key-Value (Redis, DynamoDB, Memcached)**: O(1) lookup by key, simplest model, fastest → caching, sessions
- **Columnar/OLAP (Redshift, BigQuery, ClickHouse)**: reads only the needed columns across billions of rows instead of whole rows, built for aggregations not single-row lookups → analytics/BI dashboards. Redshift/BigQuery are managed warehouses (zero ops, pay-per-query), ClickHouse can be self-hosted and trades ops burden for the fastest raw query speed.
- **Wide-column (Cassandra, HBase)**: massive write throughput, linear horizontal scaling, tunable consistency → time-series, IoT, stock ticks
- **Graph (Neo4j, Dgraph)**: traversing relationships *is* the query (friends-of-friends, shortest path) — joins this deep would be too expensive in relational → social graphs, recommendations, fraud rings
- **Search (Elasticsearch, Solr, Algolia)**: inverted index built over documents for full-text/relevance-ranked search, not meant as a primary datastore → product search, log search (ELK stack)
- **Time-Series (InfluxDB, Prometheus)**: optimized for timestamp-indexed writes with built-in retention/downsampling and aggregation over time windows → metrics/monitoring dashboards, sensor data

---

## 3. Caching
**Skip if:** DB handles load fine (<50% repeated reads), data changes constantly (e.g. balances), small/low-traffic system.
**Add when:** >80% reads hit same data, DB connections saturated/timing out.
- **In-memory (local, in-process)**: cache lives inside one app instance's RAM. Zero network hop = fastest possible. But invisible to other instances (each server has its own copy, can go stale/inconsistent) and wiped on restart. Use only when you have a single instance, or per-instance data that doesn't need to match across servers → local config cache.
- **Memcached**: separate networked service, so multiple app servers *share* the same cache (unlike local in-memory). Still bare-bones: pure key-value, no persistence, no built-in clustering/replication (client-side sharding across nodes). Use when you need a shared cache across servers but only need simple caching — no data structures, no durability, no pub-sub.
- **Redis**: also shared/distributed, but richer — data structures (lists, sets, sorted sets), persistence options, built-in clustering/replication, pub-sub. Use for sessions, carts, leaderboards, anything needing more than plain get/set or needing durability guarantees.
- **Elasticsearch**: not a cache — full-text search, log aggregation → product search, log dashboards.

---

## 4. Caching Strategies
How reads/writes flow between app, cache, and DB — orthogonal to which backend you picked in §3.
- **Cache-aside (lazy loading)**: app checks cache first; on miss, reads DB then populates cache itself. Pros: simple, resilient (cache outage just falls back to DB), only caches what's actually requested (memory-efficient). Cons: first request always misses, and cache can go stale if DB changes elsewhere → default choice, e.g. product catalog lookups.
- **Read-through**: cache sits in front of the DB and loads on miss itself — app only ever talks to the cache. Pros: simpler app code than cache-aside (loading logic lives in one place), consistent behavior across all callers. Cons: couples you to a caching layer that knows how to load from the DB → managed layers like AWS DAX in front of DynamoDB.
- **Write-through**: every write goes to cache and DB together, in sync, before returning. Pros: cache is never stale, reads right after a write are always fast and correct. Cons: adds write latency since both writes must succeed → data where staleness is unacceptable, e.g. inventory counts.
- **Write-behind (write-back)**: write goes to cache immediately, DB updated asynchronously later. Pros: fastest possible writes, can batch/coalesce DB writes to cut load. Cons: risks data loss if the cache fails before flushing → high-write-volume data tolerant of some loss, e.g. view/like counters.
- **Refresh-ahead**: cache proactively re-fetches hot keys before they expire, based on access pattern. Pros: hot data stays warm, avoids the latency spike (and thundering herd) when popular keys expire. Cons: wastes work refreshing keys that may not be requested again → CDN edge caching of trending content.

---

## 5. Cache Eviction
Which entries to discard once the cache is full.
- **LRU (Least Recently Used)**: evicts the item not accessed longest, needs access-order tracking → default general-purpose choice, Redis `allkeys-lru`.
- **LFU (Least Frequently Used)**: evicts the lowest access count, better than LRU for a stable "hot set" but slower to adapt when popularity shifts → Redis `allkeys-lfu`, CDN caching of evergreen content.
- **FIFO**: evicts the oldest inserted item regardless of usage, cheapest to implement but ignores actual access pattern → simple bounded queues/buffers.
- **TTL/expiration**: entries expire after a fixed time regardless of usage, bounds staleness rather than memory → session tokens, rate-limit counters.
- **Random**: evicts a random entry, O(1) with no bookkeeping, surprisingly competitive at scale → Redis `allkeys-random` under extreme memory pressure.
  Combine in practice: TTL to cap staleness + LRU/LFU to cap memory is a common pairing (e.g. Redis `maxmemory-policy`).

---

## 6. Sync vs Async / Message Queues
**Sync:** fast ops, need instant confirm/error handling, simple → login request/response.
**Async:** slow ops (mins), traffic spikes, need reliability/decoupling → video transcoding job.
- **Job queues** (RabbitMQ/SQS/Celery): discrete one-off tasks, priority support, flexible consumer scaling → sending emails, resizing uploaded images.
- **Pub-Sub**: one event → many independent subscribers, no shared work queue → order-placed event fanning out to billing, inventory, notifications.
- **Kafka**: durable, ordered (per partition key), replayable, high-volume streams; acks: 0=fast/unsafe, 1=leader only, all=safest → clickstream/analytics pipelines, activity feeds.
- Kafka has no native priority — use separate topics/queues or RabbitMQ priority queues.
- RabbitMQ durability: opt-in (durable queues + persistent msgs + quorum queues) — comparable to Kafka if configured.
- Autoscale consumers on queue depth / consumer lag — best signal for traffic-driven scaling.

---

## 7. REST vs gRPC
**REST:** public APIs, many client languages, need caching/human-readable/debuggable.
**gRPC:** internal service-to-service, high performance (protobuf+HTTP/2), streaming, strict typed contracts.
Example: Stripe API=REST; Uber internal dispatch=gRPC.

---

## 8. Do We Need Real-Time Communication?
**Test:** if update arrives 10s late, does anything break?
**Skip:** infrequent data, delay tolerable (dashboards, catalogs, email summaries).
**Need it:** collaborative/chat (feels broken if delayed), safety/money alerts (fraud, live location), live time-sensitive data (stock, sports, auctions).
Note: real-time complicates horizontal scaling (sticky connections + need pub-sub like Redis to bridge servers).

---

## 9. Real-Time Tech Choice
- **Polling**: simplest, wasteful (repeated empty requests), works anywhere → checking a background job's status.
- **Long polling**: works through old firewalls/proxies, some latency/overhead per reconnect → legacy chat widgets.
- **SSE**: one-way server→client, simple, auto-reconnect → stock ticker, live sports scores.
- **WebSockets**: full-duplex, both sides push → chat, collaborative editing (Google Docs).
- **gRPC streaming**: bidirectional, service-to-service → real-time telemetry between microservices.
- Push notifications (APNs/FCM) ≠ WebSocket — OS-level, works when app closed; better fit for critical alerts than WS → Uber ride-arrived alert.

---

## 10. Sharding (across DB instances)
**Skip if:** single DB handles volume/throughput/working-set fine.
**Need if:** data too big for one machine, write throughput maxed, working set exceeds memory.
- **Hash-based**: even distribution, poor range queries → user profile lookups
- **Range-based**: fast range queries, risk of hotspots → time-series/trading data
- **Directory-based**: flexible custom logic, extra hop, needs caching → multi-tenant SaaS (Vitess)
- **Geographic**: compliance/latency driven → Revolut EU data residency

---

## 11. Partitioning (within one DB)
**Skip if:** table is small/fast enough with indexing.
**Need if:** table huge even with indexes, known access slice (recent data), slow maintenance ops.
- **Range**: by date, enables pruning + easy old-data drop → orders/logs
- **List**: by discrete category → country-based, compliance
- **Hash**: even spread, no natural range → user activity events
- **Vertical**: split rarely-used/large columns from hot columns → separate bio/blob from core user row

---

## 12. Replication
**Skip if:** downtime tolerable (minutes), single server handles reads fine.
**Need if:** HA required (seconds not minutes), read-heavy load exceeds one server.
- **Primary-replica**: writes→primary, reads spread to replicas; async=lag risk → blogs/content sites
- **Sync vs Async**: sync=no data loss/slower writes (banking); async=fast/some loss risk (like counts)
- **Multi-primary**: writes accepted in multiple regions, needs conflict resolution → global collab tools
- **Quorum-based**: write ok once majority of replicas ack → Cassandra-style, tolerates node loss

---

## 13. Load Balancer (Algorithms)
**Skip if:** single backend instance — nothing to balance across.
**Need if:** traffic must be spread across multiple instances/servers.
- **Round Robin**: fixed circular order, simple and fair when servers are equal capacity; weighted variant sends more traffic to higher-capacity servers → nginx default upstream strategy.
- **Least Connections**: routes to the server with fewest active connections, beats round robin when connections are long-lived/uneven; weighted variant factors in server capacity → WebSocket/gRPC/DB connection pools.
- **Hash-based (IP/URL/Path)**: hashes a request attribute so the same client/resource always lands on the same backend → sticky sessions without a shared session store.
- **Consistent Hashing**: hashes onto a ring so only a fraction of keys remap when a node joins/leaves, unlike plain hashing which remaps almost everything → memcached client-side sharding, CDN request routing.
- **Least Response Time**: picks the server with the lowest observed latency, adapts to real load better than raw connection count → mixed-hardware fleets.
- **Least Bandwidth / Least Requests**: same idea as least-connections but keyed on bandwidth usage or in-flight request count — pick whichever metric best reflects your actual bottleneck → CDN edge nodes (bandwidth-bound), API gateways (request-bound).
- **Random / Power of Two Choices**: random picks are O(1) with no state and hold up well at scale; P2C samples two servers and takes the less loaded one for better balance at near-zero extra cost → Envoy's default policy.

---

## 14. Rate Limiting
**Skip if:** trusted low-traffic internal users, already limited upstream, early-stage/no abuse risk.
**Need if:** public API, auth/login endpoints, expensive resource calls, multi-tenant tiers.
- **Fixed window**: simple, boundary burst flaw (up to 2x limit at window edge) → basic public API tier limits
- **Sliding window log**: precise, memory-heavy → login/security endpoints
- **Sliding window counter**: cheap + accurate hybrid → general public APIs (default)
- **Token bucket**: allows bursts, enforces avg rate → most common (AWS)
- **Leaky bucket**: strictly smooths to constant rate → protecting rate-capped downstream APIs
  Enforce at: gateway/edge (most common), per-service, or per-user/key (Redis-backed).

---

## 15. Distributed Transactions
**Skip if:** everything touched lives in one DB — normal local transaction handles it → transfer between two accounts at the same bank.
**Also skip if:** strict all-or-nothing isn't required — idempotent retries + background reconciliation is simpler than any formal pattern.
**Need if:** operation spans multiple independently-owned services/DBs and needs all-or-nothing (or a guaranteed consistent end state).
- **2PC (two-phase commit)**: coordinator has every participant prepare/lock, commits only once all confirm. Strong immediate atomicity, but blocking (locks held throughout), coordinator=SPOF, doesn't scale, requires all participants to support the protocol → distributed DBs / XA transactions inside one company's infra.
- **Saga**: no locks — eventual consistency via compensating actions; choreography (event-driven, few steps, max decoupling) or orchestration (central coordinator, complex branching). Default modern choice — works across external/untrusted systems → cross-bank transfer, order→payment→shipping chain.
- **TCC (try-confirm-cancel)**: stricter saga variant — explicit try (tentative reserve), confirm (finalize), cancel (release hold) per service. Clearer semantics than ad hoc saga compensation → travel booking, hold seat/funds then confirm or release.
- **Eventual consistency + reconciliation**: each step runs independently with idempotency keys; background job fixes drift later. Simplest option, good when real-time strict correctness isn't required → updating analytics/search index after an order.
- Pair Saga with **transactional outbox** to reliably emit events atomically with DB writes.

---

## 16. Circuit Breakers
**Skip if:** calling only fast/reliable/local resources.
**Need if:** calling external service/DB/API that can slow down or fail, risking cascading failure.
- **Closed**: normal traffic, watching failure rate
- **Open**: trips after failure threshold, fails fast, no hung threads
- **Half-open**: after cooldown, tests recovery with limited requests
  Configure: failure threshold, timeout definition, cooldown length, fallback (cache/default/queue/error). Implement via client library (Resilience4j, Polly) or service mesh (Istio) → Netflix Hystrix.

---

## 17. Data Ingestion Pattern
**Skip if:** source data small/infrequent, freshness in hours is fine, direct query against source is cheap enough.
**Need if:** moving data from source systems into downstream stores/analytics/services regularly, at a scale or freshness direct querying can't handle.
- **Batch**: scheduled bulk extract/load (hourly/nightly), simple, easy to reprocess/backfill, but staleness = interval length → nightly data warehouse ETL, payroll runs.
- **Micro-batch**: batch on short fixed windows (seconds–minutes), most of batch's simplicity with near-real-time freshness → Spark Structured Streaming windows, periodic feature-store refresh.
- **Streaming**: continuous event-by-event processing as data arrives, lowest latency, but adds operational complexity (state, ordering, backpressure, replay) → clickstream analytics, fraud detection.
- **CDC (Change Data Capture)**: tail the source DB's write-ahead log/binlog to stream row-level inserts/updates/deletes, no polling load on source, captures deletes that polling misses → Debezium reading MySQL binlog into Kafka to keep a search index or cache in sync with the primary DB.
- **Polling extraction** (`SELECT WHERE updated_at > last_run`): simplest to build, no log access needed, but misses hard deletes and adds recurring query load to source → cron job syncing a small reference table.
  Combine in practice: CDC→stream for real-time sync + nightly batch for reconciliation/backfill is a common pairing (e.g. Debezium+Kafka feeding a lake, plus a daily full-table batch job to catch drift).

---

## 18. Consensus Algorithms
**Skip if:** a single source of truth is acceptable (one DB, one leader with manual failover).
**Need if:** multiple nodes must agree on shared state/leadership with no single trusted authority, and must keep working through node failures.
- **Raft**: replicated log + elected leader, leader handles all writes and replicates them in order — easier to reason about/implement correctly than Paxos → etcd, Consul, CockroachDB, Kafka KRaft (controller quorum, replacing ZooKeeper).
- **Paxos**: the original quorum-based consensus protocol for agreeing on a single value (chained as Multi-Paxos for a log) — more proven at scale but notoriously hard to implement correctly → Google Chubby, Spanner.
- **Zab (ZooKeeper Atomic Broadcast)**: Raft-like replicated-log protocol purpose-built for ZooKeeper's primary-backup model → ZooKeeper itself.
- **Gossip/SWIM**: peer-to-peer periodic state exchange for membership/failure detection, not consensus on a value — just an eventually-consistent view of the cluster → Cassandra/DynamoDB ring membership, Consul.

---

## 19. Distributed Coordination Primitives
**Skip if:** a single process/DB can own the coordination state.
**Need if:** multiple independent nodes need to safely share leadership, mutual exclusion, or a live registry of who's up.
- **Leader Election**: exactly one node is elected active leader while others stand by, remaining nodes re-elect on failure → primary DB failover, Kafka controller election.
- **Distributed Lock**: mutual exclusion across nodes via a coordination service, must be lease-based (auto-expiring) so a crashed holder can't block others forever → ZooKeeper/etcd locks, Redis Redlock around a critical section.
- **Service Discovery**: registry of live service instances and their endpoints, so callers don't hardcode addresses → Consul/Eureka, Kubernetes DNS/etcd-backed discovery.
- **Heartbeat/Lease**: nodes periodically renew a time-bound lease to prove liveness; a missed heartbeat expires the lease and the node is treated as dead → Kubernetes node health checks, Kafka consumer group liveness.
  Usually built on top of a consensus system (§18) rather than implemented from scratch.

---

## 20. Microservice Patterns
Cross-cutting patterns for structuring/operating a microservice fleet — see Saga (§15), Circuit Breakers (§16), CDC (§17) for concerns already covered elsewhere.
- **API Gateway**: single entry point handling routing/auth/rate limiting for all clients, hides internal service topology → Netflix Zuul, Kong in front of a service mesh.
- **Backend for Frontend (BFF)**: a gateway per client type (web/mobile) instead of one shared gateway, avoids over/under-fetching for different UIs → Netflix's per-device BFFs.
- **Service Mesh / Sidecar**: a proxy deployed alongside each service instance handles retries/mTLS/observability, keeping that logic out of app code → Istio/Linkerd with Envoy sidecars.
- **Database per Service**: each service owns its data exclusively, no shared schema/cross-service joins, enforces loose coupling — but cross-service operations then need Saga/eventual consistency → standard microservices baseline.
- **Strangler Fig**: a new service intercepts and reimplements slices of a monolith's functionality behind a routing layer, monolith shrinks gradually → incremental legacy migrations (Shopify, GitHub).
- **CQRS**: separate models/paths for writes (command) and reads (query), lets each scale/optimize independently, pairs well with event sourcing → read-heavy services with complex write validation, e.g. order management.
- **Event Sourcing**: store state as an append-only log of events instead of current-state rows, current state = replay of events → audit-heavy domains, banking ledgers.
- **Anti-Corruption Layer**: translation layer at a service boundary that prevents a legacy/external system's model from leaking into your domain model → integrating a third-party or legacy system without polluting your own schema.
</content>
</invoke>
