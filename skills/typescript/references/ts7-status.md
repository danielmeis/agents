# TypeScript 7.0 / 7.1 Status Notes

> This file is a snapshot as of late July 2026. TS7 ecosystem readiness is
> changing week to week — before recommending a 7.0 migration, web-search for
> anything newer than this file's date and update assumptions accordingly.

---

## Timeline So Far

| Date | Event |
|------|-------|
| March 11, 2025 | Microsoft publicly announces the Go-based native TypeScript port ("Project Corsa") |
| March 23, 2026 | TypeScript 6.0 GA — last JavaScript-based compiler release; bridge release to 7.0 |
| June 18, 2026 | TypeScript 7.0 Release Candidate published |
| July 8, 2026 | TypeScript 7.0 GA — Go-native compiler and language service |
| July 2026 (ongoing) | Ecosystem tooling breakage widely reported; `typescript-eslint` issue closed "not planned" |
| TBD (7.1) | Stable programmatic API expected — "at least several months" from 7.0 GA per Microsoft |

---

## What TS7 Actually Is

Not a rewrite — a **line-by-line port** of the existing JavaScript compiler
(codenamed Strada) into Go (codenamed Corsa). The team used the same ~20,000
internal test cases from the JS compiler to validate the port; of those,
only 74 discrepancies were found, each documented as either known incomplete
work or an intentional change tied to a deprecation.

This means: **type-checking behavior itself is preserved almost exactly**.
What's different is the surrounding surface — no programmatic API yet, and a
handful of edge cases (see "Known Rough Edges" below).

---

## Verified Performance Numbers (Independent Reports, Not Just Microsoft's)

| Project | Before (6.0) | After (7.0) | Speedup |
|---------|-------------|-------------|---------|
| VS Code (~1.5M lines) | 77.8s (Microsoft) / 125.7s (independent) | 7.5s / 10.6s | ~10.4x / ~11.9x |
| Slack CI type-check step | 7.5 min | 1.25 min | 6x (40% faster merge queue) |
| Canva editor first-error latency | 58s | 4.8s | ~12x |
| Playwright, TypeORM | — | — | 8–13x reported |

The speed claims are real and independently corroborated, not just vendor
marketing. The question is not "is it fast" — it's "does your toolchain work
on it yet."

---

## The Core Blocker: No Stable Programmatic API Until 7.1

`tsc` as a command-line tool works today on 7.0. Anything that **imports
`typescript` as a library** to walk the AST, extract type information, or
drive the compiler programmatically does not yet have a stable interface to
call. This is the single root cause of nearly all reported 7.0 breakage.

Microsoft's own framing (from the 7.0 announcement): *"Even though 7.0 RC is
close to production-ready, we won't have a stable programmatic API available
until at least several months from now with TypeScript 7.1."*

### Affected — confirmed broken as of GA + 2 weeks

- **`typescript-eslint`**: peer dependency range blocks `typescript@7`
  install with an ERESOLVE error. Issue #12518 was filed on GA day and
  closed "not planned" — the maintainers' position is that this needs to be
  solved upstream, in TypeScript itself.
- **ESLint core repos**: blocked transitively behind `typescript-eslint`.
- **`ts-jest`**: works fine as long as the `typescript` package it resolves
  stays on 6.x. Breaks with confusing transform failures (not install
  failures) if pointed at `@typescript/native-preview` directly — it calls
  internal Strada APIs that Corsa doesn't expose.
- **`ts-morph`** and custom AST transformers: same root cause as ts-jest.
  These tend to **fail silently or produce subtly wrong output** rather than
  crash outright — audit generated output carefully if you've moved these
  to 7.0 prematurely.
- **Vue, Svelte, Astro, MDX** (via Volar): cannot use TS 7 for template
  type-checking. Volar embeds TypeScript's programmatic API directly.
- **Angular** template type-checking: same limitation.

### Known Rough Edges (Beyond the API Gap)

- `tsgo` (the 7.0 binary) drops generic type parameters from JSDoc
  `@typedef` declarations that use `@template` — affects JS/JSDoc-typed
  codebases specifically, not `.ts` files
- `skipLibCheck: true` does not suppress certain parse-level errors from
  third-party `.d.ts` files under the new compiler — a compatibility survey
  of 15 pipeline tools found 9 calling Strada API functions that no longer
  exist in Corsa
- Monorepos using project references are described as "the roughest edge"
  in early adopter reports — test thoroughly before relying on `--build`
  mode with 7.0

---

## Microsoft's Official Migration Path

```bash
# 1. Install TypeScript 7.0 for its tsc binary
npm install --save-dev typescript@^7.0.0

# 2. Install the compatibility package for tools that need the 6.0 API
npm install --save-dev @typescript/typescript6

# This provides:
# - A `tsc6` executable (6.0's compiler, side by side with 7.0's tsc)
# - Re-exported TypeScript 6.0 programmatic API for tools that import
#   `typescript` as a library (typescript-eslint, ts-jest, etc.)
```

```json
// package.json — recommended npm alias approach, per Microsoft's own guidance
{
  "devDependencies": {
    "typescript": "npm:@typescript/typescript6@^6.0.3",
    "typescript-native-preview": "npm:@typescript/native-preview@^7.0.0"
  }
}
```

This lets tooling that peer-depends on `typescript` (like `typescript-eslint`)
resolve the 6.0 API, while you separately invoke the 7.0 native binary for
fast `tsc --noEmit` runs in CI.

---

## Recommendation Given Current State (Late July 2026)

1. **Migrate to TypeScript 6.0 now.** This is not optional groundwork — it's
   where the tsconfig defaults you'll need for 7.0 anyway get sorted out,
   and it's already a stable, well-supported target.
2. **Do not point your primary `typescript` dependency at 7.0** if your
   toolchain includes `typescript-eslint`, `ts-jest`, `ts-morph`, or a
   framework using Volar-based checking (Vue, Svelte, Astro).
3. **Optionally run `tsgo --noEmit` in parallel** via
   `@typescript/native-preview` purely for fast type-checking feedback in
   CI or local dev — this doesn't touch your build or lint toolchain and
   is low-risk today.
4. **Re-evaluate once TypeScript 7.1 ships** with the stable programmatic
   API — at that point `typescript-eslint` and friends should be able to
   update their peer ranges and the blockers above should clear.
5. **Before acting on any of this, search for news newer than this file.**
   The situation is moving fast; check `typescript-eslint`'s GitHub issues,
   the official TypeScript DevBlog, and your specific tools' release notes
   for anything published after this snapshot.

*Last Updated: 2026-07-29*
