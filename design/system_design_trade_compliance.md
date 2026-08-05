# Trade Compliance Researcher — System Design

**Platform:** Multi-agent trade compliance research system  
**Stack:** Hermes runtime, MCP tools, Ollama, Docker Compose, Python  
**Pattern:** Researcher → Writer two-agent workflow, model-agnostic

---

## Problem Statement

Trade compliance research is time-intensive: finding applicable regulations, tariff codes, import/export restrictions, and duty rates for specific goods across jurisdictions. A human researcher spends hours querying multiple databases, reading documents, and synthesising findings into a structured report. The system automates this research loop with two specialised agents — one that gathers, one that writes.

---

## Functional Requirements

- Accept a natural-language trade compliance query
- Researcher agent autonomously gathers data via MCP tools (iteratively, as needed)
- Writer agent synthesises findings into a structured report with citations
- Memory persists across turns — agents remember prior conversation context
- Swap model providers (Ollama local → OpenAI cloud) with one config change
- Fully containerised — deployable with `docker compose up`

## Non-Functional Requirements

- **Model-agnostic:** Any OpenAI-compatible endpoint works — Ollama, OpenAI, Groq, Anthropic
- **No hardcoded plumbing:** Agent identity and capabilities declared in config, not code
- **Extensible tools:** New MCP tools added without modifying agent logic

---

## High-Level Architecture

```
User query
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  Hermes Runtime                                     │
│                                                     │
│  ┌──────────────────┐    findings    ┌───────────┐  │
│  │ Researcher Agent │ ─────────────► │  Writer   │  │
│  │  (SOUL.md)       │                │  Agent    │  │
│  │                  │                │ (SOUL.md) │  │
│  │  tool loop:      │                └─────┬─────┘  │
│  │  → search regs   │                      │        │
│  │  → fetch tariffs │                      │        │
│  │  → read docs     │                      │        │
│  └──────────────────┘                      │        │
│                                            │        │
└────────────────────────────────────────────┼────────┘
                                             │
                                             ▼
                                    Structured Report
                                    (citations, sources)
```

---

## Detailed Component Design

### 1. Agent Identity — SOUL.md Pattern

Each agent has a `SOUL.md` file declaring its identity, persona, constraints, and operating boundaries.

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
- Agent identity is data, not code — editable without deployment
- Enforces separation of concerns at the prompt level
- Both agents use the same Hermes runtime — differentiated only by SOUL.md
- Reviewable by non-engineers (compliance teams can audit agent instructions)

---

### 2. config.yaml — Model-Agnostic Endpoint

```yaml
researcher:
  soul: ./souls/researcher.soul.md
  model_endpoint: http://localhost:11434/v1  # Ollama local
  model_name: qwen2.5:7b
  tools: [search_regulations, fetch_tariff_db, fetch_document]

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

One line. Zero code changes. Hermes reads the endpoint from config and constructs the OpenAI-compatible client. Any OpenAI-compatible API works: Ollama, OpenAI, Groq, Together, Anthropic (via compatibility layer).

---

### 3. MCP Tools — Researcher's Toolbox

Tools are defined as MCP servers — typed, schema-validated, callable by the model.

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
- `get_trade_agreement(countries)` — check applicable FTAs / bilateral agreements

**Why MCP over direct function calls:**
- Schema validation — model cannot pass malformed arguments (reduces hallucinated tool calls)
- Tool discovery — Hermes auto-generates tool descriptions for the model from the schema
- Testable independently — each tool is a standalone service

---

### 4. Agent Execution Flow

```
User: "What are the import duties for lithium batteries from China to India under HS code 8507.60?"

Hermes → Researcher agent

Researcher loop iteration 1:
  reason: "I need to find the applicable HS code duty rate"
  tool: fetch_tariff_db(hs_code="8507.60", origin="CN", destination="IN")
  result: {basic_duty: 15%, igst: 18%, cess: 0%}

Researcher loop iteration 2:
  reason: "I should check if any FTA applies between China and India for this code"
  tool: get_trade_agreement(countries=["CN", "IN"])
  result: {agreement: null, note: "India-China have no applicable FTA for this code"}

Researcher loop iteration 3:
  reason: "Check for any recent anti-dumping duty notifications"
  tool: search_regulations(hs_code="8507.60", countries=["IN"], query="anti-dumping")
  result: {notifications: [{notification_no: "...", duty: 20%, valid_until: "2027-03-31"}]}

Researcher: "I have sufficient information. Passing to Writer."

Writer agent:
  input: all Researcher findings + sources
  output: structured report (Executive Summary + Key Findings + Detailed Analysis + Sources)
```

**Memory persistence:** Hermes maintains the full conversation history across turns. If the user follows up — "What about Bangladesh instead of China?" — the Researcher has context from the previous query and doesn't re-fetch already-known data.

---

### 5. Docker Compose Deployment

```yaml
version: '3.8'
services:
  ollama:
    image: ollama/ollama
    volumes: [ollama_models:/root/.ollama]
    ports: ['11434:11434']

  mcp_tools:
    build: ./tools
    ports: ['8001:8001']
    environment:
      - REGULATION_DB_PATH=/data/regulations.db
    volumes: [./data:/data]

  hermes_runtime:
    build: ./hermes
    depends_on: [ollama, mcp_tools]
    environment:
      - CONFIG_PATH=/app/config.yaml
    volumes: [./souls:/app/souls, ./config.yaml:/app/config.yaml]
    ports: ['8000:8000']
```

`docker compose up` starts the full stack: Ollama (model inference), MCP tools server, Hermes runtime with both agents.

---

### 6. Key Engineering Decisions

| Decision | Choice | Alternative | Why |
|---|---|---|---|
| Agent identity | SOUL.md files | Hardcoded system prompts | Editable without deployment, auditable by non-engineers |
| Model config | config.yaml endpoint | Hardcoded model name | One line to swap providers — dev uses Ollama, prod uses GPT |
| Tool schema | MCP typed tools | Direct Python function calls | Schema validation prevents malformed tool calls |
| Agent separation | Researcher + Writer | Single agent | Separation of concerns — prevents Writer from calling tools |
| Memory | Hermes conversation history | Stateless per-query | Context carries across follow-up queries |
| Deployment | Docker Compose | Manual setup | Reproducible, one-command startup |

---

### 7. What I'd Do Differently

- **Streaming output:** Currently the Writer produces the full report synchronously. Streaming the report section by section would improve perceived responsiveness.
- **Tool result caching:** Fetching the same tariff code twice in one session makes redundant API calls. An in-session cache keyed by `(tool_name, args_hash)` would eliminate redundant fetches.
- **Confidence scoring:** The Writer currently treats all Researcher findings as equally reliable. A source credibility score (government DB > scraped website) would let the Writer qualify uncertain findings.
- **Evaluation:** No systematic eval of report quality. A golden set of known compliance questions with expected answers would let us measure regression when swapping models.
