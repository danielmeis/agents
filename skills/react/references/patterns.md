# React Component Patterns — Deep Reference

> Load this file for composition, compound components, controlled vs
> uncontrolled, provider patterns, render props, and HOC guidance.
> Examples use TypeScript; every pattern applies identically in plain JS —
> drop the interfaces, generics, and `: Type`/`as Type` annotations, the
> component logic itself is unchanged.

---

## Composition Over Configuration

The most important React pattern. Prefer passing `children` and specific
slot props over a single large `config` object:

```tsx
// BAD: configuration-driven — hard to extend, couples parent and child
<Modal
  title="Confirm"
  body="Are you sure?"
  primaryButtonLabel="Yes"
  secondaryButtonLabel="Cancel"
  onPrimary={handleConfirm}
  onSecondary={handleClose}
  showCloseIcon={true}
  headerVariant="danger"
/>

// GOOD: composition — consumers decide the structure
<Modal>
  <Modal.Header variant="danger">Confirm</Modal.Header>
  <Modal.Body>Are you sure?</Modal.Body>
  <Modal.Footer>
    <Button variant="secondary" onClick={handleClose}>Cancel</Button>
    <Button variant="primary" onClick={handleConfirm}>Yes</Button>
  </Modal.Footer>
</Modal>
```

---

## Compound Components

Components that work together and share implicit state through context:

```tsx
interface AccordionContextValue {
  openId: string | null;
  toggle: (id: string) => void;
}

const AccordionContext = createContext<AccordionContextValue | null>(null);

function useAccordion() {
  const ctx = useContext(AccordionContext);
  if (!ctx) throw new Error('Must be used inside Accordion');
  return ctx;
}

// Parent manages state; children consume it via context
function Accordion({ children, defaultOpen }: { children: ReactNode; defaultOpen?: string }) {
  const [openId, setOpenId] = useState<string | null>(defaultOpen ?? null);
  const toggle = useCallback((id: string) => {
    setOpenId(prev => prev === id ? null : id);
  }, []);
  const value = useMemo(() => ({ openId, toggle }), [openId, toggle]);

  return (
    <AccordionContext value={value}>
      <div>{children}</div>
    </AccordionContext>
  );
}

function AccordionItem({ id, title, children }: { id: string; title: string; children: ReactNode }) {
  const { openId, toggle } = useAccordion();
  const isOpen = openId === id;

  return (
    <div>
      <button
        onClick={() => toggle(id)}
        aria-expanded={isOpen}
        aria-controls={`accordion-panel-${id}`}
      >
        {title}
      </button>
      <div
        id={`accordion-panel-${id}`}
        role="region"
        hidden={!isOpen}
      >
        {children}
      </div>
    </div>
  );
}

// Attach sub-components as properties (namespace pattern)
Accordion.Item = AccordionItem;

// Usage — clean and explicit
<Accordion defaultOpen="faq-1">
  <Accordion.Item id="faq-1" title="What is this?">Answer 1</Accordion.Item>
  <Accordion.Item id="faq-2" title="How does it work?">Answer 2</Accordion.Item>
</Accordion>
```

---

## Controlled vs Uncontrolled Components

```tsx
// Controlled: React owns the state — required when you need to react to changes
function ControlledInput({
  value,
  onChange,
}: {
  value: string;
  onChange: (value: string) => void;
}) {
  return (
    <input
      value={value}
      onChange={e => onChange(e.target.value)}
    />
  );
}

// Uncontrolled: DOM owns the state — simpler for one-time reads (form submit)
function UncontrolledForm({ onSubmit }: { onSubmit: (name: string) => void }) {
  const nameRef = useRef<HTMLInputElement>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (nameRef.current) onSubmit(nameRef.current.value);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={nameRef} defaultValue="" />
      <button type="submit">Submit</button>
    </form>
  );
}

// > React 19+: form Actions make uncontrolled patterns natural for server actions
function ActionForm({ action }: { action: (data: FormData) => Promise<void> }) {
  return (
    <form action={action}>
      <input name="name" defaultValue="" />
      <button type="submit">Submit</button>
    </form>
  );
}

// Flexible: support both controlled and uncontrolled (like HTML inputs)
function FlexibleInput({
  value,
  defaultValue,
  onChange,
}: {
  value?: string;
  defaultValue?: string;
  onChange?: (value: string) => void;
}) {
  const isControlled = value !== undefined;
  const [internalValue, setInternalValue] = useState(defaultValue ?? '');
  const currentValue = isControlled ? value : internalValue;

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (!isControlled) setInternalValue(e.target.value);
    onChange?.(e.target.value);
  };

  return <input value={currentValue} onChange={handleChange} />;
}
```

---

## Render Props

Useful for sharing stateful logic when a custom hook isn't sufficient
(e.g., when the consumer needs to control markup structure):

```tsx
// renderProp pattern: pass a function as a prop
interface MouseTrackerProps {
  render: (position: { x: number; y: number }) => ReactNode;
}

function MouseTracker({ render }: MouseTrackerProps) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  return (
    <div
      style={{ width: '100%', height: 300 }}
      onMouseMove={e => setPosition({ x: e.clientX, y: e.clientY })}
    >
      {render(position)}
    </div>
  );
}

// Usage
<MouseTracker render={({ x, y }) => (
  <Tooltip style={{ left: x, top: y }}>
    Position: {x}, {y}
  </Tooltip>
)} />

// In most cases today, a custom hook is cleaner than render props
function useMouse() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    const handler = (e: MouseEvent) => setPosition({ x: e.clientX, y: e.clientY });
    el.addEventListener('mousemove', handler);
    return () => el.removeEventListener('mousemove', handler);
  }, []);

  return { ref, ...position };
}
```

---

## Higher-Order Components (HOC)

HOCs are largely replaced by hooks, but still useful for cross-cutting concerns
like authentication guards and analytics:

```tsx
// Auth guard HOC
function withAuth<P extends object>(Component: ComponentType<P>) {
  return function AuthenticatedComponent(props: P) {
    const { user, loading } = useAuth();

    if (loading) return <Spinner />;
    if (!user) return <Navigate to="/login" />;

    return <Component {...props} />;
  };
}

// Usage
const ProtectedDashboard = withAuth(Dashboard);

// Prefer hooks when logic doesn't require wrapping the render:
// withAuth → useAuth() + early return
// withLogger → useLogger()
// HOC is still the right tool for auth redirects and error boundaries
// (error boundaries still require class components)
```

---

## Provider Pattern (Dependency Injection)

Use context to inject dependencies — makes testing much easier:

```tsx
// Define the interface
interface ApiClient {
  getUser: (id: string) => Promise<User>;
  updateUser: (id: string, data: Partial<User>) => Promise<User>;
}

const ApiContext = createContext<ApiClient | null>(null);

export function useApi(): ApiClient {
  const api = useContext(ApiContext);
  if (!api) throw new Error('ApiProvider is required');
  return api;
}

// Production provider
export function ApiProvider({ children }: { children: ReactNode }) {
  const client = useMemo(() => createHttpApiClient(process.env.API_URL!), []);
  return <ApiContext value={client}>{children}</ApiContext>;
}

// Test provider — swap in a mock
export function MockApiProvider({
  children,
  overrides = {},
}: {
  children: ReactNode;
  overrides?: Partial<ApiClient>;
}) {
  const mock = useMemo<ApiClient>(() => ({
    getUser: async (id) => ({ id, name: 'Test User', email: 'test@example.com' }),
    updateUser: async (id, data) => ({ id, name: 'Test User', email: 'test@example.com', ...data }),
    ...overrides,
  }), [overrides]);

  return <ApiContext value={mock}>{children}</ApiContext>;
}
```

---

## State Colocation

State should live as close as possible to where it's used:

```tsx
// BAD: lifting state too high — every component re-renders on tooltip change
function App() {
  const [tooltipVisible, setTooltipVisible] = useState(false); // used only in Nav
  return (
    <>
      <Nav tooltipVisible={tooltipVisible} setTooltipVisible={setTooltipVisible} />
      <Main />     {/* re-renders on tooltip change — unnecessary */}
      <Footer />   {/* re-renders on tooltip change — unnecessary */}
    </>
  );
}

// GOOD: colocate state where it's used
function Nav() {
  const [tooltipVisible, setTooltipVisible] = useState(false); // lives here
  return <nav>{/* ... */}</nav>;
}

function App() {
  return (
    <>
      <Nav />    {/* only Nav re-renders on tooltip change */}
      <Main />
      <Footer />
    </>
  );
}
```

---

## Component Slots via Children Props

```tsx
// Explicit named slots — cleaner than many optional props
interface CardProps {
  header?: ReactNode;
  children: ReactNode;
  footer?: ReactNode;
  className?: string;
}

function Card({ header, children, footer, className }: CardProps) {
  return (
    <div className={`card ${className ?? ''}`}>
      {header && <div className="card-header">{header}</div>}
      <div className="card-body">{children}</div>
      {footer && <div className="card-footer">{footer}</div>}
    </div>
  );
}

// Usage — clean and composable
<Card
  header={<h2>Order Summary</h2>}
  footer={<Button>Checkout</Button>}
>
  <OrderLineItems items={cartItems} />
</Card>
```

---

## Conditional Rendering Patterns

```tsx
// Simple: &&
{isLoggedIn && <UserMenu />}

// Ternary: when you need an else
{isLoggedIn ? <UserMenu /> : <LoginButton />}

// Multiple conditions: early return is clearest
function StatusBadge({ status }: { status: 'active' | 'pending' | 'archived' }) {
  if (status === 'archived') return null;
  if (status === 'pending')  return <Badge color="yellow">Pending</Badge>;
  return <Badge color="green">Active</Badge>;
}

// Guard against null/undefined in optional chaining JSX
{user?.preferences?.theme && <ThemeSwitcher current={user.preferences.theme} />}

// > React 19.2+: Activity for when you want to preserve state while hiding
{/* Instead of: {activeTab === 'settings' && <Settings />} */}
<Activity mode={activeTab === 'settings' ? 'visible' : 'hidden'}>
  <Settings />
</Activity>
```
*Last Updated: 2026-08-03*
