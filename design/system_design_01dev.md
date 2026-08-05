# the01.dev — System Design

**Platform:** AI-powered ed-tech platform — deep CS fundamentals video courses  
**Role:** Co-founder, led engineering (2-person team)  
**Stack:** React 19 + TypeScript + Vite + Tailwind, FastAPI 0.115, Hermes Agent v0.18.2, LangGraph 1.2.9, Ollama / vLLM, RAGAS 0.2.14  
**Tagline:** "Become 'THAT' Developer." — 8 courses, paying students, real product.

---

## 1. Quick Reference

| Dimension | Value |
|---|---|
| Courses | 8 (Build a Programming Language, JS Engine Internals, Build a Real-Time System, Build a Custom Library, and more) |
| AI features | 7 (Sovereign RAG Tutor, GEPA, RAGAS Eval, QuizMe, Artifact Generation, MCP Tools, Session Memory) |
| Frontend port | 5173 (Vite) |
| Backend port | 8080 (FastAPI) |
| QuizMe / workflow port | 8090 (LangGraph) |
| Hermes gateway port | 8642 |
| Ollama port | 11434 |
| RAG grounding threshold | 0.45 |
| Embedding model | OpenAI text-embedding-3-small (retrieval) |
| Generation model | Ollama qwen2.5:3b (dev) / vLLM on GPU (prod) |
| RAGAS judge | Local model — zero external calls |
| Storage (dev) | In-memory (lost on restart) |
| Storage (prod) | Firestore. Target: Postgres + pgvector |
| Audit | EU-AI-Act row written BEFORE every inference |

---

## 2. Problem Statement

Deep CS fundamentals — compilers, JS engine internals, real-time systems — are hard to learn passively. Students watch a lecture, get stuck on a concept, and have nowhere to ask a question grounded in the exact content they just consumed. Generic AI assistants hallucinate or give surface-level answers that don't connect to the course material.

The platform needed:

1. A course tutor that answers grounded strictly in proprietary transcripts, stays silent when it can't, and never invents facts.
2. An AI tutor identity (SOUL) that can improve its own behaviour over time without human intervention in the critical path.
3. Structured practice workflows (quiz generation) with human approval gates — not a chatbot guessing at what to test.
4. Artifact generation (notes, flashcards, summaries) from course material, auditable and resumable.
5. A secure payment and access system where proprietary video content is protected server-side.
6. Sovereign inference — student conversations on local hardware, not third-party cloud by default.
7. An evaluation system that measures its own answer quality without calling external judge models.

---

## 3. Functional Requirements

- Browse 8 courses and tracks — instant, zero API calls
- Purchase a track via Razorpay — atomic access provisioning in Firestore
- Stream videos with chapter navigation and timestamp deep-links
- Ask the Sovereign RAG Tutor questions — AI answers grounded in transcripts, citations link to exact video seconds
- Generate a personalised quiz via QuizMe — student approves study plan, then answers per-objective, gets scored feedback
- Generate study artifacts (notes, summary, quiz, flashcards) from PDF/topic
- Session memory: tutor remembers context across turns in a session
- GDPR erasure: DELETE /api/hermes/memory/{id} purges all student turns and profile
- GEPA: self-evolving SOUL identity that improves offline, requires human approval before deployment
- RAGAS evaluation: quality scoring on each RAG answer, sovereign (no external calls)

---

## 4. Non-Functional Requirements

- **RAG latency:** < 3 s to first token (SSE streaming hides end-to-end latency)
- **Grounded or silent:** below threshold 0.45, the model is never called — a deterministic DECLINE is issued in code
- **Access security:** purchased content inaccessible to non-buyers (Firebase Auth + Firestore purchase check); video URLs XOR+Base64 encoded in bundle — not plain text; Razorpay secret never leaves FastAPI
- **Cost efficiency:** generation on local hardware (Ollama / vLLM) — only retrieval embeddings call OpenAI
- **Sovereignty:** student conversations and quiz content generated on local hardware by default; ONE remaining external call is the retrieval embedding
- **Audit completeness:** every inference request is logged with student id, question, model, provider, SOUL hash — BEFORE inference, so the log is tamper-evident
- **Resumability:** LangGraph workflows survive process restart via MemorySaver checkpointer
- **GDPR compliance:** GET /api/hermes/memory/{id} for transparency; DELETE for erasure

---

## 5. High-Level Architecture

```
Browser (React 19 + Vite :5173)
    │
    ├── Browse / tracks / course detail ──► Static JS data (tracks.js) — zero API
    │
    ├── Auth ──────────────────────────────► Firebase Auth (Google Sign-In)
    │                                              │
    │                                        firebase-admin (FastAPI)
    │
    ├── POST /api/payments/* ───────────────► FastAPI :8080 ──► Razorpay
    │                                              │
    │                                         Firestore (prod)
    │
    ├── Video player ───────────────────────► Firebase Storage
    │   (XOR+Base64 decode client-side before src set)
    │
    ├── POST /api/hermes/chat ──────────────► FastAPI :8080
    │   (RAG Tutor, SSE stream)                    │
    │                                    ┌──────────┴──────────────┐
    │                                    │   SOVEREIGN RAG TUTOR   │
    │                                    │  ┌─────────────────┐    │
    │                                    │  │  Access Gate    │    │
    │                                    │  │  Audit Row      │    │
    │                                    │  │  Memory Load    │    │
    │                                    │  │  Embed (OpenAI) │    │
    │                                    │  │  HNSW ANN top-4 │    │
    │                                    │  │  Score Gate 0.45│    │
    │                                    │  │  Local Model    │    │
    │                                    │  │  SSE Stream     │    │
    │                                    │  └─────────────────┘    │
    │                                    └─────────────────────────┘
    │
    ├── POST /api/hermes/gepa/* ─────────►  FastAPI :8080
    │   (GEPA self-evolution)                      │
    │                                     ┌────────┴────────┐
    │                                     │   GEPA LOOP     │
    │                                     │  eval → reflect │
    │                                     │  → propose →    │
    │                                     │  re-eval →      │
    │                                     │  Pareto gate →  │
    │                                     │  human approval │
    │                                     └─────────────────┘
    │
    ├── POST /workflow/quizme/* ─────────► LangGraph :8090
    │   (QuizMe HITL)                             │
    │                                    ┌─────────┴─────────┐
    │                                    │  plan_objectives  │
    │                                    │  interrupt (HITL) │
    │                                    │  generate_quiz    │
    │                                    │  interrupt (HITL) │
    │                                    │  score + feedback │
    │                                    │  summarize        │
    │                                    └───────────────────┘
    │
    └── Hermes Agent Gateway :8642 ──────► Hermes v0.18.2
        (SOUL.md identity, ReAct loop)            │
        Production routing only                   ├── HindSight memory
                                                  ├── MCP Tools (FastMCP stdio)
                                                  └── Local model (vLLM / Ollama)


Quality Loop (offline):
    RAGAS 0.2.14 ──► Local model judge ──► faithfulness + precision scores
         │
         ▼
    GEPA pipeline ──► reflect → propose → approve ──► new SOUL.md
```

---

## 6. Detailed Component Design

### 6.1 Course Catalogue (Static Data, Zero API)

All course and track data lives in `tracks.js` — a local TypeScript/JavaScript file imported at build time. Zustand provides client-side state for filters and selection.

**Why static local data:** Browse UX is instant (zero network). Courses don't change daily. Adding a course = update the file + `npm run build` + deploy. Zero backend load for the most common user action. TanStack Query handles server-state for anything that does need network (purchases, auth status).

---

### 6.2 Auth — Firebase Auth + firebase-admin

Firebase Auth (Google Sign-In) issues JWT id tokens on the client. Every protected FastAPI endpoint calls `firebase_admin.auth.verify_id_token(token)` to validate the token server-side. No session cookies, no server-side session store. The admin SDK verifies JWTs against Google's public key set offline after initial fetch — no per-request Google call.

Access to purchased courses: FastAPI reads `user.purchases` array from Firestore. If `track_id` not present → 403.

---

### 6.3 Sovereign RAG Tutor — POST /api/hermes/chat

The platform's core AI feature. The full pipeline for one question:

```
1. ACCESS GATE
   firebase_admin.verify_id_token(bearer) → uid
   hasCourse(uid, course_id) → Firestore read → 403 if not purchased

2. AUDIT ROW (written BEFORE inference)
   INSERT audit_log {uid, question, model, provider, soul_hash, ts}
   Tamper-evident: if process crashes after write but before inference,
   the log shows an attempted call with no answer — honest accounting.

3. MEMORY LOAD
   GET student_profile (topics, turn_count, first_seen, last_seen)
   GET last 3 student_turns (Q, A pairs)
   Injected into system prompt as context window.

4. EMBED QUESTION
   OpenAI text-embedding-3-small → 1536-dim vector
   (The ONE remaining external call. Closing it = swap to nomic-embed-text locally.)

5. HNSW ANN SEARCH
   pgvector / in-memory store → top-4 chunks by cosine similarity
   Metadata per chunk: {course_id, lecture_name, start_ts, end_ts}

6. SCORE GATE — threshold 0.45
   max_score = max cosine similarity across top-4 chunks

   if max_score >= 0.45:
       → inject material into SOUL system prompt
       → call local model (Ollama / vLLM)
       → stream SSE: event:sources → event:token (N times) → event:done
       → record turn (student_turns + update student_profile)

   if max_score < 0.45 AND student has NOT consented to general answers:
       → return deterministic DECLINE in code — model is NEVER called
       → offer "answer from general knowledge?" option in UI

   if max_score < 0.45 AND student has consented:
       → call local model WITHOUT retrieved context
       → flag answer as "general knowledge, not grounded in course material"
       → record turn with is_grounded=false

7. SOURCE RENDERING
   Each retrieved chunk → Source {lecture_name, start_ts, snippet, score}
   React SourceCard: "[Lecture Name | 05:42]" → click → ?t=342 deep-link
   Student jumps to the exact video moment being referenced.
```

**Key design principle: grounding is enforced in code, not by model trust.**
The model never sees a question it is not allowed to answer with the available material. Below 0.45 the model is not called at all — no prompt engineering, no system prompt instruction telling the model "don't hallucinate." The guarantee is structural.

**Dev vs. Production routing:**
- Dev: FastAPI calls the local model directly (bypasses Hermes agent loop). On small models (qwen2.5:3b), routing through the full ReAct loop caused multi-turn context loss — the loop fragmented the conversation into single-turn tool calls. Direct model calls preserve context across turns reliably.
- Production: requests route through Hermes gateway (:8642). Full ReAct loop + HindSight memory + MCP tools. Larger model handles multi-turn context better.

**Why grounding threshold at 0.45, not 0.15:**
0.15 (the prior version's threshold) was set on an in-memory cosine store with no HNSW index. After migrating to HNSW with proper index calibration, the score distribution shifted. 0.45 was calibrated empirically: retrieve chunks for 50 known in-scope questions (all scored > 0.55), then retrieve for 50 known out-of-scope questions (all scored < 0.40). The 0.45 boundary cleanly separates the two populations with no false positives on held-out test questions.

---

### 6.4 GEPA — Self-Evolving SOUL

GEPA (Generative Evaluation and Prompt Adaptation) is the offline pipeline that improves the tutor's SOUL.md system prompt without human intervention in the critical path.

```
Trigger: POST /api/hermes/gepa/run (manual or scheduled, NEVER during chat)

1. EVAL CURRENT SOUL
   Run SOUL against fixed test set (QA pairs with expected answers)
   Each pair scored 1–5: faithfulness, helpfulness, groundedness
   Compute per-pair scores and overall average baseline

2. IDENTIFY WEAKEST CASES
   Sort by score, take 3 lowest-scoring QA pairs

3. LLM REFLECTION
   Prompt: "Here is the current SOUL, here are the 3 worst-performing cases.
   What specific behaviours in the SOUL caused these failures?
   Propose a revised SOUL that addresses them."
   → candidate SOUL text

4. RE-EVAL CANDIDATE
   Run same test set against candidate SOUL
   Compute per-pair scores and candidate average

5. PARETO GATE (in code, not model trust)
   Passes if AND ONLY IF:
     a. candidate avg > baseline avg  (overall improvement)
     b. NO individual case scores < 2  (no regression on any case)
   If gate fails → discard candidate, stop. Current SOUL unchanged.

6. HUMAN APPROVAL REQUIRED
   Email / webhook to operator: "GEPA proposes new SOUL. Review attached diff."
   POST /api/hermes/gepa/approve {token} → applies new SOUL
   POST /api/hermes/gepa/reject {token} → discards

7. APPLICATION
   Write new SOUL.md with SHA-256 fingerprint in filename
   Create SOUL.md.bak from current version
   Audit log: {previous_soul_hash, new_soul_hash, baseline_avg, candidate_avg, ts}
```

**Why human approval:**
GEPA optimizes a proxy metric on a small fixed test set. Without a human gate:
- The model could game the metric (Goodhart's law) — score high on test cases by memorising them without generalising.
- A good-looking test improvement could degrade behaviour on out-of-distribution questions.
- Self-modification without oversight is an unbounded loop with no natural stopping condition.

The human gate is not a bureaucratic checkpoint — it is the architectural defense against Goodhart's law. The operator reviews the actual SOUL diff before it ever affects a student.

---

### 6.5 RAGAS Evaluation

RAGAS 0.2.14 runs against every RAG answer (or a sampled subset in high-traffic periods).

**Two core metrics:**

- **Faithfulness:** Are all claims in the answer supported by the retrieved context chunks? Measured by decomposing the answer into atomic statements and asking the judge model: "Is this statement present in or inferable from the context?" Score = (supported statements) / (total statements). A faithfulness score of 0.9 means 90% of claims are grounded; 0.6 means the model is confabulating 40% of its answer.

- **LLM Context Precision:** Of the top-4 retrieved chunks, how many were actually useful for answering the question? A chunk retrieved but not referenced in the answer is wasted retrieval. This metric surfaces systematic retrieval errors — e.g., if chunk 1 and 2 are consistently irrelevant, the HNSW index or chunking strategy needs adjustment.

**Sovereign judge:** The RAGAS evaluation calls the local model (Ollama / vLLM) as the judge. Zero external API calls. Score computation is fully on-prem. These scores feed the GEPA pipeline as the primary quality signal GEPA optimises.

---

### 6.6 QuizMe — LangGraph StateGraph

QuizMe is a structured, human-gated quiz generation workflow. Not a chatbot — every step is defined in advance, every boundary is a human approval point.

```
StateGraph (LangGraph 1.2.9):

plan_objectives
    │  Input: topic, course_id, student profile
    │  Output: list of learning objectives [{id, objective, difficulty}]
    │
INTERRUPT: HITL — student approves or edits plan
    │  Graph execution suspends here.
    │  State serialised → MemorySaver checkpointer
    │  HTTP 202 with thread_id returned to client
    │  Student reviews plan in UI
    │  POST /workflow/quizme/resume {thread_id, approved_plan}
    │  Graph resumes from checkpoint with approved plan
    │
generate_quiz
    │  One question per approved objective
    │  Multiple-choice or open-ended based on difficulty
    │
INTERRUPT: HITL — student submits answer
    │  Suspend. Student sees question in UI, types answer.
    │  POST /workflow/quizme/answer {thread_id, answer}
    │  Resume.
    │
score + feedback (loops per objective)
    │  Grade answer against rubric
    │  Generate targeted feedback: what was right, what was wrong, why
    │  Update student profile: topic mastery level
    │
summarize
    │  Aggregate scores across all objectives
    │  Produce personalised study tips: "Focus more on X, you're solid on Y"
    │
END
```

**MemorySaver checkpointer:** Every node writes state to MemorySaver before returning. If the FastAPI process restarts between the student submitting an answer and receiving feedback, the graph resumes from the last checkpoint without data loss. In production, MemorySaver is backed by Firestore or Postgres for persistence across restarts.

**Zero external model calls:** QuizMe generation, scoring, and feedback all run on the local model. The workflow is fully sovereign.

---

### 6.7 Artifact Generation — Second LangGraph Graph

A separate StateGraph that turns a PDF or topic into structured study materials.

```
pdf_or_topic_input
    │
planner node
    │  Decomposes input into: {intro, concepts[], examples[], exercises[]}
    │  Assigns each section to a generator
    │
INTERRUPT: HITL — content creator approves plan
    │  Same suspend/resume pattern as QuizMe
    │
Send-fanout (LangGraph parallel branching)
    │  One generator node per section, all run concurrently
    │  ├── generate_notes (key takeaways)
    │  ├── generate_summary (executive overview)
    │  ├── generate_quiz (practice questions)
    │  └── generate_flashcards (Q&A pairs)
    │
assemble
    │  Collect all section outputs
    │  Validate completeness (any missing section → re-run that generator)
    │  Assemble into final artifact document
    │
END → stored in Firestore, linked to course
```

**Why planner before executor:** Bad plans are cheap to catch and correct (text, fast). Bad generated content is expensive (LLM call, slow). Human approval after planning eliminates the cost of regenerating from a wrong premise. The Send-fanout means the four generators run in parallel — wall-clock time is determined by the slowest section, not the sum of all sections.

---

### 6.8 MCP Tools — FastMCP stdio Server

Two tools registered as a FastMCP stdio server. The Hermes agent runtime invokes them during its ReAct loop when it decides a tool call is needed. The server definition is pure Python; the runtime handles discovery, invocation, and result parsing.

**Tool 1: search_course_content**
- Input: `{query: str, course_id: str, top_k: int = 4}`
- Executes the same HNSW retrieval as the RAG tutor
- Returns: `[{chunk_text, lecture_name, timestamp, score}]`
- Used by Hermes when it needs to look up course material mid-conversation

**Tool 2: generate_notes**
- Input: `{topic: str, context_chunks: list[str]}`
- Calls local model to produce structured notes from the provided context
- Returns: `{notes: str, key_concepts: list[str]}`
- Used during artifact generation when Hermes operates in agentic mode

**Design:** your code defines what the tools do and registers them. Hermes decides when to call them and in what order. This separation keeps tool logic testable in isolation from the agent loop.

---

### 6.9 Session Memory + GDPR

Two storage structures per student:

```
student_turns (append-only log):
    {uid, question, answer, is_grounded, soul_hash, score, ts}
    Retention: last N turns loaded as memory context for the tutor.
    Full history available for transparency (GET /api/hermes/memory/{id}).

student_profile (running summary, updated per turn):
    {uid, topics: list[str], turn_count: int, first_seen: ts, last_seen: ts}
    Topics list is accumulated across sessions.
    Injected into system prompt: "This student has asked about X, Y, Z before."
```

**GET /api/hermes/memory/{id}:** Returns full turn history and profile for a student. Required for transparency — students can see everything the system knows about them.

**DELETE /api/hermes/memory/{id}:** GDPR erasure. Deletes all `student_turns` rows and resets `student_profile` for the uid. Firestore: batch delete. In-memory dev: dict.pop(). Audit log row is retained (legal obligation) but answer field is nulled.

**Dev vs. prod storage:** Dev uses in-memory dicts (lost on restart — fine for development). Prod uses Firestore. Target architecture is Postgres + pgvector: one database for relational data, vector index, and session memory — simplifies ops vs. maintaining Firestore + a separate vector store.

---

### 6.10 Payments & Access

```
Buy Track button
    │
POST /api/payments/razorpay-order (FastAPI :8080)
    │  firebase_admin validates JWT → uid
    │  Razorpay secret key used server-side to create order
    │  Returns {order_id, key_id (public only)}
    │
Razorpay checkout.js modal (client)
    │  User completes payment
    │
payment_success callback
    │  POST /api/payments/verify (FastAPI)
    │  Verifies Razorpay signature (HMAC-SHA256)
    │  On success: Firestore arrayUnion(track_id) into user.purchases
    │
hasCourse(uid, track_id) on every protected endpoint
    │  Reads user.purchases from Firestore
    │  FastAPI returns 403 if track_id not in list
```

**Why server-side order creation:** Razorpay's secret key must never leave FastAPI. If the client created orders, anyone with the secret could create arbitrary ₹0 orders and claim purchase.

**Prices in paise (integers):** ₹999 stored as `99900`. Integer arithmetic. No float precision errors. Common bug: `float(999) * 0.18 = 179.82000000000002`. Paise eliminates this class of error in financial calculations.

**Video protection — XOR + Base64 encoding:**

Paid lecture video URLs (Firebase Storage) are not stored in plain text anywhere in the frontend bundle. Instead they are pre-encoded offline using XOR against a cycling secret key, then Base64-encoded, and committed to `tracks.ts` as opaque ciphertext blobs. At playback time, `videoDecoder.ts` decodes them client-side:

```typescript
// videoDecoder.ts
const SECRET_KEY = "thisisasecretofus";

export function decodeVideoUrl(encoded: string): string {
  const decoded = atob(encoded);                         // Step 1: Base64-decode
  let result = "";
  for (let i = 0; i < decoded.length; i++) {
    result += String.fromCharCode(
      decoded.charCodeAt(i) ^ SECRET_KEY.charCodeAt(i % SECRET_KEY.length)
    );
  }                                                      // Step 2: XOR with cycling key
  return result;
}
```

```typescript
// CoursePlayerPage.tsx
const getVideoSrc = (src: string): string => {
  if (src.startsWith("http")) return src;   // free/preview content: plain URL
  return decodeVideoUrl(src);               // paid lectures: decode before playback
};
```

**Why XOR specifically:** Symmetric (same operation encodes and decodes), zero dependencies, instantaneous. No separate encode/decode function needed — XOR is its own inverse. Runs in the browser with no crypto libraries.

**Two-layer access model:**
1. **Primary gate (hard):** Firebase Auth + Firestore purchase check. Non-buyers never reach the player screen. Backend verifies JWT on every protected endpoint.
2. **Secondary layer (obfuscation):** XOR encoding prevents raw Firebase Storage URLs from appearing in plain text in the JavaScript bundle that ships to the browser.

**Honest limitation to know for interviews:** The secret key ships inside the compiled JS bundle. A determined user who opens `dist/assets/index-*.js` can find it and decode every URL in a few minutes. This is obfuscation, not cryptographic protection. The real security is the purchase gate — the XOR layer raises the bar above trivially copy-pasting a URL from page source, which is the threat it was designed for.

**What a more robust solution looks like:** Server-issued short-lived signed Firebase Storage URLs (TTL 1–15 min), generated only after verifying the purchase on the backend. The client gets a URL that expires — even if shared, it stops working within minutes. The trade-off is a backend round-trip before every video load. The XOR approach trades security ceiling for zero-infrastructure simplicity.

---

## 7. Two-Engine Design

The platform runs two AI orchestration engines simultaneously. Choosing which to use for a feature is not a preference — it follows a deterministic decision rule.

### The Decision Rule

**Use LangGraph (workflow engine) when:**
- You can write every step in advance
- Each step is bounded in scope
- Human approval is required at specific, known points
- The workflow must be resumable across process restarts
- The output is a specific deliverable (quiz, set of flashcards, assembled document)

**Use Hermes (agent runtime) when:**
- The AI must decide its own next move based on what it finds
- The number of steps is unknown in advance
- The task is open-ended: "answer this question about this course"
- ReAct-style tool use is needed (search → observe → reason → search again)

### Feature Assignment

| Feature | Engine | Why |
|---|---|---|
| Sovereign RAG Tutor | Hermes (prod) / Direct (dev) | Open-ended Q&A, AI decides when to use search_course_content |
| GEPA | Neither — script | Eval loop, deterministic steps, runs offline |
| RAGAS | Neither — library call | Scoring, not orchestration |
| QuizMe | LangGraph | Known steps: plan → approve → generate → approve → score × N → summarize |
| Artifact Generation | LangGraph | Known steps: plan → approve → parallel generate → assemble |
| MCP Tools | Hermes (caller) | Tools are registered; Hermes decides when to invoke them |
| Session Memory | FastAPI direct | Simple CRUD, no orchestration needed |

### Why Not Hermes for Everything?

QuizMe has a defined structure: N objectives → N questions → N score+feedback cycles → one summary. Every step is known before execution starts. Giving this to Hermes (ReAct loop) would mean the agent decides whether to generate a quiz or ask a clarifying question or look something up — introducing non-determinism into a flow where the student expects a quiz. LangGraph guarantees the flow executes the defined steps in order, suspends at the right interrupt points, and resumes from the exact checkpoint. You cannot accidentally skip the human approval step.

Conversely, the RAG tutor cannot be a LangGraph workflow. The answer to "explain virtual memory" might require one retrieval and one generation, or it might require the agent to search for "virtual memory address translation," observe that the result doesn't directly cover the student's question, then search for "page table walkthrough," then synthesise across both results. The number of tool calls is not known in advance. Hermes handles this with its ReAct loop.

---

## 8. Governance & Sovereignty

### EU-AI-Act Audit Trail

Every inference request — RAG tutor, QuizMe, artifact generation — writes an audit row BEFORE the model is called:

```python
{
  "uid": student_uid,
  "question": question_text,
  "model": "qwen2.5:3b",
  "provider": "ollama_local",
  "soul_hash": "sha256:abc123...",  # hash of SOUL.md in use at inference time
  "ts": datetime.utcnow().isoformat(),
  "answer": None   # filled in after inference completes
}
```

**Why write before inference:** If the process crashes between the audit write and the model call, the row shows an attempted inference with `answer: null`. This is honest — the system tried and failed. Writing after inference risks losing the record entirely if the process crashes post-inference but pre-write. The audit is tamper-evident: you can prove every SOUL version that was in use at every moment, because the soul_hash is written before the SOUL has any opportunity to be changed.

### Sovereign Inference

Student conversations are generated on hardware you control. In dev: Ollama on localhost. In prod: vLLM on a GPU server.

**What this means operationally:**
- Student questions, answers, course transcripts never leave your infrastructure during inference.
- Latency is predictable — no third-party API rate limits or cold starts.
- Cost is compute, not per-token API fees. At scale this is a meaningful advantage.

**The ONE remaining external call:** Retrieval embeddings use OpenAI `text-embedding-3-small`. This sends the student's question to OpenAI's servers. Closing this gap = swap `embed_question()` to call a local embedder (nomic-embed-text on Ollama) — one function, one file. The swap is intentionally isolated. Until then, the audit log records `provider: "openai"` for embedding calls so the external call is visible in the audit.

### GDPR

- Transparency: GET /api/hermes/memory/{id} returns all stored data about a student.
- Erasure: DELETE /api/hermes/memory/{id} removes all turns and profile. Audit rows are retained per legal obligation but the answer field is nulled.
- Consent gate: below score 0.45, the system offers a "general knowledge" option and stores the student's consent flag before answering without retrieved context.

---

## 9. Key Engineering Decisions

| Decision | Choice | Alternative | Why |
|---|---|---|---|
| Course data | Static JS (tracks.js) | API endpoint | Instant browse, zero backend load |
| Auth | Firebase Auth + firebase-admin JWT | Custom session | Google handles token rotation and security |
| Two orchestration engines | Hermes + LangGraph | Single framework | Agents for open-ended, workflows for known steps — mismatching them causes either rigid agents or non-deterministic workflows |
| RAG generation | Local Ollama / vLLM | GPT-4 API | Sovereign inference, cost control, audit clarity |
| Grounding gate | Score >= 0.45 in code | Prompt instruction | Structural guarantee — model cannot hallucinate on low-confidence retrieval because it is never called |
| GEPA human approval | Required | Auto-deploy | Defense against Goodhart's law — proxy metric on a small eval set should not auto-update production behaviour |
| RAGAS judge | Local model | External judge | Zero external calls for evaluation — fully sovereign quality loop |
| HITL | LangGraph interrupt() | Custom polling | Built-in checkpointing, suspend/resume across restarts |
| Streaming | SSE | REST polling | Progressive answer delivery; perceived latency < actual latency |
| Prices | Integer paise | Float rupees | Float precision errors in financial arithmetic |
| Audit timing | Write before inference | Write after | Tamper-evident; survives process crash between write and inference |
| Dev routing | Direct model call (bypass Hermes) | Full Hermes loop | Small model (qwen2.5:3b) loses multi-turn context in ReAct loop; direct call preserves context |

---

## 10. Honest Gaps

**Dev reliable path bypasses Hermes agent loop.** On qwen2.5:3b, routing through the Hermes ReAct loop causes multi-turn context loss. The loop treats each tool call as a fresh invocation, fragmenting the conversation. Direct model calls preserve the full context window. This means HindSight (Hermes's memory layer) does not activate in dev — only in production where the larger model handles ReAct correctly.

**HindSight configured but idle in dev.** HindSight is Hermes's persistent memory feature. It is configured in the Hermes agent definition but only activates on traffic that routes through the Hermes agent loop. Dev traffic routes directly, so HindSight is dormant until the production routing is active.

**Dev storage is in-memory.** All student turns, profiles, and vector chunks are lost on process restart in dev. This is an intentional trade-off (zero setup, fast iteration) but means dev cannot be used for longitudinal testing of memory features without running Firestore locally (Firebase Emulator).

**Retrieval embeddings still call OpenAI.** The one external call in an otherwise sovereign system. The fix is one function swap. It has not been made because text-embedding-3-small has better embedding quality at this scale than the local alternatives tested so far (nomic-embed-text scored ~3% lower on in-domain retrieval precision on a 500-question eval set). The trade-off is documented and will be revisited when a better local embedder is available.

---

## 11. FAQ

**Q: Walk me through the full RAG tutor pipeline from question to answer.**

A: The request hits FastAPI at :8080. Step 1: firebase-admin validates the JWT, checks Firestore that the student has purchased the relevant course. Step 2: an audit row is inserted with the question, model, SOUL hash, and timestamp — before anything else. This is the EU-AI-Act compliance row. Step 3: load the student's profile (topics, turn count) and last 3 conversation turns from the memory store; these are injected into the system prompt. Step 4: embed the student's question with OpenAI text-embedding-3-small (the one external call). Step 5: run HNSW ANN search — top-4 chunks by cosine similarity, each with metadata: lecture name, start and end timestamp. Step 6: check the score gate. If max cosine similarity across the 4 chunks is >= 0.45, the retrieved material is injected into the SOUL system prompt and the local model (Ollama in dev, vLLM in prod) is called. The response streams back as SSE: first an event:sources frame with the retrieved chunks, then event:token frames for each generated token, then event:done. Step 7: the React client renders source citations as clickable SourceCards — "[Lecture Name | 05:42]" — that deep-link to `?t=342` in the video player. Step 8: the turn is recorded in student_turns and the profile is updated. If the score gate fails (score < 0.45) and the student has not given consent for general answers, a deterministic DECLINE is returned in code — the model is never called.

---

**Q: You have a score gate at 0.45. How did you arrive at that number? What happens below it?**

A: The threshold is empirical. I built a calibration set: 50 questions known to be in scope (asked about concepts directly covered in course transcripts) and 50 questions known to be out of scope (asked about unrelated topics or content not in the platform). I ran retrieval on all 100 and plotted the score distribution. The in-scope questions clustered above 0.55, the out-of-scope questions clustered below 0.40. I set the threshold at 0.45 — the midpoint of the gap — which gave zero false positives (no out-of-scope question passed the gate) and zero false negatives (no in-scope question was blocked) on the calibration set. I then validated on a held-out set of 30 questions.

Below 0.45: the model is never called. This is structural, not prompt-based. I don't tell the model "if you don't know, say so." I don't call the model at all. A deterministic DECLINE is returned in Python: `return {"answer": "I don't have enough course material on this topic to answer accurately.", "declined": True, "reason": "score_below_threshold"}`. The student is offered a separate consent-gated "answer from general knowledge" option if they want a non-grounded answer.

---

**Q: What is GEPA? How does the self-evolution loop work?**

A: GEPA is Generative Evaluation and Prompt Adaptation — the offline pipeline that improves the tutor's SOUL.md system prompt. It runs periodically or on-demand, never during a live chat. The loop: first, evaluate the current SOUL against a fixed test set of QA pairs, scoring each 1–5 on faithfulness, helpfulness, and groundedness. Identify the 3 weakest-scoring cases. Prompt the local model: "Here is the current SOUL and here are the 3 worst cases. What behaviours caused these failures? Propose a revised SOUL that addresses them." Collect the candidate SOUL. Re-evaluate the candidate on the same test set. Apply a Pareto gate in code: the candidate must score higher on average AND score >= 2 on every individual case (no regression allowed anywhere). If the gate passes, the candidate goes to human review — the operator receives a diff and approves via POST /api/hermes/gepa/approve. The new SOUL is applied with a SHA fingerprint recorded in the audit log. The previous SOUL is kept as SOUL.md.bak.

The human approval gate is the critical safeguard. GEPA optimizes a proxy metric on a small eval set. Without the gate, the model could learn to game that specific set of questions (Goodhart's law) without improving on novel inputs. The human reviews the actual SOUL diff and decides whether the proposed change represents genuine improvement or metric gaming.

---

**Q: Why two orchestration engines? Why not just Hermes for everything?**

A: The engines solve different problems. Hermes is a ReAct agent — at each step, it decides what to do next based on what it observes. This is exactly right for the RAG tutor: "answer this question" might require one retrieval, or it might require three retrievals with reasoning between them. The number of steps is not known in advance. Hermes handles this by design.

QuizMe is different. QuizMe has a defined structure: plan objectives, get human approval, generate one question per objective, collect each answer, score and provide feedback, summarise. Every step is known before execution starts. If I give this to Hermes, the agent might skip the human approval step because it "decides" the plan looks fine. Or it might ask a clarifying question instead of generating a quiz. The ReAct loop introduces non-determinism into a flow where the student expects deterministic behaviour. LangGraph guarantees the flow executes the defined steps, in order, with interrupt points at specific nodes — not wherever the agent decides to pause.

The decision rule is: if you can enumerate every step in advance and each step is bounded → LangGraph. If the AI must decide its own path from what it finds → Hermes.

---

**Q: How do you implement HITL in LangGraph? What happens at an interrupt point?**

A: LangGraph's `interrupt()` is a first-class primitive. When the graph executor hits an interrupt node, it: serialises the entire graph state (all node outputs so far, the remaining execution path), writes it to MemorySaver (keyed by `thread_id`), and returns HTTP 202 with the `thread_id` and the data the human needs to review. The HTTP request completes. The graph is suspended.

On the client, the student sees the plan (or quiz question, depending on which interrupt) and interacts with it. When they submit their response, the client POSTs to `/workflow/quizme/resume` with `{thread_id, approved_plan}` (or `/workflow/quizme/answer` with their quiz answer). FastAPI retrieves the graph state from MemorySaver by thread_id and resumes execution from the interrupt point with the human's input injected.

If the FastAPI process restarts between the suspend and resume — MemorySaver persists state so the resume still works. In production, MemorySaver is backed by Firestore, not in-memory, so state survives server restarts and deployments. A student can start a quiz on Monday, approve the plan, answer one question, put their laptop away, and resume Thursday. The state is waiting for them.

---

**Q: Your sovereign inference uses Ollama locally. How does that generalize to production?**

A: Both Ollama (dev) and vLLM (prod) expose an OpenAI-compatible API. The generation client in FastAPI points at `OLLAMA_BASE_URL` (dev, :11434) or `VLLM_BASE_URL` (prod) via an environment variable. The model name changes (`qwen2.5:3b` in dev, a larger model in prod). No application code changes between environments — the API contract is identical. This is intentional. The abstraction means I can also swap to any OpenAI-compatible provider (Groq, Together, Anyscale) by changing the base URL and model name, without touching the generation code.

In production, vLLM runs on a GPU server with continuous batching — it handles multiple concurrent inference requests efficiently by padding requests into batches, significantly improving throughput over serial inference. The Hermes gateway routes requests to vLLM and handles retries, timeout, and fallback.

---

**Q: How do you prevent hallucination in the RAG tutor?**

A: Three layers, each independent of the others:

Layer 1 — Score gate in code. Below 0.45 cosine similarity, the model is never called. This is structural: the model cannot hallucinate on a question it never receives.

Layer 2 — Retrieved context injection. When the model is called, its system prompt contains the exact retrieved chunks formatted as: `[Course Title | Lecture Name | 05:42-08:10]\n<transcript text>`. The model is instructed to answer only from this material. The SOUL.md includes explicit grounding instructions.

Layer 3 — RAGAS faithfulness scoring after the fact. RAGAS decomposes the generated answer into atomic statements and checks each against the retrieved context using the local model as judge. Faithfulness < 0.7 on a question type signals a systematic prompt or retrieval problem that needs fixing — either the SOUL instructions aren't working for that question pattern, or the retrieved chunks are too short to provide sufficient context.

The key distinction from prompt-based approaches: layers 1 and 3 are not asking the model to behave correctly. Layer 1 is a code gate — the model has no opportunity to hallucinate because it is not invoked. Layer 3 is post-hoc measurement — it doesn't prevent hallucination but it detects it and feeds the GEPA improvement loop.

---

**Q: What is your RAGAS evaluation setup and what does faithfulness measure?**

A: RAGAS 0.2.14 runs as a post-inference quality measurement. For each RAG answer, I pass: the original question, the generated answer, and the list of retrieved context chunks. RAGAS uses the local model (Ollama / vLLM) as the judge — zero external API calls.

Faithfulness measures whether every claim in the generated answer is supported by the retrieved context. The measurement procedure: decompose the answer into N atomic statements ("the call stack grows downward," "each frame contains the return address," etc.). For each statement, ask the judge: "Is this statement directly supported by the provided context chunks?" Score = (statements supported) / N. A faithfulness score of 1.0 means every claim in the answer was in the retrieved material. A score of 0.6 means 40% of the answer's claims were invented by the model without retrieval support — those are hallucinations.

Context precision measures whether the top-4 retrieved chunks were actually useful for answering the question. A chunk retrieved but not used in the answer is wasted retrieval — either the question was more specific than the chunk covers, or the HNSW index is returning low-quality neighbours. Persistent low context precision is the signal to revisit chunking strategy (window size, overlap) or index parameters.

Both scores feed GEPA: GEPA optimises the SOUL to improve the quality signal RAGAS measures.

---

## 12. Interview Bridges

These are conceptual connections from this platform's design to industry-standard systems and research. Useful for demonstrating pattern recognition across domains.

**GEPA → Agent evaluation in production**
GEPA's structure (eval → identify weak cases → reflect → propose → re-eval → human gate) mirrors Anthropic's Constitutional AI self-critique loop: a model critiques its own outputs against a constitution, revises them, and the revision is evaluated. The architectural difference is the human approval gate. Constitutional AI at Anthropic scale can automate the gate because they have large-scale evaluation infrastructure. GEPA keeps the gate human because the eval set is small (overfitting risk is real) and self-modification of production behaviour without oversight is high-stakes on a paying student platform.

**Grounding threshold → Search result confidence gate**
Google and Bing both operate confidence gates at query-time. When a query matches no confident knowledge graph entry, the system falls back to organic results without an answer box — it does not fabricate an answer box. The structure is identical to the 0.45 gate: retrieve, score, threshold, gate. Below the threshold, the surface does not generate an answer — it shows raw results (analogous to offering "general knowledge" option). The threshold value is empirical in both cases, calibrated on known-good and known-bad queries.

**Two-engine design → Copilot Workspace and GitHub Actions**
GitHub Copilot Workspace uses an agent for the open-ended parts of coding tasks (interpreting ambiguous requirements, deciding which files to edit, writing the actual code changes). GitHub Actions is a workflow engine for the bounded, sequential parts (run tests, lint, deploy). Neither is replaced by the other — they complement. The same principle applies here: Hermes for open-ended tutoring, LangGraph for structured quiz and artifact generation. Using a workflow engine for open-ended Q&A produces rigid, fragile pipelines. Using an agent for structured workflows produces non-deterministic, hard-to-audit flows.

**Sovereign inference → Apple on-device ML**
Apple's approach to Siri, autocomplete, and photo classification: run on device, private by architecture. The privacy guarantee is structural — the data never leaves the device, so there is nothing to breach. The01.dev's sovereign inference is the same principle at a different scale: student questions and course transcripts never leave the operator's infrastructure during generation. The guarantee is not contractual (a cloud provider promising not to read your data) — it is architectural. The audit log records every external call explicitly so any deviation from sovereignty is visible.

**HITL interrupt → Human review gates in content moderation**
Large-scale content moderation pipelines (Facebook, YouTube) use classifier-based routing: high-confidence benign → auto-approve, high-confidence violation → auto-remove, low-confidence → human review queue. LangGraph's interrupt() is the same gate: above-threshold confidence in the AI's output → proceed automatically; uncertain or high-stakes step → suspend and route to human. The student's plan approval in QuizMe is exactly the low-confidence/high-stakes human review gate — the AI has generated a plan, but a wrong plan wastes the student's time, so the human reviews before execution proceeds.

---

## 13. What-If Scenarios

**"10K courses at full transcripts — what's retrieval latency and mitigation?"**

At 10K courses, assuming 20 lectures per course and 15 chunks per lecture: 3 million chunks. HNSW ANN search is O(log N) in practice — going from 10K chunks (current) to 3M chunks increases query time by roughly 2–3x (from ~15ms to ~30–45ms), not 300x. HNSW's navigable small world graph means the search path through the index stays short even at large N.

Mitigation if latency becomes noticeable: first, partition the index by course — a student asking about "JS Engine Internals" only needs retrieval against that course's chunks (15 × 20 = 300 chunks, not 3M). This is feasible because every request includes `course_id`. Second, cache the most common question embeddings — popular questions (e.g., "what is a closure") resolve from the cache without an embed call. Third, Faiss with GPU-backed HNSW reduces search time further. The pgvector → Faiss swap is one abstraction layer change in `vector_store.py`.

---

**"10 new courses added daily — how does re-indexing work without downtime?"**

The indexing pipeline is incremental: new course content is chunked, embedded (batch call), and inserted into the vector store. It does not require a full rebuild. With pgvector, this is standard INSERT with an index update — HNSW supports incremental insertion without rebuilding the entire graph. The index degrades slightly with heavy insertions (HNSW's ef_construction parameter determines the quality-speed trade-off at insert time), but query quality is maintained. A weekly REINDEX CONCURRENTLY (Postgres) rebuilds the index offline without downtime — Postgres continues serving queries from the old index until the rebuild completes.

The embedding batch call (OpenAI text-embedding-3-small) is the constraint: OpenAI rate limits at 1M tokens/minute. A 60-minute lecture transcript is roughly 9,000 tokens. 10 courses × 10 lectures = 100 transcripts × 9,000 tokens = 900,000 tokens — within the limit but close. Mitigation: queue course uploads with a rate limiter, or move to local embedder (nomic-embed-text) which has no external rate limit.

---

**"GEPA is optimizing the wrong metric — Goodhart's law kicks in. What's your defense?"**

Goodhart's law: when a measure becomes a target, it ceases to be a good measure. GEPA's specific version of this: the candidate SOUL learns to score well on the fixed 50-question eval set without improving on questions outside that set. Defenses:

First line — the Pareto gate in code. The gate requires the candidate to improve on average AND not regress on any individual case. This makes it harder (not impossible) to game: the model must broadly improve, not just ace a subset of cases while tanking others.

Second line — human approval. The operator reviews the SOUL diff before deployment. Goodhart's law gaming tends to produce weird SOUL instructions that look suspicious in a diff: overly specific phrasings, unexplained additions that look like they're designed for particular cases. A careful reviewer catches this.

Third line — hold-out set rotation. The 50-question eval set is rotated periodically with new questions the SOUL has never been evaluated on. If GEPA-improved SOUL scores drop significantly on the new hold-out questions, that's the Goodhart signal — improvement on the old set did not generalise. The human approval step then rejects the candidate.

Fourth line — RAGAS on live traffic (not just eval set). Faithfulness scores on real student questions are the ground truth signal. If a GEPA-applied SOUL improves eval scores but live faithfulness drops, that's the definitive Goodhart evidence and triggers a rollback to SOUL.md.bak.

---

## 14. What I'd Do Differently

**Close the sovereign gap on retrieval embeddings first.** The ONE external call (OpenAI for embeddings) is the most visible architectural inconsistency in a sovereign system. nomic-embed-text on Ollama is one function swap. I'd close this before building any other sovereignty feature — it's cheap and it makes the system's privacy guarantee airtight. The reason it hasn't been done yet (3% lower retrieval precision on the in-domain eval set) deserves a more rigorous investigation: maybe nomic-embed-text with fine-tuning on course transcripts would close that gap.

**Postgres + pgvector as the single store from day one.** The current architecture has Firestore for relational data and an in-memory vector store for embeddings. The target is Postgres + pgvector — one database for relational tables, vector index, session memory, and audit log. Maintaining two storage systems (Firestore + a vector store) doubles operational complexity and doubles the surfaces for data consistency bugs. Starting with Postgres from the beginning would have simplified every data access layer.

**Eval-driven development for the RAG tutor.** The RAGAS evaluation was added after the RAG tutor was built. It should have been the first thing built — a golden QA set and a RAGAS harness — so every change to the chunking strategy, threshold, or system prompt is evaluated against the same set. Retrofitting evaluation is harder than building with it from the start.

**Finer-grained HITL consent.** The current consent model for below-threshold answers is binary: yes or no to "general knowledge" answers. A better design would be session-level consent with explicit opt-out per turn. Students in exploratory mode might always want general answers; students in exam prep mode never want them. A preference saved to the student profile would eliminate the per-question consent prompt.

**LangGraph persistence from day one.** MemorySaver backed by in-memory dicts in dev means any process restart during a quiz loses the thread. Even in dev, Postgres-backed MemorySaver would prevent this. The ops cost of a local Postgres instance is trivial compared to the debugging cost of non-reproducible state-loss bugs in LangGraph workflows.

---

## Technology Comparisons

### RAG vs Fine-tuning vs Long Context

The most common AI interview question: "Why RAG instead of fine-tuning the model on your course content?"

| Dimension | RAG | Fine-tuning | Long Context |
|---|---|---|---|
| Knowledge updates | Instant (update vector store, no retraining) | Requires full retraining cycle | Instant (swap documents) |
| Source attribution | Native (retrieved chunks become citations) | None — knowledge is baked in | Native (documents in context) |
| Hallucination control | Grounding threshold enforces retrieval relevance | Model can hallucinate fine-tuned facts confidently | Long context increases distraction, reduces precision |
| Cost | Retrieval + generation per query | One-time training cost, then cheaper inference | High per-query token cost |
| What it changes | What the model knows at query time | How the model behaves / its style | What the model sees per query |
| Best for | Dynamic knowledge, multi-document, citations needed | Stable behavior, persona, domain vocabulary | Small stable corpus, simple queries |

**When fine-tuning wins:** The model needs to behave differently (not know different facts) — adopt a specific communication style, format output in a proprietary schema, or learn domain jargon that doesn't appear in pretraining. Also wins at high inference volume where retrieval latency is a bottleneck.

**When long context wins:** The entire knowledge base fits comfortably in a context window, queries are infrequent, and retrieval engineering complexity isn't worth it. GPT-4o's 128K context makes this viable for small corpora.

**For the01.dev:** RAG is the right choice. Course transcripts are updated as new courses ship — zero retraining budget. Students need to see which transcript excerpt supports the answer (Socratic teaching requires attribution). The grounding threshold (0.45) means the model is never called if retrieval returns irrelevant results — fine-tuning cannot provide this deterministic safety valve. The corpus grows unboundedly as courses are added, ruling out long context.

**Interview move:** "RAG for knowledge, fine-tuning for behavior." If a student asks why the tutor sounds like GPT-4o and not a human teacher, that would be a fine-tuning problem. The course content question is a knowledge problem — RAG is the right tool.

---

### pgvector vs Pinecone vs Chroma

For the RAG retrieval layer, three main vector store options were considered:

| Dimension | pgvector (Postgres) | Pinecone | Chroma |
|---|---|---|---|
| Where embeddings live | Same DB as relational data | External managed service | Local process or server |
| Transactions | ACID — vector + relational in one transaction | No — separate system | No |
| Scale ceiling | Single Postgres node (tens of millions of vectors) | Billions of vectors, managed | Thousands to low millions |
| Ops burden | Zero if already running Postgres | Zero (managed) | Zero (local) |
| Data sovereignty | Your infrastructure | Pinecone's cloud | Your infrastructure |
| Latency | Same DB call | External HTTP request | Local process call |
| HNSW index | Yes (approximate nearest neighbor) | Yes | Yes |

**When Pinecone wins:** Corpus is billions of vectors, you need managed scaling without infrastructure ownership, and data sovereignty is not a constraint.

**When Chroma wins:** Prototyping, local development, or very small corpora where setting up Postgres is overhead.

**For the01.dev:** pgvector because the course transcript corpus is small (thousands of chunks from ~8 courses), embeddings must be transactionally consistent with purchase records (only paid students retrieve content), and the sovereign architecture requires data never leaving the system. One fewer managed service means one fewer attack surface and one fewer monthly bill.

**Interview move:** "pgvector is not the right choice at 100M vectors. But for a corpus that grows linearly with course count — and where the purchase check, user record, and retrieved chunks all need ACID consistency — keeping vectors in the same Postgres instance is simpler, cheaper, and safer than a dedicated vector service."

---

### SSE vs WebSocket for LLM Token Streaming

The01.dev uses SSE to stream tokens from the Hermes tutor to the browser. The two real-time delivery options:

| Dimension | SSE (Server-Sent Events) | WebSocket |
|---|---|---|
| Direction | Server → Client only | Bidirectional |
| Protocol | HTTP/1.1 (standard) | Separate upgrade handshake |
| Proxy / LB compatibility | Excellent — standard HTTP | Requires explicit WebSocket support |
| Auto-reconnect | Built into browser EventSource | Application-level only |
| When client needs to send mid-stream | Not possible | Native |
| Complexity | Low — standard fetch or EventSource | Higher — custom protocol, heartbeats |

**When WebSocket wins:** The client needs to send messages while the server is still streaming — e.g., "stop generating", "edit my question mid-response". If you need bidirectional real-time (collaborative editing, live cursors), WebSocket is the only choice.

**For the01.dev:** SSE because LLM streaming is one-directional: the student sends one request, the server streams back. The client never needs to interrupt or send during streaming. SSE's automatic reconnect handles network hiccups gracefully. SSE works through standard HTTP infrastructure — no load balancer configuration changes needed.

**The event sequence:** `sources` event (retrieved chunks) → multiple `token` events (answer characters) → `done` event (generation complete). This three-phase structure is natural SSE and would not benefit from bidirectionality.

**Interview move:** "WebSocket would add complexity for no benefit here. The only reason to use WebSocket in an LLM streaming context is if the user can interrupt mid-generation — a 'stop' button that cancels the server-side inference request. We could add that with SSE too: a separate POST to `/cancel` doesn't require a bidirectional channel."

---

## Technical Dictionary

*Plain-English definitions of every term, algorithm, and tool used in this document. If something above confused you, start here.*

---

### Retrieval & Vector Search

### RAG (Retrieval-Augmented Generation)
A technique where the system looks up relevant content from a knowledge base first, then hands that content to the language model as context before generating an answer. It prevents hallucination by grounding the model in actual material rather than relying on what it memorised during training.

**Example:** When a student asks "how does the JS event loop work?", the RAG tutor retrieves the four most relevant transcript chunks from the JS Engine Internals course before the model writes a single word of its answer.

---

### Embedding
The process of turning a piece of text into a list of numbers — a vector — that captures its meaning. Texts with similar meanings produce vectors that are numerically close together in high-dimensional space, which makes it possible to search for similar meaning rather than exact keyword matches.

**Example:** The student's question "what is a closure?" and the transcript line "closures capture the surrounding lexical scope" will have a high cosine similarity because their embeddings land near each other in the 1536-dimensional space produced by text-embedding-3-small.

---

### Vector Store
A database purpose-built for storing embeddings and searching them by similarity rather than by exact value. Instead of `WHERE name = 'foo'`, you query by "find me the 4 vectors closest to this query vector."

**Example:** The01.dev stores every course transcript chunk as an embedding in a vector store (in-memory in dev, pgvector in prod) and queries it with HNSW ANN search on every RAG tutor request.

---

### HNSW (Hierarchical Navigable Small World)
An algorithm that organises vectors into a multi-layer graph where similar vectors are connected by edges. When searching, the algorithm navigates the graph layer by layer — starting broad and narrowing down — rather than comparing the query against every stored vector. The result is fast approximate search that stays quick even as the vector count grows into the millions.

**Example:** When a student asks a question, HNSW searches 10,000+ course transcript chunks in roughly 15ms by navigating the graph rather than scanning every chunk linearly.

---

### ANN (Approximate Nearest Neighbor)
A family of search algorithms that find the closest vectors to a query vector quickly by accepting a small accuracy trade-off — they might miss the single absolute closest vector, but they reliably return vectors from the top cluster. For RAG, this trade-off is fine: you want semantically relevant chunks, not a mathematically exact ranking.

**Example:** The01.dev's HNSW ANN search returns the top-4 chunks by approximate cosine similarity; missing the 5th most similar chunk by a hair has no practical effect on the quality of the tutor's answer.

---

### Cosine Similarity
A measure of the angle between two vectors in high-dimensional space. A score of 1 means the vectors point in exactly the same direction (identical meaning), 0 means they are perpendicular (unrelated), and -1 means they point in opposite directions. For text embeddings, scores typically range from 0 to 1 in practice.

**Example:** A student question about "garbage collection in V8" and a transcript chunk explaining "how V8's Orinoco GC compacts the heap" will have a cosine similarity above 0.55, well above the01.dev's 0.45 gate.

---

### Grounding Threshold (0.45)
The minimum cosine similarity score that the top retrieved chunk must achieve before the language model is allowed to run. Below this score, the system returns a deterministic decline in code — the model is never called, so it has no opportunity to hallucinate. The specific value was calibrated empirically on a set of known in-scope and out-of-scope questions.

**Example:** If a student enrolled in the JS Engine Internals course asks "what are the visa requirements for Canada?", the max retrieval score will be well below 0.45, and FastAPI returns a DECLINE response without touching the local model.

---

### Agent Architecture

### Hermes
The custom agent runtime used in the01.dev's production path. An agent runtime manages the loop of: reasoning about what to do, calling tools, reading tool results, reasoning again — until the agent has a complete answer. Hermes adds a SOUL.md identity layer, HindSight memory, and MCP tool integration on top of this loop.

**Example:** In production, a student's tutoring question routes through the Hermes gateway at port 8642, where Hermes loads the current SOUL.md, checks HindSight memory for prior conversation turns, and decides whether to invoke the `search_course_content` MCP tool.

---

### ReAct Loop
Short for Reasoning + Acting. The agent reasons about its situation in text, decides on an action (usually a tool call), observes the tool's result, then reasons again — repeating until it can produce a final answer. The number of iterations is not fixed in advance.

**Example:** The Hermes tutor might reason "the student asked about closures, I should search the course content," call `search_course_content`, observe three relevant chunks, reason "chunk two directly answers this," then generate the answer — three reasoning steps, one tool call.

---

### SOUL.md
A markdown file that defines the agent's identity, behavioural rules, and constraints. It is loaded as the system prompt on every inference call, so it shapes every response the agent produces. GEPA's job is to improve this file over time without breaking it.

**Example:** SOUL.md for the01.dev's tutor instructs the agent to answer only from retrieved course material, cite specific lecture timestamps, and acknowledge uncertainty rather than speculate — these rules are in every system prompt, not enforced by prompt engineering per-request.

---

### MCP (Model Context Protocol)
A protocol for exposing tools to an AI model in a typed, schema-validated way. The tool author defines what inputs the tool accepts and what it returns; the runtime validates the model's arguments against that schema before executing the tool. This prevents the model from calling tools with malformed or dangerous inputs.

**Example:** The `search_course_content` MCP tool accepts `{query: str, course_id: str, top_k: int}` — if the Hermes agent tries to pass an unexpected field, FastMCP rejects the call before it runs.

---

### HindSight
Hermes's memory module. It stores the history of a student's conversation turns and builds a running profile (topics covered, turn count, first and last seen). On each new request, HindSight loads the last few turns and the profile into the system prompt so the tutor has context without the student repeating themselves.

**Example:** If a student asked about closures two turns ago and now asks "how does that relate to the module pattern?", HindSight ensures the tutor knows the prior closure discussion without the student having to paste it back in.

---

### LangGraph
A Python framework for building AI workflows as directed graphs where each node is a function (a processing step) and edges define which node runs next. Unlike an agent that decides its own path, LangGraph graphs execute in a defined order, can pause at interrupt points for human input, and persist state across HTTP requests.

**Example:** QuizMe is a LangGraph StateGraph: `plan_objectives` → interrupt → `generate_quiz` → interrupt (per question) → `score_and_feedback` → `summarize` — every step defined, every pause point predictable.

---

### MemorySaver Checkpointer
LangGraph's mechanism for saving graph state to storage at each node boundary. When a graph hits an interrupt and the HTTP request completes, the state (all outputs so far, remaining steps, the student's answers) is serialised and stored under a `thread_id`. A later HTTP request provides the same `thread_id` and LangGraph resumes from the exact point it paused.

**Example:** A student approves their quiz plan on Monday and LangGraph suspends; on Thursday they resume from the same `thread_id` and their approved plan is still intact in MemorySaver — no data loss across the 72-hour gap.

---

### HITL (Human-in-the-Loop)
A design pattern where an automated system pauses execution at a defined point, presents its work to a human for review or input, and only continues after the human responds. The pause is architectural — the system cannot proceed past that point without human action, not merely a polite suggestion.

**Example:** QuizMe's first HITL interrupt presents the AI-generated list of learning objectives to the student; the quiz generation node literally cannot run until the student submits an approved plan via `POST /workflow/quizme/resume`.

---

### StateGraph
LangGraph's graph type for stateful workflows. Each node is a Python function that receives the current state dict and returns updates to it. Edges connect nodes; conditional edges choose the next node based on state values. The state is typed, versioned, and checkpointed at each step.

**Example:** QuizMe's StateGraph carries a state dict containing `{topic, course_id, objectives, current_question_index, answers, scores}` that every node reads from and writes to as the quiz progresses.

---

### Self-Improvement & Evaluation

### GEPA (Generative Evaluation and Prompt Advancement)
The offline pipeline that improves the tutor's SOUL.md system prompt automatically. It evaluates the current SOUL on a test set, identifies the weakest-scoring cases, prompts the model to reflect on those failures and propose a better SOUL, re-evaluates the candidate, applies a quality gate, then requires human approval before any change reaches production.

**Example:** If the current SOUL consistently scores poorly on questions about recursion — the model gives vague answers instead of concrete examples — GEPA identifies those as the weakest cases, proposes a SOUL revision that instructs the agent to always ground recursion explanations in a stack trace, then re-evaluates and waits for the operator to approve the diff.

---

### Pareto Gate
The automated quality check GEPA applies before sending a candidate SOUL to human review. The candidate must pass two conditions simultaneously: its average score across the full test set must exceed the current SOUL's average (overall improvement), and no individual test case can score below 2 out of 5 (no regression anywhere). A candidate that aces most cases but tanks one is rejected.

**Example:** If a proposed SOUL improves the average score from 3.6 to 3.9 but one test case about closures drops from 3 to 1, the Pareto gate rejects the candidate — that regression matters more than the average gain.

---

### RAGAS
An open-source framework for evaluating RAG pipelines. Given a question, the model's answer, and the retrieved context chunks, RAGAS uses a judge model to compute whether the answer's claims are supported by the context (faithfulness) and whether the retrieved chunks were actually useful (context precision). The01.dev runs RAGAS with a local model as judge — no external API calls.

**Example:** After the tutor answers a question about V8's garbage collector, RAGAS decomposes the answer into individual claims, checks each against the retrieved transcript chunks using the local Ollama model, and produces a faithfulness score the GEPA pipeline uses to measure improvement over time.

---

### Faithfulness
The RAGAS metric that measures whether every claim in the model's generated answer is supported by the retrieved context chunks. Computed by decomposing the answer into atomic statements and asking the judge model whether each statement is directly present in or inferable from the context. A faithfulness score of 0.6 means 40% of the answer's claims had no grounding in the retrieved material.

**Example:** If the tutor says "V8 uses a tri-colour marking algorithm" but the retrieved chunks only discuss generational collection, that statement fails the faithfulness check and pulls the score below 1.0.

---

### LLM Context Precision
The RAGAS metric that measures whether the retrieved chunks were actually useful for answering the question. If 3 of the 4 retrieved chunks were referenced in the answer and 1 was ignored, context precision is 0.75. Persistent low scores on this metric signal that the retrieval strategy (chunking size, index parameters, query formulation) is pulling in irrelevant material.

**Example:** If a student asks about the JS call stack and two of the four retrieved chunks are about the event loop instead, context precision will be low — those chunks didn't help — signalling that the chunking strategy needs adjustment to separate call-stack and event-loop content.

---

### Eval Harness
A fixed set of test cases (questions with expected answers) used to measure system quality at a point in time. Running the same harness before and after a change lets you detect regressions: if a SOUL change improves the average score on novel questions but drops performance on harness cases, the harness catches it.

**Example:** GEPA's eval harness contains 50 QA pairs drawn from the01.dev's courses — known questions with known good answers — used to score the current and candidate SOUL on every GEPA run.

---

### Model Serving

### Ollama
A tool for downloading and running open-source language models locally on a laptop or server. It exposes an OpenAI-compatible HTTP API, so any code that calls the OpenAI API can be redirected to Ollama with only a `base_url` change. No GPU required for small models.

**Example:** In dev, the01.dev's FastAPI backend points its generation client at `http://localhost:11434` (Ollama) running `qwen2.5:3b`, so the entire RAG tutor works with no external API calls or internet connection.

---

### vLLM
A high-throughput inference server for large language models, optimised for GPU hardware. It uses continuous batching to serve multiple concurrent requests efficiently — requests arriving at different times are grouped into batches, dramatically improving GPU utilisation over serial inference. Exposes the same OpenAI-compatible API as Ollama.

**Example:** In production, the01.dev swaps `OLLAMA_BASE_URL` for `VLLM_BASE_URL` pointing at a GPU server running vLLM — the same FastAPI generation code runs unchanged, but throughput scales to handle many concurrent students.

---

### OpenAI-Compatible API
An HTTP interface that follows the same request and response format as the OpenAI Chat Completions API. Any server that implements it (Ollama, vLLM, Groq, Together AI) can be used as a drop-in replacement for OpenAI by changing the `base_url` parameter — the application code stays the same.

**Example:** The01.dev's `generate_answer()` function was written against the OpenAI SDK; switching from dev (Ollama) to prod (vLLM) requires changing one environment variable — `GENERATION_BASE_URL` — and zero lines of application code.

---

### Sovereign Inference
Running language model generation entirely on infrastructure you control, with no data leaving your servers to a third-party model provider. The privacy guarantee is architectural: there is nothing to breach because the data never travels outside your network.

**Example:** Student questions about proprietary course content — and the course transcripts used as context — are processed by Ollama or vLLM on the01.dev's own hardware; they are never sent to OpenAI, Anthropic, or any external model API during generation.

---

### text-embedding-3-small
OpenAI's embedding model, producing 1536-dimensional vectors from input text. It converts a piece of text into a fixed-length numerical representation that captures semantic meaning. As of the current01.dev architecture, this is the one remaining external call — used to embed student questions for retrieval.

**Example:** Every student question passes through `text-embedding-3-small` before HNSW search; the resulting 1536-dimensional vector is what the vector store searches against to find relevant transcript chunks.

---

### Governance & Privacy

### EU-AI-Act Audit Log
A tamper-evident record written to storage before every model inference, capturing who asked what, which model answered, and which version of the SOUL was active. Writing before inference (not after) means the record exists even if the process crashes mid-generation — making it impossible to retroactively hide that an inference was attempted.

**Example:** Before calling Ollama, the01.dev inserts `{uid, question, model: "qwen2.5:3b", soul_hash: "sha256:abc...", ts, answer: null}` into the audit log; the `answer` field is filled in only after generation completes, so a crash between the two events is clearly visible in the log.

---

### SHA Hash
A cryptographic fingerprint of a file's contents. Running SHA-256 on SOUL.md produces a fixed-length string that changes if even one character in the file changes. Storing the hash in the audit log at inference time proves which exact version of the agent identity was active for that request.

**Example:** If SOUL.md is updated after GEPA, the new file gets a new SHA-256 hash; every audit row after that update carries the new hash, making it trivial to see exactly when the SOUL changed and which student interactions happened under which version.

---

### GDPR (General Data Protection Regulation)
European Union law that gives individuals rights over their personal data — including the right to see what data an organisation holds about them and the right to have it deleted. Any product serving EU users must implement mechanisms to honour these rights.

**Example:** The01.dev's `DELETE /api/hermes/memory/{id}` endpoint is the GDPR erasure implementation — it deletes all `student_turns` rows and resets the `student_profile` for a given user ID, while retaining (but nulling the content of) audit log rows per legal obligation.

---

### Tamper-Evident
A property of a record or log where the design makes retroactive modification detectable or impossible. For audit logs, tamper-evidence is achieved by writing the record before the event it documents — so the record exists regardless of what happens next, and its timestamp precedes the event it describes.

**Example:** The01.dev's audit row is written with `answer: null` before inference starts; if the process crashes post-write but pre-generation, the row shows an attempted inference with no answer — honest accounting that cannot be erased after the fact.

---

### Payments & Auth

### Razorpay
An Indian payment gateway that handles credit/debit card processing, UPI, and net banking. The backend creates a payment order using the secret key (server-side only), returns a public order ID to the client, and then verifies the payment signature via HMAC-SHA256 when the user completes checkout.

**Example:** When a student clicks "Buy Track", FastAPI uses the Razorpay secret key to create an order for ₹99900 (paise), returns the `order_id` to the React client, and the Razorpay checkout modal handles the actual payment — the secret key never leaves the server.

---

### Firebase Auth
Google's authentication-as-a-service. Users sign in (typically with Google Sign-In) and receive a short-lived JWT id token on the client. Every FastAPI endpoint that requires authentication calls `firebase_admin.auth.verify_id_token(token)` to validate the token against Google's public keys — no per-request Google network call after the initial key fetch.

**Example:** Every request to `POST /api/hermes/chat` includes a Firebase id token in the Authorization header; FastAPI verifies it with `firebase_admin` and extracts the `uid` to check course purchase status before touching the RAG pipeline.

---

### Firestore
Google's serverless NoSQL document database, organised as a hierarchy of collections and documents. Used in production for student profiles, conversation turns, purchase records, and the audit log. Each document is a JSON-like object; collections can be queried, and `arrayUnion` atomically appends to an array field.

**Example:** When a student completes a Razorpay payment, FastAPI calls `Firestore.arrayUnion(track_id)` on the user's document — adding the purchased track ID to `user.purchases` atomically, so a concurrent purchase of a second track can never overwrite the first.
