# Bo'lim 18: useRef

> `useRef` — React'ga "yashirin" mutable container yaratuvchi hook. Qaytaradigan obyekt `{ current: T }` shaklida — uning `current` property'sini istalgan vaqtda mutate qilish mumkin, lekin re-render trigger qilmaydi. Bu xususiyat ikki katta use case'ga olib keladi: (1) DOM element'larga imperativ kirish (focus, scroll, measurement), (2) re-render orasida saqlanadigan mutable qiymatlar (timer ID, latest closure, prev value). Bu bo'limda `useRef` API, DOM refs, mutable values, `forwardRef` evolyutsiyasi (R16.3 → R19 ref oddiy prop), R19 ref cleanup functions, `useImperativeHandle` chuqur, callback refs, va TypeScript pattern'lari yoritiladi.

---

## Mundarija

- [Ref Tushunchasi — Mutable Container](#ref-tushunchasi--mutable-container)
- [`useRef` API va Initialization](#useref-api-va-initialization)
- [DOM Refs — Element Access](#dom-refs--element-access)
- [Mutable Values — Non-DOM Use Cases](#mutable-values--non-dom-use-cases)
- [`ref` vs `state` — Decision Guide](#ref-vs-state--decision-guide)
- [String Refs Versiya Tarixi](#string-refs-versiya-tarixi)
- [`forwardRef` Evolyutsiyasi (R16.3 → R19)](#forwardref-evolyutsiyasi-r163--r19)
- [R19 — `ref` Oddiy Prop](#r19--ref-oddiy-prop)
- [Ref Cleanup Functions (R19)](#ref-cleanup-functions-r19)
- [`useImperativeHandle` Chuqur](#useimperativehandle-chuqur)
- [Callback Refs — Dynamic Attachment](#callback-refs--dynamic-attachment)
- [TypeScript Patterns](#typescript-patterns)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Ref Tushunchasi — Mutable Container

### Nazariya

Ref — React'ning **render cycle'idan tashqaridagi** mutable qiymat. `useRef` hook'i `{ current: T }` shaklidagi obyekt qaytaradi. Bu obyekt komponent re-render orasida **saqlanadi** (bir xil reference), va `current` property'ni istalgan vaqtda mutate qilish mumkin **re-render trigger qilmasdan**.

Bu ikki xususiyat — saqlanish va silent mutation — `useRef`'ni unikal qiladi:

| Xususiyat | `useState` | `useRef` |
|-----------|------------|----------|
| Re-render orasida saqlanadi | ✅ | ✅ |
| Mutation trigger re-render | ✅ | ❌ |
| Render paytida o'qish | ✅ (snapshot) | ⚠️ (taqiq, lekin texnik mumkin) |
| Render paytida yozish | ❌ (taqiq) | ⚠️ (taqiq, ba'zi exception) |
| Effect ichida o'qish/yozish | ✅ | ✅ |
| Mutation timing | Async (queue → next render) | Sync (darrov) |
| Bailout (Object.is) | ✅ | — (no compare) |

`useState` declarative — siz "yangi state istayman" deysiz, React render keltiradi. `useRef` imperative — siz "bu qiymatni saqlab qol" deysiz, render bilan bog'liq emas.

**Render purity bilan munosabat:**

Render funksiyasi pure bo'lishi shart (cross-ref [`09-component-basics.md`](09-component-basics.md) "Render Purity"). Render paytida `ref.current` o'qish — ba'zi holatlarda OK (e.g., latest value pattern), ba'zilarida xato (render natijasi non-deterministic):

```tsx
function Component() {
  const ref = useRef(0);
  
  // ❌ Render paytida mutation — silent bug
  ref.current = ref.current + 1;
  
  return <div>Render count: {ref.current}</div>;
  // Strict Mode'da render 2x → ref.current 2 marta oshadi
}
```

Render paytida `ref.current = X` — **render purity buzilishi**. Strict Mode 2x render'da bug ko'rinadi (boshqa kontekst bilan farq).

To'g'ri pattern — mutation faqat event handler yoki effect ichida:

```tsx
function Component() {
  const ref = useRef(0);
  
  useEffect(() => {
    ref.current = ref.current + 1;  // ✅ Effect ichida — render'dan tashqarida
  });
  
  const handleClick = () => {
    ref.current = ref.current + 1;  // ✅ Event handler — user action
  };
}
```

**Mental model:**

- `useState` — React boshqaradigan "live" qiymat (mutation = re-render)
- `useRef` — React'ga ko'rinmas "ichki" qiymat (mutation silent, faqat keyin o'qiganda ko'rinadi)

Misol — closure latest pattern (cross-ref [`16-useeffect.md`](16-useeffect.md)):

```tsx
function Component({ onChange }: { onChange: () => void }) {
  const latestCallback = useRef(onChange);
  
  useEffect(() => {
    latestCallback.current = onChange;  // Har render'da yangilanadi
  });
  
  useEffect(() => {
    const id = setInterval(() => {
      latestCallback.current();  // Doim latest
    }, 1000);
    return () => clearInterval(id);
  }, []);
}
```

`onChange` har render'da yangi function bo'lishi mumkin. `latestCallback.current` har render'da yangilanadi (mutation), lekin re-render trigger qilmaydi. Interval ichida `current` o'qish — har gal latest version.

<details>
<summary><strong>Under the Hood</strong></summary>

**Fiber'da `useRef` saqlash:**

`useRef` chaqirilganda `Hook` obyekt yaratiladi (cross-ref [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md) "Hooks Linked List"):

```ts
type Hook = {
  memoizedState: { current: T },  // Ref obyekt
  baseState: null,
  baseQueue: null,
  queue: null,                     // useState'da queue, useRef'da null
  next: Hook | null,
};
```

`useRef`'ning `Hook.memoizedState` — `{ current: T }` obyekt. `Hook.queue` esa `null` — chunki ref mutation update queue'siz bo'ladi.

**`mountRef` implementation:**

```ts
function mountRef<T>(initialValue: T): { current: T } {
  const hook = mountWorkInProgressHook();
  const ref = { current: initialValue };
  hook.memoizedState = ref;
  return ref;
}
```

Birinchi render'da `{ current: initialValue }` obyekt yaratiladi va Hook'da saqlanadi.

**`updateRef` implementation:**

```ts
function updateRef<T>(initialValue: T): { current: T } {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState;  // ✅ Eski ref obyekt qaytariladi
}
```

Keyingi render'larda **bir xil** ref obyekt qaytariladi. `initialValue` parametri **e'tiborsiz** (faqat birinchi render'da ishlatiladi). Bu degani — ikkinchi render'da `useRef('new value')` qilsangiz, `current` hali avvalgi qiymat.

**Reference identity:**

```tsx
function Component() {
  const ref = useRef(0);
  
  console.log(ref);  // Har render'da bir xil obyekt (===)
}

// Render 1: ref = { current: 0 }
// Render 2: ref = { current: 0 } (bir xil reference, === true)
// ref.current = 5 ← mutation
// Render 3: ref = { current: 5 } (bir xil reference)
```

`useRef` qaytaradigan obyekt har render'da bir xil — bu deps array'lar uchun stable reference (`useEffect`, `useCallback`).

**Render purity violation detection:**

Strict Mode'da ref mutation render paytida ikki marta bajariladi (render 2x):

```tsx
const ref = useRef(0);
ref.current++;  // Strict Mode: 0 → 1 → 2 (ikki marta)

console.log(ref.current);  // 2 (1 emas)
```

Bu bug Strict Mode'da darrov ko'rinadi — production'da 1 ga osha turgan qiymat development'da 2 ga osha turadi.

**Source citation:**

- `mountRef` / `updateRef` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`
- React docs "Referencing Values with Refs" — react.dev

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Re-render trigger qilmaslik:**

```tsx
function ClickCounter() {
  const ref = useRef(0);
  const [, forceUpdate] = useState({});
  
  const handleClick = () => {
    ref.current++;  // Mutation, lekin re-render yo'q
    console.log('Clicks:', ref.current);
  };
  
  return (
    <div>
      <p>Clicks (ref): {ref.current}</p>  {/* Eski qiymat — render trigger qilinmagan */}
      <button onClick={handleClick}>Click</button>
      <button onClick={() => forceUpdate({})}>Force render</button>
    </div>
  );
}

// Click x5 → ref.current = 5, lekin UI 0 ko'rinadi
// "Force render" → UI 5 ko'rinadi (ref.current o'qildi)
```

**Misol 2 — Reference identity:**

```tsx
function IdentityTest() {
  const ref = useRef({ count: 0 });
  const [renderCount, setRenderCount] = useState(0);
  const prevRef = useRef(ref);
  
  useEffect(() => {
    console.log('Same ref?', ref === prevRef.current);  // Doim true
  });
  
  return (
    <button onClick={() => setRenderCount(c => c + 1)}>
      Render #{renderCount}
    </button>
  );
}
```

**Misol 3 — `useState` vs `useRef`:**

```tsx
// State — re-render trigger
function StateExample() {
  const [count, setCount] = useState(0);
  
  console.log('Render with count:', count);
  
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// Ref — silent
function RefExample() {
  const ref = useRef(0);
  
  console.log('Render');  // Click bilan re-render bo'lmaydi
  
  return <button onClick={() => { ref.current++; console.log(ref.current); }}>Click</button>;
}
```

**Misol 4 — Latest closure pattern:**

```tsx
function Logger({ message }: { message: string }) {
  const latestMessage = useRef(message);
  
  useEffect(() => {
    latestMessage.current = message;  // Har render'da yangilanadi
  });
  
  useEffect(() => {
    const id = setInterval(() => {
      console.log('Latest:', latestMessage.current);  // Doim yangi
    }, 1000);
    
    return () => clearInterval(id);
  }, []);  // Bo'sh deps — interval bir marta o'rnatiladi
  
  return <div>Message: {message}</div>;
}

// message='hello' → interval 'hello' log
// message='world' → interval 'world' log (closure stale emas)
```

</details>

---

## `useRef` API va Initialization

### Nazariya

`useRef` signature:

```tsx
function useRef<T>(initialValue: T): React.MutableRefObject<T>;
function useRef<T>(initialValue: T | null): React.RefObject<T>;
function useRef<T = undefined>(): React.MutableRefObject<T | undefined>;
```

3 ta TypeScript overload — initial value va `null` semantikasiga qarab. Returned obyekt:

```tsx
interface MutableRefObject<T> {
  current: T;
}

interface RefObject<T> {
  readonly current: T | null;  // ← readonly!
}
```

`React.RefObject` `current` `readonly` — DOM ref'lar uchun (React mutation qiladi, kod o'qiydi). `MutableRefObject` esa to'liq mutable — mutable values uchun.

**3 ta tipik initialization pattern:**

**Pattern 1 — DOM ref (`null` initial):**

```tsx
const inputRef = useRef<HTMLInputElement>(null);
// Type: RefObject<HTMLInputElement>
// inputRef.current — readonly, React mount paytida tayinlaydi
```

**Pattern 2 — Mutable value (initial qiymat):**

```tsx
const countRef = useRef(0);
// Type: MutableRefObject<number>
// countRef.current — to'liq mutable
```

**Pattern 3 — Lazy mutable (undefined initial):**

```tsx
const observerRef = useRef<IntersectionObserver | undefined>(undefined);
// Type: MutableRefObject<IntersectionObserver | undefined>
// observerRef.current — keyinroq tayinlanadi
```

> **Eslatma:** R19'da `@types/react` argumentsiz overload (`useRef<T>()`) olib tashlandi. Endi har doim explicit initial qiymat (e.g., `undefined` yoki `null`) berish shart.

**Initial value faqat birinchi render'da:**

`useState` lazy initial pattern (cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md)) — `useState(() => expensive())`. `useRef`'da bunday pattern yo'q — `useRef(expensive())` har render'da `expensive()` chaqiradi (lekin natija e'tiborsiz):

```tsx
// ❌ Har render'da expensive() chaqiriladi
function Component() {
  const ref = useRef(expensiveComputation());  // Har render'da chaqiriladi
  // Birinchi render: ref.current = result1
  // Keyingi render'lar: expensiveComputation() chaqiriladi (waste), 
  //                     lekin ref.current avvalgi result1
}

// ✅ Lazy initialization workaround
function Component() {
  const ref = useRef<Result | null>(null);
  if (ref.current === null) {
    ref.current = expensiveComputation();  // Faqat birinchi render
  }
}
```

Bu workaround `useRef` mutation render paytida — texnik jihatdan render purity buziladi, lekin **birinchi render uchun OK** (Strict Mode 2x render'da `if (ref.current === null)` ikkinchi render'da false).

**Type widening trap:**

```tsx
const ref = useRef(0);  
// Type: MutableRefObject<number>
ref.current = 'hello';  // ❌ Type error — number expected

const ref2 = useRef<number | string>(0);
ref2.current = 'hello';  // ✅
```

Initial value type'ni o'rnatadi. Boshqa typelar uchun explicit generic kerak.

**`null` initial gotcha:**

```tsx
const ref = useRef<HTMLInputElement>(null);
// Type: RefObject<HTMLInputElement>
// ref.current — HTMLInputElement | null

ref.current.focus();  // ❌ Type error — possibly null
ref.current?.focus();  // ✅ Optional chaining
```

DOM ref'lar mount'dan oldin `null` (component yangi render'da DOM hali yo'q). Optional chaining yoki guard har gal kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**TypeScript overload tafsilotlari:**

```ts
// @types/react ≥ 18 (R18 davrida — 3 ta overload)
function useRef<T>(initialValue: T): MutableRefObject<T>;
function useRef<T>(initialValue: T | null): RefObject<T>;
function useRef<T = undefined>(): MutableRefObject<T | undefined>;

interface MutableRefObject<T> {
  current: T;
}

interface RefObject<T> {
  readonly current: T | null;
}
```

> **🕐 Versiya evolyutsiyasi (`useRef` types):**
> - **`@types/react` < 19:** uchchala overload mavjud — argumentsiz `useRef<T>()` ishlardi.
> - **`@types/react` ≥ 19:** argumentsiz overload **olib tashlandi**. Har chaqiruv uchun explicit initial qiymat (`undefined` yoki `null`) shart. R19 docs ham `RefObject` ni `current: T | null` (mutable, no readonly) qilib soddalashtirdi.
> - **Sabab:** Argumentsiz forma noaniq edi — kompiler initial value yo'qligini bilmasdi, runtime'da `current === undefined` holatlari kutilmagan bug'larga olib kelardi.

Overload tartib muhim — TypeScript birinchi mos overload'ni tanlaydi:

- `useRef<HTMLInputElement>(null)` → `RefObject<HTMLInputElement>` (2-overload)
- `useRef(0)` → `MutableRefObject<number>` (1-overload)
- `useRef<number | undefined>(undefined)` → `MutableRefObject<number | undefined>` (R19+ universal forma)

**`mountRef` initial value xulq-atvori:**

```ts
function mountRef<T>(initialValue: T): { current: T } {
  const hook = mountWorkInProgressHook();
  const ref = { current: initialValue };
  hook.memoizedState = ref;
  return ref;
}

function updateRef<T>(initialValue: T): { current: T } {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState;  // initialValue parametri ignore qilinadi
}
```

`updateRef` da `initialValue` parametri ishlatilmaydi — eski ref qaytariladi.

**Lazy init pattern xavfsizligi:**

```ts
function Component() {
  const ref = useRef<Heavy | null>(null);
  if (ref.current === null) {
    ref.current = new Heavy();  // Render paytida mutation
  }
}
```

Strict Mode 2x render:
- Render 1.1: `ref.current === null` → `new Heavy()` chaqiriladi → `ref.current = instance1`
- Render 1.2: `ref.current === null` false → skip (`new Heavy()` qayta chaqirilmaydi)

`if (ref.current === null)` guard tufayli construction faqat bir marta. Lekin guard'siz pattern (yoki construction unmount→remount cycle bilan birga) — side effect double-fire keltirib chiqarishi mumkin:

```ts
// ❌ Side effect render paytida — Strict Mode 2x render'ga chidamsiz
const ref = useRef<EventEmitter | null>(null);
if (ref.current === null) {
  ref.current = new EventEmitter();
  ref.current.connect();  // ❌ connect() side effect — render paytida chaqirilmasin
}

// Strict Mode unmount→remount cycle (R18+) — yangi useRef, ref.current === null,
// connect() yana chaqiriladi (eski instance hali ulangan bo'lsa, leak)
```

Ehtiyot bo'lish: lazy init faqat **pure construction** uchun (`new Map()`, `new Set()`, plain object). Side effect (network, event listener attach, subscription) bo'lsa — `useEffect` ichiga ko'chirish (cleanup pattern bilan).

**Source citation:**

- React @types — DefinitelyTyped `types/react/index.d.ts`
- React docs "useRef" reference — react.dev/reference/react/useRef

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — DOM ref:**

```tsx
function FocusInput() {
  const inputRef = useRef<HTMLInputElement>(null);
  
  const handleFocus = () => {
    inputRef.current?.focus();  // Optional chaining
  };
  
  return (
    <div>
      <input ref={inputRef} placeholder="Type here" />
      <button onClick={handleFocus}>Focus</button>
    </div>
  );
}
```

**Misol 2 — Mutable value (timer):**

```tsx
function Timer() {
  const intervalRef = useRef<number | null>(null);
  const [count, setCount] = useState(0);
  
  const start = () => {
    if (intervalRef.current !== null) return;  // Allaqachon ishlaydi
    
    intervalRef.current = window.setInterval(() => {
      setCount(c => c + 1);
    }, 1000);
  };
  
  const stop = () => {
    if (intervalRef.current !== null) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  };
  
  useEffect(() => {
    return () => stop();  // Cleanup unmount paytida
  }, []);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </div>
  );
}
```

**Misol 3 — Lazy initialization:**

```tsx
function VideoPlayer() {
  const playerRef = useRef<VideoPlayer | null>(null);
  
  // Lazy init — birinchi render'da yaratish (heavy constructor)
  if (playerRef.current === null) {
    playerRef.current = new VideoPlayer({ /* config */ });
  }
  
  useEffect(() => {
    return () => {
      playerRef.current?.destroy();  // Cleanup
      playerRef.current = null;
    };
  }, []);
  
  // playerRef.current har doim VideoPlayer (null check shart)
  return <div>{playerRef.current?.getStatus()}</div>;
}
```

**Misol 4 — `useState` vs `useRef` lazy init:**

```tsx
// useState lazy — function syntax
const [data, setData] = useState(() => expensiveCompute());

// useRef lazy — manual if check
const ref = useRef<Data | null>(null);
if (ref.current === null) {
  ref.current = expensiveCompute();
}

// useRef'da function syntax YO'Q:
const ref2 = useRef(() => expensiveCompute());  // ❌ Bu ref.current = function
                                                 //   (function call qilinmaydi)
```

**Misol 5 — TypeScript type narrowing:**

```tsx
function NarrowingDemo() {
  const ref = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    const element = ref.current;
    
    if (element === null) return;  // Type guard
    
    // Bu yerda element: HTMLDivElement (narrowed)
    element.scrollTo({ top: 0 });
    element.style.background = 'red';
  }, []);
  
  return <div ref={ref}>Content</div>;
}
```

</details>

---

## DOM Refs — Element Access

### Nazariya

`useRef`'ning eng tipik use case'i — DOM element'lariga imperative kirish. JSX'da `ref={refObject}` attribute qo'yilsa, React mount paytida DOM element'ni `refObject.current`'ga tayinlaydi:

```tsx
const inputRef = useRef<HTMLInputElement>(null);

// JSX:
<input ref={inputRef} />

// Mount:
// React: inputRef.current = <DOM input element>
```

**Lifecycle:**

```
1. Render: inputRef.current = null
2. Commit Mutation: DOM yaratiladi
3. Commit Layout (Layout sub-phase): inputRef.current = element
4. useLayoutEffect: ref.current mavjud
5. Browser Paint
6. useEffect: ref.current mavjud
```

Render paytida `ref.current = null`. Effect ichida (useLayoutEffect yoki useEffect) — DOM element. Bu cross-ref [`02-rendering.md`](02-rendering.md) "Layout sub-phase" tafsiloti.

**Use case'lar:**

| Use case | Pattern |
|----------|---------|
| Focus input | `ref.current?.focus()` |
| Scroll to element | `ref.current?.scrollIntoView()` |
| Read element size | `ref.current?.getBoundingClientRect()` |
| Play video | `videoRef.current?.play()` |
| Trigger file picker | `fileInputRef.current?.click()` |
| Animation | Web Animations API |

**Imperative API'lar:**

```ts
element.focus({ preventScroll })
element.blur()
element.click()
element.scrollIntoView({ behavior, block })
element.scroll({ top, behavior })
element.requestFullscreen()
videoElement.play()
videoElement.pause()
audioElement.load()
```

Bu API'lar — **imperative**. Declarative React paradigmasi bilan moslashtirish — `useRef` orqali.

**`ref` callback (alternativ):**

`ref` attribute obyekt yoki **function** qabul qiladi:

```tsx
// Object ref (useRef):
<input ref={inputRef} />

// Callback ref:
<input ref={(node) => {
  if (node) {
    console.log('Mounted:', node);
  } else {
    console.log('Unmounted');
  }
}} />
```

Callback ref har mount/unmount'da chaqiriladi. Dynamic attachment uchun foydali (cross-ref Section "Callback Refs").

**`ref` `null` paytlari:**

`ref.current` quyidagilarda `null`:

1. **Initial render** — DOM hali yaratilmagan
2. **Conditional render** — element render'da yo'q
3. **Component unmount** — DOM o'chirilgan (R19 cleanup'dan oldin chaqiriladi)
4. **Suspense fallback** — component pause holatda

Har gal optional chaining (`ref.current?.method()`) yoki guard (`if (ref.current)`) ishlatish.

<details>
<summary><strong>Under the Hood</strong></summary>

**Ref attachment mexanizmi:**

React JSX'da `ref` attribute'ni alohida qayta ishlaydi (boshqa props'lar bilan emas). Render Phase'da `ref` Fiber'ning `ref` field'iga saqlanadi:

```ts
type Fiber = {
  // ...
  ref: { current: T } | ((node: T | null) => void) | null,
  // ...
};
```

Commit Phase paytida React `ref`'ni "attach" qiladi:

```ts
// commitAttachRef (soddalashtirilgan)
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    const instance = finishedWork.stateNode;  // DOM element
    
    if (typeof ref === 'function') {
      ref(instance);  // Callback ref
    } else {
      ref.current = instance;  // Object ref
    }
  }
}
```

Bu Layout sub-phase'da bajariladi — `useLayoutEffect`'dan **oldin**. Demak `useLayoutEffect` ichida `ref.current` mavjud.

**Ref detach (re-attach):**

Element o'rnida o'zgarsa (e.g., conditional render, key change), React eski ref'ni detach qiladi:

```ts
function commitDetachRef(current: Fiber) {
  const currentRef = current.ref;
  if (currentRef !== null) {
    if (typeof currentRef === 'function') {
      currentRef(null);  // Callback bilan null
    } else {
      currentRef.current = null;  // Object ref null
    }
  }
}
```

Detach Mutation sub-phase'da, attach Layout sub-phase'da. Tartib: detach old → DOM update → attach new.

**Native DOM event vs React ref:**

```tsx
const ref = useRef<HTMLInputElement>(null);

useEffect(() => {
  const handler = () => console.log('input!');
  ref.current?.addEventListener('input', handler);
  
  return () => {
    ref.current?.removeEventListener('input', handler);
  };
}, []);
```

`ref.current` orqali native event listener qo'shish — React event system'dan tashqarida. Foydali ba'zi cases:

- Passive listener (`addEventListener('scroll', cb, { passive: true })`)
- Native event'lar (`gestureend`, custom events)
- Third-party library integration

Bu pattern cross-ref [`13-event-handling.md`](13-event-handling.md) "Event Delegation R17+".

**Source citation:**

- `commitAttachRef` / `commitDetachRef` — facebook/react `packages/react-reconciler/src/ReactFiberCommitWork.js`
- React docs "Manipulating the DOM with Refs" — react.dev

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Focus on mount:**

```tsx
function LoginForm() {
  const emailRef = useRef<HTMLInputElement>(null);
  
  useEffect(() => {
    emailRef.current?.focus();
  }, []);
  
  return (
    <form>
      <input ref={emailRef} type="email" placeholder="Email" />
      <input type="password" placeholder="Password" />
      <button>Login</button>
    </form>
  );
}
```

`autoFocus` attribute ham mavjud, lekin `useRef + useEffect` pattern programmatically focus uchun (conditional, dynamic).

**Misol 2 — File picker:**

```tsx
function FileUploader() {
  const fileInputRef = useRef<HTMLInputElement>(null);
  
  const openPicker = () => {
    fileInputRef.current?.click();
  };
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (file) {
      console.log('Selected:', file.name);
    }
  };
  
  return (
    <div>
      <input
        ref={fileInputRef}
        type="file"
        style={{ display: 'none' }}
        onChange={handleChange}
      />
      <button onClick={openPicker}>Upload File</button>
    </div>
  );
}
```

Hidden file input + custom button — common pattern UI customization uchun.

**Misol 3 — Video controls:**

```tsx
function VideoPlayer({ src }: { src: string }) {
  const videoRef = useRef<HTMLVideoElement>(null);
  const [isPlaying, setIsPlaying] = useState(false);
  
  const togglePlay = () => {
    if (!videoRef.current) return;
    
    if (isPlaying) {
      videoRef.current.pause();
    } else {
      videoRef.current.play();
    }
    setIsPlaying(!isPlaying);
  };
  
  const setVolume = (vol: number) => {
    if (videoRef.current) {
      videoRef.current.volume = vol;
    }
  };
  
  return (
    <div>
      <video ref={videoRef} src={src} />
      <button onClick={togglePlay}>{isPlaying ? 'Pause' : 'Play'}</button>
      <input
        type="range"
        min="0"
        max="1"
        step="0.1"
        onChange={(e) => setVolume(parseFloat(e.target.value))}
      />
    </div>
  );
}
```

**Misol 4 — Scroll to element:**

```tsx
function ScrollToSection() {
  const sectionRef = useRef<HTMLElement>(null);
  
  const scrollToSection = () => {
    sectionRef.current?.scrollIntoView({
      behavior: 'smooth',
      block: 'start',
    });
  };
  
  return (
    <div>
      <button onClick={scrollToSection}>Go to section</button>
      <div style={{ height: 2000 }}>...long content...</div>
      <section ref={sectionRef}>Target section</section>
    </div>
  );
}
```

**Misol 5 — Native event listener:**

```tsx
function PassiveScrollContainer() {
  const containerRef = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;
    
    const handler = (e: WheelEvent) => {
      // Passive — preventDefault chaqirilmaydi
      console.log('Scroll delta:', e.deltaY);
    };
    
    container.addEventListener('wheel', handler, { passive: true });
    
    return () => container.removeEventListener('wheel', handler);
  }, []);
  
  return <div ref={containerRef} style={{ overflow: 'auto', height: 400 }} />;
}
```

React'ning `onWheel` event handler'i passive emas — preventDefault'ni qo'llab-quvvatlash uchun. Passive listener uchun native `addEventListener` bilan ref orqali.

</details>

---

## Mutable Values — Non-DOM Use Cases

### Nazariya

`useRef` faqat DOM uchun emas. Har qanday mutable qiymat saqlash uchun ishlatish mumkin — re-render trigger qilmasdan. Tipik use case'lar:

**Use case 1 — Timer/Interval ID:**

`setInterval`/`setTimeout` ID — number qiymat. Cleanup uchun saqlash kerak, lekin re-render trigger qilmaslik kerak:

```tsx
function Stopwatch() {
  const intervalRef = useRef<number | null>(null);
  const [seconds, setSeconds] = useState(0);
  
  const start = () => {
    intervalRef.current = window.setInterval(() => {
      setSeconds(s => s + 1);
    }, 1000);
  };
  
  const stop = () => {
    if (intervalRef.current) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  };
  
  return <button onClick={start}>{seconds}s</button>;
}
```

**Use case 2 — Previous value:**

Joriy va avvalgi render qiymatini taqqoslash:

```tsx
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  
  useEffect(() => {
    ref.current = value;
  });
  
  return ref.current;  // Avvalgi value
}

function PriceChange({ price }: { price: number }) {
  const prevPrice = usePrevious(price);
  const direction = prevPrice !== undefined && price > prevPrice ? '↑' : '↓';
  
  return <div>{price} {direction}</div>;
}
```

`useEffect` har render'dan keyin `ref.current = value` qiladi. Lekin function body'da `ref.current` o'qilganda — hali avvalgi qiymat (effect render'dan keyin chaqiriladi).

**Use case 3 — Latest closure:**

Stale closure muammosini hal qilish (cross-ref [`16-useeffect.md`](16-useeffect.md) "Stale Closure"):

```tsx
function useLatest<T>(value: T): React.MutableRefObject<T> {
  const ref = useRef(value);
  
  useEffect(() => {
    ref.current = value;
  });
  
  return ref;
}

function ChatRoom({ message }: { message: string }) {
  const latestMessage = useLatest(message);
  
  useEffect(() => {
    const handler = () => {
      console.log('Latest:', latestMessage.current);
    };
    
    socket.on('event', handler);
    return () => socket.off('event', handler);
  }, []);  // message deps'da yo'q — latest ref orqali
}
```

**Use case 4 — Singleton/Cache:**

Komponent bo'yicha bir martalik instance:

```tsx
function MapWidget() {
  const mapRef = useRef<L.Map | null>(null);
  const containerRef = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    if (!containerRef.current) return;
    
    // Bir martalik Leaflet map yaratish
    mapRef.current = L.map(containerRef.current).setView([0, 0], 13);
    L.tileLayer('https://...').addTo(mapRef.current);
    
    return () => {
      mapRef.current?.remove();
      mapRef.current = null;
    };
  }, []);
  
  return <div ref={containerRef} style={{ height: 400 }} />;
}
```

**Use case 5 — Mount tracking:**

Komponent mount holatini bilish:

```tsx
function useIsMounted(): React.MutableRefObject<boolean> {
  const isMounted = useRef(true);
  
  useEffect(() => {
    return () => {
      isMounted.current = false;
    };
  }, []);
  
  return isMounted;
}

function AsyncComponent() {
  const isMounted = useIsMounted();
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetchData().then(result => {
      if (isMounted.current) {
        setData(result);  // Faqat mount holatda
      }
    });
  }, []);
  
  return <div>{data}</div>;
}
```

Bu pattern R17 dan oldin "Can't perform setState on unmounted component" warning'ni oldini olar edi. R17+ warning olib tashlangan, lekin pattern hali ham foydali (race condition prevention — cross-ref [`16-useeffect.md`](16-useeffect.md) "Race Conditions" `AbortController` afzal).

**Use case 6 — Render count (debug):**

```tsx
function RenderCounter() {
  const count = useRef(0);
  count.current++;  // Render paytida — debug uchun OK (production'da emas)
  
  console.log('Render #', count.current);
  
  return <div />;
}
```

Render count debug uchun. Production'da render purity buzilishi → ehtiyot bo'lish.

<details>
<summary><strong>Under the Hood</strong></summary>

**`usePrevious` mexanikasi:**

```ts
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  
  useEffect(() => {
    ref.current = value;  // Effect render dan keyin
  });
  
  return ref.current;  // Function body'da — hali eski qiymat
}
```

Lifecycle:
- Render 1, value=A: `ref.current = undefined`, return undefined
- Effect 1: `ref.current = A`
- Render 2, value=B: `ref.current = A`, return A (avvalgi)
- Effect 2: `ref.current = B`
- Render 3, value=C: `ref.current = B`, return B (avvalgi)

`useEffect` har render'dan keyin chaqiriladi → ref.current bir cycle keyin yangilanadi → return value har doim avvalgi render qiymati.

**`useLatest` mexanikasi:**

```ts
function useLatest<T>(value: T): React.MutableRefObject<T> {
  const ref = useRef(value);
  
  useEffect(() => {
    ref.current = value;
  });
  
  return ref;
}
```

`useLatest` `usePrevious`'ga o'xshash, lekin maqsad farqli. Iste'molchi `ref.current` ni event handler/timer ichida o'qiydi — latest qiymat (closure'dan tashqarida).

**Render purity va lazy init:**

```ts
function Component() {
  const ref = useRef<Heavy | null>(null);
  
  // ⚠️ Render paytida mutation
  if (ref.current === null) {
    ref.current = new Heavy();
  }
}
```

Texnik jihatdan render purity buziladi — render natijasi yaratilgan instance'ga bog'liq. Lekin:
- Strict Mode 2x render: 1-render `ref.current = instance1`, 2-render `if` shart false → idempotent
- Production: bir marta yaratiladi
- "Pure construction" (no side effect) uchun OK

React docs bu pattern'ni "Avoiding recreating the ref contents" deb ataydi va tavsiya beradi (react.dev/reference/react/useRef#avoiding-recreating-the-ref-contents).

**`MutableRefObject` vs `RefObject`:**

```ts
interface MutableRefObject<T> {
  current: T;  // mutable
}

interface RefObject<T> {
  readonly current: T | null;  // readonly
}
```

`RefObject` — DOM ref'lar uchun (React mutation qiladi, kod faqat o'qiydi). TypeScript readonly faqat compile-time check — runtime'da har ikki obyekt mutable.

**Source citation:**

- `usePrevious` pattern — React docs FAQ
- `useLatest` pattern — Dan Abramov "Making setInterval Declarative"

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — `usePrevious` hook:**

```tsx
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  
  useEffect(() => {
    ref.current = value;
  });
  
  return ref.current;
}

function StockPrice({ price }: { price: number }) {
  const prevPrice = usePrevious(price);
  
  let direction = '';
  let color = 'gray';
  
  if (prevPrice !== undefined) {
    if (price > prevPrice) { direction = '↑'; color = 'green'; }
    else if (price < prevPrice) { direction = '↓'; color = 'red'; }
  }
  
  return (
    <div style={{ color }}>
      ${price} {direction}
    </div>
  );
}
```

**Misol 2 — Latest callback (event listener):**

```tsx
function useEventListener(
  type: string,
  handler: (e: Event) => void,
  element: EventTarget = window
) {
  const savedHandler = useRef(handler);
  
  useEffect(() => {
    savedHandler.current = handler;
  });
  
  useEffect(() => {
    const wrapper = (e: Event) => savedHandler.current(e);
    
    element.addEventListener(type, wrapper);
    return () => element.removeEventListener(type, wrapper);
  }, [type, element]);
}

function ScrollLogger() {
  const [count, setCount] = useState(0);
  
  // handler har render'da yangi function bo'lishi mumkin
  useEventListener('scroll', () => {
    console.log('Count:', count);  // Latest count
  });
  
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

**Misol 3 — Mount tracking:**

```tsx
function useIsMounted() {
  const isMounted = useRef(true);
  
  useEffect(() => {
    isMounted.current = true;
    return () => {
      isMounted.current = false;
    };
  }, []);
  
  return isMounted;
}

function DelayedUpdate() {
  const isMounted = useIsMounted();
  const [text, setText] = useState('Initial');
  
  const updateLater = async () => {
    await new Promise(r => setTimeout(r, 5000));
    if (isMounted.current) {
      setText('Updated');
    }
  };
  
  return <button onClick={updateLater}>{text}</button>;
}
```

**Misol 4 — Singleton instance:**

```tsx
function ChatRoom({ roomId }: { roomId: string }) {
  const connectionRef = useRef<Connection | null>(null);
  
  useEffect(() => {
    connectionRef.current = new Connection(roomId);
    connectionRef.current.connect();
    
    return () => {
      connectionRef.current?.disconnect();
      connectionRef.current = null;
    };
  }, [roomId]);
  
  const sendMessage = (text: string) => {
    connectionRef.current?.send(text);
  };
  
  return <button onClick={() => sendMessage('hi')}>Send</button>;
}
```

**Misol 5 — Render count (debug):**

```tsx
function useRenderCount(): number {
  const count = useRef(0);
  count.current++;  // ⚠️ Render purity violation, lekin debug uchun
  return count.current;
}

function ExpensiveComponent({ data }: { data: unknown[] }) {
  const renderCount = useRenderCount();
  console.log('Render #', renderCount);
  
  return (
    <div>
      <p>Renders: {renderCount}</p>
      <ul>{data.map((d, i) => <li key={i}>{String(d)}</li>)}</ul>
    </div>
  );
}

// Strict Mode'da double render → renderCount 2x ko'p (dev only)
```

</details>

---

## `ref` vs `state` — Decision Guide

### Nazariya

`useRef` va `useState` — bir-biriga o'xshash, lekin maqsad farqli. Quyidagi savollar bilan to'g'ri tanlash:

```
Savol 1: Bu qiymat UI'da ko'rsatiladi'mi (render paytida o'qiladi'mi)?
  → ha → useState
  → yo'q → 2-savol

Savol 2: Bu qiymat o'zgarishi UI'ni yangilashi kerak'mi?
  → ha → useState
  → yo'q → useRef

Savol 3: Bu qiymat re-render orasida saqlanishi kerak'mi?
  → ha (useRef yoki useState)
  → yo'q → oddiy local variable
```

**Decision matrix:**

| Use case | Tanlov |
|----------|--------|
| Form input value (controlled) | `useState` |
| Toggle (open/close) | `useState` |
| Counter (UI'da ko'rsatiladi) | `useState` |
| Timer ID | `useRef` |
| Scroll position (UI'da ko'rsatilmaydi) | `useRef` |
| Latest callback closure | `useRef` |
| Previous value (compare uchun) | `useRef` |
| WebSocket connection | `useRef` |
| DOM element reference | `useRef` |
| Mount status flag | `useRef` |
| Submit count (analytics) | `useRef` (UI'da yo'q) |
| Submit count (UI'da ko'rsatish) | `useState` |

**Performance farqi:**

`useRef` mutation O(1), re-render yo'q. `useState` mutation update queue, render cycle, reconciliation, commit. `useRef` arzonroq, lekin UI sync yo'q.

**Anti-pattern: `useRef` UI uchun:**

```tsx
// ❌ ANTI-PATTERN
function Counter() {
  const count = useRef(0);
  
  const increment = () => {
    count.current++;
    // Hech narsa ko'rinmaydi — re-render yo'q
  };
  
  return <button onClick={increment}>{count.current}</button>;
}

// ✅ TO'G'RI
function Counter() {
  const [count, setCount] = useState(0);
  
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

UI'da ko'rsatiladigan qiymat har doim `useState` (yoki derived from state).

**Anti-pattern: `useState` lookup uchun:**

```tsx
// ❌ ANTI-PATTERN — har gal re-render
function Logger() {
  const [logs, setLogs] = useState<string[]>([]);
  
  const addLog = (msg: string) => {
    setLogs(prev => [...prev, msg]);  // Har log re-render
  };
  
  // Lekin logs UI'da ko'rsatilmaydi — faqat console.log uchun
  
  return <button onClick={() => addLog('click')}>Log</button>;
}

// ✅ Agar UI'da kerak bo'lmasa — useRef
function Logger() {
  const logs = useRef<string[]>([]);
  
  const addLog = (msg: string) => {
    logs.current.push(msg);  // Re-render yo'q
  };
  
  return <button onClick={() => addLog('click')}>Log</button>;
}
```

**Hybrid pattern — ref + state:**

Ba'zan bir qiymat ham UI'da ko'rsatiladi, ham latest closure'da kerak:

```tsx
function VolumeControl() {
  const [volume, setVolume] = useState(50);
  const volumeRef = useRef(volume);
  
  useEffect(() => {
    volumeRef.current = volume;
  });
  
  useEffect(() => {
    const interval = setInterval(() => {
      console.log('Latest volume:', volumeRef.current);  // Latest, no re-render
    }, 1000);
    
    return () => clearInterval(interval);
  }, []);
  
  return (
    <input
      type="range"
      value={volume}
      onChange={(e) => setVolume(Number(e.target.value))}
    />
  );
}
```

Volume UI'da ko'rsatiladi (slider position) → `useState`. Interval ichida latest volume kerak (interval re-create kerak emas) → `useRef`.

<details>
<summary><strong>Under the Hood</strong></summary>

**Performance comparison:**

```ts
// useState mutation
setCount(5);
// 1. Update queue'ga qo'shiladi
// 2. Schedule update (Lanes priority)
// 3. Render Phase: state hisoblash
// 4. Reconciliation: virtual tree compare
// 5. Commit Phase: DOM update
// 6. Re-render trigger qilinadi

// useRef mutation
ref.current = 5;
// 1. Direct memory write
// 2. Hech narsa
```

`useState` — N step, `useRef` — 1 step. Lekin `useState` UI sync — bu bilan maqsad farqi.

**React reconciliation va ref:**

`useRef` qaytaradigan obyekt har render'da bir xil reference. `useState` qaytaradigan value har gal yangi snapshot. Bu deps array uchun farq:

```tsx
const ref = useRef(0);
useEffect(() => {
  console.log('Effect');
}, [ref]);  // Hech qachon trigger qilinmaydi (ref bir xil)

const [count, setCount] = useState(0);
useEffect(() => {
  console.log('Effect');
}, [count]);  // count o'zgarsa trigger qilinadi
```

`ref` deps'da kerak emas — har doim bir xil reference. ESLint exhaustive-deps `ref.current` ni reactive deb hisoblamaydi.

**Mental model — concurrency:**

`useState` Concurrent Mode'da safe — render restart bo'lsa state to'g'ri qiymat ko'radi. `useRef` esa **mutation darrov** — concurrent rendering paytida tearing potensial.

```tsx
// ⚠️ Concurrent rendering xavfli
function Component() {
  const ref = useRef(0);
  ref.current++;  // ❌ Render paytida mutation
  
  return <div>{ref.current}</div>;
}

// Concurrent restart:
// Render 1.1: ref.current 0→1, return <div>1</div>
// (render restart sababli abandon)
// Render 1.2: ref.current 1→2, return <div>2</div>
// 
// Foydalanuvchi 2 ko'radi, lekin "real" render bir marta — confusing
```

Render purity invariant: ref render paytida mutate qilinmaydi.

**Source citation:**

- React docs "Choosing the State Structure" — react.dev
- React docs "When to use refs" — react.dev/learn/referencing-values-with-refs

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Counter (state to'g'ri):**

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

**Misol 2 — Click count (state agar UI'da):**

```tsx
// ✅ UI'da → useState
function ClickTracker() {
  const [clicks, setClicks] = useState(0);
  
  return (
    <div>
      <button onClick={() => setClicks(c => c + 1)}>Click</button>
      <p>Total: {clicks}</p>
    </div>
  );
}

// ✅ UI'da yo'q (analytics) → useRef
function ClickTrackerSilent() {
  const clicks = useRef(0);
  
  return (
    <button onClick={() => {
      clicks.current++;
      analytics.track('click', { count: clicks.current });
    }}>
      Click
    </button>
  );
}
```

**Misol 3 — Form draft (hybrid):**

```tsx
function DraftEditor() {
  const [content, setContent] = useState('');  // UI controlled input
  const wordCountRef = useRef(0);              // Internal count
  const lastSavedRef = useRef<string>('');     // Save tracker
  
  const handleChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    const value = e.target.value;
    setContent(value);  // UI sync
    wordCountRef.current = value.split(/\s+/).length;  // Silent
  };
  
  const save = () => {
    if (content !== lastSavedRef.current) {
      api.save(content);
      lastSavedRef.current = content;
    }
  };
  
  return (
    <div>
      <textarea value={content} onChange={handleChange} />
      <button onClick={save}>Save</button>
    </div>
  );
}
```

**Misol 4 — WebSocket (ref):**

```tsx
function ChatRoom() {
  const wsRef = useRef<WebSocket | null>(null);
  const [messages, setMessages] = useState<string[]>([]);
  
  useEffect(() => {
    wsRef.current = new WebSocket('wss://example.com');
    wsRef.current.onmessage = (e) => {
      setMessages(prev => [...prev, e.data]);
    };
    
    return () => {
      wsRef.current?.close();
      wsRef.current = null;
    };
  }, []);
  
  const send = (text: string) => {
    wsRef.current?.send(text);  // Imperative
  };
  
  return (
    <div>
      <ul>{messages.map((m, i) => <li key={i}>{m}</li>)}</ul>
      <button onClick={() => send('hi')}>Send</button>
    </div>
  );
}
```

WebSocket — connection imperative, messages declarative. Hybrid pattern.

**Misol 5 — Wrong choice debug:**

```tsx
// ❌ XATO — ref UI uchun
function BrokenToggle() {
  const isOpen = useRef(false);
  
  return (
    <button onClick={() => { isOpen.current = !isOpen.current; }}>
      {isOpen.current ? 'Close' : 'Open'}
    </button>
  );
}
// Click: hech narsa o'zgarmaydi (re-render yo'q)

// ✅ TO'G'RI — useState
function WorkingToggle() {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <button onClick={() => setIsOpen(!isOpen)}>
      {isOpen ? 'Close' : 'Open'}
    </button>
  );
}
```

</details>

---

## String Refs Versiya Tarixi

### Nazariya

> **🕐 Versiya evolyutsiyasi (Refs API):**
> - **Pre-R16 (legacy):** String refs — `<input ref="myInput" />` + `this.refs.myInput`. Class component'da only.
> - **R16+ (modern):** `React.createRef()` (class) yoki `useRef` (function). Object refs.
> - **R19:** String refs **to'liq olib tashlandi** — kompilyatsiya xatosi yoki runtime warning.
> - **Sabab:** String refs Concurrent Mode'da broken (bir xil string component bir necha marta render bo'lsa, `this.refs` qaysi instance'ni saqlaydi noaniq), type-unsafe (TypeScript inferensiya yo'q), hidden state (component'ning `this.refs` namespace'ida implicit).

**Eski API (Pre-R16):**

```tsx
class OldComponent extends React.Component {
  componentDidMount() {
    // String ref orqali
    this.refs.myInput.focus();  // ❌ Modern React'da xato
  }
  
  render() {
    return <input ref="myInput" />;  // ❌ Modern React'da xato
  }
}
```

**Modern (R16+):**

```tsx
// Class component
class ModernComponent extends React.Component {
  inputRef = React.createRef<HTMLInputElement>();
  
  componentDidMount() {
    this.inputRef.current?.focus();
  }
  
  render() {
    return <input ref={this.inputRef} />;
  }
}

// Function component
function FunctionComponent() {
  const inputRef = useRef<HTMLInputElement>(null);
  
  useEffect(() => {
    inputRef.current?.focus();
  }, []);
  
  return <input ref={inputRef} />;
}
```

**Nima uchun string refs olib tashlandi:**

1. **Concurrent Mode incompatible** — bir xil string ref ko'p instance'ga ulanishi mumkin, qaysi instance saqlanadi noaniq
2. **Type unsafe** — `this.refs.myInput` har doim `any` (TypeScript inferensiya yo'q)
3. **Hidden state** — `this.refs` "magic" namespace, kompilyator analiz qila olmaydi
4. **Bundle size** — string refs uchun React internal lookup mexanizmi
5. **Performance** — string lookup vs object reference

R19'da string refs to'liq olib tashlandi. Migration:

```tsx
// ❌ R19'da xato
<input ref="myInput" />

// ✅ Modern
const inputRef = useRef(null);
<input ref={inputRef} />
```

Codemod mavjud (`@react/codemod`) — string refs'ni avtomatik o'zgartiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

**String refs implementation (legacy):**

```ts
// React internal (R15-R16, simplified)
class Component {
  refs = {};
  
  // Render paytida ref="myInput" topiladi
  // Mount paytida React DOM element'ni this.refs.myInput'ga tayinlaydi
  
  componentDidMount() {
    // this.refs.myInput accessible
  }
}
```

Bu pattern internal "ResolveRef" funksiyasi orqali ishlardi. R16'da deprecated, R17-R18'da warning, R19'da olib tashlandi.

**`createRef` vs `useRef`:**

```ts
// createRef — har chaqiruvda yangi obyekt
const ref1 = React.createRef();
const ref2 = React.createRef();
ref1 === ref2;  // false

// useRef — komponent instance bo'yicha bir xil
function Component() {
  const ref = useRef(null);
  // Har render'da bir xil ref obyekt
}
```

Class component'larda `createRef` instance field'ida saqlanadi (instantiation paytida bir marta). Function component'larda `useRef` Hook chain'ida saqlanadi (har render'da bir xil obyekt qaytariladi).

**Source citation:**

- String refs deprecation — facebook/react CHANGELOG R16+
- R19 string refs olib tashlanishi — facebook/react R19 release notes
- @react/codemod — facebook/react `packages/react-codemod`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Migration Pre-R16 → modern:**

```tsx
// ❌ Pre-R16 (string ref)
class LegacyForm extends React.Component {
  componentDidMount() {
    this.refs.email.focus();
  }
  
  render() {
    return (
      <form>
        <input ref="email" type="email" />
        <input ref="password" type="password" />
      </form>
    );
  }
}

// ✅ R16+ (createRef class)
class ModernForm extends React.Component {
  emailRef = React.createRef<HTMLInputElement>();
  passwordRef = React.createRef<HTMLInputElement>();
  
  componentDidMount() {
    this.emailRef.current?.focus();
  }
  
  render() {
    return (
      <form>
        <input ref={this.emailRef} type="email" />
        <input ref={this.passwordRef} type="password" />
      </form>
    );
  }
}

// ✅ R16.8+ (useRef function)
function FunctionForm() {
  const emailRef = useRef<HTMLInputElement>(null);
  const passwordRef = useRef<HTMLInputElement>(null);
  
  useEffect(() => {
    emailRef.current?.focus();
  }, []);
  
  return (
    <form>
      <input ref={emailRef} type="email" />
      <input ref={passwordRef} type="password" />
    </form>
  );
}
```

**Misol 2 — Concurrent Mode incompatibility:**

```tsx
// ❌ String ref Concurrent'da broken
class Tabs extends React.Component {
  componentDidUpdate() {
    // this.refs.activeTab — qaysi tab? Render restart bo'lsa noaniq
    this.refs.activeTab?.focus();
  }
  
  render() {
    return (
      <>
        {this.props.tabs.map((tab, i) => (
          <button
            key={tab.id}
            ref={i === this.state.active ? "activeTab" : null}
          >
            {tab.label}
          </button>
        ))}
      </>
    );
  }
}

// ✅ Modern — explicit ref
class TabsModern extends React.Component {
  activeTabRef = React.createRef<HTMLButtonElement>();
  
  componentDidUpdate() {
    this.activeTabRef.current?.focus();
  }
  
  render() {
    return (
      <>
        {this.props.tabs.map((tab, i) => (
          <button
            key={tab.id}
            ref={i === this.state.active ? this.activeTabRef : null}
          >
            {tab.label}
          </button>
        ))}
      </>
    );
  }
}
```

</details>

---

## `forwardRef` Evolyutsiyasi (R16.3 → R19)

### Nazariya

> **🕐 Versiya evolyutsiyasi (`forwardRef` → ref as prop):**
> - **Pre-R16.3:** Function component'lar ref qabul qila olmaydi — class component yoki ref forwarding HOC manual implement qilingan.
> - **R16.3 (`forwardRef`):** `React.forwardRef(...)` wrapper introduced — function component'da ref qabul qilish.
> - **R19:** `ref` oddiy prop bo'ldi — `forwardRef` wrapper kerak emas. Lekin `forwardRef` **deprecated emas** — gradually phased out (mavjud kod ishlaydi).
> - **Sabab:** `forwardRef` API ortiqcha boilerplate edi (har component uchun wrapper). R19'da JSX transform ref'ni avtomatik prop sifatida o'tkazadi. Yagona qoida — function component'lar ref qabul qiladi.

**Pre-R16.3 muammo:**

```tsx
function FancyButton({ children }: { children: React.ReactNode }) {
  return <button className="fancy">{children}</button>;
}

// Parent ichida ref qo'yib bo'lmaydi:
<FancyButton ref={myRef}>Click</FancyButton>  // ❌ Function component ref qabul qilmaydi
```

R16.3 dan oldin function component'lar ref qabul qilmas edi. Yechim — class component yoki manual prop:

```tsx
// Pre-R16.3 workaround
function FancyButton({ children, innerRef }: { children: React.ReactNode; innerRef: React.RefObject<HTMLButtonElement> }) {
  return <button ref={innerRef} className="fancy">{children}</button>;
}

// Usage:
<FancyButton innerRef={myRef}>Click</FancyButton>
```

**R16.3 — `forwardRef`:**

```tsx
import { forwardRef } from 'react';

const FancyButton = forwardRef<HTMLButtonElement, { children: React.ReactNode }>(
  ({ children }, ref) => {
    return <button ref={ref} className="fancy">{children}</button>;
  }
);

// Usage:
<FancyButton ref={myRef}>Click</FancyButton>  // ✅ Ishlaydi
```

`forwardRef` HOC — function component'ni ref qabul qiladigan komponentga aylantiradi. Ref ikkinchi argument sifatida component function'ga o'tkaziladi.

**R19 — ref oddiy prop:**

```tsx
// ✅ R19 — forwardRef yo'q, ref oddiy prop
function FancyButton({
  children,
  ref,
}: {
  children: React.ReactNode;
  ref?: React.Ref<HTMLButtonElement>;
}) {
  return <button ref={ref} className="fancy">{children}</button>;
}

// Usage:
<FancyButton ref={myRef}>Click</FancyButton>
```

R19'da JSX transform ref'ni avtomatik prop sifatida o'tkazadi. `forwardRef` wrapper yo'q.

**Backward compatibility:**

R19'da `forwardRef` hali ham ishlaydi (deprecated emas, lekin "phased out"). Mavjud kod o'zgartirish shart emas. Yangi kod uchun ref oddiy prop afzal:

```tsx
// ✅ R19'da ham ishlaydi (legacy)
const FancyButton = forwardRef<HTMLButtonElement, ButtonProps>(...);

// ✅ R19+ tavsiya etilgan (oddiy prop)
function FancyButton({ ref, ...props }: ButtonProps & { ref?: React.Ref<HTMLButtonElement> }) {
  return <button ref={ref} {...props} />;
}
```

**displayName:**

`forwardRef` bilan komponent DevTools'da "ForwardRef" deb ko'rinadi (default). Display name set qilish mumkin:

```tsx
const FancyButton = forwardRef<HTMLButtonElement, ButtonProps>((props, ref) => {
  return <button ref={ref} {...props} />;
});

FancyButton.displayName = 'FancyButton';
```

R19 ref oddiy prop'da `displayName` masalasi yo'q — function declaration'da nom o'zi ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`forwardRef` implementation:**

```ts
// React (soddalashtirilgan)
function forwardRef<T, P>(render: (props: P, ref: React.Ref<T>) => React.ReactNode) {
  return {
    $$typeof: REACT_FORWARD_REF_TYPE,
    render,
  };
}
```

`forwardRef` qaytaradigan obyekt — `$$typeof: REACT_FORWARD_REF_TYPE` markered. JSX transform va Reconciler bu turni ajratadi:

- Render Phase: Reconciler `render(props, ref)` chaqiradi — ref ikkinchi argument
- Mount: Ref attach qilinadi

R19'da JSX transform `ref` prop'ni alohida qayta ishlamaydi — `props.ref` to'g'ridan-to'g'ri component function'ga o'tkaziladi:

```ts
// R19 JSX transform output
_jsx(FancyButton, { ref: myRef, children: 'Click' });

// FancyButton(props):
//   props = { ref: myRef, children: 'Click' }
//   props.ref ishlatilishi mumkin
```

**`React.Ref<T>` type:**

```ts
type Ref<T> = RefCallback<T> | RefObject<T> | null;
type RefCallback<T> = (instance: T | null) => void | (() => void);  // R19: cleanup function ham OK
type RefObject<T> = { readonly current: T | null };
```

`Ref<T>` — object ref yoki callback ref yoki null. R19'da callback ref cleanup function qaytarishi mumkin (cross-ref Section "Ref Cleanup Functions").

**Migration codemod:**

```bash
npx @react/codemod forward-ref-to-ref-prop ./src
```

Codemod `forwardRef` wrapper'ni olib tashlaydi va ref'ni prop sifatida qayta yozadi. R19 migration uchun.

**Source citation:**

- `forwardRef` — facebook/react `packages/react/src/ReactForwardRef.js`
- R19 ref as prop — React 19 release notes, RFC

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — `forwardRef` (R16.3 → R18 standart):**

```tsx
import { forwardRef } from 'react';

interface ButtonProps {
  children: React.ReactNode;
  variant?: 'primary' | 'secondary';
}

const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ children, variant = 'primary' }, ref) => {
    return (
      <button ref={ref} className={`btn-${variant}`}>
        {children}
      </button>
    );
  }
);

Button.displayName = 'Button';

// Usage
function App() {
  const ref = useRef<HTMLButtonElement>(null);
  
  useEffect(() => {
    ref.current?.focus();
  }, []);
  
  return <Button ref={ref}>Click</Button>;
}
```

**Misol 2 — R19 ref oddiy prop:**

```tsx
interface ButtonProps {
  children: React.ReactNode;
  variant?: 'primary' | 'secondary';
  ref?: React.Ref<HTMLButtonElement>;  // ✅ Oddiy prop
}

function Button({ children, variant = 'primary', ref }: ButtonProps) {
  return (
    <button ref={ref} className={`btn-${variant}`}>
      {children}
    </button>
  );
}

// Usage — bir xil
function App() {
  const ref = useRef<HTMLButtonElement>(null);
  return <Button ref={ref}>Click</Button>;
}
```

**Misol 3 — TypeScript types comparison:**

```tsx
// R18 (forwardRef)
import { forwardRef, type ForwardedRef } from 'react';

const Component = forwardRef<HTMLDivElement, { text: string }>(
  ({ text }, ref) => {
    return <div ref={ref}>{text}</div>;
  }
);

// R19 (ref as prop) — type cleaner
function Component({ text, ref }: { text: string; ref?: React.Ref<HTMLDivElement> }) {
  return <div ref={ref}>{text}</div>;
}
```

R19'da TypeScript types soddalashdi — generic ikki parametr (`forwardRef<T, P>`) o'rniga oddiy interface.

**Misol 4 — Spread props with ref:**

```tsx
// R18 (forwardRef)
const Input = forwardRef<HTMLInputElement, React.InputHTMLAttributes<HTMLInputElement>>(
  (props, ref) => {
    return <input ref={ref} {...props} />;
  }
);

// R19 (ref as prop)
function Input({
  ref,
  ...props
}: React.InputHTMLAttributes<HTMLInputElement> & { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...props} />;
}

// R19 alternative — ComponentPropsWithRef helper:
function Input(props: React.ComponentPropsWithRef<'input'>) {
  return <input {...props} />;
  // ref props ichida — to'g'ridan-to'g'ri spread'da
}
```

`ComponentPropsWithRef` (R19+) — element propslarini va ref'ni birga oladi. Wrapper component'lar uchun foydali (cross-ref [`10-props.md`](10-props.md)).

</details>

---

## R19 — `ref` Oddiy Prop

### Nazariya

R19'da function component'lar ref'ni boshqa props bilan birga qabul qiladi. `forwardRef` wrapper kerak emas:

```tsx
// R19 idiomatic
function MyInput({ ref, ...props }: React.ComponentProps<'input'>) {
  return <input ref={ref} {...props} />;
}
```

`React.ComponentProps<'input'>` — `<input>` element'ning barcha attribute'larini va `ref`'ni birga oladi (R19'da ref ham ichkariga kiritilgan).

**Migration patterns:**

| R18 (forwardRef) | R19 (ref as prop) |
|------------------|-------------------|
| `forwardRef<T, P>((props, ref) => ...)` | `function Comp({ ref, ...props }: P & { ref?: Ref<T> })` |
| `ComponentPropsWithoutRef<'input'>` | `ComponentPropsWithoutRef<'input'>` (hali mavjud) yoki `ComponentProps` |
| `ForwardedRef<T>` | `Ref<T>` |
| `displayName` | Function name avtomatik |
| Wrapper boilerplate | Yo'q |

**`ComponentProps<E>` vs `ComponentPropsWithRef<E>` vs `ComponentPropsWithoutRef<E>`:**

```ts
// R19+
type ComponentProps<E> = ComponentPropsWithRef<E>;  // Default — ref bilan

// Backward compat (R18 bilan moslashish)
type ComponentPropsWithRef<E> = ...;     // ref ichida
type ComponentPropsWithoutRef<E> = ...;  // ref ichida emas
```

R19'dan boshlab `ComponentProps` default ref bilan keladi. R18'da `ComponentProps` ref'siz edi (ref'ni alohida `forwardRef` qabul qilardi).

**Polymorphic component'lar:**

R18'da polymorphic + ref murakkab edi (cross-ref [`11-composition.md`](11-composition.md)). R19'da soddalashdi:

```tsx
// R19 polymorphic + ref
type PolymorphicProps<E extends React.ElementType> = {
  as?: E;
  children?: React.ReactNode;
} & React.ComponentPropsWithoutRef<E> & {
  ref?: React.Ref<React.ElementRef<E>>;
};

function Box<E extends React.ElementType = 'div'>({
  as,
  ref,
  children,
  ...props
}: PolymorphicProps<E>) {
  const Component = as ?? 'div';
  return <Component ref={ref} {...props}>{children}</Component>;
}

// Usage
<Box as="button" ref={buttonRef}>Click</Box>
```

R18'da `forwardRef` HOC + generic + assertion — boilerplate. R19'da function declaration + interface — clean.

**`ComponentRef<E>` helper:**

```ts
type ComponentRef<E> = E extends React.ElementType
  ? React.ElementRef<E>
  : never;
```

JSX intrinsic element'lar uchun ref tipini olish:

```tsx
type InputRef = ComponentRef<'input'>;     // HTMLInputElement
type DivRef = ComponentRef<'div'>;          // HTMLDivElement
type CustomRef = ComponentRef<typeof MyComponent>;  // Component'ning ref tipi
```

**Migration script:**

```bash
# Codemod — forwardRef → ref as prop
npx @react/codemod forward-ref-to-ref-prop ./src

# Manual migration:
# 1. forwardRef wrapper olib tashlash
# 2. ref ikkinchi argument'dan props ichiga ko'chirish
# 3. TS types ComponentProps → ComponentProps (R19) yoki PropsWithRef
# 4. displayName olib tashlash (function name avtomatik)
```

<details>
<summary><strong>Under the Hood</strong></summary>

**JSX transform R19:**

R18 va undan oldin:
```tsx
<MyComp ref={r} />
```
↓ JSX transform
```ts
React.createElement(MyComp, { ref: r });
// Yoki R17+ automatic:
_jsx(MyComp, { ref: r });
```

React paytida `ref` prop'i alohida ajratilardi — function component'ga o'tkazilmas edi (faqat class va `forwardRef`'ga).

R19'da JSX transform `ref` prop'ni boshqa props bilan birga function component'ga o'tkazadi. React internal'da `ref` prop'ning maxsus xulqi yo'q — oddiy prop.

**`React.ElementRef<E>` type:**

```ts
// R18+ helper
type ElementRef<E> =
  E extends React.ComponentType<infer P>
    ? P extends { ref?: React.Ref<infer R> } ? R : never
    : E extends keyof JSX.IntrinsicElements
    ? JSX.IntrinsicElements[E] extends React.DetailedHTMLProps<infer A, infer R> ? R : never
    : never;

// Usage
type DivRef = ElementRef<'div'>;     // HTMLDivElement
type InputRef = ElementRef<'input'>; // HTMLInputElement
```

`ElementRef<E>` — element ref tipi. Polymorphic component'lar uchun foydali.

**`useImperativeHandle` bilan kombinatsiya:**

R19'da `useImperativeHandle` ham ref oddiy prop'ni qabul qiladi:

```tsx
// R18
const Modal = forwardRef<ModalHandle, ModalProps>((props, ref) => {
  useImperativeHandle(ref, () => ({ open, close }));
  return <div>...</div>;
});

// R19
function Modal({ ref, ...props }: ModalProps & { ref?: React.Ref<ModalHandle> }) {
  useImperativeHandle(ref, () => ({ open, close }));
  return <div>...</div>;
}
```

`useImperativeHandle` API o'zgarmagan — `ref` argument'i forwardRef'dan kelgan ref yoki props.ref. Bir xil ishlaydi.

**Source citation:**

- React 19 release notes — react.dev/blog/2024
- `ComponentProps`, `ElementRef` — DefinitelyTyped @types/react

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Sodda Input (R19):**

```tsx
function MyInput(props: React.ComponentProps<'input'>) {
  return <input className="custom-input" {...props} />;
  // ref ham automatic spread bilan
}

// Usage
function App() {
  const ref = useRef<HTMLInputElement>(null);
  return <MyInput ref={ref} placeholder="Email" />;
}
```

**Misol 2 — Custom button (R19):**

```tsx
type ButtonProps = React.ComponentProps<'button'> & {
  variant?: 'primary' | 'secondary';
};

function Button({ variant = 'primary', className = '', ref, ...props }: ButtonProps) {
  return (
    <button
      ref={ref}
      className={`btn btn-${variant} ${className}`}
      {...props}
    />
  );
}
```

**Misol 3 — Migration full example:**

```tsx
// R18 (forwardRef)
import { forwardRef } from 'react';

interface CardProps {
  title: string;
  children: React.ReactNode;
}

const Card = forwardRef<HTMLDivElement, CardProps>(
  ({ title, children }, ref) => {
    return (
      <div ref={ref} className="card">
        <h3>{title}</h3>
        <div>{children}</div>
      </div>
    );
  }
);
Card.displayName = 'Card';

// R19 (ref as prop)
interface CardProps {
  title: string;
  children: React.ReactNode;
  ref?: React.Ref<HTMLDivElement>;
}

function Card({ title, children, ref }: CardProps) {
  return (
    <div ref={ref} className="card">
      <h3>{title}</h3>
      <div>{children}</div>
    </div>
  );
}
```

**Misol 4 — Polymorphic + ref (R19):**

```tsx
type PolymorphicProps<E extends React.ElementType> = {
  as?: E;
  children: React.ReactNode;
} & Omit<React.ComponentPropsWithoutRef<E>, 'as' | 'children'> & {
  ref?: React.Ref<React.ElementRef<E>>;
};

function Text<E extends React.ElementType = 'span'>({
  as,
  ref,
  children,
  ...props
}: PolymorphicProps<E>) {
  const Component = as ?? 'span';
  return <Component ref={ref} {...props}>{children}</Component>;
}

// Usage
const spanRef = useRef<HTMLSpanElement>(null);
<Text ref={spanRef}>Default</Text>

const buttonRef = useRef<HTMLButtonElement>(null);
<Text as="button" ref={buttonRef}>Button</Text>

const linkRef = useRef<HTMLAnchorElement>(null);
<Text as="a" ref={linkRef} href="/about">Link</Text>
```

**Misol 5 — Conditional ref:**

```tsx
function Item({ index, isActive }: { index: number; isActive: boolean }) {
  // R19 — ref oddiy prop sifatida conditional o'tkazish
  const activeRef = useRef<HTMLLIElement>(null);
  
  useEffect(() => {
    if (isActive) {
      activeRef.current?.scrollIntoView({ behavior: 'smooth' });
    }
  }, [isActive]);
  
  return (
    <li ref={isActive ? activeRef : undefined} className={isActive ? 'active' : ''}>
      Item {index}
    </li>
  );
}
```

</details>

---

## Ref Cleanup Functions (R19)

### Nazariya

> **🕐 Versiya evolyutsiyasi (Ref cleanup):**
> - **Pre-R19:** Callback ref `null` argument bilan chaqirilardi (component unmount paytida). Imperative cleanup `useEffect` ichida manual qilinardi.
> - **R19:** Callback ref **cleanup function** qaytarishi mumkin — DOM node o'chirilganda chaqiriladi. Effect cleanup pattern bilan teng.

R19'gacha ref callback signature:

```tsx
// Pre-R19
<div ref={(node) => {
  if (node) {
    setup(node);
  } else {
    cleanup();  // Component unmount paytida (node = null)
  }
}} />
```

Bu API noqulay — setup va cleanup ikki tarmoqda. `useEffect` ichida cleanup pattern'i (return function) — clean.

R19'da:

```tsx
// R19 — ref cleanup function
<div ref={(node) => {
  setup(node);
  
  return () => {
    cleanup(node);  // DOM node o'chirilganda
  };
}} />
```

Ref callback `useEffect`'ga o'xshash — setup + cleanup pattern.

**Use case'lar:**

```tsx
function ScrollSync() {
  return (
    <div ref={(node) => {
      if (!node) return;
      
      const onScroll = () => console.log(node.scrollTop);
      node.addEventListener('scroll', onScroll, { passive: true });
      
      return () => {
        node.removeEventListener('scroll', onScroll);
      };
    }}>
      Scrollable content
    </div>
  );
}
```

R19'gacha bu pattern `useEffect + useRef` orqali:

```tsx
// Pre-R19 equivalent
function ScrollSync() {
  const ref = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    const node = ref.current;
    if (!node) return;
    
    const onScroll = () => console.log(node.scrollTop);
    node.addEventListener('scroll', onScroll, { passive: true });
    
    return () => {
      node.removeEventListener('scroll', onScroll);
    };
  }, []);
  
  return <div ref={ref}>Scrollable content</div>;
}
```

R19 ref cleanup — boilerplate kamroq. Lekin har ikkala yondashuv ham ishlaydi.

**Mount/unmount lifecycle:**

R19 callback ref:

```
1. Mount: node yaratiladi → callback(node) chaqiriladi → cleanup function saqlanadi
2. Unmount yoki re-mount: cleanup() chaqiriladi → node = null
3. Re-mount (key change yoki conditional): yangi callback(newNode), yangi cleanup
```

`useEffect` bilan farq:
- `useEffect`: deps o'zgarganda cleanup → setup
- Ref callback: DOM node o'zgarganda cleanup → setup

DOM node ko'pincha bir xil (parent re-render → child DOM saqlanadi). Ref callback faqat node real o'zgarganda re-trigger qilinadi.

**Strict Mode 2x cycle (R18+):**

Ref callback ham Strict Mode'da 2x cycle (mount → cleanup → mount). Cleanup pattern idempotent bo'lishi shart.

<details>
<summary><strong>Under the Hood</strong></summary>

**R19 callback ref signature:**

```ts
type RefCallback<T> = (instance: T | null) => void | (() => void);
```

R18 va undan oldin: `(instance: T | null) => void` (cleanup function support yo'q).

R19'da return type — `void | (() => void)`:
- Return `undefined` (legacy) — cleanup yo'q, unmount paytida `callback(null)` chaqiriladi
- Return function — cleanup function, unmount paytida chaqiriladi (node = null callback chaqirilmaydi)

**React internal:**

```ts
// R19 commitAttachRef (soddalashtirilgan)
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (typeof ref === 'function') {
    const cleanup = ref(finishedWork.stateNode);
    
    if (typeof cleanup === 'function') {
      // R19: cleanup function saqlanadi
      finishedWork.refCleanup = cleanup;
    }
  } else if (ref !== null) {
    ref.current = finishedWork.stateNode;
  }
}

function commitDetachRef(current: Fiber) {
  const ref = current.ref;
  if (typeof ref === 'function') {
    if (current.refCleanup) {
      // R19: saved cleanup chaqiriladi
      current.refCleanup();
    } else {
      ref(null);  // Legacy behavior
    }
  } else if (ref !== null) {
    ref.current = null;
  }
}
```

R19 cleanup function saqlanadi va detach paytida chaqiriladi. Legacy callback (no return) hali ishlaydi (backward compat).

**Source citation:**

- React 19 ref cleanup — React 19 release notes
- `commitAttachRef` / `commitDetachRef` — facebook/react `packages/react-reconciler/src/ReactFiberCommitWork.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Native event listener (R19):**

```tsx
function ScrollableBox() {
  return (
    <div
      ref={(node) => {
        if (!node) return;
        
        const handler = () => {
          console.log('Scroll:', node.scrollTop);
        };
        
        node.addEventListener('scroll', handler, { passive: true });
        
        return () => {
          node.removeEventListener('scroll', handler);
        };
      }}
      style={{ overflow: 'auto', height: 300 }}
    >
      <div style={{ height: 1000 }}>Long content</div>
    </div>
  );
}
```

**Misol 2 — IntersectionObserver:**

```tsx
function LazyImage({ src }: { src: string }) {
  const [loaded, setLoaded] = useState(false);
  
  return (
    <img
      ref={(node) => {
        if (!node || loaded) return;
        
        const observer = new IntersectionObserver(([entry]) => {
          if (entry.isIntersecting) {
            setLoaded(true);
            observer.disconnect();
          }
        });
        
        observer.observe(node);
        
        return () => observer.disconnect();  // R19 cleanup
      }}
      src={loaded ? src : '/placeholder.jpg'}
      alt=""
    />
  );
}
```

**Misol 3 — Pre-R19 vs R19 comparison:**

```tsx
// Pre-R19 (useRef + useEffect)
function ChartPreR19({ data }: { data: number[] }) {
  const containerRef = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    const node = containerRef.current;
    if (!node) return;
    
    const chart = new Chart(node, { data });
    
    return () => chart.destroy();
  }, [data]);
  
  return <div ref={containerRef} />;
}

// R19 (callback ref + cleanup)
function ChartR19({ data }: { data: number[] }) {
  return (
    <div ref={(node) => {
      if (!node) return;
      
      const chart = new Chart(node, { data });
      
      return () => chart.destroy();
    }} />
  );
}
```

R19 versiyasi shorter, lekin har render'da ref callback yangi function (deps array yo'q). Performance ko'pincha sezilmaydi, lekin og'ir setup uchun `useEffect` afzal.

**Misol 4 — Conditional ref cleanup:**

```tsx
function ResizeWatcher({ enabled, onResize }: {
  enabled: boolean;
  onResize: (size: { w: number; h: number }) => void;
}) {
  return (
    <div ref={(node) => {
      if (!node || !enabled) return;
      
      const observer = new ResizeObserver(([entry]) => {
        const { width, height } = entry.contentRect;
        onResize({ w: width, h: height });
      });
      
      observer.observe(node);
      return () => observer.disconnect();
    }}>
      Resizable
    </div>
  );
}

// enabled false bo'lsa — observer setup bo'lmaydi
// enabled true → false: ref callback yangi (deps yo'q) → eski cleanup chaqiriladi
```

**Misol 5 — Legacy fallback (R18 va R19 birga):**

```tsx
// Legacy callback ref — R18+ va R19 ikkalasi
function LegacyRef() {
  return (
    <div ref={(node) => {
      if (node) {
        // Setup (mount)
        console.log('Mounted');
      } else {
        // Cleanup (unmount, node = null)
        console.log('Unmounted');
      }
    }} />
  );
}

// R19 yangi pattern (cleaner)
function ModernRef() {
  return (
    <div ref={(node) => {
      console.log('Mounted');
      return () => console.log('Unmounted');
    }} />
  );
}
```

</details>

---

## `useImperativeHandle` Chuqur

### Nazariya

`useImperativeHandle` — ref orqali parent komponentga **ixtiyoriy imperative API** ekspoz qilish hook'i. Default holat: ref DOM element'ga to'g'ridan-to'g'ri ulanadi. `useImperativeHandle` bilan ref `{ method1, method2 }` ko'rinishidagi obyektga ulanadi.

**Signature:**

```tsx
function useImperativeHandle<T, R extends T>(
  ref: React.Ref<T> | undefined,
  init: () => R,
  deps?: React.DependencyList
): void;
```

- **`ref`** — parent'dan kelgan ref (R18 forwardRef paytida 2-arg, R19 props.ref)
- **`init`** — ref'ga tayinlanadigan obyektni qaytaruvchi function
- **`deps`** — qachon `init` qaytadan chaqirilishi (deps o'zgarganda)

**Use case 1 — Modal API:**

```tsx
type ModalHandle = {
  open: () => void;
  close: () => void;
};

function Modal({ ref, children }: {
  ref?: React.Ref<ModalHandle>;
  children: React.ReactNode;
}) {
  const [isOpen, setIsOpen] = useState(false);
  
  useImperativeHandle(ref, () => ({
    open: () => setIsOpen(true),
    close: () => setIsOpen(false),
  }), []);
  
  if (!isOpen) return null;
  
  return <div className="modal">{children}</div>;
}

// Usage
function App() {
  const modalRef = useRef<ModalHandle>(null);
  
  return (
    <div>
      <button onClick={() => modalRef.current?.open()}>Open</button>
      <Modal ref={modalRef}>
        <button onClick={() => modalRef.current?.close()}>Close</button>
      </Modal>
    </div>
  );
}
```

Parent ref orqali Modal'ni ochadi/yopadi. State Modal'ning ichida — encapsulated. Lekin imperative API parent'ga ekspoz qilingan.

**Use case 2 — Video player API:**

```tsx
type VideoHandle = {
  play: () => void;
  pause: () => void;
  seek: (time: number) => void;
  getCurrentTime: () => number;
};

function VideoPlayer({ ref, src }: {
  ref?: React.Ref<VideoHandle>;
  src: string;
}) {
  const videoRef = useRef<HTMLVideoElement>(null);
  
  useImperativeHandle(ref, () => ({
    play: () => videoRef.current?.play(),
    pause: () => videoRef.current?.pause(),
    seek: (time: number) => {
      if (videoRef.current) videoRef.current.currentTime = time;
    },
    getCurrentTime: () => videoRef.current?.currentTime ?? 0,
  }), []);
  
  return <video ref={videoRef} src={src} />;
}

// Usage
function App() {
  const playerRef = useRef<VideoHandle>(null);
  
  return (
    <div>
      <VideoPlayer ref={playerRef} src="/video.mp4" />
      <button onClick={() => playerRef.current?.play()}>Play</button>
      <button onClick={() => playerRef.current?.seek(60)}>Skip 60s</button>
    </div>
  );
}
```

DOM video element parent'dan yashirin — faqat aniq belgilangan API ko'rinadi.

**Use case 3 — Form imperative submit:**

```tsx
type FormHandle = {
  submit: () => void;
  reset: () => void;
  validate: () => boolean;
};

function ContactForm({ ref }: { ref?: React.Ref<FormHandle> }) {
  const formRef = useRef<HTMLFormElement>(null);
  const [errors, setErrors] = useState<string[]>([]);
  
  useImperativeHandle(ref, () => ({
    submit: () => formRef.current?.requestSubmit(),
    reset: () => {
      formRef.current?.reset();
      setErrors([]);
    },
    validate: () => {
      const isValid = formRef.current?.checkValidity() ?? false;
      // ...
      return isValid;
    },
  }), []);
  
  return (
    <form ref={formRef} onSubmit={(e) => e.preventDefault()}>
      <input name="email" required />
      <input name="message" required />
    </form>
  );
}
```

**`deps` parametri:**

`init` function har render'da qaytadan chaqirilmasligi uchun — deps array:

```tsx
useImperativeHandle(ref, () => ({
  greet: () => alert(`Hello, ${name}`),
}), [name]);  // name o'zgarganda init qaytadan
```

`name` o'zgarsa `init()` chaqiriladi → ref obyekt yangilanadi (yangi reference). Yo'qsa — bir xil obyekt.

**Anti-pattern: declarative bilan hal qilish mumkin bo'lgan narsalarga ishlatmaslik:**

```tsx
// ❌ Anti-pattern: imperative API state uchun
type CounterHandle = {
  increment: () => void;
  getValue: () => number;
};

function Counter({ ref }: { ref?: React.Ref<CounterHandle> }) {
  const [count, setCount] = useState(0);
  
  useImperativeHandle(ref, () => ({
    increment: () => setCount(c => c + 1),
    getValue: () => count,
  }), [count]);
  
  return <div>{count}</div>;
}

// ✅ TO'G'RI: declarative
function Counter({ count, onIncrement }: { count: number; onIncrement: () => void }) {
  return (
    <div>
      <span>{count}</span>
      <button onClick={onIncrement}>+</button>
    </div>
  );
}
```

`useImperativeHandle` faqat **chinakam imperative** (DOM API'lar, focus, scroll, animation, modal trigger) uchun. State sharing → props/lift state up.

**`useImperativeHandle` qachon kerak:**

| Holat | Tanlov |
|-------|--------|
| Parent state ko'rsatadi | Props (declarative) |
| Parent submit/reset trigger qiladi | `useImperativeHandle` |
| Parent focus qo'yadi | `useImperativeHandle` (yoki ref direct) |
| Parent o'lcham o'qiydi | `useImperativeHandle` (yoki ref direct + getBoundingClientRect) |
| Parent video play/pause | `useImperativeHandle` |
| Parent modal toggle | Props yoki `useImperativeHandle` |

<details>
<summary><strong>Under the Hood</strong></summary>

**`useImperativeHandle` implementation:**

```ts
function mountImperativeHandle<T>(
  ref: { current: T } | ((inst: T) => void) | null,
  create: () => T,
  deps: Array<unknown> | null
): void {
  const effectDeps = deps !== null && deps !== undefined ? deps.concat([ref]) : null;
  
  return mountEffectImpl(
    UpdateEffect,
    HookLayout,  // Layout phase'da ishlaydi
    imperativeHandleEffect.bind(null, create, ref),
    effectDeps
  );
}

function imperativeHandleEffect<T>(
  create: () => T,
  ref: { current: T } | ((inst: T) => void) | null
): void | (() => void) {
  if (typeof ref === 'function') {
    const refCallback = ref;
    const inst = create();
    refCallback(inst);
    return () => refCallback(null);
  } else if (ref !== null) {
    const refObject = ref;
    refObject.current = create();
    return () => {
      refObject.current = null;
    };
  }
}
```

`useImperativeHandle` aslida `useLayoutEffect`'ning maxsus shakli. Layout phase'da `init()` chaqiriladi va ref'ga tayinlanadi.

Cleanup esa unmount paytida ref'ni `null` qiladi (DOM ref bilan o'xshash).

**Deps array semantikasi:**

```tsx
useImperativeHandle(ref, () => ({
  greet: () => alert(name),
}), [name]);
```

`name` o'zgarsa:
1. `useLayoutEffect` cleanup chaqiriladi (ref.current = null)
2. `init()` qaytadan chaqiriladi (yangi obyekt)
3. ref.current = yangi obyekt

Parent ref obyekt o'zgarmagan, lekin `current` yangi reference. Parent useEffect deps'da `ref` bo'lsa — trigger qilinmaydi (ref bir xil), `ref.current.greet` reference o'zgarsa — trigger qilinadi (lekin ref.current'ni deps'ga qo'yish anti-pattern).

**Source citation:**

- `mountImperativeHandle` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`
- React docs `useImperativeHandle` — react.dev/reference/react/useImperativeHandle

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Modal handle:**

```tsx
type ModalHandle = {
  open: () => void;
  close: () => void;
};

function Modal({
  ref,
  title,
  children,
}: {
  ref?: React.Ref<ModalHandle>;
  title: string;
  children: React.ReactNode;
}) {
  const [isOpen, setIsOpen] = useState(false);
  
  useImperativeHandle(ref, () => ({
    open: () => setIsOpen(true),
    close: () => setIsOpen(false),
  }), []);
  
  if (!isOpen) return null;
  
  return (
    <div className="modal-backdrop">
      <div className="modal">
        <h2>{title}</h2>
        {children}
        <button onClick={() => setIsOpen(false)}>×</button>
      </div>
    </div>
  );
}

function App() {
  const confirmRef = useRef<ModalHandle>(null);
  const alertRef = useRef<ModalHandle>(null);
  
  return (
    <div>
      <button onClick={() => confirmRef.current?.open()}>Confirm</button>
      <button onClick={() => alertRef.current?.open()}>Alert</button>
      
      <Modal ref={confirmRef} title="Confirm">
        <p>Are you sure?</p>
        <button onClick={() => confirmRef.current?.close()}>OK</button>
      </Modal>
      
      <Modal ref={alertRef} title="Alert">
        <p>Done!</p>
      </Modal>
    </div>
  );
}
```

**Misol 2 — Form handle:**

```tsx
type FormHandle = {
  submit: () => void;
  reset: () => void;
  focus: (field?: 'email' | 'message') => void;
};

function ContactForm({ ref, onSubmit }: {
  ref?: React.Ref<FormHandle>;
  onSubmit: (data: FormData) => void;
}) {
  const formRef = useRef<HTMLFormElement>(null);
  const emailRef = useRef<HTMLInputElement>(null);
  const messageRef = useRef<HTMLTextAreaElement>(null);
  
  useImperativeHandle(ref, () => ({
    submit: () => {
      formRef.current?.requestSubmit();
    },
    reset: () => {
      formRef.current?.reset();
    },
    focus: (field) => {
      if (field === 'email') emailRef.current?.focus();
      else if (field === 'message') messageRef.current?.focus();
      else emailRef.current?.focus();
    },
  }), []);
  
  return (
    <form
      ref={formRef}
      onSubmit={(e) => {
        e.preventDefault();
        onSubmit(new FormData(e.currentTarget));
      }}
    >
      <input ref={emailRef} name="email" type="email" required />
      <textarea ref={messageRef} name="message" required />
    </form>
  );
}

function App() {
  const formRef = useRef<FormHandle>(null);
  
  return (
    <div>
      <ContactForm
        ref={formRef}
        onSubmit={(data) => console.log(data)}
      />
      <button onClick={() => formRef.current?.submit()}>Submit</button>
      <button onClick={() => formRef.current?.reset()}>Reset</button>
      <button onClick={() => formRef.current?.focus('email')}>Focus Email</button>
    </div>
  );
}
```

**Misol 3 — Animated component:**

```tsx
type AnimationHandle = {
  fadeIn: () => Promise<void>;
  fadeOut: () => Promise<void>;
};

function FadeBox({ ref, children }: {
  ref?: React.Ref<AnimationHandle>;
  children: React.ReactNode;
}) {
  const elRef = useRef<HTMLDivElement>(null);
  
  useImperativeHandle(ref, () => ({
    fadeIn: async () => {
      const el = elRef.current;
      if (!el) return;
      
      el.style.opacity = '0';
      el.style.transition = 'opacity 0.3s';
      
      // Force reflow
      void el.offsetHeight;
      
      el.style.opacity = '1';
      
      await new Promise(r => setTimeout(r, 300));
    },
    fadeOut: async () => {
      const el = elRef.current;
      if (!el) return;
      
      el.style.opacity = '0';
      await new Promise(r => setTimeout(r, 300));
    },
  }), []);
  
  return <div ref={elRef}>{children}</div>;
}
```

**Misol 4 — Deps misol:**

```tsx
type CounterHandle = {
  getCount: () => number;
};

function Counter({ ref, multiplier }: {
  ref?: React.Ref<CounterHandle>;
  multiplier: number;
}) {
  const [count, setCount] = useState(0);
  
  useImperativeHandle(ref, () => ({
    getCount: () => count * multiplier,
  }), [count, multiplier]);  // Har deps o'zgarganda init qaytadan
  
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

**Misol 5 — Anti-pattern:**

```tsx
// ❌ ANTI-PATTERN — state imperative
type ToggleHandle = {
  toggle: () => void;
  getValue: () => boolean;
};

function Toggle({ ref }: { ref?: React.Ref<ToggleHandle> }) {
  const [on, setOn] = useState(false);
  
  useImperativeHandle(ref, () => ({
    toggle: () => setOn(o => !o),
    getValue: () => on,
  }), [on]);
  
  return <div>{on ? 'ON' : 'OFF'}</div>;
}

// ✅ TO'G'RI — declarative props
function Toggle({ on, onToggle }: { on: boolean; onToggle: () => void }) {
  return (
    <div>
      <span>{on ? 'ON' : 'OFF'}</span>
      <button onClick={onToggle}>Toggle</button>
    </div>
  );
}
```

State sharing — props/lift state up. `useImperativeHandle` faqat chinakam imperative (DOM, animation, focus) uchun.

</details>

---

## Callback Refs — Dynamic Attachment

### Nazariya

Callback ref — `ref` attribute'ga function (object emas) o'tkazish:

```tsx
<div ref={(node) => {
  // node — DOM element yoki null
}} />
```

Mount paytida React `node` (DOM element) bilan chaqiradi. Unmount paytida `null` bilan chaqiradi (yoki R19'da cleanup function chaqiriladi).

**Object ref vs callback ref:**

| Aspekt | Object ref (`useRef`) | Callback ref |
|--------|----------------------|--------------|
| Yaratish | `useRef(null)` | Inline function |
| Type | `RefObject<T>` | `RefCallback<T>` |
| Re-attach | Faqat element o'zgarsa | Har render'da yangi function bo'lsa |
| Use case | Static element | Dynamic, conditional, multiple elements |
| Performance | Stable reference | Yangi function har render |

**Use case 1 — Conditional ref:**

```tsx
function ConditionalRef({ shouldFocus }: { shouldFocus: boolean }) {
  return (
    <input ref={(node) => {
      if (node && shouldFocus) {
        node.focus();
      }
    }} />
  );
}
```

`shouldFocus` true bo'lganda — focus. False bo'lsa — focus qilinmaydi.

**Use case 2 — Dynamic measurement:**

```tsx
function MeasuredText({ children }: { children: string }) {
  const [width, setWidth] = useState(0);
  
  return (
    <div>
      <span ref={(node) => {
        if (node) {
          setWidth(node.offsetWidth);
        }
      }}>
        {children}
      </span>
      <p>Width: {width}px</p>
    </div>
  );
}
```

Element mount paytida o'lchaladi. Lekin `children` o'zgarsa — yangi callback yangi function (deps yo'q), re-attach trigger qilinadi.

**Use case 3 — Multiple refs (array):**

```tsx
function ListWithRefs({ items }: { items: { id: string; text: string }[] }) {
  const refsMap = useRef<Map<string, HTMLLIElement>>(new Map());
  
  return (
    <ul>
      {items.map(item => (
        <li
          key={item.id}
          ref={(node) => {
            if (node) {
              refsMap.current.set(item.id, node);
            } else {
              refsMap.current.delete(item.id);  // Cleanup
            }
          }}
        >
          {item.text}
        </li>
      ))}
    </ul>
  );
}
```

Map ichida har item uchun DOM ref. Dynamic — items o'zgarsa, refs avtomatik update.

R19 callback ref cleanup pattern:

```tsx
ref={(node) => {
  refsMap.current.set(item.id, node);
  return () => {
    refsMap.current.delete(item.id);
  };
}}
```

**Re-attach problem (legacy):**

Inline callback ref har render'da yangi function:

```tsx
function Component() {
  return (
    <div ref={(node) => console.log(node)}>
      Element
    </div>
  );
}

// Har render:
// 1. Old ref(null) — cleanup
// 2. New ref(node) — attach
```

Bu wasted work bo'lishi mumkin (e.g., expensive setup). `useCallback` bilan stabilize:

```tsx
function Component() {
  const setRef = useCallback((node: HTMLDivElement | null) => {
    console.log(node);
  }, []);
  
  return <div ref={setRef}>Element</div>;
}
```

`setRef` har render'da bir xil reference — re-attach yo'q.

**Combined object + callback ref:**

```tsx
function CombinedRef() {
  const ref = useRef<HTMLDivElement>(null);
  
  return (
    <div ref={(node) => {
      ref.current = node;       // Object ref'ga set
      
      if (node) {
        // Callback logic
        console.log('Mounted');
      }
    }} />
  );
}
```

Bu pattern — library author'lar ref'ni o'z ichida ishlatish va caller'ga ham bermoqchi bo'lganda. R18'gacha boshqa hooks (e.g., useImperativeHandle) bilan ham aralashardi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Callback ref invocation timing:**

```ts
// Commit Phase
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    const instance = finishedWork.stateNode;
    
    if (typeof ref === 'function') {
      const cleanup = ref(instance);
      // R19: cleanup function support
      if (typeof cleanup === 'function') {
        finishedWork.refCleanup = cleanup;
      }
    } else {
      ref.current = instance;
    }
  }
}
```

Callback ref `commitAttachRef`'da chaqiriladi — Layout sub-phase. `useLayoutEffect`'dan **oldin**.

**Re-attach mexanikasi:**

```ts
// Commit Phase (deps o'zgarganda)
if (oldRef !== newRef) {
  if (typeof oldRef === 'function') {
    oldRef(null);  // Old detach
  } else {
    oldRef.current = null;
  }
  
  if (typeof newRef === 'function') {
    newRef(node);  // New attach
  } else {
    newRef.current = node;
  }
}
```

Inline callback har render'da yangi function (`!==` true) → har render'da re-attach. `useCallback` bilan stable reference — re-attach faqat node o'zgarganda.

**Performance comparison:**

```
Inline callback (har render yangi):
  Render → ref(null) → ref(node) — 2x callback call
  100 element list, 10 render = 1000 callback call

Stable callback (useCallback):
  Render → no re-attach (ref bir xil) — 0 callback call
  100 element list, 10 render = 0 callback call
```

Stable ref ko'p hollarda afzal, ayniqsa setup expensive bo'lsa.

**Source citation:**

- React docs "Callback refs" — react.dev/reference/react-dom/components/common#ref-callback
- `commitAttachRef` source — facebook/react

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Auto-focus on conditional mount:**

```tsx
function SearchModal({ isOpen }: { isOpen: boolean }) {
  if (!isOpen) return null;
  
  return (
    <div className="modal">
      <input
        ref={(node) => {
          if (node) {
            node.focus();  // Mount paytida darrov
          }
        }}
        placeholder="Search..."
      />
    </div>
  );
}
```

`isOpen` true bo'lganda input mount → callback chaqiriladi → focus.

**Misol 2 — Dynamic list refs:**

```tsx
function TabList({ tabs, activeId }: {
  tabs: { id: string; label: string }[];
  activeId: string;
}) {
  const tabRefs = useRef<Map<string, HTMLButtonElement>>(new Map());
  
  useEffect(() => {
    const activeTab = tabRefs.current.get(activeId);
    activeTab?.scrollIntoView({ inline: 'center', behavior: 'smooth' });
  }, [activeId]);
  
  return (
    <div role="tablist">
      {tabs.map(tab => (
        <button
          key={tab.id}
          role="tab"
          aria-selected={tab.id === activeId}
          ref={(node) => {
            if (node) {
              tabRefs.current.set(tab.id, node);
            }
            return () => {
              tabRefs.current.delete(tab.id);  // R19 cleanup
            };
          }}
        >
          {tab.label}
        </button>
      ))}
    </div>
  );
}
```

**Misol 3 — Stable callback (useCallback):**

```tsx
function Heavy({ onMount }: { onMount: (node: HTMLDivElement) => void }) {
  const setRef = useCallback((node: HTMLDivElement | null) => {
    if (node) {
      onMount(node);  // Faqat real attach paytida
    }
  }, [onMount]);
  
  return <div ref={setRef}>Heavy Component</div>;
}
```

`useCallback` setRef'ni stabilize qiladi. Re-attach faqat `onMount` o'zgarganda.

**Misol 4 — Combined object + callback:**

```tsx
type ResizeHandlerProps = {
  onResize: (size: { w: number; h: number }) => void;
  ref?: React.Ref<HTMLDivElement>;
};

function ResizeHandler({ onResize, ref }: ResizeHandlerProps) {
  const internalRef = useRef<HTMLDivElement>(null);
  
  return (
    <div
      ref={(node) => {
        // Internal ref
        internalRef.current = node;
        
        // External ref (parent'dan)
        if (typeof ref === 'function') {
          ref(node);
        } else if (ref) {
          ref.current = node;
        }
        
        // Callback logic
        if (node) {
          const observer = new ResizeObserver(([entry]) => {
            onResize({ w: entry.contentRect.width, h: entry.contentRect.height });
          });
          observer.observe(node);
          
          return () => observer.disconnect();
        }
      }}
    />
  );
}
```

**Misol 5 — Callback ref with state setter:**

```tsx
function DivSize() {
  const [size, setSize] = useState<DOMRect | null>(null);
  
  return (
    <div
      ref={(node) => {
        if (node) {
          setSize(node.getBoundingClientRect());
        }
      }}
      style={{ padding: 20, border: '1px solid' }}
    >
      Size: {size?.width}x{size?.height}
    </div>
  );
}
```

setState ref callback ichidan — Strict Mode'da 2x trigger bo'lishi mumkin. Idempotent setter (`setSize(...)` bir xil rect bilan) ehtiyot bo'lish.

</details>

---

## TypeScript Patterns

### Nazariya

`useRef` TypeScript bilan — ko'p edge case'lar. Aniq pattern'lar:

**Pattern 1 — DOM ref:**

```tsx
const inputRef = useRef<HTMLInputElement>(null);
// Type: RefObject<HTMLInputElement>
// inputRef.current — HTMLInputElement | null
```

`null` initial — TypeScript `RefObject` overload tanlaydi. `current` `readonly` (lekin React mutate qiladi). DOM elementlar uchun standart.

**Pattern 2 — Mutable value (initial bilan):**

```tsx
const countRef = useRef(0);
// Type: MutableRefObject<number>
// countRef.current — number, mutable
```

Initial value type'ni o'rnatadi.

**Pattern 3 — Mutable value (initial yo'q):**

```tsx
// R19+ (@types/react ≥ 19) — explicit `undefined` shart
const observerRef = useRef<IntersectionObserver | undefined>(undefined);
// Type: MutableRefObject<IntersectionObserver | undefined>

// Pre-R19 — argumentsiz forma ham ishlardi (endi xato):
// const observerRef = useRef<IntersectionObserver>();
```

Generic explicit, initial `undefined` → `T | undefined`. R19'da argumentsiz overload olib tashlanganligi sababli initial qiymat majburiy.

**Pattern 4 — Element from `as` prop (R19):**

```tsx
function Button<E extends React.ElementType = 'button'>({
  as,
  ref,
  ...props
}: {
  as?: E;
  ref?: React.Ref<React.ElementRef<E>>;
}) {
  const Component = as ?? 'button';
  return <Component ref={ref} {...props} />;
}

// Usage
const buttonRef = useRef<HTMLButtonElement>(null);
<Button ref={buttonRef} />

const linkRef = useRef<HTMLAnchorElement>(null);
<Button as="a" ref={linkRef} />
```

`React.ElementRef<E>` — element tipiga mos ref tipi.

**Pattern 5 — `forwardRef` typing (R18):**

```tsx
import { forwardRef, type ForwardedRef } from 'react';

interface InputProps {
  label: string;
}

const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label }, ref) => {
    return (
      <label>
        {label}
        <input ref={ref} />
      </label>
    );
  }
);
```

`forwardRef<RefType, PropsType>` — ikki generic. `ref` ikkinchi argument `ForwardedRef<RefType>`.

**Pattern 6 — `useImperativeHandle` typing:**

```tsx
type ModalHandle = {
  open: () => void;
  close: () => void;
};

function Modal({ ref }: { ref?: React.Ref<ModalHandle> }) {
  const [isOpen, setIsOpen] = useState(false);
  
  useImperativeHandle(ref, (): ModalHandle => ({
    open: () => setIsOpen(true),
    close: () => setIsOpen(false),
  }), []);
  
  return isOpen ? <div>Modal</div> : null;
}

// Usage
const modalRef = useRef<ModalHandle>(null);
<Modal ref={modalRef} />
modalRef.current?.open();
```

`init` function return type explicit annotate qilish — TypeScript inference yaxshilanadi.

**Pattern 7 — Generic ref hook:**

```tsx
function useResizeObserver<T extends HTMLElement>(): [
  React.RefObject<T>,
  { width: number; height: number }
] {
  const ref = useRef<T>(null);
  const [size, setSize] = useState({ width: 0, height: 0 });
  
  useEffect(() => {
    if (!ref.current) return;
    
    const observer = new ResizeObserver(([entry]) => {
      setSize({ width: entry.contentRect.width, height: entry.contentRect.height });
    });
    
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);
  
  return [ref, size];
}

// Usage
const [divRef, divSize] = useResizeObserver<HTMLDivElement>();
<div ref={divRef}>Size: {divSize.width}x{divSize.height}</div>
```

**Pattern 8 — Ref callback typing:**

```tsx
const setRef: React.RefCallback<HTMLInputElement> = useCallback((node) => {
  if (node) {
    node.focus();
  }
}, []);

<input ref={setRef} />
```

`React.RefCallback<T>` — `(instance: T | null) => void | (() => void)` (R19'da cleanup function).

**Pattern 9 — Union ref types:**

```tsx
function MyComponent({
  ref,
}: {
  ref?: React.Ref<HTMLInputElement> | React.Ref<HTMLTextAreaElement>;
}) {
  // ⚠️ Union ref complex, ko'pincha conditional render bilan
}

// Yaxshi yondashuv — discriminated:
type Props =
  | { multiline?: false; ref?: React.Ref<HTMLInputElement> }
  | { multiline: true; ref?: React.Ref<HTMLTextAreaElement> };

function Field(props: Props) {
  if (props.multiline) {
    return <textarea ref={props.ref} />;
  }
  return <input ref={props.ref} />;
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

**TypeScript overload resolution:**

```ts
// useRef overloads (DefinitelyTyped, R18 davri — 3 ta overload)
function useRef<T>(initialValue: T): MutableRefObject<T>;             // 1
function useRef<T>(initialValue: T | null): RefObject<T>;             // 2
function useRef<T = undefined>(): MutableRefObject<T | undefined>;    // 3 — R19'DA OLIB TASHLANDI
```

Tartibi muhim — TypeScript birinchi mos overload tanlaydi:

- `useRef(0)` → 1: `MutableRefObject<number>` (initialValue type number)
- `useRef<HTMLInputElement>(null)` → 2: `RefObject<HTMLInputElement>` (null mos kelishi uchun explicit T)
- Pre-R19: `useRef<number>()` → 3: `MutableRefObject<number | undefined>` (initialValue yo'q)
- R19+: argumentsiz forma TS xato, `useRef<number | undefined>(undefined)` ishlatilishi shart

**`MutableRefObject` vs `RefObject`:**

```ts
interface MutableRefObject<T> {
  current: T;  // mutable
}

interface RefObject<T> {
  readonly current: T | null;  // readonly
}
```

`readonly` — TypeScript compile-time check. Runtime'da har ikki tip mutable. `RefObject` semantik signal: "DOM ref'lar uchun, React mutation qiladi".

**`React.ElementRef<E>` complex inference:**

```ts
type ElementRef<E> =
  E extends React.ComponentType<infer P>
    ? P extends { ref?: React.Ref<infer R> } ? R : never
    : E extends keyof JSX.IntrinsicElements
    ? JSX.IntrinsicElements[E] extends React.DetailedHTMLProps<infer A, infer R> ? R : never
    : never;
```

3 tarmoq:
1. Component (function/class) — props ichidan ref tipini olish
2. Intrinsic element ('div', 'button') — IntrinsicElements mapping'dan ref
3. Boshqa — never (xato)

Bu tip polymorphic component'lar uchun kritik.

**Source citation:**

- React @types — DefinitelyTyped `types/react/index.d.ts`
- React TS handbook — react.dev/learn/typescript

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Type assertions:**

```tsx
// ❌ Type assertion — kerak emas
const ref = useRef<HTMLInputElement | null>(null) as React.MutableRefObject<HTMLInputElement | null>;

// ✅ To'g'ri overload
const ref = useRef<HTMLInputElement>(null);  // RefObject<HTMLInputElement>
```

**Misol 2 — Ref initial undefined vs null:**

```tsx
// undefined initial — DOM ref uchun emas (R19+ explicit undefined shart)
const observerRef = useRef<IntersectionObserver | undefined>(undefined);
// Type: MutableRefObject<IntersectionObserver | undefined>

// null initial — DOM ref uchun
const inputRef = useRef<HTMLInputElement>(null);
// Type: RefObject<HTMLInputElement>
```

**Misol 3 — Generic hook:**

```tsx
function useFocus<T extends HTMLElement>(): React.RefObject<T> {
  const ref = useRef<T>(null);
  
  useEffect(() => {
    ref.current?.focus();
  }, []);
  
  return ref;
}

const inputRef = useFocus<HTMLInputElement>();
<input ref={inputRef} />

const buttonRef = useFocus<HTMLButtonElement>();
<button ref={buttonRef}>Auto-focused</button>
```

**Misol 4 — `useImperativeHandle` strict typing:**

```tsx
type FormHandle = {
  submit: () => void;
  validate: () => boolean;
  getData: () => FormData | null;
};

interface FormProps {
  ref?: React.Ref<FormHandle>;
  onSubmit?: (data: FormData) => void;
}

function Form({ ref, onSubmit }: FormProps) {
  const formRef = useRef<HTMLFormElement>(null);
  
  // Explicit return type — ensures all methods present
  useImperativeHandle(ref, (): FormHandle => ({
    submit: () => formRef.current?.requestSubmit(),
    validate: () => formRef.current?.checkValidity() ?? false,
    getData: () => formRef.current ? new FormData(formRef.current) : null,
  }), []);
  
  return (
    <form
      ref={formRef}
      onSubmit={(e) => {
        e.preventDefault();
        if (onSubmit && formRef.current) {
          onSubmit(new FormData(formRef.current));
        }
      }}
    >
      <input name="email" required />
    </form>
  );
}
```

**Misol 5 — R18 vs R19 typing:**

```tsx
// R18 (forwardRef)
interface InputProps {
  label: string;
}

const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label }, ref) => (
    <label>
      {label}
      <input ref={ref} />
    </label>
  )
);

// R19 (ref as prop) — type cleaner
interface InputPropsR19 {
  label: string;
  ref?: React.Ref<HTMLInputElement>;
}

function InputR19({ label, ref }: InputPropsR19) {
  return (
    <label>
      {label}
      <input ref={ref} />
    </label>
  );
}

// R19 with ComponentProps:
function InputComponentProps({
  label,
  ...props
}: { label: string } & React.ComponentProps<'input'>) {
  return (
    <label>
      {label}
      <input {...props} />
    </label>
  );
}
```

`React.ComponentProps<'input'>` R19'da ref'ni o'z ichiga oladi. Spread bilan automatic forward.

</details>

---

## Edge Cases va Gotchas

### Gotcha 1 — `useRef` `initialValue` faqat birinchi render'da

```tsx
function Component({ initialValue }: { initialValue: number }) {
  const ref = useRef(initialValue);
  
  // initialValue prop o'zgarsa ref.current o'zgarmaydi
  console.log(ref.current);  // Birinchi render qiymati
}

// Render 1: initialValue=5 → ref.current=5
// Render 2: initialValue=10 → ref.current=5 (eski)
```

`useRef` initial value faqat mount paytida ishlatiladi. Keyingi render'lar uchun `useEffect` ichida manual sync kerak:

```tsx
useEffect(() => {
  ref.current = initialValue;
}, [initialValue]);
```

### Gotcha 2 — Ref render paytida o'qish — Strict Mode

```tsx
function Component() {
  const ref = useRef(0);
  
  // ❌ Render paytida read va write
  ref.current++;
  
  return <div>{ref.current}</div>;
}

// Strict Mode (R16.3+):
// Render 1.1: ref.current 0→1
// Render 1.2: ref.current 1→2
// Output: 2 (production'da 1)
```

Render purity buzilishi → Strict Mode 2x render'da bug ko'rinadi. Production'da silent — keyingi versiya'larda buzilishi mumkin.

To'g'ri pattern — mutation effect/event handler ichida.

### Gotcha 3 — Conditional ref attach — DOM null

```tsx
function Component({ show }: { show: boolean }) {
  const ref = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    console.log(ref.current);  // null bo'lishi mumkin (show=false)
  }, [show]);
  
  return show ? <div ref={ref}>Visible</div> : <div>Hidden</div>;
  // Hidden div'da ref={ref} yo'q → ref.current null
}
```

Conditional render — ref har gal element bo'lmasligi mumkin. Effect ichida null check shart.

### Gotcha 4 — Ref deps array'da — hech qachon trigger qilmaydi

```tsx
const ref = useRef(0);

useEffect(() => {
  console.log('Effect');
}, [ref]);  // ❌ Hech qachon trigger qilinmaydi (ref bir xil reference)

// To'g'ri — ref.current'ga react qilish:
useEffect(() => {
  console.log(ref.current);
}, [ref.current]);  // ⚠️ ref.current mutation re-render trigger qilmaydi → effect ham
```

Ref deps'da kerak emas. `ref.current` ham reactive emas. Re-render trigger uchun `useState` ishlatish.

### Gotcha 5 — `useImperativeHandle` deps array'siz

```tsx
function Component({ ref, value }: { ref?: React.Ref<{ get: () => number }>; value: number }) {
  useImperativeHandle(ref, () => ({
    get: () => value,
  }));  // ❌ Deps yo'q — har render'da yangi obyekt
}

// Har render'da ref.current yangi reference
// Parent useEffect deps'da get bo'lsa — har gal trigger
```

Deps array har doim explicit (bo'sh `[]` yoki aniq deps). Yo'q bo'lsa har render'da init qaytadan chaqiriladi.

---

## Common Mistakes

### ❌ Xato 1 — UI'da `ref.current` ishlatish

```tsx
// ❌ UI re-render kutilmaydi
function Counter() {
  const count = useRef(0);
  
  return (
    <button onClick={() => count.current++}>
      {count.current}
    </button>
  );
}

// ✅ useState
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

UI'da ko'rsatiladigan qiymat har doim `useState`.

### ❌ Xato 2 — Render paytida ref mutation

```tsx
// ❌ Render purity buzilishi
function Component() {
  const ref = useRef(0);
  ref.current++;  // ❌ Strict Mode'da 2x
  
  return <div>{ref.current}</div>;
}

// ✅ Effect/event handler ichida
function Component() {
  const ref = useRef(0);
  
  useEffect(() => {
    ref.current++;
  });
  
  return <div>{ref.current}</div>;
}
```

### ❌ Xato 3 — `ref.current.method()` null check yo'q

```tsx
// ❌ Possible null
function Component() {
  const ref = useRef<HTMLInputElement>(null);
  
  useEffect(() => {
    ref.current.focus();  // ❌ TS error, runtime null possible
  }, []);
  
  return <input ref={ref} />;
}

// ✅ Optional chaining
useEffect(() => {
  ref.current?.focus();
}, []);

// ✅ Type guard
useEffect(() => {
  if (ref.current) {
    ref.current.focus();
  }
}, []);
```

### ❌ Xato 4 — `useImperativeHandle` deps yo'q

```tsx
// ❌ Har render'da init qaytadan
function Component({ ref, value }: Props) {
  useImperativeHandle(ref, () => ({
    get: () => value,
  }));  // Deps yo'q
}

// ✅ Deps explicit
useImperativeHandle(ref, () => ({
  get: () => value,
}), [value]);
```

### ❌ Xato 5 — Inline callback ref har render'da yangi

```tsx
// ❌ Setup expensive — har render'da re-attach
function HeavyComponent() {
  return (
    <div ref={(node) => {
      if (node) {
        expensiveSetup(node);  // ❌ Har render'da
      }
    }} />
  );
}

// ✅ useCallback bilan stabilize
function HeavyComponent() {
  const setRef = useCallback((node: HTMLDivElement | null) => {
    if (node) {
      expensiveSetup(node);
    }
  }, []);
  
  return <div ref={setRef} />;
}
```

---

## Amaliy Mashqlar

### Mashq 1 — `usePrevious` Hook (Oson)

Avvalgi render qiymatini qaytaruvchi hook yozing.

```tsx
function usePrevious<T>(value: T): T | undefined {
  // Implement
}

function PriceTracker({ price }: { price: number }) {
  const prev = usePrevious(price);
  return <div>Now: {price}, Was: {prev ?? '—'}</div>;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  
  useEffect(() => {
    ref.current = value;
  });
  
  return ref.current;
}
```

**Tushuntirish:**

- `useRef` mutable container, re-render trigger qilmaydi
- `useEffect` har render'dan keyin chaqiriladi (deps yo'q)
- Function body'da `ref.current` o'qish — hali avvalgi qiymat (effect render'dan keyin)

Lifecycle:
- Render 1, value=A: `ref.current=undefined`, return undefined
- Effect 1: `ref.current=A`
- Render 2, value=B: `ref.current=A`, return A
- Effect 2: `ref.current=B`

</details>

### Mashq 2 — `useFocusOnMount` Hook (Oson)

Komponent mount paytida element'ni focus qiluvchi hook yozing.

```tsx
function useFocusOnMount<T extends HTMLElement>(): React.RefObject<T> {
  // Implement
}

function LoginForm() {
  const emailRef = useFocusOnMount<HTMLInputElement>();
  return <input ref={emailRef} type="email" />;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useFocusOnMount<T extends HTMLElement>(): React.RefObject<T> {
  const ref = useRef<T>(null);
  
  useEffect(() => {
    ref.current?.focus();
  }, []);
  
  return ref;
}
```

**Tushuntirish:**

- Generic `<T extends HTMLElement>` — har xil element type'lar uchun
- `useRef<T>(null)` → `RefObject<T>`
- `useEffect` empty deps — faqat mount paytida
- Optional chaining — ref.current bo'lmasa silent skip

`useLayoutEffect` ham ishlaydi — focus paint dan oldin (cross-ref [`17-uselayouteffect.md`](17-uselayouteffect.md)).

</details>

### Mashq 3 — `useImperativeHandle` Modal API (O'rta)

Modal komponent yozing — ref orqali `open()` va `close()` ekspoz qilsin.

```tsx
type ModalHandle = {
  open: () => void;
  close: () => void;
};

function Modal({
  ref,
  children,
}: {
  ref?: React.Ref<ModalHandle>;
  children: React.ReactNode;
}) {
  // Implement
}

function App() {
  const modalRef = useRef<ModalHandle>(null);
  
  return (
    <div>
      <button onClick={() => modalRef.current?.open()}>Open</button>
      <Modal ref={modalRef}>
        <h2>Modal Content</h2>
      </Modal>
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function Modal({
  ref,
  children,
}: {
  ref?: React.Ref<ModalHandle>;
  children: React.ReactNode;
}) {
  const [isOpen, setIsOpen] = useState(false);
  
  useImperativeHandle(ref, (): ModalHandle => ({
    open: () => setIsOpen(true),
    close: () => setIsOpen(false),
  }), []);
  
  if (!isOpen) return null;
  
  return (
    <div className="modal-backdrop" onClick={() => setIsOpen(false)}>
      <div className="modal" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>
  );
}
```

**Tushuntirish:**

- `useState` — internal state (isOpen)
- `useImperativeHandle` — parent uchun open/close API
- Deps `[]` — `setIsOpen` stable, init bir marta
- Backdrop click → close (UX)
- `e.stopPropagation()` — modal ichida click backdrop'ga propagate qilmaydi

R19'da `ref` oddiy prop. R18'da `forwardRef` wrapper kerak edi.

</details>

### Mashq 4 — `useElementsRef` Map Hook (O'rta)

Dynamic list element'lariga ref'larni Map'da saqlovchi hook yozing.

```tsx
function useElementsRef<T extends HTMLElement>(): {
  setRef: (id: string) => (node: T | null) => void;
  getRef: (id: string) => T | null;
  refsMap: React.MutableRefObject<Map<string, T>>;
} {
  // Implement
}

function ScrollableList({ items }: { items: { id: string; text: string }[] }) {
  const { setRef, getRef } = useElementsRef<HTMLLIElement>();
  
  const scrollTo = (id: string) => {
    getRef(id)?.scrollIntoView({ behavior: 'smooth' });
  };
  
  return (
    <>
      {items.map(item => (
        <button key={item.id} onClick={() => scrollTo(item.id)}>
          {item.text}
        </button>
      ))}
      <ul>
        {items.map(item => (
          <li key={item.id} ref={setRef(item.id)}>
            {item.text}
          </li>
        ))}
      </ul>
    </>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useElementsRef<T extends HTMLElement>() {
  const refsMap = useRef<Map<string, T>>(new Map());
  
  const setRef = useCallback((id: string) => {
    return (node: T | null) => {
      if (node) {
        refsMap.current.set(id, node);
      } else {
        refsMap.current.delete(id);
      }
    };
  }, []);
  
  const getRef = useCallback((id: string): T | null => {
    return refsMap.current.get(id) ?? null;
  }, []);
  
  return { setRef, getRef, refsMap };
}
```

**Tushuntirish:**

- `useRef<Map<string, T>>` — Map ichida har item ref
- `setRef` — callback ref factory, id'ga bog'langan
- Mount: `refsMap.current.set(id, node)`
- Unmount: `refsMap.current.delete(id)` (R19 cleanup function alternative)
- `useCallback` — `setRef` reference stable bo'lishi uchun (re-attach yo'q)

R19 cleanup function pattern:

```tsx
const setRef = useCallback((id: string) => {
  return (node: T | null) => {
    if (node) refsMap.current.set(id, node);
    return () => refsMap.current.delete(id);
  };
}, []);
```

</details>

### Mashq 5 — `useDraggable` Hook (Qiyin)

Element'ni draggable qiluvchi hook yozing — mouse drag bilan position o'zgartirish.

```tsx
function useDraggable(): {
  ref: React.RefObject<HTMLDivElement>;
  position: { x: number; y: number };
} {
  // Implement
}

function DraggableBox() {
  const { ref, position } = useDraggable();
  
  return (
    <div
      ref={ref}
      style={{
        position: 'absolute',
        left: position.x,
        top: position.y,
        width: 100,
        height: 100,
        background: 'red',
        cursor: 'grab',
      }}
    >
      Drag me
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useDraggable(): {
  ref: React.RefObject<HTMLDivElement>;
  position: { x: number; y: number };
} {
  const ref = useRef<HTMLDivElement>(null);
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  // Drag state — mutable, no re-render
  const dragState = useRef<{
    isDragging: boolean;
    startX: number;
    startY: number;
    initialX: number;
    initialY: number;
  }>({
    isDragging: false,
    startX: 0,
    startY: 0,
    initialX: 0,
    initialY: 0,
  });
  
  useEffect(() => {
    const element = ref.current;
    if (!element) return;
    
    const handleMouseDown = (e: MouseEvent) => {
      dragState.current = {
        isDragging: true,
        startX: e.clientX,
        startY: e.clientY,
        initialX: position.x,
        initialY: position.y,
      };
      element.style.cursor = 'grabbing';
    };
    
    const handleMouseMove = (e: MouseEvent) => {
      if (!dragState.current.isDragging) return;
      
      const dx = e.clientX - dragState.current.startX;
      const dy = e.clientY - dragState.current.startY;
      
      setPosition({
        x: dragState.current.initialX + dx,
        y: dragState.current.initialY + dy,
      });
    };
    
    const handleMouseUp = () => {
      if (dragState.current.isDragging) {
        dragState.current.isDragging = false;
        element.style.cursor = 'grab';
      }
    };
    
    element.addEventListener('mousedown', handleMouseDown);
    window.addEventListener('mousemove', handleMouseMove);
    window.addEventListener('mouseup', handleMouseUp);
    
    return () => {
      element.removeEventListener('mousedown', handleMouseDown);
      window.removeEventListener('mousemove', handleMouseMove);
      window.removeEventListener('mouseup', handleMouseUp);
    };
  }, [position.x, position.y]);
  
  return { ref, position };
}
```

**Tushuntirish:**

- `useRef<HTMLDivElement>` — element ref
- `useState` — position UI'da ko'rsatiladi (left, top)
- `useRef` — drag state (isDragging, start coords) UI'da emas, re-render kerak emas
- mousedown — drag boshlanadi, initial coords saqlash
- mousemove (window'da) — drag davomida position update
- mouseup — drag tugaydi
- Cleanup — listener'lar olib tashlanadi

`position.x`/`position.y` deps'da bo'lgani uchun har drag'da effect re-attach. Optimallashtirish uchun functional update + ref pattern ishlatish mumkin.

Production'da Pointer Events (touch + mouse + pen) afzal — `pointerdown`, `pointermove`, `pointerup`, `setPointerCapture` (cross-ref [`13-event-handling.md`](13-event-handling.md)).

</details>

---

## Xulosa

`useRef` — React'ning mutable container hook'i. Asosiy fikrlar:

- **`useRef` `{ current: T }` qaytaradi** — re-render orasida saqlanadi (bir xil reference), `current` mutation re-render trigger qilmaydi.
- **Ikki katta use case:** (1) DOM refs — element'larga imperative kirish (focus, scroll, measurement, video controls), (2) Mutable values — timer ID, latest closure, prev value, singleton, mount tracker.
- **`ref` vs `state` Decision Guide:** UI'da ko'rsatiladi yoki state mutation re-render trigger qilishi kerak — `useState`. Re-render trigger qilmaydigan, internal mutable qiymat — `useRef`.
- **String refs versiya tarixi** (Versiya callout): Pre-R16 `<input ref="myInput" />` + `this.refs.myInput` → R16+ `createRef`/`useRef` modern → R19'da string refs to'liq olib tashlandi. Sabab: Concurrent Mode incompatible, type-unsafe, hidden state.
- **`forwardRef` evolyutsiyasi** (Versiya callout): Pre-R16.3 function component'lar ref qabul qilmasdi → R16.3 `forwardRef(...)` wrapper introduced → R19 `ref` oddiy prop, `forwardRef` deprecated emas (gradually phased out). Sabab: ortiqcha boilerplate, JSX transform avtomatik.
- **R19 ref oddiy prop** — function component'lar `props.ref`'ni qabul qiladi, JSX transform ref'ni boshqa props bilan birga o'tkazadi. `React.ComponentProps<E>` R19'da ref ham ichkariga kiritilgan.
- **Ref cleanup functions (R19)** (Versiya callout): Pre-R19 callback ref `null` argument bilan unmount paytida → R19 callback **cleanup function qaytarishi mumkin** (DOM node o'chirilganda). `useEffect` cleanup pattern bilan teng. Backward compat — legacy callback hali ishlaydi.
- **`useImperativeHandle`** — ref orqali parent'ga ixtiyoriy imperative API ekspoz qilish. Use case'lar: Modal open/close, video player play/pause/seek, form submit/reset/validate, animation. Anti-pattern: declarative bilan hal qilinishi mumkin bo'lgan narsalarga ishlatmaslik. Deps array har doim explicit.
- **Callback refs** — function `ref` attribute'ga. Use case'lar: conditional ref, dynamic measurement, multiple refs (Map). Inline callback har render'da yangi → `useCallback` bilan stabilize. R19 cleanup function pattern.
- **TypeScript:** `useRef<HTMLInputElement>(null)` → `RefObject`, `useRef(0)` → `MutableRefObject`. R19 da argumentsiz overload (`useRef<T>()`) olib tashlandi — endi `useRef<T | undefined>(undefined)` shart. R19 polymorphic `React.ElementRef<E>`, `React.ComponentProps<E>` ref bilan.
- **Render purity** — render paytida `ref.current = X` mutation taqiq (Strict Mode 2x render'da bug ko'rinadi). Mutation faqat event handler/effect ichida. Lazy init exception (pure construction OK).
- **Performance** — `useRef` O(1) mutation, deps array stable reference, `useState` O(N) cycle. UI sync — `useState`. Internal mutable — `useRef`.
- **Under the Hood** — Hook obyekt `memoizedState = { current: T }`, `queue = null`. `mountRef` initial value bilan, `updateRef` eski ref qaytaradi (initial parametri ignore). Commit Phase'da ref attach (Layout sub-phase) va detach (Mutation sub-phase).

Keyingi bo'lim: `useContext` — Provider/Consumer pattern, prop drilling muammosini hal qilish, performance optimization (re-render scope, splitted contexts, memoizing Provider value), R19 `<Context value>` syntax (Provider qisqartirildi).

---

**Keyingi bo'lim:** [19-usecontext.md](19-usecontext.md) — `useContext` — prop drilling muammosi, `createContext` + Provider + `useContext`, default value, multiple contexts composition, **R19 `<Context value={...}>`** as provider (`<Context.Provider>` qisqartirildi), **R19 `use(context)`** conditional context reading (Rules of Hooks bilan farq), performance (har consumer re-render Provider value o'zgarsa, reference identity object value gotcha, splitted contexts state vs dispatch, memoizing Provider value), selector pattern (`use-context-selector` library mention), Decision Guide when Context vs state library, **legacy Context API → modern Context** versiya callout, **`<Context.Provider>` → `<Context>`** R19 versiya callout, **`use(context)` R19** conditional reading versiya callout.
