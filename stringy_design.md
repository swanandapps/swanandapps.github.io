# stringy-core — System Design

**Author:** Swanand Kadam
**npm:** `stringy-core` v1.0.0
**Tagline:** Lodash but for strings

---

## Table of Contents

1. [What Is stringy-core](#1-what-is-stringy-core)
2. [Architecture Overview](#2-architecture-overview)
3. [Module Breakdown](#3-module-breakdown)
4. [API Design — Two Usage Patterns](#4-api-design--two-usage-patterns)
5. [Key Design Decisions](#5-key-design-decisions)
6. [Contribution Model](#6-contribution-model)
7. [Toolchain](#7-toolchain)
8. [Known Issues](#8-known-issues)
9. [Scaling Considerations](#9-scaling-considerations)

---

## 1. What Is stringy-core

`stringy-core` is a zero-dependency JavaScript string utility library with 50+ functions across 9 categories: case manipulation, cleaning, formatting, masking/security, extraction, validation, transformations, specialized operations, and generation.

It was built as an open-source community library with a "Contribution Station" model — the repo ships stub functions with examples and invites developers to implement and PR them. This made it a learning project as much as a utility.

---

## 2. Architecture Overview

```
Consumer (app.js)
       │
       │  import { maskEmail } from 'stringy-core'
       │  import { _s }        from 'stringy-core'
       ▼
┌──────────────────────────────────────────────────┐
│  index.js  (ESM entry point)                     │
│                                                  │
│  import * as textCaseManipulation    ──────┐     │
│  import * as textCleaning            ──────┤     │
│  import * as textFormatting          ──────┤     │
│  import * as textMaskingAndSecurity  ──────┤     │
│  import * as textMetadataAndExtract  ──────┼──▶  const _s = { ...all }
│  import * as textAnalysisAndValid    ──────┤     export { _s }
│  import * as textTransformations     ──────┤
│  import * as textSpecializedOps      ──────┤     export * from each module
│  import * as textGeneration          ──────┘     (named tree-shakeable exports)
└──────────────────────────────────────────────────┘
       │
       ▼
lib/
├── textCaseManipulation.js
├── textCleaning.js
├── textFormatting.js
├── textMaskingAndSecurity.js
├── textMetadataAndExtraction.js
├── textAnalysisAndValidation.js
├── textTransformations.js
├── textSpecializedOperations.js
└── textGeneration.js
```

---

## 3. Module Breakdown

### textCaseManipulation.js
| Function | Input | Output |
|---|---|---|
| `capitalize` | `'hello world'` | `'Hello world'` |
| `camelCase` | `'Hello world'` | `'helloWorld'` |
| `snakeCase` | `'Hello World'` | `'hello_world'` |
| `toTitleCase` | `'hello world'` | `'Hello World'` |
| `toAlternateCase` | `'hello'` | `'HeLlO'` |
| `invertCase` | `'Hello'` | `'hELLO'` |
| `swapCase` | `'Hello'` | `'hELLO'` |

### textCleaning.js
Whitespace management: `trimStart`, `trimEnd`, `normalizeWhitespace` (collapses runs of whitespace to single space), `removeWhitespace` (strips all whitespace).

### textFormatting.js
All formatting uses the browser-native `Intl` API — no custom number/date parsing.

| Function | Purpose |
|---|---|
| `formatNumber` | Inserts locale commas: `1234567` → `'1,234,567'` |
| `formatCurrency` | Locale currency formatting with configurable locale + currency code |
| `formatDate` | ISO date strings → locale date display |
| `formatTime` | Extracts time from datetime string in locale format |
| `formatDateTime` | Full date + time in readable locale format |
| `formatRelativeTime` | `'3 days ago'`, `'in 2 hours'` — uses `Intl.RelativeTimeFormat` |

### textMaskingAndSecurity.js
Privacy masking with configurable mask character and visible digit count.

```js
maskEmail('user@example.com')       // 'u***@e******.com'
maskPhone('1234567890', 4)          // '******7890'
maskCreditCard('1234567890123456')  // '************3456'
moderate(text, ['word1'], '*')      // replaces whole-word matches
```

`maskPhone` and `maskCreditCard` throw on invalid input (non-10-digit phone, non-16-digit card).

### textMetadataAndExtraction.js
The largest module — 15 regex-based extraction functions returning arrays:

`extractNumbers`, `extractURLs`, `extractEmails`, `extractHashtags`, `extractMentions`, `extractDates`, `extractPhoneNumbers`, `extractIPv4Addresses`, `extractIPv6Addresses`, `extractFilePaths`, `extractDomainNames`, `extractHTMLTags`, `extractParenthesizedContent`, `extractQuotedText`, `extractJSONStrings`

Also: `countWords`, `countCharacter`, `findPositions` (all indices of substring).

### textAnalysisAndValidation.js
- `matchesPattern(string, regex)` — tests a regex against the string
- `isPalindrome(string)` — **BUG**: returns cleaned string instead of boolean (see Known Issues)

### textTransformations.js
- `shorten(str, length, ending)` — truncate with custom suffix
- `wordWrap(str, width)` — insert newlines at word boundaries
- `shuffle(str)` — randomize character order
- `removeDuplicates(str)` — remove repeated characters (`'aabb'` → `'ab'`)

### textSpecializedOperations.js
- `isBalancedBrackets(str)` — stack-based `()[]{}` balance check, O(n)
- `levenshteinDistance(a, b)` — full DP matrix, O(m×n) time and space

### textGeneration.js
- `generateRandomString(length, chars)` — random alphanumeric by default
- `generateLoremIpsum(wordCount, startWithLorem)` — random selection from Lorem Ipsum word pool

---

## 4. API Design — Two Usage Patterns

### Pattern A: Named imports (recommended for apps)
```js
import { maskEmail, extractURLs, camelCase } from 'stringy-core';
```
Tree-shakeable — bundlers include only what's used. Zero overhead.

### Pattern B: Namespace object (convenient for scripts/exploration)
```js
import { _s } from 'stringy-core';
_s.maskEmail('user@example.com');
_s.camelCase('Hello World');
```
All 50+ functions on one object. Not tree-shakeable — entire library included in bundle.

---

## 5. Key Design Decisions

### ESM-only (`"type": "module"`)
The package is pure ESM. No CommonJS fallback. Reason: enables static analysis for tree-shaking, aligns with the modern JS ecosystem direction, keeps the build config minimal (no dual CJS/ESM output needed).

Trade-off: older toolchains (pre-Webpack 5, old Jest without Babel) need extra config. The repo ships a Babel preset to handle this in tests.

### Zero runtime dependencies
No lodash, no validator.js, no date-fns. Every function is hand-implemented. Reasons:
1. No supply chain risk — the package installs exactly what it ships
2. No transitive dependency version conflicts for downstream users
3. Forces explicit, readable implementations reviewers can audit

`Intl` is used for formatting (it's a browser/Node built-in, not a dependency).

### Pure functions throughout
Every function takes input, returns output, touches no shared state. This makes functions composable, testable in isolation, and safe to call in any context without side effects.

### Flat module-to-function mapping
Each lib file owns one category. Functions don't call across lib files. This prevents circular dependencies and keeps each module independently understandable.

---

## 6. Contribution Model

The repo uses a "Contribution Station" pattern: stub functions are shipped with:
- A comment explaining the expected input/output
- An empty function body
- A test case in the test suite

Contributors implement the body and open a PR. This model was used to grow the function count and build community engagement. Several functions (e.g., `snakeCase`, `removeWhitespace`, `shuffle`, `extractHashtags`, `formatTime`, `maskCreditCard`) were contributed via this pattern — identifiable by the consistent comment style and input/output documentation.

---

## 7. Toolchain

| Tool | Role |
|---|---|
| Jest 29 | Test runner — 50+ tests across `tests/index.test.js` and `tests/textTransformation.test.js` |
| Babel | Transpiles ESM → CommonJS for Jest (Jest runs in CJS mode by default) |
| ESLint | Lints JS files; custom rule bans `console.log` (allows `warn`, `error`, `info`) |
| Prettier | Auto-formats all JS files on commit |
| Husky | Git hooks — fires `pre-commit` |
| lint-staged | Runs ESLint + Prettier only on staged files (not the whole repo) |

### Pre-commit flow
```
git commit
  → Husky fires .husky/pre-commit
  → lint-staged runs on staged *.js files:
      1. eslint --rule '{"no-console": ["error",...]}' -f table
      2. prettier --write
  → if eslint fails → commit blocked
  → if all pass → commit proceeds
```

### Test run flow
```
npm test
  → jest
  → Babel intercepts each import: ESM → CJS
  → test files execute in Node CJS mode
  → results reported
```

---

## 8. Known Issues

### isPalindrome returns string, not boolean
**File:** `lib/textAnalysisAndValidation.js:16`

```js
// Current (broken)
function isPalindrome(string) {
  const checkPalindrome = string.replace(/[^a-zA-Z0-9]/g, '').toLowerCase();
  return checkPalindrome; // ← returns cleaned string, not boolean
}

// Correct
function isPalindrome(string) {
  const cleaned = string.replace(/[^a-zA-Z0-9]/g, '').toLowerCase();
  return cleaned === cleaned.split('').reverse().join('');
}
```

No test exists for `isPalindrome` so this was never caught.

### extractFilePaths duplicate export
`lib/textMetadataAndExtraction.js:168` — `extractFilePaths` is listed twice in the export statement. The second reference is silently ignored by the JS engine but is a dead code smell.

---

## 9. Scaling Considerations

| Concern | Current | Improvement |
|---|---|---|
| Bundle size | No tree-shaking docs — users may accidentally import `_s` | Document named-import pattern prominently in README |
| CJS support | ESM-only; CJS users need Babel | Add a CJS build output (`dist/cjs/`) via Rollup |
| Type safety | No TypeScript types | Add `.d.ts` declaration files or migrate to TypeScript |
| `isPalindrome` bug | Returns wrong type | Fix comparison + add test |
| Regex correctness | IPv6, file path regexes are partial | Harden with more comprehensive patterns |
| New contributors | Contribution Station stubs still open | Close out remaining stubs or label them clearly |
