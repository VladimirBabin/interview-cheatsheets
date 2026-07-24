# System Design Interview Cheat Sheets

---

## 1. Relational vs NoSQL
**Relational:** structured data, clear relationships, ACID needed, complex joins. → e-commerce orders/customers.
**NoSQL:** unstructured/changing data, massive scale, eventual consistency OK.
- Document (MongoDB): flexible JSON data → user profiles
- Key-Value (Redis): fast lookups/caching
- Columnar (Redshift): analytics on huge datasets
- Wide-column (Cassandra): high write throughput, time-series → IoT, stock ticks
- Graph (Neo4j): relationship-heavy data

---

## 2. Caching
**Skip if:** DB handles load fine (<50% repeated reads), data changes constantly (e.g. balances), small/low-traffic system.
**Add when:** >80% reads hit same data, DB connections saturated/timing out.
- **In-memory (local, in-process)**: cache lives inside one app instance's RAM. Zero network hop = fastest possible. But invisible to other instances (each server has its own copy, can go stale/inconsistent) and wiped on restart. Use only when you have a single instance, or per-instance data that doesn't need to match across servers (e.g. local config cache).
- **Memcached**: separate networked service, so multiple app servers *share* the same cache (unlike local in-memory). Still bare-bones: pure key-value, no persistence, no built-in clustering/replication (client-side sharding across nodes). Use when you need a shared cache across servers but only need simple caching — no data structures, no durability, no pub-sub.
- **Redis**: also shared/distributed, but richer — data structures (lists, sets, sorted sets), persistence options, built-in clustering/replication, pub-sub. Use for sessions, carts, leaderboards, anything needing more than plain get/set or needing durability guarantees.
- **Elasticsearch**: not a cache — full-text search, log aggregation
  **Sessions**: cache-only in Redis (TTL), store id/roles/token — not full profile.
  **Carts**: guest=cache only; logged-in=DB (durability) + Redis (speed), cache-aside.
  **JWT+Session hybrid**: JWT=lightweight identity; Session=mutable permissions, instantly revocable.

---

## 3. Sync vs Async / Message Queues
**Sync:** fast ops, need instant confirm/error handling, simple.
**Async:** slow ops (mins), traffic spikes, need reliability/decoupling.
- **Job queues** (RabbitMQ/SQS/Celery): discrete one-off tasks, priority support, flexible consumer scaling
- **Pub-Sub**: one event → many independent subscribers
- **Kafka**: durable, ordered (per partition key), replayable, high-volume streams; acks: 0=fast/unsafe, 1=leader only, all=safest
- **Autoscaling**: queue depth / consumer lag = best scaling signal
- Kafka has no native priority — use separate topics/queues or RabbitMQ priority queues
- RabbitMQ durability: opt-in (durable queues + persistent msgs + quorum queues) — comparable to Kafka if configured

---

## 4. REST vs gRPC
**REST:** public APIs, many client languages, need caching/human-readable/debuggable.
**gRPC:** internal service-to-service, high performance (protobuf+HTTP/2), streaming, strict typed contracts.
Example: Stripe API=REST; Uber internal dispatch=gRPC.

---

## 5. Do We Need Real-Time Communication?
**Test:** if update arrives 10s late, does anything break?
**Skip:** infrequent data, delay tolerable (dashboards, catalogs, email summaries).
**Need it:** collaborative/chat (feels broken if delayed), safety/money alerts (fraud, live location), live time-sensitive data (stock, sports, auctions).
Note: real-time complicates horizontal scaling (sticky connections + need pub-sub like Redis to bridge servers).

---

## 6. Real-Time Tech Choice
- **Polling**: simplest, wasteful, works anywhere
- **Long polling**: works through old firewalls, some latency/overhead
- **SSE**: one-way server→client, simple, auto-reconnect → stock ticker
- **WebSockets**: full-duplex, both sides push → chat, collab editing
- **gRPC streaming**: bidirectional, service-to-service
- Push notifications (APNs/FCM) ≠ WebSocket — OS-level, works when app closed; better fit for critical alerts than WS.

---

## 7. Sharding (across DB instances)
**Skip if:** single DB handles volume/throughput/working-set fine.
**Need if:** data too big for one machine, write throughput maxed, working set exceeds memory.
- **Hash-based**: even distribution, poor range queries → user profile lookups
- **Range-based**: fast range queries, risk of hotspots → time-series/trading data
- **Directory-based**: flexible custom logic, extra hop, needs caching → multi-tenant SaaS (Vitess)
- **Geographic**: compliance/latency driven → Revolut EU data residency

---

## 8. Partitioning (within one DB)
**Skip if:** table is small/fast enough with indexing.
**Need if:** table huge even with indexes, known access slice (recent data), slow maintenance ops.
- **Range**: by date, enables pruning + easy old-data drop → orders/logs
- **List**: by discrete category → country-based, compliance
- **Hash**: even spread, no natural range → user activity events
- **Vertical**: split rarely-used/large columns from hot columns → separate bio/blob from core user row

---

## 9. Replication
**Skip if:** downtime tolerable (minutes), single server handles reads fine.
**Need if:** HA required (seconds not minutes), read-heavy load exceeds one server.
- **Primary-replica**: writes→primary, reads spread to replicas; async=lag risk → blogs/content sites
- **Sync vs Async**: sync=no data loss/slower writes (banking); async=fast/some loss risk (like counts)
- **Multi-primary**: writes accepted in multiple regions, needs conflict resolution → global collab tools
- **Quorum-based**: write ok once majority of replicas ack → Cassandra-style, tolerates node loss

---

## 10. Rate Limiting
**Skip if:** trusted low-traffic internal users, already limited upstream, early-stage/no abuse risk.
**Need if:** public API, auth/login endpoints, expensive resource calls, multi-tenant tiers.
- **Fixed window**: simple, boundary burst flaw
- **Sliding window log**: precise, memory-heavy → login/security endpoints
- **Sliding window counter**: cheap + accurate hybrid → general public APIs (default)
- **Token bucket**: allows bursts, enforces avg rate → most common (AWS)
- **Leaky bucket**: strictly smooths to constant rate → protecting rate-capped downstream APIs
  Enforce at: gateway/edge (most common), per-service, or per-user/key (Redis-backed).

---

## 11. Saga Pattern
**Skip if:** transaction fits in one DB — just use normal ACID transaction.
**Need if:** transaction spans multiple services/DBs, no shared lock possible.
- **Choreography**: no coordinator, services react to events, compensate on failure events. Good for few steps, max decoupling → order→payment→shipping chain
- **Orchestration**: central coordinator drives steps + compensations explicitly. Good for many steps/complex branching → travel booking (flight+hotel+car)
- Pair with **transactional outbox** to reliably emit events atomically with DB writes.

---

## 12. Circuit Breakers
**Skip if:** calling only fast/reliable/local resources.
**Need if:** calling external service/DB/API that can slow down or fail, risking cascading failure.
- **Closed**: normal traffic, watching failure rate
- **Open**: trips after failure threshold, fails fast, no hung threads
- **Half-open**: after cooldown, tests recovery with limited requests
  Configure: failure threshold, timeout definition, cooldown length, fallback (cache/default/queue/error).
  Implement: client library (Resilience4j, Polly) or service mesh (Istio). Example: Netflix Hystrix.
