# React Testing — Deep Reference

> React Testing Library (RTL) with Vitest (preferred) or Jest.
> Tests should resemble how users interact with your app.
> Query priority: accessible queries first, never query by implementation detail.

---

## Setup

```bash
# Vitest (preferred for Vite projects) + RTL
npm install --save-dev vitest @testing-library/react @testing-library/user-event
npm install --save-dev @testing-library/jest-dom jsdom

# Jest (for CRA or non-Vite projects)
npm install --save-dev jest @testing-library/react @testing-library/user-event
npm install --save-dev @testing-library/jest-dom jest-environment-jsdom
npm install --save-dev @babel/preset-react @babel/preset-typescript
```

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './src/test/setup.ts',
  },
});
```

```ts
// src/test/setup.ts
import '@testing-library/jest-dom';
// Now matchers like .toBeInTheDocument(), .toHaveValue() are available
```

---

## Query Priority (Most to Least Preferred)

RTL queries enforce accessible HTML patterns. Use the highest-priority
query that works for your use case:

```tsx
// 1. getByRole — best: matches what screen readers see
screen.getByRole('button', { name: /submit/i })
screen.getByRole('textbox', { name: /email/i })
screen.getByRole('heading', { name: /welcome/i, level: 1 })
screen.getByRole('checkbox', { name: /remember me/i })
screen.getByRole('link', { name: /learn more/i })

// 2. getByLabelText — for form fields with labels
screen.getByLabelText(/email address/i)
screen.getByLabelText('Password')

// 3. getByPlaceholderText — only if no label exists
screen.getByPlaceholderText(/search products/i)

// 4. getByText — for non-interactive text content
screen.getByText(/terms and conditions/i)

// 5. getByAltText — for images
screen.getByAltText(/company logo/i)

// 6. getByTitle — for title attributes
screen.getByTitle(/close modal/i)

// 7. getByTestId — last resort; only when nothing semantic works
screen.getByTestId('complex-widget')
// Add data-testid="complex-widget" to the element

// NEVER: don't query by className, id, or internal implementation
// screen.container.querySelector('.my-button')  ❌
```

---

## Query Variants

```tsx
// getBy*    — throws if not found or if multiple found — use for expected elements
// queryBy*  — returns null if not found — use for asserting absence
// findBy*   — async, returns a promise — use for elements that appear asynchronously
// getAllBy*  — returns array, throws if empty
// queryAllBy* — returns array, empty if none found
// findAllBy*  — async array

// Asserting an element is NOT present
expect(screen.queryByRole('alert')).not.toBeInTheDocument();

// Waiting for async elements
const alert = await screen.findByRole('alert'); // waits up to 1000ms by default
```

---

## Basic Component Tests

```tsx
// UserCard.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { UserCard } from './UserCard';

describe('UserCard', () => {
  const mockUser = { userId: 1, displayName: 'Alice Smith' };

  it('renders the display name', () => {
    render(<UserCard {...mockUser} onSelect={() => {}} />);
    expect(screen.getByRole('heading', { name: 'Alice Smith' })).toBeInTheDocument();
  });

  it('calls onSelect with userId when clicked', async () => {
    const handleSelect = vi.fn(); // jest.fn() in Jest
    render(<UserCard {...mockUser} onSelect={handleSelect} />);

    await userEvent.click(screen.getByRole('article'));

    expect(handleSelect).toHaveBeenCalledWith(1);
    expect(handleSelect).toHaveBeenCalledTimes(1);
  });

  it('renders default avatar when avatarUrl is not provided', () => {
    render(<UserCard {...mockUser} onSelect={() => {}} />);
    expect(screen.getByRole('img')).toHaveAttribute('src', '/default-avatar.png');
  });
});
```

---

## Testing Forms and User Interactions

```tsx
// Use @testing-library/user-event v14+ — simulates real browser events
// NOT fireEvent — it fires synthetic events, not realistic user interactions

import userEvent from '@testing-library/user-event';

describe('LoginForm', () => {
  // Set up userEvent once per test (v14 API)
  const setup = () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();
    render(<LoginForm onSubmit={onSubmit} />);
    return { user, onSubmit };
  };

  it('submits with email and password', async () => {
    const { user, onSubmit } = setup();

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.type(screen.getByLabelText(/password/i), 'password123');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    expect(onSubmit).toHaveBeenCalledWith({
      email: 'alice@example.com',
      password: 'password123',
    });
  });

  it('shows validation error for empty email', async () => {
    const { user } = setup();

    await user.click(screen.getByRole('button', { name: /sign in/i }));

    expect(screen.getByRole('alert')).toHaveTextContent('Email is required');
  });

  it('disables the submit button while submitting', async () => {
    const { user, onSubmit } = setup();
    onSubmit.mockReturnValue(new Promise(() => {})); // never resolves

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.type(screen.getByLabelText(/password/i), 'pass');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    expect(screen.getByRole('button', { name: /sign in/i })).toBeDisabled();
  });
});
```

---

## Testing Async Components

```tsx
// Component that fetches data
describe('UserProfile', () => {
  it('shows loading state then user data', async () => {
    // Mock the fetch
    vi.mocked(fetchUser).mockResolvedValue({ id: '1', name: 'Alice' });

    render(<UserProfile userId="1" />);

    // Loading state
    expect(screen.getByRole('progressbar')).toBeInTheDocument();

    // Wait for data to appear
    expect(await screen.findByText('Alice')).toBeInTheDocument();
    expect(screen.queryByRole('progressbar')).not.toBeInTheDocument();
  });

  it('shows error when fetch fails', async () => {
    vi.mocked(fetchUser).mockRejectedValue(new Error('Network error'));

    render(<UserProfile userId="1" />);

    expect(await screen.findByRole('alert')).toHaveTextContent('Network error');
  });
});

// Testing with Suspense
describe('Comments (Suspense)', () => {
  it('shows fallback then content', async () => {
    const promise = Promise.resolve([{ id: '1', text: 'Hello' }]);

    render(
      <Suspense fallback={<div>Loading comments…</div>}>
        <Comments promise={promise} />
      </Suspense>
    );

    expect(screen.getByText('Loading comments…')).toBeInTheDocument();
    expect(await screen.findByText('Hello')).toBeInTheDocument();
  });
});
```

---

## Mocking Modules

```tsx
// Vitest
vi.mock('./api/users', () => ({
  fetchUser: vi.fn(),
  updateUser: vi.fn(),
}));

// Import the mocked version for type-safe access
import { fetchUser } from './api/users';
const mockFetchUser = vi.mocked(fetchUser);

// Per-test overrides
beforeEach(() => {
  mockFetchUser.mockResolvedValue({ id: '1', name: 'Alice' });
});

afterEach(() => {
  vi.clearAllMocks(); // reset call counts and implementations
});

it('uses mocked value', async () => {
  mockFetchUser.mockResolvedValueOnce({ id: '1', name: 'Override' });
  // this test gets 'Override'; next test gets the beforeEach value
});
```

---

## Testing Custom Hooks

Use `renderHook` from RTL — never test hooks in isolation by calling them directly:

```tsx
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('initializes with provided value', () => {
    const { result } = renderHook(() => useCounter(5));
    expect(result.current.count).toBe(5);
  });

  it('increments count', () => {
    const { result } = renderHook(() => useCounter(0));

    act(() => result.current.increment());

    expect(result.current.count).toBe(1);
  });

  it('resets to initial value', () => {
    const { result } = renderHook(() => useCounter(10));

    act(() => {
      result.current.increment();
      result.current.increment();
      result.current.reset();
    });

    expect(result.current.count).toBe(10);
  });
});

// Hooks that need context: provide a wrapper
describe('useTheme', () => {
  it('returns the current theme', () => {
    const { result } = renderHook(() => useTheme(), {
      wrapper: ({ children }) => <ThemeProvider>{children}</ThemeProvider>,
    });
    expect(result.current.theme).toBe('light');
  });
});
```

---

## Testing with Context and Providers

```tsx
// Create a custom render function that wraps with providers
import { render, RenderOptions } from '@testing-library/react';

function AllProviders({ children }: { children: ReactNode }) {
  return (
    <MockApiProvider>
      <ThemeProvider>
        <AuthProvider user={mockUser}>
          {children}
        </AuthProvider>
      </ThemeProvider>
    </MockApiProvider>
  );
}

function customRender(ui: ReactElement, options?: RenderOptions) {
  return render(ui, { wrapper: AllProviders, ...options });
}

// Re-export everything from RTL but override render
export * from '@testing-library/react';
export { customRender as render };

// In tests: import from your custom file, not from RTL directly
import { render, screen } from '../test/utils';

it('shows user name from context', () => {
  render(<Header />);
  expect(screen.getByText('Test User')).toBeInTheDocument();
});
```

---

## Accessibility Testing

```tsx
// @testing-library/jest-dom accessibility matchers
expect(button).toBeEnabled();
expect(input).toHaveFocus();
expect(checkbox).toBeChecked();
expect(link).toHaveAttribute('href', '/about');
expect(img).toHaveAttribute('alt', 'Company logo');
expect(field).toHaveValue('alice@example.com');
expect(region).toHaveAccessibleName('Navigation');

// Test keyboard navigation
const { user } = setup();
await user.tab();  // move focus to next focusable element
expect(screen.getByRole('button', { name: /submit/i })).toHaveFocus();

await user.keyboard('{Enter}');  // activate focused element
await user.keyboard('{Escape}'); // close modals, dropdowns, etc.
await user.keyboard('{ArrowDown}'); // navigate lists

// axe-core for automated accessibility auditing (complements manual tests)
import { axe } from 'jest-axe';

it('has no accessibility violations', async () => {
  const { container } = render(<LoginForm onSubmit={() => {}} />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

---

## What Not to Test

```tsx
// Don't test implementation details — these make tests brittle
expect(component.state.isOpen).toBe(true);           // ❌ internal state
expect(component.instance().handleClick).toBeDefined(); // ❌ method existence
expect(container.querySelector('.modal')).toBeVisible(); // ❌ CSS class

// Don't snapshot-test large component trees — they break on any change
// and tell you nothing about behavior
expect(tree).toMatchSnapshot(); // ❌ for large components

// Small snapshots are okay for things like icon output or simple templates
expect(renderIcon('star')).toMatchSnapshot(); // ✅ stable, meaningful

// Don't test third-party library behavior
// Don't test styles (use visual regression tools like Chromatic for that)
// Don't test propTypes warnings in tests — the types handle that now
```

*Last Updated: 2026-07-29*
