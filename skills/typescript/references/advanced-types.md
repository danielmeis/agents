# TypeScript Advanced Type Patterns — Deep Reference

> Load this file for conditional types, infer, mapped type modifiers,
> recursive types, branded types, and variance questions.

---

## Conditional Types

```ts
// Basic conditional type
type IsString<T> = T extends string ? true : false;
type A = IsString<'hello'>; // true
type B = IsString<42>;      // false

// Practical use: extract types based on a condition
type NonNullableFields<T> = {
  [K in keyof T]: T[K] extends null | undefined ? never : K;
}[keyof T];

interface Form {
  name: string;
  age: number | null;
  email: string | undefined;
}
type RequiredFields = NonNullableFields<Form>; // 'name'

// Distributive conditional types — applies to each member of a union separately
type ToArray<T> = T extends unknown ? T[] : never;
type Result = ToArray<string | number>; // string[] | number[] (distributed)

// Prevent distribution by wrapping in a tuple
type ToArrayNonDist<T> = [T] extends [unknown] ? T[] : never;
type Result2 = ToArrayNonDist<string | number>; // (string | number)[]
```

---

## `infer` — Extracting Types from Other Types

```ts
// Extract the element type of an array
type ElementType<T> = T extends (infer E)[] ? E : never;
type Item = ElementType<string[]>; // string

// Extract the return type of a function (this is how ReturnType is built)
type MyReturnType<T> = T extends (...args: never[]) => infer R ? R : never;

// Extract the resolved type of a Promise (this is how Awaited works)
type Unwrap<T> = T extends Promise<infer U> ? U : T;
type Resolved = Unwrap<Promise<string>>; // string

// Extract argument types
type FirstArg<T> = T extends (arg: infer A, ...rest: never[]) => unknown ? A : never;
function greet(name: string, age: number) {}
type Name = FirstArg<typeof greet>; // string

// Recursive infer — deeply unwrap nested promises
type DeepAwaited<T> = T extends Promise<infer U> ? DeepAwaited<U> : T;
type Result3 = DeepAwaited<Promise<Promise<string>>>; // string

// Extract route params from a template literal (common in routing libraries)
type ExtractParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractParams<`/${Rest}`>
    : T extends `${string}:${infer Param}`
      ? Param
      : never;

type Params = ExtractParams<'/users/:userId/posts/:postId'>; // 'userId' | 'postId'
```

---

## Mapped Types

```ts
// Basic mapped type
type Optional<T> = { [K in keyof T]?: T[K] };
type Readonly2<T> = { readonly [K in keyof T]: T[K] };

// Key remapping with 'as' (introduced in TS 4.1, still essential)
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
interface Person { name: string; age: number }
type PersonGetters = Getters<Person>;
// { getName: () => string; getAge: () => number }

// Filtering keys during mapping
type OnlyStringProps<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};
interface Mixed { name: string; age: number; email: string }
type StringProps = OnlyStringProps<Mixed>; // { name: string; email: string }

// Modifier manipulation: add/remove readonly and optional
type MutableRequired<T> = {
  -readonly [K in keyof T]-?: T[K];  // remove readonly AND remove optional
};
type ReadonlyPartial<T> = {
  +readonly [K in keyof T]+?: T[K];  // add readonly AND add optional (explicit +, same as default)
};
```

---

## Recursive Types

```ts
// JSON value type — genuinely recursive, common in real code
type JsonValue =
  | string
  | number
  | boolean
  | null
  | JsonValue[]
  | { [key: string]: JsonValue };

// Deep partial — apply Partial recursively through nested objects
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

interface Settings {
  user: { name: string; preferences: { theme: string; notifications: boolean } };
}
type PartialSettings = DeepPartial<Settings>;
// user, preferences, and all leaf fields become optional at every level

// Deep readonly
type DeepReadonly<T> = T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

// Path type for nested property access (used by libraries like lodash.get)
type Path<T, K extends keyof T = keyof T> = K extends string
  ? T[K] extends object
    ? `${K}` | `${K}.${Path<T[K]>}`
    : `${K}`
  : never;

interface Nested { a: { b: { c: number } } }
type NestedPaths = Path<Nested>; // 'a' | 'a.b' | 'a.b.c'
```

---

## Branded / Nominal Types

TypeScript's type system is structural — two types with the same shape are
interchangeable. Branded types simulate nominal typing when you need to
prevent mixing semantically different values with the same underlying type:

```ts
// Problem: these are both `string`, easy to accidentally swap
function getUser(userId: string) { /* ... */ }
function getOrder(orderId: string) { /* ... */ }
const orderId = '123';
getUser(orderId); // ❌ compiles fine, but is a bug — wrong ID type passed

// Solution: brand the types
type UserId = string & { readonly __brand: 'UserId' };
type OrderId = string & { readonly __brand: 'OrderId' };

function toUserId(id: string): UserId { return id as UserId; }
function toOrderId(id: string): OrderId { return id as OrderId; }

function getUser(userId: UserId) { /* ... */ }
function getOrder(orderId: OrderId) { /* ... */ }

const orderId2 = toOrderId('123');
// getUser(orderId2); // ✅ now a compile error — caught the bug

// Common branded type utility
type Brand<T, B extends string> = T & { readonly __brand: B };
type Email = Brand<string, 'Email'>;
type PositiveInt = Brand<number, 'PositiveInt'>;

function createEmail(value: string): Email | null {
  return /^[^@]+@[^@]+\.[^@]+$/.test(value) ? (value as Email) : null;
}
```

---

## Variance

```ts
// Covariance: function return types are covariant — a function returning a
// more specific type is assignable where a less specific return is expected
type AnimalMaker = () => { name: string };
type DogMaker = () => { name: string; breed: string };
const makeDog: DogMaker = () => ({ name: 'Rex', breed: 'Lab' });
const makeAnimal: AnimalMaker = makeDog; // ✅ covariant — safe

// Contravariance: function parameter types are contravariant — a function
// accepting a broader type is assignable where a narrower parameter is expected
type DogHandler = (dog: { name: string; breed: string }) => void;
type AnimalHandler = (animal: { name: string }) => void;
const handleAnimal: AnimalHandler = (animal) => console.log(animal.name);
const handleDog: DogHandler = handleAnimal; // ✅ contravariant — safe (animal handler can handle dogs)

// Method parameters are bivariant by default (a historical TS looseness);
// use strictFunctionTypes (part of strict: true) to make them contravariant
// like standalone functions — this catches more real bugs
interface Comparator<T> {
  compare(a: T, b: T): number; // method syntax — bivariant even under strict, use with care
}
type CompareFn<T> = (a: T, b: T) => number; // function property — properly contravariant under strict
```

---

## `keyof`, `typeof`, and Indexed Access

```ts
interface User { id: string; name: string; age: number }

// keyof — union of property names
type UserKeys = keyof User; // 'id' | 'name' | 'age'

// typeof — extract a type from a value
const defaultUser = { id: '', name: '', age: 0 };
type DefaultUserType = typeof defaultUser; // { id: string; name: string; age: number }

// Indexed access — extract a specific property's type
type UserId2 = User['id']; // string
type UserValueTypes = User[keyof User]; // string | number

// Combine for dynamic, type-safe getters
function get<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// const enum-like pattern with 'as const' + typeof + keyof (see satisfies section too)
const Roles = { Admin: 'admin', Member: 'member' } as const;
type Role = typeof Roles[keyof typeof Roles]; // 'admin' | 'member'
```

---

## Const Type Parameters

```ts
// Without const modifier: literal types get widened when inferred through generics
function firstElement<T>(arr: T[]): T {
  return arr[0];
}
const result = firstElement(['a', 'b', 'c']); // inferred as string, not 'a' | 'b' | 'c'

// With const modifier: preserves literal types
function firstElementConst<const T>(arr: T[]): T {
  return arr[0];
}
const result2 = firstElementConst(['a', 'b', 'c']); // inferred as 'a' | 'b' | 'c'

// Practical use: preserve tuple and literal inference without callers
// needing to write `as const` everywhere
function createRoute<const T extends string>(path: T): { path: T } {
  return { path };
}
const route = createRoute('/users/:id'); // path: '/users/:id', not string
```

---

## Function Type Best Practices

```ts
// Prefer function type expressions over interfaces for simple callbacks
type Callback = (error: Error | null, data?: string) => void; // ✅ concise

interface CallbackInterface {  // more verbose for the same thing
  (error: Error | null, data?: string): void;
}

// Optional parameters vs union with undefined — prefer optional for
// parameters, union for object properties you want to require presence of
function fetchData(url: string, options?: RequestOptions): Promise<Response> {} // ✅
interface Config { timeout: number | undefined } // require the key to exist, value may be undefined

// Overload signatures vs generics — use generics unless behavior genuinely
// differs by input type (see main SKILL.md for full overload example)
```

*Last Updated: 2026-07-29*
