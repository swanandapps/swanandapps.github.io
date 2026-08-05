# Kalaam — System Design

**Platform:** Programming language in Hindi, Marathi, Bengali, Telugu, and Odia  
**Published:** npm package `kalaam` v2.3.3 · kalaamlang.in  
**Target:** Tier-3 city students, age 14–18, mobile-first, fully offline  
**Recognition:** TEDx Bangalore · IEEE Nagpur · 500+ monthly users

---

## Problem Statement

Programming education in India assumes English fluency and laptop access. For tier-3 city students — no laptop, intermittent internet, no English — every existing tool is inaccessible before they even write the first line. Kalaam is the zeroth step: a programming language in the student's mother tongue, running in the browser on a budget Android phone with zero internet after first load.

---

## Functional Requirements

- Write and run programs in Hindi / Marathi / Bengali / Telugu / Odia
- Syntax identical to a simple procedural language — variables, loops, conditionals, functions
- Browser-based IDE with syntax highlighting for each language
- Learning Mode: step-by-step replay of how the interpreter evaluated each line
- Fully offline after first visit
- Published as an npm package (`kalaam-core`) with a public API for downstream use

## Non-Functional Requirements

- **Zero runtime dependencies** — the npm package has no dependencies
- **Offline-first** — service-worker cached; zero requests after first load
- **Mobile-first** — runs on a ₹5,000 Android phone browser
- **Language-agnostic parser** — adding a new language requires no parser changes
- **Test coverage:** 90–95% across lexer, parser, runtime, and regression scenarios

---

## High-Level Architecture

```
kalaamlang.in (PWA)
    │
    ├── CodeMirror editor (custom Kalaam syntax mode per language)
    │
    ├── Compile button
    │       │
    │       ▼
    │   Compile(sourcecode, languageKeywords)
    │       │
    │       ├── Phase 1: Cleaning (language-aware)
    │       ├── Phase 2: Scanning
    │       ├── Phase 3: Tokenizing
    │       ├── Phase 4: Interpretation
    │       └── Phase 5: Output
    │       │
    │       ▼
    │   { output, ExecutionStack[], isError, TimeTaken }
    │
    ├── Output panel — shows program output
    └── Learning Mode panel — replays ExecutionStack[] line by line
```

---

## Detailed Component Design

### 1. Interpreter Pipeline — 5 Phases

#### Phase 1: Cleaning (Language-Aware)

The only phase that knows about language.

```javascript
function earlyCleaning(source, languageKeywords) {
  // keyword substitution: language-specific → normalised tokens
  // "यदि" (Hindi if) → "__IF__"
  // "जर"  (Marathi if) → "__IF__"
  // "যদি" (Bengali if) → "__IF__"
  return substituted_source
}
```

After this phase, the source is in normalised tokens. All subsequent phases are identical for all languages.

**Why substitution in Phase 1 and not in the parser:**
- Centralises language knowledge in one place
- Parser is written once, works for all languages
- Adding a new language = one keyword map entry, zero parser changes
- Testing the parser doesn't require multilingual test cases

**Adding a language:**
```javascript
const LANGUAGE_MAPS = {
  hindi:   { 'यदि': '__IF__', 'अन्यथा': '__ELSE__', 'के लिए': '__FOR__', ... },
  marathi: { 'जर': '__IF__',  'नाहीतर': '__ELSE__',  'साठी': '__FOR__',  ... },
  bengali: { 'যদি': '__IF__', 'অন্যথা': '__ELSE__',  'জন্য': '__FOR__',  ... },
  // Adding Gujarati: one new entry here, zero code changes anywhere else
}
```

#### Phase 2: Scanning

Character-level scan of the normalised source.

```
normalised source string
    │
character-by-character scan
    │
cleaned_sourcedata[]  — array of characters with position metadata
```

Handles: whitespace normalisation, comment stripping, string literal detection.

#### Phase 3: Tokenizing

```
cleaned_sourcedata[]
    │
20 Push* functions + TypeChecking pass
    │
tokens[]  — typed token objects
    {type: 'KEYWORD', value: '__IF__', line: 3, col: 5}
    {type: 'IDENTIFIER', value: 'x', line: 3, col: 9}
    {type: 'OPERATOR', value: '>', line: 3, col: 11}
    {type: 'NUMBER', value: 10, line: 3, col: 13}
```

Each `Push*` function handles one token category (PushKeyword, PushIdentifier, PushNumber, PushString, etc.). TypeChecking validates token sequences — catches syntax errors at the token level before interpretation.

#### Phase 4: Interpretation

Walk `tokens[]`, execute statements, build `ExecutionStack[]`.

```javascript
function interpret(tokens) {
  const memory = {}  // variable store
  const stack = []   // ExecutionStack — append every operation
  
  for (const token of tokens) {
    switch(token.type) {
      case '__ASSIGN__':
        memory[varName] = value
        stack.push({ op: 'ASSIGN', var: varName, value, line: token.line })
        break
      case '__IF__':
        const result = evaluate(condition, memory)
        stack.push({ op: 'IF_EVAL', condition, result, line: token.line })
        // branch based on result
        break
      // ... all operations append to ExecutionStack
    }
  }
  return { memory, stack }
}
```

**Memory model:** Single flat object `{}`. Variable names are keys. No scope isolation (simplified for pedagogical clarity — scope would add complexity without learning benefit for the target age group).

#### Phase 5: Output

```javascript
return {
  output: collectedOutput.join('\n'),
  ExecutionStack: stack,
  isError: errorOccurred,
  TimeTaken: endTime - startTime
}
```

---

### 2. ExecutionStack — Learning Mode

Every operation appends a record to `ExecutionStack[]` during interpretation.

```javascript
// After evaluating: x = 5 + 3
stack.push({
  line: 1,
  sourceText: 'x = 5 + 3',
  operation: 'ASSIGN',
  variable: 'x',
  value: 8,
  explanation: 'x को 8 दिया गया (5 + 3 = 8)'  // generated in user's language
})
```

**UI replay:**
1. Student runs program
2. Learning Mode button appears
3. UI replays `ExecutionStack[]` step by step
4. Each step highlights the source line, shows what changed in memory, explains in the student's language
5. Student can go forward/backward through execution history

**Why this is pedagogically significant:** The interpreter teaches itself. No teacher required to explain "what did this line do?" — the student can see the interpreter's reasoning step by step. This is the zeroth learning tool: not just "your output is wrong," but "here's what the computer did with your code."

---

### 3. Public npm API

```javascript
import { Compile } from 'kalaam'

const result = Compile(
  sourcecode,          // string — student's code in any supported language
  languageKeywords     // object — the language map (from LANGUAGE_MAPS)
)

// Returns:
{
  output: string,          // program output
  ExecutionStack: Array,   // full execution trace
  isError: boolean,        // true if runtime error occurred
  TimeTaken: number        // milliseconds
}
```

**Zero runtime dependencies.** The package has no `dependencies` in package.json. Any JavaScript environment can run Kalaam — Node.js, browser, React Native, Deno.

**Two distribution formats:**
- ESM for modern bundlers
- CJS for Node.js compatibility (Babel transform in the package)

---

### 4. PWA — Offline Architecture

```
kalaamlang.in
    │
Service Worker (sw.js)
    │ on first load:
    ├── cache: HTML, CSS, JS bundle, CodeMirror, Kalaam interpreter
    │
    │ on subsequent loads:
    └── serve from cache — zero network requests
```

**Cache strategy:** Cache-first for all static assets. The interpreter, editor, and all language maps are cached at install time. Students can open their phone browser, no internet, write and run code — same experience as online.

**Why PWA over native app:**
- No app store account required (common barrier in tier-3 cities)
- Works on any Android browser (Chrome, Firefox, Samsung Internet)
- Update is transparent — next time internet is available, new cache version installed in background
- Installable to home screen on Android (Add to Home Screen prompt)

---

### 5. Syntax Highlighting — Custom CodeMirror Mode

Standard CodeMirror doesn't know about Hindi keywords. A custom syntax mode was written per language:

```javascript
CodeMirror.defineMode('kalaam-hindi', function() {
  return {
    token: function(stream) {
      if (stream.match('यदि'))  return 'keyword'
      if (stream.match('अन्यथा')) return 'keyword'
      // ... all Hindi keywords
      stream.next()
      return null
    }
  }
})
```

Mode is dynamically selected based on the language the student chooses.

---

### 6. Testing Strategy

**90–95% coverage across:**
- **Parser tests:** Every language keyword substitution for all 5 languages
- **Runtime tests:** Variable assignment, arithmetic, conditionals, loops, functions
- **ExecutionStack tests:** Verify correct entries generated for each operation type
- **Regression tests:** Known programs with known outputs — any interpreter change must not break these
- **Error handling tests:** Malformed input, infinite loops (timeout detection), undefined variables

**What is NOT tested (~5-10%):**
- CodeMirror syntax highlighting (UI concern, not interpreter logic)
- Service worker caching behaviour (integration concern, hard to unit test)
- Performance on specific devices (manual QA)

---

### 7. Key Engineering Decisions

| Decision | Choice | Alternative | Why |
|---|---|---|---|
| Language handling | Phase 1 substitution | Language-specific parsers | One parser for all languages, extensible |
| Package deps | Zero | Use parser libraries | Works anywhere, offline, no supply chain risk |
| Distribution | PWA | Native Android app | No app store required, any browser, easy updates |
| Memory model | Flat object | Scoped environments | Pedagogical simplicity — scope adds confusion for beginners |
| Learning Mode | ExecutionStack replay | Step debugger | Works offline, no DevTools needed, works on mobile |
| Testing | Jest + Babel | Browser-based tests | Fast CI, isolated from UI concerns |

---

### 8. What I'd Do Differently

- **Scope isolation:** Add function-level scope to the memory model — currently all variables are global, which causes subtle bugs in recursive functions
- **Error messages in the student's language:** Currently errors are in English. A Hindi error message ("अपरिभाषित चर: x") would be more helpful for the target audience
- **REPL mode:** Live execution as you type — beginner-friendly but requires debounced interpreter calls
- **More languages:** The architecture supports it with one keyword map entry — Gujarati, Tamil, Kannada are natural next candidates
- **Hosted eval:** For advanced programs (recursion, large loops), browser execution can lag. A lightweight serverless eval endpoint as fallback would help — keeping offline-first as default
