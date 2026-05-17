# Bo'lim 31: React Compiler

> React Compiler — React kodini **build-time**'da tahlil qilib, **automatic memoization** qo'shadigan kompilyator. Bu fayl Compiler'ning qanday ishlashini, source kod va kompilyatsiya qilingan output orasidagi farqni, va manual `useMemo`/`useCallback`/`React.memo`'larning kelajakdagi rolini o'rganadi. Compiler avval "React Forget" deb atalgan, R19 bilan birga eksperimental release sifatida public chiqdi va 2025-yil April'da `babel-plugin-react-compiler@1.0` stable belgisi bilan release qilindi. **Concurrent rendering — runtime mexanizm** (cross-ref `30-concurrent-react.md`), **Compiler — build-time tool**: ikki tushuncha alohida lekin bir-birini to'ldiradi (Compiler Concurrent invariants ustiga quriladi).

---

## Mundarija

- [Compiler Concept va Tarix](#compiler-concept-va-tarix)
- [Auto-Memoization Mexanizmi](#auto-memoization-mexanizmi)
- [Internal Cache Mechanism — `_c` Array](#internal-cache-mechanism--_c-array)
- [Static Analysis: HIR va Dataflow](#static-analysis-hir-va-dataflow)
- [Rules of React — Compliance Talablar](#rules-of-react--compliance-talablar)
- [`eslint-plugin-react-compiler` — Static Check](#eslint-plugin-react-compiler--static-check)
- [`babel-plugin-react-compiler` — Setup va Konfiguratsiya](#babel-plugin-react-compiler--setup-va-konfiguratsiya)
- [Compiler Output Misollari](#compiler-output-misollari)
- [Migration Path 6 Qadam](#migration-path-6-qadam)
- [Compiler Cheklovlari va Bail-Out](#compiler-cheklovlari-va-bail-out)
- [Manual Memo bilan Munosabat](#manual-memo-bilan-munosabat)
- [Library Compatibility](#library-compatibility)
- [Performance Implikatsiyalari](#performance-implikatsiyalari)
- [Future Paradigm — Manual Memoization Kelajagi](#future-paradigm--manual-memoization-kelajagi)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Compiler Concept va Tarix

### Nazariya

**React Compiler** — React kodini **build-time**'da tahlil qilib, generated JavaScript ichiga **automatic memoization logic**'ni o'rnatuvchi kompilyator. Source kod yozilganda dasturchi `useMemo`/`useCallback`/`React.memo`'ni qo'lda yozmaydi — Compiler statik analiz natijasida qaysi qiymatlarni cache qilish kerakligini aniqlaydi va kompilyatsiya output'iga o'rnatadi.

Compiler **build-time** ishlaydi — bu Babel/SWC/TypeScript transpilation pipeline'ning qismi:

```
Source code (.tsx)
       │
       ▼
Babel parser → AST
       │
       ▼
React Compiler Babel plugin
       │
       ├─ HIR (High-level Intermediate Representation) yaratiladi
       ├─ Reactivity analysis (qaysi values reactive)
       ├─ Dataflow analysis (qaysi values dependent)
       ├─ Cache slot allocation (_c[i] indexlar)
       │
       ▼
Optimized AST → Generated JavaScript (memoization bilan)
       │
       ▼
Bundler (Vite/Webpack) → bundle.js → Browser
```

> **Versiya evolyutsiyasi (Compiler):**
> - **2021–2023:** Loyihaning ichki kod-nom: "**React Forget**". Meta'da Instagram va Quest store'da ichki ishlatildi.
> - **2024 May:** React Conf 2024 — public eksperimental release. `babel-plugin-react-compiler@beta` npm'ga chiqarildi.
> - **2024 Oktabr:** RC1 (Release Candidate) status, eslint-plugin-react-compiler RC1.
> - **2025 April:** `babel-plugin-react-compiler@1.0.0` — birinchi stable release. R19 ekosistemasi bilan to'liq integration. Production ishlatish tavsiya etiladi.
> - **Compiler React versiyasidan mustaqil:** `babel-plugin-react-compiler` `target` opsiyasi orqali R17, R18, R19 (va kelajak versiyalar) bilan ishlay oladi — runtime'da `react/compiler-runtime` paketi mos kelishi yetarli. Compiler R19'ga bog'lanmagan, faqat R19 ekosistemasi bilan stable matured.
> - **Sabab:** Manual memoization (useMemo/useCallback/React.memo) — code clutter, error-prone, performance regression manbai. Compiler bu masalani **build-time'da hal qiladi** — dasturchi memoization haqida o'ylamaydi.

NIMA UCHUN Compiler kerak:

1. **Manual memoization yopiq dunyo** — har dasturchi qachon `useMemo` ishlatishni o'zicha qaror qiladi. Profile qilmasdan, premature optimization (har joyda `useMemo`) yoki under-optimization (kerakli joyda yo'q) keng tarqalgan.
2. **Reference identity gotcha'lari** — har render yangi object/array literal yangi reference, child re-render trigger. `React.memo` Context dependency'ga sezgir, Context value har render yangi bo'lsa — memo bypass.
3. **Code noise** — `useMemo`/`useCallback` chaqiriqlari logic'dan ko'p hajmni egallaydi (Idiomatik komponentda 30-50% memoization wrapper bo'lishi mumkin).
4. **Maintenance burden** — komponent o'zgarganda dependency array'ni qo'lda yangilash kerak (linter exhaustive-deps yordam beradi, lekin to'liq emas).
5. **Granularity issue** — `useMemo` bir qiymatni cache qiladi, lekin **Compiler komponent ichidagi har computation'ni alohida slot'da cache qiladi** (granular).

QANDAY ishlatiladi:

- **Build-time integration**: Babel plugin (`babel-plugin-react-compiler`) yoki SWC plugin (kelajakda).
- **Runtime overhead nol**: kompilyatsiya qilingan kod oddiy JavaScript — `react/compiler-runtime` modulidagi `c()` helper bilan cache slot'ga kirish.
- **Opt-in (R19)**: default emas — `eslint-plugin-react-compiler` violations yo'q deb tasdiqlanganda enable qilinadi.
- **Backward compatible**: manual `useMemo`/`useCallback`/`React.memo` hali ishlaydi (Compiler ularni override qilmaydi, lekin ortiqcha bo'ladi).

<details>
<summary><strong>Under the Hood</strong></summary>

Compiler architecture:

```
┌─────────────────────────────────────────────────┐
│             Source Code (.tsx)                  │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│  Babel Parser → AST (Abstract Syntax Tree)      │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│  React Compiler — Pipeline                      │
│  ┌────────────────────────────────────────┐    │
│  │ 1. Component Detection                 │    │
│  │    - PascalCase function?              │    │
│  │    - Returns JSX?                      │    │
│  │    - Hook usage?                       │    │
│  └────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────┐    │
│  │ 2. HIR (High-level IR) Construction    │    │
│  │    - Statements → instructions          │    │
│  │    - Control flow graph (CFG)          │    │
│  │    - SSA form (Static Single Assignment)│    │
│  └────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────┐    │
│  │ 3. Reactivity Analysis                  │    │
│  │    - Reactive values: props, state,     │    │
│  │      context, custom hook returns      │    │
│  │    - Non-reactive: refs, module-level  │    │
│  └────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────┐    │
│  │ 4. Dataflow Analysis                    │    │
│  │    - Value dependencies                  │    │
│  │    - Mutability inference                │    │
│  │    - Aliasing                            │    │
│  └────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────┐    │
│  │ 5. Reactive Scope Inference             │    │
│  │    - Group dependent computations       │    │
│  │    - Scope = cache boundary             │    │
│  └────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────┐    │
│  │ 6. Code Generation                      │    │
│  │    - Cache allocation (useMemoCache)    │    │
│  │    - Conditional re-execution           │    │
│  │    - Bail-out emission                  │    │
│  └────────────────────────────────────────┘    │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│  Generated JavaScript (memoized)                │
└─────────────────────────────────────────────────┘
```

**HIR (High-level Intermediate Representation):**

Compiler AST'ni HIR'ga o'tkazadi — HIR statement-level operatsiyalarni instruction-level instruction'larga bo'ladi:

```javascript
// Source
const total = items.reduce((s, x) => s + x.price, 0);

// HIR (pseudocode)
%t1 = LoadLocal items
%t2 = PropertyLoad %t1, "reduce"
%t3 = ArrowFunction (s, x) => s + x.price
%t4 = LoadConst 0
%t5 = MethodCall %t1, %t2, [%t3, %t4]
StoreLocal "total", %t5
```

**Reactive Scope:**

Compiler "reactive scope" deb ataladigan **cache boundaries**'ni aniqlaydi. Har scope — bir reactive computation va uning depencency'lari:

```javascript
// Scope 1: total depends on items
{
  const total = items.reduce((s, x) => s + x.price, 0);
}

// Scope 2: greeting depends on user.name
{
  const greeting = `Hello, ${user.name}!`;
}

// Scope 3: handleClick depends on onSubmit, items
{
  const handleClick = () => onSubmit(items);
}
```

Har scope alohida cache slot'da saqlanadi — Compiler oldindan slot indekslarini hisoblab, build-time'da generate qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Manual memoization (Compiler'siz):

```tsx
// Manual approach
import { useMemo, useCallback, memo } from 'react';

interface Item {
  id: string;
  price: number;
  name: string;
}

interface ProductListProps {
  items: Item[];
  taxRate: number;
  onItemClick: (id: string) => void;
}

export const ProductList = memo(function ProductList({
  items,
  taxRate,
  onItemClick,
}: ProductListProps) {
  const total = useMemo(
    () => items.reduce((sum, item) => sum + item.price, 0),
    [items]
  );
  
  const totalWithTax = useMemo(
    () => total * (1 + taxRate),
    [total, taxRate]
  );
  
  const handleClick = useCallback(
    (id: string) => {
      onItemClick(id);
    },
    [onItemClick]
  );
  
  return (
    <div>
      <h2>Total: {totalWithTax.toFixed(2)}</h2>
      <ul>
        {items.map((item) => (
          <li key={item.id} onClick={() => handleClick(item.id)}>
            {item.name}: {item.price}
          </li>
        ))}
      </ul>
    </div>
  );
});
```

Compiler bilan (manual memoization yo'q):

```tsx
// Compiler approach — yozish jarayoni
import type { ReactElement } from 'react';

interface Item {
  id: string;
  price: number;
  name: string;
}

interface ProductListProps {
  items: Item[];
  taxRate: number;
  onItemClick: (id: string) => void;
}

export function ProductList({
  items,
  taxRate,
  onItemClick,
}: ProductListProps): ReactElement {
  const total = items.reduce((sum, item) => sum + item.price, 0);
  const totalWithTax = total * (1 + taxRate);
  
  const handleClick = (id: string) => {
    onItemClick(id);
  };
  
  return (
    <div>
      <h2>Total: {totalWithTax.toFixed(2)}</h2>
      <ul>
        {items.map((item) => (
          <li key={item.id} onClick={() => handleClick(item.id)}>
            {item.name}: {item.price}
          </li>
        ))}
      </ul>
    </div>
  );
}
// Compiler avtomatik:
// - total ni cache qiladi (depends on items)
// - totalWithTax ni cache qiladi (depends on total, taxRate)
// - handleClick ni cache qiladi (depends on onItemClick)
// - JSX strukturasini cache qiladi (depends on items, totalWithTax)
// - Komponent'ning o'zini cache qiladi (memo bilan teng) — props o'zgarmasa render skipped
```

Hajm farqi: manual versiya 30+ qator memoization wrapper, Compiler versiya — kerakli logic only.

</details>

---

## Auto-Memoization Mexanizmi

### Nazariya

**Auto-memoization** — Compiler'ning asosiy vazifasi: dasturchi `useMemo`/`useCallback`/`React.memo` yozmaydi, Compiler kompilyatsiya paytida har reactive computation va JSX strukturasiga cache logic'ini o'rnatadi.

QANDAY ishlatiladi (qadam-qadam):

1. **Reactivity inference** — Compiler komponent input'larini topadi: props, state (`useState`/`useReducer`), context (`useContext`), custom hook return values, va `ref.current` (mutable, faqat o'qishda).
2. **Dataflow tracking** — har local variable/expression qaysi reactive value'ga bog'liqligi aniqlanadi (transitive closure).
3. **Reactive scope grouping** — bog'liq computation'lar bir scope'ga birlashadi (granular caching).
4. **Cache allocation** — har scope uchun unikal cache slot ajratiladi (`_c[0]`, `_c[1]`, ...).
5. **Code generation** — kompilyatsiya paytida har scope `if (deps changed) { recompute; cache; } else { use cached; }` patternga aylanadi.

Misol — manual vs auto memoization:

```tsx
// Manual
const total = useMemo(
  () => items.reduce((s, x) => s + x.price, 0),
  [items]
);

// Compiler output (pseudocode)
let total;
if (cache[0] !== items) {
  total = items.reduce((s, x) => s + x.price, 0);
  cache[0] = items;
  cache[1] = total;
} else {
  total = cache[1];
}
```

Granularity — har **reactive scope** alohida cache:

```tsx
// Source
function Dashboard({ user, items, settings }: Props) {
  const greeting = `Hello, ${user.name}!`;          // Scope A: user.name
  const total = items.reduce(...);                   // Scope B: items
  const theme = settings.dark ? 'dark' : 'light';    // Scope C: settings
  
  return <div>{greeting} ({total}) [{theme}]</div>;
}
```

Compiler 3 ta alohida scope identify qiladi:

- `user.name` o'zgarsa — faqat `greeting` recompute (Scope A)
- `items` o'zgarsa — faqat `total` recompute (Scope B)
- `settings` o'zgarsa — faqat `theme` recompute (Scope C)

Manual `useMemo` bilan bunday granularity'ga erishish — har computation uchun alohida `useMemo` chaqiriq, dependency array'lar — qo'lda yozilgan kod hajmi katta.

JSX caching:

```tsx
// Source
return (
  <Card>
    <Header title={user.name} />
    <Body content={items} />
  </Card>
);

// Compiler output (pseudocode)
let _Card;
let _Header;
let _Body;

if (cache[5] !== user.name) {
  cache[5] = user.name;
  _Header = <Header title={user.name} />;
  cache[6] = _Header;
} else {
  _Header = cache[6];
}

if (cache[7] !== items) {
  cache[7] = items;
  _Body = <Body content={items} />;
  cache[8] = _Body;
} else {
  _Body = cache[8];
}

if (cache[9] !== _Header || cache[10] !== _Body) {
  cache[9] = _Header;
  cache[10] = _Body;
  _Card = <Card>{_Header}{_Body}</Card>;
  cache[11] = _Card;
} else {
  _Card = cache[11];
}

return _Card;
```

Bu output — JSX strukturasi cache, parent re-render lekin children o'zgarmasa — JSX reference saqlanadi, Reconciler bailout qiladi (cross-ref `04-reconciliation.md`).

<details>
<summary><strong>Under the Hood</strong></summary>

Compiler'ning **memo cache hook** mexanizmi:

```javascript
// react/compiler-runtime
import { c as useMemoCache } from 'react/compiler-runtime';

function Component(props) {
  const $ = useMemoCache(N); // N = cache slots count
  
  // Compiler-generated code
  let t0;
  if ($[0] !== props.items) {
    t0 = props.items.reduce((s, x) => s + x.price, 0);
    $[0] = props.items;
    $[1] = t0;
  } else {
    t0 = $[1];
  }
  // ...
}
```

`useMemoCache` — Compiler-generated kod uchun maxsus internal hook. `useMemo` kabi Hook chain'da bitta slot ajratadi, lekin bitta cache array (N slot) qaytaradi (`useMemo` esa bitta value qaytaradi):

```javascript
// React internal (oddiylashtirilgan)
function useMemoCache(size) {
  const dispatcher = ReactCurrentDispatcher.current;
  return dispatcher.useMemoCache(size);
}

// mountMemoCache
function mountMemoCache(size) {
  const cache = new Array(size).fill(REACT_MEMO_CACHE_SENTINEL);
  // Sentinel value — har slot uchun "uninitialized" marker
  
  const hook = mountWorkInProgressHook();
  hook.memoizedState = cache;
  return cache;
}

// updateMemoCache
function updateMemoCache(size) {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState; // Saqlangan cache array
}
```

Cache lifecycle:

1. **Mount:** `useMemoCache(N)` chaqiriladi, `N` ta slot ajratiladi, hammasi `MEMO_CACHE_SENTINEL` (sentinel value).
2. **First render:** har scope uchun `if ($[i] === SENTINEL || dep mismatch) → compute, store; else → use cached`.
3. **Update:** `updateMemoCache` saqlangan cache'ni qaytaradi, har scope cache hit/miss check.
4. **Unmount:** Hook chain bilan birga GC.

Sentinel value — `Symbol.for('react.memo_cache_sentinel')` (o'xshash). Birinchi render'da `$[i] !== sentinel` — har doim true, scope compute. Keyingi render'larda — saqlangan dep value bilan solishtirish.

**Comparison strategy:**

Compiler default `Object.is` ishlatadi (cross-ref `04-reconciliation.md`):

```javascript
function $arePropsEqual($, deps) {
  for (let i = 0; i < deps.length; i++) {
    if (!Object.is($[i], deps[i])) return false;
  }
  return true;
}
```

Custom equality — Compiler hozircha qo'llab-quvvatlamaydi (manual `useMemo` bilan custom comparator hali kerak bo'lishi mumkin).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Granular caching demo:

```tsx
// Source
import type { ReactElement } from 'react';

interface UserProfile {
  id: string;
  name: string;
  email: string;
}

interface Settings {
  theme: 'light' | 'dark';
  language: string;
}

interface DashboardProps {
  user: UserProfile;
  settings: Settings;
  totalRevenue: number;
}

function Dashboard({ user, settings, totalRevenue }: DashboardProps): ReactElement {
  // Scope 1: greeting depends on user.name (just user.name, not whole user)
  const greeting = `Welcome back, ${user.name}!`;
  
  // Scope 2: themeClass depends on settings.theme
  const themeClass = `theme-${settings.theme}`;
  
  // Scope 3: formattedRevenue depends on totalRevenue, settings.language
  const formattedRevenue = new Intl.NumberFormat(settings.language, {
    style: 'currency',
    currency: 'USD',
  }).format(totalRevenue);
  
  return (
    <div className={themeClass}>
      <h1>{greeting}</h1>
      <p>Total revenue: {formattedRevenue}</p>
    </div>
  );
}

// Compiler 3 ta alohida scope yaratadi.
// user.email o'zgarsa — Scope 1 cached (faqat user.name kuzatiladi)
// totalRevenue o'zgarsa — Scope 3 recompute, lekin Scope 1 va 2 cached
// settings.theme o'zgarsa — Scope 2 va 3 recompute, Scope 1 cached
```

JSX struktura caching:

```tsx
import type { ReactElement } from 'react';

interface User {
  id: string;
  name: string;
}

interface Comment {
  id: string;
  text: string;
  authorId: string;
}

interface PostProps {
  user: User;
  comments: Comment[];
}

function Post({ user, comments }: PostProps): ReactElement {
  return (
    <article>
      <header>
        <h2>By {user.name}</h2> {/* Cached: depends on user.name */}
      </header>
      <ul>
        {comments.map((c) => ( {/* Cached map output: depends on comments */}
          <li key={c.id}>{c.text}</li>
        ))}
      </ul>
    </article>
  );
}

// User o'zgarmagan, comments yangilangan:
// - <h2> JSX cached (saqlangan reference)
// - <ul> recomputed (yangi comments)
// - Reconciler <h2>'ni bailout qiladi (same reference)
```

Function reference stability:

```tsx
import type { ReactElement } from 'react';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

interface TodoListProps {
  todos: Todo[];
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
}

function TodoList({ todos, onToggle, onDelete }: TodoListProps): ReactElement {
  // Compiler cache:
  // - handleToggle scope (depends on onToggle)
  // - handleDelete scope (depends on onDelete)
  
  const handleToggle = (id: string) => {
    onToggle(id);
  };
  
  const handleDelete = (id: string) => {
    onDelete(id);
  };
  
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.completed}
            onChange={() => handleToggle(todo.id)}
          />
          <span>{todo.text}</span>
          <button onClick={() => handleDelete(todo.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}

// onToggle, onDelete prop'lar parent'da har render yangi reference bo'lsa —
// Compiler cache miss → handleToggle/handleDelete recompute.
// Parent'da onToggle/onDelete useCallback yoki Compiler tomonidan stabilize qilingan bo'lsa —
// children TodoList ham o'sha cache benefit'ini oladi.
```

</details>

---

## Internal Cache Mechanism — `_c` Array

### Nazariya

Compiler'ning generated kodida har komponent boshida **`useMemoCache(N)`** chaqiriladi va **`_c` deb belgilangan array** qaytariladi (`_c` Compiler tomonidan tanlangan o'zgaruvchi nomi, "cache" qisqa shakli). `N` — komponent ichidagi cache slot'lar soni, build-time'da hisoblangan.

```tsx
// Source
function Greeting({ name }: { name: string }) {
  const greeting = `Hello, ${name}!`;
  return <h1>{greeting}</h1>;
}

// Compiler output (oddiylashtirilgan)
import { c as _c } from 'react/compiler-runtime';

function Greeting(t0) {
  const $ = _c(3); // 3 ta slot
  const { name } = t0;
  
  let t1;
  if ($[0] !== name) {
    t1 = `Hello, ${name}!`;
    $[0] = name;
    $[1] = t1;
  } else {
    t1 = $[1];
  }
  const greeting = t1;
  
  let t2;
  if ($[2] !== greeting) {
    t2 = <h1>{greeting}</h1>;
    $[2] = greeting; // Aslida boshqa slot, oddiylashtirilgan
    // Real Compiler: t2 ni alohida slot'da, deps'ni boshqa slot'da
  } else {
    t2 = $[3];
  }
  return t2;
}
```

Cache slot turlari:

1. **Dependency slots** — input value reference saqlash (props.name, items, ...).
2. **Output slots** — kompilyatsiya qilingan computation natija (greeting string, JSX element).
3. **Sentinel slots** — uninitialized marker (birinchi render uchun).

NIMA UCHUN array (object emas):

- **Performance** — array index access O(1), object property lookup property descriptor traversal.
- **Memory locality** — array contiguous heap allocation (V8 SMI optimization).
- **Build-time deterministik** — slot'lar build-time'da indekslanadi, runtime'da yangi slot qo'shilmaydi.

Slot count — Compiler statik analiz natijasi. Misol uchun 3 ta reactive scope bo'lsa va har biri 2 ta slot (deps + output) ishlatsa: `_c(6)`.

> **Eslatma:** Generated kod nomi va shakli Compiler versiyasiga qarab o'zgarishi mumkin. Bu yerda concept ko'rsatish uchun oddiylashtirilgan format ishlatilgan. Ishlab chiqaruvchi hujjatlari va `babel-plugin-react-compiler` dist'iga qarash tavsiya etiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`useMemoCache` Hook chain integration:

```javascript
// React internal
const HooksDispatcherOnMount = {
  // ...
  useMemoCache(size) {
    const cache = new Array(size).fill(REACT_MEMO_CACHE_SENTINEL);
    const hook = mountWorkInProgressHook();
    hook.memoizedState = cache;
    return cache;
  },
  // ...
};

const HooksDispatcherOnUpdate = {
  // ...
  useMemoCache(size) {
    const hook = updateWorkInProgressHook();
    let cache = hook.memoizedState;
    // Concurrent: workInProgress.alternate'dan clone qilish
    if (cache.length !== size) {
      // Cache size o'zgarsa (HMR/Fast Refresh) — yangi cache
      cache = new Array(size).fill(REACT_MEMO_CACHE_SENTINEL);
      hook.memoizedState = cache;
    }
    return cache;
  },
  // ...
};
```

Cache lifecycle ASCII:

```
First render (mount):
   useMemoCache(5) → [sentinel, sentinel, sentinel, sentinel, sentinel]
                       ↑                                              ↑
                    deps[0]                                       output[4]

Compiler-generated checks:
   if ($[0] !== input) { compute; $[0] = input; $[1] = result; }
   if ($[2] !== input2) { compute2; $[2] = input2; $[3] = result2; }
   ...

After 1st render: [val0, result0, val2, result2, JSX_root]


Second render:
   useMemoCache(5) → returns SAME cache (saqlangan)
   
Comparison:
   $[0] === input ? → cache hit, use $[1]
   $[2] === input2 ? → cache miss (input2 changed) → recompute, update $[2], $[3]
   ...
```

`Object.is` comparison — har cache hit/miss check'da:

```javascript
// Generated by Compiler
let t1;
if ($[0] !== name) { // Object.is(prev, next) ekvivalenti
  t1 = `Hello, ${name}!`;
  $[0] = name;
  $[1] = t1;
} else {
  t1 = $[1];
}
```

`!==` operator JavaScript'da `===` inverse, va `===` strict equality `Object.is` bilan **deyarli teng** (NaN va +0/-0 farqi). Compiler default `!==` ishlatadi — primitive uchun yetarli, object uchun reference comparison.

Hook chain'da `useMemoCache` slot:

```
Fiber.memoizedState (Hook linked list):
[useState] → [useEffect] → [useMemoCache] → [useState] → ...
                                ↓
                          memoizedState = [_, _, _, _, _]
                                          (cache array)
```

`useMemoCache` 1 ta hook slot ishlatadi (boshqa hook'lar kabi). Cache array — uning `memoizedState` ichidagi N elementli array.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Compiler output o'qish (qo'lda) — sodda misol:

```tsx
// Source
import type { ReactElement } from 'react';

function Counter({ initial }: { initial: number }): ReactElement {
  const doubled = initial * 2;
  return <div>{doubled}</div>;
}
```

Compiler output (pseudocode, oddiylashtirilgan):

```javascript
import { c as _c } from 'react/compiler-runtime';

function Counter(t0) {
  const $ = _c(4);
  const { initial } = t0;
  
  // Scope 1: doubled
  let t1;
  if ($[0] !== initial) {
    t1 = initial * 2;
    $[0] = initial;
    $[1] = t1;
  } else {
    t1 = $[1];
  }
  const doubled = t1;
  
  // Scope 2: JSX
  let t2;
  if ($[2] !== doubled) {
    t2 = <div>{doubled}</div>;
    $[2] = doubled;
    $[3] = t2;
  } else {
    t2 = $[3];
  }
  
  return t2;
}
```

Composite props:

```tsx
// Source
import type { ReactElement } from 'react';

interface User {
  name: string;
  age: number;
}

function UserBadge({ user }: { user: User }): ReactElement {
  const display = `${user.name} (${user.age})`;
  return <span>{display}</span>;
}
```

Compiler output:

```javascript
import { c as _c } from 'react/compiler-runtime';

function UserBadge(t0) {
  const $ = _c(5);
  const { user } = t0;
  
  // Scope 1: display
  // Compiler track qiladi user.name va user.age (depth-2 dependency)
  let t1;
  if ($[0] !== user.name || $[1] !== user.age) {
    t1 = `${user.name} (${user.age})`;
    $[0] = user.name;
    $[1] = user.age;
    $[2] = t1;
  } else {
    t1 = $[2];
  }
  const display = t1;
  
  // Scope 2: JSX
  let t2;
  if ($[3] !== display) {
    t2 = <span>{display}</span>;
    $[3] = display;
    $[4] = t2;
  } else {
    t2 = $[4];
  }
  
  return t2;
}

// User reference o'zgarsa lekin user.name va user.age bir xil bo'lsa — cache hit.
// Manual useMemo([user]) bilan — yangi user reference cache miss bo'lardi.
// Compiler property-level granularity — ko'p holatda yaxshiroq cache hit ratio.
```

Multi-input scope:

```tsx
import type { ReactElement } from 'react';

interface PriceProps {
  price: number;
  taxRate: number;
  currency: string;
}

function PriceTag({ price, taxRate, currency }: PriceProps): ReactElement {
  const formattedTotal = `${currency}${(price * (1 + taxRate)).toFixed(2)}`;
  return <span>{formattedTotal}</span>;
}
```

Compiler output:

```javascript
import { c as _c } from 'react/compiler-runtime';

function PriceTag(t0) {
  const $ = _c(6);
  const { price, taxRate, currency } = t0;
  
  let t1;
  if ($[0] !== price || $[1] !== taxRate || $[2] !== currency) {
    t1 = `${currency}${(price * (1 + taxRate)).toFixed(2)}`;
    $[0] = price;
    $[1] = taxRate;
    $[2] = currency;
    $[3] = t1;
  } else {
    t1 = $[3];
  }
  const formattedTotal = t1;
  
  let t2;
  if ($[4] !== formattedTotal) {
    t2 = <span>{formattedTotal}</span>;
    $[4] = formattedTotal;
    $[5] = t2;
  } else {
    t2 = $[5];
  }
  
  return t2;
}
```

Compiler output o'qish — debug paytida foydali (`babel-plugin-react-compiler` AST output yoki source map orqali ko'rish mumkin).

</details>

---

## Static Analysis: HIR va Dataflow

### Nazariya

Compiler **statik analiz** qiladi — runtime'siz, source kod'dan ma'lumot olib keladi. Bu jarayon ikki bosqichdan iborat: **HIR (High-level Intermediate Representation)** qurish va **dataflow analysis** o'tkazish.

### HIR — High-level IR

Babel AST source kod tuzilishini ifodalaydi (statement, expression, declaration). HIR — pastroq darajadagi **instruction-level** representation. Har source line bir nechta HIR instruction'ga aylanadi.

```javascript
// Source
const total = items.reduce((s, x) => s + x.price, 0);

// HIR (oddiylashtirilgan)
%t1 = LoadLocal "items"
%t2 = PropertyLoad %t1, "reduce"
%t3 = ArrowFunction (s, x) -> {
  %a1 = LoadParam "s"
  %a2 = LoadParam "x"
  %a3 = PropertyLoad %a2, "price"
  %a4 = BinaryOp Add, %a1, %a3
  Return %a4
}
%t4 = LoadConst 0
%t5 = MethodCall %t1, %t2, [%t3, %t4]
StoreLocal "total", %t5
```

NIMA UCHUN HIR foydali:

1. **Granularity** — har property access alohida instruction (user.name vs user.age — alohida tracking).
2. **SSA form (Static Single Assignment)** — har variable bir marta assign, dataflow oson.
3. **Control Flow Graph (CFG)** — if/else/loop/try strukturasi mantiqiy bloklar bilan.
4. **Mutation tracking** — qaysi instruction qaysi value'ni o'zgartirgan.

### Dataflow Analysis

Dataflow — **value qaerdan keladi va qayerga ketadi** kuzatish:

1. **Reachability** — qaysi value reactive scope ichida ishlatiladi.
2. **Aliasing** — bir-biriga reference saqlanadigan value'lar (mutation propagation).
3. **Mutability** — qaysi value'lar mutate qilingan (Compiler ushbu komponentni "memoize qilolmaydi" bail-out).
4. **Dependency closure** — value `A` value `B`'ga bog'liq, `B` `C`'ga — `A` transitively `C`'ga bog'liq.

Misol — dependency chain:

```typescript
const a = props.x;
const b = a + 1;
const c = b * 2;
return <div>{c}</div>;
```

Dataflow:

```
props.x → a → b → c → JSX
```

Compiler scope inference:

- `a` scope: deps = [props.x]
- `b` scope: deps = [a]
- `c` scope: deps = [b]
- JSX scope: deps = [c]

Yoki Compiler **inline qiladi** kichik computation'larni bir scope'ga (cost-benefit analysis):

- Single scope: deps = [props.x]
- Single output: JSX

Inline'ning qarori — Compiler heuristic'iga bog'liq (qiymat hajmi, side effect xavfi, granularity benefit).

### Mutability Inference

Compiler mutation'ni track qiladi:

```typescript
// Compiler analiz: items mutation qilinmagan, immutable
const items = [1, 2, 3];
const total = items.reduce((s, x) => s + x, 0);
return <div>{total}</div>;

// Compiler analiz: items mutation qilingan ❌
const items = [1, 2, 3];
items.push(4); // Mutation
return <div>{items.length}</div>;
// Compiler: bail-out — bu komponentda memoization qilolmaydi
```

Mutation aniqlanganda Compiler **ikki yo'l** tanlaydi:

1. **Local mutation** — value komponent ichida yaratilgan va faqat ichida ishlatilgan. Compiler buni **local effect** deb biladi va memoize qila oladi (mutation'ni hisobga olib).
2. **Non-local mutation** — props/state/global mutation. Compiler bu komponentni **bail-out** qiladi (memoization yo'q).

Misol:

```typescript
// Local mutation OK
function build(): Item[] {
  const result: Item[] = [];
  for (let i = 0; i < 10; i++) {
    result.push({ id: String(i), name: `Item ${i}` });
  }
  return result;
}

// Non-local mutation — bail-out
function badPush(items: Item[]) {
  items.push({ id: 'new', name: 'New' }); // ❌ Props mutation
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

HIR kompilyatsiya bosqichlari:

```
AST (from Babel parser)
   │
   ▼
1. HIR Construction
   - StatementsHIRBuilder visitor
   - SSA form conversion (every variable named once)
   - Control flow graph (CFG) edges
   │
   ▼
2. Type Inference
   - JavaScript type-flow analysis (number, string, object, ...)
   - React type detection (Element, Component, Hook return)
   - Mutability flags (frozen vs mutable)
   │
   ▼
3. Aliasing Analysis
   - Pointer/reference tracking
   - "Does variable A reference same heap object as B?"
   - Used for mutation propagation
   │
   ▼
4. Reactive Scope Inference
   - Group instructions by reactive dependency
   - Boundaries = cache slot
   - Heuristic: balance granularity vs overhead
   │
   ▼
5. Code Generation
   - HIR → AST (back to JavaScript)
   - Insert _c(N), if checks, $[i] writes
```

**SSA form misol:**

```javascript
// Source
let x = 1;
x = x + 1;
x = x * 2;

// SSA form
let x_1 = 1;
let x_2 = x_1 + 1;
let x_3 = x_2 * 2;
// Har "version" alohida, immutable
```

SSA — Compiler dataflow analysis'ga oson yondashuv beradi: har value bir marta assigned, dependency strict.

**Control flow graph:**

```
if (cond) {
  a = 1;
} else {
  a = 2;
}
b = a;
```

CFG:

```
[Entry]
   │
   ▼
[cond?]
  ├─ true  → [a = 1] ──┐
  └─ false → [a = 2] ──┤
                       ▼
              [a = phi(a_1, a_2)]
                       │
                       ▼
                  [b = a]
```

`phi` — SSA join point, ikki branch'dan kelgan value'ni birlashtiradi.

**Reactive scope heuristic:**

Compiler scope'larni qanday belgilaydi (oddiylashtirilgan):

1. Bog'liq instructions birga (transitive closure).
2. Scope hajmi balansli — ko'p instruction → 1 scope ortiqcha cache invalidation, kam instruction → granularity yo'q.
3. JSX boundary — har JSX element alohida scope (Reconciler bailout uchun).
4. Hook chaqiriqlar oraliq — scope chegarasi (hook side effect bo'lishi mumkin).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Dataflow tracing misol:

```tsx
import type { ReactElement } from 'react';

interface Order {
  id: string;
  items: { id: string; price: number }[];
  taxRate: number;
}

function OrderSummary({ order }: { order: Order }): ReactElement {
  // Dataflow: order → items, taxRate
  const items = order.items;
  const taxRate = order.taxRate;
  
  // Dataflow: items → subtotal
  const subtotal = items.reduce((sum, item) => sum + item.price, 0);
  
  // Dataflow: subtotal, taxRate → total
  const total = subtotal * (1 + taxRate);
  
  // Dataflow: order.id, items.length, total → JSX
  return (
    <div>
      <h3>Order {order.id}</h3>
      <p>{items.length} items</p>
      <p>Total: ${total.toFixed(2)}</p>
    </div>
  );
}

// Compiler analiz natijasi:
// - subtotal scope: deps = [order.items]
// - total scope: deps = [subtotal, order.taxRate]
// - JSX scope: deps = [order.id, order.items.length, total]
```

Mutation aniqlash:

```tsx
import type { ReactElement } from 'react';

interface Item {
  id: string;
  price: number;
}

// ✅ Local mutation OK
function PriceList({ items }: { items: Item[] }): ReactElement {
  const sorted = [...items]; // Local copy
  sorted.sort((a, b) => a.price - b.price); // Local mutation
  
  return (
    <ul>
      {sorted.map((item) => (
        <li key={item.id}>${item.price}</li>
      ))}
    </ul>
  );
}
// Compiler: sorted ichki mutation, lekin output immutable.
// Memoize qila oladi.

// ❌ Props mutation — bail-out
function BadPriceList({ items }: { items: Item[] }): ReactElement {
  items.sort((a, b) => a.price - b.price); // ❌ Props mutation
  
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>${item.price}</li>
      ))}
    </ul>
  );
}
// Compiler: bail-out, bu komponentda manual yoki Compiler memoization yo'q.
// eslint-plugin-react-compiler error: "Mutating prop is forbidden"
```

</details>

---

## Rules of React — Compliance Talablar

### Nazariya

Compiler ishlashi uchun komponent va hook'lar **Rules of React**'ga rioya qilishi shart. Bu qoidalar React hujjatlarining `react.dev/reference/rules` bo'limida rasmiylashtirilgan va aslida React'ning **runtime invariants**'i (cross-ref `30-concurrent-react.md`). Compiler bu invariants'ni statik analizda majburlaydi.

### Qoida 1: Komponentlar va Hook'lar Pure bo'lishi shart

```typescript
// ❌ Render Phase'da side effect
function Counter({ count }: { count: number }) {
  console.log('Render'); // ❌ Side effect (Strict Mode 2x ko'rinadi)
  document.title = `Count: ${count}`; // ❌ DOM mutation
  return <div>{count}</div>;
}

// ✅ Pure
function Counter({ count }: { count: number }) {
  return <div>{count}</div>;
}
```

Side effect manbai:
- Console logging (development'dagi kuzatish)
- DOM mutation
- Network call
- localStorage/sessionStorage write
- Module-level variable write
- `Date.now()`, `Math.random()`, `crypto.randomUUID()`, `performance.now()` (mutable external state read)

### Qoida 2: Props, State, Context Read-only

```typescript
// ❌ Props mutation
function UserCard({ user }: { user: User }) {
  user.name = user.name.toUpperCase(); // ❌
  return <h1>{user.name}</h1>;
}

// ✅ Immutable derive
function UserCard({ user }: { user: User }) {
  const displayName = user.name.toUpperCase();
  return <h1>{displayName}</h1>;
}
```

State mutation — eng tez-tez uchraydigan bug:

```typescript
// ❌ State mutation
const [items, setItems] = useState<Item[]>([]);

function handleAdd(newItem: Item) {
  items.push(newItem); // ❌ Mutation, Reconciler bailout buziladi
  setItems(items); // Same reference → eager bailout
}

// ✅ Immutable update
function handleAdd(newItem: Item) {
  setItems((prev) => [...prev, newItem]);
}
```

### Qoida 3: Hooks Top-Level

Hooks top-level (yoki R19 `use()` istisno — cross-ref `23-r19-hooks.md`):

```typescript
// ❌ Conditional hook
function MyComponent({ flag }: { flag: boolean }) {
  if (flag) {
    const [count] = useState(0); // ❌ Conditional
  }
}

// ✅ Top-level
function MyComponent({ flag }: { flag: boolean }) {
  const [count] = useState(0);
  if (flag) {
    // Use count...
  }
}
```

### Qoida 4: Refs Render Phase'da Yozilmaydi

```typescript
// ❌ Render Phase'da ref write
function Form() {
  const inputRef = useRef<HTMLInputElement>(null);
  inputRef.current = document.querySelector('input'); // ❌ Side effect
  return <input ref={inputRef} />;
}

// ✅ Effect'da
function Form() {
  const inputRef = useRef<HTMLInputElement>(null);
  useEffect(() => {
    // Read inputRef.current here — OK
  });
  return <input ref={inputRef} />;
}
```

Lekin **render Phase'da `ref.current` o'qish** ham anti-pattern:

```typescript
// ❌ Render Phase'da ref read
function Form() {
  const inputRef = useRef<HTMLInputElement>(null);
  const value = inputRef.current?.value; // ❌ External mutable state
  return <div>{value}</div>;
}

// ✅ State'da yoki effect'da
function Form() {
  const inputRef = useRef<HTMLInputElement>(null);
  const [value, setValue] = useState('');
  return <input ref={inputRef} value={value} onChange={(e) => setValue(e.target.value)} />;
}
```

### Qoida 5: Effect Setup va Cleanup Symmetry

Cross-ref `30-concurrent-react.md` Invariant 3.

```typescript
// ❌ Cleanup yo'q
useEffect(() => {
  window.addEventListener('resize', handler);
}, []);

// ✅ Cleanup
useEffect(() => {
  window.addEventListener('resize', handler);
  return () => window.removeEventListener('resize', handler);
}, []);
```

QANDAY Compiler bu qoidalarni majburlaydi:

1. **Statik analiz** — mutation/side effect detection HIR + dataflow.
2. **`eslint-plugin-react-compiler`** — kompilyatsiya'dan oldin warning/error.
3. **Bail-out** — qoidalar buzilsa Compiler shu komponentni memoize qilmaydi (fallback to manual or no memoization).

<details>
<summary><strong>Under the Hood</strong></summary>

Compiler bail-out reasons:

```
Mutation Detected:
  - Props mutation
  - State mutation (without setState)
  - Module-level variable write
  - Ref write during render
  → Bail-out: skip auto-memoization for this component

Side Effect Detected:
  - DOM mutation in render
  - Network call in render
  - console.log (warning, not bail-out)
  - Math.random / Date.now in render (external mutable read)
  → Bail-out

Pattern Confusion:
  - Conditional hook
  - Hook in non-React function
  - Hook in callback (not render or hook body)
  → Bail-out + Eslint error

Complex Closures:
  - Hook returning function captures mutable state
  - Async callback in render
  → Bail-out + warning
```

Bail-out semantics:

```javascript
// Compiler decision tree (oddiylashtirilgan)
function shouldMemoize(component) {
  if (hasMutation(component)) return false;
  if (hasSideEffect(component)) return false;
  if (hasRulesOfHooksViolation(component)) return false;
  if (hasUnsupportedPattern(component)) return false;
  return true;
}

// Bail-out output: kompilyatsiya qilingan kod manual implementation bilan ekvivalent
// (memoization yo'q, oddiy function call)
```

ESLint plugin detection:

```javascript
// eslint-plugin-react-compiler core rule
const rule = {
  create(context) {
    return {
      FunctionDeclaration(node) {
        if (isReactComponent(node)) {
          analyzeComponent(node, context);
        }
      },
      ArrowFunctionExpression(node) {
        if (isReactComponent(node)) {
          analyzeComponent(node, context);
        }
      },
    };
  },
};

function analyzeComponent(node, context) {
  // 1. Mutation detection (HIR-like analysis)
  // 2. Side effect detection
  // 3. Hook usage rules
  // 4. Report violations
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Common Rules violations va fix'lar:

```tsx
// 1. ❌ Mutation in render
import type { ReactElement } from 'react';

interface Cart {
  items: { id: string; quantity: number }[];
}

function CartTotal({ cart }: { cart: Cart }): ReactElement {
  cart.items.forEach((item) => {
    item.quantity = Math.max(0, item.quantity); // ❌ Props mutation
  });
  
  const total = cart.items.reduce((s, x) => s + x.quantity, 0);
  return <div>{total}</div>;
}

// ✅ Immutable transform
function CartTotal({ cart }: { cart: Cart }): ReactElement {
  const validItems = cart.items.map((item) => ({
    ...item,
    quantity: Math.max(0, item.quantity),
  }));
  
  const total = validItems.reduce((s, x) => s + x.quantity, 0);
  return <div>{total}</div>;
}
```

```tsx
// 2. ❌ Side effect in render
import { useState } from 'react';
import type { ReactElement } from 'react';

function PageLogger({ page }: { page: string }): ReactElement {
  fetch('/api/log', {
    method: 'POST',
    body: JSON.stringify({ page }),
  }); // ❌ Network call
  
  return <div>Page: {page}</div>;
}

// ✅ Event handler yoki useEffect
import { useEffect } from 'react';

function PageLogger({ page }: { page: string }): ReactElement {
  useEffect(() => {
    fetch('/api/log', {
      method: 'POST',
      body: JSON.stringify({ page }),
    });
    // Eslatma: Strict Mode 2x cycle 2x POST yuboradi.
    // Yaxshiroq — router navigation event'ida.
  }, [page]);
  
  return <div>Page: {page}</div>;
}
```

```tsx
// 3. ❌ Render'da Date.now / Math.random
import type { ReactElement } from 'react';

function GreetingId(): ReactElement {
  const id = Math.random().toString(36); // ❌ Mutable external read
  return <div id={id}>Hello</div>;
}

// ✅ useId (R18+, cross-ref 22)
import { useId } from 'react';

function GreetingId(): ReactElement {
  const id = useId();
  return <div id={id}>Hello</div>;
}
```

```tsx
// 4. ❌ State mutation
import { useState } from 'react';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

function TodoList(): ReactElement {
  const [todos, setTodos] = useState<Todo[]>([]);
  
  function handleToggle(id: string) {
    const todo = todos.find((t) => t.id === id);
    if (todo) {
      todo.completed = !todo.completed; // ❌ Mutation
      setTodos(todos); // Same reference → bailout
    }
  }
  
  return (
    <ul>
      {todos.map((t) => (
        <li key={t.id} onClick={() => handleToggle(t.id)}>
          {t.text}
        </li>
      ))}
    </ul>
  );
}

// ✅ Immutable update
function TodoList(): ReactElement {
  const [todos, setTodos] = useState<Todo[]>([]);
  
  function handleToggle(id: string) {
    setTodos((prev) =>
      prev.map((t) =>
        t.id === id ? { ...t, completed: !t.completed } : t
      )
    );
  }
  
  return (
    <ul>
      {todos.map((t) => (
        <li key={t.id} onClick={() => handleToggle(t.id)}>
          {t.text}
        </li>
      ))}
    </ul>
  );
}
```

```tsx
// 5. ❌ Conditional hook
function FeatureFlag({ enabled }: { enabled: boolean }): ReactElement {
  if (enabled) {
    const [count] = useState(0); // ❌ Conditional
    return <div>{count}</div>;
  }
  return <div>Disabled</div>;
}

// ✅ Top-level + conditional render
function FeatureFlag({ enabled }: { enabled: boolean }): ReactElement {
  const [count] = useState(0);
  if (!enabled) return <div>Disabled</div>;
  return <div>{count}</div>;
}
```

</details>

---

## `eslint-plugin-react-compiler` — Static Check

### Nazariya

`eslint-plugin-react-compiler` — Compiler'ning **statik tekshirish** plagini. ESLint pipeline'iga qo'shiladi va **kompilyatsiya'dan oldin** Rules of React violations'ni topadi. Bu plagin Compiler enable qilishdan oldin **majburiy** — kodni mos qilish ishlash.

### Install

```bash
npm install -D eslint-plugin-react-compiler
```

### Konfiguratsiya

```javascript
// eslint.config.js (flat config — ESLint 9+)
import reactCompiler from 'eslint-plugin-react-compiler';

export default [
  {
    plugins: {
      'react-compiler': reactCompiler,
    },
    rules: {
      'react-compiler/react-compiler': 'error',
    },
  },
];
```

Yoki classic format:

```javascript
// .eslintrc.json
{
  "plugins": ["react-compiler"],
  "rules": {
    "react-compiler/react-compiler": "error"
  }
}
```

### Plugin Tomonidan Aniqlanadigan Violations

| Violation | Sabab | Tuzatish |
|-----------|-------|----------|
| Props mutation | `props.x = y` | Immutable copy |
| State mutation | `state.push(x)` o'rniga `setState(...)` | `setState(prev => [...prev, x])` |
| Render Phase'da side effect | `console.log`, `fetch`, `localStorage.set` | `useEffect` yoki event handler |
| Conditional hook | `if (flag) useState()` | Top-level + conditional render |
| Ref write in render | `ref.current = X` render Phase'da | `useEffect` ichida |
| Hook in non-component | Regular function ichida `useState` | Custom hook (`use*` prefix) |
| Mutable global write | `module.value = X` | State yoki effect |

### Workflow

```
┌─────────────┐
│ Code editor │
└──────┬──────┘
       │ Save file
       ▼
┌──────────────────┐
│ ESLint checks    │
│ - eslint-plugin- │
│   react-compiler │
└──────┬───────────┘
       │
       ├─ No violations → Compiler can memoize
       │
       └─ Violations → Fix or bail-out
                       │
                       ▼
              ┌──────────────────┐
              │ Manual fix       │
              │ Compiler skips   │
              │ this component   │
              └──────────────────┘
```

### Plugin Output Misol

```typescript
// Source
function Counter({ count }: { count: number }) {
  document.title = `Count: ${count}`;
  return <div>{count}</div>;
}
```

ESLint output:

```
react-compiler/react-compiler:
  Cannot mutate non-local value 'document.title' during render.
  Move this side effect to useEffect or an event handler.
  
  See: https://react.dev/reference/rules/components-and-hooks-must-be-pure
```

Plagin barcha violations'ni report qiladi va dasturchi har birini qo'lda tuzatadi.

<details>
<summary><strong>Under the Hood</strong></summary>

ESLint plugin internals:

```javascript
// eslint-plugin-react-compiler (oddiylashtirilgan)
const rule = {
  meta: {
    type: 'problem',
    docs: {
      description: 'Verify React Compiler rules of React compliance',
      category: 'Possible Errors',
      recommended: 'error',
    },
    schema: [
      {
        type: 'object',
        properties: {
          panicOnBailOut: { type: 'boolean' },
          // ...
        },
      },
    ],
  },
  create(context) {
    return {
      FunctionDeclaration: visit,
      FunctionExpression: visit,
      ArrowFunctionExpression: visit,
    };
    
    function visit(node) {
      if (!isReactComponentOrHook(node)) return;
      
      // Compiler core'ni ishlatib analiz qilish
      const result = analyzeWithCompilerCore(node, context);
      
      for (const diagnostic of result.diagnostics) {
        context.report({
          node: diagnostic.node,
          message: diagnostic.message,
          severity: diagnostic.severity,
        });
      }
    }
  },
};
```

Plagin aslida `babel-plugin-react-compiler` core analizini ishlatadi va **ESLint API'ga adaptatsiya** qiladi.

Detection misollari:

```javascript
// Mutation detection
function detectMutation(hir) {
  for (const block of hir.blocks) {
    for (const inst of block.instructions) {
      if (
        inst.kind === 'StoreProperty' &&
        isReactivePropAccess(inst.target)
      ) {
        return { kind: 'mutation', node: inst.node };
      }
    }
  }
  return null;
}

// Side effect detection
function detectSideEffect(hir) {
  for (const block of hir.blocks) {
    for (const inst of block.instructions) {
      if (isImpureCall(inst)) {
        return { kind: 'side_effect', node: inst.node };
      }
    }
  }
  return null;
}

function isImpureCall(inst) {
  if (inst.kind !== 'Call') return false;
  
  const callee = inst.callee;
  // document.*, window.*, localStorage.*, fetch, console.* etc.
  if (
    isGlobalAccess(callee, 'document') ||
    isGlobalAccess(callee, 'window') ||
    isGlobalAccess(callee, 'localStorage') ||
    isGlobalAccess(callee, 'fetch') ||
    isGlobalAccess(callee, 'console')
  ) {
    return true;
  }
  
  return false;
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

ESLint config (Vite + TypeScript loyihada):

```javascript
// eslint.config.js
import js from '@eslint/js';
import typescript from 'typescript-eslint';
import reactCompiler from 'eslint-plugin-react-compiler';
import reactHooks from 'eslint-plugin-react-hooks';

export default [
  js.configs.recommended,
  ...typescript.configs.recommended,
  {
    files: ['**/*.{ts,tsx}'],
    plugins: {
      'react-compiler': reactCompiler,
      'react-hooks': reactHooks,
    },
    rules: {
      'react-compiler/react-compiler': 'error',
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn',
    },
  },
];
```

`package.json` script:

```json
{
  "scripts": {
    "lint": "eslint src/",
    "lint:fix": "eslint src/ --fix"
  }
}
```

Pre-commit hook (Husky bilan):

```bash
# .husky/pre-commit
npm run lint
```

Real-world violations va fix'lar:

```tsx
// 1. Module-level mutation
let renderCount = 0;

function Page(): ReactElement {
  renderCount += 1; // ESLint error
  return <div>Renders: {renderCount}</div>;
}

// Fix: useRef + useEffect
import { useRef, useEffect } from 'react';
import type { ReactElement } from 'react';

function Page(): ReactElement {
  const renderCountRef = useRef(0);
  useEffect(() => {
    renderCountRef.current += 1;
  });
  return <div>Renders: {renderCountRef.current}</div>;
}
```

```tsx
// 2. Render Phase'da localStorage
function ThemeBadge(): ReactElement {
  const theme = localStorage.getItem('theme') ?? 'light'; // ESLint error
  return <span className={theme}>Theme: {theme}</span>;
}

// Fix: useSyncExternalStore (cross-ref 22, 30)
import { useSyncExternalStore } from 'react';

function useTheme(): string {
  return useSyncExternalStore(
    (cb) => {
      window.addEventListener('storage', cb);
      return () => window.removeEventListener('storage', cb);
    },
    () => localStorage.getItem('theme') ?? 'light',
    () => 'light'
  );
}

function ThemeBadge(): ReactElement {
  const theme = useTheme();
  return <span className={theme}>Theme: {theme}</span>;
}
```

```tsx
// 3. Custom hook return mutation
import { useState } from 'react';

function useCounter(): { count: number; increment: () => void } {
  const [count, setCount] = useState(0);
  
  return {
    count,
    increment: () => {
      setCount((c) => c + 1);
    },
  };
  // OK — har gal yangi object reference, lekin Compiler buni avtomatik
  // memoize qiladi.
}

// Anti-pattern: object inside ref
function useCounterBad(): { count: number; increment: () => void } {
  const [count, setCount] = useState(0);
  const ref = useRef({ count: 0, increment: () => {} });
  
  ref.current.count = count; // ❌ Render Phase'da ref write
  ref.current.increment = () => setCount((c) => c + 1); // ❌ Same
  
  return ref.current;
}
```

</details>

---

## `babel-plugin-react-compiler` — Setup va Konfiguratsiya

### Nazariya

`babel-plugin-react-compiler` — Compiler'ning **build-time transformer**. Babel transpilation pipeline'iga qo'shiladi va source kod'ni memoization bilan output JavaScript'ga aylantiradi.

### Install

```bash
npm install -D babel-plugin-react-compiler
```

### Vite Konfiguratsiya

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react({
      babel: {
        plugins: [
          [
            'babel-plugin-react-compiler',
            {
              // Plugin opsiyalari
              target: '19', // React 19
            },
          ],
        ],
      },
    }),
  ],
});
```

### Webpack Konfiguratsiya

```javascript
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              '@babel/preset-env',
              '@babel/preset-react',
              '@babel/preset-typescript',
            ],
            plugins: [
              ['babel-plugin-react-compiler', { target: '19' }],
            ],
          },
        },
      },
    ],
  },
};
```

### Next.js Konfiguratsiya

```javascript
// next.config.js
module.exports = {
  experimental: {
    reactCompiler: true,
  },
};
```

Yoki options bilan:

```javascript
module.exports = {
  experimental: {
    reactCompiler: {
      compilationMode: 'annotation', // Faqat 'use memo' direktivasi bor fayllarni compile
    },
  },
};
```

### Plugin Opsiyalari

| Option | Default | Tavsif |
|--------|---------|--------|
| `target` | `'19'` | React versiyasi (`'17'`, `'18'`, `'19'`) |
| `compilationMode` | `'infer'` | `'all'`, `'annotation'`, `'infer'` |
| `panicThreshold` | `'all_errors'` | Bail-out strategy: `'all_errors'`, `'critical_errors'`, `'none'` |
| `sources` | `auto` | Qaysi fayllar compile (regex) |
| `eslintSuppressionRules` | `[]` | ESLint suppression directives |

**`compilationMode: 'annotation'`** — faqat **`'use memo'`** directive bor fayllarni compile qilish:

```tsx
'use memo'; // R19 directive — bu fayl Compiler tomonidan optimize qilinadi

import type { ReactElement } from 'react';

export function MyComponent({ data }: { data: string }): ReactElement {
  // ...
}
```

**`compilationMode: 'all'`** — barcha fayllar compile (default emas — incremental adoption uchun safe).

**`compilationMode: 'infer'`** — Compiler avtomatik aniqlaydi qaysi fayllar compile qilish kerak (default).

### Verification

Compile qilingan kodni tekshirish:

```bash
# Build qilish
npm run build

# Source map orqali generated kod ko'rish
# Yoki devtools'da kompilyatsiya qilingan komponent ko'rish
```

Generated kodda `_c(N)` chaqiriqlari mavjud bo'lishi shart:

```javascript
// Generated output (build dist/)
import { c as _c } from 'react/compiler-runtime';

function Counter(t0) {
  const $ = _c(3);
  // ...
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Plugin ichki ishlash:

```
Babel Pipeline:
   AST → [babel-plugin-react-compiler] → Modified AST
                  │
                  ├─ 1. Plugin enters component visitor
                  │
                  ├─ 2. Detect React component / hook
                  │     - PascalCase function returning JSX
                  │     - Function starting with 'use'
                  │
                  ├─ 3. Run Compiler core analysis
                  │     - HIR construction
                  │     - Reactivity inference
                  │     - Code generation
                  │
                  ├─ 4. Replace function body in AST
                  │     - Insert _c() at beginning
                  │     - Wrap reactive scopes with cache checks
                  │
                  └─ 5. Generate output via Babel
                        - Source map preservation
                        - JSX preservation (handled by other plugins)
```

`compilationMode` decision tree:

```
File source code
   │
   ▼
'use memo' directive?
   │
   ├─ Yes → COMPILE (annotation mode allows)
   │
   └─ No → Check mode
              │
              ├─ 'all' → COMPILE
              │
              ├─ 'annotation' → SKIP
              │
              └─ 'infer' → Detect heuristically
                            │
                            ├─ React imports? → COMPILE
                            ├─ JSX usage? → COMPILE
                            └─ No React → SKIP
```

`panicThreshold` semantics:

- **`'all_errors'`** (default development) — har bail-out warning sifatida log qilinadi va Compiler komponent'ni skip qiladi.
- **`'critical_errors'`** — faqat kritik bail-out (data loss xavfi) report qilinadi, oddiy bail-out silently skip.
- **`'none'`** (production) — hech qanday warning yoki error, Compiler "best-effort" rejimida ishlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq Vite konfiguratsiya (TypeScript + React 19):

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

const ReactCompilerConfig = {
  target: '19' as const,
  compilationMode: 'infer' as const,
  panicThreshold: 'all_errors' as const,
};

export default defineConfig({
  plugins: [
    react({
      babel: {
        plugins: [['babel-plugin-react-compiler', ReactCompilerConfig]],
      },
    }),
  ],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

`package.json`:

```json
{
  "name": "my-app",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint src/"
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.0",
    "babel-plugin-react-compiler": "^1.0.0",
    "eslint": "^9.0.0",
    "eslint-plugin-react-compiler": "^1.0.0",
    "eslint-plugin-react-hooks": "^5.0.0",
    "typescript": "^5.5.0",
    "vite": "^5.4.0"
  }
}
```

Per-file opt-in (incremental adoption):

```tsx
// File 1: Compiler enabled
'use memo';

import type { ReactElement } from 'react';

export function NewComponent({ data }: { data: string }): ReactElement {
  return <div>{data}</div>;
}
```

```tsx
// File 2: Compiler disabled (legacy code)
'use no memo';

import type { ReactElement } from 'react';

export function LegacyComponent({ data }: { data: string }): ReactElement {
  // Mutation yoki Rules of React violations bor — Compiler skip qiladi
  return <div>{data}</div>;
}
```

`compilationMode: 'annotation'` bilan + selective adoption:

```typescript
// vite.config.ts
const ReactCompilerConfig = {
  target: '19',
  compilationMode: 'annotation', // Faqat 'use memo' fayllar
};
```

Bu strategy katta legacy loyihada migration uchun foydali — har fayl alohida tekshirilib enable qilinadi.

Build verification:

```bash
# Build
npm run build

# Generated output ko'rish (Vite default: dist/)
ls dist/assets/

# Bir compiled file'ni ochish va `_c` chaqiriqlarini izlash
grep "_c(" dist/assets/index-*.js | head
```

Output:

```javascript
function Component(t0) {
  const $ = _c(7);
  const { user, items } = t0;
  // ...
}
```

</details>

---

## Compiler Output Misollari

### Nazariya

Compiler input (source kod) va output (kompilyatsiya qilingan kod) orasidagi farqni tushunish — manual memoization bilan solishtirish va debug uchun zarur.

### Misol 1: Sodda Counter

```tsx
// Source
import type { ReactElement } from 'react';
import { useState } from 'react';

export function Counter(): ReactElement {
  const [count, setCount] = useState(0);
  const doubled = count * 2;
  
  return (
    <div>
      <p>Count: {count}, Doubled: {doubled}</p>
      <button onClick={() => setCount((c) => c + 1)}>Increment</button>
    </div>
  );
}
```

Compiler output (oddiylashtirilgan):

```javascript
import { c as _c } from 'react/compiler-runtime';
import { useState } from 'react';

export function Counter() {
  const $ = _c(7);
  const [count, setCount] = useState(0);
  
  // Scope 1: doubled
  let t0;
  if ($[0] !== count) {
    t0 = count * 2;
    $[0] = count;
    $[1] = t0;
  } else {
    t0 = $[1];
  }
  const doubled = t0;
  
  // Scope 2: onClick callback
  let t1;
  if ($[2] === Symbol.for('react.memo_cache_sentinel')) {
    t1 = () => setCount((c) => c + 1);
    $[2] = t1;
  } else {
    t1 = $[2];
  }
  // setCount stable reference (useState kafolat) — sentinel check bir marta
  
  // Scope 3: JSX
  let t2;
  if ($[3] !== count || $[4] !== doubled || $[5] !== t1) {
    t2 = (
      <div>
        <p>Count: {count}, Doubled: {doubled}</p>
        <button onClick={t1}>Increment</button>
      </div>
    );
    $[3] = count;
    $[4] = doubled;
    $[5] = t1;
    $[6] = t2;
  } else {
    t2 = $[6];
  }
  
  return t2;
}
```

Tahlil:

- `doubled` — count'ga bog'liq, count o'zgarmasa cached.
- onClick callback — setCount stable, callback bir marta yaratiladi.
- JSX — count va doubled o'zgarsa recompute, aks holda cached.

Manual memoization equivalent:

```tsx
// Manual approach (Compiler'siz teng natija)
import { useState, useMemo, useCallback } from 'react';

export function Counter(): ReactElement {
  const [count, setCount] = useState(0);
  
  const doubled = useMemo(() => count * 2, [count]);
  
  const handleIncrement = useCallback(
    () => setCount((c) => c + 1),
    []
  );
  
  return (
    <div>
      <p>Count: {count}, Doubled: {doubled}</p>
      <button onClick={handleIncrement}>Increment</button>
    </div>
  );
}
```

### Misol 2: Children prop bilan

```tsx
// Source
import type { ReactElement, ReactNode } from 'react';

export function Card({ title, children }: { title: string; children: ReactNode }): ReactElement {
  return (
    <article>
      <h2>{title}</h2>
      {children}
    </article>
  );
}
```

Compiler output:

```javascript
import { c as _c } from 'react/compiler-runtime';

export function Card(t0) {
  const $ = _c(5);
  const { title, children } = t0;
  
  // Scope 1: <h2>
  let t1;
  if ($[0] !== title) {
    t1 = <h2>{title}</h2>;
    $[0] = title;
    $[1] = t1;
  } else {
    t1 = $[1];
  }
  
  // Scope 2: <article>
  let t2;
  if ($[2] !== t1 || $[3] !== children) {
    t2 = (
      <article>
        {t1}
        {children}
      </article>
    );
    $[2] = t1;
    $[3] = children;
    $[4] = t2;
  } else {
    t2 = $[4];
  }
  
  return t2;
}
```

Tahlil:

- `<h2>` cached — title o'zgarmasa same reference.
- `<article>` cached — h2 va children o'zgarmasa same reference (parent re-render bypass).

### Misol 3: Bail-out (mutation)

```tsx
// Source — mutation bor
import type { ReactElement } from 'react';

interface Item {
  id: string;
  qty: number;
}

export function Cart({ items }: { items: Item[] }): ReactElement {
  items.forEach((item) => {
    item.qty = Math.max(0, item.qty); // ❌ Props mutation
  });
  
  return (
    <ul>
      {items.map((i) => (
        <li key={i.id}>{i.qty}</li>
      ))}
    </ul>
  );
}
```

Compiler output (bail-out):

```javascript
// Compiler bail-out: shu komponentda memoization yo'q
// Output deyarli source bilan teng (Babel JSX transform faqat)

export function Cart(t0) {
  const { items } = t0;
  
  items.forEach((item) => {
    item.qty = Math.max(0, item.qty);
  });
  
  return (
    <ul>
      {items.map((i) => (
        <li key={i.id}>{i.qty}</li>
      ))}
    </ul>
  );
}

// Console warning (development):
// "React Compiler skipped Cart due to mutation of props.items"
```

ESLint plugin esa bu Cart'da error report qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Compiler bail-out qaror tree:

```
Component analysis:
   │
   ▼
1. Validate: hooks rules
   │
   ├─ Conditional hook → BAIL-OUT
   │
   └─ OK → continue
   
2. Validate: mutation
   │
   ├─ Mutates non-local → BAIL-OUT
   │
   └─ OK → continue

3. Validate: side effects
   │
   ├─ Render Phase'da side effect → BAIL-OUT
   │
   └─ OK → continue

4. Generate optimized output
```

Bail-out output strukturasi:

```javascript
// Bail-out: minimal transformation (faqat JSX preservation)
function Component(props) {
  // Original body — memoization yo'q
}

// Console warning chiqaradi (development):
console.warn(
  '[React Compiler] Cannot compile component "Component": ',
  reason,
  '\n  See:', sourceLocation
);
```

`panicThreshold: 'all_errors'` da bail-out — error sifatida loglanadi. `panicThreshold: 'none'` da silent skip.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq misol — kompilyatsiya qilingan output ko'rish:

```tsx
// src/components/Greeting.tsx
import type { ReactElement } from 'react';

interface User {
  id: string;
  name: string;
}

export function Greeting({ user, count }: { user: User; count: number }): ReactElement {
  const message = `Hello, ${user.name}!`;
  const total = count + 1;
  
  return (
    <div>
      <h1>{message}</h1>
      <p>Visits: {total}</p>
    </div>
  );
}
```

Build qilish:

```bash
npm run build
```

`dist/` ichidagi compiled file'da:

```javascript
// dist/assets/index-*.js (oddiylashtirilgan)
import { c as _c } from 'react/compiler-runtime';

function Greeting(t0) {
  const $ = _c(7);
  const { user, count } = t0;
  
  let t1;
  if ($[0] !== user.name) {
    t1 = `Hello, ${user.name}!`;
    $[0] = user.name;
    $[1] = t1;
  } else {
    t1 = $[1];
  }
  const message = t1;
  
  let t2;
  if ($[2] !== count) {
    t2 = count + 1;
    $[2] = count;
    $[3] = t2;
  } else {
    t2 = $[3];
  }
  const total = t2;
  
  let t3;
  if ($[4] !== message || $[5] !== total) {
    t3 = (
      <div>
        <h1>{message}</h1>
        <p>Visits: {total}</p>
      </div>
    );
    $[4] = message;
    $[5] = total;
    $[6] = t3;
  } else {
    t3 = $[6];
  }
  
  return t3;
}

export { Greeting };
```

DevTools'da Compiler-generated komponent ko'rish:

```
React DevTools (Components tab):
   ⚛ Memo(Greeting)
     props: { user, count }
     hooks: 1 (useMemoCache)
```

Hooks tab'da `useMemoCache` 1 ta hook sifatida ko'rinadi.

</details>

---

## Migration Path 6 Qadam

### Nazariya

Mavjud loyihaga React Compiler qo'shish — **incremental** jarayon. To'satdan butun loyihada enable qilish — ko'p violations va potential breaking changes. Tavsiya etilgan migration path 6 qadam:

### Qadam 1: ESLint Plugin O'rnatish

```bash
npm install -D eslint-plugin-react-compiler
```

```javascript
// eslint.config.js
export default [
  {
    plugins: { 'react-compiler': reactCompiler },
    rules: {
      'react-compiler/react-compiler': 'warn', // Birinchi marta — warn
    },
  },
];
```

Loyihani lint qilish:

```bash
npm run lint
```

Output — barcha Rules of React violations ro'yxati.

### Qadam 2: Violations'ni Tuzatish

Violations'ni 4 toifaga ajratish:

1. **Critical** — props/state mutation, render-phase side effect (xavfli, darhol fix).
2. **High** — conditional hooks, non-component hook calls.
3. **Medium** — render-phase localStorage/Date/Math (hydration mismatch xavfi).
4. **Low** — ref reads in render (performance, lekin to'g'ri output).

Har toifa uchun bosqichma-bosqich fix:

```typescript
// Critical fix — props mutation
// ❌ Before
function Cart({ items }: { items: Item[] }) {
  items.sort((a, b) => a.price - b.price);
  // ...
}

// ✅ After
function Cart({ items }: { items: Item[] }) {
  const sorted = [...items].sort((a, b) => a.price - b.price);
  // ...
}
```

### Qadam 3: ESLint'ni Error Mode'ga O'tkazish

Barcha violations tuzatilgandan keyin:

```javascript
// eslint.config.js
{
  rules: {
    'react-compiler/react-compiler': 'error', // Endi error
  },
}
```

CI pipeline'iga qo'shish — yangi PR'larda violations bloklanadi.

### Qadam 4: Compiler Plugin O'rnatish (Opt-in Mode)

```bash
npm install -D babel-plugin-react-compiler
```

```typescript
// vite.config.ts
const ReactCompilerConfig = {
  target: '19',
  compilationMode: 'annotation', // Faqat 'use memo' direktivasi bor fayllar
};
```

Bu mode'da hech qanday fayl avtomatik compile qilinmaydi — har fayl alohida opt-in.

### Qadam 5: Per-File Opt-In

Kichik komponentlardan boshlash (kichik blast radius):

```tsx
// src/components/Greeting.tsx
'use memo';

import type { ReactElement } from 'react';

export function Greeting({ name }: { name: string }): ReactElement {
  return <h1>Hello, {name}!</h1>;
}
```

Har fayl uchun:

1. Add `'use memo'` directive.
2. Test (manual + automated).
3. Profile (DevTools Profiler — re-render count taqqoslash).
4. Commit.

### Qadam 6: Manual Memoization'ni Olib Tashlash

Compiler enabled fayllarda manual `useMemo`/`useCallback`/`React.memo` ortiqcha bo'ladi:

```tsx
// Before (manual)
'use memo';

import { useMemo, useCallback, memo } from 'react';

export const ProductList = memo(function ProductList({ items, onClick }) {
  const total = useMemo(() => items.reduce(...), [items]);
  const handleClick = useCallback(() => onClick(items), [onClick, items]);
  return <ul>...</ul>;
});

// After (Compiler-only)
'use memo';

import type { ReactElement } from 'react';

export function ProductList({ items, onClick }: Props): ReactElement {
  const total = items.reduce((s, x) => s + x.price, 0);
  const handleClick = () => onClick(items);
  return <ul>...</ul>;
}
```

Olib tashlash darajasi loyihaga bog'liq — **gradual** approach tavsiya etiladi:

1. Yangi kod — manual memoization'siz.
2. Mavjud kod — touch'da olib tashlash (har refactor paytida).
3. Compiler-only kod — barcha komponentlar.

### Qadam 7 (qo'shimcha): `compilationMode: 'all'`

Loyiha barqaror bo'lganda — `'all'` mode'ga o'tish:

```typescript
const ReactCompilerConfig = {
  target: '19',
  compilationMode: 'infer', // Yoki 'all' — barcha React fayllar
};
```

Bu mode'da `'use memo'` directive shart emas — Compiler avtomatik aniqlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

Migration progress tracking:

```
Project:
├── src/
│   ├── components/
│   │   ├── Greeting.tsx    ✅ 'use memo' enabled
│   │   ├── Counter.tsx     ✅ 'use memo' enabled
│   │   ├── Form.tsx        ⬜ pending (Rules violation: state mutation)
│   │   ├── Modal.tsx       ✅ 'use memo' enabled
│   │   └── DataTable.tsx   ⬜ pending (Rules violation: ref read)
│   └── hooks/
│       ├── useDebounce.ts  ✅ 'use memo' enabled
│       └── useFetch.ts     ⬜ pending (Rules violation: race condition)
```

Profiling-based regression check workflow:

```
1. Component tanlanadi (masalan ProductList)
2. Profile snapshot — 'use memo' qo'shishdan oldin (DevTools Profiler record)
3. 'use memo' qo'shiladi
4. Profile snapshot — qo'shgandan keyin
5. Solishtirish:
   - actualDuration
   - re-render count
   - commit time
6. Regression bo'lsa — investigation, possible bail-out aniqlash
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Real-world migration misol:

```tsx
// === Step 1: Original code ===
import { useState, useMemo, useCallback, memo } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
}

interface UserListProps {
  users: User[];
  onSelect: (id: string) => void;
}

export const UserList = memo(function UserList({ users, onSelect }: UserListProps) {
  const [filter, setFilter] = useState('');
  
  const filteredUsers = useMemo(
    () => users.filter((u) => u.name.toLowerCase().includes(filter.toLowerCase())),
    [users, filter]
  );
  
  const handleSelect = useCallback(
    (id: string) => onSelect(id),
    [onSelect]
  );
  
  return (
    <div>
      <input
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
        placeholder="Filter..."
      />
      <ul>
        {filteredUsers.map((u) => (
          <li key={u.id} onClick={() => handleSelect(u.id)}>
            {u.name} ({u.email})
          </li>
        ))}
      </ul>
    </div>
  );
});
```

```tsx
// === Step 2: Compiler-enabled, manual memo retained ===
'use memo';

import { useState, useMemo, useCallback, memo } from 'react';
// Manual memo hali ishlaydi — Compiler bypass qilmaydi
// Lekin Compiler nazaridan ortiqcha

// ... (same code)
```

```tsx
// === Step 3: Manual memo olib tashlandi ===
'use memo';

import { useState } from 'react';
import type { ReactElement } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
}

interface UserListProps {
  users: User[];
  onSelect: (id: string) => void;
}

export function UserList({ users, onSelect }: UserListProps): ReactElement {
  const [filter, setFilter] = useState('');
  
  const filteredUsers = users.filter((u) =>
    u.name.toLowerCase().includes(filter.toLowerCase())
  );
  
  const handleSelect = (id: string) => onSelect(id);
  
  return (
    <div>
      <input
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
        placeholder="Filter..."
      />
      <ul>
        {filteredUsers.map((u) => (
          <li key={u.id} onClick={() => handleSelect(u.id)}>
            {u.name} ({u.email})
          </li>
        ))}
      </ul>
    </div>
  );
}

// Compiler avtomatik:
// - filteredUsers ni cache (deps: users, filter)
// - handleSelect ni cache (deps: onSelect)
// - <li> JSX'ni cache (per user)
// - <ul> JSX'ni cache (deps: filteredUsers)
// - Komponent darajasidagi memoization (memo equivalent)
```

</details>

---

## Compiler Cheklovlari va Bail-Out

### Nazariya

Compiler **barcha komponent'larni memoize qilolmaydi**. Quyidagi holatlarda bail-out qiladi:

### Cheklov 1: Mutation

```tsx
// ❌ Props/state mutation
function Bad({ items }: { items: Item[] }) {
  items.push(newItem); // Bail-out
  return <ul>...</ul>;
}

// ❌ Module-level mutation
let counter = 0;
function Counter() {
  counter += 1; // Bail-out
  return <div>{counter}</div>;
}
```

### Cheklov 2: Side Effects

```tsx
// ❌ Side effects in render
function Logger({ event }: { event: string }) {
  console.log(event); // Warning, not bail-out (logs)
  fetch('/api/log', { body: event }); // Bail-out
  document.title = event; // Bail-out
  return <div>{event}</div>;
}
```

### Cheklov 3: Conditional Hooks

```tsx
// ❌ Conditional hook
function Form({ enabled }: { enabled: boolean }) {
  if (enabled) {
    const [count] = useState(0); // Bail-out + ESLint error
  }
}
```

### Cheklov 4: Refs Render-Time Read/Write

```tsx
// ❌ Ref read/write in render
function Form() {
  const ref = useRef(0);
  ref.current += 1; // Bail-out
  return <div>{ref.current}</div>;
}

// ✅ Ref read/write in event handler / effect
function Form() {
  const ref = useRef(0);
  const handleClick = () => {
    ref.current += 1; // OK
  };
  return <button onClick={handleClick}>Click</button>;
}
```

### Cheklov 5: Async Hook (R19)

```tsx
// ❌ Async hook (kelajakda support qilinishi mumkin)
async function useAsyncData(url: string) {
  const data = await fetch(url).then((r) => r.json());
  return data;
}
```

R19'da `use(promise)` bilan bu ishlaydi (cross-ref `23-r19-hooks.md`), lekin Compiler hali to'liq qo'llab-quvvatlamasligi mumkin.

### Cheklov 6: Complex Closures

```tsx
// ❌ Hook returning function bilan mutable closure
function useUnusual() {
  let value = 0;
  return () => {
    value += 1; // Mutable closure
    return value;
  };
}

// Compiler: bail-out — closure semantics noaniq
```

### Bail-out Effects

Bail-out qilingan komponent **manual memoization**'ga qaytadi:

- Manual `useMemo`/`useCallback`/`React.memo` hali ishlaydi.
- Compiler hech qanday optimization qo'shmaydi.
- Console warning (development) chiqadi.

```javascript
// Console warning misol
[React Compiler] Skipping component "BadComponent": Cannot mutate non-local value 'items' during render. See: https://react.dev/reference/rules/components-and-hooks-must-be-pure
```

<details>
<summary><strong>Under the Hood</strong></summary>

Bail-out detection algorithm:

```javascript
// Compiler internals (oddiylashtirilgan)
function shouldCompile(component) {
  const hir = buildHIR(component);
  
  // Validation passes
  const violations = [];
  
  for (const inst of hir.instructions) {
    // 1. Mutation detection
    if (isMutation(inst) && !isLocalMutation(inst)) {
      violations.push({
        kind: 'NonLocalMutation',
        location: inst.loc,
      });
    }
    
    // 2. Side effect detection
    if (isSideEffect(inst) && isInRenderPhase(inst)) {
      violations.push({
        kind: 'RenderSideEffect',
        location: inst.loc,
      });
    }
    
    // 3. Conditional hook
    if (isHookCall(inst) && isInsideConditional(inst)) {
      violations.push({
        kind: 'ConditionalHook',
        location: inst.loc,
      });
    }
    
    // 4. Ref render-time access
    if (isRefAccess(inst) && isInRenderPhase(inst)) {
      violations.push({
        kind: 'RefInRender',
        location: inst.loc,
      });
    }
  }
  
  // Decide based on panicThreshold
  const criticalViolations = violations.filter((v) => isCritical(v));
  
  switch (config.panicThreshold) {
    case 'all_errors':
      return violations.length === 0;
    case 'critical_errors':
      return criticalViolations.length === 0;
    case 'none':
      return true; // Best-effort, ignore violations
  }
}
```

`'none'` mode'da Compiler **best-effort** — violations bo'lsa ham memoize qilishga harakat qiladi, lekin natija noto'g'ri bo'lishi mumkin (mutation propagation, stale references).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Bail-out scenarios va workaround'lar:

```tsx
// 1. Async data — use(promise) bilan
import { use } from 'react';
import type { ReactElement } from 'react';

interface User {
  id: string;
  name: string;
}

function UserProfile({ userPromise }: { userPromise: Promise<User> }): ReactElement {
  const user = use(userPromise); // ✅ R19, Compiler-friendly
  return <h1>{user.name}</h1>;
}
```

```tsx
// 2. Mutable state — useReducer afzal
import { useReducer } from 'react';

interface State {
  count: number;
  history: number[];
}

type Action = { type: 'INCREMENT' } | { type: 'RESET' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'INCREMENT':
      return {
        count: state.count + 1,
        history: [...state.history, state.count + 1],
      };
    case 'RESET':
      return { count: 0, history: [] };
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0, history: [] });
  return (
    <div>
      <span>{state.count}</span>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
    </div>
  );
}
// Reducer pure — Compiler memoize qila oladi
```

```tsx
// 3. Render-time random — useId bilan
import { useId } from 'react';
import type { ReactElement } from 'react';

function FormField({ label }: { label: string }): ReactElement {
  const id = useId(); // ✅ SSR-safe, Compiler-friendly
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}
```

```tsx
// 4. Mutation — local copy
import type { ReactElement } from 'react';

interface Item {
  id: string;
  name: string;
  price: number;
}

function PriceList({ items }: { items: Item[] }): ReactElement {
  // ✅ Local copy mutation OK
  const sorted = [...items].sort((a, b) => a.price - b.price);
  
  return (
    <ul>
      {sorted.map((item) => (
        <li key={item.id}>{item.name}: ${item.price}</li>
      ))}
    </ul>
  );
}
```

</details>

---

## Manual Memo bilan Munosabat

### Nazariya

Compiler enabled bo'lsa, **manual `useMemo`/`useCallback`/`React.memo`** ortiqcha bo'lib qoladi. Lekin ular hali ishlaydi va Compiler ularni override qilmaydi.

Strategiyalar:

### Strategy 1: Yangi Kod — Compiler-only

```tsx
'use memo';

// ✅ Manual memo'siz
function NewComponent({ items, onClick }: Props): ReactElement {
  const total = items.reduce((s, x) => s + x.price, 0);
  const handleClick = (id: string) => onClick(id);
  return <ul>...</ul>;
}
```

### Strategy 2: Mavjud Kod — Gradual Removal

Har refactor paytida olib tashlash:

```tsx
// Before
'use memo';

import { useMemo, useCallback, memo } from 'react';

const OldComponent = memo(function OldComponent({ items, onClick }) {
  const total = useMemo(() => items.reduce(...), [items]);
  const handleClick = useCallback((id) => onClick(id), [onClick]);
  return <ul>...</ul>;
});

// After (refactor qilingan)
'use memo';

import type { ReactElement } from 'react';

function OldComponent({ items, onClick }: Props): ReactElement {
  const total = items.reduce((s, x) => s + x.price, 0);
  const handleClick = (id: string) => onClick(id);
  return <ul>...</ul>;
}
```

### Strategy 3: Aralash — Critical Path Manual

Performance-critical hot path'larda manual qoldirish (Compiler-mosligini debug qilish'gacha):

```tsx
'use memo';

import { useMemo } from 'react';
import type { ReactElement } from 'react';

function VirtualizedList({ items, height }: Props): ReactElement {
  // Compiler avtomatik memoize qiladi
  const visibleItems = items.slice(0, 100);
  
  // Critical: virtualization complex computation
  // Manual useMemo retained for explicit cache control
  const itemHeights = useMemo(
    () => visibleItems.map((item) => calculateItemHeight(item)),
    [visibleItems]
  );
  
  return <div style={{ height }}>...</div>;
}
```

### Compiler vs Manual — Granularity Farqi

```tsx
// Manual: useMemo bir level granularity
const userInfo = useMemo(
  () => ({
    displayName: user.firstName + ' ' + user.lastName,
    initials: user.firstName[0] + user.lastName[0],
  }),
  [user.firstName, user.lastName]
);

// Compiler: per-property granularity
const displayName = user.firstName + ' ' + user.lastName; // Scope 1
const initials = user.firstName[0] + user.lastName[0]; // Scope 2
// user.firstName o'zgarsa — ikkalasi recompute
// user.lastName o'zgarsa — ikkalasi recompute
// Lekin user.email o'zgarsa — ikkalasi cached
```

Manual versiyada Compiler granular tracking qila olmaydi (deps array bo'yicha invalidation).

### `React.memo` Munosabati

`React.memo` HOC — komponent'ni shallow props comparison bilan memoize qiladi. Compiler avtomatik komponent darajasidagi memoization qo'shadi (parent re-render bypass), shu sababdan `React.memo` ortiqcha:

```tsx
// Compiler-only
function ProductCard({ product }: { product: Product }): ReactElement {
  return <div>...</div>;
}
// Compiler avtomatik wrap qiladi memo equivalent bilan

// Manual + Compiler — ortiqcha
const ProductCard = memo(function ProductCard({ product }: { product: Product }): ReactElement {
  return <div>...</div>;
});
// Manual memo hali ishlaydi, lekin Compiler buni override qila olmaydi
```

<details>
<summary><strong>Under the Hood</strong></summary>

Compiler-generated memoization vs manual:

| Aspect | Manual `useMemo` | Compiler `_c[i]` |
|--------|-------------------|------------------|
| Granularity | Per `useMemo` call | Per reactive scope (finer) |
| Deps array | Manual (error-prone) | Auto-inferred |
| Cache slot | Hook chain'da slot | Single `_c` array |
| Hook overhead | 1 hook per `useMemo` | 1 `useMemoCache` total |
| Comparison | `Object.is` (default) | `Object.is` (default) |
| Custom equality | Argument'siz support | Yo'q (kelajakda?) |

Performance — Compiler usually faster:

- Single `useMemoCache(N)` chaqiriq vs ko'p `useMemo` chaqiriq (N hook).
- Granular invalidation — kichik scope alohida cache miss/hit.
- Bundle size — Compiler output kichikroq (ozgina array comparisons vs HOC wrap).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Migration'ning real-world misoli:

```tsx
// === Before (manual) ===
import { useState, useMemo, useCallback, memo } from 'react';

interface Product {
  id: string;
  name: string;
  price: number;
  inStock: boolean;
}

interface ProductGridProps {
  products: Product[];
  filter: 'all' | 'in-stock' | 'sale';
  onAddToCart: (id: string) => void;
}

const ProductGrid = memo(function ProductGrid({
  products,
  filter,
  onAddToCart,
}: ProductGridProps) {
  const filtered = useMemo(() => {
    return products.filter((p) => {
      if (filter === 'in-stock') return p.inStock;
      if (filter === 'sale') return p.price < 50;
      return true;
    });
  }, [products, filter]);
  
  const sorted = useMemo(
    () => [...filtered].sort((a, b) => a.price - b.price),
    [filtered]
  );
  
  const handleAdd = useCallback(
    (id: string) => onAddToCart(id),
    [onAddToCart]
  );
  
  return (
    <div className="grid">
      {sorted.map((p) => (
        <ProductCard
          key={p.id}
          product={p}
          onAdd={() => handleAdd(p.id)}
        />
      ))}
    </div>
  );
});

const ProductCard = memo(function ProductCard({
  product,
  onAdd,
}: {
  product: Product;
  onAdd: () => void;
}) {
  return (
    <div className="card">
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button onClick={onAdd} disabled={!product.inStock}>
        Add
      </button>
    </div>
  );
});
```

```tsx
// === After (Compiler) ===
'use memo';

import { useState } from 'react';
import type { ReactElement } from 'react';

interface Product {
  id: string;
  name: string;
  price: number;
  inStock: boolean;
}

interface ProductGridProps {
  products: Product[];
  filter: 'all' | 'in-stock' | 'sale';
  onAddToCart: (id: string) => void;
}

export function ProductGrid({
  products,
  filter,
  onAddToCart,
}: ProductGridProps): ReactElement {
  const filtered = products.filter((p) => {
    if (filter === 'in-stock') return p.inStock;
    if (filter === 'sale') return p.price < 50;
    return true;
  });
  
  const sorted = [...filtered].sort((a, b) => a.price - b.price);
  
  const handleAdd = (id: string) => onAddToCart(id);
  
  return (
    <div className="grid">
      {sorted.map((p) => (
        <ProductCard
          key={p.id}
          product={p}
          onAdd={() => handleAdd(p.id)}
        />
      ))}
    </div>
  );
}

interface ProductCardProps {
  product: Product;
  onAdd: () => void;
}

export function ProductCard({ product, onAdd }: ProductCardProps): ReactElement {
  return (
    <div className="card">
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button onClick={onAdd} disabled={!product.inStock}>
        Add
      </button>
    </div>
  );
}

// Hajm: ~20 qator manual memoization wrapper olib tashlandi
// Behavior: ekvivalent (Compiler granular memoization)
```

</details>

---

## Library Compatibility

### Nazariya

Compiler shu loyihaning kodini compile qiladi, lekin **3rd-party kutubxonalarni** compile qilmaydi (default). Library kodi `node_modules` ichida bo'lib, build pipeline'idan tashqarida.

### Compatible Libraries

Library Rules of React'ga rioya qilsa — Compiler bilan **mos**:

- React core (`react`, `react-dom`)
- React Router v6+
- TanStack Query v5+
- Zustand v4+
- Redux Toolkit (modern)
- Jotai
- shadcn/ui
- Headless UI

### Potential Issues

Library mutation yoki side effect ishlatsa — Compiler kodni o'rab oladigan komponent'larda muammo bo'lishi mumkin:

```tsx
// 3rd-party library: legacy mutation
function LegacyComponent({ data }: { data: Data }) {
  // legacyLib internal mutation qiladi (props mutation)
  legacyLib.process(data); // ❌ Library kodi mutation qiladi
  return <div>{data.result}</div>;
}
```

Compiler bu `LegacyComponent`'ni bail-out qiladi (mutation traced through library call).

### Workaround: Library Boundary

Library chaqiriqlari'ni komponent boundary'sida izolatsiya qilish:

```tsx
// ✅ Library chaqirig'i useEffect'da
import { useEffect, useState } from 'react';
import type { ReactElement } from 'react';

function LegacyAdapter({ data }: { data: Data }): ReactElement {
  const [result, setResult] = useState<string | null>(null);
  
  useEffect(() => {
    const processed = legacyLib.process(data); // OK in effect
    setResult(processed.result);
  }, [data]);
  
  if (!result) return <div>Loading...</div>;
  return <div>{result}</div>;
}
```

### Library Compilation (Advanced)

Library author'lar kodini Compiler bilan kompilyatsiya qilib, dist'ga **pre-compiled** kod yuborishi mumkin:

```javascript
// library-package/src/index.tsx (source)
'use memo';

export function MyLibComponent(props) {
  // ...
}

// library-package/dist/index.js (compiled, published to npm)
import { c as _c } from 'react/compiler-runtime';
export function MyLibComponent(t0) {
  const $ = _c(N);
  // ...
}
```

Bu approach hali keng tarqalmagan — kelajakda library author'lar publish strategy'ga qo'shishi kutilmoqda.

<details>
<summary><strong>Under the Hood</strong></summary>

Babel plugin source filtering:

```javascript
// vite.config.ts
const ReactCompilerConfig = {
  target: '19',
  sources: (filename) => {
    // Faqat src/ ichidagi fayllar
    return filename.includes('/src/') && /\.tsx?$/.test(filename);
  },
};
```

Bu filter — `node_modules` fayllarni exclude qiladi (default behavior).

Library kompilyatsiya — npm publish jarayonida:

```json
{
  "name": "my-react-lib",
  "version": "2.0.0",
  "main": "dist/cjs/index.js",
  "module": "dist/esm/index.js",
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.js"
    }
  },
  "scripts": {
    "build": "babel src --out-dir dist --extensions .tsx,.ts --plugins babel-plugin-react-compiler"
  }
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Library boundary patterns:

```tsx
// 1. Map adapter
import { useEffect, useRef } from 'react';
import type { ReactElement } from 'react';

interface MapProps {
  center: [number, number];
  zoom: number;
}

function MapAdapter({ center, zoom }: MapProps): ReactElement {
  const ref = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    if (!ref.current) return;
    
    const map = legacyMapLib.create(ref.current, { center, zoom });
    
    return () => {
      map.destroy(); // Cleanup
    };
  }, [center, zoom]);
  
  return <div ref={ref} style={{ width: '100%', height: 400 }} />;
}
```

```tsx
// 2. 3rd-party form library wrapper
import { useState } from 'react';
import type { ReactElement } from 'react';

interface FormData {
  email: string;
  password: string;
}

function LoginForm(): ReactElement {
  const [formData, setFormData] = useState<FormData>({ email: '', password: '' });
  
  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    legacyFormLib.submit(formData); // OK in event handler
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={formData.email}
        onChange={(e) => setFormData({ ...formData, email: e.target.value })}
      />
      <input
        type="password"
        value={formData.password}
        onChange={(e) => setFormData({ ...formData, password: e.target.value })}
      />
      <button type="submit">Login</button>
    </form>
  );
}
```

</details>

---

## Performance Implikatsiyalari

### Nazariya

Compiler — performance optimization tool. Lekin har holatda manual memoization'dan tezroq emas. Real-world taqqoslash kerak.

### Compiler Win'lari

1. **Granular memoization** — `useMemo` bilan emaslik qiyin bo'lgan property-level granularity.
2. **Default-on optimization** — har komponentni avtomatik (manual qaror kerak emas).
3. **Bundle reduction** — `useMemo`/`useCallback`/`memo` HOC chaqiriqlari olib tashlanadi.
4. **Hook overhead reduction** — bir `useMemoCache(N)` ko'p `useMemo` chaqiriq o'rniga.

### Compiler Loss'lari

1. **Build-time overhead** — kompilyatsiya sekin (katta loyihada bundle qilish vaqti oshishi mumkin).
2. **Generated code size** — har komponentda cache logic — bundle hajmi hech qachon kichik bo'lmaydi.
3. **Runtime cache memory** — har komponent instance N ta cache slot saqlaydi.
4. **Bail-out cost** — bail-out komponent'lar manual memoization'siz, performance regress.

### Real-World Profile

DevTools Profiler bilan taqqoslash (cross-ref `34-profiling.md`):

```typescript
// Manual memoization
import { Profiler } from 'react';

<Profiler id="ManualVersion" onRender={onRenderCallback}>
  <ManualUserList users={users} />
</Profiler>

// Compiler version
<Profiler id="CompilerVersion" onRender={onRenderCallback}>
  <CompilerUserList users={users} />
</Profiler>

// onRenderCallback
function onRenderCallback(
  id: string,
  phase: 'mount' | 'update',
  actualDuration: number,
  baseDuration: number,
  startTime: number,
  commitTime: number
) {
  console.log(id, phase, actualDuration);
}
```

### Memory

Compiler cache memory — sifat-darajadagi taqqoslash:

- **Compiler approach:** komponent instance bo'yicha bitta `useMemoCache(N)` hook slot va N elementli cache array (har slot V8 heap pointer hajmida). Cached value'lar — chiqish hajmiga bog'liq (object/JSX).
- **Manual `useMemo`:** har `useMemo` chaqiriq alohida hook slot va `[value, deps]` saqlash. K ta `useMemo` chaqirig'ida K ta hook slot.

**Memory efficiency:** Compiler approach odatda kichikroq overhead — bitta cache array ko'p hook slot'lar o'rniga. Aniq baytlar V8 build, scope hajmi va cached value tipiga bog'liq, lekin order of magnitude — bir komponent instance uchun bir necha KB.

<details>
<summary><strong>Under the Hood</strong></summary>

Real-world Meta production data (React Conf 2024 keynote'da e'lon qilingan):
- Instagram web — performance improvement (median commit duration kamayishi)
- Quest store — sezilarli improvement (interaction-heavy pages)

Bu ma'lumotlar Meta'ning ichki monitoring'idan — har loyiha uchun farq qilishi mumkin. **Profile o'zingiz** — har joyda yaxshi natija kafolat emas.

Bundle hajmi taqqoslash:

- Manual memoization: `useMemo`/`useCallback`/`memo` import + HOC wrappers
- Compiler: `react/compiler-runtime` import + cache logic per component

Compiler output ko'pincha biroz kattaroq, lekin runtime ko'pincha tezroq. Trade-off — runtime foyda bundle cost'dan ko'proq odatda.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Profile-based comparison:

```tsx
'use memo';

import { Profiler } from 'react';
import type { ReactElement, ProfilerOnRenderCallback } from 'react';

const onRender: ProfilerOnRenderCallback = (
  id,
  phase,
  actualDuration
) => {
  console.log(`[${id}] ${phase}: ${actualDuration.toFixed(2)}ms`);
};

interface User {
  id: string;
  name: string;
}

function UserList({ users }: { users: User[] }): ReactElement {
  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}

function App({ users }: { users: User[] }): ReactElement {
  return (
    <Profiler id="UserList" onRender={onRender}>
      <UserList users={users} />
    </Profiler>
  );
}

// Console: [UserList] mount: ...ms
// Console: [UserList] update: ...ms
```

Bundle size monitoring:

```bash
# Bundle analyzer
npm install -D vite-bundle-analyzer
```

```typescript
// vite.config.ts
import { analyzer } from 'vite-bundle-analyzer';

export default defineConfig({
  plugins: [
    react({ babel: { plugins: [['babel-plugin-react-compiler', {}]] } }),
    analyzer(),
  ],
});
```

```bash
# Build
npm run build

# Analyzer dashboard'da Compiler-generated kod hajmini ko'rish
```

</details>

---

## Future Paradigm — Manual Memoization Kelajagi

### Nazariya

Compiler stable bo'lgan paytda **manual memoization yo'q bo'lib ketishi** kutilmoqda. Bu fundamental paradigm shift:

### Compiler'siz Paradigm (idiomatic Compiler keng tarqalishidan oldin)

- Dasturchi qachon `useMemo` ishlatishni qaror qiladi.
- Premature optimization yoki under-optimization keng tarqalgan.
- Code clutter — `useMemo`/`useCallback` chaqiriqlari logic'dan ko'p hajm.
- React.memo HOC — komponent boundary'sida qo'shimcha wrapper.
- Linter exhaustive-deps yordam beradi, lekin to'liq emas.

### Compiler-Driven Paradigm (Compiler enabled loyihalarda)

- Dasturchi pure logic yozadi.
- Compiler memoization avtomatik qo'shadi.
- Code clarity — logic-only.
- Performance default-on (premature optimization yo'q, under-optimization kam).
- Rules of React enforcement — runtime invariants build-time'da tekshiriladi.

> **Eslatma:** Compiler React versiyasiga bog'liq emas (`target: '17' | '18' | '19'`). Paradigm shift Compiler'ning **ekosistemada keng tarqalishi** bilan bog'liq, ma'lum React versiyasi bilan emas. Quyidagi timeline ham R19 release atrofidagi bozor tendentsiyasini ko'rsatadi, qattiq versiya-shartlilik emas.

### Migration Timeline

| Vaqt | Holat |
|------|-------|
| 2024 | Compiler beta, Meta production'da |
| 2025 April | `babel-plugin-react-compiler@1.0` stable |
| 2025–2026 | Major library/framework adoption (Next.js, Vite ecosystem) |
| 2027+ | Default-on (Compiler standart React app'da) |
| 2028+ | Manual `useMemo`/`useCallback` legacy (deprecation hali ehtimoldan uzoq, lekin idiomatic emas) |

### Hooks Kelajagi

Manual hook'lar **olib tashlanmaydi** — backward compatibility React'ning kuchli xususiyati:

- `useMemo` — hali ishlaydi, library author'lar uchun foydali (custom equality comparator).
- `useCallback` — hali ishlaydi, ortiqcha lekin breakage yo'q.
- `React.memo` — hali ishlaydi.
- `eslint-plugin-react-hooks/exhaustive-deps` — hali kerakli (custom hook'lar uchun).

**O'zgarish** — idiomatic React kod **manual memoization'siz**:

```tsx
// 2024 idiomatic
import { useMemo, useCallback, memo } from 'react';

const Component = memo(function Component({ items, onClick }) {
  const filtered = useMemo(() => items.filter(...), [items]);
  const handleClick = useCallback((id) => onClick(id), [onClick]);
  return <ul>...</ul>;
});

// 2027+ idiomatic
import type { ReactElement } from 'react';

function Component({ items, onClick }: Props): ReactElement {
  const filtered = items.filter((x) => /* ... */);
  const handleClick = (id: string) => onClick(id);
  return <ul>...</ul>;
}
```

### Compiler va Concurrent Rendering

Compiler **Concurrent invariants ustiga quriladi** (cross-ref `30-concurrent-react.md`):

- Render Purity — Compiler talab qiladi.
- State immutability — Compiler talab qiladi.
- Effect symmetry — Compiler tekshirmaydi (runtime), lekin tavsiya qiladi.
- External subscription consistency — `useSyncExternalStore` orqali (Compiler buni respect qiladi).

Compiler + Concurrent — **synergic** — ikkalasi ham kodga bir xil kafolat (purity, immutability) talab qiladi va birgalikda performance va correctness'ni yaxshilaydi.

### React Server Components va Compiler

RSC (cross-ref `39-rsc-server-actions.md`) ham Compiler bilan ishlay oladi:

- Server Component — server'da render, client bundle'siz.
- Compiler server'da ishlamaydi (memoization runtime React faqat client'da).
- Client Component — Compiler memoize qiladi.
- `'use client'` boundary'da Compiler activate.

<details>
<summary><strong>Under the Hood</strong></summary>

Compiler'ning fundamental approach — **prove correctness, then optimize**:

1. Static analysis kod xavfsizligini isbotlaydi (Rules of React compliance).
2. Memoization qoidalar buzilmasa qo'shiladi.
3. Runtime correctness 100% — Compiler hech qachon "noto'g'ri optimization" qilmaydi.

Roadmap (taxminiy):

```
2025:
- Stable v1.0 release ✅ (April 2025)
- Next.js native integration
- ESLint rule expansion
- Source map improvements

2026:
- SWC plugin (Babel'siz integration)
- Better library compatibility
- Custom equality comparator support
- React Native integration

2027+:
- Default-on in modern frameworks (Next.js, Remix, Vite-based React templates) — create-react-app rasman 2025'da deprecated, modern starter'lar uchun aktual emas
- TypeScript-aware optimizations
- Production telemetry integration
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Idiomatic kelajak:

```tsx
'use memo';

import { useState } from 'react';
import type { ReactElement } from 'react';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

interface TodoAppProps {
  initialTodos: Todo[];
}

export function TodoApp({ initialTodos }: TodoAppProps): ReactElement {
  const [todos, setTodos] = useState(initialTodos);
  const [filter, setFilter] = useState<'all' | 'active' | 'completed'>('all');
  const [newText, setNewText] = useState('');
  
  // Compiler avtomatik filteredTodos ni memoize qiladi
  const filteredTodos = todos.filter((t) => {
    if (filter === 'active') return !t.completed;
    if (filter === 'completed') return t.completed;
    return true;
  });
  
  // Compiler avtomatik handlers'ni memoize qiladi
  const addTodo = () => {
    if (!newText.trim()) return;
    setTodos((prev) => [
      ...prev,
      { id: crypto.randomUUID(), text: newText, completed: false },
    ]);
    setNewText('');
  };
  
  const toggleTodo = (id: string) => {
    setTodos((prev) =>
      prev.map((t) => (t.id === id ? { ...t, completed: !t.completed } : t))
    );
  };
  
  const deleteTodo = (id: string) => {
    setTodos((prev) => prev.filter((t) => t.id !== id));
  };
  
  return (
    <div>
      <input
        value={newText}
        onChange={(e) => setNewText(e.target.value)}
        onKeyDown={(e) => e.key === 'Enter' && addTodo()}
      />
      <button onClick={addTodo}>Add</button>
      
      <div>
        <button onClick={() => setFilter('all')}>All</button>
        <button onClick={() => setFilter('active')}>Active</button>
        <button onClick={() => setFilter('completed')}>Completed</button>
      </div>
      
      <ul>
        {filteredTodos.map((t) => (
          <li key={t.id}>
            <input
              type="checkbox"
              checked={t.completed}
              onChange={() => toggleTodo(t.id)}
            />
            <span style={{ textDecoration: t.completed ? 'line-through' : 'none' }}>
              {t.text}
            </span>
            <button onClick={() => deleteTodo(t.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  );
}

// Hech qanday useMemo, useCallback, React.memo
// Compiler avtomatik:
// - filteredTodos memoize (deps: todos, filter)
// - addTodo, toggleTodo, deleteTodo callbacks (stable)
// - <li> JSX scopes (per todo, granular)
// - <ul> JSX scope (deps: filteredTodos)
// - Komponent darajasidagi memoization (parent re-render bypass)
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: Compiler Bail-out Silent

Bail-out qilingan komponent **performance regression** bo'lishi mumkin (manual memoization yo'q + Compiler optimization yo'q). Console warning'larni e'tiborsiz qoldirmaslik.

```tsx
// Bail-out reason: ref read in render
function Form() {
  const inputRef = useRef<HTMLInputElement>(null);
  const value = inputRef.current?.value ?? ''; // ❌ Bail-out
  return <div>{value}</div>;
}
```

**Yechim:** ref read'ni event handler/effect'ga ko'chirish, yoki state ishlatish.

### Gotcha 2: `'use memo'` va `'use no memo'` Direktivalar

Fayl boshida directive yozish — fayl faqat string emas, file-level pragma:

```tsx
// ✅ To'g'ri
'use memo';

import ...

// ❌ Noto'g'ri (function ichida)
function Comp() {
  'use memo'; // ❌ Function-level, ignored
}
```

### Gotcha 3: Compiler + React DevTools Hooks Tab

Compiler komponent'da `useMemoCache` 1 hook sifatida ko'rinadi. Manual `useMemo`/`useCallback` ko'rinmaydi (chunki yo'q). Hook count manual versiyaga qaraganda kamroq.

```
Manual Counter component:
   Hooks tab:
     1. State (count: 0)
     2. Memo (doubled)
     3. Callback (handleIncrement)

Compiler Counter component:
   Hooks tab:
     1. State (count: 0)
     2. MemoCache [4] (cache slots)
```

### Gotcha 4: Hot Module Replacement (HMR)

HMR Compiler-generated kod bilan ba'zan muammo:

- Cache slot count o'zgarsa (komponent kodi o'zgarishi) — full reload kerak.
- State preservation buzilishi mumkin (kichik probability).

**Yechim:** development'da component kodini katta o'zgartirilganda full reload (`Cmd+R`/`Ctrl+R`).

### Gotcha 5: Inline JSX'da Reference Identity

```tsx
'use memo';

function Parent() {
  return <Child config={{ debounce: 300 }} />;
  // Compiler: <Child> JSX scope ichida {{ debounce: 300 }} har render yangi
  // Compiler hozircha bu literal'ni cache qilmaydi (R1.0)
  // Child re-render'da config har gal yangi reference
}
```

**Yechim:** module-level constant yoki extract qilish:

```tsx
'use memo';

const CHILD_CONFIG = { debounce: 300 };

function Parent() {
  return <Child config={CHILD_CONFIG} />; // Stable reference
}
```

Bu cheklov kelajakda Compiler tomonidan tuzatilishi kutilmoqda.

---

## Common Mistakes

### ❌ Xato 1: ESLint Plugin'siz Compiler Enable

```typescript
// ❌ Faqat Compiler plugin
const ReactCompilerConfig = { target: '19' };
// ESLint plugin yo'q — violations silent bail-out
```

```typescript
// ✅ Birinchi ESLint, keyin Compiler
// 1. eslint-plugin-react-compiler enable
// 2. Violations fix
// 3. babel-plugin-react-compiler enable
```

### ❌ Xato 2: Manual Memo'ni Birinchi Olib Tashlash

```tsx
// ❌ Compiler enable qilingan, lekin manual memo olib tashlangan
'use memo';

function Component({ items }: Props) {
  const total = items.reduce(...);  // Compiler enabled? Bail-out bo'lsa-chi?
  return <div>{total}</div>;
}
// Bail-out bo'lsa — manual memo'siz performance loss
```

```tsx
// ✅ Manual'ni saqlab, gradual olib tashlash
'use memo';

import { useMemo } from 'react';

function Component({ items }: Props) {
  // Compiler bypass qilmaydi, lekin profiling bilan tasdiqlangach olib tashlash
  const total = useMemo(() => items.reduce(...), [items]);
  return <div>{total}</div>;
}
```

### ❌ Xato 3: Library Mutation Ignored

```tsx
'use memo';

function Wrapper({ data }: { data: Data }) {
  legacyLib.transform(data); // ❌ Library mutation traced — bail-out
  return <div>{data.result}</div>;
}
```

```tsx
// ✅ Effect boundary
import { useEffect, useState } from 'react';

function Wrapper({ data }: { data: Data }) {
  const [result, setResult] = useState<string | null>(null);
  
  useEffect(() => {
    const transformed = legacyLib.transform(data);
    setResult(transformed.result);
  }, [data]);
  
  if (!result) return null;
  return <div>{result}</div>;
}
```

### ❌ Xato 4: `panicThreshold: 'none'` Production'da

```typescript
// ❌ Bail-out warning'lar yo'qoladi, debug qiyin
const ReactCompilerConfig = {
  panicThreshold: 'none',
};
```

```typescript
// ✅ Development'da `'all_errors'`, production'da `'critical_errors'`
const ReactCompilerConfig = {
  panicThreshold: process.env.NODE_ENV === 'production' ? 'critical_errors' : 'all_errors',
};
```

### ❌ Xato 5: Compiler Performance Profile'siz Enable

```typescript
// ❌ Profile qilmasdan global enable
const ReactCompilerConfig = { compilationMode: 'all' };
// Performance regression bo'lishi mumkin (bail-out, complex components)
```

```typescript
// ✅ Avval per-file opt-in, profile, keyin global
const ReactCompilerConfig = { compilationMode: 'annotation' };
// Per-file 'use memo' directive bilan, har birini profile qilib enable
```

---

## Amaliy Mashqlar

### Mashq 1: Manual Memo'dan Compiler'ga Migratsiya (Oson)

Quyidagi komponentni Compiler-friendly shaklga o'tkazing — manual `useMemo`/`useCallback`/`memo`'larni olib tashlang.

```tsx
import { useMemo, useCallback, memo } from 'react';

interface Article {
  id: string;
  title: string;
  views: number;
}

interface ArticleListProps {
  articles: Article[];
  sortBy: 'title' | 'views';
  onClick: (id: string) => void;
}

export const ArticleList = memo(function ArticleList({
  articles,
  sortBy,
  onClick,
}: ArticleListProps) {
  const sorted = useMemo(() => {
    return [...articles].sort((a, b) => {
      if (sortBy === 'title') return a.title.localeCompare(b.title);
      return b.views - a.views;
    });
  }, [articles, sortBy]);
  
  const handleClick = useCallback(
    (id: string) => onClick(id),
    [onClick]
  );
  
  return (
    <ul>
      {sorted.map((a) => (
        <li key={a.id} onClick={() => handleClick(a.id)}>
          {a.title} ({a.views})
        </li>
      ))}
    </ul>
  );
});
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
'use memo';

import type { ReactElement } from 'react';

interface Article {
  id: string;
  title: string;
  views: number;
}

interface ArticleListProps {
  articles: Article[];
  sortBy: 'title' | 'views';
  onClick: (id: string) => void;
}

export function ArticleList({
  articles,
  sortBy,
  onClick,
}: ArticleListProps): ReactElement {
  const sorted = [...articles].sort((a, b) => {
    if (sortBy === 'title') return a.title.localeCompare(b.title);
    return b.views - a.views;
  });
  
  const handleClick = (id: string) => onClick(id);
  
  return (
    <ul>
      {sorted.map((a) => (
        <li key={a.id} onClick={() => handleClick(a.id)}>
          {a.title} ({a.views})
        </li>
      ))}
    </ul>
  );
}
```

Olib tashlangan: `memo`, `useMemo`, `useCallback`. Compiler avtomatik:
- `sorted` ni cache (deps: articles, sortBy)
- `handleClick` ni cache (deps: onClick)
- `<li>` JSX'ni cache (per article)
- `<ul>` JSX'ni cache (deps: sorted)
- Komponent darajasidagi memoization

</details>

### Mashq 2: Rules of React Violations Tuzatish (Oson)

Quyidagi komponentda 4 ta Rules of React violation bor. Topib, tuzating.

```tsx
let pageViews = 0;

interface Product {
  id: string;
  name: string;
  price: number;
}

function ProductPage({ product }: { product: Product }): ReactElement {
  pageViews += 1;
  
  product.name = product.name.toUpperCase();
  
  fetch('/api/track', {
    method: 'POST',
    body: JSON.stringify({ productId: product.id }),
  });
  
  document.title = product.name;
  
  return (
    <div>
      <h1>{product.name}</h1>
      <p>Price: ${product.price}</p>
      <p>Page views: {pageViews}</p>
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

4 ta violation:
1. `pageViews += 1` — module-level mutation
2. `product.name = product.name.toUpperCase()` — props mutation
3. `fetch(...)` — render-phase side effect (network call)
4. `document.title = product.name` — render-phase side effect (DOM mutation)

Tuzatilgan:

```tsx
'use memo';

import { useState, useEffect, useRef } from 'react';
import type { ReactElement } from 'react';

interface Product {
  id: string;
  name: string;
  price: number;
}

function ProductPage({ product }: { product: Product }): ReactElement {
  const pageViewsRef = useRef(0);
  
  useEffect(() => {
    pageViewsRef.current += 1;
  });
  
  // Immutable transform render Phase'da OK (local computation)
  const displayName = product.name.toUpperCase();
  
  // Side effect: tracking
  useEffect(() => {
    fetch('/api/track', {
      method: 'POST',
      body: JSON.stringify({ productId: product.id }),
    });
    // Eslatma: Strict Mode 2x cycle 2x POST yuboradi.
    // Real production'da event handler (router navigation)'da bo'lishi yaxshiroq.
  }, [product.id]);
  
  // Side effect: document.title
  useEffect(() => {
    document.title = displayName;
    return () => {
      document.title = 'Default';
    };
  }, [displayName]);
  
  return (
    <div>
      <h1>{displayName}</h1>
      <p>Price: ${product.price}</p>
      <p>Page views: {pageViewsRef.current}</p>
    </div>
  );
}
```

R19'da `<title>` JSX bilan:

```tsx
function ProductPage({ product }: { product: Product }): ReactElement {
  const displayName = product.name.toUpperCase();
  // ... tracking effect
  
  return (
    <>
      <title>{displayName}</title> {/* R19 Document Metadata */}
      <div>
        <h1>{displayName}</h1>
        {/* ... */}
      </div>
    </>
  );
}
```

</details>

### Mashq 3: Compiler Output Tahlili (O'rta)

Quyidagi source kod uchun Compiler-generated output'ni qo'lda yozing (oddiylashtirilgan format). Cache slot'lar va dependency'larni aniqlang.

```tsx
'use memo';

import type { ReactElement } from 'react';

interface User {
  id: string;
  firstName: string;
  lastName: string;
  age: number;
}

function UserCard({ user, theme }: { user: User; theme: 'light' | 'dark' }): ReactElement {
  const fullName = `${user.firstName} ${user.lastName}`;
  const ageGroup = user.age < 18 ? 'minor' : 'adult';
  const className = `card card-${theme} card-${ageGroup}`;
  
  return (
    <div className={className}>
      <h2>{fullName}</h2>
      <p>{ageGroup}</p>
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```javascript
import { c as _c } from 'react/compiler-runtime';

function UserCard(t0) {
  const $ = _c(11);
  const { user, theme } = t0;
  
  // Scope 1: fullName depends on user.firstName, user.lastName
  let t1;
  if ($[0] !== user.firstName || $[1] !== user.lastName) {
    t1 = `${user.firstName} ${user.lastName}`;
    $[0] = user.firstName;
    $[1] = user.lastName;
    $[2] = t1;
  } else {
    t1 = $[2];
  }
  const fullName = t1;
  
  // Scope 2: ageGroup depends on user.age
  let t2;
  if ($[3] !== user.age) {
    t2 = user.age < 18 ? 'minor' : 'adult';
    $[3] = user.age;
    $[4] = t2;
  } else {
    t2 = $[4];
  }
  const ageGroup = t2;
  
  // Scope 3: className depends on theme, ageGroup
  let t3;
  if ($[5] !== theme || $[6] !== ageGroup) {
    t3 = `card card-${theme} card-${ageGroup}`;
    $[5] = theme;
    $[6] = ageGroup;
    $[7] = t3;
  } else {
    t3 = $[7];
  }
  const className = t3;
  
  // Scope 4: JSX depends on className, fullName, ageGroup
  let t4;
  if ($[8] !== className || $[9] !== fullName || $[10] !== ageGroup) {
    t4 = (
      <div className={className}>
        <h2>{fullName}</h2>
        <p>{ageGroup}</p>
      </div>
    );
    $[8] = className;
    $[9] = fullName;
    $[10] = ageGroup;
    // Note: real Compiler'da output ham slot'da saqlanadi (qo'shimcha slot kerak)
  } else {
    t4 = /* output saqlash uchun qo'shimcha slot */;
  }
  
  return t4;
}
```

Eslatma: real Compiler output yanada oddiy, har scope output uchun alohida slot ajratadi. Bu yerdagi format konsept ko'rsatish uchun.

Jami: 4 ta reactive scope, ~11 cache slot.

Granularity benefit:
- `user.id` o'zgarsa — barcha scope'lar cached.
- `theme` o'zgarsa — Scope 3 va Scope 4 recompute, Scope 1 va 2 cached.
- `user.age` o'zgarsa — Scope 2, 3, 4 recompute, Scope 1 cached.

</details>

### Mashq 4: Compiler Bail-out Reason Topish (O'rta)

Quyidagi 5 ta komponentni tahlil qiling. Qaysilari Compiler bail-out sodir bo'ladi va sabab nima?

```tsx
// Komponent A
function A({ items }: { items: number[] }): ReactElement {
  const total = items.reduce((s, x) => s + x, 0);
  return <div>{total}</div>;
}

// Komponent B
function B({ items }: { items: number[] }): ReactElement {
  items.sort();
  return <div>{items.join(', ')}</div>;
}

// Komponent C
function C({ count }: { count: number }): ReactElement {
  if (count > 0) {
    const [doubled, setDoubled] = useState(count * 2);
  }
  return <div>{count}</div>;
}

// Komponent D
function D({ data }: { data: Data }): ReactElement {
  const ref = useRef<HTMLDivElement>(null);
  ref.current = document.querySelector('#target') as HTMLDivElement;
  return <div ref={ref}>{data.value}</div>;
}

// Komponent E
function E({ price }: { price: number }): ReactElement {
  const sorted = [...arr].sort();
  const formatted = `$${price.toFixed(2)}`;
  return <div>{formatted}</div>;
}
```

<details>
<summary><strong>Javob</strong></summary>

**Komponent A** — ✅ **Compile bo'ladi**
- Pure render
- `reduce` mutation qilmaydi (yangi value qaytaradi)
- Compiler total ni cache qiladi (deps: items)

**Komponent B** — ❌ **Bail-out**
- `items.sort()` props mutation
- Sabab: `Cannot mutate prop "items"`
- Yechim: `[...items].sort()`

**Komponent C** — ❌ **Bail-out + ESLint error**
- `useState` if blok ichida (conditional hook)
- Sabab: `Conditional hook call`
- Yechim: top-level useState + conditional render

**Komponent D** — ❌ **Bail-out**
- `ref.current = ...` render Phase'da
- Sabab: `Ref write during render`
- Yechim: `useEffect` ichida

**Komponent E** — ✅ **Compile bo'ladi**, lekin **Edge case**
- `arr` module-level (yoki global) — Compiler module-level read'ni "mutable external" deb taxmin qilishi mumkin
- Lekin agar `arr` const bo'lsa OK
- `[...arr].sort()` lokal copy — local mutation OK
- `formatted` pure
- Aslida Compiler "module-level read" ni warning bilan qabul qiladi (silent in non-strict mode)

</details>

### Mashq 5: Custom Hook + Compiler Integration (Qiyin)

`useDebounce` custom hook yarating. Compiler-friendly bo'lishi shart — Rules of React rioya, mutable closure yo'q, race condition yo'q. Test qilib Compiler bail-out bo'lishini tekshiring.

```tsx
function useDebounce<T>(value: T, delay: number): T {
  // TODO
}

// Foydalanish
'use memo';

function SearchBox(): ReactElement {
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
'use memo';

import { useState, useEffect } from 'react';
import type { ReactElement } from 'react';

function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);
  
  useEffect(() => {
    const id = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
    
    return () => {
      clearTimeout(id); // ✅ Cleanup symmetry (cross-ref 30 Invariant 3)
    };
  }, [value, delay]);
  
  return debouncedValue;
}

// Compiler-friendly:
// - Pure render: useState + useEffect, no side effects in body
// - No mutation: setDebouncedValue immutable update
// - No conditional hooks: top-level
// - Effect cleanup symmetry: setTimeout / clearTimeout pair
// - No external mutable reads: faqat value, delay props
// - Hook starts with 'use': Compiler identify qiladi

function SearchBox(): ReactElement {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);
  
  useEffect(() => {
    if (debouncedQuery) {
      fetch(`/api/search?q=${debouncedQuery}`);
    }
  }, [debouncedQuery]);
  
  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
    />
  );
}
```

`useDebouncedCallback` variant (function-based):

```tsx
'use memo';

import { useState, useEffect, useRef, useCallback } from 'react';

function useDebouncedCallback<Args extends unknown[]>(
  callback: (...args: Args) => void,
  delay: number
): (...args: Args) => void {
  const callbackRef = useRef(callback);
  const timeoutRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  
  // Latest callback ref pattern (cross-ref 30 stale closure pattern)
  useEffect(() => {
    callbackRef.current = callback;
  });
  
  // Cleanup unmount'da
  useEffect(() => {
    return () => {
      if (timeoutRef.current !== null) {
        clearTimeout(timeoutRef.current);
      }
    };
  }, []);
  
  // useCallback hali kerakli — Compiler ham respect qiladi
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

// Compiler avtomatik bu hook'ni ham memoize qiladi:
// - Hook signature 'use*' detected
// - Pure render
// - Refs faqat effect/event handler'larda yozilgan
// - Cleanup symmetry
```

Test (Compiler bail-out check):

```bash
npm run lint
# eslint-plugin-react-compiler natija:
# ✅ No violations in useDebounce.ts
# ✅ No violations in SearchBox.tsx
```

DevTools'da check:

```
React DevTools (Components tab):
   ⚛ SearchBox
     hooks:
       1. State (query: '')
       2. Debounce (custom hook)
         a. State (debouncedValue: '')
         b. Effect (setTimeout/clearTimeout)
       3. MemoCache [N]
       4. Effect (fetch)
```

`useMemoCache` Compiler-injected hook sifatida ko'rinadi.

</details>

---

## Xulosa

React Compiler — **build-time tool** bo'lib, source kod statik analiz natijasida automatic memoization qo'shadi. Bu fayl shu mexanizmni o'rgandi:

- **Compiler Concept** — build-time tool (avval React Forget), Babel plugin orqali AST → HIR → reactivity inference → cache slot allocation → memoized output. R19 bilan birga `babel-plugin-react-compiler@1.0.0` stable (2025 April).
- **Auto-Memoization** — `useMemo`/`useCallback`/`React.memo` kerak emas, Compiler komponent ichidagi har reactive scope alohida cache slot'da saqlaydi (granular per-property tracking).
- **Internal `_c` Array** — `useMemoCache(N)` chaqiriq build-time'da hisoblangan slot count'da array ajratadi, har scope `if ($[i] !== dep) { recompute; cache; } else { use cached; }` pattern.
- **Static Analysis** — HIR (SSA form, CFG), reactivity inference (props/state/context/hook returns reactive vs refs/module-level non-reactive), dataflow analysis, mutability inference, reactive scope grouping.
- **Rules of React** — komponent va hook'lar pure (mutation yo'q, side effect yo'q, top-level hooks, refs render-time yozilmaydi, effect cleanup symmetry). Bu qoidalar runtime invariants — Compiler build-time'da majburlaydi (cross-ref `30-concurrent-react.md`).
- **`eslint-plugin-react-compiler`** — kompilyatsiya'dan oldin violation'larni topish, ESLint pipeline integratsiyasi, CI/CD'ga qo'shish.
- **`babel-plugin-react-compiler`** — Vite/Webpack/Next.js Babel pipeline'iga qo'shiladi, opsiyalar (target, compilationMode, panicThreshold, sources).
- **Migration Path 6 Qadam** — ESLint plugin → violations fix → ESLint error mode → Babel plugin (annotation mode) → per-file opt-in (`'use memo'`) → manual memo'ni gradual olib tashlash. Yangi kod default Compiler-friendly.
- **Cheklovlar va Bail-out** — mutation, side effects, conditional hooks, ref render-time access, async hooks, complex closures. Bail-out qilingan komponent manual memoization'ga qaytadi.
- **Library Compatibility** — modern library'lar mos (React Router, TanStack Query, Zustand, Redux Toolkit), legacy library mutation'i bo'lsa effect boundary bilan izolatsiya.
- **Performance** — granular memoization, default-on optimization, single `useMemoCache` overhead `useMemo`+`useCallback`+`memo` chain'idan kichik. Bundle hajmi biroz kattaroq, runtime ko'pincha tezroq. Profile birinchi.
- **Future Paradigm** — manual memoization (`useMemo`/`useCallback`/`React.memo`) yo'q bo'lib ketadi (idiomatic emas), backward compatibility saqlanadi, kod clarity yaxshilanadi, Concurrent invariants bilan synergic.

QISM 8 (Performance & Compiler) **boshlanishi** — Compiler birinchi fayl. Keyingi fayllar — re-render triggers practical (`32-rendering-behavior.md`), optimization patterns manual qoladigan joylar (`33-optimization.md`), DevTools Profiler workflow (`34-profiling.md`), code splitting va preloading (`35-code-splitting.md`), virtualization (`36-virtualization.md`).

---

**Keyingi bo'lim:** [32-rendering-behavior.md](32-rendering-behavior.md) — Re-render trigger sabablari practical perspective'dan: state change, parent re-render top-down propagation, Context value change, Reconciliation va bailout (cross-ref 04), DevTools Profiler "Why did this render?" feature, React.memo bypass scenarios (Context dep, ref change, parent inline functions), stale closure scenarios useEffect deps.
