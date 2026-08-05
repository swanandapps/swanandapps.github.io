# CodeMas — System Design

**Platform:** Real-time coding assessment at scale  
**Role:** First engineer under CTO, Masai School  
**Scale:** 10,000 concurrent students, ~10K submissions/sec at exam deadline bursts

---

## Problem Statement

Masai School runs programming exams for cohorts of 400 students simultaneously. Before CodeMas, exams were manual — trainers reviewed submissions by hand, plagiarism was undetected, and results took days. The platform needed to:
- Execute untrusted student code safely and return results in under 10 seconds
- Support 10,000 concurrent users without degrading
- Detect plagiarism at scale automatically
- Enable AI-assisted evaluation without blocking the execution path

---

## Functional Requirements

- Student submits code → receives execution result (pass/fail + output) in < 10s
- Multiple language support (Python, JavaScript, Java, C++)
- Attempt gating — max N attempts per question per exam
- Real-time result streaming to browser (no manual refresh)
- Trainer dashboard — live exam progress, cohort analytics
- Plagiarism detection — triggered automatically at exam close
- AI features — exam generation, rubric scoring, Socratic hints, trainer narratives

## Non-Functional Requirements

- **Latency:** Submission to result in < 10 seconds p95
- **Throughput:** Handle 400 simultaneous submissions at exam deadline (burst)
- **Isolation:** One student's code cannot affect another's execution environment
- **Availability:** 99.9% during exam windows
- **Safety:** Untrusted code cannot escape sandbox, consume unbounded resources, or access network

---

## High-Level Architecture

```
┌──────────────┐      POST /submit       ┌──────────────────┐
│   Browser    │ ──────────────────────► │   Django API     │
│  (Vue 3)     │                         │   (Gunicorn)     │
│              │ ◄─── SSE stream ──────  │                  │
└──────────────┘                         └────────┬─────────┘
                                                  │ Enqueue
                                                  ▼
                                         ┌──────────────────┐
                                         │   AWS SQS FIFO   │
                                         │  (per-exam queue) │
                                         └────────┬─────────┘
                                                  │ Trigger
                                                  ▼
                                         ┌──────────────────┐
                                         │  AWS Lambda      │
                                         │  (code sandbox)  │
                                         └────────┬─────────┘
                                                  │ Write result
                                                  ▼
                                         ┌──────────────────┐
                                         │   PostgreSQL     │
                                         │  (submissions +  │
                                         │   results)       │
                                         └──────────────────┘
```

---

## Detailed Component Design

### 1. Django API Layer

**Responsibilities:** Auth, submission intake, attempt gating, SSE endpoint, AI feature triggers.

**Submission endpoint (`POST /api/submit`):**
1. Authenticate student (Simple JWT)
2. `SELECT FOR UPDATE` on attempt counter row — lock before reading
3. Check `attempt_count < max_attempts` — reject if exceeded
4. `INSERT` Submission row (status: QUEUED)
5. Enqueue message to SQS FIFO (exam-specific queue)
6. Increment attempt counter
7. Return `201 Accepted` immediately — never waits for execution

**Why 201 immediately:** API latency must be flat regardless of code complexity. Async execution means no coupling between HTTP response time and code runtime.

**SSE endpoint (`GET /api/submissions/{id}/stream`):**
- Opens `text/event-stream` response
- Polls Postgres every 500ms for status change
- On `COMPLETED` or `FAILED` — pushes result, closes stream
- Browser uses native `EventSource` with automatic reconnection

**Why Postgres polling instead of Redis pub/sub:** Lambda is stateless — it cannot push to a Redis channel that a Django worker is listening on without a persistent process. Polling Postgres keeps the SSE handler stateless and Lambda-compatible. 500ms polling latency is acceptable.

---

### 2. SQS FIFO Queue

**One queue per exam.** Messages contain: `submission_id`, `student_id`, `exam_id`, `language`, `code`.

**Why FIFO:** Ordering matters for attempt replay. Deduplication ID prevents double-execution if API retries the enqueue.

**Why SQS over Redis list:**
- At exam deadline (400 submissions in 30 seconds), SQS absorbs the burst without stalling the API
- At-least-once delivery with deduplication prevents double execution
- Per-exam isolation — one exam's queue doesn't affect another
- Redis requires a persistent process (Celery worker) to consume; SQS triggers Lambda directly

---

### 3. Lambda Sandbox (Code Execution)

Each Lambda invocation handles exactly one submission.

**Execution flow:**
1. Dequeue message from SQS
2. Write code to `/tmp/solution.{ext}`
3. Spawn subprocess with resource limits: CPU 15s timeout, memory 512MB
4. Run against test cases
5. Capture stdout, stderr, exit code
6. Write result to Postgres (`UPDATE submissions SET status='COMPLETED', output=...`)
7. Invocation ends — environment is destroyed

**Why Lambda as sandbox:**
- Per-invocation isolation — no shared memory, no shared filesystem between students
- Hard 15-second timeout enforced by AWS — no infinite loop escapes
- Ephemeral — nothing persists between invocations
- Scales automatically — no worker pool management
- Zero host management — no Docker daemon, no container cleanup

**Trade-off vs Docker on persistent host:**
- Lost: Redis pub/sub for real-time result push (replaced by Postgres polling)
- Gained: true isolation, elastic scale, zero ops overhead
- Cold start risk: ~200ms on cold Lambda. Mitigated by keeping Lambdas warm during exam windows via scheduled pings.

---

### 4. PostgreSQL

**Core tables:**

```sql
submissions (
  id UUID PK,
  student_id FK,
  exam_id FK,
  question_id FK,
  code TEXT,
  language VARCHAR,
  status ENUM(QUEUED, RUNNING, COMPLETED, FAILED),
  output TEXT,
  exit_code INT,
  runtime_ms INT,
  attempt_number INT,
  created_at TIMESTAMP
)

attempt_counters (
  student_id FK,
  question_id FK,
  exam_id FK,
  count INT,
  PRIMARY KEY (student_id, question_id, exam_id)
)

results (
  submission_id FK,
  test_case_id FK,
  passed BOOLEAN,
  actual_output TEXT,
  expected_output TEXT
)
```

**Concurrency: SELECT FOR UPDATE**

```sql
BEGIN;
SELECT count FROM attempt_counters
WHERE student_id = $1 AND question_id = $2 AND exam_id = $3
FOR UPDATE;  -- locks this row
-- check count < max, then:
UPDATE attempt_counters SET count = count + 1 ...;
INSERT INTO submissions ...;
COMMIT;
```

This prevents two simultaneous tab submissions (both arriving in the same millisecond) from both succeeding past the attempt limit.

---

### 5. Architecture Migration: Redis + Celery + Docker → Lambda + SQS

**Original stack:** Django → Redis LPUSH → Celery worker (BRPOP) → Docker container → Redis pub/sub → SSE

**Migration rationale:**

| Dimension | Redis + Celery + Docker | Lambda + SQS |
|---|---|---|
| Sandbox isolation | Docker on shared host (process-level) | Per-invocation (true ephemeral) |
| Scaling workers | Manual — add Celery workers | Automatic — Lambda scales with queue depth |
| Result delivery | Redis pub/sub (fast) | Postgres poll 500ms (acceptable) |
| Host management | Persistent worker processes | Zero — no processes to manage |
| Burst handling | Queue backs up; workers are fixed | Lambda fans out automatically |
| Cold start | None (always-on workers) | ~200ms (mitigated with warmers) |

**What we gave up:** Real-time result push via pub/sub — replaced by 500ms Postgres polling. The UX impact is negligible at the student level.

---

### 6. Plagiarism Detection System

**The reframe:** Not "are two submissions similar?" (O(N²)) but "did this student write this?" (behavioral signals first, similarity second).

#### Phase 1 — Behavioural Analysis (O(N), synchronous at exam close)

Triggered by Django `pre_save` signal on `Exam.is_active` transitioning `True → False`.

For each submission, compute a risk score:

```
risk = 0.40 × paste_ratio
     + 0.30 × speed_anomaly_score
     + 0.15 × tab_switch_score
     + 0.15 × attempt_surprise_score
```

- **Paste ratio:** `paste_char_count / total_chars` — high paste ratio suggests external copy
- **Speed anomaly:** `time_to_complete / baseline_for_difficulty` — too fast for the difficulty level
- **Tab switches:** normalised count during exam — correlates with looking up answers
- **Attempt surprise:** passing attempt came very late — suggests trying many approaches (testing known solutions)

Risk > 0.2 → `BehaviouralFlag` written to DB. Runs O(N) — one pass, no comparisons between students.

#### Phase 2 — Similarity Analysis (O(K×N), queued after Phase 1)

Only submissions with HIGH behavioural risk (`risk > 0.5`) become suspects (K). Each suspect is compared against the full cohort (N) using TF-IDF cosine similarity.

```
TF-IDF vectorise all submissions per question
For each suspect k:
  cosine_similarity(k, all_N_submissions)
  flag pairs above 0.80 threshold
```

**Why TF-IDF over alternatives:**
- **MOSS:** Rate-limited API, external dependency, Java/C/Python only
- **CodeBERT:** Requires GPU, expensive, adds infrastructure complexity
- **AST comparison:** Language-specific, need separate parser per language
- **TF-IDF:** Language-agnostic, no GPU, interpretable output (shared tokens visible), fast, accurate at 2K submissions

**Result:** 5% → 95% detection rate (19× lift).

---

### 7. AI Features Layer

All 5 are **single-shot LLM calls**, event-triggered, completely off the submission critical path.

| Feature | Trigger | Latency impact |
|---|---|---|
| Rubric Scoring | Every failed submission | Zero — async after result delivered |
| Socratic Hint | Failed submission | Zero — async |
| Exam Generator | Trainer request | Zero — trainer-facing only |
| Trainer Narrative | Cohort summary request | Zero — trainer-facing only |
| Plagiarism Engine | Exam close | Zero — post-exam batch |

**Design principle:** AI results arrive after the student already has their execution result. No AI call is on the critical submission path. If GPT is down, students still get results — they just don't get rubric scores or hints.

**Fallback:** Rubric and hint features degrade gracefully — missing enrichment, not missing results.

---

### 8. Key Engineering Decisions Summary

| Decision | Choice | Alternative | Why |
|---|---|---|---|
| Execution sandbox | Lambda invocation | Docker on EC2 | Ephemeral isolation, elastic scale, zero ops |
| Queue | SQS FIFO | Redis list / Celery | Burst absorption, deduplication, per-exam isolation |
| Result delivery | SSE + Postgres poll | WebSockets / Redis pub/sub | Unidirectional, Lambda-compatible, stateless SSE handler |
| Attempt gating | SELECT FOR UPDATE | Application-level check | Race condition prevention at DB level |
| Plagiarism Phase 1 | Behavioral signals | Similarity first | O(N) vs O(N²); reframes the problem |
| Plagiarism Phase 2 | TF-IDF cosine | CodeBERT / MOSS | No GPU, language-agnostic, interpretable |
| API response | 201 immediate | Wait for result | Flat API latency regardless of code complexity |
| AI placement | Async event-triggered | Inline on submit | Zero latency added, graceful degradation if AI is down |

---

### 9. Scalability Considerations

**Current scale:** 10K concurrent users, 400 submissions/30s burst.

**Next bottleneck (100K users):**
- SSE polling: 100K × 2 reads/sec = 200K Postgres reads/sec → needs Redis Streams or WebSocket push
- Lambda concurrency: AWS default limit 1,000 concurrent executions per account → request limit increase or multi-account
- Postgres: connection pool exhaustion → PgBouncer, read replicas for analytics queries
- SQS: no bottleneck — handles millions of messages/sec

**Multi-tenancy (SaaS):**
- Add `school_id` to every table (pool model) with Postgres RLS
- Per-school SQS queues (already per-exam — extend to per-school namespace)
- Per-school Lambda concurrency limits (reserved concurrency)
- Plagiarism pool: per-school isolation — schools must not share a similarity pool

---

### 10. What I'd Do Differently

- **Replace Postgres polling with Redis Streams** for SSE — at 100K connections, 500ms polling becomes expensive; Streams push model eliminates the read load
- **Add a result cache (Redis)** — if a student refreshes mid-stream, serve from cache instead of re-polling
- **Language runtime containers pre-warmed** — Lambda cold starts hurt the first submission of an exam; pre-warm with scheduled invocations before exam start
- **Structured plagiarism evidence** — currently flags pairs; would add a side-by-side diff UI for reviewer verification
