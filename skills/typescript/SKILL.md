---
name: typescript
description: >
  TypeScript 5.9 and TypeScript 6.0 best practices, migration guidance, and
  type-system patterns. Use this skill for any TypeScript question: type
  design, generics, utility types, tsconfig configuration, strict mode,
  module resolution, the migration from TypeScript 5.x to 6.0, and the nine
  changed compiler defaults in 6.0 (strict, types: [], rootDir, module,
  target, moduleResolution, esModuleInterop, noUncheckedSideEffectImports,
  libReplacement). Also trigger for TypeScript 6.0 language additions:
  Temporal API types, Map/WeakMap getOrInsert, RegExp.escape, es2025 target,
  subpath imports. TypeScript 7.0 (the Go-native compiler) is covered only
  at a "what to know" level — see the TS7 section for why full adoption is
  premature and what breaks (typescript-eslint, ts-jest, ts-morph, Vue/Svelte/
  Astro template checking) until the 7.1 programmatic API ships.
  React prop typing basics live in the react skill; this skill covers the
  type system itself in depth.
---

# TypeScript Best Practices (5.9 baseline → 6.0 target)

> Current versions as of July 2026: TypeScript **6.0** (GA March 23, 2026,
> last JavaScript-based compiler) · TypeScript **5.9.3** (still common in
> existing projects) · TypeScript **7.0** (GA July 8, 2026, Go-native compiler
> — **not yet the target of this skill**, see the TS7 section below for why).
>
> **This skill targets TypeScript 6.0.** Code examples assume 6.0 defaults.
> Anywhere behavior differs from 5.9, it's called out explicitly with
> `> TS 5.9` / `> TS 6.0` markers so you can tell what changes when you upgrade.

---

## Why Target 6.0, Not 7.0, Right Now

TypeScript 7.0 is a from-scratch Go port of the compiler (Project Corsa) —
roughly 10x faster type-checking, but it **does not ship a stable
programmatic API**. That API isn't landing until TypeScript 7.1. Tools that
drive the compiler through code, not just `tsc`, depend on that API:

| Tool | Status on TS 7.0 (as of GA, July 2026) |
|------|------------------------------------------|
| `typescript-eslint` | Broken — GitHub issue closed "not planned"; peer dependency range blocks `typescript@7` install entirely (ERESOLVE) |
| `ts-jest` | Fails with cryptic transform errors if pointed at the native compiler; works if `typescript` stays pinned to 6.x |
| `ts-morph` / custom AST transformers | Fails silently or produces subtly wrong output — worse than a hard crash |
| Vue, Svelte, Astro, MDX template checking | Cannot use TS 7 yet — these embed TypeScript's programmatic API via Volar |
| Angular template type-checking | Same limitation as above |

**The practical takeaway**: `tsc` itself (command-line compiling and
type-checking) is production-ready on 7.0. Anything that imports
`typescript` as a library is not. Microsoft publishes `@typescript/typescript6`
— a compatibility package with a `tsc6` binary — specifically so tooling can
keep running against the 6.0 API while `tsc` itself runs on 7.

```json
// package.json — running 7.0's tsc while keeping tooling on the 6.0 API
{
  "devDependencies": {
    "typescript": "^7.0.0",
    "@typescript/typescript6": "npm:typescript@^6.0.3"
  }
}
```

**Recommendation**: adopt TypeScript 6.0 now — it's the version that clears
the tsconfig debt and gets your codebase 7.0-ready. Hold on migrating the
`typescript` package your build tooling depends on to 7.0 until either your
critical tools (`typescript-eslint`, `ts-jest`, framework template checkers)
confirm compatibility, or TypeScript 7.1 ships with the stable programmatic API.

---

## TypeScript 6.0: The Nine Changed Defaults

This is the single most important thing to know about 6.0. If your
`tsconfig.json` relied on implicit defaults, upgrading from 5.9 to 6.0
**will break your build** — not because your code is wrong, but because the
compiler now enforces what it previously left permissive.

| Setting | 5.9 default | 6.0 default | Impact |
|---------|-------------|-------------|--------|
| `strict` | `false` | `true` | Hundreds of new errors possible in unconfigured projects |
| `types` | all `@types/*` auto-included | `[]` (empty) | Missing globals from `@types/node`, `@types/jest`, etc. |
| `rootDir` | inferred from input files | tsconfig directory | Output nesting changes (`dist/src/index.js` instead of `dist/index.js`) |
| `module` | `commonjs` | `esnext` | ESM output instead of CJS |
| `target` | `es2016`-ish | `es2025` (floating) | Modern syntax emitted; ES5 polyfills no longer auto-included |
| `moduleResolution` | `node` | `bundler` | Resolution algorithm changes for bundler-based projects |
| `esModuleInterop` | `false` | `true` | Default import interop behavior changes |
| `noUncheckedSideEffectImports` | `false` | `true` | Bare `import "./styles.css"` now checked for existence |
| `libReplacement` | `true` | `false` | Fewer unnecessary module resolution watches (performance win) |

### The `types: []` Change (Most Commonly Missed)

> TS 6.0

Before 6.0, omitting `types` in `compilerOptions` meant "include every
`@types/*` package found in `node_modules` automatically." In 6.0, the
default is an empty array — **nothing is included automatically**.

```jsonc
// tsconfig.json — BEFORE 6.0 (implicit, worked without listing anything)
{
  "compilerOptions": {
    // types field omitted = all @types/* auto-included
  }
}

// tsconfig.json — REQUIRED as of 6.0
{
  "compilerOptions": {
    "types": ["node", "jest"]   // list exactly what you use
  }
}
```

Symptoms if you miss this: `Cannot find name 'process'`, `Cannot find name
'describe'`, `Cannot find name 'Buffer'` — ambient globals from `@types/node`,
`@types/jest`, `@types/bun`, etc. silently disappear.

```bash
# To temporarily restore old (5.9-style) behavior while you audit:
# "types": ["*"]
# This is an escape hatch, not a long-term fix — it defeats the
# performance win this change was made for.
```

**Performance upside**: Microsoft reports 20–50% build time improvements on
projects that previously auto-loaded many unused `@types/*` packages —
this is real, not just a config chore.

### The `rootDir` Change

```jsonc
// If your build output suddenly nests as dist/src/index.js instead of
// dist/index.js, this is why. Set it explicitly:
{
  "compilerOptions": {
    "rootDir": "./src"
  }
}
```

### Strict Mode by Default

```jsonc
// If you're not ready for strict mode across the whole codebase:
{
  "compilerOptions": {
    "strict": false   // explicit opt-out — buys time, schedule the real fix
  }
}

// Better: gradual adoption — enable individual flags one at a time
{
  "compilerOptions": {
    "strict": false,
    "strictNullChecks": true,   // usually the highest-value flag to enable first
    "noImplicitAny": true       // enable next
    // add strictPropertyInitialization, strictFunctionTypes, etc. incrementally
  }
}
```

### Removed / Deprecated Options

```jsonc
// REMOVED entirely in 6.0 — these are now compile errors:
// "target": "es5"                  → compile to es2015+, transpile down separately if needed
// "moduleResolution": "classic"    → use "bundler" or "node16"/"nodenext"
// "module": "amd" | "umd" | "system" → use "esnext" or "commonjs"

// Import assertions replaced by import attributes:
// OLD: import data from "./data.json" assert { type: "json" };
// NEW: import data from "./data.json" with { type: "json" };

// Temporary escape hatch — silences deprecation warnings in 6.0 ONLY
// Does NOT carry forward to 7.0; deprecated options are removed there entirely
{
  "compilerOptions": {
    "ignoreDeprecations": "6.0"   // short-term only — not a real fix
  }
}
```

### Migration Tooling

```bash
# Static analysis pass — scans for patterns that will break, WITHOUT
# running the type checker. Safe to run on 5.9 before upgrading.
tsc --ts6-migration

# Codemod that automates most tsconfig.json changes
npx ts5to6

# After upgrading, surface all errors before making further changes
tsc --noEmit
```

### Recommended 6.0 tsconfig.json Baseline

```jsonc
{
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "lib": ["ES2025", "DOM"],           // dom.iterable now merged into "DOM" — no need to list separately
    "target": "ES2025",
    "types": ["node"],                   // add "jest"/"vitest"/etc. as needed
    "strict": true,
    "rootDir": "./src",
    "outDir": "./dist",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "isolatedDeclarations": true,        // stable in 6.0 — enables parallel .d.ts generation
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

---

## New Language Features in 6.0

> TS 6.0

```ts
// Map/WeakMap upsert methods — cleans up the check-then-set pattern
const cache = new Map<string, number>();

// OLD (5.9 and earlier)
let timeout: number;
if (cache.has('timeout')) {
  timeout = cache.get('timeout')!;
} else {
  timeout = 5000;
  cache.set('timeout', timeout);
}

// NEW (6.0+)
const timeoutValue = cache.getOrInsert('timeout', 5000);

// For expensive default computation — only runs if key is missing
const computed = cache.getOrInsertComputed('cacheKey', () => performExpensiveCalculation());

// RegExp.escape — safe regex construction from user input
function matchWholeWord(word: string, text: string): RegExpMatchArray | null {
  const escaped = RegExp.escape(word);       // no more manual escaping
  const regex = new RegExp(`\\b${escaped}\\b`, 'g');
  return text.match(regex);
}

// Temporal API types — built in, no polyfill types needed
// CAUTION: written from scratch — NOT interassignable with temporal-polyfill
// or @js-temporal/polyfill types. Remove the polyfill's type dependency if
// your runtime supports Temporal natively.
const today = Temporal.Now.plainDateISO();
const target = Temporal.PlainDate.from('2026-09-01');
const diff = today.until(target, { largestUnit: 'day' });
console.log(`${diff.days} days until launch`);

// Subpath imports (Node.js #/ prefix) — maps to package.json "imports" field
import { helper } from '#internal/helper';

// lib simplification — dom.iterable and dom.asynciterable now merged into "dom"
// OLD: "lib": ["dom", "dom.iterable"]
// NEW: "lib": ["dom"]
```

---

## Core Type Design Principles

- Prefer `interface` for object shapes that might be extended; `type` for
  unions, intersections, tuples, and mapped types
- Never use `any` — use `unknown` and narrow, or be specific
- Avoid type assertions (`as`) unless you have information TypeScript can't infer
- Prefer `readonly` for data that shouldn't be mutated after creation
- Design types to make invalid states unrepresentable — use discriminated unions

```ts
// BAD: any disables all type checking
function process(data: any) { return data.value; } // no safety at all

// GOOD: unknown forces narrowing before use
function process(data: unknown): string {
  if (typeof data === 'object' && data !== null && 'value' in data) {
    return String((data as { value: unknown }).value);
  }
  throw new Error('Invalid data shape');
}

// BETTER: know your shape, type it properly
interface ProcessInput { value: string }
function process(data: ProcessInput): string { return data.value; }
```

---

## Discriminated Unions — Making Invalid States Unrepresentable

```ts
// BAD: optional fields allow invalid combinations
interface RequestState {
  loading?: boolean;
  data?: User;
  error?: string;
}
// Nothing stops { loading: true, data: {...}, error: "oops" } — all three at once

// GOOD: discriminated union — only valid combinations are representable
type RequestState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: User }
  | { status: 'error'; error: string };

function render(state: RequestState) {
  switch (state.status) {
    case 'idle':    return <Idle />;
    case 'loading': return <Spinner />;
    case 'success': return <UserView user={state.data} />; // data is narrowed, guaranteed present
    case 'error':   return <ErrorMessage message={state.error} />; // error guaranteed present
  }
}
```

---

## Utility Types

```ts
interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'member';
}

// Partial — all properties optional (great for update payloads)
function updateUser(id: string, changes: Partial<User>): void { /* ... */ }

// Required — all properties required (opposite of Partial)
type CompleteUser = Required<Partial<User>>; // back to fully required

// Pick — select specific properties
type UserPreview = Pick<User, 'id' | 'name'>;

// Omit — exclude specific properties
type PublicUser = Omit<User, 'email'>;

// Record — map keys to a consistent value type
type UsersById = Record<string, User>;
type RoleCounts = Record<User['role'], number>; // { admin: number; member: number }

// Readonly — prevent mutation
type FrozenUser = Readonly<User>;

// ReturnType / Parameters — derive types from functions
function createUser(name: string, email: string): User { /* ... */ }
type CreateUserArgs = Parameters<typeof createUser>;   // [string, string]
type CreateUserReturn = ReturnType<typeof createUser>; // User

// Awaited — unwrap Promise types (essential for async function return types)
async function fetchUser(): Promise<User> { /* ... */ }
type FetchedUser = Awaited<ReturnType<typeof fetchUser>>; // User, not Promise<User>

// NonNullable — strip null/undefined
type DefiniteUser = NonNullable<User | null | undefined>; // User
```

---

## `satisfies` — Validate Without Widening

```ts
// Problem: explicit type annotation loses literal type inference
const config: Record<string, string | number> = {
  host: 'localhost',
  port: 3000,
};
config.port; // typed as string | number — lost the fact it's specifically a number

// Problem: no annotation means no validation
const config2 = {
  host: 'localhost',
  port: 3000,
  // typo below is NOT caught
  timeuot: 5000,
};

// SOLUTION: satisfies validates the shape AND keeps literal inference
const config3 = {
  host: 'localhost',
  port: 3000,
} satisfies Record<string, string | number>;

config3.port; // typed as number (preserved!) — not string | number
// config3.foo would be a compile error — typo caught
```

---

## Generics

```ts
// Generic function — reusable across types while preserving type safety
function firstItem<T>(items: T[]): T | undefined {
  return items[0];
}
const num = firstItem([1, 2, 3]);        // number | undefined
const str = firstItem(['a', 'b']);       // string | undefined

// Constrained generics — restrict what T can be
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
const user = { id: '1', name: 'Alice' };
const name = getProperty(user, 'name');  // string
// getProperty(user, 'invalid')          // compile error — 'invalid' not a key of user

// Generic constraints with default types
interface ApiResponse<TData = unknown> {
  data: TData;
  status: number;
  error: string | null;
}
function isSuccess<T>(response: ApiResponse<T>): response is ApiResponse<T> & { data: T } {
  return response.status >= 200 && response.status < 300 && response.error === null;
}

// Generic classes
class Repository<T extends { id: string }> {
  private items = new Map<string, T>();

  add(item: T): void { this.items.set(item.id, item); }
  get(id: string): T | undefined { return this.items.get(id); }
  getAll(): T[] { return Array.from(this.items.values()); }
}
const userRepo = new Repository<User>();
```

---

## Type Narrowing

```ts
// typeof narrowing
function process(value: string | number) {
  if (typeof value === 'string') {
    return value.toUpperCase(); // narrowed to string
  }
  return value.toFixed(2); // narrowed to number
}

// in operator narrowing
interface Circle { kind: 'circle'; radius: number }
interface Square { kind: 'square'; side: number }
function area(shape: Circle | Square) {
  if ('radius' in shape) {
    return Math.PI * shape.radius ** 2; // narrowed to Circle
  }
  return shape.side ** 2; // narrowed to Square
}

// instanceof narrowing
function handleError(error: unknown) {
  if (error instanceof Error) {
    console.error(error.message); // narrowed to Error
  } else {
    console.error('Unknown error', error);
  }
}

// Custom type guards
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value &&
    'email' in value
  );
}

// Assertion functions — narrow by throwing instead of returning boolean
function assertIsUser(value: unknown): asserts value is User {
  if (!isUser(value)) throw new Error('Not a User');
}
function handle(data: unknown) {
  assertIsUser(data);
  console.log(data.name); // narrowed to User after the assertion
}
```

---

## Template Literal Types

```ts
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type ApiRoute = `/api/${string}`;
type Endpoint = `${HttpMethod} ${ApiRoute}`;

const endpoint: Endpoint = 'GET /api/users'; // valid
// const bad: Endpoint = 'FETCH /api/users'; // compile error — FETCH not a HttpMethod

// Practical use: type-safe event names
type EventName = `on${Capitalize<'click' | 'hover' | 'focus'>}`;
// 'onClick' | 'onHover' | 'onFocus'

// Mapped type combined with template literals — auto-generate handler prop types
type EventHandlers<T extends string> = {
  [K in T as `on${Capitalize<K>}`]: (event: Event) => void;
};
type ButtonHandlers = EventHandlers<'click' | 'hover'>;
// { onClick: (event: Event) => void; onHover: (event: Event) => void }
```

---

## Function Overloads

```ts
// Multiple call signatures for a single function
function createElement(tag: 'a'): HTMLAnchorElement;
function createElement(tag: 'div'): HTMLDivElement;
function createElement(tag: 'input'): HTMLInputElement;
function createElement(tag: string): HTMLElement {
  return document.createElement(tag);
}

const link = createElement('a');   // typed as HTMLAnchorElement
const div  = createElement('div'); // typed as HTMLDivElement

// Prefer a single signature with generics/unions when overloads aren't necessary
// Overloads are for genuinely different behavior per input type, not just
// different-looking equivalent signatures
```

---

## Module System

```ts
// ESM is the 6.0 default — module: "esnext" or "nodenext"
export interface User { id: string; name: string }
export function createUser(name: string): User { /* ... */ }
export default class UserService { /* ... */ }

// Type-only imports — explicit, tree-shakeable, no runtime import
import type { User } from './types';
import { type User as UserType, createUser } from './types'; // mixed import

// Re-exports
export type { User } from './types';
export { createUser } from './user-service';

// Import attributes (6.0+) — replaces the deprecated "assert" syntax
import data from './data.json' with { type: 'json' };

// Subpath imports (6.0+) — Node.js #/ prefix, maps to package.json "imports"
import { helper } from '#internal/helper';
```

---

## Common Type Errors and Fixes

```ts
// "Object is possibly undefined" — with strict mode's strictNullChecks
function getName(user: User | undefined) {
  // return user.name; // ❌ error: user is possibly undefined
  return user?.name;    // ✅ optional chaining
  // OR: if (!user) return undefined; return user.name; // explicit guard
}

// "Property does not exist on type" — usually a real bug, or needs a type guard
interface ApiUser { id: string; fullName: string }
function greet(user: ApiUser) {
  // console.log(user.name); // ❌ property is fullName, not name — real bug caught
  console.log(user.fullName); // ✅
}

// Index signature errors — TS won't let you assume a key exists
const scores: Record<string, number> = { alice: 90 };
// const bob = scores['bob'].toFixed(2); // ❌ with noUncheckedIndexedAccess: bob is number | undefined
const bob = scores['bob'];
if (bob !== undefined) bob.toFixed(2); // ✅ explicit check

// Excess property checks on object literals
interface Config { host: string; port: number }
// const c: Config = { host: 'x', port: 1, extra: true }; // ❌ excess property 'extra'
const c: Config = { host: 'x', port: 1 }; // ✅
const obj = { host: 'x', port: 1, extra: true };
const c2: Config = obj; // ✅ allowed — excess check only applies to literals directly
```

---

## Reference Files

Load these when the task goes deeper than the summaries above:

- **`references/migration.md`** — full step-by-step 5.9 → 6.0 migration
  playbook: codemods, per-flag strict mode rollout, monorepo considerations,
  CI setup for parallel version testing
- **`references/advanced-types.md`** — conditional types, infer, mapped type
  modifiers, recursive types, branded/nominal types, variance
- **`references/tooling.md`** — ESLint (`typescript-eslint`) configuration,
  build tool integration (Vite, esbuild, tsup), monorepo project references,
  declaration file authoring, the current TS7 tooling compatibility matrix
- **`references/ts7-status.md`** — living notes on TypeScript 7.0/7.1 ecosystem
  readiness; check this before considering a 7.0 migration for `typescript`
  itself (not just `tsc` for type-checking)

> React-specific prop typing patterns: see the **react** skill.
> Next.js type patterns (route params, server actions): see the **nextjs** skill.

*Last Updated: 2026-07-29*
