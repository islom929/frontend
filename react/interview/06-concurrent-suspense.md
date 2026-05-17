# Concurrent React va Suspense — Interview Savollari

> Concurrent rendering, lanes priority, time slicing, tearing, `startTransition`, `useDeferredValue`, Suspense (code splitting va data), Streaming SSR, Selective Hydration, Progressive Hydration. R18 va R19 farqlari inline yoritiladi.

---

## Mundarija

**QISM A: Sync vs Concurrent** (savollar 1-4)
**QISM B: startTransition + Tearing** (savollar 5-7)
**QISM C: Suspense** (savollar 8-12)
**QISM D: Streaming SSR** (savollar 13-15)
**QISM E: Output & Bug Fix** (savollar 16-18)
**QISM F: R19 Hooks va Form Actions** (savollar 19-20)

**Jami:** 20 savol — Junior+ (2), Middle (7), Middle+ (6), Senior (5)

---

## QISM A: Sync vs Concurrent

### 1. Sync rendering qanday muammolarni keltiradi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Sync rendering — render fazasi to'liq bajarilmaguncha browser'ning main thread bloklanadi. Katta tree (1000+ komponent) yoki qimmat hisoblash render — frame budget'i (16ms) oshib ketsa, **jank** (UI muzlash, kechiktirilgan input) yuzaga keladi. R17 va R18 dan oldingi versiyalar to'liq sync edi: state update → render → commit (uzilmaydi). Concurrent rendering bu muammolarni hal qiladi: render uziladi, priority bo'yicha yield, browser tasks (paint, input) priority oladi.

### To'liq tushuntirish

**Sync render muammolari:**

1. **Long render block** — heavy component tree (10K items, complex JSX) — render 100ms+ olishi mumkin. Browser bu vaqtda hech narsa qila olmaydi (paint, scroll, input).

2. **Input lag** — typing yoki click paytida — React render bo'layotgan bo'lsa, input event handler kechiktiriladi.

3. **Frame drops** — 60fps target — har frame uchun 16.67ms. Render shu vaqtdan oshsa — frame skip → user "freeze" his qiladi.

4. **No priority** — kichik update (counter) va katta update (filter 10K) bir xil oqim bilan ishlaydi. Counter UI baribir laggy ko'rinadi.

5. **No interruption** — long-running render abort qilib bo'lmaydi (yangi state kelsa ham).

**Misol — sync render block:**

```tsx
// R17 sync render
function HeavyList({ items }: { items: Item[] }) {
  // 10K items, har biri qimmat JSX (chart)
  return (
    <ul>
      {items.map((i) => (
        <ChartItem key={i.id} data={i.data} />  // Heavy compute
      ))}
    </ul>
  );
}

// Setting state → 200ms render block
// Bu paytda: input ignored, animation paused, scroll dropped
```

**Cooperative scheduling — Concurrent:**

Render fazasi small chunks'ga bo'linadi (~5ms har biri). Har chunk'dan keyin React `MessageChannel.postMessage` orqali browser'ga yield qiladi. Browser high-priority task'lar bajaradi (paint, input). Keyin React davom etadi.

**Concurrent yechimi:**

```tsx
// R18+ concurrent
function App() {
  const [query, setQuery] = useState("");
  const [items, setItems] = useState<Item[]>([]);

  const handleSearch = (q: string) => {
    setQuery(q);
    startTransition(() => {
      // Heavy update — low priority, interruptible
      setItems(filterItems(q));
    });
  };

  return (
    <>
      <input value={query} onChange={(e) => handleSearch(e.target.value)} />
      <HeavyList items={items} />
    </>
  );
}
// Typing — fast (high priority)
// HeavyList — yieldable, abort qilinadi yangi typing kelsa
```

### Kod misoli

**Tipik sync block scenario:**

```tsx
// ❌ Sync block — 200ms render
function SearchPage() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<Result[]>([]);

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const q = e.target.value;
    setQuery(q);

    // Heavy filter — sync
    const filtered = bigDataSet.filter((d) => d.match(q));
    setResults(filtered);
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      <ResultList results={results} />
    </>
  );
}
// Har keystroke: 200ms freeze (filter + render)
// User "ABC" tez yozsa — input lag 600ms
```

**Concurrent yechim:**

```tsx
import { useTransition, useState } from "react";

function SearchPage() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<Result[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const q = e.target.value;
    setQuery(q); // Sync — high priority (input responsive)

    startTransition(() => {
      // Concurrent — low priority, interruptible
      setResults(bigDataSet.filter((d) => d.match(q)));
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <ResultList results={results} />
    </>
  );
}
// Input: instant (no lag)
// Results: deferred, abort qilinadi yangi typing kelsa
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Sync render mexanizmi (R17):**

```typescript
// Mental model
function performWork() {
  while (workInProgress !== null) {
    workInProgress = performUnitOfWork(workInProgress);
    // No yield — full sync
  }
  commitRoot();
}
```

To'liq tree traverse — uzulish yo'q.

**Concurrent render mexanizmi (R18):**

```typescript
function performWorkConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
  }
  if (workInProgress !== null) {
    // Render incomplete — yield to browser
    Scheduler.scheduleCallback(performWorkConcurrent);
  } else {
    commitRoot();
  }
}

function shouldYield(): boolean {
  // 5ms time slice
  return performance.now() - frameDeadline >= 0;
}
```

**`MessageChannel` — yield mechanism:**

```typescript
// Scheduler internal
const channel = new MessageChannel();
const port1 = channel.port1;
const port2 = channel.port2;

port1.onmessage = () => {
  // Continue render
  performWorkConcurrent();
};

function scheduleYield() {
  port2.postMessage(null); // Trigger event loop iteration
}
```

`MessageChannel` — `setTimeout(fn, 0)` dan tezroq (no 4ms minimum, no clamping).

**Frame budget hisobi:**

```
60fps → 16.67ms / frame
React budget: 5ms (concurrent slice)
Reserved for browser: 11.67ms (paint, layout, GC, etc.)
```

**Lanes priority — concurrent enabler:**

```typescript
SyncLane = 1                           // Highest — event handler updates
InputContinuousLane = 4                // Drag, scroll
DefaultLane = 16                        // Most updates
TransitionLane1...16 = 128...4194304    // Transitions
IdleLane = 1073741824                   // Lowest
```

Higher priority lanes — render first, abort lower-priority renders.

**Sync vs Concurrent measurement:**

```
Initial render 10K items (R17 sync):
- Total time: 200ms
- Main thread blocked: 200ms
- Frame drops: 12 (3.3x dropped)

Initial render 10K items (R18 concurrent + transition):
- Total time: 250ms (slightly more due to yields)
- Main thread blocked per slice: 5ms
- Frame drops: 0
- User experience: instant input, gradual content fill
```

**`createRoot` opt-in (R18):**

```tsx
// R17:
import ReactDOM from "react-dom";
ReactDOM.render(<App />, root); // Legacy mode — sync

// R18+:
import { createRoot } from "react-dom/client";
createRoot(root).render(<App />); // Concurrent mode — opt-in
```

R17 → R18 migration: `createRoot` switch — Concurrent features enabled.

**Concurrent benefits summary:**

1. **Interruptible rendering** — abort low-priority for high-priority
2. **Time slicing** — yield to browser
3. **Smart batching** — automatic batching all updates (R18)
4. **Suspense for data** — async render with fallback
5. **Streaming SSR** — chunked HTML delivery

</details>

### Edge Cases

- **`useEffect` with sync render**: Effect setTimeout chaqirsa — async, lekin React render davomida sync. Effect handler ichidagi setState — re-render trigger.
- **Production sync block detection**: Long Tasks API (>50ms) — performance.observer bilan log.
- **Single-threaded JS**: Concurrent React Browser thread'lar emas. Faqat **logical** concurrent (yield + resume), single thread bo'yicha.

### Follow-up savollar

- "Concurrent — multithreading?" — Yo'q. JavaScript single-threaded. Concurrent = cooperative scheduling (yield + resume).
- "R17 backward-compatible?" — Ha. `ReactDOM.render` legacy mode'da ishlaydi (sync). `createRoot` — opt-in Concurrent.
- "Web Workers — alternative?" — Ha, lekin different concept. Worker — actual parallel thread. React Concurrent — main thread cooperative scheduling.

</details>

---

### 2. Concurrent rendering qanday yechim taklif qiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Concurrent rendering 4 ta texnika bilan ishlaydi: (1) **Lanes priority** — har update'ga priority bitmap. (2) **Time slicing** — render uziladi har 5ms'da, browser'ga yield. (3) **Interruption** — high-priority update keladi → low-priority render abort. (4) **Concurrent commit** — bir necha render natijasi alohida commit. Asosiy API'lar: `useTransition`, `useDeferredValue`, `Suspense`, `startTransition`. R18'da default `createRoot` orqali opt-in. `<StrictMode>` Concurrent invariants'ni dev'da test qiladi (2x render, 2x effect setup-cleanup).

### To'liq tushuntirish

**Concurrent texnikalari:**

**1. Lanes priority bitmap:**

```typescript
// Har update — lane assigned
event handler setState → SyncLane (1)
startTransition setState → TransitionLane (128, 256, ...)
useDeferredValue → TransitionLane
React internal cleanup → IdleLane

// Render: highest priority lane first
```

**2. Time slicing:**

```typescript
// Reconciler render loop
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

// shouldYield():
// - 5ms slice exhausted → yield
// - Higher priority pending → yield (abort)
```

**3. Interruption:**

```tsx
function App() {
  const [highPriority, setHigh] = useState(0);
  const [lowPriority, setLow] = useState(0);

  return (
    <>
      <button onClick={() => setHigh(h => h + 1)}>High</button>
      <button onClick={() => startTransition(() => setLow(l => l + 1))}>
        Low
      </button>
    </>
  );
}

// Low priority render boshlandi
// Click "High" → abort low-priority, render high-priority
// Keyin low-priority restart
```

**4. Concurrent commit (rendering finalization):**

```
Render phase (interruptible) → Commit phase (synchronous)
                ↓
        Multiple in-flight renders may exist
        But only one commits at a time
```

**API'lar:**

| API | Use case |
|-----|---------|
| `useTransition` | Mark setState as non-urgent |
| `useDeferredValue` | Lag a value behind |
| `Suspense` | Wait for async (lazy, data) |
| `startTransition` | Non-urgent update (no isPending) |
| `<StrictMode>` | Concurrent invariants test |

### Kod misoli

**`useTransition` — search with non-blocking filter:**

```tsx
import { useState, useTransition } from "react";

interface Product {
  id: string;
  name: string;
  category: string;
}

function ProductSearch({ products }: { products: Product[] }) {
  const [query, setQuery] = useState("");
  const [filtered, setFiltered] = useState<Product[]>(products);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const q = e.target.value;
    setQuery(q);

    startTransition(() => {
      const result = products.filter((p) =>
        p.name.toLowerCase().includes(q.toLowerCase())
      );
      setFiltered(result);
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} placeholder="Search..." />
      {isPending && <span>Updating...</span>}
      <ul>
        {filtered.map((p) => (
          <li key={p.id}>{p.name}</li>
        ))}
      </ul>
    </>
  );
}

// Typing fast — input always responsive
// Filter renders at lower priority — abort qilinadi yangi keystroke kelsa
```

**`useDeferredValue` — equivalent without `setState`:**

```tsx
import { useState, useDeferredValue, useMemo } from "react";

function ProductSearch({ products }: { products: Product[] }) {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  // Heavy filter — useDeferredValue lag bilan
  const filtered = useMemo(
    () => products.filter((p) => p.name.includes(deferredQuery)),
    [products, deferredQuery]
  );

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ul>
        {filtered.map((p) => (
          <li key={p.id}>{p.name}</li>
        ))}
      </ul>
    </>
  );
}

// query — sync (input responsive)
// deferredQuery — lags behind, abort qilinadi
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Lanes priority hierarchy (selected):**

```typescript
// react-reconciler/src/ReactFiberLane.js
export const SyncHydrationLane = 1;
export const SyncLane = 2;
export const InputContinuousHydrationLane = 4;
export const InputContinuousLane = 8;
export const DefaultHydrationLane = 16;
export const DefaultLane = 32;
export const TransitionHydrationLane = 64;
export const TransitionLane1 = 128;
export const TransitionLane2 = 256;
// ... up to TransitionLane16
export const RetryLane1 = 33554432;
// ...
export const IdleLane = 1073741824;
```

**`requestUpdateLane` — priority assignment:**

```typescript
function requestUpdateLane(fiber: Fiber): Lane {
  if ((fiber.mode & ConcurrentMode) === NoMode) {
    return SyncLane;
  }
  if (CurrentBatchConfig.transition !== null) {
    return claimNextTransitionLane();
  }
  return getCurrentEventPriorityLane();
}

function getCurrentEventPriorityLane(): Lane {
  const eventName = window.event?.type;
  switch (eventName) {
    case "click":
    case "keydown":
    case "input":
      return SyncLane;
    case "drag":
    case "scroll":
      return InputContinuousLane;
    default:
      return DefaultLane;
  }
}
```

**Render loop with yield:**

```typescript
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function performWorkOnRoot(root: FiberRoot) {
  do {
    try {
      workLoopConcurrent();
      break; // No more work
    } catch (error) {
      handleThrow(root, error);
    }
  } while (true);

  if (workInProgress !== null) {
    // Yield + resume
    return RootInProgress;
  }
  return RootCompleted;
}
```

**`shouldYield` implementation:**

```typescript
let frameDeadline = 0;
const yieldInterval = 5; // ms

function shouldYield(): boolean {
  if (performance.now() < frameDeadline) {
    return false; // Continue
  }
  // Check if browser has higher priority work
  if (needsPaint) return true;
  // ...
  return performance.now() - frameDeadline >= 0;
}
```

**Scheduler integration:**

```typescript
// scheduler/src/SchedulerPriorities.js
const ImmediatePriority = 1;     // Sync
const UserBlockingPriority = 2;  // 250ms timeout
const NormalPriority = 3;        // 5000ms timeout
const LowPriority = 4;           // 10000ms timeout
const IdlePriority = 5;          // No timeout

// Priority → expiration time
function getExpirationTime(priority: number): number {
  switch (priority) {
    case ImmediatePriority: return performance.now() - 1;
    case UserBlockingPriority: return performance.now() + 250;
    case NormalPriority: return performance.now() + 5000;
    case LowPriority: return performance.now() + 10000;
    case IdlePriority: return performance.now() + 1073741823;
  }
}
```

**Interruption mechanism:**

```typescript
// Higher priority update arrives during low-priority render
function ensureRootIsScheduled(root: FiberRoot) {
  const nextLanes = getNextLanes(root, ...);
  const newCallbackPriority = getHighestPriorityLane(nextLanes);

  if (newCallbackPriority > currentCallbackPriority) {
    // Cancel current work
    Scheduler.cancelCallback(existingCallback);
    // Schedule with higher priority
    Scheduler.scheduleCallback(newCallbackPriority, performWork);
  }
}
```

**Cancellation — abort low-priority render:**

```typescript
// workInProgress tree discarded
// New render starts from current
// State updates queue preserved (replayed in next render)
```

**Concurrent invariants — Strict Mode dev:**

```tsx
<StrictMode>
  {/* Components render 2x in dev — idempotency check */}
  {/* Effects setup-cleanup-setup — cleanup correctness */}
  {/* useState initializer 2x */}
</StrictMode>
```

Concurrent rendering may render 2x, abort + retry. Pure components — immune. Impure — bug exposed.

**Memoization durumostan kelishimi:**

Concurrent rendering aborts mid-render. State updates replayed. Memoization (`useMemo`) cache — preserved across aborts (Fiber memoizedState).

</details>

### Edge Cases

- **`createRoot` not used**: Legacy `ReactDOM.render` — sync mode, no concurrent features. R18 majburiyat — `createRoot` migration.
- **`startTransition` outside render**: Possible — `startTransition` standalone import. Use in event handlers.
- **`isPending` flicker**: `useTransition` — `isPending` true paytida loading indicator. Bu vaqt qisqa bo'lsa — flicker. Suspense bilan combine.

### Follow-up savollar

- "Concurrent rendering — opt-in yoki default?" — `createRoot` opt-in (R18). Components ichida APIs (useTransition) — explicit opt-in per update.
- "All updates concurrent ishlaydimi?" — Ha. Hatto sync update'lar (event handlers) Concurrent infrastructure'da o'tadi (lekin SyncLane priority).
- "Concurrent + Class component compatibility?" — Compatible, lekin `componentWillUpdate` deprecated. `getDerivedStateFromProps` afzal.

</details>

---

### 3. Render purity invariantlari va Concurrent rendering safety [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Concurrent rendering uchun komponent **pure function** bo'lishi shart: bir xil props/state → bir xil JSX, side effects yo'q. Render abort qilinishi mumkin (interruption), restart qilinishi mumkin (Strict Mode 2x), multiple renders bo'lishi mumkin (different priorities). Side effect (DOM mutation, fetch, setState in body) — qaytariladigan qilinmaydi → state divergence. Render-safe operations: read state/props, compute derived values, return JSX. Side effect-restricted: `useEffect`, event handlers, refs (write only in effects/handlers).

### To'liq tushuntirish

**Render purity rules:**

| Action | Render OK | Render Anti-pattern |
|--------|-----------|---------------------|
| Read state/props | ✅ | — |
| Compute derived value | ✅ | — |
| Call pure function | ✅ | — |
| Return JSX | ✅ | — |
| `useState` setter call | — | ❌ infinite loop |
| `setRef.current = ...` | — | ❌ tearing |
| `document.title = ...` | — | ❌ DOM mutation |
| `fetch(...)` | — | ❌ network in render |
| `Math.random()` | — | ❌ different each render |
| `Date.now()` | — | ❌ different each render |
| `localStorage.setItem` | — | ❌ side effect |

**Render abort scenarios:**

1. **Higher priority interrupt** — render uziladi
2. **Suspense throw** — Promise yoki value waiting
3. **Error in component** — Error Boundary catch
4. **Concurrent feature** — `startTransition` lower priority abort

**Strict Mode dev — invariant testing:**

```tsx
<StrictMode>
  <App />
</StrictMode>

// Effects:
// - Components render 2x (verify pure)
// - Effects setup-cleanup-setup (verify cleanup)
// - useState initializer 2x
// - useMemo factory 2x (verify pure)
// - useReducer reducer 2x (verify pure)
```

### Kod misoli

**Anti-patterns va fix:**

```tsx
// ❌ Anti-pattern 1: setState in render
function BadCounter() {
  const [count, setCount] = useState(0);
  setCount(count + 1); // ❌ infinite loop
  return <p>{count}</p>;
}

// ✅ setState in event handler or effect
function GoodCounter() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    setCount((c) => c + 1);
  }, []); // Mount only
  return <p>{count}</p>;
}

// ❌ Anti-pattern 2: Mutation
function BadList({ items }: { items: Item[] }) {
  items.sort(); // ❌ mutates props
  return <ul>{items.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}

// ✅ Immutable
function GoodList({ items }: { items: Item[] }) {
  const sorted = [...items].sort((a, b) => a.name.localeCompare(b.name));
  return <ul>{sorted.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}

// ❌ Anti-pattern 3: External mutable
let renderCount = 0;
function BadComponent() {
  renderCount++; // ❌ external state mutation
  return <p>{renderCount}</p>;
}

// ✅ State or ref
function GoodComponent() {
  const renderCount = useRef(0);
  useEffect(() => {
    renderCount.current++;
  });
  return <p>{renderCount.current}</p>;
}

// ❌ Anti-pattern 4: Date.now in render
function BadTimestamp() {
  return <p>{Date.now()}</p>; // ❌ different each render
}

// ✅ State + effect
function GoodTimestamp() {
  const [now, setNow] = useState(() => Date.now());
  useEffect(() => {
    const id = setInterval(() => setNow(Date.now()), 1000);
    return () => clearInterval(id);
  }, []);
  return <p>{now}</p>;
}

// ❌ Anti-pattern 5: Ref mutation in render
function BadInput() {
  const inputRef = useRef<HTMLInputElement>(null);
  if (inputRef.current) {
    inputRef.current.value = "default"; // ❌ DOM mutation
  }
  return <input ref={inputRef} />;
}

// ✅ Effect
function GoodInput() {
  const inputRef = useRef<HTMLInputElement>(null);
  useEffect(() => {
    if (inputRef.current) {
      inputRef.current.value = "default";
    }
  }, []);
  return <input ref={inputRef} />;
}
```

**Pure render safe scenarios:**

```tsx
// ✅ Read state, compute, return JSX
function ProductCard({ product }: { product: Product }) {
  const formattedPrice = `$${product.price.toFixed(2)}`;
  const isExpensive = product.price > 100;

  return (
    <div>
      <h3>{product.name}</h3>
      <p>{formattedPrice}</p>
      {isExpensive && <span>Premium</span>}
    </div>
  );
}

// ✅ Conditional setState with bailout
function Counter({ initial }: { initial: number }) {
  const [count, setCount] = useState(initial);
  // Pseudo-derived state — but conditional setState
  // React detects sameness via Object.is — bailout
  // Safe in concurrent
  if (count > 100) {
    setCount(0); // Reset — but only fires when count actually > 100
  }
  return <p>{count}</p>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Strict Mode 2x render mechanism:**

```typescript
// In dev mode with <StrictMode>:
function renderWithHooks(...args) {
  const result = Component(...args); // First render

  // In strict mode dev:
  // 1. Discard result
  // 2. Reset hooks state
  // 3. Re-render with same inputs

  if (__DEV__ && (mode & StrictMode) !== NoMode) {
    const result2 = Component(...args); // Second render
    // result === result2 must hold (purity check)
  }

  return result;
}
```

**`useState` initializer 2x:**

```tsx
const [data] = useState(() => {
  // ❌ In strict mode dev: runs 2x
  fetch("/log");
  return computeInitial();
});

// ✅ Pure initializer
const [data] = useState(() => computeInitial());
useEffect(() => {
  fetch("/log"); // Side effect in effect
}, []);
```

**`useReducer` reducer 2x:**

```tsx
function reducer(state: State, action: Action): State {
  // ❌ Side effect — runs 2x in dev
  console.log("dispatch", action);
  fetch("/log");
  return { ...state, count: state.count + 1 };
}

// ✅ Pure reducer
function reducer(state: State, action: Action): State {
  return { ...state, count: state.count + 1 };
}
```

**Concurrent abort scenarios:**

```typescript
// Scenario 1: Higher priority interrupt
// startTransition setState → low priority render
// Click event → high priority render
// Low priority render aborted (workInProgress discarded)
// Re-rendered later from current state

// Scenario 2: Suspense throw
// Component reads from cache, not present → throw Promise
// Reconciler catches → render fallback
// Promise resolves → re-render

// Scenario 3: Error throw
// Component throws Error → Error Boundary catches
// Component fails → fallback UI

// Scenario 4: Render conflict
// State updates queued during render
// Render completes with stale state
// Re-render scheduled with fresh state
```

**State divergence example:**

```tsx
let counter = 0;

function ImpureComponent() {
  counter++; // ❌ Side effect

  // Concurrent abort scenario:
  // 1. Render starts: counter = 1
  // 2. Higher priority interrupt: render aborted
  // 3. Re-render starts: counter = 2 (already incremented)
  // 4. UI shows counter = 2 — but render only succeeded once!
  // → State divergence

  return <p>{counter}</p>;
}
```

**Pure render — abort safe:**

```tsx
function PureComponent({ items }: { items: Item[] }) {
  const sum = items.reduce((s, i) => s + i.value, 0);
  // Concurrent abort:
  // 1. Render starts: sum computed
  // 2. Abort: sum discarded (no side effect)
  // 3. Re-render: sum recomputed (same input → same output)
  // → No divergence
  return <p>Sum: {sum}</p>;
}
```

**`useSyncExternalStore` — safe external state:**

```tsx
// ❌ Direct subscribe in render
function ImpureComponent() {
  const [value, setValue] = useState(externalStore.getValue());
  externalStore.subscribe(setValue); // ❌ side effect each render
  return <p>{value}</p>;
}

// ✅ useSyncExternalStore
import { useSyncExternalStore } from "react";

function PureComponent() {
  const value = useSyncExternalStore(
    externalStore.subscribe,
    externalStore.getValue
  );
  return <p>{value}</p>;
}
```

`useSyncExternalStore` — Concurrent-safe subscription. Tearing prevention guaranteed.

**Effects'da bajariladigan ishlar:**

| Operation | useEffect | useLayoutEffect | Render body |
|-----------|-----------|-----------------|-------------|
| `setState` (with bailout) | ✅ | ✅ | ⚠️ careful |
| DOM read | ✅ | ✅ | ❌ |
| DOM write | ✅ | ✅ | ❌ |
| `fetch`/network | ✅ | ⚠️ avoid (sync block) | ❌ |
| `setTimeout` | ✅ | ✅ | ❌ |
| Subscribe | ✅ | ✅ | ❌ |
| Mutation tracking | ✅ | ✅ | ❌ |

</details>

### Edge Cases

- **`useState` setter in render with condition**: `if (cond) setState(...)` — React handles bailout via `Object.is`. Safe in concurrent (no infinite loop).
- **`console.log` in render**: Pure (output only). Strict Mode 2x — duplicated logs. OK for debugging.
- **Object literal as default state**: `useState({...})` — recreated each render. Use lazy initializer: `useState(() => ({...}))`.

### Follow-up savollar

- "Concurrent rendering — render N times?" — Yo'q, kafolatlangan times. Lekin abort + restart natijasida muloqot bor. Pure components — abort-safe.
- "Strict Mode production'da bormi?" — Yo'q. Faqat dev. Production single render.
- "How to detect impurity?" — ESLint `react-compiler/react-compiler` rule, Strict Mode dev (visual check), Profiler (suspicious behavior).

</details>

---

### 4. Concurrent anti-patterns va ulardan qutulish [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Concurrent React'da 5 ta katta anti-pattern: (1) **External state in render** — `let count = 0; count++` kabi mutable globals. (2) **Refs read in render** — tearing risk. (3) **Subscribe in render** — multiple subscriptions, leak. (4) **Imperative DOM in render** — paint inconsistency. (5) **Async side effects in render** (fetch, etc) — race conditions on abort. Yechimlar: `useState` for state, `useRef` for ref (read in effect/handler), `useSyncExternalStore` for external subscribe, `useEffect` for DOM/async.

### To'liq tushuntirish

**Anti-pattern 1: External mutable state**

```tsx
// ❌ Module-scope counter
let renderCount = 0;

function Component() {
  renderCount++;
  // Concurrent abort: counter advanced multiple times
  // - 1st render attempt: counter = 1
  // - Aborted (higher priority interrupt)
  // - 2nd render attempt: counter = 2
  // - Committed: UI shows 2, but render only "happened" once
  return <p>{renderCount}</p>;
}

// ✅ useState
function Component() {
  const [renderCount, setRenderCount] = useState(0);
  useEffect(() => {
    setRenderCount((c) => c + 1);
  });
  return <p>{renderCount}</p>;
}

// ✅ useRef (mutable ref)
function Component() {
  const renderCount = useRef(0);
  useEffect(() => {
    renderCount.current++;
  });
  return <p>{renderCount.current}</p>;
}
```

**Anti-pattern 2: Ref read in render**

```tsx
// ❌ Read ref in render — tearing
function ScrollPosition() {
  const positionRef = useRef(0);
  return <p>Position: {positionRef.current}</p>;
  // Concurrent: positionRef.current may differ between abort/restart
}

// ✅ useSyncExternalStore for external state read in render
function ScrollPosition() {
  const position = useSyncExternalStore(
    (callback) => {
      window.addEventListener("scroll", callback);
      return () => window.removeEventListener("scroll", callback);
    },
    () => window.scrollY
  );
  return <p>Position: {position}</p>;
}
```

**Anti-pattern 3: Subscribe in render**

```tsx
// ❌ Subscribe in render
function Component() {
  const [value, setValue] = useState(0);
  externalStore.subscribe(setValue); // ❌ each render — new subscription
  return <p>{value}</p>;
  // Concurrent abort: subscription leaks
}

// ✅ useEffect or useSyncExternalStore
function Component() {
  const [value, setValue] = useState(externalStore.getValue());
  useEffect(() => {
    const unsubscribe = externalStore.subscribe(setValue);
    return unsubscribe;
  }, []);
  return <p>{value}</p>;
}
```

**Anti-pattern 4: Imperative DOM in render**

```tsx
// ❌ Direct DOM in render
function Component() {
  const ref = useRef<HTMLInputElement>(null);
  if (ref.current) {
    ref.current.focus(); // ❌ during render
  }
  return <input ref={ref} />;
}

// ✅ useEffect / useLayoutEffect
function Component() {
  const ref = useRef<HTMLInputElement>(null);
  useEffect(() => {
    ref.current?.focus();
  }, []);
  return <input ref={ref} />;
}
```

**Anti-pattern 5: Async side effects in render**

```tsx
// ❌ Fetch in render
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  fetch(`/api/users/${userId}`).then(r => r.json()).then(setUser);
  // ❌ fires on every render, race conditions, no cleanup
  return user ? <p>{user.name}</p> : <Spinner />;
}

// ✅ useEffect with cleanup
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  useEffect(() => {
    const controller = new AbortController();
    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then(r => r.json())
      .then(setUser)
      .catch(err => {
        if (err.name !== "AbortError") console.error(err);
      });
    return () => controller.abort();
  }, [userId]);
  return user ? <p>{user.name}</p> : <Spinner />;
}

// ✅ R19 — use() with Suspense
function UserProfile({ userId }: { userId: string }) {
  const user = use(fetchUser(userId)); // Suspense catches Promise
  return <p>{user.name}</p>;
}
// fetchUser must return cached/stable Promise (else infinite suspend)
```

### Kod misoli

**Tearing problem demo:**

```tsx
// External store
let storeValue = 0;
const listeners = new Set<() => void>();

const externalStore = {
  getValue: () => storeValue,
  setValue: (v: number) => {
    storeValue = v;
    listeners.forEach((l) => l());
  },
  subscribe: (callback: () => void) => {
    listeners.add(callback);
    return () => listeners.delete(callback);
  },
};

// ❌ Tearing-prone
function TearingComponent() {
  const value = externalStore.getValue(); // Read in render
  return <p>{value}</p>;
  // Concurrent: render starts, value = 5
  //             store updates to 10 (interrupt)
  //             render aborts, restarts, value = 10
  //             Different snapshots in same UI → tearing
}

// ✅ useSyncExternalStore
function SafeComponent() {
  const value = useSyncExternalStore(
    externalStore.subscribe,
    externalStore.getValue
  );
  return <p>{value}</p>;
  // Guaranteed consistent snapshot across render
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why external state is dangerous:**

```typescript
// Imagine 3 components reading store:
let store = 5;

function A() { return <p>A: {store}</p>; }
function B() { return <p>B: {store}</p>; }
function C() { return <p>C: {store}</p>; }

// Concurrent render:
// 1. Render A: A reads store = 5
// 2. Yield to browser
// 3. External code: store = 10
// 4. Resume: render B reads store = 10
// 5. Render C reads store = 10
// → UI: A=5, B=10, C=10 — TEARING
```

**`useSyncExternalStore` — fix mechanism:**

```typescript
// React internal mental model
function useSyncExternalStore(subscribe, getSnapshot) {
  const fiber = currentlyRenderingFiber;
  const [snapshot, setSnapshot] = useState(getSnapshot());

  useEffect(() => {
    const handleChange = () => {
      const newSnapshot = getSnapshot();
      if (!Object.is(snapshot, newSnapshot)) {
        setSnapshot(newSnapshot);
      }
    };
    const unsubscribe = subscribe(handleChange);
    return unsubscribe;
  }, [subscribe]);

  // Important: also check during render for tearing
  // If snapshot changed between renders without notification — re-render
  const currentSnapshot = getSnapshot();
  if (!Object.is(snapshot, currentSnapshot)) {
    setSnapshot(currentSnapshot);
  }

  return snapshot;
}
```

**Strict Mode dev exposes anti-patterns:**

```tsx
<StrictMode>
  {/* These render 2x in dev */}
  <BadCounter />          {/* Counter doubles → bug obvious */}
  <BadFetcher />          {/* Fetch fires 2x → bug obvious */}
  <BadSubscriber />       {/* 2 subscriptions → bug obvious */}
</StrictMode>
```

**Concurrent rendering — re-render multiplier:**

```
Single sync render: 1x
Concurrent abort + restart: 2x
Strict Mode dev: 2x or 4x with concurrent
```

Anti-patterns proportionally amplified.

**`use()` (R19) — async-friendly render:**

```tsx
import { use } from "react";

function UserProfile({ userId }: { userId: string }) {
  const userPromise = useMemo(() => fetchUser(userId), [userId]);
  const user = use(userPromise); // Throws Promise if pending
  return <p>{user.name}</p>;
}

// Wrap in Suspense + ErrorBoundary
<ErrorBoundary fallback={<Error />}>
  <Suspense fallback={<Spinner />}>
    <UserProfile userId="123" />
  </Suspense>
</ErrorBoundary>
```

`use()` — render'da async-safe (Concurrent integrated).

**Anti-patterns map to fixes:**

| Anti-pattern | Fix |
|--------------|-----|
| Module-scope mutable | `useState` / `useRef` |
| Direct ref read in render | `useEffect` read |
| Subscribe in render | `useSyncExternalStore` |
| DOM mutation in render | `useEffect` / `useLayoutEffect` |
| Fetch in render | `useEffect` cleanup, R19 `use()` |
| Class lifecycle async | Effects |
| `setState` in render (uncontrolled) | Bailout pattern, `useEffect` |

**Eslint rules — automated detection:**

```json
{
  "plugins": ["react-hooks", "react-compiler"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn",
    "react-compiler/react-compiler": "error"
  }
}
```

**Migration checklist (R17 → R18 Concurrent):**

1. Replace `ReactDOM.render` with `createRoot`
2. Wrap `<App />` in `<StrictMode>` (dev) — exposes bugs
3. Replace external state subscribe with `useSyncExternalStore`
4. Add cleanup to all `useEffect` (subscriptions, async)
5. Move side effects out of render
6. Test with `<StrictMode>` enabled — fix any bugs
7. Adopt `useTransition` / `useDeferredValue` for non-urgent updates

</details>

### Edge Cases

- **`useEffect` cleanup not called on abort**: Cleanup on unmount/dep change, not on render abort. Render abort = clean state, no cleanup needed.
- **`useLayoutEffect` blocking**: Sync after DOM mutation, before paint. Concurrent rendering — interruptible, but `useLayoutEffect` sync.
- **`React.cache` (RSC)**: Server-side memoization — different from client `useMemo`. Concurrent-friendly.

### Follow-up savollar

- "Refs barcha hollarda anti-pattern?" — Yo'q. Refs — DOM access (focus, scroll), mutable state (timer IDs). Faqat **read in render** xavfli.
- "How to detect tearing in production?" — Telemetry: snapshot mismatch count, `useSyncExternalStore` consumers monitoring. Library — `react-error-boundary`.
- "Class komponent concurrent-safe?" — Mostly yes. Lekin deprecated lifecycles (componentWillUpdate, componentWillReceiveProps) — concurrent-incompatible.

</details>

---

## QISM B: startTransition + Tearing

### 5. `startTransition` ichki mexanikasi va lanes mapping [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`startTransition(fn)` — `fn` ichida bajarilgan `setState`'larni **TransitionLane**'ga mark qiladi (DefaultLane o'rniga). Reconciler bu lane'ni quyi priority deb biladi: render uziladi (high-priority kelsa abort), browser yield ko'p qiladi. `useTransition` — `[isPending, startTransition]` qaytaradi (pending state bilan). API ikki shaklda: standalone `startTransition` (no isPending) va `useTransition` (with isPending). R18 majburiy `createRoot`. **R19'da `startTransition` async funksiyani qabul qiladi** (`await` ichida ishlaydi — Form Actions asosi). Internal: `CurrentBatchConfig.transition` flag → `claimNextTransitionLane` orqali lane assignment.

### To'liq tushuntirish

**API formalari:**

```tsx
import { startTransition, useTransition } from "react";

// 1. Standalone — no isPending
function Component() {
  const handleClick = () => {
    startTransition(() => {
      setState(newValue);
    });
  };
}

// 2. Hook — with isPending
function Component() {
  const [isPending, startTransition] = useTransition();
  const handleClick = () => {
    startTransition(() => {
      setState(newValue);
    });
  };
  return <>{isPending && <Spinner />}<button onClick={handleClick}>...</button></>;
}
```

**Internal — lanes assignment:**

```typescript
// React internal
let CurrentBatchConfig = { transition: null };

function startTransition(scope: () => void) {
  const prevTransition = CurrentBatchConfig.transition;
  CurrentBatchConfig.transition = {};
  try {
    scope();
  } finally {
    CurrentBatchConfig.transition = prevTransition;
  }
}

// dispatchSetState
function dispatchSetState(fiber, queue, action) {
  const lane = requestUpdateLane(fiber);
  // ...
}

function requestUpdateLane(fiber: Fiber): Lane {
  if (CurrentBatchConfig.transition !== null) {
    return claimNextTransitionLane(); // TransitionLane1...16
  }
  return getCurrentEventPriorityLane(); // SyncLane / DefaultLane
}
```

**TransitionLane1...16:**

```typescript
const TransitionLane1 = 0b0000000000000000000000010000000;   // 128
const TransitionLane2 = 0b0000000000000000000000100000000;   // 256
const TransitionLane3 = 0b0000000000000000000001000000000;   // 512
// ... up to TransitionLane16
const TransitionLane16 = 0b0000000000000000100000000000000;  // 4194304

let nextTransitionLane = TransitionLane1;

function claimNextTransitionLane(): Lane {
  const lane = nextTransitionLane;
  nextTransitionLane <<= 1;
  if (nextTransitionLane > TransitionLane16) {
    nextTransitionLane = TransitionLane1;
  }
  return lane;
}
```

Round-robin allocation 16 ta lane orasida.

**Render priority:**

```typescript
// Lane priority order
SyncLane > InputContinuousLane > DefaultLane > TransitionLane1...16 > IdleLane

// Reconciler — render highest priority lane first
// Lower priority renders abort if higher priority pending
```

### Kod misoli

**`useTransition` — search bilan:**

```tsx
import { useState, useTransition } from "react";

interface Movie {
  id: string;
  title: string;
  year: number;
  rating: number;
}

function MovieSearch({ allMovies }: { allMovies: Movie[] }) {
  const [query, setQuery] = useState("");
  const [filtered, setFiltered] = useState<Movie[]>(allMovies);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const q = e.target.value;
    setQuery(q); // SyncLane — input updated immediately

    startTransition(() => {
      // TransitionLane — non-urgent
      const result = allMovies.filter((m) =>
        m.title.toLowerCase().includes(q.toLowerCase())
      );
      setFiltered(result);
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <span style={{ color: "gray" }}>Filtering...</span>}
      <ul style={{ opacity: isPending ? 0.5 : 1 }}>
        {filtered.map((m) => (
          <li key={m.id}>
            {m.title} ({m.year}) — {m.rating}/10
          </li>
        ))}
      </ul>
    </>
  );
}

// User types fast (e.g., "ABC"):
// 1. Type "A" → query="A" (SyncLane), filter starts (TransitionLane)
// 2. Type "B" → query="AB" (SyncLane), filter aborts, new filter starts
// 3. Type "C" → query="ABC" (SyncLane), filter aborts, new filter starts
// 4. User pauses → final filter completes
// → Input always responsive, list lags but consistent
```

**Standalone `startTransition` — no `isPending`:**

```tsx
import { startTransition } from "react";

function FilterButton({ onApply }: { onApply: () => void }) {
  const handleClick = () => {
    startTransition(() => {
      onApply(); // setState'lar low priority bilan
    });
  };
  return <button onClick={handleClick}>Apply Filter</button>;
}
```

**Multiple state updates in transition:**

```tsx
function handleSearch(q: string) {
  setQuery(q); // SyncLane
  startTransition(() => {
    setLoading(true);              // TransitionLane
    setResults(filterData(q));     // Same TransitionLane
    setSelectedId(null);            // Same TransitionLane
  });
}
// Barcha 3 setState bir batch + bir lane bilan
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`useTransition` ichi:**

```typescript
// react/src/ReactHooks.js (mental model)
function mountTransition(): [boolean, (callback: () => void) => void] {
  const [isPending, setPending] = useState(false);
  const start = (callback: () => void) => {
    setPending(true); // SyncLane
    const prevTransition = ReactCurrentBatchConfig.transition;
    ReactCurrentBatchConfig.transition = {};
    try {
      setPending(false); // TransitionLane (within transition)
      callback();
    } finally {
      ReactCurrentBatchConfig.transition = prevTransition;
    }
  };
  return [isPending, start];
}
```

**`isPending` — multi-lane behavior:**

`isPending` true va false setlari turli lane'larda:
- `setPending(true)` — SyncLane (immediate UI feedback)
- `setPending(false)` — TransitionLane (when transition completes)

Bu Reconciler'ga 2 ta render generates: 1) sync render with pending=true, 2) transition render with pending=false.

**Render queue:**

```typescript
// Render lanes priority queue
// 1. Sync flush — handle SyncLane updates (input, isPending=true)
// 2. Concurrent flush — handle TransitionLane updates (filter, isPending=false)
```

**Abort scenario:**

```typescript
// Render trace:
// 1. User types "A"
//    - SyncLane: setQuery("A"), setPending(true)
//    - Sync render commits: input="A", spinner shown
//    - TransitionLane: setResults(filterA), setPending(false)
//    - Concurrent render starts (low priority)
// 2. User types "B" (mid-render)
//    - SyncLane: setQuery("AB"), setPending(true)
//    - HIGHER PRIORITY → abort transition render
//    - Discard workInProgress
//    - Sync render commits: input="AB", spinner shown
//    - TransitionLane: setResults(filterAB), setPending(false)
//    - Concurrent render restarts with new state
```

**Transition lane recycling:**

`claimNextTransitionLane` — round-robin 16 lanes. After 16 transitions — wraps. Multiple concurrent transitions — different lanes.

```typescript
// 16 simultaneous transitions:
// Each gets unique lane
// Reconciler renders highest priority lane first
// Each lane independent (separate workInProgress?)
// In practice: combined into single workInProgress, but tracked per-lane
```

**Async vs sync inside transition:**

```tsx
// ❌ Async inside transition — only sync part affected
startTransition(async () => {
  await fetch(...);
  setState(...); // Outside transition (after await)
});

// ✅ R19 — async transitions (with action support)
startTransition(async () => {
  // Async transitions in R19
  await mutateAsync();
  setState(...); // Still inside transition
});
```

R18'da `startTransition` faqat sync function'larni qabul qiladi. R19'da async transition support qo'shildi (Server Actions integration uchun).

**`useDeferredValue` vs `useTransition`:**

| Aspect | `useTransition` | `useDeferredValue` |
|--------|-----------------|---------------------|
| API | Wraps `setState` | Wraps value |
| Source | Inside scope | Outside (passed value) |
| `isPending` | Built-in | Manual derive |
| Use case | Action-triggered transitions | Derived value lag |

```tsx
// useTransition — wraps setState
const [isPending, startTransition] = useTransition();
const handleSearch = (q) => startTransition(() => setQuery(q));

// useDeferredValue — wraps value
const deferredQuery = useDeferredValue(query);
// Equivalent to: setQuery sync, deferredQuery transition lane
```

**`startTransition` standalone vs `useTransition` Hook:**

| Standalone | Hook |
|------------|------|
| No `isPending` | `isPending` true/false |
| Outside component (any module) | Inside component only |
| Single API call | 2-element tuple |
| Light Hook overhead | Hook overhead (~0.5µs) |

**Performance:**

```
Without useTransition:
- Heavy filter blocks UI for 200ms
- Input lag visible
- Frame drops during filter

With useTransition:
- Filter starts in low priority
- Yields every 5ms to browser
- Input event interrupts filter (abort + restart)
- Total filter time slightly longer (250ms) due to yield overhead
- But UI never blocked → user perception: instant
```

</details>

### Edge Cases

- **`startTransition` doesn't make sync work async**: `startTransition(() => { /* heavy compute */ })` — heavy compute still sync. Only `setState`'lar transition lane'ga.
- **Promise inside transition**: `startTransition(async () => ...)` — only sync portion in transition. Awaited continuations — outside transition.
- **`flushSync` inside transition**: Forces sync flush, breaks transition. Avoid mixing.

### Follow-up savollar

- "`startTransition` keyin `setState` outside transition?" — Outside transition — DefaultLane (or SyncLane if event handler).
- "Multiple transitions concurrent ishlaydimi?" — Ha, har biri o'z lane'da. Reconciler interleaves.
- "Transition lane saturated bo'ladimi (16+)?" — Wraps round-robin. Practical limit not reached.

</details>

---

### 6. Tearing va `useSyncExternalStore` chuqur mexanizmi [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Tearing — bitta render davomida external state bir necha marta o'qilib, **bir-biriga zid qiymatlar** UI'da ko'rinishi. Concurrent rendering yield qilganda external store update bo'lsa — kelajakdagi sub-tree o'qishlari yangi value, oldingi sub-tree (already rendered) eski value → UI inconsistency. `useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)` — Concurrent-safe subscription. Mexanizm: render paytida `getSnapshot` ikki marta chaqiriladi (begin va end of render), agar farq bo'lsa — re-render. Subscribe — listener register, store change'da setState trigger.

### To'liq tushuntirish

**Tearing problem:**

```tsx
// External store
let counter = 0;
const listeners = new Set<() => void>();

const store = {
  getValue: () => counter,
  increment: () => {
    counter++;
    listeners.forEach((l) => l());
  },
  subscribe: (fn: () => void) => {
    listeners.add(fn);
    return () => listeners.delete(fn);
  },
};

// ❌ Direct read — tearing
function CounterDisplay() {
  const value = store.getValue(); // Read in render
  return (
    <>
      <p>Top: {value}</p>
      {/* Long subtree — yield happens here */}
      <ExpensiveSubtree />
      <p>Bottom: {value}</p>
    </>
  );
}

// Concurrent timeline:
// 1. Render starts: CounterDisplay reads value=0
// 2. <p>Top: 0</p> rendered
// 3. <ExpensiveSubtree /> renders, yields 5ms slice
// 4. Browser tick — store.increment() called externally → counter=1
// 5. Render resumes
// 6. <p>Bottom: 0</p> rendered (stale local variable, but reading value=0 from this scope)
//    — Actually JS captured 0 in `value` const → no tearing in THIS case
//    — But if reread inside, would tear

// Real tearing — multiple components reading store directly:
function A() { return <p>A: {store.getValue()}</p>; }
function B() { return <p>B: {store.getValue()}</p>; }
function App() {
  return (
    <>
      <A />
      {/* Long render — yield */}
      <B />
    </>
  );
}
// 1. A renders: store=0
// 2. Yield, store updated to 1
// 3. B renders: store=1
// → UI: A=0, B=1 — TEARING
```

**`useSyncExternalStore` API:**

```typescript
function useSyncExternalStore<T>(
  subscribe: (callback: () => void) => () => void,
  getSnapshot: () => T,
  getServerSnapshot?: () => T,
): T;
```

**Parameters:**

| Parameter | Type | Purpose |
|-----------|------|---------|
| `subscribe` | `(callback) => unsubscribe` | Register listener |
| `getSnapshot` | `() => T` | Read current value |
| `getServerSnapshot` | `() => T` (optional) | SSR snapshot |

**Tearing prevention:**

`useSyncExternalStore` mechanism:

1. Render: read `getSnapshot()` — store first call
2. Render begin: subscribe via effect
3. Render: read `getSnapshot()` again — verify consistency
4. If snapshots differ → re-render with consistent snapshot
5. Subscription persistent — store updates trigger re-render

### Kod misoli

**Tearing-safe external store:**

```tsx
import { useSyncExternalStore } from "react";

interface Store<T> {
  getValue: () => T;
  setValue: (v: T) => void;
  subscribe: (cb: () => void) => () => void;
}

function createStore<T>(initial: T): Store<T> {
  let value = initial;
  const listeners = new Set<() => void>();

  return {
    getValue: () => value,
    setValue: (v) => {
      value = v;
      listeners.forEach((l) => l());
    },
    subscribe: (cb) => {
      listeners.add(cb);
      return () => listeners.delete(cb);
    },
  };
}

const counterStore = createStore(0);

// ✅ Tearing-safe
function CounterDisplay() {
  const value = useSyncExternalStore(
    counterStore.subscribe,
    counterStore.getValue
  );
  return <p>Counter: {value}</p>;
}

function CounterButtons() {
  const value = useSyncExternalStore(
    counterStore.subscribe,
    counterStore.getValue
  );
  return (
    <button onClick={() => counterStore.setValue(value + 1)}>
      Increment ({value})
    </button>
  );
}
```

**SSR-aware store:**

```tsx
import { useSyncExternalStore } from "react";

interface ThemeStore {
  getValue: () => "light" | "dark";
  getServerValue: () => "light" | "dark";
  setValue: (v: "light" | "dark") => void;
  subscribe: (cb: () => void) => () => void;
}

const themeStore: ThemeStore = {
  getValue: () => {
    return (localStorage.getItem("theme") as "light" | "dark") ?? "light";
  },
  getServerValue: () => {
    // No localStorage on server
    return "light";
  },
  setValue: (v) => {
    localStorage.setItem("theme", v);
    listeners.forEach((l) => l());
  },
  subscribe: (cb) => {
    // ...
  },
};

// SSR-safe
function ThemeProvider({ children }: { children: React.ReactNode }) {
  const theme = useSyncExternalStore(
    themeStore.subscribe,
    themeStore.getValue,
    themeStore.getServerValue,
  );

  return <div className={theme}>{children}</div>;
}
```

**Real-world: Redux integration:**

```tsx
// Redux-like store
const reduxStore = createStore(reducer);

function useReduxState<T>(selector: (state: AppState) => T): T {
  return useSyncExternalStore(
    reduxStore.subscribe,
    () => selector(reduxStore.getState()),
  );
}

function UserName() {
  const name = useReduxState((state) => state.user.name);
  return <p>Hello, {name}</p>;
}
```

`react-redux` v8+ ichida `useSyncExternalStore` bilan implement qilingan.

<details>
<summary><strong>Deep Dive</strong></summary>

**`useSyncExternalStore` internal:**

```typescript
// react/src/ReactHooks.js (mental model)
function mountSyncExternalStore<T>(
  subscribe: (cb: () => void) => () => void,
  getSnapshot: () => T,
  getServerSnapshot?: () => T,
): T {
  const fiber = currentlyRenderingFiber;

  // 1. Get snapshot
  let nextSnapshot;
  const isHydrating = getIsHydrating();
  if (isHydrating) {
    nextSnapshot = getServerSnapshot?.() ?? getSnapshot();
  } else {
    nextSnapshot = getSnapshot();
  }

  // 2. Schedule subscribe + check effect
  const hook = mountWorkInProgressHook();
  hook.memoizedState = nextSnapshot;

  const inst: StoreInstance<T> = {
    value: nextSnapshot,
    getSnapshot,
  };

  // Schedule effect to subscribe
  pushEffect(HookHasEffect | Passive, () => {
    inst.value = nextSnapshot;
    inst.getSnapshot = getSnapshot;
    if (checkIfSnapshotChanged(inst)) {
      // Snapshot changed since render — force re-render
      forceStoreRerender(fiber);
    }
    return subscribe(() => {
      if (checkIfSnapshotChanged(inst)) {
        forceStoreRerender(fiber);
      }
    });
  });

  return nextSnapshot;
}
```

**`checkIfSnapshotChanged`:**

```typescript
function checkIfSnapshotChanged<T>(inst: StoreInstance<T>): boolean {
  const latestGetSnapshot = inst.getSnapshot;
  const prevValue = inst.value;
  try {
    const nextValue = latestGetSnapshot();
    return !Object.is(prevValue, nextValue);
  } catch (error) {
    return true;
  }
}
```

`Object.is` solishtirish — yangi snapshot eski'dan farq qilsa — re-render trigger.

**Concurrent tearing fix flow:**

```
1. Render starts (transition lane)
2. Component reads getSnapshot() = 5
3. Yield to browser
4. External update: store now 10
5. Render resumes
6. Component A reads value = 5 (cached in fiber's hook state)
7. Render commits with value = 5
8. After commit — useSyncExternalStore effect runs
9. checkIfSnapshotChanged: stored 5, current 10 → changed
10. forceStoreRerender — schedule re-render with value = 10
11. Re-render commits with value = 10 (consistent across all readers)
```

**Performance considerations:**

```typescript
// getSnapshot called frequently:
// - Mount: 1x
// - Each render: 1x
// - Each external update: 1x per subscriber

// getSnapshot must be stable reference (or cached via memoization)
// If returns new object each call → infinite re-render!

// ❌ Wrong:
useSyncExternalStore(subscribe, () => ({ ...store.state })); // New object each call!

// ✅ Cache:
const memoizedSnapshot = useMemo(() => store.state, [store.state]);
useSyncExternalStore(subscribe, () => memoizedSnapshot);
```

**`useSyncExternalStoreWithSelector` (selector hook):**

```typescript
import { useSyncExternalStoreWithSelector } from "use-sync-external-store/with-selector";

const userName = useSyncExternalStoreWithSelector(
  store.subscribe,
  store.getState,
  store.getServerState,
  (state) => state.user.name,
  (a, b) => a === b, // Custom equality
);
```

Selector — only re-render when selected value changes.

**SSR — `getServerSnapshot`:**

Server-side `localStorage`/`window` etc unavailable. `getServerSnapshot` provides server-rendered snapshot. Avoids hydration mismatch.

```typescript
// Hydration mismatch detection
if (renderedOnServer && getServerSnapshot()) {
  // Compare server vs client snapshot
  // If different → log warning, re-render
}
```

**Browser API integration:**

```tsx
// Subscribe to window events
function useWindowSize() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener("resize", callback);
      return () => window.removeEventListener("resize", callback);
    },
    () => ({ width: window.innerWidth, height: window.innerHeight }),
    () => ({ width: 0, height: 0 }), // SSR fallback
  );
}

// ❌ getSnapshot returns new object each call!
// → Infinite re-render

// ✅ Cache snapshot:
function useWindowSize() {
  const snapshotRef = useRef({ width: 0, height: 0 });

  return useSyncExternalStore(
    (callback) => {
      const handler = () => {
        snapshotRef.current = {
          width: window.innerWidth,
          height: window.innerHeight,
        };
        callback();
      };
      window.addEventListener("resize", handler);
      return () => window.removeEventListener("resize", handler);
    },
    () => snapshotRef.current, // Stable reference
  );
}
```

**Library ecosystem:**

- **Zustand** — `useSyncExternalStore` based
- **Jotai** — atom subscriptions
- **Redux v9** — `useSyncExternalStore`
- **MobX** — observer wrapper bilan

</details>

### Edge Cases

- **`getSnapshot` throws**: Treated as "snapshot changed" — re-render scheduled.
- **`subscribe` returns void**: Cleanup not called → memory leak. Always return unsubscribe function.
- **Hydration mismatch**: Server snapshot != client snapshot → React logs warning, falls back to client.

### Follow-up savollar

- "Tearing R17'da bormidi?" — Yo'q. Sync rendering — render uziladi yo'q. Tearing concurrent-specific.
- "Library bilan ishlatish kerakmi?" — Custom subscriptions — `useSyncExternalStore`. Library state mgmt (Zustand, Redux v8+) — built-in. Library afzal real apps.
- "Performance impact?" — `getSnapshot` har render call'da. Stable reference va minimal computation.

</details>

---

### 7. `useDeferredValue` advanced patterns [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useDeferredValue(value)` — value'ni Concurrent rendering'da **lag** qildiradi. Sync render'da current value bilan UI yangilanadi (high-priority part), keyin Concurrent render'da deferred value bilan re-render (low-priority part — heavy computation). Differs from `useTransition`: API state value bilan emas, balki **derived value lag** beradi. Use case: search filter (input — sync, list — deferred), large list (selected — sync, full data — deferred). Internal: separate render lane (transition lane). **R19 yangilik:** ikkinchi argument `initialValue` — birinchi render'da deferred value boshlang'ich qiymati (mount mismatch'larni oldini olish), shaklda `useDeferredValue(value, initialValue?)`.

### To'liq tushuntirish

**API:**

```typescript
// R18
function useDeferredValue<T>(value: T): T;

// R19 — optional initialValue (mount uchun)
function useDeferredValue<T>(value: T, initialValue?: T): T;
```

**Returns:** Lagged version of `value`. During fast updates — returns previous value. During idle time — converges to current value. R19'da `initialValue` berilsa, birinchi render'da deferred copy `initialValue` bo'ladi (suspended initial render'da foydali).

**`useDeferredValue` vs `useTransition`:**

| Feature | `useDeferredValue` | `useTransition` |
|---------|---------------------|-----------------|
| Wraps | Value | setState callback |
| Source | Pass value | Inside scope |
| Result | Deferred value | `[isPending, startTransition]` |
| isPending derive | Manual: `value !== deferredValue` | Built-in |
| Use case | Derived heavy compute | Action-triggered transitions |

**Internal mechanism:**

```typescript
// React internal (mental model)
function useDeferredValue<T>(value: T): T {
  const [, setDeferredValue] = useState(value);
  const lastValue = useRef(value);

  // First sync render: returns value as-is
  // Subsequent: returns lastValue, schedule transition

  if (value !== lastValue.current) {
    // Schedule transition update
    startTransition(() => {
      setDeferredValue(value);
    });
    return lastValue.current; // Lagged
  }
  lastValue.current = value;
  return value;
}
```

**Visual timeline:**

```
User types "ABC" rapidly:

Time:    0ms    50ms   100ms   150ms   200ms
Input:   A      AB     ABC     ABC     ABC
Defer:   "" → A → AB → AB → ABC

Sync renders: input updates instantly
Concurrent renders: deferred value catches up after pause
```

### Kod misoli

**Search with deferred filter:**

```tsx
import { useState, useDeferredValue, useMemo } from "react";

interface Article {
  id: string;
  title: string;
  body: string;
}

function ArticleSearch({ articles }: { articles: Article[] }) {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  // Heavy filter — uses deferred query
  const filtered = useMemo(() => {
    return articles.filter(
      (a) =>
        a.title.toLowerCase().includes(deferredQuery.toLowerCase()) ||
        a.body.toLowerCase().includes(deferredQuery.toLowerCase())
    );
  }, [articles, deferredQuery]);

  // Visual feedback when stale
  const isStale = query !== deferredQuery;

  return (
    <>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <p>
        {filtered.length} results
        {isStale && <span style={{ color: "gray" }}> (updating...)</span>}
      </p>
      <ul style={{ opacity: isStale ? 0.5 : 1 }}>
        {filtered.map((a) => (
          <li key={a.id}>
            <h3>{a.title}</h3>
            <p>{a.body.slice(0, 100)}...</p>
          </li>
        ))}
      </ul>
    </>
  );
}
```

**Comparison — without `useDeferredValue`:**

```tsx
// ❌ Sync — input lag during heavy filter
function BadSearch({ articles }: { articles: Article[] }) {
  const [query, setQuery] = useState("");

  const filtered = useMemo(() => {
    return articles.filter((a) => a.title.includes(query));
  }, [articles, query]);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ul>
        {filtered.map((a) => <li key={a.id}>{a.title}</li>)}
      </ul>
    </>
  );
}

// 10K articles, complex filter — typing laggy
```

**Pattern: deferred + memo'lashtirilgan child:**

```tsx
import { useState, useDeferredValue, memo } from "react";

interface ProductListProps {
  query: string;
  allProducts: Product[];
}

const ProductList = memo(function ProductList({ query, allProducts }: ProductListProps) {
  const filtered = allProducts.filter((p) => p.name.includes(query));
  return <ul>{filtered.map((p) => <li key={p.id}>{p.name}</li>)}</ul>;
});

function App() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {/* memo bilan: deferredQuery o'zgarmasa, ProductList re-render YO'Q */}
      <ProductList query={deferredQuery} allProducts={products} />
    </>
  );
}

// memo + deferred: list updates only when typing pauses
```

**Pattern: skeleton loading:**

```tsx
import { useState, useDeferredValue, Suspense } from "react";

function App() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Suspense fallback={<Skeleton />}>
        <SearchResults query={deferredQuery} />
      </Suspense>
    </>
  );
}

// Initial: Skeleton
// First search: data fetch (Suspense) → results
// New typing: deferredQuery old → previous results visible (no flash)
//             new query → re-fetch in background
//             Suspense — keep old UI, no flash to skeleton
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Lane assignment:**

```typescript
// useDeferredValue internal — uses transition lane
// First render: value committed in current lane
// Stale value: schedule re-render in transition lane

// Internal pseudo:
function useDeferredValue<T>(value: T): T {
  const [hookValue, setHookValue] = useState(value);
  const fiber = currentlyRenderingFiber;

  if (!Object.is(value, hookValue)) {
    // Schedule transition
    fiber.lanes = mergeLanes(fiber.lanes, TransitionLane);
    setHookValue(value); // In transition lane
    return hookValue; // Stale value returned
  }

  return hookValue;
}
```

**Timeout argument (R19+):**

```tsx
// R19 experimental — timeout argument
const deferred = useDeferredValue(value, { timeoutMs: 2000 });
// If transition takes longer than 2000ms, force commit
// (Not yet stable, may change)
```

**Concurrent suspend integration:**

```tsx
function SearchResults({ query }: { query: string }) {
  // Suspends if query results not cached
  const data = use(fetchResults(query));
  return <ResultsList items={data} />;
}

function App() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Suspense fallback={<Skeleton />}>
        <SearchResults query={deferredQuery} />
      </Suspense>
    </>
  );
}

// User types → query updates sync
// deferredQuery lags
// SearchResults uses deferredQuery
// If new fetch needed → Suspense
// React shows previous results (no flash) until new ready
```

**`useDeferredValue` for entire components:**

```tsx
function Tab({ children }: { children: React.ReactNode }) {
  const deferredChildren = useDeferredValue(children);
  return <div>{deferredChildren}</div>;
}

// Renders children at lower priority
// Useful for tab transitions — keep current tab responsive
```

**Comparison with debounce:**

| Aspect | Debounce | `useDeferredValue` |
|--------|----------|---------------------|
| Strategy | Wait fixed time after input | Yield to higher priority |
| Concurrent integration | No | Yes (transition lane) |
| Setup | `setTimeout` | Hook |
| Tunable delay | Yes (e.g., 300ms) | No (priority-based) |
| Resource usage | Same | Adaptive |

```tsx
// Debounce equivalent
function useDebounced<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  return debounced;
}

// Usage:
const debouncedQuery = useDebounced(query, 300);

// Difference:
// debounce: always 300ms delay (regardless of system load)
// useDeferredValue: yields to high priority, catches up when idle
//                   adapts to system performance
```

**Performance characteristics:**

```
useDeferredValue overhead:
- Hook slot allocation: ~0.5µs
- Per render comparison: ~0.1µs

Heavy computation amortization:
- Without: blocks main thread
- With: yields, retries on abort
- Net cost: similar, but UI responsive
```

**Multiple deferred values:**

```tsx
function App() {
  const [query, setQuery] = useState("");
  const [filters, setFilters] = useState<Filter[]>([]);

  const deferredQuery = useDeferredValue(query);
  const deferredFilters = useDeferredValue(filters);

  const results = useMemo(() => {
    return performHeavySearch(deferredQuery, deferredFilters);
  }, [deferredQuery, deferredFilters]);

  // ...
}

// Both lag independently
// Combined update — single transition lane
```

**`useDeferredValue` and Suspense — keep stale UI:**

```tsx
function ProductList({ query }: { query: string }) {
  const data = use(fetchProducts(query)); // suspends
  return <List items={data} />;
}

function App() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  return (
    <Suspense fallback={<Skeleton />}>
      {/* Pass deferredQuery — stale UI kept while new fetches */}
      <ProductList query={deferredQuery} />
    </Suspense>
  );
}

// Initial: Skeleton → results for ""
// User types "phone": query="phone", deferredQuery="" still
// ProductList still renders with deferredQuery="" → no Suspense
// Eventually deferredQuery="phone", new fetch suspends, Skeleton shown OR
// React keeps old UI (with deferredQuery="") visible while new loads
```

</details>

### Edge Cases

- **First render**: `useDeferredValue(value)` — returns value immediately (no lag on first render).
- **`useMemo` dependency**: Use `deferredValue` as dep, not `value` — heavy computation only on deferred update.
- **Server rendering**: Returns initial value (no concurrent on server).

### Follow-up savollar

- "`useDeferredValue` vs `useTransition` — qaysi qachon?" — Value source o'zingiznikimi (`useTransition` setter) yoki tashqi (props, library — `useDeferredValue`). Decision based on control.
- "Multiple `useDeferredValue`'lar parallel?" — Bir transition lane'da group qilinadi (per fiber). Independent transitions emas.
- "`useDeferredValue` tearing'ga moyilmi?" — Yo'q. `useState` based — Concurrent-safe.

</details>

---

### 8. `useTransition` vs `useDeferredValue` — decision tree [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useTransition`** — state setter'ni transition'da o'rash (siz state'ni nazorat qilasiz). **`useDeferredValue`** — passed value'dan deferred copy yaratish (value parent yoki library'dan). Decision: state setter qo'lingizdami → `useTransition`. Faqat value oladigan komponent → `useDeferredValue`.

### Kod misoli

```tsx
// useTransition — own state
function Search() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value);  // urgent
    startTransition(() => {
      setResults(filter(items, e.target.value));  // non-urgent
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <Results results={results} />
    </>
  );
}

// useDeferredValue — value from props/parent
function ResultsList({ query }: { query: string }) {
  const deferredQuery = useDeferredValue(query);  // deferred from parent

  const results = useMemo(
    () => filter(items, deferredQuery),
    [deferredQuery]
  );

  return (
    <ul>
      {results.map((r) => <li key={r.id}>{r.name}</li>)}
    </ul>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**API differences:**

| Aspect | useTransition | useDeferredValue |
|--------|---------------|-------------------|
| Returns | `[isPending, startTransition]` | `deferred` value |
| Wraps | State setter call | Value (any) |
| Use case | Multi-state update | Single value derive |
| isPending | Built-in | Compute via comparison |

**Combine — isPending from useDeferredValue:**

```tsx
function ResultsList({ query }: { query: string }) {
  const deferredQuery = useDeferredValue(query);
  const isPending = query !== deferredQuery;  // ← derived

  return (
    <>
      {isPending && <Spinner />}
      <Results query={deferredQuery} />
    </>
  );
}
```

**Same lane internally:**

Both use TransitionLane — non-urgent. Difference is API ergonomics.

**Performance:**

- `useTransition` — explicit start, clear scope
- `useDeferredValue` — magic (lag detection), simpler API

</details>

### Edge Cases

- **Sync API in startTransition**: `fetch` won't be deferred. Wrap state update only.
- **useDeferredValue initial**: Same as current value. Defer only after subsequent updates.
- **Both in same component**: Each creates own transition.

### Follow-up savollar

- "Compiler auto-applies?" — No. Manual decision.
- "Performance overhead?" — Minimal. Lane bookkeeping ~5%, perceived smoothness huge.

</details>

---

## QISM C: Suspense

### 9. Suspense — code splitting uchun qanday ishlatiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`<Suspense fallback={<Spinner />}>` — komponentlar tree'ning bir qismini async loading paytida fallback bilan almashtiruvchi boundary. Asosiy use case — `React.lazy` bilan code splitting. Lazy komponent birinchi render'ida — chunk yuklanadi (Promise), Suspense Promise'ni catch qiladi va `fallback` ko'rsatadi. Promise resolve bo'lganda — children render. Multiple lazy komponent — bitta Suspense ostida, biri yuklanmaguncha hammasi fallback. Granularity — `Suspense` boundary'larni intuitively joylash kerak (page, section, item — har bo'lakda).

### To'liq tushuntirish

**API:**

```tsx
import { Suspense, lazy } from "react";

const Heavy = lazy(() => import("./Heavy"));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Heavy />
    </Suspense>
  );
}
```

**Suspense boundary lifecycle:**

1. **Initial mount** — children mount attempt, lazy throws Promise
2. **Suspense catches** — fallback rendered
3. **Promise resolves** — re-render attempt
4. **Children render** — fallback unmounted, children visible

**Multiple lazy components:**

```tsx
const Header = lazy(() => import("./Header"));
const Sidebar = lazy(() => import("./Sidebar"));
const Content = lazy(() => import("./Content"));

function App() {
  // Bitta Suspense — 3 ta lazy yuklanmaguncha hammasi fallback
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Header />
      <Sidebar />
      <Content />
    </Suspense>
  );
}

// Alternative — har biri o'z Suspense'i bilan
function AppGranular() {
  return (
    <>
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>
      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>
      <Suspense fallback={<ContentSkeleton />}>
        <Content />
      </Suspense>
    </>
  );
}
// Granular: parts independently load
```

**Granularity strategy:**

- **Page-level**: 1 Suspense for entire page (simplest, but coarse)
- **Section-level**: Per major section (header, sidebar, content)
- **Item-level**: Per list item (best UX, most chunks)

### Kod misoli

**Route-level code splitting:**

```tsx
import { lazy, Suspense } from "react";
import { BrowserRouter, Routes, Route } from "react-router-dom";

const Home = lazy(() => import("./pages/Home"));
const Dashboard = lazy(() => import("./pages/Dashboard"));
const Profile = lazy(() => import("./pages/Profile"));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageSpinner />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/profile" element={<Profile />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

**Modal lazy load:**

```tsx
import { lazy, Suspense, useState } from "react";

const SettingsModal = lazy(() => import("./SettingsModal"));

function Header() {
  const [showSettings, setShowSettings] = useState(false);

  return (
    <header>
      <button onClick={() => setShowSettings(true)}>Settings</button>
      {showSettings && (
        <Suspense fallback={<div>Loading settings...</div>}>
          <SettingsModal onClose={() => setShowSettings(false)} />
        </Suspense>
      )}
    </header>
  );
}
```

**Tab interface — lazy per tab:**

```tsx
import { lazy, Suspense, useState } from "react";

const OverviewTab = lazy(() => import("./tabs/Overview"));
const AnalyticsTab = lazy(() => import("./tabs/Analytics"));
const ReportsTab = lazy(() => import("./tabs/Reports"));

type TabName = "overview" | "analytics" | "reports";

function Dashboard() {
  const [tab, setTab] = useState<TabName>("overview");

  return (
    <>
      <nav>
        <button onClick={() => setTab("overview")}>Overview</button>
        <button onClick={() => setTab("analytics")}>Analytics</button>
        <button onClick={() => setTab("reports")}>Reports</button>
      </nav>
      <Suspense fallback={<TabSkeleton />}>
        {tab === "overview" && <OverviewTab />}
        {tab === "analytics" && <AnalyticsTab />}
        {tab === "reports" && <ReportsTab />}
      </Suspense>
    </>
  );
}

// First click on tab — chunk loads, fallback shown
// Subsequent clicks — instant (chunk cached)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Suspense throw mechanism:**

```typescript
// react-reconciler/src/ReactFiberThrow.js (mental model)
function throwException(root, returnFiber, sourceFiber, value, rootRenderLanes) {
  if (value !== null && typeof value === "object" && typeof value.then === "function") {
    // Thrown a Promise — Suspense
    const wakeable = value;
    let workInProgress = returnFiber;

    do {
      if (workInProgress.tag === SuspenseComponent) {
        // Found Suspense boundary
        const suspenseFlags = workInProgress.flags;
        if ((suspenseFlags & ShouldCapture) === NoFlags) {
          // Mark as captured
          workInProgress.flags |= ShouldCapture;
          attachPingListener(root, wakeable, rootRenderLanes);
          return;
        }
      }
      workInProgress = workInProgress.return;
    } while (workInProgress !== null);

    // No Suspense found — error
    throw new Error("Component suspended without Suspense boundary");
  }
  // Other errors handled by Error Boundary
}
```

**Fallback rendering flow:**

```typescript
// SuspenseFiber render
function updateSuspenseComponent(...) {
  const newProps = workInProgress.pendingProps;
  const showFallback = workInProgress.flags & ShouldCapture;

  if (showFallback) {
    // Render fallback in place of children
    return mountSuspenseFallback(workInProgress, primaryChildren, fallbackChildren, renderLanes);
  } else {
    return mountSuspensePrimaryChildren(workInProgress, primaryChildren, renderLanes);
  }
}
```

**Hidden tree (Offscreen):**

When Suspense shows fallback, primary children rendered as **Offscreen** (hidden but state preserved). After Promise resolves, hidden tree revealed.

```typescript
// R18+ Offscreen mode
// Hidden subtree: rendered but display: none
// State preserved (component instances kept)
// Re-mounted on reveal
```

**Promise tracking:**

```typescript
// attachPingListener
function attachPingListener(root: FiberRoot, wakeable: Wakeable, lanes: Lanes) {
  let pingCache = root.pingCache;
  if (pingCache === null) {
    pingCache = root.pingCache = new WeakMap();
  }

  let threadIDs = pingCache.get(wakeable);
  if (threadIDs === undefined) {
    threadIDs = new Set();
    pingCache.set(wakeable, threadIDs);
  }

  if (!threadIDs.has(lanes)) {
    threadIDs.add(lanes);
    const ping = pingSuspendedRoot.bind(null, root, wakeable, lanes);
    wakeable.then(ping, ping);
  }
}

function pingSuspendedRoot(root, wakeable, pingedLanes) {
  // Wake up Suspense boundary, schedule re-render
  root.pingedLanes = mergeLanes(root.pingedLanes, pingedLanes);
  ensureRootIsScheduled(root);
}
```

**Multiple Promise tracking:**

If multiple lazy components, each throws Promise. Suspense boundary collects all, waits for **all** to resolve before unwrapping fallback.

```typescript
// Internal: Suspense waits for all suspended children
// Re-render attempt after each Promise resolution
// If still suspended → fallback continues
// Once all resolved → primary rendered
```

**Network behavior:**

```
1. Initial render: lazy() chunks not loaded
2. Suspense throws Promise(s) → fallback rendered
3. Browser fetches chunks (parallel if multiple)
4. All chunks loaded → Promise resolves
5. React re-renders, primary children mount
6. Components mount with Hooks initialized
7. Effects run after commit
```

**Concurrent integration:**

In R18 Concurrent mode:
- Suspense rendering is interruptible
- Higher priority updates can interrupt Suspense fallback render
- Once unsuspended, primary children render with appropriate priority

**Error handling:**

If lazy chunk fails to load (network error):
- Promise rejects
- Suspense doesn't re-render fallback (Promise's rejection)
- Error Boundary catches the rejection
- Fallback strategy: error UI with retry button

</details>

### Edge Cases

- **Multiple `Suspense` boundaries**: Inner boundary catches first. Outer only if inner unable to handle.
- **`Suspense` outside lazy**: `Suspense` without lazy children — no fallback shown (unless children throw Promise from `use()` etc).
- **HMR (Hot Module Replacement)**: Module update — lazy re-imports. Suspense fallback during HMR.

### Follow-up savollar

- "`Suspense` event handler ichida ishlaydimi?" — Yo'q. Event handlers — sync code path. Async work — `useEffect` yoki `use()` (R19) bilan.
- "`Suspense` SSR'da ishlaydimi?" — Ha, R18+ streaming SSR. Suspense boundary — server'da fallback emit, client hydrate keyin update.
- "`Suspense` deeply nested OK?" — Ha. Inner boundaries catch first. Strategy: fine-grained (per section) better UX.

</details>

---

### 10. Suspense for data fetching va R19 `use()` Hook [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da `use(promise)` — render paytida Promise'ni unwrap qiluvchi Hook. Promise pending bo'lsa — Suspense boundary catch qiladi (fallback). Resolved bo'lsa — value qaytadi. Rejected bo'lsa — Error Boundary catch. Boshqa Hook'lardan farq: `use()` conditional/loop ichida chaqirilishi mumkin (Rules of Hooks bekor qilinmadi). Use case: server-side data fetch (RSC), client-side resource (cached promise). Promise stable bo'lishi shart — har render yangi promise → infinite suspend. Pattern: `useMemo(() => fetch(...), [deps])`.

### To'liq tushuntirish

**API:**

```tsx
import { use } from "react";

function Component({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise); // Suspends if pending
  return <p>{user.name}</p>;
}

// Wrap in Suspense + ErrorBoundary
<ErrorBoundary fallback={<Error />}>
  <Suspense fallback={<Spinner />}>
    <Component userPromise={fetchUser("123")} />
  </Suspense>
</ErrorBoundary>
```

**`use()` vs `useState`:**

| Aspect | `use()` | `useState` |
|--------|---------|------------|
| Rules of Hooks | Conditional/loop OK | Top-level only |
| Returns | Resolved value | State + setter |
| Suspends | Yes (if pending) | No |
| Error handling | Throws (Error Boundary) | Manual try-catch |

**Promise must be stable:**

```tsx
// ❌ New promise each render → infinite suspend
function BadComponent({ userId }: { userId: string }) {
  const user = use(fetch(`/api/users/${userId}`).then((r) => r.json()));
  return <p>{user.name}</p>;
}

// ✅ Memoized promise
function GoodComponent({ userId }: { userId: string }) {
  const userPromise = useMemo(
    () => fetch(`/api/users/${userId}`).then((r) => r.json()),
    [userId]
  );
  const user = use(userPromise);
  return <p>{user.name}</p>;
}

// ✅✅ Promise from parent (stable reference)
function App() {
  const userPromise = useMemo(() => fetchUser("123"), []);
  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}

function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);
  return <p>{user.name}</p>;
}
```

**Conditional `use()`:**

```tsx
function Component({ shouldLoad, dataPromise }: Props) {
  if (shouldLoad) {
    const data = use(dataPromise); // ✅ Conditional OK
    return <p>{data}</p>;
  }
  return <p>No data</p>;
}
```

### Kod misoli

**RSC pattern (Next.js):**

```tsx
// app/users/[id]/page.tsx (Server Component)
async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

export default function UserPage({ params }: { params: { id: string } }) {
  const userPromise = fetchUser(params.id); // Server promise

  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}

// app/users/[id]/UserProfile.tsx (Client Component)
"use client";

import { use } from "react";

export function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

// Server: starts fetching, streams initial HTML with skeleton
// Client: hydrates skeleton, awaits promise resolution
// Promise resolves → unwrap value → render real UI
```

**Client-side cache integration:**

```tsx
// Cache for stable promises
const cache = new Map<string, Promise<User>>();

function getUserPromise(id: string): Promise<User> {
  if (!cache.has(id)) {
    cache.set(id, fetch(`/api/users/${id}`).then((r) => r.json()));
  }
  return cache.get(id)!;
}

function UserProfile({ userId }: { userId: string }) {
  const user = use(getUserPromise(userId));
  return <p>{user.name}</p>;
}
```

**With `react-query` integration:**

```tsx
import { useSuspenseQuery } from "@tanstack/react-query";

function UserProfile({ userId }: { userId: string }) {
  const { data: user } = useSuspenseQuery({
    queryKey: ["user", userId],
    queryFn: () => fetchUser(userId),
  });
  return <p>{user.name}</p>;
}

// useSuspenseQuery throws Promise — Suspense catches
```

**Conditional context resolution:**

```tsx
import { use } from "react";

function ThemedContent() {
  const theme = use(ThemeContext);
  // ✅ use() inside condition (unlike useContext)
  if (theme.mode === "advanced") {
    const advancedConfig = use(AdvancedConfigContext);
    return <AdvancedView config={advancedConfig} />;
  }
  return <SimpleView theme={theme} />;
}
```

`use(Context)` ham — `useContext` ekvivalenti, lekin conditional ishlatilishi mumkin.

<details>
<summary><strong>Deep Dive</strong></summary>

**`use()` ichi:**

```typescript
// react/src/ReactHooks.js (mental model)
function use<T>(usable: Usable<T>): T {
  if (usable !== null && typeof usable === "object") {
    // Promise (Thenable)
    if (typeof (usable as Thenable<T>).then === "function") {
      const thenable = usable as Thenable<T>;
      return useThenable(thenable);
    }
    // Context
    if ((usable as ReactContext<T>).$$typeof === REACT_CONTEXT_TYPE) {
      const context = usable as ReactContext<T>;
      return readContext(context);
    }
  }
  throw new Error("Invalid argument to use()");
}

function useThenable<T>(thenable: Thenable<T>): T {
  const status = thenable.status;
  if (status === "fulfilled") {
    return thenable.value;
  } else if (status === "rejected") {
    throw thenable.reason;
  }

  // Pending — suspend
  trackUsedThenable(thenable);
  throw thenable;
}
```

**Status tracking:**

```typescript
type Thenable<T> =
  | { status: "fulfilled"; value: T; then: any }
  | { status: "rejected"; reason: unknown; then: any }
  | { status: "pending"; then: any }
  | Promise<T>; // Native Promise — wrap with status

function trackUsedThenable<T>(thenable: Thenable<T>): T {
  if (typeof thenable.then === "function") {
    if (!("status" in thenable)) {
      // Wrap native Promise to track status
      const wrapped = thenable;
      wrapped.then(
        (value) => {
          (wrapped as any).status = "fulfilled";
          (wrapped as any).value = value;
        },
        (reason) => {
          (wrapped as any).status = "rejected";
          (wrapped as any).reason = reason;
        },
      );
      (wrapped as any).status = "pending";
    }
  }
  return thenable as T;
}
```

**Why `use()` doesn't follow Rules of Hooks:**

`use()` doesn't store state in Fiber's hook list (unlike `useState`). It reads from external Promise. Conditional/loop usage doesn't desync hook order.

```typescript
// Internal:
// use() does NOT advance hook pointer
// It directly reads Promise/Context value
```

**Tracking pattern — useMemo for stable Promise:**

```tsx
const promise = useMemo(() => fetchData(id), [id]);
const data = use(promise);
```

`useMemo` ensures Promise is created once per `id`, stable across renders.

**`use()` vs `useEffect` + state:**

```tsx
// Old pattern
function OldComponent({ id }: { id: string }) {
  const [data, setData] = useState<Data | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    setLoading(true);
    fetchData(id).then((d) => {
      setData(d);
      setLoading(false);
    });
  }, [id]);

  if (loading) return <Spinner />;
  return <p>{data?.value}</p>;
}

// New pattern
function NewComponent({ id }: { id: string }) {
  const dataPromise = useMemo(() => fetchData(id), [id]);
  const data = use(dataPromise);
  return <p>{data.value}</p>;
}

<Suspense fallback={<Spinner />}>
  <NewComponent id="123" />
</Suspense>
```

**Benefits:**
- Less boilerplate
- Auto loading state (Suspense fallback)
- Auto error propagation (Error Boundary)
- Better composability

**Promise lifecycle in Suspense:**

```
1. Component renders, calls use(promise)
2. promise.status === "pending" → throw promise
3. Suspense catches → render fallback
4. Suspense attaches .then() listener
5. Promise resolves → status = "fulfilled"
6. Suspense scheduled re-render
7. Component re-renders, use(promise) returns value
8. Children render
```

**Server vs client `use()`:**

- **Server**: Promise resolved on server, value serialized
- **Client**: Promise streamed from server (RSC), or local fetch

**Caveats:**

- Promise must be stable across renders (memoize or pass from stable source)
- Cache implementation responsibility on developer
- Suspense boundary required (else error)

</details>

### Edge Cases

- **`use()` outside Suspense**: If Promise pending → component suspends → no boundary → error.
- **Multiple `use()` calls**: Each independently tracked. All must resolve before children render.
- **Promise rejection without ErrorBoundary**: Unhandled error → propagates up. Error Boundary catches.

### Follow-up savollar

- "`use()` Hook'larning Rules of Hooks'iga rioya qilmaydimi?" — `use()` Fiber state'ga bog'lanmaydi (Promise external). Shu sababli conditional OK.
- "`use()` server uchun yoki client uchun?" — Ikkalasi. RSC'da server promise serialize qilinadi, client'da unwrap.
- "`use(thenable)` vs `await thenable` async function ichida?" — `use()` — render-safe (Suspense). `await` — async function only (RSC server components yoki async event handlers).

</details>

---

### 11. Nested Suspense boundaries — granularity strategy [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Nested Suspense — bir necha boundary'ni tree'da joylash. Asosiy printsip: **innermost boundary catches first**. Bu user uchun granular UX'ni beradi: bir qism yuklanganda boshqa qismlar ham yuklanmasligini ko'rsatadi. Strategy: route-level (page wrapper) → section-level (sidebar, content) → item-level (per row). Trade-off: granular = ko'p chunks, complex tree; coarse = sodda, lekin bitta bo'lak yuklash uzoq bo'lsa, butun page fallback'da. R19'da Suspense — promotion semantics yangilangan, fallback transitions smoother.

### To'liq tushuntirish

**Boundary catching order:**

```tsx
<Suspense fallback={<OuterFallback />}>
  <Suspense fallback={<InnerFallback />}>
    <LazyComponent />
  </Suspense>
</Suspense>

// LazyComponent suspends:
// - InnerFallback shown (innermost catches)
// - OuterFallback NOT shown
```

**Granularity tiers:**

```tsx
// Level 1: App-wide
<Suspense fallback={<AppShell />}>
  <App />
</Suspense>

// Level 2: Route
<Suspense fallback={<PageSkeleton />}>
  <Routes>...</Routes>
</Suspense>

// Level 3: Section
<Suspense fallback={<SidebarSkeleton />}>
  <Sidebar />
</Suspense>

// Level 4: Item
<Suspense fallback={<ItemSkeleton />}>
  <ItemDetails />
</Suspense>
```

**Trade-offs:**

| Granularity | Pros | Cons |
|-------------|------|------|
| Coarse (1 boundary) | Simple, fewer fallbacks | Whole page suspends |
| Medium (3-5 boundaries) | Balanced UX | Moderate complexity |
| Fine (per item) | Best UX, parts load independently | Many chunks, complex |

### Kod misoli

**Multi-level dashboard:**

```tsx
import { Suspense, lazy } from "react";

const Header = lazy(() => import("./Header"));
const Sidebar = lazy(() => import("./Sidebar"));
const MainContent = lazy(() => import("./MainContent"));
const Footer = lazy(() => import("./Footer"));

function Dashboard() {
  return (
    <Suspense fallback={<AppSkeleton />}>
      {/* Outer — entire app fallback */}
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>

      <main className="dashboard-grid">
        <Suspense fallback={<SidebarSkeleton />}>
          <Sidebar />
        </Suspense>

        <Suspense fallback={<ContentSkeleton />}>
          <MainContent />
        </Suspense>
      </main>

      <Suspense fallback={<FooterSkeleton />}>
        <Footer />
      </Suspense>
    </Suspense>
  );
}

// Each section loads independently
// User sees parts as they become available
```

**Item-level granularity (data fetching):**

```tsx
import { Suspense, use } from "react";

interface User {
  id: string;
  name: string;
  posts: Promise<Post[]>; // Streamed promise
}

function UserCard({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);

  return (
    <article>
      <h3>{user.name}</h3>
      {/* Inner boundary for posts */}
      <Suspense fallback={<PostsSkeleton />}>
        <UserPosts postsPromise={user.posts} />
      </Suspense>
    </article>
  );
}

function UserPosts({ postsPromise }: { postsPromise: Promise<Post[]> }) {
  const posts = use(postsPromise);
  return (
    <ul>
      {posts.map((p) => <li key={p.id}>{p.title}</li>)}
    </ul>
  );
}

function App() {
  const usersPromise = useMemo(() => fetchUsers(), []);

  return (
    <Suspense fallback={<UsersSkeleton />}>
      <UsersList usersPromise={usersPromise} />
    </Suspense>
  );
}

function UsersList({ usersPromise }: { usersPromise: Promise<User[]> }) {
  const users = use(usersPromise);
  return (
    <div>
      {users.map((u) => (
        <UserCard
          key={u.id}
          userPromise={Promise.resolve(u)}
        />
      ))}
    </div>
  );
}

// User name shows as soon as user data loads
// Posts load independently per user
```

**Coordination — `SuspenseList` (status note):**

```tsx
// Experimental API — not yet stable
// Coordinates multiple Suspense boundaries

import { unstable_SuspenseList as SuspenseList, Suspense } from "react";

function CoordinatedSuspense() {
  return (
    <SuspenseList revealOrder="forwards">
      <Suspense fallback={<Spinner1 />}><Slow1 /></Suspense>
      <Suspense fallback={<Spinner2 />}><Slow2 /></Suspense>
      <Suspense fallback={<Spinner3 />}><Slow3 /></Suspense>
    </SuspenseList>
  );
}

// revealOrder:
// - "forwards" — show in order (1, 2, 3)
// - "backwards" — show in reverse (3, 2, 1)
// - "together" — all at once
```

**Note:** `SuspenseList` API hozircha unstable, may change in future versions.

<details>
<summary><strong>Deep Dive</strong></summary>

**Boundary traversal — bottom-up:**

```typescript
// React internal — throwException
function findSuspenseBoundary(returnFiber: Fiber): Fiber | null {
  let workInProgress = returnFiber;
  do {
    if (workInProgress.tag === SuspenseComponent) {
      return workInProgress;
    }
    workInProgress = workInProgress.return;
  } while (workInProgress !== null);
  return null;
}
```

Reconciler traverses from suspended Fiber up the tree (return chain), stops at first SuspenseComponent.

**Hidden tree preservation (R18+):**

```tsx
<Suspense fallback={<Spinner />}>
  <ComponentWithState />  {/* useState, useEffect */}
</Suspense>

// First render: ComponentWithState mounts, state initialized
// Suspends → fallback shown
// ComponentWithState's Fiber preserved in hidden tree
// On resume: same Fiber, state preserved
// Effects re-run? No — hidden tree state untouched
```

**Multiple suspenders coordination:**

```typescript
// If 3 components suspend independently:
// Each throws Promise
// Suspense boundary collects all
// Promise.all-like behavior — wait for all
// Only when all resolved → render primary children
```

**`Suspense` and `useTransition`:**

```tsx
function App() {
  const [tab, setTab] = useState("overview");
  const [isPending, startTransition] = useTransition();

  const handleTabChange = (newTab: string) => {
    startTransition(() => setTab(newTab));
  };

  return (
    <>
      <TabSelector onChange={handleTabChange} />
      {isPending && <Spinner />}
      <Suspense fallback={<TabSkeleton />}>
        {tab === "overview" && <OverviewLazy />}
        {tab === "details" && <DetailsLazy />}
      </Suspense>
    </>
  );
}

// Click new tab:
// 1. setTab in transition lane
// 2. New tab component lazy
// 3. Suspense throws Promise
// 4. WITHOUT transition: fallback shows immediately
// 5. WITH transition: React keeps old UI visible (no flash to fallback)
//    isPending = true → can show inline spinner
//    Once new chunk loaded → swap to new tab
```

**`Suspense` fallback as full replacement:**

When suspense triggered, fallback **replaces** primary children. Primary tree hidden but state preserved.

```typescript
// Hidden tree props:
{
  display: "none",
  visibility: "hidden",
  // OR completely off-DOM (Offscreen)
}

// State preserved:
// Hooks state in Fiber preserved
// Effects: cleanup runs, then setup again on reveal
```

**Performance implications:**

```
Coarse Suspense (1 boundary, 1MB chunks):
- Initial: 1 fallback shown
- Load time: 5s
- User experience: blank for 5s

Fine Suspense (10 boundaries, 100KB each):
- Initial: 10 small fallbacks
- Load time: 1s for first chunk, parallel
- User experience: progressive reveal
```

**Streaming SSR + Suspense:**

```tsx
// Server: suspends → emit HTML with placeholder
// Client: receives placeholder, no JS yet
// Server continues streaming chunks
// Client receives chunk, replaces placeholder
// Hydration starts when interactive

// Multiple boundaries:
// Each emits its own placeholder
// Each unblocks independently
// Granular streaming = better TTFB perception
```

**Avoiding fallback flicker:**

```tsx
// ❌ Quick load — fallback flashes
<Suspense fallback={<Spinner />}>
  <FastComponent />  {/* loads in 100ms */}
</Suspense>

// ✅ Delayed fallback (custom)
function DelayedSuspense({ children, fallback, delay = 200 }: Props) {
  const [showFallback, setShowFallback] = useState(false);

  useEffect(() => {
    const timer = setTimeout(() => setShowFallback(true), delay);
    return () => clearTimeout(timer);
  }, []);

  return (
    <Suspense fallback={showFallback ? fallback : null}>
      {children}
    </Suspense>
  );
}

// 100ms load — no fallback shown
// 500ms load — fallback after 200ms
```

</details>

### Edge Cases

- **Suspense after early resolve**: Promise resolves before Suspense catches — no fallback. React's Concurrent rendering checks status synchronously.
- **Boundary update during suspend**: Parent re-renders while child suspending — fallback continues, primary tree updates queued.
- **Re-suspending**: Component already mounted, then re-suspends (e.g., new fetch) — Suspense boundary may show fallback again or keep stale UI (depends on transition).

### Follow-up savollar

- "Suspense fallback'ni qayta o'zgartirish mumkinmi?" — Ha. `<Suspense fallback={fallback}>` har render'da yangi fallback berishi mumkin (re-render bo'lsa).
- "Outer Suspense qachon catch qiladi?" — Inner boundary bypass qilingan bo'lsa (e.g., re-throw). Default — inner first.
- "Suspense + memo bilan ishlash?" — memo bypass — props o'zgarmasa. Suspense boundary state'i bypass'ga ta'sir qilmaydi.

</details>

---

### 12. SuspenseList — coordination patterns (status note) [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`SuspenseList` — bir necha `Suspense` boundary'lar yuklash tartibini koordinatsiya qiluvchi komponent. **Hozirda unstable** (R18-R19) — `unstable_SuspenseList` shaklida. Props: `revealOrder` (`forwards`/`backwards`/`together`), `tail` (`collapsed`/`hidden`). Use case: multiple async sections — controlled reveal order. Fallback: native ordering (random load completion). Future: stable release ko'p versiyalardan keyin (Meta hali interneal'da). Workaround: manual coordination via `useState` flags.

### To'liq tushuntirish

**API (unstable):**

```tsx
import { unstable_SuspenseList as SuspenseList, Suspense } from "react";

interface SuspenseListProps {
  revealOrder?: "forwards" | "backwards" | "together";
  tail?: "collapsed" | "hidden";
  children: React.ReactNode;
}

<SuspenseList revealOrder="forwards" tail="collapsed">
  <Suspense fallback={<S1 />}><Lazy1 /></Suspense>
  <Suspense fallback={<S2 />}><Lazy2 /></Suspense>
  <Suspense fallback={<S3 />}><Lazy3 /></Suspense>
</SuspenseList>
```

**`revealOrder`:**

| Value | Behavior |
|-------|----------|
| `forwards` | Show in tree order (1, then 2, then 3) |
| `backwards` | Show reverse (3, then 2, then 1) |
| `together` | Show all together when all loaded |

**`tail`:**

| Value | Behavior |
|-------|----------|
| `collapsed` | Show only first unloaded fallback |
| `hidden` | Hide fallbacks of unloaded items |

### Kod misoli

**Forward reveal:**

```tsx
import { unstable_SuspenseList as SuspenseList, Suspense } from "react";

const Article = lazy(() => import("./Article"));

function ArticlesList({ articleIds }: { articleIds: string[] }) {
  return (
    <SuspenseList revealOrder="forwards" tail="collapsed">
      {articleIds.map((id) => (
        <Suspense key={id} fallback={<ArticleSkeleton />}>
          <Article id={id} />
        </Suspense>
      ))}
    </SuspenseList>
  );
}

// Article 1 loads in 500ms
// Article 2 loads in 100ms
// Article 3 loads in 300ms

// Without SuspenseList: 2 → 3 → 1 (load order)
// With SuspenseList forwards: 1 → 2 → 3 (tree order)
//   Wait for 1 (500ms), then show 2 (already loaded), then 3 (already loaded)
```

**Together reveal — atomic:**

```tsx
<SuspenseList revealOrder="together">
  <Suspense fallback={<Skeleton1 />}><Section1 /></Suspense>
  <Suspense fallback={<Skeleton2 />}><Section2 /></Suspense>
  <Suspense fallback={<Skeleton3 />}><Section3 /></Suspense>
</SuspenseList>

// Wait for all 3 to load → show all simultaneously
// Use case: synchronized layouts where order matters
```

**Manual coordination workaround (stable):**

```tsx
import { useState, Suspense } from "react";

function CoordinatedList<T>({
  items,
  renderItem,
}: {
  items: T[];
  renderItem: (item: T, onLoad: () => void) => React.ReactNode;
}) {
  const [loadedCount, setLoadedCount] = useState(0);

  const onItemLoad = useCallback(() => {
    setLoadedCount((c) => c + 1);
  }, []);

  return (
    <>
      {items.slice(0, loadedCount + 1).map((item, idx) => (
        <Suspense key={idx} fallback={<Skeleton />}>
          {renderItem(item, onItemLoad)}
        </Suspense>
      ))}
    </>
  );
}

// Manual progressive reveal
// Limited — doesn't handle all SuspenseList cases
```

<details>
<summary><strong>Deep Dive</strong></summary>

**SuspenseList implementation strategy:**

```typescript
// React internal — SuspenseList Fiber tag
const SuspenseListComponent = 19;

// Children traversal:
// 1. Identify all Suspense children
// 2. Track suspense status per child
// 3. Apply revealOrder strategy:
//    - forwards: only reveal child if previous resolved
//    - backwards: only reveal child if subsequent resolved
//    - together: reveal all only when all resolved
```

**Ordering semantics:**

```
Tree:
<SuspenseList revealOrder="forwards">
  <Suspense fallback={<S1>}><A /></Suspense>  {/* loads in 500ms */}
  <Suspense fallback={<S2>}><B /></Suspense>  {/* loads in 100ms */}
  <Suspense fallback={<S3>}><C /></Suspense>  {/* loads in 300ms */}
</SuspenseList>

Timeline:
0ms:    <S1 />, <S2 />, <S3 /> shown
100ms:  B loaded, but waits for A
300ms:  C loaded, but waits for A and B
500ms:  A loaded → show A
        Same render: B already loaded → show B
        Same render: C already loaded → show C

Result: A → B → C in same commit (after 500ms)
```

**`tail="collapsed"`:**

```
Show only first unloaded fallback:

Initial:
<S1 />  ← fallback shown
(S2, S3 hidden)

After A loads:
<A />
<S2 />  ← only first remaining fallback shown
(S3 hidden)

After B loads:
<A />
<B />
<S3 />  ← only first remaining
```

**Concurrent rendering integration:**

`SuspenseList` works with Concurrent rendering. Renders interruptible, ordering maintained across abort/restart.

**Why unstable?**

- API design uncertain (revealOrder values, edge cases)
- Performance characteristics under iteration
- Server rendering interaction
- Composition with other Suspense features

**Alternatives in production:**

```tsx
// 1. Single Suspense (simplest)
<Suspense fallback={<Skeleton />}>
  {/* All async children */}
</Suspense>
// All-or-nothing reveal

// 2. Manual flags
const [section1Loaded, setSection1Loaded] = useState(false);
// ... track loading per section

// 3. react-query / TanStack Query
// useSuspenseQueries — multiple queries with status
```

**`SuspenseList` vs `Promise.all`:**

```typescript
// Promise.all — JS coordination
const [data1, data2, data3] = await Promise.all([
  fetch1(), fetch2(), fetch3()
]);
// All-or-nothing — JS-level

// SuspenseList — React rendering coordination
// Reveal order semantics for UI
```

**Future direction:**

```typescript
// Possibility — composition with R19 actions
// SuspenseList + useFormStatus integration
// Enhanced with Server Actions
```

Stable release timeline: TBD. Track React releases.

</details>

### Edge Cases

- **Mixed sync/async children**: `<SuspenseList>` containing non-Suspense children — those rendered immediately.
- **Nested SuspenseList**: Outer coordinates inner — inner's order depends on outer's strategy.
- **Conditional Suspense in list**: Conditionally rendered Suspense — inclusion in ordering depends on render output.

### Follow-up savollar

- "Production'da SuspenseList ishlatish kerakmi?" — Hozircha `unstable_*` prefix. Stable API kelguncha — manual coordination yoki `together` semantics achievable via single Suspense.
- "Server Components bilan SuspenseList?" — Streaming SSR'da chunk order — natural forwards. SuspenseList client-side coordination uchun.
- "Performance impact?" — Minimal. Coordination logic — small overhead per Fiber.

</details>

---

### 13. `Suspense` + `ErrorBoundary` integratsiya patterns [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Suspense` faqat **Promise**'larni catch qiladi (loading state). **Error**'larni catch qilmaydi — bu Error Boundary'ning vazifasi. Production'da har Suspense boundary atrofida Error Boundary kerak: `<ErrorBoundary fallback={<ErrorUI />}><Suspense fallback={<Spinner />}>{children}</Suspense></ErrorBoundary>`. Order muhim: ErrorBoundary outer (Suspense'ning render error'larini ham catch qiladi). Pattern: shared async-safe wrapper komponent. Library: `react-error-boundary` — re-throw, retry, reset support.

### To'liq tushuntirish

**Why both:**

```tsx
import { Suspense } from "react";
import { ErrorBoundary } from "react-error-boundary";

// ❌ Faqat Suspense — error fall through
<Suspense fallback={<Spinner />}>
  <ComponentThatMayError />  {/* Error → propagates up uncaught */}
</Suspense>

// ✅ Both — comprehensive
<ErrorBoundary FallbackComponent={ErrorUI}>
  <Suspense fallback={<Spinner />}>
    <ComponentThatMayError />
  </Suspense>
</ErrorBoundary>
```

**Order — ErrorBoundary outer:**

```tsx
// ❌ ErrorBoundary inside Suspense
<Suspense fallback={<Spinner />}>
  <ErrorBoundary fallback={<Error />}>
    <Component />
  </ErrorBoundary>
</Suspense>
// Suspense doesn't render fallback for errors — propagates up

// ✅ ErrorBoundary outside Suspense
<ErrorBoundary fallback={<Error />}>
  <Suspense fallback={<Spinner />}>
    <Component />
  </Suspense>
</ErrorBoundary>
// Catches both render errors AND errors during suspended render
```

**Promise rejection scenario:**

```tsx
function Component() {
  const data = use(fetchData()); // Promise may reject
  return <p>{data}</p>;
}

<ErrorBoundary fallback={<NetworkError />}>
  <Suspense fallback={<Spinner />}>
    <Component />
  </Suspense>
</ErrorBoundary>

// fetchData rejects:
// 1. use() throws rejection
// 2. ErrorBoundary catches
// 3. <NetworkError /> rendered
```

### Kod misoli

**Reusable async wrapper:**

```tsx
import { Suspense, ReactNode } from "react";
import { ErrorBoundary, FallbackProps } from "react-error-boundary";

interface AsyncBoundaryProps {
  children: ReactNode;
  loadingFallback?: ReactNode;
  errorFallback?: (props: FallbackProps) => ReactNode;
  onError?: (error: Error) => void;
  onReset?: () => void;
}

function AsyncBoundary({
  children,
  loadingFallback = <Spinner />,
  errorFallback = (p) => <DefaultError {...p} />,
  onError,
  onReset,
}: AsyncBoundaryProps) {
  return (
    <ErrorBoundary
      fallbackRender={errorFallback}
      onError={onError}
      onReset={onReset}
    >
      <Suspense fallback={loadingFallback}>{children}</Suspense>
    </ErrorBoundary>
  );
}

// Usage
function App() {
  return (
    <AsyncBoundary
      loadingFallback={<UserSkeleton />}
      errorFallback={({ error, resetErrorBoundary }) => (
        <div>
          <p>Error: {error.message}</p>
          <button onClick={resetErrorBoundary}>Retry</button>
        </div>
      )}
      onError={(err) => logToTelemetry(err)}
    >
      <UserProfile id="123" />
    </AsyncBoundary>
  );
}
```

**Granular error boundaries:**

```tsx
function Dashboard() {
  return (
    <ErrorBoundary fallback={<AppError />}>
      <Suspense fallback={<DashboardSkeleton />}>
        <Header />

        <main>
          {/* Inner — section-level error handling */}
          <ErrorBoundary fallback={<SidebarError />}>
            <Suspense fallback={<SidebarSkeleton />}>
              <Sidebar />
            </Suspense>
          </ErrorBoundary>

          <ErrorBoundary fallback={<ContentError />}>
            <Suspense fallback={<ContentSkeleton />}>
              <Content />
            </Suspense>
          </ErrorBoundary>
        </main>
      </Suspense>
    </ErrorBoundary>
  );
}

// Sidebar error → SidebarError shown, content unaffected
// Content error → ContentError shown, sidebar unaffected
// Header error → AppError shown (not caught by inner)
```

**Retry pattern:**

```tsx
import { ErrorBoundary } from "react-error-boundary";
import { useState } from "react";

function DataView({ retryKey }: { retryKey: number }) {
  const data = use(fetchData(retryKey)); // retryKey changes → new fetch
  return <div>{data}</div>;
}

function App() {
  const [retryKey, setRetryKey] = useState(0);

  return (
    <ErrorBoundary
      fallbackRender={({ error, resetErrorBoundary }) => (
        <div>
          <p>Failed to load: {error.message}</p>
          <button
            onClick={() => {
              setRetryKey((k) => k + 1);
              resetErrorBoundary();
            }}
          >
            Retry
          </button>
        </div>
      )}
    >
      <Suspense fallback={<Spinner />}>
        <DataView retryKey={retryKey} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

**`react-error-boundary` library:**

```tsx
import { ErrorBoundary, useErrorBoundary } from "react-error-boundary";

function MyComponent() {
  const { showBoundary } = useErrorBoundary();

  const handleAsyncError = async () => {
    try {
      await riskyOperation();
    } catch (err) {
      showBoundary(err); // Manually trigger ErrorBoundary
    }
  };

  return <button onClick={handleAsyncError}>Risky</button>;
}

<ErrorBoundary fallbackRender={({ error }) => <p>{error.message}</p>}>
  <MyComponent />
</ErrorBoundary>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Error vs Promise — different mechanisms:**

```typescript
// React internal — throwException
function throwException(root, returnFiber, sourceFiber, value) {
  // Promise — Suspense catches
  if (typeof value?.then === "function") {
    findSuspenseBoundary(returnFiber);
    // ...
  }
  // Error — ErrorBoundary catches
  else {
    findErrorBoundary(returnFiber);
    // ...
  }
}
```

**ErrorBoundary class:**

```tsx
class ErrorBoundary extends React.Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    // Log to telemetry
    logErrorToService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

**R19 root-level error callbacks:**

```tsx
import { createRoot } from "react-dom/client";

const root = createRoot(document.getElementById("root")!, {
  onCaughtError: (error, errorInfo) => {
    // Error caught by ErrorBoundary
    logTelemetry({ type: "caught", error, errorInfo });
  },
  onUncaughtError: (error, errorInfo) => {
    // Error not caught by any ErrorBoundary
    logTelemetry({ type: "uncaught", error, errorInfo });
  },
  onRecoverableError: (error, errorInfo) => {
    // Hydration mismatch, etc — auto-recovered
    logTelemetry({ type: "recoverable", error, errorInfo });
  },
});

root.render(<App />);
```

**Error during suspended render:**

```typescript
// Component A throws Promise → Suspense catches
// Suspense renders fallback
// Fallback throws Error → who catches?

// Answer: ErrorBoundary above Suspense
// If ErrorBoundary inside Suspense — falls through to outer
```

**Async error in event handler:**

```tsx
// ErrorBoundary doesn't catch async errors in event handlers
function Component() {
  const handleClick = async () => {
    try {
      await riskyAsync();
    } catch (err) {
      // Handle here — ErrorBoundary won't catch
    }
  };
  return <button onClick={handleClick}>...</button>;
}
```

**`use()` Promise rejection — caught:**

```tsx
function Component() {
  const data = use(rejectingPromise); // Rejection thrown
  return <p>{data}</p>;
}

<ErrorBoundary fallback={<Error />}>
  <Suspense fallback={<Spinner />}>
    <Component />
  </Suspense>
</ErrorBoundary>

// rejectingPromise rejects:
// 1. use() throws rejection
// 2. ErrorBoundary catches (above Suspense)
// 3. <Error /> rendered
```

**Recoverable errors (R18+):**

```typescript
// Hydration mismatch detected:
// - HTML on server: <p>Server</p>
// - Client: <p>Client</p>
// React: re-renders client side, logs onRecoverableError
// User-visible: minor flash, but no crash
```

**Error boundary reset semantics:**

```tsx
<ErrorBoundary
  resetKeys={[someKey]}  // Reset when key changes
  onReset={() => clearCache()}
  fallbackRender={({ resetErrorBoundary }) => (
    <button onClick={resetErrorBoundary}>Try again</button>
  )}
>
  <Component />
</ErrorBoundary>
```

`react-error-boundary` provides:
- `resetKeys` — auto-reset on key change
- `onReset` — callback on reset
- `resetErrorBoundary` — manual reset

**Streaming SSR — error during streaming:**

```typescript
// Server: render starts, streams initial HTML
// Component throws during streaming
// Server: emits error boundary fallback HTML
// Client: receives error fallback, no crash
```

R18+ streaming SSR — graceful error handling per chunk.

</details>

### Edge Cases

- **Errors during fallback render**: ErrorBoundary'ning fallback'i o'zi error throw qilsa — outer ErrorBoundary catch qiladi.
- **Hooks errors**: useEffect setup errors caught, but cleanup errors only logged (not by ErrorBoundary).
- **Render errors during transition**: Concurrent rendering — error in transition lane → boundary catches, primary tree unchanged.

### Follow-up savollar

- "ErrorBoundary functional components'da yo'qmi?" — Hozircha class only. RFC bor (`useErrorBoundary` Hook), lekin stable emas.
- "Async error handler — how?" — Try-catch in handler, then `showBoundary(err)` (`react-error-boundary` library) yoki `setState`'da error.
- "Network error retry'ni qanday qilaman?" — `react-query` autoRetry (with backoff). Suspense + retry — manual `retryKey` pattern.

</details>

---

## QISM D: Streaming SSR

### 14. Streaming SSR — concept va R18 implementation [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Streaming SSR — server'da HTML chunked tarzda generate qilib, **birinchi byte tezroq** (TTFB) yuboradi, qolgan kontentni async data resolve bo'lganda streamlaydi. R17 SSR: butun page `renderToString` (sync, all-or-nothing). R18+ `renderToReadableStream` / `renderToPipeableStream`: Suspense boundary'lar — server'da chunk boundary. Suspended part — placeholder HTML, async resolve'da real chunk emit. Client side — `hydrateRoot` chunk-by-chunk hydrate qiladi (selective hydration). Natija: TTFB qisqa, FCP/LCP yaxshilanadi.

### To'liq tushuntirish

**R17 SSR limitations:**

```typescript
// Old API
const html = ReactDOMServer.renderToString(<App />);
// Sync — waits for entire tree
// One big string
// No streaming
// Slow data → slow TTFB
```

**R18+ Streaming API:**

```typescript
// Browser stream
import { renderToReadableStream } from "react-dom/server";

const stream = await renderToReadableStream(<App />);

// Node.js stream
import { renderToPipeableStream } from "react-dom/server";

const { pipe, abort } = renderToPipeableStream(<App />, {
  onShellReady() {
    pipe(response);
  },
});
```

**Streaming flow:**

```
1. Server: render starts
2. Render shell (header, layout, fast content)
3. Encounter suspended component
4. Emit shell HTML + placeholder for suspended
5. Continue rendering — suspended async resolves
6. Emit suspended chunk's HTML
7. Client receives chunks progressively
8. Browser parses HTML as it arrives
9. Suspense boundaries replaced as chunks arrive
```

**Server-side chunk emission:**

```html
<!-- First chunk: shell -->
<!DOCTYPE html>
<html>
<body>
  <div id="root">
    <header>...</header>
    <!--$?-->
    <template id="B:0"></template>
    <!--/$-->  <!-- Placeholder for suspended -->
    <footer>...</footer>
  </div>

  <!-- Subsequent chunk (when suspended resolves): -->
  <div hidden id="S:0">
    <!-- Real content for suspended boundary 0 -->
    <article>...</article>
  </div>
  <script>
    // Patch DOM: replace template with content
    $RC("B:0", "S:0");
  </script>
</body>
</html>
```

`$RC` — runtime helper (small inline script) that swaps placeholder with content.

### Kod misoli

**Express + R18 streaming:**

```tsx
import express from "express";
import { renderToPipeableStream } from "react-dom/server";
import App from "./App";

const app = express();

app.get("*", (req, res) => {
  let didError = false;

  const { pipe, abort } = renderToPipeableStream(<App url={req.url} />, {
    bootstrapScripts: ["/main.js"],
    onShellReady() {
      // Shell rendered — start streaming
      res.statusCode = didError ? 500 : 200;
      res.setHeader("Content-Type", "text/html");
      pipe(res);
    },
    onShellError(error) {
      // Shell failed — fallback HTML
      res.statusCode = 500;
      res.setHeader("Content-Type", "text/html");
      res.send("<h1>Something went wrong</h1>");
    },
    onError(error) {
      didError = true;
      console.error(error);
    },
    onAllReady() {
      // Optional — wait for all (non-streaming mode)
    },
  });

  // Abort if request hangs
  setTimeout(abort, 10000);
});
```

**Modern (Next.js App Router):**

```tsx
// app/page.tsx (Server Component)
import { Suspense } from "react";

async function fetchPosts(): Promise<Post[]> {
  const res = await fetch("/api/posts");
  return res.json();
}

async function fetchUser(): Promise<User> {
  const res = await fetch("/api/user");
  return res.json();
}

// Async Server Component
async function Posts() {
  const posts = await fetchPosts(); // 2s API
  return (
    <ul>
      {posts.map((p) => <li key={p.id}>{p.title}</li>)}
    </ul>
  );
}

async function UserHeader() {
  const user = await fetchUser(); // 500ms API
  return <h1>Hello, {user.name}</h1>;
}

export default function Page() {
  return (
    <>
      <Suspense fallback={<HeaderSkeleton />}>
        <UserHeader />  {/* Streams when user loads (500ms) */}
      </Suspense>

      <Suspense fallback={<PostsSkeleton />}>
        <Posts />  {/* Streams when posts load (2s) */}
      </Suspense>
    </>
  );
}

// Browser timeline:
// 0ms: TTFB — shell HTML received
// 500ms: UserHeader chunk replaces skeleton
// 2000ms: Posts chunk replaces skeleton
// User sees: blank → shell → user → posts (progressive)
```

**Client hydration:**

```tsx
import { hydrateRoot } from "react-dom/client";

hydrateRoot(document, <App />);

// Hydration starts when JS loads
// Selective hydration: priority based on user interaction
// Shell hydrates first, suspended chunks hydrate when streamed
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Server-side rendering modes:**

| Mode | API | Behavior |
|------|-----|----------|
| Sync (R17) | `renderToString` | All-or-nothing, slow TTFB |
| Stream (R18 Node) | `renderToPipeableStream` | Progressive, streaming |
| Stream (Edge) | `renderToReadableStream` | Web Streams API |
| Static | `renderToStaticMarkup` | No hydration metadata |

**`renderToReadableStream` (Edge runtime):**

```typescript
const stream = await renderToReadableStream(<App />, {
  bootstrapScripts: ["/main.js"],
  signal: request.signal, // AbortController
});

return new Response(stream, {
  headers: { "Content-Type": "text/html; charset=utf-8" },
});
```

Used in Cloudflare Workers, Vercel Edge, Deno Deploy.

**Streaming protocol — embedded HTML:**

```html
<!-- Shell with placeholder -->
<div id="root">
  <header>App Header</header>
  <main>
    <!--$?--><template id="B:0"></template><!--/$-->
  </main>
  <footer>App Footer</footer>
</div>

<!-- Late chunk (suspended resolves) -->
<div hidden id="S:0"><article>Real Content</article></div>
<script>
  function $RC(tplId, contentId) {
    const tpl = document.getElementById(tplId);
    const content = document.getElementById(contentId);
    if (tpl && content) {
      tpl.replaceWith(...content.childNodes);
      content.remove();
    }
  }
  $RC("B:0", "S:0");
</script>
```

**Comment markers:**

- `<!--$?-->` — start of suspended boundary (pending)
- `<!--$-->` — start of resolved boundary
- `<!--/$-->` — end of suspense boundary

**Hydration matching:**

```typescript
// hydrateRoot reads:
// - DOM structure
// - Comment markers — Suspense boundaries
// - Hydrate sub-tree by sub-tree

// If chunk hasn't arrived yet:
// - Server placeholder visible
// - Client waits for chunk
// - When chunk arrives + DOM updated, hydrate that sub-tree
```

**Error handling during streaming:**

```typescript
const { pipe, abort } = renderToPipeableStream(<App />, {
  onShellError(err) {
    // Shell render failed — page can't load
    // Send fallback HTML
  },
  onError(err) {
    // Error in streamed chunk — Suspense fallback shown
    // Log for telemetry
  },
});
```

**Backpressure:**

```typescript
// Client slow to receive → server pauses streaming
// pipe() handles backpressure automatically
// Edge runtimes — Web Streams handle natively
```

**Streaming benefits:**

```
Without streaming (R17):
- TTFB: 2000ms (wait for all data)
- FCP: 2100ms
- TTI: 2500ms
- User experience: blank for 2s

With streaming (R18):
- TTFB: 50ms (immediate)
- FCP: 100ms (shell rendered)
- TTI: 800ms (selective hydration)
- User experience: immediate UI, progressive content
```

**`bootstrapScripts` — auto inject:**

```typescript
renderToPipeableStream(<App />, {
  bootstrapScripts: ["/main.js", "/runtime.js"],
});

// Server emits: <script src="/main.js"></script>
// Client downloads + hydrates after shell
```

**Server Actions (R19) — separate concept:**

Streaming SSR — initial render delivery. Server Actions — interactivity (mutations). Composable but distinct.

</details>

### Edge Cases

- **JS disabled**: Shell rendered (no JS). Suspense placeholders remain (no replacement). Critical content above placeholder.
- **Slow client**: Backpressure — server pauses. Total time bounded.
- **Network failure during streaming**: Partial HTML received. React error recovery.

### Follow-up savollar

- "Streaming vs SSG farqi?" — SSG (Static Site Generation) — build-time HTML. Streaming SSR — request-time, async data. Trade-off: cache vs fresh.
- "Streaming bilan SEO?" — Crawlers wait for full response. Streaming chunks — eventually all delivered. SEO compatible.
- "ISR (Incremental Static Regeneration)?" — Next.js feature — cached SSR pages, regenerated on-demand. Different from streaming.

</details>

---

### 15. Selective Hydration (R18) deep mexanizmi [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Selective Hydration — R18'da hydrate qilish jarayoni interruptible va prioritized. SSR HTML allaqachon DOM'da mavjud, lekin **interactivity yo'q** (event listener'lar yo'q). Hydration: React tree'ni walking qilib, har Fiber uchun event listener attach va Hooks state initialization. R17 — sync, blocking. R18 — concurrent, lane-based. User click qilganda hydration **shu bo'lakka prioritize** qilinadi. Suspense boundary — alohida hydration unit. Boundary hydrate bo'lmasdan turib, sub-tree interactive emas (lekin display'i normal).

### To'liq tushuntirish

**Hydration phases:**

```
Server: Render → HTML emitted
                    ↓
Client receives:    DOM tree (visual)
                    ↓
Browser parses:     CSS applied, paints
                    ↓
JS loads:           React bundle ready
                    ↓
hydrateRoot():      Walk tree, attach handlers, init state
                    ↓
Interactive:        Click works, useEffect runs
```

**R17 hydration (sync):**

```typescript
ReactDOM.hydrate(<App />, root);
// Blocking — entire tree at once
// Long page: hydration 200ms+ blocks main thread
// User clicks before hydration → ignored
```

**R18 selective hydration:**

```typescript
import { hydrateRoot } from "react-dom/client";
hydrateRoot(root, <App />);
// Concurrent, interruptible
// Per-Suspense boundary
// User interaction prioritizes that boundary
```

**Hydration order:**

1. **Shell** (initial outer tree) — first
2. **Suspense boundaries** — as data arrives (server-side)
3. **User-clicked area** — prioritized when interaction detected

### Kod misoli

**Selective hydration trace:**

```tsx
function Page() {
  return (
    <>
      <Header />  {/* Hydrates first */}
      <Suspense fallback={<Skeleton />}>
        <Sidebar />  {/* Hydrates when streamed + when clicked */}
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <Content />  {/* Hydrates when streamed + when clicked */}
      </Suspense>
      <Footer />  {/* Hydrates after main */}
    </>
  );
}

// Client timeline:
// 0ms: HTML received, DOM displayed
// 50ms: bundle loaded
// 60ms: hydrateRoot starts
// 70ms: Header hydrated
// 80ms: User clicks Sidebar (Suspense not yet hydrated)
// 80ms: React detects click — pauses Footer hydration
// 90ms: Sidebar hydrated, click handler runs
// 100ms: Footer hydration resumes
// 110ms: Content hydrates (eventually)
```

**Streaming + hydration:**

```tsx
// Server emits initial chunk:
<div id="root">
  <header>App</header>
  <!-- Suspended boundaries shown as skeletons -->
  <!--$?--><template id="B:0"></template><!--/$-->
</div>
<script src="/main.js"></script>

// Client loads main.js, hydrateRoot starts
// Header hydrates immediately

// Server later emits suspended chunk:
<div hidden id="S:0"><article>Content</article></div>
<script>$RC("B:0", "S:0")</script>

// DOM updated: skeleton replaced with content
// React detects new content, hydrates Suspense sub-tree

// User can interact with hydrated parts
// Non-hydrated parts: visual but click-disabled
```

**Click before hydration — replay:**

```tsx
function App() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
}

// Server HTML: <button>0</button>
// User clicks before hydration:
// 1. Browser bubbles click to button
// 2. No JS handler — nothing happens (R17 dropped)
// 3. R18: React captures click in delegation
// 4. After hydration: replays click
// 5. setCount runs: count = 1
```

R18 — `discreteUpdates` — captures user input pre-hydration, replays after.

<details>
<summary><strong>Deep Dive</strong></summary>

**`hydrateRoot` internal:**

```typescript
// react-dom/src/client/ReactDOMRoot.js (mental model)
function hydrateRoot<T>(
  container: Element,
  initialChildren: ReactNode,
  options?: HydrateOptions
): ReactDOMRoot {
  const root = createHydrationContainer(...);
  scheduleInitialHydrationOnRoot(root, lane);
  return new ReactDOMHydrationRoot(root);
}
```

**Hydration uses Fiber tree walking:**

```typescript
// react-reconciler/src/ReactFiberHydrationContext.js
function tryToClaimNextHydratableInstance(fiber: Fiber) {
  const nextInstance = nextHydratableInstance;
  if (nextInstance === null) {
    throwOnHydrationMismatch();
    return;
  }
  // Match Fiber type with DOM node type
  // Attach Fiber to DOM node
  fiber.stateNode = nextInstance;
  nextHydratableInstance = nextHydratableSibling(nextInstance);
}
```

**Comment marker parsing:**

```typescript
// Detect Suspense boundaries
function isSuspenseInstancePending(node: Comment): boolean {
  return node.data === "$?"; // Pending
}

function isSuspenseInstanceFallback(node: Comment): boolean {
  return node.data === "$!"; // Failed
}

function getSuspenseInstanceFallbackErrorDetails(...) {
  // Extract error info if needed
}
```

**Lane-based hydration scheduling:**

```typescript
// Hydration lanes
SyncHydrationLane = 1;
DefaultHydrationLane = 16;

// User interaction during hydration:
// 1. Click event captured
// 2. Find target Fiber's Suspense boundary
// 3. Schedule SyncHydrationLane on that boundary
// 4. Reconciler picks up high priority — hydrate that boundary first
// 5. Replay click event after hydration
```

**Selective hydration prioritization:**

```typescript
// Internal pseudo:
function attemptDiscreteHydration(fiber: Fiber) {
  // Find suspended boundary
  let boundary = fiber;
  while (boundary !== null) {
    if (boundary.tag === SuspenseComponent && isHydratingBoundary(boundary)) {
      // Schedule sync hydration
      scheduleUpdateOnFiber(boundary, SyncHydrationLane);
      return;
    }
    boundary = boundary.return;
  }
}

// Event listener (capture phase):
document.addEventListener("click", (e) => {
  const target = e.target;
  const fiber = getClosestInstanceFromNode(target);
  if (fiber && isHydrating(fiber)) {
    e.preventDefault();
    e.stopPropagation();
    attemptDiscreteHydration(fiber);
    // Replay event after hydration
    queueDiscreteEvent(...);
  }
});
```

**Hydration mismatch detection:**

```typescript
function tryToClaimNextHydratableInstance(fiber: Fiber) {
  const nextInstance = nextHydratableInstance;
  if (canHydrateInstance(fiber, nextInstance)) {
    // OK — claim
    fiber.stateNode = nextInstance;
  } else {
    // Mismatch detected
    if (didNotMatchHydratedTextInstance(...)) {
      // Recover via re-render
      onRecoverableError(error);
    } else {
      throwOnHydrationMismatch();
    }
  }
}
```

R18+ — recoverable mismatches: log warning, continue.

**Server vs client tree comparison:**

```typescript
// Server rendered:
<p>Hello, {userName}</p>  // userName = "Server"

// Client rendered:
<p>Hello, {userName}</p>  // userName = "Client" (different state)

// Hydration: text mismatch
// React: log onRecoverableError
// Re-render with client state
// Visual flash possible
```

**Hydration boundary marker structure:**

```html
<!-- Suspense boundary -->
<!--$-->    <!-- Hydrated -->
<div>...</div>
<!--/$-->

<!--$?-->   <!-- Pending -->
<template id="B:0"></template>
<!--/$-->

<!--$!-->   <!-- Error -->
<!--/$-->
```

**Concurrent hydration with `useTransition`:**

```tsx
// Selective hydration uses transition lanes
// User interaction: SyncHydrationLane (high)
// Default: DefaultHydrationLane (low)
// Yields to browser
```

**Performance characteristics:**

```
Pre-R18 hydration (1MB JS, 100KB HTML):
- Hydration: 200ms blocking
- Time to Interactive: 250ms
- User clicks during hydration → ignored

R18 selective hydration:
- Initial: 50ms (shell)
- User interaction → prioritized boundary: 80ms
- Total time slightly higher (yields), but TTI better
- Click pre-hydration → replayed
```

**Best practices:**

1. **Granular Suspense boundaries** — better selective hydration
2. **`<Suspense>` near user interaction** — prioritized first
3. **Lazy bundle** for non-critical paths
4. **Avoid hydration mismatches** — server/client state alignment

</details>

### Edge Cases

- **`useEffect` order during hydration**: Effects run after commit. Hydration commits per boundary. Effects fire as boundaries hydrate.
- **`useState` initializer mismatch**: Server vs client different — hydration mismatch warning. Use `getServerSnapshot` in `useSyncExternalStore`.
- **Client-only state (`window`, etc.)**: Use `useSyncExternalStore` with `getServerSnapshot` providing safe fallback.

### Follow-up savollar

- "Selective hydration — Concurrent Mode'ga bog'liqmi?" — Ha. `createRoot`/`hydrateRoot` (Concurrent) majburiy.
- "Hydration mismatch'ni qanday minimize qilish?" — Server va client'da bir xil source of truth. Date.now/Math.random — `useEffect` ichida.
- "`useId` hydration uchun?" — Server va client'da bir xil ID generate qiladi (Fiber tree position based) — hydration-safe.

</details>

---

### 16. Progressive Hydration patterns [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Progressive Hydration — sahifa qisman hydrate qilingan holatga ruxsat berish, asosan **above-fold** content tezroq interactive bo'lishi uchun. R18'da Suspense boundary'lar — progressive hydration unit'lari. Strategy: critical content (header, hero) — eager hydrate; below-fold (footer, comments) — lazy/deferred. Patterns: Island Architecture (Astro), Selective Hydration (R18), Defer Hydration (Qwik). Performance: TTI yaxshilanadi (less initial JS to execute), bundle splitting effective.

### To'liq tushuntirish

**Progressive vs full hydration:**

```
Full hydration (R17):
- Entire app hydrated at once
- All event listeners attached upfront
- TTI = total hydration time
- 1MB JS → 200ms hydration

Progressive (R18+):
- Critical paths hydrate first
- Lazy boundaries hydrate on interaction
- TTI = critical path hydration
- Total JS amortized over time
```

**Strategies:**

1. **Suspense-based** (R18) — already discussed
2. **Island Architecture** (Astro, Marko) — discrete islands of interactivity
3. **Defer hydration** — `<Suspense>` boundary that doesn't load JS until interaction
4. **Resumability** (Qwik) — no hydration, serialize state directly

**R18 progressive hydration:**

```tsx
function App() {
  return (
    <>
      <Header />  {/* Hydrates first */}
      <Hero />    {/* Hydrates first */}

      <Suspense fallback={<Skeleton />}>
        <BelowFoldContent />  {/* Hydrates when streamed */}
      </Suspense>

      <Suspense fallback={<Skeleton />}>
        <Footer />  {/* Hydrates last */}
      </Suspense>
    </>
  );
}
```

### Kod misoli

**Lazy hydration with intersection observer:**

```tsx
import { useState, useEffect, useRef, lazy, Suspense } from "react";

interface LazyHydrateProps {
  children: React.ReactNode;
  fallback: React.ReactNode;
}

function LazyHydrate({ children, fallback }: LazyHydrateProps) {
  const ref = useRef<HTMLDivElement>(null);
  const [hydrated, setHydrated] = useState(false);

  useEffect(() => {
    if (!ref.current) return;
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting) {
          setHydrated(true);
          observer.disconnect();
        }
      },
      { rootMargin: "100px" }
    );
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  return (
    <div ref={ref}>
      {hydrated ? children : fallback}
    </div>
  );
}

// Usage
function App() {
  return (
    <>
      <Header />
      <Hero />
      {/* Below-fold content lazy hydrates */}
      <LazyHydrate fallback={<Skeleton />}>
        <Comments />
      </LazyHydrate>
    </>
  );
}
```

**Interaction-based hydration:**

```tsx
function InteractiveOnHover({ children }: { children: React.ReactNode }) {
  const [hydrated, setHydrated] = useState(false);

  return (
    <div
      onMouseEnter={() => setHydrated(true)}
      onFocus={() => setHydrated(true)}
    >
      {hydrated ? children : <SkeletonOrPlaceholder />}
    </div>
  );
}
```

**Island architecture (manual):**

```tsx
// Each "island" is independently hydrated
// Useful for content-heavy sites with sparse interactivity

const NewsletterIsland = lazy(() => import("./NewsletterSignup"));
const ChatIsland = lazy(() => import("./LiveChat"));

function Article() {
  return (
    <article>
      <h1>Article Title</h1>
      <p>Static content...</p>

      {/* Island 1 — interactive */}
      <Suspense fallback={null}>
        <NewsletterIsland />
      </Suspense>

      <p>More static content...</p>

      {/* Island 2 — interactive */}
      <Suspense fallback={null}>
        <ChatIsland />
      </Suspense>
    </article>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Bundle splitting impact:**

```
Pre-progressive:
- main.js: 1MB (includes everything)
- TTI: 200ms+

Progressive:
- main.js: 200KB (critical path)
- below-fold.js: 300KB (lazy)
- footer.js: 100KB (lazy)
- TTI: 50ms (critical only)
```

**Architecture comparisons:**

| Approach | Hydration | Trade-off |
|----------|-----------|-----------|
| MPA (multi-page) | None | Full reload, slow nav |
| SPA (R17) | Full | Long TTI, slow start |
| SSR + R17 hydration | Full | Sync block, slow TTI |
| SSR + R18 selective | Granular | Concurrent, faster TTI |
| Astro Islands | Per-island | Static + interactive |
| Qwik resumability | None (serialized state) | Complex tooling |

**`<Suspense>` defer hydration pattern:**

```tsx
// react-dom v19+: defer hydration via dynamic import
const DeferredButton = lazy(() => import("./Button"));

function Page() {
  return (
    <>
      <h1>Static Content</h1>
      <Suspense fallback={<button>Loading...</button>}>
        {/* JS loaded only when this section is needed */}
        <DeferredButton />
      </Suspense>
    </>
  );
}

// SSR: emit static button placeholder
// Client: hydrate when JS loads / user interacts
```

**Resumability vs Hydration:**

```
Hydration (React):
1. Server renders HTML + serializes state in JSON
2. Client downloads HTML + JSON + JS bundle
3. JS bundle reconstructs component tree (Fiber)
4. Attaches event handlers
5. Sets up internal state

Resumability (Qwik):
1. Server renders HTML + inline event handler hints
2. Client downloads HTML
3. NO bundle needed initially
4. Click → load specific handler chunk
5. Execute → resume

Trade-off:
- React: more upfront cost, simpler model
- Qwik: minimal upfront, complex tooling
```

**Hydration cost breakdown:**

```typescript
// Per component:
// 1. Component function called: ~5-50µs
// 2. Hooks initialized: ~0.5µs each
// 3. JSX evaluated: ~5-10µs
// 4. Event listeners attached: ~5-10µs each
// 5. Effects scheduled: ~1µs each

// 1000 components:
// ~50-100ms hydration time
```

**Selective hydration triggers:**

R18 — automatic prioritization:
- User interaction (click, key, focus) → hydrate target's boundary first
- Visible (intersection observer) → can prioritize via custom logic
- Above-fold → naturally first in tree order

**Hydration error recovery:**

```typescript
// Mismatch in hydration:
// R17: client throws error, full re-render
// R18: log warning, recover via re-render of mismatched section
// onRecoverableError callback fired

const root = hydrateRoot(container, <App />, {
  onRecoverableError(error, errorInfo) {
    logTelemetry({ type: "hydration_recovery", error });
  },
});
```

**Performance budgets:**

```
Initial JS bundle: < 100KB gzipped (critical path)
TTI: < 800ms on 3G
Hydration: < 100ms per Suspense boundary
First Input Delay: < 100ms
INP: < 200ms
```

**Production patterns:**

1. **Above-fold critical** — eager hydrate (header, hero, primary CTA)
2. **Modals/dialogs** — lazy hydrate on trigger
3. **Below-fold** — defer hydrate via Intersection Observer
4. **Footer** — last hydrate
5. **Third-party widgets** (chat, analytics) — async load + hydrate

**Image lazy loading equivalent:**

```html
<img src="..." loading="lazy" />  <!-- Browser native -->
```

For React components — manual via `IntersectionObserver` (no built-in API yet).

**Future: React Server Components (RSC):**

```
RSC reduces hydration cost:
- Static parts — server-rendered, no client JS
- Interactive parts — only those need hydration
- Bundle smaller
```

</details>

### Edge Cases

- **Hydration mismatch in lazy section**: First render of lazy boundary creates new tree (no hydration needed for new content).
- **Loading state visible during deferred hydration**: User clicks while not yet hydrated — interaction queued, runs after.
- **Multiple boundaries hydrating simultaneously**: Concurrent — yields among them. Priority by interaction.

### Follow-up savollar

- "Progressive vs Suspense — bir narsami?" — Suspense — mechanism. Progressive hydration — strategy. Suspense — building block, you compose for progressive.
- "Bundle size impact'ni qanday o'lchash?" — Webpack/Vite Bundle Analyzer. Bundle visualizer. Per-chunk size tracking.
- "Browser native lazy hydration?" — Yo'q. React abstraction'i. Standardlashtirish RFC bor.

</details>

---

## QISM E: Output va Bug Fix

### 17. Output savol — `useTransition` behavior [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa savol

```tsx
import { useState, useTransition } from "react";

function App() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState("a");
  const [isPending, startTransition] = useTransition();

  console.log(`render: count=${count}, text=${text}, pending=${isPending}`);

  const handleClick = () => {
    setCount((c) => c + 1); // SyncLane
    startTransition(() => {
      setText((t) => t + "b"); // TransitionLane
    });
  };

  return <button onClick={handleClick}>Click</button>;
}
```

**Savol:** Click bosilganda console.log'lar ketma-ketligi qanday bo'ladi?

### Javob

**Initial mount:**
```
render: count=0, text=a, pending=false
```

**1-click:**
```
render: count=1, text=a, pending=true
render: count=1, text=ab, pending=false
```

Qadamlar:
1. `setCount(c => c + 1)` — SyncLane (event handler)
2. `startTransition(...)` — `setText((t) => t + "b")` TransitionLane bilan
3. `useTransition` ham `setIsPending(true)` SyncLane'da chaqiradi
4. Sync render: `count=1`, `text=a` (stale — transition hali bajarilmagan), `pending=true`
5. Concurrent render starts (TransitionLane): `text` yangilanadi
6. Final commit: `count=1`, `text=ab`, `pending=false` (transition tugadi)

**2-click (yana):**
```
render: count=2, text=ab, pending=true
render: count=2, text=abb, pending=false
```

Bir xil pattern. Har click — 2 ta render.

<details>
<summary><strong>Deep Dive</strong></summary>

**`useTransition` ichida nima bo'ladi:**

```typescript
// Mental model
function useTransition() {
  const [isPending, setIsPending] = useState(false);

  const startTransition = (callback) => {
    setIsPending(true); // SyncLane (high priority)

    const prevTransition = ReactCurrentBatchConfig.transition;
    ReactCurrentBatchConfig.transition = {};
    try {
      setIsPending(false); // TransitionLane (low priority)
      callback();
    } finally {
      ReactCurrentBatchConfig.transition = prevTransition;
    }
  };

  return [isPending, startTransition];
}
```

`setIsPending(true)` — SyncLane (immediate UI feedback).
`setIsPending(false)` — TransitionLane (when transition commits).

**Render lanes flush order:**

```
Click event handler runs:
  - setCount(c+1) → SyncLane
  - setIsPending(true) → SyncLane
  - startTransition: open transition scope
  - setIsPending(false) → TransitionLane
  - setText(t+"b") → TransitionLane
  - close scope

React schedules:
  1. SyncLane render (first)
  2. TransitionLane render (after)
```

**Sync render output:**

```
State at sync render:
  count = 1 (new)
  text = "a" (old — transition hasn't run)
  isPending = true (new)
→ console.log: render: count=1, text=a, pending=true
```

**Concurrent render (TransitionLane):**

```
React continues with TransitionLane:
  text = "ab" (new — transition update applied)
  isPending = false (new — transition completing)
→ console.log: render: count=1, text=ab, pending=false
```

**Why count stays 1 in transition render?**

`count` was already updated in sync render. Subsequent transition render — same `count` (no new update for it). Only `text` and `isPending` change.

**Discrete batching:**

Both SyncLane and TransitionLane updates batched within event handler. R18 automatic batching extends to all events.

**Variation — without `startTransition`:**

```tsx
const handleClick = () => {
  setCount((c) => c + 1);
  setText((t) => t + "b"); // Both DefaultLane (or SyncLane in event)
};

// Single render:
// render: count=1, text=ab, pending=false (always false)
```

**Variation — `flushSync`:**

```tsx
import { flushSync } from "react-dom";

const handleClick = () => {
  flushSync(() => {
    setCount((c) => c + 1);
  });
  // First render flushed synchronously
  startTransition(() => {
    setText((t) => t + "b");
  });
};

// Renders:
// render: count=1, text=a, pending=false
// render: count=1, text=a, pending=true
// render: count=1, text=ab, pending=false
```

`flushSync` forces immediate render.

**Strict Mode dev 2x:**

```
With <StrictMode>:
render: count=0, text=a, pending=false  (initial 1st)
render: count=0, text=a, pending=false  (initial 2nd — strict)

Click:
render: count=1, text=a, pending=true   (sync 1st)
render: count=1, text=a, pending=true   (sync 2nd)
render: count=1, text=ab, pending=false (transition 1st)
render: count=1, text=ab, pending=false (transition 2nd)
```

</details>

### Edge Cases

- **`setCount` and `setText` aynan teng oldingi qiymatlar**: `Object.is` bailout — render trigger qilinmaydi.
- **Click rapid (transition incomplete)**: New click while transition pending — abort + restart with new state.
- **Strict Mode**: 2x renders for both sync and transition.

### Follow-up savollar

- "Why two renders, not one?" — Different priorities (Sync, Transition) — separate render passes.
- "`isPending` short-lived?" — Yes. True only between sync render and transition commit. Microseconds-milliseconds.
- "Without React 18 createRoot — same behavior?" — No. Legacy mode (R17 ReactDOM.render) — sync only, no transition lanes. `useTransition` becomes no-op.

</details>

---

### 18. Output savol — Suspense fallback timing [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa savol

```tsx
import { Suspense, useState, lazy } from "react";

const Heavy = lazy(() =>
  // Simulate 1s load
  new Promise((resolve) => {
    setTimeout(() => resolve({ default: HeavyImpl }), 1000);
  })
);

function HeavyImpl() {
  console.log("Heavy render");
  return <p>Heavy</p>;
}

function App() {
  const [show, setShow] = useState(false);
  console.log(`App render: show=${show}`);

  return (
    <>
      <button onClick={() => setShow(true)}>Show</button>
      {show && (
        <Suspense fallback={<p>Loading...</p>}>
          <Heavy />
        </Suspense>
      )}
    </>
  );
}
```

**Savol:** Mount + click + 1s wait — log ketma-ketligi va DOM holati har qadamda?

### Javob

**Mount:**
```
App render: show=false
```
DOM:
```
<button>Show</button>
```

**Click (immediately):**
```
App render: show=true
```
DOM:
```
<button>Show</button>
<p>Loading...</p>
```

(Heavy lazy import boshlandi, Promise pending → Suspense fallback)

**After 1s (Promise resolves):**
```
Heavy render
```
DOM:
```
<button>Show</button>
<p>Heavy</p>
```

(Promise resolved → React re-renders, Heavy mounts)

**Click again (Show pressed when Heavy already loaded):**

Hech narsa o'zgarmaydi (`show` bir xil — bailout). Cache'lan Heavy loaded.

<details>
<summary><strong>Deep Dive</strong></summary>

**`React.lazy` cache mechanism:**

```typescript
// LazyComponent state machine
type LazyState =
  | { _status: -1; _result: () => Promise<...> } // Uninitialized
  | { _status: 0; _result: Promise<...> }        // Pending
  | { _status: 1; _result: ComponentType }       // Resolved
  | { _status: 2; _result: Error };              // Rejected

// Once resolved, status = 1 forever
// Subsequent renders — instant return component
```

**Suspense fallback transition:**

```typescript
// State: show=true, lazy pending
// Reconciler renders:
// 1. Encounter <Heavy /> Fiber, tag = LazyComponent
// 2. lazy._init() called, status = 0 (pending)
// 3. Throws Promise
// 4. Suspense boundary catches
// 5. Render fallback in place
// 6. attachPingListener — wait for Promise resolve
```

**On Promise resolve:**

```typescript
// pingSuspendedRoot:
// 1. Mark root.pingedLanes
// 2. Schedule re-render
// 3. Reconciler retries from Suspense boundary
// 4. <Heavy /> renders, lazy now status=1
// 5. lazy._result is HeavyImpl
// 6. HeavyImpl rendered → "Heavy render" log
```

**Hidden tree behavior:**

In R18, suspended primary children are placed in **hidden** tree (not unmounted). State preserved if it had any.

But `Heavy` is a fresh import — no prior state. So it mounts fresh on resolve.

**Variation — show=false then true rapidly:**

```tsx
// Click Show, then Hide before lazy resolves:
setShow(true);  // Heavy starts loading
setShow(false); // Heavy unmounts (Suspense unmounts)
// Lazy load continues in background (not cancelled)
// When resolves — cached, no DOM update
```

**Variation — Promise rejection:**

```tsx
const Heavy = lazy(() =>
  new Promise((resolve, reject) => {
    setTimeout(() => reject(new Error("Failed")), 1000);
  })
);

// Click:
// App render: show=true
// (1s pass)
// Error thrown — propagates up
// If no ErrorBoundary → uncaught error
```

**Variation — `useTransition`:**

```tsx
const [isPending, startTransition] = useTransition();

const handleClick = () => {
  startTransition(() => setShow(true));
};

// Click:
// App render: show=false (sync, isPending=true)
// (transition lane starts)
// Heavy lazy throws
// Suspense fallback NOT shown (transition keeps old UI)
// Wait 1s for lazy
// Heavy renders
// App render: show=true (transition commits)
// User: button → button + Heavy (no flash to "Loading...")
```

`useTransition` + Suspense — keeps previous UI visible during transition.

**Network tab observation:**

```
// Click time: 0ms
// Network: lazy chunk request fired
// Browser fetches chunk
// 1000ms: chunk arrives
// React: re-render, Heavy mounts
```

**Strict Mode dev:**

Initial mount logs 2x:
```
App render: show=false
App render: show=false
```

Click logs 2x for both sync and resolved:
```
App render: show=true
App render: show=true
(1s wait)
Heavy render
Heavy render
```

</details>

### Edge Cases

- **Show toggled rapidly**: Lazy load happens once (cached). Suspense fallback briefly visible.
- **Multiple lazy components**: Each independent Promise. Suspense waits for all.
- **`show` reset to false during loading**: Lazy load continues. When complete, but `show` is false → no render.

### Follow-up savollar

- "Heavy mount qila olmasa — fallback nima?" — Hozirgi Suspense fallback. ErrorBoundary outer — error catch.
- "Initial mount'da `show=true` bo'lsa — fallback ko'rinadimi?" — Ha, mount paytida Heavy lazy bo'lsa, fallback render.
- "Heavy import-time error — qachon throw bo'ladi?" — `lazy()` chaqirilganda emas (lazy). Birinchi render'da, lazy._init() ichida.

</details>

---

### 19. Bug fix savol — race condition + transition [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa savol

```tsx
import { useState, useTransition, useEffect } from "react";

interface User {
  id: string;
  name: string;
}

function UserSearch() {
  const [query, setQuery] = useState("");
  const [users, setUsers] = useState<User[]>([]);
  const [isPending, startTransition] = useTransition();

  useEffect(() => {
    fetch(`/api/users?q=${query}`)
      .then((r) => r.json())
      .then((data) => {
        startTransition(() => {
          setUsers(data);
        });
      });
  }, [query]);

  return (
    <>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
      />
      {isPending && <Spinner />}
      <ul>
        {users.map((u) => <li key={u.id}>{u.name}</li>)}
      </ul>
    </>
  );
}
```

**Bug:** Bu kodda race condition bor. User tez yozsa, oldingi fetch'lar (slow) yangi fetch'lardan keyin keladi → eski natijalar ko'rsatiladi.

**Savol:** Bug'ni aniqlang va tuzating.

### Javob

**Bug:** Multiple in-flight fetch'lar — har keystroke yangi fetch boshlaydi. Fetch order kafolat berilmaydi (network latency varies). Eski fetch yangi fetch'dan keyin resolve qilsa — eski natijalar `setUsers` chaqirib, UI eski'ga qaytadi.

**Yechim 1 — `AbortController`:**

```tsx
useEffect(() => {
  const controller = new AbortController();

  fetch(`/api/users?q=${query}`, { signal: controller.signal })
    .then((r) => r.json())
    .then((data) => {
      startTransition(() => {
        setUsers(data);
      });
    })
    .catch((err) => {
      if (err.name !== "AbortError") {
        console.error(err);
      }
    });

  return () => controller.abort(); // Cleanup — abort previous
}, [query]);
```

**Yechim 2 — Stale flag:**

```tsx
useEffect(() => {
  let stale = false;

  fetch(`/api/users?q=${query}`)
    .then((r) => r.json())
    .then((data) => {
      if (stale) return;
      startTransition(() => {
        setUsers(data);
      });
    });

  return () => {
    stale = true; // Mark previous as stale
  };
}, [query]);
```

**Yechim 3 — R19 `use()` + Suspense:**

```tsx
import { use, useMemo, Suspense } from "react";

function UserList({ query }: { query: string }) {
  const promise = useMemo(
    () => fetch(`/api/users?q=${query}`).then((r) => r.json()),
    [query]
  );
  const users = use(promise);
  return (
    <ul>
      {users.map((u: User) => <li key={u.id}>{u.name}</li>)}
    </ul>
  );
}

function UserSearch() {
  const [query, setQuery] = useState("");
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Suspense fallback={<Spinner />}>
        <UserList query={query} />
      </Suspense>
    </>
  );
}

// React handles staleness via Suspense
// Each query change → new promise → Suspense rerenders
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Race condition timeline:**

```
Time:  0ms      100ms     200ms     300ms     400ms     500ms
Type:  "a"      "ab"      "abc"
Fetch: A starts B starts  C starts
Resp:                              C resolves     A resolves     B resolves
                                   setUsers(C)    setUsers(A)    setUsers(B)
                                   UI: C results  UI: A (stale!) UI: B (stale!)
```

**Why slowest fetch wins (without fix):**

JS network requests not ordered. Server may respond in any order. Fastest wins UI temporarily, slower overrides later.

**`AbortController` semantics:**

```typescript
const controller = new AbortController();

fetch(url, { signal: controller.signal })
  .then(...)
  .catch((err) => {
    if (err.name === "AbortError") {
      // Aborted — don't process
    }
  });

// Later:
controller.abort(); // Cancels in-flight request
// Promise rejects with AbortError
```

**Cleanup function timing:**

```typescript
useEffect(() => {
  // Setup: query="a", fetch A
  return () => {
    // Cleanup: when query changes to "ab"
    // Abort A
  };
}, [query]);

// Order:
// 1. query changes to "ab"
// 2. React: prepare to re-run effect
// 3. Run cleanup of previous effect → abort A
// 4. Run new effect → fetch B
// 5. ... query changes to "abc" ...
// 6. Cleanup → abort B
// 7. New effect → fetch C
// 8. C completes → setUsers(C) — only fetch alive
```

**Stale flag advantages:**

- Works without AbortController (older browsers / non-fetch APIs)
- Doesn't require server to handle abort
- Lighter weight (no API call cancellation overhead)

**Stale flag disadvantages:**

- Network bandwidth wasted (response received but ignored)
- Server resources wasted

**Choose strategy:**

| Use case | Strategy |
|----------|----------|
| Fetch with abort support | `AbortController` |
| Non-fetch APIs (legacy XHR) | Stale flag |
| WebSocket / SSE | Custom cleanup logic |
| Library (react-query, swr) | Built-in handling |

**`react-query` solution:**

```tsx
import { useQuery } from "@tanstack/react-query";

function UserSearch() {
  const [query, setQuery] = useState("");

  const { data: users = [], isLoading } = useQuery({
    queryKey: ["users", query],
    queryFn: () => fetch(`/api/users?q=${query}`).then((r) => r.json()),
    keepPreviousData: true,
  });

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {isLoading && <Spinner />}
      <ul>
        {users.map((u: User) => <li key={u.id}>{u.name}</li>)}
      </ul>
    </>
  );
}
```

`react-query` handles:
- Query deduplication
- Stale-while-revalidate
- Automatic abort
- Cache management

**`useDeferredValue` complement:**

```tsx
function UserSearch() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  const { data, isLoading } = useQuery({
    queryKey: ["users", deferredQuery],
    queryFn: () => fetchUsers(deferredQuery),
  });

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {/* Stale flag UI */}
      {query !== deferredQuery && <span>Updating...</span>}
      {/* Results */}
      <ul>{data?.map(...)}</ul>
    </>
  );
}
```

**`useTransition` doesn't fix race:**

```tsx
// Wrapping fetch in startTransition doesn't fix race
// startTransition only changes priority lane
// Network responses still arrive in any order
```

`useTransition` — UI rendering priority. Race condition — network ordering. Different problems.

**R19 actions API:**

```tsx
function UserSearch() {
  const [query, setQuery] = useState("");
  const [state, formAction, isPending] = useActionState(
    async (_, formData: FormData) => {
      const q = formData.get("query") as string;
      return fetchUsers(q);
    },
    [],
  );

  // Actions auto-cancel previous (R19 server actions)
}
```

R19 Server Actions — built-in cancellation semantics.

</details>

### Edge Cases

- **Network error mid-flight**: Cleanup aborts. Error not thrown to user.
- **Same query repeated**: `useEffect` re-runs only if deps change. Same query — no new fetch.
- **Query empty string**: Fetch fires for empty query — may want guard: `if (!query) return;`.

### Follow-up savollar

- "`startTransition` race'ni fix qiladimi?" — Yo'q. Faqat priority. Race fix — abort/stale logic alohida.
- "Production'da qaysi yechim afzal?" — `react-query` / `swr` / `tanstack/query` — built-in. Custom code uchun: `AbortController`.
- "Server-side throttling kerakmi?" — Yes, `if-modified-since`, request deduplication backend'da yaxshi. Lekin client-side fix majburiy (network latency).

</details>

---

## QISM F: R19 Hooks va Form Actions

### 20. `useId` Hook nima va SSR'da nima uchun u kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useId` — har component instance uchun **unique, stable, SSR-safe** ID generate qiluvchi React Hook (R18+). Asosiy use case: accessibility attribute'lar (`htmlFor`, `aria-describedby`) — server va client'da bir xil ID kerak (hydration mismatch'ni oldini olish). `Math.random()` yoki `uuid()` ishlamaydi — har render yangi raqam, SSR vs client'da turli. `useId` Fiber'ning tree path'iga asoslangan deterministik ID qaytaradi (`:r0:`, `:r1:` format). React 18'da introduced — React 17'gacha bunday Hook yo'q edi (manual workaround kerak edi).

### To'liq tushuntirish

**Muammo — SSR ID generation:**

```tsx
// ❌ Math.random() — har gal yangi
function Field({ label }: { label: string }) {
  const id = Math.random().toString(36); // server: "abc", client: "xyz"
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}

// SSR: htmlFor="abc", id="abc"
// Client hydration: htmlFor="xyz", id="xyz"
// → Hydration mismatch (different attributes)
```

**Yechim — `useId`:**

```tsx
import { useId } from "react";

function Field({ label }: { label: string }) {
  const id = useId(); // ":r0:" — deterministic per Fiber position

  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}
```

`useId` Fiber tree position'idan derive qilinadi. Server va client bir xil tree → bir xil ID.

**Multiple IDs from single hook:**

```tsx
function FormField() {
  const id = useId(); // ":r0:"

  return (
    <>
      <label htmlFor={`${id}-input`}>Name</label>
      <input id={`${id}-input`} aria-describedby={`${id}-hint`} />
      <span id={`${id}-hint`}>Enter your full name</span>
    </>
  );
}
```

> **Best practice:** Single `useId` chaqirib, suffix bilan multiple ID. Har element uchun alohida `useId` chaqirish ortiqcha.

**Tree path encoding (format):**

```
:r0:        — root depth 0
:r1:        — root depth 1
:R3l5:      — nested deeper
```

Format React internal — sintaks `:` (CSS-incompatible) ataylab tanlangan — CSS selector'da ishlatmaslik uchun.

### Kod misoli

**Accessible form (`<label>` + `<input>` + `<span>` hint):**

```tsx
import { useId, useState } from "react";

interface InputFieldProps {
  label: string;
  hint?: string;
  type?: string;
  value: string;
  onChange: (v: string) => void;
}

function InputField({ label, hint, type = "text", value, onChange }: InputFieldProps) {
  const id = useId();
  const inputId = `${id}-input`;
  const hintId = `${id}-hint`;

  return (
    <div>
      <label htmlFor={inputId}>{label}</label>
      <input
        id={inputId}
        type={type}
        value={value}
        onChange={(e) => onChange(e.target.value)}
        aria-describedby={hint ? hintId : undefined}
      />
      {hint && <span id={hintId} className="hint">{hint}</span>}
    </div>
  );
}

function SignupForm() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");

  return (
    <form>
      <InputField
        label="Email"
        type="email"
        value={email}
        onChange={setEmail}
        hint="We'll never share your email"
      />
      <InputField
        label="Password"
        type="password"
        value={password}
        onChange={setPassword}
        hint="At least 8 characters"
      />
    </form>
  );
}
```

Har `InputField` instance — alohida `useId` → barcha ID'lar unique. SSR safe.

<details>
<summary><strong>Deep Dive</strong></summary>

**`identifierPrefix` — multiple React roots:**

Bir xil page'da multiple React roots (microfrontend) — ID collision xavfi. `createRoot` `identifierPrefix` opsiyasi bilan boshqariladi:

```tsx
import { createRoot } from "react-dom/client";

const rootA = createRoot(document.getElementById("app-a")!, {
  identifierPrefix: "a-",
});

const rootB = createRoot(document.getElementById("app-b")!, {
  identifierPrefix: "b-",
});

// rootA: useId() → ":a-r0:"
// rootB: useId() → ":b-r0:"
// No collision
```

**Streaming SSR va `useId`:**

```
Streaming SSR (R18):
- HTML chunks order avval emas (Suspense fallback'lar har joyda)
- Lekin Fiber tree position deterministic
- `useId` Fiber position'ga asoslangan → order'dan mustaqil
- Client hydration ham bir xil ID generate qiladi
```

**`useId` qaerda ishlatilmasin:**

```tsx
// ❌ List key'lari uchun
function List({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => {
        const id = useId(); // ❌ Rules of Hooks violation (loop)
        return <li key={id}>{item.name}</li>;
      })}
    </ul>
  );
}

// ✅ Item.id ishlatish
{items.map((item) => <li key={item.id}>...</li>)}
```

`useId` — komponent uchun. List item key'lari — data ID (server'dan keladi).

**`useId` polyfill'i (R17 va undan oldin):**

```tsx
let counter = 0;
function manualId() {
  const [id] = useState(() => `id-${++counter}`);
  return id;
}

// Problem: SSR'da counter shared (multiple requests interleave)
// → Memory leak + race
// → useId only safe in R18+
```

**Internal mechanism (mental model):**

```typescript
// react-reconciler/src/ReactFiberHooks.js
function mountId(): string {
  const hook = mountWorkInProgressHook();
  const root = getWorkInProgressRoot();
  const identifierPrefix = root.identifierPrefix;
  let id: string;

  if (getIsHydrating()) {
    // Hydration: consume server-rendered ID
    const treeId = getTreeId();
    id = ':' + identifierPrefix + 'R' + treeId + ':';
  } else {
    // Client: generate from current Fiber position
    const globalClientId = globalClientIdCounter++;
    id = ':' + identifierPrefix + 'r' + globalClientId.toString(32) + ':';
  }

  hook.memoizedState = id;
  return id;
}
```

Hydration mode'da server ID re-used. Client-only render'da counter increment.

**Suspense + `useId`:**

```tsx
function Layout() {
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile />
    </Suspense>
  );
}

function UserProfile() {
  const id = useId();
  // ...
}
```

Suspense fallback va resolved content — har biri alohida tree pozitsiyasi. `useId` har gal bir xil bo'lib qoladi (Fiber persisted).

</details>

### Edge Cases

- **`useId` inside `useMemo`/`useCallback`**: `useMemo` ichida `useId` chaqirib bo'lmaydi (Rules of Hooks — nested hook). `useId` ni top-level chaqirib, value'ni memo'lashtirish kerak bo'lsa pass qilish.
- **CSS selectors**: `:r0:` ichidagi colon CSS'da invalid. `getElementById(":r0:")` ishlaydi (DOM ID), lekin `querySelector("#:r0:")` — escape kerak (`#\\:r0\\:`).
- **Test environments**: `useId` har test run'da bir xil ID qaytaradi (deterministic). Snapshot test'larida foydali.
- **Conditional rendering**: Komponent unmount → mount, `useId` o'zgaradi (yangi Fiber pozitsiyasi).

### Follow-up savollar

- "`useId` nima uchun React 17'da yo'q edi?" — R18 Streaming SSR uchun maxsus dizayn qilindi. R17'da serial SSR — manual workaround mumkin edi. R18 Concurrent + Streaming bilan deterministic ID majburiy bo'ldi.
- "`useId` `uuid()` o'rnini bosadimi?" — Yo'q. `useId` — DOM ID uchun (component-bound). `uuid()` — data ID (database, network). Har biri o'z domain'ida.
- "Bir komponent uchun necha `useId` chaqirsa bo'ladi?" — Cheklov yo'q, lekin best practice — bitta `useId` + suffix.

</details>

---

### 21. R19 Form Actions — `startTransition(async)` integratsiyasi, `useFormStatus`, `useActionState` [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19 **Form Actions** — `<form action={fn}>` syntax bilan declarative form handling. Action function async bo'lishi mumkin (`startTransition(async)` ostida ishlaydi). 3 ta R19 yangi hook integratsiyasi: (1) **`useActionState`** — action state (pending/error/value) ni reduce qiladi, action wrap'lab pending status beradi. (2) **`useFormStatus`** — `<form>` ichidagi child component'da form pending status'ini o'qiydi (no prop drilling). (3) **`useOptimistic`** — optimistic UI update, action yakunlanguncha. R19 `startTransition` async funksiyani qabul qiladi (R18'da faqat sync) — bu Form Actions'ning asosi. Server Actions (`'use server'`) ham shu pattern — fn server'da ishlaydi, client transition'da kutadi.

### To'liq tushuntirish

**R18 vs R19 — `startTransition` async support:**

```tsx
// R18 — async TAQIQ
startTransition(() => {
  setLoading(true);
  await fetch(...); // ❌ "Functions passed to startTransition must not return promises"
});

// R19 — async OK
startTransition(async () => {
  setLoading(true);
  await fetch(...);
  setLoading(false);
});
```

R19'da `startTransition` async fn'ni qabul qiladi va internal pending lifecycle'ni boshqaradi.

**`<form action={fn}>` — declarative form:**

```tsx
async function createPost(formData: FormData) {
  const title = formData.get("title") as string;
  await fetch("/api/posts", { method: "POST", body: JSON.stringify({ title }) });
}

function NewPostForm() {
  return (
    <form action={createPost}>
      <input name="title" />
      <button type="submit">Create</button>
    </form>
  );
}
```

- `action` prop function qabul qiladi
- Form submit → React `startTransition(async () => action(formData))` chaqiradi
- Avtomat: `FormData` berish, transition'ga wrap qilish
- Browser default action (page reload) o'chiriladi

**`useActionState` — state management:**

```tsx
import { useActionState } from "react";

interface State {
  error: string | null;
  success: boolean;
}

async function createPost(prevState: State, formData: FormData): Promise<State> {
  const title = formData.get("title") as string;

  if (!title.trim()) {
    return { error: "Title required", success: false };
  }

  try {
    await fetch("/api/posts", { method: "POST", body: JSON.stringify({ title }) });
    return { error: null, success: true };
  } catch (e) {
    return { error: (e as Error).message, success: false };
  }
}

function NewPostForm() {
  const [state, formAction, isPending] = useActionState(
    createPost,
    { error: null, success: false } // initial state
  );

  return (
    <form action={formAction}>
      <input name="title" disabled={isPending} />
      <button type="submit" disabled={isPending}>
        {isPending ? "Creating..." : "Create"}
      </button>
      {state.error && <p className="error">{state.error}</p>}
      {state.success && <p className="success">Created!</p>}
    </form>
  );
}
```

`useActionState(action, initialState)`:
- Returns `[state, wrappedAction, isPending]`
- `wrappedAction` — form'ga uzatiladi
- Submit'da `action(prevState, formData)` chaqiriladi
- Return value `state` ga keladi
- `isPending` — action async resolution kutmoqda

**`useFormStatus` — child form info:**

```tsx
import { useFormStatus } from "react-dom";

function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? "Sending..." : "Submit"}
    </button>
  );
}

function ContactForm() {
  return (
    <form action={sendMessage}>
      <input name="email" />
      <textarea name="message" />
      <SubmitButton /> {/* Auto-detects parent form state */}
    </form>
  );
}
```

- `useFormStatus` — `react-dom`'dan import (not `react`)
- `<form>` ichidagi descendant component'da chaqiriladi
- `pending` — form submission in progress
- Prop drilling kerak emas (form context)

> **Cheklov:** `useFormStatus` faqat `<form>` ichida ishlaydi (parent ancestor form). Form'siz — pending always false.

### Kod misoli

**To'liq integratsiya — `useActionState` + `useFormStatus` + `useOptimistic`:**

```tsx
import { useActionState, useOptimistic } from "react";
import { useFormStatus } from "react-dom";

interface Comment {
  id: string;
  text: string;
  sending?: boolean;
}

async function addComment(
  prevComments: Comment[],
  formData: FormData,
): Promise<Comment[]> {
  const text = formData.get("text") as string;

  // Network call
  const newComment = await fetch("/api/comments", {
    method: "POST",
    body: JSON.stringify({ text }),
  }).then((r) => r.json());

  return [...prevComments, newComment];
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? "Adding..." : "Add Comment"}
    </button>
  );
}

function CommentSection({ initial }: { initial: Comment[] }) {
  const [comments, formAction] = useActionState(addComment, initial);

  // Optimistic — UI tezroq update bo'ladi
  const [optimisticComments, addOptimistic] = useOptimistic(
    comments,
    (state: Comment[], newText: string) => [
      ...state,
      { id: `temp-${Date.now()}`, text: newText, sending: true },
    ],
  );

  async function handleSubmit(formData: FormData) {
    const text = formData.get("text") as string;
    addOptimistic(text); // Optimistic immediate
    // formAction will fire from form action prop
  }

  return (
    <div>
      <ul>
        {optimisticComments.map((c) => (
          <li key={c.id} style={{ opacity: c.sending ? 0.5 : 1 }}>
            {c.text} {c.sending && "(sending...)"}
          </li>
        ))}
      </ul>

      <form action={formAction}>
        <input name="text" required />
        <SubmitButton />
      </form>
    </div>
  );
}
```

Workflow:
1. User types text → submit
2. `useOptimistic` darhol `addOptimistic(text)` — UI new comment ko'rsatadi (sending)
3. `formAction` chaqiriladi — `addComment` async ishlatadi
4. Server response keladi — `useActionState` `comments` yangilaydi
5. `useOptimistic` real state'ga align bo'ladi (sending flag yo'qoladi)

<details>
<summary><strong>Deep Dive</strong></summary>

**`startTransition(async)` internal lifecycle:**

```typescript
// React internal (mental model)
function startTransition(fn: () => void | Promise<void>) {
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = { /* transition info */ };

  try {
    const result = fn();

    if (result && typeof result.then === "function") {
      // Async — wait for resolution
      result.then(
        () => { /* commit final state */ },
        (err) => { /* error handling */ }
      );
    }
  } finally {
    ReactCurrentBatchConfig.transition = prevTransition;
  }
}
```

R18'da async `result` `then` available bo'lsa ham — pending state'ni boshqarish yo'q edi (warning).

R19'da async fn aniq qo'llab-quvvatlanadi: pending state aniq boshqariladi, error handling integrated, transitions lifecycle full async.

**Form Actions browser semantics:**

```html
<!-- HTML form: action="URL" — page reload -->
<form action="/api/submit" method="POST">

<!-- R19: action={fn} — JS handler, no reload -->
<form action={async (fd) => {...}}>
```

R19 React `action` prop:
- Type `string` (URL) → standart HTML
- Type `function` → React handler, browser default suppressed
- Auto `preventDefault()`, auto `FormData` extraction

**Server Actions (`'use server'`):**

```tsx
// app/actions.ts (Next.js App Router)
'use server';

export async function createPost(formData: FormData) {
  // Server'da ishlaydi
  await db.posts.create({ title: formData.get("title") });
}

// app/page.tsx (client)
import { createPost } from './actions';

function NewPostForm() {
  return (
    <form action={createPost}>
      <input name="title" />
      <button>Create</button>
    </form>
  );
}
```

Server Action:
- Funksiya bundle'da emas (server-side)
- Client'dan chaqirilganda — RPC call (React internal)
- `formData` HTTP body sifatida yuboriladi
- Response → client state update

**`useActionState` vs `useReducer`:**

| Aspect | `useActionState` | `useReducer` |
|--------|------------------|--------------|
| Async support | ✅ Built-in (async action) | ❌ Sync only |
| Pending state | ✅ `isPending` | ❌ Manual |
| Form integration | ✅ Returns `formAction` for `<form action={...}>` | ❌ Manual |
| State signature | `(prevState, formData) => nextState` | `(state, action) => state` |

`useActionState` — `useReducer` for forms with async semantics.

**`useFormStatus` properties:**

```typescript
type FormStatus = {
  pending: boolean;
  data: FormData | null;     // Current submission's form data
  method: 'get' | 'post' | null;
  action: ((formData: FormData) => void | Promise<void>) | string | null;
};
```

`data` — submission davomida form'dagi qiymatlar (optimistic preview uchun).

**Error handling — Error Boundary:**

```tsx
function FormErrorBoundary({ children }: { children: ReactNode }) {
  return (
    <ErrorBoundary fallback={<p>Form crashed</p>}>
      {children}
    </ErrorBoundary>
  );
}

// Async action error → action result returned to useActionState
// Action function thrown error → propagates to ErrorBoundary
```

**`useOptimistic` cleanup:**

`useOptimistic` action complete bo'lganda real state'ga align bo'ladi avtomat — manual reset kerak emas.

```tsx
const [optimistic, setOptimistic] = useOptimistic(actualValue, reducer);

// Action davomida: optimistic = reducer(actualValue, newInput)
// Action complete: optimistic = actualValue (synced)
```

**Migration from R18 patterns:**

```tsx
// R18 pattern
function Form() {
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    setSubmitting(true);
    try {
      const formData = new FormData(e.currentTarget as HTMLFormElement);
      await submit(formData);
    } catch (err) {
      setError((err as Error).message);
    } finally {
      setSubmitting(false);
    }
  }

  return <form onSubmit={handleSubmit}>...</form>;
}

// R19 pattern — declarative
function Form() {
  const [state, formAction, isPending] = useActionState(
    async (_, formData) => {
      try {
        await submit(formData);
        return { error: null };
      } catch (err) {
        return { error: (err as Error).message };
      }
    },
    { error: null }
  );

  return <form action={formAction}>...</form>;
}
```

</details>

### Edge Cases

- **Form action async error**: Action throws → propagates to Error Boundary. Action returns `{error}` — handle as data.
- **Multiple submits**: React `useActionState` queues — pending action complete bo'lguncha new submit qabul qilinmaydi (yoki batched).
- **Server Action — JS disabled**: Browser HTML form fallback (action URL'ga POST). Progressive enhancement.
- **`useOptimistic` race**: Action davomida new optimistic — barchasi sequential apply. Action complete bo'lganda real state alignement.
- **Form reset**: `action` complete bo'lsa, React form'ni reset qiladi (default). `onReset` o'zgartirish mumkin.

### Follow-up savollar

- "Server Actions client'da ham ishlaydimi?" — `'use server'` directive — funksiya server'da bajariladi. Client'dan import RPC orqali chaqiradi. Pure client action — directive'siz.
- "`useFormStatus` `react`'dan import qilinadimi?" — Yo'q, `react-dom` (DOM-specific). React Native'da yo'q.
- "Optimistic update'ni qanday rollback qilamiz?" — `useOptimistic` avtomat rollback qiladi (action error bo'lsa, original state qaytadi). Manual control kerak bo'lsa — separate state.
- "Form action'da redirect qanday qilinadi?" — Next.js Server Actions'da `redirect()` chaqiriladi. Pure React'da — action'dan keyin `useRouter`/`navigate` manual.

</details>

---

### 22. Suspense fallback timing — flicker prevention [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Suspense fallback'lar **immediate** ko'rinadi (default) — fast network'da "flicker" (loading appears for ~50ms). Yechim: **`useTransition`** wrap (transition fallback'ni delay qiladi), **min display time** custom hook (fallback minimum 200ms ko'rinadi), yoki **stale content** (eski content + isPending indicator).

### Kod misoli

```tsx
// ❌ Direct Suspense — flicker
function Tabs({ active }: { active: string }) {
  return (
    <Suspense fallback={<Spinner />}>
      <TabContent name={active} />
    </Suspense>
  );
}
// Click tab → Spinner flash (50-100ms) → content
// Annoying

// ✅ Transition wrap — keep old content during loading
function TabsBetter() {
  const [active, setActive] = useState("a");
  const [isPending, startTransition] = useTransition();

  const handleTabClick = (name: string) => {
    startTransition(() => setActive(name));
  };

  return (
    <>
      <nav>
        <button onClick={() => handleTabClick("a")}>A</button>
        <button onClick={() => handleTabClick("b")}>B</button>
      </nav>
      <div style={{ opacity: isPending ? 0.5 : 1 }}>
        <Suspense fallback={<Spinner />}>
          <TabContent name={active} />
        </Suspense>
      </div>
    </>
  );
}
// Smooth: old content stays, fades during transition
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Min display time pattern:**

```tsx
function useMinDisplayTime(loading: boolean, minMs: number = 200) {
  const [showLoading, setShowLoading] = useState(loading);
  const startTimeRef = useRef(0);

  useEffect(() => {
    if (loading) {
      startTimeRef.current = Date.now();
      setShowLoading(true);
    } else {
      const elapsed = Date.now() - startTimeRef.current;
      const remaining = Math.max(0, minMs - elapsed);
      const timer = setTimeout(() => setShowLoading(false), remaining);
      return () => clearTimeout(timer);
    }
  }, [loading, minMs]);

  return showLoading;
}
```

**Skeleton vs Spinner:**

```tsx
// Skeleton — better perceived performance
<Suspense fallback={<SkeletonCard />}>
  <ProductCard />
</Suspense>
// Skeleton matches final layout — minimal layout shift
```

**Stale-while-revalidate pattern:**

```tsx
function useStaleWhileRevalidate<T>(promise: Promise<T>, staleValue?: T) {
  const [value, setValue] = useState<T | undefined>(staleValue);

  useEffect(() => {
    let cancelled = false;
    promise.then((newValue) => {
      if (!cancelled) setValue(newValue);
    });
    return () => { cancelled = true };
  }, [promise]);

  return value;
}
```

**TanStack Query placeholderData:**

```tsx
const { data } = useQuery({
  queryKey: ["product", id],
  queryFn: fetchProduct,
  placeholderData: (prev) => prev,  // keep old while loading new
});
```

</details>

### Edge Cases

- **Fast resolution**: Suspense fallback may flash briefly. Use min display time.
- **Nested Suspense**: Inner suspends inside outer — fallbacks nested.
- **Suspense + Error Boundary**: Error suspends — Error Boundary catches before Suspense.

### Follow-up savollar

- "Why doesn't React auto-delay fallback?" — Implementation complexity. Library territory.
- "TanStack Query handles this?" — Yes — `keepPreviousData`, `placeholderData` patterns.

</details>

---

## Xulosa

Bu fayl Concurrent React va Suspense'ning to'liq spektrini qamrab oldi:

- **QISM A — Sync vs Concurrent** (4 savol): Sync limitations, Concurrent solutions, render purity, anti-patterns
- **QISM B — startTransition + Tearing** (3 savol): `startTransition` mexanizmi, tearing va `useSyncExternalStore`, `useDeferredValue`
- **QISM C — Suspense** (5 savol): Code splitting, R19 `use()`, nested boundaries, SuspenseList, ErrorBoundary integration
- **QISM D — Streaming SSR** (3 savol): Streaming concept, selective hydration, progressive hydration patterns
- **QISM E — Output & Bug Fix** (3 savol): `useTransition` behavior trace, Suspense timing, race condition + transition fix
- **QISM F — R19 Hooks va Form Actions** (2 savol): `useId` (SSR-safe ID), Form Actions integration (`startTransition(async)`, `useActionState`, `useFormStatus`, `useOptimistic`)

**Asosiy mental model'lar:**

1. **Lanes priority** — har update'ga bitmap (Sync, Transition, Idle)
2. **Render purity** — Concurrent invariants (abort + restart safe)
3. **Tearing** — external state consistency (`useSyncExternalStore`)
4. **Suspense** — Promise unwrap, granular boundaries
5. **Streaming + Hydration** — progressive UI delivery
6. **Race conditions** — abort, stale flag, library

**Keyingi fayl:** `07-coding-challenges.md` — 20 ta majburiy implement (custom hooks, HOCs, patterns).

