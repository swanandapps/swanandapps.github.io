# Swanand Kadam — Engineering Profile

**Use this document to:** instantly understand who Swanand is as an engineer, what he has built, the technical decisions behind each project, and how to help him in interviews, system design discussions, or strategic conversations.

---

## Who He Is

Swanand Kadam is a full-stack engineer whose work sits at the intersection of AI systems, scalable infrastructure, and real business impact. He built India's first Hindi programming language, led engineering at a coding assessment platform serving 10,000 concurrent students, co-founded an AI-powered ed-tech company, and shipped sovereign local-first AI agents for financial compliance. He has been consistently early — building on runtimes and protocols within weeks of their launch — and his best work solves problems for people who had no tool at all, not just a better tool.

**Interview positioning:** "Systems that scale. AI that ships. Code that matters."

**Role pattern:** First engineer or co-founder — high ownership, architecture decisions his own, reported directly to CTO or no one.

---

## Engineering Traits (Consistent Across All Projects)

These patterns repeat across every project and reveal how he thinks:

**1. Infrastructure over application**
He builds the layer others build on top of. Kalaam is a language runtime. stringy-core is a utility library. Munshi is agent infrastructure for financial compliance. He thinks like a platform engineer even when building products.

**2. Reframing the problem before building**
CodeMas plagiarism: instead of "are two submissions similar?" he asked "did this student write this?" — a completely different framing that changed the entire detection approach. This is his most consistent intellectual move.

**3. AI is async, never on the critical path**
In CodeMas, all 5 AI-powered features (rubric scoring, Socratic hints, exam generation, narratives, plagiarism) are async and triggered by events. None sit between submission and result. Zero latency added to the student experience. He designs AI as an additive layer, not a blocker.

**4. Sovereignty and local-first as design principles**
Munshi: nothing leaves the machine. Financial data, model inference, and results all stay local. This wasn't a constraint imposed on him — he chose it because sensitive financial data shouldn't move over the network. Privacy by architecture, not policy.

**5. Correctness over convenience**
Munshi uses Python's `Decimal` type for all tax arithmetic, never floats. GST computation across thousands of invoices accumulates floating-point errors that compound into wrong filings. He caught and fixed this class of bug at design time, not after.

**6. Human-in-the-loop as a first-class design concern**
Munshi requires explicit human approval before any consequential action — filing, reconciliation verdict, ITC claim. The audit trail captures every model decision, tool call, and computation in plain English. Non-technical owners can verify everything before submission. He built HITL consciously, not as an afterthought.

**7. The zeroth step**
Kalaam wasn't a better coding tool — it was the tool for people who had no tool at all. Tier-3 city students with no laptop, intermittent internet, and no English. He finds the excluded category and builds for them.

**8. Scale awareness from day one**
Every architecture decision in CodeMas traces back to 10,000 concurrent students. SSE over WebSockets because SSE is Lambda-compatible. SQS FIFO because it decouples execution from the API during deadline bursts. SELECT FOR UPDATE because race conditions happen when 400 students submit at the exam deadline simultaneously.

---

## Projects

---

### 1. CodeMas — AI Coding Assessment Platform

**What it is:** Real-time coding exam platform. Students write code in a browser, submit, and get results in under 10 seconds. Trainers monitor live, detect plagiarism, generate exams with AI.

**Role:** First engineer reporting to CTO at Masai School.

**Scale:** 10,000 concurrent students. This number informed every design decision.

**Business impact:** Automation eliminated manual exam oversight, making multi-cohort operation viable. Each cohort = 400 students × ₹3L = **$2M+ in revenue per cohort unlocked for Masai.**

---

#### Architecture Migration: Redis + Celery + Docker → Lambda + SQS

The original execution layer used Redis LPUSH, Celery workers (BRPOP), and Docker containers running on a persistent host. This was migrated to AWS Lambda + SQS FIFO.

**Trade-offs made in the migration:**

| | Redis + Celery + Docker | Lambda + SQS FIFO |
|---|---|---|
| Isolation | Docker per container (shared host) | Per-invocation (truly ephemeral) |
| Pub/sub | Redis pub/sub for result delivery | Postgres poll (500ms) |
| Host management | Persistent workers, scaling manual | Zero — Lambda scales automatically |
| Deadline bursts | Celery queue backs up | SQS absorbs, Lambda fans out |
| Cost model | Always-on workers | Pay per execution |

Lost: persistent pub/sub for real-time result streaming (replaced by Postgres polling — acceptable 500ms latency).
Gained: zero host management, true per-invocation isolation, elastic scale at exam deadline bursts.

#### Submission Pipeline (Current Architecture)

```
Student → POST /submit → Django API → INSERT Submission → Enqueue to SQS → 201
Browser opens SSE stream
SQS → Lambda (dequeues) → Executes code (Lambda IS the sandbox) → Writes Result to Postgres
SSE endpoint polls Postgres every 500ms → status change detected → push result → stream closes
```

**Key decisions:**

- **Lambda as the sandbox** — Every submission executes in its own Lambda invocation. Fully isolated, ephemeral, hard 15-second timeout. Lambda's per-invocation isolation gives sandboxing without running Docker on a persistent host. No shared state between executions.

- **SQS FIFO** — Decouples execution from the API. During exam deadline (400 students submit in 30 seconds), the queue absorbs the burst without stalling the API. Each exam gets its own queue for isolation.

- **SSE over WebSockets** — Code execution results are unidirectional (server pushes once). SSE chosen for: native browser reconnection, HTTP/1.1 proxy compatibility, and simpler Lambda scaling. No WebSocket lifecycle management needed.

- **SELECT FOR UPDATE for attempt gating** — Before accepting a submission, the API checks attempt count with `SELECT FOR UPDATE`. Prevents race conditions when two tab submissions arrive within milliseconds — common at exam end. This is the critical correctness guarantee.

- **201 immediate response** — The API never waits for execution. Returns 201 as soon as the submission is queued. Browser then opens SSE. This keeps API latency flat regardless of execution time.

---

#### AI Features Layer (5 async LLM workflows, zero latency impact)

All 5 are async, triggered by events, and execute entirely off the submission critical path. They are additive — results appear after the student already has their execution result. These are not agents — each is a single-shot LLM call triggered by an event, not an autonomous system with memory or tool use.

| Feature | Trigger | What it does |
|---|---|---|
| **Rubric Scoring** | Every failed submission | GPT grades on correctness, code quality, approach, edge cases (0–2 each) with per-dimension justifications |
| **Socratic Hint** | Failed submissions | GPT generates a pedagogical nudge pointing at the concept without revealing the answer |
| **Exam Generator** | Trainer request | GPT drafts complete exams (questions + test cases) from topic and difficulty; human reviews before any persistence |
| **Trainer Narrative** | Cohort summary request | AI-generated cohort summaries and per-student performance narratives |
| **Plagiarism Engine** | Exam close (pre_save signal) | Two-phase detection — behavioral signals + TF-IDF (not LLM-based) |

All 5 features share a single GPT-4o-mini endpoint, are idempotent, and write results to Postgres only.

---

#### Plagiarism Detection System (The Reframing)

**The insight:** Reframed from "are two submissions similar?" to "did this student write this?" Completely different problem — the second question uses behavioral evidence, not just code comparison.

**Phase 1 — Behavioural (runs synchronously at exam close):**
Scores four signals per submission:
- Paste ratio: `paste_char_count / total_chars` (weight 0.40)
- Speed vs difficulty baseline: `time_to_complete / baseline_for_difficulty` (weight 0.30)
- Tab switches: normalized count (weight 0.15)
- Attempt surprise: how late the passing attempt came (weight 0.15)

Risk score > 0.2 → writes BehaviouralFlag. Runs in O(N) time — one pass over all submissions.

**Phase 2 — Similarity (queued per question after Phase 1):**
TF-IDF cosine similarity. The key insight: only HIGH-confidence behavioural suspects are compared against the full cohort — O(K×N) not O(N²). Flags pairs above 0.80 similarity.

**Why TF-IDF over CodeBERT/MOSS/AST:**
Fast, no GPU required, interpretable output for trainers, accurate enough at 2,000 submissions. CodeBERT would require GPU infrastructure. MOSS has rate limits. AST comparison is language-specific. TF-IDF is consistent across Python, JS, Java, C++.

**Triggered by:** Django `pre_save` signal on Exam model when `is_active` transitions True→False.

**Result:** Detection rate 5% → 95% — a 19× lift.

---

#### Tech Stack

Django 4.2 + DRF, Simple JWT, AWS Lambda (sandbox + API), AWS SQS FIFO, SSE (text/event-stream), GPT-4o-mini, PostgreSQL (ACID), Vue 3 + Vite (frontend), CodeMirror 6 (editor), ECharts (trainer dashboard)

---

### 2. the01.dev — AI-Powered Ed-Tech Platform

**What it is:** Full-stack ed-tech platform selling deep CS fundamentals video courses. 8 courses — Build a Programming Language, Build a Custom Library, JS Engine Internals, etc. Rewritten in React 18 + FastAPI with a RAG-powered AI learning assistant and a multi-agent content pipeline.

**Role:** Co-founder. Led engineering (2-person team). Architecture owned end-to-end.

---

#### Multi-Agent Content Pipeline

**Pattern:** Planner-Executor with Supervisor-Worker and human-in-the-loop suspend/resume.

Built with LangGraph. PDF-to-Lesson pipeline:
1. Planner agent decomposes a PDF into a lesson structure
2. Human reviews and approves the plan (HITL suspend/resume)
3. Executor agents generate each section in parallel
4. Supervisor reviews output, routes failed sections back to workers

**Impact:** ~67% reduction in unnecessary LLM calls via on-demand generation (only generate what gets approved).

---

#### RAG Course Assistant

**Stack:** pgvector with HNSW indexing, hybrid retrieval, GPT-4.1-mini, SSE streaming.

**Pipeline:**
1. **Indexing:** `bootstrap()` → load `seed_transcripts.json` → `chunk_courses()` (180-word windows, 35-word overlap, step=145) → `embed_many()` batch → pgvector HNSW index (PostgreSQL)
2. **Query:** `embed(question)` → HNSW ANN search (top 8) → `_rank_matches()`: semantic (weight 1.0) + lexical overlap (Jaccard, stop-word filtered) → `max(semantic, lexical)` per chunk → top 5 → gate ≥ 0.15
3. **Answer:** context blocks `[Course | Lecture | MM:SS-MM:SS]\ntext` → GPT-4.1-mini → `_to_source()` maps chunks to `Source {timestamp, snippet, score}` → React SourceCard deep-links into video player at exact second

**Key decisions:**
- **Hybrid cosine + BM25 (Jaccard):** Pure semantic search misses exact keyword matches (function names, error messages). Hybrid catches both. The `max()` combination means either signal can promote a chunk.
- **Score gate ≥ 0.15:** Below this threshold, no LLM call — returns a canned "not enough context" message. Saves cost, prevents hallucination. LLM is never called on ambiguous retrieval.
- **pgvector HNSW:** Efficient approximate nearest neighbor search. Scales without full-table scan.
- **Source timestamps:** Answers deep-link to the exact second in the video. Not just "this lecture" but `?t=342` on the player URL.

**Fallback:** Local SHA-256 hash-projection embedding + extractive answer. No API key needed for dev.

---

#### Payments and Access

- Razorpay: server-side order creation only (secret never leaves FastAPI). Client receives `order_id` + public `key_id`. Razorpay checkout.js modal on client.
- Prices stored in paise (integers) to avoid float precision issues.
- Atomic access provisioning: `addCourse()` updates localStorage immediately on payment success. Production path: Firestore `arrayUnion` per uid.
- XOR-encoded video protection + Firebase App Check for proprietary content.

---

#### Tech Stack

React 18 + TypeScript + Vite + Tailwind + TanStack Query v5 + React Router v6, FastAPI (Python), OpenAI SDK (text-embedding-3-small + gpt-4.1-mini), pgvector (HNSW), LangGraph, Razorpay, Firebase (Auth + Storage), HTML5 video streaming

---

### 3. Munshi — Sovereign GST & Trade Compliance Agent

**What it is:** Local-first AI agent for GST reconciliation. Built for Bharatvarsh Arts (₹5Cr/$500K+ revenue business). Replaces manual Excel workflows with plain-English queries. Nothing ever leaves the machine.

**Business impact:** Entire GST, invoicing, and reconciliation workflow automated for a ₹5Cr business.

---

#### The Core Problem

GSTR-2B reconciliation: matching your purchase invoices against what your suppliers filed with the government. If they match, you can claim Input Tax Credit (ITC). Missed matches = money forfeited.

Standard tools do exact string matching. Vendor names are messy. "Tata Steel Ltd" vs "TATA STEEL LIMITED" — same supplier, exact matching fails, ITC lost.

---

#### Pipeline

```
Purchase Invoices + GSTR-2B → Fuzzy Matcher → ambiguous cases → Hermes Agent (model judges)
→ Decimal Compute (exact rupees) → Audit Trail → Human Approval → Filing
```

**Fuzzy matching:** Normalize vendor names, GSTIN prefix matching, amount tolerance ±2%. Auto-matches the clear cases.

**Model judgment:** Only ambiguous cases go to the model. The model explains its reasoning in plain English. It never computes tax — it only judges "is this the same supplier?"

**Decimal arithmetic:** All tax computation uses Python's `Decimal` type. Never floats. IEEE 754 accumulates errors across thousands of invoices that compound into incorrect filings. Decimal eliminates this class of bug entirely.

**Human-in-the-loop:** Every consequential action — filing, reconciliation verdict, ITC claim — requires explicit owner approval. The audit trail captures every model decision, tool call, and computation outcome.

---

#### Sovereign Architecture

Everything runs on the local machine:
- **Ollama** runs model inference locally (Qwen, Llama, or any compatible model)
- **FastAPI** handles the REST API and persistence
- **MCP tools** fetch GSTR-2B data, read invoice files, compute tax, write audit logs
- **Hermes runtime** manages the agent loop, memory, and tool calls
- **No data leaves the machine** — privacy by architecture

The model is a tool that judges ambiguous matches. It is not a service. Financial data never travels over a network.

---

#### Tech Stack

Hermes runtime (Nous Research, MIT), MCP tools (custom), FastAPI, Ollama (local inference), Python Decimal, Docker (container), SQLite + local files (persistence)

---

### 4. Trade Compliance Researcher — Multi-Agent System

**What it is:** Two-agent pipeline (Researcher → Writer) built on the Hermes runtime. Gathers trade-compliance regulatory data via MCP tools and synthesises it into structured reports. Model-agnostic.

---

#### Architecture

**Researcher agent:** Gathers regulatory data via MCP tools (regulation search, tariff DB lookup, document fetcher). Calls tools iteratively. Memory persists across turns.

**Writer agent:** Receives Researcher's findings. Synthesises into a structured report with citations and sources.

Both agents share conversation memory across turns via Hermes. Neither handles the other's task — separation of concerns is enforced by SOUL.md identity files.

**Model swap:** The model endpoint is one key in `config.yaml`. Switching from Ollama/Qwen to OpenAI or Groq changes that one line. Zero code changes.

**SOUL.md pattern:** Agent identity, persona, tone, and operating constraints declared in a markdown file. `config.yaml` declares capabilities and tools. No agent plumbing written from scratch.

---

#### Tech Stack

Hermes runtime, MCP tools, Ollama (default), Docker Compose (full stack), Python

---

### 5. Kalaam — India's First Hindi Programming Language

**What it is:** A programming language in Hindi, Marathi, Bengali, Telugu, and Odia. Mobile-first, fully offline, zero API calls. Built for tier-3 city students aged 14–18 who have no laptop and intermittent internet. The "zeroth step" into programming.

**Published:** npm package `kalaam` v2.3.3 (zero runtime dependencies). Frontend at kalaamlang.in.

**Recognition:** TEDx Bangalore speaker — "Why we should be able to code in our own languages."

**Users:** 500+ monthly active users.

---

#### Interpreter Pipeline (5 Phases, Pure JavaScript)

```
Source Code (any language) → Phase 1: Cleaning → Phase 2: Scanning → Phase 3: Tokenizing → Phase 4: Interpretation → kalaam{} output
```

**Phase 1 — Cleaning:** `earlyCleaning()` + keyword substitution. Language-specific keywords (Hindi, Marathi, etc.) are substituted into normalized tokens before any parsing. This is the only phase that knows about language. Everything after this is language-agnostic.

**Phase 2 — Scanning:** Character-level scan → `cleaned_sourcedata[]`

**Phase 3 — Tokenizing:** `cleaned_sourcedata[]` → typed `tokens[]` via 20 `Push*` functions + TypeChecking pass.

**Phase 4 — Interpretation:** Walk `tokens[]`, execute against `memory{}`, build `ExecutionStack[]`.

**Output:** `kalaam{}` object: `{ output, ExecutionStack, isError, TimeTaken }`

---

#### Key Innovations

**Adding a new language = 1 keyword map entry, zero parser changes.** The interpreter is language-agnostic. Phase 1 handles substitution. Adding Telugu required no parser changes — just one keyword map object.

**Learning Mode (ExecutionStack[]):** Every operation appends to `ExecutionStack[]`. The UI replays it line-by-line in the student's language, showing what each line does and why. Students see how the interpreter evaluates their code. No teacher required. This is the key pedagogical innovation — the interpreter teaches itself.

**Zero runtime dependencies.** The npm package has no dependencies. Works completely offline. Any environment that runs JavaScript can run Kalaam.

**PWA with service-worker cache.** After first load, no internet needed. Runs on a budget Android phone with no connectivity. This is the physical reach requirement — tier-3 cities have intermittent internet.

---

#### Public API

```javascript
import { Compile } from 'kalaam'
const result = Compile(sourcecode, languageKeywords)
// returns: { output, ExecutionStack, isError, TimeTaken }
```

---

#### Tech Stack

Pure JavaScript interpreter (zero deps), Vue 2 + Quasar (PWA), CodeMirror (custom Kalaam syntax mode), npm package `kalaam`, Jest test suite, Service Worker (offline cache)

---

### 6. Bharatvarsh.art — D2C E-Commerce Platform

**What it is:** Indian cultural wall art sold direct-to-consumer online. Built and led engineering end-to-end — architecture, payments, inventory, and delivery integrations from scratch.

**Business impact:** ₹2Cr+ in revenue.

**Key insight from this project:** The printer (supplier) made more money than the poster seller. Infrastructure businesses — where others build on top of you and do the marketing — are more defensible than application businesses. This insight shapes how he thinks about product strategy.

---

### 7. stringy-core — JavaScript String Utility Library

**What it is:** Zero-dependency JavaScript string utility library. "Lodash but for strings." 50+ pure functions across 9 modules. Published to npm as `stringy-core`. Built as an open-source contribution platform.

**Traction:** 19 forks. Published on npm.

---

#### Architecture

**Two export patterns:**
1. Named imports: `import { maskEmail } from 'stringy-core'` — tree-shakeable, bundler-optimized
2. Namespace: `import { _s } from 'stringy-core'` — convenience, all functions on `_s`

**9 modules:** textCaseManipulation, textCleaning, textFormatting (all via `Intl` API), textMaskingAndSecurity, textMetadataAndExtraction (15 regex extractors), textAnalysisAndValidation, textTransformations, textSpecializedOperations (levenshteinDistance via DP matrix), textGeneration

**Tooling:** ESM (`"type": "module"`), Babel (ESM→CJS for Jest), Husky + lint-staged (pre-commit ESLint + Prettier)

---

## System Design Decision Patterns

These are the trade-off decisions that appear across projects and reveal his engineering judgment:

| Decision | Choice | Why |
|---|---|---|
| Real-time delivery | SSE over WebSockets | Unidirectional push, native reconnection, proxy-compatible, Lambda-friendly |
| Code execution isolation | Lambda invocation | Ephemeral per-invocation isolation without Docker host management |
| Queue for async work | SQS FIFO | Absorbs deadline bursts, decouples execution, per-exam isolation |
| Concurrency safety | SELECT FOR UPDATE | Prevents attempt-count race conditions at exam deadline |
| Tax arithmetic | Python Decimal | Float accumulation errors corrupt tax filings at scale |
| AI placement | Async, event-triggered | Never on the critical path. Additive, not blocking. |
| Plagiarism detection | Behavioral + TF-IDF | O(K×N) not O(N²). Behavioral signals first, similarity only on suspects. |
| RAG retrieval | Hybrid cosine + Jaccard | Pure semantic misses exact keyword matches. Max() combination keeps both signals. |
| Model selection | GPT-4o-mini / local Ollama | Cost-efficient for async LLM features (CodeMas). Local for sensitive financial data (Munshi). |
| Financial data handling | Local-first, no network | Privacy by architecture. Sensitive data never transmitted. |
| Agent identity | SOUL.md + config.yaml | Declarative, no plumbing from scratch. Separation of agent identity from capabilities. |
| Embeddings fallback | SHA-256 hash projection | Dev environment works without API key. No external dependency for local dev. |

---

## Interview Reference

### 40-Second Intro

> "I'm Swanand — software engineer focused on AI systems and scalable platforms. At Masai School I was the first engineer under the CTO, building a real-time coding exam platform for 10,000 concurrent students with AI-powered features for automated grading, plagiarism detection, and exam generation. I also co-founded the01.dev — an ed-tech platform with a multi-agent content pipeline and a RAG course assistant. And I built Kalaam, a programming language in five Indian languages, which got me a TEDx Bangalore invite. I'm looking for a role where I'm building things with real product stakes — AI systems, backend infra, full stack."

### Strongest Talking Points

1. **CodeMas plagiarism reframing** — "We reframed the problem from 'are submissions similar?' to 'did this student write this?' That changed the entire detection approach — behavioral signals (paste ratio, speed, tab switches) run first in O(N), then TF-IDF cosine only on suspects, O(K×N) not O(N²). Detection went from 5% to 95%."

2. **Lambda as sandbox decision** — "We use Lambda invocations as the execution sandbox. Each submission gets its own invocation — ephemeral, isolated, hard timeout. No Docker to manage on a persistent host. SQS absorbs the burst when 400 students submit at the exam deadline."

3. **AI async design principle** — "All 5 AI features in CodeMas are async and event-triggered. None sit between submission and result. Students get their execution result in under 10 seconds. AI enrichment appears after. Zero latency impact."

4. **Kalaam's zeroth step insight** — "The question wasn't 'how do we make coding easier?' It was 'who can't code at all, and why?' Tier-3 city students had no laptop, no English, no internet. Kalaam is Hindi/Marathi/Bengali/Telugu/Odia, offline, runs in the browser on a budget Android. The zeroth step."

5. **Munshi sovereignty** — "Financial data is sensitive. Nothing leaves the machine. Ollama runs local inference. The model only judges ambiguous invoice matches — it never computes tax. All arithmetic is Python Decimal, never float. The audit trail is in plain English so a non-technical owner can verify every decision before filing."

### Common Interview Questions

**"Walk me through the CodeMas submission pipeline."**
POST /submit → Django checks attempts (SELECT FOR UPDATE) → INSERT Submission → enqueue SQS → 201 → browser opens SSE. Lambda dequeues → executes code in isolated invocation → writes Result rows to Postgres → updates Submission status → SSE polls Postgres (500ms) → status change detected → push result → stream closes.

**"Why SSE and not WebSockets?"**
Code execution is unidirectional — server pushes once. SSE gives you native browser reconnection (EventSource), works through HTTP/1.1 proxies without upgrade negotiation, and scales naturally on Lambda (no persistent connection management). WebSockets are the right tool for bidirectional communication. This isn't that.

**"How did you handle 10,000 concurrent students?"**
Every design decision traces back to that number. Lambda scales horizontally per invocation. SQS absorbs bursts. SELECT FOR UPDATE prevents race conditions. SSE keeps connections cheap. The AI features are completely off the critical path — they can't degrade the submission experience.

**"What's the hardest technical problem you've solved?"**
The plagiarism system reframing. The naive approach (compare every submission pair) is O(N²) — at 2,000 submissions it's 4 million comparisons. We flipped the model: behavioral signals score each submission independently in O(N), flagging suspects. TF-IDF cosine only runs on suspects against the full cohort — O(K×N) where K is small. Combined with the conceptual reframe ("did this student write this?"), it went from 5% to 95% detection accuracy.

---

## What He's Currently Thinking About

Actively exploring what comes next in the open-source AI infrastructure space. Interested in problems that are genuinely first — not gap analysis on the existing stack, but building for where AI is heading. Specifically interested in:
- The coordination/trust layer for multi-agent systems
- Infrastructure that becomes mandatory as agents proliferate
- The "zeroth step" problem for AI in emerging markets

He does not want to build products that are already being built by well-funded teams. He wants to be first the way Kalaam was first.
