# Bo'lim 15: Hooks Asoslari

> Hooks — function component'larga state va lifecycle imkoniyatlarini bergan R16.8'da kiritilgan API. Bu bo'lim Hooks'ning paydo bo'lish sabablarini, Rules of Hooks invariant'ini, Fiber'dagi `memoizedState` linked list strukturasini, dispatcher swap mexanikasini, va custom hooks pattern'larini chuqur yoritadi.

---

## Mundarija

- [Nima Uchun Hooks — Class Muammolari](#nima-uchun-hooks--class-muammolari)
- [Rules of Hooks](#rules-of-hooks)
- [Hooks Linked List — Fiber `memoizedState`](#hooks-linked-list--fiber-memoizedstate)
- [Hook Indexing va Call Order](#hook-indexing-va-call-order)
- [Mount Path vs Update Path](#mount-path-vs-update-path)
- [Dispatcher Swap — `ReactCurrentDispatcher`](#dispatcher-swap--reactcurrentdispatcher)
- [First Render — Hook Creation](#first-render--hook-creation)
- [Re-render — Hook Lookup](#re-render--hook-lookup)
- [Conditional Hook — Nima Uchun TAQIQ](#conditional-hook--nima-uchun-taqiq)
- [Custom Hooks — Pattern](#custom-hooks--pattern)
- [Hook Composition](#hook-composition)
- [ESLint Plugin — `eslint-plugin-react-hooks`](#eslint-plugin--eslint-plugin-react-hooks)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Nima Uchun Hooks — Class Muammolari

### Nazariya

Hooks — R16.8 (2019) da React'ga qo'shildi. Class component'lardagi 3 ta katta muammoni hal qilish uchun yaratildi.

**Muammo 1 — `this` binding:**

Class component method'lari `this` context'ga bog'liq. JSX'da method uzatilsa — `this` yo'qoladi:

```tsx
// ❌ Class component — this binding muammo
class Counter extends React.Component {
  state = { count: 0 };
  
  increment() {
    // ❌ this undefined — JSX onClick chaqirsa
    this.setState({ count: this.state.count + 1 });
  }
  
  render() {
    return <button onClick={this.increment}>+</button>;
    //                       ↑ this binding yo'q — runtime error
  }
}

// Yechim 1: bind constructor'da
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.increment = this.increment.bind(this);  // Manual bind
  }
}

// Yechim 2: arrow function — class field
class Counter extends React.Component {
  state = { count: 0 };
  
  increment = () => {
    this.setState({ count: this.state.count + 1 });
  };
  
  render() {
    return <button onClick={this.increment}>+</button>;
  }
}
```

Function component'da `this` yo'q. Hooks closure orqali state'ga kirish — bind muammosi yo'q:

```tsx
// ✅ Hooks — closure orqali, this yo'q
function Counter() {
  const [count, setCount] = useState(0);
  
  const increment = () => {
    setCount(c => c + 1);
  };
  
  return <button onClick={increment}>+</button>;
}
```

**Muammo 2 — Lifecycle Soup:**

Class component'larda bir logic bir nechta lifecycle method'ga tarqalgan:

```tsx
// ❌ Subscription logic 3 ta lifecycle method'da
class ChatRoom extends React.Component<{ roomId: string }> {
  state = { messages: [] };
  
  componentDidMount() {
    this.subscribe();  // Subscription
    document.title = `Chat: ${this.props.roomId}`;  // Document title
  }
  
  componentDidUpdate(prevProps: { roomId: string }) {
    if (prevProps.roomId !== this.props.roomId) {
      this.unsubscribe();
      this.subscribe();  // Re-subscribe
    }
    document.title = `Chat: ${this.props.roomId}`;  // Yana title update
  }
  
  componentWillUnmount() {
    this.unsubscribe();  // Cleanup
  }
  
  // 3 ta method'da related logic
}
```

Hooks bilan har **mustaqil concern** o'z `useEffect` ichida:

```tsx
// ✅ Hooks — har concern alohida useEffect
function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState([]);
  
  // Subscription logic
  useEffect(() => {
    const subscription = subscribe(roomId, setMessages);
    return () => subscription.unsubscribe();  // Cleanup birga
  }, [roomId]);
  
  // Document title — alohida concern
  useEffect(() => {
    document.title = `Chat: ${roomId}`;
  }, [roomId]);
  
  return <MessageList messages={messages} />;
}
```

Har `useEffect` — **bitta concern**. Setup va cleanup birga. Logic bir joyda, scattered emas.

**Muammo 3 — Logic Reuse (HOC Hell):**

Class component'larda logic share — HOC (Higher-Order Component) yoki render props. Wrapper hell:

```tsx
// ❌ HOC chain — wrapper hell
class App extends React.Component {
  render() {
    return <UserProfile />;
  }
}

const EnhancedApp = withAuth(withTheme(withFeatureFlags(withAnalytics(App))));
// 4 ta HOC — 4 ta wrapper komponent
// React DevTools'da: <Auth><Theme><FeatureFlags><Analytics><App /></...></...></...></...>
// Props naming clash, hard to debug
```

Hooks bilan — custom hooks composition:

```tsx
// ✅ Custom hooks composition
function App() {
  const auth = useAuth();
  const theme = useTheme();
  const flags = useFeatureFlags();
  useAnalytics();  // Side effect only
  
  return <UserProfile auth={auth} theme={theme} flags={flags} />;
}
```

Custom hook'lar — oddiy funksiyalar. Composition aniq, props clash yo'q, DevTools'da clean tree.

> **🕐 Versiya evolyutsiyasi (Hooks):**
> - **R0.14 — R16.7:** Class component'lar — state va lifecycle uchun yagona usul.
> - **R16.8 (2019 fevral):** Hooks RC va stable release. `useState`, `useEffect`, `useContext`, `useReducer`, `useCallback`, `useMemo`, `useRef`, `useImperativeHandle`, `useLayoutEffect`, `useDebugValue` qo'shildi.
> - **R18 (2022):** `useTransition`, `useDeferredValue`, `useId`, `useSyncExternalStore`, `useInsertionEffect` qo'shildi.
> - **R19 (2024):** `use()`, `useFormStatus`, `useActionState`, `useOptimistic` qo'shildi.
> - **Sabab:** Function components — class'larning HOC/render props complexity'sini bartaraf qilish. Composition, code reuse, va concurrent rendering uchun zaruriy.

<details>
<summary><strong>Under the Hood</strong></summary>

Class component vs function component — Reconciler farq:

```ts
// Class component (Fiber tag: ClassComponent)
fiber.tag === ClassComponent
fiber.type === Counter (class reference)
fiber.stateNode === counterInstance (new Counter(props))
fiber.memoizedState === { count: 0 }  // this.state

// Function component (Fiber tag: FunctionComponent)
fiber.tag === FunctionComponent
fiber.type === Counter (function reference)
fiber.stateNode === null  // No instance!
fiber.memoizedState === Hook linked list  // hooks chain
```

Class — instance + `this.state`. Function — closure + hooks linked list.

**`this` binding ECMAScript:**

```ts
class A {
  method() { return this; }
}

const a = new A();
a.method();           // a (correct)
const m = a.method;
m();                  // undefined (strict mode) — this lost
```

JS function call'da `this` — call site'ga bog'liq. Method extracted bo'lsa, `this` yo'qoladi.

**Class lifecycle methods order:**

```
Mount:
  constructor
  componentDidMount

Update:
  componentDidUpdate

Unmount:
  componentWillUnmount
```

Har lifecycle — **time event**: "shu paytda chaqiriladi". Hooks — **synchronization model**: "shu state bilan sync bo'lib turing" (cross-ref [`16-useeffect.md`](16-useeffect.md)).

**HOC implementation — extra Fiber:**

```tsx
function withAuth(Component) {
  return function WithAuth(props) {
    const auth = useAuth();
    return <Component {...props} auth={auth} />;
  };
}

// Result tree:
// <WithAuth>
//   <Component auth={auth} {...props} />
// </WithAuth>
```

Har HOC — qo'shimcha Fiber level. 4 HOC = 4 extra Fiber + 4 extra render. DevTools'da nested visible.

Custom hook — **same Fiber'da**:

```tsx
function App() {
  const auth = useAuth();  // Hook chaqiriladi App Fiber'da
  return <Component auth={auth} />;
}

// Result tree:
// <App>     ← hooks chain auth, theme, flags
//   <Component auth={auth} />
// </App>
```

Bir Fiber, ko'p hook. Tree depth kichikroq, render kam.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Class → Function migration:

```tsx
// ❌ Class component
class TimerClass extends React.Component {
  state = { seconds: 0 };
  intervalId?: number;
  
  componentDidMount() {
    this.intervalId = window.setInterval(() => {
      this.setState((s) => ({ seconds: s.seconds + 1 }));
    }, 1000);
  }
  
  componentWillUnmount() {
    if (this.intervalId) clearInterval(this.intervalId);
  }
  
  render() {
    return <p>{this.state.seconds}s</p>;
  }
}

// ✅ Function component
function Timer() {
  const [seconds, setSeconds] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      setSeconds((s) => s + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);
  
  return <p>{seconds}s</p>;
}
```

Logic share — custom hook:

```tsx
// Reusable interval logic
function useInterval(callback: () => void, delay: number | null) {
  const savedCallback = useRef(callback);
  
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);
  
  useEffect(() => {
    if (delay === null) return;
    
    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}

// Usage 1:
function Timer() {
  const [seconds, setSeconds] = useState(0);
  useInterval(() => setSeconds((s) => s + 1), 1000);
  return <p>{seconds}s</p>;
}

// Usage 2:
function Polling() {
  const [data, setData] = useState(null);
  useInterval(async () => {
    const r = await fetch('/api/status');
    setData(await r.json());
  }, 5000);
  return <pre>{JSON.stringify(data)}</pre>;
}
```

HOC vs Hook comparison:

```tsx
// HOC pattern
function withAuth(Component) {
  return function WithAuth(props) {
    const [user, setUser] = useState<User | null>(null);
    useEffect(() => {
      fetch('/api/me').then((r) => r.json()).then(setUser);
    }, []);
    return <Component {...props} user={user} />;
  };
}

const ProfileWithAuth = withAuth(Profile);

// Hook pattern
function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  useEffect(() => {
    fetch('/api/me').then((r) => r.json()).then(setUser);
  }, []);
  return user;
}

function Profile() {
  const user = useAuth();
  return user ? <h1>{user.name}</h1> : <p>Loading...</p>;
}
```

Hook — direct, no wrapper.

</details>

---

## Rules of Hooks

### Nazariya

**Rules of Hooks** — React'ning fundamental invariantlari. Buzilsa — runtime error yoki silent bug.

**Qoida 1: Hooks — faqat top level'da chaqiriladi**

Hook'lar **conditional**, **loop**, yoki **nested function**'larda chaqirilmasligi shart. Faqat:
- Component function tanasida
- Custom hook tanasida

```tsx
// ✅ TO'G'RI
function Counter() {
  const [count, setCount] = useState(0);  // Top level
  const [name, setName] = useState('');   // Top level
  
  useEffect(() => { /* ... */ }, []);     // Top level
  
  return <div>{count} {name}</div>;
}

// ❌ XATO — conditional
function BadCounter({ enabled }: { enabled: boolean }) {
  if (enabled) {
    const [count, setCount] = useState(0);  // ❌ Conditional
  }
  
  return <div>...</div>;
}

// ❌ XATO — loop
function BadList({ items }: { items: Item[] }) {
  for (const item of items) {
    const [open, setOpen] = useState(false);  // ❌ Loop
  }
}

// ❌ XATO — early return'dan keyin
function BadComponent({ data }: { data: Data | null }) {
  if (!data) return null;
  
  const [count, setCount] = useState(0);  // ❌ Early return'dan keyin
  return <div>{count}</div>;
}

// ❌ XATO — nested function
function BadHandler() {
  const handleClick = () => {
    const [count, setCount] = useState(0);  // ❌ Nested function
  };
  return <button onClick={handleClick}>...</button>;
}
```

**Qoida 2: Hooks — faqat React function/custom hook ichida chaqiriladi**

```tsx
// ✅ Component function
function App() {
  const [count] = useState(0);  // OK
}

// ✅ Custom hook (use prefix)
function useCounter() {
  const [count] = useState(0);  // OK
  return count;
}

// ❌ Regular function
function regularFunction() {
  const [count] = useState(0);  // ❌ Throw at runtime
}

// ❌ Class component
class App extends React.Component {
  someMethod() {
    const [count] = useState(0);  // ❌ Throw
  }
}
```

**Nima uchun bu qoidalar:**

React hook'larni **chaqiruv tartibi**ga qarab indekslashtiradi (positional). Conditional hook → tartib buziladi → noto'g'ri indeks:

```tsx
// Render 1 (cond=true):
const [a] = useState(1);   // hook[0]
if (cond) {
  const [b] = useState(2); // hook[1]
}
const [c] = useState(3);   // hook[2]

// Render 2 (cond=false):
const [a] = useState(1);   // hook[0] ✓
// (b skip qilinadi)
const [c] = useState(3);   // hook[1] ❌ — eski b'ning slot'iga keldi
// React: c'ning state = b'ning eski state — silent bug
```

React'ning runtime check'i:
- "Rendered fewer hooks than during the previous render" — hook count past
- "Rendered more hooks than during the previous render" — hook count yuqori

**Qoida tegishli kontekst:**

```tsx
// Hook chaqirilishi mumkin:
function MyComponent() {}     // ✅ Component
function useCustomHook() {}    // ✅ Custom hook (use prefix)

// Hook chaqirilishi MUMKIN EMAS:
function utility() {}          // ❌ Regular utility function
const obj = {                  // ❌ Object method
  method() { useState(0); }
};
class MyClass {                // ❌ Class
  method() { useState(0); }
}
```

`use` prefix — convention. ESLint `react-hooks/rules-of-hooks` `use*` boshlangan funksiyalarni custom hook deb biladi.

<details>
<summary><strong>Under the Hood</strong></summary>

React runtime check — hook count tracking:

```ts
// react-reconciler internal (soddalashtirilgan)
let currentlyRenderingFiber: Fiber | null = null;
let currentHook: Hook | null = null;
let workInProgressHook: Hook | null = null;

function renderWithHooks(...) {
  // Reset state
  currentlyRenderingFiber = workInProgress;
  workInProgress.memoizedState = null;
  workInProgressHook = null;
  
  // Component'ni chaqirish
  const children = Component(props);
  
  // After render — check hook count
  if (current !== null) {
    const currentMemoizedState = current.memoizedState;
    let prevHook = currentMemoizedState;
    let nextHook = workInProgress.memoizedState;
    
    let prevCount = 0;
    while (prevHook !== null) {
      prevCount++;
      prevHook = prevHook.next;
    }
    
    let nextCount = 0;
    while (nextHook !== null) {
      nextCount++;
      nextHook = nextHook.next;
    }
    
    if (prevCount !== nextCount) {
      throw new Error(
        `Rendered ${nextCount > prevCount ? 'more' : 'fewer'} hooks ` +
        `than during the previous render.`
      );
    }
  }
  
  return children;
}
```

Hook count — render'lar bo'ylab teng bo'lishi shart.

**Hook chaqiruv outside component — Invalid Hook Call:**

```ts
// react internal
const ContextOnlyDispatcher = {
  useState: throwInvalidHookError,
  useEffect: throwInvalidHookError,
  // ... boshqa hook'lar
};

function throwInvalidHookError() {
  throw new Error(
    'Invalid hook call. Hooks can only be called inside of the body of a function component.'
  );
}

// Render tashqarisida ReactCurrentDispatcher.current = ContextOnlyDispatcher
// Hook chaqirilsa — throw
```

Render paytida — `HooksDispatcherOnMount` yoki `HooksDispatcherOnUpdate`. Tashqarisida — `ContextOnlyDispatcher` (faqat `useContext` ishlatilishi mumkin, qolganlar throw).

**`use` prefix convention:**

ESLint plugin `eslint-plugin-react-hooks/rules-of-hooks` static analysis qiladi:

```ts
// AST analysis
function isHookCall(node: CallExpression): boolean {
  if (node.callee.type === 'Identifier') {
    const name = node.callee.name;
    return name.startsWith('use') && /^use[A-Z0-9]/.test(name);
  }
  if (node.callee.type === 'MemberExpression') {
    // React.useState
    const name = node.callee.property.name;
    return name.startsWith('use') && /^use[A-Z0-9]/.test(name);
  }
  return false;
}
```

`use` + uppercase letter — hook deb sanaladi. `useState`, `useEffect`, `useCustomHook` — yes. `usable`, `userInfo` — no.

**Strict Mode 2x render — hook check:**

Strict Mode'da component 2 marta render qilinadi (cross-ref [`02-rendering.md`](02-rendering.md), [`09-component-basics.md`](09-component-basics.md)). Hook count'ni 2 marta tekshirish — conditional hook shu yerda topiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Top-level only:

```tsx
function GoodComponent({ userId }: { userId: number | null }) {
  // ✅ Top-level — har doim chaqiriladi
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    if (userId === null) {
      setUser(null);
      setLoading(false);
      return;
    }
    
    setLoading(true);
    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then((u) => {
        setUser(u);
        setLoading(false);
      });
  }, [userId]);
  
  // Conditional logic — hook'lardan keyin OK
  if (userId === null) return <p>No user selected</p>;
  if (loading) return <p>Loading...</p>;
  if (!user) return <p>Not found</p>;
  
  return <h1>{user.name}</h1>;
}
```

Anti-pattern — early return then hook:

```tsx
// ❌ Conditional hook
function BadComponent({ data }: { data: Data | null }) {
  if (!data) return <p>No data</p>;  // Early return
  
  const [count, setCount] = useState(0);  // ❌ Conditional hook
  return <div>{count}</div>;
}

// ✅ Hooks first, conditional return after
function GoodComponent({ data }: { data: Data | null }) {
  const [count, setCount] = useState(0);  // ✅ Top-level
  
  if (!data) return <p>No data</p>;
  return <div>{count}</div>;
}
```

Anti-pattern — loop:

```tsx
// ❌ Loop
function BadList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => {
        const [open, setOpen] = useState(false);  // ❌ Loop ichida
        return <li key={item.id}>...</li>;
      })}
    </ul>
  );
}

// ✅ Child component'ga ajratish
function ItemRow({ item }: { item: Item }) {
  const [open, setOpen] = useState(false);  // ✅ Top-level
  return <li>...</li>;
}

function GoodList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => (
        <ItemRow key={item.id} item={item} />
      ))}
    </ul>
  );
}
```

Custom hook — `use` prefix:

```tsx
// ✅ Custom hook
function useToggle(initial = false): [boolean, () => void] {
  const [value, setValue] = useState(initial);
  const toggle = () => setValue((v) => !v);
  return [value, toggle];
}

function App() {
  const [open, toggleOpen] = useToggle();
  return <button onClick={toggleOpen}>{open ? 'Close' : 'Open'}</button>;
}

// ❌ Regular function — runtime error
function toggleHelper(initial = false) {
  const [value, setValue] = useState(initial);  // ❌ Throw — Invalid Hook Call
  return value;
}
```

</details>

---

## Hooks Linked List — Fiber `memoizedState`

### Nazariya

Har Component Fiber'ning `memoizedState` slot'ida hook'lar **linked list** sifatida saqlanadi. Bu — Hooks'ning fundamental data struktura'si.

```ts
// Fiber struktura (hooks perspective)
{
  type: Counter,
  pendingProps: {},
  memoizedState: Hook,  // ← Hooks chain head
  // ...
}

// Hook obyekt
type Hook = {
  memoizedState: any;       // Hook'ning joriy qiymati (state, deps, ref, ...)
  baseState: any;           // Initial yoki baseline state
  baseQueue: Update | null; // Pending updates queue
  queue: UpdateQueue | null; // Update queue (useState, useReducer)
  next: Hook | null;        // Keyingi hook (linked list)
};
```

**Misol — komponent + 3 hook:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);    // Hook 0
  const [name, setName] = useState('');      // Hook 1
  useEffect(() => { /* ... */ }, []);        // Hook 2
  
  return <div>{count} {name}</div>;
}
```

Linked list strukturasi:

```
fiber.memoizedState
  → Hook 0 (useState count)
    memoizedState: 0
    queue: { pending: null, dispatch: setCount, ... }
    next: Hook 1
  
  → Hook 1 (useState name)
    memoizedState: ''
    queue: { pending: null, dispatch: setName, ... }
    next: Hook 2
  
  → Hook 2 (useEffect)
    memoizedState: { tag, create, deps, destroy }
    queue: null
    next: null
```

**Har hook — alohida memory cell:**

- `useState` — `memoizedState` = current value, `queue` = update queue
- `useEffect` — `memoizedState` = effect object, `queue` = null
- `useRef` — `memoizedState` = `{ current: value }`, `queue` = null
- `useMemo` — `memoizedState` = `[memoValue, deps]`, `queue` = null
- `useCallback` — `memoizedState` = `[callback, deps]`, `queue` = null
- `useContext` — context'dan o'qiydi, `memoizedState` minimal
- `useReducer` — `memoizedState` = state, `queue` = update queue (useState'ga o'xshash)

**Linked list — array emas:**

Nima uchun linked list, array emas? Performance va memory:

- Linked list — har hook alohida allocation (V8 hidden class friendly)
- Array — barcha hook'lar contiguous memory (cache friendly), lekin resize cost
- React internal — linked list (per-Fiber memory layout)

Praktik perspektivada: array kabi qarash mumkin (`hooks[0]`, `hooks[1]`, ...), lekin internal'da linked list.

<details>
<summary><strong>Under the Hood</strong></summary>

Hook obyekt creation:

```ts
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };
  
  if (workInProgressHook === null) {
    // Birinchi hook — head
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // Append linked list
    workInProgressHook.next = hook;
    workInProgressHook = hook;
  }
  
  return workInProgressHook;
}
```

Har hook chaqiruvi — yangi obyekt yaratiladi va list'ga ulanadi. `workInProgressHook` — joriy iteratsiya pointer'i.

**Update path — clone:**

```ts
function updateWorkInProgressHook(): Hook {
  let nextWorkInProgressHook: Hook | null;
  
  if (workInProgressHook === null) {
    // Birinchi hook
    const current = currentlyRenderingFiber.alternate;
    if (current !== null) {
      nextWorkInProgressHook = current.memoizedState;
    } else {
      throw new Error('Should have a work-in-progress.');
    }
  } else {
    nextWorkInProgressHook = workInProgressHook.next;
  }
  
  if (nextWorkInProgressHook !== null) {
    // Mavjud hook — clone (workInProgress fiber uchun)
    const clone: Hook = {
      memoizedState: nextWorkInProgressHook.memoizedState,
      baseState: nextWorkInProgressHook.baseState,
      baseQueue: nextWorkInProgressHook.baseQueue,
      queue: nextWorkInProgressHook.queue,
      next: null,
    };
    
    if (workInProgressHook === null) {
      currentlyRenderingFiber.memoizedState = workInProgressHook = clone;
    } else {
      workInProgressHook.next = clone;
      workInProgressHook = clone;
    }
  } else {
    throw new Error('Rendered more hooks than during the previous render.');
  }
  
  return workInProgressHook;
}
```

Update'da — har hook clone qilinadi (immutability). `current.memoizedState`'dan o'qib, `workInProgress`'ga ko'chiriladi.

**Memory layout visualization:**

```
Render 1 (Mount):
  current = null
  workInProgress.memoizedState = null
  
  useState(0) called:
    hook0 = { memoizedState: 0, ... }
    workInProgress.memoizedState = hook0
    workInProgressHook = hook0
  
  useState('') called:
    hook1 = { memoizedState: '', ... }
    hook0.next = hook1
    workInProgressHook = hook1
  
  useEffect(...) called:
    hook2 = { memoizedState: { tag, create, deps }, ... }
    hook1.next = hook2
    workInProgressHook = hook2

End: workInProgress.memoizedState = hook0 → hook1 → hook2 → null

After Commit:
  current = workInProgress (committed)
  
Render 2 (Update):
  current = previous fiber
  workInProgress.memoizedState = null (reset)
  
  useState(0) called:
    Find current.memoizedState = hook0_old
    Clone: hook0_new = { memoizedState: hook0_old.memoizedState, ... }
    workInProgress.memoizedState = hook0_new
  
  useState('') called:
    Next from current: hook1_old (current.memoizedState.next)
    Clone: hook1_new
    hook0_new.next = hook1_new
  
  useEffect(...) called:
    Same pattern
```

**Pointer state during render:**

- `currentlyRenderingFiber` — joriy render qilinayotgan Fiber
- `currentHook` — eski Fiber'dagi hook (update'da clone manbai)
- `workInProgressHook` — yangi Fiber'da yaratilayotgan hook

Bu — Render Phase davomida updated. Render tugagandan keyin — null.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

DevTools'da hooks ko'rish:

```tsx
function Demo() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('Alice');
  const inputRef = useRef<HTMLInputElement>(null);
  const memoValue = useMemo(() => count * 2, [count]);
  
  useEffect(() => {
    document.title = `Count: ${count}`;
  }, [count]);
  
  return (
    <div>
      <input ref={inputRef} value={name} onChange={(e) => setName(e.target.value)} />
      <p>{count} (memo: {memoValue})</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}

// React DevTools "hooks" panel:
// Hook 0: State — 0          (count)
// Hook 1: State — 'Alice'    (name)
// Hook 2: Ref — { current: <input> }
// Hook 3: Memo — 0           (memoValue)
// Hook 4: Effect — passive
```

Hook count — Strict Mode check:

```tsx
function Renderer() {
  const [count, setCount] = useState(0);
  console.log('Render');
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// Strict Mode dev:
// Render
// Render  ← 2x render (cross-ref 09-component-basics.md)
// Hook count check — match qilishi kerak
```

Hook chain demo:

```tsx
function ChainDemo() {
  // Hook 0
  const [user, setUser] = useState<User | null>(null);
  
  // Hook 1
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  // Hook 2
  const userId = useMemo(() => user?.id ?? null, [user]);
  
  // Hook 3
  useEffect(() => {
    if (userId) {
      fetch(`/api/user/${userId}`).then((r) => r.json()).then(setUser);
    }
  }, [userId]);
  
  // Hook 4
  const themeRef = useRef(theme);
  
  // Hook 5
  useEffect(() => {
    themeRef.current = theme;
  }, [theme]);
  
  return <div>...</div>;
}

// Linked list:
// hook0 (useState user) → hook1 (useState theme) → hook2 (useMemo userId)
//   → hook3 (useEffect fetch) → hook4 (useRef theme) → hook5 (useEffect sync) → null
```

</details>

---

## Hook Indexing va Call Order

### Nazariya

React **hook nomini bilmaydi** — faqat **chaqiruv tartibi**ga qarab indekslashtiradi. Bu — Hooks'ning fundamental implementatsiya choice'i.

```tsx
function Component() {
  const [a, setA] = useState(1);   // Hook index 0
  const [b, setB] = useState(2);   // Hook index 1
  const [c, setC] = useState(3);   // Hook index 2
}
```

Render 1 — Hook list:
```
hook[0] = { state: 1 }  (a)
hook[1] = { state: 2 }  (b)
hook[2] = { state: 3 }  (c)
```

Render 2 — `useState` chaqirilganda React **hook[N]** ni topadi (N = chaqiruv tartibi):
```
useState(1) → hook[0] (a) → return [1, setA]
useState(2) → hook[1] (b) → return [2, setB]
useState(3) → hook[2] (c) → return [3, setC]
```

**Tartib har render'da bir xil bo'lishi shart:**

```tsx
// Render 1 (cond=true):
useState(1);   // hook[0]
if (cond) {
  useState(2); // hook[1]
}
useState(3);   // hook[2]

// Render 2 (cond=false):
useState(1);   // hook[0] ✓
// (cond false — skip qilinadi)
useState(3);   // hook[1] ❌ — eski hook[1] (b) ga keldi!
// React: c'ning state = b'ning eski state (silent bug)
```

`useState(3)` chaqirilganda — React `hook[1]`ga indekslashtiradi. Lekin `hook[1]` aslida `b`'ning slot'i edi. `c` `b`'ning state'ini "yutib qoladi".

**Fix — har doim har hook chaqirish:**

```tsx
// Render har doim — bir xil tartib
useState(1);   // hook[0]
useState(2);   // hook[1]  — har doim chaqiriladi
useState(3);   // hook[2]
```

Conditional logic hook ichida (ichki check):

```tsx
function Component({ cond }: { cond: boolean }) {
  const [a] = useState(1);
  const [b] = useState(2);  // Har doim chaqiriladi
  const [c] = useState(3);
  
  // Conditional usage — render natijasida
  return <div>{a}, {cond && b}, {c}</div>;
}
```

`b` har doim deklaratsiya qilinadi, lekin **ishlatilishi** conditional. Hook count stable.

**Skip pattern — `null` initial:**

```tsx
function Component({ enabled }: { enabled: boolean }) {
  // Har doim chaqiriladi
  const [data, setData] = useState<Data | null>(null);
  
  useEffect(() => {
    if (!enabled) {
      setData(null);
      return;
    }
    fetch('/api').then((r) => r.json()).then(setData);
  }, [enabled]);
  
  return <div>{data?.value}</div>;
}
```

Hook chaqiruvlari har doim. Logic ichida — conditional.

<details>
<summary><strong>Under the Hood</strong></summary>

Hook lookup — linked list traversal:

```ts
// Update render — useState chaqirilgan
function updateState<S>(...): [S, ...] {
  // 1. workInProgressHook'ni topish
  const hook = updateWorkInProgressHook();
  // ↑ Linked list traversal: hook.next
  
  // 2. Hook'dan state qaytarish
  return [hook.memoizedState, hook.queue.dispatch];
}
```

`updateWorkInProgressHook` — linked list'da keyingi hook:

```ts
function updateWorkInProgressHook(): Hook {
  let nextWorkInProgressHook: Hook | null;
  
  if (workInProgressHook === null) {
    // Birinchi hook — current.memoizedState (head)
    const current = currentlyRenderingFiber.alternate;
    nextWorkInProgressHook = current?.memoizedState ?? null;
  } else {
    // Keyingi hook — workInProgressHook.next
    nextWorkInProgressHook = workInProgressHook.next;
  }
  
  if (nextWorkInProgressHook === null) {
    throw new Error('Rendered more hooks than during the previous render.');
  }
  
  // Clone (immutability)
  const clone: Hook = { ...nextWorkInProgressHook, next: null };
  
  if (workInProgressHook === null) {
    currentlyRenderingFiber.memoizedState = workInProgressHook = clone;
  } else {
    workInProgressHook.next = clone;
    workInProgressHook = clone;
  }
  
  return workInProgressHook;
}
```

Pointer pattern:
- 1-hook: `current.memoizedState` (head)
- N-hook: oldingi `hook.next`

**Conditional hook violation — runtime check:**

```ts
// Render boshida currentHook null
let currentHook: Hook | null = null;

function nextCurrentHook(): Hook | null {
  if (currentHook === null) {
    return current.memoizedState;
  }
  return currentHook.next;
}

// Hook chaqirilganda nextCurrentHook ishlatiladi
// Agar return null va workInProgressHook ham null — render finished
// Agar return null va keyingi hook chaqirilsa — "Rendered more hooks"
```

**Hook nomini bilmaydi — sabab:**

Hooks API kichik va sodda — hook nomi parametr emas (`useState` vs `useEffect`). React faqat tartibni tracks. Bu — performance + simplicity:

```ts
// Pseudo-alternative (named hooks):
const [count] = useState('count', 0);  // Yo'q — bu API qabul qilinmagan
```

Named hooks — qo'shimcha overhead (Map lookup), conflict (har komponent uchun unique kafolat), va Hooks API'ning soddaligini buzgan bo'lardi. React positional design — minimal API surface.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Tartib stable — to'g'ri pattern:

```tsx
function GoodComponent({ enabled, data }: { enabled: boolean; data: Data | null }) {
  // Har render — har hook chaqiriladi (har xil tartibda EMAS)
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');
  const itemCount = useMemo(() => data?.items.length ?? 0, [data]);
  
  useEffect(() => {
    if (!enabled) return;
    document.title = `${name}: ${count}`;
  }, [enabled, name, count]);
  
  // Conditional usage — render JSX'da
  if (!data) return <p>Loading...</p>;
  if (!enabled) return <p>Disabled</p>;
  
  return (
    <div>
      <h1>{name}</h1>
      <p>Count: {count}, Items: {itemCount}</p>
    </div>
  );
}
```

Conditional hook — anti-pattern + fix:

```tsx
// ❌ Anti-pattern
function BadComponent({ feature }: { feature: 'A' | 'B' }) {
  if (feature === 'A') {
    const [a] = useState(0);  // ❌ hook[0] only when feature='A'
  }
  const [b] = useState(0);    // ❌ hook[0] yoki hook[1] — depends
}

// ✅ Fix — har doim har hook
function GoodComponent({ feature }: { feature: 'A' | 'B' }) {
  const [a] = useState(0);  // hook[0] — har doim
  const [b] = useState(0);  // hook[1] — har doim
  
  return feature === 'A' ? <div>{a}</div> : <div>{b}</div>;
}
```

Skip pattern — `null` enabled:

```tsx
type Props = { enabled: boolean; userId: number };

function FetchDemo({ enabled, userId }: Props) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(false);
  
  useEffect(() => {
    if (!enabled || !userId) {
      setUser(null);
      setLoading(false);
      return;
    }
    
    setLoading(true);
    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then((u) => {
        setUser(u);
        setLoading(false);
      });
  }, [enabled, userId]);
  
  // Hook'lar har doim chaqiriladi (top-level)
  // Logic — useEffect ichida conditional
}
```

Component split — hook scope:

```tsx
// ❌ Loop ichida hook
function BadList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => {
        const [open, setOpen] = useState(false);  // ❌ Loop ichida
        return <li key={item.id}>{item.name}</li>;
      })}
    </ul>
  );
}

// ✅ Per-item Component
function ItemRow({ item }: { item: Item }) {
  const [open, setOpen] = useState(false);  // ✅ Top-level
  return (
    <li>
      <button onClick={() => setOpen((o) => !o)}>
        {item.name} {open ? '▼' : '▶'}
      </button>
      {open && <p>Details...</p>}
    </li>
  );
}

function GoodList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => (
        <ItemRow key={item.id} item={item} />
      ))}
    </ul>
  );
}
```

Har `<ItemRow>` — alohida Fiber. Har Fiber — alohida hooks linked list. `useState(false)` — har item uchun mustaqil.

</details>

---

## Mount Path vs Update Path

### Nazariya

Hooks ikki implementatsiyaga ega: **Mount** (birinchi render) va **Update** (keyingi render'lar). React render boshida qaysi'ni ishlatish kerakligini aniqlaydi.

**Mount path — birinchi render:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);  // mountState chaqiriladi
}
```

`mountState`:
1. Yangi Hook obyekt yaratiladi
2. Initial value hisoblanadi (yoki lazy chaqiriladi)
3. Queue yaratiladi
4. Dispatch function yaratiladi (bound to fiber + queue)
5. `[initialState, dispatch]` qaytariladi

**Update path — keyingi render'lar:**

```tsx
// Re-render
function Counter() {
  const [count, setCount] = useState(0);  // updateState chaqiriladi (initial e'tiborsiz)
}
```

`updateState`:
1. Mavjud Hook obyekt linked list'dan topiladi (clone)
2. Pending update queue process qilinadi
3. Yangi state hisoblanadi
4. `[newState, dispatch]` qaytariladi (dispatch eski reference)

**Render type tekshiruv:**

```ts
function renderWithHooks(...) {
  if (current === null) {
    // Birinchi render — mount
    ReactCurrentDispatcher.current = HooksDispatcherOnMount;
  } else {
    // Keyingi render — update
    ReactCurrentDispatcher.current = HooksDispatcherOnUpdate;
  }
  
  // Component chaqirish
  const children = Component(props);
  
  // Reset
  ReactCurrentDispatcher.current = ContextOnlyDispatcher;
  
  return children;
}
```

`current === null` — Fiber'ning eski versiyasi yo'q (mount). `current !== null` — eski versiya bor (update).

**Re-render — initial value e'tiborsiz:**

```tsx
function Counter({ initial }: { initial: number }) {
  const [count, setCount] = useState(initial);
  // Initial value faqat birinchi render'da o'qiladi
  // Keyingi render'larda — queue'dan latest state olinadi
  // initial prop o'zgarsa — count o'zgarmaydi
}
```

Bu — `useState`'ning fundamental semantic'i. Initial value — "starting value", har render'da reset emas.

**Mount → unmount → mount — yangi mount:**

```tsx
function Parent() {
  const [show, setShow] = useState(true);
  return (
    <>
      {show && <Counter />}
      <button onClick={() => setShow(s => !s)}>Toggle</button>
    </>
  );
}
```

User click "Toggle":
- show: true → false → Counter **unmount** (Fiber tree'dan o'chiriladi)
- show: false → true → Counter **mount** (yangi Fiber, yangi hooks chain)

Yangi mount — `current === null` yana, `mountState` chaqiriladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`HooksDispatcherOnMount` vs `HooksDispatcherOnUpdate`:

```ts
const HooksDispatcherOnMount: Dispatcher = {
  useState: mountState,
  useEffect: mountEffect,
  useLayoutEffect: mountLayoutEffect,
  useRef: mountRef,
  useMemo: mountMemo,
  useCallback: mountCallback,
  useContext: readContext,
  useReducer: mountReducer,
  useDebugValue: mountDebugValue,
  useDeferredValue: mountDeferredValue,
  useTransition: mountTransition,
  useId: mountId,
  useSyncExternalStore: mountSyncExternalStore,
  useInsertionEffect: mountInsertionEffect,
};

const HooksDispatcherOnUpdate: Dispatcher = {
  useState: updateState,
  useEffect: updateEffect,
  useLayoutEffect: updateLayoutEffect,
  useRef: updateRef,
  useMemo: updateMemo,
  useCallback: updateCallback,
  useContext: readContext,
  useReducer: updateReducer,
  useDebugValue: updateDebugValue,
  useDeferredValue: updateDeferredValue,
  useTransition: updateTransition,
  useId: updateId,
  useSyncExternalStore: updateSyncExternalStore,
  useInsertionEffect: updateInsertionEffect,
};
```

Har hook — ikki versiya. Mount — **yaratish** logic, Update — **lookup + apply** logic.

**`mountState` — full code:**

```ts
function mountState<S>(initialState: S | (() => S)): [S, Dispatch<SetStateAction<S>>] {
  const hook = mountWorkInProgressHook();
  
  // Lazy initial
  const computed = typeof initialState === 'function'
    ? (initialState as () => S)()
    : initialState;
  
  hook.memoizedState = computed;
  hook.baseState = computed;
  
  const queue: UpdateQueue<S, S> = {
    pending: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: computed,
  };
  hook.queue = queue;
  
  const dispatch: Dispatch<SetStateAction<S>> = (
    queue.dispatch = dispatchSetState.bind(null, currentlyRenderingFiber, queue)
  );
  
  return [computed, dispatch];
}
```

**`updateState` — uses `updateReducer`:**

```ts
function updateState<S>(initialState: S): [S, Dispatch<SetStateAction<S>>] {
  return updateReducer(basicStateReducer, initialState);
}

function updateReducer<S, I, A>(
  reducer: (s: S, a: A) => S,
  initialArg: I,
  init?: (i: I) => S
): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  
  if (queue === null) {
    throw new Error('Should have a queue.');
  }
  
  // Pending updates'ni base'ga ko'chirish
  let baseQueue = hook.baseQueue;
  let pendingQueue = queue.pending;
  
  if (pendingQueue !== null) {
    // ... merge logic
    queue.pending = null;
  }
  
  // Iterate queue va state hisoblash
  let newState = hook.baseState;
  let update = baseQueue;
  
  while (update !== null) {
    newState = reducer(newState, update.action);
    update = update.next;
  }
  
  hook.memoizedState = newState;
  
  return [newState, queue.dispatch];
}
```

Update path — eski state'ni baseline sifatida olib, queue'dan apply qiladi (cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md)).

**Mount → Unmount → Mount lifecycle:**

```ts
// Mount 1
fiber1.alternate = null
fiber1.memoizedState = hooks_chain_1
ReactCurrentDispatcher = HooksDispatcherOnMount

// Update 1
fiber2 = createWorkInProgress(fiber1)
fiber2.alternate = fiber1
ReactCurrentDispatcher = HooksDispatcherOnUpdate

// Unmount
fiber tree'dan olib tashlanadi
hooks_chain GC eligible

// Mount 2 (yangi instance)
fiber3.alternate = null  ← yangi Fiber
ReactCurrentDispatcher = HooksDispatcherOnMount  ← yana mount
```

Component'ning state — instance bound. Unmount → state yo'qoladi. Yangi mount → fresh state.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Initial value e'tiborsiz keyingi render:

```tsx
function Counter({ initialCount }: { initialCount: number }) {
  const [count, setCount] = useState(initialCount);
  // initialCount faqat birinchi render'da o'qiladi
  
  return (
    <div>
      <p>Count: {count}</p>
      <p>Initial (prop): {initialCount}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}

function App() {
  const [base, setBase] = useState(0);
  
  return (
    <div>
      <input
        type="number"
        value={base}
        onChange={(e) => setBase(Number(e.target.value))}
      />
      <Counter initialCount={base} />
      {/* base o'zgarsa: Counter prop yangilanadi, lekin count state — initial */}
    </div>
  );
}
```

key trick — yangi mount uchun:

```tsx
function App() {
  const [base, setBase] = useState(0);
  
  return (
    <div>
      <input
        type="number"
        value={base}
        onChange={(e) => setBase(Number(e.target.value))}
      />
      <Counter key={base} initialCount={base} />
      {/* key o'zgarsa: Counter unmount/mount, fresh state */}
    </div>
  );
}
```

Cross-ref [`08-list-rendering.md`](08-list-rendering.md) — `key` va Komponent Identity.

Mount/Update DevTools demo:

```tsx
function MountDemo() {
  console.log('Component function called');
  
  const [count, setCount] = useState(() => {
    console.log('Lazy init called');
    return 0;
  });
  
  useEffect(() => {
    console.log('Effect mount');
    return () => console.log('Effect cleanup');
  }, []);
  
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// Mount:
// "Component function called"
// "Lazy init called"  ← faqat 1-render'da
// "Effect mount"

// Click +:
// "Component function called"  ← Yangi render
// (Lazy init chaqirilmaydi — updateState)
// (Effect deps [] — qayta chaqirilmaydi)

// Strict Mode:
// "Component function called" x2  ← 2x render
// "Lazy init called" x2
// "Effect mount" → "Effect cleanup" → "Effect mount" (R18+ effect cycle)
```

</details>

---

## Dispatcher Swap — `ReactCurrentDispatcher`

### Nazariya

`ReactCurrentDispatcher` — global mutable container. React render boshida `current` ni tegishli dispatcher'ga o'rnatadi.

```ts
// react package — global state
const ReactCurrentDispatcher = {
  current: null,  // Mutable — React internal'da swap qilinadi
};

// useState chaqirilganda
function useState<S>(initialState: S): [...] {
  const dispatcher = ReactCurrentDispatcher.current;
  if (dispatcher === null) {
    throw new Error('Invalid hook call.');
  }
  return dispatcher.useState(initialState);
}
```

`useState` — barcha hook'lar shu pattern'ni follow qiladi: dispatcher'dan implementation'ni chaqirish.

**Dispatcher uchta:**

1. **`HooksDispatcherOnMount`** — birinchi render
2. **`HooksDispatcherOnUpdate`** — keyingi render'lar
3. **`ContextOnlyDispatcher`** — render tashqarisida (faqat `useContext` ruxsat)

**Render flow:**

```ts
function renderWithHooks(...) {
  // Set dispatcher
  ReactCurrentDispatcher.current = current === null
    ? HooksDispatcherOnMount
    : HooksDispatcherOnUpdate;
  
  try {
    // Component chaqirish — hook'lar dispatcher'ni ishlatadi
    const children = Component(props);
    return children;
  } finally {
    // Reset
    ReactCurrentDispatcher.current = ContextOnlyDispatcher;
  }
}
```

**Render tashqarisida — invalid hook call:**

```tsx
const [count] = useState(0);  // ❌ Module-level — Throw
```

Module yuklanganda — render aktiv emas. `ReactCurrentDispatcher.current = ContextOnlyDispatcher`. `useState` chaqiriladi → `ContextOnlyDispatcher.useState` → throw.

```ts
const ContextOnlyDispatcher = {
  useState: throwInvalidHookError,
  useEffect: throwInvalidHookError,
  useReducer: throwInvalidHookError,
  // ... barcha
  useContext: readContext,  // ← Faqat useContext ishlaydi
};

function throwInvalidHookError() {
  throw new Error(
    'Invalid hook call. Hooks can only be called inside of the body of a function component. ' +
    'This could happen for one of the following reasons:\n' +
    '1. You might have mismatching versions of React and the renderer\n' +
    '2. You might be breaking the Rules of Hooks\n' +
    '3. You might have more than one copy of React in the same app'
  );
}
```

**Render boshqa Fiber'da — dispatcher swap:**

Component A render qilinayotganda dispatcher mount/update. A ichida B child Component chaqirilsa — B uchun yana dispatcher swap (chunki B alohida Fiber):

```ts
// A render boshida: dispatcher = OnMount/OnUpdate (A uchun)
function A() {
  const [count] = useState(0);  // A's hook
  return <B />;  // B element
}

// Reconciler B Fiber'ga keladi
// renderWithHooks(B fiber) chaqiriladi
// dispatcher swap (B uchun mount/update)
function B() {
  const [name] = useState('');  // B's hook
  return <div>...</div>;
}
```

Har Fiber render — alohida dispatcher swap. Hook chain mustaqil.

<details>
<summary><strong>Under the Hood</strong></summary>

`ReactCurrentDispatcher` — react package'da global:

```ts
// react/src/ReactHooks.js (soddalashtirilgan)
import ReactCurrentDispatcher from './ReactCurrentDispatcher';

export function useState<S>(initialState: S | (() => S)): [...] {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}

function resolveDispatcher(): Dispatcher {
  const dispatcher = ReactCurrentDispatcher.current;
  if (dispatcher === null) {
    throw new Error('Invalid hook call.');
  }
  return dispatcher;
}
```

react paket — Hooks API'ning **shell**. Real implementation — react-reconciler paketda.

**Cross-package dispatcher:**

```
react package
  ├─ ReactCurrentDispatcher (mutable global)
  ├─ useState (calls dispatcher.useState)
  └─ useEffect (calls dispatcher.useEffect)

react-reconciler package
  ├─ HooksDispatcherOnMount (real implementation)
  ├─ HooksDispatcherOnUpdate (real implementation)
  └─ Sets ReactCurrentDispatcher.current during render
```

Bu architecture — react package'ni renderer-agnostic qiladi. React DOM, React Native, React Three Fiber — har biri o'z reconciler bilan, lekin hook'lar bir xil.

**Multiple React copies — duplicate dispatcher:**

```
Project:
  - node_modules/react (copy A)
  - node_modules/some-lib/node_modules/react (copy B)

Component'lar har xil React import qilishi mumkin:
  - App imports react (copy A)
  - Lib imports react (copy B)

Lib'ning hook'lari:
  - copy B'ning ReactCurrentDispatcher'ni o'qiydi
  - Lekin renderer copy A'ning dispatcher'ni set qilgan
  - Lib's dispatcher null → Invalid Hook Call
```

Bu — "more than one copy of React" warning sababi. Yechim:
- npm `dedupe` ishlatish
- bundler `resolve.alias`
- peerDependency'lar to'g'ri set qilish

**Dispatcher invalidation — Server Components:**

R19'da Server Components alohida dispatcher ishlatadi (`HooksDispatcherOnServer`):

```ts
const HooksDispatcherOnServer: Dispatcher = {
  useState: throwInvalidHookError,  // Server'da state yo'q
  useEffect: throwInvalidHookError,
  // ...
  useId: serverUseId,  // Faqat ba'zilar ishlaydi
};
```

Server Component'da `useState` chaqirilsa — throw. Cross-ref [`39-rsc-server-actions.md`](39-rsc-server-actions.md).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Module-level hook chaqiruv — invalid:

```tsx
// ❌ Module yuklanganda — render emas
const [globalCount] = useState(0);  // Throw: Invalid Hook Call

// ✅ Faqat component/custom hook ichida
function App() {
  const [count] = useState(0);  // OK
  return <div>{count}</div>;
}
```

useContext outside render — Server Components yoki specific tools:

```tsx
// useContext — ContextOnlyDispatcher'da ishlaydi (lekin invariants bilan)
// Bu — internal mexanizm, oddiy code'da ishlatilmaydi

// Standard pattern:
function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be inside ThemeProvider');
  return ctx;
}
```

Multiple React copies error message:

```
Warning: Invalid hook call. Hooks can only be called inside of the body of a function component. 
This could happen for one of the following reasons:
1. You might have mismatching versions of React and the renderer (such as React DOM)
2. You might be breaking the Rules of Hooks
3. You might have more than one copy of React in the same app
```

Diagnostic:

```bash
# npm
npm ls react

# Output (good):
# my-app@1.0.0
# └── react@19.0.0

# Output (bad — duplicate):
# my-app@1.0.0
# ├── react@19.0.0
# └── some-lib@1.0.0
#     └── react@18.0.0  ← duplicate
```

Yechim:

```json
// package.json
{
  "resolutions": {
    "react": "19.0.0",
    "react-dom": "19.0.0"
  }
}
```

Yoki bundler config:

```ts
// vite.config.ts
export default {
  resolve: {
    dedupe: ['react', 'react-dom'],
  },
};
```

</details>

---

## First Render — Hook Creation

### Nazariya

Birinchi render (mount) — **hook obyektlari yaratiladi va Fiber.memoizedState linked list'ga ulanadi**.

```tsx
function Counter() {
  const [count, setCount] = useState(0);   // Hook 0
  const [name, setName] = useState('');     // Hook 1
  useEffect(() => { /* ... */ }, []);       // Hook 2
}
```

**Step-by-step mount:**

1. **`renderWithHooks` boshlanadi:**
   ```ts
   currentlyRenderingFiber = workInProgress;
   workInProgress.memoizedState = null;  // Reset
   workInProgressHook = null;            // Reset
   ReactCurrentDispatcher.current = HooksDispatcherOnMount;
   ```

2. **Component chaqiriladi:**
   ```ts
   const children = Component(props);  // Counter(props)
   ```

3. **Hook 0: `useState(0)` → `mountState`:**
   ```ts
   const hook = mountWorkInProgressHook();
   // hook = { memoizedState: null, queue: null, next: null }
   // workInProgress.memoizedState = hook (head)
   // workInProgressHook = hook
   
   hook.memoizedState = 0;
   hook.queue = { pending: null, dispatch: setCount, ... };
   
   return [0, setCount];
   ```

4. **Hook 1: `useState('')` → `mountState`:**
   ```ts
   const hook = mountWorkInProgressHook();
   // hook = { memoizedState: null, queue: null, next: null }
   // hook0.next = hook1
   // workInProgressHook = hook1
   
   hook.memoizedState = '';
   hook.queue = { ..., dispatch: setName };
   
   return ['', setName];
   ```

5. **Hook 2: `useEffect(...)` → `mountEffect`:**
   ```ts
   const hook = mountWorkInProgressHook();
   // hook2 yaratiladi, hook1.next = hook2
   
   hook.memoizedState = {
     tag: HookPassive,
     create: () => { /* effect */ },
     destroy: undefined,
     deps: [],
     next: null,
   };
   
   // Effect chain'ga ham qo'shiladi (alohida list — useEffect uchun)
   pushEffect(...)
   ```

6. **Component qaytaradi children, render davom etadi.**

7. **`renderWithHooks` tugaydi:**
   ```ts
   ReactCurrentDispatcher.current = ContextOnlyDispatcher;
   currentlyRenderingFiber = null;
   workInProgressHook = null;
   ```

**Final state — Fiber:**

```
fiber.memoizedState
  → hook0 (useState count)
    memoizedState: 0
    next: hook1
  → hook1 (useState name)
    memoizedState: ''
    next: hook2
  → hook2 (useEffect)
    memoizedState: { tag, create, deps, ... }
    next: null
```

**Effect list — alohida:**

`useEffect`/`useLayoutEffect` — hook chain'da, lekin **effect'larni run qilish** uchun alohida list (Fiber'da `updateQueue`):

```ts
fiber.updateQueue = {
  effects: [effect0, effect1, ...],  // Effect run order
};
```

Render Phase'da effect register, Commit Phase'da effect'lar run (cross-ref [`16-useeffect.md`](16-useeffect.md)).

<details>
<summary><strong>Under the Hood</strong></summary>

`mountWorkInProgressHook` — full implementation:

```ts
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };
  
  if (workInProgressHook === null) {
    // Birinchi hook — Fiber'ning memoizedState = head
    currentlyRenderingFiber!.memoizedState = workInProgressHook = hook;
  } else {
    // Append linked list
    workInProgressHook.next = hook;
    workInProgressHook = hook;
  }
  
  return workInProgressHook;
}
```

Bir-biriga ulanadigan oddiy linked list. Har hook chaqiruvi — bir node.

**`mountEffect` — effect register:**

```ts
function mountEffect(create: () => () => void, deps: any[] | null): void {
  return mountEffectImpl(
    PassiveEffect | PassiveStaticEffect,  // Effect flags
    HookPassive,                           // Hook tag
    create,
    deps
  );
}

function mountEffectImpl(fiberFlags: Flags, hookFlags: HookFlags, create: () => () => void, deps: any[] | null): void {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  
  currentlyRenderingFiber!.flags |= fiberFlags;  // Effect run qilish kerak
  
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,  // Has effect (run qilish kerak)
    create,
    undefined,                   // destroy (cleanup) — keyin set bo'ladi
    nextDeps
  );
}

function pushEffect(tag: HookFlags, create: () => () => void, destroy: any, deps: any[] | null): Effect {
  const effect: Effect = { tag, create, destroy, deps, next: null };
  
  let componentUpdateQueue = currentlyRenderingFiber!.updateQueue;
  if (componentUpdateQueue === null) {
    componentUpdateQueue = { lastEffect: null, ...};
    currentlyRenderingFiber!.updateQueue = componentUpdateQueue;
  }
  
  // Circular linked list of effects
  const lastEffect = componentUpdateQueue.lastEffect;
  if (lastEffect === null) {
    effect.next = effect;
    componentUpdateQueue.lastEffect = effect;
  } else {
    effect.next = lastEffect.next;
    lastEffect.next = effect;
    componentUpdateQueue.lastEffect = effect;
  }
  
  return effect;
}
```

Effect — hook chain'da AND effect chain'da. Hook chain — render order, effect chain — Commit Phase run order.

**`mountRef`:**

```ts
function mountRef<T>(initialValue: T): { current: T } {
  const hook = mountWorkInProgressHook();
  const ref = { current: initialValue };
  hook.memoizedState = ref;
  return ref;
}
```

Ref — `{ current: T }` object. Hook'da reference saqlanadi. Update'da bir xil object qaytadi.

**`mountMemo`:**

```ts
function mountMemo<T>(nextCreate: () => T, deps: any[] | null): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

Memo — `[value, deps]` tuple. Update'da deps bilan compare.

**`mountCallback`:**

```ts
function mountCallback<T>(callback: T, deps: any[] | null): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

Callback — `[fn, deps]` tuple. Update'da deps tekshirib, callback qaytadan saqlanadi yoki bir xil reference.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Hook count tracking:

```tsx
function HookCountDemo() {
  // Mount: hooks list yaratiladi
  const [a] = useState(1);    // Hook 0
  const [b] = useState(2);    // Hook 1
  const memo = useMemo(() => a + b, [a, b]);  // Hook 2
  const cb = useCallback(() => console.log(a, b), [a, b]);  // Hook 3
  const ref = useRef<number>(0);  // Hook 4
  useEffect(() => { /* ... */ }, [a]);  // Hook 5
  
  return <div>{memo}</div>;
}

// React DevTools "Components" panel — hooks 0-5 ko'rinadi
```

Custom hook + nested:

```tsx
function useDouble(value: number) {
  const doubled = useMemo(() => value * 2, [value]);  // Inner hook
  return doubled;
}

function App() {
  const [count, setCount] = useState(0);  // Hook 0 (App)
  const doubled = useDouble(count);        // useDouble inner → Hook 1 (App'da)
  
  return <p>{count} x 2 = {doubled}</p>;
}

// App'ning hooks chain:
// hook0 (useState count)
//   → hook1 (useMemo from useDouble — App'da, inline'iga teng)
```

Custom hook'ning hook'lari — outer komponent Fiber'ning hook chain'iga qo'shiladi (custom hook alohida Fiber emas).

Effect register order:

```tsx
function MultipleEffects() {
  useEffect(() => {
    console.log('Effect 1');
    return () => console.log('Cleanup 1');
  }, []);
  
  useEffect(() => {
    console.log('Effect 2');
    return () => console.log('Cleanup 2');
  }, []);
  
  useEffect(() => {
    console.log('Effect 3');
    return () => console.log('Cleanup 3');
  }, []);
  
  return null;
}

// Mount:
// "Effect 1"
// "Effect 2"
// "Effect 3"
// (Run order — registration order)

// Unmount:
// "Cleanup 1"
// "Cleanup 2"
// "Cleanup 3"
// (Cleanup order — registration order)
```

Effect chain — circular linked list, Commit Phase'da iterate.

</details>

---

## Re-render — Hook Lookup

### Nazariya

Re-render (update) — **mavjud hook obyektlari linked list'dan topiladi va clone qilinadi**, yangi state apply qilinadi.

```tsx
function Counter() {
  const [count, setCount] = useState(0);   // updateState
  const [name, setName] = useState('');     // updateState
}
```

**Step-by-step update:**

1. **`renderWithHooks` boshlanadi:**
   ```ts
   currentlyRenderingFiber = workInProgress;
   workInProgress.memoizedState = null;  // Reset (yangi clone'lar uchun)
   workInProgressHook = null;
   ReactCurrentDispatcher.current = HooksDispatcherOnUpdate;
   ```

2. **Component chaqiriladi:**
   ```ts
   const children = Component(props);
   ```

3. **Hook 0: `useState(0)` → `updateState` → `updateReducer`:**
   ```ts
   const hook = updateWorkInProgressHook();
   // current.memoizedState'dan hook0_old olinadi
   // hook0_clone yaratiladi (memoizedState, queue, baseState ko'chiriladi)
   // workInProgress.memoizedState = hook0_clone
   // workInProgressHook = hook0_clone
   
   // Pending update'lar process qilinadi
   const queue = hook.queue;
   let newState = hook.baseState;
   let update = queue.pending;
   while (update !== null) {
     newState = basicStateReducer(newState, update.action);
     update = update.next;
   }
   
   hook.memoizedState = newState;
   queue.pending = null;
   
   return [newState, queue.dispatch];
   ```

4. **Hook 1: `useState('')` → `updateState`:**
   ```ts
   const hook = updateWorkInProgressHook();
   // workInProgressHook.next = hook1_old (current'dan)
   // hook1_clone yaratiladi
   // hook0_clone.next = hook1_clone
   ```

5. **`renderWithHooks` tugaydi.**

**Hook count check:**

```ts
// Update tugagandan keyin
if (didFinishSilently) {
  // OK
} else if (currentHook !== null && currentHook.next !== null) {
  // Eski hook'lar qoldi — fewer hooks called
  throw new Error('Rendered fewer hooks than during the previous render.');
}
```

**`workInProgressHook` vs `currentHook`:**

- `currentHook` — eski Fiber'dagi hook (eski versiya, source)
- `workInProgressHook` — yangi Fiber'dagi hook (yangi versiya, clone)

Update render'da har hook ikki references — eski (read) va yangi (write).

**Bailout — render skip:**

Eager bailout (cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md)):

```ts
function dispatchSetState(...) {
  if (queue.pending === null) {
    const eagerState = basicStateReducer(lastRenderedState, action);
    if (Object.is(eagerState, lastRenderedState)) {
      return;  // Bailout — render trigger qilinmaydi
    }
  }
  // ... enqueue + schedule
}
```

`setCount(0)` chaqirilsa va eski state ham 0 — render umuman boshlanmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

`updateWorkInProgressHook` — full implementation:

```ts
function updateWorkInProgressHook(): Hook {
  // 1. nextWorkInProgressHook'ni topish
  let nextWorkInProgressHook: Hook | null;
  let nextCurrentHook: Hook | null;
  
  if (currentHook === null) {
    // Birinchi hook
    const current = currentlyRenderingFiber.alternate;
    nextCurrentHook = current?.memoizedState ?? null;
  } else {
    nextCurrentHook = currentHook.next;
  }
  
  if (workInProgressHook === null) {
    nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
  } else {
    nextWorkInProgressHook = workInProgressHook.next;
  }
  
  if (nextWorkInProgressHook !== null) {
    // workInProgress'da hook clone allaqachon yaratilgan (re-render mid-cycle)
    workInProgressHook = nextWorkInProgressHook;
    nextWorkInProgressHook = workInProgressHook.next;
    currentHook = nextCurrentHook;
  } else {
    // Yangi clone yaratish
    if (nextCurrentHook === null) {
      throw new Error('Rendered more hooks than during the previous render.');
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
      workInProgressHook.next = newHook;
      workInProgressHook = newHook;
    }
  }
  
  return workInProgressHook;
}
```

Pointer state — uch variable:
- `currentHook` — eski (source) traversal
- `workInProgressHook` — yangi (target) traversal
- `nextWorkInProgressHook` — keyingi yangi hook (mid-cycle re-render uchun)

**Mid-cycle re-render:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  
  if (count === 0) {
    setCount(1);  // ⚠️ Render'da setCount — mid-cycle re-render
  }
}
```

React bu pattern'ni qo'llab-quvvatlaydi (rare exception). Mid-cycle: render boshqatdan, lekin hook'lar bir xil tartibda. `nextWorkInProgressHook` — saqlangan progress'dan davom etish.

Lekin cheksiz loop xavfli. React `Maximum update depth exceeded` warning beradi (depth limit React internal'da belgilangan).

**`useReducer` deep:**

`useState` aslida `useReducer(basicStateReducer, initial)`'ning maxsus shakli:

```ts
function basicStateReducer<S>(state: S, action: S | ((s: S) => S)): S {
  return typeof action === 'function' ? (action as (s: S) => S)(state) : action;
}

function updateState(initialState) {
  return updateReducer(basicStateReducer, initialState);
}
```

`useReducer` (cross-ref [`20-usereducer.md`](20-usereducer.md)) — bir xil internal mexanizm.

**Hook clone — immutability:**

Update'da hook'lar **clone** qilinadi (`workInProgress` Fiber uchun). Eski Fiber `current` o'zgarmaydi (immutability for Concurrent rendering — render uziladi va workInProgress tashlanadi):

```ts
// Old fiber (current):
fiber.memoizedState = hook0_old → hook1_old → hook2_old

// New fiber (workInProgress):
workInProgress.memoizedState = hook0_new → hook1_new → hook2_new
//                              ↑ Cloned values from old

// Render finished, commit:
// current = workInProgress (swap)
```

Concurrent rendering xavfsiz — workInProgress tashlansa, current butun.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Re-render — initial value e'tiborsiz:

```tsx
function MountVsUpdate() {
  console.log('Render');
  
  const [count, setCount] = useState(() => {
    console.log('Lazy init');  // Faqat 1-render
    return 0;
  });
  
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// Mount:
// "Render"
// "Lazy init"

// Click +:
// "Render"
// (Lazy init chaqirilmaydi — updateState path, initial e'tiborsiz)
```

Bailout demo — eager state:

```tsx
function BailoutDemo() {
  const [count, setCount] = useState(0);
  console.log('Render', count);
  
  return (
    <Stack gap={4}>
      <button onClick={() => setCount(0)}>setCount(0)</button>
      {/* Eager bailout — render bo'lmaydi (Object.is(0, 0) === true) */}
      
      <button onClick={() => setCount(c => c)}>setCount(c => c)</button>
      {/* Functional bailout — c=0, return 0, Object.is(0, 0) === true → bailout */}
      
      <button onClick={() => setCount(c => c + 1)}>setCount(c => c + 1)</button>
      {/* Render trigger */}
    </Stack>
  );
}

// Click "setCount(0)" — "Render 0" ikkinchi marta chiqmaydi (bailout)
// Click "+1" — "Render 1" yangi
```

Pending updates queue:

```tsx
function PendingDemo() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setCount(c => c + 1);  // queue: [+1]
    setCount(c => c + 1);  // queue: [+1, +1]
    setCount(c => c + 1);  // queue: [+1, +1, +1]
    
    // updateState: state=0
    // Apply queue: 0 → 1 → 2 → 3
    // Final: 3
  };
  
  return <button onClick={handleClick}>{count}</button>;
}
```

</details>

---

## Conditional Hook — Nima Uchun TAQIQ

### Nazariya

Conditional hook chaqiruv — **silent bug** manbai. React positional indexing'dan xato natija oladi.

**Misol — anti-pattern:**

```tsx
function Bad({ feature }: { feature: 'A' | 'B' }) {
  if (feature === 'A') {
    const [a, setA] = useState(0);  // hook[0] — only when feature='A'
  }
  const [b, setB] = useState(0);    // hook[0] yoki hook[1] — depends!
}
```

**Render 1 (feature='A'):**

```
hook[0] = a (state: 0)
hook[1] = b (state: 0)
```

**Render 2 (feature='B'):**

```
useState(0) → hook[0]  // React: bu a edi! Lekin endi b ishlatadi
// b — eski a'ning state'iga ega bo'ladi (silent bug)
// Yoki React detect qiladi va throw: "Rendered fewer hooks"
```

**Yana misol — silent state corruption:**

```tsx
function MyComponent({ adminMode }: { adminMode: boolean }) {
  if (adminMode) {
    useEffect(() => {
      console.log('Admin mounted');
    }, []);  // hook[0] — only adminMode=true
  }
  
  const [count, setCount] = useState(0);  // hook[0] yoki hook[1]
  
  // Render 1 (adminMode=true): hooks = [admin-effect, count]
  // Render 2 (adminMode=false): hooks = [count]
  // count — eski admin-effect'ning slot'iga keldi
  // useState(0) chaqirilganda — eski effect data interpret qilinadi
  // → silent bug yoki throw
}
```

**To'g'ri yondashuv — har doim har hook:**

```tsx
function Good({ feature }: { feature: 'A' | 'B' }) {
  const [a, setA] = useState(0);  // Always
  const [b, setB] = useState(0);  // Always
  
  return feature === 'A' ? <div>{a}</div> : <div>{b}</div>;
}
```

Hook'lar har doim deklaratsiya qilinadi (top-level). Conditional logic — render output'da yoki effect ichida.

**Conditional logic effect ichida:**

```tsx
function FetchOnEnable({ enabled, userId }: { enabled: boolean; userId: number }) {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    if (!enabled) {
      setUser(null);
      return;
    }
    
    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then(setUser);
  }, [enabled, userId]);
  // ↑ useEffect har doim chaqiriladi, logic ichida conditional
  
  return user ? <h1>{user.name}</h1> : <p>No user</p>;
}
```

**Component split — qachon kerak:**

Agar conditional logic murakkab va bir necha hook bog'liq bo'lsa — alohida child component:

```tsx
function Parent({ adminMode }: { adminMode: boolean }) {
  return adminMode ? <AdminFeatures /> : <UserFeatures />;
}

function AdminFeatures() {
  const [adminData, setAdminData] = useState<Data | null>(null);
  useEffect(() => {
    fetch('/api/admin').then((r) => r.json()).then(setAdminData);
  }, []);
  return <div>...</div>;
}

function UserFeatures() {
  const [userData, setUserData] = useState<Data | null>(null);
  useEffect(() => {
    fetch('/api/user').then((r) => r.json()).then(setUserData);
  }, []);
  return <div>...</div>;
}
```

Har Component — alohida Fiber, alohida hooks chain. Conditional Component — type identity o'zgaradi (cross-ref [`04-reconciliation.md`](04-reconciliation.md)) — eski unmount, yangi mount, fresh state.

<details>
<summary><strong>Under the Hood</strong></summary>

Hook count violation — runtime error:

```ts
// react-reconciler internal
function renderWithHooks(...) {
  // ... render
  
  const didFinishHook = workInProgressHook;
  
  if (current !== null) {
    // Hook count check
    let prevHookCount = countHooks(current.memoizedState);
    let nextHookCount = countHooks(workInProgress.memoizedState);
    
    if (nextHookCount < prevHookCount) {
      throw new Error(
        'Rendered fewer hooks than during the previous render. ' +
        'This is likely caused by an unintentional early return statement.'
      );
    }
    
    if (nextHookCount > prevHookCount) {
      // workInProgress'da extra hook — bu qaerdan keldi?
      throw new Error('Rendered more hooks than during the previous render.');
    }
  }
}
```

Aniq farqlash — **fewer** vs **more**:

- **Fewer:** Hook conditional skip → keyingi hook'lar pozitsiyaga ko'chadi → React detect
- **More:** Hook conditional add → mavjud hook list'dan keyin yangi → React detect

**`updateWorkInProgressHook` — null check:**

```ts
function updateWorkInProgressHook(): Hook {
  let nextCurrentHook: Hook | null;
  
  if (currentHook === null) {
    const current = currentlyRenderingFiber.alternate;
    nextCurrentHook = current?.memoizedState ?? null;
  } else {
    nextCurrentHook = currentHook.next;
  }
  
  if (nextCurrentHook === null) {
    // Eski hooks list tugadi — lekin hook chaqirilyapti
    throw new Error('Rendered more hooks than during the previous render.');
  }
  
  // ... clone
}
```

Eski list tugagan, lekin yangi hook chaqirilgan → "more hooks".

**Silent corruption — agar count match bo'lsa:**

```tsx
// Render 1
useState(1);  // hook[0] = state: 1
useEffect(...);  // hook[1] = effect

// Render 2 — different order:
useEffect(...);  // hook[0] = ??? (eski useState slot)
useState(1);     // hook[1] = ??? (eski useEffect slot)
```

Hook count match (2 = 2), lekin **type mismatch**. Effect data state sifatida interpret qilinadi → undefined behavior.

Modern React — dev mode'da hook type check:

```ts
// Dev mode validation
if (__DEV__) {
  if (workInProgressHook?.type !== currentHook?.type) {
    console.warn('Hook order changed.');
  }
}
```

Lekin production'da — type check yo'q (performance). Silent bug.

**ESLint plugin — static check:**

`eslint-plugin-react-hooks/rules-of-hooks` — AST-level static analysis. Conditional/loop/nested hook chaqiruvni topadi:

```ts
// Conditional in if statement — flagged
if (cond) {
  const [x] = useState(0);  // ESLint: react-hooks/rules-of-hooks
}

// Conditional && expression — flagged
const result = cond && useState(0);  // ESLint: same

// Loop — flagged
for (const item of items) {
  useState(0);  // ESLint: same
}

// Nested function — flagged (unless function name starts with `use`)
function inner() {
  useState(0);  // ESLint: same (chunki inner — hook emas)
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Anti-pattern + fix examples:

```tsx
// ❌ 1. Conditional in if
function Bad1({ enabled }: { enabled: boolean }) {
  if (enabled) {
    const [count] = useState(0);  // ❌
  }
}

// ✅ 1. Hook always, logic conditional
function Good1({ enabled }: { enabled: boolean }) {
  const [count, setCount] = useState(0);  // ✅
  
  // Conditional logic
  return enabled ? <div>{count}</div> : null;
}

// ❌ 2. Early return then hook
function Bad2({ user }: { user: User | null }) {
  if (!user) return null;
  const [tab, setTab] = useState('overview');  // ❌
}

// ✅ 2. Hook before return
function Good2({ user }: { user: User | null }) {
  const [tab, setTab] = useState('overview');  // ✅
  
  if (!user) return null;
  return <Tabs value={tab} onChange={setTab} />;
}

// ❌ 3. Loop
function Bad3({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => {
        const [open] = useState(false);  // ❌
        return <li>...</li>;
      })}
    </ul>
  );
}

// ✅ 3. Per-item Component
function Item({ item }: { item: Item }) {
  const [open, setOpen] = useState(false);  // ✅
  return <li>...</li>;
}
function Good3({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => <Item key={item.id} item={item} />)}
    </ul>
  );
}

// ❌ 4. Try-catch wrap
function Bad4() {
  try {
    const [count] = useState(0);  // ❌ try-catch — conditional implicit
  } catch (e) {}
}

// ✅ 4. Hook outside try-catch
function Good4() {
  const [count] = useState(0);  // ✅
  
  // Try-catch other logic
  try {
    // ...
  } catch (e) {}
}
```

Component split — feature toggle:

```tsx
type FeatureFlag = 'A' | 'B';

function Bad({ feature }: { feature: FeatureFlag }) {
  if (feature === 'A') {
    const [aData] = useState<A | null>(null);
    useEffect(() => { fetchA(); }, []);
  }
  if (feature === 'B') {
    const [bData] = useState<B | null>(null);
    useEffect(() => { fetchB(); }, []);
  }
  // ❌ Hooks conditional — undefined behavior
}

function Good({ feature }: { feature: FeatureFlag }) {
  return feature === 'A' ? <FeatureA /> : <FeatureB />;
}

function FeatureA() {
  const [data, setData] = useState<A | null>(null);
  useEffect(() => {
    fetchA().then(setData);
  }, []);
  return <div>...</div>;
}

function FeatureB() {
  const [data, setData] = useState<B | null>(null);
  useEffect(() => {
    fetchB().then(setData);
  }, []);
  return <div>...</div>;
}
```

</details>

---

## Custom Hooks — Pattern

### Nazariya

**Custom hook** — `use` prefix bilan boshlanadigan funksiya. Boshqa hook'larni chaqiradi va logic'ni capsulate qiladi.

```tsx
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  
  const increment = () => setCount((c) => c + 1);
  const decrement = () => setCount((c) => c - 1);
  const reset = () => setCount(initial);
  
  return { count, increment, decrement, reset };
}

// Ishlatish
function App() {
  const { count, increment, decrement, reset } = useCounter(0);
  
  return (
    <Stack gap={4}>
      <p>{count}</p>
      <Inline gap={4}>
        <button onClick={increment}>+</button>
        <button onClick={decrement}>-</button>
        <button onClick={reset}>Reset</button>
      </Inline>
    </Stack>
  );
}
```

**Naming convention — `use` prefix:**

- `use<X>` (use + uppercase) — custom hook
- ESLint Rules of Hooks `use*` boshlangan funksiyalarni hook deb biladi
- React DevTools `use*` funksiyalarning state'ini ko'rsatadi

**Custom hook anatomy:**

```tsx
function useFetch<T>(url: string): { data: T | null; loading: boolean; error: Error | null } {
  // 1. Built-in hooks
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  // 2. Effect for side effects
  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    setError(null);
    
    fetch(url)
      .then((r) => r.json())
      .then((d: T) => {
        if (!cancelled) {
          setData(d);
          setLoading(false);
        }
      })
      .catch((err: Error) => {
        if (!cancelled) {
          setError(err);
          setLoading(false);
        }
      });
    
    return () => { cancelled = true; };
  }, [url]);
  
  // 3. Return value (state, actions, ...)
  return { data, loading, error };
}
```

**Hook composition:**

Custom hook boshqa hook'larni chaqirishi mumkin (oddiy va custom):

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  
  return debounced;
}

// Compose:
function useDebouncedFetch<T>(url: string, delay: number) {
  const debouncedUrl = useDebounce(url, delay);
  return useFetch<T>(debouncedUrl);
}

// Usage:
function SearchResults({ query }: { query: string }) {
  const { data, loading } = useDebouncedFetch<Result[]>(`/api/search?q=${query}`, 300);
  
  if (loading) return <p>Searching...</p>;
  return <ul>{data?.map((r) => <li key={r.id}>{r.title}</li>)}</ul>;
}
```

**Reusability vs over-abstraction:**

Custom hook — logic share qilish uchun. Lekin har komponent uchun custom hook yaratish — over-abstraction:

```tsx
// ❌ Over-abstraction
function useGreeting(name: string) {
  return useMemo(() => `Hello, ${name}!`, [name]);
}

// ✅ Inline — sodda
const greeting = `Hello, ${name}!`;
```

Custom hook — qo'shilgan qiymat bo'lsa (multiple hooks combined, side effects, complex logic).

**TypeScript generic'lar:**

```tsx
function useLocalStorage<T>(key: string, defaultValue: T): [T, (v: T) => void] {
  const [value, setValue] = useState<T>(() => {
    if (typeof window === 'undefined') return defaultValue;
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : defaultValue;
  });
  
  const setValueAndStore = (v: T) => {
    setValue(v);
    localStorage.setItem(key, JSON.stringify(v));
  };
  
  return [value, setValueAndStore];
}

// Type inference:
const [theme, setTheme] = useLocalStorage<'light' | 'dark'>('theme', 'light');
const [user, setUser] = useLocalStorage<User | null>('user', null);
```

<details>
<summary><strong>Under the Hood</strong></summary>

Custom hook — same Fiber:

```tsx
function useMyHook() {
  const [a] = useState(0);  // App'ning Fiber'ida hook yaratiladi
  const [b] = useState(0);  // App'ning Fiber'ida
  return a + b;
}

function App() {
  const sum = useMyHook();
  // App'ning hook chain: hook0 (useState a), hook1 (useState b)
  return <div>{sum}</div>;
}
```

`useMyHook` — alohida Fiber emas. Uning ichidagi hook'lar — App'ning Fiber'ining `memoizedState` linked list'iga qo'shiladi.

DevTools display:

```
App
  Hooks:
    0: State (a)
    1: State (b)
  Custom hooks:
    useMyHook
      Hooks:
        State (a)
        State (b)
```

DevTools custom hook'larni group qilib ko'rsatadi (visual organization), lekin internal'da bir xil chain.

**Composition tree:**

```tsx
function useA() {
  useState(0);    // hook
  useEffect(...); // hook
}

function useB() {
  useA();         // composition
  useState(0);    // hook
}

function App() {
  useB();         // composition
  useState(0);    // hook
}

// App's hook chain:
// hook0 (useA's useState)
// hook1 (useA's useEffect)
// hook2 (useB's useState)
// hook3 (App's useState)
```

Composition — recursive function call. Hook'lar **flat list**'ga qo'shiladi.

**Closure capture:**

```tsx
function useCounter() {
  const [count, setCount] = useState(0);
  
  // increment — closure ichida count'ni capture qiladi
  const increment = () => setCount(count + 1);
  // ⚠️ count snapshot — async kontekstda stale (cross-ref 12)
  
  // Functional update — safe
  const incrementSafe = () => setCount((c) => c + 1);
  
  return { count, increment, incrementSafe };
}
```

Custom hook ichida closure — bir xil semantikasi (cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md) — Closure Trap).

**Memoization in custom hook:**

```tsx
function useMyCallback() {
  const [count, setCount] = useState(0);
  
  // Har render yangi function — consumer'da memo bailout buziladi
  const handler = () => count + 1;
  
  // ✅ useCallback bilan stable
  const stableHandler = useCallback(() => count + 1, [count]);
  
  return { count, handler, stableHandler };
}

function App() {
  const { count, handler, stableHandler } = useMyCallback();
  return <Memo onClick={stableHandler} />;
}
```

Custom hook'da `useMemo`/`useCallback` — consumer'larga stable references taqdim etish uchun (cross-ref [`21-usememo-usecallback.md`](21-usememo-usecallback.md)).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda useToggle:

```tsx
function useToggle(initial = false): [boolean, () => void, (v: boolean) => void] {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue((v) => !v), []);
  return [value, toggle, setValue];
}

function App() {
  const [open, toggleOpen] = useToggle();
  
  return (
    <Stack gap={4}>
      <button onClick={toggleOpen}>{open ? 'Close' : 'Open'}</button>
      {open && <p>Content</p>}
    </Stack>
  );
}
```

useDebounce:

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  
  return debounced;
}

function SearchInput() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);
  
  useEffect(() => {
    if (debouncedQuery) {
      console.log('Search:', debouncedQuery);
      // API call only after 300ms of no typing
    }
  }, [debouncedQuery]);
  
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

usePrevious:

```tsx
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  
  useEffect(() => {
    ref.current = value;
  });
  
  return ref.current;
}

function PriceTracker({ price }: { price: number }) {
  const previousPrice = usePrevious(price);
  
  const trend =
    previousPrice === undefined
      ? null
      : price > previousPrice
      ? '↑'
      : price < previousPrice
      ? '↓'
      : '→';
  
  return <p>${price} {trend}</p>;
}
```

useLocalStorage:

```tsx
function useLocalStorage<T>(
  key: string,
  defaultValue: T
): [T, (value: T) => void, () => void] {
  const [value, setValue] = useState<T>(() => {
    if (typeof window === 'undefined') return defaultValue;
    
    try {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : defaultValue;
    } catch {
      return defaultValue;
    }
  });
  
  const setValueAndStore = useCallback(
    (newValue: T) => {
      setValue(newValue);
      try {
        localStorage.setItem(key, JSON.stringify(newValue));
      } catch (err) {
        console.error('Failed to save to localStorage:', err);
      }
    },
    [key]
  );
  
  const remove = useCallback(() => {
    setValue(defaultValue);
    localStorage.removeItem(key);
  }, [key, defaultValue]);
  
  return [value, setValueAndStore, remove];
}

function App() {
  const [theme, setTheme] = useLocalStorage<'light' | 'dark'>('theme', 'light');
  const [user, setUser, removeUser] = useLocalStorage<User | null>('user', null);
  
  return (
    <div>
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        Theme: {theme}
      </button>
      {user && (
        <Inline gap={4}>
          <span>Logged in as {user.name}</span>
          <button onClick={removeUser}>Logout</button>
        </Inline>
      )}
    </div>
  );
}
```

useFetch — generic + typed:

```tsx
type FetchState<T> = {
  data: T | null;
  loading: boolean;
  error: Error | null;
};

function useFetch<T>(url: string): FetchState<T> {
  const [state, setState] = useState<FetchState<T>>({
    data: null,
    loading: true,
    error: null,
  });
  
  useEffect(() => {
    let cancelled = false;
    setState({ data: null, loading: true, error: null });
    
    fetch(url)
      .then((r) => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r.json();
      })
      .then((data: T) => {
        if (!cancelled) setState({ data, loading: false, error: null });
      })
      .catch((error: Error) => {
        if (!cancelled) setState({ data: null, loading: false, error });
      });
    
    return () => { cancelled = true; };
  }, [url]);
  
  return state;
}

// Usage:
type User = { id: number; name: string };

function UserPage({ userId }: { userId: number }) {
  const { data: user, loading, error } = useFetch<User>(`/api/users/${userId}`);
  
  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;
  if (!user) return null;
  
  return <h1>{user.name}</h1>;
}
```

</details>

---

## Hook Composition

### Nazariya

**Hook composition** — kichik custom hook'larni birlashtirib, complex logic yaratish. React'ning eng kuchli pattern'i.

**Misol — useAuth + useFetch:**

```tsx
function useAuth() {
  const [user, setUser] = useLocalStorage<User | null>('user', null);
  
  const login = async (email: string, password: string) => {
    const response = await fetch('/api/login', {
      method: 'POST',
      body: JSON.stringify({ email, password }),
    });
    const data = await response.json();
    setUser(data.user);
  };
  
  const logout = () => setUser(null);
  
  return { user, login, logout };
}

function useAuthenticatedFetch<T>(url: string): FetchState<T> {
  const { user } = useAuth();
  
  // Faqat authenticated user uchun fetch
  return useFetch<T>(user ? url : '');
}

function MyDashboard() {
  const { user, logout } = useAuth();
  const { data, loading } = useAuthenticatedFetch<Dashboard>('/api/dashboard');
  
  if (!user) return <LoginPrompt />;
  if (loading) return <p>Loading...</p>;
  
  return (
    <div>
      <header>
        Welcome, {user.name}
        <button onClick={logout}>Logout</button>
      </header>
      <Dashboard data={data} />
    </div>
  );
}
```

**Composition layers:**

```
useFetch<T>(url)               ← Low-level: fetch + state
  ↑
useAuthenticatedFetch<T>(url)  ← Mid-level: auth + fetch
  ↑
useDashboard()                  ← High-level: domain-specific
  ↑
MyDashboard component           ← UI
```

**Custom hook'lar — ToolKit:**

Common reusable hooks (kursdan tashqari kutubxonalarda):

- `useDebounce<T>(value, delay)` — value debounced
- `useThrottle<T>(value, limit)` — rate-limited updates
- `useIntersectionObserver(ref)` — scroll-into-view detect
- `useMediaQuery(query)` — responsive design
- `useEventListener(event, handler)` — DOM event subscribe
- `useClickOutside(ref, handler)` — click outside detect
- `useKeyPress(key)` — keyboard shortcut

**Custom hook ko'p ko'rinishlari:**

```tsx
// 1. Tuple return
function useCounter(): [number, () => void, () => void] {
  const [count, setCount] = useState(0);
  return [count, () => setCount(c => c + 1), () => setCount(c => c - 1)];
}

const [count, inc, dec] = useCounter();

// 2. Object return
function useCounter() {
  const [count, setCount] = useState(0);
  return {
    count,
    increment: () => setCount(c => c + 1),
    decrement: () => setCount(c => c - 1),
  };
}

const { count, increment, decrement } = useCounter();

// 3. Single value
function useDouble(value: number): number {
  return useMemo(() => value * 2, [value]);
}

const doubled = useDouble(5);
```

Convention:
- **2 ta qiymat** — tuple (`useState`-like)
- **3+ qiymat** — object (named access)
- **1 qiymat** — direct return

<details>
<summary><strong>Under the Hood</strong></summary>

Composition — flat hook list:

```tsx
function useA() {
  useState(0);    // hook
}

function useB() {
  useA();         // composition
  useEffect(...); // hook
}

function useC() {
  useB();         // composition
  useRef(null);   // hook
}

function App() {
  useC();         // composition
}

// App's hook chain (flat):
// hook0 (useA's useState)
// hook1 (useB's useEffect)
// hook2 (useC's useRef)
```

Recursion — call stack:

```
App() called
  └─ useC() called
       └─ useB() called
            └─ useA() called
                 └─ useState(0) — hook[0] yaratiladi
            └─ useEffect(...) — hook[1] yaratiladi
       └─ useRef(null) — hook[2] yaratiladi
```

Har hook chaqiruv — `mountWorkInProgressHook()` yoki `updateWorkInProgressHook()` chaqiriladi. Linked list'ga qo'shiladi.

**DevTools display — composition tracking:**

React DevTools custom hook'larni nested ko'rsatish:

```
Component: MyDashboard
  Hooks:
    Custom: useAuth
      Hooks:
        Custom: useLocalStorage
          State: { name: 'Alice' }
        ...
    Custom: useAuthenticatedFetch
      Hooks:
        Custom: useAuth (← yana chaqiriladi, alohida slot)
          ...
        Custom: useFetch
          State: { data: ..., loading: false, error: null }
          Effect: ...
```

DevTools — visual organization, internal'da flat list. Har custom hook chaqiruvi — alohida hook'lar set yaratadi.

**Composition cost:**

Har custom hook chaqiruv — function call overhead (V8 optimize qiladi, minimal). Hook'lar yaratish — linked list append O(1).

Performance issue — uzoq composition (10+ levels) emas, balki **each hook'ning o'z work**:

- `useEffect` setup/cleanup
- `useMemo` deps comparison
- `useState` queue process

Real bottleneck — yirik state, ko'p effect, tez updates. Cross-ref [`33-optimization.md`](33-optimization.md).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

useAsync — fetch wrapper:

```tsx
function useAsync<T, P extends unknown[]>(
  fn: (...args: P) => Promise<T>,
  immediate = true
) {
  const [state, setState] = useState<{
    data: T | null;
    loading: boolean;
    error: Error | null;
  }>({
    data: null,
    loading: immediate,
    error: null,
  });
  
  const execute = useCallback(async (...args: P) => {
    setState({ data: null, loading: true, error: null });
    try {
      const data = await fn(...args);
      setState({ data, loading: false, error: null });
      return data;
    } catch (error) {
      setState({ data: null, loading: false, error: error as Error });
      throw error;
    }
  }, [fn]);
  
  useEffect(() => {
    if (immediate) {
      execute(...([] as unknown as P));
    }
  }, []);
  
  return { ...state, execute };
}

// Usage 1: immediate
function UserPage({ userId }: { userId: number }) {
  const fetchUser = useCallback(() => fetch(`/api/users/${userId}`).then(r => r.json()), [userId]);
  const { data, loading } = useAsync(fetchUser);
  
  if (loading) return <p>Loading...</p>;
  return <h1>{data?.name}</h1>;
}

// Usage 2: manual trigger
function ManualFetch() {
  const { data, loading, execute } = useAsync(
    () => fetch('/api/data').then(r => r.json()),
    false
  );
  
  return (
    <Stack gap={4}>
      <button onClick={() => execute()} disabled={loading}>
        {loading ? 'Loading...' : 'Fetch'}
      </button>
      {data && <pre>{JSON.stringify(data)}</pre>}
    </Stack>
  );
}
```

useMediaQuery + composition:

```tsx
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() => {
    if (typeof window === 'undefined') return false;
    return window.matchMedia(query).matches;
  });
  
  useEffect(() => {
    const mql = window.matchMedia(query);
    const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
    mql.addEventListener('change', handler);
    return () => mql.removeEventListener('change', handler);
  }, [query]);
  
  return matches;
}

// Composition:
function useResponsive() {
  const isMobile = useMediaQuery('(max-width: 768px)');
  const isTablet = useMediaQuery('(min-width: 769px) and (max-width: 1024px)');
  const isDesktop = useMediaQuery('(min-width: 1025px)');
  
  return { isMobile, isTablet, isDesktop };
}

function ResponsiveLayout() {
  const { isMobile, isTablet, isDesktop } = useResponsive();
  
  if (isMobile) return <MobileLayout />;
  if (isTablet) return <TabletLayout />;
  return <DesktopLayout />;
}
```

useIntersectionObserver:

```tsx
function useIntersectionObserver(
  ref: React.RefObject<HTMLElement>,
  options?: IntersectionObserverInit
): boolean {
  const [isIntersecting, setIsIntersecting] = useState(false);
  
  useEffect(() => {
    const target = ref.current;
    if (!target) return;
    
    const observer = new IntersectionObserver(
      ([entry]) => setIsIntersecting(entry.isIntersecting),
      options
    );
    
    observer.observe(target);
    return () => observer.disconnect();
  }, [ref, options]);
  
  return isIntersecting;
}

// Composition: lazy image
function LazyImage({ src, alt }: { src: string; alt: string }) {
  const ref = useRef<HTMLImageElement>(null);
  const isVisible = useIntersectionObserver(ref, { threshold: 0.1 });
  
  return (
    <img
      ref={ref}
      src={isVisible ? src : ''}
      alt={alt}
      loading="lazy"
    />
  );
}
```

</details>

---

## ESLint Plugin — `eslint-plugin-react-hooks`

### Nazariya

`eslint-plugin-react-hooks` — Rules of Hooks va deps array'ni static analysis bilan tekshiruvchi ESLint plugin. Modern React loyihalarida **majburiy**.

**Ikki qoida:**

1. **`react-hooks/rules-of-hooks`** — Rules of Hooks tekshiruvi
2. **`react-hooks/exhaustive-deps`** — useEffect/useMemo/useCallback deps array tekshiruvi

**Setup:**

```bash
npm install --save-dev eslint-plugin-react-hooks
```

```json
// .eslintrc
{
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

**`rules-of-hooks` topadigan xatolar:**

```tsx
// ❌ Conditional hook
if (cond) {
  useState(0);  // ← ESLint error
}

// ❌ Loop hook
for (const item of items) {
  useState(0);  // ← ESLint error
}

// ❌ Early return then hook
if (!user) return null;
useState(0);  // ← ESLint error

// ❌ Hook in regular function
function notHook() {
  useState(0);  // ← ESLint error
}

// ❌ Hook in class
class MyClass {
  method() {
    useState(0);  // ← ESLint error
  }
}
```

**`exhaustive-deps` topadigan xatolar:**

```tsx
function Bad({ userId }: { userId: number }) {
  useEffect(() => {
    fetch(`/api/users/${userId}`);  // userId ishlatiladi
  }, []);  // ← ESLint warning: missing dependency 'userId'
}

// Fix: Add userId to deps
function Good({ userId }: { userId: number }) {
  useEffect(() => {
    fetch(`/api/users/${userId}`);
  }, [userId]);  // ✅
}
```

**Rule auto-fix:**

VS Code yoki ESLint CLI bilan auto-fix:

```bash
eslint --fix src/
```

`exhaustive-deps` — missing dep'larni avtomatik qo'sha oladi. Lekin **ehtiyot bo'ling** — ba'zi case'larda manual control kerak.

**Disable rule (kam ishlatish):**

```tsx
// eslint-disable-next-line react-hooks/exhaustive-deps
useEffect(() => {
  fetchOnce();
}, []);  // ESLint warning ignored
```

Disable — faqat aniq biladigan exception case'larda. Default — rule'ga amal qilish.

**Custom hook detection:**

ESLint `use*` boshlangan funksiyalarni custom hook deb biladi va ularning ichida hook chaqiruvni Rules of Hooks tomonidan tekshiradi:

```tsx
// ✅ ESLint biladi
function useMyHook() {
  useState(0);  // OK
}

// ❌ ESLint regular function deb biladi
function myHelper() {
  useState(0);  // ← ESLint error
}

// ⚠️ ESLint tekshira olmaydi (lowercase)
function usable() {
  useState(0);  // ← ESLint warning yo'q (lekin runtime error)
}
```

Convention strict: `use<Uppercase>` — hook, boshqa nomlar — regular function.

<details>
<summary><strong>Under the Hood</strong></summary>

ESLint plugin AST-based static analysis:

```ts
// eslint-plugin-react-hooks internal
module.exports = {
  rules: {
    'rules-of-hooks': {
      create(context) {
        return {
          CallExpression(node) {
            if (isHookCall(node)) {
              checkHookCall(node, context);
            }
          },
        };
      },
    },
    'exhaustive-deps': {
      create(context) {
        return {
          CallExpression(node) {
            if (isEffectCall(node) || isMemoCall(node)) {
              checkDeps(node, context);
            }
          },
        };
      },
    },
  },
};

function isHookCall(node) {
  if (node.callee.type === 'Identifier') {
    return /^use[A-Z]/.test(node.callee.name);
  }
  if (node.callee.type === 'MemberExpression') {
    return /^use[A-Z]/.test(node.callee.property.name);
  }
  return false;
}

function checkHookCall(node, context) {
  // AST tree'da yuqoriga walk
  let parent = node.parent;
  while (parent) {
    if (parent.type === 'IfStatement' || parent.type === 'ForStatement') {
      context.report({
        node,
        message: 'React Hook is called conditionally.',
      });
      return;
    }
    parent = parent.parent;
  }
}
```

**`exhaustive-deps` analysis:**

```ts
function checkDeps(node, context) {
  const callback = node.arguments[0];
  const depsArray = node.arguments[1];
  
  if (!depsArray) return;  // No deps array
  
  // Callback ichida ishlatilgan variable'larni topish
  const usedVariables = analyzeVariables(callback);
  
  // Deps array'da declared
  const declaredDeps = depsArray.elements.map((e) => e.name);
  
  // Missing
  for (const v of usedVariables) {
    if (!declaredDeps.includes(v) && !isStable(v)) {
      context.report({
        node: depsArray,
        message: `React Hook has a missing dependency: '${v}'.`,
        fix: (fixer) => addToDeps(fixer, depsArray, v),
      });
    }
  }
  
  // Unnecessary
  for (const d of declaredDeps) {
    if (!usedVariables.includes(d)) {
      context.report({
        node: depsArray,
        message: `Unnecessary dependency: '${d}'.`,
      });
    }
  }
}

function isStable(variable) {
  // useState's setter, useRef's current — stable, deps'ga kiritmaslik mumkin
  return variable.kind === 'state-setter' || variable.kind === 'ref';
}
```

**Plugin updates:**

`eslint-plugin-react-hooks` — React komandasi ishlab chiqaradi. R18, R19 yangi hook'lar bilan plugin yangilanadi:

- R18: `useTransition`, `useId`, `useSyncExternalStore` deps tekshiruvi
- R19: `use()` (Promise unwrap) — deps yo'q (one-shot)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

ESLint config example:

```json
// .eslintrc.json
{
  "extends": ["react-app", "plugin:react-hooks/recommended"],
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

VS Code integration:

```json
// .vscode/settings.json
{
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "eslint.validate": ["typescript", "typescriptreact"]
}
```

Save'da auto-fix.

Common fix scenarios:

```tsx
// 1. Missing dep
function Bad({ id }: { id: number }) {
  useEffect(() => {
    fetch(`/api/${id}`);
  }, []);  // ⚠️ Warning: missing 'id'
}

function Good({ id }: { id: number }) {
  useEffect(() => {
    fetch(`/api/${id}`);
  }, [id]);  // ✅
}

// 2. Function dep — useCallback
function Bad({ items }: { items: Item[] }) {
  const filter = (item: Item) => item.active;  // Har render yangi
  
  const filtered = useMemo(() => items.filter(filter), [items, filter]);
  // ⚠️ filter har render yangi → useMemo bailout buziladi
}

function Good({ items }: { items: Item[] }) {
  const filter = useCallback((item: Item) => item.active, []);
  
  const filtered = useMemo(() => items.filter(filter), [items, filter]);
  // ✅ filter stable — useMemo bailout ishlaydi
}

// 3. Stable refs — deps'ga kiritmaslik
function Stable() {
  const ref = useRef(0);
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    ref.current = count;
    setCount(count);  // setCount stable — kiritmaslik mumkin
  }, [count]);  // ✅ ref va setCount stable, deps'ga kerak emas
}
```

Custom hook detection:

```tsx
// ✅ ESLint correctly identifies as hook
function useFetch(url: string) {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetch(url).then(r => r.json()).then(setData);
  }, [url]);  // ✅ ESLint checks deps
  
  return data;
}

// ❌ Lowercase — not a hook
function fetcher(url: string) {
  const [data] = useState(null);  // ❌ ESLint error
}

// ✅ Naming convention
function fetchOnce() {
  // No hook usage — OK as regular function
  return fetch('/api/data').then(r => r.json());
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: Hook Inside Object Method

```tsx
// ❌ Object method — not a React function
const utils = {
  fetchData() {
    const [data] = useState(null);  // ❌ Throw
  }
};
```

Hook'lar faqat function declaration/expression sifatida e'lon qilingan komponent yoki custom hook ichida.

---

### Gotcha 2: Conditional Effect Dependency

```tsx
function Bad({ enabled, data }: { enabled: boolean; data: Data | null }) {
  if (enabled) {
    useEffect(() => {
      // ❌ Conditional hook
    }, [data]);
  }
}

// ✅ Effect har doim, conditional logic ichida
function Good({ enabled, data }: { enabled: boolean; data: Data | null }) {
  useEffect(() => {
    if (!enabled) return;
    // ✅ Logic conditional
  }, [enabled, data]);
}
```

---

### Gotcha 3: Custom Hook Without `use` Prefix

```tsx
// ⚠️ ESLint can't verify Rules of Hooks
function getData() {
  const [data] = useState(null);  // Runtime works, lekin ESLint warning yo'q
}

// ✅
function useData() {
  const [data] = useState(null);  // ESLint validates
}
```

`use` prefix — ESLint plugin convention. Nomlash standart'iga rioya qiling.

---

### Gotcha 4: Hook Order Change Within Component

```tsx
function Bad({ adminMode }: { adminMode: boolean }) {
  if (adminMode) {
    useEffect(...);  // hook[0] only adminMode=true
  }
  const [count] = useState(0);  // hook[0] yoki hook[1]
}
```

Hook count va type har render bir xil bo'lishi shart. `useState(0)` har xil pozitsiyada — eski state corrupt.

---

### Gotcha 5: Closure in Custom Hook — Stale Capture

```tsx
function useCounter() {
  const [count, setCount] = useState(0);
  
  // ❌ Closure capture
  const increment = () => setCount(count + 1);
  
  return { count, increment };
}

function App() {
  const { count, increment } = useCounter();
  
  useEffect(() => {
    setTimeout(increment, 1000);  // ⚠️ Stale closure — count snapshot
    setTimeout(increment, 2000);
  }, []);
}

// Fix: functional update inside hook
function useCounter() {
  const [count, setCount] = useState(0);
  const increment = () => setCount((c) => c + 1);  // ✅ Latest state
  return { count, increment };
}
```

---

## Common Mistakes

### ❌ Xato 1: Conditional Hook

```tsx
if (cond) {
  useState(0);  // ❌
}

// ✅ Always call, conditional logic in JSX/useEffect
const [count] = useState(0);
return cond ? <div>{count}</div> : null;
```

---

### ❌ Xato 2: Hook in Loop

```tsx
items.map(() => useState(0));  // ❌

// ✅ Per-item child component
items.map((item) => <Item key={item.id} item={item} />);
```

---

### ❌ Xato 3: Hook After Early Return

```tsx
if (!data) return null;
const [tab] = useState('');  // ❌

// ✅ Hooks first
const [tab] = useState('');
if (!data) return null;
```

---

### ❌ Xato 4: Hook in Regular Function

```tsx
function helper() {
  useState(0);  // ❌
}

// ✅ Custom hook (use prefix)
function useHelper() {
  return useState(0);
}
```

---

### ❌ Xato 5: ESLint Plugin Yo'q

```bash
# ❌ React project without hook lint
npm install react react-dom

# ✅ Always install hook plugin
npm install --save-dev eslint-plugin-react-hooks
```

ESLint Rules of Hooks — runtime bug'larni dev paytida topish.

---

## Amaliy Mashqlar

### Mashq 1: useToggle (Oson)

`useToggle(initial = false)` custom hook yarating: `[value, toggle, setValue]` tuple. Closure trap'siz.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState, useCallback } from 'react';

function useToggle(initial = false): [boolean, () => void, (v: boolean) => void] {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue((v) => !v), []);
  return [value, toggle, setValue];
}

// Usage:
function Demo() {
  const [open, toggleOpen, setOpen] = useToggle();
  
  return (
    <Stack gap={4}>
      <button onClick={toggleOpen}>{open ? 'Close' : 'Open'}</button>
      <button onClick={() => setOpen(true)}>Force Open</button>
      <button onClick={() => setOpen(false)}>Force Close</button>
      {open && <p>Content</p>}
    </Stack>
  );
}
```

`useCallback` — toggle stable reference (memo bailout uchun). Functional update — closure trap'siz.

</details>

---

### Mashq 2: usePrevious (Oson)

`usePrevious<T>(value)` hook: oldingi render'dagi value qaytarsin.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useEffect, useRef } from 'react';

function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  
  useEffect(() => {
    ref.current = value;
  });  // No deps array — har render'da yangilanadi
  
  return ref.current;  // Joriy render — eski value
}

// Usage:
function Counter() {
  const [count, setCount] = useState(0);
  const prev = usePrevious(count);
  
  return (
    <Stack gap={4}>
      <p>Now: {count}, Was: {prev ?? 'first render'}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </Stack>
  );
}
```

**Mexanizm:**

1. Render boshlanadi — `ref.current` eski value (yoki undefined birinchi render'da)
2. Render qaytadi — bu eski value display qilinadi
3. useEffect chaqiriladi — `ref.current = newValue`
4. Keyingi render'da — `ref.current` endi eski value (boshqa render'lar uchun)

</details>

---

### Mashq 3: useDebounce (O'rta)

`useDebounce<T>(value, delay)` hook: value debounced.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState, useEffect } from 'react';

function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);  // Cleanup — value yangilansa eski timer cancel
  }, [value, delay]);
  
  return debounced;
}

// Usage:
function SearchPage() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);
  const [results, setResults] = useState<Result[]>([]);
  
  useEffect(() => {
    if (!debouncedQuery) {
      setResults([]);
      return;
    }
    
    fetch(`/api/search?q=${encodeURIComponent(debouncedQuery)}`)
      .then((r) => r.json())
      .then(setResults);
  }, [debouncedQuery]);
  
  return (
    <Stack gap={8}>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <p>Searching: {debouncedQuery}</p>
      <ul>
        {results.map((r) => <li key={r.id}>{r.title}</li>)}
      </ul>
    </Stack>
  );
}
```

**Mexanizm:**

1. User typing — query her keystroke o'zgaradi
2. useEffect har query change'da timer set qiladi (300ms)
3. Cleanup — query keyingi keystroke'da o'zgarsa, eski timer cancel
4. 300ms tinch tursa — timer fire, debounced yangilanadi

</details>

---

### Mashq 4: useFetch (O'rta)

Generic `useFetch<T>(url)` hook: data, loading, error qaytarsin. Race condition-safe (cancellation).

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState, useEffect } from 'react';

type FetchState<T> = {
  data: T | null;
  loading: boolean;
  error: Error | null;
};

function useFetch<T>(url: string): FetchState<T> {
  const [state, setState] = useState<FetchState<T>>({
    data: null,
    loading: true,
    error: null,
  });
  
  useEffect(() => {
    let cancelled = false;
    setState({ data: null, loading: true, error: null });
    
    fetch(url)
      .then((r) => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r.json();
      })
      .then((data: T) => {
        if (!cancelled) {
          setState({ data, loading: false, error: null });
        }
      })
      .catch((error: Error) => {
        if (!cancelled) {
          setState({ data: null, loading: false, error });
        }
      });
    
    return () => {
      cancelled = true;
    };
  }, [url]);
  
  return state;
}

// Yoki AbortController:
function useFetchWithAbort<T>(url: string): FetchState<T> {
  const [state, setState] = useState<FetchState<T>>({
    data: null,
    loading: true,
    error: null,
  });
  
  useEffect(() => {
    const controller = new AbortController();
    setState({ data: null, loading: true, error: null });
    
    fetch(url, { signal: controller.signal })
      .then((r) => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r.json();
      })
      .then((data: T) => {
        setState({ data, loading: false, error: null });
      })
      .catch((error: Error) => {
        if (error.name === 'AbortError') return;
        setState({ data: null, loading: false, error });
      });
    
    return () => controller.abort();
  }, [url]);
  
  return state;
}

// Usage:
type Post = { id: number; title: string };

function PostsPage() {
  const { data: posts, loading, error } = useFetch<Post[]>('/api/posts');
  
  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;
  if (!posts) return null;
  
  return (
    <ul>
      {posts.map((post) => <li key={post.id}>{post.title}</li>)}
    </ul>
  );
}
```

**Asosiy nuqtalar:**

1. **Race condition** — eski URL response keyingisidan keyin kelishi mumkin
2. **`cancelled` flag** — boolean orqali skip
3. **`AbortController`** (alternative) — request fully cancel
4. **Cleanup** — useEffect return

</details>

---

### Mashq 5: Hooks Order Bug (Qiyin)

Quyidagi komponentdagi Rules of Hooks violation'ni toping. Conditional logic'ni hook'lardan tashqariga ko'chiring va component split orqali fix qiling.

```tsx
function FeatureView({ feature, userId }: { feature: 'A' | 'B'; userId: number }) {
  if (feature === 'A') {
    const [aData, setAData] = useState<A | null>(null);
    
    useEffect(() => {
      fetch(`/api/a/${userId}`).then((r) => r.json()).then(setAData);
    }, [userId]);
    
    return <div>A: {aData?.value}</div>;
  }
  
  const [bData, setBData] = useState<B | null>(null);
  
  useEffect(() => {
    fetch(`/api/b/${userId}`).then((r) => r.json()).then(setBData);
  }, [userId]);
  
  return <div>B: {bData?.value}</div>;
}
```

<details>
<summary><strong>Javob</strong></summary>

**Bug:** Hooks conditional — `feature === 'A'` bo'lsa hook[0] = aData, useEffect; `feature === 'B'` bo'lsa hook[0] = bData, useEffect. Render'lar orasida feature switch — hook order o'zgaradi → silent state corruption yoki throw.

**Fix 1: Component split (afzal):**

```tsx
function FeatureView({ feature, userId }: { feature: 'A' | 'B'; userId: number }) {
  return feature === 'A' ? <FeatureA userId={userId} /> : <FeatureB userId={userId} />;
}

function FeatureA({ userId }: { userId: number }) {
  const [aData, setAData] = useState<A | null>(null);
  
  useEffect(() => {
    fetch(`/api/a/${userId}`).then((r) => r.json()).then(setAData);
  }, [userId]);
  
  return <div>A: {aData?.value}</div>;
}

function FeatureB({ userId }: { userId: number }) {
  const [bData, setBData] = useState<B | null>(null);
  
  useEffect(() => {
    fetch(`/api/b/${userId}`).then((r) => r.json()).then(setBData);
  }, [userId]);
  
  return <div>B: {bData?.value}</div>;
}
```

**Fix 2: Single component, conditional logic ichida:**

```tsx
function FeatureView({ feature, userId }: { feature: 'A' | 'B'; userId: number }) {
  // Hooks always called (top-level)
  const [aData, setAData] = useState<A | null>(null);
  const [bData, setBData] = useState<B | null>(null);
  
  useEffect(() => {
    if (feature !== 'A') return;
    fetch(`/api/a/${userId}`).then((r) => r.json()).then(setAData);
  }, [feature, userId]);
  
  useEffect(() => {
    if (feature !== 'B') return;
    fetch(`/api/b/${userId}`).then((r) => r.json()).then(setBData);
  }, [feature, userId]);
  
  if (feature === 'A') return <div>A: {aData?.value}</div>;
  return <div>B: {bData?.value}</div>;
}
```

**Comparison:**

| Approach | Pros | Cons |
|----------|------|------|
| **Component split** | Clean, isolated state, type-safe per component | More files |
| **Single component** | Single file | Both hooks always run (memory + minor perf) |

**Tavsiya:** Component split — chunki feature'lar mustaqil, isolated state ham logic clean.

`feature` o'zgarsa → FeatureView re-render → eski subcomponent (FeatureA) unmount, yangi (FeatureB) mount. State automatic reset (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — Type Comparison).

</details>

---

## Xulosa

**Hooks paydo bo'lish sababi:**
- Class component'lardagi 3 muammo: `this` binding, lifecycle soup, HOC hell
- R16.8 (2019) — Hooks API (useState, useEffect, useContext, useReducer, useRef, useMemo, useCallback)
- R18 (2022) — Concurrent hooks (useTransition, useDeferredValue, useId, useSyncExternalStore, useInsertionEffect)
- R19 (2024) — `use()`, useFormStatus, useActionState, useOptimistic

**Rules of Hooks:**
- **Top level only** — conditional/loop/early-return-then-hook/nested-function TAQIQ
- **React function/custom hook only** — regular function/class TAQIQ
- **`use` prefix convention** — ESLint plugin static analysis

**Fiber memoizedState — hooks linked list:**
- Har Component Fiber — `memoizedState` linked list (hooks chain)
- Har hook — `{ memoizedState, queue, baseState, baseQueue, next }`
- React **call order**'ga qarab indekslashtiradi (positional indexing)

**Mount vs Update:**
- **Mount:** `mountState`/`mountEffect`/... — yangi hook obyektlari yaratiladi
- **Update:** `updateState`/`updateEffect`/... — mavjud hook'lar topiladi va clone qilinadi
- **Dispatcher swap** — `ReactCurrentDispatcher.current` render boshida (HooksDispatcherOnMount/OnUpdate)
- Render tashqarisida — `ContextOnlyDispatcher` (Invalid Hook Call throw)

**Conditional hook nima uchun TAQIQ:**
- React positional indexing — hook[N] N-chi chaqiruv
- Conditional → tartib buziladi → silent state corruption yoki throw
- "Rendered fewer/more hooks than during the previous render" runtime error

**Custom hooks:**
- `use` prefix — convention va ESLint requirement
- Composition — boshqa hook'larni chaqirish (flat hook list, recursive call)
- Naming convention — tuple (2 value), object (3+ value), direct (1 value)
- Common patterns — useToggle, usePrevious, useDebounce, useFetch, useLocalStorage, useMediaQuery, useIntersectionObserver

**ESLint plugin:**
- `eslint-plugin-react-hooks` — majburiy modern React loyihalarida
- `react-hooks/rules-of-hooks` — Rules of Hooks tekshiruvi
- `react-hooks/exhaustive-deps` — useEffect/useMemo/useCallback deps array
- `use<Uppercase>` convention — plugin static analysis

Keyingi bo'limda useEffect deep dive: "lifecycle hook EMAS, sinxronizatsiya" mental model, side effects, dependency array, cleanup, timing (Commit'dan keyin), Strict Mode 2x effect cycle, race conditions (AbortController), va "You Might Not Need an Effect" anti-patterns.

---

**Keyingi bo'lim:** [16-useeffect.md](16-useeffect.md) — useEffect deep dive: **lifecycle hook EMAS — sinxronizatsiya mexanizmi** (MAJBURIY first section, React docs philosophy), side effects tushunchasi, dependency array (`[]`/`[deps]`/no array), cleanup (return function), timing (Commit'dan keyin, browser paint dan keyin), effect ordering (child before parent), pitfalls (infinite loop, missing deps, object/array referential, race conditions AbortController), Strict Mode 2x effect cycle (R18+ — sinxronizatsiya invariant'i), "You Might Not Need an Effect" anti-patterns (Dan Abramov post — sinxronizatsiya modelidan kelib chiqadi), Under the Hood (passive flag, effect list, cleanup chain).
