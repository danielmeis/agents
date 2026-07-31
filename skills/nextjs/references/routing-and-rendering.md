# Next.js 16.2+ Routing & Rendering — Deep Dive

> Load this file for parallel routes, intercepting routes, streaming,
> loading UI, and React 19.2 rendering features as they apply to Next.js.

---

## Parallel Routes

Render multiple independent pages in the same layout, each with its own
loading/error state. Every slot needs `default.js` in 16.2+ (see main
SKILL.md — this is a hard build failure if missing).

```
app/dashboard/
├── layout.tsx
├── page.tsx
├── @team/
│   ├── default.tsx
│   ├── page.tsx
│   └── loading.tsx
└── @analytics/
    ├── default.tsx
    ├── page.tsx
    └── loading.tsx
```

```tsx
// app/dashboard/layout.tsx — slots are passed as props matching folder names
export default function DashboardLayout({
  children,
  team,
  analytics,
}: {
  children: ReactNode;
  team: ReactNode;
  analytics: ReactNode;
}) {
  return (
    <div className="dashboard-grid">
      <main>{children}</main>
      <aside>{team}</aside>
      <aside>{analytics}</aside>
    </div>
  );
}
```

```tsx
// Conditional rendering based on state (e.g. auth-gated slots)
export default function DashboardLayout({
  user,
  admin,
}: {
  user: ReactNode;
  admin: ReactNode;
}) {
  const isAdmin = checkAdminStatus();
  return isAdmin ? admin : user;
}
```

---

## Intercepting Routes

Show a route in a different context (e.g. a modal) without losing the
underlying page — classic use case is an Instagram-style photo modal that's
also a full page on direct navigation/refresh.

```
app/
├── feed/
│   └── page.tsx
├── photo/
│   └── [id]/
│       └── page.tsx           # full page — direct nav or refresh
└── @modal/
    ├── default.tsx
    └── (.)photo/
        └── [id]/
            └── page.tsx        # intercepted — shown as modal from feed
```

Convention prefixes:
- `(.)` — intercept a route at the **same** level
- `(..)` — intercept a route **one level up**
- `(..)(..)` — intercept **two levels up**
- `(...)` — intercept from the **root**

```tsx
// app/@modal/(.)photo/[id]/page.tsx — rendered as a modal when navigated
// to from within the feed; app/photo/[id]/page.tsx renders instead on
// direct navigation or refresh
import { Modal } from '@/components/Modal';

export default async function PhotoModal({ params }: PageProps) {
  const { id } = await params;
  const photo = await getPhoto(id);
  return (
    <Modal>
      <PhotoDetail photo={photo} />
    </Modal>
  );
}
```

```tsx
// app/dashboard/layout.tsx — must render both the modal slot and children
export default function Layout({
  children,
  modal,
}: {
  children: ReactNode;
  modal: ReactNode;
}) {
  return (
    <>
      {children}
      {modal}
    </>
  );
}
```

---

## Route Groups

Organize routes without affecting the URL structure — useful for applying
different layouts to logical sections:

```
app/
├── (marketing)/
│   ├── layout.tsx          # marketing-specific layout (nav, footer)
│   ├── about/page.tsx        # /about
│   └── pricing/page.tsx       # /pricing
└── (app)/
    ├── layout.tsx            # app-specific layout (sidebar, auth check)
    ├── dashboard/page.tsx      # /dashboard
    └── settings/page.tsx        # /settings
```

```tsx
// app/(marketing)/layout.tsx
export default function MarketingLayout({ children }: { children: ReactNode }) {
  return (
    <>
      <MarketingNav />
      {children}
      <MarketingFooter />
    </>
  );
}

// app/(app)/layout.tsx — completely different chrome, same root layout parent
export default async function AppLayout({ children }: { children: ReactNode }) {
  const session = await getValidatedSession(); // real auth check here
  if (!session) redirect('/login');
  return (
    <div className="app-shell">
      <Sidebar user={session.user} />
      {children}
    </div>
  );
}
```

---

## Streaming With Suspense

```tsx
// Stream slow data without blocking the whole page
export default async function Dashboard() {
  return (
    <div>
      <Header />                              {/* renders immediately */}
      <Suspense fallback={<StatsSkeleton />}>
        <SlowStats />                          {/* streams in when ready */}
      </Suspense>
      <Suspense fallback={<ActivitySkeleton />}>
        <RecentActivity />                      {/* independent stream */}
      </Suspense>
    </div>
  );
}

async function SlowStats() {
  const stats = await getExpensiveStats(); // takes 2+ seconds
  return <StatsPanel data={stats} />;
}
```

```tsx
// loading.tsx — automatic Suspense boundary for an entire route segment
// app/dashboard/loading.tsx
export default function Loading() {
  return <DashboardSkeleton />;
}
// Next.js wraps page.tsx in this Suspense boundary automatically —
// no manual <Suspense> needed at the route level
```

---

## View Transitions (React 19.2, via Next.js 16.2+)

```tsx
'use client';
import { unstable_ViewTransition as ViewTransition } from 'react';
// Note: API name may stabilize (drop unstable_ prefix) in future React
// releases — check current React version's exact export name

export function ProductCard({ product }: { product: Product }) {
  return (
    <ViewTransition>
      <Link href={`/products/${product.id}`}>
        <img src={product.image} alt={product.name} />
        <h3>{product.name}</h3>
      </Link>
    </ViewTransition>
  );
}

// Paired with matching ViewTransition on the destination page,
// this animates the shared element (image) between list and detail views
```

---

## `<Activity>` for Tab/Panel Navigation State

```tsx
// React 19.2's Activity component (see react skill for full API) pairs
// well with Next.js client-side navigation for preserving state across
// tab switches within a single route, without a full page navigation
'use client';
import { Activity, useState } from 'react';

export function DashboardTabs() {
  const [tab, setTab] = useState<'overview' | 'reports'>('overview');
  return (
    <>
      <TabBar current={tab} onChange={setTab} />
      <Activity mode={tab === 'overview' ? 'visible' : 'hidden'}>
        <OverviewPanel />
      </Activity>
      <Activity mode={tab === 'reports' ? 'visible' : 'hidden'}>
        <ReportsPanel />  {/* preserves scroll position and fetched data when hidden */}
      </Activity>
    </>
  );
}
```

---

## Prefetching & Navigation (16.2+ Improvements)

Next.js 16.2 overhauled the prefetching pipeline — mostly automatic, but
worth understanding for performance tuning:

- **Layout deduplication**: when prefetching multiple URLs sharing a common
  layout, the layout downloads once instead of per-URL — significant win on
  apps with deep nested layouts
- **Incremental prefetching**: only uncached segments are fetched on
  navigation, not the whole route tree

```tsx
// Link prefetching is automatic in the viewport by default
import Link from 'next/link';

<Link href="/products/123">View Product</Link>
// Prefetches automatically when the link enters the viewport (production only)

// Disable prefetching for links you know won't be visited soon
<Link href="/rarely-visited" prefetch={false}>Rarely Visited</Link>
```

```tsx
// refresh() from next/cache — sync client router state after a Server
// Action without a full tag-based cache invalidation (e.g. notification
// badges, cart counts)
'use server';
import { refresh } from 'next/cache';

export async function markNotificationRead(id: string) {
  await db.notifications.update({ where: { id }, data: { read: true } });
  await refresh(); // updates client-visible state, lighter than revalidateTag
}
```

---

## Loading UI Patterns

```tsx
// Skeleton components should match the real content's layout closely
// to avoid layout shift when the real content streams in
function StatsSkeleton() {
  return (
    <div className="stats-grid" aria-busy="true" aria-label="Loading statistics">
      {Array.from({ length: 4 }).map((_, i) => (
        <div key={i} className="stat-card skeleton" />
      ))}
    </div>
  );
}

// Nested loading boundaries for granular control
export default async function ProductPage({ params }: PageProps) {
  const { id } = await params;
  return (
    <div>
      <Suspense fallback={<HeaderSkeleton />}>
        <ProductHeader id={id} />
      </Suspense>
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews id={id} />
      </Suspense>
      <Suspense fallback={<RelatedSkeleton />}>
        <RelatedProducts id={id} />
      </Suspense>
    </div>
  );
}
// Each section streams independently — slow reviews don't block
// fast-loading related products from appearing
```

*Last Updated: 2026-07-30*
