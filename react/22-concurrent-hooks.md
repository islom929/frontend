# Bo'lim 22: Concurrent Hooks (R18)

> R18 React'ga **Concurrent rendering** (default, opt-in toggleable "Concurrent Mode" emas) va u bilan bog'liq 4 ta yangi hook olib keldi: `useTransition` (non-urgent state update + isPending flag), `useDeferredValue` (defer expensive value), `useSyncExternalStore` (external store subscription tearing prevention library author'lar uchun), `useId` (SSR-safe deterministic unique ID generation). Bu hook'lar Concurrent rendering (Lanes priority, time slicing, render restart) bilan to'g'ridan-to'g'ri bog'lanadi (cross-ref [`05-scheduler-lanes.md`](05-scheduler-lanes.md)). Bu bo'limda har hook'ning API, mexanizmi, use case'lari, decision tree va edge case'lari yoritiladi.

---

## Mundarija

- [Concurrent Rendering Recap](#concurrent-rendering-recap)
- [`useTransition` API](#usetransition-api)
- [`startTransition` Function](#starttransition-function)
- [`isPending` Flag Pattern](#ispending-flag-pattern)
- [`useDeferredValue` API](#usedeferredvalue-api)
- [`useTransition` vs `useDeferredValue` — Decision](#usetransition-vs-usedeferredvalue--decision)
- [`useSyncExternalStore` API](#usesyncexternalstore-api)
- [Tearing Prevention — Concurrent rendering](#tearing-prevention--concurrent-rendering)
- [`useId` API](#useid-api)
- [Hydration Mismatch Prevention](#hydration-mismatch-prevention)
- [Decision Tree — Qaysi Hook Qachon](#decision-tree--qaysi-hook-qachon)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Concurrent Rendering Recap

### Nazariya

R18'dan boshlab Concurrent rendering — **default** (`createRoot`). R16-R17'dagi Sync rendering R18'da legacy mode'ga ko'chgan (`ReactDOM.render` deprecated), R19'da `ReactDOM.render` va `ReactDOM.hydrate` butunlay olib tashlangan — faqat `createRoot`/`hydrateRoot` qoldi.

**Terminologiya:** R16-R17 davridagi "Concurrent Mode" (yoqib-o'chiriladigan alohida rejim) tushunchasi R18'da bekor qilingan. R18 buning o'rniga "Concurrent rendering" / "Concurrent features" beradi: alohida rejim yo'q, `createRoot` bilan Concurrent rendering default yoqilgan, `useTransition`/`useDeferredValue` kabi feature'lar opt-in. Quyida "Concurrent rendering" shu ma'noda ishlatiladi.

**Concurrent rendering asosiy printsiplari** (cross-ref [`05-scheduler-lanes.md`](05-scheduler-lanes.md)):

1. **Time slicing** — render katta ish bo'lsa kichik chunks'ga bo'linadi (~5ms har chunk), browser frame'lar orasida bajariladi
2. **Interruptible rendering** — render davomida high-priority work (user input) kelsa, joriy render to'xtatiladi va high-priority ish birinchi bajariladi
3. **Lanes** — har update priority lane'ga o'zlashtiriladi (SyncLane, InputContinuousLane, DefaultLane, TransitionLane×15, RetryLane×4, IdleLane)
4. **Render restart** — render qaytarilishi mumkin (state o'zgarsa, priority interrupt bo'lsa)

**Sync vs Concurrent farq:**

```
Sync (R17):
  setState() → Render boshlandi → Tugaguncha bloklanadi → Commit
  (Browser javob bermaydi render davomida)

Concurrent (R18):
  setState() → Render scheduled → Time slicing → Interruptible
  (Browser javob beradi, high-priority work parallel)
```

**Concurrent rendering uchun yangi hook'lar:**

| Hook | Maqsad |
|------|--------|
| `useTransition` | State update'ni non-urgent transition lane'ga ko'chirish |
| `useDeferredValue` | Value'ni "ortda" yangilash (priority pastroq) |
| `useSyncExternalStore` | External store subscription Concurrent-safe |
| `useId` | SSR-safe unique ID (Concurrent rendering paytida ham deterministic) |

**Use case'lar — qachon kerak:**

- Search/filter — typing fast, results compute slow → `useDeferredValue`
- Tab switch — heavy content render → `useTransition`
- External library state — Redux/Zustand/Recoil → `useSyncExternalStore`
- Form labels — `htmlFor`/`aria-describedby` IDs → `useId`

**Concurrent hook'lar ishlamay qoladigan holatlar:**

- R18 legacy `ReactDOM.render` (R19'da bu API yo'q)
- `flushSync` ichida — barcha state updates SyncLane'ga majburlanadi (transitions ham)
- React internal sync work (initial render, error recovery)

R19'da `createRoot` yagona entry point — Concurrent default. R18'da `ReactDOM.render` (legacy) Concurrent'ni o'chiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Lanes va priority (cross-ref 05):**

```ts
// React internal — Lanes bitmap (source: react-reconciler/src/ReactFiberLane.js, v19.0.0)
const SyncHydrationLane            = 0b0000000000000000000000000000001;  // bit 0
const SyncLane                     = 0b0000000000000000000000000000010;  // bit 1
const InputContinuousHydrationLane = 0b0000000000000000000000000000100;  // bit 2
const InputContinuousLane          = 0b0000000000000000000000000001000;  // bit 3
const DefaultHydrationLane         = 0b0000000000000000000000000010000;  // bit 4
const DefaultLane                  = 0b0000000000000000000000000100000;  // bit 5
const TransitionHydrationLane      = 0b0000000000000000000000001000000;  // bit 6
const TransitionLane1              = 0b0000000000000000000000010000000;  // bit 7
// ... TransitionLane2..15 — bits 8-21 (jami 15 transition lane, TransitionLane1..15) ...
// RetryLane1..4 — bits 22-25, SelectiveHydrationLane — bit 26
const IdleHydrationLane            = 0b0001000000000000000000000000000;  // bit 27
const IdleLane                     = 0b0010000000000000000000000000000;  // bit 28
const OffscreenLane                = 0b0100000000000000000000000000000;  // bit 29
const DeferredLane                 = 0b1000000000000000000000000000000;  // bit 30
// TotalLanes = 31 (bitmask 31 bitli butun son sifatida ifodalanadi)
```

`useTransition` update'lari → TransitionLane (priority DefaultLane'dan pastroq).

**Render restart:**

```
Render TransitionLane started
  ↓ (5ms chunk)
Pause for browser
  ↓
User clicks (SyncLane update)
  ↓
React: SyncLane > TransitionLane → Pause TransitionLane render
  ↓
Render SyncLane (commit qilinadi)
  ↓
Resume TransitionLane render (yangi state bilan, restart)
```

Concurrent rendering = interruptible + restart safe. Render funksiyasi pure shart (cross-ref [`09-component-basics.md`](09-component-basics.md) "Render Purity").

**Time slicing — `MessageChannel`:**

```ts
// React Scheduler (cross-ref 05) — soddalashtirilgan pseudo-code
const channel = new MessageChannel();
channel.port1.onmessage = performWorkUntilDeadline;

function performWorkUntilDeadline() {
  const start = performance.now();
  // React Scheduler yield deadline ~5ms (browser frame budget'iga moslashtirilgan)
  // Aniq qiymat React Scheduler implementation'iga bog'liq, internal detail
  const yieldInterval = 5;
  
  while (workQueue.length > 0 && performance.now() - start < yieldInterval) {
    doWork();
  }
  
  if (workQueue.length > 0) {
    channel.port2.postMessage(null);  // Reschedule keyingi macrotask
  }
}
```

`MessageChannel` — `setTimeout(fn, 0)`'dan ko'p hollarda yaxshiroq (browser'lar `setTimeout`'da minimum delay clamp qilishi mumkin, masalan 4ms). Cross-browser ishlaydi (`requestIdleCallback` Safari'da yo'q edi, shuning uchun React MessageChannel bilan o'z scheduler'ini quradi).

**Source citation:**

- React 18 release notes — react.dev/blog/2022/03/29/react-v18
- Concurrent rendering RFC — reactjs/rfcs

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — `createRoot` Concurrent enable:**

```tsx
// React 18+ — Concurrent default (R19'da yagona entry point)
import { createRoot } from 'react-dom/client';

const root = createRoot(document.getElementById('root')!);
root.render(<App />);

// R18 legacy — Sync mode (deprecated, R19'da olib tashlangan)
// import ReactDOM from 'react-dom';
// ReactDOM.render(<App />, document.getElementById('root'));
```

**Misol 2 — Sync vs Concurrent demo:**

```tsx
function App() {
  const [count, setCount] = useState(0);
  const [results, setResults] = useState<Item[]>([]);
  
  // Sync mode: setResults expensive bo'lsa — UI freeze
  // Concurrent rendering + useTransition: setResults non-urgent → render interruptible
  // (Render bloklashi mumkin, lekin urgent input kelsa to'xtatiladi va keyin davom ettiriladi)
  
  const [isPending, startTransition] = useTransition();
  
  const handleClick = () => {
    setCount(c => c + 1);  // Urgent (default lane — input/event handler)
    
    startTransition(() => {
      setResults(generateExpensiveResults());  // Non-urgent (TransitionLane)
    });
  };
  
  return (
    <>
      <button onClick={handleClick}>Click ({count})</button>
      {isPending && <span>Loading...</span>}
      <List items={results} />
    </>
  );
}
```

`count` urgent — darrov ko'rinadi. `results` non-urgent — yangilanish davomida UI bloklanmaydi.

**Misol 3 — Render restart safety:**

```tsx
// ❌ Render purity buzilishi — restart bug
function Bad() {
  const [count, setCount] = useState(0);
  
  count++;  // ❌ Render paytida mutation
  
  return <div>{count}</div>;
}

// Concurrent restart paytida `count` ikki marta increment bo'lishi mumkin

// ✅ Pure render
function Good() {
  const [count, setCount] = useState(0);
  
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

</details>

---

## `useTransition` API

### Nazariya

`useTransition` — state update'ni "non-urgent" deb belgilash uchun hook.

**Signature:**

```tsx
// React 18
function useTransition(): [
  isPending: boolean,
  startTransition: (scope: () => void) => void
];

// React 19+ — async scope qo'llab-quvvatlanadi
function useTransition(): [
  isPending: boolean,
  startTransition: (scope: () => void | Promise<void>) => void
];
```

Returned tuple:

| Index | Tip | Vazifa |
|-------|-----|--------|
| 0 | `boolean` | `isPending` — transition ishlamoqdami |
| 1 | `(scope) => void` | `startTransition` — scope ichidagi state updates transition'ga (R19'dan async ham) |

**Sodda usage:**

```tsx
function Component() {
  const [isPending, startTransition] = useTransition();
  const [results, setResults] = useState<Item[]>([]);
  
  const handleSearch = (query: string) => {
    startTransition(() => {
      setResults(searchItems(query));  // Non-urgent
    });
  };
  
  return (
    <>
      {isPending && <Spinner />}
      <SearchInput onSearch={handleSearch} />
      <List items={results} />
    </>
  );
}
```

`startTransition` ichidagi state update'lar **TransitionLane**'ga o'zlashtiriladi — DefaultLane'dan past priority. UI urgent updates (typing, click) bilan parallel ishlaydi.

**Lifecycle:**

```
1. handleSearch chaqirildi
2. startTransition(() => setResults(...))
3. setResults TransitionLane'ga schedule qilinadi
4. isPending = true (boshlangan)
5. React render TransitionLane (time-sliced)
6. Render davomida user clicks → SyncLane priority high → render pause
7. SyncLane render + commit
8. Resume TransitionLane render
9. TransitionLane commit
10. isPending = false (tugagan)
```

**`isPending` UI feedback:**

```tsx
return (
  <div>
    <input onChange={(e) => handleSearch(e.target.value)} />
    
    {isPending ? (
      <Spinner />
    ) : (
      <List items={results} />
    )}
    
    {/* Yoki opacity + non-disabled */}
    <div style={{ opacity: isPending ? 0.5 : 1, pointerEvents: isPending ? 'none' : 'auto' }}>
      <List items={results} />
    </div>
  </div>
);
```

User feedback — transition davomida loading state.

**Async transitions (R18 vs R19 farqi):**

```tsx
// ❌ R18: async callback ishlamaydi — `await` keyin transition flag tashqarida,
// `setData` default lane'ga tushadi.
// ✅ R19+: Async transitions qo'llab-quvvatlanadi — React `await` davomida
// transition kontekstini saqlaydi (Actions foundation).
startTransition(async () => {
  const data = await fetch('/api');
  setData(data);  // R19: transition lane | R18: default lane (bug)
});

// ✅ Har ikki versiyada ishlaydigan portable pattern
const data = await fetch('/api');
startTransition(() => {
  setData(data);
});
```

R18'da `startTransition` callback sinxron bo'lishi shart edi (har `await` transition kontekstini yo'qotardi). R19'da bu cheklov olib tashlandi — async actions birinchi-class. **Eslatma:** `setTimeout` orqali yuborilgan state updates baribir transition lane'da bo'lmaydi (callback hech qaysi versiyada saqlanmaydi).

**Multiple updates bir transition'da:**

```tsx
startTransition(() => {
  setQuery(q);
  setFilter(f);
  setPage(1);
  // Hammasi bir transition (TransitionLane), batched
});
```

Bir `startTransition` ichidagi barcha updates bir transition lane'da batched.

<details>
<summary><strong>Under the Hood</strong></summary>

**`useTransition` implementation (soddalashtirilgan, v19.0.0):**

`startTransition` transition flag'ni global `ReactSharedInternals.T`'ga o'rnatadi (R18'da bu joy `ReactCurrentBatchConfig.transition` deb nomlanardi). Flag o'rnatilganida callback ichidagi `setState` chaqiruvlari transition lane'ga o'zlashtiriladi:

```ts
// react/src/ReactStartTransition.js asosida
function startTransition(scope, options) {
  const prevTransition = ReactSharedInternals.T;
  const currentTransition = {};
  ReactSharedInternals.T = currentTransition;  // transition flag set

  try {
    const returnValue = scope();  // callback ichidagi setState'lar — TransitionLane
    // R19: scope async function qaytarsa, React promise'ni kutib turadi (Actions)
    // ...
  } finally {
    ReactSharedInternals.T = prevTransition;  // flag tiklanadi
  }
}
```

`setPending(true)` startTransition ichida emas, `useTransition` hook tomonidan boshqariladi: hook avval `isPending` state'ini sync ravishda `true` qiladi (darrov ko'rinadi), so'ng transition tugaganda `false`'ga qaytaradi.

`ReactSharedInternals.T` null bo'lmagan paytda `dispatchSetState` transition lane so'raydi:

```ts
// react-reconciler/src/ReactFiberHooks.js asosida (soddalashtirilgan)
function dispatchSetState(fiber, queue, action) {
  const lane = requestUpdateLane(fiber);
  // requestUpdateLane ichida: ReactSharedInternals.T !== null bo'lsa →
  //   requestTransitionLane() (TransitionLane), aks holda →
  //   getCurrentUpdatePriority asosida SyncLane / InputContinuousLane / DefaultLane
  scheduleUpdateOnFiber(root, fiber, lane);
}
```

`claimNextTransitionLane` 15 ta TransitionLane'dan birini round-robin tartibida tanlaydi (`nextTransitionLane <<= 1`, oxiriga yetganda `TransitionLane1`'ga qaytadi).

**Source citation:**

- `useTransition` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`
- Transition lanes — facebook/react `packages/react-reconciler/src/ReactFiberLane.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Sodda search:**

```tsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setQuery(value);  // Urgent — input darrov yangilanadi
    
    startTransition(() => {
      setResults(searchItems(value));  // Non-urgent — slow compute
    });
  };
  
  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : <List items={results} />}
    </>
  );
}
```

User type qiladi — `query` darrov ko'rinadi. `results` keyinroq yangilanadi (UI bloklanmaydi).

**Misol 2 — Tab switching:**

```tsx
type Tab = 'home' | 'profile' | 'settings';

function Tabs() {
  const [tab, setTab] = useState<Tab>('home');
  const [isPending, startTransition] = useTransition();
  
  const switchTab = (newTab: Tab) => {
    startTransition(() => {
      setTab(newTab);  // Heavy tab content render — non-urgent
    });
  };
  
  return (
    <div>
      <nav style={{ opacity: isPending ? 0.7 : 1 }}>
        <button onClick={() => switchTab('home')} disabled={isPending}>Home</button>
        <button onClick={() => switchTab('profile')} disabled={isPending}>Profile</button>
        <button onClick={() => switchTab('settings')} disabled={isPending}>Settings</button>
      </nav>
      
      <main>
        {tab === 'home' && <HomeContent />}
        {tab === 'profile' && <ProfileContent />}
        {tab === 'settings' && <SettingsContent />}
      </main>
    </div>
  );
}
```

Tab switch — heavy render (e.g., 1000+ items). User darrov click javobini ko'radi (button highlight, opacity), keyin tab content yangilanadi.

**Misol 3 — Multiple updates batched:**

```tsx
function FilterPanel() {
  const [filters, setFilters] = useState<Filter>({});
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();
  
  const applyFilters = (newFilters: Filter) => {
    startTransition(() => {
      // Hammasi bir transition'da batched
      setFilters(newFilters);
      setResults(applyFilter(allItems, newFilters));
    });
  };
  
  return <>...</>;
}
```

**Misol 4 — Async tashqarida:**

```tsx
function DataLoader({ id }: { id: string }) {
  const [data, setData] = useState<Data | null>(null);
  const [isPending, startTransition] = useTransition();
  
  useEffect(() => {
    const load = async () => {
      const result = await fetch(`/api/${id}`).then(r => r.json());
      
      // Async tashqarida, faqat state update transition'da
      startTransition(() => {
        setData(result);
      });
    };
    
    load();
  }, [id]);
  
  return <>{data ? <Display data={data} /> : <Spinner />}</>;
}
```

</details>

---

## `startTransition` Function

### Nazariya

`startTransition` — `useTransition` qaytaradigan function, lekin hook tashqarida ham ishlatish mumkin (`React.startTransition` import).

**Standalone usage:**

```tsx
import { startTransition } from 'react';

function handleClick() {
  startTransition(() => {
    setState(newValue);  // TransitionLane
  });
}
```

**`useTransition` vs `startTransition`:**

| Aspekt | `useTransition` (hook) | `startTransition` (standalone) |
|--------|------------------------|--------------------------------|
| `isPending` flag | ✅ | ❌ |
| Component ichida ishlash | Faqat | Component yoki tashqari |
| Hook rules | Top-level | Yo'q (hook emas) |

**`isPending` kerak bo'lmasa — standalone:**

```tsx
import { startTransition } from 'react';

function ItemList({ items }: { items: Item[] }) {
  const [filter, setFilter] = useState('');
  
  const handleFilterChange = (newFilter: string) => {
    // isPending kerak emas — UI feedback yo'q
    startTransition(() => {
      setFilter(newFilter);
    });
  };
}
```

UI feedback (spinner) kerak bo'lmasa — `startTransition` standalone foydali.

**Component tashqarida ishlatish:**

```tsx
import { startTransition } from 'react';

// Module-level event handler
window.addEventListener('keydown', (e) => {
  if (e.key === 'Escape') {
    startTransition(() => {
      // State update — komponent state'siga kirish kerak
      // Bu pattern kamdan-kam, lekin imkoni bor
    });
  }
});
```

**`startTransition` chegaralari:**

```tsx
// ⚠️ R18'da async ishlamaydi (await keyin lane lost)
// ✅ R19'da async ishlaydi (Actions support)
startTransition(async () => {
  await fetch('/api');
  setState(data);  // R19: transition | R18: default lane
});

// ❌ setTimeout har ikki versiyada TransitionLane'da emas
startTransition(() => {
  setTimeout(() => {
    setState(value);  // ❌ Transition kontekstidan tashqarida
  }, 100);
});

// ✅ Sync — har versiyada ishlaydi
startTransition(() => {
  setState1(a);
  setState2(b);
  setState3(c);
});
```

R18'da transition flag faqat **sinxron** callback execution davomida active. R19'da React `await` davomida transition'ni internal saqlaydi (async Actions). `setTimeout` callback hech qaysi versiyada saqlanmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Standalone vs hook:**

```ts
// react/src/ReactStartTransition.js asosida (soddalashtirilgan, v19.0.0)
export function startTransition(scope: () => void | Promise<void>) {
  const prevTransition = ReactSharedInternals.T;  // R18: ReactCurrentBatchConfig.transition
  const currentTransition = {};
  ReactSharedInternals.T = currentTransition;

  try {
    const returnValue = scope();  // sync execution
    // R19: scope async bo'lsa, reconciler set qilgan callback (ReactSharedInternals.S)
    // returnValue (promise) bilan chaqiriladi — async action entangle qilinadi
    const onStartTransitionFinish = ReactSharedInternals.S;
    if (onStartTransitionFinish !== null) {
      onStartTransitionFinish(currentTransition, returnValue);
    }
  } finally {
    ReactSharedInternals.T = prevTransition;  // flag sinxron tiklanadi
  }
}

// useTransition standalone'ni isPending state bilan o'raydi
export function useTransition(): [boolean, typeof startTransition] {
  const [isPending, setPending] = useState(false);

  // Real React: `start` Hook chain'da cache qilingan (komponent qaytadan render
  // bo'lsa ham bir xil reference). Soddalashtirilgan misol uchun har render'da
  // yangi qaytarilgandek ko'rsatilgan.
  const start = (scope: () => void) => {
    setPending(true);
    startTransition(() => {
      scope();
      setPending(false);
    });
  };

  return [isPending, start];
}
```

Standalone `startTransition` — base API. `useTransition` `isPending` state qo'shadi va `start` function'ni Hook chain ichida cache qiladi (consumer'lar `useEffect` deps'ga ishlatishi mumkin).

**Async muammosi (R18) va yechim (R19):**

```ts
// R18 — sync-only model
startTransition(async () => {
  await fetch('/api');
  // ReactCurrentBatchConfig.transition allaqachon previous (try/finally)
  // setState — TransitionLane'da emas, default lane'da (bug)
  setState(data);
});
```

R18'da transition flag faqat sinxron execution davomida active edi: `await`'dan keyingi `setState` flag tiklangach ishga tushadi, shuning uchun transition lane'ni yo'qotadi. R19'da React Actions infrastructure'si async transition'ni qo'llab-quvvatlaydi. Mexanizm flag'ni "saqlab turish" emas: `ReactSharedInternals.T` baribir sinxron `finally`'da tiklanadi. Buning o'rniga `startTransition` scope'dan qaytgan promise'ni reconciler set qilgan callback'ga (`ReactSharedInternals.S`) uzatadi; reconciler bu async action'ni entangle qiladi (pending thenable'ni kuzatib turadi) va promise resolve bo'lguncha transition'ni tugatmaydi. Natijada `await`'dan keyingi `setState`'lar ham o'sha transition tarkibida qoladi. Bu — `useActionState`/`<form action>` foundation.

**Source citation:**

- `startTransition` — facebook/react `packages/react/src/ReactStartTransition.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Standalone `startTransition`:**

```tsx
import { startTransition } from 'react';

function SearchInput({ onSearch }: { onSearch: (q: string) => void }) {
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    
    // Search trigger — non-urgent
    startTransition(() => {
      onSearch(value);
    });
  };
  
  return <input onChange={handleChange} />;
}

// Component'da `useTransition` kerak emas — faqat state update transition'da
// `isPending` kerak bo'lsa parent'da `useTransition`
```

**Misol 2 — Hook + standalone hybrid:**

```tsx
import { useTransition, startTransition } from 'react';

function Page() {
  const [isPending, hookStartTransition] = useTransition();
  
  const handleNavigate = (url: string) => {
    // isPending kerak — hook startTransition
    hookStartTransition(() => {
      navigateTo(url);
    });
  };
  
  const handlePrefetch = (url: string) => {
    // isPending kerak emas — standalone
    startTransition(() => {
      prefetchData(url);
    });
  };
  
  return <>{isPending && <Spinner />}...</>;
}
```

**Misol 3 — Async tashqarida pattern:**

```tsx
import { startTransition } from 'react';

async function loadData(id: string) {
  const result = await fetch(`/api/${id}`).then(r => r.json());
  
  // State update transition'da
  startTransition(() => {
    setData(result);
    setLoading(false);
  });
}
```

Async wrap state update'larini bir transition'da.

</details>

---

## `isPending` Flag Pattern

### Nazariya

`isPending` — boolean flag, transition ishlamoqdami ko'rsatadi. UI feedback (spinner, opacity, disable) uchun.

**Pattern variantlari:**

**Variant 1 — Spinner show:**

```tsx
const [isPending, startTransition] = useTransition();

return (
  <>
    {isPending && <Spinner />}
    <Content />
  </>
);
```

Klassik loading indicator.

**Variant 2 — Opacity + disable:**

```tsx
return (
  <div style={{
    opacity: isPending ? 0.5 : 1,
    pointerEvents: isPending ? 'none' : 'auto',
  }}>
    <Content />
  </div>
);
```

Eski content semi-transparent, interaction disabled.

**Variant 3 — Skeleton:**

```tsx
return (
  <>
    {isPending ? <ContentSkeleton /> : <Content data={data} />}
  </>
);
```

Loading skeleton — UX yaxshi.

**Variant 4 — Stale content + indicator:**

```tsx
return (
  <>
    <SmallSpinner visible={isPending} />
    <Content data={data} />  {/* Eski data, transition davomida ko'rinadi */}
  </>
);
```

Eski content qoladi, kichik indicator yangilanish jarayonida.

**`isPending` lifecycle:**

```
0. isPending = false
1. startTransition(() => setState(...))
2. isPending = true (synchronously, after setPending(true) commit)
3. Transition render (time-sliced, interruptible)
4. Transition commit
5. isPending = false
```

`isPending` darrov true (sync), transition tugaganda false.

**Multiple transitions overlap:**

```tsx
const [isPending, startTransition] = useTransition();

const handleClick1 = () => startTransition(() => setState1(...));
const handleClick2 = () => startTransition(() => setState2(...));

// Birinchi click: isPending = true, transition 1 boshlandi
// Ikkinchi click (transition 1 davomida): isPending hali true, transition 2 ham boshlanadi
// Transition 1 va 2 parallel
// Hammasi tugagandan keyin: isPending = false
```

`isPending` umumiy — har transition complete bo'lguncha true.

<details>
<summary><strong>Under the Hood</strong></summary>

**`isPending` — useState bilan:**

```ts
function useTransition() {
  const [isPending, setPending] = useState(false);
  
  const start = (callback) => {
    setPending(true);  // higher priority — darrov visible
    
    startTransition(() => {
      callback();
      setPending(false);  // Transition lane — keyin
    });
  };
  
  return [isPending, start];
}
```

`setPending(true)` — transition tashqarida chaqirilsa default lane'da (urgent). User darrov isPending = true ko'radi.
`setPending(false)` — TransitionLane (transition callback ichida, transition tugagandan keyin commit).

**Strict Mode bilan o'zaro ta'sir:**

Strict Mode (development) komponent funksiyasini 2x chaqiradi (purity test). `isPending` value bir xil bo'lib qoladi — `useState(false)` deterministic, transition callback ham idempotent bo'lishi kerak. Production'da single cycle.

**Source citation:**

- `useTransition` — facebook/react source
- React docs `useTransition` — react.dev/reference/react/useTransition

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Spinner pattern:**

```tsx
function ProductSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Product[]>([]);
  const [isPending, startTransition] = useTransition();
  
  const handleSearch = (q: string) => {
    setQuery(q);
    startTransition(() => {
      setResults(searchProducts(q));
    });
  };
  
  return (
    <div>
      <input value={query} onChange={(e) => handleSearch(e.target.value)} />
      
      {isPending && <div className="spinner">Loading...</div>}
      
      <ul>
        {results.map(p => <li key={p.id}>{p.name}</li>)}
      </ul>
    </div>
  );
}
```

**Misol 2 — Stale content pattern:**

```tsx
function NewsFeed({ filter }: { filter: NewsFilter }) {
  const [articles, setArticles] = useState<Article[]>([]);
  const [isPending, startTransition] = useTransition();
  
  useEffect(() => {
    fetch(`/api/news?${stringify(filter)}`)
      .then(r => r.json())
      .then(data => {
        startTransition(() => {
          setArticles(data);  // Heavy render — transition
        });
      });
  }, [filter]);
  
  return (
    <div>
      {isPending && <small>Updating...</small>}
      
      {/* Eski content qoladi transition davomida */}
      <div style={{ opacity: isPending ? 0.7 : 1 }}>
        {articles.map(a => <Article key={a.id} data={a} />)}
      </div>
    </div>
  );
}
```

**Misol 3 — Disable interaction:**

```tsx
function Wizard() {
  const [step, setStep] = useState(1);
  const [isPending, startTransition] = useTransition();
  
  const goToStep = (n: number) => {
    startTransition(() => setStep(n));
  };
  
  return (
    <div>
      <fieldset disabled={isPending}>
        <button onClick={() => goToStep(1)}>1</button>
        <button onClick={() => goToStep(2)}>2</button>
        <button onClick={() => goToStep(3)}>3</button>
      </fieldset>
      
      <StepContent step={step} />
    </div>
  );
}
```

`fieldset disabled` — transition davomida button'lar disabled.

**Misol 4 — Multiple transitions:**

```tsx
function MultiAction() {
  const [filter, setFilter] = useState('');
  const [sort, setSort] = useState<'asc' | 'desc'>('asc');
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();
  
  const handleFilterChange = (f: string) => {
    setFilter(f);
    startTransition(() => {
      setResults(applyFilterSort(allItems, f, sort));
    });
  };
  
  const handleSortChange = (s: 'asc' | 'desc') => {
    setSort(s);
    startTransition(() => {
      setResults(applyFilterSort(allItems, filter, s));
    });
  };
  
  // isPending — har transition davomida true
  return (
    <div>
      <input onChange={(e) => handleFilterChange(e.target.value)} />
      <button onClick={() => handleSortChange('asc')}>Asc</button>
      <button onClick={() => handleSortChange('desc')}>Desc</button>
      
      {isPending && <Spinner />}
      <List items={results} />
    </div>
  );
}
```

</details>

---

## `useDeferredValue` API

### Nazariya

`useDeferredValue` — value'ni "defer" qiladi (priority pastroq render). `useTransition` state update joyida control beradi, `useDeferredValue` value o'qish joyida.

**Signature:**

```tsx
function useDeferredValue<T>(value: T, initialValue?: T): T;
```

`value` — joriy qiymat. Returned: deferred (eski yoki yangi) qiymat.

**Sodda usage:**

```tsx
function SearchResults({ query }: { query: string }) {
  const deferredQuery = useDeferredValue(query);
  
  // query — fast (har keystroke)
  // deferredQuery — slow (priority pastroq, urgent updates'dan keyin)
  
  const results = useMemo(
    () => searchItems(deferredQuery),
    [deferredQuery]
  );
  
  return <List items={results} />;
}
```

User type qiladi — `query` darrov o'zgaradi. `deferredQuery` "ortda" yangilanadi (urgent renders'dan keyin).

**`useDeferredValue` mexanikasi:**

```
Initial render (mount):
  query = "abc"
  deferredQuery = "abc" (initial render — defer YO'Q, joriy qiymat qaytariladi)

User types "abcd":
Render N+1 (urgent):
  query = "abcd"
  deferredQuery = "abc" (eski qiymat — defer)
  
Render N+2 (deferred, low priority):
  query = "abcd"
  deferredQuery = "abcd" (yangilangan)
```

**Muhim:** Initial render'da `useDeferredValue` defer qilmaydi — joriy `value`'ni qaytaradi (mount paytida ham eski qiymat yo'q). Defer faqat **update**'lar paytida ishlaydi (value o'zgarganda). Keyin React ikki render qiladi — birinchisi urgent (eski deferredQuery), keyingisi non-urgent (yangi deferredQuery).

**Stale value detection:**

```tsx
function Component({ query }: { query: string }) {
  const deferredQuery = useDeferredValue(query);
  
  // Aniqlash — deferred qiymat eski'mi
  const isStale = query !== deferredQuery;
  
  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      <Results query={deferredQuery} />
    </div>
  );
}
```

`query !== deferredQuery` — deferred render hali tugamagan. UI feedback (opacity, spinner) ko'rsatish mumkin.

**`useTransition` bilan farq:**

```tsx
// useTransition — state setter joyida
const [isPending, startTransition] = useTransition();
const handleChange = (q: string) => {
  setQuery(q);  // Urgent
  startTransition(() => {
    setResults(search(q));  // Non-urgent
  });
};

// useDeferredValue — value read joyida
const deferredQuery = useDeferredValue(query);
const results = useMemo(() => search(deferredQuery), [deferredQuery]);
```

`useTransition` — sizda setter bor. `useDeferredValue` — sizda faqat value bor (e.g., props).

**Initial value (R19+):**

```tsx
const deferred = useDeferredValue(value, initialValue);
// initialValue — birinchi render'da qaytariladi (value o'rniga)
// Keyingi render'larda value defer qilinadi (oddiy useDeferredValue mantig'i)
// Foydali: SSR'da initial render uchun deterministic fallback
```

R19'da qo'shilgan ikkinchi argument — initial value (kamdan-kam ishlatiladi, kompleks SSR yoki eager hydration scenarios uchun).

<details>
<summary><strong>Under the Hood</strong></summary>

**`useDeferredValue` implementation (soddalashtirilgan, v19.0.0):**

```ts
// react-reconciler/src/ReactFiberHooks.js — updateDeferredValueImpl asosida
function updateDeferredValueImpl<T>(hook, prevValue: T, value: T): T {
  if (Object.is(value, prevValue)) {
    return value;  // o'zgarmadi — bail out
  }

  // renderLanes urgent bo'lsa (faqat non-urgent lane'lardan iborat EMAS) →
  // eski qiymatni qaytar va keyinroq deferred render rejalashtir
  const shouldDeferValue = !includesOnlyNonUrgentLanes(renderLanes);
  if (shouldDeferValue) {
    const deferredLane = requestDeferredLane();  // DeferredLane
    currentlyRenderingFiber.lanes = mergeLanes(
      currentlyRenderingFiber.lanes,
      deferredLane,
    );
    markSkippedUpdateLanes(deferredLane);
    return prevValue;  // eski qiymat — keyin DeferredLane render'da yangilanadi
  }

  // render allaqachon non-urgent bo'lsa — defer qilish shart emas, yangi qiymat
  hook.memoizedState = value;
  return value;
}
```

`useDeferredValue` `scheduleUpdateOnFiber`'ni o'zi chaqirmaydi. Buning o'rniga fiber'ning `lanes`'iga `DeferredLane` qo'shadi (`mergeLanes`) va `markSkippedUpdateLanes` bilan bu lane'ni skipped deb belgilaydi — natijada React keyin shu fiber uchun deferred render spawn qiladi.

**Lanes:**

`useDeferredValue` deferred render uchun alohida `DeferredLane`'dan foydalanadi (`requestDeferredLane`). Bu — urgent update'lardan past priority. Joriy render allaqachon non-urgent (`includesOnlyNonUrgentLanes`) bo'lsa, defer qilinmaydi — yangi qiymat darrov qaytariladi.

**`useTransition` vs `useDeferredValue` lane:**

`useTransition` setState'larni TransitionLane'ga o'zlashtiradi (`claimNextTransitionLane`), `useDeferredValue` esa value o'zgarganda DeferredLane'da deferred render spawn qiladi. Ikkalasi ham urgent update'lardan past priority — API farq (write vs read), natija o'xshash (low-priority deferred render, interruptible).

**Source citation:**

- `useDeferredValue` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`
- React docs — react.dev/reference/react/useDeferredValue

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Search results:**

```tsx
function App() {
  const [query, setQuery] = useState('');
  
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Results query={query} />
    </>
  );
}

function Results({ query }: { query: string }) {
  const deferredQuery = useDeferredValue(query);
  
  // query — typing'da har keystroke yangilanadi (input lag yo'q)
  // deferredQuery — slow update (search compute heavy)
  
  const results = useMemo(
    () => generateLargeList(deferredQuery),  // Heavy
    [deferredQuery]
  );
  
  const isStale = query !== deferredQuery;
  
  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      <ul>{results.map(r => <li key={r.id}>{r.name}</li>)}</ul>
    </div>
  );
}
```

**Misol 2 — Heavy chart:**

```tsx
function Dashboard({ data }: { data: DataPoint[] }) {
  const deferredData = useDeferredValue(data);
  
  const isStale = data !== deferredData;
  
  return (
    <div>
      {isStale && <small>Rendering chart...</small>}
      <Chart data={deferredData} />  {/* Heavy render */}
    </div>
  );
}
```

`data` props o'zgarsa — chart render slow. `deferredData` defer qiladi → urgent renders bloklanmaydi.

**Misol 3 — `useDeferredValue` + `React.memo`:**

```tsx
const ExpensiveList = React.memo(function ExpensiveList({ items }: { items: Item[] }) {
  // Heavy render
  return <ul>{items.map(i => <Item key={i.id} {...i} />)}</ul>;
});

function Page({ filter }: { filter: string }) {
  const allItems = useMemo(() => loadItems(), []);
  
  // Filter result deferred — urgent render skip
  const filtered = useMemo(
    () => allItems.filter(i => i.name.includes(filter)),
    [allItems, filter]
  );
  
  const deferredFiltered = useDeferredValue(filtered);
  
  return <ExpensiveList items={deferredFiltered} />;
  // ExpensiveList faqat deferredFiltered o'zgarsa re-render
  // Urgent renders'da deferredFiltered eski → skip
}
```

`React.memo` + `useDeferredValue` — heavy component urgent updates'da bloklanmaydi.

**Misol 4 — Stale UI feedback:**

```tsx
function FilterableList({ items, filter }: { items: Item[]; filter: string }) {
  const deferredFilter = useDeferredValue(filter);
  const isStale = filter !== deferredFilter;
  
  const filtered = useMemo(
    () => items.filter(i => i.name.includes(deferredFilter)),
    [items, deferredFilter]
  );
  
  return (
    <div>
      {isStale && <span className="stale-indicator">Updating...</span>}
      
      <div style={{ opacity: isStale ? 0.6 : 1 }}>
        {filtered.map(i => <div key={i.id}>{i.name}</div>)}
      </div>
    </div>
  );
}
```

</details>

---

## `useTransition` vs `useDeferredValue` — Decision

### Nazariya

Ikki hook bir xil maqsad — render priority control. Lekin ishlatish joyi va kontekst farq qiladi.

**Decision matrix:**

| Holat | Tanlov |
|-------|--------|
| State setter sizda | `useTransition` |
| Faqat value (props) sizda | `useDeferredValue` |
| `isPending` UI kerak | `useTransition` |
| Stale value detection (`oldValue !== newValue`) kerak | `useDeferredValue` |
| Multiple state updates bir transition'da | `useTransition` |
| Single value defer | `useDeferredValue` |

**Ekvivalentlik (ba'zi holatlarda):**

```tsx
// useTransition with single state
const [results, setResults] = useState<Item[]>([]);
const [isPending, startTransition] = useTransition();

const handleSearch = (q: string) => {
  startTransition(() => {
    setResults(search(q));
  });
};

// useDeferredValue equivalent (taxminan)
const [query, setQuery] = useState('');
const deferredQuery = useDeferredValue(query);
const results = useMemo(() => search(deferredQuery), [deferredQuery]);
```

Lekin — `useDeferredValue` `useTransition` bilan **bir xil emas**. Implementation o'xshash, lekin use case farq.

**`useTransition` afzal — setter bor:**

```tsx
function App() {
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();
  
  // ✅ useTransition — setter bor, isPending UI feedback
  const handleTabClick = (newTab: string) => {
    startTransition(() => setTab(newTab));
  };
  
  return (
    <>
      {isPending && <Spinner />}
      <Content tab={tab} />
    </>
  );
}
```

**`useDeferredValue` afzal — props orqali value:**

```tsx
function Results({ query }: { query: string }) {
  // ✅ useDeferredValue — query props (setter yo'q)
  const deferredQuery = useDeferredValue(query);
  return <List query={deferredQuery} />;
}

function App() {
  const [query, setQuery] = useState('');
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Results query={query} />  {/* Results setter'siz */}
    </>
  );
}
```

**Birga ishlatish:**

Ikkalasi bir xil app'da, har xil holatlar uchun:

```tsx
function ComplexApp() {
  const [filter, setFilter] = useState('');
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();
  
  // useDeferredValue — filter props orqali Results'ga
  return (
    <>
      <input onChange={(e) => setFilter(e.target.value)} />
      <button onClick={() => startTransition(() => setTab('other'))}>
        Switch
      </button>
      
      {isPending && <Spinner />}
      <Results filter={filter} tab={tab} />
    </>
  );
}

function Results({ filter, tab }: { filter: string; tab: string }) {
  const deferredFilter = useDeferredValue(filter);
  return <FilteredList filter={deferredFilter} tab={tab} />;
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Implementation similarity:**

```ts
// useTransition (soddalashtirilgan)
function useTransition() {
  const [isPending, setPending] = useState(false);

  const start = (callback) => {
    setPending(true);  // urgent — darrov isPending = true
    startTransition(() => {  // ReactSharedInternals.T set → ichidagi setState TransitionLane
      callback();
      setPending(false);  // TransitionLane — transition tugagach
    });
  };

  return [isPending, start];
}

// useDeferredValue (soddalashtirilgan — aniq mexanizm yuqorida "useDeferredValue API" UH'da)
function useDeferredValue(value) {
  const hook = ...;
  const prevValue = hook.memoizedState;

  // Render urgent (faqat non-urgent lane'lardan EMAS) va qiymat o'zgargan bo'lsa:
  // fiber.lanes'ga DeferredLane qo'shiladi + markSkippedUpdateLanes — React keyin
  // shu fiber uchun DeferredLane render spawn qiladi. Hook o'zi update SCHEDULE QILMAYDI.
  if (!includesOnlyNonUrgentLanes(renderLanes) && !Object.is(prevValue, value)) {
    const deferredLane = requestDeferredLane();
    currentlyRenderingFiber.lanes = mergeLanes(currentlyRenderingFiber.lanes, deferredLane);
    markSkippedUpdateLanes(deferredLane);
    return prevValue;  // hozir eski; DeferredLane render'da yangi qiymat qaytadi
  }

  hook.memoizedState = value;
  return value;
}
```

`useTransition` setState'larni TransitionLane'ga o'zlashtiradi, `useDeferredValue` esa fiber lanes'iga DeferredLane qo'shadi (o'zi update schedule qilmaydi). Ikkala holatda ham natija — urgent update'lardan past priority'da deferred render.

**API farq sabab:**

- `useTransition` — transition wrapping (multiple updates bir transition'da). `isPending` flag.
- `useDeferredValue` — single value defer. Stale detection (`oldValue !== newValue` orqali).

**Source citation:**

- React docs Decision Tree — react.dev
- "useTransition vs useDeferredValue" community articles

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — `useTransition` use case:**

```tsx
function TabSwitcher() {
  const [activeTab, setActiveTab] = useState('home');
  const [isPending, startTransition] = useTransition();
  
  // Setter bor + isPending kerak → useTransition
  const handleTabClick = (tab: string) => {
    startTransition(() => setActiveTab(tab));
  };
  
  return (
    <>
      <nav>
        <button onClick={() => handleTabClick('home')} disabled={isPending}>Home</button>
        <button onClick={() => handleTabClick('about')} disabled={isPending}>About</button>
      </nav>
      <TabContent tab={activeTab} />
    </>
  );
}
```

**Misol 2 — `useDeferredValue` use case:**

```tsx
function ProductsList({ searchQuery }: { searchQuery: string }) {
  // Faqat value (props), setter parent'da → useDeferredValue
  const deferredQuery = useDeferredValue(searchQuery);
  
  const results = useMemo(
    () => searchProducts(deferredQuery),
    [deferredQuery]
  );
  
  return <List items={results} />;
}
```

**Misol 3 — Custom hook bilan:**

```tsx
// Custom hook — useDeferredValue qulay
function useFilteredItems<T extends { name: string }>(items: T[], filter: string): T[] {
  const deferredFilter = useDeferredValue(filter);
  return useMemo(
    () => items.filter(item => item.name.includes(deferredFilter)),
    [items, deferredFilter]
  );
}

// Consumer
function Page() {
  const [filter, setFilter] = useState('');
  const filtered = useFilteredItems(allItems, filter);
  
  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      <List items={filtered} />
    </>
  );
}
```

`useDeferredValue` custom hook ichida — clean API.

**Misol 4 — Birga:**

```tsx
function Dashboard() {
  const [filter, setFilter] = useState('');
  const [tab, setTab] = useState<'list' | 'chart'>('list');
  const [isPending, startTransition] = useTransition();
  
  return (
    <div>
      {/* Filter — useDeferredValue (Results props orqali) */}
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      
      {/* Tab switch — useTransition (heavy render isPending feedback) */}
      <nav style={{ opacity: isPending ? 0.7 : 1 }}>
        <button onClick={() => startTransition(() => setTab('list'))}>List</button>
        <button onClick={() => startTransition(() => setTab('chart'))}>Chart</button>
      </nav>
      
      {tab === 'list' ? <Results filter={filter} /> : <Chart filter={filter} />}
    </div>
  );
}

function Results({ filter }: { filter: string }) {
  const deferredFilter = useDeferredValue(filter);
  // ...
}
```

</details>

---

## `useSyncExternalStore` API

### Nazariya

`useSyncExternalStore` (R18+) — external store subscription primitive. Library author'lar uchun (Redux, Zustand, Recoil, etc.). Application code'da kamdan-kam ishlatiladi.

**Signature:**

```tsx
function useSyncExternalStore<T>(
  subscribe: (callback: () => void) => () => void,
  getSnapshot: () => T,
  getServerSnapshot?: () => T
): T;
```

Argumentlar:

| Argument | Tip | Vazifa |
|----------|-----|--------|
| `subscribe` | `(cb) => () => void` | Subscribe function (returns cleanup) |
| `getSnapshot` | `() => T` | Joriy snapshot olish |
| `getServerSnapshot` | `() => T` (optional) | SSR snapshot |

**Sodda usage:**

```tsx
function useWindowWidth(): number {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('resize', callback);
      return () => window.removeEventListener('resize', callback);
    },
    () => window.innerWidth,
    () => 1024  // SSR fallback
  );
}

function Component() {
  const width = useWindowWidth();
  return <div>Width: {width}</div>;
}
```

**Lifecycle:**

```
1. Mount:
   - getSnapshot() → initial value
   - subscribe(callback) → store ga ulanish
   
2. Store change:
   - callback() chaqiriladi
   - React: getSnapshot() qayta chaqirilib snapshot olinadi
   - Object.is(oldSnapshot, newSnapshot) — false bo'lsa re-render
   
3. Unmount:
   - cleanup() (subscribe return)
```

**Snapshot consistency:**

`getSnapshot` har chaqiruvda **bir xil reference** qaytarishi kerak (data o'zgarmasa). Aks holda — har render'da yangi snapshot → infinite re-render:

```tsx
// ❌ Har getSnapshot yangi obyekt
const useStore = () => useSyncExternalStore(
  subscribe,
  () => ({ items: store.items, count: store.count })  // ❌ Har gal yangi {}
);

// ✅ Cached snapshot
const useStore = () => useSyncExternalStore(
  subscribe,
  () => store.snapshot  // ✅ Store ichida cached
);

// ✅ Selector pattern
const useStoreItem = (id: string) => useSyncExternalStore(
  subscribe,
  () => store.items.get(id)  // ✅ Bir xil reference (Map)
);
```

**SSR support — `getServerSnapshot`:**

```tsx
function useWindowWidth() {
  return useSyncExternalStore(
    subscribe,
    () => window.innerWidth,        // Client
    () => 1024                       // Server fallback (window yo'q)
  );
}
```

`getServerSnapshot` — SSR paytida deterministic qiymat. Hydration mismatch oldini olish (cross-ref [`06-hydration.md`](06-hydration.md)).

`getServerSnapshot` ham bir xil reference qaytarish kerak — har gal `() => ({})` har gal yangi → mismatch.

<details>
<summary><strong>Under the Hood</strong></summary>

**`useSyncExternalStore` implementation (soddalashtirilgan — mexanikani tushuntirish uchun):**

```ts
function useSyncExternalStore<T>(
  subscribe,
  getSnapshot,
  getServerSnapshot,
): T {
  // Render paytida snapshot
  const value = getSnapshot();
  
  // SSR detect (real React: server vs client dispatcher orqali — ReactSharedInternals.H)
  if (typeof window === 'undefined') {
    return getServerSnapshot ? getServerSnapshot() : value;
  }
  
  // Subscribe — commit phase'da ulanadi (real React commit hook orqali,
  // useEffect emas — bu pseudocode soddalashtirish)
  useEffect(() => {
    const handleStoreChange = () => {
      // Render restart trigger (real React: internal scheduler API)
      // Public forceUpdate yo'q — pseudocode tushuntirish uchun
      scheduleRerender();
    };
    
    return subscribe(handleStoreChange);
  }, [subscribe]);
  
  return value;
}
```

Real implementation murakkab: render paytida `getSnapshot()` o'qiladi va `pushStoreConsistencyCheck` orqali snapshot consistency check ro'yxatga olinadi; subscribe esa commit phase'da ulanadi (`useEffect` emas, alohida internal effect). Bu pseudocode faqat foydalanish modelini tushuntirish uchun — internal API'lar public emas.

**Tearing prevention:**

Concurrent rendering paytida render bir necha chunk'larga bo'linadi. Chunk'lar orasida store o'zgarsa — bir chunk eski snapshot, boshqa chunk yangi snapshot bilan render qilishi mumkin → **tearing** (UI inconsistency):

```
Component A render: store.value = 1
(store.value = 2 update)
Component B render: store.value = 2
→ A va B teng emas (tearing)
```

Mexanizm: har komponent render'da o'qigan snapshot'ni saqlab qo'yadi (`pushStoreConsistencyCheck`). Commit phase'dan oldin React `checkIfSnapshotChanged` bilan har bir saqlangan snapshot'ni yangi `getSnapshot()` natijasi bilan taqqoslaydi. Agar render va commit orasida store o'zgargan bo'lsa — React `forceStoreRerender` chaqiradi, bu esa **SyncLane**'da (sinxron, interruptible emas) yangi render rejalashtiradi. Sinxron re-render butun daraxtni yangi snapshot bilan bir xil holatda qayta render qiladi — shu tariqa tearing oldi olinadi.

**Source citation:**

- `useSyncExternalStore` RFC #211 — reactjs/rfcs
- React docs — react.dev/reference/react/useSyncExternalStore

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Window size hook:**

```tsx
function useWindowSize() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('resize', callback);
      return () => window.removeEventListener('resize', callback);
    },
    () => ({ width: window.innerWidth, height: window.innerHeight }),
    () => ({ width: 1024, height: 768 })  // SSR fallback
  );
}

// ⚠️ Problem — har getSnapshot yangi obyekt → infinite re-render
// Fix:
let cachedSize = { width: 0, height: 0 };

function useWindowSize() {
  return useSyncExternalStore(
    (callback) => {
      const handler = () => {
        cachedSize = { width: window.innerWidth, height: window.innerHeight };
        callback();
      };
      window.addEventListener('resize', handler);
      handler();  // Initial
      return () => window.removeEventListener('resize', handler);
    },
    () => cachedSize  // Cached reference
  );
}
```

**Misol 2 — Local storage subscription:**

```tsx
function useLocalStorage<T>(key: string, initialValue: T): T {
  const subscribe = useCallback(
    (callback: () => void) => {
      window.addEventListener('storage', callback);
      return () => window.removeEventListener('storage', callback);
    },
    []
  );
  
  const getSnapshot = useCallback(() => {
    const item = localStorage.getItem(key);
    return item ? JSON.parse(item) as T : initialValue;
  }, [key, initialValue]);
  
  return useSyncExternalStore(
    subscribe,
    getSnapshot,
    () => initialValue  // SSR
  );
}

// ⚠️ getSnapshot har gal yangi obyekt parse → infinite re-render
// Fix: cache parsed value
```

**Misol 3 — Custom store (Redux-like):**

```tsx
type Store<T> = {
  getState: () => T;
  subscribe: (cb: () => void) => () => void;
};

function createStore<T>(initial: T): Store<T> & { setState: (next: T) => void } {
  let state = initial;
  const subscribers = new Set<() => void>();
  
  return {
    getState: () => state,
    setState: (next: T) => {
      state = next;
      subscribers.forEach(cb => cb());
    },
    subscribe: (cb) => {
      subscribers.add(cb);
      return () => subscribers.delete(cb);
    },
  };
}

const counterStore = createStore({ count: 0 });

function useCounter() {
  return useSyncExternalStore(
    counterStore.subscribe,
    counterStore.getState
  );
}

function Counter() {
  const { count } = useCounter();
  return (
    <button onClick={() => counterStore.setState({ count: count + 1 })}>
      {count}
    </button>
  );
}
```

**Misol 4 — Online/offline status:**

```tsx
function useOnlineStatus(): boolean {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('online', callback);
      window.addEventListener('offline', callback);
      return () => {
        window.removeEventListener('online', callback);
        window.removeEventListener('offline', callback);
      };
    },
    () => navigator.onLine,
    () => true  // SSR — assume online
  );
}

function NetworkStatus() {
  const isOnline = useOnlineStatus();
  return <span>{isOnline ? '🟢 Online' : '🔴 Offline'}</span>;
}
```

**Misol 5 — Selector pattern:**

```tsx
const store = createStore({ items: [] as Item[], filter: '' });

function useStoreSelector<T>(selector: (state: typeof store extends Store<infer S> ? S : never) => T): T {
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState())
  );
}

function ItemCount() {
  // Faqat items.length subscribe — boshqa state o'zgarsa skip
  const count = useStoreSelector(state => state.items.length);
  return <span>{count}</span>;
}
```

⚠️ Selector pattern — har gal selector yangi qiymat bo'lsa re-render. Reference equality kerak (Object.is). Object/array selector — `useMemo` yoki Reselect-style libraries.

</details>

---

## Tearing Prevention — Concurrent rendering

### Nazariya

**Tearing** — Concurrent rendering paytida external state o'zgarsa, bir komponent'lar eski snapshot, boshqalari yangi snapshot bilan render qilishi mumkin → UI inconsistency.

**Tearing scenariy:**

```tsx
let externalState = { count: 0 };
const subscribers = new Set<() => void>();

function increment() {
  externalState = { count: externalState.count + 1 };
  subscribers.forEach(cb => cb());
}

// ❌ useState + useEffect (tearing-prone)
function CounterA() {
  const [count, setCount] = useState(externalState.count);
  
  useEffect(() => {
    const cb = () => setCount(externalState.count);
    subscribers.add(cb);
    return () => subscribers.delete(cb);
  }, []);
  
  return <span>A: {count}</span>;
}

function CounterB() {
  const [count, setCount] = useState(externalState.count);
  
  useEffect(() => {
    const cb = () => setCount(externalState.count);
    subscribers.add(cb);
    return () => subscribers.delete(cb);
  }, []);
  
  return <span>B: {count}</span>;
}

// Concurrent rendering scenariy:
// Render boshlandi:
//   CounterA render: count=5 (eski state)
//   (increment chaqirildi externally → externalState.count = 6)
//   CounterB render: count=6 (yangi state)
// → UI: A=5, B=6 (TEARING!)
```

**`useSyncExternalStore` yechim:**

```tsx
function useExternalCount(): number {
  return useSyncExternalStore(
    (cb) => {
      subscribers.add(cb);
      return () => subscribers.delete(cb);
    },
    () => externalState.count
  );
}

function CounterA() {
  const count = useExternalCount();
  return <span>A: {count}</span>;
}

function CounterB() {
  const count = useExternalCount();
  return <span>B: {count}</span>;
}

// Concurrent rendering:
//   Render boshlandi:
//   CounterA render: count=5, snapshot saqlandi (consistency check)
//   (increment → store o'zgardi)
//   CounterB render: count=... (yangi getSnapshot)
//   Commit'dan oldin React consistency check: render'dagi snapshot != joriy snapshot
//   → forceStoreRerender (SyncLane, sinxron re-render)
//   Sinxron re-render: CounterA=6, CounterB=6
// → UI: A=6, B=6 (consistent, NO tearing)
```

`useSyncExternalStore` snapshot consistency'ni kafolatlaydi — render va commit orasida store o'zgarsa, React SyncLane'da sinxron re-render qiladi (butun daraxt yangi snapshot bilan bir xil holatda).

**Why `useSyncExternalStore` library author'lar uchun:**

Application code odatda `useState`/`useReducer` ishlatadi (React-internal state, tearing yo'q). External store — `useSyncExternalStore` zaruriy.

Modern state library'lar `useSyncExternalStore` ishlatadi:

- **Redux** (R18+) — `react-redux` v8+
- **Zustand** — built-in
- **Jotai** — built-in
- **MobX** — `mobx-react-lite` v3.4+
- **Recoil** — built-in

Application code shu library'lar API'sidan foydalanadi.

**External store + `useState` + manual subscription — tearing risk:**

```tsx
// ❌ External store uchun R18 Concurrent + manual subscription = tearing potential
function useStore() {
  const [value, setValue] = useState(store.getState());
  
  useEffect(() => {
    return store.subscribe(() => setValue(store.getState()));
  }, []);
  
  return value;
}

// ✅ useSyncExternalStore — Concurrent-safe
function useStore() {
  return useSyncExternalStore(store.subscribe, store.getState);
}
```

External store (Redux/Zustand/custom) — `useSyncExternalStore` migration tearing prevention beradi. **Eslatma:** Komponent o'zining `useState`/`useReducer` (React-internal) state'i tearing'dan immune — chunki React state har gal joriy render snapshot bilan birga ishlaydi (synchronous, render coherent). Tearing muammosi faqat **external mutation manbai** (React'dan tashqari store) bilan.

<details>
<summary><strong>Under the Hood</strong></summary>

**Snapshot consistency check:**

```ts
// react-reconciler/src/ReactFiberHooks.js asosida (soddalashtirilgan, v19.0.0)
function updateSyncExternalStore(subscribe, getSnapshot) {
  const nextSnapshot = getSnapshot();  // render paytida o'qiladi

  // render paytidagi snapshot consistency check'ga ro'yxatga olinadi
  pushStoreConsistencyCheck(fiber, getSnapshot, nextSnapshot);

  // commit phase'da (yoki passive effect'da):
  //   checkIfSnapshotChanged(inst): Object.is(inst.value, getSnapshot()) === false bo'lsa
  //   → forceStoreRerender(fiber) → enqueueConcurrentRenderForLane(fiber, SyncLane)
  //   → scheduleUpdateOnFiber(root, fiber, SyncLane)  // sinxron re-render

  return nextSnapshot;
}
```

React render davomida o'qigan snapshot'ni saqlaydi (`pushStoreConsistencyCheck`), commit'dan oldin uni qayta `getSnapshot()` natijasi bilan taqqoslaydi (`checkIfSnapshotChanged`). Mos kelmasa — `forceStoreRerender` SyncLane'da sinxron re-render rejalashtiradi. Snapshot'ni butun render bo'yi "muzlatib" qo'yish emas — render'dan keyingi consistency check.

**`useSyncExternalStore` SSR `getServerSnapshot`:**

SSR'da `window` yo'q, store deterministic bo'lishi shart:

```ts
function useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot) {
  if (typeof window === 'undefined') {
    if (!getServerSnapshot) {
      throw new Error('Missing getServerSnapshot — required for SSR');
    }
    return getServerSnapshot();
  }
  // ... client logic
}
```

`getServerSnapshot` server'da chaqiriladi. Hydration paytida — client `getSnapshot` bilan compare. Mismatch → hydration warning.

**Source citation:**

- React 18 release notes — Concurrent rendering tearing
- `useSyncExternalStore` RFC tearing examples

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Tearing demonstration:**

```tsx
// ❌ Tearing-prone
let counter = 0;
const subs = new Set<() => void>();

function CounterDisplay() {
  const [value, setValue] = useState(counter);
  
  useEffect(() => {
    const cb = () => setValue(counter);
    subs.add(cb);
    return () => subs.delete(cb);
  }, []);
  
  return <span>{value}</span>;
}

// Multiple instances + concurrent update + interrupt → tearing

// ✅ Tearing-safe
function CounterDisplay() {
  const value = useSyncExternalStore(
    (cb) => { subs.add(cb); return () => subs.delete(cb); },
    () => counter
  );
  return <span>{value}</span>;
}
```

**Misol 2 — Real library pattern:**

```tsx
// Hypothetical mini-store
function createStore<T>(initial: T) {
  let state = initial;
  const subs = new Set<() => void>();
  
  return {
    getState: () => state,
    setState: (next: T) => {
      state = next;
      subs.forEach(cb => cb());
    },
    subscribe: (cb: () => void) => {
      subs.add(cb);
      return () => subs.delete(cb);
    },
  };
}

const store = createStore({ count: 0 });

// Hook — useSyncExternalStore
function useStore() {
  return useSyncExternalStore(store.subscribe, store.getState);
}

// Multiple components — consistent
function CounterA() {
  const { count } = useStore();
  return <span>A: {count}</span>;
}

function CounterB() {
  const { count } = useStore();
  return <span>B: {count}</span>;
}

function App() {
  return (
    <>
      <CounterA />
      <CounterB />
      <button onClick={() => store.setState({ count: store.getState().count + 1 })}>+</button>
    </>
  );
}
// A va B doim bir xil count
```

**Misol 3 — Selector with reference equality:**

```tsx
const store = createStore({ items: [] as Item[], filter: '' });

function useStoreItems(): Item[] {
  return useSyncExternalStore(
    store.subscribe,
    () => store.getState().items  // Bir xil reference (filter o'zgarsa items o'zgarmaydi)
  );
}

function useStoreFilter(): string {
  return useSyncExternalStore(
    store.subscribe,
    () => store.getState().filter
  );
}

// items o'zgarsa — useStoreItems consumer re-render
// filter o'zgarsa — useStoreFilter consumer re-render
// Crossover yo'q — selector reference equality
```

</details>

---

## `useId` API

### Nazariya

`useId` (R18+) — komponent uchun **deterministic unique ID** generate qiluvchi hook. SSR-safe (server va client bir xil ID), accessibility (ARIA) uchun mo'ljallangan.

**Signature:**

```tsx
function useId(): string;
```

Returned: unique ID string (e.g., `:r0:`, `:r1:`).

**Sodda usage:**

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
```

`useId` ID'ni Fiber tree path asosida generate qiladi — server va client bir xil ID → hydration mismatch yo'q.

**Multiple IDs single hook:**

```tsx
function FormGroup() {
  const id = useId();
  
  return (
    <fieldset>
      <label htmlFor={`${id}-name`}>Name</label>
      <input id={`${id}-name`} />
      
      <label htmlFor={`${id}-email`}>Email</label>
      <input id={`${id}-email`} />
    </fieldset>
  );
}
```

Bir `useId` chaqiruvi — base ID. Suffix bilan multiple IDs.

**`useId` formatti:**

ID format React versiya'siga bog'liq, lekin tipik shakl `:` prefix/suffix bilan unikal pattern (masalan `:r0:`, `:r1:`, `:R1:`). Tafsilotlar implementation detail — foydalanuvchi format'ga bevosita tayanmasligi kerak (faqat string sifatida ishlatish).

`:` character — HTML ID'da valid (HTML5 spec). CSS selector'da `:` pseudo-class belgisi, shu sababli ID'ni CSS'da tanlash uchun escape qilinishi kerak (`#\:r0\:`). React'ning bu pattern foydalanuvchi tomonidan qo'lda yoziladigan ID'lar bilan to'qnashmasligini ta'minlaydi.

**`useId` aksessuar uchun:**

```tsx
// Label association
<label htmlFor={id}>...</label>
<input id={id} />

// ARIA describedby
<input aria-describedby={id} />
<div id={id}>Help text</div>

// ARIA labelledby
<div id={id}>Heading</div>
<section aria-labelledby={id}>...</section>
```

**Anti-pattern — `useId` keys uchun:**

```tsx
// ❌ Keys uchun ishlatmaslik
function List({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => {
        const id = useId();  // ❌ Rules of Hooks buzilishi: hook .map() callback ichida
        return <li key={id}>{item.name}</li>;
      })}
    </ul>
  );
}

// ✅ Keys — item.id (data'dan)
function List({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

`useId` — UI element IDs uchun, list keys uchun emas.

<details>
<summary><strong>Under the Hood</strong></summary>

**`useId` ID generation:**

```ts
// react-reconciler/src/ReactFiberHooks.js — mountId asosida (soddalashtirilgan, v19.0.0)
function mountId(): string {
  const hook = mountWorkInProgressHook();
  const identifierPrefix = root.identifierPrefix;
  let id;

  if (getIsHydrating()) {
    // Hydration/server path: Fiber tree path asosida deterministic
    const treeId = getTreeId();  // ReactFiberTreeContext — daraxtdagi pozitsiya
    id = ':' + identifierPrefix + 'R' + treeId + ':';  // uppercase R
  } else {
    // Client-only mount: global counter
    id = ':' + identifierPrefix + 'r' + globalClientId.toString(32) + ':';  // lowercase r
  }

  hook.memoizedState = id;
  return id;
}
```

Hydration path'da (`R` prefix) ID Fiber tree pozitsiyasidan generatsiya qilinadi (`getTreeId`) — server va client bir xil pozitsiya → bir xil ID. Client-only mount'da (`r` prefix) global counter ishlatiladi.

**Why `:` characters:**

`:` HTML5 spec bo'yicha valid ID character. CSS selector'da pseudo-class belgisi (`:hover`), shuning uchun HTML ID'ni CSS'da tanlash uchun escape qilinishi kerak (`#\:r0\:` yoki `[id=":r0:"]`). React'ning bu pattern foydalanuvchi qo'lda yozadigan ID'lar bilan to'qnashmaslikni ta'minlaydi (foydalanuvchi ID'lari odatda `:` bilan boshlanmaydi).

**Hydration:**

```
Server render:
  <FormField /> at tree position X → useId returns ID_X (e.g., ':r0:')
  HTML: <label htmlFor=":r0:">...</label>

Client hydrate:
  <FormField /> at tree position X → useId returns same ID_X (':r0:')
  Match → no warning
```

Tree position bir xil → ID bir xil → hydration safe. ID formati implementation detail — foydalanuvchi konkret format'ga bog'lanmasligi kerak.

**Strict Mode:**

R18+ Strict Mode'da `useId` ham 2x render bilan stabil — bir xil ID qaytariladi (idempotent).

**Source citation:**

- `useId` RFC — reactjs/rfcs
- React docs — react.dev/reference/react/useId

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Form field with label:**

```tsx
function NameField() {
  const id = useId();
  
  return (
    <div>
      <label htmlFor={id}>Name:</label>
      <input id={id} type="text" />
    </div>
  );
}
```

**Misol 2 — ARIA describedby:**

```tsx
function PasswordField() {
  const inputId = useId();
  const helpId = useId();
  
  return (
    <div>
      <label htmlFor={inputId}>Password:</label>
      <input
        id={inputId}
        type="password"
        aria-describedby={helpId}
      />
      <div id={helpId}>
        Password must be at least 8 characters.
      </div>
    </div>
  );
}
```

**Misol 3 — Multiple fields single useId:**

```tsx
function ContactForm() {
  const id = useId();
  
  return (
    <fieldset>
      <legend id={`${id}-legend`}>Contact Information</legend>
      
      <label htmlFor={`${id}-name`}>Name:</label>
      <input id={`${id}-name`} />
      
      <label htmlFor={`${id}-email`}>Email:</label>
      <input id={`${id}-email`} />
      
      <label htmlFor={`${id}-phone`}>Phone:</label>
      <input id={`${id}-phone`} />
    </fieldset>
  );
}
```

Bir hook chaqiruvi — multiple IDs (suffix bilan).

**Misol 4 — Tabs:**

```tsx
function Tabs({ items }: { items: { label: string; content: React.ReactNode }[] }) {
  const baseId = useId();
  const [activeIndex, setActiveIndex] = useState(0);
  
  return (
    <div>
      <div role="tablist">
        {items.map((item, i) => (
          <button
            key={i}
            id={`${baseId}-tab-${i}`}
            role="tab"
            aria-controls={`${baseId}-panel-${i}`}
            aria-selected={i === activeIndex}
            onClick={() => setActiveIndex(i)}
          >
            {item.label}
          </button>
        ))}
      </div>
      
      {items.map((item, i) => (
        <div
          key={i}
          id={`${baseId}-panel-${i}`}
          role="tabpanel"
          aria-labelledby={`${baseId}-tab-${i}`}
          hidden={i !== activeIndex}
        >
          {item.content}
        </div>
      ))}
    </div>
  );
}
```

**Misol 5 — Custom hook with useId:**

```tsx
function useFormField(): {
  inputId: string;
  errorId: string;
  inputProps: { id: string; 'aria-describedby': string };
  errorProps: { id: string };
} {
  const inputId = useId();
  const errorId = useId();
  
  return {
    inputId,
    errorId,
    inputProps: { id: inputId, 'aria-describedby': errorId },
    errorProps: { id: errorId },
  };
}

function EmailField({ error }: { error?: string }) {
  const { inputProps, errorProps } = useFormField();
  
  return (
    <>
      <input type="email" {...inputProps} />
      {error && <span {...errorProps}>{error}</span>}
    </>
  );
}
```

</details>

---

## Hydration Mismatch Prevention

### Nazariya

SSR'da server va client farq qiluvchi qiymatlar — hydration mismatch (cross-ref [`06-hydration.md`](06-hydration.md)). `useId` bu muammoni hal qiladi (deterministic IDs uchun).

**Hydration mismatch sabablari:**

1. **`Math.random()`** — har chaqiruvda farqli
2. **`Date.now()`** — server va client vaqt
3. **`window.*`** — server'da yo'q
4. **Random IDs** — server va client farq

```tsx
// ❌ Hydration mismatch
function BadField() {
  const id = `field-${Math.random()}`;  // ❌ Server: 0.123, Client: 0.456
  return <input id={id} />;
}

// Server HTML: <input id="field-0.123">
// Client render: <input id="field-0.456">
// Hydration mismatch warning
```

**`useId` yechim:**

```tsx
function GoodField() {
  const id = useId();  // ✅ Server: ':r0:', Client: ':r0:'
  return <input id={id} />;
}
```

**Boshqa hydration mismatch oldini olish pattern'lari:**

**Pattern 1 — `useEffect` bilan client-only state:**

```tsx
function ClientOnlyValue() {
  const [width, setWidth] = useState(1024);  // SSR fallback
  
  useEffect(() => {
    setWidth(window.innerWidth);  // Client'da update
  }, []);
  
  return <div>Width: {width}</div>;
}

// Server render: <div>Width: 1024</div>
// Client render (initial): <div>Width: 1024</div> (no mismatch)
// Effect runs: setWidth(actualWidth) → re-render
```

**Pattern 2 — `suppressHydrationWarning`:**

```tsx
<div suppressHydrationWarning>
  Time: {new Date().toISOString()}
</div>
```

Aniq nuqta hydration warning'ni o'chiradi. Anti-pattern global ishlatish (cross-ref [`06-hydration.md`](06-hydration.md)).

**Pattern 3 — `useSyncExternalStore` `getServerSnapshot`:**

```tsx
function useWindowWidth() {
  return useSyncExternalStore(
    (cb) => { window.addEventListener('resize', cb); return () => window.removeEventListener('resize', cb); },
    () => window.innerWidth,
    () => 1024  // SSR deterministic
  );
}
```

`getServerSnapshot` SSR uchun deterministic value.

**Pattern 4 — `useId` keys uchun emas:**

```tsx
// ❌ useId list keys uchun
function List({ items }: { items: string[] }) {
  return (
    <ul>
      {items.map(item => {
        const id = useId();  // ❌ Rules of Hooks buzilishi (.map() callback komponent body emas)
        return <li key={id}>{item}</li>;
      })}
    </ul>
  );
}

// ✅ Stable keys data'dan
function List({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map(item => <li key={item.id}>{item.name}</li>)}
    </ul>
  );
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`useId` Fiber path:**

```
Root
├─ Header
│  └─ Logo (useId → ':r0:')
├─ Main
│  └─ Form
│     └─ FormField (useId → ':r1:')
│     └─ FormField (useId → ':r2:')
└─ Footer
```

Tree position deterministic — server va client bir xil tree → bir xil IDs.

**Conditional render trapi:**

```tsx
function App({ showHeader }: { showHeader: boolean }) {
  return (
    <>
      {showHeader && <Header />}  {/* Conditional */}
      <Main />  {/* useId chaqiriladi */}
    </>
  );
}

// showHeader=true: Main useId → ':r1:' (Header birinchi)
// showHeader=false: Main useId → ':r0:' (Header yo'q)
// Server va client farq qilsa → mismatch
```

Conditional rendering — tree position farq → ID farq. Server va client bir xil shartlar bilan render qilish kerak.

**Source citation:**

- React docs `useId` — react.dev/reference/react/useId
- Hydration mismatch — react.dev/reference/react-dom/client/hydrateRoot

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — `useId` SSR-safe:**

```tsx
// app/Form.tsx
function Form() {
  const nameId = useId();
  const emailId = useId();
  
  return (
    <form>
      <label htmlFor={nameId}>Name</label>
      <input id={nameId} />
      
      <label htmlFor={emailId}>Email</label>
      <input id={emailId} type="email" />
    </form>
  );
}

// Server: <input id=":r0:"> ... <input id=":r1:">
// Client: <input id=":r0:"> ... <input id=":r1:">
// Match — no hydration warning
```

**Misol 2 — `Math.random` anti-pattern fix:**

```tsx
// ❌ Mismatch
function RandomField() {
  const [id] = useState(`field-${Math.random()}`);  // Server: random1, Client: random2
  return <input id={id} />;
}

// ✅ useId
function StableField() {
  const id = useId();
  return <input id={id} />;
}
```

**Misol 3 — Date.now mismatch fix:**

```tsx
// ❌ Mismatch — server time vs client time
function Timestamp() {
  return <span>Loaded at: {Date.now()}</span>;
}

// ✅ Client-only useEffect
function Timestamp() {
  const [time, setTime] = useState<number | null>(null);
  
  useEffect(() => {
    setTime(Date.now());
  }, []);
  
  return <span>{time !== null ? `Loaded at: ${time}` : 'Loading...'}</span>;
}

// ✅ suppressHydrationWarning (specific case)
function Timestamp() {
  return <span suppressHydrationWarning>Loaded at: {Date.now()}</span>;
}
```

**Misol 4 — `useSyncExternalStore` for browser API:**

```tsx
// getSnapshot bir xil reference qaytarishi kerak — aks holda infinite re-render.
// Shuning uchun viewport cache qilinadi, faqat resize'da yangilanadi.
const serverViewport = { width: 1024, height: 768 };
let cachedViewport = serverViewport;

function useViewport() {
  return useSyncExternalStore(
    (cb) => {
      const handler = () => {
        cachedViewport = { width: window.innerWidth, height: window.innerHeight };
        cb();
      };
      handler();  // initial client value
      window.addEventListener('resize', handler);
      return () => window.removeEventListener('resize', handler);
    },
    () => cachedViewport,        // cached reference (client)
    () => serverViewport,        // SSR deterministic, barqaror reference
  );
}
```

</details>

---

## Decision Tree — Qaysi Hook Qachon

### Nazariya

Concurrent hooks tanlash uchun decision tree:

```
1. State setter sizda + isPending UI feedback kerak?
   → useTransition

2. Faqat value (props) defer qilish kerak?
   → useDeferredValue

3. External store subscription (Redux/Zustand/custom)?
   → useSyncExternalStore (library author bo'lsangiz)
   → Library bilan ishlasangiz — useSelector / useStore (library API)

4. Unique ID kerak (label, ARIA)?
   → useId

5. Hech qaysi mos kelmasa?
   → useState / useEffect (sodda case'lar)
```

**Qachon Concurrent hook kerak emas:**

- Sodda counter, toggle → `useState`
- One-time setup → `useEffect`
- Memoization → `useMemo` / `useCallback`
- Reducer pattern → `useReducer`

Concurrent hooks — **performance/concurrency edge case**'lar uchun. Boshqa joyda overkill.

**Real-world tanlov:**

**Search input — `useDeferredValue`:**

```tsx
function App() {
  const [query, setQuery] = useState('');
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Results query={query} />
    </>
  );
}

function Results({ query }: { query: string }) {
  const deferred = useDeferredValue(query);  // ✅
  // Setter parent'da, faqat value props
}
```

**Tab switch — `useTransition`:**

```tsx
function Tabs() {
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();  // ✅
  
  // Setter local + isPending UI feedback
}
```

**Library state — `useSyncExternalStore`:**

```tsx
// Library code (Redux, Zustand)
function useStore<T>(selector: (s: State) => T): T {
  return useSyncExternalStore(...);
}

// App code — library API
const value = useStore(state => state.value);
```

**Form IDs — `useId`:**

```tsx
function FormField() {
  const id = useId();  // ✅
  return <><label htmlFor={id} /><input id={id} /></>;
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Concurrent hooks performance:**

- `useTransition` — TransitionLane scheduling overhead
- `useDeferredValue` — DeferredLane scheduling
- `useSyncExternalStore` — snapshot consistency check
- `useId` — Fiber tree path traversal

Har biri — minimal overhead. Faqat kerak bo'lgan joyda ishlatish.

**Source citation:**

- React docs Concurrent hooks — react.dev
- React 18 release notes

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Decision flow:**

```tsx
// Use case: Search input + heavy results render

// Variant A — useTransition (state setter joyida)
function Search() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value);
    startTransition(() => {
      setResults(searchItems(e.target.value));
    });
  };
  
  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <List items={results} />
    </>
  );
}

// Variant B — useDeferredValue (props orqali)
function Search() {
  const [query, setQuery] = useState('');
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Results query={query} />
    </>
  );
}

function Results({ query }: { query: string }) {
  const deferred = useDeferredValue(query);
  const results = useMemo(() => searchItems(deferred), [deferred]);
  return <List items={results} />;
}
```

Ikkalasi — bir xil natija. Tanlov: setter joyiga qaratilgan vs value abstraction.

**Misol 2 — Multiple hooks together:**

```tsx
function Dashboard() {
  // useId — labels uchun
  const filterId = useId();
  
  // useTransition — heavy operations
  const [isPending, startTransition] = useTransition();
  
  // useState — sodda
  const [filter, setFilter] = useState('');
  const [tab, setTab] = useState('list');
  
  return (
    <div>
      <label htmlFor={filterId}>Filter:</label>
      <input
        id={filterId}
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
      />
      
      <button onClick={() => startTransition(() => setTab('chart'))}>
        Chart
      </button>
      
      {isPending && <Spinner />}
      
      <Results filter={filter} tab={tab} />
    </div>
  );
}

function Results({ filter, tab }: { filter: string; tab: string }) {
  // useDeferredValue — filter props orqali
  const deferredFilter = useDeferredValue(filter);
  
  return tab === 'list'
    ? <List filter={deferredFilter} />
    : <Chart filter={deferredFilter} />;
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1 — `startTransition` async R18 vs R19

```tsx
// ⚠️ R18'da: await keyin transition flag tashqarida → setState default lane'ga
// ✅ R19'da: async scope native qo'llab-quvvatlanadi (Actions)
startTransition(async () => {
  await fetch('/api');
  setState(data);  // R19: transition | R18: default lane
});

// ✅ Portable pattern — har ikki versiyada ishlaydi
const data = await fetch('/api');
startTransition(() => {
  setState(data);
});
```

### Gotcha 2 — `useSyncExternalStore` snapshot identity

```tsx
// ❌ Har getSnapshot yangi obyekt → infinite re-render
const value = useSyncExternalStore(
  subscribe,
  () => ({ count: store.count })  // ❌ Yangi {}
);

// ✅ Cached snapshot
const value = useSyncExternalStore(
  subscribe,
  () => store.snapshot  // Cached
);
```

### Gotcha 3 — `useDeferredValue` initial render

```tsx
function Results({ query }: { query: string }) {
  const deferred = useDeferredValue(query);
  
  // Initial render: deferred === query (sync)
  // Update render: deferred eski qiymat (defer)
}
```

Birinchi render'da deferred === value (sync). Keyingi render'larda defer.

### Gotcha 4 — `useId` uniqueness scope

```tsx
// useId — komponent instance bo'yicha unique
function Field() {
  const id = useId();
  return <input id={id} />;
}

function App() {
  return (
    <>
      <Field />  {/* id = ':r0:' */}
      <Field />  {/* id = ':r1:' */}
      <Field />  {/* id = ':r2:' */}
    </>
  );
}
// Har instance unique
```

### Gotcha 5 — `useTransition` urgent updates parallel

```tsx
const [isPending, startTransition] = useTransition();

const handleSearch = (q: string) => {
  setQuery(q);  // Urgent — render immediately
  
  startTransition(() => {
    setResults(search(q));  // Non-urgent — render later
  });
};

// Render 1 (urgent): query updated, results eski
// Render 2 (transition): results yangilangan
// User darrov input ko'radi, results keyinroq
```

Urgent + non-urgent parallel — UX yaxshi.

---

## Common Mistakes

### ❌ Xato 1 — R18 codebase'da `startTransition` async callback

```tsx
// ⚠️ R18 — async scope ishlamaydi (await keyin transition lane lost)
// ✅ R19 — async scope native, transition kontekstida await
startTransition(async () => {
  const data = await fetch('/api');
  setState(data);  // R19: transition | R18: default lane (bug)
});

// ✅ Portable — sync wrap, async tashqarida
const data = await fetch('/api');
startTransition(() => setState(data));
```

### ❌ Xato 2 — `useSyncExternalStore` har gal yangi snapshot

```tsx
// ❌ Infinite loop
const value = useSyncExternalStore(
  subscribe,
  () => store.items.map(transform)  // ❌ Har gal yangi array
);

// ✅ Manba o'zgarmasa cache'langan natijani qaytaradi
let cachedSource: Item[] | null = null;
let cachedResult: TransformedItem[] = [];

function getTransformedItems() {
  if (cachedSource !== store.items) {
    cachedSource = store.items;
    cachedResult = store.items.map(transform);  // faqat manba o'zgarganda qayta hisoblanadi
  }
  return cachedResult;  // bir xil reference (Object.is barqaror)
}

const value = useSyncExternalStore(subscribe, getTransformedItems);
```

### ❌ Xato 3 — `useId` keys uchun

```tsx
// ❌ Keys uchun
{items.map(() => {
  const id = useId();  // ❌ Rules of Hooks buzilishi (.map() callback)
  return <li key={id} />;
})}

// ✅ Stable keys
{items.map(item => <li key={item.id} />)}
```

### ❌ Xato 4 — `useTransition` setState tashqarida

```tsx
// ❌ Side effect transition scope ichida
startTransition(() => {
  document.title = 'New';  // ❌ scope sinxron ishlaydi — DOM mutation darrov bo'ladi,
                           //    transition commit'iga bog'lanmaydi (defer qilinmaydi)
  setState(value);
});

// ✅ Faqat state update — side effect tashqarida
document.title = 'New';
startTransition(() => setState(value));
```

`startTransition` scope'i sinxron, bir marta ishga tushadi: faqat ichidagi `setState` chaqiruvlari TransitionLane'ga o'zlashtiriladi. `document.title` kabi side effect darrov bajariladi va transition commit'i bilan muvofiqlashtirilmaydi — agar transition interrupt bo'lib commit qilinmasa ham, DOM mutation allaqachon yuz bergan bo'ladi. Side effect'ni scope tashqarisiga (yoki commit bilan bog'lash uchun `useEffect`'ga) qo'ying.

### ❌ Xato 5 — `useDeferredValue` setter o'rniga

```tsx
// ❌ Setter sizda — useTransition kerak
function Component() {
  const [query, setQuery] = useState('');
  const deferred = useDeferredValue(query);
  
  // Setter sizda — useTransition cleaner
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => setQuery(e.target.value);
  // ...
}

// ✅ useTransition (setter joyi)
const [isPending, startTransition] = useTransition();
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  startTransition(() => setQuery(e.target.value));
};
```

`useDeferredValue` — props orqali value uchun. Setter sizda — `useTransition` afzal.

---

## Amaliy Mashqlar

### Mashq 1 — `useFilteredList` (Oson)

`useDeferredValue` bilan filter hook. Stale indicator.

```tsx
function useFilteredList<T>(items: T[], filter: string, predicate: (item: T, filter: string) => boolean) {
  // Implement
}

function App() {
  const [filter, setFilter] = useState('');
  const { filtered, isStale } = useFilteredList(items, filter, (item, f) => item.name.includes(f));
  // ...
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useFilteredList<T>(
  items: T[],
  filter: string,
  predicate: (item: T, filter: string) => boolean
): { filtered: T[]; isStale: boolean } {
  const deferredFilter = useDeferredValue(filter);
  const isStale = filter !== deferredFilter;
  
  const filtered = useMemo(
    () => items.filter(item => predicate(item, deferredFilter)),
    [items, deferredFilter, predicate]
  );
  
  return { filtered, isStale };
}
```

`useDeferredValue` filter'ni defer qiladi → heavy filter compute urgent renders'ni bloklamaydi. `isStale` UI feedback.

</details>

### Mashq 2 — `useToggleWithTransition` (Oson)

`useTransition` bilan toggle hook. Heavy content render uchun.

```tsx
function useToggleWithTransition(initial = false): {
  value: boolean;
  toggle: () => void;
  isPending: boolean;
} {
  // Implement
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useToggleWithTransition(initial = false) {
  const [value, setValue] = useState(initial);
  const [isPending, startTransition] = useTransition();
  
  const toggle = useCallback(() => {
    startTransition(() => {
      setValue(v => !v);
    });
  }, []);
  
  return { value, toggle, isPending };
}

// Usage
function HeavyTabSwitcher() {
  const { value: showChart, toggle, isPending } = useToggleWithTransition();
  
  return (
    <>
      <button onClick={toggle} disabled={isPending}>
        {showChart ? 'List' : 'Chart'}
      </button>
      {isPending && <Spinner />}
      {showChart ? <Chart /> : <List />}
    </>
  );
}
```

</details>

### Mashq 3 — `useExternalState` (O'rta)

`useSyncExternalStore` wrapper. Subscribe + getSnapshot pattern bilan generic hook.

```tsx
type Store<T> = {
  getState: () => T;
  subscribe: (cb: () => void) => () => void;
};

function useExternalState<T, S = T>(
  store: Store<T>,
  selector?: (state: T) => S
): S {
  // Implement
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useExternalState<T, S = T>(
  store: Store<T>,
  selector?: (state: T) => S
): S {
  const select = selector ?? ((s: T) => s as unknown as S);
  
  return useSyncExternalStore(
    store.subscribe,
    () => select(store.getState())
  );
}

// Usage
const counterStore = createStore({ count: 0, name: 'counter' });

function CounterDisplay() {
  // Full state
  const state = useExternalState(counterStore);
  return <span>{state.count}</span>;
}

function CounterCount() {
  // Selector
  const count = useExternalState(counterStore, s => s.count);
  return <span>{count}</span>;
}
```

⚠️ Selector pattern — har gal selector yangi result bo'lsa re-render. Object/array selector — `useMemo` yoki Reselect.

</details>

### Mashq 4 — `useFormFields` (O'rta)

`useId` bilan form fields hook. Multiple fields, ARIA support.

```tsx
function useFormFields<K extends string>(fields: K[]): Record<K, {
  inputId: string;
  errorId: string;
  inputProps: { id: string; 'aria-describedby': string };
  errorProps: { id: string };
}> {
  // Implement
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useFormFields<K extends string>(fields: K[]): Record<K, {
  inputId: string;
  errorId: string;
  inputProps: { id: string; 'aria-describedby': string };
  errorProps: { id: string };
}> {
  const baseId = useId();
  
  return useMemo(() => {
    const result = {} as Record<K, {
      inputId: string;
      errorId: string;
      inputProps: { id: string; 'aria-describedby': string };
      errorProps: { id: string };
    }>;
    
    for (const field of fields) {
      const inputId = `${baseId}-${field}`;
      const errorId = `${baseId}-${field}-error`;
      
      result[field] = {
        inputId,
        errorId,
        inputProps: { id: inputId, 'aria-describedby': errorId },
        errorProps: { id: errorId },
      };
    }
    
    return result;
  }, [baseId, fields]);
}

// Usage
function ContactForm() {
  const fields = useFormFields(['name', 'email', 'message']);
  
  return (
    <form>
      <label htmlFor={fields.name.inputId}>Name</label>
      <input {...fields.name.inputProps} />
      <span {...fields.name.errorProps}></span>
      
      <label htmlFor={fields.email.inputId}>Email</label>
      <input {...fields.email.inputProps} type="email" />
      <span {...fields.email.errorProps}></span>
    </form>
  );
}
```

`useId` base ID + suffix multiple fields — clean, type-safe.

</details>

### Mashq 5 — `useMediaQuery` with `useSyncExternalStore` (Qiyin)

CSS media query subscription hook. Tearing-safe, SSR-compatible.

```tsx
function useMediaQuery(query: string): boolean {
  // Implement
}

function ResponsiveLayout() {
  const isMobile = useMediaQuery('(max-width: 768px)');
  return isMobile ? <MobileNav /> : <DesktopNav />;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useMediaQuery(query: string): boolean {
  const subscribe = useCallback((callback: () => void) => {
    const mediaQuery = window.matchMedia(query);
    
    // Initial check + listener
    mediaQuery.addEventListener('change', callback);
    
    return () => {
      mediaQuery.removeEventListener('change', callback);
    };
  }, [query]);
  
  const getSnapshot = useCallback(() => {
    return window.matchMedia(query).matches;
  }, [query]);
  
  const getServerSnapshot = useCallback(() => {
    return false;  // SSR fallback — assume not match
  }, []);
  
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}

// Usage
function Layout() {
  const isMobile = useMediaQuery('(max-width: 768px)');
  const prefersDark = useMediaQuery('(prefers-color-scheme: dark)');
  
  return (
    <div className={prefersDark ? 'dark' : 'light'}>
      {isMobile ? <Mobile /> : <Desktop />}
    </div>
  );
}
```

**Tushuntirish:**

- `useSyncExternalStore` — Concurrent-safe (tearing prevention)
- `subscribe` — `matchMedia` change listener
- `getSnapshot` — joriy media query state
- `getServerSnapshot` — SSR fallback (deterministic)
- `useCallback` — subscribe va getSnapshot stable references

`getSnapshot` boolean qaytaradi — primitive, reference identity muammo yo'q.

`useState + useEffect` bilan bo'lsa — Concurrent rendering paytida tearing potential. `useSyncExternalStore` Concurrent-safe.

</details>

---

## Xulosa

R18+ Concurrent hooks — Concurrent rendering bilan birga keldi. Asosiy fikrlar:

- **Concurrent rendering (R18+)** — default rendering (alohida "Concurrent Mode" rejimi emas). Time slicing, interruptible rendering, Lanes priority. Cross-ref [`05-scheduler-lanes.md`](05-scheduler-lanes.md).
- **`useTransition`** — `[isPending, startTransition] = useTransition()`. State update'ni TransitionLane'ga (non-urgent priority). `isPending` UI feedback (spinner, opacity, disable). R18: `startTransition(callback)` sinxron only; R19+: async scope qo'llab-quvvatlanadi (Actions foundation).
- **`startTransition` standalone** — `React.startTransition(cb)` hook tashqarida. `isPending` yo'q. Component tashqarida ham ishlatish mumkin.
- **`isPending` flag pattern** — variants: spinner show, opacity + disable, skeleton, stale content + indicator. Multiple transitions overlap — `isPending` umumiy.
- **`useDeferredValue`** — `useDeferredValue<T>(value)` value'ni defer qiladi (DeferredLane priority). Stale value detection (`oldValue !== newValue`). Initial render — `deferred === value` (sync).
- **`useTransition` vs `useDeferredValue`** — bir xil maqsad, har xil API: `useTransition` setter joyida (write), `useDeferredValue` value joyida (read). `isPending` kerak — `useTransition`. Faqat props value — `useDeferredValue`.
- **`useSyncExternalStore`** — `useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)`. Library author'lar uchun (Redux, Zustand, Jotai, Recoil). `getSnapshot` har chaqiruvda **bir xil reference** qaytarish shart (yo'qsa infinite re-render). `getServerSnapshot` SSR uchun deterministic value.
- **Tearing prevention** — Concurrent rendering paytida external state o'zgarsa tearing yuz berishi mumkin. `useState + useEffect` manual subscription tearing-prone (R18+ Concurrent), `useSyncExternalStore` Concurrent-safe: render'da o'qilgan snapshot consistency check'ga olinadi (`pushStoreConsistencyCheck`), commit'dan oldin yangi `getSnapshot()` bilan taqqoslanadi (`checkIfSnapshotChanged`); mos kelmasa `forceStoreRerender` SyncLane'da sinxron re-render qiladi.
- **`useId`** — `useId(): string`. Deterministic unique ID Fiber tree path asosida. SSR-safe (server va client bir xil ID). Format `:r0:`, `:r1:`. Multiple IDs single hook (suffix). Anti-pattern: keys uchun (Rules of Hooks buzilishi).
- **Hydration mismatch prevention** — `useId` `Math.random`/`Date.now`/`window.*` patterns o'rniga. `useEffect` client-only state. `suppressHydrationWarning` aniq nuqta. `useSyncExternalStore` `getServerSnapshot` browser API'lar uchun.
- **Decision Tree** — state setter + isPending → `useTransition`, faqat value props → `useDeferredValue`, external store → `useSyncExternalStore` (library), unique ID → `useId`. Sodda case'lar — `useState`/`useEffect`/`useReducer` (Concurrent hooks overkill).
- **Modern state library'lar** `useSyncExternalStore` ishlatadi — Redux v8+, Zustand, Jotai, MobX, Recoil. Application code library API orqali.

Keyingi bo'lim: R19 yangi hooks — `use()` (Promise va Context conditional reading), `useFormStatus` (form submission state), `useActionState` (form action state, R18 `useFormState` rename), `useOptimistic` (optimistic UI updates).

---

**Keyingi bo'lim:** [23-r19-hooks.md](23-r19-hooks.md) — React 19 yangi hooks: `use()` (Promise va Context — conditional context reading R19, Suspense bilan integration), `useFormStatus()` (form submission holati pending/data/method/action `<form action>` ichida cross-ref 13), `useActionState()` (form action state management, avval R18 `useFormState` edi), `useOptimistic()` (optimistic UI updates rollback on error). Decision tree — qaysi hook qachon, R19 form ekosistema bilan integration.
