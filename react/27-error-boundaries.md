# Bo'lim 27: Error Boundaries

> Error Boundary — React komponentlar daraxtida (subtree) yuz beradigan render xatoliklarini ushlab oladigan, fallback UI ko'rsatadigan, va error reporting service'ga uzatadigan maxsus class komponent. JavaScript'dagi `try/catch` bloklari komponent render davomidagi xatoliklarni ushlay olmaydi — Error Boundary shu cheklovni hal qiladi. Hozirgacha kursda **faqat function komponentlar** ishlatilgan, lekin Error Boundary R19'da ham faqat class komponent orqali yoziladi: `getDerivedStateFromError` va `componentDidCatch` lifecycle method'larning hook ekvivalenti yo'q. Bu fayl **class komponent minimal primer**ni va Error Boundary'ning to'liq lifecycle'ini qamrab oladi.

---

## Mundarija

- [Kirish — Error Handling React'da](#kirish--error-handling-reactda)
- [Class Component Minimal Primer](#class-component-minimal-primer)
- [Class Lifecycle Methods — Hooks Equivalents](#class-lifecycle-methods--hooks-equivalents)
- [`UNSAFE_*` Lifecycle Methods (Deprecated)](#unsafe_-lifecycle-methods-deprecated)
- [Error Boundary Concept va API](#error-boundary-concept-va-api)
- [`getDerivedStateFromError` Static Method](#getderivedstatefromerror-static-method)
- [`componentDidCatch` Instance Method](#componentdidcatch-instance-method)
- [Error Catching Scope — Caught va Not Caught](#error-catching-scope--caught-va-not-caught)
- [Event Handlers va Async Errors — Manual Try/Catch](#event-handlers-va-async-errors--manual-trycatch)
- [Error Boundary Placement Strategy](#error-boundary-placement-strategy)
- [Error Recovery — `key` Prop bilan Boundary Reset](#error-recovery--key-prop-bilan-boundary-reset)
- [Fallback UI Patterns](#fallback-ui-patterns)
- [`react-error-boundary` Library](#react-error-boundary-library)
- [R19 Root Callbacks — Application-Level Error Handling](#r19-root-callbacks--application-level-error-handling)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Kirish — Error Handling React'da

### Nazariya

JavaScript'da xatoliklar `try/catch` bloklari yoki `Promise.catch` orqali ushlab olinadi. Lekin React komponent daraxti ichida xato yuz bersa — masalan, `null.toUpperCase()` render paytida — React **butun ilova daraxtini unmount** qiladi (R16+). Bu R15 bilan farq qiluvchi xatti-harakat: oldingi versiyalarda bitta komponentdagi xato silently saqlanardi va UI `corrupted` holatda qolardi.

R16 (2017) **Error Boundaries** kontseptsiyasini joriy qildi — komponent daraxtining bir qismida (subtree) yuz beradigan xatolikni **ushlab olib**, fallback UI ko'rsatish imkoni paydo bo'ldi.

```tsx
// Error boundary o'rab olgan subtree
<ErrorBoundary fallback={<p>Something went wrong</p>}>
  <BuggyComponent />  {/* Agar render xato beradi → ErrorBoundary fallback ko'rsatadi */}
</ErrorBoundary>
```

NIMA UCHUN bu kerak:

1. **Graceful degradation** — bir feature bug bersa, butun ilova qulamasin. Faqat shu feature'ning UI'i fallback'ga o'tadi.
2. **User experience** — "Something went wrong, try again" message yaxshiroqdan ko'ra blank white screen.
3. **Error reporting** — production'da xatoliklar Sentry/LogRocket'ga yuborilishi kerak.
4. **Recovery** — foydalanuvchi reset tugmasi bilan ilovani qayta yuklashi mumkin.

QANDAY ISHLAYDI: Error Boundary — **class komponent**, `getDerivedStateFromError` (static) va `componentDidCatch` (instance) lifecycle method'larni implement qiladi. React render Phase'da xato yuz bersa:

1. React xatoni eng yaqin Error Boundary'ga "ko'taradi" (parent chain bo'ylab).
2. Boundary'ning `getDerivedStateFromError` chaqiriladi — state yangilanadi (`hasError: true`).
3. Boundary `componentDidCatch` chaqiriladi — error logging.
4. Boundary re-render — `state.hasError` true bo'lsa fallback UI render.

```
Render error:
  null.toUpperCase()
       │
       │ React catches error
       ▼
  Find nearest ErrorBoundary
       │
       ▼
  ErrorBoundary.getDerivedStateFromError(error)
       │
       ▼
  state = { hasError: true }
       │
       ▼
  ErrorBoundary.componentDidCatch(error, info)  ← log to Sentry
       │
       ▼
  Re-render — fallback UI
```

**Eng muhim cheklov**: Error Boundary **render Phase** xatolarini ushlaydi. **Event handlers, async code (setTimeout, Promise)** xatoliklarini ushlamaydi. SSR'da `getDerivedStateFromError` ishlaydi (state update — fallback HTML), lekin `componentDidCatch` chaqirilmaydi (Commit Phase server'da yo'q). Bu cheklov'lar va workaround'lar keyingi section'larda.

> **Versiya evolyutsiyasi (Error Handling):**
> - **R15 va eski:** Komponent xatolari silently corrupt qiladi UI'ni. Reliable error handling yo'q.
> - **R16 (2017):** Error Boundaries kiritildi (`componentDidCatch`). Boundary yo'q bo'lsa — butun daraxt unmount, eng yaqin boundary topilsa fallback ko'rsatadi.
> - **R16.6 (2018):** `getDerivedStateFromError` qo'shildi (state update from error, SSR-compatible).
> - **R19 (2024):** Root-level callbacks (`onCaughtError`/`onUncaughtError`/`onRecoverableError`) — application-wide error reporting boundary'larsiz.

<details>
<summary><strong>Under the Hood</strong></summary>

R16'gacha Error Handling holati:

```javascript
// R15 — komponent ichida xato
function ProductList() {
  return (
    <ul>
      {products.map(p => p.name.toUpperCase())}  // products undefined bo'lsa
    </ul>
  );
}
// R15: silent corruption — DOM partial rendered, no recovery
// R16+: error caught by boundary OR app crashes
```

R16+ React Reconciler render Phase'da xato yuz bersa:

```javascript
// react-reconciler/src/ReactFiberWorkLoop.js (simplified)
function performWork(workInProgress) {
  try {
    // Component render
    nextChildren = renderComponent(workInProgress);
  } catch (thrownValue) {
    // Find nearest Error Boundary
    const boundary = findNearestErrorBoundary(workInProgress);
    
    if (boundary) {
      // Mark boundary to update state
      boundary.flags |= DidCapture;
      boundary.state = {
        ...boundary.state,
        ...boundary.constructor.getDerivedStateFromError(thrownValue),
      };
      
      // Schedule componentDidCatch in commit phase
      boundary.deferredError = thrownValue;
    } else {
      // No boundary — propagate to root
      // R18+: createRoot's onUncaughtError callback fires
      throw thrownValue;
    }
  }
}
```

Error Boundary mexanizmi — Fiber traversal. Xato yuz bergan Fiber'dan parent chain bo'ylab tepaga (root'gacha) Error Boundary qidiriladi. `flags |= DidCapture` — Commit Phase'da boundary fallback'ni render qilishini bildiradi.

`componentDidCatch` Commit Phase'da sinxron chaqiriladi (error logging, side effect ruxsat etilgan), `getDerivedStateFromError` Render Phase'da (state update — re-render trigger).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Bug komponent — render error:

```tsx
import React from 'react';

function BuggyComponent({ data }: { data: { name: string } | null }) {
  // ❌ null-check yo'q — data null bo'lsa render Phase'da TypeError
  return <h1>{(data as { name: string }).name.toUpperCase()}</h1>;
}

// Without ErrorBoundary
function App() {
  return <BuggyComponent data={null} />;
  // R16+: React app crashes — entire UI unmounts
  // Console: "TypeError: Cannot read properties of null (reading 'name')"
}

// With ErrorBoundary
class ErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };
  
  static getDerivedStateFromError(_error: Error) {
    return { hasError: true };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('ErrorBoundary caught:', error, info);
  }
  
  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}

function App() {
  return (
    <ErrorBoundary fallback={<p>Something went wrong</p>}>
      <BuggyComponent data={null} />
    </ErrorBoundary>
  );
  // BuggyComponent xato bersa — fallback ko'rinadi
  // Boshqa app qismi (Header, Sidebar) saqlanadi
}
```

Error Boundary ichidagi xato — fallback render. Tashqaridagi UI normal qoladi.

</details>

---

## Class Component Minimal Primer

### Nazariya

Class komponent — `React.Component` (yoki `React.PureComponent`) class'idan inherit qilingan ES6 class. Hooks (R16.8) gacha React'ning standart komponent yozish usuli edi. Hozir **faqat Error Boundary** uchun kerak (yana ba'zi 3rd-party library'larda).

Minimal class komponent:

```tsx
class Counter extends React.Component<{ initial: number }, { count: number }> {
  state = { count: this.props.initial };
  
  handleClick = () => {
    this.setState(prev => ({ count: prev.count + 1 }));
  };
  
  render() {
    return (
      <div>
        <p>Count: {this.state.count}</p>
        <button onClick={this.handleClick}>Increment</button>
      </div>
    );
  }
}
```

**TypeScript generic'lar:**

```tsx
class ComponentName extends React.Component<PropsType, StateType, SnapshotType>
```

- `PropsType` — props interface
- `StateType` — state interface
- `SnapshotType` (optional) — `getSnapshotBeforeUpdate` return type

**Asosiy elementlar:**

1. **`constructor` (optional)** — state initialization, method binding (arrow methods'da kerak emas).
2. **`state`** — class field yoki constructor'da. Faqat `this.setState()` orqali yangilanadi.
3. **`render()`** — JSX qaytaruvchi method (majburiy).
4. **Lifecycle methods (optional)** — `componentDidMount`, `componentDidUpdate`, va h.k.
5. **Event handlers** — arrow function class fields (auto-bind) yoki regular method + manual bind.

**`this.setState`** — async, batched, partial state update:

```tsx
class Form extends React.Component<{}, { name: string; email: string }> {
  state = { name: '', email: '' };
  
  updateName = (name: string) => {
    this.setState({ name });
  };
  
  updateEmail = (email: string) => {
    this.setState({ email });
  };
  
  appendChar = (char: string) => {
    this.setState(prev => ({ name: prev.name + char }));
  };
}
```

Method binding muammo'si — arrow function class field bilan hal qilinadi:

```tsx
// ❌ Regular method — `this` undefined event handler ichida
class BadComponent extends React.Component {
  handleClick() {
    this.setState(...);  // ❌ TypeError — `this` is undefined
  }
  render() {
    return <button onClick={this.handleClick}>Click</button>;
  }
}

// ✅ Arrow method — `this` auto-bound
class GoodComponent extends React.Component {
  handleClick = () => {
    this.setState(...);  // ✅ `this` class instance'ga bog'langan
  };
  render() {
    return <button onClick={this.handleClick}>Click</button>;
  }
}

// ✅ Constructor bind
class GoodWithBind extends React.Component {
  constructor(props) {
    super(props);
    this.handleClick = this.handleClick.bind(this);
  }
  handleClick() { /* ... */ }
}
```

NIMA UCHUN class hozir kamdan-kam ishlatiladi:

| Aspect | Class | Function + Hooks |
|--------|-------|------------------|
| `this` binding | ❌ Manual | ✅ N/A |
| Lifecycle methods | Ko'p (5+) | `useEffect` (1) |
| Logic reuse | HOC + Render Props | Custom hooks |
| Code volume | Yuqori | Past |
| TypeScript | OK | Yaxshi (inference) |
| React Compiler | Limited | Full optimization |
| Concurrent rendering | OK lekin care kerak | Idiomatic |

QANDAY ISHLAYDI: class komponent React Reconciler tomonidan yangidan instance yaratiladi har mount paytida. State `instance.state`'da, `setState` Re-render trigger qiladi. Lifecycle methods Reconciler tomonidan har Phase'da chaqiriladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Class component vs Function component Reconciler farq:

```javascript
// react-reconciler/src/ReactFiberBeginWork.js (simplified)
function beginWork(workInProgress) {
  switch (workInProgress.tag) {
    case ClassComponent: {
      let instance = workInProgress.stateNode;
      if (instance === null) {
        instance = constructClassInstance(workInProgress, Component);
      }
      
      instance.props = props;
      
      const newState = Component.getDerivedStateFromProps?.(props, instance.state);
      if (newState) instance.state = { ...instance.state, ...newState };
      
      const children = instance.render();
      reconcileChildren(workInProgress, children);
      
      return workInProgress.child;
    }
    
    case FunctionComponent: {
      ReactCurrentDispatcher.current = HooksDispatcherOnMount;
      const children = renderWithHooks(workInProgress, Component, props);
      reconcileChildren(workInProgress, children);
      return workInProgress.child;
    }
  }
}
```

Class komponent `stateNode` — class instance saqlanadi (Fiber'da). Function komponent `stateNode` null (faqat `memoizedState` Hook linked list).

`Component.prototype.isReactComponent` — class identifier:

```javascript
class Component { /* ... */ }
Component.prototype.isReactComponent = {};

if (Component.prototype && Component.prototype.isReactComponent) {
  // Class component
}
```

Bu prototype property — `extends React.Component`'dan kelgan inheritance bilan keladi.

`PureComponent` — `Component`'dan farq: `shouldComponentUpdate` default `shallowEqual` props/state comparison. Bailout `React.memo`'ga o'xshash.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Class komponent counter:

```tsx
import React from 'react';

interface CounterProps {
  initial?: number;
  step?: number;
}

interface CounterState {
  count: number;
}

const DEFAULT_INITIAL = 0;
const DEFAULT_STEP = 1;

class Counter extends React.Component<CounterProps, CounterState> {
  state: CounterState = {
    count: this.props.initial ?? DEFAULT_INITIAL,
  };
  
  increment = () => {
    this.setState(prev => ({ count: prev.count + (this.props.step ?? DEFAULT_STEP) }));
  };
  
  decrement = () => {
    this.setState(prev => ({ count: prev.count - (this.props.step ?? DEFAULT_STEP) }));
  };
  
  reset = () => {
    this.setState({ count: this.props.initial ?? DEFAULT_INITIAL });
  };
  
  render() {
    return (
      <div>
        <button onClick={this.decrement}>−</button>
        <span>{this.state.count}</span>
        <button onClick={this.increment}>+</button>
        <button onClick={this.reset}>Reset</button>
      </div>
    );
  }
}

<Counter initial={10} step={5} />
```

Class component with lifecycle:

```tsx
interface TimerState {
  time: number;
}

class Timer extends React.Component<{}, TimerState> {
  state: TimerState = { time: 0 };
  intervalId: ReturnType<typeof setInterval> | null = null;
  
  componentDidMount() {
    this.intervalId = setInterval(() => {
      this.setState(prev => ({ time: prev.time + 1 }));
    }, 1000);
  }
  
  componentWillUnmount() {
    if (this.intervalId !== null) {
      clearInterval(this.intervalId);
    }
  }
  
  render() {
    return <div>Time: {this.state.time}s</div>;
  }
}
```

Class vs Function ekvivalent:

```tsx
// Class version
class CounterClass extends React.Component<{}, { count: number }> {
  state = { count: 0 };
  
  componentDidMount() {
    document.title = `Count: ${this.state.count}`;
  }
  
  componentDidUpdate() {
    document.title = `Count: ${this.state.count}`;
  }
  
  componentWillUnmount() {
    document.title = 'React App';
  }
  
  render() {
    return (
      <div>
        <p>Count: {this.state.count}</p>
        <button onClick={() => this.setState({ count: this.state.count + 1 })}>+1</button>
      </div>
    );
  }
}

// Function version (ekvivalent)
function CounterFunction() {
  const [count, setCount] = React.useState(0);
  
  React.useEffect(() => {
    document.title = `Count: ${count}`;
    return () => {
      document.title = 'React App';
    };
  }, [count]);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}
```

Function variant — qisqa, lifecycle method'lar bir `useEffect`'da.

</details>

---

## Class Lifecycle Methods — Hooks Equivalents

### Nazariya

Class komponent **lifecycle methods** — har bir bosqichda chaqiriladigan maxsus method'lar. Hooks (R16.8) bularning aksariyatini almashtirdi.

**Mount lifecycle:**

| Class Method | Hooks Equivalent |
|--------------|------------------|
| `constructor` | `useState`/`useReducer` (lazy init) |
| `componentDidMount` | `useEffect(() => {...}, [])` |
| `getDerivedStateFromProps` | Render bevosita (no hook needed) |

**Update lifecycle:**

| Class Method | Hooks Equivalent |
|--------------|------------------|
| `shouldComponentUpdate` | `React.memo`, `useMemo` |
| `componentDidUpdate` | `useEffect(() => {...}, [deps])` |
| `getSnapshotBeforeUpdate` | `useLayoutEffect` (read-then-write pattern) |

**Unmount lifecycle:**

| Class Method | Hooks Equivalent |
|--------------|------------------|
| `componentWillUnmount` | `useEffect` cleanup return |

**Error handling:**

| Class Method | Hooks Equivalent |
|--------------|------------------|
| `componentDidCatch` | ❌ Hook yo'q (R19+ ham hooks bilan amalga oshirib bo'lmaydi) |
| `getDerivedStateFromError` | ❌ Hook yo'q |

**Eng muhim:** Error Boundary lifecycle method'lari (`componentDidCatch`/`getDerivedStateFromError`) hooks bilan ekvivalent yo'q. Shu sababli class komponent **hozir ham kerak**.

> **Versiya evolyutsiyasi (Class Lifecycle Methods → Hooks):**
> - **Pre-R16.8 (2018):** Class lifecycle methods majburiy (`componentDidMount`, `componentDidUpdate`, `componentWillUnmount`, `componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate`).
> - **R16.8+ (Hooks):** `useEffect`, `useLayoutEffect`, `useState`, `useReducer` lifecycle method'larning aksariyatini almashtirdi. **Sabab:** lifecycle method'lar logic reuse'ni qiyinlashtirardi (HOC, render props bilan kurashish), `this` binding murakkab edi, lifecycle "soup" — bir method'da turli concerns aralash. **MUHIM:** `useEffect` lifecycle method'ning **mexanik almashtiruvi emas** — sync model (cross-ref [`16-useeffect.md`](16-useeffect.md)).
> - **R19+:** Error boundaries hali class shart (`getDerivedStateFromError`/`componentDidCatch`). Hook ekvivalenti yo'q — `react-error-boundary` kutubxonasi function komponentlar uchun wrapper beradi (ichida class ishlatadi).

NIMA UCHUN `useEffect` lifecycle equivalent EMAS:

```tsx
// ❌ Misconception — useEffect == componentDidMount + componentDidUpdate + componentWillUnmount
useEffect(() => {
  // mount + update logic
  return () => {
    // unmount logic
  };
}, [/* deps */]);
```

Bu **mental model** notog'ri. `useEffect` — **synchronization** mexanizmi (cross-ref [`16-useeffect.md`](16-useeffect.md)). Har dependency change'da effect re-run (cleanup + setup). Lifecycle method'da componentDidUpdate bir marta chaqiriladi — useEffect har deps change'da ikki phase (cleanup eski + setup yangi).

QANDAY ISHLAYDI:

```
Class Lifecycle:
  Mount: constructor → render → componentDidMount
  Update: getDerivedStateFromProps → shouldComponentUpdate → render → getSnapshotBeforeUpdate → componentDidUpdate
  Unmount: componentWillUnmount

Hooks (function):
  Render: useState/useReducer/useMemo/useCallback evaluate
  Commit: useLayoutEffect cleanup (prev) → useLayoutEffect setup
  Paint
  After paint: useEffect cleanup (prev) → useEffect setup
  Unmount: useEffect cleanup
```

`useEffect` async — paint'dan keyin. `useLayoutEffect` sync — paint'dan oldin (cross-ref [`17-uselayouteffect.md`](17-uselayouteffect.md)). Class `componentDidMount/Update` — `useLayoutEffect`'ga yaqin (sync, paint'dan oldin).

<details>
<summary><strong>Under the Hood</strong></summary>

Lifecycle methods chaqirish tartib (Mount):

```
1. constructor(props)
   - state initialization
   - method binding

2. static getDerivedStateFromProps(props, state)
   - props → state derivation (rare use case)

3. render()
   - JSX qaytaradi

4. // React commits to DOM

5. componentDidMount()
   - DOM access OK
   - subscriptions, fetch
```

Update tartib:

```
1. static getDerivedStateFromProps(props, state)

2. shouldComponentUpdate(nextProps, nextState)
   - return false → render skip (optimization)

3. render()

4. getSnapshotBeforeUpdate(prevProps, prevState)
   - DOM read before commit (scroll position)
   - return value passed to componentDidUpdate

5. // React commits to DOM

6. componentDidUpdate(prevProps, prevState, snapshot)
   - DOM updated
   - side effects
```

Unmount:

```
1. componentWillUnmount()
   - cleanup subscriptions, timers
```

Hooks lifecycle (cross-ref [`17-uselayouteffect.md`](17-uselayouteffect.md)):

```
Render Phase:
  useState/useReducer evaluate state
  useMemo evaluate (if deps changed)
  useCallback (if deps changed)
  
Commit Phase (Mutation sub-phase):
  useInsertionEffect (R18+) — CSS-in-JS

Commit Phase (Layout sub-phase):
  useLayoutEffect cleanup (prev render)
  ref attach
  useLayoutEffect setup

Browser Paint

Passive:
  useEffect cleanup (prev render)
  useEffect setup
```

Class va hooks lifecycle taxminan ekvivalent, lekin mental model boshqacha:
- **Class**: discrete events ("mount qildi", "update bo'ldi")
- **Hooks**: sync state ("state X ga sync, deps change'da resync")

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Lifecycle method'lar va hooks ekvivalent:

```tsx
interface Product {
  id: string;
  name: string;
}

interface ProductListState {
  products: Product[];
  loading: boolean;
}

// === Class version ===
class ProductListClass extends React.Component<{ url: string }, ProductListState> {
  state: ProductListState = { products: [], loading: true };
  
  componentDidMount() {
    this.fetchProducts();
  }
  
  componentDidUpdate(prevProps: { url: string }) {
    if (prevProps.url !== this.props.url) {
      this.fetchProducts();
    }
  }
  
  componentWillUnmount() {
    // No subscriptions in this example
  }
  
  fetchProducts = async () => {
    this.setState({ loading: true });
    const response = await fetch(this.props.url);
    const products: Product[] = await response.json();
    this.setState({ products, loading: false });
  };
  
  render() {
    if (this.state.loading) return <p>Loading...</p>;
    return <ul>{this.state.products.map(product => <li key={product.id}>{product.name}</li>)}</ul>;
  }
}

// === Hooks version ===
function ProductListHooks({ url }: { url: string }) {
  const [products, setProducts] = React.useState<Product[]>([]);
  const [loading, setLoading] = React.useState(true);
  
  React.useEffect(() => {
    let cancelled = false;
    
    setLoading(true);
    fetch(url)
      .then(response => response.json())
      .then((data: Product[]) => {
        if (!cancelled) {
          setProducts(data);
          setLoading(false);
        }
      });
    
    return () => { cancelled = true; };
  }, [url]);
  
  if (loading) return <p>Loading...</p>;
  return <ul>{products.map(product => <li key={product.id}>{product.name}</li>)}</ul>;
}
```

Hooks variant — boilerplate kamroq, race condition handling (`cancelled` flag) tabiiyroq.

`getSnapshotBeforeUpdate` — DOM o'zgarishidan oldin DOM read:

```tsx
class ScrollList extends React.Component<{ items: string[] }, { scrollTop: number }> {
  listRef = React.createRef<HTMLUListElement>();
  state = { scrollTop: 0 };
  
  getSnapshotBeforeUpdate(prevProps: { items: string[] }) {
    if (prevProps.items.length < this.props.items.length) {
      const list = this.listRef.current;
      return list ? list.scrollHeight - list.scrollTop : null;
    }
    return null;
  }
  
  componentDidUpdate(
    _prevProps: { items: string[] },
    _prevState: { scrollTop: number },
    snapshot: number | null,
  ) {
    if (snapshot !== null && this.listRef.current) {
      this.listRef.current.scrollTop = this.listRef.current.scrollHeight - snapshot;
    }
  }
  
  render() {
    return (
      <ul ref={this.listRef}>
        {this.props.items.map((item, i) => <li key={i}>{item}</li>)}
      </ul>
    );
  }
}

// Hooks equivalent — useLayoutEffect read-then-write
function ScrollListHooks({ items }: { items: string[] }) {
  const listRef = React.useRef<HTMLUListElement>(null);
  const prevItemsLengthRef = React.useRef(items.length);
  
  React.useLayoutEffect(() => {
    const list = listRef.current;
    if (!list) return;
    
    if (prevItemsLengthRef.current < items.length) {
      const distanceFromBottom = list.scrollHeight - list.scrollTop;
      list.scrollTop = list.scrollHeight - distanceFromBottom;
    }
    
    prevItemsLengthRef.current = items.length;
  }, [items.length]);
  
  return <ul ref={listRef}>{items.map((item, i) => <li key={i}>{item}</li>)}</ul>;
}
```

</details>

---

## `UNSAFE_*` Lifecycle Methods (Deprecated)

### Nazariya

R16.3+ uchta lifecycle method `UNSAFE_*` prefix'i bilan **deprecated** belgilandi:

1. **`UNSAFE_componentWillMount`** (eski: `componentWillMount`) — render'dan oldin chaqiriladi.
2. **`UNSAFE_componentWillReceiveProps`** (eski: `componentWillReceiveProps`) — props change'da chaqiriladi.
3. **`UNSAFE_componentWillUpdate`** (eski: `componentWillUpdate`) — re-render'dan oldin chaqiriladi.

NIMA UCHUN deprecated:

1. **Concurrent Rendering bilan mos emas** — R18+ Concurrent Mode render Phase'ni interrupt qiladi va restart qilishi mumkin (cross-ref [`05-scheduler-lanes.md`](05-scheduler-lanes.md)). `componentWillMount` yoki `componentWillUpdate` ichida side effect (fetch, subscription) **bir necha marta chaqirilishi mumkin** — buggy behavior.

2. **Misuse** — bu method'lar ko'p hollarda noto'g'ri ishlatilardi:
   - `componentWillMount` ichida `setState` — render lifecycle dirty.
   - `componentWillReceiveProps` ichida API call — race condition'lar.
   - `componentWillUpdate` ichida DOM read — yangi DOM yo'q hali.

**Yangi alternative'lar:**

| Eski | Yangi |
|------|-------|
| `componentWillMount` | `componentDidMount` yoki `constructor` |
| `componentWillReceiveProps` | `getDerivedStateFromProps` (static) yoki `componentDidUpdate` |
| `componentWillUpdate` | `getSnapshotBeforeUpdate` + `componentDidUpdate` |

Hooks bilan barchasi `useEffect` patterns'ga ko'chadi.

> **Versiya evolyutsiyasi (`UNSAFE_*` Lifecycle Methods):**
> - **Pre-R16.3:** `componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate` — standart lifecycle methods.
> - **R16.3 (2018):** `UNSAFE_*` prefix qo'shildi (deprecation warning). Eski API hali ishlaydi (backward compat).
> - **R17+:** Eski API'lar console warning beradi.
> - **R19:** Hali bor, lekin Concurrent rendering'da unpredictable. Codemod migrate.

QANDAY ISHLAYDI:

```javascript
// React Reconciler (simplified)
function commitClassComponent(workInProgress, instance) {
  if (instance.componentWillMount && !instance.UNSAFE_componentWillMount) {
    console.warn(
      'componentWillMount is deprecated. Use componentDidMount or constructor.'
    );
  }
  
  // Concurrent rendering interrupt scenario
  if (renderRestarted) {
    // componentWillMount may have side effects
    // Bug: side effect runs multiple times
  }
}
```

StrictMode `UNSAFE_*` lifecycle method'lar ishlatilganda development'da console warning beradi — migration kerakligini bildirish uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

`componentWillMount` Concurrent rendering buggy scenario:

```javascript
class BadComponent extends React.Component {
  // ❌ UNSAFE — fetch every render
  UNSAFE_componentWillMount() {
    fetch('/api/data').then(r => this.setState({ data: r }));
    // Concurrent rendering interrupts → restart → fetch called again
    // Result: bir "mount" uchun fetch bir necha marta chaqiriladi
  }
  
  componentDidMount() {
    // ✅ SAFE — called once after commit
    fetch('/api/data').then(r => this.setState({ data: r }));
  }
}
```

R18+ Concurrent Mode render Phase'ni har joyda yo'qotishi mumkin (yield to higher priority). `UNSAFE_*` ichidagi side effects multiple times chaqiriladi.

`componentWillReceiveProps` race condition:

```javascript
class ProfilePage extends React.Component {
  UNSAFE_componentWillReceiveProps(nextProps) {
    if (nextProps.userId !== this.props.userId) {
      fetch(`/api/users/${nextProps.userId}`).then(user => {
        this.setState({ user });  // Race condition possible
      });
    }
  }
}

// Scenario:
// 1. props.userId = 1 → fetch /api/users/1 (slow)
// 2. props.userId = 2 → fetch /api/users/2 (fast, returns first)
// 3. setState({ user: user2 }) — UI shows user 2
// 4. setState({ user: user1 }) — UI shows user 1 (stale, wrong)
```

Modern alternative — `componentDidUpdate` + AbortController, yoki `useEffect` + cleanup.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Migration `UNSAFE_*` → modern:

```tsx
interface User {
  id: string;
  name: string;
}

// ❌ Pre-R16.3 — UNSAFE
class OldUserCard extends React.Component<{ userId: string }, { user: User | null }> {
  state = { user: null };
  
  UNSAFE_componentWillMount() {
    fetch(`/api/users/${this.props.userId}`)
      .then(response => response.json())
      .then(user => this.setState({ user }));
  }
  
  UNSAFE_componentWillReceiveProps(nextProps: { userId: string }) {
    if (nextProps.userId !== this.props.userId) {
      fetch(`/api/users/${nextProps.userId}`)
        .then(response => response.json())
        .then(user => this.setState({ user }));
    }
  }
  
  render() {
    return <div>{this.state.user?.name}</div>;
  }
}

// ✅ R16.3+ — Modern class
class NewUserCard extends React.Component<{ userId: string }, { user: User | null }> {
  state: { user: User | null } = { user: null };
  controller: AbortController | null = null;
  
  componentDidMount() {
    this.fetchUser();
  }
  
  componentDidUpdate(prevProps: { userId: string }) {
    if (prevProps.userId !== this.props.userId) {
      this.controller?.abort();
      this.fetchUser();
    }
  }
  
  componentWillUnmount() {
    this.controller?.abort();
  }
  
  fetchUser = async () => {
    this.controller = new AbortController();
    try {
      const response = await fetch(`/api/users/${this.props.userId}`, { 
        signal: this.controller.signal 
      });
      const user: User = await response.json();
      this.setState({ user });
    } catch (err) {
      if (err instanceof Error && err.name === 'AbortError') return;
      throw err;
    }
  };
  
  render() {
    return <div>{this.state.user?.name}</div>;
  }
}

// ✅ R16.8+ — Hooks
function UserCardHooks({ userId }: { userId: string }) {
  const [user, setUser] = React.useState<User | null>(null);
  
  React.useEffect(() => {
    const controller = new AbortController();
    
    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then(response => response.json())
      .then((data: User) => setUser(data))
      .catch((err: Error) => {
        if (err.name !== 'AbortError') throw err;
      });
    
    return () => controller.abort();
  }, [userId]);
  
  return <div>{user?.name}</div>;
}
```

Hooks variant — eng qisqa, race condition handling tabiiy (cleanup return), Concurrent-safe.

</details>

---

## Error Boundary Concept va API

### Nazariya

**Error Boundary** — ikki lifecycle method'dan biri yoki ikkalasini implement qiluvchi class komponent:

1. **`static getDerivedStateFromError(error)`** — Render Phase'da chaqiriladi. Yangi state qaytaradi (re-render trigger).
2. **`componentDidCatch(error, info)`** — Commit Phase'da chaqiriladi (browser paint'idan oldin, sinxron). Error logging uchun.

```tsx
class ErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };
  
  static getDerivedStateFromError(_error: Error) {
    return { hasError: true };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('Error caught:', error);
    console.error('Component stack:', info.componentStack);
  }
  
  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

API signature:

```tsx
interface ErrorInfo {
  componentStack: string;
  digest?: string;  // R18+ for SSR errors
}

interface ErrorBoundaryComponent {
  getDerivedStateFromError?: (error: Error) => Partial<State> | null;
  componentDidCatch?: (error: Error, info: ErrorInfo) => void;
}
```

Ikkala method'ning vazifalari farq:

| Method | Phase | Vazifa | SSR |
|--------|-------|--------|-----|
| `getDerivedStateFromError` | Render | State update | ✅ Ishlaydi |
| `componentDidCatch` | Commit (paint'dan oldin, sinxron) | Logging | ❌ Server'da chaqirilmaydi |

NIMA UCHUN ikki method:

- **`getDerivedStateFromError`** — pure (no side effects), state update via return. Server'da ham ishlaydi (SSR error handling).
- **`componentDidCatch`** — side effects ruxsat etilgan (logging, analytics). Faqat client'da chaqiriladi.

QANDAY ISHLAYDI:

```
Component subtree throws error
       │
       ▼
React catches in render
       │
       ▼
Find nearest ErrorBoundary up the tree
       │
       ├─ Found:
       │   ├─ Call getDerivedStateFromError(error) → state update
       │   ├─ Re-render with hasError: true → fallback UI
       │   └─ Call componentDidCatch(error, info) → logging (commit, paint'dan oldin)
       │
       └─ Not found:
           ├─ R15 va eski: silent corruption (boundaries yo'q edi)
           ├─ R16+: entire tree unmounts (default behavior)
           └─ R19: onUncaughtError callback (root-level)
```

> **Versiya evolyutsiyasi (Error Boundary API):**
> - **R16 (2017):** `componentDidCatch(error, info)` kiritildi.
> - **R16.6 (2018):** `getDerivedStateFromError(error)` qo'shildi — SSR-compatible state update.
> - **R18 (2022):** `digest` field SSR error'lari uchun (hydration mismatch). `onRecoverableError` root option qo'shildi.
> - **R19 (2024):** Root-level callbacks (`onCaughtError`/`onUncaughtError`/`onRecoverableError`) — application-wide error reporting.

<details>
<summary><strong>Under the Hood</strong></summary>

Error Boundary lifecycle Reconciler ichida:

```javascript
// react-reconciler/src/ReactFiberWorkLoop.js (simplified)
function handleError(workInProgress, error) {
  let boundary = workInProgress.return;
  while (boundary !== null) {
    if (
      boundary.tag === ClassComponent &&
      typeof boundary.type.getDerivedStateFromError === 'function'
    ) {
      break;
    }
    if (
      boundary.tag === ClassComponent &&
      typeof boundary.stateNode.componentDidCatch === 'function'
    ) {
      break;
    }
    boundary = boundary.return;
  }
  
  if (boundary !== null) {
    if (typeof boundary.type.getDerivedStateFromError === 'function') {
      const newState = boundary.type.getDerivedStateFromError(error);
      boundary.memoizedState = { ...boundary.memoizedState, ...newState };
    }
    boundary.flags |= DidCapture;
    
    boundary.deferredErrorInfo = {
      error,
      componentStack: getComponentStack(workInProgress),
    };
    
    scheduleUpdateOnFiber(boundary);
  } else {
    if (root.onUncaughtError) {
      root.onUncaughtError(error, getComponentStack(workInProgress));
    }
    throw error;
  }
}

function commitErrorBoundary(boundary) {
  if (boundary.deferredErrorInfo) {
    const { error, componentStack } = boundary.deferredErrorInfo;
    boundary.stateNode.componentDidCatch?.(error, { componentStack });
    boundary.deferredErrorInfo = null;
  }
}
```

Boundary search — Fiber `return` chain. Component stack — `getComponentStack` traverses Fiber tree.

`flags |= DidCapture` — boundary fallback'ni render qilishini bildiradi (Reconciler'ga signal).

R19 root callbacks integration:

```javascript
function createRoot(container, options) {
  return {
    render(children) { /* ... */ },
    onCaughtError: options?.onCaughtError ?? defaultLogError,
    onUncaughtError: options?.onUncaughtError ?? defaultThrowError,
    onRecoverableError: options?.onRecoverableError ?? defaultLogError,
  };
}

function callBoundaryCallbacks(boundary, error, info) {
  boundary.stateNode.componentDidCatch?.(error, info);
  
  if (root.onCaughtError) {
    root.onCaughtError(error, info);
  }
}
```

R19 — boundary'lar va root callbacks ikkalasi ham chaqiriladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq Error Boundary implementation:

```tsx
import React from 'react';

interface ErrorBoundaryProps {
  children: React.ReactNode;
  fallback?: React.ReactNode | ((error: Error) => React.ReactNode);
  onError?: (error: Error, info: React.ErrorInfo) => void;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends React.Component<ErrorBoundaryProps, ErrorBoundaryState> {
  state: ErrorBoundaryState = {
    hasError: false,
    error: null,
  };
  
  static getDerivedStateFromError(error: Error): Partial<ErrorBoundaryState> {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('ErrorBoundary caught:', error);
    console.error('Component stack:', info.componentStack);
    
    this.props.onError?.(error, info);
  }
  
  render() {
    if (this.state.hasError && this.state.error) {
      const { fallback } = this.props;
      
      if (typeof fallback === 'function') {
        return fallback(this.state.error);
      }
      
      return fallback ?? <DefaultFallback error={this.state.error} />;
    }
    
    return this.props.children;
  }
}

function DefaultFallback({ error }: { error: Error | null }) {
  return (
    <div className="error-fallback" role="alert">
      <h2>Something went wrong</h2>
      <p>{error?.message ?? 'Unknown error'}</p>
    </div>
  );
}

// Usage
<ErrorBoundary 
  fallback={(err) => <p>Error: {err.message}</p>}
  onError={(err, info) => logErrorToService(err, info)}
>
  <RiskyComponent />
</ErrorBoundary>
```

Granular Error Boundaries:

```tsx
function App() {
  return (
    <Layout>
      <ErrorBoundary fallback={<HeaderFallback />}>
        <Header />
      </ErrorBoundary>
      
      <ErrorBoundary fallback={<SidebarFallback />}>
        <Sidebar />
      </ErrorBoundary>
      
      <ErrorBoundary fallback={<MainFallback />}>
        <MainContent />
      </ErrorBoundary>
      
      <ErrorBoundary fallback={<FooterFallback />}>
        <Footer />
      </ErrorBoundary>
    </Layout>
  );
}
```

Har section o'z boundary'iga ega — bir section error bersa, qolganlari ishlamoqda davom etadi.

</details>

---

## `getDerivedStateFromError` Static Method

### Nazariya

`getDerivedStateFromError` — **static method** (instance emas, class'ning o'zida). Render Phase'da chaqiriladi va yangi state qaytaradi.

```tsx
static getDerivedStateFromError(error: Error): Partial<State> | null {
  return { hasError: true };
}
```

**Static** — `this` mavjud emas. Faqat error argumenti bilan ishlash mumkin. Side effects **TAQIQ** (pure function).

NIMA UCHUN static:

1. **SSR-compatible** — server'da side effects yo'q, faqat state derivation.
2. **Pure** — har xato uchun deterministic state update.
3. **No `this` access** — instance method'lardan farqli, side effect mumkin emas.

QANDAY ISHLAYDI:

```
Render error in subtree
       │
       ▼
React calls Boundary.getDerivedStateFromError(error)
       │
       ▼
Method returns partial state: { hasError: true }
       │
       ▼
Boundary re-renders with new state
       │
       ▼
render() returns fallback UI
```

`getDerivedStateFromError` — Render Phase'da. Side effect yo'q. State update orqali fallback render trigger.

**Eng muhim qoida — pure**:

```tsx
// ❌ Anti-pattern — side effect in static method
static getDerivedStateFromError(error: Error) {
  console.error(error);  // ❌ side effect — TAQIQ
  fetch('/api/log', { ... });  // ❌ side effect — TAQIQ
  return { hasError: true };
}

// ✅ To'g'ri — pure
static getDerivedStateFromError(error: Error) {
  return { hasError: true, error };
}
```

Side effects (logging, fetch) `componentDidCatch`'da bajariladi (instance method, side effect ruxsat).

**Return value**:

- `Partial<State>` — yangi state (existing state bilan merge).
- `null` — state o'zgarmaydi.

```tsx
// Most common pattern
static getDerivedStateFromError(error: Error) {
  return { hasError: true, error };
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

`getDerivedStateFromError` SSR'da:

```javascript
// react-dom/src/server/ReactDOMFizzServer.js (simplified)
function renderClassComponent(workInProgress, error) {
  if (error) {
    const newState = Component.getDerivedStateFromError(error);
    if (newState !== null) {
      instance.state = { ...instance.state, ...newState };
      return instance.render();
    }
  }
}
```

Server-side rendering xato bersa, `getDerivedStateFromError` chaqiriladi va fallback HTML server'dan yuboriladi. Client'da hydration bilan boundary state synced.

`componentDidCatch` SSR'da chaqirilmaydi (Commit Phase yo'q server'da).

R18+ Streaming SSR + Suspense Boundary kombinatsiyasi:

```tsx
<Suspense fallback={<Spinner />}>
  <ErrorBoundary fallback={<ErrorMessage />}>
    <AsyncComponent />  {/* throws error */}
  </ErrorBoundary>
</Suspense>
```

- Suspense ushlaydi: throw'd Promise (async data)
- ErrorBoundary ushlaydi: throw'd Error

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

State update with error info:

```tsx
interface State {
  hasError: boolean;
  error: Error | null;
  errorTimestamp: number | null;
}

class DetailedErrorBoundary extends React.Component<{ children: React.ReactNode }, State> {
  state: State = { hasError: false, error: null, errorTimestamp: null };
  
  static getDerivedStateFromError(error: Error): Partial<State> {
    return {
      hasError: true,
      error,
      errorTimestamp: Date.now(),
    };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('Caught:', error);
  }
  
  render() {
    if (this.state.hasError) {
      return (
        <div role="alert">
          <h2>Error</h2>
          <p>{this.state.error?.message}</p>
          {this.state.errorTimestamp !== null && (
            <small>At: {new Date(this.state.errorTimestamp).toLocaleString()}</small>
          )}
        </div>
      );
    }
    return this.props.children;
  }
}
```

Conditional fallback:

```tsx
interface State {
  errorType: 'network' | 'auth' | 'unknown' | null;
}

class TypedErrorBoundary extends React.Component<{ children: React.ReactNode }, State> {
  state: State = { errorType: null };
  
  static getDerivedStateFromError(error: Error): Partial<State> {
    let errorType: State['errorType'] = 'unknown';
    
    if (error.message.includes('NetworkError') || error.message.includes('fetch')) {
      errorType = 'network';
    } else if (error.message.includes('Unauthorized') || error.message.includes('403')) {
      errorType = 'auth';
    }
    
    return { errorType };
  }
  
  render() {
    const { errorType } = this.state;
    
    switch (errorType) {
      case 'network':
        return <NetworkErrorFallback />;
      case 'auth':
        return <AuthErrorFallback />;
      case 'unknown':
        return <GenericErrorFallback />;
      default:
        return this.props.children;
    }
  }
}
```

Error type categorization in static method — pure logic OK.

</details>

---

## `componentDidCatch` Instance Method

### Nazariya

`componentDidCatch(error, info)` — instance method, Commit Phase'da sinxron chaqiriladi (browser paint'idan oldin). **Side effects ruxsat etilgan** — logging, analytics, error reporting.

```tsx
componentDidCatch(error: Error, info: React.ErrorInfo) {
  console.error('Caught:', error);
  console.error('Component stack:', info.componentStack);
  
  Sentry.captureException(error, {
    extra: { componentStack: info.componentStack },
  });
}
```

`React.ErrorInfo`:

```tsx
interface ErrorInfo {
  componentStack: string;
  digest?: string;  // R18+ for hydration errors
}
```

`componentStack` — komponent daraxtidagi xato yo'li (multi-line string):

```
    at BuggyComponent (App.tsx:15:10)
    at div
    at ErrorBoundary (App.tsx:30:5)
    at App (App.tsx:50:3)
```

**Production'da error reporting**:

```tsx
componentDidCatch(error: Error, info: React.ErrorInfo) {
  Sentry.withScope((scope) => {
    scope.setExtras({ componentStack: info.componentStack });
    Sentry.captureException(error);
  });
  
  LogRocket.captureException(error);
  
  fetch('/api/log-error', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      message: error.message,
      stack: error.stack,
      componentStack: info.componentStack,
      url: window.location.href,
      userAgent: navigator.userAgent,
      timestamp: Date.now(),
    }),
  }).catch(() => { /* swallow log errors */ });
}
```

`fetch('/api/log-error')` `.catch(() => {})` — log fetch xato bersa, ilova crash bo'lmasligi uchun.

NIMA UCHUN ikkita method (`getDerivedStateFromError` + `componentDidCatch`):

| Aspect | `getDerivedStateFromError` | `componentDidCatch` |
|--------|----------------------------|---------------------|
| Phase | Render | Commit (paint'dan oldin) |
| `this` | ❌ Static (no `this`) | ✅ Instance |
| Side effects | ❌ Pure | ✅ OK |
| SSR | ✅ Ishlaydi | ❌ Skip |
| Use case | State update | Logging |

QANDAY ISHLAYDI:

```
Render error
       │
       ▼
getDerivedStateFromError(error) ← state update (Render Phase)
       │
       ▼
Re-render boundary with hasError: true → fallback UI
       │
       ▼
Commit Phase:
   ├─ fallback DOM'ga yoziladi
   └─ componentDidCatch(error, info) ← logging (sinxron, commit ichida)
       │
       ▼
Browser paints (fallback UI ko'rinadi)
```

`componentDidCatch` Commit Phase'da, fallback DOM mutation'idan keyin va paint'dan oldin sinxron chaqiriladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`componentDidCatch` chaqirish tartibi:

```javascript
// react-reconciler/src/ReactFiberCommitWork.js (simplified)
function commitErrorBoundary(boundary, errorInfo) {
  const instance = boundary.stateNode;
  
  if (typeof instance.componentDidCatch === 'function') {
    try {
      instance.componentDidCatch(errorInfo.error, {
        componentStack: errorInfo.componentStack,
        digest: errorInfo.digest,
      });
    } catch (errorInCommit) {
      // captureCommitPhaseError — yangi xatoni yuqoridagi eng yaqin
      // boundary'ga ko'taradi (topilmasa root'da uncaught)
      captureCommitPhaseError(boundary, boundary.return, errorInCommit);
    }
  }
}
```

`componentDidCatch` o'zi xato bersa, React uni yutib yubormaydi — `captureCommitPhaseError` orqali shu boundary'ning **yuqorisidagi** eng yaqin Error Boundary'ga ko'tariladi. Topilmasa root darajasiga (uncaught) yetadi. Boundary o'z xatosini o'zi ushlay olmaydi.

R18+ `digest` — hydration mismatch error'lar uchun unique ID (server stack vs client stack farqi).

```tsx
componentDidCatch(error: Error, info: React.ErrorInfo) {
  if (info.digest) {
    console.warn('Hydration mismatch:', info.digest);
  } else {
    console.error('Render error:', error);
  }
}
```

R19 `componentDidCatch` deprecated **emas**, lekin `onCaughtError` root callback alternative.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Production-grade error logging:

```tsx
import * as Sentry from '@sentry/react';

class ProductionErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };
  
  static getDerivedStateFromError(_error: Error) {
    return { hasError: true };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    Sentry.withScope((scope) => {
      scope.setExtras({
        componentStack: info.componentStack,
      });
      scope.setTag('error_boundary', 'true');
      Sentry.captureException(error);
    });
    
    if (window.analytics) {
      window.analytics.track('Error Boundary Triggered', {
        message: error.message,
        component: this.constructor.name,
      });
    }
    
    if (process.env.NODE_ENV === 'development') {
      console.group('Error Boundary');
      console.error(error);
      console.error('Component Stack:', info.componentStack);
      console.groupEnd();
    }
  }
  
  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}
```

Custom logging endpoint:

```tsx
class LoggedErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback: React.ReactNode; userId?: string },
  { hasError: boolean }
> {
  state = { hasError: false };
  
  static getDerivedStateFromError() {
    return { hasError: true };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    // React componentDidCatch return qiymatini await qilmaydi — fetch fire-and-forget
    fetch('/api/error-log', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        message: error.message,
        stack: error.stack,
        componentStack: info.componentStack,
        url: window.location.href,
        userAgent: navigator.userAgent,
        userId: this.props.userId,
        timestamp: new Date().toISOString(),
      }),
    }).catch((logError: unknown) => {
      console.error('Failed to log error:', logError);
    });
  }
  
  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}
```

</details>

---

## Error Catching Scope — Caught va Not Caught

### Nazariya

Error Boundary **render Phase** xatolarini ushlaydi. Quyidagi vaziyatlarda xato **caught**:

| Nima | Caught | Sabab |
|------|--------|-------|
| Render method | ✅ | Render Phase |
| Lifecycle methods | ✅ | Render/Commit Phase |
| Constructor | ✅ | Initialization |
| Hooks (in function components) | ✅ | Render Phase |
| Children render | ✅ | Subtree render |

Quyidagi vaziyatlarda **NOT caught**:

| Nima | Not Caught | Sabab |
|------|------------|-------|
| Event handlers | ❌ | Render Phase tashqarisida |
| Async (`setTimeout`, `Promise.then`) | ❌ | Async — render Phase tashqarisida |
| Server-Side Rendering errors | ❌ (mostly) | SSR'da `componentDidCatch` skip |
| Errors in error boundary itself | ❌ | Boundary qidiruvi parent'dan boshlanadi — yuqoridagi boundary'ga ko'tariladi |

NIMA UCHUN bu cheklov:

1. **Event handler render/commit'dan tashqarida** — handler React'ning render/commit work loop'idan keyin, alohida call stack'da chaqiriladi. Shu sababli throw qilingan xato boundary capture path'iga kirmaydi.
2. **Async code execution context** — `setTimeout(callback, 1000)` callback'i React stack tashqarida bajariladi.
3. **SSR'da Commit Phase yo'q** — `componentDidCatch` faqat Commit Phase'da ishlaydi, server'da Commit Phase bo'lmagani uchun chaqirilmaydi (`getDerivedStateFromError` esa Render Phase'da, SSR'da fallback HTML beradi).
4. **Boundary o'z xatosini o'zi ushlamaydi** — boundary qidiruvi parent fiber'dan (`fiber.return`) boshlanadi, shuning uchun boundary hech qachon o'zining ancestor zanjirida bo'lmaydi. Render yoki `componentDidCatch` xatosi yuqoridagi keyingi boundary'ga ko'tariladi (agar o'zini o'zi ushlasa, fallback render xatosi cheksiz tsiklga olib kelardi).

QANDAY ISHLAYDI: caught/not caught misollar:

```tsx
// ✅ Caught — render Phase
function BuggyRender() {
  return <h1>{undefined.toUpperCase()}</h1>;
}

// ❌ NOT caught — event handler
function BuggyHandler() {
  const handleClick = () => {
    throw new Error('Event handler error');
  };
  return <button onClick={handleClick}>Click</button>;
}

// ❌ NOT caught — async
function BuggyAsync() {
  React.useEffect(() => {
    setTimeout(() => {
      throw new Error('Async error');
    }, 1000);
  }, []);
  return <div>Test</div>;
}

// ✅ Caught — hook in render Phase
function BuggyHook() {
  const [value, setValue] = React.useState(() => {
    throw new Error('Lazy init error');
  });
  return <div>{value}</div>;
}

// ✅ Caught — useEffect setup sync
function BuggyEffect() {
  React.useEffect(() => {
    throw new Error('Sync effect error');
  }, []);
  return <div>Test</div>;
}
```

Hooks lifecycle xatolari (Render Phase + Commit Phase) caught. Async tashqi (setTimeout, Promise.then) — caught EMAS.

> **Versiya evolyutsiyasi (Error Catching Scope):**
> - **R16 (2017):** Render Phase + lifecycle errors caught. Event handlers, async — not caught. Boundary yo'q bo'lsa — butun daraxt unmount.
> - **R18 (2022):** `useTransition` transition funksiyasi ichidagi xato boundary'ga yetadi.
> - **R19 (2024):** `onUncaughtError` root callback for errors that no boundary catches.

<details>
<summary><strong>Under the Hood</strong></summary>

Event handler React synthetic event system tomonidan chaqiriladi — bu render/commit work loop'idan tashqarida, alohida call stack'da bo'ladi:

```javascript
// react-dom-bindings/src/events/DOMPluginEventSystem.js (conceptual)
function dispatchEvent(event) {
  // Handler render/commit fiber traversal'idan tashqarida chaqiriladi.
  // Bu yerda throw qilingan xato Error Boundary capture path'iga
  // (throwException / DidCapture flag) kirmaydi — oddiy JS xato sifatida
  // window.onerror'ga ketadi.
  listener(event);
}
```

Async errors (setTimeout, Promise) — JavaScript event loop bilan bog'liq:

```javascript
setTimeout(() => {
  throw new Error('Async');
}, 100);

fetch('/api').then(() => {
  throw new Error('Promise');
});
```

React render va commit ishini o'z call stack'ida bajaradi (R18+ Concurrent rendering interrupt qilinsa ham, har bir ish birligi shu stack'da ishlaydi). `setTimeout`/`Promise.then` callback'lari esa keyinroq, React stack'idan tashqarida ishga tushadi — shu sababli throw qilingan xato boundary'ga yetib bormaydi.

**Workaround**: manual try/catch + setState:

```tsx
function ManualHandler() {
  const [error, setError] = React.useState<Error | null>(null);
  
  const handleClick = async () => {
    try {
      await riskyAsyncOperation();
    } catch (err) {
      setError(err as Error);
    }
  };
  
  if (error) throw error;
  
  return <button onClick={handleClick}>Click</button>;
}
```

`setError` keyin `if (error) throw error` — state'dan render Phase'ga "translate" qiladi, boundary ushlaydi. Bu **idiomatic pattern**.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Caught vs Not Caught examples:

```tsx
import React from 'react';

class ErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };
  
  static getDerivedStateFromError() {
    return { hasError: true };
  }
  
  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}

// === Caught ===

function CaughtRender({ title }: { title: string | null }) {
  // title=null bilan chaqirilsa, render Phase'da TypeError
  return <h1>{(title as string).toUpperCase()}</h1>;  // ✅ Caught
}

function CaughtLazyInit() {
  const [v] = React.useState(() => {
    throw new Error('Lazy init');  // ✅ Caught
  });
  return <div>{v}</div>;
}

function CaughtEffect() {
  React.useEffect(() => {
    throw new Error('Sync effect');  // ✅ Caught
  }, []);
  return <div />;
}

// === NOT Caught ===

function NotCaughtHandler() {
  const handleClick = () => {
    throw new Error('Event');  // ❌ Not caught
  };
  return <button onClick={handleClick}>Click</button>;
}

function NotCaughtTimeout() {
  React.useEffect(() => {
    setTimeout(() => {
      throw new Error('Timeout');  // ❌ Not caught
    }, 100);
  }, []);
  return <div />;
}

function NotCaughtPromise() {
  React.useEffect(() => {
    fetch('/api/data').then(() => {
      throw new Error('Promise');  // ❌ Not caught
    });
  }, []);
  return <div />;
}

// === Manual handling pattern ===

function ManualErrorPattern() {
  const [error, setError] = React.useState<Error | null>(null);
  
  const handleClick = async () => {
    try {
      const response = await fetch('/api/risky');
      if (!response.ok) throw new Error('API failed');
    } catch (err) {
      setError(err as Error);
    }
  };
  
  React.useEffect(() => {
    const handler = (e: PromiseRejectionEvent) => {
      setError(e.reason);
    };
    window.addEventListener('unhandledrejection', handler);
    return () => window.removeEventListener('unhandledrejection', handler);
  }, []);
  
  if (error) {
    throw error;  // ✅ Re-throw in render — boundary catches
  }
  
  return (
    <div>
      <button onClick={handleClick}>Risky operation</button>
    </div>
  );
}

<ErrorBoundary fallback={<p>Something went wrong</p>}>
  <ManualErrorPattern />
</ErrorBoundary>
```

`window.addEventListener('unhandledrejection')` — global handler for uncaught Promise rejections. Component-level state'ga uzatish.

</details>

---

## Event Handlers va Async Errors — Manual Try/Catch

### Nazariya

Event handler va async error'lar Error Boundary ushlamasligi sababli, **manual try/catch** + `setState` pattern ishlatiladi:

```tsx
function RiskyActionButton() {
  const [error, setError] = React.useState<Error | null>(null);
  
  const handleClick = async () => {
    try {
      await riskyOperation();
    } catch (err) {
      setError(err as Error);
    }
  };
  
  if (error) throw error;
  
  return <button onClick={handleClick}>Click</button>;
}

<ErrorBoundary fallback={<ErrorMessage />}>
  <RiskyActionButton />
</ErrorBoundary>
```

QANDAY ISHLAYDI:

1. Event handler async function chaqiradi.
2. Async function xato bersa, `try/catch` ushlaydi.
3. `setError(err)` — state update.
4. Re-render → `if (error) throw error` — Render Phase'da throw.
5. Error Boundary `getDerivedStateFromError` chaqiriladi.

NIMA UCHUN bu pattern: barcha xato'lar bir umumiy **fallback UI**'ga olib keladi (boundary tomonidan). Komponent ichida fallback UI duplicate qilmaslik kerak.

**Alternative — `useErrorHandler` hook (community pattern):**

```tsx
function useErrorHandler() {
  const [error, setError] = React.useState<Error | null>(null);
  if (error) throw error;
  return setError;
}

function RiskyActionButton() {
  const handleError = useErrorHandler();
  
  const handleClick = async () => {
    try {
      await riskyOperation();
    } catch (err) {
      handleError(err as Error);
    }
  };
  
  return <button onClick={handleClick}>Click</button>;
}
```

`useErrorHandler` — re-throw logic'ni hook'ga qadoqlaydi. `react-error-boundary` library'da `useErrorBoundary` deb nomlanadi.

**Global async errors** — `unhandledrejection` event:

```tsx
function GlobalErrorHandler() {
  const handleError = useErrorHandler();
  
  React.useEffect(() => {
    const handler = (e: PromiseRejectionEvent) => {
      e.preventDefault();
      handleError(e.reason instanceof Error ? e.reason : new Error(String(e.reason)));
    };
    
    window.addEventListener('unhandledrejection', handler);
    return () => window.removeEventListener('unhandledrejection', handler);
  }, [handleError]);
  
  return null;
}

<ErrorBoundary fallback={<ErrorMessage />}>
  <GlobalErrorHandler />
  <App />
</ErrorBoundary>
```

`unhandledrejection` — barcha unhandled Promise rejections'ni ushlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

`window.unhandledrejection` event timing:

```javascript
fetch('/api').then(() => {
  throw new Error('Async');
});

// Microtask queue:
// 1. fetch resolves
// 2. .then callback throws
// 3. Promise rejection
// 4. Browser checks if .catch attached
// 5. No .catch → unhandledrejection event fired
```

`unhandledrejection` browser global event — har domain bo'yicha bir handler.

`e.preventDefault()` — default browser behavior (console error) bloklash. Component'da handle qilish.

R19 root `onUncaughtError` — application-level catch. `unhandledrejection` global handler bilan komplementar.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`useErrorHandler` hook:

```tsx
import React from 'react';

export function useErrorHandler(): (error: unknown) => void {
  const [, setError] = React.useState<Error | null>(null);
  
  return React.useCallback((error: unknown) => {
    setError(() => {
      throw error instanceof Error ? error : new Error(String(error));
    });
  }, []);
}

function ApiButton() {
  const throwError = useErrorHandler();
  const [loading, setLoading] = React.useState(false);
  
  const handleClick = async () => {
    setLoading(true);
    try {
      const response = await fetch('/api/action', { method: 'POST' });
      if (!response.ok) throw new Error(`API error: ${response.status}`);
      const data = await response.json();
      console.log('Success:', data);
    } catch (err) {
      throwError(err);
    } finally {
      setLoading(false);
    }
  };
  
  return (
    <button onClick={handleClick} disabled={loading}>
      {loading ? 'Loading...' : 'Submit'}
    </button>
  );
}

<ErrorBoundary fallback={<p>Something went wrong</p>}>
  <ApiButton />
</ErrorBoundary>
```

`setState` callback — `setError(() => { throw error; })` — yangi state computation paytida throw → Render Phase error → boundary catches.

Global async error handler:

```tsx
function GlobalAsyncErrorHandler({ children }: { children: React.ReactNode }) {
  const throwError = useErrorHandler();
  
  React.useEffect(() => {
    const rejectionHandler = (e: PromiseRejectionEvent) => {
      e.preventDefault();
      throwError(e.reason);
    };
    
    const errorHandler = (e: ErrorEvent) => {
      e.preventDefault();
      throwError(e.error || new Error(e.message));
    };
    
    window.addEventListener('unhandledrejection', rejectionHandler);
    window.addEventListener('error', errorHandler);
    
    return () => {
      window.removeEventListener('unhandledrejection', rejectionHandler);
      window.removeEventListener('error', errorHandler);
    };
  }, [throwError]);
  
  return <>{children}</>;
}

<ErrorBoundary fallback={<GlobalErrorFallback />}>
  <GlobalAsyncErrorHandler>
    <App />
  </GlobalAsyncErrorHandler>
</ErrorBoundary>
```

Form submission with error handling:

```tsx
function ContactForm() {
  const throwError = useErrorHandler();
  const [submitting, setSubmitting] = React.useState(false);
  
  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setSubmitting(true);
    
    try {
      const formData = new FormData(e.currentTarget);
      const response = await fetch('/api/contact', {
        method: 'POST',
        body: formData,
      });
      
      if (!response.ok) {
        throw new Error(`Submit failed: ${response.statusText}`);
      }
    } catch (err) {
      if (err instanceof Error && err.message.includes('Network')) {
        throwError(err);
      } else {
        console.error(err);
      }
    } finally {
      setSubmitting(false);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" required />
      <button disabled={submitting}>Submit</button>
    </form>
  );
}
```

</details>

---

## Error Boundary Placement Strategy

### Nazariya

Error Boundary qayerga joylashtirish — placement strategy. Ikki extreme:

1. **Global boundary** — butun app atrofida bir boundary.
2. **Granular boundaries** — har section, route, feature uchun alohida boundary.

```tsx
// ❌ Faqat global
function App() {
  return (
    <ErrorBoundary fallback={<GlobalError />}>
      <Layout>
        <Header />
        <Sidebar />
        <MainContent />
        <Footer />
      </Layout>
    </ErrorBoundary>
  );
}
// Header'da xato → entire app shows GlobalError

// ✅ Granular
function App() {
  return (
    <ErrorBoundary fallback={<GlobalError />}>
      <Layout>
        <ErrorBoundary fallback={<HeaderError />}>
          <Header />
        </ErrorBoundary>
        
        <ErrorBoundary fallback={<SidebarError />}>
          <Sidebar />
        </ErrorBoundary>
        
        <ErrorBoundary fallback={<MainError />}>
          <MainContent />
        </ErrorBoundary>
        
        <ErrorBoundary fallback={<FooterError />}>
          <Footer />
        </ErrorBoundary>
      </Layout>
    </ErrorBoundary>
  );
}
```

NIMA UCHUN granular afzal:

1. **Localized failure** — bir feature qulasin, qolganlari ishlamoqda davom etadi.
2. **Better UX** — foydalanuvchi nima xato bo'ldi tushunadi.
3. **Recovery** — bir feature reset qilinishi mumkin.

**Placement guidelines:**

- **Per-route boundary** — har route alohida.
- **Per-feature boundary** — feature-specific.
- **Per-widget boundary** — widget'lar (Chart, Map, third-party embed).
- **Top-level safety net** — app root'da bir boundary fallback.

```
App
├── TopLevelBoundary (catch-all)
│   ├── Layout
│   │   ├── HeaderBoundary
│   │   │   └── Header
│   │   ├── SidebarBoundary
│   │   │   └── Sidebar
│   │   ├── RouteBoundary
│   │   │   └── Route Content
│   │   │       └── WidgetBoundary
│   │   │           └── Chart
│   │   └── FooterBoundary
│   │       └── Footer
```

QANDAY ISHLAYDI: xato bo'lsa, React eng yaqin boundary'ni topadi va shu yerga "ko'taradi". Granular boundary'lar — xato faqat shu boundary'gacha "tarqaladi".

<details>
<summary><strong>Under the Hood</strong></summary>

Boundary search — Fiber traversal:

```javascript
function findNearestErrorBoundary(fiber) {
  let current = fiber.return;
  while (current !== null) {
    if (
      current.tag === ClassComponent &&
      (
        typeof current.type.getDerivedStateFromError === 'function' ||
        typeof current.stateNode.componentDidCatch === 'function'
      )
    ) {
      return current;
    }
    current = current.return;
  }
  return null;
}
```

Fiber.return — parent pointer. Eng yaqin (innermost) boundary topiladi.

R18+ Concurrent rendering — Suspense Boundary va Error Boundary ikkalasi bir vaqtda Fiber traversal'da qidiriladi:

```tsx
<ErrorBoundary fallback={<ErrorUI />}>
  <Suspense fallback={<Loader />}>
    <AsyncComponent />  {/* throws Error → ErrorBoundary; throws Promise → Suspense */}
  </Suspense>
</ErrorBoundary>
```

Cross-ref [`29-suspense-lazy.md`](29-suspense-lazy.md) — Suspense Boundary.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Per-route boundary (React Router v6):

```tsx
import { Routes, Route } from 'react-router-dom';

function App() {
  return (
    <ErrorBoundary fallback={<GlobalError />}>
      <Routes>
        <Route 
          path="/" 
          element={
            <ErrorBoundary fallback={<HomePageError />}>
              <HomePage />
            </ErrorBoundary>
          } 
        />
        <Route 
          path="/dashboard" 
          element={
            <ErrorBoundary fallback={<DashboardError />}>
              <Dashboard />
            </ErrorBoundary>
          } 
        />
      </Routes>
    </ErrorBoundary>
  );
}
```

Widget-level boundary:

```tsx
function Dashboard() {
  return (
    <div className="dashboard">
      <ErrorBoundary fallback={<WidgetError name="Stats" />}>
        <StatsWidget />
      </ErrorBoundary>
      
      <ErrorBoundary fallback={<WidgetError name="Chart" />}>
        <ChartWidget />
      </ErrorBoundary>
      
      <ErrorBoundary fallback={<WidgetError name="Activity" />}>
        <ActivityFeed />
      </ErrorBoundary>
    </div>
  );
}

function WidgetError({ name }: { name: string }) {
  return (
    <div className="widget-error" role="alert">
      <p>{name} unavailable</p>
      <button onClick={() => window.location.reload()}>Reload</button>
    </div>
  );
}
```

Lazy-loaded section boundary:

```tsx
const HeavyChart = React.lazy(() => import('./HeavyChart'));

function ChartSection() {
  return (
    <ErrorBoundary fallback={<ChartError />}>
      <Suspense fallback={<ChartSkeleton />}>
        <HeavyChart />
      </Suspense>
    </ErrorBoundary>
  );
}
```

</details>

---

## Error Recovery — `key` Prop bilan Boundary Reset

### Nazariya

Error Boundary xato ushlagandan keyin **fallback UI**'da qoladi. State `hasError: true` saqlanadi. **Recovery** — boundary'ni reset qilish va children'ni qaytadan render qilish.

QANDAY ISHLAYDI:

**Strategy 1: `key` prop bilan boundary remount**

```tsx
function App() {
  const [resetKey, setResetKey] = React.useState(0);
  
  return (
    <>
      <button onClick={() => setResetKey(k => k + 1)}>Retry</button>
      
      <ErrorBoundary key={resetKey} fallback={<ErrorMessage />}>
        <BuggyComponent />
      </ErrorBoundary>
    </>
  );
}
```

`key` change → React unmounts old boundary, mounts new one. State reset → `hasError: false` → children render again.

Bu — **Reconciliation key matching** (cross-ref [`04-reconciliation.md`](04-reconciliation.md)).

**Strategy 2: Internal reset method**

```tsx
class ResettableErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback: (props: { reset: () => void }) => React.ReactNode },
  { hasError: boolean; error: Error | null }
> {
  state = { hasError: false, error: null as Error | null };
  
  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }
  
  reset = () => {
    this.setState({ hasError: false, error: null });
  };
  
  render() {
    if (this.state.hasError) {
      return this.props.fallback({ reset: this.reset });
    }
    return this.props.children;
  }
}

<ResettableErrorBoundary 
  fallback={({ reset }) => (
    <div>
      <p>Something went wrong</p>
      <button onClick={reset}>Try again</button>
    </div>
  )}
>
  <BuggyComponent />
</ResettableErrorBoundary>
```

**Strategy 3: `resetKeys` prop (`react-error-boundary` library)**

```tsx
import { ErrorBoundary } from 'react-error-boundary';

function App() {
  const [userId, setUserId] = React.useState('1');
  
  return (
    <>
      <button onClick={() => setUserId('2')}>User 2</button>
      
      <ErrorBoundary
        FallbackComponent={ErrorFallback}
        resetKeys={[userId]}
      >
        <UserProfile userId={userId} />
      </ErrorBoundary>
    </>
  );
}
```

`resetKeys` array — qaysi value'lar o'zgarsa boundary reset.

NIMA UCHUN bu pattern'lar:

- `key` reset — universal (har boundary uchun ishlaydi).
- Internal reset — explicit user action ("Retry" button).
- `resetKeys` — declarative, props-driven.

<details>
<summary><strong>Under the Hood</strong></summary>

`key` prop reset mexanizmi:

```javascript
// react-reconciler/src/ReactChildFiber.js (simplified)
function reconcileChild(returnFiber, currentChild, newChild, key) {
  if (currentChild !== null && currentChild.key === key) {
    return updateFiber(currentChild, newChild);
  } else {
    deleteChild(returnFiber, currentChild);
    return createFiberFromElement(newChild);
  }
}
```

Yangi `key` — yangi Fiber → yangi instance → fresh state.

`react-error-boundary` `resetKeys` implementation:

```javascript
class ErrorBoundary extends React.Component {
  componentDidUpdate(prevProps) {
    const { resetKeys } = this.props;
    const prevResetKeys = prevProps.resetKeys ?? [];
    
    if (
      resetKeys?.some((key, i) => !Object.is(key, prevResetKeys[i])) ||
      resetKeys?.length !== prevResetKeys.length
    ) {
      this.reset();
    }
  }
  
  reset = () => {
    this.props.onReset?.();
    this.setState({ hasError: false, error: null });
  };
}
```

`Object.is` comparison — primitives va reference equality.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Reset with retry button:

```tsx
class ResettableErrorBoundary extends React.Component<
  {
    children: React.ReactNode;
    fallback: (props: { error: Error; reset: () => void }) => React.ReactElement;
    onReset?: () => void;
  },
  { hasError: boolean; error: Error | null }
> {
  state = { hasError: false, error: null as Error | null };
  
  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('Caught:', error, info);
  }
  
  reset = () => {
    this.props.onReset?.();
    this.setState({ hasError: false, error: null });
  };
  
  render() {
    if (this.state.hasError && this.state.error) {
      return this.props.fallback({ error: this.state.error, reset: this.reset });
    }
    return this.props.children;
  }
}

function ProductPage({ productId }: { productId: string }) {
  return (
    <ResettableErrorBoundary
      onReset={() => console.log('Boundary reset')}
      fallback={({ error, reset }) => (
        <div role="alert">
          <h2>Failed to load product</h2>
          <p>{error.message}</p>
          <button onClick={reset}>Try again</button>
        </div>
      )}
    >
      <ProductDetails productId={productId} />
    </ResettableErrorBoundary>
  );
}
```

`key`-based reset (route navigation):

```tsx
function RouteWithBoundary() {
  const location = useLocation();
  
  return (
    <ErrorBoundary key={location.pathname} fallback={<RouteError />}>
      <Routes>
        <Route path="/users/:id" element={<UserPage />} />
        <Route path="/products/:id" element={<ProductPage />} />
      </Routes>
    </ErrorBoundary>
  );
}
```

`react-error-boundary` library:

```tsx
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }: { error: Error; resetErrorBoundary: () => void }) {
  return (
    <div role="alert" className="error-fallback">
      <h2>Oops!</h2>
      <p>{error.message}</p>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

function App() {
  const [userId, setUserId] = React.useState('1');
  const [retryCount, setRetryCount] = React.useState(0);
  
  return (
    <ErrorBoundary
      FallbackComponent={ErrorFallback}
      resetKeys={[userId, retryCount]}
      onReset={(details) => {
        console.log('Reset triggered', details);
      }}
      onError={(error, info) => {
        Sentry.captureException(error, { extra: { componentStack: info.componentStack } });
      }}
    >
      <UserProfile userId={userId} />
    </ErrorBoundary>
  );
}
```

</details>

---

## Fallback UI Patterns

### Nazariya

Fallback UI — Error Boundary ushladi va children render qilolmaganda ko'rsatiladigan interface. Effective fallback:

1. **Foydalanuvchiga aniq xabar** — "Something went wrong" yoki specific message.
2. **Recovery action** — "Retry" tugmasi yoki link.
3. **Boundary'ga mos** — global vs widget vs route.
4. **Accessibility** — `role="alert"`, screen reader announcement.

Pattern variants:

**Pattern 1: Skeleton-style fallback**:

```tsx
function ProductSkeletonError() {
  return (
    <div className="product-card error">
      <div className="skeleton" aria-hidden="true" />
      <p role="alert" className="error-message">Failed to load product</p>
      <button onClick={() => window.location.reload()}>Reload page</button>
    </div>
  );
}
```

**Pattern 2: Detailed error with stack** (development):

```tsx
function DevErrorFallback({ error }: { error: Error }) {
  if (process.env.NODE_ENV !== 'development') {
    return <ProductionErrorFallback />;
  }
  
  return (
    <div className="dev-error" role="alert">
      <h2>Error</h2>
      <p>{error.message}</p>
      <pre>{error.stack}</pre>
    </div>
  );
}
```

**Pattern 3: Retry with exponential backoff**:

```tsx
function RetryableFallback({ error, reset }: { error: Error; reset: () => void }) {
  const [retryAttempt, setRetryAttempt] = React.useState(0);
  const [autoRetrying, setAutoRetrying] = React.useState(false);
  
  const handleRetry = async () => {
    if (retryAttempt >= 3) return;
    
    setAutoRetrying(true);
    const delay = Math.pow(2, retryAttempt) * 1000;
    await new Promise(r => setTimeout(r, delay));
    
    setRetryAttempt(a => a + 1);
    setAutoRetrying(false);
    reset();
  };
  
  return (
    <div role="alert">
      <p>{error.message}</p>
      <button onClick={handleRetry} disabled={autoRetrying || retryAttempt >= 3}>
        {autoRetrying ? `Retrying...` : `Retry (${retryAttempt}/3)`}
      </button>
      {retryAttempt >= 3 && (
        <p>Max retries reached. Please refresh the page.</p>
      )}
    </div>
  );
}
```

**Pattern 4: Contextual fallback**:

```tsx
function GlobalFallback() {
  return (
    <div className="full-screen-error">
      <h1>Something went wrong</h1>
      <button onClick={() => window.location.reload()}>Reload application</button>
    </div>
  );
}

function RouteFallback() {
  const navigate = useNavigate();
  return (
    <div className="route-error">
      <h2>This page is unavailable</h2>
      <button onClick={() => navigate('/')}>Go home</button>
    </div>
  );
}

function WidgetFallback({ widgetName }: { widgetName: string }) {
  return (
    <div className="widget-error">
      <p>{widgetName} is currently unavailable</p>
    </div>
  );
}
```

NIMA UCHUN: UX consistency. Foydalanuvchi xato turini boundary placement bilan tushunadi.

<details>
<summary><strong>Under the Hood</strong></summary>

Accessibility considerations:

```tsx
// ✅ ARIA role="alert" — screen reader announces immediately
<div role="alert">
  <p>Error message</p>
</div>

// ✅ Focus management — focus error message on appearance
function AccessibleErrorFallback({ error }: { error: Error }) {
  const errorRef = React.useRef<HTMLDivElement>(null);
  
  React.useEffect(() => {
    errorRef.current?.focus();
  }, []);
  
  return (
    <div ref={errorRef} role="alert" tabIndex={-1}>
      <h2>Error</h2>
      <p>{error.message}</p>
    </div>
  );
}
```

`role="alert"` — implicit `aria-live="assertive"`. Screen reader darrov o'qiydi.

`tabIndex={-1}` — focus programmatic ko'chirish.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Multi-tier fallback strategy:

```tsx
import * as Sentry from '@sentry/react';

interface FallbackProps {
  error: Error;
  reset: () => void;
}

function GlobalErrorFallback({ error, reset }: FallbackProps) {
  React.useEffect(() => {
    Sentry.captureException(error);
  }, [error]);
  
  return (
    <div className="global-error" role="alert">
      <h1>Application Error</h1>
      <p>We're sorry, something went wrong. Please try again.</p>
      <div className="actions">
        <button onClick={reset}>Try again</button>
        <button onClick={() => window.location.href = '/'}>Go home</button>
      </div>
      {process.env.NODE_ENV === 'development' && (
        <details className="error-details">
          <summary>Error details (dev only)</summary>
          <pre>{error.stack}</pre>
        </details>
      )}
    </div>
  );
}

function RouteErrorFallback({ error, reset }: FallbackProps) {
  return (
    <div className="route-error" role="alert">
      <h2>This page failed to load</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Reload section</button>
    </div>
  );
}

function WidgetErrorFallback({ name }: { name: string }) {
  return (
    <div className="widget-error" role="alert">
      <p>{name} unavailable</p>
    </div>
  );
}
```

Smart retry with telemetry:

```tsx
function SmartErrorFallback({ error, reset }: FallbackProps) {
  const [retryCount, setRetryCount] = React.useState(0);
  const lastErrorRef = React.useRef<string | null>(null);
  
  React.useEffect(() => {
    if (lastErrorRef.current === error.message) {
      Sentry.captureMessage('Repeated error after retry', {
        level: 'warning',
        extra: { error: error.message, retryCount },
      });
    }
    lastErrorRef.current = error.message;
  }, [error, retryCount]);
  
  const handleRetry = () => {
    setRetryCount(c => c + 1);
    reset();
  };
  
  if (retryCount >= 3) {
    return (
      <div role="alert" className="error-permanent">
        <h2>Persistent error</h2>
        <p>Please refresh the page or contact support.</p>
        <button onClick={() => window.location.reload()}>Refresh page</button>
      </div>
    );
  }
  
  return (
    <div role="alert">
      <p>{error.message}</p>
      <button onClick={handleRetry}>
        Retry ({3 - retryCount} attempts left)
      </button>
    </div>
  );
}
```

</details>

---

## `react-error-boundary` Library

### Nazariya

`react-error-boundary` — community library (Brian Vaughn, ex-React core) — Error Boundary'ning ergonomic wrapper'i. Hook-based reset, declarative fallback components, resetKeys.

API:

```tsx
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary
  FallbackComponent={ErrorFallback}
  onError={(error, info) => Sentry.captureException(error)}
  onReset={(details) => console.log('Reset:', details)}
  resetKeys={[userId, route]}
>
  <App />
</ErrorBoundary>
```

`FallbackComponent` props:
- `error: Error` — caught error
- `resetErrorBoundary: () => void` — reset function

`useErrorBoundary` hook:

```tsx
import { useErrorBoundary } from 'react-error-boundary';

function RiskyActionButton() {
  const { showBoundary } = useErrorBoundary();
  
  const handleClick = async () => {
    try {
      await riskyOperation();
    } catch (err) {
      showBoundary(err);
    }
  };
  
  return <button onClick={handleClick}>Try risky op</button>;
}
```

`showBoundary(error)` — manual error trigger.

NIMA UCHUN library'dan foydalanish:

| Aspect | Manual ErrorBoundary | `react-error-boundary` |
|--------|----------------------|------------------------|
| Implementation | Custom class | Pre-built |
| `resetKeys` | Manual `componentDidUpdate` | Built-in |
| Hook integration | Manual `useErrorHandler` | `useErrorBoundary` |
| TypeScript | Manual types | Library types |
| Bundle | 0 (own code) | Kichik qo'shimcha (dependency'siz) |

Production'da library afzal — battle-tested, edge cases handled, hooks-friendly.

QANDAY ISHLAYDI: library implementation `componentDidCatch` + `getDerivedStateFromError` + `componentDidUpdate` (resetKeys comparison).

```tsx
// react-error-boundary internals (simplified)
class ErrorBoundary extends React.Component {
  state = { error: null };
  
  static getDerivedStateFromError(error) {
    return { error };
  }
  
  componentDidCatch(error, info) {
    this.props.onError?.(error, info);
  }
  
  componentDidUpdate(prevProps) {
    const { resetKeys = [] } = this.props;
    const prevKeys = prevProps.resetKeys ?? [];
    
    if (
      resetKeys.length !== prevKeys.length ||
      resetKeys.some((k, i) => !Object.is(k, prevKeys[i]))
    ) {
      this.reset({ reason: 'keys', prevResetKeys: prevKeys, resetKeys });
    }
  }
  
  reset = (details) => {
    this.props.onReset?.(details);
    this.setState({ error: null });
  };
  
  render() {
    if (this.state.error) {
      const { FallbackComponent, fallbackRender, fallback } = this.props;
      
      const props = {
        error: this.state.error,
        resetErrorBoundary: () => this.reset({ reason: 'imperative-api' }),
      };
      
      if (FallbackComponent) return <FallbackComponent {...props} />;
      if (fallbackRender) return fallbackRender(props);
      if (fallback) return fallback;
      throw new Error('ErrorBoundary requires fallback');
    }
    
    return this.props.children;
  }
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

`react-error-boundary` xususiyatlari:

- Kichik hajmli (kichik bundle qo'shimchasi)
- TypeScript types included
- No external dependencies

API exports:

```tsx
import { 
  ErrorBoundary,
  useErrorBoundary,
  withErrorBoundary,
} from 'react-error-boundary';

const SafeComponent = withErrorBoundary(Component, {
  FallbackComponent: ErrorFallback,
  onError: handleError,
});
```

`withErrorBoundary` — HOC wrapper. `useErrorBoundary` — hook. `ErrorBoundary` — explicit JSX wrapper. Three APIs same library.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Basic usage:

```tsx
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }: { error: Error; resetErrorBoundary: () => void }) {
  return (
    <div role="alert">
      <p>Something went wrong:</p>
      <pre>{error.message}</pre>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

function App() {
  return (
    <ErrorBoundary
      FallbackComponent={ErrorFallback}
      onError={(error, info) => {
        console.error('Error:', error);
        Sentry.captureException(error, { extra: info });
      }}
    >
      <Dashboard />
    </ErrorBoundary>
  );
}
```

`resetKeys` for prop-driven reset:

```tsx
function UserProfilePage() {
  const [userId, setUserId] = React.useState('1');
  
  return (
    <>
      <button onClick={() => setUserId('2')}>Load User 2</button>
      
      <ErrorBoundary
        FallbackComponent={ErrorFallback}
        resetKeys={[userId]}
        onReset={() => console.log('Reset due to userId change')}
      >
        <UserProfile userId={userId} />
      </ErrorBoundary>
    </>
  );
}
```

`useErrorBoundary` hook for async errors:

```tsx
import { useErrorBoundary } from 'react-error-boundary';

function ApiButton() {
  const { showBoundary } = useErrorBoundary();
  
  const handleClick = async () => {
    try {
      const response = await fetch('/api/data');
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
    } catch (err) {
      showBoundary(err);
    }
  };
  
  return <button onClick={handleClick}>Fetch</button>;
}

<ErrorBoundary FallbackComponent={ErrorFallback}>
  <ApiButton />
</ErrorBoundary>
```

`withErrorBoundary` HOC:

```tsx
import { withErrorBoundary } from 'react-error-boundary';

function Dashboard() {
  // ...
}

const SafeDashboard = withErrorBoundary(Dashboard, {
  FallbackComponent: ErrorFallback,
  onError: (error, info) => Sentry.captureException(error),
});

<SafeDashboard />
```

</details>

---

## R19 Root Callbacks — Application-Level Error Handling

### Nazariya

R19 (2024) `createRoot` API'ga uchta error callback qo'shdi — boundary'larsiz application-wide error handling.

```tsx
import { createRoot } from 'react-dom/client';

const container = document.getElementById('root');
if (!container) throw new Error('Root container topilmadi');

const root = createRoot(container, {
  onCaughtError: (error, errorInfo) => {
    console.error('Caught:', error);
    Sentry.captureException(error, { extra: errorInfo });
  },
  
  onUncaughtError: (error, errorInfo) => {
    console.error('Uncaught (CRITICAL):', error);
    Sentry.captureException(error, {
      level: 'fatal',
      extra: errorInfo,
    });
  },
  
  onRecoverableError: (error, errorInfo) => {
    console.warn('Recovered:', error);
    Sentry.captureMessage('Hydration mismatch', {
      level: 'warning',
      extra: errorInfo,
    });
  },
});

root.render(<App />);
```

**Three callbacks:**

| Callback | Trigger | Severity |
|----------|---------|----------|
| `onCaughtError` | Error Boundary caught | Warning |
| `onUncaughtError` | No boundary — app crashes | Fatal |
| `onRecoverableError` | Auto-recovered (hydration mismatch) | Info/Warning |

NIMA UCHUN R19 callbacks:

1. **Centralized error reporting** — har boundary'da `componentDidCatch` yozish kerak emas. Root level bir joy.
2. **Uncaught errors visibility** — boundary yo'q bo'lsa ham log qoldiriladi.
3. **Hydration mismatch monitoring** — R18+ hydration auto-recover qiladi (cross-ref [`06-hydration.md`](06-hydration.md)).

QANDAY ISHLAYDI:

```
Error in subtree
       │
       ├─ Boundary caught
       │   ├─ getDerivedStateFromError → state update
       │   ├─ componentDidCatch → component-level callback
       │   └─ onCaughtError(error, info) → root-level callback (NEW R19)
       │
       └─ No boundary
           └─ onUncaughtError(error, info) → root-level callback (NEW R19)
                  └─ Default: app crashes (entire tree unmounts)

Hydration mismatch
       │
       ├─ R18 (default): silent recovery + console warning
       └─ R19 onRecoverableError(error, info) → reportable
```

> **Versiya evolyutsiyasi (Root Callbacks):**
> - **R16:** Error Boundary (`componentDidCatch`). Boundary yo'q bo'lsa — butun daraxt unmount.
> - **R18:** `onRecoverableError` root option (hydration auto-recovery reporting). `digest` field SSR error'lari uchun.
> - **R19:** Root callbacks to'liq to'plami (`onCaughtError`/`onUncaughtError`/`onRecoverableError`) — centralized error handling.

`errorInfo` argument — `{ componentStack: string }` (xato yuz bergan komponent yo'li). `onRecoverableError`'da ba'zi holatlarda asl sabab `error.cause`'da bo'ladi.

<details>
<summary><strong>Under the Hood</strong></summary>

R19 root callbacks Reconciler integration:

```javascript
// react-dom/src/client/ReactDOMRoot.js (simplified)
function createRoot(container, options = {}) {
  const root = createContainer(container);
  
  root.onCaughtError = options.onCaughtError ?? defaultCaughtErrorHandler;
  root.onUncaughtError = options.onUncaughtError ?? defaultUncaughtErrorHandler;
  root.onRecoverableError = options.onRecoverableError ?? defaultRecoverableErrorHandler;
  
  return {
    render(children) {
      updateContainer(children, root);
    },
    unmount() {
      updateContainer(null, root);
    },
  };
}

function handleError(workInProgress, error) {
  const boundary = findNearestErrorBoundary(workInProgress);
  
  if (boundary) {
    callBoundaryLifecycle(boundary, error);
    
    if (root.onCaughtError) {
      root.onCaughtError(error, {
        componentStack: getComponentStack(workInProgress),
      });
    }
  } else {
    if (root.onUncaughtError) {
      root.onUncaughtError(error, {
        componentStack: getComponentStack(workInProgress),
      });
    }
    throw error;
  }
}
```

User callback'lar default behavior'ni override qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Production setup:

```tsx
import { createRoot } from 'react-dom/client';
import * as Sentry from '@sentry/react';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
});

const container = document.getElementById('root');
if (!container) throw new Error('Root container topilmadi');

const root = createRoot(container, {
  onCaughtError: (error, errorInfo) => {
    Sentry.captureException(error, {
      level: 'warning',
      extra: {
        componentStack: errorInfo.componentStack,
        errorBoundary: 'caught',
      },
    });
  },
  
  onUncaughtError: (error, errorInfo) => {
    Sentry.captureException(error, {
      level: 'fatal',
      extra: {
        componentStack: errorInfo.componentStack,
        errorBoundary: 'none',
      },
    });
    
    // Show emergency UI via safe DOM API (createElement + textContent — XSS-safe alternative)
    const errorDiv = document.createElement('div');
    errorDiv.setAttribute('role', 'alert');
    
    const heading = document.createElement('h1');
    heading.textContent = 'Critical Error';
    
    const message = document.createElement('p');
    message.textContent = 'The application encountered a fatal error.';
    
    const button = document.createElement('button');
    button.textContent = 'Reload';
    button.onclick = () => window.location.reload();
    
    errorDiv.appendChild(heading);
    errorDiv.appendChild(message);
    errorDiv.appendChild(button);
    
    container.replaceChildren(errorDiv);
  },
  
  onRecoverableError: (error, errorInfo) => {
    Sentry.captureMessage('Hydration mismatch', {
      level: 'info',
      extra: {
        error: error.message,
        componentStack: errorInfo.componentStack,
      },
    });
  },
});

root.render(<App />);
```

Combined with Error Boundaries:

```tsx
const root = createRoot(container, {
  onCaughtError: (error, info) => {
    logToService(error, { level: 'warning', boundary: 'caught' });
  },
  onUncaughtError: (error, info) => {
    logToService(error, { level: 'fatal', boundary: 'none' });
  },
});

function App() {
  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <Layout>
        <ErrorBoundary fallback={<RouteError />}>
          <Routes />
        </ErrorBoundary>
      </Layout>
    </ErrorBoundary>
  );
}
```

Boundary va root callback ikkalasi chaqiriladi (boundary'siz xato ham log qoldiriladi).

Development debugging:

```tsx
const root = createRoot(container, {
  onCaughtError: (error, info) => {
    console.group('Caught Error');
    console.error(error);
    console.error('Stack:', info.componentStack);
    console.groupEnd();
  },
  onUncaughtError: (error, info) => {
    console.group('Uncaught Error');
    console.error(error);
    console.error('Stack:', info.componentStack);
    console.groupEnd();
  },
  onRecoverableError: (error) => {
    console.warn('Recovered:', error.message);
  },
});
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: Error Boundary o'zi xato bersa

Boundary `render` method'ida xato bo'lsa — recursive error oldini olish uchun React boundary'ni **upper boundary**'ga ko'taradi yoki crash qiladi.

```tsx
class BadBoundary extends React.Component {
  state = { hasError: false };
  
  static getDerivedStateFromError(error: Error) {
    return { hasError: true };
  }
  
  render() {
    if (this.state.hasError) {
      throw new Error('Fallback render error');  // ❌ recursive
    }
    return this.props.children;
  }
}

<ErrorBoundary fallback={<p>Outer fallback</p>}>
  <BadBoundary>
    <BuggyChild />
  </BadBoundary>
</ErrorBoundary>
```

Production'da fallback UI **simple va safe** bo'lishi shart.

### Gotcha 2: `componentDidCatch` o'zi xato bersa

`componentDidCatch` ichida throw qilingan xatoni React yutib yubormaydi — `captureCommitPhaseError` orqali shu boundary'ning **yuqorisidagi** eng yaqin Error Boundary'ga ko'tariladi. Topilmasa root darajasiga (uncaught) yetadi.

```tsx
class Boundary extends React.Component {
  componentDidCatch(error: Error) {
    throw new Error('Logging failed');  // ❌ Yuqoridagi boundary'ga ko'tariladi (yoki uncaught)
  }
}
```

Boundary o'z `componentDidCatch` xatosini o'zi ushlay olmaydi. Logging code defensive (`.catch(() => {})`) yozilishi kerak.

### Gotcha 3: Hooks bilan amalga oshirib bo'lmaydi (R19'da ham)

`getDerivedStateFromError`/`componentDidCatch` lifecycle method'larining hook ekvivalenti yo'q — R19'da ham Error Boundary class komponent sifatida yoziladi. `react-error-boundary` library `useErrorBoundary` hook'i `showBoundary` beradi — bu **programmatic trigger** (xatoni qo'lda boundary'ga ko'tarish), lekin boundary'ning o'zi (library ichida) hali class komponent.

### Gotcha 4: Boundary `key` o'zgarishi children'larni ham unmount qiladi

```tsx
<ErrorBoundary key={resetKey}>
  <ExpensiveComponent />  {/* Re-mounted on reset (state lost) */}
</ErrorBoundary>
```

Bu **expected behavior** (state reset).

### Gotcha 5: Async errors bilan error boundary'lar

```tsx
// ❌ Async error in event handler — boundary doesn't catch
const handleClick = async () => {
  await riskyOp();
};

// ✅ Manual handling
const handleClick = async () => {
  try {
    await riskyOp();
  } catch (err) {
    throwError(err);
  }
};
```

`useErrorHandler` pattern kerak.

---

## Common Mistakes

### ❌ Xato 1: `getDerivedStateFromError` ichida side effect

```tsx
// ❌ Side effect taqiq
static getDerivedStateFromError(error: Error) {
  console.error(error);  // ❌
  fetch('/api/log', {...});  // ❌
  return { hasError: true };
}

// ✅ Side effects componentDidCatch'da
componentDidCatch(error: Error) {
  console.error(error);
  fetch('/api/log', {...});
}
```

**Sabab:** Static method pure bo'lishi shart (Render Phase, SSR-compatible).

### ❌ Xato 2: Event handler error'larini boundary kutadi

```tsx
// ❌ Boundary kutib bo'lmaydi
function SubmitButton() {
  const handleClick = () => {
    throw new Error('Click error');  // Browser console error, no boundary
  };
}

// ✅ Manual try/catch + throwError
function SubmitButton() {
  const throwError = useErrorHandler();
  const handleClick = () => {
    try {
      // ...
    } catch (err) {
      throwError(err);
    }
  };
}
```

### ❌ Xato 3: Faqat global boundary

```tsx
// ❌ Bir boundary entire app
<ErrorBoundary>
  <App />
</ErrorBoundary>

// ✅ Granular + global safety net
<ErrorBoundary>
  <App>
    <ErrorBoundary>
      <Routes />
    </ErrorBoundary>
    <Sidebar>
      <ErrorBoundary>
        <Widget />
      </ErrorBoundary>
    </Sidebar>
  </App>
</ErrorBoundary>
```

### ❌ Xato 4: `componentDidCatch` ichida xatoni handle qilmaslik

```tsx
// ❌ Sync throw — yuqoridagi boundary'ga ko'tariladi (yoki uncaught)
componentDidCatch(error) {
  throw new Error('Logging failed');
}

// ✅ Defensive — log xatosi yutiladi, boundary buzilmaydi
componentDidCatch(error) {
  fetch('/api/log').catch(() => {});
}
```

`fetch(...).then(() => { throw ... })` — rejection async bo'lgani uchun `componentDidCatch` stack'idan tashqarida, `unhandledrejection`'ga ketadi. Sync throw esa `captureCommitPhaseError` orqali yuqoriga ko'tariladi.

### ❌ Xato 5: SSR'da `componentDidCatch`'ga tayanish

```tsx
// ❌ SSR'da componentDidCatch chaqirilmaydi
componentDidCatch(error) {
  // Server-side will skip this
}

// ✅ getDerivedStateFromError SSR-compatible
static getDerivedStateFromError(error) {
  return { hasError: true };
}
```

---

## Amaliy Mashqlar

### Mashq 1: Sodda Error Boundary (Oson)

`SimpleErrorBoundary` class komponent yarating — `getDerivedStateFromError` + `componentDidCatch`. Fallback prop ReactNode yoki function (error access).

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React from 'react';

interface ErrorBoundaryProps {
  children: React.ReactNode;
  fallback?: React.ReactNode | ((error: Error) => React.ReactNode);
  onError?: (error: Error, info: React.ErrorInfo) => void;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

export class SimpleErrorBoundary extends React.Component<ErrorBoundaryProps, ErrorBoundaryState> {
  state: ErrorBoundaryState = {
    hasError: false,
    error: null,
  };
  
  static getDerivedStateFromError(error: Error): Partial<ErrorBoundaryState> {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    this.props.onError?.(error, info);
  }
  
  render() {
    if (this.state.hasError && this.state.error) {
      const { fallback } = this.props;
      
      if (typeof fallback === 'function') {
        return fallback(this.state.error);
      }
      
      return fallback ?? (
        <div role="alert">
          <p>Something went wrong: {this.state.error.message}</p>
        </div>
      );
    }
    
    return this.props.children;
  }
}

function BuggyComponent({ user }: { user: { name: string } | null }) {
  return <h1>{(user as { name: string }).name}</h1>;  // user=null bilan render Phase TypeError
}

<SimpleErrorBoundary 
  fallback={(err) => <p>Error: {err.message}</p>}
  onError={(err, info) => console.error('Caught:', err, info)}
>
  <BuggyComponent user={null} />
</SimpleErrorBoundary>
```

</details>

### Mashq 2: `useErrorHandler` hook (Oson)

`useErrorHandler` hook yarating — caller `throwError(err)` chaqirsa, error boundary'ga uzatiladi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React from 'react';

export function useErrorHandler(): (error: unknown) => void {
  const [, setError] = React.useState<Error | null>(null);
  
  return React.useCallback((error: unknown) => {
    setError(() => {
      throw error instanceof Error 
        ? error 
        : new Error(String(error));
    });
  }, []);
}

function ApiButton() {
  const throwError = useErrorHandler();
  
  const handleClick = async () => {
    try {
      const response = await fetch('/api/risky');
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
    } catch (err) {
      throwError(err);
    }
  };
  
  return <button onClick={handleClick}>Submit</button>;
}

<SimpleErrorBoundary fallback={<p>Error</p>}>
  <ApiButton />
</SimpleErrorBoundary>
```

</details>

### Mashq 3: Resettable Error Boundary (O'rta)

`ResettableErrorBoundary` yarating — `resetKeys` prop bilan, `componentDidUpdate` ichida resetKeys o'zgarganini detect qiladi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React from 'react';

interface ResettableErrorBoundaryProps {
  children: React.ReactNode;
  fallback: (props: { error: Error; reset: () => void }) => React.ReactElement;
  resetKeys?: unknown[];
  onReset?: () => void;
  onError?: (error: Error, info: React.ErrorInfo) => void;
}

interface ResettableErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

export class ResettableErrorBoundary extends React.Component<
  ResettableErrorBoundaryProps,
  ResettableErrorBoundaryState
> {
  state: ResettableErrorBoundaryState = {
    hasError: false,
    error: null,
  };
  
  static getDerivedStateFromError(error: Error): Partial<ResettableErrorBoundaryState> {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    this.props.onError?.(error, info);
  }
  
  componentDidUpdate(prevProps: ResettableErrorBoundaryProps) {
    const { resetKeys = [] } = this.props;
    const prevKeys = prevProps.resetKeys ?? [];
    
    const keysChanged =
      resetKeys.length !== prevKeys.length ||
      resetKeys.some((key, i) => !Object.is(key, prevKeys[i]));
    
    if (keysChanged && this.state.hasError) {
      this.reset();
    }
  }
  
  reset = () => {
    this.props.onReset?.();
    this.setState({ hasError: false, error: null });
  };
  
  render() {
    if (this.state.hasError && this.state.error) {
      return this.props.fallback({
        error: this.state.error,
        reset: this.reset,
      });
    }
    return this.props.children;
  }
}

function UserPage({ userId }: { userId: string }) {
  return (
    <ResettableErrorBoundary
      resetKeys={[userId]}
      onReset={() => console.log('Boundary reset')}
      fallback={({ error, reset }) => (
        <div role="alert">
          <p>{error.message}</p>
          <button onClick={reset}>Try again</button>
        </div>
      )}
    >
      <UserProfile userId={userId} />
    </ResettableErrorBoundary>
  );
}
```

**Tushuntirish:**
- `componentDidUpdate` resetKeys comparison.
- `Object.is` shallow comparison.
- Reset → `setState({ hasError: false })`.

</details>

### Mashq 4: Async error handling (O'rta)

`AsyncErrorBoundary` — global async error handler. `unhandledrejection` event bilan unhandled Promise rejection'larni ushlaydi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React from 'react';

interface AsyncErrorBoundaryProps {
  children: React.ReactNode;
  fallback: React.ReactNode | ((error: Error) => React.ReactNode);
  onError?: (error: Error, info?: React.ErrorInfo) => void;
}

interface AsyncErrorBoundaryState {
  error: Error | null;
}

export class AsyncErrorBoundary extends React.Component<
  AsyncErrorBoundaryProps,
  AsyncErrorBoundaryState
> {
  state: AsyncErrorBoundaryState = { error: null };
  
  static getDerivedStateFromError(error: Error) {
    return { error };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    this.props.onError?.(error, info);
  }
  
  componentDidMount() {
    window.addEventListener('unhandledrejection', this.handleRejection);
    window.addEventListener('error', this.handleError);
  }
  
  componentWillUnmount() {
    window.removeEventListener('unhandledrejection', this.handleRejection);
    window.removeEventListener('error', this.handleError);
  }
  
  handleRejection = (e: PromiseRejectionEvent) => {
    e.preventDefault();
    const error = e.reason instanceof Error ? e.reason : new Error(String(e.reason));
    this.setState({ error });
    this.props.onError?.(error);
  };
  
  handleError = (e: ErrorEvent) => {
    e.preventDefault();
    const error = e.error instanceof Error ? e.error : new Error(e.message);
    this.setState({ error });
    this.props.onError?.(error);
  };
  
  render() {
    if (this.state.error) {
      const { fallback } = this.props;
      return typeof fallback === 'function' ? fallback(this.state.error) : fallback;
    }
    return this.props.children;
  }
}

function App() {
  return (
    <AsyncErrorBoundary
      fallback={(err) => <p>Error: {err.message}</p>}
      onError={(err) => Sentry.captureException(err)}
    >
      <ComponentWithAsyncError />
    </AsyncErrorBoundary>
  );
}

function ComponentWithAsyncError() {
  const handleClick = () => {
    fetch('/api/error').then(() => {
      throw new Error('Async error');
    });
  };
  
  return <button onClick={handleClick}>Trigger async error</button>;
}
```

**Tushuntirish:**
- Standard render Phase boundary.
- `unhandledrejection` window event — global async error capture.
- `error` window event — runtime errors (not React-related).
- `e.preventDefault()` — browser default block.

</details>

### Mashq 5: Production-grade Error Boundary (Qiyin)

`ProductionErrorBoundary` yarating — Sentry integration (mock), retry mechanism (max 3 attempts, exponential backoff), error categorization, R19 root callback compatibility.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React from 'react';

const mockSentry = {
  captureException: (error: Error, context?: Record<string, unknown>) => {
    console.log('[Sentry] Captured:', error, context);
  },
};

type ErrorCategory = 'network' | 'render' | 'auth' | 'parse' | 'unknown';

interface ErrorContext {
  category: ErrorCategory;
  retryCount: number;
  userId?: string;
  url: string;
  timestamp: number;
}

interface ProductionErrorBoundaryProps {
  children: React.ReactNode;
  userId?: string;
  fallback: (props: {
    error: Error;
    context: ErrorContext;
    retry: () => Promise<void>;
    canRetry: boolean;
  }) => React.ReactElement;
  maxRetries?: number;
}

interface ProductionErrorBoundaryState {
  error: Error | null;
  retryCount: number;
  retrying: boolean;
}

export class ProductionErrorBoundary extends React.Component<
  ProductionErrorBoundaryProps,
  ProductionErrorBoundaryState
> {
  state: ProductionErrorBoundaryState = {
    error: null,
    retryCount: 0,
    retrying: false,
  };
  
  static defaultProps = {
    maxRetries: 3,
  };
  
  static getDerivedStateFromError(error: Error) {
    return { error };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    const context = this.buildErrorContext(error);
    
    mockSentry.captureException(error, {
      extra: {
        componentStack: info.componentStack,
        ...context,
      },
      tags: {
        category: context.category,
        retryCount: context.retryCount.toString(),
      },
    });
  }
  
  buildErrorContext = (error: Error): ErrorContext => {
    let category: ErrorCategory = 'unknown';
    
    const message = error.message.toLowerCase();
    if (message.includes('network') || message.includes('fetch') || message.includes('http')) {
      category = 'network';
    } else if (message.includes('unauthorized') || message.includes('forbidden') || message.includes('403') || message.includes('401')) {
      category = 'auth';
    } else if (message.includes('parse') || message.includes('json')) {
      category = 'parse';
    } else if (error.stack?.includes('render')) {
      category = 'render';
    }
    
    return {
      category,
      retryCount: this.state.retryCount,
      userId: this.props.userId,
      url: window.location.href,
      timestamp: Date.now(),
    };
  };
  
  retry = async () => {
    const maxRetries = this.props.maxRetries ?? 3;
    
    if (this.state.retryCount >= maxRetries) {
      return;
    }
    
    this.setState({ retrying: true });
    
    const delay = Math.pow(2, this.state.retryCount) * 1000;
    await new Promise(resolve => setTimeout(resolve, delay));
    
    this.setState(prev => ({
      error: null,
      retryCount: prev.retryCount + 1,
      retrying: false,
    }));
  };
  
  render() {
    if (this.state.error) {
      const context = this.buildErrorContext(this.state.error);
      const maxRetries = this.props.maxRetries ?? 3;
      const canRetry = !this.state.retrying && this.state.retryCount < maxRetries;
      
      return this.props.fallback({
        error: this.state.error,
        context,
        retry: this.retry,
        canRetry,
      });
    }
    
    return this.props.children;
  }
}

function App() {
  return (
    <ProductionErrorBoundary
      userId="user-123"
      maxRetries={3}
      fallback={({ error, context, retry, canRetry }) => (
        <div role="alert" className={`error-${context.category}`}>
          <h2>Error in {context.category}</h2>
          <p>{error.message}</p>
          <small>Attempt {context.retryCount + 1}</small>
          
          {canRetry ? (
            <button onClick={retry}>Retry</button>
          ) : (
            <button onClick={() => window.location.reload()}>Reload page</button>
          )}
        </div>
      )}
    >
      <RiskyApp />
    </ProductionErrorBoundary>
  );
}

import { createRoot } from 'react-dom/client';

const container = document.getElementById('root');
if (!container) throw new Error('Root container topilmadi');

const root = createRoot(container, {
  onCaughtError: (error, info) => {
    mockSentry.captureException(error, {
      level: 'warning',
      extra: { componentStack: info.componentStack, source: 'caught' },
    });
  },
  onUncaughtError: (error, info) => {
    mockSentry.captureException(error, {
      level: 'fatal',
      extra: { componentStack: info.componentStack, source: 'uncaught' },
    });
  },
  onRecoverableError: (error, info) => {
    mockSentry.captureException(error, {
      level: 'info',
      extra: { componentStack: info.componentStack, source: 'recovered' },
    });
  },
});

root.render(<App />);
```

**Tushuntirish:**
- Error categorization (network/render/auth/parse/unknown).
- Sentry mock integration.
- Exponential backoff retry (1s, 2s, 4s).
- Max retries (3 attempts) + manual reload after.
- R19 root callbacks (multiple severity levels).
- Component context (userId, URL, timestamp).

</details>

---

## Xulosa

Error Boundaries — React'ning xato handling fundamentalini ifodalovchi pattern. Class komponent shart (R19'da ham `getDerivedStateFromError`/`componentDidCatch` hook ekvivalenti yo'q). Asosiy fikrlar:

- **Error Boundary Concept** — class komponent `getDerivedStateFromError` (static, Render Phase, pure) + `componentDidCatch` (instance, Commit Phase, side effects OK) implement qiladi. Render Phase xatolar boundary'ga "ko'tarilib", state update + fallback UI ko'rsatadi. R16+ fundamental pattern.
- **Class Component Minimal Primer** — `extends React.Component`, `state` (class field yoki constructor), `render()` (majburiy), event handlers (arrow methods auto-bind), `setState` (async, batched, partial). R16.8+ aksariyat use case'lar hooks bilan almashtirildi, faqat Error Boundary uchun class shart.
- **Class Lifecycle Methods → Hooks Equivalents** — `componentDidMount` ≈ `useEffect([])`, `componentDidUpdate` ≈ `useEffect([deps])`, `componentWillUnmount` ≈ cleanup return. Error boundary methods (`getDerivedStateFromError`/`componentDidCatch`) hooks bilan **ekvivalent yo'q**.
- **`UNSAFE_*` Lifecycle Methods Deprecated** — R16.3'da `componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate` `UNSAFE_*` prefix bilan deprecated. **Sabab:** Concurrent rendering bilan mos emas. Modern alternative: `componentDidMount`, `getDerivedStateFromProps`, `getSnapshotBeforeUpdate`.
- **`getDerivedStateFromError`** — static method, Render Phase, pure (side effect TAQIQ), SSR-compatible. State update via return value.
- **`componentDidCatch`** — instance method, Commit Phase'da sinxron (paint'dan oldin), side effects OK (logging, Sentry, fetch). `React.ErrorInfo` (componentStack, R18+ digest). SSR'da skip.
- **Error Catching Scope** — **Caught:** render method, lifecycle methods, constructor, hooks render Phase, children render. **NOT Caught:** event handlers, async (`setTimeout`/`Promise.then`), SSR-specific paths, error boundary itself errors.
- **Event Handlers va Async Errors** — manual try/catch + `setState` callback throw → render Phase error → boundary catches. `unhandledrejection` window event for global async error capture.
- **Error Boundary Placement Strategy** — granular boundaries afzal (per-route, per-feature, per-widget) + top-level safety net. Localized failure, better UX, recovery mechanisms.
- **Error Recovery — `key` Prop Reset** — boundary `key` change → unmount + mount → state reset → children re-render. `componentDidUpdate` ichida `resetKeys` comparison (`react-error-boundary` library pattern).
- **Fallback UI Patterns** — skeleton-style, detailed (development), retry with exponential backoff, contextual (global/route/widget). Accessibility (`role="alert"`, focus management).
- **`react-error-boundary` Library** — community library (kichik, dependency'siz). `ErrorBoundary` (class), `useErrorBoundary` (hook, `showBoundary`), `withErrorBoundary` (HOC). `resetKeys`, `onReset`, `onError` props.
- **R19 Root Callbacks** — `createRoot(container, { onCaughtError, onUncaughtError, onRecoverableError })` application-wide error handling. Boundary'larsiz centralized error reporting.
- **Edge Cases** — boundary o'zi (render yoki `componentDidCatch`) xato bersa, React uni yutmaydi — `captureCommitPhaseError` orqali yuqoridagi eng yaqin boundary'ga ko'tariladi (topilmasa uncaught), R19 hooks API yo'q hali, async errors manual handling shart.

Versiya evolyutsiyasi:
- R16 (2017): `componentDidCatch` introduced
- R16.3 (2018): `UNSAFE_*` deprecated, `getDerivedStateFromProps`/`getSnapshotBeforeUpdate` added
- R16.6 (2018): `getDerivedStateFromError` added (SSR-compatible)
- R18 (2022): `onRecoverableError` root option + `digest` field SSR error'lari uchun
- R19 (2024): Root-level error callbacks (`onCaughtError`/`onUncaughtError`/`onRecoverableError`)

Cross-references:

- [`02-rendering.md`](02-rendering.md) — Render+Commit Phases
- [`04-reconciliation.md`](04-reconciliation.md) — Reconciliation key matching (boundary key reset)
- [`06-hydration.md`](06-hydration.md) — Hydration mismatch (R19 onRecoverableError)
- [`16-useeffect.md`](16-useeffect.md) — `useEffect` synchronization model
- [`18-useref.md`](18-useref.md) — `forwardRef` evolution
- [`22-concurrent-hooks.md`](22-concurrent-hooks.md) — `useId` SSR-safe
- [`25-legacy-patterns.md`](25-legacy-patterns.md) — HOC `withErrorBoundary`
- [`26-compound-components.md`](26-compound-components.md) — Compound Components ARIA
- [`29-suspense-lazy.md`](29-suspense-lazy.md) — Suspense Boundary

---

**Keyingi bo'lim:** [28-portals.md](28-portals.md) — Portals: `createPortal` API DOM tree vs React tree (event bubbling React tree bo'ylab DOM emas — important advanced detail), Modal/Tooltip/Dropdown use cases, focus management va a11y considerations (focus trap), z-index issues va stacking contexts, SSR considerations (portal target server'da yo'q workaround).
