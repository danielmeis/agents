---
name: nextjs
description: >
  Next.js 16.2+ App Router best practices, Cache Components, the "use cache"
  directive, proxy.ts (formerly middleware.ts), Server Components, Server
  Actions, and Turbopack. Use this skill for any Next.js question: routing,
  data fetching, caching strategy, cacheLife/cacheTag, revalidateTag,
  layouts, parallel/intercepting routes, image optimization, and deployment.
  CRITICAL: this skill targets Next.js 16.2+ specifically. Most AI training
  data and tutorials describe Next.js 14/15 patterns that are now WRONG —
  middleware.ts, synchronous params, implicit fetch caching, and revalidate
  exports are all outdated or removed in 16. If code or advice looks like
  Next.js 15 (sync params, middleware.ts, no "use cache", bare revalidateTag
  with one argument), treat it as stale and correct it to the 16.2+ pattern
  shown in this skill. Next.js 16 breaking changes are covered in depth —
  always default to 16.2+ syntax unless the user explicitly states they are
  on an older version.
  React itself (hooks, component patterns): see the react skill.
  TypeScript type system: see the typescript skill.
---

# Next.js 16.2+ Best Practices

> Current stable: **16.2.11** (Active LTS, July 2026) · Next.js 16.0 GA'd
> October 21, 2025 · Next.js 15.5.21 is the Maintenance LTS line if you're
> mid-migration. React 19.2 ships inside Next.js 16 by default.
>
> **This skill assumes Next.js 16.2+.** If you see code using `middleware.ts`,
> synchronous `params`, no `"use cache"` directive, or `revalidateTag(tag)`
> with a single argument — that's Next.js 14/15 syntax. Correct it.

---

## ⚠️ Read This First: Why 16 Breaks Assumptions From Training Data

Next.js 16 is not an incremental release. It changed three fundamental
assumptions that most existing code — and most AI training data — was built
on. If you're generating or reviewing Next.js code and it doesn't account
for these, it is very likely stale 14/15-era output:

1. **How the app is bundled** — Turbopack is the default for both `next dev`
   and `next build`. Webpack requires an explicit `--webpack` flag.
2. **Where request-interception code runs** — `middleware.ts` is deprecated
   in favor of `proxy.ts`, which runs **only** on the Node.js runtime (edge
   is not supported there — if you need edge, you must stay on `middleware.ts`).
3. **What gets cached and when** — caching is now fully explicit. Nothing is
   cached by default. You opt in per-function/component with `"use cache"`.

Each of these can silently break a production app if code was written
assuming 14/15 defaults. Treat any Next.js code you see without awareness of
these three shifts as suspect.

**Next.js itself ships guidance for this exact problem.** New Next.js 16
projects include an `AGENTS.md` file with this message, written directly to
AI coding agents:

> This version has breaking changes — APIs, conventions, and file structure
> may all differ from your training data. Read the relevant guide in
> `node_modules/next/dist/docs/` before writing any code. Heed deprecation
> notices.

Take that at face value. If a project has `node_modules/next/dist/docs/`
available, check it for the exact installed version's guides before
generating Next.js code — the docs shipped with the package are guaranteed
to match the version actually running, which this skill (or any training
data) cannot guarantee for a fast-moving framework. If a project has its own
`AGENTS.md`, read it before writing any code, the same way you would this
skill file.

---

## Version Feature Map

| Feature | Next.js 15 | Next.js 16.2+ |
|---------|-----------|----------------|
| Default bundler | Webpack (Turbopack opt-in via `--turbopack`) | **Turbopack** (opt-out via `--webpack`) |
| Request interception file | `middleware.ts` | **`proxy.ts`** (`middleware.ts` deprecated, edge-only) |
| `params` / `searchParams` | Sync (with async migration path) | **Async only** — sync access fully removed |
| `cookies()` / `headers()` / `draftMode()` | Sync (with async migration path) | **Async only** |
| Data caching | Implicit (`fetch` cached by default) | **Explicit** — `"use cache"` directive required |
| `revalidate` route export | Supported | **Removed** — use `cacheLife()` inside `"use cache"` |
| `revalidateTag(tag)` | One argument | **Two arguments required**: `revalidateTag(tag, profile)` |
| Parallel route slots | `default.js` optional | **`default.js` required** — build fails without it |
| `next/image` `priority` prop | Supported | **Deprecated** — use `preload` |
| AMP support | Deprecated | **Removed entirely** |
| `serverRuntimeConfig`/`publicRuntimeConfig` | Supported | **Removed** — use env vars |
| React version | 19.0 | **19.2** (Activity, useEffectEvent, View Transitions) |
| Minimum Node.js | 18.18+ | **20.9+** |

---

## Core Principles

- Server Components by default; add `'use client'` only when you need
  interactivity, browser APIs, or React state/effects
- All request-time APIs (`params`, `searchParams`, `cookies()`, `headers()`,
  `draftMode()`) are **async** — always `await` them
- Nothing is cached unless you explicitly mark it with `"use cache"`
- Keep `proxy.ts` thin — routing decisions only, never auth logic or DB calls
- Prefer Server Actions over API routes for form mutations within the app
- Use `next/image` and `next/font` for all images and fonts — never raw `<img>`/`<link>`

---

## Async Request APIs (Breaking Since 16.0, No Exceptions)

> Next.js 16.2+ — sync access was removed entirely; there is no fallback

```tsx
// app/blog/[slug]/page.tsx
interface PageProps {
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
}

export default async function BlogPost({ params, searchParams }: PageProps) {
  const { slug } = await params;
  const { highlight } = await searchParams;

  const post = await getPost(slug);
  return <Article post={post} highlighted={highlight === 'true'} />;
}

// generateMetadata also needs async params
export async function generateMetadata({ params }: PageProps) {
  const { slug } = await params;
  const post = await getPost(slug);
  return { title: post.title };
}
```

```tsx
// cookies(), headers(), draftMode() — all async now
import { cookies, headers, draftMode } from 'next/headers';

export default async function Page() {
  const cookieStore = await cookies();
  const headersList = await headers();
  const { isEnabled } = await draftMode();

  const token = cookieStore.get('session')?.value;
  const userAgent = headersList.get('user-agent');

  return <div>{/* ... */}</div>;
}
```

```bash
# Generate type helpers for typed async params automatically
npx next typegen

# Automated codemod for the whole migration
npx @next/codemod@canary upgrade latest
```

---

## proxy.ts (Formerly middleware.ts)

> Next.js 16.2+ — the file is renamed; the runtime constraint is new

```ts
// proxy.ts — at the project root (or inside app/ — check your Next.js docs
// version; placement conventions have shifted between 16.0 and 16.2 patch releases)
import { NextRequest, NextResponse } from 'next/server';

// Function MUST be named 'proxy' (default or named export both work,
// named 'proxy' is recommended even for default exports)
export function proxy(request: NextRequest) {
  const token = request.cookies.get('session')?.value;

  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*'],
};
```

### Critical Constraints

- **Runtime is Node.js only, and cannot be configured.** If you need the
  Edge runtime, you must keep using the deprecated `middleware.ts` — there
  is no `export const runtime = 'edge'` option for `proxy.ts`.
- **`proxy.ts` cannot return response bodies** (JSON/HTML). It can only
  redirect, rewrite, or modify headers — it's a thin network gatekeeper,
  not a backend handler.
- **Never put authentication logic, database calls, or heavy session
  validation in `proxy.ts`.** This was the root cause of CVE-2025-29927 (a
  middleware auth-bypass vulnerability under high load). Do real auth
  checks in Layouts or Route Handlers instead — `proxy.ts` can do a cheap
  cookie-presence check for redirect purposes, nothing more.

```ts
// BAD: heavy auth logic in proxy.ts — the anti-pattern that caused CVE-2025-29927
export function proxy(request: NextRequest) {
  const session = await validateJWTAgainstDatabase(request); // ❌ never do this here
  if (!session) return NextResponse.redirect(new URL('/login', request.url));
}

// GOOD: cheap presence check only; real validation happens deeper in the stack
export function proxy(request: NextRequest) {
  const hasCookie = request.cookies.has('session'); // ❌ cheap check, not real auth
  if (!hasCookie) return NextResponse.redirect(new URL('/login', request.url));
  return NextResponse.next();
}

// Real validation belongs in a layout or route handler:
// app/dashboard/layout.tsx
export default async function DashboardLayout({ children }: { children: ReactNode }) {
  const session = await getValidatedSession(); // real DB/JWT check happens here
  if (!session) redirect('/login');
  return <>{children}</>;
}
```

```bash
# Migration
mv middleware.ts proxy.ts
# Then rename the exported function from `middleware` to `proxy`
# Config flags with "middleware" in the name are also renamed:
# skipMiddlewareUrlNormalize → skipProxyUrlNormalize
# The codemod handles both automatically:
npx @next/codemod@canary middleware-to-proxy
```

---

## Cache Components & the `"use cache"` Directive

> Next.js 16.2+ — this is the single biggest mental model shift from 15

Caching is now **fully explicit**. No `fetch` call, page, or component is
cached unless you say so. This is the opposite of the Next.js 15 model,
where fetches were cached by default and you opted out.

### Enable Cache Components

```ts
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  cacheComponents: true,
};

export default nextConfig;
```

### The Three Directive Scopes

```tsx
// 1. FILE-LEVEL: every async export in the file is cached
// app/blog/page.tsx
'use cache';

export default async function BlogPage() {
  const posts = await fetchAllPosts();
  return <PostList posts={posts} />;
}
// Use for: fully static pages with zero per-user data (marketing, docs, blog index)

// 2. COMPONENT-LEVEL: cache a specific component, not the whole page
// components/ProductList.tsx
import { cacheLife, cacheTag } from 'next/cache';

async function ProductList({ category }: { category: string }) {
  'use cache';
  cacheLife('hours');
  cacheTag('products', `category-${category}`);

  const data = await getProducts(category);
  return <List items={data} />;
}
// Best practice: place "use cache" as close to the data fetch as possible —
// component or function level, not layout level, for granular control

// 3. FUNCTION-LEVEL: cache a data-fetching function directly
async function getProduct(id: string) {
  'use cache';
  cacheLife('hours');
  cacheTag(`product-${id}`);

  return db.products.findUnique({ where: { id } });
}
```

### Personalized Data — `"use cache: private"`

```tsx
// For per-user cached data — never share this across users
async function getUserDashboard(userId: string) {
  'use cache: private';
  cacheLife('minutes');

  return db.dashboards.findByUser(userId);
}

// ⚠️ CRITICAL: Cache Components don't know about your security model.
// If you cache a function that returns user-specific data with the plain
// "use cache" directive (not "use cache: private"), that data CAN be
// served to other users. Always verify cached functions don't leak data
// across users.
```

### `cacheLife` — Controlling Duration

```tsx
import { cacheLife } from 'next/cache';

// Built-in profiles: 'seconds', 'minutes', 'hours', 'days', 'weeks', 'max'
async function getData() {
  'use cache';
  cacheLife('hours'); // uses the built-in 'hours' profile
  return fetch('https://api.example.com/data').then(r => r.json());
}

// Custom inline profile — three numbers control behavior:
// stale: how long the cached value serves without checking
// revalidate: triggers a background refresh after this long
// expire: hard upper limit before the entry is evicted entirely
async function getPricing() {
  'use cache';
  cacheLife({ stale: 60, revalidate: 300, expire: 3600 });
  return fetch('https://api.example.com/pricing').then(r => r.json());
}
```

```ts
// Define named custom profiles globally in next.config.ts
// next.config.ts
const nextConfig: NextConfig = {
  cacheComponents: true,
  cacheLife: {
    catalog: { stale: 3600, revalidate: 900, expire: 86400 },
  },
};
// Then reference by name: cacheLife('catalog')
```

### `cacheTag` and Invalidation

```tsx
import { cacheLife, cacheTag } from 'next/cache';

async function getCachedUser(id: string) {
  'use cache';
  cacheLife('hours');
  cacheTag('users', `user-${id}`);
  return db.users.findUnique({ where: { id } });
}
```

```tsx
// app/actions.ts
'use server';
import { revalidateTag } from 'next/cache';

export async function updateProduct(id: string, data: ProductInput) {
  await db.products.update({ where: { id }, data });

  // revalidateTag REQUIRES a second argument in 16.2+ — a cacheLife profile
  // The single-argument form is deprecated and produces a TypeScript error
  revalidateTag(`product-${id}`, 'max'); // 'max' recommended for most cases —
                                          // enables background (stale-while-revalidate) refresh
  revalidateTag('products', 'hours');
}

// revalidateTag(tag) alone — ❌ DEPRECATED, will not type-check in 16.2+
// revalidateTag('products'); // TypeScript error
```

### `updateTag` — Immediate Write-Through Invalidation

```tsx
'use server';
import { updateTag } from 'next/cache';

export async function deleteComment(id: string) {
  await db.comments.delete({ where: { id } });
  // updateTag: immediate invalidation, no stale-while-revalidate window
  // Use when the user needs to see the change instantly (their own action)
  await updateTag(`comments-${id}`);
}
```

### `revalidateTag` vs `updateTag` vs `refresh`

| API | Use when |
|-----|----------|
| `revalidateTag(tag, profile)` | Eventual consistency is fine — blog posts, product listings, background content |
| `updateTag(tag)` | User needs to see the effect of their own action immediately |
| `refresh()` (from `next/cache`) | Refresh the client-side router from a Server Action without a full tag invalidation — good for notification badges, cart counts |

### Migration From the Old Model

```tsx
// OLD (Next.js 15) — implicit caching, revalidate export
export const revalidate = 3600; // ❌ removed in 16

export default async function Page() {
  const data = await fetch('https://api.example.com/data'); // cached by default in 15
  return <div>{/* ... */}</div>;
}

// NEW (Next.js 16.2+) — explicit, function/component scoped
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
// OLD: dynamic = 'force-static'
export const dynamic = 'force-static'; // ❌ deprecated in 16

// NEW: "use cache" with cacheLife('max')
'use cache';
import { cacheLife } from 'next/cache';
cacheLife('max');
```

**Practical migration path:**
1. Enable `cacheComponents: true` in `next.config.ts`
2. Add `"use cache"` to genuinely static pages first (landing, blog, docs, pricing)
3. Set up `cacheLife` profiles for content needing time-based revalidation
4. Use `cacheTag` + `revalidateTag`/`updateTag` in Server Actions for on-demand invalidation
5. Expect a temporary performance dip on unmigrated pages — they now render
   dynamically by default until you explicitly cache them

---

## Partial Prerendering (PPR) via Cache Components

```tsx
// PPR now works through Cache Components rather than a standalone flag.
// If you were using experimental.ppr in Next.js 15 canaries, do NOT
// upgrade directly — see the Cache Components migration guide first.
// next.config.ts — legacy PPR flag (15 canary only, not 16)
// const nextConfig = { experimental: { ppr: true } }; // ❌ stay on 15 canary if using this

// In 16.2+, mixing static and dynamic content happens naturally:
// cached components render into the static shell; everything else streams
export default async function ProductPage({ params }: PageProps) {
  const { id } = await params;

  return (
    <div>
      <ProductHeader productId={id} />       {/* cached — part of static shell */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <LiveReviews productId={id} />        {/* dynamic — streams in */}
      </Suspense>
    </div>
  );
}

async function ProductHeader({ productId }: { productId: string }) {
  'use cache';
  cacheLife('hours');
  const product = await getProduct(productId);
  return <h1>{product.name}</h1>;
}

async function LiveReviews({ productId }: { productId: string }) {
  // No "use cache" — always dynamic, fetched fresh on every request
  const reviews = await getReviews(productId);
  return <ReviewList reviews={reviews} />;
}
```

---

## Turbopack (Default Bundler)

```bash
# Turbopack is now automatic — no flags needed
next dev
next build

# Opt out to Webpack (only if you have unmigrated custom webpack config)
next build --webpack
next dev --webpack

# Force Turbopack even with a webpack config present (ignores the webpack config)
next build --turbopack
```

```ts
// next.config.ts — Turbopack config replaces webpack config
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  turbopack: {
    rules: {
      '*.svg': {
        loaders: ['@svgr/turbopack'],
        as: '*.js',
      },
    },
    resolveAlias: {
      'legacy-lib': './src/lib/legacy-adapter',
    },
  },
};

export default nextConfig;
```

```ts
// REMOVED entirely in 16 — these config keys now cause build errors:
// webpack: () => { ... }              → use turbopack.rules
// swcMinify: true                     → always on via Turbopack now
// experimental.appDir: true           → always on, App Router is the only router
```

```bash
# If your build fails with:
# "ERROR: This build is using Turbopack, with a `webpack` config and no `turbopack` config."
# You have three choices:
# 1. Migrate the webpack config to turbopack.rules
# 2. next build --webpack (keep using webpack)
# 3. next build --turbopack (ignore the webpack config, use Turbopack defaults)
```

```ts
// Turbopack filesystem caching (beta) — faster cold starts on large apps
const nextConfig: NextConfig = {
  experimental: {
    turbopackFileSystemCacheForDev: true,
  },
};
```

---

## next/image Changes

> Next.js 16.2+ — several breaking default changes, mostly security-motivated

```tsx
// priority prop is DEPRECATED — use preload instead
// OLD (15)
<Image src="/hero.jpg" alt="Hero" width={1200} height={600} priority />

// NEW (16.2+)
<Image src="/hero.jpg" alt="Hero" width={1200} height={600} preload />
// In most cases, loading="eager" or fetchPriority="high" is preferred over
// preload — reserve preload for the single true LCP (largest contentful
// paint) image on the page
```

```ts
// next.config.ts — image defaults that changed in 16
const nextConfig: NextConfig = {
  images: {
    // domains is DEPRECATED — use remotePatterns for stricter matching
    remotePatterns: [
      { protocol: 'https', hostname: '**.example.com' },
    ],
    minimumCacheTTL: 14400,     // now 4 hours by default (was 60s)
    maximumRedirects: 3,        // now capped at 3 by default (was unlimited)
    qualities: [75],            // now defaults to [75] only (was 1-100 range)
    dangerouslyAllowLocalIP: false, // must explicitly opt in for private network images
  },
};
```

```tsx
// Local images with query strings now require explicit config
// (prevents enumeration attacks)
<Image src="/assets/photo?v=1" alt="Photo" width={100} height={100} />
// Requires:
const nextConfig: NextConfig = {
  images: {
    localPatterns: [{ pathname: '/assets/**', search: '?v=*' }],
  },
};
```

```bash
# next/legacy/image is deprecated — migrate to next/image
npx @next/codemod next-image-to-legacy-image  # if migrating FROM pages router legacy
npx @next/codemod next-image-experimental      # then to the modern next/image API
```

---

## Parallel Routes — `default.js` Now Required

> Next.js 16.2+ — this is a hard build failure, not a warning

```
app/
├── @analytics/
│   ├── default.js   ← REQUIRED in 16.2+, build fails without it
│   └── page.js
├── @team/
│   ├── default.js   ← REQUIRED in 16.2+
│   └── page.js
└── layout.js
```

```tsx
// app/@analytics/default.js — minimum viable default.js
import { notFound } from 'next/navigation';

export default function Default() {
  notFound(); // or `return null;` to render nothing for unmatched routes
}
```

---

## Server Actions

```tsx
// Colocated in a Server Component file
async function updateProfile(formData: FormData) {
  'use server';
  const name = formData.get('name') as string;
  await db.users.update({ where: { id: userId }, data: { name } });
  revalidateTag('profile', 'max');
}

export default function ProfileForm() {
  return (
    <form action={updateProfile}>
      <input name="name" />
      <button type="submit">Save</button>
    </form>
  );
}
```

```ts
// Or in a dedicated 'use server' file — preferred for reuse across components
// app/actions.ts
'use server';

import { revalidateTag, updateTag } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const post = await db.posts.create({ data: { title } });
  revalidateTag('posts', 'max');
  redirect(`/blog/${post.slug}`);
}
```

```tsx
// React 19.2's useActionState + useFormStatus pair naturally with Server Actions
// (full patterns live in the react skill — this is the Next.js-specific wiring)
'use client';
import { useActionState } from 'react';
import { createPost } from './actions';

export function NewPostForm() {
  const [state, action, isPending] = useActionState(createPost, null);
  return (
    <form action={action}>
      <input name="title" />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating…' : 'Create Post'}
      </button>
    </form>
  );
}
```

---

## Routing Fundamentals

```
app/
├── layout.tsx              # Root layout — required, wraps everything
├── page.tsx                 # / route
├── loading.tsx               # Suspense fallback for this segment
├── error.tsx                  # Error boundary for this segment (must be 'use client')
├── not-found.tsx               # 404 UI for this segment
├── blog/
│   ├── page.tsx              # /blog
│   └── [slug]/
│       └── page.tsx           # /blog/:slug (dynamic segment)
├── (marketing)/               # Route group — doesn't affect URL
│   ├── about/page.tsx          # /about
│   └── pricing/page.tsx         # /pricing
└── dashboard/
    ├── layout.tsx               # Nested layout for /dashboard/*
    └── @analytics/                # Parallel route slot
        ├── default.js             # Required in 16.2+
        └── page.tsx
```

```tsx
// Dynamic segments
// app/blog/[slug]/page.tsx → params.slug
// app/shop/[...categories]/page.tsx → params.categories (catch-all array)
// app/shop/[[...categories]]/page.tsx → optional catch-all

// generateStaticParams — pre-render dynamic routes at build time
export async function generateStaticParams() {
  const posts = await getAllPostSlugs();
  return posts.map(post => ({ slug: post.slug }));
}
```

---

## Error Handling

```tsx
// app/dashboard/error.tsx — MUST be a Client Component
'use client';

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    logErrorToService(error);
  }, [error]);

  return (
    <div>
      <h2>Something went wrong</h2>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

```tsx
// app/dashboard/not-found.tsx
export default function NotFound() {
  return <div>This page could not be found.</div>;
}

// Trigger it programmatically
import { notFound } from 'next/navigation';
export default async function Page({ params }: PageProps) {
  const { id } = await params;
  const item = await getItem(id);
  if (!item) notFound();
  return <ItemView item={item} />;
}
```

---

## Metadata

```tsx
// Static metadata
export const metadata: Metadata = {
  title: 'My App',
  description: 'App description',
};

// Dynamic metadata — params is async, same as page components
export async function generateMetadata({ params }: PageProps): Promise<Metadata> {
  const { slug } = await params;
  const post = await getPost(slug);
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: { images: [post.coverImage] },
  };
}
```

---

## Deployment Notes

```bash
# next dev and next build now use SEPARATE output directories,
# enabling concurrent execution
# next dev  → .next/dev
# next build → .next (production)
# A lockfile mechanism prevents multiple concurrent dev/build instances

# Self-hosted: minimum Node.js version is now 20.9+
node --version  # must be 20.9 or higher

# Verify current version and check for available updates
npx next info
```

```bash
# ESLint config: @next/eslint-plugin-next now defaults to Flat Config
# (aligns with ESLint v10 dropping legacy config support)
# Review the plugin API reference if migrating an existing .eslintrc setup
```

---

## Reference Files

Load these when the task goes deeper than the summaries above:

- **`references/migration.md`** — full step-by-step 15 → 16.2 migration
  playbook: codemod commands, container/memory behavior changes, common
  build failures and fixes, CSS import path changes under Turbopack
- **`references/caching-deep-dive.md`** — advanced Cache Components patterns:
  nested cache scopes, cache debugging, `cacheComponents` interaction with
  ISR, ISR-to-Cache-Components migration edge cases, ownership patterns for
  large teams
- **`references/routing-and-rendering.md`** — parallel routes, intercepting
  routes, route groups, streaming with Suspense, loading UI patterns,
  View Transitions (React 19.2), Activity-based navigation state

> React hooks, component patterns, and TypeScript typing for React: see the
> **react** skill. TypeScript type system itself: see the **typescript** skill.

*Last Updated: 2026-07-30*
