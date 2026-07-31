# Next.js 15 → 16.2 Migration Playbook

> Load this file when actively performing a migration. Step-by-step, with
> the exact codemod commands and the specific failures you're likely to hit.

---

## Before You Start

```bash
# Check current version and Node.js compatibility first
npx next info
node --version   # must be 20.9+ for Next.js 16

# If on Next.js 14 or older: migrate to 15 FIRST, then to 16.
# Two smaller migrations are safer than one large jump, and skipping
# straight from 14 misses the async-request-API deprecation warnings
# that 15 would have surfaced.
```

```bash
# The automated codemod handles most of the mechanical changes
npx @next/codemod@canary upgrade latest

# Or upgrade packages manually
npm install next@latest react@latest react-dom@latest
```

```bash
# Next.js DevTools MCP server can also drive codemods via natural language
# if configured: "Next Devtools, help me upgrade my Next.js app to version 16"
# or "migrate my app to cache components" — it understands file structure
# and applies the right codemods automatically.
```

---

## Step 1: Async Request APIs

This is almost always the highest-volume change. Every `page.tsx`,
`layout.tsx`, `route.ts`, and `generateMetadata` using `params`,
`searchParams`, `cookies()`, `headers()`, or `draftMode()` needs updating.

```bash
# Generate typed helpers for params/searchParams first — makes the
# migration type-safe and catches missed spots at compile time
npx next typegen
```

```tsx
// Search your codebase for these patterns and fix each one:
// grep -rn "params\." app/ --include="*.tsx" --include="*.ts"
// grep -rn "cookies()" app/
// grep -rn "headers()" app/

// BEFORE
export default function Page({ params }: { params: { slug: string } }) {
  const { slug } = params; // ❌ sync access removed
}

// AFTER
export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params; // ✅
}
```

This applies to: `layout.js`, `page.js`, `route.js`, `default.js`,
`opengraph-image`, `twitter-image`, `icon`, and `apple-icon` files.

---

## Step 2: middleware.ts → proxy.ts

```bash
# Automated codemod for the rename
npx @next/codemod@canary middleware-to-proxy
```

```bash
# Manual steps if the codemod misses anything:
mv middleware.ts proxy.ts
# Rename the exported function: middleware → proxy
# Rename config flags: skipMiddlewareUrlNormalize → skipProxyUrlNormalize
```

```ts
// Audit every instance of request data access in the file — the codemod
// handles renaming but NOT runtime behavior changes. You must manually
// verify:

// 1. No response bodies returned (JSON/HTML) — proxy.ts can only
//    redirect, rewrite, or modify headers
// 2. No 'edge' runtime declaration — proxy.ts is Node.js only
//    Remove: export const runtime = 'edge';
// 3. No heavy auth/DB logic — move real validation to layouts/route handlers
// 4. cookies()/headers() calls inside proxy.ts are now async — same as
//    everywhere else in 16.2+
```

```
Common failure after migration:
"Build error mentioning edge runtime incompatibility"
→ Usually a pre-existing library that doesn't support Edge Runtime.
  The rename surfaces it because the build now correctly discovers
  the file. Remove `export const runtime = 'edge'` or swap to an
  edge-compatible package, or keep the file as middleware.ts if you
  truly need Edge Runtime.

"NextResponse.redirect() calls that worked before now do nothing or loop"
→ Usually a matcher configuration issue — the matcher pattern doesn't
  match the incoming request path, so proxy.ts never runs for that route.
  Debug with: console.log(request.nextUrl.pathname) temporarily.
  Remember: matcher runs against pathname WITHOUT query strings.

"Logout loop — cookie never clears"
→ Classic proxy.ts auth bug. Requires creating a mutable response and
  attaching Set-Cookie headers correctly rather than relying on the
  request object. Check your auth library's Next.js 16 migration guide
  (Auth.js, Clerk, Supabase all published specific guidance for this).
```

---

## Step 3: Caching Model Migration

This is the largest mental-model shift and often the most time-consuming
step. Do not rush it — a rushed migration usually means either "everything
is slow now" (forgot to cache) or "stale data everywhere" (wrong cacheLife).

```bash
# 1. Enable Cache Components
```
```ts
// next.config.ts
const nextConfig: NextConfig = { cacheComponents: true };
```

```bash
# 2. Find every fetch() call that relied on implicit caching
grep -rn "fetch(" app/ --include="*.tsx" --include="*.ts" | grep -v node_modules

# 3. Find every revalidate export — these are now non-functional
grep -rn "export const revalidate" app/

# 4. Find every dynamic export
grep -rn "export const dynamic" app/

# 5. Find every single-argument revalidateTag call — will be TS errors
grep -rn "revalidateTag(" app/
```

```tsx
// Migration pattern per file:

// OLD
export const revalidate = 3600;
export default async function Page() {
  const data = await fetch('https://api.example.com/data');
  return <div>{/* ... */}</div>;
}

// NEW
import { cacheLife } from 'next/cache';

async function getData() {
  'use cache';
  cacheLife({ revalidate: 3600 });
  return fetch('https://api.example.com/data').then(r => r.json());
}

export default async function Page() {
  const data = await getData();
  return <div>{/* ... */}</div>;
}
```

```tsx
// If you used PPR via experimental.ppr in a 15 canary:
// DO NOT upgrade directly to 16 without reading the Cache Components
// migration guide first — PPR behavior changed fundamentally.
// Stay on your current 15 canary until you've mapped out the
// "use cache" equivalent for each PPR boundary you had.
```

**Recommended order for large apps:**
1. Marketing/static pages first (lowest risk, easiest to verify)
2. Blog/docs/content pages (moderate risk, cacheable with clear TTLs)
3. Product/catalog pages (need `cacheTag` for invalidation on updates)
4. User-specific/dashboard pages last (need `"use cache: private"`, highest risk)

---

## Step 4: Turbopack / Webpack Config

```bash
# If your build fails immediately with:
# "ERROR: This build is using Turbopack, with a `webpack` config and no `turbopack` config."
```

```ts
// Migrate custom webpack loaders/rules to turbopack.rules
// OLD
const nextConfig = {
  webpack: (config) => {
    config.module.rules.push({ test: /\.svg$/, use: ['@svgr/webpack'] });
    return config;
  },
};

// NEW
const nextConfig: NextConfig = {
  turbopack: {
    rules: {
      '*.svg': { loaders: ['@svgr/turbopack'], as: '*.js' },
    },
  },
};
```

```bash
# If migration isn't feasible immediately, fall back temporarily:
next build --webpack
# But treat this as technical debt — Turbopack-only plugins are
# increasingly common and webpack fallback won't be supported forever
```

```
Common issues:
"Many webpack-based Next.js plugins (next-images, next-fonts, next-sitemap)
don't work properly yet with Turbopack"
→ Check each plugin's GitHub issues for Turbopack compatibility status
  before migrating. Some have Turbopack-native replacements.

"CSS imports behave differently"
→ Styles must be imported from inside app/ or src/ and imported properly
  in the layout file — Turbopack is stricter about import resolution
  than webpack was.

"Blurry images in production that weren't blurry before"
→ blurDataURL was causing unnecessary compression under the new image
  pipeline. Disable or regenerate blur placeholders after migrating.
```

---

## Step 5: next/image Config Updates

```ts
// next.config.ts additions needed
const nextConfig: NextConfig = {
  images: {
    // If you used `domains`, migrate to remotePatterns (domains deprecated)
    remotePatterns: [
      { protocol: 'https', hostname: '**.example.com' },
    ],
    // If any local images use query strings, add explicit patterns
    localPatterns: [
      { pathname: '/assets/**', search: '?v=*' },
    ],
    // If you optimize images from a private/local network
    dangerouslyAllowLocalIP: true, // only if genuinely needed
  },
};
```

```bash
# Find every use of the deprecated priority prop
grep -rn "priority" app/ --include="*.tsx" | grep -i image
# Replace with preload (for the true LCP image) or loading="eager" /
# fetchPriority="high" (for other above-the-fold images)
```

---

## Step 6: Parallel Routes — Add Missing default.js

```bash
# Find every parallel route slot (@folder) missing a default.js
find app -type d -name "@*" | while read dir; do
  if [ ! -f "$dir/default.js" ] && [ ! -f "$dir/default.tsx" ]; then
    echo "MISSING default.js: $dir"
  fi
done
```

```tsx
// Minimum viable default.js for each missing slot
import { notFound } from 'next/navigation';
export default function Default() {
  notFound(); // or `return null;`
}
```

---

## Step 7: Remove Removed APIs

```bash
# AMP — fully removed, no fallback
grep -rn "useAmp\|amp: true" app/ pages/

# Runtime config — removed, migrate to env vars
grep -rn "serverRuntimeConfig\|publicRuntimeConfig" .
```

```ts
// OLD
module.exports = {
  serverRuntimeConfig: { mySecret: 'secret' },
  publicRuntimeConfig: { staticFolder: '/static' },
};

// NEW — standard environment variables
// .env.local
MY_SECRET=secret
NEXT_PUBLIC_STATIC_FOLDER=/static
```

---

## Step 8: Container / Memory Behavior Changes

If self-hosting in Docker or similar, verify memory limits after upgrading —
Turbopack's build process has different memory characteristics than webpack's.
Watch for OOM kills in CI or container orchestration that didn't happen on 15.

```bash
# Check actual build memory usage
next build --debug   # verbose timing output shows where time/memory goes
```

---

## Step 9: Full Verification Pass

```bash
npx next info                      # confirm version and config
npx tsc --noEmit                   # if using TypeScript
npm run build                       # actual production build
npm test                            # full test suite
npx eslint . --ext .ts,.tsx         # lint — note @next/eslint-plugin-next
                                     # now defaults to Flat Config
```

```bash
# Smoke test the areas most likely to have regressed:
# - Auth flows (proxy.ts + layout validation)
# - Any page relying on caching behavior (check for stale OR
#   unexpectedly-dynamic content)
# - Parallel route slots (all @folders render correctly)
# - Image-heavy pages (check quality, no broken local images)
```

---

## Rollback Plan

```bash
# If blocked, pin back to Next.js 15 maintenance LTS
npm install next@15.5.21 react@19.0 react-dom@19.0

# Document blockers before rolling back
echo "Blocked on: <specific reason>" >> MIGRATION_NOTES.md
```

---

## AGENTS.md — Read Before Generating Code

New Next.js 16 projects ship an `AGENTS.md` file specifically warning AI
coding agents that training data is unreliable for this version. If the
project you're working in has this file, or has `node_modules/next/dist/docs/`
available, read the version-matched docs there before writing migration code
— they're guaranteed to match the exact installed version in a way no
external skill or training data can be.

*Last Updated: 2026-07-30*
