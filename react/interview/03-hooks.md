# React Hooks — Interview Savollari

> 10 ta hook bo'yicha to'liq mastery: useState, useEffect, useLayoutEffect, useInsertionEffect, useRef, useContext, useReducer, useMemo, useCallback, concurrent hooks (useTransition, useDeferredValue, useSyncExternalStore, useId), R19 hooks (use, useFormStatus, useActionState, useOptimistic), custom hooks.
>
> Har savolda **Deep Dive** — hooks linked list internal'i, mount/update path, dispatcher swap, har hook ning aniq mexanizmi.

---

## Mundarija

- [**QISM A: Hooks Fundamentals** (savollar 1-5)](#qism-a)
- [**QISM B: useState** (savollar 6-9)](#qism-b)
- [**QISM C: useEffect** (savollar 10-15)](#qism-c)
- [**QISM D: useLayoutEffect va useInsertionEffect** (savollar 16-18)](#qism-d)
- [**QISM E: useRef** (savollar 19-23)](#qism-e)
- [**QISM F: useContext** (savollar 24-27)](#qism-f)
- [**QISM G: useReducer** (savollar 28-30)](#qism-g)
- [**QISM H: useMemo va useCallback** (savollar 31-34)](#qism-h)
- [**QISM I: Concurrent Hooks (R18)** (savollar 44-49)](#qism-i)
- [**QISM J: R19 Hooks** (savollar 50-53)](#qism-j)
- [**QISM K: Custom Hooks** (savollar 54-56)](#qism-k)

---

<a id="qism-a"></a>

## QISM A: Hooks Fundamentals

### 1. Hooks nima va nima uchun joriy qilingan? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Hooks** (R16.8, 2019) — function komponentlar uchun **state, lifecycle, va boshqa React features**'ni ishlatish imkoniyatini beradigan API. Pre-R16.8 — bularni ishlatish uchun **class komponent majburiy** edi. Hooks paydo bo'lishi sababi: class komponent muammolari (`this` binding, lifecycle "soup", logic reuse hell — HOC/render props pattern murakkabliklari), kod tushunarliligi, type inference (TypeScript bilan).

### To'liq tushuntirish

**Class komponent muammolari:**

1. **`this` binding** — har metod constructor'da `bind` qilish kerak edi yoki arrow function syntax
2. **Lifecycle soup** — bir method ichida turli concerns (data fetching, subscriptions, DOM mutations)
3. **Logic reuse** — HOC/render props orqali — wrapper hell, prop collision, indirection
4. **State hard to share** — props drilling, Context API verbose
5. **Code splitting** — class methods can't be tree-shaken
6. **Type inference** — TypeScript class generics murakkab

**Hooks afzalliklari:**

1. **No `this`** — function scoped, predictable
2. **Compose logic** — custom hooks via composition
3. **Granular reuse** — hook = unit of behavior
4. **Better TypeScript inference**
5. **Smaller bundle** — tree-shakeable
6. **Single mental model** — function only

### Kod misoli

```tsx
// Pre-Hooks (Class Component)
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
    this.handleClick = this.handleClick.bind(this);  // manual bind
  }

  componentDidMount() {
    document.title = `Count: ${this.state.count}`;
  }

  componentDidUpdate(prevProps, prevState) {
    if (prevState.count !== this.state.count) {
      document.title = `Count: ${this.state.count}`;
    }
  }

  componentWillUnmount() {
    document.title = "Default";
  }

  handleClick() {
    this.setState({ count: this.state.count + 1 });
  }

  render() {
    return <button onClick={this.handleClick}>{this.state.count}</button>;
  }
}

// Hooks (Function Component)
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `Count: ${count}`;
    return () => { document.title = "Default"; };  // cleanup
  }, [count]);

  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
// Lifecycle "soup" yo'q — har concern alohida useEffect
// `this` yo'q
// Logic reuse: extract custom hook
```

**Custom hook composition:**

```tsx
// Class — HOC for reuse (wrapper hell)
const withCounter = (Component) => class extends React.Component {
  state = { count: 0 };
  increment = () => this.setState({ count: this.state.count + 1 });
  render() {
    return <Component {...this.props} count={this.state.count} increment={this.increment} />;
  }
};

const EnhancedButton = withCounter(MyButton);
const DoubleEnhanced = withSomethingElse(withAuth(withCounter(MyButton)));
// 4 wrappers, prop collision risk, no granularity

// Hook — composable, granular
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = useCallback(() => setCount(c => c + 1), []);
  return { count, increment };
}

function MyButton() {
  const { count, increment } = useCounter();
  const auth = useAuth();
  const data = useFetch("/api/data");
  // Compose multiple hooks naturally
  return <button onClick={increment}>{count}</button>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why "Hooks" name:**

- Hook into React features (state, effects, context)
- Don't replace components — augment function components
- "Hook" — small, composable, named function

**Problems Hooks solve:**

**1. `this` binding hell:**

```tsx
// Class
class Form extends React.Component {
  state = { name: "" };

  // ❌ Without bind/arrow
  handleChange(e) {
    this.setState({ name: e.target.value });
    // `this` undefined when called as event handler
  }

  // ✅ Arrow function in class field
  handleChange = (e) => {
    this.setState({ name: e.target.value });
  };

  // ✅ Constructor bind
  constructor(props) {
    super(props);
    this.handleChange = this.handleChange.bind(this);
  }
}

// Function — no `this`
function Form() {
  const [name, setName] = useState("");
  const handleChange = (e) => setName(e.target.value);  // simple
}
```

**2. Lifecycle "soup":**

```tsx
// Class — multiple concerns mixed
class Dashboard extends React.Component {
  componentDidMount() {
    this.subscribeToData();           // data subscription
    document.title = this.props.title;  // DOM mutation
    this.timer = setInterval(this.tick, 1000);  // timer
  }

  componentDidUpdate(prevProps) {
    if (prevProps.title !== this.props.title) {
      document.title = this.props.title;  // mixed with above
    }
    if (prevProps.userId !== this.props.userId) {
      this.unsubscribeFromData();
      this.subscribeToData();
    }
  }

  componentWillUnmount() {
    this.unsubscribeFromData();
    clearInterval(this.timer);
  }
}

// Hooks — separate effects per concern
function Dashboard({ title, userId }) {
  // Concern 1: data subscription
  useEffect(() => {
    const sub = subscribe(userId);
    return () => sub.unsubscribe();
  }, [userId]);

  // Concern 2: document title
  useEffect(() => {
    document.title = title;
    return () => { document.title = "Default"; };
  }, [title]);

  // Concern 3: timer
  useEffect(() => {
    const id = setInterval(tick, 1000);
    return () => clearInterval(id);
  }, []);
}
```

**3. Logic reuse (HOC vs Hooks):**

HOC pattern issues:
- Wrapper hell (4 levels of HOCs)
- Prop name collision
- Static methods lost
- Difficult to debug (DevTools shows wrappers)
- Type inference complex (HOC-of-HOC TS hell)

Hooks:
- Direct call, no wrapper
- No prop name conflict (named return)
- Custom hooks = simple functions
- DevTools shows hook names
- TypeScript inference natural

**4. Server vs Client:**

Class lifecycle methods don't run on server (`componentDidMount` etc.). Need different code paths.

Hooks: `useEffect` doesn't run on server (consistent), but `useState` initializer does. Cleaner mental model.

**Hooks API stability:**

Released R16.8 (Feb 2019). 7 years of production use (as of 2026). Stable, no breaking changes to core hooks.

R18 added: `useTransition`, `useDeferredValue`, `useSyncExternalStore`, `useId`, `useInsertionEffect`.

R19 added: `use`, `useFormStatus`, `useActionState`, `useOptimistic`.

**ESLint plugin:**

```json
{
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

`rules-of-hooks` — enforces top-level call order. `exhaustive-deps` — checks dependency arrays.

**Performance characteristics:**

- Function component re-renders avoid class instance allocation (no `this` setup, no constructor)
- Hook bundle size smaller (no class boilerplate)
- Memoization (`React.memo`, `useMemo`) easier in functions

**TypeScript with Hooks:**

```tsx
// useState type inference
const [count, setCount] = useState(0);  // count: number

// Explicit type
const [user, setUser] = useState<User | null>(null);

// useReducer
type Action = { type: "increment" } | { type: "decrement" };
const [state, dispatch] = useReducer<Reducer<State, Action>>(reducer, initial);
```

Class generics:
```tsx
class DataView<T> extends React.Component<Props<T>, State<T>> { /* ... */ }
// Verbose, less inference
```

**Concurrent rendering reliance:**

Hooks designed with concurrent rendering in mind:
- `useState` — render-phase pure
- `useEffect` — post-commit
- `useTransition` — concurrent priority
- Pre-R16.3 class lifecycle methods (`componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate`) — concurrent rendering bilan incompatible, `UNSAFE_` prefix bilan rename qilingan

**When to still use class:**

- Error boundaries (R19'da ham hooks-based alternative yo'q — hali class kerak)
- Legacy codebase (gradual migration)
- Specific lifecycle that hooks don't replicate (`getSnapshotBeforeUpdate`)

</details>

### Edge Cases

- **Mixed class + hooks**: Allowed, but can't use hooks INSIDE class. Class wraps function, function uses hooks.
- **Hooks in non-React functions**: Error — "Hooks can only be called inside the body of a function component."
- **Hooks in event handlers**: Error — same rule. Only top-level of component or custom hook.

### Follow-up savollar

- "Class komponent o'rinida `useState` ishlatish mumkinmi?" — Yo'q. Class state — `this.state`. Hooks — function-only.
- "Hooks paydo bo'lganda class deprecated bo'ldimi?" — Yo'q. Class qoldi (error boundaries hali kerak). Lekin yangi kod hooks bilan yoziladi.
- "Performance — class vs function?" — Function instance allocation overhead'iga ega emas. Real apps'da rendering optimization (memo, bailout) muhimroq, class vs function tanlovidan emas.

</details>

---

### 2. Rules of Hooks — qanday va nima uchun? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Rules of Hooks** — 2 ta majburiy qoida: (1) **Top-level only** — hooks faqat function component yoki custom hook'ning **eng yuqori darajasida** chaqirilishi kerak (loop, condition, nested function ichida emas), (2) **React functions only** — faqat function component yoki custom hook'dan chaqiriladi (regular JS function emas). Sabab: hooks **call order**'ga bog'liq (linked list index'i) — har render bir xil order bo'lishi shart.

### To'liq tushuntirish

**Qoida 1: Top-level only**

```tsx
function Component() {
  // ✅ TO'G'RI — top level
  const [count, setCount] = useState(0);

  // ❌ NOTO'G'RI — condition
  if (someCondition) {
    const [name, setName] = useState("");
  }

  // ❌ NOTO'G'RI — loop
  for (const item of items) {
    useState(item);
  }

  // ❌ NOTO'G'RI — nested function
  function handleClick() {
    const [x, setX] = useState(0);  // taqiqlanadi
  }

  // ❌ NOTO'G'RI — try/catch
  try {
    const [y, setY] = useState(0);
  } catch (e) {}

  // ❌ NOTO'G'RI — return (early)
  if (props.disabled) return null;
  const [z, setZ] = useState(0);  // hech qachon erta return bo'lsa hooks call'ni o'tkazib yuboradi
}
```

**Qoida 2: React functions only**

```tsx
// ✅ TO'G'RI — function component
function MyComponent() {
  const [x, setX] = useState(0);
}

// ✅ TO'G'RI — custom hook (use* prefix)
function useCustom() {
  const [x, setX] = useState(0);
  return x;
}

// ❌ NOTO'G'RI — regular function
function helper() {
  const [x, setX] = useState(0);  // error
}

// ❌ NOTO'G'RI — class component
class MyClass extends React.Component {
  render() {
    const [x, setX] = useState(0);  // error
    return null;
  }
}

// ❌ NOTO'G'RI — async function
async function MyAsyncComponent() {
  const [x, setX] = useState(0);  // error (R19 RSC server async OK, but no hooks)
}
```

### Kod misoli

```tsx
// ❌ Buggy: conditional hook
function ProfileBuggy({ userId }: { userId: string | null }) {
  if (!userId) return <p>Not logged in</p>;  // ❌ early return before hooks
  const [user, setUser] = useState<User | null>(null);  // ← skipped if userId null
  // ... rest
}

// User logs in: userId becomes truthy
// Re-render: useState now called (hook count changed!)
// React: "Rendered more hooks than during the previous render."
```

```tsx
// ✅ Fixed: hook always called
function ProfileGood({ userId }: { userId: string | null }) {
  const [user, setUser] = useState<User | null>(null);  // ✅ top-level

  if (!userId) return <p>Not logged in</p>;

  // Use user state below
  return <div>{user?.name}</div>;
}
```

```tsx
// ❌ Conditional inside hook
function Comp() {
  if (someCondition) {
    useEffect(() => {  // ❌ conditional
      doSomething();
    }, []);
  }
}

// ✅ Move condition INSIDE hook
function Comp() {
  useEffect(() => {
    if (someCondition) {  // ✅ condition inside
      doSomething();
    }
  }, [someCondition]);
}
```

```tsx
// ❌ Loop hook
function List({ items }: { items: Item[] }) {
  return items.map((item) => {
    const [data, setData] = useState(null);  // ❌ count varies
    return <div>{data}</div>;
  });
}

// ✅ Component per item — each has its own hook
function ListItem({ item }: { item: Item }) {
  const [data, setData] = useState(null);
  return <div>{data}</div>;
}

function List({ items }: { items: Item[] }) {
  return items.map((item) => <ListItem key={item.id} item={item} />);
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why call order matters — linked list:**

```typescript
// React internal: per-fiber hooks linked list
fiber.memoizedState = {
  hook1 (useState 'count', value: 0)
    → next: hook2 (useEffect, deps: [count])
      → next: hook3 (useRef, value: null)
        → next: null
};

// First render:
// 1. useState() — creates hook1 at index 0
// 2. useEffect() — creates hook2 at index 1
// 3. useRef() — creates hook3 at index 2

// Second render — index-based lookup:
// 1. useState() — reads hook at index 0 (hook1) ✅
// 2. useEffect() — reads hook at index 1 (hook2) ✅
// 3. useRef() — reads hook at index 2 (hook3) ✅

// If conditional skipped useState:
// First render: 3 hooks
// Second render: only 2 calls
//   useEffect() reads index 0 (hook1) — wrong! hook1 is useState
//   Type mismatch — corruption
```

**React's runtime check:**

```typescript
function updateWorkInProgressHook() {
  if (currentlyRenderingFiber.memoizedState === null) {
    throw new Error("Rendered more hooks than during the previous render.");
  }
  // OR
  if (workInProgressHook.next === null && currentHook.next !== null) {
    throw new Error("Rendered fewer hooks than during the previous render.");
  }
}
```

React detects mismatch and throws.

**ESLint `rules-of-hooks` enforcement:**

```typescript
// Linter detects:
function Bad() {
  if (cond) {
    useState(0);  // ESLint error
  }
}

// Linter checks:
// - Hook called at top level (not inside if/for/while/try/&&/||/?:)
// - Caller is function component (PascalCase) or custom hook (camelCase starting with "use")
```

**`use*` naming convention:**

```tsx
// React identifies via name:
function useCustom() {  // OK — starts with "use"
  return useState(0);
}

function helper() {     // ERROR — doesn't start with "use"
  return useState(0);
}

// PascalCase = component
function MyComponent() { return useState(0); }

// camelCase + "use" = custom hook
function useCustom() { return useState(0); }

// camelCase no "use" = regular function (hooks not allowed)
function doStuff() { return useState(0); }  // ERROR
```

**Why `use*` prefix:**

ESLint rule treats `use*` as hook context. Without prefix, can't statically analyze.

```typescript
// ESLint rule:
function ruleOfHooks(node) {
  const callee = node.callee.name;
  if (callee.startsWith("use") && callee[3] === callee[3].toUpperCase()) {
    // It's a hook call
    checkContext(node);
  }
}
```

`useState` — "use" + "S" (capital). `useradius` — "use" + "r" (lowercase) — NOT a hook (heuristic).

**`use()` (R19) — special case:**

```tsx
// R19 — `use()` conditional/loop ichida chaqirilishi mumkin
function Component({ promise, fallback, showFallback }: Props) {
  if (showFallback) return <p>{fallback}</p>;
  const data = use(promise);  // ✅ conditional OK
  return <div>{data}</div>;
}
```

`use()` — relaxed Rules of Hooks. Sabab: u fiber'da **o'z hook slot'i yo'q** (linked list'ga qo'shilmaydi). Promise tracking thenable status orqali, context reading direct. React docs `use`'ni "hook" deb ataydi, lekin boshqa hook'lardan farqli — conditional/loop ichida chaqirish mumkin.

**Custom hook calling other hooks:**

```tsx
function useCustomA() {
  const [x, setX] = useState(0);  // hook 1 of useCustomA
  return x;
}

function useCustomB() {
  const [y, setY] = useState(0);  // hook 1 of useCustomB
  const x = useCustomA();          // adds 1 hook (hook 2 of useCustomB? or hook 1 of useCustomA?)
  return [x, y];
}

// Each hook call adds to component's linked list:
// fiber.memoizedState:
//   useCustomB.useState (y)
//     → useCustomB.useCustomA.useState (x)
//       → null

// 2 hooks total in fiber's list
```

Hooks compose flat — custom hooks don't create nested lists. Just sequential calls.

**Multiple component instances:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  return <p>{count}</p>;
}

// 3 instances:
<Counter />
<Counter />
<Counter />

// Each instance — own fiber, own memoizedState linked list
// 3 separate count states
```

**Render time hook indexing:**

```typescript
let currentlyRenderingFiber: Fiber | null = null;
let workInProgressHook: Hook | null = null;
let currentHook: Hook | null = null;

function renderWithHooks(fiber: Fiber, Component: Function) {
  currentlyRenderingFiber = fiber;
  workInProgressHook = null;

  // Mount or update?
  if (current !== null && current.memoizedState !== null) {
    // Update — read existing hooks
    HooksDispatcher = HooksDispatcherOnUpdate;
  } else {
    // Mount — create new hooks
    HooksDispatcher = HooksDispatcherOnMount;
  }

  const children = Component();

  currentlyRenderingFiber = null;
  return children;
}

function useState(initial) {
  // Dispatch to mount or update version
  return HooksDispatcher.useState(initial);
}
```

`HooksDispatcher` — global, swapped before each render.

**Why React doesn't use named hooks (would solve order):**

```tsx
// Hypothetical named API
const [count, setCount] = useState("count", 0);

// Pros: order-independent
// Cons:
// - Verbose
// - Names must be unique per component
// - Refactoring (renaming) more painful
// - Doesn't compose well (custom hooks need namespacing)
```

React team chose call order for simplicity. Linter enforces correctness.

**Class component analog (no rules):**

```tsx
class Counter extends React.Component {
  state = { count: 0 };

  render() {
    if (this.props.visible) {
      // Class — `this.state` ob'ektga conditional access mumkin
      return <p>{this.state.count}</p>;
    }
    return <p>nothing</p>;
  }
}
```

Class state — single object, indexing yo'q (key-based access). Hooks — call order'ga bog'liq (linked list traversal). Trade-off: composability (hooks) vs loose ordering (class).

</details>

### Edge Cases

- **Hooks after early return**: Hooks below return — sometimes called, sometimes not. Linter catches.
- **Hooks in IIFE**: `(() => { useState(0); })()` — invalid (called from regular function).
- **Hooks in callback**: `array.map(() => useState(0))` — invalid.

### Follow-up savollar

- "Why React doesn't enforce at runtime?" — Performance. Linter sufficient for dev.
- "Conditional rendering after hooks — OK?" — Yes. Hooks always called, then conditional render: `if (loading) return <Spinner />;`
- "useEffect inside if — alternative?" — Always call hook, conditional logic inside: `useEffect(() => { if (cond) doSomething(); }, [cond]);`

</details>

---

### 3. Hooks linked list mexanizmi — `memoizedState` strukturasi [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Har Fiber'ning `memoizedState` field'i — **hooks linked list**'ning birinchi node'i. Har hook chaqirilganda, list'ga **yangi node** qo'shiladi (mount) yoki **navbatdagi node** o'qiladi (update). Hook node'lari `next` pointer bilan bog'langan. `useState`, `useEffect`, `useRef`, `useMemo` — hammasi shu list'da tartibli yashaydi. Index-based lookup — rules of hooks call order'ga bog'liqligi shu mexanizmdan kelib chiqadi.

### To'liq tushuntirish

**Hook node strukturasi:**

```typescript
type Hook = {
  memoizedState: any;       // hook value (state, deps + value, ref, ...)
  baseState: any;           // useState — base state for queue
  baseQueue: any;           // pending state updates
  queue: UpdateQueue | null; // useState's update queue
  next: Hook | null;         // next hook in list
};

type UpdateQueue = {
  pending: Update | null;    // circular linked list of pending updates
  dispatch: Function;
  lastRenderedReducer: Function;
  lastRenderedState: any;
};

type Update = {
  action: any;
  next: Update;
  hasEagerState: boolean;
  eagerState: any;
};
```

**Visualization:**

```
Fiber:
  memoizedState ─→ Hook1 (useState, count=0)
                     │ next
                     ↓
                   Hook2 (useEffect, deps=[count])
                     │ next
                     ↓
                   Hook3 (useRef, current=null)
                     │ next
                     ↓
                   null
```

### Kod misoli

```tsx
function Component() {
  const [count, setCount] = useState(0);    // Hook 1
  useEffect(() => { /* ... */ }, [count]);   // Hook 2
  const ref = useRef<HTMLDivElement>(null);  // Hook 3
  const memo = useMemo(() => count * 2, [count]);  // Hook 4

  // Fiber.memoizedState:
  //   Hook1: { memoizedState: 0, queue: {...}, next: Hook2 }
  //   Hook2: { memoizedState: { create, destroy, deps: [0] }, next: Hook3 }
  //   Hook3: { memoizedState: { current: null }, next: Hook4 }
  //   Hook4: { memoizedState: [0, [0]], next: null }
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`mountWorkInProgressHook` — first render:**

```typescript
let currentlyRenderingFiber: Fiber;
let workInProgressHook: Hook | null;

function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };

  if (workInProgressHook === null) {
    // First hook — assign to fiber
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // Subsequent — append to chain
    workInProgressHook = workInProgressHook.next = hook;
  }

  return workInProgressHook;
}
```

**`updateWorkInProgressHook` — subsequent renders:**

```typescript
let currentHook: Hook | null;

function updateWorkInProgressHook(): Hook {
  let nextCurrentHook: Hook | null;

  if (currentHook === null) {
    // First hook of this render
    const current = currentlyRenderingFiber.alternate;
    nextCurrentHook = current?.memoizedState ?? null;
  } else {
    // Subsequent
    nextCurrentHook = currentHook.next;
  }

  let nextWorkInProgressHook: Hook | null;
  if (workInProgressHook === null) {
    nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
  } else {
    nextWorkInProgressHook = workInProgressHook.next;
  }

  if (nextWorkInProgressHook !== null) {
    // Reuse existing WIP hook
    workInProgressHook = nextWorkInProgressHook;
    currentHook = nextCurrentHook;
  } else {
    // Clone from current
    if (nextCurrentHook === null) {
      throw new Error("Rendered more hooks than during the previous render.");
    }
    currentHook = nextCurrentHook;
    const newHook: Hook = {
      memoizedState: currentHook.memoizedState,
      baseState: currentHook.baseState,
      baseQueue: currentHook.baseQueue,
      queue: currentHook.queue,
      next: null,
    };

    if (workInProgressHook === null) {
      currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
    } else {
      workInProgressHook = workInProgressHook.next = newHook;
    }
  }

  return workInProgressHook;
}
```

**Hook-specific data in `memoizedState`:**

| Hook | `memoizedState` content |
|------|-----|
| `useState` | Current state value |
| `useReducer` | Current state value |
| `useEffect` | `{ create, destroy, deps, tag }` (Effect object) |
| `useLayoutEffect` | Same as useEffect with different tag |
| `useRef` | `{ current: value }` |
| `useMemo` | `[value, deps]` |
| `useCallback` | `[callback, deps]` |
| `useContext` | None (read directly from provider) |
| `useId` | Generated ID string |
| `useTransition` | `[isPending, startTransition]` |
| `useDeferredValue` | Deferred value |
| `useSyncExternalStore` | Snapshot |

**`useState` queue mechanism:**

```typescript
type Update = {
  action: any;          // value or updater function
  next: Update;         // circular link
  hasEagerState: boolean;
  eagerState: any;
};

// Hook structure for useState:
hook.queue = {
  pending: null | Update,  // circular list — pending LAST node'ga ishora qiladi
  dispatch: setState,
  lastRenderedReducer: basicStateReducer,
  lastRenderedState: 0,
};

// On setState(value):
function dispatchSetState(fiber, queue, action) {
  const update: Update = {
    action,
    next: null,
    hasEagerState: false,
    eagerState: null,
  };

  // Append to circular linked list
  const pending = queue.pending;
  if (pending === null) {
    update.next = update;  // self-link (circular start)
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  queue.pending = update;

  scheduleUpdateOnFiber(fiber);
}

// On render:
function updateState() {
  const hook = updateWorkInProgressHook();
  let newState = hook.baseState;

  if (hook.queue.pending !== null) {
    const first = hook.queue.pending.next;
    let update = first;
    do {
      const action = update.action;
      newState = typeof action === "function"
        ? action(newState)  // functional
        : action;            // direct
      update = update.next;
    } while (update !== first);  // circular traverse
  }

  hook.memoizedState = newState;
  hook.queue.pending = null;
  return [newState, hook.queue.dispatch];
}
```

**Effect linked list (separate from hooks):**

```typescript
// useEffect creates Effect object stored in hook.memoizedState
type Effect = {
  tag: HookFlag;        // Passive (useEffect) | Layout (useLayoutEffect) | Insertion (useInsertionEffect)
  create: () => void | (() => void);
  destroy: (() => void) | undefined;
  deps: any[] | null;
  next: Effect;          // circular linked list of effects in fiber
};

// Hook.memoizedState for useEffect:
hook.memoizedState = {
  tag: PassiveEffect,
  create: () => { /* effect callback */ },
  destroy: undefined,  // populated after effect runs
  deps: [count, name],
  next: null,
};

// All effects in fiber linked separately:
fiber.updateQueue = {
  lastEffect: Effect (tail of circular list),
};
```

**Hook count consistency check:**

```typescript
// During render, after hooks called:
function checkHookConsistency(fiber: Fiber) {
  const current = fiber.alternate;
  if (current === null) return; // Mount, no check

  let prevHook = current.memoizedState;
  let newHook = fiber.memoizedState;
  let count = 0;

  while (prevHook !== null && newHook !== null) {
    prevHook = prevHook.next;
    newHook = newHook.next;
    count++;
  }

  if (prevHook === null && newHook !== null) {
    throw new Error("Rendered more hooks than during the previous render.");
  }
  if (prevHook !== null && newHook === null) {
    throw new Error("Rendered fewer hooks than during the previous render.");
  }
}
```

**Conditional hook violation trace:**

```tsx
function Buggy({ flag }: { flag: boolean }) {
  const [a, setA] = useState(1);  // Hook 0
  if (flag) {
    const [b, setB] = useState(2);  // Hook 1 (conditional!)
  }
  const [c, setC] = useState(3);    // Hook 2 (or Hook 1 depending on flag)
}

// Render 1: flag=true
//   Hooks: [a=1, b=2, c=3]  (3 hooks)
//   memoizedState linked list: a → b → c → null

// Render 2: flag=false
//   Hooks called: useState(a), useState(c)  (2 calls)
//   But list traversal:
//     1st useState() reads hook[0] (was a) — OK, a=1
//     2nd useState() reads hook[1] (was b!) — c gets b's state value
//     React noticed: 3rd hook expected but only 2 called
//     Error: "Rendered fewer hooks"
```

**Mount path vs update path:**

```typescript
// Mount path (first render)
const HooksDispatcherOnMount = {
  useState: mountState,
  useEffect: mountEffect,
  useReducer: mountReducer,
  useRef: mountRef,
  useMemo: mountMemo,
  useCallback: mountCallback,
  useContext: readContext,
  // ...
};

// Update path (subsequent renders)
const HooksDispatcherOnUpdate = {
  useState: updateState,
  useEffect: updateEffect,
  useReducer: updateReducer,
  useRef: updateRef,
  useMemo: updateMemo,
  useCallback: updateCallback,
  useContext: readContext,
  // ...
};

// Before render:
function renderWithHooks(fiber: Fiber, Component: Function) {
  if (fiber.memoizedState === null) {
    HooksDispatcher = HooksDispatcherOnMount;
  } else {
    HooksDispatcher = HooksDispatcherOnUpdate;
  }
  return Component();
}
```

`useState` actually calls `HooksDispatcher.useState`:

```typescript
import { ReactCurrentDispatcher } from "shared/ReactSharedInternals";

export function useState(initialState) {
  const dispatcher = ReactCurrentDispatcher.current;
  return dispatcher.useState(initialState);
}
```

Dispatcher swap allows mount vs update behavior.

**Hooks during error recovery:**

If hooks throw mid-render, dispatcher reset on error catch. Next render starts fresh.

**Re-render scenarios:**

1. State change → re-render → update path
2. Strict Mode → mount → unmount → mount cycle (dev only)
3. Suspense → unmount on suspend, remount on resolve

**Memory layout (V8):**

Har Hook node — `memoizedState`, `baseState`, `baseQueue`, `queue`, `next` field'lariga ega oddiy obyekt. V8 hidden class optimization qo'llaniladi.

**Performance:**

- Linked list traversal — O(n) where n = hooks count
- Sequential access — V8 inline cache do'st (har hook bir xil shape)

</details>

### Edge Cases

- **Hook count change between renders**: Throws error. Recover by avoiding conditional hooks.
- **Different hook types at same index**: Type mismatch silent (memoizedState shape varies). Bug-prone.
- **Custom hook with conditional internal hook**: Same rules apply — custom hook is just a function with hooks.

### Follow-up savollar

- "Linked list — why not array?" — Linked list cheaper for append (O(1) vs array O(n) for grow). Index access not needed (sequential).
- "Hooks order matters even between custom hooks?" — Yes. Custom hook calls treated as flat sequential calls in fiber.
- "DevTools shows hook list?" — Yes. React DevTools "Components" panel shows hook order, names (with `useDebugValue` annotations).

</details>

---

### 4. Mount path vs Update path — dispatcher swap [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Har render boshida React **`HooksDispatcher`**'ni o'zgartiradi: birinchi render — **`HooksDispatcherOnMount`** (yangi hooks yaratadi, `mountState`, `mountEffect`, ...), keyingi render'lar — **`HooksDispatcherOnUpdate`** (mavjud hooks'larni o'qiydi, `updateState`, `updateEffect`, ...). `useState`, `useEffect` kabi funksiyalar global dispatcher'ni chaqiradi — implementation render context'iga qarab tanlanadi.

### To'liq tushuntirish

**Dispatcher mexanizmi:**

```typescript
// Global current dispatcher
let currentDispatcher: HooksDispatcher | null = null;

// `useState` is just a forwarding function
function useState(initial) {
  return currentDispatcher.useState(initial);
}

// React swaps before each render:
function renderWithHooks(current, fiber, Component, props) {
  if (current === null || current.memoizedState === null) {
    // First render — mount path
    currentDispatcher = HooksDispatcherOnMount;
  } else {
    // Subsequent renders — update path
    currentDispatcher = HooksDispatcherOnUpdate;
  }

  // Reset render state
  workInProgressHook = null;
  currentHook = null;

  // Call component (uses hooks)
  const children = Component(props);

  // Reset
  currentDispatcher = ContextOnlyDispatcher;
  workInProgressHook = null;
  currentHook = null;

  return children;
}
```

### Kod misoli

```typescript
// HooksDispatcherOnMount
const HooksDispatcherOnMount = {
  useState: mountState,
  useEffect: mountEffect,
  useReducer: mountReducer,
  useRef: mountRef,
  useMemo: mountMemo,
  useCallback: mountCallback,
  useContext: readContext,
  useDebugValue: mountDebugValue,
  useLayoutEffect: mountLayoutEffect,
  useInsertionEffect: mountInsertionEffect,
  useImperativeHandle: mountImperativeHandle,
  useId: mountId,
  useTransition: mountTransition,
  useDeferredValue: mountDeferredValue,
  useSyncExternalStore: mountSyncExternalStore,
};

// HooksDispatcherOnUpdate
const HooksDispatcherOnUpdate = {
  useState: updateState,
  useEffect: updateEffect,
  useReducer: updateReducer,
  useRef: updateRef,
  useMemo: updateMemo,
  useCallback: updateCallback,
  useContext: readContext,
  useDebugValue: updateDebugValue,
  useLayoutEffect: updateLayoutEffect,
  useInsertionEffect: updateInsertionEffect,
  useImperativeHandle: updateImperativeHandle,
  useId: updateId,
  useTransition: updateTransition,
  useDeferredValue: updateDeferredValue,
  useSyncExternalStore: updateSyncExternalStore,
};
```

**Mount vs update implementations:**

```typescript
// mountState — first render
function mountState<S>(initial: S | (() => S)): [S, Dispatch<S>] {
  const hook = mountWorkInProgressHook();

  hook.memoizedState = hook.baseState =
    typeof initial === "function" ? initial() : initial;

  hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: hook.memoizedState,
  };

  const dispatch = dispatchSetState.bind(null, currentlyRenderingFiber, hook.queue);
  hook.queue.dispatch = dispatch;

  return [hook.memoizedState, dispatch];
}

// updateState — subsequent renders
function updateState<S>(initial: any): [S, Dispatch<S>] {
  return updateReducer(basicStateReducer, initial);
}

function updateReducer<S, A>(reducer, initial) {
  const hook = updateWorkInProgressHook();
  let newState = hook.baseState;

  // Process pending updates
  const pending = hook.queue.pending;
  if (pending !== null) {
    const first = pending.next;
    let update = first;
    do {
      const action = update.action;
      newState = reducer(newState, action);
      update = update.next;
    } while (update !== first);
    hook.queue.pending = null;
  }

  hook.memoizedState = newState;
  hook.queue.lastRenderedState = newState;

  return [newState, hook.queue.dispatch];
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why dispatcher swap (vs branching in each hook):**

```typescript
// Without dispatcher (branching)
function useState(initial) {
  if (currentlyRenderingFiber.alternate === null) {
    return mountState(initial);
  }
  return updateState(initial);
}

// vs Dispatcher swap — branch once per render
currentDispatcher = HooksDispatcherOnMount; // or Update
useState(initial); // → currentDispatcher.useState(initial)
```

Dispatcher: branch decision made once. Per-hook lookup avoids per-call branch.

**`ContextOnlyDispatcher` — outside render:**

```typescript
const ContextOnlyDispatcher = {
  useState: () => { throw new Error("Hooks can only be called inside the body of a function component."); },
  // ... all hooks throw
  useContext: readContext,  // context read'i server-side'da ham ishlaydi
};

// After render finishes, reset to ContextOnlyDispatcher
// Hook calls outside render → throw error
```

**Strict Mode dev — different dispatcher:**

```typescript
// Development with Strict Mode
const HooksDispatcherOnMountInDEV = {
  useState: function(initial) {
    // Track hook for warnings
    const hook = mountState(initial);
    // ... debug instrumentation
    return hook;
  },
  // ...
};
```

Dev dispatchers add tracking, warnings, dev-only checks.

**Re-render via state change — same dispatcher:**

```typescript
function dispatchSetState(fiber, queue, action) {
  enqueueUpdate(queue, action);
  scheduleUpdateOnFiber(fiber);  // schedule re-render
}

// Re-render:
// Component called again
// renderWithHooks runs
// alternate exists → HooksDispatcherOnUpdate
```

**Mount within update (rare):**

```tsx
function Parent() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(true)}>Show</button>
      {show && <Child />}  {/* Child mounts after first render */}
    </>
  );
}

// Parent render 1: HooksDispatcherOnMount (Parent's first render)
// Parent render 2 (after click): HooksDispatcherOnUpdate (Parent's update)
//   Child renders for first time
//   Child uses HooksDispatcherOnMount (Child's first render)
```

Each fiber has its own mount/update state. Parent updating doesn't affect child mounting.

**`renderWithHooks` integration:**

```typescript
function updateFunctionComponent(current, workInProgress, Component, nextProps, renderLanes) {
  prepareToReadContext(workInProgress, renderLanes);

  let nextChildren = renderWithHooks(
    current,
    workInProgress,
    Component,
    nextProps,
    null,
    renderLanes
  );

  // ... reconcile children
}
```

`renderWithHooks` — entry point for function component rendering, handles dispatcher.

**Hook ordering during render:**

```typescript
function renderWithHooks(...) {
  workInProgressHook = null;
  currentHook = null;

  // Component called — hooks called sequentially
  const children = Component(props);

  // After render:
  // workInProgressHook — last hook called
  // fiber.memoizedState — first hook (head of list)
}
```

After render, fiber's hook list represents the current state. Used by next render's update path.

**Multiple call dispatchers (e.g., re-renders during render):**

```tsx
function Component() {
  const [count, setCount] = useState(0);

  if (count < 5) {
    setCount(count + 1);  // ← re-render during render!
    // React detects, schedules immediate re-render
  }

  return <p>{count}</p>;
}

// First render:
// useState — mount path → count = 0
// setCount(1) called → schedule re-render
// Continue render (count=0 used in JSX)
// Render finishes
// React detects pending update → re-render immediately
// Second render (within same render batch):
// useState — UPDATE path (alternate now exists from first WIP commit? actually not yet committed...)
```

This "render-phase update" — React handles by completing first render, then re-rendering with queued state. Pure render constraint important.

**Server Components dispatcher:**

```typescript
// React Server Component renderer
const ServerHooksDispatcher = {
  useState: () => { throw new Error("useState not available in Server Components"); },
  useEffect: () => { throw new Error("useEffect not available in Server Components"); },
  useContext: readServerContext,  // Server Context allowed
  use: serverUse,  // R19 — `use(promise)` in server
};
```

Server context — different dispatcher. Most hooks throw (no state in RSC).

**Dispatcher access in module-level code:**

```typescript
import { useState } from "react";

// Outside render — useState calls ContextOnlyDispatcher.useState
// → throws error

// Inside component — render() chaqirilganda dispatcher swap qilinadi
function Component() {
  const [x] = useState(0);  // OK
}
```

</details>

### Edge Cases

- **Hooks during commit phase**: Forbidden — commit phase has its own dispatcher (DevTools warns).
- **Hooks in `useEffect` callback**: Forbidden — callback runs outside render.
- **Hooks in setTimeout/Promise**: Forbidden — same reason.

### Follow-up savollar

- "Why not just check `current === null` per hook?" — Performance. Dispatcher swap once per render vs N times per render.
- "Custom dispatcher possible?" — `react/__internal` access yes (for testing libraries). Not recommended for app code.
- "Server-side hooks?" — Limited. RSC has restricted dispatcher. SSR (`react-dom/server`) — most hooks no-op or throw for state.

</details>

---

### 5. Output: conditional hook bug [Output] [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi komponent qachon va qanday error chiqaradi?

```tsx
function MysteryComponent({ user }: { user: User | null }) {
  if (!user) return <p>Loading</p>;

  const [count, setCount] = useState(0);
  const [name, setName] = useState(user.name);

  useEffect(() => {
    document.title = `${name}: ${count}`;
  }, [name, count]);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <input value={name} onChange={(e) => setName(e.target.value)} />
    </div>
  );
}

// Steps:
// 1. Render with user = null
// 2. Render with user = { name: "Ali" }
// 3. Click button (setCount(1))
```

### Javob

**Render 1: user = null**
- Returns `<p>Loading</p>` early
- No hooks called (early return BEFORE hooks)
- Fiber's memoizedState: null

**Render 2: user = { name: "Ali" }**
- Early return skipped (user truthy)
- `useState(0)` called — Hook 0 created (count = 0)
- `useState("Ali")` called — Hook 1 created (name = "Ali")
- `useEffect(...)` called — Hook 2 created
- All 3 hooks now in linked list
- ✅ Renders normally

**But on Render 3 (next state change):**

If state change happens AFTER render 2 → user might still be truthy → all 3 hooks still called → OK.

But what if user becomes null again? On render with `user = null`:
- Early return → 0 hooks called
- Fiber expected 3 hooks (from render 2's memoizedState)
- React error: **"Rendered fewer hooks than during the previous render."**

**Crash scenarios:**
- Render 1 (user=null): 0 hooks. Memoized: null. ✅ OK.
- Render 2 (user=valid): 3 hooks. Memoized: [Hook0, Hook1, Hook2]. ✅ OK (mount path).
- Render 3 (user=null): 0 hooks BUT memoized has 3 hooks → **ERROR**.

```
Error: Rendered fewer hooks than during the previous render.
   in MysteryComponent
```

**Fix — hooks unconditional:**

```tsx
function FixedComponent({ user }: { user: User | null }) {
  // ✅ Hooks ALWAYS called, conditional logic AFTER
  const [count, setCount] = useState(0);
  const [name, setName] = useState(user?.name ?? "");

  useEffect(() => {
    if (!user) return;
    document.title = `${name}: ${count}`;
  }, [name, count, user]);

  // Conditional render AFTER hooks
  if (!user) return <p>Loading</p>;

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <input value={name} onChange={(e) => setName(e.target.value)} />
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Detailed trace:**

**Render 1 (user = null):**

```typescript
// React renders component:
HooksDispatcher = HooksDispatcherOnMount; // first render
workInProgressHook = null;

// Component code:
if (!user) return <p>Loading</p>;
// Early return — no hooks called

// After render:
fiber.memoizedState = null;  // hooks linked list empty
```

**Render 2 (user becomes valid):**

```typescript
// React renders component:
const current = fiber.alternate;
HooksDispatcher = (current.memoizedState !== null)
  ? HooksDispatcherOnUpdate
  : HooksDispatcherOnMount;
// current.memoizedState === null (from render 1) → MOUNT path

// Component code:
if (!user) return <p>Loading</p>;  // skipped, user truthy
const [count, setCount] = useState(0);  // mountState — Hook 0 created
const [name, setName] = useState("Ali"); // mountState — Hook 1 created
useEffect(() => { ... }, [name, count]); // mountEffect — Hook 2 created

// After render:
fiber.memoizedState = Hook0 → Hook1 → Hook2
```

**Render 3 (user = null again):**

```typescript
const current = fiber.alternate;
HooksDispatcher = HooksDispatcherOnUpdate;  // memoizedState exists from render 2

// Component code:
if (!user) return <p>Loading</p>;  // early return!

// After render:
// React detects: previous render had 3 hooks, this render had 0
// throws: "Rendered fewer hooks"
```

**Why React doesn't allow this pattern:**

Hook indexing breaks. Hook 0 from render 2 has count=0. Render 3 calls 0 hooks. Hook 0 in memoizedState is "orphaned" — but if next render calls 1 hook, that hook reads memoizedState[0] which has count state → corrupted state for new hook.

**`Rendered fewer hooks` error:**

```typescript
function checkHookConsistency(fiber: Fiber) {
  const current = fiber.alternate;
  if (current === null) return;

  let prevHookCount = countHooks(current.memoizedState);
  let currentHookCount = countHooks(fiber.memoizedState);

  if (currentHookCount < prevHookCount) {
    throw new Error("Rendered fewer hooks than during the previous render.");
  }
  if (currentHookCount > prevHookCount) {
    throw new Error("Rendered more hooks than during the previous render.");
  }
}
```

**Common variations:**

```tsx
// Variation 1: Conditional hooks
function Comp({ flag }) {
  const [a] = useState(0);
  if (flag) {
    const [b] = useState(0);  // ❌ conditional
  }
  // flag toggling causes mismatch
}

// Variation 2: Loop hooks
function Comp({ items }) {
  for (const item of items) {
    useState(item);  // ❌ loop
  }
  // items.length change causes mismatch
}

// Variation 3: Try/catch hooks
function Comp() {
  try {
    useEffect(() => { /* might throw */ });  // ❌ try/catch wrap
  } catch {}
  // Inconsistent behavior
}

// Variation 4: Hook in callback
function Comp() {
  const handler = () => {
    useState(0);  // ❌ can't call from event handler
  };
}
```

**Lint catches all these:**

```typescript
// ESLint rules-of-hooks
function Comp({ flag }) {
  const [a] = useState(0);
  if (flag) {
    const [b] = useState(0);  // Error: "React Hook 'useState' is called conditionally"
  }
}
```

**`use()` hook (R19) — relaxed rules:**

```tsx
function Comp({ promise, flag }: Props) {
  if (!flag) return <p>Disabled</p>;  // ✅ OK with use()
  const data = use(promise);          // ✅ conditional OK
  return <div>{data}</div>;
}
```

`use()` doesn't store state in fiber's hooks list (no slot). Different mechanism.

**Practical guidance:**

1. **Always call hooks at top**
2. **Conditional logic via deps array** or **inside hook body**
3. **Conditional render AFTER all hooks**

```tsx
function Component({ enabled, data }) {
  // ✅ Hooks first
  const [state, setState] = useState(initial);

  useEffect(() => {
    if (!enabled) return;  // ✅ condition inside hook
    doSomething(data);
  }, [enabled, data]);

  // ✅ Conditional render after hooks
  if (!enabled) return null;

  return <div>{state}</div>;
}
```

**Custom hook with internal conditional:**

```tsx
function useFetch(url: string | null) {
  const [data, setData] = useState(null);

  useEffect(() => {
    if (!url) return;  // ✅ condition inside
    fetch(url).then((r) => r.json()).then(setData);
  }, [url]);

  return data;
}

// Caller:
function Component({ id }: { id: string | null }) {
  const data = useFetch(id ? `/api/${id}` : null);  // ✅ always called
  // ...
}
```

</details>

### Edge Cases

- **Hook called conditionally — but always same condition**: `if (true) useState(0)` — still error (linter detects).
- **Hook in early return path**: Same problem — varies between renders.
- **Hook in component returning null vs JSX**: Hooks before return execute always — OK. Hooks after early return — bug.

### Follow-up savollar

- "Why React doesn't auto-fix?" — Auto-fix would rewrite code structure. Linter helps developer understand and fix.
- "Conditional `useEffect`?" — `useEffect` always called, condition inside.
- "How to skip a hook conditionally?" — Conditional logic INSIDE the hook (e.g., `useEffect(() => { if (!flag) return; ... }, [flag])`).

</details>

---

### 6. Hooks dispatcher — `ReactCurrentDispatcher.current` mexanizmi [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Hooks ichki implementation **dispatcher pattern**'ga asoslangan. `ReactCurrentDispatcher.current` — global mutable referenc, render time'da React unga `HooksDispatcherOnMount` yoki `HooksDispatcherOnUpdate` o'rnatadi. `useState`, `useEffect` va boshqa hook'lar shu dispatcher'ning method'larini chaqiradi. Render tashqarisida — dispatcher null, "Invalid hook call" error.

### Kod misoli

```typescript
// React internal (sodda)
const ReactCurrentDispatcher = { current: null };

const HooksDispatcherOnMount = {
  useState: mountState,
  useEffect: mountEffect,
  useMemo: mountMemo,
  useRef: mountRef,
  // ...
};

const HooksDispatcherOnUpdate = {
  useState: updateState,
  useEffect: updateEffect,
  useMemo: updateMemo,
  useRef: updateRef,
  // ...
};

// useState public API
function useState(initialState) {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}

function resolveDispatcher() {
  if (!ReactCurrentDispatcher.current) {
    throw new Error("Invalid hook call. Hooks can only be called inside body of a function component.");
  }
  return ReactCurrentDispatcher.current;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Render-time swap:**

```typescript
function renderWithHooks(fiber, Component, props) {
  const isUpdate = fiber.alternate !== null && fiber.alternate.memoizedState !== null;

  ReactCurrentDispatcher.current = isUpdate
    ? HooksDispatcherOnUpdate
    : HooksDispatcherOnMount;

  try {
    const children = Component(props);  // useState/useEffect chaqiruvlari
    return children;
  } finally {
    ReactCurrentDispatcher.current = null;  // reset
  }
}
```

**Dispatcher swap reason:**

- **Mount**: Yangi hook yaratish, linked list'ga qo'shish
- **Update**: Mavjud hook'ni topish, queue apply qilish

Different code paths — separate dispatchers for clarity + perf.

**Dev mode dispatcher:**

```typescript
const HooksDispatcherOnMountInDEV = {
  ...HooksDispatcherOnMount,
  useState: (initialState) => {
    currentHookNameInDev = "useState";
    warnInvalidHookAccess();
    return mountState(initialState);
  },
  // ... extra warnings
};
```

Dev — extra warnings (Rules of Hooks check, etc.). Production — minimal.

**Why error outside component:**

```typescript
function useCustomHook() {
  return useState(0);  // ❌ no component context
}

useCustomHook();  // Error: dispatcher null
// Fix: call inside component
function MyComponent() {
  const [state] = useCustomHook();  // ✅ OK
}
```

**Server-side dispatcher (different):**

```typescript
const HooksDispatcherOnServer = {
  useState: (init) => [typeof init === "function" ? init() : init, () => {}],
  useEffect: () => {},  // no-op
  useLayoutEffect: () => {  // warning
    warnIfNotEnvironmentRender();
  },
  // ...
};
```

Server render — different dispatcher (no commit, no effects).

**Multiple React instances (microfrontends) — dispatcher conflict:**

```javascript
// 2 React versions in same page — separate ReactCurrentDispatcher refs
// Mixed hooks → Error: dispatcher mismatch
```

Avoid via single React instance, peer dependencies.

</details>

### Edge Cases

- **Hook in event handler**: Error — handler runs outside render. Move to `useEffect`.
- **Hook in `useEffect` callback**: Error — effect runs after render. Hooks only in render body.
- **Hook in async function**: Error — async runs outside render context.

### Follow-up savollar

- "Why mutable global instead of context?" — Context lookup overhead. Mutable global O(1) faster for hot path.
- "Concurrent rendering — dispatcher safe?" — Yes, set/reset per-render synchronously.
- "Test mocking hooks?" — `@testing-library/react-hooks` (deprecated) yoki render in component wrapper.

</details>

---

### 7. Output: stale closure in setInterval [Output] [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi komponent timer'ni boshlaydi. 5 sekund ichida 5 marta bosilsa, oxirida count nima bo'ladi?

```tsx
function StaleClosureCounter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);  // ← stale closure
    }, 1000);
    return () => clearInterval(id);
  }, []);  // ← empty deps

  return <p>{count}</p>;
}
```

### Javob

**Result: count = 1** (forever stuck at 1)

**Sabab:**

`useEffect` empty deps `[]` — effect mount'da bir marta. `setCount(count + 1)` — closure captures `count = 0` (mount value). Har second:
- setCount(0 + 1) = setCount(1)
- setCount(0 + 1) = setCount(1)  (count still 0 in closure!)
- setCount(0 + 1) = setCount(1)
- ...

State updates from 0 → 1 → 1 → 1 → 1 (idempotent — no actual change after first).

**Fixes:**

```tsx
// ✅ Functional update
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1);  // ← always latest
  }, 1000);
  return () => clearInterval(id);
}, []);

// ✅ Or include in deps
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(id);
}, [count]);
// But: clears + recreates interval each tick (wasteful)

// ✅ useRef for latest value
function GoodCounter() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);

  useEffect(() => {
    countRef.current = count;
  }, [count]);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(countRef.current + 1);  // ← always latest
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <p>{count}</p>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Closure mechanism:**

```typescript
// Render 1: count = 0
// useEffect callback created, captures count = 0 in closure
// setInterval callback: () => setCount(0 + 1)

// State updates to 1 (after first tick)
// Render 2: count = 1
// But useEffect doesn't re-run (deps unchanged: [])
// Old setInterval still uses count = 0 closure
```

**Functional update — bypass closure:**

```typescript
setCount(c => c + 1)
// Reducer-like signature
// React applies: state = (c => c + 1)(currentState)
// Always uses latest state from queue
```

**`useRef` mutable — no re-render:**

```tsx
const ref = useRef(0);
ref.current = 5;  // mutate, no re-render
console.log(ref.current);  // 5
```

`useRef.current` — mutable, render bypass. Read latest in interval/timeout closures.

**ESLint rule — `react-hooks/exhaustive-deps`:**

```typescript
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);  // ← lint warning: count missing in deps
  }, 1000);
  return () => clearInterval(id);
}, []);  // ← warning: missing dep 'count'
```

ESLint catches this — but suppressing warning is common (false positive in some patterns).

**Pattern alternatives:**

```tsx
// 1. useReducer — closure-immune
function useCounterWithReducer() {
  const [count, dispatch] = useReducer((s) => s + 1, 0);

  useEffect(() => {
    const id = setInterval(() => dispatch(), 1000);  // dispatch closure stable
    return () => clearInterval(id);
  }, []);

  return count;
}

// 2. useEffectEvent (experimental) — stable callback with latest closure
// (2026 holicha experimental, production'da ishlatish tavsiya etilmaydi)

// 3. requestAnimationFrame for smooth updates
function useTimer() {
  const [count, setCount] = useState(0);
  const ref = useRef(count);
  ref.current = count;

  useEffect(() => {
    let rafId;
    let last = performance.now();

    function tick(now) {
      if (now - last >= 1000) {
        setCount(c => c + 1);
        last = now;
      }
      rafId = requestAnimationFrame(tick);
    }

    rafId = requestAnimationFrame(tick);
    return () => cancelAnimationFrame(rafId);
  }, []);

  return count;
}
```

**Bug detection — Strict Mode:**

Strict Mode 2x effect — if cleanup wrong, double interval (count jumps 2 each tick).

```tsx
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000);
  // ❌ no cleanup
}, []);
// StrictMode: 2 intervals running, count + 2 per tick
```

</details>

### Edge Cases

- **`useEffect` deps include count**: Re-creates interval each tick — wasteful, but correct.
- **`useState` direct value `setCount(count)`**: Same closure issue.
- **Multiple state in interval**: All captured at mount — all stale.

### Follow-up savollar

- "Why React doesn't auto-update closures?" — Closures are JavaScript semantics. React doesn't override.
- "useEffectEvent (experimental) — solves this?" — Ha, stable callback with latest values uchun mo'ljallangan. 2026 holicha experimental.
- "How to test stale closure bug?" — Time-based test (advance fake timers), assert state value after multiple ticks.

</details>

---

<a id="qism-b"></a>

## QISM B: useState

### 8. `useState` API va lazy initializer [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useState(initialValue)`** — `[state, setState]` tuple qaytaradi. **Lazy initializer** — `useState(() => expensiveCompute())` — function pass qilinsa, **faqat mount paytida** chaqiriladi (re-render'larda skip). StrictMode'da 2 marta chaqiriladi, lekin **birinchi natija** saqlanadi. Expensive initial state uchun. `useState(value)` — value har render'da hisoblanadi (lekin faqat mount'da ishlatiladi — performance impact bo'lsa lazy ishlatish).

### To'liq tushuntirish

**API forms:**

```tsx
// 1. Direct value
const [count, setCount] = useState(0);

// 2. Lazy initializer (function)
const [count, setCount] = useState(() => computeInitial());

// 3. Type annotation
const [user, setUser] = useState<User | null>(null);
```

**Lazy initializer benefits:**

- Expensive computation skipped on re-renders
- Function only called once (at mount)
- Same as `useMemo(() => ..., [])` for initial value

### Kod misoli

```tsx
// ❌ Direct value — function called every render
function App() {
  const [items, setItems] = useState(parseLargeData(rawData));
  // parseLargeData runs on EVERY render (but result discarded after first)
  return <List items={items} />;
}

// ✅ Lazy initializer — function called once
function App() {
  const [items, setItems] = useState(() => parseLargeData(rawData));
  // parseLargeData runs ONCE at mount
  return <List items={items} />;
}

// localStorage initialization
function App() {
  const [theme, setTheme] = useState<"light" | "dark">(() => {
    return (localStorage.getItem("theme") as "light" | "dark") ?? "light";
  });
  // localStorage read once at mount
}

// Computed initial state
function PriceCalculator({ items }: { items: Item[] }) {
  const [total, setTotal] = useState(() => {
    return items.reduce((sum, item) => sum + item.price, 0);
  });
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal implementation:**

```typescript
// mountState
function mountState<S>(initialState: S | (() => S)): [S, Dispatch<SetStateAction<S>>] {
  const hook = mountWorkInProgressHook();

  // Lazy initialization
  if (typeof initialState === "function") {
    initialState = (initialState as () => S)();  // call once
  }

  hook.memoizedState = hook.baseState = initialState;

  hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: initialState,
  };

  const dispatch = (hook.queue.dispatch = dispatchSetState.bind(
    null,
    currentlyRenderingFiber,
    hook.queue
  ));

  return [hook.memoizedState, dispatch];
}

// updateState — initial value IGNORED on re-renders
function updateState<S>(initialState: any): [S, Dispatch<SetStateAction<S>>] {
  return updateReducer(basicStateReducer);
  // initialState argument unused (already in hook.memoizedState)
}
```

**Function vs object initial:**

```tsx
// Object — direct OK (cheap)
const [user, setUser] = useState({ name: "Ali", age: 25 });

// Function — for expensive
const [data, setData] = useState(() => loadHeavyData());
```

**Lazy with closure:**

```tsx
function App({ initial }: { initial: number }) {
  // Function captures props at mount time
  const [count, setCount] = useState(() => initial * 2);
  // Subsequent prop changes — count NOT recomputed
  // Use useEffect or key prop to reset
}
```

**Reset state via key:**

```tsx
function ParentResetter({ id }: { id: string }) {
  return <Form key={id} />;  // id change → Form remounts → state fresh
}

function Form() {
  const [value, setValue] = useState(() => loadInitial());
}
```

**Updater function:**

```tsx
const [count, setCount] = useState(0);

// Direct value
setCount(5);

// Updater function (functional update)
setCount(prev => prev + 1);

// When to use functional:
// - Need previous state value
// - Avoid stale closure
// - Multiple updates in batch
```

**StrictMode 2x mount:**

```tsx
function App() {
  const [data] = useState(() => {
    console.log("init");  // dev mode: logs 2x (mount-unmount-mount)
    return computeInitial();
  });
}
```

Lazy initializer pure (must be pure for Strict Mode 2x).

**Compared to other hooks:**

| Hook | Lazy support |
|------|--------------|
| `useState(initial)` | Function pass |
| `useReducer(reducer, initial, init?)` | 3rd arg `init(initial)` for lazy |
| `useRef(initial)` | No lazy (always evaluated) |
| `useMemo(() => x, [])` | Always lazy |

**`useRef` lazy alternative:**

```tsx
// useRef doesn't support lazy
const ref = useRef(expensiveCompute());  // ❌ runs every render

// Workaround
function useLazyRef<T>(init: () => T): React.MutableRefObject<T> {
  const ref = useRef<T | undefined>(undefined);
  if (ref.current === undefined) {
    ref.current = init();
  }
  return ref as React.MutableRefObject<T>;
}

const ref = useLazyRef(() => expensiveCompute());
```

**Performance benchmark:**

```tsx
const [data] = useState(JSON.parse(localStorage.getItem("data") ?? "{}"));
// JSON.parse va localStorage.getItem har render'da chaqiriladi
// (lekin natija faqat mount'da ishlatiladi — quyi render'lar uchun isrof)

const [data] = useState(() => JSON.parse(localStorage.getItem("data") ?? "{}"));
// JSON.parse va localStorage.getItem faqat mount paytida bir marta
```

**TypeScript inference:**

```tsx
// Inferred: number
const [count, setCount] = useState(0);

// Explicit when null/undefined initial
const [user, setUser] = useState<User | null>(null);

// Lazy initializer — return type inferred
const [data, setData] = useState(() => ({ count: 0 }));
// data: { count: number }
```

**Multiple useState vs single object:**

```tsx
// Option A: Multiple useState
const [name, setName] = useState("");
const [age, setAge] = useState(0);
const [email, setEmail] = useState("");

// Option B: Single object
const [user, setUser] = useState({ name: "", age: 0, email: "" });

// Pros/cons:
// A: granular updates, no spread, clearer
// B: atomic updates, but spread on each setState

// Modern React: prefer A unless related fields
```

**Object update pattern:**

```tsx
const [user, setUser] = useState({ name: "Ali", age: 25 });

// ❌ Don't mutate
user.age = 26;
setUser(user);  // same reference, no re-render

// ✅ New object
setUser({ ...user, age: 26 });

// ✅ Functional update
setUser(prev => ({ ...prev, age: 26 }));
```

**`useState` vs `useReducer`:**

```tsx
// useState — simple
const [count, setCount] = useState(0);

// useReducer — complex (multiple actions)
const [state, dispatch] = useReducer(reducer, initial);
```

</details>

### Edge Cases

- **`useState(undefined)` vs `useState()`**: Ikkalasi ham ishlaydi — initial state `undefined`. TS'da `useState<T | undefined>(undefined)` ishlatilgani aniqroq.
- **Lazy with side effect**: Discouraged. Strict Mode 2 marta chaqiradi (purity test).
- **`null` initial type narrowing**: TypeScript needs explicit type — `useState<T | null>(null)`.

### Follow-up savollar

- "Lazy on every render?" — No. `useMemo` for that. `useState` lazy only at mount.
- "Re-init on prop change?" — Use `key` prop or `useEffect`.
- "Multiple useState vs reducer?" — useState for unrelated, reducer for related transitions.

</details>

---

### 9. Functional update vs direct value — qachon qaysi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Direct value** — `setCount(5)` — current closure'dagi qiymatdan foydalanadi (stale closure trap mumkin). **Functional update** — `setCount(prev => prev + 1)` — React queue'da pending update sifatida saqlanib, render time'da prev arg bilan chaqiriladi (stale yo'q). Eski state'ga **bog'liq** update'lar uchun functional kerak (closure trap'dan saqlanish, batching da ketma-ket update'lar).

### To'liq tushuntirish

**Stale closure trap:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);  // count from closure (current render)
    setCount(count + 1);  // same closure value
    setCount(count + 1);  // same closure value
    // After: count = 1 (not 3!)
  };

  return <button onClick={handleClick}>{count}</button>;
}
// Click: 0 → 1 (not 3)
```

**Functional update fix:**

```tsx
const handleClick = () => {
  setCount(prev => prev + 1);  // 0 → 1
  setCount(prev => prev + 1);  // 1 → 2
  setCount(prev => prev + 1);  // 2 → 3
  // After: count = 3
};
```

### Kod misoli

```tsx
// Race condition with async
function Counter() {
  const [count, setCount] = useState(0);

  // ❌ Async — closure may be stale
  const handleAsync = async () => {
    await fetch("/api");
    setCount(count + 1);  // count from closure (potentially old)
  };

  // ✅ Functional — latest state
  const handleAsync = async () => {
    await fetch("/api");
    setCount(prev => prev + 1);
  };

  return <button onClick={handleAsync}>{count}</button>;
}
```

```tsx
// useEffect with interval
function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setSeconds(seconds + 1);  // ❌ stale — seconds always 0 (initial closure)
    }, 1000);
    return () => clearInterval(id);
  }, []);  // empty deps — closure frozen

  return <p>{seconds}</p>;
}
// After 5s: still 0 → 1 (only first tick increments)

// ✅ Functional — always latest
useEffect(() => {
  const id = setInterval(() => {
    setSeconds(prev => prev + 1);
  }, 1000);
  return () => clearInterval(id);
}, []);
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Queue mechanism:**

```typescript
// dispatchSetState
function dispatchSetState(fiber, queue, action) {
  const update = {
    action,
    next: null,
    hasEagerState: false,
    eagerState: null,
  };

  // Eager state computation (optimization)
  if (
    fiber.lanes === NoLanes &&
    (fiber.alternate === null || fiber.alternate.lanes === NoLanes)
  ) {
    // No pending updates — try eager bailout
    const lastRenderedReducer = queue.lastRenderedReducer;
    if (lastRenderedReducer === basicStateReducer) {
      try {
        const currentState = queue.lastRenderedState;
        const eagerState = lastRenderedReducer(currentState, action);
        update.hasEagerState = true;
        update.eagerState = eagerState;

        if (Object.is(eagerState, currentState)) {
          // Same value — bailout (no re-render scheduled)
          return;
        }
      } catch (error) {
        // ignore, fall through
      }
    }
  }

  enqueueUpdate(queue, update);
  scheduleUpdateOnFiber(fiber);
}

// Render time — process queue
function updateReducer(reducer, initial) {
  const hook = updateWorkInProgressHook();
  let newState = hook.baseState;

  const pending = hook.queue.pending;
  if (pending !== null) {
    const first = pending.next;
    let update = first;
    do {
      const action = update.action;
      newState = reducer(newState, action);  // basicStateReducer
      update = update.next;
    } while (update !== first);
    hook.queue.pending = null;
  }

  hook.memoizedState = newState;
  return [newState, hook.queue.dispatch];
}

function basicStateReducer(state, action) {
  return typeof action === "function" ? action(state) : action;
}
```

**Direct value vs functional in queue:**

```typescript
// setCount(5) — direct
queue.pending = { action: 5 };
// Render: newState = basicStateReducer(prevState, 5) = 5

// setCount(prev => prev + 1) — functional
queue.pending = { action: prev => prev + 1 };
// Render: newState = basicStateReducer(prevState, prev => prev + 1) = prev + 1
```

**Multiple updates in queue:**

```typescript
// setCount(5); setCount(prev => prev + 1); setCount(10);
queue.pending = [
  { action: 5 },
  { action: prev => prev + 1 },
  { action: 10 }
];

// Render time:
// state = 0 (initial)
// state = 5 (direct)
// state = 6 (functional: prev=5)
// state = 10 (direct overrides)
// memoizedState = 10
```

**Eager state optimization:**

```typescript
// If no pending updates and reducer simple:
setCount(0);  // current state already 0
// → eagerState = 0
// → Object.is(0, 0) = true
// → BAILOUT — no re-render scheduled
```

This avoids unnecessary renders.

**Functional update — same value bailout:**

```tsx
const [count, setCount] = useState(0);

setCount(prev => prev);  // returns same value
// React: Object.is(prev, prev) = true → bailout
```

**Closure capture per render:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  // Each render: new closure with current count value

  const handleClick = () => {
    console.log(count);  // captured at this render
    setCount(count + 1);  // captured at this render
  };

  return <button onClick={handleClick}>{count}</button>;
}

// Render 1: count=0, handleClick captures 0
// Click → setCount(1) → re-render
// Render 2: count=1, handleClick (new) captures 1
// Each render — fresh closure
```

**Why functional safer:**

```tsx
// Multiple setState in batch
setCount(count + 1);  // count=0 closure
setCount(count + 1);  // count=0 closure (same)
setCount(count + 1);  // count=0 closure (same)
// All call setState with action=1
// Queue: [1, 1, 1]
// Render: state = 1 (final)

// vs functional
setCount(prev => prev + 1);  // queue: [fn]
setCount(prev => prev + 1);  // queue: [fn, fn]
setCount(prev => prev + 1);  // queue: [fn, fn, fn]
// Render: state = 0 → 1 → 2 → 3
```

**`useReducer` for complex state:**

```tsx
type Action = { type: "increment" } | { type: "reset" };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case "increment": return state + 1;
    case "reset": return 0;
  }
}

const [count, dispatch] = useReducer(reducer, 0);

// dispatch — always uses latest state (via reducer)
dispatch({ type: "increment" });  // no closure issue
```

**`flushSync` vs functional:**

```tsx
// flushSync — sync render between updates
flushSync(() => setCount(count + 1));
// After flushSync, count updated
flushSync(() => setCount(count + 1));  // count from CURRENT render (after first sync)

// vs functional (no flushSync needed)
setCount(prev => prev + 1);
setCount(prev => prev + 1);
```

**TypeScript:**

```tsx
const [count, setCount] = useState(0);

setCount(5);  // (value: number) => void
setCount(prev => prev + 1);  // (updater: (prev: number) => number) => void

// Combined type
type Dispatch<S> = (value: S | ((prevState: S) => S)) => void;
```

**Performance:**

- Direct: faster (no function call)
- Functional: slight overhead (closure call)
- Real apps: negligible difference

Use functional when correctness needed, direct otherwise.

</details>

### Edge Cases

- **`setCount(undefined)`**: state = `undefined`. `Object.is(prev, undefined)` — agar oldingi state ham `undefined` bo'lsa, bailout.
- **`setCount(NaN)`**: Object.is(NaN, NaN) = true (special case) → bailout if prev NaN.
- **Functional throws**: Throws during render. React catches, error boundary handles.

### Follow-up savollar

- "ESLint catches direct in stale context?" — `react-hooks/exhaustive-deps` partially. Doesn't fully detect stale closure.
- "Functional in event handler — overkill?" — Independent updates direct OK. Dependent updates — functional safer.
- "useReducer always solves stale?" — Yes. dispatch always uses latest state via reducer.

</details>

---

### 10. State queue mexanizmi — circular linked list [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useState`'ning queue — **circular linked list**. Har `setState` chaqiruvi `Update` object yaratadi va queue'ning oxiriga qo'shadi (`pending` pointer doim oxirgi update'ga ishora qiladi). Render time'da queue traverse qilinadi (head → tail → head, circular), reducer har action'ga qo'llanadi. Tugagach queue tozalanadi. Multi-priority lanes uchun **priority filtering** ham bor.

### To'liq tushuntirish

**Circular linked list:**

```typescript
type Update = {
  lane: Lane;
  action: any;
  next: Update;          // points to NEXT update (circular)
  hasEagerState: boolean;
  eagerState: any;
};

// queue.pending — pointer to LAST update (tail)
// queue.pending.next — first update (head)

// Empty queue: pending = null
// 1 update: pending = update; update.next = update; (self-loop)
// 2+ updates: pending = lastUpdate; lastUpdate.next = firstUpdate;
```

### Kod misoli

```typescript
// Visualization: 3 updates added
// setCount(1); setCount(2); setCount(3);

// Queue state:
// pending → update3 (action=3)
//             ↓ next
//           update1 (action=1)
//             ↓ next
//           update2 (action=2)
//             ↓ next
//           update3 (action=3) ← circular back

// Traversal (render time):
// first = pending.next = update1
// process update1 (state = 1)
// process update2 (state = 2)
// process update3 (state = 3)
// stop when update.next === first (circular complete)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`enqueueUpdate` implementation:**

```typescript
function enqueueUpdate(queue, update) {
  const pending = queue.pending;

  if (pending === null) {
    // Empty queue — self-loop
    update.next = update;
  } else {
    // Insert at end of circular list
    update.next = pending.next;  // new update points to old head
    pending.next = update;        // old tail points to new update
  }

  queue.pending = update;  // pending = new tail
}
```

**Why circular:**

- Allows O(1) append at tail
- Easy traversal (start from `pending.next` = head, stop at `pending.next` = head again)
- No null check needed mid-traverse

**Render-time queue processing:**

```typescript
function updateReducer<S, A>(reducer: (state: S, action: A) => S, initial: any): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  queue.lastRenderedReducer = reducer;

  const current = currentHook;

  // Move pending to baseQueue (in case of priority filtering)
  let baseQueue = current.baseQueue;
  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
    if (baseQueue !== null) {
      // Merge baseQueue and pendingQueue
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;
    }
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }

  if (baseQueue !== null) {
    const first = baseQueue.next;
    let newState = current.baseState;

    let newBaseState: S | null = null;
    let newBaseQueueFirst: Update<S, A> | null = null;
    let newBaseQueueLast: Update<S, A> | null = null;
    let update = first;

    do {
      const updateLane = update.lane;
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // Skip — not in render lanes (priority filtering)
        const clone: Update<S, A> = {
          lane: updateLane,
          action: update.action,
          hasEagerState: update.hasEagerState,
          eagerState: update.eagerState,
          next: null,
        };

        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }

        // Mark fiber's lanes as still pending
        currentlyRenderingFiber.lanes = mergeLanes(currentlyRenderingFiber.lanes, updateLane);
        markSkippedUpdateLanes(updateLane);
      } else {
        // In render lanes — process
        if (newBaseQueueLast !== null) {
          // Add to base queue (preserve for next render)
          newBaseQueueLast = newBaseQueueLast.next = {
            lane: NoLane,
            action: update.action,
            hasEagerState: update.hasEagerState,
            eagerState: update.eagerState,
            next: null,
          };
        }

        // Apply action
        if (update.hasEagerState) {
          newState = update.eagerState;
        } else {
          const action = update.action;
          newState = reducer(newState, action);
        }
      }

      update = update.next;
    } while (update !== first);

    // Update hook
    if (newBaseQueueLast === null) {
      newBaseState = newState;
    } else {
      newBaseQueueLast.next = newBaseQueueFirst;
    }

    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;

    queue.lastRenderedState = newState;
  }

  return [hook.memoizedState, queue.dispatch];
}
```

**Priority filtering — concurrent rendering:**

```typescript
// User clicks → SyncLane setState
// Background → DefaultLane setState

// renderLanes = SyncLane (high priority render first)
// Queue:
//   update1 (lane: SyncLane)
//   update2 (lane: DefaultLane)
//   update3 (lane: SyncLane)

// Process:
//   update1 (SyncLane in renderLanes) → apply
//   update2 (DefaultLane NOT in SyncLane) → skip, save to baseQueue
//   update3 (SyncLane in renderLanes) → apply

// After render:
//   memoizedState = result of update1, update3
//   baseQueue = update2 (preserved for next render)
//   pendingLanes = DefaultLane

// Next render with DefaultLane:
//   Process baseQueue (update2)
//   memoizedState = result of update1 + update2 + update3
```

**`baseState` and `baseQueue`:**

- `memoizedState` — current state shown in UI
- `baseState` — state at start of pending updates (before any priority skipped)
- `baseQueue` — updates not yet applied (saved across renders)

For consistency: low-pri update from before high-pri update applied first when low-pri renders.

**Eager state — bailout optimization:**

```typescript
// In dispatchSetState:
function dispatchSetState(fiber, queue, action) {
  // Eager state — if no pending updates, compute eagerly
  if (
    fiber.lanes === NoLanes &&
    (fiber.alternate === null || fiber.alternate.lanes === NoLanes)
  ) {
    const lastRenderedReducer = queue.lastRenderedReducer;
    if (lastRenderedReducer !== null) {
      try {
        const currentState = queue.lastRenderedState;
        const eagerState = lastRenderedReducer(currentState, action);

        update.hasEagerState = true;
        update.eagerState = eagerState;

        if (Object.is(eagerState, currentState)) {
          // Same value — bailout, no re-render
          return;
        }
      } catch {
        // ignore
      }
    }
  }

  // ... enqueue update, schedule render
}
```

**Stale closure within setState updater:**

```tsx
const [count, setCount] = useState(0);

const handleClick = () => {
  setCount(prev => {
    console.log(count);  // count from CLOSURE (might be stale)
    return prev + 1;     // prev — latest from queue
  });
};
```

**Render time vs dispatch time:**

```tsx
// Dispatch time (event handler)
setCount(prev => prev + 1);  // queued, not yet executed

// Render time
// Updater function called with prev = current state
// Returns new state
// React updates memoizedState
```

**Multiple components, same state hook:**

```tsx
// Each component instance — own state, own queue
function Counter() {
  const [count, setCount] = useState(0);  // separate hook per instance
}

<Counter />  // instance 1 with own queue
<Counter />  // instance 2 with own queue
```

**`flushSync` impact on queue:**

```typescript
// flushSync forces immediate render
flushSync(() => {
  setCount(1);
});
// → enqueue update, schedule sync render, flush

// After flushSync — queue processed, state committed
flushSync(() => {
  setCount(prev => prev + 1);  // prev = 1 (latest)
});
```

**Functional update inside class component:**

```tsx
class Counter extends Component {
  state = { count: 0 };

  handleClick = () => {
    this.setState(prevState => ({ count: prevState.count + 1 }));
    // Same pattern as functional useState
  };
}
```

**Memory pressure:**

- Har Update object — `action`, `next`, `hasEagerState`, `eagerState`, `lane` field'lari
- Render time'da queue traverse + clear qilinadi (`queue.pending = null`)
- Updates yangi render'da qaytadan qo'shiladi

</details>

### Edge Cases

- **Update during render**: Queued for re-render, processed in next cycle.
- **Updater throws**: React catches, error boundary handles.
- **Same value bailout**: Object.is check at queue processing.

### Follow-up savollar

- "Why circular linked list?" — O(1) append, easy traverse, no separate head/tail.
- "Priority lanes filtering — overhead?" — Minor. Most updates same lane (no skip).
- "Can I clear queue manually?" — No. React internal.

</details>

---

### 11. State equality bailout — `Object.is` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useState`'da agar yangi state value joriy state'ga **`Object.is`** orqali teng bo'lsa, React **re-render skip** qiladi (bailout). `Object.is` — `===` ga o'xshash, lekin ikki farq bor: `Object.is(NaN, NaN)` true qaytaradi (`===` da false), `Object.is(+0, -0)` false qaytaradi (`===` da true). Reference equality — object'larda yangi reference (har gal yangi `{...obj}`) — bailout ishlamaydi. Optimization: same primitive value setState → no render.

### To'liq tushuntirish

**`Object.is` semantics:**

```typescript
Object.is(1, 1)              // true
Object.is("a", "a")          // true
Object.is(null, null)        // true
Object.is(undefined, undefined)  // true
Object.is(NaN, NaN)          // true  (=== da false)
Object.is(+0, -0)            // false (=== da true)

Object.is({}, {})            // false (different references)
Object.is([], [])            // false
Object.is(fn, fn)            // true (same reference)
```

### Kod misoli

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  console.log("render");

  return (
    <button onClick={() => setCount(0)}>  {/* same value */}
      {count}
    </button>
  );
}

// Mount: "render"
// Click 1: setCount(0)
//   Object.is(0, 0) = true → bailout
//   No "render" log
// Click 2: setCount(0)
//   Same — bailout
//   No log

// vs setCount(c => 0)
function Counter2() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => 0)}>{count}</button>;
}
// Functional — runs reducer first, then check
// Reducer returns 0 (current 0) — eager state bailout if no other updates
// (but functional updaters don't always benefit from eager state)
```

**Object reference issue:**

```tsx
function App() {
  const [user, setUser] = useState({ name: "Ali" });

  const handleClick = () => {
    setUser({ name: "Ali" });  // ❌ new object reference
    // Object.is({...}, {...}) = false → re-render
  };
}

// vs same reference
const userObj = { name: "Ali" };
const handleClick = () => {
  setUser(userObj);  // same reference
  // Object.is(prev, userObj) — depends if prev is userObj
};
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Bailout in dispatchSetState (eager):**

```typescript
function dispatchSetState(fiber, queue, action) {
  // Eager state computation
  if (
    fiber.lanes === NoLanes &&
    (fiber.alternate === null || fiber.alternate.lanes === NoLanes)
  ) {
    const lastRenderedReducer = queue.lastRenderedReducer;
    if (lastRenderedReducer === basicStateReducer) {
      try {
        const currentState = queue.lastRenderedState;
        const eagerState = basicStateReducer(currentState, action);

        update.hasEagerState = true;
        update.eagerState = eagerState;

        if (Object.is(eagerState, currentState)) {
          // Same value — early bailout, don't even schedule
          return;
        }
      } catch {
        // ignore
      }
    }
  }

  // ... otherwise enqueue and schedule
  enqueueUpdate(queue, update);
  scheduleUpdateOnFiber(fiber);
}
```

**Render-time bailout:**

```typescript
// In updateReducer, after processing queue:
if (Object.is(newState, hook.memoizedState)) {
  // No actual change
  // markUpdateLaneFromFiberToRoot still ran (during dispatch)
  // But Reconciler will detect didReceiveUpdate = false
  didReceiveUpdate = false;
}
```

**Two layers of bailout:**

1. **Dispatch-time eager bailout**: Before scheduling render. Most efficient.
2. **Render-time bailout**: During render, after queue processed. Component still rendered, but children may bail.

**Object literal trap:**

```tsx
const [config, setConfig] = useState({ theme: "light", lang: "en" });

// Multiple paths to same logical state
setConfig({ theme: "light", lang: "en" });  // new ref
setConfig(prev => ({ ...prev, theme: "light" }));  // new ref

// Both trigger re-render — same content but different reference
```

**Mitigation: structural comparison:**

```tsx
function setStateIfChanged<T>(
  setState: (value: T) => void,
  currentValue: T,
  newValue: T
) {
  if (!isEqual(currentValue, newValue)) {  // lodash deep equal
    setState(newValue);
  }
}

// Or memoize source
const config = useMemo(() => ({ theme, lang }), [theme, lang]);
setConfig(config);
```

**`Object.is` vs `===`:**

```typescript
// Object.is special cases:
Object.is(NaN, NaN)  // true
Object.is(+0, -0)    // false

NaN === NaN          // false
+0 === -0            // true
```

React `Object.is` tanladi — `NaN` self-equality avoids unnecessary re-render when state legitimately NaN. `===` da `NaN !== NaN` — har `setCount(NaN)` re-render trigger qilardi.

**Performance impact:**

```tsx
// Without bailout
setCount(0);  // count is 0
// → re-render, children re-render, expensive work

// With bailout
setCount(0);
// → no render, no work
```

Especially valuable for:
- Frequent setState calls (e.g., scroll handler updating same value)
- Memoized children (avoid invalidating memo)

**Lazy initial value not affected:**

```tsx
const [count, setCount] = useState(0);

// Initial value comparison happens once at mount (not bailout)
// Bailout: subsequent setState matches current state
```

**Class component analog:**

```tsx
class Counter extends Component {
  state = { count: 0 };

  handleClick = () => {
    this.setState({ count: 0 });
    // Class — NO automatic bailout
    // Always re-renders unless shouldComponentUpdate returns false
  };
}

// Hooks have built-in bailout, class doesn't (need manual implementation)
```

**`PureComponent` (class) — shallow:**

```tsx
class Counter extends PureComponent {
  state = { count: 0 };

  handleClick = () => {
    this.setState({ count: 0 });
    // PureComponent — shallow check
    // { count: 0 } vs { count: 0 } — different references, but shallow keys equal
    // Bailout
  };
}
```

PureComponent — shallow comparison. useState — strict Object.is.

**Functional setState eager state:**

```tsx
const [count, setCount] = useState(0);
setCount(prev => prev);  // returns prev

// Eager: if no pending updates, run reducer eagerly
// reducer(0, prev => prev) = 0
// Object.is(0, 0) = true → bailout
```

But if other updates pending:
```tsx
setCount(prev => prev + 1);  // queued
setCount(prev => prev);      // can't eager bailout (other updates may change state)
// Both queued, processed at render
```

**`useReducer` similar:**

```tsx
const [state, dispatch] = useReducer(reducer, initial);

dispatch({ type: "noop" });  // reducer returns same state
// Object.is(same, same) = true → bailout
```

**Strict Mode 2x:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  console.log("render");
  // ...
}

// Strict Mode dev:
// Mount → "render" logged twice (intentional 2x)
// setCount(0) → bailout (no re-render)
```

Bailout works in Strict Mode same as production.

</details>

### Edge Cases

- **`setCount(NaN)`**: Object.is(NaN, NaN) = true. Bailout if was NaN.
- **`setCount(-0)` after `+0`**: Object.is(+0, -0) = false. Will re-render.
- **`setUser(user)` (same reference)**: Bailout. But user object mutation untracked — UI may not reflect changes.

### Follow-up savollar

- "Custom equality?" — useState no API for custom. useMemo or selector pattern for custom equality.
- "Bailout transitive — children?" — Yes. If parent bails, children skip too (same lane).
- "Bailout in concurrent rendering?" — Same. Render-time bailout based on memoizedState comparison.

</details>

---

### 12. `useState` initializer — lazy, fan-out [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useState(initialValue)` — initial value har render'da hisoblanadi (lekin faqat ilk render'da ishlatiladi). **Lazy initializer** — `useState(() => expensiveCompute())` — function faqat ilk render'da chaqiriladi. Use case: localStorage parse, expensive default. StrictMode 2x — initializer 2 marta chaqiriladi (purity check).

### Kod misoli

```tsx
// ❌ Each render: localStorage parse (expensive)
function BadCounter() {
  const [count, setCount] = useState(JSON.parse(localStorage.getItem("count") ?? "0"));
  return <p>{count}</p>;
}

// ✅ Lazy — only first render
function GoodCounter() {
  const [count, setCount] = useState(() => {
    const stored = localStorage.getItem("count");
    return stored ? JSON.parse(stored) : 0;
  });
  return <p>{count}</p>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal:**

```typescript
function mountState(initialState) {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = typeof initialState === "function"
    ? initialState()  // lazy
    : initialState;   // direct value
  // ...
}
```

**StrictMode 2x:**

```tsx
const [data] = useState(() => {
  console.log("init");  // dev: 2 marta (StrictMode purity check)
  return fetchInitialDataSync();  // ⚠️ if has side effect, 2 marta!
});
```

Pure initializer essential.

**Initial value vs initial computation:**

```tsx
// Direct value — cheap (no harm to recompute)
useState(0);
useState("");
useState({ a: 1 });  // ⚠️ new reference each render, but value ignored after mount

// Lazy — for expensive
useState(() => expensiveCompute());
useState(() => deserialize(localStorage.getItem("x")));
```

**`useReducer` lazy init:**

```tsx
const [state, dispatch] = useReducer(
  reducer,
  props.id,           // initial arg
  (id) => loadById(id) // lazy init function (3rd param)
);
```

</details>

### Edge Cases

- **Function as initial state**: `useState(myFn)` — React calls myFn (treats as lazy). To store function as state: `useState(() => myFn)`.
- **Async initializer**: `useState(async () => fetch(...))` — returns Promise as state value (anti-pattern).
- **State setter with function**: `setX(fn)` — React calls fn (functional update). Same trap.

### Follow-up savollar

- "When to use lazy init?" — Expensive compute (parse JSON, decode), localStorage read, large initial state.
- "Compiler — auto lazy?" — Compiler doesn't auto-wrap. Manual.

</details>

---

<a id="qism-c"></a>

## QISM C: useEffect

### 13. `useEffect` "lifecycle hook" EMAS — sinxronizatsiya mexanizmi [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useEffect` — "lifecycle hook" emas, balki "sinxronizatsiya mexanizmi"**. Class komponentlardagi `componentDidMount` + `componentDidUpdate` + `componentWillUnmount` mexanik birlashishi **emas**. React rasmiy filosofiyasi: *"Effects are NOT lifecycle events — they are synchronization mechanisms"*. To'g'ri mental model: "tashqi tizim bilan **sinxron bo'lib tur**" (declarative). Lifecycle: "X paytida Y qil" (imperative). Bu farq Strict Mode 2x effect, deps array, "You Might Not Need an Effect" — barchasi sinxronizatsiya modelidan kelib chiqadi.

### To'liq tushuntirish

**Lifecycle mental model (XATO):**

```
Class lifecycle:
- Mount: "do X once when component appears"
- Update: "do X again when props change"
- Unmount: "undo X when component disappears"

useEffect translation (NOTO'G'RI):
useEffect(() => doX(), []) ≈ componentDidMount
useEffect(() => doX(), [deps]) ≈ componentDidUpdate
useEffect(() => doX(); return undoX, []) ≈ componentDidMount + componentWillUnmount
```

**Sinxronizatsiya mental model (TO'G'RI):**

```
Effect = "synchronize component with external system"
- "While component exists, keep X synced with state"
- "If state changes, re-sync"
- "If component unmounts, stop synchronization"

Effect is declarative:
"For these inputs, this is the side effect needed"
React handles when to apply.
```

### Kod misoli

**Lifecycle mental model (anti-pattern):**

```tsx
// ❌ "Mount paytida fetch"
function User({ id }: { id: string }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(id).then(setUser);
  }, []);  // ❌ id deps yo'q — id o'zgarsa, fetch ishlamaydi!

  return <div>{user?.name}</div>;
}

// User id="1" → fetch user 1, ko'rinadi
// id="2" props o'zgardi — fetch ishga tushmaydi (deps boshqa)
```

**Sinxronizatsiya mental model:**

```tsx
// ✅ "User komponent ↔ id'ga mos user data"
function User({ id }: { id: string }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    let cancelled = false;
    fetchUser(id).then((data) => {
      if (!cancelled) setUser(data);
    });
    return () => { cancelled = true; };
  }, [id]);  // ✅ id'ga bog'liq — id o'zgarsa, re-sync

  return <div>{user?.name}</div>;
}

// id="1" → fetch user 1
// id="2" → cleanup (cancel old) + new fetch user 2
// Always synced with current id
```

<details>
<summary><strong>Deep Dive</strong></summary>

**React docs philosophy (react.dev):**

> "Effects let you specify side effects that are caused by rendering itself, rather than by a particular event."
> "Effects let you 'step outside' of React and synchronize your components with some external system like a non-React widget, network, or the browser DOM."

Source: react.dev/learn/synchronizing-with-effects

**Why "lifecycle" thinking causes bugs:**

```tsx
// ❌ Lifecycle thinking
function Form({ initialName }: { initialName: string }) {
  const [name, setName] = useState("");

  useEffect(() => {
    setName(initialName);  // "mount paytida set"
  }, []);

  // Bug: initialName o'zgarsa, name yangilanmaydi
}

// ✅ Synchronization thinking
function Form({ initialName }: { initialName: string }) {
  // initialName changes → re-sync via key
  return <FormInternal key={initialName} initialName={initialName} />;
}

function FormInternal({ initialName }: { initialName: string }) {
  const [name, setName] = useState(initialName);  // initial value
  // No effect needed — React handles via key remount
}
```

**"Mount → cleanup → mount" cycle (Strict Mode 2x):**

```tsx
useEffect(() => {
  console.log("setup");
  return () => console.log("cleanup");
}, []);

// Strict Mode dev:
// "setup"
// "cleanup"  ← simulated unmount
// "setup"    ← simulated remount

// React: "If your effect can survive remount cycle, it's correctly synchronizing.
//         If it breaks, you're using lifecycle pattern (one-time mount)."
```

**Why 2x exposes lifecycle bugs:**

```tsx
// Lifecycle pattern
useEffect(() => {
  // "Mount paytida bir marta fetch"
  fetch("/api").then(setData);
}, []);

// Strict Mode 2x:
// fetch 1 → /api response 1
// (mount cycle, no cleanup)
// fetch 2 → /api response 2 (race condition!)
// First response may overwrite second

// Synchronization pattern
useEffect(() => {
  const controller = new AbortController();
  fetch("/api", { signal: controller.signal }).then(setData);
  return () => controller.abort();
}, []);

// Strict Mode 2x:
// fetch 1 → controller.abort() (cleanup) → fetch 1 cancelled
// fetch 2 → response (only this matters)
// Race condition handled
```

**Effects vs event handlers:**

```tsx
// Event handler — "on click, do X"
function Form() {
  const handleSubmit = () => {
    submitForm(data);  // imperative, single action
  };

  return <button onClick={handleSubmit}>Submit</button>;
}

// Effect — "while submitting, keep UI synced"
function Form() {
  const [submitting, setSubmitting] = useState(false);

  useEffect(() => {
    if (!submitting) return;
    submitForm(data).then(() => setSubmitting(false));
  }, [submitting, data]);
  // Synchronizes "submitting state" with "submission process"
}
```

Most actions belong in event handlers (imperative), not effects.

**"You Might Not Need an Effect" (Dan Abramov):**

```tsx
// ❌ Effect for derived state
function Profile({ user }) {
  const [fullName, setFullName] = useState("");

  useEffect(() => {
    setFullName(`${user.firstName} ${user.lastName}`);
  }, [user]);

  return <div>{fullName}</div>;
}

// ✅ Compute during render
function Profile({ user }) {
  const fullName = `${user.firstName} ${user.lastName}`;
  return <div>{fullName}</div>;
}
```

```tsx
// ❌ Effect for cached computation
function List({ items, query }) {
  const [filtered, setFiltered] = useState([]);

  useEffect(() => {
    setFiltered(items.filter(i => i.name.includes(query)));
  }, [items, query]);

  return <ul>{filtered.map(...)}</ul>;
}

// ✅ useMemo or compute directly
function List({ items, query }) {
  const filtered = useMemo(
    () => items.filter(i => i.name.includes(query)),
    [items, query]
  );
  return <ul>{filtered.map(...)}</ul>;
}
```

```tsx
// ❌ Effect to handle event
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    if (count > 0) {
      sendAnalytics("incremented", count);  // ❌ event-like
    }
  }, [count]);
}

// ✅ Event handler
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    const newCount = count + 1;
    setCount(newCount);
    sendAnalytics("incremented", newCount);  // ✅ event
  };
}
```

**When effects ARE correct:**

```tsx
// External system synchronization
useEffect(() => {
  const subscription = externalAPI.subscribe(handler);
  return () => subscription.unsubscribe();
}, []);

// DOM API
useEffect(() => {
  document.title = title;
  return () => { document.title = "Default"; };
}, [title]);

// Network requests (synchronizing local state with server)
useEffect(() => {
  let cancelled = false;
  fetch(url).then(data => { if (!cancelled) setData(data); });
  return () => { cancelled = true; };
}, [url]);

// Animations
useEffect(() => {
  const animation = animate(element, { from, to });
  return () => animation.stop();
}, [from, to]);

// Browser API
useEffect(() => {
  const handler = () => setSize(window.innerWidth);
  window.addEventListener("resize", handler);
  return () => window.removeEventListener("resize", handler);
}, []);
```

**Class lifecycle to effect — DON'T translate mechanically:**

```tsx
// Class:
componentDidMount() {
  fetchData(this.props.id).then(data => this.setState({ data }));
}

componentDidUpdate(prevProps) {
  if (prevProps.id !== this.props.id) {
    fetchData(this.props.id).then(data => this.setState({ data }));
  }
}

componentWillUnmount() {
  // ... cleanup if any
}

// Effect — synchronization (not 3 separate methods)
useEffect(() => {
  let cancelled = false;
  fetchData(id).then(data => { if (!cancelled) setData(data); });
  return () => { cancelled = true; };
}, [id]);
```

The effect handles all 3 lifecycle methods because it's about **keeping data in sync with id**, not "do X at lifecycle moment Y."

**Mental model exercise:**

Ask: "What is this effect synchronizing?"

| Answer | Effect | Notes |
|--------|--------|-------|
| Subscription with id | `useEffect(() => sub(id), [id])` | OK |
| Document title with state | `useEffect(() => { doc.title = x }, [x])` | OK |
| Computed value | NOT an effect — compute in render |
| Event response | NOT an effect — event handler |
| Initial-only setup | NOT an effect (probably) — useState lazy |

</details>

### Edge Cases

- **`useEffect` for analytics**: Often debate — event handler better for action analytics. Effect OK for "page view" type.
- **Subscribe in render — race**: `useEffect` for subscribe (not in render body — pure render rule).
- **Conditional sync**: `useEffect(() => { if (!enabled) return; sync(); }, [enabled])` — conditional inside.

### Follow-up savollar

- "Class lifecycle methods deprecated?" — `componentDidMount`/`componentDidUpdate`/`componentWillUnmount` hali ishlaydi. `UNSAFE_componentWillMount`/`UNSAFE_componentWillReceiveProps`/`UNSAFE_componentWillUpdate` — R16.3'da `UNSAFE_` prefix qo'shildi (deprecated), kelajakda olib tashlanishi mumkin. Function komponentlar — hooks tavsiya etiladi.
- "useEffect bilan async?" — Effect callback can't be async (return value reserved for cleanup). Wrap async inside.
- "useEffect once vs lifecycle?" — Often misuse. `[]` deps for "run once" is lifecycle thinking. Synchronization with no deps usually wrong.

</details>

---

### 14. Dependency array — `[]`, `[deps]`, no array [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Dependency array** — effect qachon qayta chaqirilishini boshqaradi: (1) **No array** (`useEffect(fn)`) — har render'da ishlaydi, (2) **Empty `[]`** — faqat mount va unmount'da, (3) **`[a, b]`** — `a` yoki `b` `Object.is` orqali o'zgarsa qayta ishlaydi. Deps ichida **all reactive values** (props, state, function/object referenced inside effect) bo'lishi shart — exhaustive-deps ESLint rule.

### To'liq tushuntirish

**3 ta forma:**

```tsx
// 1. No array — every render
useEffect(() => {
  console.log("every render");
});
// Mount, every update — runs

// 2. Empty array — once (mount only)
useEffect(() => {
  console.log("mount");
  return () => console.log("unmount");
}, []);

// 3. With deps — when deps change
useEffect(() => {
  console.log("count changed:", count);
}, [count]);
// Mount + when count changes
```

**Comparison: `Object.is`:**

```tsx
useEffect(() => { /* ... */ }, [user]);
// React compares prevDeps[0] vs newDeps[0] via Object.is
// user (object) — reference comparison
// New object every render → effect runs every render
```

### Kod misoli

```tsx
// ❌ Missing dep — stale closure
function Counter({ initial }: { initial: number }) {
  const [count, setCount] = useState(initial);

  useEffect(() => {
    document.title = `Count: ${count}`;
  }, []);  // ❌ count missing
  // First render: title = "Count: 0"
  // count changes: effect doesn't run, title stays "Count: 0"
}

// ✅ Exhaustive deps
useEffect(() => {
  document.title = `Count: ${count}`;
}, [count]);

// ❌ Object dep — runs every render
function App({ user }: { user: User }) {
  const config = { theme: "light" };  // new object every render

  useEffect(() => {
    init(config);
  }, [config]);  // config new ref every render → effect runs every render
}

// ✅ Stable reference
const config = useMemo(() => ({ theme: "light" }), []);
useEffect(() => {
  init(config);
}, [config]);

// ✅ Inline value
useEffect(() => {
  init({ theme: "light" });
}, []);  // no config dep needed

// ❌ Function dep — same problem
function App() {
  const handleData = (data) => process(data);  // new function every render

  useEffect(() => {
    fetchData().then(handleData);
  }, [handleData]);  // runs every render
}

// ✅ useCallback
const handleData = useCallback((data) => process(data), []);

// ✅ Inline (move into effect)
useEffect(() => {
  const handleData = (data) => process(data);
  fetchData().then(handleData);
}, []);
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal: deps comparison:**

```typescript
function updateEffect(create, deps) {
  const hook = updateWorkInProgressHook();
  const prevEffect = hook.memoizedState;

  if (deps !== null && prevEffect !== null) {
    const prevDeps = prevEffect.deps;
    if (areHookInputsEqual(deps, prevDeps)) {
      // Deps unchanged — skip effect
      hook.memoizedState = prevEffect;
      return;
    }
  }

  // Deps changed (or no deps) — schedule effect
  currentlyRenderingFiber.flags |= PassiveEffect;
  hook.memoizedState = pushEffect(create, prevEffect.destroy, deps);
}

function areHookInputsEqual(deps, prevDeps) {
  if (prevDeps === null) return false;
  if (deps.length !== prevDeps.length) {
    console.warn("Deps array length changed!");
    return false;
  }
  for (let i = 0; i < deps.length; i++) {
    if (!Object.is(deps[i], prevDeps[i])) return false;
  }
  return true;
}
```

**`exhaustive-deps` ESLint rule:**

```json
{
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

Warns when:
- Reactive value used inside effect but not in deps
- Stale closure risk

```tsx
function Component({ id }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetchData(id).then(setData);
    // ESLint: React Hook useEffect has missing dependency: 'id'
  }, []);  // missing id
}
```

**Why deps matter (synchronization):**

```tsx
// "Sync data with id"
useEffect(() => {
  fetchData(id).then(setData);
}, [id]);

// Mount: id=1 → fetch user 1
// id=2: deps changed → cleanup + new effect → fetch user 2
// id=1 again: deps changed (1 → 2 → 1) → fetch user 1 again
```

**Common deps mistakes:**

```tsx
// ❌ 1. Object dep
useEffect(() => {
  init({ option: true });
}, [{ option: true }]);  // ❌ new object every render
// Effect runs every render

// ✅ Primitive
useEffect(() => {
  init({ option });
}, [option]);

// ❌ 2. Function dep
useEffect(() => {
  callback();
}, [callback]);  // callback new ref → runs every render

// ✅ useCallback or move inside
const callback = useCallback(() => action(), []);

// ❌ 3. State setter as "dep"
useEffect(() => {
  setData(newData);
}, [setData]);  // ⚠️ setData stable, but "feels" like dep
// React guarantees setState is stable — don't add to deps

// ✅ Just use without deps
useEffect(() => {
  setData(newData);
}, [newData]);
```

**Reactive vs non-reactive:**

```tsx
// Reactive (changes between renders) — must be in deps
const props.id;       // can change
state.count;          // can change
context value;        // can change
non-stable references; // function/object literals

// Non-reactive (stable) — DON'T add to deps
useState's setter;   // stable
useReducer's dispatch; // stable
useRef's ref object;  // stable
useRef.current;        // changes but not reactive (ref pattern)
```

**Empty deps `[]` — common errors:**

```tsx
// ❌ Effect uses props but [] deps
function Greeting({ name }) {
  useEffect(() => {
    console.log("Hello", name);
  }, []);  // name will always be initial render's name (stale)
}

// ✅ Deps include name
useEffect(() => {
  console.log("Hello", name);
}, [name]);
```

**No deps — runs every render (rare):**

```tsx
// ⚠️ Often a bug — every render = side effect every render
function Bad() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `Count: ${count}`;
  });  // no deps — every render
  // OK here, but verbose. Better with [count].
}
```

Generally `[]` or `[deps]` — explicit deps clearer.

**Stale closure with deps:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);  // ❌ count from closure (initial 0)
    }, 1000);
    return () => clearInterval(id);
  }, []);  // bo'sh deps — closure frozen at initial render

  // After 5s: count stays at 1 (only first tick)
}

// ✅ Functional update — no closure dep
useEffect(() => {
  const id = setInterval(() => {
    setCount(prev => prev + 1);
  }, 1000);
  return () => clearInterval(id);
}, []);

// ✅ Add count to deps (creates new interval per change)
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(id);
}, [count]);  // re-creates interval on each change (suboptimal)
```

**`useEffectEvent` (experimental, `unstable_*` prefix bilan):**

```tsx
import { experimental_useEffectEvent as useEffectEvent } from "react";

function Component({ url }: { url: string }) {
  const [data, setData] = useState(null);

  // Effect event — captures latest closure, not in deps
  const onConnected = useEffectEvent(() => {
    log(`Connected to ${url}`);
  });

  useEffect(() => {
    const conn = createConnection();
    conn.on("connected", onConnected);
    return () => conn.disconnect();
  }, []);  // url deps'da kerak emas (onConnected har gal latest url'ni ko'radi)
}
```

`useEffectEvent` — "stable function but always latest closure" muammosini hal qiladi. 2026 holicha **experimental** (`unstable_*` prefix bilan, stable emas, production'da ishlatish tavsiya etilmaydi).

**Deps array allocation:**

```tsx
useEffect(() => {}, [a, b, c]);
// Har render'da yangi `[a, b, c]` array yaratiladi
// `areHookInputsEqual` orqali prev deps bilan solishtiriladi
// Faqat oldingi deps ham saqlanadi (hook.memoizedState ichida)
```

**`useMemo` and `useCallback` deps work similarly:**

```tsx
const memoValue = useMemo(() => compute(a), [a]);
const memoCallback = useCallback(() => action(b), [b]);
```

Same `Object.is` comparison.

</details>

### Edge Cases

- **Deps array length change**: Warning. Deps must be same length across renders.
- **NaN in deps**: Object.is(NaN, NaN) = true → no re-run.
- **Boolean toggling**: `[true]` vs `[false]` — different, re-run.

### Follow-up savollar

- "exhaustive-deps disable kerakmi?" — Rare. Disable comments — code smell.
- "Empty deps for one-time?" — Rarely correct. Synchronization model — most effects have real deps.
- "Class shouldComponentUpdate analog?" — `useEffect` doesn't have. Use `useMemo`/memo for skip render. Effect is post-commit.

</details>

---

### 15. Cleanup function — qachon va nima uchun [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Cleanup function** — `useEffect` callback'idan return qilingan funksiya. Chaqiriladi: (1) **keyingi effect run'idan oldin** (yangi cycle) — eski state cleanup qilish, (2) **komponent unmount paytida** — resource release. Maqsad: subscription cancel, timer clear, listener detach, abort fetch. Sinxronizatsiya invariant: effect "qaytadan o'rnatilishi"ga chidamli bo'lishi kerak.

### To'liq tushuntirish

**Cleanup chaqirilish vaqti:**

```
Mount:
  effect()  → returns cleanup
  ...
Update (deps changed):
  cleanup()  ← OLD effect's cleanup
  effect()   ← NEW effect runs
  ...
Unmount:
  cleanup()  ← FINAL cleanup
```

**Use cases:**

1. **Subscriptions** — unsubscribe
2. **Timers** — clearTimeout/clearInterval
3. **Event listeners** — removeEventListener
4. **Network** — AbortController.abort()
5. **DOM mutations** — undo (rare)

### Kod misoli

```tsx
// 1. Subscription
function ChatRoom({ userId }: { userId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);

  useEffect(() => {
    const subscription = chatAPI.subscribe(userId, (msg) => {
      setMessages(prev => [...prev, msg]);
    });

    return () => {
      subscription.unsubscribe();  // cleanup
    };
  }, [userId]);

  // userId changes:
  // 1. Cleanup old subscription
  // 2. Subscribe new
}

// 2. Timer
function Countdown({ seconds }: { seconds: number }) {
  const [remaining, setRemaining] = useState(seconds);

  useEffect(() => {
    const id = setInterval(() => {
      setRemaining(prev => Math.max(0, prev - 1));
    }, 1000);

    return () => clearInterval(id);  // cleanup
  }, []);

  return <p>{remaining}s</p>;
}

// 3. Event listener
function WindowSize() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handler = () => setWidth(window.innerWidth);
    window.addEventListener("resize", handler);

    return () => window.removeEventListener("resize", handler);
  }, []);

  return <p>{width}px</p>;
}

// 4. Network — AbortController
function UserProfile({ id }: { id: string }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/users/${id}`, { signal: controller.signal })
      .then(r => r.json())
      .then(setUser)
      .catch(err => {
        if (err.name !== "AbortError") throw err;
      });

    return () => controller.abort();
  }, [id]);

  return <div>{user?.name}</div>;
}

// 5. DOM mutation cleanup
function PageTitle({ title }: { title: string }) {
  useEffect(() => {
    const previousTitle = document.title;
    document.title = title;

    return () => {
      document.title = previousTitle;  // restore
    };
  }, [title]);
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal: effect linked list:**

```typescript
type Effect = {
  tag: HookFlag;
  create: () => void | (() => void);
  destroy: (() => void) | undefined;
  deps: any[] | null;
  next: Effect;
};

// useEffect creates Effect:
function pushEffect(tag, create, destroy, deps): Effect {
  const effect: Effect = { tag, create, destroy, deps, next: null };

  // Add to fiber's effect list
  let componentUpdateQueue = currentlyRenderingFiber.updateQueue;
  if (componentUpdateQueue === null) {
    componentUpdateQueue = { lastEffect: null };
    currentlyRenderingFiber.updateQueue = componentUpdateQueue;
  }

  if (componentUpdateQueue.lastEffect === null) {
    effect.next = effect;  // self-loop (first)
  } else {
    effect.next = componentUpdateQueue.lastEffect.next;
    componentUpdateQueue.lastEffect.next = effect;
  }
  componentUpdateQueue.lastEffect = effect;

  return effect;
}
```

**Commit phase — effect execution:**

```typescript
// After commit phase (passive)
function flushPassiveEffects() {
  // 1. Run cleanups for OLD effects (alternate fiber)
  commitPassiveUnmountEffects(root);

  // 2. Run NEW effects
  commitPassiveMountEffects(root);
}

function commitPassiveUnmountEffects(fiber: Fiber) {
  // Walk fiber tree, find passive effects
  // For each effect with destroy: call destroy()
  if (fiber.flags & Passive) {
    const updateQueue = fiber.updateQueue;
    if (updateQueue !== null) {
      let effect = updateQueue.lastEffect.next;
      do {
        if (effect.destroy !== undefined) {
          effect.destroy();
        }
        effect = effect.next;
      } while (effect !== updateQueue.lastEffect.next);
    }
  }

  // Recurse children
  // ...
}

function commitPassiveMountEffects(fiber: Fiber) {
  if (fiber.flags & Passive) {
    const updateQueue = fiber.updateQueue;
    if (updateQueue !== null) {
      let effect = updateQueue.lastEffect.next;
      do {
        const create = effect.create;
        const destroy = create();  // run effect, save destroy fn
        effect.destroy = typeof destroy === "function" ? destroy : undefined;
        effect = effect.next;
      } while (effect !== updateQueue.lastEffect.next);
    }
  }

  // Recurse children
  // ...
}
```

**Order: cleanup all → effects all:**

```tsx
function Parent() {
  useEffect(() => {
    console.log("parent setup");
    return () => console.log("parent cleanup");
  }, [trigger]);

  return <Child />;
}

function Child() {
  useEffect(() => {
    console.log("child setup");
    return () => console.log("child cleanup");
  }, [trigger]);
}

// Trigger change → all cleanups → all effects
// Output:
// child cleanup     (child first — bottom-up cleanup)
// parent cleanup
// child setup       (child first — bottom-up effect)
// parent setup
```

Bottom-up — children cleanup before parents (child's cleanup may depend on parent context).

**Cleanup with stale closure:**

```tsx
function Component({ id }: { id: string }) {
  useEffect(() => {
    console.log("setup:", id);
    return () => console.log("cleanup:", id);  // uses closure id
  }, [id]);
}

// id="1":
//   setup: 1

// id="2":
//   cleanup: 1  ← old closure (id was 1)
//   setup: 2    ← new closure (id is 2)
```

Cleanup closure captures values at effect time, not call time.

**StrictMode 2x:**

```tsx
useEffect(() => {
  console.log("setup");
  return () => console.log("cleanup");
}, []);

// Strict Mode dev:
// "setup"
// "cleanup"  ← simulated unmount
// "setup"    ← simulated remount
```

Tests cleanup correctness. Effect must survive cleanup-setup cycle (resource not leaked, no race).

**Common cleanup mistakes:**

```tsx
// ❌ Cleanup yo'q — leak
useEffect(() => {
  const id = setInterval(callback, 1000);
}, []);
// Component unmounts but interval still runs (memory leak, callback called on unmounted)

// ❌ Cleanup return false — lost
useEffect(() => {
  setInterval(callback, 1000);
  return false;  // ❌ not function — ignored
}, []);

// ❌ Async cleanup — return value not handled
useEffect(() => {
  const promise = fetch("/api");
  return async () => {
    await promise;  // ❌ promise return — React waits for what?
  };
}, []);
// React expects sync function return. Async wrapping unclear behavior.

// ✅ Cleanup pattern
useEffect(() => {
  let cancelled = false;
  fetch("/api").then(data => {
    if (!cancelled) setData(data);
  });
  return () => { cancelled = true; };
}, []);
```

**Race condition prevention:**

```tsx
function Search({ query }: { query: string }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    let cancelled = false;
    fetch(`/search?q=${query}`).then(r => r.json()).then(data => {
      if (!cancelled) setResults(data);  // only if still relevant
    });
    return () => { cancelled = true; };
  }, [query]);
}

// Fast typing: "a" → "ab" → "abc"
// Effect 1 (query="a") starts fetch
// Effect 2 (query="ab") cleanup ("a" cancelled) + new fetch
// Effect 3 (query="abc") cleanup ("ab" cancelled) + new fetch
// Only "abc" results applied (others cancelled)
```

**`AbortController` — modern fetch cancel:**

```tsx
useEffect(() => {
  const controller = new AbortController();
  fetch(url, { signal: controller.signal })
    .then(r => r.json())
    .then(setData)
    .catch(err => {
      if (err.name === "AbortError") return;  // expected
      console.error(err);
    });
  return () => controller.abort();
}, [url]);
```

`AbortController` — actually cancels network request (saves bandwidth).

**Multiple effects, multiple cleanups:**

```tsx
useEffect(() => {
  const id1 = setInterval(tick, 1000);
  return () => clearInterval(id1);
}, []);

useEffect(() => {
  const id2 = setTimeout(timeout, 5000);
  return () => clearTimeout(id2);
}, []);

// Each useEffect has own cleanup, runs independently
```

**Cleanup for derived values (rare):**

```tsx
useEffect(() => {
  const previousTitle = document.title;
  document.title = newTitle;
  return () => {
    document.title = previousTitle;  // restore
  };
}, [newTitle]);
```

**No cleanup needed:**

```tsx
// Pure computation — no cleanup
useEffect(() => {
  console.log("rendered with count:", count);
}, [count]);

// State update without subscription — no cleanup
useEffect(() => {
  if (count > 10) setOverflow(true);
}, [count]);
```

</details>

### Edge Cases

- **Cleanup throws**: React still continues. Error not caught by error boundary (post-commit).
- **Cleanup with `setState`**: Allowed — state queued for next render.
- **Conditional cleanup**: Return `() => {}` if condition met, otherwise return undefined? Better: always return function.

### Follow-up savollar

- "Cleanup runs even on unmount?" — Yes. Any "destroy" call (deps change or unmount).
- "Order: parent cleanup vs child cleanup?" — Bottom-up: child cleanup first.
- "Cleanup async OK?" — Function should be sync. Inside, fire-and-forget async OK.

</details>

---

### 16. Race conditions va `AbortController` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Race condition** — bir nechta async operatsiyalar parallel ishlaganda, eski natija yangi'dan **keyin kelishi** mumkin → UI yangi state'ga eskirgan ma'lumot ko'rsatadi. **Yechim**: cleanup function bilan **cancel flag** (`let cancelled = true`) yoki **AbortController** (network requests uchun). Modern fetch API'da `signal: controller.signal` — actual network cancellation.

### To'liq tushuntirish

**Race condition senariy:**

```tsx
function User({ id }: { id: string }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    fetch(`/api/users/${id}`)
      .then(r => r.json())
      .then(setUser);
  }, [id]);

  return <div>{user?.name}</div>;
}

// Timeline:
// T=0: id="1" → fetch user 1 (slow, 1000ms)
// T=100: id="2" → fetch user 2 (fast, 200ms)
// T=300: user 2 response → setUser(user2) ✓
// T=1000: user 1 response → setUser(user1) ❌ (overwrites user 2!)
```

### Kod misoli

**Yechim 1: Cancel flag (cancelled boolean):**

```tsx
function User({ id }: { id: string }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    let cancelled = false;

    fetch(`/api/users/${id}`)
      .then(r => r.json())
      .then(data => {
        if (!cancelled) {
          setUser(data);
        }
      });

    return () => { cancelled = true; };
  }, [id]);

  return <div>{user?.name}</div>;
}

// id changes:
// 1. Old effect cleanup: cancelled = true
// 2. New effect: cancelled = false (new closure)
// 3. Old fetch resolves: cancelled true → setUser skipped ✓
// 4. New fetch resolves: cancelled false → setUser ✓
```

**Yechim 2: AbortController (cancels network):**

```tsx
function User({ id }: { id: string }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/users/${id}`, { signal: controller.signal })
      .then(r => r.json())
      .then(setUser)
      .catch(err => {
        if (err.name === "AbortError") return;  // expected on abort
        throw err;
      });

    return () => controller.abort();
  }, [id]);

  return <div>{user?.name}</div>;
}

// id changes:
// 1. Old effect cleanup: controller.abort() — fetch cancelled (network level)
// 2. fetch promise rejects with AbortError
// 3. .catch handles AbortError silently
// 4. New fetch starts
```

**Yechim 3: useId / latest pattern:**

```tsx
function User({ id }: { id: string }) {
  const [user, setUser] = useState<User | null>(null);
  const latestIdRef = useRef(id);

  useEffect(() => {
    latestIdRef.current = id;
  }, [id]);

  useEffect(() => {
    fetch(`/api/users/${id}`)
      .then(r => r.json())
      .then(data => {
        if (latestIdRef.current === id) {
          setUser(data);
        }
      });
  }, [id]);
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why race conditions happen:**

JavaScript single-threaded but async. Multiple promises can be in-flight concurrently. Resolution order ≠ initiation order.

```
T=0: fetch("user1") promise pending (1000ms latency)
T=100: fetch("user2") promise pending (200ms latency)
T=300: user2 promise resolves → callback runs
T=1000: user1 promise resolves → callback runs
```

Without cancellation, both callbacks execute → "out of order" updates.

**Cancel flag mechanism:**

```typescript
useEffect(() => {
  let cancelled = false;  // closure variable

  asyncOperation().then(result => {
    if (!cancelled) {
      setData(result);
    }
  });

  return () => { cancelled = true; };
}, [deps]);
```

- Closure captures `cancelled` reference
- Effect runs → `cancelled = false` (default)
- Cleanup → sets `cancelled = true`
- Async resolves → checks closure variable
- Each effect cycle has own closure with own `cancelled`

**`AbortController` — better than flag for fetch:**

```tsx
const controller = new AbortController();
fetch(url, { signal: controller.signal });
// Network request bilan to'g'ridan-to'g'ri bog'liq

controller.abort();
// Aborts:
// - Pending request cancelled (network level)
// - Promise rejects with AbortError
// - Saves bandwidth
```

**`AbortController` doesn't cancel state:**

```tsx
useEffect(() => {
  const controller = new AbortController();

  fetch(url, { signal: controller.signal }).then(r => r.json()).then(data => {
    // Even with controller.abort(), if response already received,
    // .then runs! AbortController prevents pending request, not resolved.
    setData(data);
  });

  return () => controller.abort();
}, [url]);

// If request resolves before cleanup → data set
// If request pending → aborted, .then doesn't run (rejected)
```

For full safety: combine `AbortController` + `cancelled` flag:

```tsx
useEffect(() => {
  let cancelled = false;
  const controller = new AbortController();

  fetch(url, { signal: controller.signal })
    .then(r => r.json())
    .then(data => {
      if (!cancelled) setData(data);
    })
    .catch(err => {
      if (err.name === "AbortError") return;
      throw err;
    });

  return () => {
    cancelled = true;
    controller.abort();
  };
}, [url]);
```

**Strict Mode 2x effect — race aware:**

```tsx
useEffect(() => {
  let cancelled = false;
  fetch(url).then(data => {
    if (!cancelled) setData(data);
  });
  return () => { cancelled = true; };
}, [url]);

// Strict Mode dev:
// Mount 1: cancelled=false, fetch1 starts
// Cleanup 1: cancelled1=true
// Mount 2: cancelled2=false, fetch2 starts
// fetch1 resolves: cancelled1 true → skip
// fetch2 resolves: cancelled2 false → setData ✓
```

Strict Mode 2x exposes race conditions immediately during dev.

**Concurrent rendering + race:**

```tsx
// React may interrupt and re-render
// Effects only run after commit (consistent state)
// But still: async operations from prior renders pending
```

Cancellation invariant — must work across all renders.

**Reusable hook pattern:**

```tsx
function useFetch<T>(url: string | null): { data: T | null; loading: boolean; error: Error | null } {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    if (!url) return;

    let cancelled = false;
    const controller = new AbortController();

    setLoading(true);
    setError(null);

    fetch(url, { signal: controller.signal })
      .then(r => {
        if (!r.ok) throw new Error(r.statusText);
        return r.json();
      })
      .then(d => {
        if (!cancelled) {
          setData(d);
          setLoading(false);
        }
      })
      .catch(err => {
        if (cancelled || err.name === "AbortError") return;
        setError(err);
        setLoading(false);
      });

    return () => {
      cancelled = true;
      controller.abort();
    };
  }, [url]);

  return { data, loading, error };
}

// Usage:
function UserProfile({ id }: { id: string }) {
  const { data: user, loading, error } = useFetch<User>(`/api/users/${id}`);

  if (loading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  return <div>{user?.name}</div>;
}
```

**Modern alternatives — TanStack Query, SWR:**

Production apps usually use:
- **TanStack Query** (`@tanstack/react-query`) — caching, deduplication, race-safe
- **SWR** — similar
- These libraries handle race conditions, caching, retries automatically

Vanilla `useEffect` for simple cases or learning.

**Race condition tests:**

```tsx
test("handles race condition", async () => {
  const { rerender } = render(<User id="1" />);

  // Simulate slow id=1 response
  mockFetch("/api/users/1", { delay: 1000, response: { name: "User 1" } });

  // Quick id change
  rerender(<User id="2" />);
  mockFetch("/api/users/2", { delay: 100, response: { name: "User 2" } });

  // Wait
  await screen.findByText("User 2");

  // Wait for slow response (which should be ignored)
  await new Promise(r => setTimeout(r, 1500));

  // User 2 still shown (not overwritten by User 1)
  expect(screen.getByText("User 2")).toBeInTheDocument();
});
```

**Common race scenarios:**

1. Search/filter typing fast
2. Tab switching
3. Pagination clicks
4. Filter changes
5. Authentication state changes (logout during fetch)

**Performance:**

- AbortController: native, fast, ~0 overhead
- cancelled flag: closure variable, ~0 overhead
- Both safe for production

</details>

### Edge Cases

- **AbortError handling**: `.catch` must distinguish AbortError from real errors.
- **Stream-based responses**: AbortController cancels stream too.
- **Workers/WebSocket**: Need explicit close() in cleanup.

### Follow-up savollar

- "TanStack Query handles race?" — Yes, deduplication + abort pattern internally. Recommended for production.
- "Promise.all race?" — Yes, similar issues. Handle in same way.
- "useReducer for async state?" — Reducer pattern + dispatch — safer than multiple useState. Race still possible without cancellation.

</details>

---

### 17. Strict Mode 2x effect (R18+) — invariant tekshiruv [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Strict Mode 2x effect** (R18+) — development paytida har effect **mount → cleanup → mount** cycle bilan ishlaydi. Maqsad: **idempotency tekshiruv** — effect cleanup-resilient bo'lishi shart (resource leak yo'q, race condition yo'q). Production'da yo'q (faqat dev). Concurrent rendering invariantlarini majburlash: mount-unmount cycle har vaqt sodir bo'lishi mumkin.

### To'liq tushuntirish

**Behavior:**

```tsx
useEffect(() => {
  console.log("setup");
  return () => console.log("cleanup");
}, []);

// Strict Mode dev:
// "setup"     ← initial mount
// "cleanup"   ← simulated unmount
// "setup"     ← simulated remount

// Production:
// "setup"     ← only mount, no synthetic cycle
```

**Why:** Concurrent rendering may unmount + remount components (offscreen, suspended, etc.). Effects must handle this gracefully.

### Kod misoli

```tsx
// ❌ Anti-pattern: side effect without cleanup
function Logger() {
  useEffect(() => {
    fetch("/log", { method: "POST", body: "mounted" });
  }, []);
}

// Strict Mode 2x:
// fetch 1 (mount)
// (no cleanup)
// fetch 2 (remount) → DUPLICATE log entry
// Server side: 2 logs for 1 mount

// ✅ Correct — cleanup handles
function Logger() {
  useEffect(() => {
    const controller = new AbortController();
    fetch("/log", { method: "POST", body: "mounted", signal: controller.signal });
    return () => controller.abort();  // cancel
  }, []);
}

// Strict Mode 2x:
// fetch 1 starts → cleanup aborts → fetch 1 cancelled
// fetch 2 starts (mount) → completes
// Server side: 1 log entry ✓
```

```tsx
// ❌ Subscription without cleanup
function ChatRoom() {
  useEffect(() => {
    socket.connect();
  }, []);
  // No disconnect!
}

// Strict Mode 2x:
// connect 1
// (no disconnect)
// connect 2 → 2 connections!

// ✅ Cleanup
useEffect(() => {
  socket.connect();
  return () => socket.disconnect();
}, []);

// Strict Mode 2x:
// connect 1
// disconnect 1
// connect 2 → 1 active connection
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why concurrent rendering needs this:**

R18+ Concurrent rendering features:
- Offscreen/Activity (R19) — render hidden tabs
- Suspense fallback — content unmount-remount
- Future: state preservation across navigation

These all require: effects survive mount-unmount cycles.

**Strict Mode dev cycle:**

```typescript
// Internal (sodda)
function commitMount(fiber) {
  if (__DEV__ && isStrictModeFiber(fiber)) {
    runMountEffects(fiber);    // first mount
    runCleanupEffects(fiber);   // synthetic unmount
    runMountEffects(fiber);    // remount
  } else {
    runMountEffects(fiber);
  }
}
```

`useState` initializer also runs 2x in Strict Mode (must be pure):

```tsx
const [data] = useState(() => {
  console.log("init");  // logs 2x in dev
  return computeInitial();
});
```

**Common bugs Strict Mode catches:**

```tsx
// 1. Subscription without cleanup
useEffect(() => {
  store.subscribe(handler);
}, []);
// 2x mount → 2 listeners → memory leak + duplicate calls

// 2. Event listeners without removal
useEffect(() => {
  document.addEventListener("click", handler);
}, []);
// 2 listeners

// 3. Side effect in effect (e.g., logging mount)
useEffect(() => {
  analytics.track("page_view");
}, []);
// 2 page_view events!

// Fix for 3: idempotency check
useEffect(() => {
  if (!hasTracked.current) {
    analytics.track("page_view");
    hasTracked.current = true;
  }
}, []);
// Still bad — should rely on cleanup
```

**Counter-example: Strict Mode legitimate flag:**

```tsx
function PaymentForm() {
  const [submitting, setSubmitting] = useState(false);

  useEffect(() => {
    if (!submitting) return;

    let cancelled = false;
    submitPayment(data).then(() => {
      if (!cancelled) setSubmitting(false);
    });

    return () => { cancelled = true; };
  }, [submitting, data]);
}

// Strict Mode 2x mount with submitting=true:
// Effect 1: submitPayment starts → cleanup cancelled → no commit
// Effect 2: submitPayment again? — Yes, but cancelled

// Issue: 2 payment submissions sent to server!
// Even though state ignored both, server may charge 2x.
```

**Fix: idempotency or single-fire pattern:**

```tsx
// Idempotency key
useEffect(() => {
  if (!submitting) return;

  const idempotencyKey = uuid();
  fetch("/payment", {
    method: "POST",
    headers: { "Idempotency-Key": idempotencyKey },
    body: JSON.stringify(data),
  });
}, [submitting, data]);

// Server handles idempotency key — duplicate requests ignored
```

**`useEffect` pure intent:**

```tsx
// Effect should "describe synchronization":
// "While submitting=true, send payment data"

// If state submitting=true → mount cycle should re-attempt
// Server responsibility: idempotency

// Wrong mental model:
// "When click happens, send payment"
// → use event handler, not effect
```

**StrictMode wrapper:**

```tsx
import { StrictMode } from "react";

const root = createRoot(container);
root.render(
  <StrictMode>
    <App />
  </StrictMode>
);

// Top-level — entire app
// Or sub-tree
function App() {
  return (
    <>
      <Header />
      <StrictMode>
        <NewFeature />  {/* only this in strict mode */}
      </StrictMode>
    </>
  );
}
```

**`useState`/`useReducer` initializer 2x:**

```tsx
function Counter() {
  const [count] = useState(() => {
    console.log("init");
    return 0;
  });
}

// Strict Mode dev:
// "init"
// "init"  ← 2x
```

Initializer must be pure (no side effects).

**`useMemo` 2x:**

```tsx
const expensive = useMemo(() => {
  console.log("compute");
  return computeHeavy();
}, [deps]);

// Strict Mode dev:
// "compute"
// "compute"  ← 2x
```

useMemo callback should be pure.

**`useEffect` 2x — full cycle:**

```tsx
useEffect(() => {
  console.log("E");
  return () => console.log("C");
}, []);

// Strict Mode dev:
// E  (initial)
// C  (synthetic cleanup)
// E  (remount)
// (real unmount when applicable: C)

// Production:
// E  (mount)
// (later: C on real unmount)
```

**`useLayoutEffect` 2x — same:**

```tsx
useLayoutEffect(() => {
  console.log("L");
  return () => console.log("L-C");
}, []);

// Strict Mode dev:
// L
// L-C
// L
```

**Disable Strict Mode 2x — possible?**

No flag. Strict Mode 2x is intentional dev tool. Disabling masks bugs.

If specific test/setup breaks — fix the test to be idempotent.

**Class component 2x:**

R18 also makes class lifecycle 2x in Strict Mode:
- `componentDidMount` → `componentWillUnmount` → `componentDidMount`
- Catches old class lifecycle bugs same way

**Performance impact (production):**

Zero. Strict Mode 2x is dev-only. Production runs effects once.

**Production bug if Strict Mode disabled:**

Bugs that Strict Mode would catch can still manifest in production with concurrent features:
- Suspense fallback → effect cleanup → no resync after promise resolves
- Offscreen → mount-hidden cycle
- Future React features

So even though dev 2x is "fake", real concurrent scenarios trigger same patterns.

</details>

### Edge Cases

- **`useState` initializer with side effect**: 2x in dev exposes. Refactor to pure.
- **`useEffect` with no cleanup**: Many use cases need none. Document.title set is OK if value matches (idempotent).
- **`useRef` initializer**: Not 2x — refs don't have lazy initializer.

### Follow-up savollar

- "Strict Mode break my code — disable?" — No. Fix the code. Strict Mode reveals real bugs.
- "Production with concurrent features — same bugs?" — Yes. Strict Mode dev simulates concurrent scenarios.
- "Library handles Strict Mode?" — Modern libraries (TanStack Query, Zustand) tested with Strict Mode. Older libraries may have issues.

</details>

---

### 18. "You Might Not Need an Effect" — anti-patterns [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Effect anti-patterns** (Dan Abramov post): (1) **Derived state** — useEffect setState dan kelib chiqqan boshqa setState — render time'da hisoblash kerak, (2) **Caching computation** — useMemo ishlatish, (3) **Event response** — event handler'da, effectda emas, (4) **Initial state** — useState initializer, (5) **Resetting state on prop change** — `key` prop bilan, (6) **Notify parent on state change** — event handler'da uzatish. Effect faqat **external system synchronization** uchun.

### To'liq tushuntirish

**Antipattern 1: Derived state**

```tsx
// ❌ Effect for derived state
function Profile({ user }: { user: User }) {
  const [fullName, setFullName] = useState("");

  useEffect(() => {
    setFullName(`${user.firstName} ${user.lastName}`);
  }, [user]);

  return <div>{fullName}</div>;
}
// Issues: extra render, extra state, lag

// ✅ Compute during render
function Profile({ user }: { user: User }) {
  const fullName = `${user.firstName} ${user.lastName}`;
  return <div>{fullName}</div>;
}
```

**Antipattern 2: Caching computation**

```tsx
// ❌ Effect for cache
function List({ items, query }: Props) {
  const [filtered, setFiltered] = useState<Item[]>([]);

  useEffect(() => {
    setFiltered(items.filter(i => i.name.includes(query)));
  }, [items, query]);

  return <ul>{filtered.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}

// ✅ Compute during render (or useMemo for expensive)
function List({ items, query }: Props) {
  const filtered = items.filter(i => i.name.includes(query));
  // Or with memoization:
  // const filtered = useMemo(() => items.filter(...), [items, query]);
  return <ul>{filtered.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}
```

**Antipattern 3: Event response**

```tsx
// ❌ Effect for event response
function ProductPage({ product }: { product: Product }) {
  const [cart, setCart] = useState<Cart>({ items: [] });

  useEffect(() => {
    if (cart.items.length > 0) {
      sendAnalytics("cart_added", cart);  // ❌
    }
  }, [cart]);

  const handleAddToCart = () => {
    setCart(prev => ({ ...prev, items: [...prev.items, product] }));
  };
}

// ✅ Event handler
function ProductPage({ product }: { product: Product }) {
  const [cart, setCart] = useState<Cart>({ items: [] });

  const handleAddToCart = () => {
    const newCart = { ...cart, items: [...cart.items, product] };
    setCart(newCart);
    sendAnalytics("cart_added", newCart);  // ✅ event
  };
}
```

**Antipattern 4: Resetting state on prop change**

```tsx
// ❌ Effect for reset
function Form({ userId }: { userId: string }) {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");

  useEffect(() => {
    setName("");
    setEmail("");
  }, [userId]);  // reset on userId change
}

// ✅ Key prop (full remount)
function FormWrapper({ userId }: { userId: string }) {
  return <Form key={userId} />;
  // userId change → Form remounts, fresh state
}
```

**Antipattern 5: Notify parent**

```tsx
// ❌ Effect to notify
function Toggle({ onChange }: { onChange: (on: boolean) => void }) {
  const [on, setOn] = useState(false);

  useEffect(() => {
    onChange(on);  // ❌ extra render
  }, [on, onChange]);
}

// ✅ Event handler
function Toggle({ onChange }: { onChange: (on: boolean) => void }) {
  const [on, setOn] = useState(false);

  const handleToggle = () => {
    const newValue = !on;
    setOn(newValue);
    onChange(newValue);  // ✅ in event
  };
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Source: react.dev/learn/you-might-not-need-an-effect**

Dan Abramov / React team published this guide. Key insight: many effect uses are anti-patterns; correct approach often event handler or render-time computation.

**Antipattern 6: Fetching data:**

```tsx
// ❌ useEffect for data
useEffect(() => {
  fetchUser(id).then(setUser);
}, [id]);

// ⚠️ Vanilla useEffect — race conditions, no cache, no retry
// ✅ Library — TanStack Query, SWR
const { data: user } = useQuery({ queryKey: ["user", id], queryFn: () => fetchUser(id) });

// ✅ Or framework — Next.js, Remix loaders
// Server-side data fetching, no useEffect needed
```

**Antipattern 7: Application initialization:**

```tsx
// ❌ Component-level init
function App() {
  useEffect(() => {
    initApp();  // runs on every mount (Strict Mode 2x: twice!)
  }, []);
}

// ✅ Module-level
let initialized = false;
if (!initialized) {
  initApp();
  initialized = true;
}

function App() {
  return <Main />;
}

// Or top-level await (RSC), or app entry point
```

**Antipattern 8: Storing previous prop:**

```tsx
// ❌ Effect to track prev
function Component({ value }: { value: number }) {
  const [prev, setPrev] = useState(value);

  useEffect(() => {
    setPrev(value);
  }, [value]);

  // Use prev for comparison...
}

// ✅ Compute during render (Dan's adjusted state pattern)
function Component({ value }: { value: number }) {
  const [prevValue, setPrevValue] = useState(value);
  if (value !== prevValue) {
    setPrevValue(value);  // setState during render — OK if conditional
    // React handles: setState during render = re-render with new state
  }
}

// Or custom hook
function usePrevious<T>(value: T): T | undefined {
  // R19: zero-arg `useRef<T>()` olib tashlandi — initial value MAJBURIY
  const ref = useRef<T | undefined>(undefined);
  useEffect(() => { ref.current = value; });
  return ref.current;
}
```

**Antipattern 9: Synchronizing two state variables:**

```tsx
// ❌ Effect to sync
function App() {
  const [items, setItems] = useState<Item[]>([]);
  const [count, setCount] = useState(0);

  useEffect(() => {
    setCount(items.length);  // sync count with items.length
  }, [items]);
}

// ✅ Derive
function App() {
  const [items, setItems] = useState<Item[]>([]);
  const count = items.length;  // computed
}
```

**Antipattern 10: Calling parent callback for state change:**

```tsx
// ❌ Effect for parent notification
function Toggle({ initial, onChange }: { initial: boolean; onChange: (v: boolean) => void }) {
  const [on, setOn] = useState(initial);

  useEffect(() => {
    onChange(on);
  }, [on]);
}

// ✅ Lift state up — parent owns state
function Toggle({ on, onChange }: { on: boolean; onChange: (v: boolean) => void }) {
  return <button onClick={() => onChange(!on)}>{on ? "On" : "Off"}</button>;
}
// Parent: const [on, setOn] = useState(false); <Toggle on={on} onChange={setOn} />
```

**Antipattern 11: Initializing with prop:**

```tsx
// ❌ useEffect to init
function Form({ initialUser }: { initialUser: User }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    setUser(initialUser);
  }, []);
}

// ✅ useState initializer
function Form({ initialUser }: { initialUser: User }) {
  const [user, setUser] = useState(initialUser);
}
```

**Decision tree: do I need an effect?**

```
Question: "What is this effect doing?"

A. Computing value from state/props
   → No, compute during render (or useMemo)

B. Reacting to event
   → No, event handler

C. Initializing state
   → No, useState initializer

D. Resetting state when prop changes
   → No, key prop

E. Notifying parent
   → No, event handler + lifted state

F. Synchronizing with external system
   → YES, useEffect (subscriptions, DOM, network)

G. Updating document.title, browser API
   → YES, useEffect

H. Setting up timers, intervals
   → YES, useEffect with cleanup
```

**Effect "external system" criteria:**

- DOM mutation (`document.title`, focus, scroll)
- Browser API (`window`, `localStorage`, `fetch`)
- 3rd-party library (subscriptions, animations)
- WebSocket, IndexedDB, Worker

If you're staying entirely within React state — usually no effect needed.

**`useEffect` vs `useLayoutEffect` vs `useInsertionEffect`:**

- `useEffect` — async, post-paint (default)
- `useLayoutEffect` — sync, pre-paint (DOM measurement)
- `useInsertionEffect` — pre-layout (CSS-in-JS injection)

All same anti-patterns apply.

**Performance impact of useless effects:**

```tsx
// ❌ Anti-pattern
function Profile({ user }) {
  const [fullName, setFullName] = useState("");

  useEffect(() => {
    setFullName(`${user.firstName} ${user.lastName}`);
  }, [user]);

  return <div>{fullName}</div>;
}

// Render flow:
// 1. Initial render: fullName = "" (UI shows empty briefly)
// 2. Commit
// 3. useEffect runs: setFullName(...)
// 4. Re-render with fullName
// 5. Commit
// → 2 commits, brief flash of empty UI

// ✅ Direct computation
function Profile({ user }) {
  const fullName = `${user.firstName} ${user.lastName}`;
  return <div>{fullName}</div>;
}
// → 1 render, 1 commit, no flash
```

**`useEffect` vs `useSyncExternalStore`:**

For external store subscriptions (Redux, Zustand):

```tsx
// ❌ useEffect + useState — tearing risk
function Counter() {
  const [count, setCount] = useState(store.getCount());
  useEffect(() => {
    return store.subscribe(() => setCount(store.getCount()));
  }, []);
}

// ✅ useSyncExternalStore — no tearing
function Counter() {
  const count = useSyncExternalStore(
    store.subscribe,
    store.getCount
  );
}
```

</details>

### Edge Cases

- **Effect for cross-cutting concern (logging)**: Sometimes OK (e.g., page view tracking on mount). Idempotency key for safety.
- **Effect for reset on multiple deps**: Key prop or explicit reset in event.
- **`useEffect` with empty deps for "init"**: Often anti-pattern. Strict Mode 2x catches.

### Follow-up savollar

- "All effects bad?" — No. External sync is correct use. Most others — anti-patterns.
- "ESLint catches?" — Some patterns. Manual review needed for full coverage.
- "React Compiler helps?" — Yes — auto-memoization avoids many "I need effect for cache" cases.

</details>

---

### 19. Effect dep array — object/array reference traps [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useEffect` deps `Object.is` comparison ishlatadi. Object/array literals (`{a:1}`, `[1,2]`) — har render yangi reference → effect har render qayta ishlaydi (loop). Yechim: `useMemo` bilan stabilize, primitive deps ishlatish (`obj.id`), yoki `useEffectEvent` (RFC).

### Kod misoli

```tsx
// ❌ Object reference trap
function BadComponent({ userId }: { userId: string }) {
  const filters = { active: true, type: "premium" };  // new reference each render
  useEffect(() => {
    fetchUsers(userId, filters);
  }, [userId, filters]);  // ❌ filters new each render — effect loops
}

// ✅ useMemo stabilize
function GoodComponent({ userId }: { userId: string }) {
  const filters = useMemo(
    () => ({ active: true, type: "premium" }),
    []
  );
  useEffect(() => {
    fetchUsers(userId, filters);
  }, [userId, filters]);
}

// ✅ Primitive deps
function BetterComponent({ userId }: { userId: string }) {
  useEffect(() => {
    const filters = { active: true, type: "premium" };
    fetchUsers(userId, filters);
  }, [userId]);  // primitive only
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Object.is comparison:**

```typescript
// React compares deps via Object.is
function depsEqual(prev, next) {
  for (let i = 0; i < prev.length; i++) {
    if (!Object.is(prev[i], next[i])) return false;
  }
  return true;
}

// Object reference: { a: 1 } !== { a: 1 } (different objects)
```

**Common patterns:**

```tsx
// 1. Inline array
useEffect(() => {}, [tags]);  // ⚠️ if tags is array prop

// 2. Inline function
useEffect(() => callback(), [callback]);  // ⚠️ if callback is inline arrow

// 3. Computed object
useEffect(() => {}, [{ value }]);  // ❌ always new
```

**Fix patterns:**

```tsx
// JSON stringify (works, but slow for deep)
useEffect(() => {}, [JSON.stringify(filters)]);

// Specific properties
useEffect(() => {}, [filters.active, filters.type]);

// useMemo
const filters = useMemo(() => ({ active: true }), []);

// useCallback for functions
const handler = useCallback(() => {}, []);
useEffect(() => handler(), [handler]);
```

**ESLint helps:**

`react-hooks/exhaustive-deps` — warns missing deps. Doesn't fix reference instability — that's developer's job.

</details>

### Edge Cases

- **Empty object literal**: `useEffect(() => {}, [{}])` — new object each render, infinite loop.
- **`useState` setter as dep**: Stable (React guarantees). No need to include.
- **`useRef` as dep**: Stable across renders. Including unnecessary.

### Follow-up savollar

- "useEffectEvent — solves this?" — Experimental API. Separates "latest closure" from "deps tracking".
- "React Compiler auto-stabilize?" — Yes, Compiler memoizes intermediate values.

</details>

---

<a id="qism-d"></a>

## QISM D: useLayoutEffect va useInsertionEffect

### 20. `useLayoutEffect` vs `useEffect` — timing farq [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useEffect`** — async, **paint'dan keyin** chaqiriladi (passive). Render bloklanmaydi. **`useLayoutEffect`** — sync, Commit Phase'ning **Layout sub-phase**'ida, **paint'dan oldin** chaqiriladi. DOM measurement va visual flicker'ni oldini olish uchun. Ikkalasi ham bir xil API (`useEffect`/`useLayoutEffect(setup, deps)`), faqat timing farq.

### To'liq tushuntirish

**Pipeline:**

```
Render Phase
  ↓
Commit Phase:
  ├─ useInsertionEffect (R18) — Mutation OLDIDAN, style injection uchun
  ├─ Mutation (DOM change — append/remove/update)
  ├─ Layout sub-phase:
  │   ├─ refs attach qilinadi
  │   ├─ useLayoutEffect ← SYNC, paint'dan oldin
  │   └─ useImperativeHandle
  ↓
Browser Paint
  ↓
useEffect ← ASYNC, paint'dan keyin (passive)
```

**Use case farqi:**

| Hook | Use case |
|------|----------|
| `useEffect` | API calls, subscriptions, non-visual side effects |
| `useLayoutEffect` | DOM measurement, scroll position, focus, prevent visual flicker |

### Kod misoli

```tsx
// useEffect — async, paint'dan keyin
function NotificationCount() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `(${count}) Notifications`;
  }, [count]);

  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
// Title update — paint'dan keyin OK (visual'ga ta'sir qilmaydi)

// useLayoutEffect — DOM measurement
function Tooltip({ targetRef }: { targetRef: React.RefObject<HTMLElement> }) {
  const [pos, setPos] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    const rect = targetRef.current?.getBoundingClientRect();
    if (rect) {
      setPos({ top: rect.bottom + 5, left: rect.left });
    }
  }, []);

  return <div style={{ position: "absolute", ...pos }}>Tooltip</div>;
}
// useLayoutEffect: measure → setPos → re-render → paint
// All before browser shows initial position
// User sees correct position immediately (no flicker)

// useEffect for same — visual flicker:
useEffect(() => {
  // ... (paint already happened, tooltip at default 0,0)
  setPos({ top: ..., left: ... });
  // re-render, repaint at correct position
});
// Flicker: tooltip flashes at 0,0 then moves
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal:**

```typescript
// useLayoutEffect — Layout flag (sync)
function mountLayoutEffect(create, deps) {
  const hookFlags = HookLayout | HookHasEffect;
  return mountEffectImpl(UpdateEffect, hookFlags, create, deps);
}

// useEffect — Passive flag (async)
function mountEffect(create, deps) {
  const hookFlags = HookPassive | HookHasEffect;
  return mountEffectImpl(PassiveEffect, hookFlags, create, deps);
}

// Commit phase
function commitLayoutEffects(root) {
  // Sync — runs in Layout sub-phase
  walkFiberTree(fiber => {
    if (fiber.flags & UpdateEffect) {
      runEffect(fiber);  // sync, blocks paint
    }
  });
}

function commitPassiveEffects() {
  // Async — runs after paint
  walkFiberTree(fiber => {
    if (fiber.flags & PassiveEffect) {
      runEffect(fiber);
    }
  });
}
```

**`useLayoutEffect` performance impact:**

```tsx
// ❌ Heavy computation in useLayoutEffect
useLayoutEffect(() => {
  const expensiveResult = heavyCompute();  // long sync work
  setData(expensiveResult);
});
// Browser paint sync ravishda bloklanadi — visible jank

// ✅ Move to useEffect if non-visual
useEffect(() => {
  const expensiveResult = heavyCompute();
  setData(expensiveResult);
});
```

**SSR + `useLayoutEffect`:**

```tsx
function Component() {
  useLayoutEffect(() => {
    // Server: no DOM, can't run
    // Warning: "useLayoutEffect does nothing on the server"
  }, []);
}
```

Workarounds:
1. Use `useEffect` instead (if not strictly needed)
2. `typeof window` guard
3. `useIsomorphicLayoutEffect` pattern:

```tsx
const useIsomorphicLayoutEffect =
  typeof window !== "undefined" ? useLayoutEffect : useEffect;
```

**`useLayoutEffect` setState:**

```tsx
useLayoutEffect(() => {
  const rect = ref.current?.getBoundingClientRect();
  if (rect && rect.right > window.innerWidth) {
    setPosition(window.innerWidth - rect.width);
  }
}, [content]);

// setState inside useLayoutEffect:
// - Schedules re-render synchronously
// - Render + commit happens before paint
// - Browser sees only final state
```

**Order of effects in commit:**

```
Commit Phase:
1. Before Mutation
2. Mutation (DOM updated)
3. Layout (refs attached)
   - componentDidMount/Update (class)
   - useLayoutEffect (function)
4. Browser paint

After paint:
5. Passive effects
   - useEffect (function)
```

**Real-world: Animation start position:**

```tsx
function Modal({ open, onClose }: Props) {
  const ref = useRef<HTMLDivElement>(null);
  const [animateIn, setAnimateIn] = useState(false);

  useLayoutEffect(() => {
    if (!open) return;
    // Set initial position (off-screen)
    if (!ref.current) return;
    ref.current.style.transform = "translateY(100vh)";

    // Trigger animation by removing transform (next paint)
    const el = ref.current;
    requestAnimationFrame(() => {
      el.style.transform = "translateY(0)";
    });
  }, [open]);

  return open ? (
    <div ref={ref} style={{ transition: "transform 0.3s" }}>
      Modal content
    </div>
  ) : null;
}
```

**`useLayoutEffect` cleanup:**

```tsx
useLayoutEffect(() => {
  // Sync setup
  const id = trackElement(ref.current);
  return () => {
    untrackElement(id);  // sync cleanup
  };
}, []);
```

Cleanup also sync, runs in Layout sub-phase.

**`useEffect` race with paint:**

```tsx
function Dropdown() {
  const [open, setOpen] = useState(true);
  const [pos, setPos] = useState({ top: 0 });
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {  // ❌ async — flicker
    if (!open || !ref.current) return;
    setPos({ top: ref.current.getBoundingClientRect().bottom });
  }, [open]);

  // Render 1: open=true, pos={top:0} → paint with position 0
  // Effect runs: setPos → render 2 → paint with correct position
  // User sees jump from 0 to correct
}
```

`useLayoutEffect` fixes: setPos runs sync, render 2 happens before paint, no flicker.

**Class equivalent:**

```tsx
// componentDidMount/Update == useLayoutEffect (sync)
class Tooltip extends React.Component {
  componentDidMount() {
    const rect = this.ref.current.getBoundingClientRect();
    this.setState({ pos: { top: rect.bottom } });
  }

  componentDidUpdate(prevProps) {
    if (prevProps.target !== this.props.target) {
      // Same logic
    }
  }
}
```

Class lifecycles roughly map to `useLayoutEffect`. `useEffect` (async) has no class equivalent.

**`useLayoutEffect` vs `useImperativeHandle`:**

```tsx
// useImperativeHandle — same timing as useLayoutEffect (sync, after Mutation)
useImperativeHandle(ref, () => ({
  focus: () => inputRef.current?.focus(),
}), []);
```

Both sync in Layout sub-phase.

**Concurrent rendering implication:**

`useLayoutEffect` blocks paint — long sync work delays UI. Especially bad with concurrent rendering (yieldsiz mid-render).

Best practice: keep `useLayoutEffect` minimal.

</details>

### Edge Cases

- **`useLayoutEffect` with setState loop**: Risk of infinite render. React detects, throws.
- **SSR**: `useLayoutEffect` warns. Use `useEffect` or guard.
- **Cleanup timing**: Sync. Same as effect timing.

### Follow-up savollar

- "`useLayoutEffect` always slower?" — Yes (sync). Use only when paint flicker prevention needed.
- "`getBoundingClientRect` causes layout?" — Yes. Force layout recalc. Combine with `useLayoutEffect` carefully.
- "Animation library uses which?" — Framer Motion uses both: `useLayoutEffect` for measurement, `useEffect` for state updates.

</details>

---

### 21. `useLayoutEffect` use cases — DOM measurement [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Asosiy use case'lar**: (1) **DOM measurement** — `getBoundingClientRect`, `offsetHeight`, scroll position, (2) **Tooltip/dropdown positioning** — element joylashuvini hisoblab, paint'dan oldin set qilish, (3) **Animation start position** — initial transform set qilib, animation trigger, (4) **Focus management** — DOM mutation'dan keyin sync focus, (5) **Scroll restoration** — old scroll position'ni saqlab, yangi DOM'ga apply.

### Kod misoli

```tsx
// 1. DOM measurement — auto-resize
function AutoResizeTextarea({ value, onChange }: Props) {
  const ref = useRef<HTMLTextAreaElement>(null);

  useLayoutEffect(() => {
    if (!ref.current) return;
    ref.current.style.height = "auto";
    ref.current.style.height = `${ref.current.scrollHeight}px`;
  }, [value]);

  return <textarea ref={ref} value={value} onChange={onChange} />;
}

// 2. Tooltip positioning
function Tooltip({ children, content }: Props) {
  const targetRef = useRef<HTMLSpanElement>(null);
  const tooltipRef = useRef<HTMLDivElement>(null);
  const [show, setShow] = useState(false);
  const [pos, setPos] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    if (!show || !targetRef.current || !tooltipRef.current) return;
    const target = targetRef.current.getBoundingClientRect();
    const tip = tooltipRef.current.getBoundingClientRect();

    let top = target.bottom + 5;
    let left = target.left + (target.width - tip.width) / 2;

    // Keep in viewport
    if (left < 0) left = 5;
    if (left + tip.width > window.innerWidth) {
      left = window.innerWidth - tip.width - 5;
    }

    setPos({ top, left });
  }, [show, content]);

  return (
    <>
      <span
        ref={targetRef}
        onMouseEnter={() => setShow(true)}
        onMouseLeave={() => setShow(false)}
      >
        {children}
      </span>
      {show && (
        <div ref={tooltipRef} style={{ position: "fixed", ...pos }}>
          {content}
        </div>
      )}
    </>
  );
}

// 3. Scroll restoration on list change
function MessageList({ messages }: { messages: Message[] }) {
  const containerRef = useRef<HTMLDivElement>(null);
  const prevHeightRef = useRef(0);

  useLayoutEffect(() => {
    if (!containerRef.current) return;
    const newHeight = containerRef.current.scrollHeight;
    const heightDiff = newHeight - prevHeightRef.current;

    // If new messages added at top, adjust scroll
    if (heightDiff > 0) {
      containerRef.current.scrollTop += heightDiff;
    }

    prevHeightRef.current = newHeight;
  }, [messages]);

  return (
    <div ref={containerRef} style={{ overflow: "auto", height: 400 }}>
      {messages.map(m => <div key={m.id}>{m.text}</div>)}
    </div>
  );
}

// 4. Focus management
function Modal({ open, children }: Props) {
  const focusRef = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    if (!open) return;
    focusRef.current?.focus();  // sync focus, no flash of wrong focus
  }, [open]);

  if (!open) return null;
  return (
    <div ref={focusRef} tabIndex={-1}>
      {children}
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**DOM measurement APIs:**

```typescript
// Read-only — trigger layout calculation
element.getBoundingClientRect()  // {top, left, width, height, ...}
element.offsetHeight              // CSS box height
element.offsetWidth
element.clientHeight              // content + padding
element.scrollHeight              // total scrollable area
element.scrollTop                 // current scroll
element.scrollLeft

// Read causes "force layout" (synchronous reflow)
// Browser must recalculate layout if pending mutations
```

**Layout thrashing:**

```tsx
// ❌ Read-write-read-write loop forces layout multiple times
useLayoutEffect(() => {
  for (const el of elements) {
    const rect = el.getBoundingClientRect();  // forces layout
    el.style.height = `${rect.width}px`;       // mutates
    // Next iteration: layout invalid, forces again
  }
});

// ✅ Batch reads, then writes
useLayoutEffect(() => {
  const rects = elements.map(el => el.getBoundingClientRect());  // all reads first
  rects.forEach((rect, i) => {
    elements[i].style.height = `${rect.width}px`;  // all writes after
  });
});
// Single layout cycle
```

**`useEffect` for DOM measurement — flicker:**

```tsx
function Position({ target }: { target: HTMLElement }) {
  const [top, setTop] = useState(0);  // initial: 0

  useEffect(() => {
    setTop(target.getBoundingClientRect().bottom);
  }, [target]);

  return <div style={{ top }}>...</div>;
}

// Render 1: top=0 → paint at top:0
// Effect runs (after paint): setTop → re-render → paint at correct top
// User sees flicker from 0 to correct position
```

`useLayoutEffect` runs before paint — no flicker.

**ResizeObserver alternative:**

```tsx
function Container() {
  const [size, setSize] = useState({ width: 0, height: 0 });
  const ref = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    if (!ref.current) return;
    const observer = new ResizeObserver(entries => {
      const { width, height } = entries[0].contentRect;
      setSize({ width, height });
    });
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  return <div ref={ref}>{size.width}x{size.height}</div>;
}
```

ResizeObserver — async, fires after paint. But works with useLayoutEffect for setup.

**Server-side rendering issue:**

```tsx
function Page() {
  useLayoutEffect(() => {  // ❌ SSR warning
    // ...
  }, []);
}

// Solution 1: skip on server
const useIsomorphicLayoutEffect =
  typeof window !== "undefined" ? useLayoutEffect : useEffect;

// Solution 2: defer to client-only render
function Page() {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);

  if (!mounted) return null;
  return <ClientOnly />;
}
```

**Strict Mode 2x:**

```tsx
useLayoutEffect(() => {
  console.log("L");
  return () => console.log("L-cleanup");
}, []);

// Dev:
// L
// L-cleanup
// L
```

Same as useEffect — must handle remount cycle.

**`useLayoutEffect` + `setState` infinite loop guard:**

```tsx
// ❌ Infinite
useLayoutEffect(() => {
  setState(state + 1);  // every render triggers another
});

// ✅ Conditional
useLayoutEffect(() => {
  if (someCondition) {
    setState(...);
  }
});
```

React detects infinite loops and throws.

**Animation libraries — useLayoutEffect for performance:**

```tsx
// Framer Motion (under hood)
useLayoutEffect(() => {
  // Measure final position
  // Calculate FLIP (First, Last, Invert, Play) animation
}, [layoutDeps]);
```

FLIP technique: measure before/after, compute diff, animate via transform.

**`useImperativeHandle` runs same time:**

```tsx
useImperativeHandle(ref, () => ({
  focus: () => inputRef.current?.focus(),
}), []);

useLayoutEffect(() => {
  ref.current?.focus();  // sync, paint'dan oldin
});

// Both run in Layout sub-phase
```

**Performance considerations:**

`useLayoutEffect` sync ishlaydi va browser paint'ni bloklaydi. Uzunroq ish — visible UI freeze. Faqat DOM measurement va flicker oldini olish uchun ishlatish. Heavy work → `useEffect` (paint'dan keyin async).

**`useLayoutEffect` for libraries:**

Component library tooltips, dropdowns, modals — typically use useLayoutEffect for positioning.

</details>

### Edge Cases

- **`getBoundingClientRect` returns 0**: DOM not yet committed (called before mutation). Use useLayoutEffect, not useEffect.
- **Pseudo-elements (`::before`)**: Not in JS DOM. Can't measure with getBoundingClientRect of pseudo.
- **Hidden elements**: `display: none` → no layout box. `getBoundingClientRect` returns 0.

### Follow-up savollar

- "Use `useLayoutEffect` by default?" — No. `useEffect` default. `useLayoutEffect` only when paint flicker.
- "Performance test difference?" — Profiler. `useLayoutEffect` adds to commit time, `useEffect` to passive time.
- "When useEffect with sync setState OK?" — When flicker invisible (e.g., update title — text not rendered).

</details>

---

### 22. `useInsertionEffect` (R18) — CSS-in-JS uchun [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useInsertionEffect`** (R18) — CSS-in-JS library'lar uchun maxsus hook. Commit Phase'ning **Mutation sub-phase'idan OLDIN** sync chaqiriladi (`useLayoutEffect`'dan ham oldin, refs attach qilinmagan). Maqsad: dynamic stylesheet'larni inject qilish — DOM mutation va keyingi `useLayoutEffect` measurements yangi style'larni hisobga olishi uchun. Userland kod uchun emas — faqat library author'lar.

### To'liq tushuntirish

**Timing (Commit Phase):**

```
Render Phase (done)
  ↓
Commit Phase boshlandi:
1. useInsertionEffect ← Style inject HERE (DOM mutation OLDIDAN)
2. Mutation (DOM elements yangilanadi/yaratilad)
3. Layout
   - refs attach qilinadi
   - useLayoutEffect (DOM measure — yangi stylelar bilan)
4. Browser paint
5. Passive effects (useEffect — async)
```

**Manba:** [react.dev/reference/react/useInsertionEffect](https://react.dev/reference/react/useInsertionEffect) — *"useInsertionEffect is for CSS-in-JS library authors"*

**Use case:**

```tsx
// Library code (styled-components, emotion internal)
function useStyleInjection(css: string) {
  useInsertionEffect(() => {
    const style = document.createElement("style");
    style.textContent = css;
    document.head.appendChild(style);
    return () => document.head.removeChild(style);
  }, [css]);
}
```

### Kod misoli

```tsx
// CSS-in-JS hook (sodda)
function useDynamicCSS(css: string) {
  useInsertionEffect(() => {
    const id = `style-${Math.random()}`;
    const styleEl = document.createElement("style");
    styleEl.id = id;
    styleEl.textContent = css;
    document.head.appendChild(styleEl);
    return () => {
      document.getElementById(id)?.remove();
    };
  }, [css]);
}

// Usage
function Card({ color }: { color: string }) {
  useDynamicCSS(`.card-${color} { color: ${color}; }`);
  return <div className={`card card-${color}`}>Content</div>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why separate from useLayoutEffect:**

```
useLayoutEffect — Layout sub-phase'da (Mutation'dan keyin, refs attached)
- DOM measurements bu yerda qilinadi
- Agar style shu yerda inject qilinsa, measurements yangi style'ga moslab qaytadan ishlashi kerak (layout thrashing)

Yechim: useInsertionEffect Mutation OLDIDAN ishlaydi
- Style'ni inject qiladi (DOM measurements yo'q, refs yo'q)
- Mutation amalga oshganda yangi style allaqachon DOM'da
- useLayoutEffect to'g'ri (final) layout'ni measure qiladi
```

**`useInsertionEffect` restrictions:**

```tsx
useInsertionEffect(() => {
  // ❌ NO refs.current — refs not yet attached
  ref.current?.style.color = "red";

  // ❌ NO setState — would trigger re-render
  setState(newValue);

  // ✅ DOM mutation (style injection)
  document.head.appendChild(styleEl);
}, []);
```

**Commit flow:**

```
1. React renders → workInProgress tree
2. Commit boshlandi:
   a. useInsertionEffect: append <style> to <head> (Mutation OLDIDAN)
   b. Mutation: DOM updated (elements created, attributes set, styles applied)
   c. Layout: refs attached, useLayoutEffect runs
      - Element measurements injected stylelarni hisobga oladi
3. Browser paint
4. useEffect runs (async, post-paint)
```

**Why not just useLayoutEffect:**

```tsx
// ❌ Inject in useLayoutEffect
useLayoutEffect(() => {
  appendStyle();
});

// Issues:
// - If useLayoutEffect in child runs before parent (depth-first):
//   Child's useLayoutEffect measures DOM before parent injects style → wrong measurements
// - Order issues, performance worse
```

**Order: parent vs child:**

```
useInsertionEffect order: child first (bottom-up)
useLayoutEffect order: child first
useEffect order: child first

But useInsertionEffect runs before all useLayoutEffect.
→ All styles injected, then all useLayoutEffect can measure
```

**Real-world: styled-components, emotion:**

```typescript
// Emotion (sodda) — inject style for dynamic className
function styledDiv(props) {
  const className = useGenerateClassName(props);

  useInsertionEffect(() => {
    if (!isInjected(className)) {
      const css = generateCSS(props);
      injectStyle(css, className);
    }
  }, [className]);

  return <div className={className} {...props} />;
}
```

**Application code — don't use:**

```tsx
// ❌ Don't use useInsertionEffect for app logic
function MyComponent() {
  useInsertionEffect(() => {
    fetchData();  // ❌ wrong timing
  });
}

// ✅ Use useEffect or useLayoutEffect
useEffect(() => {
  fetchData();
});
```

`useInsertionEffect` is a low-level escape hatch for library authors only.

**Cleanup runs in same phase:**

```tsx
useInsertionEffect(() => {
  console.log("insert");
  return () => console.log("cleanup");
}, []);

// On unmount:
// "cleanup" runs in Insertion phase (before Layout cleanup)
```

**Limitations:**

- No state updates
- No ref reads
- Pure DOM mutation only
- Userland not recommended

**Migration from `useLayoutEffect` for libraries:**

Pre-R18 styled-components used `useLayoutEffect` for style injection. R18+ migrated to `useInsertionEffect` for better timing.

</details>

### Edge Cases

- **`useInsertionEffect` in nested components**: Order: deepest child first. All children's useInsertionEffect run before useLayoutEffect.
- **SSR**: Same as useLayoutEffect — warning. Most CSS-in-JS extracts at SSR time.
- **Cleanup of removed style**: Be careful with shared styles (multiple components use same className).

### Follow-up savollar

- "When app code might use useInsertionEffect?" — Almost never. If you find a need, likely a wrong abstraction.
- "Difference from `useLayoutEffect` if I just inject `<style>`?" — Layout would invalidate measurements. Insertion runs before Layout.
- "React DevTools shows it?" — Yes, like other hooks.

</details>

---

### 23. `useLayoutEffect` SSR warning va workaround [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Server'da `useLayoutEffect` ishlamaydi (no DOM) — React warning: "useLayoutEffect does nothing on the server". Workaround: `useIsomorphicLayoutEffect` pattern (`typeof window !== "undefined" ? useLayoutEffect : useEffect`). Best — `useEffect` agar visual sync kerak emas.

### Kod misoli

```tsx
// useIsomorphicLayoutEffect pattern
import { useEffect, useLayoutEffect } from "react";

const useIsomorphicLayoutEffect =
  typeof window !== "undefined" ? useLayoutEffect : useEffect;

function MyComponent() {
  useIsomorphicLayoutEffect(() => {
    // DOM measurement — only on client
  }, []);
  return <div />;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Server dispatcher:**

```typescript
const HooksDispatcherOnServer = {
  useLayoutEffect: () => {
    if (__DEV__) {
      warnIfNotEnvironmentRender("useLayoutEffect");
    }
  },
  useEffect: () => {},  // also no-op
  // ...
};
```

**SSR warning:**

Server'da useLayoutEffect chaqirilganda warning chiqadi. `useIsomorphicLayoutEffect` pattern bilan hal qilinadi.

**Library pattern:**

```tsx
// Common in libraries (Radix, Headless UI, etc.)
import { useLayoutEffect, useEffect } from "react";

export const useIsomorphicLayoutEffect =
  typeof window !== "undefined" && typeof window.document?.createElement !== "undefined"
    ? useLayoutEffect
    : useEffect;
```

**Why warning matters:**

Layout effects run sync before paint. Server has no paint cycle → no semantic. Warning informs developer to handle SSR explicitly.

</details>

### Edge Cases

- **Hydration**: `useLayoutEffect` runs after hydration completes (client-side).
- **`useInsertionEffect` SSR**: Same — server no-op.
- **`useEffect` SSR**: Same — server no-op (no commit phase).

### Follow-up savollar

- "Why `useEffect` doesn't warn on server?" — `useEffect` semantically OK to skip (post-paint). `useLayoutEffect` semantic implies sync — warning surfaces SSR awareness.
- "RSC + useLayoutEffect?" — RSC server-only, no client effects. Use Client Component for effects.

</details>

---

<a id="qism-e"></a>

## QISM E: useRef

### 24. `useRef` vs `useState` — qachon qaysi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useRef`** — mutable container, **re-render trigger qilmaydi**. `ref.current` — qiymat saqlash, lekin UI'ga ta'sir qilmaydi. **`useState`** — re-render trigger qiladi, UI shu state'ga bog'liq. Ref'lar: DOM element access, timer ID, prev value, latest closure. State: UI'da ko'rsatiladigan qiymat.

### To'liq tushuntirish

| Aspekt | useState | useRef |
|--------|----------|--------|
| Trigger re-render | ✅ Ha | ❌ Yo'q |
| Render'da o'qish | ✅ Ha | ⚠️ Avoid |
| Mutation | setState | ref.current = ... |
| Persistence | Cross renders | Cross renders |
| Use case | UI value | Hidden value |

### Kod misoli

```tsx
// useState — UI value
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// useRef — DOM access
function Input() {
  const inputRef = useRef<HTMLInputElement>(null);
  return (
    <>
      <input ref={inputRef} />
      <button onClick={() => inputRef.current?.focus()}>Focus</button>
    </>
  );
}

// useRef — timer ID
function Timer() {
  const intervalRef = useRef<number | null>(null);

  useEffect(() => {
    intervalRef.current = window.setInterval(() => console.log("tick"), 1000);
    return () => {
      if (intervalRef.current) clearInterval(intervalRef.current);
    };
  }, []);

  return null;
}

// useRef — prev value
function PrevValue({ value }: { value: number }) {
  // R19: `useRef<number>()` zero-arg olib tashlandi — initial value MAJBURIY
  const prevRef = useRef<number | undefined>(undefined);

  useEffect(() => {
    prevRef.current = value;
  }, [value]);

  return <p>Now: {value}, Prev: {prevRef.current ?? "—"}</p>;
}

// useRef — mutable counter (no re-render)
function ClickCounter() {
  const clicksRef = useRef(0);  // doesn't trigger render

  return (
    <button onClick={() => {
      clicksRef.current += 1;
      console.log("Clicks:", clicksRef.current);
    }}>
      Click (no UI update)
    </button>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal:**

```typescript
// mountRef
function mountRef<T>(initial: T): { current: T } {
  const hook = mountWorkInProgressHook();
  const ref = { current: initial };
  hook.memoizedState = ref;
  return ref;
}

// updateRef — same ref object across renders
function updateRef<T>(): { current: T } {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState;  // same object reference
}
```

**Identity stability:**

```tsx
const ref = useRef(0);
// Render 1: ref = { current: 0 } at memory 0x100
// Render 2: ref = SAME object at 0x100
// Render 3: ref = SAME object at 0x100

// Mutation:
ref.current = 5;
// Object identity unchanged, only `current` field
```

**`ref.current` vs `current = ...`:**

```tsx
// ✅ Mutation OK (no re-render)
ref.current = 5;

// ❌ Reading in render — anti-pattern
function Comp() {
  const ref = useRef(0);
  return <p>{ref.current}</p>;  // ⚠️ unsafe in concurrent rendering
}
// If ref mutated mid-render — different value across re-tries
```

**Render-time read warnings:**

In Strict Mode dev, React may warn or break for unsafe ref reads.

**`useState` for UI, `useRef` for hidden state:**

```tsx
// ❌ useState for non-UI
function ClickAnalytics() {
  const [clicks, setClicks] = useState(0);

  const handleClick = () => {
    setClicks(c => c + 1);  // re-renders entire component for analytics counter
    sendAnalytics(clicks);
  };
}

// ✅ useRef for non-UI
function ClickAnalytics() {
  const clicksRef = useRef(0);

  const handleClick = () => {
    clicksRef.current += 1;
    sendAnalytics(clicksRef.current);
  };
}
```

**`useRef` lazy initial — workaround:**

```tsx
// ❌ useRef no lazy
const ref = useRef(expensiveCompute());  // runs every render

// ✅ Lazy ref pattern
function useLazyRef<T>(init: () => T): React.MutableRefObject<T> {
  const ref = useRef<T | undefined>(undefined);
  if (ref.current === undefined) {
    ref.current = init();
  }
  return ref as React.MutableRefObject<T>;
}
```

**`useRef` for callback ref alternative:**

```tsx
function MyComponent() {
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (ref.current) {
      // DOM available
      console.log(ref.current.offsetHeight);
    }
  });

  return <div ref={ref}>...</div>;
}
```

</details>

### Edge Cases

- **`useRef(null)` initial type**: TypeScript needs explicit `useRef<HTMLInputElement>(null)`.
- **Mutation in render**: Allowed but unsafe in concurrent rendering. Avoid.
- **`useRef` in dep array**: Don't add — ref is stable, no purpose.

### Follow-up savollar

- "Why ref doesn't trigger render?" — Mutating object property doesn't tell React. Only setState/dispatch trigger.
- "useRef vs let in component?" — let resets each render. useRef persists.
- "Multiple refs to same DOM?" — Use ref forwarding or callback refs (see Q23).

</details>

---

### 25. `forwardRef` (R18) vs ref as prop (R19) [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**R18 `forwardRef`** — `ref` prop'ini child komponentga **forward** qilish uchun wrapper. R18'da ref oddiy prop'dek pass qilinmaydi. **R19 ref as prop** — `forwardRef` kerak emas, `ref` oddiy prop. Mavjud `forwardRef` kod hali ishlaydi (deprecated, lekin olib tashlanmagan), yangi kod ref'ni oddiy prop sifatida qabul qiladi. Migration optional.

### Kod misoli

```tsx
// R18 — forwardRef wrapper
import { forwardRef } from "react";

const Input = forwardRef<HTMLInputElement, { placeholder?: string }>(
  function Input(props, ref) {
    return <input ref={ref} {...props} />;
  }
);

// Usage:
const inputRef = useRef<HTMLInputElement>(null);
<Input ref={inputRef} placeholder="Type..." />;

// R19 — ref as plain prop
interface InputProps {
  placeholder?: string;
  ref?: React.Ref<HTMLInputElement>;
}

function Input({ ref, ...props }: InputProps) {
  return <input ref={ref} {...props} />;
}

// Usage same:
<Input ref={inputRef} placeholder="Type..." />;
```

<details>
<summary><strong>Deep Dive</strong></summary>

**R18 `forwardRef` internals:**

```typescript
function forwardRef<P, R>(render: (props: P, ref: React.Ref<R>) => React.ReactNode) {
  return {
    $$typeof: REACT_FORWARD_REF_TYPE,
    render,
  };
}

// React detects $$typeof, calls render with (props, ref)
// Otherwise — ref is special prop, not passed to children automatically
```

**Why forwardRef needed in R18:**

```tsx
// R18 — without forwardRef
function Input(props) {
  return <input {...props} />;
}

const ref = useRef(null);
<Input ref={ref} />;  // ⚠️ ref doesn't reach <input>!
// React: ref is "special" prop, doesn't pass through normally
// You'd get: ref.current = <Input> instance? No — function comp has no instance
// ref is null
```

**R19 — ref is regular prop:**

```tsx
function Input({ ref, ...props }) {
  return <input ref={ref} {...props} />;
}

const ref = useRef(null);
<Input ref={ref} />;  // ref reaches input ✅
```

**Migration:**

```tsx
// Before (R18):
const Button = forwardRef<HTMLButtonElement, Props>((props, ref) => {
  return <button ref={ref} {...props} />;
});

// After (R19):
function Button({ ref, ...props }: Props & { ref?: React.Ref<HTMLButtonElement> }) {
  return <button ref={ref} {...props} />;
}

// Both forms valid in R19
// forwardRef hali deprecated emas
```

**`React.RefAttributes`:**

```tsx
// R19 type pattern
type ButtonProps = {
  label: string;
} & React.RefAttributes<HTMLButtonElement>;
```

**Use cases for ref forwarding:**

- Native input wrapper (focus, blur, validation)
- Custom button component (focus management, keyboard)
- Library components (e.g., react-router Link with ref)

**`useImperativeHandle` with forwardRef:**

```tsx
// R18
const Input = forwardRef<{ focus: () => void }, Props>((props, ref) => {
  const inputRef = useRef<HTMLInputElement>(null);
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
  }), []);
  return <input ref={inputRef} {...props} />;
});

// R19
function Input({ ref, ...props }: Props & { ref?: React.Ref<{ focus: () => void }> }) {
  const inputRef = useRef<HTMLInputElement>(null);
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
  }), []);
  return <input ref={inputRef} {...props} />;
}
```

**Why forwardRef phased out:**

- Wrapper component complexity (HOC pattern)
- Generic components hard with forwardRef
- Inconsistent — ref special vs regular prop
- TypeScript inference better with regular prop

</details>

### Edge Cases

- **`forwardRef` with generics**: TypeScript struggle. R19 ref as prop simpler.
- **`displayName`**: forwardRef wrapper hides component name in DevTools — manual `Component.displayName = "X"`.
- **Spread props with forwardRef**: Works. `<Inner {...props} ref={ref} />`.

### Follow-up savollar

- "Why R19 changed?" — DX, simpler types, no wrapper boilerplate.
- "forwardRef deprecated?" — R19'da deprecated (console warning). Lekin olib tashlanmagan, mavjud kod ishlaydi. Yangi kod uchun ref as prop tavsiya.
- "Class components ref?" — Class instances accept ref by default — refers to instance.

</details>

---

### 26. R19 ref cleanup callback [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**R19 ref cleanup** — ref callback (`<div ref={fn}>`)'dan **return cleanup function** qaytarish mumkin. DOM node o'chirilganda chaqiriladi. Pre-R19 — cleanup'ni `useEffect` ichida manual qilish kerak edi. Yangi syntax — DOM lifecycle bilan tabiiyroq.

### Kod misoli

```tsx
// Pre-R19 — manual cleanup via useEffect
function ResizeWatcher() {
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!ref.current) return;
    const observer = new ResizeObserver(entries => {
      console.log("Resized");
    });
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  return <div ref={ref}>...</div>;
}

// R19 — ref callback with cleanup
function ResizeWatcher() {
  return (
    <div ref={(node) => {
      if (!node) return;
      const observer = new ResizeObserver(() => console.log("Resized"));
      observer.observe(node);
      return () => observer.disconnect();  // ✅ R19 cleanup
    }}>
      ...
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Ref callback evolution:**

```tsx
// Pre-R19 — callback ref
<div ref={(node) => {
  if (node) {
    // mount: node = HTMLElement
  } else {
    // unmount: node = null
  }
}} />

// Pattern: check node truthy/falsy for mount/unmount
// Cleanup logic awkward (split mount setup vs unmount cleanup)

// R19 — return cleanup
<div ref={(node) => {
  // Setup with node
  doSomething(node);

  return () => {
    // Cleanup automatically called when DOM removed
    doCleanup();
  };
}} />
```

**Use cases:**

```tsx
// 1. ResizeObserver
<div ref={(node) => {
  if (!node) return;
  const observer = new ResizeObserver(() => updateSize(node));
  observer.observe(node);
  return () => observer.disconnect();
}} />

// 2. IntersectionObserver
<div ref={(node) => {
  if (!node) return;
  const observer = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting) loadContent();
  });
  observer.observe(node);
  return () => observer.disconnect();
}} />

// 3. Event listener on DOM node
<canvas ref={(node) => {
  if (!node) return;
  const handler = (e) => draw(node, e);
  node.addEventListener("mousemove", handler);
  return () => node.removeEventListener("mousemove", handler);
}} />
```

**Trigger conditions:**

```
Cleanup runs when:
1. Element unmounted from DOM
2. Ref callback identity changes (re-render with new function)
3. Element moved (rare — depends on key changes)
```

**Best practice — useCallback for stability:**

```tsx
function Component() {
  const refCallback = useCallback((node: HTMLElement | null) => {
    if (!node) return;
    setupNode(node);
    return () => cleanupNode(node);
  }, []);  // stable reference

  return <div ref={refCallback}>...</div>;
}

// Without useCallback: refCallback new on every render → cleanup + re-setup every render
```

**vs useEffect:**

```tsx
// useEffect approach
const ref = useRef<HTMLDivElement>(null);
useEffect(() => {
  if (!ref.current) return;
  const observer = new ResizeObserver(...);
  observer.observe(ref.current);
  return () => observer.disconnect();
}, []);  // [] — effect doesn't react to ref change

// R19 callback approach — direct DOM lifecycle
<div ref={(node) => {
  if (!node) return;
  // ...
  return () => observer.disconnect();
}} />
```

**Backward compat:**

```tsx
// Returning anything other than function — old behavior
<div ref={(node) => {
  console.log(node);
  // No return — works as before
}} />

// Returning function — R19+ cleanup
<div ref={(node) => {
  return () => console.log("cleanup");
}} />
```

**Caveats:**

- Cleanup runs synchronously when DOM removed
- Don't update state in cleanup (or guard) — component may be unmounting
- TypeScript types may need manual annotation

</details>

### Edge Cases

- **Returning non-function**: Treated as no cleanup (previous behavior).
- **Async cleanup**: Should be sync. Schedule async work, fire-and-forget.
- **Stale ref in cleanup**: Closure captures node reference — be careful.

### Follow-up savollar

- "Same as `useEffect` cleanup?" — Similar mechanism. Ref-scoped, tied to DOM lifecycle.
- "Pre-R19 alternative?" — useEffect with deps, or manual mount/unmount in callback.
- "Callback ref vs object ref?" — Object ref (`useRef`) — shared identity. Callback ref — function called on attach/detach.

</details>

---

### 27. `useImperativeHandle` — imperative API expose [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useImperativeHandle(ref, factoryFn, deps?)`** — komponent **imperative API**'sini parent ref orqali expose qiladi. Default ref behavior'ni override qiladi (DOM node o'rniga custom object). `deps` argument ko'rsatilishi tavsiya etiladi — bo'lmasa har render'da factory qayta chaqiriladi. Use case: video player controls, modal open/close, complex component'da declarative bilan hal qilib bo'lmaydigan operations. Anti-pattern: declarative bilan hal qilinishi mumkin bo'lgan narsalarga ishlatmaslik.

### Kod misoli

```tsx
interface VideoPlayerRef {
  play: () => void;
  pause: () => void;
  seek: (seconds: number) => void;
  getCurrentTime: () => number;
}

function VideoPlayer({ ref, src }: {
  src: string;
  ref?: React.Ref<VideoPlayerRef>;
}) {
  const videoRef = useRef<HTMLVideoElement>(null);

  useImperativeHandle(ref, () => ({
    play: () => videoRef.current?.play(),
    pause: () => videoRef.current?.pause(),
    seek: (seconds) => {
      if (videoRef.current) videoRef.current.currentTime = seconds;
    },
    getCurrentTime: () => videoRef.current?.currentTime ?? 0,
  }), []);

  return <video ref={videoRef} src={src} controls />;
}

// Parent
function App() {
  const playerRef = useRef<VideoPlayerRef>(null);

  return (
    <>
      <VideoPlayer ref={playerRef} src="/video.mp4" />
      <button onClick={() => playerRef.current?.play()}>Play</button>
      <button onClick={() => playerRef.current?.seek(30)}>Seek 30s</button>
    </>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal:**

```typescript
function mountImperativeHandle(ref, create, deps) {
  const effectDeps = deps !== null && deps !== undefined ? [...deps, ref] : null;
  return mountEffectImpl(
    UpdateEffect | LayoutStaticEffect,
    HookLayout,
    () => imperativeHandleEffect(create, ref),
    effectDeps
  );
}

function imperativeHandleEffect(create, ref) {
  if (typeof ref === "function") {
    const value = create();
    ref(value);
    return () => ref(null);
  } else if (ref !== null && ref !== undefined) {
    const value = create();
    ref.current = value;
    return () => { ref.current = null; };
  }
}
```

**Timing — Layout sub-phase:**

```
Commit Phase:
1. Mutation
2. Layout
   ├─ useImperativeHandle (sync, like useLayoutEffect)
   └─ useLayoutEffect
3. Browser paint
```

**Deps array:**

```tsx
useImperativeHandle(ref, () => ({
  doSomething: () => action(state),
}), [state]);  // Re-creates handle when state changes

// Without deps — re-create every render (wasteful)
useImperativeHandle(ref, () => ({...}));

// Empty deps — never re-create (stale closures!)
useImperativeHandle(ref, () => ({
  current: () => state,  // ❌ state from initial closure
}), []);
```

**With forwardRef (R18):**

```tsx
const VideoPlayer = forwardRef<VideoPlayerRef, Props>(
  function VideoPlayer(props, ref) {
    const videoRef = useRef<HTMLVideoElement>(null);

    useImperativeHandle(ref, () => ({
      play: () => videoRef.current?.play(),
      // ...
    }), []);

    return <video ref={videoRef} src={props.src} />;
  }
);
```

**With ref as prop (R19):**

```tsx
function VideoPlayer({ ref, src }: Props & { ref?: React.Ref<VideoPlayerRef> }) {
  const videoRef = useRef<HTMLVideoElement>(null);

  useImperativeHandle(ref, () => ({
    play: () => videoRef.current?.play(),
  }), []);

  return <video ref={videoRef} src={src} />;
}
```

**Anti-patterns:**

```tsx
// ❌ Imperative for what should be declarative
useImperativeHandle(ref, () => ({
  setLabel: (label) => setLabelState(label),  // ❌ should be prop
}), []);

// ✅ Use prop instead
<MyComponent label={label} />

// ❌ Exposing internal state
useImperativeHandle(ref, () => ({
  state,  // ❌ encapsulation broken
}), [state]);

// ✅ Expose action methods only
useImperativeHandle(ref, () => ({
  reset: () => setState(initial),
  validate: () => /* logic */ true,
}), []);
```

**Real-world examples:**

- Modal: `open()`, `close()`, `getRoot()`
- Form: `submit()`, `reset()`, `validate()`, `getValues()`
- Animation: `play()`, `pause()`, `restart()`
- Editor (rich text): `getContent()`, `setContent()`, `focus()`

**`useImperativeHandle` vs `forwardRef` alone:**

```tsx
// Just forwardRef — exposes DOM element
const Input = forwardRef<HTMLInputElement>((props, ref) => {
  return <input ref={ref} {...props} />;
});
// parent: ref.current = HTMLInputElement (full DOM API)

// forwardRef + useImperativeHandle — custom API
const Input = forwardRef<{ focus: () => void }>((props, ref) => {
  const inputRef = useRef<HTMLInputElement>(null);
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
  }), []);
  return <input ref={inputRef} {...props} />;
});
// parent: ref.current = { focus: () => void } (limited API)
```

`useImperativeHandle` — encapsulation. Hide DOM, expose minimal interface.

</details>

### Edge Cases

- **Forgot deps array**: Re-creates handle every render. Performance issue.
- **Stale closure in handle**: Empty deps + state inside → stale.
- **No ref passed**: Handle still computed but unused. No error.

### Follow-up savollar

- "When use vs prop?" — Imperative when caller "doing" action, prop when state change. Focus, play, scroll — imperative. Visibility, content — prop.
- "Memory cost?" — Object created per render (unless deps stable). Negligible.
- "Class component analog?" — Methods on instance. ref.current = instance, call instance.method().

</details>

---

### 28. Callback refs — dynamic attachment [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Callback ref** — `ref` prop'iga function pass qilish: `<div ref={(node) => ...}>`. Function chaqiriladi: (1) **mount paytida** node bilan, (2) **unmount paytida** `null` bilan, (3) **ref function reference o'zgarsa** — old function null bilan, new function node bilan. Use case: dynamic attachment, multiple refs to same node, ref-based DOM measurements that trigger state.

### Kod misoli

```tsx
// 1. Basic callback ref
function Comp() {
  const setRef = (node: HTMLDivElement | null) => {
    if (node) console.log("Mounted:", node);
    else console.log("Unmounted");
  };
  return <div ref={setRef}>Content</div>;
}

// 2. DOM measurement triggering state
function Resizable() {
  const [width, setWidth] = useState(0);

  const measureRef = useCallback((node: HTMLDivElement | null) => {
    if (!node) return;
    setWidth(node.getBoundingClientRect().width);
  }, []);

  return <div ref={measureRef}>Width: {width}</div>;
}

// 3. Combining multiple refs
function useCombinedRefs<T>(...refs: React.Ref<T>[]): React.RefCallback<T> {
  return useCallback((node) => {
    refs.forEach(ref => {
      if (typeof ref === "function") ref(node);
      else if (ref) (ref as React.MutableRefObject<T | null>).current = node;
    });
  }, refs);
}

const MyComponent = forwardRef<HTMLDivElement>((props, externalRef) => {
  const internalRef = useRef<HTMLDivElement>(null);
  const combinedRef = useCombinedRefs(externalRef, internalRef);

  return <div ref={combinedRef}>...</div>;
});
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Object ref vs callback ref:**

```tsx
// Object ref — useRef
const ref = useRef<HTMLDivElement>(null);
<div ref={ref} />;
// React: ref.current = node (or null on unmount)

// Callback ref — function
<div ref={(node) => {
  // node = HTMLElement | null
}} />
```

**When callback ref called:**

```tsx
function Comp() {
  const setRef = (node) => {
    console.log(node ? "attached" : "detached");
  };

  return <div ref={setRef}>...</div>;
}

// Mount: setRef(<div>)
// Unmount: setRef(null)

// Re-render with NEW function:
function Comp() {
  const setRef = (node) => {  // ❌ new function every render
    console.log(node);
  };
  return <div ref={setRef}>...</div>;
}

// Each render:
// 1. Call OLD setRef(null)  ← cleanup
// 2. Call NEW setRef(node)  ← attach
// = 2 calls per render!
```

**Stable function via useCallback:**

```tsx
function Comp() {
  const setRef = useCallback((node) => {
    console.log(node);
  }, []);  // stable

  return <div ref={setRef}>...</div>;
}

// Mount: setRef(<div>)
// Re-renders: NO call (function same reference)
// Unmount: setRef(null)
```

**Use case: measure on mount + state:**

```tsx
function ContentBox({ children }: { children: React.ReactNode }) {
  const [height, setHeight] = useState(0);

  const measureRef = useCallback((node: HTMLElement | null) => {
    if (node) setHeight(node.offsetHeight);
  }, []);

  return (
    <>
      <p>Height: {height}</p>
      <div ref={measureRef}>{children}</div>
    </>
  );
}

// vs useRef + useLayoutEffect:
function ContentBox({ children }) {
  const ref = useRef<HTMLDivElement>(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    if (ref.current) setHeight(ref.current.offsetHeight);
  });

  return <div ref={ref}>{children}</div>;
}
```

Both work. Callback ref triggers immediately on mount; useLayoutEffect — Layout phase.

**Conditional refs:**

```tsx
function Comp({ measure }: { measure: boolean }) {
  return (
    <div ref={measure ? (node) => console.log(node?.offsetHeight) : null}>
      Content
    </div>
  );
}

// measure=true: callback called on mount
// measure=false: ref ignored (null)
// Toggle measure → callback identity changes → mount/unmount cycle
```

**`React.RefCallback<T>` type:**

```typescript
type RefCallback<T> = (instance: T | null) => void;
type Ref<T> = RefCallback<T> | RefObject<T> | null;
```

**Advanced: combining refs:**

```tsx
import { useMemo } from "react";

function useMergedRefs<T>(...refs: React.Ref<T>[]): React.RefCallback<T> {
  return useMemo(() => {
    return (node: T | null) => {
      refs.forEach(ref => {
        if (typeof ref === "function") {
          ref(node);
        } else if (ref) {
          (ref as React.MutableRefObject<T | null>).current = node;
        }
      });
    };
  }, refs);
}
```

**R19 ref cleanup integration:**

```tsx
// R19 — return cleanup from callback
<div ref={(node) => {
  if (!node) return;
  const observer = new ResizeObserver(...);
  observer.observe(node);
  return () => observer.disconnect();
}} />
```

Pre-R19: needed `useEffect` for cleanup.

**Performance:**

- Callback ref — har attach/detach'da function chaqiriladi
- Object ref + useEffect ko'proq qo'llaniladi (more declarative)
- Callback ref timing critical bo'lganda (mount-time measurement) tanlanadi

</details>

### Edge Cases

- **Inline arrow function**: Re-created every render. Use useCallback for stability.
- **Returning value from ref function**: Pre-R19 — ignored. R19 — function returned = cleanup.
- **null check**: Always check `if (node)` before access.

### Follow-up savollar

- "Object ref or callback ref — which default?" — Object ref (useRef) — simpler. Callback ref for mount-time effects, multiple refs.
- "Performance overhead?" — Callback called twice (re-render with new fn) vs once (object ref). Stable callback negligible.
- "Type errors with callback ref?" — TypeScript usually fine with `RefCallback<T>` type.

</details>

---

### 29. `useRef` initial value — har render evaluation [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useRef(initialValue)` — `initialValue` har render'da hisoblanadi (lekin ignore qilinadi mount'dan keyin). Lazy init pattern yo'q (`useState`'dan farqli). Workaround: `useRef(null)` + lazy assign in render: `if (ref.current === null) ref.current = expensive();`. Yoki `useMemo`-based instance ref.

### Kod misoli

```tsx
// ❌ Each render: expensive call (result discarded after mount)
function BadComponent() {
  const ref = useRef(expensiveCompute());  // called each render!
  return <div>{ref.current}</div>;
}

// ✅ Lazy assign pattern
function GoodComponent() {
  const ref = useRef<MyClass | null>(null);
  if (ref.current === null) {
    ref.current = new MyClass();  // only first render
  }
  return <div>{ref.current.value}</div>;
}

// ✅ Custom hook
function useLazyRef<T>(init: () => T): MutableRefObject<T> {
  const ref = useRef<T | null>(null);
  if (ref.current === null) {
    ref.current = init();
  }
  return ref as MutableRefObject<T>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why no lazy initializer:**

`useRef` API minimal — `useRef(value)` only. `useState` has lazy `useState(() => ...)`. Inconsistency, but R19+ unchanged.

**StrictMode 2x:**

```tsx
// StrictMode dev: 2x mount
function MyComponent() {
  const ref = useRef(new MyClass());  // 2 instances created!
  // First instance dropped, second kept
  // If MyClass has constructor side effects — bug
}

// ✅ Lazy pattern is StrictMode-safe
const ref = useRef<MyClass | null>(null);
if (ref.current === null) {
  ref.current = new MyClass();
}
// First call: ref.current = instance1
// Second call (StrictMode): ref.current already set, skip
```

**Reset pattern:**

```tsx
// Reset on prop change
function Component({ id }: { id: string }) {
  const ref = useRef<MyClass | null>(null);
  const lastIdRef = useRef(id);

  if (lastIdRef.current !== id) {
    ref.current?.cleanup();
    ref.current = new MyClass(id);
    lastIdRef.current = id;
  }
}
```

</details>

### Edge Cases

- **`useRef(undefined)`**: TypeScript — `ref.current: undefined`. Type narrowing needed.
- **Object as initial**: New object each render but only first kept. Wasteful.

### Follow-up savollar

- "useState alternative?" — Yes, lazy init: `useState(() => new MyClass())`. But state triggers re-render on update, ref doesn't.
- "Why React doesn't add useRef lazy init?" — API consistency low priority. Custom hook workaround sufficient.

</details>

---

<a id="qism-f"></a>

## QISM F: useContext

### 30. `useContext` + Provider — prop drilling yechimi [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Context API** — komponent tree bo'ylab data'ni props orqali drilling qilmasdan uzatish. **`createContext(defaultValue)`** — context object yaratadi. **`<Context.Provider value={x}>`** (R18) yoki **`<Context value={x}>`** (R19) — value taqdim etadi. **`useContext(Context)`** — provider'dan value o'qiydi (no provider bo'lsa default). Use case: theme, user, locale, dependency injection.

### Kod misoli

```tsx
import { createContext, useContext, useState } from "react";

// Context yaratish
type Theme = "light" | "dark";
const ThemeContext = createContext<Theme>("light");
const ThemeSetterContext = createContext<(t: Theme) => void>(() => {});

// Provider
function App() {
  const [theme, setTheme] = useState<Theme>("light");

  return (
    // R18: <ThemeContext.Provider value={theme}>
    // R19: <ThemeContext value={theme}>
    <ThemeContext value={theme}>
      <ThemeSetterContext value={setTheme}>
        <Page />
      </ThemeSetterContext>
    </ThemeContext>
  );
}

// Consumer (deep in tree)
function ThemedButton() {
  const theme = useContext(ThemeContext);
  const setTheme = useContext(ThemeSetterContext);

  return (
    <button
      style={{ background: theme === "dark" ? "#000" : "#fff" }}
      onClick={() => setTheme(theme === "dark" ? "light" : "dark")}
    >
      Toggle theme
    </button>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal:**

```typescript
function createContext<T>(defaultValue: T): React.Context<T> {
  const context = {
    $$typeof: REACT_CONTEXT_TYPE,
    Provider: null,
    Consumer: null,
    _currentValue: defaultValue,
    _currentValue2: defaultValue,  // for secondary renderer
    _threadCount: 0,
  };
  context.Provider = {
    $$typeof: REACT_PROVIDER_TYPE,
    _context: context,
  };
  context.Consumer = context;
  return context;
}

function useContext<T>(context: React.Context<T>): T {
  // Subscribe fiber to this context
  const dispatcher = ReactCurrentDispatcher.current;
  return dispatcher.readContext(context);
}
```

**Context update propagation:**

When Provider value changes:
1. Reconciler walks subtree
2. Each fiber subscribed to this context — mark `lanes`
3. Re-render those fibers (and parents up to root)
4. Memo wrappers BYPASSED for context consumers

**Multiple contexts pattern:**

```tsx
// Composition
function App() {
  return (
    <UserContext value={user}>
      <ThemeContext value={theme}>
        <LangContext value={lang}>
          <Page />
        </LangContext>
      </ThemeContext>
    </UserContext>
  );
}

// Helper
function Providers({ children }) {
  return (
    <UserContext value={user}>
      <ThemeContext value={theme}>
        <LangContext value={lang}>
          {children}
        </LangContext>
      </ThemeContext>
    </UserContext>
  );
}
```

**Default value:**

```tsx
const ThemeContext = createContext<Theme>("light");

function Standalone() {
  const theme = useContext(ThemeContext);  // "light" (no provider above)
  return <p>{theme}</p>;
}

// Useful for testing, default behavior
```

**`Context.Consumer` (legacy):**

```tsx
<ThemeContext.Consumer>
  {(theme) => <p>{theme}</p>}
</ThemeContext.Consumer>
```

Function-as-children render prop. Pre-Hooks pattern. Modern: useContext.

**Context vs prop drilling:**

```tsx
// Prop drilling — verbose
<App user={user}>
  <Layout user={user}>
    <Header user={user}>
      <Avatar user={user} />
    </Header>
  </Layout>
</App>

// Context — clean
<UserContext value={user}>
  <App>
    <Layout>
      <Header>
        <Avatar />  {/* useContext inside */}
      </Header>
    </Layout>
  </App>
</UserContext>
```

**Context vs Redux/Zustand:**

| | Context | Redux | Zustand |
|--|---------|-------|---------|
| Use case | Slow-changing data | Fast updates, state management | Simple global state |
| Performance | All consumers re-render | Selector — only subscribers | Selector |
| DevTools | Limited | Excellent | Good |
| Boilerplate | Low | Medium | Low |

Context — for theme, user, locale, etc. (low frequency changes).
Redux/Zustand — for high-frequency state (form inputs, selections).

**`use-context-selector` (dai-shi):**

```tsx
import { useContextSelector } from "use-context-selector";

const value = useContextSelector(MyContext, (state) => state.user.name);
// Re-renders only when state.user.name changes (subset)
```

Not part of React core. Library handles selector pattern.

</details>

### Edge Cases

- **No provider above**: Default value used.
- **Multiple providers same context**: Closest to consumer wins.
- **Provider unmounts**: Consumers default value (or remount).

### Follow-up savollar

- "Context performance — every consumer re-renders?" — Yes, on Provider value change. Mitigate via memo, splitting contexts.
- "Best practice: split contexts?" — Often yes. State and dispatcher separate (e.g., ThemeContext and ThemeSetterContext).
- "Type-safe context?" — Use TypeScript with explicit type. Default value can be `null` with check or initial actual value.

</details>

---

### 31. Context performance — value reference va re-renders [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Provider'ning `value` prop reference (`Object.is` orqali) o'zgarsa, **barcha consumer'lar re-render**. Object literal (`<Provider value={{ x }}>`) — har gal yangi reference → har Provider re-render'da consumer'lar re-render. **Yechim**: `useMemo` bilan stable reference, **split contexts** (state vs dispatcher), or memoize consumer komponentlar.

### Kod misoli

```tsx
// ❌ Inline object — new ref every render
function App() {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeContext value={{ theme, setTheme }}>  {/* ❌ */}
      <Heavy />
    </ThemeContext>
  );
}

// ✅ Memoize value
function App() {
  const [theme, setTheme] = useState("light");
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  return (
    <ThemeContext value={value}>
      <Heavy />
    </ThemeContext>
  );
}

// ✅✅ Split contexts (better)
function App() {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeContext value={theme}>
      <ThemeSetterContext value={setTheme}>  {/* setTheme stable */}
        <Heavy />
      </ThemeSetterContext>
    </ThemeContext>
  );
}

// Consumer using only setter — doesn't re-render on theme change
function ToggleButton() {
  const setTheme = useContext(ThemeSetterContext);
  return <button onClick={() => setTheme("dark")}>Toggle</button>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why every consumer re-renders:**

Context value change → React walks tree → finds consumers → marks them for re-render. No selector mechanism in core API.

**`useSyncExternalStore` for selectors:**

```tsx
const store = createStore({ user: null, theme: "light" });
const StoreContext = createContext(store);

function useSelector<T>(selector: (s: typeof store.state) => T): T {
  const store = useContext(StoreContext);
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.state)
  );
}

// Consumer subscribes to slice, not whole context
const user = useSelector(s => s.user);  // re-renders only on user change
```

**Splitting context pattern:**

```tsx
// Bad — single context with state and setter
const AppContext = createContext({ data, setData, otherState, setOtherState });

// Good — separate contexts
const DataContext = createContext(initialData);
const SetDataContext = createContext<typeof setData>(() => {});
const OtherStateContext = createContext(otherInitial);
```

Components consuming only setter don't re-render on data change.

**`use-context-selector` library:**

```tsx
import { createContext, useContextSelector } from "use-context-selector";

const ctx = createContext({ a: 0, b: 0 });

function Component() {
  const a = useContextSelector(ctx, (s) => s.a);
  // Re-renders only when s.a changes
}
```

Userland library, not React core.

**Multiple contexts cost:**

```tsx
function Component() {
  const a = useContext(AContext);
  const b = useContext(BContext);
  const c = useContext(CContext);
  // Re-renders on any A, B, or C change
}
```

Component subscribed to all 3. Each context update triggers re-render.

**Memoizing consumers:**

```tsx
const MemoChild = memo(function Child() {
  // Doesn't use context
  return <p>Static</p>;
});

function Parent() {
  return (
    <ThemeContext value={theme}>
      <MemoChild />  {/* No context use → memo bailout works */}
    </ThemeContext>
  );
}
```

If consumer doesn't use context, memo bailout works.

But if it does:
```tsx
const MemoChild = memo(function Child() {
  const theme = useContext(ThemeContext);  // subscribes
  return <p>{theme}</p>;
});

// memo BYPASSED for context — Child re-renders on theme change
```

**Context propagation efficiency:**

```typescript
// React algorithm
function propagateContextChange(workInProgress, context, renderLanes) {
  let fiber = workInProgress.child;
  while (fiber !== null) {
    if (subscribesToContext(fiber, context)) {
      markLanes(fiber, renderLanes);
      // Walk up to root: parents' childLanes set
    }
    fiber = nextFiber(fiber);
  }
}
```

Walks subtree, marks consumers. O(n) where n = subtree size.

**Real-world pattern: state + dispatch + actions:**

```tsx
type State = { count: number; name: string };
type Action = { type: "increment" } | { type: "setName"; name: string };

const StateContext = createContext<State>({ count: 0, name: "" });
const DispatchContext = createContext<React.Dispatch<Action>>(() => {});

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "increment": return { ...state, count: state.count + 1 };
    case "setName": return { ...state, name: action.name };
  }
}

function AppProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(reducer, { count: 0, name: "" });

  return (
    <StateContext value={state}>
      <DispatchContext value={dispatch}>  {/* dispatch stable! */}
        {children}
      </DispatchContext>
    </StateContext>
  );
}

// Components using only dispatch don't re-render on state change
function IncrementButton() {
  const dispatch = useContext(DispatchContext);
  return <button onClick={() => dispatch({ type: "increment" })}>+</button>;
}
```

**Selector hook implementation:**

```tsx
function useStateSelector<T>(selector: (s: State) => T): T {
  const state = useContext(StateContext);
  return useMemo(() => selector(state), [state, selector]);
  // Component still re-renders, but useMemo skips if selector result same
  // Issue: component still re-renders even if selector result unchanged
  // True selector pattern: useSyncExternalStore
}
```

**`useSyncExternalStore` for fine-grained:**

```tsx
function useStoreSlice<T>(selector: (s: State) => T): T {
  const subscribe = useCallback((cb) => store.subscribe(cb), []);
  const getSnapshot = useCallback(() => selector(store.state), [selector]);
  return useSyncExternalStore(subscribe, getSnapshot);
}
```

External store + selector = fine-grained subscription.

</details>

### Edge Cases

- **Provider remount**: Consumers reset (Provider value = default → new value).
- **Memo + context**: Context bypasses memo for consumers.
- **Conditional consumer**: `useContext` always called (Rules of Hooks). Use selector pattern for slice.

### Follow-up savollar

- "Context for high-frequency state (form input)?" — Avoid. Re-renders all consumers. Use Redux/Zustand.
- "When split context?" — When some consumers only need setter or subset of state.
- "memo + context — works?" — memo bailout doesn't apply for context changes. Bailout works for parent re-renders without context change.

</details>

---

### 32. R19: `<Context value>` va `use(context)` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**R19 ikki yangilik**: (1) **`<Context value>`** — `<Context.Provider>` o'rniga to'g'ridan-to'g'ri Context as JSX (`<Context.Provider>` R19'da deprecated, console warning chiqadi), (2) **`use(context)`** — `useContext`'dan farqli, **conditional / loop ichida** ishlatilishi mumkin (Rules of Hooks bypass — `use()` hook emas, special function).

### Kod misoli

```tsx
import { createContext, use } from "react";

const ThemeContext = createContext<"light" | "dark">("light");

// R18 syntax
function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Page />
    </ThemeContext.Provider>
  );
}

// R19 syntax — shorter
function App() {
  return (
    <ThemeContext value="dark">
      <Page />
    </ThemeContext>
  );
}

// Conditional use(context) — R19
function Component({ enabled }: { enabled: boolean }) {
  if (!enabled) return null;
  const theme = use(ThemeContext);  // ✅ conditional OK with use()!
  return <p>{theme}</p>;
}

// vs useContext — must be top-level
function Component({ enabled }: { enabled: boolean }) {
  const theme = useContext(ThemeContext);  // top-level
  if (!enabled) return null;
  return <p>{theme}</p>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why `<Context value>`:**

```tsx
// R18 — `.Provider` extra boilerplate
<MyContext.Provider value={x}>
  <App />
</MyContext.Provider>

// R19 — Context can directly act as Provider
<MyContext value={x}>
  <App />
</MyContext>
```

Less verbose, JSX cleaner. Functional equivalence.

**`<Context.Provider>` deprecated?**

R19'da deprecated (console warning chiqadi). Lekin hali olib tashlanmagan — ikkala syntax ishlaydi. Yangi kod uchun `<Context value>` tavsiya.

**`use(context)` semantics:**

```tsx
// useContext — Rules of Hooks (top-level)
function Comp() {
  const x = useContext(Context);  // ✅ top-level
  if (cond) {
    const y = useContext(Other);  // ❌ conditional
  }
}

// use() — relaxed Rules
function Comp() {
  const x = use(Context);
  if (cond) {
    const y = use(Other);  // ✅ conditional OK
  }
  for (const item of items) {
    const z = use(Other);  // ✅ loop OK
  }
}
```

**Why `use()` allows conditional:**

`use()` doesn't store hook state in fiber's hooks list (no slot). Different mechanism — directly reads context value at call time.

```typescript
// use(context) — no fiber state
function use<T>(thenable: Thenable<T> | Context<T>): T {
  if (isContext(thenable)) {
    return readContext(thenable);  // direct read, no hook entry
  }
  if (isThenable(thenable)) {
    return readThenable(thenable);  // suspense integration
  }
  throw new Error("Invalid use() argument");
}
```

**`use()` for Promises:**

```tsx
function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);  // suspends if pending
  return <div>{user.name}</div>;
}

// Wrap in Suspense
<Suspense fallback={<Spinner />}>
  <UserProfile userPromise={fetchUser()} />
</Suspense>
```

`use(promise)` — suspends component until resolved (concurrent rendering).

**Combining use() with conditions:**

```tsx
function Toggle({ on, promise }: Props) {
  if (!on) return <p>Off</p>;

  // ✅ Conditional use — fetch only when needed
  const data = use(promise);
  return <p>{data}</p>;
}
```

**`use()` rules:**

1. ✅ Inside component or custom hook (not regular function)
2. ✅ Conditional, in loop, in early return
3. ❌ Inside event handler (still rules apply)
4. ❌ Inside useEffect callback

**Difference summary:**

| | `useContext` | `use(context)` |
|--|--------------|----------------|
| Rules of Hooks | Strict (top-level) | Relaxed (conditional OK) |
| Type | Hook | Special function |
| Performance | Same | Same |
| Promise support | No | Yes |
| Available since | R16.8 | R19 |

**`<Context.Provider>` migration:**

```tsx
// Codemod (manual or automated):
// Find: <SomeContext.Provider value={x}>
// Replace: <SomeContext value={x}>
// (where SomeContext = createContext)
```

`.Provider` syntax keeps backward compat. No urgent migration.

**Default values still work:**

```tsx
const ThemeContext = createContext<Theme>("light");

// No provider — default
function StandaloneComponent() {
  const theme = useContext(ThemeContext);  // "light" (default)
  return <p>{theme}</p>;
}
```

</details>

### Edge Cases

- **Old codebases mixing both syntaxes**: Both work, no conflict.
- **`use(context)` in event handler**: Error — same Rules of Hooks restriction (use can't be called from event handler).
- **TypeScript types**: `use()` typed correctly for Context, Promise overloads.

### Follow-up savollar

- "Should I migrate to `<Context value>`?" — Optional. New code prefer. Old code — when refactoring nearby.
- "use() vs useContext performance?" — Same. Both call `readContext` internally.
- "use() in custom hook?" — Yes, allowed. Custom hooks can use any hook.

</details>

---

### 33. Context selector pattern — fine-grained subscription [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Context'ning eng katta muammosi**: any `value` o'zgarsa, **barcha consumer'lar** re-render. **Selector pattern** — consumer faqat **state'ning bir qismiga subscribe** bo'ladi, o'zgargan slice'ga moslab re-render. React core'da yo'q — `use-context-selector` library yoki `useSyncExternalStore` bilan implement.

### Kod misoli

```tsx
// ❌ Vanilla context — all consumers re-render
const Context = createContext({ user: null, theme: "light", count: 0 });

function UserName() {
  const { user } = useContext(Context);  // re-renders on theme/count change too!
  return <p>{user?.name}</p>;
}

// ✅ Solution 1: Split contexts
const UserContext = createContext<User | null>(null);
const ThemeContext = createContext<Theme>("light");
const CountContext = createContext(0);

function UserName() {
  const user = useContext(UserContext);  // only re-renders on user change
  return <p>{user?.name}</p>;
}

// ✅ Solution 2: use-context-selector library
import { createContext, useContextSelector } from "use-context-selector";

const Context = createContext({ user: null, theme: "light", count: 0 });

function UserName() {
  const userName = useContextSelector(Context, (s) => s.user?.name);
  // Re-renders only when user.name changes
  return <p>{userName}</p>;
}

// ✅ Solution 3: useSyncExternalStore + store
import { useSyncExternalStore } from "react";

const store = createStore({ user: null, theme: "light", count: 0 });

function useStoreSlice<T>(selector: (s: typeof store.state) => T): T {
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.state)
  );
}

function UserName() {
  const userName = useStoreSlice(s => s.user?.name);
  return <p>{userName}</p>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`use-context-selector` implementation (sodda):**

```typescript
import { createContext as origCreateContext, useContext, useEffect, useRef, useReducer, useCallback, ReactNode } from "react";

type Listener<T> = (next: T) => void;

interface ContextValue<T> {
  value: T;
  subscribe: (l: Listener<T>) => () => void;
  notify: () => void;
}

function createContextSelector<T>(initial: T) {
  const Context = origCreateContext<ContextValue<T>>({
    value: initial,
    subscribe: () => () => {},
    notify: () => {},
  });

  function Provider({ children, value }: { children: ReactNode; value: T }) {
    const valueRef = useRef(value);
    const listenersRef = useRef(new Set<Listener<T>>());

    valueRef.current = value;

    useEffect(() => {
      listenersRef.current.forEach(l => l(value));
    }, [value]);

    const ctxValue = useMemo(() => ({
      get value() { return valueRef.current; },
      subscribe: (l: Listener<T>) => {
        listenersRef.current.add(l);
        return () => listenersRef.current.delete(l);
      },
      notify: () => {},
    }), []);

    return <Context value={ctxValue}>{children}</Context>;
  }

  function useSelector<R>(selector: (s: T) => R): R {
    const ctx = useContext(Context);
    const [state, setState] = useReducer((_, action: R) => action, ctx.value);

    useEffect(() => {
      return ctx.subscribe((next) => {
        const newSelected = selector(next);
        setState(newSelected);
      });
    }, [ctx, selector]);

    return selector(state);
  }

  return { Provider, useSelector };
}
```

**`useSyncExternalStore` approach:**

```typescript
function createSelectorStore<T>(initial: T) {
  let state = initial;
  const listeners = new Set<() => void>();

  const subscribe = (cb: () => void) => {
    listeners.add(cb);
    return () => listeners.delete(cb);
  };

  const setState = (newState: Partial<T>) => {
    state = { ...state, ...newState };
    listeners.forEach(l => l());
  };

  const useSelector = <R>(selector: (s: T) => R): R => {
    return useSyncExternalStore(
      subscribe,
      () => selector(state)
    );
  };

  return { state: () => state, setState, useSelector };
}

const store = createSelectorStore({ user: null, theme: "light" });
const userName = store.useSelector(s => s.user?.name);
```

**Why `useSyncExternalStore`:**

- Tearing prevention in concurrent rendering
- Selector-based subscription
- React core API (R18+)

**Performance comparison:**

```tsx
// Vanilla context
const value = useContext(BigContext);  // gets entire object
// Component re-renders on ANY change

// Selector
const slice = useContextSelector(BigContext, s => s.specificField);
// Re-renders only when specificField changes
```

**Trade-offs:**

| Approach | Pros | Cons |
|----------|------|------|
| Split contexts | React core, simple | Boilerplate, multiple Providers |
| use-context-selector | Selector pattern, simple API | External dependency |
| useSyncExternalStore + store | React core, fine-grained | More setup |
| Redux/Zustand | Full state mgmt, devtools | Library overhead |

**Practical guidance:**

- **2-3 contexts**: Split (no library needed)
- **Many slices, complex state**: External store (Zustand, Redux)
- **Subscriber pattern needed**: `useSyncExternalStore` or library

**`useMemo` doesn't help selector pattern:**

```tsx
function UserName() {
  const ctx = useContext(BigContext);
  const userName = useMemo(() => ctx.user.name, [ctx.user.name]);
  // Component STILL re-renders (useContext re-renders)
  // useMemo just skips deeper computation
}
```

`useMemo` only skips computation, doesn't prevent component re-render.

**`React.memo` + context:**

```tsx
const MemoChild = memo(function Child() {
  const ctx = useContext(BigContext);
  return <p>{ctx.user?.name}</p>;
});

// Parent re-renders without context change: MemoChild bails (memo)
// Context value changes: MemoChild re-renders (context bypasses memo)
```

memo helps for non-context re-renders, doesn't help context changes.

</details>

### Edge Cases

- **Selector returning new object**: Selector function should return primitives or stable refs. `s => ({ a: s.a })` — new object every call.
- **Selector with side effect**: Bad — selector should be pure.
- **Multiple selectors same component**: Each subscribes independently.

### Follow-up savollar

- "React core selector?" — Not yet. RFC discussions over years. `useSyncExternalStore` is the building block.
- "`use(context)` selector support?" — No. Same as `useContext` for context, no slice.
- "Why no built-in?" — Complexity, edge cases. Userland libraries serve.

</details>

---

### 34. Context default value — qachon ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`createContext(defaultValue)` — default value faqat **Provider yo'q bo'lganda** ishlatiladi. Provider value=undefined yoki null — default ishlatilmaydi (undefined/null aniq value). Use case: optional context (theme), test wrapper'siz render. Anti-pattern: default'ga real data qo'yish (Provider unutilsa noto'g'ri behavior).

### Kod misoli

```tsx
// Default value
const ThemeContext = createContext<"light" | "dark">("light");

function ThemedButton() {
  const theme = useContext(ThemeContext);  // ✅ Provider yo'q bo'lsa "light"
  return <button className={theme}>Click</button>;
}

// Without Provider
<ThemedButton />  // theme = "light" (default)

// With Provider
// R19: <ThemeContext value="dark">
// R18: <ThemeContext.Provider value="dark">
<ThemeContext value="dark">
  <ThemedButton />  // theme = "dark" (Provider value)
</ThemeContext>

// Provider value=undefined → STILL undefined (NOT default)
<ThemeContext value={undefined as any}>
  <ThemedButton />  // theme = undefined (warning at runtime)
</ThemeContext>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Type-safe context with required Provider:**

```tsx
// Force Provider via undefined default + custom hook
const UserContext = createContext<User | undefined>(undefined);

function useUser(): User {
  const user = useContext(UserContext);
  if (user === undefined) {
    throw new Error("useUser must be used within UserProvider");
  }
  return user;
}

function ProfileView() {
  const user = useUser();  // ✅ Type narrowed to User (not undefined)
  return <p>{user.name}</p>;
}
```

**Multiple defaults:**

```tsx
const ConfigContext = createContext({
  theme: "light",
  language: "en",
  notifications: true,
});

function App() {
  return (
    <ConfigContext.Provider value={{ theme: "dark", language: "uz", notifications: true }}>
      <Page />
    </ConfigContext.Provider>
  );
}
```

**`use(context)` — R19 conditional:**

```tsx
// R19: conditional context read
function MyComponent({ shouldRead }: { shouldRead: boolean }) {
  if (shouldRead) {
    const ctx = use(MyContext);  // ✅ R19 — conditional OK
    return <p>{ctx}</p>;
  }
  return null;
}
```

**Default value gotchas:**

```tsx
// ❌ Default with side effects
const ApiContext = createContext({
  fetchUser: () => fetch("/api/user"),  // function executes on default
});
// If Provider missing, fetchUser called repeatedly
```

</details>

### Edge Cases

- **`useContext` outside Provider**: Returns default value, no error.
- **Provider value=null**: `useContext` returns null (NOT default).
- **Provider value reference**: Changing reference triggers all consumer re-renders.

### Follow-up savollar

- "Why default value design?" — Optional context (e.g., theme can have sensible default). Test rendering without wrapper.
- "How to make Provider mandatory?" — undefined default + throw in custom hook (above pattern).

</details>

---

<a id="qism-g"></a>

## QISM G: useReducer

### 35. `useReducer` vs `useState` — qachon qaysi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useState`** — sodda state (single value, independent updates). **`useReducer`** — murakkab state, **multiple actions**, related state transitions, complex update logic. Reducer pattern: `(state, action) => newState`. `dispatch` reference stable (useCallback kerak emas), state transitions declarative va testable, large state object'larda yaxshi pattern.

### To'liq tushuntirish

| Aspekt | useState | useReducer |
|--------|----------|------------|
| Use case | Simple values | Complex state, multiple actions |
| API | `[value, setValue]` | `[state, dispatch]` |
| Update | `setValue(x)` | `dispatch({ type, payload })` |
| State transition | Inline | Reducer function (centralized) |
| Test | Direct setValue | Test reducer separately |
| Dispatch stability | setValue stable | dispatch stable |

**Use useReducer when:**
- 3+ related state values
- Multiple action types
- Complex update logic (validation, derived state)
- Action history (undo/redo)
- State machine pattern

### Kod misoli

```tsx
// useState — simple
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// useReducer — complex state
type State = {
  count: number;
  history: number[];
  step: number;
};

type Action =
  | { type: "increment" }
  | { type: "decrement" }
  | { type: "reset" }
  | { type: "setStep"; step: number }
  | { type: "undo" };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "increment":
      return {
        ...state,
        count: state.count + state.step,
        history: [...state.history, state.count],
      };
    case "decrement":
      return {
        ...state,
        count: state.count - state.step,
        history: [...state.history, state.count],
      };
    case "reset":
      return { count: 0, history: [], step: 1 };
    case "setStep":
      return { ...state, step: action.step };
    case "undo": {
      const last = state.history.at(-1);
      if (last === undefined) return state;
      return {
        ...state,
        count: last,
        history: state.history.slice(0, -1),
      };
    }
  }
}

function ComplexCounter() {
  const [state, dispatch] = useReducer(reducer, {
    count: 0,
    history: [],
    step: 1,
  });

  return (
    <div>
      <p>Count: {state.count}</p>
      <p>Step: {state.step}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>−</button>
      <button onClick={() => dispatch({ type: "undo" })}>Undo</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
      <input
        type="number"
        value={state.step}
        onChange={(e) => dispatch({ type: "setStep", step: Number(e.target.value) })}
      />
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal:**

```typescript
function mountReducer<S, A>(reducer: (s: S, a: A) => S, initial: S, init?: (i: S) => S): [S, Dispatch<A>] {
  const hook = mountWorkInProgressHook();
  const initialState = init !== undefined ? init(initial) : initial;
  hook.memoizedState = hook.baseState = initialState;
  hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: initialState,
  };
  const dispatch = (hook.queue.dispatch = dispatchReducerAction.bind(null, currentlyRenderingFiber, hook.queue));
  return [hook.memoizedState, dispatch];
}

function updateReducer<S, A>(reducer: (s: S, a: A) => S): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook();
  let newState = hook.baseState;

  // Process queue
  if (hook.queue.pending !== null) {
    const first = hook.queue.pending.next;
    let update = first;
    do {
      const action = update.action;
      newState = reducer(newState, action);
      update = update.next;
    } while (update !== first);
    hook.queue.pending = null;
  }

  hook.memoizedState = newState;
  hook.queue.lastRenderedReducer = reducer;
  return [newState, hook.queue.dispatch];
}
```

`useState` actually internally uses `useReducer` with `basicStateReducer`:

```typescript
function basicStateReducer<S>(state: S, action: SetStateAction<S>): S {
  return typeof action === "function" ? (action as (s: S) => S)(state) : action;
}

function mountState<S>(initial: S | (() => S)) {
  return mountReducer(basicStateReducer, initial);
}
```

**Lazy initialization (3rd arg):**

```tsx
function init(initialCount: number): State {
  return { count: initialCount, history: [], step: 1 };
}

const [state, dispatch] = useReducer(reducer, 5, init);
// init(5) called once at mount → { count: 5, history: [], step: 1 }
// Useful for expensive initial state computation
```

**Discriminated union types:**

```tsx
type Action =
  | { type: "ADD"; payload: { name: string; price: number } }
  | { type: "REMOVE"; id: string }
  | { type: "CLEAR" };

// TypeScript narrows action.type — exhaustive check
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "ADD":
      action.payload.name;  // ✅ narrowed
      // action.id;  // ❌ TS error
      return ...;
    case "REMOVE":
      action.id;  // ✅ narrowed
      return ...;
    case "CLEAR":
      return ...;
    default: {
      const _exhaustive: never = action;  // exhaustiveness check
      return state;
    }
  }
}
```

**`dispatch` stability:**

```tsx
const [state, dispatch] = useReducer(reducer, initial);

// dispatch — stable across renders (same reference)
useEffect(() => {
  dispatch({ type: "increment" });
}, [dispatch]);  // ⚠️ dispatch never changes, no need in deps
```

dispatch reference stable — pass to children without useCallback wrapping.

**No closure trap:**

```tsx
function Counter() {
  const [count, dispatch] = useReducer((s, a) => s + a, 0);

  useEffect(() => {
    const id = setInterval(() => {
      dispatch(1);  // always uses latest state via reducer
    }, 1000);
    return () => clearInterval(id);
  }, []);  // [] deps OK — dispatch stable, no closure issue
}
```

vs useState with closure:

```tsx
const [count, setCount] = useState(0);
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);  // closure: stale count
  }, 1000);
  return () => clearInterval(id);
}, []);
```

**State machine pattern:**

```tsx
type State =
  | { status: "idle" }
  | { status: "loading"; query: string }
  | { status: "success"; data: Data; query: string }
  | { status: "error"; error: Error; query: string };

type Action =
  | { type: "FETCH"; query: string }
  | { type: "SUCCESS"; data: Data }
  | { type: "ERROR"; error: Error }
  | { type: "RESET" };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "FETCH":
      return { status: "loading", query: action.query };
    case "SUCCESS":
      if (state.status !== "loading") return state;  // invalid transition
      return { status: "success", data: action.data, query: state.query };
    case "ERROR":
      if (state.status !== "loading") return state;
      return { status: "error", error: action.error, query: state.query };
    case "RESET":
      return { status: "idle" };
  }
}
```

State machine — controlled transitions, no impossible states.

**`useImmer` for nested updates:**

```tsx
import { produce } from "immer";

function reducer(state: State, action: Action): State {
  return produce(state, (draft) => {
    switch (action.type) {
      case "ADD_ITEM":
        draft.items.push(action.item);  // mutate-style, immer creates new
        break;
      case "UPDATE_ITEM":
        const item = draft.items.find(i => i.id === action.id);
        if (item) item.name = action.name;
        break;
    }
  });
}
```

**Compared to Redux:**

| | useReducer | Redux |
|--|------------|-------|
| Scope | Component | Global |
| Persistence | Component lifetime | App lifetime |
| Middleware | None | Yes (thunk, saga) |
| DevTools | None | Excellent |
| Boilerplate | Low | Higher |

useReducer + Context = mini Redux for component subtree.

**Async actions:**

```tsx
// Reducer is sync — async logic in dispatch caller
function Component() {
  const [state, dispatch] = useReducer(reducer, initial);

  const fetchData = async () => {
    dispatch({ type: "FETCH" });
    try {
      const data = await api.getData();
      dispatch({ type: "SUCCESS", data });
    } catch (error) {
      dispatch({ type: "ERROR", error });
    }
  };

  return <button onClick={fetchData}>Fetch</button>;
}

// Or thunk-like wrapper
function useThunkReducer<S, A, T>(
  reducer: (s: S, a: A) => S,
  initial: S
): [S, (action: A | ((dispatch: Dispatch<A>, getState: () => S) => void)) => void] {
  const [state, dispatch] = useReducer(reducer, initial);
  const stateRef = useRef(state);
  stateRef.current = state;

  const enhancedDispatch = useCallback((action) => {
    if (typeof action === "function") {
      action(dispatch, () => stateRef.current);
    } else {
      dispatch(action);
    }
  }, []);

  return [state, enhancedDispatch];
}
```

</details>

### Edge Cases

- **Reducer not pure**: Side effects in reducer — bug. Reducer must be pure.
- **Same state returned**: Object.is bailout (no re-render).
- **Returning undefined**: TypeScript catches. Runtime — state becomes undefined.

### Follow-up savollar

- "useState bilan multiple values vs useReducer?" — 2-3 unrelated values: useState. Related, complex transitions: useReducer.
- "Reducer test qilish?" — Pure function — easy: `expect(reducer(state, action)).toEqual(expected)`.
- "useReducer + Context = global?" — Yes, popular pattern (Q30).

</details>

---

### 36. Action discriminated unions + exhaustive check [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Discriminated union** — TypeScript pattern: union type'da har variant **discriminator** (umumiy field, masalan `type`) bilan. Reducer'da `switch (action.type)` — TS har case'da action'ni narrowed type sifatida ko'radi. **Exhaustiveness check** — `default` case'da `const _: never = action;` — agar yangi action turi qo'shilsa lekin handle qilinmagan bo'lsa, TS error.

### Kod misoli

```tsx
// Discriminated union
type Action =
  | { type: "INCREMENT" }
  | { type: "DECREMENT" }
  | { type: "SET_VALUE"; value: number }
  | { type: "ADD_ITEM"; item: { id: string; name: string } };

type State = {
  count: number;
  items: { id: string; name: string }[];
};

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "INCREMENT":
      // TS: action narrowed to { type: "INCREMENT" }
      // action.value — ❌ Property 'value' does not exist
      return { ...state, count: state.count + 1 };

    case "DECREMENT":
      return { ...state, count: state.count - 1 };

    case "SET_VALUE":
      return { ...state, count: action.value };  // ✅ value available

    case "ADD_ITEM":
      return { ...state, items: [...state.items, action.item] };  // ✅ item available

    default: {
      // Exhaustiveness check
      const _exhaustive: never = action;
      // If new action type added but not handled here, TS error:
      // "Type ... is not assignable to type 'never'"
      return state;
    }
  }
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Type narrowing with switch:**

```typescript
type A = { type: "x"; xData: number } | { type: "y"; yData: string };

function fn(a: A) {
  if (a.type === "x") {
    a.xData;  // narrowed to { type: "x"; xData: number }
    // a.yData;  // ❌ error
  } else {
    a.yData;  // narrowed to { type: "y"; yData: string }
  }
}
```

**Exhaustiveness pattern:**

```tsx
type Action = { type: "A" } | { type: "B" };

function reducer(state, action: Action) {
  switch (action.type) {
    case "A": return ...;
    case "B": return ...;
    default: {
      const _: never = action;  // catch unhandled
      throw new Error("Unhandled action");
    }
  }
}

// If we add new action:
type Action = { type: "A" } | { type: "B" } | { type: "C" };

// reducer not updated → TS error in default case:
// "Type '{ type: "C" }' is not assignable to type 'never'"
// Forces handling C
```

**Helper function for exhaustive check:**

```typescript
function assertNever(value: never): never {
  throw new Error(`Unhandled value: ${JSON.stringify(value)}`);
}

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "INCREMENT": return ...;
    case "DECREMENT": return ...;
    default: return assertNever(action);
  }
}
```

**Action creators (Redux-style):**

```tsx
const actions = {
  increment: () => ({ type: "INCREMENT" }) as const,
  setValue: (value: number) => ({ type: "SET_VALUE", value }) as const,
  addItem: (item: Item) => ({ type: "ADD_ITEM", item }) as const,
};

type Action = ReturnType<typeof actions[keyof typeof actions]>;
// Action = { type: "INCREMENT" } | { type: "SET_VALUE"; value: number } | { type: "ADD_ITEM"; item: Item }
```

**`as const` for literal types:**

```typescript
const a = { type: "INCREMENT" };           // type: { type: string }
const b = { type: "INCREMENT" } as const;  // type: { type: "INCREMENT" }
```

`as const` — narrow type to literal.

**Payload pattern:**

```typescript
// Some teams prefer payload field
type Action =
  | { type: "INCREMENT" }
  | { type: "SET_VALUE"; payload: number }
  | { type: "ADD_ITEM"; payload: Item };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "INCREMENT":
      return { ...state, count: state.count + 1 };
    case "SET_VALUE":
      return { ...state, count: action.payload };
    case "ADD_ITEM":
      return { ...state, items: [...state.items, action.payload] };
  }
}
```

Both patterns valid. Modern preference — flat (no payload wrapper).

**With reducer hook:**

```tsx
const [state, dispatch] = useReducer(reducer, initial);

dispatch({ type: "INCREMENT" });
dispatch({ type: "SET_VALUE", value: 10 });
// dispatch({ type: "INCREMENT", value: 10 });  // ❌ TS error: extra prop
// dispatch({ type: "SET_VALUE" });  // ❌ TS error: missing value
```

TypeScript ensures correct shape per action type.

**Dispatch as action creator:**

```tsx
function useTypedReducer() {
  const [state, dispatch] = useReducer(reducer, initial);

  return {
    state,
    increment: () => dispatch({ type: "INCREMENT" }),
    setValue: (value: number) => dispatch({ type: "SET_VALUE", value }),
  };
}

// Usage
const { state, increment } = useTypedReducer();
increment();
```

**Generic reducer pattern:**

```typescript
type Reducer<S, A> = (state: S, action: A) => S;

const reducer: Reducer<State, Action> = (state, action) => {
  switch (action.type) {
    // ...
  }
};
```

**Async actions don't fit reducer (sync only):**

```tsx
// ❌ Async in reducer
function reducer(state, action) {
  switch (action.type) {
    case "FETCH":
      return await api.fetch();  // ❌ async
  }
}

// ✅ Async outside, dispatch sync actions
async function fetchData(dispatch) {
  dispatch({ type: "FETCH_START" });
  try {
    const data = await api.fetch();
    dispatch({ type: "FETCH_SUCCESS", data });
  } catch (error) {
    dispatch({ type: "FETCH_ERROR", error });
  }
}
```

**Redux Toolkit's createSlice pattern:**

```tsx
// Redux Toolkit reduces boilerplate
const counterSlice = createSlice({
  name: "counter",
  initialState: { count: 0 },
  reducers: {
    increment: (state) => { state.count += 1; },  // immer inside
    setValue: (state, action) => { state.count = action.payload; },
  },
});
// Auto-generated action creators and types
```

For useReducer, manual discriminated union — more boilerplate, but core React.

</details>

### Edge Cases

- **Adding new action without handling**: TS exhaustiveness catches.
- **Action with optional fields**: `{ type: "X"; payload?: number }` — TS handles undefined.
- **Generic actions**: `{ type: "SET"; key: keyof State; value: any }` — possible but loose typing.

### Follow-up savollar

- "Why discriminator field?" — TS narrowing. Can be `kind`, `type`, etc. — `type` convention.
- "Reducer pure means?" — No side effects, no API calls, no Date.now. Same input → same output.
- "Test reducer?" — Pure function, easy unit test: `expect(reducer(state, action)).toEqual(expected)`.

</details>

---

### 37. Reducer + Context = mini state container [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Reducer + Context** — useReducer'ni Provider ichiga qo'yib, butun subtree'ga state va dispatch'ni expose qilish. Mini Redux pattern: action dispatching, predictable state updates, separation of concerns. State va dispatch alohida context'larda — performance optimal (split contexts).

### Kod misoli

```tsx
import { createContext, useContext, useReducer, ReactNode } from "react";

// 1. Types
type State = {
  user: User | null;
  cart: Item[];
  theme: "light" | "dark";
};

type Action =
  | { type: "LOGIN"; user: User }
  | { type: "LOGOUT" }
  | { type: "ADD_TO_CART"; item: Item }
  | { type: "REMOVE_FROM_CART"; itemId: string }
  | { type: "TOGGLE_THEME" };

// 2. Reducer
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "LOGIN":
      return { ...state, user: action.user };
    case "LOGOUT":
      return { ...state, user: null, cart: [] };
    case "ADD_TO_CART":
      return { ...state, cart: [...state.cart, action.item] };
    case "REMOVE_FROM_CART":
      return { ...state, cart: state.cart.filter(i => i.id !== action.itemId) };
    case "TOGGLE_THEME":
      return { ...state, theme: state.theme === "light" ? "dark" : "light" };
  }
}

// 3. Contexts (split for performance)
const StateContext = createContext<State | null>(null);
const DispatchContext = createContext<React.Dispatch<Action> | null>(null);

// 4. Provider
function AppProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(reducer, {
    user: null,
    cart: [],
    theme: "light",
  });

  return (
    <StateContext value={state}>
      <DispatchContext value={dispatch}>
        {children}
      </DispatchContext>
    </StateContext>
  );
}

// 5. Custom hooks
function useAppState() {
  const state = useContext(StateContext);
  if (!state) throw new Error("useAppState must be used within AppProvider");
  return state;
}

function useAppDispatch() {
  const dispatch = useContext(DispatchContext);
  if (!dispatch) throw new Error("useAppDispatch must be used within AppProvider");
  return dispatch;
}

// 6. Action creators (optional)
function useAppActions() {
  const dispatch = useAppDispatch();
  return {
    login: (user: User) => dispatch({ type: "LOGIN", user }),
    logout: () => dispatch({ type: "LOGOUT" }),
    addToCart: (item: Item) => dispatch({ type: "ADD_TO_CART", item }),
    removeFromCart: (itemId: string) => dispatch({ type: "REMOVE_FROM_CART", itemId }),
    toggleTheme: () => dispatch({ type: "TOGGLE_THEME" }),
  };
}

// 7. Usage
function App() {
  return (
    <AppProvider>
      <Header />
      <Main />
    </AppProvider>
  );
}

function Header() {
  const { user } = useAppState();
  const { logout } = useAppActions();

  return (
    <header>
      {user ? (
        <>
          <span>{user.name}</span>
          <button onClick={logout}>Logout</button>
        </>
      ) : (
        <LoginButton />
      )}
    </header>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why split state and dispatch contexts:**

```tsx
// ❌ Single context — every consumer re-renders on any state change
const AppContext = createContext({ state, dispatch });

function ButtonOnly() {
  const { dispatch } = useContext(AppContext);  // re-renders on state changes!
  return <button onClick={() => dispatch(...)}>Click</button>;
}

// ✅ Split — dispatch consumer immune to state changes
function ButtonOnly() {
  const dispatch = useContext(DispatchContext);  // dispatch stable
  return <button onClick={() => dispatch(...)}>Click</button>;
}
// Re-renders ONLY on Provider mount/unmount
```

**Pattern variations:**

```tsx
// 1. Single context — simpler
const AppContext = createContext({ state, dispatch });

// 2. Split contexts — performance
const StateContext, DispatchContext;

// 3. Separate hooks per slice — granular
function useUserState() { ... }
function useCartState() { ... }
```

**Async actions:**

```tsx
function useAppActions() {
  const dispatch = useAppDispatch();

  const login = useCallback(async (credentials: Credentials) => {
    dispatch({ type: "LOGIN_START" });
    try {
      const user = await api.login(credentials);
      dispatch({ type: "LOGIN_SUCCESS", user });
    } catch (error) {
      dispatch({ type: "LOGIN_ERROR", error });
    }
  }, [dispatch]);

  return { login };
}
```

**Combine reducers (Redux-style):**

```tsx
const userReducer = (state: UserState, action: Action) => { ... };
const cartReducer = (state: CartState, action: Action) => { ... };

function rootReducer(state: AppState, action: Action): AppState {
  return {
    user: userReducer(state.user, action),
    cart: cartReducer(state.cart, action),
  };
}
```

**Selector pattern:**

```tsx
function useAppSelector<T>(selector: (state: State) => T): T {
  const state = useAppState();
  return selector(state);  // re-renders on any state change
  // For fine-grained: useSyncExternalStore
}

// Usage
const userName = useAppSelector(s => s.user?.name);
```

**Local context for sub-tree:**

```tsx
// Module-specific context
function CartProvider({ children }: { children: ReactNode }) {
  const [cart, dispatch] = useReducer(cartReducer, []);
  return (
    <CartStateContext value={cart}>
      <CartDispatchContext value={dispatch}>
        {children}
      </CartDispatchContext>
    </CartStateContext>
  );
}

// In feature:
<CartProvider>
  <CartUI />
</CartProvider>
```

Co-locate state with feature.

**Dispatch as side effect runner:**

```tsx
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "ADD_ITEM": {
      const newState = { ...state, items: [...state.items, action.item] };
      // ❌ Side effect in reducer (bad)
      // localStorage.setItem("items", JSON.stringify(newState.items));
      return newState;
    }
  }
}

// ✅ Side effects in useEffect or middleware-like wrapper
function CartProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, initial);

  useEffect(() => {
    localStorage.setItem("items", JSON.stringify(state.items));
  }, [state.items]);

  // ...
}
```

**Compared to Zustand:**

```tsx
// Zustand
import { create } from "zustand";

const useStore = create<State>((set, get) => ({
  count: 0,
  increment: () => set(s => ({ count: s.count + 1 })),
  reset: () => set({ count: 0 }),
}));

const count = useStore(s => s.count);  // selector
const increment = useStore(s => s.increment);
```

Zustand: less boilerplate, built-in selector, devtools. But external dependency.

useReducer + Context: pure React, more code.

**Performance comparison:**

| | useReducer + Context | Zustand | Redux |
|--|----------------------|---------|-------|
| Boilerplate | Medium | Low | High |
| Selector | Manual | Built-in | useSelector |
| DevTools | None | Yes | Yes |
| Bundle size | 0 | Small | Medium |
| Async middleware | Manual | Manual | Yes (thunk/saga) |

For small apps: useReducer + Context.
For complex: Zustand or Redux Toolkit.

**Real-world: nested providers:**

```tsx
function App() {
  return (
    <UserProvider>
      <ThemeProvider>
        <CartProvider>
          <Pages />
        </CartProvider>
      </ThemeProvider>
    </UserProvider>
  );
}
```

Each provider = independent reducer + context. Modular.

</details>

### Edge Cases

- **No Provider above**: useContext returns null (with throw guard) or default.
- **Provider value reference**: state and dispatch separately memoized in split.
- **Dispatch identity**: useReducer guarantees stable. No useCallback needed.

### Follow-up savollar

- "When useReducer + Context > Redux?" — Small/medium apps. Single team. Limited middleware needs.
- "DevTools with this pattern?" — Custom dev hook. Or migrate to Zustand/Redux for built-in.
- "Async actions without thunk?" — Function components dispatch async wrappers. Or library.

</details>

---

### 38. `useReducer` lazy init va action creator pattern [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useReducer(reducer, initialArg, init?)` — uchinchi `init` argument lazy initializer. Ishlatish: `useReducer(reducer, props.id, (id) => loadFromCache(id))` — initial state bir marta `init(initialArg)` orqali yaratiladi. Action creators — TypeScript discriminated union'lar bilan exhaustive checking, action object'lar funktsiya orqali yaratish (typo'larni oldini olish).

### Kod misoli

```tsx
type State = { items: Item[]; loading: boolean };
type Action =
  | { type: "LOAD"; payload: Item[] }
  | { type: "ADD"; payload: Item }
  | { type: "REMOVE"; payload: { id: string } }
  | { type: "CLEAR" };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "LOAD": return { ...state, items: action.payload, loading: false };
    case "ADD": return { ...state, items: [...state.items, action.payload] };
    case "REMOVE": return { ...state, items: state.items.filter(i => i.id !== action.payload.id) };
    case "CLEAR": return { ...state, items: [] };
    default: {
      const _exhaustive: never = action;  // ✅ exhaustive check
      return state;
    }
  }
}

// Action creators (type-safe)
const actions = {
  load: (items: Item[]): Action => ({ type: "LOAD", payload: items }),
  add: (item: Item): Action => ({ type: "ADD", payload: item }),
  remove: (id: string): Action => ({ type: "REMOVE", payload: { id } }),
  clear: (): Action => ({ type: "CLEAR" }),
};

function ItemList({ initialId }: { initialId: string }) {
  const [state, dispatch] = useReducer(
    reducer,
    initialId,
    (id) => ({ items: loadFromCache(id), loading: false })  // lazy init
  );

  return (
    <button onClick={() => dispatch(actions.add({ id: "x", name: "X" }))}>
      Add
    </button>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal:**

```typescript
function mountReducer(reducer, initialArg, init) {
  const hook = mountWorkInProgressHook();
  const initialState = init !== undefined ? init(initialArg) : initialArg;
  hook.memoizedState = initialState;
  hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: initialState,
  };
  // ...
}
```

**`useState` is `useReducer` underneath:**

```typescript
function basicStateReducer(state, action) {
  return typeof action === "function" ? action(state) : action;
}

function mountState(initial) {
  return mountReducer(basicStateReducer, initial);
}
```

**Action creator benefits:**

- Type safety (TypeScript narrowing)
- Refactoring (rename safely)
- Testability (assert action shape)
- DevTools integration (action names)

**vs Redux:**

`useReducer` — local state. Redux — global, middleware, devtools, time-travel. Both share reducer pattern.

</details>

### Edge Cases

- **Action without payload**: `{ type: "CLEAR" }` — payload optional in TS union.
- **Bailout**: Reducer returns same state — Object.is bailout, no re-render.
- **Async actions**: Reducer pure — no async. Use `useEffect` after dispatch, or middleware (Redux Thunk pattern).

### Follow-up savollar

- "When useReducer over useState?" — 3+ related state values, complex state transitions, cross-action dependencies.
- "Reducer + Context for global state?" — Yes, common pattern. Provider with useReducer, consumers via useContext.

</details>

---

<a id="qism-h"></a>

## QISM H: useMemo va useCallback

### 39. `useMemo` nima qiladi va qachon ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useMemo(factory, deps)`** — computed value'ni memoize qiladi (cache). Deps `Object.is` orqali bir xil bo'lsa, oldingi cached value qaytadi (factory chaqirilmaydi). **Cache kafolat yo'q** — React har qanday vaqtda cache'ni invalidate qilishi mumkin (memory pressure, Offscreen). Ikki use case: (1) **expensive computation** skip qilish, (2) **referential identity** stabil tutish (object/array — `React.memo` child uchun props reference saqlash).

### Kod misoli

```tsx
// 1. Expensive computation
function App({ users }: { users: User[] }) {
  const sortedUsers = useMemo(() => {
    console.log("sorting");
    return [...users].sort((a, b) => a.name.localeCompare(b.name));
  }, [users]);

  return <UserList users={sortedUsers} />;
}
// users unchanged → "sorting" not logged, cached value returned

// 2. Referential identity for memo'd children
const ExpensiveChild = memo(function Child({ data }: { data: { x: number } }) {
  return <div>{data.x}</div>;
});

function Parent() {
  const [count, setCount] = useState(0);

  // ❌ inline object — new ref every render
  const data = { x: 5 };
  return <ExpensiveChild data={data} />;
  // Each Parent re-render → new data → memo bypass

  // ✅ memoized — stable ref
  const data = useMemo(() => ({ x: 5 }), []);
  return <ExpensiveChild data={data} />;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal:**

```typescript
function mountMemo<T>(factory: () => T, deps: any[] | null): T {
  const hook = mountWorkInProgressHook();
  const value = factory();
  hook.memoizedState = [value, deps];
  return value;
}

function updateMemo<T>(factory: () => T, deps: any[] | null): T {
  const hook = updateWorkInProgressHook();
  const prevState = hook.memoizedState;
  if (deps !== null && prevState !== null) {
    const [prevValue, prevDeps] = prevState;
    if (areHookInputsEqual(deps, prevDeps)) {
      return prevValue;  // bailout — return cached
    }
  }
  const newValue = factory();
  hook.memoizedState = [newValue, deps];
  return newValue;
}
```

**`Object.is` deps comparison:**

```typescript
function areHookInputsEqual(deps, prevDeps) {
  if (prevDeps === null) return false;
  if (deps.length !== prevDeps.length) return false;
  for (let i = 0; i < deps.length; i++) {
    if (!Object.is(deps[i], prevDeps[i])) return false;
  }
  return true;
}
```

**When useMemo helps:**

```tsx
// 1. Heavy computation
const filtered = useMemo(() =>
  hugeArray.filter(complexCondition),
  [hugeArray]
);

// 2. Reference stability for memoized child
const config = useMemo(() => ({ option: true }), []);
<MemoizedChild config={config} />;

// 3. Reference stability for deps
const sortedItems = useMemo(() => sort(items), [items]);
useEffect(() => {
  doSomething(sortedItems);
}, [sortedItems]);  // sortedItems reference stable when items unchanged
```

**When NOT helpful:**

```tsx
// ❌ Cheap computation
const total = useMemo(() => items.reduce((s, i) => s + i, 0), [items]);
// Sum likely faster than memo overhead

// ❌ Always-changing deps
const result = useMemo(() => compute(time), [Date.now()]);
// Time changes every render → cache useless

// ❌ Inline (no memo dep)
useMemo(() => doX(), []);
// Just call doX() once — useMemo overkill
```

**Performance overhead:**

```typescript
// useMemo cost:
// - Hook node yaratish/lookup
// - Deps array allocation
// - Object.is per dep
// - Deps o'zgargan bo'lsa: factory chaqirish

// useMemo qachon foyda beradi:
// - Computation cost > useMemo overhead
// - Yoki reference identity stable bo'lishi kerak (memo'd child, deps array)
// React Profiler bilan o'lchash kerak
```

**React Compiler ko'p `useMemo` ni shart qilmaydi:**

```tsx
// React Compiler'siz — manual memoization
const sortedItems = useMemo(() => sort(items), [items]);

// React Compiler bilan — automatic
const sortedItems = sort(items);  // compiler intermediate cache insert qiladi
```

React Compiler — alohida tool (Babel plugin), R19 bilan release. Memoization automatic, lekin opt-in (config kerak). Manual `useMemo` hali ham qo'llaniladi (compiler ishlatilmagan kodlarda).

**`useMemo` vs lazy initialization:**

```tsx
// useState lazy — once at mount
const [data] = useState(() => expensiveCompute());

// useMemo — re-runs when deps change
const data = useMemo(() => expensiveCompute(), []);

// Different: useMemo is recomputed value, useState is state
```

**Strict Mode 2x:**

```tsx
const value = useMemo(() => {
  console.log("compute");  // dev: logs 2x in StrictMode
  return computeExpensive();
}, [deps]);
```

Factory must be pure (StrictMode tests).

**Multiple memos:**

```tsx
const a = useMemo(() => computeA(x), [x]);
const b = useMemo(() => computeB(y), [y]);
const c = useMemo(() => combine(a, b), [a, b]);

// Chain — each memoizes its part
```

**Memoizing JSX elements:**

```tsx
const cachedHeader = useMemo(() => <Header user={user} />, [user]);

return (
  <div>
    {cachedHeader}
    <Main />
  </div>
);
// Header element cached when user unchanged
// Reconciler element identity bailout
```

**`useMemo` with function:**

```tsx
const callback = useMemo(() => () => doSomething(value), [value]);
// = useCallback(() => doSomething(value), [value])
// Same effect
```

</details>

### Edge Cases

- **No deps array**: Recomputes every render (rarely useful).
- **Empty deps `[]`**: Computes once at mount.
- **Returning function from useMemo**: Same as useCallback.

### Follow-up savollar

- "useMemo overuse — performance worse?" — Possible. Each useMemo has overhead. Profile before adding.
- "useMemo for primitive values?" — Useless — primitives compared by value, not reference.
- "DevTools shows useMemo cache hits?" — Indirect via Profiler. Not explicit.

</details>

---

### 40. `useCallback` — `useMemo` wrapper [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useCallback(fn, deps)`** — function reference'ini memoize qiladi. **Equivalent**: `useMemo(() => fn, deps)`. `useMemo` kabi, **cache kafolat yo'q** — React har qanday vaqtda invalidate qilishi mumkin. Maqsad: function identity'ni stabil tutish — memoized child komponentlarga function pass qilganda, har render'da yangi function reference yaratilmaydi → memo bailout ishlaydi. Effect/memo deps array'da function'ni stable qilish uchun.

### Kod misoli

```tsx
// useCallback ≡ useMemo with function
const handleClick = useCallback(() => action(value), [value]);
// Equivalent:
const handleClick = useMemo(() => () => action(value), [value]);

// Use case: memoized child
const MemoButton = memo(function Button({ onClick, label }: Props) {
  console.log("Button render");
  return <button onClick={onClick}>{label}</button>;
});

function Parent() {
  const [count, setCount] = useState(0);

  // ❌ New function every render
  const handleClick = () => console.log("clicked");
  return <MemoButton onClick={handleClick} label="Click" />;
  // Parent re-render → new handleClick → memo bypass → Button re-renders

  // ✅ Stable function
  const handleClick = useCallback(() => console.log("clicked"), []);
  return <MemoButton onClick={handleClick} label="Click" />;
  // handleClick stable → memo bailout → Button skips render
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal:**

```typescript
function mountCallback<T>(callback: T, deps: any[] | null): T {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = [callback, deps];
  return callback;
}

function updateCallback<T>(callback: T, deps: any[] | null): T {
  const hook = updateWorkInProgressHook();
  const prevState = hook.memoizedState;
  if (deps !== null && prevState !== null) {
    const [prevCallback, prevDeps] = prevState;
    if (areHookInputsEqual(deps, prevDeps)) {
      return prevCallback;  // return cached
    }
  }
  hook.memoizedState = [callback, deps];
  return callback;
}
```

`useCallback(fn, deps)` semantik jihatdan `useMemo(() => fn, deps)` bilan teng — internal implementation ham deyarli bir xil (alohida `Callback` hook flag bilan). Intent clearer — function reference stability uchun mo'ljallangan.

**Equivalence:**

```tsx
// These are identical:
const cb = useCallback(() => fn(x), [x]);
const cb = useMemo(() => () => fn(x), [x]);

// Slight DX difference — useCallback clearer for callbacks
```

**When useCallback helps:**

```tsx
// 1. Memoized child component
const MemoChild = memo(Child);
const handler = useCallback(...);  // stable
<MemoChild onClick={handler} />;

// 2. useEffect dependency
const fetchData = useCallback(() => api.get(`/users/${id}`), [id]);
useEffect(() => {
  fetchData();
}, [fetchData]);  // fetchData stable when id unchanged

// 3. Custom hook return
function useTimer() {
  const [count, setCount] = useState(0);
  const reset = useCallback(() => setCount(0), []);
  return { count, reset };
  // reset stable across renders — caller can use in deps
}
```

**When NOT helpful:**

```tsx
// ❌ No memoized child, no deps
function App() {
  const handleClick = useCallback(() => alert("click"), []);
  return <button onClick={handleClick}>Click</button>;
  // Native button doesn't care about reference identity
  // Plain function would work same
}

// ❌ Function passed once, never compared
const handler = useCallback(() => x, []);
fetch(url, { method: "POST" }).then(handler);
// handler used once — useCallback unnecessary
```

**`useCallback` + `useEffect` pattern:**

```tsx
function Component({ id }: { id: string }) {
  const [data, setData] = useState(null);

  const fetchData = useCallback(async () => {
    const result = await api.get(`/users/${id}`);
    setData(result);
  }, [id]);

  useEffect(() => {
    fetchData();
  }, [fetchData]);

  return <div>{data?.name}</div>;
}

// Without useCallback:
const fetchData = async () => { ... };  // new every render
useEffect(() => { fetchData(); }, [fetchData]);  // runs every render!
```

**Stable callbacks for event handlers (controversial):**

```tsx
// Some teams useCallback all handlers
const handleClick = useCallback(() => doX(), [deps]);

// Others: only when needed (passed to memo'd children)

// Modern guidance with React Compiler: don't useCallback manually,
// Compiler adds automatically.
```

**Closure dep pattern:**

```tsx
const [count, setCount] = useState(0);

// ❌ Closure trap
const handler = useCallback(() => {
  console.log(count);  // closure: count value at memo time
}, []);  // empty deps → handler frozen with initial count

// Fix: include count in deps
const handler = useCallback(() => {
  console.log(count);
}, [count]);
// handler recreated when count changes — but useCallback wraps re-creation

// Alternative: useRef for "always latest"
const countRef = useRef(count);
useEffect(() => { countRef.current = count; });
const handler = useCallback(() => {
  console.log(countRef.current);  // always latest
}, []);  // stable
```

**`useEffectEvent` (experimental — avval `useEvent` deb taklif qilingan):**

2026 holicha experimental — production'da ishlatib bo'lmaydi.

```tsx
import { experimental_useEffectEvent as useEffectEvent } from "react";

function Component({ url }: { url: string }) {
  const onConnected = useEffectEvent(() => {
    log(url);  // captures latest url
  });

  useEffect(() => {
    const conn = createConnection();
    conn.on("connected", onConnected);  // url deps'da kerak emas
    return () => conn.disconnect();
  }, []);
}
```

`useEffectEvent` — "stable function but always sees latest props/state" muammosini hal qiladi.

**`useCallback` vs ref for stability:**

```tsx
// useCallback — recreate on dep change
const handler = useCallback(() => action(x), [x]);

// useRef — never recreate, but read latest from ref
const xRef = useRef(x);
useEffect(() => { xRef.current = x; });
const handler = useRef(() => action(xRef.current)).current;
```

**Performance reality:**

`useCallback` overhead — `useMemo` ga teng (ichida `useMemo` orqali implement qilingan: `useMemo(() => fn, deps)`).

Native DOM event handler (`<button onClick>`)'lar uchun `useCallback` foyda bermaydi — element har render'da qaytadan yaratiladi va handler attribute attach qilinadi. Memo'd child komponentlarga function pass qilinganda `useCallback` muhim — props shallow comparison'da reference o'zgarmasligi kerak.

</details>

### Edge Cases

- **`useCallback(undefined, deps)`**: Returns cached `undefined`. Strange.
- **Inline arrow vs callback**: Inline new every render, useCallback stable.
- **Compiler era**: Manual useCallback may be redundant.

### Follow-up savollar

- "Always useCallback?" — No. Only when memo'd child or in deps array.
- "useCallback returns undefined when fn invalid?" — No, returns whatever you pass. Don't pass undefined.
- "useCallback for inline callbacks?" — `<button onClick={() => x}>` — inline, no useCallback. OK if no memo'd child.

</details>

---

### 41. `memo` + `useCallback` paired pattern [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`React.memo`** child re-render'ni props comparison bilan oldini oladi (shallow check). **`useCallback`** parent'da function reference'ini stabil tutadi → memo'd child'ning props o'zgarmasligini ta'minlaydi. Birga ishlatiladi: parent state update → child memo bailout → faqat parent re-render. Object props'lari uchun ham `useMemo` bilan birga.

### Kod misoli

```tsx
// Memo'd child — props shallow compared
const MemoCard = memo(function Card({
  user,
  onSelect,
}: {
  user: User;
  onSelect: (id: string) => void;
}) {
  console.log("Card render:", user.name);
  return (
    <div onClick={() => onSelect(user.id)}>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  );
});

// ❌ Without useCallback/useMemo
function App({ users }: { users: User[] }) {
  const [selected, setSelected] = useState<string | null>(null);

  const handleSelect = (id: string) => setSelected(id);  // ❌ new every render

  return (
    <>
      <p>Selected: {selected}</p>
      {users.map(u => (
        <MemoCard key={u.id} user={u} onSelect={handleSelect} />
      ))}
    </>
  );
}
// Click any card → setSelected → App re-render → handleSelect new ref
// → ALL MemoCard's re-render (memo bypass)

// ✅ With useCallback
function App({ users }: { users: User[] }) {
  const [selected, setSelected] = useState<string | null>(null);

  const handleSelect = useCallback((id: string) => {
    setSelected(id);
  }, []);  // stable

  return (
    <>
      <p>Selected: {selected}</p>
      {users.map(u => (
        <MemoCard key={u.id} user={u} onSelect={handleSelect} />
      ))}
    </>
  );
}
// Click → setSelected → App re-render → handleSelect SAME ref
// → MemoCard bails out (props unchanged: user same, onSelect same)
// Only App re-renders (state change visible there)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why both needed:**

1. `React.memo` — opt-in for child to bail out on equal props
2. `useCallback`/`useMemo` — parent provides stable references

Without one, the other ineffective:
- `memo` without stable refs → bypass every render
- `useCallback` without `memo` → no benefit (function still recreated logically)

**Object props pattern:**

```tsx
const MemoChild = memo(function Child({ config }: { config: { theme: string } }) {
  return <div>{config.theme}</div>;
});

function Parent() {
  const [state, setState] = useState(0);

  // ❌ Inline object — new ref
  return <MemoChild config={{ theme: "dark" }} />;

  // ✅ useMemo
  const config = useMemo(() => ({ theme: "dark" }), []);
  return <MemoChild config={config} />;

  // ✅ Hoist out of render (if static)
  // (outside component)
  // const STATIC_CONFIG = { theme: "dark" };
  // return <MemoChild config={STATIC_CONFIG} />;
}
```

**Children prop trap:**

```tsx
const MemoCard = memo(function Card({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>;
});

function Parent() {
  const [_, setX] = useState(0);
  return (
    <MemoCard>
      <p>Static content</p>  {/* New element every render! */}
    </MemoCard>
  );
}
// children element new ref every render → memo bypass
```

Workaround: pass primitive or memoize:

```tsx
const cachedChildren = useMemo(() => <p>Static</p>, []);
<MemoCard>{cachedChildren}</MemoCard>;
```

Or:

```tsx
// Element passed from grandparent — stable from there
function Grandparent() {
  return (
    <Parent>
      <p>Static</p>  {/* this element stable in Grandparent's scope */}
    </Parent>
  );
}

function Parent({ children }) {
  const [_, setX] = useState(0);
  return <MemoCard>{children}</MemoCard>;
  // Parent re-renders, but children prop reference from Grandparent — stable
  // memo bailout works
}
```

**Custom equality with memo:**

```tsx
const Card = memo(
  function Card({ user, onSelect }: Props) { ... },
  (prev, next) => {
    // Custom — ignore onSelect changes
    return prev.user.id === next.user.id && prev.user.name === next.user.name;
  }
);

// Pros: ignore unstable function refs
// Cons: maintenance burden, easy to miss changes
```

**`useCallback` + `useMemo` together:**

```tsx
function Parent() {
  const [count, setCount] = useState(0);

  const config = useMemo(() => ({ size: 100 }), []);
  const handleClick = useCallback(() => setCount(c => c + 1), []);

  return <MemoChild config={config} onClick={handleClick} count={count} />;
  // count primitive — only changing prop
  // count change → MemoChild re-renders (count differs)
  // Other props (config, onClick) stable — but irrelevant once count changes
}
```

Memoization helps when **most props stable** but **occasionally one changes**.

**Profile before optimizing:**

```tsx
// Use React DevTools Profiler
// 1. Record interaction
// 2. Check if MemoChild re-renders unnecessarily
// 3. Identify changed props (Profiler shows)
// 4. Memoize those (useCallback/useMemo)
```

Don't memoize prematurely.

**React Compiler future:**

```tsx
// Pre-Compiler
const handleClick = useCallback(() => setCount(c => c + 1), []);
const config = useMemo(() => ({ size: 100 }), []);

// Compiler era — write naturally
const handleClick = () => setCount(c => c + 1);
const config = { size: 100 };
// Compiler memoizes everything
```

**Memo + selector for context:**

```tsx
const MemoUserName = memo(function UserName() {
  const user = useContext(UserContext);
  return <p>{user?.name}</p>;
});

// Memo doesn't help — context bypasses memo
// Use selector pattern (useSyncExternalStore) for fine-grained
```

</details>

### Edge Cases

- **memo + useCallback but no benefit**: Profile. Sometimes other props change, defeating memo.
- **memo with new element children**: Children prop hard to stabilize. Use composition pattern.
- **memo + useImperativeHandle**: imperative API stable enough for memo.

### Follow-up savollar

- "memo + useCallback — overhead worth?" — Profile. Often yes for expensive children.
- "memo'd component called every render?" — Function called when not bailout. Bailout — function not called (cached output).
- "Compiler removes memo too?" — Yes — auto-memoization replaces React.memo for many cases.

</details>

---

### 42. When NOT to memoize — premature optimization [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Memoization erkin emas** — `useMemo`/`useCallback`/`React.memo` har biri overhead'ga ega. **Skip qilish kerak**: (1) **cheap computation** (lookup, sum kichik array), (2) **always-changing deps** (cache useless), (3) **non-memoized children** (memo only helps with memo'd children), (4) **primitive props** (compared by value), (5) **infrequent re-renders**, (6) **Compiler ishlatilganda** (auto-memoization).

### Kod misoli

**Anti-patterns (don't memoize):**

```tsx
// ❌ Cheap computation
const total = useMemo(() => items.reduce((s, i) => s + i, 0), [items]);
// Sum likely faster than useMemo overhead

// ❌ Inline calculation
const half = useMemo(() => width / 2, [width]);
// Division is single CPU op

// ❌ Always changing
const time = useMemo(() => Date.now(), [Date.now()]);
// Cache useless

// ❌ Native button (no memo'd child)
function Comp() {
  const handleClick = useCallback(() => alert("hi"), []);
  return <button onClick={handleClick}>Click</button>;
  // Native button doesn't compare onClick — useCallback wasted
}

// ❌ Primitive prop
const MemoChild = memo(({ count }: { count: number }) => <p>{count}</p>);
// count primitive — cheap to compare
// memo overhead > savings

// ❌ Infrequent renders
function StaticHeader() {
  // Renders once at mount (no state)
  return <header>Static</header>;
}
// memo overhead per check, but no re-renders happen
// Wasted optimization
```

**When TO memoize:**

```tsx
// ✅ Expensive computation
const sortedUsers = useMemo(() => {
  return users.toSorted((a, b) => /* complex comparator */);  // long sync work for large arrays
}, [users]);

// ✅ Object/function passed to memo'd child
const MemoCard = memo(Card);
const config = useMemo(() => ({ theme }), [theme]);
const onClick = useCallback(() => action(), []);
<MemoCard config={config} onClick={onClick} />;

// ✅ Reference for useEffect deps
const processData = useCallback(async () => {
  await api.process(data);
}, [data]);
useEffect(() => { processData(); }, [processData]);
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Cost of useMemo/useCallback:**

```typescript
// Har useMemo/useCallback chaqiruvi:
// - Hook node lookup
// - Deps array allocation
// - Object.is per dep
// - Deps o'zgargan bo'lsa: factory call

// Real overhead: kichik, lekin trivial computation'lar uchun
// (sum, single arithmetic) — overhead computation'dan ko'p bo'lishi mumkin
// React Profiler bilan o'lchash kerak
```

**Savings example — large list:**

```tsx
// Memo'lanmasdan: parent re-render → barcha Cell re-render
function Cell({ row }: { row: Row }) {
  return <td>{row.value}</td>;
}

// 1000 cells va parent tez-tez re-render bo'lsa
// → 1000 ta Cell re-render har gal
// → DOM diff overhead

// Memo bilan:
const MemoCell = memo(Cell);
// Faqat row prop o'zgargan Cell'lar re-render
// Boshqalari memo bailout — DOM diff skip
```

For lists (1000+ items), memoization critical. For 5-10 components, often unnecessary.

**`React.memo` deep equality cost:**

```tsx
const DeepMemo = memo(Card, (prev, next) => isEqual(prev, next));

// isEqual recurses entire object tree
// 100 fields nested — slow
// Often slower than just re-rendering
```

Default shallow check fastest. Custom equality for specific cases.

**Profiling — measure first:**

```typescript
// React DevTools Profiler:
// 1. Click "Record"
// 2. Interact with app
// 3. Check Flame Graph
// 4. Identify slow components
// 5. Memoize those, re-test

// Don't optimize without data
```

**`React.memo` overhead:**

```typescript
// memo komponent'ni wrap qiladi — `MemoComponent` yoki `SimpleMemoComponent` Fiber tag bilan
// Har render — props shallow comparison qilinadi
// Bailout bo'lsa — child render skip

// Foydali:
// - Heavy DOM ren qiluvchi komponent'lar (savings > comparison cost)
// - Ko'p sibling (kombinatsiyalangan savings)

// Foydasiz:
// - Trivial leaf komponent'lar (faqat text)
// - Props doim o'zgaradigan komponent'lar (bailout deyarli yo'q)
```

**`useCallback` no-op cases:**

```tsx
// ❌ DOM event handler — no benefit
function Comp() {
  const handleClick = useCallback(() => alert("hi"), []);
  return <button onClick={handleClick}>Click</button>;
}

// Native button:
// - Doesn't compare onClick reference
// - Re-attaches listener every render anyway
// - useCallback wasted

// ✅ Memo'd child — benefit
function Comp() {
  const handleClick = useCallback(() => alert("hi"), []);
  return <MemoButton onClick={handleClick} />;
}
```

**Children + memo — nuance:**

```tsx
const MemoCard = memo(Card);

function App() {
  const [_, setX] = useState(0);
  return (
    <MemoCard>
      <p>Static</p>  {/* element new every App render */}
    </MemoCard>
  );
}
// children prop new ref → memo bypasses
// Useless memoization
```

Workaround: pass children from grandparent (stable in grandparent's scope).

**React Compiler — eliminates manual:**

```tsx
// Pre-Compiler — manual everywhere
const data = useMemo(() => transform(items), [items]);
const onClick = useCallback(() => action(data), [data]);
const MemoChild = memo(Child);

// Compiler era — none of this
const data = transform(items);
const onClick = () => action(data);
function Child() { ... }
// Compiler memoizes optimally
```

R19+ — manual memo less needed.

**When to skip even with Compiler:**

- Custom areEqual for memo (Compiler can't auto-derive)
- Specific reference identity needs (animation libraries)
- Browser API integration

**Cost-benefit:**

| Component complexity | Memo benefit | Verdict |
|---------------------|--------------|---------|
| Trivial (`<p>{x}</p>`) | Kichik | Skip memo |
| Heavy DOM (table row) | Sezilarli | Memo |
| Complex computation | Katta | useMemo |
| Katta list (ko'p item) | Critical | memo + useCallback |
| Static (no re-renders) | Nol | Don't bother |

**Modern guidance (2026):**

- Use Compiler when possible
- Profile before manual memoization
- Apply memoization to lists, complex components, expensive computations
- Skip for trivial leaves, infrequent renders

</details>

### Edge Cases

- **`React.memo` for component with no props**: `<StaticCard />` — no props, always equal, always bails. But: useless if component has no state.
- **Custom areEqual lying**: Returns true when props actually different — UI shows stale.
- **memo'd component with internal state**: Bailout doesn't prevent self-state-driven re-renders.

### Follow-up savollar

- "Default memoize everything?" — No. Profile, measure, then optimize.
- "Compiler vs manual?" — Compiler for new code. Manual for opt-out (rare cases).
- "useMemo without deps array?" — Recomputes every render — defeats purpose.

</details>

---

### 43. `useMemo` vs `useCallback` — semantic equivalence [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useCallback(fn, deps)` ≡ `useMemo(() => fn, deps)` — semantik teng. `useCallback` faqat **syntactic sugar** function memoization uchun (DX yaxshilash). Ikkalasi ham `memoizedState`'da deps + value saqlaydi, `Object.is` comparison bilan deps tekshiradi. Farq: `useMemo` callback chaqiradi (`fn()` natijasi), `useCallback` callback'ni qaytaradi (`fn` o'zi).

### Kod misoli

```tsx
// useCallback
const handler = useCallback(() => {
  console.log("click");
}, []);

// Equivalent useMemo
const handler = useMemo(() => () => {
  console.log("click");
}, []);

// Type signature
function useCallback<T extends Function>(callback: T, deps: any[]): T;
function useMemo<T>(factory: () => T, deps: any[]): T;
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal implementation:**

```typescript
function mountCallback(callback, deps) {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  hook.memoizedState = [callback, nextDeps];  // store function + deps
  return callback;
}

function updateCallback(callback, deps) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prev = hook.memoizedState;

  if (prev !== null && nextDeps !== null && areHookInputsEqual(nextDeps, prev[1])) {
    return prev[0];  // return cached callback
  }

  hook.memoizedState = [callback, nextDeps];
  return callback;
}

function mountMemo(factory, deps) {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const value = factory();  // call factory
  hook.memoizedState = [value, nextDeps];
  return value;
}
```

**Why both APIs:**

- `useCallback` — clearer intent (function memoization)
- `useMemo` for non-function value — easier to read (`useMemo(() => expensive())`)

**React Compiler — both ortiqcha:**

```tsx
// Pre-Compiler — manual
const handler = useCallback(() => {}, []);
const data = useMemo(() => compute(), []);

// Post-Compiler — auto-memoize, manual ortiqcha
const handler = () => {};
const data = compute();
```

**Common mistakes:**

```tsx
// ❌ Memoizing primitive
const count = useMemo(() => 5, []);  // overhead > benefit

// ❌ Wrong deps
useCallback((id) => fetch(id), []);  // id from outer scope?

// ✅ Correct
useCallback((id: string) => fetch(`/api/${id}`), []);  // id is parameter
```

</details>

### Edge Cases

- **No deps array**: Re-creates each render (no memoization).
- **Empty deps**: Same callback/value forever.
- **Deps reference change**: Re-creates.

### Follow-up savollar

- "Performance — when worth it?" — `React.memo`'d child or expensive compute. Default skip — premature optimization.
- "Compiler renders these unnecessary?" — Yes, mostly. But manual memoization still works.

</details>

---

<a id="qism-i"></a>

## QISM I: Concurrent Hooks (R18)

### 44. `useTransition` — urgent vs non-urgent updates [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useTransition()`** — `[isPending, startTransition]` qaytaradi. `startTransition(fn)` ichidagi setState'lar **non-urgent (TransitionLane)** sifatida belgilanadi — interrupt qilinishi mumkin yangi urgent update kelganda. `isPending` — transition davom etayotganini bildiradi (UI'da spinner ko'rsatish uchun). Use case: heavy filtering, search, navigation — input responsiveness saqlash.

### Kod misoli

```tsx
function SearchPage() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const newQuery = e.target.value;
    setQuery(newQuery);  // urgent — input updates immediately

    startTransition(() => {
      // Non-urgent — can be interrupted
      const filtered = items.filter(i => i.name.includes(newQuery));
      setResults(filtered);
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <ResultsList items={results} />
    </>
  );
}

// Fast typing:
// 1. "a" → query="a" (immediate), startTransition(filter)...
// 2. "ab" → query="ab" (immediate, before filter completes)
//    Old transition INTERRUPTED, new one starts
// 3. Input stays responsive throughout
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal:**

```typescript
function startTransition(scope: () => void) {
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = { /* transition info */ };
  try {
    scope();
  } finally {
    ReactCurrentBatchConfig.transition = prevTransition;
  }
}

// Inside scope, setState calls are marked with TransitionLane priority
// React schedules these as low priority (interruptible)
```

**Use cases:**

```tsx
// 1. Search/filter
startTransition(() => setResults(filter(items, query)));

// 2. Tab switching (heavy tabs)
startTransition(() => setActiveTab("Reports"));

// 3. Form auto-save (background)
startTransition(() => setDraft({ ...formData }));

// 4. Pagination
startTransition(() => setPage(page + 1));
```

**`isPending`:**

```tsx
const [isPending, startTransition] = useTransition();

return (
  <div className={isPending ? "pending" : ""}>
    {isPending && <Spinner />}
    <Content />
  </div>
);

// isPending true: transition active (rendering low priority)
// isPending false: transition complete (committed)
```

**Standalone `startTransition`:**

```tsx
import { startTransition } from "react";

// Outside component — no isPending tracking
startTransition(() => {
  setFoo(value);
});
```

Useful for one-off transitions without UI feedback.

**Async functions (R19+):**

R19 `startTransition` **async function** qabul qila boshladi. R18'da scope sync bo'lishi kerak edi (async setState'lar transition'dan tashqarida bo'lardi):

```tsx
// R18 — async setState transition'dan tushib ketadi
startTransition(async () => {
  const data = await fetchData();
  setData(data);  // ❌ R18'da: bu setState transition'da emas
});

// R19 — to'liq async support
startTransition(async () => {
  const data = await fetchData();
  setData(data);  // ✅ R19'da: bu setState ham transition lane'da
});
```

**Sync code:**

```tsx
// Sync setState — har vaqt transition lane'da
startTransition(() => {
  setData(precomputedData);
});
```

`startTransition` rendering priority'ga ta'sir qiladi — kodingiz execution'ini sekinlashtirmaydi yoki to'xtatmaydi.

**Multiple transitions:**

```tsx
const [isPendingA, startTransitionA] = useTransition();
const [isPendingB, startTransitionB] = useTransition();

// Each tracks own transition, separate isPending
```

React lane'lar set sifatida boshqariladi — har `useTransition` chaqiruvi alohida lane oladi. Detallar internal va versiyalar orasida o'zgarishi mumkin (`ReactFiberLane` source'da batafsil).

**`startTransition` + Suspense:**

```tsx
function App() {
  const [tab, setTab] = useState("home");
  const [isPending, startTransition] = useTransition();

  return (
    <>
      <button onClick={() => startTransition(() => setTab("about"))}>
        About
      </button>
      <Suspense fallback={<Spinner />}>
        {tab === "home" ? <Home /> : <About />}
      </Suspense>
    </>
  );
}

// Without transition: tab change → suspend → fallback shown
// With transition: keep current tab UI until new tab ready (no flash)
```

**Interrupted transition:**

```tsx
// User clicks tab "About" → transition starts
startTransition(() => setTab("about"));

// Mid-render, user clicks tab "Settings" → another transition
startTransition(() => setTab("settings"));

// React: abandon "About" render, switch to "settings"
// "About" never commits
```

**Detecting transition:**

```tsx
function MyButton({ onClick, disabled, children }: Props) {
  const [isPending, startTransition] = useTransition();

  return (
    <button
      disabled={disabled || isPending}
      onClick={() => startTransition(onClick)}
    >
      {isPending ? "..." : children}
    </button>
  );
}
```

**`startTransition` outside React:**

```tsx
// Imperative — not in component
import { startTransition } from "react";

window.addEventListener("scroll", () => {
  startTransition(() => {
    setScrollPos(window.scrollY);
  });
});
```

</details>

### Edge Cases

- **`startTransition` with sync state**: Inner setState marked transition. External setState (separate call) — urgent.
- **Returning value from scope**: Discarded. Side effects only.
- **Async ops in scope (R18)**: Setup sync, async resumed in next event — outside transition. **R19**: async function to'liq support qilinadi — `await` keyin ham setState transition'da qoladi.

### Follow-up savollar

- "When startTransition vs useDeferredValue?" — `startTransition`: when you control setState. `useDeferredValue`: when you receive value as prop.
- "Transition vs throttle?" — Throttle delays calls. Transition deprioritizes work — React decides when to render.
- "isPending after transition completes?" — Resets to false. Track transition window.

</details>

---

### 45. `useDeferredValue` — deferred rendering [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useDeferredValue(value)`** — value'ning **deferred** versiyasini qaytaradi. Initial render — current value. State change — value urgent yangilanadi, deferred copy keyinroq (transition lane'da). Maqsad: heavy child render'ni input lag'siz qilish — input darhol yangilanadi, heavy display deferred. `useTransition` ga o'xshash, lekin **value-based** (props/state'ni control qila olmaganda).

### Kod misoli

```tsx
function ProductPage() {
  const [filter, setFilter] = useState("");
  const deferredFilter = useDeferredValue(filter);

  return (
    <>
      <input
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
      />
      {/* Heavy filtered list uses deferred value */}
      <ExpensiveList filter={deferredFilter} />
    </>
  );
}

// Type "a":
// - filter = "a" (urgent)
// - deferredFilter = "" (still old, transition lane)
// - Input updates immediately
// - ExpensiveList renders with old filter, then with "a" later

// Type "ab":
// - filter = "ab"
// - deferredFilter = "a" (or "" if "a" hasn't committed yet)
// - Input "ab", list lags by one keystroke
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal:**

```typescript
function useDeferredValue<T>(value: T): T {
  const hook = updateWorkInProgressHook();
  const prevValue = hook.memoizedState;

  if (prevValue !== null) {
    if (Object.is(value, prevValue)) {
      return prevValue;  // same — return cached
    }
    // Schedule transition update
    scheduleUpdate(currentlyRenderingFiber, TransitionLane);
  }

  // Mark deferred — return current (will update later in transition)
  hook.memoizedState = value;
  return prevValue ?? value;  // initial: return value, subsequent: return prev until transition completes
}
```

(Simplified — actual implementation more nuanced)

**Difference from useTransition:**

| | useTransition | useDeferredValue |
|--|---------------|------------------|
| Control | startTransition wraps setState | Wraps existing value |
| Use case | When you setState | When you receive prop/state |
| API | `[isPending, startTransition]` | Just deferred value |

**With memoized child:**

```tsx
const MemoExpensive = memo(ExpensiveList);

function ProductPage() {
  const [filter, setFilter] = useState("");
  const deferredFilter = useDeferredValue(filter);

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      <MemoExpensive filter={deferredFilter} />
    </>
  );
}

// memo + deferred = best:
// - Input fast: filter updates urgently
// - List deferred + memo: re-renders when deferredFilter changes
// - Re-render is interruptible (transition)
```

**Visual feedback for delay:**

```tsx
function App() {
  const [filter, setFilter] = useState("");
  const deferredFilter = useDeferredValue(filter);

  const isDeferred = filter !== deferredFilter;

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      <div style={{ opacity: isDeferred ? 0.5 : 1 }}>
        <ExpensiveList filter={deferredFilter} />
      </div>
    </>
  );
}

// While list is updating, dim it (visual hint)
```

**`useDeferredValue` for Suspense:**

```tsx
function App() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Suspense fallback={<Spinner />}>
        <SearchResults query={deferredQuery} />
      </Suspense>
    </>
  );
}

// SearchResults suspends — fallback shown for old query
// New query also suspends — current results visible until new ready
// Smooth transition without flash
```

**`useDeferredValue` initial render:**

```tsx
const deferredValue = useDeferredValue("initial");
// First render: returns "initial" (no defer)
// Subsequent renders: defers updates
```

**R19: `useDeferredValue(value, initialValue?)` — 2-argument signature:**

```tsx
// R18 — initial render returns current value
const deferred = useDeferredValue(query);
// Mount: returns query (no defer)

// R19 — explicit initial value for first render
const deferred = useDeferredValue(query, "");
// Mount: returns "" (initialValue) → triggers transition → after settle returns query
// Foydali: initial render'da expensive child'ni placeholder bilan tezroq paint qilish
```

`initialValue` first render'da qaytariladi, keyin React `value`'ga transition qiladi. Skeleton/placeholder pattern uchun foydali.

**Compared to debounce:**

```tsx
// Debounce — delay all updates by N ms
const debouncedFilter = useDebounce(filter, 300);

// useDeferredValue — defer based on React's scheduler
const deferredFilter = useDeferredValue(filter);

// Debounce: fixed time delay
// Deferred: as fast as React can render (interruptible)
// On fast device: deferredValue updates almost immediately
// On slow device: deferredValue updates when React has time
```

useDeferredValue — adaptive to performance.

**`useDeferredValue` + heavy computation:**

```tsx
function App() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  const results = useMemo(() => {
    return heavySearch(items, deferredQuery);
  }, [items, deferredQuery]);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ResultsList results={results} />
    </>
  );
}

// Search runs only on deferredQuery change
// Input fast, search interruptible
```

**Edge case — value type:**

```tsx
// Primitives — Object.is
const dStr = useDeferredValue("hello");
const dNum = useDeferredValue(42);

// Objects — reference comparison
const dObj = useDeferredValue({ x: 1 });
// New ref each render → always defers
// Workaround: useMemo for object value
```

**Cleanup — no API:**

```tsx
const deferred = useDeferredValue(value);

// On unmount — auto cleanup
// On value change — old deferred discarded
// No manual cleanup
```

</details>

### Edge Cases

- **Value type changes**: TS strict — same type. Runtime: works but bug-prone.
- **Initial value**: First render returns `value` directly (no defer).
- **Comparison via Object.is**: Object props always defer — useMemo source value.

### Follow-up savollar

- "useDeferredValue vs useState transition?" — useState + startTransition more control. useDeferredValue when you receive value (prop, context).
- "Performance benefit?" — Mitigates input lag. Profile before/after.
- "Multiple deferred values?" — Multiple useDeferredValue calls, each independent.

</details>

---

### 46. `useSyncExternalStore` — tearing prevention [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)`** — external store (Redux, Zustand, browser API)'ni React'ga **safe** subscribe qilish. Concurrent rendering'da **tearing prevention** — React render davomida store o'zgarsa, consistent snapshot guarantees. R18'da kiritildi (avval `useState` + `useEffect` pattern bug-prone edi). SSR'da `getServerSnapshot` **majburiy** — server'da `window`/DOM API yo'q, fallback value berish kerak.

### Kod misoli

```tsx
// External store
const store = {
  state: { count: 0 },
  listeners: new Set<() => void>(),

  getState() { return this.state; },

  setState(newState: typeof this.state) {
    this.state = newState;
    this.listeners.forEach(l => l());
  },

  subscribe(listener: () => void) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  },
};

// Hook
function useCount(): number {
  return useSyncExternalStore(
    (cb) => store.subscribe(cb),  // subscribe
    () => store.getState().count   // getSnapshot
  );
}

// Component
function Counter() {
  const count = useCount();
  return <p>{count}</p>;
}

// Update from anywhere
store.setState({ count: 1 });
// All subscribed components re-render (tearing-free)
```

```tsx
// Browser API — window.innerWidth
function useWindowWidth(): number {
  return useSyncExternalStore(
    (cb) => {
      window.addEventListener("resize", cb);
      return () => window.removeEventListener("resize", cb);
    },
    () => window.innerWidth,
    () => 1024  // SSR fallback
  );
}

function Component() {
  const width = useWindowWidth();
  return <p>{width}px</p>;
}
```

```tsx
// localStorage
function useLocalStorage<T>(key: string, defaultValue: T): T {
  return useSyncExternalStore(
    (cb) => {
      window.addEventListener("storage", cb);
      return () => window.removeEventListener("storage", cb);
    },
    () => {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : defaultValue;
    },
    () => defaultValue  // SSR
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why tearing matters:**

Concurrent rendering'da React komponentlarni **interrupt** qilishi mumkin. Render davomida external store o'zgarsa, bir komponent (parent) eski qiymat ko'rsatadi, boshqa (child) yangi — **tearing** (yirtilish):

```tsx
// External store, har 100ms o'zgaradi
function ParentList() {
  return (
    <>
      <UserCount />      {/* count=5 ko'rinishida render boshlanadi */}
      <SlowChild />      {/* uzun render — bu vaqtda store yangilanadi */}
      <UserListItems />  {/* count=6 ko'rinishida render qiladi */}
    </>
  );
  // UI: UserCount "5 users" deydi, lekin UserListItems 6 ta item ko'rsatadi
  // → user uchun mantiqsiz: "5 users" yozuvi va 6 ta item
}
```

`useSyncExternalStore` esa **bir render davomida bir xil snapshot**ni qaytaradi. Tearing detection mexanizmi store render davomida o'zgarsa, sync ravishda re-render qiladi.

**`useSyncExternalStore` guarantee:**

React har render davomida bir xil snapshot'ni qaytaradi. Render davomida store o'zgarsa, React tearing detection ishga tushiradi va sync re-render qiladi (concurrent yielding'ni o'tkazib).

```typescript
// Soddalashtirilgan signature:
function useSyncExternalStore<T>(
  subscribe: (cb: () => void) => () => void,
  getSnapshot: () => T,
  getServerSnapshot?: () => T  // SSR uchun majburiy emas, lekin SSR'da chaqirilsa kerak
): T {
  // 1. Snapshot olish (har render)
  const value = getSnapshot();

  // 2. Subscribe — useLayoutEffect bilan (sync, paint'dan oldin)
  // Tearing detection esa render davomida ishlaydi
  useLayoutEffect(() => {
    const unsubscribe = subscribe(() => {
      // Store o'zgargan — force update
      forceStoreRerender();
    });
    return unsubscribe;
  }, [subscribe]);

  return value;
}
```

(Soddalashtirilgan — actual implementation tearing detection, lane management, SSR boshqaradi)

**Tearing detection:**

```typescript
// React internal
function useSyncExternalStoreImpl(subscribe, getSnapshot) {
  const fiber = currentlyRenderingFiber;
  const hook = mountWorkInProgressHook();

  const value = getSnapshot();

  hook.memoizedState = {
    value,
    getSnapshot,
  };

  // Subscribe in effect
  useEffect(() => {
    const handleChange = () => {
      const newValue = getSnapshot();
      if (!Object.is(hook.memoizedState.value, newValue)) {
        // Force re-render
        forceUpdate();
      }
    };
    return subscribe(handleChange);
  }, [subscribe, getSnapshot]);

  // Detect tearing — check snapshot consistency
  if (!Object.is(value, hook.memoizedState.value)) {
    // Snapshot changed during render — re-render to be safe
    fiber.lanes |= SyncLane;
    forceUpdate();
  }

  return value;
}
```

**Pre-R18 pattern (tearing-prone, R18+ uchun yaroqsiz):**

```tsx
// Eski pattern — concurrent rendering'da tearing yuzaga keladi
function useStore() {
  const [state, setState] = useState(store.getState());

  useEffect(() => {
    const unsubscribe = store.subscribe(() => {
      setState(store.getState());
    });
    return unsubscribe;
  }, []);

  return state;
}

// Issues:
// 1. Tearing in concurrent rendering
// 2. Race conditions on initial render
// 3. Subscription timing bugs
```

**`getServerSnapshot` for SSR:**

```typescript
useSyncExternalStore(
  subscribe,
  getSnapshot,
  getServerSnapshot  // optional, for SSR
);

// Server: getServerSnapshot() (no DOM/browser API)
// Client: getSnapshot()
// Hydration: server snapshot used initially, then client snapshot
```

If no `getServerSnapshot`, hook throws on server render.

**Stable references:**

```tsx
// ❌ Inline functions — re-subscribe every render
function useFoo() {
  return useSyncExternalStore(
    (cb) => store.subscribe(cb),  // new function ref every call!
    () => store.getState()
  );
}

// ✅ useCallback or hoisted
const subscribe = (cb) => store.subscribe(cb);
const getSnapshot = () => store.getState();

function useFoo() {
  return useSyncExternalStore(subscribe, getSnapshot);
}
```

If subscribe changes, re-subscribe. Performance overhead.

**Selector with `useSyncExternalStore`:**

```tsx
function useStoreSelector<T>(selector: (state: State) => T): T {
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState())
  );
}

// Caveat: getSnapshot must return stable value across calls
// If selector returns new object each call → tearing detection fails
```

**Stable selector requirement:**

```tsx
// ❌ Bad selector — new object each call
const obj = useSyncExternalStore(
  store.subscribe,
  () => ({ x: store.getState().x })  // new object every call
);
// React: tearing detected, infinite re-render!

// ✅ Stable — return same value if unchanged
const x = useSyncExternalStore(
  store.subscribe,
  () => store.getState().x  // primitive, stable
);

// Or memoize selector result inside store
```

**Library integration:**

Most state management libraries (Zustand, react-redux 8+) use `useSyncExternalStore` internally for tearing-safety.

**Performance:**

- `getSnapshot` called per render
- `subscribe` only on mount (or when references change)
- Tearing check: O(1)
- Comparable to `useState` + `useEffect` performance, but safer

**`useSyncExternalStoreWithSelector` (in `use-sync-external-store/with-selector` package):**

```tsx
import { useSyncExternalStoreWithSelector } from "use-sync-external-store/with-selector";

const x = useSyncExternalStoreWithSelector(
  store.subscribe,
  store.getState,
  store.getState,  // server
  (state) => state.x,  // selector
  Object.is  // equality
);
```

Builds selector pattern on top of `useSyncExternalStore`.

</details>

### Edge Cases

- **Snapshot returns new object each call**: Re-render loop. Cache or use selector pattern.
- **Subscribe function changes**: Re-subscribes. Use stable reference.
- **SSR without `getServerSnapshot`**: Throws. Provide for SSR support.

### Follow-up savollar

- "Why R18 added this hook?" — Tearing in concurrent rendering. Old `useState + useEffect` pattern unsafe.
- "Replace Redux with this?" — Building block, not full state mgmt. Redux (react-redux 8+) uses `useSyncExternalStore` internally.
- "Performance vs useContext?" — Faster: no whole-tree re-render, granular subscription.

</details>

---

### 47. `useId` — SSR-safe unique ID [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useId()`** — komponent uchun unique ID qaytaradi. ID server va client'da bir xil — **hydration mismatch**siz. Use case: form labels (`<label htmlFor>`), aria attributes (`aria-labelledby`, `aria-describedby`), accessibility. R18+. Random ID (`Math.random`)'dan farqli — deterministic per fiber tree path.

### Kod misoli

```tsx
function FormField({ label }: { label: string }) {
  const id = useId();

  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}

// Server render: id = ":r0:"
// Client hydrate: id = ":r0:"  (matches!)
```

```tsx
// Multiple inputs — extend with suffix
function PasswordField({ id }: { id?: string }) {
  const generatedId = useId();
  const finalId = id ?? generatedId;

  return (
    <>
      <input id={`${finalId}-password`} type="password" />
      <input id={`${finalId}-confirm`} type="password" />
    </>
  );
}

// Or
function ContactForm() {
  const id = useId();

  return (
    <>
      <label htmlFor={`${id}-name`}>Name</label>
      <input id={`${id}-name`} />

      <label htmlFor={`${id}-email`}>Email</label>
      <input id={`${id}-email`} />
    </>
  );
}
```

```tsx
// ARIA labelledby
function Dialog({ children }: { children: React.ReactNode }) {
  const titleId = useId();
  const descId = useId();

  return (
    <div role="dialog" aria-labelledby={titleId} aria-describedby={descId}>
      <h2 id={titleId}>Title</h2>
      <p id={descId}>Description</p>
      {children}
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal:**

```typescript
function useId(): string {
  const hook = updateWorkInProgressHook();
  if (currentlyRenderingFiber.alternate === null) {
    // Mount — generate ID based on fiber tree path
    const treeId = getTreeId(currentlyRenderingFiber);
    const id = `:r${treeId}:`;
    hook.memoizedState = id;
    return id;
  }
  // Update — return cached
  return hook.memoizedState;
}

function getTreeId(fiber: Fiber): string {
  // Walk up tree, build path
  let path = "";
  let current = fiber;
  while (current.return !== null) {
    path = (current.index ?? 0) + path;
    current = current.return;
  }
  return encodeId(path);
}
```

(Simplified — actual implementation more complex)

**ID format: `:r{n}:`**

```tsx
const id1 = useId();  // ":r0:"
const id2 = useId();  // ":r1:"
const id3 = useId();  // ":r2:"
```

Format: `:r{number}:` — chosen to be invalid CSS selector (avoid accidental `getElementById` collision).

**Why not just `Math.random()`:**

```tsx
// ❌ Random — different on server vs client
function FormField() {
  const id = Math.random().toString();  // different each call!
  return <input id={id} />;
}

// SSR: id = "0.123"
// Client: id = "0.456"
// Hydration mismatch!

// ✅ useId — deterministic
function FormField() {
  const id = useId();  // same on server and client
  return <input id={id} />;
}
```

**Why not counter:**

```tsx
// ❌ Counter — order-dependent
let counter = 0;
function FormField() {
  const id = `id-${counter++}`;  // depends on render order
  return <input id={id} />;
}

// Server may render different order than client (hydration)
// Counter values diverge
```

**`useId` based on fiber path:**

ID derives from fiber's position in tree. Same tree on server and client → same ID. Independent of render order.

**Use case: not for list keys:**

```tsx
// ❌ useId for list keys
function List() {
  return items.map(item => <li key={useId()}>{item}</li>);  // ❌ rules of hooks
}

// useId can't be conditionally called per item
// For list keys, use item.id from data
```

**Single useId for multiple IDs:**

```tsx
// More efficient — single useId for related elements
function Field() {
  const id = useId();
  return (
    <>
      <label htmlFor={`${id}-input`}>Name</label>
      <input id={`${id}-input`} aria-describedby={`${id}-hint`} />
      <span id={`${id}-hint`}>Help text</span>
    </>
  );
}

// vs multiple useId calls — also OK
function Field() {
  const inputId = useId();
  const hintId = useId();
  return (
    <>
      <label htmlFor={inputId}>Name</label>
      <input id={inputId} aria-describedby={hintId} />
      <span id={hintId}>Help text</span>
    </>
  );
}
```

**`identifierPrefix` (createRoot/hydrateRoot):**

```typescript
const root = createRoot(container, {
  identifierPrefix: "myapp-",
});

// IDs: "myapp-:r0:", "myapp-:r1:", etc.
```

Useful when multiple React apps on same page (avoid ID collisions).

**`useId` doesn't change:**

```tsx
function Comp() {
  const id = useId();
  // Same ID across re-renders for this fiber
}
```

ID stable for component instance.

**Strict Mode 2x:**

```tsx
const id = useId();  // dev: same value across 2x mount
```

Strict Mode doesn't change ID generation.

**Performance:**

- Path computation O(depth) — komponent tree chuqurligi
- Mount paytida bir marta hisoblanadi, `hook.memoizedState`'da cache qilinadi
- Re-render'larda — cached ID qaytadi (qayta hisoblash yo'q)

**Use cases:**

- Form labels: `<label htmlFor={id}>`
- ARIA: `aria-labelledby`, `aria-describedby`
- React component pairs requiring DOM connection
- Generating unique class names (rare)

**Not for:**

- List keys (use data ID)
- React keys (different concept)
- localStorage keys (need stable across renders, but useId is stable per component instance)

</details>

### Edge Cases

- **`useId` in conditional**: Rules of Hooks violation. Always top-level.
- **Multiple `useId` in same component**: Each unique. Reset on mount.
- **SSR with React.lazy**: Lazy-loaded components get IDs after hydration — careful with hydration timing.

### Follow-up savollar

- "useId for unique class names?" — Possible, but rare. Usually use CSS modules or unique class strategy.
- "Cross-component referencing?" — Need parent to generate ID and pass via props.
- "useId conflict with browser ID?" — `:r0:` invalid CSS selector — won't conflict with normal usage.

</details>

---

### 48. Concurrent hooks decision tree [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Decision tree**: (1) **`useTransition`** — when YOU control setState (filtering, navigation), (2) **`useDeferredValue`** — when you RECEIVE value (prop/context) and want defer, (3) **`useSyncExternalStore`** — external store subscription (tearing prevention), (4) **`useId`** — unique IDs for accessibility/forms.

### Comparison table

| Hook | Use case | Returns | Use when |
|------|----------|---------|----------|
| `useTransition` | Mark state updates as non-urgent | `[isPending, startTransition]` | Search, filter, navigation, heavy state changes |
| `useDeferredValue` | Defer rendering with received value | Deferred value | Receive prop/context, want lag for heavy child |
| `useSyncExternalStore` | External store subscription | Snapshot | Redux, Zustand, browser APIs, libraries |
| `useId` | Unique identifier | String ID | Form labels, ARIA attrs |

### Decision flow

```
Need to defer expensive work?
├── YES
│   ├── I control setState? → useTransition
│   └── I receive value? → useDeferredValue
└── NO

Need to subscribe to external store?
├── YES → useSyncExternalStore
└── NO

Need unique ID for forms/ARIA?
├── YES → useId
└── NO

Don't need concurrent hooks
```

### Kod misoli

```tsx
// Scenario 1: Search — useTransition
function Search() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    setQuery(e.target.value);  // urgent
    startTransition(() => {
      setResults(filter(items, e.target.value));  // non-urgent
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <Results items={results} />
    </>
  );
}

// Scenario 2: Receive prop — useDeferredValue
function Wrapper({ filter }: { filter: string }) {
  const deferredFilter = useDeferredValue(filter);
  return <ExpensiveList filter={deferredFilter} />;
}

// Scenario 3: External store
function CartCount() {
  const count = useSyncExternalStore(
    cartStore.subscribe,
    () => cartStore.getState().items.length
  );
  return <p>{count}</p>;
}

// Scenario 4: Form
function Form() {
  const id = useId();
  return (
    <>
      <label htmlFor={`${id}-name`}>Name</label>
      <input id={`${id}-name`} />
    </>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`useTransition` vs `useDeferredValue` — subtle:**

```tsx
// useTransition — explicit control
function Comp() {
  const [filter, setFilter] = useState("");
  const [, startTransition] = useTransition();

  const handleChange = (e) => {
    startTransition(() => {
      setFilter(e.target.value);  // setState in transition
    });
  };
}

// useDeferredValue — derive from value
function Comp() {
  const [filter, setFilter] = useState("");
  const deferredFilter = useDeferredValue(filter);
  // setFilter urgent, deferredFilter lags
}
```

Use `useTransition` when wrapping the cause; `useDeferredValue` when wrapping the effect.

**Combination scenarios:**

```tsx
// useTransition + useDeferredValue overkill
function Bad() {
  const [filter, setFilter] = useState("");
  const [, startTransition] = useTransition();
  const deferred = useDeferredValue(filter);  // unnecessary if startTransition

  // Just use one
}

// useSyncExternalStore + useDeferredValue
function Reasonable() {
  const value = useSyncExternalStore(...);
  const deferred = useDeferredValue(value);  // defer expensive UI based on value
  return <Heavy data={deferred} />;
}
```

**Performance characteristics:**

| Hook | Impact |
|------|--------|
| `useTransition` | Marks updates non-urgent — interruptible |
| `useDeferredValue` | Schedules deferred update — lower priority |
| `useSyncExternalStore` | Subscription overhead, tearing detection |
| `useId` | Negligible |

**`useEffectEvent` (experimental):**

2026 holicha experimental — production'da ishlatib bo'lmaydi.

```tsx
import { experimental_useEffectEvent as useEffectEvent } from "react";

function Component({ value }: Props) {
  const onSomething = useEffectEvent(() => {
    log(value);  // captures latest value
  });

  useEffect(() => {
    setupListener(onSomething);
  }, []);  // value deps'da kerak emas
}
```

`useEffectEvent` — "stable function with latest closure" muammosini hal qiladi. `useCallback` alternativasi non-render dependencies uchun.

**R19 form/action hooks:**

`useOptimistic`, `useFormStatus`, `useActionState` (QISM J'da batafsil).

**Patterns:**

```tsx
// Pattern 1: Search with optimistic UI
function Search() {
  const [query, setQuery] = useState("");
  const [, startTransition] = useTransition();

  return (
    <>
      <input
        value={query}
        onChange={(e) => {
          setQuery(e.target.value);
          startTransition(() => fetchResults(e.target.value));
        }}
      />
    </>
  );
}

// Pattern 2: Tab switching with Suspense
function Tabs() {
  const [tab, setTab] = useState("home");
  const [, startTransition] = useTransition();

  return (
    <>
      <button onClick={() => startTransition(() => setTab("about"))}>About</button>
      <Suspense fallback={<Spinner />}>
        <Content tab={tab} />
      </Suspense>
    </>
  );
}

// Pattern 3: External store with selector
const store = createStore({ ... });

function useStoreSlice<T>(selector: (s: State) => T): T {
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState())
  );
}

// Pattern 4: Form with ARIA
function Form() {
  const id = useId();
  return (
    <form>
      <label htmlFor={`${id}-email`}>Email</label>
      <input id={`${id}-email`} aria-describedby={`${id}-help`} />
      <p id={`${id}-help`}>We'll never share your email.</p>
    </form>
  );
}
```

</details>

### Edge Cases

- **All concurrent hooks in same component**: OK. Independent.
- **Concurrent rendering disabled**: useTransition no-op. useDeferredValue immediate.
- **SSR**: useId works. useSyncExternalStore needs `getServerSnapshot`. useTransition/useDeferredValue limited.

### Follow-up savollar

- "When use none?" — Static, simple components. No async, no expensive renders.
- "Mixing transition with regular setState?" — Same render cycle if same lane batch. Transitions deprioritized.
- "TanStack Query/SWR — uses these?" — Yes, internally for tearing-safe caching.

</details>

---

### 49. `useSyncExternalStore` SSR — getServerSnapshot pattern [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)` — concurrent-safe external store integration. **3-chi argument** `getServerSnapshot` — SSR'da chaqiriladi (server'da). Hydration mismatch oldi olish — server snapshot (stable) vs client snapshot (dynamic) ajratish. Patternlar: window size (server: false default), localStorage (server: empty), media query.

### Kod misoli

```tsx
// Window size hook
function useWindowSize() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener("resize", callback);
      return () => window.removeEventListener("resize", callback);
    },
    () => ({ width: window.innerWidth, height: window.innerHeight }),  // client snapshot
    () => ({ width: 0, height: 0 })  // server snapshot (SSR-safe default)
  );
}

// Media query hook
function useMediaQuery(query: string) {
  return useSyncExternalStore(
    (callback) => {
      const mql = window.matchMedia(query);
      mql.addEventListener("change", callback);
      return () => mql.removeEventListener("change", callback);
    },
    () => window.matchMedia(query).matches,  // client
    () => false  // server (default — matches=false on SSR)
  );
}

// localStorage hook (SSR-safe)
function useLocalStorage(key: string) {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener("storage", callback);
      return () => window.removeEventListener("storage", callback);
    },
    () => localStorage.getItem(key),
    () => null  // server has no localStorage
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Without getServerSnapshot:**

```tsx
// ❌ SSR error
function BadHook() {
  return useSyncExternalStore(
    subscribe,
    () => window.innerWidth  // ❌ window undefined on server
  );
}
// Server render: ReferenceError
```

**Snapshot stability — concurrent rendering:**

```typescript
// React calls getSnapshot multiple times during render
// Must return same value if store unchanged (Object.is)
function getSnapshot() {
  return store.state;  // ✅ stable reference if state unchanged
}

// ❌ Anti-pattern
function badGetSnapshot() {
  return { ...store.state };  // ❌ new object each call → tearing detected
}
```

**Tearing prevention:**

`useSyncExternalStore` ensures all components see same snapshot in single render — no inconsistency.

**Library integration (Redux, Zustand):**

```tsx
// Redux v8+ — built-in
import { useSelector } from "react-redux";
// useSelector internally uses useSyncExternalStore

// Zustand
const useStore = create<State>(...);
// internally useSyncExternalStore
```

**Manual `subscribe` pattern:**

```tsx
const store = {
  state: { count: 0 },
  listeners: new Set<() => void>(),
  subscribe(listener: () => void) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  },
  setState(newState: any) {
    this.state = newState;
    this.listeners.forEach(l => l());
  },
};

function useStore() {
  return useSyncExternalStore(
    store.subscribe.bind(store),
    () => store.state,
    () => store.state  // SSR initial
  );
}
```

</details>

### Edge Cases

- **`getSnapshot` returns object**: Must be stable reference (memoize or store).
- **Subscribe returns void**: Should return cleanup. Memory leak otherwise.
- **Multiple components, same store**: Each subscribes — store state shared, snapshot consistent.

### Follow-up savollar

- "When useSyncExternalStore vs useState?" — useState — internal state. useSyncExternalStore — external (Redux, browser API).
- "Performance overhead?" — Minimal. Subscription called once per mount, snapshot per render.

</details>

---

<a id="qism-j"></a>

## QISM J: R19 Hooks

### 50. `use()` — Promise va Context conditional [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`use()` (R19)** — Promise yoki Context'ni conditional/loop ichida o'qish imkonini beradigan **special function** (hook emas, Rules of Hooks bypass). `use(promise)` — Suspense bilan integration: pending promise — komponent suspends. `use(context)` — `useContext` analogi, lekin conditional ishlaydi.

### Kod misoli

```tsx
import { use, Suspense } from "react";

// 1. use(promise) — Suspense integration
async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);  // suspends if pending
  return <div>{user.name}</div>;
}

function App() {
  const userPromise = useMemo(() => fetchUser("123"), []);

  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}

// 2. Conditional use(promise)
function Conditional({ enabled, promise }: Props) {
  if (!enabled) return <p>Disabled</p>;
  const data = use(promise);  // ✅ conditional OK
  return <div>{data}</div>;
}

// 3. use(context) — conditional context
const ThemeContext = createContext("light");

function Component({ flag }: { flag: boolean }) {
  if (!flag) return null;
  const theme = use(ThemeContext);  // ✅ conditional OK
  return <div>{theme}</div>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal:**

```typescript
function use<T>(thenable: Thenable<T> | Context<T>): T {
  if (isContext(thenable)) {
    return readContext(thenable);  // direct context read
  }
  if (isThenable(thenable)) {
    return readThenable(thenable);  // suspense integration
  }
  throw new Error("Invalid use() argument");
}

function readThenable<T>(thenable: Thenable<T>): T {
  if (thenable.status === "fulfilled") {
    return thenable.value;
  }
  if (thenable.status === "rejected") {
    throw thenable.reason;
  }
  if (thenable.status === "pending") {
    throw thenable;  // Suspense catches
  }
  // Unknown — track and trigger Suspense
  thenable.then(
    (value) => { thenable.status = "fulfilled"; thenable.value = value; },
    (error) => { thenable.status = "rejected"; thenable.reason = error; }
  );
  throw thenable;
}
```

**Rules of `use()`:**

```tsx
// ✅ Inside component
function Comp() {
  const data = use(promise);
}

// ✅ Inside custom hook
function useData() {
  return use(promise);
}

// ✅ Conditional / loop
function Comp({ enabled }) {
  if (enabled) {
    const data = use(promise);  // OK
  }
}

// ❌ Inside event handler
function Comp() {
  const handler = () => {
    const data = use(promise);  // ❌ error
  };
}

// ❌ Inside useEffect
function Comp() {
  useEffect(() => {
    const data = use(promise);  // ❌ error
  });
}
```

`use()` only in render (component or custom hook body).

**Why use() vs useContext for context:**

```tsx
// useContext — top-level only
function Comp({ flag }) {
  const theme = useContext(ThemeContext);  // top-level
  if (!flag) return null;
  return <div>{theme}</div>;
}

// use(context) — conditional
function Comp({ flag }) {
  if (!flag) return null;
  const theme = use(ThemeContext);  // OK
  return <div>{theme}</div>;
}
```

`use()` doesn't require top-level (no fiber state).

**Promise patterns:**

```tsx
// Pattern 1: Pass promise as prop
async function getUser(id: string) {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

function App({ id }: { id: string }) {
  const userPromise = useMemo(() => getUser(id), [id]);

  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}

function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}
```

**Pattern 2: Server Components:**

```tsx
// Server Component (RSC)
async function ProductsPage() {
  const productsPromise = db.products.findMany();  // promise

  return (
    <Suspense fallback={<Spinner />}>
      <ProductList productsPromise={productsPromise} />
    </Suspense>
  );
}

function ProductList({ productsPromise }: { productsPromise: Promise<Product[]> }) {
  const products = use(productsPromise);  // suspend until resolved
  return <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

**Pattern 3: Cache and re-use:**

```tsx
// Promise must be cached — re-creating throws different promise each render

// ❌ Bad
function Comp() {
  const promise = fetchData();  // new each render → suspends infinitely
  const data = use(promise);
}

// ✅ Cached
function Comp() {
  const promise = useMemo(() => fetchData(), []);
  const data = use(promise);
}

// ✅ Or external cache
const dataCache = new Map();
function getCached(key: string) {
  if (!dataCache.has(key)) {
    dataCache.set(key, fetchData(key));
  }
  return dataCache.get(key);
}

function Comp({ key }: { key: string }) {
  const data = use(getCached(key));
}

// ✅ React cache (RSC) — request-scoped deduplication
import { cache } from "react";
const fetchUser = cache(async (id: string) => {
  return fetch(`/api/users/${id}`).then(r => r.json());
});

function ServerView() {
  const userPromise = fetchUser("123");
  const user = use(userPromise);  // same request → cached
  return <p>{user.name}</p>;
}
```

**Comparison with other patterns:**

| Pattern | API | Pros | Cons |
|---------|-----|------|------|
| `use(promise)` | R19 native | Suspense integration | Manual cache |
| `useEffect + setState` | Pre-R18 | Familiar | Race conditions |
| TanStack Query | Library | Cache, retry, dedupe | External dep |
| RSC `await` | Server | Native server async | Server only |

**use() with throwable:**

```tsx
// Suspense for promises
const promise = fetchData();
// Status: "pending" — throws promise (Suspense catches)
// Status: "fulfilled" — returns value
// Status: "rejected" — throws error (ErrorBoundary catches)
```

**use() for conditional sub-data:**

```tsx
function Profile({ userId, showStats }: Props) {
  const user = use(userPromise);

  if (!showStats) return <div>{user.name}</div>;

  // Conditional use() — fetch stats only if needed
  const stats = use(statsPromise);
  return (
    <div>
      <p>{user.name}</p>
      <Stats data={stats} />
    </div>
  );
}
```

**use() Promise tracking:**

```typescript
// React tracks thenable state
const thenable = {
  status: "pending",  // | "fulfilled" | "rejected"
  value?: T,
  reason?: any,
};

// React modifies thenable directly
thenable.then(
  v => { thenable.status = "fulfilled"; thenable.value = v; },
  r => { thenable.status = "rejected"; thenable.reason = r; }
);
```

**Migration from useEffect + setState:**

```tsx
// Old
function User({ id }: { id: string }) {
  const [user, setUser] = useState<User | null>(null);
  useEffect(() => {
    fetchUser(id).then(setUser);
  }, [id]);

  if (!user) return <Spinner />;
  return <div>{user.name}</div>;
}

// New (R19)
function User({ id }: { id: string }) {
  const userPromise = useMemo(() => fetchUser(id), [id]);
  const user = use(userPromise);
  return <div>{user.name}</div>;
}

// Wrapped in Suspense:
<Suspense fallback={<Spinner />}>
  <User id="123" />
</Suspense>
```

**Limitations:**

- Promise must be referentially stable (memoize/cache)
- Error boundary needed for rejection handling
- Suspense boundary required

</details>

### Edge Cases

- **`use()` outside Suspense**: Promise rejection → app crash. Wrap in error boundary.
- **`use()` in useEffect**: Error. Hook context required.
- **Re-creating promise**: Suspense triggers each render. Cache promise.

### Follow-up savollar

- "use() vs await?" — `await` async function only (useEffect can't be async). `use()` works in render.
- "use() in Server Components?" — Yes. Both server and client components support.
- "Replace TanStack Query?" — No. `use()` is primitive. Library handles caching, retry, dedupe.

</details>

---

### 51. `useFormStatus` + `useActionState` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**R19 form hooks**: (1) **`useFormStatus`** — `<form>` ichida ishlatiladi, current submission state'ni qaytaradi (`pending`, `data`, `method`, `action`), (2) **`useActionState(action, initialState)`** — form action state management — `[state, dispatch, isPending]` qaytaradi (avval `useFormState` deyilardi, R19'da rename). `<form action={fn}>` (R19) bilan integration — server actions yoki client functions.

### Kod misoli

```tsx
// 1. useFormStatus — form holatini bilish
import { useFormStatus } from "react-dom";

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? "Submitting..." : "Submit"}
    </button>
  );
}

function ContactForm() {
  return (
    <form action={async (formData) => {
      await sendContact(formData);
    }}>
      <input name="name" />
      <input name="email" />
      <SubmitButton />  {/* knows form's pending status */}
    </form>
  );
}

// 2. useActionState — action result management
import { useActionState } from "react";

interface FormState {
  message: string;
  success: boolean;
}

async function submitAction(prevState: FormState, formData: FormData): Promise<FormState> {
  try {
    const name = formData.get("name") as string;
    await api.submit({ name });
    return { message: "Success!", success: true };
  } catch (error) {
    return { message: "Error: " + error.message, success: false };
  }
}

function MyForm() {
  const [state, formAction, isPending] = useActionState(submitAction, {
    message: "",
    success: false,
  });

  return (
    <form action={formAction}>
      <input name="name" />
      <button type="submit" disabled={isPending}>Submit</button>
      {state.message && (
        <p style={{ color: state.success ? "green" : "red" }}>
          {state.message}
        </p>
      )}
    </form>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`useFormStatus` API:**

```typescript
function useFormStatus(): {
  pending: boolean;
  data: FormData | null;
  method: "get" | "post" | null;
  action: ((formData: FormData) => void | Promise<void>) | string | null;
};
```

**Constraint:** Must be called in component **inside `<form>`**:

```tsx
// ❌ Outside form — always pending=false
function Page() {
  const { pending } = useFormStatus();  // useless
  return <form><input /></form>;
}

// ✅ Inside form
function Page() {
  return (
    <form>
      <input />
      <SubmitButton />  {/* useFormStatus inside */}
    </form>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Submit</button>;
}
```

**`<form action={fn}>` (R19):**

```tsx
// R19 — function as action
<form action={async (formData) => {
  // formData = FormData object
  await api.submit(formData);
}}>
  <input name="email" />
  <button type="submit">Submit</button>
</form>

// On submit:
// 1. Browser default form submission prevented
// 2. action function called with FormData
// 3. Form fields submitted (uncontrolled style)
```

**`useActionState` mechanism:**

```typescript
function useActionState<S, P>(
  action: (state: S, payload: P) => Promise<S> | S,
  initialState: S
): [S, (payload: P) => void, boolean] {
  const [state, setState] = useState(initialState);
  const [isPending, setIsPending] = useState(false);

  const formAction = useCallback(async (payload: P) => {
    setIsPending(true);
    try {
      const newState = await action(state, payload);
      setState(newState);
    } finally {
      setIsPending(false);
    }
  }, [state, action]);

  return [state, formAction, isPending];
}
```

(Simplified — actual implementation uses transitions internally)

**Server Actions integration:**

```tsx
// app/actions.ts
"use server";

export async function submitContact(prevState: State, formData: FormData) {
  const name = formData.get("name") as string;
  await db.contacts.create({ data: { name } });
  return { message: "Saved!" };
}

// Client component
"use client";
import { useActionState } from "react";
import { submitContact } from "./actions";

function ContactForm() {
  const [state, action, isPending] = useActionState(submitContact, { message: "" });

  return (
    <form action={action}>
      <input name="name" />
      <button disabled={isPending}>Save</button>
      {state.message && <p>{state.message}</p>}
    </form>
  );
}

// Submit:
// 1. action(state, formData) → fetch to server
// 2. Server runs submitContact
// 3. Returns new state
// 4. Client updates with new state
```

**Migration from `useFormState` → `useActionState`:**

```tsx
// R18 (Canary) — useFormState
import { useFormState } from "react-dom";
const [state, action] = useFormState(submitAction, initial);

// R19 — rename to useActionState (moved to "react", isPending qo'shildi)
import { useActionState } from "react";
const [state, action, isPending] = useActionState(submitAction, initial);
// Yangi: isPending tuple'ning 3-elementi
// Yangi: react'dan import (react-dom'dan emas)
```

**vs traditional onSubmit:**

```tsx
// Old pattern
function Form() {
  const [pending, setPending] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setPending(true);
    setError(null);
    try {
      const formData = new FormData(e.currentTarget);
      await submit(formData);
    } catch (err) {
      setError(err.message);
    } finally {
      setPending(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" />
      <button disabled={pending}>Submit</button>
      {error && <p>{error}</p>}
    </form>
  );
}

// R19 pattern — less boilerplate
function Form() {
  const [error, action, isPending] = useActionState(
    async (_, formData) => {
      try {
        await submit(formData);
        return null;
      } catch (err) {
        return err.message;
      }
    },
    null
  );

  return (
    <form action={action}>
      <input name="email" />
      <button disabled={isPending}>Submit</button>
      {error && <p>{error}</p>}
    </form>
  );
}
```

**Form reset behavior:**

```tsx
// R19 form action — uncontrolled inputs auto-reset on success
<form action={async (formData) => {
  await submit(formData);
  // Form fields reset to defaults
}}>
  <input name="message" />
  <button>Send</button>
</form>

// To prevent reset:
<form action={async (formData) => {
  // ... action logic
  return { reset: false };  // not standard, depends on framework
}}>
```

**`useFormStatus` for nested components:**

```tsx
function Form() {
  return (
    <form action={submitAction}>
      <NestedField />
      <SubmitButton />
    </form>
  );
}

function NestedField() {
  const { pending } = useFormStatus();
  return <input disabled={pending} />;
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Submit</button>;
}

// Both NestedField and SubmitButton see pending state
// useFormStatus accesses parent <form> via context
```

</details>

### Edge Cases

- **`useFormStatus` outside form**: Returns `{ pending: false, data: null, ... }`.
- **Multiple forms**: `useFormStatus` reads nearest `<form>` ancestor.
- **`useActionState` action errors**: Throws — error boundary handles.

### Follow-up savollar

- "Server vs client action?" — Function with `"use server"` — server. Plain function — client.
- "Progressive enhancement?" — `<form action={fn}>` works without JS (server actions). Client actions require JS.
- "TanStack Form integration?" — Different paradigm. R19 hooks for simple forms; TanStack Form for complex.

</details>

---

### 52. `useOptimistic` — optimistic UI [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useOptimistic(state, updateFn)`** — server response kutmasdan UI'ni optimistic ravishda yangilaydi. Action davomida optimistic state ko'rsatiladi. Action complete bo'lganda optimistic state real state bilan almashtiriladi. Failure bo'lsa — automatic rollback (real state qaytadi). Use case: "Like" button, optimistic todo add, message send.

### Kod misoli

```tsx
import { useOptimistic, useState } from "react";

interface Message {
  id: string;
  text: string;
  sending?: boolean;
}

function Chat() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [optimisticMessages, addOptimistic] = useOptimistic<Message[], string>(
    messages,
    (state, newText) => [
      ...state,
      { id: `temp-${Date.now()}`, text: newText, sending: true },
    ]
  );

  const sendMessage = async (formData: FormData) => {
    const text = formData.get("text") as string;
    addOptimistic(text);  // immediate UI update

    const realMessage = await api.sendMessage(text);
    setMessages(prev => [...prev, realMessage]);  // commit
  };

  return (
    <>
      <form action={sendMessage}>
        <input name="text" />
        <button>Send</button>
      </form>
      <ul>
        {optimisticMessages.map(msg => (
          <li key={msg.id} style={{ opacity: msg.sending ? 0.5 : 1 }}>
            {msg.text}
            {msg.sending && " (sending...)"}
          </li>
        ))}
      </ul>
    </>
  );
}

// Send "Hello":
// 1. addOptimistic("Hello") → optimisticMessages immediately includes it (with sending: true)
// 2. UI shows "Hello (sending...)" with reduced opacity
// 3. API request runs
// 4. Response: real message
// 5. setMessages commits real
// 6. Optimistic state matches real (no longer sending)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**API signature:**

```typescript
function useOptimistic<S, A>(
  state: S,
  updateFn: (currentState: S, optimisticValue: A) => S
): [S, (value: A) => void];
```

- `state` — actual state (truth)
- `updateFn` — derives optimistic state from action
- Returns: optimistic state + addOptimistic function

**Behavior:**

1. Initial: optimisticState === state
2. addOptimistic(value): updateFn(state, value) → new optimistic state
3. State updates: optimisticState reverts to state (real wins)
4. Multiple addOptimistic: chain updateFn calls

**Like button example:**

```tsx
function Post({ post }: { post: Post }) {
  const [likes, setLikes] = useState(post.likes);
  const [optimisticLikes, setOptimisticLikes] = useOptimistic<number, "like" | "unlike">(
    likes,
    (current, action) => action === "like" ? current + 1 : current - 1
  );

  const toggleLike = async () => {
    const action = post.userLiked ? "unlike" : "like";
    setOptimisticLikes(action);  // immediate UI
    const newLikes = await api.toggleLike(post.id);
    setLikes(newLikes);
  };

  return (
    <button onClick={toggleLike}>
      ❤ {optimisticLikes}
    </button>
  );
}
```

**With error handling:**

```tsx
function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [error, setError] = useState<string | null>(null);
  const [optimisticTodos, addOptimisticTodo] = useOptimistic<Todo[], Todo>(
    todos,
    (state, newTodo) => [...state, newTodo]
  );

  const addTodo = async (text: string) => {
    const tempId = `temp-${Date.now()}`;
    const optimisticTodo: Todo = { id: tempId, text, done: false };

    addOptimisticTodo(optimisticTodo);
    setError(null);

    try {
      const realTodo = await api.createTodo(text);
      setTodos(prev => [...prev, realTodo]);
    } catch (err) {
      setError("Failed to add. Please try again.");
      // optimistic state auto-reverts to real `todos`
    }
  };

  return (
    <>
      {error && <p style={{ color: "red" }}>{error}</p>}
      <ul>
        {optimisticTodos.map(t => (
          <li key={t.id}>{t.text}</li>
        ))}
      </ul>
    </>
  );
}
```

**Rollback on failure:**

When server returns error or no setState commits the optimistic value, optimistic state reverts to actual state automatically.

**Multiple optimistic updates:**

```tsx
const [state, addOptimistic] = useOptimistic(
  baseState,
  (current, update) => ({ ...current, ...update })
);

// Multiple actions
addOptimistic({ status: "saving" });  // state has saving=true
addOptimistic({ progress: 50 });       // state has saving=true, progress=50
// All applied sequentially via updateFn
```

**With useActionState:**

```tsx
function MessageForm() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [optimisticMessages, addOptimistic] = useOptimistic(messages, (state, text: string) => [
    ...state,
    { id: `temp-${Date.now()}`, text, sending: true },
  ]);

  const [error, action, isPending] = useActionState(
    async (_, formData: FormData) => {
      const text = formData.get("text") as string;
      addOptimistic(text);

      try {
        const newMsg = await api.sendMessage(text);
        setMessages(prev => [...prev, newMsg]);
        return null;
      } catch (err) {
        return err.message;
      }
    },
    null
  );

  return (
    <>
      <form action={action}>
        <input name="text" />
        <button disabled={isPending}>Send</button>
      </form>
      {error && <p>{error}</p>}
      <ul>{optimisticMessages.map(m => <li key={m.id}>{m.text}</li>)}</ul>
    </>
  );
}
```

**Server Actions + useOptimistic:**

```tsx
// app/actions.ts
"use server";
export async function addTodo(text: string) {
  return db.todo.create({ data: { text } });
}

// Client
"use client";
import { useOptimistic } from "react";
import { addTodo } from "./actions";

function TodoList({ initialTodos }: { initialTodos: Todo[] }) {
  const [optimisticTodos, addOptimistic] = useOptimistic(
    initialTodos,
    (state, text: string) => [...state, { id: "temp", text, sending: true }]
  );

  return (
    <>
      <form action={async (formData) => {
        const text = formData.get("text") as string;
        addOptimistic(text);
        await addTodo(text);
      }}>
        <input name="text" />
        <button>Add</button>
      </form>
      <ul>{optimisticTodos.map(t => <li key={t.id}>{t.text}</li>)}</ul>
    </>
  );
}
```

**Transition context:**

`useOptimistic`'ning rollback semantics — optimistic state real state'ga "qaytadi" — transition tugaganda automatic. Form action ichida yoki `startTransition` ichida chaqirilishi tavsiya etiladi:

```tsx
// ✅ Form action ichida — transition automatic
<form action={async (formData) => {
  addOptimistic(formData.get("text"));
  await api.call(formData);
  // Action tugagandan keyin: optimistic discarded, real state ko'rinadi
}} />

// ✅ startTransition ichida
const handleClick = () => {
  startTransition(async () => {
    addOptimistic(value);
    await api.call(value);
  });
};
```

**Rollback mexanizmi:** `useOptimistic` yangi state slot saqlamaydi — render time'da `state` argument'iga `updateFn` qo'llanadi. Action tugaganda re-render'da `state` o'zgaradi (yoki o'zgarmaydi), va optimistic accumulated updates natural ravishda discard qilinadi.

**Real-world UX:**

Without optimistic UI:
```
Click → loading state → network round-trip → response → UI updates
```

With optimistic UI:
```
Click → UI updates immediately → response → UI confirms (or rollback on error)
```

User-perceived latency: instant. Actual network latency hidden by speculative UI.

**Edge cases:**

```tsx
// Multiple rapid actions
addOptimistic("first");
addOptimistic("second");
// Both applied via updateFn

// Server error + optimistic
addOptimistic(value);
try {
  await api.call();
} catch (e) {
  // optimistic reverts when no setState commits
}

// Stale closure in updateFn
const [state, add] = useOptimistic(
  s,
  (current, action) => {
    // current — latest optimistic state
    // action — value passed to add()
    // No stale issue — React passes current
  }
);
```

</details>

### Edge Cases

- **Calling outside transition**: Warning, may not apply. Use inside form action or startTransition.
- **Network failure**: Optimistic reverts. UX: show error message.
- **Concurrent optimistic updates**: All applied via updateFn chain.

### Follow-up savollar

- "useOptimistic vs manual state management?" — Auto rollback simpler. Manual gives more control.
- "Animation during transition?" — Possible. Add `transitioning` flag in optimistic state.
- "Server returns different shape?" — Reconciler diffs. Optimistic shape should match server shape closely.

</details>

---

### 53. R19 hooks decision tree [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**R19 form/action hooks**:
- **`use(promise)`** — Suspense-based async data fetching, conditional OK
- **`use(context)`** — `useContext` alternative, conditional OK
- **`useFormStatus`** — `<form>` ichida current submission state
- **`useActionState`** — form action state management (replaces `useFormState`)
- **`useOptimistic`** — optimistic UI updates with auto-rollback

### Decision tree

```
Need async data?
├── In Server Component → await directly
├── In Client Component
│   ├── With Suspense → use(promise)
│   └── Without Suspense → useEffect + useState (or library)

Building form?
├── Need submission state UI → useFormStatus (in submit button)
├── Need form result/error state → useActionState
├── Need optimistic UI → useOptimistic

Reading context?
├── Top-level → useContext
└── Conditional → use(context)
```

### Kod misoli

```tsx
// Combined R19 form pattern
"use client";
import { useActionState, useOptimistic, useFormStatus } from "react";

interface Comment {
  id: string;
  text: string;
  author: string;
}

interface FormState {
  error: string | null;
  resetKey: number;
}

async function submitComment(prevState: FormState, formData: FormData): Promise<FormState> {
  const text = formData.get("text") as string;
  if (!text || text.length < 3) {
    return { error: "Comment too short", resetKey: prevState.resetKey };
  }
  try {
    await api.createComment({ text });
    return { error: null, resetKey: prevState.resetKey + 1 };
  } catch (err) {
    return { error: err.message, resetKey: prevState.resetKey };
  }
}

function CommentForm({ initialComments }: { initialComments: Comment[] }) {
  // Form state with useActionState
  const [formState, formAction, isPending] = useActionState(submitComment, {
    error: null,
    resetKey: 0,
  });

  // Optimistic comments
  const [comments, setComments] = useState(initialComments);
  const [optimisticComments, addOptimistic] = useOptimistic<Comment[], string>(
    comments,
    (state, text) => [
      ...state,
      { id: `temp-${Date.now()}`, text, author: "You" },
    ]
  );

  return (
    <>
      <form action={async (formData) => {
        addOptimistic(formData.get("text") as string);
        await formAction(formData);
      }}>
        <input name="text" key={formState.resetKey} />
        <SubmitButton />  {/* useFormStatus inside */}
      </form>
      {formState.error && <p style={{ color: "red" }}>{formState.error}</p>}
      <ul>
        {optimisticComments.map(c => (
          <li key={c.id}>{c.author}: {c.text}</li>
        ))}
      </ul>
    </>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button type="submit" disabled={pending}>Post</button>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Hook combinations:**

| Use case | Hooks |
|----------|-------|
| Simple form | `useActionState` |
| Form with submit UI | `useActionState` + `useFormStatus` |
| Form with optimistic UI | `useActionState` + `useOptimistic` |
| Async data | `use(promise)` + `<Suspense>` |
| Context (conditional) | `use(context)` |

**Pre-R19 vs R19 comparison:**

```tsx
// Pre-R19 — form with manual state
function Form() {
  const [pending, setPending] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setPending(true);
    try {
      const data = new FormData(e.currentTarget);
      await submit(data);
    } catch (err) {
      setError(err.message);
    } finally {
      setPending(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="x" />
      <button disabled={pending}>Submit</button>
      {error && <p>{error}</p>}
    </form>
  );
}

// R19 — concise
function Form() {
  const [error, action, isPending] = useActionState(
    async (_, formData) => {
      try {
        await submit(formData);
        return null;
      } catch (e) {
        return e.message;
      }
    },
    null
  );

  return (
    <form action={action}>
      <input name="x" />
      <button disabled={isPending}>Submit</button>
      {error && <p>{error}</p>}
    </form>
  );
}
```

**Server Component + Client Component:**

```tsx
// app/page.tsx (Server Component)
async function ProductsPage() {
  const products = await db.products.findMany();
  return <ProductList initialProducts={products} />;
}

// app/ProductList.tsx (Client Component)
"use client";
function ProductList({ initialProducts }: { initialProducts: Product[] }) {
  const [optimisticProducts, addOptimistic] = useOptimistic(initialProducts, /* ... */);
  // R19 hooks here
}
```

**Migration strategy:**

1. R18'dagi `useFormState` (`react-dom`'dan) → R19 `useActionState` (`react`'dan, rename) — yangi `isPending` qaytaradi
2. For new forms — try `<form action>` + R19 hooks
3. Existing onSubmit forms — gradually migrate
4. Optimistic UIs — add `useOptimistic` where helpful
5. Async data fetching — `use(promise)` for Suspense-style

**`useActionState` form reset:**

```tsx
const [state, action] = useActionState(submit, initial);

// On success, form fields auto-reset (uncontrolled)
// To preserve values, control inputs:
<input name="text" defaultValue={state.text} />
```

**Browser fallback:**

```tsx
// Without JS (progressive enhancement)
<form action="/api/submit" method="post">
  <input name="text" />
  <button>Submit</button>
</form>
// Browser default form submission, no React

// With JS — `action={fn}` overrides
<form action={async (formData) => {
  await submit(formData);
}}>
```

R19 — both work, JS enhances.

**Performance:**

- `useFormStatus` — lightweight (just reads context)
- `useActionState` — wraps useReducer + useTransition (small overhead)
- `useOptimistic` — useState + cleanup logic
- All comparable to manual implementations

**Errors and exceptions:**

```tsx
// Server action throws → useActionState's action throws → error boundary catches
// Or handle in action:
async function action(prevState, formData) {
  try {
    await submit(formData);
    return { ok: true };
  } catch (err) {
    return { ok: false, error: err.message };
  }
}
```

</details>

### Edge Cases

- **`useActionState` reset**: Set new initial via key prop or component remount.
- **`useOptimistic` outside transition**: Limited effect.
- **`useFormStatus` no parent form**: Returns default (`pending: false`).

### Follow-up savollar

- "Use TanStack Form / React Hook Form?" — Complex forms — yes. Simple R19 hooks sufficient for basic cases.
- "Server actions vs API routes?" — Server actions: tighter integration, type safety. API routes: standard REST, framework-agnostic.
- "When use() vs Suspense?" — Always together. `use()` triggers Suspense; Suspense provides fallback.

</details>

---

<a id="qism-k"></a>

## QISM K: Custom Hooks

### 54. Custom hooks pattern va composition [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Custom hook** — `use*` prefix bilan funksiya, ichida boshqa hooks chaqiradi. Maqsad: **logic reuse** komponentlar orasida (HOC/render props alternative). Custom hook **state share qilmaydi** — har chaqiruv yangi instance state. Composition: bir custom hook boshqa hooks'ni o'z ichiga oladi (built-in yoki custom).

### Kod misoli

```tsx
// 1. Basic custom hook
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = useCallback(() => setCount(c => c + 1), []);
  const decrement = useCallback(() => setCount(c => c - 1), []);
  const reset = useCallback(() => setCount(initial), [initial]);
  return { count, increment, decrement, reset };
}

// Usage
function Component() {
  const { count, increment, reset } = useCounter(0);
  return (
    <>
      <p>{count}</p>
      <button onClick={increment}>+</button>
      <button onClick={reset}>Reset</button>
    </>
  );
}

// Two instances — separate state
function App() {
  return (
    <>
      <Component />  {/* count A */}
      <Component />  {/* count B (independent) */}
    </>
  );
}

// 2. Composition — custom hook using custom hook
function useUserData(userId: string) {
  const fetcher = useCallback(() => api.getUser(userId), [userId]);
  const { data, loading, error } = useFetch(fetcher);
  const formattedName = useMemo(() => data?.name?.toUpperCase(), [data]);

  return { user: data, formattedName, loading, error };
}

function useFetch<T>(fetcher: () => Promise<T>) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    fetcher()
      .then(d => { if (!cancelled) setData(d); })
      .catch(e => { if (!cancelled) setError(e); })
      .finally(() => { if (!cancelled) setLoading(false); });
    return () => { cancelled = true; };
  }, [fetcher]);

  return { data, loading, error };
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why `use*` prefix:**

1. ESLint rule detects (rules-of-hooks)
2. Convention — readers know it's a hook
3. React DevTools displays as hook

```tsx
// ✅ Detected as hook
function useFoo() {
  const [x] = useState(0);
  return x;
}

// ❌ Not detected — ESLint may not lint properly
function getFoo() {
  const [x] = useState(0);  // ⚠️ Rules of Hooks may not catch
  return x;
}
```

**State per instance:**

```tsx
function useCounter() {
  const [count, setCount] = useState(0);
  return [count, setCount];
}

function CompA() {
  const [count] = useCounter();  // CompA's own count
}

function CompB() {
  const [count] = useCounter();  // CompB's own count, independent
}

// Counters separate — useState per fiber, custom hook just calls useState
```

**Custom hook returns:**

```tsx
// Tuple — like useState
function useCounter() {
  const [count, setCount] = useState(0);
  return [count, setCount] as const;
}

// Object — named API
function useCounter() {
  const [count, setCount] = useState(0);
  return { count, setCount };
}

// Single value
function useDoubled(value: number) {
  return useMemo(() => value * 2, [value]);
}
```

Object usually preferred for clarity (multi-property).

**Logic reuse — custom hook benefits:**

```tsx
// Pre-Hooks (HOC) — wrapper hell, prop collision
const withCounter = (Component) => {
  return class extends React.Component {
    state = { count: 0 };
    increment = () => this.setState({ count: this.state.count + 1 });
    render() {
      return <Component {...this.props} count={this.state.count} increment={this.increment} />;
    }
  };
};

const EnhancedButton = withCounter(MyButton);
// Multiple HOCs: withAuth(withTheme(withCounter(MyButton)))
// Wrapper hell, prop collision (counter prop name)

// Custom hooks — clean composition
function MyButton() {
  const { count, increment } = useCounter();
  const auth = useAuth();
  const theme = useTheme();
  // No wrappers, named state
}
```

**Common custom hooks:**

```tsx
// useToggle
function useToggle(initial = false): [boolean, () => void] {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle];
}

// useLocalStorage
function useLocalStorage<T>(key: string, initial: T): [T, (v: T) => void] {
  const [value, setValue] = useState<T>(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : initial;
    } catch {
      return initial;
    }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

// useDebounce
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}

// usePrevious
function usePrevious<T>(value: T): T | undefined {
  // R19: zero-arg `useRef<T>()` olib tashlandi — initial value MAJBURIY
  const ref = useRef<T | undefined>(undefined);
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}

// useEventListener
function useEventListener<K extends keyof WindowEventMap>(
  event: K,
  handler: (e: WindowEventMap[K]) => void,
  element: Window | HTMLElement = window
) {
  const handlerRef = useRef(handler);
  useEffect(() => { handlerRef.current = handler; }, [handler]);

  useEffect(() => {
    const wrapper = (e: any) => handlerRef.current(e);
    element.addEventListener(event, wrapper);
    return () => element.removeEventListener(event, wrapper);
  }, [event, element]);
}
```

**Custom hook tests:**

```tsx
// renderHook from React Testing Library
import { renderHook, act } from "@testing-library/react";

test("useCounter", () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);

  act(() => {
    result.current.increment();
  });

  expect(result.current.count).toBe(1);
});
```

**Composition deeply:**

```tsx
function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  // ... auth logic
  return { user, login, logout };
}

function useFetchAuthorized(url: string) {
  const { user } = useAuth();  // composition
  const fetcher = useCallback(async () => {
    if (!user) throw new Error("Not authenticated");
    return fetch(url, {
      headers: { Authorization: `Bearer ${user.token}` }
    }).then(r => r.json());
  }, [url, user]);

  return useFetch(fetcher);  // composition
}

function MyComponent() {
  const { data } = useFetchAuthorized("/api/profile");
  // ...
}
```

**Custom hook return stability:**

```tsx
function useStableData() {
  const [data, setData] = useState({ x: 1 });
  return data;  // ⚠️ if data changes, returns new ref
}

// Stabilize:
function useStableData() {
  const [data, setData] = useState({ x: 1 });
  return useMemo(() => data, [data.x]);  // memoize based on important field
}
```

**Custom hook with reducer:**

```tsx
function useUndoRedo<T>(initial: T) {
  const [state, dispatch] = useReducer(reducer, {
    past: [],
    present: initial,
    future: [],
  });

  return {
    state: state.present,
    canUndo: state.past.length > 0,
    canRedo: state.future.length > 0,
    set: (value: T) => dispatch({ type: "SET", payload: value }),
    undo: () => dispatch({ type: "UNDO" }),
    redo: () => dispatch({ type: "REDO" }),
  };
}
```

**Performance considerations:**

- Custom hooks have no inherent overhead beyond hooks they call
- Internal `useCallback`/`useMemo` for return values stable across renders
- Don't over-optimize — only stabilize when consumers benefit

</details>

### Edge Cases

- **Custom hook in non-PascalCase function**: ESLint may not catch hooks rules.
- **Returning arrays vs objects**: Arrays good for tuple semantics (`[value, setter]`). Objects for many properties.
- **State sharing**: Custom hooks DO NOT share state. Each call separate state.

### Follow-up savollar

- "Custom hook share state across components?" — No. State per component instance. For shared state — Context or external store.
- "ESLint exhaustive-deps for custom hooks?" — Yes — uses fetcher etc. checked.
- "Test custom hooks?" — `renderHook` from RTL. Or wrap in test component.

</details>

---

### 55. `useDebugValue` — DevTools annotation [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useDebugValue(value, format?)`** — React DevTools'da custom hook'ning ko'rinishini boyitish uchun. Faqat custom hook ichida ishlatiladi. DevTools "Components" panelida hook nomi yonida shu value ko'rinadi. Production'da no-op (only dev). `format` function — expensive formatting uchun (faqat DevTools open bo'lganda chaqiriladi).

### Kod misoli

```tsx
import { useState, useEffect, useDebugValue } from "react";

// Without useDebugValue
function useOnlineStatus() {
  const [online, setOnline] = useState(navigator.onLine);
  // DevTools: useOnlineStatus → State: true
}

// With useDebugValue
function useOnlineStatus() {
  const [online, setOnline] = useState(navigator.onLine);

  useEffect(() => {
    const goOnline = () => setOnline(true);
    const goOffline = () => setOnline(false);
    window.addEventListener("online", goOnline);
    window.addEventListener("offline", goOffline);
    return () => {
      window.removeEventListener("online", goOnline);
      window.removeEventListener("offline", goOffline);
    };
  }, []);

  useDebugValue(online ? "Online ✓" : "Offline ✗");

  return online;
}

// DevTools: useOnlineStatus: "Online ✓"
```

```tsx
// Format function — for expensive
function useDate(date: Date) {
  useDebugValue(date, (d) => d.toISOString());
  // d.toISOString() only called when DevTools open
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**API:**

```typescript
function useDebugValue<T>(value: T, format?: (value: T) => any): void;
```

**Why format function:**

```tsx
// Expensive computation always
useDebugValue(largeObject, JSON.stringify);
// JSON.stringify always runs

// Skipped in production / DevTools closed
useDebugValue(largeObject, (obj) => JSON.stringify(obj));
// stringify only when DevTools needs the value
```

`format` callback called only when DevTools is open and inspecting the component.

**Use case:**

```tsx
function useUser(id: string) {
  const [user, setUser] = useState<User | null>(null);
  // ...
  useDebugValue(user, (u) => u ? `${u.name} (${u.role})` : "Loading");
  return user;
}
```

DevTools shows useful summary instead of `{ id: "...", name: "...", ... }`.

**Multiple hooks:**

```tsx
function useComplex() {
  const [a] = useState(...);
  useDebugValue(a, (v) => `A: ${v}`);

  const [b] = useState(...);
  useDebugValue(b, (v) => `B: ${v}`);

  // Each useDebugValue separate display
}
```

**Production:**

```tsx
// Production: useDebugValue no-op
function useFoo() {
  useDebugValue("debug info");  // not executed in prod
  return useState(0);
}
```

**`React.memo` + useDebugValue:**

Custom hooks inside memoized components — useDebugValue still works in dev.

**Limited use:**

`useDebugValue` is **rarely needed**. Most custom hooks self-explanatory. Use only for:

- Complex internal state (e.g., subscription status, queue length)
- Expensive computation (with format function)
- Library hooks (give consumers visibility)

**DevTools display:**

```
Components panel:
  MyComponent
    useOnlineStatus: "Online ✓"
    useState: true
    useEffect
```

**Performance:**

- Dev: minor overhead (format call)
- Production: zero (stripped)

**Best practice — library hooks:**

```tsx
// React Query, SWR, etc. expose useDebugValue
import { useQuery } from "@tanstack/react-query";

const { data, status } = useQuery(...);
// React Query internally:
// useDebugValue({ status, data }, (s) => `${s.status}: ${s.data ? "loaded" : "loading"}`);
// → DevTools shows query state
```

</details>

### Edge Cases

- **`useDebugValue` outside custom hook**: No-op (no React error).
- **`useDebugValue` with undefined**: Shows nothing.
- **Format function throws**: DevTools may show error.

### Follow-up savollar

- "Always add useDebugValue?" — No. Only when value not obvious.
- "DevTools required?" — Yes, for visibility. Not affecting runtime.
- "Custom hooks performance?" — Negligible.

</details>

---

### 56. Common custom hooks — useDebounce, useLocalStorage, useIntersectionObserver, useFetch [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Real-world apps'da ko'p ishlatiladigan custom hooks: **`useDebounce`** (debounce value), **`useLocalStorage`** (localStorage sync), **`useIntersectionObserver`** (lazy load, infinite scroll), **`useFetch`** (data fetching), **`useEventListener`** (DOM event), **`useMediaQuery`** (responsive), **`useOnClickOutside`** (modal close).

### Kod misoli

```tsx
// 1. useDebounce — delays value
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);

  return debounced;
}

// Usage
function Search() {
  const [query, setQuery] = useState("");
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (debouncedQuery) {
      api.search(debouncedQuery).then(setResults);
    }
  }, [debouncedQuery]);

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

```tsx
// 2. useLocalStorage — persist to localStorage
function useLocalStorage<T>(key: string, initial: T): [T, (v: T | ((prev: T) => T)) => void] {
  const [value, setValue] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initial;
    } catch {
      return initial;
    }
  });

  const setStoredValue = useCallback((next: T | ((prev: T) => T)) => {
    setValue((prev) => {
      const newValue = typeof next === "function" ? (next as Function)(prev) : next;
      try {
        localStorage.setItem(key, JSON.stringify(newValue));
      } catch {
        // ignore quota errors
      }
      return newValue;
    });
  }, [key]);

  // Sync across tabs
  useEffect(() => {
    const handler = (e: StorageEvent) => {
      if (e.key === key && e.newValue !== null) {
        setValue(JSON.parse(e.newValue));
      }
    };
    window.addEventListener("storage", handler);
    return () => window.removeEventListener("storage", handler);
  }, [key]);

  return [value, setStoredValue];
}

// Usage
const [theme, setTheme] = useLocalStorage<"light" | "dark">("theme", "light");
```

```tsx
// 3. useIntersectionObserver — element visibility
function useIntersectionObserver(
  ref: React.RefObject<Element>,
  options: IntersectionObserverInit = {}
): boolean {
  const [isIntersecting, setIsIntersecting] = useState(false);

  useEffect(() => {
    if (!ref.current) return;
    const observer = new IntersectionObserver(
      ([entry]) => setIsIntersecting(entry.isIntersecting),
      options
    );
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, [ref, options.threshold, options.root, options.rootMargin]);

  return isIntersecting;
}

// Usage — lazy load image
function LazyImage({ src }: { src: string }) {
  const ref = useRef<HTMLImageElement>(null);
  const isVisible = useIntersectionObserver(ref, { threshold: 0.1 });

  return (
    <img
      ref={ref}
      src={isVisible ? src : "/placeholder.png"}
      alt=""
    />
  );
}
```

```tsx
// 4. useFetch — basic fetch
function useFetch<T>(url: string | null): {
  data: T | null;
  loading: boolean;
  error: Error | null;
} {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    if (!url) return;

    let cancelled = false;
    const controller = new AbortController();
    setLoading(true);
    setError(null);

    fetch(url, { signal: controller.signal })
      .then((r) => {
        if (!r.ok) throw new Error(r.statusText);
        return r.json();
      })
      .then((d) => {
        if (!cancelled) {
          setData(d);
          setLoading(false);
        }
      })
      .catch((err) => {
        if (cancelled || err.name === "AbortError") return;
        setError(err);
        setLoading(false);
      });

    return () => {
      cancelled = true;
      controller.abort();
    };
  }, [url]);

  return { data, loading, error };
}
```

```tsx
// 5. useEventListener
function useEventListener<K extends keyof WindowEventMap>(
  event: K,
  handler: (e: WindowEventMap[K]) => void,
  element: Window | HTMLElement = window
) {
  const handlerRef = useRef(handler);

  useEffect(() => {
    handlerRef.current = handler;
  }, [handler]);

  useEffect(() => {
    const listener = (e: any) => handlerRef.current(e);
    element.addEventListener(event, listener);
    return () => element.removeEventListener(event, listener);
  }, [event, element]);
}

// Usage
useEventListener("scroll", () => {
  console.log("scrolled", window.scrollY);
});

useEventListener("keydown", (e) => {
  if (e.key === "Escape") closeModal();
});
```

```tsx
// 6. useMediaQuery
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() => window.matchMedia(query).matches);

  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
    mediaQuery.addEventListener("change", handler);
    return () => mediaQuery.removeEventListener("change", handler);
  }, [query]);

  return matches;
}

// Usage
const isMobile = useMediaQuery("(max-width: 768px)");
const prefersDark = useMediaQuery("(prefers-color-scheme: dark)");
```

```tsx
// 7. useOnClickOutside
function useOnClickOutside<T extends HTMLElement>(
  ref: React.RefObject<T>,
  handler: () => void
) {
  useEffect(() => {
    const listener = (event: MouseEvent | TouchEvent) => {
      if (!ref.current || ref.current.contains(event.target as Node)) {
        return;
      }
      handler();
    };

    document.addEventListener("mousedown", listener);
    document.addEventListener("touchstart", listener);
    return () => {
      document.removeEventListener("mousedown", listener);
      document.removeEventListener("touchstart", listener);
    };
  }, [ref, handler]);
}

// Usage
function Modal({ onClose }: { onClose: () => void }) {
  const ref = useRef<HTMLDivElement>(null);
  useOnClickOutside(ref, onClose);

  return <div ref={ref}>Modal content</div>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Best practices for custom hooks:**

1. **`use*` prefix** — ESLint detects, convention
2. **Stable callbacks** — `useCallback` for returns
3. **Cleanup** — every subscription, listener, timer
4. **TypeScript** — generics for flexibility
5. **Edge cases** — null guards, server-side, race conditions

**TypeScript generics:**

```tsx
function useFetch<T>(url: string): { data: T | null; ... } {
  // T flows through state and return
}

const { data: user } = useFetch<User>("/api/user");
```

**SSR-safe hooks:**

```tsx
function useWindowSize() {
  const [size, setSize] = useState({
    width: typeof window !== "undefined" ? window.innerWidth : 0,
    height: typeof window !== "undefined" ? window.innerHeight : 0,
  });

  useEffect(() => {
    const handler = () => setSize({ width: window.innerWidth, height: window.innerHeight });
    window.addEventListener("resize", handler);
    return () => window.removeEventListener("resize", handler);
  }, []);

  return size;
}
```

**Hook libraries:**

- **react-use** — 100+ hooks
- **usehooks-ts** — TypeScript-first
- **react-hook-form** — form state
- **TanStack Query** — data fetching
- **Zustand** — state management

**Custom hook tests:**

```tsx
import { renderHook, act } from "@testing-library/react";

test("useDebounce delays value", async () => {
  jest.useFakeTimers();
  const { result, rerender } = renderHook(
    ({ value }) => useDebounce(value, 300),
    { initialProps: { value: "a" } }
  );

  expect(result.current).toBe("a");

  rerender({ value: "b" });
  expect(result.current).toBe("a");  // not yet debounced

  act(() => {
    jest.advanceTimersByTime(300);
  });

  expect(result.current).toBe("b");
});
```

**Performance: avoid re-creating internal callbacks:**

```tsx
function useToggle() {
  const [value, setValue] = useState(false);

  // ❌ New function each render — caller's deps churn
  const toggle = () => setValue(v => !v);

  // ✅ Stable
  const toggle = useCallback(() => setValue(v => !v), []);

  return [value, toggle];
}
```

**Compose multiple hooks:**

```tsx
function useDebounceFetch<T>(url: string | null, delay = 300) {
  const debouncedUrl = useDebounce(url, delay);
  return useFetch<T>(debouncedUrl);
}

// Combines debounce + fetch
```

**`useEffectEvent` (experimental) — ref pattern alternativasi:**

```tsx
// experimental_ prefix bilan — production'da ishlatib bo'lmaydi
import { experimental_useEffectEvent as useEffectEvent } from "react";

function useEventListener(event, handler) {
  const stableHandler = useEffectEvent(handler);

  useEffect(() => {
    window.addEventListener(event, stableHandler);
    return () => window.removeEventListener(event, stableHandler);
  }, [event]);
}
```

`useEffectEvent` — handler always sees latest props/state without re-subscribing.

**Common gotchas:**

```tsx
// ❌ Stale closure in useEventListener (without ref pattern)
function useEventListener(event, handler) {
  useEffect(() => {
    window.addEventListener(event, handler);
    return () => window.removeEventListener(event, handler);
  }, [event, handler]);  // handler dep — re-subscribe every render if inline
}

// ✅ Ref pattern (or useEffectEvent)
function useEventListener(event, handler) {
  const handlerRef = useRef(handler);
  useEffect(() => { handlerRef.current = handler; }, [handler]);

  useEffect(() => {
    const listener = (e) => handlerRef.current(e);
    window.addEventListener(event, listener);
    return () => window.removeEventListener(event, listener);
  }, [event]);
}
```

</details>

### Edge Cases

- **`useFetch` with stale URL**: Cleanup cancels prev. Handle race in deps.
- **`useLocalStorage` quota exceeded**: Try-catch on setItem.
- **`useIntersectionObserver` no support**: Polyfill or fallback.

### Follow-up savollar

- "Library hooks vs custom?" — TanStack Query for fetch, react-use for utilities, custom for app-specific.
- "Test custom hooks in CI?" — RTL + jest. Or Vitest.
- "Performance budget for hooks?" — Har hook'ning real overhead'i kichik. React Profiler bilan o'lchash kerak — micro-optimization premature bo'ladi.

</details>

---

## Xulosa

Bu faylda quyidagilar yoritildi:

**QISM A — Hooks Fundamentals (1-5)**: Hooks rationale, Rules of Hooks, linked list mexanizmi, mount/update path, dispatcher swap, conditional hook bug.

**QISM B — useState (6-9)**: API va lazy initializer, functional update vs direct, queue mechanism (circular linked list), state equality bailout.

**QISM C — useEffect (10-15)**: Lifecycle EMAS — sinxronizatsiya, dependency array, cleanup function, race conditions + AbortController, Strict Mode 2x effect, "You Might Not Need an Effect" anti-patterns.

**QISM D — useLayoutEffect & useInsertionEffect (16-18)**: Timing farq, DOM measurement use cases, useInsertionEffect (CSS-in-JS).

**QISM E — useRef (19-23)**: useRef vs useState, forwardRef R18 vs R19 ref as prop, R19 ref cleanup, useImperativeHandle, callback refs.

**QISM F — useContext (24-27)**: Provider, value reference performance, R19 `<Context value>` va `use(context)`, selector pattern.

**QISM G — useReducer (28-30)**: vs useState, discriminated unions + exhaustive check, Reducer + Context state container.

**QISM H — useMemo & useCallback (31-34)**: useMemo mechanics, useCallback (useMemo wrapper), memo + useCallback paired, when NOT to memoize.

**QISM I — Concurrent Hooks R18 (44-49)**: useTransition, useDeferredValue, useSyncExternalStore (tearing prevention), useId, decision tree.

**QISM J — R19 Hooks (50-53)**: use() hook, useFormStatus + useActionState, useOptimistic, decision tree.

**QISM K — Custom Hooks (54-56)**: Pattern va composition, useDebugValue, common custom hooks library.

**Keyingi:** [04-patterns.md](04-patterns.md) — Components, Composition, Patterns, Error Boundaries, Portals — function components, props, lifting state, controlled/uncontrolled, render props (legacy), HOC (legacy), compound components, error boundaries (R19 root callbacks), portals.



















