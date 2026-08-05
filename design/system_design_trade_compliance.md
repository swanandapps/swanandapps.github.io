# Trade Compliance Researcher — System Design

**Platform:** Multi-agent trade compliance research system
**Stack:** Hermes runtime, MCP tools, Ollama, Docker Compose, Python
**Pattern:** Researcher → Writer two-agent workflow, model-agnostic
**Deployment:** `docker compose up` — one command, full stack

---

## Quick Reference

| Dimension | Detail |
|---|---|
| Pattern | Two-agent pipeline: Researcher gathers, Writer synthesises |
| Runtime | Hermes (wraps any OpenAI-compatible model) |
| Model (dev) | Ollama + qwen2.5:7b — fully local, no API cost |
| Model (prod) | One config line to swap → gpt-4o-mini or any cloud model |
| Agent identity | SOUL.md files — data, not code |
| Tool protocol | MCP — typed, schema-validated, model-callable |
| Memory | Hermes conversation history — persists across turns |
| Deployment | Docker Compose — ollama + mcp_tools + hermes_runtime |
| Key constraint | Writer has zero tools — synthesis only, enforced by config |

---

## Problem Statement

Trade compliance research is time-intensive and high-stakes: finding applicable regulations, tariff codes, import/export restrictions, duty rates, anti-dumping notifications, and active FTAs for a specific commodity across two jurisdictions can take a human researcher several hours. The data lives in scattered sources — government regulation databases, tariff schedule PDFs, trade agreement annexures — and must be synthesised into a coherent, citeable report.

Errors are costly: underpaying duty creates customs liability; overpaying creates cash flow problems; missing a restriction can get a shipment seized.

The system automates this research loop with two specialised agents. One iteratively queries authoritative data sources until it has sufficient information. The other synthesises all findings into a structured, citeable report that a compliance officer can act on. Neither agent does the other's job — a deliberate architectural constraint.

---

## Functional Requirements

- Accept a natural-language trade compliance query (commodity, origin, destination, HS code)
- Researcher agent autonomously gathers data via MCP tools, iterating until findings are sufficient
- Writer agent synthesises findings into a structured report with full citations
- Report structure: Executive Summary → Key Findings → Detailed Analysis → Sources
- Writer flags ambiguities and data gaps the Researcher did not resolve
- Memory persists across turns — agents retain prior conversation context for follow-up queries
- Swap model providers (Ollama local → OpenAI cloud) with a single config.yaml line change — zero code changes
- Fully containerised — deployable with `docker compose up` on any machine with Docker
- New MCP tools can be added without modifying agent logic — extend the tools server only

---

## Non-Functional Requirements

- **Model-agnostic:** Any OpenAI-compatible endpoint works — Ollama, OpenAI, Groq, Together, Anthropic via compatibility layer
- **No hardcoded plumbing:** Agent identity, capabilities, and model choice are declared in config and SOUL.md, not in code
- **Extensible tools:** New MCP tools are added as server-side Python functions with a schema decorator — agents discover them automatically
- **Auditability:** SOUL.md files are plain Markdown — compliance teams and non-engineers can read and audit agent instructions
- **Reproducible dev environment:** Docker Compose pins all service versions — no "works on my machine" drift
- **Separation of concerns enforced:** Writer's tool list is empty in config — it cannot make tool calls even if the SOUL.md instructed it to

---

## High-Level Architecture

```
User Query
    │
    ▼
┌────────────────────────────────────────────────────────────────────┐
│  Hermes Runtime                                                    │
│                                                                    │
│  ┌─────────────────────────────┐     findings     ┌─────────────┐ │
│  │     Researcher Agent        │ ───────────────► │   Writer    │ │
│  │     (researcher.soul.md)    │                  │   Agent     │ │
│  │                             │                  │(writer.soul │ │
│  │  iteration 1: fetch_tariff  │                  │    .md)     │ │
│  │  iteration 2: get_trade_agr │                  │             │ │
│  │  iteration 3: search_regs   │                  │  no tools   │ │
│  │  → "sufficient, pass over"  │                  │             │ │
│  └──────────────┬──────────────┘                  └──────┬──────┘ │
│                 │                                         │        │
│    tool calls   │                                         │        │
└─────────────────┼─────────────────────────────────────── │ ───────┘
                  │                                         │
                  ▼                                         ▼
     ┌────────────────────────┐                  Structured Report
     │    MCP Tools Server    │                  (citations, sources,
     │                        │                   flagged gaps)
     │  search_regulations    │
     │  fetch_tariff_db       │
     │  fetch_document        │
     │  get_trade_agreement   │
     └────────────┬───────────┘
                  │
                  ▼
     ┌────────────────────────┐
     │   Ollama (local LLM)   │
     │   qwen2.5:7b           │
     │   http://localhost:    │
     │   11434/v1             │
     └────────────────────────┘
```

Both agents run on the same Hermes runtime. They are identical at the infrastructure level. What differentiates them is their SOUL.md file and their tool list in config.yaml.

---

## Detailed Component Design

### 1. Agent Identity — SOUL.md Pattern

Each agent has a `SOUL.md` file declaring its identity, persona, constraints, and operating boundaries. This file is the agent's system prompt — but externalised as a file, not hardcoded in source.

**Researcher SOUL.md:**
```markdown
You are a trade compliance researcher. Your role is to gather accurate,
current regulatory information using the tools available to you.

- Call tools iteratively until you have sufficient information
- Never synthesise or write the final report — that is the Writer's role
- Cite every source by tool call and result
- If a tool returns insufficient data, try a different search strategy
- Stop researching when you have enough to answer the query completely
```

**Writer SOUL.md:**
```markdown
You are a technical writer specialising in trade compliance reports.
Your role is to synthesise the Researcher's findings into a clear,
structured report.

- Do not call any tools — you write from what the Researcher found
- Every claim must be traceable to a specific finding
- Structure: Executive Summary → Key Findings → Detailed Analysis → Sources
- Flag any gaps or ambiguities the Researcher did not resolve
```

**Why SOUL.md:**
- Agent identity is data, not code — editable without deployment, reviewable in a pull request
- Compliance teams and non-engineers can read and audit agent instructions without touching Python
- Both agents use the same Hermes runtime — differentiated only by SOUL.md and config
- SOUL.md changes are tracked in version control like any other file — you get history, blame, and rollback
- Avoids the "who owns the system prompt?" problem: it's a file, with an owner, checked into the repo

---

### 2. config.yaml — Model-Agnostic Endpoint

```yaml
researcher:
  soul: ./souls/researcher.soul.md
  model_endpoint: http://localhost:11434/v1  # Ollama local
  model_name: qwen2.5:7b
  tools: [search_regulations, fetch_tariff_db, fetch_document, get_trade_agreement]

writer:
  soul: ./souls/writer.soul.md
  model_endpoint: http://localhost:11434/v1  # same or different
  model_name: qwen2.5:7b
  tools: []  # Writer has no tools — synthesis only
```

**To switch to OpenAI:**
```yaml
model_endpoint: https://api.openai.com/v1
model_name: gpt-4o-mini
```

**To switch to Groq:**
```yaml
model_endpoint: https://api.groq.com/openai/v1
model_name: llama-3.1-70b-versatile
```

One line. Zero code changes. Hermes reads the endpoint from config and constructs the OpenAI-compatible HTTP client. The tool-calling format, message structure, and streaming protocol are identical across all OpenAI-compatible providers. Swapping the endpoint swaps the model, nothing else.

This is the same pattern LiteLLM and LangChain use for provider abstraction — a unified interface over heterogeneous endpoints. The difference here is that it's custom-built into Hermes rather than delegated to a library.

---

### 3. MCP Tools — Researcher's Toolbox

Tools are defined as MCP (Model Context Protocol) servers — typed, schema-validated, callable by the model via a standard protocol.

```python
@mcp_tool(
  name="search_regulations",
  description="Search trade regulations database by HS code, country, or commodity",
  input_schema={
    "hs_code": {"type": "string", "description": "6-digit HS tariff code"},
    "countries": {"type": "array", "items": {"type": "string"}},
    "query": {"type": "string"}
  }
)
def search_regulations(hs_code, countries, query):
    # fetch from regulation DB or scrape authorised sources
    return {"regulations": [...], "sources": [...]}
```

**Available tools:**
- `search_regulations(hs_code, countries, query)` — tariff and regulation search
- `fetch_tariff_db(hs_code, origin, destination)` — duty rates lookup
- `fetch_document(url)` — retrieve and parse regulatory documents
- `get_trade_agreement(countries)` — check applicable FTAs and bilateral agreements

**Why MCP over direct Python function calls:**
- Schema validation — the MCP layer validates every tool call before execution. If the model hallucinates a parameter that isn't in the schema, the call is rejected before it reaches the Python function. This eliminates a category of failure at the boundary.
- Tool discovery — Hermes generates the tool descriptions for the model from the MCP schema automatically. No manual prompt engineering of tool descriptions in the system prompt.
- Testable independently — each MCP tool is a standalone service. You can test `fetch_tariff_db` with curl or a test client without running Hermes or a model at all.
- Standardised protocol — MCP is a published spec. Any MCP-compatible runtime can use these tools, not just Hermes.

---

### 4. Agent Execution Flow

Complete example trace — real query, all three iteration steps:

```
User: "What are the import duties for lithium batteries from China to India
       under HS code 8507.60?"

Hermes → Researcher agent (has tools, starts iterating)

──── Iteration 1 ─────────────────────────────────────────────────────
Researcher reasoning: "I need the base duty rate for this HS code."
Tool call: fetch_tariff_db(hs_code="8507.60", origin="CN", destination="IN")
Tool result: {
  basic_duty: "15%",
  igst: "18%",
  social_welfare_surcharge: "10% of basic duty",
  cess: "0%",
  source: "CBIC Customs Tariff 2024-25"
}

──── Iteration 2 ─────────────────────────────────────────────────────
Researcher reasoning: "I should check if any FTA reduces the basic duty."
Tool call: get_trade_agreement(countries=["CN", "IN"])
Tool result: {
  agreement: null,
  note: "India and China have no operative bilateral FTA covering HS 8507.60.
         ASEAN FTA does not apply — China is not an ASEAN member."
}

──── Iteration 3 ─────────────────────────────────────────────────────
Researcher reasoning: "Check for anti-dumping or safeguard duties — common
                       for Chinese electronics."
Tool call: search_regulations(
  hs_code="8507.60",
  countries=["IN"],
  query="anti-dumping duty"
)
Tool result: {
  notifications: [{
    notification_no: "ADD-2022-174",
    duty: "20%",
    valid_until: "2027-03-31",
    authority: "DGTR / Ministry of Finance",
    description: "Anti-dumping duty on lithium-ion cells from China"
  }]
}

Researcher: "Sufficient information gathered. Passing all findings to Writer."

──── Writer phase ─────────────────────────────────────────────────────
Writer input: all three iteration results above
Writer output (structured report):

## Executive Summary
Importing lithium batteries (HS 8507.60) from China to India attracts a
total effective duty of approximately 56.7%, comprising basic customs duty,
an active anti-dumping duty, IGST, and applicable surcharges. No FTA relief
is available.

## Key Findings
1. Basic Customs Duty: 15% (CBIC Tariff 2024-25)
2. Anti-Dumping Duty: 20% — ADD-2022-174, valid until 2027-03-31
3. IGST: 18% on assessable value + duties
4. Social Welfare Surcharge: 10% of basic duty (1.5% effective)
5. No applicable FTA — India-China have no operative trade agreement

## Detailed Analysis
[full breakdown with effective rate calculation and CIF basis assumptions]

## Sources
- CBIC Customs Tariff 2024-25 (fetch_tariff_db)
- DGTR Notification ADD-2022-174 (search_regulations)
- India-China trade agreement status (get_trade_agreement)
```

**Memory persistence:** Hermes maintains the full conversation history across turns. If the user follows up — "What about batteries from South Korea instead of China?" — the Researcher already has the India tariff structure in context and goes straight to checking FTA applicability (India-Korea CEPA), avoiding re-fetching already-known data.

---

### 5. Docker Compose Deployment

```yaml
version: '3.8'
services:
  ollama:
    image: ollama/ollama
    volumes: [ollama_models:/root/.ollama]
    ports: ['11434:11434']
    # on first run: docker exec ollama ollama pull qwen2.5:7b

  mcp_tools:
    build: ./tools
    ports: ['8001:8001']
    environment:
      - REGULATION_DB_PATH=/data/regulations.db
      - TARIFF_DB_PATH=/data/tariffs.db
    volumes: [./data:/data]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
      interval: 10s
      retries: 5

  hermes_runtime:
    build: ./hermes
    depends_on:
      ollama: {condition: service_started}
      mcp_tools: {condition: service_healthy}
    environment:
      - CONFIG_PATH=/app/config.yaml
      - MCP_TOOLS_URL=http://mcp_tools:8001
    volumes:
      - ./souls:/app/souls
      - ./config.yaml:/app/config.yaml
    ports: ['8000:8000']

volumes:
  ollama_models:
```

`docker compose up` starts the full stack. Service dependency order is enforced: Hermes waits for the MCP tools health check before starting. Ollama model volumes persist across restarts — the 4GB model download only happens once.

---

## Why Two Agents Instead of One

This is the most common design question and deserves a dedicated answer.

### The naive design

A single agent could theoretically do both: call tools to gather data, then write the report in the same response. Many simple agent systems work this way. It works for simple queries. It breaks down at scale.

### What goes wrong with one agent

**Prompt drift under tool-calling pressure.** When a single agent is mid-way through its tool-calling loop (three tool calls in, accumulating results), its attention is optimised for gathering. Asking it to shift modes and write a structured, citeable report in the same response degrades both tasks. The model has to hold two incompatible mental modes simultaneously.

**Synthesis contaminates research.** A single agent that writes as it researches tends to stop researching once it has enough to write a plausible-looking report — even if the data is incomplete. The Researcher's SOUL.md explicitly forbids synthesis, which keeps it in pure-gathering mode until it declares sufficiency.

**Tool access is uncontrollable in a single agent.** If one agent both researches and writes, there is no clean way to prevent it from calling tools during the writing phase. You can say "don't call tools now" in the system prompt, but models are not reliably instruction-following under all contexts. In the two-agent design, the Writer literally has `tools: []` in config — it cannot call tools even if it tries.

**Debugging is harder.** When a single agent produces a bad report, you don't know if the problem was bad research (wrong tool calls, missed data) or bad synthesis (correct data, poorly structured output). With two agents, you can inspect the Researcher's output independently. The failure is localised.

### The MapReduce analogy

This is MapReduce at the agent level. The Researcher is the Map phase: it fans out across tools, collecting raw data from multiple sources in parallel (or sequentially) and emitting structured findings. The Writer is the Reduce phase: it takes all findings, collapses them into a single coherent output, and applies structure.

MapReduce separates these phases for the same reason: gathering and aggregating are different operations with different computational profiles, and mixing them produces worse results than doing them sequentially with clean handoffs.

### The SOUL.md constraint is deliberate

The separation isn't just a runtime constraint — it's a design signal. When the Writer's SOUL.md says "do not call any tools — you write from what the Researcher found," it's not just an instruction. It's documentation of the system's intended information flow. Anyone reading the SOUL.md files understands the architecture without reading a line of Python.

---

## Key Engineering Decisions

| Decision | Choice | Alternative | Why |
|---|---|---|---|
| Agent identity | SOUL.md files | Hardcoded system prompts in Python | Editable without deployment; auditable by compliance teams; version-controlled as data |
| Model config | config.yaml endpoint + model_name | Hardcoded model name in source | One line to swap providers — Ollama for dev, GPT-4o for prod, zero code changes |
| Tool schema | MCP typed tools with input_schema | Direct Python function calls | Schema validation rejects malformed tool calls before execution; tool discovery is automatic |
| Agent separation | Researcher + Writer | Single agent does both | Enforces research/synthesis separation; Writer's tool list is empty in config — cannot call tools |
| Memory | Hermes conversation history across turns | Stateless per-query (no memory) | Follow-up queries have full context; Researcher avoids re-fetching already-known data |
| Deployment | Docker Compose with health checks | Manual setup or bare-metal scripts | Reproducible, one-command startup; service dependency order enforced via healthchecks |
| Writer has no tools | `tools: []` in config | Soft constraint in SOUL.md only | Hard enforcement — the runtime won't provide tool schemas to Writer regardless of SOUL.md |

---

## FAQ

**Q: Walk me through what happens from user query to structured report.**

The user sends a natural-language query to the Hermes runtime. Hermes loads the Researcher agent's SOUL.md as the system prompt and the Researcher's tool list from config.yaml. The model enters a tool-calling loop: it reasons about what it needs to know, calls the appropriate MCP tool, receives structured results, and reasons again. It iterates — typically 2-5 tool calls — until it has sufficient information, then emits a "findings complete" signal. Hermes then loads the Writer agent with its SOUL.md and passes all Researcher findings as context. The Writer, which has no tools, synthesises a structured report with Executive Summary, Key Findings, Detailed Analysis, and Sources, flagging any gaps. The report is returned to the user.

---

**Q: Why two separate agents instead of one agent that does both research and writing?**

Two reasons: cognitive mode separation and hard enforcement. A single agent in a tool-calling loop is optimised for gathering — asking it to simultaneously produce a structured, citeable report degrades both tasks. The model tends to stop researching once it can write a plausible report, even if data is incomplete. More importantly: with one agent, there is no hard way to prevent it from calling tools during the writing phase — you can instruct it not to, but instructions aren't guarantees. With two agents, the Writer's `tools: []` in config means the runtime provides zero tool schemas to the Writer. It cannot call tools even if it tries. The separation is enforced at the infrastructure level, not the prompt level.

---

**Q: How does the model know when to stop researching? What prevents an infinite tool loop?**

Two mechanisms. First, the Researcher's SOUL.md instructs it to stop when it has enough to fully answer the query — the model's own sufficiency judgment. This works well for well-bounded queries but is a soft constraint. Second, Hermes enforces a maximum iteration count (configurable in config.yaml, default 10). If the Researcher exceeds the limit, Hermes terminates the loop and passes whatever findings exist to the Writer, which then flags the incompleteness in the report. In production I would also add tool-call deduplication: if the model calls the same tool with identical arguments twice, the second call is served from cache and the iteration count is not incremented — this prevents the common "search the same thing slightly differently" loop.

---

**Q: How does the model-agnostic config work? What exactly makes it swap from Ollama to OpenAI?**

Hermes constructs an OpenAI-compatible HTTP client using the `model_endpoint` and `model_name` from config.yaml. When it makes a request — whether to `http://localhost:11434/v1` (Ollama) or `https://api.openai.com/v1` (OpenAI) — the request payload is identical: a `POST /chat/completions` with `model`, `messages`, and `tools` fields. Both endpoints return identically structured responses. The OpenAI client spec is the de facto standard for LLM APIs — Ollama, Groq, Together, and others implement it deliberately so tools built for OpenAI work against them with no code changes. The only difference is the base URL and sometimes the API key header. Hermes reads both from config. Changing one line in config.yaml changes where the HTTP request goes.

---

**Q: Why MCP instead of just calling Python functions directly?**

Three reasons. Schema validation: MCP validates every tool call against the declared `input_schema` before the Python function is ever called. If the model hallucinates an argument — passes a string where an array is expected, or omits a required field — the MCP layer rejects it and returns a structured error back to the model. This prevents a class of silent failures where bad arguments reach the function and produce garbage results. Tool discovery: Hermes generates the tool descriptions the model sees directly from the MCP schema — no manual prompt engineering of tool descriptions. Testability: MCP tools are standalone services you can test with curl or a test client without running Hermes or a model. Direct Python function calls would couple tool testing to the agent runtime.

---

**Q: What happens if a tool returns insufficient data? How does the Researcher handle it?**

The Researcher's SOUL.md instructs it to try a different search strategy if a tool returns insufficient data. In practice this means reformulating the query — trying a broader HS code range, querying by commodity name instead of code, or using `fetch_document` to retrieve a source document directly and extracting the relevant section. If all strategies are exhausted and the data still isn't available, the Researcher signals completion with a "data not found" note for that dimension. The Writer then includes this gap explicitly in the report — "Anti-dumping duty status could not be confirmed; manual verification against DGTR notifications is recommended." This is preferable to the Researcher silently passing incomplete findings as if they were complete.

---

**Q: How does memory work across turns? If the user follows up, what context does the agent have?**

Hermes maintains the full conversation history in memory — every user message, every tool call, every tool result, every agent response — as a growing message array. This is the same mechanism as any chat completion API: the `messages` parameter in each request includes the full prior history. When the user sends a follow-up query, Hermes appends it to the existing history and sends the whole thing to the model. The Researcher therefore has full context from the prior turn: it knows which tariff rates it already fetched, which agreements it already checked, and what the user's original intent was. For the common follow-up pattern — "what about X country instead?" — the Researcher can skip re-fetching the destination-side data it already knows and go straight to the new variable.

---

**Q: How would you evaluate report quality? You currently have no systematic eval.**

This is a real gap. The current system has no automated way to measure whether the report is accurate, complete, or well-structured. My approach would be a golden-set eval: build a set of 20-30 known compliance questions where the correct answers are verifiable (e.g., "HS 8507.60, CN→IN, as of 2024-10-01" with verified duty rates from official sources). For each question, run the full pipeline and score the report on: (1) factual accuracy — are the duty rates correct?, (2) completeness — did it find all applicable duties including anti-dumping?, (3) structure adherence — does the report follow the declared sections?, (4) citation quality — is every claim linked to a source?. Run this eval on every model swap and every SOUL.md change. It catches regressions immediately — if switching from qwen2.5 to a smaller model causes the Researcher to miss anti-dumping duties 30% of the time, you know before deploying.

---

## Interview Bridges

These are the conceptual connections that matter in interviews — how this project maps to broader engineering patterns you'd encounter at any company.

### SOUL.md → System prompt management in production AI systems

The SOUL.md pattern solves the same problem every production AI system faces: who owns the system prompt, how do you version it, and who can change it without a deployment? At OpenAI, Anthropic, and every company building AI products, this is a real ops problem. System prompts that live in code require a code deployment to change. System prompts that live in a database can be changed by product managers without engineers, which can break the system silently. SOUL.md is a middle path: the prompt is a file, checked into the repo, subject to PR review and CI, but readable and editable by non-engineers in a GitHub UI. Any production agent system needs this pattern — whether it's called SOUL.md, a prompt template, or a system config.

### Researcher + Writer → MapReduce

This is MapReduce at the agent level. Map: fan out across data sources, collect raw findings, emit structured intermediate results. Reduce: take all intermediate results, collapse to a single coherent output. Google used MapReduce to process web crawl data into search indexes. Hadoop used it for batch analytics. The pattern is identical — separate the gathering phase from the aggregation phase, enforce a clean handoff, let each phase be optimised independently. When someone asks "why two agents?" in an interview, MapReduce is the mental model that makes the answer immediately legible to any engineer.

### MCP typed tools → Function calling validation

MCP's `input_schema` is the same mechanism as OpenAI function calling's `parameters` or Anthropic tool use's `input_schema`. In all three cases, the schema is a JSON Schema object that the API validates before the tool is called. The model cannot pass arguments that don't match the schema — the API rejects them and returns a structured error. This is how LLM systems make tool calls reliable: move validation to the boundary, not into the tool itself. If you've used OpenAI function calling, you've implemented this pattern. MCP just standardises the protocol so the same tool can be called by any runtime, not just OpenAI's.

### Model-agnostic config → Provider abstraction pattern

The `model_endpoint` + `model_name` config pattern is what LiteLLM, LangChain's model wrappers, and AWS Bedrock's unified API all do: provide a single interface that routes to different underlying providers based on configuration. The insight is that the OpenAI chat completions spec is the de facto standard — if you code to that interface, you can route to any provider that implements it. In production systems this matters because: models improve (you want to swap without code changes), pricing changes (you want to move to a cheaper provider), and providers have outages (you want a fallback). The config-based abstraction makes all of these operational concerns zero-code changes.

### Memory across turns → Session memory in any multi-turn AI system

The "growing message array" approach Hermes uses is exactly what ChatGPT, Claude.ai, and every commercial chat product uses. The complication at scale is that the array grows unboundedly — a 100-turn conversation eventually exceeds context windows and token budgets. Production systems handle this with summarisation (compress old turns into a summary, append new turns to that summary) or retrieval (store all turns, retrieve the k most relevant to the current query). The trade-off is faithfulness vs. cost: full history is most faithful, summarisation loses nuance, retrieval is cheap but may miss context. This system uses full history (fine for the short research sessions it's designed for) and would need one of these strategies at scale.

### Tool result caching gap → Idempotent tools + memoisation

The current system makes redundant tool calls when the same tariff code is queried twice in a session. The fix is memoisation keyed by `(tool_name, args_hash)`. This is the same pattern as HTTP caching (cache-control, ETags), GraphQL DataLoader, and React Query's query deduplication: when two callers request the same resource with the same key, serve the first from the origin, serve subsequent ones from cache. For trade compliance data — tariff rates, regulation text, FTA applicability — the data doesn't change within a session, so the cache never needs invalidation. In API design this is the "idempotent operation" pattern: identical inputs always produce identical outputs, so it's always safe to cache.

---

## What-If Scenarios

**"The Researcher gets into a tool loop — calls the same tool 20 times. How do you fix it?"**

This is a real failure mode for agent systems. Three layers of defence:

Layer 1 — Memoisation with loop detection: keep a call log of `(tool_name, args_hash)` for the session. Before executing a tool call, check if it's in the log. If it is, return the cached result and increment a "repeated call" counter. If the repeated call counter exceeds 3 for any single tool, inject a system message into the conversation: "You have called this tool multiple times with the same arguments. The result will not change. Consider whether you have enough information or whether you need a different approach." This often breaks the loop by giving the model explicit feedback.

Layer 2 — Hard iteration cap: Hermes stops the loop after N iterations (configurable, default 10). Whatever findings exist get passed to the Writer, which flags the loop in its report.

Layer 3 — SOUL.md instruction: add "Before each tool call, check if you have already called this tool with these arguments. If so, do not call it again." Soft constraint, but combined with layers 1 and 2 it covers the failure mode. In production I would also add monitoring: alert when any agent exceeds 8 tool calls on a single query — that's a signal of either a bad SOUL.md or a model regression.

---

**"You want to add a third agent: a Fact-Checker that verifies the Writer's claims before publishing. How does the architecture change?"**

The pipeline extends to three stages: Researcher → Writer → Fact-Checker. The Fact-Checker receives the Writer's draft report and the Researcher's raw findings. Its SOUL.md: "You are a trade compliance fact-checker. For each claim in the report, verify it against the Researcher's source data. Flag any claim that is not directly supported by a tool result. Return: [VERIFIED] claims, [UNSUPPORTED] claims, and a revised report with unsupported claims removed or qualified."

The Fact-Checker gets the same MCP tools as the Researcher (or a read-only subset) so it can independently verify a claim if the source data is ambiguous — for example, re-calling `fetch_tariff_db` to confirm a duty rate the Writer stated. The config.yaml gains a third entry: `fact_checker: { soul: ./souls/fact_checker.soul.md, tools: [fetch_tariff_db, search_regulations] }`.

Hermes orchestration changes from a two-step handoff to a three-step one. The key design question is whether the Fact-Checker can send the report back to the Writer for revision (a loop) or simply emits a final report with flags. A loop risks infinite back-and-forth; a single-pass Fact-Checker is simpler and more predictable. I'd start with single-pass and add the revision loop only if the output quality justifies the added latency.

---

**"You need to serve 100 users concurrently. Docker Compose doesn't scale. What's your production architecture?"**

Docker Compose is a single-host tool. For 100 concurrent users you need horizontal scaling across multiple hosts.

**Stateless agent workers:** Hermes runtime containers are stateless between requests — they load SOUL.md and config at startup, handle a request, and return. This means they scale horizontally with no coordination. Deploy Hermes as a Kubernetes Deployment with an HPA (Horizontal Pod Autoscaler) triggered by request queue depth or CPU. 100 concurrent users → 10-20 Hermes pods, depending on model latency.

**Model inference bottleneck:** Ollama running locally in each pod doesn't work — you'd need one GPU per pod. Instead, deploy a shared inference server: one (or a few) GPU nodes running Ollama or vLLM with a load balancer in front. All Hermes pods point to the shared inference endpoint. vLLM is better than Ollama for concurrent requests — it supports continuous batching, which dramatically improves throughput under load.

**Session memory:** Currently Hermes holds conversation history in-process memory. With multiple pods, a user's follow-up request might hit a different pod. Move conversation history to Redis — key by session ID, serialize the message array as JSON. All pods read from and write to the same Redis. TTL the sessions (e.g., 2 hours) to prevent unbounded growth.

**MCP tools:** The MCP tools server also needs to scale — or more likely, it's already stateless (reads from a database, returns results) and can be deployed as multiple replicas behind a load balancer.

**Queue for async reports:** For long-running research queries (5+ tool calls, 30+ second latency), move to async: accept the query, return a job ID immediately, push the job to a Celery + Redis queue, Hermes workers pull jobs and write results to a database, user polls or receives a webhook when the report is ready. This decouples request handling from job execution and lets the system absorb traffic spikes without timing out.

---

## What I'd Do Differently

### 1. Streaming output

The Writer currently produces the full report synchronously — the user sees nothing for 10-30 seconds, then gets the complete report. This is poor UX for longer reports. The fix is streaming: Hermes streams the Writer's tokens to the client as they're generated, using Server-Sent Events or WebSockets. The user sees the Executive Summary appear first, then Key Findings, then Detailed Analysis — the same report, but the perceived latency drops from 30 seconds to "started in under 2 seconds." OpenAI's streaming API (`stream: true`) supports this; Ollama does too. The Hermes runtime would need to proxy the token stream to the client, but this is a 20-line change. The higher engineering value is in the Researcher phase: if the Researcher's reasoning ("I found basic duty 15%, now checking for anti-dumping...") were streamed to the user, they'd have visibility into the research process and could interrupt early if they already know a piece of data.

### 2. Tool result caching with session scope

Every tool call currently makes a fresh HTTP request to the MCP tools server, which queries the database or fetches a document. If the Researcher calls `fetch_tariff_db(hs_code="8507.60", origin="CN", destination="IN")` and then the user asks a follow-up that requires the same data, it's fetched again. An in-session cache keyed by `(tool_name, sha256(sorted(args.items())))` eliminates all redundant fetches. Trade compliance data — tariff schedules, regulation text, FTA applicability — is stable within a session (it changes once a year at budget, not mid-query). The cache never needs invalidation within a session. Beyond eliminating latency, caching makes the loop-detection problem easier: if the model calls the same tool twice, the second call is instant, the loop terminates faster, and the repeated-call counter still triggers the intervention.

### 3. Source credibility scoring

The Writer currently treats all Researcher findings as equally reliable. A result from `fetch_tariff_db` pointing to the CBIC official tariff schedule is more reliable than a result from `fetch_document` parsing a PDF scraped from a trade consultancy's blog. A source credibility score — government DB (0.95), official gazette notification (0.90), regulatory authority website (0.80), third-party source (0.50) — would let the Writer qualify uncertain findings appropriately. "The anti-dumping duty is reported at 20% (source reliability: 0.80 — DGTR website; recommend confirming against official gazette notification)." This is the difference between a research tool and a compliance tool: a compliance officer needs to know not just the finding but how much to trust it.

### 4. Structured output from the Researcher

Currently the Researcher emits natural language: "I found the basic duty is 15% and there is an anti-dumping duty of 20%." The Writer then parses this natural language to structure the report. This is fragile — natural language has ambiguity. A better design: the Researcher emits a structured JSON object (enforced via function calling or structured output mode) with typed fields: `{ "basic_duty": { "rate": 0.15, "source": "CBIC 2024-25", "tool": "fetch_tariff_db" }, "anti_dumping_duty": { "rate": 0.20, "valid_until": "2027-03-31", "notification_no": "ADD-2022-174", "source": "DGTR", "tool": "search_regulations" } }`. The Writer then receives clean structured data instead of natural language, which is both more reliable and easier to validate.

### 5. Systematic evaluation

There is currently no automated way to know if the system is getting better or worse. A golden-set eval is essential: 30 known compliance questions with verified answers drawn from official sources. After any change — model swap, SOUL.md edit, new tool, Hermes update — run the eval and score: factual accuracy (are duty rates correct?), completeness (were all applicable duties found?), structure adherence (does the report follow the declared template?), citation validity (is every claim linked to a source?). This is table stakes for any AI system in production. Without it, you're flying blind — a new model that's cheaper might silently miss anti-dumping duties 40% of the time and you'd only know when a client gets hit with an unexpected duty.

### 6. Confidence-aware report generation

Related to source scoring: the Writer could include an overall confidence level per finding. "Basic customs duty: 15% — HIGH confidence (official tariff schedule). Anti-dumping duty: 20% — MEDIUM confidence (DGTR website; verify against official gazette). No applicable FTA — HIGH confidence (confirmed against ASEAN and bilateral agreement lists)." This gives the compliance officer a triage view: which findings need independent verification before acting on them, and which are already confirmed from authoritative sources. The model can derive this from the source credibility scores — the infrastructure is the same, it just needs to surface the scores to the user rather than using them only internally.

---

## Technology Comparisons

### Orchestration vs Choreography in Multi-Agent Systems

The Trade Compliance Researcher uses two agents: a Researcher and a Writer. The architectural question is how they coordinate.

| Dimension | Orchestration | Choreography |
|---|---|---|
| Control flow | Central controller directs each step | Agents react to events or artifacts from each other |
| Visibility | Explicit — you can read the workflow definition | Implicit — behavior emerges from reactions |
| HITL integration | Natural — pause points are explicit | Hard — no central place to inject a pause |
| Debugging | Easy — trace the controller's state | Hard — must reconstruct causality from event logs |
| Coupling | Agents coupled to the orchestrator | Agents coupled only to the artifact format |
| Single point of failure | Yes — orchestrator failure stops everything | No — agents can restart independently |
| Best for | Known steps, human approval gates, audit trails | Independent agents with stable interfaces, parallel pipelines |
| Examples | LangGraph StateGraph, Temporal workflows | Event-driven microservices, this Researcher+Writer pair |

**When orchestration wins (and when to use LangGraph):** The steps are known in advance, you need HITL approval gates, or you need durable checkpointing so the workflow survives a process restart. QuizMe in the01.dev is a perfect orchestration case — plan objectives → HITL approve → generate questions → HITL submit → grade — every step is known, every pause is defined.

**Why this system uses choreography:** The Researcher and Writer are semantically independent. The Writer needs only one thing from the Researcher: the structured findings artifact. It does not need to be embedded in the same control flow. This means:
- The Researcher model can be swapped (different LLM, different tool set) without touching the Writer.
- The Writer can be improved (better citation format, different output structure) without touching the Researcher.
- Both agents have independent SOULs — different identities, different instructions, different tool access.

If a human approval step between research and synthesis were required (compliance officer reviews raw findings before the Writer synthesises them), orchestration would be the right upgrade. The Researcher returns its artifact, a human approves, and only then does the Writer run. LangGraph's interrupt mechanism would handle this naturally.

**Interview move:** "Choreography here because the coupling point is only the artifact schema — what the Researcher returns. As long as that schema is stable, either agent can evolve independently. The moment we need a human gate between research and synthesis, we'd upgrade to LangGraph orchestration: one interrupt between phases, MemorySaver checkpointer so the pause survives a server restart."

---

## Technical Dictionary

*Plain-English definitions of every term, algorithm, and tool used in this document. If something above confused you, start here.*

---

### Trade & Customs Domain

### HS Code (Harmonised System Code)
An international 6-digit product classification code assigned by the World Customs Organisation and used by customs authorities in nearly every country to identify goods for tariff and regulatory purposes. The first six digits are universal; countries extend them with additional digits for domestic specificity (India uses 8 digits, the United States uses 10). Every tariff rate, anti-dumping duty, and FTA concession is anchored to an HS code.

**Example:** When the Researcher calls `fetch_tariff_db(hs_code="8507.60", origin="CN", destination="IN")`, the HS code `8507.60` tells the database to look up lithium-ion batteries specifically — not batteries in general.

---

### Basic Customs Duty
The standard import tax levied on a product when it crosses into India, expressed as a percentage of its CIF (cost + insurance + freight) value and defined in the Customs Tariff Act. It is the baseline charge that every importer pays before any additional surcharges, IGST, or anti-dumping duties are stacked on top.

**Example:** In the lithium battery trace, `fetch_tariff_db` returns a basic customs duty of 15%, which the Writer uses as the foundation of the effective-rate calculation before adding IGST and the anti-dumping duty.

---

### IGST on Imports
Integrated Goods and Services Tax applied at the point of import in addition to basic customs duty. Unlike basic duty, IGST paid on imports qualifies for Input Tax Credit (ITC), meaning the importer can offset it against their downstream GST liability — so the net cost to a registered business is lower than the headline rate suggests.

**Example:** The Researcher's `fetch_tariff_db` result shows IGST at 18%, which the Writer includes in the landed cost breakdown and notes is recoverable via ITC for registered importers.

---

### Social Welfare Surcharge
An additional duty levied on top of basic customs duty — typically 10% of the basic duty amount — that funds government social welfare schemes. Because it is calculated on the basic duty rather than the CIF value directly, it is often overlooked in quick-and-dirty landed cost estimates.

**Example:** For lithium batteries with a 15% basic duty, the social welfare surcharge is 10% of 15%, yielding an effective 1.5% additional charge — a small but real component that the Writer surfaces in the Key Findings section.

---

### Anti-Dumping Duty
A punitive import tax imposed by a country's trade authority (in India, the DGTR) when a foreign exporter is found to be selling goods below fair market price, harming domestic producers. It is country-specific, product-specific, and time-limited — each notification carries an expiry date — so it must always be checked against current official notifications, not assumed to be static.

**Example:** The Researcher's third iteration calls `search_regulations` and finds notification ADD-2022-174, an active 20% anti-dumping duty on lithium-ion cells from China valid until 2027-03-31, which turns out to be the largest single duty component in the report.

---

### FTA (Free Trade Agreement)
A bilateral or multilateral treaty that reduces or eliminates tariffs between signatory countries, but only for goods that meet the agreement's Rules of Origin criteria. Having an FTA between two countries does not automatically mean every product gets preferential rates; the product must qualify as "originating" from the partner country under the agreement's specific rules.

**Example:** The Researcher calls `get_trade_agreement(countries=["CN", "IN"])` to check whether any FTA could reduce the 15% basic duty on lithium batteries — the result confirms no operative bilateral FTA exists between India and China for this HS code.

---

### Rules of Origin
The criteria defined within a Free Trade Agreement that determine whether a product qualifies as genuinely "originating" from a partner country and therefore eligible for preferential tariff rates. Simply shipping goods through a country or re-exporting them does not qualify; the rules typically require a minimum percentage of value addition or a change in tariff classification to occur within the exporting country.

**Example:** Even if India and China had an FTA, lithium cells assembled in China from Korean cathode materials might not meet the Rules of Origin threshold — the `get_trade_agreement` tool would need to check the specific value-addition requirements, not just the existence of an agreement.

---

### Landed Cost
The total all-in cost of importing a product: purchase price plus freight, insurance, and every applicable duty and charge — basic customs duty, social welfare surcharge, IGST, anti-dumping duty, and any other cess. Landed cost is what the importer actually pays to get the goods into their warehouse, and it is what both the Trade Compliance Researcher and Munshi compute.

**Example:** The Writer's Executive Summary reports a total effective duty of approximately 56.7% for lithium batteries — that figure is the landed cost percentage built from all the components the Researcher gathered across three tool-call iterations.

---

### Infrastructure

### Hermes
The custom AI agent runtime that hosts both the Researcher and Writer agents. Hermes manages loading SOUL.md as the system prompt, maintaining conversation history across turns, routing tool calls to the MCP tools server, enforcing iteration limits, and making OpenAI-compatible HTTP calls to whatever model endpoint is configured. It is the container in which each agent runs.

**Example:** Hermes loads `researcher.soul.md` at startup, passes the user's query and the full tool schema to the model, receives the tool call for `fetch_tariff_db`, executes it against the MCP tools server, and feeds the result back to the model for the next iteration.

---

### SOUL.md
A Markdown file that defines an agent's identity, behavioural rules, and hard constraints. It is loaded verbatim as the agent's system prompt, but because it is a plain file checked into version control rather than a string in source code, it can be read and edited by compliance teams and non-engineers without a code deployment — and every change is tracked in git history.

**Example:** The Researcher's `researcher.soul.md` contains the instruction "Never synthesise or write the final report — that is the Writer's role," which keeps the Researcher in pure-gathering mode regardless of what the user asks in the query.

---

### ReAct Loop (Reasoning + Acting)
The core execution pattern of the Researcher agent: the model reasons about what it needs to know, calls a tool to get that information, reads the result, reasons about what it still needs, and calls another tool — repeating until it judges that it has sufficient information to hand off to the Writer. The name comes from the "Reason + Act" paper that formalised this agent design pattern.

**Example:** In the lithium battery trace the ReAct loop runs three times: reason → call `fetch_tariff_db` → reason → call `get_trade_agreement` → reason → call `search_regulations` → reason → "sufficient, pass to Writer."

---

### MCP (Model Context Protocol)
A typed, schema-validated protocol for exposing tools to an AI model. Each tool declares a name, description, and `input_schema` (a JSON Schema object); the MCP layer validates every tool call the model emits against that schema before the underlying Python function is ever called. If the model passes a wrong argument type or omits a required field, MCP returns a structured error that the model reads and must correct.

**Example:** If the Researcher model attempts to call `fetch_tariff_db` with `origin` as an integer instead of a string, MCP rejects the call with a schema validation error before the database is queried, and the model sees the error in its next context window.

---

### Tool Calling
The mechanism by which an AI model invokes an external function. The model emits a structured tool call containing a function name and typed arguments; the agent runtime (Hermes) intercepts it, executes the function, and injects the result back into the model's context as the next message. The model never directly executes code — it only describes what it wants to call.

**Example:** When the Researcher model emits `search_regulations(hs_code="8507.60", countries=["IN"], query="anti-dumping duty")`, Hermes intercepts that call, passes it to the MCP tools server, waits for the response, and feeds the returned notification data back to the model.

---

### HITL (Human-in-the-Loop)
A design pattern where a system pauses at a defined checkpoint — between pipeline stages, before a consequential action, or after a draft is produced — for a human to review and approve before execution continues. It is the pattern used by the 0.1% DEV QuizMe workflow via LangGraph interrupts, but it is explicitly not implemented in the current Trade Compliance Researcher.

**Example:** A future version of the system could introduce HITL between the Researcher and Writer phases — a compliance officer reviews the raw findings before the Writer synthesises them — but today the handoff is automatic and the Writer produces output without human approval.

---

### max_iterations
A Hermes configuration parameter that sets the maximum number of ReAct loop iterations the Researcher is allowed to execute before Hermes forcibly terminates the loop. It is a hard circuit breaker: once the limit is reached, whatever findings the Researcher has accumulated are passed directly to the Writer, which then flags the incompleteness in its report. The default is 10.

**Example:** If the Researcher enters a loop calling `search_regulations` repeatedly with slightly reformulated queries and never declares sufficiency, Hermes cuts the loop at iteration 10 and passes partial findings to the Writer, which notes "Anti-dumping duty status could not be fully confirmed within the allowed research depth."

---

### Evaluation & Quality

### RAGAS
A RAG (Retrieval-Augmented Generation) evaluation framework that measures two key quality metrics: faithfulness (are the model's claims actually supported by the retrieved context, or did the model hallucinate?) and context precision (were the retrieved chunks genuinely useful for answering the question, or was noise retrieved?). It is identified as a gap in the current Trade Compliance system.

**Example:** A RAGAS faithfulness check on the Writer's report would verify that every duty rate stated — "15% basic duty," "20% anti-dumping" — is directly traceable to a specific Researcher tool result, and flag any claim the Writer introduced that has no grounding in the retrieved findings.

---

### Confidence Scoring
A quality signal assigned to each finding that reflects how certain the system is about that finding, based on source authority, the number of corroborating sources, and the specificity of the data. Confidence scoring is identified as a future improvement in the current system — today all findings are treated as equally reliable regardless of whether they came from an official government database or a scraped third-party PDF.

**Example:** A `fetch_tariff_db` result pointing to the official CBIC tariff schedule would receive a HIGH confidence score (0.95), while a `fetch_document` result parsed from a trade consultancy blog would receive a MEDIUM score (0.50), prompting the Writer to flag that finding for manual verification.

---

### Structured Output
Returning data in a defined, typed format — typically a JSON schema — rather than free-text prose. When the Researcher emits structured output, the Writer receives typed fields with known keys rather than natural language it must parse, eliminating ambiguity and making the synthesis step more reliable and easier to validate programmatically.

**Example:** Instead of the Researcher writing "the basic duty is 15% from CBIC," structured output would yield `{ "basic_duty": { "rate": 0.15, "source": "CBIC Tariff 2024-25", "tool": "fetch_tariff_db" } }` — a typed object the Writer can reference without natural-language parsing.

---

### Infrastructure

### Docker Compose
A tool for defining and running a multi-container application as a single, declarative configuration file. It specifies which containers exist, how they depend on each other, which ports they expose, and which volumes they share — and starts the full stack with a single `docker compose up` command. Health checks in the file ensure containers start in the correct dependency order.

**Example:** The Trade Compliance stack's `docker-compose.yaml` defines three services — `ollama`, `mcp_tools`, and `hermes_runtime` — and uses a health check on `mcp_tools` so that Hermes never starts until the tools server is confirmed ready to accept requests.

---

### Ollama
A tool for running open-source large language models locally on a developer's machine. It provides an OpenAI-compatible HTTP API at `http://localhost:11434/v1`, meaning any code written against OpenAI's client can be pointed at Ollama with only a URL change. It handles model downloading, GPU memory management, and request serving.

**Example:** In development, both the Researcher and Writer agents send their `POST /chat/completions` requests to Ollama running `qwen2.5:7b` locally — no API key, no network latency, no per-token cost.

---

### OpenAI-Compatible API
An HTTP interface that follows OpenAI's request and response format for chat completions and tool calling — specifically `POST /chat/completions` with `model`, `messages`, and `tools` fields returning a `choices` array. Because OpenAI's spec became the de facto industry standard, providers including Ollama, Groq, Together AI, and Azure OpenAI all implement it, meaning any client built for OpenAI works against them with only a base URL change.

**Example:** Hermes constructs identical HTTP payloads whether it is pointed at `http://localhost:11434/v1` (Ollama) or `https://api.openai.com/v1` (OpenAI) — the `model_endpoint` in `config.yaml` is the only thing that changes between a local dev run and a cloud production run.

---

### config.yaml
The Trade Compliance system's single configuration file, which declares the model endpoint, model name, SOUL.md path, and tool list for each agent. Separating these values from code means that swapping model providers, changing agent identity, or adding a tool to an agent's allowlist requires no Python changes — only a file edit and a container restart.

**Example:** Switching the Researcher from the local Ollama `qwen2.5:7b` to OpenAI's `gpt-4o-mini` for a production run requires changing exactly two lines in `config.yaml` — `model_endpoint` and `model_name` — with no changes to Hermes, the MCP tools, or either SOUL.md file.

---

### Stdio Server (FastMCP)
A Model Context Protocol server that communicates over standard input and output rather than HTTP. The FastMCP library makes it straightforward to define MCP tools as decorated Python functions and expose them over stdio; Hermes spawns the tools server as a subprocess and communicates with it through its stdin/stdout pipes. This is lightweight and requires no network port, making it simpler to run locally and in Docker.

**Example:** The Trade Compliance MCP tools server — defining `search_regulations`, `fetch_tariff_db`, `fetch_document`, and `get_trade_agreement` — runs as a FastMCP stdio server that Hermes starts as a child process, so all inter-process communication happens through pipes rather than HTTP calls between containers.
