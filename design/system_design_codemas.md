# CodeMas — System Design

---

## Quick Reference

| Field            | Detail                                                                  |
|------------------|-------------------------------------------------------------------------|
| Platform         | Real-time coding assessment — secure, proctored online exams            |
| My Role          | Sole engineer — architecture, backend, frontend, infra, deployment      |
| Scale            | 10,000 concurrent students; ~10K submissions/sec at exam-deadline bursts |
| Submission SLA   | Result delivered to browser within 10s (p95 target)                    |
| Result delivery  | Client-side polling — frontend polls Postgres result via API every 1–2s |
| Plagiarism       | Two-phase post-exam pipeline; 19× improvement in catch rate             |
| AI Features      | 5 LLM features; zero AI on the submission critical path                 |
| Stack            | Vue 3 + Vite · Django 4.2 + DRF · AWS SQS FIFO · AWS Lambda · Postgres  |

---

## Problem Statement

Trainers need to run coding exams for cohorts of up to 10,000 students simultaneously.
Every student submits code that must be executed in isolation, graded, and returned to the
browser — in near real-time — without any student being able to cheat by viewing or copying
another student's solution.

Three hard constraints shape the entire architecture:

1. **Isolation** — code from different students must never share a process or filesystem.
2. **Fairness** — every submission attempt must be counted exactly once; retries on network
   failure must not re-execute code.
3. **Scale** — submission volume spikes sharply at the final minutes of an exam when many
   students submit simultaneously. The system must absorb that burst without queuing so deep
   that results arrive after the exam closes.

---

## Functional Requirements

- Students can submit code for a question during an active exam window.
- Each submission is executed in an isolated sandbox and given a pass/fail verdict with output.
- The student's browser receives the result without requiring a manual page refresh.
- A student is allowed at most N attempts per question (enforced server-side, not client-side).
- Trainers see a live dashboard showing submission counts updated approximately every 10 seconds.
- After an exam closes, a plagiarism pipeline runs automatically — no manual trigger required.
- Trainers can use AI tools for rubric scoring, plagiarism explanation, and exam generation.
- Students receive AI-generated hints on failed attempts and an exam performance summary.

## Non-Functional Requirements

- **Throughput:** sustain ~10K submissions/sec during deadline bursts.
- **Latency:** result delivered to browser within 10s (p95) under normal load.
- **Isolation:** one Lambda invocation per submission; no shared state between students.
- **Exactly-once execution:** a submission retried due to a network blip must not run twice.
- **Availability:** exam delivery is higher priority than plagiarism analysis. Plagiarism pipeline
  failure must not affect the exam itself.
- **No data loss:** every submission and its result is durably written to Postgres.

---

## High-Level Architecture

```
                     +--------------------------------------------------+
                     |               STUDENT BROWSER                    |
                     |   Vue 3 SPA                                      |
                     |   POST /submit → 201 → polls GET /result/{id}/  |
                     |   every 1–2s until status != PENDING            |
                     +-------------------+------------------------------+
                                         |
                              POST /api/submissions/create/
                                         |
                                         v
                     +-----------------------------------------------------+
                     |                   DJANGO 4.2 API                    |
                     |  * Validates JWT (Simple JWT)                       |
                     |  * SELECT FOR UPDATE → attempt counter check        |
                     |  * INSERT submission row (status=PENDING)           |
                     |  * SQS.send_message → FIFO queue                   |
                     |  * Returns 201 Accepted                             |
                     |                                                     |
                     |  GET /api/submissions/{id}/                         |
                     |  * Returns current status + result JSON             |
                     |  * Frontend polls this until terminal status        |
                     +------------------+----------------------------------+
                                        |
                              SQS.send_message
                                        |
                                        v
                          +----------------------------+
                          |   AWS SQS FIFO Queue       |
                          |  MessageGroupId=exam_id    |
                          |  Deduplication on sub UUID |
                          +------------+---------------+
                                       |
                            Lambda trigger (batch=1)
                                       |
                                       v
                          +----------------------------+
                          |   AWS Lambda (Sandbox)     |
                          |  * Ephemeral per invoke    |
                          |  * Runs student code via   |
                          |    subprocess with limits  |
                          |  * Hard 15s AWS timeout    |
                          |  * Writes result to PG     |
                          +------------+---------------+
                                       |
                          UPDATE submissions SET status='COMPLETE', result=...
                                       |
                                       v
                          +----------------------------+
                          |       PostgreSQL            |
                          |  submissions table         |
                          |  attempt_counters table    |
                          |  exams, questions, users   |
                          +----------------------------+
                                       |
                          Frontend poll hits this via
                          GET /api/submissions/{id}/

                  +--------------------------------------------------+
                  |         POST-EXAM PLAGIARISM PIPELINE             |
                  |  Triggered by pre_save signal on Exam.is_active  |
                  |  Phase 1 (Behavioral) → Phase 2 (TF-IDF+cosine) |
                  +--------------------------------------------------+

                  +--------------------------+
                  |   TRAINER DASHBOARD      |
                  |  Single fetch on mount   |
                  |  Plagiarism tab polls 5s |
                  |  (stops when no running) |
                  +--------------------------+
```

---

## Detailed Component Design

### 1. Django API Layer

**Submission endpoint — `POST /api/submissions/create/`**

The endpoint does four things in sequence before returning:

1. **JWT validation** — Simple JWT verifies the token. If expired or tampered, 401 immediately.
   No database hit required for this check.

2. **Attempt gate (SELECT FOR UPDATE)**

   ```sql
   BEGIN;
   SELECT attempt_count FROM attempt_counters
   WHERE student_id = %s AND question_id = %s
   FOR UPDATE;
   -- if attempt_count >= max_attempts: rollback, return 429
   UPDATE attempt_counters SET attempt_count = attempt_count + 1
   WHERE student_id = %s AND question_id = %s;
   COMMIT;
   ```

   The `FOR UPDATE` lock ensures that two simultaneous submissions from the same student
   (double-click, network retry, two open tabs) cannot both slip past the attempt gate.
   Only one transaction holds the lock at a time; the other waits and then reads the
   already-incremented counter.

3. **Insert submission row** — status=`PENDING`, stores question_id, student_id, submitted_code,
   timestamp. This row is the source of truth. Lambda will update it; the frontend will poll it.

4. **Enqueue to SQS FIFO** — `MessageGroupId=exam_id` keeps submissions for the same exam
   in order. `MessageDeduplicationId` is set to the submission's UUID (generated at insert).
   If the same message is sent twice within the 5-minute deduplication window (network retry),
   SQS silently drops the duplicate. Lambda runs exactly once.

Returns **201 Accepted** — the result is not yet ready; the frontend begins polling.

---

**Result polling endpoint — `GET /api/submissions/{id}/`**

Simple read on the submissions table by primary key. Returns the current status and, once
complete, the full result JSON (verdict, stdout, stderr, execution_time_ms).

Frontend polls every 1–2 seconds. When `status != 'PENDING'`, it renders the result and stops polling.

This is the correct architecture after the SQS + Lambda migration. SSE was removed because its push advantage (Redis pub/sub channel) no longer existed — after Lambda writes to Postgres, the server has no way to know the result is ready without polling Postgres itself. A server that polls Postgres every 500ms and forwards via SSE is architecturally equivalent to the client polling Postgres directly. Removing SSE simplifies the stack: no StreamingHttpResponse, no long-lived connections to manage, no proxy timeout configuration (`X-Accel-Buffering: no`). Client polling is the honest, minimal architecture.

---

**Trainer dashboard endpoint — `GET /api/dashboard/{exam_id}/`**

Aggregates submission counts, pass rates, and per-question breakdowns. Result cached in Redis
with TTL=10s. Trainer frontend fetches once on mount. The plagiarism status sub-tab polls every
5 seconds while any plagiarism run shows `status='running'`; polling stops automatically when
all runs reach a terminal state.

---

### 2. SQS FIFO Queue

**Why FIFO over Standard?**

Standard SQS can deliver messages out of order and may deliver the same message more than once.
For exam submissions, out-of-order delivery is acceptable, but double-execution is not — running
a student's code twice wastes Lambda budget and can create duplicate result rows.

FIFO queues provide:
- **Exactly-once delivery** within the deduplication window (5 minutes). If Django retries
  the `send_message` call with the same `MessageDeduplicationId`, SQS discards the duplicate.
- **Per-group ordering** via `MessageGroupId`. Submissions for the same exam are processed
  in arrival order, so earlier submitters get results earlier.

**Throughput:** A single SQS FIFO queue supports ~3,000 msg/sec per message group.
With `MessageGroupId=exam_id`, multiple exams running simultaneously each have their own
throughput lane. If a single exam exceeds this, the mitigation is to shard by
`exam_id + student_id_bucket`.

---

### 3. Lambda Sandbox

Lambda is the code execution engine. Each invocation is an isolated ephemeral container.

**What happens inside a Lambda invocation:**

1. Receive SQS event — extract submission_id, student code, language, question test cases.
2. Write student code to `/tmp/solution.py` (or relevant extension).
3. Execute via `subprocess.run(["python", "/tmp/solution.py"], capture_output=True, timeout=10)`.
4. Compare stdout against expected output per test case. Build verdict: PASS/FAIL + actual output.
5. `UPDATE submissions SET status='COMPLETE', result=..., executed_at=NOW() WHERE id=...`
6. Return. Container is reclaimed. `/tmp` is gone.

**Hard constraints:**
- AWS maximum Lambda timeout: 15 minutes per invocation. We configure 15s — enough for
  legitimate code, prevents runaway infinite loops.
- Memory capped at 256MB for the code sandbox function.
- No network egress from the Lambda — student code cannot make HTTP calls or phone home.
  Achieved via VPC placement with no internet gateway or a restrictive security group.
- `/tmp` (512MB by default) is per-invocation scratch space. Not shared between invocations.

**Concurrency:** Lambda scales horizontally. 10,000 simultaneous submissions = up to 10,000
concurrent Lambda invocations. AWS account-level concurrency limit (default 1,000 per region)
must be raised before an exam at this scale.

---

### 4. Postgres Schema (Key Tables)

```sql
CREATE TABLE submissions (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id    INTEGER REFERENCES users(id),
    question_id   INTEGER REFERENCES questions(id),
    exam_id       INTEGER REFERENCES exams(id),
    code          TEXT NOT NULL,
    language      VARCHAR(20) NOT NULL,
    status        VARCHAR(20) NOT NULL DEFAULT 'PENDING',  -- PENDING | COMPLETE | ERROR
    result        JSONB,        -- { verdict, stdout, stderr, execution_time_ms }
    submitted_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    executed_at   TIMESTAMPTZ
);

CREATE TABLE attempt_counters (
    student_id    INTEGER REFERENCES users(id),
    question_id   INTEGER REFERENCES questions(id),
    attempt_count INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (student_id, question_id)
);

CREATE TABLE behavioural_flags (
    id            SERIAL PRIMARY KEY,
    student_id    INTEGER REFERENCES users(id),
    exam_id       INTEGER REFERENCES exams(id),
    risk_score    FLOAT NOT NULL,
    signals       JSONB,  -- { paste_ratio, speed_anomaly, tab_switches, attempt_surprise }
    flagged_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE similarity_flags (
    id                  SERIAL PRIMARY KEY,
    exam_id             INTEGER REFERENCES exams(id),
    question_id         INTEGER REFERENCES questions(id),
    student_a_id        INTEGER REFERENCES users(id),
    student_b_id        INTEGER REFERENCES users(id),
    similarity_score    FLOAT NOT NULL,
    flagged_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Key indexes:**
```sql
-- Frontend polls this on every request; must be instant
CREATE INDEX idx_submissions_id_status ON submissions(id, status);

-- Plagiarism Phase 2 needs all submissions for a question
CREATE INDEX idx_submissions_exam_question ON submissions(exam_id, question_id);
```

---

### 5. Result Delivery — Client Polling

After receiving 201, the Vue frontend starts a polling loop:

```javascript
// Vue frontend (ExamTakeView.vue)
async function pollResult(submissionId) {
  const response = await api.get(`/submissions/${submissionId}/`)
  if (response.data.status !== 'PENDING') {
    finalizeResult(response.data)   // render verdict + test case breakdown
    return
  }
  setTimeout(() => pollResult(submissionId), 1500)  // poll every 1.5s
}
```

When the result arrives (status = 'COMPLETE' or 'ERROR'), the frontend stops polling and
renders the verdict. No long-lived connections. No server-side streaming. No proxy configuration.

**Why client polling is the right answer here (not SSE):**

With the SQS + Lambda architecture, Lambda writes the result to Postgres and exits. The Django
server has no event it can subscribe to that tells it "this submission is done." The only way
for the server to know is to poll Postgres. A Django SSE handler that polls Postgres every 500ms
on the client's behalf is doing the same work as the client polling directly — it just adds
a persistent HTTP connection and server-side polling overhead on top. Removing SSE eliminated:
- Long-lived HTTP connections (Gunicorn worker blocked for the duration)
- Proxy timeout configuration (`proxy_read_timeout`, `X-Accel-Buffering`)
- The polling-theater abstraction where "push" was actually polling

The honest mechanism is client polling. If true push is needed at scale, the correct investment
is a dedicated WebSocket layer (API Gateway WebSocket, Pusher) where Lambda explicitly notifies
the connection manager after writing to Postgres.

**Scale comparison:**

| Mechanism | Postgres reads at 10K concurrent users | Server connections | Complexity |
|---|---|---|---|
| SSE + Postgres poll (removed) | 20,000 reads/sec (server polls on behalf of each client) | 10K persistent HTTP | StreamingHttpResponse, X-Accel-Buffering, proxy timeout config |
| **Client polling (current)** | ~6,700 reads/sec (client polls every 1.5s) | Stateless HTTP requests | Standard REST endpoint, no special config |
| WebSocket push (ideal at scale) | Only Lambda writes (~10K writes/exam burst) | WebSocket connections via Gateway | Lambda → push notification → client |

Client polling is ~33% fewer Postgres reads than SSE polling was, because the interval is 1.5s
vs 500ms. It also makes the server stateless — Gunicorn workers are not tied up holding open
connections.

---

### 6. Plagiarism Detection Pipeline

Triggered automatically by a Django `pre_save` signal on the `Exam` model:

```python
@receiver(pre_save, sender=Exam)
def trigger_plagiarism_on_exam_close(sender, instance, **kwargs):
    try:
        prev = Exam.objects.get(pk=instance.pk)
    except Exam.DoesNotExist:
        return
    if prev.is_active and not instance.is_active:
        run_exam_plagiarism_check.delay(instance.id)
```

Signal fires regardless of how the exam is closed — API, admin panel, management command.

---

**Phase 1 — Behavioural Scoring**

Runs synchronously inside `run_exam_plagiarism_check` at exam close. Four signals, each
returning a float 0.0–1.0:

| Signal | Weight | Calculation |
|---|---|---|
| `paste_signal` | 0.40 | `paste_char_count / len(code)` — ≥0.7 → 1.0; ≥0.5 → 0.7; ≥0.3 → 0.3 |
| `time_signal` | 0.30 | Actual time vs difficulty baseline — ≤0.2× baseline → 1.0; ≤0.4× → 0.7; ≤0.6× → 0.3 |
| `tab_switch_signal` | 0.15 | Browser focus-loss events — ≥5 → 1.0; ≥3 → 0.5; ≥1 → 0.2 |
| `submission_surprise_signal` | 0.15 | `SequenceMatcher(prev_code, curr_code).ratio()` — low similarity + passed → 1.0 |

Weights come from the `PlagiarismPolicy` model (Redis-cached, TTL 60s — allows live tuning
without a deploy). `BehaviouralFlag` stored only if `risk_score > min_risk_score` (default 0.2).

Confidence label assignment:
- 3+ signals fired → `confidence = 'low'` (low confidence in innocence — most suspicious)
- 2 signals → `confidence = 'medium'`
- 0–1 signals → `confidence = 'high'` (high confidence they wrote it themselves — clean)

---

**Phase 2 — Similarity Analysis**

One `run_question_plagiarism_check` task queued per question, staggered 30s apart
(`countdown = question_index * 30`) to avoid hammering Postgres simultaneously.

1. Fetch `BehaviouralFlag` records for this exam+question as the suspect set K.
2. Fetch the best submission per student across the full cohort N (passed attempt if any, else latest).
3. **Code normalization** before TF-IDF:
   - Python: `ast.NodeTransformer` strips comments/docstrings, renames all user-defined
     variables to `VAR_0, VAR_1, ...` — defeats trivial variable renaming tricks.
   - JavaScript: regex substitution of identifiers.
   - Other languages: whitespace normalization only.
4. TF-IDF vectorize all N submissions for the question.
5. For each suspect k in K: compute cosine similarity between k's vector and all N vectors.
6. Flag pairs above the similarity threshold (from `PlagiarismPolicy`).
7. Create `PlagiarismFlag` with `detection_method='behavioural_filtered_tfidf'`.
8. Queue `explain_plagiarism.delay(flag_id)` for each new flag — AI writes the prose explanation.

**Why O(K×N), not O(N²):**

At N=10,000 students and 20 questions, O(N²) = 2 billion pairwise comparisons per question,
or 40 billion total. This takes minutes per question. Behavioral pre-filter reduces K to ~5–10%
of the cohort, dropping compute by 90%. O(K×N) with K=500 and N=10,000 = 5 million comparisons
per question — seconds, not minutes.

---

### 7. AI Features — Full Implementation Detail

Five LLM features. All use GPT-4o-mini (`OPENAI_MODEL` env var). All use `response_format={"type": "json_object"}`, `temperature=0.2`, 30s timeout. All are best-effort — `LLMUnavailable` exceptions are logged and swallowed, never propagated to the student. **Zero AI on the submission critical path.**

---

**Feature 1 — Rubric Scoring**

| Field | Detail |
|---|---|
| Trigger | After every submission (pass or fail), inside `execute_submission` task |
| Endpoint | `GET /api/ai/rubric/{submission_id}/` |
| Model | GPT-4o-mini |
| Input to LLM | Problem statement, student code, language, test case results |
| Output shape | `{correctness: {score, note}, code_quality: {score, note}, approach: {score, note}, edge_cases: {score, note}, overall_feedback}` |
| Scoring | 4 dimensions, each 0–2. `correctness` is tied directly to test results |
| Poll pattern | Frontend polls at 2s interval, max 15 attempts. Returns `{"status":"pending"}` until stored |
| Idempotent | Yes — won't re-score if `RubricScore` row already exists for this submission |
| Failure mode | Rubric tab shows error message. Submission result unaffected. |

---

**Feature 2 — Socratic Hint**

| Field | Detail |
|---|---|
| Trigger | Only on FAILED submissions, inside `execute_submission` task |
| Endpoint | `GET /api/ai/hint/{submission_id}/` |
| Model | GPT-4o-mini |
| Input to LLM | Problem statement, student code, language, failed test case (input / expected / actual output) |
| Prompt constraint | "Patient coding tutor" — 1–3 sentence nudge. Must NOT reveal the answer or write code. Ask a question, don't give the solution. |
| Output shape | `{"hint": "<1-3 sentences>"}` |
| Poll pattern | Same 2s / 15-attempt pattern as rubric |
| Idempotent | Yes — `AIHint` row is reused if it exists |
| Failure mode | Hint section shows error. Student still sees their failed test output. |

---

**Feature 3 — Plagiarism Explanation**

| Field | Detail |
|---|---|
| Trigger | When Phase 2 creates a new `PlagiarismFlag`, `explain_plagiarism.delay(flag_id)` is queued |
| Endpoint | `GET /api/ai/plagiarism-explanation/{flag_id}/` (trainer-only) |
| Model | GPT-4o-mini |
| Input to LLM | Both students' normalized code, their behavioural signals, cosine similarity score |
| Prompt framing | "Evidence not verdict" — describe WHY this pair is suspicious, not whether they cheated |
| Output shape | `{summary: "...", matches: ["specific pattern 1", "specific pattern 2", ...], confidence: "low\|medium\|high"}` |
| Idempotent | Yes — `PlagiarismExplanation` row reused |
| Failure mode | Trainer sees "explanation unavailable." Flag itself still visible with raw score. |

---

**Feature 4 — Exam Performance Summary**

| Field | Detail |
|---|---|
| Trigger | On first `GET /api/ai/exam-summary/{exam_id}/` request (student-facing) |
| Model | GPT-4o-mini |
| Input to LLM | Per-question outcomes (pass/fail/attempts), rubric scores across all questions |
| Prompt framing | 3–4 sentences addressed to "you." Note overall performance, one strength, one area to improve. |
| Output shape | `{"summary": "<3-4 sentences>"}` |
| Caching | Result stored in `ExamSummary` model; subsequent requests return stored version, no LLM call |
| Failure mode | Summary section shows error. Exam results page still works. |

---

**Feature 5 — AI Exam Draft Generation**

| Field | Detail |
|---|---|
| Trigger | `POST /api/ai/generate-exam/` (trainer-only) |
| Model | GPT-4o-mini |
| Input | Topic, programming language, difficulty level |
| Output shape | `{title, description, questions: [{title, description, difficulty, test_cases: [{input_data, expected_output}]}]}` |
| Persistence | Does NOT persist — returns draft JSON for trainer to review and edit before saving via `POST /api/exams/create-full/` |
| Fallback | Returns 503 if `OPENAI_API_KEY` is not set |
| Failure mode | Trainer sees error message. Creating exams manually still works. |

---

**AI Features — Architecture principle**

All five AI features are intentionally outside the submission critical path:

```
Submission critical path (no LLM):
Student → POST /submit → SQS → Lambda → Postgres → GET /result poll → browser

AI features (async, best-effort):
execute_submission task → score_rubric.delay()    ← Celery queues these after result written
                       → generate_hint.delay()    ← only on FAILED
Phase 2 plagiarism   → explain_plagiarism.delay() ← after flag created
Trainer dashboard    → generate_exam_summary()    ← on demand, cached
Trainer action       → generate_exam_draft()      ← on demand, not cached
```

If OpenAI is down: exams run normally. Students get their test results. Rubric and hints
eventually appear when service recovers (Celery retry). Plagiarism flags are visible without
explanations. Exam drafts return 503.

---

## Architecture Migration: Redis + Celery + Docker → SQS + Lambda

The original architecture used Celery workers pulling from Redis and Docker containers for code isolation. The migration to Lambda + SQS was a deliberate trade — not an upgrade in all dimensions. Each row below is a talking point.

| Dimension | Redis + Celery + Docker | SQS + Lambda | Winner |
|---|---|---|---|
| **Cost at low traffic** | Redis + workers running 24/7, paying for idle | Pay per invocation — zero idle cost | ▶ Lambda |
| **Cost at peak** | Pre-provisioned workers, fixed capacity | Scales elastically, per-invocation cost adds up | Draw |
| **Burst scaling** | Manual — provision more Celery workers or EC2 | Automatic — Lambda scales to account concurrency limit | ▶ Lambda |
| **Execution isolation** | Docker container, shared host kernel | Fully ephemeral per invocation, no shared state | ▶ Lambda |
| **Ops overhead** | Manage Redis, Celery, Docker daemon, worker fleet | SQS + Lambda fully managed, zero infra to operate | ▶ Lambda |
| **Result delivery** | Redis pub/sub → true push, near-instant, zero polling | Postgres poll — client polls every 1.5s, up to 1.5s added latency | ▶ Redis/Celery |
| **Cold start latency** | Always warm (persistent workers) | 1–3s cold start (Provisioned Concurrency mitigates) | ▶ Redis/Celery |
| **Max execution time** | Configurable, no hard ceiling | 15-minute hard AWS cap (set to 15s for sandbox) | ▶ Docker |
| **Real-time feel** | Truly event-driven end-to-end (Redis pub/sub) | Event-driven execution, polling last mile | ▶ Redis/Celery |
| **Exactly-once delivery** | `acks_late=True` + Redis visibility timeout — correct but requires careful config | SQS FIFO `MessageDeduplicationId` — built-in, 5-minute window, simpler | ▶ Lambda |
| **Observability** | Celery Flower, custom Redis metrics, Docker stats | CloudWatch Logs per invocation, X-Ray, built-in SQS depth metrics | ▶ Lambda |

> **Migration rationale:** We traded true event-driven push and always-warm workers for pay-per-use cost, automatic burst scaling, and per-invocation sandbox isolation. The cost is up to 1.5s of polling latency on result delivery — acceptable for an exam platform, not acceptable for a live coding game. Lambda's ephemeral model means one student's rogue process literally cannot affect another's.

---

## Why Client Polling — SSE vs Client Polling After the Lambda Migration

SSE was the right last-mile mechanism when result delivery was Redis pub/sub. After migrating to Lambda (which writes to Postgres and exits), SSE lost its push advantage entirely. This table shows why client polling was the right call.

| Aspect | SSE (removed) | Client Polling (current) |
|---|---|---|
| **HTTP overhead** | 1 persistent connection per user held open on a Gunicorn worker | 1 new stateless request per poll tick (~1.5s) |
| **Result latency** | Same — both wait on a Postgres poll | Same — both wait on a Postgres poll |
| **Server impact** | Django holds connection open; worker blocked for the duration | Stateless — worker free after each response |
| **Proxy / infra** | Sometimes breaks — buffering proxies drop SSE events; requires `X-Accel-Buffering: no` + `proxy_read_timeout 300s` | Always works — standard HTTP request-response, no special Nginx config |
| **Works serverless** | No — needs persistent Django; Lambda cannot hold an SSE connection open | Yes — any server, including Lambda |
| **Architectural honesty** | "Polling theater" — SSE framing without true push underneath | Clean match to what is actually happening |
| **Postgres reads at 10K users** | 20,000 reads/sec (server polls every 500ms on behalf of each client) | ~6,700 reads/sec (client polls every 1.5s, stateless) |

> **Key insight:** After the Lambda migration, SSE was doing the same work as client polling — just with a persistent HTTP connection on top. Removing it made the server stateless, dropped Postgres read load by ~67%, and eliminated all the proxy configuration. If true push is needed at scale, the correct investment is API Gateway WebSocket where Lambda explicitly notifies a connection manager — not a polling loop dressed up as streaming.

---

## Key Engineering Decisions

| Decision | Chosen | Alternatives considered | Why |
|---|---|---|---|
| Execution sandbox | AWS Lambda | Docker on EC2, Firecracker, gVisor | Lambda gives per-invocation isolation with zero host management; Docker on EC2 requires fleet management |
| Queue | SQS FIFO | SQS Standard, Redis pub/sub, Kafka | FIFO gives built-in deduplication; Kafka is overkill for this event shape |
| Result delivery | Client polling every 1.5s | SSE (removed), WebSocket push | After Lambda migration, SSE was polling theater anyway. Client polling is honest, simpler, and stateless for the server. |
| Attempt gate | SELECT FOR UPDATE (Postgres) | Application-level lock, Redis INCR | Postgres transaction gives durability and atomicity in one operation |
| Plagiarism trigger | Django pre_save signal | Cron job, admin button | Signal fires automatically on state transition; no human intervention or scheduling required |
| Plagiarism scope | K×N (behavioral pre-filter) | N² exhaustive pairwise | N² = 2 billion comparisons at 10K students; behavioral filter drops K to ~5–10%, 90% compute reduction |
| Code normalization | AST rename (Python) + regex (JS) | Raw text | Defeats trivial renaming tricks; catches students who just changed variable names |
| Plagiarism weights | Configurable via PlagiarismPolicy model (Redis-cached) | Hardcoded constants | Allows live weight tuning without a deploy; A/B testable |
| AI features | Async Celery tasks + polling endpoints | Synchronous in submission response, streaming | Exam SLA cannot depend on LLM availability; async makes AI optional without blocking the critical path |
| Trainer dashboard | Single fetch on mount + Redis cache TTL=10s | Constant polling | Dashboard data doesn't change per-second; single fetch is sufficient |

---

## FAQ — Interview Questions

**Q: Why did you remove SSE?**

After the SQS + Lambda migration, SSE lost its core advantage. In the original architecture,
Lambda published to a Redis pub/sub channel and the Django SSE handler was subscribed — true
server push with zero polling. After migrating to Lambda writing results to Postgres, the SSE
handler had no event to subscribe to. It had to poll Postgres every 500ms on the client's behalf.

A server polling Postgres every 500ms and forwarding via SSE is doing exactly the same work as
the client polling a REST endpoint every 1.5s — but with the added overhead of a persistent HTTP
connection on a Gunicorn worker and Nginx proxy timeout configuration. We removed SSE entirely.
Client polling is the honest mechanism. If we need true push in the future, the right investment
is API Gateway WebSocket where Lambda explicitly notifies the connection manager after writing.

---

**Q: Why did you choose Lambda over Docker on a persistent host?**

Two reasons: isolation and operations. Docker containers on the same host share the kernel.
A misbehaving student program triggering a kernel-level issue could affect other containers on
that host. Lambda invocations are fully isolated — one invocation panicking affects only itself.

Operationally, Docker on EC2 means sizing a fleet for peak load (10K concurrent exams), paying
for that capacity 24/7, and managing patching, Docker daemon health, and Celery worker supervision.
Lambda eliminates all of that. The execution environment is managed by AWS. The trade-off is
cold starts — mitigated with Provisioned Concurrency sized to the registered student count before
each exam.

---

**Q: Walk me through SELECT FOR UPDATE — what race condition does it prevent?**

Without the lock: two browser tabs submit simultaneously for the same student and question. Both
read `attempt_count=2` (limit is 3), both see 2 < 3, both increment. But because they read
before either commits, both read 2 and both write 3. Student consumes one attempt but two
executions get through. The counter correctly shows 3 but two Lambda invocations run.

`SELECT FOR UPDATE` makes the read-increment-write atomic. The first transaction holds the row
lock while it reads, increments, and commits. The second blocks on `SELECT FOR UPDATE` until
the first commits. It then reads the already-incremented value (3 = limit), rejects the
submission, and rolls back. Classic lost-update problem. Postgres row-level lock is the correct
tool.

---

**Q: Why behavioral signals before TF-IDF similarity? Why not run similarity first since it's more definitive?**

Because similarity is O(N²) in the exhaustive case, and behavioral is O(N). At N=10,000 students
and 20 questions, exhaustive pairwise TF-IDF is 2 billion comparisons — minutes of compute, which
is unacceptable for post-exam processing.

Behavioral scoring reads each student's event log once and computes a weighted sum — runs in
seconds at N=10,000. It generates the suspect set K (typically 5–10% of the cohort). Phase 2
then runs O(K×N) similarity — K=500 students versus N=10,000 full cohort = 5 million comparisons
per question. Seconds, not minutes.

The behavioral signals are also reliable enough as a coarse filter. A student with paste_ratio=0.9
and who submitted in 30 seconds on a hard problem has a high prior probability of cheating.
Similarity confirms it and identifies the source; behavioral doesn't need to be the final verdict.

---

**Q: How does SQS FIFO deduplication work and why does it matter here?**

Every `send_message` call includes a `MessageDeduplicationId` — we use the submission UUID
generated when we insert the Postgres row. SQS maintains a deduplication journal for 5 minutes.
If a second message arrives with the same ID within that window, SQS discards it and returns
success to the caller.

Why this matters: HTTP is unreliable. Django's `send_message` call might time out from Django's
perspective — the request got through, SQS accepted it, but the TCP response was lost. Django's
retry would send the message again. Without deduplication, SQS enqueues it twice, Lambda runs
twice, two result rows appear. With FIFO deduplication, the retry is silently dropped. Lambda
runs exactly once. The submission UUID is the right key because it was generated before calling
SQS — stable, unique per submission.

---

**Q: If GPT is down, what happens to students?**

Nothing affecting exam delivery. The submission critical path is:
`POST /submit → SQS → Lambda → Postgres → GET /result poll`
No LLM anywhere in this chain.

Rubric scoring and hint generation are Celery tasks queued after Lambda writes the result. They
will retry when OpenAI recovers — students will eventually see rubric scores and hints. Plagiarism
explanations are similarly queued post-exam. Exam draft generation returns 503 immediately.

An exam with delayed rubric scores is far better than an exam that cannot accept submissions.

---

**Q: How would you scale to 100K concurrent users? What breaks first?**

Three bottlenecks in order:

1. **Postgres write load from Lambda.** 100K submissions burst = 100K concurrent `UPDATE`
   statements hitting Postgres simultaneously. Read replicas don't help here (these are writes).
   Fix: Postgres connection pooler (PgBouncer) in transaction-pooling mode; consider write-ahead
   batching or a lower-latency result store (DynamoDB) for the execution result path.

2. **SQS FIFO throughput ceiling.** A single FIFO queue supports ~3,000 msg/sec per message
   group. With `MessageGroupId=exam_id`, a single massive exam at 10K submissions/sec needs
   message group sharding: `exam_id + shard_id` where `shard_id = student_id % N`. Sacrifices
   strict per-exam ordering; per-student ordering within a shard is maintained.

3. **Lambda concurrency limit.** AWS default is 1,000 per region. At 100K concurrent students,
   need a service limit increase to 100K+. Pre-exam operational checklist item, not an architecture
   change, but requires lead time.

Client polling at 100K users = ~67K Postgres reads/sec (100K ÷ 1.5s interval). Manageable
with PgBouncer + read replicas serving the `GET /submissions/{id}/` endpoint, since these are
primary-key reads on a cached hot path.

---

**Q: Walk me through the plagiarism pipeline end-to-end.**

1. Trainer closes the exam. `Exam.is_active` transitions True → False.
2. Django `pre_save` signal fires synchronously. `run_exam_plagiarism_check.delay(exam_id)` is queued.
3. Celery picks up the task. Phase 1 runs: for each student, compute 4 behavioral signals,
   weighted sum, store `BehaviouralFlag` if score > 0.2.
4. Phase 1 finishes. One `run_question_plagiarism_check` task queued per question, staggered
   30s apart.
5. Each question task: fetch suspects (behaviorally flagged students), fetch all cohort submissions,
   normalize code (AST rename for Python), build TF-IDF matrix, compute cosine similarity for
   each suspect against the full cohort, flag pairs above threshold.
6. For each new `PlagiarismFlag`: queue `explain_plagiarism.delay(flag_id)`.
7. GPT-4o-mini writes the prose explanation ("evidence not verdict"). Stored in `PlagiarismExplanation`.
8. Trainer views flagged pairs in the dashboard. For each: similarity score, behavioral signals,
   AI-generated explanation of structural patterns.

---

## Interview Bridges — Real-World Analogues

**SQS + Lambda → YouTube video processing**
Upload → queue → isolated transcoding worker → result stored → user polls or gets notification.
Same decoupling pattern. Different timescale (minutes for video vs seconds for code execution).

**SELECT FOR UPDATE → BookMyShow seat reservation**
Two users click "Book" on the last seat simultaneously. Row lock on the seat ensures only one
succeeds. CodeMas uses the same lock for the attempt counter.

**Client polling → Any mobile app checking order status**
Swiggy, Zomato, Amazon — "your order is being prepared" screens poll a status endpoint every
few seconds. No SSE, no WebSocket. Simple, stateless, scales horizontally.

**Plagiarism O(K×N) → Large-scale dedup pipelines**
Spotify song dedup, Dropbox block dedup, Stack Overflow near-duplicate detection — all use a
cheap filter first to reduce the candidate set, then expensive similarity on candidates only.

**Behavioral signals first → Razorpay/PayPal fraud detection**
Payment processors flag transactions using behavioral signals (typing speed, device fingerprint,
session duration) before running expensive ML fraud models. Behavioral acts as fast coarse filter;
ML confirms. Same pattern.

**AST-based code normalization → Compiler intermediate representation**
Before running any analysis, compilers reduce source code to a normalized form (AST or IR) that
is independent of variable naming or formatting. CodeMas applies the same principle for fair
plagiarism comparison.

---

## What-If Scenarios

**"CodeMas is now a SaaS for 500 schools. What changes?"**

Multi-tenancy becomes the dominant concern. Changes required:

- **Data isolation:** Add `school_id` to all tables. Postgres Row-Level Security: `CREATE POLICY
  school_isolation ON submissions USING (school_id = current_setting('app.school_id'))`.
  Set as a Postgres session variable at the start of each request.
- **JWT claims:** Include `school_id` in JWT payload. Every submission endpoint validates
  that the student, exam, and question all belong to the same school.
- **SQS:** One queue per school for premium tenants (cleaner isolation, separate limits);
  shared queue with `MessageGroupId = school_id + exam_id` for standard tier.
- **Lambda:** Stateless — no changes needed. Add `school_id` tagging to CloudWatch Logs
  for per-tenant observability and billing.
- **Usage metering:** Track submissions per school in a ledger table. Hook Lambda writes
  into a usage event stream.

**"Scale to 100K concurrent users. What breaks first?"**

See FAQ entry above. Summary: Postgres write load from Lambda hits first (100K concurrent UPDATEs),
followed by SQS FIFO throughput ceiling (needs message group sharding), followed by Lambda
concurrency limit (account limit increase required).

**"Add Python, Java, C++, Go, and Rust. What changes in Lambda?"**

Recommended: container-based Lambda with all runtimes pre-installed (build a Docker image with
Python, JRE, GCC, Go compiler, Rust compiler; publish to ECR; use as Lambda image).

Submission message already has a `language` field. Lambda handler gets a dispatch table:
```python
RUNNERS = {
    'python': ('python', '/tmp/solution.py'),
    'java':   ('javac /tmp/Solution.java && java Solution', None),
    'cpp':    ('g++ -o /tmp/solution /tmp/solution.cpp && /tmp/solution', None),
}
compile_cmd, run_cmd = RUNNERS[language]
```

Compiled languages add 1–5s of compilation latency. Set per-language Lambda timeout accordingly.
Java JVM needs more memory (512MB vs 256MB). Everything else — SQS, Postgres, client polling —
unchanged.

---

## What I'd Do Differently

1. **Provisioned Concurrency as an exam-creation step.** First time a large exam hits the
   default 1,000-invocation concurrent limit, results visibly queue up. Fix: exam-creation
   workflow that sets reserved concurrency on the Lambda function proportional to registered
   student count.

2. **WebSocket push for result delivery.** Client polling is correct and simple, but at 100K
   users the read load on Postgres is still significant. The correct long-term investment is
   API Gateway WebSocket where Lambda explicitly notifies the connection manager after writing —
   zero polling, true event-driven.

3. **Move the attempt gate to optimistic concurrency.** The current `SELECT FOR UPDATE` approach
   serializes all submissions from the same student. An optimistic model — INSERT attempt, check
   count afterward, compensate if over limit — avoids the lock and scales better. The submission
   UUID already provides idempotency for SQS; the same key could make the INSERT idempotent at
   the Postgres level.

4. **Store AI prompts as versioned database records.** Today the five AI feature prompts are
   f-strings in view/task code. Changing a prompt requires a code deploy. Storing prompts as
   versioned records allows prompt iteration without deployment and enables shadow-testing a
   new prompt version against the current one.

5. **Plagiarism progress visibility for trainers.** The pipeline runs silently. Trainers have
   no visibility into whether it started, how far along it is, or whether it failed. A simple
   `PlagiarismRun` status table with progress percentage, polled by the dashboard, would make
   post-exam UX significantly clearer.

---

## Technology Comparisons

### SQS FIFO vs Kafka vs RabbitMQ

CodeMas uses AWS SQS FIFO as the queue between submission API and Lambda executor. The three main options:

| Dimension | SQS FIFO | Kafka | RabbitMQ |
|---|---|---|---|
| Delivery guarantee | Exactly-once per message group | At-least-once (idempotent consumers needed) | At-least-once (manual ack) |
| Ordering | Per message group | Per partition | FIFO per queue |
| Throughput | 3,000 msg/s per group | Millions/s | High, cluster-dependent |
| Replay / rewind | No | Yes (configurable retention) | No |
| Multiple consumers | One consumer per queue | Many independent consumer groups | Flexible routing via exchanges |
| Ops burden | Zero — fully managed | High — cluster, ZooKeeper/KRaft, replication | Medium — cluster, policies |
| Cost model | Per request | Infrastructure + egress | Infrastructure |

**When Kafka wins:** You need event replay (re-process old submissions for audit), multiple independent consumers reading the same event stream, or throughput beyond SQS limits.

**When RabbitMQ wins:** You need complex routing — e.g., route to different worker pools based on submission language or question type. Exchanges and bindings handle this natively.

**For CodeMas:** SQS FIFO is the right call. Each submission maps to exactly one Lambda invocation — no replay needed, no fan-out, no routing. The FIFO guarantee prevents double execution if the API retries. The managed nature means no cluster to run at 3am when an exam is live. The burst at exam-close (students rushing to submit in the final minutes) is absorbed automatically by SQS without pre-provisioning.

**Interview move:** "Why not Kafka?" — Because Kafka's value (replay, multiple consumer groups, millions-per-second) is irrelevant here. SQS FIFO is simpler, managed, and exactly as powerful as the problem requires. Over-engineering the queue is one of the most common architecture mistakes.

---

### Lambda vs Celery + Docker vs Fargate

The code sandbox — the component that actually executes student-submitted code — is one of the most important design decisions in CodeMas.

| Dimension | AWS Lambda | Celery + Docker | AWS Fargate |
|---|---|---|---|
| Isolation unit | Per-invocation ephemeral sandbox | Container (may share host kernel) | Task-level container |
| Cold start | <100ms (warm), ~500ms (cold) | Instant (persistent workers) | 10–30s (task startup) |
| Resource limits | Hard: 10GB RAM, 15min max, configurable CPU | Soft: configurable, but requires explicit enforcement | Hard: task-level limits |
| Scaling | Instant, per-invocation auto-scale | Manual capacity planning or Kubernetes | Task-level, slower than Lambda |
| Billing | Per 1ms of execution | Workers run 24/7, idle cost | Per vCPU-second of task |
| Ops burden | Zero | Medium — worker fleet, broker, monitoring | Medium — task definitions, cluster |
| Code isolation strength | Strong (ephemeral Lambda runtime per invocation) | Container-level (host kernel shared) | Strong (container-level) |

**When Celery + Docker wins:** Long-running tasks (>15 min), GPU access needed, persistent state between invocations, or complex multi-step pipelines that benefit from staying in one process.

**When Fargate wins:** You need more control than Lambda allows (longer runtime, more memory, specific OS), but still want managed container scheduling without running an EC2 fleet.

**For CodeMas:** Lambda is the right sandbox. Each submission is a short-lived isolated execution (seconds, not minutes). Hard resource limits prevent runaway student code from consuming more than its share. The per-invocation billing model is perfect for burst-heavy exam traffic — you pay exactly for what runs, not for idle workers between exams. The Lambda runtime is discarded after each invocation, so student A's file system cannot leak to student B's execution.

**Interview move:** "Why not containers?" — Security isolation for arbitrary code execution is stronger with Lambda's ephemeral-per-invocation model than with a persistent Docker worker, because persistent workers accumulate state. Lambda also eliminates capacity planning — Lambda scales from 0 to 10,000 concurrent executions automatically.

---

## Technical Dictionary

*Plain-English definitions of every term, algorithm, and tool used in this document. If something above confused you, start here.*

---

### Queueing & Messaging

### SQS FIFO
AWS Simple Queue Service is Amazon's hosted message queue — a buffer that lets one system drop work in and another system pick it up, without either side needing to know about the other. The FIFO (First In, First Out) variant guarantees that messages are delivered in the exact order they arrived within each group, and that each message is processed exactly once within a five-minute deduplication window.

**Example:** When 500 students submit code in the last minute of an exam, their submissions land in the SQS FIFO queue in arrival order, and each one triggers exactly one Lambda invocation — no duplicates, no skipped submissions.

---

### MessageGroupId
A label you attach to each SQS FIFO message that defines its ordering lane. Messages with the same `MessageGroupId` are processed in strict order relative to each other, while messages with different group IDs can be processed in parallel — giving you both ordering within a group and throughput across groups.

**Example:** CodeMas sets `MessageGroupId=exam_id` so submissions within the same exam are processed in order, while two different exams running simultaneously each get their own lane and don't block each other.

---

### MessageDeduplicationId
A unique ID you attach to an SQS FIFO message so the queue can recognize and silently discard any duplicate. If the same ID is sent twice within five minutes, SQS accepts the second call but never delivers the second message to a consumer.

**Example:** CodeMas uses the submission's UUID (generated when the Postgres row is inserted) as the `MessageDeduplicationId` — so if Django's `send_message` call times out and the client retries, SQS drops the duplicate and Lambda runs exactly once.

---

### SQS Standard Queue
The simpler, higher-throughput variant of SQS that does not guarantee ordering and may deliver the same message more than once ("at-least-once delivery"). It supports essentially unlimited throughput but requires consumers to handle duplicates themselves.

**Example:** A notification system that emails students their results could use a Standard Queue — a duplicate email is annoying but harmless, and the higher throughput ceiling handles bursts more easily than FIFO.

---

### Dead Letter Queue (DLQ)
A separate queue where SQS automatically moves messages that have failed processing a configurable number of times. Instead of silently dropping bad messages, the DLQ preserves them for inspection and debugging.

**Example:** If a Lambda invocation crashes five times on the same malformed submission, SQS routes that submission to the DLQ so an engineer can inspect the bad payload rather than losing it forever.

---

### Compute & Execution

### AWS Lambda
A "serverless" compute service where you upload code and AWS runs it in response to events, without you managing any servers. Each invocation gets its own isolated environment, runs for the duration of the function, and then disappears. You pay only for the time your code actually runs.

**Example:** Each student's code submission triggers one Lambda invocation that executes the student's code, checks the output against test cases, and writes the result to Postgres — all without CodeMas running a persistent server to do this work.

---

### Cold start
The extra delay that occurs when Lambda has to provision a brand-new container for an invocation because no warm container is available. This one-time setup — downloading the runtime, loading your code, initializing globals — typically adds one to three seconds before your function code even begins running.

**Example:** If an exam starts and 5,000 students submit in the first seconds, many Lambda invocations will cold-start simultaneously, adding up to three seconds of latency to those first results.

---

### Provisioned Concurrency
A Lambda setting that keeps a specified number of containers pre-initialized and ready to handle invocations immediately, eliminating cold starts for that many concurrent requests. You pay for these warm containers whether or not they are actively processing a request.

**Example:** Before a 10,000-student exam, CodeMas sets Provisioned Concurrency to match the registered student count so the first wave of submissions hits warm containers and results arrive within the 10-second SLA.

---

### Ephemeral container
"Ephemeral" means temporary and non-persistent. Each Lambda invocation runs in an isolated container whose `/tmp` directory is wiped when the invocation ends, and whose state is never visible to any other invocation — there is no shared memory, shared filesystem, or shared process space between two Lambda calls.

**Example:** Student A's code writes a file to `/tmp/output.txt` during their Lambda invocation; when that invocation ends, the file is gone and Student B's invocation starts with a completely clean `/tmp`.

---

### VPC (Virtual Private Cloud)
A logically isolated section of the AWS network that you control — your own private slice of AWS where you define subnets, routing rules, and firewall-like security groups. Resources inside a VPC can be completely cut off from the public internet if you choose.

**Example:** CodeMas runs the Lambda sandbox inside a VPC with no internet gateway attached, so student code physically cannot make outbound HTTP calls to external services or leak answers to collaborators.

---

### Database & Concurrency

### SELECT FOR UPDATE
A Postgres SQL clause that, when added to a `SELECT` query inside a transaction, places an exclusive row-level lock on the rows it reads. Any other transaction trying to read those same rows with `FOR UPDATE` will block and wait until the first transaction commits or rolls back.

**Example:** When two browser tabs submit simultaneously for the same student, the first tab's transaction locks the `attempt_counters` row, increments it, and commits; the second tab's transaction then unblocks, reads the already-incremented count, and correctly rejects the submission.

---

### Lost-update problem
A classic database race condition where two transactions read the same value, both compute a new value based on what they read, and both write their result — so one write overwrites the other, as if the first transaction never happened. The "lost update" is the write that got silently discarded.

**Example:** Without `SELECT FOR UPDATE`, two simultaneous submissions both read `attempt_count=2`, both compute `2 < 3` (allowed), both write `3` — the student used one attempt slot but two Lambda invocations ran.

---

### PgBouncer
A lightweight Postgres connection pooler that sits between your application and Postgres. Because Postgres creates an OS-level process for each connection, raw connections exhaust server resources quickly at scale. PgBouncer maintains a small pool of real Postgres connections and multiplexes many application connections through them.

**Example:** With 10,000 Lambda invocations each trying to open their own Postgres connection simultaneously, Postgres would crash; PgBouncer absorbs all those connection attempts and serves them through a pool of, say, 100 real connections.

---

### Transaction-pooling mode
PgBouncer's most aggressive pooling strategy: a real Postgres backend connection is held only for the duration of a single transaction, then immediately returned to the pool for another client to reuse. This is more efficient than session-pooling (where a connection is held for the entire client session) but requires that applications not use session-level state like temporary tables.

**Example:** In transaction-pooling mode, 10,000 short Lambda transactions share 100 real Postgres connections without any Lambda invocation ever waiting long for a connection slot.

---

### JSONB
Postgres's binary-encoded JSON column type. Unlike plain `JSON`, JSONB decomposes the JSON into a binary format that supports indexing and fast key-lookup queries — you can query inside the JSON without parsing the whole document every time.

**Example:** The `result` column stores `{ verdict, stdout, stderr, execution_time_ms }` as JSONB so the Django API can return the full result object in one column read, and future queries like "find all submissions where verdict='PASS'" can be indexed efficiently.

---

### Algorithms & ML

### TF-IDF (Term Frequency-Inverse Document Frequency)
A method for turning text (or code) into a numerical vector that captures how important each word is within a document relative to a collection of documents. Words that appear often in one document but rarely across all documents get high weight — they are distinctive. Common words that appear everywhere get low weight — they are uninformative.

**Example:** The token `bubble_sort` appears in 3 of 10,000 submissions — it gets high weight in those three vectors. The token `return` appears in every submission — it gets near-zero weight everywhere, so it doesn't make any pair look more similar than they are.

---

### Cosine similarity
A way to measure how similar two TF-IDF vectors are by computing the cosine of the angle between them. The result is always between 0 and 1: a score of 1 means the two documents are identical in word distribution; 0 means they share no terms at all.

**Example:** Two students' submissions that were clearly copied from the same source produce a cosine similarity of 0.94 — flagged well above the 0.80 threshold — while two different correct solutions to the same problem produce 0.41.

---

### AST (Abstract Syntax Tree)
A tree-shaped data structure that represents the structure of source code after parsing, stripped of all formatting and names. Each node is a code construct (function, loop, assignment) rather than raw text. Because variable names are replaced with the tree's structural positions, an AST captures the logic of code independently of how it was named.

**Example:** Two submissions where one uses `x, y, z` and the other uses `a, b, c` as variable names look identical after AST normalization — both become `VAR_0, VAR_1, VAR_2` — so TF-IDF correctly sees them as the same code rather than different.

---

### O(N²) vs O(K×N)
Big-O notation describes how the number of operations grows as input size N grows. O(N²) means every pair of N items is compared — if N doubles, work quadruples. O(K×N) means K items are each compared against all N — if K is small (5–10% of N), this is roughly linear even as N grows large.

**Example:** At N=10,000 students, O(N²) exhaustive pairwise comparison = 50 million pairs per question; with behavioral pre-filtering reducing suspects to K=500, O(K×N) = 5 million comparisons per question — a 10× reduction in compute that cuts processing from minutes to seconds.

---

### Real-time Delivery

### SSE (Server-Sent Events)
A browser standard for one-directional streaming over a single long-lived HTTP connection: the server pushes events to the client whenever it has something new to say, without the client asking. The connection is opened once and stays open until the server or client closes it.

**Example:** In the original CodeMas architecture, the Django server held an SSE connection open per student and pushed the result the moment Redis pub/sub delivered it from the Celery worker — true server push with no polling. This was removed after the Lambda migration because Lambda cannot hold an SSE connection open.

---

### Client polling
A pattern where the frontend repeatedly sends a normal HTTP request to a status endpoint on a fixed interval, checks the response, and either renders the result or schedules another request. The server is stateless — it simply returns whatever is in the database at that moment.

**Example:** After receiving a 201 from `POST /submit`, the Vue frontend calls `GET /submissions/{id}/` every 1.5 seconds until the status field is no longer `PENDING`, then renders the verdict.

---

### WebSocket
A protocol that upgrades a standard HTTP connection into a full-duplex, bidirectional channel — either the client or the server can send a message at any time without the other side having to request it first. Unlike SSE, the server can initiate messages and the client can respond on the same connection.

**Example:** An ideal future version of CodeMas would use API Gateway WebSocket: when Lambda finishes writing the result to Postgres, it sends a push notification directly to the student's open WebSocket connection — zero polling, true real-time.

---

### Redis pub/sub
A publish/subscribe messaging pattern built into Redis. A publisher sends a message to a named channel; every subscriber listening on that channel receives the message immediately. It is not a durable queue — messages are delivered only to currently connected subscribers and are not stored.

**Example:** In the original Celery-based architecture, the Celery worker published the execution result to a Redis channel named after the submission ID, and the Django SSE handler (subscribed to that channel) immediately forwarded it to the student's browser — true server push with zero database polling.

---

### Auth & Web

### JWT (JSON Web Token)
A compact, self-contained token used for authentication. It has three base64-encoded parts separated by dots: a header (algorithm used), a payload (claims like user ID and expiry), and a cryptographic signature. The server can verify the signature without hitting a database, making JWT validation fast and stateless.

**Example:** When a student logs in, Django issues a JWT signed with the server's secret key; on every subsequent request, Simple JWT verifies the signature in microseconds without a database lookup, then extracts the student ID from the payload.

---

### Simple JWT
A third-party Django REST Framework library that handles the full JWT lifecycle: generating access and refresh tokens on login, validating tokens on each request, and handling token expiry and rotation. It plugs into DRF's authentication system as a drop-in.

**Example:** CodeMas uses Simple JWT so that the `POST /api/submissions/create/` endpoint automatically rejects requests with expired or tampered tokens before the view code even runs.

---

### DRF (Django REST Framework)
A toolkit built on top of Django that makes building JSON APIs faster and more consistent. It provides serializers (for converting model instances to/from JSON), class-based views (for common CRUD patterns), authentication backends, and permission classes — all wired together by convention.

**Example:** The `SubmissionCreateView` in CodeMas is a DRF `CreateAPIView` that handles request parsing, JWT auth, and serialization in a few dozen lines instead of hundreds.

---

### Gunicorn
A production-grade WSGI server for Python web applications. It runs multiple worker processes (or threads/coroutines) that each handle incoming HTTP requests, sitting behind Nginx which handles TLS termination and static file serving.

**Example:** CodeMas runs Gunicorn with gevent workers so that each worker process can handle thousands of concurrent polling connections without blocking — critical when 10,000 students are polling for results simultaneously.

---

### gevent
A Python concurrency library that uses "greenlets" — lightweight cooperative threads that yield control to each other during I/O waits. A single OS thread can run thousands of greenlets because they switch whenever they would otherwise block on network or disk I/O, not on a time slice.

**Example:** With gevent workers, one Gunicorn worker process handles thousands of concurrent `GET /submissions/{id}/` poll requests because each greenlet yields during the Postgres read, letting others run — instead of blocking an entire OS thread per request.

---

### AI Features

### GPT-4o-mini
OpenAI's smaller, faster, cheaper variant of the GPT-4o model family. "Mini" means it is distilled to a fraction of the size of the full model — lower cost per token and lower latency, at some quality trade-off. It is well-suited for structured-output tasks where speed and cost matter more than maximum reasoning depth.

**Example:** CodeMas uses GPT-4o-mini for all five AI features because rubric scoring and hint generation are latency-sensitive best-effort tasks, not reasoning-heavy research — mini's speed and cost profile fits perfectly.

---

### Socratic hint
A teaching method where instead of giving the student the answer, you ask a guiding question that leads them to discover it themselves. Named after the Greek philosopher Socrates, who taught entirely through questioning. In AI prompting, it means constraining the model to questions and nudges, not code or solutions.

**Example:** When a student's solution fails because they forgot to handle an empty list, the Socratic hint says "What happens to your loop when the input has zero elements?" rather than "Add `if not lst: return 0` at the top."

---

### Celery
A distributed task queue for Python. Your application pushes a task (a Python function call) onto a broker (Redis or RabbitMQ); a pool of Celery worker processes picks tasks off the broker and executes them, independently of the web request that queued them. This decouples slow or non-critical work from the HTTP response cycle.

**Example:** After Lambda writes a submission result to Postgres, the Django API queues `score_rubric.delay(submission_id)` and `generate_hint.delay(submission_id)` via Celery — these run asynchronously in the background while the student's browser is already showing their test results.
