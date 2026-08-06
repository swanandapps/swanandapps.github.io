# Interview Prep Master Guide — Swanand Kadam

> Printable study guide covering all 6 projects across every dimension you'll be asked about.
> Read this before any system design or technical interview.

---

## 1. Tech Stacks at a Glance

| Project | Frontend | Backend | AI / Agent | Data | Infra |
|---|---|---|---|---|---|
| **CodeMas** | Vue 3 + Vite | Django 4.2 + DRF + Simple JWT | — | Postgres + MongoDB + SQS FIFO | Lambda, Nginx, Gunicorn, Docker |
| **the01.dev** | React 19 + TS + Vite + Tailwind | FastAPI 0.115 + Pydantic 2.10 | Hermes v0.18.2 + LangGraph 1.2.9 + RAGAS 0.2.14 + GEPA | Firestore (prod), in-mem (dev) | Ollama/vLLM, FastMCP, uvicorn |
| **Munshi** | — (agent CLI/API) | FastAPI + Hermes runtime | Hermes + SOUL.md + GEPA eval harness | SQLite (local, Decimal columns) | Docker Compose, Ollama (local-only) |
| **Kalaam** | Vue 2 + Quasar (PWA) | npm package (pure JS, zero deps) | — | In-browser memory{} flat object | Netlify static, service worker cache |
| **stringy-core** | — (library) | npm package (ESM) | — | — | npm publish, Husky + lint-staged |
| **Trade Compliance** | — (agent CLI) | Hermes runtime × 2 | Researcher SOUL + Writer SOUL | In-memory / file | Docker Compose, Ollama |

---

## 2. Claims Made — and How to Defend Each

Every claim below will come up in interviews. Know the defense before you make the claim.

### CodeMas

| Claim | What to say when pushed |
|---|---|
| "10,000 concurrent students" | SQS FIFO absorbs submission bursts. Lambda scales horizontally per invocation. SSE connections are lightweight persistent HTTP — Gunicorn handles thousands with gevent workers. Postgres handles reads via PgBouncer connection pooler. |
| "Plagiarism catches 19× more cases" | Behavioral phase 1 catches low-effort copiers who never submit duplicate code. Similarity phase 2 catches code-share rings. Previous system was exact-match only — caught near-zero behavioral cheating. The 19× is relative improvement over that baseline. |
| "Secure sandbox" | Lambda in VPC with no outbound internet access. `/tmp` wiped per invocation. 128MB memory limit. 15-second hard timeout enforced by AWS. Student code cannot phone home or access other students' data. |
| "Real-time results" | Honest version: Django polls Postgres every 500ms and forwards via SSE. True push existed only when Redis pub/sub was the delivery mechanism. After Lambda migration, SSE is "polling theater" — architecturally equivalent to client polling. Own this trade-off when asked. |

### the01.dev

| Claim | What to say when pushed |
|---|---|
| "Grounded or silent" | Enforced in code, not model trust. Score gate at 0.45: below threshold, `return DECLINE` fires before the model is ever invoked. Model can only hallucinate within the retrieved material, not from training data at large. |
| "Sovereign inference" | Generation: 100% local — Ollama in dev (qwen2.5:3b), vLLM on GPU in prod. One remaining external call: retrieval embeddings use OpenAI text-embedding-3-small. Closing it is a one-file swap to nomic-embed-text. |
| "Self-evolving SOUL via GEPA" | Pipeline runs offline, never during live chat. Human approves before any SOUL write. Pareto gate: candidate avg > baseline avg AND no case scores < 2. Every applied SOUL SHA-fingerprinted in audit log. |
| "EU-AI-Act compliant audit" | Audit row written BEFORE inference — not after. Fields: student_id, question, model, provider, SOUL hash, timestamp. Cannot be retroactively modified. |
| "GDPR — right to erasure" | DELETE /api/hermes/memory/{id} removes all stored turns and profile for that student. GET version gives transparency. |

### Munshi

| Claim | What to say when pushed |
|---|---|
| "Nothing leaves the machine" | Inference on local Ollama. SQLite on-disk. No cloud API for generation. Fully local by architecture, not policy. |
| "Never invents a number" | Enforced by tool architecture: convert_currency, compute_gst, lookup_hsn are deterministic Python functions. Model calls tool; tool returns Decimal; model narrates. SOUL.md: "if a capability isn't built yet, say so plainly." |
| "Prepare, don't commit" | HITL approval required before any consequential action. Model shows computed summary; human approves. Nothing filed, issued, or imported without human in the loop. |

### Kalaam

| Claim | What to say when pushed |
|---|---|
| "Fully offline" | Cache-first service worker: after first load, zero network required. Interpreter is pure JS, zero runtime deps, runs in the browser. Verified on airplane mode. |
| "Adding a language = 1 keyword map entry, no code change" | Only Phase 1 (Cleaning) is language-aware. It substitutes language keywords to normalized English tokens. Parser and interpreter operate on normalized tokens — they see no language-specific syntax. |

### stringy-core

| Claim | What to say when pushed |
|---|---|
| "Zero runtime dependencies" | package.json has no `dependencies` key, only `devDependencies`. `npm install stringy-core` installs nothing extra. Bundle is self-contained. |
| "Tree-shakeable" | All functions are named exports from individual modules. A bundler doing static analysis on `import { maskEmail }` can drop the other 49 functions at compile time. |

---

## 3. Toughest Challenge — Per Project

### CodeMas

**Challenge:** At exam close, all 10,000 students submit within a 30-second burst. Django must check each student's attempt counter atomically — if two tab refreshes race, both pass the limit check and double-insert.

**How solved:** SELECT FOR UPDATE inside a Postgres transaction. Row lock on `submission_attempts` means the second concurrent request blocks until the first commits. It then reads the updated count, sees the limit is reached, and rejects.

**What to say:** "The hard problem wasn't scale — SQS handles that by design. The hard problem was the race condition at the submission gate. SELECT FOR UPDATE is the textbook answer. Alternative: optimistic locking (check version on UPDATE, retry on mismatch) — better for low-contention, but adds retry complexity we didn't need."

---

### the01.dev

**Challenge:** Dev model (qwen2.5:3b) loses multi-turn context when routed through the Hermes ReAct loop. Student asks Q1, gets answer, asks follow-up Q2 — Hermes re-invokes tools and the answer doesn't reference Q1's context.

**How solved:** Dev reliable path bypasses the agent loop and calls the model directly with conversation history injected. Same SOUL, same tools, different invocation path based on model capability. Production model routes through full Hermes.

**What to say:** "This is a production constraint that doesn't appear in demos. Small models and agentic loops don't compose well — context gets fragmented across tool calls. Our mitigation is environment-aware routing. HindSight memory is configured but only activates when traffic routes through the full Hermes agent loop. It's a documented gap, not a hidden one."

---

### Munshi

**Challenge:** GST reconciliation needs to match invoices across GSTR-2A and internal purchase register — different naming conventions, abbreviations, date formats. Exact match fails on 40–50% of real invoices.

**How solved:** Four-signal fuzzy pipeline: (1) GSTIN prefix match, (2) name token set ratio ≥ 85% after normalization, (3) amount within ±2%, (4) date within 7 days. Auto-match when all 4 agree. Escalate to model when 2–3 agree. No-match when <2. Model only touches the ambiguous 20%.

**What to say:** "The key insight: don't put the model on the hot path for every invoice. The four-signal pipeline filters 80% of cases automatically. The model sees only the genuine judgment calls."

---

### Kalaam

**Challenge:** Meaningful error messages for students who don't know English. The interpreter fails on line 3 — the error needs to be in Hindi, not English.

**Current state (honest):** English error messages. ExecutionStack gives the step trace in the student's language, so they can see WHERE it failed. Error text localization is scoped backlog.

**What to say:** "Localized errors need template strings per language — different from keyword substitution which only maps source code terms. It's scoped and feasible, not a rearchitecting problem. Known gap."

---

### stringy-core

**Challenge:** ESM-first package that also needs to run in Jest, which defaults to Node.js CommonJS.

**How solved:** Babel transform in `jest.config.js` (`@babel/preset-env` targeting current Node). Tests run against CJS-transpiled code. npm package ships ESM. Two distinct consumption paths.

**What to say:** "Ship ESM for bundlers and tree-shaking. Use Babel transform only in the test environment, never in package output. The ESM/CJS split is the #1 source of confusion in modern JS libraries — separate the concerns."

---

### Trade Compliance

**Challenge:** Preventing the Researcher agent from tool-looping — calling the same tool repeatedly because results feel incomplete.

**How solved:** Three-layer defense: (1) SOUL.md tells it to stop when enough data is gathered, (2) Hermes hard turn limit (max_iterations in config), (3) MCP schema validation rejects malformed args — error becomes context that nudges a different call.

**What to say:** "Pure SOUL instructions aren't enough — models under pressure drift from them. You need a hard circuit breaker in the runtime. The turn limit is that circuit breaker."

---

## 4. Tool Calling Reliability

Applies to: the01.dev (Hermes MCP), Munshi (same), Trade Compliance (Researcher's 4 tools).

### What fails

| Failure mode | What happens |
|---|---|
| Malformed arguments | Model passes wrong type or missing required field |
| Tool timeout | External API doesn't respond within deadline |
| Unexpected return format | Tool returns `null` or error object instead of typed struct |
| Infinite tool loop | Model keeps calling because results feel incomplete |
| Hallucinated tool name | Model invents a tool name that doesn't exist |

### Defense at each layer

| Layer | Mechanism |
|---|---|
| MCP schema validation | Each tool has a typed JSON schema. Wrong args = rejected at the boundary; error becomes conversation context |
| Retry with backoff | Hermes retries transient failures up to 3× with exponential backoff |
| SOUL.md constraint | "Never invoke a tool you don't have. If a capability isn't built, say so plainly." |
| Turn limit (circuit breaker) | Hard cap on max_iterations in Hermes config. Loops cannot run forever |
| Structured return types | Tools return typed dicts, not prose. Caller code validates shape before passing to model |
| Deterministic tools for numbers | In Munshi: all financial computation in Python. Tool return value is the ground truth — model narrates, never computes |

### Interview answer

"Tool calling reliability has three failure modes: bad args, bad return, bad loops. MCP schema validation kills bad args at the boundary. Turn limits kill loops. Bad returns are the real risk — design tools to return structured data so your code can validate the shape before passing it to the model. If validation fails, decline gracefully."

---

## 5. Lambda Cold Start + Warming (CodeMas)

### The problem

Cold start adds 2–4 seconds after a container is recycled. At exam start, all containers are cold. Student submits → Lambda cold → 4s overhead → 5–7s total perceived latency. Unacceptable.

### Options and trade-offs

| Strategy | Mechanism | Cost | When to use |
|---|---|---|---|
| Provisioned Concurrency | Lambda keeps N containers warm at all times | Pay when idle | Predictable load with known peak |
| Scheduled ping | CloudWatch Event triggers dummy Lambda every 5 min | Negligible | Only keeps 1 warm; burst demand still cold-starts |
| Pre-exam warm-up trigger | Admin action 5 min before exam warms M containers | Dev effort | Requires exam schedule awareness |
| Smaller deployment package | Less code = faster init | Discipline | Helps, doesn't eliminate cold starts |
| ARM Lambda (Graviton) | Faster init + ~20% cheaper | Small migration | Best default for new Lambdas |

### Current approach

Provisioned Concurrency set to estimated peak concurrent Lambda executions.

Peak math: 10,000 students × 30-second burst window ÷ 1.5s average execution = ~500 concurrent Lambdas at peak. Provision 500 containers = guaranteed warm at burst. Between exams: set provisioned concurrency to 0. Pay only during exam windows.

### Interview answer

"Cold starts are a Lambda fact of life. Provisioned Concurrency is the right answer when load is predictable — and an exam platform has a perfectly predictable load schedule. Provision to your peak concurrency estimate, turn it off between exams, pay only when you need it. Alternative: trigger warm-up 5 minutes before exam open via admin action."

---

## 6. Scaling Problems — What Breaks First

### CodeMas — 10× (100K users)

| Bottleneck | Current limit | Breaking point | Fix |
|---|---|---|---|
| SSE connection state | Gunicorn gevent workers | ~50K open connections per instance | Dedicated SSE microservice (Node.js) or drop SSE for client polling |
| Postgres SSE reads | 500ms poll × 100K connections | 200K reads/sec → pool saturation | PgBouncer + read replicas + Redis result cache (TTL 30s post-completion) |
| SQS FIFO throughput | 3,000 msg/sec per queue | ~30K submissions/burst | Per-exam queues already isolate; or switch to Standard queue + idempotency key |
| Lambda concurrency | AWS default: 1,000/account | ~33K submissions/min | Request quota increase via AWS console |

### the01.dev — 10K courses

| Bottleneck | Current limit | Breaking point | Fix |
|---|---|---|---|
| ANN retrieval latency | In-memory HNSW | ~50K vectors before latency spikes | Partition index by course; lazy-load course index on first access |
| Re-indexing on new content | Full rebuild | Minutes of downtime per new course | Write-behind: append new chunks live, rebuild index async |
| LangGraph checkpoints | MemorySaver (in-process) | Process restart wipes in-flight workflows | Postgres or Redis checkpointer |
| vLLM GPU throughput | Single 40GB GPU | ~50 concurrent inference requests | Multi-GPU tensor parallelism or quantised model (4-bit, 4× throughput) |

### Munshi — 50 accounting firms (multi-tenant)

| Bottleneck | Current limit | Breaking point | Fix |
|---|---|---|---|
| Local architecture | Single machine, N=1 | Any N > 1 | Per-firm Postgres schema or row-level security; shared Hermes runtime |
| HITL flow | CLI approval | >10 approvals/day | Web dashboard with approval queue and webhook notifications |
| Ollama model serving | Single instance | Concurrent requests from same firm | Request queue or shared vLLM endpoint |

---

## 7. Concurrency Problems

### CodeMas: Submission Race Condition

**Problem:** Two browser tabs submit the same (student, question) pair simultaneously. Both pass the attempt limit check before either writes. Double-insert.

**Solution:**
```sql
BEGIN;
SELECT attempt_count FROM submission_attempts
  WHERE student_id = $1 AND question_id = $2
  FOR UPDATE;           -- row lock acquired here
-- if count >= limit: ROLLBACK; return 429
UPDATE submission_attempts
  SET attempt_count = attempt_count + 1
  WHERE student_id = $1 AND question_id = $2;
INSERT INTO submissions (...) VALUES (...);
COMMIT;                 -- lock released here
```

Second request blocks on `FOR UPDATE` until first commits, reads the incremented count, sees limit reached, rejects.

**Alternative:** Optimistic locking — read version column, UPDATE WHERE version = read_version, retry on 0 rows affected. Better for low-contention, adds retry complexity. SELECT FOR UPDATE is simpler here.

---

### CodeMas: SQS Deduplication

**Problem:** Django times out waiting for SQS ack, retries. SQS receives the message twice. Lambda executes the submission twice.

**Solution:** SQS FIFO with `MessageDeduplicationId = submission_uuid` (generated once before enqueue). Within the 5-minute deduplication window, any message with the same ID is silently dropped. Lambda sees exactly one message.

**Why FIFO over Standard:** Standard is at-least-once but unordered. A student's second attempt must be processed after their first (the first informs the attempt counter). FIFO with `MessageGroupId = exam_id + student_id` enforces ordering per student per exam.

---

### the01.dev: LangGraph Checkpoint Conflict

**Problem:** Student opens QuizMe in two tabs, resumes workflow from both. Two concurrent requests read the same checkpoint, both try to write the next state.

**Solution (current):** MemorySaver is in-process — single-threaded event loop prevents true simultaneous writes in dev. In production with Postgres checkpointer: optimistic locking on checkpoint version. First write wins; second reads new checkpoint, retries.

---

## 8. Hallucination Problems

### the01.dev: Grounding Gate

The model is never called below threshold 0.45. Code enforcement, not model trust.

```python
results = vector_store.similarity_search_with_score(question, k=4)
top_score = results[0][1]  # cosine similarity of best chunk

if top_score < THRESHOLD:   # 0.45
    if not user_consented_general_knowledge:
        return {"type": "decline", "msg": "No relevant course material found."}
    else:
        return general_answer_flagged_as_ungrounded

# Only here if top_score >= 0.45
context = "\n".join([r[0].page_content for r in results])
# Inject context into SOUL system prompt → call model
```

Threshold calibrated on 100-question eval set: 50 in-scope (should answer), 50 out-of-scope (should decline). F1 maximized at 0.45.

**Residual risk:** Model answers within retrieved chunks. If a chunk is wrong, the model amplifies it. Mitigation: RAGAS faithfulness checks whether model claims are supported by the chunks.

**Interview answer:** "Zero hallucination is indefensible for any LLM system. The defensible claim: we don't generate when retrieval is weak. Below 0.45, the model is never invoked. Above 0.45, hallucination risk is constrained to the retrieved material, not the full training set."

---

### Munshi: Deterministic Numbers

**Two layers:**

1. **Tool architecture:** All financial computation in Python (`compute_gst`, `convert_currency`, `lookup_hsn_duty_rate`). Model calls tool; tool returns Decimal; model narrates. Number comes from the tool, not the model.

2. **SOUL.md constraint:** "NEVER invent a number — not a duty rate, tax amount, HS/HSN code, or any rupee/euro figure. If you don't have a tool for the computation, say so."

**Why Python Decimal (not float):**
```python
# float — wrong at scale
1250.75 * 0.18  →  225.13499999999999

# Decimal — exact
Decimal('1250.75') * Decimal('0.18')  →  Decimal('225.1350')
```
At ₹5Cr annual revenue, float errors compound across thousands of invoices to hundreds of rupees. Wrong tax filing.

**Residual risk:** Model could fabricate a tool argument (e.g. wrong HS code to lookup). Mitigation: MCP schema validates arg types; audit log records every tool call; HITL approval before commitment.

---

## 9. Multi-Tenancy

### CodeMas: Per-Exam Isolation (Current)

- **SQS:** `MessageGroupId = exam_id`. Each exam gets its own ordered lane. One exam's burst doesn't delay another.
- **Postgres:** All queries exam-scoped via `exam_id` FK. No cross-exam data accessible at query level.
- **Lambda:** Stateless per invocation. No shared state between exams.
- **SSE:** Per-student stream, keyed to `submission_id`. No cross-student broadcast.
- **Redis cache:** Keyed `exam_id + metric`, TTL 10s.

**Extending to multi-school SaaS:**
- Add `school_id` to all tables
- Postgres Row-Level Security: `CREATE POLICY school_isolation ON submissions USING (school_id = current_setting('app.school_id'))`
- School JWT claims: `school_id` in token payload, set as Postgres session variable per request
- Per-school SQS queues or `MessageGroupId = school_id + exam_id`
- Per-school Lambda concurrency limits to prevent one school starving another

---

### the01.dev: Per-Student Memory

- Each student: Firebase Auth UID as `student_id`
- HindSight memory keyed by `student_id`: running profile + last N turns
- Audit log rows keyed by `student_id`
- GDPR delete: `DELETE /api/hermes/memory/{id}` removes all rows

**Extending to institutions:**
- Add `institution_id` to student profile
- Scope HNSW retrieval to institution's course catalogue (separate index per institution, or metadata filter)
- Per-institution Razorpay sub-merchant accounts

---

## 10. Event-Driven Architecture

### CodeMas: Full Submission Pipeline

```
Student Browser
  │  POST /submit  (sync — student needs 201 fast)
  ▼
Django (Gunicorn + gevent)
  │  SELECT FOR UPDATE  (attempt gate, ~2ms)
  │  INSERT submissions (status='pending')
  │  SQS.send_message(MessageGroupId=exam_id, DeduplicationId=sub_uuid)
  │  return 201  ← student gets this in ~10ms; execution is async from here
  ▼
SQS FIFO Queue (per-exam lane — absorbs burst, orders per student)
  │  Lambda event source mapping (auto-scales to queue depth)
  ▼
Lambda Sandbox (VPC, no internet, /tmp fresh each invocation, 15s timeout)
  │  Execute student code, capture stdout/stderr/exit_code
  │  UPDATE submissions SET result=..., status='completed' WHERE id=...
  ▼
Postgres (result written)
  ▲
  │  Django SSE handler polls every 500ms
  │  SELECT result FROM submissions WHERE id=$1
  │  If result != null: push SSE event, close stream
  ▼
Student Browser (receives result via SSE)
```

**Why SQS decoupling matters:** Django returns 201 in ~10ms. Lambda takes 1–5 seconds. Without the queue, Django blocks a Gunicorn worker for 5 seconds × 10K students = saturated thread pool.

**Honest SSE note:** After Lambda migration, Django polls Postgres every 500ms and forwards via SSE. True push only existed with Redis pub/sub. Client polling every 500ms is architecturally equivalent. If building today: drop SSE for client polling, or invest in WebSocket push.

---

### the01.dev: Streaming Token Response

```
Student Browser  (POST /api/hermes/chat + EventSource open simultaneously)
  ▼
FastAPI
  │  Write audit row (BEFORE inference — tamper-evident)
  │  Load HindSight memory (profile + last 3 turns)
  │  Embed question (OpenAI text-embedding-3-small)
  │  HNSW ANN → top-4 chunks by cosine similarity
  │  Check: top_score >= 0.45?
  ├── No: SSE {type:"decline"}, close
  └── Yes: inject chunks into SOUL system prompt
            call model (Ollama/vLLM, streaming)
            SSE stream:
              → {type:"sources", chunks:[...]}  (first)
              → {type:"token", content:"..."} × N tokens
              → {type:"done"}
  ▼
Student Browser (renders sources immediately, streams tokens as they arrive)
```

**Why SSE over WebSocket:** SSE is unidirectional (server→client) — exactly what streaming token generation needs. WebSocket is bidirectional and adds complexity for no benefit. SSE also has built-in reconnect via `Last-Event-ID`.

---

## 11. Deployment — How Each Project Runs

### CodeMas
```
Internet
  → Nginx (SSL termination, proxy_read_timeout 300s for SSE — key config)
  → Gunicorn (--worker-class gevent, --workers 4)
  → Django
  → AWS SQS → Lambda (AWS-managed, VPC, no server to manage)
  → RDS Postgres
  → MongoDB (submission history, analytics)
```
- Nginx: `proxy_read_timeout 300s` — SSE connections are long-lived; default 60s kills them.
- Gunicorn gevent workers: handle thousands of concurrent SSE connections with shared thread pool.
- Lambda: deployed via Serverless Framework or AWS CDK. VPC config: private subnet, no outbound internet.
- DB migrations: `python manage.py migrate` in pre-deploy hook.

---

### the01.dev
```
5 processes (localhost in dev / Docker Compose in prod):
  :5173  Vite (frontend)
  :8080  FastAPI (main backend — auth, payments, catalogue)
  :8642  Hermes gateway (agent runtime, isolated venv)
  :8090  LangGraph server (QuizMe + artifact gen, isolated venv)
  :11434 Ollama (local model serving)
```
- Each Python service in its own venv — Hermes, LangGraph, and RAGAS have conflicting transitive deps.
- Firestore credentials: service account JSON via `GOOGLE_APPLICATION_CREDENTIALS` env var.
- In prod: Docker Compose or Kubernetes, same ports, Nginx at the edge.

---

### Munshi
```
Docker Compose:
  hermes_runtime  (Hermes agent gateway)
  ollama          (local model — pull on first run)
  mcp_tools       (FastMCP stdio server, 4 tools)
  sqlite volume   (file-mounted, persists across restarts)
```
Entirely local. No internet connectivity after initial `ollama pull`. Sovereignty is architectural.

---

### Kalaam
```
npm publish → npm registry (library consumers)
Vue 2 + Quasar → build → dist/ → Netlify (static hosting)
Service worker: cache-first, auto-updates on new deploy
```
Zero server required for end users. All computation in the browser.

---

### stringy-core
```
npm publish --access public
```
Package on npm registry. Consumers `npm install stringy-core`. No server, no runtime infra.

---

### Trade Compliance
```
Docker Compose:
  ollama          (local model)
  mcp_tools       (4 MCP tools)
  researcher      (Hermes runtime, researcher SOUL.md, has tools)
  writer          (Hermes runtime, writer SOUL.md, no tools — config.yaml tools:[])
```
Swap `model_endpoint` in config.yaml from `http://ollama:11434/v1` to `https://api.openai.com/v1` → runs on OpenAI. No code change.

---

## 12. Data Modeling

### CodeMas: Core Tables

```sql
-- Exam structure
CREATE TABLE exams (
  id UUID PRIMARY KEY,
  is_active BOOLEAN,         -- pre_save signal on True→False triggers plagiarism
  school_id UUID,
  created_at TIMESTAMPTZ
);
CREATE TABLE questions (
  id UUID PRIMARY KEY,
  exam_id UUID REFERENCES exams,
  difficulty TEXT,            -- used in speed_anomaly scoring (Phase 1 plagiarism)
  time_limit_seconds INT
);

-- Submission flow — the critical tables
CREATE TABLE submission_attempts (
  id UUID PRIMARY KEY,
  student_id UUID,
  question_id UUID REFERENCES questions,
  attempt_count INT DEFAULT 0,
  UNIQUE(student_id, question_id)   -- one row per student per question
  -- SELECT FOR UPDATE locks this row for each submission
);

CREATE TABLE submissions (
  id UUID PRIMARY KEY,              -- also SQS MessageDeduplicationId
  student_id UUID,
  question_id UUID REFERENCES questions,
  code TEXT,
  status TEXT,                      -- 'pending' | 'running' | 'completed' | 'error'
  result JSONB,                     -- {stdout, stderr, exit_code, time_ms}
  submitted_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ
  -- SSE handler polls: SELECT result FROM submissions WHERE id = $1
);
CREATE INDEX idx_submissions_pending ON submissions(id) WHERE status = 'pending';

-- Plagiarism results
CREATE TABLE behavioural_flags (
  student_id UUID,
  question_id UUID REFERENCES questions,
  risk_score FLOAT,                 -- paste_ratio*0.40 + speed*0.30 + tabs*0.15 + surprise*0.15
  flagged BOOLEAN                   -- risk_score > 0.2
);
CREATE TABLE similarity_flags (
  student_a UUID,
  student_b UUID,
  question_id UUID REFERENCES questions,
  similarity_score FLOAT            -- TF-IDF cosine similarity, flagged > 0.80
);
```

---

### the01.dev: Firestore Collections

```
students/{student_id}
  profile: {
    topics_covered: string[],
    turn_count: int,
    first_seen: timestamp,
    last_seen: timestamp
  }
  turns: [     ← HindSight sliding window (last N turns)
    { question, answer, sources: [{chunk_id, score}], timestamp }
  ]

audit_log/{row_id}    ← written BEFORE inference, immutable after write
  student_id: string
  question: string
  model: string
  provider: string    -- "ollama/qwen2.5:3b" | "vllm/mistral-7b"
  soul_hash: string   -- SHA-256 of SOUL.md at inference time
  timestamp: timestamptz

quiz_sessions/{session_id}   ← LangGraph MemorySaver checkpoint
  state: {
    objectives: string[],
    current_index: int,
    answers: [{question, student_answer, score, feedback}],
    final_score: float | null
  }
  thread_id: string
```

Vector store: HNSW in-memory index. Each vector metadata: `{course_id, module_id, chunk_index, text}`. Retrieval uses cosine similarity. Threshold: 0.45.

---

### Munshi: SQLite Schema

```sql
CREATE TABLE invoices (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  vendor_gstin TEXT,
  vendor_name TEXT,
  invoice_number TEXT,
  amount_inr TEXT,           -- stored as TEXT, loaded as Decimal — NEVER REAL
  date DATE,
  source TEXT                -- 'gstr2a' | 'purchase_register'
);

CREATE TABLE matches (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  invoice_a_id INT REFERENCES invoices,
  invoice_b_id INT REFERENCES invoices,
  match_type TEXT,           -- 'auto' | 'model_approved' | 'no_match'
  confidence_signals INT,    -- count of 4 signals that agreed (4=auto, 2-3=model)
  approved_by_human BOOLEAN DEFAULT FALSE,
  approved_at TIMESTAMPTZ
);

CREATE TABLE audit_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  action TEXT,               -- 'tool_call' | 'model_inference' | 'human_approval'
  tool_name TEXT,
  input_json TEXT,
  output_json TEXT,
  timestamp TIMESTAMPTZ
);
```

**Why TEXT for amount_inr, not REAL:** SQLite REAL is 64-bit float. `1250.75 * 0.18 = 225.13499999999999`. Load as `Decimal(row['amount_inr'])` from TEXT → exact arithmetic throughout.

---

### Kalaam: In-Interpreter Runtime State

```javascript
// Not a database — the interpreter's working state
memory = {
  'x': 5,
  'greeting': 'नमस्ते',
  'result': 13
}
// Flat object, intentionally. No scope chains — scope adds confusion for beginners.
// Trade-off: recursive functions would overwrite caller's variables. Known limitation.

ExecutionStack = [
  { step: 1, line: 1, action: 'assign', variable: 'x', value: 5 },
  { step: 2, line: 2, action: 'assign', variable: 'y', value: 8 },
  { step: 3, line: 3, action: 'add', operands: ['x','y'], result: 13 },
  { step: 4, line: 3, action: 'assign', variable: 'result', value: 13 },
  { step: 5, line: 4, action: 'print', value: 13, output: '13' }
]
// UI replays this array step-by-step in the student's own language.
// No teacher required — the stack IS the explanation.
```

---

## 13. Event-Driven vs. Request-Response — When to Use Each

| Criterion | Request-Response | Event-Driven (queue / stream) |
|---|---|---|
| Latency requirement | < 100ms, caller is blocking | > 1s acceptable, or caller can wait |
| Reliability requirement | OK to lose on crash | Must not lose (queue persists on disk) |
| Load pattern | Steady, predictable | Bursty, spiky |
| Caller needs result | Immediately | Eventually (poll or push) |
| Decoupling needed | Tight coupling is fine | Producer and consumer must scale independently |

**CodeMas split:** Submission acceptance = request-response (student needs 201 in < 100ms). Code execution = event-driven via SQS (async, result via SSE poll). Correct split. Running code execution synchronously in the API worker = 5s × 10K students = saturated thread pool.

---

## 14. Interview Bridges — Your Work to Real-World Systems

| Your system | Maps to |
|---|---|
| CodeMas SQS + Lambda | YouTube video processing (upload → queue → transcoding worker → CDN) |
| CodeMas SELECT FOR UPDATE | BookMyShow seat reservation (same race condition, same lock) |
| CodeMas SSE polling Postgres | Twitter timeline polling (after real-time push broke down at scale) |
| CodeMas plagiarism O(K×N) | Large-scale dedup in data pipelines (Spotify song dedup, Dropbox block dedup) |
| CodeMas behavioral signals first | Razorpay/PayPal fraud detection (behavior flags before deep ML model) |
| 01dev GEPA self-improvement | Anthropic's Constitutional AI self-critique loop |
| 01dev grounding gate 0.45 | Google Search confidence gate (below threshold → no answer box shown) |
| 01dev two-engine design | GitHub Copilot Workspace (agent for open-ended, workflow for CI/deploy) |
| 01dev sovereign inference | Apple on-device AI (privacy by architecture, not policy) |
| 01dev HITL interrupt in LangGraph | Content moderation human review queues (flag → queue → human decision) |
| Munshi local-first | Apple Neural Engine (processing on device, not in cloud) |
| Munshi Decimal over float | Every bank, payment processor, and accounting system ever built |
| Munshi 4-signal fuzzy pipeline | Entity resolution in dbt, Snowflake, Databricks data pipelines |
| Munshi SOUL.md constraints | Guardrails pattern for production AI agents |
| Kalaam 5-phase interpreter | CPython pipeline (tokenize → parse → compile → eval) |
| Kalaam Phase 1 keyword substitution | C preprocessor macros; SQL query rewriting in ORM layers |
| Kalaam ExecutionStack Learning Mode | VS Code debugger step-through (call stack is the same concept) |
| Kalaam zero deps | npm left-pad incident — zero deps = zero supply chain risk |
| stringy-core tree-shaking | Bundle size optimization in any production React/Vue app |
| stringy-core Intl API | i18n patterns at scale (Google, Microsoft use ICU/Intl internally) |
| Trade Compliance Researcher + Writer | MapReduce — gather phase, then synthesis phase |
| Trade Compliance MCP validation | OpenAI function calling schema enforcement (same contract) |
| Trade Compliance model-agnostic config | LiteLLM, LangChain model wrappers — provider abstraction |

---

## 15. What I'd Do Differently — Per Project

| Project | Top changes if starting over |
|---|---|
| **CodeMas** | (1) Drop SSE, use client polling — SSE is polling theater post-Lambda migration. (2) Postgres RLS for multi-school SaaS from day 1. (3) Provisioned Concurrency on exam schedule to eliminate cold starts. |
| **the01.dev** | (1) Local embedding model (nomic-embed-text) from day 1 — eliminate the OpenAI dependency. (2) Postgres checkpointer instead of MemorySaver — LangGraph workflows survive restarts. (3) Integration tests for the grounding gate before anything else. |
| **Munshi** | (1) Complete the GEPA self-improvement loop (reflection + proposal + Pareto gate). (2) Web approval UI instead of CLI for HITL. (3) Tool result caching — avoid redundant convert_currency calls for same inputs. |
| **Kalaam** | (1) Error messages in the student's language (template strings per language). (2) REPL mode — execute on keystroke, debounced 300ms. (3) Formal grammar spec — enables linter, formatter, IDE support. |
| **stringy-core** | (1) TypeScript from day 1 — return types would have caught the isPalindrome boolean bug at compile time. (2) Vitest benchmarks for O(n²) functions. (3) CDN IIFE build so contributors can use it via `<script>` tag without a bundler. |
| **Trade Compliance** | (1) Researcher returns structured JSON, not prose — Writer gets typed input, not text to parse. (2) Confidence score per finding so Writer can flag low-confidence claims. (3) RAGAS-style eval for report quality — currently no systematic evaluation. |

---

## 16. Common Follow-Up Questions — Quick Answers

**"How do you monitor all this in production?"**
CodeMas: CloudWatch for Lambda errors/duration, Postgres slow query log, Nginx access log for SSE connection drops.
the01.dev: Audit log IS the monitoring — every inference logged before it happens. Check `audit_log` for volume and SOUL hash distribution.
Munshi: SQLite `audit_log` table; every tool call and HITL decision recorded with inputs and outputs.

---

**"What's your testing strategy?"**
CodeMas: Django TestCase for API endpoints; pytest for plagiarism scoring logic; no Lambda integration tests (known gap).
the01.dev: FastAPI TestClient for endpoints; RAGAS eval set for grounding gate; no LangGraph workflow tests (gap).
Kalaam: Jest, 90–95% coverage on interpreter; Husky blocks commit if tests fail.
stringy-core: Jest, all 50+ functions covered; Husky + lint-staged enforces ESLint + Prettier on commit.

---

**"How do you handle secrets?"**
CodeMas: Django SECRET_KEY + DB credentials in env vars, never in code.
the01.dev: Firebase service account via `GOOGLE_APPLICATION_CREDENTIALS`; Razorpay keys in `.env`; OpenAI key in `.env`.
Munshi: No cloud secrets — local-only. Ollama needs no API key.
Trade Compliance: OpenAI key in `.env` if using cloud mode; Ollama mode needs no secrets.

---

**"How do you handle versioning / backward compatibility?"**
stringy-core: Semantic versioning; named exports are the public API contract; no breaking changes to existing function signatures.
Kalaam: npm semver; `Compile()` signature is the public contract.
CodeMas: URL prefix versioning (/api/v1/). Others: internal services — no external versioning needed.

---

**"Walk me through a scaling problem."**
Use the bottleneck tables in Section 6. State the current scale, name the first thing that breaks, explain WHY it breaks with numbers, then give the fix.

Example: "At 10× CodeMas load, the SSE connection state on a single Gunicorn instance saturates at ~50K open connections. Fix: dedicated SSE microservice in Node.js, or drop SSE for client polling — which is architecturally equivalent anyway after the Lambda migration."

---

**"How would you add observability to your RAG system?"**
the01.dev approach: (1) Audit log captures every inference with model + SOUL hash — correlation ID links question to answer to retrieved chunks. (2) RAGAS faithfulness score runs offline on a sample of turns — flags drift in answer quality over time. (3) Gate declination rate is a health metric — if it spikes, either the eval set is breaking retrieval or students are asking off-topic questions.

---

**"What's the hardest part of building with LLMs vs. traditional software?"**
"Traditional software: deterministic — same input, same output, easy to test. LLMs: probabilistic — same input, different output each time. This changes everything about testing (you test distributions, not exact values), observability (you log inputs and outputs, not just errors), and trust (you verify with tools, not model self-report). The SOUL.md pattern, the grounding gate, the audit log — all of these are adaptations to the non-deterministic nature of the model."

---

## 17. STAR Behavioral Stories

Use these when the interviewer says "tell me about a time when…" Pick the story that matches the question, then deliver it in under 2 minutes. Don't read it word for word — know the shape.

---

### Story 1 — Difficult Technical Decision Under Pressure
*Use for: "Tell me about a hard call you had to make" / "Tell me about a time you had to choose between two bad options"*

**Situation:** At CodeMas (Masai School), we were running Celery workers on a persistent server to execute student code. We had sandbox isolation using Docker — one container per submission. At exam close, all 10,000 students submitted simultaneously. The submission burst was overwhelming the queue.

**Task:** Evaluate migrating to AWS Lambda for the code execution layer. The architecture change was significant — Lambda is ephemeral, Celery workers are persistent. I had to make the call without a long evaluation window because an exam was coming.

**Action:** I mapped out the trade-off across 9 dimensions — isolation, scale, cold start, cost, delivery mechanism. The hardest part: Lambda gave up the Redis pub/sub channel that powered true SSE push. After the migration, SSE would become "polling theater" — Django polling Postgres every 500ms and forwarding. Architecturally equivalent to client polling. I documented this honestly before recommending migration. The isolation gain (fresh `/tmp` per invocation, no shared process state between students) and the elastic scale justified it.

**Result:** Migrated to Lambda + SQS FIFO. Zero submission processing failures at the next exam close. Infra overhead dropped to zero — Lambda scales itself. The SSE trade-off is documented and something I own in every conversation about the system.

---

### Story 2 — Solving a Problem Nobody Had Named Yet
*Use for: "Tell me about an innovative solution" / "Tell me about something you built from scratch"*

**Situation:** At Masai School, the plagiarism problem was framed as "are these submissions similar?" — pure code similarity. The system compared TF-IDF cosine similarity between submissions. Students had started sharing code in subtly modified forms to beat the threshold.

**Task:** Redesign the detection approach. The similarity-first framing was the wrong model of the problem.

**Action:** Reframed it: "Did this student write this code?" That's a different question. A student who copied would show behavioural anomalies — unusually high paste ratio, submission speed faster than the difficulty baseline, tab switches during the problem (looking at another window). Built a two-phase system: Phase 1 runs synchronously at exam close, scores four behavioural signals with weighted formula (paste ratio 0.40, speed anomaly 0.30, tab switches 0.15, attempt surprise 0.15). Only HIGH-confidence behavioural suspects become the K in Phase 2 — then we do O(K×N) similarity, not O(N²). This cut the comparison space dramatically while catching more edge cases.

**Result:** 19× improvement in catching cases over the previous similarity-only system. The key insight — behaviour flags what ML confirms.

---

### Story 3 — Owning an Architecture Limitation
*Use for: "Tell me about a failure" / "Tell me about a time you had to admit you were wrong"*

**Situation:** On the01.dev, I was routing the RAG tutor through the full Hermes agent loop — SOUL.md system prompt, MCP tool calls (search_course_content, generate_notes), HindSight memory. This is the "right" architecture. But in dev, the model is qwen2.5:3b — a 3-billion parameter model running on Ollama locally.

**Task:** The tutor was working for single-turn questions but multi-turn conversations were breaking — the model would answer Q2 without any reference to Q1's context. Students were getting disjointed answers.

**Action:** Investigated the ReAct loop. The issue: when Hermes routes through the agent loop, tool calls fragment the conversation context. The small model can't maintain coherent multi-turn state across tool invocations. The fix in the short term was to build a "reliable path" that bypasses the agent loop for dev — calls the model directly with conversation history injected, same SOUL, same retrieved context, just no tool orchestration overhead. The full Hermes routing works correctly on production-scale models.

**Result:** Multi-turn coherence restored in dev. But I documented this as a gap — HindSight memory is configured but only activates on the full Hermes path. I didn't hide it; I put it in the architecture doc so anyone reading the code understands the dev/prod difference. The lesson: validate agent loop behaviour on the model you'll actually run it with, not a larger one.

---

### Story 4 — Building for a User You're Not
*Use for: "Tell me about empathy in design" / "Tell me about a non-obvious constraint you solved"*

**Situation:** Built Kalaam — a programming language in Hindi, Marathi, Bengali, Telugu, and Odia — for tier-3 city students aged 14–18 with no laptop and intermittent internet. Most programming tools assume a desktop, internet, and English literacy. None of those apply.

**Task:** Design an interpreter and learning platform that works under these constraints without compromising on the actual learning outcome.

**Action:** Three key decisions driven entirely by the user, not by what's technically elegant: (1) Mobile-first PWA with cache-first service worker — works offline after first load, no app install needed. (2) Zero runtime dependencies in the npm package — no supply chain risk, no CDN calls. (3) Flat memory model instead of scoped environments — technically incorrect for a "real" language, but scope adds cognitive load that the target audience doesn't need yet. The interpreter's `ExecutionStack[]` records every operation in a replay-friendly format — the UI plays back the execution step-by-step in the student's language, replacing the need for a teacher to explain how the interpreter evaluates their code.

**Result:** 500+ monthly users, TEDx Bangalore talk, 5 languages supported. Adding a 6th language requires one keyword map entry and zero parser changes.

---

### Story 5 — Privacy as Architecture, Not Policy
*Use for: "Tell me about a security or privacy decision" / "Tell me about a constraint that changed your design"*

**Situation:** Munshi is a GST reconciliation agent for Bharatvarsh Arts, a performing arts company with ₹5Cr annual revenue. The owner wanted AI-assisted reconciliation but was clear: financial data — invoices, GSTIN numbers, transaction amounts — could not go to any external API.

**Task:** Build an AI agent that handles sensitive financial data within a non-negotiable sovereignty constraint.

**Action:** Designed the entire system local-first. Ollama runs the model on the owner's own machine. SQLite is the database — file on disk, owner controls it. FastMCP tools do all financial computation (compute_gst, convert_currency, lookup_hsn) in Python with Decimal arithmetic — the model calls the tool, the tool returns the number. The model only narrates; it never computes a financial figure. The SOUL.md has an explicit sovereignty clause: "Nothing leaves the machine. That is a promise, not a preference." This is enforced by architecture — there's no code path that makes an external API call for generation.

**Result:** Full GST reconciliation capability with zero data leaving the machine. The one design insight worth carrying: sovereignty is an architectural property, not a configuration option. If you can flip a flag and data goes to the cloud, it's not truly sovereign.

---

## 18. Business Impact — What You Actually Unlocked

When talking to EMs, PMs, or non-technical interviewers, translate every project into outcomes. Memorise these one-liners.

| Project | Business outcome | For whom |
|---|---|---|
| **CodeMas** | Each cohort: 400 students × ₹3L = ₹12Cr revenue unlocked per cohort. Plagiarism detection 19× better. AI features cut exam creation time by ~80%. | Masai School — India's largest coding bootcamp |
| **Munshi** | Hours of manual GST reconciliation per month → minutes. ₹5Cr in annual transactions handled with Decimal-accurate tax computation. Zero risk of data breach — nothing leaves the machine. | Bharatvarsh Arts — performing arts company |
| **the01.dev** | 8 deep CS courses generating recurring revenue. 67% fewer LLM inference calls via on-demand generation and smart caching. GEPA means tutor quality improves automatically over time. | 0.1% Dev — own product, Co-founder |
| **Kalaam** | 500+ monthly active learners from tier-3 cities who had no other entry point into programming. TEDx Bangalore talk. 5 Indian languages, zero internet required after first load. | Self-built open-source — npm `kalaam` v2.3.3 |
| **stringy-core** | Published npm library, 19 forks, actively maintained open-source contribution platform. Zero deps means zero supply chain risk for any project that installs it. | Open-source community |
| **Trade Compliance** | One query that would take a compliance analyst 2–3 hours of manual tariff lookup, FTA check, and anti-dumping research → done in 60–90 seconds, fully cited. | Exporters, importers, trade compliance teams |

---

### When pushed on revenue numbers

- ₹12Cr per cohort at Masai: 400 students × ₹3L fee = ₹12Cr gross revenue per cohort. CodeMas is the exam and assessment infrastructure that made each cohort possible at 10K concurrent scale.
- ₹2Cr for Bharatvarsh Arts: mentioned in context of Munshi + Trade Compliance being built to support their operations. This is their annual revenue figure, not a number we generated.
- 67% fewer LLM calls: on-demand artifact generation (notes, quiz, flashcards) generates content only when a student requests it, not upfront for every enrolled student. Compared to eager pre-generation.
- 80% exam creation time reduction: AI-assisted question generation, rubric writing, and difficulty calibration at CodeMas. Trainers review and approve rather than write from scratch.

---

## 19. Day-of Cheat Sheet (Read this last, right before you walk in)

---

**Who you are:** Full-stack + AI engineer, 6+ years. Built high-concurrency platforms, LLM agent systems, RAG pipelines, and a programming language. First engineer under a CTO, co-founder, open-source creator. You build things that ship.

---

**The 6 projects — one line each:**

| # | Project | One line |
|---|---|---|
| 1 | **CodeMas** | Real-time exam platform for 10K concurrent students — SQS + Lambda sandbox + two-phase plagiarism (behavioural first, then similarity) |
| 2 | **the01.dev** | Sovereign RAG tutor with self-evolving SOUL (GEPA), two-engine design (Hermes agent + LangGraph workflow), 100% local inference |
| 3 | **Munshi** | Local-first GST agent — nothing leaves the machine, Decimal arithmetic, HITL before any consequential action |
| 4 | **Kalaam** | Programming language in 5 Indian languages — 5-phase interpreter, offline PWA, zero deps, ExecutionStack = built-in teacher |
| 5 | **stringy-core** | Zero-dependency JS string utility library — 50+ functions, tree-shakeable ESM, Intl API, open-source contribution platform |
| 6 | **Trade Compliance** | Two-agent system (Researcher + Writer, both Hermes) — gather then synthesise, model-agnostic config, fully local |

---

**One claim + defense per project:**

- CodeMas → "19× improvement in catching plagiarism" → behavioural phase catches what similarity never sees
- the01.dev → "grounded or silent" → threshold 0.45 in code — model is never called below it
- Munshi → "nothing leaves the machine" → Ollama + SQLite + no external generation calls, architectural not policy
- Kalaam → "adding a language = 1 file change" → only Phase 1 is language-aware, parser sees normalized tokens
- stringy-core → "zero runtime deps" → no `dependencies` key in package.json, confirmed
- Trade Compliance → "model-agnostic" → change one line in config.yaml, no code change

---

**One toughest challenge per project:**

- CodeMas → SELECT FOR UPDATE for the race condition at submission burst
- the01.dev → small model loses multi-turn context in Hermes loop → reliable path bypasses agent loop in dev
- Munshi → 40–50% of invoices fail exact match → four-signal fuzzy pipeline, model only sees the 20% that are genuinely ambiguous
- Kalaam → error messages in English for students who read Hindi → known gap, scoped backlog
- stringy-core → ESM package in Jest (CommonJS default) → Babel transform in test env only
- Trade Compliance → agent tool loop → three-layer defence: SOUL + turn limit + MCP schema rejection

---

**If you get nervous:** pick the CodeMas submission pipeline or the 01dev RAG tutor pipeline and walk it step by step. You know both cold. Start with the user action (student submits code / student asks a question), walk through every system component in order, name the trade-offs at each step. That's a complete system design answer.
