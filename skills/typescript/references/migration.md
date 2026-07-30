# TypeScript 5.9 → 6.0 Migration Playbook

> Load this file when actively performing a migration, not just reading
> about what changed. Step-by-step, with rollback strategy.

---

## Pre-Migration Checklist

```bash
# 1. Confirm current version and audit for issues before touching anything
npx tsc --version

# 2. Run the static migration analysis — does NOT run the type checker,
#    only scans for removed/changed syntax and config patterns.
#    Safe to run on 5.9 — makes zero changes to your build.
npx tsc --ts6-migration > migration-report.txt

# 3. Review the report. It includes file names, line numbers, and fixes for:
#    - Removed module formats (amd, umd, system)
#    - target: "es5" usage
#    - moduleResolution: "classic" usage
#    - Namespace augmentation across module boundaries
#    - Import assertion syntax (assert vs with)
#
#    It does NOT catch:
#    - New strict mode errors (run tsc --noEmit separately for those)
#    - Runtime behavior changes
#    - types: [] related missing-global errors
```

---

## Step-by-Step Migration

### Step 1: Upgrade the package, don't change config yet

```bash
npm install --save-dev typescript@6.0
npx tsc --noEmit  # see the full blast radius before changing anything
```

### Step 2: Fix the `types` field first (highest-impact, most common break)

```jsonc
// Add explicit types — this alone resolves most "Cannot find name" errors
{
  "compilerOptions": {
    "types": ["node"]   // add "jest", "vitest", "bun", etc. as your project needs
  }
}
```

```bash
# Identify what you actually need by checking installed @types packages
ls node_modules/@types
```

### Step 3: Fix `rootDir` if output structure changed

```bash
# Symptom: dist/src/index.js instead of dist/index.js
```

```jsonc
{
  "compilerOptions": {
    "rootDir": "./src"
  }
}
```

### Step 4: Decide your strict mode strategy

**Option A — full strict, fix everything now (best long-term, more upfront work):**

```jsonc
{ "compilerOptions": { "strict": true } }
```

```bash
npx tsc --noEmit 2>&1 | tee strict-errors.txt
wc -l strict-errors.txt  # gauge the size of the problem
```

**Option B — explicit opt-out, schedule the work separately (buys time):**

```jsonc
{ "compilerOptions": { "strict": false } }
```

**Option C — gradual rollout, recommended for large codebases:**

```jsonc
{
  "compilerOptions": {
    "strict": false,
    // Enable incrementally, in this order (highest value → lowest friction first)
    "strictNullChecks": true,        // catches the most real bugs
    "noImplicitAny": true,           // forces explicit typing
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,  // often the most tedious — do last
    "noImplicitThis": true,
    "alwaysStrict": true
  }
}
```

Track progress with a burndown — count remaining errors per flag weekly:

```bash
# Script: count errors when flipping one flag at a time
for flag in strictNullChecks noImplicitAny strictPropertyInitialization; do
  echo "=== $flag ==="
  npx tsc --noEmit --$flag true 2>&1 | grep -c "error TS"
done
```

### Step 5: Fix module/target/moduleResolution together

These three interact — changing one without the others often causes new
errors. Test as a set:

```jsonc
{
  "compilerOptions": {
    "module": "nodenext",           // or "esnext" for bundler-only projects
    "moduleResolution": "nodenext", // matches module: nodenext
    "target": "ES2025"
  }
}
```

```bash
# If you're on a bundler (Vite, esbuild, webpack) rather than running
# Node directly, use "bundler" resolution instead:
```

```jsonc
{
  "compilerOptions": {
    "module": "esnext",
    "moduleResolution": "bundler"
  }
}
```

### Step 6: Fix side-effect import errors

```ts
// noUncheckedSideEffectImports: true now verifies these imports resolve
import './styles.css';        // now checked — must actually exist
import './polyfills';         // now checked — must actually exist

// If you use CSS Modules or similar and get false positives, add a
// declaration file:
```

```ts
// src/types/css.d.ts
declare module '*.css';
declare module '*.scss';
```

### Step 7: Run the codemod for remaining boilerplate changes

```bash
npx ts5to6
# Review the diff carefully — codemods are a starting point, not a guarantee
git diff --stat
```

### Step 8: Full verification pass

```bash
npx tsc --noEmit                    # type-check
npm run build                        # actual build
npm test                             # full test suite
npx eslint . --ext .ts,.tsx          # lint (see tooling.md for eslint config)
```

---

## Monorepo Considerations

```jsonc
// Each package's tsconfig.json needs its own explicit types/rootDir —
// defaults don't cascade the way you might expect across project references

// packages/api/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "types": ["node"],
    "rootDir": "./src",
    "outDir": "./dist"
  },
  "references": [{ "path": "../shared" }]
}

// packages/web/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "types": ["vite/client"],   // different types per package — this is expected
    "rootDir": "./src",
    "outDir": "./dist"
  },
  "references": [{ "path": "../shared" }]
}
```

```bash
# Migrate leaf packages (no internal dependents) first, then work inward
# Use project references build mode to catch cross-package breaks early
npx tsc --build --verbose
```

---

## CI: Running Old and New Versions in Parallel

Useful during a gradual team-wide migration, or when testing 7.0 readiness
without committing to it:

```json
// package.json
{
  "scripts": {
    "typecheck:5.9": "tsc --noEmit -p tsconfig.ts59.json",
    "typecheck:6.0": "tsc --noEmit -p tsconfig.json",
    "typecheck": "npm run typecheck:6.0"
  },
  "devDependencies": {
    "typescript": "^6.0.3",
    "typescript-5.9": "npm:typescript@5.9.3"
  }
}
```

```yaml
# .github/workflows/typecheck.yml
name: Type Check
on: [pull_request]
jobs:
  typecheck:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        ts-version: ['5.9', '6.0']
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run typecheck:${{ matrix.ts-version }}
        continue-on-error: ${{ matrix.ts-version == '6.0' }}  # non-blocking during transition
```

---

## Rollback Plan

If the migration reveals blockers you can't resolve quickly:

```bash
# Revert the typescript package
npm install --save-dev typescript@5.9.3

# Keep the tsconfig changes that are compatible with 5.9 (types field, rootDir)
# — these don't hurt on 5.9 and reduce future migration work

# Document blockers before rolling back so the next attempt is faster
echo "Blocked on: <reason>" >> MIGRATION_NOTES.md
```

---

## Known Gotchas Encountered in Real Migrations

```ts
// 1. Temporal type conflicts with polyfill packages
// If you have temporal-polyfill or @js-temporal/polyfill installed,
// their types are NOT compatible with TS 6.0's built-in Temporal types.
// Fix: remove the polyfill's @types dependency (keep the runtime polyfill
// if your target environment needs it, just drop the type declarations)

// 2. Stable type ordering for published libraries
// If you publish a library and snapshot-test generated .d.ts output,
// enable this to match the ordering behavior TS 7.0 will use:
```

```jsonc
{
  "compilerOptions": {
    "stableTypeOrdering": true
    // Note: this can add up to 25% type-checking slowdown — don't enable
    // globally unless you specifically need deterministic output ordering
  }
}
```

```ts
// 3. isolatedDeclarations surfaces new errors in files with inferred
// complex return types — you'll need explicit return type annotations
// on exported functions with non-trivial inferred types

// BEFORE (inferred type, breaks isolatedDeclarations)
export function getConfig() {
  return { host: 'localhost', port: 3000, features: computeFeatures() };
}

// AFTER (explicit annotation required)
export function getConfig(): AppConfig {
  return { host: 'localhost', port: 3000, features: computeFeatures() };
}
```

*Last Updated: 2026-07-29*
