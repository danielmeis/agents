# Next.js 16.2+ Cache Components — Deep Dive

> Load this file for advanced caching patterns, debugging, ISR migration
> edge cases, and team ownership strategies for large apps.

---

## Nested Cache Scopes

Cache scopes can nest — an outer `"use cache"` boundary and an inner one
with a different `cacheLife` are both valid and independently tracked:

```tsx
// Outer: page-level shell, cached for a day
export default async function CategoryPage({ params }: PageProps) {
  const { category } = await params;
  return (
    <div>
      <CategoryHeader category={category} />   {/* cached separately, see below */}
      <Suspense fallback={<ProductGridSkeleton />}>
        <ProductGrid category={category} />      {/* cached separately, shorter TTL */}
      </Suspense>
      <LiveStockBanner category={category} />    {/* not cached — always fresh */}
    </div>
  );
}

async function CategoryHeader({ category }: { category: string }) {
  'use cache';
  cacheLife('days'); // category metadata rarely changes
  cacheTag(`category-${category}`);
  const info = await getCategoryInfo(category);
  return <h1>{info.name}</h1>;
}

async function ProductGrid({ category }: { category: string }) {
  'use cache';
  cacheLife('minutes'); // inventory/pricing changes more often
  cacheTag(`category-${category}`, 'products');
  const products = await getProducts(category);
  return <Grid items={products} />;
}

async function LiveStockBanner({ category }: { category: string }) {
  // No "use cache" — this is intentionally dynamic, always request-fresh
  const stockLevel = await getRealtimeStock(category);
  return stockLevel < 10 ? <Banner>Low stock!</Banner> : null;
}
```

Each cached boundary is its own cache entry — invalidating one tag doesn't
touch the others unless they share that tag.

---

## Debugging Cache Behavior

```tsx
// In development, Next.js DevTools shows cache hit/miss info in the
// terminal output when Cache Components is enabled. Watch for:
// - "Cache MISS" on requests you expected to be cached (check cacheLife/tags)
// - "Dynamic hole" warnings — a component inside a cached boundary that
//   reads request-time data (cookies, headers, searchParams) forces the
//   entire boundary to be dynamic

// Common "dynamic hole" mistake:
async function ProductList({ category }: { category: string }) {
  'use cache';
  cacheLife('hours');

  const cookieStore = await cookies(); // ❌ reading cookies inside "use cache"
  // This either errors or silently forces the whole function dynamic,
  // defeating the purpose of caching it

  const products = await getProducts(category);
  return <Grid items={products} />;
}

// FIX: pass request-time data in as a prop from an uncached parent instead
export default async function Page() {
  const cookieStore = await cookies(); // read here, in the dynamic parent
  const region = cookieStore.get('region')?.value ?? 'US';
  return <ProductList category="shoes" region={region} />;
}

async function ProductList({ category, region }: { category: string; region: string }) {
  'use cache';
  cacheLife('hours');
  cacheTag(`category-${category}`, `region-${region}`);
  const products = await getProducts(category, region);
  return <Grid items={products} />;
}
```

```ts
// Third-party debugging tools exist that wrap cached functions to surface
// cache misses, dynamic holes, missing tags, and repeated fetches directly
// in the terminal during development. Search current npm ecosystem for
// "use cache debugger" tooling if you need this level of visibility on a
// large migration.
```

---

## `cacheComponents` Interaction With ISR

Traditional ISR (`generateStaticParams` + implicit revalidation) still works
in 16.2+, but it's being superseded by Cache Components for new code:

```tsx
// ISR still works — generateStaticParams pre-renders at build time
export async function generateStaticParams() {
  const posts = await getAllPostSlugs();
  return posts.map(post => ({ slug: post.slug }));
}

// But combine it with "use cache" for the data fetching, not the old
// revalidate export, to get the full benefit of the new caching model
async function getPost(slug: string) {
  'use cache';
  cacheLife('hours');
  cacheTag(`post-${slug}`);
  return db.posts.findUnique({ where: { slug } });
}

export default async function BlogPost({ params }: PageProps) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <Article post={post} />;
}
```

**When to still reach for `generateStaticParams`**: when you want the pages
to exist as fully static HTML at build time (best TTFB, works even if your
data source is briefly down). Cache Components' `"use cache"` still renders
dynamically on cache miss — `generateStaticParams` guarantees the page
exists ahead of any request.

---

## Cache Ownership Patterns for Large Teams

On large codebases, uncoordinated `cacheTag` usage causes two problems:
tag collisions (two features accidentally sharing a tag, causing
over-invalidation) and orphaned tags (a tag nothing ever revalidates,
serving stale data forever).

```ts
// Recommended: centralize tag naming in one module, don't string-literal
// tags scattered across the codebase

// lib/cache-tags.ts
export const cacheTags = {
  product: (id: string) => `product:${id}`,
  productsByCategory: (category: string) => `products:category:${category}`,
  userDashboard: (userId: string) => `dashboard:${userId}`,
  allProducts: 'products:all',
} as const;

// Usage — consistent, discoverable, greppable
import { cacheTags } from '@/lib/cache-tags';

async function getProduct(id: string) {
  'use cache';
  cacheTag(cacheTags.product(id));
  return db.products.findUnique({ where: { id } });
}

// Invalidation — same source of truth
export async function updateProduct(id: string, data: ProductInput) {
  'use server';
  await db.products.update({ where: { id }, data });
  revalidateTag(cacheTags.product(id), 'max');
}
```

```ts
// Document ownership per tag family — who's responsible for revalidating it
// lib/cache-tags.ts (extended with comments as living documentation)
export const cacheTags = {
  // Owned by: catalog team. Revalidated in: app/actions/products.ts
  product: (id: string) => `product:${id}`,
  // Owned by: growth team. Revalidated in: app/actions/dashboard.ts
  userDashboard: (userId: string) => `dashboard:${userId}`,
} as const;
```

---

## `cacheLife` Profile Strategy by Content Type

| Content type | Suggested profile | Reasoning |
|---|---|---|
| Marketing/landing pages | `'max'` or a custom profile with a very large `expire` | Rarely changes, safe to cache aggressively |
| Blog posts, docs | `'days'` + `cacheTag` per post | Long-lived, invalidate explicitly on publish/edit |
| Product catalog | `'hours'` + `cacheTag` per category | Balance freshness against origin load |
| Pricing/inventory | `'minutes'` or custom `{ stale: 60, revalidate: 300 }` | Changes frequently, staleness has real cost |
| User dashboards | `'use cache: private'` + `'minutes'` | Per-user, moderate freshness need |
| Live/real-time data | No `"use cache"` at all | Genuinely needs to be dynamic every request |

```ts
// Centralize these as named profiles in next.config.ts so the whole team
// uses consistent, documented values instead of ad hoc inline objects
// Note: `expire` is a number of seconds, not a boolean — there's no
// literal "never" value, so marketing content uses a very large number instead
const nextConfig: NextConfig = {
  cacheComponents: true,
  cacheLife: {
    marketing:  { stale: 86400,  revalidate: 604800, expire: 31536000 }, // ~1 year
    blog:       { stale: 3600,   revalidate: 86400,  expire: 2592000 },
    catalog:    { stale: 300,    revalidate: 3600,    expire: 86400 },
    pricing:    { stale: 60,     revalidate: 300,      expire: 3600 },
    dashboard:  { stale: 30,     revalidate: 120,       expire: 600 },
  },
};
```

---

## Testing Cached Components

```tsx
// Cache Components complicate testing because "use cache" changes
// behavior between dev and prod, and between cached/uncached calls.

// For unit tests: extract the data-fetching logic from the cache directive
// where practical, so you can test the logic without the caching wrapper

// Instead of:
async function getProduct(id: string) {
  'use cache';
  cacheLife('hours');
  return db.products.findUnique({ where: { id } });
}

// Consider:
async function fetchProduct(id: string) {
  return db.products.findUnique({ where: { id } }); // pure, testable
}
async function getProduct(id: string) {
  'use cache';
  cacheLife('hours');
  return fetchProduct(id);
}

// Test fetchProduct directly; getProduct is a thin caching wrapper
// that doesn't need its own unit test (integration/e2e tests cover it)
```

*Last Updated: 2026-07-31*
