# SYSTEM DESIGN PATTERNS
**Senior Architect Edition — First and Final Reference**

> **How to use this:** Read once fully. Before any interview, read Part 1 and Part 4. For domain-specific prep (real-time, ML systems), read the relevant Part 3 sections. The goal is not memorisation — it's internalising the *why* so your answers feel like thinking, not recall.

---

## Quick Navigation

| You need... | Go to |
|---|---|
| How to structure a 45-min interview | Part 1 |
| Numbers and estimation math | Part 2 |
| Pattern catalog (what + when + trade-offs) | Part 3 |
| Quick "which one do I pick" decision tables | Part 4 |
| Estimation worked examples | Part 5 |
| Classic problem skeletons (URL shortener, chat, etc.) | Part 6 |
| Your projects as proof points | Part 7 |
| Full glossary | Part 8 |

**Section map:**
A. Read Scalability · B. Write Scalability · C. Consistency · D. Real-Time · E. Data Storage · F. Caching · G. Messaging · H. API Design · I. Resilience · J. Security · K. Observability · L. Geographic Distribution · M. Storage Patterns · N. Availability & Reliability · O. Architecture Patterns · P. Infrastructure Fundamentals · Q. Location-Based Patterns

---

## Part 1: The Interview Playbook

### The 45-Minute Structure

Misallocating time is the most common failure mode. Candidates spend 30 minutes on requirements, have 10 minutes left, and produce a shallow design.

| Phase | Duration | What you're doing |
|---|---|---|
| Requirements | 5–7 min | Clarify scope, users, scale, constraints |
| Estimation | 3–5 min | Back-of-envelope: QPS, storage, bandwidth |
| High-level design | 8–10 min | Components, data flow, no implementation details yet |
| Deep dive | 15–20 min | Pick 2–3 hard problems and go deep |
| Trade-offs & evolution | 5–7 min | What would break at 10×, what you'd do differently |

**The cardinal rule:** Drive the interview. Don't wait for the interviewer to ask "what about caching?" — bring it up yourself and reason through it.

---

### Clarifying Questions That Always Matter

Before drawing anything:

**Scale:**
- How many users? DAU vs MAU?
- Read:write ratio?
- Uniform or bursty load? (social feed vs exam deadline burst)
- SLA? (99.9% = 8.7 hrs/year downtime; 99.99% = 52 min/year)

**Data:**
- What's the data model — relational or document?
- How long do we retain data? Any archival requirements?
- Consistency requirements — bank transfer or social feed?

**Scope:**
- What's in scope for today? (global availability? analytics? mobile?)
- Any hard constraints? (on-prem, specific cloud, compliance)

**The single best question to ask:** "What's the hardest problem in this system?" — tells you where the interviewer wants you to go deep.

---

### How to Talk About Trade-offs (The Senior-Level Skill)

Junior engineers pick a tool. Senior engineers explain what they're giving up.

Template for every choice:

> "I'll use X because [primary reason]. The trade-off is [what we lose]. That's acceptable here because [why this system can tolerate it]. If that constraint changes, we'd revisit by [alternative]."

Examples:
- "I'll use eventual consistency here because a user seeing a feed post 200ms late is acceptable. If this were a bank balance, I'd need linearisability and strong reads from the primary."
- "I'll use SQS FIFO over Standard because order matters within an exam session. The trade-off is 3,000 msg/sec per group vs unlimited — but per-exam traffic is well under that."
- "I'll use Redis for this cache and accept data loss on restart. If durability is required, we'd enable AOF persistence — but that trades write latency."

---

### What Interviewers Actually Evaluate

| Dimension | What they watch for |
|---|---|
| **Breadth** | Do you know the tool landscape? Can you name the right solution category? |
| **Depth** | Do you understand *why* a tool works? Can you explain its internals? |
| **Trade-offs** | Do you acknowledge what you're giving up? Can you defend your choices? |
| **Communication** | Do you think out loud? Is your answer structured? |
| **Failure thinking** | Do you ask "what breaks?" not just "how does it work?" |
| **Numbers** | Can you sanity-check your design with rough math? |
| **Pragmatism** | Do you know when the simple solution is right? |

---

### Handling Pushback

If the interviewer says "that won't scale" or "that's not right":

1. **Don't capitulate immediately.** "Can you help me understand the concern you see?"
2. **Acknowledge the valid point.** "You're right that X becomes a problem at N scale."
3. **Offer the evolution.** "At current scale I'd start simple; as we cross Y threshold, I'd migrate to Z."

Interviewers often test whether you'll fold or hold.

---

## Part 2: Numbers You Must Know Cold

### Latency Hierarchy

| Operation | Latency |
|---|---|
| CPU L1 cache hit | ~0.5 ns |
| L2 cache hit | ~7 ns |
| RAM read | ~100 ns |
| SSD read (sequential) | ~100 µs |
| HDD seek | ~10 ms |
| Same-datacenter round-trip | ~0.5 ms |
| Cross-region (US East → West) | ~40 ms |
| US → Europe | ~80 ms |
| US → Asia Pacific | ~150 ms |

**Two numbers that matter most:** RAM is ~100× faster than SSD. SSD is ~100× faster than HDD.

---

### Throughput Benchmarks (rough, for estimation)

| System | Read QPS | Write QPS |
|---|---|---|
| Postgres (single node, tuned) | ~10,000 | ~2,000 |
| Postgres + read replicas | ~50,000+ | ~2,000 |
| Redis (single node) | ~100,000 | ~100,000 |
| Cassandra (cluster) | ~500,000 | ~500,000 |
| Kafka (per partition) | ~1M msgs/sec | ~500K msgs/sec |
| Typical REST API (Django/FastAPI) | ~2,000–5,000 | ~1,000–2,000 |

---

### Storage Quick Reference

| Data type | Typical size |
|---|---|
| Tweet / short text | ~280 bytes |
| User profile row | ~1 KB |
| Photo (compressed) | ~200 KB |
| HD video (1 min) | ~150 MB |
| 4K video (1 min) | ~400 MB |

---

### The Estimation Framework (10 steps)

1. Ask: DAU (daily active users)
2. Ask: reads vs writes ratio
3. Compute: QPS = DAU × actions/day ÷ 86,400
4. Add: peak multiplier (2–5×)
5. Compute: storage/year = write QPS × data size × 86,400 × 365
6. Compute: bandwidth = read QPS × response size
7. Ask: replication factor (typically 3×)
8. Apply: cache hit ratio (80% cache hit = 20% of reads reach DB)
9. Round aggressively: 86,400 ≈ 10⁵
10. **State assumptions out loud** — interviewers credit the reasoning, not just the number

### Mental Math Shortcuts

| Fact | Value |
|---|---|
| Seconds in a day | 86,400 ≈ 10⁵ |
| 1M DAU × 1 action/day | ~12 QPS |
| 1B DAU × 1 action/day | ~12,000 QPS |
| 1M photos/day × 200KB | 200 GB/day storage |
| Twitter at peak | ~500K reads/sec, ~6K writes/sec |

---

## Part 3: Core Patterns Catalog

Each entry: **What it is → Use when → Avoid when → Key trade-off → Failure mode → Proof point**

---

### A. Read Scalability Patterns

---

#### A1. Read Replicas

**What it is:** Secondary copies of your primary database that serve read queries while the primary handles writes only.

**Use when:** Reads >> writes. Queries can tolerate slight staleness (replication lag). You've already exhausted indexes and connection pooling.

**Avoid when:** You need read-your-own-writes consistency. Replication lag is unacceptable (financial balances, inventory).

**Key trade-off:** Replication lag (typically 1ms–1s, can spike under load). Reads may return stale data.

**Failure mode:** If replica lag grows during a primary overload event, replicas compound the problem — clients get stale data even after writes succeed.

**Interview move:** "Before adding replicas, I'd check: are we connection-pool-limited (PgBouncer), query-limited (missing index), or truly throughput-limited? Replicas only solve the last one."

##### Master-Slave vs Master-Master Replication

| | Master-Slave | Master-Master |
|---|---|---|
| Writes | Primary only | Both nodes accept writes |
| Reads | Primary + replicas | Both nodes |
| Conflict risk | None | Yes — concurrent writes to same row need resolution |
| Failover | Promote replica (manual or automatic) | Other master continues immediately |
| Use when | Read scaling; simple consistency | High write availability; geographic distribution |

**Master-Master conflict resolution:** Last-write-wins (LWW) by timestamp is simplest but risks losing data on clock skew. Application-level merge is safest but complex. Most systems avoid master-master for transactional data and use it only where conflicts are rare or acceptable (e.g., session stores, counters).

---

#### A2. CDN (Content Delivery Network)

**What it is:** Geographically distributed proxy servers that cache and serve content from the edge nearest the user.

**Use when:** Static assets (JS, CSS, images, video). Read-heavy content with infrequent updates. Global user base needing low latency.

**Avoid when:** Content is personalised per user. Content updates need instant global propagation.

**Key trade-off:** Cache invalidation is hard. CDN serves stale content until TTL expires or you actively purge.

**Fix for stale content on deploy:** Content-hash filenames (`main.abc123.js`). The URL changes on every deploy, forcing a cache miss globally.

---

##### Push CDN vs Pull CDN — The Fundamental Decision

This is the first architectural choice you make when introducing a CDN. Everything else (TTL, invalidation) is secondary.

| Dimension | Pull CDN | Push CDN |
|---|---|---|
| **How it works** | Content stays on your origin. First request → CDN pulls from origin → caches at edge. Subsequent requests → cache hit. | You upload content to CDN edge nodes yourself, before any user requests it. |
| **First request** | Slow (cache miss → origin pull) | Fast (content already at edge) |
| **Who manages the edge** | CDN manages itself — caches on demand, evicts on TTL | You manage it — you push updates, you control what's cached |
| **Storage cost** | Only caches what's been requested | Pays for everything you push, regardless of demand |
| **Best for** | Unpredictable traffic, large libraries, content you can't pre-predict | Predictable spikes, small known content set, large media files |
| **Stale content risk** | TTL-based; old content served until expiry | You control it — push the new version when it's ready |
| **Example** | A blog's images — cached on first visitor, served from edge after | Netflix pre-positioning episode files to ISP-level edges hours before a premiere |

**When Push CDN wins:**
- You know the traffic spike is coming (movie premiere at 8pm — push files at 7pm)
- The content is large and an origin pull during a spike would overwhelm your servers
- You have a small, predictable content set (a software release, a sporting event's media pack)

**When Pull CDN wins:**
- Content library is large and you can't predict what will be popular
- Traffic patterns are organic and unpredictable
- You want zero operational overhead — the CDN manages itself

**The hybrid reality:** Most large platforms use both. Static assets (JS, CSS) use pull. Large known media files (a film releasing at midnight) use push. The decision is per content type, not per system.

**Interview move:** "Before picking pull or push, I'd ask: do we know in advance what content will be popular, and can we predict when? If yes, push gives us a guaranteed cache hit from the first request. If the content library is large and access is unpredictable, pull with a good TTL strategy is operationally simpler."

---

#### A3. Denormalisation

**What it is:** Storing pre-computed or duplicated data to speed up reads, at the cost of write complexity.

**Use when:** Joins are the read bottleneck. Data is read far more than written. Data rarely changes.

**Avoid when:** Data changes frequently — you'll have consistency nightmares maintaining multiple copies.

**Key trade-off:** Every write must update multiple places. Consistency is maintained in application code, not the DB.

---

#### A4. Search Index (Elasticsearch / OpenSearch)

**What it is:** A separate index built from your primary store, optimised for full-text search, faceting, and aggregation.

**Use when:** Full-text search (`LIKE '%query%'` on Postgres is a full table scan). Faceted filtering. Autocomplete, fuzzy matching, synonyms.

**Avoid when:** You need transactional consistency between index and DB. The query could be served by a proper Postgres index.

**Key trade-off:** The index is always slightly behind the primary. You need a sync mechanism (CDC, dual-write, or change stream).

**Failure mode:** Index drift — DB has the record, search doesn't. Fix: periodic full reindex + real-time CDC for ongoing sync.

---

#### A5. Load Balancing

**What it is:** Distributes incoming requests across multiple servers so no single server becomes the bottleneck.

##### Layer 4 vs Layer 7

| | Layer 4 (Transport) | Layer 7 (Application) |
|---|---|---|
| Operates on | TCP/UDP packets | HTTP requests |
| Sees | IP + port | URL, headers, cookies, body |
| Speed | Faster (no parsing) | Slower (full HTTP parsing) |
| Routing logic | IP hash, round-robin | Path-based, header-based, host-based |
| Use when | Raw throughput matters, non-HTTP traffic | You need content-aware routing |

##### Load Balancing Algorithms

| Algorithm | How | Use when |
|---|---|---|
| **Round-robin** | Requests go to each server in turn | Servers are identical, requests are similar weight |
| **Least connections** | Route to server with fewest active connections | Requests have variable duration (some are slow) |
| **IP hash** | Hash the client IP → always same server | Session affinity needed without sticky cookies |
| **Weighted round-robin** | Servers get requests proportional to their weight | Servers have different capacities |
| **Consistent hashing** | Hash the request key → same server for same key | Cache locality (CDN, distributed cache) |

##### Sticky Sessions (Session Affinity)

Forces a client to always hit the same server. Required when in-memory session state lives on the server (not in a shared store).

**The problem it creates:** One server becomes a hot spot. If it dies, that user's session is gone.

**The better solution:** Store session state externally (Redis, DB). Then any server can handle any request — no stickiness needed.

##### Health Checks

The load balancer must know which servers are healthy before routing to them.

- **Active health check:** LB pings `/health` on a schedule. If N consecutive checks fail, the server is removed from rotation.
- **Passive health check:** LB observes real traffic. If a server returns too many 5xx errors, it's removed.

**Liveness vs Readiness (critical for Kubernetes):**
- **Liveness probe:** Is the process alive? If not → restart the container.
- **Readiness probe:** Is the server ready to receive traffic? If not → remove from load balancer pool (but don't restart). Used during startup, DB migrations, graceful shutdown.

**Interview move:** "I'd always store session state in Redis, not in-process. That makes servers stateless, eliminates sticky session headaches, and lets me add or remove instances freely."

---

#### A6. Horizontal vs Vertical Scaling

**Vertical scaling (scale up):** Buy a bigger machine. More CPU cores, more RAM, faster SSD.
- Simple — no code changes, no distributed systems complexity
- Hard ceiling — there's a maximum machine size
- Single point of failure — one big machine is one failure domain
- Cost-inefficient at the top end

**Horizontal scaling (scale out):** Add more machines. Run multiple instances behind a load balancer.
- No hard ceiling — add machines as needed
- Requires stateless services (sessions in Redis, not in-process)
- Requires a load balancer
- More complex to operate

**The escalation path for any system bottleneck:**
1. Profile first — find the actual bottleneck (CPU, memory, I/O, network)
2. Optimise the code or query first — often the real fix
3. Vertical scale — simple, fast, no architecture change
4. Horizontal scale — when vertical ceiling is hit or HA is required

**Interview move:** "I'd always ask what the bottleneck actually is before scaling. Horizontal scaling solves throughput. Vertical scaling solves latency (bigger machine, faster single-request execution). They're not interchangeable."

---

#### A7. Forward Proxy vs Reverse Proxy

Two completely different things that share the word "proxy." Confusing them in an interview is a red flag.

| | Forward Proxy | Reverse Proxy |
|---|---|---|
| **Sits in front of** | Clients | Servers |
| **Who configures it** | The client (or their org) | The server operator |
| **Client knows target server?** | No — proxy forwards on client's behalf | Yes — client calls the proxy; proxy picks backend |
| **Server knows real client?** | No — sees proxy IP | No — sees proxy IP (unless X-Forwarded-For header) |
| **Primary use cases** | Corporate internet filtering, anonymity (VPN-like), caching for internal users | Load balancing, SSL termination, caching, WAF, DDoS protection |
| **Examples** | Squid, corporate VPNs | NGINX, HAProxy, AWS ALB, Cloudflare |

**Forward proxy — the corporate firewall model:** Every employee's browser is configured to route through the proxy. The proxy can block YouTube, cache popular sites, and log all traffic. The external website sees the proxy's IP, not the employee's.

**Reverse proxy — the standard web architecture model:** Users call `api.example.com`. That resolves to a reverse proxy (NGINX). The proxy terminates SSL, inspects the request, and forwards to the appropriate backend service. The backend sees `127.0.0.1` or the X-Forwarded-For header.

**Why reverse proxy in every production system:**
- **SSL termination** — handle TLS once at the edge; backend services speak plain HTTP internally
- **Load balancing** — distribute across multiple backend instances
- **Caching** — cache static responses; backends never touched for cached paths
- **WAF / DDoS** — absorb and filter malicious traffic before it reaches the application

---

### B. Write Scalability Patterns

---

#### B1. Message Queue / Async Processing

**What it is:** Decouple the request (accepted immediately) from the work (processed later). Producer → Queue → Consumer.

**Use when:** Work takes longer than an acceptable HTTP response. Work is idempotent and retryable. Load is bursty (queue absorbs spikes).

**Avoid when:** The client needs a synchronous result. Work must happen in strict global order across all producers.

**Key trade-off:** Client gets "Accepted", not a result. You must build a status-check mechanism (polling or callback).

**Failure mode:** Consumers fall behind. Queue depth grows unboundedly. Fix: consumer autoscaling on queue depth + DLQ for poison messages.

**Proof point:** CodeMas — POST /submit → 201 → SQS FIFO → Lambda → result in Postgres → client polls GET /result/{id}/ every 1.5s.

---

#### B2. CQRS (Command Query Responsibility Segregation)

**What it is:** Separate your write model (commands) from your read model (queries). Different stores or schemas optimised for each.

**Use when:** Read and write patterns are fundamentally different. Read scalability needs vastly exceed write needs.

**Avoid when:** System is simple. CQRS adds operational complexity that plain CRUD doesn't need.

**Key trade-off:** Eventual consistency between write and read models. Synchronisation complexity.

**Interview move:** "I'd suggest CQRS only after hitting the ceiling on a single store. Start simple, extract read models when you have a specific scaling problem."

---

#### B3. Event Sourcing

**What it is:** Store the full sequence of events that led to current state, not current state itself. State = replay of all events.

**Use when:** You need a complete audit trail (financial, compliance). You need to replay history or time-travel to any past state. Multiple downstream consumers need different projections.

**Avoid when:** Your team isn't experienced with it — it's a significant mental model shift. Querying current state frequently (requires projections + snapshots).

**Key trade-off:** Current-state queries require building and maintaining projections (read models). Schema evolution of historical events is painful.

---

#### B4. Database Sharding

**What it is:** Horizontally partition your database by a shard key. Each shard holds a subset of rows on a separate server.

**Use when:** Single-node write throughput is saturated. Data volume exceeds one machine's capacity.

**Avoid when:** You haven't exhausted vertical scaling, connection pooling, and read replicas first. Queries require cross-shard joins.

**Key trade-off:** Cross-shard queries become expensive. Resharding is painful. Uneven shard keys create hot shards.

**Good shard key properties:**
1. High cardinality (many possible values)
2. Evenly distributed (no hot keys)
3. Co-locates related data (all of one user's data on the same shard)
4. Immutable (user_id = good; user_location = bad)

---

#### B5. Rate Limiting

**What it is:** Control the rate at which a client can make requests. Rejects (HTTP 429) or queues requests exceeding the limit.

| Algorithm | How it works | Use when |
|---|---|---|
| **Token bucket** | Bucket holds N tokens; refills at R/sec; each request consumes 1 | Allow burst up to N, steady-state at R. Most common. |
| **Leaky bucket** | Requests queue; processed at fixed rate | Smooth output regardless of input bursts |
| **Fixed window counter** | Count requests per window; reset at boundary | Simple; has burst problem at window edges |
| **Sliding window log** | Log each request timestamp; count within rolling window | Accurate; memory-intensive |
| **Sliding window counter** | Weighted average of current + previous window | Good balance of accuracy and memory |

**Storage:** Redis — atomic INCR + TTL. Distributed token bucket uses Lua scripts for atomicity.

---

### C. Consistency & Coordination

---

#### C1. CAP Theorem

A distributed system can guarantee at most two of: **C**onsistency, **A**vailability, **P**artition Tolerance. Since partitions *will* happen, the real choice is CP vs AP.

| Choice | Behaviour during partition | Examples |
|---|---|---|
| **CP** | Returns error rather than stale data | Postgres, HBase, Zookeeper |
| **AP** | Returns potentially stale data rather than error | Cassandra, DynamoDB, CouchDB |

**Interview move:** "CAP is the partition scenario. PACELC is the everyday question — even without a partition, we trade latency vs consistency. That's usually more relevant to product decisions."

---

#### C2. Consistency Levels (weakest to strongest)

| Level | Guarantee | Use case |
|---|---|---|
| **Eventual** | All nodes will eventually agree | Social feeds, DNS, metrics |
| **Monotonic read** | A client never reads older data than it has already read | Session reads |
| **Read-your-own-writes** | A client always sees its own writes | Post a comment, see it immediately |
| **Causal** | Causally related operations appear in order | Comment threads |
| **Strong (linearisable)** | Every read returns the latest write from any client | Bank balances, inventory |

**Proof point:** CodeMas uses eventual — client polling every 1.5s means seeing a result 3 seconds late is acceptable. A bank balance is not acceptable with eventual consistency.

---

#### C3. Distributed Locks

**What it is:** Mutual exclusion across processes or machines. Only one holder at a time.

| Implementation | Properties | Use when |
|---|---|---|
| **Redis SETNX + TTL** | Simple, widely used | Most use cases; accept TTL risk |
| **Redlock (multi-node Redis)** | More robust | Require stronger guarantees |
| **Zookeeper ephemeral nodes** | Strong consistency; leader election standard | Critical section, leader election |
| **Postgres advisory locks** | `pg_try_advisory_lock(key)` | DB-centric systems; no extra infra |

**TTL Goldilocks problem:** Too short — holder's work outlasts the lock (another holder starts). Too long — failures leave the lock held indefinitely.

**Proof point:** CodeMas — `SELECT FOR UPDATE` on attempt_counter is a row-level Postgres lock. Prevents two concurrent submissions both passing the attempt limit check.

---

#### C4. Saga Pattern

**What it is:** Implement a distributed transaction as a sequence of local transactions. On failure, compensating transactions roll back completed steps.

| Style | How | Pros | Cons |
|---|---|---|---|
| **Choreography** | Each service reacts to events | Decentralised | Hard to track overall state |
| **Orchestration** | Central saga orchestrator commands each service | Easy to debug | Single point of failure |

**Use when:** Multi-service transactions without 2PC. Steps can be compensated on failure (refund, cancel reservation).

**Avoid when:** Compensation is impossible. You need true atomicity.

---

#### C5. Leader Election

**What it is:** Ensure exactly one node acts as leader for a responsibility at any time. All others are followers.

**Algorithms:**
- **Raft** — understandable consensus; used in etcd, Consul, CockroachDB
- **Paxos** — original consensus; complex to implement correctly
- **Bully** — node with highest ID wins; simple but not partition-safe

**Use when:** Single writer needed (primary DB, cron scheduler). Anti-split-brain: two nodes must never both believe they're leader.

---

#### C6. Two-Phase Commit (2PC)

**What it is:** A protocol for achieving an atomic transaction across multiple independent databases or services. Either all participants commit, or all abort.

**Two phases:**
1. **Prepare phase:** The coordinator asks every participant "can you commit?" Each participant writes the transaction to its WAL and replies Yes or No.
2. **Commit phase:** If all say Yes, coordinator sends Commit to all. If any say No, coordinator sends Abort to all.

**The fatal problem — coordinator failure:** If the coordinator crashes after sending Prepare but before sending Commit, participants are stuck in a locked state waiting indefinitely. This is the "blocking" nature of 2PC.

| Property | 2PC | Saga |
|---|---|---|
| Atomicity | True atomicity — all or nothing | Eventual — compensating transactions on failure |
| Blocking on failure | Yes — participants lock if coordinator dies | No — each step independent |
| Performance | Slow — multiple round trips, locks held | Fast — local commits, async |
| Use when | Short, fast transactions requiring true ACID | Long-running, multi-service workflows |

**Why Saga beats 2PC for microservices:** In a microservices world, locking resources across services for the duration of a coordinator round-trip is too expensive and too fragile. Saga accepts eventual consistency in exchange for availability.

**When 2PC is still used:** Within a single database cluster where the coordinator is the DB itself (Postgres distributed transactions). Not across independent services.

---

### D. Real-Time & Streaming

---

#### D1. Comparison Table

| Mechanism | Direction | Connection | Reconnect | Scale | Use when |
|---|---|---|---|---|---|
| **Client polling** | Client → Server | New HTTP each poll | Automatic | Easy (stateless) | Result in DB after async process |
| **Long polling** | Server → Client | Held open until event | Manual | Medium | Low-frequency events, HTTP only |
| **SSE** | Server → Client | Persistent HTTP stream | Auto (EventSource) | Medium | Dashboard, notifications, 1-way push |
| **WebSocket** | Bidirectional | Persistent TCP | Manual | Harder (stateful) | Chat, live collaboration, gaming |
| **Kafka** | Server → Consumer groups | Persistent, offset-based | From offset | High + durable | Replayable streams, multi-consumer |

---

#### D2. When Each Wins

**Client polling:** Result is written to DB after async work (Lambda writes to Postgres → client polls). Stateless, easy to scale horizontally.

**SSE:** Server has an event to push. One-directional. Simpler than WebSocket. Note: requires held HTTP connections — each client occupies a server thread unless using async (gevent, asyncio).

**WebSocket:** Both sides need to send. Latency < 100ms required. Note: stateful — needs sticky sessions or a pub/sub layer (Redis) behind a load balancer.

**Kafka:** Durable, replayable, multiple independent consumers at massive scale. Use when you can't afford to lose messages and need replay.

---

#### D3. CDC (Change Data Capture)

**What it is:** Reading the database's write-ahead log (WAL) to capture every insert/update/delete as an event, without changing application code.

**Tools:** Debezium (open source), AWS DMS, PG Logical

**Use when:** Syncing data to another store (Elasticsearch, data warehouse) without touching application code. Invalidating caches on DB change. Getting events from legacy code that doesn't publish events.

**Key trade-off:** WAL consumers must keep pace with write rate, or lag grows unboundedly.

---

### E. Data Storage Selection

---

#### E1. The Decision Framework (ask in order)

1. **What access pattern dominates?**
   - Key lookups → Redis or DynamoDB
   - Relational queries with joins → Postgres
   - Full-text search → Elasticsearch
   - Ordered time-series → Cassandra or TimescaleDB
   - Graph traversal → Neo4j
   - Semantic similarity → pgvector or Pinecone

2. **What consistency is required?**
   - Strong ACID → SQL (Postgres, MySQL)
   - Eventual ok, scale matters → NoSQL (Cassandra, DynamoDB, MongoDB)

3. **What's the write pattern?**
   - High write throughput, append-only → Cassandra (LSM tree)
   - Moderate writes with complex queries → Postgres (B-tree)

4. **What's the scale?**
   - < 10M rows, < 10K QPS → Postgres with good indexes
   - > 100M rows, > 50K read QPS → Read replicas + cache
   - > 1B rows or > 100K write QPS → Sharding or Cassandra

---

#### E2. Database Quick Reference

| DB | Type | Sweet spot | Avoid when |
|---|---|---|---|
| **PostgreSQL** | Relational | Complex queries, ACID, < 10TB | Write throughput > 10K/s without sharding |
| **Redis** | Key-value + data structures | Cache, session, leaderboard, pub/sub, rate limit | Primary datastore (memory-only risk) |
| **MongoDB** | Document | Flexible schema, nested docs, heterogeneous data | Highly relational data requiring joins |
| **Cassandra** | Wide-column | Time-series, IoT, write-heavy, high availability | Complex queries, joins, aggregations |
| **DynamoDB** | Key-value / document | Serverless, auto-scale, AWS-native, simple access | Complex queries, no fixed access pattern |
| **Elasticsearch** | Search | Full-text, faceted filtering, analytics | Primary transactional store |
| **ClickHouse** | Columnar OLAP | Analytics, aggregations over large datasets | Frequent single-row updates |
| **TimescaleDB** | Time-series (Postgres ext.) | Metrics, IoT sensor data, monitoring | General relational queries |
| **Neo4j** | Graph | Social graphs, recommendations, fraud detection | Non-graph data |
| **pgvector** | Vector (Postgres ext.) | Semantic search, RAG, < 100M vectors | Billions of vectors |
| **Pinecone** | Vector (managed) | Billion-scale semantic search | Self-hosted or cost-sensitive |

---

#### E3. Database Federation

**What it is:** Split a single monolithic database into multiple databases by functional domain. Users DB, Products DB, Orders DB — each an independent database server.

Different from sharding: **sharding splits rows of the same table** across servers. **Federation splits tables by function** — different concerns live on different servers entirely.

| | Sharding | Federation |
|---|---|---|
| Splits by | Row (horizontal) | Function/domain (vertical) |
| Example | Users 0-499K on shard 1, 500K-1M on shard 2 | User DB, Product DB, Order DB |
| Cross-partition joins | Hard (cross-shard) | Hard (cross-DB) |
| Scales | Write throughput for one table | Independent scaling per domain |
| Use when | One table is too big for one machine | Different tables have different access patterns |

**When federation wins:** The User service reads users all day. The Order service reads orders all day. If they share a DB, they share connection pools, query plans, and failure domains. Federation gives each service its own DB, letting them scale independently. This is a common step on the microservices journey.

**The downside:** No cross-DB joins. If the Order service needs user details, it either calls the User service API or accepts some data duplication.

---

#### E4. Materialized Views

**What it is:** A pre-computed query result stored as a physical table. Unlike a regular view (which re-runs the query on every access), a materialized view is computed once and stored — reads hit the stored result.

**Use when:** A query is expensive (joins, aggregations) and the underlying data doesn't change every second. Dashboard metrics, reporting tables, pre-computed leaderboards.

```sql
CREATE MATERIALIZED VIEW course_stats AS
  SELECT course_id, COUNT(*) as total_students, AVG(score) as avg_score
  FROM enrollments
  GROUP BY course_id;

REFRESH MATERIALIZED VIEW course_stats;  -- run on schedule or on data change
```

**Trade-off:** Staleness. The view reflects data at the last refresh. Refresh frequency = freshness vs compute cost. `CONCURRENTLY` flag lets Postgres refresh without locking reads.

**Interview move:** "Instead of adding an index or denormalising, I'd first ask: can this query be a materialized view refreshed every 5 minutes? It gives read performance without touching the write path."

---

#### E5. The N+1 Query Problem

**What it is:** A loop that fires one query to get N parent records, then fires N additional queries to fetch related records for each parent — N+1 queries total.

```python
# N+1: fetches 1 course list, then 1 query per course for instructor
courses = Course.objects.all()           # 1 query
for course in courses:
    print(course.instructor.name)        # N queries — one per course
```

```python
# Fixed: one query with JOIN
courses = Course.objects.select_related('instructor').all()  # 1 query
```

**Why it matters at scale:** 100 courses = 101 queries. 10,000 courses = 10,001 queries. Each query has network round-trip overhead. At 10ms per query, 10,001 queries = 100 seconds. One JOIN = 10ms.

**Fix strategies:**
- **Eager loading** — `select_related()` (Django), `include()` (Rails), `JOIN FETCH` (JPQL)
- **Batch loading** — fetch all IDs first, then `WHERE id IN (...)` — one query for all children
- **DataLoader pattern** — batches and deduplicates queries in GraphQL resolvers

---

### F. Caching Strategies

---

#### F1. Strategy Comparison

| Strategy | Read flow | Write flow | Staleness risk | Use when |
|---|---|---|---|---|
| **Cache-aside (lazy)** | Miss → app reads DB → app fills cache | App writes DB; cache invalidated or TTL expires | Medium | Most common; app controls cache |
| **Write-through** | Hit → serve cache | App writes cache AND DB simultaneously | Low | Must be fresh immediately after writes |
| **Write-behind** | Hit → serve cache | App writes cache only; async flush to DB | Very low reads; data loss risk on crash | Extreme write throughput |
| **Read-through** | Miss → cache reads DB automatically | App writes DB; cache updated on next read | Medium | ORM-style caching |
| **Refresh-ahead** | Always hit (proactively refreshed) | Cache refreshes entries automatically before TTL expires, based on predicted access | Very low | Predictable access patterns (news homepage, trending content) |

---

#### F2. Cache Invalidation Strategies

| Strategy | How | Use when |
|---|---|---|
| **TTL** | Set expiry; accept staleness window | Data changes infrequently |
| **Event-driven** | Invalidate on write event | Data changes frequently; must be fresh |
| **Stampede protection** | Lock + background refresh | High traffic; many concurrent misses on same key |
| **Content-hash key** | URL includes hash of content (assets) | Static assets; never invalidate |
| **Stale-while-revalidate** | Serve stale; refresh in background | UI needs speed; slightly stale is ok |

---

#### F3. Common Cache Problems

**Cache stampede (thundering herd):** Popular key expires. Thousands of concurrent requests all miss and hammer the database simultaneously.
- Fix: Probabilistic early expiration (refresh slightly before TTL) or fetch-lock (only one thread fetches; others wait).

**Hot key:** One key (viral tweet, popular product) gets millions of reads/sec. Single Redis node becomes the bottleneck.
- Fix: Local in-process LRU cache in the app, or key sharding (prefix with random suffix, round-robin reads).

---

### G. Messaging & Event Patterns

---

#### G1. Queue vs Pub/Sub vs Streaming

| Pattern | Delivery | Consumers | Replay? | Use when |
|---|---|---|---|---|
| **Queue** (SQS, RabbitMQ) | One consumer per message | Competing consumers | No | Background jobs; work queue |
| **Pub/Sub** (SNS, Redis pub/sub) | All subscribers | Fan-out | No | Notifications; event fan-out |
| **Streaming** (Kafka, Kinesis) | One consumer group | Multiple independent groups | Yes (from offset) | Audit log; replayable events; multi-consumer |

---

#### G2. Delivery Guarantees

| Guarantee | Meaning | Cost | Use when |
|---|---|---|---|
| **At-most-once** | May be dropped; never duplicated | Lowest | Metrics, telemetry — loss is ok |
| **At-least-once** | Delivered at least once; may duplicate | Medium | Default; make consumers idempotent |
| **Exactly-once** | Exactly once, guaranteed | Highest | Financial transactions, inventory |

**Pragmatic rule:** Design for at-least-once + idempotent consumers. Exactly-once is expensive and often unnecessary.

**Idempotency key pattern:** Each message carries a unique ID. Consumer checks: "Have I processed this ID?" (Redis SET NX or DB upsert). If yes, skip.

---

#### G3. The Outbox Pattern

**Problem:** You write to DB and publish to a queue. If the publish fails after the DB write, you have inconsistency.

**Solution:** Write the event to an `outbox` table in the same DB transaction as your domain write. A separate process reads the outbox and publishes to the queue.

```sql
BEGIN;
  INSERT INTO orders (id, ...) VALUES (...);
  INSERT INTO outbox (event_type, payload) VALUES ('order_created', '{...}');
COMMIT;
-- Outbox poller reads committed rows and publishes to queue
```

The event is published if and only if the DB transaction committed.

---

#### G4. Backpressure

**What it is:** A mechanism for a consumer to signal to a producer to slow down when it can't keep up. Without backpressure, a fast producer overwhelms a slow consumer until memory is exhausted or messages are dropped.

**The problem without backpressure:**
- Producer writes 50K messages/sec into a queue
- Consumer processes 10K messages/sec
- Queue depth grows 40K/sec → eventually exhausts memory or hits max depth → messages dropped or OOM crash

**How to handle it:**

| Approach | How | Use when |
|---|---|---|
| **Bounded queue** | Queue has a max size; producer blocks or drops when full | You control both producer and consumer |
| **Consumer autoscaling** | Add consumers when queue depth grows past threshold | Cloud environment, elastic workloads |
| **Rate limiting the producer** | Producer slows down when queue depth is high | Producer can tolerate being throttled |
| **Load shedding** | Deliberately drop low-priority messages when overloaded | Real-time systems where stale data is useless |
| **Reactive Streams / async back-pressure** | Protocol-level signalling (gRPC flow control, Kafka consumer lag) | Library/framework handles it automatically |

**Interview move:** "I'd instrument queue depth and consumer lag as key metrics. Autoscale consumers when depth exceeds a threshold. Add a DLQ for messages that fail after max retries. If the queue is chronically deep, the root cause is either an underprovisioned consumer fleet or a burst the system was never designed for."

---

#### G5. Schema Evolution (Kafka / Event Streams)

**The problem:** You have 100 consumers reading a Kafka topic. You need to change the message format (add a field, rename a field, change a type). How do you do it without breaking existing consumers?

**Rules for safe evolution:**
1. **Always add fields as optional with defaults** — existing consumers that don't know the field skip it; new consumers can use it
2. **Never rename a field** — to consumers it looks like deletion + addition; use an alias first
3. **Never change a field's type** — int → string breaks deserialization
4. **Never remove a required field** — old producers still send it; new consumers break

**Schema Registry:** A central service (Confluent Schema Registry, AWS Glue) that stores and versions Avro/Protobuf schemas. Producers register a schema; consumers validate against it. The registry enforces compatibility rules (backward, forward, full) before a new schema is accepted.

| Compatibility mode | Meaning |
|---|---|
| **Backward** | New schema can read data written with the old schema. Safe for consumer upgrades first. |
| **Forward** | Old schema can read data written with the new schema. Safe for producer upgrades first. |
| **Full** | Both directions. Most restrictive. Safest. |

**Practical approach:** Use Protobuf or Avro (not raw JSON) for Kafka messages. Register schemas. Enforce backward compatibility. Deploy consumers before producers when adding fields.

---

### H. API Design

---

#### H1. REST vs gRPC vs GraphQL

| Dimension | REST | gRPC | GraphQL |
|---|---|---|---|
| Protocol | HTTP/1.1 or 2 | HTTP/2 (binary) | HTTP |
| Format | JSON (text) | Protobuf (binary) | JSON |
| Contract | Implicit (OpenAPI optional) | Strict .proto schema | Typed schema |
| Performance | Good | Best | Good |
| Browser support | Yes | Limited (needs proxy) | Yes |
| Streaming | SSE / WebSocket | Native bidirectional | Subscriptions |
| Best for | Public APIs, web, mobile | Internal microservices | Complex client queries |
| Over-fetching | Common problem | Structured response | Solved |

---

#### H2. Pagination Patterns

| Pattern | How | Use when |
|---|---|---|
| **Offset** | `?page=3&size=20` | Total count needed; stable sort |
| **Cursor** | `?after=cursor_id` | Infinite scroll; items can be inserted |
| **Keyset** | `?after_created_at=...&after_id=...` | Large tables; efficient via compound index |

**Why offset pagination breaks at scale:** `OFFSET 10000 LIMIT 20` makes Postgres scan and discard 10,000 rows. Cursor pagination uses an index to start exactly where you left off — O(log n), not O(n).

---

#### H3. API Gateway

Handles cross-cutting concerns at the edge so individual services don't have to:
- Authentication & authorisation
- Rate limiting
- Request routing & load balancing
- Protocol translation (REST → gRPC)
- SSL termination
- Request/response logging

Tools: Kong, AWS API Gateway, NGINX, Envoy.

---

#### H4. Service Discovery

**The problem:** In a dynamic cluster, services spin up and down constantly. IP addresses change. How does Service A know the current address of Service B?

**Two models:**

| Model | How | Examples |
|---|---|---|
| **Client-side discovery** | Client queries a service registry directly, picks an instance, and calls it. Client owns the load balancing logic. | Netflix Eureka, Consul (direct) |
| **Server-side discovery** | Client calls a load balancer or proxy. The LB queries the registry and forwards to a healthy instance. Client is unaware of discovery. | AWS ALB + ECS, Kubernetes Services, Envoy |

**Service registry:** A database of service instances and their health status. Services register on startup, deregister on shutdown. Health checks confirm liveness.

**DNS-based discovery (Kubernetes):** Every Kubernetes Service gets a stable DNS name (`my-service.namespace.svc.cluster.local`). Clients call the DNS name; kube-dns resolves to a healthy pod IP. Simple, no special client library needed.

**Interview move:** "In Kubernetes I'd use a Service + ClusterIP — DNS-based server-side discovery, zero client changes. Outside Kubernetes, Consul with Envoy sidecars gives the same result with automatic health-check-driven deregistration."

#### H5. API Versioning

**The problem:** You need to change an API in a way that would break existing clients. How do you evolve without forcing simultaneous upgrades?

| Strategy | Example | Pros | Cons |
|---|---|---|---|
| **URL versioning** | `/api/v1/users` `/api/v2/users` | Explicit, cacheable, easy to document | URL pollution; clients must update imports |
| **Header versioning** | `Accept: application/vnd.api+json; version=2` | Clean URLs | Less visible, harder to test in browser |
| **Query param** | `/api/users?version=2` | Easy to add | Not RESTful; easily forgotten |

**Deprecation lifecycle:** v1 deprecated → 6-month sunset window → traffic monitoring → v1 removed. Never remove without a sunset period and traffic data showing zero usage.

---

#### H6. Backend for Frontend (BFF) Pattern

**What it is:** Instead of one generic API serving all clients (web, mobile, third-party), create a separate backend tailored to each client type.

```
Mobile App  ──►  Mobile BFF  ──┐
Web App     ──►  Web BFF     ──┼──► Core Microservices
Third-party ──►  Public API  ──┘
```

**Why:** A mobile app on a 3G connection needs a single coarse-grained API call that returns exactly the fields needed — no over-fetching, no multiple round trips. A web app on broadband can handle finer-grained calls. A generic API is a compromise that serves neither well.

**What each BFF does:**
- Aggregates multiple microservice calls into one response (eliminates mobile round trips)
- Shapes the response for the specific client (drops unused fields, flattens nested structures)
- Handles client-specific auth flows (mobile uses refresh token rotation; web uses cookies)
- Independently deployable — mobile BFF can change without affecting the web BFF

**When not to use it:** Small teams. One client type. The API is already simple and well-shaped. BFF adds operational overhead (another service to deploy and monitor per client type).

**Interview move:** "I'd introduce BFF when mobile and web clients have diverging needs and the generic API is getting cluttered with `?include_fields=` hacks to avoid over-fetching. It's a natural split point when teams own different clients."

---

### I. Resilience Patterns

---

#### I1. Circuit Breaker

**What it is:** Monitor calls to a dependency. When failures exceed a threshold, "open" the circuit — fail fast without calling the dependency. After a reset timeout, "half-open" and test.

| State | Behaviour |
|---|---|
| Closed | Normal; failures counted |
| Open | Fail immediately; no call made; reset timer running |
| Half-open | One test request; success → close; failure → open again |

Prevents cascading failures where one slow dependency brings down the whole system.

---

#### I2. Retry with Exponential Backoff + Jitter

```
delay = base_delay × (2 ^ attempt) + random(0, jitter)
```

**Why jitter matters:** Without it, all failed requests retry simultaneously, creating a thundering herd on the recovering service. Jitter spreads retries across time.

**Max retries:** 3–5 is typical. Beyond that, send to a queue rather than blocking the thread.

---

#### I3. Bulkhead

**What it is:** Isolate resources (thread pools, connection pools, semaphores) per dependency. A slow dependency consumes only its own pool, not the shared one.

The ship analogy: watertight compartments mean one flooded section doesn't sink the ship.

---

#### I4. Timeout

**Rule:** Every network call must have a timeout. Never block indefinitely.

- **Connection timeout:** How long to wait to establish a connection (1–5s typical)
- **Read timeout:** How long to wait for a response after connecting (domain-specific)

**Interview move:** "I'd set the timeout at the 99th percentile of expected response time, not the average. Waiting for a slow p99 is acceptable; waiting for a hung connection indefinitely is not."

---

#### I5. Health Checks — Liveness vs Readiness

These are two distinct probes that serve different purposes. Confusing them causes production incidents.

| Probe | Question it answers | On failure |
|---|---|---|
| **Liveness** | Is this process alive and not deadlocked? | Kubernetes restarts the container |
| **Readiness** | Is this instance ready to serve traffic right now? | Kubernetes removes it from the load balancer pool — does NOT restart |

**Why the distinction matters:**
- During startup, a server may be alive (process running) but not ready (DB migration still running, cache warming). Readiness keeps it out of rotation until it's ready. Without readiness, the LB routes traffic to a server that returns 500s for 30 seconds on every deploy.
- During a graceful shutdown, mark readiness as failing first. The LB drains existing connections. Then the process exits. Without this, in-flight requests are dropped on every deploy.

**What each endpoint should check:**

| Probe | Check |
|---|---|
| `/healthz` (liveness) | Can the process respond at all? Just return 200. No dependency checks. |
| `/readyz` (readiness) | Is the DB connection pool healthy? Are required caches warm? Are background jobs running? |

---

#### I6. Deployment Strategies

**Blue-Green Deployment:**
Two identical production environments — Blue (live) and Green (new version). Deploy to Green while Blue serves all traffic. Switch the load balancer to Green. If Green has issues, switch back to Blue in seconds.

- Zero-downtime deploy
- Instant rollback
- Cost: running two full environments simultaneously

**Canary Deployment:**
Route a small percentage of traffic (1–5%) to the new version. Monitor error rate, latency, and business metrics. Gradually increase the percentage if metrics are healthy. Roll back by sending 0% to the new version.

- Lower risk than full deploy — only a fraction of users see a bad version
- Requires traffic splitting at the load balancer or service mesh level
- Requires good monitoring — you need to detect problems in the canary slice

**Feature Flags:**
Deploy code that is off by default. Enable it for specific users, cohorts, or percentages via a config service (LaunchDarkly, GrowthBook, Unleash) without deploying new code.

- Decouples deploy from release — merge anytime, ship when ready
- Enables A/B testing and gradual rollouts
- Operational cost: flag debt accumulates; old flags must be cleaned up

| Strategy | Rollback speed | Traffic exposure | Cost |
|---|---|---|---|
| Standard deploy | Minutes (redeploy) | 100% on bad code | Low |
| Blue-green | Seconds (LB switch) | 100% (but instant rollback) | 2× infra cost |
| Canary | Immediate (0% canary) | 1–5% on bad code | Small overhead |
| Feature flag | Immediate (toggle off) | Only enabled users | Flag management overhead |

---

#### I7. Disaster Recovery — RTO and RPO

Two numbers every architect must know. Interviewers ask these directly.

**RTO (Recovery Time Objective):** How long can the system be down before the business is materially harmed? The maximum acceptable downtime from failure to recovery.

**RPO (Recovery Point Objective):** How much data loss is acceptable? The maximum time gap between the last good backup and the failure event.

| Scenario | RTO | RPO |
|---|---|---|
| Banking core system | Minutes | Zero (no data loss tolerated) |
| E-commerce checkout | 30 min | 5 min |
| Internal analytics dashboard | 4 hours | 24 hours |
| Dev/staging environment | Days | Days |

**DR Strategies (from cheapest to most robust):**

| Strategy | RTO | RPO | Cost | How |
|---|---|---|---|---|
| **Backup & Restore** | Hours–days | Hours | Lowest | Periodic backups to S3; restore on failure |
| **Pilot Light** | 10–30 min | Minutes | Low | Minimal standby (DB replica running, app servers off); start app servers on failure |
| **Warm Standby** | Minutes | Seconds | Medium | Scaled-down replica of production always running; scale up on failure |
| **Active-Active (Multi-region)** | Near zero | Near zero | Highest | Full production in both regions; traffic splits; instant failover |

**Interview move:** "I'd start by asking: what are the RTO and RPO requirements? Those two numbers drive the entire DR architecture. A 4-hour RTO can use backup-restore. A 5-minute RTO requires warm standby at minimum. Zero RPO requires synchronous replication, which trades write latency."

---

### J. Security & Auth

---

#### J1. JWT + Refresh Token Pattern

**JWT structure:** `header.payload.signature` (base64, dot-separated). Server validates signature without a DB lookup — stateless.

| Token | TTL | Use |
|---|---|---|
| Access token | 15 min | API auth on every request |
| Refresh token | 7–30 days | Stored server-side; used only to get new access tokens |

**Why short access TTL:** If stolen, it expires in 15 minutes. Without a revocation blocklist, you can't invalidate it — shorter TTL = smaller attack window.

---

#### J2. OAuth 2.0 / OIDC

| Term | Purpose |
|---|---|
| OAuth 2.0 | Delegated authorisation framework. "Allow this app to read my Google Drive." |
| OIDC | Authentication layer on OAuth 2.0. Returns an ID token (who you are). |
| Authorization code flow | Standard for web apps. Code exchanged server-side for tokens. |
| PKCE | Extension for public clients (SPAs, mobile). Prevents code interception. |

---

#### J3. mTLS (Mutual TLS)

Both client and server present certificates during the TLS handshake. Both authenticate each other.

Use for: service-to-service communication inside a cluster. Service mesh (Istio, Linkerd) handles mTLS automatically without application code changes.

---

#### J4. Secrets Management

**The problem:** Your application needs database passwords, API keys, and certificates. Hardcoding them in source code or `.env` files in production is a security incident waiting to happen.

| Approach | Risk | Use |
|---|---|---|
| Hardcoded in code | Leaked in git history forever | Never |
| `.env` file on server | Leaked if server is compromised; no rotation | Dev only |
| Environment variables from CI/CD | Better; no file on disk | Acceptable for low-sensitivity |
| **Secrets manager** | Centralised, audited, rotatable, least-privilege access | Production |

**Tools:**
- **AWS Secrets Manager** — managed, auto-rotation, IAM-scoped access, $0.40/secret/month
- **HashiCorp Vault** — self-hosted, powerful, complex to operate; dynamic secrets (generates DB creds on demand with TTL)
- **Kubernetes Secrets** — base64-encoded (not encrypted by default); encrypted at rest with KMS; works well within a cluster

**Dynamic secrets (Vault):** Instead of a static password, Vault generates a unique DB credential for each app instance with a TTL. On TTL expiry, the credential is revoked. Compromise of one credential is contained to one instance and one TTL window.

**Interview move:** "I'd never put secrets in environment variables for production. I'd use AWS Secrets Manager with IAM role-based access — the instance role grants permission to read specific secrets, no credentials needed to fetch credentials."

---

### L. Geographic Distribution

---

#### L1. Active-Active vs Active-Passive Multi-Region

| Model | How | Write consistency | Use when |
|---|---|---|---|
| **Active-Passive** | One region is primary (handles writes). Other region(s) are replicas (read-only). On primary failure, promote a replica. | Strong (all writes to one region) | Simpler; acceptable failover RTO of minutes |
| **Active-Active** | All regions accept reads and writes simultaneously. Changes replicated asynchronously between regions. | Eventual (cross-region replication lag) | Highest availability; global write throughput |

**The active-active conflict problem:** User A in Mumbai and User B in London both update the same record simultaneously in their local regions. Replication delivers both updates to each region. Which one wins?

**Conflict resolution strategies:**
- **Last-write-wins (LWW)** — the write with the latest timestamp wins. Simple. Risk: clock skew means the "latest" timestamp is not always the logically last write.
- **Application-level resolution** — the app receives both versions and resolves (e.g., shopping cart: merge both carts; bank balance: flag for human review).
- **CRDTs (Conflict-free Replicated Data Types)** — data structures designed to merge without conflict (counters, sets, maps). No conflict to resolve.

---

#### L2. Geo-Routing

Routing a user's request to the nearest region or the region best suited to serve them.

| Method | How | Use when |
|---|---|---|
| **Latency-based routing** | DNS resolves to the endpoint with lowest measured latency from the user's location | Global user base; want lowest RTT |
| **Geolocation routing** | DNS resolves based on geographic region of the request IP | Data residency requirements (EU data stays in EU) |
| **Failover routing** | Primary region serves all traffic; secondary becomes active only on health check failure | Active-passive setup |

**Data residency:** GDPR requires EU user data to remain in EU. Geolocation routing + region-isolated data stores enforces this. Users in the EU always hit EU endpoints; their data never crosses to US regions.

---

### M. Storage Patterns

---

#### M1. Object Storage (S3)

**What it is:** Flat, key-value storage for binary objects (files). Infinitely scalable, cheap, durable (11 nines). Not a database — no queries, no transactions, no updates (only put/get/delete by key).

**Use when:** Images, video, audio, documents, backups, data lake. Anything that's read as a whole file. Write-once, read-many.

**Key patterns:**

**Presigned URLs:** Instead of routing file downloads through your server, generate a time-limited URL that lets the client download directly from S3. Your server CPU and bandwidth are not involved.
```
# Server generates:
url = s3.presigned_url('get_object', Bucket='...', Key='...', ExpiresIn=3600)
# Client downloads directly from S3 — server is not in the loop
```

**Multipart upload:** For files > 100MB, split into parts and upload concurrently. Benefits: parallel upload (faster), resumable (restart failed parts without re-uploading completed parts), no single-request size limit.

**Lifecycle policies:** Automatically transition objects to cheaper storage tiers (S3 Standard → S3 Infrequent Access → S3 Glacier) based on age. Delete old objects automatically. Critical for cost control on growing data lakes.

**S3 vs Database — when each wins:**

| Situation | Use S3 | Use DB |
|---|---|---|
| Store a video file | ✓ | ✗ |
| Store video metadata (title, duration, upload_date) | ✗ | ✓ |
| Query "all videos uploaded in the last 7 days" | ✗ | ✓ |
| Serve a file to millions of users | ✓ (+ CDN) | ✗ |

**Interview move:** "I'd store the file in S3 and the metadata (filename, size, content-type, S3 key) in Postgres. Serve the file via a presigned URL or CDN — never stream it through the application server."

---

#### M2. Database Indexing

Missing this is the most common gap for engineers who know SQL but not its internals.

**B-Tree index (default):** Tree structure sorted by the indexed column. Supports: equality (`=`), range (`>`, `<`, `BETWEEN`), ordering (`ORDER BY`). Used for almost everything.

**Hash index:** Maps the exact value to a row pointer. O(1) lookup. Supports only equality (`=`). Cannot do range queries or sorting. Rarely the right choice — B-tree handles equality too, and also handles ranges.

**Composite index:** An index on multiple columns. `CREATE INDEX ON orders(user_id, created_at)`. The leftmost prefix rule: a composite index on (A, B, C) can be used for queries filtering on A, A+B, or A+B+C — but not B alone or C alone.

**Partial index:** An index that only includes rows matching a condition. `CREATE INDEX ON orders(created_at) WHERE status = 'PENDING'`. Smaller, faster, cheaper — covers only the rows you actually query.

**Covering index:** An index that includes all columns a query needs. The query is answered entirely from the index without touching the table rows at all (index-only scan). Example: `CREATE INDEX ON orders(user_id) INCLUDE (total, status)`.

**When an index makes things worse:**
- Tables with < 1,000 rows — sequential scan is faster than index overhead
- Very high write rate — every write must update all indexes on the table
- Low-cardinality columns (boolean, status with 3 values) — the DB reads half the table anyway, skip the index

**Interview move:** "Before adding an index I'd run EXPLAIN ANALYZE to see the actual query plan and where the time goes. An index on a foreign key is almost always right. An index on a low-cardinality column is almost always wrong."

---

#### M3. SQL Tuning — Schema and Query Level

Indexing is not the only lever. When EXPLAIN ANALYZE shows a slow query but indexes are already in place, these are the next checks:

**Schema choices that compound at scale:**

| Choice | Guidance |
|---|---|
| `CHAR(n)` vs `VARCHAR(n)` | Use `VARCHAR` — `CHAR` pads to fixed length; wastes disk on short strings |
| `VARCHAR(255)` everywhere | Size the column to reality; oversized columns waste memory in sort/group operations |
| `TEXT` for short strings | `TEXT` and `VARCHAR` are equivalent in Postgres; fine — but use `VARCHAR(n)` on MySQL where TEXT disables some optimisations |
| `DECIMAL` for money | Never `FLOAT` for currency — floating-point precision errors compound. Use `DECIMAL(19,4)` or store as integer cents. |
| Storing binary blobs in DB | Move to S3. BLOBs bloat the DB, skip the cache, and can't be indexed. |

**Query-level tuning:**

- **Avoid `SELECT *`** — fetch only the columns you need; wide rows fill buffer pages faster
- **Avoid joins you can avoid** — for read-heavy paths, denormalise or use a covering index
- **Partition hot tables** — if a table has a "hot" time range (last 7 days of orders), partition by date and route queries to the hot partition only
- **Use the query cache carefully** — Postgres doesn't have a built-in query cache (unlike MySQL); rely on Redis/application caching instead
- **Rewrite correlated subqueries as JOINs** — `SELECT ... WHERE id IN (SELECT id FROM ...)` rewritten as a JOIN can use index-nested-loop instead of scanning per row

**The tuning order of operations:**
1. `EXPLAIN ANALYZE` — see the actual plan and row estimates
2. Add a missing index
3. Rewrite the query (JOIN instead of subquery, pagination)
4. Denormalise or materialise
5. Partition the table
6. Archive old data

---

### K. Observability

---

#### K1. The Three Pillars

| Pillar | What | Tools |
|---|---|---|
| **Logs** | Timestamped events. "Order 123 failed at 14:32." | ELK Stack, CloudWatch, Datadog |
| **Metrics** | Numeric measurements over time. QPS, error rate, p99 latency. | Prometheus + Grafana, Datadog |
| **Traces** | End-to-end request path across services. Where did the time go? | Jaeger, Zipkin, OpenTelemetry |

**Correlation ID:** A unique ID assigned at the edge, passed through every service call and log line. Lets you reconstruct the full journey of any single request.

---

#### K2. RED Method (for services)

| Letter | Metric | Target |
|---|---|---|
| **R** ate | Requests per second | Baseline × peak multiplier |
| **E** rror rate | % requests returning 5xx | < 0.1% |
| **D** uration | Latency distribution (p50, p95, p99) | SLO-driven |

---

#### K3. SLI / SLO / SLA

| Term | Definition | Example |
|---|---|---|
| **SLI** | The metric you measure | p99 latency |
| **SLO** | Internal target | p99 < 500ms, 99.9% of the time |
| **SLA** | External contract with penalty | 99.9% uptime; refund if breached |

**Error budget:** SLO 99.9% → 0.1% downtime budget = 8.7 hrs/year. When the budget is consumed, new deploys pause until it recovers.

---

### N. Availability & Reliability

---

#### N1. Availability — The Nines

| Availability | Downtime / year | Downtime / month |
|---|---|---|
| 99% (two nines) | 87.6 hours | 7.3 hours |
| 99.9% (three nines) | 8.76 hours | 43.8 minutes |
| 99.99% (four nines) | 52.6 minutes | 4.4 minutes |
| 99.999% (five nines) | 5.26 minutes | 26 seconds |

**Availability in series:** System is available only when ALL components are up. If component A = 99.9% and component B = 99.9%, combined = 99.9% × 99.9% = 99.8%. Each dependency reduces overall availability.

**Availability in parallel (redundancy):** System is available when ANY component is up. If both nodes fail = 0.1% × 0.1% = 0.01% chance both fail simultaneously. Combined availability = 1 - (0.001 × 0.001) = 99.9999%. Redundancy dramatically increases availability.

**Interview move:** "Adding a dependency always reduces availability — the combined SLA is the product. If I'm promised 99.99% for my service, I must budget my dependencies' failure rates. Three 99.9% dependencies in series = ~99.7% — I've burned my budget before writing a line of code."

---

#### N2. Availability vs Reliability vs Fault Tolerance

| Term | Meaning | Example |
|---|---|---|
| **Availability** | % of time the system responds to requests | 99.9% uptime |
| **Reliability** | System performs its function correctly over time | Answers are always accurate, not just returned |
| **Fault Tolerance** | System continues operating despite component failures | One node dies; traffic shifts to healthy nodes with no user impact |

A system can be highly available but unreliable (returns 200 but with wrong data). A system can be reliable but not fault-tolerant (single node, always correct, but one crash = full downtime).

---

### O. Architecture Patterns

---

#### O1. Monolith vs Microservices

**Start here before choosing microservices:**

| Dimension | Monolith | Microservices |
|---|---|---|
| Deployment unit | One deployable artifact | One per service |
| Team ownership | Shared codebase | Each team owns their service |
| Scaling | Scale the whole app | Scale individual services independently |
| Development speed (early) | Fast — no network calls, simple local dev | Slow — service boundaries, networking, contracts |
| Development speed (at scale) | Slow — merge conflicts, coordination overhead | Fast — teams deploy independently |
| Operational complexity | Low | High — service discovery, distributed tracing, eventual consistency |
| Data management | Single DB; easy transactions | Each service owns its DB; distributed transactions are hard |
| Failure isolation | One bug can take down everything | Failure in one service doesn't affect others (if circuit breakers in place) |
| When to choose | Early stage, small team, < 5 engineers | Large org, multiple teams, need independent deployability |

**The Modular Monolith:** The middle ground. A single deployable artifact internally structured into well-defined modules with enforced boundaries. Modules can't import each other's internals — only public APIs. Easier to split into microservices later when the boundaries are already clean.

**The Distributed Monolith (anti-pattern):** Microservices that are so tightly coupled they must deploy together. You get all the operational complexity of microservices with none of the independence benefits. Happens when services share a database, have synchronous chains across 5 services, or have no clear domain ownership.

**Interview move:** "I'd always start with a monolith and extract microservices only when there's a specific pain point — a team stepping on another team's deploys, a specific service that needs 10× the compute of everything else, or a clear domain boundary that's causing merge conflicts. Never extract based on speculation."

---

#### O2. Event-Driven Architecture (EDA)

**What it is:** Components communicate by producing and consuming events rather than calling each other directly. Producers emit events without knowing who consumes them. Consumers react to events independently.

```
User signs up
    │
    ▼ (event: user.created)
  ┌─────┬──────────────┬────────────────┐
  │     │              │                │
Email  Analytics  Onboarding flow   Recommendations
service  service     service          service
```

**Key properties:**
- **Loose coupling** — producer doesn't know consumers exist; consumers don't know about each other
- **Async** — producer doesn't wait for consumers; returns immediately
- **Independently scalable** — each consumer scales based on its own load
- **Resilient** — if one consumer is down, events queue up and are processed when it recovers

**EDA vs Request-Response:**

| | Request-Response (REST/gRPC) | Event-Driven |
|---|---|---|
| Coupling | Tight — caller must know callee's address | Loose — producer knows only the event broker |
| Latency | Synchronous — caller waits | Async — fire and forget |
| Error handling | Caller handles failure | Events retry from queue; DLQ for poison messages |
| Use when | Need immediate response | Downstream work can happen later; multiple consumers needed |

**When EDA is the right call:** User registration (send email + update analytics + start onboarding — all independently). Order placed (fulfillment + inventory + notification — parallel and decoupled). Audit logging — every action emits an event; the audit consumer writes them. CDC — DB change events fan out to multiple subscribers.

---

#### O3. MapReduce

**What it is:** A programming model for processing large datasets in parallel across a cluster. Two phases: **Map** (transform each record independently) and **Reduce** (aggregate the mapped results).

```
Input: 1 TB of web server logs

Map phase (runs on every machine in parallel):
  For each log line → emit (url, 1)

Reduce phase (aggregates by key):
  For each url → sum all the 1s → (url, total_hits)

Output: URL hit counts across the entire 1 TB dataset
```

**Why it scales:** The Map step is embarrassingly parallel — each record processed independently, no coordination. The framework (Hadoop, Spark) handles data splitting, scheduling, and failure recovery automatically.

**Map → Shuffle → Reduce:**
1. **Map** — Workers transform input records into (key, value) pairs
2. **Shuffle** — Framework groups all values by key, routes each group to the same Reducer
3. **Reduce** — Aggregates all values for each key

**When to reach for it:**
- Batch processing over massive datasets (TB–PB scale)
- You can express the problem as: process each record → group by key → aggregate per group
- One-time or scheduled jobs (not real-time)

**Modern equivalent:** Apache Spark replaced MapReduce in most use cases. Spark keeps data in memory between stages instead of writing to disk between Map and Reduce — 10–100× faster for iterative algorithms. The MapReduce mental model still applies.

**When it's the wrong tool:** Real-time processing (use Kafka Streams, Flink, or Spark Streaming). Simple queries over structured data (use a columnar DB like ClickHouse). Interactive analytics (too slow for on-demand queries).

**Interview move:** "MapReduce fits batch aggregation over enormous datasets. For real-time aggregation — like a trending topics counter — I'd use stream processing (Kafka + Flink) instead. The key question is always: does the job need to finish in minutes (batch) or milliseconds (stream)?"

---

### P. Infrastructure Fundamentals

---

#### P1. DNS — How Domain Names Resolve

**Why architects need this:** Every system design starts with users typing a URL. DNS is step one.

```
User types api.example.com
    │
    ▼
DNS Resolver (your ISP or 8.8.8.8)
    │  Not cached? Query the hierarchy:
    ├──► Root server (.) → "ask .com TLD"
    ├──► .com TLD server → "ask example.com nameserver"
    └──► example.com Authoritative NS → returns IP: 93.184.216.34
    │
    ▼
TCP connection to 93.184.216.34
```

**Key record types:**

| Record | Purpose | Example |
|---|---|---|
| **A** | Domain → IPv4 address | `api.example.com → 93.184.216.34` |
| **AAAA** | Domain → IPv6 address | `api.example.com → 2600:1404::1` |
| **CNAME** | Domain → another domain (alias) | `www.example.com → example.com` |
| **MX** | Mail server for this domain | `example.com → mail.example.com` |
| **TXT** | Arbitrary text (SPF, DKIM, verification) | `"v=spf1 include:sendgrid.net ~all"` |

**TTL (Time To Live):** How long DNS resolvers cache the answer. Low TTL (60s) = fast failover but more DNS traffic. High TTL (86400s) = less DNS traffic but slow propagation of IP changes.

**DNS for load balancing:** Round-robin DNS — return multiple A records; clients pick one. Not true load balancing (no health checks, sticky client caches), but useful for very basic distribution.

---

#### P2. Virtual Machines vs Containers

| | Virtual Machine | Container |
|---|---|---|
| Isolation level | Full OS — own kernel, own drivers | Process-level — shares host kernel |
| Startup time | Minutes (boot OS) | Seconds (start process) |
| Size | GBs (full OS image) | MBs (app + deps only) |
| Overhead | High — hypervisor + full OS per VM | Low — thin layer above host kernel |
| Security boundary | Strong — separate kernel | Weaker — kernel is shared |
| Use when | Hard multi-tenancy, different OS needed, strong isolation required | Microservices, CI/CD, Kubernetes, fast scaling |

**Hypervisor:** Software that creates and runs VMs. Type 1 (bare-metal): runs directly on hardware (VMware ESXi, KVM). Type 2 (hosted): runs on a host OS (VirtualBox, VMware Workstation).

**Why containers won for microservices:** Seconds to start vs minutes. Dozen MB image vs GB disk. Docker + Kubernetes makes orchestration, scaling, and rolling deploys practical at scale. VMs still win where you need hard security boundaries (cloud multi-tenancy) or a different OS.

---

#### P3. TCP vs UDP

Two transport-layer protocols. You must know when each is appropriate — it comes up in every real-time system design question.

| | TCP | UDP |
|---|---|---|
| **Connection** | Connection-oriented (3-way handshake) | Connectionless |
| **Delivery guarantee** | Guaranteed — retransmits lost packets | Best-effort — packets may be dropped |
| **Ordering** | Guaranteed in-order delivery | No ordering guarantee |
| **Flow control** | Yes — slows sender to match receiver | No |
| **Overhead** | Higher (headers + state + ACKs) | Lower (minimal header, no state) |
| **Latency** | Higher (retransmits add latency) | Lower |

**Use TCP when:** All data must arrive and in order. HTTP, database connections, file transfer, email, SSH.

**Use UDP when:** Speed matters more than guaranteed delivery. Late data is worse than no data. Real-time audio/video (VoIP, video calls), live game state, DNS queries, metric telemetry.

**The video call example:** A dropped video frame is visible for 33ms then gone. Waiting for a retransmit takes 50–300ms and causes a freeze — worse than the dropped frame. UDP + application-level error concealment is the right choice.

**HTTP/3 (QUIC):** HTTP/3 runs over QUIC, which is built on UDP. It implements its own reliable delivery, connection migration, and congestion control — getting TCP-like reliability without TCP's head-of-line blocking. Used by YouTube, Google, Cloudflare.

---

#### P4. Unique ID Generation at Scale

Generating globally unique IDs across a distributed system is a classic interview problem. Requirements: unique, (optionally) sortable by time, no coordination bottleneck.

**Option 1 — UUID (Universally Unique Identifier):**
- 128-bit random value. Effectively zero collision probability.
- No coordination — each node generates independently.
- Problem: not sortable. Random writes into B-tree indexes cause page splits (random I/O). Bad for high-write tables.
- Use when: you need uniqueness without any ordering requirement.

**Option 2 — Database auto-increment:**
- Simple. Sequential. Sortable.
- Single point of coordination — the database is the bottleneck.
- Fails when you shard (each shard generates its own sequence; no global ordering).

**Option 3 — Snowflake ID (Twitter, 2010):**
A 64-bit integer composed of:
```
| 41 bits timestamp (ms since epoch) | 10 bits machine ID | 12 bits sequence |
```
- Sortable by time — higher ID = newer
- No coordination — machine ID is assigned once at startup
- Generates up to 4,096 unique IDs per millisecond per machine
- Epoch of ~69 years before overflow (with a custom epoch start)

**Option 4 — ULID (Universally Unique Lexicographically Sortable Identifier):**
- 128-bit: 48-bit timestamp + 80-bit random
- Lexicographically sortable (unlike UUID v4)
- URL-safe encoding
- No machine ID configuration needed

**When it comes up in interviews:** Any system requiring user-generated content, orders, messages, events. The question tests whether you know why random UUIDs hurt write performance at scale.

**Interview move:** "I'd use Snowflake IDs for primary keys on write-heavy tables — sortable, no DB coordination, and the timestamp embedded in the ID gives me partition routing by time for free."

---

### Q. Location-Based System Patterns

---

#### Q1. Geohashing

**What it is:** Encodes a geographic coordinate (latitude, longitude) into a short alphanumeric string. Nearby locations share a common prefix. Prefix length = precision.

```
San Francisco (37.7749, -122.4194) → "9q8yy"
A nearby block                     → "9q8yx"   ← shares prefix "9q8y"
New York City                      → "dr5ru"   ← different prefix entirely
```

**Why the prefix property matters:** To find all restaurants within 1km of the user, query all geohash cells with the same 5-character prefix — one range scan in the DB instead of a `WHERE ST_Distance(...) < 1000` full table scan.

**Precision table:**

| Prefix length | Cell size | Use case |
|---|---|---|
| 4 chars | ~40 km × 20 km | Country-level |
| 5 chars | ~4.8 km × 4.8 km | City neighbourhood |
| 6 chars | ~1.2 km × 0.6 km | Street level |
| 7 chars | ~150 m × 150 m | Building level |

**Edge case — boundary problem:** A user at the edge of a geohash cell is close to locations in the neighbouring cell, but they share no prefix. Fix: query the 8 surrounding cells in addition to the user's cell (9 cells total).

**How to store and query:**
```sql
-- Store
INSERT INTO places (name, lat, lon, geohash) VALUES ('Cafe', 37.77, -122.41, '9q8yy');
CREATE INDEX ON places(geohash);

-- Query: find nearby places
SELECT * FROM places WHERE geohash LIKE '9q8yy%';  -- fast index range scan
```

**Your systems:** An Uber-like system stores driver locations with geohash precision 6 (street level). On rider request, compute rider's geohash, query surrounding 9 cells for available drivers.

---

#### Q2. Quadtrees

**What it is:** A tree where each node represents a rectangular region. Each node has exactly 4 children — NW, NE, SW, SE. Subdivision continues until each leaf contains at most N points (or reaches minimum cell size).

**When quadtrees beat geohashing:**
- **Dynamic density:** Geohash cells are fixed size. Quadtree cells subdivide where density is high (Manhattan) and stay large where density is low (rural areas). Better for unevenly distributed data.
- **Spatial queries:** "Find all 10 nearest points" is a natural tree traversal. Geohash needs to expand prefix length iteratively until enough results are found.

**Use cases:** Google Maps tiles (divide the world into quadrants recursively), spatial indexing in game engines, collision detection, 2D range queries.

**Interview move:** "For most proximity search questions I'd use geohashing — it maps directly to a database range query and is operationally simple. I'd reach for quadtrees when density is extremely uneven and fixed-size cells would either be too coarse (rural) or return too many results (urban) at the same precision level."

---

---

### SQL vs NoSQL

| Question | SQL wins | NoSQL wins |
|---|---|---|
| Need ACID transactions? | ✓ | ✗ |
| Complex joins and aggregations? | ✓ | ✗ |
| Flexible / evolving schema? | ✗ | ✓ |
| Horizontal write scaling required? | ✗ | ✓ |
| Simple key lookups dominate? | Either | ✓ |
| Strict consistency required? | ✓ | Depends |
| High write throughput (> 10K/s)? | ✗ (without sharding) | ✓ |

**Default:** Start with Postgres. Add NoSQL only when you've hit a specific ceiling Postgres can't solve.

---

### Cache Strategy Selection

| Situation | Use |
|---|---|
| Read-heavy, data changes infrequently, staleness ok | Cache-aside + TTL |
| Write-heavy, reads must be fresh immediately | Write-through |
| Extreme write throughput, can tolerate some loss | Write-behind |
| Access pattern unknown at write time | Cache-aside (lazy) |
| Static assets with long TTL | Content-hash in URL + max-age |

---

### Messaging Pattern Selection

| Situation | Use |
|---|---|
| Background job, one consumer | Queue (SQS Standard) |
| Background job, must not duplicate, ordering matters | Queue (SQS FIFO) |
| Fan-out to multiple subscribers | Pub/Sub (SNS) |
| Audit log, replayable events, multi-consumer | Kafka / Kinesis |
| Reliable event publish from within a DB transaction | Outbox pattern |

---

### Real-Time Delivery Selection

| Situation | Use |
|---|---|
| Result in DB after async process | Client polling |
| Infrequent server-initiated events, HTTP only | Long polling |
| Dashboard / live feed, server pushes | SSE |
| Chat, collaborative editing, gaming | WebSocket |
| Large-scale durable event stream | Kafka |

---

### Consistency Level Selection

| Data | Needed | Use |
|---|---|---|
| Bank balance, inventory | Linearisable | Postgres strong reads from primary |
| User sees their own post | Read-your-own-writes | Sticky session to primary, or write-through cache |
| Social feed | Eventual | Read replicas, Cassandra, cache |
| Comment thread order | Causal | Causal sessions |
| Analytics | Eventual | Pre-aggregated, stream processing |

---

### REST vs gRPC vs GraphQL

| Situation | Use |
|---|---|
| Public API, browser clients | REST |
| Internal service-to-service, performance critical | gRPC |
| Mobile clients needing flexible queries | GraphQL |
| Streaming RPC (server push) | gRPC server streaming |
| Simple CRUD | REST |

---

## Part 5: Estimation Worked Examples

---

### Example 1: Design WhatsApp

**Assumptions:** 2B users, 50% DAU = 1B DAU. 40 messages/day/user. Average message = 100 bytes. 10% include media (avg 100 KB).

**QPS:**
- Write: 1B × 40 ÷ 86,400 = **~500K messages/sec**
- Read: ~5× writes = **~2.5M reads/sec**

**Storage:**
- Text: 500K/sec × 100B × 86,400 × 365 = **~1.5 PB/year**
- Media: 50K/sec × 100KB × 86,400 × 365 = **~150 PB/year**

**Design decisions this math drives:**
- 500K write QPS → Cassandra (designed for this; partition by conversation_id)
- 150 PB/year media → S3 + CDN; thumbnail on upload, lazy-load full
- 2.5M read QPS → Redis cache for recent messages per conversation

---

### Example 2: Design a URL Shortener

**Assumptions:** 100M new URLs/day. Read:Write = 100:1. 5-year retention.

**QPS:**
- Write: 100M ÷ 86,400 = **~1,200 writes/sec**
- Read: **~120,000 reads/sec**

**Storage:**
- Row size: ~300 bytes (7-char code + 200-char URL + metadata)
- 100M/day × 365 × 5 years = 182B rows → **~55 TB**

**Design decisions:**
- 1,200 write QPS → Postgres handles this comfortably
- 120K read QPS → Redis cache essential (80%+ hit ratio for popular URLs)
- 55 TB at 5 years → Sharding by short_code hash

---

### Example 3: Design Twitter

**Assumptions:** 300M DAU. 500K tweets/day (1% of users tweet). Read:Write = 1000:1. Average tweet = 280 bytes.

**QPS:**
- Write: 500K ÷ 86,400 = **~6 writes/sec** (low — Twitter is extremely read-heavy)
- Read: **~6,000 reads/sec** (p50); peak up to **500K reads/sec** (viral content)

**Storage:**
- Tweets: 500K/day × 280B × 365 × 10 years = **~500 GB/year** (tiny)
- Media: the real storage problem — 50K photo tweets/day × 200KB = 10 GB/day

**Design decision this math drives:**
- Storage of text is cheap — the architecture problem is *fan-out* (distributing a tweet to 100M followers)
- This drives the push vs pull debate in feed generation

---

## Part 6: Classic Problem Skeletons

Each problem: Clarify → Estimate → Core Design → Deep Dives to offer

---

### 1. URL Shortener

**Clarify:** Read:write ratio? Custom aliases? Analytics? Expiry?

**Core design:**
- Short code: hash(long_url) → take 7 chars. Collision → retry with suffix.
- DB: Postgres. Schema: `short_code (PK), long_url, created_at, user_id, expiry`
- Cache: Redis. Key: short_code → long_url. TTL: 24h.
- Redirect: 302 (every hit tracked for analytics) vs 301 (browser caches, no tracking)
- At scale: shard by short_code (consistent hashing)

**Deep dives to offer:** Hash collision handling. Custom alias uniqueness. Analytics pipeline (ClickHouse for click events). Multi-region with geo-routing.

---

### 2. Rate Limiter

**Clarify:** Per user? Per IP? Per API key? What happens on exceed — reject or queue?

**Core design:**
- Algorithm: Token bucket (burst-tolerant, steady refill)
- Storage: Redis. Key: `rate_limit:{user_id}:{window}`. Lua script for atomic check-and-decrement.
- Placement: API Gateway layer before traffic reaches services
- Response: HTTP 429 with `Retry-After` header

**Deep dives to offer:** Sliding window vs fixed window. Distributed rate limiting across multiple gateway nodes (Redis shared state). Soft limits (warn) vs hard limits (reject). Priority tiers.

---

### 3. Notification Service

**Clarify:** Channels (push, email, SMS, in-app)? Volume? Delivery guarantees? User preferences?

**Core design:**
- Event producer → Kafka (one topic per channel type)
- Per-channel workers consume independently: push worker, email worker, SMS worker
- Deduplication: notification_id as idempotency key (Redis SET NX)
- User preferences: DB table — per user, per notification type, per channel (opt-out)

**Deep dives to offer:** Thundering herd on bulk notifications (rate-limit workers). Priority lanes (transactional vs marketing). Retry on failed delivery. Delivery receipts. Unsubscribe handling.

---

### 4. Chat System

**Clarify:** 1-to-1 or group? Max group size? Persistence? Online presence?

**Core design:**
- WebSocket for real-time delivery. Each chat server holds connections for a set of users.
- Message path: sender → chat server → Kafka → receiver's chat server → WebSocket push.
- Chat server registry in Redis: user_id → server_id (so the sender's server knows where to route).
- Storage: Cassandra. Partition key: conversation_id. Cluster key: message_timestamp (DESC for recent-first reads).
- Offline delivery: when receiver's chat server doesn't find an active WebSocket, writes to their message inbox in DB. Client fetches on next connect.

**Deep dives to offer:** Message ordering. Group fan-out at scale (fan-out on write for small groups; fan-out on read for large groups). End-to-end encryption. Message deletion / unsend. Read receipts.

---

### 5. Twitter / News Feed

**Clarify:** Post + read volume? Celebrity accounts? Real-time vs eventual? Personalisation?

**Core design — hybrid fan-out:**
- Normal users (< 1M followers): **fan-out on write** — when they post, push to all followers' feed caches in Redis (sorted set by timestamp). Instant reads.
- Celebrities (> 1M followers): **fan-out on read** — skip pre-populating. At read time, merge cached feed + recent celebrity tweets.
- Why hybrid: fan-out on write for 1M followers means 1M Redis writes per tweet — too expensive for celebrities.

**Storage:** Cassandra for tweets. Redis sorted sets for per-user feed caches (keep last 800 tweets max).

**Deep dives to offer:** Trending topics (stream processing, sliding window count). Search (Elasticsearch). Recommendation engine (collaborative filtering). Ad targeting. Pagination.

---

### 6. Distributed Job Scheduler

**Clarify:** One-time vs recurring? Delay SLA? Priority? Exactly-once?

**Core design:**
- Jobs table in Postgres: `id, run_at, status (pending/running/done/failed), payload, retry_count`
- Scheduler: `SELECT FOR UPDATE SKIP LOCKED WHERE run_at <= NOW() AND status = 'pending' LIMIT 10` — workers compete without conflicts
- Workers: process job, update status to done or failed, compute next_run_at for recurring
- Missed jobs (system down during run_at): scheduler catches up on start

**Deep dives to offer:** Exactly-once execution (idempotency in the job itself). Priority queues (multiple tables or priority column with index). Fan-out jobs (1 parent → N child jobs). Distributed workers (partition by queue name). Dead jobs (max_retries exceeded → DLQ table).

---

### 7. Key-Value Store (Design Redis)

**Clarify:** Persistence? Max value size? Replication? Consistency level?

**Core design:**
- In-memory hash map for O(1) get/set
- WAL (write-ahead log) for crash recovery
- Consistent hashing to distribute keys across nodes
- Replication: write to N nodes; quorum W + R > N for strong consistency
- Compaction: SSTable + LSM tree for write-optimised disk layout

**Deep dives to offer:** LSM tree vs B-tree. Conflict resolution (vector clocks, last-write-wins). Gossip protocol for cluster membership. Failure detection. Leader election per shard.

---

### 8. Video Streaming (Design YouTube)

**Clarify:** Upload? Playback? Recommendations? Comments? Scale?

**Core design:**
- **Upload:** Client → Upload service → raw video to S3 → enqueue transcoding job → Fargate/Lambda transcodes to multiple resolutions (360p/720p/1080p/4K) → store segments in S3
- **Playback:** Client → CDN (serves HLS/DASH segments directly from S3). Origin is never hit during playback. Adaptive bitrate: client switches quality based on bandwidth.
- **Metadata:** Postgres for video metadata. Cassandra for view counts (write-heavy increment).
- **Never serve video from the origin server directly.**

**Deep dives to offer:** Transcoding pipeline (ffmpeg, parallelism). Adaptive bitrate (HLS segments + .m3u8 manifest). CDN cache invalidation on re-upload. Recommendation system (matrix factorisation or two-tower neural net). Anti-abuse (CSAM, copyright detection).

---

### 9. Web Crawler

**Clarify:** Domain scope (entire web vs. specific sites)? Freshness (re-crawl frequency)? Scale (pages/sec)? Robots.txt respect? Deduplication required?

**Core design:**
- **Seed URLs** → URL Frontier (priority queue, ordered by page rank + recency)
- **Crawlers** (stateless workers) dequeue URLs, fetch with HTTP, extract links
- **DNS caching** at the crawler — avoid per-URL DNS lookups (expensive at scale)
- **Deduplication** — URL seen set (Redis Bloom filter or hash set) + content hash to detect duplicate pages at different URLs
- **Storage** — raw HTML → S3; metadata (URL, crawl_time, status, outlinks) → Postgres or Cassandra

```
URL Frontier (priority queue)
    │
    ▼
Crawler workers (fetch HTML)
    │           │
    ▼           ▼
Extract links  Store HTML to S3
    │
    ▼
Deduplicate → enqueue new URLs back into Frontier
```

**Politeness:** Respect `robots.txt` — fetch and cache per domain on first visit. Rate-limit per domain (1 req/sec default). Use `crawl-delay` directive if present.

**Re-crawl strategy:** High-value pages (news homepages) re-crawled every hour. Static docs re-crawled weekly. Detect change frequency from response headers (`Last-Modified`, `ETag`).

**Scale math example:** 1B pages, 10KB each → 10TB HTML storage. 10 pages/sec/worker → need 1,157 workers to crawl 1B pages in 24 hours.

**Deep dives to offer:** URL normalisation (canonical URL dedup). JavaScript rendering (headless Chrome for SPAs). Distributed URL frontier (Kafka topics by domain). Anti-bot evasion (User-Agent, rate limiting). Content dedup via SimHash.

---

## Part 7: Your Projects as Proof Points

When the interviewer asks about a pattern, here's your lived evidence:

| Pattern / Concept | Your Project | One-line proof |
|---|---|---|
| Message queue + async execution | CodeMas | SQS FIFO → Lambda → Postgres; client polls result |
| Exactly-once delivery (deduplication) | CodeMas | SQS FIFO dedup on submission UUID; 5-min window |
| Row-level lock / SELECT FOR UPDATE | CodeMas | Attempt counter lock prevents double-count race |
| Client polling (stateless result delivery) | CodeMas | Frontend polls GET /result/{id}/ every 1.5s |
| Batch post-processing pipeline | CodeMas | Plagiarism triggered by Exam.is_active pre_save signal |
| TF-IDF + cosine similarity at scale | CodeMas | Plagiarism Phase 2; O(K×N) not O(N²) via behavioral pre-filter |
| Behavioral anomaly scoring | CodeMas | 4-signal weighted score: paste ratio + speed + tabs + attempts |
| DLQ / failure isolation | CodeMas | SQS DLQ for Lambda failures; exam unaffected |
| RAG (Retrieval-Augmented Generation) | the01.dev | Course content retrieval with 0.45 grounding threshold |
| Vector search (HNSW / ANN) | the01.dev | pgvector ANN index for semantic retrieval at query time |
| Grounding threshold (deterministic refusal) | the01.dev | Below 0.45 cosine similarity: model never called |
| Agent runtime (ReAct loop) | the01.dev / Munshi / Trade | Hermes: reason → tool call → observe → repeat |
| HITL (Human-in-the-Loop) | the01.dev (QuizMe) | LangGraph interrupt: human approves before quiz generates |
| Deterministic workflow engine | the01.dev | LangGraph StateGraph with MemorySaver checkpointer |
| Self-improving AI system (eval → propose → gate) | the01.dev | GEPA: eval → reflect → propose → Pareto gate → human approves |
| Evaluation framework | the01.dev | RAGAS: faithfulness + context precision, sovereign judge |
| Local model serving (sovereign inference) | the01.dev / Munshi / Trade | Ollama/vLLM; zero external generation API calls |
| EU-AI-Act audit logging | the01.dev | Row written before every inference: who, what, model, SOUL hash |
| MCP (typed tool protocol) | the01.dev / Munshi / Trade | FastMCP stdio server; arguments schema-validated before execution |
| Financial decimal arithmetic | Munshi | Python Decimal for GST at ₹5Cr scale; float gives wrong answers |
| Fuzzy string matching | Munshi | Token set ratio for vendor name reconciliation vs GSTR-2A |
| Multi-language interpreter design | Kalaam | 5-phase pipeline: clean → scan → tokenize → interpret → output |
| Offline-first PWA | Kalaam | Service worker + cache-first; zero API calls at runtime |
| npm library with tree-shaking | stringy-core | ESM named exports, zero deps, Babel for Jest compat |
| Multi-agent orchestration | Trade Compliance | Researcher agent → Writer agent; independent SOUL.md per agent |

---

## Part 8: Technical Dictionary

---

### Distributed Systems

**CAP Theorem** — A distributed system can guarantee at most two of: Consistency (all nodes return the same data), Availability (every request gets a response), Partition Tolerance (works despite network failures). Since partitions are unavoidable, the real choice is CP (return an error during a partition) vs AP (return potentially stale data).

**PACELC** — Extension of CAP. Even when there is No partition, you still trade Latency vs Consistency. PACELC forces you to think about the everyday trade-off, not just the rare partition scenario. More relevant for most product decisions than CAP alone.

**Eventual consistency** — Given enough time with no new writes, all replicas will converge to the same value. You may read stale data briefly. Example: your follower count on Twitter might be 1 second behind the true value.

**Linearisability (strong consistency)** — The strongest guarantee. Every operation appears to take effect atomically at a single point between its start and end. Every read returns the latest completed write from any client. Example: bank balance — you must never see a stale value.

**Consensus** — Getting a distributed cluster to agree on a single value. Algorithms: Raft (readable, used in etcd), Paxos (original, complex). Used in: leader election, distributed locks, replicated state machines.

**Split brain** — Two partitions each believe they're the active primary and both accept writes. When the partition heals, you have conflicting data with no ground truth. Prevention: quorum writes (require majority acknowledgement before committing).

**Quorum** — A majority of nodes (N/2 + 1). For a 5-node cluster, quorum = 3. Operations requiring quorum can tolerate (N - quorum) node failures while maintaining consistency.

---

### Storage & Databases

**ACID** — Atomicity (all or nothing), Consistency (constraints always satisfied), Isolation (concurrent transactions don't interfere), Durability (committed data survives crashes). The guarantee of relational databases.

**BASE** — Basically Available, Soft state, Eventually consistent. The NoSQL trade-off: sacrifices strict consistency for availability and horizontal scalability.

**LSM Tree (Log-Structured Merge Tree)** — Write-optimised storage. All writes go to an in-memory buffer (MemTable), flushed to sorted immutable files (SSTables) on disk. Reads merge multiple SSTables. Used by: Cassandra, RocksDB. Fast writes, slightly slower reads than B-tree.

**B-Tree** — Read-optimised storage. Data in sorted tree pages on disk. Reads: O(log n) page fetches. Writes update in place. Used by: Postgres, MySQL. Faster reads, slower sequential writes vs LSM.

**WAL (Write-Ahead Log)** — Before modifying data on disk, the change is written to an append-only log first. If the system crashes mid-write, the WAL is replayed to restore consistency. Also used for CDC (reading the log to capture change events without touching the application).

**Consistent hashing** — A hashing scheme where adding or removing a node only remaps ~1/N of keys (not all of them). Used in distributed caches and databases to minimise data migration when nodes join or leave.

**JSONB** — Postgres's binary JSON column type. Stored as binary (not text) so it's faster to query than the `json` type. Supports GIN indexes for querying nested fields. Useful for flexible schema within a relational DB.

**Connection pool** — A cache of database connections reused across requests. Opening a new Postgres connection costs ~5ms and a backend process. A pool of 20 connections serves hundreds of concurrent requests by reusing them. PgBouncer is a Postgres-specific connection pooler.

---

### Caching

**Cache hit ratio** — The percentage of reads served from cache without hitting the database. 80% hit ratio means only 20% of reads reach the DB. A cache is only worth its operational cost if the hit ratio is high enough to meaningfully reduce DB pressure.

**Eviction policy** — The rule for removing items when the cache is full. LRU (Least Recently Used): remove what hasn't been accessed longest. LFU (Least Frequently Used): remove least-accessed. TTL: remove when timer expires.

**Cache-aside (lazy loading)** — The application manages the cache. On miss: app reads from DB, populates cache, returns. On write: app writes to DB and invalidates (or updates) cache. Most common pattern; app has full control.

**Write-through** — Every write goes to both cache and DB. Cache always up to date. Writes are slightly slower. Reads always fast.

**Write-behind (write-back)** — Writes go to cache only. An async process flushes to DB. Extremely fast writes. Risk: data in cache-only state is lost if the cache crashes before the flush.

**TTL (Time to Live)** — How long a cached item remains valid. After it expires, the next read is a cache miss and fetches fresh data from the source.

**Cache stampede** — A popular cache key expires. Thousands of concurrent requests all miss at the same instant, all hit the database simultaneously. Mitigation: probabilistic early refresh (renew slightly before TTL expires) or fetch-locking (one thread fetches; others wait for it).

---

### Messaging

**Idempotency** — An operation that produces the same result no matter how many times it executes. Example: `SET balance = 1000` is idempotent; `INCREMENT balance BY 100` is not. Idempotent consumers can safely process duplicate messages without side effects.

**Dead Letter Queue (DLQ)** — A holding area for messages that repeatedly fail processing. After N failed attempts, the message moves to the DLQ instead of blocking the main queue. Allows manual inspection and replay.

**Partition key (Kafka)** — Determines which partition a message is routed to. Messages with the same partition key always go to the same partition, guaranteeing ordering within that key. Example: all events for order_id=123 on the same partition.

**Consumer group (Kafka)** — A set of consumers that collectively consume a topic. Each partition is read by exactly one consumer in the group. Multiple independent consumer groups can each read the full topic independently (no interference).

**Offset (Kafka)** — The sequential position of a message within a partition. Consumers commit their current offset. On restart, they resume from the last committed offset — enables exactly-once-style processing and replay.

---

### API & Web

**Idempotency key** — A client-generated unique ID sent with a POST request (e.g., payment). The server stores it; if the same key arrives again (network retry), the server returns the stored result without re-executing. Safe retries on non-idempotent operations.

**Cursor pagination** — "Next page" is identified by a cursor (last item's ID or timestamp), not an offset. Immune to insertions mid-pagination. Efficient on large tables — the DB can seek to the cursor position via index, not scan to offset.

**Webhook** — A reverse API call. Instead of polling an external service, you register a URL and the service POSTs to you when an event happens. Example: Razorpay POSTs to `/webhook/payment` when payment succeeds.

**Service mesh** — Infrastructure layer handling service-to-service communication (mTLS, retries, circuit breaking, observability) without application code changes. Implemented as sidecar proxies (Envoy). Examples: Istio, Linkerd.

---

### AI & ML Systems

**RAG (Retrieval-Augmented Generation)** — Retrieve relevant documents from a knowledge base first, then pass them as context before the model generates a response. Grounds the model in real content; dramatically reduces hallucination compared to generation from training data alone.

**Embedding** — A dense vector (list of numbers) representing the meaning of text. Sentences with similar meaning produce vectors that are numerically close together in vector space. Computed by an embedding model (OpenAI text-embedding-3-small, sentence-transformers, etc.).

**HNSW (Hierarchical Navigable Small World)** — A graph-based algorithm for fast approximate nearest-neighbor search. Builds a multi-layer graph connecting similar vectors. Search traverses from the top layer (coarse) to the bottom (fine), visiting only a small fraction of vectors. Finds nearest neighbors in milliseconds across millions of vectors.

**ANN (Approximate Nearest Neighbor)** — Finding vectors "close enough" to a query without checking every vector in the database. Trades a small accuracy loss for massive speed gain. Acceptable for RAG because a slightly imprecise match is almost always still relevant context.

**Grounding threshold** — A minimum similarity score below which the system refuses to call the model at all — enforced in application code, not as a model instruction. Deterministically prevents hallucination by ensuring the model is only called when relevant context exists.

**Agent (AI)** — A model in a loop that calls tools, reads results, and decides its next action autonomously until it reaches a conclusion. Unlike single prompt-response, an agent can do multiple rounds of reasoning and tool use.

**ReAct loop (Reasoning + Acting)** — The agent generates a thought (what to do), takes an action (tool call), observes the result, reasons again. Continues until the model decides it has sufficient information to answer.

**MCP (Model Context Protocol)** — A typed, schema-validated interface for exposing tools to AI models. Tool arguments are validated against a JSON schema before execution. Invalid arguments return an error the model can read and correct — unlike untyped function calls.

**SOUL.md** — A markdown file loaded as the system prompt that defines an agent's identity, personality, constraints, and hard rules. Separating identity from code means agent behaviour can change without a code deployment.

**Sovereign inference** — Running AI generation entirely on infrastructure you control, with no external API calls for the generation step. Guarantees data never leaves your servers. Implemented with local model servers (Ollama for dev, vLLM for production).

**GEPA (Generative Evaluation and Prompt Advancement)** — A self-improvement pipeline for agent identity (SOUL.md). Steps: eval current SOUL on a fixed test set → reflect on weakest cases → propose an improved SOUL → gate with Pareto criterion → require human approval → write with backup. Improves agent quality without touching code.

**Pareto gate** — A quality filter accepting a new system version only if it improves the average score AND no individual test case scores below a floor. Prevents "better on average but regressed on edge cases" — the most common failure mode of automated improvement loops.

**RAGAS** — A RAG evaluation framework. Measures faithfulness (are all claims in the model's answer supported by retrieved chunks?) and context precision (were the retrieved chunks actually relevant to the question?). Used to measure and improve RAG pipeline quality.

---

### Algorithms

**TF-IDF (Term Frequency-Inverse Document Frequency)** — A method to convert text into a vector by weighting words. Words that appear frequently in one document but rarely across all documents get high weight — they are characteristic of that document. Used in search and plagiarism detection.

**Cosine similarity** — Measures the angle between two vectors. Range: -1 to 1. Score of 1 = identical direction (same meaning). Score of 0 = orthogonal (unrelated). Used to compare TF-IDF vectors for similarity search and plagiarism detection.

**Levenshtein distance** — The minimum number of single-character edits (insert, delete, replace) to transform one string into another. Computed with an O(m×n) dynamic programming matrix. Used for fuzzy matching and spell checking.

**Consistent hashing** — A technique where hash keys are mapped onto a ring. Nodes own arcs of the ring. Adding or removing a node only affects keys that were owned by that node — not all keys. Minimises reshuffling in distributed caches and databases.

**Token bucket** — A rate limiting algorithm. A bucket holds up to N tokens; each request consumes one; tokens refill at a fixed rate R per second. Allows bursts up to N while enforcing an average rate of R. The most common rate limiting algorithm in production.

**LSM tree compaction** — SSTables (sorted string table files) accumulate on disk from MemTable flushes. Compaction merges multiple SSTables into one, removes deleted/overwritten entries, and keeps disk usage bounded. The background cost of write-optimised storage.

---

### Architecture & Networking

**Forward proxy** — A proxy that sits in front of clients. Clients route outbound traffic through it. The destination server sees the proxy's IP, not the client's. Used for content filtering, anonymity, and caching in corporate environments.

**Reverse proxy** — A proxy that sits in front of servers. Clients call the proxy; it forwards to the appropriate backend. The backend sees the proxy's IP. Handles SSL termination, load balancing, caching, and WAF. Every production web server uses one.

**Monolith** — A single deployable artifact containing all application functionality. One codebase, one process, one database. Fast to develop early. Harder to scale and deploy independently at large team sizes.

**Modular monolith** — A monolith internally structured into enforced modules with clear boundaries. Modules interact only via defined APIs. Easier to split into microservices later when boundaries are already clean.

**Microservices** — An architecture where each business capability is a separate, independently deployable service with its own codebase and database. Enables team autonomy and independent scaling. Adds distributed systems complexity.

**Distributed monolith** — An anti-pattern: microservices that are so tightly coupled they must deploy together. All the operational cost of microservices with none of the independence. Caused by shared databases, synchronous call chains, or no clear domain ownership.

**Event-Driven Architecture (EDA)** — Components communicate via events through a broker rather than direct calls. Producer emits an event without knowing who consumes it. Consumers react independently. Enables loose coupling and parallel processing.

**BFF (Backend for Frontend)** — A separate backend service tailored to a specific client type (mobile, web, third-party). Aggregates microservice calls, shapes responses for the client, handles client-specific auth. Eliminates over-fetching without polluting the core API.

**DNS (Domain Name System)** — A globally distributed system that translates domain names (api.example.com) to IP addresses. Hierarchical: root servers → TLD servers → authoritative nameservers. Results cached by TTL.

**DNS A record** — Maps a domain to an IPv4 address directly.

**DNS CNAME record** — Maps a domain to another domain (alias). The final resolution is the target domain's A record.

**TTL (DNS Time To Live)** — How long DNS resolvers cache a response before re-querying. Low TTL = fast failover, more DNS traffic. High TTL = less traffic, slower propagation.

**Geohashing** — Encoding a geographic coordinate (lat, lon) as a short string. Nearby locations share a common string prefix, enabling fast proximity lookups via a single database range scan on the prefix.

**Quadtree** — A spatial data structure where each node represents a rectangular region with exactly 4 children (NW, NE, SW, SE). Subdivides until each leaf has at most N points. Adapts to density — small cells where data is dense, large cells where sparse.

**RTO (Recovery Time Objective)** — How long the system can be down before business damage becomes unacceptable. The target for how quickly to restore service after a failure.

**RPO (Recovery Point Objective)** — How much data loss is acceptable. The target for how old the most recent backup or replica can be at the time of failure.

**Pilot light** — A DR strategy where only the minimum core of the system runs continuously (usually the database replica). Application servers are off but can be started quickly. Faster than cold backup-restore; cheaper than warm standby.

**Warm standby** — A DR strategy where a scaled-down replica of full production runs continuously. On failure, scale up. RTO measured in minutes.

**Virtual Machine (VM)** — A software emulation of a physical computer. Runs a full OS with its own kernel. Strong isolation. GBs of disk, minutes to start. Managed by a hypervisor.

**Container** — A lightweight process-level isolation using the host OS kernel. Shares kernel with host. MBs of image, seconds to start. Weaker isolation than VM. Docker is the dominant container runtime. Orchestrated by Kubernetes.

**Hypervisor** — Software that creates and manages virtual machines. Type 1 (bare-metal): runs directly on hardware. Type 2 (hosted): runs as an app on a host OS.

**Database Federation** — Splitting a single database into multiple databases by functional domain (users DB, orders DB, products DB). Different from sharding — splits by table function, not by row. Each domain scales independently.

**Materialized view** — A pre-computed query result stored as a physical table. Unlike a regular view, it is computed once and refreshed on schedule. Eliminates expensive real-time join/aggregation queries at the cost of staleness.

**N+1 query problem** — A loop that fetches N parent records in one query, then issues N additional queries for related records. Total: N+1 queries. Fix: eager loading (JOIN) or batch loading (WHERE id IN (...)).

**Master-Master replication** — Both database nodes accept writes. Changes are replicated to each other. Enables high write availability and geographic distribution. Requires conflict resolution when concurrent writes target the same row.

**MapReduce** — A programming model for parallel batch processing at scale. Map phase: transform each input record into (key, value) pairs, independently and in parallel. Reduce phase: aggregate all values for each key. Framework handles data splitting, scheduling, and fault tolerance. Used in Hadoop. Spark replaced it for most use cases with in-memory processing.

**Snowflake ID** — A 64-bit globally unique, time-ordered ID. Structure: 41 bits timestamp + 10 bits machine ID + 12 bits sequence. No coordination required (machine ID assigned at startup). Generates up to 4,096 unique IDs per millisecond per machine. Sortable — higher ID = newer entry. Used by Twitter, Discord, Instagram.

**Bloom filter** — A space-efficient probabilistic data structure for set membership. Returns "definitely not in set" or "probably in set" (small false positive rate, never false negatives). Used for URL deduplication in web crawlers, cache-miss optimization (don't query DB for keys that definitely don't exist).

**TCP (Transmission Control Protocol)** — Connection-oriented transport protocol. Guarantees delivery, ordering, and flow control. Higher overhead than UDP. Used when all data must arrive: HTTP, databases, file transfer.

**UDP (User Datagram Protocol)** — Connectionless transport protocol. Best-effort delivery — packets may be dropped or arrive out of order. Lower overhead, lower latency. Used when speed beats guaranteed delivery: video calls, live game state, DNS.

**OLTP (Online Transaction Processing)** — Databases optimised for high-throughput, low-latency transactional operations — inserts, updates, point lookups. Row-oriented storage. Examples: Postgres, MySQL. The primary database for most web applications.

**OLAP (Online Analytical Processing)** — Databases optimised for large aggregation queries across many rows (e.g., "total revenue by region last year"). Column-oriented storage — reads only the columns needed, compresses well, fast scans. Examples: ClickHouse, BigQuery, Redshift. Never the primary transactional database.

**Refresh-ahead** — A cache update strategy where the cache proactively refreshes entries before they expire, based on predicted access patterns. The client always gets a cache hit (no miss latency). Requires the system to know which keys will be requested next. Fails if prediction accuracy is low (useless refreshes waste compute).

### Infrastructure & Deployment

**Load balancer** — A server (or cluster) that distributes incoming requests across a pool of backend servers. Performs health checks and removes unhealthy servers from rotation automatically. Layer 4 operates at TCP level; Layer 7 at HTTP level with content-aware routing.

**Sticky session (session affinity)** — Routing all requests from a given client to the same backend server. Required when in-process session state exists. Anti-pattern in horizontally scaled systems — prefer external session stores (Redis) so any server can handle any request.

**Horizontal scaling** — Adding more machines to handle more load. Requires stateless services. Enables high availability through redundancy. The correct response to sustained throughput bottlenecks.

**Vertical scaling** — Upgrading to a more powerful machine (more CPU, RAM). Simple, no code changes. Has a hard ceiling. Single point of failure. The correct first response to latency bottlenecks.

**Service discovery** — The mechanism by which services in a cluster find each other's current addresses. DNS-based (Kubernetes Services), client-side (Consul, Eureka), or server-side (load balancer + registry). Necessary because IP addresses change as instances are added or replaced.

**Health check** — An endpoint or probe that reports whether a service instance is functioning correctly. Liveness (`/healthz`) checks if the process is alive; readiness (`/readyz`) checks if it can currently serve traffic. Used by load balancers and Kubernetes to route traffic only to healthy instances.

**Blue-green deployment** — Running two identical production environments. Traffic switches from Blue (current) to Green (new) at the load balancer. Instant rollback by switching back. Requires 2× infrastructure during the transition.

**Canary deployment** — Routing a small percentage (1–5%) of traffic to a new version. Monitoring confirms it's healthy before expanding to 100%. Limits blast radius of a bad deploy to a small fraction of users.

**Feature flag** — A runtime toggle that enables or disables code paths without a new deployment. Decouples deploy from release. Enables gradual rollouts, A/B testing, and instant kill-switch for problematic features.

**Presigned URL** — A time-limited URL generated by the server that grants the holder temporary permission to access a specific S3 object directly. The client downloads from S3 without the request passing through the application server.

**Multipart upload** — Uploading a large file by splitting it into parts and uploading concurrently. Enables parallel upload (faster), resumable transfers (restart only failed parts), and bypasses single-request size limits. Standard for files > 100MB.

**Active-Active** — All regions/nodes accept writes simultaneously. Highest availability and write throughput. Requires conflict resolution strategy for concurrent writes to the same data.

**Active-Passive** — One region/node is primary (handles writes). Others are replicas (read-only standby). Simpler consistency model. Failover requires promoting a replica, which has downtime measured in minutes.

**Backpressure** — A mechanism for a consumer to signal a producer to slow down when it can't keep up. Without it, a fast producer overwhelms a slow consumer. Implemented via bounded queues, consumer autoscaling, producer rate limiting, or load shedding.

**Schema registry** — A central service that stores and versions message schemas (Avro, Protobuf) for event streams. Enforces compatibility rules before a new schema is published. Prevents producers and consumers from silently breaking each other on schema changes.

**Two-phase commit (2PC)** — A distributed transaction protocol ensuring all-or-nothing commits across multiple systems. Phase 1: coordinator asks all participants "can you commit?" Phase 2: if all say yes, commit; otherwise abort. Blocking — if the coordinator fails after Phase 1, participants hold locks indefinitely.

**Saga** — An alternative to 2PC for distributed transactions. A sequence of local transactions where each step publishes an event; failure triggers compensating transactions. Non-blocking and highly available; accepts eventual consistency instead of true atomicity.

**Secrets manager** — A service for securely storing, rotating, and auditing access to sensitive credentials (DB passwords, API keys, certificates). Avoids hardcoding or storing secrets in environment files. Examples: AWS Secrets Manager, HashiCorp Vault.

**Dynamic secrets** — Credentials generated on-demand with a short TTL (e.g., a DB username/password created by Vault for one app instance for 1 hour, then revoked). Compromise is limited to one instance and one TTL window.

**Composite index** — A database index on multiple columns. Used by queries that filter on the leftmost prefix of the indexed columns. `INDEX(user_id, created_at)` speeds up queries filtering on `user_id` or `user_id + created_at`, but not `created_at` alone.

**Covering index** — An index that contains all columns needed by a query, so the query can be answered from the index without reading table rows. Eliminates I/O for the data rows, giving index-only scan performance.

**Partial index** — An index that only covers rows matching a WHERE condition. Smaller, faster, cheaper than a full-table index. Ideal for low-selectivity columns where only a subset of rows is ever queried (`WHERE status = 'PENDING'`).

---

*Print this. Read it before every system design interview. The framework in Part 1 tells you how to run the room. The patterns in Part 3 give you the vocabulary. The proof points in Part 7 give you the credibility.*
