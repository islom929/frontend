# Bo'lim 21: useMemo va useCallback

> `useMemo` va `useCallback` — React'ning memoization hook'lari. `useMemo` computed value'ning **referential identity**'sini, `useCallback` esa function'ning identity'sini render orasida stabil tutadi. Texnik jihatdan ikki hook ekvivalent: `useCallback(fn, deps) ≡ useMemo(() => fn, deps)`. Bu bo'lim **mexanika va API**'ga fokus qilinadi — `Hook.memoizedState`'da deps + value tuple saqlash, `Object.is` deps comparison, `areHookInputsEqual` algorithm. **Real qo'llash patternlari [`33-optimization.md`](33-optimization.md)** da chuqur (React.memo bilan kombinatsiya, optimization strategiyalari). **React Compiler ta'siri** (1.0 stable 2025, React 17 va undan yuqori versiyalar bilan mos) — auto-memoization (manual `useMemo`/`useCallback` kerak emas bo'lib qolishi) [`31-react-compiler.md`](31-react-compiler.md) da batafsil.

---

## Mundarija

- [Memoization Concept](#memoization-concept)
- [`useMemo` API](#usememo-api)
- [`useCallback` API](#usecallback-api)
- [Texnik Ekvivalentlik](#texnik-ekvivalentlik)
- [Mexanika — `Hook.memoizedState` Tuple](#mexanika--hookmemoizedstate-tuple)
- [`Object.is` Deps Comparison](#objectis-deps-comparison)
- [When to Use — `React.memo` bilan](#when-to-use--reactmemo-bilan)
- [When to Use — Dependency Arrays](#when-to-use--dependency-arrays)
- [When NOT to Use — Premature Optimization](#when-not-to-use--premature-optimization)
- [Cost vs Benefit Trade-off](#cost-vs-benefit-trade-off)
- [React Compiler — Auto-Memoization](#react-compiler--auto-memoization)
- [Decision Guide](#decision-guide)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Memoization Concept

### Nazariya

Memoization — computer science'da klassik optimization texnikasi: **funksiya natijasini cache'lash, bir xil input bilan qayta chaqirilganda hisoblamasdan cached qiymat qaytarish**.

```ts
// Sodda memoization misol
const cache = new Map<number, number>();

function memoizedSquare(n: number): number {
  if (cache.has(n)) {
    return cache.get(n)!;  // Cache hit — hisoblamasdan
  }
  
  const result = n * n;     // Cache miss — hisoblash
  cache.set(n, result);
  return result;
}

memoizedSquare(5);  // Compute: 25 (cache miss)
memoizedSquare(5);  // Cache hit: 25 (hisoblamasdan)
```

Memoization tradeoff: **hisoblash vaqti** o'rniga **xotira**. Cache lookup hisoblashdan tezroq bo'lsa — foyda.

**React'da memoization — render context:**

React komponent har render'da function body'ni qayta ishlaydi. Body ichidagi `const x = compute()` — har render'da yangi qiymat:

```tsx
function Component({ items }: { items: Item[] }) {
  // Har render'da qayta hisoblanadi
  const sortedItems = [...items].sort((a, b) => a.price - b.price);
  
  return <List items={sortedItems} />;
}
```

`items` o'zgarmasa ham — `sortedItems` har render'da yangi computation va yangi array reference. Performance va consumer re-render muammosi.

`useMemo` bilan — deps o'zgarmasa cached natija:

```tsx
function Component({ items }: { items: Item[] }) {
  // items o'zgarsa qayta hisoblanadi, aks holda eski natija
  const sortedItems = useMemo(
    () => [...items].sort((a, b) => a.price - b.price),
    [items]
  );
  
  return <List items={sortedItems} />;
}
```

**Ikki maqsad — memoization React'da:**

1. **Performance** — expensive computation'ni qayta hisoblamaslik
2. **Referential identity** — Object/array/function reference render orasida stabil tutish (consumer re-render skip uchun)

Ikkinchi maqsad — React'ga xos. JavaScript object identity (cross-ref [`16-useeffect.md`](16-useeffect.md) "Object/Array Deps", [`19-usecontext.md`](19-usecontext.md) "Object Value Gotcha") React'ning Object.is comparison bilan ishlaydi:

- `useEffect` deps array — yangi reference → effect qayta ishlaydi
- `useMemo` deps array — yangi reference → memo invalidate
- `React.memo` props — yangi reference → re-render
- Context Provider value — yangi reference → consumer'lar re-render

Memoization — bu identity'ni stabil tutish vositasi.

**Memoization bepul emas:**

```ts
// Sodda compute
const x = a + b;

// Memoized
const x = useMemo(() => a + b, [a, b]);
// Hook chain slot allocation
// Deps array allocation
// Object.is comparison har render
// Trivial compute uchun overhead >> compute cost
```

Memoization overhead — sodda compute uchun qaytmaydi. Faqat **expensive compute** yoki **identity stability** muhim bo'lganda foydali.

<details>
<summary><strong>Under the Hood</strong></summary>

**Computer science'da memoization:**

```
Memoization tarixi:
- Donald Michie (1968): "Memo functions"
- Top-down dynamic programming asosi
- Pure function determinizm shart (bir xil input = bir xil output)
```

Memoization pure function uchun ishlaydi. Side effect bo'lsa — cache noto'g'ri (e.g., date-dependent function).

**JavaScript'da generic memoize:**

```ts
function memoize<Args extends unknown[], R>(
  fn: (...args: Args) => R
): (...args: Args) => R {
  const cache = new Map<string, R>();
  
  return (...args: Args): R => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key)!;
    
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

const memoizedSlowFn = memoize(slowFunction);
```

Lodash `_.memoize`, ramda `R.memoize` — boshqa library'lar. React `useMemo` esa **per-component, per-hook** scope (har komponent instance uchun alohida cache).

**React `useMemo` cache scope:**

```ts
// Component instance #1
function MyComp() {
  const x = useMemo(() => compute(), []);
  // Cache: Hook obyekt'da, faqat shu instance
}

// Component instance #2 (boshqa render)
// Yangi Hook obyekt, yangi cache
```

Har Hook chaqiruvi alohida `Hook` obyekt — cache'lar bog'liq emas. `useMemo` global cache emas.

**Source citation:**

- "Structure and Interpretation of Computer Programs" (SICP) — memoization
- React docs `useMemo` — react.dev/reference/react/useMemo

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Generic memoize vs `useMemo`:**

```ts
// Generic memoize (lodash-style)
function memoize<Args extends unknown[], R>(
  fn: (...args: Args) => R
): (...args: Args) => R {
  const cache = new Map<string, R>();
  return (...args: Args) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key)!;
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

const memoFib = memoize(function fib(n: number): number {
  if (n < 2) return n;
  return memoFib(n - 1) + memoFib(n - 2);
});

memoFib(40);  // Tez (cache bilan)
```

```tsx
// React useMemo — komponent ichida
function Fibonacci({ n }: { n: number }) {
  const result = useMemo(() => fib(n), [n]);
  return <div>fib({n}) = {result}</div>;
}
// Cache faqat shu komponent uchun, faqat shu hook
```

Generic memoize — global cache (memory leak xavfi). `useMemo` — komponent instance scope (mount/unmount bilan cleanup).

**Misol 2 — Identity stability use case:**

```tsx
// ❌ Har render'da yangi config object
function Page() {
  const config = { theme: 'dark', locale: 'en' };  // Yangi reference har render
  return <ThemeContext.Provider value={config}>...</ThemeContext.Provider>;
}

// ✅ useMemo bilan stable reference
function Page() {
  const config = useMemo(() => ({ theme: 'dark', locale: 'en' }), []);
  return <ThemeContext.Provider value={config}>...</ThemeContext.Provider>;
}
```

Performance gain — Context consumer'lar har render'da re-render qilinmaydi. Compute trivial, lekin identity muhim.

**Misol 3 — Expensive compute:**

```tsx
function Statistics({ data }: { data: number[] }) {
  // ❌ Har render'da O(n log n) sort + O(n) statistics
  const sorted = [...data].sort();
  const stats = {
    median: sorted[Math.floor(sorted.length / 2)],
    mean: data.reduce((a, b) => a + b, 0) / data.length,
    stdDev: calculateStdDev(data),
  };
  
  return <Display stats={stats} />;
}

// ✅ useMemo — data o'zgarmasa cached
function Statistics({ data }: { data: number[] }) {
  const stats = useMemo(() => {
    const sorted = [...data].sort();
    return {
      median: sorted[Math.floor(sorted.length / 2)],
      mean: data.reduce((a, b) => a + b, 0) / data.length,
      stdDev: calculateStdDev(data),
    };
  }, [data]);
  
  return <Display stats={stats} />;
}
```

Million element array uchun — memoization sezilarli foyda.

</details>

---

## `useMemo` API

### Nazariya

`useMemo` signature:

```tsx
function useMemo<T>(
  factory: () => T,
  deps: React.DependencyList | undefined
): T;
```

Argumentlar:

| Argument | Tip | Vazifa |
|----------|-----|--------|
| `factory` | `() => T` | Compute function — qiymat hisoblovchi callback |
| `deps` | `unknown[]` yoki `undefined` | Dependency array — qachon qayta hisoblash |

Returned: cached yoki yangi `T` qiymat.

**Sodda usage:**

```tsx
const result = useMemo(() => {
  return expensiveComputation(a, b);
}, [a, b]);

// `a` yoki `b` o'zgarsa — factory qaytadan chaqiriladi
// Aks holda — eski qiymat qaytariladi (cache)
```

**Lifecycle:**

```
1. Mount:
   - factory() chaqiriladi
   - Natija + deps Hook.memoizedState'ga saqlanadi
   - Natija qaytariladi

2. Update (deps o'zgarmagan):
   - Object.is comparison har dep — barchasi bir xil
   - Eski natija qaytariladi (factory chaqirilmaydi)

3. Update (deps o'zgargan):
   - factory() qayta chaqiriladi
   - Yangi natija + yangi deps Hook.memoizedState'ga
   - Yangi natija qaytariladi
```

**Deps array variantlari:**

```tsx
// Variant 1 — Deps undefined (har render'da factory chaqiriladi)
const x = useMemo(() => compute(), undefined);
// `@types/react` signature: deps argument MAJBURIY (DependencyList | undefined)
// `useMemo(() => compute())` — TypeScript hatosi (Expected 2 arguments)
// `undefined` uzatilsa — memoization yo'q, har render'da re-compute

// Variant 2 — Faqat mount paytida
const x = useMemo(() => compute(), []);
// Lifelong cached qiymat (mount → unmount)

// Variant 3 — Aniq deps
const x = useMemo(() => compute(a, b), [a, b]);
// a yoki b o'zgarsa qayta hisoblash
```

Variant 1 — kamdan-kam ishlatiladi (memoization yo'q, faqat extra overhead). Variant 2 — bir martalik computation (e.g., constant table). Variant 3 — eng tipik.

**Return type inference:**

```tsx
const items = useMemo(() => filterItems(data), [data]);
// Type: ReturnType<typeof filterItems>

const config = useMemo<Config>(() => ({ theme: 'dark' }), []);
// Type: Config (explicit)
```

TypeScript factory return type'idan inference qiladi. Generic `<T>` aniqlash mumkin.

**Strict Mode 2x cycle:**

R18+ Strict Mode'da `useMemo` factory ham 2x chaqiriladi (purity test). Pure bo'lishi shart:

```tsx
let counter = 0;

function Component() {
  const value = useMemo(() => {
    counter++;  // ❌ Side effect — Strict Mode'da counter 2x oshadi
    return computeStuff();
  }, []);
}
```

Side effect — `useEffect`'da yoki event handler'da.

**`useMemo` vs `useState` lazy:**

```tsx
// useMemo — har deps o'zgarsa qayta hisoblash
const value = useMemo(() => expensive(), [deps]);

// useState lazy — faqat mount paytida
const [value] = useState(() => expensive());
// Keyingi render'larda compute chaqirilmaydi
// Lekin re-compute kerak bo'lsa setValue manual chaqirish
```

`useState` lazy — bir martalik. `useMemo` — deps-based.

<details>
<summary><strong>Under the Hood</strong></summary>

**`mountMemo` implementation:**

```ts
function mountMemo<T>(
  nextCreate: () => T,
  deps: Array<unknown> | null
): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const nextValue = nextCreate();  // Factory chaqirish
  hook.memoizedState = [nextValue, nextDeps];  // Tuple saqlash
  return nextValue;
}
```

Mount paytida factory chaqiriladi va `[value, deps]` tuple Hook'da saqlanadi.

**`updateMemo` implementation:**

```ts
function updateMemo<T>(
  nextCreate: () => T,
  deps: Array<unknown> | null
): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;  // Eski [value, deps]
  
  if (nextDeps !== null) {
    const prevDeps = prevState[1];
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      // Deps bir xil — eski qiymat qaytariladi
      return prevState[0];
    }
  }
  
  // Deps o'zgargan — qayta hisoblash
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

Update path — `areHookInputsEqual` deps comparison. Bir xil bo'lsa skip, aks holda re-compute.

**`Hook.memoizedState` tuple struktura:**

```ts
type Hook = {
  memoizedState: any,  // useMemo'da: [T, deps]
  baseState: null,
  baseQueue: null,
  queue: null,         // useMemo'da: null (state queue yo'q)
  next: Hook | null,
};
```

`useMemo` Hook obyekt — minimal. State queue yo'q (mutation yo'q), faqat tuple saqlash.

**Source citation:**

- `mountMemo` / `updateMemo` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`
- React docs `useMemo` — react.dev/reference/react/useMemo

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Sodda computation:**

```tsx
function ProductList({ products, query }: { products: Product[]; query: string }) {
  const filtered = useMemo(
    () => products.filter(p => p.name.toLowerCase().includes(query.toLowerCase())),
    [products, query]
  );
  
  return <ul>{filtered.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

`products` yoki `query` o'zgarmasa — eski filtered array qaytariladi.

**Misol 2 — Object identity stabilize:**

```tsx
function Provider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  
  // ❌ Har render'da yangi obyekt
  const value = { user, setUser };
  
  // ✅ useMemo bilan stable
  const value = useMemo(() => ({ user, setUser }), [user]);
  
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```

Cross-ref [`19-usecontext.md`](19-usecontext.md) "Memoizing Provider Value".

**Misol 3 — Lazy table:**

```tsx
function CharCodeTable() {
  // Faqat mount paytida hisoblanadi (deps [])
  const codes = useMemo(() => {
    const table = new Map<string, number>();
    for (let i = 0; i < 128; i++) {
      table.set(String.fromCharCode(i), i);
    }
    return table;
  }, []);
  
  return <pre>{Array.from(codes.entries()).map(([k, v]) => `${k}=${v}`).join(' ')}</pre>;
}
```

Constant table mount paytida bir marta yaratiladi.

**Misol 4 — TypeScript explicit type:**

```tsx
type Config = { apiUrl: string; timeout: number };

function App({ url, timeout }: { url: string; timeout: number }) {
  const config = useMemo<Config>(
    () => ({ apiUrl: url, timeout }),
    [url, timeout]
  );
  
  return <ApiContext.Provider value={config}>...</ApiContext.Provider>;
}
```

Generic `<Config>` — explicit return type.

**Misol 5 — Pure factory invariant:**

```tsx
function Component() {
  // ❌ Side effect factory ichida
  const x = useMemo(() => {
    document.title = 'New title';  // ❌ DOM mutation
    return computeValue();
  }, []);
  
  // ✅ Side effect — useEffect'da
  useEffect(() => {
    document.title = 'New title';
  }, []);
  
  const x = useMemo(() => computeValue(), []);
}
```

Factory pure shart — Strict Mode 2x call'da side effect 2x bajariladi (silent bug).

</details>

---

## `useCallback` API

### Nazariya

`useCallback` signature:

```tsx
function useCallback<T extends Function>(
  callback: T,
  deps: React.DependencyList
): T;
```

Argumentlar:

| Argument | Tip | Vazifa |
|----------|-----|--------|
| `callback` | `T extends Function` | Memoize qilinadigan function |
| `deps` | `unknown[]` | Dependency array |

Returned: bir xil yoki yangi function reference.

**Sodda usage:**

```tsx
const handleClick = useCallback(
  () => {
    console.log('Click', id);
  },
  [id]
);
```

`id` o'zgarmasa — bir xil function reference. Aks holda yangi function (yangi closure).

**Lifecycle:**

```
1. Mount:
   - Callback va deps Hook.memoizedState'ga saqlanadi
   - Callback qaytariladi

2. Update (deps o'zgarmagan):
   - Eski callback qaytariladi (bir xil reference)

3. Update (deps o'zgargan):
   - Yangi callback va deps saqlanadi
   - Yangi callback qaytariladi
```

**Diqqat — `useCallback` callback'ni hech qachon "chaqirmaydi":**

```tsx
const fn = useCallback(() => console.log('hi'), []);
// fn — function reference
// useCallback fn'ni hech qachon o'zi chaqirmaydi
// Caller manual chaqirishi shart: fn()
```

`useMemo` factory'ni chaqirib **natijasini** qaytaradi. `useCallback` callback'ni chaqirmasdan — **callback'ni o'zini** qaytaradi.

**TypeScript inference:**

```tsx
// Inference — callback signature'idan
const handleClick = useCallback((e: MouseEvent) => {
  console.log(e.x);
}, []);
// Type: (e: MouseEvent) => void

// Explicit generic
const handler = useCallback<(id: string) => void>((id) => {
  console.log(id);
}, []);
// Type: (id: string) => void
```

Inference avtomatik — explicit kamdan-kam kerak.

**Closure trapi:**

```tsx
function Component({ count }: { count: number }) {
  const handleClick = useCallback(() => {
    console.log(count);  // Closure — count'ni oladi
  }, []);  // ❌ count deps'da yo'q — stale closure
  
  return <button onClick={handleClick}>{count}</button>;
}
```

`useCallback` — closure'ni "saqlaydi". Deps array'da count yo'q → har render'dagi callback eski count'ni "ko'radi" (stale closure). Cross-ref [`16-useeffect.md`](16-useeffect.md) "Stale Closure".

`react-hooks/exhaustive-deps` linter bu xatoni topadi. Yechimlar:

```tsx
// ✅ Yechim 1 — count deps'da
const handleClick = useCallback(() => {
  console.log(count);
}, [count]);  // count o'zgarsa — yangi callback

// ✅ Yechim 2 — useRef latest pattern
const countRef = useRef(count);
useEffect(() => { countRef.current = count; });

const handleClick = useCallback(() => {
  console.log(countRef.current);  // Latest
}, []);
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`mountCallback` implementation:**

```ts
function mountCallback<T>(
  callback: T,
  deps: Array<unknown> | null
): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  hook.memoizedState = [callback, nextDeps];  // Callback + deps
  return callback;
}
```

Mount paytida — callback to'g'ridan-to'g'ri saqlanadi (chaqirilmaydi).

**`updateCallback` implementation:**

```ts
function updateCallback<T>(
  callback: T,
  deps: Array<unknown> | null
): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  
  if (nextDeps !== null) {
    const prevDeps = prevState[1];
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      return prevState[0];  // Eski callback
    }
  }
  
  hook.memoizedState = [callback, nextDeps];  // Yangi callback
  return callback;
}
```

`updateMemo` bilan deyarli identik — farq faqat `callback` `nextCreate()` chaqirish o'rniga to'g'ridan-to'g'ri saqlash.

**`useCallback` aslida `useMemo`:**

```ts
// Behaviorally equivalent (semantik teng), lekin har biri o'z closure'i:
const fn1 = useCallback(callback, deps);
const fn2 = useMemo(() => callback, deps);

// fn1 va fn2 — turli function instance'lar (fn1 !== fn2),
// lekin har biri o'z render'lari orasida stable (deps bir xil bo'lsa).
```

Internal'da implementation faqat bir kichik farq bilan ekvivalent.

**Source citation:**

- `mountCallback` / `updateCallback` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`
- React docs `useCallback` — react.dev/reference/react/useCallback

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Stable handler:**

```tsx
function Parent({ onSelect }: { onSelect: (id: string) => void }) {
  // ❌ Har render'da yangi function
  const handleSelect = (id: string) => onSelect(id);
  
  // ✅ Stable reference
  const handleSelect = useCallback((id: string) => onSelect(id), [onSelect]);
  
  return <Child onSelect={handleSelect} />;
}
```

`Child` `React.memo` bo'lsa — `handleSelect` stable bo'lganda re-render skip.

**Misol 2 — Deps to'g'ri:**

```tsx
function Search({ query }: { query: string }) {
  const handleSearch = useCallback(() => {
    fetch(`/api/search?q=${query}`);
  }, [query]);  // query o'zgarsa yangi function
  
  return <Button onClick={handleSearch}>Search</Button>;
}
```

`query` deps'da. ESLint exhaustive-deps verify qiladi.

**Misol 3 — Empty deps stable:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  
  // ✅ setCount stable (useState setter)
  const increment = useCallback(() => setCount(c => c + 1), []);
  
  // increment har doim bir xil reference
  return <button onClick={increment}>{count}</button>;
}
```

Functional update bilan deps minimize.

**Misol 4 — Closure trap:**

```tsx
function Component({ id }: { id: string }) {
  // ❌ Stale closure
  const handler = useCallback(() => {
    console.log(id);
  }, []);  // id deps'da yo'q
  
  // Render 1: id='a' → handler closure id='a'
  // Render 2: id='b' → handler hali ham closure id='a' (stale)
  
  // ✅ id deps'da
  const handler = useCallback(() => {
    console.log(id);
  }, [id]);
  
  return <button onClick={handler}>Click</button>;
}
```

ESLint warning — exhaustive-deps fix tavsiya beradi.

**Misol 5 — Custom hook return:**

```tsx
function useToggle(initial = false) {
  const [state, setState] = useState(initial);
  
  // ✅ Stable handlers — consumer useEffect deps'da OK
  const toggle = useCallback(() => setState(s => !s), []);
  const setTrue = useCallback(() => setState(true), []);
  const setFalse = useCallback(() => setState(false), []);
  
  return { state, toggle, setTrue, setFalse };
}

// Consumer:
function Modal() {
  const { state: isOpen, toggle } = useToggle();
  
  useEffect(() => {
    document.addEventListener('keydown', toggle);
    return () => document.removeEventListener('keydown', toggle);
  }, [toggle]);  // toggle stable — effect bir marta o'rnatiladi
  
  return isOpen ? <dialog open>Modal content</dialog> : null;
}
```

Custom hook'ning return functions stable bo'lsa — consumer code clean.

</details>

---

## Texnik Ekvivalentlik

### Nazariya

`useCallback(fn, deps) ≡ useMemo(() => fn, deps)` — semantik teng. `useCallback` aslida `useMemo`'ning sintaktik shakli.

**Proof:**

```tsx
// useCallback
const fn1 = useCallback(callback, [a, b]);

// useMemo equivalent
const fn2 = useMemo(() => callback, [a, b]);

// fn1 — bir xil callback reference (deps o'zgarmasa)
// fn2 — bir xil callback reference (deps o'zgarmasa)
// Behaviorally identical
```

**API farq:**

| Aspekt | `useCallback` | `useMemo` |
|--------|---------------|-----------|
| Argument | `(callback, deps)` | `(factory, deps)` |
| Factory chaqiriladi | Yo'q (callback to'g'ridan-to'g'ri) | Ha (factory chaqirilib natija saqlanadi) |
| Returned | Callback function | Factory natijasi |
| Use case | Function reference stabilize | Computed value stabilize |

`useCallback(fn, deps)` o'rniga `useMemo(() => fn, deps)` yozish texnik jihatdan ishlaydi, lekin **idiomatic emas**.

**Internal source code:**

```ts
// React internal — soddalashtirilgan
function mountCallback(callback, deps) {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = [callback, deps];
  return callback;
}

function mountMemo(create, deps) {
  const hook = mountWorkInProgressHook();
  const value = create();  // ← factory chaqiriladi
  hook.memoizedState = [value, deps];
  return value;
}

// Update path bir xil — areHookInputsEqual deps comparison
// Faqat factory call vs callback save farq
```

Implementation deyarli identik — `mountMemo` factory chaqiradi, `mountCallback` chaqirmaydi.

**`useMemo` factory bilan function qaytarish — `useCallback` ekvivalent:**

```tsx
// Bu ikkalasi semantik ekvivalent (bir xil xulq-atvor):
const fn1 = useCallback(() => doStuff(a), [a]);
const fn2 = useMemo(() => () => doStuff(a), [a]);  // Function returning function

// Lekin fn1 !== fn2 (turli function instance'lar — turli source position).
// Har biri o'z render'lari orasida stable (deps bir xil bo'lsa fn1_prev === fn1_curr).
```

`useMemo` factory function qaytarsa — `useCallback`'ga teng. Lekin syntax xunuk (`() => () =>`).

**Tarixiy sabab:**

`useCallback` qulay shorthand sifatida qo'shilgan (function references — eng tipik memoization use case). `useMemo` umumiy purpose.

**Tanlash:**

```tsx
// ✅ Function reference — useCallback (idiomatic)
const handleClick = useCallback(() => doStuff(id), [id]);

// ✅ Computed value (object, array, primitive) — useMemo
const config = useMemo(() => ({ a, b }), [a, b]);
const sorted = useMemo(() => [...items].sort(), [items]);

// ❌ Anti-pattern — useMemo function uchun
const handleClick = useMemo(() => () => doStuff(id), [id]);
// Texnik ishlaydi, lekin readable emas — useCallback afzal
```

Konvensiya: function — `useCallback`, value — `useMemo`.

<details>
<summary><strong>Under the Hood</strong></summary>

**Implementation comparison:**

```ts
// useCallback
function updateCallback(callback, deps) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  
  if (nextDeps !== null) {
    const prevDeps = prevState[1];
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      return prevState[0];  // Eski callback
    }
  }
  
  hook.memoizedState = [callback, nextDeps];
  return callback;
}

// useMemo
function updateMemo(create, deps) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  
  if (nextDeps !== null) {
    const prevDeps = prevState[1];
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      return prevState[0];  // Eski value
    }
  }
  
  const nextValue = create();  // ← factory chaqirish
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

Farq — bir qator: `create()` chaqirish vs `callback` saqlash.

**Performance bir xil:**

Ikki hook bir xil performance — hook chain slot, deps allocation, areHookInputsEqual comparison.

**`useCallback` faqat convenience:**

Tarixiy: hooks RFC'da `useMemo` birinchi taklif qilingan. `useCallback` keyin qo'shilgan — function reference memoization eng tez-tez ishlatiladigan use case.

**Source citation:**

- `mountMemo` vs `mountCallback` — facebook/react source
- Hooks RFC #68 — reactjs/rfcs

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Side-by-side ekvivalent:**

```tsx
// useCallback
const fn1 = useCallback((id: string) => {
  doStuff(id);
}, []);

// useMemo equivalent (semantik)
const fn2 = useMemo(() => (id: string) => {
  doStuff(id);
}, []);

// Identical behavior — har biri o'z render'lari orasida STABLE:
console.log(fn1 === fn1_prev_render);  // true (deps bir xil bo'lsa)
console.log(fn2 === fn2_prev_render);  // true (deps bir xil bo'lsa)

// LEKIN fn1 va fn2 — turli function instance'lar:
console.log(fn1 === fn2);  // false (turli source code position'lar)
```

**Misol 2 — `useMemo` factory function returning function:**

```tsx
// ⚠️ Texnik OK, lekin readable emas
const handler = useMemo(() => {
  return (id: string) => doStuff(id);
}, []);

// ✅ useCallback idiomatic
const handler = useCallback((id: string) => doStuff(id), []);
```

**Misol 3 — useMemo bilan computed function (advanced):**

```tsx
function useEventHandler(action: 'play' | 'pause') {
  // useMemo'da function tanlash
  const handler = useMemo(() => {
    if (action === 'play') {
      return () => video.play();
    }
    return () => video.pause();
  }, [action]);
  
  return handler;
}
```

Bu ham `useCallback` bilan yozilishi mumkin:

```tsx
const handler = useCallback(() => {
  if (action === 'play') {
    video.play();
  } else {
    video.pause();
  }
}, [action]);
```

Tanlash kontekstga qarab — `useMemo` agar function tanlash logic ko'p.

**Misol 4 — Anti-pattern check:**

```tsx
// ❌ Anti-pattern — useMemo function uchun (readable emas)
const handleClick = useMemo(() => () => alert('hi'), []);

// ✅ useCallback — idiomatic
const handleClick = useCallback(() => alert('hi'), []);
```

**Misol 5 — `useCallback` chaqirmaydi:**

```tsx
let counter = 0;

function Component() {
  // useCallback callback'ni hech qachon chaqirmaydi (saqlaydi va reference qaytaradi)
  const fn = useCallback(() => {
    counter++;
    return counter;
  }, []);
  
  // Render — counter o'zgarmaydi (fn hali chaqirilmadi)
  // Eslatma: console.log render ichida — concurrent rendering'da
  // multiple times bo'lishi mumkin (Strict Mode 2x va boshqa scenario'larda)
  
  // fn() chaqirilganda — counter increment bo'ladi
}

// useMemo factory — har deps o'zgarganda chaqiriladi
let counter2 = 0;

function Component2() {
  const value = useMemo(() => {
    counter2++;
    return counter2;
  }, []);
  
  // Mount paytida factory chaqiriladi
  // Strict Mode'da factory 2x chaqirilishi mumkin (purity test)
  // Production'da bir marta (deps bir xil)
}
```

`useCallback` callback'ni saqlaydi (chaqirmaydi). `useMemo` factory'ni chaqirib natijani saqlaydi.

</details>

---

## Mexanika — `Hook.memoizedState` Tuple

### Nazariya

`useMemo` va `useCallback` Hook'lari `Hook.memoizedState`'da `[value, deps]` tuple saqlaydi. Bu — hooks linked list'dagi bir node (cross-ref [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md) "Hooks Linked List").

**Hook obyekt struktura:**

```ts
type Hook = {
  memoizedState: any,   // useMemo/useCallback: [value, deps]
  baseState: null,      // useMemo/useCallback uchun ishlatilmaydi
  baseQueue: null,      // useMemo/useCallback uchun ishlatilmaydi
  queue: null,          // useMemo/useCallback uchun ishlatilmaydi
  next: Hook | null,    // Keyingi hook chain'da
};
```

`useMemo`/`useCallback` Hook obyekt — minimal. State queue yo'q (mutation yo'q), faqat tuple.

**Tuple shakli:**

```ts
type MemoState = [value: T, deps: unknown[] | null];

// Misol
useMemo(() => 42, []);            // [42, []]
useMemo(() => obj, [a, b]);        // [obj, [a, b]]
useCallback(() => fn, [x]);        // [callbackFn, [x]]
```

`memoizedState[0]` — saqlangan qiymat (yoki callback). `memoizedState[1]` — deps array.

**Hook indexing va Rules of Hooks:**

```tsx
function Component() {
  useState(0);                    // Hook 0
  useMemo(() => 1, []);           // Hook 1
  useCallback(() => {}, []);      // Hook 2
  useEffect(() => {}, []);        // Hook 3 (useEffect void qaytaradi)
}
```

Har hook chaqiruvi linked list'da position'ga ega. Conditional hook → position o'zgaradi → silent state corruption (cross-ref [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md) "Conditional Hook Why TAQIQ").

**Memory layout:**

```
Fiber.memoizedState → Hook 0 (useState) ─┐
                                          │
                       Hook 1 (useMemo)   ─┤ memoizedState: [value, deps]
                                          │
                       Hook 2 (useCallback) ─┤ memoizedState: [callback, deps]
                                          │
                       Hook 3 (useEffect) ─┘
```

Linked list traversal — render har gal Fiber.memoizedState'dan boshlanadi.

**Update detection algoritm:**

```ts
function updateMemo(nextCreate, nextDeps) {
  const hook = updateWorkInProgressHook();  // Position'dagi hook
  const prevState = hook.memoizedState;     // Eski [value, deps]
  
  if (nextDeps !== null && prevState !== null) {
    const prevDeps = prevState[1];
    
    // Deps comparison
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      return prevState[0];  // Cache hit
    }
  }
  
  // Cache miss — re-compute
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

`areHookInputsEqual` — keyingi section.

<details>
<summary><strong>Under the Hood</strong></summary>

**`mountWorkInProgressHook` vs `updateWorkInProgressHook`:**

```ts
// Mount path
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };
  
  if (workInProgressHook === null) {
    // Birinchi hook — Fiber.memoizedState head
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // Append linked list
    workInProgressHook = workInProgressHook.next = hook;
  }
  
  return workInProgressHook;
}

// Update path
function updateWorkInProgressHook(): Hook {
  let nextCurrentHook: Hook | null;
  
  if (currentHook === null) {
    nextCurrentHook = currentlyRenderingFiber.alternate?.memoizedState ?? null;
  } else {
    nextCurrentHook = currentHook.next;
  }
  
  // Yangi Hook obyekt (immutability for Concurrent)
  const newHook: Hook = {
    memoizedState: nextCurrentHook.memoizedState,  // Eski state copy
    baseState: nextCurrentHook.baseState,
    baseQueue: nextCurrentHook.baseQueue,
    queue: nextCurrentHook.queue,
    next: null,
  };
  
  // Append yangi list
  if (workInProgressHook === null) {
    currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
  } else {
    workInProgressHook = workInProgressHook.next = newHook;
  }
  
  currentHook = nextCurrentHook;
  return workInProgressHook;
}
```

Mount — yangi hook list quriladi. Update — har render'da yangi hook list quriladi (immutability for Concurrent restart safety).

**Per-hook memory:**

```
Bir useMemo Hook overhead:
- Hook obyekt: 5 ta field (memoizedState, baseState, baseQueue, queue, next)
- memoizedState array: 2 element [value, deps]
- Deps array: deps.length element
- Saqlangan value (factory natijasi)

Bir useCallback Hook overhead:
- Hook obyekt: 5 ta field
- memoizedState array: 2 element [callback, deps]
- Deps array
- Function reference
```

Memoization bepul emas — har hook chain'da yangi Hook obyekt, tuple va deps array allocate qilinadi. Aniq byte hajmi JS engine (V8 va boshqalar) object representation'iga bog'liq, lekin har bir memoized hook qo'shimcha allocation talab qiladi.

**Source citation:**

- `mountMemo` / `updateMemo` / `mountCallback` / `updateCallback` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Hook chain visualization:**

```tsx
function Component({ a, b }: { a: number; b: number }) {
  const [count, setCount] = useState(0);
  // Hook 0: queue, memoizedState: 0
  
  const sum = useMemo(() => a + b, [a, b]);
  // Hook 1: memoizedState: [sum, [a, b]]
  
  const handleClick = useCallback(() => setCount(c => c + 1), []);
  // Hook 2: memoizedState: [handleClickFn, []]
  
  return <button onClick={handleClick}>{count + sum}</button>;
}

// Fiber.memoizedState = Hook 0 → Hook 1 → Hook 2 → null
```

**Misol 2 — Conditional hook silent bug:**

```tsx
function Component({ flag }: { flag: boolean }) {
  // ❌ Conditional hook — Rules of Hooks buzilishi
  if (flag) {
    const x = useMemo(() => compute(), []);
  }
  
  const y = useState(0);  // Position o'zgaradi flag bilan
}

// flag=true: hook chain = [useMemo, useState]
// flag=false: hook chain = [useState]
// React confused — hook count match emas → throw / state corruption
```

**Misol 3 — Memory layout demo:**

```tsx
function Heavy() {
  // 100 ta useMemo — 100 ta Hook slot
  const a = useMemo(() => compute1(), []);
  const b = useMemo(() => compute2(), []);
  // ...
  const x = useMemo(() => compute100(), []);
  
  // Memory: 100 × Hook obyekt + 100 × tuple + 100 × deps array
  // Render: 100 × areHookInputsEqual call
  
  return null;
}
```

100 ta `useMemo` — bir komponent'da kamdan-kam, lekin hook count'ga e'tibor.

**Misol 4 — Tuple inspection (debug):**

```tsx
function useTrackedMemo({ a, b }: { a: number; b: number }) {
  const value = useMemo(() => a * b, [a, b]);
  
  // useDebugValue — React DevTools'da custom hook value ko'rsatish
  useDebugValue({ value, deps: [a, b] });
  
  return value;
}

function Component({ a, b }: { a: number; b: number }) {
  const value = useTrackedMemo({ a, b });
  return <div>{value}</div>;
}
```

`useDebugValue` — public API, custom hook ichida ishlatiladi. React DevTools panel'da hook value (deps va memoized result) ko'rinadi. Komponent ichida to'g'ridan-to'g'ri `Hook.memoizedState` tuple'ga kirish API'si yo'q — debug uchun `useDebugValue` yoki Profiler.

</details>

---

## `Object.is` Deps Comparison

### Nazariya

`useMemo` va `useCallback` deps array'ni `Object.is` orqali element-wise solishtiradi. Algoritm — `areHookInputsEqual`.

**`areHookInputsEqual` implementation:**

```ts
function areHookInputsEqual(
  nextDeps: Array<unknown>,
  prevDeps: Array<unknown> | null,
): boolean {
  if (prevDeps === null) {
    // Birinchi render — prevDeps yo'q, har doim "o'zgargan"
    return false;
  }
  
  if (__DEV__) {
    if (nextDeps.length !== prevDeps.length) {
      // Deps array uzunligi o'zgarsa — faqat DEV warning (early return YO'Q)
      console.error(
        'The final argument passed to %s changed size between renders. ' +
        'Expected size: %s. Current size: %s.',
        hookName, prevDeps.length, nextDeps.length
      );
    }
  }
  
  // Loop min(prevDeps.length, nextDeps.length) bo'yicha yuradi
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (Object.is(nextDeps[i], prevDeps[i])) {
      continue;  // Bir xil — keyingi
    }
    return false;  // O'zgargan — re-compute
  }
  
  return true;  // Solishtirilgan elementlar bir xil
}
```

Algoritm:

1. Birinchi render — `prevDeps === null` → false (re-compute)
2. Deps uzunligi farq — faqat DEV'da `console.error` warning, ammo funksiya darhol false qaytarmaydi: loop qisqaroq uzunlik bo'yicha yuradi
3. Har dep `Object.is` comparison
4. Birinchi farqda → false
5. Solishtirilgan elementlar bir xil → true (skip re-compute)

**`Object.is` semantikasi:**

```ts
Object.is(NaN, NaN);     // true   ← === false bo'lardi
Object.is(0, -0);        // false  ← === true bo'lardi
Object.is(+0, +0);       // true
Object.is('a', 'a');     // true
Object.is({}, {});       // false  (har xil reference)
Object.is(null, null);   // true
Object.is(undefined, undefined); // true
```

Primitive value'lar — value equality. Object'lar — reference equality.

**Object/array deps trapi (cross-ref [`16-useeffect.md`](16-useeffect.md), [`19-usecontext.md`](19-usecontext.md)):**

```tsx
function Component({ items }: { items: Item[] }) {
  // ❌ Har render'da yangi obyekt
  const config = { theme: 'dark' };
  
  const value = useMemo(
    () => doStuff(items, config),
    [items, config]  // ❌ config har render'da yangi reference → memo invalidate
  );
}

// ✅ Primitive deps yoki memoized config
const value = useMemo(
  () => doStuff(items, 'dark'),
  [items]  // ✅
);
```

Object/array/function deps — har render'da yangi reference → memo har gal invalidate. Memoization befoyda.

**Function deps — useCallback bilan stabilize:**

```tsx
function Parent({ data }: { data: Item[] }) {
  // ❌ Har render'da yangi handler
  const handler = (item: Item) => process(item);
  
  // ✅ useCallback
  const handler = useCallback((item: Item) => process(item), []);
  
  // Pass to child — handler stable bo'lsa React.memo skip qiladi
  return <List data={data} onItemClick={handler} />;
}
```

**`undefined` deps — har render'da:**

```tsx
const value = useMemo(() => compute(), undefined);  // Deps undefined
// `react-hooks/exhaustive-deps` ESLint rule warning beradi
// Runtime: deps null → areHookInputsEqual qaytadan tekshirilmaydi → har render'da factory chaqirilishi
// Memoization yo'q
```

Deps explicit yozish kerak (bo'sh `[]` yoki real deps). Argumentni o'tkazib yuborish (`useMemo(() => compute())`) — TypeScript signature'da hato (deps argument required).

<details>
<summary><strong>Under the Hood</strong></summary>

**`Object.is` vs `===`:**

ECMAScript spec — `SameValue` algorithm:

```ts
// Object.is(x, y) — SameValue
// 1. If Type(x) is different from Type(y) → false
// 2. If both are NaN → true
// 3. If both are +0 → true
// 4. If both are -0 → true
// 5. If +0 and -0 (different signs) → false
// 6. Else: return x === y
```

`===` (Strict Equality) — spec'dagi `IsStrictlyEqual` algorithm. `Object.is` (`SameValue`) bilan farq faqat Number qiymatlarida: `NaN` va signed zero (`+0`/`-0`). Non-Number qiymatlar uchun ikkalasi bir xil natija beradi (spec'da umumiy `SameValueNonNumber` operatsiyasiga tayanadi).

**Why `Object.is` for deps:**

NaN-aware comparison. Misol:

```ts
const x = NaN;
const y = NaN;

x === y;             // false ← bug-prone
Object.is(x, y);     // true ← React deps'da to'g'ri xulq-atvor
```

Agar deps'da NaN bo'lsa — `===` bilan har gal "o'zgargan" deb hisoblanardi. `Object.is` bilan to'g'ri.

**`-0` edge case:**

```ts
const x = -0;
const y = +0;

x === y;             // true
Object.is(x, y);     // false ← React deps'da yangi value deb hisoblanadi
```

`-0` deps'da — kamdan-kam, lekin `Math.sign` qaytaradigan signed zero, IEEE 754 floating-point division natijasi (`-1 / Infinity === -0`) va h.k. joylarda uchraydi.

**Performance:**

`Object.is` — V8 va boshqa engine'larda optimized. Reference comparison O(1) (pointer comparison). Primitive comparison ham O(1).

`areHookInputsEqual` — O(N) (N = deps length). Tipik N=1-5, overhead minimal.

**Source citation:**

- `Object.is` — ECMAScript spec SameValue algorithm
- `areHookInputsEqual` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Primitive deps OK:**

```tsx
function Component({ id, count }: { id: string; count: number }) {
  // ✅ Primitive deps — Object.is value equality
  const value = useMemo(() => `${id}-${count}`, [id, count]);
  
  // id='a', count=5 → "a-5"
  // id='a', count=5 → "a-5" (cache hit)
  // id='b', count=5 → "b-5" (cache miss, re-compute)
}
```

**Misol 2 — Object deps trap:**

```tsx
function Component({ items }: { items: Item[] }) {
  // ❌ Inline object — har render yangi reference
  const value = useMemo(
    () => sortItems(items, { ascending: true }),
    [items, { ascending: true }]  // ❌ {ascending: true} har gal yangi
  );
  
  // ✅ Primitive deps
  const value = useMemo(
    () => sortItems(items, { ascending: true }),
    [items]  // ✅
  );
  
  // ✅ Memoized config
  const config = useMemo(() => ({ ascending: true }), []);
  const value = useMemo(
    () => sortItems(items, config),
    [items, config]
  );
}
```

**Misol 3 — Function deps:**

```tsx
function Parent({ multiplier }: { multiplier: number }) {
  // ❌ Inline function — har render yangi
  const compute = (x: number) => x * multiplier;
  
  const Child = React.memo(function Child({ compute }: { compute: (x: number) => number }) {
    return <div>{compute(5)}</div>;
  });
  
  // Har render'da Child re-render (compute yangi reference)
  
  // ✅ useCallback
  const compute = useCallback((x: number) => x * multiplier, [multiplier]);
  // multiplier stable bo'lsa — Child skip
}
```

**Misol 4 — NaN deps:**

```tsx
function Component({ value }: { value: number }) {
  // value = NaN bo'lsa
  const result = useMemo(() => doStuff(value), [value]);
  
  // Render 1: value=NaN → cache: [result, [NaN]]
  // Render 2: value=NaN → Object.is(NaN, NaN) === true → cache hit ✅
  
  // === bilan bo'lsa (React ishlatmaydi):
  // NaN === NaN → false → har gal cache miss
}
```

`Object.is` NaN-aware — to'g'ri xulq-atvor.

**Misol 5 — Deps length warning:**

```tsx
function Component({ flag }: { flag: boolean }) {
  // ❌ Deps array uzunligi conditional
  const value = useMemo(() => compute(), flag ? [1, 2] : [1]);
  
  // Console (DEV): "The final argument passed to useMemo changed size between renders"
  // React early-return qilmaydi: areHookInputsEqual qisqaroq uzunlik bo'yicha
  // solishtiradi → noaniq (kutilmagan) cache xulq-atvori
}

// ✅ Deps uzunligi statik — har render'da bir xil son element
const value = useMemo(() => compute(), [1, flag ? 2 : 1]);
```

Deps array uzunligi har render'da bir xil bo'lishi shart — bu Rules of Hooks invariant'i. Uzunlik o'zgarsa, comparison qisqaroq array bo'yicha yuradi va natija ishonchsiz bo'ladi.

</details>

---

## When to Use — `React.memo` bilan

### Nazariya

`React.memo` — komponent'ni memoize qiladi: props referential equality bo'lsa re-render skip. `useMemo`/`useCallback` `React.memo` bilan birga props identity stabilize uchun ishlatiladi.

**Tipik pattern:**

```tsx
// Memoized child
const ExpensiveList = React.memo(function ExpensiveList({
  items,
  onSelect,
}: {
  items: Item[];
  onSelect: (id: string) => void;
}) {
  // Heavy rendering
  return <ul>{items.map(i => <li key={i.id} onClick={() => onSelect(i.id)}>{i.name}</li>)}</ul>;
});

// Parent
function Parent() {
  const [data, setData] = useState<Item[]>([]);
  const [filter, setFilter] = useState('');
  
  // ❌ Memo befoyda — har render'da yangi reference
  const filteredItems = data.filter(i => i.name.includes(filter));  // Yangi array
  const handleSelect = (id: string) => console.log(id);              // Yangi function
  
  return <ExpensiveList items={filteredItems} onSelect={handleSelect} />;
  // ExpensiveList har render'da re-render (props yangi)
}

// ✅ useMemo + useCallback
function Parent() {
  const [data, setData] = useState<Item[]>([]);
  const [filter, setFilter] = useState('');
  
  const filteredItems = useMemo(
    () => data.filter(i => i.name.includes(filter)),
    [data, filter]
  );
  
  const handleSelect = useCallback((id: string) => console.log(id), []);
  
  return <ExpensiveList items={filteredItems} onSelect={handleSelect} />;
  // data/filter o'zgarmasa — ExpensiveList skip
}
```

**`React.memo` shallow comparison:**

```tsx
const Memo = React.memo(Component);
// Default — shallowEqual props

const Memo = React.memo(Component, (prev, next) => {
  // Custom comparator
  return prev.id === next.id;  // True → skip
});
```

`shallowEqual` — har prop `Object.is` comparison. Bir xil bo'lsa skip.

**`useMemo`/`useCallback` `React.memo`'siz befoyda:**

```tsx
function Parent() {
  const items = useMemo(() => sortItems(data), [data]);
  
  return <Child items={items} />;  // Child memo emas — har gal re-render
}

function Child({ items }: { items: Item[] }) {
  return <ul>...</ul>;
}

// items stable, lekin Child har gal re-render (parent re-render bilan)
// useMemo befoyda
```

`React.memo` bilan birga — pattern qiymat oladi.

**Konvensiya:**

```
Parent: useMemo + useCallback (props stabilize)
Child:  React.memo (props compare)

Birga ishlatiladi. Faqat birortasi — yarim foyda.
```

**Cross-ref [`33-optimization.md`](33-optimization.md):**

Real-world optimization patterns — `React.memo` chuqur, custom comparator, when memo helps vs hurts, profiling. Bu fayl mexanikaga fokus.

<details>
<summary><strong>Under the Hood</strong></summary>

**`React.memo` implementation:**

```ts
function memo<P>(
  Component: React.ComponentType<P>,
  compare?: (prev: P, next: P) => boolean,
): React.ComponentType<P> {
  return {
    $$typeof: REACT_MEMO_TYPE,
    type: Component,
    compare: compare ?? null,
  };
}

// Reconciler check
function updateMemoComponent(...) {
  const Component = workInProgress.type.type;
  const compare = workInProgress.type.compare ?? shallowEqual;
  
  if (compare(prevProps, nextProps)) {
    // Bailout — skip render
    return bailoutOnAlreadyFinishedWork(...);
  }
  
  // Re-render
  return processChildren(...);
}
```

`shallowEqual` har prop `Object.is`. Custom comparator — to'liq nazorat.

**`useMemo` + `React.memo` synergy:**

```
Parent re-render
├─ useMemo: deps bir xil → eski reference qaytariladi
├─ useCallback: deps bir xil → eski callback qaytariladi
└─ <Child items={items} onSelect={handler} />
   ├─ React.memo shallowEqual:
   │  - prev.items === next.items (useMemo bilan stable)
   │  - prev.onSelect === next.onSelect (useCallback bilan stable)
   └─ True → bailout, Child re-render skip
```

Stable references → memo bailout.

**Context bilan memo cheklovi:**

```tsx
const Memo = React.memo(function Child() {
  const theme = useContext(ThemeContext);
  return <div className={theme}>...</div>;
});

// Context value o'zgarsa — useContext consumer'lar re-render qilinadi
// React.memo faqat props shallowEqual tekshiradi — context subscription'ni "ko'rmaydi"
// Context subscription'lar memo props comparison'dan oldin keladi (reconciler context propagation)
```

`React.memo` faqat props identity bilan ishlaydi. Context subscription mustaqil kanal — value o'zgarsa, `useContext` ishlatuvchi har bir komponent (memo bo'lsa ham) qayta render qilinadi.

**Source citation:**

- `React.memo` — facebook/react `packages/react/src/memo.js`
- `updateMemoComponent` — facebook/react `packages/react-reconciler/src/ReactFiberBeginWork.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Stable props pattern:**

```tsx
const Item = React.memo(function Item({ data, onClick }: { data: Data; onClick: () => void }) {
  console.log('Item render');
  return <li onClick={onClick}>{data.name}</li>;
});

function List({ items }: { items: Data[] }) {
  const handleClick = useCallback(() => console.log('clicked'), []);
  
  return (
    <ul>
      {items.map(item => (
        <Item key={item.id} data={item} onClick={handleClick} />
      ))}
    </ul>
  );
}

// items o'zgarmasa — Item re-render skip
// handleClick stable, item.data stable → memo bailout
```

**Misol 2 — Memo without stable references:**

```tsx
const Child = React.memo(Heavy);

function Parent({ data }: { data: Data[] }) {
  // ❌ items har render yangi
  const items = data.map(d => ({ ...d, formatted: formatItem(d) }));
  
  return <Child items={items} />;
  // React.memo befoyda — items har gal yangi
}

// ✅ useMemo
function Parent({ data }: { data: Data[] }) {
  const items = useMemo(
    () => data.map(d => ({ ...d, formatted: formatItem(d) })),
    [data]
  );
  
  return <Child items={items} />;
  // data o'zgarmasa — items stable, Child skip
}
```

**Misol 3 — Context+memo cheklovi:**

```tsx
const ThemedButton = React.memo(function ThemedButton({ label }: { label: string }) {
  const theme = useContext(ThemeContext);
  return <button className={theme}>{label}</button>;
});

function Page() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <ThemedButton label="Click" />
      {/* theme o'zgarsa — ThemedButton re-render qilinadi (useContext subscription) */}
      {/* React.memo faqat props shallowEqual qiladi, context subscription'ni bloklamaydi */}
    </ThemeContext.Provider>
  );
}
```

**Misol 4 — Custom comparator:**

```tsx
const ItemList = React.memo(
  function ItemList({ items }: { items: Item[] }) {
    return <ul>{items.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
  },
  (prev, next) => {
    // Faqat array length va first item id o'zgarsa re-render
    return (
      prev.items.length === next.items.length &&
      prev.items[0]?.id === next.items[0]?.id
    );
  }
);
```

Custom comparator — niche cases (deep comparison qisman). Default `shallowEqual` ko'p hollarda yetadi.

**Misol 5 — Memo + useCallback + useMemo full pattern:**

```tsx
const Chart = React.memo(function Chart({
  data,
  config,
  onPointClick,
}: {
  data: Point[];
  config: ChartConfig;
  onPointClick: (point: Point) => void;
}) {
  // Heavy rendering
  return <canvas>...</canvas>;
});

function Dashboard({ rawData }: { rawData: RawData }) {
  const data = useMemo(() => transformData(rawData), [rawData]);
  
  const config = useMemo<ChartConfig>(
    () => ({ width: 800, height: 400, theme: 'dark' }),
    []
  );
  
  const onPointClick = useCallback((point: Point) => {
    console.log('clicked', point);
  }, []);
  
  return <Chart data={data} config={config} onPointClick={onPointClick} />;
  // Hammasi stable — Chart skip rawData o'zgarmaganda
}
```

</details>

---

## When to Use — Dependency Arrays

### Nazariya

`useEffect`, `useMemo`, `useCallback`, `useImperativeHandle` — har biri deps array qabul qiladi. Object/array/function deps — `useMemo`/`useCallback` bilan stabilize.

**`useEffect` deps trap:**

```tsx
function Component({ items }: { items: Item[] }) {
  // ❌ Inline object deps
  useEffect(() => {
    fetch('/api', { body: JSON.stringify({ items, limit: 10 }) });
  }, [{ items, limit: 10 }]);  // ❌ Har render yangi
  
  // ✅ useMemo
  const params = useMemo(() => ({ items, limit: 10 }), [items]);
  
  useEffect(() => {
    fetch('/api', { body: JSON.stringify(params) });
  }, [params]);
}
```

`useEffect` deps array'da object — `useMemo` bilan stabilize. Cross-ref [`16-useeffect.md`](16-useeffect.md) "Object/Array Deps".

**Custom hook deps:**

```tsx
function useFetch<T>(url: string, options: RequestInit) {
  // ❌ options har render yangi (caller'da inline)
  useEffect(() => {
    fetch(url, options);
  }, [url, options]);
}

// Caller — options stabilize qilish kerak:
function Component() {
  const options = useMemo<RequestInit>(
    () => ({ method: 'POST', headers: { 'Content-Type': 'application/json' } }),
    []
  );
  
  useFetch('/api', options);
}
```

Custom hook'ning deps — caller'da stabilize qilinishi kerak.

**Callback deps:**

```tsx
function Component({ data }: { data: Item[] }) {
  // ❌ handler har render yangi
  useEffect(() => {
    eventBus.on('change', handler);
    return () => eventBus.off('change', handler);
  }, [handler]);  // handler har render yangi → effect har render
  
  const handler = (event: Event) => process(event, data);
  
  // ✅ useCallback
  const handler = useCallback((event: Event) => process(event, data), [data]);
  
  useEffect(() => {
    eventBus.on('change', handler);
    return () => eventBus.off('change', handler);
  }, [handler]);  // data o'zgarmasa effect bir marta
}
```

**`react-hooks/exhaustive-deps` linter:**

```tsx
// Linter har deps array tekshiradi — exhaustive-deps rule
useEffect(() => {
  fetch(url, options);
}, [url]);  // ⚠️ Linter: 'options' deps'da yo'q

useMemo(() => compute(a, b), [a]);  // ⚠️ Linter: 'b' deps'da yo'q

useCallback((x) => x + a, []);  // ⚠️ Linter: 'a' deps'da yo'q
```

Linter — barcha reactive deps majburlaydi. Object/function/array deps — `useMemo`/`useCallback` bilan stabilize qilish tavsiya.

<details>
<summary><strong>Under the Hood</strong></summary>

**Linter algorithm:**

```ts
// react-hooks/exhaustive-deps soddalashtirilgan
function lintHookCall(node) {
  // 1. Hook call topish (useEffect, useMemo, useCallback, ...)
  // 2. Callback va deps array AST'dan olish
  // 3. Callback ichida ishlatilgan identifier'lar
  // 4. Reactive identifier'lar (props, state, hook returns)
  // 5. Deps bilan compare
  // 6. Mismatch → warning
}
```

Linter AST analysis bilan — runtime'da ishlamaydi. Compile-time check.

**Stable references — eslint exception:**

```tsx
const [state, setState] = useState(0);

useEffect(() => {
  setState(1);  // setState — stable
}, []);  // ✅ setState deps'da kerak emas (linter biladi)

const [s, dispatch] = useReducer(reducer, init);

useEffect(() => {
  dispatch({ type: 'a' });  // dispatch — stable
}, []);  // ✅ dispatch deps'da kerak emas
```

ESLint plugin `useState` setter va `useReducer` dispatch'larni stable deb biladi — deps'da yozish shart emas (warning yo'q).

**Source citation:**

- `react-hooks/exhaustive-deps` — facebook/react `packages/eslint-plugin-react-hooks`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Effect deps stabilize:**

```tsx
function ChatRoom({ userId }: { userId: string }) {
  // Connection options
  const options = useMemo(
    () => ({ userId, retry: 3, timeout: 5000 }),
    [userId]
  );
  
  useEffect(() => {
    const conn = createConnection(options);
    conn.connect();
    return () => conn.disconnect();
  }, [options]);  // userId o'zgarmasa — connection stable
}
```

**Misol 2 — Custom hook'ga callback uzatish:**

```tsx
function useThrottledFn<T extends (...args: any[]) => void>(
  fn: T,
  delay: number
): T {
  const lastCallRef = useRef(0);
  
  return useCallback(((...args: Parameters<T>) => {
    const now = Date.now();
    if (now - lastCallRef.current >= delay) {
      lastCallRef.current = now;
      fn(...args);
    }
  }) as T, [fn, delay]);
}

// Consumer
function Component({ onSearch }: { onSearch: (q: string) => void }) {
  // ❌ Har render yangi — throttle bekor
  const handler = (q: string) => onSearch(q);
  
  // ✅ useCallback
  const handler = useCallback((q: string) => onSearch(q), [onSearch]);
  const throttled = useThrottledFn(handler, 300);
}
```

**Misol 3 — Computed deps:**

```tsx
function Component({ items, threshold }: { items: Item[]; threshold: number }) {
  // Filtered items — boshqa hooks'da deps sifatida
  const filtered = useMemo(
    () => items.filter(i => i.value > threshold),
    [items, threshold]
  );
  
  // Effect — filtered'ga reactive
  useEffect(() => {
    console.log('Filtered:', filtered.length);
  }, [filtered]);
  
  // useCallback — filtered'ga reactive
  const handleClick = useCallback(() => {
    sendAnalytics(filtered);
  }, [filtered]);
}
```

`useMemo` natija boshqa hook'larda deps sifatida ishlatiladi — chain'dagi har bir element stable bo'lishi kerak.

**Misol 4 — Linter exhaustive-deps fix:**

```tsx
function Component({ id, type }: { id: string; type: string }) {
  // ❌ Linter warning: 'type' deps'da yo'q
  const fetcher = useCallback(async () => {
    return await fetch(`/api/${type}/${id}`);
  }, [id]);
  
  // ✅ Fix
  const fetcher = useCallback(async () => {
    return await fetch(`/api/${type}/${id}`);
  }, [id, type]);
}
```

</details>

---

## When NOT to Use — Premature Optimization

### Nazariya

Memoization — bepul emas. Har `useMemo`/`useCallback` chaqiruvi:

1. Hook chain'da slot egallaydi (memory)
2. Deps array allocation
3. `Object.is` comparison har render'da
4. Saqlangan tuple memory

Sodda compute uchun — overhead asl compute'dan ko'p:

```tsx
// ❌ Befoyda — primitive compute trivial
const sum = useMemo(() => a + b, [a, b]);
// useMemo overhead (Hook chain slot + deps comparison) >>> a + b cost
// Trivial arithmetic uchun memoization foyda bermaydi

// ✅ Inline
const sum = a + b;
```

**Premature optimization principle:**

> "Premature optimization is the root of all evil." — Donald Knuth

React docs (`react.dev`) ham aytadi:

> "Most of the time, you don't need to optimize re-renders. React is fast by default."

**Profile birinchi, optimize keyin:**

```
1. Komponent ishlaydi (functionality)
2. DevTools Profiler bilan render measure
3. Slow render aniqlash (60fps frame budget ≈ 1000/60 ≈ 16.7ms — undan oshsa frame tushib qoladi)
4. Bottleneck topish
5. Optimize (useMemo / useCallback / React.memo)
6. Re-measure — improvement test
```

Profiler'siz optimization — taxmin (ko'pincha noto'g'ri).

**Memoization befoyda holatlar:**

| Holat | Sabab |
|-------|-------|
| Primitive compute (`a + b`) | Compute < lookup overhead |
| Single use of value | Re-compute cost = memo cost |
| Different deps every render | Memo har gal invalidate (befoyda) |
| Component small + few props | Re-render arzon |
| `React.memo` qilinmagan child | Stable props befoyda |
| Context dep | Memo context subscription'ni bloklamaydi |

**Mythical "useMemo always" anti-pattern:**

```tsx
// ❌ Anti-pattern — har joyda useMemo
function Component({ a, b }: { a: number; b: number }) {
  const sum = useMemo(() => a + b, [a, b]);  // Befoyda
  const product = useMemo(() => a * b, [a, b]);  // Befoyda
  const message = useMemo(() => `${a} + ${b} = ${sum}`, [a, b, sum]);  // Befoyda
  const handler = useCallback(() => console.log(sum), [sum]);  // Befoyda (Child memo emas)
  
  return <div>{message}</div>;
  // Hech qanday performance gain — faqat code noise + overhead
}

// ✅ Sodda
function Component({ a, b }: { a: number; b: number }) {
  const sum = a + b;
  const product = a * b;
  const message = `${a} + ${b} = ${sum}`;
  const handler = () => console.log(sum);
  
  return <div>{message}</div>;
}
```

`useMemo`/`useCallback` har joyda — code noise + overhead, performance gain yo'q.

**Real cost — code maintenance:**

`useMemo`/`useCallback` har biri Hook slot, deps array, source code overhead beradi. Bundle'ga ta'siri React core'da included — qo'shimcha import yo'q. Asosiy cost — code complexity (deps to'g'ri saqlash, ESLint warning'lar, closure trap'lar).

React Compiler (1.0 stable 2025, React 17 va undan yuqori versiyalar bilan mos) — bu manual maintenance'ni olib tashlaydi: build-time auto-memoization (Babel plugin), runtime overhead qo'shmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Memoization cost analysis:**

Per-call overhead:
- Hook chain slot (linked list traversal)
- Deps array allocation
- `areHookInputsEqual`: O(N) deps × `Object.is` comparison
- Memory: Hook obyekt + tuple

Compute cost qiyosi:
- Primitive arithmetic — engine optimization bilan minimal
- Array.map (kichik array) — tez
- Array.sort/filter (katta array) — sezilarli kechikish potential

Decision:
- Heavy compute (sort/transformations large data) → Memoization foydali
- Trivial compute → Memoization befoyda

Aniq raqamlar uchun DevTools Profiler bilan measurement.

**`React.memo` cost:**

`shallowEqual` props comparison — O(N) props × `Object.is`. Bailout — komponent re-render qilinmaydi (deep tree skip).

Trade-off:
- Heavy komponent + stable props → katta foyda (subtree skip)
- Sayoz komponent yoki always-changing props → marginal yoki overhead

Real-world — DevTools Profiler measure.

**Source citation:**

- React docs "When to use useMemo" — react.dev/reference/react/useMemo#optimizing-a-custom-hook
- Mark Erikson "A (Mostly) Complete Guide to React Rendering Behavior" — blog.isquaredsoftware.com

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Befoyda useMemo:**

```tsx
// ❌ Trivial compute
function Component({ price, quantity }: { price: number; quantity: number }) {
  const total = useMemo(() => price * quantity, [price, quantity]);
  // 1 multiplication — useMemo overhead asl compute'dan sezilarli ko'p
  
  return <div>{total}</div>;
}

// ✅ Inline
function Component({ price, quantity }: { price: number; quantity: number }) {
  const total = price * quantity;
  return <div>{total}</div>;
}
```

**Misol 2 — useMemo with always-changing deps:**

```tsx
// ❌ Deps har render'da yangi — memo har gal invalidate
function Component({ items }: { items: Item[] }) {
  const filtered = useMemo(
    () => items.filter(i => i.active),
    [items.filter(i => i.active)]  // ❌ Filter natijasi har render yangi
  );
  
  // useMemo befoyda — har gal cache miss
}

// ✅ Sodda
function Component({ items }: { items: Item[] }) {
  const filtered = items.filter(i => i.active);
  return <ul>...</ul>;
}
```

**Misol 3 — useCallback without React.memo child:**

```tsx
// ❌ Befoyda — Child memo emas
function Parent() {
  const handler = useCallback(() => console.log('hi'), []);
  return <Child onClick={handler} />;
}

function Child({ onClick }: { onClick: () => void }) {  // No React.memo
  return <button onClick={onClick}>Click</button>;
}

// Parent re-render → Child re-render (har holatda)
// useCallback befoyda

// ✅ Sodda inline
function Parent() {
  return <Child onClick={() => console.log('hi')} />;
}
```

**Misol 4 — Profile-driven optimization:**

```tsx
function Dashboard({ data }: { data: BigData }) {
  // 1. Birinchi: sodda yozing
  const stats = computeStats(data);  // Slow?
  const chart = renderChart(data);   // Slow?
  
  // 2. DevTools Profiler bilan measure
  // 3. Slow function aniqlash
  // 4. Optimize
  
  const stats = useMemo(() => computeStats(data), [data]);
  // Faqat slow function uchun
}
```

Optimization — measurement asosida, taxmin emas.

**Misol 5 — Code clarity vs micro-optimization:**

```tsx
// ⚠️ Micro-optimization — code noise
function User({ user }: { user: User }) {
  const fullName = useMemo(() => `${user.firstName} ${user.lastName}`, [user]);
  const initials = useMemo(() => `${user.firstName[0]}${user.lastName[0]}`, [user]);
  const formatted = useMemo(() => formatUser(user), [user]);
  
  return <div>{fullName} ({initials})</div>;
}

// ✅ Clean
function User({ user }: { user: User }) {
  const fullName = `${user.firstName} ${user.lastName}`;
  const initials = `${user.firstName[0]}${user.lastName[0]}`;
  const formatted = formatUser(user);
  
  return <div>{fullName} ({initials})</div>;
}
```

Sodda code — readable, maintainable. Performance — DevTools profile birinchi.

</details>

---

## Cost vs Benefit Trade-off

### Nazariya

Memoization — trade-off. Har `useMemo`/`useCallback`:

**Cost:**
1. Code complexity — wrapper, deps array
2. Bundle size — Hook implementation
3. Runtime overhead — Hook chain, comparison, allocation
4. Maintenance — deps to'g'ri saqlash, ESLint warnings
5. Debug difficulty — closure traps, stale data

**Benefit:**
1. Performance — expensive compute skip
2. Re-render skip — `React.memo` bilan
3. Stable references — useEffect deps, Context value
4. Predictable identity — third-party libs

**Decision matrix:**

| Compute cost | Re-render frequency | React.memo? | Tanlov |
|--------------|---------------------|-------------|--------|
| Trivial | Low | No | Inline |
| Trivial | High | No | Inline |
| Trivial | Low | Yes | Inline (stable props bo'lmasa memo bailout ishlamaydi) |
| Trivial | High | Yes | useMemo / useCallback |
| Expensive | Low | Any | useMemo |
| Expensive | High | Any | useMemo (KRITIK) |

**Profiling — birinchi qadam:**

```tsx
// React DevTools Profiler
// 1. Record session
// 2. Slow renders aniqlash
// 3. Component'ning render duration
// 4. Bottleneck topish
// 5. Optimize selectively
```

DevTools Profiler — chuqur [`34-profiling.md`](34-profiling.md).

**Real-world heuristics:**

```
✅ Memoize:
- O(N²) sort/filter
- Heavy data transformations
- Object/function passed to React.memo child
- Context Provider value
- useEffect deps (object/function)

❌ Don't memoize:
- Primitive compute (sum, concat)
- Single-use values
- Constants (use module-level)
- Different every render (memo befoyda)
- React.memo'siz child uchun callback
```

**`React.memo` heuristics:**

```
✅ Use:
- Heavy render (chart, large list)
- Frequent parent re-render
- Stable props (useMemo + useCallback)

❌ Don't use:
- Cheap render
- Always-changing props
- Context-heavy component (memo context subscription'ni bloklamaydi)
- Children prop (almost always changes)
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Performance trade-off — qachon memoize foyda:**

```
Re-render skip benefit:
- Heavy component (deep tree, ko'p children, expensive compute) — sezilarli
- Trivial component (sodda DOM, kam props) — minimal

Memoization overhead:
- useMemo/useCallback: Hook chain slot, deps comparison
- React.memo bailout: shallowEqual props comparison

Decision rule:
- Skip benefit > overhead → memoize
- Skip benefit ≈ overhead → don't bother
```

Aniq raqamlar device, content, React versiyasi va workload'ga bog'liq. DevTools Profiler — har holat uchun real measurement.

**Bundle impact:**

`useMemo`/`useCallback` React core'da bor — qo'shimcha bundle yo'q. `React.memo` wrapper component yaratadi — har wrapper minimal source overhead. Asosiy cost'i runtime overhead emas, balki maintenance complexity (deps array, closure trap'lar).

**Source citation:**

- React DevTools Profiler docs — react.dev
- "Will it Re-render?" — webdev workshop (Josh W. Comeau)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Selective memoization:**

```tsx
function Dashboard({ data }: { data: RawData }) {
  // ✅ Heavy — useMemo
  const aggregated = useMemo(() => aggregate(data), [data]);  // O(N²)
  
  // ❌ Trivial — inline
  const itemCount = data.items.length;
  
  // ✅ Stable for memo'd child
  const handleSelect = useCallback((id: string) => {
    console.log(id);
  }, []);
  
  return (
    <div>
      <p>Items: {itemCount}</p>
      <Aggregated data={aggregated} onSelect={handleSelect} />
    </div>
  );
}

const Aggregated = React.memo(function Aggregated({
  data,
  onSelect,
}: {
  data: AggregatedData;
  onSelect: (id: string) => void;
}) {
  return <table>...</table>;
});
```

`useMemo` faqat heavy compute uchun. `useCallback` `React.memo` bilan birga.

**Misol 2 — Profile result example:**

```
DevTools Profiler (nisbiy magnitude — aniq qiymatlar device/workload'ga bog'liq):

Component       | Render Time | Self Time
Dashboard       | dominant    | minor
├─ Header       | negligible  | negligible
├─ Sidebar      | minor       | minor
└─ ChartArea    | dominant    | negligible
   └─ Chart     | dominant    | dominant   ← Bottleneck (deyarli butun render vaqti)

Optimization target: Chart component
Strategy: useMemo for chart data, React.memo for Chart, useCallback for handlers
```

Profile-driven — bottleneck'ka focus.

**Misol 3 — Cost-benefit comparison:**

```tsx
// Scenario A — Heavy component
const HeavyList = React.memo(function HeavyList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map(i => (
        <li key={i.id}>
          <ExpensiveItem data={i} />
        </li>
      ))}
    </ul>
  );
});

// Parent
function Parent() {
  const [filter, setFilter] = useState('');
  const items = useMemo(() => filterItems(allItems, filter), [filter]);
  
  return (
    <>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      <HeavyList items={items} />
      {/* filter change'da items rebuilds, HeavyList re-renders */}
      {/* HeavyList ichida har item ExpensiveItem — slow */}
      {/* Memoization MUHIM */}
    </>
  );
}
```

```tsx
// Scenario B — Trivial component
function TrivialDisplay({ value }: { value: number }) {
  return <span>{value}</span>;
}

function Parent() {
  const [x, setX] = useState(0);
  const doubled = useMemo(() => x * 2, [x]);  // ❌ Befoyda
  
  return <TrivialDisplay value={doubled} />;
  // Compute trivial, render trivial — memo befoyda
}
```

</details>

---

## React Compiler — Auto-Memoization

### Nazariya

React Compiler (avval "React Forget") — build-time tool. JSX va Hooks code'ini tahlil qilib **avtomatik memoization** qo'yadi. Manual `useMemo`/`useCallback` kerak emas bo'lib qolishi mumkin. 2024'da open-source release, **1.0 stable 2025**. React 17 va undan yuqori versiyalar bilan mos (opt-in Babel plugin, R19'ga bog'lanmagan).

**Compiler nima qiladi:**

```tsx
// Yozilgan kod (oddiy React)
function Component({ items, filter }: Props) {
  const filtered = items.filter(i => i.name.includes(filter));
  const handler = (id: string) => console.log(id);
  
  return <List items={filtered} onClick={handler} />;
}

// Compiler output (taxminiy — slot'lar boshida sentinel symbol bilan to'ldiriladi)
function Component({ items, filter }: Props) {
  const $ = _c(4);  // Memo cache — har slot dastlab sentinel
  
  let filtered;
  if ($[0] !== items || $[1] !== filter) {
    filtered = items.filter(i => i.name.includes(filter));
    $[0] = items;
    $[1] = filter;
    $[2] = filtered;
  } else {
    filtered = $[2];
  }
  
  let handler;
  if ($[3] === Symbol.for('react.memo_cache_sentinel')) {
    handler = (id: string) => console.log(id);
    $[3] = handler;
  } else {
    handler = $[3];
  }
  
  return <List items={filtered} onClick={handler} />;
}
```

Compiler manual `useMemo`/`useCallback` o'rniga avtomatik cache infrastructure qo'yadi. Developer manual yozish kerak emas.

**Foydalar:**

1. **Code clarity** — `useMemo`/`useCallback` boilerplate yo'q
2. **Performance** — avtomatik optimal memoization
3. **Reduced bug surface** — closure traps, stale deps yo'q
4. **Bundle size** — manual memoization wrappers olib tashlanishi mumkin
5. **Optimal granularity** — Compiler granular kesh qo'yadi

**Cheklov:**

1. **Rules of React** — Compiler input pure bo'lishi shart (no side effects in render, no mutation, etc.)
2. **Opt-in** — Babel plugin orqali yoqiladi (default'da o'chiq)
3. **eslint-plugin-react-compiler** — code Compiler-safe ekanligini tekshiradi
4. **Library compat** — eski library'lar Compiler bilan ishlamasligi mumkin (Rules of React buzilgan code)

**Migration path:**

```
1. Install Compiler (Babel plugin)
2. eslint-plugin-react-compiler — code violations topish
3. Fix violations (Rules of React amal qilish)
4. Compiler enable
5. Manual useMemo/useCallback olib tashlash (gradually)
6. Test — performance regression yo'qmi
```

**Ko'p hollarda — manual memoization saqlanadi:**

Compiler stable bo'lsa ham, mavjud kod'larda manual `useMemo`/`useCallback` o'rinida qoladi (backward compat). Yangi loyihalar Compiler bilan boshlanishi mumkin; eski loyihalar gradual migration qiladi (ESLint plugin bilan Rules of React violations fix → Compiler enable → manual hooks step-by-step olib tashlash).

**Bu fayl context'da:**

`useMemo`/`useCallback` — hali React'ning standart toolkit'i. Compiler joriy etilgan loyihalarda kamroq yoziladi, lekin internal mexanika (Hook chain, deps comparison) — Compiler-generated code shu primitive'lar ustida quriladi. Manual hooks bilim Compiler era'da ham foydali (debugging, migration, custom hooks).

Cross-ref [`31-react-compiler.md`](31-react-compiler.md) — Compiler chuqur, Rules of React, eslint plugin, migration.

<details>
<summary><strong>Under the Hood</strong></summary>

**Compiler architecture:**

```
Input (JSX + Hooks code)
  ↓
Babel parse → AST
  ↓
Compiler analysis:
  - Pure functions detection
  - Reactive variables tracking
  - Memoization opportunities
  ↓
Code generation:
  - Cache slots ($_c)
  - Conditional re-compute
  - Stable references
  ↓
Output (memoized code)
```

Compiler — static analysis tool. Runtime'da ishlamaydi (build-time).

**Rules of React (Compiler input):**

```
1. No side effects in render
2. No mutation of props/state
3. Idempotent (Strict Mode 2x render OK)
4. Pure functions (deterministic)
5. Hooks rules
```

Bu rules buzilsa — Compiler noto'g'ri memoize qilishi mumkin. ESLint plugin tekshiradi.

**Cache mexanizmi:**

```ts
// Compiler runtime — _c helper
function _c(size: number): unknown[] {
  return new Array(size).fill(REACT_MEMO_SENTINEL);
}

// Cache — Hook chain'da ($-variable)
// Per-render slot allocation
// Avtomatik invalidation Reactive deps asosida
```

Compiler internal — React core'da special hook chain.

**Source citation:**

- React Compiler docs — react.dev/learn/react-compiler
- Compiler RFC — reactjs/rfcs

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Manual vs Compiler:**

```tsx
// Manual (R18 / Compiler'siz)
function Component({ items, filter, onSelect }: Props) {
  const filtered = useMemo(
    () => items.filter(i => i.name.includes(filter)),
    [items, filter]
  );
  
  const handleSelect = useCallback(
    (id: string) => onSelect(id),
    [onSelect]
  );
  
  return <List items={filtered} onSelect={handleSelect} />;
}

// Compiler era (Compiler enabled — Babel plugin)
function Component({ items, filter, onSelect }: Props) {
  const filtered = items.filter(i => i.name.includes(filter));
  const handleSelect = (id: string) => onSelect(id);
  
  return <List items={filtered} onSelect={handleSelect} />;
}
// Compiler avtomatik memoization qo'shadi
// Code: clean, less boilerplate
```

**Misol 2 — Rules of React violations:**

```tsx
// ❌ Compiler buzadi — render paytida mutation
function Component({ items }: Props) {
  items.sort();  // ❌ Mutation — Rules of React buzilishi
  return <List items={items} />;
}

// ❌ Compiler buzadi — render paytida side effect
function Component() {
  document.title = 'New';  // ❌ Side effect
  return <div>...</div>;
}

// ✅ Compiler-safe
function Component({ items }: Props) {
  const sorted = [...items].sort();  // Immutable
  return <List items={sorted} />;
}
```

ESLint plugin bu violations'ni topadi.

**Misol 3 — Manual memoization Compiler era'da:**

```tsx
// Compiler enabled — manual memo saqlanadi (legacy code)
function Component({ data }: Props) {
  const expensive = useMemo(() => heavyCompute(data), [data]);
  // Compiler bu useMemo'ni respect qiladi (manual yozilgan)
  // Yoki o'z kesh bilan replace qiladi (Compiler decision)
  
  return <Display value={expensive} />;
}
```

Backward compat — manual + Compiler birga ishlaydi.

**Misol 4 — Migration strategy:**

```tsx
// 1-bosqich — eslint-plugin-react-compiler
// Code violations topish, fix

// 2-bosqich — Compiler enable
// .babelrc:
{
  "plugins": ["babel-plugin-react-compiler"]
}

// 3-bosqich — Manual memo gradually olib tashlash
// Boshlang'ich:
const filtered = useMemo(() => items.filter(...), [items]);
// Compiler era:
const filtered = items.filter(...);  // Compiler avtomatik memoize

// 4-bosqich — Test
// DevTools Profiler — performance regression yo'qmi
```

Gradual migration — risky changes step-by-step.

**Misol 5 — Compiler advantages summary:**

```tsx
// Pre-Compiler — boilerplate
function ProductList({ products, filter, onSelect }: Props) {
  const filtered = useMemo(
    () => products.filter(p => p.name.includes(filter)),
    [products, filter]
  );
  
  const sortedFiltered = useMemo(
    () => [...filtered].sort((a, b) => a.price - b.price),
    [filtered]
  );
  
  const handleSelect = useCallback(
    (id: string) => onSelect(id),
    [onSelect]
  );
  
  const stats = useMemo(
    () => ({
      total: filtered.length,
      avgPrice: filtered.reduce((s, p) => s + p.price, 0) / filtered.length,
    }),
    [filtered]
  );
  
  return <List items={sortedFiltered} stats={stats} onSelect={handleSelect} />;
}

// Compiler era
function ProductList({ products, filter, onSelect }: Props) {
  const filtered = products.filter(p => p.name.includes(filter));
  const sortedFiltered = [...filtered].sort((a, b) => a.price - b.price);
  const handleSelect = (id: string) => onSelect(id);
  const stats = {
    total: filtered.length,
    avgPrice: filtered.reduce((s, p) => s + p.price, 0) / filtered.length,
  };
  
  return <List items={sortedFiltered} stats={stats} onSelect={handleSelect} />;
}
// Bir xil performance, sezilarli kamroq boilerplate (useMemo/useCallback wrapper'lar yo'q)
```

</details>

---

## Decision Guide

### Nazariya

Memoization decision — kontekst asosida. Quyidagi savollar bilan:

```
1. React Compiler available?
   ha → Manual yozmaslik, Compiler ishonib (rules amal qilingan)
   yo'q → 2-savol

2. Bu computed value'mi (useMemo) yoki function (useCallback)?
   value → useMemo
   function → useCallback

3. Compute expensive'mi (DevTools Profiler bilan o'lchangan)?
   ha (sezilarli render time olayotgan) → Memoize
   yo'q → 4-savol

4. Reference identity muhim'mi?
   ha (React.memo child, useEffect deps, Context value) → Memoize
   yo'q → Inline (no memo)

5. Deps stable bo'la oladimi?
   ha → Memoize
   yo'q (har render yangi) → Memoize befoyda
```

**Decision matrix:**

| Holat | Memoize? |
|-------|----------|
| Primitive arithmetic | ❌ |
| String concat | ❌ |
| Array.map (small) | ❌ |
| Array.sort/filter (large) | ✅ |
| Object passed to React.memo'd child | ✅ |
| Function passed to React.memo'd child | ✅ |
| Function passed to useEffect deps | ✅ |
| Context Provider value | ✅ |
| Single-use value | ❌ |
| Constants | Module-level (no memo) |
| React Compiler enabled | ❌ (Compiler avtomatik memoize) |

**Heuristics:**

```
"useMemo when":
- Compute sezilarli render time'ga ta'sir qiladi (Profiler measurement)
- Object/array/function passed to memo'd child
- Context Provider value
- Custom hook returns object/array/function (consumer'lar deps'ga ishlatishi mumkin)

"useCallback when":
- Function passed to React.memo child
- Function passed to useEffect deps
- Function passed to custom hook (hook ichidagi useEffect deps uchun)

"Don't memoize when":
- Compute trivial
- Single-use
- Different every render anyway
- Compiler available

"Always memoize" anti-pattern — code noise + overhead, no gain
```

**Compiler era guidance:**

```
Compiler joriy etilmagan loyihalarda:
- Manual useMemo / useCallback saqlash
- Profile-driven optimization
- React.memo bilan birga

Compiler enabled loyihalarda:
- Manual yozmaslik (Compiler avtomatik)
- Rules of React amal qilish (eslint-plugin-react-compiler check)
- Manual memo legacy code'da saqlanadi (gradual migration)
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`useMemo` chuqur ishonish:**

`useMemo` cache'lashni **kafolat bermaydi**. React docs (react.dev `useMemo` reference):

> "You may rely on useMemo as a performance optimization, not as a semantic guarantee."

React kelajakda cache'larni invalidate qilishi mumkin (masalan, offscreen rendering yoki memory pressure scenario'larida). Demak `useMemo` faqat performance optimization — semantic correctness uchun emas:

```tsx
// ❌ Anti-pattern — useMemo'ga semantic guarantee deb ishonish
const id = useMemo(() => crypto.randomUUID(), []);
// React kelajakda re-compute qilishi mumkin → id o'zgaradi → bug

// ✅ useState lazy — kafolat
const [id] = useState(() => crypto.randomUUID());
// ID hayotida bir xil
```

`useState` lazy — guaranteed bir martalik. `useMemo` — performance hint.

**Compiler kafolat:**

Compiler-generated memoization ham — performance optimization. Compiler `useState`'ga turn qiladi semantic guarantee kerak bo'lsa.

**Source citation:**

- React docs `useMemo` "Should you add useMemo everywhere?" — react.dev
- "Before You memo()" — Dan Abramov post

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Decision matrix in action:**

```tsx
function Dashboard({ data, filter, onItemClick }: Props) {
  // ❌ Trivial — inline
  const itemCount = data.length;
  
  // ✅ Heavy compute — useMemo
  const stats = useMemo(() => calculateStats(data), [data]);
  
  // ✅ Object passed to memo'd child — useMemo
  const config = useMemo(() => ({ theme: 'dark', locale: 'en' }), []);
  
  // ✅ Function passed to memo'd child — useCallback
  const handleClick = useCallback((id: string) => onItemClick(id), [onItemClick]);
  
  // ❌ Single-use string — inline
  const title = `Stats: ${itemCount} items`;
  
  return (
    <div>
      <h1>{title}</h1>
      <MemoizedStats stats={stats} config={config} onItemClick={handleClick} />
    </div>
  );
}

const MemoizedStats = React.memo(StatsComponent);
```

**Misol 2 — When inline:**

```tsx
function Item({ name, count }: Props) {
  // Inline — render fast, no memo'd child, no deps
  const label = `${name} (${count})`;
  const handleClick = () => console.log(name);
  
  return <button onClick={handleClick}>{label}</button>;
}
```

Sodda komponent — har gal inline. Memo overhead asossiz.

**Misol 3 — Custom hook return:**

```tsx
function useUser(id: string) {
  const [data, setData] = useState<User | null>(null);
  
  // ✅ Function returned — useCallback (consumer deps'ga ishlatishi mumkin)
  const refetch = useCallback(() => {
    fetchUser(id).then(setData);
  }, [id]);
  
  // ✅ Object returned — useMemo (consumer destructuring deps'ga ishlatishi mumkin)
  return useMemo(() => ({ data, refetch }), [data, refetch]);
}

// Consumer
function Component({ userId }: { userId: string }) {
  const { data, refetch } = useUser(userId);
  
  useEffect(() => {
    refetch();  // refetch stable — effect bir marta
  }, [refetch]);
}
```

Custom hook return — consumer'lar deps sifatida ishlatishi mumkin → memoize tavsiya.

**Misol 4 — Compiler era simulation:**

```tsx
// Compiler enabled (babel-plugin-react-compiler)

function ProductPage({ products, filter }: Props) {
  const filtered = products.filter(p => p.category === filter);
  const sorted = [...filtered].sort((a, b) => a.price - b.price);
  const handleClick = (id: string) => analytics.track('click', { id });
  
  return <List items={sorted} onClick={handleClick} />;
}

// Compiler avtomatik:
// - filtered → memoized [products, filter]
// - sorted → memoized [filtered]
// - handleClick → stable reference
// Kod toza, performance avtomatik
```

**Misol 5 — Profile-driven decision:**

```
1. Sodda yozish:
   function Component({ data }) {
     const stats = computeStats(data);
     const items = transformItems(data);
     return <Display stats={stats} items={items} />;
   }

2. DevTools Profiler (nisbiy magnitude):
   - Component render: dominant
   - computeStats: dominant ← bottleneck (render vaqtining katta qismi)
   - transformItems: minor
   - Display render: minor

3. Optimize bottleneck:
   const stats = useMemo(() => computeStats(data), [data]);
   // transformItems va Display memoize qilmaymiz — fast enough

4. Re-measure:
   - data o'zgarmaganda computeStats skip qilinadi
   - render time sezilarli kamayadi (faqat transformItems + Display qoladi)
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1 — `useMemo` kafolat emas

```tsx
const id = useMemo(() => crypto.randomUUID(), []);
// React kelajakda cache "unutishi" mumkin → id o'zgaradi
```

`useMemo` — performance hint, semantic guarantee emas. ID kerak bo'lsa `useState` lazy:

```tsx
const [id] = useState(() => crypto.randomUUID());  // Guaranteed
```

### Gotcha 2 — Inline object deps invalidation

```tsx
const value = useMemo(
  () => compute(a, options),
  [a, { strict: true }]  // ❌ Object literal har render yangi → cache invalidate
);

// ✅ Primitive deps yoki memoized
const value = useMemo(
  () => compute(a, { strict: true }),
  [a]
);
```

### Gotcha 3 — `useCallback` closure trap

```tsx
function Component({ count }: { count: number }) {
  const handler = useCallback(() => {
    console.log(count);  // ❌ Stale closure
  }, []);  // count deps'da yo'q
  
  // Render 1: count=0 → handler closure 0
  // Render 2: count=1 → handler closure 0 (stale)
  
  // ✅ count deps'da
  const handler = useCallback(() => console.log(count), [count]);
}
```

ESLint exhaustive-deps catches.

### Gotcha 4 — `useMemo` factory side effect

```tsx
// ❌ Strict Mode 2x call → side effect 2x
const value = useMemo(() => {
  console.log('computing');  // 2x dev'da
  analytics.track('compute');  // ❌ 2x dev'da
  return computeValue();
}, []);

// ✅ Side effect — useEffect'da
useEffect(() => {
  analytics.track('compute');
}, []);

const value = useMemo(() => computeValue(), []);
```

### Gotcha 5 — Deps array uzunligi

```tsx
const value = useMemo(
  () => compute(),
  flag ? [a, b] : [a]  // ❌ Uzunlik conditional
);
// Console (DEV): "The final argument passed to useMemo changed size between renders"
// React early-return qilmaydi: comparison qisqaroq uzunlik bo'yicha → noaniq cache
```

Deps array uzunligi har render'da bir xil bo'lishi shart — bu Rules of Hooks invariant'i.

---

## Common Mistakes

### ❌ Xato 1 — `useMemo` har joyda

```tsx
// ❌ Anti-pattern
const sum = useMemo(() => a + b, [a, b]);  // Trivial

// ✅ Inline
const sum = a + b;
```

### ❌ Xato 2 — `useCallback` `React.memo`'siz

```tsx
// ❌ Befoyda
function Parent() {
  const handler = useCallback(() => {}, []);
  return <Child onClick={handler} />;  // Child memo emas
}

// ✅ Yoki memo qil, yoki callback olib tashlash
const Child = React.memo(ChildComponent);
```

### ❌ Xato 3 — Object/function deps stabilize qilmaslik

```tsx
// ❌ Inline obj deps
useEffect(() => {
  fetch('/api', { body: JSON.stringify(opts) });
}, [{ ...opts }]);  // Har render yangi

// ✅ useMemo
const stableOpts = useMemo(() => opts, [opts]);
useEffect(() => {
  fetch('/api', { body: JSON.stringify(stableOpts) });
}, [stableOpts]);
```

### ❌ Xato 4 — `useMemo` cache semantic guarantee deb

```tsx
// ❌ ID kafolatlanmaydi
const id = useMemo(() => crypto.randomUUID(), []);

// ✅ useState lazy
const [id] = useState(() => crypto.randomUUID());
```

### ❌ Xato 5 — Linter ignore

```tsx
// ❌ exhaustive-deps disable
const handler = useCallback(() => {
  doStuff(value);
}, []);  // eslint-disable-next-line — ❌ stale closure

// ✅ Deps to'g'ri
const handler = useCallback(() => doStuff(value), [value]);
```

---

## Amaliy Mashqlar

### Mashq 1 — `usePrevious` (Oson)

`usePrevious` hook yozing — `useMemo`'ga bog'liq emas, lekin reference identity demonstrate qiladi.

```tsx
function usePrevious<T>(value: T): T | undefined {
  // Implement using useRef + useEffect (cross-ref 18)
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

`useMemo`/`useCallback` — bu hook uchun kerak emas (`useRef` reference identity stable). Memoization har joyda kerak emas — task'ga mos hook tanlash.

</details>

### Mashq 2 — `useStableCallback` (Oson)

Always-stable callback hook. Latest closure pattern bilan (cross-ref [`18-useref.md`](18-useref.md)).

```tsx
function useStableCallback<T extends (...args: any[]) => any>(callback: T): T {
  // Implement
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useStableCallback<T extends (...args: any[]) => any>(callback: T): T {
  const ref = useRef(callback);
  
  useEffect(() => {
    ref.current = callback;
  });
  
  return useCallback(((...args: Parameters<T>) => {
    return ref.current(...args);
  }) as T, []);
}
```

**Tushuntirish:**

- `useRef` — latest callback
- `useEffect` — har render'da ref yangilash
- `useCallback` empty deps — returned function stable
- Wrapper function `ref.current(...args)` — har gal latest callback

`useCallback` consumer's deps qoidalarini bypass qilish (React'ning experimental `useEffectEvent`'iga o'xshash, lekin manual).

</details>

### Mashq 3 — `useDeepMemo` (O'rta)

Deep equality check bilan `useMemo` — object/array deep comparison.

```tsx
function useDeepMemo<T>(factory: () => T, deps: unknown[]): T {
  // Implement using deep equality
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function deepEqual(a: unknown, b: unknown): boolean {
  if (a === b) return true;
  if (a === null || b === null) return false;
  if (typeof a !== typeof b) return false;
  
  if (Array.isArray(a) && Array.isArray(b)) {
    if (a.length !== b.length) return false;
    return a.every((item, i) => deepEqual(item, b[i]));
  }
  
  if (typeof a === 'object' && typeof b === 'object') {
    const objA = a as Record<string, unknown>;
    const objB = b as Record<string, unknown>;
    const keysA = Object.keys(objA);
    const keysB = Object.keys(objB);
    if (keysA.length !== keysB.length) return false;
    return keysA.every(k => deepEqual(objA[k], objB[k]));
  }
  
  return false;
}

function useDeepMemo<T>(factory: () => T, deps: unknown[]): T {
  const ref = useRef<{ deps: unknown[]; value: T } | null>(null);
  
  if (ref.current === null || !deepEqual(ref.current.deps, deps)) {
    ref.current = { deps, value: factory() };
  }
  
  return ref.current.value;
}

// Usage
function Component({ filter }: { filter: { category: string; tags: string[] } }) {
  // Standart useMemo — har render'da filter yangi → invalidate
  // useDeepMemo — deep compare bilan, content bir xil bo'lsa cache hit
  const filtered = useDeepMemo(() => doStuff(filter), [filter]);
}
```

**Tushuntirish:**

- Deep equality — recursive comparison
- `useRef` — eski deps + value saqlash
- Render paytida ref mutation — Strict Mode'da safe (idempotent)

⚠️ Performance: deep equality O(N) — tez-tez ishlatish performance hit. Faqat aniq use case (e.g., user filter object). Production'da `use-deep-compare-effect` library yoki normalize state better.

</details>

### Mashq 4 — `useMemoizedFn` Library Pattern (O'rta)

Always-memoized function — `useCallback` empty deps with latest closure (Mashq 2 simplified).

```tsx
// ahooks library'sidagi useMemoizedFn pattern
function useMemoizedFn<T extends (...args: any[]) => any>(fn: T): T {
  // Implement
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useMemoizedFn<T extends (...args: any[]) => any>(fn: T): T {
  const fnRef = useRef<T>(fn);
  fnRef.current = fn;  // Render paytida latest fn saqlanadi
  
  const memoizedFn = useRef<T | null>(null);
  if (memoizedFn.current === null) {
    memoizedFn.current = function (this: unknown, ...args: Parameters<T>) {
      return fnRef.current.apply(this, args);
    } as T;
  }
  
  return memoizedFn.current;
}

// Usage
function Parent({ data }: { data: Item[] }) {
  // Always-stable callback — data o'zgarsa ham bir xil reference
  const handleClick = useMemoizedFn((id: string) => {
    console.log(id, data);  // Latest data accessible
  });
  
  return <Child onClick={handleClick} />;  // Child memo bilan stable
}
```

**Tushuntirish:**

- `fnRef` — latest function
- `memoizedFn` — once-created stable wrapper
- Wrapper `fnRef.current.apply(this, args)` — latest fn chaqiriladi

Bu pattern — ahooks (Alibaba), useEvent (RFC) ekvivalent. Production'da useful when stable reference + latest closure birga kerak.

</details>

### Mashq 5 — `useDebounce` with `useMemo` (Qiyin)

Debounced value hook — `useMemo` bilan debounced function yaratish.

```tsx
function useDebounce<T>(value: T, delay: number): T {
  // Implement
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  
  return debounced;
}

// Variant with debounce function
function useDebouncedCallback<T extends (...args: any[]) => void>(
  callback: T,
  delay: number
): T {
  const callbackRef = useRef(callback);
  
  useEffect(() => {
    callbackRef.current = callback;
  });
  
  // useMemo — debounced function bir marta yaratiladi (delay o'zgarmasa)
  const debounced = useMemo(() => {
    let timeoutId: ReturnType<typeof setTimeout> | null = null;
    
    return ((...args: Parameters<T>) => {
      if (timeoutId !== null) clearTimeout(timeoutId);
      timeoutId = setTimeout(() => {
        callbackRef.current(...args);
      }, delay);
    }) as T;
  }, [delay]);
  
  return debounced;
}

// Usage
function SearchBox() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);
  
  useEffect(() => {
    if (debouncedQuery) {
      fetch(`/api/search?q=${debouncedQuery}`);
    }
  }, [debouncedQuery]);
  
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}

function CommentForm() {
  const debouncedSave = useDebouncedCallback((text: string) => {
    fetch('/api/draft', { method: 'POST', body: text });
  }, 500);
  
  return (
    <textarea onChange={(e) => debouncedSave(e.target.value)} />
  );
}
```

**Tushuntirish:**

`useDebounce` — value debounce (cross-ref [`16-useeffect.md`](16-useeffect.md) Mashq 1).

`useDebouncedCallback`:
- `useRef` — latest callback
- `useMemo` — debounce wrapper (delay o'zgarmasa stable)
- `setTimeout` cleanup — har call'da eski timeout bekor

Hybrid pattern: `useMemo` (memoization) + `useRef` (latest closure) + `useEffect` (sync). Production library pattern (lodash debounce equivalent).

</details>

---

## Xulosa

`useMemo` va `useCallback` — React'ning memoization hook'lari. Asosiy fikrlar:

- **Memoization concept** — input cache → output reuse. JavaScript engine'larda standart optimization. React'da: (1) expensive compute skip, (2) referential identity stabilize.
- **`useMemo<T>(factory, deps)`** — factory chaqirib natijani saqlaydi. Deps `Object.is` bilan compare, bir xil bo'lsa cache hit.
- **`useCallback<T>(callback, deps)`** — callback'ni saqlaydi (chaqirmaydi). Deps bir xil bo'lsa eski reference qaytariladi.
- **Texnik ekvivalentlik** — `useCallback(fn, deps) ≡ useMemo(() => fn, deps)`. `useCallback` aslida `useMemo`'ning sintaktik shakli (function reference uchun convenience). Internal'da deyarli identik implementation.
- **Mexanika** — `Hook.memoizedState = [value, deps]` tuple. Mount'da factory chaqiriladi (`useMemo`) yoki callback saqlanadi (`useCallback`). Update'da `areHookInputsEqual` deps comparison: bir xil → cache hit, farq → re-compute.
- **`Object.is` comparison** — primitive value equality, object reference equality. NaN-aware (`Object.is(NaN, NaN) === true`). Object/array/function deps har render yangi reference → memo invalidate.
- **When to use** — `React.memo` bilan birga (props identity stabilize), useEffect/useCallback deps array (object/function deps), Context Provider value, expensive compute. Custom hook return (consumer deps'ga ishlatishi mumkin).
- **When NOT to use** — primitive compute (overhead > benefit), single-use values, always-changing deps (memo befoyda), `React.memo`'siz child uchun callback, Context dep (memo context subscription'ni bloklamaydi).
- **Cost vs benefit** — har memoization Hook chain slot, deps allocation, comparison overhead. Profile-driven (DevTools Profiler). "useMemo har joyda" — anti-pattern.
- **React Compiler (1.0 stable 2025) — paradigmasi o'zgartiradi:** auto-memoization manual `useMemo`/`useCallback` o'rniga. Compiler Rules of React amal qilingan code uchun avtomatik cache infrastructure qo'yadi (build-time, Babel plugin). React 17 va undan yuqori versiyalar bilan mos. Manual hooks legacy code'da saqlanadi (backward compat). Cross-ref [`31-react-compiler.md`](31-react-compiler.md).
- **`useMemo` semantic guarantee emas** — performance hint. React kelajakda cache "unutishi" mumkin. Semantic-critical (ID generation, instance identity) holatlarida `useState` lazy ishlatish kerak.
- **`react-hooks/exhaustive-deps` linter** — barcha reactive deps majburlaydi. Stable references (`setState`, `dispatch`) deps'da kerak emas (linter biladi).
- **Real qo'llash patternlari [`33-optimization.md`](33-optimization.md)** da chuqur — `React.memo` strategiyalari, custom comparator, key-based reset, profiling-driven optimization.

**Keyingi bo'lim:** [22-concurrent-hooks.md](22-concurrent-hooks.md) — `useTransition`, `useDeferredValue`, `useSyncExternalStore`, `useId` — Concurrent Mode (R18) hooks. Non-urgent updates, deferred values, external store subscription (tearing-safe), SSR-safe ID generation.

---

**Keyingi bo'lim:** [22-concurrent-hooks.md](22-concurrent-hooks.md) — Concurrent hooks (R18+): `useTransition` (non-urgent state update, isPending flag), `useDeferredValue` (defer expensive value), `useSyncExternalStore` (external store subscription tearing-safe, library author'lar uchun), `useId` (SSR-safe unique ID generation, hydration mismatch oldini olish).
