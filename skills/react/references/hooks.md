# React Hooks — Deep Reference

> Load this file for advanced hook patterns, all lesser-used hooks,
> custom hook architecture, and common pitfall debugging.
> Examples use TypeScript; every pattern applies identically in plain JS —
> drop the interfaces, generics, and `: Type`/`as Type` annotations, the
> hook logic itself is unchanged.

---

## Rules of Hooks (Enforced by ESLint)

1. Only call hooks at the top level — never inside loops, conditions, or nested functions
2. Only call hooks from React function components or custom hooks
3. `use()` (React 19+) is the sole exception — it can be called conditionally
4. Custom hooks must start with `use`

```tsx
// eslint-plugin-react-hooks enforces these at lint time
// react-hooks/recommended-latest preset includes React Compiler rules (19.2+)
```

---

## useMemo

```tsx
// When to use: expensive computation that shouldn't re-run on every render
const filteredAndSorted = useMemo(() => {
  return items
    .filter(item => item.status === filter)
    .sort((a, b) => a.name.localeCompare(b.name));
}, [items, filter]);

// When NOT to use: simple operations — the memo overhead costs more than it saves
const doubled = items.map(x => x * 2); // fine without useMemo
const fullName = `${first} ${last}`;   // fine without useMemo

// Stable object reference for useEffect or third-party library equality checks
const config = useMemo(() => ({
  endpoint: apiUrl,
  timeout: 5000,
  retries: 3,
}), [apiUrl]);
```

---

## useCallback

```tsx
// When to use: stable function reference needed by a memoized child or effect dep
const handleDelete = useCallback(async (id: string) => {
  await api.delete(id);
  setItems(prev => prev.filter(item => item.id !== id));
}, []); // no deps — setItems is stable; api is a module-level import

// With React Compiler enabled: useCallback is largely redundant
// Use it as an escape hatch when you need a guaranteed stable reference

// Common mistake: unnecessary useCallback on every function
// BAD — this creates a new function anyway (the useCallback wrapper itself)
const handleClick = useCallback(() => doSomething(), []); // wasteful if child isn't memoized
```

---

## useLayoutEffect

Fires synchronously after DOM mutations but before the browser paints.
Use it when you need to read DOM measurements before the user sees the update:

```tsx
// Use case: measure DOM element and set state before paint to avoid flicker
function Tooltip({ content, targetRef }: { content: string; targetRef: RefObject<HTMLElement> }) {
  const [position, setPosition] = useState({ top: 0, left: 0 });
  const tooltipRef = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    if (!targetRef.current || !tooltipRef.current) return;
    const targetRect = targetRef.current.getBoundingClientRect();
    const tooltipRect = tooltipRef.current.getBoundingClientRect();
    setPosition({
      top: targetRect.bottom + window.scrollY,
      left: targetRect.left + (targetRect.width - tooltipRect.width) / 2,
    });
  }, [targetRef, content]);

  return (
    <div ref={tooltipRef} style={{ position: 'absolute', ...position }}>
      {content}
    </div>
  );
}

// Rule: default to useEffect; only reach for useLayoutEffect when you see flicker
// useLayoutEffect runs on the server too in Next.js — causes a warning;
// use useEffect or guard with typeof window !== 'undefined' in SSR contexts
```

---

## useInsertionEffect

Only for CSS-in-JS libraries that need to inject styles before layout reads.
Do not use in application code — this is a library author API:

```tsx
// Library use only — fires before useLayoutEffect and useEffect
// Used by styled-components, Emotion, etc. to inject <style> tags
useInsertionEffect(() => {
  const style = document.createElement('style');
  style.textContent = `.my-class { color: red; }`;
  document.head.appendChild(style);
  return () => document.head.removeChild(style);
}, []);
```

---

## useImperativeHandle

Expose a controlled imperative API from a component rather than the raw DOM node:

```tsx
interface VideoPlayerHandle {
  play: () => void;
  pause: () => void;
  seek: (time: number) => void;
}

// React 18: must use forwardRef
const VideoPlayer = forwardRef<VideoPlayerHandle, { src: string }>(
  function VideoPlayer({ src }, ref) {
    const videoRef = useRef<HTMLVideoElement>(null);

    useImperativeHandle(ref, () => ({
      play:  () => videoRef.current?.play(),
      pause: () => videoRef.current?.pause(),
      seek:  (t) => { if (videoRef.current) videoRef.current.currentTime = t; },
    }), []); // deps: re-create handle only when deps change

    return <video ref={videoRef} src={src} />;
  }
);

// > React 19+: ref is just a prop
function VideoPlayer({ src, ref }: { src: string; ref?: React.Ref<VideoPlayerHandle> }) {
  const videoRef = useRef<HTMLVideoElement>(null);
  useImperativeHandle(ref, () => ({
    play:  () => videoRef.current?.play(),
    pause: () => videoRef.current?.pause(),
    seek:  (t) => { if (videoRef.current) videoRef.current.currentTime = t; },
  }), []);
  return <video ref={videoRef} src={src} />;
}

// Usage
const playerRef = useRef<VideoPlayerHandle>(null);
playerRef.current?.play();
```

---

## useId

Generate stable unique IDs for accessibility — safe in SSR (no hydration mismatch):

```tsx
// Never use Math.random() or a counter for IDs in React — causes hydration errors
function FormField({ label }: { label: string }) {
  const id = useId();  // produces ':r0:', ':r1:', etc. — stable across SSR/client

  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </div>
  );
}

// Multiple related IDs from one useId call
function RadioGroup({ options }: { options: string[] }) {
  const groupId = useId();
  return (
    <fieldset>
      {options.map((opt, i) => (
        <label key={opt}>
          <input type="radio" id={`${groupId}-${i}`} name={groupId} value={opt} />
          {opt}
        </label>
      ))}
    </fieldset>
  );
}
```

---

## useDebugValue

Label custom hooks in React DevTools — only useful in hooks shared across a team:

```tsx
function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useDebugValue(isOnline ? 'Online' : 'Offline');
  // DevTools now shows: OnlineStatus: "Online"

  // For expensive formatting, pass a formatter function (only called by DevTools)
  useDebugValue(lastUpdated, date => date.toISOString());

  useEffect(() => {
    const on  = () => setIsOnline(true);
    const off = () => setIsOnline(false);
    window.addEventListener('online', on);
    window.addEventListener('offline', off);
    return () => {
      window.removeEventListener('online', on);
      window.removeEventListener('offline', off);
    };
  }, []);

  return isOnline;
}
```

---

## useSyncExternalStore

The correct way to subscribe to external (non-React) stores — used internally
by state management libraries and browser API wrappers:

```tsx
function useWindowSize() {
  return useSyncExternalStore(
    // subscribe: called with a callback React fires when it wants a re-check
    (callback) => {
      window.addEventListener('resize', callback);
      return () => window.removeEventListener('resize', callback);
    },
    // getSnapshot: must return the same value if nothing has changed
    () => `${window.innerWidth}x${window.innerHeight}`,
    // getServerSnapshot: optional — what to return during SSR
    () => '0x0'
  );
}

// Use it for any external mutable store (Redux, Zustand, browser APIs)
// Never useEffect + useState for external store subscriptions — use this
```

---

## useTransition vs useDeferredValue — Decision Guide

| Scenario | Use |
|----------|-----|
| You control the state update that's slow | `useTransition` |
| You receive a value as a prop and can't control when it changes | `useDeferredValue` |
| Showing a spinner while a transition is in progress | `useTransition` (`isPending`) |
| Showing stale UI while new content renders | `useDeferredValue` (compare `value !== deferredValue`) |
| Form input + expensive filtered list | Either works; `useDeferredValue` is simpler |
| Navigation between routes | `useTransition` (Next.js does this for you) |

---

## Common Pitfalls Reference

### Stale Closure

```tsx
// BUG: count is captured at 0 and never updates in the interval
useEffect(() => {
  const id = setInterval(() => {
    console.log(count); // always 0
    setCount(count + 1); // always sets to 1
  }, 1000);
  return () => clearInterval(id);
}, []); // ❌ count not in deps

// FIX 1: functional update — no closure needed
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(id);
}, []);

// FIX 2: useEffectEvent (React 19.2+) for reading without subscribing
const onTick = useEffectEvent(() => {
  console.log(count); // always fresh
  setCount(count + 1);
});
useEffect(() => {
  const id = setInterval(onTick, 1000);
  return () => clearInterval(id);
}, []);
```

### Infinite useEffect Loop

```tsx
// BUG: setData triggers re-render → effect runs → setData → infinite loop
useEffect(() => {
  setData(process(data)); // ❌ data is a dep, setData changes it → loop
}, [data]);

// FIX: separate the derived value — don't put derived state in useEffect
const processedData = useMemo(() => process(data), [data]);
```

### Object/Function Deps

```tsx
// BUG: options is a new object every render → effect runs every render
function Component({ userId }: { userId: string }) {
  const options = { include: ['name', 'email'] }; // new ref each render
  useEffect(() => {
    fetchUser(userId, options); // ❌
  }, [userId, options]);

  // FIX 1: move outside component if it doesn't depend on props/state
  // FIX 2: useMemo if it does depend on props/state
  const stableOptions = useMemo(
    () => ({ include: ['name', 'email'] }),
    [] // empty deps: never changes
  );
}
```

### Missing Cleanup on Async Effects

```tsx
// BUG: component unmounts before fetch completes → setState on unmounted component
useEffect(() => {
  fetchData().then(data => setData(data)); // ❌ may run after unmount
}, []);

// FIX: always use AbortController
useEffect(() => {
  const controller = new AbortController();
  fetchData({ signal: controller.signal })
    .then(setData)
    .catch(err => { if (err.name !== 'AbortError') setError(err); });
  return () => controller.abort();
}, []);
```

---

## Custom Hook Patterns

### Composition: hooks that use other hooks

```tsx
// Build complex hooks from simpler ones
function useUserProfile(userId: string) {
  const { data: user, loading, error } = useFetch<User>(`/api/users/${userId}`);
  const isAdmin = useMemo(() => user?.role === 'admin', [user]);
  const displayName = user ? `${user.firstName} ${user.lastName}` : '';

  return { user, loading, error, isAdmin, displayName };
}
```

### Returning Stable Tuples

```tsx
// Return [value, setter] as const for proper TypeScript tuple inference
function useToggle(initial = false): [boolean, () => void] {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle];
}

// Or return an object for named access (clearer when returning many values)
function usePagination(total: number, perPage = 20) {
  const [page, setPage] = useState(1);
  const totalPages = Math.ceil(total / perPage);
  const hasPrev = page > 1;
  const hasNext = page < totalPages;

  return {
    page,
    totalPages,
    hasPrev,
    hasNext,
    goTo: setPage,
    prev: () => setPage(p => Math.max(1, p - 1)),
    next: () => setPage(p => Math.min(totalPages, p + 1)),
  };
}
```
*Last Updated: 2026-08-03*
