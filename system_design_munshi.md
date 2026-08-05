# Munshi — System Design

**Platform:** Sovereign local-first AI agent for GST reconciliation and financial compliance  
**Built for:** Bharatvarsh Arts (₹5Cr / $500K+ annual revenue)  
**Stack:** Hermes runtime, MCP tools, FastAPI, Ollama, Python Decimal, Docker, SQLite

---

## Problem Statement

Bharatvarsh Arts processes hundreds of purchase invoices monthly and must reconcile them against GSTR-2B (government supplier filings) to claim Input Tax Credit (ITC). The existing workflow was entirely manual — Excel spreadsheets, human matching, error-prone tax computation. Key constraints:
- Financial data is sensitive — cannot be sent to any external API or cloud service
- Tax computation must be exact — float arithmetic causes compounding errors at scale
- Non-technical owner must be able to verify every AI decision before filing
- Human must approve all consequential actions — the agent cannot file independently

---

## Functional Requirements

- Plain-English queries: "How much ITC can I claim this month?"
- Fuzzy reconciliation of purchase invoices against GSTR-2B records
- Exact tax computation (CGST, SGST, IGST per invoice)
- Audit trail of every model decision, tool call, and computation in plain English
- Human approval gate before any consequential action (filing, ITC claim, reconciliation verdict)
- Persistent memory for approved financial classifications (carry forward month-to-month)

## Non-Functional Requirements

- **Data sovereignty:** Zero data over network — all inference, storage, and computation local
- **Correctness:** Tax arithmetic exact to the paisa — no float errors
- **Explainability:** Non-technical owner can understand and verify every decision
- **Reliability:** Model failures must not produce silent wrong results — surface clearly

---

## High-Level Architecture

```
Owner (plain-English query)
    │
    ▼
Hermes Runtime (local)
    │ agent loop: reason → select tool → call → observe → repeat
    │
    ├── MCP Tool: fetch_gstr2b()    → reads GSTR-2B JSON
    ├── MCP Tool: read_invoices()   → reads purchase invoice CSV/JSON
    ├── MCP Tool: fuzzy_match()     → matches invoices to GSTR-2B
    ├── MCP Tool: compute_tax()     → Python Decimal arithmetic
    ├── MCP Tool: write_audit()     → appends to audit trail (SQLite)
    └── MCP Tool: request_approval() → pauses, presents to owner
    │
    ▼
Ollama (local inference — Qwen / Llama)
    │ model runs on local GPU/CPU — no data transmitted
    │
    ▼
Human approval gate
    │ owner reviews audit trail, approves or rejects
    │
    ▼
Consequential action executed (or rejected)
```

**Everything runs on the local machine.** Ollama handles model inference locally. No data leaves the boundary.

---

## Detailed Component Design

### 1. Hermes Runtime (Agent Loop)

Hermes (Nous Research, MIT licensed) manages the agentic loop:
- **Reason:** Model reads conversation history + tool results, decides next action
- **Act:** Calls selected MCP tool with typed arguments
- **Observe:** Tool result appended to context
- **Repeat:** Until task is complete or human approval is required

**Why Hermes over custom agent loop:**
- Zero plumbing from scratch — tool registration, memory, and loop are handled
- MCP protocol gives typed, schema-validated tool calls — reduces model hallucinating tool arguments
- Agent identity declared in `SOUL.md` — persona, constraints, operating boundaries in plain text

**SOUL.md declares:**
- Agent identity and tone (formal, cautious, financial domain)
- Operating constraints: never file without human approval, always show computation
- What the agent can and cannot do independently

---

### 2. Fuzzy Invoice Matching

The core reconciliation problem: purchase invoices use vendor names like "TATA STEEL LTD" — GSTR-2B has "Tata Steel Limited". Exact matching fails. ITC is forfeited.

**Matching pipeline:**

```
For each purchase invoice:
    1. Normalise vendor name (lowercase, strip punctuation, expand abbreviations)
    2. GSTIN prefix match (first 10 chars of 15-char GSTIN)
    3. Amount match with ±2% tolerance
    4. Date proximity check (within 30 days)
    
If all 4 signals agree → AUTO_MATCH (deterministic, no model call)
If 2-3 signals agree  → AMBIGUOUS → escalate to model
If < 2 signals agree  → NO_MATCH
```

**Model role (ambiguous cases only):**
The model receives the normalised invoice and the candidate GSTR-2B record. It reasons in plain English: "The vendor 'Tata Steel Ltd' and 'TATA STEEL LIMITED' are likely the same entity given the GSTIN prefix match and similar amount. Recommend: MATCH with confidence HIGH."

The model never computes tax. It only judges whether two records refer to the same supplier.

**Why model only for ambiguous cases:**
- ~80% of invoices auto-match deterministically — zero LLM cost
- Model judgment is applied only where human intuition is needed
- Keeps the system interpretable — deterministic logic is auditable, model judgment is flagged for review

---

### 3. Tax Computation — Python Decimal

All tax arithmetic uses Python's `Decimal` module. Never floats.

**Why Decimal matters at ₹5Cr scale:**

```python
# Float (WRONG)
>>> 333.33 + 333.33 + 333.34
999.9999999999999  # not 1000.00

# Decimal (CORRECT)
>>> Decimal('333.33') + Decimal('333.33') + Decimal('333.34')
Decimal('1000.00')
```

At 500 invoices/month, float errors compound. An ITC claim that's ₹0.01 off per invoice = ₹5 error per month — small individually, but causes mismatches with GSTN portal computations which use exact arithmetic.

**Computation flow:**
```python
def compute_itc(invoice):
    taxable_value = Decimal(str(invoice['taxable_value']))
    igst_rate = Decimal(str(invoice['igst_rate'])) / 100
    igst = (taxable_value * igst_rate).quantize(Decimal('0.01'))
    return igst
```

All intermediate results quantized to 2 decimal places (paise). Final aggregation uses Decimal accumulator.

---

### 4. Human-in-the-Loop (HITL)

**Consequential actions requiring approval:**
- Accepting a reconciliation verdict (MATCH / NO_MATCH)
- ITC claim amount submission
- Any financial filing action

**Approval flow:**
1. Agent calls `request_approval(action, justification, computation_detail)`
2. Agent loop pauses — no further actions taken
3. UI presents the pending action with full audit context
4. Owner reviews: audit trail, model reasoning, computed amounts
5. Owner approves (action executes) or rejects (action cancelled, feedback logged)
6. Approved classifications persist in SQLite — same judgment not repeated next month

**Audit trail format (plain English):**
```
[2026-07-15 14:32] Tool called: fuzzy_match()
  Invoice: TATA STEEL LTD, ₹47,500, GSTIN 27AAACT2727Q1ZW
  Candidate: TATA STEEL LIMITED, ₹47,500, GSTIN 27AAACT2727Q1ZW
  Model reasoning: "GSTIN prefix matches exactly. Name is a known abbreviation. Amount identical."
  Decision: MATCH (confidence: HIGH)
  Status: AWAITING APPROVAL

[2026-07-15 14:34] Owner approved MATCH verdict
  ITC claimable: ₹8,550.00 (IGST @ 18%)
  Computation: Decimal('47500.00') × Decimal('0.18') = Decimal('8550.00')
```

---

### 5. Persistent Memory

Hermes maintains memory across sessions via SQLite.

**What is persisted:**
- Approved vendor name resolutions ("TATA STEEL LTD" = "TATA STEEL LIMITED" → persist, don't ask again)
- Recurring invoice patterns (same supplier, similar amount each month → pre-approve category)
- Rejected model judgments with owner feedback (used to improve prompts)

**What is NOT persisted in the model's context:** Financial data. Raw invoices and GSTR-2B records are read fresh from source files each session — never stored in the agent's memory. Only the classified verdicts are remembered.

---

### 6. Data Flow & Storage

```
/data/invoices/        — purchase invoice CSVs (local filesystem)
/data/gstr2b/          — GSTR-2B JSON downloads (local filesystem)
/db/munshi.sqlite      — verdicts, approved matches, audit trail, memory
/logs/audit.log        — plain-English audit trail (append-only)
```

All storage is local. SQLite chosen for simplicity — no server process, zero configuration, fully embedded. For a multi-user version, replace with PostgreSQL.

---

### 7. Key Engineering Decisions

| Decision | Choice | Alternative | Why |
|---|---|---|---|
| Architecture | Local-first, Ollama | Cloud API (GPT-4) | Data sovereignty — financial data never transmitted |
| Tax arithmetic | Python Decimal | float | Exact computation, no accumulation errors |
| Model role | Judge ambiguous matches only | Compute tax | Deterministic for clear cases, model for judgment-required cases |
| HITL | Before all consequential actions | Post-hoc review | Owner verifies before action, not after — no rollback needed |
| Agent runtime | Hermes | Custom loop | Zero plumbing, typed MCP tool calls |
| Memory | SQLite (verdicts only) | Full conversation history | Raw financial data not persisted; only classifications |
| Matching | Fuzzy (GSTIN + name + amount + date) | Exact string match | Vendor name inconsistencies are the actual problem |

---

### 8. Scalability Considerations

**Current:** Single business, ~500 invoices/month. Everything runs on one machine.

**To serve 50 accounting firms:**
- Move from local-first to hosted (FastAPI + PostgreSQL per tenant)
- Model inference: Ollama → cloud API (one config line change in Hermes)
- Data residency: each firm's data stays in their own DB partition
- HITL: multi-user approval routing — which accountant approves which invoice category
- Audit trail: immutable append-only log per firm (compliance requirement)

**Trade-off of going cloud:** Lose the privacy-by-architecture guarantee. Must implement data encryption in transit and at rest, access controls, audit logs for the platform itself (not just the AI decisions).

---

### 9. What I'd Do Differently

- **Structured outputs for model judgments:** Force the model to return `{verdict, confidence, reasoning}` as structured JSON — easier to process and audit than freeform text
- **Confidence calibration:** Track model confidence vs actual accuracy over time — recalibrate thresholds
- **Batch approval UI:** Currently one-at-a-time — for 500 invoices, a batch review with filtering (all HIGH confidence matches shown in a table, one-click approve all) would dramatically improve owner UX
- **Automated regression tests:** Run a golden set of known invoice pairs through the matcher monthly — catch if model updates degrade matching accuracy
