# Swanand Kadam — Interview Prep System

**How to use this file:**
Upload or paste this entire document into any AI (Claude, ChatGPT, Gemini). Then say: "Start the interview session." The AI will act as a senior interviewer, drill Swanand on his own projects, and bridge into broader system design questions.

**What this covers:**
1. All projects — architecture, trade-offs, AI features
2. Targeted question banks per project
3. System design bridges — from his actual decisions to canonical SD patterns (design YouTube, design Twitter, design a booking system)

---

## PART 1 — WHO IS SWANAND

Senior Full-Stack and AI Engineer, 6+ years. Specialises in high-concurrency platforms, LLM-powered features, RAG pipelines, and local-first agent systems. Built things from zero to production in small teams — first engineer under a CTO, co-founder, open-source creator.

**Core positioning:** Systems that scale. AI that ships. Code that matters.

**Tech surface:** Python, Django, FastAPI, React 18 + TypeScript, PostgreSQL, pgvector, AWS Lambda, SQS, SSE, LangGraph, Hermes runtime, MCP, Ollama, Docker, Redis (prior), Celery (prior), Vue, Node.js, npm packaging.

---

## PART 2 — PROJECT CONTEXT (embedded, self-contained)

### PROJECT 1: CodeMas — Real-Time Coding Assessment Platform

**Company:** Masai School. **Role:** First engineer under CTO. **Scale:** 10,000 concurrent students.

**Business impact:** Each cohort = 400 students × ₹3L = $2M+ in revenue unlocked per cohort. Plagiarism detection: 5% → 95% (19× lift). AI features reduced exam creation effort by 80%.

---

**Submission pipeline (current architecture):**
```
Student → POST /submit → Django API
  → SELECT FOR UPDATE (attempt gate)
  → INSERT Submission (Postgres)
  → Enqueue to SQS FIFO
  → 201 response immediately

Browser opens SSE stream (/results/{id})

SQS → Lambda invocation (one per submission)
  → Lambda executes student code (isolated, 15s timeout)
  → Writes result to Postgres

SSE endpoint polls Postgres every 500ms
  → Status change detected → push to browser → stream closes
```

**Architecture migration:** Originally Redis LPUSH → Celery BRPOP → Docker container on a persistent host. Migrated to SQS FIFO → Lambda. Trade-offs:
- Lost: Redis pub/sub for real-time result push (replaced by Postgres polling, 500ms acceptable)
- Gained: true per-invocation isolation (no shared host), zero worker management, elastic scale at deadline bursts
- Docker on a persistent host required managing worker processes, restarts, and container lifecycle. Lambda is fully ephemeral — each submission gets a clean slate with a hard 15-second timeout.

**Key decisions and why:**
- **SQS FIFO over Redis list:** Decouples execution from API. During deadline (400 students submit in 30s), SQS absorbs the burst. Each exam gets its own queue for isolation. At-least-once delivery with deduplication ID prevents double execution.
- **Lambda as sandbox:** Per-invocation isolation — no shared memory, no shared filesystem, hard timeout enforced by AWS. No need to run Docker on a persistent host or manage container cleanup.
- **SSE over WebSockets:** Code execution is unidirectional (server pushes once). SSE: native browser reconnection via EventSource, works through HTTP/1.1 proxies without upgrade negotiation, Lambda-compatible. WebSockets need persistent connection management — wrong tool here.
- **SELECT FOR UPDATE:** Before accepting a submission, locks the attempt row. Prevents two simultaneous submissions (two browser tabs at exam deadline) from both incrementing the attempt counter past the limit. Without this, a race condition allows unlimited attempts.
- **201 immediate response:** API returns 201 as soon as the submission is queued. Never waits for execution. Keeps API latency flat at <100ms regardless of code complexity.

---

**5 AI-powered features (not agents — single-shot LLM calls, event-triggered):**

All are async, triggered by events, completely off the submission critical path. GPT enrichment appears after the student already has their execution result. Zero latency added.

| Feature | Trigger | What it does |
|---|---|---|
| Rubric Scoring | Every failed submission | GPT grades on correctness, code quality, approach, edge cases (0–2 each) with justifications |
| Socratic Hint | Failed submission | GPT generates a pedagogical nudge — points at the concept, never reveals the answer |
| Exam Generator | Trainer request | GPT drafts full exam (questions + test cases) from topic + difficulty; human reviews before persistence |
| Trainer Narrative | Cohort summary request | GPT generates per-student and cohort-level performance summaries |
| Plagiarism Engine | Exam close (pre_save signal) | Two-phase: behavioral + TF-IDF (not LLM-based) |

All 5 share a single GPT-4o-mini endpoint, are idempotent, write to Postgres only.

---

**Plagiarism system — the reframing:**

Original question: "Are two submissions similar?" → O(N²) comparisons at 2K submissions = 4M pairs.
Reframed question: "Did this student write this?" → behavioral evidence first, similarity second.

**Phase 1 — Behavioural (O(N), synchronous at exam close):**
Scores every submission independently:
- Paste ratio: `paste_chars / total_chars` (weight 0.40)
- Speed vs difficulty baseline: `time_taken / expected_time_for_difficulty` (weight 0.30)
- Tab switches: normalised count (weight 0.15)
- Attempt surprise: how late the passing attempt arrived (weight 0.15)
Risk > 0.2 → BehaviouralFlag written to DB.

**Phase 2 — Similarity (O(K×N), queued after Phase 1):**
Only HIGH-confidence suspects (K) are compared against the full cohort (N). TF-IDF cosine similarity. Pairs above 0.80 similarity → flagged.

Why TF-IDF over CodeBERT/MOSS/AST: no GPU required, language-agnostic (works across Python/JS/Java/C++), interpretable output for trainers, accurate enough at 2K submissions.

---

### PROJECT 2: the01.dev — AI-Powered Ed-Tech Platform

**Role:** Co-founder. Led engineering (2-person team). **Stack:** React 18 + TypeScript + Vite, FastAPI (Python), pgvector (HNSW), LangGraph, GPT-4.1-mini, text-embedding-3-small, Razorpay, Firebase.

---

**RAG course assistant pipeline:**

*Indexing (on startup):*
`seed_transcripts.json` → `chunk_courses()` (180-word windows, 35-word overlap, step=145) → `embed_many()` batch → pgvector HNSW index (PostgreSQL)

*Query:*
User question → `embed(question)` → HNSW ANN search (top 8 candidates) → `_rank_matches()`: semantic score (weight 1.0) + lexical overlap/Jaccard (stop-word filtered) → `max(semantic, lexical)` per chunk → top 5 → score gate ≥ 0.15

*Answer:*
Context blocks `[Course | Lecture | MM:SS-MM:SS]\ntext` → GPT-4.1-mini → `_to_source()` → `Source {timestamp, snippet, score}` → React SourceCard deep-links to video player at `?t=seconds`

**Key decisions:**
- **Hybrid cosine + BM25 (Jaccard):** Pure semantic misses exact keyword matches (function names, error messages). The `max()` combination: either signal can promote a chunk. Recall improves without sacrificing precision.
- **Score gate ≥ 0.15:** Below threshold → returns canned "not enough context" message. LLM never called on ambiguous retrieval. Saves cost, prevents hallucination. The gate is the cheapest safety mechanism.
- **pgvector HNSW:** Approximate nearest neighbour — O(log N) vs O(N×D) for exact scan. Scales without full-table scan as transcript corpus grows.
- **Source timestamps:** Not just "this lecture" — answers include `?t=342` deep-links. Users jump to the exact second in the video.

---

**LangGraph multi-agent content pipeline (PDF → Lesson):**

Planner agent → decomposes PDF into lesson structure → Human approval (HITL suspend/resume) → Executor agents run in parallel (one per section) → Supervisor reviews each output → routes failed sections back to worker → assembles final lesson.

~67% reduction in unnecessary LLM calls via on-demand generation (only generate what gets approved).

---

**Payments:** Razorpay server-side order creation (secret never leaves FastAPI). Client gets `order_id` + public `key_id`. Prices stored in paise (integer) — no float precision issues. Atomic access provisioning on payment success.

---

### PROJECT 3: Munshi — Sovereign GST Agent

**Built for:** Bharatvarsh Arts (₹5Cr/$500K+ revenue business). Replaces Excel-based GST workflows with plain-English queries. **Key constraint:** financial data must never leave the machine.

**Stack:** Hermes runtime (Nous Research), MCP tools (custom), FastAPI, Ollama (local inference), Python Decimal, Docker, SQLite.

---

**Architecture (fully local):**
```
Owner query → Hermes runtime → MCP tool selection
  → Fetch GSTR-2B / Read invoices / Compute tax / Write audit log
  → Fuzzy match engine (GSTIN prefix + name normalisation + amount tolerance ±2%)
  → Ambiguous cases → model judges (plain-English reasoning)
  → Decimal computation (exact rupees, never float)
  → Audit trail entry
  → Human approval gate → consequential action
```

Everything runs on the local machine. Ollama runs model inference locally. No data transmitted over any network. Privacy by architecture, not policy.

**Key decisions:**
- **Local-first:** GST data is sensitive. Cloud APIs create data residency risk. Local inference (Ollama) means the model is also local — no prompt data transmitted.
- **Fuzzy match first, model second:** Deterministic matching handles the clear cases fast. Model only sees ambiguous ones — keeps cost zero for 80%+ of matches.
- **Python Decimal for all tax arithmetic:** IEEE 754 float accumulates errors across thousands of invoices. At ₹5Cr revenue scale, that error compounds into wrong ITC claims or incorrect filings. Decimal eliminates this class of bug entirely.
- **HITL before consequential actions:** Filing, reconciliation verdicts, ITC claims all require explicit owner approval. Model can judge, but cannot act. Audit trail in plain English so a non-technical owner can verify every decision.

---

### PROJECT 4: Trade Compliance Researcher — Multi-Agent System

**Stack:** Hermes runtime, MCP tools, Ollama (default), Docker Compose, Python.

**Architecture:**
Researcher agent → gathers regulatory data iteratively via MCP tools (regulation search, tariff DB, document fetcher) → passes findings to Writer agent → Writer synthesises structured compliance report with citations.

Both agents share conversation memory across turns via Hermes. SOUL.md declares agent identity, tone, constraints. `config.yaml` declares tools and model. One config line to swap Ollama for OpenAI or any OpenAI-compatible endpoint — zero code changes.

---

### PROJECT 5: Kalaam — India's First Hindi Programming Language

**Published:** npm package `kalaam` v2.3.3. Frontend: kalaamlang.in. **Users:** 500+ monthly. **Recognition:** TEDx Bangalore, IEEE Nagpur.

**5-phase interpreter pipeline (pure JavaScript, zero runtime deps):**

1. **Cleaning:** `earlyCleaning()` + keyword substitution. Language-specific keywords (Hindi: "यदि", Marathi: "जर") → normalised tokens. ONLY phase that knows about language. Everything after is language-agnostic.
2. **Scanning:** Character-level scan → `cleaned_sourcedata[]`
3. **Tokenizing:** `cleaned_sourcedata[]` → typed `tokens[]` via 20 `Push*` functions + TypeChecking pass
4. **Interpretation:** Walk `tokens[]`, execute against `memory{}`, build `ExecutionStack[]`
5. **Output:** Return `kalaam{}` object: `{ output, ExecutionStack, isError, TimeTaken }`

**Adding a new language = 1 keyword map entry, zero parser changes.** The cleaning phase handles substitution before parsing. All 5 supported languages (Hindi, Marathi, Bengali, Telugu, Odia) required no parser modification.

**ExecutionStack (Learning Mode):** Every operation appends to `ExecutionStack[]`. UI replays it line-by-line in the student's language — shows what each line does and how the interpreter evaluates it. No teacher required.

**Offline PWA:** Service-worker cached. After first load, runs with zero connectivity on a budget Android phone. This is the physical reach requirement — tier-3 cities have intermittent internet.

Public API: `Compile(sourcecode, languageKeywords)` → `{ output, ExecutionStack, isError, TimeTaken }`

---

### PROJECT 6: Bharatvarsh.art — D2C E-Commerce

Indian cultural wall art. Built and led engineering end-to-end. ₹2Cr+ in revenue. Key insight from this: the printer (supplier/infrastructure) made more money than the poster seller — infrastructure businesses are more defensible than application businesses.

---

### PROJECT 7: stringy-core — JavaScript String Utility Library

npm package. 50+ pure functions, 9 modules, zero runtime deps, ESM. Two export patterns: named tree-shakeable imports + `_s` namespace. 19 forks. Modules include: textCaseManipulation, textFormatting (all via Intl API), textMaskingAndSecurity, textMetadataAndExtraction (15 regex extractors), textSpecializedOperations (levenshteinDistance via DP matrix).

---

## PART 3 — INTERVIEWER INSTRUCTIONS

You are a senior staff engineer interviewing Swanand. You have read all of Part 2 above. You know his systems in detail.

**Rules:**
- Ask one question at a time. Wait for his answer before the next.
- If the answer is correct but shallow, say "go deeper" or ask a specific follow-up.
- If he gives a number (19×, $2M, 500ms), ask "how did you measure that?"
- If he names a technology, ask "why that one and not X?" where X is the obvious alternative.
- If he gets stuck, don't give the answer. Ask "what would you look at first?" or "what breaks before you get there?"
- After 3-4 questions on one topic, pivot: either a what-if extension or a system design bridge.
- End every session with: "You were sharp on [X]. You need more work on [Y]. Review [Z] before your next interview."

**At the start of every session, ask:**
> "Which mode?
> 1. Project drill — pick a project, I go deep on trade-offs and architecture
> 2. AI systems — tool calling, RAG, observability, evaluation, agent reliability
> 3. What-if scenarios — I take your existing projects and add new constraints
> 4. System design bridges — from your projects to canonical SD problems
> 5. Full mock — I pick everything, you answer cold"

---

## PART 4 — QUESTION BANK

### CodeMas Trade-offs

- Walk me through submission pipeline from click to result.
- Why SQS FIFO over Redis list or a DB-backed queue?
- What did you give up by choosing Lambda over Docker on a persistent host?
- Lambda has cold starts. How does that affect a student submitting at an exam deadline?
- Your SSE polls Postgres every 500ms. At 10K connections that's 20K reads/sec on Postgres. How did you handle that load?
- Why SSE and not WebSockets?
- What does SELECT FOR UPDATE prevent exactly, and when does it actually matter in your system?
- You said 201 immediately. What happens to the submission if the Lambda invocation fails after you've returned 201?
- Walk me through the plagiarism system — why two phases?
- Why behavioral signals first and not similarity first?
- Why TF-IDF over MOSS, CodeBERT, or AST comparison?
- A student copies from GitHub and renames all variables. Your TF-IDF catches that?
- What's your false positive rate on plagiarism? How would you know if it's too high?
- If GPT-4o-mini is down, what happens to students mid-exam?
- You say rubric scoring is off the critical path. How do you technically guarantee that?
- How do you know if the rubric scoring is actually accurate? What's your eval loop?

### AI Systems (tool calling, RAG, observability)

- Tool calling fails silently — wrong tool selected, bad arguments, hallucinated parameters. How do you detect that in production?
- What's your strategy for mitigating tool call failures? Retry logic? Schema validation? Fallback?
- What observability do you have on your LLM features — what can you see when something goes wrong?
- How do you prevent prompt injection when user queries go into your RAG context?
- Your score gate is 0.15. How did you pick that number? How would you know if it's wrong?
- Walk me through your hybrid retrieval — what does BM25 catch that cosine misses?
- If your vector index grows 10× — what's the retrieval latency impact with HNSW?
- How do you evaluate RAG quality? What metrics? RAGAS? Human eval?
- The LangGraph supervisor routes failed executor output back to workers. How does it decide what "failed" means?
- How does human-in-the-loop suspend/resume work in LangGraph technically?
- Munshi: the model judges ambiguous invoice matches. What if the model is wrong and the owner approves? Who's accountable?
- How does your audit trail map from raw tool call arguments/results to plain English?

### Kalaam

- Walk me through all 5 interpreter phases.
- Why do keyword substitution in Phase 1 and not in the parser?
- Show me in words what it takes to add a new language.
- How does ExecutionStack get built during interpretation?
- 90-95% test coverage — what are you NOT testing and why?
- What's the hardest bug you fixed in the interpreter?

---

## PART 5 — WHAT-IF SCENARIOS

### CodeMas → SaaS

- CodeMas is now a SaaS serving 500 clients. Each client has their own students. How does your DB schema change?
- How do you handle tenant isolation for code execution? Can one client's Lambda affect another's?
- Your plagiarism system compares within an exam. In SaaS, do clients share a plagiarism pool or not?
- How do you price this? Per submission? Per student? What's your cost model per unit?
- One client runs 10K exams, another runs 10. How does your Lambda concurrency quota management work?

### CodeMas → 100K concurrent users

- What breaks first when you go from 10K to 100K concurrent users?
- Postgres polling at 500ms per SSE connection breaks at 100K connections. What do you replace it with?
- SQS + Lambda can handle the burst, but you hit Lambda concurrency limits per AWS account. What's your mitigation?
- Your SELECT FOR UPDATE becomes a bottleneck at 100K. What's the alternative?

### Munshi → Multi-user SaaS

- You built for one business, fully local. Now 50 accounting firms want it. What changes architecturally?
- Does local-first still work for a multi-user product? What's the alternative and what do you lose?
- Your HITL is one owner approving. In a firm with 5 accountants, how does approval routing work?

### RAG → Production Scale

- Your pgvector store works now. At 10K courses with full transcripts, what's your retrieval latency?
- You index on startup. If 10 new courses are added daily, how does re-indexing work without downtime?
- Your score gate returns a canned message below 0.15. A user asks a valid question your transcripts don't cover — is a canned message the right UX? What else could you do?

### CodeMas → Real-Time Collaborative Coding

- Students can now pair-program in the exam. Two browsers edit the same code. How do you handle concurrent edits?
- What protocol do you use — OT, CRDTs, or something else? Why?

---

## PART 6 — SYSTEM DESIGN BRIDGES

*Each bridge starts from a real decision in one of Swanand's projects, then extends it to a canonical system design problem. The goal: practice recognising patterns across domains.*

---

### Bridge 1: CodeMas SQS+Lambda → Design a Video Processing Pipeline (YouTube)

**Your decision:** Student submits code → SQS queue → Lambda processes → result stored → browser polls.

**The pattern:** Async job processing with durable queue, isolated worker, and status polling.

**Bridge question:**
> "You built exactly this pattern for code execution. Now design YouTube's video processing pipeline. A user uploads a video — walk me through what happens before it's watchable in multiple resolutions."

*What this covers:* blob storage (S3), job queue (SQS/Kafka), transcoding workers (EC2/Lambda), CDN delivery, progress polling vs webhook, idempotency, dead letter queues, retry with exponential backoff.

**Follow-ups:**
- Your Lambda has a 15s timeout. A 4K video transcoding takes 20 minutes. What's your architecture now?
- How do you handle a transcoding worker crash mid-job? What state do you need to persist?
- One queue or multiple queues for different video qualities?

---

### Bridge 2: CodeMas SSE → Design a Real-Time Notification System (Twitter/X)

**Your decision:** You chose SSE over WebSockets for unidirectional, server-pushed results.

**The pattern:** Server-push for live updates. When does each protocol win?

**Bridge question:**
> "Design Twitter's real-time feed — when you follow someone and they tweet, your feed updates within a second. Walk me through the architecture."

*What this covers:* fan-out on write vs fan-out on read, pub/sub (Redis/Kafka), WebSockets vs SSE vs polling, connection management at scale, celebrity problem (accounts with 50M followers), timeline storage.

**Follow-ups:**
- SSE at 10M concurrent users. Each connection is held open on your server. What's the cost? What's your alternative?
- A celebrity tweets. You need to fan-out to 50M followers. How fast? What's your strategy?
- How does Twitter decide when to fan-out on write vs fan-out on read?

---

### Bridge 3: CodeMas Plagiarism O(K×N) → Design a Duplicate Detection System

**Your decision:** Don't compare all pairs O(N²). Score each item independently first to find suspects K, then compare suspects against full set O(K×N).

**The pattern:** Candidate generation before expensive comparison.

**Bridge question:**
> "Design a system that detects duplicate listings on an e-commerce platform. New listings come in at 10K/day. Existing catalogue has 50M items. How do you find near-duplicates at scale?"

*What this covers:* LSH (Locality Sensitive Hashing), MinHash for Jaccard similarity, candidate generation, blocking/bucketing, approximate vs exact matching, offline batch vs online real-time.

**Follow-ups:**
- Your TF-IDF at 50M products is O(N) per new listing. That's too slow. What's the alternative?
- How does Airbnb detect duplicate property listings? What signals do they use beyond text?
- What's your false positive rate target and how does it change your architecture?

---

### Bridge 4: SELECT FOR UPDATE → Design a Ticket Booking System (BookMyShow)

**Your decision:** Lock the attempt row before incrementing to prevent two simultaneous submissions both succeeding.

**The pattern:** Pessimistic locking to prevent race conditions on shared mutable state.

**Bridge question:**
> "Design BookMyShow's seat booking flow. Two users click 'Book Seat A1' simultaneously. Exactly one should succeed. Walk me through the architecture."

*What this covers:* pessimistic vs optimistic locking, SELECT FOR UPDATE, distributed locks (Redis SETNX), two-phase commit, compensation patterns (SAGA), temporary reservation with TTL, at-most-once vs at-least-once.

**Follow-ups:**
- SELECT FOR UPDATE works in a single Postgres instance. You've sharded your DB across 5 nodes. How do you prevent double-booking now?
- You reserve a seat for 10 minutes while the user pays. Payment fails. How do you release the reservation automatically?
- 1M users trying to book a Coldplay concert simultaneously. SELECT FOR UPDATE creates a thundering herd. What's your mitigation?

---

### Bridge 5: RAG Pipeline → Design an Enterprise Search System

**Your decision:** Embed transcripts → pgvector HNSW → hybrid cosine + BM25 retrieval → score gate → LLM answer.

**The pattern:** Multi-stage retrieval: candidate generation → re-ranking → answer synthesis.

**Bridge question:**
> "Design Notion's AI Q&A — a user asks 'what did we decide about the pricing strategy last quarter?' and gets an answer grounded in their workspace docs. Walk me through the full system."

*What this covers:* chunking strategies, embedding models, vector DBs (pgvector/Pinecone/Weaviate), hybrid retrieval, re-ranking (cross-encoders), context window management, hallucination prevention, freshness/staleness.

**Follow-ups:**
- A document is updated after it was indexed. How do you handle stale embeddings?
- Your BM25 is good for exact keyword matches. The user asks in different words than the doc uses. How does your hybrid handle that?
- Your score gate filters low-confidence retrievals. But for enterprise search, "I don't know" might be worse than a low-confidence answer. How do you adjust?
- How do you handle a workspace with 10M documents where HNSW index rebuild takes 6 hours?

---

### Bridge 6: LangGraph Multi-Agent → Design an AI Coding Assistant (GitHub Copilot Workspace)

**Your decision:** Planner decomposes the task, executor agents run sections in parallel, supervisor routes failures back to workers, HITL for approval.

**The pattern:** Orchestrator-worker with supervision, human checkpoints, and compensation.

**Bridge question:**
> "Design GitHub Copilot Workspace — a developer says 'add Stripe payments to this codebase.' The AI reads the code, plans the changes, implements them, runs tests, and submits a PR. Walk me through the agent architecture."

*What this covers:* planning vs execution separation, tool use (code read, write, run tests), parallelism, failure recovery, human checkpoints, sandboxing for code execution, agent memory.

**Follow-ups:**
- The executor writes code that breaks existing tests. How does the supervisor handle that?
- Your planner produces a bad decomposition that only reveals itself 10 steps in. How do you recover?
- How do you prevent the agent from deleting production data when it has write access to the repo?

---

### Bridge 7: Munshi Local-First → Design a Privacy-Preserving Analytics System

**Your decision:** All computation stays on the local machine. Model inference is local (Ollama). No data over the network.

**The pattern:** Edge computation, federated learning, zero-trust data architecture.

**Bridge question:**
> "Design Apple's on-device intelligence — Siri can answer questions about your emails and messages, but Apple promises they never see your data. How do you architect that?"

*What this covers:* on-device ML, federated learning, differential privacy, model distillation (large cloud model → small on-device model), secure enclaves, what can and can't be done locally.

**Follow-ups:**
- Your on-device model is smaller and less capable. How do you decide what stays on-device and what goes to the cloud?
- A user's on-device model gets out of date. How do you push model updates without exposing their data?
- Munshi's tax computations are deterministic Python. What happens when you need to compute something the local model can't handle?

---

### Bridge 8: Kalaam Interpreter → Design a Code Execution Platform (LeetCode)

**Your decision:** Browser-based interpreter, service-worker cached, runs offline, each language = one keyword map.

**The pattern:** Secure, isolated, multi-language code execution at scale.

**Bridge question:**
> "Design LeetCode's code execution backend. A user submits Python code, it runs against hidden test cases, and returns pass/fail with runtime and memory in under 5 seconds. 100K submissions/day across 20 languages."

*What this covers:* sandboxing strategies (gVisor, nsjail, Firecracker, WASM, Lambda), resource limits (CPU/memory/time), language runtimes, test case isolation, queue management, result storage.

**Follow-ups:**
- A student submits `while True: pass`. How do you prevent it from running forever?
- Your Lambda approach costs $X per execution. At 100K submissions/day, what's your monthly bill and how do you optimise it?
- How do you prevent one submission from reading another user's test cases from the filesystem?
- LeetCode needs to support a new language (Rust). What does that change in your architecture?

---

### Bridge 9: CodeMas Event-Driven → Design an Order Processing System (Flipkart)

**Your decision:** Submission event → SQS → Lambda (async processing) → status written to DB → polling for result.

**The pattern:** Event-driven processing with eventual consistency and status polling.

**Bridge question:**
> "Design Flipkart's order processing pipeline. User clicks 'Buy Now' — walk me through what happens across payment, inventory, warehouse, and delivery."

*What this covers:* SAGA pattern, choreography vs orchestration, event sourcing, idempotency, compensation transactions, distributed transactions (two-phase commit vs SAGA), outbox pattern.

**Follow-ups:**
- Payment succeeds but inventory reservation fails. How do you refund? That's a distributed transaction — how do you handle it without two-phase commit?
- Your order event is processed twice due to at-least-once delivery. How do you ensure the customer is charged only once?
- How does the SAGA pattern differ from what you did in CodeMas?

---

### Bridge 10: Plagiarism Behavioral Signals → Design a Fraud Detection System

**Your decision:** Use behavioral signals (paste ratio, speed, tab switches) alongside code similarity — not just what was submitted, but how.

**The pattern:** Behavioral anomaly detection. Features from user behaviour, not just content.

**Bridge question:**
> "Design Razorpay's fraud detection system. A payment is made — you have 200ms to decide if it's fraudulent before approving it."

*What this covers:* feature engineering (device fingerprint, velocity, geolocation delta, transaction history), ML model serving (latency vs accuracy), rule engine + ML ensemble, real-time feature store, false positive cost vs false negative cost.

**Follow-ups:**
- Your plagiarism system runs post-exam — latency doesn't matter. Fraud detection has 200ms. What changes?
- You flag a legitimate transaction as fraud. The user can't pay. What's the cost of a false positive vs a false negative?
- A new fraud pattern emerges your model has never seen. How do you detect it?
- Swanand's behavioral signals are hand-crafted features. In fraud detection, how do you decide which features to use?

---

## PART 7 — FINAL INTERVIEW RULES

After every session, the interviewer must give:

**Sharp on:** [list topics where answers were detailed, precise, and unprompted]

**Needs work on:** [list topics where answers were shallow, hesitant, or incorrect]

**Review before next interview:** [2–3 specific concepts — e.g., "Redis pub/sub internals", "SAGA pattern", "HNSW index mechanics"]

---

## PART 8 — COMPLETE SYSTEM DESIGN PATTERNS REFERENCE

Use this section as a refresher before any interview. For each pattern the interviewer should be able to ask: "You used X in your project — now explain how it applies to Y."

---

### Scalability

| Pattern | What it solves |
|---|---|
| **Horizontal scaling** | Add more machines instead of bigger machines — stateless services scale this way |
| **Vertical scaling** | Bigger machine — simpler, but has a hard ceiling |
| **Auto-scaling** | Scale in/out based on metrics (CPU, queue depth) — AWS ASG, Kubernetes HPA |
| **Load balancing** | Distribute traffic across instances — round robin, least connections, consistent hash, L4 vs L7 |
| **Database sharding** | Split data across DBs by shard key (user ID, geo) — watch for hot shards |
| **Read replicas** | Offload reads from primary — replication lag is the trade-off |
| **CQRS** | Separate read and write models — optimise each independently |
| **Event sourcing** | Store events not state — replay to rebuild. Audit trail for free. Complexity cost. |
| **Cell-based architecture** | Partition users into isolated cells — blast radius of one cell's failure is contained. Slack, Amazon use this. |
| **Geo-distribution / multi-region** | Serve users from nearest region — active-active vs active-passive trade-off |

---

### Multi-Tenancy

| Pattern | What it solves |
|---|---|
| **Silo model (DB per tenant)** | Strongest isolation, highest cost — right for enterprise with compliance requirements |
| **Pool model (shared DB, tenant_id column)** | Cheapest — one wrong query leaks cross-tenant data. Mitigate with RLS. |
| **Bridge model (schema per tenant)** | Shared DB instance, separate schema — middle ground, Postgres handles well |
| **Row-level security (RLS)** | DB enforces tenant filter at query level — protection even if app layer forgets WHERE clause |
| **Tenant-aware caching** | Cache keys must include tenant ID: `cache:{tenant_id}:{resource}` — never share across tenants |
| **Noisy neighbour** | One tenant's heavy load degrades others — per-tenant rate limits, per-tenant queues |
| **Tenant-aware connection pooling** | Prevent one tenant's slow queries from starving pool for all others |
| **Tenant isolation for compute** | Standard tier = shared infra, Enterprise = dedicated — tiered pricing model |
| **Cross-tenant data leakage** | Hardest SaaS bug — integration tests must assert cross-tenant queries return empty |

---

### Caching

| Pattern | What it solves |
|---|---|
| **Cache-aside (lazy loading)** | App checks cache, loads DB on miss, writes to cache — most common pattern |
| **Write-through** | Write to cache and DB simultaneously — never stale, higher write latency |
| **Write-behind (write-back)** | Write to cache, async flush to DB — fast writes, risk of data loss on crash |
| **Read-through** | Cache sits in front of DB, handles its own population |
| **CDN caching** | Static assets served from edge — lower global latency |
| **Cache eviction policies** | LRU (least recently used), LFU (least frequently used), TTL-based |
| **Thundering herd / cache stampede** | Many requests hit DB simultaneously on cache miss — fix: mutex lock or probabilistic early expiry |
| **Cache warming** | Pre-populate cache before traffic hits — avoid cold start latency spike |
| **Distributed cache** | Redis Cluster — sharded, replicated cache across machines |

---

### Async & Queuing

| Pattern | What it solves |
|---|---|
| **Message queue** | Decouple producer and consumer — absorbs bursts (SQS, RabbitMQ) |
| **Pub/sub** | One publisher, many subscribers — fan-out (Kafka, Redis pub/sub, SNS) |
| **Consumer groups (Kafka)** | Multiple consumers share a topic — each partition consumed by one consumer in the group |
| **Dead letter queue (DLQ)** | Failed messages go here after N retries — investigate separately, don't block main queue |
| **Outbox pattern** | Write DB row and event in one transaction — guarantees event is published even if service crashes after write |
| **Competing consumers** | Multiple workers pull from same queue — parallel processing, each message processed once |
| **Priority queue** | High-priority messages processed before low-priority |
| **Backpressure** | Fast producer overwhelming slow consumer — signal producer to slow down, or buffer, or drop. Critical in streaming. |
| **Fan-out** | One event triggers multiple downstream processes — Order placed → notify warehouse, send email, update analytics |
| **Exactly-once delivery** | Hard to guarantee in distributed systems — use idempotency keys to simulate it |

---

### Reliability & Resilience

| Pattern | What it solves |
|---|---|
| **Circuit breaker** | Stop calling a failing service — fail fast, allow recovery (Closed → Open → Half-open) |
| **Retry with exponential backoff + jitter** | Retry failing calls with increasing delay + random jitter — avoids thundering herd on retry |
| **Bulkhead** | Isolate resources per service — one slow dependency doesn't exhaust thread pool for all |
| **Timeout** | Don't wait forever — set max wait, fail fast, let caller handle it |
| **Fallback** | If primary fails, return cached/default/degraded response — degrade gracefully |
| **Idempotency** | Same request N times = same result — critical for retries. Use idempotency keys. |
| **Load shedding** | Drop low-priority traffic under extreme load — protect core functionality |
| **Graceful degradation** | Core features work even when non-core services fail |
| **Health check / heartbeat** | Regular pings detect failure before users do — liveness vs readiness probes |
| **Chaos engineering** | Deliberately inject failures — find weaknesses before production does |

---

### Distributed Transactions

| Pattern | What it solves |
|---|---|
| **Two-phase commit (2PC)** | Atomic commit across multiple services — slow, coordinator is SPOF, mostly avoided |
| **SAGA — Choreography** | Services react to each other's events — decentralised, harder to debug |
| **SAGA — Orchestration** | Central orchestrator calls services in sequence — easier to reason about, single point of control |
| **Compensating transactions** | If a SAGA step fails, run a compensating action to undo previous steps — not a rollback, an explicit reversal |
| **Eventual consistency** | Distributed system will become consistent given enough time — accept temporary inconsistency for availability |

---

### Consistency Models

| Model | What it means |
|---|---|
| **ACID** | Atomicity, Consistency, Isolation, Durability — traditional relational DB guarantees |
| **BASE** | Basically Available, Soft state, Eventually consistent — NoSQL trade-off |
| **CAP theorem** | In a partition, choose Consistency OR Availability — you can't have both |
| **PACELC** | Extends CAP — even without partition, trade off Latency vs Consistency |
| **Strong consistency** | Every read sees the most recent write — expensive, requires coordination |
| **Eventual consistency** | Reads may be stale temporarily — cheap, scales well |
| **Causal consistency** | Causally related operations seen in order by all nodes |
| **Read-your-writes** | After you write, you always read your own write — weaker than strong, stronger than eventual |
| **Linearizability** | Operations appear instant and globally ordered — strongest model, highest cost |
| **Serializability** | Concurrent transactions produce same result as some serial order — database isolation level |

---

### Microservices Patterns

| Pattern | What it solves |
|---|---|
| **Service discovery** | Services find each other dynamically — client-side (Eureka) vs server-side (AWS ALB) |
| **Service mesh** | Handles retries, mTLS, tracing between services at infrastructure level — Istio, Linkerd |
| **Sidecar pattern** | Proxy runs alongside app container — handles cross-cutting concerns (auth, logging, retries) |
| **Strangler fig** | Gradually replace monolith — wrap old system, migrate feature by feature, never big-bang rewrite |
| **API gateway** | Single entry point — auth, rate limiting, routing, SSL termination, protocol translation |
| **BFF (Backend for Frontend)** | Separate backend per client type — mobile gets optimised mobile API, web gets its own |
| **Shared nothing** | Each service owns its data — no shared DB between services |

---

### Data Pipelines & Streaming

| Pattern | What it solves |
|---|---|
| **Batch processing** | Process large volumes of data on a schedule — Spark, Hadoop. High throughput, high latency. |
| **Stream processing** | Process data as it arrives — Kafka Streams, Flink. Low latency, event-by-event. |
| **Lambda architecture** | Batch layer (accurate, slow) + Speed layer (approximate, fast) + Serving layer merges both |
| **Kappa architecture** | Stream-only — no batch layer. Simpler. Reprocess by replaying the stream. |
| **ETL** | Extract → Transform → Load — transform before storage |
| **ELT** | Extract → Load → Transform — load raw, transform in-warehouse (BigQuery, Snowflake) |
| **Change Data Capture (CDC)** | Stream DB changes as events — Debezium reads Postgres WAL. Sync secondary systems without polling. |

---

### Database Internals

| Concept | What it means |
|---|---|
| **B-tree index** | Default index — balanced tree, great for reads, point lookups, range queries |
| **LSM tree** | Log-structured merge tree — optimised for writes (Cassandra, RocksDB). Reads require merging. |
| **Write-ahead log (WAL)** | Every write goes to WAL first — DB survives crash by replaying WAL on restart |
| **Covering index** | Index contains all columns a query needs — no table lookup required, fastest possible read |
| **Composite index order** | `(a, b)` index helps queries on `a` and `(a, b)` but NOT on `b` alone |
| **N+1 query problem** | Loading N records, then querying each individually — fix with JOIN or batch fetch |
| **Materialized view** | Pre-computed query result stored as a table — fast reads, needs refresh on data change |
| **Denormalisation** | Duplicate data to avoid JOINs — trade write complexity for read speed |
| **Partitioning** | Split one large table into smaller partitions by range/hash — Postgres native, transparent to queries |

---

### Storage Types

| Type | When to use |
|---|---|
| **Block storage (EBS)** | Low latency, raw disk, attached to one instance — databases, OS volumes |
| **Object storage (S3)** | Unlimited scale, high durability, high latency — images, videos, backups, data lake |
| **File storage (EFS)** | Shared, mountable by many instances simultaneously — shared config, ML model weights |
| **In-memory storage (Redis)** | Sub-millisecond — cache, session store, leaderboards, rate limiting |
| **Time-series DB (InfluxDB, TimescaleDB)** | Metrics, sensor data — optimised for time-range queries, automatic downsampling |

---

### Geographic Distribution

| Pattern | What it solves |
|---|---|
| **Active-active multi-region** | Traffic served from multiple regions simultaneously — no failover delay, conflict resolution required |
| **Active-passive multi-region** | One region handles traffic, other is on standby — simpler, failover has delay |
| **Geo-routing** | Route user to nearest region — DNS-based (Route53 latency routing) or Anycast |
| **Data residency / sovereignty** | GDPR — EU user data must stay in EU. Architecture must enforce regional data boundaries. |
| **CDN edge nodes** | Cache content globally — user hits closest PoP, not origin server |

---

### Consensus & Coordination

| Pattern | What it solves |
|---|---|
| **Leader election** | One node coordinates — Raft algorithm. etcd, ZooKeeper handle this. |
| **Raft consensus** | Distributed agreement — leader elected, log replicated, majority quorum required to commit |
| **Fencing tokens** | Prevent split-brain — stale leader gets a monotonically increasing token; storage rejects writes from old tokens |
| **Distributed lock (Redis SETNX / Redlock)** | Mutual exclusion across machines — set if not exists with TTL, Redlock uses quorum across Redis nodes |

---

### Concurrency & Locking

| Pattern | What it solves |
|---|---|
| **Pessimistic locking (SELECT FOR UPDATE)** | Lock before read — prevents concurrent writes, higher contention |
| **Optimistic locking (version/ETag)** | No lock — check version on write, retry on conflict — lower contention, works for low-conflict scenarios |
| **Distributed lock** | Lock across machines — Redis SETNX with TTL |
| **Rate limiting** | Token bucket (bursts allowed), leaky bucket (smooth rate), sliding window (per-user per-minute) |
| **Semaphore** | Limit concurrent operations — only N workers can run this job simultaneously |

---

### API Design

| Pattern | What it solves |
|---|---|
| **Cursor-based pagination** | Stable pagination on live data — offset breaks when rows are inserted/deleted between pages |
| **Offset pagination** | Simple but breaks on live data — fine for static datasets |
| **API versioning** | URL path (`/v1/`), Accept header, query param — URL is most common, headers are cleanest |
| **Idempotency keys** | POST with `Idempotency-Key` header — server deduplicates, safe to retry payment APIs |
| **Webhook pattern** | Server pushes to caller's URL on event — inversion of polling. Caller must handle retries and failures. |
| **Long polling** | Client requests, server holds open until data ready — simpler than WebSocket, higher latency than SSE |
| **SSE** | Unidirectional server push — notifications, live results. HTTP/1.1 compatible. |
| **WebSockets** | Bidirectional persistent connection — chat, multiplayer, collaborative editing |
| **gRPC** | Binary, schema-defined (protobuf), streaming — fast internal service communication |

---

### Release & Deployment Patterns

| Pattern | What it solves |
|---|---|
| **Blue-green deployment** | Two identical envs — instant traffic switch, instant rollback |
| **Canary deployment** | Roll out to X% of users first — catch failures before full rollout |
| **Rolling deployment** | Replace instances one-by-one — no downtime, slower rollout |
| **Feature flags** | Deploy code but disable feature — decouple deployment from release |
| **Dark launch** | Run new code in parallel with old — compare outputs, no user impact |
| **Shadow traffic** | Mirror real traffic to new service — test under real load without affecting users |

---

### Search & Retrieval

| Pattern | What it solves |
|---|---|
| **Inverted index** | Word → list of docs — foundation of all full-text search |
| **BM25** | Lexical relevance — keyword match with term frequency + inverse document frequency |
| **Vector search (ANN)** | Semantic similarity — HNSW, IVF-Flat, LSH indexes in pgvector/Pinecone/Weaviate |
| **Hybrid retrieval** | BM25 + vector combined — exact match + semantic recall |
| **Re-ranking** | Two-stage: fast ANN retrieval → expensive cross-encoder re-rank — better precision, higher latency |
| **RAG** | Retrieve context → LLM generates grounded answer |
| **Faceted search** | Filter by structured attributes alongside text search |

---

### AI / Agent Patterns

| Pattern | What it solves |
|---|---|
| **RAG** | Ground LLM answers in retrieved documents — reduces hallucination |
| **Tool use / function calling** | LLM selects and calls tools to complete a task |
| **ReAct (Reason + Act)** | Agent reasons, acts, observes result, repeats — iterative task completion |
| **Planner-Executor** | Planner decomposes task, executors run steps — separation of concerns |
| **Orchestrator-Worker** | Central orchestrator dispatches to specialised agents |
| **Supervisor-Worker** | Supervisor reviews output, routes failures back to workers |
| **HITL (Human-in-the-loop)** | Human approval gate before consequential actions |
| **Multi-agent (Researcher + Writer)** | Specialised agents collaborate — separation of concerns enforced by identity |
| **Memory (short/long term)** | Session memory vs persistent memory store across conversations |
| **Eval loop** | Automated quality measurement — RAGAS (faithfulness, relevance, retrieval quality) |
| **Score gate** | Below confidence threshold, skip LLM call — saves cost, prevents hallucination |
| **Chain-of-thought** | Step-by-step reasoning before answer — improves accuracy on complex tasks |

---

### Data Modelling Patterns

| Pattern | What it solves |
|---|---|
| **Adjacency list** | Parent ID on each row — simple hierarchy in SQL, N+1 risk for deep trees |
| **Materialized path** | Store full path string (`/1/4/7/`) — fast subtree reads, harder updates |
| **Nested sets** | Left/right boundaries for each node — fast subtree reads, expensive writes |
| **Closure table** | All ancestor-descendant pairs stored — flexible queries, highest storage cost |
| **Polymorphic association** | One FK column references multiple tables — flexible but breaks foreign key constraints |
| **Soft delete** | `deleted_at` timestamp instead of DELETE — preserves history, complicates all queries |

---

### Observability

| Pattern | What it solves |
|---|---|
| **Metrics** | Numeric time-series — latency p99, error rate, throughput, saturation |
| **Structured logs** | JSON logs with consistent fields — queryable in Datadog, Splunk, CloudWatch |
| **Distributed tracing** | End-to-end request path across services — OpenTelemetry, Jaeger, Datadog APM |
| **Alerting** | Notify when metric crosses threshold — avoid alert fatigue with SLO-based alerts |
| **SLI / SLO / SLA** | SLI = what you measure, SLO = your target, SLA = contractual commitment |
| **Error budget** | SLO allows X% downtime per month — burn rate determines when to stop releases |
| **Runbook** | Step-by-step guide for responding to an alert — reduces MTTR |
| **Dashboards** | Real-time view of system health — golden signals: latency, traffic, errors, saturation |

---

### Security

| Pattern | What it solves |
|---|---|
| **Zero trust** | Never trust, always verify — even internal service-to-service traffic |
| **OAuth 2.0 / JWT** | Delegated auth — access token, refresh token, stateless session |
| **mTLS** | Mutual TLS — both client and server authenticate each other |
| **Secrets management** | Never secrets in code — AWS Secrets Manager, Vault |
| **API key rotation** | Secrets cycle regularly — breach has limited blast radius |
| **Rate limiting** | Prevent abuse — per-IP, per-user, per-API key |
| **Input validation** | Validate at system boundary — SQL injection, XSS, command injection prevention |
| **Principle of least privilege** | Services get only permissions they need — blast radius of compromise is limited |
| **Encryption at rest / in transit** | Data protected when stored and when moving — TLS, AES-256 |
| **DDoS mitigation** | Absorb/deflect volumetric attacks — CloudFront, AWS Shield, rate limiting at edge |

---

*This document was built from real projects. Every architecture decision, trade-off, and system design bridge is grounded in something Swanand actually built and shipped.*
