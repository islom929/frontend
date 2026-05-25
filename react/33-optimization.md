# Bo'lim 33: Optimization — Real Qo'llash Patternlari

> Bu fayl **optimization patterns** ni real qo'llash perspektivasidan o'rganadi: qachon `React.memo`/`useMemo`/`useCallback` ishlatish, qachon **ishlatmaslik**, key-based reset trick, state splitting strategy, Context optimization (split state vs dispatch, selector pattern), virtualization preview, va React Compiler era manual qoladigan qismlar. Mexanika (`useMemo`/`useCallback` ichki ishlash, areHookInputsEqual algorithm) `21-usememo-usecallback.md`'da chuqur o'rganilgan; re-render trigger'lar va bypass scenarios `32-rendering-behavior.md`'da. Bu fayl shu fundament ustiga **production-grade optimization decisions** qo'shadi. Asosiy tamoyil: **profile birinchi, optimize keyin** (Donald Knuth: "premature optimization is the root of all evil"). DevTools Profiler bilan o'lchamasdan optimization — code noise va potential performance regression.

---

## Mundarija

- [Premature Optimization — Profile Birinchi](#premature-optimization--profile-birinchi)
- [Measure va Identify — DevTools Workflow](#measure-va-identify--devtools-workflow)
- [`React.memo` Real Patterns](#reactmemo-real-patterns)
- [`React.memo` Custom Comparator](#reactmemo-custom-comparator)
- [`useMemo` Real Patterns](#usememo-real-patterns)
- [`useCallback` Real Patterns](#usecallback-real-patterns)
- [Key-Based Reset Trick](#key-based-reset-trick)
- [Splitting State Across Components](#splitting-state-across-components)
- [Lift State Down — Re-render Scope Cheklash](#lift-state-down--re-render-scope-cheklash)
- [Context Optimization — Split, Memo, Selector](#context-optimization--split-memo-selector)
- [List Virtualization Preview](#list-virtualization-preview)
- [Optimization Anti-Patterns](#optimization-anti-patterns)
- [React Compiler Era — Manual Qoladigan Qismlar](#react-compiler-era--manual-qoladigan-qismlar)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Premature Optimization — Profile Birinchi

### Nazariya

Donald Knuth'ning klassik tezisi (Computing Surveys, 1974): "Premature optimization is the root of all evil." Bu React'ga ham to'liq tegishli — har joyda `useMemo`/`useCallback`/`React.memo` qo'shish:

1. **Code noise** — logic'dan ko'proq memoization wrapper'lar (memoization-heavy komponentda boilerplate logic kodidan ko'p hajmni egallashi mumkin).
2. **Performance regression** — memoization ham overhead'iga ega (cache slot allocation, deps comparison). Cheap component'da benefit < cost.
3. **Bug surface** — `useCallback` deps stale bo'lishi, `useMemo` cache "garantiya emas" (React aniq sabablar bilan cache'ni tashlashi mumkin).
4. **Maintenance burden** — har komponent o'zgarishida deps array'ni qo'lda yangilash.

React docs (`react.dev/learn/you-might-not-need-an-effect`) explicitly: **"You don't need to optimize re-renders. React is fast by default."**

NIMA UCHUN profile birinchi:

- **Hot path identifikatsiyasi** — barcha komponent'larning 5%'i performance'ning 95%'ini tashkil qiladi (Pareto principle). Optimization shu 5%'ga.
- **Real-world impact** — sintetik benchmark va real user metric'lar farq qiladi. Network latency, low-end device, complex DOM — har biri profile'ga ta'sir qiladi.
- **Confirmation bias yo'q** — "Bu komponent slow bo'lishi kerak" intuition ko'pincha noto'g'ri. Profile aniq raqam beradi.

Optimization decision tree:

```
Performance issue bor?
   │
   ├─ Yo'q → Optimization KERAK EMAS (default behavior yetarli)
   │
   └─ Ha → Profile (DevTools Profiler)
              │
              ├─ Bottleneck topildi? → Apply targeted optimization
              │                           │
              │                           ▼
              │                       Re-profile, confirm improvement
              │
              └─ Topilmadi → Optimization shart emas, boshqa muammoni qidirish
                              (Network, DB query, third-party script)
```

Profile birinchi qo'llaniladigan vaziyatlar:

1. **Initial mount slow** — Network/loading optimization (code splitting, lazy load).
2. **Interaction laggy** — re-render optimization (memo, virtualization).
3. **Animation drop frame** — Concurrent rendering, useTransition (cross-ref `22-concurrent-hooks.md`).
4. **Memory leak** — useEffect cleanup, ref management.
5. **Bundle size** — code splitting (cross-ref `35-code-splitting.md`).

Har biri uchun **alohida tool**'lar:

- React DevTools Profiler — re-render frequency va duration.
- Chrome DevTools Performance — JavaScript execution flame chart, Network, paint timing.
- Lighthouse / Core Web Vitals — production user metrics: **LCP** (Largest Contentful Paint), **INP** (Interaction to Next Paint — March 2024'da FID o'rniga stable), **CLS** (Cumulative Layout Shift).
- Bundle analyzer — kod hajmi.

> **Eslatma:** React Compiler era'da manual memoization aksariyat ortiqcha (cross-ref `31-react-compiler.md`). Compiler `'use memo'` directive bilan komponent darajasidagi va per-property granular memoization qo'shadi. Profile zarur bo'lgan optimization scenarios kelajakda kamayadi.

<details>
<summary><strong>Under the Hood</strong></summary>

`useMemo` cache "garantiya emas" — React docs (`react.dev/reference/react/useMemo` "Caveats" bo'limi):

> "React will not throw away the cached value unless there is a specific reason to do that. For example, in development, React throws away the cache when you edit the file of your component. Both in development and in production, React will throw away the cache if your component suspends during the initial mount."

Bu degani `useMemo` faqat **performance hint** — semantik garantiya emas. Komponent quyidagi holatlarda qayta hisoblaydi:

1. Deps o'zgardi (asosiy holat).
2. Strict Mode 2x render (development — purity check).
3. Component re-mount (key change, conditional render, unmount/mount).
4. Development'da source file edit (HMR/Fast Refresh).
5. Initial mount paytida component suspend bo'ldi (production'da ham).

Bu sababdan `useMemo` ni **state replacement** sifatida ishlatish anti-pattern:

```javascript
// ❌ Anti-pattern
const id = useMemo(() => crypto.randomUUID(), []); // Garantiya emas — cache evict bo'lsa yangi id

// ✅ State sifatida
const [id] = useState(() => crypto.randomUUID());
```

`useState` lazy initial — bir marta mount'da garantiyali. `useMemo` esa cache hint.

Optimization cost-benefit hisoblash:

```
Optimization cost = code complexity + bundle size + runtime overhead
Optimization benefit = render time saved × render frequency

Decision: benefit > cost → optimize
```

Misol: Komponent har 16ms render'da 0.1ms saving — 1000 render'da 100ms. Lekin kod 20 qator memoization wrapper'i qo'shsa va 5 ta dasturchi har refactor'da deps array'ni yangilashi kerak bo'lsa — cost katta.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Optimization decision flow:

```tsx
// === Stage 1: Naive implementation ===
import { useState } from 'react';
import type { ReactElement } from 'react';

interface Order {
  id: string;
  total: number;
  items: { id: string; price: number }[];
}

function OrderSummary({ orders }: { orders: Order[] }): ReactElement {
  const totalRevenue = orders.reduce((sum, o) => sum + o.total, 0);
  const totalItems = orders.reduce((sum, o) => sum + o.items.length, 0);
  
  return (
    <div>
      <p>Revenue: ${totalRevenue}</p>
      <p>Items: {totalItems}</p>
      <ul>
        {orders.map((o) => (
          <li key={o.id}>Order {o.id}: ${o.total}</li>
        ))}
      </ul>
    </div>
  );
}

function App(): ReactElement {
  const [orders] = useState<Order[]>([/* ... 100 orders ... */]);
  return <OrderSummary orders={orders} />;
}
// Initial mount: ~1ms (100 orders) — cheap
// No re-renders unless orders changes
// Premature optimization (useMemo har joyda) — code noise, no measurable benefit
```

```tsx
// === Stage 2: Profile sodir bo'lgan muammo ===
// User reports: "Adding new order causes 1-second freeze"
// Profile: OrderSummary re-render takes 800ms
// Root cause: orders array yangi push'da har order uchun complex computation
// Optimization needed!

import { useState, useMemo } from 'react';

function OrderSummary({ orders }: { orders: Order[] }): ReactElement {
  // Targeted optimization: heavy reduce'larni cache
  const totalRevenue = useMemo(
    () => orders.reduce((sum, o) => sum + o.total, 0),
    [orders]
  );
  
  const totalItems = useMemo(
    () => orders.reduce((sum, o) => sum + o.items.length, 0),
    [orders]
  );
  
  return (/* ... */);
}
// Profile after: re-render 800ms → 50ms (16x improvement)
// useMemo justified by measurement
```

Compiler era — manual optimization ortiqcha:

```tsx
'use memo';

import { useState } from 'react';
import type { ReactElement } from 'react';

interface Order {
  id: string;
  total: number;
  items: { id: string; price: number }[];
}

function OrderSummary({ orders }: { orders: Order[] }): ReactElement {
  // No useMemo — Compiler avtomatik cache qiladi
  const totalRevenue = orders.reduce((sum, o) => sum + o.total, 0);
  const totalItems = orders.reduce((sum, o) => sum + o.items.length, 0);
  
  return (/* ... */);
}
// Compiler: totalRevenue va totalItems cache (deps: orders)
// orders o'zgarmasa — cached, 800ms freeze yo'q.
```

</details>

---

## Measure va Identify — DevTools Workflow

### Nazariya

Optimization workflow 5 qadam:

### Qadam 1: Symptom Identification

User report yoki monitoring:
- "Search box typing laggy"
- "Modal open animation slow"
- "List scroll janky"
- p95 commit duration > 50ms (production telemetry)

### Qadam 2: Reproduce

Symptom-ga olib keladigan stepslarni aniqlash:
1. Open `/dashboard`
2. Type "test" in search box
3. Observe laggy keystrokes

### Qadam 3: Profile Record

DevTools Profiler:
1. Open React DevTools → Profiler tab
2. Settings → "Record why each component rendered while profiling" enable
3. Click record (circle button)
4. Reproduce steps
5. Stop record

Output: Flame chart har commit uchun.

### Qadam 4: Analyze

Flame chart'da:
- **Long bars** — slow commits.
- **Many commits** — re-render frequency yuqori.
- **Wide bars** — komponent ko'p vaqt oladi.

"Why did this render?" panel — re-render manbasi:
- "Hooks changed" — state update (kutilgan).
- "Props changed" — parent yangi prop berdi (potentially fixable with memo).
- "Parent component rendered" — top-down propagation (memo bilan to'xtatish mumkin).
- "Context changed" — Context value o'zgardi (split with selector).

### Qadam 5: Apply Optimization

Targeted — faqat bottleneck'ga:

| Reason | Optimization |
|--------|-------------|
| Props changed (object/array literal) | `useMemo` parent'da |
| Props changed (function) | `useCallback` parent'da |
| Parent rendered, props bir xil | `React.memo` |
| Context changed har gal | Context split / selector |
| Heavy computation har render | `useMemo` cache |
| List 1000+ items | Virtualization (cross-ref [`36-virtualization.md`](36-virtualization.md)) |

### Qadam 6: Re-profile va Confirm

Bir xil scenario'ni profile — improvement aniqlash:

```
Before: actualDuration ~ 50ms
After:  actualDuration ~ 5ms (sezilarli improvement) ✅
```

Agar improvement yo'q yoki regression — optimization olib tashlanadi.

### Qadam 7: Production Monitoring

`<Profiler>` callback bilan production telemetry:

```tsx
import { Profiler } from 'react';

const onRender = (id, phase, actualDuration) => {
  if (actualDuration > 50) {
    monitoring.recordSlowRender({ id, actualDuration });
  }
};

<Profiler id="HotPath" onRender={onRender}>
  <HotPath />
</Profiler>
```

p95 commit duration trend monitoring — regression alert.

NIMA UCHUN bu workflow zarur:

- **Confirmation bias yo'q** — intuition'ga emas, raqamlarga asoslanadi.
- **Right tool** — re-render performance React DevTools, JavaScript hot loop Chrome Performance.
- **Regression detection** — har optimization re-measure bilan tasdiqlanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

Profiler API metric:

```typescript
type ProfilerOnRenderCallback = (
  id: string,
  phase: 'mount' | 'update' | 'nested-update',
  actualDuration: number,
  baseDuration: number,
  startTime: number,
  commitTime: number
) => void;
```

- **`actualDuration`** — bu commit'da real ish vaqti (bailout'lar tashqarida).
- **`baseDuration`** — agar barcha komponent'lar re-render bo'lsa, taxminiy vaqt.
- **`startTime`** — render boshlangan vaqt (`performance.now()`).
- **`commitTime`** — commit tugagan vaqt.

Memoization efficiency formula:

```
Efficiency = (baseDuration - actualDuration) / baseDuration

100% — barcha bailout, hech qanday render
50% — yarmi bailout
0% — hech qanday memoization
```

Production'da p95 metric:

```javascript
// 100 ta render durations: [1, 2, 3, ..., 100]
// p95 = sorted[Math.floor(durations.length * 0.95)]
// p95 = 95
// Bu degani 95% renders < 95ms
```

p95 commit duration target: < 16ms (1 frame at 60fps). Agar p95 > 50ms — visible jank.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Production profile setup:

```tsx
import { Profiler } from 'react';
import type { ReactElement, ProfilerOnRenderCallback } from 'react';

interface Metric {
  id: string;
  phase: string;
  actualDuration: number;
  baseDuration: number;
  timestamp: number;
}

const buffer: Metric[] = [];
const FLUSH_INTERVAL_MS = 10000;

const onRender: ProfilerOnRenderCallback = (
  id,
  phase,
  actualDuration,
  baseDuration
) => {
  buffer.push({
    id,
    phase,
    actualDuration,
    baseDuration,
    timestamp: Date.now(),
  });
  
  // Slow render warning
  if (actualDuration > 16) {
    console.warn(`[Slow render] ${id}: ${actualDuration.toFixed(2)}ms`);
  }
};

// Periodic flush
setInterval(() => {
  if (buffer.length === 0) return;
  
  const batch = buffer.splice(0, buffer.length);
  navigator.sendBeacon(
    '/api/monitoring/react',
    JSON.stringify(batch)
  );
}, FLUSH_INTERVAL_MS);

function App(): ReactElement {
  return (
    <Profiler id="App" onRender={onRender}>
      <Profiler id="Header" onRender={onRender}>
        <Header />
      </Profiler>
      <Profiler id="MainContent" onRender={onRender}>
        <MainContent />
      </Profiler>
    </Profiler>
  );
}
```

Workflow misol — search optimization:

```tsx
// === Step 1: Initial implementation ===
import { useState } from 'react';
import type { ReactElement } from 'react';

interface Product {
  id: string;
  name: string;
  category: string;
}

function SearchableList({ products }: { products: Product[] }): ReactElement {
  const [query, setQuery] = useState('');
  
  const filtered = products.filter((p) =>
    p.name.toLowerCase().includes(query.toLowerCase())
  );
  
  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ul>
        {filtered.map((p) => (
          <li key={p.id}>{p.name} — {p.category}</li>
        ))}
      </ul>
    </div>
  );
}

// === Step 2: Profile says ===
// 1000+ products, typing causes 100ms+ commits
// "Why did this render?" — Hooks changed (query)
// Bottleneck: filter + map per keystroke

// === Step 3: Optimization ===
import { useMemo, useTransition } from 'react';

function SearchableList({ products }: { products: Product[] }): ReactElement {
  const [query, setQuery] = useState('');
  const [deferredQuery, setDeferredQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  
  function handleChange(value: string) {
    setQuery(value); // Sync — input responsive
    startTransition(() => {
      setDeferredQuery(value); // Non-urgent
    });
  }
  
  const filtered = useMemo(
    () => products.filter((p) =>
      p.name.toLowerCase().includes(deferredQuery.toLowerCase())
    ),
    [products, deferredQuery]
  );
  
  return (
    <div>
      <input value={query} onChange={(e) => handleChange(e.target.value)} />
      {isPending && <span>Filtering...</span>}
      <ul style={{ opacity: isPending ? 0.5 : 1 }}>
        {filtered.map((p) => (
          <li key={p.id}>{p.name} — {p.category}</li>
        ))}
      </ul>
    </div>
  );
}

// === Step 4: Re-profile ===
// Input responsive (sync), filter Transition Lane interruptible
// p95 commit duration ~ 10ms (sezilarli improvement)

// === Step 5: Production monitoring ===
import { Profiler } from 'react';

<Profiler id="SearchableList" onRender={onRender}>
  <SearchableList products={products} />
</Profiler>
```

</details>

---

## `React.memo` Real Patterns

### Nazariya

`React.memo` — komponent'ni shallow props comparison bilan memoize qiluvchi HOC. Mexanika `21-usememo-usecallback.md` da, bypass scenarios `32-rendering-behavior.md` da.

**Real qo'llanish patterns:**

### Pattern 1: List Item Component

Eng tez-tez foydali use case — list item:

```tsx
import { memo } from 'react';
import type { ReactElement } from 'react';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

const TodoItem = memo(function TodoItem({
  todo,
  onToggle,
  onDelete,
}: {
  todo: Todo;
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
}): ReactElement {
  return (
    <li>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() => onToggle(todo.id)}
      />
      <span>{todo.text}</span>
      <button onClick={() => onDelete(todo.id)}>Delete</button>
    </li>
  );
});

function TodoList({ todos, onToggle, onDelete }: TodoListProps): ReactElement {
  return (
    <ul>
      {todos.map((t) => (
        <TodoItem
          key={t.id}
          todo={t}
          onToggle={onToggle}
          onDelete={onDelete}
        />
      ))}
    </ul>
  );
}
```

Lekin `onToggle`/`onDelete` parent'da inline function bo'lsa — har render yangi reference, memo bypass. Yechim — `useCallback` parent'da:

```tsx
function TodoApp(): ReactElement {
  const [todos, setTodos] = useState<Todo[]>([]);
  
  const handleToggle = useCallback((id: string) => {
    setTodos((prev) =>
      prev.map((t) => (t.id === id ? { ...t, completed: !t.completed } : t))
    );
  }, []); // Functional update — no deps needed
  
  const handleDelete = useCallback((id: string) => {
    setTodos((prev) => prev.filter((t) => t.id !== id));
  }, []);
  
  return <TodoList todos={todos} onToggle={handleToggle} onDelete={handleDelete} />;
}
```

### Pattern 2: Expensive Presentational

Heavy render (chart, complex layout, animation):

```tsx
const PerformanceChart = memo(function PerformanceChart({
  data,
}: {
  data: ChartData[];
}): ReactElement {
  // 100+ SVG elements computation
  return <svg>...</svg>;
});

// Parent state o'zgarishi PerformanceChart'ga ta'sir qilmasa — memo bailout
```

### Pattern 3: Tez-tez Re-render Bo'luvchi Parent

Form input parent'i (har keystroke render bo'ladi):

```tsx
function ContactForm(): ReactElement {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  
  return (
    <form>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <input value={email} onChange={(e) => setEmail(e.target.value)} />
      <SubmitButton /> {/* Static — memo bilan saqlash */}
    </form>
  );
}

const SubmitButton = memo(function SubmitButton(): ReactElement {
  return <button type="submit">Submit</button>;
});
// Har keystroke ContactForm re-render, lekin SubmitButton bailout.
```

### Pattern 4: Context Provider Children

```tsx
function CartProvider({ children }: { children: ReactNode }): ReactElement {
  const [items, setItems] = useState<CartItem[]>([]);
  
  // ... addItem, removeItem callbacks ...
  
  const value = useMemo(() => ({ items, addItem, removeItem }), [items, addItem, removeItem]);
  
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}
// CartProvider state o'zgarsa re-render, lekin children prop reference saqlanadi
// (parent yaratgan) — children re-render bo'lmaydi (Reconciler bailout via reference identity).
```

Bu yerda `React.memo` shart emas — children prop reference identity bailout (cross-ref `04-reconciliation.md`).

### Qachon `React.memo` ishlatmaslik

1. **Cheap render** — span/div bilan oddiy text:
   ```tsx
   const Cheap = memo(({ value }) => <span>{value}</span>);
   // Memo overhead > render saving
   ```

2. **Props har gal yangi** — inline literals'siz iloji yo'q:
   ```tsx
   <Memo data={getData()} /> // getData har render yangi qaytarsa — bypass har gal
   ```

3. **Doim re-render kerak** — parent state komponent props'iga to'liq ta'sir qiladi:
   ```tsx
   <Memo value={count} /> // Har count change'da re-render kerak
   // Memo overhead foydasi yo'q
   ```

4. **React Compiler enabled** — avtomatik komponent darajasidagi memo (cross-ref `31-react-compiler.md`):
   ```tsx
   'use memo';
   function Component({ data }) { /* ... */ } // Memo automatic
   ```

<details>
<summary><strong>Under the Hood</strong></summary>

`React.memo` Fiber tag — `MemoComponent`. Reconciler `updateMemoComponent` chaqiradi:

```javascript
function updateMemoComponent(current, workInProgress, Component, nextProps) {
  if (current !== null) {
    const prevProps = current.memoizedProps;
    
    let isPropsEqual;
    if (Component.compare) {
      isPropsEqual = Component.compare(prevProps, nextProps);
    } else {
      isPropsEqual = shallowEqual(prevProps, nextProps);
    }
    
    if (isPropsEqual && current.ref === workInProgress.ref) {
      return bailoutOnAlreadyFinishedWork(current, workInProgress, ...);
    }
  }
  
  // Render component
}
```

`shallowEqual` algoritm (cross-ref `32-rendering-behavior.md`):

```javascript
function shallowEqual(prev, next) {
  if (Object.is(prev, next)) return true;
  if (typeof prev !== 'object' || typeof next !== 'object') return false;
  if (prev === null || next === null) return false;
  
  const keysA = Object.keys(prev);
  const keysB = Object.keys(next);
  if (keysA.length !== keysB.length) return false;
  
  for (const key of keysA) {
    if (!Object.is(prev[key], next[key])) return false;
  }
  return true;
}
```

Memo cost (per render):

- Shallow comparison: O(N) where N = props count.
- Object comparison: O(1) (`Object.is`).
- Function comparison: O(1) (`Object.is`).

Total: 5-10 props uchun ~0.01ms (negligible). Memo overhead ahamiyatli faqat very cheap render uchun.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Pattern 1 — TodoList full implementation:

```tsx
import { memo, useState, useCallback } from 'react';
import type { ReactElement } from 'react';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

interface TodoItemProps {
  todo: Todo;
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
}

const TodoItem = memo(function TodoItem({
  todo,
  onToggle,
  onDelete,
}: TodoItemProps): ReactElement {
  console.log(`TodoItem ${todo.id} rendered`);
  return (
    <li>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() => onToggle(todo.id)}
      />
      <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
        {todo.text}
      </span>
      <button onClick={() => onDelete(todo.id)}>Delete</button>
    </li>
  );
});

function TodoApp(): ReactElement {
  const [todos, setTodos] = useState<Todo[]>([
    { id: '1', text: 'Learn React', completed: false },
    { id: '2', text: 'Build app', completed: false },
  ]);
  const [filter, setFilter] = useState<'all' | 'active' | 'completed'>('all');
  
  // Stable callbacks
  const handleToggle = useCallback((id: string) => {
    setTodos((prev) =>
      prev.map((t) => (t.id === id ? { ...t, completed: !t.completed } : t))
    );
  }, []);
  
  const handleDelete = useCallback((id: string) => {
    setTodos((prev) => prev.filter((t) => t.id !== id));
  }, []);
  
  const filtered = todos.filter((t) => {
    if (filter === 'active') return !t.completed;
    if (filter === 'completed') return t.completed;
    return true;
  });
  
  return (
    <div>
      <select value={filter} onChange={(e) => setFilter(e.target.value as typeof filter)}>
        <option value="all">All</option>
        <option value="active">Active</option>
        <option value="completed">Completed</option>
      </select>
      <ul>
        {filtered.map((t) => (
          <TodoItem
            key={t.id}
            todo={t}
            onToggle={handleToggle}
            onDelete={handleDelete}
          />
        ))}
      </ul>
    </div>
  );
}
// Toggle todo "1" → faqat TodoItem id="1" re-render (boshqalar bailout)
// Filter change → barcha TodoItem'lar bailout (props bir xil)
// New todo qo'shilsa → faqat yangi item mount, eskilar bailout
```

Pattern 4 — Static parts memo:

```tsx
import { memo, useState } from 'react';
import type { ReactElement } from 'react';

const StaticHeader = memo(function StaticHeader(): ReactElement {
  console.log('StaticHeader rendered');
  return (
    <header>
      <h1>Application</h1>
      <nav>
        <a href="/">Home</a>
        <a href="/about">About</a>
      </nav>
    </header>
  );
});

const StaticFooter = memo(function StaticFooter(): ReactElement {
  console.log('StaticFooter rendered');
  return <footer>© 2026</footer>;
});

function App(): ReactElement {
  const [count, setCount] = useState(0);
  
  return (
    <>
      <StaticHeader />
      <main>
        <button onClick={() => setCount((c) => c + 1)}>Count: {count}</button>
      </main>
      <StaticFooter />
    </>
  );
}
// Click har count change: App re-render
// StaticHeader, StaticFooter — bailout (props yo'q, bir xil)
// Real-world: nav links, branding — kamdan-kam o'zgaradi.
```

</details>

---

## `React.memo` Custom Comparator

### Nazariya

`React.memo`'ning ikkinchi argumenti — custom comparator function:

```typescript
const Component = memo(
  function Component(props) { /* ... */ },
  (prevProps, nextProps) => {
    // return true → props teng → render skip
    // return false → props farq → re-render
  }
);
```

Default — `shallowEqual`. Custom — qo'shimcha logic.

NIMA UCHUN custom kerak:

### Use Case 1: Property Subset Comparison

```tsx
interface User {
  id: string;
  name: string;
  email: string;
  lastLogin: Date; // Bu har refresh'da o'zgaradi
  metadata: Record<string, unknown>;
}

interface UserBadgeProps {
  user: User;
}

// ❌ Default shallow: lastLogin har refresh yangi reference → re-render har gal
const UserBadgeDefault = memo(function UserBadge({ user }: UserBadgeProps) {
  return <span>{user.name}</span>;
});

// ✅ Custom: faqat name muhim
const UserBadge = memo(
  function UserBadge({ user }: UserBadgeProps) {
    return <span>{user.name}</span>;
  },
  (prev, next) => prev.user.name === next.user.name
);
```

### Use Case 2: ID-Based Identity

```tsx
interface ProductCardProps {
  product: Product;
}

// User reference o'zgarsa lekin id bir xil → bailout
const ProductCard = memo(
  function ProductCard({ product }: ProductCardProps) {
    return <div>{product.name}</div>;
  },
  (prev, next) => prev.product.id === next.product.id
);
```

**XAVF:** Agar `product.id` bir xil lekin `product.name` o'zgargan bo'lsa — UI eski name ko'rsatadi (silent bug).

### Use Case 3: Deep Equality Check

```tsx
import { dequal } from 'dequal'; // Library

const ConfigPanel = memo(
  function ConfigPanel({ config }: { config: Config }) {
    return <div>{/* render config */}</div>;
  },
  (prev, next) => dequal(prev.config, next.config)
);
// Lekin: deep comparison har render — overhead.
// Faqat reference o'zgarish lekin shape bir xil scenarios uchun.
```

### Use Case 4: Negation (Custom return logic)

```tsx
interface AnimatedListProps {
  items: Item[];
  isAnimating: boolean;
}

const AnimatedList = memo(
  function AnimatedList({ items, isAnimating }: AnimatedListProps) {
    return <ul>{/* ... */}</ul>;
  },
  (prev, next) => {
    // Animation paytida re-render qilinmaydi (visual stability)
    if (next.isAnimating) return true;
    
    // Shallow check
    return (
      prev.items === next.items &&
      prev.isAnimating === next.isAnimating
    );
  }
);
```

### XAVFLAR

1. **Bug surface** — comparator buggy bo'lsa, UI stale data ko'rsatadi.
2. **Maintenance burden** — Component shapes o'zgarishi bilan comparator yangilanishi shart.
3. **Performance regression** — deep comparison O(N) bo'lsa, har render expensive.
4. **Reference equality lost** — `prev.items === next.items` shallow, lekin items[0] o'zgargan bo'lsa miss.

NIMA UCHUN ko'pincha **better solution**:

- `useMemo` parent'da — props reference stability ta'minlash.
- `useCallback` parent'da — function stability.
- State splitting — re-render scope cheklash.
- Compiler — avtomatik granular cache.

Custom comparator faqat **specific scenario**'da, shu yerda boshqa solution ishlamasa.

<details>
<summary><strong>Under the Hood</strong></summary>

Custom comparator React internal:

```javascript
function updateMemoComponent(current, workInProgress, Component, nextProps) {
  // Component.compare bor bo'lsa — custom comparator
  const prevProps = current.memoizedProps;
  
  let isPropsEqual;
  if (Component.compare !== null) {
    // Custom
    isPropsEqual = Component.compare(prevProps, nextProps);
  } else {
    // Default shallow
    isPropsEqual = shallowEqual(prevProps, nextProps);
  }
  
  if (isPropsEqual && current.ref === workInProgress.ref) {
    return bailout;
  }
  
  return render;
}
```

Custom comparator chaqiriladi har render'da (parent re-render, props o'zgarmasa ham). Comparator overhead = comparator complexity × render frequency.

Misol:

```
Default shallow: ~0.01ms per render
Custom (id check): ~0.001ms (faster!)
Custom (deep equal): ~0.5ms (50x slower)
```

Deep equal har 1000 ta render'da 500ms total — visible jank.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Real-world examples:

```tsx
import { memo } from 'react';
import type { ReactElement } from 'react';

// === Example 1: Comments thread ===
interface Comment {
  id: string;
  text: string;
  authorId: string;
  createdAt: string;
  reactions: Record<string, number>; // Tez-tez yangilanadi
}

interface CommentItemProps {
  comment: Comment;
  onReply: (id: string) => void;
}

// Reactions qisman update — author/text o'zgarmasa re-render kerak emas
const CommentItem = memo(
  function CommentItem({ comment, onReply }: CommentItemProps): ReactElement {
    return (
      <div>
        <h4>Author: {comment.authorId}</h4>
        <p>{comment.text}</p>
        <span>Reactions: {Object.values(comment.reactions).reduce((s, c) => s + c, 0)}</span>
        <button onClick={() => onReply(comment.id)}>Reply</button>
      </div>
    );
  },
  (prev, next) => {
    // Faqat hash check — comment ID + text + reactions count
    const prevReactionsCount = Object.values(prev.comment.reactions).reduce((s, c) => s + c, 0);
    const nextReactionsCount = Object.values(next.comment.reactions).reduce((s, c) => s + c, 0);
    
    return (
      prev.comment.id === next.comment.id &&
      prev.comment.text === next.comment.text &&
      prevReactionsCount === nextReactionsCount &&
      prev.onReply === next.onReply
    );
  }
);
```

```tsx
// === Example 2: Map markers ===
interface MapMarker {
  id: string;
  lat: number;
  lng: number;
  label: string;
  tooltip: string;
}

interface MarkerProps {
  marker: MapMarker;
  isSelected: boolean;
  onClick: (id: string) => void;
}

// Faqat position va selected status muhim, tooltip click'da expand
const Marker = memo(
  function Marker({ marker, isSelected, onClick }: MarkerProps): ReactElement {
    return (
      <div
        style={{
          position: 'absolute',
          left: marker.lng * 100,
          top: marker.lat * 100,
          background: isSelected ? 'red' : 'blue',
        }}
        onClick={() => onClick(marker.id)}
      >
        {marker.label}
      </div>
    );
  },
  (prev, next) => {
    return (
      prev.marker.id === next.marker.id &&
      prev.marker.lat === next.marker.lat &&
      prev.marker.lng === next.marker.lng &&
      prev.marker.label === next.marker.label &&
      prev.isSelected === next.isSelected &&
      prev.onClick === next.onClick
      // tooltip e'tibordan tashqari — re-render trigger qilmaydi
    );
  }
);
```

```tsx
// === Example 3: Animation pause ===
interface CarouselSlideProps {
  slide: { id: string; image: string; caption: string };
  isAnimating: boolean;
  index: number;
}

const CarouselSlide = memo(
  function CarouselSlide({ slide, isAnimating, index }: CarouselSlideProps): ReactElement {
    return (
      <div className={`slide ${isAnimating ? 'animating' : ''}`}>
        <img src={slide.image} alt={slide.caption} />
        <p>{slide.caption}</p>
      </div>
    );
  },
  (prev, next) => {
    // Animation paytida re-render qilinmaydi (visual smooth)
    if (next.isAnimating && !prev.isAnimating) return false; // Animation start — render
    if (prev.isAnimating && !next.isAnimating) return false; // Animation end — render
    if (next.isAnimating) return true; // Animation paytida skip
    
    // Default shallow (animation tashqarisida)
    return (
      prev.slide === next.slide &&
      prev.index === next.index
    );
  }
);
```

</details>

---

## `useMemo` Real Patterns

### Nazariya

`useMemo` — komponent ichida **referential identity stabilize** yoki **expensive computation cache** uchun. Mexanika `21-usememo-usecallback.md` da.

### Pattern 1: Stable Object/Array Props for Memoized Children

```tsx
import { useMemo, memo } from 'react';

const Chart = memo(function Chart({ config }: { config: ChartConfig }) {
  // Heavy SVG render
  return <svg>...</svg>;
});

function Dashboard(): ReactElement {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  // ✅ Stable reference — Chart memo bailout
  const chartConfig = useMemo<ChartConfig>(
    () => ({
      colors: theme === 'dark' ? ['#fff'] : ['#000'],
      animation: true,
    }),
    [theme]
  );
  
  return <Chart config={chartConfig} />;
}
```

### Pattern 2: Expensive Computation Cache

```tsx
function ReportPage({ data }: { data: ReportData[] }): ReactElement {
  // ✅ Expensive aggregation — cache
  const summary = useMemo(() => {
    return {
      total: data.reduce((s, d) => s + d.value, 0),
      avg: data.reduce((s, d) => s + d.value, 0) / data.length,
      max: Math.max(...data.map((d) => d.value)),
      min: Math.min(...data.map((d) => d.value)),
    };
  }, [data]);
  
  return <SummaryCard {...summary} />;
}
```

Lekin "expensive" — relative. Profile bilan tasdiqlanadi:

```typescript
// 100 element data — reduce ~0.05ms — useMemo cost ~0.02ms — barely worth it
// 10000 element data — reduce ~5ms — useMemo cache ~0.02ms hit — very worth it
```

### Pattern 3: Filtered/Sorted Data

```tsx
function ProductList({ products, filter, sortBy }: Props): ReactElement {
  const visibleProducts = useMemo(
    () =>
      products
        .filter((p) => p.category === filter)
        .sort((a, b) => {
          if (sortBy === 'price') return a.price - b.price;
          return a.name.localeCompare(b.name);
        }),
    [products, filter, sortBy]
  );
  
  return (
    <ul>
      {visibleProducts.map((p) => (
        <li key={p.id}>{p.name}</li>
      ))}
    </ul>
  );
}
```

### Pattern 4: Context Value

```tsx
interface CartContextValue {
  items: CartItem[];
  addItem: (item: CartItem) => void;
}

function CartProvider({ children }: { children: ReactNode }): ReactElement {
  const [items, setItems] = useState<CartItem[]>([]);
  
  const addItem = useCallback((item: CartItem) => {
    setItems((prev) => [...prev, item]);
  }, []);
  
  // ✅ Stable Context value
  const value = useMemo<CartContextValue>(
    () => ({ items, addItem }),
    [items, addItem]
  );
  
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}
// Bu Pattern muhim — Context value yangi reference bo'lsa barcha consumers re-render
// (cross-ref 19, 32 — Context propagation memo bypass)
```

### Pattern 5: Derived State from Props

```tsx
function UserProfile({ user }: { user: User }): ReactElement {
  const fullName = useMemo(
    () => `${user.firstName} ${user.lastName}`.toUpperCase(),
    [user.firstName, user.lastName]
  );
  
  // Lekin bu primitive computation — useMemo overkill
  // ✅ Better: const fullName = `${user.firstName} ${user.lastName}`.toUpperCase();
}
```

Primitive computation `useMemo` overhead'i computation'dan ko'p — anti-pattern.

### Qachon `useMemo` ishlatmaslik

1. **Primitive computation** — `count + 1`, `name.toUpperCase()`.
2. **Cheap object** — `{ x: 1, y: 2 }` — useMemo overhead > saving.
3. **Reference identity'ga ehtiyoj yo'q** — komponent memo'siz, deps dep emas.
4. **Compiler enabled** — `'use memo'` directive avtomatik cache.

<details>
<summary><strong>Under the Hood</strong></summary>

`useMemo` mexanika (cross-ref `21-usememo-usecallback.md`):

```javascript
// mountMemo
function mountMemo(factory, deps) {
  const hook = mountWorkInProgressHook();
  const value = factory();
  hook.memoizedState = [value, deps];
  return value;
}

// updateMemo
function updateMemo(factory, deps) {
  const hook = updateWorkInProgressHook();
  const prevDeps = hook.memoizedState?.[1];
  
  if (areHookInputsEqual(deps, prevDeps)) {
    return hook.memoizedState[0]; // Cache hit
  }
  
  // Cache miss — recompute
  const value = factory();
  hook.memoizedState = [value, deps];
  return value;
}
```

`areHookInputsEqual` — `Object.is` per dep.

Cost-benefit hisoblash:

```
useMemo overhead per render: ~0.01ms
   (allocation + Object.is per dep + cache lookup)

Computation saving (cache hit): variable
   - String concat: 0.001ms saving — useMemo MORE expensive
   - Reduce 1000 items: 0.5ms saving — worth it
   - Heavy SVG generation: 5ms saving — definitely worth it
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Pattern combinations:

```tsx
import { useState, useMemo, useCallback, memo } from 'react';
import type { ReactElement } from 'react';

interface Order {
  id: string;
  customerId: string;
  total: number;
  status: 'pending' | 'completed' | 'cancelled';
  items: OrderItem[];
}

interface OrderItem {
  id: string;
  productId: string;
  quantity: number;
  price: number;
}

interface OrderListProps {
  orders: Order[];
  filter: 'all' | 'pending' | 'completed' | 'cancelled';
  onSelectOrder: (id: string) => void;
}

const OrderRow = memo(function OrderRow({
  order,
  onSelect,
}: {
  order: Order;
  onSelect: (id: string) => void;
}): ReactElement {
  return (
    <tr onClick={() => onSelect(order.id)}>
      <td>{order.id}</td>
      <td>${order.total.toFixed(2)}</td>
      <td>{order.status}</td>
      <td>{order.items.length} items</td>
    </tr>
  );
});

function OrderList({ orders, filter, onSelectOrder }: OrderListProps): ReactElement {
  // Filtered list — cache
  const visibleOrders = useMemo(() => {
    if (filter === 'all') return orders;
    return orders.filter((o) => o.status === filter);
  }, [orders, filter]);
  
  // Aggregations — cache
  const stats = useMemo(() => {
    return {
      total: visibleOrders.reduce((s, o) => s + o.total, 0),
      count: visibleOrders.length,
      avg: visibleOrders.length > 0
        ? visibleOrders.reduce((s, o) => s + o.total, 0) / visibleOrders.length
        : 0,
    };
  }, [visibleOrders]);
  
  return (
    <div>
      <header>
        <p>Total: ${stats.total.toFixed(2)} ({stats.count} orders)</p>
        <p>Average: ${stats.avg.toFixed(2)}</p>
      </header>
      <table>
        <tbody>
          {visibleOrders.map((o) => (
            <OrderRow key={o.id} order={o} onSelect={onSelectOrder} />
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

</details>

---

## `useCallback` Real Patterns

### Nazariya

`useCallback` — function reference stabilize. `useMemo(() => fn, deps)` ekvivalenti (cross-ref `21-usememo-usecallback.md`).

### Pattern 1: Memoized Child onClick

```tsx
const Button = memo(function Button({ onClick, label }: ButtonProps) {
  return <button onClick={onClick}>{label}</button>;
});

function Toolbar(): ReactElement {
  const [count, setCount] = useState(0);
  
  // ❌ Inline — Button memo bypass
  // <Button onClick={() => setCount((c) => c + 1)} label="Add" />
  
  // ✅ Stable callback
  const handleAdd = useCallback(() => {
    setCount((c) => c + 1);
  }, []); // Functional update — no deps needed
  
  return <Button onClick={handleAdd} label={`Add (${count})`} />;
}
```

### Pattern 2: useEffect Dep

```tsx
import { useEffect, useCallback } from 'react';

function ChatWindow({ roomId }: { roomId: string }): ReactElement {
  const handleMessage = useCallback((event: MessageEvent) => {
    console.log('Received:', event.data);
  }, []); // Stable
  
  useEffect(() => {
    const ws = new WebSocket(`wss://chat/${roomId}`);
    ws.addEventListener('message', handleMessage);
    return () => {
      ws.removeEventListener('message', handleMessage);
      ws.close();
    };
  }, [roomId, handleMessage]); // handleMessage stable — effect har roomId change'da
}
```

### Pattern 3: Custom Hook Returns

```tsx
function useTodos(): {
  todos: Todo[];
  addTodo: (text: string) => void;
  removeTodo: (id: string) => void;
} {
  const [todos, setTodos] = useState<Todo[]>([]);
  
  const addTodo = useCallback((text: string) => {
    setTodos((prev) => [...prev, { id: crypto.randomUUID(), text, completed: false }]);
  }, []);
  
  const removeTodo = useCallback((id: string) => {
    setTodos((prev) => prev.filter((t) => t.id !== id));
  }, []);
  
  return { todos, addTodo, removeTodo };
}
// Consumer:
function App() {
  const { todos, addTodo, removeTodo } = useTodos();
  
  // addTodo, removeTodo stable — useCallback parent'da kerak emas
  return <TodoList todos={todos} onAdd={addTodo} onRemove={removeTodo} />;
}
```

### Pattern 4: Stable Reference for Optimized Children

```tsx
function ParentList({ items }: { items: Item[] }): ReactElement {
  const handleItemClick = useCallback((id: string) => {
    console.log('Clicked:', id);
  }, []);
  
  return (
    <ul>
      {items.map((item) => (
        <MemoListItem
          key={item.id}
          item={item}
          onClick={handleItemClick} // Bir xil reference uzatiladi
        />
      ))}
    </ul>
  );
}
```

### Qachon `useCallback` ishlatmaslik

1. **Memo'siz consumer** — useCallback befoyda agar receiver `React.memo`'siz:
   ```tsx
   <PlainChild onClick={handleClick} /> // useCallback overhead, foyda yo'q
   ```

2. **Inline OK joylar** — har render kichik function literal:
   ```tsx
   <button onClick={() => alert('hi')}>Click</button> // Inline OK, memo'siz consumer
   ```

3. **Compiler enabled** — Compiler avtomatik:
   ```tsx
   'use memo';
   const handleClick = () => doSomething(); // Compiler cache
   ```

4. **Function logic hisoblash** — useCallback'ga emas, useMemo'ga:
   ```tsx
   const result = useCallback(() => heavyCompute(x), [x]); // ❌ Function reference
   const result = useMemo(() => heavyCompute(x), [x]); // ✅ Computed value
   ```

<details>
<summary><strong>Under the Hood</strong></summary>

`useCallback` source (cross-ref `21-usememo-usecallback.md`):

```javascript
// Conceptual equivalence (real React'da alohida `mountCallback`/`updateCallback` funksiyalari):
function useCallback(callback, deps) {
  return useMemo(() => callback, deps);
}
```

Konseptual ekvivalent. Real React internal'da alohida hook implementation (`mountCallback` va `updateCallback` `ReactFiberHooks.js`'da), lekin mexanika `useMemo` bilan deyarli bir xil — Hook chain'da bir slot, `areHookInputsEqual` deps check, cached value qaytarish.

`useCallback` ahamiyati — function reference saqlash:

```javascript
// Render 1
const fn1 = useCallback(() => x, []);
// fn1 = function-ref-100

// Render 2
const fn2 = useCallback(() => x, []);
// areHookInputsEqual([], []) → true
// Return cached function-ref-100 (bir xil reference)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Real-world data table:

```tsx
import { useState, useCallback, memo } from 'react';
import type { ReactElement } from 'react';

interface Row {
  id: string;
  name: string;
  email: string;
}

interface RowProps {
  row: Row;
  onEdit: (id: string) => void;
  onDelete: (id: string) => void;
  onSelect: (id: string) => void;
}

const TableRow = memo(function TableRow({
  row,
  onEdit,
  onDelete,
  onSelect,
}: RowProps): ReactElement {
  return (
    <tr onClick={() => onSelect(row.id)}>
      <td>{row.name}</td>
      <td>{row.email}</td>
      <td>
        <button onClick={(e) => { e.stopPropagation(); onEdit(row.id); }}>
          Edit
        </button>
        <button onClick={(e) => { e.stopPropagation(); onDelete(row.id); }}>
          Delete
        </button>
      </td>
    </tr>
  );
});

function DataTable(): ReactElement {
  const [rows, setRows] = useState<Row[]>([]);
  const [selectedId, setSelectedId] = useState<string | null>(null);
  
  // 3 ta callback — barchasi stable (functional update, no deps)
  const handleEdit = useCallback((id: string) => {
    console.log('Edit:', id);
  }, []);
  
  const handleDelete = useCallback((id: string) => {
    setRows((prev) => prev.filter((r) => r.id !== id));
  }, []);
  
  const handleSelect = useCallback((id: string) => {
    setSelectedId(id);
  }, []);
  
  return (
    <table>
      <tbody>
        {rows.map((row) => (
          <TableRow
            key={row.id}
            row={row}
            onEdit={handleEdit}
            onDelete={handleDelete}
            onSelect={handleSelect}
          />
        ))}
      </tbody>
    </table>
  );
}
// 1000+ rows: selectedId o'zgarsa — barcha TableRow bailout (props bir xil)
// Faqat selected row state UI'sida ko'rinadi (CSS/parent bilan)
// Stable callbacks orqali memo bailout ishlaydi.
```

</details>

---

## Key-Based Reset Trick

### Nazariya

**Key-based reset** — komponent state'ini "reset" qilish uchun `key` prop o'zgartirish. Komponent same type, lekin `key` farqli → Reconciler unmount + mount qiladi (cross-ref `08-list-rendering.md`, `04-reconciliation.md`).

```tsx
<UserForm key={userId} userId={userId} />

// userId o'zgarsa:
// 1. Eski UserForm unmount (state lost, useEffect cleanup)
// 2. Yangi UserForm mount (initial state)
```

NIMA UCHUN bu trick foydali:

1. **Form reset** — yangi user tanlanganda formni eski state'siz boshlash.
2. **Modal reset** — modal yopilib qayta ochilsa state reset.
3. **Route change** — URL parametri o'zgarganda komponent fresh state.
4. **Filter reset** — filter o'zgarsa list state reset (hover, expanded items).

Misollar:

### Use Case 1: User Detail Form Reset

```tsx
function UserDetailPage(): ReactElement {
  const [selectedUserId, setSelectedUserId] = useState<string>('user-1');
  
  return (
    <>
      <UserList onSelect={setSelectedUserId} />
      <UserDetailForm key={selectedUserId} userId={selectedUserId} />
    </>
  );
}

function UserDetailForm({ userId }: { userId: string }): ReactElement {
  const [draft, setDraft] = useState({ name: '', email: '' });
  const [isDirty, setIsDirty] = useState(false);
  
  // Yangi user tanlanganda key o'zgaradi → unmount + mount
  // draft, isDirty default state bilan boshlanadi
  
  return <form>...</form>;
}
```

vs `useEffect` reset (anti-pattern):

```tsx
function UserDetailForm({ userId }: { userId: string }): ReactElement {
  const [draft, setDraft] = useState({ name: '', email: '' });
  
  // ❌ Anti-pattern — derived state useEffect
  useEffect(() => {
    setDraft({ name: '', email: '' });
  }, [userId]);
  // Re-render: draft eski qiymat → useEffect reset → yana re-render → draft yangi
  // 2 render cycle, intermediate state ko'rinishi mumkin
}
```

Key-based reset — atomic, single render.

### Use Case 2: Modal State Reset

**Muhim nuance:** `key` trick faqat komponent **mounted bo'lib turgan paytda** ishlaydi. Agar Modal `{isOpen && <Modal>...}` orqali conditional render qilinsa, har close'da Modal va children unmount bo'ladi — fresh state allaqachon kafolat qilingan (key kerak emas). `key` trick foydali bo'ladigan stsenariy: Modal har doim mounted bo'lib, internal CSS hide/show (yoki Portal) bilan visibility boshqarsa.

```tsx
// Modal har doim mounted (CSS hide)
function Modal({ open, onClose, children }: ModalProps): ReactElement {
  return (
    <div
      style={{ display: open ? 'block' : 'none' }}
      className="modal-backdrop"
      onClick={onClose}
    >
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>
  );
}

function App(): ReactElement {
  const [isOpen, setIsOpen] = useState(false);
  const [openCount, setOpenCount] = useState(0);
  
  function handleOpen() {
    setOpenCount((c) => c + 1); // Force children re-mount via key change
    setIsOpen(true);
  }
  
  return (
    <>
      <button onClick={handleOpen}>Open Modal</button>
      <Modal open={isOpen} onClose={() => setIsOpen(false)}>
        <FormContent key={openCount} /> {/* key trick — even though Modal stays mounted */}
      </Modal>
    </>
  );
}
// Modal yopilib qayta ochilsa: openCount++ → FormContent key change → unmount + mount → fresh state
// Modal o'zi mounted, faqat FormContent reset bo'ladi.
```

### Use Case 3: Route-Based Reset (React Router pattern)

```tsx
import { useParams } from 'react-router-dom';

function ProductPage(): ReactElement {
  const { productId } = useParams<{ productId: string }>();
  
  return <ProductDetail key={productId} productId={productId!} />;
}
// Route /products/123 → /products/456: productId o'zgaradi → ProductDetail unmount + mount
// Hover state, expand state, scroll position — barchasi reset
```

### Use Case 4: Filter-Based Reset

```tsx
function ProductList({ category }: { category: string }): ReactElement {
  return <ProductTable key={category} category={category} />;
}

function ProductTable({ category }: { category: string }): ReactElement {
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name');
  const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());
  
  // Category change → key change → unmount + mount
  // sortBy va selectedIds reset
  
  return <table>...</table>;
}
```

NIMA UCHUN ehtiyot bo'lish:

1. **State lost** — kutilgan natija, lekin user ish jarayonini saqlash kerak bo'lsa muammo (form data lost).
2. **useEffect cleanup** — har key change'da effects cleanup → setup. Memory hot path'da overhead.
3. **DOM node yangi** — cursor position, scroll position lost.
4. **Re-mount cost** — initial state computation, useEffect setup — har key change.

Tavsiya etiladi — **explicit decision** (key bilan reset xohlanadigan natija ekanligini tasdiqlash).

<details>
<summary><strong>Under the Hood</strong></summary>

Reconciler key handling (cross-ref `04-reconciliation.md`):

```javascript
function updateElement(returnFiber, current, element) {
  if (current !== null) {
    if (current.elementType === element.type && current.key === element.key) {
      // Same type AND same key — update path
      return useFiber(current, element.props);
    }
  }
  
  // Different type or different key — create new
  // (eski Fiber unmount, yangi mount)
}
```

`key` prop — Fiber identity. Same key + same type → update. Farq → unmount + mount.

Key change lifecycle:

```
Old Fiber (key=A):
   - useEffect cleanup
   - DOM removal
   - Hook chain GC

New Fiber (key=B):
   - DOM creation (yangi element)
   - Component(props) chaqirilishi
   - Hook initial state
   - useEffect setup
```

Performance cost:

- DOM element create + insert: ~1-5ms (browser)
- useEffect cleanup + setup: variable (network, event listener)
- Hook initial state: cheap
- Children mount: depends on subtree size

Misol: large form key reset — 10-50ms (form fields, validation, listeners). Bu visible jank, lekin user expectation bilan moslashadi (yangi user tanlanganda fresh form).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Production form pattern:

```tsx
import { useState } from 'react';
import type { ReactElement } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
}

interface UserFormProps {
  user: User;
  onSave: (user: User) => void;
}

function UserForm({ user, onSave }: UserFormProps): ReactElement {
  // Initial state'ni props'dan
  const [draft, setDraft] = useState<User>(user);
  const [isDirty, setIsDirty] = useState(false);
  
  function handleChange<K extends keyof User>(field: K, value: User[K]) {
    setDraft((prev) => ({ ...prev, [field]: value }));
    setIsDirty(true);
  }
  
  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    onSave(draft);
    setIsDirty(false);
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        value={draft.name}
        onChange={(e) => handleChange('name', e.target.value)}
      />
      <input
        value={draft.email}
        onChange={(e) => handleChange('email', e.target.value)}
      />
      {isDirty && <span>Unsaved changes</span>}
      <button type="submit">Save</button>
    </form>
  );
}

interface UserListPageProps {
  users: User[];
  onSaveUser: (user: User) => void;
}

function UserListPage({ users, onSaveUser }: UserListPageProps): ReactElement {
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const selectedUser = users.find((u) => u.id === selectedId);
  
  return (
    <div>
      <ul>
        {users.map((u) => (
          <li
            key={u.id}
            onClick={() => setSelectedId(u.id)}
            style={{ background: u.id === selectedId ? '#eee' : 'transparent' }}
          >
            {u.name}
          </li>
        ))}
      </ul>
      
      {selectedUser && (
        <UserForm
          key={selectedUser.id} // ✅ Key reset
          user={selectedUser}
          onSave={onSaveUser}
        />
      )}
    </div>
  );
}
// Tanlangan user'ni o'zgartirish:
// 1. setSelectedId(newId)
// 2. UserForm key change
// 3. Eski form unmount (draft state lost — ✅ kutilgan)
// 4. Yangi form mount with user.email/name
// User typing'ni boshqa user'ga olib o'tmaydi.
```

Modal reset — Modal har doim mounted (CSS hide bilan), children key orqali reset:

```tsx
import { useState } from 'react';

interface ModalProps {
  open: boolean;
  onClose: () => void;
  children: ReactNode;
}

// Modal CSS hide ishlatadi (har doim mounted, faqat visibility o'zgaradi)
function Modal({ open, onClose, children }: ModalProps): ReactElement {
  return (
    <div
      className="modal-backdrop"
      style={{ display: open ? 'flex' : 'none' }}
      onClick={onClose}
    >
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>
  );
}

function App(): ReactElement {
  const [isOpen, setIsOpen] = useState(false);
  const [resetKey, setResetKey] = useState(0);
  
  function openModal() {
    setResetKey((k) => k + 1); // Force ContactForm re-mount
    setIsOpen(true);
  }
  
  return (
    <>
      <button onClick={openModal}>Open Form</button>
      <Modal open={isOpen} onClose={() => setIsOpen(false)}>
        <ContactForm key={resetKey} />
      </Modal>
    </>
  );
}
// Modal har doim mounted (children persist).
// Har openModal'da resetKey o'zgaradi → ContactForm key change → unmount + mount → fresh state.
// (avval-yarim to'ldirilgan form ko'rinmaydi)

// Eslatma: agar Modal `if (!open) return null` pattern ishlatsa, conditional render
// children'ni unmount qiladi va resetKey redundant — fresh state allaqachon kafolat qilingan.
```

</details>

---

## Splitting State Across Components

### Nazariya

**State splitting** — bir katta state'ni alohida komponent'larga bo'lish strategy. Re-render scope'ni cheklab, parent re-render frequency'ni kamaytiradi.

NIMA UCHUN foydali:

- **Parent re-render → barcha children top-down propagation**.
- **State isolation** — bir state qism o'zgarsa, boshqa qismlar memo bailout.
- **Re-render scope cheklash** — komponent o'z state'ini boshqarsa, parent re-render kerak emas.

### Anti-Pattern: Bitta State Provider

```tsx
// ❌ Anti-pattern: barcha state parent'da
function Dashboard(): ReactElement {
  const [user, setUser] = useState<User | null>(null);
  const [orders, setOrders] = useState<Order[]>([]);
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [searchQuery, setSearchQuery] = useState('');
  
  return (
    <div>
      <UserPanel user={user} setUser={setUser} />
      <OrderTable orders={orders} setOrders={setOrders} />
      <NotificationFeed notifications={notifications} setNotifications={setNotifications} />
      <SearchBar query={searchQuery} setQuery={setSearchQuery} />
    </div>
  );
}
// Har state setter ishlatilsa — Dashboard re-render → barcha children re-render
// (memo bilan ham — props'da setter funksiyasi bo'lsa stable bo'lishi shart)
// SearchBar typing — har keystroke barcha komponent'lar render bo'ladi
```

### Pattern: Local State per Component

```tsx
// ✅ Local state har komponent'da
function Dashboard(): ReactElement {
  return (
    <div>
      <UserPanel /> {/* Local state */}
      <OrderTable /> {/* Local state */}
      <NotificationFeed /> {/* Local state */}
      <SearchBar /> {/* Local state */}
    </div>
  );
}

function SearchBar(): ReactElement {
  const [query, setQuery] = useState(''); // Faqat SearchBar re-render har keystroke'da
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

### Pattern: State Container Component

State ko'p joyda kerak bo'lsa — Context yoki state container:

```tsx
function UserProvider({ children }: { children: ReactNode }): ReactElement {
  const [user, setUser] = useState<User | null>(null);
  
  return (
    <UserContext.Provider value={user}>
      <UserDispatchContext.Provider value={setUser}>
        {children}
      </UserDispatchContext.Provider>
    </UserContext.Provider>
  );
}
```

Provider re-render bo'ladi har user change'da, lekin children prop reference saqlanadi (cross-ref `04-reconciliation.md` element identity bailout):

```tsx
function App(): ReactElement {
  return (
    <UserProvider>
      <Dashboard /> {/* Reference saqlanadi — UserProvider re-render bypass */}
    </UserProvider>
  );
}
```

### Pattern: Component Splitting

Bir komponent ichida ko'p state — alohida komponent'larga bo'lish:

```tsx
// ❌ Bir katta komponent
function ContactForm(): ReactElement {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [phone, setPhone] = useState('');
  // ... 10 ta input
  
  // Har input keystroke — butun ContactForm re-render
  // 10+ ta input render bo'ladi
}

// ✅ Sub-component splitting
function ContactForm(): ReactElement {
  const [data, setData] = useState({ name: '', email: '', phone: '' });
  
  return (
    <form>
      <NameField value={data.name} onChange={(name) => setData((d) => ({ ...d, name }))} />
      <EmailField value={data.email} onChange={(email) => setData((d) => ({ ...d, email }))} />
      <PhoneField value={data.phone} onChange={(phone) => setData((d) => ({ ...d, phone }))} />
    </form>
  );
}
// Hali ContactForm re-render har field change'da, lekin
// memo + useCallback bilan optimization mumkin
```

Yoki state'ni har sub-component'ga ko'chirish (uncontrolled bilan):

```tsx
// FormData orqali submit-only read (cross-ref 14)
function ContactForm(): ReactElement {
  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const data = {
      name: formData.get('name'),
      email: formData.get('email'),
      phone: formData.get('phone'),
    };
    submit(data);
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="name" defaultValue="" /> {/* Uncontrolled */}
      <input name="email" defaultValue="" />
      <input name="phone" defaultValue="" />
      <button type="submit">Submit</button>
    </form>
  );
}
// Hech qanday state, har input mustaqil DOM'da
// ContactForm hech qachon re-render bo'lmaydi
// R19 form action bilan to'g'ridan-to'g'ri (cross-ref 13)
```

<details>
<summary><strong>Under the Hood</strong></summary>

State splitting — re-render scope mathematical:

```
Komponent A render time = T_a
Komponent B render time = T_b
Komponent C render time = T_c

Bitta state, parent A re-render har 100ms'da:
   Total render = T_a + T_b + T_c (har 100ms)
   Per second: 10 × (T_a + T_b + T_c)

Split state — A o'zgaradi har 100ms, B o'zgarmaydi, C har 1s:
   A re-render: 10 × T_a (per second)
   B re-render: 0 (memo bailout)
   C re-render: 1 × T_c
   Total: 10*T_a + T_c
   
Saving: 10*(T_b + T_c) - T_c = 10*T_b + 9*T_c
```

Real misol — search box:

```
T_a (SearchBar) = 1ms
T_b (UserList 1000 items) = 50ms
T_c (Stats) = 5ms

Bitta state — 10 keystroke/sec:
   Total: 10 × (1 + 50 + 5) = 560ms/sec — 56% CPU

Split:
   SearchBar re-render: 10 × 1 = 10ms
   UserList re-render: 0 (props bir xil)
   Stats re-render: 0
   Total: 10ms/sec — 1% CPU
   
Saving: 56% → 1% (50x improvement)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Real-world dashboard refactor:

```tsx
// === Before: monolithic ===
import { useState } from 'react';

function DashboardBefore(): ReactElement {
  const [user, setUser] = useState<User | null>(null);
  const [orders, setOrders] = useState<Order[]>([]);
  const [filter, setFilter] = useState('');
  const [sortBy, setSortBy] = useState<'date' | 'total'>('date');
  
  // Filter typing → Dashboard re-render → UserPanel + OrderTable re-render
  
  const filteredOrders = orders.filter((o) =>
    o.id.toLowerCase().includes(filter.toLowerCase())
  );
  
  return (
    <div>
      <UserPanel user={user} setUser={setUser} />
      <OrderTable orders={filteredOrders} sortBy={sortBy} setSortBy={setSortBy} />
      <FilterBar filter={filter} setFilter={setFilter} />
    </div>
  );
}

// === After: split ===
function DashboardAfter(): ReactElement {
  return (
    <div>
      <UserPanelContainer />
      <OrderTableContainer />
    </div>
  );
}

function UserPanelContainer(): ReactElement {
  const [user, setUser] = useState<User | null>(null);
  return <UserPanel user={user} onChange={setUser} />;
}

function OrderTableContainer(): ReactElement {
  const [orders] = useState<Order[]>([]);
  const [filter, setFilter] = useState('');
  const [sortBy, setSortBy] = useState<'date' | 'total'>('date');
  
  const filteredOrders = useMemo(
    () =>
      orders.filter((o) => o.id.toLowerCase().includes(filter.toLowerCase())),
    [orders, filter]
  );
  
  return (
    <>
      <FilterBar filter={filter} onChange={setFilter} />
      <OrderTable orders={filteredOrders} sortBy={sortBy} onSortChange={setSortBy} />
    </>
  );
}
// Filter typing — faqat OrderTableContainer re-render
// UserPanelContainer bypass (state alohida)
```

</details>

---

## Lift State Down — Re-render Scope Cheklash

### Nazariya

**Lift state down** — `lift state up`'ning teskarisi. State'ni minimal level'ga **tushirish** — qaysi komponent'lar haqiqatan ham state'ga muhtoj. Re-render scope'ni cheklab, optimization yutadi.

```tsx
// ❌ Anti-pattern: state Parent'da, faqat 1 child ishlatadi
function Parent(): ReactElement {
  const [hovered, setHovered] = useState(false); // ❌ Parent re-render har hover'da
  
  return (
    <>
      <ExpensiveSidebar />
      <Tooltip
        onMouseEnter={() => setHovered(true)}
        onMouseLeave={() => setHovered(false)}
        visible={hovered}
      />
    </>
  );
}

// ✅ State'ni Tooltip ichida
function Parent(): ReactElement {
  return (
    <>
      <ExpensiveSidebar />
      <Tooltip /> {/* State ichida */}
    </>
  );
}

function Tooltip(): ReactElement {
  const [hovered, setHovered] = useState(false);
  return (
    <div
      onMouseEnter={() => setHovered(true)}
      onMouseLeave={() => setHovered(false)}
    >
      {hovered && <span>Tooltip!</span>}
    </div>
  );
}
// Hover toggle — faqat Tooltip re-render, ExpensiveSidebar bypass
```

NIMA UCHUN bu effective:

- State Parent'da bo'lsa → Parent re-render har state change'da → barcha children top-down propagation.
- State Child'da bo'lsa → faqat Child re-render.
- React.memo'siz ham bu pattern ishlaydi (komponent boundary natural barrier).

### Pattern 1: Hover/Focus State

```tsx
// Har list item'da hover state — local
const ListItem = memo(function ListItem({ item }: { item: Item }): ReactElement {
  const [hovered, setHovered] = useState(false);
  
  return (
    <li
      onMouseEnter={() => setHovered(true)}
      onMouseLeave={() => setHovered(false)}
      style={{ background: hovered ? '#eee' : 'transparent' }}
    >
      {item.name}
    </li>
  );
});
```

### Pattern 2: Form Field Validation

```tsx
function FormField({ name, validator }: FormFieldProps): ReactElement {
  const [value, setValue] = useState('');
  const [error, setError] = useState<string | null>(null);
  
  function handleBlur() {
    setError(validator(value));
  }
  
  return (
    <div>
      <input
        name={name}
        value={value}
        onChange={(e) => setValue(e.target.value)}
        onBlur={handleBlur}
      />
      {error && <span className="error">{error}</span>}
    </div>
  );
}

// FormField'ning re-render scope — faqat o'zi.
// Parent form har FormField change'da re-render kerak emas.
```

### Pattern 3: Modal/Dropdown Open State

```tsx
function Dropdown({ children }: { children: ReactNode }): ReactElement {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div>
      <button onClick={() => setIsOpen((v) => !v)}>Toggle</button>
      {isOpen && <div className="dropdown-content">{children}</div>}
    </div>
  );
}

// Dropdown open/close — local state
// Parent komponent re-render qilmaydi
```

### Pattern 4: Animation State

```tsx
function AnimatedCard({ children }: { children: ReactNode }): ReactElement {
  const [isAnimating, setIsAnimating] = useState(false);
  
  function startAnimation() {
    setIsAnimating(true);
    setTimeout(() => setIsAnimating(false), 300);
  }
  
  return (
    <div
      className={isAnimating ? 'card animating' : 'card'}
      onClick={startAnimation}
    >
      {children}
    </div>
  );
}
// Animation isolated — boshqa komponent'lar re-render qilmaydi
```

### Qachon Lift State Up Kerak

State **shared** bo'lsa (cross-ref `14-lifting-and-controlled.md`):

- Sibling komponent'lar bir state'ga muhtoj.
- Parent state asosida child render qaror qiladi.
- State persistence (route change'da saqlanadi).

Ko'pincha **hybrid** — state'ning **bir qismi** parent'da, **boshqa qismi** child'da:

```tsx
function Form(): ReactElement {
  const [submittedData, setSubmittedData] = useState<FormData | null>(null); // Parent
  
  return (
    <>
      <FormFields onSubmit={setSubmittedData} /> {/* Local state ichida */}
      <PreviewPanel data={submittedData} />
    </>
  );
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Re-render scope — Reconciler perspective:

```
State at Parent:
   Parent
     ├─ Child A (re-render har Parent change'da)
     ├─ Child B (re-render har Parent change'da)
     └─ Child C (re-render har Parent change'da)

State at Child B:
   Parent (no re-render)
     ├─ Child A (no re-render — Parent unchanged)
     ├─ Child B (re-render only)
     └─ Child C (no re-render)
```

Reconciler tree walk:

- Parent re-render → render Child A, B, C (top-down).
- Child B re-render → faqat Child B subtree.

Bu pattern komponent boundaries'ni natural barrier sifatida ishlatadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Refactor: lift state down

```tsx
// === Before: state at parent ===
import { useState, memo } from 'react';
import type { ReactElement } from 'react';

interface Item {
  id: string;
  name: string;
}

const Sidebar = memo(function Sidebar(): ReactElement {
  console.log('Sidebar rendered');
  return <aside>Heavy sidebar</aside>;
});

function ListBefore({ items }: { items: Item[] }): ReactElement {
  const [hoveredId, setHoveredId] = useState<string | null>(null);
  
  return (
    <div>
      <Sidebar />
      <ul>
        {items.map((item) => (
          <li
            key={item.id}
            onMouseEnter={() => setHoveredId(item.id)}
            onMouseLeave={() => setHoveredId(null)}
            style={{ background: hoveredId === item.id ? '#eee' : 'transparent' }}
          >
            {item.name}
          </li>
        ))}
      </ul>
    </div>
  );
}
// Har hover: ListBefore re-render → Sidebar re-render (memo'siz bo'lganda)
// Sidebar memo bilan: bailout, lekin ListBefore body executes anyway

// === After: state at item ===
const ListItem = memo(function ListItem({ item }: { item: Item }): ReactElement {
  const [hovered, setHovered] = useState(false);
  
  return (
    <li
      onMouseEnter={() => setHovered(true)}
      onMouseLeave={() => setHovered(false)}
      style={{ background: hovered ? '#eee' : 'transparent' }}
    >
      {item.name}
    </li>
  );
});

function ListAfter({ items }: { items: Item[] }): ReactElement {
  return (
    <div>
      <Sidebar />
      <ul>
        {items.map((item) => (
          <ListItem key={item.id} item={item} />
        ))}
      </ul>
    </div>
  );
}
// Har hover: faqat ListItem re-render
// Sidebar va ListAfter — bypass (state move qilingan)
// 1000+ items: hover response sezilarli darajada tez (faqat o'sha item render bo'ladi)
```

</details>

---

## Context Optimization — Split, Memo, Selector

### Nazariya

Context — popular state-sharing mexanizm, lekin `default behavior` bilan **performance trap**'lar mavjud (cross-ref `19-usecontext.md`, `32-rendering-behavior.md`):

1. **Context value har render yangi reference** — barcha consumers re-render har gal.
2. **Tez-tez o'zgaruvchi state Context'da** — har consumer re-render frequency yuqori.
3. **Selector pattern yo'q** — Consumer butun value'ni oladi, faqat slice'ga subscribe qila olmaydi.

### Optimization 1: Provider Value Memo

```tsx
// ❌ Anti-pattern
function CartProvider({ children }: { children: ReactNode }): ReactElement {
  const [items, setItems] = useState<CartItem[]>([]);
  
  function addItem(item: CartItem) {
    setItems((prev) => [...prev, item]);
  }
  
  return (
    <CartContext.Provider value={{ items, addItem }}> {/* ❌ Yangi obyekt har render */}
      {children}
    </CartContext.Provider>
  );
}

// ✅ Memoized
function CartProvider({ children }: { children: ReactNode }): ReactElement {
  const [items, setItems] = useState<CartItem[]>([]);
  
  const addItem = useCallback((item: CartItem) => {
    setItems((prev) => [...prev, item]);
  }, []);
  
  const value = useMemo(() => ({ items, addItem }), [items, addItem]);
  
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}
```

### Optimization 2: Split Contexts (State vs Dispatch)

State o'zgarib turadigan, dispatch (setter) stable. Ularni alohida Context'larga ajratish:

```tsx
// State Context (re-render har items o'zgarganda)
const CartItemsContext = createContext<CartItem[] | null>(null);

// Dispatch Context (har doim stable)
type CartAction =
  | { type: 'ADD'; item: CartItem }
  | { type: 'REMOVE'; id: string }
  | { type: 'CLEAR' };

const CartDispatchContext = createContext<((action: CartAction) => void) | null>(null);

function CartProvider({ children }: { children: ReactNode }): ReactElement {
  const [items, dispatch] = useReducer(cartReducer, []);
  
  return (
    <CartItemsContext.Provider value={items}>
      <CartDispatchContext.Provider value={dispatch}> {/* Always stable */}
        {children}
      </CartDispatchContext.Provider>
    </CartItemsContext.Provider>
  );
}

// Consumer hooks
function useCartItems(): CartItem[] {
  const items = useContext(CartItemsContext);
  if (items === null) throw new Error('useCartItems must be used within CartProvider');
  return items;
}

function useCartDispatch(): (action: CartAction) => void {
  const dispatch = useContext(CartDispatchContext);
  if (dispatch === null) throw new Error('useCartDispatch must be used within CartProvider');
  return dispatch;
}

// AddButton — faqat dispatch ishlatadi
function AddButton({ item }: { item: CartItem }): ReactElement {
  const dispatch = useCartDispatch();
  return <button onClick={() => dispatch({ type: 'ADD', item })}>Add</button>;
}
// AddButton items o'zgarganda re-render bo'lmaydi (faqat dispatch context — stable).

// CartList — items'ga subscribe
function CartList(): ReactElement {
  const items = useCartItems();
  return <ul>{items.map((i) => <li key={i.id}>{i.name}</li>)}</ul>;
}
// CartList items o'zgarsa re-render.
```

### Optimization 3: Granular Contexts

Bir katta state'ni ko'p Context'larga bo'lish:

```tsx
// ❌ Bitta katta Context
const AppContext = createContext({
  user: null,
  theme: 'light',
  notifications: [], // Tez-tez yangilanadi
  preferences: {},
});

// ✅ Granular
const UserContext = createContext<User | null>(null);
const ThemeContext = createContext<Theme>('light');
const NotificationsContext = createContext<Notification[]>([]);
const PreferencesContext = createContext<Preferences>({});

// UserPanel — faqat UserContext consumer
// ThemeBadge — faqat ThemeContext consumer
// NotificationBell — faqat NotificationsContext consumer

// Notifications.push() — faqat NotificationsContext consumers re-render
// User logout — faqat UserContext consumers re-render
```

### Optimization 4: Selector Pattern

`useContextSelector` library yoki `useSyncExternalStore` (cross-ref `22-concurrent-hooks.md`):

```tsx
// use-context-selector library
import { createContext, useContextSelector } from 'use-context-selector';

const StoreContext = createContext<{ users: User[]; products: Product[] }>(null!);

function UserCount(): ReactElement {
  // Faqat users.length'ga subscribe — products o'zgarsa re-render bo'lmaydi
  const userCount = useContextSelector(StoreContext, (state) => state.users.length);
  return <span>{userCount} users</span>;
}
```

`use-context-selector` — Daishi Kato (Zustand author). Vanilla React'da `useSyncExternalStore` bilan custom selector qurish mumkin, lekin verbose.

> **Eslatma:** Modern state libraries (Zustand, Jotai, Redux Toolkit) selector pattern'ni built-in beradi. Katta app'larda Context o'rniga state library tavsiya etiladi (cross-ref `19-usecontext.md`).

<details>
<summary><strong>Under the Hood</strong></summary>

Context propagation cost:

```
Provider value change:
   propagateContextChange(workInProgress, context, renderLanes)
   
   Tree traverse: O(N) — N = consumer count
   Each consumer: lane priority assigned
   Render Phase: each consumer re-rendered
```

State libraries (Zustand) `useSyncExternalStore` orqali:

```
Store update:
   listeners.forEach(callback)
   
   Each subscriber: callback chaqiriladi
   Re-render only if `Object.is(prevSnapshot, newSnapshot)` false
   Selector pattern: faqat slice'ni qayta hisoblash
```

Misol: 100 consumer, store o'zgaradi:

- Context default: 100 re-render (har consumer).
- Context selector: ~10 re-render (faqat tegishli slice'ga subscribe qilganlar).
- Zustand selector: ~10 re-render (selector returns same reference for same slice).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Production cart implementation:

```tsx
import {
  createContext,
  useContext,
  useReducer,
  useCallback,
  useMemo,
} from 'react';
import type { ReactElement, ReactNode } from 'react';

interface CartItem {
  id: string;
  productId: string;
  quantity: number;
  price: number;
}

interface CartState {
  items: CartItem[];
  total: number;
  itemCount: number;
}

type CartAction =
  | { type: 'ADD'; item: Omit<CartItem, 'id'> }
  | { type: 'REMOVE'; id: string }
  | { type: 'UPDATE_QTY'; id: string; quantity: number }
  | { type: 'CLEAR' };

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'ADD': {
      const newItem: CartItem = { ...action.item, id: crypto.randomUUID() };
      const items = [...state.items, newItem];
      return {
        items,
        total: items.reduce((s, i) => s + i.price * i.quantity, 0),
        itemCount: items.length,
      };
    }
    case 'REMOVE': {
      const items = state.items.filter((i) => i.id !== action.id);
      return {
        items,
        total: items.reduce((s, i) => s + i.price * i.quantity, 0),
        itemCount: items.length,
      };
    }
    case 'UPDATE_QTY': {
      const items = state.items.map((i) =>
        i.id === action.id ? { ...i, quantity: action.quantity } : i
      );
      return {
        items,
        total: items.reduce((s, i) => s + i.price * i.quantity, 0),
        itemCount: items.length,
      };
    }
    case 'CLEAR':
      return { items: [], total: 0, itemCount: 0 };
  }
}

// Split contexts
const CartItemsContext = createContext<CartItem[] | null>(null);
const CartTotalContext = createContext<number | null>(null);
const CartCountContext = createContext<number | null>(null);
const CartDispatchContext = createContext<((action: CartAction) => void) | null>(null);

function CartProvider({ children }: { children: ReactNode }): ReactElement {
  const [state, dispatch] = useReducer(cartReducer, {
    items: [],
    total: 0,
    itemCount: 0,
  });
  
  return (
    <CartDispatchContext.Provider value={dispatch}>
      <CartItemsContext.Provider value={state.items}>
        <CartTotalContext.Provider value={state.total}>
          <CartCountContext.Provider value={state.itemCount}>
            {children}
          </CartCountContext.Provider>
        </CartTotalContext.Provider>
      </CartItemsContext.Provider>
    </CartDispatchContext.Provider>
  );
}

// Hooks
function useCartItems(): CartItem[] {
  const ctx = useContext(CartItemsContext);
  if (ctx === null) throw new Error('Must be within CartProvider');
  return ctx;
}

function useCartTotal(): number {
  const ctx = useContext(CartTotalContext);
  if (ctx === null) throw new Error('Must be within CartProvider');
  return ctx;
}

function useCartCount(): number {
  const ctx = useContext(CartCountContext);
  if (ctx === null) throw new Error('Must be within CartProvider');
  return ctx;
}

function useCartDispatch(): (action: CartAction) => void {
  const ctx = useContext(CartDispatchContext);
  if (ctx === null) throw new Error('Must be within CartProvider');
  return ctx;
}

// Consumers — har biri faqat o'ziga kerak Context'ga subscribe
function CartIcon(): ReactElement {
  const count = useCartCount(); // Faqat itemCount o'zgarganda re-render
  return <div>🛒 {count}</div>;
}

function CartTotal(): ReactElement {
  const total = useCartTotal(); // Faqat total o'zgarganda
  return <span>${total.toFixed(2)}</span>;
}

function AddProductButton({ product }: { product: Omit<CartItem, 'id'> }): ReactElement {
  const dispatch = useCartDispatch(); // Stable forever
  return <button onClick={() => dispatch({ type: 'ADD', item: product })}>Add</button>;
}
// AddProductButton hech qachon re-render bo'lmaydi cart o'zgarganda
// (faqat dispatch context, dispatch always stable).
```

</details>

---

## List Virtualization Preview

### Nazariya

**Virtualization** — faqat **ko'rinadigan** list items'ni DOM'ga render. 1000+ item list — 10-20 ta visible item (scroll'siz). Boshqalar virtual sifatida hisoblanadi (DOM emas).

NIMA UCHUN kerak:

- **DOM size** — 10000 ta `<li>` DOM hajmi ~50MB browser memory.
- **Initial mount time** — 10000 React Element create + reconciliation = 500ms+.
- **Scroll performance** — har scroll event Reconciler ishlaydi (event throttling bilan), lekin DOM update'lar ko'p.

Virtualization solution:

1. Container scroll position track.
2. Visible range hisoblash (`scrollTop`, `containerHeight`, `itemHeight`).
3. Faqat visible items render.
4. Spacer top/bottom — scroll height to'g'ri.

Library options (cross-ref `36-virtualization.md` chuqur):

- **`react-window`** — minimal, well-known.
- **`@tanstack/react-virtual`** — modern, headless, TanStack family.
- **`react-virtualized`** — feature-rich, lekin legacy.

Misol (`react-window`):

```tsx
import { FixedSizeList } from 'react-window';
import type { ReactElement, CSSProperties } from 'react';

interface RowProps {
  index: number;
  style: CSSProperties;
}

interface ItemData {
  id: string;
  name: string;
}

function VirtualizedList({ items }: { items: ItemData[] }): ReactElement {
  function Row({ index, style }: RowProps): ReactElement {
    const item = items[index];
    return (
      <div style={style}>
        {item.name}
      </div>
    );
  }
  
  return (
    <FixedSizeList
      height={600}        // Container height
      itemCount={items.length}
      itemSize={50}       // Each row height
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
// 1000+ items: faqat ~12 row visible at any time
// DOM: ~12 elements (vs 1000+)
// Scroll: smooth 60fps
```

Vanilla React virtualization (manual, simplified):

```tsx
import { useState, useRef, useEffect } from 'react';

interface SimpleVirtualListProps {
  items: ItemData[];
  itemHeight: number;
  containerHeight: number;
}

function SimpleVirtualList({
  items,
  itemHeight,
  containerHeight,
}: SimpleVirtualListProps): ReactElement {
  const [scrollTop, setScrollTop] = useState(0);
  const containerRef = useRef<HTMLDivElement>(null);
  
  const startIndex = Math.floor(scrollTop / itemHeight);
  const endIndex = Math.min(
    startIndex + Math.ceil(containerHeight / itemHeight) + 1,
    items.length
  );
  
  const visibleItems = items.slice(startIndex, endIndex);
  
  return (
    <div
      ref={containerRef}
      style={{ height: containerHeight, overflow: 'auto' }}
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
    >
      <div style={{ height: items.length * itemHeight, position: 'relative' }}>
        {visibleItems.map((item, i) => (
          <div
            key={item.id}
            style={{
              position: 'absolute',
              top: (startIndex + i) * itemHeight,
              height: itemHeight,
              left: 0,
              right: 0,
            }}
          >
            {item.name}
          </div>
        ))}
      </div>
    </div>
  );
}
// Production: library afzal (edge cases, dynamic heights, accessibility)
```

Qachon virtualization kerak:

- **List 100+ items va lag bor** — measure first.
- **Complex item rendering** — chart, image, nested structure.
- **Long scroll experience** — chat, table, infinite list.

Qachon kerak emas:

- **Pagination** afzal — 20-50 items per page.
- **Cheap items** — oddiy text — DOM hajmi katta emas.
- **Short list** — 50 ta item bilan virtualization overhead foyda yo'q.

> **Eslatma:** Virtualization chuqur o'rganish `36-virtualization.md`'da. Bu fayl optimization toolbox'ni eslatib o'tish uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

Virtualization algorithm:

```
Container height = 600px
Item height = 50px
Visible items count = 12 (600 / 50)

Scroll position = 250px
Start index = floor(250 / 50) = 5
End index = 5 + 12 + 1 = 18 (overscan: 1)

Render: items[5..18] (14 items)
DOM elements: 14
Total items: 10000

Memory (sifat-darajada):
   Without virtualization: 10000 DOM elementlari + listener'lar + layout state — barchasi heap'da.
   With virtualization: ~14 DOM elementlari (visible + overscan) — qolgan item'lar JS array'da metadata sifatida.

Saving: DOM element soni 10000 → 14 (visible window'da), katta ro'yxatlar uchun memory va layout/paint cost order of magnitude past tushadi (aniq raqamlar item complexity va browser engine'ga bog'liq).
```

Scroll handling:

- `scroll` event throttling (RAF or debounce).
- IntersectionObserver alternative (modern, no event handler).
- Variable height items: harder, kerak `dynamic-height` mode.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`@tanstack/react-virtual` modern pattern:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';
import type { ReactElement } from 'react';

interface Item {
  id: string;
  name: string;
  email: string;
}

function UserList({ users }: { users: Item[] }): ReactElement {
  const parentRef = useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: users.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60, // 60px per row
    overscan: 5,
  });
  
  return (
    <div
      ref={parentRef}
      style={{
        height: 600,
        overflow: 'auto',
      }}
    >
      <div
        style={{
          height: virtualizer.getTotalSize(),
          width: '100%',
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualRow) => {
          const user = users[virtualRow.index];
          return (
            <div
              key={virtualRow.key}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: `${virtualRow.size}px`,
                transform: `translateY(${virtualRow.start}px)`,
              }}
            >
              <div>{user.name} — {user.email}</div>
            </div>
          );
        })}
      </div>
    </div>
  );
}
// 100,000 users: ~10 visible items, smooth scroll
```

</details>

---

## Optimization Anti-Patterns

### Nazariya

Yo'l qo'yiladigan xatolar va ularning real cost'i:

### Anti-Pattern 1: `useMemo` Har Joyda

```tsx
function Greeting({ name }: { name: string }): ReactElement {
  const message = useMemo(() => `Hello, ${name}!`, [name]); // ❌ Useless
  return <h1>{message}</h1>;
}

// ✅ Direct
function Greeting({ name }: { name: string }): ReactElement {
  const message = `Hello, ${name}!`;
  return <h1>{message}</h1>;
}
```

Cost:
- useMemo overhead: ~0.01ms
- Computation: ~0.001ms (string concat)
- Net: useMemo bilan sezilarli SLOWER (overhead asl computation'dan ko'p)

### Anti-Pattern 2: `useCallback` `React.memo`'siz

```tsx
function Parent(): ReactElement {
  const handleClick = useCallback(() => alert('hi'), []); // ❌ Useless
  return <PlainButton onClick={handleClick} />;
}

function PlainButton({ onClick }: { onClick: () => void }) {
  return <button onClick={onClick}>Click</button>;
}
// PlainButton memo'siz — har Parent re-render'da PlainButton re-render har holatda
// useCallback foyda yo'q, faqat overhead
```

### Anti-Pattern 3: `React.memo` Always-Changing Props

```tsx
const SearchBar = memo(function SearchBar({ value, onChange }) {
  return <input value={value} onChange={(e) => onChange(e.target.value)} />;
});

function App(): ReactElement {
  const [query, setQuery] = useState('');
  return <SearchBar value={query} onChange={setQuery} />;
}
// query har keystroke o'zgaradi → SearchBar har gal re-render
// memo foyda yo'q (props doim yangi)
// Memo overhead: shallow comparison har render
```

### Anti-Pattern 4: Premature Context Splitting

```tsx
// ❌ 10 ta Context — over-engineering
const NameContext = createContext('');
const AgeContext = createContext(0);
const EmailContext = createContext('');
const PhoneContext = createContext('');
// ...

// ✅ Bir Provider, kerakli granularity
const UserContext = createContext({ name, age, email, phone });
// Real-world: profile o'zgarsa barcha consumers re-render — OK,
// chunki user data tez-tez o'zgarmaydi.
```

### Anti-Pattern 5: `useMemo` for Side Effects

```tsx
// ❌ Side effect useMemo'da
const data = useMemo(() => {
  fetch('/api/data').then(r => r.json()).then(setItems); // ❌ Side effect
  return items;
}, [items]);
```

```tsx
// ✅ useEffect
useEffect(() => {
  fetch('/api/data').then(r => r.json()).then(setItems);
}, []);
```

### Anti-Pattern 6: `React.memo` `useState` Setter Wrap

```tsx
const Memoized = memo(function Memoized({ setCount }: { setCount: Dispatch<...> }) { ... });

function Parent(): ReactElement {
  const [count, setCount] = useState(0);
  return <Memoized setCount={setCount} />;
}
// setCount stable forever — memo bailout always.
// Memoized memo overkill — wrap'siz bir xil natija (parent re-render top-down propagation
// lekin Memoized component pure, render time minimal).
```

### Anti-Pattern 7: Inline `useMemo` returning JSX

```tsx
function Parent({ items }: { items: Item[] }): ReactElement {
  // ❌ Anti-pattern: useMemo to memoize JSX
  const list = useMemo(
    () => items.map((i) => <li key={i.id}>{i.name}</li>),
    [items]
  );
  
  return <ul>{list}</ul>;
}
```

JSX creation cheap. memoization overhead > saving. Compiler era avtomatik.

```tsx
// ✅ Direct
function Parent({ items }: { items: Item[] }): ReactElement {
  return (
    <ul>
      {items.map((i) => <li key={i.id}>{i.name}</li>)}
    </ul>
  );
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Anti-patterns cost analysis:

```
Pattern: useMemo on primitive computation
   Without: 1 string concat ~0.001ms
   With: useMemo overhead ~0.01ms
   Net: sezilarli SLOWER (string concat'dan useMemo overhead ko'proq)

Pattern: useCallback for non-memoized child
   Without: function literal allocation ~0.0001ms
   With: useCallback overhead + deps comparison ~0.005ms
   Net: 50x SLOWER (negligible absolute)

Pattern: memo with always-changing props
   Without: render ~0.5ms
   With: shallow comparison + render ~0.51ms
   Net: 0.01ms slower per render
   1000 renders: 10ms wasted
```

Memoization tax accumulates — bundle size, parse time, runtime overhead.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Anti-pattern audit:

```tsx
// ❌ Anti-pattern collection
import { useMemo, useCallback, memo, useState } from 'react';

interface User {
  id: string;
  name: string;
}

const UserAvatar = memo(function UserAvatar({ user }: { user: User }) {
  // Memo bilan oddiy span — overkill
  return <span>{user.name[0]}</span>;
});

function OverMemoizedDashboard(): ReactElement {
  const [count, setCount] = useState(0);
  
  // 1. ❌ Useless useMemo
  const doubled = useMemo(() => count * 2, [count]);
  
  // 2. ❌ Useless useCallback (no memo child)
  const handleAdd = useCallback(() => setCount((c) => c + 1), []);
  
  // 3. ❌ useMemo for cheap object
  const config = useMemo(() => ({ size: 'md' }), []);
  
  // 4. ❌ useMemo for JSX
  const items = useMemo(() => Array.from({ length: 10 }, (_, i) => <li key={i}>{i}</li>), []);
  
  return (
    <>
      <button onClick={handleAdd}>Count: {count}, Doubled: {doubled}</button>
      <UserAvatar user={{ id: '1', name: 'Ali' }} /> {/* ❌ Inline object — memo bypass */}
      <ul>{items}</ul>
    </>
  );
}

// ✅ Cleaned up
function CleanDashboard(): ReactElement {
  const [count, setCount] = useState(0);
  const doubled = count * 2; // Direct
  
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>
        Count: {count}, Doubled: {doubled}
      </button>
      <span>A</span> {/* UserAvatar overkill replaced */}
      <ul>
        {Array.from({ length: 10 }, (_, i) => <li key={i}>{i}</li>)}
      </ul>
    </>
  );
}

// React Compiler era — automatic
// 'use memo';
// Compiler granular cache, manual hooks olib tashlanadi.
```

</details>

---

## React Compiler Era — Manual Qoladigan Qismlar

### Nazariya

React Compiler (cross-ref `31-react-compiler.md`) avtomatik komponent darajasidagi memoization va per-property granular cache qo'shadi. **Manual `useMemo`/`useCallback`/`React.memo` aksariyat ortiqcha bo'ladi.**

Lekin **Compiler bail-out qiladi** quyidagi hollarda:

1. Mutation (props/state/module-level).
2. Side effects in render.
3. Conditional hooks.
4. Refs render-time read/write.
5. Async hooks (R19 partial).
6. Complex closures.

Bail-out komponent — manual memoization fallback.

### Manual Qoladigan Patterns

### Pattern 1: Custom Equality Comparator

Compiler `Object.is` comparison qiladi. Custom logic kerak bo'lsa — manual `React.memo` comparator:

```tsx
const ProductCard = memo(
  function ProductCard({ product }: { product: Product }): ReactElement {
    return <div>{product.name}</div>;
  },
  (prev, next) => prev.product.id === next.product.id // ID-based identity
);
```

Compiler bu pattern'ni avtomatik tark eta olmaydi (Compiler default Object.is).

### Pattern 2: Library Boundary

Library mutation/side effect bilan ishlasa, Compiler bail-out qiladi shu komponentni. Manual memoization yoki effect boundary:

```tsx
// Library wrapper — Compiler bail-out
'use no memo';

import { useEffect, useState } from 'react';

function ChartAdapter({ data }: { data: ChartData }): ReactElement {
  const [chart, setChart] = useState<ChartInstance | null>(null);
  
  useEffect(() => {
    const c = chartLibrary.create(/* ... */); // Library mutation
    setChart(c);
    return () => c.destroy();
  }, [data]);
  
  return <div ref={chart?.containerRef} />;
}
// Manual memo ham qo'shilishi mumkin
```

### Pattern 3: Force Reset / Imperative Patterns

Key-based reset, useImperativeHandle, useRef'ga qo'lda yozish — Compiler granular cache qila olmaydi:

```tsx
// Key reset — Compiler hech narsa qila olmaydi
<UserForm key={userId} userId={userId} />

// useImperativeHandle — manual
const VideoPlayer = forwardRef(function VideoPlayer(props, ref) {
  useImperativeHandle(ref, () => ({
    play: () => videoRef.current?.play(),
    pause: () => videoRef.current?.pause(),
  }), []);
  // ...
});
```

### Pattern 4: Memoization with Side Effect Tracking

`useEffect` deps stability — useCallback hali ham foydali (Compiler effect setup'ni cache qila olmaydi mukammal):

```tsx
'use memo';

function ChatRoom({ roomId }: { roomId: string }): ReactElement {
  const onMessage = useCallback((event: MessageEvent) => {
    console.log(event.data);
  }, []);
  
  useEffect(() => {
    const ws = new WebSocket(`wss://chat/${roomId}`);
    ws.addEventListener('message', onMessage);
    return () => {
      ws.removeEventListener('message', onMessage);
      ws.close();
    };
  }, [roomId, onMessage]); // useCallback orqali stable
}
```

`useEffectEvent` RFC (experimental — R19 canary/experimental build'larda mavjud, stable'da yo'q; cross-ref `30-concurrent-react.md`) bu pattern'ni soddalashtirishi mumkin (kelajakda stable bo'lganda).

### Pattern 5: External Store Subscription

`useSyncExternalStore` (cross-ref `22-concurrent-hooks.md`, `30-concurrent-react.md`) — Compiler buni respect qiladi, lekin custom selector pattern qo'lda:

```tsx
// Library author primitive
function useStoreSelector<T, S>(
  store: ExternalStore<T>,
  selector: (state: T) => S
): S {
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState()),
    () => selector(store.getServerState())
  );
}
// Compiler: bu Custom hook, Rules of React compliant — memoize qiladi.
// Lekin selector caching manual (yoki library bilan).
```

### Migration Checklist (Compiler era)

```
✅ Yangi kod — manual memoization'siz (Compiler avtomatik)
✅ Mavjud kod — manual'ni gradual olib tashlash (refactor paytida)
✅ Custom comparator — manual qoldirish (Compiler default Object.is)
✅ Library boundary — manual yoki 'use no memo'
✅ Imperative refs — manual (Compiler limited)
⚠ useCallback effect deps — hali kerak (R19 useEffectEvent kelguncha)
⚠ Selector pattern — library yoki manual
```

<details>
<summary><strong>Under the Hood</strong></summary>

Compiler bail-out semantic:

```
Component analyzed
   │
   ├─ Bail-out detected → 'use no memo' equivalent (no auto-memoization)
   │                       Manual hooks ishlaydi avval kabi
   │
   └─ No bail-out → Auto-memoize (cache slots, JSX cache, callbacks)
                    Manual hooks ortiqcha lekin ishlaydi
```

`'use memo'` directive — Compiler enable for file. `'use no memo'` — disable.

Per-component decisions:

- File-level: `'use memo'` directive enables Compiler.
- Function-level: Compiler auto-detects component vs hook.
- Bail-out: silently skip per component, log warning in dev.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Mixed Compiler + manual pattern:

```tsx
'use memo';

import { useState, useEffect, useRef, memo, useImperativeHandle, forwardRef } from 'react';
import type { ReactElement } from 'react';

interface VideoPlayerHandle {
  play(): void;
  pause(): void;
  seek(time: number): void;
}

interface VideoPlayerProps {
  src: string;
  onProgress: (time: number) => void;
}

// useImperativeHandle — Compiler limited support
const VideoPlayer = forwardRef<VideoPlayerHandle, VideoPlayerProps>(function VideoPlayer(
  { src, onProgress },
  ref
) {
  const videoRef = useRef<HTMLVideoElement>(null);
  
  // Imperative API — manual
  useImperativeHandle(
    ref,
    () => ({
      play: () => videoRef.current?.play(),
      pause: () => videoRef.current?.pause(),
      seek: (time) => {
        if (videoRef.current) videoRef.current.currentTime = time;
      },
    }),
    []
  );
  
  // Effect with stable callback — useCallback hali kerak (R19 hozirda)
  useEffect(() => {
    const video = videoRef.current;
    if (!video) return;
    
    function handleTimeUpdate() {
      onProgress(video!.currentTime);
    }
    
    video.addEventListener('timeupdate', handleTimeUpdate);
    return () => video.removeEventListener('timeupdate', handleTimeUpdate);
  }, [onProgress]);
  // onProgress prop — agar parent useCallback qilmasa, har render effect re-setup
  // Compiler era: parent automatic memoize qiladi onProgress'ni
  
  return <video ref={videoRef} src={src} controls />;
});

// Custom comparator memo — manual
interface ChartProps {
  data: { id: string; values: number[] };
  options: ChartOptions;
}

const Chart = memo(
  function Chart({ data, options }: ChartProps): ReactElement {
    return <svg>...</svg>;
  },
  (prev, next) => {
    // Custom: faqat data.id muhim (data.values har real-time update'da yangi reference)
    return (
      prev.data.id === next.data.id &&
      prev.options === next.options
    );
  }
);
// Compiler era'da Compiler default Object.is — bu custom logic'ni override qila olmaydi.
// Manual memo qoldiriladi.
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: `useMemo` Cache "Garantiya Emas"

React docs: `useMemo` performance hint, semantik garantiya emas. React internal cache evict qila oladi:

```tsx
const id = useMemo(() => crypto.randomUUID(), []);
// ❌ Cache evict bo'lsa yangi id — bug

const [id] = useState(() => crypto.randomUUID());
// ✅ Lazy initial state — bir marta mount'da garantiyali
```

### Gotcha 2: `React.memo` Custom Comparator Reference Identity Lost

```tsx
const MyMemo = memo(MyComponent, (prev, next) => prev.items === next.items);
// items[0] o'zgargan bo'lsa lekin items reference saqlangan bo'lsa — miss bug
// Yaxshi: shallow check or selective comparison
```

### Gotcha 3: `useCallback` Stale Closure Trap

```tsx
const handleClick = useCallback(() => {
  console.log(count); // ❌ count closure stale agar deps yo'q
}, []);

// ✅ Functional update yoki deps:
const handleClick = useCallback(() => {
  setCount((c) => {
    console.log(c); // Latest c
    return c + 1;
  });
}, []);
```

### Gotcha 4: Key-Based Reset Memory Leak

```tsx
<HeavyComponent key={resetKey} />
// Har resetKey o'zgarishida full unmount/mount
// useEffect cleanup chaqiriladi, lekin agar cleanup buggy bo'lsa — leak
```

Cleanup symmetry har doim (cross-ref `30-concurrent-react.md` Invariant 3).

### Gotcha 5: Context Provider Value Memoization Deps Trap

```tsx
const value = useMemo(
  () => ({ items, addItem }),
  [items, addItem] // ✅ Both deps
);
// Lekin addItem useCallback bilan stable bo'lsa, items o'zgarsa har gal yangi value
// Bu kutilgan — items consumer'lar re-render bo'lishi kerak
```

---

## Common Mistakes

### ❌ Xato 1: Profile'siz Optimization

```tsx
// ❌ "Bu komponent slow bo'lishi kerak" intuition
const Component = memo(function Component() {
  const value = useMemo(() => computation(), []);
  const handler = useCallback(() => {}, []);
  // ... har joyda memoization
});
```

```tsx
// ✅ Profile birinchi, optimize keyin
function Component(): ReactElement {
  const value = computation();
  const handler = () => {};
  return <div>...</div>;
}
// Profile bottleneck topsa — targeted optimize
```

### ❌ Xato 2: `useMemo` Primitive Computation

```tsx
const total = useMemo(() => count + 1, [count]); // ❌ Useless overhead
```

```tsx
const total = count + 1; // ✅ Direct
```

### ❌ Xato 3: `React.memo` Without Stable Props

```tsx
const Memo = memo(MyComponent);
<Memo data={{ x: 1 }} onClick={() => {}} /> // ❌ Inline literals → bypass
```

```tsx
const data = useMemo(() => ({ x: 1 }), []);
const handler = useCallback(() => {}, []);
<Memo data={data} onClick={handler} /> // ✅
```

### ❌ Xato 4: Context Value Memoization Yo'q

```tsx
<MyContext.Provider value={{ items, action }}> {/* ❌ Yangi obyekt har render */}
```

```tsx
const value = useMemo(() => ({ items, action }), [items, action]);
<MyContext.Provider value={value}>
```

### ❌ Xato 5: Anonymous Nested Component

```tsx
function Parent({ items }: { items: Item[] }): ReactElement {
  function Item({ item }: { item: Item }) { return <li>...</li>; } // ❌ Yangi reference
  return <ul>{items.map((i) => <Item item={i} />)}</ul>;
}
```

```tsx
// ✅ Module-level
function Item({ item }: { item: Item }) { return <li>...</li>; }

function Parent({ items }: { items: Item[] }): ReactElement {
  return <ul>{items.map((i) => <Item key={i.id} item={i} />)}</ul>;
}
```

---

## Amaliy Mashqlar

### Mashq 1: Memo va useCallback Qo'llash (Oson)

`<TodoList>` komponenti har keystroke'da barcha `<TodoItem>` re-render qiladi. Optimize qiling.

```tsx
import { useState } from 'react';
import type { ReactElement } from 'react';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

function TodoItem({ todo, onToggle }: { todo: Todo; onToggle: (id: string) => void }): ReactElement {
  console.log(`TodoItem ${todo.id} render`);
  return (
    <li>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() => onToggle(todo.id)}
      />
      {todo.text}
    </li>
  );
}

function TodoList(): ReactElement {
  const [todos, setTodos] = useState<Todo[]>([
    { id: '1', text: 'A', completed: false },
    { id: '2', text: 'B', completed: false },
  ]);
  const [search, setSearch] = useState('');
  
  function toggleTodo(id: string) {
    setTodos((prev) =>
      prev.map((t) => (t.id === id ? { ...t, completed: !t.completed } : t))
    );
  }
  
  return (
    <div>
      <input value={search} onChange={(e) => setSearch(e.target.value)} />
      <ul>
        {todos.filter(t => t.text.includes(search)).map((t) => (
          <TodoItem key={t.id} todo={t} onToggle={toggleTodo} />
        ))}
      </ul>
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState, useCallback, useMemo, memo } from 'react';
import type { ReactElement } from 'react';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

interface TodoItemProps {
  todo: Todo;
  onToggle: (id: string) => void;
}

const TodoItem = memo(function TodoItem({ todo, onToggle }: TodoItemProps): ReactElement {
  console.log(`TodoItem ${todo.id} render`);
  return (
    <li>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() => onToggle(todo.id)}
      />
      {todo.text}
    </li>
  );
});

function TodoList(): ReactElement {
  const [todos, setTodos] = useState<Todo[]>([
    { id: '1', text: 'A', completed: false },
    { id: '2', text: 'B', completed: false },
  ]);
  const [search, setSearch] = useState('');
  
  // Stable callback
  const toggleTodo = useCallback((id: string) => {
    setTodos((prev) =>
      prev.map((t) => (t.id === id ? { ...t, completed: !t.completed } : t))
    );
  }, []);
  
  // Memoized filter
  const filtered = useMemo(
    () => todos.filter((t) => t.text.includes(search)),
    [todos, search]
  );
  
  return (
    <div>
      <input value={search} onChange={(e) => setSearch(e.target.value)} />
      <ul>
        {filtered.map((t) => (
          <TodoItem key={t.id} todo={t} onToggle={toggleTodo} />
        ))}
      </ul>
    </div>
  );
}

// Behavior:
// Search typing — TodoList re-render, filtered recomputed
// Filtered items match: TodoItem bailout (todo, onToggle bir xil)
// Filtered items don't match: TodoItem unmount (key boshqa)
// Toggle todo: faqat shu TodoItem re-render

// React Compiler era:
// 'use memo';
// Avtomatik — manual memo/useCallback/useMemo olib tashlash mumkin.
```

</details>

### Mashq 2: Key-Based Reset (Oson)

`<UserForm>` user select bo'lganda eski draft holatini saqlamaydi. Tuzating.

```tsx
function UserDetailPage(): ReactElement {
  const [users] = useState<User[]>([/* ... */]);
  const [selectedId, setSelectedId] = useState(users[0]?.id);
  const selectedUser = users.find((u) => u.id === selectedId);
  
  return (
    <>
      <select value={selectedId} onChange={(e) => setSelectedId(e.target.value)}>
        {users.map((u) => <option key={u.id} value={u.id}>{u.name}</option>)}
      </select>
      {selectedUser && <UserForm user={selectedUser} />}
    </>
  );
}

function UserForm({ user }: { user: User }): ReactElement {
  const [draft, setDraft] = useState(user); // ❌ Faqat mount'da
  
  return (
    <form>
      <input value={draft.name} onChange={(e) => setDraft({ ...draft, name: e.target.value })} />
    </form>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

User o'zgarganda `<UserForm>`'ni full unmount/mount — `key` prop:

```tsx
function UserDetailPage(): ReactElement {
  const [users] = useState<User[]>([/* ... */]);
  const [selectedId, setSelectedId] = useState(users[0]?.id);
  const selectedUser = users.find((u) => u.id === selectedId);
  
  return (
    <>
      <select value={selectedId} onChange={(e) => setSelectedId(e.target.value)}>
        {users.map((u) => <option key={u.id} value={u.id}>{u.name}</option>)}
      </select>
      {selectedUser && (
        <UserForm key={selectedUser.id} user={selectedUser} /> // ✅ Key reset
      )}
    </>
  );
}
```

Behavior:
- `selectedId` change: `selectedUser.id` change → `key` change → UserForm unmount + mount.
- Unmount: eski draft state lost (useEffect cleanup chaqiriladi).
- Mount: yangi useState initial qiymat (`user` props bilan).
- Yangi user'ning ma'lumotlari fresh form'da.

Alternative — useEffect derived state (anti-pattern):

```tsx
// ❌ Anti-pattern
function UserForm({ user }: { user: User }): ReactElement {
  const [draft, setDraft] = useState(user);
  
  useEffect(() => {
    setDraft(user); // ❌ User change'da sync
  }, [user]);
  
  // 2 render cycle, intermediate state ko'rinadi
}
```

Key-based reset — atomic, 1 render cycle, clean.

</details>

### Mashq 3: Lift State Down (O'rta)

`<Dashboard>` har hover'da barcha komponent'larni re-render qiladi. State'ni split qiling.

```tsx
import { useState } from 'react';
import type { ReactElement } from 'react';

interface Item {
  id: string;
  name: string;
}

function Sidebar(): ReactElement {
  console.log('Sidebar render');
  // ... heavy sidebar
  return <aside>Sidebar</aside>;
}

function ItemList({ items, hoveredId, onHover }: {
  items: Item[];
  hoveredId: string | null;
  onHover: (id: string | null) => void;
}): ReactElement {
  return (
    <ul>
      {items.map((item) => (
        <li
          key={item.id}
          onMouseEnter={() => onHover(item.id)}
          onMouseLeave={() => onHover(null)}
          style={{ background: hoveredId === item.id ? '#eee' : 'transparent' }}
        >
          {item.name}
        </li>
      ))}
    </ul>
  );
}

function Dashboard(): ReactElement {
  const [hoveredId, setHoveredId] = useState<string | null>(null);
  const [items] = useState<Item[]>([
    { id: '1', name: 'A' },
    { id: '2', name: 'B' },
  ]);
  
  return (
    <div>
      <Sidebar />
      <ItemList items={items} hoveredId={hoveredId} onHover={setHoveredId} />
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

State'ni `<ListItem>` ichiga ko'chiramiz — har item o'z hover state'ini boshqaradi:

```tsx
import { useState, memo } from 'react';
import type { ReactElement } from 'react';

interface Item {
  id: string;
  name: string;
}

const Sidebar = memo(function Sidebar(): ReactElement {
  console.log('Sidebar render');
  return <aside>Sidebar</aside>;
});

const ListItem = memo(function ListItem({ item }: { item: Item }): ReactElement {
  const [hovered, setHovered] = useState(false); // ✅ Local state
  
  return (
    <li
      onMouseEnter={() => setHovered(true)}
      onMouseLeave={() => setHovered(false)}
      style={{ background: hovered ? '#eee' : 'transparent' }}
    >
      {item.name}
    </li>
  );
});

function ItemList({ items }: { items: Item[] }): ReactElement {
  return (
    <ul>
      {items.map((item) => (
        <ListItem key={item.id} item={item} />
      ))}
    </ul>
  );
}

function Dashboard(): ReactElement {
  const [items] = useState<Item[]>([
    { id: '1', name: 'A' },
    { id: '2', name: 'B' },
  ]);
  
  return (
    <div>
      <Sidebar />
      <ItemList items={items} />
    </div>
  );
}

// Behavior:
// Hover item — faqat shu ListItem re-render
// Sidebar memo bailout (props yo'q, bir xil)
// Dashboard re-render qilmaydi (state Dashboard'da yo'q)
// 1000+ items: hover scope minimal, no top-down propagation
```

</details>

### Mashq 4: Context Split — State vs Dispatch (O'rta)

`<NotificationProvider>` har action'da barcha consumer'larni re-render qiladi. Split qiling.

```tsx
import { createContext, useContext, useState, useCallback } from 'react';
import type { ReactNode, ReactElement } from 'react';

interface Notification {
  id: string;
  text: string;
}

interface NotificationContextValue {
  notifications: Notification[];
  add: (text: string) => void;
  remove: (id: string) => void;
}

const NotificationContext = createContext<NotificationContextValue | null>(null);

function NotificationProvider({ children }: { children: ReactNode }): ReactElement {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  
  const add = useCallback((text: string) => {
    setNotifications((prev) => [...prev, { id: crypto.randomUUID(), text }]);
  }, []);
  
  const remove = useCallback((id: string) => {
    setNotifications((prev) => prev.filter((n) => n.id !== id));
  }, []);
  
  return (
    <NotificationContext.Provider value={{ notifications, add, remove }}>
      {children}
    </NotificationContext.Provider>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { createContext, useContext, useReducer, useCallback } from 'react';
import type { ReactNode, ReactElement } from 'react';

interface Notification {
  id: string;
  text: string;
}

type NotificationAction =
  | { type: 'ADD'; text: string }
  | { type: 'REMOVE'; id: string };

function reducer(state: Notification[], action: NotificationAction): Notification[] {
  switch (action.type) {
    case 'ADD':
      return [...state, { id: crypto.randomUUID(), text: action.text }];
    case 'REMOVE':
      return state.filter((n) => n.id !== action.id);
  }
}

// Split contexts
const NotificationsContext = createContext<Notification[] | null>(null);
const NotificationDispatchContext = createContext<((action: NotificationAction) => void) | null>(null);

function NotificationProvider({ children }: { children: ReactNode }): ReactElement {
  const [notifications, dispatch] = useReducer(reducer, []);
  
  return (
    <NotificationDispatchContext.Provider value={dispatch}>
      <NotificationsContext.Provider value={notifications}>
        {children}
      </NotificationsContext.Provider>
    </NotificationDispatchContext.Provider>
  );
}

// Hooks
function useNotifications(): Notification[] {
  const ctx = useContext(NotificationsContext);
  if (ctx === null) throw new Error('Must be within NotificationProvider');
  return ctx;
}

function useNotificationDispatch(): (action: NotificationAction) => void {
  const ctx = useContext(NotificationDispatchContext);
  if (ctx === null) throw new Error('Must be within NotificationProvider');
  return ctx;
}

// Consumers
function NotificationList(): ReactElement {
  const notifications = useNotifications(); // Re-render har list change'da
  return (
    <ul>
      {notifications.map((n) => (
        <li key={n.id}>{n.text}</li>
      ))}
    </ul>
  );
}

function AddNotificationButton(): ReactElement {
  const dispatch = useNotificationDispatch(); // Stable forever
  return (
    <button onClick={() => dispatch({ type: 'ADD', text: 'New!' })}>
      Add
    </button>
  );
}
// AddNotificationButton notifications o'zgarsa ham re-render bo'lmaydi
// (faqat dispatch context, dispatch always stable).
```

Avantajlar:
- `useReducer` dispatch automatically stable.
- Notifications consumer va dispatch consumer mustaqil re-render scope.
- ~50% fewer re-renders typical app'da (dispatch consumer'lar ko'p — buttons).

</details>

### Mashq 5: Production Optimization Workflow (Qiyin)

`<DataDashboard>` production'da slow. Profile qilib, bottleneck topib, optimize qiling.

```tsx
import { useState, useEffect } from 'react';
import type { ReactElement } from 'react';

interface DataPoint {
  id: string;
  timestamp: number;
  value: number;
  category: string;
}

function DataChart({ data, type }: { data: DataPoint[]; type: 'line' | 'bar' }): ReactElement {
  // 1000+ SVG elements
  return <svg>...</svg>;
}

function DataTable({ data }: { data: DataPoint[] }): ReactElement {
  return (
    <table>
      <tbody>
        {data.map((d) => (
          <tr key={d.id}>
            <td>{new Date(d.timestamp).toISOString()}</td>
            <td>{d.value}</td>
            <td>{d.category}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}

function DataDashboard(): ReactElement {
  const [data, setData] = useState<DataPoint[]>([]);
  const [filter, setFilter] = useState('');
  const [chartType, setChartType] = useState<'line' | 'bar'>('line');
  
  useEffect(() => {
    fetch('/api/data')
      .then((r) => r.json())
      .then(setData);
  }, []);
  
  const filtered = data.filter((d) => d.category.includes(filter));
  
  return (
    <div>
      <input
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
        placeholder="Filter category..."
      />
      <select value={chartType} onChange={(e) => setChartType(e.target.value as 'line' | 'bar')}>
        <option value="line">Line</option>
        <option value="bar">Bar</option>
      </select>
      <DataChart data={filtered} type={chartType} />
      <DataTable data={filtered} />
    </div>
  );
}
```

Profile natijasi: filter typing — 200ms+ commits, 1000+ DataPoint'lar.

<details>
<summary><strong>Javob</strong></summary>

Optimization 4 ta yondashuv:

```tsx
import {
  useState,
  useEffect,
  useMemo,
  useTransition,
  memo,
} from 'react';
import type { ReactElement } from 'react';

interface DataPoint {
  id: string;
  timestamp: number;
  value: number;
  category: string;
}

// 1. Chart memo — filtered/type bir xil bo'lsa re-render bo'lmaydi
const DataChart = memo(function DataChart({
  data,
  type,
}: {
  data: DataPoint[];
  type: 'line' | 'bar';
}): ReactElement {
  return <svg>...</svg>;
});

// 2. DataTable memo
const DataTable = memo(function DataTable({ data }: { data: DataPoint[] }): ReactElement {
  return (
    <table>
      <tbody>
        {data.map((d) => (
          <tr key={d.id}>
            <td>{new Date(d.timestamp).toISOString()}</td>
            <td>{d.value}</td>
            <td>{d.category}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
});

function DataDashboard(): ReactElement {
  const [data, setData] = useState<DataPoint[]>([]);
  const [filter, setFilter] = useState('');
  const [deferredFilter, setDeferredFilter] = useState('');
  const [chartType, setChartType] = useState<'line' | 'bar'>('line');
  const [isPending, startTransition] = useTransition();
  
  useEffect(() => {
    fetch('/api/data')
      .then((r) => r.json())
      .then(setData);
  }, []);
  
  function handleFilterChange(value: string) {
    setFilter(value); // Sync — input responsive
    
    // 3. useTransition — filter non-urgent
    startTransition(() => {
      setDeferredFilter(value);
    });
  }
  
  // 4. Filter result memoization
  const filtered = useMemo(
    () => data.filter((d) => d.category.includes(deferredFilter)),
    [data, deferredFilter]
  );
  
  return (
    <div>
      <input
        value={filter}
        onChange={(e) => handleFilterChange(e.target.value)}
        placeholder="Filter category..."
      />
      <select value={chartType} onChange={(e) => setChartType(e.target.value as 'line' | 'bar')}>
        <option value="line">Line</option>
        <option value="bar">Bar</option>
      </select>
      {isPending && <span>Filtering...</span>}
      <div style={{ opacity: isPending ? 0.5 : 1 }}>
        <DataChart data={filtered} type={chartType} />
        <DataTable data={filtered} />
      </div>
    </div>
  );
}

// Profile after:
// - Filter typing: input sync responsive
// - Heavy filter + render: Transition Lane interruptible
// - Chart/Table memo: faqat data o'zgarsa re-render
// - p95 commit duration: 200ms → 20ms (sezilarli improvement)

// 5. Future: Virtualization (cross-ref `36-virtualization.md`) for DataTable agar 10000+ rows
import { FixedSizeList } from 'react-window';
// 10000 rows: ~12 visible items, smooth scroll

// 6. Compiler era:
// 'use memo';
// Compiler avtomatik filter memoize, Chart/Table memo equivalent
// Manual hooks olib tashlash mumkin
```

5 ta optimization step:

1. ✅ `React.memo` for Chart, Table — top-down propagation block.
2. ✅ `useMemo` for filtered — expensive computation cache.
3. ✅ `useTransition` — input sync, filter non-urgent (Concurrent rendering).
4. ✅ `useDeferredValue` alternative pattern — value defer.
5. ⚠ Virtualization (cross-ref [`36-virtualization.md`](36-virtualization.md)) — 10000+ rows uchun.

</details>

---

## Xulosa

Bu fayl optimization'ning real qo'llash patternlari:

- **Premature optimization root of all evil** — profile birinchi (DevTools Profiler), optimize keyin. React docs: "you don't need to optimize re-renders. React is fast by default".
- **Measure va Identify** — 7 qadam workflow (symptom → reproduce → record → analyze "Why did this render?" → optimization → re-profile → production monitoring).
- **`React.memo` real patterns** — list item, expensive presentational, tez-tez re-render parent, Context Provider children. Custom comparator faqat specific scenarios (ID-based identity, deep equal, animation pause).
- **`useMemo`/`useCallback`** — referential identity stabilize, expensive computation cache, useEffect dep stability, custom hook returns. Anti-pattern primitive computation va memo'siz consumer.
- **Key-based reset** — komponent state reset trick (form/modal/route/filter). Unmount + mount, state lost, useEffect cleanup → setup. Atomic 1-render cycle (vs useEffect derived state 2-render).
- **State splitting** — bitta state Provider re-render scope kengaytiradi, alohida komponent'larga split (parent re-render frequency kamayadi). Container component, local state, FormData uncontrolled.
- **Lift state down** — state'ni komponent boundary'ga tushirish (hover, modal open, animation). Re-render scope minimal.
- **Context optimization** — Provider value memo, split contexts (state vs dispatch), granular Context (User/Theme/Notifications), selector pattern (use-context-selector library yoki state library Zustand/Jotai).
- **Virtualization preview** (cross-ref [`36-virtualization.md`](36-virtualization.md)) — `react-window`/`@tanstack/react-virtual`, 100,000+ items smooth scroll.
- **Anti-Patterns** — useMemo har joyda (primitive computation), useCallback memo'siz, memo always-changing props, premature Context splitting, useMemo for side effects, memo useState setter wrap, useMemo for JSX.
- **React Compiler era** — manual aksariyat ortiqcha, lekin manual qoladigan: custom comparator, library boundary, imperative refs, useEffect dep stability, selector pattern. Compiler bail-out manual fallback.

Keyingi fayl [`34-profiling.md`](34-profiling.md) — DevTools Profiler chuqur, `<Profiler>` component, production profiling, Web Vitals integration.

---

**Keyingi bo'lim:** [34-profiling.md](34-profiling.md) — DevTools Profiler chuqur (record, flame chart, ranked chart, commit details), `<Profiler>` component programmatic API (onRender callback fields actualDuration/baseDuration/startTime/commitTime), production profiling (`react-dom/profiling` build), Web Vitals integration (LCP/INP/CLS — March 2024'da INP FID o'rniga stable), real-world workflow (slow component identify → optimize → measure), Highlight Updates option, Strict Mode 2x render impact on profile.
