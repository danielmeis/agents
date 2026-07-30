# TypeScript Tooling — Deep Reference

> Load this file for ESLint configuration, build tool integration, monorepo
> project references, declaration file authoring, and the TS7 compatibility
> matrix in full detail.

---

## typescript-eslint Configuration

```bash
npm install --save-dev typescript-eslint eslint
```

```js
// eslint.config.js (flat config — standard as of ESLint 9+)
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked, // requires type info — slower, catches more
  {
    languageOptions: {
      parserOptions: {
        projectService: true,        // auto-discovers tsconfig.json files
        tsconfigRootDir: import.meta.dirname,
      },
    },
    rules: {
      '@typescript-eslint/no-unused-vars': ['warn', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/consistent-type-imports': 'error', // enforce `import type`
      '@typescript-eslint/no-non-null-assertion': 'warn',
    },
  },
);
```

### ⚠️ TypeScript 7.0 Compatibility (as of GA, July 2026)

```bash
# typescript-eslint's peer dependency range blocks typescript@7 entirely:
npm install typescript@7
# npm error ERESOLVE could not resolve
# npm error peer typescript@"<6.1.0" from typescript-eslint

# The GitHub issue tracking this (typescript-eslint #12518) was closed
# "not planned" — the maintainers' position is that the fix must come from
# TypeScript's side (the stable programmatic API, landing in 7.1).

# Workaround: keep the `typescript` package your tooling resolves at 6.x
# via an npm alias, while running tsc itself from 7.0 for fast type-checking
```

```json
// package.json — run 7.0's tsc for type-checking, keep eslint on 6.0's API
{
  "devDependencies": {
    "typescript": "npm:@typescript/typescript6@^6.0.3",
    "typescript-native": "npm:typescript@^7.0.0"
  },
  "scripts": {
    "typecheck:fast": "typescript-native/bin/tsc --noEmit",
    "lint": "eslint ."
  }
}
```

---

## Build Tool Integration

### Vite

```ts
// vite.config.ts — Vite handles TS via esbuild/oxc for transpilation only;
// it does NOT type-check. Run tsc separately, or use vite-plugin-checker.
import { defineConfig } from 'vite';
import checker from 'vite-plugin-checker';

export default defineConfig({
  plugins: [
    checker({ typescript: true }), // surfaces type errors in dev server overlay
  ],
});
```

```bash
npm install --save-dev vite-plugin-checker
```

### esbuild

```js
// esbuild transpiles only — no type-checking at all
// Always run tsc --noEmit in a separate step (pre-commit, CI, or watch mode)
```

```json
// package.json
{
  "scripts": {
    "build": "tsc --noEmit && esbuild src/index.ts --bundle --outfile=dist/index.js",
    "dev": "concurrently \"tsc --noEmit --watch\" \"esbuild --watch ...\""
  }
}
```

### tsup (library bundling)

```ts
// tsup.config.ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['esm', 'cjs'],
  dts: true,               // generates .d.ts — respects isolatedDeclarations
  clean: true,
  sourcemap: true,
  target: 'es2025',
});
```

---

## Monorepo: Project References

```jsonc
// tsconfig.base.json — shared settings
{
  "compilerOptions": {
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "isolatedDeclarations": true,
    "composite": true,           // required for project references
    "declaration": true,          // required for project references
    "declarationMap": true
  }
}

// packages/shared/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "types": ["node"],
    "rootDir": "./src",
    "outDir": "./dist"
  }
}

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

// Root tsconfig.json — orchestrates the build graph
{
  "files": [],
  "references": [
    { "path": "./packages/shared" },
    { "path": "./packages/api" },
    { "path": "./packages/web" }
  ]
}
```

```bash
# Build in dependency order, only rebuilding what changed
npx tsc --build

# Force a clean rebuild
npx tsc --build --force

# Watch mode across the whole graph
npx tsc --build --watch
```

---

## Declaration Files (`.d.ts`) Authoring

```ts
// Publishing a library? isolatedDeclarations (stable in 6.0) enables
// parallel .d.ts generation — but requires explicit return types on
// all exported functions with non-trivial inferred types

// GOOD: explicit return type — works with isolatedDeclarations
export function createClient(config: ClientConfig): ApiClient {
  return new ApiClientImpl(config);
}

// BAD: inferred return type — breaks isolatedDeclarations
export function createClient(config: ClientConfig) {
  return new ApiClientImpl(config); // implicit return type, error under isolatedDeclarations
}

// Ambient declaration files for non-TS modules
// src/types/svg.d.ts
declare module '*.svg' {
  const content: string;
  export default content;
}

// Augmenting existing types (module augmentation)
// src/types/express.d.ts
import 'express';
declare module 'express' {
  interface Request {
    user?: { id: string; role: string };
  }
}
```

---

## Full TS7 Ecosystem Compatibility Matrix

As of TypeScript 7.0 GA (July 8, 2026):

| Tool / Framework | Status | Notes |
|---|---|---|
| `tsc` (compile + type-check) | ✅ Production-ready | The Go-native compiler itself works |
| `typescript-eslint` | ❌ Broken | Peer dep blocks install; issue closed "not planned" |
| ESLint core | ❌ Blocked | Blocked behind `typescript-eslint`'s resolution |
| `ts-jest` | ⚠️ Partial | Works if `typescript` resolves to 6.x; breaks pointed at native compiler |
| `ts-node` | ⚠️ Check current status | Depends on programmatic API — verify before relying on it |
| `ts-morph` | ❌ Unreliable | Fails silently or produces wrong output — audit output carefully |
| Vue (Volar-based checking) | ❌ Not yet | Embeds TS programmatic API |
| Svelte | ❌ Not yet | Same limitation |
| Astro | ❌ Not yet | Same limitation |
| MDX | ❌ Not yet | Same limitation |
| Angular template checking | ❌ Not yet | Same limitation |
| Custom AST transformers | ❌ Unreliable | Same root cause as ts-morph |

**Expected resolution**: TypeScript 7.1 ships the stable programmatic API.
Microsoft has stated it's "at least several months" from the 7.0 GA date
(July 2026), and is actively working with the maintainers of the affected
projects. Re-check this matrix against current release notes before
committing to a 7.0 migration for anything beyond ad hoc `tsc --noEmit` runs.

```bash
# Safe today: use tsgo purely for fast type-checking, keep everything else on 6.x
npm install --save-dev @typescript/native-preview
npx tsgo --noEmit   # fast type-check, doesn't touch your build/lint toolchain
```

---

## Recommended package.json Setup (Mid-2026 Transition Period)

```json
{
  "devDependencies": {
    "typescript": "^6.0.3",
    "@typescript/native-preview": "^7.0.0",
    "typescript-eslint": "^8.0.0",
    "eslint": "^9.0.0"
  },
  "scripts": {
    "typecheck": "tsc --noEmit",
    "typecheck:fast": "tsgo --noEmit",
    "lint": "eslint .",
    "build": "tsc --noEmit && tsup"
  }
}
```

This keeps `typescript-eslint` and all your existing tooling stable on 6.0
while giving you the option to spot-check compile speed and catch any
7.0-specific discrepancies via `tsgo` in parallel, without committing your
whole toolchain to it yet.

*Last Updated: 2026-07-29*
