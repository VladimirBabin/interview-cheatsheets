# System Design Interview Cheat Sheets

---

## 1. Relational vs NoSQL
**Relational:** structured data, clear relationships, ACID/multi-row transactions needed, ad hoc queries/joins → e-commerce orders/customers, banking ledgers.
**NoSQL:** schema unstructured/changes often, massive scale, eventual consistency OK, access pattern known upfront so data can be denormalized for it.
Decision test: do you need joins/multi-row ACID across entities (→ relational), or do you know your query pattern in advance and need to scale past one machine (→ pick a NoSQL type below)?

---

## 2. NoSQL Database Types
Pick by dominant access pattern — each trades relational flexibility (joins, ad hoc queries) for one specific strength.
- **Document (MongoDB, Firestore)**: schema flexible/varies per record, nested data fetched in one read, no joins needed → user profiles, product catalogs
- **Key-Value (Redis, DynamoDB)**: O(1) lookup by key, simplest model, fastest → caching, sessions
- **Columnar (Redshift, BigQuery)**: reads only a few columns across billions of rows instead of whole rows → analytics/BI on huge datasets
- **Wide-column (Cassandra, HBase)**: massive write throughput, linear horizontal scaling, tunable consistency → time-series, IoT, stock ticks
- **Graph (Neo4j)**: traversing relationships *is* the query (friends-of-friends, shortest path) — joins this deep would be too expensive in relational → social graphs, recommendations, fraud rings

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

## 5. Sync vs Async / Message Queues
**Sync:** fast ops, need instant confirm/error handling, simple → login request/response.
**Async:** slow ops (mins), traffic spikes, need reliability/decoupling → video transcoding job.
- **Job queues** (RabbitMQ/SQS/Celery): discrete one-off tasks, priority support, flexible consumer scaling → sending emails, resizing uploaded images.
- **Pub-Sub**: one event → many independent subscribers, no shared work queue → order-placed event fanning out to billing, inventory, notifications.
- **Kafka**: durable, ordered (per partition key), replayable, high-volume streams; acks: 0=fast/unsafe, 1=leader only, all=safest → clickstream/analytics pipelines, activity feeds.
- Kafka has no native priority — use separate topics/queues or RabbitMQ priority queues.
- RabbitMQ durability: opt-in (durable queues + persistent msgs + quorum queues) — comparable to Kafka if configured.
- Autoscale consumers on queue depth / consumer lag — best signal for traffic-driven scaling.

---

## 6. REST vs gRPC
**REST:** public APIs, many client languages, need caching/human-readable/debuggable.
**gRPC:** internal service-to-service, high performance (protobuf+HTTP/2), streaming, strict typed contracts.
Example: Stripe API=REST; Uber internal dispatch=gRPC.

---

## 7. Do We Need Real-Time Communication?
**Test:** if update arrives 10s late, does anything break?
**Skip:** infrequent data, delay tolerable (dashboards, catalogs, email summaries).
**Need it:** collaborative/chat (feels broken if delayed), safety/money alerts (fraud, live location), live time-sensitive data (stock, sports, auctions).
Note: real-time complicates horizontal scaling (sticky connections + need pub-sub like Redis to bridge servers).

---

## 8. Real-Time Tech Choice
- **Polling**: simplest, wasteful (repeated empty requests), works anywhere → checking a background job's status.
- **Long polling**: works through old firewalls/proxies, some latency/overhead per reconnect → legacy chat widgets.
- **SSE**: one-way server→client, simple, auto-reconnect → stock ticker, live sports scores.
- **WebSockets**: full-duplex, both sides push → chat, collaborative editing (Google Docs).
- **gRPC streaming**: bidirectional, service-to-service → real-time telemetry between microservices.
- Push notifications (APNs/FCM) ≠ WebSocket — OS-level, works when app closed; better fit for critical alerts than WS → Uber ride-arrived alert.

---

## 9. Sharding (across DB instances)
**Skip if:** single DB handles volume/throughput/working-set fine.
**Need if:** data too big for one machine, write throughput maxed, working set exceeds memory.
- **Hash-based**: even distribution, poor range queries → user profile lookups
- **Range-based**: fast range queries, risk of hotspots → time-series/trading data
- **Directory-based**: flexible custom logic, extra hop, needs caching → multi-tenant SaaS (Vitess)
- **Geographic**: compliance/latency driven → Revolut EU data residency

---

## 10. Partitioning (within one DB)
**Skip if:** table is small/fast enough with indexing.
**Need if:** table huge even with indexes, known access slice (recent data), slow maintenance ops.
- **Range**: by date, enables pruning + easy old-data drop → orders/logs
- **List**: by discrete category → country-based, compliance
- **Hash**: even spread, no natural range → user activity events
- **Vertical**: split rarely-used/large columns from hot columns → separate bio/blob from core user row

---

## 11. Replication
**Skip if:** downtime tolerable (minutes), single server handles reads fine.
**Need if:** HA required (seconds not minutes), read-heavy load exceeds one server.
- **Primary-replica**: writes→primary, reads spread to replicas; async=lag risk → blogs/content sites
- **Sync vs Async**: sync=no data loss/slower writes (banking); async=fast/some loss risk (like counts)
- **Multi-primary**: writes accepted in multiple regions, needs conflict resolution → global collab tools
- **Quorum-based**: write ok once majority of replicas ack → Cassandra-style, tolerates node loss

---

## 12. Rate Limiting
**Skip if:** trusted low-traffic internal users, already limited upstream, early-stage/no abuse risk.
**Need if:** public API, auth/login endpoints, expensive resource calls, multi-tenant tiers.
- **Fixed window**: simple, boundary burst flaw (up to 2x limit at window edge) → basic public API tier limits
- **Sliding window log**: precise, memory-heavy → login/security endpoints
- **Sliding window counter**: cheap + accurate hybrid → general public APIs (default)
- **Token bucket**: allows bursts, enforces avg rate → most common (AWS)
- **Leaky bucket**: strictly smooths to constant rate → protecting rate-capped downstream APIs
  Enforce at: gateway/edge (most common), per-service, or per-user/key (Redis-backed).

---

## 13. Distributed Transactions
**Skip if:** everything touched lives in one DB — normal local transaction handles it → transfer between two accounts at the same bank.
**Also skip if:** strict all-or-nothing isn't required — idempotent retries + background reconciliation is simpler than any formal pattern.
**Need if:** operation spans multiple independently-owned services/DBs and needs all-or-nothing (or a guaranteed consistent end state).
- **2PC (two-phase commit)**: coordinator has every participant prepare/lock, commits only once all confirm. Strong immediate atomicity, but blocking (locks held throughout), coordinator=SPOF, doesn't scale, requires all participants to support the protocol → distributed DBs / XA transactions inside one company's infra.
- **Saga**: no locks — eventual consistency via compensating actions; choreography (event-driven, few steps, max decoupling) or orchestration (central coordinator, complex branching). Default modern choice — works across external/untrusted systems → cross-bank transfer, order→payment→shipping chain.
- **TCC (try-confirm-cancel)**: stricter saga variant — explicit try (tentative reserve), confirm (finalize), cancel (release hold) per service. Clearer semantics than ad hoc saga compensation → travel booking, hold seat/funds then confirm or release.
- **Eventual consistency + reconciliation**: each step runs independently with idempotency keys; background job fixes drift later. Simplest option, good when real-time strict correctness isn't required → updating analytics/search index after an order.
- Pair Saga with **transactional outbox** to reliably emit events atomically with DB writes.

---

## 14. Circuit Breakers
**Skip if:** calling only fast/reliable/local resources.
**Need if:** calling external service/DB/API that can slow down or fail, risking cascading failure.
- **Closed**: normal traffic, watching failure rate
- **Open**: trips after failure threshold, fails fast, no hung threads
- **Half-open**: after cooldown, tests recovery with limited requests
  Configure: failure threshold, timeout definition, cooldown length, fallback (cache/default/queue/error). Implement via client library (Resilience4j, Polly) or service mesh (Istio) → Netflix Hystrix.

## 15. Data Ingestion Pattern                                                                                                                                                                                                                                                                                    
**Skip if:** source data small/infrequent, freshness in hours is fine, direct query against source is cheap enough.                                                                                                                                                                                              
**Need if:** moving data from source systems into downstream stores/analytics/services regularly, at a scale or freshness direct querying can't handle.                                                                                                                                                          
- **Batch**: scheduled bulk extract/load (hourly/nightly), simple, easy to reprocess/backfill, but staleness = interval length → nightly data warehouse ETL, payroll runs.                                                                                                                                       
- **Micro-batch**: batch on short fixed windows (seconds–minutes), most of batch's simplicity with near-real-time freshness → Spark Structured Streaming windows, periodic feature-store refresh.                                                                                                                
- **Streaming**: continuous event-by-event processing as data arrives, lowest latency, but adds operational complexity (state, ordering, backpressure, replay) → clickstream analytics, fraud detection.                                                                                                         
- **CDC (Change Data Capture)**: tail the source DB's write-ahead log/binlog to stream row-level inserts/updates/deletes, no polling load on source, captures deletes that polling misses → Debezium reading MySQL binlog into Kafka to keep a search index or cache in sync with the primary DB.                
- **Polling extraction** (`SELECT WHERE updated_at > last_run`): simplest to build, no log access needed, but misses hard deletes and adds recurring query load to source → cron job syncing a small reference table.                                                                                            
  Combine in practice: CDC→stream for real-time sync + nightly batch for reconciliation/backfill is a common pairing (e.g. Debezium+Kafka feeding a lake, plus a daily full-table batch job to catch drift).
