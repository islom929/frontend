# React Patterns — Interview Savollari

> Components, Composition, State Management, Event Handling, Legacy Patterns (HOC, Render Props), Compound Components, Error Boundaries, Portals — barcha komponent-darajadagi pattern'lar.

---

## Mundarija

- [**QISM A: Components & Render Purity** (savollar 1-4)](#qism-a)
- [**QISM B: Props** (savollar 5-10)](#qism-b)
- [**QISM C: Composition** (savollar 11-13)](#qism-c)
- [**QISM D: State Lifting & Controlled** (savollar 14-17)](#qism-d)
- [**QISM E: Event Handling** (savollar 18-22)](#qism-e)
- [**QISM F: Legacy Patterns** (savollar 23-26)](#qism-f)
- [**QISM G: Compound Components & Children API** (savollar 27-29)](#qism-g)
- [**QISM H: Error Boundaries** (savollar 30-33)](#qism-h)
- [**QISM I: Portals** (savollar 34-36)](#qism-i)

---

<a id="qism-a"></a>

## QISM A: Components & Render Purity

### 1. Function components va render purity invariant [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Function component** — JS function bo'lib, props oladi va JSX (React Element) qaytaradi. **Render purity** — fundamental invariant: **bir xil props/state → bir xil JSX**. No side effects in render body (no `setState`, no DOM mutation, no fetch, no `Math.random`, no `Date.now`). Concurrent rendering, Strict Mode 2x, memoization, server rendering — barchasi shu invariantga tayanadi.

### To'liq tushuntirish

**Render purity qoidalari:**

- ✅ Same props/state → same JSX (deterministic)
- ✅ No side effects in body
- ✅ No DOM mutation
- ✅ No setState (outside event/effect)
- ✅ No fetch / external I/O
- ✅ No reading mutable globals (Date.now, Math.random)
- ✅ Idempotent — rendering N times = rendering once

**Why purity matters:**

1. **Concurrent rendering** — render may be aborted, restarted, run multiple times
2. **Strict Mode 2x** — explicit invariant testing in dev
3. **Memoization** — bailout assumes pure render
4. **Server rendering** — same input must produce consistent output

### Kod misoli

```tsx
// ✅ Pure
function Greeting({ name }: { name: string }) {
  return <h1>Hello, {name}</h1>;
}

// ❌ Side effect — setState in body
function BadCounter() {
  const [count, setCount] = useState(0);
  setCount(count + 1);  // ❌ infinite loop
  return <p>{count}</p>;
}

// ❌ Mutation
function BadList({ items }: { items: Item[] }) {
  items.sort();  // ❌ mutates props
  return <ul>{items.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}

// ✅ Immutable
function GoodList({ items }: { items: Item[] }) {
  const sorted = [...items].sort((a, b) => a.name.localeCompare(b.name));
  return <ul>{sorted.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}

// ❌ DOM mutation
function BadGreeting({ title }: { title: string }) {
  document.title = title;  // ❌ side effect
  return <h1>{title}</h1>;
}

// ✅ Effect for DOM
function GoodGreeting({ title }: { title: string }) {
  useEffect(() => {
    document.title = title;
    return () => { document.title = "Default"; };
  }, [title]);
  return <h1>{title}</h1>;
}

// ❌ Reading mutable
function BadId() {
  const id = Math.random();  // ❌ different each render
  return <p>{id}</p>;
}

// ✅ Stable ID
function GoodId() {
  const id = useId();  // SSR-safe stable
  return <p>{id}</p>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Render abort scenarios:**

```typescript
// React may abort render mid-way:
// 1. Higher priority update interrupts
// 2. Suspense throws
// 3. Error in component
// → workInProgress tree discarded
// → Render restarted with same or new state
```

If render impure, abort + restart → different output → tearing/inconsistency.

**Side effects belong in:**

- **Event handlers** — user interaction (click, change)
- **`useEffect`** — synchronization with external systems
- **`useLayoutEffect`** — DOM measurement, sync DOM mutation

**Strict Mode 2x render dev:**

```tsx
function Counter() {
  const [c, setC] = useState(0);
  console.log("render");  // logs 2x in StrictMode dev

  // Side effect would run 2x — exposes bugs
}
```

**`useState` with impure initializer:**

```tsx
// ❌ Side effect in initializer
const [data] = useState(() => {
  fetch("/log");  // ❌ runs twice in StrictMode dev
  return 0;
});

// ✅ Pure initializer
const [data] = useState(() => 0);
// Effects in useEffect
```

**Reading external mutable values:**

```tsx
// ❌ External state read in render
let counter = 0;
function Component() {
  counter++;  // ❌ side effect
  return <p>{counter}</p>;
}

// ❌ Reading mutable global
function Component() {
  const time = Date.now();  // ❌ different each render
  return <p>{time}</p>;
}

// ✅ Read via state/effect
function Component() {
  const [time, setTime] = useState(() => Date.now());
  useEffect(() => {
    const id = setInterval(() => setTime(Date.now()), 1000);
    return () => clearInterval(id);
  }, []);
  return <p>{time}</p>;
}
```

**Mutating refs in render:**

```tsx
// ❌ Mutate ref in render
function Component() {
  const ref = useRef(0);
  ref.current++;  // ❌ unsafe in concurrent rendering
  return <p>{ref.current}</p>;
}
// Render abort → ref.current advanced wrongly
```

**Pure render even with class:**

```tsx
class OldComponent extends React.Component {
  render() {
    // Same purity rules apply
    // No setState, no API calls, no DOM mutation
    return <p>{this.props.value}</p>;
  }
}
```

**Compiler enforcement:**

React Compiler (R19) auto-memoizes assuming purity. Impure code — Compiler skips (warnings via ESLint).

**`useSyncExternalStore` for external state:**

```tsx
// ✅ Safe external store read in render
const value = useSyncExternalStore(store.subscribe, store.getSnapshot);
```

`useSyncExternalStore` — guaranteed consistent reads across re-renders.

</details>

### Edge Cases

- **Conditional setState in render**: Can be valid in rare cases (state derived from props). React handles via re-render queue.
- **`console.log` in render**: Pure (output, no state change). But duplicated in StrictMode.
- **Reading `useRef.current`**: Generally avoid in render. If needed, ensure stability.

### Follow-up savollar

- "Pure render component'ning testi?" — Snapshot test: same props → same output.
- "Class lifecycle methods purity?" — `render()` pure. `componentDidMount` for side effects.
- "Compiler ishlatilsa purity to'liq emas?" — Compiler hali validation cheklovlari (mutation tracker), aniq cases miss qilishi mumkin.

</details>

---

### 2. Idempotency va Strict Mode [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Idempotency** — bir xil operatsiyani **bir necha marta bajarish bir martalik bilan teng natija beradi**. React rendering uchun: komponent N marta render qilinsa ham, oxirgi DOM holati bir martalik render bilan bir xil bo'lishi shart. Strict Mode 2x render — idempotency tekshiradi (dev). Side effects ham idempotent (effect cleanup-resilient).

### To'liq tushuntirish

**Idempotency React uchun nima uchun muhim:**

1. **Concurrent rendering** — render abort qilinib, qaytadan boshlanishi mumkin. Har safar bir xil natija berishi shart
2. **Strict Mode dev** — 2x render, 2x effect setup-cleanup-setup — impure code xatolarini aniqlash uchun
3. **Memoization** — `React.memo` va `useMemo` idempotent render'ga tayanadi (same input = same output)
4. **Server rendering** — server va client render bir xil HTML berishi shart (hydration uchun)

**Idempotent vs deterministic farqi:**
- **Deterministic** — same input = same output (har safar)
- **Idempotent** — N marta bajarish = 1 marta bajarish (natija ortiqcha o'zgarmaydi)
- React ikkalasini talab qiladi: render pure (deterministic) va effects idempotent (cleanup-resilient)

### Kod misoli

```tsx
// ✅ Idempotent
function Counter() {
  const [count, setCount] = useState(0);
  // Render N times: count stays consistent
  return <p>{count}</p>;
}

// ❌ Non-idempotent — counter incremented in render
let renderCount = 0;
function Bad() {
  renderCount++;  // ❌ side effect
  return <p>Render: {renderCount}</p>;
}

// Strict Mode dev: rendered 2x → renderCount = 2 instead of 1
// Production concurrent rendering: renderCount unpredictable

// ✅ Idempotent effect
useEffect(() => {
  const sub = api.subscribe(handler);
  return () => sub.unsubscribe();
}, []);

// ❌ Non-idempotent effect
useEffect(() => {
  api.subscribe(handler);  // no cleanup
}, []);
// Mount-unmount-mount: 2 subscriptions
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Strict Mode dev simulates:**

- 2x render
- 2x effect setup-cleanup-setup
- 2x useState initializer
- 2x useMemo factory

```tsx
function App() {
  console.log("render");  // logs 2x in dev
  const [data] = useState(() => {
    console.log("init");  // logs 2x in dev
    return computeInitial();
  });
  return <p>{data}</p>;
}
```

**Production concurrent rendering:**

```typescript
// React may execute:
// 1. Render
// 2. Suspended → throw promise
// 3. Restart render after promise resolves
// 4. Render
// 5. Higher priority interrupt → discard, restart
// 6. Render

// All these executions must be idempotent.
```

**Idempotent operations:**

```tsx
// ✅ Setting value (idempotent)
useEffect(() => {
  document.title = "New Title";
});
// Mount-cleanup-mount: title still "New Title"

// ❌ Counter (non-idempotent)
useEffect(() => {
  counter++;
});
// Mount-cleanup-mount: counter advanced

// ✅ Subscribe with cleanup (idempotent)
useEffect(() => {
  const sub = subscribe();
  return () => sub.unsubscribe();
});

// ❌ Async fetch without cleanup
useEffect(() => {
  fetch("/api");  // 2 fetches in dev StrictMode
});
```

**useState initializer purity:**

```tsx
// ❌ Side effect in initializer
const [items] = useState(() => {
  console.log("init");  // logs 2x in dev
  fetch("/log");          // ❌ 2 fetches!
  return [];
});

// ✅ Pure
const [items] = useState(() => []);

// Effects in useEffect
useEffect(() => {
  fetch("/log");  // proper cleanup
}, []);
```

**Idempotency for refs:**

```tsx
// ❌ Mutate ref in render
function Comp() {
  const ref = useRef(0);
  ref.current++;
  // 2x render: ref.current = 2 instead of 1
  return <p>{ref.current}</p>;
}

// ✅ Mutate ref in effect/handler
function Comp() {
  const ref = useRef(0);
  useEffect(() => {
    ref.current++;
  });
  return <p>{ref.current}</p>;
}
```

**Idempotent server rendering:**

```tsx
// SSR — same input must produce same HTML
function Page({ user }) {
  const id = Math.random();  // ❌ different on server vs client
  return <p>{id}</p>;
}

// ✅ Pure
function Page({ user }) {
  const id = useId();  // stable across SSR/client
  return <p>{id}</p>;
}
```

**Hydration assumes idempotency:**

```tsx
// Server render produces HTML
// Client hydrates — expects same render

// Non-idempotent → hydration mismatch
function Time() {
  return <p>{new Date().toLocaleString()}</p>;
}

// ✅ Use useEffect or `suppressHydrationWarning`
```

**Idempotency for state machines:**

```tsx
// ✅ Pure reducer (idempotent transition)
function reducer(state, action) {
  switch (action.type) {
    case "INC":
      return { count: state.count + 1 };
  }
}
// Same state + action → same new state

// ❌ Impure reducer
function reducer(state, action) {
  console.log("dispatch");  // ❌ side effect
  fetch("/log");             // ❌
  return { count: state.count + 1 };
}
```

</details>

### Edge Cases

- **Idempotent vs deterministic**: Idempotent = N applications = 1. Deterministic = same input → same output. React requires both.
- **Network requests in effect**: Idempotent if cleanup cancels (AbortController).
- **Animation start in effect**: Stop on cleanup, restart on remount. Idempotent.

### Follow-up savollar

- "Idempotency kafolati yo'q nima bo'ladi?" — Concurrent rendering bug'lar (state inconsistency). Strict Mode catches.
- "Test idempotency?" — Run twice, check same result. RTL with StrictMode default.
- "Sluggish but idempotent — performance impact?" — StrictMode dev 2x render. Production 1x — no penalty.

</details>

---

### 3. Component naming — PascalCase vs lowercase [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JSX'da **PascalCase** (`<Component />`) — komponent (function/class). **lowercase** (`<div>`) — HTML/SVG element. JSX transform shu farq bilan `_jsx(Component, ...)` (function reference) yoki `_jsx("div", ...)` (string tag) qiladi. ESLint rule majburlaydi.

### To'liq tushuntirish

JSX transpiler (Babel/SWC/TypeScript) element tag'ni case bo'yicha aniqlaydi:
- **PascalCase** (`<Header />`) — user-defined component. Transform: `_jsx(Header, {})` — `Header` function/class reference sifatida pass qilinadi
- **lowercase** (`<div />`) — built-in HTML/SVG element. Transform: `_jsx("div", {})` — string sifatida pass qilinadi
- **Dot notation** (`<Components.Button />`) — member expression, case ahamiyatsiz (har doim expression sifatida resolve qilinadi)

Bu convention React runtime uchun muhim — string tag'lar DOM element yaratadi, function reference'lar component render qiladi.

### Kod misoli

```tsx
function Header() { return <h1>Title</h1>; }

// ✅ PascalCase — recognized as component
<Header />
// _jsx(Header, ...) — function reference

// ❌ lowercase — treated as HTML tag
<header />
// _jsx("header", ...) — HTML element
// React: component named "Header" not used

function App() {
  return (
    <div>             {/* HTML */}
      <Header />       {/* Component */}
    </div>
  );
}
```

```tsx
// Convention by case:
// - Functions/Classes → PascalCase
// - HTML elements → lowercase

// Member access — same rule
const Components = {
  Button: () => <button />,
};

<Components.Button />  // ✅ component
<components.button />  // ❌ treated as HTML
```

<details>
<summary><strong>Deep Dive</strong></summary>

**JSX transform logic:**

```tsx
// Source
<Foo />
<foo />

// Transform output (sodda):
_jsx(Foo, {});      // PascalCase → reference (function/class)
_jsx("foo", {});    // lowercase → string (HTML/SVG element)
```

`_jsx(string, ...)` — host component (DOM tag).
`_jsx(function, ...)` — user component.

**Special case: JSX namespace:**

```tsx
declare global {
  namespace JSX {
    interface IntrinsicElements {
      "custom-element": { foo: string };  // Web Component
    }
  }
}

<custom-element foo="bar" />  // ✅ recognized as HTML (kebab-case)
```

Custom HTML element names allowed (kebab-case for Web Components).

**Member access exception:**

```tsx
import * as Module from "./module";

<Module.Button />  // ✅ resolves to function reference
<Module.button />  // ✅ also resolves (member, not just lowercase)
```

If accessed via dot notation, treated as expression — works regardless of case.

**Inline component — wrong:**

```tsx
function App() {
  // ❌ inside render
  function inline() {  // lowercase
    return <p />;
  }
  return <inline />;  // treated as HTML "inline" tag!
}

// ✅ PascalCase (and hoist outside render)
function Inline() {
  return <p />;
}
function App() {
  return <Inline />;
}
```

**Browser parsing:**

Browser tries to render unknown lowercase tag (e.g., `<header>` → renders as HTML5 element, no error). React: `_jsx("header", ...)` → DOM `header` element.

If component name written in lowercase, React renders as HTML tag (silent failure).

**ESLint rule:**

```json
{
  "rules": {
    "react/jsx-pascal-case": "error"
  }
}
```

**TypeScript awareness:**

```tsx
// TS uses JSX.IntrinsicElements vs ComponentType
const a: JSX.Element = <Component />;  // function component
const b: JSX.Element = <div />;          // intrinsic (HTML)
```

**SVG elements — also lowercase:**

```tsx
<svg viewBox="0 0 100 100">
  <circle cx="50" cy="50" r="40" />  // SVG element
  <rect x="0" y="0" width="100" height="100" />
</svg>
```

Both HTML and SVG use lowercase.

</details>

### Edge Cases

- **`<MyComponent.SubComponent />`**: Member access — works.
- **PascalCase HTML-like name**: `<Header />` — component, not HTML5 `<header>`.
- **Custom Elements**: `<my-element />` — kebab-case allowed for Web Components.

### Follow-up savollar

- "Why PascalCase convention?" — Distinguishes user code from HTML. JSX transform rule.
- "Component all lowercase?" — Possible via member access, not direct.
- "ESLint warning vs error?" — Configurable. Production config — error.

</details>

---

### 4. Output: render side effect bug [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi komponent nima qiladi va Strict Mode'da nima ko'rinadi?

```tsx
let renderCount = 0;

function Counter() {
  renderCount++;  // side effect
  console.log("Render:", renderCount);
  return <p>Render count: {renderCount}</p>;
}

function App() {
  return (
    <StrictMode>
      <Counter />
    </StrictMode>
  );
}
```

### Javob

**Production**:
```
Render: 1
```
DOM: `<p>Render count: 1</p>`

**Development with StrictMode**:
```
Render: 1
Render: 2
```
DOM: `<p>Render count: 2</p>` (last render output)

**Tushuntirish:**

- `renderCount++` — side effect in render body (impure)
- StrictMode 2x render exposes bug
- `renderCount` increments twice in dev (intentional 2x mount)
- Production: 1 render initially, but concurrent rendering may cause unpredictable counts

**Fix — pure render:**

```tsx
// Use useState (state owned by React)
function Counter() {
  const [count, setCount] = useState(0);
  // count owned by React, no global pollution
  return <p>Count: {count}</p>;
}

// Or useRef (mutate in effect, not render)
function Counter() {
  const renderCount = useRef(0);
  useEffect(() => {
    renderCount.current++;
  });
  return <p>Render count: {renderCount.current}</p>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why side effect in render bad:**

1. Render may run multiple times (Strict Mode, concurrent)
2. Aborted renders still execute body
3. Server vs client divergence
4. Prevents memoization correctness
5. Violates React mental model

**Concurrent rendering scenarios:**

```typescript
// 1. Render starts
// 2. Higher priority interrupt → render aborted
// 3. State changed → re-render scheduled
// 4. Render again from scratch

// renderCount incremented during aborted render!
// Final value unpredictable
```

**Strict Mode dev behavior:**

```tsx
// Dev:
// First mount cycle:
//   1. Render → renderCount = 1
//   2. (synthetic unmount)
//   3. Re-render → renderCount = 2
// → Final UI: 2

// Production:
//   Mount → renderCount = 1
// → UI: 1
```

**Bugs from impure render:**

```tsx
// Analytics tracking
function Page({ id }) {
  analytics.track("view", id);  // ❌ multiple track events
  return <Content id={id} />;
}

// Multiple tracks on:
// - StrictMode 2x mount
// - Re-renders
// - Aborted renders

// Fix: useEffect once
function Page({ id }) {
  useEffect(() => {
    analytics.track("view", id);
  }, [id]);  // tracked once per id
  return <Content id={id} />;
}
```

**External state mutation:**

```tsx
// ❌ Mutate global cache
const cache = new Map();
function Component({ id }) {
  if (!cache.has(id)) {
    cache.set(id, fetchData(id));  // ❌ side effect
  }
  // ...
}

// ✅ Use ref or stable cache
function Component({ id }) {
  const cacheRef = useRef(new Map());
  useEffect(() => {
    if (!cacheRef.current.has(id)) {
      fetchData(id).then(d => cacheRef.current.set(id, d));
    }
  }, [id]);
  // ...
}
```

**TypeScript can't catch this:**

TS doesn't enforce purity. ESLint rules + StrictMode runtime checks.

**Compiler limitations:**

React Compiler tries to detect mutations. Some patterns missed. Best practice — write pure code.

**Common offenders:**

```tsx
// 1. Console
console.log("render");  // OK (no state change)

// 2. Counter
counter++;  // ❌

// 3. Date
const time = Date.now();  // ❌ varies each render

// 4. Random
const id = Math.random();  // ❌

// 5. Mutation
items.sort();  // ❌ mutates

// 6. setState
setState(x);  // ❌ infinite loop

// 7. fetch
fetch("/api");  // ❌

// 8. DOM
document.title = title;  // ❌
```

**Allowed in render:**

- Reading props/state
- Computing derived values
- Calling other pure functions
- Conditional logic (ternary, &&)
- JSX construction

</details>

### Edge Cases

- **`console.log` in render**: Pure (no state change). 2x in StrictMode.
- **Reading immutable global**: OK (`process.env.NODE_ENV`).
- **Throwing error in render**: Caught by error boundary.

### Follow-up savollar

- "ESLint rule for this?" — `react-hooks/exhaustive-deps`, `react/no-direct-mutation-state` — partial detection.
- "Strict Mode disable?" — Possible but discouraged. Catches real bugs.
- "Pure but expensive — optimize?" — `useMemo` for derivation, `React.memo` for component.

</details>

---

<a id="qism-b"></a>

## QISM B: Props

### 5. Props basics + immutability invariant [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Props** = komponentga beriladigan parametrlar. Runtime'da har doim object (`{ name, onClick, ... }`). Read-only — child mutate qilmasligi kerak. Bu **konventsiya** (React runtime'da `Object.freeze` deep-freeze qilmaydi) — TS `Readonly<P>` va ESLint qoidalari orqali enforce qilinadi. Mutation bug-prone: parent state'ni buzishi, memoization'ni siniqtirishi va concurrent rendering'da nondeterministic xatti-harakat keltirib chiqarishi mumkin.

### Kod misoli

```tsx
interface UserCardProps {
  name: string;
  age: number;
  onSelect: (name: string) => void;
}

function UserCard({ name, age, onSelect }: UserCardProps) {
  return (
    <div>
      <span>{name} ({age})</span>
      <button onClick={() => onSelect(name)}>Select</button>
    </div>
  );
}

// Usage
<UserCard name="Ali" age={25} onSelect={(n) => console.log(n)} />

// ❌ Mutation
function BadCard({ user }: { user: User }) {
  user.name = "Modified";  // ❌ mutates parent's data
  return <p>{user.name}</p>;
}

// ❌ Spread mutation
function BadList({ items }: { items: Item[] }) {
  items.push({ id: "new" });  // ❌
  return <ul>{items.map(...)}</ul>;
}

// ✅ Immutable
function GoodList({ items }: { items: Item[] }) {
  const sorted = [...items].sort(...);  // copy first
  return <ul>{sorted.map(...)}</ul>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why immutable:**

- One-way data flow integrity
- React state assumes data unchanged unless setState
- Concurrent rendering: render may abort, mutation persists
- Memoization correctness (React.memo shallow equal — relies on stable refs)

**Props at runtime:**

```typescript
// React calls component:
const element = _jsx(MyComponent, props);
// MyComponent(props) — receives props object
// Dev mode: Object.freeze(props) — shallow freeze (top-level assignment xato beradi)
// Production: freeze qilinmaydi — mutation silently muvaffaqiyatga erishadi
```

**Default values — JS default parameters:**

```tsx
// R19+ — defaultProps function component'larda deprecated (console warning beradi)
// JS default parameters ishlatish kerak
function Button({ label = "Click", disabled = false }: Props) {
  return <button disabled={disabled}>{label}</button>;
}

// Pre-R19 pattern — defaultProps (R19'da function component'lar uchun deprecated)
// Class component'larda hali supported
Button.defaultProps = { label: "Click", disabled: false };
```

**Props as function parameters:**

```tsx
// All props approach
function Form(props: { name: string; onChange: (n: string) => void }) {
  return <input value={props.name} onChange={(e) => props.onChange(e.target.value)} />;
}

// Destructure (preferred)
function Form({ name, onChange }: { name: string; onChange: (n: string) => void }) {
  return <input value={name} onChange={(e) => onChange(e.target.value)} />;
}

// With rename
function Form({ name: userName, onChange: setUserName }: Props) { ... }

// With default
function Button({ label = "Submit", ...rest }: Props) { ... }
```

**TypeScript props patterns:**

```tsx
interface ButtonProps {
  label: string;
  onClick: () => void;
  variant?: "primary" | "secondary";
  disabled?: boolean;
}

interface ExtendedButtonProps extends ButtonProps {
  icon?: string;
}

// Union type
type Button = { label: string } | { children: React.ReactNode };

// Discriminated union
type Action =
  | { type: "save" }
  | { type: "delete"; confirm: boolean };
```

**HTML element props:**

```tsx
// Native button props
import type { ComponentProps } from "react";

function CustomButton(props: ComponentProps<"button"> & { variant: string }) {
  const { variant, className = "", ...rest } = props;
  return <button className={`btn btn-${variant} ${className}`} {...rest} />;
}

// Usage:
<CustomButton variant="primary" onClick={handleClick} disabled={loading} type="submit">
  Save
</CustomButton>
```

**Refs as prop (R19):**

```tsx
function Input({ ref, ...props }: InputProps & { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...props} />;
}
```

**Mutating props bug:**

```tsx
// Parent
const user = { name: "Ali" };
<Profile user={user} />;
// ... user object referenced elsewhere

// Child
function Profile({ user }: { user: User }) {
  user.name = "Modified";  // ❌ mutates parent's user object
  return <p>{user.name}</p>;
}

// Issues:
// 1. Parent's user.name now "Modified" silently
// 2. React doesn't detect (reference unchanged)
// 3. Other components using same user — also "Modified"
// 4. State out of sync
```

**Runtime mutation detection:**

React dev mode'da props object'ni **shallow freeze** qiladi (`Object.freeze(props)`) — top-level property'ga assign qilish `TypeError` beradi. Lekin **deep freeze emas** — nested object mutation'lar (`props.user.name = "X"`) catch qilinmaydi. Production'da freeze yo'q — mutation silently muvaffaqiyatga erishadi.

Himoya mexanizmlari:
- **Dev mode freeze** — shallow level mutation catch qilinadi
- **TypeScript** — props parameter `Readonly<P>` ko'rinishida olinadi (interface declaration orqali)
- **ESLint** — `no-param-reassign` (props mutation oldini olish)
- **StrictMode dev** — 2x render orqali mutation effekti ko'rinarli bo'ladi

**Reference identity:**

```tsx
// ❌ Inline object — new ref each render
function Parent() {
  return <Child config={{ theme: "dark" }} />;
}

// ✅ Memoized
function Parent() {
  const config = useMemo(() => ({ theme: "dark" }), []);
  return <Child config={config} />;
}
```

For memo'd children, reference matters.

**Props validation pre-R19 (`propTypes`):**

```tsx
// Pre-R19 — runtime check (deprecated)
Button.propTypes = {
  label: PropTypes.string.isRequired,
  onClick: PropTypes.func.isRequired,
};

// R19 — TypeScript compile-time
interface ButtonProps {
  label: string;
  onClick: () => void;
}
```

</details>

### Edge Cases

- **Children prop**: Special prop, accessed as `children`. ReactNode type.
- **Spread `{...props}`**: Forward all props. Beware unintended HTML attribute leak.
- **Boolean shorthand**: `<input disabled />` = `disabled={true}`.

### Follow-up savollar

- "Props vs state?" — Props from parent, state internal. Props read-only, state mutable via setState.
- "TypeScript props inference?" — Yes, function parameter type inferred for props.
- "Default props in function components?" — JS default parameters. R19 — defaultProps removed.

</details>

---

### 6. `children` prop va `ReactNode` types [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`children`** — special prop, JSX `<Comp>...</Comp>` ichidagi nodes. `React.ReactNode` type — most permissive (string, number, JSX, array, fragment, null/undefined). Other types: `ReactElement` (faqat JSX element), `JSX.Element` (`ReactElement<any, any>` alias — TS'da JSX expression return type). Composition pattern asosi.

### Kod misoli

```tsx
import type { ReactNode } from "react";

interface CardProps {
  children: ReactNode;
  title?: string;
}

function Card({ children, title }: CardProps) {
  return (
    <div className="card">
      {title && <h2>{title}</h2>}
      <div className="card-body">{children}</div>
    </div>
  );
}

// Usage — any valid React content
<Card title="Hello">
  <p>Text</p>
  <strong>Bold</strong>
</Card>

<Card>
  Just text
</Card>

<Card>
  {[<p key="1">A</p>, <p key="2">B</p>]}
</Card>

<Card>{null}</Card>  // OK — skipped
```

```tsx
// Function as children (render prop)
interface Props {
  children: (open: boolean, toggle: () => void) => ReactNode;
}

function Disclosure({ children }: Props) {
  const [open, setOpen] = useState(false);
  return <>{children(open, () => setOpen(o => !o))}</>;
}

// Usage
<Disclosure>
  {(open, toggle) => (
    <>
      <button onClick={toggle}>{open ? "Hide" : "Show"}</button>
      {open && <div>Content</div>}
    </>
  )}
</Disclosure>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Type definitions:**

```typescript
// React.ReactNode — most permissive
type ReactNode =
  | ReactElement
  | string
  | number
  | bigint
  | Iterable<ReactNode>  // arrays, generators
  | ReactPortal
  | boolean       // skipped in render
  | null          // skipped
  | undefined;    // skipped

// React.ReactElement — JSX element only
interface ReactElement<P = any, T extends string | JSXElementConstructor<any> = string | JSXElementConstructor<any>> {
  type: T;
  props: P;
  key: Key | null;
}

// JSX.Element — function component return type
namespace JSX {
  type Element = React.ReactElement<any, any>;
}
```

**When to use which:**

| Type | Use case |
|------|----------|
| `ReactNode` | `children` prop (most permissive) |
| `ReactElement` | Specific JSX element required |
| `JSX.Element` | Component return type (TS infers) |
| `ReactPortal` | Specifically a portal |

**`children` access patterns:**

```tsx
// 1. Render directly
<div>{children}</div>

// 2. Conditional
{children && <div>{children}</div>}

// 3. With React.Children API
import { Children } from "react";

function Layout({ children }: { children: ReactNode }) {
  const count = Children.count(children);
  return (
    <>
      <p>Total: {count}</p>
      {Children.map(children, (child, i) => (
        <div key={i}>Child {i}: {child}</div>
      ))}
    </>
  );
}

// 4. Function as children (render prop)
function Wrapper({ children }: { children: () => ReactNode }) {
  return <>{children()}</>;
}
```

**`PropsWithChildren`:**

```tsx
import type { PropsWithChildren } from "react";

interface CardProps {
  title: string;
}

// Add children automatically
function Card({ title, children }: PropsWithChildren<CardProps>) {
  return <div><h2>{title}</h2>{children}</div>;
}

// Equivalent
function Card({ title, children }: CardProps & { children?: ReactNode }) { ... }
```

**Children types — runtime:**

```tsx
const children = "Hello";          // string
const children = <p>Hello</p>;     // ReactElement
const children = [<p key="1" />, <p key="2" />];  // array
const children = null;              // null — render output bo'sh, lekin Children.count uni hisoblaydi
const children = false;             // boolean — render output bo'sh, lekin Children.count uni hisoblaydi
const children = 42;                // number (rendered as text)
```

**`React.Children` API:**

```tsx
import { Children } from "react";

Children.map(children, fn)       // map over children
Children.forEach(children, fn)   // iterate
Children.count(children)         // count
Children.toArray(children)       // → flat array (with auto keys)
Children.only(children)          // assert exactly one child
```

**`Children.toArray` for stable keys:**

```tsx
function Layout({ children }: { children: ReactNode }) {
  const childArray = Children.toArray(children);
  return (
    <ol>
      {childArray.map((child, i) => (
        <li key={i}>{child}</li>
      ))}
    </ol>
  );
}
```

**`cloneElement`:**

```tsx
import { cloneElement, isValidElement } from "react";

function Inject({ children, extra }: Props) {
  return Children.map(children, (child) => {
    if (isValidElement(child)) {
      return cloneElement(child, { extra });
    }
    return child;
  });
}
```

**Composition patterns:**

```tsx
// Slot pattern
function Layout({ header, content, footer }: {
  header?: ReactNode;
  content: ReactNode;
  footer?: ReactNode;
}) {
  return (
    <>
      <header>{header}</header>
      <main>{content}</main>
      <footer>{footer}</footer>
    </>
  );
}

<Layout
  header={<Logo />}
  content={<Dashboard />}
  footer={<Copyright />}
/>
```

**`children` and memoization:**

```tsx
const MemoCard = memo(Card);

function Parent() {
  const [_, setX] = useState(0);
  return (
    <MemoCard>
      <p>Static</p>  {/* New element ref each render! */}
    </MemoCard>
  );
}
// memo bypassed (children new ref)

// Workaround: parent up the tree owns children
function Grandparent() {
  return <Parent><p>Static</p></Parent>;
}
function Parent({ children }) {
  return <MemoCard>{children}</MemoCard>;  // children stable
}
```

**Multiple "slot" props vs children:**

```tsx
// Single children — composition
<Card>
  <Header />
  <Content />
  <Footer />
</Card>

// Slots — explicit positioning
<Card
  header={<Header />}
  content={<Content />}
  footer={<Footer />}
/>
```

Pros/cons:
- children: simpler, less boilerplate
- Slots: explicit, named positions, type-safe order

</details>

### Edge Cases

- **`children` as function**: Not standard JSX (transpiled differently). Render prop pattern.
- **Multiple `children`**: Always array under hood.
- **`null` child**: Skipped.

### Follow-up savollar

- "JSX.Element vs ReactElement?" — JSX.Element is React.ReactElement<any, any>. Same in practice.
- "React.Children deprecated?" — No. Used for Compound Components. Modern alternative: Context.
- "Why children not in Props by default in TS?" — Explicit type. PropsWithChildren helper.

</details>

---

### 7. Spread attributes — qaytadan ko'rib chiqish [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`<Comp {...props} />` — JS spread syntax JSX'da. Object'ning barcha key-value'larini attribute sifatida pass qiladi. Use case: prop forwarding (wrapper komponentlar). Xavfli: barcha props (anglagan/anglamagan) DOM'ga yetishi (HTML invalid attribute warning), security (sensitive data leak), override conflicts.

### Kod misoli

```tsx
// Wrapper button
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary";
}

function Button({ variant = "primary", className = "", ...rest }: ButtonProps) {
  return (
    <button
      className={`btn btn-${variant} ${className}`}
      {...rest}  // forward all native button props
    />
  );
}

// Usage
<Button variant="secondary" onClick={save} disabled={loading}>
  Save
</Button>
// onClick, disabled forwarded to <button>

// ❌ Override risk
function Card({ ...props }) {
  return <div className="card" {...props} />;
  // If props.className → overrides "card"!
}

// ✅ Order — class always added
function Card({ className, ...props }) {
  return <div className={`card ${className ?? ""}`} {...props} />;
}

// ❌ Sensitive data leak
function Form({ user }: { user: User & { password: string } }) {
  return <form data-user={JSON.stringify(user)} />;
  // password in DOM!
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Spread + override priority:**

```tsx
// Spread first — props override defaults
<button className="default" {...props} />
// If props.className → wins

// Spread last — defaults override props
<button {...props} className="forced" />
// className forced
```

**Combining values:**

```tsx
function Wrapper({ className, style, ...rest }: Props) {
  return (
    <div
      className={`wrapper ${className ?? ""}`}  // combine
      style={{ ...defaultStyle, ...style }}     // merge
      {...rest}
    />
  );
}
```

**Filtering props:**

```tsx
// Custom prop "loading" — not for DOM
function Button({ loading, disabled, ...rest }: ButtonProps) {
  return (
    <button {...rest} disabled={loading || disabled}>
      {loading ? "..." : rest.children}
    </button>
  );
  // loading destructured, not passed to DOM
}
```

**`shouldForwardProp` (styled-components):**

```tsx
import styled from "styled-components";

const StyledButton = styled.button.withConfig({
  shouldForwardProp: (prop) => !["variant", "size"].includes(prop),
})<{ variant: string }>`
  /* styles */
`;
```

**`React.ComponentProps<T>`:**

```tsx
import type { ComponentProps } from "react";

type ButtonProps = ComponentProps<"button"> & {
  variant: string;
};

// All native <button> props plus variant
```

**Polymorphic component pattern:**

```tsx
type Props<T extends React.ElementType> = {
  as?: T;
} & React.ComponentPropsWithoutRef<T>;

function Box<T extends React.ElementType = "div">({
  as,
  ...rest
}: Props<T>) {
  const Component = as || "div";
  return <Component {...rest} />;
}

<Box as="section" id="hero">Section</Box>
<Box as="a" href="/about">Link</Box>
```

**Native event handler types:**

```tsx
function Form(props: ComponentProps<"form">) {
  return <form {...props} />;
}

<Form onSubmit={(e) => e.preventDefault()}>...</Form>
// e: FormEvent<HTMLFormElement> — type-safe
```

**Ref forwarding (R19):**

```tsx
function Input({ ref, ...rest }: ComponentProps<"input"> & { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...rest} />;
}
```

**Forward + transform:**

```tsx
function ConfirmButton({ onClick, ...rest }: ButtonProps) {
  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    if (window.confirm("Are you sure?")) {
      onClick?.(e);
    }
  };
  return <button {...rest} onClick={handleClick} />;
}
```

**`as` cast for unknown props:**

```tsx
type Props = Record<string, any>;

function Generic({ ...props }: Props) {
  return <div {...props} />;
  // Anything goes (loose typing)
}
```

</details>

### Edge Cases

- **Spread with `key`**: Key extracted before spread. `<Item key={id} {...props} />` works.
- **Spread + ref**: R18 — special prop. R19 — regular prop, in spread.
- **Spread overriding event handlers**: Order matters.

### Follow-up savollar

- "Spread `aria-*`?" — kebab-case preserved. `<button aria-label="Save" />` from spread.
- "Spread DOM attribute warnings?" — `shouldForwardProp` pattern, explicit destructure.
- "Performance impact?" — Object spread allocates new object. Negligible for typical use.

</details>

---

### 8. Polymorphic components — `as` prop pattern [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Polymorphic component** — `as` prop bilan render qilingan element type'ni o'zgartirish (`<Box as="section">`, `<Box as="a">`). TypeScript'da generic — `<E extends ElementType>` bilan element-specific props inference. Use case: design system primitives (Box, Text), reusable layouts.

### Kod misoli

```tsx
import type { ElementType, ComponentPropsWithoutRef, Ref } from "react";

type BoxProps<E extends ElementType> = {
  as?: E;
  ref?: Ref<E extends keyof HTMLElementTagNameMap ? HTMLElementTagNameMap[E] : any>;
} & Omit<ComponentPropsWithoutRef<E>, "as" | "ref">;

function Box<E extends ElementType = "div">({
  as,
  ref,
  ...props
}: BoxProps<E>) {
  const Component = (as || "div") as ElementType;
  return <Component ref={ref} {...props} />;
}

// Usage — type-safe per element
<Box as="div" id="hero" />
<Box as="section" aria-label="Hero" />
<Box as="a" href="/about">Link</Box>
<Box as="button" onClick={handleClick}>Click</Box>

// Without "as" — defaults to "div"
<Box className="container">...</Box>
```

```tsx
// Custom component as polymorphic target
function MyComponent({ x }: { x: number }) {
  return <p>{x}</p>;
}

<Box as={MyComponent} x={42} />
```

<details>
<summary><strong>Deep Dive</strong></summary>

**TypeScript pattern:**

```typescript
// ElementType — string ('div') or component type (function/class)
type ElementType<P = any> =
  | { [K in keyof JSX.IntrinsicElements]: P extends JSX.IntrinsicElements[K] ? K : never }[keyof JSX.IntrinsicElements]
  | ComponentType<P>;
```

**`ComponentPropsWithoutRef<T>`:**

```typescript
// Native element
ComponentProps<"button">  // includes ref
ComponentPropsWithoutRef<"button">  // without ref

// Custom component
type MyProps = ComponentPropsWithRef<typeof MyButton>;
```

**Props inference per element:**

```tsx
<Box as="a" href="/about" />
// TS knows: as="a" → href required (or optional anchor prop)

<Box as="div" href="/about" />
// TS error: href doesn't exist on div

<Box as="button" disabled />
// TS knows: button has disabled

<Box as="div" disabled />
// TS error: disabled not on div
```

**Real-world libraries:**

- **Radix UI** — Polymorphic primitives
- **Stitches**, **vanilla-extract** — CSS-in-JS with polymorphic
- **Headless UI** — `as` prop everywhere

**Implementation patterns:**

```tsx
// Pattern 1: Simple polymorphic
function Box<E extends ElementType = "div">(
  props: { as?: E } & React.ComponentPropsWithoutRef<E>
) {
  const { as, ...rest } = props;
  const Component = as || "div";
  return <Component {...rest} />;
}

// Pattern 2: With ref forwarding (R19)
function Box<E extends ElementType = "div">(
  props: { as?: E; ref?: PolymorphicRef<E> } & React.ComponentPropsWithoutRef<E>
) {
  const { as, ref, ...rest } = props;
  const Component = (as || "div") as ElementType;
  return <Component ref={ref} {...rest} />;
}

// Pattern 3: With slot composition (Radix-style)
function Slot({ children, ...props }: SlotProps) {
  if (isValidElement(children)) {
    return cloneElement(children, { ...props, ...children.props });
  }
  return null;
}
```

**Restrictions:**

```tsx
// ❌ Type can't be too generic
<Box as={undefined} />  // TS error or fallback to default

// ❌ Multiple type collisions
<Box as="a" href="/about" disabled />  // disabled not on <a>
```

**Performance:**

```tsx
// Each render: as resolved
<Box as="div" />
// → Component = "div" (string)
// → React renders <div>
// Same as <div /> directly — no overhead
```

**Library implementations:**

```tsx
// Radix UI Polymorphic.Component
import { Polymorphic } from "@radix-ui/react-polymorphic";

const MyComponent = forwardRef<MyComponentElement, MyComponentOwnProps>(
  function MyComponent({ as: Tag = "div", ...rest }, ref) {
    return <Tag ref={ref} {...rest} />;
  }
) as Polymorphic.ForwardRefComponent<"div", MyComponentOwnProps>;
```

**`asChild` pattern (Radix):**

```tsx
// Use child as actual element (no wrapper)
<Button asChild>
  <a href="/about">Custom child</a>
</Button>
// Renders <a> with Button's behavior, no wrapper
```

**Common anti-pattern:**

```tsx
// ❌ Loose any
function Box(props: any) {
  const Tag = props.as || "div";
  return <Tag {...props} />;
}
// No type safety
```

**TypeScript performance:**

Generic polymorphic components — heavy on TS compiler (combinatorial type expansion). Real-world impact:
- 1-2 polymorphic components: fine
- 10+ levels nested: TS slow
- Workaround: simpler typing, opt-out generics

</details>

### Edge Cases

- **`as={null}`**: Falls back to default.
- **`as` as user component**: Pass component reference. Component receives all props.
- **`as` with refs**: TS may struggle. Use `ComponentPropsWithRef`.

### Follow-up savollar

- "Polymorphic vs styled component variants?" — Polymorphic flexible, runtime element. Variants compile-time, fixed.
- "asChild vs as?" — asChild renders child as element (no wrapper). as renders specified element.
- "Performance penalty?" — Runtime: minimal. Compile-time TS: noticeable for many uses.

</details>

---

### 9. Generic components — `<List<T>>` pattern [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Generic component** — TypeScript generics bilan komponent. Type'lar consumer'ga moslashadi. Common: `<List<T>>` — items va render function generic'siyada bog'liq. JSX'da generic syntax: `<List<User> items={users} ... />`.

### Kod misoli

```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor?: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item, i) => (
        <li key={keyExtractor ? keyExtractor(item) : i}>
          {renderItem(item)}
        </li>
      ))}
    </ul>
  );
}

// Usage with type inference
const users: User[] = [{ id: "1", name: "Ali" }];

<List
  items={users}
  renderItem={(user) => <span>{user.name}</span>}  // user: User inferred
  keyExtractor={(user) => user.id}
/>

// Explicit generic
<List<User>
  items={users}
  renderItem={(user) => <span>{user.name}</span>}
/>
```

```tsx
// Form generic
interface FormProps<T> {
  initial: T;
  onSubmit: (values: T) => void;
  render: (props: { values: T; setField: <K extends keyof T>(key: K, value: T[K]) => void }) => React.ReactNode;
}

function Form<T>({ initial, onSubmit, render }: FormProps<T>) {
  const [values, setValues] = useState<T>(initial);
  const setField = <K extends keyof T>(key: K, value: T[K]) => {
    setValues(prev => ({ ...prev, [key]: value }));
  };

  return (
    <form onSubmit={(e) => { e.preventDefault(); onSubmit(values); }}>
      {render({ values, setField })}
    </form>
  );
}

// Usage
<Form
  initial={{ name: "", email: "" }}
  onSubmit={(data) => console.log(data)}
  render={({ values, setField }) => (
    <>
      <input value={values.name} onChange={(e) => setField("name", e.target.value)} />
      <input value={values.email} onChange={(e) => setField("email", e.target.value)} />
    </>
  )}
/>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Generic naming convention:**

| Generic | Meaning |
|---------|---------|
| `T` | General data type |
| `P` | Component props |
| `E` | HTML element type |
| `R` | Ref type |
| `K`, `V` | Key, value (Record) |
| `T1`, `T2` | Multiple generics |

**Constrained generics:**

```tsx
function Select<T extends { id: string; label: string }>({
  items,
}: {
  items: T[];
}) {
  return <select>{items.map(i => <option value={i.id}>{i.label}</option>)}</select>;
}

// T must have id and label
<Select items={[{ id: "1", label: "A" }]} />  // ✅
<Select items={[{ id: "1" }]} />  // ❌ missing label
```

**Default generic:**

```tsx
function List<T = string>({ items }: { items: T[] }) { ... }

<List items={["a", "b"]} />  // T = string (default)
<List<number> items={[1, 2]} />  // T = number (explicit)
```

**Generic with `forwardRef` (R18):**

```tsx
import { forwardRef } from "react";

interface ListProps<T> {
  items: T[];
}

// ❌ forwardRef erases generic
const List = forwardRef<HTMLUListElement, ListProps<???>>(...);  // can't fit generic

// Workaround — cast after definition
function ListInner<T>(props: ListProps<T>, ref: React.Ref<HTMLUListElement>) {
  return <ul ref={ref}>...</ul>;
}

const List = forwardRef(ListInner) as <T>(
  props: ListProps<T> & { ref?: React.Ref<HTMLUListElement> }
) => React.ReactElement;
```

**R19 — ref as prop simpler:**

```tsx
function List<T>({ items, ref }: ListProps<T> & { ref?: React.Ref<HTMLUListElement> }) {
  return <ul ref={ref}>...</ul>;
}

<List<User> items={users} ref={listRef} />
```

**Generic state types:**

```tsx
function useTuple<A, B>(initial: [A, B]) {
  const [tuple, setTuple] = useState<[A, B]>(initial);
  return [tuple, setTuple] as const;
}

const [pair, setPair] = useTuple([1, "hello"]);  // [number, string]
```

**Generic Context:**

```tsx
function createGenericContext<T>() {
  const Context = createContext<T | undefined>(undefined);

  const Provider = Context.Provider;

  function useContext() {
    const ctx = React.useContext(Context);
    if (!ctx) throw new Error("Provider missing");
    return ctx;
  }

  return [Provider, useContext] as const;
}

const [UserProvider, useUser] = createGenericContext<User>();
```

**Common pattern: data table:**

```tsx
interface Column<T> {
  key: keyof T;
  label: string;
  render?: (value: T[keyof T], row: T) => React.ReactNode;
}

interface TableProps<T> {
  data: T[];
  columns: Column<T>[];
  onRowClick?: (row: T) => void;
}

function Table<T>({ data, columns, onRowClick }: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map(col => <th key={String(col.key)}>{col.label}</th>)}
        </tr>
      </thead>
      <tbody>
        {data.map((row, i) => (
          <tr key={i} onClick={() => onRowClick?.(row)}>
            {columns.map(col => (
              <td key={String(col.key)}>
                {col.render ? col.render(row[col.key], row) : String(row[col.key])}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Usage
<Table<User>
  data={users}
  columns={[
    { key: "name", label: "Name" },
    { key: "age", label: "Age", render: (age) => <strong>{age}</strong> },
  ]}
  onRowClick={(user) => console.log(user)}
/>
```

**TypeScript inference:**

```tsx
// Automatic inference from items
<List items={users} renderItem={(u) => u.name} />
// T inferred as User (from items: User[])
// renderItem param: u: User

// Explicit when inference fails
<List<User> items={users} renderItem={(u) => u.name} />
```

**Limitations:**

- Generic components — verbose JSX syntax (`<List<User> ...>`)
- Some TypeScript versions have inference quirks
- Complex constraints — slow TS compilation

**`Component<P>` vs Generic:**

```tsx
// Class with generic
class GenericList<T> extends React.Component<{ items: T[] }> {
  render() { ... }
}

<GenericList<User> items={users} />
```

</details>

### Edge Cases

- **Inferred generic**: TS infers from arguments. Explicit when ambiguous.
- **Nested generics**: TS may fail. Simplify or split.
- **`<T>` vs `<T,>` ambiguity**: TSX may treat `<T>` as JSX. Use `<T,>` (trailing comma) or `extends`.

### Follow-up savollar

- "Generic with default?" — `<T = string>` — fallback when not specified.
- "Inference vs explicit?" — Inference cleaner. Explicit when complex or ambiguous.
- "Generic forwardRef?" — Workaround with cast (see code).

</details>

---

### 10. Discriminated union props — variant components [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Discriminated union — TypeScript pattern: bir necha variant'lar `kind` (yoki `type`) field bilan ajratiladi. Component prop'larida — variant'lar har xil prop set'iga ega. TypeScript exhaustive narrowing ishlaydi: variant tekshirilganda boshqa props faqat shu variant ichida access qilinadi.

### Kod misoli

```tsx
type ButtonProps =
  | { variant: "primary"; label: string; onClick: () => void }
  | { variant: "link"; label: string; href: string }
  | { variant: "icon"; icon: React.ReactNode; ariaLabel: string };

function Button(props: ButtonProps) {
  switch (props.variant) {
    case "primary":
      return <button onClick={props.onClick}>{props.label}</button>;
    case "link":
      return <a href={props.href}>{props.label}</a>;
    case "icon":
      return <button aria-label={props.ariaLabel}>{props.icon}</button>;
  }
}

// Usage
<Button variant="primary" label="Save" onClick={save} />
<Button variant="link" label="About" href="/about" />
<Button variant="icon" icon={<Heart />} ariaLabel="Like" />

// ❌ TypeScript error — wrong props for variant
<Button variant="link" label="X" onClick={x} />  // missing href, extra onClick
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Exhaustive switch:**

```tsx
function Button(props: ButtonProps) {
  switch (props.variant) {
    case "primary": return <button onClick={props.onClick}>{props.label}</button>;
    case "link": return <a href={props.href}>{props.label}</a>;
    case "icon": return <button>{props.icon}</button>;
    default: {
      const _exhaustive: never = props;  // ✅ Compile error if case missed
      return null;
    }
  }
}
```

**With shared props:**

```tsx
type BaseProps = { className?: string; testId?: string };
type ButtonProps = BaseProps & (
  | { variant: "primary"; label: string }
  | { variant: "link"; href: string }
);
```

</details>

### Edge Cases

- **Optional discriminator**: `variant?: "x" | "y"` — TypeScript can't narrow without explicit check.
- **Boolean discriminator**: `isLink: true | false` — also works.

### Follow-up savollar

- "Exhaustive switch default case?" — `never` type assertion — compile-time xato agar variant qo'shilsa lekin handle qilinmasa.
- "Performance impact?" — Zero runtime — types erased. Discriminator check standard JS `switch`/`if`.
- "Multiple discriminators?" — Bir necha field bo'yicha narrow qilish mumkin, lekin bitta discriminator (variant/type/kind) konventsiya.

</details>

---

<a id="qism-c"></a>

## QISM C: Composition

### 11. Composition vs Inheritance — React falsafasi [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React **inheritance** o'rniga **composition** tavsiya qiladi. Inheritance: `class Child extends Parent` — tightly coupled, brittle. **Composition**: komponentlar bir-biriga **wrap** qilinadi yoki **slots** orqali content inject qilinadi (children prop). Flexibility, reusability, "favor composition over inheritance" — engineering principle.

### Kod misoli

```tsx
// ❌ Inheritance approach (NOT React way)
class BaseDialog {
  render() { return <div className="dialog">...</div>; }
}

class ConfirmDialog extends BaseDialog {
  // override render?
}

// ✅ Composition
function Dialog({ children, title, onClose }: Props) {
  return (
    <div className="dialog">
      <h2>{title}</h2>
      <button onClick={onClose}>×</button>
      <div className="content">{children}</div>
    </div>
  );
}

function ConfirmDialog({ message, onConfirm, onClose }: Props) {
  return (
    <Dialog title="Confirm" onClose={onClose}>
      <p>{message}</p>
      <button onClick={onConfirm}>Confirm</button>
      <button onClick={onClose}>Cancel</button>
    </Dialog>
  );
}

function DeleteDialog({ onDelete, onClose }: Props) {
  return (
    <ConfirmDialog
      message="Delete this item?"
      onConfirm={onDelete}
      onClose={onClose}
    />
  );
}
```

```tsx
// Layout composition
function Page({ children }: { children: ReactNode }) {
  return (
    <>
      <Header />
      <main>{children}</main>
      <Footer />
    </>
  );
}

function HomePage() {
  return (
    <Page>
      <Hero />
      <Features />
    </Page>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why composition over inheritance:**

1. **Flexibility** — combine N components, not chain hierarchy
2. **No fragile base class** — changes to parent don't break children
3. **Explicit interfaces** — props/children clear, vs inherited methods hidden
4. **Easier testing** — components mockable
5. **Type safety** — generics + composition = type-safe primitives

**React docs (since 2014):**
> "We don't have any examples where we would recommend creating component inheritance hierarchies."

**Composition primitives:**

1. **`children` prop** — generic content
2. **Named slots** (`header`, `footer` props) — explicit positions
3. **Render props** — function children for parameterized rendering
4. **Custom hooks** — logic reuse without inheritance
5. **HOC** (legacy) — wrapping pattern (function-based)

**Slots example:**

```tsx
function SplitLayout({
  left,
  right,
  ratio = 0.5,
}: {
  left: ReactNode;
  right: ReactNode;
  ratio?: number;
}) {
  return (
    <div style={{ display: "grid", gridTemplateColumns: `${ratio}fr ${1 - ratio}fr` }}>
      <div>{left}</div>
      <div>{right}</div>
    </div>
  );
}

<SplitLayout
  left={<Sidebar />}
  right={<Content />}
/>
```

**Composition for variants:**

```tsx
// Inheritance — multiple subclasses
class PrimaryButton extends BaseButton { ... }
class SecondaryButton extends BaseButton { ... }
class IconButton extends BaseButton { ... }

// Composition — props
function Button({ variant, ...props }: { variant?: "primary" | "secondary" }) {
  return <button className={`btn btn-${variant}`} {...props} />;
}

<Button variant="primary">Save</Button>
<Button variant="secondary">Cancel</Button>
```

**`React.Component` extension — when:**

Hooks era — class extension only for error boundaries (until alternative). Otherwise: function components + composition.

**Anti-pattern: `extends Component` for behavior:**

```tsx
// ❌ Class hierarchy
class Loader extends Component {
  state = { loading: false };
  startLoading = () => this.setState({ loading: true });
}

class DataLoader extends Loader {
  // inherit loading state — tightly coupled
}

// ✅ Custom hook composition
function useLoading() {
  const [loading, setLoading] = useState(false);
  return { loading, setLoading };
}

function DataComponent() {
  const { loading, setLoading } = useLoading();
  // ...
}
```

**Higher-order composition:**

```tsx
// Compose multiple wrappers
function App() {
  return (
    <ThemeProvider>
      <AuthProvider>
        <CartProvider>
          <Pages />
        </CartProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}

// vs single inherited "AppContext"
```

**Specialization vs configuration:**

```tsx
// Specialization (composition)
function ConfirmButton(props: ButtonProps) {
  return <Button {...props} variant="warning" />;
}

// Configuration (props)
<Button variant="warning">Confirm</Button>
```

Both valid. Specialization — fewer props, clearer intent.

**Render prop composition:**

```tsx
function MouseTracker({ render }: { render: (pos: { x: number; y: number }) => ReactNode }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  // ...
  return <>{render(pos)}</>;
}

<MouseTracker render={(pos) => <p>Mouse: {pos.x}, {pos.y}</p>} />
```

**Inheritance edge case — error boundaries:**

```tsx
class ErrorBoundary extends React.Component<Props, { hasError: boolean }> {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  render() {
    if (this.state.hasError) return <p>Error</p>;
    return this.props.children;
  }
}
```

Class still required for error boundaries. Hooks alternative — `react-error-boundary` library (wraps class internally).

</details>

### Edge Cases

- **Inheritance for libraries**: Some libs use class hierarchies internally (Three.js wrappers). React-side composition.
- **Mixin pattern**: Deprecated in React. Custom hooks replace.
- **Multiple inheritance**: Not supported in JS classes. Composition handles.

### Follow-up savollar

- "When inheritance OK?" — Almost never in React. Error boundaries class only.
- "Custom hooks vs HOC?" — Custom hooks preferred (R16.8+). HOC legacy.
- "Tightly coupled components?" — Compound components pattern (Q23).

</details>

---

### 12. Slots — named children pattern [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Slots** — komponent multiple **named ReactNode props** qabul qilib, har birini aniq pozitsiyada render qilish. children'ning "named version". Vue's `<slot>` analog. Use case: layout components (header/main/footer), card components (image/title/actions), modal (title/content/footer).

### Kod misoli

```tsx
interface CardProps {
  image?: ReactNode;
  title: ReactNode;
  description?: ReactNode;
  actions?: ReactNode;
}

function Card({ image, title, description, actions }: CardProps) {
  return (
    <article className="card">
      {image && <div className="card-image">{image}</div>}
      <header className="card-header">{title}</header>
      {description && <div className="card-body">{description}</div>}
      {actions && <footer className="card-footer">{actions}</footer>}
    </article>
  );
}

// Usage
<Card
  image={<img src="/photo.jpg" alt="" />}
  title={<h2>Product Name</h2>}
  description={<p>Lorem ipsum...</p>}
  actions={
    <>
      <button>Buy</button>
      <button>Save</button>
    </>
  }
/>
```

```tsx
// Modal slots
interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  header?: ReactNode;
  children: ReactNode;
  footer?: ReactNode;
}

function Modal({ isOpen, onClose, header, children, footer }: ModalProps) {
  if (!isOpen) return null;
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal" onClick={(e) => e.stopPropagation()}>
        {header && <header>{header}</header>}
        <main>{children}</main>
        {footer && <footer>{footer}</footer>}
      </div>
    </div>,
    document.body
  );
}

<Modal
  isOpen={open}
  onClose={() => setOpen(false)}
  header={<h2>Confirm</h2>}
  footer={
    <>
      <button onClick={confirm}>OK</button>
      <button onClick={() => setOpen(false)}>Cancel</button>
    </>
  }
>
  <p>Are you sure?</p>
</Modal>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Slots vs `children`:**

| Aspect | `children` | Named slots |
|--------|-----------|-------------|
| Flexibility | High (anything) | Limited (named) |
| Order | Caller controls | Component controls |
| Type safety | ReactNode | Per-slot types |
| Boilerplate | Less | More |

**When slots:**
- Specific positions (header/footer)
- Type-specific content (image vs text)
- Reusable layout

**When children:**
- Generic content
- Caller decides structure
- Composition flexibility

**`React.Children` for slot extraction (alternative):**

```tsx
import { Children, isValidElement } from "react";

function Card({ children }: { children: ReactNode }) {
  let header, body, footer;
  Children.forEach(children, (child) => {
    if (isValidElement(child)) {
      if (child.type === CardHeader) header = child;
      else if (child.type === CardBody) body = child;
      else if (child.type === CardFooter) footer = child;
    }
  });

  return (
    <article>
      {header}
      {body}
      {footer}
    </article>
  );
}

// Compound usage
<Card>
  <CardHeader>Title</CardHeader>
  <CardBody>Content</CardBody>
  <CardFooter>Actions</CardFooter>
</Card>
```

**Compound components (Q23) — slot variant:**

```tsx
function Card({ children }: { children: ReactNode }) {
  return <article className="card">{children}</article>;
}

Card.Header = function Header({ children }: { children: ReactNode }) {
  return <header className="card-header">{children}</header>;
};

Card.Body = function Body({ children }: { children: ReactNode }) {
  return <div className="card-body">{children}</div>;
};

<Card>
  <Card.Header>Title</Card.Header>
  <Card.Body>Content</Card.Body>
</Card>
```

Compound components — slots via component identity.

**Vue `<slot>` comparison:**

```vue
<!-- Vue -->
<template>
  <div class="card">
    <slot name="header"></slot>
    <slot></slot>  <!-- default -->
    <slot name="footer"></slot>
  </div>
</template>

<!-- Usage -->
<Card>
  <template v-slot:header>Title</template>
  <p>Default content</p>
  <template v-slot:footer>Actions</template>
</Card>
```

React: named props (more verbose). Vue: declarative slot syntax.

**TypeScript slot types:**

```tsx
interface DialogProps {
  // Each slot specific type
  trigger: React.ReactElement<{ onClick: () => void }>;
  title: string;
  body: ReactNode;
  actions?: ReactNode;
}
```

**Performance:**

Slots — same as regular props. ReactNode pass-through.

**Pattern: scoped slots:**

```tsx
// Vue scoped slot — pass data to slot
// React render prop pattern
interface ListProps<T> {
  items: T[];
  itemSlot: (item: T) => ReactNode;  // scoped slot
  emptySlot?: ReactNode;
}

function List<T>({ items, itemSlot, emptySlot }: ListProps<T>) {
  if (items.length === 0) return <>{emptySlot}</>;
  return <ul>{items.map((i, idx) => <li key={idx}>{itemSlot(i)}</li>)}</ul>;
}

<List
  items={users}
  itemSlot={(user) => <span>{user.name}</span>}
  emptySlot={<p>No users</p>}
/>
```

**`as` polymorphic prop with slots:**

```tsx
function Heading({ level = 1, children }: { level?: 1 | 2 | 3 | 4 | 5 | 6; children: ReactNode }) {
  const Tag = `h${level}` as `h${1 | 2 | 3 | 4 | 5 | 6}`;
  return <Tag>{children}</Tag>;
}
```

</details>

### Edge Cases

- **Slot null/undefined**: Skipped in render.
- **Slot order**: Component controls, not consumer.
- **Slot reuse**: Same prop value can be used multiple places (rare).

### Follow-up savollar

- "Slots vs compound components?" — Slots: explicit props, simpler. Compound: nested API, flexible.
- "Slot conditional rendering?" — `{slot && <wrapper>{slot}</wrapper>}`.
- "Multiple instances of same slot?" — Use array prop or compound.

</details>

---

### 13. Inversion of control — render takeover [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Inversion of control (IoC)** — komponent **render qarorini** parent'ga (caller) topshiradi. Library specifies behavior, caller specifies markup. Render props, function-as-children, compound components — IoC pattern'lari. Caller renderingni control qilib, library logic'ni reuse qiladi.

### Kod misoli

```tsx
// Library — Disclosure (logic)
function Disclosure({
  children,
}: {
  children: (open: boolean, toggle: () => void) => ReactNode;
}) {
  const [open, setOpen] = useState(false);
  const toggle = () => setOpen(o => !o);
  return <>{children(open, toggle)}</>;
}

// Caller — controls rendering
<Disclosure>
  {(open, toggle) => (
    <>
      <button onClick={toggle} aria-expanded={open}>
        {open ? "Hide" : "Show"} details
      </button>
      {open && (
        <div>
          <p>Hidden content</p>
        </div>
      )}
    </>
  )}
</Disclosure>

// Caller can render anything — accordion, dropdown, modal trigger
```

```tsx
// IoC for data fetching
function Fetch<T>({
  url,
  children,
}: {
  url: string;
  children: (state: { data: T | null; loading: boolean; error: Error | null }) => ReactNode;
}) {
  const { data, loading, error } = useFetch<T>(url);
  return <>{children({ data, loading, error })}</>;
}

<Fetch<User> url="/api/user/123">
  {({ data, loading, error }) => {
    if (loading) return <Spinner />;
    if (error) return <Error message={error.message} />;
    if (!data) return null;
    return <UserCard user={data} />;
  }}
</Fetch>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**IoC pattern variants:**

1. **Render prop** — `render` prop function
2. **Function as children** — `children` is function
3. **Component as prop** — `<List as={CustomItem} />`
4. **Compound components** — child components access parent state via context
5. **Custom hooks** — caller chooses how to use returned values

**Why IoC:**

- Library: handles logic (state, effects, data)
- Consumer: handles UI (markup, styling)
- Separation of concerns
- Reusable across UI patterns

**Render props pattern (legacy):**

```tsx
function Toggle({ render }: { render: (props: { on: boolean; toggle: () => void }) => ReactNode }) {
  const [on, setOn] = useState(false);
  return <>{render({ on, toggle: () => setOn(!on) })}</>;
}

<Toggle render={({ on, toggle }) => <button onClick={toggle}>{on ? "On" : "Off"}</button>} />
```

**Function as children:**

```tsx
<Toggle>{({ on, toggle }) => <button onClick={toggle}>{on ? "On" : "Off"}</button>}</Toggle>

// Same as render prop but uses `children` prop
```

**Custom hook (modern IoC):**

```tsx
function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn(o => !o), []);
  return { on, toggle };
}

// Caller — full control over markup
function MyComponent() {
  const { on, toggle } = useToggle();
  return <button onClick={toggle}>{on ? "On" : "Off"}</button>;
}
```

Hooks — IoC without component wrappers (cleaner).

**Compound components IoC:**

```tsx
function Tabs({ children, value, onChange }: Props) {
  return <TabsContext value={{ value, onChange }}>{children}</TabsContext>;
}

Tabs.List = function List({ children }) { return <div role="tablist">{children}</div>; };
Tabs.Tab = function Tab({ children, value }) {
  const ctx = useContext(TabsContext);
  return <button onClick={() => ctx.onChange(value)}>{children}</button>;
};
Tabs.Panel = function Panel({ children, value }) {
  const ctx = useContext(TabsContext);
  return ctx.value === value ? <div>{children}</div> : null;
};

// Caller — full control over structure
<Tabs value={tab} onChange={setTab}>
  <Tabs.List>
    <Tabs.Tab value="a">A</Tabs.Tab>
    <Tabs.Tab value="b">B</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="a">Panel A</Tabs.Panel>
  <Tabs.Panel value="b">Panel B</Tabs.Panel>
</Tabs>
```

**Pros of IoC:**

- Maximum flexibility
- Reusable across designs
- Logic decoupled from rendering
- Type-safe (TS narrows callbacks)

**Cons:**

- Boilerplate (render functions, context)
- Verbose JSX
- Steeper learning curve

**Modern preference: hooks > render props:**

```tsx
// Old (render props)
<Mouse render={({ x, y }) => <p>{x},{y}</p>} />

// Modern (hooks)
const { x, y } = useMouse();
return <p>{x},{y}</p>;
```

Hooks more concise, no wrapper component.

**When render props still useful:**

- Data layer with complex render UI: `<Query>{(data) => <UI data={data} />}</Query>`
- Library targeting non-hook codebases
- Components needing children-relative state (Disclosure, Combobox)

**`children` as function — TypeScript:**

```tsx
interface DisclosureProps {
  children: (open: boolean, toggle: () => void) => React.ReactNode;
}
```

**Composition with IoC:**

```tsx
<DataProvider source={api}>
  {(data) => (
    <Layout>
      <Sidebar items={data.items} />
      <Main content={data.content} />
    </Layout>
  )}
</DataProvider>
```

Combine multiple IoC layers.

</details>

### Edge Cases

- **Render prop returns nothing**: Returns `null` or `undefined` — skipped.
- **Multiple render props**: `<Comp render={x} fallback={y} />` — multiple slots.
- **Render prop performance**: Function recreated each render. Memoize if needed.

### Follow-up savollar

- "When IoC vs hooks?" — Hooks for logic-only (no wrapper). IoC for component structure (Disclosure, Combobox).
- "Render props in 2026?" — Less common (hooks dominate). Compound components preferred for component IoC.
- "Performance overhead?" — Function call per render. Negligible.

</details>

---

<a id="qism-d"></a>

## QISM D: State Lifting & Controlled

### 14. Lifting state up — sibling komponentlar state share qilolmaydi [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Sibling components** state'ni bir-biriga to'g'ridan-to'g'ri pass qilolmaydi (one-way data flow). **Lifting state up** — shared state'ni **eng yaqin umumiy parent**'ga ko'chirish, props orqali siblings'ga pass qilish. State management foundation, pre-Context era pattern. Hozirgi modern: Context, external store (Zustand/Redux) bilan kombinatsiya.

### Kod misoli

```tsx
// ❌ State in child — sibling can't access
function Parent() {
  return (
    <>
      <Input />  {/* state inside */}
      <Display />  {/* needs state from Input — can't access! */}
    </>
  );
}

// ✅ Lift state up
function Parent() {
  const [text, setText] = useState("");

  return (
    <>
      <Input value={text} onChange={setText} />
      <Display text={text} />
    </>
  );
}

function Input({ value, onChange }: { value: string; onChange: (v: string) => void }) {
  return <input value={value} onChange={(e) => onChange(e.target.value)} />;
}

function Display({ text }: { text: string }) {
  return <p>You typed: {text}</p>;
}
```

```tsx
// Multi-level lifting — temperature converter
function TemperatureConverter() {
  const [celsius, setCelsius] = useState(0);

  const fahrenheit = (celsius * 9) / 5 + 32;
  const setFahrenheit = (f: number) => setCelsius(((f - 32) * 5) / 9);

  return (
    <>
      <TemperatureInput
        scale="C"
        value={celsius}
        onChange={setCelsius}
      />
      <TemperatureInput
        scale="F"
        value={fahrenheit}
        onChange={setFahrenheit}
      />
    </>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**One-way data flow recap:**

```
Parent (state owner)
  ├─ ChildA (props down, events up)
  └─ ChildB (props down, events up)

ChildA → ChildB: NOT directly
ChildA → Parent (callback) → ChildB (props): VIA PARENT
```

**Lifting decision tree:**

```
Need to share state?
├── YES
│   ├── Few components, shallow tree → Lift to common parent
│   ├── Deep tree → Context
│   ├── Many components, complex → External store (Zustand, Redux)
│   └── Server state → TanStack Query, SWR
└── NO
    └── Local state in component
```

**Lifting cost:**

```tsx
// State at root, pass props deeply
function App() {
  const [user, setUser] = useState(null);
  return <Layout user={user} setUser={setUser} />;
}

function Layout({ user, setUser }) {
  return <Header user={user} setUser={setUser} />;
}

function Header({ user, setUser }) {
  return <UserBadge user={user} setUser={setUser} />;
}

function UserBadge({ user, setUser }) {
  return <span>{user.name}</span>;
}

// "Prop drilling" — mid components forward props they don't use
// Solution: Context (Q24)
```

**When NOT to lift:**

- State only used in one component → keep local
- Lifting adds complexity > saves
- Data fits server/global state pattern → use external store

**Performance impact:**

```tsx
// Lifting state to App — App re-renders on every change
// All children re-render (memo helps)

// Local state — only that component re-renders
```

**`useReducer` for lifted state:**

```tsx
function App() {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <>
      <Sidebar state={state} dispatch={dispatch} />
      <Main state={state} dispatch={dispatch} />
    </>
  );
}

// Reducer pattern fits lifted complex state
```

**Co-location vs lifting:**

```tsx
// Co-locate — state near consumer
function ItemList({ items }) {
  return items.map(item => <ItemRow key={item.id} item={item} />);
}

function ItemRow({ item }) {
  const [expanded, setExpanded] = useState(false);  // local, per row
  return <div onClick={() => setExpanded(!expanded)}>{item.name}</div>;
}

// vs lifted — single state for all rows
function ItemList({ items }) {
  const [expandedIds, setExpandedIds] = useState<Set<string>>(new Set());
  // ... manage expanded set globally
}
```

Co-locate when each instance has independent state.

**Modern alternatives to lifting:**

```tsx
// 1. Context
const StateContext = createContext(...);

function App() {
  const [state, setState] = useState(...);
  return (
    <StateContext value={{ state, setState }}>
      <Tree />
    </StateContext>
  );
}

// 2. External store (Zustand)
const useStore = create((set) => ({ user: null, setUser: (u) => set({ user: u }) }));
// Any component can subscribe — no lifting

// 3. Query library (TanStack Query)
const { data: user } = useQuery({ queryKey: ["user"], queryFn: fetchUser });
// Server state — no lifting needed
```

**Lifting + Context combo:**

```tsx
// Lift state to provider component, expose via context
function AppStateProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, initial);
  return (
    <StateContext value={{ state, dispatch }}>
      {children}
    </StateContext>
  );
}

function App() {
  return (
    <AppStateProvider>
      <Tree />  {/* Any child accesses state via context */}
    </AppStateProvider>
  );
}
```

**Pattern: state machine lifted:**

```tsx
function App() {
  const [machine, send] = useMachine(appStateMachine);

  return (
    <>
      <Sidebar machine={machine} send={send} />
      <Main machine={machine} send={send} />
    </>
  );
}
```

XState + lift state up.

</details>

### Edge Cases

- **State both parent and child have**: Anti-pattern. One source of truth.
- **Sync state between parent props and child state**: Use key for reset, or controlled component.
- **Lift state across non-parent boundary**: Need refactor — find common parent.

### Follow-up savollar

- "Always lift to root?" — No. Lift to nearest common parent.
- "Lifting hurts performance?" — Yes — parent re-renders, all children too. Memo and split components.
- "Alternative without lifting?" — Context, external store, server state library.

</details>

---

### 15. Controlled vs Uncontrolled inputs [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Controlled** — React state input value'ni boshqaradi (`value` + `onChange`). **Uncontrolled** — DOM o'zi value'ni saqlaydi (`defaultValue`, `useRef` bilan o'qish). Controlled — real-time UI sync, validation, formatting. Uncontrolled — performance, DOM-native, simpler for one-time read on submit. R19 form actions — uncontrolled-friendly.

### Kod misoli

```tsx
// Controlled
function ControlledForm() {
  const [name, setName] = useState("");

  return (
    <form>
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
      <p>Hello, {name}!</p>  {/* real-time */}
    </form>
  );
}

// Uncontrolled
function UncontrolledForm() {
  const inputRef = useRef<HTMLInputElement>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    console.log(inputRef.current?.value);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} defaultValue="Default" />
      <button>Submit</button>
    </form>
  );
}

// Hybrid — controlled with formatting
function PhoneInput() {
  const [phone, setPhone] = useState("");

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const digits = e.target.value.replace(/\D/g, "").slice(0, 10);
    const formatted = digits.replace(/(\d{3})(\d{3})(\d{4})/, "($1) $2-$3");
    setPhone(formatted);
  };

  return <input value={phone} onChange={handleChange} placeholder="(555) 123-4567" />;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Comparison:**

| Aspect | Controlled | Uncontrolled |
|--------|-----------|--------------|
| State owner | React | DOM |
| Initial value | `value` | `defaultValue` |
| Updates | `onChange` callback | DOM internally |
| Read | State variable | `ref.current.value` |
| Validation | Real-time | On submit |
| Re-render | Each keystroke | None |
| Forms libraries | Formik (controlled) | React Hook Form (uncontrolled) |

**`value={null}` issue:**

```tsx
const [value, setValue] = useState<string | null>(null);
<input value={value} />  // ⚠️ controlled-to-uncontrolled warning

// Fix
<input value={value ?? ""} />
```

**`readOnly` for intentional read-only:**

```tsx
<input value="X" readOnly />
// No warning, intentional
```

**Form types:**

| Element | Controlled props |
|---------|------------------|
| `<input type="text">` | `value` + `onChange` |
| `<input type="checkbox">` | `checked` + `onChange` |
| `<input type="radio">` | `checked` + `onChange` |
| `<input type="file">` | Uncontrolled (security) |
| `<textarea>` | `value` + `onChange` |
| `<select>` | `value` + `onChange` |
| `<select multiple>` | `value: string[]` + `onChange` |

**File input — always uncontrolled:**

```tsx
const fileInputRef = useRef<HTMLInputElement>(null);

const handleSubmit = () => {
  const file = fileInputRef.current?.files?.[0];
  if (file) uploadFile(file);
};

<input type="file" ref={fileInputRef} />
```

Browser security — JS can't set `<input type="file">.value` (only user can).

**Performance:**

```tsx
// Controlled — re-render every keystroke
const [value, setValue] = useState("");
<input value={value} onChange={(e) => setValue(e.target.value)} />

// Uncontrolled — no re-render
const ref = useRef<HTMLInputElement>(null);
<input ref={ref} defaultValue="" />

// 1000-input form: controlled — 1000 components re-render per keystroke
// React Hook Form solves this with refs (uncontrolled-leaning)
```

**React Hook Form (uncontrolled-leaning):**

```tsx
import { useForm } from "react-hook-form";

function Form() {
  const { register, handleSubmit } = useForm();

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("name")} />  {/* register uses refs */}
      <button>Submit</button>
    </form>
  );
}

// register returns { ref, onChange, onBlur, name } — uncontrolled with React-managed updates
```

**Formik (controlled):**

```tsx
<Formik initialValues={{ name: "" }} onSubmit={onSubmit}>
  <Field name="name" />  {/* controlled */}
</Formik>
```

**R19 form actions — uncontrolled-friendly:**

```tsx
function Form() {
  return (
    <form action={async (formData) => {
      const name = formData.get("name") as string;
      await submit(name);
    }}>
      <input name="name" defaultValue="Initial" />
      <button>Submit</button>
    </form>
  );
}
// React reads form on submit — uncontrolled-style
```

**Decision guide:**

```
Need real-time UI based on input?
├── YES → Controlled
│   ├── Validation on type
│   ├── Format on type
│   ├── Computed display
│   └── Conditional UI

Need just submit value?
├── YES → Uncontrolled
│   ├── Performance critical (large form)
│   ├── Simple form
│   └── R19 form action
```

**Mixed:**

```tsx
function HybridForm() {
  const nameRef = useRef<HTMLInputElement>(null);
  const [email, setEmail] = useState("");  // controlled

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    const name = nameRef.current?.value;
    submitForm({ name, email });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={nameRef} defaultValue="" />  {/* uncontrolled */}
      <input value={email} onChange={(e) => setEmail(e.target.value)} />  {/* controlled */}
      {email && <p>Email: {email}</p>}  {/* real-time UI */}
    </form>
  );
}
```

**Common bug: changing `value` prop without `onChange`:**

```tsx
function Component() {
  const [editable, setEditable] = useState(false);

  return (
    <input
      value="locked"  // ❌ no onChange — input frozen
      onChange={editable ? handleChange : undefined}
    />
  );
}
// Warning: "You provided a value prop to a form field without an onChange handler."
```

</details>

### Edge Cases

- **`<select>` `defaultValue`**: Use `defaultValue` on `<select>`, not `selected` on `<option>`.
- **Multiple radio buttons**: `checked` based on group state.
- **`onChange` for checkbox**: `e.target.checked`, not `e.target.value`.

### Follow-up savollar

- "Performance: many controlled inputs?" — React Hook Form (uncontrolled), or memoize.
- "Validation timing?" — Controlled: real-time. Uncontrolled: on submit. Hybrid: blur events.
- "R19 form action with controlled?" — Possible. `<form action>` works with both. Server action receives FormData regardless.

</details>

---

### 16. State management decision tree [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Decision**: (1) **Local state** (`useState`) — single component, (2) **Lift state** — siblings/ancestors share, (3) **Context** — deep tree, infrequent changes (theme, user), (4) **External store** (Zustand, Redux, Jotai) — global, frequent updates, complex, (5) **Server state** (TanStack Query, SWR) — server data caching/syncing, (6) **URL state** (React Router) — shareable, navigable.

### Decision tree

```
What kind of state?
├── UI / form values
│   ├── Single component → useState
│   ├── Siblings → lift to parent
│   ├── Deep tree (theme, locale) → Context
│   └── Many consumers, frequent → Zustand/Jotai
├── Server data
│   ├── Read → TanStack Query / SWR
│   └── Write/mutation → Same library
├── URL state (filters, pagination)
│   └── React Router / Next.js searchParams
└── Form state
    ├── Simple → useState
    ├── Complex → React Hook Form / Formik
    └── R19 → useActionState
```

### Kod misoli

```tsx
// 1. Local state
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// 2. Lifted state
function Parent() {
  const [filter, setFilter] = useState("");
  return (
    <>
      <FilterInput value={filter} onChange={setFilter} />
      <FilteredList filter={filter} />
    </>
  );
}

// 3. Context (theme)
const ThemeContext = createContext("light");

function App() {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeContext value={theme}>
      <Pages />
    </ThemeContext>
  );
}

// 4. External store (Zustand)
import { create } from "zustand";

interface CartState {
  items: Item[];
  addItem: (item: Item) => void;
}

const useCart = create<CartState>((set) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
}));

function Cart() {
  const items = useCart(s => s.items);
  return <p>Items: {items.length}</p>;
}

// 5. Server state (TanStack Query)
import { useQuery } from "@tanstack/react-query";

function UserProfile({ id }: { id: string }) {
  const { data, isPending, error } = useQuery({
    queryKey: ["user", id],
    queryFn: () => api.getUser(id),
  });

  if (isPending) return <Spinner />;
  if (error) return <Error />;
  return <div>{data.name}</div>;
}

// 6. URL state (Next.js)
"use client";
import { useSearchParams, useRouter } from "next/navigation";

function ProductFilter() {
  const params = useSearchParams();
  const router = useRouter();

  const category = params.get("category") ?? "all";

  return (
    <select
      value={category}
      onChange={(e) => router.push(`?category=${e.target.value}`)}
    >
      <option value="all">All</option>
      <option value="phones">Phones</option>
    </select>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Library comparison:**

| Library | Use case | Strengths | Weaknesses |
|---------|----------|-----------|------------|
| useState | Local | Built-in, simple | Local only |
| useReducer | Complex local | Centralized logic | Local only |
| Context | Theme, user, deps injection | Built-in, no library | Re-render all consumers |
| Redux | Large apps, undo, time-travel | DevTools, ecosystem | Boilerplate |
| Redux Toolkit | Modern Redux | Less boilerplate | Still some |
| Zustand | Medium apps | Simple, fast | Smaller ecosystem |
| Jotai | Atomic state | Fine-grained | Conceptually different |
| Recoil | Atomic state (Meta) | Async support | Archived/unmaintained (2024+), Jotai alternative |
| TanStack Query | Server state | Caching, retry, dedupe | Server-focused |
| SWR | Server state | Smaller, simpler | Less features |
| React Hook Form | Forms | Performance | Form-only |
| Formik | Forms | Mature | Performance |

**Server state characteristics:**

- Async (network)
- Caching needed
- Retry on failure
- Refetch on focus
- Optimistic updates
- Pagination/infinite scroll

→ Server state libraries (TanStack Query, SWR).

**Client state characteristics:**

- Sync
- Immediate
- No caching needed
- UI/form values

→ useState, Context, Zustand.

**Mixing patterns:**

```tsx
// Common modern app
function App() {
  return (
    <QueryClientProvider client={queryClient}>  {/* server state */}
      <ThemeProvider>  {/* context — UI */}
        <ZustandStoreProvider>  {/* global UI state */}
          <Router>
            <Pages />  {/* URL state */}
          </Router>
        </ZustandStoreProvider>
      </ThemeProvider>
    </QueryClientProvider>
  );
}
```

**Anti-patterns:**

```tsx
// ❌ Server data in Redux/Zustand
const useStore = create(() => ({
  user: null,
}));

useEffect(() => {
  api.getUser().then(user => useStore.setState({ user }));
}, []);

// ✅ TanStack Query handles caching, deduplication, refetch
const { data: user } = useQuery({ queryKey: ["user"], queryFn: api.getUser });
```

```tsx
// ❌ Lift state too high
function App() {
  const [hover, setHover] = useState(false);  // ❌ hover specific to one button
  return <Tree />;
}

// ✅ Co-locate
function HoverButton() {
  const [hover, setHover] = useState(false);
  return <button onMouseEnter={...}>Hover</button>;
}
```

```tsx
// ❌ Everything in Context
const AppContext = createContext({ user, theme, cart, settings, ... });

// ✅ Split contexts
const UserContext = createContext(null);
const ThemeContext = createContext("light");
// or use Zustand for granular subscriptions
```

**Performance considerations:**

```tsx
// useState — local, fast
// Context — all consumers re-render on value change
// Zustand — selector-based, only subscribers re-render
// Redux — connect/useSelector with shallow equal
// TanStack Query — query-based subscriptions

// Choose based on update frequency and consumer count
```

**State machines (XState, useReducer + state machine):**

```tsx
import { useMachine } from "@xstate/react";

const fetchMachine = createMachine({
  id: "fetch",
  initial: "idle",
  states: {
    idle: { on: { FETCH: "loading" } },
    loading: { invoke: { src: "fetchData", onDone: "success", onError: "error" } },
    success: { on: { FETCH: "loading" } },
    error: { on: { RETRY: "loading" } },
  },
});

function Component() {
  const [state, send] = useMachine(fetchMachine);
  // ...
}
```

For complex flows with many transitions.

</details>

### Edge Cases

- **State synced server + client**: React Query + optimistic updates.
- **State persisted across reloads**: localStorage + custom hook, or Zustand persist.
- **State across tabs**: BroadcastChannel API or localStorage events.

### Follow-up savollar

- "Redux vs Zustand for new project?" — Zustand: simpler, less boilerplate. Redux Toolkit: ecosystem, devtools.
- "When Context gets too slow?" — Many fast-changing consumers. Migrate to Zustand or selector library.
- "Single store vs multiple?" — Domain-based separation. Auth, cart, settings — separate stores.

</details>

---

### 17. Form library patterns — controlled vs uncontrolled-leaning [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Form library'lar 2 ta yondashuv: **controlled** (Formik, react-final-form) — har keystroke state update, real-time validation. **Uncontrolled-leaning** (React Hook Form) — DOM owns state, ref-based read on submit, faster (no per-keystroke re-render). RHF — modern preferred (performance, less re-renders).

### Kod misoli

```tsx
// React Hook Form (uncontrolled-leaning)
import { useForm } from "react-hook-form";

interface LoginForm {
  email: string;
  password: string;
}

function LoginRHF() {
  const { register, handleSubmit, formState: { errors } } = useForm<LoginForm>();

  const onSubmit = (data: LoginForm) => {
    console.log(data);  // { email, password }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("email", { required: true, pattern: /^.+@.+$/ })} />
      {errors.email && <span>Email required and valid</span>}

      <input type="password" {...register("password", { minLength: 8 })} />
      {errors.password && <span>Password 8+ chars</span>}

      <button type="submit">Login</button>
    </form>
  );
}

// Formik (controlled)
import { Formik, Field, Form, ErrorMessage } from "formik";

function LoginFormik() {
  return (
    <Formik
      initialValues={{ email: "", password: "" }}
      validate={(values) => {
        const errors: any = {};
        if (!values.email) errors.email = "Required";
        return errors;
      }}
      onSubmit={(values) => console.log(values)}
    >
      <Form>
        <Field name="email" />
        <ErrorMessage name="email" />
        <Field type="password" name="password" />
        <ErrorMessage name="password" />
        <button type="submit">Login</button>
      </Form>
    </Formik>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**RHF performance:**

- Uncontrolled — input DOM owns value
- React re-render only on submit, error, watch'd field
- 1000 fields — single re-render on submit (vs Formik N re-renders)

**Validation:**

```tsx
// RHF + Zod
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

function Form() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(schema),
  });
  // ...
}
```

**Controlled benefits:**

- Real-time validation (as user types)
- Cross-field dependencies (one field changes another)
- Conditional fields based on input

**R19 form actions:**

```tsx
"use client";
import { useActionState } from "react";

function LoginR19() {
  const [state, action, isPending] = useActionState(async (prev, formData) => {
    const email = formData.get("email") as string;
    return await loginAction(email);
  }, { error: null });

  return (
    <form action={action}>
      <input name="email" />
      <button disabled={isPending}>Login</button>
      {state.error && <span>{state.error}</span>}
    </form>
  );
}
```

R19 form actions — uncontrolled-friendly (FormData), works with Server Actions.

</details>

### Edge Cases

- **Dynamic fields**: RHF `useFieldArray` — dynamic add/remove with index management.
- **File upload**: Both libraries — read `e.target.files`, special handling.
- **Multi-step wizard**: State management across steps — controlled state at parent, uncontrolled per step.

### Follow-up savollar

- "RHF vs Formik for new project?" — RHF tavsiya etiladi (faol development, yaxshiroq performance, kichikroq bundle). Formik'ning development tezligi sekinlashgan (oxirgi major release 2021).
- "Server Actions replace form libraries?" — Partially. Validation, complex UX still need library.

</details>

---

<a id="qism-e"></a>

## QISM E: Event Handling

### 18. Synthetic events — nima va nima uchun [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**SyntheticEvent** — React'ning **cross-browser normalized** event wrapper. Native DOM event'larni o'rab oladi, har browser'da consistent API beradi. `e.target`, `e.preventDefault()`, `e.stopPropagation()` — native'ga o'xshash. `e.nativeEvent` — original DOM event'ga access. R17+'da event pooling olib tashlandi (zamonaviy browser'larda GC yetarli samarali, pooling ortiqcha murakkablik qo'shar edi).

### To'liq tushuntirish

**Why SyntheticEvent:**

- Browser event API normalization (tarixiy sabab, hozir kam relevant lekin architecture saqlanib qolgan)
- Performance — pooling (R16-R17, olib tashlangan)
- Type-safe (TS) — `MouseEvent<HTMLButtonElement>`
- React-specific — `onChange` semantics differ from native

### Kod misoli

```tsx
// MouseEvent
function Button({ onClick }: { onClick: (e: React.MouseEvent<HTMLButtonElement>) => void }) {
  return <button onClick={(e) => {
    console.log(e.target);          // <button>
    console.log(e.currentTarget);   // <button> (always element with handler)
    console.log(e.nativeEvent);     // native MouseEvent
    e.preventDefault();
    e.stopPropagation();
    onClick(e);
  }}>Click</button>;
}

// ChangeEvent (input)
function Input({ onChange }: { onChange: (value: string) => void }) {
  return <input onChange={(e: React.ChangeEvent<HTMLInputElement>) => {
    onChange(e.target.value);
  }} />;
}

// FormEvent
function Form({ onSubmit }: { onSubmit: () => void }) {
  return <form onSubmit={(e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    onSubmit();
  }}>...</form>;
}

// KeyboardEvent
function SearchBox() {
  const [query, setQuery] = useState("");
  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      onKeyDown={(e: React.KeyboardEvent<HTMLInputElement>) => {
        if (e.key === "Enter") {
          search(query);
        }
        if (e.key === "Escape") {
          setQuery("");
        }
      }}
    />
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**SyntheticEvent class:**

```typescript
interface SyntheticEvent<T = Element, E = Event> {
  bubbles: boolean;
  currentTarget: EventTarget & T;
  defaultPrevented: boolean;
  eventPhase: number;
  isTrusted: boolean;
  nativeEvent: E;
  preventDefault(): void;
  stopPropagation(): void;
  target: EventTarget;
  timeStamp: number;
  type: string;
  isDefaultPrevented(): boolean;
  isPropagationStopped(): boolean;
  persist(): void;  // R16, no-op in R17+
}
```

**Event types:**

```tsx
// Common
React.MouseEvent<HTMLButtonElement>
React.ChangeEvent<HTMLInputElement>
React.FormEvent<HTMLFormElement>
React.KeyboardEvent<HTMLInputElement>
React.FocusEvent<HTMLInputElement>
React.DragEvent<HTMLDivElement>
React.WheelEvent<HTMLDivElement>
React.TouchEvent<HTMLDivElement>
React.PointerEvent<HTMLDivElement>
React.AnimationEvent<HTMLDivElement>
React.TransitionEvent<HTMLDivElement>
React.ClipboardEvent<HTMLInputElement>
React.CompositionEvent<HTMLInputElement>  // IME
```

**`target` vs `currentTarget`:**

```tsx
<div onClick={(e) => {
  console.log(e.target);          // actual clicked element (could be child)
  console.log(e.currentTarget);   // element with handler (the div)
}}>
  <button>Click</button>
</div>

// Click button:
// e.target = <button>
// e.currentTarget = <div>
```

**`preventDefault`:**

```tsx
<a href="/about" onClick={(e) => {
  e.preventDefault();  // prevent navigation
  // custom logic (SPA routing)
  router.push("/about");
}}>About</a>

<form onSubmit={(e) => {
  e.preventDefault();  // prevent default submission
  customSubmit();
}}>
```

**`stopPropagation`:**

```tsx
<div onClick={() => console.log("outer")}>
  <button onClick={(e) => {
    e.stopPropagation();
    console.log("inner");
  }}>Click</button>
</div>

// Click button: only "inner" logged
// Without stopPropagation: "inner" then "outer"
```

**`onChange` semantics:**

```tsx
// React's onChange = HTML's oninput (every keystroke)
<input onChange={(e) => console.log(e.target.value)} />

// HTML's onchange (commit on blur) — not directly available
// React equivalent: onBlur
<input onBlur={(e) => console.log(e.target.value)} />
```

**Event pooling (R16-R17, removed):**

```tsx
// R16 — events pooled (reused)
function Component() {
  const handleClick = (e) => {
    setTimeout(() => {
      console.log(e.target);  // ❌ event nullified by pool
    }, 100);
  };
}

// R16 fix
const handleClick = (e) => {
  e.persist();  // remove from pool
  setTimeout(() => console.log(e.target), 100);
};

// R17+ — pooling removed, events not nullified
const handleClick = (e) => {
  setTimeout(() => console.log(e.target), 100);  // ✅ works
};
// e.persist() — no-op (kept for compat)
```

Modern browsers efficient enough — pooling unnecessary.

**Event delegation:**

R16: events delegated to `document`. R17+: delegated to root container (allows multiple React roots, microfrontends).

**Async event handling:**

```tsx
// Async handler
const handleClick = async (e: React.MouseEvent) => {
  await api.call();
  // R17+: e still valid (no pooling)
  // R16: e nullified, save value first
};
```

**Custom synthetic events — Web Components:**

```tsx
function MyComponent() {
  const ref = useRef<HTMLElement>(null);

  useEffect(() => {
    const handler = (e: CustomEvent) => {
      console.log(e.detail);
    };
    ref.current?.addEventListener("custom-event", handler as EventListener);
    return () => ref.current?.removeEventListener("custom-event", handler as EventListener);
  }, []);

  return <my-web-component ref={ref} />;
}
```

R19 — better Web Component custom event support.

**Event handlers don't trigger re-render directly:**

```tsx
const handleClick = (e: MouseEvent) => {
  console.log("clicked");  // no re-render
  setCount(c => c + 1);    // state change → re-render
};
```

</details>

### Edge Cases

- **`onChange` not firing**: For controlled inputs, value prop without onChange — frozen.
- **Touch events**: `onTouchStart`, `onTouchEnd`. Pointer events preferred (unified).
- **`onMouseEnter` vs `onMouseOver`**: Enter/leave don't bubble. Over/out bubble.

### Follow-up savollar

- "Synthetic vs native events?" — React wraps native, normalizes. Use Synthetic in React.
- "Adding native listener?" — `useEffect` + ref + addEventListener (for Web Components, scroll on document, etc.).
- "TypeScript event type help?" — IDE autocomplete based on JSX element type.

</details>

---

### 19. Event delegation — R16 vs R17+ [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Event delegation** — React har individual element'ga listener attach qilmaydi, **bitta listener** root container'ga attach qiladi va event bubbling orqali handle qiladi. **R16**: events delegated to `document`. **R17+**: delegated to **root container** (createRoot's element). Sabab: multiple React versions bir page'da ishlashi (microfrontends), gradual upgrades.

### To'liq tushuntirish

**Internal mechanism:**

```typescript
// React on createRoot:
function createRoot(container: Element) {
  // Attach single delegated listener
  ALL_REACT_EVENTS.forEach(eventType => {
    container.addEventListener(eventType, dispatchEvent);
  });
}

function dispatchEvent(nativeEvent: Event) {
  // Find React fiber for event.target
  const fiber = findFiberFromDOM(nativeEvent.target);
  // Walk fiber tree, find handlers
  // Call handlers in bubble order
}
```

**Comparison:**

| | R16 | R17+ |
|--|-----|------|
| Delegation root | `document` | root container (`createRoot` element) |
| Multiple React versions | ❌ Conflicts | ✅ Each app independent |
| Microfrontends | Hard | Easier |
| `e.stopPropagation()` outside React | Doesn't stop React handlers | Stops them (root level) |

### Kod misoli

```tsx
// R17+ — multiple roots independent
const widgetRoot = createRoot(document.getElementById("widget"));
const appRoot = createRoot(document.getElementById("app"));

// Click in widget:
// - Native: bubbles up document
// - React: bubbles only within widget fiber tree (widget root delegated listener handles it)
// App root listener — DOESN'T receive (different root container)

// R16 (va R17 legacy mode) ReactDOM.render bilan:
// R16: barcha app'lar `document`-level delegation share qiladi
// R17 legacy mode: hali `ReactDOM.render` (lekin delegation root container'ga o'tgan)
// Click in widget could trigger app's bubble handlers (if not stopped)
```

```tsx
// Manually adding native listener
function Component() {
  useEffect(() => {
    const handler = (e: MouseEvent) => {
      console.log("native click");
    };
    document.addEventListener("click", handler);
    return () => document.removeEventListener("click", handler);
  }, []);

  return (
    <button onClick={(e) => {
      e.stopPropagation();
      console.log("React click");
    }}>
      Click
    </button>
  );
}

// R16: stopPropagation prevents both native and React document handlers
// R17+: stopPropagation prevents React (root level), native document handler still fires
```

<details>
<summary><strong>Deep Dive</strong></summary>

**R16 issues:**

```tsx
// Two React versions in same page:
// React 16 app delegates to document
// React 16 plugin delegates to document
// Conflict — cleanup, version mismatches
```

R17 fixed by per-root delegation.

**`createRoot` listener attachment:**

```typescript
function listenToAllSupportedEvents(rootContainerElement: Element) {
  const supportedEvents = [
    "click", "mousemove", "mousedown", "mouseup",
    "keydown", "keyup", "keypress",
    "submit", "change", "input",
    "focus", "blur",
    // ... ~70 events
  ];

  supportedEvents.forEach(eventType => {
    rootContainerElement.addEventListener(
      eventType,
      dispatchDiscreteEvent,
      { capture: false }
    );
    rootContainerElement.addEventListener(
      eventType,
      dispatchContinuousEvent,
      { capture: true }  // for some
    );
  });
}
```

R17+ attaches ~70 listeners to root container, not document.

**Microfrontends benefit:**

```tsx
// Multiple React roots
<div id="header-app">{/* React 18 */}</div>
<div id="main-app">{/* React 19 */}</div>
<div id="sidebar-app">{/* React 17 */}</div>

// Each independent
// React versions don't conflict
// Cleanup: each root.unmount() removes its listeners
```

**Event bubbling — React tree:**

```tsx
<div onClick={() => console.log("outer")}>
  <button onClick={() => console.log("inner")}>Click</button>
</div>

// Click button:
// 1. Native click on <button>
// 2. Native bubbles up document
// 3. Root container's React listener catches
// 4. React walks fiber tree:
//    - inner handler fires
//    - outer handler fires (bubble)
```

**Capture phase:**

```tsx
<div onClickCapture={() => console.log("outer-capture")}>
  <button onClick={() => console.log("inner")}>Click</button>
</div>

// Click button:
// 1. Capture: outer-capture fires
// 2. Bubble: inner fires (no outer onClick — only Capture)
```

**`stopPropagation` behavior:**

```tsx
<div onClick={() => console.log("outer")}>
  <button onClick={(e) => {
    e.stopPropagation();
    console.log("inner");
  }}>Click</button>
</div>

// Click: only "inner" (React-level propagation stopped)

// But native event still bubbles to root container — just doesn't reach outer handler
```

**Native vs React event order:**

```tsx
// useEffect attaches native listener
useEffect(() => {
  const handler = (e) => console.log("native");
  window.addEventListener("click", handler);
  return () => window.removeEventListener("click", handler);
}, []);

// React handler
<button onClick={() => console.log("react")}>Click</button>

// Click button:
// 1. Native click on button
// 2. Bubbles to root container (React handler fires)
// 3. Bubbles further to window (native handler fires)
// Order: react, native (root container before window)
```

R17+ — React listeners on root, not document.

**Synthetic event creation:**

```typescript
function dispatchEvent(nativeEvent: Event) {
  const syntheticEvent = createSyntheticEvent(nativeEvent);
  const fiber = getFiberFromDOM(nativeEvent.target);

  // Walk fiber tree (React tree, not DOM tree)
  let current = fiber;
  const handlers = [];

  while (current) {
    const handler = getEventHandler(current, eventType);
    if (handler) handlers.push({ fiber: current, handler });
    current = current.return;  // up React tree
  }

  // Capture phase (top-down)
  handlers.reverse().forEach(({ handler }) => {
    if (syntheticEvent.isPropagationStopped()) return;
    handler(syntheticEvent);
  });

  // Bubble phase (bottom-up)
  handlers.forEach(({ handler }) => {
    if (syntheticEvent.isPropagationStopped()) return;
    handler(syntheticEvent);
  });
}
```

**Fiber tree, not DOM tree:**

```tsx
// React tree may differ from DOM tree:
function App() {
  return (
    <Modal>  {/* Portal — different DOM */}
      <button onClick={() => console.log("from modal")}>Click</button>
    </Modal>
  );
}

// DOM:
<div id="root">
  <App>
    {/* Modal portal renders to document.body */}
  </App>
</div>
<div id="modal-root">
  <button>Click</button>
</div>

// React tree:
// Root → App → Modal → button

// Click button:
// React walks React tree (Modal in tree, even though DOM separate)
// Bubble: button → Modal → App → Root
```

Portal events bubble through React tree, not DOM tree.

**`onScroll` doesn't bubble:**

```tsx
// Native scroll doesn't bubble
// React onScroll - on element with handler

<div onScroll={() => console.log("scroll")}>
  <div style={{ overflow: "auto" }}>...</div>
</div>
// Scroll inner div — outer onScroll won't fire
```

</details>

### Edge Cases

- **Multiple roots overlap**: DOM children can be inside multiple roots. React handles per fiber tree.
- **Native event on shadow DOM**: React doesn't traverse shadow DOM by default.
- **`onScroll` on window**: Use `useEffect` + addEventListener.

### Follow-up savollar

- "R16 → R17 breaking changes?" — Mostly invisible. `e.stopPropagation()` semantics changed for cross-react-version scenarios.
- "Native event ordering vs React?" — Native fires first (target), React handlers fire when event reaches root container.
- "Test with delegation?" — RTL `userEvent.click` simulates real bubbling.

</details>

---

### 20. R19 form actions — `<form action={fn}>` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**R19 form actions** — `<form action={asyncFunction}>` — function prop sifatida form'ga attach qilinadi. Submit'da function `FormData` argument bilan chaqiriladi. **Server actions** (`"use server"` directive) yoki **client actions** (oddiy async function) bo'lishi mumkin. `useFormStatus` (child component ichida) pending state beradi, `useActionState` form state management. Uncontrolled form-friendly, progressive enhancement (server actions JS'siz ham ishlaydi).

### Kod misoli

```tsx
// Client action — function in component
function ContactForm() {
  return (
    <form action={async (formData: FormData) => {
      const name = formData.get("name") as string;
      const email = formData.get("email") as string;
      await api.sendContact({ name, email });
    }}>
      <input name="name" />
      <input name="email" type="email" />
      <button type="submit">Send</button>
    </form>
  );
}

// With useFormStatus
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? "Sending..." : "Send"}</button>;
}

function ContactForm() {
  return (
    <form action={async (formData) => { await sendContact(formData); }}>
      <input name="name" />
      <SubmitButton />
    </form>
  );
}

// Server action
"use server";
export async function submitContact(formData: FormData) {
  const name = formData.get("name") as string;
  await db.contacts.create({ data: { name } });
  revalidatePath("/contacts");
}

// Client component
"use client";
import { submitContact } from "./actions";

function ContactForm() {
  return (
    <form action={submitContact}>
      <input name="name" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Pre-R19 form pattern:**

```tsx
function Form() {
  const [pending, setPending] = useState(false);

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setPending(true);
    const formData = new FormData(e.currentTarget);
    await api.submit(formData);
    setPending(false);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="x" />
      <button disabled={pending}>Submit</button>
    </form>
  );
}
```

**R19 form pattern:**

```tsx
function Form() {
  return (
    <form action={async (formData) => {
      await api.submit(formData);
    }}>
      <input name="x" />
      <SubmitButton />  // useFormStatus inside
    </form>
  );
}
```

Less boilerplate, automatic pending state.

**Action types:**

| Type | Marker | Where runs |
|------|--------|------------|
| Client action | Plain function | Browser |
| Server action | `"use server"` directive | Server (Node.js/Edge) |

**Server action mechanism:**

```tsx
// Build time: bundler creates RPC endpoint
// Form submit: POST request to that endpoint
// Server: runs original function with FormData
// Response: RSC payload (UI updates)
```

**Form reset:**

```tsx
<form action={async (formData) => {
  await submit(formData);
  // R19: form auto-reset on success (uncontrolled inputs)
}}>
  <input name="text" defaultValue="" />
  <button>Submit</button>
</form>

// Reset: input.value back to defaultValue
```

**Progressive enhancement (server actions):**

```tsx
// Without JS — form submits as POST to server endpoint
<form action={serverAction}>
  <input name="email" />
  <button>Subscribe</button>
</form>
// Browser default form submission → server runs action
// Page reloads with new state

// With JS — React intercepts, submits via fetch, no reload
```

Server actions degrade gracefully without JS.

**`useActionState` integration:**

```tsx
const [state, action, isPending] = useActionState(
  async (prevState, formData) => {
    const result = await submit(formData);
    return result;  // becomes new state
  },
  initialState
);

<form action={action}>
  <input name="x" />
  <button disabled={isPending}>Submit</button>
  {state && <p>{state.message}</p>}
</form>
```

**`useOptimistic` integration:**

```tsx
function TodoList({ initialTodos }: Props) {
  const [todos, setTodos] = useState(initialTodos);
  const [optimisticTodos, addOptimistic] = useOptimistic(todos, (state, text) => [
    ...state,
    { id: "temp", text, sending: true },
  ]);

  return (
    <form action={async (formData) => {
      const text = formData.get("text") as string;
      addOptimistic(text);  // immediate UI
      const newTodo = await api.addTodo(text);
      setTodos(prev => [...prev, newTodo]);
    }}>
      <input name="text" />
      <button>Add</button>
    </form>
  );
}
```

**Form action with fetcher pattern:**

```tsx
function searchAction(formData: FormData) {
  const query = formData.get("query") as string;
  // ... search logic
}

<form action={searchAction}>
  <input name="query" />
  <button>Search</button>
</form>
```

**Limitations:**

- `<form action={fn}>` — function as prop (not URL string)
- `<form action="/api/submit">` — still works (string URL, native submit)
- Mixing — string and function not allowed simultaneously

**Type safety:**

```tsx
async function submitContact(formData: FormData): Promise<void> {
  const name = formData.get("name") as string;  // type assertion (FormData returns string|File|null)
  await db.contacts.create({ data: { name } });
}

<form action={submitContact}>
  <input name="name" />
</form>
```

For better types: Zod validation in action.

**`useFormStatus` access:**

```tsx
function NestedField() {
  const { pending, data, method, action } = useFormStatus();
  return <div>Submitting: {pending ? "yes" : "no"}</div>;
}

<form action={submit}>
  <input name="x" />
  <NestedField />  {/* useFormStatus inside form */}
</form>
```

`useFormStatus` reads nearest `<form>` ancestor.

**Comparison with libraries:**

```
R19 form actions: simple, native form submission, server actions
React Hook Form: complex forms, validation, performance
Formik: full-featured, validation, controlled
```

For simple forms — R19 hooks. Complex — library.

**Multi-step forms:**

```tsx
function MultiStep() {
  const [step, setStep] = useState(1);

  if (step === 1) {
    return (
      <form action={(fd) => { saveStep1(fd); setStep(2); }}>
        <input name="name" />
        <button>Next</button>
      </form>
    );
  }
  if (step === 2) {
    return (
      <form action={(fd) => { saveStep2(fd); setStep(3); }}>
        <input name="email" />
        <button>Next</button>
      </form>
    );
  }
  return <p>Done!</p>;
}
```

</details>

### Edge Cases

- **`<form action="/url">`**: Still works (string URL — native form submission).
- **Mixing string and function**: `<form action={fn} method="POST">` — works.
- **Server action throw**: Error boundary catches.

### Follow-up savollar

- "Why not just onSubmit?" — onSubmit still works. action= clearer for async form actions, especially server actions.
- "Server actions security?" — Functions exposed via RPC. Validate inputs (Zod), check auth.
- "Form action vs onSubmit?" — Both valid. `action`: declarative, server actions friendly. `onSubmit`: imperative, more control.

</details>

---

### 21. Event handler patterns [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Common patterns**: (1) **Inline handler** — `onClick={() => action()}`, (2) **Method reference** — `onClick={handleClick}`, (3) **Arguments via closure** — `onClick={() => action(id)}`, (4) **Stable callback** — `useCallback`, (5) **Debounce/throttle** — custom hook, (6) **Async handler** — try/catch error handling.

### Kod misoli

```tsx
// 1. Inline arrow
<button onClick={() => console.log("clicked")}>Click</button>

// 2. Method reference
function Component() {
  const handleClick = () => console.log("clicked");
  return <button onClick={handleClick}>Click</button>;
}

// 3. Arguments via closure
{items.map(item => (
  <button key={item.id} onClick={() => deleteItem(item.id)}>
    Delete
  </button>
))}

// 4. Stable callback (memoized child)
const MemoButton = memo(Button);

function Parent() {
  const handleClick = useCallback((id: string) => {
    console.log(id);
  }, []);

  return items.map(item => (
    <MemoButton
      key={item.id}
      onClick={() => handleClick(item.id)}  // ⚠️ inline still — bypasses memo
    />
  ));
}

// Better: pass id as data attribute or onClick handler stable
const MemoButton = memo(({ onClick, id }: { onClick: (id: string) => void; id: string }) => (
  <button onClick={() => onClick(id)}>Click</button>
));

function Parent() {
  const handleClick = useCallback((id: string) => console.log(id), []);
  return items.map(item => (
    <MemoButton key={item.id} onClick={handleClick} id={item.id} />
  ));
}
```

```tsx
// 5. Debounced handler
function useDebouncedCallback<T extends (...args: never[]) => void>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  const timeoutRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  return useCallback((...args: Parameters<T>) => {
    if (timeoutRef.current) clearTimeout(timeoutRef.current);
    timeoutRef.current = setTimeout(() => fn(...args), delay);
  }, [fn, delay]);
}

function Search() {
  const handleSearch = useDebouncedCallback((query: string) => {
    api.search(query);
  }, 300);

  return <input onChange={(e) => handleSearch(e.target.value)} />;
}

// 6. Async handler
function Form() {
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      await api.submit();
    } catch (err) {
      setError(err instanceof Error ? err.message : "Unknown error");
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      ...
      {error && <p>{error}</p>}
    </form>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Inline vs method reference:**

```tsx
// Inline — new function each render
<button onClick={() => console.log("clicked")} />

// Method reference — same reference if stable (in body)
const handleClick = () => console.log("clicked");
<button onClick={handleClick} />

// For native button: doesn't matter (DOM doesn't compare)
// For memo'd child: matters (props comparison)
```

**Naming convention:**

| Prefix | Meaning |
|--------|---------|
| `onAction` | Event prop (e.g., onClick) |
| `handleAction` | Handler function inside component |

```tsx
function Card({ onSelect }: { onSelect: (id: string) => void }) {  // prop
  const handleClick = () => onSelect(id);  // handler
  return <div onClick={handleClick}>...</div>;
}
```

**`prevent default` patterns:**

```tsx
// Anchor preventing navigation
<a href="/about" onClick={(e) => {
  e.preventDefault();
  router.push("/about");
}}>About</a>

// Form preventing submission
<form onSubmit={(e) => {
  e.preventDefault();
  customSubmit();
}}>

// Drag/drop
<div onDragOver={(e) => e.preventDefault()} onDrop={handleDrop}>
```

**Conditional handler:**

```tsx
<button onClick={isReady ? handleSubmit : undefined}>
  Submit
</button>

// Or disabled
<button disabled={!isReady} onClick={handleSubmit}>
  Submit
</button>
```

**Multiple handlers (chain):**

```tsx
function combineHandlers<T>(...handlers: ((e: T) => void)[]) {
  return (e: T) => {
    handlers.forEach(h => h(e));
  };
}

<button onClick={combineHandlers(logClick, trackAnalytics, handleLogic)}>
  Click
</button>
```

**Throttle vs debounce:**

```tsx
// Debounce — delay until inactive (last call wins)
useDebouncedCallback(searchAPI, 300);

// Throttle — limit rate (regular intervals)
useThrottledCallback(scrollHandler, 100);

// Use case:
// - Debounce: search-as-you-type (avoid until done typing)
// - Throttle: scroll, resize (rate-limit, not skip)
```

**Async error handling:**

```tsx
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  try {
    await api.submit();
  } catch (err) {
    if (err instanceof NetworkError) {
      setError("Network error");
    } else {
      setError("Unexpected error");
      throw err;  // re-throw for error boundary
    }
  }
};
```

**Stale closure in event handler:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  // ❌ Closure over count
  useEffect(() => {
    const handler = () => console.log(count);  // stale
    window.addEventListener("click", handler);
    return () => window.removeEventListener("click", handler);
  }, []);  // empty deps — handler captures initial count

  return <p>{count}</p>;
}

// ✅ Add count to deps
useEffect(() => {
  const handler = () => console.log(count);
  window.addEventListener("click", handler);
  return () => window.removeEventListener("click", handler);
}, [count]);
// Re-attach on count change

// ✅ Or ref pattern
const countRef = useRef(count);
useEffect(() => { countRef.current = count; });
useEffect(() => {
  const handler = () => console.log(countRef.current);
  window.addEventListener("click", handler);
  return () => window.removeEventListener("click", handler);
}, []);
```

**Capture vs bubble:**

```tsx
// Bubble (default)
<div onClick={parentHandler}>
  <button onClick={childHandler}>Click</button>
</div>
// Order: child → parent

// Capture
<div onClickCapture={parentHandler}>
  <button onClick={childHandler}>Click</button>
</div>
// Order: parent (capture) → child (bubble)
```

**Event object access:**

```tsx
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
  e.target;          // clicked element (could be child)
  e.currentTarget;   // element with handler
  e.preventDefault();
  e.stopPropagation();
  e.nativeEvent;     // native MouseEvent

  // Type-specific
  e.button;          // 0=left, 1=middle, 2=right
  e.shiftKey;
  e.ctrlKey;
};
```

**Drag and drop:**

```tsx
function Draggable() {
  return (
    <div
      draggable
      onDragStart={(e) => e.dataTransfer.setData("text/plain", "data")}
    >
      Drag me
    </div>
  );
}

function DropZone() {
  return (
    <div
      onDragOver={(e) => e.preventDefault()}
      onDrop={(e) => {
        e.preventDefault();
        const data = e.dataTransfer.getData("text/plain");
        console.log("Dropped:", data);
      }}
    >
      Drop here
    </div>
  );
}
```

**Keyboard shortcuts:**

```tsx
function KeyboardComponent() {
  useEffect(() => {
    const handler = (e: KeyboardEvent) => {
      if (e.metaKey && e.key === "k") {
        e.preventDefault();
        openSearch();
      }
    };
    window.addEventListener("keydown", handler);
    return () => window.removeEventListener("keydown", handler);
  }, []);

  return null;
}
```

</details>

### Edge Cases

- **Inline function performance**: For native elements, no impact. For memo'd children, matters.
- **Async event leak**: Cleanup with `cancelled` flag or AbortController.
- **Touch + mouse**: Use Pointer events (unified).

### Follow-up savollar

- "useCallback always for handlers?" — Only for memo'd children. Native — no.
- "Event handler in render body?" — Possible but creates new function each render.
- "Test handlers?" — RTL `userEvent.click(button)` simulates.

</details>

---

### 22. Native event listener vs synthetic — qachon useEffect bilan attach? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React synthetic events (`onClick`, `onChange`) — root container'ga delegated. Lekin ba'zi event'lar React'da yo'q (`scroll` window'da, `resize`, custom events) yoki sync handling kerak (`wheel` passive=false, `touchstart` preventDefault). Bu paytlar `useEffect` ichida `addEventListener` + cleanup. Common: window/document events, third-party DOM event'lar, custom events.

### Kod misoli

```tsx
// Window scroll
function ScrollWatcher() {
  const [scrollY, setScrollY] = useState(0);

  useEffect(() => {
    const handler = () => setScrollY(window.scrollY);
    window.addEventListener("scroll", handler, { passive: true });
    return () => window.removeEventListener("scroll", handler);
  }, []);

  return <p>Scroll: {scrollY}</p>;
}

// Window resize
function WindowSize() {
  const size = useSyncExternalStore(
    (callback) => {
      window.addEventListener("resize", callback);
      return () => window.removeEventListener("resize", callback);
    },
    () => ({ w: window.innerWidth, h: window.innerHeight }),
    () => ({ w: 0, h: 0 })
  );
  return <p>{size.w}x{size.h}</p>;
}

// Custom event (Web Components)
function MyComponent() {
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;

    const handler = (e: CustomEvent<{ value: string }>) => {
      console.log(e.detail.value);
    };

    // CustomEvent type'lari TS standard EventMap'da yo'q — type assertion zarur
    el.addEventListener("my-event", handler as EventListener);
    return () => el.removeEventListener("my-event", handler as EventListener);
  }, []);

  return <div ref={ref}><my-component /></div>;
}

// Wheel preventDefault (passive=false)
function WheelTrap() {
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;

    const handler = (e: WheelEvent) => {
      e.preventDefault();  // requires passive=false
      // custom scroll logic
    };

    el.addEventListener("wheel", handler, { passive: false });
    return () => el.removeEventListener("wheel", handler);
  }, []);

  return <div ref={ref}>Scroll trap</div>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**React event vs native:**

| Event | React | Native useEffect |
|-------|-------|-------------------|
| Click | `onClick` | rare |
| Change | `onChange` | rare |
| Scroll (element) | `onScroll` | window scroll only |
| Resize | ❌ | yes (window) |
| Wheel passive=false | ❌ (passive=true default) | yes |
| Touch passive=false | ❌ | yes |
| Custom events | ❌ | yes |
| `popstate`, `hashchange` | ❌ | yes |
| `beforeunload` | ❌ | yes |

**Why React passive=true default:**

Modern browsers — `touchstart`, `wheel` default passive (no preventDefault). React inherits. Override needs native attach.

**Memory leak patterns:**

```tsx
// ❌ Missing cleanup
useEffect(() => {
  window.addEventListener("scroll", handler);
  // no return — leak on unmount
}, []);

// ❌ Stale closure in handler
useEffect(() => {
  const handler = () => console.log(count);  // closure
  window.addEventListener("scroll", handler);
  return () => window.removeEventListener("scroll", handler);
}, []);  // count stale

// ✅ Latest via useRef
const countRef = useRef(count);
countRef.current = count;
useEffect(() => {
  const handler = () => console.log(countRef.current);
  window.addEventListener("scroll", handler);
  return () => window.removeEventListener("scroll", handler);
}, []);
```

**Custom hook abstraction:**

```tsx
function useEventListener<K extends keyof WindowEventMap>(
  type: K,
  handler: (e: WindowEventMap[K]) => void,
  options?: AddEventListenerOptions
) {
  const handlerRef = useRef(handler);
  handlerRef.current = handler;

  useEffect(() => {
    const wrapped = (e: WindowEventMap[K]) => handlerRef.current(e);
    window.addEventListener(type, wrapped, options);
    return () => window.removeEventListener(type, wrapped, options);
  }, [type]);
}

// Usage — closure-safe
function App() {
  const [count, setCount] = useState(0);
  useEventListener("scroll", (e) => {
    console.log(count);  // ✅ always latest
  });
}
```

</details>

### Edge Cases

- **Multiple listeners**: Each useEffect attach separately. No deduplication.
- **Throttle/debounce**: Wrap handler with throttle, cleanup cancel.
- **Passive vs non-passive**: Mixed events need careful flag.

### Follow-up savollar

- "useEvent (RFC) for this?" — Yes, designed for stable callback. Not yet stable.
- "AbortController for cleanup?" — `addEventListener` accepts `signal`. Single `controller.abort()` removes all.

</details>

---

<a id="qism-f"></a>

## QISM F: Legacy Patterns

### 23. Render Props pattern [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Render Props** — komponent **function** prop qabul qiladi (yoki `children` function), shu funksiyani internal state bilan chaqirib render qiladi. Caller funksiyada custom UI yozadi. Logic reuse pre-Hooks pattern. R16.8+ Hooks pattern preferred (custom hook). Hali ko'p library'larda mavjud (Formik render prop, etc.).

### Kod misoli

```tsx
// Mouse tracker — render prop
interface MouseTrackerProps {
  render: (pos: { x: number; y: number }) => ReactNode;
}

function MouseTracker({ render }: MouseTrackerProps) {
  const [pos, setPos] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handler = (e: MouseEvent) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener("mousemove", handler);
    return () => window.removeEventListener("mousemove", handler);
  }, []);

  return <>{render(pos)}</>;
}

// Usage
<MouseTracker render={(pos) => (
  <div>Mouse: {pos.x}, {pos.y}</div>
)} />
```

```tsx
// Function as children variant
<MouseTracker>
  {(pos) => <div>Mouse: {pos.x}, {pos.y}</div>}
</MouseTracker>

// Implementation
function MouseTracker({ children }: { children: (pos: { x: number; y: number }) => ReactNode }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  // ...
  return <>{children(pos)}</>;
}
```

```tsx
// Modern Hook equivalent
function useMouse() {
  const [pos, setPos] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handler = (e: MouseEvent) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener("mousemove", handler);
    return () => window.removeEventListener("mousemove", handler);
  }, []);

  return pos;
}

function MyComponent() {
  const pos = useMouse();
  return <div>Mouse: {pos.x}, {pos.y}</div>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why "render props":**

Component renders via prop function. Caller controls render output, library provides state/logic.

**Compared to HOC:**

```tsx
// HOC
const withMouse = (Component) => {
  return class extends React.Component {
    state = { x: 0, y: 0 };
    componentDidMount() {
      window.addEventListener("mousemove", this.handler);
    }
    handler = (e) => this.setState({ x: e.clientX, y: e.clientY });
    render() {
      return <Component {...this.props} mouse={this.state} />;
    }
  };
};

const EnhancedComp = withMouse(MyComp);
```

```tsx
// Render prop
<MouseTracker render={(mouse) => <MyComp mouse={mouse} />} />
```

```tsx
// Custom Hook (modern)
function MyComp() {
  const mouse = useMouse();
  return <p>{mouse.x}</p>;
}
```

**Render prop pros:**

- Explicit data flow (no magic prop injection)
- TypeScript-friendly
- Composition

**Render prop cons:**

- Verbose JSX
- "Wrapper hell" — nested render props
- Each render creates new function (memo issues)

**`React.Children.only`:**

```tsx
import { Children } from "react";

function StrictChild({ children }: { children: ReactNode }) {
  return Children.only(children);  // throws if more than 1 child
}
```

**Render prop with multiple values:**

```tsx
interface DataLoaderProps<T> {
  url: string;
  children: (state: { data: T | null; loading: boolean; error: Error | null }) => ReactNode;
}

function DataLoader<T>({ url, children }: DataLoaderProps<T>) {
  const { data, loading, error } = useFetch<T>(url);
  return <>{children({ data, loading, error })}</>;
}

<DataLoader<User> url="/api/user">
  {({ data, loading, error }) => {
    if (loading) return <Spinner />;
    if (error) return <Error />;
    if (!data) return null;
    return <UserCard user={data} />;
  }}
</DataLoader>
```

**Real-world libraries:**

- **Formik** — render prop or Form component
- **Apollo Client (older)** — `<Query>{({ data }) => ...}</Query>`
- **react-router (older)** — `<Route render={({ match }) => ...} />`
- **Headless UI** — uses render prop pattern in some components

**Modern alternatives:**

```tsx
// Old
<Query>{({ data }) => <UI data={data} />}</Query>

// New (hook)
const { data } = useQuery();
return <UI data={data} />;
```

**Render prop performance:**

```tsx
function App() {
  return (
    <MouseTracker render={(pos) => (
      // ❌ New function each render
      <div>{pos.x}</div>
    )} />
  );
}
```

Render function recreated each render. For memo'd children downstream — bypass.

**`useCallback` for stable render prop:**

```tsx
const renderMouse = useCallback((pos) => <div>{pos.x}</div>, []);

<MouseTracker render={renderMouse} />
```

But often not needed (no perf issue for typical use).

**Render prop with state:**

```tsx
function StateProvider({ children, initial }: { children: (state: any, set: (s: any) => void) => ReactNode; initial: any }) {
  const [state, setState] = useState(initial);
  return <>{children(state, setState)}</>;
}

<StateProvider initial={0}>
  {(state, setState) => (
    <button onClick={() => setState(state + 1)}>{state}</button>
  )}
</StateProvider>
```

**Hooks superseded most render prop use:**

| Use case | Pre-hooks | Hooks |
|----------|-----------|-------|
| State | render prop | useState |
| Effects | render prop | useEffect |
| Context | Consumer | useContext |
| Logic reuse | render prop / HOC | custom hook |

**When render props still useful:**

- Component-level inversion of control (Disclosure, Combobox)
- Library API targeting old codebases
- When wrapper component naturally fits

</details>

### Edge Cases

- **`children` as function vs `render` prop**: Same pattern, different prop name.
- **Multiple render props**: `<Comp render={...} fallback={...} />` — multiple slots.
- **Render prop returns null**: Skipped.

### Follow-up savollar

- "Why hooks better?" — No wrapper, less verbose, better composition, type inference.
- "Render props in 2026?" — Less common but still valid for component IoC.
- "Performance: render prop vs hook?" — Hooks slightly less overhead (no wrapper component).

</details>

---

### 24. HOC (Higher-Order Components) — legacy [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**HOC** — function takes Component, returns enhanced Component. Pattern: `withSomething(Component)`. Pre-Hooks era logic reuse mechanism. Pros: composition, reusability. Cons: wrapper hell, prop collisions, hidden complexity, hard typing. R16.8+ Custom Hooks preferred. Few cases still HOC: ref forwarding wrappers, library auth wrappers (legacy).

### Kod misoli

```tsx
// HOC — withAuth
function withAuth<P extends { user?: User }>(Component: React.ComponentType<P>) {
  return function WithAuthComponent(props: Omit<P, "user">) {
    const { user } = useContext(AuthContext);

    if (!user) return <LoginRequired />;

    return <Component {...(props as P)} user={user} />;
  };
}

// Original component
interface ProfileProps {
  user: User;
  showAvatar?: boolean;
}

function Profile({ user, showAvatar }: ProfileProps) {
  return <div>{user.name}</div>;
}

// Enhanced
const ProtectedProfile = withAuth(Profile);

<ProtectedProfile showAvatar />  // user injected by HOC
```

```tsx
// withLoading HOC
function withLoading<P extends object>(Component: React.ComponentType<P>) {
  return function WithLoadingComponent({ loading, ...props }: P & { loading: boolean }) {
    if (loading) return <Spinner />;
    return <Component {...(props as P)} />;
  };
}

const LoadingProfile = withLoading(Profile);

<LoadingProfile loading={isLoading} user={user} />
```

```tsx
// Modern hook equivalent
function Profile({ user }: { user: User }) {
  const { user: authUser } = useAuth();  // hook
  const { loading } = useLoading();      // hook

  if (loading) return <Spinner />;
  if (!authUser) return <LoginRequired />;

  return <div>{user.name}</div>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**HOC convention:**

```tsx
// Naming: with* prefix
withAuth, withLoading, withRouter, withTheme

// Returns enhanced component
const Enhanced = withSomething(Original);

// Composable
const Enhanced = withAuth(withLoading(withTheme(Component)));
```

**HOC pros:**

- Reusable logic
- Can wrap any component (function or class)
- Compatible with both functions and classes

**HOC cons:**

1. **Wrapper hell** — `withAuth(withTheme(withLoading(withRouter(Component))))`
2. **Prop name collision** — `withAuth` injects `user`, `withTheme` also injects `user` — conflict
3. **Static methods lost** — Without `hoist-non-react-statics`, statics gone
4. **DisplayName** — DevTools shows `WithAuthWithThemeComponent` (cryptic)
5. **Type inference complex** — Generic HOC hard to type
6. **Implicit dependencies** — Where does `user` come from? Hidden in HOC

**`hoist-non-react-statics`:**

```tsx
import hoistNonReactStatics from "hoist-non-react-statics";

function withAuth(Component) {
  function Enhanced(props) { ... }
  hoistNonReactStatics(Enhanced, Component);
  return Enhanced;
}

// Original.someStaticMethod available on Enhanced
```

**displayName:**

```tsx
function withAuth(Component) {
  function Enhanced(props) { ... }
  Enhanced.displayName = `withAuth(${Component.displayName || Component.name || "Component"})`;
  return Enhanced;
}
```

**Ref forwarding HOC (R18 — verbose):**

```tsx
function withAuth<P>(Component: React.ComponentType<P>) {
  return forwardRef<any, P>((props, ref) => {
    const { user } = useAuth();
    if (!user) return <LoginRequired />;
    return <Component {...props} user={user} ref={ref} />;
  });
}
```

R19 — ref as regular prop (simpler).

**HOC composition:**

```tsx
// Option 1: nested
const Enhanced = withA(withB(withC(Component)));

// Option 2: compose helper
import { compose } from "lodash/fp";

const enhance = compose(withA, withB, withC);
const Enhanced = enhance(Component);

// Option 3: pipe (more readable)
import { pipe } from "lodash/fp";

const enhance = pipe(withA, withB, withC);
const Enhanced = enhance(Component);
```

**Custom Hook vs HOC:**

```tsx
// HOC version
const Enhanced = withAuth(withTheme(MyComponent));

// Hook version
function MyComponent() {
  const { user } = useAuth();
  const { theme } = useTheme();
  return <div>{user.name} ({theme})</div>;
}
```

Hooks: clearer dependencies, no wrappers, simpler types.

**When HOC still useful:**

1. **Cross-cutting concerns at app boundary** — `withErrorBoundary`, `withSuspense`
2. **Library wrappers** — Redux's `connect` (legacy), styled-components factory
3. **Conditional component rendering** — `withFeatureFlag`

```tsx
// Modern alternative — composition
function ProtectedRoute({ children }: { children: ReactNode }) {
  const { user } = useAuth();
  if (!user) return <LoginRequired />;
  return <>{children}</>;
}

<ProtectedRoute>
  <Profile />
</ProtectedRoute>
```

Wrapper component instead of HOC — simpler.

**`memo` va `forwardRef` texnik jihatdan HOC-like:**

```tsx
const MemoButton = memo(Button);  // HOC-like wrapper pattern
const FwdInput = forwardRef(Input);  // HOC-like (R18 pattern)
// R19: forwardRef function component'lar uchun kerak emas (ref oddiy prop)
// memo hali aktual va keng ishlatiladi
```

**TypeScript HOC complexity:**

```tsx
// HOC injecting prop, removing it from required
type WithAuthProps = { user: User };

function withAuth<P extends WithAuthProps>(
  Component: React.ComponentType<P>
): React.ComponentType<Omit<P, "user">> {
  return function WithAuth(props: Omit<P, "user">) {
    const { user } = useAuth();
    // HOC'da generic props forwarding TS limitation — workaround
    return <Component {...(props as P)} user={user} />;
  };
}

// Strict typing harder than custom hook — HOC'ning asosiy kamchiliklaridan biri
```

**Migration strategy:**

```tsx
// Step 1: extract logic to hook
function useAuthCheck() {
  const { user } = useAuth();
  return user;
}

// Step 2: replace HOC with hook in component
function Profile() {
  const user = useAuthCheck();
  if (!user) return <LoginRequired />;
  // ...
}

// Step 3: remove HOC (no callers)
```

</details>

### Edge Cases

- **HOC and key prop**: Key not forwarded automatically. Manual handling.
- **HOC with ref**: forwardRef in HOC (R18) or ref-as-prop (R19).
- **HOC composition order**: `withA(withB(C))` — innermost (B) closest to original C.

### Follow-up savollar

- "Convert HOC to hook?" — Extract logic to hook, replace HOC usages with hook calls.
- "When HOC over hook?" — Cross-cutting wrapper (error boundary, suspense). Else hook.
- "memo va forwardRef deprecated?" — `memo` deprecated emas, aktual. `forwardRef` R19'da function component'lar uchun kerak emas (ref oddiy prop), lekin deprecated emas — class component'lar va backward compatibility uchun mavjud.

</details>

---

### 25. Custom hooks vs HOC vs Render Props — comparison [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Custom hooks** (modern, R16.8+) — function-based logic reuse, no wrapper, clean composition. **HOC** — function returning Component, wrapper-based. **Render props** — Component with function child/prop. Custom hooks **strictly superior** for most cases — no wrapper hell, type-safe, composable. HOC/render props legacy patterns hali ba'zi librarylarda saqlanib turadi.

### Comparison table

| Aspect | Custom Hooks | HOC | Render Props |
|--------|--------------|-----|--------------|
| API | Function call | Wrapper component | JSX prop |
| Composition | Sequential calls | Nested wrappers | Nested JSX |
| TypeScript | Excellent | Complex | Good |
| DevTools | Hook name | "WithX(Y)" | Component tree |
| Wrapper | None | One per HOC | Component itself |
| Children API | None | Same | `children` is function |
| Performance | Best | Slight wrapper overhead | Function recreation |

### Kod misoli

**Same logic, three patterns:**

```tsx
// 1. Custom Hook (modern preferred)
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = useCallback(() => setCount(c => c + 1), []);
  return { count, increment };
}

function Component() {
  const { count, increment } = useCounter();
  return <button onClick={increment}>{count}</button>;
}

// 2. HOC (legacy)
function withCounter<P>(Component: React.ComponentType<P & { count: number; increment: () => void }>) {
  return function WithCounter(props: P) {
    const [count, setCount] = useState(0);
    const increment = () => setCount(c => c + 1);
    return <Component {...props} count={count} increment={increment} />;
  };
}

const Component = withCounter(({ count, increment }) => (
  <button onClick={increment}>{count}</button>
));

// 3. Render Props (legacy)
function Counter({ children }: { children: (count: number, inc: () => void) => ReactNode }) {
  const [count, setCount] = useState(0);
  return <>{children(count, () => setCount(c => c + 1))}</>;
}

function Component() {
  return (
    <Counter>
      {(count, increment) => <button onClick={increment}>{count}</button>}
    </Counter>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Migration paths:**

```tsx
// Old HOC
const Enhanced = withAuth(withTheme(Component));

// New hooks
function Component() {
  const { user } = useAuth();
  const { theme } = useTheme();
  return ...;
}

// Old render prop
<DataLoader>{(data) => <UI data={data} />}</DataLoader>

// New hook
function Comp() {
  const data = useDataLoader();
  return <UI data={data} />;
}
```

**When NOT to migrate:**

- Library code with public API using HOC/render props
- Codebase fully invested in pattern
- Specific use case where wrapper component natural

**TypeScript inference:**

```tsx
// Hook — natural inference
const { count } = useCounter();  // count: number

// HOC — generic gymnastics
const Enhanced = withCounter(Component);  // Component prop types modified

// Render prop — function arguments inferred
<Counter>{(count, inc) => ...}</Counter>  // count: number, inc: () => void
```

**Hook composition:**

```tsx
// Compose multiple hooks naturally
function useUserProfile(id: string) {
  const user = useUser(id);
  const posts = usePosts(id);
  const follows = useFollows(id);
  return { user, posts, follows };
}

// HOC equivalent
const Enhanced = withUser(withPosts(withFollows(Component)));
// Verbose, prop collisions possible
```

**Render prop combination:**

```tsx
// Verbose
<UserProvider>
  {user => (
    <PostsProvider userId={user.id}>
      {posts => (
        <FollowsProvider userId={user.id}>
          {follows => <Profile user={user} posts={posts} follows={follows} />}
        </FollowsProvider>
      )}
    </PostsProvider>
  )}
</UserProvider>
```

vs hooks:

```tsx
function Profile({ id }: { id: string }) {
  const user = useUser(id);
  const posts = usePosts(id);
  const follows = useFollows(id);
  return <div>{user.name}</div>;
}
```

**Library examples:**

- **React Query**: hooks (`useQuery`)
- **Apollo Client**: hooks now (was render props)
- **Formik**: render prop or hook
- **react-router**: hooks (`useNavigate`, `useLocation`)
- **Redux**: hooks (`useSelector`, `useDispatch`)

**Hooks limitations:**

- Component IoC (compound components, accordion content) — render props or compound components better
- Pre-hooks codebases — gradual migration

**Performance:**

```typescript
// Hook — bevosita function call, qo'shimcha Fiber node yo'q
useCounter();

// HOC — wrapper component + qo'shimcha Fiber node
withCounter(Component)

// Render prop — function call + wrapper Fiber node
<Counter>{(c, inc) => ...}</Counter>
```

Hooks bir oz tezroq (kam wrapping va Fiber overhead), lekin amaliyotda farq sezilarsiz — har uch usul ham mikro-optimizatsiya emas, **DX va kod tuzilishi** asosiy farq.

**Mental model:**

- **Hook**: "I want this behavior" (logic injection)
- **HOC**: "I want this component enhanced" (component transformation)
- **Render prop**: "I want flexible rendering" (UI inversion)

</details>

### Edge Cases

- **Conditional logic**: Hooks must be top-level. HOC/render props more flexible (but no rules of hooks).
- **Class components**: HOC works for classes. Hooks function-only.
- **Static methods**: HOC needs hoist-non-react-statics. Hook N/A.

### Follow-up savollar

- "Cannot use hook in class — what to do?" — Convert to function. Or use HOC bridge (rare).
- "Render prop dead?" — Mostly. Compound components and hooks supersede.
- "Best pattern for new code?" — Hooks for logic, compound components for component IoC.

</details>

---

### 26. HOC TypeScript pattern — generic forwarding [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

HOC TypeScript signature: `<P extends object>(Component: ComponentType<P>) => ComponentType<P & ExtraProps>` — preserve original props (P) plus add extra. Real-world: `withAuth(Component) => Component & { user: User }`. R19'da function component'larda `forwardRef` kerak emas (ref oddiy prop), shuning uchun HOC'da ref forwarding soddalashgan.

### Kod misoli

```tsx
import { ComponentType } from "react";

// Generic HOC
function withLoading<P extends { isLoading?: boolean }>(
  Component: ComponentType<P>
): ComponentType<P> {
  return function WithLoading(props: P) {
    if (props.isLoading) return <Spinner />;
    return <Component {...props} />;
  };
}

// HOC adding props
function withAuth<P extends object>(
  Component: ComponentType<P & { user: User }>
): ComponentType<P> {
  return function WithAuth(props: P) {
    const user = useAuth();
    if (!user) return <Login />;
    return <Component {...props} user={user} />;
  };
}

// Usage
interface ProfileProps {
  showEmail: boolean;
  user: User;
}

function Profile({ showEmail, user }: ProfileProps) {
  return <div>{user.name}{showEmail && user.email}</div>;
}

const AuthedProfile = withAuth(Profile);
// AuthedProfile expects only { showEmail: boolean }
// user injected by HOC
<AuthedProfile showEmail />
```

<details>
<summary><strong>Deep Dive</strong></summary>

**displayName:**

```tsx
function withLogging<P>(Component: ComponentType<P>): ComponentType<P> {
  const Wrapped = (props: P) => {
    console.log("render", Component.displayName);
    return <Component {...props} />;
  };
  Wrapped.displayName = `withLogging(${Component.displayName ?? Component.name})`;
  return Wrapped;
}
```

**Forwarded ref (R18 pattern):**

```tsx
function withRef<P, R>(
  Component: React.ForwardRefExoticComponent<P & React.RefAttributes<R>>
) {
  return forwardRef<R, P>((props, ref) => {
    return <Component {...props} ref={ref} />;
  });
}
```

**Compose multiple HOCs:**

```tsx
const enhance = compose(withAuth, withLogging, withErrorBoundary);
const EnhancedComponent = enhance(MyComponent);
```

**Modern alternative — custom hook:**

```tsx
// Instead of withAuth HOC
function useAuth() { ... }

function Profile({ showEmail }: { showEmail: boolean }) {
  const user = useAuth();
  if (!user) return <Login />;
  return <div>{user.name}</div>;
}
```

Custom hooks — cleaner, no wrapping, better TypeScript.

</details>

### Edge Cases

- **Static methods**: HOC loses `Component.staticMethod`. Manually copy or use `hoist-non-react-statics`.
- **Prop name collision**: HOC adds prop with same name — overwrites. Be explicit.

### Follow-up savollar

- "Why HOC fading?" — Custom hooks simpler, no wrapping, better DX.
- "Still useful when?" — Component-level concerns (error boundary, route guard) — wrapper-based.

</details>

---

<a id="qism-g"></a>

## QISM G: Compound Components & Children API

### 27. Compound component pattern [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Compound components** — parent komponent state'ni bola komponentlar bilan **implicit** ravishda share qiladi (Context orqali). API: `<Tabs><Tabs.List><Tabs.Tab>...</Tabs.Tab></Tabs.List></Tabs>`. Caller flexible structure (markup, ordering), parent encapsulates logic. Use case: design system primitives — Tabs, Accordion, Select, Menu.

### Kod misoli

```tsx
// Tabs compound component
import { createContext, useContext } from "react";

interface TabsContextValue {
  active: string;
  setActive: (id: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error("Must be inside <Tabs>");
  return ctx;
}

interface TabsProps {
  defaultValue: string;
  children: React.ReactNode;
}

export function Tabs({ defaultValue, children }: TabsProps) {
  const [active, setActive] = useState(defaultValue);
  const value = useMemo(() => ({ active, setActive }), [active]);

  return <TabsContext value={value}>{children}</TabsContext>;
}

Tabs.List = function TabsList({ children }: { children: ReactNode }) {
  return <div role="tablist" className="tabs-list">{children}</div>;
};

Tabs.Tab = function Tab({ value, children }: { value: string; children: ReactNode }) {
  const { active, setActive } = useTabsContext();
  return (
    <button
      role="tab"
      aria-selected={active === value}
      onClick={() => setActive(value)}
    >
      {children}
    </button>
  );
};

Tabs.Panel = function TabPanel({ value, children }: { value: string; children: ReactNode }) {
  const { active } = useTabsContext();
  if (active !== value) return null;
  return <div role="tabpanel">{children}</div>;
};

// Usage — flexible markup
<Tabs defaultValue="overview">
  <Tabs.List>
    <Tabs.Tab value="overview">Overview</Tabs.Tab>
    <Tabs.Tab value="details">Details</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="overview">
    <p>Overview content</p>
  </Tabs.Panel>
  <Tabs.Panel value="details">
    <p>Details content</p>
  </Tabs.Panel>
</Tabs>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Implementation patterns:**

**1. Context-based (modern):**

```tsx
// Parent provides state via context
// Children read via useContext
// Flexibility: any structure, any depth
```

**2. `cloneElement`-based (legacy):**

```tsx
function Tabs({ children }: { children: ReactNode }) {
  const [active, setActive] = useState(0);

  return (
    <>
      {Children.map(children, (child, i) => {
        if (isValidElement(child)) {
          return cloneElement(child, { active: active === i, onClick: () => setActive(i) });
        }
        return child;
      })}
    </>
  );
}

// Usage
<Tabs>
  <Tab>One</Tab>
  <Tab>Two</Tab>
</Tabs>
// Tabs injects props into Tab via cloneElement
```

`cloneElement` legacy — Context preferred (more flexible).

**Real-world libraries:**

- **Radix UI**: `<Tabs.Root><Tabs.List>...`
- **Reach UI**: similar
- **Chakra UI**: `<Tabs><TabList><Tab>`
- **Headless UI**: `<Listbox>...`

**TypeScript pattern:**

```tsx
// Type the namespace
type TabsType = {
  (props: TabsProps): JSX.Element;
  List: typeof TabsList;
  Tab: typeof Tab;
  Panel: typeof TabPanel;
};

const Tabs: TabsType = ((props) => { ... }) as TabsType;
Tabs.List = TabsList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Or named exports
export const Tabs = TabsRoot;
export const TabsList = ...;
export const Tab = ...;
export const TabPanel = ...;
```

**Accordion compound:**

```tsx
const AccordionContext = createContext<...>(null);

function Accordion({ children, multiple = false }) {
  const [openItems, setOpenItems] = useState<Set<string>>(new Set());

  const toggle = (id: string) => {
    setOpenItems(prev => {
      const next = new Set(prev);
      if (next.has(id)) {
        next.delete(id);
      } else {
        if (!multiple) next.clear();
        next.add(id);
      }
      return next;
    });
  };

  return (
    <AccordionContext value={{ openItems, toggle }}>
      {children}
    </AccordionContext>
  );
}

Accordion.Item = function Item({ id, children }) {
  const { openItems } = useContext(AccordionContext);
  return (
    <ItemContext value={{ id, isOpen: openItems.has(id) }}>
      <div>{children}</div>
    </ItemContext>
  );
};

Accordion.Trigger = function Trigger({ children }) {
  const item = useContext(ItemContext);
  const { toggle } = useContext(AccordionContext);
  return (
    <button onClick={() => toggle(item.id)} aria-expanded={item.isOpen}>
      {children}
    </button>
  );
};

Accordion.Content = function Content({ children }) {
  const item = useContext(ItemContext);
  return item.isOpen ? <div>{children}</div> : null;
};
```

**Pros:**

- Caller controls markup structure
- Type-safe (named children types)
- Encapsulated state
- Reusable

**Cons:**

- More boilerplate than simple props
- Context overhead
- Misuse possible (component outside parent)

**Slot-based comparison:**

```tsx
// Slots — explicit positions
<Tabs
  list={<TabsList>...</TabsList>}
  panels={[<TabPanel>...</TabPanel>]}
/>

// Compound — flexible, more natural
<Tabs>
  <Tabs.List>...</Tabs.List>
  <Tabs.Panel>...</Tabs.Panel>
</Tabs>
```

Compound — more JSX-like, flexible.

**Sub-component naming:**

```tsx
// Pattern 1: namespace (Tabs.List)
Tabs.List = ...
// Cons: Tabs.List can't be tree-shaken if Tabs imported

// Pattern 2: named exports (TabsList)
export const TabsList = ...
// Pros: tree-shaking
```

Modern libraries prefer named exports.

**State exposure:**

```tsx
// Optional: expose state via render prop in parent
function Tabs({ children }: { children: ReactNode | ((state: TabsState) => ReactNode) }) {
  const [active, setActive] = useState(...);

  if (typeof children === "function") {
    return <TabsContext value={...}>{children({ active })}</TabsContext>;
  }

  return <TabsContext value={...}>{children}</TabsContext>;
}

// Usage
<Tabs>
  {({ active }) => <p>Active: {active}</p>}
</Tabs>
```

Hybrid: compound components + render prop.

**`asChild` pattern (Radix):**

```tsx
<Tabs.Trigger asChild>
  <CustomButton>Click</CustomButton>
</Tabs.Trigger>

// asChild — Trigger's behavior on CustomButton (no wrapper)
```

Composition with custom rendering.

</details>

### Edge Cases

- **Compound child outside parent**: Throws (useContext null check).
- **Parent without children**: Renders empty.
- **`React.Children.toArray` for filtering**: Strip non-Tab children.

### Follow-up savollar

- "Tabs.List vs separate export?" — Tree-shaking concern. Modern: named exports.
- "Compound vs single component with config prop?" — Compound: flexible markup. Single: simpler API. Use case dictates.
- "Compound + memo?" — Each subcomponent memo'd if expensive. Parent re-renders on state — children may re-render.

</details>

---

### 28. `React.Children` API + `cloneElement` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`React.Children`** — children'ni iterate qilish uchun utility methods: `Children.map`, `Children.forEach`, `Children.count`, `Children.toArray`, `Children.only`. **`cloneElement(element, newProps)`** — element'ni clone qiladi yangi props bilan. Pre-Hooks compound components pattern. Modern: Context preferred.

### Kod misoli

```tsx
import { Children, cloneElement, isValidElement, ReactNode, ReactElement } from "react";

// Children.map — transform each child
function NumberedList({ children }: { children: ReactNode }) {
  return (
    <ol>
      {Children.map(children, (child, i) => (
        <li>
          <span>#{i + 1}</span> {child}
        </li>
      ))}
    </ol>
  );
}

<NumberedList>
  <p>First</p>
  <p>Second</p>
  <p>Third</p>
</NumberedList>

// Children.count
function Stat({ children }: { children: ReactNode }) {
  return <p>Total: {Children.count(children)}</p>;
}

// Children.toArray — flatten to array (auto keys)
function ReverseList({ children }: { children: ReactNode }) {
  return <>{Children.toArray(children).reverse()}</>;
}

// cloneElement — inject props into children
function FormGroup({ children, error }: { children: ReactNode; error?: string }) {
  return Children.map(children, child => {
    if (isValidElement(child)) {
      return cloneElement(child as ReactElement<any>, { className: error ? "error" : "" });
    }
    return child;
  });
}

<FormGroup error="Invalid">
  <input name="x" />
</FormGroup>
// → <input name="x" className="error" />
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`Children` API:**

```typescript
namespace React {
  namespace Children {
    function map<T, C>(children: C, fn: (child: C, index: number) => T): T[];
    function forEach<C>(children: C, fn: (child: C, index: number) => void): void;
    function count(children: any): number;
    function only<C>(children: C): C;  // throws if more than 1
    function toArray(children: any): ReactChild[];
  }
}
```

**`cloneElement`:**

```typescript
function cloneElement<P>(
  element: ReactElement<P>,
  props?: Partial<P>,
  ...children: ReactNode[]
): ReactElement<P>;
```

**`isValidElement`:**

```typescript
function isValidElement<P>(object: any): object is ReactElement<P>;
```

**Common patterns:**

```tsx
// 1. Inject props
const enhanced = Children.map(children, child => {
  if (isValidElement(child)) {
    return cloneElement(child, { className: "added" });
  }
  return child;
});

// 2. Filter children
const onlyButtons = Children.toArray(children).filter(child =>
  isValidElement(child) && child.type === Button
);

// 3. Count specific type
const buttonCount = Children.toArray(children).filter(child =>
  isValidElement(child) && child.type === Button
).length;

// 4. Reorder
const reversed = Children.toArray(children).reverse();
```

**Why `Children.map` over `array.map`:**

```tsx
// Why not just children.map()?
{children.map(child => ...)}  // ❌ children may not be array

// Children can be:
// - undefined / null (no children)
// - single element
// - array (har xil chuqurlikda)
// - string / number (text)
// - boolean (render output'da ko'rinmaydi)

// Children.map handles all cases
// MUHIM: Children.toArray va Children.map null/undefined/boolean children'larni
// natijadan filter qiladi. Children.count esa null/false'ni hisobga olishi mumkin —
// aniq behavior React versiyasi va children shaklingiga bog'liq.
Children.map(children, fn)  // safe — null/undefined/boolean'larni avtomatik filter qiladi
```

**`Children.toArray` — auto keys:**

```tsx
// Children.toArray adds suffix to keys
const arr = Children.toArray(children);
// Each element has key: original key + ".$index"

// vs raw array — keys must be unique
```

**`Children.only`:**

```tsx
function Wrapper({ children }: { children: ReactNode }) {
  const child = Children.only(children);  // throws if not exactly 1
  return <div>{child}</div>;
}

<Wrapper>
  <p>Single</p>
</Wrapper>  // OK

<Wrapper>
  <p>One</p>
  <p>Two</p>
</Wrapper>  // ❌ throws
```

Used for "must be single element" components (forwardRef wrappers).

**`cloneElement` use cases:**

```tsx
// 1. Compound component prop injection
function Tabs({ children, value }: Props) {
  return Children.map(children, child => {
    if (isValidElement(child) && child.type === Tab) {
      return cloneElement(child, { active: child.props.value === value });
    }
    return child;
  });
}

// 2. Wrapper that adds behavior
function Hoverable({ children }: { children: ReactNode }) {
  const [hovered, setHovered] = useState(false);

  return Children.map(children, child => {
    if (isValidElement(child)) {
      return cloneElement(child, {
        onMouseEnter: () => setHovered(true),
        onMouseLeave: () => setHovered(false),
        "data-hovered": hovered,
      });
    }
    return child;
  });
}

// 3. Polymorphic with extra props
function Slot({ children, ...props }: SlotProps) {
  if (isValidElement(children)) {
    return cloneElement(children, { ...props, ...children.props });
  }
  return null;
}
```

**Modern Context alternative:**

```tsx
// cloneElement-based
function Tabs({ children, value }) {
  return Children.map(children, child =>
    cloneElement(child, { active: ..., onSelect: ... })
  );
}

// Context-based (modern)
function Tabs({ children, value, onChange }) {
  return (
    <TabsContext value={{ value, onChange }}>
      {children}
    </TabsContext>
  );
}
function Tab({ value }) {
  const { value: active, onChange } = useContext(TabsContext);
  return <button onClick={() => onChange(value)}>{value}</button>;
}
```

Context: more flexible (any depth), no cloneElement boilerplate.

**`cloneElement` issues:**

- Lost component identity (memo bypassed)
- Type system issues (props injected without TS knowing)
- Nested children not affected
- Performance: extra element creation

**Modern preference: Context > cloneElement.**

**Children traversal performance:**

```tsx
// O(N) traversal
Children.map(children, fn);

// For large lists, consider virtualization
```

**`Children.toArray` keys:**

```tsx
const arr = Children.toArray(
  <>
    <p key="a">A</p>
    <p key="b">B</p>
  </>
);
// arr[0].key = ".$a"  (prefixed)
// arr[1].key = ".$b"
```

Useful for sorting/filtering while preserving keys.

</details>

### Edge Cases

- **`Children.map` returns null**: child filtered out.
- **`cloneElement` overrides existing prop**: New props win.
- **`Children.only` with no children**: Throws.

### Follow-up savollar

- "Children.map vs JSX map?" — Children.map handles non-array children safely.
- "cloneElement deprecation?" — Rasmiy deprecated emas, lekin React docs "legacy API" sifatida ko'rsatadi va Context-based alternative tavsiya qiladi. Yangi kod'da ishlatmaslik afzal.
- "When `Children.toArray`?" — Filtering, sorting, counting specific children types.

</details>

---

### 29. Modern Compound vs Children API [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Modern compound components** — Context-based (state share, flexibility). **Children API + cloneElement** — legacy compound (R16 era). Modern preferred: clearer data flow, type-safe, no cloneElement issues, deeply nested children OK. Children API hali kerak: list filtering, forced single child, prop injection in specific cases.

### Comparison

| Aspect | Modern (Context) | Legacy (cloneElement) |
|--------|------------------|------------------------|
| State share | Context | cloneElement props |
| Nesting depth | Any | Direct children only |
| TypeScript | Excellent | Limited (injected props hidden) |
| memo compatibility | Yes | Breaks (cloned element new ref) |
| Boilerplate | Provider + useContext | Children.map + cloneElement |
| Use case | Most | Specific (slot patterns) |

### Kod misoli

```tsx
// Legacy — cloneElement
function Form({ children, errors }: Props) {
  return Children.map(children, (child) => {
    if (isValidElement(child) && child.type === Input) {
      const error = errors[child.props.name];
      return cloneElement(child, { error });
    }
    return child;
  });
}

<Form errors={{ email: "Invalid" }}>
  <Input name="email" />
  <Input name="password" />
</Form>
// Inputs get error prop based on name

// Modern — Context
const FormContext = createContext<{ errors: Record<string, string> }>({ errors: {} });

function Form({ children, errors }: Props) {
  return (
    <FormContext value={{ errors }}>
      {children}
    </FormContext>
  );
}

function Input({ name }: { name: string }) {
  const { errors } = useContext(FormContext);
  const error = errors[name];
  return (
    <div>
      <input name={name} className={error ? "error" : ""} />
      {error && <p>{error}</p>}
    </div>
  );
}

// Same usage, but flexible nesting
<Form errors={{ email: "Invalid" }}>
  <fieldset>  {/* nested OK */}
    <Input name="email" />
  </fieldset>
</Form>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why Context wins:**

1. **Deep nesting** — Context propagates any depth. cloneElement only direct children.

```tsx
// cloneElement — broken with nesting
<Tabs>
  <div>
    <Tab value="a">A</Tab>  {/* Tabs.cloneElement doesn't reach this */}
  </div>
</Tabs>

// Context — works
<Tabs value="a">
  <div>
    <Tab value="a">A</Tab>  {/* Tab uses useContext */}
  </div>
</Tabs>
```

2. **TypeScript** — Context value type-safe. cloneElement injects props without TS knowing.

3. **`React.memo`** — cloneElement creates new element, breaks memo. Context propagation invalidates memo only for consumers.

4. **DevTools** — Context shows in tree. cloneElement injected props invisible.

**When cloneElement still useful:**

```tsx
// Slot pattern (Radix)
function Slot({ children, ...props }: SlotProps) {
  if (isValidElement(children)) {
    return cloneElement(children, {
      ...mergeProps(props, children.props),  // merge handlers
    });
  }
  return null;
}

// Usage
<Tooltip>
  <Slot>
    <button onClick={existingHandler}>Click</button>
  </Slot>
</Tooltip>
// Tooltip injects its props (like onMouseOver) into button
// Without wrapping
```

`asChild` pattern (Radix UI) — uses cloneElement.

**Compound + Slot hybrid:**

```tsx
// Radix-like
<Tooltip.Root>
  <Tooltip.Trigger asChild>
    <button>Hover me</button>
  </Tooltip.Trigger>
  <Tooltip.Content>
    Tooltip text
  </Tooltip.Content>
</Tooltip.Root>

// Trigger uses Slot (asChild) — no wrapper around button
// Trigger's behavior (onMouseOver, etc.) merged into button's existing props
```

**`Children.map` for compound validation:**

```tsx
function Tabs({ children }: { children: ReactNode }) {
  // Validate only Tab children
  const validChildren = Children.toArray(children).filter(
    child => isValidElement(child) && child.type === Tab
  );

  return (
    <TabsContext value={...}>
      {validChildren}
    </TabsContext>
  );
}
```

**Performance:**

| Pattern | Cost |
|---------|------|
| Children.map + cloneElement | O(N) per render — element creation |
| Context | O(1) provider, O(N) consumer re-render on value change |

Both negligible for small N.

**Library examples:**

- **Radix UI**: Context + Slot pattern (asChild)
- **Headless UI**: Context-based
- **React Router (v6)**: Context-based
- **Reach UI (deprecated)**: Was Context-based

**Migration:**

```tsx
// Before — cloneElement
function Tabs({ children, value }) {
  return Children.map(children, child =>
    cloneElement(child, { active: child.props.value === value })
  );
}

// After — Context
function Tabs({ children, value }) {
  return <TabsContext value={value}>{children}</TabsContext>;
}

function Tab({ value, children }) {
  const active = useContext(TabsContext);
  return <button data-active={active === value}>{children}</button>;
}
```

**Combine when needed:**

```tsx
function Form({ children, errors }) {
  return (
    <FormContext value={{ errors }}>
      {Children.map(children, (child, i) => (
        <FormField key={i}>{child}</FormField>
      ))}
    </FormContext>
  );
}
// Context for shared state + Children.map for layout
```

</details>

### Edge Cases

- **`cloneElement` with key**: Preserves original key unless overridden.
- **Context default**: When no Provider, default value used.
- **Mixed compound and regular children**: Context-based handles arbitrary mix.

### Follow-up savollar

- "When use Children.map?" — List filtering by type, validating compound children, layout transformations.
- "Slot pattern advantages?" — Avoid wrapper div, merge props naturally.
- "Library best practice?" — Modern: Context + Slot (Radix-style) + named exports.

</details>

---

### 30. `react-error-boundary` library — hooks-based reset [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`react-error-boundary` — popular library: `<ErrorBoundary>` wrapper with reset support. Function-component friendly hook `useErrorBoundary` (v4+: `showBoundary` method). Reset patterns: `resetKeys` (array — change reset boundary), `onReset` callback, manual `resetErrorBoundary` from fallback. Class boundary (vanilla React) — no built-in reset.

### Kod misoli

```tsx
import { ErrorBoundary, useErrorBoundary } from "react-error-boundary";

function FallbackUI({ error, resetErrorBoundary }: {
  error: Error;
  resetErrorBoundary: () => void;
}) {
  return (
    <div role="alert">
      <h2>Something went wrong</h2>
      <pre>{error.message}</pre>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

function App() {
  return (
    <ErrorBoundary
      FallbackComponent={FallbackUI}
      onReset={() => {
        // Cleanup state
      }}
      resetKeys={[someKey]}  // Auto-reset when key changes
      onError={(error, info) => {
        Sentry.captureException(error);
      }}
    >
      <RiskyComponent />
    </ErrorBoundary>
  );
}

// Hook for triggering errors from event handlers
function MyButton() {
  const { showBoundary } = useErrorBoundary();

  const handleClick = async () => {
    try {
      await riskyAsyncCall();
    } catch (error) {
      showBoundary(error);  // ✅ Caught by boundary
    }
  };

  return <button onClick={handleClick}>Run</button>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why library over vanilla:**

- Vanilla class boundary — no reset, no hooks integration
- `react-error-boundary` — modern API, async handler support, reset patterns

**`resetKeys`:**

```tsx
function MyApp() {
  const [retryCount, setRetryCount] = useState(0);

  return (
    <ErrorBoundary
      resetKeys={[retryCount]}  // boundary resets when retryCount changes
      FallbackComponent={Fallback}
    >
      <Content />
      <button onClick={() => setRetryCount(c => c + 1)}>Retry</button>
    </ErrorBoundary>
  );
}
```

**Async errors handling:**

```tsx
// ❌ Boundary doesn't catch async
async function handleClick() {
  await fetchData();  // throws — uncaught
}

// ✅ showBoundary
function MyButton() {
  const { showBoundary } = useErrorBoundary();
  return (
    <button onClick={async () => {
      try {
        await fetchData();
      } catch (e) {
        showBoundary(e);
      }
    }}>
      Run
    </button>
  );
}

// Or react-error-boundary's withErrorBoundary HOC + handler
```

**R19 root callbacks vs library:**

```tsx
// R19 root-level
const root = createRoot(container, {
  onCaughtError: (error, info) => Sentry.captureException(error),
});
// Plus boundary catches

// Library
<ErrorBoundary
  onError={(error, info) => Sentry.captureException(error)}
  FallbackComponent={Fallback}
>
```

R19 + library — combined for full coverage.

</details>

### Edge Cases

- **Reset clears state — re-throws**: If error source not fixed, boundary re-catches.
- **`resetKeys` with reference type**: Object/array reference — every render new, infinite reset. Use primitives.
- **Nested boundaries**: Inner catches first. If re-thrown, outer catches.

### Follow-up savollar

- "Suspense + ErrorBoundary?" — Suspense catches Promise (loading), ErrorBoundary catches Error. Both useful.
- "When build custom vs use library?" — Library — production. Custom — minimal scope, no async.

</details>

---

<a id="qism-h"></a>

## QISM H: Error Boundaries

### 31. Error boundaries — class komponentlar [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Error boundary** — komponent tree'da xatolarni tutib, fallback UI ko'rsatadigan komponent. **Faqat class komponent** — `getDerivedStateFromError` (error state set) + `componentDidCatch` (logging) lifecycle methods. R19 ham hook-based error boundary API bermagan — class component yagona yo'l. `react-error-boundary` library ichida class ishlatib, hooks-friendly wrapper beradi.

### Kod misoli

```tsx
import { Component, ErrorInfo, ReactNode } from "react";

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

interface ErrorBoundaryProps {
  fallback: (error: Error) => ReactNode;
  children: ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  // Update state on error (render fallback)
  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  // Log error
  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error("Error caught:", error, errorInfo);
    this.props.onError?.(error, errorInfo);
  }

  reset = () => this.setState({ hasError: false, error: null });

  render() {
    if (this.state.hasError && this.state.error) {
      return this.props.fallback(this.state.error);
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary
  fallback={(error) => (
    <div>
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
    </div>
  )}
  onError={(error) => Sentry.captureException(error)}
>
  <App />
</ErrorBoundary>
```

```tsx
// react-error-boundary library (modern)
import { ErrorBoundary } from "react-error-boundary";

function ErrorFallback({ error, resetErrorBoundary }: { error: Error; resetErrorBoundary: () => void }) {
  return (
    <div>
      <p>Error: {error.message}</p>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

<ErrorBoundary FallbackComponent={ErrorFallback} onReset={() => {}}>
  <App />
</ErrorBoundary>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Lifecycle methods:**

```typescript
class ErrorBoundary extends Component {
  // Static — synchronously sets state to render fallback
  static getDerivedStateFromError(error: Error) {
    return { hasError: true };
  }

  // Side effects — logging
  componentDidCatch(error: Error, errorInfo: { componentStack: string }) {
    // Log to monitoring service
    Sentry.captureException(error, { extra: errorInfo });
  }
}
```

**Why class:**

- Hooks haven't replicated `getDerivedStateFromError` (returning state from error)
- React core decision: error boundary needs synchronous state update on render error
- R19+ root callbacks alternative (Q28)

**What error boundaries catch:**

- Render errors
- Lifecycle errors (componentDidMount, componentDidUpdate, etc.)
- Constructor errors
- Children render errors

**What they DON'T catch:**

- Event handlers (use try/catch)
- Async code (setTimeout, Promise) — use error handling
- Server-side rendering errors
- Errors in error boundary itself

**Granular vs global:**

```tsx
// Global
<ErrorBoundary fallback={<Error />}>
  <App />
</ErrorBoundary>

// Granular — each section
<ErrorBoundary fallback={<HeaderError />}>
  <Header />
</ErrorBoundary>
<ErrorBoundary fallback={<MainError />}>
  <Main />
</ErrorBoundary>
```

Granular: error in one section doesn't break entire app.

**Reset patterns:**

```tsx
// Reset via key
function App() {
  const [resetKey, setResetKey] = useState(0);
  return (
    <ErrorBoundary key={resetKey} fallback={...}>
      <Component />
    </ErrorBoundary>
  );
}

// User clicks "Retry"
const retry = () => setResetKey(k => k + 1);
// → ErrorBoundary remounts, hasError = false
```

`react-error-boundary` provides `resetErrorBoundary` automatically.

**Error handling outside boundaries:**

```tsx
// Event handler
const handleClick = () => {
  try {
    riskyOperation();
  } catch (error) {
    setError(error);
    Sentry.captureException(error);
  }
};

// Async
useEffect(() => {
  let cancelled = false;
  fetchData()
    .then(data => { if (!cancelled) setData(data); })
    .catch(error => {
      if (!cancelled) setError(error);
    });
  return () => { cancelled = true; };
}, []);
```

**Logging:**

```tsx
componentDidCatch(error, errorInfo) {
  // Custom logger
  log({
    error: error.message,
    stack: error.stack,
    componentStack: errorInfo.componentStack,
    timestamp: Date.now(),
    url: window.location.href,
    user: getCurrentUser(),
  });
}
```

**Production patterns:**

```tsx
// App.tsx
<ErrorBoundary fallback={<CrashScreen />}>
  <Router>
    {/* Per-route boundaries */}
    <Route path="/dashboard">
      <ErrorBoundary fallback={<DashboardError />}>
        <Dashboard />
      </ErrorBoundary>
    </Route>
  </Router>
</ErrorBoundary>
```

**`react-error-boundary` features (v4+):**

- Hooks-friendly wrapper
- `useErrorBoundary` hook (`showBoundary` — async error'larni nearest boundary'ga yo'naltirish)
- `withErrorBoundary` HOC

```tsx
import { useErrorBoundary } from "react-error-boundary";

function AsyncComponent() {
  const { showBoundary } = useErrorBoundary();

  const fetchData = async () => {
    try {
      const data = await api.get();
      // ...
    } catch (error) {
      showBoundary(error);  // throw to nearest boundary
    }
  };
}
```

**Class component minimum primer:**

```tsx
class MyClass extends React.Component<Props, State> {
  // State as class field
  state = { count: 0 };

  // Lifecycle
  componentDidMount() {
    // ≈ useEffect(() => {}, []) mount portion
  }

  componentDidUpdate(prevProps: Props, prevState: State) {
    // ≈ useEffect deps comparison
  }

  componentWillUnmount() {
    // ≈ useEffect cleanup return
  }

  // Render (required)
  render() {
    return <div>{this.state.count}</div>;
  }
}
```

For error boundaries — class still required.

</details>

### Edge Cases

- **Error during error boundary render**: Propagates to parent boundary.
- **Error in `componentDidCatch`**: Propagates upward (don't throw in catch).
- **Async error**: Not caught. Use try/catch in async code.

### Follow-up savollar

- "Hook-based error boundary?" — Not in React core (R19). `react-error-boundary` library uses class internally.
- "SSR errors?" — `react-dom/server` has different handling. R19 `onError` callback in renderToReadableStream.
- "Error boundary best practices?" — Granular boundaries, log to monitoring, provide retry UI.

</details>

---

### 32. Error boundary scope — render/lifecycle vs events/async [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Error boundary'lar **faqat render**, **lifecycle**, va **constructor** errorlarini tutadi. **Catch qilmaydi**: (1) **event handlers** (try/catch ishlatish), (2) **async code** (setTimeout, fetch promise) — manual error handling, (3) **SSR** — alohida mexanizm, (4) **error boundary itself** errors.

### Kod misoli

```tsx
function BuggyComponent() {
  // ✅ Caught — render error
  if (Math.random() > 0.5) throw new Error("Render error");

  return (
    <button onClick={() => {
      // ❌ NOT caught — event handler
      throw new Error("Click error");
    }}>
      Click
    </button>
  );
}

// ✅ Wrap in try/catch for events
function FixedButton() {
  const [error, setError] = useState<Error | null>(null);

  const handleClick = () => {
    try {
      riskyOperation();
    } catch (err) {
      setError(err as Error);
    }
  };

  if (error) throw error;  // re-throw to error boundary

  return <button onClick={handleClick}>Click</button>;
}
```

```tsx
// Async — not caught
function AsyncBug() {
  useEffect(() => {
    setTimeout(() => {
      throw new Error("Timeout error");  // ❌ not caught by boundary
    }, 1000);
  }, []);
}

// Fix — re-throw via state
function FixedAsync() {
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    setTimeout(() => {
      try {
        riskyAsync();
      } catch (err) {
        setError(err as Error);
      }
    }, 1000);
  }, []);

  if (error) throw error;  // ✅ caught by boundary

  return <div>Content</div>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why event handlers not caught:**

Event handlers run **outside React's render phase**. React doesn't wrap them in try/catch. If error throws, it propagates to native event loop (uncaught).

```typescript
// React internal — event dispatch
function dispatchEvent(nativeEvent) {
  const handlers = collectHandlers(nativeEvent.target);
  handlers.forEach(handler => {
    handler(nativeEvent);  // no try/catch
  });
}
```

**Why async not caught:**

Async code (Promise, setTimeout) — outside React's call stack. React doesn't manage these. Error must be handled where async code runs.

**Pattern: re-throw to boundary:**

```tsx
function useAsyncError() {
  const [, setError] = useState();
  return useCallback((error: Error) => {
    setError(() => { throw error; });
  }, []);
}

function Component() {
  const throwAsyncError = useAsyncError();

  const fetchData = async () => {
    try {
      await api.get();
    } catch (error) {
      throwAsyncError(error as Error);
    }
  };

  return <button onClick={fetchData}>Fetch</button>;
}
```

`useState`'s setter accepts function — calling it during render propagates throw.

**`useErrorBoundary` from `react-error-boundary` (v4+):**

```tsx
import { useErrorBoundary } from "react-error-boundary";

function AsyncComponent() {
  const { showBoundary } = useErrorBoundary();

  const fetchData = async () => {
    try {
      await api.get();
    } catch (error) {
      showBoundary(error);
    }
  };
}
```

`showBoundary` error'ni nearest boundary'ga yo'naltiradi — keyingi render'da boundary catch qiladi.

**Error in different scenarios:**

```tsx
// 1. Render — caught
function A() {
  throw new Error("Render");
}

// 2. Lifecycle — caught
class B extends Component {
  componentDidMount() {
    throw new Error("Mount");  // caught
  }
}

// 3. Constructor — caught
class C extends Component {
  constructor(props) {
    super(props);
    throw new Error("Constructor");  // caught
  }
}

// 4. Event handler — NOT caught
<button onClick={() => { throw new Error("Click"); }} />

// 5. Async — NOT caught
useEffect(() => {
  setTimeout(() => { throw new Error("Async"); }, 100);
});

// 6. Promise rejection — NOT caught (unhandled)
Promise.reject(new Error("Reject"));
```

**Promise rejection handler (window-level):**

```tsx
useEffect(() => {
  const handler = (e: PromiseRejectionEvent) => {
    console.error("Unhandled rejection:", e.reason);
    Sentry.captureException(e.reason);
  };
  window.addEventListener("unhandledrejection", handler);
  return () => window.removeEventListener("unhandledrejection", handler);
}, []);
```

Catches unhandled promise rejections globally.

**`window.onerror`:**

```tsx
useEffect(() => {
  window.onerror = (msg, src, line, col, error) => {
    console.error("Global error:", error);
    Sentry.captureException(error);
  };
}, []);
```

Catches uncaught synchronous errors globally (event handlers).

**Async error patterns:**

```tsx
// Pattern 1: Try/catch in handler
const handleClick = async () => {
  try {
    await api.call();
  } catch (error) {
    setError(error as Error);
  }
};

// Pattern 2: useAsyncError throw
const throwError = useAsyncError();
const handleClick = async () => {
  try {
    await api.call();
  } catch (error) {
    throwError(error as Error);
  }
};

// Pattern 3: TanStack Query
const { error } = useQuery({ queryKey: ["data"], queryFn: api.get });
if (error) throw error;  // re-throw to boundary
```

**`useEffect` errors:**

```tsx
// ❌ Throws to global handler
useEffect(() => {
  throw new Error("Effect");  // not caught
});

// ✅ Set state, throw on render
const [effectError, setEffectError] = useState<Error | null>(null);

useEffect(() => {
  try {
    riskyOperation();
  } catch (error) {
    setEffectError(error as Error);
  }
}, []);

if (effectError) throw effectError;  // re-throws on render
```

**Suspense boundaries vs error boundaries:**

- Suspense — handles **promise** thrown (loading state)
- Error boundary — handles **error** thrown
- Both compose

```tsx
<ErrorBoundary fallback={<Error />}>
  <Suspense fallback={<Spinner />}>
    <AsyncComponent />
  </Suspense>
</ErrorBoundary>

// AsyncComponent suspends → Suspense fallback
// AsyncComponent errors → ErrorBoundary fallback
```

</details>

### Edge Cases

- **Error during getDerivedStateFromError**: Propagates to parent boundary.
- **componentDidCatch with side effect**: Logged but doesn't affect render.
- **Error in fallback render**: Propagates to parent boundary.

### Follow-up savollar

- "Why React doesn't wrap event handlers?" — Performance. Most events don't error. try/catch overhead.
- "Async error best practice?" — Library (TanStack Query) handles. Manual: try/catch + state + throw on render.
- "Production error logging?" — Sentry, LogRocket, Datadog. Hook into `componentDidCatch` and `unhandledrejection`.

</details>

---

### 33. R19 root error callbacks [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**R19** `createRoot`/`hydrateRoot`'da error reporting callback'lari to'liqlandi: **`onCaughtError`** (R19 yangi) — error boundary tomonidan tutilgan errorlar, **`onUncaughtError`** (R19 yangi) — boundarysiz "qochgan" errorlar, **`onRecoverableError`** (R18'dan beri mavjud) — recovery bo'lgan errorlar (asosan hydration mismatch). Production monitoring uchun ideal — barcha React errorlarni global capture qilish.

### Kod misoli

```tsx
import { createRoot } from "react-dom/client";

const container = document.getElementById("root");
if (!container) throw new Error("Root element not found");

const root = createRoot(container, {
  // Caught by error boundary
  onCaughtError: (error, errorInfo) => {
    console.error("Caught:", error);
    Sentry.captureException(error, {
      tags: { type: "react-caught" },
      extra: errorInfo,
    });
  },

  // Uncaught — escaped to root
  onUncaughtError: (error, errorInfo) => {
    console.error("Uncaught (app crashed):", error);
    Sentry.captureException(error, {
      tags: { type: "react-uncaught" },
      level: "fatal",
      extra: errorInfo,
    });
    // Show full-page error UI
    showAppCrashScreen();
  },

  // Recoverable (e.g., hydration mismatch)
  onRecoverableError: (error, errorInfo) => {
    console.warn("Recovered:", error);
    Sentry.captureMessage("Hydration recovered", {
      level: "warning",
      extra: { error, errorInfo },
    });
  },
});

root.render(<App />);
```

<details>
<summary><strong>Deep Dive</strong></summary>

**3 callback comparison:**

| Callback | When fires | Severity |
|----------|------------|----------|
| `onCaughtError` | Error caught by `<ErrorBoundary>` | Medium (handled) |
| `onUncaughtError` | Error escaped all boundaries | High (app may crash) |
| `onRecoverableError` | Hydration mismatch, recovery succeeded | Low (warning) |

**`onCaughtError` — boundary caught:**

```tsx
class Boundary extends Component {
  componentDidCatch(error, info) {
    // local handling
  }
  // ...
}

<Boundary>
  <ComponentThatThrows />
</Boundary>

// Both fire:
// 1. Boundary.componentDidCatch
// 2. Root.onCaughtError
```

**`onUncaughtError` — no boundary:**

```tsx
// No <ErrorBoundary> in tree
function App() {
  throw new Error("oops");  // escapes to root
}

// Fires onUncaughtError
// React unmounts entire tree (component-level recovery impossible)
```

**`onRecoverableError` — auto-recovered:**

Triggered when:
- Hydration mismatch (server HTML differs from client expected)
- Render bailout to client-side render
- Other "recovered" scenarios

```tsx
hydrateRoot(container, <App />, {
  onRecoverableError: (error, errorInfo) => {
    if (error.message.includes("Hydration")) {
      // Hydration mismatch — subtree fell back to client render
      // App functional, but UX worse (no SSR for that subtree)
      logToMonitoring("hydration-mismatch", { error });
    }
  },
});
```

**Production monitoring pattern:**

```tsx
import * as Sentry from "@sentry/react";

const root = createRoot(container, {
  onCaughtError: (error, errorInfo) => {
    Sentry.captureException(error, {
      tags: { boundary: "caught" },
      extra: { componentStack: errorInfo.componentStack },
    });
  },
  onUncaughtError: (error, errorInfo) => {
    Sentry.captureException(error, {
      tags: { boundary: "uncaught" },
      level: "fatal",
      extra: { componentStack: errorInfo.componentStack },
    });
  },
  onRecoverableError: (error, errorInfo) => {
    Sentry.captureMessage("Recoverable error", {
      level: "warning",
      extra: { error, errorInfo },
    });
  },
});
```

**Error info shape:**

```typescript
// Root callback'lar uchun errorInfo
type ErrorInfo = {
  componentStack: string;  // React component stack trace
};

// SSR onError — digest return qiladi, client'da error.digest sifatida keladi
// hydrateRoot onRecoverableError'da error object'da digest bo'lishi mumkin
```

**`digest` — server-client error correlation (SSR):**

```typescript
// Server SSR error — digest return qilinadi
const stream = await renderToReadableStream(<App />, {
  onError: (error) => {
    console.error("SSR error", error);
    return "abc123";  // digest — client'ga uzatiladi
  },
});

// Client hydration — onRecoverableError'da error.digest orqali match
hydrateRoot(container, <App />, {
  onRecoverableError: (error) => {
    // error.digest === "abc123" (server'dan kelgan)
    correlateErrors(error);
  },
});
```

**Hooks alternative (in plans):**

R19 introduced root callbacks. Future may add hooks (`useErrorBoundary` discussed). Hook-based error boundaries — community library `react-error-boundary`.

**Per-boundary callbacks:**

```tsx
class ErrorBoundary extends Component {
  componentDidCatch(error, info) {
    this.props.onError?.(error, info);  // local
  }
  // ...
}

<ErrorBoundary onError={(error) => log("local", error)}>
  <Component />
</ErrorBoundary>

// Root callback (in createRoot) — global
// Boundary callback — local
// Both fire for caught errors
```

**SSR errors (different):**

```tsx
// renderToReadableStream onError
const stream = await renderToReadableStream(<App />, {
  onError: (error) => {
    Sentry.captureException(error);
    return "abc123";  // returned digest sent to client
  },
});
```

**Migration from R18:**

```tsx
// R18 — faqat onRecoverableError mavjud edi
const root = createRoot(container, {
  onRecoverableError,
});

// R19 — qo'shildi: onCaughtError, onUncaughtError
const root = createRoot(container, {
  onCaughtError, onUncaughtError, onRecoverableError
});
```

**`identifierPrefix` related:**

```tsx
const root = createRoot(container, {
  identifierPrefix: "myapp-",
  onCaughtError, ...
});
```

Multiple options for advanced use cases.

</details>

### Edge Cases

- **All 3 callbacks for same error**: `onCaughtError` (if caught) AND `onRecoverableError` (if recovered) — separate scenarios.
- **Callback throws**: Caught silently, doesn't crash app.
- **Server vs client digests**: Match via opaque hash.

### Follow-up savollar

- "All apps need root callbacks?" — Production: yes. Dev: console may suffice.
- "vs componentDidCatch?" — componentDidCatch local. Root callback global.
- "Hook for boundary errors?" — `react-error-boundary` library. R19 core: only root callbacks.

</details>

---

<a id="qism-i"></a>

## QISM I: Portals

### 34. `createPortal` + event bubbling [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`createPortal(children, container)`** — child'larni boshqa DOM node'ga (parent dan tashqari) render qiladi. Use case: modals, tooltips, dropdowns, notifications. **Event bubbling React tree bo'ylab** ishlaydi (DOM tree emas) — portal child event'lari React parent'ga bubble qiladi.

### Kod misoli

```tsx
import { createPortal } from "react-dom";

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  children: React.ReactNode;
}

function Modal({ isOpen, onClose, children }: ModalProps) {
  if (!isOpen) return null;

  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.body  // render to body, not parent
  );
}

function Page() {
  const [open, setOpen] = useState(false);

  return (
    <div onClick={() => console.log("page clicked")}>
      <button onClick={() => setOpen(true)}>Open</button>
      <Modal isOpen={open} onClose={() => setOpen(false)}>
        <p>Modal content</p>
        <button onClick={() => console.log("modal button")}>Modal Button</button>
      </Modal>
    </div>
  );
}

// Click "Modal Button":
// DOM bubble: button → modal-content → modal-overlay → body
// React bubble: button → Modal → Page (via React tree)
// "modal button" + "page clicked" both logged!
```

<details>
<summary><strong>Deep Dive</strong></summary>

**DOM tree vs React tree:**

```
DOM tree:
  body
    ├─ #root
    │   └─ Page
    │       └─ <button>Open</button>
    └─ Modal Overlay (portal target)
         └─ <button>Modal Button</button>

React tree:
  Page
    ├─ button (Open)
    └─ Modal (creates portal)
         └─ button (Modal Button)
```

Click "Modal Button":
- **DOM bubble**: button → overlay → body (DOM hierarchy)
- **React bubble**: button → Modal → Page (React fiber tree)

**Why portal events bubble in React tree:**

React event delegation (root container) catches event at native level (DOM bubble). Then walks React fiber tree (which has portal as child of original parent), invoking handlers in React-tree order.

```typescript
// React internal
function dispatchEvent(nativeEvent) {
  const target = nativeEvent.target;  // DOM element
  const fiber = getFiberFromDOM(target);

  // Walk React tree (return pointer)
  const handlers = [];
  let current = fiber;
  while (current) {
    if (current.props.onClick) {
      handlers.push(current);
    }
    current = current.return;  // React parent (not DOM parent!)
  }

  // Invoke handlers in React-tree bubble order
  for (const handler of handlers) {
    if (event.isPropagationStopped()) break;
    handler.props.onClick(event);
  }
}
```

**Portal use cases:**

```tsx
// 1. Modal
<Modal>...</Modal>  // renders to body

// 2. Tooltip
<Tooltip>Hover info</Tooltip>  // renders absolute, top of DOM

// 3. Notification toast
<Toast>...</Toast>  // overlay at top

// 4. Dropdown menu (above scroll containers)
<Dropdown>...</Dropdown>

// 5. Drag-and-drop ghost
```

**Why portals needed:**

```css
/* CSS issues without portal */
.parent {
  overflow: hidden;     /* clips children */
  position: relative;
  z-index: 1;          /* nested z-index issues */
}

.modal {
  position: absolute;  /* but clipped by parent! */
  z-index: 9999;       /* but limited by parent's stacking context */
}
```

Portal renders outside parent → escapes CSS constraints.

**Portal target — DOM node:**

```tsx
// Target must exist in DOM
const portalTarget = document.getElementById("modal-root");

if (portalTarget) {
  createPortal(<Modal />, portalTarget);
}

// Or document.body (always exists)
createPortal(<Modal />, document.body);
```

**SSR considerations:**

```tsx
function Modal({ children }: Props) {
  // ❌ SSR — document undefined
  return createPortal(children, document.body);
}

// ✅ Guard
function Modal({ children }: Props) {
  if (typeof document === "undefined") return null;
  return createPortal(children, document.body);
}

// ✅ Or only render after mount
function Modal({ children }: Props) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);
  if (!mounted) return null;
  return createPortal(children, document.body);
}
```

**Multiple portals:**

```tsx
function App() {
  return (
    <>
      <Toolbar />
      <Modal>...</Modal>          {/* portal 1 to body */}
      <Tooltip>...</Tooltip>       {/* portal 2 to body */}
      <Notification>...</Notification>  {/* portal 3 to body */}
    </>
  );
}

// All portals render to body, alongside main #root
```

**Cleanup on unmount:**

```tsx
function Modal({ isOpen, children }: Props) {
  if (!isOpen) return null;
  return createPortal(children, document.body);
}

// isOpen=false: createPortal not called, content removed from DOM
// React handles unmount automatically
```

**Custom portal container:**

```tsx
function PortalRoot() {
  return <div id="portal-root" />;
}

function App() {
  return (
    <>
      <PortalRoot />  {/* dedicated portal target */}
      <MainApp />
    </>
  );
}

// Modal renders to #portal-root
function Modal({ children }: Props) {
  const target = document.getElementById("portal-root");
  if (!target) return null;
  return createPortal(children, target);
}
```

**Z-index management:**

```tsx
// Portal escapes parent stacking context
// Manage z-index globally:

const Z_INDEX = {
  modal: 1000,
  tooltip: 2000,
  notification: 3000,
};

<div style={{ zIndex: Z_INDEX.modal }}>...</div>
```

**`stopPropagation` in portal:**

```tsx
<Page onClick={handlePageClick}>
  <Modal>
    <button onClick={(e) => {
      e.stopPropagation();  // Stops React bubble
      handleModalClick();
    }}>
      Modal Action
    </button>
  </Modal>
</Page>

// Without stopPropagation:
// handleModalClick runs
// handlePageClick runs (React bubble)

// With stopPropagation:
// Only handleModalClick
```

</details>

### Edge Cases

- **Portal target removed**: Content disappears. Add to `document.body` for safety.
- **Multiple roots**: Each root has own delegation. Portal in one root's tree but DOM in another's container — events handled by source root.
- **Stacking context**: Portal escapes parent's, but body's stacking applies.

### Follow-up savollar

- "Portal scrolling within parent?" — No, portal renders absolute. Inherit/copy parent's scroll if needed.
- "Test portals?" — RTL handles via `screen.getByText` (DOM-wide query).
- "Portal performance?" — Same as regular render — single DOM tree, just different parent.

</details>

---

### 35. Portal focus management va a11y [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Modal portal'da focus management** majburiy a11y uchun: (1) **focus trap** — focus modal ichida qoladi (Tab/Shift+Tab cycling), (2) **initial focus** — modal ochilganda first focusable element'ga, (3) **return focus** — modal yopilganda original element'ga, (4) **`aria-modal="true"`**, **`role="dialog"`**, **`aria-labelledby`** atributlar. `focus-trap-react` library yoki manual.

### Kod misoli

```tsx
import { useEffect, useRef } from "react";
import { createPortal } from "react-dom";

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
}

function Modal({ isOpen, onClose, title, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);
  const triggerRef = useRef<HTMLElement | null>(null);
  const titleId = useId();

  useEffect(() => {
    if (!isOpen) return;

    // Save current focus (return focus on close)
    triggerRef.current = document.activeElement as HTMLElement;

    // Focus first focusable in modal
    const firstFocusable = modalRef.current?.querySelector<HTMLElement>(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    firstFocusable?.focus();

    // Escape to close
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === "Escape") {
        onClose();
      }
      if (e.key === "Tab") {
        // Focus trap
        const focusable = modalRef.current?.querySelectorAll<HTMLElement>(
          'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
        );
        if (!focusable || focusable.length === 0) return;

        const first = focusable[0];
        const last = focusable[focusable.length - 1];

        if (e.shiftKey && document.activeElement === first) {
          e.preventDefault();
          last.focus();
        } else if (!e.shiftKey && document.activeElement === last) {
          e.preventDefault();
          first.focus();
        }
      }
    };

    document.addEventListener("keydown", handleKeyDown);
    return () => {
      document.removeEventListener("keydown", handleKeyDown);
      // Return focus to original element
      triggerRef.current?.focus();
    };
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <div
      className="modal-overlay"
      onClick={onClose}
      role="presentation"
    >
      <div
        ref={modalRef}
        className="modal-content"
        role="dialog"
        aria-modal="true"
        aria-labelledby={titleId}
        onClick={(e) => e.stopPropagation()}
        tabIndex={-1}
      >
        <h2 id={titleId}>{title}</h2>
        {children}
        <button onClick={onClose} aria-label="Close">×</button>
      </div>
    </div>,
    document.body
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**a11y requirements for modals:**

1. **Focus management**:
   - Initial focus inside modal
   - Trap focus (Tab/Shift+Tab cycle)
   - Return focus on close

2. **ARIA attributes**:
   - `role="dialog"` or `role="alertdialog"`
   - `aria-modal="true"`
   - `aria-labelledby` (title ID)
   - `aria-describedby` (description ID)

3. **Keyboard**:
   - Escape closes
   - Enter on close button
   - Tab trapped in modal

4. **Screen reader**:
   - Announce modal opening
   - Background content hidden (`aria-hidden="true"` on main app)

**`focus-trap-react` library:**

```tsx
import FocusTrap from "focus-trap-react";

<FocusTrap active={isOpen}>
  <div role="dialog" aria-modal="true">
    {children}
  </div>
</FocusTrap>
```

Handles trap automatically.

**Background content `aria-hidden`:**

```tsx
function Modal({ isOpen, children }: Props) {
  useEffect(() => {
    if (!isOpen) return;

    const main = document.getElementById("main");
    main?.setAttribute("aria-hidden", "true");

    return () => {
      main?.removeAttribute("aria-hidden");
    };
  }, [isOpen]);

  // ...
}
```

Yoki `inert` attribute (HTML standard — Chrome 102+, hozirda barcha asosiy browser'larda):

```tsx
useEffect(() => {
  if (!isOpen) return;
  document.getElementById("main")?.setAttribute("inert", "");
  return () => document.getElementById("main")?.removeAttribute("inert");
}, [isOpen]);
```

`inert` — element'ni a11y tree'dan yashiradi va non-interactive qiladi. **HTML platform feature** (React tomonidan qo'shilmagan), modern modal background uchun `aria-hidden`'dan ko'ra to'g'riroq tanlov.

**Body scroll lock:**

```tsx
useEffect(() => {
  if (!isOpen) return;
  const original = document.body.style.overflow;
  document.body.style.overflow = "hidden";
  return () => { document.body.style.overflow = original; };
}, [isOpen]);
```

Prevents background scroll when modal open.

**Focus on close — `inert` cleanup:**

```tsx
const triggerRef = useRef<HTMLElement | null>(null);

const openModal = () => {
  triggerRef.current = document.activeElement as HTMLElement;
  setIsOpen(true);
};

const closeModal = () => {
  setIsOpen(false);
  // Use setTimeout for after DOM update
  setTimeout(() => triggerRef.current?.focus(), 0);
};
```

**Library: Radix UI Dialog:**

```tsx
import * as Dialog from "@radix-ui/react-dialog";

<Dialog.Root>
  <Dialog.Trigger>Open</Dialog.Trigger>
  <Dialog.Portal>
    <Dialog.Overlay />
    <Dialog.Content>
      <Dialog.Title>Title</Dialog.Title>
      <Dialog.Description>Description</Dialog.Description>
      <Dialog.Close>Close</Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

Radix handles all a11y automatically — production recommendation.

**Tooltip a11y:**

```tsx
function Tooltip({ children, content }: { children: ReactElement; content: string }) {
  const id = useId();

  const triggerWithAria = cloneElement(children, {
    "aria-describedby": id,
  });

  return (
    <>
      {triggerWithAria}
      <Portal>
        <div role="tooltip" id={id}>{content}</div>
      </Portal>
    </>
  );
}
```

**Drag-drop a11y:**

```tsx
function Draggable({ id, children }: Props) {
  return (
    <div
      role="button"
      aria-label={`Drag ${id}`}
      tabIndex={0}
      onKeyDown={(e) => {
        if (e.key === "Enter" || e.key === " ") {
          // Trigger drag start with keyboard
        }
      }}
    >
      {children}
    </div>
  );
}
```

**Notification toast a11y:**

```tsx
<div role="status" aria-live="polite">
  {message}
</div>

// Or for important
<div role="alert" aria-live="assertive">
  {urgentMessage}
</div>
```

`role="status"` — non-urgent. `role="alert"` — urgent (read immediately).

**Test a11y:**

```tsx
// jest-axe
import { axe } from "jest-axe";

test("modal accessible", async () => {
  const { container } = render(<Modal isOpen={true} />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

**Real-world checklist:**

- ✅ `role="dialog"` + `aria-modal="true"`
- ✅ `aria-labelledby` (title ID)
- ✅ `aria-describedby` (if description)
- ✅ Initial focus inside modal
- ✅ Focus trap
- ✅ Return focus on close
- ✅ Escape to close
- ✅ Background `aria-hidden` or `inert`
- ✅ Body scroll lock
- ✅ Click outside to close
- ✅ Animated transitions don't break focus

</details>

### Edge Cases

- **No focusable elements in modal**: Add `tabIndex={-1}` to modal container.
- **Modal triggers another modal**: Stack focus management (Radix handles).
- **Close before mount**: useEffect cleanup runs without setup — guard.

### Follow-up savollar

- "ARIA dialog vs alertdialog?" — Dialog: regular. Alertdialog: requires immediate attention (errors, confirmations).
- "Test focus management?" — RTL `userEvent.tab()` simulates keyboard navigation.
- "iOS Safari focus issues?" — Some Safari versions: focus on modal doesn't work without `tabIndex`. Set `tabIndex={-1}` on dialog.

</details>

---

### 36. Portal SSR — server'da target DOM yo'q [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Server-side render'da `document` yo'q — `createPortal(children, document.body)` xato beradi. SSR-safe pattern: portal target'ni `useEffect` ichida set qilish (client-only mount), `typeof document` guard, yoki "use client" directive (RSC). Hydration paytida portal'lar correctly attach qilinadi.

### Kod misoli

```tsx
// ❌ SSR xato
function BadPortal({ children }: { children: React.ReactNode }) {
  return createPortal(children, document.body);  // document undefined on server
}

// ✅ Client-only mount
function SafePortal({ children }: { children: React.ReactNode }) {
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
  }, []);

  if (!mounted) return null;  // server: nothing
  return createPortal(children, document.body);
}

// ⚠️ typeof guard — server'da ishlaydi, lekin hydration mismatch xavfi bor
// (server: null, client: portal content — React hydration warning beradi)
// useEffect + mounted pattern afzal
function GuardedPortal({ children }: { children: React.ReactNode }) {
  if (typeof document === "undefined") return null;
  return createPortal(children, document.body);
}

// ✅ RSC pattern — Client Component
"use client";
import { createPortal } from "react-dom";

export function ClientPortal({ children }: { children: React.ReactNode }) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);
  if (!mounted) return null;
  return createPortal(children, document.body);
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why server-side issue:**

- Server: no DOM (`document`, `window` undefined)
- `createPortal` needs container DOM node
- React skips portals during `renderToString` (R18+) — but call still throws if we try to access `document`

**Hydration with portals:**

```tsx
// Server renders without portal content
// Client hydrates: portal target is in DOM (e.g., #portal-root)
// Portal mounts content into target
```

**Pre-defined target in HTML:**

```html
<!-- index.html -->
<body>
  <div id="root"></div>
  <div id="portal-root"></div>  <!-- Portal target -->
</body>
```

```tsx
function Modal({ children }: { children: React.ReactNode }) {
  if (typeof document === "undefined") return null;
  const target = document.getElementById("portal-root");
  if (!target) return null;
  return createPortal(children, target);
}
```

**Dynamic portal target:**

```tsx
function PortalToTarget({ targetId, children }: { targetId: string; children: React.ReactNode }) {
  const [target, setTarget] = useState<HTMLElement | null>(null);

  useEffect(() => {
    setTarget(document.getElementById(targetId));
  }, [targetId]);

  if (!target) return null;
  return createPortal(children, target);
}
```

**Library — react-portal:**

```tsx
import { Portal } from "react-portal";

function Modal() {
  return (
    <Portal>
      <div className="modal">...</div>
    </Portal>
  );
}
```

**Next.js App Router:**

```tsx
"use client";
// Portal works in client component
import { createPortal } from "react-dom";

export function ToastPortal({ children }) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);
  return mounted ? createPortal(children, document.body) : null;
}
```

</details>

### Edge Cases

- **Portal in Suspense**: Portal content suspends — Suspense fallback shows in original tree position.
- **Portal event bubbling**: React tree (parent-of-portal) catches events, not DOM tree.
- **Multiple portals same target**: All append in order. No deduplication.

### Follow-up savollar

- "Portal vs absolute positioning?" — Portal escapes overflow:hidden, z-index stacking context. CSS — limited.
- "Test portal with RTL?" — `screen.getByText` finds anywhere in document (not just container).

</details>

---

## Xulosa

Bu faylda quyidagilar yoritildi:

**QISM A — Components & Render Purity (1-4)**: Function components, render purity invariant, idempotency, naming convention, render side effect bug.

**QISM B — Props (5-10)**: Props basics + immutability, children prop + ReactNode types, spread attributes, polymorphic components (`as` prop), generic components, discriminated union props.

**QISM C — Composition (11-13)**: Composition vs Inheritance, slots/named children, inversion of control.

**QISM D — State Lifting & Controlled (14-17)**: Lifting state up, controlled vs uncontrolled, state management decision tree, form library patterns.

**QISM E — Event Handling (18-22)**: Synthetic events, event delegation R16/R17+, R19 form actions, event handler patterns, native vs synthetic event listener.

**QISM F — Legacy Patterns (23-26)**: Render Props, HOC, custom hooks vs HOC vs render props, HOC TypeScript pattern.

**QISM G — Compound Components & Children API (27-29)**: Compound component pattern, React.Children API + cloneElement, modern compound vs Children API.

**QISM H — Error Boundaries (30-33)**: react-error-boundary library, error boundaries (class), error scope (render/lifecycle vs events/async), R19 root error callbacks.

**QISM I — Portals (34-36)**: createPortal + event bubbling, focus management + a11y, portal SSR.

**Keyingi:** [05-performance.md](05-performance.md) — Performance: memo, useMemo/useCallback application, React Compiler, profiling, code splitting, virtualization.



