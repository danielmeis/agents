# React Performance — Deep Reference

> Load this file for profiling, virtualization, bundle splitting, Compiler
> adoption, and deciding between performance tools.

---

## Measure First — React DevTools Profiler

Never optimize without measuring. The Profiler shows which components render,
how long they take, and why they rendered:

1. Install React DevTools browser extension
2. Open DevTools → **Profiler** tab → click **Record**
3. Interact with the slow part of the UI → click **Stop**
4. Inspect the flame graph — look for wide or tall bars
5. Click a component bar → check **"Why did this render?"**

React 19.2+ adds **Performance Tracks** in the browser's native Performance
panel (Chrome DevTools → Performance → React lanes visible in the timeline).

```tsx
// Wrap a subtree in Profiler to collect programmatic data
import { Profiler } from 'react';

function onRenderCallback(
  id: string,
  phase: 'mount' | 'update' | 'nested-update',
  actualDuration: number,
  baseDuration: number,
) {
  if (actualDuration > 16) { // over one frame at 60fps
    console.warn(`[Profiler] ${id} (${phase}): ${actualDuration.toFixed(1)}ms`);
  }
}

<Profiler id="ProductList" onRender={onRenderCallback}>
  <ProductList items={items} />
</Profiler>
```

---

## Re-render Causes and Fixes

A component re-renders when:
1. Its own state changes
2. Its parent re-renders (default behavior — not always a problem)
3. Context value it consumes changes
4. Its key changes

```tsx
// PROBLEM: Parent re-renders → all children re-render by default
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <ExpensiveChild />  {/* re-renders on every button click */}
    </>
  );
}

// FIX 1: React.memo — skip re-render when props haven't changed (shallow)
const ExpensiveChild = memo(function ExpensiveChild() {
  return <div>Expensive</div>;
});

// FIX 2: Children as props — the child doesn't re-render because it's
//         already rendered by the grandparent; just passed through
function Parent({ children }: { children: ReactNode }) {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      {children}  {/* won't re-render — it's owned by the grandparent */}
    </div>
  );
}
```

---

## React Compiler — Adoption Strategy

> React 19+ (also works on 17 and 18)

The React Compiler (stable October 2025) replaces most manual memoization.
On Vite 8, `@vitejs/plugin-react` v6 uses oxc instead of Babel — you need
`@rolldown/plugin-babel` to run the Babel-based compiler plugin:

```bash
# Step 1: install ESLint rules first (zero risk — just lint)
npm install --save-dev eslint-plugin-react-hooks@latest

# Update eslint config to use recommended-latest preset
# Fix any reported Rules of React violations before proceeding

# Step 2: install the compiler (still --save-exact to pin behavior)
npm install --save-dev --save-exact babel-plugin-react-compiler@latest
npm install --save-dev @rolldown/plugin-babel   # needed for Vite 8

# Step 3: configure
```

```ts
// vite.config.ts (Vite 8 + @vitejs/plugin-react v6)
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import babel from '@rolldown/plugin-babel';

export default defineConfig({
  plugins: [
    babel({
      plugins: ['babel-plugin-react-compiler'],
      exclude: /node_modules/,
      // Exclude Storybook stories — they often violate Rules of React
      // exclude: [/node_modules/, /\.stories\.(tsx|jsx)$/],
    }),
    react(),
  ],
  build: {
    sourcemap: true, // Essential — compiled output is unreadable without it
  },
});
```

```bash
# Next.js: built-in support
# next.config.ts
const nextConfig = { experimental: { reactCompiler: true } };
```

**Gradual rollout:**

```tsx
// Opt out a specific file/component while rolling out compiler:
export function LegacyComponent() {
  "use no memo";
  // Compiler skips this entire component
  // Use for: components with intentional mutations, test fixtures,
  //          third-party wrappers with reference-equality requirements
}
```

**After enabling the Compiler:**
- Remove `useMemo` and `useCallback` wrapping simple values — the compiler handles them
- Keep `useMemo`/`useCallback` only as explicit escape hatches where you need
  a specific memoization guarantee (see main SKILL.md for examples)
- Keep `React.memo` — the compiler doesn't remove it but it becomes less necessary

---

## Context Performance

Context re-renders every consumer when the value reference changes.
With large trees, split context by update frequency:

```tsx
// BAD: one context for everything — any state change re-renders all consumers
const AppContext = createContext({ user, theme, cart, setCart, setTheme });

// GOOD: split by how often each piece changes
const UserContext   = createContext<User | null>(null);      // changes rarely
const ThemeContext  = createContext<ThemeValue | null>(null); // changes sometimes
const CartContext   = createContext<CartValue | null>(null);  // changes often

// Each component only subscribes to what it needs
function CartIcon() {
  const { cart } = useCart();  // only re-renders when cart changes
  // Not affected by user or theme changes
}

// Memoize the context value to prevent unnecessary re-renders
function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const value = useMemo(() => ({ theme, setTheme }), [theme]); // ← essential
  return <ThemeContext value={value}>{children}</ThemeContext>;
}
```

---

## List Virtualization

For lists with hundreds or thousands of items — only render what's visible:

```tsx
// react-window: lightweight, well-maintained
import { FixedSizeList } from 'react-window';

const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
  <div style={style}>  {/* style prop from react-window is required */}
    <ProductCard product={products[index]} />
  </div>
);

function ProductList({ products }: { products: Product[] }) {
  return (
    <FixedSizeList
      height={600}          // visible height of the container
      itemCount={products.length}
      itemSize={80}         // height of each row in px
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}

// VariableSizeList for dynamic heights
// react-virtual (TanStack Virtual) for more complex cases (tables, grids)
```

---

## Code Splitting and Lazy Loading

```tsx
// Route-level splitting — most impactful
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings  = lazy(() => import('./pages/Settings'));
const Reports   = lazy(() => import('./pages/Reports'));

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings"  element={<Settings />} />
        <Route path="/reports"   element={<Reports />} />
      </Routes>
    </Suspense>
  );
}

// Component-level splitting for heavy components
const RichTextEditor = lazy(() =>
  import('./components/RichTextEditor').then(m => ({ default: m.RichTextEditor }))
);

// Preload on hover — improves perceived performance
function NavLink({ to, label, component: Component }: NavLinkProps) {
  const handleMouseEnter = () => {
    // Trigger the dynamic import on hover, not on click
    import(`./pages/${to}`);
  };
  return <a href={to} onMouseEnter={handleMouseEnter}>{label}</a>;
}
```

---

## Activity for Expensive Tabs

> React 19.2+

`<Activity>` preserves state and DOM while hiding content — better than
unmounting for tabs where remounting is expensive:

```tsx
import { Activity } from 'react';

function ReportsDashboard() {
  const [activeTab, setActiveTab] = useState<'sales' | 'traffic' | 'revenue'>('sales');

  return (
    <div>
      <nav>
        {(['sales', 'traffic', 'revenue'] as const).map(tab => (
          <button key={tab} onClick={() => setActiveTab(tab)}
            aria-selected={activeTab === tab}>
            {tab}
          </button>
        ))}
      </nav>

      {/* Each chart fetches its own data and is expensive to remount */}
      <Activity mode={activeTab === 'sales'   ? 'visible' : 'hidden'}>
        <SalesChart />      {/* state and fetch preserved when hidden */}
      </Activity>
      <Activity mode={activeTab === 'traffic' ? 'visible' : 'hidden'}>
        <TrafficChart />
      </Activity>
      <Activity mode={activeTab === 'revenue' ? 'visible' : 'hidden'}>
        <RevenueChart />
      </Activity>
    </div>
  );
}
// When hidden: effects unmount, updates defer, DOM is hidden (display: none)
// When re-shown: effects remount, but no data refetch, no scroll position loss
```

---

## useTransition vs useDeferredValue — When to Use Each

```tsx
// useTransition: you control the state update
function FilterableList() {
  const [filter, setFilter] = useState('');
  const [isPending, startTransition] = useTransition();

  return (
    <>
      <input
        onChange={e => {
          // Input update is urgent (user sees it immediately)
          // List filtering is non-urgent (can be interrupted)
          startTransition(() => setFilter(e.target.value));
        }}
      />
      {isPending && <Spinner />}
      <HeavyList filter={filter} />
    </>
  );
}

// useDeferredValue: you receive a prop you don't control
function HeavyList({ filter }: { filter: string }) {
  const deferredFilter = useDeferredValue(filter);
  const isStale = filter !== deferredFilter;

  // Render with the deferred value — React renders this in the background
  return (
    <ul style={{ opacity: isStale ? 0.6 : 1, transition: 'opacity 0.2s' }}>
      {expensiveFilter(items, deferredFilter).map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

---

## Common Performance Anti-Patterns

```tsx
// 1. Anonymous functions in JSX (new reference every render)
// BAD
<Button onClick={() => handleClick(item.id)} />  // new fn every render
// GOOD (or let the Compiler handle it)
const handleItemClick = useCallback(() => handleClick(item.id), [item.id, handleClick]);
<Button onClick={handleItemClick} />

// 2. Inline objects/arrays as props
// BAD
<Chart options={{ animation: true, theme: 'dark' }} />  // new object every render
// GOOD
const chartOptions = useMemo(() => ({ animation: true, theme: 'dark' }), []);
<Chart options={chartOptions} />

// 3. Using array index as key
// BAD — breaks when items are reordered, added, or removed
{items.map((item, i) => <Item key={i} {...item} />)}
// GOOD
{items.map(item => <Item key={item.id} {...item} />)}

// 4. State that should be derived
// BAD — double source of truth
const [filteredItems, setFilteredItems] = useState(items);
useEffect(() => setFilteredItems(items.filter(active)), [items, active]);
// GOOD
const filteredItems = useMemo(() => items.filter(i => i.active === active), [items, active]);

// 5. Unnecessary global state — causes app-wide re-renders
// Keep state as local as possible; only lift when truly shared
```
*Last Updated: 2026-07-29*
