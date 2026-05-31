# Bo'lim 30: Concurrent Rendering — Mental Model va Invariants

> Concurrent rendering — React 18'da **default** sifatida stable bo'lgan ichki rendering modeli. Bu fayl `useTransition`/`useDeferredValue` API'larini qaytarib tushuntirmaydi (ular `22-concurrent-hooks.md`'da) va Scheduler'ning bit-darajali implementation'ini ham takrorlamaydi (u `05-scheduler-lanes.md`'da). Bu yerda **mental model** va **invariants** o'rganiladi: nima uchun komponent yozish usuli sync rendering bilan solishtirganda o'zgaradi, nima uchun render purity endi qattiq talab, Strict Mode 2x effect cycle nima uchun qo'shilgan, tearing nima va nima uchun external store consistency muammoga aylanadi, va `startTransition` invariants bilan qanday bog'lanadi.

---

## Mundarija

- [Sync vs Concurrent Rendering — Mental Shift](#sync-vs-concurrent-rendering--mental-shift)
- [Render Phase Restartable — Purity Invariant](#render-phase-restartable--purity-invariant)
- [Strict Mode 2x Effect Cycle (R18+)](#strict-mode-2x-effect-cycle-r18)
- [Tearing — Concurrent External Store Muammosi](#tearing--concurrent-external-store-muammosi)
- [Invariant 1: Render Purity](#invariant-1-render-purity)
- [Invariant 2: State Updates Functional Merge](#invariant-2-state-updates-functional-merge)
- [Invariant 3: Effect Cleanup va Setup Pair Symmetry](#invariant-3-effect-cleanup-va-setup-pair-symmetry)
- [Invariant 4: External Subscription Consistency](#invariant-4-external-subscription-consistency)
- [Anti-Pattern 1: Render-Time Mutation](#anti-pattern-1-render-time-mutation)
- [Anti-Pattern 2: setState Callback Side Effects](#anti-pattern-2-setstate-callback-side-effects)
- [Anti-Pattern 3: Stale Closure Concurrent State](#anti-pattern-3-stale-closure-concurrent-state)
- [Anti-Pattern 4: External Mutable Reads During Render](#anti-pattern-4-external-mutable-reads-during-render)
- [`startTransition` Invariants Context'ida](#starttransition-invariants-contextida)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Sync vs Concurrent Rendering — Mental Shift

### Nazariya

**Sync rendering** (R16/R17 default) — render bir marta boshlangach, **yakuniga yetguncha to'xtatib bo'lmaydi**. Komponent funksiyasi chaqiriladi → Reconciler virtual tree quradi → Commit Phase'da DOM'ga mutation qiladi. Bu jarayon **uninterruptible**: agar ota-komponent uzoq vaqt render bo'lsa (frame budget ≈16.67ms dan oshsa), brauzerning main thread shu davr ichida to'liq band bo'ladi, hech qanday user input/animation/scroll handler ishlamaydi.

**Concurrent rendering** (R18+ default `createRoot` orqali) — rendering modeli **interruptible** va **restartable** bo'lishni qo'llab-quvvatlaydi: Transition/Deferred ish davomida Scheduler `frameYieldMs` (R16.5+ source'da `5` konstanta, R18/R19'da o'zgarmagan) bo'yicha `shouldYield()` chaqiradi va main thread'ni brauzerga qaytaradi. (Oddiy DefaultLane update'lar esa blocking — interruptibility faqat Transition uchun amalda yoqiladi, quyida batafsil.) R18'da Scheduler'ga `isInputPending` Web API integration kodi qo'shildi (`enableIsInputPending` feature flag orqali; default `false` OSS production'da, Meta internal'da `true`) — flag yoqilganda paint kerak yoki user input pending bo'lsa, yieldda yuqori sezgirlik. User input keladi, paint kerak bo'ladi, yoki yuqori-priority lane chiqadi — Reconciler **joriy work-in-progress tree'ni tashlaydi va boshqatdan boshlaydi**.

> **Versiya evolyutsiyasi (Rendering Model):**
> - **R0.14–R15:** Stack Reconciler — recursive DFS, single-pass, **interrupt qilib bo'lmaydi** (call stack'dagi recursion'ni to'xtatish chaqiriq orqasiga qaytarish'siz iloji yo'q).
> - **R16:** Fiber Reconciler joriy etildi — **iterativ DFS** linked list pointers (child/sibling/return), interruption mexanikasi infrastructure sifatida tayyor, lekin **production sync mode** default.
> - **R18 (2022):** `createRoot` API → **Concurrent rendering default**, `useTransition`/`useDeferredValue` stable, automatic batching kengaytirildi.
> - **R19:** Concurrent + Server Components + Compiler integration'i, asosiy mental model R18'dagi bilan bir xil — concurrent invariants barchasi qoladi.
> - **Sabab:** Sync rendering 60fps frame budget (≈16.67ms) ichiga sig'mas edi katta tree'larda — input lag, jank, dropped frames. Concurrent rendering Scheduler'ni high-priority work uchun ajratadi.

Bu o'zgarish **bir qator fundamental ta'sir** qiladi:

1. **Komponent funksiyasi bir necha marta chaqirilishi mumkin** bitta logical render uchun (interrupt → restart). Bu reaktiv emas, **strukturaviy** o'zgarish.
2. **Render Phase'da side effect taqiq** — agar render restart bo'lsa, side effect **bir necha marta** bajariladi (network call, DOM mutation, localStorage write). Bu R17 va undan oldin ham anti-pattern edi, lekin R18+'da **kuzatilishi mumkin bo'lgan bug** sifatida tez-tez ko'rinadi.
3. **State snapshot per render** — har render alohida snapshot bo'lib, yangi snapshot eskisini qaytadan hisoblamaydi.
4. **Mental model: "render = pure function call"** — har chaqirilganda bir xil natija (ref/state/props bir xil bo'lsa).

Sync mental model'ga ko'nikkan dasturchilar Concurrent'ni "asynchronous" deb tushunadilar — bu noto'g'ri. Concurrent rendering hali ham **synchronous JavaScript execution** ichida sodir bo'ladi (single-threaded), faqat **bir nechta yieldable bo'laklarga** bo'linadi. Promise yoki async/await yo'q render ichida — faqat task scheduling.

Ikkinchi muhim nuqta: **Concurrent rendering opt-in emas** R18'dan boshlab. `createRoot` chaqirilsa — barcha update'lar concurrent work loop (`performConcurrentWorkOnRoot`) orqali o'tadi va automatic batching avtomatik ishlaydi. Lekin oddiy update'lar (DefaultLane) **blocking** — ya'ni time-slicing'siz, bir blokda yakuniga yetadi. Interruptible (uziluvchi, time-sliced) rendering faqat `startTransition`/`useTransition`/`useDeferredValue` orqali boshlangan Transition update'lar uchun yoqiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Concurrent rendering Scheduler'ning **work loop**'iga asoslangan:

```
performConcurrentWorkOnRoot(root)
  ├─ renderRootConcurrent(root, lanes)
  │    ├─ workLoopConcurrent()
  │    │    while (workInProgress !== null && !shouldYield())
  │    │      performUnitOfWork(workInProgress)
  │    │
  │    ├─ shouldYield() === true
  │    │    └─ exit loop, return RootInProgress
  │    │
  │    └─ tree complete?
  │         ├─ Yes → commitRoot(root)
  │         └─ No  → schedule continuation via MessageChannel
  │
  └─ continuation: ensureRootIsScheduled
```

`shouldYield` har `performUnitOfWork`'dan keyin chaqiriladi, asosi:

```javascript
// React Scheduler source
function shouldYield() {
  const timeElapsed = currentTime - startTime;
  if (timeElapsed < frameYieldMs) {
    return false;
  }
  // Yield if input is pending or main thread elapsed >= 5ms
  if (enableIsInputPending) {
    if (needsPaint || scheduling.isInputPending(continuousOptions)) {
      return true;
    }
    return timeElapsed >= maxYieldMs;
  }
  return true;
}
```

`frameYieldMs = 5` ms (R16.5+ source'da kiritilgan konstanta, R18/R19'da o'zgarmagan) — yield interval. R17 va undan oldinroq Scheduler ham 5ms yield ishlatardi; R18'dagi asosiy yangilik — Chrome `isInputPending` Web API integration kodining `enableIsInputPending` feature flag orqali kiritilishi (flag yoqilsa, input/paint pending bo'lsa darhol yield) va lane priority bilan birga ishlash (paint priority hisobi). OSS build'da `enableIsInputPending = false` default — fallback `timeElapsed >= frameYieldMs` bo'yicha yield ishlaydi.

**Restart mexanikasi:** agar render davomida yangi yuqori-priority update kelsa (masalan, click event SyncLane'ga keladi, joriy render Transition Lane), Reconciler:

1. `prepareFreshStack(root, lanes)` — workInProgress tree'ni null qilib qayta yaratadi
2. Yangi (yuqori) lane bilan boshlaydi
3. Past-priority work keyinroq qayta scheduled bo'ladi (lane bitmap'da hali bor)

Restart paytida **wasted work** bo'ladi — qisman qurilgan tree tashlab yuboriladi. Bu sababdan render purity zarur: agar render-time mutation bo'lsa, restart natijada **side effect ikki marta** sodir bo'ladi.

ASCII timeline (concurrent vs sync):

```
SYNC RENDERING (R17):
[========================== 50ms render =========================] commit
                                                                 │
0ms                                                            50ms

  ↑ User click @ 30ms
  ↑ Input lag = 20ms (kutib turadi)


CONCURRENT RENDERING (R18+):
[==5ms==] yield [==5ms==] yield [click] [==2ms===] commit-click
                                  ↓
                               Restart with new state
                               [==5ms==] [==5ms==] [==5ms==] [==5ms==] commit
                              
0ms                          30ms                                       55ms

  ↑ User click @ 30ms → Yield → Sync click commit ≈ 31-32ms
  ↑ Input lag ≈ 1-2ms
```

`MessageChannel.postMessage` macro-task scheduling sifatida ishlatiladi (cross-ref `05-scheduler-lanes.md`) — `setTimeout(fn, 0)` o'rniga, chunki `MessageChannel` 4ms minimum delay'ni chetlab o'tadi (HTML5 spec setTimeout clamp).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sync vs Concurrent farqini ko'rsatuvchi minimal misol:

```tsx
// ❌ Sync rendering (R17 yoki R18'da `ReactDOM.render`):
import { render } from 'react-dom';
render(<App />, document.getElementById('root'));
// Har setState event handler ichida bo'lsa, batched.
// Tashqarida (Promise/setTimeout/native event listener) — alohida render.

// ✅ Concurrent rendering (R18+):
import { createRoot } from 'react-dom/client';

const container = document.getElementById('root');
if (!container) throw new Error('Root container "#root" topilmadi');
const root = createRoot(container);
root.render(<App />);
// Har setState barcha context'larda batched.
// useTransition/startTransition orqali interruptible work.
```

Interruption demo (faqat `useTransition` ichida explicit):

```tsx
import { useState, useTransition } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();

  function handleChange(value: string) {
    setQuery(value); // Urgent: input darhol yangilanadi (SyncLane)

    startTransition(() => {
      // Non-urgent: og'ir filter operatsiyasi (TransitionLane)
      const filtered = computeExpensiveResults(value);
      setResults(filtered);
    });
  }

  return (
    <>
      <input value={query} onChange={(e) => handleChange(e.target.value)} />
      {isPending && <span>Yangilanmoqda...</span>}
      <ResultList items={results} />
    </>
  );
}
```

Bu yerda foydalanuvchi tez yozsa, `setQuery` har keystroke'ga sync render qiladi (input "responsive" bo'lib qoladi), lekin `setResults` Transition Lane'da ishlanadi va keyingi keystroke bilan **uziladi va qayta boshlanadi**. Eski natijalar tashlab yuboriladi.

> **Eslatma:** `useTransition` API tafsiloti `22-concurrent-hooks.md`'da. Bu yerda faqat **mental model** uchun ishlatildi.

</details>

---

## Render Phase Restartable — Purity Invariant

### Nazariya

Concurrent rendering kafolatining eng kuchli tomoni — **render funksiyasi har vaqt restart bo'lishi mumkin**. Komponent funksiyasi bir logical render uchun **bir necha marta** chaqirilishi mumkin: production'da odatda 1 marta, Strict Mode dev'da minimum 2 marta, concurrent restart sodir bo'lsa N+ marta (yuqori-priority update interrupted Transition'ni qayta-qayta restartlashi mumkin). Reconciler shu hodisani **silently** qiladi — komponent buni sezmaydi.

Bu sababdan **Render Purity** Concurrent rendering'da **invariant**, anti-pattern emas:

- Render funksiyasi **bir xil props/state/context** uchun **bir xil JSX** qaytarishi shart (idempotent).
- Render Phase ichida **side effect bajarilmaydi**: DOM mutation, network call, `setState` (boshqa komponentga), `localStorage.setItem`, `document.title = ...`, console logging (development logs istisno emas — Strict Mode `disableLogs` internal pattern bilan boshqaradi).
- Render funksiyasi **mutable external state'ni o'qimaydi** — `Date.now()`, `Math.random()`, `window.innerWidth`, `localStorage.getItem(...)` — bu qiymatlar har chaqirilganda farqli bo'lsa, idempotency buziladi.

NIMA UCHUN bu qattiq talab:

1. **Restart safety** — agar render restart bo'lsa, qisman bajarilgan effect (HTTP POST, DOM write) **ikki marta** sodir bo'ladi va orqaga qaytarib bo'lmaydi.
2. **Wasted work tolerable** — Reconciler restart'ni "free" deb hisoblaydi. Agar render purity bo'lsa, restart faqat CPU cycle'larni sarflaydi, lekin **kuzatilishi mumkin bo'lgan bug yaratmaydi**.
3. **Strict Mode 2x render** — development paytida har komponent funksiyasini ikki marta chaqiradi (R16.3+) — purity'ni darhol fosh qiladi (mutation `count++` har render'da boshqacha qiymat).
4. **Compiler talabi** — React Compiler (cross-ref `31-react-compiler.md`) komponentni **pure function** deb hisoblaydi va auto-memoization qiladi. Mutation aniqlansa, Compiler shu komponentni optimize qilmaydi (bails out) va diagnostic beradi; aniqlanmaydigan subtle mutation bo'lsa — memoized qiymat stale qolib, runtime'da noto'g'ri natija beradi.

Render Phase **qayerda boshlanadi va tugaydi**:

- **Boshlash:** Komponent funksiyasi chaqirilishi (`Component(props)`).
- **Tugash:** JSX `return` qaytarilishi.

Shu oraliqda **ruxsat etilgan**: `useState`/`useReducer` snapshot o'qish, `useMemo`/`useCallback` cached computation, `useContext` value o'qish, lazy initial state (`useState(() => expensive())` — bir marta mount'da).

**Ruxsat etilmagan**: setState boshqa komponentga (Render Phase ichida warning — "Cannot update a component while rendering a different component"), DOM mutation, fetch, Date/Math/Crypto/Performance random reads, mutable global o'qish, render Phase'da `ref.current = x` yozish yoki `ref.current` o'qish (ref re-render tetiklamaydi va commit'dan oldin uning qiymati eskirgan/initsializatsiya qilinmagan bo'lishi mumkin — bu purity invariant'ni buzadi), uncached side effect (har render bajariladigan).

Yagona istisno: **rendering phase setState o'zining state'iga** — bu shart bo'yicha pure (yangi state old state'dan derive bo'ladi), lekin to'g'ri ishlatilsa — `useState` ga setter ichida `setState(prev => derive(prev))` yangi state'ni hisoblaydi va **render qaytadan boshlanadi** (warning'siz, faqat **bir marta to'g'rilash** sifatida ruxsat).

<details>
<summary><strong>Under the Hood</strong></summary>

React'ning rasmiy "Rules of React" hujjatida (`react.dev/reference/rules`) Render Purity quyidagi qoidalar to'plamiga bo'linadi:

1. **Components and Hooks must be pure** — bir xil input → bir xil output.
2. **Components must not mutate non-local values** — props, state, context, refs Render Phase'da.
3. **Side effects must run outside of render** — useEffect, event handlers, useSyncExternalStore.

Reconciler'ning **invariant check** mexanizmlari:

**Strict Mode double invocation (R16.3+ render):** development build'da Strict Mode subtree'dagi komponent uchun `renderWithHooks` funksiyasi komponentni bir marta chaqiradi, keyin hook state'ini reset qilib **darhol qaytadan** chaqiradi (`react-reconciler/src/ReactFiberHooks.js`'dagi dev double-render yo'li). Konseptual ko'rinishda:

```javascript
// Konseptual model (development only)
function renderComponentDEV(Component, props) {
  let result = Component(props); // birinchi chaqiriq
  if (workInProgressIsStrictMode) {
    // hook state reset → ikkinchi chaqiriq (purity check)
    result = Component(props);
  }
  return result;
}
```

Strict Mode shuningdek: `useState` initializer, `useMemo` factory, `useReducer` reducer — barchasini **ikki marta** chaqiradi.

**Compiler-time invariant** (React Compiler):
```javascript
// Pseudocode: Compiler analiz
function CartSummary({ items }) {
  items.push('extra'); // ❌ non-local value mutation — Compiler bu komponentni
                       //    optimize qilmaydi (bails out) + eslint diagnostic
  
  const total = items.reduce((sum, item) => sum + item.price, 0);
  // ✅ Pure read, memoized as _c[0]
  
  return <div>{total}</div>;
}
```

Compiler `eslint-plugin-react-compiler` orqali statik check qiladi (cross-ref `31-react-compiler.md`).

**Bailout va render purity bog'liqligi:**

Reconciler'ning bailout algoritmi (cross-ref `04-reconciliation.md`) **default FunctionComponent uchun `prevProps === nextProps` reference equality** ga asoslanadi (`MemoComponent` uchun `shallowEqual`). Agar Fiber'da pending update yo'q va props/context o'zgarmasa, Reconciler komponentni **qayta render qilmaydi** va eski JSX'ni qayta ishlatadi:

```
Update propagation
       │
       ▼
   Reconciler tries Component
       │
       ├─ prevProps === nextProps && no context change && no pending lanes
       │  → bailoutOnAlreadyFinishedWork → komponent funksiyasi chaqirilmaydi
       │
       └─ changed → renderWithHooks(Component, nextProps) chaqiriladi
```

Agar render impure bo'lsa (mutation render Phase'da), bailout natijasida **mutation skipped** bo'lib qoladi va silent bug paydo bo'ladi: state'da kuzatilmaydigan o'zgarish saqlanadi.

**Restart wasted work cost:**

Restart paytida partially-built tree tashlanadi. Agar tree `n` Fiber'dan iborat bo'lsa va restart `k`-Fiber'da sodir bo'lsa, `k` ta Fiber qayta yaratiladi. Bu **CPU cost**, lekin observability nol — agar render pure bo'lsa.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Render-time mutation klassik bug:

```tsx
// ❌ Anti-pattern: Render Phase mutation
let renderCount = 0; // module-level mutable

function Counter() {
  renderCount++; // ❌ Side effect render'da
  console.log('Render count:', renderCount); // ❌ Side effect
  
  return <div>Count: {renderCount}</div>;
}

// Strict Mode 2x render natijasi:
// Render count: 1
// Render count: 2  ← Strict Mode 2-chi chaqirig'i
// UI: "Count: 2"
// Production: "Count: 1" — ikki natija farqli, BUG.
```

To'g'ri yondashuv:

```tsx
// ✅ State'da renderCount'ni tutish (lekin counter useEffect'da increment)
import { useState, useEffect, useRef } from 'react';

function Counter() {
  const renderCountRef = useRef(0);
  
  useEffect(() => {
    renderCountRef.current += 1; // ✅ Effect'da side effect
  });
  
  return <div>Renders: {renderCountRef.current}</div>;
}
// Strict Mode 2x effect cycle: mount → cleanup → mount,
// useRef qiymati Strict Mode'da 2x increment'ga to'g'ri keladi (qoldirilsa).
```

Idempotency invariant — har chaqiriqda bir xil natija:

```tsx
// ❌ Anti-pattern: render Phase'da Date.now()
function Timestamp() {
  const now = Date.now(); // ❌ Har chaqiriqda boshqa qiymat
  return <span>{new Date(now).toISOString()}</span>;
}
// Strict Mode 2x render: ikki marta chaqirilganda ikki farqli now qiymati,
// UI ko'rinmaydi (faqat 2-chi chaqiriq result'i ishlatiladi),
// lekin restart bo'lsa — vaqt o'zgaradi, hydration mismatch SSR'da.

// ✅ Side effect via state:
import { useState, useEffect } from 'react';

function Timestamp() {
  const [now, setNow] = useState<number | null>(null);
  
  useEffect(() => {
    setNow(Date.now()); // ✅ Mount'dan keyin
    const id = setInterval(() => setNow(Date.now()), 1000);
    return () => clearInterval(id);
  }, []);
  
  if (now === null) return <span>--</span>;
  return <span>{new Date(now).toISOString()}</span>;
}
```

Render Phase'da setState bug:

```tsx
// ❌ Anti-pattern: render'da boshqa komponentga setState
import { useState } from 'react';

function Parent() {
  const [count, setCount] = useState(0);
  return <Child count={count} onCount={setCount} />;
}

function Child({ count, onCount }: { count: number; onCount: (n: number) => void }) {
  if (count < 5) {
    onCount(count + 1); // ❌ Render'da parent setState — warning + cheksiz loop xavfi
  }
  return <span>{count}</span>;
}

// React warning: "Cannot update a component while rendering a different component"

// ✅ Effect'da yoki event handler'da:
import { useEffect } from 'react';

function Child({ count, onCount }: { count: number; onCount: (n: number) => void }) {
  useEffect(() => {
    if (count < 5) onCount(count + 1); // ✅ Mount/update'dan keyin
  }, [count, onCount]);
  
  return <span>{count}</span>;
}
// Lekin bu pattern hali ham anti-pattern — derived state Parent'da
// to'g'ridan-to'g'ri hisoblansin.
```

Komponent o'zining state'iga rendering-time setState — **istisno** sifatida ruxsat etilgan:

```tsx
// ✅ Render Phase'da o'zining state'iga setState — derived state o'zgarsa
import { useState } from 'react';

function Form({ items }: { items: string[] }) {
  const [filter, setFilter] = useState('');
  const [prevItemsLen, setPrevItemsLen] = useState(items.length);
  
  // Items o'zgardi — filter'ni reset qilamiz
  if (items.length !== prevItemsLen) {
    setPrevItemsLen(items.length);
    setFilter(''); // ✅ Render restart, lekin one-time correction
  }
  
  return (
    <input
      value={filter}
      onChange={(e) => setFilter(e.target.value)}
    />
  );
}
// Bu pattern React docs'da "store information from previous renders" deb tasvirlangan.
// Lekin asosiy holatlarda key trick yoki useEffect afzal — cheksiz loop xavfi yo'q.
```

</details>

---

## Strict Mode 2x Effect Cycle (R18+)

### Nazariya

`<StrictMode>` — React'ning **development-only** wrapper komponenti, anti-pattern'larni darhol fosh qilish uchun mo'ljallangan. R18'gacha (R16.3+) faqat **render funksiyasi 2x chaqirilardi** — purity'ni tekshirish. R18'da yangi mexanizm qo'shildi: **effect mount/cleanup/mount cycle 2x**.

> **Versiya evolyutsiyasi (Strict Mode):**
> - **R16.3+ (2018):** `StrictMode` joriy etildi — render funksiyalarini 2x chaqirish (komponent body, useState initializer, useReducer reducer, useMemo factory).
> - **R17.x:** Strict Mode `console.log` ni ikki marta ko'rsatardi (chalkashlik).
> - **R18 (2022):** `console.*` chaqiriqlar **disabled** Strict Mode 2-chi render'da (dev-mode `console.log` `disableLogs` internal pattern), va **yangi** — `useEffect`/`useLayoutEffect` har mount'da `mount → cleanup → mount` cycle bajariladi.
> - **R19:** Bir xil Strict Mode mexanizmi, qo'shimcha invariants check (callback ref cleanup function support — Strict Mode bu cleanup'ni ham 2x chaqirib tekshiradi).
> - **Sabab:** R18 Concurrent rendering + Offscreen API + Fast Refresh "remount" simulatsiyasi (komponent unmount → mount qayta ishlatilganda effect to'g'ri bo'lishi kerak).

Bu cycle development-only invariants tekshiruvi:

1. Komponent mount → effect setup ishlaydi.
2. **Strict Mode** uni darhol unmount qilgandek effect cleanup'ni chaqiradi.
3. **Strict Mode** mount qaytaradi — effect setup yana chaqiriladi.

Maqsad: **cleanup va setup pair correctness'ni majburlash**. Agar cleanup yo'q bo'lsa yoki noto'g'ri bo'lsa — bug 2-chi mount'da darhol ko'rinadi (ikki marta event listener, ikki marta interval, race condition).

```
Mount sequence (Strict Mode dev):

Component mounts
       │
       ▼
useEffect setup #1 (analytics.track)
       │
       ▼
Strict Mode forces cleanup
       │
       ▼
useEffect cleanup #1 (analytics.untrack)
       │
       ▼
Strict Mode forces re-mount
       │
       ▼
useEffect setup #2 (analytics.track)
       │
       ▼
[Komponent endi normal lifecycle'da]
```

NIMA UCHUN R18'da qo'shildi:

1. **Concurrent rendering preparation** — render restart bo'lsa, useEffect cleanup → setup ham takrorlanishi mumkin (nadir holat). Strict Mode bu invariant'ni har mount'da test qiladi.
2. **Offscreen API** (R18 experimental) — komponent "yashirilishi" mumkin (DOM'da qoladi, lekin ko'rinmaydi). Yashirilish va qaytarib ko'rsatish — effect cleanup → setup cycle'iga teng.
3. **Fast Refresh** — file save'da komponent state saqlanib effects qayta ishga tushadi. Cleanup buzuq bo'lsa, dev experience yomon.
4. **Error recovery** — Suspense yoki Error Boundary recovery'sida re-mount mumkin.

Strict Mode 2x effect cycle **production'da yo'q** — `React.StrictMode` development-only behavior. `process.env.NODE_ENV === 'development'` check internal'da.

**Idempotency invariant**: cleanup + setup pair **bir xil natijaga** olib kelishi kerak — go'yo komponent hech qachon unmount bo'lmagandek. Misollar:

- **Subscription:** `subscribe()` setup, `unsubscribe()` cleanup. 2x cycle: subscribe → unsubscribe → subscribe → server bir subscription ko'radi.
- **Fetch:** AbortController setup, `controller.abort()` cleanup. 2x cycle: fetch boshlanadi → abort → yangi fetch boshlanadi → faqat yangi fetch javobi ishlatiladi.
- **Analytics:** `track('view')` setup. 2x cycle: track → cleanup'da nima qilinadi? Agar untrack API yo'q bo'lsa — server'da **ikki view** ko'rinadi. Bu sababdan analytics event handler'ga ko'chirilishi tavsiya etiladi (cross-ref `16-useeffect.md` "You Might Not Need an Effect").

<details>
<summary><strong>Under the Hood</strong></summary>

Commit Phase'da effect mount qilingach, Strict Mode subtree uchun React qo'shimcha `mount → cleanup → mount` qadamini bajaradi. Bu alohida dev-only yo'l (`react-reconciler/src/ReactFiberCommitWork.js`'dagi double-invoke effects logikasi) — normal mount tugagandan keyin effect'ni unmount qilib qaytadan mount qiladi. Konseptual ko'rinishda:

```javascript
// Konseptual model (DEV only)
function mountPassiveEffectsDEV(effectList, isStrictMode) {
  for (const effect of effectList) {
    if (effect.tag === HookPassive) {
      effect.destroy = effect.create(); // birinchi mount
    }
  }

  if (isStrictMode) {
    // qo'shimcha cleanup → mount cycle
    for (const effect of effectList) {
      if (effect.tag === HookPassive) {
        if (effect.destroy) effect.destroy(); // cleanup
        effect.destroy = effect.create();     // re-mount
      }
    }
  }
}
```

Strict Mode cycle **faqat mount** paytida sodir bo'ladi (initial render). Update'larda effect oddiy `cleanup → setup` qiladi (deps o'zgarsa).

**Disable strategy** — bug bilan kurashish o'rniga effect'ni to'g'ri yozish:

```javascript
// ❌ Anti-pattern: Strict Mode'ni o'chirish
// <StrictMode>'ni olib tashlash bug'ni yashiradi, lekin tuzatmaydi.
// Production'da hech qachon strict mode yo'q,
// lekin Concurrent restart yoki Fast Refresh bug ko'rinadi.

// ✅ Effect'ni to'g'ri yozing — cleanup symmetry bilan
useEffect(() => {
  const controller = new AbortController();
  fetch(url, { signal: controller.signal })
    .then(/* ... */);
  return () => controller.abort();
}, [url]);
```

**Reactivity vs Idempotency:**

Strict Mode 2x cycle "**network 2x request**" holatiga olib kelishi mumkin. Bu React xatosi emas — bu effect'da cleanup yetishmayotganini fosh qiladi. To'g'ri implementation: `AbortController` bilan birinchi request bekor qilinadi va faqat 2-chi request resolve bo'ladi. Server'da ikki request bo'lsa ham — javob faqat oxirgisi ishlatiladi, kuzatilishi mumkin side effect yo'q.

HTTP semantikasida (RFC 9110) GET, PUT, DELETE — idempotent (bir necha marta yuborilsa ham server holati bir xil qoladi; GET'da qo'shimcha network cost'dan tashqari side effect yo'q). POST — idempotent emas: ikki marta yuborilsa ikkita resurs yaratilishi mumkin. Idempotent bo'lmagan operatsiyalarni (odatda POST, yoki idempotency key'siz mutatsiyalar) `useEffect` ichiga qo'ymaslik kerak — ular event handler'da bajarilsin, chunki Strict Mode 2x cycle ularni ikki marta tetiklaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Subscription pattern — Strict Mode safe:

```tsx
import { useEffect } from 'react';

function ChatRoom({ roomId }: { roomId: string }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    
    return () => {
      connection.disconnect(); // ✅ Cleanup
    };
  }, [roomId]);
  
  return <div>Room: {roomId}</div>;
}

// Strict Mode 2x cycle:
// 1. connect()
// 2. disconnect()
// 3. connect()
// Server: bir aktiv connection ko'radi (cleanup ishlaganligi sabab)
```

Race condition — AbortController orqali safe:

```tsx
import { useEffect, useState } from 'react';

interface User {
  id: string;
  name: string;
}

function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    const controller = new AbortController();
    
    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then((res) => res.json())
      .then((data: User) => setUser(data))
      .catch((err: unknown) => {
        if (err instanceof Error && err.name !== 'AbortError') console.error(err);
      });
    
    return () => controller.abort(); // ✅ Cleanup
  }, [userId]);
  
  return user ? <h1>{user.name}</h1> : <p>Loading...</p>;
}

// Strict Mode 2x cycle:
// 1. fetch boshlanadi (request #1)
// 2. controller.abort() — request #1 cancelled
// 3. fetch boshlanadi (request #2)
// Faqat request #2 javobi setUser'ga keladi.
```

Analytics — useEffect anti-pattern:

```tsx
// ❌ Anti-pattern: analytics useEffect'da
import { useEffect } from 'react';

function ProductPage({ productId }: { productId: string }) {
  useEffect(() => {
    analytics.track('product_view', { productId });
  }, [productId]);
  
  return <div>Product: {productId}</div>;
}
// Strict Mode 2x: track ikki marta yuboriladi (untrack API yo'q).
// Server'da view count 2x — analytics buzuq.

// ✅ Event handler'da yoki Effect Event'da
import { useEffect } from 'react';

function ProductPage({ productId }: { productId: string }) {
  useEffect(() => {
    // ✅ Faqat router navigation'i paytida event sifatida
    if (typeof window !== 'undefined' && window.history.state?.tracked !== productId) {
      analytics.track('product_view', { productId });
      window.history.replaceState({ tracked: productId }, '');
    }
  }, [productId]);
  
  return <div>Product: {productId}</div>;
}

// ✅ Yoki router level'da, link click handler'ida:
function ProductLink({ id }: { id: string }) {
  return (
    <a
      href={`/products/${id}`}
      onClick={() => analytics.track('product_view', { productId: id })}
    >
      View
    </a>
  );
}
```

Timer pattern:

```tsx
import { useEffect, useState } from 'react';

function Clock() {
  const [time, setTime] = useState(new Date());
  
  useEffect(() => {
    const id = setInterval(() => setTime(new Date()), 1000);
    return () => clearInterval(id); // ✅ Cleanup
  }, []);
  
  return <span>{time.toISOString()}</span>;
}
// Strict Mode 2x: interval start → clear → interval start
// Faqat bir aktiv interval (cleanup ishlagani sabab)
```

</details>

---

## Tearing — Concurrent External Store Muammosi

### Nazariya

**Tearing** — Concurrent rendering paytida UI'ning **ikki qismida turli qiymat** ko'rinishi. Bu muammo external mutable store (React state'idan tashqari — Redux store, Zustand, observable, browser API) bilan birga ishlatilganda paydo bo'ladi.

Sync rendering'da tearing **mumkin emas** — render bir uzilmas blok bo'lib bajariladi, store o'zgarishi render davomida sodir bo'lmaydi (single thread). Concurrent rendering'da render **interrupt bo'ladi**, va shu interrupt orasida store o'zgarishi mumkin → keyingi qism eski qiymatdan render bo'lgan, joriy qism yangi qiymatdan render bo'lmoqda → **tear**.

```
External store: count = 0

Component A renders → count = 0 ko'radi → UI: "0"
       │
       ▼
[YIELD] — Scheduler main thread'ni qaytaradi
       │
       ▼
External event: store.setCount(1)
       │
       ▼
Component B renders → count = 1 ko'radi → UI: "1"
       │
       ▼
Commit: UI ekranda "0 / 1" — TEAR!
```

NIMA UCHUN React state'da tearing yo'q:

- React state Reconciler tomonidan **render davomida snapshot** sifatida ushlab turiladi (workInProgress.memoizedState).
- Render davomida `setState` chaqirilsa — yangi update keyingi render uchun queue'ga tushadi, joriy render eski snapshot bilan davom etadi.
- Har komponent bir xil snapshot ko'radi.

External store bu kafolat'ga ega emas — har `store.getState()` joriy qiymatni qaytaradi, snapshot yo'q.

NIMA UCHUN bu Concurrent specific:

- **Sync render:** `getState() → render → next component → getState() → render → commit` — barchasi atomic block, store o'zgarmaydi.
- **Concurrent render:** `getState() → render → YIELD → store change → getState() → render → ...` — interrupt orasida o'zgarish mumkin.

`useSyncExternalStore` (R18+, cross-ref `22-concurrent-hooks.md`) — bu muammoga **rasmiy yechim**:

```typescript
const value = useSyncExternalStore(
  subscribe,        // (callback) => unsubscribe
  getSnapshot,      // () => Snapshot — current value
  getServerSnapshot // () => Snapshot — for SSR
);
```

Mexanizm:

1. Render davomida `getSnapshot()` chaqiriladi va olingan qiymat hook state'ida saqlanadi.
2. React shu render uchun saqlangan snapshot bilan ishlaydi, shu bilan birga keyingi commit'da consistency check'ni rejalashtiradi (`pushStoreConsistencyCheck`).
3. Commit oldidan React store hali ham o'sha snapshot'ni qaytarayotganini tekshiradi (`checkIfSnapshotChanged`). Agar render davomida (yield orasida) store o'zgargan bo'lsa — snapshot mos kelmaydi.
4. Mos kelmasa, React `forceStoreRerender` orqali **sinxron (blocking) re-render** rejalashtiradi — bu re-render time-slicing'siz, bir blokda bajariladi, shu sababli barcha komponent yangi snapshot'ni bir xil ko'radi va tearing yuzaga kelmaydi.
5. Commit'dan keyin `subscribe` orqali change event kutiladi — store o'zgarsa, listener `getSnapshot`'ni qayta o'qib yangi render boshlaydi.

Bu mexanizm Redux v8+, Zustand v4+, MobX, Jotai — barcha modern store kutubxonalari ishlatadigan **rasmiy primitive**. Library author'lar uchun mo'ljallangan, application kod'da kamdan-kam ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`useSyncExternalStore` implementation'i (soddalashtirilgan):

```javascript
function useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot) {
  const fiber = currentlyRenderingFiber;
  
  // 1. Render boshida snapshot olamiz
  const snapshot = getSnapshot();
  
  // 2. Snapshot'ni hook state'da saqlaymiz
  const [stableSnapshot, setSnapshot] = useState(snapshot);
  
  // 3. Render davomida snapshot o'zgarishini check qilamiz
  if (!Object.is(stableSnapshot, snapshot)) {
    setSnapshot(snapshot);
    // Snapshot o'zgargan bo'lsa, React commit oldidagi consistency check
    // orqali forceStoreRerender'ni sinxron (blocking) re-render sifatida
    // rejalashtiradi — time-slicing'siz, tearing'ni oldini olish uchun.
  }
  
  // 4. Subscribe to store changes
  useEffect(() => {
    const handleStoreChange = () => {
      const newSnapshot = getSnapshot();
      setSnapshot(newSnapshot);
    };
    
    const unsubscribe = subscribe(handleStoreChange);
    
    // Subscription o'rnatilgan vaqtda snapshot o'zgargan bo'lsa
    handleStoreChange();
    
    return unsubscribe;
  }, [subscribe, getSnapshot]);
  
  return snapshot;
}
```

Asosiy implementation `react/src/ReactHooks.js` va `useSyncExternalStoreShim.js` (R18'gacha). R18'da `useSyncExternalStore` rasmiy hook sifatida qo'shildi, shu paytgacha `use-sync-external-store` package shim sifatida ishlatilgan.

**Tearing detection:** React Reconciler snapshot consistency'ni render va commit chegarasida tekshiradi:

1. `mountSyncExternalStore` — initial snapshot saqlanadi va commit uchun consistency check `pushStoreConsistencyCheck` orqali qator'ga qo'shiladi.
2. `updateSyncExternalStore` — har render'da yangi snapshot olinadi va saqlangan snapshot bilan `Object.is` solishtiriladi.
3. Agar farq bo'lsa (yoki commit oldidagi `checkIfSnapshotChanged` mosligini topmasa), `forceStoreRerender(fiber)` chaqiriladi — bu **sinxron (blocking) re-render** trigger qiladi.

`getServerSnapshot` SSR uchun — server'da `subscribe` ishlamaydi, `getSnapshot` ham window/storage API'ga bog'liq bo'lishi mumkin. Server uchun maxsus snapshot kerak.

**Performance trade-off:** `useSyncExternalStore` har render'da `getSnapshot` chaqiradi. Agar `getSnapshot` har chaqiriqda yangi reference qaytarsa (`{}` literal, array literal), `Object.is` har doim false bo'ladi → React store o'zgargan deb hisoblab har render'dan keyin yana re-render rejalashtiradi → cheksiz re-render loop. Library author'lar `getSnapshot` ichida memoization yoki primitive return qilishi shart.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Tearing demo — naive store ulanishi:

```tsx
// ❌ Anti-pattern: Concurrent rendering bilan tearing xavfi
let externalCount = 0;
const listeners = new Set<() => void>();

function setExternalCount(value: number) {
  externalCount = value;
  listeners.forEach((l) => l());
}

function useNaiveStore() {
  // useState'ni store sync uchun ishlatish — tearing xavfi
  const [, forceUpdate] = useState({});
  
  useEffect(() => {
    const listener = () => forceUpdate({});
    listeners.add(listener);
    return () => {
      listeners.delete(listener);
    };
  }, []);
  
  return externalCount; // Har render'da global qiymat
}

function CounterDisplay() {
  const count = useNaiveStore();
  return <span>{count}</span>;
}

// Concurrent rendering'da:
// Render boshida count = 0
// YIELD
// setExternalCount(1) chaqiriladi
// Render davom etadi, ikkinchi <CounterDisplay /> count = 1 ko'radi
// UI: "0 / 1" — TEAR
```

To'g'ri yondashuv — `useSyncExternalStore`:

```tsx
import { useSyncExternalStore } from 'react';

let externalCount = 0;
const listeners = new Set<() => void>();

function setExternalCount(value: number) {
  externalCount = value;
  listeners.forEach((l) => l());
}

function subscribe(callback: () => void) {
  listeners.add(callback);
  return () => {
    listeners.delete(callback);
  };
}

function getSnapshot() {
  return externalCount;
}

function useExternalCount() {
  return useSyncExternalStore(subscribe, getSnapshot, () => 0);
}

function CounterDisplay() {
  const count = useExternalCount();
  return <span>{count}</span>;
}

// Concurrent rendering'da:
// Render boshida snapshot = 0
// YIELD
// setExternalCount(1)
// React: snapshot consistency check failed → sync re-render
// Barcha komponent'lar bir xil snapshot bilan render bo'ladi
// UI: "1 / 1" — NO TEAR
```

Browser API ulanishi (`window.matchMedia`):

```tsx
import { useSyncExternalStore } from 'react';

function useMediaQuery(query: string): boolean {
  const subscribe = (callback: () => void) => {
    const mql = window.matchMedia(query);
    mql.addEventListener('change', callback);
    return () => mql.removeEventListener('change', callback);
  };
  
  const getSnapshot = () => window.matchMedia(query).matches;
  
  // SSR: window yo'q
  const getServerSnapshot = () => false;
  
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}

// Foydalanish
function ResponsiveLayout() {
  const isWide = useMediaQuery('(min-width: 1024px)');
  return isWide ? <DesktopLayout /> : <MobileLayout />;
}
// Concurrent + Strict Mode'da tearing yo'q
```

`localStorage` — cross-tab sync:

```tsx
import { useSyncExternalStore } from 'react';

function useLocalStorage<T>(key: string, defaultValue: T): T {
  const subscribe = (callback: () => void) => {
    window.addEventListener('storage', callback);
    return () => window.removeEventListener('storage', callback);
  };
  
  const getSnapshot = (): T => {
    const raw = localStorage.getItem(key);
    if (raw === null) return defaultValue;
    try {
      return JSON.parse(raw) as T;
    } catch {
      return defaultValue;
    }
  };
  
  const getServerSnapshot = (): T => defaultValue;
  
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}

// Eslatma: getSnapshot har render'da JSON.parse qiladi.
// Agar T object/array bo'lsa, har chaqiriq yangi reference qaytaradi
// va `Object.is` har doim false bo'ladi → infinite loop xavfi.
// Yechim: getSnapshot'da memoization (cache last raw + parsed).
```

</details>

---

## Invariant 1: Render Purity

### Nazariya

**Render Purity Invariant** — komponent funksiyasi **bir xil props/state/context** uchun **bir xil JSX** qaytarishi va **side effect bo'lmasligi** kerak. Bu **eng muhim** Concurrent invariant — qolgan invariant'lar shu fundament ustiga quriladi.

**Pure render** ta'rifi:

1. **Determinizm** — input bir xil bo'lsa, output bir xil. `Math.random`, `Date.now`, `crypto.randomUUID`, `performance.now`, mutable global o'qish — barchasi taqiq.
2. **Side effect yo'q** — `console.log` (Strict Mode 2x ko'rinadi), DOM mutation (`document.title = ...`), network call (`fetch`), `localStorage.setItem`, ref mutation render Phase'da.
3. **Mutation yo'q** — props/state/context immutable, faqat o'qiladi.

NIMA UCHUN **invariant**:

- **Restart safety** — render restart bo'lsa, side effect ikki marta sodir bo'ladi.
- **Strict Mode 2x render** — har komponent ikki marta chaqiriladi (R16.3+), purity buzilsa darhol ko'rinadi.
- **Compiler talabi** — React Compiler (1.0 stable 2025-oktyabr-7, alohida opt-in build vositasi) komponent'ni pure deb hisoblaydi va auto-memoization qiladi. Mutation bo'lsa, Compiler shu komponentni xavfsiz optimize qila olmaydi va uni memoization'dan chiqarib yuboradi (bailout).
- **Bailout** — Reconciler default'da `prevProps === nextProps` reference equality bilan bailout qiladi (`React.memo` HOC esa shallow comparison qiladi). Mutation reference'ni saqlaydi → bailout natijada o'zgarish ko'rinmaydi (cross-ref `04-reconciliation.md`).
- **Concurrent restart** — render davomida yuqori-priority update kelsa, joriy work tashlab yuboriladi va qayta boshlanadi. Side effect bo'lsa — observable change.

QANDAY tekshirish (development):

1. **Strict Mode** — komponent funksiyasini 2x chaqiradi. Mutation qiymatda farq beradi (`renderCount++`).
2. **`eslint-plugin-react-hooks`** — `react-hooks/exhaustive-deps` linter via static analysis.
3. **`eslint-plugin-react-compiler`** — yangi rules-of-react checks (mutation, side effect).
4. **React Compiler** — compile vaqtida static analysis; mutation yoki rules-of-react buzilishi aniqlansa, Compiler shu komponentni optimize qilmasdan tashlab ketadi (bailout) va diagnostika beradi. Manual bypass: `"use no memo"` directive.

Yagona istisno — **lazy initial state**:

```typescript
useState(() => expensiveComputation()); // Faqat 1-marta mount'da
useReducer(reducer, initialArg, init);  // Init function 1-marta
useMemo(() => compute(), [deps]);        // Memo factory deps'ga qarab
```

Bu factory'lar render Phase'da chaqiriladi, lekin **bir marta** (mount yoki deps change), va React ularni **pure deb taxmin** qiladi (Strict Mode 2x chaqiradi).

<details>
<summary><strong>Under the Hood</strong></summary>

Render Purity'ning **observable consequences** (kuzatilishi mumkin oqibatlar):

**1. Memoization break:**

```javascript
// React Compiler ichki algoritmi (oddiy holat)
function CartSummary({ items }) {
  // Compiler analiz qiladi:
  // - items prop, immutable
  // - total computed from items, depends on items only
  // - Cache slot: _c[0]
  
  if (cache[0] !== items) {
    cache[0] = items;
    cache[1] = items.reduce((sum, item) => sum + item.price, 0); // total
  }
  const total = cache[1];
  
  return <div>{total}</div>;
}
```

Agar `items.push('extra')` qilinsa render ichida — `items` reference o'zgarmaganligi sababli Cache slot yangilanmaydi va memoized `total` **eskirgan qiymat**da qotib qoladi. Compiler bunday mutation'ni aniqlasa, komponentni umuman optimize qilmaydi (bailout).

**2. Bailout break:**

```javascript
// Default FunctionComponent bailout (Reconciler)
function shouldBailoutDefault(prevProps, nextProps) {
  return prevProps === nextProps; // === reference equality
  // Mutation bo'lsa — reference saqlanadi → bailout TRUE → re-render skipped
}

// MemoComponent bailout (React.memo HOC)
function shouldBailoutMemo(prevProps, nextProps) {
  return shallowEqual(prevProps, nextProps); // har property `Object.is`
  // Mutation bo'lsa — har property reference bir xil → shallowEqual true → bailout
}

// Ikkala holatda ham mutation natijasi bir xil: yangi qiymat ko'rinmaydi.
```

**3. Concurrent restart amplification:**

```
Render starts (count = 0, side effect: mutate global)
       │
       ▼
counter += 1 → counter = 1
       │
       ▼
[INTERRUPT] — high priority work keldi
       │
       ▼
Restart render (count = 0)
       │
       ▼
counter += 1 → counter = 2  ← bug!
       │
       ▼
Commit: UI 1, counter 2 — inconsistent
```

**Strict Mode disable internal:**

```javascript
// React DEV: console disabled in second render
function disableLogs() {
  prevLog = console.log;
  prevError = console.error;
  console.log = noop;
  console.error = noop;
}

function reenableLogs() {
  console.log = prevLog;
  console.error = prevError;
}

// Konseptual model: ikkinchi (purity-check) chaqiriqda loglar o'chiriladi
function renderComponentDEV(Component, props, isStrictMode) {
  let result = Component(props); // birinchi chaqiriq
  if (isStrictMode) {
    disableLogs();
    try {
      result = Component(props); // ikkinchi chaqiriq, loglar suppressed
    } finally {
      reenableLogs();
    }
  }
  return result;
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Klassik purity bug'lar:

```tsx
// ❌ 1. Module-level mutable
let totalRenders = 0;

function Counter() {
  totalRenders += 1; // ❌ Render side effect
  return <div>Renders: {totalRenders}</div>;
}

// ✅ Effect'da:
import { useEffect, useRef } from 'react';

function Counter() {
  const renderCountRef = useRef(0);
  useEffect(() => {
    renderCountRef.current += 1;
  });
  return <div>Renders: {renderCountRef.current}</div>;
}
```

```tsx
// ❌ 2. Props mutation
interface Order {
  items: string[];
  total: number;
}

function OrderSummary({ order }: { order: Order }) {
  order.items.push('Tax'); // ❌ Mutation
  return <div>{order.items.join(', ')}</div>;
}

// ✅ Immutable copy:
function OrderSummary({ order }: { order: Order }) {
  const itemsWithTax = [...order.items, 'Tax']; // ✅ Yangi array
  return <div>{itemsWithTax.join(', ')}</div>;
}
```

```tsx
// ❌ 3. Date.now / Math.random render Phase'da
function GreetingCard() {
  const greeting = Math.random() > 0.5 ? 'Hello' : 'Hi'; // ❌
  return <h1>{greeting}!</h1>;
}
// Strict Mode 2x render: ikki marta chaqiriladi, ikki farqli greeting,
// hydration mismatch SSR'da.

// ✅ State'da (mount'da bir marta) yoki useMemo:
import { useState } from 'react';

function GreetingCard() {
  const [greeting] = useState(() =>
    Math.random() > 0.5 ? 'Hello' : 'Hi'
  );
  return <h1>{greeting}!</h1>;
}
```

```tsx
// ❌ 4. Document mutation
function PageTitle({ title }: { title: string }) {
  document.title = title; // ❌ Side effect
  return null;
}

// ✅ useEffect:
import { useEffect } from 'react';

function PageTitle({ title }: { title: string }) {
  useEffect(() => {
    document.title = title;
  }, [title]);
  return null;
}

// ✅ Yoki R19 Document Metadata API (cross-ref 37):
function PageTitle({ title }: { title: string }) {
  return <title>{title}</title>;
}
```

```tsx
// ❌ 5. localStorage'dan render Phase'da o'qish
function UserSettings() {
  const theme = localStorage.getItem('theme') ?? 'light'; // ❌
  return <div className={theme}>Settings</div>;
}
// Hydration mismatch SSR'da (server'da localStorage yo'q).

// ✅ useSyncExternalStore (cross-ref 22):
import { useSyncExternalStore } from 'react';

function useTheme() {
  return useSyncExternalStore(
    (cb) => {
      window.addEventListener('storage', cb);
      return () => window.removeEventListener('storage', cb);
    },
    () => localStorage.getItem('theme') ?? 'light',
    () => 'light' // SSR
  );
}

function UserSettings() {
  const theme = useTheme();
  return <div className={theme}>Settings</div>;
}
```

</details>

---

## Invariant 2: State Updates Functional Merge

### Nazariya

**State Updates Functional Merge Invariant** — bir nechta `setState` chaqiriqlari **bir-biriga bog'liq bo'lsa**, **functional update** (`setX(prev => derive(prev))`) ishlatish kerak. Direct value (`setX(newValue)`) eski snapshot'dan derive qilingan bo'lsa, race condition va Concurrent rendering paytida yo'qolgan update'lar paydo bo'lishi mumkin.

NIMA UCHUN — **Snapshot per render** mexanizmidan kelib chiqadi:

1. Har render alohida snapshot — `count` qiymati render davomida o'zgarmaydi (closure orqali capture qilinadi).
2. `setState(count + 1)` — render snapshot'dan `count` ni o'qib, `count + 1` qiymatini queue'ga tushiradi.
3. Bir render ichida bir nechta `setState(count + 1)` — barchasi bir xil `count + 1` qiymati queue'ga tushadi (snapshot bir xil).
4. Concurrent rendering'da — yuqori-priority update kelishi yoki Transition restart paytida update'lar **qayta hisoblanadi**.

Klassik bug:

```tsx
// ❌ Direct value — uchta setCount bir xil natija
function handleClick() {
  setCount(count + 1); // count = 0 → 1
  setCount(count + 1); // count = 0 → 1 (snapshot hali ham 0)
  setCount(count + 1); // count = 0 → 1 (snapshot hali ham 0)
}
// Natija: count = 1 (3 emas)

// ✅ Functional update — har bittasi avvalgi natijadan
function handleClick() {
  setCount((c) => c + 1); // 0 → 1
  setCount((c) => c + 1); // 1 → 2
  setCount((c) => c + 1); // 2 → 3
}
// Natija: count = 3
```

Concurrent rendering'da bu invariant **kuchayadi**:

- **Transition restart** — `useTransition` ichida boshlangan render uzilishi va restart bo'lsa, direct value snapshot eski state'dan o'qiydi. Functional update **commit paytidagi state**'dan o'qiydi.
- **Async setState** — setTimeout, Promise, useEffect ichidagi setState — closure'dagi old state ushlab qoladi (stale closure bug). Functional update bu muammoni yengadi.
- **Lane priority skip** — yuqori-priority update keladi, joriy update keyinroq ishlanadi. Direct value avvalgi snapshot'dan, functional **joriy state**'dan derive.

NIMA QILISH:

- **Bog'liq state**: `setX(prev => derive(prev))`.
- **Mustaqil qiymat** (form input, controlled): `setX(newValue)` OK — joriy state'dan bog'liq emas.
- **Bog'liq state'lar guruhi**: `useReducer` ko'pincha aniqroq (cross-ref `20-usereducer.md`).

Reducer pattern functional update'ning umumlashtirilishi:
```typescript
dispatch({ type: 'INCREMENT' });
// reducer ichida: (state, action) => ({ ...state, count: state.count + 1 })
```

<details>
<summary><strong>Under the Hood</strong></summary>

Update queue circular linked list (cross-ref `12-state-and-usestate.md`):

```javascript
// React internal
function dispatchSetState(fiber, queue, action) {
  const update = {
    lane,
    action,         // direct value YOKI function
    next: null,
  };
  
  // Append to circular linked list
  if (queue.pending === null) {
    update.next = update;
  } else {
    update.next = queue.pending.next;
    queue.pending.next = update;
  }
  queue.pending = update;
  
  // Eager bailout: queue bo'sh bo'lsa, React yangi state'ni shu yerda
  // hisoblab ko'radi (basicStateReducer functional/direct action'ni o'zi
  // ajratadi). Natija joriy state bilan Object.is bo'yicha teng bo'lsa —
  // re-render rejalashtirilmaydi.
  if (fiber.lanes === NoLanes) {
    const eagerState = basicStateReducer(currentState, action);
    if (Object.is(eagerState, currentState)) {
      return; // Bailout, no re-render
    }
  }
  
  scheduleUpdateOnFiber(fiber, lane);
}

function processUpdateQueue() {
  // Render Phase: queue'ni iterate qilamiz
  let baseState = currentState;
  let update = queue.pending.next;
  
  do {
    if (typeof update.action === 'function') {
      // Functional: avvalgi natijadan derive
      baseState = update.action(baseState);
    } else {
      // Direct: snapshot'dan o'qilgan qiymat
      baseState = update.action;
    }
    update = update.next;
  } while (update !== queue.pending.next);
  
  return baseState;
}
```

**Concurrent restart va functional update'lar**:

```
Render at lane=Transition (count=0)
       │
       ├─ setCount(c => c + 1) → queue: [fn1]
       ├─ setCount(c => c + 1) → queue: [fn1, fn2]
       │
       ▼
Process queue: 0 → fn1(0)=1 → fn2(1)=2 → newState=2
       │
       ▼
[INTERRUPT] — Sync update: setCount(10)
       │
       ▼
Sync render: count=0 → setCount(10) → count=10 → COMMIT
       │
       ▼
Restart Transition render at lane=Transition
       │
       ▼
Process queue from baseState=10: fn1(10)=11 → fn2(11)=12 → newState=12 → COMMIT
```

**Direct value** bo'lsa: queue'da `[1, 1]` saqlangan bo'lardi, restart paytida `count=10 → 1 → 1 → 1` — sync update'ni **bekor qiladi** (bug).

Functional bilan: queue'da `[fn1, fn2]` saqlangan, restart paytida `count=10 → 11 → 12` — sync update saqlanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Async setState — stale closure bug:

```tsx
import { useState } from 'react';

// ❌ Direct value — async context'da stale closure
function ClickCounter() {
  const [count, setCount] = useState(0);
  
  function handleClick() {
    setTimeout(() => {
      setCount(count + 1); // ❌ count yopilgan render snapshot'da
    }, 1000);
  }
  
  return <button onClick={handleClick}>Count: {count}</button>;
}
// Tez bosish: count=0, 5 ta click 1 sekundda
// 1 sekunddan keyin: 5 ta setCount(0+1) = 1
// Natija: count=1 (5 emas)

// ✅ Functional update
function ClickCounter() {
  const [count, setCount] = useState(0);
  
  function handleClick() {
    setTimeout(() => {
      setCount((c) => c + 1); // ✅ Latest state
    }, 1000);
  }
  
  return <button onClick={handleClick}>Count: {count}</button>;
}
// 5 ta click → 5 ta setCount(c => c+1) = c+1, c+1, c+1, c+1, c+1
// Natija: count=5
```

useEffect interval — stale closure'dan saqlanish:

```tsx
import { useState, useEffect } from 'react';

// ❌ count deps siz — closure stale
function Timer() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // ❌ count har doim 0
    }, 1000);
    return () => clearInterval(id);
  }, []); // count yo'q deps'da
  
  return <div>{count}</div>;
}
// Birinchi setInterval'da count=0 yopilgan
// 1, 2, 3 sekundlarda: setCount(0+1) = setCount(1) — har doim 1
// Natija: count har doim 1.

// ✅ Functional update — deps shart emas
function Timer() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      setCount((c) => c + 1); // ✅ Latest c
    }, 1000);
    return () => clearInterval(id);
  }, []); // OK - functional update deps siz
  
  return <div>{count}</div>;
}
// Natija: count 1, 2, 3, ... har sekund.
```

useReducer — bog'liq state'lar guruhi:

```tsx
import { useReducer } from 'react';

interface CartItem {
  id: string;
  qty: number;
  price: number;
}

interface CartState {
  items: CartItem[];
  total: number;
}

type CartAction =
  | { type: 'ADD'; id: string; price: number }
  | { type: 'REMOVE'; id: string }
  | { type: 'CLEAR' };

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'ADD': {
      const existing = state.items.find((i) => i.id === action.id);
      const items = existing
        ? state.items.map((i) =>
            i.id === action.id ? { ...i, qty: i.qty + 1 } : i
          )
        : [...state.items, { id: action.id, qty: 1, price: action.price }];
      return {
        items,
        total: state.total + action.price,
      };
    }
    case 'REMOVE': {
      const item = state.items.find((i) => i.id === action.id);
      if (!item) return state;
      return {
        items: state.items.filter((i) => i.id !== action.id),
        total: state.total - item.price * item.qty,
      };
    }
    case 'CLEAR':
      return { items: [], total: 0 };
    default:
      return state;
  }
}

function ShoppingCart() {
  const [cart, dispatch] = useReducer(cartReducer, { items: [], total: 0 });
  
  // Bog'liq update'lar — reducer'da atomically
  function handleAdd() {
    dispatch({ type: 'ADD', id: 'prod-1', price: 25 });
    dispatch({ type: 'ADD', id: 'prod-2', price: 15 });
    dispatch({ type: 'ADD', id: 'prod-1', price: 25 }); // qty++
  }
  
  return (
    <div>
      Total: {cart.total}, Items: {cart.items.length}
      <button onClick={handleAdd}>Add 3</button>
    </div>
  );
}
```

</details>

---

## Invariant 3: Effect Cleanup va Setup Pair Symmetry

### Nazariya

**Effect Cleanup va Setup Pair Symmetry Invariant** — `useEffect` (va `useLayoutEffect`/`useInsertionEffect`) ichidagi setup va cleanup **simmetrik** bo'lishi shart. Setup'da yaratilgan barcha resource cleanup'da to'liq tozalanishi kerak. Bu invariant Concurrent rendering'da **majburiy** chunki effect'lar **ko'p marta** chaqirilishi mumkin (Strict Mode 2x cycle, deps change, unmount).

Setup va cleanup pair'i:

| Setup operation | Cleanup operation |
|-----------------|-------------------|
| `subscribe()` | `unsubscribe()` |
| `addEventListener` | `removeEventListener` |
| `setInterval` | `clearInterval` |
| `setTimeout` | `clearTimeout` |
| `fetch + signal` | `controller.abort()` |
| `MutationObserver.observe` | `observer.disconnect()` |
| WebSocket open | `socket.close()` |
| Animation start | Animation cancel |
| DOM modification | DOM revert |
| Singleton lock | Singleton release |

NIMA UCHUN simmetrik bo'lishi shart:

1. **Strict Mode 2x cycle (R18+)** — har mount'da setup → cleanup → setup. Cleanup yo'q yoki noto'liq bo'lsa — duplicate subscription, double interval, racing fetch.
2. **Deps change** — useEffect dep array o'zgarsa cleanup → setup. Cleanup yo'q bo'lsa — eski subscription saqlanadi (memory leak).
3. **Unmount** — komponent yo'q qilinsa cleanup. Cleanup yo'q bo'lsa — listener saqlanadi va o'lgan komponentga ishora qiladi (memory leak).
4. **Concurrent restart** — render restart bo'lganda effect chaqirilmaydi (effect Commit'dan keyin), lekin agar effect bajarilgan bo'lsa va commit bekor qilingan bo'lsa, cleanup hech qachon chaqirilmaydi (rare edge case).

QANDAY tekshirish:

- **Strict Mode** — `mount → cleanup → mount` cycle ko'rinadi.
- **Manual unmount/remount** — komponent'ni 5-10 marta mount/unmount qilib monitoring (DevTools profiler).
- **Memory profile** — Chrome DevTools heap snapshot, listener count, retained DOM nodes.

Cleanup yozmaslik anti-pattern'i:

```tsx
// ❌ Cleanup yo'q
useEffect(() => {
  window.addEventListener('resize', handler);
  // No return → leak
}, []);

// ✅ Cleanup
useEffect(() => {
  window.addEventListener('resize', handler);
  return () => window.removeEventListener('resize', handler);
}, []);
```

**Idempotency invariant** — cleanup + setup birga bajarilganda effect "neutral" bo'lishi kerak (state hech qachon o'zgartirilmagandek). Misollar:

- ✅ `subscribe → unsubscribe` — server'da bir aktiv subscription qoladi.
- ✅ `addEventListener → removeEventListener` — DOM'da bir listener.
- ❌ `analytics.track('view')` — cleanup nima qilishi kerak? `untrack` API yo'q. Bu sababdan analytics useEffect'da bo'lishi yomon (cross-ref `16-useeffect.md` "You Might Not Need an Effect").

Effect Phase tartib (R18+):

```
Commit Phase (Mutation sub-phase complete)
       │
       ▼
Layout Phase: useLayoutEffect cleanup → setup (sync, paint'dan oldin)
       │
       ▼
Browser Paint
       │
       ▼
Passive Phase: useEffect cleanup → setup (async via MessageChannel)
   ↑ R18 atomic separation: avval BARCHA cleanup, keyin BARCHA setup
```

R18 atomic separation — agar uchta komponent useEffect ishlatsa, R18 avval uchta cleanup'ni bajaradi, keyin uchta setup'ni. R17'da har komponent uchun `cleanup → setup` ketma-ket. Bu o'zgarish — Concurrent re-mount mexanikasi uchun (Offscreen API).

<details>
<summary><strong>Under the Hood</strong></summary>

Effect chain Fiber.updateQueue (cross-ref `15-hooks-fundamentals.md`):

```javascript
// Fiber struct (oddiylashtirilgan)
{
  memoizedState: Hook | null, // Hook linked list
  updateQueue: {
    lastEffect: Effect | null, // Effect chain (circular)
  }
}

// Effect struct
{
  tag: HookPassive | HookLayout | HookInsertion,
  create: () => destroy?,  // Setup function
  destroy: () => void,     // Cleanup function (tagilgan)
  deps: Array | null,
  next: Effect,            // Circular linked list
}
```

R18 atomic effect bajarilishi:

```javascript
// commitPassiveMountEffects bottom-up DFS
function commitPassiveUnmountEffects(finishedRoot) {
  // 1-pass: avval BARCHA cleanup (bottom-up)
  for (const effect of allPassiveEffects) {
    if (effect.destroy !== undefined) {
      const destroy = effect.destroy;
      effect.destroy = undefined;
      destroy(); // Cleanup
    }
  }
}

function commitPassiveMountEffects(finishedRoot) {
  // 2-pass: BARCHA setup
  for (const effect of allPassiveEffects) {
    effect.destroy = effect.create(); // Setup, save destroy
  }
}
```

R17 da har effect uchun `cleanup → setup` synchron bajarilardi:

```javascript
// R17 (oddiylashtirilgan)
function commitHookEffectListMount() {
  for (const effect of allEffects) {
    if (effect.destroy) effect.destroy(); // Cleanup
    effect.destroy = effect.create();     // Setup
  }
}
```

R18'da atomic separation — Offscreen API uchun, komponent "yashiringan" bo'lsa cleanup bajariladi va keyinroq mount paytida setup, lekin atomic.

**Cleanup'siz subscription oqibati:**

```javascript
// Component A mount
const subA = subscribe(handlerA); // listeners.add(handlerA)

// Component A re-mount (Strict Mode)
// No cleanup
const subA2 = subscribe(handlerA); // listeners.add(handlerA) — AGAIN
// Now: listeners has handlerA TWICE

// Store change → handlerA called TWICE
// State updated TWICE → can cause double events
```

**Memory leak — sifat-darajadagi ta'sir:**

Cleanup'siz subscription/listener'larning quyidagi resurslari saqlanadi:

- **Closure scope** — handler funksiyasi yopilgan barcha local'lar (state, props, ref'lar). Komponent yopilgach ham GC qila olmaydi, chunki listener registry to'plamida ushlanib turadi.
- **Listener entry** — store/event-target ichidagi listener ro'yxat slot'i (Set/Array entry).
- **Fiber retention** — handler closure orqali Fiber tree'ning bir qismi (parent chain) reachable bo'lib qoladi → GC ushlamaydi.

Ko'p mount/unmount cycle (masalan, navigation orqali bir komponent qayta-qayta yaratilsa) cleanupsiz holatda heap kattalashadi va listener registry sekinlashadi — har store change handler'larning **sonidan ortib boruvchi soni** chaqiriladi. Aniq qiymatlar bundle, scope o'lchami va store implementation'iga bog'liq.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Subscription pattern:

```tsx
import { useEffect, useState } from 'react';

interface Notification {
  id: string;
  message: string;
}

function NotificationFeed({ userId }: { userId: string }) {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  
  useEffect(() => {
    const ws = new WebSocket(`wss://api.example.com/notifications/${userId}`);
    
    ws.onmessage = (event) => {
      const notif = JSON.parse(event.data) as Notification;
      setNotifications((prev) => [...prev, notif]);
    };
    
    return () => {
      ws.close(); // ✅ Cleanup symmetry
    };
  }, [userId]);
  
  return (
    <ul>
      {notifications.map((n) => (
        <li key={n.id}>{n.message}</li>
      ))}
    </ul>
  );
}
```

Event listener — cleanup symmetry:

```tsx
import { useEffect, useState } from 'react';

function WindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });
  
  useEffect(() => {
    function handleResize() {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    }
    
    window.addEventListener('resize', handleResize);
    
    return () => {
      window.removeEventListener('resize', handleResize); // ✅ Bir xil reference
    };
  }, []);
  
  return <div>{size.width} × {size.height}</div>;
}
```

Fetch + AbortController:

```tsx
import { useEffect, useState } from 'react';

interface User {
  id: string;
  name: string;
}

function UserCard({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    const controller = new AbortController();
    
    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then((data: User) => setUser(data))
      .catch((err: unknown) => {
        if (err instanceof Error && err.name === 'AbortError') return; // Expected
        if (err instanceof Error) setError(err);
      });
    
    return () => controller.abort(); // ✅ Cancel inflight
  }, [userId]);
  
  if (error) return <div>Error: {error.message}</div>;
  if (!user) return <div>Loading...</div>;
  return <div>{user.name}</div>;
}
```

MutationObserver — DOM modification cleanup:

```tsx
import { useEffect, useRef, useState } from 'react';

function ContentEditable() {
  const ref = useRef<HTMLDivElement>(null);
  const [content, setContent] = useState('');
  
  useEffect(() => {
    if (!ref.current) return;
    
    const observer = new MutationObserver(() => {
      setContent(ref.current?.textContent ?? '');
    });
    
    observer.observe(ref.current, {
      childList: true,
      characterData: true,
      subtree: true,
    });
    
    return () => observer.disconnect(); // ✅ Cleanup
  }, []);
  
  return (
    <div>
      <div ref={ref} contentEditable suppressContentEditableWarning>
        Edit me
      </div>
      <p>Content length: {content.length}</p>
    </div>
  );
}
```

</details>

---

## Invariant 4: External Subscription Consistency

### Nazariya

**External Subscription Consistency Invariant** — external store'ga subscribe qilinganda, har retry/restart paytida **bir xil snapshot** ko'rinishi kerak. Bu Concurrent rendering'da fundamental — render uziladi va qayta boshlanadi (Transition restart, Suspense retry, Error Boundary recovery), va har retry'da subscription "fresh" bo'lishi kerak (bir xil source, bir xil format, bir xil semantics).

NIMA UCHUN bu invariant zarur:

1. **Concurrent restart** — render uzilib qayta boshlansa, store snapshot eski (interrupt vaqtidagi) yoki yangi (latest) bo'lishi mumkin. `useSyncExternalStore` bu mexanizm'ni rasmiylashtiradi.
2. **Suspense retry** — component throw promise → fallback → resolve → retry. Retry render'ning o'zi subscription'ni qayta o'rnatmaydi (subscribe passive effect'da, render retry'da qayta ishlamaydi), lekin store'dan yangi snapshot olinadi.
3. **Error Boundary recovery** — komponent throw → boundary fallback → reset → retry. Retry'da fresh snapshot kerak.
4. **Tearing prevention** — barcha komponent'lar bir xil snapshot ko'rishi kerak (aks holda React sinxron re-render bilan tuzatadi).

QANDAY ishlatish:

- **Library author**: `useSyncExternalStore` (cross-ref `22-concurrent-hooks.md`) — Redux, Zustand, MobX, Jotai shu primitive'dan foydalanadi.
- **Application author**: ko'pincha library API ishlatiladi (`useSelector`, `useStore`), `useSyncExternalStore` to'g'ridan-to'g'ri kamdan-kam.
- **Custom store** — kichik global state uchun `useSyncExternalStore` yetarli.

`getSnapshot` qoidalari:

1. **Pure** — har chaqiriqda bir xil natija (state o'zgarmaganda).
2. **Stable reference** — agar state o'zgarmagan bo'lsa, snapshot reference o'zgarmasligi kerak (Object.is true). Aks holda — har render'da yangi reference → infinite loop xavfi.
3. **Synchronous** — Promise qaytarmaydi, async iterator emas. State'ning **joriy qiymati**.
4. **Error-free** — exception throw qilmaydi (yoki throw qilsa — render restart).

`subscribe` qoidalari:

1. **Returns unsubscribe function** — cleanup symmetry.
2. **Stable identity** — har render'da yangi reference qaytarish anti-pattern (unsubscribe → resubscribe har render).
3. **Multiple subscribers** — bir store'ga ko'p komponent subscribe qila oladi.

`getServerSnapshot` (SSR):

1. **Deterministic** — server'da tushunarli qiymat (default state, browser API'siz).
2. **No browser globals** — `window`, `document`, `localStorage` yo'q.
3. **Hydration mismatch** — agar server snapshot va client first snapshot farq qilsa, hydration warning paydo bo'ladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`useSyncExternalStore` Concurrent integration:

```javascript
// React internal
function mountSyncExternalStore(subscribe, getSnapshot, getServerSnapshot) {
  const fiber = currentlyRenderingFiber;
  const root = getWorkInProgressRoot();
  
  let snapshot;
  if (getServerSnapshot && isHydrating) {
    snapshot = getServerSnapshot();
  } else {
    snapshot = getSnapshot();
  }
  
  const hook = mountWorkInProgressHook();
  hook.memoizedState = snapshot;
  
  const inst = {
    value: snapshot,
    getSnapshot,
  };
  hook.queue = inst;
  
  // Subscribe via passive effect
  mountEffect(() => subscribeToStore(fiber, inst, subscribe), [subscribe]);
  
  // Sync check via update effect
  pushEffect(
    HookHasEffect | HookPassive,
    () => updateStoreInstance(fiber, inst, snapshot, getSnapshot)
  );
  
  return snapshot;
}

function updateSyncExternalStore(subscribe, getSnapshot, getServerSnapshot) {
  const fiber = currentlyRenderingFiber;
  const hook = updateWorkInProgressHook();
  const inst = hook.queue; // mount paytida saqlangan { value, getSnapshot }
  
  // Read latest snapshot
  const newSnapshot = getSnapshot();
  
  // Tearing check: avvalgi snapshot bilan solishtirish
  if (!Object.is(hook.memoizedState, newSnapshot)) {
    hook.memoizedState = newSnapshot;
    markWorkInProgressReceivedUpdate(); // Bu force re-render
  }
  
  // Concurrent: render davomida store o'zgarmaganligini tekshirish
  if (
    inst.getSnapshot !== getSnapshot ||
    !Object.is(inst.value, newSnapshot)
  ) {
    inst.value = newSnapshot;
    inst.getSnapshot = getSnapshot;
    
    // Render uziluvchi (blocking/transition) lane'da bo'lsa, commit oldidagi
    // consistency check uchun shu store ro'yxatga qo'shiladi — mos kelmasa
    // forceStoreRerender sinxron re-render trigger qiladi.
    if (includesBlockingLane(root, renderLanes) ||
        includesTransitionLane(root, renderLanes)) {
      pushStoreConsistencyCheck(fiber, getSnapshot, newSnapshot);
    }
  }
  
  return newSnapshot;
}
```

**Tearing detection** render va commit chegarasida:

```
Render at lane=Transition
       │
       ▼
useSyncExternalStore → snapshot = 0
       │
       ▼
[YIELD] — user input
       │
       ▼
External: store.set(1)
       │
       ▼
Resume render at lane=Transition
       │
       ▼
useSyncExternalStore → snapshot = 1 (yangi)
       │
       ▼
React: tear detected → forceStoreRerender (sinxron re-render)
       │
       ▼
Sync re-render with snapshot=1, all components see 1
```

**Snapshot stability requirement:**

```javascript
// ❌ Anti-pattern: har chaqiriqda yangi object
const getSnapshot = () => ({ items: store.items, total: store.total });
// Object.is({...}, {...}) === false har doim
// → har render'da snapshot mismatch → infinite loop

// ✅ Stable reference (memoized inside store)
const getSnapshot = () => store.state; // Same reference until store.set()
// Object.is(store.state, store.state) === true
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Custom external store:

```tsx
import { useSyncExternalStore } from 'react';

// Store implementation
function createStore<T>(initialState: T) {
  let state = initialState;
  const listeners = new Set<() => void>();
  
  return {
    getState: () => state,
    setState: (next: T | ((prev: T) => T)) => {
      state = typeof next === 'function'
        ? (next as (prev: T) => T)(state)
        : next;
      listeners.forEach((l) => l());
    },
    subscribe: (listener: () => void) => {
      listeners.add(listener);
      return () => {
        listeners.delete(listener);
      };
    },
  };
}

// Usage
interface Counter {
  count: number;
}

const counterStore = createStore<Counter>({ count: 0 });

function useCounter(): Counter {
  return useSyncExternalStore(
    counterStore.subscribe,
    counterStore.getState,
    () => ({ count: 0 }) // SSR snapshot
  );
}

function CounterDisplay() {
  const { count } = useCounter();
  return <span>{count}</span>;
}

function CounterButton() {
  return (
    <button
      onClick={() =>
        counterStore.setState((s) => ({ count: s.count + 1 }))
      }
    >
      Increment
    </button>
  );
}
```

Selector pattern — partial subscription:

```tsx
import { useSyncExternalStore } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
  preferences: { theme: 'light' | 'dark'; lang: string };
}

const userStore = createStore<User>({
  id: '',
  name: '',
  email: '',
  preferences: { theme: 'light', lang: 'en' },
});

function useUserSelector<T>(selector: (state: User) => T): T {
  return useSyncExternalStore(
    userStore.subscribe,
    () => selector(userStore.getState()),
    () => selector({ id: '', name: '', email: '', preferences: { theme: 'light', lang: 'en' } })
  );
}

// Komponent faqat o'zi tanlagan slice'ga subscribe
function ThemeBadge() {
  const theme = useUserSelector((s) => s.preferences.theme);
  return <span className={`theme-${theme}`}>Theme: {theme}</span>;
}

// ❌ Eslatma: bu naive selector — har render selector ishlaydi va string return qiladi.
// Concurrent re-tearing detection uchun snapshot stability kerak,
// agar selector primitive qaytarsa OK, object qaytarsa — useMemo + manual cache.
```

Browser API integration:

```tsx
import { useSyncExternalStore } from 'react';

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
    () => true // SSR: assume online
  );
}

function ConnectionStatus() {
  const isOnline = useOnlineStatus();
  return (
    <div className={isOnline ? 'online' : 'offline'}>
      {isOnline ? 'Online' : 'Offline'}
    </div>
  );
}
```

</details>

---

## Anti-Pattern 1: Render-Time Mutation

### Nazariya

**Render-time mutation** — render Phase ichida external state, props, refs, yoki module-level variable'ga yozish. Bu Render Purity invariant'ining eng aniq buzilishi va Concurrent rendering'da **observable bug** keltirib chiqaradi.

Mutatsiya turlari va oqibatlari:

| Mutation source | Dev oqibat | Prod oqibat | Concurrent oqibat |
|-----------------|-----------|-------------|-------------------|
| Module-level variable | Strict Mode 2x render farq | Bug ko'rinmaydi | Restart 2x mutation |
| Props mutation | Bailout buziladi | Re-render skipped | Snapshot inconsistency |
| State mutation (`state.x = y`) | Bailout (Object.is true) | Render skipped | Same |
| DOM mutation (`document.title`) | Strict Mode 2x set | OK production | Restart 2x set |
| Ref `.current` write | Strict Mode 2x ko'rinishi | Race | Race |
| `localStorage.setItem` | Strict Mode 2x write | Stale data | Restart 2x write |
| Counter/timer Mut. | 2x increment | Bug ko'rinmaydi | 2x increment |

NIMA UCHUN Concurrent'da kuchayadi:

- **Restart** — render uzilib qayta boshlansa, mutation **2x sodir bo'ladi**.
- **Strict Mode 2x render** — har render Strict Mode'da 2x. Mutation 2x ko'rinadi development'da.
- **Bailout** — Reconciler shallow comparison'siga asoslanadi. Mutation reference saqlaydi → bailout TRUE → re-render skipped → mutation **ko'rinmaydi**.

Aniqlash usullari:

1. **Strict Mode'da test** — mutation 2x ko'rinadi.
2. **`eslint-plugin-react-compiler`** — static analysis (React versiyasidan mustaqil, Rules of React linter).
3. **DevTools Profiler** — yo'qolgan re-render'lar (bailout silently skip).

Standart yechim — **mutation'ni tashqariga olib chiqish**:

- State'da: `setState({...prev, x: y})`.
- Effect'da: `useEffect(() => { mutate(); }, [deps])`.
- Event handler'da: `onClick={() => mutate()}`.
- Module-level read-only: `const config = { ... }; export default config;` (mutate qilinmaydi).

<details>
<summary><strong>Under the Hood</strong></summary>

Reconciler bailout va mutation. `beginWork` ichida bailout qarori props reference (`===`, Object.is emas) va pending update yo'qligi bo'yicha qabul qilinadi:

```javascript
// beginWork (oddiylashtirilgan)
function beginWork(current, workInProgress, renderLanes) {
  const oldProps = current?.memoizedProps;
  const newProps = workInProgress.pendingProps;

  if (
    oldProps === newProps &&              // props reference === (Object.is emas)
    !hasContextChanged() &&
    !checkScheduledUpdateOrContext(current, renderLanes) // pending update yo'q
  ) {
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }

  // o'zgarish bor → komponentni qayta render qilamiz
  return updateFunctionComponent(current, workInProgress, ...);
}

function bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes) {
  // childLanes ham bo'sh bo'lsa — butun subtree skip
  if ((workInProgress.childLanes & renderLanes) === NoLanes) {
    return null;
  }
  // aks holda bolalarni clone qilib pastga tushamiz
  cloneChildFibers(current, workInProgress);
  return workInProgress.child;
}
```

Agar props mutation bo'lsa:

```javascript
const items = ['a', 'b'];
<List items={items} />
// items.push('c') — same reference
<List items={items} />
// prev === next → bailout TRUE (props uses ===)
// items.length === 3, lekin <List> render qilinmaydi
// UI: faqat 'a', 'b' ko'rinadi
```

State mutation va eager bailout:

```javascript
const [user, setUser] = useState({ name: 'Ali', age: 25 });

function handleEdit() {
  user.age = 26; // Mutation
  setUser(user); // Same reference
  // Eager bailout: Object.is(prev, next) === true → setState skipped
  // UI: hali ham age=25
}
```

Concurrent restart va module-level mutation:

```javascript
let api_calls = 0;

function ProductList() {
  api_calls += 1; // Module mutation
  fetch('/api/products'); // Render side effect
  // ...
}

// Render starts (api_calls = 0 → 1)
// [YIELD]
// User input: high priority work
// Restart render (api_calls = 1 → 2)
// fetch called 2x
// Network: 2 requests
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Counter bug:

```tsx
// ❌ Module-level counter — Strict Mode 2x render
let totalRenders = 0;

function App() {
  totalRenders += 1; // ❌
  return <div>{totalRenders}</div>;
}
// Strict Mode dev: render 1 va 2, totalRenders=2 (UI: "2")
// Real production: render 1, totalRenders=1 (UI: "1")
// Inconsistent.

// ✅ State'da:
import { useState, useEffect } from 'react';

function App() {
  const [renderCount, setRenderCount] = useState(0);
  
  useEffect(() => {
    setRenderCount((c) => c + 1);
    // Strict Mode 2x effect cycle: setup → cleanup → setup
    // Functional update + cleanup'siz: count 2x increment
    // Production: 1x
  });
  
  return <div>{renderCount}</div>;
}
// Bu ham bug — useEffect bilan har render counter increment cheksiz loop.
// Counter renderlari uchun useRef + useEffect:

import { useRef, useEffect } from 'react';

function App() {
  const renderCountRef = useRef(0);
  
  useEffect(() => {
    renderCountRef.current += 1;
  });
  
  return <div>{renderCountRef.current}</div>;
}
// Display: oxirgi commit'dagi qiymat (effect avval bajarilgan).
// Strict Mode 2x effect: ref.current 2x increment.
```

Props mutation:

```tsx
interface UserData {
  name: string;
  roles: string[];
}

// ❌ Props mutation
function UserCard({ user }: { user: UserData }) {
  user.roles.push('admin'); // ❌ Props mutation
  return <div>{user.name}: {user.roles.join(', ')}</div>;
}

// Bailout consequence:
// 1. <UserCard user={user} /> rendered, user.roles = ['user', 'admin']
// 2. parent setState() — user reference unchanged
// 3. Reconciler: Object.is(prev, next) === true → bailout
// 4. UserCard re-render qilinmaydi, lekin user.roles endi ['user', 'admin', 'admin']
// 5. UI hali ham 'user, admin' ko'rsatadi (bailout natijasida)

// ✅ Immutable derive
function UserCard({ user }: { user: UserData }) {
  const allRoles = [...user.roles, 'admin']; // ✅ Yangi array
  return <div>{user.name}: {allRoles.join(', ')}</div>;
}
```

DOM mutation:

```tsx
// ❌ Render Phase'da document.title
function PageTitle({ title }: { title: string }) {
  document.title = title; // ❌
  return null;
}

// Strict Mode 2x render: title 2x set (oxirgisi qoladi, lekin observable).
// Restart: title set ikki marta navbatda.

// ✅ useEffect:
import { useEffect } from 'react';

function PageTitle({ title }: { title: string }) {
  useEffect(() => {
    document.title = title;
    return () => {
      document.title = 'Default'; // Optional cleanup
    };
  }, [title]);
  return null;
}

// ✅ Yoki R19 Document Metadata (cross-ref 37):
function PageTitle({ title }: { title: string }) {
  return <title>{title}</title>;
}
```

localStorage write:

```tsx
// ❌ Render'da localStorage write
function ThemeProvider({ theme }: { theme: string }) {
  localStorage.setItem('theme', theme); // ❌
  return <div className={theme}>...</div>;
}

// Strict Mode 2x: setItem 2x (idempotent, lekin restart'da 2x I/O).

// ✅ useEffect:
import { useEffect } from 'react';

function ThemeProvider({ theme, children }: { theme: string; children: React.ReactNode }) {
  useEffect(() => {
    localStorage.setItem('theme', theme);
  }, [theme]);
  
  return <div className={theme}>{children}</div>;
}
```

</details>

---

## Anti-Pattern 2: setState Callback Side Effects

### Nazariya

**setState callback side effects** — `setState(prev => { /* side effect */; return newState; })` ichida side effect bajarish. Functional update callback **pure reducer** bo'lishi shart — `(prev) => next`, side effect yo'q.

NIMA UCHUN bu anti-pattern:

1. **Callback ko'p marta chaqirilishi mumkin** — Concurrent restart, Strict Mode, retry. React side effect'ning bir necha marta sodir bo'lishini kutmaydi.
2. **Eager bailout vaqtida ham chaqiriladi** — queue bo'sh bo'lsa, React update'ni `dispatchSetState`'ning o'zida hisoblab ko'radi (eager evaluation). Functional updater bo'lsa, funksiya shu yerda — render'dan tashqarida, kutilmagan paytda — chaqiriladi. Ichidagi side effect ham shunda sodir bo'ladi.
3. **Snapshot consistency** — reducer/updater pure bo'lsa, har retry'da bir xil natija. Side effect bo'lsa — observable inconsistency.

Anti-pattern misollari:

```tsx
// ❌ Side effect functional update ichida
setCount((prev) => {
  console.log('Setting count'); // ❌ Strict Mode 2x render: 2x log
  fetch('/api/log', { method: 'POST' }); // ❌ Network 2x
  document.title = `Count: ${prev + 1}`; // ❌ DOM mutation
  return prev + 1;
});

// ❌ Async logic
setItems((prev) => {
  const next = [...prev];
  fetch('/api/items').then((r) => r.json()).then((d) => next.push(...d));
  // ❌ async ishlamaydi — next return qilinadi promise resolve'dan oldin
  return next;
});
```

To'g'ri pattern — **side effect tashqarida**:

```tsx
// ✅ Side effect oldindan, setState faqat data update
async function handleClick() {
  await fetch('/api/log', { method: 'POST' }); // Side effect
  setCount((prev) => prev + 1); // Pure update
}

// ✅ Yoki useEffect ichida side effect, setState qaytadan chaqiriladi
useEffect(() => {
  document.title = `Count: ${count}`;
}, [count]);
```

QANDAY aniqlash:

- **Strict Mode** — callback 2x chaqiriladi, side effect 2x ko'rinadi.
- **`eslint-plugin-react-compiler`** — pure reducer rule (React versiyasidan mustaqil).

<details>
<summary><strong>Under the Hood</strong></summary>

Functional update queue iteration (cross-ref `12-state-and-usestate.md`):

```javascript
// Render Phase: queue iterate
function processUpdateQueue(queue, oldState) {
  let baseState = oldState;
  let update = queue.pending.next; // Birinchi update
  
  do {
    if (typeof update.action === 'function') {
      // Functional update — chaqiriladi
      baseState = update.action(baseState);
      // ❌ Agar action ichida side effect bo'lsa — har iteration'da
    } else {
      baseState = update.action;
    }
    update = update.next;
  } while (update !== queue.pending.next);
  
  return baseState;
}
```

**Concurrent restart impact:**

```
Render at lane=Default
       │
       ▼
processUpdateQueue iterates 3 functional updates
       │
       ▼
[Side effect bajarildi 3 marta]
       │
       ▼
[INTERRUPT] — Sync update keldi
       │
       ▼
Restart render
       │
       ▼
processUpdateQueue iterates 3 functional updates (qaytadan)
       │
       ▼
[Side effect bajarildi 3 marta YANA]
       │
       ▼
Total: 6 side effect (kutilgan: 3 yoki 1)
```

**Eager bailout consequences:**

```javascript
// dispatchSetState eager check (queue bo'sh bo'lganda)
if (fiber.lanes === NoLanes) {
  // basicStateReducer functional action'ni o'zi chaqiradi:
  //   typeof action === 'function' ? action(currentState) : action
  const eagerState = basicStateReducer(currentState, action);
  if (Object.is(eagerState, currentState)) {
    return; // No re-render
  }
}

// Functional updater bo'lsa, eager evaluation uni AYNAN shu yerda chaqiradi —
// render'dan tashqarida. Ichidagi side effect render boshlanmasdan oldin
// sodir bo'ladi. Render Phase'da queue qayta iteratsiya qilinsa — yana chaqiriladi.
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Console logging — Strict Mode 2x ko'rinishi:

```tsx
import { useState } from 'react';

// ❌ Logging functional update'da
function Counter() {
  const [count, setCount] = useState(0);
  
  function handleClick() {
    setCount((prev) => {
      console.log('Increment from', prev); // ❌ Strict Mode 2x render: 2x log
      return prev + 1;
    });
  }
  
  return <button onClick={handleClick}>{count}</button>;
}

// ✅ Logging tashqarida:
function Counter() {
  const [count, setCount] = useState(0);
  
  function handleClick() {
    console.log('Increment from', count); // ✅ Bir marta event handler'da
    setCount((prev) => prev + 1);
  }
  
  return <button onClick={handleClick}>{count}</button>;
}
```

Network call — 2x request:

```tsx
import { useState } from 'react';

// ❌ Fetch functional update'da
function ItemsList() {
  const [items, setItems] = useState<string[]>([]);
  
  function handleAdd() {
    setItems((prev) => {
      fetch('/api/items', { method: 'POST', body: JSON.stringify({ count: prev.length + 1 }) });
      // ❌ Strict Mode: 2x POST
      // ❌ Concurrent restart: 2x POST
      return [...prev, 'New item'];
    });
  }
  
  return (
    <div>
      <button onClick={handleAdd}>Add</button>
      <ul>{items.map((i, idx) => <li key={idx}>{i}</li>)}</ul>
    </div>
  );
}

// ✅ Network call alohida:
function ItemsList() {
  const [items, setItems] = useState<string[]>([]);
  
  async function handleAdd() {
    await fetch('/api/items', {
      method: 'POST',
      body: JSON.stringify({ count: items.length + 1 }),
    });
    setItems((prev) => [...prev, 'New item']);
  }
  
  return (
    <div>
      <button onClick={handleAdd}>Add</button>
      <ul>{items.map((i, idx) => <li key={idx}>{i}</li>)}</ul>
    </div>
  );
}
```

Reducer ichida side effect — useReducer parallel:

```tsx
import { useReducer } from 'react';

interface CounterState {
  count: number;
}

type CounterAction = { type: 'INCREMENT' };

// ❌ Side effect reducer ichida
function reducer(state: CounterState, action: CounterAction): CounterState {
  switch (action.type) {
    case 'INCREMENT': {
      console.log('Incrementing'); // ❌ Strict Mode 2x reducer
      analytics.track('count_increment'); // ❌ 2x track
      return { count: state.count + 1 };
    }
    default:
      return state;
  }
}

// ✅ Side effect dispatch'dan oldin/keyin
function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  
  function handleIncrement() {
    analytics.track('count_increment'); // ✅ Bir marta event handler'da
    dispatch({ type: 'INCREMENT' });
  }
  
  return <button onClick={handleIncrement}>{state.count}</button>;
}

// ✅ Yoki useEffect bilan after-effect
import { useEffect } from 'react';

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  
  useEffect(() => {
    if (state.count > 0) {
      analytics.track('count_changed', { count: state.count });
    }
  }, [state.count]);
  
  return <button onClick={() => dispatch({ type: 'INCREMENT' })}>{state.count}</button>;
}
```

</details>

---

## Anti-Pattern 3: Stale Closure Concurrent State

### Nazariya

**Stale closure** — closure'da yopilgan state reference'i komponent'ning **eski snapshot**'idan, joriy snapshot'dan emas. Concurrent rendering bu muammoni **kuchaytiradi** — render restart, transition, async setState — barchasi closure'ning eski qiymat'ini ushlash imkoniyatini oshiradi.

Klassik stale closure manbalari:

1. **Empty deps `useEffect`** — interval/timer/listener handler eski state ushlaydi.
2. **Async setState** — Promise/setTimeout callback eski state.
3. **Event listener attached once** — `useEffect(() => { window.addEventListener(...); }, [])` handler eski state.
4. **Reducer-style update'siz state read** — handler ichida `count` o'qish ([cross-ref] direct value setState).

NIMA UCHUN Concurrent'da kuchayadi:

- **Transition pending** — useTransition ichidagi ish commit qilinmagan vaqtda ekranda hali ham eski state ko'rinadi (Transition ishlanmaguncha). Async context'da eski state'ni o'qish bug keltirib chiqaradi.
- **Suspense retry** — komponent re-mount bo'lsa closure qayta yaratiladi, lekin saqlangan timer/listener eski closure'ga ega.
- **Interrupted render** — render uzilib boshqa state bilan resume bo'lsa, ba'zi closure'lar eski snapshot'dan.

Stale closure'ni hal qilishning **3 standart pattern**:

### Pattern 1: Functional update
```tsx
setCount((c) => c + 1); // Latest c, closure-independent
```

### Pattern 2: useRef latest pattern
```tsx
const latestCountRef = useRef(count);
useEffect(() => { latestCountRef.current = count; });

useEffect(() => {
  const id = setInterval(() => {
    console.log(latestCountRef.current); // Always latest
  }, 1000);
  return () => clearInterval(id);
}, []);
```

### Pattern 3: Effect deps to'g'ri
```tsx
useEffect(() => {
  const id = setInterval(() => {
    console.log(count); // Closure'da count, deps'ga kiritilgan
  }, 1000);
  return () => clearInterval(id);
}, [count]); // Deps to'g'ri → har count change'da interval qayta yaratiladi
```

> **Versiya evolyutsiyasi (Effect Event):**
> - **Pre-R19 (R18 va undan oldin):** Stale closure muammoga uchta pattern (functional update / useRef latest / deps to'g'ri).
> - **Experimental (RFC stage):** `experimental_useEffectEvent` hook canary build'lar uchun — non-reactive event handler'lar uchun. Render'da yopilgan state'ni har chaqiriqda **latest** qiymat bilan o'qish, deps'ga kiritmasdan.
> - **Status:** R19.x stable'da hali yo'q (faqat experimental/canary build'larda mavjud). Production'da ishlatish tavsiya etilmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

Closure capture mexanikasi:

```javascript
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    // Bu closure mount paytida yaratiladi
    // count = 0 yopiladi
    const id = setInterval(() => {
      console.log(count); // count = 0 har doim
      setCount(count + 1); // setCount(0 + 1) har doim
    }, 1000);
    return () => clearInterval(id);
  }, []); // Deps yo'q → effect qayta yaratilmaydi
}
```

Har render yangi closure yaratiladi, lekin effect faqat birinchi mount'da bajarilgan, **birinchi closure'ni** ushlab qoladi.

`useRef latest pattern` ichki ishlash:

```javascript
function Counter() {
  const [count, setCount] = useState(0);
  const ref = useRef(count); // ref.current = 0
  
  // Har render'da ref ni yangilash
  useEffect(() => {
    ref.current = count; // Sync with latest
  });
  
  useEffect(() => {
    // Closure ref reference'ni yopadi (mutable container)
    // ref.current har chaqiriqda latest qiymat
    const id = setInterval(() => {
      console.log(ref.current); // Latest count
    }, 1000);
    return () => clearInterval(id);
  }, []);
}
```

`useRef` mutable container (cross-ref `18-useref.md`) — `ref.current` write/read render'siz, closure'da reference yopiladi (har read latest qiymat).

`useEffectEvent` React'da built-in hook (canary/experimental) — quyida uning g'oyasini ko'rsatuvchi userland approximation (React'ning haqiqiy implementation'i emas; React versiyasi maxsus linter/compiler qo'llab-quvvatlashiga ega):

```javascript
// Userland approximation (React'ning ichki implementation'i emas)
function useEffectEventApprox(fn) {
  const ref = useRef(fn);
  
  useEffect(() => {
    ref.current = fn; // Latest reference
  });
  
  return useCallback((...args) => ref.current(...args), []);
}

// React'ning built-in useEffectEvent ishlatilishi (canary/experimental):
function Counter() {
  const [count, setCount] = useState(0);
  
  const onTick = useEffectEvent(() => {
    console.log(count); // Latest count, no deps required
  });
  
  useEffect(() => {
    const id = setInterval(onTick, 1000);
    return () => clearInterval(id);
  }, []); // No deps — onTick stable
}
```

R19 hali stable emas — production'da `useRef latest pattern` ishlatilsin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Interval bug — 3 ta yechim:

```tsx
import { useState, useEffect, useRef } from 'react';

// ❌ Stale closure
function TimerBug() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      console.log(count); // ❌ Har doim 0
      setCount(count + 1); // ❌ Har doim setCount(1)
    }, 1000);
    return () => clearInterval(id);
  }, []); // count yo'q deps'da
  
  return <div>{count}</div>;
}
// Output: 0, 0, 0, 0, ... (count har sekund 1 ga set qilinadi)
// UI: 0 → 1, keyin o'zgarmaydi.

// ✅ Yechim 1: Functional update
function Timer1() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      setCount((c) => c + 1); // ✅ Latest c
    }, 1000);
    return () => clearInterval(id);
  }, []);
  
  return <div>{count}</div>;
}
// Output: count 1, 2, 3, 4, ...

// ✅ Yechim 2: useRef latest pattern
function Timer2() {
  const [count, setCount] = useState(0);
  const latestCountRef = useRef(count);
  
  useEffect(() => {
    latestCountRef.current = count; // Sync har render
  });
  
  useEffect(() => {
    const id = setInterval(() => {
      console.log(latestCountRef.current); // ✅ Latest
      setCount((c) => c + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);
  
  return <div>{count}</div>;
}

// ✅ Yechim 3: Deps to'g'ri
function Timer3() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      console.log(count); // ✅ Latest count (closure rebuilt)
      setCount((c) => c + 1);
    }, 1000);
    return () => clearInterval(id);
  }, [count]); // count deps'da
  
  return <div>{count}</div>;
}
// Eslatma: bu pattern har count change'da interval qayta yaratiladi.
// Performance overhead — har sekund clearInterval+setInterval.
// Yechim 1 yoki 2 afzal.
```

Async setState — Promise:

```tsx
import { useState } from 'react';

interface Order {
  id: string;
  status: 'pending' | 'confirmed';
}

// ❌ Stale closure async
function OrderConfirm() {
  const [orders, setOrders] = useState<Order[]>([]);
  
  async function confirmOrder(id: string) {
    await fetch(`/api/orders/${id}/confirm`, { method: 'POST' });
    
    // ❌ orders snapshot fetch boshlanish vaqtidagi
    const updated = orders.map((o) =>
      o.id === id ? { ...o, status: 'confirmed' as const } : o
    );
    setOrders(updated);
    // Boshqa order qo'shilgan bo'lsa await davomida — yo'qoladi
  }
  
  return <button onClick={() => confirmOrder('order-1')}>Confirm</button>;
}

// ✅ Functional update
function OrderConfirm() {
  const [orders, setOrders] = useState<Order[]>([]);
  
  async function confirmOrder(id: string) {
    await fetch(`/api/orders/${id}/confirm`, { method: 'POST' });
    
    setOrders((prev) =>
      prev.map((o) =>
        o.id === id ? { ...o, status: 'confirmed' as const } : o
      )
    ); // ✅ Latest prev
  }
  
  return <button onClick={() => confirmOrder('order-1')}>Confirm</button>;
}
```

Event listener — useRef pattern:

```tsx
import { useState, useEffect, useRef } from 'react';

function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<string[]>([]);
  const messagesRef = useRef(messages);
  
  // Har render'da ref'ni yangilash
  useEffect(() => {
    messagesRef.current = messages;
  });
  
  useEffect(() => {
    const ws = new WebSocket(`wss://chat/${roomId}`);
    
    ws.onmessage = (event) => {
      const newMsg = event.data;
      // Latest messages'ni ushlab qoladi
      console.log('Total messages so far:', messagesRef.current.length);
      setMessages((prev) => [...prev, newMsg]);
    };
    
    return () => ws.close();
  }, [roomId]); // Faqat roomId deps
  
  return (
    <ul>
      {messages.map((m, idx) => (
        <li key={idx}>{m}</li>
      ))}
    </ul>
  );
}
```

</details>

---

## Anti-Pattern 4: External Mutable Reads During Render

### Nazariya

**External mutable reads** — render Phase ichida mutable external state'ni o'qish: `Date.now()`, `Math.random()`, `window.innerWidth`, `localStorage.getItem(...)`, `document.cookie`, `performance.now()`. Bu Render Purity invariant'ining buzilishi va Concurrent rendering paytida tearing/inconsistency keltirib chiqaradi.

NIMA UCHUN bu anti-pattern:

1. **Determinizm yo'q** — har chaqiriqda boshqa qiymat qaytadi.
2. **Strict Mode 2x render** — ikki marta chaqirilganda ikki farqli qiymat → idempotency buziladi.
3. **Hydration mismatch** — server'da `window` yo'q, client'da bor → mismatch warning.
4. **Tearing** — render davomida external state o'zgarishi mumkin (Concurrent yield orasida).
5. **Snapshot inconsistency** — har komponent boshqa qiymat ko'radi.

Standart yechim'lar:

| External source | Yechim |
|-----------------|--------|
| `Date.now()` / `performance.now()` | `useState` + `useEffect` (timer) |
| `Math.random()` / `crypto.randomUUID()` | `useState(() => randomFn())` lazy init |
| `window.innerWidth` | `useSyncExternalStore` (resize listener) |
| `localStorage.getItem` | `useSyncExternalStore` (storage listener) |
| `document.cookie` | `useSyncExternalStore` |
| `navigator.onLine` | `useSyncExternalStore` (online/offline events) |
| `matchMedia` | `useSyncExternalStore` (change event) |

Bir tomonlama — **`useId`** SSR-safe ID generation uchun (cross-ref `22-concurrent-hooks.md`). `Math.random()` o'rniga ARIA, form ID'lar uchun.

NIMA UCHUN `useState` factory faqat mount'da bir marta chaqiriladi:

```tsx
const [id] = useState(() => crypto.randomUUID());
// ✅ Mount: factory bir marta chaqiriladi (Strict Mode'da 2 marta lekin natija saqlanmaydi)
// Update: factory chaqirilmaydi, saqlangan qiymat qaytariladi
// SSR: server'da generate, client'da hydrate'da SSR qiymati ishlatiladi
```

Lekin SSR uchun `crypto.randomUUID()` mismatch xavfi — server va client har xil ID. `useId` SSR-safe (Fiber tree path'ga asoslangan deterministic).

<details>
<summary><strong>Under the Hood</strong></summary>

`useState` lazy initial implementation:

```javascript
function mountState(initialState) {
  const hook = mountWorkInProgressHook();
  
  if (typeof initialState === 'function') {
    // Lazy initial: function chaqiriladi
    initialState = initialState();
  }
  // Else: direct value
  
  hook.memoizedState = hook.baseState = initialState;
  // ...
  return [initialState, dispatch];
}

function updateState() {
  const hook = updateWorkInProgressHook();
  // initialState e'tiborsiz (mount'da olingan)
  // memoizedState saqlangan qiymat
  return [hook.memoizedState, hook.queue.dispatch];
}
```

Strict Mode'da lazy initializer 2x chaqiriladi (purity check) — agar har ikki chaqiriq turli qiymat bersa (`Math.random()`), bu impurity'ni fosh qiladi. Komponent state esa baribir bitta deterministic qiymat bilan boshlanadi.

`useSyncExternalStore` SSR snapshot:

```javascript
function mountSyncExternalStore(subscribe, getSnapshot, getServerSnapshot) {
  // SSR: getServerSnapshot ishlatiladi (window yo'q)
  if (isHydrating) {
    if (!getServerSnapshot) {
      throw new Error('Missing getServerSnapshot');
    }
    return getServerSnapshot();
  }
  // Client: getSnapshot
  return getSnapshot();
}
```

Hydration mismatch detection:

```
SSR render: getServerSnapshot() = false
       │
       ▼
HTML: <div class="mobile">...
       │
       ▼
Client mount (hydration): 
  First read: getServerSnapshot() = false (consistent)
  Effect: subscribe + check getSnapshot() = true (window.matchMedia)
       │
       ▼
React: snapshot mismatch → re-render with true → "desktop"
       │
       ▼
DOM: class="mobile" → class="desktop"
```

`onRecoverableError` callback (R18+) — silent recovery'larni log qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

ID generation:

```tsx
import { useState, useId } from 'react';

// ❌ Render'da random ID
function FormField() {
  const id = `field-${Math.random()}`; // ❌ Har render boshqa
  return (
    <>
      <label htmlFor={id}>Name</label>
      <input id={id} />
    </>
  );
}
// Strict Mode 2x render: 2 farqli ID, label/input mismatch.

// ✅ useId — SSR-safe deterministic
function FormField() {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>Name</label>
      <input id={id} />
    </>
  );
}

// ✅ Yoki useState factory (mount'da bir marta)
function FormField() {
  const [id] = useState(() => `field-${crypto.randomUUID()}`);
  return (
    <>
      <label htmlFor={id}>Name</label>
      <input id={id} />
    </>
  );
}
// Eslatma: SSR'da mismatch xavfi (server crypto vs client crypto har xil natija).
// useId afzal SSR uchun.
```

Window dimension:

```tsx
import { useSyncExternalStore } from 'react';

// ❌ Render'da window
function ResponsiveLayout() {
  const isMobile = window.innerWidth < 768; // ❌ SSR'da error
  return isMobile ? <MobileNav /> : <DesktopNav />;
}

// ✅ useSyncExternalStore
function useWindowWidth(): number {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('resize', callback);
      return () => window.removeEventListener('resize', callback);
    },
    () => window.innerWidth,
    () => 0 // SSR snapshot
  );
}

function ResponsiveLayout() {
  const width = useWindowWidth();
  const isMobile = width < 768;
  return isMobile ? <MobileNav /> : <DesktopNav />;
}
```

Online status:

```tsx
import { useSyncExternalStore } from 'react';

// ❌ Render'da navigator.onLine
function ConnectionBadge() {
  const isOnline = navigator.onLine; // ❌ SSR'da error
  return <span>{isOnline ? 'Online' : 'Offline'}</span>;
}

// ✅ useSyncExternalStore
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
    () => true
  );
}

function ConnectionBadge() {
  const isOnline = useOnlineStatus();
  return <span>{isOnline ? 'Online' : 'Offline'}</span>;
}
```

Date.now — timer:

```tsx
import { useState, useEffect } from 'react';

// ❌ Render'da Date.now
function Clock() {
  const now = Date.now(); // ❌ Har render boshqa
  return <span>{new Date(now).toLocaleTimeString()}</span>;
}

// ✅ State + interval
function Clock() {
  const [now, setNow] = useState<number | null>(null);
  
  useEffect(() => {
    setNow(Date.now()); // Initial after mount
    const id = setInterval(() => setNow(Date.now()), 1000);
    return () => clearInterval(id);
  }, []);
  
  if (now === null) return <span>--:--:--</span>;
  return <span>{new Date(now).toLocaleTimeString()}</span>;
}
```

</details>

---

## `startTransition` Invariants Context'ida

### Nazariya

`startTransition(fn)` — Concurrent rendering'ning **explicit interruption boundary**'si. Funksiya ichidagi `setState` chaqiriqlari **TransitionLane** (low-priority) lane'ga tushadi va render uziladi/restart bo'lishi mumkin. Bu API `22-concurrent-hooks.md`'da batafsil — bu yerda **invariants context**'ida ko'rib chiqamiz.

`startTransition`ning invariants'ga ta'siri:

1. **Render Purity invariant kuchayadi** — Transition render uzilib qayta boshlanishi mumkin. Side effect render'da → 2x sodir bo'ladi.
2. **State updates functional merge** — Transition uzilib boshqa render bilan birikmadi bo'lsa, direct value setState eski snapshot'dan o'qiydi. Functional update — joriy state'dan.
3. **Effect symmetry** — Transition pending davomida boshqa render commit bo'lishi mumkin. Effect cleanup → setup pair to'g'ri ishlashi shart.
4. **External subscription consistency** — Transition davomida store o'zgarsa, `useSyncExternalStore` tearing'ni aniqlaydi va sinxron (blocking) re-render trigger qiladi.

`startTransition` ichida **siyosat**:

| Operatsiya | Tavsiya | Sabab |
|-----------|---------|-------|
| `setState(value)` direct | OK (mustaqil) | Snapshot mavjud |
| `setState(prev => ...)` | Afzal (bog'liq) | Restart safe |
| Side effect (`fetch`, `console.log`) | Taqiq | Restart 2x |
| `localStorage.setItem` | Taqiq | Side effect |
| Transition render'da komponent suspend bo'lishi | OK | Suspense fallback Transition'ni bloklamaydi — eski UI saqlanadi |
| Transition render'da error throw bo'lishi | OK | Error Boundary ushlaydi |

`startTransition` bilan birga `useTransition`'dan qaytgan `isPending` boolean — UI feedback uchun (yuk indikatori, dim opacity).

```tsx
import { useState, useTransition } from 'react';

function SearchBox() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();
  
  function handleChange(value: string) {
    setQuery(value); // Sync (urgent)
    
    startTransition(() => {
      // Transition (non-urgent)
      const filtered = expensiveFilter(value);
      setResults(filtered);
    });
  }
  
  return (
    <>
      <input value={query} onChange={(e) => handleChange(e.target.value)} />
      <div style={{ opacity: isPending ? 0.5 : 1 }}>
        <ResultList items={results} />
      </div>
    </>
  );
}
```

Bu yerda `setQuery` darhol UI'ni yangilaydi (input responsive), `setResults` Transition'da — uziladi va tez yozish paytida wasted work tashlab yuboriladi.

NIMA UCHUN Transition uzilishi safe:

- `setResults` chaqirilgan, queue'ga tushgan, lekin commit bo'lmagan.
- Yangi user input keladi → sync `setQuery` chaqiriladi.
- React Transition render'ni bekor qiladi va yangi render boshlaydi (yangi query bilan).
- Eski Transition's setResults call e'tiborsiz qoladi (ma'no jihatdan qayta hisoblanadi).

Render purity invariant kuchayadi:

```tsx
// ❌ Anti-pattern: render Phase'da side effect
function ExpensiveList({ filter }: { filter: string }) {
  console.log('Filtering...'); // ❌ Strict Mode 2x + Transition restart 2x
  const items = expensiveCompute(filter); // ❌ Side effect (DB query, logging)
  return <ul>{items.map((i) => <li key={i.id}>{i.name}</li>)}</ul>;
}

// ✅ Pure render + useMemo + Transition'da setState
function ExpensiveList({ filter }: { filter: string }) {
  const items = useMemo(() => expensiveCompute(filter), [filter]); // ✅ Pure cache
  return <ul>{items.map((i) => <li key={i.id}>{i.name}</li>)}</ul>;
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

`startTransition` implementation:

R19'da transition holati `ReactSharedInternals.T` orqali boshqariladi (R18'dagi `ReactCurrentBatchConfig.transition` o'rniga):

```javascript
function startTransition(scope) {
  const prevTransition = ReactSharedInternals.T; // R19: .T (R18'da ReactCurrentBatchConfig.transition)
  ReactSharedInternals.T = { /* transition object */ };
  
  try {
    scope(); // Funksiya ichidagi setState'lar TransitionLane'ga
  } finally {
    ReactSharedInternals.T = prevTransition;
  }
}

function dispatchSetState(fiber, queue, action) {
  let lane;
  if (ReactSharedInternals.T !== null) {
    lane = claimNextTransitionLane(); // TransitionLane1...Lane16
  } else {
    lane = requestUpdateLane(); // SyncLane / DefaultLane
  }
  
  // Update queue'ga yuklash
  // ...
}
```

Transition lane priority:

```
SyncLane > InputContinuousLane > DefaultLane > TransitionLane1..16 > IdleLane
```

Transition Lane'da rendering:

1. `performConcurrentWorkOnRoot` — TransitionLane'ni oladi.
2. `renderRootConcurrent` — work loop.
3. Har 5ms `shouldYield` → yield.
4. Yangi yuqori-priority work keladi → `prepareFreshStack` (restart).
5. Sync render bajariladi.
6. Transition keyinroq qaytadan boshlanadi.

Wasted work cost:

```
Transition started: setResults(filtered_v1)
       │
       ▼
Render to Fiber #100 (out of 1000)
       │
       ▼
[INTERRUPT] — User typed: setQuery('y')
       │
       ▼
Sync render: input updated
       │
       ▼
Restart Transition with filtered_v2
       │
       ▼
Render to Fiber #150
       │
       ▼
[INTERRUPT] — User typed: setQuery('ye')
       │
       ▼
Sync render
       │
       ▼
Restart Transition with filtered_v3
       │
       ▼
Render to completion
       │
       ▼
Commit
```

Sarflangan ish: 100+150 = 250 Fiber render — wasted. Lekin UI responsive (input lag yo'q).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Search box bilan filter:

```tsx
import { useState, useTransition, useMemo } from 'react';

interface Product {
  id: string;
  name: string;
  category: string;
}

function ProductSearch({ products }: { products: Product[] }) {
  const [query, setQuery] = useState('');
  const [filteredProducts, setFilteredProducts] = useState<Product[]>(products);
  const [isPending, startTransition] = useTransition();
  
  function handleSearch(value: string) {
    setQuery(value); // Sync — input responsive
    
    startTransition(() => {
      // Transition — wasted work tolerable
      const filtered = products.filter((p) =>
        p.name.toLowerCase().includes(value.toLowerCase())
      );
      setFilteredProducts(filtered);
    });
  }
  
  return (
    <div>
      <input
        value={query}
        onChange={(e) => handleSearch(e.target.value)}
        placeholder="Search..."
      />
      {isPending && <span>Filtering...</span>}
      <ul style={{ opacity: isPending ? 0.5 : 1 }}>
        {filteredProducts.map((p) => (
          <li key={p.id}>{p.name} — {p.category}</li>
        ))}
      </ul>
    </div>
  );
}
```

Tab switching (Transition + Suspense):

```tsx
import { useState, useTransition, Suspense, lazy } from 'react';

const HomeTab = lazy(() => import('./HomeTab'));
const ProductsTab = lazy(() => import('./ProductsTab'));
const SettingsTab = lazy(() => import('./SettingsTab'));

type TabId = 'home' | 'products' | 'settings';

function App() {
  const [activeTab, setActiveTab] = useState<TabId>('home');
  const [isPending, startTransition] = useTransition();
  
  function handleTabChange(tab: TabId) {
    startTransition(() => {
      setActiveTab(tab); // Transition — code split chunk yuklash uziladi
    });
  }
  
  return (
    <div>
      <nav style={{ opacity: isPending ? 0.5 : 1 }}>
        <button onClick={() => handleTabChange('home')}>Home</button>
        <button onClick={() => handleTabChange('products')}>Products</button>
        <button onClick={() => handleTabChange('settings')}>Settings</button>
      </nav>
      <Suspense fallback={<div>Loading...</div>}>
        {activeTab === 'home' && <HomeTab />}
        {activeTab === 'products' && <ProductsTab />}
        {activeTab === 'settings' && <SettingsTab />}
      </Suspense>
    </div>
  );
}
// Transition + Suspense: Suspense fallback ko'rinmaydi tab switch'da,
// chunki Transition pending davomida eski tab'ni saqlaydi.
// User: bir lahzaga eski tab + dim opacity, keyin yangi tab.
```

Standalone `startTransition` (hook tashqarida):

```tsx
import { startTransition, useState } from 'react';

function ContactForm() {
  const [formData, setFormData] = useState({ name: '', email: '' });
  const [validationErrors, setValidationErrors] = useState<Record<string, string>>({});
  
  function handleChange(field: 'name' | 'email', value: string) {
    setFormData((prev) => ({ ...prev, [field]: value })); // Sync
    
    startTransition(() => {
      // Validation — non-urgent
      const errors = validateForm({ ...formData, [field]: value });
      setValidationErrors(errors);
    });
  }
  
  return (
    <form>
      <input
        value={formData.name}
        onChange={(e) => handleChange('name', e.target.value)}
      />
      {validationErrors.name && <span>{validationErrors.name}</span>}
      
      <input
        type="email"
        value={formData.email}
        onChange={(e) => handleChange('email', e.target.value)}
      />
      {validationErrors.email && <span>{validationErrors.email}</span>}
    </form>
  );
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: Strict Mode `console.log` 2x ko'rinmaydi (R18+)

R18'gacha Strict Mode 2-chi render'da `console.log` ham 2x ko'rinardi — chalkashlik. R18+ — 2-chi render'da console disabled (`disableLogs` internal pattern).

```tsx
function Counter() {
  console.log('Render'); // ❌ R17: 2x ko'rinadi, R18+: 1x
  return <div>...</div>;
}
```

**Yechim:** R18 default behavior to'g'ri — production behavior'ga mos. Test qilish uchun `<StrictMode>`'siz mount qilib log'larni ko'rish.

### Gotcha 2: `useState` lazy initial Strict Mode 2x chaqiriladi

```tsx
function App() {
  const [data] = useState(() => {
    console.log('Init'); // ❌ Strict Mode: 2x log
    return loadData();
  });
}
```

**Yechim:** Initial idempotent bo'lishi shart. Side effect — taqiq. Faqat **derive** allowed (parse JSON, compute initial structure).

### Gotcha 3: `useEffect` cleanup setup'siz mount paytida

Birinchi mount'da cleanup chaqirilmaydi — chunki avval setup yo'q. Strict Mode 2x cycle: setup → cleanup → setup. Ko'plab developerlar "mount → setup, unmount → cleanup" deb bilishadi va Strict Mode'ning interim cleanup'ni kutmaydilar.

```tsx
useEffect(() => {
  console.log('Setup'); // 1, 3
  return () => console.log('Cleanup'); // 2
}, []);
// Strict Mode: Setup, Cleanup, Setup
```

**Yechim:** Effect symmetry — har setup'ga mos cleanup. Idempotent operations afzal.

### Gotcha 4: External store `getSnapshot` har chaqiriqda yangi reference qaytaradi

```tsx
function getSnapshot() {
  return { ...store.state }; // ❌ Yangi object har chaqiriq
}
// Object.is(snap1, snap2) === false → infinite loop xavfi
```

**Yechim:** Stable reference — store ichki state bir xil reference. Yoki primitive return. Yoki kichik selector bilan primitive extract.

### Gotcha 5: `startTransition` async function — R18 vs R19 farqi

```tsx
// R18: async ichidagi await keyin setState — Transition tashqarida bo'lib qolardi
startTransition(async () => {
  await fetch('/api');     // R18: await tugagandan keyin Transition scope tugagan
  setState(data);          // R18: bu Sync (Transition emas)
});
```

**R18 chegarasi:** `startTransition` callback Promise qaytarsa, React Promise resolve'ini kutmasdi. `await` keyingi kod Transition boundary tashqarisida bajarilardi.

**R19 yangiliklar (Async Transitions):** `startTransition` async function'larni qo'llab-quvvatlaydi va `await` paytidagi setState'lar ham Transition Lane'da qoladi. `useTransition` `isPending` Promise resolve'gacha `true` bo'ladi. Bu — `useActionState`/Form Actions asosi.

```tsx
// R19 — Async Transition
startTransition(async () => {
  await fetch('/api');     // ✅ R19: Transition saqlanadi
  setState(data);          // ✅ Transition'da
});
```

**R18-compatible pattern:**

```tsx
async function load() {
  const data = await fetch('/api');
  startTransition(() => {
    setState(data); // ✅ Sync setState Transition'da
  });
}
```

R19'da `useActionState`/`<form action>`'lar async actions'ni avtomatik Transition wrap qiladi (cross-ref `23-r19-hooks.md`).

---

## Common Mistakes

### ❌ Xato 1: Render Phase'da side effect

```tsx
// ❌ Anti-pattern
function App() {
  fetch('/api/log'); // ❌ Network call render'da
  document.title = 'Page'; // ❌ DOM mutation
  return <div>...</div>;
}

// ✅ useEffect'da
function App() {
  useEffect(() => {
    fetch('/api/log');
    document.title = 'Page';
  }, []);
  return <div>...</div>;
}
```

### ❌ Xato 2: Direct value setState bog'liq state'da

```tsx
// ❌ Stale snapshot
function handleClick() {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
}
// count: 0 → 1 (3 emas)

// ✅ Functional update
function handleClick() {
  setCount((c) => c + 1);
  setCount((c) => c + 1);
  setCount((c) => c + 1);
}
// count: 0 → 3
```

### ❌ Xato 3: useEffect cleanup yo'q

```tsx
// ❌ Memory leak
useEffect(() => {
  window.addEventListener('scroll', handleScroll);
}, []);

// ✅ Cleanup symmetry
useEffect(() => {
  window.addEventListener('scroll', handleScroll);
  return () => window.removeEventListener('scroll', handleScroll);
}, []);
```

### ❌ Xato 4: External mutable read render'da

```tsx
// ❌ Hydration mismatch
function App() {
  const isMobile = window.innerWidth < 768; // ❌ SSR'da window yo'q
  return isMobile ? <Mobile /> : <Desktop />;
}

// ✅ useSyncExternalStore
function App() {
  const isMobile = useMediaQuery('(max-width: 767px)');
  return isMobile ? <Mobile /> : <Desktop />;
}
```

### ❌ Xato 5: Strict Mode'ni o'chirish bug bilan kurashish o'rniga

```tsx
// ❌ Bug yashirish
// <App />  // StrictMode'siz
// Bug development'da ko'rinmaydi, production'da ko'rinadi.

// ✅ Effect'ni to'g'rilash
function ChatRoom({ roomId }: { roomId: string }) {
  useEffect(() => {
    const conn = createConnection(roomId);
    conn.connect();
    return () => conn.disconnect(); // ✅ Cleanup
  }, [roomId]);
}
```

---

## Amaliy Mashqlar

### Mashq 1: Pure Component Refactor (Oson)

Quyidagi komponent'da render Phase'da nechta side effect bor? Topib, refactor qiling.

```tsx
let visitCount = 0;

function ProductPage({ productId }: { productId: string }) {
  visitCount += 1;
  document.title = `Product ${productId}`;
  console.log('Page rendered:', visitCount);
  fetch(`/api/products/${productId}/track`, { method: 'POST' });
  
  return <h1>Product: {productId}</h1>;
}
```

<details>
<summary><strong>Javob</strong></summary>

4 ta side effect: `visitCount++`, `document.title = ...`, `console.log`, `fetch`.

```tsx
import { useEffect, useRef } from 'react';

function ProductPage({ productId }: { productId: string }) {
  const visitCountRef = useRef(0);
  
  useEffect(() => {
    visitCountRef.current += 1;
    console.log('Page rendered:', visitCountRef.current);
  });
  
  useEffect(() => {
    document.title = `Product ${productId}`;
    return () => {
      document.title = 'Default';
    };
  }, [productId]);
  
  useEffect(() => {
    fetch(`/api/products/${productId}/track`, { method: 'POST' });
    // Eslatma: Strict Mode 2x cycle 2x POST yuboradi.
    // Yaxshiroq — event handler'da (router navigation event).
  }, [productId]);
  
  return <h1>Product: {productId}</h1>;
}
```

R19'da `<title>` JSX'da:

```tsx
function ProductPage({ productId }: { productId: string }) {
  // ... track effect
  return (
    <>
      <title>Product {productId}</title>
      <h1>Product: {productId}</h1>
    </>
  );
}
```

</details>

### Mashq 2: Stale Closure Fix (Oson)

`useChatTimer` custom hook har 5 sekundda chat'dagi xabarlar sonini console'ga log qilishi kerak. Hozir har log'da `0` ko'rinadi. Tuzating.

```tsx
import { useState, useEffect } from 'react';

function useChatTimer() {
  const [messages, setMessages] = useState<string[]>([]);
  
  useEffect(() => {
    const id = setInterval(() => {
      console.log('Total messages:', messages.length); // ❌ Always 0
    }, 5000);
    return () => clearInterval(id);
  }, []);
  
  return [messages, setMessages] as const;
}
```

<details>
<summary><strong>Javob</strong></summary>

`useRef latest pattern`:

```tsx
import { useState, useEffect, useRef } from 'react';

function useChatTimer() {
  const [messages, setMessages] = useState<string[]>([]);
  const messagesRef = useRef(messages);
  
  // Har render'da ref'ni latest bilan sync
  useEffect(() => {
    messagesRef.current = messages;
  });
  
  useEffect(() => {
    const id = setInterval(() => {
      console.log('Total messages:', messagesRef.current.length); // ✅ Latest
    }, 5000);
    return () => clearInterval(id);
  }, []);
  
  return [messages, setMessages] as const;
}
```

Alternativ: deps to'g'ri, lekin har xabarda interval qayta yaratiladi (performance overhead):

```tsx
useEffect(() => {
  const id = setInterval(() => {
    console.log('Total messages:', messages.length);
  }, 5000);
  return () => clearInterval(id);
}, [messages]); // Performance: har xabarda interval reset
```

useRef pattern afzal — interval bir marta yaratiladi, har log'da latest qiymat.

</details>

### Mashq 3: Tearing-Safe Counter (O'rta)

Global `counterStore`'ni Concurrent rendering'ga safe qiling. Ikki komponent (`<Display />` va `<Total />`) bir vaqtda render bo'lganda bir xil qiymat ko'rishi kerak.

```tsx
// Store
let count = 0;
const listeners = new Set<() => void>();

export function increment() {
  count += 1;
  listeners.forEach((l) => l());
}

// ❌ Naive hook
function useCount() {
  const [, forceUpdate] = useState({});
  
  useEffect(() => {
    const listener = () => forceUpdate({});
    listeners.add(listener);
    return () => {
      listeners.delete(listener);
    };
  }, []);
  
  return count;
}
```

<details>
<summary><strong>Javob</strong></summary>

`useSyncExternalStore`:

```tsx
import { useSyncExternalStore } from 'react';

let count = 0;
const listeners = new Set<() => void>();

export function increment() {
  count += 1;
  listeners.forEach((l) => l());
}

function subscribe(callback: () => void): () => void {
  listeners.add(callback);
  return () => {
    listeners.delete(callback);
  };
}

function getSnapshot(): number {
  return count;
}

function getServerSnapshot(): number {
  return 0;
}

export function useCount(): number {
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}

// Usage
function Display() {
  const c = useCount();
  return <span>Display: {c}</span>;
}

function Total() {
  const c = useCount();
  return <span>Total: {c * 2}</span>;
}

function App() {
  return (
    <>
      <Display />
      <Total />
      <button onClick={increment}>+</button>
    </>
  );
}
```

`useSyncExternalStore` Concurrent rendering paytida tearing'ni aniqlaydi va sinxron (blocking) re-render trigger qiladi — ikki komponent bir xil snapshot ko'radi.

</details>

### Mashq 4: Effect Symmetry Audit (O'rta)

Quyidagi `useChatRoom` hook'ida 3 ta cleanup invariant buzilishi bor. Topib, tuzating.

```tsx
import { useEffect, useState } from 'react';

interface Message {
  id: string;
  text: string;
}

function useChatRoom(roomId: string) {
  const [messages, setMessages] = useState<Message[]>([]);
  
  useEffect(() => {
    // 1. WebSocket
    const ws = new WebSocket(`wss://chat/${roomId}`);
    ws.onmessage = (e) => {
      setMessages((prev) => [...prev, JSON.parse(e.data)]);
    };
    
    // 2. Polling fallback
    const pollId = setInterval(() => {
      if (ws.readyState !== WebSocket.OPEN) {
        fetch(`/api/messages/${roomId}`).then((r) => r.json());
      }
    }, 5000);
    
    // 3. Window focus listener
    window.addEventListener('focus', () => ws.send('REFRESH'));
    
    // ❌ No cleanup
  }, [roomId]);
  
  return messages;
}
```

<details>
<summary><strong>Javob</strong></summary>

3 ta cleanup yo'q: WebSocket close, clearInterval, removeEventListener.

```tsx
import { useEffect, useState } from 'react';

interface Message {
  id: string;
  text: string;
}

function useChatRoom(roomId: string) {
  const [messages, setMessages] = useState<Message[]>([]);
  
  useEffect(() => {
    const ws = new WebSocket(`wss://chat/${roomId}`);
    ws.onmessage = (e) => {
      setMessages((prev) => [...prev, JSON.parse(e.data)]);
    };
    
    const pollId = setInterval(() => {
      if (ws.readyState !== WebSocket.OPEN) {
        fetch(`/api/messages/${roomId}`).then((r) => r.json());
      }
    }, 5000);
    
    function handleFocus() {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send('REFRESH');
      }
    }
    window.addEventListener('focus', handleFocus);
    
    return () => {
      ws.close(); // ✅ Cleanup 1
      clearInterval(pollId); // ✅ Cleanup 2
      window.removeEventListener('focus', handleFocus); // ✅ Cleanup 3
    };
  }, [roomId]);
  
  return messages;
}
```

Strict Mode 2x cycle:

1. setup: ws connect, interval start, listener add
2. cleanup: ws close, interval clear, listener remove
3. setup: ws connect (yangi), interval start, listener add

Server: bir aktiv WebSocket connection (eski yopilgan).

</details>

### Mashq 5: Concurrent-Safe `useDebounce` (Qiyin)

`useDebounce` hook yarating: input qiymatini `delay` ms keyin update qiladi. Concurrent rendering safe bo'lishi shart — Strict Mode 2x cycle, race condition, stale closure'larsiz. Functional update va cleanup symmetry ishlating.

```tsx
function useDebounce<T>(value: T, delay: number): T {
  // TODO
}

// Foydalanish
function SearchBox() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);
  
  useEffect(() => {
    if (debouncedQuery) {
      fetch(`/api/search?q=${debouncedQuery}`);
    }
  }, [debouncedQuery]);
  
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState, useEffect } from 'react';

function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);
  
  useEffect(() => {
    const id = setTimeout(() => {
      setDebouncedValue(value); // ✅ Direct value (mustaqil, snapshot OK)
    }, delay);
    
    return () => {
      clearTimeout(id); // ✅ Cleanup symmetry
    };
  }, [value, delay]);
  
  return debouncedValue;
}

// Strict Mode 2x cycle:
// 1. setTimeout(id1, delay)
// 2. clearTimeout(id1) — eski timer bekor
// 3. setTimeout(id2, delay)
// Faqat oxirgi setTimeout natija beradi (debounced).

// Race condition'siz:
// - Tez yozish: har keystroke setTimeout + clearTimeout (eski)
// - Faqat oxirgi keystroke'dan delay ms keyin debouncedValue update.
```

`useDebouncedCallback` variant (function debounce):

```tsx
import { useRef, useCallback, useEffect } from 'react';

function useDebouncedCallback<Args extends unknown[]>(
  callback: (...args: Args) => void,
  delay: number
): (...args: Args) => void {
  const callbackRef = useRef(callback);
  const timeoutRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  
  // Latest callback ref pattern
  useEffect(() => {
    callbackRef.current = callback;
  });
  
  // Cleanup on unmount
  useEffect(() => {
    return () => {
      if (timeoutRef.current !== null) {
        clearTimeout(timeoutRef.current);
      }
    };
  }, []);
  
  return useCallback(
    (...args: Args) => {
      if (timeoutRef.current !== null) {
        clearTimeout(timeoutRef.current);
      }
      timeoutRef.current = setTimeout(() => {
        callbackRef.current(...args);
      }, delay);
    },
    [delay]
  );
}

// Foydalanish
function SearchBox() {
  const handleSearch = useDebouncedCallback((q: string) => {
    fetch(`/api/search?q=${q}`);
  }, 300);
  
  return <input onChange={(e) => handleSearch(e.target.value)} />;
}
```

Concurrent rendering safe:

- Strict Mode 2x cycle: timer cleared, qayta yaratiladi.
- Latest callback: `useRef` orqali, deps siz.
- Cleanup unmount'da: timer leak yo'q.

</details>

---

## Xulosa

Concurrent rendering — React'ning **fundamental rendering modeli**, R18+'da default. Bu fayl **mental model va invariants**'ga fokus qildi:

- **Sync vs Concurrent** — render uninterruptible vs interruptible. Komponent funksiyasi bir logical render uchun bir nechta marta chaqirilishi mumkin (interrupt → restart, Strict Mode 2x cycle).
- **Render Phase Restartable** — Render Purity invariant: bir xil input → bir xil JSX, side effect taqiq, mutation taqiq. Strict Mode bu invariant'ni darhol fosh qiladi.
- **Strict Mode 2x Effect Cycle (R18+)** — har mount'da `setup → cleanup → setup` cycle, sabab Concurrent re-mount mexanikasi (Offscreen API, Fast Refresh, Error recovery).
- **Tearing** — Concurrent yield orasida external store o'zgarsa, UI'ning ikki qismida turli qiymat ko'rinishi. `useSyncExternalStore` rasmiy yechim — tearing aniqlansa sinxron (blocking) re-render trigger qiladi.
- **4 ta Concurrent Invariants** — render purity, state functional merge, effect cleanup/setup symmetry, external subscription consistency.
- **4 ta Anti-Pattern** — render-time mutation, setState callback side effects, stale closure, external mutable reads.
- **`startTransition`** — explicit Concurrent boundary, invariants context'ida — render purity kuchayadi (restart 2x), functional update afzal, cleanup symmetry shart.

QISM 7 (Advanced Patterns) **TUGADI** ✅. Keyingi qism — QISM 8 (Performance & Compiler) — React Compiler bilan boshlanadi: build-time auto-memoization, manual `useMemo`/`useCallback`/`React.memo` kerak emas bo'lib qolishi.

---

**Keyingi bo'lim:** [31-react-compiler.md](31-react-compiler.md) — React Compiler (alohida opt-in build-time vosita, React 17/18/19 bilan ishlaydi; avval React Forget): source kod statik analiz, auto-memoization, internal cache mechanism, Rules of React compliance, `eslint-plugin-react-compiler`, `babel-plugin-react-compiler` opt-in, migration path 6 qadam, future paradigm — manual memoization yo'q bo'lib ketishi.
