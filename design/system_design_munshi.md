# Munshi — System Design

---

## Quick Reference

| Field | Value |
|---|---|
| Platform | Sovereign local-first AI agent |
| Role | GST reconciliation + trade compliance assistant |
| Client | Bharatvarsh Arts — ₹5Cr (~$600K) annual revenue |
| Stack | Hermes runtime, MCP tools, FastAPI, Ollama, Python Decimal, Docker, SQLite |
| Sovereignty promise | Nothing leaves the machine. Not a preference — a hard architectural constraint. |
| Model | Qwen / Llama via Ollama (local GPU/CPU inference) |
| Language | Python |
| Data store | SQLite (verdicts, audit trail, persistent memory) |

---

## Problem Statement

Bharatvarsh Arts — an arts-and-crafts business with ₹5Cr annual revenue — processes hundreds of purchase invoices monthly. These must be reconciled against GSTR-2B (the government's supplier filing portal) to claim Input Tax Credit (ITC). Historically this was an entirely manual process: Excel spreadsheets, human eyes, and error-prone arithmetic. Four hard constraints ruled out every existing SaaS solution:

1. **Financial data is sensitive.** Sending invoices, amounts, or GST details to any cloud API is a non-starter for the owner.
2. **Tax computation must be exact.** Floating-point arithmetic accumulates errors at scale; even a paisa-level error per invoice causes filing mismatches with the GSTN portal.
3. **Non-technical owner must verify every decision.** The system must be explainable in plain English, not model internals.
4. **The agent cannot file independently.** Every consequential action — claiming ITC, reconciling a verdict, importing an order — must pause for human approval.

Munshi solves all four with a local-first AI agent that reasons in plain English, matches invoices using deterministic rules where possible, escalates ambiguous cases to a local model for judgment, and never takes irreversible action without the owner's explicit sign-off.

---

## Functional Requirements

- **Plain-English queries:** "How much ITC can I claim this month?" → agent reads invoices and GSTR-2B, matches, computes, and presents a verified total.
- **Fuzzy GST reconciliation:** Match purchase invoices against GSTR-2B records despite vendor name inconsistencies, minor amount differences, and date variations.
- **Exact tax computation:** CGST, SGST, IGST per invoice, aggregated correctly — no float errors.
- **Cross-border landed cost with currency auto-detection:** When the owner asks for a quote for a foreign destination (Germany, USA, UAE), automatically convert to the customer's local currency. Bill in EUR, USD, AED — without being asked. Show INR equivalent for owner's records.
- **Audit trail:** Every tool call, model judgment, and computation logged in plain English. Non-technical owner can read and verify the entire chain of reasoning.
- **Human-in-the-loop (HITL):** Agent pauses and presents a computed summary before any consequential action. Owner approves or rejects; feedback is logged.
- **Persistent memory:** Approved vendor name resolutions and classifications carry forward month-to-month. Same judgment is not repeated.
- **Eval harness:** A GEPA-style measurement loop to benchmark prompt variants against labeled test cases.

---

## Non-Functional Requirements

| Requirement | Target |
|---|---|
| Data sovereignty | Zero bytes over the network — all inference, storage, and computation on the owner's machine |
| Correctness | Tax arithmetic exact to the paisa — verified against GSTN portal computations |
| Explainability | Every decision auditable in plain English by a non-technical user |
| Reliability | Model failures surface clearly — no silent wrong results |
| Latency | Acceptable for an interactive assistant; not a real-time API |
| Privacy-by-architecture | No encryption policy needed — data never leaves the machine |
| Overclaiming constraint | Agent must never offer capabilities it cannot actually fulfill |

---

## High-Level Architecture

```
Owner (plain-English query)
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Hermes Runtime (local)                                      │
│                                                              │
│  SOUL.md declares identity, constraints, operating limits    │
│                                                              │
│  Agent loop:  Reason → Select Tool → Call → Observe → Repeat│
└──────────────────────────┬──────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │     MCP Tool Layer      │
              │                         │
              ├── fetch_gstr2b()        │  reads GSTR-2B JSON (local file)
              ├── read_invoices()       │  reads purchase invoice CSV/JSON
              ├── fuzzy_match()         │  4-signal deterministic pipeline
              ├── compute_tax()         │  Python Decimal arithmetic
              ├── convert_currency()    │  local rate lookup + Decimal
              ├── write_audit()         │  appends to audit trail (SQLite)
              └── request_approval()   │  pauses loop, presents to owner
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │  Ollama (local model)   │
              │  Qwen / Llama           │
              │  Runs on local GPU/CPU  │
              │  No data transmitted    │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │   Human Approval Gate   │
              │   Owner reviews:        │
              │   - Audit trail         │
              │   - Model reasoning     │
              │   - Computed amounts    │
              │   Approves or rejects   │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │  Consequential Action   │
              │  executes (or rejected) │
              │                         │
              │  SQLite (local)         │
              │  - Verdicts             │
              │  - Approved matches     │
              │  - Audit log            │
              │  - Persistent memory    │
              └─────────────────────────┘

Everything runs on the owner's machine. Ollama handles inference locally.
```

---

## Detailed Component Design

### 1. Hermes Runtime + SOUL.md

Hermes (Nous Research, MIT licensed) manages the agentic loop. It provides tool registration, typed MCP tool calls, memory management, and the reason-act-observe loop — out of the box, without custom plumbing.

**The loop:**
1. **Reason:** Model reads conversation history + tool results, decides next action
2. **Act:** Calls the selected MCP tool with typed, schema-validated arguments
3. **Observe:** Tool result appended to context window
4. **Repeat:** Until task is complete or human approval is required

**Why Hermes over a custom agent loop:**
- MCP protocol enforces typed schemas — reduces model hallucinating tool arguments
- Tool registration is declarative, not imperative
- Memory and loop management handled by the runtime, not hand-rolled code

**SOUL.md — the agent's operating charter:**

SOUL.md is a plain-text file loaded at startup that declares the agent's identity, constraints, and operating limits. Key clauses:

- *"You decide and phrase; deterministic tools compute. NEVER invent a number — not a duty rate, tax amount, HS/HSN code, or any rupee/euro figure."* The model reasons and coordinates; it does not perform arithmetic.

- *"Prepare, don't commit. Anything consequential — issuing an invoice, importing orders, filing — you PREPARE and present an exact, computed summary for a human to approve."* The agent is a preparation system, not an execution system.

- *"Only offer what you can actually do. Do NOT offer to check FTA/preferential rates, screen sanctions, or look up anything you have no tool for — say so plainly. An aspirational offer you can't fulfil is overclaiming, and overclaiming is the one thing you never do."* Scope is hard-bounded by available tools.

- *"Sovereignty: You run entirely on infrastructure the owner controls. Nothing leaves the machine. That is a promise, not a preference."* This is an architectural fact, not a policy.

---

### 2. Fuzzy Invoice Matching — The 4-Signal Pipeline

The core reconciliation problem: purchase invoices use vendor names like "TATA STEEL LTD" — GSTR-2B has "TATA STEEL LIMITED". Exact string matching fails on every case like this. ITC is forfeited on every failed match. The fuzzy matcher closes this gap.

**Pipeline (per invoice):**

```
For each purchase invoice:

  Step 1 — Normalise vendor name:
    lowercase → strip punctuation → expand common abbreviations
    ("LTD" → "LIMITED", "PVT" → "PRIVATE", "CO" → "COMPANY")
    → compute Levenshtein similarity against normalised GSTR-2B name

  Step 2 — GSTIN prefix match:
    Compare first 10 characters of 15-character GSTIN
    (first 10 = state code + PAN-equivalent; last 5 = entity suffix)
    → exact match or no-match

  Step 3 — Amount tolerance check:
    | invoice_amount - gstr2b_amount | / gstr2b_amount ≤ 2%
    → within tolerance or out-of-tolerance

  Step 4 — Date proximity check:
    | invoice_date - gstr2b_date | ≤ 30 days
    → within proximity or out-of-proximity

Decision:
  All 4 signals agree  →  AUTO_MATCH    (no model call)
  2-3 signals agree    →  AMBIGUOUS     →  escalate to model
  < 2 signals agree    →  NO_MATCH      (no model call)
```

**Signal rationale:**
- GSTIN prefix is the strongest signal — it encodes state and taxpayer entity. An exact prefix match with a small name variation is almost always the same vendor.
- Name normalisation catches the "Ltd / Limited / Pvt Ltd" class of mismatches — the most common failure mode.
- Amount ±2% catches rounding differences between invoice and GSTR-2B representation.
- Date proximity catches month-boundary filing differences (invoice dated Jan 31, GSTR-2B reflects Feb 1).

**Model role — ambiguous cases only:**

When the pipeline returns AMBIGUOUS, the model receives the normalised pair and reasons in plain English:

```
Invoice:   TATA STEEL LTD, ₹47,500.00, GSTIN 27AAACT2727Q1ZW, 2026-06-30
GSTR-2B:   TATA STEEL LIMITED, ₹47,500.00, GSTIN 27AAACT2727Q1ZW, 2026-07-02

Model reasoning: "GSTIN prefix matches exactly (27AAACT2727Q). 
Name difference is a standard abbreviation ('LTD' vs 'LIMITED'). 
Amount is identical. Date is 2 days apart — consistent with month-boundary 
filing lag. Recommend: MATCH (confidence HIGH)."
```

The model never computes tax. It only judges whether two records refer to the same supplier, and provides a confidence level and reasoning chain that becomes part of the audit trail.

**Why model only for ambiguous cases:**
- ~80% of invoices auto-match deterministically — zero LLM inference cost and zero latency for the majority case
- Model judgment is reserved for cases where human intuition is genuinely required
- Keeps the system auditable: deterministic logic is unambiguous, model judgment is explicitly flagged

---

### 3. Tax Computation — Python Decimal

All tax arithmetic uses Python's `Decimal` module. Floats are categorically prohibited in financial computation.

**Why Decimal matters at ₹5Cr scale:**

```python
# Float (WRONG) — binary representation error
>>> 333.33 + 333.33 + 333.34
999.9999999999999          # not 1000.00

# Decimal (CORRECT) — exact base-10 arithmetic
>>> from decimal import Decimal
>>> Decimal('333.33') + Decimal('333.33') + Decimal('333.34')
Decimal('1000.00')
```

At 500 invoices/month, float errors compound. A ₹0.01 error per invoice = ₹5 per month. Small individually, but:
1. GSTN portal uses exact arithmetic. A mismatch — even one paisa — triggers a reconciliation failure.
2. An ITC overclaim, even by ₹1, is a compliance issue.
3. Errors compound across months and across line items within a single invoice.

**Computation pattern:**

```python
from decimal import Decimal, ROUND_HALF_UP

def compute_itc(invoice: dict) -> dict:
    taxable_value = Decimal(str(invoice['taxable_value']))
    
    # GST rates are percentages — always convert via Decimal
    igst_rate  = Decimal(str(invoice['igst_rate']))  / Decimal('100')
    cgst_rate  = Decimal(str(invoice['cgst_rate']))  / Decimal('100')
    sgst_rate  = Decimal(str(invoice['sgst_rate']))  / Decimal('100')
    
    # Quantize each component to paise (2 decimal places)
    igst = (taxable_value * igst_rate).quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)
    cgst = (taxable_value * cgst_rate).quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)
    sgst = (taxable_value * sgst_rate).quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)
    
    total_itc = igst + cgst + sgst   # Decimal + Decimal — still exact
    
    return {
        'taxable_value': taxable_value,
        'igst': igst,
        'cgst': cgst,
        'sgst': sgst,
        'total_itc': total_itc
    }
```

**Aggregation pattern:**

```python
from decimal import Decimal

def aggregate_itc(invoices: list) -> Decimal:
    # Use Decimal accumulator — never sum floats and convert at the end
    total = Decimal('0.00')
    for inv in invoices:
        total += inv['total_itc']   # both are Decimal — addition is exact
    return total
```

**The key invariant:** Numbers enter the system as strings (from CSV/JSON). They are immediately wrapped in `Decimal(str(...))`. They never touch float at any point. All intermediate results are quantized to 2 decimal places before the next operation.

---

### 4. Human-in-the-Loop (HITL)

The agent's operating principle is: **prepare, don't commit**. Every consequential action is assembled into a full, computed summary and held for owner approval. The agent cannot unilaterally act.

**Actions requiring approval:**
- Accepting a reconciliation verdict (MATCH / NO_MATCH / AMBIGUOUS → verdict)
- ITC claim total for a filing period
- Any invoice import, order entry, or filing action

**Approval flow:**

```
1. Agent calls request_approval(action, justification, computation_detail)
2. Agent loop pauses — no further tool calls or actions
3. UI surfaces the pending action with:
   - The proposed action in plain English
   - The full audit trail leading to this decision
   - Model reasoning (if the model was involved)
   - Exact computed amounts with Decimal provenance
4. Owner reviews and makes one of two choices:
   a. APPROVE → action executes, verdict written to SQLite, audit log updated
   b. REJECT  → action cancelled, owner's feedback logged, agent re-plans
5. Approved classifications persist in SQLite (don't ask again next month)
```

**Audit trail format (plain English, append-only):**

```
[2026-07-15 14:32:01] TOOL CALL: fuzzy_match()
  Input:
    Invoice:   TATA STEEL LTD, ₹47,500.00, GSTIN 27AAACT2727Q1ZW, 2026-06-30
    Candidate: TATA STEEL LIMITED, ₹47,500.00, GSTIN 27AAACT2727Q1ZW, 2026-07-02
  Signals:
    GSTIN prefix:  MATCH (27AAACT2727Q matches 27AAACT2727Q)
    Name:          MATCH after normalisation (edit distance 3)
    Amount:        MATCH (identical)
    Date:          MATCH (2-day gap, within 30-day window)
  Pipeline result: AUTO_MATCH (all 4 signals)
  No model call required.

[2026-07-15 14:32:02] TOOL CALL: compute_tax()
  taxable_value: Decimal('47500.00')
  igst_rate:     18%
  igst:          Decimal('47500.00') × Decimal('0.18') = Decimal('8550.00')
  ITC claimable: ₹8,550.00

[2026-07-15 14:32:03] APPROVAL REQUESTED
  Action: Accept MATCH verdict for TATA STEEL LTD → TATA STEEL LIMITED
  ITC consequence: ₹8,550.00 claimable IGST
  Awaiting owner approval.

[2026-07-15 14:34:18] OWNER APPROVED
  Verdict persisted to SQLite. Memory updated: "TATA STEEL LTD" resolves to
  "TATA STEEL LIMITED" (GSTIN 27AAACT2727Q1ZW) — will auto-match next month.
```

---

### 5. Currency Auto-Detection

Munshi handles cross-border sales for Bharatvarsh Arts. When the owner asks for a landed cost or quote for a foreign destination, the agent bills in the customer's local currency automatically — without being asked.

**Inference rule (from SOUL.md):**
- Destination country → currency is inferred deterministically from a country-to-currency lookup
- Germany → EUR, USA → USD, UAE → AED, domestic India → INR (no conversion)
- The agent leads with the foreign-currency figure; includes INR equivalent for the owner's records
- If the destination is ambiguous (e.g. "Europe" without a specific country), the agent asks for clarification — it does not guess

**Conversion flow:**

```
Owner: "What's the landed cost for a shipment to Germany — ₹85,000 order value?"

Agent steps:
  1. Identify destination: Germany → infer currency: EUR
  2. Call convert_currency(amount=Decimal('85000.00'), from_currency='INR', to_currency='EUR')
     → uses local exchange rate lookup (not an external API call)
     → rate: Decimal('0.01095')
     → EUR amount: (Decimal('85000.00') × Decimal('0.01095')).quantize(Decimal('0.01'))
                 = Decimal('930.75')
  3. Add landed cost components (customs duty, shipping, insurance) in EUR — all Decimal
  4. Present to owner:
     "Total landed cost: EUR 1,120.50 (₹1,02,283 at current rate)
      Breakdown: Order value EUR 930.75 + Customs EUR 139.25 + Shipping EUR 50.50"

Currency auto-detection: No prompt asking "which currency?" for a German customer.
```

**Handling ambiguity:**
- Unambiguous country → currency is inferred silently
- Ambiguous destination ("Middle East", "Europe") → agent asks specifically: "Which country? I'll use the correct currency automatically."
- Currency explicitly stated by owner → agent respects it, skips inference
- The agent never uses float for currency conversion — all rates and amounts are Decimal throughout

**SOUL.md constraint:** If Munshi has no tool to fetch live exchange rates and the local rate file is stale, it says so plainly rather than using a potentially outdated figure. "My rate data is from [date]. Please confirm before quoting."

---

### 6. Eval Harness — GEPA Measurement Half

Munshi includes a GEPA-style evaluation harness in `backend/app/evals/harness.py`. This is the **measurement half** of self-improvement — it benchmarks prompt variants against labeled test cases and ranks them by accuracy. The full GEPA loop (reflection, proposal, and automated approval) is not yet built.

**What the harness does:**

The harness defines two sets of labeled test cases:

```python
EASY_CASES = [
    # 9 labeled cases — title, expected HSN code
    # These should be solvable by deterministic rules alone
    {"input": "brass handicraft figurine", "expected_hsn": "83062910"},
    {"input": "wooden photo frame",        "expected_hsn": "44190000"},
    # ... 7 more
]

HARD_CASES = [
    # 2 labeled cases — no catalog keyword; requires model reasoning or memory
    {"input": "decorative wall panel with mixed materials", "expected_hsn": "83062990"},
    {"input": "artisan textile wall hanging",              "expected_hsn": "63019090"},
]
```

**run_eval() signature:**

```python
def run_eval(cases: list[dict]) -> dict:
    """
    Run the agent against labeled cases.
    Returns: {total, correct, accuracy, seconds, rows}

    This is a GEPA-style loop we built, NOT the GEPA algorithm itself.
    It is the measurement half (eval + variant scoring).
    The reflection/proposal/approval loop is not built yet.
    """
    results = []
    start = time.time()
    for case in cases:
        prediction = agent.run(case['input'])       # runs the full agent loop
        correct = prediction.hsn == case['expected_hsn']
        results.append({'case': case, 'prediction': prediction, 'correct': correct})
    elapsed = time.time() - start
    correct_count = sum(r['correct'] for r in results)
    return {
        'total':    len(cases),
        'correct':  correct_count,
        'accuracy': correct_count / len(cases),
        'seconds':  elapsed,
        'rows':     results
    }
```

**score_variants() — prompt ranking:**

```python
def score_variants(variants: list[str], cases: list[dict]) -> list[dict]:
    """
    Run each prompt variant through run_eval and rank by accuracy.
    Returns variants sorted by accuracy descending.
    """
    scores = []
    for variant in variants:
        agent.set_system_prompt(variant)
        result = run_eval(cases)
        scores.append({'variant': variant, 'accuracy': result['accuracy']})
    return sorted(scores, key=lambda x: x['accuracy'], reverse=True)
```

**EASY vs HARD split rationale:**
- EASY cases (9): test deterministic coverage — if the HSN code lookup or catalog match works, these should all pass
- HARD cases (2): test genuine model reasoning or memory — no shortcut path; require the agent to use context, prior verdicts, or general trade knowledge

**What's NOT in the harness:**
- Automatic prompt reflection (GEPA step: "why did this variant fail?")
- Automated prompt proposal generation
- Approval loop for new prompt adoption

These are planned future work. The harness gives the measurement infrastructure; the self-improvement loop would sit on top.

---

### 7. Persistent Memory

Hermes maintains memory across sessions via SQLite. The key design decision: **raw financial data is not persisted**. Only classified verdicts and approved resolutions are stored.

**What is persisted (SQLite):**

| Table | Contents | Purpose |
|---|---|---|
| `vendor_resolutions` | `(raw_name, canonical_name, gstin, approved_by, approved_at)` | Don't re-ask: "TATA STEEL LTD" → "TATA STEEL LIMITED" |
| `match_verdicts` | `(invoice_id, gstr2b_id, verdict, confidence, model_reasoning)` | Audit trail of all reconciliation decisions |
| `audit_log` | Append-only plain-English log | Human-readable history of every tool call and decision |
| `approved_actions` | `(action_type, action_detail, owner_approved_at)` | Record of what the owner has explicitly signed off on |

**What is NOT persisted in the agent's memory:**
- Raw invoice data (re-read from source CSVs each session)
- Raw GSTR-2B records (re-read from government JSON each session)
- Unapproved model judgments (rejected reasoning is logged but not acted on)

**Why this split:** Financial records are source-of-truth documents that should not be duplicated in the agent's memory. The agent's memory is for *judgments* about those records — specifically, judgments the owner has approved. This keeps the memory small, auditable, and free of stale financial data.

---

### 8. Data Flow & Storage

```
Input sources (local filesystem, read-only):
  /data/invoices/2026-07/    — purchase invoice CSVs
  /data/gstr2b/2026-07/      — GSTR-2B JSON downloads

Agent processing (in-memory, local):
  Hermes runtime context window
  Python Decimal arithmetic (no persistence of intermediate values)
  Ollama model inference (local GPU/CPU)

Output storage (local SQLite, write):
  /db/munshi.sqlite
    ├── vendor_resolutions   (approved name mappings)
    ├── match_verdicts       (reconciliation results)
    ├── audit_log            (append-only plain-English trail)
    └── approved_actions     (HITL approvals)

Network:
  NONE. Zero bytes transmitted.
```

**Storage choices:**
- **SQLite** for all persistent state: zero server process, zero configuration, fully embedded in the application, sufficient for a single-business workload at this scale
- **Append-only audit log**: never delete or update log entries — compliance requirement. New verdicts write new rows; corrections are noted as new entries referencing the original
- **Local filesystem for source data**: invoices and GSTR-2B stay in their original format, owned by the owner, not duplicated into a database the agent manages

---

### 9. Key Engineering Decisions

| Decision | Choice | Alternative | Why |
|---|---|---|---|
| Architecture | Local-first, Ollama | Cloud API (GPT-4, Gemini) | Data sovereignty — financial data never leaves the machine. Privacy-by-architecture, not policy. |
| Tax arithmetic | Python Decimal | float | Exact base-10 computation; no accumulation errors; matches GSTN portal's arithmetic |
| Model role | Judge ambiguous matches only | Compute tax, perform all matching | Deterministic rules handle ~80% of cases; model reserved for genuine judgment calls |
| HITL timing | Before consequential action | Post-hoc review | Owner verifies before irreversible action, not after — no rollback problem |
| Agent runtime | Hermes (Nous Research) | Custom agent loop | Zero plumbing; typed MCP tool calls reduce hallucinated arguments |
| Memory scope | SQLite verdicts only | Full conversation history | Raw financial data not persisted; only approved classifications |
| Matching strategy | 4-signal fuzzy pipeline | Exact string match | Vendor name inconsistencies are the actual problem; exact match solves nothing |
| Currency handling | Auto-detect from destination | Always ask | Owner should not have to specify EUR for a German customer every time |
| Overclaiming constraint | SOUL.md hard-bound to available tools | Soft guidelines | Agent must never offer a capability it cannot fulfill; aspirational offers are a form of hallucination |
| Eval harness | GEPA-style measurement loop | Manual testing | Structured benchmarking catches prompt regressions; builds toward self-improvement |

---

### 10. Framework Decision — Hermes vs LangChain vs LangGraph

This is one of the most common interview questions on any agent project: "Why not LangChain?" or "Would you use LangGraph here?" Knowing the answer signals that you understand the tool landscape, not just the one tool you happened to use.

---

#### The Three Tools at a Glance

| | Hermes | LangChain | LangGraph |
|---|---|---|---|
| **What it is** | Lightweight agentic runtime (Nous Research, MIT) | LLM framework + agent abstractions + integrations | Graph-based workflow engine (built on LangChain) |
| **Primary pattern** | ReAct loop: Reason → Tool → Observe → Repeat | AgentExecutor / LCEL chains | StateGraph: fixed nodes + edges + state |
| **Best for** | Open-ended tasks where the AI decides its own tool path | Rapid prototyping with many integrations | Known multi-step workflows with HITL and checkpointing |
| **Tool protocol** | Native MCP — typed JSON Schema validation before execution | Function calling — schema-based but optional enforcement | Inherits LangChain tools |
| **HITL support** | Custom — implement `request_approval()` as a tool | Custom | Built-in `interrupt()` / resume via thread_id |
| **Persistence** | Bring your own (SQLite, Redis, etc.) | Bring your own | MemorySaver (in-memory) or DB checkpointer (Postgres) |
| **Statefulness** | Context window + external memory | Context window + external memory | Full state machine — state is typed and checkpointed per node |
| **Breaking changes** | Low | High — frequently between versions | Medium |
| **Ecosystem size** | Small and focused | Very large (200+ integrations) | LangChain ecosystem |
| **Overhead** | Low — minimal abstractions | Medium — many layers | Medium — graph setup cost |
| **Schema validation** | Native and enforced (MCP) | Optional | Optional |

---

#### When Each Tool Wins

**Use Hermes when:**
- The task is open-ended — the model must decide which tools to call and in what order
- You need typed, schema-validated MCP tool calls (important where wrong arguments have real consequences)
- You want minimal framework overhead and a focused runtime
- Local-first deployment matters (Hermes works with Ollama natively)

**Use LangChain when:**
- You need many integrations quickly (10+ data sources, many model providers, many vector stores)
- Rapid prototyping where iteration speed matters more than strict typing
- The ecosystem's pre-built chains (PDF loader, text splitter, retriever chains) save significant effort
- You're okay managing frequent API changes between versions

**Use LangGraph when:**
- The steps are known before execution starts — the workflow is a fixed graph, not a dynamic choice
- HITL is a first-class requirement — you need suspend/resume across HTTP requests or time boundaries
- Long-running tasks must survive server restarts (DB checkpointer)
- You need parallel fan-out — multiple sub-agents running concurrently with a barrier sync
- The 01dev QuizMe pattern: `plan_objectives → human approves → generate_quiz → student submits → evaluate`

---

#### For Munshi Specifically — Why Hermes and Not the Others

Munshi's task is fundamentally **open-ended**: the owner asks an arbitrary question in plain English. The agent decides which tools to call and in what order, based on what the query requires. "How much ITC can I claim?" takes a different tool path than "What's the landed cost for a shipment to Germany?" The model chooses dynamically — this is an agent task, not a workflow task.

**Why not LangGraph:** LangGraph is designed for known step sequences with fixed edges between nodes. Munshi's `request_approval()` gate can appear at different points in the tool chain depending on the query. The approval position is dynamic. If you put it in a fixed graph position, you either gate too early (before the agent has all the information) or too late (after unnecessary work). The open-ended nature of the tool-calling path is exactly what LangGraph was not designed for.

**Why not LangChain AgentExecutor:** This would have been a reasonable alternative. LangChain's AgentExecutor runs a ReAct-style loop and could handle Munshi's tool calls. The deciding factor was MCP tool schema validation — in a financial system, a hallucinated tool argument (wrong tax rate, wrong GSTIN prefix) has real compliance consequences. Hermes enforces the JSON Schema before the function executes and returns a structured error the model can read and correct. LangChain's function calling has schema support but it's not as tightly enforced at the runtime layer.

**The financial context matters:** In a system where a wrong number can cause a GST filing error, typed tool interfaces aren't nice-to-have — they're a correctness requirement. MCP enforcement is architectural protection, not just good practice.

---

#### The General Decision Rule

```
Is the task open-ended — does the AI decide its own next step?
    │
    YES → Agent loop (Hermes or LangChain AgentExecutor)
    │        │
    │        └── Need typed MCP enforcement? → Hermes
    │            Lots of integrations needed? → LangChain
    │
    NO → Workflow (LangGraph)
         │
         └── Need HITL suspend/resume?      → LangGraph interrupt
             Need to survive server restarts? → LangGraph + DB checkpointer
             Parallel sub-agents?           → LangGraph fan-out
```

**The common mistake:** Reaching for LangGraph because it's popular, then discovering the workflow doesn't have fixed steps and fighting the graph structure. Or reaching for a full agent framework when the steps are fully known and a simple workflow would be more predictable and cheaper to operate.

**Interview move:** "I chose Hermes because Munshi's task is open-ended — the model decides its tool path dynamically, and MCP's typed schema validation is important in a financial system where wrong arguments have real consequences. If Munshi had a fixed workflow — say, always: fetch → match → compute → present — I would have used LangGraph for the built-in HITL and checkpointing. The tool should match the problem shape."

---

## 11. Data Engineering Layer — Analytics Platform

*Added: 2026-08-12 to 2026-08-13. A full ELT data platform built on top of the agent, serving clean governed analytics data back to the agent so it can answer business questions ("total revenue this month?", "which orders silently came back?") from tested, audited data rather than raw exports.*

---

### 11.1 Why a Data Layer?

The GST/invoice side (Sections 1–10) is sovereign and local — everything stays on the owner's machine. The analytics side addresses a different problem: the owner couldn't answer basic business questions because her data lived in **five disconnected systems** and she was the only human connecting them manually.

Three problems came out of discovery:
1. **Broken order truth (B2C):** An order RTO'd at the courier, payment already collected, but Shopify still shows it as fulfilled. Nobody flags it automatically.
2. **B2B has no source system:** Bulk wholesale orders and price negotiations happen on WhatsApp in Hinglish/Marathi — overwritten in Excel, history lost, disputes unanswerable.
3. **No unified revenue view** across B2C + B2B channels.

---

### 11.2 Data Sources — Source-of-Truth Mapping

*First step before writing any code: decide who owns each fact.*

| System | Owns (source of truth for) |
|---|---|
| **Shopify** | B2C order details, products, customers, cancellations, refunds |
| **GoKwik** | Prepaid payment confirmation, COD/RTO risk score |
| **NimbusPost** | Shipment status, delivery/RTO outcomes, COD remittance |
| **WhatsApp** | B2B orders + negotiated contract prices (no real system exists) |

**Why source-of-truth mapping matters:** All three B2C systems share no common identifier. GoKwik uses its own payment ID; NimbusPost uses the AWB number; Shopify uses the order name (`#14437`). The `xref_order_identity` mart is the identity bridge that resolves this.

**Data-quality issues discovered from profiling the real exports:**
- Naive line counts lied: 28k raw lines vs 3,977 real records — embedded newlines in quoted CSV fields
- Three different date formats: ISO 8601 (Shopify), DD-MM-YYYY (NimbusPost), M/D/YYYY H (GoKwik)
- Casing drift: `COD` vs `cod`, `Paid` vs `paid`
- Grain mismatch: Shopify at line-item grain, NimbusPost at shipment grain, GoKwik at order grain
- Re-ships: an RTO'd order gets a new NimbusPost shipment with `-Copy` appended; both rows must be preserved

---

### 11.3 Full Stack Architecture

```
 ┌─────────────────── SOURCE SYSTEMS ────────────────────────┐
 │  Shopify CSV      GoKwik CSV      NimbusPost CSV           │
 │  (line-item)      (order grain)   (shipment grain)         │
 │                                                            │
 │  WhatsApp         ──LLM extraction──► B2B_EXTRACTED        │
 │  (free-text msg)  (gpt-mini batch)                         │
 └──────────────────┬─────────────────────────────────────────┘
                    │  Python ELT loaders (load_raw.py)
                    │  VARIANT JSON landing (schema-on-read)
                    ▼
 ┌─────────────────── SNOWFLAKE RAW ─────────────────────────┐
 │  SHOPIFY_ORDERS   GOKWIK_PAYMENTS   NIMBUSPOST_SHIPMENTS   │
 │  WHATSAPP_MESSAGES   B2B_EXTRACTED                         │
 │  (all as VARIANT — no brittle upfront DDL)                 │
 └──────────────────┬─────────────────────────────────────────┘
                    │  dbt staging (views — always fresh)
                    ▼
 ┌─────────────────── STAGING SCHEMA (views) ────────────────┐
 │  stg_shopify__orders         (2,109 orders; line-item→order)│
 │  stg_nimbuspost__shipments   (2,397 rows; date/casing clean)│
 │  stg_gokwik__payments        (2,106 rows; timestamp parse) │
 │  stg_whatsapp__b2b           (extracted B2B intents)       │
 │  int_nimbuspost_by_order     (shipment→order grain reduce) │
 └──────────────────┬─────────────────────────────────────────┘
                    │  dbt marts (tables — fast agent queries)
                    ▼
 ┌─────────────────── MARTS SCHEMA (tables) ─────────────────┐
 │  xref_order_identity   ← identity bridge (no shared ID)   │
 │  fct_order             ← Order-360 (all facts per order)  │
 │  b2c_exceptions        ← 247 actionable issues today      │
 │  dim_vendor_contract   ← SCD Type 2 B2B price history     │
 │  fct_b2b_order         ← structured B2B orders from chat  │
 │  fct_revenue           ← unified B2C+B2B revenue          │
 │  dim_date              ← conformed date dimension          │
 │                                                            │
 │  PII governance: Snowflake Dynamic Data Masking on         │
 │  customer_name / customer_email / customer_phone           │
 └──────────────────┬─────────────────────────────────────────┘
                    │  backend/app/warehouse/marts.py
                    │  AGENT_READER role (SELECT-only, MARTS only)
                    ▼
 ┌─────────────────── LLM AGENT TOOLS ───────────────────────┐
 │  order_status()         find_customer_orders()            │
 │  list_exceptions()      vendor_contract_price()           │
 │  exceptions_summary()   revenue_summary()                 │
 │  run_sql() ← guarded text-to-SQL (read-only, auto-LIMIT)  │
 └──────────────────┬─────────────────────────────────────────┘
                    │
                    ▼  Owner asks: "Which orders silently came back?"
                 Agent answers from tested, auditable mart data
```

---

### 11.4 ELT Pipeline — Python Loaders to Snowflake RAW

**Why VARIANT for raw landing:**
Rather than define brittle DDL for Shopify's 80-column export, every row lands as a single `VARIANT` (JSON) column. Staging models use `record:"Col Name"::type` for schema-on-read. This pattern handles messy/wide exports without any upfront schema negotiation — the standard Snowflake ELT landing approach.

**Loading flow:**
```python
# load_raw.py — idempotent TRUNCATE + batch insert
csv.DictReader  →  JSON-serialize each row
  →  TRUNCATE raw table
  →  batch INSERT (200 rows/batch) via PARSE_JSON()
```

**WhatsApp B2B extraction pipeline (unstructured → structured):**
```
RAW.WHATSAPP_MESSAGES (32 raw messages, mixed Hinglish/Marathi)
    │
    │  extract_b2b.py — LLM batch extraction (8 msgs/batch)
    │  gpt-mini: "Extract {intent, product, size, qty, agreed_price, confidence}"
    │  AI extracts meaning from text; it never calculates a number
    ▼
RAW.B2B_EXTRACTED (typed table — schema now known)
    │
    ▼
stg_whatsapp__b2b → fct_b2b_order → fct_revenue (B2B channel)
```

**"AI decides, deterministic code computes"** applies here too: the LLM reads a price from the text (`"₹180 per piece"`), never calculates one. `contract_line_value = qty × price` is computed in SQL.

---

### 11.5 dbt Model Lineage

**Staging layer (views — 1:1 with sources, cheap, always fresh):**

| Model | Key work done |
|---|---|
| `stg_shopify__orders` | Collapse line-item grain → order grain via `QUALIFY ROW_NUMBER()`; derive `is_cancelled`, coerce amounts via `TRY_TO_DECIMAL` |
| `stg_nimbuspost__shipments` | Parse DD-MM-YYYY dates; derive `is_rto`, `is_delivered`, `is_remitted`, `is_reship` flags |
| `stg_gokwik__payments` | Parse M/D/YYYY H timestamps; derive `is_paid`; keep `rto_risk` label + score |
| `stg_whatsapp__b2b` | Pass-through of `B2B_EXTRACTED`; rename `wa_id → vendor_id`, derive `msg_date` |
| `int_nimbuspost_by_order` | Window-function reduce: shipment grain → order grain; preserve `was_reshipped`, `ever_rto` history |

**Marts layer (tables — fast agent queries):**

| Mart | What it answers |
|---|---|
| `xref_order_identity` | Which orders appear in all three systems? Where are the gaps? `match_status`: all_three (1,987), partial, no_shopify (38) |
| `fct_order` | Order-360: every fact per order — totals, status, flags, AWB, tracking URL, customer identity (masked), `data_as_of` freshness |
| `b2c_exceptions` | **247 actionable issues today**: 155 RTO'd but Shopify shows fulfilled; 53 prepaid RTO with refund owed; 38 no-Shopify; 1 COD unremitted; 6 reship-watch |
| `dim_vendor_contract` | SCD Type 2 B2B price history: "what price was agreed with Vendor X on Date Y?" — built directly from WhatsApp effective dates, not dbt snapshots |
| `fct_b2b_order` | Structured B2B orders from WhatsApp chat, joined to valid contract price on order date |
| `fct_revenue` | Unified B2C + B2B: ₹15.4L all-time; July 2026: B2B (₹32k) outpaced B2C (₹30k) |
| `dim_date` | Conformed calendar; both channels roll up through the same date definitions |

---

### 11.6 PII Governance — Snowflake Dynamic Data Masking

**The tradeoff stated honestly:** customer identities (name, email, phone) live in Snowflake — protected by access control, not by never leaving the machine. This is the enterprise-standard approach that enables customer-level lookups. It deliberately departs from the sovereign/local-first story of the GST side.

**How masking works:**

```sql
-- Applied via dbt post_hook on fct_order after every dbt run:
CREATE OR REPLACE MASKING POLICY pii_mask AS (v STRING) RETURNS STRING →
  CASE WHEN CURRENT_ROLE() IN ('ACCOUNTADMIN','PII_READER','AGENT_READER_PII')
       THEN v
       ELSE '***MASKED***'
  END;

ALTER TABLE MARTS.FCT_ORDER MODIFY COLUMN customer_name
  SET MASKING POLICY pii_mask FORCE;
-- FORCE re-pins on every CREATE OR REPLACE TABLE (dbt rebuild drops column policies)
```

**Role access:**

| Role | Can see PII | Can query MARTS | Use case |
|---|---|---|---|
| `ACCOUNTADMIN` | Yes | Yes | Owner / DBA |
| `AGENT_READER_PII` | Yes | Yes | `find_customer_orders` tool |
| `AGENT_READER` | No — sees `***MASKED***` | Yes | `run_sql` text-to-SQL (ad-hoc analytics) |
| Any other role | No | No | Default |

**Why the text-to-SQL tool (`run_sql`) uses the masked role:** Ad-hoc analytics questions asked by the owner should never accidentally surface PII in the LLM's context. The authorized lookup tool (`find_customer_orders`) connects with the PII-allowed role explicitly when customer identity is the answer to the question.

---

### 11.7 Agent Analytics Tools

Seven functions in `backend/app/warehouse/marts.py`, all read-only, all returning exact Decimal strings (never floats):

| Tool | What it does |
|---|---|
| `order_status(order_number)` | Order-360 lookup in `fct_order`; normalizes order number format (`#14437`, `order 14437` → `14437`) |
| `list_exceptions(exception_type, limit)` | Query `b2c_exceptions` with optional type filter; returns order, type, reason, next action |
| `exceptions_summary()` | Count by exception type — "what needs me today" view |
| `revenue_summary(month)` | B2C + B2B from `fct_revenue`, grouped by channel, optional YYYY-MM filter |
| `vendor_contract_price(vendor, product, on_date)` | ILIKE fuzzy vendor match + SCD date range; the B2B dispute resolver |
| `find_customer_orders(query, limit)` | ILIKE on name + phone; authorized role sees real PII |
| `run_sql(sql, limit)` | Guarded text-to-SQL: single SELECT only, forbidden-keyword scan, auto-LIMIT, 30s timeout, `AGENT_READER` (PII masked), SQL echoed for audit |

**`run_sql` guardrails layered in order:**
1. Must be a single statement (no semicolons)
2. Must start with `SELECT` or `WITH`
3. Regex scan for 18 forbidden keywords (`INSERT`, `UPDATE`, `DELETE`, `DROP`, `CREATE`, `ALTER`, `GRANT`, `REVOKE`, `TRUNCATE`, `CALL`, `COPY`, `USE`, `PUT`, `REMOVE`, `UNLOAD`, `EXECUTE`, `BEGIN`, `COMMIT`)
4. Auto-appends `LIMIT` if absent
5. 30-second `statement_timeout` enforced at session level
6. Connects as `AGENT_READER` — SELECT-only on MARTS, PII masked

---

### 11.8 Data Engineering Key Decisions

| Decision | Choice | Alternative | Why |
|---|---|---|---|
| Warehouse | Snowflake | DuckDB, BigQuery, Redshift | Snowflake VARIANT for schema-on-read landing; Dynamic Data Masking native; deliberate cloud/production learning track |
| Operational store | SQLite (local) | Snowflake | GST/invoice data is sovereign — never leaves the machine; Snowflake is analytics-only |
| Raw landing | VARIANT (JSON) | Upfront DDL | 80-column Shopify exports change without warning; schema-on-read via `record:"Col"::type` in staging |
| Transformation | dbt | Custom SQL scripts | Layering (raw→staging→marts), testability, lineage, documentation; staging as views = always fresh |
| Materialization | Staging = views, Marts = tables | All tables or all views | Views cost nothing and stay fresh; marts are the query target — tables give fast agent response |
| SCD Type 2 | Built directly from effective-dated data | dbt snapshots | WhatsApp data already carries full history with dates; no need for snapshot mechanism |
| PII governance | Snowflake Dynamic Data Masking + RBAC | Hash at load | Masking enables customer lookups; hash-at-load is irreversible and breaks `find_customer_orders` |
| Orchestration | None (manual `dbt run` or cron) | Dagster, Airflow | One pipeline, one schedule, no inter-job dependencies — orchestration overhead ≫ benefit |
| Managed ingestion | None (custom Python loader) | Airbyte, Fivetran | Sources are CSV exports today; one-file change when APIs become available; right-sized to business scale |
| Exception pattern | Exception list only | Full dashboard | Owner needs to act on anomalies, not browse metrics; surfacing 247 issues from 2,100 orders > a chart of everything |

---

### 11.9 Technology Comparisons — Data Engineering

**Snowflake vs BigQuery vs Redshift vs DuckDB**

| | Snowflake | BigQuery | Redshift | DuckDB |
|---|---|---|---|---|
| **Pricing** | Per-second compute + storage | Per-query or per-slot | Per-node (reserved) | Free (open-source) |
| **VARIANT / semi-structured** | Native — `PARSE_JSON`, `record:"Col"`, `LATERAL FLATTEN` | `JSON_EXTRACT` | `SUPER` type | Native JSON |
| **Dynamic Data Masking** | Native — column-level masking policies | Column-level policies (limited) | Row-level security only | None |
| **Scale** | Petabytes | Petabytes | Petabytes | GBs (single machine) |
| **Setup** | Cloud SaaS — minutes | Cloud SaaS — minutes | Cluster provisioning | Embedded — zero setup |
| **For Bharatvarsh:** | Native VARIANT + DDM made it the right call | Would work; DDM less native | Overkill; cluster cost | Right for local/sovereign track; wrong for cloud analytics |

**For Bharatvarsh:** Snowflake's VARIANT column and native Dynamic Data Masking were load-bearing. VARIANT eliminated brittle DDL for 80-column exports. DDM enabled the governed agent query layer without rebuilding the data. **DuckDB would be the right choice for the fully sovereign, local-first track** (same SQL dialect, zero infra, embedded) — it's not used here because the analytics engagement deliberately chose the cloud/production learning track.

---

**dbt vs custom SQL scripts vs Pandas**

| | dbt | Custom SQL | Pandas |
|---|---|---|---|
| **Testing** | Built-in `not_null`, `unique`, `accepted_values`, custom SQL tests | Manual / none | Manual assertions |
| **Lineage** | Automatic DAG from `ref()` | None | None |
| **Documentation** | Auto-generated from YAML | None | None |
| **Idempotent** | Yes (CREATE OR REPLACE) | Manual TRUNCATE logic | Manual |
| **Best for** | SQL-native transformations with quality gates | One-off queries | Row-by-row Python logic |
| **For Bharatvarsh:** | 37 tests, clean lineage, documented marts | Would work at this scale but no safety net | Wrong tool — no SQL pushdown |

**Interview move:** "dbt gave me three things custom SQL scripts don't: layering (raw → staging → marts so logic is traceable), tests that run as part of the pipeline (37 tests, all green), and idempotent `CREATE OR REPLACE` so I can run it as many times as needed without side effects. For one pipeline at this scale, the overhead is near zero and the safety net is real."

---

## FAQ

**Q: Walk me through what happens when the owner asks "how much ITC can I claim this month?"**

A full agentic loop runs end-to-end. Hermes reasons that it needs GSTR-2B data and invoice data for the current period, so it calls `fetch_gstr2b(period='2026-07')` and `read_invoices(period='2026-07')`. With both datasets in context, it calls `fuzzy_match()` for each invoice. Invoices where all 4 signals agree auto-match immediately. Invoices with 2-3 agreeing signals are batched and sent to the local model for judgment. The model returns a verdict and reasoning for each. Invoices with fewer than 2 signals go to NO_MATCH. Once all verdicts are settled, `compute_tax()` runs on every matched invoice using Python Decimal. The results are aggregated. Finally, `request_approval()` is called with the full breakdown: total claimable ITC, per-invoice breakdown, model judgments for ambiguous cases, and a complete audit trail. The loop pauses. The owner reviews, and either approves (ITC total is final) or rejects specific items (those are reclassified and the total recomputed). Only after approval does the agent present the number as the owner's official ITC claim.

**Q: Why local-first? What would you lose by running this on a cloud API?**

You would lose the sovereignty promise — the core value proposition for this client. The owner of Bharatvarsh Arts will not transmit invoice data, GSTIN numbers, or tax amounts to any external service. That's not paranoia; it's a reasonable position for a business owner handling sensitive financial data. Running on a cloud API also means you need an internet connection, you're subject to the provider's data retention policies (even if they claim to delete), and you lose the ability to air-gap the system entirely. The trade-off is capability: a local Qwen or Llama model is less capable than GPT-4o. But the matching is mostly deterministic — the model only handles ambiguous cases — so the capability gap matters less than it would in a fully model-driven system. Privacy-by-architecture (data never moves) is stronger than privacy-by-policy (data moves but the provider promises not to misuse it).

**Q: How does the fuzzy matcher decide when a case is ambiguous enough to escalate to the model?**

The matcher runs all 4 signals (GSTIN prefix, normalised name similarity, amount within ±2%, date within 30 days) for each invoice-candidate pair. It counts how many signals agree. If all 4 agree, it's AUTO_MATCH — no model needed. If 1 or 0 agree, it's NO_MATCH — the model can't help here either, because there's not enough signal to work with. The interesting zone is 2-3 agreeing signals: for example, GSTIN prefix matches and amount matches, but the name is quite different and the date is 25 days apart. That's where human judgment — or model judgment standing in for human judgment — is genuinely needed. The threshold of "2-3 signals" was chosen to maximize the auto-match rate (and thus minimize model calls and latency) while still escalating cases that genuinely need reasoning.

**Q: The model judges ambiguous invoice matches. What happens if the model is wrong? How do you catch that?**

Three safeguards. First, every model judgment goes through the HITL approval gate — the owner sees the model's reasoning and the two records side by side before the verdict is accepted. A wrong judgment gets caught at review time. Second, rejected model judgments are logged with the owner's feedback, which can be used to improve the system prompt. Third, the eval harness lets you run a set of labeled test cases through the system and measure accuracy — if model accuracy degrades after a prompt change or model update, the harness catches it. The deeper risk is the model being confidently wrong on a case the owner doesn't scrutinize carefully. The mitigation is the audit trail: every model judgment is logged with its confidence level and reasoning. Over time, patterns of confident-but-wrong judgments become visible in the log and can be addressed with prompt changes or threshold adjustments.

**Q: Why Python Decimal and not float? Walk me through what breaks at ₹5Cr scale.**

Floats use binary (base-2) representation. Many base-10 fractions — including financial amounts like 333.33 — cannot be represented exactly in binary. You get rounding errors at the representation level, before any arithmetic happens. When you add 500 such amounts, the errors accumulate. At ₹5Cr annual revenue with ~500 invoices/month, the accumulated error across a filing period can easily reach a few rupees. That sounds small, but GSTN portal computations use exact decimal arithmetic. A filing that differs from the portal's computation by even ₹1 triggers a reconciliation mismatch — not a financial catastrophe, but a compliance failure that requires a corrected filing. Python Decimal solves this by using base-10 arithmetic with configurable precision. `Decimal('333.33')` is exactly 333.33. Addition is exact. The cost is performance (Decimal is slower than float), which is irrelevant at 500 invoices/month.

**Q: Your SOUL.md says "never invent a number." How is that enforced — the model could still do it.**

It's enforced architecturally, not just by instruction. The SOUL.md constraint tells the model to coordinate and reason, but all numeric outputs come from deterministic tools: `compute_tax()` performs the arithmetic, `convert_currency()` performs the conversion. The model's job in the tool-calling architecture is to select the right tool, pass the right arguments, and reason about the result — not to produce numbers itself. If the model tries to emit a tax amount in free text rather than calling `compute_tax()`, that text is not written to the audit log, not used in the ITC total, and not presented to the owner as a verified figure. The only numbers that propagate through the system are those returned by tool calls. The system prompt constraint reinforces this, but the real enforcement is that the agent loop only acts on tool results, not on model-generated prose. This is the advantage of the Hermes tool-use architecture over a plain conversational model.

**Q: How does the currency auto-detection work? What if the country's currency is ambiguous?**

Munshi maintains a local country-to-currency lookup table (no external API needed). When the owner provides a destination country, the agent looks it up deterministically: Germany → EUR, USA → USD, UAE → AED, India → INR. Most countries have one unambiguous currency. For genuinely ambiguous cases — "Europe" (which country?), "Caribbean" (USD? local currency?), or a country with multiple accepted trading currencies — the agent asks specifically rather than guessing. The SOUL.md constraint is clear: the agent only offers what it can actually do. If the destination is "Europe" and the agent cannot determine a specific country, it says "Which country? I'll use the correct currency automatically." It does not guess EUR and then discover the customer is in Switzerland (CHF). Once the currency is resolved, the conversion uses a local rate table and Decimal arithmetic throughout — no floats, no external API.

**Q: Does Munshi self-improve? What does the eval harness do?**

Munshi has the measurement half of self-improvement. The eval harness in `backend/app/evals/harness.py` defines labeled test cases — 9 EASY (deterministic rules should handle these) and 2 HARD (require model reasoning or memory) — and runs the full agent pipeline against them to produce an accuracy score. The `score_variants()` function ranks prompt variants by accuracy, allowing you to compare "does this new system prompt perform better than the current one?" That's GEPA-style measurement: structured benchmarking with labeled cases, automated scoring, variant comparison. What is NOT built yet is the reflection-proposal-approval loop that would close the GEPA cycle: the system does not yet automatically analyze why a variant failed, propose a new variant, or adopt a better prompt without human review. The harness gives you the infrastructure to run that loop manually — you change the prompt, run the harness, compare scores. Full automated self-improvement (where the model proposes prompt changes and the harness validates them) is planned future work.

---

## Interview Bridges

**Local-first → Apple on-device AI (privacy-by-architecture, not policy)**

Munshi's local-first design is the same pattern Apple uses to justify on-device AI features: privacy isn't something you promise in a Terms of Service — it's something you make physically impossible to violate. When Safari's machine learning features run on your device, there's no network call to make, no data to intercept, no server to subpoena. Munshi applies the same principle to financial data. This is stronger than any cloud provider's privacy policy because it doesn't depend on the provider keeping their word. It's architectural. An interviewer at Apple or any privacy-conscious company will immediately recognize this pattern and why it matters.

**HITL before consequential action → content moderation human review queue**

Every major content platform — YouTube, Facebook, Twitter — has a human review queue that sits between an automated classifier and an irreversible moderation action (account suspension, content removal). The automated system classifies; a human confirms before the hammer falls. Munshi's `request_approval()` gate is the same pattern applied to financial compliance. The automated system (fuzzy matcher + model) classifies; the human (business owner) confirms before the ITC claim is filed. The key insight is the same: automation handles volume, human judgment handles stakes. When discussing this in an interview, the bridge to content moderation makes it immediately concrete to anyone who has worked on trust & safety.

**Decimal over float → financial systems (same pattern in every bank and payment processor)**

Every financial system — Stripe, Square, every bank's core banking software — stores monetary amounts as integers (in the smallest denomination, e.g., cents/paise) or as Decimal types, never as float. The reason is identical to Munshi's: float cannot represent base-10 fractions exactly, and financial systems require exact arithmetic. This is not a niche decision; it's industry-standard practice that any senior engineer in fintech knows by heart. Munshi implements the same pattern at a smaller scale, for the same reason. In an interview, you can say "I applied the same arithmetic discipline you'd see in Stripe's payment processing."

**Fuzzy matching pipeline → entity resolution in data warehousing (Snowflake, dbt)**

The problem Munshi solves — the same entity described differently in two systems — is called entity resolution (or record linkage) in data engineering. dbt projects routinely need to match "TATA STEEL LTD" in a CRM against "Tata Steel Limited" in an ERP before joining tables. Snowflake and Databricks have built-in fuzzy join functions. The 4-signal pipeline (name similarity + identifier prefix + amount tolerance + date proximity) is a multi-signal entity resolution approach — the standard practice when a single signal isn't reliable enough. An interviewer with a data engineering background will recognize this framing immediately, and it positions Munshi's matching logic as applied data engineering, not a one-off hack.

**Overclaiming constraint in SOUL.md → guardrails pattern for production agents**

The SOUL.md constraint — "only offer what you can actually do" — is a formalization of the guardrails pattern that every production AI deployment needs. Language models are trained to be helpful, which means they tend to offer capabilities they don't actually have. In a customer service bot, this means promising a refund the system can't process. In a financial agent, this means offering to "check FTA rates" when there's no tool to do so. Munshi solves this by hard-bounding the agent's scope to its actual tool set, in writing, in the system prompt. This is a design pattern, not just a policy: the agent's identity (SOUL.md) explicitly lists what it can and cannot do. Any team building production agents needs this pattern — and being able to articulate it with a concrete example (Munshi) is a strong interview signal.

---

## What-If Scenarios

**"50 accounting firms want to use Munshi. Local-first doesn't work. What changes?"**

The architecture changes substantially. Local-first was the right call for a single business owner who controls their own machine. For 50 firms, you need a multi-tenant hosted service.

The stack migration: Ollama on a local machine → a hosted model API (or a private GPU cluster). SQLite → PostgreSQL with one schema per tenant (or row-level security with a `tenant_id` column). Single-user HITL → multi-user approval routing (which accountant approves which invoice category for which firm). Append-only audit log → per-tenant, compliance-grade immutable log with access controls.

The new hard problems: data residency (some firms may have regulatory requirements about where data is stored), tenant isolation (firm A's invoices cannot be visible to firm B's agent), and the privacy-by-architecture guarantee disappears. You now need encryption in transit and at rest, access control, and a platform audit trail (not just an agent audit trail — who on the platform accessed whose data?). The business model also changes: you're no longer delivering a local application, you're operating a SaaS with reliability and uptime obligations.

The Hermes runtime and SOUL.md pattern survive the migration — they're not inherently local. The Decimal arithmetic, fuzzy matcher, and HITL flow survive unchanged. The sovereignty promise does not survive, and you need explicit trade-off conversations with each firm about what they're giving up.

**"Munshi's model makes a wrong match that causes an ITC claim error. How do you detect and recover?"**

The first line of defense is the HITL gate — the owner sees every model judgment before it becomes a verdict. A wrong match should be caught at review time. But assume it slips through (owner approves a match quickly without scrutinizing).

Detection: The audit trail is append-only and contains the model's reasoning for every judgment. When the GSTN portal rejects the filing or raises a discrepancy, you trace back through the audit log to the specific match verdict. Because the log includes the model's reasoning and confidence level, you can identify whether this was a confident-wrong match (bad model judgment) or an uncertain match the owner approved anyway (a HITL gap).

Recovery: File a corrected GSTR-3B (the correction return). In SQLite, mark the affected verdict as `CORRECTED` and log the correction with a reference to the original entry. Do not delete or update the original — the audit trail must be immutable.

Systemic improvement: Add the wrongly-matched invoice pair to the HARD test cases in the eval harness. Run `score_variants()` to find a prompt variant that handles this class of error correctly. Update the system prompt. Run the full eval to confirm no regressions. This turns a production failure into a labeled test case — the GEPA loop, applied manually.

**"Add full GEPA self-improvement. What's the remaining work?"**

The measurement half is built: the eval harness runs cases, scores them, and ranks prompt variants. What's missing is the improvement half — the reflection-proposal-approval cycle.

Concretely:

1. **Failure analysis:** After `run_eval()`, analyze the WRONG cases. What did the model predict? What was expected? Is there a pattern (all failures on hard cases? on specific product categories?). This analysis currently happens manually; a GEPA loop would automate it using the model itself ("here are 3 cases you got wrong — what went wrong in your reasoning?").

2. **Prompt proposal:** The model proposes a revised system prompt that addresses the identified failures. This is the part that requires careful guardrails — a model proposing changes to its own instructions is a control risk. In Munshi's context, the proposed prompt would be presented to the owner (or a developer) for review before adoption.

3. **Automated validation:** Run `score_variants([current_prompt, proposed_prompt], test_cases)` and adopt the proposed prompt only if accuracy improves and no regressions appear on the EASY cases.

4. **Approval gate for prompt changes:** Just as financial actions require HITL approval, prompt changes should require explicit sign-off. The eval harness gives you the evidence; a human decides whether to apply the change.

The infrastructure investment is moderate — the harness is already built, and Hermes supports prompt updates. The risk management work (ensuring proposed prompts don't introduce subtle regressions or drift the agent's behavior outside SOUL.md constraints) is the harder problem.

---

## What I'd Do Differently

- **Structured outputs for model judgments.** Currently the model returns freeform text for ambiguous match reasoning. Force it to return `{"verdict": "MATCH", "confidence": "HIGH", "reasoning": "..."}` as structured JSON via Hermes's structured output mode. Freeform text is harder to parse for the audit log and harder to analyze in aggregate.

- **Confidence calibration over time.** Track model confidence level vs actual accuracy (as confirmed by owner approvals and rejections). A model that says "HIGH confidence" and is wrong 30% of the time on HIGH-confidence judgments needs its thresholds recalibrated. The eval harness gives the infrastructure; the calibration loop needs to be built.

- **Batch approval UI for high-volume periods.** For 500 invoices, a one-at-a-time approval flow is painful. A batch review interface — "here are 400 AUTO_MATCH results at HIGH confidence, sorted by amount — approve all, or spot-check before approving" — would dramatically improve owner UX without compromising the HITL principle.

- **Live exchange rates with staleness detection.** The current currency conversion uses a local rate table. Add a staleness timestamp and warn the owner if rates are more than N days old before presenting a foreign-currency quote.

- **Automated regression testing on model updates.** Each time the local Ollama model is updated (new Qwen or Llama version), run the full eval harness automatically and surface any accuracy regression before the owner uses the updated system. Model updates are silent breaking changes if not caught.

- **HARD case expansion.** The eval harness currently has 2 HARD cases — not enough to meaningfully measure model performance on genuinely difficult matches. Adding 10-20 HARD cases (sourced from actual rejected matches the owner has corrected over time) would make the harness a much stronger regression signal.

---

## Technical Dictionary

*Plain-English definitions of every term, algorithm, and tool used in this document. If something above confused you, start here.*

---

### GST & Indian Tax Domain

### GST (Goods and Services Tax)
India's unified indirect tax, introduced in 2017, that replaced a patchwork of central and state levies — excise duty, VAT, service tax, and others — with a single nationwide framework. Every business whose annual turnover exceeds a threshold (currently ₹20L for most states) must register, collect GST from customers, and file periodic returns. GST is structured in three tiers: IGST for inter-state transactions, and CGST + SGST split equally for intra-state ones.

**Example:** When Bharatvarsh Arts buys raw materials from a supplier in Maharashtra and resells handicrafts to a buyer in Gujarat, it pays IGST on the inter-state sale and claims ITC on any GST it paid on those inputs.

---

### GSTIN (Goods and Services Tax Identification Number)
A 15-character alphanumeric unique identifier issued to every GST-registered entity in India. The structure encodes state, taxpayer identity, and entity number: the first 2 digits are the state code, the next 10 mirror the taxpayer's PAN, and the remaining 3 characters distinguish multiple registrations under the same PAN. It is the primary reliable identifier used in reconciliation — more stable than vendor names.

**Example:** In Munshi's fuzzy matching pipeline, comparing the first 10 characters of two GSTINs (stripping the entity suffix) is the strongest single signal for confirming two records belong to the same supplier.

---

### GSTR-2A / GSTR-2B
Auto-generated GST return forms produced by the government portal that list all inward supplies (purchases) as reported by a business's suppliers. GSTR-2A updates dynamically as suppliers file; GSTR-2B is a locked monthly snapshot. A business can only claim ITC for invoices that appear here and match its own purchase records — unmatched invoices mean forfeited tax credit.

**Example:** Munshi's `fetch_gstr2b()` tool reads the GSTR-2B JSON downloaded from the portal and loads it into the agent's context so it can be matched against the owner's local purchase invoice CSVs.

---

### ITC (Input Tax Credit)
The mechanism that prevents GST from being taxed repeatedly across a supply chain. When a business pays GST on a purchase, that amount becomes a credit that can be offset against the GST it owes on its own sales. The net GST liability is: (GST collected from customers) minus (ITC from supplier invoices that appear in GSTR-2B and match your records).

**Example:** After Munshi reconciles all matched invoices for July 2026, it calls `compute_tax()` on each matched invoice to calculate ITC claimable — the sum becomes the total the owner can deduct before filing.

---

### HSN Code (Harmonised System of Nomenclature)
An internationally standardised six-to-eight-digit product classification code maintained by the World Customs Organization. In India's GST framework, every good must be assigned an HSN code; the code determines the applicable GST rate. Businesses above a turnover threshold are required to declare HSN codes on invoices and in filings.

**Example:** Munshi's eval harness uses HSN code prediction as the ground-truth label for its test cases — a correct prediction ("brass handicraft figurine" → `83062910`) confirms the agent is classifying trade goods accurately.

---

### Anti-Dumping Duty
A supplementary import tariff imposed when a foreign government or exporter is found to be selling goods in India below their fair market (or home-market) price, in a way that harms domestic industry. It is separate from and layered on top of basic customs duty, has an expiry date tied to the specific country and product, and is subject to sunset reviews.

**Example:** When Munshi computes a landed cost for an import from a country subject to an anti-dumping notification on a relevant product category, it must include this duty in the total — missing it would produce an understated cost estimate that could lead to a loss-making import decision.

---

### IGST / CGST / SGST
The three forms GST takes depending on the geography of a transaction. IGST (Integrated GST) applies to inter-state sales and imports; it is collected entirely by the central government and later apportioned to the destination state. CGST (Central GST) and SGST (State GST) apply to intra-state transactions and are split equally between the central government and the state where the transaction occurs. The applicable type is determined by comparing buyer and seller locations.

**Example:** In Munshi's `compute_tax()` function, each invoice carries separate `igst_rate`, `cgst_rate`, and `sgst_rate` fields; only one set is non-zero per invoice, depending on whether the underlying transaction crossed a state border.

---

### Algorithms & Matching

### Fuzzy Matching
A family of string comparison techniques that assign a similarity score between two strings that are similar but not identical, rather than returning a binary match/no-match. Unlike exact matching, fuzzy matching handles variations in spacing, punctuation, abbreviation, and word order. It is essential anywhere human-entered data from two different systems must be linked.

**Example:** Munshi's fuzzy matcher recognises "TATA STEEL LTD" and "TATA STEEL LIMITED" as the same supplier by normalising both strings (expanding "LTD" to "LIMITED") and computing a Levenshtein similarity score, rather than rejecting the pair as different strings.

---

### Token Set Ratio
A specific fuzzy matching algorithm (from the `rapidfuzz` / `fuzzywuzzy` family) that splits both input strings into individual word tokens, sorts them alphabetically, then computes similarity on the sorted result. Because sorting removes the effect of word order, "Pvt Ltd Tata Steel" and "Tata Steel Pvt Ltd" score 100% despite having different word positions. This is especially effective for vendor names where legal suffixes move around.

**Example:** Munshi's name normalisation step uses token set ratio as one ingredient in vendor name similarity scoring, ensuring that supplier names with rearranged legal suffixes do not count as dissimilar.

---

### Python Decimal
Python's built-in `decimal.Decimal` type, which performs arithmetic in base-10 (decimal) rather than binary floating-point. Numbers are stored as exact decimal values with configurable precision, and arithmetic on them is exact to the specified number of decimal places. This makes it the correct type for any financial computation where even sub-paisa errors are unacceptable.

**Example:** Every tax computation in Munshi wraps raw invoice values in `Decimal(str(value))` immediately on ingestion and never converts back to float — all intermediate results, including CGST, SGST, and IGST per invoice, are computed and quantized to two decimal places using `ROUND_HALF_UP`.

---

### Floating-Point Error
The unavoidable rounding imprecision that arises when base-10 decimal fractions are represented in binary (base-2) floating-point format. Many common financial amounts — such as 333.33 or 1250.75 — cannot be expressed exactly in binary, so the stored value is a close approximation. When many such approximations are added together, the small per-value errors accumulate into a larger visible error.

**Example:** Adding 500 float-arithmetic GST amounts in Munshi could produce a total that differs from the GSTN portal's exact-decimal result by several rupees, triggering a filing mismatch — which is why Munshi prohibits float in all tax computations.

---

### Agent Architecture

### Hermes
The open-source agentic runtime (developed by Nous Research, MIT licensed) that Munshi uses to manage its reasoning loop, tool registration, and memory. Hermes provides the scaffolding for a ReAct-style loop — reason, call a tool, observe the result, repeat — without requiring custom agent plumbing. It also enforces typed, schema-validated MCP tool calls, which reduces the likelihood of the model generating malformed arguments.

**Example:** When the owner asks "how much ITC can I claim this month?", Hermes manages the sequence of tool calls — `fetch_gstr2b()`, `read_invoices()`, `fuzzy_match()`, `compute_tax()`, `request_approval()` — tracking results in the context window between each step.

---

### SOUL.md
A plain-text Markdown file loaded as the agent's system prompt at startup that defines its identity, operating constraints, and hard limits. Rather than relying on general model training to govern behaviour, SOUL.md makes the agent's rules explicit and auditable: a developer or owner can read the file and know exactly what the agent has been told it can and cannot do.

**Example:** Munshi's SOUL.md includes the clause "NEVER invent a number — not a duty rate, tax amount, HS/HSN code, or any rupee/euro figure," which instructs the model to delegate all numeric computation to deterministic tools rather than generating figures itself.

---

### MCP (Model Context Protocol)
A typed, schema-validated protocol for exposing callable tools to an AI model in a structured way. Instead of the model generating tool calls as free-form text that must be parsed, MCP defines each tool's name, input schema, and output schema upfront. The runtime validates arguments against the schema before executing the tool, catching malformed calls before they reach the underlying function.

**Example:** Munshi's `compute_tax()` tool is registered via MCP with a schema requiring `taxable_value`, `igst_rate`, `cgst_rate`, and `sgst_rate` as typed inputs — if the model tries to call it with a missing field or the wrong type, Hermes rejects the call before it reaches the Python function.

---

### LangChain
An open-source Python and JavaScript framework for building LLM-powered applications. Provides abstractions for chains (sequences of LLM calls and tool calls), agents (ReAct loops with tool use), memory, and a large library of pre-built integrations with data sources, vector stores, and model providers. Widely used for rapid prototyping. Known for a large ecosystem and frequent breaking API changes between versions.

**Example:** A developer building a prototype that needs to load PDFs, split text, embed chunks, and run a retrieval chain would reach for LangChain's pre-built components — faster than writing each piece from scratch.

**Vs Hermes:** LangChain is broader but more abstract. Hermes is more focused on the ReAct loop with native MCP tool schema enforcement — important in financial systems where wrong arguments have real consequences.

---

### LangGraph
A graph-based workflow engine built on top of LangChain. Applications are modelled as a StateGraph — nodes are functions (including LLM calls), edges define the flow, and the shared state is typed and checkpointed. Features built-in HITL via `interrupt()` / resume, and durable persistence via checkpointers (MemorySaver for dev, Postgres for production). Optimised for known, multi-step workflows — not for open-ended agent loops where the AI decides its own path.

**Example:** 01dev's QuizMe feature uses LangGraph: `plan_objectives → interrupt (human reviews) → generate_quiz → interrupt (student submits) → evaluate`. Every step is fixed; the graph structure matches the workflow exactly.

**Vs Hermes:** LangGraph is the right tool when every step is known before execution starts and HITL with suspend/resume is a requirement. Hermes is the right tool when the agent's tool path is dynamic and decided at runtime.

---

### HITL (Human-in-the-Loop)
The design pattern of inserting a mandatory human review and approval step before the system takes a consequential or irreversible action. The automated system prepares a complete summary of the proposed action and the reasoning behind it; the human then decides whether to approve or reject. HITL trades speed for control — the human is the last check before the action is committed.

**Example:** Before Munshi records a MATCH verdict for any invoice pair and makes it eligible for ITC, it calls `request_approval()`, surfaces the matched records, model reasoning (if applicable), and computed tax amount to the owner, and halts until the owner explicitly approves or rejects.

---

### ReAct Loop
An agent execution pattern (Reasoning + Acting) in which the model alternates between reasoning about the current state and calling a tool to advance it. The model reads the conversation history and prior tool results, decides what to do next, calls a tool, receives the result, appends it to context, and reasons again. This continues until the task is complete or a stopping condition (like a HITL gate) is reached.

**Example:** Munshi's Hermes runtime runs a ReAct loop for each owner query: reason about what data is needed → call `fetch_gstr2b()` → observe the result → reason about which invoices to match → call `fuzzy_match()` → observe verdicts → reason about tax computation → call `compute_tax()` → observe amounts → call `request_approval()` → pause.

---

### Deterministic Tool
A function that always returns the same output for the same input, with no model or probabilistic component involved in the computation. The result is computed by code — arithmetic, lookup tables, or rule-based logic — not by an LLM. Deterministic tools are auditable, testable, and produce no hallucinated results.

**Example:** `compute_tax()` in Munshi is deterministic: given `taxable_value=Decimal('47500.00')` and `igst_rate=18`, it always returns `Decimal('8550.00')` — no model call is made, and the result is mathematically verifiable by the owner.

---

### Evaluation

### GEPA-Style Eval Harness
An evaluation framework modelled on the GEPA (Generalised Evaluation and Prompt Adaptation) pattern, which runs a fixed set of labeled test cases through the agent system and scores accuracy. The "style" qualifier matters: Munshi implements the measurement half — running cases and scoring variants — but not the full self-improvement loop, which would also include automated failure reflection, prompt proposal, and a Pareto-gated approval cycle.

**Example:** Munshi's `harness.py` runs 9 EASY and 2 HARD labeled HSN classification cases through the full agent pipeline, producing accuracy and timing metrics that allow developers to compare prompt variants before deploying a new SOUL.md.

---

### run_eval(cases)
A function in `backend/app/evals/harness.py` that accepts a list of labeled test cases, passes each one through the full agent pipeline, compares the agent's prediction against the expected answer, and returns aggregate accuracy, elapsed time, and per-case results. It is the primary measurement tool for assessing agent quality.

**Example:** `run_eval(HARD_CASES)` tells the developer what fraction of the genuinely difficult HSN classification tasks — the ones that require model reasoning rather than a simple lookup — the current SOUL.md version handles correctly.

---

### score_variants(variants, cases)
A function that iterates over a list of candidate system prompt strings, runs `run_eval()` for each one against the same test cases, and returns the variants sorted by accuracy descending. It is the mechanism for empirically selecting the best prompt version rather than choosing by intuition.

**Example:** Before deploying a revised SOUL.md that adds stronger HSN guidance, Munshi's developer runs `score_variants([current_soul, proposed_soul], EASY_CASES + HARD_CASES)` to confirm the proposed version does not regress on cases the current version already handles correctly.

---

### Infrastructure

### SQLite
A self-contained, serverless, file-based relational database. The entire database is a single file on disk; no separate database process is required, and no network connection is needed to access it. It is well-suited to single-user or single-process applications where simplicity and portability matter more than concurrent write throughput.

**Example:** Munshi stores all persistent state — vendor resolutions, match verdicts, audit log entries, and approved actions — in a single `munshi.sqlite` file on the owner's local machine, requiring zero database administration and leaving no data on any remote server.

---

### Docker Compose
A tool for defining a multi-service application as a single declarative configuration file (`docker-compose.yml`) and bringing all services up or down together with one command. It handles networking between containers, volume mounts, and environment variables, so the developer does not need to manage each container separately.

**Example:** Munshi's local stack — the Hermes runtime, Ollama model server, and MCP tool layer — is defined in a `docker-compose.yml` so the owner can start the entire system with `docker compose up` and stop it with `docker compose down`, without needing to know which ports or processes are involved.

---

### Ollama
An open-source tool for running large language models locally on a laptop or desktop, exposing them via an OpenAI-compatible HTTP API. It handles model download, quantisation for CPU/GPU, and serving, so an application can switch between local models by changing a single URL and model name rather than rewriting inference code.

**Example:** Munshi points its Hermes runtime at `http://localhost:11434` (Ollama's default port) and uses a locally downloaded Qwen or Llama model for all inference — the same API shape as OpenAI, but every token is generated on the owner's machine with no data transmitted anywhere.

---

### Sovereignty
In software architecture, the property that all data storage, computation, and inference occur entirely within infrastructure the owner controls — with no code paths that transmit data to external services for processing. Sovereignty is an architectural guarantee, not a policy: it holds because there are no external API calls in the codebase, not because the provider promises confidentiality.

**Example:** Munshi achieves sovereignty by running Ollama locally for inference, SQLite locally for storage, and a local rate table for currency conversion — there is no HTTP call in the codebase that sends invoice data, tax amounts, or GSTIN numbers to any external host.
