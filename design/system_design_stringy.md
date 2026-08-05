# stringy-core — System Design

**Platform:** Zero-dependency JavaScript string utility library
**Published:** npm as `stringy-core` · 19 forks
**Pattern:** "Lodash but for strings" — 50+ pure functions, 9 modules, ESM-first

---

## Quick Reference

| Property | Value |
|---|---|
| Package | `stringy-core` |
| Registry | npm |
| Community | 19 forks |
| Functions | 50+ |
| Modules | 9 |
| Runtime dependencies | 0 |
| Primary format | ESM (`"type": "module"`) |
| CJS compat | Babel transform |
| Export patterns | Named tree-shakeable + `_s` namespace |
| Formatting strategy | `Intl` API (no locale bundles) |
| Algorithm highlights | Levenshtein O(m×n) DP, stack-based bracket matching |
| Tooling | Jest, Husky, lint-staged, ESLint, Prettier |

---

## Problem Statement

JavaScript's native `String` API is thin by design. Developers repeatedly rewrite the same string utilities across projects — masking a credit card number, computing edit distance between strings, extracting all URLs from a block of text, formatting a phone number for the current locale. Libraries like Lodash cover arrays and objects comprehensively but treat strings as a secondary concern.

The result: every team builds their own `utils/string.js`. No tests. No edge case handling. Locale formatting done wrong. Credit card masks that expose too many digits.

stringy-core fills this gap as a dedicated, zero-dependency, tree-shakeable string utility library with two additional goals:

1. **Zero surface area for supply chain attacks.** No runtime dependencies means no transitive risk.
2. **An open-source contribution platform.** Stub functions with defined signatures invite community PRs, growing the library without expanding the core team.

---

## Design Goals

| Goal | Mechanism |
|---|---|
| Zero runtime dependencies | All logic in pure JS — no lodash, moment, or third-party imports |
| Tree-shakeable | Named ESM exports; bundlers statically analyse import graphs |
| Locale-correct formatting | `Intl` API (browser/Node native) — no locale data to ship |
| Contribution-friendly | `// CONTRIBUTION_STUB` pattern defines the API before implementation exists |
| Backward compatible | Semantic versioning enforced; no signature changes in minor releases |
| Works everywhere | ESM primary + Babel CJS transform covers browser, Node, Deno, React Native |

---

## Architecture

### Module Structure

```
stringy-core/
├── index.js                         — named exports + _s namespace re-export
├── package.json                     — "type": "module", exports map
├── babel.config.js                  — ESM → CJS for Jest
├── .husky/pre-commit                — runs lint-staged
├── .lintstagedrc                    — ESLint + Prettier on staged files
└── src/
    ├── textCaseManipulation.js      — camelCase, snakeCase, titleCase, etc.
    ├── textCleaning.js              — trim, normalize, remove whitespace
    ├── textFormatting.js            — formatCurrency, formatDate (all Intl)
    ├── textMaskingAndSecurity.js    — maskEmail, maskPhone, maskCreditCard
    ├── textMetadataAndExtraction.js — 15 regex extractors
    ├── textAnalysisAndValidation.js — matchesPattern, isPalindrome (bug: see below)
    ├── textTransformations.js       — shorten, wordWrap, shuffle
    ├── textSpecializedOperations.js — levenshteinDistance, isBalancedBrackets
    └── textGeneration.js            — generateRandomString, generateLoremIpsum
```

### Two Export Patterns

**Pattern 1 — Named tree-shakeable imports (recommended):**

```javascript
import { maskEmail, formatCurrency, levenshteinDistance } from 'stringy-core'

maskEmail('user@example.com')        // us**@example.com
formatCurrency(1299, 'INR', 'en-IN') // ₹1,299.00
levenshteinDistance('kitten', 'sitting') // 3
```

The bundler (Vite, Webpack, Rollup) performs static analysis on your import graph. If you only import `maskEmail`, the other 49+ functions are never included in the output bundle. This is the recommended pattern for any production app.

**Pattern 2 — `_s` namespace (convenience / REPL / scripts):**

```javascript
import { _s } from 'stringy-core'

_s.maskEmail('user@example.com')
_s.formatCurrency(1299, 'INR', 'en-IN')
_s.levenshteinDistance('kitten', 'sitting')
```

All functions are attached to a single object. Useful in Node.js scripts, quick CLI tools, or exploratory REPL sessions where bundle size is not a concern. The `_s` prefix is intentional — it avoids collision with any global `s` variable and signals "string utility namespace."

---

## Module Details

### textCaseManipulation

Converts between naming conventions:

```javascript
import { camelCase, snakeCase, titleCase, invertCase, swapCase, toAlternateCase }
  from 'stringy-core'

camelCase('hello world')       // 'helloWorld'
snakeCase('helloWorld')        // 'hello_world'
titleCase('the quick fox')     // 'The Quick Fox'
invertCase('Hello')            // 'hELLO'
swapCase('Hello World')        // 'hELLO wORLD'
toAlternateCase('hello')       // 'hElLo'
```

All functions treat runs of whitespace, hyphens, and underscores as word boundaries. `camelCase` and `snakeCase` are normalised from any input convention — `'hello-world'`, `'hello_world'`, and `'helloWorld'` all produce the same result.

### textCleaning

Whitespace control:

```javascript
import { trimStart, trimEnd, normalizeWhitespace, removeWhitespace }
  from 'stringy-core'

normalizeWhitespace('hello   world\t\nfoo')  // 'hello world foo'
removeWhitespace('hello world')              // 'helloworld'
```

`normalizeWhitespace` collapses all internal whitespace (spaces, tabs, newlines) to a single space and trims both ends. Useful before storing user-submitted text to a database.

### textFormatting — built entirely on the `Intl` API

```javascript
import { formatNumber, formatCurrency, formatDate, formatTime,
         formatDateTime, formatRelativeTime } from 'stringy-core'

formatNumber(1234567.89, 'en-IN')            // '12,34,567.89' (Indian grouping)
formatCurrency(1234.56, 'INR', 'en-IN')      // '₹1,234.56'
formatCurrency(1234.56, 'USD', 'en-US')      // '$1,234.56'
formatDate(new Date(), 'en-IN', { dateStyle: 'long' })   // '5 August 2026'
formatRelativeTime(-3, 'days', 'en')         // '3 days ago'
formatRelativeTime(2, 'hours', 'fr')         // 'dans 2 heures'
```

Every function delegates to `Intl.NumberFormat`, `Intl.DateTimeFormat`, or `Intl.RelativeTimeFormat`. The browser or Node runtime provides locale data — stringy-core ships zero locale files of its own.

### textMaskingAndSecurity

```javascript
import { maskEmail, maskPhone, maskCreditCard, maskString, moderate }
  from 'stringy-core'

maskEmail('user@example.com')        // 'us**@example.com'
maskPhone('+919876543210')           // '+91987****210'
maskCreditCard('4111111111111111')   // '4111 **** **** 1111'
maskString('password123', 3, 3)     // 'pas******123'
moderate('This is damn annoying')   // 'This is **** annoying'
```

`maskEmail` preserves the first two characters and the full domain — enough for the user to recognise the account, not enough to expose it in a log. `maskCreditCard` follows PCI-DSS display conventions (first 4, last 4 visible). `maskString(str, leadingVisible, trailingVisible)` is the generalised form for any sensitive field.

### textMetadataAndExtraction — 15 regex extractors

```javascript
import { extractURLs, extractEmails, extractHashtags, extractMentions,
         extractIPs, extractHTMLTags, extractDates, extractJSONStrings,
         extractFilePaths, extractDomains, extractPhones,
         extractCreditCardPatterns, extractHexColours, extractMarkdownLinks }
  from 'stringy-core'

const text = 'Check out https://swanand.dev and email me@example.com #coding @swanand'

extractURLs(text)       // ['https://swanand.dev']
extractEmails(text)     // ['me@example.com']
extractHashtags(text)   // ['#coding']
extractMentions(text)   // ['@swanand']
```

Each extractor uses a single-pass regex against the input string and returns an array of all matches (empty array if none found). Regex patterns are kept in module scope — not recompiled on every call.

### textAnalysisAndValidation

```javascript
import { matchesPattern, isPalindrome } from 'stringy-core'

matchesPattern('hello123', /^[a-z]+\d+$/)  // true
isPalindrome('racecar')   // BUG: returns 'racecar' — see Known Bug section
```

### textTransformations

```javascript
import { shorten, wordWrap, shuffle, removeDuplicates } from 'stringy-core'

shorten('The quick brown fox jumps', 15)  // 'The quick brow…'
wordWrap('hello world foo bar', 10)       // 'hello\nworld foo\nbar'
shuffle('hello')                          // e.g. 'olhle' (random each call)
removeDuplicates('programming')           // 'progamin'
```

`shorten` appends an ellipsis character (`…`) and respects the length limit including the ellipsis. `wordWrap` breaks on whitespace boundaries — it will not break mid-word.

### textSpecializedOperations

```javascript
import { levenshteinDistance, isBalancedBrackets } from 'stringy-core'

levenshteinDistance('kitten', 'sitting')  // 3
levenshteinDistance('saturday', 'sunday') // 3

isBalancedBrackets('([{}])')   // true
isBalancedBrackets('([)]')     // false
isBalancedBrackets('{[}')      // false
```

**Levenshtein algorithm (O(m×n) DP matrix):**

Creates a 2D matrix where `dp[i][j]` = minimum edits to transform `source[0..i]` into `target[0..j]`. Fills left-to-right, top-to-bottom. When characters match, cost is `dp[i-1][j-1]`. When they differ, cost is `1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])` — the minimum of delete, insert, or substitute. Final answer is `dp[m][n]`.

**isBalancedBrackets algorithm (stack-based):**

Iterates the string character-by-character. Opening brackets (`(`, `[`, `{`) are pushed onto a stack. Closing brackets pop the stack and compare — if the popped value is not the matching opener, return false. After the full string, the stack must be empty for the input to be balanced.

### textGeneration

```javascript
import { generateRandomString, generateLoremIpsum } from 'stringy-core'

generateRandomString(12)    // e.g. 'aB3kP9mZqRtX'
generateRandomString(8, { uppercase: false, numbers: false }) // 'xqpmtrka'
generateLoremIpsum(3)       // 3 sentences of placeholder Latin
```

`generateRandomString` uses `Math.random()` — suitable for placeholder IDs and test fixtures, not for cryptographic tokens. For security tokens, the `maskString` approach or native `crypto.getRandomValues` is appropriate.

---

## Engineering Decisions

| Decision | Choice | Alternatives Considered | Why this choice |
|---|---|---|---|
| Zero runtime deps | No third-party imports | lodash, validator.js, libphonenumber | Works anywhere with no install risk; no supply chain exposure |
| ESM primary | `"type": "module"` | CommonJS, dual package | Tree-shakeable by default in all modern bundlers |
| CJS compat | Babel transform in dev/test | Separate CJS build | Keeps one source of truth; Jest needs CJS, not production |
| Intl for formatting | Browser/Node native API | custom format strings, dayjs, date-fns | No locale data to bundle; 100+ locales already in runtime |
| DP for Levenshtein | O(m×n) full matrix | Diagonal optimisation, BK-tree | Readable, correct, no micro-optimisation needed at library level |
| Stack for brackets | O(n) single pass | Regex, recursion | Classic O(n) algorithm; recursive approach risks stack overflow |
| Husky + lint-staged | Pre-commit hooks | CI-only linting | Catches issues locally — faster feedback loop for contributors |
| Semantic versioning | Major.Minor.Patch | CalVer | Breaking changes are explicitly signalled with a major bump |
| Contribution stubs | `// CONTRIBUTION_STUB` pattern | Issue tracker only | The API is defined first; contributors implement a known contract |

---

## Open Source Contribution Platform

stringy-core doubles as a structured contribution platform. Functions are defined — name, signature, JSDoc — before the implementation exists. The stub pattern signals this clearly:

```javascript
/**
 * Compute the Soundex phonetic code for a string.
 * @param {string} str - Input string
 * @returns {string} Four-character Soundex code (e.g. 'R163' for 'Robert')
 */
export function soundex(str) {
  // CONTRIBUTION_STUB: implement Soundex phonetic algorithm
  // See: https://en.wikipedia.org/wiki/Soundex
  throw new Error('Not implemented — open a PR!')
}
```

**Why this pattern works:**

The function signature and expected behaviour are locked in before implementation. A contributor knows exactly what to build. Code review focuses on correctness and test coverage, not API design debates.

**PR standards enforced at review:**

1. **Correctness** — unit tests required. The PR must demonstrate the function handles the happy path, empty string, and at least two edge cases.
2. **API stability** — the implemented function must match the stub signature exactly. Parameter names, order, and return type are the contract.
3. **Zero deps** — no new runtime dependencies. The reviewer checks `package.json` diff.
4. **Lint clean** — the pre-commit hook runs locally, but CI runs ESLint + Prettier on the PR branch. PRs that fail lint are not reviewed until green.

**Release policy for stubs:**

Stubs are not released in minor versions. A stub that throws `Error('Not implemented')` is better than a missing export (consumers can detect it) but must never ship to a patch release as-is. Only complete, tested implementations are included in releases.

---

## Known Bug — `isPalindrome`

**Module:** `textAnalysisAndValidation`

**Current behaviour (incorrect):**

```javascript
isPalindrome('racecar')   // returns 'racecar'   — should return true
isPalindrome('hello')     // returns 'olleh'      — should return false
isPalindrome('A man a plan a canal Panama')  // returns 'amanaplanacanalpanama'
```

**Root cause:**

The function correctly cleans the input (lowercases, strips non-alphanumeric), but the final return statement returns the cleaned string instead of the boolean comparison result:

```javascript
// Current (wrong):
return cleaned   // returns the string

// Fix — one character change:
return cleaned === cleaned.split('').reverse().join('')
```

**Why it matters in interview context:**

This is a deliberate demonstration that published open-source software ships with bugs. The function has been in production through multiple minor versions. Fixing it is a patch release (no API change — return type changes from `string` to `boolean`, which is the documented contract). This bug also surfaces the importance of `typeof` return-type testing, not just value testing — a test that did `if (isPalindrome('racecar'))` would pass (truthy string) while the semantics were wrong.

---

## FAQ

### "Why zero runtime dependencies? What does that actually mean for library consumers?"

When a package has runtime dependencies, every consumer transitively installs them. If `stringy-core` depended on `lodash`, every project that installed `stringy-core` would also download all of lodash — even if the project already had lodash at a different version. This creates two problems: bundle size (especially on the browser) and supply chain risk.

Supply chain risk is the more serious concern. The 2016 `left-pad` incident took down thousands of production builds when a single 11-line npm package was unpublished. More dangerous are compromised packages — when an attacker gains publish access to a transitive dependency and injects malicious code. A package with zero dependencies eliminates the entire transitive tree as an attack surface.

For consumers, zero dependencies means: `npm install stringy-core` installs exactly one package. No lockfile churn from transitive updates. No peer dependency conflicts. Works in any environment without compatibility shims.

### "What's the difference between the two export patterns? When would I use each?"

Named imports (`import { maskEmail } from 'stringy-core'`) are for production application code. The bundler performs dead code elimination — functions you don't import are not included in the output bundle. If you import only `maskEmail`, Vite or Webpack analyses your module graph and excludes all other functions from the final JavaScript. This matters when you're shipping to browsers where every byte costs load time.

The `_s` namespace (`import { _s } from 'stringy-core'`) imports a single object that holds all functions. It is convenient for Node.js scripts, CLI tools, and exploratory work. Since you import the whole `_s` object, tree-shaking cannot eliminate individual unused methods — the bundler does not know which `_s.*` calls will execute at runtime. Use it where bundle size does not matter. The underscore prefix was chosen to avoid colliding with a local variable named `s`.

Rule of thumb: named imports in app code, `_s` in scripts and tools.

### "Why use the `Intl` API for formatting instead of writing your own formatter?"

Manual formatters fail at the edges of locale rules. Indian number grouping (`12,34,567` rather than `1,234,567`) breaks any formatter that hardcodes groups of three. Arabic scripts write numbers right-to-left. French uses non-breaking spaces as thousands separators. German uses commas where English uses dots for decimals.

The `Intl` API (ECMA-402) is built into every modern browser and into Node.js since v13. The runtime ships locale data and handles all these edge cases. `stringy-core` delegates completely — `new Intl.NumberFormat(locale, options).format(value)` — and gets 100+ correct locale implementations for free. A hand-rolled formatter would need to ship locale data files (potentially megabytes) and would still miss edge cases.

The tradeoff is that very old environments (IE 11) have incomplete `Intl` support. That is an acceptable tradeoff for a 2024 library targeting modern JavaScript.

### "Walk me through how Levenshtein distance is computed. What's the time complexity?"

Levenshtein distance is the minimum number of single-character edits (insertions, deletions, substitutions) to transform one string into another. For `'kitten'` → `'sitting'` the answer is 3 (substitute k→s, substitute e→i, insert g at end).

The algorithm builds a 2D matrix of size `(m+1) × (n+1)` where m = source length, n = target length:

1. Initialise: `dp[0][j] = j` (cost to build target from empty = j insertions). `dp[i][0] = i` (cost to reach empty from source = i deletions).
2. Fill left-to-right, top-to-bottom: if `source[i-1] === target[j-1]`, cost is zero and `dp[i][j] = dp[i-1][j-1]`. If they differ: `dp[i][j] = 1 + Math.min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])` — the minimum of delete from source, insert into source, or substitute.
3. Answer: `dp[m][n]`.

**Time complexity:** O(m×n) — every cell is filled exactly once.
**Space complexity:** O(m×n) — the full matrix is held in memory.

A space-optimised version uses two rows instead (O(n)), which stringy-core does not currently implement.

### "`isBalancedBrackets` — walk me through the stack-based algorithm."

The algorithm makes a single left-to-right pass over the string in O(n) time with O(n) auxiliary space (the stack).

Logic character-by-character:
- If the character is an opening bracket (`(`, `[`, `{`) → push it onto the stack.
- If the character is a closing bracket (`)`, `]`, `}`) → check the stack:
  - If the stack is empty → no matching opener exists → return `false`.
  - Pop the top. If the popped opener does not match the current closer → return `false`.
- Ignore all non-bracket characters.

After processing the full string, the stack must be empty. A non-empty stack means unclosed openers remain → return `false`. Empty stack → return `true`.

Example trace for `([{}])`:
- `(` → push → stack: `[(]`
- `[` → push → stack: `[(, []`
- `{` → push → stack: `[(, [, {]`
- `}` → pop `{`, matches → stack: `[(, []`
- `]` → pop `[`, matches → stack: `[(]`
- `)` → pop `(`, matches → stack: `[]`
- End: stack empty → `true`

Example of failure: `([)]` — when `)` is encountered, the top of stack is `[` not `(` → mismatch → `false`.

### "You have a known bug in `isPalindrome`. Why hasn't it been fixed yet?"

Honest answer: the bug is real, tracked, and has a one-line fix. It has not shipped in a patch release yet because open-source maintenance has scheduling constraints. There is no CI that enforces return-type correctness — the tests likely assert truthiness (`if (isPalindrome('racecar'))`) rather than strict boolean equality (`isPalindrome('racecar') === true`). This is a lesson in test design: test the contract, not just the outcome.

For an interview, the more interesting point is what this reveals about API design: changing a return type from `string` to `boolean` is technically a breaking change even if it fixes a bug, because any consumer who did `const result = isPalindrome(x); result.toUpperCase()` would break. The correct fix ships in a minor release with a deprecation warning, or is documented as a behaviour correction in the patch notes. stringy-core's practical stance is that returning a string from a function named `isPalindrome` is clearly wrong and a boolean fix is not a breaking change in spirit.

### "What does Husky + lint-staged give you in an open-source project?"

Husky installs git hooks — scripts that run automatically at specific points in the git workflow. The pre-commit hook runs `lint-staged` before each commit.

`lint-staged` runs configured tools only against files that are staged for the current commit, not the entire codebase. For stringy-core that is ESLint (catch style and logic issues) + Prettier (enforce consistent formatting). A contributor who writes code with tabs instead of spaces, or uses `var` instead of `const`, gets rejected locally before they can even create a commit.

For an open-source project with dozens of contributors, this matters because:
1. **Code review is about logic, not style.** Lint handles formatting debates mechanically.
2. **CI time is expensive.** Catching lint failures locally is faster and cheaper than in CI.
3. **Low barrier for contributors.** The contributor does not need to memorise the style guide — the hook enforces it automatically.

The alternative — CI-only linting — works but creates a slower feedback loop: contributor pushes, CI fails, contributor reads log, fixes, pushes again. Husky shortens that to a local rejection and immediate fix.

### "How does tree-shaking work? If I import only `maskEmail`, what ships in my bundle?"

Tree-shaking is a form of dead code elimination that bundlers (Vite, Rollup, Webpack) perform on ES modules. It relies on the static structure of `import`/`export` statements — unlike CommonJS `require()`, ESM imports are resolved at parse time, not runtime.

When you write `import { maskEmail } from 'stringy-core'`, the bundler builds a module graph. It knows that `stringy-core/index.js` exports `maskEmail`, `formatCurrency`, `levenshteinDistance`, and 47 other functions. But it also knows that your code only references `maskEmail`. Every function not reachable from an import statement in your code is marked dead — and excluded from the bundle.

What actually ships: the `maskEmail` function definition and any private helper functions it calls within its module. Nothing from `textFormatting.js`, `textSpecializedOperations.js`, or any other module that `maskEmail` does not import.

Tree-shaking requires that modules have no side effects. stringy-core modules are pure — they only define and export functions. This is declared in `package.json` with `"sideEffects": false`, which is a signal to bundlers that every file in the package is safe to exclude if unused.

---

## Interview Bridges

These are topics where stringy-core is a concrete, real example you can use to anchor broader engineering conversations.

### Zero Dependencies → npm Supply Chain Security

The 2016 `left-pad` incident: Azer Koçulu unpublished 250+ npm packages over a naming dispute. `left-pad` was 11 lines of code. Thousands of production builds broke including Babel and React's build pipeline. The lesson was that the npm ecosystem had a single-point-of-failure problem — one removed package could cascade through hundreds of dependent packages.

More dangerous: compromised packages. When attackers gain publish access to a popular transitive dependency, they can inject malicious code into every downstream consumer. This has happened with `event-stream` (2018), `ua-parser-js` (2021), and others. A library with zero runtime dependencies has no transitive attack surface at all.

stringy-core is an example of a library where I made this tradeoff deliberately — everything in pure JS, no shortcuts from external packages.

### Tree-Shaking + Named Exports → Bundle Size in Production

Bundle size is a first-order metric for web performance. Every extra kilobyte costs parse time (JavaScript must be parsed before executing) and transfer time. On mobile networks this is especially significant.

Named ESM exports are the mechanism that enables tree-shaking. Libraries that use `module.exports = { ... }` (CommonJS) cannot be tree-shaken because `module.exports` is a runtime object — the bundler cannot statically determine which properties are accessed.

stringy-core is an example of getting this right: 50+ functions available, but a production app that only needs string masking pays for exactly one function.

### `Intl` API → Internationalisation Patterns at Scale

The `Intl` API is the right mental model for any internationalisation problem. The pattern: never hardcode locale assumptions in business logic. Delegate to a locale-aware runtime API. This applies beyond string formatting — `Intl.Collator` for locale-correct sorting, `Intl.PluralRules` for "1 item" vs "2 items" vs "2 items" (Russian has four plural forms), `Intl.ListFormat` for "a, b, and c" in English vs the equivalent in Japanese.

Every major consumer app (Airbnb, Uber, Shopify) has a team dedicated to i18n for exactly this reason. stringy-core demonstrates the right habit at library level: locale as a parameter, never hardcoded.

### Levenshtein Distance → Fuzzy Search, Spell Checking, Git Blame

Levenshtein distance is foundational in:
- **Autocomplete and fuzzy search** — IDEs rank suggestions by edit distance from what you typed
- **Spell checkers** — "did you mean X?" suggestions are the closest-distance words in the dictionary
- **git blame / diff** — git's rename detection computes similarity scores between file contents using edit-distance variants
- **DNA sequencing** — aligning genetic sequences is a variation of the same DP problem (Smith-Waterman)

Being able to walk through the DP matrix from memory is a strong signal in a systems interview. Knowing the space optimisation (two-row DP) is a bonus.

### `isBalancedBrackets` → Classic Stack Interview Problem

Stack-based bracket matching is one of the most common interview problems because it cleanly isolates the stack data structure:
- One pass, O(n) time
- Stack depth is O(n) worst case
- Demonstrates understanding of LIFO order and the relationship between openers and closers

Extensions the interviewer may ask: handle multi-character openers (like HTML tags), count depth, return the index of the first unmatched bracket rather than a boolean.

### Contribution Stubs → Open-Source Maintainership, API-First Design

The stub pattern is API-first design made explicit. You design the interface before the implementation — the function name, parameters, return type, and documented behaviour. This forces you to think about the consumer's experience before writing a line of implementation code.

This pattern maps directly to how mature open-source projects manage contributions: RFC process (React, Rust) → approved RFC becomes a spec → implementation follows. The stub is a micro-version of this: the spec is the JSDoc comment and the throwing stub.

It also demonstrates good maintainership: the reviewer's job is easier when the contract is pre-defined. They focus on whether the implementation matches the spec, not on whether the spec is right.

---

## What-If Scenarios

### "You need to add TypeScript support. What's the migration path?"

There are three levels of TypeScript support, each incrementally more work:

**Level 1 — Declaration files only (`.d.ts`):** Write `index.d.ts` by hand. Consumers get autocomplete and type errors without changing any source. This is what stringy-core currently does (manually maintained). Downside: the types drift from the implementation unless you have a process to keep them in sync.

**Level 2 — JSDoc type annotations in source:** Add `@param {string}` and `@returns {boolean}` annotations to every function. TypeScript can extract types from JSDoc without `.ts` source files. `tsc --allowJs --declaration --emitDeclarationOnly` generates `.d.ts` files automatically. This eliminates drift and keeps source as `.js`.

**Level 3 — Migrate source to TypeScript:** Rename files to `.ts`, add strict types, remove the Babel CJS transform (TypeScript's compiler handles that), update `tsconfig.json` with `"module": "ESNext"` and `"declaration": true`. Build step: `tsc` outputs typed `.d.ts` + `.js` files. This gives the strongest guarantees but adds a compilation step and increases onboarding friction for contributors unfamiliar with TypeScript.

The migration path: Level 1 (already done) → Level 2 (no breaking change, quick win) → Level 3 (next major version, opt-in for contributors).

### "A user reports that `levenshteinDistance` is too slow on 10,000-character strings. How do you fix it?"

The current O(m×n) DP matrix for 10,000-character strings allocates a 10,000 × 10,000 matrix — 100 million cells. At 8 bytes per number (JavaScript float64), that is 800MB. It will OOM before it runs slowly.

Fixes in order of effort:

**1. Space optimisation (quick, no algorithm change):** The full matrix is not needed — only the previous row. Use two arrays of length `n+1` and alternate. Space drops from O(m×n) to O(n). Time stays O(m×n).

**2. Early termination threshold:** If the caller provides a `maxDistance` parameter, abandon computation when all cells in a row exceed the threshold. Most fuzzy search use cases only care whether distance < some threshold (e.g. "is this a typo?"). This prunes the work significantly when strings differ widely.

**3. Ukkonen's algorithm:** O(kn) where k is the actual edit distance. When strings are similar (small k), this is dramatically faster than O(mn). The implementation is more complex.

**4. Bitap algorithm:** O(mn/wordsize) using bitwise operations. Practical speedup for short patterns (< 64 chars) against long texts.

For the library, the right answer is 1 + 2: ship the space optimisation immediately (no API change), add an optional `maxDistance` parameter for performance-sensitive callers.

### "You want to publish a CDN build for `<script>` tag usage. What changes?"

ESM modules cannot be used directly via a `<script src="...">` tag in all environments — they require either `type="module"` or a bundler. For CDN usage without a build step, you need an IIFE (Immediately Invoked Function Expression) or UMD (Universal Module Definition) build.

**Build step:** Add Rollup or esbuild to the dev toolchain. Configure an IIFE output that exposes a global variable — e.g. `window.stringy` or `window._s`. The build command generates `dist/stringy-core.min.js`.

**Package.json changes:** Add an `exports` map:
```json
{
  "exports": {
    ".": {
      "import": "./index.js",
      "require": "./dist/stringy-core.cjs",
      "browser": "./dist/stringy-core.min.js"
    }
  }
}
```

**CDN publication:** npm packages are automatically mirrored to `unpkg.com` and `cdn.jsdelivr.net`. Once `dist/stringy-core.min.js` is included in the package, it is available at:
`https://unpkg.com/stringy-core@latest/dist/stringy-core.min.js`

**Usage:**
```html
<script src="https://unpkg.com/stringy-core@latest/dist/stringy-core.min.js"></script>
<script>
  console.log(window._s.maskEmail('user@example.com'))
</script>
```

**Implications:** The IIFE build cannot be tree-shaken — it always ships all functions. That is acceptable for CDN usage where the developer chose not to use a bundler. For app development, named ESM imports remain the recommended path.

---

## What I'd Do Differently

### TypeScript Source

The library currently ships a manually maintained `.d.ts` file. This is fragile — function signatures can change and types drift. Writing source in TypeScript from the start would generate accurate types automatically and catch type errors in the implementation itself. Migrating 9 modules from `.js` to `.ts` is straightforward; the harder part is deciding the right level of genericism (does `maskString` accept `string | number` or strictly `string`?). The correct approach is to migrate module by module starting with `textFormatting`, which benefits most from typed locale parameters.

### Performance Benchmarks

There are no benchmarks in the repository. Consumers who call `levenshteinDistance` in a hot loop have no data on whether it is fast enough for their use case. A benchmark suite (using `node:perf_hooks` or Vitest bench) would document: how Levenshtein scales with string length, how fast the regex extractors are on large text blocks, and whether `generateLoremIpsum` is synchronous-safe at scale. This data would make the "too slow" issue reporter's question answerable before they even file it.

### CDN / Browser Build

There is no pre-built IIFE bundle. A developer who wants to experiment in a browser console or use stringy-core in a plain HTML file needs to set up Vite or Webpack first. A CDN build would lower the contribution barrier significantly — contributors could prototype a new function in CodePen without a build step. The build output would be a single minified file generated by Rollup in the CI pipeline, published to npm's `dist/` directory, and available via unpkg automatically.

### End-to-End Return Type Tests

The `isPalindrome` bug would have been caught by a test that asserted `typeof isPalindrome('racecar') === 'boolean'` or `isPalindrome('racecar') === true` (strict equality). All assertion-based tests in the current suite use truthy checks. Adding `typeof` assertions to every function's test suite is a one-time fix that prevents this class of bug from recurring.

---

## Technology Comparisons

### ESM vs CommonJS vs UMD

stringy-core ships as `"type": "module"` (pure ESM). For a utility library, module format is one of the most consequential packaging decisions.

| Dimension | ESM | CommonJS (CJS) | UMD |
|---|---|---|---|
| Syntax | `import` / `export` | `require()` / `module.exports` | IIFE wrapper — works as CJS, AMD, or browser global |
| Tree-shaking | Yes — statically analysable | No — dynamic `require()` can't be analyzed | No |
| Node.js support | Native (Node 12+) | Native (always) | Via CJS mode |
| Browser support | Native (modern browsers) | No (needs bundler) | Yes (script tag) |
| Jest compatibility | Requires Babel or `--experimental-vm-modules` | Native | Native |
| Standard | ES2015 — the current standard | Node.js legacy | Pre-bundler legacy pattern |

**When CJS still wins:** You are publishing a Node.js CLI tool where tree-shaking is irrelevant, your consumers are on Node < 12, or you need maximum compatibility with the oldest tooling.

**When UMD still wins:** You're publishing a browser library intended to be included directly via `<script>` tag in environments without a bundler. Rare in 2025.

**For stringy-core:** Pure ESM because tree-shaking is the primary consumer-facing benefit of a utility library. A consumer who only uses `maskEmail` should not bundle `levenshteinDistance`. CJS `require()` is dynamic — bundlers cannot statically determine which exports are used, so they include everything. ESM `import { maskEmail }` is static — bundlers eliminate everything else. The Babel + Jest workaround (Babel transpiles ESM → CJS for the test runner) is a one-time dev tooling cost, not a consumer concern.

**Interview move:** "The tradeoff with pure ESM is Jest compatibility — you need Babel for the test runner. We accepted that cost because the consumer benefit (tree-shaking) is permanent, while the Babel config is a one-time setup. Dual publishing (ESM + CJS) would eliminate the Jest workaround but doubles the build pipeline complexity for a zero-runtime-deps utility library."

---

### Named Exports vs Namespace Export vs Default Export

stringy-core offers two import patterns. Understanding why is a good signal of API design maturity.

| Pattern | Syntax | Tree-shakeable | Autocomplete | Common in |
|---|---|---|---|---|
| Named exports | `import { maskEmail } from 'stringy-core'` | Yes | Full — IDE shows available exports | Most utility libraries |
| Namespace | `import { _s } from 'stringy-core'` | No — the whole `_s` object is imported | Yes — after `_s.` | Convenience wrappers |
| Default export | `import stringy from 'stringy-core'` | No | Depends on what `default` is | Frameworks, single-class libs |

**The `_s` namespace pattern:** All 50+ functions are collected onto one object and exported as `_s`. This is a convenience layer — `_s.maskEmail('...')` is cleaner in scripts and REPL sessions than managing many named imports. The cost is that bundlers cannot tree-shake: importing `_s` brings every function into the bundle even if only one is used.

**For stringy-core:** Both patterns are offered deliberately. Named imports for production application code where bundle size matters. Namespace `_s` for scripting, one-off use, and consumers who prefer the Lodash-style `_.method()` ergonomic. The dual pattern is documented in the README so consumers make an informed choice.

**The `isPalindrome` bug in this context:** The function returns the cleaned string instead of a boolean. `typeof isPalindrome('racecar') === 'string'` — true. This is a semantic bug in the named export. The fix is one line: `return cleaned === cleaned.split('').reverse().join('')`. The fact that stringy-core has typed JSDoc for the function shows the intent was boolean; the implementation diverged. Adding `typeof` assertions to every function test prevents this class of bug from recurring — the current test suite uses truthy checks which a string passes.

---

## Technical Dictionary

*Plain-English definitions of every term, algorithm, and tool used in this document. If something above confused you, start here.*

---

### JavaScript Module System

### ESM (ECMAScript Module)
The modern JavaScript module system using `import` and `export` syntax, standardised in ES2015. It is designed for static analysis: bundlers can determine at compile time exactly which exports are used, enabling dead code elimination.

**Example:** stringy-core sets `"type": "module"` in `package.json`, making every `.js` file in the package an ES module, so `import { maskEmail } from 'stringy-core'` works without any transformation in modern environments.

---

### CJS (CommonJS)
Node.js's original module system using `require()` to load modules at runtime. Because module resolution is dynamic, bundlers cannot determine which exports are used at compile time, making CJS code impossible to tree-shake.

**Example:** Jest defaults to a CJS environment, which is why stringy-core's `babel.config.js` exists — it transforms the ESM source into CJS on the fly so Jest can `require()` the modules during test runs.

---

### Named export
A function or value exported under a specific identifier using `export function foo()` or `export { foo }`. Consumers import it by that exact name. Because the name is statically known, bundlers can track precisely which exports are used.

**Example:** `export function maskEmail(str)` in `textMaskingAndSecurity.js` is a named export — a consumer who writes `import { maskEmail } from 'stringy-core'` gets exactly that function and nothing else ships in their bundle.

---

### Default export
Exporting one thing as the primary output of a module using `export default`. A module can have only one default export, and consumers may import it under any name they choose. Default exports are less amenable to tree-shaking because bundlers track them by position, not by name.

**Example:** stringy-core deliberately uses named exports rather than default exports so that individual utilities like `formatCurrency` can be eliminated from a bundle if the consumer never imports them.

---

### Namespace export
Bundling many functions under a single object and re-exporting that object, e.g. `export const _s = { maskEmail, maskPhone, formatCurrency, ... }`. Consumers import the whole object at once for convenience. Individual properties on the object cannot be tree-shaken because bundlers cannot statically predict which properties will be accessed at runtime.

**Example:** stringy-core's `_s` object collects all 50+ functions under one import — `import { _s } from 'stringy-core'` — intended for Node scripts and REPL sessions where bundle size does not matter.

---

### Tree-shaking
A bundler optimisation that statically analyses which exports are imported across the entire application and removes all others (dead code elimination). It works because ESM `import` statements are resolved before code runs, giving the bundler a complete map of what is and is not used.

**Example:** An app that imports only `levenshteinDistance` from stringy-core will have all of `textMaskingAndSecurity`, `textFormatting`, and the other seven modules excluded from its production bundle, because the bundler can prove they are never referenced.

---

### Bundler
A tool (Rollup, Webpack, Vite, esbuild) that takes many JavaScript source files and combines them into one or a few output files for deployment. During this process the bundler performs tree-shaking, minification, and scope hoisting.

**Example:** A React app that depends on stringy-core runs Vite as its bundler — Vite's Rollup core performs tree-shaking and emits a final `main.js` that includes only the stringy-core functions the app actually imports.

---

### Algorithms

### Levenshtein distance
The minimum number of single-character edits — insertions, deletions, and substitutions — needed to transform one string into another. It quantifies how "different" two strings are, making it the foundational metric for spell checking, fuzzy search, and diff algorithms.

**Example:** `levenshteinDistance('kitten', 'sitting')` returns `3` because three edits are required: substitute `k` → `s`, substitute `e` → `i`, and insert `g` at the end.

---

### Dynamic programming (DP matrix)
A technique that solves a problem by breaking it into overlapping subproblems, storing each result in a table so it is never recomputed. Levenshtein distance uses an `(m+1) × (n+1)` matrix where each cell represents the minimum edit cost for a prefix pair — each cell depends only on the three cells above, to the left, and diagonally above-left.

**Example:** To compute `levenshteinDistance('saturday', 'sunday')`, stringy-core allocates a 9 × 7 matrix and fills 63 cells exactly once, reading previously computed values rather than recursing, then returns the value in the bottom-right corner.

---

### O(m×n)
Big O notation expressing that the Levenshtein algorithm's running time (and memory) grows proportionally to the product of the two string lengths. If both strings are 100 characters long, the algorithm performs 10,000 cell computations.

**Example:** Calling `levenshteinDistance` on two 1,000-character strings fills a 1,001 × 1,001 matrix — about one million cells — which is why the What-If section flags O(m×n) memory as a practical concern and recommends the two-row space optimisation for very long strings.

---

### isBalancedBrackets (stack-based)
An algorithm that checks whether every opening bracket in a string has a correctly nested, correctly ordered matching closing bracket. It processes the string in a single left-to-right pass, using a stack to track unmatched openers.

**Example:** `isBalancedBrackets('([{}])')` pushes `(`, `[`, `{` in order, then pops and matches each as `}`, `]`, `)` are encountered, ending with an empty stack and returning `true`.

---

### Stack
A data structure where items are added (pushed) and removed (popped) from the same end — Last In, First Out (LIFO). It naturally models nesting: the most recently opened bracket must be closed before any earlier one.

**Example:** In `isBalancedBrackets`, when the input is `{[`, the stack holds `['{', '[']`; the `[` was pushed last, so it must be matched first — if a `}` arrives before a `]`, the mismatch is detected immediately by comparing the popped `[` against the expected partner of `}`.

---

### Regex (Regular Expression)
A pattern language for matching, searching, and extracting text. Patterns describe character classes, repetition, anchors, and groups in a compact syntax that the runtime evaluates against an input string.

**Example:** stringy-core's `textMetadataAndExtraction` module defines 15 regex patterns — one per extractor — compiled once at module scope and reused on each call, so `extractURLs(text)` runs the URL pattern against any input string without recompiling the regex each time.

---

### Internationalisation

### Intl API (Internationalization API)
A built-in browser and Node.js API (standardised as ECMA-402) for locale-sensitive formatting of numbers, dates, currencies, and relative times. The runtime ships locale data for 100+ locales, so no external locale files need to be bundled with the library.

**Example:** `formatCurrency(1299, 'INR', 'en-IN')` delegates to `new Intl.NumberFormat('en-IN', { style: 'currency', currency: 'INR' }).format(1299)` and returns `'₹1,299.00'` using the Indian grouping convention, with zero locale data shipped inside stringy-core.

---

### Locale
A language-and-region code (e.g. `en-IN`, `hi-IN`, `de-DE`) that specifies formatting conventions: decimal and thousands separators, currency symbol placement, date ordering, and calendar system. Different locales can produce dramatically different output from the same number.

**Example:** Passing locale `'en-IN'` to `formatNumber(1234567.89)` produces `'12,34,567.89'` (Indian lakh grouping), while `'de-DE'` produces `'1.234.567,89'` (German convention with dot as thousands separator and comma as decimal).

---

### Intl.NumberFormat
A constructor from the Intl API that formats numbers according to a given locale and set of options (currency, percent, decimal places, grouping). It handles every locale's grouping conventions, digit systems, and sign placement correctly.

**Example:** stringy-core's `formatNumber` and `formatCurrency` functions both create an `Intl.NumberFormat` instance with the caller-supplied locale and options, then call `.format(value)` — the entire locale-correct rendering is handled by the runtime.

---

### Intl.RelativeTimeFormat
A constructor from the Intl API that formats a duration as a human-readable relative time phrase in the given locale — for example `'3 days ago'` in English or `'dans 2 heures'` in French — without hardcoding any language strings in the library.

**Example:** `formatRelativeTime(-3, 'days', 'en')` calls `new Intl.RelativeTimeFormat('en', { numeric: 'auto' }).format(-3, 'days')` and returns `'3 days ago'`, while the same call with `'fr'` returns `'il y a 3 jours'`.

---

### Tooling & Quality

### Husky
A tool that installs scripts into Git hooks so they run automatically at specific points in the Git workflow — most commonly `pre-commit` (before a commit is recorded) and `pre-push`. It ensures quality checks run locally before code reaches the remote.

**Example:** stringy-core's `.husky/pre-commit` file runs `lint-staged` automatically whenever a contributor runs `git commit`, catching ESLint errors and Prettier violations before the commit is finalised.

---

### lint-staged
A tool that runs configured linters and formatters only on files that are currently staged for a commit, rather than on the entire codebase. This keeps pre-commit hooks fast regardless of project size.

**Example:** If a contributor stages only `textMaskingAndSecurity.js`, lint-staged runs ESLint and Prettier on that one file rather than all nine source modules, so the pre-commit check completes in milliseconds instead of seconds.

---

### ESLint
A JavaScript linting tool that statically analyses source code to find and report patterns that violate configurable rules — unused variables, inconsistent return types, use of `var` instead of `const`, and more.

**Example:** ESLint catches if a contributor accidentally leaves `var result` unreferenced in a new stringy-core function, failing the pre-commit hook and prompting a fix before the code is ever committed.

---

### Prettier
An opinionated code formatter that rewrites source files to enforce a consistent style — indentation width, quote style, trailing commas, line length — automatically, without requiring manual style decisions from contributors.

**Example:** If a contributor submits a PR with mixed tab and space indentation in `textGeneration.js`, Prettier's lint-staged integration rewrites the file to use consistent spaces before the commit is allowed, so the PR diff shows only logical changes.

---

### Babel
A JavaScript compiler (transpiler) that transforms modern JavaScript syntax into an older or differently formatted form for compatibility. In stringy-core's case, Babel converts ESM `import`/`export` syntax into CommonJS `require`/`module.exports` so Jest — which runs in a CJS environment — can execute the tests.

**Example:** `babel.config.js` in stringy-core configures `@babel/preset-env` with `{ targets: { node: 'current' } }`, so when Jest loads `textSpecializedOperations.js`, Babel rewrites its `export function levenshteinDistance` into a `module.exports` form Jest understands.

---

### jest.config.js
Jest's configuration file, which tells the test runner how to find tests, which transforms to apply to source files, and how to report results. In an ESM project, this file is where Babel is wired in so Jest can process ES module syntax.

**Example:** stringy-core's `jest.config.js` sets `transform: { '^.+\\.js$': 'babel-jest' }`, instructing Jest to pass every `.js` source file through Babel before executing it, which is what makes the ESM imports in the test files work.

---

### Contribution stub
A function defined with a complete public signature and JSDoc documentation but a body that throws `Error('Not implemented')`, explicitly marked with a `// CONTRIBUTION_STUB` comment. It signals to community contributors exactly what to build without leaving API design decisions open.

**Example:** The `soundex` function in stringy-core is a contribution stub — its parameter name, return type, and expected output format are documented, so a new contributor can implement the Soundex algorithm and submit a PR without needing to discuss what the function should be called or what it should return.

---

### Semantic versioning (semver)
A version numbering convention — `MAJOR.MINOR.PATCH` — with defined rules: increment `MAJOR` for breaking changes (removed or changed API), `MINOR` for new backward-compatible features, and `PATCH` for backward-compatible bug fixes. Published on npm so package managers can safely resolve compatible updates.

**Example:** The `isPalindrome` return-type fix (string → boolean) is debated as either a `PATCH` release (correcting documented behaviour) or a `MAJOR` bump (technically a breaking change for any consumer who called `.toUpperCase()` on the result) — semver forces this ambiguity to be resolved explicitly before the fix ships.
