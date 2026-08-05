# stringy-core — System Design

**Platform:** Zero-dependency JavaScript string utility library  
**Published:** npm as `stringy-core` · 19 forks  
**Pattern:** "Lodash but for strings" — 50+ pure functions, 9 modules, ESM

---

## Problem Statement

JavaScript's native string API is thin. Developers repeatedly write the same utility functions — mask an email, format a currency, extract URLs from text, compute Levenshtein distance — across projects. Lodash covers arrays and objects but leaves strings largely untouched. stringy-core fills this gap as a dedicated, zero-dependency, tree-shakeable string utility library.

Secondary goal: an open-source contribution platform — stub functions invite community PRs, structured releases maintain backward compatibility.

---

## Architecture

### Module Structure

```
stringy-core/
├── index.js              — named exports + _s namespace
├── src/
│   ├── textCaseManipulation.js
│   ├── textCleaning.js
│   ├── textFormatting.js        — all via Intl API
│   ├── textMaskingAndSecurity.js
│   ├── textMetadataAndExtraction.js
│   ├── textAnalysisAndValidation.js
│   ├── textTransformations.js
│   ├── textSpecializedOperations.js
│   └── textGeneration.js
```

### Two Export Patterns

```javascript
// 1. Named tree-shakeable imports — bundler only ships what's used
import { maskEmail, formatCurrency } from 'stringy-core'

// 2. Namespace — all functions on one object
import { _s } from 'stringy-core'
_s.maskEmail('user@example.com')
```

Named imports allow tree-shaking — an app that only uses `maskEmail` doesn't bundle the other 49 functions.

### Key Module Details

**textFormatting** — all via `Intl` API:
```javascript
formatCurrency(1234.56, 'INR', 'en-IN')  // ₹1,234.56
formatDate(new Date(), 'en-IN', {dateStyle: 'long'})  // 15 जुलाई 2026
formatRelativeTime(-3, 'days', 'en')  // 3 days ago
```
Using `Intl` instead of manual formatting: locale-correct output for 100+ locales, no custom formatting logic to maintain.

**textMaskingAndSecurity:**
```javascript
maskEmail('user@example.com')   // us**@example.com
maskPhone('+919876543210')      // +91987****210
maskCreditCard('4111111111111111')  // 4111 **** **** 1111
```

**textSpecializedOperations:**
```javascript
levenshteinDistance('kitten', 'sitting')  // 3 — O(m×n) DP matrix
isBalancedBrackets('([{}])')  // true — stack-based
```

**textMetadataAndExtraction** — 15 regex extractors:
URLs, emails, hashtags, mentions (@user), IPs, HTML tags, dates, JSON strings, file paths, domains, phone numbers, credit card patterns, hex colours, markdown links.

---

## Engineering Decisions

| Decision | Choice | Why |
|---|---|---|
| Zero deps | No runtime dependencies | Works anywhere — browser, Node, Deno, React Native. No supply chain risk. |
| ESM primary | `"type": "module"` | Tree-shakeable, modern bundler-friendly |
| CJS compat | Babel transform in package | Older Node.js and Jest compatibility |
| Intl for formatting | Browser/Node native API | No locale data to bundle, 100+ locales correct |
| DP for Levenshtein | O(m×n) matrix | Standard implementation, predictable, correct |
| Husky + lint-staged | Pre-commit ESLint + Prettier | Consistent code quality across community PRs |
| Semantic versioning | Major.Minor.Patch | Breaking changes are clearly signalled |

---

## Open Source Contribution Platform

Functions marked `// CONTRIBUTION_STUB` invite community PRs:

```javascript
export function soundex(str) {
  // CONTRIBUTION_STUB: implement Soundex phonetic algorithm
  throw new Error('Not implemented — open a PR!')
}
```

**Contribution standards maintained:**
- PR review covers: correctness (unit tests required), API design (backward compatible), zero deps (no new dependencies)
- Release planning: stubs don't ship in minor releases — only complete implementations
- Public API stability: function signatures frozen after first release in a minor version

---

## Known Bug

`isPalindrome` returns the cleaned string instead of a boolean:

```javascript
// Current (wrong):
isPalindrome('racecar')  // returns 'racecar' instead of true

// Fix:
return cleaned === cleaned.split('').reverse().join('')
```

This is a known bug tracked for the next patch release.

---

## What I'd Do Differently

- **TypeScript source:** Type definitions for 50+ functions would significantly improve DX — currently ships with manually maintained `.d.ts` file
- **Benchmarks:** No performance benchmarks — Levenshtein on long strings could be slow; users don't know until they hit it
- **Browser bundle:** No pre-built IIFE/UMD bundle — requires a bundler to use in a `<script>` tag. A CDN build would lower the contribution barrier.
