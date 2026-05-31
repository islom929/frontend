# Bo'lim 17: useLayoutEffect va useInsertionEffect

> `useLayoutEffect` — `useEffect` bilan bir xil API'ga ega, lekin React Commit Phase'ning Layout sub-phase'ida **synchron** chaqiriladi (DOM yangilangandan keyin, browser paint'dan oldin). DOM measurement, tooltip positioning, scroll restoration, focus management uchun mo'ljallangan. `useInsertionEffect` (R18) — Layout effect'dan ham oldin chaqiriladi, faqat CSS-in-JS library'lar uchun. Bu bo'limda har uch hook'ning timing farqlari, use case'lari, performance ta'sirlari, SSR cheklovi va `useIsomorphicLayoutEffect` workaround pattern'i chuqur yoritiladi.

---

## Mundarija

- [Timing Recap — `useLayoutEffect` Qayerda Joylashgan](#timing-recap--uselayouteffect-qayerda-joylashgan)
- [`useLayoutEffect` Signature va Asoslari](#uselayouteffect-signature-va-asoslari)
- [Timing Diagram — Render → Commit → Paint](#timing-diagram--render--commit--paint)
- [DOM Measurement Use Case](#dom-measurement-use-case)
- [Tooltip Positioning Pattern](#tooltip-positioning-pattern)
- [Scroll Restoration va Scroll Lock](#scroll-restoration-va-scroll-lock)
- [Focus Management](#focus-management)
- [Performance Pitfalls — Paint Bloklash](#performance-pitfalls--paint-bloklash)
- [SSR Cheklov — `useLayoutEffect` Server'da](#ssr-cheklov--uselayouteffect-serverda)
- [`useIsomorphicLayoutEffect` Workaround](#useisomorphiclayouteffect-workaround)
- [`useInsertionEffect` (R18) — CSS-in-JS Uchun](#useinsertioneffect-r18--css-in-js-uchun)
- [Decision Guide — `useEffect` / `useLayoutEffect` / `useInsertionEffect`](#decision-guide--useeffect--uselayouteffect--useinsertioneffect)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Timing Recap — `useLayoutEffect` Qayerda Joylashgan

### Nazariya

Avvalgi bo'lim ([`16-useeffect.md`](16-useeffect.md) "Effect Timing — Passive vs Layout") `useEffect`'ning paint'dan keyin chaqirilishini ko'rsatdi. `useLayoutEffect` esa paint'dan **oldin** chaqiriladi. Aniq farqni tushunish uchun React render lifecycle'ini batafsil ko'rib chiqish kerak (cross-ref [`02-rendering.md`](02-rendering.md)):

```
React Render Lifecycle (R18+):

1. Render Phase (memory'da, parallel mumkin, restart safe)
   ├─ Component function chaqiriladi
   ├─ JSX → Fiber tree (workInProgress)
   └─ Reconciliation (eski vs yangi tree)

2. Commit Phase (sync, atomic, restart yo'q)
   ├─ Before Mutation sub-phase
   │   └─ getSnapshotBeforeUpdate (legacy class)
   │
   ├─ Mutation sub-phase
   │   ├─ useInsertionEffect (R18+, CSS-in-JS) — DOM mutation'dan oldin
   │   ├─ Refs detach (old refs)
   │   └─ DOM mutation (insert/update/remove)
   │
   └─ Layout sub-phase
       ├─ Refs attach (new refs)
       ├─ useLayoutEffect (cleanup → setup, sync)
       └─ componentDidMount/componentDidUpdate (legacy class)

3. Browser Paint
   └─ Yangi DOM ekranda ko'rinadi

4. Passive Effects Phase (async, MessageChannel)
   └─ useEffect (cleanup → setup, async)
```

`useLayoutEffect` 2-bosqichning **Layout sub-phase**'ida chaqiriladi. Bu degani:

- DOM **allaqachon yangilangan** (Mutation sub-phase'da)
- Foydalanuvchi yangi DOM'ni **hali ko'rmagan** (paint qilinmagan)
- `useLayoutEffect` ichidan DOM o'qish (`getBoundingClientRect`, `offsetWidth`) **ishonchli** — yangi layout
- `useLayoutEffect` ichidan DOM yozish (style change, scroll set) — paint'dan oldin → flicker yo'q

`useLayoutEffect` — **synchron**. Browser paint qilolmaydi `useLayoutEffect` tugaguncha. Demak agar effect og'ir bo'lsa (uzoq CPU ish) — paint kechikadi → user UI freeze deb sezadi.

**`useEffect` bilan farq jadval:**

| Aspekt | `useEffect` | `useLayoutEffect` |
|--------|-------------|-------------------|
| Vaqti | Browser paint'dan **keyin** | DOM mutation'dan keyin, paint'dan **oldin** |
| Sync/Async | Async (MessageChannel scheduling) | Sync (paint'ni bloklaydi) |
| API | `useEffect(setup, deps?)` | `useLayoutEffect(setup, deps?)` |
| Cleanup | Bir xil pattern | Bir xil pattern |
| DOM measurement | Eski layout (paint dan keyin yangi paint kerak) | Yangi layout to'g'ridan-to'g'ri |
| Visual flicker | Bo'lishi mumkin (DOM o'zgarishi paint orasida) | Yo'q (paint'dan oldin) |
| Performance | Paint'ni bloklamaydi | Paint kechikishi mumkin |
| SSR | Ishlaydi (server'da hech narsa qilmaydi) | Server'da warning beradi |
| Use case | Network, subscription, timer, log | DOM measure, scroll, focus, layout-dependent calc |

**Rasmiy tavsiya:** `useEffect` default tanlov. `useLayoutEffect` faqat **visual flicker oldini olish** kerak bo'lganda. React docs (`react.dev` "useLayoutEffect"): *"This Hook is used less often than `useEffect`."*

<details>
<summary><strong>Under the Hood</strong></summary>

**Commit Phase implementation (soddalashtirilgan):**

```ts
function commitRoot(root: FiberRoot) {
  const finishedWork = root.finishedWork;
  
  // 1. Before Mutation (snapshot uchun)
  commitBeforeMutationEffects(root, finishedWork);
  
  // 2. Mutation
  commitMutationEffects(root, finishedWork);
  // - DOM update
  // - Refs detach
  // - useInsertionEffect (R18+)
  
  // 3. Layout (sync, paint dan oldin)
  commitLayoutEffects(finishedWork, root);
  // - Refs attach
  // - useLayoutEffect cleanup → setup
  // - componentDidMount / componentDidUpdate
  
  // 4. Schedule passive effects
  if (rootDoesHavePassiveEffects) {
    scheduleCallback(NormalSchedulerPriority, () => {
      flushPassiveEffects();
      // useEffect cleanup → setup (async)
      return null;
    });
  }
}
```

`commitLayoutEffects` — synchron. Browser bu funksiya tugaguncha keyingi step'ga (paint) o'ta olmaydi. JavaScript single-threaded — paint'ga vaqt yo'q.

**`commitLayoutEffectOnFiber` ichida:**

```ts
function commitHookEffectListMount(hookFlags: HookFlags, finishedWork: Fiber) {
  const updateQueue = finishedWork.updateQueue;
  let lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    
    do {
      if ((effect.tag & hookFlags) === hookFlags) {
        // Layout effect: tag = HookHasEffect | HookLayout
        const create = effect.create;
        effect.destroy = create();
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}

// Chaqirilish:
commitHookEffectListMount(HookHasEffect | HookLayout, fiber);
```

`HookLayout` flag — Layout sub-phase'da ishlaydigan effect'larni ajratadi. `HookPassive` esa passive (useEffect) uchun.

**Hook flags bitmask:**

```ts
const HookHasEffect = 0b0001;   // Effect run kerak (deps o'zgarganda set)
const HookInsertion = 0b0010;   // useInsertionEffect (R18+)
const HookLayout = 0b0100;      // useLayoutEffect
const HookPassive = 0b1000;     // useEffect
```

Bitmask'da bir nechta bayrog'ni AND qilib filterlash mumkin. React har sub-phase'da kerakli bayrog'lar bilan effect'larni run qiladi.

**Source citation:**

- `commitLayoutEffects` — facebook/react `packages/react-reconciler/src/ReactFiberCommitWork.js`
- HookFlags — facebook/react `packages/react-reconciler/src/ReactHookEffectTags.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Order test:**

```tsx
function OrderTest() {
  console.log('1. Render');
  
  useInsertionEffect(() => {
    console.log('3. useInsertionEffect (mutation oldidan)');
  });
  
  useLayoutEffect(() => {
    console.log('4. useLayoutEffect (mutation keyin, paint oldidan, sync)');
  });
  
  useEffect(() => {
    console.log('6. useEffect (paint keyin, async)');
  });
  
  console.log('2. JSX qaytariladi');
  
  return <div>Test</div>;
}

// Output (DevTools console):
// 1. Render
// 2. JSX qaytariladi
// (Commit Phase Mutation: DOM update)
// 3. useInsertionEffect
// (Commit Phase Layout: refs attach)
// 4. useLayoutEffect
// (5. Browser paint — visible)
// 6. useEffect
```

**Misol 2 — Visual farq (flicker):**

```tsx
// useEffect — flicker bor
function FlickeringBox() {
  const ref = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    if (ref.current) {
      ref.current.style.transform = 'translateX(200px)';
      // Paint dan keyin — user avval initial position'ni, keyin o'zgarishni ko'radi
    }
  }, []);
  
  return <div ref={ref} style={{ width: 100, height: 100, background: 'red' }} />;
}

// useLayoutEffect — flicker yo'q
function StableBox() {
  const ref = useRef<HTMLDivElement>(null);
  
  useLayoutEffect(() => {
    if (ref.current) {
      ref.current.style.transform = 'translateX(200px)';
      // Paint dan oldin — user to'g'ridan-to'g'ri yangi position ko'radi
    }
  }, []);
  
  return <div ref={ref} style={{ width: 100, height: 100, background: 'red' }} />;
}
```

**Misol 3 — Cross-ref `02-rendering.md`:**

```tsx
// Bu komponent Commit Phase'ning har sub-phase'iga teging
function CommitPhaseDemo() {
  const ref = useRef<HTMLDivElement>(null);
  
  // Mutation sub-phase'da chaqiriladi (R18+)
  useInsertionEffect(() => {
    // DOM o'zgartirish (CSS injection) — bu yerda mumkin
    // Lekin DOM'ni o'qish — qiyin (mutation hali tugamagan)
  });
  
  // Layout sub-phase'da chaqiriladi
  useLayoutEffect(() => {
    if (ref.current) {
      const rect = ref.current.getBoundingClientRect();
      console.log('Layout phase rect:', rect);  // ✅ Yangi layout
    }
  });
  
  // Paint dan keyin chaqiriladi
  useEffect(() => {
    if (ref.current) {
      const rect = ref.current.getBoundingClientRect();
      console.log('Passive phase rect:', rect);  // ✅ Ham yangi (paint qilingan)
    }
  });
  
  return <div ref={ref}>Demo</div>;
}
```

</details>

---

## `useLayoutEffect` Signature va Asoslari

### Nazariya

`useLayoutEffect` API `useEffect` bilan **aynan bir xil**:

```tsx
function useLayoutEffect(
  setup: () => void | (() => void),
  dependencies?: ReadonlyArray<unknown>
): void;
```

- **`setup`** — Layout sub-phase'da chaqiriladigan callback. `void` yoki cleanup function qaytaradi.
- **`dependencies`** — qachon qaytadan run qilishni belgilaydigan deps (Object.is comparison).

Cleanup, deps semantikasi, Strict Mode 2x cycle (R18+), exhaustive-deps linter — barchasi `useEffect` bilan bir xil ishlaydi. Faqat **timing** farq qiladi.

**Migration `useEffect` ↔ `useLayoutEffect`:**

```tsx
// Mavjud useEffect kod:
useEffect(() => {
  // Setup
  return () => { /* cleanup */ };
}, [deps]);

// Yangi useLayoutEffect kod (faqat hook nomini o'zgartirish):
useLayoutEffect(() => {
  // Setup — endi paint'dan oldin
  return () => { /* cleanup */ };
}, [deps]);
```

API farq yo'q. Migration bir hook nomini o'zgartirish.

**`useLayoutEffect` ichida `setState` — yangi cycle:**

`useLayoutEffect` ichida `setState` chaqirilsa, React **darhol qayta render qiladi** va yangi commit cycle boshlanadi. Browser paint qilmaydi avvalgi natijani — faqat oxirgi natijani ko'radi:

```tsx
function ConditionalLayout() {
  const ref = useRef<HTMLDivElement>(null);
  const [width, setWidth] = useState<number | null>(null);
  
  useLayoutEffect(() => {
    if (ref.current && width === null) {
      const measured = ref.current.offsetWidth;
      setWidth(measured);  // ⚠️ Yangi cycle (sync, paint dan oldin)
    }
  });
  
  return (
    <div ref={ref}>
      Width: {width === null ? 'measuring...' : `${width}px`}
    </div>
  );
}

// Lifecycle:
// 1. Render: width=null → JSX 'measuring...'
// 2. Commit Mutation: DOM yangilandi
// 3. Commit Layout: useLayoutEffect → setWidth(N)
// 4. (Paint qilmaydi — yangi cycle)
// 5. Render: width=N → JSX `${N}px`
// 6. Commit Mutation: DOM yangilandi
// 7. Commit Layout: useLayoutEffect → if shart false, skip
// 8. Browser paint: foydalanuvchi to'g'ridan-to'g'ri `${N}px` ni ko'radi
```

`useEffect`'da bu pattern flicker keltirib chiqaradi: avval 'measuring...' paint, keyin `${N}px` paint. `useLayoutEffect`'da paint orasida o'zgarish ko'rinmaydi.

**Trade-off:** Yangi cycle sync (paint kechikadi) → ko'p ishlatish performance'ga ta'sir qiladi.

**Strict Mode (R18+):**

`useLayoutEffect` ham 2x cycle'ga uchraydi (mount → cleanup → mount). Hammasi `useEffect`'dagi kabi:

```tsx
function StrictModeTest() {
  useLayoutEffect(() => {
    console.log('Layout setup');
    return () => console.log('Layout cleanup');
  }, []);
  
  return null;
}

// Strict Mode (R18+):
// Layout setup
// Layout cleanup
// Layout setup
//
// Cleanup invariant — useLayoutEffect ham bir xil
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`useLayoutEffect` implementation — `mountLayoutEffect`:**

```ts
function mountLayoutEffect(
  create: () => () => void,
  deps: Array<unknown> | null
): void {
  let fiberFlags: Flags = UpdateEffect;
  if (currentlyRenderingFiber.mode & StrictEffectsMode) {
    fiberFlags |= MountLayoutDevEffect;  // Strict Mode 2x cycle
  }
  
  return mountEffectImpl(
    fiberFlags,
    HookLayout,  // ← HookPassive emas
    create,
    deps
  );
}
```

`HookPassive` o'rniga `HookLayout` flag — Commit Phase'da Layout sub-phase'da run qilinishini belgilaydi.

**`updateLayoutEffect` — bir xil mexanizm:**

```ts
function updateLayoutEffect(
  create: () => () => void,
  deps: Array<unknown> | null
): void {
  return updateEffectImpl(UpdateEffect, HookLayout, create, deps);
}
```

Faqat hook flag farq. Implementation logic deps comparison va effect register `useEffect`'dagi bilan bir xil.

**Bir vaqtda chaqirilish — Layout sub-phase:**

```ts
function commitLayoutEffectOnFiber(
  finishedRoot: FiberRoot,
  current: Fiber | null,
  finishedWork: Fiber,
  committedLanes: Lanes,
): void {
  if ((finishedWork.flags & LayoutMask) !== NoFlags) {
    switch (finishedWork.tag) {
      case FunctionComponent:
      case ForwardRef:
      case SimpleMemoComponent:
        // ... refs attach ...
        
        // useLayoutEffect setup chaqirish
        commitHookEffectListMount(
          HookLayout | HookHasEffect,
          finishedWork
        );
        break;
      // ... boshqa fiber turlari ...
    }
  }
}
```

`useLayoutEffect` cleanup esa avval chaqiriladi (`commitLayoutUnmount`). R18'da atomic separation: avval barcha cleanup, keyin barcha setup (`useEffect` bilan bir xil pattern).

**Source citation:**

- `mountLayoutEffect` / `updateLayoutEffect` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`
- `commitLayoutEffectOnFiber` — facebook/react `packages/react-reconciler/src/ReactFiberCommitWork.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Bir xil API:**

```tsx
function ApiCompare() {
  const [count, setCount] = useState(0);
  
  // useEffect
  useEffect(() => {
    console.log('useEffect:', count);
    return () => console.log('useEffect cleanup:', count);
  }, [count]);
  
  // useLayoutEffect — bir xil API
  useLayoutEffect(() => {
    console.log('useLayoutEffect:', count);
    return () => console.log('useLayoutEffect cleanup:', count);
  }, [count]);
  
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// Click — count 0→1:
// Output:
// useLayoutEffect cleanup: 0  ← avval Layout cleanup
// useLayoutEffect: 1            ← keyin Layout setup
// (Browser paint)
// useEffect cleanup: 0          ← passive cleanup
// useEffect: 1                  ← passive setup
```

**Misol 2 — setState ichida (yangi cycle):**

```tsx
function MeasureFirst() {
  const ref = useRef<HTMLSpanElement>(null);
  const [width, setWidth] = useState(0);
  
  useLayoutEffect(() => {
    if (ref.current) {
      const w = ref.current.offsetWidth;
      if (w !== width) {
        setWidth(w);  // Yangi cycle, sync
      }
    }
  });
  
  return (
    <div>
      <span ref={ref}>Hello, World!</span>
      <p>Measured width: {width}px</p>
    </div>
  );
}

// 1. Render: width=0
// 2. Commit Mutation: DOM
// 3. Commit Layout: setWidth(150)
// 4. Render: width=150
// 5. Commit Mutation
// 6. Commit Layout: w === width → skip
// 7. Paint: foydalanuvchi 150px ni ko'radi (0px ko'rmaydi)
```

**Misol 3 — Cleanup pattern:**

```tsx
function ResizeListener() {
  const [size, setSize] = useState({ w: 0, h: 0 });
  
  useLayoutEffect(() => {
    const update = () => {
      setSize({ w: window.innerWidth, h: window.innerHeight });
    };
    
    update();  // Initial measure
    window.addEventListener('resize', update);
    
    return () => window.removeEventListener('resize', update);
  }, []);
  
  return <div>{size.w} × {size.h}</div>;
}

// Layout phase'da initial measure → paint dan oldin to'g'ri size
// useEffect bilan bo'lsa: avval 0×0 paint, keyin yangi size paint (flicker)
```

</details>

---

## Timing Diagram — Render → Commit → Paint

### Nazariya

Quyidagi diagrammada React render lifecycle'ining har bosqichida hooklar qachon chaqirilishi:

```
Time ─────────────────────────────────────────────────────────────────►

│ Render    │ Commit Phase                       │ Browser │ Passive  │
│ Phase     │                                    │ Paint   │ Effects  │
│ (memory)  │ Mutation │ Layout                  │         │ Phase    │
│           │          │                         │         │          │
│ • Comp fn │ • useIns │ • Refs attach           │ • Pixel │ • useEff │
│ • JSX     │   ertion │ • useLayoutEffect       │   draw  │   ect    │
│ • Diff    │   Effect │   (cleanup → setup)     │         │   (clean │
│           │   (R18)  │ • cdMount/cdUpdate      │         │   → setup│
│           │ • Refs   │   (legacy class)        │         │          │
│           │   detach │                         │         │          │
│           │ • DOM    │                         │         │          │
│           │   update │                         │         │          │
│           │          │                         │         │          │
│           │   ↓ sync ↓                         │   ↓     │   ↓      │
│           │   Browser bloklangan               │   ↓     │   async  │
│           │                                    │         │   (Mssg  │
│           │                                    │         │   Channel)
└───────────┴──────────┴─────────────────────────┴─────────┴──────────┘
```

**Asosiy nuqtalar:**

1. **Render Phase** — sof memory'da, DOM tegmaydi. Concurrent Mode'da to'xtatilishi mumkin.
2. **Commit Phase Mutation** — `useInsertionEffect` (R18+) avval chaqiriladi (CSS injection), keyin eski refs detach, so'ngra DOM o'zgaradi.
3. **Commit Phase Layout** — yangi refs attach, `useLayoutEffect` chaqiriladi (cleanup → setup), legacy class `componentDidMount/Update`. **Synchron — browser paint qilmaydi**.
4. **Browser Paint** — foydalanuvchi yangi UI'ni ko'radi.
5. **Passive Effects Phase** — `useEffect` chaqiriladi (cleanup → setup). **Async — MessageChannel orqali keyingi task'da**.

**Synchron va Async farqi vizuel:**

```
Render → Mutation → Layout (sync, paint kechikadi) → Paint → Passive (async)
                    ↑                                          ↑
                    Bu yerda CPU intensive ish bo'lsa,         Bu yerda CPU
                    paint sekinlanadi                           ish bo'lsa, paint'ga
                                                                ta'sir qilmaydi
```

**`useLayoutEffect` ichida `setState`:**

Layout phase'da `setState` chaqirilsa, React darhol yangi cycle boshlaydi:

```
Render 1 → Mutation 1 → Layout 1 (setState chaqirildi) →
Render 2 → Mutation 2 → Layout 2 → Paint → Passive
                                    ↑
                            Faqat 2-natija paint qilinadi
```

Bu — flicker'ni oldini olish mexanizmi. Lekin paint ikki cycle ishi tugaguncha kechikadi.

**`useEffect` ichida `setState`:**

```
Render 1 → Mutation 1 → Layout 1 → Paint 1 (1-natija ko'rinadi) → Passive (setState chaqirildi) →
Render 2 → Mutation 2 → Layout 2 → Paint 2 (2-natija ko'rinadi)
```

Foydalanuvchi avval 1-natijani, keyin 2-natijani ko'radi. Visible o'zgarish bor.

**Browser paint timing:**

60fps display'da browser har frame uchun ~16.67ms (1000ms / 60) vaqtga ega. JavaScript single-threaded — Layout phase ish vaqti shu budjetga qo'shiladi. Layout phase uzoq cho'zilsa, paint kechikadi (jank potensial). Ko'p Layout effect bo'lsa — frame budget tugashi mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

**Browser frame budget:**

60fps display'da bitta frame ~16.67ms (1000ms / 60). Bu vaqt ichida frame to'liq tayyorlanishi kerak:

```
Frame ~16.67ms (60fps)
├─ JavaScript (event handler, effect, render)
├─ Style calculation (CSS qoidalarini element'larga moslash)
├─ Layout (browser geometriya hisobi — reflow)
├─ Paint (piksel chizish)
└─ Composite (layer'larni birlashtirish)
```

Google'ning RAIL model'i bo'yicha JavaScript bitta frame ichida ~10ms'dan oshmasligi tavsiya etiladi — qolgan vaqt browser'ning style/layout/paint ishiga qoladi. JavaScript bu budjetdan oshib ketsa, browser frame'ni o'z vaqtida tayyorlay olmaydi → frame drop → user "jank" sezadi.

`useLayoutEffect` JavaScript execution'da, paint'dan oldin sinxron ishlaydi. Layout phase uzoq cho'zilsa — frame drop xavfi.

**Concurrent rendering bilan synergiya:**

Render Phase Concurrent Mode'da time slicing (5ms chunks) bilan ishlaydi (cross-ref [`05-scheduler-lanes.md`](05-scheduler-lanes.md)). Lekin Commit Phase **doim sync** — to'xtatib bo'lmaydi.

Demak `useLayoutEffect` — Concurrent rendering tomonidan optimize qilinmaydi. Paint kechikishi kafolatlanadi.

**Strict Mode'da timing:**

```
Render → Render (2x R16.3+) → Mutation → Layout → Paint → Passive
                                          ↓
                                          Strict Mode (R18+):
                                          Layout setup → Layout cleanup → Layout setup
                                          (yana paint kechikadi)
```

Development'da Strict Mode 2x cycle Layout effect uchun ham qo'llaniladi.

**Source citation:**

- RAIL model / frame budget — web.dev "Measure performance with the RAIL model"
- Concurrent rendering / `frameYieldMs` — facebook/react `packages/scheduler/src/Scheduler.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Console timing visualization:**

```tsx
function TimingDemo() {
  const [showText, setShowText] = useState(false);
  
  useLayoutEffect(() => {
    const start = performance.now();
    while (performance.now() - start < 50) {
      // Simulate 50ms ish
    }
    console.log('Layout effect tugadi (50ms ish)');
  });
  
  return (
    <div>
      <button onClick={() => setShowText(!showText)}>
        Toggle
      </button>
      {showText && <p>Hello!</p>}
    </div>
  );
}

// Click bilan: Toggle → 50ms freeze (Layout effect ish) → Paint
// User toggle javobini 50ms keyin ko'radi (jank)
```

**Misol 2 — Visible flicker `useEffect`'da:**

```tsx
function MeasureWithUseEffect() {
  const ref = useRef<HTMLDivElement>(null);
  const [maxHeight, setMaxHeight] = useState(0);
  
  useEffect(() => {
    if (ref.current) {
      const items = ref.current.querySelectorAll('.item');
      let max = 0;
      items.forEach(item => {
        max = Math.max(max, (item as HTMLElement).offsetHeight);
      });
      setMaxHeight(max);
    }
  }, []);
  
  return (
    <div ref={ref} style={{ display: 'flex' }}>
      <div className="item" style={{ height: maxHeight || 'auto' }}>Short</div>
      <div className="item" style={{ height: maxHeight || 'auto' }}>
        Long content<br />multiple lines
      </div>
    </div>
  );
}

// Lifecycle:
// 1. Render: maxHeight=0, items 'auto' height
// 2. Paint 1: items har xil heightlar
// 3. useEffect: measure → setMaxHeight(60)
// 4. Render: maxHeight=60
// 5. Paint 2: items teng height (60px)
//
// User Paint 1 va Paint 2 orasidagi o'zgarishni ko'radi (flicker)
```

**Misol 3 — Stable `useLayoutEffect`'da:**

```tsx
function MeasureWithUseLayoutEffect() {
  const ref = useRef<HTMLDivElement>(null);
  const [maxHeight, setMaxHeight] = useState(0);
  
  useLayoutEffect(() => {
    if (ref.current) {
      const items = ref.current.querySelectorAll('.item');
      let max = 0;
      items.forEach(item => {
        max = Math.max(max, (item as HTMLElement).offsetHeight);
      });
      setMaxHeight(max);
    }
  }, []);
  
  return (
    <div ref={ref} style={{ display: 'flex' }}>
      <div className="item" style={{ height: maxHeight || 'auto' }}>Short</div>
      <div className="item" style={{ height: maxHeight || 'auto' }}>
        Long content<br />multiple lines
      </div>
    </div>
  );
}

// Lifecycle:
// 1. Render: maxHeight=0
// 2. Commit Mutation: DOM yangilandi
// 3. Commit Layout: measure → setMaxHeight(60) → yangi cycle
// 4. Render: maxHeight=60
// 5. Commit: items teng height
// 6. Paint: foydalanuvchi to'g'ridan-to'g'ri 60px height'ni ko'radi
//
// Flicker yo'q
```

</details>

---

## DOM Measurement Use Case

### Nazariya

DOM measurement — `useLayoutEffect`'ning eng tipik use case'i. Komponent mount yoki update bo'lganida real DOM o'lchamini olish, undan boshqa hisoblash uchun foydalanish.

**Tipik scenariy:**

1. Komponent render qilinadi (initial DOM)
2. JavaScript real DOM o'lchamini o'qiydi (`getBoundingClientRect`, `offsetWidth`)
3. Olingan o'lcham asosida boshqa narsa hisoblaydi (e.g., tooltip positionsi, dynamic layout)
4. State yangilanadi → komponent re-render → yangi o'lcham bilan
5. Browser paint — foydalanuvchi faqat oxirgi natijani ko'radi

`useEffect`'da bu pattern flicker keltirib chiqaradi: paint orasida o'zgarish ko'rinadi. `useLayoutEffect` paint'dan oldin ishni tugatadi → flicker yo'q.

**DOM measurement API'lari:**

| API | Qaytadi | Use case |
|-----|---------|----------|
| `element.offsetWidth` / `offsetHeight` | int (px) | Padding + border bilan |
| `element.clientWidth` / `clientHeight` | int (px) | Padding bilan, border'siz |
| `element.scrollWidth` / `scrollHeight` | int (px) | Content full size |
| `element.getBoundingClientRect()` | DOMRect | x, y, width, height, top, right, bottom, left |
| `getComputedStyle(element)` | CSSStyleDeclaration | Computed CSS values |
| `window.innerWidth` / `innerHeight` | int (px) | Viewport size |

**Eng ko'p ishlatiladigani:** `getBoundingClientRect()` — element positionsi va o'lchami bir vaqtda. Float values qaytaradi (sub-pixel precision).

**Layout thrashing — qochish kerak:**

Layout phase'da DOM read va write aralash qilish — performance muammo. Browser har read paytida layout recalculate qilishga majbur:

```tsx
// ❌ Layout thrashing
useLayoutEffect(() => {
  for (const item of items) {
    item.style.width = `${item.offsetWidth + 10}px`;  // ❌ Read-write-read-write
  }
}, [items]);

// ✅ Read birinchi, keyin write
useLayoutEffect(() => {
  const widths = items.map(item => item.offsetWidth);  // Read all
  items.forEach((item, i) => {
    item.style.width = `${widths[i] + 10}px`;  // Write all
  });
}, [items]);
```

**`ResizeObserver` alternativ:**

`getBoundingClientRect` — bir martalik measurement. Element o'lchami o'zgarsa avtomatik update yo'q. `ResizeObserver` API element o'lchami o'zgarganda callback chaqiradi:

```tsx
function useElementSize(ref: React.RefObject<HTMLElement>) {
  const [size, setSize] = useState({ width: 0, height: 0 });
  
  useLayoutEffect(() => {
    if (!ref.current) return;
    
    const observer = new ResizeObserver(([entry]) => {
      const { width, height } = entry.contentRect;
      setSize({ width, height });
    });
    
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, [ref]);
  
  return size;
}
```

`ResizeObserver` — Chrome 64+, Firefox 69+, Safari 13.1+. Modern browser'larda universal.

<details>
<summary><strong>Under the Hood</strong></summary>

**Browser layout calculation:**

DOM o'qish (e.g., `offsetWidth`) browser'ni layout calculate qilishga majbur qiladi (agar o'tgan layout o'zgartirilgan bo'lsa). Bu jarayon "reflow" deb ataladi.

```ts
// ❌ Forced synchronous layout (FSL)
elementA.style.width = '100px';   // Write — layout invalid
const widthB = elementB.offsetWidth;  // Read — browser FSL kerak
elementA.style.height = '50px';   // Write
const widthC = elementC.offsetWidth;  // Read — yana FSL
// 2 ta FSL — 2x sekin
```

```ts
// ✅ Read-then-write pattern
const widthB = elementB.offsetWidth;  // Read first (bir FSL)
const widthC = elementC.offsetWidth;  // Cached
elementA.style.width = '100px';   // Write all
elementA.style.height = '50px';
// 1 ta FSL
```

`useLayoutEffect` ichida bu pattern muhim — har FSL paint kechikishi.

**`getBoundingClientRect` semantikasi:**

```ts
interface DOMRect {
  x: number;        // Element top-left x koordinatasi (viewport)
  y: number;        // Element top-left y koordinatasi (viewport)
  width: number;
  height: number;
  top: number;      // y bilan teng
  right: number;    // x + width
  bottom: number;   // y + height
  left: number;     // x bilan teng
}
```

`x`, `y` viewport'ga nisbatan (page'ga emas). Page'ga nisbatan: `x + window.scrollX`, `y + window.scrollY`.

**`ResizeObserver` performance:**

`ResizeObserver` browser'ning o'z layout cycle'iga integration qilingan — har frame'da bir marta callback chaqiradi (debounced). `useLayoutEffect`'da poll'lash o'rniga afzal.

```ts
// ❌ Polling (sekin)
useLayoutEffect(() => {
  const interval = setInterval(() => {
    if (ref.current) setSize(ref.current.getBoundingClientRect());
  }, 100);
  return () => clearInterval(interval);
}, []);

// ✅ ResizeObserver (efficient)
useLayoutEffect(() => {
  if (!ref.current) return;
  const observer = new ResizeObserver(...);
  observer.observe(ref.current);
  return () => observer.disconnect();
}, []);
```

**Source citation:**

- Forced synchronous layout — Chrome DevTools Performance docs
- ResizeObserver — W3C Resize Observer API spec

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Element width measurement:**

```tsx
function useElementWidth() {
  const ref = useRef<HTMLDivElement>(null);
  const [width, setWidth] = useState(0);
  
  useLayoutEffect(() => {
    if (ref.current) {
      setWidth(ref.current.offsetWidth);
    }
  }, []);
  
  return [ref, width] as const;
}

function ResponsiveBlock() {
  const [ref, width] = useElementWidth();
  
  return (
    <div ref={ref} style={{ padding: 16 }}>
      <p>Container width: {width}px</p>
      {width > 500 ? <DesktopLayout /> : <MobileLayout />}
    </div>
  );
}
```

**Misol 2 — Equal column heights:**

```tsx
function EqualHeightColumns({ columns }: { columns: string[] }) {
  const ref = useRef<HTMLDivElement>(null);
  const [height, setHeight] = useState<number | null>(null);
  
  useLayoutEffect(() => {
    if (!ref.current) return;
    
    const items = ref.current.querySelectorAll<HTMLElement>('.column');
    
    // Read first
    const heights = Array.from(items).map(el => el.scrollHeight);
    const max = Math.max(...heights);
    
    // Write to state (keyingi render minHeight'ni qo'llaydi)
    setHeight(max);
  }, [columns]);
  
  return (
    <div ref={ref} style={{ display: 'flex', gap: 16 }}>
      {columns.map((text, i) => (
        <div
          key={i}
          className="column"
          style={{
            flex: 1,
            minHeight: height ?? 'auto',
            border: '1px solid #ccc',
            padding: 12,
          }}
        >
          {text}
        </div>
      ))}
    </div>
  );
}
```

**Misol 3 — `ResizeObserver` hook:**

```tsx
function useResizeObserver<T extends HTMLElement>(): [
  React.RefObject<T>,
  { width: number; height: number }
] {
  const ref = useRef<T>(null);
  const [size, setSize] = useState({ width: 0, height: 0 });
  
  useLayoutEffect(() => {
    if (!ref.current) return;
    
    const observer = new ResizeObserver(([entry]) => {
      const { width, height } = entry.contentRect;
      setSize({ width, height });
    });
    
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);
  
  return [ref, size];
}

function AdaptiveLayout() {
  const [ref, { width }] = useResizeObserver<HTMLDivElement>();
  
  return (
    <div ref={ref}>
      <p>Width: {width}px</p>
      {width < 768 ? <Mobile /> : width < 1024 ? <Tablet /> : <Desktop />}
    </div>
  );
}
```

**Misol 4 — `getBoundingClientRect` viewport tracking:**

```tsx
function useElementPosition<T extends HTMLElement>() {
  const ref = useRef<T>(null);
  const [position, setPosition] = useState({ x: 0, y: 0, width: 0, height: 0 });
  
  useLayoutEffect(() => {
    if (!ref.current) return;
    
    const update = () => {
      if (!ref.current) return;
      const rect = ref.current.getBoundingClientRect();
      setPosition({
        x: rect.x,
        y: rect.y,
        width: rect.width,
        height: rect.height,
      });
    };
    
    update();
    window.addEventListener('scroll', update);
    window.addEventListener('resize', update);
    
    return () => {
      window.removeEventListener('scroll', update);
      window.removeEventListener('resize', update);
    };
  }, []);
  
  return [ref, position] as const;
}
```

**Misol 5 — Read-then-write pattern:**

```tsx
function ScaleByContent() {
  const ref = useRef<HTMLDivElement>(null);
  
  useLayoutEffect(() => {
    if (!ref.current) return;
    
    const items = ref.current.querySelectorAll<HTMLElement>('.item');
    
    // Read all (bir FSL)
    const widths = Array.from(items).map(item => item.offsetWidth);
    
    // Write all
    items.forEach((item, i) => {
      const scale = Math.min(200 / widths[i], 1);
      item.style.transform = `scale(${scale})`;
    });
  });
  
  return (
    <div ref={ref}>
      <div className="item">Item 1</div>
      <div className="item">Item with much longer text</div>
      <div className="item">Short</div>
    </div>
  );
}
```

</details>

---

## Tooltip Positioning Pattern

### Nazariya

Tooltip — element ustiga hover qilganda paydo bo'ladigan kichik UI. Position'ini aniqlash uchun:

1. Trigger element positionsini o'qish (`getBoundingClientRect`)
2. Tooltip o'lchamini o'qish
3. Viewport chegaralarini hisobga olish (off-screen bo'lmasligi)
4. Tooltip positionsini hisoblash
5. State yangilash → tooltip render qilingan joyda

Bu — `useLayoutEffect`'ning klassik use case. `useEffect`'da paint orasida tooltip noto'g'ri joyda ko'rinadi (flicker).

**Position'lash logikasi:**

```
Trigger (button)
    │
    ├─ getBoundingClientRect() → { x, y, width, height }
    │
    ├─ Tooltip default: trigger ustida (top)
    │   tooltip.x = trigger.x + trigger.width/2 - tooltip.width/2
    │   tooltip.y = trigger.y - tooltip.height - 8
    │
    └─ Off-screen check:
        if (tooltip.y < 0) → trigger ostiga (bottom)
        if (tooltip.x < 0) → left = 0
        if (tooltip.x + tooltip.width > viewport.width) → align right
```

**Modern alternative — Floating UI (kursdan tashqari):**

`@floating-ui/react` — production'da tooltip/popover/dropdown uchun library. `getBoundingClientRect`, viewport detection, scroll tracking, collision avoidance — barchasi avtomatik. Bu kursda manual implement misol uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

**Portal'lar bilan integration:**

Tooltip ko'pincha `<body>` ostida render qilinadi (z-index, overflow:hidden parent'lardan qutilish). React Portal (`createPortal`, cross-ref [`28-portals.md`](28-portals.md)):

```tsx
import { createPortal } from 'react-dom';

function Tooltip({ targetRef, children }: {
  targetRef: React.RefObject<HTMLElement>;
  children: React.ReactNode;
}) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useLayoutEffect(() => {
    if (!targetRef.current) return;
    const rect = targetRef.current.getBoundingClientRect();
    setPosition({ x: rect.x + rect.width / 2, y: rect.y - 8 });
  }, [targetRef]);
  
  return createPortal(
    <div style={{ position: 'fixed', left: position.x, top: position.y }}>
      {children}
    </div>,
    document.body
  );
}
```

Portal — `useLayoutEffect` bilan birga. Render Phase Portal child'larni kerakli DOM joyiga joylashtiradi, useLayoutEffect Layout Phase'da measure qiladi.

**Scroll va resize tracking:**

Tooltip positionsi viewport o'zgarishlarida (scroll, resize) yangilanishi kerak. `useLayoutEffect` ichida event listener:

```ts
useLayoutEffect(() => {
  const update = () => calculatePosition();
  
  update();
  window.addEventListener('scroll', update, { passive: true });
  window.addEventListener('resize', update);
  
  return () => {
    window.removeEventListener('scroll', update);
    window.removeEventListener('resize', update);
  };
}, []);
```

`{ passive: true }` — scroll listener performance uchun (preventDefault chaqirilmaydi).

**Source citation:**

- Floating UI library — `@floating-ui/react` GitHub
- React Portal — react.dev `createPortal`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Sodda tooltip (above):**

```tsx
function Tooltip({ children, content }: {
  children: React.ReactElement;
  content: string;
}) {
  const triggerRef = useRef<HTMLElement>(null);
  const tooltipRef = useRef<HTMLDivElement>(null);
  const [show, setShow] = useState(false);
  const [pos, setPos] = useState({ x: 0, y: 0 });
  
  useLayoutEffect(() => {
    if (!show || !triggerRef.current || !tooltipRef.current) return;
    
    const triggerRect = triggerRef.current.getBoundingClientRect();
    const tooltipRect = tooltipRef.current.getBoundingClientRect();
    
    setPos({
      x: triggerRect.x + triggerRect.width / 2 - tooltipRect.width / 2,
      y: triggerRect.y - tooltipRect.height - 8,
    });
  }, [show]);
  
  return (
    <>
      {React.cloneElement(children, {
        ref: triggerRef,
        onMouseEnter: () => setShow(true),
        onMouseLeave: () => setShow(false),
      })}
      {show && (
        <div
          ref={tooltipRef}
          style={{
            position: 'fixed',
            left: pos.x,
            top: pos.y,
            background: '#333',
            color: 'white',
            padding: '4px 8px',
            borderRadius: 4,
            pointerEvents: 'none',
          }}
        >
          {content}
        </div>
      )}
    </>
  );
}

// Usage:
<Tooltip content="Save changes">
  <button>Save</button>
</Tooltip>
```

**Misol 2 — Viewport-aware tooltip:**

```tsx
function SmartTooltip({ children, content }: {
  children: React.ReactElement;
  content: string;
}) {
  const triggerRef = useRef<HTMLElement>(null);
  const tooltipRef = useRef<HTMLDivElement>(null);
  const [show, setShow] = useState(false);
  const [position, setPosition] = useState<{
    x: number;
    y: number;
    placement: 'top' | 'bottom';
  }>({ x: 0, y: 0, placement: 'top' });
  
  useLayoutEffect(() => {
    if (!show || !triggerRef.current || !tooltipRef.current) return;
    
    const trigger = triggerRef.current.getBoundingClientRect();
    const tooltip = tooltipRef.current.getBoundingClientRect();
    const viewport = {
      width: window.innerWidth,
      height: window.innerHeight,
    };
    
    let x = trigger.x + trigger.width / 2 - tooltip.width / 2;
    let y = trigger.y - tooltip.height - 8;
    let placement: 'top' | 'bottom' = 'top';
    
    // Off-screen top → bottom'ga ko'chirish
    if (y < 0) {
      y = trigger.y + trigger.height + 8;
      placement = 'bottom';
    }
    
    // Off-screen left
    if (x < 8) x = 8;
    // Off-screen right
    if (x + tooltip.width > viewport.width - 8) {
      x = viewport.width - tooltip.width - 8;
    }
    
    setPosition({ x, y, placement });
  }, [show, content]);
  
  return (
    <>
      {React.cloneElement(children, {
        ref: triggerRef,
        onMouseEnter: () => setShow(true),
        onMouseLeave: () => setShow(false),
      })}
      {show && (
        <div
          ref={tooltipRef}
          role="tooltip"
          style={{
            position: 'fixed',
            left: position.x,
            top: position.y,
            background: '#333',
            color: 'white',
            padding: '6px 10px',
            borderRadius: 4,
            pointerEvents: 'none',
            fontSize: 14,
          }}
        >
          {content}
        </div>
      )}
    </>
  );
}
```

**Misol 3 — Scroll-aware (event listener):**

```tsx
function StickyTooltip({ targetRef, content }: {
  targetRef: React.RefObject<HTMLElement>;
  content: string;
}) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  
  useLayoutEffect(() => {
    if (!targetRef.current) return;
    
    const updatePosition = () => {
      if (!targetRef.current) return;
      const rect = targetRef.current.getBoundingClientRect();
      setPos({
        x: rect.x + rect.width / 2,
        y: rect.bottom + 4,
      });
    };
    
    updatePosition();
    window.addEventListener('scroll', updatePosition, { passive: true });
    window.addEventListener('resize', updatePosition);
    
    return () => {
      window.removeEventListener('scroll', updatePosition);
      window.removeEventListener('resize', updatePosition);
    };
  }, [targetRef]);
  
  return (
    <div
      style={{
        position: 'fixed',
        left: pos.x,
        top: pos.y,
        transform: 'translateX(-50%)',
        background: 'white',
        padding: 8,
        boxShadow: '0 2px 8px rgba(0,0,0,0.1)',
      }}
    >
      {content}
    </div>
  );
}
```

</details>

---

## Scroll Restoration va Scroll Lock

### Nazariya

Scroll restoration — sahifa orasida o'tganida scroll positionsini saqlash va qaytarish. Klassik use case'lar:

- Back button: avvalgi sahifaga qaytganda scroll positionsi
- Tab switching: tab'ga qaytganida scroll
- Modal close: modal yopilganida sahifa scroll positionsi tiklanadi

`useLayoutEffect` ishlatiladi chunki scroll set'i paint'dan oldin bo'lishi kerak — yo'qsa user avval boshqa positionni, keyin to'g'ri positionni ko'radi (flicker).

**Scroll API:**

| API | Ma'no |
|-----|-------|
| `window.scrollY` / `pageYOffset` | Sahifa vertikal scroll px |
| `element.scrollTop` | Element ichki vertikal scroll |
| `window.scrollTo({ top, behavior })` | Scroll positionsini set qilish |
| `element.scrollIntoView()` | Element ko'rinishga keltirish |

**Scroll lock — body scroll'ni to'xtatish:**

Modal ochilganida body scroll'ni to'xtatish (modal ichida scroll faqat). Tipik pattern:

```tsx
useLayoutEffect(() => {
  const original = document.body.style.overflow;
  document.body.style.overflow = 'hidden';
  
  return () => {
    document.body.style.overflow = original;
  };
}, []);
```

`useLayoutEffect` chunki user modal ochish bilan birga "scroll hidden"'ni darrov ko'rishi kerak. `useEffect`'da scroll lock paint orasida, modal ochilgandan keyin yoqiladi.

**iOS Safari muammo:**

iOS Safari'da `overflow: hidden` body scroll'ni to'xtatmaydi (mobile bug). Yechim — `position: fixed` + scroll position saqlash:

```tsx
function useBodyScrollLock(active: boolean) {
  useLayoutEffect(() => {
    if (!active) return;
    
    const scrollY = window.scrollY;
    document.body.style.position = 'fixed';
    document.body.style.top = `-${scrollY}px`;
    document.body.style.width = '100%';
    
    return () => {
      document.body.style.position = '';
      document.body.style.top = '';
      document.body.style.width = '';
      window.scrollTo(0, scrollY);  // Scroll qaytarish
    };
  }, [active]);
}
```

**Restoration pattern — sessionStorage:**

```tsx
function useScrollRestoration(key: string) {
  useLayoutEffect(() => {
    const saved = sessionStorage.getItem(`scroll:${key}`);
    if (saved) {
      window.scrollTo(0, parseInt(saved, 10));
    }
    
    return () => {
      sessionStorage.setItem(`scroll:${key}`, String(window.scrollY));
    };
  }, [key]);
}
```

`useLayoutEffect` — restoration paint'dan oldin → user to'g'ridan-to'g'ri saqlangan positionni ko'radi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Browser native scroll restoration:**

Browser'lar default scroll restoration o'z mexanizmiga ega (back/forward navigation). React app'larda bu ko'pincha buziladi (SPA navigation history pop bilan ishlamaydi).

`history.scrollRestoration` API:

```ts
history.scrollRestoration = 'manual';  // Default 'auto'
// Endi React (yoki router) o'zi scroll boshqaradi
```

React Router, Next.js — `history.scrollRestoration = 'manual'` qo'yib, o'z restoration ishlatadi.

**`scrollIntoView` semantikasi:**

```ts
element.scrollIntoView({
  behavior: 'smooth',  // yoki 'instant', 'auto'
  block: 'start',      // 'start' | 'center' | 'end' | 'nearest'
  inline: 'nearest',
});
```

`useLayoutEffect`'da `behavior: 'smooth'` ishlatish — animation paint orasida boshlanadi (smooth animation paint cycle'da). Ko'pincha `behavior: 'instant'` afzal layout phase'da.

**Source citation:**

- `history.scrollRestoration` — WHATWG HTML spec
- `scrollIntoView` — CSSOM View Module spec

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Modal scroll lock:**

```tsx
function Modal({ isOpen, children }: { isOpen: boolean; children: React.ReactNode }) {
  useLayoutEffect(() => {
    if (!isOpen) return;
    
    const originalOverflow = document.body.style.overflow;
    document.body.style.overflow = 'hidden';
    
    return () => {
      document.body.style.overflow = originalOverflow;
    };
  }, [isOpen]);
  
  if (!isOpen) return null;
  
  return (
    <div className="modal-backdrop">
      <div className="modal-content">{children}</div>
    </div>
  );
}
```

**Misol 2 — Tab content scroll preservation:**

```tsx
function TabPanel({ tabId, content }: { tabId: string; content: string }) {
  const ref = useRef<HTMLDivElement>(null);
  const scrollPositions = useRef<Record<string, number>>({});
  
  useLayoutEffect(() => {
    if (!ref.current) return;
    
    // Restore
    const saved = scrollPositions.current[tabId] ?? 0;
    ref.current.scrollTop = saved;
    
    // Save on cleanup
    return () => {
      if (ref.current) {
        scrollPositions.current[tabId] = ref.current.scrollTop;
      }
    };
  }, [tabId]);
  
  return (
    <div ref={ref} style={{ height: 400, overflowY: 'auto' }}>
      {content}
    </div>
  );
}
```

**Misol 3 — Scroll to bottom (chat):**

```tsx
function ChatMessages({ messages }: { messages: Message[] }) {
  const ref = useRef<HTMLDivElement>(null);
  
  useLayoutEffect(() => {
    if (ref.current) {
      ref.current.scrollTop = ref.current.scrollHeight;
      // Yangi message qo'shilganda — eng pastga scroll
      // useLayoutEffect: paint dan oldin → user faqat oxirgi positionni ko'radi
    }
  }, [messages.length]);
  
  return (
    <div ref={ref} style={{ height: 400, overflowY: 'auto' }}>
      {messages.map(m => <div key={m.id}>{m.text}</div>)}
    </div>
  );
}
```

**Misol 4 — sessionStorage restoration:**

```tsx
function useScrollRestoration(key: string) {
  useLayoutEffect(() => {
    const saved = sessionStorage.getItem(`scroll:${key}`);
    if (saved) {
      window.scrollTo(0, parseInt(saved, 10));
    }
    
    const handleSave = () => {
      sessionStorage.setItem(`scroll:${key}`, String(window.scrollY));
    };
    
    window.addEventListener('beforeunload', handleSave);
    
    return () => {
      handleSave();
      window.removeEventListener('beforeunload', handleSave);
    };
  }, [key]);
}

function ProductPage({ productId }: { productId: string }) {
  useScrollRestoration(`product:${productId}`);
  
  return <div>...</div>;
}
```

**Misol 5 — iOS Safari safe scroll lock:**

```tsx
function useBodyScrollLock(active: boolean) {
  useLayoutEffect(() => {
    if (!active) return;
    
    const scrollY = window.scrollY;
    const original = {
      position: document.body.style.position,
      top: document.body.style.top,
      width: document.body.style.width,
      overflow: document.body.style.overflow,
    };
    
    document.body.style.position = 'fixed';
    document.body.style.top = `-${scrollY}px`;
    document.body.style.width = '100%';
    document.body.style.overflow = 'hidden';
    
    return () => {
      document.body.style.position = original.position;
      document.body.style.top = original.top;
      document.body.style.width = original.width;
      document.body.style.overflow = original.overflow;
      window.scrollTo(0, scrollY);
    };
  }, [active]);
}
```

</details>

---

## Focus Management

### Nazariya

Focus management — DOM element'ni programmatically focus qilish (`element.focus()`). Accessibility uchun kritik:

- Modal ochilganida — birinchi input'ga focus
- Modal yopilganida — trigger button'ga focus qaytarish
- Form validation error — birinchi error input'ga focus
- Search bar — Cmd+K bilan focus
- Page navigation — yangi page'ning main heading'iga focus

`useLayoutEffect` ishlatiladi chunki focus paint'dan oldin bo'lishi kerak — yo'qsa user "blink" sezadi (avval cursor bir joyda, keyin boshqa joyda).

**`element.focus()` semantikasi:**

```ts
element.focus({
  preventScroll: false,  // Default — focus element ko'rinadigan joyga scroll
});
```

`preventScroll: true` — focus qiladi, lekin scroll qilmaydi. Modal pattern'larda foydali (modal ichida focus, lekin sahifa scroll bo'lmasin).

**Focus trap:**

Modal ichida Tab tugmasi faqat modal ichidagi element'lar orasida o'tishi kerak (focus modal'dan chiqib ketmasligi). Bu — accessibility WCAG 2.1 talab.

**`autoFocus` attribute:**

JSX'da `autoFocus` attribute mavjud, lekin u **mount paytida bir marta** ishlaydi:

```tsx
<input autoFocus />  // Mount paytida focus
```

Conditional focus (e.g., dynamically focus kerak bo'lganda) — `useRef` + `useLayoutEffect`:

```tsx
function ConditionalFocus({ shouldFocus }: { shouldFocus: boolean }) {
  const ref = useRef<HTMLInputElement>(null);
  
  useLayoutEffect(() => {
    if (shouldFocus) {
      ref.current?.focus();
    }
  }, [shouldFocus]);
  
  return <input ref={ref} />;
}
```

**Focus return — modal close:**

Modal yopilganida focus modalni ochgan element'ga qaytishi kerak:

```tsx
function useFocusReturn() {
  const previousFocus = useRef<HTMLElement | null>(null);
  
  useLayoutEffect(() => {
    previousFocus.current = document.activeElement as HTMLElement;
    
    return () => {
      previousFocus.current?.focus();
    };
  }, []);
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Browser focus model:**

Har vaqt bitta element fokusda. `document.activeElement` — joriy fokusdagi element. Element fokuslanadigan bo'lishi uchun:

- Native interactive (button, input, textarea, select, a[href])
- `tabindex` attribute (`tabindex="0"` — tab order, `tabindex="-1"` — focus mumkin lekin tab'da yo'q)

`useLayoutEffect`'da `focus()` chaqirish — paint'dan oldin focus o'rnatiladi. Browser focus state DOM'da yangilanadi, paint paytida cursor to'g'ri joyda.

**Focus event ordering:**

```ts
// Focus A'dan B'ga ko'chganda event tartibi (MDN FocusEvent "Order of events"):
// 1. `blur`     — A focus'ni yo'qotadi (bubble qilmaydi)
// 2. `focusout` — A'da, `blur`'dan keyin (bubble qiladi)
// 3. `focus`    — B focus oladi (bubble qilmaydi)
// 4. `focusin`  — B'da, `focus`'dan keyin (bubble qiladi)
```

`blur`/`focus` bubble qilmaydi; `focusout`/`focusin` esa bubble qiladi (delegation uchun). React `onBlur`/`onFocus` — SyntheticEvent sifatida bubble bo'ladigan variantga (`focusout`/`focusin`) bog'lanadi, shuning uchun React'da `onBlur`/`onFocus` parent'ga bubble bo'ladi.

**Focus trap implementation:**

```ts
function useFocusTrap(containerRef: React.RefObject<HTMLElement>) {
  useLayoutEffect(() => {
    const container = containerRef.current;
    if (!container) return;
    
    const focusableSelector = 'button, input, textarea, select, a[href], [tabindex]:not([tabindex="-1"])';
    
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      
      const focusable = container.querySelectorAll<HTMLElement>(focusableSelector);
      const first = focusable[0];
      const last = focusable[focusable.length - 1];
      
      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault();
        last.focus();
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault();
        first.focus();
      }
    };
    
    container.addEventListener('keydown', handleKeyDown);
    return () => container.removeEventListener('keydown', handleKeyDown);
  }, [containerRef]);
}
```

**`inert` attribute (R19+):**

R19'da `inert` attribute qo'llab-quvvatlanadi — element va uning child'larini focus va event'lardan ajratadi:

```tsx
<div inert={!modalOpen}>
  Sahifa kontenti — modal ochiq bo'lsa interactive emas
</div>
<Modal isOpen={modalOpen} />
```

`inert` — focus trap'ning yarmini avtomatik hal qiladi. Browser support: Chrome 102+, Firefox 112+, Safari 15.5+.

**Source citation:**

- `element.focus()` — WHATWG HTML spec
- `inert` attribute — WHATWG HTML spec
- WCAG 2.1 focus management — W3C accessibility guidelines

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Modal birinchi input focus:**

```tsx
function LoginModal({ isOpen }: { isOpen: boolean }) {
  const inputRef = useRef<HTMLInputElement>(null);
  
  useLayoutEffect(() => {
    if (isOpen && inputRef.current) {
      inputRef.current.focus();
    }
  }, [isOpen]);
  
  if (!isOpen) return null;
  
  return (
    <div className="modal">
      <h2>Login</h2>
      <input ref={inputRef} type="email" placeholder="Email" />
      <input type="password" placeholder="Password" />
      <button>Submit</button>
    </div>
  );
}
```

**Misol 2 — Form validation error focus:**

```tsx
type FormErrors = Record<string, string>;

function ContactForm() {
  const [errors, setErrors] = useState<FormErrors>({});
  const refs = {
    name: useRef<HTMLInputElement>(null),
    email: useRef<HTMLInputElement>(null),
    message: useRef<HTMLTextAreaElement>(null),
  };
  
  useLayoutEffect(() => {
    const firstErrorField = Object.keys(errors)[0] as keyof typeof refs;
    if (firstErrorField && refs[firstErrorField].current) {
      refs[firstErrorField].current.focus();
    }
  }, [errors]);
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    const newErrors: FormErrors = {};
    // Validation
    if (!nameValue) newErrors.name = 'Required';
    setErrors(newErrors);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input ref={refs.name} aria-invalid={!!errors.name} />
      <input ref={refs.email} aria-invalid={!!errors.email} />
      <textarea ref={refs.message} aria-invalid={!!errors.message} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

**Misol 3 — Focus return on modal close:**

```tsx
function useFocusReturn(active: boolean) {
  const previousFocus = useRef<HTMLElement | null>(null);
  
  useLayoutEffect(() => {
    if (!active) return;
    
    previousFocus.current = document.activeElement as HTMLElement;
    
    return () => {
      previousFocus.current?.focus();
    };
  }, [active]);
}

function Dialog({ isOpen, onClose }: { isOpen: boolean; onClose: () => void }) {
  useFocusReturn(isOpen);
  
  if (!isOpen) return null;
  
  return (
    <div role="dialog">
      <button onClick={onClose}>Close</button>
    </div>
  );
}

// Lifecycle:
// 1. User trigger button'ni bosadi → openDialog()
// 2. Dialog mount → useFocusReturn save: document.activeElement = trigger
// 3. User Close bosadi → onClose()
// 4. Dialog unmount → cleanup: trigger.focus()
// 5. Focus trigger button'ga qaytdi
```

**Misol 4 — Focus trap (modal):**

```tsx
function useFocusTrap<T extends HTMLElement>(active: boolean) {
  const containerRef = useRef<T>(null);
  
  useLayoutEffect(() => {
    if (!active || !containerRef.current) return;
    
    const container = containerRef.current;
    const focusable = container.querySelectorAll<HTMLElement>(
      'button, input, textarea, select, a[href], [tabindex]:not([tabindex="-1"])'
    );
    const first = focusable[0];
    const last = focusable[focusable.length - 1];
    
    first?.focus();
    
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      
      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault();
        last?.focus();
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault();
        first?.focus();
      }
    };
    
    container.addEventListener('keydown', handleKeyDown);
    return () => container.removeEventListener('keydown', handleKeyDown);
  }, [active]);
  
  return containerRef;
}

function Modal({ isOpen }: { isOpen: boolean }) {
  const ref = useFocusTrap<HTMLDivElement>(isOpen);
  
  if (!isOpen) return null;
  
  return (
    <div ref={ref} role="dialog">
      <input placeholder="First" />
      <input placeholder="Second" />
      <button>OK</button>
      <button>Cancel</button>
      {/* Tab faqat bu 4 element orasida */}
    </div>
  );
}
```

**Misol 5 — Cmd+K search bar focus:**

```tsx
function GlobalSearch() {
  const inputRef = useRef<HTMLInputElement>(null);
  const [open, setOpen] = useState(false);
  
  useLayoutEffect(() => {
    if (open && inputRef.current) {
      inputRef.current.focus();
    }
  }, [open]);
  
  useEffect(() => {
    const handler = (e: KeyboardEvent) => {
      if ((e.metaKey || e.ctrlKey) && e.key === 'k') {
        e.preventDefault();
        setOpen(true);
      } else if (e.key === 'Escape') {
        setOpen(false);
      }
    };
    
    window.addEventListener('keydown', handler);
    return () => window.removeEventListener('keydown', handler);
  }, []);
  
  if (!open) return null;
  
  return (
    <div className="search-overlay">
      <input ref={inputRef} placeholder="Search..." />
    </div>
  );
}
```

</details>

---

## Performance Pitfalls — Paint Bloklash

### Nazariya

`useLayoutEffect` synchron — browser paint qila olmaydi effect tugaguncha. Effect ichida og'ir ish bo'lsa — paint kechikadi, user "jank" sezadi.

**Frame budget — 60fps va 120fps:**

Frame budget refresh rate'dan kelib chiqadi: `1000ms / refresh_rate`.

| Refresh rate | Frame budget |
|--------------|--------------|
| 60Hz (default) | ~16.67ms (1000 / 60) |
| 120Hz (modern displays) | ~8.33ms (1000 / 120) |

Bu budjet ichida JavaScript, style, layout, paint, composite — barchasi sig'ishi kerak. Refresh rate qancha baland bo'lsa, budget shuncha qisqa, demak `useLayoutEffect` ishiga toqat shuncha kam. 120Hz display'da paint'ni bloklamaslik uchun Layout phase ishi 60Hz'dagidan ham qisqaroq bo'lishi kerak.

**Layout phase'ni og'irlashtirmaslik:**

`useLayoutEffect` ichidagi ish (querySelectorAll, ko'p element measurement, layout thrashing) frame budget'ni yeydi. Aniq ms qiymatlar device, brauzer va kontentga bog'liq — taxminiy raqamga tayanmaslik, har bir holatni DevTools Performance tab bilan o'lchash kerak. Asosiy printsip: layout thrashing (read-write aralash) bir xil ishni read-then-write pattern'idan sezilarli darajada sekinlashtiradi.

**Paint bloklash signal'lari:**

1. Click javobi sezilarli kechikish bilan ko'rinadi (RAIL model'i interaktiv javob uchun ~100ms budjetni tavsiya etadi)
2. Scroll → uzilish (har frame Layout effect)
3. Animation → frame drop (transition glitch)
4. Input typing → kechikish (har keystroke)

**Yechim 1 — Effect logic optimize qilish:**

```tsx
// ❌ Har render'da og'ir hisoblash
useLayoutEffect(() => {
  const items = document.querySelectorAll('.item');
  let total = 0;
  items.forEach(item => {
    total += item.getBoundingClientRect().height;
  });
  setTotalHeight(total);
});

// ✅ Deps bilan — faqat kerak bo'lganda
useLayoutEffect(() => {
  // ... og'ir hisoblash
}, [items]);
```

**Yechim 2 — `useEffect`'ga ko'chirish (visual flicker yo'q bo'lsa):**

```tsx
// useLayoutEffect kerak emas bo'lsa — useEffect afzal
useEffect(() => {
  // Network call, subscription — paint dan keyin OK
});
```

**Yechim 3 — `requestAnimationFrame` (animation):**

```tsx
useLayoutEffect(() => {
  const id = requestAnimationFrame(() => {
    // Heavy animation logic
    // Yangi frame'da — paint bloklamaydi joriy frame
  });
  
  return () => cancelAnimationFrame(id);
}, [trigger]);
```

`requestAnimationFrame` — keyingi paint'dan oldin chaqiriladi. Joriy paint bloklanmaydi.

**Yechim 4 — `ResizeObserver` / `IntersectionObserver`:**

```tsx
// ❌ Har render'da measure
useLayoutEffect(() => {
  if (ref.current) setSize(ref.current.getBoundingClientRect());
});

// ✅ Browser API — efficient
useLayoutEffect(() => {
  if (!ref.current) return;
  const observer = new ResizeObserver(([entry]) => {
    setSize(entry.contentRect);
  });
  observer.observe(ref.current);
  return () => observer.disconnect();
}, []);
```

**Profiling — DevTools Performance tab:**

```
1. DevTools → Performance tab
2. Record bosish
3. Foydalanish (click, scroll)
4. Stop
5. "Long Tasks" (50ms+) qidirish
6. "useLayoutEffect" yoki layout phase'ni topish
```

Cross-ref [`34-profiling.md`](34-profiling.md) — DevTools Profiler chuqur.

<details>
<summary><strong>Under the Hood</strong></summary>

**Browser rendering pipeline:**

```
JavaScript → Style → Layout → Paint → Composite
   ↓           ↓        ↓        ↓         ↓
  V8       Style    Geometriya  Piksel    GPU
           recalc   (reflow)    chizish   layer
```

JavaScript bloklasa — keyingi qadamlar kechikadi. Pipeline har frame'da takrorlanadi (60Hz'da ~16.67ms'da bir marta).

`useLayoutEffect`:
- JavaScript stage'da
- Browser paint qila olmaydi JavaScript tugaguncha

**Long Tasks API:**

```ts
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 50) {
      console.warn('Long task:', entry);
    }
  }
});

observer.observe({ type: 'longtask', buffered: true });
```

Browser Long Tasks API 50ms+ task'larni topadi. Production monitoring uchun.

**`requestIdleCallback` — useEffect alternativ:**

```ts
useEffect(() => {
  const id = requestIdleCallback(() => {
    // Browser idle vaqtda chaqiriladi
    expensiveWork();
  }, { timeout: 1000 });
  
  return () => cancelIdleCallback(id);
});
```

`requestIdleCallback` — browser bo'sh vaqtda chaqiriladi. Browser support: Chrome 47+ (2015-12), Firefox 55+ (2017-08), **Safari 16.4+** (2023-03) — `requestIdleCallback`'ni qo'shdi (eski Safari versiyalarda yo'q edi, shu sababli React MessageChannel fallback ishlatadi — cross-ref [`05-scheduler-lanes.md`](05-scheduler-lanes.md)).

**Source citation:**

- Performance pipeline — Chrome DevTools Performance docs
- Long Tasks API — W3C spec
- `requestIdleCallback` — W3C Cooperative Scheduling spec

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Layout thrashing va yechimi:**

```tsx
// ❌ Layout thrashing — har item uchun read+write
function ScalingItems({ items }: { items: { id: string; text: string }[] }) {
  const containerRef = useRef<HTMLDivElement>(null);
  
  useLayoutEffect(() => {
    if (!containerRef.current) return;
    
    const elements = containerRef.current.querySelectorAll<HTMLElement>('.item');
    elements.forEach(el => {
      const w = el.offsetWidth;     // Read (FSL)
      el.style.width = `${w + 10}px`; // Write
      // Har iteration FSL → 100 element = 100 FSL
    });
  }, [items]);
  
  return (
    <div ref={containerRef}>
      {items.map(item => <div key={item.id} className="item">{item.text}</div>)}
    </div>
  );
}

// ✅ Read-then-write — bir FSL
function ScalingItemsOptimized({ items }: { items: { id: string; text: string }[] }) {
  const containerRef = useRef<HTMLDivElement>(null);
  
  useLayoutEffect(() => {
    if (!containerRef.current) return;
    
    const elements = Array.from(
      containerRef.current.querySelectorAll<HTMLElement>('.item')
    );
    
    // Read all (bir FSL)
    const widths = elements.map(el => el.offsetWidth);
    
    // Write all
    elements.forEach((el, i) => {
      el.style.width = `${widths[i] + 10}px`;
    });
  }, [items]);
  
  return (
    <div ref={containerRef}>
      {items.map(item => <div key={item.id} className="item">{item.text}</div>)}
    </div>
  );
}
```

**Misol 2 — Heavy work — `requestAnimationFrame`:**

```tsx
function HeavyAnimation({ active }: { active: boolean }) {
  const ref = useRef<HTMLDivElement>(null);
  
  useLayoutEffect(() => {
    if (!active || !ref.current) return;
    
    const node = ref.current;
    let frameId: number;
    let progress = 0;
    
    const animate = () => {
      progress += 0.01;
      node.style.transform = `translateX(${progress * 100}px)`;
      
      if (progress < 1) {
        frameId = requestAnimationFrame(animate);
      }
    };
    
    frameId = requestAnimationFrame(animate);
    
    return () => cancelAnimationFrame(frameId);
  }, [active]);
  
  return <div ref={ref}>Animated</div>;
}
```

**Misol 3 — `ResizeObserver` polling o'rniga:**

```tsx
// ❌ Polling — har 100ms layout calc
function BadResize() {
  const [size, setSize] = useState({ w: 0, h: 0 });
  const ref = useRef<HTMLDivElement>(null);
  
  useLayoutEffect(() => {
    const interval = setInterval(() => {
      if (ref.current) {
        const rect = ref.current.getBoundingClientRect();
        setSize({ w: rect.width, h: rect.height });
        // Har 100ms FSL — wasted work
      }
    }, 100);
    
    return () => clearInterval(interval);
  }, []);
  
  return <div ref={ref}>{size.w}x{size.h}</div>;
}

// ✅ ResizeObserver — efficient
function GoodResize() {
  const [size, setSize] = useState({ w: 0, h: 0 });
  const ref = useRef<HTMLDivElement>(null);
  
  useLayoutEffect(() => {
    if (!ref.current) return;
    
    const observer = new ResizeObserver(([entry]) => {
      setSize({ w: entry.contentRect.width, h: entry.contentRect.height });
    });
    
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);
  
  return <div ref={ref}>{size.w}x{size.h}</div>;
}
```

**Misol 4 — Conditional `useLayoutEffect`:**

```tsx
// useLayoutEffect kerak emas bo'lsa — useEffect afzal
function MaybeFlicker({ visible }: { visible: boolean }) {
  // Visible bo'lganda DOM measurement → useLayoutEffect
  // Visible bo'lmaganda — measure ham bekor → effect skip
  
  const ref = useRef<HTMLDivElement>(null);
  const [width, setWidth] = useState(0);
  
  useLayoutEffect(() => {
    if (!visible) return;
    
    if (ref.current) {
      setWidth(ref.current.offsetWidth);
    }
  }, [visible]);
  
  return visible ? <div ref={ref}>Width: {width}</div> : null;
}
```

</details>

---

## SSR Cheklov — `useLayoutEffect` Server'da

### Nazariya

Server-Side Rendering (SSR) — React component'larni server'da HTML'ga render qilish (cross-ref [`06-hydration.md`](06-hydration.md)). Server'da:

- DOM yo'q (`document`, `window` mavjud emas — Node.js)
- Browser API'lar yo'q (`getBoundingClientRect`, `offsetWidth`, focus)
- Layout calculation yo'q

`useLayoutEffect` — server'da chaqirilmaydi (effect'lar faqat client'da). Lekin React **warning** beradi:

```
Warning: useLayoutEffect does nothing on the server, because its effect cannot
be encoded into the server renderer's output format. This will lead to a
mismatch between the initial, non-hydrated UI and the intended UI. To avoid
this, useLayoutEffect should only be used in components that render exclusively
on the client. See https://reactjs.org/link/uselayouteffect-ssr for common
fixes.
```

**Sabab:**

`useLayoutEffect` ko'pincha DOM measurement qilib, state'ni yangilaydi. Server'da effect chaqirilmasa — SSR HTML state'ning boshlang'ich qiymatini ishlatadi (e.g., `width = 0`). Client'ning initial hydration render'i ham shu boshlang'ich qiymatni ishlatadi (effect hali ishlamagan), demak server HTML va client initial render **mos keladi** — DOM hydration mismatch yo'q. Mismatch keyin yuzaga keladi: hydration tugagach `useLayoutEffect` ishlaydi → yangi state (`width = 150`) → re-render. Foydalanuvchi `0` dan `150` ga o'tishni ko'radi — bu **flicker**. React bu holatda hydration error emas, balki `useLayoutEffect`'ga oid alohida warning beradi (yuqoridagi xabar): server output "intended UI" bilan mos kelmaydi.

**Misol — mismatch potensial:**

```tsx
function MeasuredBox() {
  const ref = useRef<HTMLDivElement>(null);
  const [width, setWidth] = useState(0);
  
  useLayoutEffect(() => {
    if (ref.current) setWidth(ref.current.offsetWidth);
  }, []);
  
  return <div ref={ref}>Width: {width}</div>;
}

// Server HTML: <div>Width: 0</div>
// Client initial hydration render: <div>Width: 0</div> — server bilan MOS (mismatch yo'q)
// Hydration tugagach: useLayoutEffect → setWidth(150) → re-render
// Yangi render: <div>Width: 150</div> — foydalanuvchi 0 → 150 o'zgarishini ko'radi (flicker)
```

**Yechim 1 — Client-only render:**

`useLayoutEffect` faqat client'da kerak bo'lsa — komponentni client-only qilish:

```tsx
function ClientOnly({ children }: { children: React.ReactNode }) {
  const [mounted, setMounted] = useState(false);
  
  useEffect(() => setMounted(true), []);
  
  if (!mounted) return null;
  
  return <>{children}</>;
}

// Usage:
<ClientOnly>
  <MeasuredBox />
</ClientOnly>
```

**Yechim 2 — `useEffect` o'rniga:**

Agar visual flicker muhim emas — `useEffect`'ga o'tish. Server'da chaqirilmaydi, lekin warning ham yo'q (`useEffect` SSR safe).

**Yechim 3 — `useIsomorphicLayoutEffect` workaround:**

Library'lar (e.g., chakra-ui, mantine) `useIsomorphicLayoutEffect` ishlatadi: client'da `useLayoutEffect`, server'da `useEffect`:

```tsx
const useIsomorphicLayoutEffect =
  typeof window !== 'undefined' ? useLayoutEffect : useEffect;
```

Keyingi section'da batafsil.

**Yechim 4 — Next.js / Remix `'use client'`:**

R19 RSC (React Server Components) bilan `'use client'` directive komponent'ni client-only qiladi:

```tsx
'use client';

import { useLayoutEffect } from 'react';

function MeasuredBox() {
  // ... useLayoutEffect ...
}
```

`'use client'` — komponent server'da render qilinmaydi (faqat client). Warning yo'q.

**`useLayoutEffect` SSR warning va haqiqiy hydration mismatch — farq:**

Yuqoridagi `useLayoutEffect` scenariysida DOM **hydration mismatch yo'q** — server HTML (`Width: 0`) va client'ning initial hydration render'i (`Width: 0`) mos keladi. React beradigan warning bu yerda `useLayoutEffect`'ga xos ("does nothing on the server") — DOM mismatch warning emas. Natija — post-hydration flicker.

Haqiqiy hydration mismatch boshqa narsa: server HTML va client'ning **birinchi** render'i bir-biriga mos kelmasligi (masalan `Date.now()`, `Math.random()` yoki `typeof window` ga bog'liq render). Bunday holatda React (R18+) DOM farqini topib, ogohlantirish beradi va shu subtree'ni client'da qaytadan render qilib tuzatadi (client recovery). Bu mexanizm `useLayoutEffect` flicker'iga emas, render output farqiga tegishli.

<details>
<summary><strong>Under the Hood</strong></summary>

**React server renderer:**

`renderToString` (sync) yoki `renderToReadableStream` (Web Streams) — React tree'ni HTML string'ga aylantiradi. Bu jarayonda:

- Component function chaqiriladi
- `useState` initial value hisoblanadi
- `useEffect`/`useLayoutEffect` callback'lari **chaqirilmaydi**
- DOM API'lar mavjud emas (`document` undefined)

```ts
// Server (Node.js)
import { renderToString } from 'react-dom/server';

const html = renderToString(<App />);
// html = '<div>...</div>'

// Effect callback'lari chaqirilmadi
// State faqat initial value
```

**Warning implementation:**

```ts
// React internal (soddalashtirilgan)
function mountLayoutEffect(create, deps) {
  if (__DEV__ && typeof window === 'undefined') {
    // Server'da — warning
    console.warn(
      'Warning: useLayoutEffect does nothing on the server...'
    );
  }
  
  // Mount logic
}
```

Warning faqat development'da. Production'da silent (lekin mismatch hali ham yuzaga keladi).

**`useId` — SSR-safe ID:**

`useId` (R18+) — SSR'da bir xil ID server'da va client'da:

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

`useId` `useLayoutEffect` o'rniga ID generation uchun — SSR mismatch yo'q.

**Source citation:**

- `useLayoutEffect` SSR warning — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`
- React server renderer — facebook/react `packages/react-dom/src/server/`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — SSR warning:**

```tsx
// SSR app (Next.js, Remix, etc.)
function MeasuredText({ text }: { text: string }) {
  const ref = useRef<HTMLSpanElement>(null);
  const [width, setWidth] = useState(0);
  
  useLayoutEffect(() => {  // ⚠️ Server'da warning
    if (ref.current) setWidth(ref.current.offsetWidth);
  }, [text]);
  
  return (
    <div>
      <span ref={ref}>{text}</span>
      <p>Width: {width}px</p>
    </div>
  );
}

// Server: <p>Width: 0px</p>
// Client initial hydration render: <p>Width: 0px</p> — mos (mismatch yo'q)
// Hydration tugagach effect ishlaydi → <p>Width: 150px</p> (post-hydration flicker)
// React useLayoutEffect SSR warning'i (DOM mismatch emas)
```

**Misol 2 — ClientOnly wrapper:**

```tsx
function ClientOnly({ children, fallback = null }: {
  children: React.ReactNode;
  fallback?: React.ReactNode;
}) {
  const [mounted, setMounted] = useState(false);
  
  useEffect(() => {
    setMounted(true);
  }, []);
  
  if (!mounted) return <>{fallback}</>;
  
  return <>{children}</>;
}

// Usage:
function Page() {
  return (
    <div>
      <h1>Server-rendered title</h1>
      <ClientOnly fallback={<div style={{ height: 100 }} />}>
        <MeasuredBox />  {/* useLayoutEffect ichida — client-only */}
      </ClientOnly>
    </div>
  );
}
```

**Misol 3 — Next.js `'use client'`:**

```tsx
// app/components/MeasuredBox.tsx
'use client';  // Bu komponent client'da

import { useLayoutEffect, useRef, useState } from 'react';

export default function MeasuredBox() {
  const ref = useRef<HTMLDivElement>(null);
  const [width, setWidth] = useState(0);
  
  useLayoutEffect(() => {
    if (ref.current) setWidth(ref.current.offsetWidth);
  }, []);
  
  return <div ref={ref}>Width: {width}</div>;
}

// app/page.tsx (server component)
import MeasuredBox from './components/MeasuredBox';

export default function Page() {
  return (
    <div>
      <h1>Page title</h1>
      <MeasuredBox />  {/* Client'da render */}
    </div>
  );
}
```

**Misol 4 — `useId` SSR-safe:**

```tsx
// ❌ Math.random — server va client farq
function BadField({ label }: { label: string }) {
  const [id] = useState(`field-${Math.random()}`);  // SSR mismatch
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}

// ✅ useId — SSR safe
function GoodField({ label }: { label: string }) {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}

// Server: id = ":r0:"
// Client: id = ":r0:" (bir xil)
// Mismatch yo'q
```

</details>

---

## `useIsomorphicLayoutEffect` Workaround

### Nazariya

`useIsomorphicLayoutEffect` — community pattern (rasmiy React API emas). Library author'lar SSR warning'ni oldini olish uchun ishlatadi:

```tsx
import { useEffect, useLayoutEffect } from 'react';

const useIsomorphicLayoutEffect =
  typeof window !== 'undefined' ? useLayoutEffect : useEffect;

export { useIsomorphicLayoutEffect };
```

**Logika:**

- Client (`window` mavjud) → `useLayoutEffect` (paint dan oldin)
- Server (`window` undefined) → `useEffect` (server'da chaqirilmaydi, warning yo'q)

Library code'da xavfsiz: agar SSR'da render qilinsa — warning yo'q. Client'da `useLayoutEffect` semantikasi.

**Trade-off:**

- **Foyda:** Library SSR safe (warning yo'q)
- **Kamchilik:** Server'da effect logic ishlamaydi → SSR HTML va client'da mismatch potensial

Application code'da ishlatish — savol. Agar effect SSR uchun kerak bo'lmasa OK. Aks holda mismatch bug.

**Mashhur library'larda:**

- `framer-motion` — `useIsomorphicLayoutEffect`
- `react-use` — `useIsomorphicLayoutEffect`
- `chakra-ui` — `useSafeLayoutEffect`
- `mantine` — `useIsomorphicEffect`

Hammasi bir xil pattern. Nom farq, mexanizm bir xil.

**Custom hook implementation:**

```tsx
import { useEffect, useLayoutEffect } from 'react';

const canUseDOM = typeof window !== 'undefined' &&
  typeof window.document !== 'undefined' &&
  typeof window.document.createElement !== 'undefined';

const useIsomorphicLayoutEffect = canUseDOM ? useLayoutEffect : useEffect;
```

`canUseDOM` — `window` + `document` mavjudligi. Edge case'larda ishonchli (e.g., Web Workers, jsdom test environment).

**R18 fix — `useInsertionEffect` SSR safe:**

R18'da `useInsertionEffect` SSR'da warning bermaydi. Library author'lar uchun yangi alternativ. Lekin `useInsertionEffect` — CSS-in-JS uchun, application code uchun emas.

**TypeScript types:**

```ts
import type { useEffect, useLayoutEffect } from 'react';

type EffectHook = typeof useEffect | typeof useLayoutEffect;

const useIsomorphicLayoutEffect: EffectHook =
  typeof window !== 'undefined' ? useLayoutEffect : useEffect;
```

Bir xil signature — `useEffect` va `useLayoutEffect` API identik.

<details>
<summary><strong>Under the Hood</strong></summary>

**React conditional hook calls — Rules of Hooks buzilishi?**

`useIsomorphicLayoutEffect` quyidagicha:

```ts
const useIsomorphicLayoutEffect =
  typeof window !== 'undefined' ? useLayoutEffect : useEffect;
```

Bu — module-level conditional. Render time'da `useIsomorphicLayoutEffect` har doim **bir xil** hook (`useLayoutEffect` client'da, `useEffect` server'da). Rules of Hooks buzilmaydi — bir render davomida bir xil hook chaqiriladi.

Render orasida hook o'zgarishi — ya'ni, "Render 1: useEffect, Render 2: useLayoutEffect" — buzilish bo'lardi. Lekin server vs client farq bir xil session'da bo'lmaydi (server render bir process, client render boshqa process).

**Edge case — Hydration mismatch:**

`useIsomorphicLayoutEffect` server'da `useEffect`'ga teng (chaqirilmaydi), client'da `useLayoutEffect`'ga teng (chaqiriladi). Initial render output ikkala tomonda bir xil bo'lgani uchun (effect hali ishlamagan) — DOM hydration mismatch yo'q. Agar effect state'ni yangilasa, o'zgarish hydration'dan keyin yuzaga keladi.

Client'da `useLayoutEffect` paint'dan oldin ishlagani uchun bu state o'zgarishi flicker bermaydi (user faqat oxirgi natijani ko'radi). Agar `useEffect` ishlatilsa — o'zgarish paint'dan keyin → flicker. Shu sababli library author'lar client'da `useLayoutEffect` semantikasini saqlash uchun bu pattern'ni ishlatadi (cross-ref [`06-hydration.md`](06-hydration.md)).

**`canUseDOM` ishonchli check:**

```ts
// Eng oddiy
typeof window !== 'undefined'

// Robustroq (jsdom va Web Workers uchun)
typeof window !== 'undefined' &&
  typeof window.document !== 'undefined' &&
  typeof window.document.createElement !== 'undefined'
```

jsdom (Jest test environment) `window`'ni ta'minlaydi — lekin ko'pincha `useLayoutEffect` test'larda `useEffect` deb qabul qilinadi.

**Source citation:**

- `useIsomorphicLayoutEffect` pattern — community (Donavon West, Sebastian Markbåge community discussions)
- `canUseDOM` — facebook/react `packages/shared/canUseDOM.js` (legacy)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Custom hook:**

```tsx
// hooks/useIsomorphicLayoutEffect.ts
import { useEffect, useLayoutEffect } from 'react';

const useIsomorphicLayoutEffect =
  typeof window !== 'undefined' ? useLayoutEffect : useEffect;

export default useIsomorphicLayoutEffect;
```

**Misol 2 — Library code'da ishlatish:**

```tsx
import useIsomorphicLayoutEffect from './useIsomorphicLayoutEffect';

function useElementSize<T extends HTMLElement>(): [
  React.RefObject<T>,
  { width: number; height: number }
] {
  const ref = useRef<T>(null);
  const [size, setSize] = useState({ width: 0, height: 0 });
  
  useIsomorphicLayoutEffect(() => {
    if (!ref.current) return;
    
    const update = () => {
      if (!ref.current) return;
      const rect = ref.current.getBoundingClientRect();
      setSize({ width: rect.width, height: rect.height });
    };
    
    update();
    window.addEventListener('resize', update);
    
    return () => window.removeEventListener('resize', update);
  }, []);
  
  return [ref, size];
}

// SSR'da warning yo'q
// Client'da useLayoutEffect semantikasi (no flicker)
```

**Misol 3 — `canUseDOM` robust check:**

```tsx
const canUseDOM = !!(
  typeof window !== 'undefined' &&
  window.document &&
  window.document.createElement
);

const useIsomorphicLayoutEffect = canUseDOM ? useLayoutEffect : useEffect;
```

**Misol 4 — Library wrapping:**

```tsx
// Hypothetical UI library
import useIsomorphicLayoutEffect from './hooks/useIsomorphicLayoutEffect';

export function Tooltip({ children, content }: TooltipProps) {
  const [position, setPosition] = useState<Position | null>(null);
  
  useIsomorphicLayoutEffect(() => {
    // SSR'da skip, client'da paint dan oldin
    calculatePosition();
  }, [children]);
  
  return <>...</>;
}

// Consumer SSR app'da ishlata oladi — warning yo'q
```

**Misol 5 — TS types:**

```tsx
import { useEffect, useLayoutEffect } from 'react';
import type { EffectCallback, DependencyList } from 'react';

type IsomorphicLayoutEffect = (
  effect: EffectCallback,
  deps?: DependencyList
) => void;

const useIsomorphicLayoutEffect: IsomorphicLayoutEffect =
  typeof window !== 'undefined' ? useLayoutEffect : useEffect;

export default useIsomorphicLayoutEffect;
```

</details>

---

## `useInsertionEffect` (R18) — CSS-in-JS Uchun

### Nazariya

`useInsertionEffect` — R18'da qo'shilgan hook. **Faqat CSS-in-JS library'lar uchun mo'ljallangan** (styled-components, emotion, stitches, vanilla-extract). Application code'da ishlatilmaydi.

**Timing:**

`useInsertionEffect` Commit Phase'ning Mutation sub-phase'ida, lekin DOM mutation'dan **oldin** chaqiriladi. Demak `useLayoutEffect`'dan ham oldin:

```
Commit Phase tartib:
1. Before Mutation
2. Mutation sub-phase:
   ├─ useInsertionEffect (R18+) ← bu yerda
   ├─ Refs detach (old)
   └─ DOM Mutation (insert/update/remove)
3. Layout sub-phase:
   ├─ Refs attach (new)
   ├─ useLayoutEffect (cleanup → setup)
   └─ componentDidMount/Update
```

**Sabab — CSS injection layout calculation'dan oldin:**

CSS-in-JS library'lar dynamic CSS yaratadi va `<style>` tag'ga inject qiladi. Agar bu DOM mutation'dan keyin bo'lsa:

```
1. DOM mutation: <div className="dynamic-class-abc"> qo'shildi
2. (Browser layout calc — class yo'q, default style)
3. <style>.dynamic-class-abc { color: red; }</style> inject
4. (Browser layout recalc — yangi style)
```

Layout 2 marta hisoblanadi → performance muammo. `useInsertionEffect` CSS injection'ni DOM mutation'dan **oldin** qiladi:

```
1. useInsertionEffect: <style>.dynamic-class-abc { color: red; }</style> inject
2. DOM mutation: <div className="dynamic-class-abc"> qo'shildi
3. (Browser layout calc — style mavjud, bir marta)
```

Bir layout pass — efficient.

**Cheklov — DOM ga kirish yo'q:**

`useInsertionEffect` ichida ref.current null (DOM hali yangilanmagan):

```tsx
function CssInJsComponent() {
  const ref = useRef<HTMLDivElement>(null);
  
  useInsertionEffect(() => {
    console.log('Insertion effect:', ref.current);  // null
    
    // ❌ DOM o'qish — hali yangilanmagan
    // ✅ <style> inject mumkin
  });
  
  useLayoutEffect(() => {
    console.log('Layout effect:', ref.current);  // <div>
    
    // ✅ DOM o'qish va yozish
  });
  
  return <div ref={ref}>Test</div>;
}
```

**Application code'da ishlatish — anti-pattern:**

```tsx
// ❌ Anti-pattern
function MyComponent() {
  useInsertionEffect(() => {
    console.log('User action handler...');  // ❌ Bu uchun useEffect yoki useLayoutEffect
  });
  
  return null;
}
```

React docs aniq aytadi: `useInsertionEffect` — CSS-in-JS library author'lar uchun. Boshqa joyda ishlatish — kelajakdagi React versiyalarida buzilishi mumkin (semantics CSS-in-JS uchun optimize qilingan).

**`useInsertionEffect` SSR-safe:**

`useLayoutEffect`'dan farqli, `useInsertionEffect` server'da warning bermaydi. Bu CSS-in-JS library'lar SSR'da CSS extract qilish uchun.

**Library misollari:**

- **emotion** — `useInsertionEffect` ichida CSS rule injection
- **styled-components v6** — `useInsertionEffect` qo'llab-quvvatlashi bilan chiqdi
- **stitches** — yangi versiyalarda
- **vanilla-extract** — build-time CSS, runtime hook kerak emas

<details>
<summary><strong>Under the Hood</strong></summary>

**`mountInsertionEffect` implementation:**

```ts
function mountInsertionEffect(
  create: () => () => void,
  deps: Array<unknown> | null
): void {
  return mountEffectImpl(
    UpdateEffect,
    HookInsertion,  // ← HookLayout va HookPassive emas
    create,
    deps
  );
}
```

`HookInsertion` flag — Mutation sub-phase'da chaqirilishini belgilaydi.

**`commitMutationEffects` ichida:**

```ts
function commitMutationEffects(root, finishedWork) {
  // 1. useInsertionEffect chaqirish (insertion qilishdan oldin)
  commitHookEffectListMount(
    HookInsertion | HookHasEffect,
    finishedWork
  );
  
  // 2. Eski refs detach (old)
  // ...
  
  // 3. DOM mutation (insert/update/remove)
  // ...
}
```

`useInsertionEffect` — Mutation sub-phase boshida, DOM mutation'dan oldin. Yangi refs esa keyingi (Layout) sub-phase'da attach qilinadi.

**Nima uchun "insertion":**

Nom — CSS rule "insertion"'ga ishora. CSSStyleSheet API'da `insertRule` method bor:

```ts
const sheet = new CSSStyleSheet();
sheet.insertRule('.dynamic-class { color: red; }', 0);
document.adoptedStyleSheets = [sheet];
```

CSS-in-JS library'lar shu kabi API ishlatadi (yoki `<style>` tag'ga `textContent` qo'shadi). `useInsertionEffect` — bu insertion uchun optimal vaqt.

**Source citation:**

- `useInsertionEffect` — react.dev "useInsertionEffect" reference; React 18 Working Group (reactwg/react-18) Library Upgrade Guide muhokamasi
- emotion implementation — emotion-js/emotion repo

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — CSS-in-JS hook (simplified):**

```tsx
// CSS-in-JS library author code
function useDynamicCSS(className: string, styles: string) {
  useInsertionEffect(() => {
    // 1. <style> tag'ni topish yoki yaratish
    let styleTag = document.querySelector<HTMLStyleElement>(
      `style[data-class="${className}"]`
    );
    
    if (!styleTag) {
      styleTag = document.createElement('style');
      styleTag.dataset.class = className;
      document.head.appendChild(styleTag);
    }
    
    // 2. CSS rule yozish
    styleTag.textContent = `.${className} { ${styles} }`;
    
    return () => {
      styleTag?.remove();
    };
  }, [className, styles]);
}

// Usage (library consumer perspective)
function StyledBox() {
  useDynamicCSS('my-box', 'background: red; padding: 16px;');
  return <div className="my-box">Styled</div>;
}

// Lifecycle:
// 1. Render: <div className="my-box">
// 2. Commit Mutation start
// 3. useInsertionEffect: <style>.my-box { background: red; padding: 16px; }</style> inject
// 4. DOM mutation: <div> qo'shildi (style allaqachon mavjud)
// 5. Browser layout calc — bir marta, to'g'ri style bilan
```

**Misol 2 — `useInsertionEffect` ichida DOM null:**

```tsx
function DOMAccessTest() {
  const ref = useRef<HTMLDivElement>(null);
  
  useInsertionEffect(() => {
    console.log('Insertion ref:', ref.current);
    // null — DOM hali yangilanmagan
  });
  
  useLayoutEffect(() => {
    console.log('Layout ref:', ref.current);
    // <div> — DOM yangilangan
  });
  
  return <div ref={ref}>Test</div>;
}

// Output:
// Insertion ref: null
// Layout ref: <div>...</div>
```

**Misol 3 — Application code anti-pattern:**

```tsx
// ❌ ANTI-PATTERN
function DontDoThis() {
  useInsertionEffect(() => {
    fetch('/api/log', {  // ❌ Bu uchun useEffect
      method: 'POST',
      body: 'tracking'
    });
  });
  
  return null;
}

// ✅ TO'G'RI
function DoThis() {
  useEffect(() => {
    fetch('/api/log', {
      method: 'POST',
      body: 'tracking'
    });
  });
  
  return null;
}
```

**Misol 4 — emotion-style hook:**

```tsx
// Hypothetical CSS-in-JS library
// Ichida hook chaqirgani uchun bu funksiya o'zi ham hook — Rules of Hooks
// amal qiladi: nomi `use` bilan boshlanadi, component top level'ida chaqiriladi.
function useCss(styles: string): string {
  const hash = simpleHash(styles);
  const className = `css-${hash}`;
  
  // useInsertionEffect'ni ichida chaqiradi (hook return qilmaydi, chaqiradi)
  useDynamicCSS(className, styles);
  
  return className;
}

function StyledButton() {
  const className = useCss(`
    background: blue;
    color: white;
    padding: 8px 16px;
    border-radius: 4px;
  `);
  
  return <button className={className}>Click</button>;
}
```

</details>

---

## Decision Guide — `useEffect` / `useLayoutEffect` / `useInsertionEffect`

### Nazariya

Quyidagi savollar bilan to'g'ri hook tanlash:

```
1. Bu CSS-in-JS library kodi'mi (rule injection)?
   → ha → useInsertionEffect (R18+)
   → yo'q → 2-savol

2. Visible visual o'zgarish kerakmi (DOM measurement,
   tooltip, scroll, focus)?
   → ha → useLayoutEffect
   → yo'q → useEffect (default tanlov)

3. SSR'da ishlash kerakmi?
   → ha + useLayoutEffect kerak → useIsomorphicLayoutEffect (library)
                                   yoki ClientOnly wrapper (app)
   → ha + useEffect → useEffect SSR safe
```

**Default tanlov — `useEffect`:**

Aksariyat holatlarda `useEffect` to'g'ri tanlov (React docs `useLayoutEffect`'ni `useEffect`'dan kamroq ishlatilishini ta'kidlaydi). `useLayoutEffect` faqat:

- DOM measurement → state update → re-render kerak
- Tooltip/popover/modal positionlash
- Scroll restoration
- Focus management
- Layout-dependent calculation (equal heights, dynamic sizing)

`useEffect` ishlaydigan joylar:

- Network requests (fetch, GraphQL)
- Subscriptions (WebSocket, EventEmitter)
- Timers (setInterval, setTimeout)
- Logging, analytics
- Document title, browser history
- LocalStorage/sessionStorage sync
- Animation triggers (CSS class toggle, then transition)

**`useInsertionEffect` deyarli hech qachon ishlatilmaydi:**

Application developer'lar uchun `useInsertionEffect` kerak emas. Faqat CSS-in-JS library author'lar uchun. Bu kursda mention qilingani — awareness uchun.

**Decision matrix:**

| Use case | Hook |
|----------|------|
| Network fetch | `useEffect` |
| WebSocket subscribe | `useEffect` |
| Timer (interval, timeout) | `useEffect` |
| Logging | `useEffect` |
| DOM measure → state | `useLayoutEffect` |
| Tooltip position | `useLayoutEffect` |
| Scroll restoration | `useLayoutEffect` |
| Focus modal input | `useLayoutEffect` |
| Equal column heights | `useLayoutEffect` |
| Body scroll lock | `useLayoutEffect` |
| Document title | `useEffect` |
| Animation start | `useEffect` (yoki `requestAnimationFrame`) |
| CSS-in-JS rule inject | `useInsertionEffect` (library) |

**Performance triage:**

Ko'p `useLayoutEffect` paint'ni sekinlashtiradi. Profiling:

1. DevTools Performance tab → "Long Tasks" qidirish
2. `useLayoutEffect` callback duration o'lchash
3. Layout phase frame budget'ga sezilarli ulush qo'shsa → optimize qilish (read-then-write, ResizeObserver, useEffect ga ko'chirish)

<details>
<summary><strong>Under the Hood</strong></summary>

**Hooks priority — schedule timing:**

```
Schedule timing:
1. Render Phase — concurrent (interruptible)
2. Commit Phase — sync (atomic)
   - Mutation sub-phase:
     - useInsertionEffect (R18+) — Mutation start
     - Refs detach (old)
     - DOM mutation
   - Layout sub-phase:
     - Refs attach (new)
     - useLayoutEffect
3. Browser Paint
4. Passive Effects Phase — async (MessageChannel)
   - useEffect
```

Schedule order: Insertion → Layout → Paint → Passive. Ishlash tartibi bilan bog'liq.

**Yangi hooks (R18+, R19+):**

| Hook | R | Use case |
|------|---|----------|
| `useEffect` | R16.8 | Side effect default |
| `useLayoutEffect` | R16.8 | Sync DOM read/write |
| `useInsertionEffect` | R18 | CSS-in-JS |
| `useId` | R18 | SSR-safe ID |
| `useSyncExternalStore` | R18 | External store (tearing-safe) |
| `useTransition` | R18 | Non-urgent update |
| `useDeferredValue` | R18 | Defer expensive value |
| `use()` | R19 | Promise / context (conditional) |
| `useFormStatus` | R19 | Form pending state |
| `useActionState` | R19 | Server action state |
| `useOptimistic` | R19 | Optimistic update |

`useEffect`/`useLayoutEffect`/`useInsertionEffect` triada — synchronization mexanizmlari. Boshqa hooks — boshqa concerns.

**Source citation:**

- React docs "useEffect", "useLayoutEffect", "useInsertionEffect" — react.dev
- R18 RFC — reactjs/rfcs

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Wrong vs Right hook tanlovi:**

```tsx
// ❌ useLayoutEffect network uchun
function UserCard({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);
  
  useLayoutEffect(() => {  // ❌ Paint bloklash, kerak emas
    fetch(`/api/users/${userId}`).then(r => r.json()).then(setUser);
  }, [userId]);
}

// ✅ useEffect
function UserCard({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetch(`/api/users/${userId}`).then(r => r.json()).then(setUser);
  }, [userId]);
}
```

**Misol 2 — `useEffect` flicker bo'lsa → `useLayoutEffect`:**

```tsx
// ❌ Flicker
function Tooltip({ targetRef }: { targetRef: React.RefObject<HTMLElement> }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    if (targetRef.current) {
      const rect = targetRef.current.getBoundingClientRect();
      setPosition({ x: rect.x, y: rect.y });
    }
  }, [targetRef]);
  
  // Avval (0, 0)'da paint, keyin to'g'ri positionda paint
  return <div style={{ position: 'fixed', left: position.x, top: position.y }} />;
}

// ✅ No flicker
function Tooltip({ targetRef }: { targetRef: React.RefObject<HTMLElement> }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useLayoutEffect(() => {
    if (targetRef.current) {
      const rect = targetRef.current.getBoundingClientRect();
      setPosition({ x: rect.x, y: rect.y });
    }
  }, [targetRef]);
  
  return <div style={{ position: 'fixed', left: position.x, top: position.y }} />;
}
```

**Misol 3 — `useLayoutEffect` SSR + warning:**

```tsx
// ❌ Server'da warning
function Box() {
  const ref = useRef<HTMLDivElement>(null);
  const [w, setW] = useState(0);
  
  useLayoutEffect(() => {
    if (ref.current) setW(ref.current.offsetWidth);
  }, []);
  
  return <div ref={ref}>Width: {w}</div>;
}

// ✅ ClientOnly wrap
function App() {
  return (
    <ClientOnly>
      <Box />
    </ClientOnly>
  );
}

// Yoki ✅ useIsomorphicLayoutEffect (library code'da)
const useIsomorphicLayoutEffect =
  typeof window !== 'undefined' ? useLayoutEffect : useEffect;

function Box() {
  // ...
  useIsomorphicLayoutEffect(() => {
    if (ref.current) setW(ref.current.offsetWidth);
  }, []);
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1 — `useLayoutEffect` ichidagi `setState` ikki cycle

`useLayoutEffect` ichida `setState` chaqirilsa, React darhol qayta render + commit qiladi (paint'dan oldin). Bu performance ta'sir qiladi:

```tsx
useLayoutEffect(() => {
  setState(newValue);  // Paint dan oldin yangi cycle
});

// Lifecycle:
// Render 1 → Commit 1 (Layout: setState) → Render 2 → Commit 2 → Paint
// 2 cycle paint dan oldin
```

Cheksiz loop xavfi:

```tsx
useLayoutEffect(() => {
  setState(value + 1);  // ❌ Har cycle'da yangi setState — infinite loop
});  // Deps yo'q
```

Deps yoki shart bilan break:

```tsx
useLayoutEffect(() => {
  if (value < 10) {
    setState(value + 1);
  }
}, [value]);
```

### Gotcha 2 — Strict Mode 2x cycle paint kechikishi

Strict Mode'da `useLayoutEffect` 2x cycle (setup → cleanup → setup). Development'da effect ichidagi ish ikki marta bajariladi, demak paint kechikishi shunga mos ravishda oshadi:

```tsx
useLayoutEffect(() => {
  // CPU-intensive ish (measurement, hisoblash)
});

// Production: ish bir marta → paint shuncha kechikadi
// Development (Strict Mode): ish ikki marta (cleanup + setup) → paint ko'proq kechikadi
```

Production'da bu yo'q — Strict Mode dev only. Demak Strict Mode'dagi qo'shimcha kechikish development artefakti, real performance o'lchovi emas.

### Gotcha 3 — `useLayoutEffect` ichida portal child measurement

Portal child'lar boshqa DOM tree'da. `useLayoutEffect`'da portal'ning DOM hali yaratilmagan bo'lishi mumkin:

```tsx
function Modal({ children }) {
  return createPortal(
    <div className="modal">{children}</div>,
    document.body
  );
}

function Parent() {
  useLayoutEffect(() => {
    const modalEl = document.querySelector('.modal');
    console.log(modalEl);  // ✅ Portal child Layout phase'da mavjud
  });
  
  return <Modal>Content</Modal>;
}
```

R18'dan boshlab portal Layout effect'lar to'g'ri tartibda ishlaydi (parent'dan oldin child). Lekin DOM order o'zgargan — querySelector kerakli element'ni topa olishini tekshirish kerak.

### Gotcha 4 — `useLayoutEffect` cleanup'da DOM o'qish

Cleanup paytida DOM hali mavjud bo'lishi mumkin (component unmount jarayonida) yoki yo'q (boshqa update'da):

```tsx
useLayoutEffect(() => {
  const ref = elementRef.current;
  
  return () => {
    // ⚠️ ref.current null bo'lishi mumkin (R19+ ref cleanup)
    if (ref) {
      // Closure'da saqlangan ref ishonchli
      ref.style.opacity = '0';
    }
  };
}, []);
```

R19'da ref callback cleanup function qo'shildi (cross-ref [`18-useref.md`](18-useref.md)) — DOM o'chirilganda darrov chaqiriladi.

### Gotcha 5 — `useLayoutEffect` Suspense fallback bilan

Ikki holatni ajratish kerak:

**1. Initial suspend** — component birinchi marta render paytida suspend qilsa (`use(promise)` yoki throw promise), u **hali commit qilinmagan**. Effect setup ham, cleanup ham chaqirilmaydi, chunki effect umuman ro'yxatdan o'tmagan:

```tsx
function UserCard({ userId }: { userId: string }) {
  const user = use(fetchUser(userId));  // R19 — suspend (render phase)
  
  useLayoutEffect(() => {
    console.log('Setup');
    return () => console.log('Cleanup');
  }, [userId]);
  
  return <div>{user.name}</div>;
}

// Birinchi render'da suspend: Setup ham, Cleanup ham chaqirilmaydi
// (component commit bosqichiga yetib bormagan)
```

**2. Re-suspend** — allaqachon ko'rsatilgan (commit qilingan) content qayta suspend qilib, fallback'ga qaytsa. R18'dan boshlab React bu content'ning **Layout effect'larini cleanup qiladi** (destroy), keyin content qaytadan ko'rsatilganda ularni **qaytadan yaratadi**. Bu R18 upgrade guide'da aniq hujjatlangan o'zgarish — komponent library'lar Suspense bilan layout'ni to'g'ri o'lchashi uchun kerak bo'lgan.

```tsx
// Content ko'rsatilgan → re-suspend → fallback:
// 1. useLayoutEffect cleanup chaqiriladi (effect destroy)
// 2. Fallback ko'rsatiladi
// 3. Content qaytib kelganda — useLayoutEffect setup qaytadan chaqiriladi
```

Demak resurs yaratuvchi Layout effect (event listener, subscription) Suspense bilan birga to'g'ri cleanup/re-setup tsiklidan o'tadi.

---

## Common Mistakes

### ❌ Xato 1 — `useEffect` flicker keltirsa ham qoldirish

```tsx
// ❌ Flicker visible
function Box() {
  const ref = useRef<HTMLDivElement>(null);
  const [width, setWidth] = useState(0);
  
  useEffect(() => {
    if (ref.current) setWidth(ref.current.offsetWidth);
  }, []);
  
  return (
    <div ref={ref}>
      <p>Width: {width}px</p>
      <div style={{ width: `${width / 2}px`, background: 'red', height: 20 }} />
    </div>
  );
}

// User avval width=0 paint, keyin width=150 paint (flicker)

// ✅ useLayoutEffect — flicker yo'q
function Box() {
  // ... useLayoutEffect bilan
}
```

### ❌ Xato 2 — `useLayoutEffect` har joyda

```tsx
// ❌ Network uchun useLayoutEffect — paint bloklash
function Component() {
  useLayoutEffect(() => {
    fetch('/api/data').then(r => r.json()).then(setData);  // ❌ Paint dan oldin
  }, []);
}

// ✅ Network — useEffect
function Component() {
  useEffect(() => {
    fetch('/api/data').then(r => r.json()).then(setData);
  }, []);
}
```

### ❌ Xato 3 — Layout thrashing

```tsx
// ❌ Read-write-read pattern
useLayoutEffect(() => {
  items.forEach(item => {
    item.style.height = `${item.offsetHeight + 10}px`;  // ❌ Har iteration FSL
  });
});

// ✅ Read-then-write
useLayoutEffect(() => {
  const heights = items.map(item => item.offsetHeight);
  items.forEach((item, i) => {
    item.style.height = `${heights[i] + 10}px`;
  });
});
```

### ❌ Xato 4 — SSR'da `useLayoutEffect` warning'ni ignore

```tsx
// ❌ Console warning'ni ignore qilish
function MeasuredBox() {
  useLayoutEffect(() => {  // SSR warning
    // ...
  }, []);
}

// ✅ ClientOnly wrap (app code)
function App() {
  return (
    <ClientOnly>
      <MeasuredBox />
    </ClientOnly>
  );
}

// Yoki ✅ Next.js 'use client' directive
// 'use client';
// function MeasuredBox() {...}
```

### ❌ Xato 5 — `useInsertionEffect` application code'da

```tsx
// ❌ ANTI-PATTERN
function MyComponent() {
  useInsertionEffect(() => {
    document.title = 'My Page';  // ❌ Bu useEffect uchun
  }, []);
}

// ✅ useEffect
function MyComponent() {
  useEffect(() => {
    document.title = 'My Page';
  }, []);
}

// useInsertionEffect — faqat CSS-in-JS library author'lar uchun
```

---

## Amaliy Mashqlar

### Mashq 1 — `useElementSize` Hook (Oson)

`ResizeObserver` bilan element o'lchamini track qiluvchi hook yozing.

```tsx
function useElementSize<T extends HTMLElement>(): [
  React.RefObject<T>,
  { width: number; height: number }
] {
  // Implement
}

function Card() {
  const [ref, { width, height }] = useElementSize<HTMLDivElement>();
  return <div ref={ref}>{width} × {height}</div>;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useElementSize<T extends HTMLElement>(): [
  React.RefObject<T>,
  { width: number; height: number }
] {
  const ref = useRef<T>(null);
  const [size, setSize] = useState({ width: 0, height: 0 });
  
  useLayoutEffect(() => {
    if (!ref.current) return;
    
    const observer = new ResizeObserver(([entry]) => {
      const { width, height } = entry.contentRect;
      setSize({ width, height });
    });
    
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);
  
  return [ref, size];
}
```

**Tushuntirish:**

- `useLayoutEffect` — initial measurement paint'dan oldin (no flicker)
- `ResizeObserver` — element o'lchami o'zgarganda avtomatik
- Cleanup'da `disconnect()` — memory leak oldini olish
- Generic `<T>` — har xil HTML element'lar uchun

`useEffect` bilan bo'lsa: initial render'da `0 × 0` paint qilingan, keyin yangi paint. Flicker bor.

</details>

### Mashq 2 — `useScrollPosition` Hook (Oson)

Scroll positionsi (`x`, `y`) ni track qiluvchi hook yozing.

```tsx
function useScrollPosition(): { x: number; y: number } {
  // Implement
}

function ScrollIndicator() {
  const { y } = useScrollPosition();
  return <div>Scrolled: {y}px</div>;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useScrollPosition(): { x: number; y: number } {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useLayoutEffect(() => {
    const update = () => {
      setPosition({ x: window.scrollX, y: window.scrollY });
    };
    
    update();
    window.addEventListener('scroll', update, { passive: true });
    
    return () => window.removeEventListener('scroll', update);
  }, []);
  
  return position;
}
```

**Tushuntirish:**

- `useLayoutEffect` — initial scroll position paint'dan oldin
- `{ passive: true }` — scroll listener performance uchun
- Cleanup'da listener remove

Performance pitfall: har scroll event'da `setState` re-render keltirib chiqaradi. Throttle/debounce qo'shish mumkin (cross-ref [`16-useeffect.md`](16-useeffect.md) "Mashq 1 useDebounce").

</details>

### Mashq 3 — `useFocusTrap` Hook (O'rta)

Modal ichida Tab navigation'ni trap qiluvchi hook yozing. Tab oxirgi element'dan birinchi'ga, Shift+Tab birinchi'dan oxirgi'ga o'tsin.

```tsx
function useFocusTrap<T extends HTMLElement>(active: boolean): React.RefObject<T> {
  // Implement
}

function Modal({ isOpen }: { isOpen: boolean }) {
  const ref = useFocusTrap<HTMLDivElement>(isOpen);
  
  if (!isOpen) return null;
  
  return (
    <div ref={ref} role="dialog">
      <input placeholder="First" />
      <input placeholder="Last" />
      <button>OK</button>
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useFocusTrap<T extends HTMLElement>(active: boolean): React.RefObject<T> {
  const containerRef = useRef<T>(null);
  
  useLayoutEffect(() => {
    if (!active || !containerRef.current) return;
    
    const container = containerRef.current;
    const focusableSelector =
      'button, input, textarea, select, a[href], [tabindex]:not([tabindex="-1"])';
    
    const getFocusable = () =>
      Array.from(container.querySelectorAll<HTMLElement>(focusableSelector));
    
    // Initial focus
    const focusable = getFocusable();
    focusable[0]?.focus();
    
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      
      const elements = getFocusable();
      if (elements.length === 0) return;
      
      const first = elements[0];
      const last = elements[elements.length - 1];
      
      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault();
        last.focus();
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault();
        first.focus();
      }
    };
    
    container.addEventListener('keydown', handleKeyDown);
    
    return () => {
      container.removeEventListener('keydown', handleKeyDown);
    };
  }, [active]);
  
  return containerRef;
}
```

**Tushuntirish:**

- `useLayoutEffect` — initial focus paint'dan oldin (modal ochilishida darrov)
- `focusableSelector` — barcha interactive element'lar
- Tab event listener container darajasida
- Shift+Tab birinchi → oxirgi, Tab oxirgi → birinchi
- Cleanup'da listener remove

R19'dan `inert` attribute (parent darajasida) — focus trap'ning yarmini avtomatik hal qiladi (background komponent'larni inert qilish). Lekin `useFocusTrap` modal ichida Tab cycle uchun hali ham kerak.

</details>

### Mashq 4 — `useTooltipPosition` Hook (O'rta)

Tooltip positionsini hisoblovchi hook yozing. Viewport chegaralarini hisobga olib, tooltip'ni "above" yoki "below" placement'ga o'tkazsin.

```tsx
function useTooltipPosition(
  triggerRef: React.RefObject<HTMLElement>,
  show: boolean
): { x: number; y: number; placement: 'top' | 'bottom' } {
  // Implement
}

function Tooltip({ targetRef, content, show }: {
  targetRef: React.RefObject<HTMLElement>;
  content: string;
  show: boolean;
}) {
  const { x, y, placement } = useTooltipPosition(targetRef, show);
  
  if (!show) return null;
  
  return (
    <div style={{ position: 'fixed', left: x, top: y }}>
      {content} ({placement})
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useTooltipPosition(
  triggerRef: React.RefObject<HTMLElement>,
  show: boolean
): { x: number; y: number; placement: 'top' | 'bottom' } {
  const [position, setPosition] = useState({
    x: 0,
    y: 0,
    placement: 'top' as 'top' | 'bottom',
  });
  
  useLayoutEffect(() => {
    if (!show || !triggerRef.current) return;
    
    const update = () => {
      if (!triggerRef.current) return;
      const rect = triggerRef.current.getBoundingClientRect();
      const tooltipHeight = 32;  // Approximate yoki ref bilan o'lchash
      const margin = 8;
      
      let placement: 'top' | 'bottom' = 'top';
      let y = rect.top - tooltipHeight - margin;
      
      // Off-screen top → bottom'ga
      if (y < 0) {
        y = rect.bottom + margin;
        placement = 'bottom';
      }
      
      const x = rect.left + rect.width / 2;
      
      setPosition({ x, y, placement });
    };
    
    update();
    window.addEventListener('scroll', update, { passive: true });
    window.addEventListener('resize', update);
    
    return () => {
      window.removeEventListener('scroll', update);
      window.removeEventListener('resize', update);
    };
  }, [show, triggerRef]);
  
  return position;
}
```

**Tushuntirish:**

- `useLayoutEffect` — position paint'dan oldin (no flicker)
- Off-screen check — tooltip viewport'dan chiqib ketmasligi
- Scroll/resize event'lar — position yangilanishi
- `tooltipHeight` approximate — production'da ref orqali o'lchash kerak

Production'da `@floating-ui/react` library ishlatish tavsiya — collision detection, virtual element'lar, transform handling avtomatik.

</details>

### Mashq 5 — `useAutosize` Generic Hook (Qiyin)

Textarea'ni content asosida avtomatik o'lchamlovchi hook yozing. User type qilganda textarea balandligi o'sishi kerak.

```tsx
function useAutosize(
  ref: React.RefObject<HTMLTextAreaElement>,
  value: string
): void {
  // Implement
}

function CommentBox() {
  const ref = useRef<HTMLTextAreaElement>(null);
  const [value, setValue] = useState('');
  
  useAutosize(ref, value);
  
  return (
    <textarea
      ref={ref}
      value={value}
      onChange={e => setValue(e.target.value)}
      style={{ resize: 'none', overflow: 'hidden' }}
    />
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useAutosize(
  ref: React.RefObject<HTMLTextAreaElement>,
  value: string
): void {
  useLayoutEffect(() => {
    const element = ref.current;
    if (!element) return;
    
    // 1. Reset height — keyingi scrollHeight aniq bo'ladi
    element.style.height = 'auto';
    
    // 2. Set new height based on scrollHeight
    element.style.height = `${element.scrollHeight}px`;
  }, [ref, value]);
}
```

**Tushuntirish:**

- `useLayoutEffect` — height update paint'dan oldin (no flicker)
- Step 1: `height = 'auto'` — element o'z natural balandligiga qaytadi (avval o'rnatilgan height yo'q bo'ladi)
- Step 2: `scrollHeight` — content full height (visible + hidden)
- Step 2: `height = scrollHeight px` — element scrollbar'siz full content height
- Deps `[ref, value]` — har value o'zgarganda recalculate

Bu pattern `value` o'zgarganda darrov ishlaydi — paint dan oldin yangi height. `useEffect` bilan bo'lsa: avval eski height paint, keyin yangi paint (flicker, ayniqsa tezkor typing'da).

Edge case'lar:
- `min-height`/`max-height` CSS — JS read'da hisobga olinadi
- Padding/border — `box-sizing: border-box` bilan textarea CSS hal qiladi
- `value` prop o'zgarmaganda (controlled component bo'lmasa) — bu hook ishlamaydi

Production'da `react-textarea-autosize` library ishlatish tavsiya (cross-browser, edge case'lar handled).

</details>

---

## Xulosa

`useLayoutEffect` va `useInsertionEffect` — `useEffect`'ning **timing variant**'lari. Asosiy fikrlar:

- **`useLayoutEffect`** Commit Phase'ning Layout sub-phase'ida sync chaqiriladi — DOM yangilangandan keyin, browser paint'dan oldin. DOM measurement, tooltip positioning, scroll restoration, focus management, layout-dependent calculation uchun.
- **API `useEffect` bilan bir xil** — `useLayoutEffect(setup, deps?)`. Cleanup, deps semantikasi, Strict Mode 2x cycle (R18+), exhaustive-deps linter — barchasi bir xil ishlaydi.
- **Sinxron — paint bloklaydi.** Effect ichida og'ir ish bo'lsa, browser paint kechikadi → user "jank" sezadi. Layout phase ishi frame budget'ga sezilarli ulush qo'shsa optimize qilish kerak (read-then-write pattern, ResizeObserver, useEffect'ga ko'chirish).
- **`useEffect` default tanlov** — `useLayoutEffect` faqat visual flicker oldini olish kerak bo'lganda. React docs: *"This Hook is used less often than `useEffect`."*
- **Layout thrashing** — read va write aralashtirilsa, browser har read uchun layout recalculate qiladi (Forced Synchronous Layout). Read-then-write pattern: avval barcha read, keyin barcha write.
- **DOM measurement API'lari** — `offsetWidth`/`offsetHeight` (with padding+border), `getBoundingClientRect` (full DOMRect), `getComputedStyle`, `ResizeObserver` (efficient o'lcham track), `IntersectionObserver` (visibility track).
- **SSR cheklov** — `useLayoutEffect` server'da chaqirilmaydi va React `useLayoutEffect`'ga xos warning beradi ("does nothing on the server" — DOM hydration mismatch emas). Server HTML va client initial render mos keladi; effect state'ni yangilasa, o'zgarish hydration'dan keyin yuzaga keladi (post-hydration flicker). Yechimlar: `ClientOnly` wrapper, `useEffect`'ga o'tish, Next.js `'use client'` directive, `useIsomorphicLayoutEffect` workaround (library code uchun).
- **`useIsomorphicLayoutEffect` pattern** — `typeof window !== 'undefined' ? useLayoutEffect : useEffect`. Library author'lar (framer-motion, react-use, mantine, chakra-ui) ishlatadi. Application code'da odatda `ClientOnly` wrap afzal.
- **`useInsertionEffect` (R18)** — Mutation sub-phase'da, DOM mutation'dan **oldin** chaqiriladi. **Faqat CSS-in-JS library'lar uchun** (emotion, styled-components v6+). Application code'da ishlatish anti-pattern. SSR-safe (warning yo'q). DOM kirish yo'q (`ref.current` null).
- **Decision Guide:** CSS-in-JS rule injection → `useInsertionEffect` | DOM measure/scroll/focus → `useLayoutEffect` | Boshqa hammasi → `useEffect`.
- **Under the Hood** — `HookLayout` flag (`useLayoutEffect`), `HookInsertion` flag (`useInsertionEffect`), `HookPassive` flag (`useEffect`). Bitmask Commit Phase'da kerakli sub-phase'da effect run qilish uchun. `commitLayoutEffectOnFiber` sync, `flushPassiveEffects` async (MessageChannel).

Keyingi bo'lim: `useRef` — mutable container, DOM access, ref forwarding evolution (R18 `forwardRef` → R19 ref oddiy prop), ref cleanup (R19), `useImperativeHandle` chuqur, callback refs.

---

**Keyingi bo'lim:** [18-useref.md](18-useref.md) — `useRef` mutable container (re-render trigger qilmaydi), DOM refs element access, mutable values (timer ID, prev value, latest closure), ref vs state decision guide, **`forwardRef` evolyutsiyasi (R16.3 → R19)** — R19'da ref oddiy prop bo'ldi, `forwardRef` esa hali to'liq ishlaydi (deprecated emas, warning yo'q; kelajakda deprecate rejalashtirilgan), **ref cleanup functions (R19)** — DOM node o'chirilganda chaqiriladi, `useImperativeHandle` chuqur (imperative API expose, video player/modal/focus management misol), callback refs (dynamic attachment).
