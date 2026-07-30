---
name: react
description: >
  React 18.3.1 and React 19+ best practices, component patterns, hooks,
  performance, memoization, TypeScript prop typing, and testing with React
  Testing Library. Use this skill for any React question: useState, useEffect,
  useReducer, useCallback, useMemo, useRef, useContext, Suspense, error
  boundaries, concurrent features, forwardRef, composition patterns, custom
  hooks, form handling, and the React Compiler. Also trigger for React 19+
  APIs: useActionState, useOptimistic, useFormStatus, use(), Activity,
  useEffectEvent, ref-as-prop, and Context-as-provider shorthand.
  Next.js App Router, RSC, and server actions are out of scope — see the
  nextjs skill. Deep TypeScript patterns belong in the typescript skill.
  Version legend used throughout this skill:
  (baseline) = React 18.3.1 and React 19+
  > React 19+ = requires React 19.0 or higher
  > React 19.2+ = requires React 19.2 or higher
---

# React Best Practices (18.3.1 baseline · 19+ callouts)

> Current versions: React **19.2.7** (latest, June 2026) · React **18.3.1**
> (LTS). React Compiler **1.0** stable (October 2025).
> Examples default to TypeScript. Plain JS differences are noted inline.
> Anything **not marked** works on both 18.3.1 and 19+.

---

## Version Feature Map

| Feature | 18.3.1 | 19+ |
|---------|--------|-----|
| Automatic batching, `createRoot`, `hydrateRoot` | ✅ | ✅ |
| `useTransition`, `useDeferredValue`, `useId` | ✅ | ✅ |
| `Suspense`, `React.lazy`, error boundaries | ✅ | ✅ |
| `forwardRef` for passing refs | ✅ required | ✅ deprecated → ref is a prop |
| `<Context.Provider>` wrapper | ✅ required | ✅ still works |
| `<Context>` as provider (no `.Provider`) | ❌ | ✅ |
| `useActionState` | ❌ | ✅ |
| `useOptimistic` | ❌ | ✅ |
| `useFormStatus` | ❌ | ✅ |
| `use()` hook | ❌ | ✅ |
| ref callback cleanup return | ❌ | ✅ |
| `<Activity>` component | ❌ | ✅ 19.2+ |
| `useEffectEvent` | ❌ | ✅ 19.2+ |
| React Compiler 1.0 | ✅ works on 17+ | ✅ optimized for 19 |

---

## Core Principles

- One component per file; filename matches component name (`UserCard.tsx`)
- Components are pure functions — same props always produce the same output
- Never mutate props or state directly — always return new values
- Derive values rather than sync them — if you can compute it, don't store it
- Lift state to the lowest common ancestor, no higher
- Keep components small; extract complex logic into named custom hooks
- TypeScript: always type props explicitly; avoid `any`; use `unknown` for external data
- Plain JS: use PropTypes in development for runtime checks

---

## Component Fundamentals

```tsx
// TypeScript: explicit props interface is required — no implicit any
interface UserCardProps {
  userId: number;
  displayName: string;
  avatarUrl?: string;            // optional
  onSelect: (id: number) => void;
}

// Defaults via destructuring — React 19 removed defaultProps for function components
export function UserCard({
  userId,
  displayName,
  avatarUrl = '/default-avatar.png',
  onSelect,
}: UserCardProps) {
  return (
    <article onClick={() => onSelect(userId)}>
      <img src={avatarUrl} alt={`${displayName}'s avatar`} />
      <h2>{displayName}</h2>
    </article>
  );
}
```

```jsx
// Plain JS: use PropTypes for runtime type checking
import PropTypes from 'prop-types';

export function UserCard({ userId, displayName, avatarUrl = '/default-avatar.png', onSelect }) {
  return ( /* same JSX */ );
}

UserCard.propTypes = {
  userId:      PropTypes.number.isRequired,
  displayName: PropTypes.string.isRequired,
  avatarUrl:   PropTypes.string,
  onSelect:    PropTypes.func.isRequired,
};
```

---

## useState

```tsx
// Initializer function for expensive initial state — runs once, not every render
const [items, setItems] = useState<Item[]>(() => parseStoredItems());

// Functional update — always use when next state depends on previous
// This is safe in concurrent mode and avoids stale closures
setCount(prev => prev + 1);

// Object state: spread; never mutate
const [form, setForm] = useState({ name: '', email: '' });
setForm(prev => ({ ...prev, email: 'new@example.com' }));

// BAD: mutating state — React won't detect the change
form.email = 'new@example.com'; setForm(form); // ❌
```

---

## useEffect

```tsx
// Data fetching with abort cleanup — the standard pattern
useEffect(() => {
  const controller = new AbortController();

  fetchUser(userId, { signal: controller.signal })
    .then(setUser)
    .catch(err => {
      if (err.name !== 'AbortError') setError(err);
    });

  return () => controller.abort(); // runs before next effect or on unmount
}, [userId]);

// Empty array: run once after mount (runs twice in StrictMode — write cleanup)
useEffect(() => {
  const id = analytics.track('page_view');
  return () => analytics.untrack(id);
}, []);
```

### The Most Common useEffect Mistakes

```tsx
// 1. Missing dependency → stale closure
useEffect(() => {
  document.title = `Hello, ${name}`; // stale if name changes
}, []); // ❌ name missing from deps

// 2. Inline object/array in deps → infinite loop
useEffect(() => {
  doSomething(options);
}, [options]); // ❌ new reference every render if options is created inline

// GOOD: stabilize with useMemo or move outside the component
const stableOptions = useMemo(() => ({ limit: 10, page }), [page]);
useEffect(() => { doSomething(stableOptions); }, [stableOptions]);

// 3. Derived state via useEffect → use useMemo instead
// BAD
const [fullName, setFullName] = useState('');
useEffect(() => {
  setFullName(`${first} ${last}`); // ❌ extra render, needless effect
}, [first, last]);

// GOOD
const fullName = `${first} ${last}`; // just derive it
```

---

## useReducer

Prefer `useReducer` over multiple `useState` when updates are related or when
the next state depends on multiple pieces of current state:

```tsx
type State = { count: number; status: 'idle' | 'loading' | 'error'; error: string | null };
type Action =
  | { type: 'increment' | 'decrement' | 'reset' }
  | { type: 'set_error'; payload: string };

const initialState: State = { count: 0, status: 'idle', error: null };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment': return { ...state, count: state.count + 1 };
    case 'decrement': return { ...state, count: state.count - 1 };
    case 'set_error': return { ...state, status: 'error', error: action.payload };
    case 'reset':     return initialState;
    default:          return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <span>{state.count}</span>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
    </>
  );
}
```

---

## useRef

```tsx
// DOM reference
const inputRef = useRef<HTMLInputElement>(null);
const handleFocus = () => inputRef.current?.focus();
<input ref={inputRef} />

// React 18: forwardRef required to pass a ref to a child component
const FancyInput = forwardRef<HTMLInputElement, { label: string }>(
  ({ label }, ref) => <label>{label}<input ref={ref} /></label>
);

// > React 19+: ref is just a prop — forwardRef is no longer needed
function FancyInput({ label, ref }: { label: string; ref?: React.Ref<HTMLInputElement> }) {
  return <label>{label}<input ref={ref} /></label>;
}
// forwardRef still works in 19 but is deprecated — migrate when convenient

// Mutable value that persists without triggering re-render
const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);
useEffect(() => {
  intervalRef.current = setInterval(tick, 1000);
  return () => { if (intervalRef.current) clearInterval(intervalRef.current); };
}, []);

// > React 19+: ref callback can return a cleanup function
<div ref={node => {
  if (!node) return;
  const observer = new ResizeObserver(onResize);
  observer.observe(node);
  return () => observer.disconnect(); // cleanup when element is removed
}} />
```

---

## Context

```tsx
interface ThemeContextValue {
  theme: 'light' | 'dark';
  setTheme: (t: 'light' | 'dark') => void;
}

const ThemeContext = createContext<ThemeContextValue | null>(null);

// Always export a typed hook — never call useContext directly in consumers
export function useTheme(): ThemeContextValue {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used inside ThemeProvider');
  return ctx;
}

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const value = useMemo(() => ({ theme, setTheme }), [theme]);

  // React 18: <ThemeContext.Provider value={value}>
  // > React 19+: Context directly as a component — no .Provider
  return <ThemeContext value={value}>{children}</ThemeContext>;
}
```

---

## Memoization

### React Compiler (React 19+, also works on 18.x and 17)

> React 19+ (Compiler)

The React Compiler (stable 1.0, October 2025) automatically inserts memoization
at build time. When it's active, manual `useMemo`, `useCallback`, and
`React.memo` are largely unnecessary — the compiler handles them more accurately
than humans do.

**Setup on Vite 8 (2026 — `@vitejs/plugin-react` v6 dropped Babel for oxc):**

```bash
npm install --save-dev --save-exact babel-plugin-react-compiler@latest
npm install --save-dev @rolldown/plugin-babel
npm install --save-dev eslint-plugin-react-hooks@latest
```

```js
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';          // v6+ uses oxc internally
import babel from '@rolldown/plugin-babel';         // needed for Compiler since oxc replaced Babel in v6

export default defineConfig({
  plugins: [
    babel({
      plugins: ['babel-plugin-react-compiler'],
      exclude: /node_modules/,
    }),
    react(),
  ],
  build: { sourcemap: true }, // essential — compiler output is unreadable without it
});
```

**ESLint (enable lint rules before enabling compilation):**

```json
// .eslintrc or eslint.config.js
{
  "extends": ["plugin:react-hooks/recommended-latest"]
  // recommended-latest includes compiler-specific rules
}
```

**Gradual adoption — run ESLint first, fix violations, then enable:**

```js
// "use no memo" — opt a specific component out of compilation
// Use for fixtures that intentionally mutate, or when you need precise control
export function MyComponent() {
  "use no memo";
  // compiler skips this component entirely
}
```

**When to still write useMemo/useCallback manually (even with Compiler):**

```tsx
// 1. When a value feeds into a useEffect dependency array and you need
//    a guaranteed stable reference the compiler can't infer
const stableConfig = useMemo(() => ({ endpoint, timeout }), [endpoint, timeout]);
useEffect(() => { connect(stableConfig); }, [stableConfig]);

// 2. When a third-party library uses reference equality for optimization
//    and the compiler's output isn't producing a stable reference
const columns = useMemo<ColumnDef[]>(() => [
  { key: 'name', header: 'Name' },
], []); // some table libraries break without this

// Think of useMemo/useCallback as escape hatches for precise control,
// not default tools — the Compiler handles the rest
```

### Manual Memoization (React 18, or when Compiler is not enabled)

```tsx
// React.memo — skip re-render when props haven't changed (shallow comparison)
const UserCard = memo(function UserCard({ user, onSelect }: UserCardProps) {
  return <div onClick={() => onSelect(user.id)}>{user.name}</div>;
});

// useMemo — memoize expensive computed values
const sortedItems = useMemo(
  () => [...items].sort((a, b) => a.name.localeCompare(b.name)),
  [items] // only re-sort when items changes
);

// useCallback — stable function reference for memoized children or effect deps
const handleSelect = useCallback((id: number) => {
  setSelectedId(id);
  onSelect?.(id);
}, [onSelect]); // only recreate when onSelect changes
```

---

## Suspense and Lazy Loading

```tsx
// Code splitting with React.lazy — baseline in both versions
const HeavyChart = lazy(() => import('./HeavyChart'));
const AdminPanel = lazy(() => import('./AdminPanel'));

function Dashboard() {
  return (
    <Suspense fallback={<Skeleton />}>
      <HeavyChart data={data} />
    </Suspense>
  );
}

// Nested Suspense boundaries — granular loading states
function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Header />
      <Suspense fallback={<ContentSkeleton />}>
        <MainContent />
      </Suspense>
      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>
    </Suspense>
  );
}
```

---

## Error Boundaries

Class components are still required for error boundaries (no hook equivalent yet):

```tsx
interface ErrorBoundaryState { hasError: boolean; error: Error | null }

class ErrorBoundary extends Component<
  { children: ReactNode; fallback: ReactNode },
  ErrorBoundaryState
> {
  constructor(props: { children: ReactNode; fallback: ReactNode }) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    logError(error, info.componentStack);
  }

  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}

// Usage — wrap async sections
<ErrorBoundary fallback={<ErrorMessage />}>
  <Suspense fallback={<Loading />}>
    <AsyncComponent />
  </Suspense>
</ErrorBoundary>
```

---

## Concurrent Features (Baseline — both versions)

```tsx
// useTransition — mark state updates as non-urgent
// Keeps the UI responsive while expensive renders happen in the background
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Result[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (value: string) => {
    setQuery(value);                       // urgent — update input immediately
    startTransition(() => {
      setResults(heavyFilter(value));      // non-urgent — can be interrupted
    });
  };

  return (
    <>
      <input value={query} onChange={e => handleSearch(e.target.value)} />
      {isPending ? <Spinner /> : <ResultsList items={results} />}
    </>
  );
}

// useDeferredValue — defer re-rendering an expensive child
function ProductList({ filter }: { filter: string }) {
  const deferredFilter = useDeferredValue(filter);
  const isStale = filter !== deferredFilter;

  return (
    <div style={{ opacity: isStale ? 0.6 : 1 }}>
      <ExpensiveList filter={deferredFilter} />
    </div>
  );
}
```

---

## React 19+ APIs

> React 19+

### useActionState — async actions with pending and error state

Replaces the old `useFormState` (deprecated in 19):

```tsx
import { useActionState } from 'react';

type FormState = { error: string | null; success: boolean };

async function updateProfile(_prev: FormState, formData: FormData): Promise<FormState> {
  const name = formData.get('name') as string;
  if (!name.trim()) return { error: 'Name is required', success: false };
  await api.updateProfile({ name });
  return { error: null, success: true };
}

function ProfileForm() {
  const [state, action, isPending] = useActionState(updateProfile, {
    error: null,
    success: false,
  });

  return (
    <form action={action}>
      <input name="name" />
      {state.error && <p role="alert">{state.error}</p>}
      {state.success && <p>Saved!</p>}
      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving…' : 'Save'}
      </button>
    </form>
  );
}
```

### useFormStatus — access parent form state from a child

```tsx
import { useFormStatus } from 'react-dom';

// Must be a child of the form element — not in the form component itself
function SubmitButton({ label }: { label: string }) {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Loading…' : label}
    </button>
  );
}

function MyForm() {
  return (
    <form action={submitAction}>
      <input name="email" type="email" />
      <SubmitButton label="Subscribe" />  {/* useFormStatus works here */}
    </form>
  );
}
```

### useOptimistic — instant UI with async rollback

```tsx
import { useOptimistic } from 'react';

function MessageList({ messages }: { messages: Message[] }) {
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    (current, newMessage: Message) => [...current, newMessage]
  );

  async function sendMessage(formData: FormData) {
    const text = formData.get('text') as string;
    const tempMessage = { id: crypto.randomUUID(), text, sending: true };

    addOptimistic(tempMessage);          // immediately show in UI
    await api.sendMessage(text);         // real request in background
    // on success: messages prop updates and replaces optimistic state
    // on error: wrap in try/catch and handle rollback via state
  }

  return (
    <form action={sendMessage}>
      <ul>
        {optimisticMessages.map(m => (
          <li key={m.id} style={{ opacity: m.sending ? 0.6 : 1 }}>{m.text}</li>
        ))}
      </ul>
      <input name="text" /><button type="submit">Send</button>
    </form>
  );
}
```

### use() — read promises and context conditionally

```tsx
import { use, Suspense } from 'react';

// IMPORTANT: the promise must be created outside render and cached
// If created inside render, Suspense fallback never clears
const commentsPromise = fetchComments(postId); // outside component or in a stable ref

function Comments({ promise }: { promise: Promise<Comment[]> }) {
  // Can be called conditionally — the only hook that can be
  const comments = use(promise); // suspends until resolved
  return <ul>{comments.map(c => <li key={c.id}>{c.text}</li>)}</ul>;
}

function PostPage() {
  return (
    <Suspense fallback={<CommentsSkeleton />}>
      <Comments promise={commentsPromise} />
    </Suspense>
  );
}

// use() also reads context — useful when inside a conditional
function ConditionalFeature({ show }: { show: boolean }) {
  if (!show) return null;
  const theme = use(ThemeContext); // valid — use() can be conditional
  return <div className={theme}>...</div>;
}
```

### Activity — preserve state while hiding UI

> React 19.2+

```tsx
import { Activity } from 'react';

// Replaces conditional rendering when you want to preserve component state
// 'hidden': hides children, unmounts effects, defers updates
// 'visible': shows children, mounts effects, processes updates normally

function TabPanel({ tabs }: { tabs: Tab[] }) {
  const [activeTab, setActiveTab] = useState(0);

  return (
    <>
      <nav>{tabs.map((t, i) => <button key={t.id} onClick={() => setActiveTab(i)}>{t.label}</button>)}</nav>
      {tabs.map((tab, i) => (
        <Activity key={tab.id} mode={i === activeTab ? 'visible' : 'hidden'}>
          <tab.Component />
          {/* State is preserved when hidden — no remount on tab switch */}
        </Activity>
      ))}
    </>
  );
}
```

### useEffectEvent — stable event callbacks in effects

> React 19.2+

Solves the stale closure problem without adding to the dependency array:

```tsx
import { useEffectEvent } from 'react';

function ChatRoom({ roomId, userId }: { roomId: string; userId: string }) {
  const onMessage = useEffectEvent((message: string) => {
    // Always sees latest userId without being a dep of the effect
    logMessage(userId, message);
  });

  useEffect(() => {
    const socket = connectToRoom(roomId);
    socket.on('message', onMessage); // stable reference, never stale
    return () => socket.disconnect();
  }, [roomId]); // roomId only — onMessage is NOT a dependency
}

// Rule: only use useEffectEvent for callbacks that are conceptually "events"
// Don't use it just to silence exhaustive-deps lint warnings
```

---

## Custom Hooks

```tsx
// Pattern: one concern per hook, always prefixed with 'use'

function useLocalStorage<T>(key: string, initialValue: T) {
  const [stored, setStored] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = useCallback((value: T | ((prev: T) => T)) => {
    try {
      const valueToStore = value instanceof Function ? value(stored) : value;
      setStored(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error('useLocalStorage write error:', error);
    }
  }, [key, stored]);

  return [stored, setValue] as const;
}

// Async data fetching hook
function useFetch<T>(url: string) {
  const [state, setState] = useState<{
    data: T | null;
    loading: boolean;
    error: Error | null;
  }>({ data: null, loading: true, error: null });

  useEffect(() => {
    const controller = new AbortController();
    setState(prev => ({ ...prev, loading: true, error: null }));

    fetch(url, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json() as Promise<T>;
      })
      .then(data => setState({ data, loading: false, error: null }))
      .catch(err => {
        if (err.name !== 'AbortError') {
          setState({ data: null, loading: false, error: err });
        }
      });

    return () => controller.abort();
  }, [url]);

  return state;
}
```

---

## TypeScript Patterns for React

```tsx
// Children types
interface WrapperProps {
  children: ReactNode;        // any renderable content (most common)
  // children: ReactElement;  // must be a React element specifically
  // children: ReactNode[];   // must be an array
}

// Event handler types
interface FormProps {
  onChange: (e: React.ChangeEvent<HTMLInputElement>) => void;
  onSubmit: (e: React.FormEvent<HTMLFormElement>) => void;
  onClick:  (e: React.MouseEvent<HTMLButtonElement>) => void;
  onKeyDown:(e: React.KeyboardEvent<HTMLInputElement>) => void;
}

// Generic components
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => ReactNode;
  keyExtractor: (item: T) => string | number;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return <ul>{items.map((item, i) => <li key={keyExtractor(item)}>{renderItem(item, i)}</li>)}</ul>;
}

// Component ref type
const ref = useRef<HTMLDivElement>(null);

// useState with explicit type when inference fails
const [user, setUser] = useState<User | null>(null);

// Discriminated union for component variants
type ButtonProps =
  | { variant: 'primary'; onClick: () => void; href?: never }
  | { variant: 'link';    href: string;        onClick?: never };

// Plain JS: skip all of the above — just write the component
```

---

## Performance Checklist

> **Read `references/performance.md`** for profiling, virtualization, bundle
> splitting, and image/asset optimization patterns.

Quick rules:

- Avoid creating objects/arrays/functions inside JSX — they're new references every render
- Keep state as local as possible — global state causes wide re-renders
- Split context when different parts of the tree need different update rates
- Use `key` correctly — never use array index as key for dynamic lists
- Measure before optimizing — React DevTools Profiler first, then fix

```tsx
// BAD: new function reference every render → child re-renders every time
<Button onClick={() => handleClick(item.id)} />

// GOOD: stable reference with useCallback (or let the Compiler handle it)
const handleItemClick = useCallback(() => handleClick(item.id), [item.id]);
<Button onClick={handleItemClick} />

// BAD: array index as key — breaks reconciliation on reorder/insert/delete
{items.map((item, index) => <Item key={index} {...item} />)}

// GOOD: stable unique ID
{items.map(item => <Item key={item.id} {...item} />)}
```

---

## Reference Files

Load these when the task goes deeper than the summaries above:

- **`references/hooks.md`** — all hooks in depth, custom hook patterns, the rules
  of hooks, `useImperativeHandle`, `useLayoutEffect`, `useInsertionEffect`,
  `useDebugValue`, and common pitfall patterns
- **`references/performance.md`** — React DevTools Profiler, virtualization with
  `react-window`, bundle splitting, Compiler adoption strategy, `<Activity>`
  for expensive tabs, `useDeferredValue` vs `useTransition` decision guide
- **`references/patterns.md`** — composition, compound components, render props,
  controlled vs uncontrolled, provider patterns, higher-order components
- **`references/testing.md`** — React Testing Library setup and queries,
  async testing, mocking hooks and modules, accessibility-first test patterns,
  Vitest vs Jest

> Next.js App Router, RSC, and server actions: see the **nextjs** skill.
> Deep TypeScript (generics, utility types, declaration merging): see the **typescript** skill.

*Last Updated: 2026-07-29*
