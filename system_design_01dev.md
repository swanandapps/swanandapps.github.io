# the01.dev — System Design

**Platform:** AI-powered ed-tech platform — deep CS fundamentals video courses  
**Role:** Co-founder, led engineering (2-person team)  
**Stack:** React 18 + TypeScript, FastAPI, PostgreSQL + pgvector, LangGraph, GPT-4.1-mini

---

## Problem Statement

Deep CS fundamentals (compiler design, JS engine internals, real-time systems) are hard to learn passively. Students watch a video, get stuck, and have no way to ask questions grounded in the exact course content. The platform needed:
- A course assistant that answers questions grounded strictly in proprietary transcripts
- A content pipeline that converts raw PDFs/videos into structured lessons without manual curation
- A secure payment and access system protecting proprietary video content
- AI evaluation that improves over time

---

## Functional Requirements

- Browse courses and tracks (zero API calls for browsing)
- Purchase a track via Razorpay — atomic access provisioning
- Watch videos with chapter navigation, timestamp deep-links
- Ask questions — AI answers grounded in course transcripts with source citations
- Source citations link to exact video timestamp
- PDF → Lesson pipeline for content creators
- AI-generated practice questions with adaptive scheduling

## Non-Functional Requirements

- **RAG latency:** < 3 seconds for answer + sources (SSE streaming hides this)
- **Answer quality:** Zero hallucination — grounded only in proprietary content
- **Access security:** Purchased content inaccessible to non-buyers
- **Cost efficiency:** LLM called only when retrieval confidence is sufficient

---

## High-Level Architecture

```
Browser (React 18)
    │
    ├── Browse / tracks / course detail ──► Static JS data (tracks.js) — zero API calls
    │
    ├── POST /api/payments/razorpay-order ──► FastAPI ──► Razorpay (server-side)
    │
    ├── Video player ──► Firebase Storage (direct URL streaming)
    │
    └── AI assistant ──► FastAPI RAG service
                              │
                         ┌────┴─────────────────┐
                         │   pgvector (HNSW)    │
                         │   PostgreSQL          │
                         └────┬─────────────────┘
                              │ top-8 candidates
                         ┌────┴─────────────────┐
                         │   _rank_matches()    │
                         │   hybrid scoring     │
                         └────┬─────────────────┘
                              │ top-5, gate ≥ 0.15
                         ┌────┴─────────────────┐
                         │   GPT-4.1-mini       │
                         └────┬─────────────────┘
                              │ SSE stream
                         ┌────┴─────────────────┐
                         │  React SourceCard    │
                         │  (?t=timestamp link) │
                         └──────────────────────┘
```

---

## Detailed Component Design

### 1. Course Catalogue (Static Data, Zero API)

All course and track data lives in `tracks.js` — a local JavaScript file imported at build time.

```javascript
// Zero API call — instant browse
import { tracks } from './data/tracks.js'
```

**Why static local data:** Browse UX is instant (no network). Courses don't change frequently. New courses = update the file + deploy. Zero backend load for the most common user action.

---

### 2. RAG Course Assistant

#### Indexing Pipeline (on startup)

```
seed_transcripts.json
    │ chunk_courses()
    ├── 180-word windows
    ├── 35-word overlap
    └── step = 145 words
    │ embed_many() batch
    └── text-embedding-3-small (OpenAI)
    │ pgvector HNSW index
    └── PostgreSQL
```

**Chunking strategy:**
- 180-word windows: large enough for semantic coherence, small enough for precise citation
- 35-word overlap: ensures concepts that straddle chunk boundaries are captured
- Metadata per chunk: `{course_id, lecture_name, start_timestamp, end_timestamp}`

**Why pgvector with HNSW:**
- HNSW (Hierarchical Navigable Small World): O(log N) approximate nearest neighbour search
- Stored in PostgreSQL — same DB as the rest of the platform, no separate vector DB infra
- Scales to tens of thousands of chunks without infrastructure change

#### Query Pipeline

```
User question
    │ embed(question)
    │ text-embedding-3-small
    ▼
HNSW ANN search → top 8 candidates
    │
    ▼
_rank_matches() — hybrid scoring per chunk:
    semantic_score  = cosine similarity (weight 1.0)
    lexical_score   = Jaccard overlap on stop-word-filtered tokens
    final_score     = max(semantic_score, lexical_score)
    │
sort by final_score → top 5
    │
score gate: if max_score < 0.15 → return canned message, NO LLM call
    │
context blocks: [Course | Lecture | MM:SS-MM:SS]\nchunk_text
    │
GPT-4.1-mini (streamed via SSE)
    │
_to_source(): map chunks → Source {timestamp, snippet, score}
    │
React SourceCard → ?t=seconds deep-link into video player
```

**Why hybrid retrieval (cosine + Jaccard):**
- Pure semantic search misses exact keyword matches: function names, error messages, specific method signatures
- Pure lexical search misses paraphrased questions
- `max(semantic, lexical)` — either signal can promote a chunk independently. Recall improves without harming precision.

**Why score gate at 0.15:**
- Below this threshold, retrieval is ambiguous — LLM would hallucinate or generate generic answers
- Returns a canned "I don't have enough context" message — better UX than a confident wrong answer
- Eliminates LLM cost on off-topic or out-of-scope queries
- 0.15 was calibrated by testing against a sample of in-scope and out-of-scope questions

**Source timestamp deep-links:**
- `_to_source()` maps each chunk back to `{start_timestamp}` from its metadata
- React SourceCard renders as a clickable card: "[Lecture Name | 05:42]"
- Clicking navigates the video player to `?t=342` (seconds)
- Students jump to the exact moment being referenced — not just the lecture

---

### 3. LangGraph Multi-Agent Content Pipeline

For content creators uploading PDFs to generate structured lessons.

```
PDF upload
    │
Planner agent (LangGraph)
    │ decomposes into lesson structure
    │ {intro, concepts[], examples[], exercises[]}
    │
Human approval (HITL suspend/resume)
    │ creator reviews plan
    │ approves / modifies
    │
Executor agents (parallel, one per section)
    │ generate each section independently
    │
Supervisor agent
    │ reviews each output
    │ routes failed sections back to executor
    │ assembles final lesson on all-pass
    │
lesson stored in DB
```

**Why Planner-Executor separation:**
- Planner is cheap (fast, small output) — bad plans are caught before expensive executor runs
- Human checkpoint after planning — creator steers content before any generation cost
- ~67% reduction in LLM calls vs generating everything speculatively

**HITL implementation in LangGraph:**
- Graph execution pauses at `interrupt()` node after planning
- State is serialised and stored (checkpoint)
- Creator reviews via UI and submits approval
- Graph resumes from checkpoint with approved plan

**Supervisor routing:**
- Each executor output is scored for completeness and coherence
- Below threshold → re-run executor with refined prompt
- Prevents cascading bad content — one failed section doesn't block others

---

### 4. Payments & Access

```
Buy Track button
    │
POST /api/payments/razorpay-order (FastAPI)
    │ creates order server-side (secret key never leaves FastAPI)
    │ returns {order_id, key_id (public)}
    │
Razorpay checkout.js modal (client)
    │ user completes payment
    │
payment_success callback
    │ addCourse(track_id) → localStorage update (immediate)
    │ production path: Firestore arrayUnion per uid
    │
hasCourse(id) check on video player → chapters 3+ gated
```

**Why server-side order creation:**
- Razorpay secret key must never reach the browser — exposes the ability to create arbitrary orders
- Client only receives `order_id` and public `key_id` — cannot create orders independently

**Prices in paise (integers):**
- ₹999 stored as `99900` — integer arithmetic, zero float precision issues
- Common bug: `₹333.33 × 3 = ₹999.99` in float arithmetic. Integer paise eliminates this class of error.

**Video protection:**
- Firebase Storage URLs are signed and time-limited
- XOR encoding on video IDs — obfuscates direct URL construction
- Firebase App Check — prevents API calls from non-app clients

---

### 5. Data Model

```sql
courses (id, title, description, track_id, duration_seconds, order_index)
lectures (id, course_id, title, timestamp_start, timestamp_end, transcript_text)
chunks (id, lecture_id, text, embedding vector(1536), start_ts, end_ts, course_id)
purchases (id, user_id, track_id, razorpay_order_id, amount_paise, created_at)
```

**pgvector index:**
```sql
CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

---

### 6. Key Engineering Decisions

| Decision | Choice | Alternative | Why |
|---|---|---|---|
| Course data | Static JS file | API endpoint | Instant browse, zero backend load |
| Vector store | pgvector HNSW | Pinecone / Weaviate | Same DB, no extra infra, O(log N) search |
| Retrieval | Hybrid cosine + Jaccard | Pure semantic | Catches exact keyword matches semantic misses |
| LLM gate | Score ≥ 0.15 | Always call LLM | Cost control + hallucination prevention |
| Streaming | SSE | REST polling | Progressive answer delivery, perceived faster |
| Multi-agent | LangGraph | Custom orchestration | Built-in state, checkpoints, HITL support |
| Payments | Razorpay server-side order | Client-side | Secret key protection |
| Prices | Integer paise | Float rupees | Float precision errors in financial arithmetic |

---

### 7. Scalability Considerations

**RAG at 10K courses:**
- HNSW search: O(log N) — stays fast at 10× scale
- Re-indexing on new course: incremental insert to pgvector — no full rebuild
- Embedding cost: batch embed on course upload, not at query time

**Concurrent users:**
- FastAPI is async — handles many concurrent SSE streams
- pgvector queries are read-only — add read replicas under high load
- LLM calls are the bottleneck — rate limit per user, queue on high load

**Multi-tenant (multiple ed-tech clients):**
- Add `tenant_id` to all tables, Postgres RLS per tenant
- Separate pgvector index per tenant — prevent cross-tenant retrieval

---

### 8. What I'd Do Differently

- **Evaluation loop (RAGAS):** Measure faithfulness, relevance, retrieval quality on a golden QA set. Currently no automated eval — quality is assessed by human review.
- **Streaming embeddings:** For long transcripts, embed incrementally rather than chunking entire library on startup
- **Re-ranking with cross-encoder:** After HNSW retrieval, a cross-encoder re-ranker would improve precision at the cost of ~50ms additional latency — worth it for complex multi-part questions
- **User query logging:** Log queries + scores (with consent) to identify systematic gaps in transcript coverage
