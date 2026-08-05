# Kalaam — System Design

---

## Quick Reference

| | |
|---|---|
| **Platform** | Programming language in Hindi, Marathi, Bengali, Telugu, Odia |
| **npm package** | `kalaam` v2.3.3 |
| **Website** | kalaamlang.in |
| **Target users** | Tier-3 city students, age 14–18, mobile-first, fully offline |
| **Recognition** | TEDx Bangalore · IEEE Nagpur · 500+ monthly users |
| **Architecture** | Pure JS interpreter · PWA · Custom CodeMirror mode |
| **Dependencies** | Zero runtime dependencies |
| **Test coverage** | 90–95% |

---

## Problem Statement

Programming education in India assumes English fluency and laptop access. For tier-3 city students — no laptop, intermittent internet, no English — every existing tool is inaccessible before they write the first line. There is no "hello world" that speaks to them.

Kalaam is the zeroth step: a programming language in the student's mother tongue, running in the browser on a budget Android phone with zero internet after first load. The design constraint is not performance — it is access. A student who cannot install an app, cannot understand English documentation, and cannot trust a stable internet connection should still be able to learn what a variable is, what a loop does, and what happens when their code runs. Kalaam answers all three.

The broader thesis: the language of instruction is not a soft concern. It is the primary access control. Kalaam removes it.

---

## Functional Requirements

- Write and run programs in Hindi / Marathi / Bengali / Telugu / Odia
- Syntax identical to a simple procedural language — variables, loops, conditionals, functions
- Browser-based IDE with per-language syntax highlighting
- Learning Mode: step-by-step replay of how the interpreter evaluated each line, in the student's language
- Fully offline after first visit (no requests, no server, no API)
- Published as an npm package (`kalaam`) with a stable public API for downstream integration

---

## Non-Functional Requirements

- **Zero runtime dependencies** — the npm package ships no dependencies; works in any JS environment
- **Offline-first** — service-worker cached; zero network requests after first load
- **Mobile-first** — runs on a ₹5,000 Android phone browser with no native app install
- **Language-agnostic parser** — adding a new language requires zero parser changes
- **Test coverage:** 90–95% across lexer, scanner, runtime, and regression scenarios
- **Installable** — Add to Home Screen on Android; behaves like a native app

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    kalaamlang.in  (PWA)                     │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  CodeMirror Editor                                   │  │
│  │  (custom kalaam-{language} syntax mode)              │  │
│  └──────────────────────┬───────────────────────────────┘  │
│                         │  sourcecode + languageKeywords    │
│                         ▼                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Compile(sourcecode, languageKeywords)               │  │
│  │                                                      │  │
│  │  Phase 1: Cleaning   ── language-aware               │  │
│  │       ↓                                              │  │
│  │  Phase 2: Scanning   ── character-level scan         │  │
│  │       ↓                                              │  │
│  │  Phase 3: Tokenizing ── 20 Push* functions           │  │
│  │       ↓                                              │  │
│  │  Phase 4: Interpretation ── walk tokens, run ops     │  │
│  │       ↓                                              │  │
│  │  Phase 5: Output     ── assemble result object       │  │
│  └──────────────────────┬───────────────────────────────┘  │
│                         │                                   │
│             { output, ExecutionStack[], isError, TimeTaken }│
│                         │                                   │
│            ┌────────────┴──────────────┐                   │
│            ▼                           ▼                    │
│   ┌─────────────────┐      ┌─────────────────────────┐    │
│   │  Output Panel   │      │  Learning Mode Panel    │    │
│   │  (program out)  │      │  (ExecutionStack replay) │    │
│   └─────────────────┘      └─────────────────────────┘    │
│                                                             │
│  Service Worker (sw.js) — cache-first, all assets offline  │
└─────────────────────────────────────────────────────────────┘

npm package `kalaam`
  └── Compile()  ──  same 5-phase pipeline, zero deps
      usable in Node.js / browser / React Native / Deno
```

---

## Detailed Component Design

### 1. Interpreter Pipeline — 5 Phases

The interpreter is a linear 5-phase pipeline. The critical design principle: **only Phase 1 knows about language**. Every subsequent phase operates on normalised tokens and is completely language-agnostic. This is what makes adding a new language trivially cheap.

---

#### Phase 1: Cleaning (Language-Aware)

The only phase that knows about language. It performs two operations:
1. `earlyCleaning()` — strip comments, normalize whitespace, handle encoding edge cases
2. Keyword substitution — replace every language-specific keyword with a normalised internal token

```javascript
function earlyCleaning(source, languageKeywords) {
  // keyword substitution: language-specific → normalised tokens
  // "यदि" (Hindi if)    → "__IF__"
  // "जर"  (Marathi if)  → "__IF__"
  // "যদি" (Bengali if)  → "__IF__"
  // "అయితే" (Telugu if) → "__IF__"
  return substituted_source
}
```

After this phase, the source is a string of normalised tokens. Phases 2–5 see no language at all.

**Why substitution in Phase 1 and not in the parser:**
- Centralises all language knowledge in one place — a single map per language
- Parser is written once, tested once, works for all 5 languages
- Adding a language = one keyword map entry, zero parser changes, zero new test files
- Parser bugs are language-agnostic — one fix covers all languages

**Adding a language:**
```javascript
const LANGUAGE_MAPS = {
  hindi:   { 'यदि': '__IF__', 'अन्यथा': '__ELSE__', 'के लिए': '__FOR__', 'प्रिंट': '__PRINT__', ... },
  marathi: { 'जर': '__IF__',  'नाहीतर': '__ELSE__',  'साठी': '__FOR__',  'दाखवा': '__PRINT__',  ... },
  bengali: { 'যদি': '__IF__', 'অন্যথা': '__ELSE__',  'জন্য': '__FOR__',  'মুদ্রণ': '__PRINT__', ... },
  telugu:  { 'అయితే': '__IF__', 'లేకపోతే': '__ELSE__', 'కోసం': '__FOR__', 'చూపించు': '__PRINT__', ... },
  odia:    { 'ଯଦି': '__IF__', 'ନଚେତ': '__ELSE__',   'ପାଇଁ': '__FOR__', 'ଦେଖାଅ': '__PRINT__',   ... },
  // Adding Gujarati: one new entry here — zero changes anywhere else in the codebase
}
```

---

#### Phase 2: Scanning

Character-level scan of the normalised source string. Produces `cleaned_sourcedata[]`.

```
normalised source string (output of Phase 1)
    │
character-by-character scan
    │
cleaned_sourcedata[]  — array of characters with position metadata
                        [{char: '_', pos: 0}, {char: '_', pos: 1}, ...]
```

Handles: whitespace normalisation, multi-character token boundary detection, string literal detection (preserves content inside quotes), comment stripping if any remain.

The scanner's job is purely structural — it does not assign meaning. It produces the character stream that Phase 3 will group into tokens.

---

#### Phase 3: Tokenizing

Groups the `cleaned_sourcedata[]` character stream into typed token objects.

```
cleaned_sourcedata[]
    │
20 Push* functions + TypeChecking pass
    │
tokens[]  — array of typed token objects

Example — after scanning "x = 5 + 3":
    { type: 'IDENTIFIER', value: 'x',    line: 1, col: 1 }
    { type: 'OPERATOR',   value: '=',    line: 1, col: 3 }
    { type: 'NUMBER',     value: 5,      line: 1, col: 5 }
    { type: 'OPERATOR',   value: '+',    line: 1, col: 7 }
    { type: 'NUMBER',     value: 3,      line: 1, col: 9 }

Example — after scanning "__IF__ x > 10":
    { type: 'KEYWORD',    value: '__IF__', line: 3, col: 1 }
    { type: 'IDENTIFIER', value: 'x',     line: 3, col: 8 }
    { type: 'OPERATOR',   value: '>',     line: 3, col: 10 }
    { type: 'NUMBER',     value: 10,      line: 3, col: 12 }
```

Each `Push*` function handles one token category:
- `PushKeyword` — normalised tokens like `__IF__`, `__FOR__`, `__PRINT__`
- `PushIdentifier` — variable and function names
- `PushNumber` — integer and float literals
- `PushString` — string literals (content preserved verbatim)
- `PushOperator` — arithmetic, comparison, assignment operators
- ... (20 total, one per token category)

`TypeChecking` runs after tokenization — validates token sequences for structural correctness. Catches syntax errors at the token level before any execution is attempted. Produces clear error messages with line and column.

---

#### Phase 4: Interpretation

Walks `tokens[]` sequentially. Executes each statement. Builds `ExecutionStack[]` as it goes.

```javascript
function interpret(tokens) {
  const memory = {}  // flat variable store — intentionally no scope
  const stack  = []  // ExecutionStack — every operation appends here
  let output   = ''

  for (const token of tokens) {
    switch (token.type) {

      case '__ASSIGN__':
        const value = evaluate(rhs, memory)
        memory[varName] = value
        stack.push({ op: 'ASSIGN', var: varName, value, line: token.line })
        break

      case '__IF__':
        const result = evaluate(condition, memory)
        stack.push({ op: 'IF_EVAL', condition, result, line: token.line })
        if (!result) skipToElseOrEnd()
        break

      case '__PRINT__':
        const val = evaluate(argument, memory)
        output += val + '\n'
        stack.push({ op: 'PRINT', value: val, line: token.line })
        break

      case '__FOR__':
        // loop body appends one entry per iteration to ExecutionStack
        break

      // every operation type appends to stack
    }
  }

  return { memory, stack, output }
}
```

**Memory model:** Single flat object `{}`. Variable names are string keys. No scope isolation — intentionally. The target age group (14–18, no prior CS exposure) finds scoping rules confusing before they understand variables. Kalaam postpones that complexity. Variables are global, functions can see and modify them freely. This is a pedagogical decision, not a technical limitation.

---

#### Phase 5: Output

Assembles the final result object from Phase 4's return values and timing data.

```javascript
return {
  output:         collectedOutput.join('\n'),  // program's printed output
  ExecutionStack: stack,                        // full execution trace
  isError:        errorOccurred,               // true if any runtime error
  TimeTaken:      endTime - startTime          // milliseconds
}
```

This is the object that `Compile()` returns. The UI uses all four fields: `output` for the output panel, `ExecutionStack` for Learning Mode, `isError` to decide whether to show an error state, `TimeTaken` for the subtle performance indicator.

---

### 2. ExecutionStack — Learning Mode

Every operation the interpreter performs appends one record to `ExecutionStack[]`. The stack is the complete execution history of the program — every assignment, every condition check, every loop iteration, every print.

```javascript
// After evaluating: x = 5 + 3
// (sourcecode was in Hindi: "x = 5 + 3" — syntax is shared)
stack.push({
  line:        1,
  sourceText:  'x = 5 + 3',
  operation:   'ASSIGN',
  variable:    'x',
  value:       8,
  explanation: 'x को 8 दिया गया (5 + 3 = 8)'   // generated in user's language
})

// After evaluating: __IF__ x > 5
stack.push({
  line:        3,
  sourceText:  'यदि x > 5',
  operation:   'IF_EVAL',
  condition:   'x > 5',
  result:      true,
  memory:      { x: 8 },                        // memory snapshot at this point
  explanation: 'शर्त सच है — x (8) 5 से बड़ा है'
})
```

**UI replay:**
1. Student runs the program
2. Learning Mode button appears in the output panel
3. UI replays `ExecutionStack[]` step by step, one record at a time
4. Each step: highlights the source line, shows memory snapshot, shows the explanation in the student's language
5. Student can step forward and backward through execution history — a full rewind of how the computer thought

**Why this matters pedagogically:**

The interpreter teaches itself. No teacher, no textbook, no English explanation required to understand "what did this line do?" — the student can replay the interpreter's reasoning at their own pace, in their own language. This is the zeroth learning tool: not just "your output is wrong," but "here is exactly what the computer did with your code, step by step, and here is what your memory looked like after each step."

The `ExecutionStack` is also what makes Kalaam distinctive beyond "a language in Hindi." Any language can translate keywords. Only Kalaam makes the interpreter's reasoning inspectable and replayable in the student's language.

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
  output:         string,   // program output (stdout equivalent)
  ExecutionStack: Array,    // full execution trace — one record per operation
  isError:        boolean,  // true if a runtime error occurred
  TimeTaken:      number    // milliseconds — interpreter wall time
}
```

**Zero runtime dependencies.** The package has no `dependencies` in `package.json`. Any JavaScript environment can run Kalaam — Node.js, browser, React Native, Deno, Bun. No transitive dependency risk, no supply chain surface, no `node_modules` bloat for downstream users.

**Two distribution formats:**
- ESM for modern bundlers (tree-shakeable)
- CJS for Node.js compatibility (Babel transform in the package build)

**The API is intentionally minimal.** Two inputs, one output object. Callers do not need to manage interpreter state — each `Compile()` call is stateless and deterministic. Same input always produces same output.

---

### 4. PWA — Offline Architecture

```
First visit (internet required):
kalaamlang.in
    │
    ├── Browser fetches: HTML, CSS, JS bundle, CodeMirror, Kalaam interpreter
    │
    └── Service Worker (sw.js) installs:
            ├── caches HTML shell
            ├── caches JS bundle (includes interpreter + all language maps)
            ├── caches CodeMirror + custom syntax modes
            └── caches all static assets

All subsequent visits (no internet required):
Browser request → Service Worker intercepts → serves from cache
    Zero network requests. Zero server involved.
```

**Cache strategy:** Cache-first for all static assets. The interpreter, editor, language maps, and UI assets are cached at install time. A student can open their phone browser, with airplane mode on, and write and run code. Same experience as online.

**Why PWA over a native Android app:**
- No app store account required — a common barrier in tier-3 cities (Google Play requires a card, a phone number, sometimes age verification)
- Works on any Android browser: Chrome, Firefox, Samsung Internet — no specific runtime
- Updates are transparent — next time internet is available, the service worker fetches a new cache version in the background, no user action
- Installable to home screen on Android via "Add to Home Screen" prompt — the student gets an app icon, full-screen launch, no browser chrome; looks and feels native
- One codebase serves both the web and the "app" — no Kotlin, no Play Store review cycle

---

### 5. Syntax Highlighting — Custom CodeMirror Mode

Standard CodeMirror does not know about Hindi keywords. A custom syntax mode is defined per language so that `यदि` highlights as a keyword exactly like `if` does in JavaScript mode.

```javascript
CodeMirror.defineMode('kalaam-hindi', function() {
  const keywords = ['यदि', 'अन्यथा', 'के लिए', 'प्रिंट', 'कार्य', 'वापस']
  return {
    token: function(stream) {
      for (const kw of keywords) {
        if (stream.match(kw)) return 'keyword'
      }
      if (stream.match(/\d+/)) return 'number'
      if (stream.match(/\"[^\"]*\"/)) return 'string'
      stream.next()
      return null
    }
  }
})
```

Mode is dynamically selected when the student switches language — the editor re-tokenizes with the new mode. One mode definition per language, registered at startup.

This is a thin layer — it maps surface syntax to CSS classes (`cm-keyword`, `cm-number`, `cm-string`). The interpreter does not use CodeMirror's token output; it runs the `Compile()` pipeline independently on the raw source string.

---

### 6. Testing Strategy

**90–95% coverage across:**
- **Substitution tests:** Every keyword in every language map → correct normalised token
- **Scanner tests:** Character sequences → correct `cleaned_sourcedata[]` arrays
- **Tokenizer tests:** Character arrays → correct typed `tokens[]` for every token category
- **Runtime tests:** Variable assignment, arithmetic (all operators), conditionals (true/false branches), loops (fixed count, while-style), functions (call, return)
- **ExecutionStack tests:** Verify correct record appended for each operation type — correct fields, correct values, correct line reference
- **Regression tests:** Canonical programs with known outputs — any interpreter change that breaks these is a regression
- **Error handling tests:** Undefined variables, type mismatches, malformed syntax, timeout detection for infinite loops

**What is NOT tested (~5–10%):**
- CodeMirror syntax highlighting (UI concern — modes are tested visually, not in Jest)
- Service worker caching behaviour (integration concern — requires a browser; hard to unit test in CI)
- Performance on specific physical devices (manual QA on a ₹5,000 Android phone)
- Cross-browser rendering differences (manual QA)

This gap is deliberate. The interpreter logic is the core product. UI and caching behaviour change less often and have higher cost-per-test than value. The 90–95% coverage gives confidence that the interpreter is correct; the remaining surface is covered by manual testing before release.

---

### 7. Key Engineering Decisions

| Decision | Choice | Alternative Considered | Reason |
|---|---|---|---|
| Language handling | Phase 1 keyword substitution | Language-specific parsers | One parser for all languages; adding a language is one map entry |
| Package runtime deps | Zero | Use `chevrotain`, `nearley`, or similar parser libs | Works anywhere offline; no supply chain risk; no transitive installs |
| Distribution | PWA | Native Android app | No app store required; any browser; transparent updates; one codebase |
| Memory model | Flat object `{}` | Scoped environments (closure-based) | Pedagogical simplicity — scope adds confusion before variables make sense |
| Learning Mode | ExecutionStack replay | Step debugger / DevTools | Works offline, works on mobile, no DevTools needed, in student's language |
| Testing | Jest + Babel | Browser-based tests | Fast CI, isolated from UI, pure interpreter logic coverage |
| Syntax highlighting | Custom CodeMirror mode | Regex-based dumb highlighting | Proper tokenization, extensible per language, integrates with editor UX |
| API design | Stateless `Compile(src, lang)` | Stateful interpreter object | Simplest possible surface; no lifecycle management for callers |

---

## FAQ

**Q: Walk me through all 5 interpreter phases with an example like `x = 5 + 3`.**

Assume the student writes this in Hindi context (though the syntax `x = 5 + 3` is shared across languages — only keywords are localized):

Phase 1 (Cleaning): `earlyCleaning()` runs. The line `x = 5 + 3` has no language keywords, so it passes through unchanged. Substitution would fire if this were `यदि x > 5` — `यदि` becomes `__IF__`.

Phase 2 (Scanning): Character-level scan produces `cleaned_sourcedata[]` — roughly `['x', ' ', '=', ' ', '5', ' ', '+', ' ', '3']` with position metadata.

Phase 3 (Tokenizing): The 20 `Push*` functions group characters into typed tokens:
```
{ type: 'IDENTIFIER', value: 'x', line: 1 }
{ type: 'OPERATOR',   value: '=', line: 1 }
{ type: 'NUMBER',     value: 5,   line: 1 }
{ type: 'OPERATOR',   value: '+', line: 1 }
{ type: 'NUMBER',     value: 3,   line: 1 }
```

Phase 4 (Interpretation): The interpreter walks tokens, sees `IDENTIFIER = EXPR`, evaluates `5 + 3 = 8`, writes `memory['x'] = 8`, appends to `ExecutionStack[]`:
```javascript
{ op: 'ASSIGN', var: 'x', value: 8, line: 1, explanation: 'x को 8 दिया गया' }
```

Phase 5 (Output): Assembles `{ output: '', ExecutionStack: [...], isError: false, TimeTaken: 2 }`. No print in this program, so output is empty. ExecutionStack has the one ASSIGN record.

---

**Q: Why keyword substitution in Phase 1 instead of at the parser level?**

The parser-level alternative would be: give the parser a vocabulary object, and at every point where it expects a keyword, check if the current token matches any of the language-specific words. This works, but it means the parser has language awareness throughout its code — conditional checks at every grammar rule.

Phase 1 substitution eliminates that entirely. After Phase 1, the source is in a single normalised language. The parser is a pure function of normalised tokens. It has no `if (language === 'hindi')` anywhere.

The practical benefit: adding Gujarati takes one day (write the keyword map, run the substitution tests). With parser-level handling, adding Gujarati means auditing every grammar rule in the parser and ensuring the new tokens are handled. One approach scales to 10 languages easily; the other becomes a maintenance problem.

---

**Q: How does adding a new language like Gujarati work in practice? Show me the change.**

Literally one object entry in `LANGUAGE_MAPS`:

```javascript
const LANGUAGE_MAPS = {
  // existing entries unchanged
  hindi:   { 'यदि': '__IF__', 'अन्यथा': '__ELSE__', ... },
  marathi: { 'जर': '__IF__',  'नाहीतर': '__ELSE__',  ... },

  // new entry:
  gujarati: {
    'જો': '__IF__',
    'નહીંતો': '__ELSE__',
    'માટે': '__FOR__',
    'છાપો': '__PRINT__',
    'કાર્ય': '__FUNCTION__',
    'પાછો': '__RETURN__',
    // ... all keywords
  }
}
```

Then:
1. Add substitution tests for `gujarati` keyword map (one test file, same template as existing language tests)
2. Define a `kalaam-gujarati` CodeMirror mode (copy an existing mode, swap the keyword list)
3. Add "Gujarati" to the language selector UI dropdown

Zero changes to: the scanner, the tokenizer, the interpreter, the ExecutionStack logic, the output assembly, the npm API surface, or any existing language's behaviour.

---

**Q: Your interpreter has 90–95% test coverage. What's the 5–10% you're NOT testing and why?**

Three categories:

**CodeMirror syntax highlighting:** The custom modes are tested visually — I load a language, type a keyword, verify it highlights. This is not in the Jest suite because CodeMirror requires a DOM, which Jest does not have without jsdom setup. The risk is low: a broken syntax mode degrades the editor experience but does not break execution. The interpreter does not use CodeMirror's token output.

**Service worker caching:** Cache-first behaviour requires a browser context with a real service worker lifecycle. Mocking it in Jest would test the mock, not the behaviour. I test this manually before each release: open in Chrome DevTools > Application > Service Workers, verify assets cached, go offline, reload.

**Physical device performance:** The interpreter running a 1000-iteration loop on a ₹5,000 Android phone is not something Jest can measure. I test this manually. The design constraint (pure JS, no heavy deps) keeps performance acceptable on low-end hardware, but it is not quantitatively guaranteed.

The 90–95% that is tested covers everything that could silently produce wrong output — substitution, scanning, tokenizing, interpretation correctness, ExecutionStack record correctness. The untested surface degrades UX but does not corrupt correctness.

---

**Q: What's the design tradeoff of a flat memory model vs scoped environments?**

A scoped environment (like JavaScript's closures or Python's LEGB rule) gives you:
- Variable isolation: a function's locals don't pollute the caller's scope
- Correct recursion: each call frame has its own `x`, `i`, etc.
- The ability to implement closures, higher-order functions

A flat memory model gives you:
- Simplicity: one mental model — there is one `memory{}`, variables go in, variables come out
- Transparency: the student can always see what every variable is; nothing is hidden in a scope
- Pedagogical clarity: the concept of "a variable holds a value" is not muddied by "but only in this scope"

The cost of flat memory: recursive functions break. If a function `factorial(n)` calls `factorial(n-1)`, both calls share the same `memory['n']`. The inner call overwrites `n` with `n-1`, and the outer call sees the modified value on return. The result is wrong.

This is a known limitation. For the target audience (first exposure to programming, variables, loops, conditionals), recursion is not the first lesson. The flat model is correct for the scope of what Kalaam teaches. It would need to change before Kalaam could teach functional programming or any non-trivial recursion.

---

**Q: Why PWA instead of a native Android app for your target audience?**

Four concrete reasons:

First, Google Play account barriers. Creating a Google account to download an app requires a phone number and sometimes a credit/debit card for age verification. Tier-3 city students often share a family SIM on a feature phone, and the family's primary smartphone may not have a Google account set up. A browser URL has no such barrier.

Second, app store review latency. A bug fix to a native app takes 1–3 days through Play Store review. A PWA update deploys in minutes — the service worker fetches the new cache version on next network contact, invisibly, without user action.

Third, device compatibility. A native APK targets a specific Android API level. Samsung Internet, older Chrome builds, and budget phone manufacturers sometimes ship modified Chromium builds. The PWA runs on whatever browser the phone has. The target audience often cannot upgrade their browser either.

Fourth, no install friction. "Go to kalaamlang.in in your browser" is a simpler instruction than "open the Play Store, search, install, grant permissions." The PWA's "Add to Home Screen" prompt bridges the gap — the student gets an app icon and full-screen launch after visiting once.

The tradeoff: PWAs have weaker push notification support and cannot access some native device APIs. Kalaam does not need either.

---

**Q: How does Learning Mode work technically? Walk me through what ExecutionStack contains.**

During Phase 4 (Interpretation), every operation that executes appends one record to `ExecutionStack[]`. The record captures: what happened, what line it happened on, what the memory looked like after, and a human-readable explanation generated in the student's chosen language.

For a program like:
```
x = 10
y = 3
z = x + y
प्रिंट z
```

The `ExecutionStack` after execution:
```javascript
[
  {
    line: 1, sourceText: 'x = 10',
    op: 'ASSIGN', variable: 'x', value: 10,
    memoryAfter: { x: 10 },
    explanation: 'x को 10 दिया गया'
  },
  {
    line: 2, sourceText: 'y = 3',
    op: 'ASSIGN', variable: 'y', value: 3,
    memoryAfter: { x: 10, y: 3 },
    explanation: 'y को 3 दिया गया'
  },
  {
    line: 3, sourceText: 'z = x + y',
    op: 'ASSIGN', variable: 'z', value: 13,
    memoryAfter: { x: 10, y: 3, z: 13 },
    explanation: 'z को 13 दिया गया (x=10 + y=3 = 13)'
  },
  {
    line: 4, sourceText: 'प्रिंट z',
    op: 'PRINT', value: 13,
    explanation: '13 स्क्रीन पर दिखाया गया'
  }
]
```

The UI presents a step-by-step replay: on Step 1, highlight line 1, show `memory = { x: 10 }`, show the Hindi explanation. Student clicks Next — Step 2 highlights line 2, memory updates, new explanation appears. They can go back to Step 1. The full execution history is always available. On a mobile screen with no teacher in the room, this is the closest equivalent to a debugger and a tutor combined.

---

**Q: The package has zero runtime dependencies. What's the cost of that decision?**

The benefit is clear: no transitive risk, works anywhere, offline, no supply chain. The costs are real:

**You reimplement things.** A parser combinator library like `chevrotain` or `nearley` would give you a formal grammar definition, error recovery, and tested performance. Writing the tokenizer from scratch with 20 `Push*` functions works, but it is more code to maintain and more potential for subtle bugs in edge cases that the library's test suite would have caught.

**Error recovery is harder.** Parser libraries typically include error recovery strategies — skip to the next statement, report multiple errors per run. A hand-written tokenizer tends to fail on the first error. Kalaam currently stops on the first syntax error. For a beginner editor, this is actually fine (show one error at a time, fix it, try again). But it would be a limitation for a production language.

**Performance tuning is manual.** Profiled parser libraries are often faster than hand-written alternatives. For Kalaam's programs (tens to hundreds of lines), this does not matter. For a 10,000-line program, it would.

**The net assessment:** For Kalaam's use case, zero deps is the right call. The scope of programs students write fits well within a hand-written interpreter. The supply chain and offline benefits outweigh the maintenance cost of owning the parser. If Kalaam were a general-purpose language targeting production use, the calculus would be different.

---

## Interview Bridges

These bridges connect Kalaam's design decisions to well-known patterns in production systems. Use them when the interviewer asks "so what?" or "how does this relate to real-world systems?"

**5-phase interpreter → compilers and interpreters in production:**
Every production language runtime uses a similar pipeline. Python's CPython: source → tokenize → parse → compile to bytecode → interpret bytecode. V8 (JavaScript): parse → AST → Ignition (interpreter) → Turbofan (JIT compiler). Kalaam's pipeline is a simplified version of the same pattern. The interviewer asking "how does your interpreter work?" is asking you to describe a fundamental CS concept. Kalaam gives you a system you built and understand end-to-end, rather than a textbook answer.

**Keyword substitution in Phase 1 → preprocessing and macro expansion:**
The C preprocessor runs before the compiler and performs text substitution (`#define` → replacement text). SQL query rewriting in databases rewrites a logical query to a physical plan before execution. Kalaam's Phase 1 is the same pattern: transform the input into a canonical form before the main processing pipeline sees it. The benefit — centralised transformation, simpler downstream — applies in all three cases.

**ExecutionStack Learning Mode → debuggers and step-through execution:**
VS Code's debugger shows a call stack, local variables, and lets you step forward/backward. It works by intercepting the runtime's execution events and recording state at each step. Kalaam's `ExecutionStack` is the same mechanism — the interpreter fires events (ASSIGN, IF_EVAL, PRINT) and records state at each one. The difference is that Kalaam's trace is complete (all steps), while most debuggers are on-demand (you set breakpoints). Production observability tools like OpenTelemetry use the same idea: instrument the code path, record spans, replay them later.

**PWA offline-first → service worker cache strategies:**
Any offline-capable web app uses service workers with a cache strategy. The options are: cache-first (Kalaam's choice), network-first (always try network, fall back to cache), stale-while-revalidate (serve cache immediately, update in background). Kalaam uses cache-first because correctness (the interpreter the student tests against must be the same as what's cached) matters more than having the latest version on every load. Knowing these strategies and when to pick each one is directly applicable to any web app that needs offline support.

**Zero runtime dependencies → supply chain security:**
The npm left-pad incident (2016): a developer unpublished a small package that hundreds of projects depended on transitively. Thousands of builds broke. Kalaam's zero-deps decision eliminates this surface. In production security reviews, transitive dependency counts matter: each `npm install` adds risk. The argument for zero deps in a library context — especially one that runs arbitrary code on mobile devices — is the same argument security teams make about supply chain risk. Knowing this incident and the argument it produced is expected knowledge for any Node.js engineer.

**Flat memory model → tradeoffs in beginner programming language design:**
Scratch uses a flat variable model intentionally — global variables are the first mental model, scope comes later. Logo, BASIC, early Python tutorials often delay scope. The academic research on this (Pea, Soloway, Spohrer) shows that scope is a significant conceptual hurdle. Kalaam's flat model is a deliberate pedagogical tradeoff, not a technical oversight. In an interview, framing this as a conscious decision with known costs and benefits (you know recursion breaks) is stronger than either defending it as perfect or apologising for it as a shortcut.

---

## What-If Scenarios

**"A student tries to write a recursive function. What breaks with the flat memory model?"**

With a flat memory model, all variables are global. When a function calls itself, both the outer and inner invocations share the same `memory{}` object. Consider:

```
कार्य factorial(n)
  यदि n = 0
    वापस 1
  वापस n * factorial(n - 1)
```

Call `factorial(3)`:
- Outer call sets `memory['n'] = 3`
- Inner call `factorial(2)` sets `memory['n'] = 2` — overwrites
- Inner call `factorial(1)` sets `memory['n'] = 1` — overwrites
- Inner call `factorial(0)` sets `memory['n'] = 0` — overwrites
- Returns 1
- `factorial(1)` tries to compute `1 * factorial(0)`, but `memory['n']` is now 0
- All bets are off — the result is wrong

The fix is function-level scope: each call gets its own `memory` frame. Before calling a function, push the current frame onto a call stack; after returning, pop and restore. This is how every production language with first-class functions works. It is the right solution. Kalaam deliberately deferred it because scope is a harder concept than variables, and the target audience needs variables first.

**"You want to add a REPL mode (live execution as you type). What changes in the interpreter?"**

A REPL executes code as the student types, giving immediate feedback. The changes required:

First, the UI needs a debounce — fire `Compile()` not on every keypress but after a short pause (300–500ms). This is a UI-only change, no interpreter change needed.

Second, the interpreter needs partial compilation tolerance. Currently, mid-statement code (typing `x = 5` mid-way through `x = 5 + 3`) will fail the TypeChecking phase with a syntax error. The REPL should not show an error for incomplete input. Options: catch syntax errors and show nothing (silent failure), or add a partial-parse mode that attempts to execute complete statements and stops gracefully at incomplete ones.

Third, the `ExecutionStack` will grow with every keypress. The UI replay model assumes a complete program; a REPL fires it incrementally. You would need to decide: clear the stack on each recompile (fresh run) or append (history mode). Fresh run is simpler and correct for a REPL.

Fourth, persistent memory between REPL lines is a design decision. A true REPL keeps memory between evaluations — `x = 5` on line 1, then `print(x)` on line 2 works because the session's memory is preserved. The current stateless `Compile()` call produces a fresh `memory{}` each time. For a REPL, you would need to pass an initial memory state in or make the interpreter stateful.

The core interpreter pipeline does not change. The work is in the UI debounce, the error tolerance for partial input, and the memory persistence model.

**"Kalaam needs to support 10 new Indian languages in the next quarter. Walk me through the work required."**

India has 22 scheduled languages plus hundreds of regional ones. Adding 10 new languages to Kalaam has three cost buckets:

**Cheap (interpreter):** One keyword map per language in `LANGUAGE_MAPS`. If the language has unique keywords for `if`, `else`, `for`, `print`, `function`, `return` — that is roughly 20–30 entries. One engineer, one day per language, including substitution tests. 10 languages = 10 days of engineering.

**Medium (editor):** One CodeMirror mode per language. Same template as existing modes, swap the keyword list and Unicode character ranges for identifiers. One engineer, half a day per language. This is testable visually.

**Hard (content and validation):** The keyword maps must be linguistically correct. `यदि` is correct Hindi for "if" but a Konkani speaker will have a different word, and it matters to get it right — wrong keywords would feel condescending or incorrect to native speakers. This requires community review or a linguist per language. This is the actual bottleneck. The engineering is the easy part.

**Infrastructure consideration:** 10 new languages means 10 new CodeMirror modes bundled in the JS. Bundle size increases. At some point (maybe 15+ languages), lazy-loading modes by selected language becomes worth the complexity. The current architecture loads all modes at startup because 5 modes is negligible; 15 modes deserves measurement.

The 10-language expansion is feasible in a quarter with the right community partnerships. The interpreter architecture will not block it.

---

## What I'd Do Differently

- **Scope isolation:** Add function-level scope to the memory model — currently all variables are global, which causes silent bugs in recursive functions. The fix is a call stack: push a new frame on function entry, pop on return. I deferred this because scope is a harder concept than variables for the target age group, but it would need to be addressed before Kalaam could teach intermediate topics.

- **Error messages in the student's language:** Currently runtime and syntax errors are in English ("undefined variable: x"). A student who cannot read English gets no actionable information. A Hindi error message ("अपरिभाषित चर: x — इस नाम का कोई variable नहीं बना है") would close this gap. The error string generation is localizable using the same keyword map infrastructure — it would be straightforward to add.

- **REPL mode:** Live execution as you type (with debounce) is the fastest feedback loop for learning. Watching a variable update in real time as you change its assignment teaches the concept faster than write-compile-read cycles. The interpreter supports it — the UI work is adding debounce, partial-parse tolerance, and a memory persistence model between REPL lines.

- **Hosted eval fallback:** For computationally heavy programs (large loops, recursive functions once scope is added), browser execution on a ₹5,000 Android phone will lag. A lightweight serverless eval endpoint — AWS Lambda or Cloudflare Workers — could serve as an optional fallback for programs that exceed a local time threshold. The offline-first default is preserved: try local, if it times out, offer to run on server (if internet available). This keeps the experience good for simple programs on any device while not hard-blocking advanced programs on low-end hardware.

- **More languages:** The architecture supports a new language with one keyword map entry. Gujarati, Tamil, and Kannada are the obvious next candidates by speaker population and tier-3 city distribution. The bottleneck is not engineering — it is finding community reviewers to validate the keyword choices are linguistically correct and culturally appropriate.

- **Formal grammar definition:** The current tokenizer is hand-written with 20 `Push*` functions. It works and is well-tested, but a formal grammar (using a PEG grammar library or similar) would be self-documenting, easier to extend with new syntax, and would provide better error recovery out of the box. The zero-deps constraint rules out adding a parser library to the npm package, but a build-time grammar compiler that emits the hand-written tokenizer's equivalent code would preserve the zero-dep output while giving the developer-experience benefits of a formal grammar.

---

## Technology Comparisons

### Interpreter vs Compiler vs Transpiler vs Bytecode VM

For a project that is itself a programming language, interviewers will ask: "What kind of language implementation is this, and why?"

| Model | How it works | Build step | Execution speed | Tracing support | Examples |
|---|---|---|---|---|---|
| Tree-walking interpreter | Walks tokens/AST at runtime, executes directly | None | Slowest | Native — every step is visible | Kalaam, early Ruby |
| Bytecode VM | Compiles to bytecode ahead of time, VM executes bytecode | Yes (compile) | Fast | Possible but indirect | Python (CPython), JVM |
| Native compiler | Translates to machine code | Yes (compile) | Fastest | Requires debug symbols | C, Rust, Go |
| Transpiler | Compiles to another high-level language | Yes (transpile) | Depends on target runtime | Via source maps | TypeScript, Babel |

**When a bytecode VM wins:** You need better performance and portability. Python's CPython compiles to `.pyc` bytecode before execution — the VM runs bytecode, not raw source text.

**When a native compiler wins:** Maximum execution speed, systems programming, or when the language is used for performance-critical applications.

**For Kalaam:** Tree-walking interpreter because **Learning Mode is a first-class product feature, not a debug artifact**. Every operation appends to `ExecutionStack[]` — this is what the UI replays to teach the student how the interpreter evaluated their program. A bytecode or compiled model would require adding explicit instrumentation to produce this trace; a tree-walking interpreter produces it naturally as it executes. Additionally: zero build step matters on mobile (no compile phase before running a program), and student programs are small (tens of lines) so raw execution speed is irrelevant.

**Interview move:** "The design principle was: the execution trace is the product, not the output. Everything else follows from that. A tree-walking interpreter gives you the trace for free. A compiled model gives you speed — which a 14-year-old's first 20-line program doesn't need."

---

### PWA vs React Native vs Native App

Kalaam targets students who access the internet primarily on a low-end Android phone, often with intermittent connectivity.

| Dimension | PWA | React Native | Native (Android/iOS) |
|---|---|---|---|
| Installation | URL → browser → Add to Home Screen | Play Store / App Store download | Play Store / App Store download |
| Install barrier | None — works in browser before install | Must find, download, install (~50MB) | Must find, download, install (~100MB+) |
| Offline support | Service worker + Cache API | AsyncStorage + offline-capable | Full device storage access |
| Device API access | Limited (no Bluetooth, limited camera) | Full native API access | Full native API access |
| Codebase | One web codebase | One JS codebase (bridges to native) | Two codebases (Kotlin + Swift) |
| Update mechanism | Deploy to web server — instant | Play Store review cycle | Play Store review cycle |
| Cost to ship update | Immediate | Days (review) | Days (review) |

**When React Native wins:** You need deep device integration (camera, Bluetooth, push notifications, offline sync with native storage), or the app experience must be indistinguishable from native.

**When native wins:** Maximum performance, platform-specific UI conventions, and features that require native APIs with no web equivalent.

**For Kalaam:** PWA was not a trade-off — it was the only sensible choice for this problem. The target demographic (tier-3 city students, intermittent internet, low-end Android) faces a meaningful barrier at every step of the Play Store install flow: find the app, have storage space, download over cellular. A PWA is a URL — the teacher pastes it in a WhatsApp group and the student opens it immediately in Chrome. After first load, the service worker caches the entire app (interpreter, editor, language data). Every subsequent visit is fully offline. The interpreter is pure JavaScript: it was already running in the browser. PWA was the natural architecture for a zero-install, zero-server, offline-capable JavaScript interpreter.

**Interview move:** "React Native would have solved a problem we didn't have — deep device integration — while creating a problem we didn't want — installation friction for students who may never complete a Play Store download. PWA gave us offline capability and instant access in one architecture."

---

## Technical Dictionary

*Plain-English definitions of every term, algorithm, and tool used in this document. If something above confused you, start here.*

---

### Interpreter & Language Theory

### Interpreter
A program that reads source code and executes it directly, line by line, without first producing a separate compiled binary. It translates and runs the code at the same time, which makes it slower at runtime than compiled code but faster to iterate during development.

**Example:** Kalaam's `Compile()` function is an interpreter — it takes a student's Hindi source code and executes it immediately in the browser, with no separate compilation step.

---

### Compiler
A program that translates source code entirely into machine code (or bytecode) before any execution happens. The translation is a separate, one-time step; after that the compiled output runs fast without re-reading the original source.

**Example:** Kalaam is not a compiler — there is no compiled artifact; the source is re-interpreted on every `Compile()` call. If Kalaam were a compiler, it would produce a binary file the phone could run directly.

---

### Tokenization / Lexing
The first step in parsing: break raw source text into a flat list of typed pieces called tokens (numbers, identifiers, operators, keywords). The tokenizer (also called a lexer) does not care about grammar or meaning — it only recognises the smallest meaningful units in the text.

**Example:** In Kalaam's Phase 3, the character stream `x = 5 + 3` is broken into five tokens: `IDENTIFIER("x")`, `OPERATOR("=")`, `NUMBER(5)`, `OPERATOR("+")`, `NUMBER(3)` — before the interpreter knows what any of it means.

---

### Parser
A program that takes the token list produced by the lexer and builds a structured representation by applying grammar rules. It understands that `x = 5 + 3` means "assign the result of adding 5 and 3 to the variable x," not just a sequence of symbols.

**Example:** Kalaam's Phase 4 acts as a simple parser — it walks the token list and recognises patterns like `IDENTIFIER OPERATOR(=) EXPR` as an assignment statement, then executes it.

---

### AST (Abstract Syntax Tree)
A tree data structure where each node represents a language construct: an assignment, an addition, a function call, a conditional. Compilers and interpreters typically build an AST as an intermediate form between tokenization and execution, making it easier to analyse or optimise the program before running it.

**Example:** Kalaam does not build an explicit AST — it interprets directly from the token list in a single pass. A future version that needed optimisation or formal error recovery would likely add an AST between Phase 3 and Phase 4.

---

### Interpretation Phase
The step in which the token list (or AST) is actually executed: variables are looked up, arithmetic is performed, functions are called, output is produced. This is where the program "runs" — all prior phases only prepare the input for this step.

**Example:** Kalaam's Phase 4 is the interpretation phase — it walks the `tokens[]` array, evaluates each statement against `memory{}`, and appends an entry to `ExecutionStack[]` for every operation it performs.

---

### Memory Model
How a language runtime stores and retrieves variables during execution. The choice of memory model determines whether variables can "see" each other across function calls and whether recursion works correctly.

**Example:** Kalaam uses a flat key-value object as its memory model — all variables live in one shared `{}` object with no scoping. This means a student's variable `x` is always visible everywhere, which simplifies the mental model but breaks recursive functions.

---

### Scoped Environments
An alternative memory model where each function call gets its own isolated variable space (a "scope" or "frame"). Variables in one scope do not affect variables in another. This is required for correct recursion and is how JavaScript, Python, and most production languages work.

**Example:** Kalaam deliberately avoids scoped environments — the target audience of 14–18 year-olds finds the concept of scope confusing before they have mastered what a variable is. Scope would need to be added before Kalaam could correctly teach recursive functions.

---

### Keyword Substitution
A preprocessing technique that replaces language-specific surface tokens (e.g. Hindi `यदि`, Marathi `जर`) with a single normalised internal token (e.g. `__IF__`) before any parsing happens. Everything downstream of the substitution step is language-agnostic.

**Example:** In Kalaam's Phase 1, `earlyCleaning()` replaces every language-specific keyword with its normalised form so that Phase 2 through Phase 5 never need to know which human language the student wrote in.

---

### ExecutionStack
Kalaam's step-by-step trace of every operation the interpreter performs during a single run. Each entry records the operation type, the values involved, a memory snapshot, the source line, and a human-readable explanation in the student's language. The UI replays it as a visual lesson.

**Example:** When a student runs `x = 5 + 3` in Hindi mode, the `ExecutionStack` gains one entry: `{ op: 'ASSIGN', var: 'x', value: 8, explanation: 'x को 8 दिया गया (5 + 3 = 8)' }` — which the Learning Mode panel then displays step by step.

---

### PWA & Offline

### PWA (Progressive Web App)
A web application that uses browser APIs to behave like a native installed app — it can work offline, be added to the home screen, and launch full-screen without a browser address bar. No app store is required; the user just visits a URL.

**Example:** kalaamlang.in is a PWA — a student on a budget Android phone can visit it once with internet, tap "Add to Home Screen," and from that point run Kalaam programs in airplane mode exactly as if it were a native app.

---

### Service Worker
A JavaScript file that the browser installs separately from the main page and runs in the background. It intercepts all network requests the page makes and can serve responses from a local cache instead of the network, enabling offline functionality.

**Example:** Kalaam's `sw.js` service worker intercepts every request for the interpreter bundle, CodeMirror, and language maps — serving them from cache on every visit after the first, so the app works with zero network connectivity.

---

### Cache-First Strategy
A service worker caching strategy where the browser serves a resource from its local cache immediately, without checking the network at all. The network is only contacted if the resource is not in the cache. This makes the app load instantly and work fully offline.

**Example:** Kalaam uses cache-first for all static assets — when a student opens the app, the service worker serves the interpreter and editor from cache without making a single network request, even if the phone has no signal.

---

### Cache API
The browser API for storing and retrieving HTTP request/response pairs locally on the device. The service worker uses it to cache an app's assets at install time and serve them later when the network is unavailable.

**Example:** When a student visits kalaamlang.in for the first time, the service worker uses the Cache API to store the HTML shell, JS bundle, and language maps — everything Kalaam needs to run — so subsequent visits require no download.

---

### Offline-First
A design philosophy where the application is built to work without internet as the primary, default case — not as a degraded fallback. The network is treated as an enhancement (for fetching updates) rather than a requirement.

**Example:** Kalaam is offline-first by design: the interpreter, editor, and all language data are cached at first load, and no feature of the app makes a network request after that. A student with intermittent internet still gets the full experience.

---

### Packaging & Quality

### npm Package
A reusable JavaScript library published to the npm registry (npmjs.com), the standard repository for JavaScript packages. Anyone can install it with `npm install <name>` and import its functions into their own project.

**Example:** Kalaam's interpreter is published as the `kalaam` npm package (v2.3.3) — a developer building an educational app could `import { Compile } from 'kalaam'` and embed Kalaam's execution engine in their own product.

---

### Zero Runtime Dependencies
A package that has no entries in the `dependencies` field of its `package.json`. It ships no libraries that downstream consumers must also install. Only `devDependencies` (for testing and tooling) are present, and those are not installed by consumers.

**Example:** Installing `kalaam` downloads exactly one package — the `kalaam` package itself. No transitive libraries, no risk of a dependency being unpublished or compromised, and the interpreter works in any JavaScript environment without needing a specific ecosystem.

---

### ESM (ECMAScript Module)
The modern JavaScript module system using `import` and `export` keywords, standardised in ES2015. ESM modules are statically analysable, which allows bundlers to "tree-shake" (remove unused exports) and produce smaller output files.

**Example:** Kalaam's npm package ships in ESM format, so a project using only the `Compile` function can be bundled without including any other Kalaam utilities — keeping the final JavaScript bundle as small as possible for mobile users.

---

### Jest
A JavaScript testing framework developed by Meta. It provides a test runner, assertion library, and code coverage reporting in one package, with no browser required — tests run in Node.js via a simulated DOM environment.

**Example:** Kalaam's 90–95% test coverage is measured and enforced by Jest — every substitution, scanner output, and tokenizer result is verified by Jest assertions, and the suite runs in CI on every commit.

---

### CodeMirror
A browser-based code editor component that provides syntax highlighting, line numbers, keyboard shortcuts, and an extension API. It renders inside a `<div>` on any web page and is the editor behind many popular online coding tools.

**Example:** The Kalaam IDE uses CodeMirror as its editor surface, with a custom syntax mode registered per language so that `यदि` highlights as a keyword in exactly the same colour and style as `if` does in a JavaScript file.

---

### Syntax Highlighting Mode
A CodeMirror plugin (called a "mode") that tells the editor how to tokenize source code and which CSS class to assign to each token type — keywords get one colour, strings another, numbers another. A custom mode is required for any language CodeMirror does not know about by default.

**Example:** Kalaam defines one CodeMirror mode per supported language (e.g. `kalaam-hindi`, `kalaam-marathi`). Each mode lists that language's keywords so the editor colours them correctly as the student types.

---

### Semantic Versioning (semver)
A version numbering convention in the format `MAJOR.MINOR.PATCH`. A MAJOR bump signals a breaking change (existing code may need updates). A MINOR bump adds functionality in a backward-compatible way. A PATCH bump fixes bugs without changing the API.

**Example:** The `kalaam` package is on v2.3.3 — the `2` means there was at least one breaking change since the original release (perhaps a change to the `Compile()` return shape), `3` minor features have been added since then, and 3 patch fixes have been applied to the current minor version.

---

### Accessibility & Impact

### Tier-3 City
Cities in India that fall outside the major metros (Mumbai, Delhi, Bangalore, Hyderabad, Chennai) and the established Tier-2 cities. They typically have lower internet reliability, lower English literacy rates, higher proportion of mobile-only internet users, and fewer physical computing resources like laptops or computer labs.

**Example:** Kalaam is designed specifically for tier-3 city students — the offline-first PWA architecture, mobile-first UI, and mother-tongue instruction language all directly address constraints that are common in tier-3 cities but rarely prioritised by mainstream edtech products.

---

### TEDx Talk
An independently organised event licensed by TED that follows the TED format — short, idea-focused talks recorded and published online. TEDx talks go through a local organising committee rather than the central TED organisation, making them more accessible to regional speakers and topics.

**Example:** Swanand gave a TEDx talk at TEDx Bangalore about Kalaam and the case for teaching programming in Indian mother tongues — the talk reaches an audience of educators and policymakers who can advocate for this approach beyond the platform itself.
