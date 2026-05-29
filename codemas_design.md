# CodeMas — System Design Knowledge Base

**Prepared for:** Interview with Drew Barclay at Entrupy  
**Role:** Senior Full-Stack Engineer  
**Author:** Swanand Kadam  
**Date:** May 2026

---

## Table of Contents

1. [What Is CodeMas](#1-what-is-codemas)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Tech Stack Decisions](#3-tech-stack-decisions)
4. [Data Models](#4-data-models)
5. [Submission Pipeline — The Core Flow](#5-submission-pipeline--the-core-flow)
6. [Execution Engine — Docker Sandbox](#6-execution-engine--docker-sandbox)
7. [Real-Time Result Delivery — SSE](#7-real-time-result-delivery--sse)
8. [Plagiarism Detection System](#8-plagiarism-detection-system)
   - 8.1 [Policy Versioning and Changelog](#81-policy-versioning-and-changelog)
   - 8.2 [Cohort Investigation and Cheating Ring Detection](#82-cohort-investigation-and-cheating-ring-detection)
9. [Code Similarity Search](#9-code-similarity-search)
10. [Auto-Exam Lifecycle — Celery Beat](#10-auto-exam-lifecycle--celery-beat)
11. [Trainer Dashboard and Monitoring](#11-trainer-dashboard-and-monitoring)
12. [Cost Analysis Engine](#12-cost-analysis-engine)
13. [Authentication and Authorization](#13-authentication-and-authorization)
14. [Concurrency and Scaling](#14-concurrency-and-scaling)
15. [Infrastructure Cost Comparison — EC2 vs Fargate vs Lambda](#15-infrastructure-cost-comparison--ec2-vs-fargate-vs-lambda)
16. [Entrupy Parallels](#16-entrupy-parallels)
17. [Known Gaps and Honest Limitations](#17-known-gaps-and-honest-limitations)
18. [What to Say to Drew — Quick Reference](#18-what-to-say-to-drew--quick-reference)

---

## 1. What Is CodeMas

CodeMas is a real-time coding exam platform built to demonstrate systems thinking for a Senior Full-Stack role. Students write code in a browser editor, submit it, and receive test-case results in under 10 seconds. Trainers monitor their cohort live, review plagiarism flags, and manage exam lifecycle.

### Primary User Flows

**Student Flow:**
1. Log in, see active exams
2. Enter exam — countdown timer starts
3. Write code in browser (CodeMirror editor)
4. Submit — receives pass/fail with per-test-case breakdown in <10 seconds
5. Up to 3 attempts per question (configurable)

**Trainer Flow:**
1. Build exam — add questions, test cases (hidden), set duration
2. Enroll students
3. Monitor live cohort dashboard during exam
4. After exam closes — review plagiarism flags (behavioural + similarity)
5. Review individual submission detail, annotate flags, confirm/dismiss

### Scale Target
10,000 concurrent students. This is the load number that informs every design decision.

---

## 2. High-Level Architecture

```
                           ┌─────────────────────────────────────────────┐
                           │              BROWSER (Vue 3 + Vite)          │
                           │                                               │
                           │  CodeMirror 6   Pinia Store   ECharts        │
                           │  (editor)       (state)       (charts)       │
                           │                                               │
                           │  EventSource ←────────────── SSE stream      │
                           │  Axios       ──────────────→ REST API        │
                           └────────────────┬────────────────┬────────────┘
                                            │ HTTP/REST       │ SSE (text/event-stream)
                                            ▼                 ▼
                           ┌─────────────────────────────────────────────┐
                           │            Django 4.2 + DRF + Gunicorn       │
                           │                                               │
                           │  /api/submissions/create/   POST             │
                           │  /api/submissions/{id}/stream/  GET (SSE)    │
                           │  /api/plagiarism/flags/     GET              │
                           │  /api/plagiarism/search/    POST             │
                           │  /api/dashboard/            GET              │
                           │  /api/cost/                 GET              │
                           │                                               │
                           │  Simple JWT auth   |   Role-based guards     │
                           └────────┬─────────────────────┬───────────────┘
                                    │                      │
                    ┌───────────────▼──┐      ┌──────────▼──────────────┐
                    │   PostgreSQL     │      │         Redis            │
                    │   (ACID writes)  │      │                          │
                    │                 │      │  Broker (Celery queue)   │
                    │  Users          │      │  Cache (10s TTL)         │
                    │  Exams          │      │  Pub/Sub (SSE delivery)  │
                    │  Questions      │      │  JWT blacklist           │
                    │  Submissions    │      └──────────┬───────────────┘
                    │  BehaviouralFlag│                 │
                    │  PlagiarismFlag │      ┌──────────▼───────────────┐
                    │  PlagiarismPolicy│     │    Celery Workers        │
                    └──────────────────┘    │                          │
                                            │  execute_submission      │
                                            │  run_exam_plagiarism_    │
                                            │    check (Phase 1+2)     │
                                            │  run_question_plagiarism_│
                                            │    check                 │
                                            │  auto_manage_exams       │
                                            │    (Celery Beat, 60s)    │
                                            └──────────┬───────────────┘
                                                       │
                                            ┌──────────▼───────────────┐
                                            │    Docker Engine         │
                                            │                          │
                                            │  python:3.11-alpine      │
                                            │  node:18-alpine          │
                                            │  openjdk:17-slim         │
                                            │  gcc:12                  │
                                            │                          │
                                            │  --memory=128m           │
                                            │  --cpus=0.5              │
                                            │  --network=none          │
                                            │  --pids-limit=10         │
                                            │  --read-only             │
                                            │  5s timeout kill         │
                                            └──────────────────────────┘
```

### Django App Structure

```
backend/
├── accounts/        # User model (Student / Trainer / Admin roles)
├── exams/           # Exam, Question, TestCase, ExamEnrollment models
│                    # + auto_manage_exams Celery Beat task
├── submissions/     # Submission model, create/list/detail views
│                    # + SSE streaming endpoint (sse.py)
├── execution/       # execute_submission task — runs Docker
│                    # (Lambda executor stub — commented out)
├── results/         # Result model — per-test-case outcome
├── plagiarism/      # BehaviouralFlag, PlagiarismFlag, PlagiarismPolicy
│                    # + all detection tasks + code similarity search
├── dashboard/       # Trainer KPI view, student detail, exam history
│                    # + CostAnalysisView (Fargate pricing model)
├── leaderboard/     # Student score/streak views
└── codemas/         # settings.py, urls.py, celery.py, signals.py
```

---

## 3. Tech Stack Decisions

### Decision Table

| Layer | Choice | Alternatives Considered | Reason for Choice |
|---|---|---|---|
| Frontend framework | Vue 3 + Vite | React, Angular | Composition API, fast HMR, familiar |
| Code editor | CodeMirror 6 | Monaco, Ace | Modular, tree-shakeable, great mobile support |
| State management | Pinia | Vuex, Redux | Lighter, TypeScript-native |
| Charts | ECharts | Chart.js, D3 | Large dataset performance (10k students) |
| Real-time delivery | SSE (EventSource) | WebSockets, polling | Server→client only; SSE is the right tool |
| Backend framework | Django 4.2 + DRF | FastAPI, Node.js | ORM, migrations, admin, batteries included |
| Authentication | Simple JWT | Session auth, OAuth | Stateless, works for mobile/SPA clients |
| Async jobs | Celery + Redis | AWS SQS + Lambda | Needs persistent process for pub/sub + Beat |
| Code execution | Docker | Lambda, vm2, Judge0 | See section 6 for full reasoning |
| Primary DB | PostgreSQL | MySQL, MongoDB | ACID transactions — critical for attempt counting |
| Unstructured data | MongoDB | PostgreSQL JSONB | Flexible schema for code blobs and logs |
| Cache + broker | Redis | Memcached, RabbitMQ | Pub/Sub capability for SSE delivery |
| Plagiarism scoring | TF-IDF + cosine | CodeBERT, MOSS, AST | Fast, no GPU, interpretable at 2k submissions |
| Deployment model | Docker on EC2 | AWS Fargate, Lambda | See section 15 for cost breakdown |

---

## 4. Data Models

### Core Models (PostgreSQL)

```
┌──────────────────────────────────────┐
│              User                    │
├──────────────────────────────────────┤
│ id (BigAutoField PK)                 │
│ username, email, password            │
│ role: student | trainer | admin      │
│ cohort: CharField(100)               │
│                                      │
│ @property score → points from        │
│   passed unique questions            │
│ @property current_streak → days      │
│   with consecutive submissions       │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│              Exam                    │
├──────────────────────────────────────┤
│ id, title, description               │
│ created_by → FK(User)                │
│ duration_minutes                     │
│ is_active: BooleanField              │
│ start_time, end_time: DateTimeField  │
│   (null = manual control)            │
│ created_at                           │
│                                      │
│ pre_save signal: when is_active      │
│ changes False → queue plagiarism     │
│ batch check                          │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│            Question                  │
├──────────────────────────────────────┤
│ id, exam → FK(Exam)                  │
│ title, description                   │
│ language: python|javascript|java|cpp │
│ difficulty: easy|medium|hard         │
│ tags: CharField (comma-separated)    │
│ max_attempts: PositiveIntegerField   │
│ @property points: easy=10, med=20,   │
│   hard=30                            │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│            TestCase                  │
├──────────────────────────────────────┤
│ question → FK(Question)              │
│ input_data: TextField                │
│ expected_output: TextField           │
│ is_hidden: BooleanField (default=True│
│   hidden = student never sees it)    │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│           Submission                 │
├──────────────────────────────────────┤
│ student → FK(User)                   │
│ exam → FK(Exam)                      │
│ question → FK(Question)              │
│ code: TextField                      │
│ language: CharField(20)              │
│ attempt_number: PositiveIntegerField │
│ status: pending|running|passed|      │
│         failed|error                 │
│ submitted_at: DateTimeField          │
│ time_remaining_seconds               │
│ execution_time_ms                    │
│ plagiarism_score                     │
│ ip_address: GenericIPAddressField    │
│                                      │
│ -- Behavioural signals (from browser)│
│ had_paste_event: BooleanField        │
│ paste_char_count: IntegerField       │
│ time_to_complete_seconds             │
│ tab_switch_count                     │
│                                      │
│ -- Processing status flags           │
│ behavioural_check_done: BooleanField │
│ similarity_check_done: BooleanField  │
│                                      │
│ -- Pre-computed for search           │
│ normalized_code: TextField           │
│   (stored at submission time after   │
│   execution completes)               │
│                                      │
│ @property percent_time_used          │
│   = (total - remaining) / total      │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│              Result                  │
├──────────────────────────────────────┤
│ submission → FK(Submission)          │
│ test_case → FK(TestCase)             │
│ passed: BooleanField                 │
│ actual_output: TextField             │
│ execution_time_ms                    │
│ error_message: TextField             │
└──────────────────────────────────────┘
```

### Plagiarism Models

```
┌──────────────────────────────────────┐
│         BehaviouralFlag              │
├──────────────────────────────────────┤
│ submission → OneToOneField           │
│ risk_score: FloatField (0.0–1.0)     │
│ confidence: low | medium | high      │
│   low = 1 signal, med = 2, high = 3+│
│                                      │
│ -- Individual signal scores (0–1.0)  │
│ paste_score                          │
│ time_score                           │
│ snapshot_score  (reused for tab)     │
│ submission_score                     │
│                                      │
│ signals_triggered: JSONField         │
│   human-readable list e.g.:          │
│   ["Pasted 68% of code",             │
│    "Solved hard in 2m 14s"]          │
│                                      │
│ review_status: pending|confirmed|    │
│               dismissed              │
│ review_note: TextField               │
│ reviewed_by → FK(User, null=True)    │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│          PlagiarismFlag              │
├──────────────────────────────────────┤
│ submission_1 → FK(Submission)        │
│ submission_2 → FK(Submission)        │
│ UNIQUE(submission_1, submission_2)   │
│                                      │
│ similarity_score: FloatField         │
│ flagged: BooleanField                │
│ detection_method: CharField          │
│   (auditable — methods evolve)       │
│                                      │
│ review_status: pending|confirmed|    │
│               dismissed              │
│ review_note: TextField               │
│ reviewed_by → FK(User, null=True)    │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│       PlagiarismPolicy (singleton)   │
├──────────────────────────────────────┤
│ paste_weight:       0.40             │
│ time_weight:        0.30             │
│ tab_weight:         0.15             │
│ submission_weight:  0.15             │
│ min_risk_score:     0.20             │
│ similarity_threshold: 0.80           │
│ easy_baseline_s:    90               │
│ medium_baseline_s:  240              │
│ hard_baseline_s:    480              │
│ updated_by, updated_at               │
│                                      │
│ @classmethod get() → get_or_create   │
│   pk=1 (singleton pattern)           │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│          PolicyVersion               │
├──────────────────────────────────────┤
│ changed_by → FK(User, null=True)     │
│ changed_at: DateTimeField(auto_now_  │
│   add=True)                          │
│ change_reason: TextField             │
│ snapshot_before: JSONField           │
│   {field: value, ...} for all 9     │
│   POLICY_FIELDS at time of change   │
│ snapshot_after: JSONField            │
│   full state after the save         │
│                                      │
│ ordering: ['-changed_at']            │
│                                      │
│ Active period of each version:       │
│   from changed_at until the next     │
│   newer version's changed_at         │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│          ReviewAuditLog              │
├──────────────────────────────────────┤
│ flag_type: behavioural | similarity  │
│ flag_id: IntegerField                │
│ action: confirmed | dismissed        │
│ note: TextField                      │
│ reviewed_by → FK(User)               │
│ created_at                           │
└──────────────────────────────────────┘
```

### ExamEnrollment

```
┌──────────────────────────────────────┐
│         ExamEnrollment               │
├──────────────────────────────────────┤
│ exam → FK(Exam)                      │
│ student → FK(User)                   │
│ enrolled_at                          │
│ UNIQUE(exam, student)                │
└──────────────────────────────────────┘
```

---

## 5. Submission Pipeline — The Core Flow

This is the end-to-end journey of a single code submission from browser to result.

```
BROWSER                     DJANGO                  REDIS               CELERY WORKER
  │                            │                      │                       │
  │  1. POST /submissions/     │                      │                       │
  │     create/                │                      │                       │
  │  {code, language,          │                      │                       │
  │   question, had_paste,     │                      │                       │
  │   paste_char_count,        │                      │                       │
  │   time_to_complete,        │                      │                       │
  │   tab_switch_count,        │                      │                       │
  │   time_remaining}          │                      │                       │
  │ ─────────────────────────► │                      │                       │
  │                            │                      │                       │
  │                            │ 2. Check attempts    │                       │
  │                            │    cache key:        │                       │
  │                            │    attempts:{uid}:   │                       │
  │                            │    {qid}             │                       │
  │                            │ ───────────────────► │                       │
  │                            │ ◄─────────────────── │                       │
  │                            │    returns 0-3       │                       │
  │                            │                      │                       │
  │                            │ 3. If >= max_attempts│                       │
  │                            │    → 403 immediately │                       │
  │                            │                      │                       │
  │                            │ 4. Create Submission │                       │
  │                            │    record (pending)  │                       │
  │                            │    in PostgreSQL     │                       │
  │                            │                      │                       │
  │                            │ 5. Increment attempt │                       │
  │                            │    counter in Redis  │                       │
  │                            │    TTL = 86400s      │                       │
  │                            │ ───────────────────► │                       │
  │                            │                      │                       │
  │                            │ 6. Queue execution   │                       │
  │                            │    task in Redis:    │                       │
  │                            │    execute_submission│                       │
  │                            │ ───────────────────► │                       │
  │                            │                      │ 7. Worker picks up    │
  │                            │                      │    execute_submission │
  │                            │                      │ ────────────────────► │
  │  8. Response: 201          │                      │                       │
  │     {id: submission_id}    │                      │                       │
  │ ◄───────────────────────── │                      │                       │
  │                            │                      │                       │
  │  9. Open SSE stream:       │                      │                       │
  │     GET /submissions/      │                      │                       │
  │     {id}/stream/           │                      │                       │
  │     ?token={jwt}           │                      │                       │
  │ ─────────────────────────► │                      │                       │
  │                            │ 10. Django subscribes│                       │
  │                            │     to Redis channel │                       │
  │                            │     submission:{id}  │                       │
  │                            │ ───────────────────► │                       │
  │                            │                      │                       │
  │  11. SSE: {status:waiting} │                      │                       │
  │ ◄───────────────────────── │                      │                       │
  │                            │                      │                       │
  │                            │                      │       12. Run Docker  │
  │                            │                      │           container   │
  │                            │                      │           per test    │
  │                            │                      │           case        │
  │                            │                      │                       │
  │                            │                      │       13. Save Result │
  │                            │                      │           rows to PG  │
  │                            │                      │                       │
  │                            │                      │       14. Update sub  │
  │                            │                      │           status,     │
  │                            │                      │           save        │
  │                            │                      │           normalized_ │
  │                            │                      │           code        │
  │                            │                      │                       │
  │                            │                      │ 15. Publish to        │
  │                            │                      │     submission:{id}   │
  │                            │                      │ ◄─────────────────── │
  │                            │ 16. Django receives  │                       │
  │                            │     message on       │                       │
  │                            │     subscribed chan  │                       │
  │                            │ ◄─────────────────── │                       │
  │                            │                      │                       │
  │  17. SSE: {status:passed}  │                      │                       │
  │ ◄───────────────────────── │                      │                       │
  │                            │                      │                       │
  │  18. Fetch full results:   │                      │                       │
  │     GET /results/          │                      │                       │
  │     ?submission={id}       │                      │                       │
  │ ─────────────────────────► │                      │                       │
  │ ◄───────────────────────── │                      │                       │
  │  per-test-case breakdown   │                      │                       │
```

### Attempt Limiting — Why Redis (not just PostgreSQL)

The attempt count is stored in Redis with `cache_key = f"attempts:{user.id}:{question_id}"`. This raises an important question: why not just count rows in the Submission table?

**The problem:** Two simultaneous requests both hit `SELECT COUNT(*)` — both see 2 attempts — both create a 3rd submission — student gets 4 attempts.

**Why PostgreSQL alone doesn't fully solve this without SELECT FOR UPDATE:** A naive ORM count is not atomic. You'd need a SELECT FOR UPDATE advisory lock, which adds latency and complexity.

**Why Redis works here:** Redis is single-threaded and all operations are atomic. `INCR` and `SET` with TTL are atomic by design. The cache is used as a fast counter; the actual Submission record is still written to PostgreSQL for persistence. On cache miss (node restart), the system falls back gracefully — the check is best-effort protection, not the only barrier.

**Why PostgreSQL still handles the canonical count:** `attempt_number` in the Submission model is the authoritative record. Redis is the fast gate.

```
ATTEMPT LIMITER FLOW:

  Request arrives
       │
       ▼
  cache_key = "attempts:{uid}:{qid}"
  attempts_used = cache.get(key, default=0)   ← O(1) Redis GET
       │
       ├── if attempts_used >= max_attempts → 403 Forbidden immediately
       │
       └── else:
               attempt_number = attempts_used + 1
               cache.set(key, attempt_number, timeout=86400)  ← TTL 24h
               create Submission(attempt_number=attempt_number)
               enqueue execute_submission
```

### Rate Limiting (DRF Throttle)

Beyond per-question attempt limits, the submission endpoint has its own rate throttle:

```python
class SubmissionThrottle(UserRateThrottle):
    scope = 'submissions'  # maps to 30/hour in DEFAULT_THROTTLE_RATES
```

General API: 200/hour. Anonymous: 20/hour. Submissions specifically: 30/hour. This prevents burst abuse at the API level before even reaching the attempt logic.

---

## 6. Execution Engine — Docker Sandbox

### Why Not the Alternatives

#### vm2 (JavaScript sandbox)
- Rejected. JavaScript cannot safely sandbox JavaScript. An unfixable escape was found in 2023 and the library was deprecated by its own maintainers. Running arbitrary student code in the same Node.js process as the backend is not an option.

#### Judge0
- Rejected. It is an external API dependency during a live exam. Any outage of Judge0 would immediately fail all student submissions — unacceptable for a high-stakes exam scenario.
- Additionally, Judge0 had 3 CVEs published in April 2024: symlink attacks and SSRF vulnerabilities that could be used to steal AWS credentials. Using it would mean accepting someone else's security patch timeline during live exams.

#### AWS Lambda (seriously evaluated)
- Lambda was evaluated seriously. The free tier covers 5.2 million invocations/month — which would make execution essentially free.
- **Why Lambda was not chosen:** SSE needs a persistent connection that Lambda cannot hold (Lambda invocations are stateless and terminate). Celery Beat (the auto-close scheduler) needs an always-running process. Cold start time for Lambda is 8–15 seconds vs Docker's ~500ms warm start — over the 10-second SLA for results. A Lambda migration would require replacing Celery with SQS, Celery Beat with EventBridge, and SSE with API Gateway WebSockets — a complete rewrite for $0.29/month savings. Not worth it at this scale.

#### Docker — Chosen
Works on Mac (local development), portable to any Linux host, uses kernel namespaces and cgroups for isolation, explicit deny-everything security posture.

### Security Constraints (every container)

```
docker run
  --memory=128m         # mem_limit in docker-py
  --cpus=0.5            # nano_cpus=500_000_000
  --network=none        # network_disabled=True — cannot call home
  --pids-limit=10       # prevents fork bombs
  --read-only           # read-only filesystem — no writing to disk
  --user nobody         # non-root user — cannot escalate
  --rm                  # auto-remove after exit — no container persistence
  [5s timeout]          # killed with SIGKILL after 5 seconds
```

### Language Configuration

```python
language_config = {
    'python': {
        'image': 'python:3.11-alpine',
        'command': ['python3', '-c', wrapped_python]
    },
    'javascript': {
        'image': 'node:18-alpine',
        'command': ['node', '-e', wrapped_js]
    },
    'java': {
        'image': 'openjdk:17-slim',
        'command': ['sh', '-c', java_cmd]
    },
    'cpp': {
        'image': 'gcc:12',
        'command': ['sh', '-c', cpp_cmd]
    },
}
```

### Two-File Pattern

For Python and JavaScript, input is injected directly:

```python
# Python wrapper — stdin injected before student code
wrapped_python = f"import sys, io\nsys.stdin = io.StringIO({repr(input_data + chr(10))})\n{code}"

# JavaScript wrapper — stdin piped before student code  
wrapped_js = f"process.stdin.push({repr(input_data + chr(10))});\nprocess.stdin.push(null);\n{code}"
```

For compiled languages (Java, C++), the pattern is: write source to `/tmp`, compile, pipe input, execute.

The student never sees the test case inputs that the container runs against — only `is_hidden=False` test cases are exposed in the UI. The runner (judge) is fully server-side.

### Lambda Stub (Future Migration Path)

The `execution/tasks.py` file contains `run_in_lambda()` fully implemented but commented out. The switch is a one-line code change:

```python
# Current (Docker):
output, error, exec_time = run_in_docker(code, language, input_data)

# Lambda (commented out):
# output, error, exec_time = run_in_lambda(code, language, input_data)
```

This demonstrates the architecture is migration-ready without having paid the rewrite cost prematurely.

### Container Cost Formula

Cost is tracked per submission using Fargate pricing as the model:

```
cost = hours × vCPU × $0.04048/hr
     + hours × GB   × $0.004445/hr

where hours = (exec_ms + cold_start_ms) / 3_600_000
      vCPU = 0.5  (matches --cpus=0.5)
      GB = 0.125  (matches --memory=128m)
      cold_start_ms = 10,000  (realistic Fargate task init)
```

---

## 7. Real-Time Result Delivery — SSE

### Why SSE and not WebSockets

WebSockets are bidirectional. Result delivery is strictly server-to-client. SSE (Server-Sent Events) is the correct tool when you only need one direction. It works over standard HTTP, requires no handshake protocol, needs no library, and reconnects automatically on drop.

### SSE vs Polling Comparison

| Aspect | SSE | Long Polling | Short Polling |
|---|---|---|---|
| Direction | Server→Client | Server→Client | Server→Client |
| Connection overhead | One HTTP connection per stream | New connection per poll cycle | New connection per N seconds |
| Latency | Event-driven, immediate | Up to poll cycle latency | Up to interval latency |
| Server load at 2000 students | 2000 open connections | 2000 × (polls/min) requests | 2000 × (1/interval) req/s |
| Reconnect on drop | Automatic via Last-Event-ID | Manual | Not applicable |
| Implementation | Django StreamingHttpResponse | Standard HTTP | Standard HTTP |

### Implementation — Server Side

```python
# submissions/sse.py

def event_stream():
    r = redis.Redis.from_url(os.environ.get('REDIS_URL'))
    pubsub = r.pubsub()
    channel = f"submission:{submission_id}"
    pubsub.subscribe(channel)

    # immediately send "waiting" to confirm stream is open
    yield f"data: {json.dumps({'status': 'waiting'})}\n\n"

    for message in pubsub.listen():
        if message['type'] == 'message':
            yield f"data: {message['data'].decode('utf-8')}\n\n"
            break  # one message is enough — result is final

    pubsub.unsubscribe(channel)

response = StreamingHttpResponse(event_stream(), content_type='text/event-stream')
response['Cache-Control'] = 'no-cache'
response['X-Accel-Buffering'] = 'no'  # disable nginx buffering — critical
```

### Implementation — Client Side (Vue)

```typescript
// ExamTakeView.vue

function listenForResult(submissionId: number): void {
    const token = localStorage.getItem('access_token')
    
    // SSE (EventSource) does not support custom headers
    // JWT must be passed as query param
    eventSource = new EventSource(
        `http://localhost:8000/api/submissions/${submissionId}/stream/?token=${token}`
    )

    eventSource.onmessage = async (event) => {
        const data = JSON.parse(event.data)
        submissionStatus.value = data.status

        if (data.status === 'passed' || data.status === 'failed') {
            // fetch full per-test-case results now that execution is done
            const resultsRes = await api.get(`/results/?submission=${submissionId}`)
            results.value = resultsRes.data
            eventSource?.close()
        }
    }
}
```

### SSE Auth — The Token-in-Query-Param Problem

`EventSource` is a browser API. It does not support custom headers. The standard way to authenticate SSE is to pass the JWT as a query parameter: `?token={jwt}`. Django's SSE view reads this from `request.GET.get('token')` before falling back to the `Authorization` header. This is a deliberate design choice, not a security gap — the token is validated identically.

### The Full SSE Flow

```
BROWSER                  DJANGO (SSE endpoint)         CELERY WORKER         REDIS
  │                              │                            │                  │
  │ GET /stream/?token={jwt}     │                            │                  │
  ├─────────────────────────────►│                            │                  │
  │                              │ subscribe(submission:{id}) │                  │
  │                              ├───────────────────────────────────────────►  │
  │                              │                            │                  │
  │ data: {"status":"waiting"}   │                            │                  │
  │◄─────────────────────────────┤                            │                  │
  │                              │                            │                  │
  │          (waiting...)        │                   (Docker runs...)            │
  │                              │                            │                  │
  │                              │              publish(submission:{id},         │
  │                              │              {"status":"passed"})             │
  │                              │                            ├─────────────────►│
  │                              │                            │                  │
  │                              │◄───────────────────────────────────────────── │
  │                              │ message received on channel                   │
  │                              │                            │                  │
  │ data: {"status":"passed"}    │                            │                  │
  │◄─────────────────────────────┤                            │                  │
  │                              │ unsubscribe, stream closes │                  │
  │                              │                            │                  │
  │ GET /results/?sub={id}       │                            │                  │
  ├─────────────────────────────►│                            │                  │
  │◄─────────────────────────────┤                            │                  │
  │ per-test-case results        │                            │                  │
```

### Replay on Refresh (Last-Event-ID)

If a student refreshes during execution, the browser automatically sends `Last-Event-ID` in the reconnect request. The result is cached in Redis with a 1-hour TTL, so if execution has already completed, the status is returned immediately without waiting.

---

## 8. Plagiarism Detection System

This is the most technically sophisticated feature and the one most worth discussing in depth.

### The Core Problem

Naive all-pairs comparison: every submission against every other submission. At 10,000 students, that is 10,000² / 2 = **50 million comparisons**. Running TF-IDF and cosine similarity 50 million times is not feasible on exam close.

The solution: a two-layer filter that cuts the real work to O(K×N) where K is a small subset of suspected students.

### Layer 1 — Behavioural Scoring (O(N) per exam, runs at exam close)

Runs post-exam in batch. Called synchronously (not via `.delay()`) inside `run_exam_plagiarism_check` for every submission before Phase 2 is queued. Does not compare against other students — looks only at each student's own signals.

**Why post-exam, not per-submission:** All submission data is final, cross-question patterns are visible, no false positives from partial data, and there is no actionable surface for trainers mid-exam. Running synchronously inside the batch task guarantees Phase 1 records exist before Phase 2 queries them.

```
EXAM CLOSES → run_exam_plagiarism_check starts
         │
         ▼
    for each submission in exam:
        run_behavioural_check(sub.id)  ← direct call, not .delay()
         │
         ├─► paste_signal()
         │     paste_ratio = paste_char_count / len(code)
         │     ≥ 0.70 → 1.0   (weight: 0.40)
         │     ≥ 0.50 → 0.7
         │     ≥ 0.30 → 0.3
         │     < 0.30 → 0.0
         │
         ├─► time_signal()
         │     ratio = elapsed / DIFFICULTY_BASELINE
         │     baselines: easy=90s, medium=240s, hard=480s
         │     ≤ 0.20 → 1.0   (weight: 0.30)
         │     ≤ 0.40 → 0.7
         │     ≤ 0.60 → 0.3
         │     > 0.60 → 0.0
         │
         ├─► tab_switch_signal()
         │     ≥ 5 switches → 1.0   (weight: 0.15)
         │     ≥ 3 switches → 0.5
         │     ≥ 1 switch   → 0.2
         │     0 switches   → 0.0
         │
         └─► submission_surprise_signal()
               compare current code vs previous attempt
               using SequenceMatcher.ratio()
               if ratio < 0.25 AND status == 'passed' → 1.0  (weight: 0.15)
               if ratio < 0.40 → 0.6
               if ratio < 0.55 → 0.2

         risk_score = 0.40*paste + 0.30*time + 0.15*tab + 0.15*surprise

         signals_fired = count(score > 0 for each signal)
         confidence = HIGH (3+) | MEDIUM (2) | LOW (1)

         if risk_score > 0.2:
             BehaviouralFlag.objects.update_or_create(...)
         
         Submission.behavioural_check_done = True
```

**Why weights are set this way:**
- Paste (0.40): most direct evidence. If 70% of your code arrived via paste, that is hard to misinterpret.
- Time (0.30): strong signal. A hard problem solved in 18 seconds is statistically impossible for genuine work.
- Tab switch (0.15): corroborating evidence only. 1-2 switches is normal (documentation). 5+ is suspicious.
- Submission surprise (0.15): useful context, but only applies when a prior attempt exists (attempt 1 has nothing to compare against).

**Why threshold 0.2 to store:**
Storing every clean submission wastes database space and creates noise in the trainer UI. Only store when there is something to review.

### Layer 2 — Targeted Cross-Comparison (O(K×N), runs at exam close)

```
EXAM CLOSES (manual or auto-close via Celery Beat)
         │
         ▼
    Exam.pre_save signal fires
         │
         ▼
    run_exam_plagiarism_check.delay(exam_id)
         │
         ▼
    Phase 1 (synchronous loop — runs before queuing Phase 2):
    for each submission in exam:
        run_behavioural_check(sub.id)  ← direct call, not .delay()
    → BehaviouralFlag records now exist for all submissions
         │
         ▼
    Phase 2 (queued per question, staggered):
    for each question in exam:
        run_question_plagiarism_check.apply_async(
            args=[question_id],
            countdown=i * 30  ← staggered to avoid CPU spike
        )
```

```
run_question_plagiarism_check(question_id):
         │
         ▼
    Query HIGH-confidence suspects (Phase 1 data must exist first):
    high_risk_ids = BehaviouralFlag
                    .filter(question=q, confidence='high')
                    .values_list('submission__student_id')

         │
         ▼
    Phase 2: Pull best submission for EVERY student
    (passed attempt if exists, else latest attempt)
    → all_subs list (N items)

         │
         ▼
    Phase 3: AST-normalize all N submissions
    normalize_code(sub.code, sub.language)
    → strip variable names → VAR_0, VAR_1...
    → strip comments, docstrings
    → unparse back to source

         │
         ▼
    Phase 4: TF-IDF vectorize full cohort ONCE
    vectorizer = TfidfVectorizer()
    matrix = vectorizer.fit_transform(normalized)
    sim_matrix = cosine_similarity(matrix)
    → O(N·V) time, O(N²) space for sim_matrix

         │
         ▼
    Phase 5: Upper-triangle comparison
    Only check pair (i, j) if:
      all_subs[i].student_id IN high_risk_ids
      OR
      all_subs[j].student_id IN high_risk_ids

    if sim_matrix[i][j] >= 0.80:
        PlagiarismFlag.objects.get_or_create(
            submission_1, submission_2,
            similarity_score=sim,
            detection_method='behavioural_filtered_tfidf'
        )
        notify_plagiarism_flagged(pf)
    
    Submission.similarity_check_done = True
    for all subs in this question
```

### Why Suspects vs Full Cohort (not Suspects vs Suspects)

This is a subtle but critical design decision.

If you compare only flagged suspects against each other, you miss the source student. A student who typed slowly and didn't paste has clean behavioural signals. If a flagged student copied from them, they appear nowhere in the behavioural flags. Suspects-vs-suspects would never find that pair.

The correct approach: suspects are the **query set**, the full cohort is the **search space**.

```
SUSPECTS vs SUSPECTS (wrong):
  K=10 suspects, 10 sources never flagged
  → 10×10 = 100 comparisons
  → MISSES all 10 sources

SUSPECTS vs FULL COHORT (correct):
  K=10 suspects, N=200 total students
  → 10×200 = 2,000 comparisons
  → CATCHES sources regardless of their own behaviour signals
```

### Complexity Comparison

| Approach | Complexity | 10k students, 5% flag rate |
|---|---|---|
| All-pairs naive | O(N²) | 50,000,000 comparisons |
| Suspects vs suspects | O(K²) | 250,000 comparisons, misses sources |
| **Our approach (suspects vs cohort)** | **O(K×N)** | **5,000,000 comparisons** |

At 5% K/N ratio: our approach is 10x cheaper than naive while catching the cases that suspects-vs-suspects misses.

### Code Normalization — AST Variable Renaming

Before TF-IDF comparison, all user-defined names are replaced with sequential tokens. This catches the most common evasion technique: renaming variables.

```python
# Original student code:
def twoSum(nums, target):
    seen = {}
    for i, val in enumerate(nums):
        complement = target - val
        if complement in seen:
            return [seen[complement], i]
        seen[val] = i

# After AST normalization:
def VAR_0(VAR_1, VAR_2):
    VAR_3 = {}
    for VAR_4, VAR_5 in enumerate(VAR_1):
        VAR_6 = VAR_2 - VAR_5
        if VAR_6 in VAR_3:
            return [VAR_3[VAR_6], VAR_4]
        VAR_3[VAR_5] = VAR_4
```

Now TF-IDF sees the same token stream regardless of what the student named their variables. Comments and docstrings are stripped before normalization.

Python uses `ast.parse()` + `ast.NodeTransformer` + `ast.unparse()` — structural transformation. JavaScript uses regex substitution (not AST, since Python's `ast` module only handles Python).

### Auto-Close Trigger Chain

```
Celery Beat
(every 60 seconds)
     │
     ▼
auto_manage_exams()
     │
     ├── find exams where end_time < now AND is_active=True
     │
     ├── for each: exam.is_active = False; exam.save()
     │                    │
     │                    ▼
     │          pre_save signal on Exam model fires
     │          (codemas/signals.py)
     │                    │
     │                    ▼
     │          run_exam_plagiarism_check.delay(exam_id)
     │
     └── find exams where start_time <= now AND is_active=False
         for each: exam.is_active = True; exam.save()
```

This means manual close and auto-close behave identically. The signal is the single source of truth for triggering plagiarism detection.

### Pre-Save Signal (Django Signals)

```python
# codemas/signals.py

@receiver(pre_save, sender=Exam)
def on_exam_close(sender, instance, **kwargs):
    if instance.pk:
        old = Exam.objects.get(pk=instance.pk)
        # only fire when transitioning from active → inactive
        if old.is_active and not instance.is_active:
            from plagiarism.tasks import run_exam_plagiarism_check
            run_exam_plagiarism_check.delay(instance.pk)
```

This pattern ensures the plagiarism job is always triggered regardless of how the exam was closed (manual trainer action, Celery Beat scheduler, or admin panel).

### Plagiarism Review Workflow

```
DETECTION                TRAINER UI                    AUDIT LOG
     │                       │                              │
     ▼                       │                              │
BehaviouralFlag created  ──► AnnotationQueueView           │
(risk > 0.2)                 │                              │
                             │  Trainer sees:               │
PlagiarismFlag created  ───► │  - risk_score bar            │
(similarity >= 0.80)         │  - which signals fired       │
                             │  - signal scores             │
                             │  - matched students          │
                             │  - similarity %              │
                             │  - "View" to jump to         │
                             │    other submission          │
                             │                              │
                             │  Trainer clicks:             │
                             │  [Confirm] or [Dismiss]      │
                             │     │                        │
                             │     ▼                        │
                             │  PATCH /plagiarism/          │
                             │        flags/{id}/review/    │
                             │     │                        │
                             │     ▼                        │
                             │  ReviewAuditLog.create() ───►│
                             │  (flag_type, flag_id,        │
                             │   action, note,              │
                             │   reviewed_by, timestamp)    │
```

---

## 8.1 Policy Versioning and Changelog

### Why Versioning Matters

The `PlagiarismPolicy` singleton stores nine configurable thresholds (weights, baselines, similarity threshold). When a trainer changes these values mid-cohort, the algorithm's scoring changes immediately for all future submissions. Without a changelog, a disputed flag from two weeks ago cannot be audited — the trainer cannot know what weights were in effect when the flag was generated.

### Snapshot vs Diff Approach

Full JSON snapshots were chosen over field-level diffs for three reasons:

1. **The model is small** — nine float/int fields. Storing the complete state before and after every change wastes negligible space.
2. **Point-in-time query is trivial** — "what policy was active on date X?" = find the latest `PolicyVersion` with `changed_at <= X`. One lookup, no reconstruction from a chain of diffs.
3. **Dispute resolution is unambiguous** — a `snapshot_before` contains the exact values the algorithm used to generate a flag, with no dependency on intermediate state.

### PolicyVersion Model

```python
POLICY_FIELDS = [
    'paste_weight', 'time_weight', 'tab_weight', 'submission_weight',
    'min_risk_score', 'similarity_threshold',
    'easy_baseline_s', 'medium_baseline_s', 'hard_baseline_s',
]
```

Every `PUT /api/plagiarism/policy/` call:
1. Reads current state → `snapshot_before`
2. Saves updated policy
3. Reads new state → `snapshot_after`
4. Creates a `PolicyVersion` record with both snapshots and the trainer's `change_reason`

### Affected Exam Detection

The changelog view computes which exams overlapped each policy's active window:

```
Each PolicyVersion is active from its changed_at
until the next newer version supersedes it.

Active period: [period_start, period_end)
  period_start = v.changed_at
  period_end   = versions[i-1].changed_at  (the newer version)
               = now                        (for the current version)

Temporal overlap query:
  Exam.objects.filter(
      start_time__lt=period_end,
      end_time__gt=period_start,
  )

Overlap means: the exam started before the policy ended
               AND the exam ended after the policy started.
```

Each overlapping exam is labeled:
- **"started under this policy"** — exam started after the policy took effect
- **"in-flight at change"** — exam was already running when the policy changed

This gives trainers full context for dispute resolution: "this student was flagged under policy v3, which had a paste_weight of 0.40 and similarity_threshold of 0.75."

---

## 8.2 Cohort Investigation and Cheating Ring Detection

### The Problem with Flat Flag Queues

The annotation queue shows individual flags in isolation. A trainer reviewing 40 pending flags has no way to know whether those flags cluster around one exam (indicating a systemic problem) or are spread uniformly. Worse, they cannot see whether the flagged students form a group — if A copied from B who copied from C, the queue shows three separate flags with no indication they're connected.

### Three-Level Investigation Model

```
Level 1 — Cohort Health Feed  (/plagiarism/cohorts/)
  All exams ranked by flag rate
  Flag rate = flagged_students / total_submissions
  Health: green (<10%), amber (10–25%), red (>25%)

Level 2 — Cohort Deep Dive  (/plagiarism/cohorts/<exam_id>/)
  KPI strip: total students, flagged count, flag rate, ring count
  Cheating rings (connected components)
  Student risk ranking

Level 3 — Student Case  (/support/student/<id>/)
  Existing StudentCaseView — linked from both rings and student table
```

### Combined Risk Score

Each student in a cohort gets a `combined_risk` score computed from both signal types:

```
b_risk     = behavioural flag risk_score (0.0–1.0), or 0 if none
sim_factor = min(similarity_pairs_count / 5.0, 1.0)

combined_risk = b_risk * 0.6 + sim_factor * 0.4
```

Weight rationale: behavioural signals are direct evidence from this student's own session (60%); similarity pairs are cross-student evidence that requires context (40%). A student with both a high behavioural score and multiple similarity matches ranks highest.

### Cheating Ring Detection — Connected Components

The core insight: if A has a similarity flag with B, and B has a similarity flag with C, then A, B, and C are likely in the same plagiarism ring — even if A and C never directly matched. A flat flag queue cannot surface this because it only shows pairs.

Implementation: BFS on the similarity flag adjacency graph.

```
Build adjacency list from all PlagiarismFlags in this exam:
  adj = {student_id: {other_student_id, ...}}

  for each PlagiarismFlag(submission_1, submission_2, flagged=True):
      adj[s1.student_id].add(s2.student_id)
      adj[s2.student_id].add(s1.student_id)

BFS connected components:
  visited = set()
  for start_id in adj:
      if start_id in visited: continue
      
      component = []
      stack = [start_id]
      while stack:
          node = stack.pop()
          if node in visited: continue
          visited.add(node); component.append(node)
          stack.extend(adj[node] - visited)
      
      if len(component) >= 2:
          emit ring(members=component, ...)

Ring stats:
  avg_similarity — mean similarity score across all edges in component
  max_similarity — highest single pair score
  confirmed      — how many edges have been confirmed by a trainer
```

### Why This Catches Cases the Flag Queue Misses

```
Scenario: 5 students, 4 similarity flags
  A ─── B ─── C ─── D ─── E

Flag queue shows: 4 separate pairs (A-B, B-C, C-D, D-E)
Ring detection shows: one ring of 5 members

Without ring detection:
  Trainer reviews A-B, dismisses if only 84% similar
  Never realizes A-B-C-D-E are one network

With ring detection:
  Ring of 5 appears immediately — trainer investigates the whole group
```

### Recidivism Tracking

For each flagged student, `past_flags` counts how many similarity or behavioural flags they have in **other** exams. A student who has been flagged in three consecutive exams is a different risk profile from a first-time flag.

```python
past_sim = PlagiarismFlag.objects.filter(
    flagged=True,
    Q(submission_1__student_id=sid) | Q(submission_2__student_id=sid)
).exclude(
    Q(submission_1_id__in=sub_ids_this_exam) | Q(submission_2_id__in=sub_ids_this_exam)
).count()

past_b = BehaviouralFlag.objects.filter(
    submission__student_id=sid
).exclude(submission__exam=exam).count()

student['past_flags'] = past_sim + past_b
```

---

## 9. Code Similarity Search

A separate feature distinct from plagiarism detection. Trainers can paste any code snippet and find the most similar submissions across all exams and all students.

### Why This Exists

- Investigating a suspected source of plagiarism: paste the common code to find who originally had it
- Searching across exam boundaries: plagiarism that spans multiple exams (student reuses old code)
- Debugging: finding which students are using a specific anti-pattern

### Algorithm

```
TRAINER pastes query code
         │
         ▼
    POST /api/plagiarism/search/
    {code: "...", language: "python", top_k: 15}

         │
         ▼
    1. Normalize query: normalize_code(query, language)

         │
         ▼
    2. Fetch corpus: Submission.objects[:2000]
       (cap at 2000 for memory — TF-IDF matrix grows with N)

         │
         ▼
    3. For each submission: use pre-stored normalized_code
       if empty (old submission): normalize on the fly

         │
         ▼
    4. Check Redis cache:
       cache_key = 'tfidf:' + MD5(
           ','.join(f'{s.id}:{s.language}' for s in submissions)
       )
       
       CACHE HIT:  deserialize (vectorizer, corpus_matrix) from pickle
                   query_vec = vectorizer.transform([query])
       
       CACHE MISS: vectorizer = TfidfVectorizer()
                   corpus_matrix = vectorizer.fit_transform(corpus)
                   cache.set(key, pickle.dumps((vectorizer, corpus_matrix)),
                             timeout=300)
                   query_vec = vectorizer.transform([query])

         │
         ▼
    5. scores = cosine_similarity(query_vec, corpus_matrix)[0]

         │
         ▼
    6. Sort descending, return top_k where score > 0.1
```

### Two Critical Optimizations

**Optimization 1: Pre-computed normalized_code**

At submission time, after execution completes, the code is normalized and stored:

```python
# execution/tasks.py
from plagiarism.tasks import normalize_code
submission.normalized_code = normalize_code(submission.code, submission.language)
submission.save()
```

This means at search time, we do not need to normalize 2000 submissions. We just read the pre-stored normalized text. At 2000 submissions, this saves approximately 2000 × (AST parse + unparse) calls per search.

**Optimization 2: TF-IDF matrix cached in Redis**

The vectorizer and corpus matrix are serialized with pickle and cached for 5 minutes. The cache key is the MD5 hash of all submission IDs and languages — it automatically invalidates when a new submission is created (the hash changes).

```
Without cache:  every search call → fit_transform(2000 docs) → ~50ms
With cache:     subsequent searches → transform([1 doc]) → ~2ms
                Cache invalidates when new submissions arrive
```

### Why Not a Vector Database (pgvector, FAISS, Pinecone)

This is a common question and the answer has an important subtlety:

**TF-IDF vectors are corpus-relative.** The IDF score for a token depends on how many documents in the corpus contain it. If you compute a TF-IDF vector for a submission on Day 1 and store it, that vector is **stale** on Day 30 when 200 more submissions have been added. The IDF denominators have changed.

**Neural embeddings (CodeBERT) are corpus-independent.** The model weights don't change. An embedding computed at submission time is valid forever. This is when a vector database makes sense — pre-compute the embedding once, store it, search at query time.

**pgvector** was attempted on PostgreSQL 15 on macOS. The extension could not be built. This is a known macOS build issue, not an architectural blocker.

**At current scale (2000 submissions):** TF-IDF runs in ~50ms. This is not a bottleneck.

**Migration path to vector DB (at 100k+ submissions):**
1. Add CodeBERT embedding (400MB model, GPU-preferred)
2. Compute embedding at submission time, store in `Submission.embedding` column
3. Use pgvector or FAISS index for approximate nearest-neighbour search
4. Search becomes: embed query → FAISS.search(query_vec, k=15) → sub-millisecond

### Similarity Techniques Reference

| Technique | What It Detects | Complexity | Used In CodeMas |
|---|---|---|---|
| TF-IDF + cosine | Token frequency similarity | O(N·V) build, O(V) query | Yes — similarity search |
| AST comparison | Structural equivalence (for→while) | O(N) per pair | Yes — normalization step |
| CodeBERT embeddings | Semantic meaning | O(1) per query if pre-built | Future path |
| MOSS token fingerprinting | Partial copying (substrings) | O(N·W) | Not implemented |
| Levenshtein / SequenceMatcher | Character-level edit distance | O(M×N) per pair | Yes — submission_surprise signal |

---

## 10. Auto-Exam Lifecycle — Celery Beat

### Celery Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                       Celery Setup                                  │
│                                                                     │
│  settings.py:                                                       │
│    CELERY_BROKER_URL = REDIS_URL                                   │
│    CELERY_RESULT_BACKEND = REDIS_URL                               │
│    CELERY_BEAT_SCHEDULE = {                                        │
│        'auto-manage-exams': {                                      │
│            'task': 'exams.tasks.auto_manage_exams',               │
│            'schedule': 60.0,  # every 60 seconds                  │
│        }                                                           │
│    }                                                               │
└────────────────────────────────────────────────────────────────────┘

                     ┌─────────────────────────────────────┐
                     │            Redis                     │
                     │                                     │
                     │  Queue: celery (default)            │
                     │    - execute_submission tasks        │
                     │    - run_exam_plagiarism_check       │
                     │    - run_question_plagiarism_check   │
                     │                                     │
                     │  Pub/Sub channels:                  │
                     │    - submission:{id}                 │
                     └────────────────┬────────────────────┘
                                      │
                     ┌────────────────▼────────────────────┐
                     │         Celery Worker               │
                     │  (python -m celery -A codemas       │
                     │   worker --loglevel=info)           │
                     │                                     │
                     │  + Celery Beat                      │
                     │  (python -m celery -A codemas       │
                     │   beat --loglevel=info)             │
                     └─────────────────────────────────────┘
```

### Why Celery Cannot Be Replaced by Lambda Here

1. **Persistent Redis pub/sub subscription**: The SSE endpoint holds an open `pubsub.listen()` loop. Lambda terminates after each invocation — it cannot hold this connection open. When the Celery worker publishes to `submission:{id}`, there must be a persistent listener on the other side.

2. **Celery Beat scheduler**: Runs `auto_manage_exams` every 60 seconds. This is an always-running process. AWS EventBridge could replace it, but that adds a dependency on AWS infrastructure and requires a different invocation model.

3. **Cost analysis**: Full Lambda migration (Celery → SQS, Celery Beat → EventBridge, SSE → API Gateway WebSockets) would save approximately $0.29/month at 5,000 submissions/month. The rewrite cost is not justified.

### Task Staggering on Exam Close

When an exam closes, it may have 10+ questions. All plagiarism checks are staggered with a `countdown` delay to avoid a CPU spike:

```python
for i, question in enumerate(exam.questions.all()):
    run_question_plagiarism_check.apply_async(
        args=[question.id],
        countdown=i * 30,  # 0s, 30s, 60s, 90s...
    )
```

This distributes CPU load across time rather than hitting all questions simultaneously.

---

## 11. Trainer Dashboard and Monitoring

### Dashboard KPIs

```
GET /api/dashboard/

Returns:
  kpis:
    total_students         ← User.objects.filter(role='student').count()
    total_submissions      ← Submission.objects.count()
    pass_rate              ← passed / total * 100
    plagiarism_flags       ← PlagiarismFlag.filter(flagged=True, review_status='pending').count()

  timeline:
    last 14 days, per-day: {date, total, passed}
    gaps filled with 0s — no missing dates in chart

  question_stats:
    per question: pass_rate, avg_exec_time, avg_attempts_to_pass

  language_distribution:
    {python: N, javascript: N, java: N, cpp: N}

  cohort_comparison:
    per cohort: student_count, total_submissions, pass_rate

  top_performers (top 5 by pass rate)
  struggling_students (pass_rate < 40%, up to 5)
```

### Submission Detail View

The most information-dense view in the system. Shows:

```
┌─────────────────────────────────────────────────────────────────┐
│ Student: alice | Exam: Week 3 | Q2: Two Sum | Attempt 2         │
├─────────────────────────────────────────────────────────────────┤
│ KPIs: exec_time=234ms | time_used=67% | test_cases=3/3         │
│       ip_address=10.0.0.45 | behav_done=✓ | sim_done=✓         │
├─────────────────────────────────────────────────────────────────┤
│ CODE:                                                            │
│ def solution(nums, target): ...                                  │
├─────────────────────────────────────────────────────────────────┤
│ TEST CASES:                                                      │
│ [1] input=[2,7,11,15], target=9 | expected=[0,1] | got=[0,1] ✓ │
│ [2] input=[3,2,4], target=6     | expected=[1,2] | got=[1,2] ✓  │
│ [3] input=[3,3], target=6       | expected=[0,1] | got=[0,1] ✓  │
├─────────────────────────────────────────────────────────────────┤
│ BEHAVIOURAL FLAGS:                                               │
│ risk_score: 0.73 ████████████░░ HIGH confidence                 │
│ Signals: "Pasted 68% of code (142 chars)"                       │
│          "Solved medium in 1m 12s (baseline 4m)"               │
│ [Confirm Plagiarism]  [Dismiss]                                  │
├─────────────────────────────────────────────────────────────────┤
│ SIMILARITY FLAGS:                                                │
│ Matched: bob | 87% similar | [View Submission #142]             │
│ Status: pending review                                           │
└─────────────────────────────────────────────────────────────────┘
```

### Annotation Queue

Separate view that lists all pending behavioural and similarity flags. Trainers can filter by status (pending/confirmed/dismissed) and review in bulk. Every review action is written to `ReviewAuditLog` for a full audit trail.

---

## 12. Cost Analysis Engine

### Cost Formula

```python
# dashboard/views.py

_FARGATE_VCPU_PER_HOUR = 0.04048   # USD per vCPU-hour
_FARGATE_GB_PER_HOUR   = 0.004445  # USD per GB-hour
_CONTAINER_VCPU        = 0.5       # matches --cpus=0.5
_CONTAINER_GB          = 0.125     # matches --memory=128m
_COLD_START_MS         = 10000     # Fargate task init (conservative)

def _cost_for_ms(exec_ms):
    total_ms = (exec_ms or 500) + _COLD_START_MS
    hours = total_ms / 3_600_000
    return (hours * _CONTAINER_VCPU * _FARGATE_VCPU_PER_HOUR +
            hours * _CONTAINER_GB    * _FARGATE_GB_PER_HOUR)
```

Note: the 10-second cold start dominates for fast code (sub-second Python). Most of the cost is container spin-up, not execution. This is exposed as an optimization suggestion in the UI.

### Fixed Monthly Infrastructure

| Component | Monthly Cost | Notes |
|---|---|---|
| Redis (ElastiCache t3.micro) | $13.00 | us-east-1, on-demand |
| PostgreSQL (RDS t3.micro) | $26.00 | us-east-1, on-demand |
| Celery worker | $0.00 | Fargate model — no dedicated machine |
| **Fixed total** | **$39.00** | |

### Optimization Suggestions (Auto-Generated)

The cost view generates context-aware suggestions based on actual submission data:

1. **High failed-submission cost**: if >35% of compute is on failed submissions → add partial feedback
2. **High retry rate**: if >30% of submissions are retries → show compiler errors inline
3. **Cold start dominance**: if cold start is >60% of total time → pre-warm containers
4. **Slow languages**: if avg exec > 2000ms for a language → stricter time limits
5. **Low pass rate**: if overall <50% → review question difficulty calibration
6. **Spot instances**: if >500 total submissions → switch Celery workers to EC2 Spot

---

## 13. Authentication and Authorization

### JWT with Simple JWT

```
ACCESS_TOKEN_LIFETIME  = 1 day   (development; production should be 15min)
REFRESH_TOKEN_LIFETIME = 7 days

Flow:
  POST /api/token/       → returns {access, refresh}
  POST /api/token/refresh/ → returns {access} using refresh token
```

### Role-Based Guards

All views check `request.user.role` explicitly. There is no Django permission framework in use — role is checked inline:

```python
if request.user.role != 'trainer':
    return Response({'error': 'Forbidden'}, status=403)
```

Roles: `student`, `trainer`, `admin`.

### SSE Authentication Special Case

EventSource (browser SSE API) does not support custom headers. Token is passed as a URL query parameter:

```
GET /api/submissions/{id}/stream/?token={access_token}
```

The SSE view reads both `HTTP_AUTHORIZATION` header and `?token=` query param, validating both identically via `AccessToken(token_str)`.

### Rate Limiting

```python
# settings.py
'DEFAULT_THROTTLE_RATES': {
    'anon':        '20/hour',   # login attempts, unauthenticated
    'user':        '200/hour',  # general authenticated use
    'submissions': '30/hour',   # submission endpoint specifically
}
```

Submission throttle is tighter because each submission spins a Docker container — 200/hour from one user would be a CPU denial-of-service.

---

## 14. Concurrency and Scaling

### How 10,000 Concurrent Students Was Approached

**Job queue architecture is the key.** Without a queue, 2000 simultaneous submissions would hit 2000 Docker `client.containers.run()` calls from Django synchronously — exhausting threads and Docker daemon connections immediately.

With Celery + Redis:
1. Django receives submission, writes to DB, enqueues task: ~5ms
2. Returns 201 to student immediately
3. Worker pool processes containers at controlled concurrency

```
10,000 students submit simultaneously
           │
           ▼
Django (Gunicorn, N workers)
  - validate auth: ~1ms
  - check Redis counter: ~1ms
  - write Submission to PG: ~5ms
  - enqueue task: ~2ms
  - return 201: ~10ms total
           │
           ▼
Redis queue absorbs the spike
  - 10,000 tasks queued, 0 dropped
           │
           ▼
Celery workers drain queue
  - each worker: 1 task at a time
  - each task: 1-5 Docker containers
  - queue drains at worker_count × (1/avg_task_time) rate
  - result delivered via SSE as each task completes
```

### Dashboard Polling Cache

The trainer cohort dashboard was polling every 10 seconds. With 300 real students, this produced 30 requests/second hitting the database. Fixed by Redis caching the dashboard response with a 10-second TTL and adding jitter to prevent all clients polling synchronously:

```python
# Dashboard response cached 10s
cache_key = f"dashboard:trainer:{request.user.id}"
cached = cache.get(cache_key)
if cached:
    return Response(cached)
# ... compute ...
cache.set(cache_key, data, timeout=10)
```

Jitter: the SSE "waiting" initial message resets a client-side interval timer. Different students submit at different times, naturally spreading their polling intervals.

### The 2000 Concurrent User Number

This is a real usage number from running CodeMas with actual students at Masai School, not a synthetic load test. It is both a strength (validated against real load) and a limitation (not tested beyond 2000 with instrumented load testing).

**What would be added from day one now:**
- k6 load tests simulating 2000 VUs all submitting simultaneously
- Measure p95 latency on submission endpoint AND SSE result delivery
- Deadline spike scenario: all users submit in a 30-second window
- Set pass/fail thresholds in CI pipeline
- The 30-second spike is the hardest pattern and the primary justification for the job queue

### Concurrency Tests Written

- Jest unit tests: score parser, attempt limiter, result storage
- Integration tests: full submit-to-result flow including actual Docker execution
- Real exam sessions: monitored EC2 CPU, memory, container count, scaled gradually

---

## 15. Infrastructure Cost Comparison — EC2 vs Fargate vs Lambda

### At 5,000 Submissions Per Month

```
COST COMPARISON (5,000 submissions/month)

EC2 (t3.small)
  Redis ElastiCache:  $13.00
  RDS t3.micro:       $26.00
  EC2 t3.small:       $15.00 (always on, regardless of submissions)
  Compute:            included in EC2
  ─────────────────────────────────
  TOTAL: $54.00/month

Fargate (containers on demand)
  Redis ElastiCache:  $13.00
  RDS t3.micro:       $26.00
  Celery worker:       $0.00 (no dedicated EC2)
  5k submissions:      $0.30
  ─────────────────────────────────
  TOTAL: $39.30/month  (saves $14.70 vs EC2)

Lambda (serverless functions)
  Redis ElastiCache:  $13.00
  RDS t3.micro:       $26.00
  Lambda 5k invokes:   $0.007
  ─────────────────────────────────
  TOTAL: $39.01/month  (saves $14.29 vs EC2)
```

### Crossover Point: EC2 vs Fargate

```
EC2 fixed cost:    $54.00/month (flat)
Fargate:           $39.00 + ($0.30 / 5000) × N

Solve for N where Fargate == EC2:
  39.00 + (0.30/5000) × N = 54.00
  (0.00006) × N = 15.00
  N = 250,000 submissions/month

Below 250,000: Fargate is cheaper
Above 250,000: EC2 becomes cheaper (flat cost amortized)
```

```
COST vs SUBMISSIONS

Cost ($)
  │
60│  EC2 (flat $54) ─────────────────────────────────────────────
  │                                          ╱ Fargate
50│                                        ╱
  │                                      ╱
40│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─╱
  │  Fargate ($39 + compute)         ╱ crossover ~250k submissions
30│                                ╱
  │                              ╱
20│                            ╱
  │                          ╱  Lambda ≈ Fargate at this scale
10│                        ╱    (Lambda wins on compute cost
  │                      ╱      but requires full rewrite)
  └────────────────────────────────────────────────────────────►
       50k    100k    150k    200k    250k   300k   submissions
```

### Lambda Tradeoffs Summary

| Factor | Lambda | Docker/EC2 |
|---|---|---|
| Cold start | 100ms | 500ms warm, 8-15s cold (Fargate) |
| Persistent connections | Cannot hold | Yes (SSE, pub/sub) |
| Always-running process | Cannot (terminates per invoke) | Yes (Celery Beat) |
| Free tier | 5.2M invocations/month | No |
| Rewrite required | Yes (Celery→SQS, SSE→WS) | No |
| Cost at 5k/month | $39.01 | $39.30 (Fargate), $54 (EC2) |
| Savings vs EC2 | $14.99/month | $14.70/month (Fargate) |
| Verdict at this scale | Not worth rewrite | Fargate is right choice |

---

## 16. Entrupy Parallels

These are the specific connections between CodeMas design decisions and Entrupy's engineering problems.

### Direct Architecture Parallels

| CodeMas Component | Entrupy Equivalent | Why It's the Same Problem |
|---|---|---|
| Docker execution pipeline | Image processing / ML inference pipeline | Both: async job, resource-limited container, result returned via queue |
| Celery job queue (Redis broker) | Async authentication request processing | Both: spike absorption, ordered processing, retry semantics |
| TF-IDF + cosine similarity search | Image similarity search (CNN embeddings + vector DB) | Same mathematical foundation: vector space, cosine distance |
| Trainer cohort dashboard | Internal monitoring dashboards | Real-time aggregation, time-series, role-based data views |
| Exam builder + test cases | Data annotation tools + knowledge encoding | Structured ground truth creation, version control, validation |
| Hidden test cases | Protected ground truth data | Information asymmetry by design — students/users never see judge data |
| Attempt limiting (Redis atomic counter) | Policy enforcement systems | Rate limiting, usage gates, abuse prevention |
| SSE result delivery | Real-time status updates in internal tools | Event-driven UI updates without polling overhead |
| Role-based auth (student/trainer/admin) | Account management systems | RBAC with explicit guards per endpoint |
| Code similarity search (TF-IDF + cache) | Image search across 25M microscopic images | Query normalization, corpus indexing, approximate nearest neighbour |
| Annotation queue (plagiarism review) | Authentication specialist annotation workflow | Human-in-the-loop review, audit trails, confirmation/dismissal workflow |
| normalized_code pre-computed at submit time | Embeddings pre-computed at image upload time | Decouple computation from query — don't pay cost at search time |
| Redis TF-IDF matrix cache (MD5 key) | Vector index pre-built, queried at search time | Amortize build cost, invalidate on corpus change |

### The Core Shared Problem: Scale + Accuracy

**CodeMas:** 10,000 concurrent students need results in <10s. Plagiarism detection must be fast enough to not delay exam close. TF-IDF gives 50ms search at 2k submissions.

**Entrupy:** 25 million microscopic images. Authenticating a new item means searching across 25M vectors. Pre-computed embeddings + vector index = sub-second at any scale.

**The decision pattern is identical:** Don't compute on query. Compute at write time. Cache/index the result. Query against the cached representation.

### The Human-in-the-Loop Pattern

**CodeMas:** Plagiarism algorithm raises flags. Trainer confirms or dismisses. Every decision logged to `ReviewAuditLog` with `reviewed_by`, `action`, `note`, `created_at`. Algorithm never takes unilateral action — it surfaces evidence.

**Entrupy:** Authentication algorithm produces a confidence score. Human specialist reviews edge cases. Decision is logged. This is the same annotation workflow.

The design principle: **algorithms filter, humans decide.** The algorithm reduces 10,000 comparisons to 50 flags for human review. The human provides the final judgment and the audit trail.

### Behavioural Signals Parallel

**CodeMas behavioural signals:** paste events, time-to-complete, tab switches, attempt discontinuity. Captured by the client, sent with the submission, stored immutably.

**Entrupy authentication signals:** image quality, metadata, provenance data, item history. Captured at submission, used as features. Neither system trusts the submitter's account of their own behaviour — data is captured passively.

---

## 17. Known Gaps and Honest Limitations

Being clear about limitations demonstrates better engineering judgment than overselling.

### Plagiarism Detection Gaps

| Scenario | Detection Status | Why |
|---|---|---|
| Manual retyping (no paste events) | **Not detected by paste signal** | No clipboard event fires. Time signal may catch it if fast enough. |
| AI-generated code (typed slowly) | **Not detected** | Looks like genuine work if pasted manually or typed. No corpus of "AI output" to compare against. |
| Pre-exam memorization | **Not detected** | Cannot detect without a reference corpus of solutions memorized before the exam. |
| Collusion between two slow typists | **Not detected by either layer** | Both have clean behavioural signals. Neither is a suspect. O(K×N) approach misses pairs where K=0. |
| Cross-language copying (Python→JavaScript) | **Not detected** | Normalization is language-specific. Cross-language similarity requires semantic analysis (CodeBERT). |

### Technical Debt

1. **JavaScript normalization uses regex, not AST**: Python normalization uses `ast.parse()` which is structural. JavaScript normalization uses regex-based identifier substitution — it cannot handle all syntax correctly. A proper JS AST normalizer (using `esprima` or similar) would be more accurate.

2. **`snapshot_score` field name is misleading**: The field was intended for code-growth monitoring but was repurposed for tab switch score to avoid a database migration. A future migration should rename it to `tab_score`.

3. **TF-IDF at scale**: At 2000 submissions, TF-IDF runs in ~50ms. At 100,000+ submissions, this becomes a bottleneck. The migration path is CodeBERT embeddings stored at submission time and pgvector/FAISS at query time.

4. **No load testing framework**: Real-world 2000 concurrent users was the validation, not instrumented load tests. k6 tests from day one would provide p95 SLAs and regression detection.

5. **JWT access token lifetime is 1 day**: This is a development setting. Production should be 15 minutes with refresh token rotation.

6. **Dashboard queries are N+1**: The `DashboardView` has a loop over all students to compute stats. At 10,000 students, this is 10,000 ORM queries. Production fix: use `annotate()` with subqueries or a materialized view.

### Infrastructure Gaps

1. **No horizontal scaling configuration**: Gunicorn runs with default workers. A production deployment would configure `workers = 2 * CPU_count + 1` and use async workers (gevent/eventlet) for SSE connections.

2. **Docker images not pre-pulled**: Cold start is dominated by image pull (~200MB for Python Alpine). Pre-baking images into the EC2 AMI or using a local registry would cut cold start from 10s to ~200ms.

3. **No circuit breaker on Docker execution**: If the Docker daemon goes down, all Celery tasks fail. A circuit breaker pattern with fallback to "execution unavailable" would prevent queue backup.

---

## 18. What to Say to Drew — Quick Reference

These are conversation anchors for specific topics Drew might raise.

---

### "Walk me through your architecture."

Start with the problem: 10,000 concurrent students, results in under 10 seconds. The core constraint is that code execution is expensive and unpredictable — you can't do it synchronously in an HTTP request. The job queue is the answer. Student submits → Django validates and enqueues in ~10ms → returns 201 → student opens SSE stream → Celery worker runs Docker → publishes to Redis → SSE pushes result. The student waits 3-8 seconds depending on execution time, never the Django request latency.

---

### "Why SSE and not WebSockets?"

WebSockets are for bidirectional communication. Result delivery is server-to-client only. SSE is simpler: it's a long HTTP response with `text/event-stream` content type. No handshake, no library, browser reconnects automatically using Last-Event-ID. The only gotcha is that EventSource doesn't support custom headers, so we pass the JWT as a query parameter.

---

### "Why did you choose Docker over Lambda?"

Three reasons. One: SSE needs a persistent connection — Lambda terminates after each invocation and cannot hold a pub/sub subscription open. Two: Celery Beat needs an always-running process. Three: the cost savings from switching to Lambda at this scale are $0.29/month, which doesn't justify the complete rewrite (Celery → SQS, Celery Beat → EventBridge, SSE → API Gateway WebSockets). Lambda stub exists in the codebase — one-line change to switch — but the infrastructure overhead isn't worth it yet.

---

### "Explain your plagiarism system."

It's two layers, both running post-exam when the exam closes. Layer 1 (behavioural) runs first, synchronously in the exam-close Celery task: four signals (paste ratio, time-to-complete vs difficulty baseline, tab switches, attempt discontinuity) scored and stored as BehaviouralFlags. Running post-exam gives full context — all submissions are final, no false positives from partial data mid-exam, and trainers have no actionable surface during the exam anyway. Layer 2 runs after: TF-IDF cross-comparison restricted to high-confidence suspects vs the full cohort. Running Phase 1 synchronously before queuing Phase 2 is a deliberate ordering guarantee — Phase 2 filters by HIGH-confidence BehaviouralFlags, so those records must exist first. At 10k students with 5% flag rate: 5M comparisons instead of 50M naive.

---

### "How does the similarity search work?"

Trainer pastes code → we normalize it (strip variable names, strip comments) → fetch up to 2000 submissions from the database → check Redis for a cached TF-IDF matrix → if miss, build matrix and cache for 5 minutes with an MD5 cache key → transform the query into the same vector space → cosine similarity → return top 15. Two optimizations: normalized_code is pre-stored at submission time so we skip normalization at search time; the TF-IDF matrix is cached with a key that invalidates when new submissions arrive. At 2000 submissions, this runs in ~50ms. I'd switch to CodeBERT embeddings and pgvector at 100k+ because TF-IDF IDF scores shift as the corpus grows — stored vectors from Day 1 would be stale.

---

### "How would you scale this to 10 million users?"

The current bottleneck at scale is the Celery worker pool and Docker daemon. Migration path: (1) Replace Docker with AWS Lambda for execution — cold start acceptable at that scale with provisioned concurrency. (2) Replace Redis Celery broker with SQS for durability and auto-scaling. (3) Replace SSE with API Gateway WebSockets — SSE over HTTP/1.1 has a 6-connection-per-domain browser limit. (4) Replace the TF-IDF similarity search with CodeBERT embeddings stored at submission time and pgvector/FAISS for nearest-neighbour search. (5) Add read replicas for PostgreSQL — the dashboard queries are read-heavy. (6) Horizontally scale Django behind a load balancer. The plagiarism system needs a distributed TF-IDF or a switch to pre-computed embeddings — the current in-memory approach doesn't distribute.

---

### "What would you do differently?"

k6 load tests from day one. The 2000 concurrent user validation was real traffic but not instrumented — I didn't have p95 latency SLAs or automated regression detection. I'd also fix the N+1 in the dashboard view immediately (it loops over all students), add circuit breakers on the Docker executor, and rename the `snapshot_score` field to `tab_score` before it becomes embedded in client code. On the plagiarism side: the JavaScript normalizer uses regex instead of AST — it's inaccurate for complex syntax. A proper JS AST normalizer is the right fix.

---

### "How do you handle audit trails for algorithm changes?"

The plagiarism policy is configurable — trainers can adjust signal weights and thresholds via sliders. Every change creates a `PolicyVersion` record with a full JSON snapshot of the state before and after. When a trainer disputes a flag from two weeks ago, I can query the changelog, find which policy version was active at that time, and show exactly what threshold was used. It's the same idea as Entrupy versioning confidence thresholds for different product categories — old authentications should be re-evaluatable under the policy that was live at the time, not the current one.

The affected-exam detection is a temporal overlap query: `Exam.filter(start_time__lt=period_end, end_time__gt=period_start)`. This tells the trainer which exams ran under each policy version — useful for understanding whether a threshold change improved precision.

---

### "How do you investigate cohort-level patterns, not just individual flags?"

The annotation queue shows individual flags in isolation, which is fine for reviewing one flag but useless for asking "why does this exam have a 30% flag rate?" The cohort investigation view aggregates by exam, ranks exams by flag rate with health indicators (green/amber/red), and lets the trainer drill into a specific cohort.

The most valuable piece is cheating ring detection. I build an adjacency graph from similarity flags — if A matched B and B matched C, all three are in one ring even if A and C never directly matched. I run connected-component BFS and surface these clusters rather than making the trainer discover the network manually from the flat flag queue. A ring of 5 students linked by 4 similarity flags is a completely different investigation than 4 isolated pairs.

---

### "How does this relate to what Entrupy does?"

Entrupy's core loop: receive item → run ML inference → return authentication result. CodeMas's core loop: receive code → run execution → return pass/fail. Same architectural pattern — async job queue absorbs the spike, inference runs in an isolated container, result delivered via event stream. The plagiarism system maps directly to image similarity: I'm doing TF-IDF + cosine on code tokens; Entrupy does CNN embeddings + cosine on image feature vectors. The threshold, the pre-computation optimization, the human-in-the-loop annotation workflow — structurally identical problems. The scale difference (2k submissions vs 25M images) is why I'd need pgvector/FAISS at Entrupy's scale, and I've thought through that migration path.

---

*This document covers the complete system design of CodeMas as of May 2026. Every decision described here has corresponding code in the repository. Latest additions: Policy Versioning and Changelog (§8.1), Cohort Investigation and Cheating Ring Detection (§8.2).*
