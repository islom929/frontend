# Bo'lim 32: Rendering Behavior — Practical Re-render Triggers

> Re-render — komponent funksiyasining qayta chaqirilishi va Reconciler tomonidan virtual tree'ning qayta hisoblanishi. Bu fayl **practical perspective**'dan re-render trigger'larini, propagation'ni va debugging'ni o'rganadi: state change, parent re-render top-down, Context value change, force update, `React.memo` bypass scenarios, reference equality gotchas, va DevTools "Why did this render?" feature. Reconciler bailout algorithm internal mexanikasi `04-reconciliation.md`'da chuqur (bu yerda **practical implications**), `useMemo`/`useCallback` mexanika `21-usememo-usecallback.md`'da (bu yerda **trigger scenarios**), Compiler auto-memoization `31-react-compiler.md`'da (bu yerda **manual qoladigan qismlar**). Bu fayl debugging skill — production app'da "nima uchun re-render bo'ldi?" savoliga aniq javob bera olish.

---

## Mundarija

- [Re-render Asosiy Tushunchasi](#re-render-asosiy-tushunchasi)
- [Trigger 1: State Change (`setState`)](#trigger-1-state-change-setstate)
- [Trigger 2: Parent Re-render — Top-Down Propagation](#trigger-2-parent-re-render--top-down-propagation)
- [Trigger 3: Context Value Change](#trigger-3-context-value-change)
- [Trigger 4: Force Update Pattern](#trigger-4-force-update-pattern)
- [Reconciler Bailout — Re-render Skip Manbai](#reconciler-bailout--re-render-skip-manbai)
- [`React.memo` va Shallow Comparison](#reactmemo-va-shallow-comparison)
- [`React.memo` Bypass Scenarios](#reactmemo-bypass-scenarios)
- [Reference Equality Gotchas](#reference-equality-gotchas)
- ["New Element Same Type" — Update Path](#new-element-same-type--update-path)
- [Stale Closure Scenarios](#stale-closure-scenarios)
- [DevTools "Why did this render?"](#devtools-why-did-this-render)
- [Highlight Updates va Profile Workflow](#highlight-updates-va-profile-workflow)
- [Compiler Era Re-render Behavior](#compiler-era-re-render-behavior)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Re-render Asosiy Tushunchasi

### Nazariya

**Re-render** — komponent funksiyasining qayta chaqirilishi natijasida yangi JSX tree hisoblanishi va Reconciler shu yangi tree'ni eski bilan solishtirib, kerakli DOM o'zgarishlarini Commit Phase'da qo'llashi.

Re-render **DOM update bilan teng emas**:

- **Render Phase** — komponent funksiyasi chaqiriladi, JSX qaytariladi, Reconciler diff hisoblaydi.
- **Commit Phase** — Reconciler aniqlagan DOM mutation'lar qo'llaniladi.

Re-render bo'lgani — **DOM o'zgardi** degani emas. Reconciler bailout (cross-ref `04-reconciliation.md`) re-render bo'lgan komponent uchun DOM hech qanday mutation qilmasligi mumkin.

Re-render trigger'larining 4 ta asosiy turi:

1. **State change** (`setState`/`dispatch`) — komponent o'z state'ini o'zgartiradi.
2. **Parent re-render** — parent komponent re-render bo'lsa, **default'da barcha children re-render** bo'ladi. Sabab: parent funksiyasi qayta chaqirilganda JSX qayta hisoblanadi va har children uchun `React.createElement` (yoki R17+ `jsx()`) **yangi React Element obyekt** qaytaradi. Reconciler `prevProps !== nextProps` (reference inequality) ko'radi va children'ni re-render qiladi. **Reconciler shallow comparison qilmaydi** — bu faqat `React.memo` HOC tomonidan amalga oshiriladi.
3. **Context value change** — `useContext`/`use(context)` ishlatilgan komponentlar Context value o'zgarganda re-render bo'ladi (har consumer parent re-render bo'lmasa ham mustaqil ravishda update lane oladi — `propagateContextChange` subtree traversal).
4. **Force update** — `useReducer((x) => x + 1, 0)`/`useState({})` pattern bilan komponent'ni qo'l bilan re-render qilish (kamdan-kam — modern alternativ `useSyncExternalStore`).

NIMA UCHUN bu farq muhim:

- **Performance debugging** — "Re-render bo'ldi" demak, "DOM o'zgardi" emas. Profile'ga qaraganda re-render count shubhali bo'lishi mumkin lekin Commit duration nol bo'lishi mumkin.
- **Optimization strategy** — re-render'ni oldini olish (React.memo) yoki Reconciler bailout'ga ishonish (default), yoki DOM update'ni minimallashtirish — har biri alohida pattern.
- **Mental model accuracy** — junior'lar "har setState DOM update" deb taxmin qiladilar. Reality: setState → render → diff → bailout (deyarli har doim) yoki minimal mutation.

> **Versiya evolyutsiyasi (Re-render Behavior):**
> - **Pre-R16:** Stack Reconciler — sync, uninterruptible. Re-render boshlangach yakuniga qadar to'xtatib bo'lmaydi.
> - **R16+:** Fiber Reconciler — interruptible (R18'da Concurrent default). Re-render uzilib qayta boshlanishi mumkin.
> - **R18+:** Automatic Batching — har context'da multiple `setState` bir render'ga birlashadi (sync vs async).
> - **R19 (Compiler stable):** React Compiler 1.0 ekosistemada keng tarqaldi — komponent darajasidagi memoization avtomatik (cross-ref `31-react-compiler.md`). Compiler R17/R18/R19 bilan ishlaydi (`target` opsiyasi), R19 ga bog'lanmagan — R19 release atrofida bozor adoption oshdi.
> - **Sabab:** Re-render mexanizmi reference comparison'ga asoslangan: Reconciler `prevProps === nextProps` (`!==` reference inequality) tekshiradi (`useState` eager bailout va `React.memo` shallow check esa `Object.is` ishlatadi — primitive ekvivalentlik + NaN/±0 nuance). Bu farq mental model uchun muhim: **Reconciler shallow compare qilmaydi, faqat `React.memo` qiladi**.

<details>
<summary><strong>Under the Hood</strong></summary>

Re-render lifecycle:

```
setState chaqirildi
       │
       ▼
Update queue'ga tushadi (Fiber.queue)
       │
       ▼
ScheduleUpdateOnFiber → Lane priority assigned
       │
       ▼
Scheduler workLoop
       │
       ▼
Reconciler render Phase
       │
       ├─ Component(props) chaqiriladi
       ├─ JSX yaratiladi
       ├─ Diff: prev fiber vs work-in-progress
       ├─ Bailout check (Object.is, React.memo)
       │
       ▼
Commit Phase
       │
       ├─ Mutation sub-phase: DOM updates
       ├─ Layout sub-phase: useLayoutEffect
       │
       ▼
Browser paint
       │
       ▼
Passive effects (useEffect)
```

Re-render statistic — `actualDuration` vs `baseDuration` (Profiler API):

- `actualDuration` — bu commit'da real ish vaqti (bailout'lar tashqarida).
- `baseDuration` — agar barcha komponent'lar re-render bo'lsa, taxminiy vaqt.

Yaxshi optimization — `actualDuration` `baseDuration`'dan ancha kichik (ko'p bailout).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Re-render kuzatuvchi sodda misol:

```tsx
import { useState, useEffect, useRef } from 'react';
import type { ReactElement } from 'react';

function RenderCounter({ name }: { name: string }): ReactElement {
  const renderCountRef = useRef(0);
  
  useEffect(() => {
    renderCountRef.current += 1;
    console.log(`[${name}] Renders:`, renderCountRef.current);
  });
  
  return <span>{name}: {renderCountRef.current}</span>;
}

function App(): ReactElement {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>Increment: {count}</button>
      <RenderCounter name="Always" />
    </div>
  );
}
// Har click: App re-render, RenderCounter ham re-render (parent triggered).
// Console:
//   [Always] Renders: 1
//   [Always] Renders: 2
//   [Always] Renders: 3
```

</details>

---

## Trigger 1: State Change (`setState`)

### Nazariya

**State change** — eng tez-tez sodir bo'ladigan re-render trigger. `useState` setter (`setCount`), `useReducer` dispatch, yoki R19 `useActionState`/`useOptimistic` action'lar bilan boshqariladi.

`setState` lifecycle:

1. Setter chaqiriladi: `setCount(c => c + 1)` yoki `setCount(5)`.
2. Update queue'ga `Update` obyekt push (cross-ref `12-state-and-usestate.md`).
3. Eager bailout check — agar yangi state eski bilan `Object.is` true bo'lsa, render schedule qilinmaydi (cross-ref `04-reconciliation.md`).
4. Aks holda — Reconciler scheduler'iga update lane priority bilan tushadi.
5. Render Phase'da komponent funksiyasi chaqiriladi.
6. Reconciler children'ni bailout qilishi yoki re-render qilishi mumkin (props/context o'zgarmagan bo'lsa).

NIMA QILMAYDI `setState`:

- **DOM'ni darhol o'zgartirmaydi** — Render Phase'dan keyin Commit Phase.
- **Sync chaqiriqda yangi state'ni qaytarmaydi** — `setCount(5); console.log(count)` — `count` hali eski qiymat (snapshot per render).
- **Eski state bilan teng bo'lsa render qilmaydi** — eager bailout (cross-ref `12-state-and-usestate.md`).

`setState` re-render edge case'lari:

```tsx
const [count, setCount] = useState(0);

// 1. Same value — eager bailout (no re-render)
setCount(0); // count hali 0 → no render
setCount((c) => c); // identity function → no render

// 2. New value — render
setCount(1); // 0 !== 1 → render

// 3. Same reference object — eager bailout
const obj = { x: 1 };
setObj(obj); // same reference → no render
setObj(obj); // hali bir xil → no render

// 4. New reference, same shape — render bo'ladi
setObj({ x: 1 }); // new reference → eager bailout false → re-render
// Re-render bo'ladi (komponent funksiyasi chaqiriladi), lekin agar JSX
// output bir xil bo'lsa — DOM mutation yo'q (diff bir xil natija beradi).
// Children memo'd bo'lsa, ularning props o'zgarmagan bo'lsa, ular bailout.
```

Bog'liq `setState` chaqiriqlari (cross-ref `30-concurrent-react.md` Invariant 2):

```tsx
function handleClick() {
  setCount(count + 1); // ❌ Snapshot-based (closure)
  setCount(count + 1); // ❌ Snapshot-based — bir xil natija
  setCount(count + 1); // ❌ count = 0 → 1 (3 emas)
}

// vs

function handleClick() {
  setCount((c) => c + 1); // ✅ Functional — latest c
  setCount((c) => c + 1); // ✅ Functional — c = 1 + 1
  setCount((c) => c + 1); // ✅ count = 0 → 3
}
```

R18+ Automatic Batching — har context'dagi multiple `setState` bir render'ga birlashadi (cross-ref `02-rendering.md`). R17'gacha batching faqat React event handler ichida ishladi (async context, Promise callback, native event'da emas):

```tsx
async function handleClick() {
  setLoading(true);
  setData(null);
  // R17 va R18+: bir render (loading=true, data=null) — handler ichida batched
  
  const result = await fetch('/api/data');
  // `await` keyin — Promise microtask, React event handler context tugagan
  
  setData(result);
  setLoading(false);
  // R17: 2 ta render (setData → render, setLoading → render — async context, no batch)
  // R18+: 1 ta render (data=result, loading=false birga) — automatic batching har context'da
}
// Jami: R17 = 3 render (1+2), R18+ = 2 render (1+1)
```

<details>
<summary><strong>Under the Hood</strong></summary>

`dispatchSetState` mexanizmi (cross-ref `12-state-and-usestate.md`):

```javascript
function dispatchSetState(fiber, queue, action) {
  const lane = requestUpdateLane(fiber);
  
  const update = {
    lane,
    action,
    eagerReducer: null,
    eagerState: null,
    next: null,
  };
  
  // Eager bailout check
  if (
    fiber.lanes === NoLanes &&
    (fiber.alternate === null || fiber.alternate.lanes === NoLanes)
  ) {
    const lastRenderedReducer = queue.lastRenderedReducer;
    if (lastRenderedReducer !== null) {
      const currentState = queue.lastRenderedState;
      const eagerState = lastRenderedReducer(currentState, action);
      
      if (Object.is(eagerState, currentState)) {
        // No render needed — bailout
        return;
      }
      
      update.eagerReducer = lastRenderedReducer;
      update.eagerState = eagerState;
    }
  }
  
  // Schedule render
  enqueueUpdate(fiber, queue, update);
  scheduleUpdateOnFiber(fiber, lane);
}
```

Dispatch-time eager bailout faqat `dispatchSetState`'da mavjud — `useState` setter shu funksiyaga bog'lanadi. `useReducer` dispatch (`dispatchReducerAction`) bu eager check'ni o'tkazmaydi: u update'ni queue'ga qo'yib, to'g'ridan-to'g'ri `scheduleUpdateOnFiber` chaqiradi. Sabab — `dispatchSetState` `basicStateReducer` (action function yoki value) ishlatadi va uni render'dan tashqari xavfsiz chaqira oladi; `useReducer` esa ixtiyoriy side-effect'li bo'lishi mumkin bo'lgan user-provided reducer'ni ishlatadi, shuning uchun uni dispatch paytida eager chaqirish o'tkazib yuboriladi.

`useReducer` uchun "teng state → bailout" render Phase'da sodir bo'ladi: `updateReducer` yangi state'ni hisoblaganda, agar natija joriy state bilan `Object.is` true bo'lsa, fiber `didReceiveUpdate = false` bilan belgilanib, child re-render'lar bailout qilinadi. Ya'ni `useReducer`'da reducer baribir render paytida chaqiriladi, shunchaki dispatch paytidagi tezkor qisqartirish yo'q.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Eager bailout demonstration:

```tsx
import { useState, useEffect, useRef } from 'react';
import type { ReactElement } from 'react';

function CounterDemo(): ReactElement {
  const [count, setCount] = useState(0);
  const renderCountRef = useRef(0);
  
  useEffect(() => {
    renderCountRef.current += 1;
  });
  
  return (
    <div>
      <p>Count: {count}, Renders: {renderCountRef.current}</p>
      <button onClick={() => setCount(0)}>Set to 0 (no render)</button>
      <button onClick={() => setCount(count)}>Set to current (no render)</button>
      <button onClick={() => setCount((c) => c + 1)}>Increment (render)</button>
    </div>
  );
}
// Birinchi 2 ta button — eager bailout, render count o'zgarmaydi.
// Faqat Increment button render trigger qiladi.
```

Object reference va re-render:

```tsx
import { useState } from 'react';
import type { ReactElement } from 'react';

interface User {
  name: string;
  age: number;
}

function UserCard(): ReactElement {
  const [user, setUser] = useState<User>({ name: 'Ali', age: 25 });
  
  function setSameReference() {
    setUser(user); // Same reference — eager bailout
  }
  
  function setNewReferenceSameValues() {
    setUser({ name: user.name, age: user.age }); // ❌ New ref → re-render
  }
  
  function setUpdated() {
    setUser((prev) => ({ ...prev, age: prev.age + 1 })); // ✅ New ref, new value
  }
  
  return (
    <div>
      <p>{user.name}, {user.age}</p>
      <button onClick={setSameReference}>Same ref (no render)</button>
      <button onClick={setNewReferenceSameValues}>New ref same values (re-render!)</button>
      <button onClick={setUpdated}>Update age (re-render)</button>
    </div>
  );
}
// "New ref same values" — eager bailout false (`Object.is` yangi reference uchun false),
// komponent funksiyasi qayta chaqiriladi (re-render). Lekin yangi JSX'da
// `{user.name}` va `{user.age}` matni avvalgisi bilan bir xil — Reconciler
// diff hech qanday DOM mutation aniqlamaydi (text content identical). 
// Profile: render count oshadi, lekin commit duration ≈ 0.
```

</details>

---

## Trigger 2: Parent Re-render — Top-Down Propagation

### Nazariya

**Default React behavior** — parent komponent re-render bo'lsa, **barcha children komponentlari ham re-render bo'ladi**. Bu Reconciler'ning fundamental qoidasi: parent funksiyasidan chiqqan yangi JSX tree'da har Element re-create bo'ladi (yangi reference), va Reconciler children'ni shu yangi tree bilan reconcile qiladi.

NIMA UCHUN top-down default:

1. **Reconciler dependency tracking yo'q** — Reconciler komponentlar orasidagi dependency'larni track qilmaydi (qaysi child qaysi parent prop'ga bog'liq). Conservative strategy: barchasini qayta hisoblash (children'ni re-render qilish), keyin diff orqali real DOM o'zgarishlarni aniqlash. Manual opt-in optimization (`React.memo`) yoki Compiler avtomatik bu strategy'ni o'zgartiradi.
2. **JSX yangi Element reference** — parent funksiyasi qayta chaqirilganda har JSX expression (`<Child />`) `React.createElement(Child, props)` chaqiriqiga aylanadi va **yangi obyekt** qaytaradi (props ham yangi obyekt). Reconciler `prevProps !== nextProps` ko'radi (reference inequality).
3. **Props o'zgarishi children'da ko'rinishi kerak** — parent yangi value uzatsa, children render natijasi yangilanishi kerak. Default re-render kafolat: hech bir prop change "yo'qotilmaydi".

Re-render propagation **DOM update bilan teng emas**:

```
Parent re-render
       │
       ├─ Child1 funksiyasi chaqiriladi
       │  - JSX hisoblanadi (yangi reference)
       │  - Reconciler diff hisoblaydi
       │  - Bailout (props bir xil) — DOM o'zgarmaydi
       │
       ├─ Child2 funksiyasi chaqiriladi
       │  - JSX hisoblanadi
       │  - Bailout — DOM o'zgarmaydi
       │
       └─ Child3 funksiyasi chaqiriladi
          - JSX yangi (props o'zgargan)
          - Reconciler diff hisoblaydi
          - DOM mutation
```

Top-down propagation'ni **to'xtatish usullari** (re-render skip):

1. **`React.memo`** — komponent props shallow comparison qiladi va props bir xil bo'lsa re-render qilmaydi (memo'd Fiber'da `bailoutOnAlreadyFinishedWork`).
2. **Children prop / composition** — JSX parent'da yaratilmaganligi sababli element reference o'zgarmaydi → Reconciler subtree'ni qayta diff qilmaydi (cross-ref `11-composition.md`).
3. **`useMemo` bilan element cache** — `useMemo(() => <Child />, [deps])` bilan stable element reference saqlash.
4. **Compiler auto-memoization** — `'use memo'` directive'da Compiler komponent darajasidagi memo va element cache avtomatik qo'shadi (cross-ref `31-react-compiler.md`).

> **Eslatma:** `React.lazy`/`<Suspense>` re-render propagation'ni to'xtatmaydi — ular **loading/error state'ini boshqarish vositalari**. Suspense fallback ko'rsatilganda primary children render qilinmaydi (suspended), lekin parent re-render bo'lsa Reconciler primary children'ni qayta sinab ko'radi. Bu memoization stsenariy emas, asynchronous state mexanizmi (cross-ref `29-suspense-lazy.md`).

**MUHIM farq — ikki turdagi "no DOM mutation":**

1. **Reconciler bailout (`bailoutOnAlreadyFinishedWork`)** — komponent funksiyasi **umuman chaqirilmaydi**. Shartlar: `prevProps === nextProps && !hasContextChanged() && fiber.lanes === NoLanes`. Default `FunctionComponent` Fiber'da bu kamdan-kam (parent re-render JSX yangi reference yaratadi). `MemoComponent` Fiber'da `React.memo` shallow check muvaffaqiyatli bo'lsa bailout sodir bo'ladi. JSX hisoblanmaydi, hooks chaqirilmaydi, children traverse qilinmaydi.

2. **Same diff result** — komponent funksiyasi **chaqiriladi** (re-render), JSX hisoblanadi, lekin Reconciler eski va yangi tree'ni solishtirgach bir xil natija topadi (text bir xil, attribute bir xil) — **hech qanday DOM mutation generate qilinmaydi**. Bu "bailout" emas — bu "diff produces empty mutation list".

NIMA UCHUN bu farq muhim:

```tsx
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveChild /> {/* Bu re-render bo'ladi har count change'da */}
    </>
  );
}

function ExpensiveChild() {
  // 1000 ta computation har render'da
  const result = heavyCompute();
  return <div>{result}</div>;
}
```

Bu yerda `ExpensiveChild` har Parent re-render'da chaqiriladi va `heavyCompute()` bajariladi. Hatto props yo'q bo'lsa ham, Reconciler komponent'ni "mustaqil" deb hisoblamaydi — uni o'zi chaqirib, JSX'ni qaytaradi va keyin diff qiladi.

Yechim: `React.memo`:

```tsx
const ExpensiveChild = memo(function ExpensiveChild() {
  const result = heavyCompute();
  return <div>{result}</div>;
});
// Endi props yo'q bo'lsa, har doim bailout — re-render bo'lmaydi.
```

<details>
<summary><strong>Under the Hood</strong></summary>

Reconciler render walk algorithm (oddiylashtirilgan):

```javascript
function performUnitOfWork(workInProgress) {
  const current = workInProgress.alternate;
  
  // 1. Begin work — komponent funksiyasini chaqirish
  const next = beginWork(current, workInProgress, renderLanes);
  
  // beginWork ichida:
  //   - Component(props) chaqiriladi
  //   - JSX qaytariladi
  //   - Diff hisoblanadi
  //   - Children Fiber'lar yaratiladi/yangilanadi
  
  if (next === null) {
    // 2. Complete work — leaf'ga yetdi, parent'ga qaytamiz
    completeUnitOfWork(workInProgress);
  } else {
    // 3. Children'ga o'tamiz
    workInProgress = next;
  }
}
```

`beginWork` ichidagi komponent type bo'yicha switch:

```javascript
function beginWork(current, workInProgress, renderLanes) {
  switch (workInProgress.tag) {
    case FunctionComponent:
      return updateFunctionComponent(current, workInProgress, ...);
    case MemoComponent:
      return updateMemoComponent(current, workInProgress, ...);
    case ClassComponent:
      return updateClassComponent(current, workInProgress, ...);
    // ...
  }
}

function updateFunctionComponent(current, workInProgress, ...) {
  // Bailout check
  if (current !== null) {
    const prevProps = current.memoizedProps;
    const nextProps = workInProgress.pendingProps;
    
    if (
      prevProps !== nextProps ||
      hasContextChanged() ||
      workInProgress.lanes !== NoLanes
    ) {
      // Render
      const nextChildren = renderWithHooks(current, workInProgress, Component, nextProps, ...);
      reconcileChildren(current, workInProgress, nextChildren);
      return workInProgress.child;
    } else {
      // Bailout
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
  }
  // First render
  // ...
}
```

`prevProps !== nextProps` — bu **reference comparison** (React internal'da `!==` strict inequality ishlatiladi props uchun; `Object.is` esa `useState` eager bailout va `React.memo` shallow check ichida ishlatiladi). Parent re-render qilganda, JSX yaratilgan paytda har children uchun yangi `pendingProps` object yaratiladi (`React.createElement` natijasi yangi `props` object). Shu sababdan default'da `prevProps !== nextProps` true (yangi reference) — children re-render bo'ladi.

`React.memo` esa **shallow comparison** qiladi — props object'ining property'larini birma-bir tekshiradi. Property values bir xil bo'lsa, render skip.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Top-down propagation demo:

```tsx
import { useState, useRef, useEffect } from 'react';
import type { ReactElement } from 'react';

function ChildA(): ReactElement {
  const renderCountRef = useRef(0);
  useEffect(() => {
    renderCountRef.current += 1;
  });
  return <span>ChildA renders: {renderCountRef.current}</span>;
}

function ChildB({ value }: { value: number }): ReactElement {
  const renderCountRef = useRef(0);
  useEffect(() => {
    renderCountRef.current += 1;
  });
  return <span>ChildB ({value}) renders: {renderCountRef.current}</span>;
}

function Parent(): ReactElement {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>Parent count: {count}</button>
      <ChildA />
      <ChildB value={42} />
    </div>
  );
}
// Har click: count o'zgaradi, Parent re-render.
// ChildA va ChildB ham re-render bo'ladi (top-down propagation),
// hatto props o'zgarmagan bo'lsa ham.
```

`React.memo` bilan to'xtatish:

```tsx
import { memo, useState, useRef, useEffect } from 'react';
import type { ReactElement } from 'react';

const MemoizedChildA = memo(function ChildA(): ReactElement {
  const renderCountRef = useRef(0);
  useEffect(() => {
    renderCountRef.current += 1;
  });
  return <span>ChildA renders: {renderCountRef.current}</span>;
});

const MemoizedChildB = memo(function ChildB({ value }: { value: number }): ReactElement {
  const renderCountRef = useRef(0);
  useEffect(() => {
    renderCountRef.current += 1;
  });
  return <span>ChildB ({value}) renders: {renderCountRef.current}</span>;
});

function Parent(): ReactElement {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>Parent count: {count}</button>
      <MemoizedChildA />
      <MemoizedChildB value={42} />
    </div>
  );
}
// Har click: Parent re-render, lekin MemoizedChildA/B props bir xil → bailout.
// ChildA va ChildB renders count: 1 (o'zgarmaydi).
```

Children composition — re-render scope cheklash:

```tsx
import { useState, useRef, useEffect } from 'react';
import type { ReactElement, ReactNode } from 'react';

function ExpensiveChild(): ReactElement {
  const renderCountRef = useRef(0);
  useEffect(() => {
    renderCountRef.current += 1;
  });
  return <div>Expensive renders: {renderCountRef.current}</div>;
}

// ❌ Anti-pattern: ExpensiveChild Counter ichida ishlatilib, har state change'da re-render
function DashboardWithInlineChild(): ReactElement {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <ExpensiveChild />
    </div>
  );
}

// ✅ Composition: ExpensiveChild Counter'ning tashqarisida yaratilib, children sifatida uzatilgan
function Counter({ children }: { children: ReactNode }): ReactElement {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      {children}
    </div>
  );
}

function DashboardWithComposition(): ReactElement {
  return (
    <Counter>
      <ExpensiveChild /> {/* Element Counter'ning tashqarisida (DashboardWithComposition) yaratilgan */}
    </Counter>
  );
}

function App(): ReactElement {
  return <DashboardWithComposition />;
}
// Counter state o'zgarsa — Counter re-render, lekin children ExpensiveChild
// reference Counter'ning tashqarisida yaratilgan va o'zgarmaydi — Reconciler bailout.
// ExpensiveChild renders: 1 (o'zgarmaydi).
```

</details>

---

## Trigger 3: Context Value Change

### Nazariya

**Context value change** — `Context.Provider`'ning `value` prop o'zgarganda **barcha consumer komponentlar** (`useContext`/`use(context)`) re-render bo'ladi. Bu shuningdek **`React.memo`'ni bypass qiladi** — Context dep memo'dan kuchliroq.

Context propagation lifecycle:

1. `<Context.Provider value={...}>` JSX'da yangi value berildi.
2. Reconciler Provider Fiber'da `value` o'zgarganini aniqlaydi (`Object.is` comparison).
3. `propagateContextChange` chaqiriladi — barcha subtree'dagi `useContext`/`use(context)` ishlatuvchi Fiber'larni topadi.
4. Topilgan Fiber'larga lane priority assigned — ular re-render bo'lishi kerak.
5. Render Phase'da consumer'lar yangi value bilan render bo'ladi.

**MUHIM nuance** — Context propagation **`React.memo`'dan kuchliroq**:

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

const MemoizedConsumer = memo(function Consumer() {
  const theme = useContext(ThemeContext);
  return <div className={theme}>Hello</div>;
});

function App() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Toggle
      </button>
      <MemoizedConsumer />
    </ThemeContext.Provider>
  );
}
// Toggle bosilganda: theme o'zgaradi, Provider value o'zgaradi.
// MemoizedConsumer.props o'zgarmagan bo'lsa ham — Context o'zgargan,
// React.memo bypass — Consumer re-render bo'ladi.
```

Bu mantiqiy: Consumer Context'dan value o'qiydi va shu value asosida render qiladi. Agar Context o'zgarsa, lekin Consumer re-render bo'lmasa — UI eski value ko'rsatadi (silent bug).

Context value gotchas:

### Gotcha 1: Object literal Provider value

```tsx
// ❌ Anti-pattern: har render yangi object
function App() {
  const [count, setCount] = useState(0);
  
  return (
    <UserContext.Provider value={{ name: 'Ali', count }}>
      <Children />
    </UserContext.Provider>
  );
}
// {{ name, count }} har render yangi reference.
// Object.is comparison false → barcha consumers re-render har count change'da.
// Hatto consumer faqat name'ni ishlatsa ham!
```

Yechim — `useMemo` (yoki React Compiler avtomatik):

```tsx
function App() {
  const [count, setCount] = useState(0);
  
  const value = useMemo(() => ({ name: 'Ali', count }), [count]);
  
  return (
    <UserContext.Provider value={value}>
      <Children />
    </UserContext.Provider>
  );
}
// value reference faqat count o'zgarsa yangi.
```

### Gotcha 2: Context split

Bir Context'da turli o'zgaruvchilik darajasidagi value'lar:

```tsx
// ❌ One large Context
const AppContext = createContext({
  user: null,         // O'zgarmaydi (login'da bir marta)
  theme: 'light',     // Kamdan-kam o'zgaradi
  notifications: [],  // Tez-tez o'zgaradi
});
// notifications.push() har consumer (user reader, theme reader) re-render qiladi.

// ✅ Split Contexts
const UserContext = createContext(null);
const ThemeContext = createContext('light');
const NotificationsContext = createContext([]);
// Notifications o'zgarsa — faqat NotificationsContext consumers re-render.
```

### Gotcha 3: Selector pattern

Library'lar (Zustand, Redux) `useSelector` orqali partial subscription beradi:

```tsx
const theme = useSelector((state) => state.theme);
// Faqat state.theme o'zgarsa re-render.
```

Vanilla React'da `use-context-selector` library yoki `useSyncExternalStore` bilan custom selector pattern.

<details>
<summary><strong>Under the Hood</strong></summary>

`propagateContextChange` algorithm (oddiylashtirilgan):

```javascript
function propagateContextChange(workInProgress, context, renderLanes) {
  let fiber = workInProgress.child;
  
  while (fiber !== null) {
    let nextFiber;
    
    // Fiber.dependencies — Context'larga subscription
    const list = fiber.dependencies;
    
    if (list !== null) {
      let dependency = list.firstContext;
      while (dependency !== null) {
        if (dependency.context === context) {
          // Bu Fiber Context'ni o'qiydi → mark for update
          fiber.lanes = mergeLanes(fiber.lanes, renderLanes);
          
          // Alternate'ni ham yangilang
          const alternate = fiber.alternate;
          if (alternate !== null) {
            alternate.lanes = mergeLanes(alternate.lanes, renderLanes);
          }
          
          // Parent chain bo'ylab childLanes'ni yangilang (cross-ref 04 bailout)
          scheduleContextWorkOnParentPath(fiber.return, renderLanes);
          
          break;
        }
        dependency = dependency.next;
      }
    }
    
    // Continue DFS traversal
    if (fiber.child !== null) {
      fiber = fiber.child;
    } else {
      // ...
    }
  }
}
```

Har consumer Fiber'ning `dependencies.firstContext` linked list'i bor — qaysi Context'larga subscribe qilingan. Provider value o'zgarsa, propagation algorithm subtree'ni traverse qilib, har subscriber'ga lane priority qo'shadi.

`React.memo` bypass — chunki `propagateContextChange` consumer Fiber'ga **render lane** qo'shadi (`fiber.lanes = mergeLanes(...)`). `updateMemoComponent` shallow props comparison'ga o'tishdan oldin Fiber'da kutilayotgan ish borligini tekshiradi (`checkScheduledUpdateOrContext`); lane mavjud bo'lsa, shallow bailout o'tkazib yuboriladi va consumer re-render bo'ladi. Shu sababli props bir xil bo'lsa ham, Context o'zgargani memo'ni "yengadi".

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Context split pattern:

```tsx
import { createContext, useContext, useState, useMemo, useReducer } from 'react';
import type { ReactElement, ReactNode } from 'react';

interface User {
  id: string;
  name: string;
}

// State Context (read)
const UserContext = createContext<User | null>(null);

// Dispatch Context (write) — stable reference
const UserDispatchContext = createContext<((user: User | null) => void) | null>(null);

interface UserProviderProps {
  children: ReactNode;
}

function UserProvider({ children }: UserProviderProps): ReactElement {
  const [user, setUser] = useState<User | null>(null);
  
  return (
    <UserContext.Provider value={user}>
      <UserDispatchContext.Provider value={setUser}>
        {children}
      </UserDispatchContext.Provider>
    </UserContext.Provider>
  );
}

// Consumer hooks
function useUser(): User | null {
  return useContext(UserContext);
}

function useSetUser(): (user: User | null) => void {
  const setUser = useContext(UserDispatchContext);
  if (setUser === null) {
    throw new Error('useSetUser must be used within UserProvider');
  }
  return setUser;
}

// Komponent faqat user'ni o'qiydi
function UserDisplay(): ReactElement {
  const user = useUser();
  return <span>{user?.name ?? 'Guest'}</span>;
}

// Komponent faqat setUser'ni ishlatadi
function LoginButton(): ReactElement {
  const setUser = useSetUser();
  return (
    <button onClick={() => setUser({ id: '1', name: 'Ali' })}>
      Login
    </button>
  );
}
// LoginButton faqat dispatch context'ga subscribe — user o'zgarsa LoginButton re-render bo'lmaydi.
// UserDisplay user context'ga subscribe — faqat user o'zgarsa re-render.
```

Provider value memoization:

```tsx
import { createContext, useContext, useState, useMemo, useCallback } from 'react';
import type { ReactElement, ReactNode } from 'react';

interface CartContextValue {
  items: string[];
  addItem: (item: string) => void;
  removeItem: (item: string) => void;
}

const CartContext = createContext<CartContextValue | null>(null);

interface CartProviderProps {
  children: ReactNode;
}

function CartProvider({ children }: CartProviderProps): ReactElement {
  const [items, setItems] = useState<string[]>([]);
  
  // ✅ Stable handlers
  const addItem = useCallback((item: string) => {
    setItems((prev) => [...prev, item]);
  }, []);
  
  const removeItem = useCallback((item: string) => {
    setItems((prev) => prev.filter((i) => i !== item));
  }, []);
  
  // ✅ Memoized value
  const value = useMemo<CartContextValue>(
    () => ({ items, addItem, removeItem }),
    [items, addItem, removeItem]
  );
  
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}

// React Compiler avtomatik:
// 'use memo';
// function CartProvider({ children }) {
//   const [items, setItems] = useState([]);
//   const addItem = (item) => setItems(prev => [...prev, item]);
//   const removeItem = (item) => setItems(prev => prev.filter(i => i !== item));
//   const value = { items, addItem, removeItem };
//   return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
// }
// Compiler value object'ni va handlers'ni avtomatik memoize qiladi.
```

</details>

---

## Trigger 4: Force Update Pattern

### Nazariya

**Force update** — komponent'ni qo'l bilan re-render qilish. React'da rasmiy `forceUpdate` API hooks'da yo'q (class component'da `this.forceUpdate()`), lekin pattern'lar mavjud:

### Pattern 1: `useReducer({})` increment

```tsx
function useForceUpdate(): () => void {
  const [, forceUpdate] = useReducer((x: number) => x + 1, 0);
  return forceUpdate;
}

function Component() {
  const forceUpdate = useForceUpdate();
  
  return <button onClick={forceUpdate}>Re-render</button>;
}
```

### Pattern 2: `useState({})`

```tsx
function useForceUpdate(): () => void {
  const [, setState] = useState({});
  return () => setState({});
}
```

NIMA UCHUN ishlaydi:

- `useReducer((x) => x + 1, 0)` — har dispatch render'ni schedule qiladi; render Phase'da reducer yangi number qaytaradi va avvalgisi bilan teng chiqmaydi → render.
- `useState({})` — `{}` har gal yangi reference, `Object.is({}, {})` false → render.

QACHON force update kerak:

1. **External mutable state** — komponent state emas, lekin mutate bo'lgan (`useRef.current`, third-party library state).
2. **Imperative library integration** — Three.js, D3.js, vanilla DOM mutation — komponent qayta render qilishi kerak.
3. **Debug** — manual re-render kuzatish uchun (development only).

QACHON force update **anti-pattern**:

1. **State'ga aylantirib bo'ladigan holat** — `useState`/`useReducer` afzal.
2. **Effect'da setState** — derived state (cross-ref `16-useeffect.md` "You Might Not Need an Effect").
3. **Side effect propagation** — context/event handler afzal.

Modern alternativ — `useSyncExternalStore` (R18+, cross-ref `22-concurrent-hooks.md`):

```tsx
// External mutable store + force re-render — yo'q
function useExternalCount() {
  const [, forceUpdate] = useReducer((x) => x + 1, 0);
  
  useEffect(() => {
    return externalStore.subscribe(forceUpdate);
  }, []);
  
  return externalStore.count;
}

// ✅ Modern: useSyncExternalStore (Concurrent-safe, cross-ref 30 tearing prevention)
function useExternalCount(): number {
  return useSyncExternalStore(
    externalStore.subscribe,
    () => externalStore.count,
    () => 0 // SSR
  );
}
```

`useSyncExternalStore` afzal — Concurrent rendering'da tearing prevention, library author primitive.

<details>
<summary><strong>Under the Hood</strong></summary>

`useReducer` increment mexanizmi (reducer render Phase'da chaqiriladi):

```javascript
// reducer: (x) => x + 1
// initial: 0

// First dispatch → render Phase (updateReducer):
//   state = 0
//   newState = reducer(0) = 1
//   Object.is(0, 1) === false → didReceiveUpdate = true → render

// Second dispatch → render Phase:
//   state = 1
//   newState = reducer(1) = 2
//   Object.is(1, 2) === false → render
```

`useReducer` dispatch'da eager bailout yo'q — har dispatch render'ni schedule qiladi. Reducer render Phase'da chaqirilib, natija avvalgi state bilan `Object.is` solishtiriladi; increment har gal yangi qiymat bergani uchun teng chiqmaydi va render bo'ladi.

`useState({})`:

```javascript
// initial: {}
// setState({}) chaqirilganda:
//   prev = {} (reference R1)
//   next = {} (reference R2)
//   Object.is(R1, R2) === false → render
```

`{}` literal har chaqiriqda yangi heap allocation, yangi reference.

Force update'ning **performance cost**:

- 1 ta state slot (Hook chain'da)
- Re-render trigger qilish overhead'i (lane scheduling)
- Komponent funksiyasi chaqiriladi
- Reconciler diff hisoblaydi
- Bailout bo'lmasa — children ham re-render

Bu cost manual force update'ni "free" emas — har chaqiriq performance impact'i bor.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Force update hook:

```tsx
import { useReducer, useEffect, useRef } from 'react';
import type { ReactElement } from 'react';

function useForceUpdate(): () => void {
  const [, forceUpdate] = useReducer((x: number) => x + 1, 0);
  return forceUpdate;
}

// Use case: External mutable state
let globalNotifications: string[] = [];
const subscribers = new Set<() => void>();

function addNotification(msg: string) {
  globalNotifications.push(msg);
  subscribers.forEach((s) => s());
}

function NotificationList(): ReactElement {
  const forceUpdate = useForceUpdate();
  
  useEffect(() => {
    subscribers.add(forceUpdate);
    return () => {
      subscribers.delete(forceUpdate);
    };
  }, [forceUpdate]);
  
  return (
    <ul>
      {globalNotifications.map((msg, i) => (
        <li key={i}>{msg}</li>
      ))}
    </ul>
  );
}

// Eslatma: bu pattern Concurrent rendering'da tearing xavfi.
// Modern yondashuv — useSyncExternalStore:

import { useSyncExternalStore } from 'react';

function subscribe(callback: () => void): () => void {
  subscribers.add(callback);
  return () => {
    subscribers.delete(callback);
  };
}

function getSnapshot(): string[] {
  return globalNotifications;
}

function NotificationListModern(): ReactElement {
  const notifications = useSyncExternalStore(
    subscribe,
    getSnapshot,
    () => []
  );
  
  return (
    <ul>
      {notifications.map((msg, i) => (
        <li key={i}>{msg}</li>
      ))}
    </ul>
  );
}
// Eslatma: getSnapshot ma'lumot o'zgarmaganda bir xil reference, o'zgarganda yangi
// reference qaytarishi kerak (React natijani Object.is bilan solishtiradi).
// globalNotifications.push() mutation reference'ni saqlaydi — Object.is true →
// React update'ni sezmaydi. Yaxshiroq — immutable copy:

function addNotificationImmutable(msg: string) {
  globalNotifications = [...globalNotifications, msg]; // Yangi reference
  subscribers.forEach((s) => s());
}
```

Imperative library integration:

```tsx
import { useEffect, useRef, useReducer } from 'react';
import type { ReactElement } from 'react';

interface ExternalStore {
  data: { x: number; y: number };
  subscribe(cb: () => void): () => void;
}

function ChartComponent({ store }: { store: ExternalStore }): ReactElement {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const [, forceRedraw] = useReducer((x: number) => x + 1, 0);
  
  useEffect(() => {
    return store.subscribe(forceRedraw);
  }, [store]);
  
  useEffect(() => {
    if (!canvasRef.current) return;
    const ctx = canvasRef.current.getContext('2d');
    if (!ctx) return;
    
    ctx.clearRect(0, 0, 400, 300);
    ctx.fillStyle = 'blue';
    ctx.fillRect(store.data.x, store.data.y, 10, 10);
  }); // No deps — har render
  
  return <canvas ref={canvasRef} width={400} height={300} />;
}
```

</details>

---

## Reconciler Bailout — Re-render Skip Manbai

### Nazariya

**Reconciler bailout** — komponent funksiyasini **umuman chaqirmaslik** (skip). Bu `bailoutOnAlreadyFinishedWork` ichki funksiya orqali sodir bo'ladi: agar Fiber'da pending update yo'q va props/context o'zgarmagan bo'lsa, Reconciler komponent body'ni chaqirmaydi va saqlangan eski natija ishlatiladi. **Diqqat: bu "DOM mutation yo'q" sodir bo'ladigan boshqa stsenariy emas** — bailout'da hatto funksiya body ham chaqirilmaydi (hooks ham, JSX hisoblanmaydi, side effect bo'lishi mumkin bo'lgan log'lar ham yo'q).

Bailout 3 ta asosiy mexanizm orqali sodir bo'ladi (cross-ref `04-reconciliation.md`):

1. **Eager bailout (dispatch vaqtida)** — `setState(sameValue)` chaqirilsa, `dispatchSetState` darhol `Object.is(eagerState, currentState)` tekshiradi va teng bo'lsa render schedule qilmaydi. Bu dispatch-time qisqartirish faqat `useState`'da mavjud; `useReducer` dispatch'ida (`dispatchReducerAction`) yo'q — `useReducer`'da "teng state → bailout" render Phase'da, `updateReducer` reducer'ni chaqirib natijani `Object.is` bilan solishtirgach sodir bo'ladi.
2. **Reference bailout (`bailoutOnAlreadyFinishedWork`)** — render walk paytida `prevProps === nextProps && !hasContextChanged() && fiber.lanes === NoLanes` bo'lsa bailout. Default `FunctionComponent` Fiber'da kamdan-kam (parent re-render JSX yangi reference yaratadi); `SimpleMemoComponent`/`MemoComponent` Fiber'da (`React.memo` natijasi) shallow check muvaffaqiyatli bo'lsa sodir bo'ladi. `forwardRef` o'zi shallow comparison qilmaydi — bailout uchun `memo(forwardRef(...))` kerak.
3. **Element identity bailout** — JSX element reference parent ichida saqlanib qolsa (`useMemo(() => <Child />, deps)` yoki composition orqali `children` prop sifatida uzatilsa), Reconciler `oldElement === newElement` ko'radi va shu subtree'ni qayta diff qilmaydi (faqat element identity bailout, props o'zgarmagani sababli).

`useMemo`/`useCallback` o'zlari bailout emas — ular **stable reference berib bailout sodir bo'lishiga sharoit yaratadigan vositalar** (props/element reference bir xil bo'lishi uchun).

**Bailout natijasi: re-render umuman bo'lmaydi** (funksiya chaqirilmaydi → JSX yo'q → diff yo'q → DOM mutation yo'q):

```tsx
function Parent({ name }: { name: string }) {
  return (
    <>
      <h1>{name}</h1>
      <ExpensiveChild /> {/* Bailout candidate */}
    </>
  );
}

const ExpensiveChild = memo(function ExpensiveChild() {
  return <div>Heavy work</div>;
});

function App() {
  const [name, setName] = useState('Ali');
  return <Parent name={name} />;
}
// Parent re-render bo'ladi har name change'da.
// ExpensiveChild props yo'q → memo bailout → komponent chaqirilmaydi.
```

Lekin bailout **propagation'ni avtomatik to'xtatmaydi**:

```tsx
function Parent({ name }: { name: string }) {
  return (
    <>
      <h1>{name}</h1>
      <Child /> {/* memo'siz */}
      <ExpensiveDescendant />
    </>
  );
}

function Child() {
  return (
    <div>
      <ExpensiveDescendant />
    </div>
  );
}

const ExpensiveDescendant = memo(function ExpensiveDescendant() {
  return <span>...</span>;
});
// Parent re-render → Child re-render (memo'siz) →
//   Child ichidagi <ExpensiveDescendant /> JSX yangi reference,
//   lekin props bir xil → ExpensiveDescendant memo bailout.
```

Bailout strategy'lari:

| Pattern | Bailout sodir bo'ladi? |
|---------|------------------------|
| `<Child />` (default) | Yo'q (har Parent re-render'da Child re-render) |
| `memo(Child)` (props bir xil) | Ha (komponent chaqirilmaydi) |
| `<Child key={stableKey} />` | Yo'q (key faqat identity, render qilinadi) |
| `<Child {...spreadProps} />` (yangi reference har render) | Yo'q (memo bo'lsa ham bypass) |
| `useMemo(() => <Child />, [deps])` | Element identity bailout (`<Child />` reference saqlanadi) |
| Children prop / composition | Element identity bailout (Parent ichida yaratilmagan) |

**Bailout DOM mutation'ga ta'siri va o'xshash stsenariy farqi:**

- **Bailout sodir bo'lsa** — komponent funksiyasi chaqirilmaydi → JSX yo'q → diff yo'q → DOM mutation yo'q. Bu eng samarali stsenariy.
- **Bailout bo'lmasa (komponent re-render'da)**, lekin yangi JSX eski bilan **identical** bo'lsa — komponent funksiyasi chaqiriladi, hooks ishlaydi, JSX hisoblanadi, lekin Reconciler diff (`reconcileChildren`) hech qanday DOM mutation generate qilmaydi (text, attribute, child element bir xil). **Bu bailout EMAS** — bu "diff produces empty mutation list" deb ataladigan boshqa stsenariy. CPU sarflanadi (function body + hooks + JSX), faqat DOM kafolat qilingan.

Performance: **bailout funksiya chaqirig'ini saqlaydi** (eng katta saving). "Empty diff" esa faqat DOM mutation overhead'ini saqlaydi, lekin function CPU sarflaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

Bailout decision tree (oddiylashtirilgan):

```javascript
function bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes) {
  if (current !== null) {
    // Saved alternate exist — bailout possible
    workInProgress.dependencies = current.dependencies;
  }
  
  // Check if any children have updates
  const childLanes = workInProgress.childLanes;
  
  if (!includesSomeLane(renderLanes, childLanes)) {
    // No children updates — full bailout
    return null;
  }
  
  // Some children need update — partial bailout
  // Clone children but skip current Fiber's render
  cloneChildFibers(current, workInProgress);
  return workInProgress.child;
}
```

`childLanes` optimization (R17+) — Reconciler `Fiber.childLanes` bitmap'da subtree'da update bor-yo'qligini saqlaydi. Bailout bo'lsa va childLanes ham bo'sh bo'lsa, butun subtree'ni tashlab ketamiz (ko'p bailout).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Bailout va re-render kuzatish:

```tsx
import { memo, useState, useRef, useEffect, useCallback } from 'react';
import type { ReactElement } from 'react';

function useRenderCount(name: string): number {
  const ref = useRef(0);
  useEffect(() => {
    ref.current += 1;
    console.log(`[${name}] render #${ref.current}`);
  });
  return ref.current;
}

// Ikki child — memo'li va memo'siz
const MemoChild = memo(function MemoChild({ value }: { value: number }): ReactElement {
  useRenderCount('MemoChild');
  return <span>{value}</span>;
});

function PlainChild({ value }: { value: number }): ReactElement {
  useRenderCount('PlainChild');
  return <span>{value}</span>;
}

function Parent(): ReactElement {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>Count: {count}</button>
      <MemoChild value={42} /> {/* Bailout: value bir xil */}
      <PlainChild value={42} /> {/* No bailout: top-down propagation */}
    </div>
  );
}
// Click har gal:
//   [Parent] render #N
//   [PlainChild] render #N (top-down)
// MemoChild render count o'zgarmaydi (bailout).
```

Element identity bailout via `useMemo`:

```tsx
import { useMemo, useState, memo } from 'react';
import type { ReactElement } from 'react';

const Child = memo(function Child({ data }: { data: { x: number } }): ReactElement {
  return <div>{data.x}</div>;
});

function ChartPanelInlineProps(): ReactElement {
  const [count, setCount] = useState(0);
  
  // ❌ Inline object — har render yangi reference
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <Child data={{ x: 42 }} />
    </>
  );
}
// Child memo bo'lsa ham — props.data har render yangi reference → bailout false.

function ChartPanelStableProps(): ReactElement {
  const [count, setCount] = useState(0);
  
  // ✅ Stable reference via useMemo
  const data = useMemo(() => ({ x: 42 }), []);
  
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <Child data={data} />
    </>
  );
}
// data reference bir xil → Child memo bailout → render skip.
```

Compiler era — automatic bailout:

```tsx
'use memo';

import { useState } from 'react';
import type { ReactElement } from 'react';

interface Item {
  id: string;
  name: string;
}

interface ListProps {
  items: Item[];
}

// Compiler avtomatik memo wrap
function List({ items }: ListProps): ReactElement {
  return (
    <ul>
      {items.map((i) => (
        <li key={i.id}>{i.name}</li>
      ))}
    </ul>
  );
}

function App(): ReactElement {
  const [count, setCount] = useState(0);
  
  // Compiler avtomatik items array'ni cache qiladi
  const items: Item[] = [
    { id: '1', name: 'Apple' },
    { id: '2', name: 'Banana' },
  ];
  
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <List items={items} />
    </>
  );
}
// Compiler items'ni har render bir xil reference bilan saqlaydi (deps yo'q).
// List memo equivalent + props bir xil → bailout.
// Manual: useMemo(() => [...], []) yoki module-level const.
```

</details>

---

## `React.memo` va Shallow Comparison

### Nazariya

**`React.memo`** — komponent'ni shallow props comparison bilan memoize qiluvchi HOC. Props bir xil bo'lsa render skip, aks holda render. Default behavior — `Object.is` har property uchun.

```tsx
import { memo } from 'react';

interface ToggleButtonProps {
  value: string;
  onClick: () => void;
}

const ToggleButton = memo(function ToggleButton({ value, onClick }: ToggleButtonProps) {
  return <button onClick={onClick}>{value}</button>;
});
```

Shallow comparison algorithm:

```typescript
function shallowEqual(prevProps, nextProps) {
  if (Object.is(prevProps, nextProps)) return true;
  
  if (
    typeof prevProps !== 'object' ||
    prevProps === null ||
    typeof nextProps !== 'object' ||
    nextProps === null
  ) return false;
  
  const keysA = Object.keys(prevProps);
  const keysB = Object.keys(nextProps);
  
  if (keysA.length !== keysB.length) return false;
  
  for (const key of keysA) {
    if (
      !Object.prototype.hasOwnProperty.call(nextProps, key) ||
      !Object.is(prevProps[key], nextProps[key])
    ) {
      return false;
    }
  }
  
  return true;
}
```

Props har property level'ida solishtiriladi (1 daraja chuqurlik):

- **Primitive** — `Object.is` (string/number/boolean ekvivalentlik).
- **Object/Array reference** — `Object.is` (reference equality, shape e'tiborga olinmaydi).
- **Function reference** — `Object.is` (callback bir xil reference bo'lishi shart).

NIMA UCHUN faqat shallow:

- **Performance** — deep comparison O(n) har render'da expensive.
- **Predictable** — reference equality semantic'ini React'ning boshqa joylarida ham ishlatiladi (Reconciler, Hook deps).

QACHON `React.memo` foydali:

1. **Pure presentational komponent** — props uzatilgan va boshqa hech qanday holat yo'q.
2. **Tez-tez re-render bo'luvchi parent** — parent har keystroke render bo'lsa, child memo afzal.
3. **Expensive render** — virtualized list item, chart, animation.

QACHON `React.memo` befoyda:

1. **Props har gal yangi reference** — inline object/array/function har render yangi → memo bypass har gal.
2. **Komponent doim re-render** — props o'zgarib turadi (ko'p reactive value'lar).
3. **Cheap render** — komponent oddiy span/div, memo overhead render cost'idan ko'p.

`React.memo` custom comparator:

```tsx
const UserHeader = memo(
  function UserHeader({ user }: { user: User }) {
    return <h1>{user.name}</h1>;
  },
  (prevProps, nextProps) => {
    // Custom check: faqat user.name'ga qarash
    return prevProps.user.name === nextProps.user.name;
  }
);
```

Custom comparator return value:
- `true` — props teng → re-render skip
- `false` — props farq → re-render

NIMA UCHUN custom kerak:

- **Deep comparison** — nested object'lar (lekin performance trade-off).
- **Property subset** — faqat ba'zi property'larni tracking.
- **Manual logic** — `user.id === prev.user.id` identity'ga asoslangan check.

> **Eslatma:** React Compiler enabled loyihalarda `React.memo` aksariyat holatda kerak emas — Compiler avtomatik komponent darajasidagi memoization qo'shadi (cross-ref `31-react-compiler.md`).

<details>
<summary><strong>Under the Hood</strong></summary>

`React.memo` HOC implementation:

```javascript
function memo(type, compare) {
  return {
    $$typeof: REACT_MEMO_TYPE,
    type: type,
    compare: compare === undefined ? null : compare,
  };
}
```

Reconciler'da `MemoComponent` Fiber tag handling:

```javascript
function updateMemoComponent(current, workInProgress, Component, nextProps) {
  if (current !== null) {
    const prevProps = current.memoizedProps;
    
    // Comparator check
    let isPropsEqual;
    if (Component.compare) {
      // Custom comparator
      isPropsEqual = Component.compare(prevProps, nextProps);
    } else {
      // Default shallow equal
      isPropsEqual = shallowEqual(prevProps, nextProps);
    }
    
    if (isPropsEqual && current.ref === workInProgress.ref) {
      // Bailout
      return bailoutOnAlreadyFinishedWork(current, workInProgress, ...);
    }
  }
  
  // Render component
  // ...
}
```

`React.memo` Fiber tag — `MemoComponent` (cross-ref `03-fiber-architecture.md`).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Default shallow comparison:

```tsx
import { memo, useState } from 'react';
import type { ReactElement } from 'react';

interface CardProps {
  title: string;
  count: number;
  onAction: () => void;
}

const Card = memo(function Card({ title, count, onAction }: CardProps): ReactElement {
  return (
    <div>
      <h3>{title}</h3>
      <p>Count: {count}</p>
      <button onClick={onAction}>Action</button>
    </div>
  );
});

function App(): ReactElement {
  const [globalCount, setGlobalCount] = useState(0);
  
  // ❌ Inline function — har render yangi reference
  return (
    <div>
      <button onClick={() => setGlobalCount((c) => c + 1)}>Global: {globalCount}</button>
      <Card
        title="Test"
        count={42}
        onAction={() => console.log('Action!')} // ❌ New ref har render
      />
    </div>
  );
}
// Card memo bypass: title='Test' bir xil, count=42 bir xil, onAction yangi reference.
// Re-render har globalCount change'da.

// ✅ useCallback bilan stabilize
import { useCallback } from 'react';

function AppFixed(): ReactElement {
  const [globalCount, setGlobalCount] = useState(0);
  
  const handleAction = useCallback(() => {
    console.log('Action!');
  }, []); // Deps yo'q — bir marta yaratiladi
  
  return (
    <div>
      <button onClick={() => setGlobalCount((c) => c + 1)}>Global: {globalCount}</button>
      <Card
        title="Test"
        count={42}
        onAction={handleAction}
      />
    </div>
  );
}
// Card memo bailout: barcha props bir xil reference.
```

Custom comparator:

```tsx
import { memo } from 'react';
import type { ReactElement } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
  lastLogin: Date;
}

interface UserBadgeProps {
  user: User;
}

const UserBadge = memo(
  function UserBadge({ user }: UserBadgeProps): ReactElement {
    return (
      <div>
        <h3>{user.name}</h3>
        <p>{user.email}</p>
      </div>
    );
  },
  (prev, next) => {
    // Faqat name va email muhim — lastLogin har refresh'da o'zgaradi
    return (
      prev.user.name === next.user.name &&
      prev.user.email === next.user.email
    );
  }
);

// Real-world: custom comparator id-based
const UserBadgeById = memo(
  function UserBadgeById({ user }: UserBadgeProps): ReactElement {
    return <h3>{user.name}</h3>;
  },
  (prev, next) => prev.user.id === next.user.id
);
// User reference o'zgarsa ham, id bir xil bo'lsa render skip.
// Risk: agar user.name o'zgarsa lekin id bir xil — eski name ko'rsatadi (bug).
```

</details>

---

## `React.memo` Bypass Scenarios

### Nazariya

`React.memo` **5 ta tipik scenario**'da bypass bo'ladi (memo ishlamaydi, komponent re-render bo'ladi):

### Scenario 1: Inline Object/Array/Function Props

```tsx
// ❌ Inline literals — har render yangi reference
<Memoized data={{ x: 1 }} />
<Memoized items={[1, 2, 3]} />
<Memoized onClick={() => console.log('hi')} />
```

Yechim: `useMemo`/`useCallback` yoki module-level constant:

```tsx
// ✅ Stable reference
const data = useMemo(() => ({ x: 1 }), []);
const items = useMemo(() => [1, 2, 3], []);
const onClick = useCallback(() => console.log('hi'), []);

<Memoized data={data} items={items} onClick={onClick} />
```

Yoki React Compiler avtomatik (cross-ref `31-react-compiler.md`).

### Scenario 2: Context Dep

```tsx
const ThemeContext = createContext('light');

const Memoized = memo(function Memoized() {
  const theme = useContext(ThemeContext); // Context dep
  return <div className={theme}>...</div>;
});

// Context value o'zgarsa — Memoized re-render bo'ladi (memo bypass).
```

Bu memo'ning fundamental cheklovi: Context dep memo'dan kuchliroq (yuqoridagi Trigger 3 bo'limi).

### Scenario 3: Children Prop O'zgarishi

```tsx
const Memoized = memo(function Memoized({ children }: { children: ReactNode }) {
  return <div>{children}</div>;
});

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <Memoized>
      <span>{count}</span> {/* Har count change'da yangi JSX */}
    </Memoized>
  );
}
// children prop har render yangi React Element reference.
// Memoized props.children !== prev.children → memo bypass.
```

### Scenario 4: Spread Props bilan Unstable Nested Values

Spread o'z-o'zidan bypass keltirib chiqarmaydi (shallow comparison har property'ni alohida tekshiradi, primitive value'lar bir xil bo'lsa equal). Lekin spread'da **nested unstable reference** bo'lsa, har render yangi inner object/array/function → bypass:

```tsx
function Parent() {
  const [count, setCount] = useState(0);
  
  // ❌ Inner config — har render yangi nested reference
  const allProps = {
    name: 'Ali',          // primitive, OK
    config: { lang: 'uz' }, // ❌ inline object — yangi reference har render
    onTrigger: () => alert(count), // ❌ inline function — yangi reference
  };
  
  return <Memoized {...allProps} />;
}

const Memoized = memo(function Memoized(props: { name: string; config: { lang: string }; onTrigger: () => void }) {
  return <div onClick={props.onTrigger}>{props.name} / {props.config.lang}</div>;
});
// React.memo shallow comparison:
// - prev.name === next.name → equal (primitive)
// - prev.config !== next.config → NOT equal (yangi obyekt reference)
// - prev.onTrigger !== next.onTrigger → NOT equal (yangi function reference)
// → bypass har render
```

**Yechim:** spread ichidagi nested object/array/function'larni `useMemo`/`useCallback` bilan stabilize, yoki module-level constant. Faqat primitive value'lar spread'da problem emas.

### Scenario 5: Ref Prop O'zgarishi

```tsx
const Memoized = memo(function Memoized({ value }: { value: number }) {
  return <div>{value}</div>;
});

function Parent() {
  const ref1 = useRef(null);
  const ref2 = useRef(null);
  const [useFirst, setUseFirst] = useState(true);
  
  return <Memoized ref={useFirst ? ref1 : ref2} value={42} />;
}
// useFirst toggle — ref reference o'zgaradi → memo bypass.
```

R18'da `forwardRef` shart edi memo'da ref pass qilish uchun. R19'da ref oddiy prop, memo o'zgarmasdan ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

`React.memo` shallow comparison source-code style:

```javascript
function shallowEqual(prevProps, nextProps) {
  if (Object.is(prevProps, nextProps)) return true;
  
  if (
    typeof prevProps !== 'object' || prevProps === null ||
    typeof nextProps !== 'object' || nextProps === null
  ) {
    return false;
  }
  
  const keysA = Object.keys(prevProps);
  const keysB = Object.keys(nextProps);
  
  if (keysA.length !== keysB.length) return false;
  
  for (const key of keysA) {
    if (!hasOwn(nextProps, key) || !Object.is(prevProps[key], nextProps[key])) {
      return false;
    }
  }
  
  return true;
}
```

Bypass har property level'ida `Object.is` fail bo'lganda sodir bo'ladi. Object literal `{ x: 1 }` har render yangi heap allocation — `Object.is({x:1}, {x:1}) === false`.

Children prop edge case:

```javascript
// JSX <Memoized><span /></Memoized>
// React.createElement(Memoized, null, React.createElement('span', null))
// Children: React Element {type: 'span', props: {}, ...} — yangi obyekt har render
```

Har render React Element yangi reference. Hatto JSX bir xil bo'lsa ham, `React.createElement` har gal yangi obyekt qaytaradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

5 ta bypass scenario va yechim:

```tsx
import { memo, useState, useMemo, useCallback, useContext, createContext } from 'react';
import type { ReactElement, ReactNode } from 'react';

// === Scenario 1: Inline literals ===
interface BadgeProps {
  config: { color: string };
  items: string[];
  onClick: () => void;
}

const Badge = memo(function Badge({ config, items, onClick }: BadgeProps): ReactElement {
  return (
    <div style={{ color: config.color }}>
      <button onClick={onClick}>{items.join(', ')}</button>
    </div>
  );
});

// ❌ Bypass har render
function ParentBypass1(): ReactElement {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <Badge
        config={{ color: 'red' }} // ❌ New
        items={['a', 'b']}        // ❌ New
        onClick={() => alert('hi')} // ❌ New
      />
    </>
  );
}

// ✅ Stable references
function ParentFix1(): ReactElement {
  const [count, setCount] = useState(0);
  
  const config = useMemo(() => ({ color: 'red' }), []);
  const items = useMemo(() => ['a', 'b'], []);
  const handleClick = useCallback(() => alert('hi'), []);
  
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <Badge config={config} items={items} onClick={handleClick} />
    </>
  );
}

// === Scenario 2: Context dep ===
const ThemeContext = createContext('light');

const Themed = memo(function Themed(): ReactElement {
  const theme = useContext(ThemeContext);
  return <div className={`theme-${theme}`}>Hello</div>;
});

function ParentBypass2(): ReactElement {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <button onClick={() => setTheme((t) => t === 'light' ? 'dark' : 'light')}>
        Toggle
      </button>
      <Themed /> {/* Theme o'zgarsa Themed re-render — memo bypass */}
    </ThemeContext.Provider>
  );
}
// Bu pattern memo bypass — istalgan natija (Context dep semantic).

// === Scenario 3: Children prop ===
const Wrapper = memo(function Wrapper({ children }: { children: ReactNode }): ReactElement {
  return <div className="wrapper">{children}</div>;
});

function ParentBypass3(): ReactElement {
  const [count, setCount] = useState(0);
  return (
    <Wrapper>
      <span>{count}</span> {/* Har count change'da yangi React Element */}
    </Wrapper>
  );
}
// Wrapper memo bypass — children har render yangi.

// === Scenario 4: Spread Props bilan unstable nested ===
interface SpreadConsumerProps {
  name: string;
  config: { lang: string };
  onTrigger: () => void;
}

const SpreadConsumer = memo(function SpreadConsumer(props: SpreadConsumerProps): ReactElement {
  return <div onClick={props.onTrigger}>{props.name} / {props.config.lang}</div>;
});

function ParentBypass4(): ReactElement {
  const [count, setCount] = useState(0);
  
  // ❌ Inner config/handler — har render yangi reference
  const allProps: SpreadConsumerProps = {
    name: 'Ali',           // primitive, OK
    config: { lang: 'uz' }, // ❌ inline object
    onTrigger: () => alert(count), // ❌ inline function
  };
  
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <SpreadConsumer {...allProps} />
    </>
  );
}
// React.memo har property'ni tekshiradi:
// - name primitive bir xil
// - config har render yangi reference → not equal
// - onTrigger har render yangi reference → not equal
// → bypass har count change'da

// ✅ Fix: stabilize nested references
function ParentFix4(): ReactElement {
  const [count, setCount] = useState(0);
  
  const config = useMemo(() => ({ lang: 'uz' }), []);
  const onTrigger = useCallback(() => alert(count), [count]); // count dep
  
  const allProps: SpreadConsumerProps = { name: 'Ali', config, onTrigger };
  
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <SpreadConsumer {...allProps} />
    </>
  );
}
// config stable. onTrigger count o'zgarganda yangi reference (kutilgan — alert latest count).

// === Scenario 5: Ref prop ===
import { useRef } from 'react';

const RefConsumer = memo(function RefConsumer({ value }: { value: number }): ReactElement {
  return <div>{value}</div>;
});

function ParentBypass5(): ReactElement {
  const ref1 = useRef<HTMLDivElement>(null);
  const ref2 = useRef<HTMLDivElement>(null);
  const [useFirst, setUseFirst] = useState(true);
  
  return (
    <>
      <button onClick={() => setUseFirst((v) => !v)}>Toggle</button>
      <RefConsumer ref={useFirst ? ref1 : ref2} value={42} />
    </>
  );
}
// useFirst toggle: ref reference o'zgaradi → memo bypass.
// Bu odatda bug emas — ref switch real semantic'ga aloqador.
```

</details>

---

## Reference Equality Gotchas

### Nazariya

JavaScript'da object/array/function reference equality (`Object.is`/`===`) reference identity'ga asoslanadi, **shape**'ga emas. Bu React'da re-render trigger'larida muhim:

```javascript
const a = { x: 1 };
const b = { x: 1 };

a === b;          // false — ikki alohida heap object
Object.is(a, b);  // false — reference farqli (shape bir xil bo'lsa ham)

const c = a;
c === a;          // true — bir xil reference
```

Bu fundamental — lekin React Element'larida ham ishlaydi:

```typescript
// JSX
<Child value={1} />

// JSX transform natijasi
React.createElement(Child, { value: 1 })

// Har createElement chaqiriqi yangi React Element obyekt qaytaradi
React.createElement(Child, { value: 1 }) === React.createElement(Child, { value: 1 })
// false
```

Inline object/array/function — har render yangi reference. Bu memo bypass va Hook deps gotcha'lariga olib keladi:

### Gotcha 1: Inline Object Props

```tsx
// ❌ Har render yangi config
<MemoChild config={{ color: 'red' }} />

// ✅ Stable
const config = useMemo(() => ({ color: 'red' }), []);
<MemoChild config={config} />
```

### Gotcha 2: Inline Array Props

```tsx
// ❌ Har render yangi array
<MemoChild items={[1, 2, 3]} />

// ✅ Stable (deps yo'q — bir marta)
const items = useMemo(() => [1, 2, 3], []);
<MemoChild items={items} />

// ✅ Module-level (eng oson)
const items = [1, 2, 3];
function Parent() {
  return <MemoChild items={items} />;
}
```

### Gotcha 3: Inline Function Props

```tsx
// ❌ Har render yangi function
<MemoChild onClick={() => doSomething()} />

// ✅ Stable
const handleClick = useCallback(() => doSomething(), []);
<MemoChild onClick={handleClick} />
```

### Gotcha 4: Hook Dep Array

```tsx
// ❌ user object har render yangi
const userInfo = { name: 'Ali', age: 25 };
useEffect(() => {
  fetch(`/api/user/${userInfo.name}`);
}, [userInfo]); // ❌ Har render dep o'zgaradi → effect har render

// ✅ Primitive deps
useEffect(() => {
  fetch(`/api/user/${userInfo.name}`);
}, [userInfo.name]); // Primitive — Object.is OK

// ✅ Yoki stable reference
const userInfo = useMemo(() => ({ name: 'Ali', age: 25 }), []);
useEffect(() => {
  fetch(`/api/user/${userInfo.name}`);
}, [userInfo]); // Stable reference
```

### Gotcha 5: Computed Property — Useless useMemo

```tsx
// ❌ Useless useMemo — primitive computation
const total = useMemo(() => count + 1, [count]);
// useMemo overhead'i computation'dan ko'p
```

```tsx
// ✅ Direct compute
const total = count + 1;
```

`useMemo`/`useCallback` faqat referential identity yoki expensive computation uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

JavaScript reference identity:

```
Heap memory:
   ┌─────────┐
   │ {x: 1}  │ ← address 0x100
   └─────────┘
   ┌─────────┐
   │ {x: 1}  │ ← address 0x200
   └─────────┘

Variables:
   const a = ref(0x100);
   const b = ref(0x200);
   
   a === b → 0x100 === 0x200 → false
```

`Object.is` object'lar uchun `===` bilan bir xil ishlaydi (reference identity solishtiradi). Farq faqat primitive'larda: `Object.is(NaN, NaN)` true (lekin `NaN === NaN` false), `Object.is(+0, -0)` false (lekin `+0 === -0` true). React reference solishtirish kerak bo'lgan joylarda `Object.is` ishlatadi: `useState` eager bailout, `React.memo` shallow check, `useMemo`/`useCallback` deps. Reconciler'ning props bailout check'i esa `!==` reference inequality ishlatadi.

JSX transform va React Element identity:

```javascript
// Source
<Child value={1} />

// Babel transform (R17+ automatic):
import { jsx as _jsx } from 'react/jsx-runtime';
_jsx(Child, { value: 1 })

// _jsx React Element obyektini quradi (oddiylashtirilgan, R19)
function jsx(type, props, key) {
  return {
    $$typeof: Symbol.for('react.transitional.element'),
    type,
    key,
    props, // R19: ref props ichida (alohida element field emas)
  };
}
// Har chaqiriq yangi obyekt qaytaradi
```

Har JSX expression `jsx()` (R17+ automatic runtime) yoki `React.createElement` chaqiriqiga aylanadi va yangi obyekt qaytaradi. Shu sababdan inline JSX `<Child />` har render yangi reference.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Reference vs value comparison:

```tsx
import { useEffect, useState } from 'react';
import type { ReactElement } from 'react';

function ReferenceDemo(): ReactElement {
  const [count, setCount] = useState(0);
  
  // Inline objects
  const obj1 = { x: 1 };
  const obj2 = { x: 1 };
  
  console.log('obj1 === obj2:', obj1 === obj2); // false
  console.log('Same shape, different reference');
  
  // Effect with object dep
  useEffect(() => {
    console.log('Effect ran');
  }, [obj1]); // ❌ Har render yangi obj1 → effect har render
  
  return <div>{count}</div>;
}
```

Stable reference patterns:

```tsx
import { useMemo, useCallback, useRef, useState } from 'react';
import type { ReactElement } from 'react';

// Pattern 1: useMemo for stable object
function Pattern1(): ReactElement {
  const config = useMemo(() => ({ apiUrl: 'https://api.example.com' }), []);
  return <Child config={config} />;
}

// Pattern 2: Module-level constant
const API_CONFIG = { apiUrl: 'https://api.example.com' };

function Pattern2(): ReactElement {
  return <Child config={API_CONFIG} />;
}

// Pattern 3: useRef for mutable container
function Pattern3(): ReactElement {
  const configRef = useRef({ apiUrl: 'https://api.example.com' });
  return <Child config={configRef.current} />;
}

// Pattern 4: useCallback for stable function
function Pattern4(): ReactElement {
  const [count, setCount] = useState(0);
  
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []);
  
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <MemoChild onClick={handleClick} />
    </>
  );
}

// Pattern 5: Functional update — closure-free
function Pattern5(): ReactElement {
  const [count, setCount] = useState(0);
  
  const increment = useCallback(() => {
    setCount((c) => c + 1); // Latest c — no count dep needed
  }, []); // No deps — handler stable forever
  
  return (
    <>
      <span>{count}</span>
      <MemoChild onClick={increment} />
    </>
  );
}
```

React Compiler — automatic stable reference:

```tsx
'use memo';

import { useState } from 'react';
import type { ReactElement } from 'react';

interface ChildProps {
  config: { apiUrl: string };
  onClick: () => void;
}

// Compiler memo equivalent
function Child({ config, onClick }: ChildProps): ReactElement {
  return (
    <div>
      <p>{config.apiUrl}</p>
      <button onClick={onClick}>Click</button>
    </div>
  );
}

function Parent(): ReactElement {
  const [count, setCount] = useState(0);
  
  // Compiler avtomatik:
  // - config object'ni memoize qiladi (deps yo'q — har gal bir xil)
  // - handleClick'ni memoize qiladi (deps yo'q)
  const config = { apiUrl: 'https://api.example.com' };
  const handleClick = () => console.log('Clicked');
  
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <Child config={config} onClick={handleClick} />
    </>
  );
}
// Compiler-generated kod: config va handleClick cache slot'da saqlanadi.
// Manual useMemo/useCallback yozish kerak emas.
```

</details>

---

## "New Element Same Type" — Update Path

### Nazariya

Reconciler diff algoritmining muhim qismi — **"new element same type"** detect qilish va **update path**'ga kirish (yangi mount qilmay, mavjud Fiber'ni qayta ishlatish va props yangilash).

Reconciliation 2 heuristic (cross-ref `04-reconciliation.md`):

1. **Different element types** — to'liq rebuild (eski subtree unmount, yangi mount).
2. **Same type, different props** — **update path** (Fiber saqlanadi, props yangilanadi).

NIMA UCHUN bu farq muhim:

- **State preservation** — same type → state saqlanadi (Hook chain Fiber'da).
- **DOM reuse** — same type → DOM node saqlanadi, faqat attribute'lar yangilanadi.
- **Effect lifecycle** — same type → useEffect cleanup → setup (deps o'zgargan bo'lsa), unmount yo'q.

```tsx
// Source
{condition ? <UserCard user={u} /> : <ProductCard product={p} />}

// condition true → false:
// - UserCard unmount (Fiber yo'qoladi, hook state, useEffect cleanup)
// - ProductCard mount (yangi Fiber, yangi state)
```

vs.

```tsx
// Source
<UserCard user={condition ? userA : userB} />

// condition true → false:
// - UserCard same type → update path
// - props.user reference yangi
// - Render qayta hisoblanadi
// - State saqlanadi
// - useEffect[user] dep o'zgargan bo'lsa cleanup → setup
```

Type identity rules:

- **String type (HTML element)** — `'div' === 'div'` (string equality).
- **Function/Class component** — referential identity (`UserCard === UserCard`).
- **`memo`-wrapped** — `MemoComponent` Fiber tag, internal `Component.type` reference.
- **`forwardRef`-wrapped** — `ForwardRef` Fiber tag (R19'da ref oddiy prop, lekin `forwardRef` hali ishlaydi).

Anonymous nested komponent — anti-pattern:

```tsx
// ❌ Anti-pattern: yangi function har Parent render'da
function Parent({ items }: { items: Item[] }) {
  function Item({ item }: { item: Item }) { // ❌ Har Parent render yangi reference
    return <li>{item.name}</li>;
  }
  
  return (
    <ul>
      {items.map((item) => (
        <Item key={item.id} item={item} />
      ))}
    </ul>
  );
}
// Har Parent render: yangi Item function (yangi reference)
// Reconciler: type !== prevType → unmount + mount → state lost har render
```

Yechim:

```tsx
// ✅ Komponent tashqarida (module-level)
function Item({ item }: { item: Item }) {
  return <li>{item.name}</li>;
}

function Parent({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((i) => (
        <Item key={i.id} item={i} />
      ))}
    </ul>
  );
}
```

Update path va state preservation interplay:

```tsx
// Form input — controlled
<input
  type="text"
  value={value}
  onChange={(e) => setValue(e.target.value)}
/>

// Re-render: input same type → update path
// DOM node saqlanadi → cursor position saqlanadi (yaxshi)
// React internal: only diff'd attributes yangilanadi
```

vs.

```tsx
// Type o'zgarsa — DOM node yangi
<input type={isPassword ? 'password' : 'text'} value={value} />

// type 'text' → 'password': different attribute, update path
// DOM node saqlanadi (input element type setable)

// vs

{isInput ? <input value={value} /> : <textarea value={value} />}

// input → textarea: different element type → unmount/mount
// DOM node almashtiriladi
// Cursor position lost
```

Key prop bilan force unmount/mount (cross-ref `08-list-rendering.md`):

```tsx
<UserCard key={userId} user={user} />

// userId o'zgarsa: same type, lekin key farqli → unmount + mount
// State reset, useEffect cleanup → setup
```

<details>
<summary><strong>Under the Hood</strong></summary>

Reconciliation type comparison:

```javascript
function updateElement(returnFiber, current, element, lanes) {
  if (current !== null) {
    if (current.elementType === element.type) {
      // Same type — update path
      const existing = useFiber(current, element.props);
      existing.ref = coerceRef(returnFiber, current, element);
      existing.return = returnFiber;
      return existing;
    }
  }
  
  // Different type or no current — create new
  const created = createFiberFromElement(element, returnFiber.mode, lanes);
  created.ref = coerceRef(returnFiber, current, element);
  created.return = returnFiber;
  return created;
}
```

`current.elementType === element.type` — bu **`===` strict equality** (`Object.is` emas). Function/class komponent uchun referential identity, HTML element uchun string equality (`'div' === 'div'`).

`useFiber` — Fiber'ning copy-on-write yangilash:

```javascript
function useFiber(fiber, pendingProps) {
  const clone = createWorkInProgress(fiber, pendingProps);
  clone.index = 0;
  clone.sibling = null;
  return clone;
}

function createWorkInProgress(current, pendingProps) {
  let workInProgress = current.alternate;
  
  if (workInProgress === null) {
    workInProgress = createFiber(...);
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    workInProgress.pendingProps = pendingProps;
    // ...
  }
  
  // Hook chain (memoizedState) saqlanadi
  workInProgress.memoizedState = current.memoizedState;
  
  return workInProgress;
}
```

`stateNode` (DOM node yoki class instance) saqlanadi — DOM node reuse.
`memoizedState` (Hook linked list) saqlanadi — state preserved.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Same type vs different type:

```tsx
import { useState, useEffect } from 'react';
import type { ReactElement } from 'react';

function UserCard({ user }: { user: { name: string } }): ReactElement {
  const [editing, setEditing] = useState(false);
  
  useEffect(() => {
    console.log('UserCard mounted/updated for', user.name);
    return () => {
      console.log('UserCard cleanup');
    };
  }, [user.name]);
  
  return (
    <div>
      <h2>{user.name}</h2>
      <button onClick={() => setEditing((e) => !e)}>
        {editing ? 'Save' : 'Edit'}
      </button>
    </div>
  );
}

function ProductCard({ product }: { product: { name: string } }): ReactElement {
  return <div>Product: {product.name}</div>;
}

// === Scenario 1: Same type, different props ===
function App1(): ReactElement {
  const [user, setUser] = useState({ name: 'Ali' });
  
  return (
    <>
      <button onClick={() => setUser({ name: 'Bob' })}>Switch user</button>
      <UserCard user={user} />
    </>
  );
}
// Switch: UserCard same type → update path.
// editing state saqlanadi, useEffect deps[user.name] o'zgargan → cleanup + setup.

// === Scenario 2: Different types ===
function App2(): ReactElement {
  const [showUser, setShowUser] = useState(true);
  
  return (
    <>
      <button onClick={() => setShowUser((v) => !v)}>Toggle</button>
      {showUser ? (
        <UserCard user={{ name: 'Ali' }} />
      ) : (
        <ProductCard product={{ name: 'Apple' }} />
      )}
    </>
  );
}
// Toggle: type o'zgaradi → unmount UserCard, mount ProductCard.
// editing state lost, useEffect cleanup chaqiriladi.

// === Scenario 3: Same type, same props (key bilan force unmount) ===
function App3(): ReactElement {
  const [userId, setUserId] = useState('1');
  
  return (
    <>
      <button onClick={() => setUserId((id) => (id === '1' ? '2' : '1'))}>
        Switch ID
      </button>
      <UserCard key={userId} user={{ name: `User ${userId}` }} />
    </>
  );
}
// Key change: same type, lekin key farq → unmount + mount.
// editing state reset (force unmount).
```

Anonymous nested anti-pattern:

```tsx
import { useState } from 'react';
import type { ReactElement } from 'react';

interface Item {
  id: string;
  name: string;
}

// ❌ Anti-pattern
function ItemListWithNestedComponent({ items }: { items: Item[] }): ReactElement {
  // ItemRow function har Parent render'da yangi reference
  function ItemRow({ item }: { item: Item }): ReactElement {
    const [hovered, setHovered] = useState(false);
    return (
      <li
        onMouseEnter={() => setHovered(true)}
        onMouseLeave={() => setHovered(false)}
      >
        {item.name} {hovered && '(hovered)'}
      </li>
    );
  }
  
  return (
    <ul>
      {items.map((i) => (
        <ItemRow key={i.id} item={i} />
      ))}
    </ul>
  );
}
// ItemListWithNestedComponent re-render: ItemRow function reference yangi.
// Har ItemRow: type !== prevType → unmount + mount.
// hovered state har render reset, mouse hover ko'rinmaydi.

// ✅ Komponent module-level
function ItemRow({ item }: { item: Item }): ReactElement {
  const [hovered, setHovered] = useState(false);
  return (
    <li
      onMouseEnter={() => setHovered(true)}
      onMouseLeave={() => setHovered(false)}
    >
      {item.name} {hovered && '(hovered)'}
    </li>
  );
}

function ItemListWithExtractedComponent({ items }: { items: Item[] }): ReactElement {
  return (
    <ul>
      {items.map((i) => (
        <ItemRow key={i.id} item={i} />
      ))}
    </ul>
  );
}
// ItemRow reference stable. Hover holati saqlanadi.
```

</details>

---

## Stale Closure Scenarios

### Nazariya

**Stale closure** — funksiya ichida yopilgan state/props reference komponentning **eski snapshot**'ida qoladi. Re-render bo'lsa — yangi snapshot yaratiladi, lekin agar funksiya **eski closure'ni saqlab tursa** (cross-ref `30-concurrent-react.md`), eski qiymatlar ishlatiladi.

Re-render behavior bilan bog'liq stale closure scenarios:

### Scenario 1: useEffect Empty Deps

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // ❌ count = 0 har gal (stale)
    }, 1000);
    return () => clearInterval(id);
  }, []); // Empty deps — closure once
  
  return <div>{count}</div>;
}
```

Yechim: functional update yoki ref pattern (cross-ref `30-concurrent-react.md`):

```tsx
useEffect(() => {
  const id = setInterval(() => {
    setCount((c) => c + 1); // ✅ Latest c
  }, 1000);
  return () => clearInterval(id);
}, []);
```

### Scenario 2: Event Handler Closure

```tsx
function ScrollSpy() {
  const [scrollY, setScrollY] = useState(0);
  
  useEffect(() => {
    function handleScroll() {
      console.log('Current scrollY:', scrollY); // ❌ Stale closure
      setScrollY(window.scrollY);
    }
    
    window.addEventListener('scroll', handleScroll);
    return () => window.removeEventListener('scroll', handleScroll);
  }, []); // ❌ scrollY closure'da
  
  return <div>Scroll: {scrollY}</div>;
}
```

Yechim: useRef latest pattern yoki deps to'g'ri:

```tsx
// useRef latest
const scrollYRef = useRef(scrollY);
useEffect(() => {
  scrollYRef.current = scrollY;
});

useEffect(() => {
  function handleScroll() {
    console.log('Current scrollY:', scrollYRef.current); // ✅ Latest
    setScrollY(window.scrollY);
  }
  window.addEventListener('scroll', handleScroll);
  return () => window.removeEventListener('scroll', handleScroll);
}, []);
```

### Scenario 3: Async Callback

```tsx
function SubmitForm() {
  const [data, setData] = useState({ email: '' });
  
  async function handleSubmit() {
    await fetch('/api/save', { method: 'POST', body: JSON.stringify(data) });
    
    // data — handler yaratilgan render'dagi snapshot (await'dan keyin ham o'sha).
    console.log('Saved:', data);
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        value={data.email}
        onChange={(e) => setData({ email: e.target.value })}
      />
      <button>Submit</button>
    </form>
  );
}
// handleSubmit submit bosilgan render'dagi `data`'ni closure'da ushlaydi.
// Agar user submit'dan keyin (await davom etayotganda) yana yozsa, yangi keystroke'lar
// re-render qiladi va yangi `data` yaratadi — lekin ishlab turgan handleSubmit eski
// snapshot'ni saqlaydi. Bu form submit uchun odatda kutilgan xulq (submit paytidagi
// qiymat). Latest qiymat kerak bo'lsa — useRef latest pattern (Scenario 2 yechimi).
```

### Scenario 4: useReducer Closure

```tsx
function Counter() {
  const [count, dispatch] = useReducer((s) => s + 1, 0);
  
  function handleClick() {
    dispatch();
  }
  
  // ✅ dispatch stable reference — closure issue yo'q
  return <button onClick={handleClick}>{count}</button>;
}
```

`useState` setter va `useReducer` dispatch — stable references. Stale closure muammo yo'q (cross-ref `12-state-and-usestate.md`).

NIMA UCHUN re-render bilan bog'liq:

- Re-render → yangi closure (yangi `count` snapshot).
- Lekin **avval saqlangan callback'lar** eski closure'ni ushlab qoladi (interval, listener, async).
- Re-render trigger bo'lsa ham, eski closure'lar yangilanmaydi (yangi setInterval qayta yaratilmasa).

<details>
<summary><strong>Under the Hood</strong></summary>

JavaScript closure capture:

```javascript
function Component() {
  const count = state.count; // 0
  
  useEffect(() => {
    // Bu closure mount paytida yaratiladi
    // count = 0 yopiladi
    const id = setInterval(() => {
      console.log(count); // count = 0 har doim
    }, 1000);
    return () => clearInterval(id);
  }, []);
}

// Re-render bo'lsa — Component qayta chaqiriladi.
// Yangi closure yaratiladi (count = 1 yopiladi).
// Lekin useEffect deps yo'q → effect qayta ishlamaydi.
// setInterval'dagi callback eski closure'da count = 0 ishlatadi.
```

`useRef latest pattern`:

```javascript
function Component() {
  const count = state.count;
  const ref = useRef(count); // ref.current = 0
  
  // Har render ref.current ni yangilash
  useEffect(() => {
    ref.current = count;
  });
  
  useEffect(() => {
    // Closure ref reference'ni yopadi (mutable container)
    // ref.current har chaqiriqda latest
    const id = setInterval(() => {
      console.log(ref.current); // Latest count
    }, 1000);
    return () => clearInterval(id);
  }, []);
}
```

`useRef` mutable container — `ref.current` o'qish/yozish render'siz, closure'da reference yopiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

3 ta yechim taqqoslash:

```tsx
import { useState, useEffect, useRef } from 'react';
import type { ReactElement } from 'react';

// === Solution 1: Functional update ===
function CounterFn(): ReactElement {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      setCount((c) => c + 1); // ✅ Latest c
    }, 1000);
    return () => clearInterval(id);
  }, []);
  
  return <div>{count}</div>;
}
// Eng oddiy — setCount o'z latest state'ini olishi yetarli.

// === Solution 2: useRef latest pattern ===
function CounterRef(): ReactElement {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);
  
  useEffect(() => {
    countRef.current = count;
  });
  
  useEffect(() => {
    const id = setInterval(() => {
      console.log('Current:', countRef.current);
      setCount((c) => c + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);
  
  return <div>{count}</div>;
}
// Foydali: closure ichida latest state'ni o'qish kerak bo'lganda.

// === Solution 3: Deps to'g'ri ===
function CounterDeps(): ReactElement {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      console.log('Current:', count); // ✅ Latest count (closure rebuilt)
      setCount((c) => c + 1);
    }, 1000);
    return () => clearInterval(id);
  }, [count]);
  
  return <div>{count}</div>;
}
// Performance: har count change'da interval cleared + recreated.
// Eng kam afzal — overhead.
```

Re-render bilan stale closure interplay:

```tsx
import { useState, useEffect } from 'react';
import type { ReactElement } from 'react';

interface Notification {
  id: string;
  text: string;
}

function NotificationFeed(): ReactElement {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  
  useEffect(() => {
    const ws = new WebSocket('wss://...');
    
    ws.onmessage = (e) => {
      const newNotif = JSON.parse(e.data) as Notification;
      
      // ❌ Stale: notifications mount vaqtidagi snapshot
      // setNotifications([...notifications, newNotif]);
      
      // ✅ Functional update — latest
      setNotifications((prev) => [...prev, newNotif]);
    };
    
    return () => ws.close();
  }, []);
  
  return (
    <ul>
      {notifications.map((n) => (
        <li key={n.id}>{n.text}</li>
      ))}
    </ul>
  );
}
```

useEffectEvent (RFC, experimental — R19 canary/experimental build'larda):

```tsx
// Hozirgi pattern
const onMessage = useCallback((data: Notification) => {
  setNotifications((prev) => [...prev, data]);
  // user latest qiymatini ham ishlatish kerak bo'lsa — stale closure
}, []);

// R19 RFC (kelajakda)
import { experimental_useEffectEvent as useEffectEvent } from 'react';

function Component() {
  const [user, setUser] = useState(null);
  const [notifications, setNotifications] = useState([]);
  
  // useEffectEvent: non-reactive, latest closure
  const handleMessage = useEffectEvent((data: Notification) => {
    setNotifications((prev) => [...prev, data]);
    if (user) console.log(user.id, data); // Latest user
  });
  
  useEffect(() => {
    const ws = new WebSocket('wss://...');
    ws.onmessage = (e) => handleMessage(JSON.parse(e.data));
    return () => ws.close();
  }, []); // No deps needed — handleMessage stable
}
// Eslatma: useEffectEvent R19'da hali stable emas (RFC stage).
// Production: useRef latest pattern.
```

</details>

---

## DevTools "Why did this render?"

### Nazariya

**React DevTools Profiler** — re-render trigger sabablarini topish uchun asosiy debugging vositasi. Production'da React DevTools browser extension orqali ishlatiladi (Chrome, Firefox, Edge).

DevTools 2 ta tab beradi:

1. **Components tab** — tree, props, state, hooks ko'rish.
2. **Profiler tab** — record va analyze re-renders.

Profiler workflow (cross-ref `34-profiling.md` chuqur):

1. **Record start** — Profiler tab'da circle button.
2. Komponent bilan ishlash (click, type, navigate).
3. **Record stop**.
4. **Flame chart** — har commit duration ko'rsatiladi.
5. Bir komponentni tanlash → "Why did this render?" panel.

"Why did this render?" panel — re-render sabablari ro'yxati:

| Reason | Tasvirlash |
|--------|------------|
| The first time the component rendered | Initial mount |
| Hooks changed | useState/useReducer state change |
| Props changed: (props list) | Parent berdi yangi props |
| Parent component rendered | Top-down propagation, props o'zgarmagan |
| Context changed | useContext value change |

Bu informatsiya **debug birinchi qadami** — re-render manbai qaysi.

Settings (Profiler tab oxirida):

- **Highlight updates when components render** — har render bo'lgan komponent atrofida yashil rectangle (visual feedback).
- **Record why each component rendered while profiling** — har commit'dagi sabablarni saqlash (overhead, faqat debug uchun).

NIMA UCHUN bu vosita kritik:

- **Performance bottleneck topish** — qaysi komponent ko'p re-render bo'ladi.
- **Bypass diagnose** — `React.memo` ishlamayapti? Profiler "Props changed: onClick" deb ko'rsatadi.
- **Context propagation tracking** — Context value o'zgarsa qaysi consumers re-render.

Production profile:

- Development build — `actualDuration` real, lekin DEV warning'lar overhead.
- Production build — `react-dom/profiling` (cross-ref `34-profiling.md`).

> **Eslatma:** "Why did this render?" specific React DevTools'ning Profiler functionality'i — DevTools versiyasiga qarab UI farq qilishi mumkin. Latest DevTools install qilish tavsiya etiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

React DevTools API:

```javascript
// React internal — devtools hook
window.__REACT_DEVTOOLS_GLOBAL_HOOK__.onCommitFiberRoot(rendererID, root);

// DevTools Profiler — onRender callback
import { Profiler } from 'react';

function onRenderCallback(
  id,           // Profiler "id" prop
  phase,        // 'mount' | 'update'
  actualDuration, // Time spent rendering in this commit
  baseDuration,   // Estimated time without memoization
  startTime,
  commitTime
) {
  // Send to monitoring service
}

<Profiler id="App" onRender={onRenderCallback}>
  <App />
</Profiler>
```

`actualDuration` — bailout'lar tashqarida real ish vaqti. Bailout bo'lsa, komponent uchun 0.

`baseDuration` — agar barcha komponent'lar re-render bo'lsa, taxminiy vaqt. Memoization metrik:

```
Memoization efficiency = (baseDuration - actualDuration) / baseDuration

Misol:
- baseDuration = 50ms
- actualDuration = 5ms (90% bailout)
- Efficiency = 0.9 (90%)
```

"Why did this render?" sabablarni topishda DevTools `Fiber.alternate` va `Fiber.memoizedProps`'ni solishtiradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`<Profiler>` component (programmatic):

```tsx
import { Profiler, useState } from 'react';
import type { ReactElement, ProfilerOnRenderCallback } from 'react';

const onRender: ProfilerOnRenderCallback = (
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime
) => {
  console.log(`[${id}] ${phase}:`, {
    actualDuration: actualDuration.toFixed(2),
    baseDuration: baseDuration.toFixed(2),
    efficiency: (((baseDuration - actualDuration) / baseDuration) * 100).toFixed(0) + '%',
  });
};

function App(): ReactElement {
  const [count, setCount] = useState(0);
  
  return (
    <Profiler id="App" onRender={onRender}>
      <button onClick={() => setCount((c) => c + 1)}>Count: {count}</button>
      <ExpensiveList />
    </Profiler>
  );
}
```

DevTools ishlatish workflow:

```typescript
// 1. Browser extension install: React DevTools
// 2. Application'ni ochish
// 3. F12 → React DevTools tab
// 4. "Profiler" sub-tab
// 5. Settings → "Record why each component rendered while profiling" enable
// 6. Record button (circle)
// 7. Action perform (button click, type, etc.)
// 8. Stop record
// 9. Flame chart — har commit
// 10. Komponent tanlash → "Why did this render?" panel
```

Production profile (cross-ref `34-profiling.md`):

```typescript
// vite.config.ts — production profiling
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      // Production build'da profiling instrumentation'ni yoqadi.
      'react-dom$': 'react-dom/profiling',
    },
  },
});
```

</details>

---

## Highlight Updates va Profile Workflow

### Nazariya

**Highlight Updates** — DevTools'ning visual debug feature. Har render bo'lgan komponent DOM'ida bir lahzaga yashil rectangle ko'rinadi. Bu re-render qachon va qaerda sodir bo'layotganini ko'rsatadi visual feedback bilan.

Settings yoqish:

```
1. F12 → React DevTools
2. Settings (gear icon)
3. "Highlight updates when components render" — checkbox enable
4. Save
```

Endi har render visual rectangle ko'rinadi. Rangi render frequency'ga qarab (vaqt oynasi ichida ko'p marta render bo'lsa, issiqroq rang):

- **Ko'k / Yashil** — kam frequency (normal, kutilgan re-render)
- **Sariq** — o'rta frequency (e'tibor talab)
- **Qizil** — yuqori frequency (performance issue — tez ketma-ket re-render, optimization kerak)

Aniq color mapping DevTools versiyasiga qarab biroz farq qiladi — qizil rectangle har doim diqqat manbai.

NIMA UCHUN foydali:

- **Visual debug** — re-render manzili tezlik bilan ko'rinadi.
- **Unexpected re-render** — sodir bo'ladigan komponent qaysi ekanini topish.
- **Performance hotspot** — qizil rectangle = optimization kerak.

Profile workflow (production debugging):

### Step 1: Suspected Bottleneck'ni topish

```typescript
// User reports: "Search box laggy"
// Hypothesis: SearchBox tez-tez re-render bo'lyapti
```

### Step 2: Highlight Updates yoqish

User'ga repro stepsni bajartirib, qaysi komponent qizil chiziq oladi.

### Step 3: Profiler Record

- Record start
- Repro action perform
- Record stop
- Flame chart inspect

### Step 4: "Why did this render?" Analyze

Komponent ni tanlash, har commit uchun reason:

```
Commit #1 (mount): The first time the component rendered
Commit #2: Hooks changed (useState 'query')
Commit #3: Props changed: onResults
Commit #4: Parent component rendered
```

### Step 5: Optimization Strategy

| Reason | Strategy |
|--------|----------|
| Hooks changed | Bu kutilgan — state update'ga response |
| Props changed: onClick | `useCallback` parent'da |
| Props changed: data | `useMemo` parent'da yoki Compiler enable |
| Parent component rendered | `React.memo` yoki Compiler |
| Context changed | Context split, selector pattern |

### Step 6: Re-profile

Optimization'dan keyin profile qayta — improvement tasdiqlash.

### Step 7: Real-User Monitoring (production)

`<Profiler>` callback orqali monitoring service'ga (Datadog, Sentry, custom):

```tsx
import { Profiler } from 'react';

const onRender = (id, phase, actualDuration, baseDuration) => {
  // Send to monitoring
  monitoring.recordReactRender({ id, phase, actualDuration, baseDuration });
};

<Profiler id="SearchBox" onRender={onRender}>
  <SearchBox />
</Profiler>
```

<details>
<summary><strong>Under the Hood</strong></summary>

React DevTools — Highlight Updates implementation:

```javascript
// DevTools internal
function highlightUpdates(node, color) {
  const rect = node.getBoundingClientRect();
  const overlay = document.createElement('div');
  overlay.style.position = 'fixed';
  overlay.style.top = rect.top + 'px';
  overlay.style.left = rect.left + 'px';
  overlay.style.width = rect.width + 'px';
  overlay.style.height = rect.height + 'px';
  overlay.style.border = `2px solid ${color}`;
  overlay.style.pointerEvents = 'none';
  
  document.body.appendChild(overlay);
  
  setTimeout(() => {
    document.body.removeChild(overlay);
  }, 100);
}

// Color logic based on render frequency
function getHighlightColor(renderCount) {
  if (renderCount === 1) return 'green';
  if (renderCount === 2) return 'yellow';
  return 'red';
}
```

`onCommitFiberRoot` hook DevTools'da har commit'da chaqiriladi va `Fiber.actualDuration`, `lanes` info'ni o'qib, visual update emit qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Production monitoring integration:

```tsx
import { Profiler } from 'react';
import type { ReactElement, ProfilerOnRenderCallback } from 'react';

// Monitoring service (e.g., Datadog, Sentry)
interface PerformanceMetric {
  componentId: string;
  phase: 'mount' | 'update' | 'nested-update';
  actualDuration: number;
  baseDuration: number;
  timestamp: number;
}

const metrics: PerformanceMetric[] = [];

const onRender: ProfilerOnRenderCallback = (
  id,
  phase,
  actualDuration,
  baseDuration
) => {
  metrics.push({
    componentId: id,
    phase: phase as PerformanceMetric['phase'],
    actualDuration,
    baseDuration,
    timestamp: Date.now(),
  });
  
  // Slow render warning
  if (actualDuration > 16) {
    console.warn(`[Slow render] ${id}: ${actualDuration.toFixed(2)}ms`);
  }
};

// Periodic flush to monitoring
setInterval(() => {
  if (metrics.length > 0) {
    fetch('/api/monitoring/react', {
      method: 'POST',
      body: JSON.stringify(metrics),
    });
    metrics.length = 0;
  }
}, 10000);

function App(): ReactElement {
  return (
    <Profiler id="App" onRender={onRender}>
      <Profiler id="Header" onRender={onRender}>
        <Header />
      </Profiler>
      <Profiler id="MainContent" onRender={onRender}>
        <MainContent />
      </Profiler>
      <Profiler id="Sidebar" onRender={onRender}>
        <Sidebar />
      </Profiler>
    </Profiler>
  );
}
// Production'da nested Profiler'lar — granular monitoring.
// Performance budget: actualDuration > 16ms (1 frame) — warning.
```

Real-world debug session example:

```
Symptom: SearchBox typing laggy

Hypothesis: SearchResults re-rendering on every keystroke

Step 1: Highlight Updates ON
- Type in search box
- SearchResults: red rectangle (frequent re-render)
- ResultItem: yellow (parent triggered, o'rta frequency)

Step 2: Profiler Record
- Type 5 chars in search box
- Stop record

Step 3: Flame chart
- 5 commits, each ~25ms (long!)
- SearchResults dominates

Step 4: "Why did this render?"
- SearchResults: "Hooks changed (useState 'results')"
- ResultItem (1000+): "Parent component rendered"

Step 5: Optimization
- Wrap ResultItem in React.memo
- useCallback for onSelect
- useMemo for filtered results

Step 6: Re-profile
- Same actions, commits ~5ms (improved 5x)
- ResultItem: green (no re-render unless item changes)

Step 7: Production monitoring
- <Profiler> wrapping SearchBox
- Track p95 actualDuration
- Alert if > 50ms regression
```

</details>

---

## Compiler Era Re-render Behavior

### Nazariya

React Compiler (`'use memo'` directive, cross-ref `31-react-compiler.md`) re-render behavior'ni o'zgartiradi:

### Avval (Manual Memoization)

- Default — `<Component />` har render'da Component funksiyasi chaqiriladi.
- `React.memo` — props bir xil bo'lsa skip.
- `useMemo`/`useCallback` — referential identity stabilize.
- Manual qaror — har komponent uchun.

### Compiler Era

- Default `'use memo'` — Compiler komponent darajasidagi memo equivalent qo'shadi.
- Avtomatik — JSX cache, computation cache, function reference cache.
- Granular — har reactive scope alohida cache slot.
- Manual `React.memo`/`useMemo`/`useCallback` ortiqcha (ishlaydi, lekin Compiler bypass qilmaydi).

Re-render trigger'lar Compiler'da:

| Trigger | Compiler Era Behavior |
|---------|----------------------|
| State change | Render trigger (default kabi) |
| Parent re-render | **Automatic memo** — props bir xil bo'lsa render skip |
| Context change | Render trigger (default kabi, memo bypass) |
| Force update | Render trigger |

Bypass scenarios Compiler era:

| Scenario | Compiler vs Manual |
|----------|---------------------|
| Inline object prop | Compiler: avtomatik cache. Manual: `useMemo` kerak. |
| Inline function prop | Compiler: avtomatik cache. Manual: `useCallback` kerak. |
| Context dep | Compiler: hali ham bypass (semantic). Manual: bir xil. |
| Children prop | Compiler: cache, lekin children o'zi yangi reference bo'lsa — bypass. Manual: bir xil. |
| Ref prop | Compiler: memo bypass. Manual: bir xil. |

**Compiler'ning re-render impact** — production'da:

- Komponent re-render sezilarli kamayishi (Meta React Conf 2024 keynote'da Instagram web va Quest store production data — aniq qiymatlar workload va komponent strukturasiga qarab farq qiladi).
- Granularity yaxshilanishi — har property level cache (manual `useMemo` bilan erishish qiyin bo'lgan precision).
- Code clarity — manual memoization wrapper'lar olib tashlanadi.

Cheklov — Compiler bail-out'da behavior manual'ga qaytadi:

```tsx
'use memo';

function BailedComponent({ items }: { items: Item[] }) {
  items.push(extraItem); // ❌ Mutation — Compiler bail-out
  return <ul>...</ul>;
}
// Compiler: bu komponentni memoize qilmaydi.
// Behavior: avval kabi (manual yoki memo'siz).
```

> **Migration:** Compiler enable qilingan loyihada manual `useMemo`/`useCallback`/`React.memo` aksariyat olib tashlanishi mumkin (cross-ref `31-react-compiler.md` Migration Path 6 Qadam).

<details>
<summary><strong>Under the Hood</strong></summary>

Compiler-generated memo equivalent:

```javascript
// Source
'use memo';

function ProductCard({ product }) {
  return <div>{product.name}</div>;
}

// Compiler output (oddiylashtirilgan)
import { c as _c } from 'react/compiler-runtime';

function ProductCard(t0) {
  const $ = _c(2);
  const { product } = t0;
  
  let t1;
  if ($[0] !== product) {
    t1 = <div>{product.name}</div>;
    $[0] = product;
    $[1] = t1;
  } else {
    t1 = $[1];
  }
  return t1;
}
```

Bu output `React.memo` bilan ekvivalent, lekin:
- Komponent darajasidagi memo (parent re-render bypass).
- Property-level granularity (Compiler product.name'gacha track qiladi mumkin).

Kombinatsiya — parent ham `'use memo'`, child ham:

```javascript
// Parent re-render bo'lsa:
// - Parent'ning JSX outputi cache, child component reference saqlanadi.
// - Child'ning input (props) bir xil → cache hit.
// - Child component funksiyasi chaqirilmaydi.
// - Re-render zanjirida bailout.
```

Compiler — Reconciler bailout'ni **explicit code generation** orqali ta'minlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Compiler era — re-render kuzatish:

```tsx
'use memo';

import { useState, useRef, useEffect } from 'react';
import type { ReactElement } from 'react';

function useRenderCount(name: string): number {
  const ref = useRef(0);
  useEffect(() => {
    ref.current += 1;
    console.log(`[${name}] render #${ref.current}`);
  });
  return ref.current;
}

interface Item {
  id: string;
  name: string;
}

function ItemRow({ item }: { item: Item }): ReactElement {
  useRenderCount(`ItemRow-${item.id}`);
  return <li>{item.name}</li>;
}

function ItemList({ items }: { items: Item[] }): ReactElement {
  useRenderCount('ItemList');
  return (
    <ul>
      {items.map((i) => (
        <ItemRow key={i.id} item={i} />
      ))}
    </ul>
  );
}

function App(): ReactElement {
  useRenderCount('App');
  const [count, setCount] = useState(0);
  
  // Compiler avtomatik items'ni cache qiladi
  const items: Item[] = [
    { id: '1', name: 'Apple' },
    { id: '2', name: 'Banana' },
  ];
  
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>Count: {count}</button>
      <ItemList items={items} />
    </>
  );
}
// Click har count++:
//   [App] render
//   ItemList: bailout (items reference bir xil — Compiler cache)
//   ItemRow-*: bailout (parent ItemList chaqirilmadi)

// Manual versiya bilan solishtirish:
// const items = useMemo(() => [...], []);
// const ItemList = memo(...);
// const ItemRow = memo(...);
// Bir xil natija, lekin manual code 30+ qator memoization wrapper.
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: Strict Mode 2x Render — Performance Profile Misleading

Development'da `<StrictMode>` har render 2x chaqiradi (cross-ref `30-concurrent-react.md`). Profile actualDuration bu sababdan production'dan 2x katta bo'ladi.

**Yechim:** Production build (`react-dom/profiling`) bilan profile, yoki `<StrictMode>` o'chirib (faqat profile vaqtida — production'da hech qachon).

### Gotcha 2: `<Fragment>` re-render scope

Fragment Fiber `Fragment` tag bilan yaratiladi, lekin `stateNode` (DOM) yo'q — children DOM tree'da grandparent'ning bevosita children sifatida ko'rinadi:

```tsx
<>
  <ChildA />
  <ChildB />
</>
// vs
<div>
  <ChildA />
  <ChildB />
</div>
// Reconciler: ikkalasi ham bir xil — ChildA va ChildB diff har holatda.
// Lekin <div> Fiber yaratiladi (HostComponent), Fragment Fiber memo'ga affect qilmaydi.
```

### Gotcha 3: `useState` initializer Strict Mode 2x

```tsx
const [data] = useState(() => loadData());
// Strict Mode'da loadData() 2x chaqiriladi (purity check).
// Side effect bo'lmasligi shart (cross-ref 30 Render Purity).
```

### Gotcha 4: `key` change — Force Unmount

```tsx
<UserCard key={userId} user={user} />
// userId o'zgarsa: same type, lekin key farq → unmount + mount.
// Hooks state lost, useEffect cleanup chaqiriladi.
// Bu istalgan natija (key trick — state reset uchun).
```

### Gotcha 5: `Suspense` fallback re-render

Suspense `fallback` ko'rsatilganda, primary children komponent'lari Fiber tree'dan o'chirilmaydi — ular hidden (offscreen) holatda saqlanadi (R18+). State saqlanadi, lekin effect'lar (layout va passive) vaqtincha cleanup qilinadi:

```tsx
<Suspense fallback={<Loading />}>
  <DataView /> {/* Throw promise — fallback ko'rsatiladi */}
</Suspense>
// DataView Fiber tree'da qoladi (unmount emas) — Hook state (useState) saqlanadi.
// useEffect/useLayoutEffect cleanup chaqiriladi (effect'lar vaqtincha uziladi, R18+).
// Promise resolve bo'lsa — DataView qayta ko'rinadi, effect'lar qayta setup bo'ladi.
// State o'zgarmaydi (to'liq remount emas — faqat effect lifecycle qayta tiklanadi).
```

---

## Common Mistakes

### ❌ Xato 1: `React.memo` Inline Props bilan

```tsx
const Child = memo(function Child({ data }) { return <div>...</div>; });

<Child data={{ x: 1 }} /> // ❌ Bypass
```

```tsx
const data = useMemo(() => ({ x: 1 }), []);
<Child data={data} /> // ✅
```

### ❌ Xato 2: useEffect Empty Deps + Stale State

```tsx
useEffect(() => {
  setInterval(() => setCount(count + 1), 1000); // ❌ Stale count
}, []);
```

```tsx
useEffect(() => {
  setInterval(() => setCount((c) => c + 1), 1000); // ✅ Functional
}, []);
```

### ❌ Xato 3: Anonymous Nested Component

```tsx
function Parent({ items }) {
  function Item({ item }) { return <li>...</li>; } // ❌ Yangi reference har render
  return <ul>{items.map(i => <Item item={i} />)}</ul>;
}
```

```tsx
function Item({ item }) { return <li>...</li>; } // ✅ Module-level

function Parent({ items }) {
  return <ul>{items.map(i => <Item item={i} />)}</ul>;
}
```

### ❌ Xato 4: Context Value Yangi Object Har Render

```tsx
<UserContext.Provider value={{ name, age }}> {/* ❌ */}
```

```tsx
const value = useMemo(() => ({ name, age }), [name, age]);
<UserContext.Provider value={value}> {/* ✅ */}
```

### ❌ Xato 5: useMemo Primitive Computation

```tsx
const total = useMemo(() => count + 1, [count]); // ❌ Useless
```

```tsx
const total = count + 1; // ✅ Direct
```

---

## Amaliy Mashqlar

### Mashq 1: Re-render Trigger Aniqlash (Oson)

Quyidagi komponent'da nima sodir bo'ladi `setCount(c => c + 1)` chaqirilganda? Qaysi komponentlar re-render bo'ladi?

```tsx
function App() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <Header />
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Footer />
    </div>
  );
}

const Header = memo(function Header() {
  return <h1>App</h1>;
});

function Footer() {
  return <footer>© 2026</footer>;
}
```

<details>
<summary><strong>Javob</strong></summary>

`setCount(c => c + 1)` chaqirilganda:

1. **App** re-render — state o'zgardi (count).
2. **Header** — `React.memo` props bo'sh (prevProps va nextProps ikkalasi `{}` empty object, shallow equal true) → `bailoutOnAlreadyFinishedWork` → **Header funksiyasi umuman chaqirilmaydi**.
3. **Footer** — top-down propagation, memo'siz → Footer funksiyasi **chaqiriladi** (re-render). JSX `<footer>© 2026</footer>` har gal bir xil natija — Reconciler diff DOM mutation generate qilmaydi (bu bailout EMAS, "empty diff" stsenariy: function CPU sarflanadi, faqat DOM kafolat qilinmaydi).
4. **button** — App'ning bevosita JSX'i ichida, alohida komponent emas. App re-render'ida button element qayta yaratiladi va `{count}` text content o'zgargani sababli Reconciler `commitTextUpdate` chaqiradi → DOM text node update.

Optimization (agar Footer expensive bo'lsa):

```tsx
const Footer = memo(function Footer() {
  return <footer>© 2026</footer>;
});
// Endi Footer ham bailout.
```

React Compiler: agar `'use memo'` enabled bo'lsa — Footer avtomatik memo, manual kerak emas.

</details>

### Mashq 2: Context Value Memoization (Oson)

Quyidagi `<CartProvider>` har Provider re-render'da barcha consumer'larni re-render qiladi. Tuzating.

```tsx
import { createContext, useState } from 'react';
import type { ReactNode } from 'react';

interface CartContextValue {
  items: string[];
  addItem: (item: string) => void;
}

const CartContext = createContext<CartContextValue | null>(null);

export function CartProvider({ children }: { children: ReactNode }) {
  const [items, setItems] = useState<string[]>([]);
  
  function addItem(item: string) {
    setItems((prev) => [...prev, item]);
  }
  
  return (
    <CartContext.Provider value={{ items, addItem }}>
      {children}
    </CartContext.Provider>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

Muammo: `value={{ items, addItem }}` har render yangi object reference. Hatto items o'zgarmagan bo'lsa ham, har Provider render'da consumer'lar re-render bo'ladi.

Yechim — `useMemo` + `useCallback`:

```tsx
import { createContext, useState, useMemo, useCallback } from 'react';
import type { ReactNode, ReactElement } from 'react';

interface CartContextValue {
  items: string[];
  addItem: (item: string) => void;
}

const CartContext = createContext<CartContextValue | null>(null);

export function CartProvider({ children }: { children: ReactNode }): ReactElement {
  const [items, setItems] = useState<string[]>([]);
  
  // ✅ Stable callback
  const addItem = useCallback((item: string) => {
    setItems((prev) => [...prev, item]);
  }, []);
  
  // ✅ Memoized value
  const value = useMemo<CartContextValue>(
    () => ({ items, addItem }),
    [items, addItem]
  );
  
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}
```

Endi:
- `addItem` stable reference (deps yo'q, setItems ham stable).
- `value` faqat items o'zgarsa yangi reference.
- Consumer'lar items o'zgarmasa re-render bo'lmaydi.

React Compiler: `'use memo'` directive bilan barcha avtomatik memoize qilinadi (manual useMemo/useCallback shart emas).

Yana yaxshiroq pattern — Context split (state vs dispatch):

```tsx
const CartContext = createContext<string[] | null>(null);
const CartDispatchContext = createContext<((item: string) => void) | null>(null);

export function CartProvider({ children }: { children: ReactNode }): ReactElement {
  const [items, setItems] = useState<string[]>([]);
  
  const addItem = useCallback((item: string) => {
    setItems((prev) => [...prev, item]);
  }, []);
  
  return (
    <CartContext.Provider value={items}>
      <CartDispatchContext.Provider value={addItem}>
        {children}
      </CartDispatchContext.Provider>
    </CartContext.Provider>
  );
}

// useCart — items'ga subscribe
// useCartDispatch — addItem'ga subscribe (har doim stable)
```

Komponent faqat dispatch ishlatsa, items o'zgarganda re-render bo'lmaydi.

</details>

### Mashq 3: React.memo Bypass Aniqlash (O'rta)

Quyidagi 4 ta scenario'dan har biri `<Memoized />`'ni re-render qildiradimi yoki yo'q? Sababini aytib bering.

```tsx
const Memoized = memo(function Memoized({ data, onClick }: Props) {
  return <button onClick={onClick}>{data.label}</button>;
});

// Scenario A
function ParentA() {
  const [count, setCount] = useState(0);
  const data = { label: 'Click me' };
  const onClick = () => alert('hi');
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Memoized data={data} onClick={onClick} />
    </>
  );
}

// Scenario B
const STATIC_DATA = { label: 'Click me' };
function ParentB() {
  const [count, setCount] = useState(0);
  const onClick = useCallback(() => alert('hi'), []);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Memoized data={STATIC_DATA} onClick={onClick} />
    </>
  );
}

// Scenario C
function ParentC() {
  const [count, setCount] = useState(0);
  const data = useMemo(() => ({ label: 'Click me' }), []);
  const onClick = useCallback(() => alert('hi'), []);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Memoized data={data} onClick={onClick} />
    </>
  );
}

// Scenario D
function ParentD() {
  const [count, setCount] = useState(0);
  const onClick = useCallback(() => alert(count), [count]); // count deps
  const data = useMemo(() => ({ label: 'Click me' }), []);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Memoized data={data} onClick={onClick} />
    </>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

**Scenario A** — ❌ Bypass har render
- `data` har render yangi object reference
- `onClick` har render yangi function reference
- Both fail shallow comparison

**Scenario B** — ✅ Bailout
- `STATIC_DATA` module-level (stable)
- `onClick` useCallback (stable, deps yo'q)
- All props bir xil → memo bailout

**Scenario C** — ✅ Bailout
- `data` useMemo (stable, deps yo'q)
- `onClick` useCallback (stable)
- All props bir xil → memo bailout

**Scenario D** — ❌ Bypass har count change'da
- `data` stable (useMemo deps yo'q)
- `onClick` useCallback deps `[count]` → count o'zgarganda yangi reference
- onClick prop fail comparison

Scenario D fix — ikki yondashuv:

```tsx
// ✅ Yechim 1: useRef latest pattern (cross-ref 30 Stale Closure)
const countRef = useRef(count);
useEffect(() => { countRef.current = count; });
const onClick = useCallback(() => alert(countRef.current), []);
// Tavsiya etiladi: stable callback + latest value via ref
```

```tsx
// ⚠️ Anti-pattern: setState updater ichida side effect
const onClick = useCallback(() => {
  setCount((c) => {
    alert(c); // ❌ Updater pure bo'lishi shart (cross-ref 30 Anti-pattern 2)
    return c;
  });
}, []);
// React docs: "An updater should only return next state, not cause side effects."
// Strict Mode 2x cycle bunda alert ikki marta chiqadi.
```

```tsx
// ✅ Yechim 2: Compiler era — useCallback yo'q
'use memo';
const onClick = () => alert(count);
// Compiler avtomatik granular cache qiladi.
```

React Compiler: avtomatik tracking — `count` to'g'ridan-to'g'ri ishlatish OK, Compiler granular cache qiladi.

</details>

### Mashq 4: Stale Closure Debug (O'rta)

Quyidagi `useDebounce` hook to'g'ri ishlamayapti — `value` updates'lar e'tiborsiz qoladi. Sababini topib, tuzating.

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);
  
  useEffect(() => {
    setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
  }, []); // ❌
  
  return debouncedValue;
}
```

<details>
<summary><strong>Javob</strong></summary>

Muammolar:

1. **Empty deps `[]`** — `value` o'zgarsa effect qayta ishlamaydi → `setTimeout` bir marta yaratiladi va birinchi value'ni saqlaydi.
2. **Cleanup yo'q** — Strict Mode 2x cycle'da yoki value tez-tez o'zgarsa, ko'p timer parallel ishlaydi.
3. **Stale closure** — setTimeout callback `value` mount paytidagi snapshot.

Tuzatilgan:

```tsx
import { useState, useEffect } from 'react';

function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);
  
  useEffect(() => {
    const id = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
    
    return () => {
      clearTimeout(id); // ✅ Cleanup
    };
  }, [value, delay]); // ✅ Deps to'g'ri
  
  return debouncedValue;
}

// Lifecycle:
// 1. Mount: debouncedValue = value
// 2. value o'zgaradi → useEffect qayta ishlaydi
// 3. Avvalgi timer cleanup (clearTimeout)
// 4. Yangi setTimeout
// 5. delay ms keyin: setDebouncedValue(value)

// Strict Mode 2x cycle:
// Mount → cleanup → mount: ikkinchi setTimeout, lekin avvalgi cleared.
// Hatto tez user input: har keystroke avvalgi timer cancel.
```

Test:

```tsx
function SearchBox(): ReactElement {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);
  
  useEffect(() => {
    if (debouncedQuery) {
      console.log('Search:', debouncedQuery);
    }
  }, [debouncedQuery]);
  
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
// Tez yozish: faqat oxirgi keystroke'dan 300ms keyin search.
```

</details>

### Mashq 5: Production Profile + Optimization (Qiyin)

`UserDashboard` komponenti production'da slow — profile qilib bottleneck topib, optimize qiling. Boshlang'ich kod:

```tsx
import { useState, useEffect } from 'react';
import type { ReactElement } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
  avatar: string;
}

interface Notification {
  id: string;
  text: string;
  read: boolean;
}

function UserAvatar({ user }: { user: User }): ReactElement {
  // Imitatsiya: heavy image processing
  const processedAvatar = user.avatar.toUpperCase();
  return <img src={processedAvatar} alt={user.name} />;
}

function NotificationItem({ notification }: { notification: Notification }): ReactElement {
  return <li className={notification.read ? 'read' : 'unread'}>{notification.text}</li>;
}

function UserDashboard({ userId }: { userId: string }): ReactElement {
  const [user, setUser] = useState<User | null>(null);
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [searchQuery, setSearchQuery] = useState('');
  
  useEffect(() => {
    fetch(`/api/users/${userId}`).then(r => r.json()).then(setUser);
    fetch(`/api/notifications/${userId}`).then(r => r.json()).then(setNotifications);
  }, [userId]);
  
  const filteredNotifications = notifications.filter(n =>
    n.text.toLowerCase().includes(searchQuery.toLowerCase())
  );
  
  if (!user) return <div>Loading...</div>;
  
  return (
    <div>
      <UserAvatar user={user} />
      <h1>{user.name}</h1>
      <input
        value={searchQuery}
        onChange={(e) => setSearchQuery(e.target.value)}
        placeholder="Search notifications..."
      />
      <ul>
        {filteredNotifications.map((n) => (
          <NotificationItem key={n.id} notification={n} />
        ))}
      </ul>
    </div>
  );
}
```

Symptom: search box typing laggy (1000+ notifications).

<details>
<summary><strong>Javob</strong></summary>

Profile natijasi (taxminiy):

```
Profiler record (5 keystrokes):
- UserDashboard: 5 commits (each ~50ms)
- UserAvatar: 5 renders ❌ (har keystroke'da, lekin user props bir xil)
- NotificationItem: 5000 renders ❌ (1000 × 5 keystrokes)

"Why did this render?":
- UserAvatar: "Parent component rendered"
- NotificationItem: "Parent component rendered"
```

Bottleneck:
1. Har keystroke'da UserAvatar va NotificationItem'lar re-render bo'ladi (top-down).
2. NotificationItem 1000 ta — 5000 unnecessary re-render.
3. `filteredNotifications` har render qayta hisoblanadi.

Optimization:

```tsx
import { useState, useEffect, useMemo, useCallback, memo } from 'react';
import type { ReactElement } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
  avatar: string;
}

interface Notification {
  id: string;
  text: string;
  read: boolean;
}

// 1. UserAvatar memo — props bir xil bo'lsa re-render skip
const UserAvatar = memo(function UserAvatar({ user }: { user: User }): ReactElement {
  const processedAvatar = useMemo(() => user.avatar.toUpperCase(), [user.avatar]);
  return <img src={processedAvatar} alt={user.name} />;
});

// 2. NotificationItem memo
const NotificationItem = memo(function NotificationItem({
  notification,
}: {
  notification: Notification;
}): ReactElement {
  return <li className={notification.read ? 'read' : 'unread'}>{notification.text}</li>;
});

function UserDashboard({ userId }: { userId: string }): ReactElement {
  const [user, setUser] = useState<User | null>(null);
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [searchQuery, setSearchQuery] = useState('');
  
  useEffect(() => {
    fetch(`/api/users/${userId}`).then(r => r.json()).then(setUser);
    fetch(`/api/notifications/${userId}`).then(r => r.json()).then(setNotifications);
  }, [userId]);
  
  // 3. Filtered list memoize
  const filteredNotifications = useMemo(
    () => notifications.filter((n) =>
      n.text.toLowerCase().includes(searchQuery.toLowerCase())
    ),
    [notifications, searchQuery]
  );
  
  if (!user) return <div>Loading...</div>;
  
  return (
    <div>
      <UserAvatar user={user} />
      <h1>{user.name}</h1>
      <input
        value={searchQuery}
        onChange={(e) => setSearchQuery(e.target.value)}
        placeholder="Search notifications..."
      />
      <ul>
        {filteredNotifications.map((n) => (
          <NotificationItem key={n.id} notification={n} />
        ))}
      </ul>
    </div>
  );
}
```

Re-profile natijasi (taxminiy):

```
- UserDashboard: 5 commits (each ~10ms, 5x improvement)
- UserAvatar: 1 render (initial) ✅
- NotificationItem: 1000 renders initial, then re-render only filtered changes ✅
```

Qo'shimcha optimization (advanced):

```tsx
// 4. useTransition — search non-urgent (cross-ref 22, 30)
import { useTransition } from 'react';

function UserDashboard({ userId }: { userId: string }): ReactElement {
  const [searchQuery, setSearchQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  
  function handleSearchChange(value: string) {
    startTransition(() => {
      setSearchQuery(value);
    });
  }
  
  // ...
  
  return (
    <input
      value={searchQuery}
      onChange={(e) => handleSearchChange(e.target.value)}
      placeholder={isPending ? 'Filtering...' : 'Search...'}
    />
    // ...
  );
}

// 5. Virtualization (cross-ref 36) — 1000+ items
// react-window yoki @tanstack/react-virtual
// Faqat ko'rinadigan items DOM'ga render
```

React Compiler era:

```tsx
'use memo';

// Avtomatik:
// - UserAvatar memo equivalent
// - NotificationItem memo equivalent
// - filteredNotifications cache
// - processedAvatar cache
// Manual memoization olib tashlash mumkin
```

</details>

---

## Xulosa

Bu fayl re-render trigger'lar va debugging'ni practical perspective'dan o'rgandi:

- **Re-render** = komponent funksiyasi qayta chaqirilishi + Reconciler diff. **DOM update bilan teng emas** — bailout bo'lsa DOM mutation yo'q.
- **4 ta asosiy trigger** — state change (setState/dispatch), parent re-render (top-down propagation, default), Context value change (har consumer mustaqil), force update (rare patterns).
- **Top-down propagation** — parent re-render → children default re-render. To'xtatish: `React.memo`, children prop / composition, `useMemo` bilan element cache, Compiler. (`<Suspense>` propagation'ni to'xtatmaydi — u loading/error state vositasi.)
- **Reconciler bailout** — komponent funksiyasini umuman chaqirmaslik. 3 ta mexanizm: eager bailout (faqat `useState` dispatch'ida), reference bailout (`bailoutOnAlreadyFinishedWork` + `React.memo` shallow check), element identity bailout. `useMemo`/`useCallback` o'zi bailout emas — stable reference berib bailout sharoitini yaratadi.
- **`React.memo`** — shallow props comparison, default `Object.is` per property. Custom comparator `(prev, next) => boolean`. Compiler era — ortiqcha lekin ishlaydi.
- **5 ta `React.memo` bypass scenarios** — inline literals (object/array/function), Context dep (semantic), children prop (yangi React Element), spread props (har property check), ref prop (reference change).
- **Reference equality gotchas** — JavaScript `===`/`Object.is` reference identity, JSX transform har gal yangi React Element. Stable references via `useMemo`/`useCallback`/module-level constants/Compiler.
- **"New element same type"** — Reconciler update path, state preservation, DOM reuse. Different type → unmount/mount, state lost. Anonymous nested anti-pattern.
- **Stale closure** — empty deps useEffect, async callback, event listener. 3 yechim: functional update, useRef latest pattern, deps to'g'ri. `useEffectEvent` RFC (experimental — R19 canary/experimental build'larda).
- **DevTools "Why did this render?"** — re-render manbai topish: hooks changed, props changed (list), parent rendered, Context changed.
- **Highlight Updates** — visual debug, render frequency rangli.
- **Profile workflow** — bottleneck → highlight → record → why → optimize → re-profile → production monitoring (Profiler API).
- **Compiler era** — automatic memo equivalent, granular per-property cache, manual memoization aksariyat olib tashlash mumkin.

QISM 8 (Performance & Compiler) — 2/6 yozildi. Keyingi fayl `33-optimization.md` — `React.memo` real qo'llanish patterns, key-based reset trick, splitting state across components, Compiler era manual qoladigan optimization'lar.

---

**Keyingi bo'lim:** [33-optimization.md](33-optimization.md) — Optimization patterns real qo'llanish: `React.memo` shallow comparison + custom comparator, `useMemo`/`useCallback` qachon kerak qachon kerak emas (mexanika 21'da), key-based reset trick (komponent'ni "remount" qilib state reset), splitting state across components (re-render scope cheklash), Context optimization (split state vs dispatch, Provider value memo), premature optimization (Knuth) — profile birinchi, React Compiler era manual qoladigan qismlar.
