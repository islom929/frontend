# Bo'lim 12: State va useState

> State — komponent'ning ichki, render'lar bo'ylab saqlanadigan **xotirasi**. `useState` — function component'larda state'ni e'lon qilish va boshqarish uchun fundamental hook. Bu bo'lim 3 sub-bo'limda yoritiladi: Mental Model (State nima, State vs Props, Immutability invariant), useState API (initialization, functional updates, batching, immutable update patterns), va Under the Hood (Fiber state queue, linked list updates, mountState vs updateState mexanikasi).

---

## Mundarija

**(1) Mental Model:**
- [State Nima](#state-nima)
- [State vs Props](#state-vs-props)
- [Immutability Invariant](#immutability-invariant)

**(2) useState API va Updates:**
- [`useState` API Asoslari](#usestate-api-asoslari)
- [Initialization — Value vs Lazy Initial](#initialization--value-vs-lazy-initial)
- [Functional Updates va Closure Trap](#functional-updates-va-closure-trap)
- [Update Queueing va Batching](#update-queueing-va-batching)
- [Immutable Updates — Objects va Arrays](#immutable-updates--objects-va-arrays)
- [Immer Alternative](#immer-alternative)

**(3) Under the Hood:**
- [Fiber State Queue — Linked List](#fiber-state-queue--linked-list)
- [`mountState` vs `updateState`](#mountstate-vs-updatestate)
- [Bailout va `Object.is`](#bailout-va-objectis)

- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## State Nima

### Nazariya

State — Component'ning **ichki xotirasi**: render'lar bo'ylab saqlanib, har render paytida o'qiladigan, va o'zgarganda yangi render trigger qiladigan ma'lumot.

```tsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  // count — state qiymati
  // setCount — state'ni o'zgartirish funksiyasi
  
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}
```

**State 5 ta xususiyatga ega:**

1. **Persistent across renders** — komponent qayta render bo'lganda saqlanadi (initial value qaytmaydi)
2. **Per-instance** — har komponent instance'i o'z mustaqil state'iga ega
3. **Triggers re-render** — `setState` chaqirilsa, React komponent'ni qayta render qiladi
4. **Snapshot per render** — render ichida `state` qiymati o'zgarmaydi (immutable snapshot)
5. **Asynchronous updates** — `setState` chaqirilgandan keyin state darhol yangilanmaydi (keyingi render'da)

**Per-instance — ko'p instance bir komponent uchun:**

```tsx
function App() {
  return (
    <>
      <Counter />  {/* Instance 1 — count=0 dan boshlaydi */}
      <Counter />  {/* Instance 2 — mustaqil count */}
      <Counter />  {/* Instance 3 — mustaqil count */}
    </>
  );
}

// Har Counter o'z state'iga ega — biri click qilingani boshqasiga ta'sir qilmaydi
```

State Component'ning function tanasida e'lon qilingan oddiy local variable EMAS — chunki har render funksiya qayta chaqiriladi va local variable'lar har safar yangidan yaratiladi:

```tsx
// ❌ Local variable — state EMAS
function BadCounter() {
  let count = 0;  // Har render'da 0 ga reset
  
  const increment = () => {
    count = count + 1;  // Local variable o'zgartiriladi
    // Lekin re-render trigger yo'q — UI yangilanmaydi
  };
  
  return <button onClick={increment}>Count: {count}</button>;
  // Click qilinsa — count o'zgaradi, lekin UI 0 ko'rsatadi
}
```

`useState` orqali state — React'ning ichki saqlash mexanizmida (Fiber'ning `memoizedState`) yashaydi. Har render'da bir xil qiymat qaytadi, va `setState` chaqirilsa — React state'ni yangilaydi va re-render qiladi.

**Snapshot per render:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
    // Bu render'da count = 0 (snapshot)
    // Uchala chaqiruv: setCount(1), setCount(1), setCount(1) — bir xil qiymat
    // Queue: [1, 1, 1] — basicStateReducer(state, 1) = 1 har safar
    // Natija: keyingi render'da count = 1, NOT 3
  };
  
  return <button onClick={handleClick}>Count: {count}</button>;
}
```

`count` render ichida **immutable snapshot** — JS const variable (`useState` destructured local). Uni o'zgartirish mumkin emas. Faqat `setCount` keyingi render uchun yangi qiymat belgilaydi.

3 marta inkrement uchun functional update kerak (keyingi section'da batafsil):

```tsx
const handleClick = () => {
  setCount(c => c + 1);
  setCount(c => c + 1);
  setCount(c => c + 1);
  // Natija: count = 3
};
```

<details>
<summary><strong>Under the Hood</strong></summary>

`useState` Fiber'ning `memoizedState` linked list'ida saqlanadi (cross-ref [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md)).

**Fiber struktura (state perspective):**

```ts
{
  type: Counter,
  pendingProps: {},
  memoizedProps: {},
  memoizedState: {
    // Birinchi useState
    memoizedState: 0,         // joriy state qiymati
    queue: {
      pending: null,           // pending update'lar (circular linked list)
      lanes: 0,
      dispatch: setCountFn,    // setState reference
      // ...
    },
    next: null,                // keyingi hook (linked list)
  },
  // ...
}
```

**Mount paytida:**

```ts
function mountState<S>(initialState: S | (() => S)) {
  const hook = mountWorkInProgressHook();
  const computed = typeof initialState === 'function' 
    ? (initialState as () => S)() 
    : initialState;
  hook.memoizedState = computed;
  hook.baseState = computed;
  
  const queue = {
    pending: null,
    lanes: 0,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: computed,
  };
  hook.queue = queue;
  
  const dispatch = dispatchSetState.bind(null, currentlyRenderingFiber, queue);
  queue.dispatch = dispatch;
  
  return [computed, dispatch];
}
```

**Update paytida:**

```ts
function updateState<S>(initialState: S) {
  return updateReducer(basicStateReducer, initialState);
  // useState — useReducer'ning maxsus shakl, basicStateReducer bilan
}

function basicStateReducer<S>(state: S, action: S | ((s: S) => S)): S {
  return typeof action === 'function' ? (action as (s: S) => S)(state) : action;
}
```

**Per-instance state:**

Har Component instance — alohida Fiber. Har Fiber — alohida `memoizedState`. Shu sababli `<Counter />` ko'p marta render qilinsa, har biri o'z state'iga ega.

```
Component Tree:
  App
    Counter (Fiber 1) → memoizedState: { count: 5 }
    Counter (Fiber 2) → memoizedState: { count: 3 }
    Counter (Fiber 3) → memoizedState: { count: 0 }
```

**Snapshot per render:**

`useState` har render'da **bir xil qiymat** qaytaradi (chunki shu render davomida hook'lar dispatcher fixed). State faqat keyingi render boshlanganda yangilanadi:

```ts
// Render 1: count = 0
function Counter() {
  const [count, setCount] = useState(0); // 0 qaytaradi
  // count — ushbu render'ning const snapshot'i
  // ...
}

// setCount(5) chaqirilgach:
// React: yangi render trigger
// Render 2: count = 5
function Counter() {
  const [count, setCount] = useState(0); // initial e'tiborsiz, queue'dan 5 qaytaradi
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

State — render'lar bo'ylab saqlanadi:

```tsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>+</button>
      <button onClick={() => setCount(count - 1)}>-</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

Per-instance state — har Counter mustaqil:

```tsx
function MultiCounter() {
  return (
    <div>
      <h2>Counter 1</h2>
      <Counter />
      <h2>Counter 2</h2>
      <Counter />
      <h2>Counter 3</h2>
      <Counter />
    </div>
  );
  // Har Counter o'z mustaqil count'iga ega
  // Click qilinganda faqat shu instance'ning state o'zgaradi
}
```

Object state — kompleks ma'lumot:

```tsx
type User = { name: string; email: string; age: number };

function UserForm() {
  const [user, setUser] = useState<User>({
    name: '',
    email: '',
    age: 0,
  });
  
  return (
    <Stack gap={8}>
      <input
        value={user.name}
        onChange={(e) => setUser({ ...user, name: e.target.value })}
        placeholder="Name"
      />
      <input
        value={user.email}
        onChange={(e) => setUser({ ...user, email: e.target.value })}
        placeholder="Email"
      />
      <input
        type="number"
        value={user.age}
        onChange={(e) => setUser({ ...user, age: Number(e.target.value) })}
        placeholder="Age"
      />
    </Stack>
  );
}
```

Local variable — state EMAS:

```tsx
// ❌ Local variable, har render'da reset
function BadCounter() {
  let count = 0;
  return (
    <button onClick={() => count++}>
      {count}
    </button>
  );
  // Click — count o'zgaradi (memory'da), lekin UI render'da 0 qoladi
  // Bu — React'ning e'tiborga olmagan state
}

// ✅ useState — React'ning state mexanizmi
function GoodCounter() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount(c => c + 1)}>
      {count}
    </button>
  );
}
```

Snapshot per render — uchta setCount:

```tsx
function ButtonExample() {
  const [count, setCount] = useState(0);
  
  const handleClickWrong = () => {
    // ❌ Bir xil qiymatdan 3 marta inkrement
    setCount(count + 1);  // count = 0, set to 1
    setCount(count + 1);  // count hali ham 0, set to 1
    setCount(count + 1);  // count hali ham 0, set to 1
    // Natija: count = 1 (NOT 3)
  };
  
  const handleClickRight = () => {
    // ✅ Functional update — har safar latest state
    setCount(c => c + 1);  // c = 0, return 1
    setCount(c => c + 1);  // c = 1, return 2
    setCount(c => c + 1);  // c = 2, return 3
    // Natija: count = 3
  };
  
  return (
    <Inline gap={8}>
      <p>Count: {count}</p>
      <button onClick={handleClickWrong}>+1 wrong</button>
      <button onClick={handleClickRight}>+3 right</button>
    </Inline>
  );
}
```

</details>

---

## State vs Props

### Nazariya

State va Props — Component'larga ma'lumot taqdim etishning **ikkita asosiy mexanizmi**, lekin ular semantik va texnik jihatdan farq qiladi.

| Xususiyat | State | Props |
|-----------|-------|-------|
| **Manba** | Komponent ichidan (`useState`) | Parent'dan (atributlar) |
| **O'zgartirish** | `setState` orqali | Read-only (immutable) |
| **Render trigger** | `setState` ishga tushadi | Parent re-render bilan |
| **Egasi (owner)** | Komponent o'zi | Parent komponent |
| **Persistence** | Component lifecycle bo'ylab | Har render'da yangi |
| **Initial qiymat** | `useState(initial)` | Parent JSX'da uzatadi |
| **Tekshirish vaqti** | Runtime | Compile-time (TS bilan) |

**Tanlash strategiyasi:**

- **State** — bu komponent ichida **boshqariladigan** ma'lumot (input value, modal open/close, current tab)
- **Props** — bu komponent **tashqaridan oladigan** ma'lumot (user data, config, callbacks)

```tsx
type UserCardProps = {
  user: { name: string; email: string };  // ← prop (tashqaridan)
  onEdit: () => void;                       // ← prop callback
};

function UserCard({ user, onEdit }: UserCardProps) {
  const [expanded, setExpanded] = useState(false);  // ← state (ichki)
  
  return (
    <div>
      <h3>{user.name}</h3>
      {expanded && <p>{user.email}</p>}
      <button onClick={() => setExpanded(!expanded)}>
        {expanded ? 'Collapse' : 'Expand'}
      </button>
      <button onClick={onEdit}>Edit</button>
    </div>
  );
}
```

`user` va `onEdit` — props (parent'dan keladi). `expanded` — state (komponent o'zi boshqaradi).

**State'ni props'ga aylantirish — "lifting state up":**

Agar bir state ikki sibling'a kerak bo'lsa, parent'ga ko'tariladi (cross-ref [`14-lifting-and-controlled.md`](14-lifting-and-controlled.md)):

```tsx
// ❌ Sibling'lar mustaqil state'ga ega — sinxronizatsiya yo'q
function BadParent() {
  return (
    <>
      <ChildA />  {/* o'z count */}
      <ChildB />  {/* o'z count */}
    </>
  );
}

// ✅ State parent'ga lift qilinadi
function GoodParent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <ChildA count={count} onChange={setCount} />
      <ChildB count={count} />
    </>
  );
}
```

**Derived state — anti-pattern:**

State'dan boshqa state'ni hisoblash — ko'pchilik holatda anti-pattern. Render paytida derived qiymatni hisoblash afzal:

```tsx
// ❌ Derived state — duplikat ma'lumot
function BadList({ items }: { items: Item[] }) {
  const [filtered, setFiltered] = useState(items);
  const [search, setSearch] = useState('');
  
  // ❌ useEffect bilan sync — keraksiz
  useEffect(() => {
    setFiltered(items.filter((i) => i.name.includes(search)));
  }, [items, search]);
  
  return <ul>{filtered.map((i) => <li key={i.id}>{i.name}</li>)}</ul>;
}

// ✅ Derive during render — pure
function GoodList({ items }: { items: Item[] }) {
  const [search, setSearch] = useState('');
  const filtered = items.filter((i) => i.name.includes(search));
  // ✅ Pure derivation, har render'da hisoblanadi
  
  return <ul>{filtered.map((i) => <li key={i.id}>{i.name}</li>)}</ul>;
}
```

`useMemo` katta derivation'lar uchun (cross-ref [`21-usememo-usecallback.md`](21-usememo-usecallback.md)):

```tsx
function ExpensiveList({ items }: { items: Item[] }) {
  const [search, setSearch] = useState('');
  const filtered = useMemo(
    () => items.filter((i) => i.name.includes(search)).sort(),
    [items, search]
  );
  return <ul>{filtered.map((i) => <li key={i.id}>{i.name}</li>)}</ul>;
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Reconciler nuqtai-nazaridan State va Props farq:

**Props:** Fiber'ning `pendingProps` slot'ida (parent JSX'dan keladi):

```ts
fiber.pendingProps = { user, onEdit };
fiber.memoizedProps = { user, onEdit };  // commit'dan keyin
```

**State:** Fiber'ning `memoizedState` slot'idagi hooks linked list'da:

```ts
fiber.memoizedState = {
  // useState hook 1
  memoizedState: false,  // expanded
  queue: { pending: null, ... },
  next: null,  // boshqa hook'lar
};
```

**Re-render trigger sabablari:**

Komponent qachon re-render qiladi:

1. **State o'zgardi** — `setState` chaqirildi va yangi qiymat eskidan farq (Object.is)
2. **Props o'zgardi** — Parent re-render qildi va Element identity yangi
3. **Context o'zgardi** — `useContext` consumer (cross-ref [`19-usecontext.md`](19-usecontext.md))
4. **Force update** — class komponent yoki `useReducer` dispatch (state bilan bir xil semantika)

**State update lifecycle:**

```
1. setState(newValue) chaqiriladi
2. React: scheduleUpdateOnFiber(fiber, lane)
3. Update queue'ga update obyekt qo'shiladi
4. Reconciler: render scheduled (lane priority)
5. Render Phase: 
   - useState chaqiriladi
   - basicStateReducer queue'dan latest state hisoblaydi
   - Object.is(prev, next) ?
     - Equal → bailout (cross-ref 04-reconciliation.md)
     - Diff → render davom etadi
6. Commit Phase: DOM yangilanadi
```

**Element identity va props o'zgarishi:**

```tsx
function Parent() {
  const [tick, setTick] = useState(0);
  return <Child user={{ name: 'Alice' }} />;
}

function Child({ user }) { /* ... */ }
```

Parent har render'da yangi `user` object yaratadi → `user` reference yangi → Child uchun props yangi (Object.is === false). Child har Parent render'da re-render qiladi.

`useMemo` orqali stabilize qilish mumkin:

```tsx
const user = useMemo(() => ({ name: 'Alice' }), []);
// User reference stable
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

State + Props birga:

```tsx
type UserProfileProps = {
  userId: number;
};

function UserProfile({ userId }: UserProfileProps) {
  // Props — userId tashqaridan
  // State — UI ichki
  const [editing, setEditing] = useState(false);
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then(setUser);
  }, [userId]);
  
  if (!user) return <p>Loading...</p>;
  
  return (
    <div>
      <h2>{user.name}</h2>
      {editing ? (
        <UserEditForm user={user} onSave={setUser} onCancel={() => setEditing(false)} />
      ) : (
        <button onClick={() => setEditing(true)}>Edit</button>
      )}
    </div>
  );
}
```

Lifting state — controlled child:

```tsx
type ChildProps = {
  value: string;
  onChange: (value: string) => void;
};

function Child({ value, onChange }: ChildProps) {
  // value — prop (controlled — parent owns)
  // onChange — callback (child → parent)
  return (
    <input
      value={value}
      onChange={(e) => onChange(e.target.value)}
    />
  );
}

function Parent() {
  // State parent'da — bir source of truth
  const [name, setName] = useState('');
  
  return (
    <Stack gap={8}>
      <Child value={name} onChange={setName} />
      <p>You typed: {name}</p>
    </Stack>
  );
}
```

Derived state — pure render:

```tsx
type Product = { id: number; name: string; price: number };

function ProductList({ products }: { products: Product[] }) {
  const [search, setSearch] = useState('');
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name');
  
  // ✅ Derived — render paytida hisoblash
  const filtered = products
    .filter((p) => p.name.toLowerCase().includes(search.toLowerCase()))
    .sort((a, b) => {
      if (sortBy === 'name') return a.name.localeCompare(b.name);
      return a.price - b.price;
    });
  
  return (
    <Stack gap={12}>
      <input
        value={search}
        onChange={(e) => setSearch(e.target.value)}
        placeholder="Search..."
      />
      <select value={sortBy} onChange={(e) => setSortBy(e.target.value as 'name' | 'price')}>
        <option value="name">Sort by name</option>
        <option value="price">Sort by price</option>
      </select>
      <ul>
        {filtered.map((p) => <li key={p.id}>{p.name} — ${p.price}</li>)}
      </ul>
    </Stack>
  );
  // ✅ filtered ham search/sortBy/products o'zgarsa avtomatik qaytadan hisoblanadi
}
```

Anti-pattern — duplicated state:

```tsx
// ❌ Duplikat — props va state'da bir xil ma'lumot
function BadCard({ user }: { user: User }) {
  const [name, setName] = useState(user.name);
  // ❌ user.name o'zgarsa, state sync emas
  // useEffect bilan sync qilish — over-engineering
  
  return <h3>{name}</h3>;
}

// ✅ Faqat prop'dan o'qish
function GoodCard({ user }: { user: User }) {
  return <h3>{user.name}</h3>;
}

// ✅✅ Agar user.name'ni edit qilish kerak bo'lsa — controlled
function EditableCard({ user, onUpdate }: { user: User; onUpdate: (u: User) => void }) {
  const [draft, setDraft] = useState(user.name);
  
  return (
    <Stack gap={4}>
      <input value={draft} onChange={(e) => setDraft(e.target.value)} />
      <button onClick={() => onUpdate({ ...user, name: draft })}>Save</button>
    </Stack>
  );
  // draft — local edit state, save'da parent state yangilanadi
}
```

</details>

---

## Immutability Invariant

### Nazariya

State **immutable** — uni **mutate qilish taqiq**. Yangi state har doim **yangi reference** sifatida yaratiladi.

```tsx
const [items, setItems] = useState([1, 2, 3]);

// ❌ Mutation
items.push(4);
setItems(items);
// React: Object.is(items, items) === true → bailout, re-render yo'q

// ✅ Yangi reference
setItems([...items, 4]);
// React: Object.is(eski, yangi) === false → re-render
```

**Nima uchun React `Object.is` orqali tekshiradi?**

1. **Performance:** Shallow equality — O(1), deep equality — O(n). Har state update'ida O(1) check tezroq.
2. **Concurrent rendering:** State referensial taqqoslash — bailout aniq va deterministic. Deep equality — engine implementation'iga bog'liq, rezimlarda nomuvofiq.
3. **Functional purity:** Immutable updates funktsional dasturlash invariantlariga moslashadi. Har state — historical snapshot.

**`Object.is` vs `===`:**

```ts
Object.is(NaN, NaN);    // true (=== false)
Object.is(+0, -0);      // false (=== true)

// Boshqa har holatda Object.is === ===
Object.is(1, 1);        // true
Object.is({}, {});      // false (turli reference)
Object.is(null, null);  // true
```

React `Object.is`'ni shu sababli ishlatadi — `NaN` state'ni to'g'ri taqqoslash uchun.

**Immutable update patterns (qisqa preview, batafsil keyingi section'da):**

```tsx
// Object
const [user, setUser] = useState({ name: 'Alice', age: 30 });

setUser({ ...user, age: 31 });           // ✅ spread + override
setUser((u) => ({ ...u, age: u.age + 1 })); // ✅ functional + spread

// Array
const [items, setItems] = useState([1, 2, 3]);

setItems([...items, 4]);                  // ✅ append
setItems(items.filter((i) => i !== 2));   // ✅ remove
setItems(items.map((i) => i * 2));        // ✅ transform
```

**Mutation — sabablar:**

JavaScript'da object/array immutable EMAS. Push, splice, sort, reverse — **mutating method'lar**. Spread, slice, filter, map — **non-mutating** (yangi reference).

```ts
const arr = [1, 2, 3];
arr.push(4);  // arr o'zgardi (mutating)

const newArr = [...arr, 5];  // arr o'zgarmadi, newArr — yangi
```

**TypeScript bilan immutability enforce:**

```tsx
type Props = {
  items: readonly number[];  // yoki ReadonlyArray<number>
};

function List({ items }: Props) {
  items.push(4);  // ❌ TS error — push not on readonly array
  const newItems = [...items, 4];  // ✅ OK
}
```

`readonly` va `ReadonlyArray<T>` — TypeScript'ning lib.es5.d.ts ichida built-in (import kerak emas). Runtime'da Array hali ham o'zgartirilishi mumkin (type assertion bilan), lekin kod yozish paytida xato topiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`Object.is` semantikasi — ECMAScript spec'ning `SameValue` abstract operation'i (cross-ref `/js/03-equality-and-typing.md`). Polyfill ekvivalenti:

```ts
function objectIs(x: unknown, y: unknown): boolean {
  if (x === y) {
    // +0 vs -0 ajratish: 1/+0 = Infinity, 1/-0 = -Infinity
    return x !== 0 || 1 / (x as number) === 1 / (y as number);
  }
  // NaN vs NaN — NaN o'ziga teng bo'lmaydi (===), shu sababli ikkalasi ham NaN bo'lsa true
  return x !== x && y !== y;
}
```

**React state comparison (basicStateReducer):**

```ts
function basicStateReducer<S>(state: S, action: S | ((s: S) => S)): S {
  return typeof action === 'function' ? (action as (s: S) => S)(state) : action;
}
```

Reducer hisoblash, lekin equality check `processUpdateQueue`'da:

```ts
function dispatchSetState<S>(fiber: Fiber, queue: UpdateQueue<S>, action: S) {
  const update = createUpdate(action);
  
  // Eager bailout — faqat Fiber'da pending work yo'q bo'lganda xavfsiz
  if (
    fiber.lanes === NoLanes &&
    (fiber.alternate === null || fiber.alternate.lanes === NoLanes)
  ) {
    const lastRenderedState = queue.lastRenderedState;
    const eagerState = basicStateReducer(lastRenderedState, action);
    if (Object.is(eagerState, lastRenderedState)) {
      // Bailout — render trigger qilmaymiz
      return;
    }
  }
  
  enqueueUpdate(queue, update);
  scheduleUpdateOnFiber(fiber);
}
```

Agar yangi qiymat eski bilan `Object.is` teng va Fiber'da boshqa pending update yo'q — React render'ni umuman trigger qilmaydi. Bu — eager bailout (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — Bailout Sabab 4).

**Immutable update va shallow comparison chain:**

```tsx
const [user, setUser] = useState({ name: 'Alice', age: 30 });

setUser({ ...user, age: 31 });
// Yangi object — reference farqli
// React: Object.is(eski, yangi) === false → render
```

```tsx
setUser((u) => { u.age = 31; return u; });
// ❌ Mutation — reference bir xil
// React: Object.is(eski, yangi) === true → bailout
// UI yangilanmaydi, lekin u.age = 31 (data desync)
```

**Performance — Object.freeze va dev mode:**

React production'da state'ni freeze qilmaydi. Lekin development'da `Object.freeze` ishlatib, mutation aniqlanishi mumkin (3rd-party library'lar, Redux Toolkit `produce` ichida).

`Object.freeze` runtime cost — har property `configurable: false` o'rnatish va validation. Production'da bu overhead qabul qilinmaydigan, shu sababli React state'ni freeze qilmaydi.

**JavaScript'da immutable persistence (Immer):**

Immer — copy-on-write technique:

```ts
const next = produce(state, (draft) => {
  draft.user.age = 31;  // mutate-like API
});
// Immer: structural sharing — faqat o'zgargan path yangi reference
```

Internal'da Proxy bilan track qiladi. Output — yangi reference, lekin o'zgarmagan branch'lar — eski reference (memory efficient).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Anti-pattern — array mutation:

```tsx
function BadTodoList() {
  const [todos, setTodos] = useState<Todo[]>([
    { id: 1, text: 'Buy milk', done: false },
    { id: 2, text: 'Walk dog', done: false },
  ]);
  
  const toggleTodo = (id: number) => {
    // ❌ Mutation — reference bir xil
    const todo = todos.find((t) => t.id === id);
    if (todo) todo.done = !todo.done;
    setTodos(todos);
    // React: Object.is(todos, todos) === true → bailout
    // UI yangilanmaydi (data desync)
  };
  
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.done}
            onChange={() => toggleTodo(todo.id)}
          />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

✅ Immutable update:

```tsx
function GoodTodoList() {
  const [todos, setTodos] = useState<Todo[]>([
    { id: 1, text: 'Buy milk', done: false },
    { id: 2, text: 'Walk dog', done: false },
  ]);
  
  const toggleTodo = (id: number) => {
    setTodos(todos.map((todo) =>
      todo.id === id ? { ...todo, done: !todo.done } : todo
    ));
    // ✅ Yangi array, yangi todo object (faqat to'g'ri keladigan)
  };
  
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.done}
            onChange={() => toggleTodo(todo.id)}
          />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

`Object.is` quirk — `NaN`:

```tsx
function NumericInput() {
  const [num, setNum] = useState<number>(0);
  
  const handleChange = (val: string) => {
    setNum(Number(val));  // 'abc' → NaN
  };
  
  // Agar num = NaN bo'lsa, yana NaN set qilish — React Object.is(NaN, NaN) === true → bailout
  // (=== bo'lganda false bo'lardi va re-render trigger qilinardi — bug)
  
  return (
    <input
      value={String(num)}
      onChange={(e) => handleChange(e.target.value)}
    />
  );
}
```

TypeScript — readonly enforce:

```tsx
type State = Readonly<{
  user: Readonly<User>;
  items: ReadonlyArray<Item>;
}>;

const [state, setState] = useState<State>({
  user: { name: 'Alice' },
  items: [],
});

// ❌ TS errors:
state.user.name = 'Bob';     // Cannot assign to read-only property
state.items.push({...});      // Property 'push' does not exist
// ✅ Yangi reference
setState({
  ...state,
  user: { ...state.user, name: 'Bob' },
  items: [...state.items, newItem],
});
```

</details>

---

## `useState` API Asoslari

### Nazariya

`useState` — function component'larning fundamental Hook'i. State e'lon qilish va boshqarish uchun ishlatiladi.

```tsx
import { useState } from 'react';

const [value, setValue] = useState(initialValue);
//      ↑           ↑                ↑
//      |           |                initial value (faqat birinchi render'da)
//      |           setter function (yangi qiymat trigger)
//      current state qiymati
```

**Tuple return:**

`useState` — 2-element tuple qaytaradi:

1. **Current value** — joriy state qiymati
2. **Setter function** — state'ni yangilash funksiyasi

Destructuring odatda `[value, setValue]` pattern'i bilan, lekin nom istalgan bo'lishi mumkin (`[count, setCount]`, `[user, setUser]`, va h.k.).

**TypeScript generic:**

```tsx
const [count, setCount] = useState<number>(0);
const [user, setUser] = useState<User | null>(null);
const [items, setItems] = useState<Item[]>([]);
```

`useState<T>` — T tip parametri. Ko'pchilik holatda type inference avtomatik:

```tsx
const [count, setCount] = useState(0);
// TS infer: useState<number>

const [user, setUser] = useState({ name: 'Alice' });
// TS infer: useState<{ name: string }>

const [user, setUser] = useState<User | null>(null);
// Explicit — chunki initial null bo'lsa, T = null bo'ladi (kerak emas)
```

**Setter — ikki shakl:**

1. **Direct value** — `setValue(newValue)`
2. **Functional update** — `setValue((prev) => newValue)`

```tsx
const [count, setCount] = useState(0);

setCount(5);                    // Direct
setCount((prev) => prev + 1);   // Functional
```

Functional update — closure trap'dan qutulish va batched update'lar ichida latest state olish uchun (keyingi section'da batafsil).

**Setter type:**

```ts
type Dispatch<A> = (value: A) => void;

const [count, setCount]: [number, Dispatch<SetStateAction<number>>] = useState(0);

type SetStateAction<S> = S | ((prevState: S) => S);
```

`Dispatch<SetStateAction<S>>` — setter signature'i. Direct value yoki functional update qabul qiladi.

**Multiple state — alohida useState chaqiruvlari:**

```tsx
function UserForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [age, setAge] = useState(0);
  // ✅ Har state — alohida hook
}
```

Bir komponent ichida bir nechta `useState` chaqirilishi mumkin va tavsiya qilinadi (alohida state'lar — alohida hook). Single object state — ko'p maydonli form'lar uchun:

```tsx
function UserForm() {
  const [user, setUser] = useState({ name: '', email: '', age: 0 });
  
  return (
    <input
      value={user.name}
      onChange={(e) => setUser({ ...user, name: e.target.value })}
    />
  );
}
```

Tanlash konteksga bog'liq:
- **Mustaqil state'lar** — alohida `useState`
- **Bog'liq state'lar** (har doim birga o'zgaradi) — single object state yoki `useReducer`

<details>
<summary><strong>Under the Hood</strong></summary>

**`useState<T>(initialValue)` signature:**

```ts
function useState<T>(
  initialState: T | (() => T)
): [T, Dispatch<SetStateAction<T>>];
```

`Dispatch<SetStateAction<T>>` — setter union signature.

**Hooks dispatcher swap:**

`useState` aslida React'ning **dispatcher** orqali ishlaydi. Mount va update — turli implementation:

```ts
// react-reconciler/ReactFiberHooks.js (soddalashtirilgan)
let ReactCurrentDispatcher = {
  current: null
};

const HooksDispatcherOnMount = {
  useState: mountState,
  useEffect: mountEffect,
  // ...
};

const HooksDispatcherOnUpdate = {
  useState: updateState,
  useEffect: updateEffect,
  // ...
};

function renderWithHooks(...) {
  // Mount yoki update'ga qarab dispatcher set
  ReactCurrentDispatcher.current = current === null
    ? HooksDispatcherOnMount
    : HooksDispatcherOnUpdate;
  
  const children = Component(props);
  
  ReactCurrentDispatcher.current = ContextOnlyDispatcher;
  
  return children;
}
```

Hook'lar (`useState`, `useEffect`, ...) `ReactCurrentDispatcher.current[hookName]` orqali joriy implementation'ni topadi.

**`mountState` — birinchi render:**

```ts
function mountState<T>(
  initialState: T | (() => T)
): [T, Dispatch<SetStateAction<T>>] {
  const hook = mountWorkInProgressHook();  // Yangi hook obyekt yaratiladi
  
  const computed = typeof initialState === 'function'
    ? (initialState as () => T)()
    : initialState;
  
  hook.memoizedState = computed;
  hook.baseState = computed;
  
  const queue = {
    pending: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: computed,
  };
  hook.queue = queue;
  
  const dispatch = dispatchSetState.bind(null, currentlyRenderingFiber, queue);
  queue.dispatch = dispatch;
  
  return [computed, dispatch];
}
```

**Hooks linked list:**

Har Component Fiber'da hook'lar **linked list** orqali saqlanadi:

```
fiber.memoizedState 
  → hook1 (useState) 
    → hook2 (useState) 
      → hook3 (useEffect) 
        → null
```

`mountWorkInProgressHook` — yangi hook node yaratadi va list'ga ulayadi:

```ts
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };
  
  if (workInProgressHook === null) {
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    workInProgressHook.next = hook;
    workInProgressHook = hook;
  }
  return workInProgressHook;
}
```

**Rules of Hooks — pozitsiya muhim:**

Hook'lar har render'da **bir xil tartibda** chaqirilishi shart (cross-ref [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md) — Rules of Hooks). React linked list'ga pozitsiya bo'yicha murojaat qiladi:

```tsx
function Component() {
  const [a, setA] = useState(1);  // hook position 0
  const [b, setB] = useState(2);  // hook position 1
  const [c, setC] = useState(3);  // hook position 2
}
```

Render 1: a=1, b=2, c=3 hook list'da position 0, 1, 2.

Render 2 (conditional bo'lsa):

```tsx
if (cond) {
  const [a, setA] = useState(1);  // ❌ position skip qilinishi mumkin
}
const [b, setB] = useState(2);  // ❌ position 0 ga keladi
```

Hook list pozitsiyasi buziladi — React noto'g'ri state'ni qaytaradi. ESLint plugin (`eslint-plugin-react-hooks`) bu pattern'ni topa oladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Eng oddiy useState:

```tsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <Inline gap={8}>
      <button onClick={() => setCount(count + 1)}>+</button>
      <span>{count}</span>
      <button onClick={() => setCount(count - 1)}>-</button>
    </Inline>
  );
}
```

Multiple state — alohida hooks:

```tsx
function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [remember, setRemember] = useState(false);
  const [submitting, setSubmitting] = useState(false);
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setSubmitting(true);
    try {
      await login({ email, password, remember });
    } finally {
      setSubmitting(false);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input value={email} onChange={(e) => setEmail(e.target.value)} placeholder="Email" />
        <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} placeholder="Password" />
        <label>
          <input type="checkbox" checked={remember} onChange={(e) => setRemember(e.target.checked)} />
          Remember me
        </label>
        <button type="submit" disabled={submitting}>
          {submitting ? 'Logging in...' : 'Login'}
        </button>
      </Stack>
    </form>
  );
}
```

Single object state — bog'liq maydonlar:

```tsx
type FormData = {
  name: string;
  email: string;
  age: number;
  newsletter: boolean;
};

function ProfileForm() {
  const [form, setForm] = useState<FormData>({
    name: '',
    email: '',
    age: 0,
    newsletter: false,
  });
  
  const updateField = <K extends keyof FormData>(field: K, value: FormData[K]) => {
    setForm((prev) => ({ ...prev, [field]: value }));
  };
  
  return (
    <Stack gap={8}>
      <input
        value={form.name}
        onChange={(e) => updateField('name', e.target.value)}
        placeholder="Name"
      />
      <input
        value={form.email}
        onChange={(e) => updateField('email', e.target.value)}
        placeholder="Email"
      />
      <input
        type="number"
        value={form.age}
        onChange={(e) => updateField('age', Number(e.target.value))}
        placeholder="Age"
      />
      <label>
        <input
          type="checkbox"
          checked={form.newsletter}
          onChange={(e) => updateField('newsletter', e.target.checked)}
        />
        Subscribe to newsletter
      </label>
    </Stack>
  );
}
```

TypeScript generic — explicit:

```tsx
type Status = 'idle' | 'loading' | 'success' | 'error';

function StatusIndicator() {
  const [status, setStatus] = useState<Status>('idle');
  // Explicit — TS aniq Status union'ni biladi
  
  return (
    <div>
      Status: {status}
      <button onClick={() => setStatus('loading')}>Start</button>
      <button onClick={() => setStatus('success')}>Done</button>
      <button onClick={() => setStatus('error')}>Fail</button>
      <button onClick={() => setStatus('idle')}>Reset</button>
      
      {/* TS error:
      <button onClick={() => setStatus('unknown')}>X</button>
      */}
    </div>
  );
}
```

Nullable state:

```tsx
type User = { id: number; name: string };

function UserDetails() {
  const [user, setUser] = useState<User | null>(null);
  // Initial null — explicit type kerak (chunki TS infer null only)
  
  useEffect(() => {
    fetch('/api/me').then((r) => r.json()).then(setUser);
  }, []);
  
  if (!user) return <p>Loading...</p>;
  return <h1>{user.name}</h1>;
}
```

</details>

---

## Initialization — Value vs Lazy Initial

### Nazariya

`useState`'ga initial value 2 ta usul bilan uzatilishi mumkin:

1. **Value** — `useState(initialValue)` — qiymat
2. **Lazy initial** — `useState(() => initialValue)` — funksiya

```tsx
const [count, setCount] = useState(0);              // value
const [data, setData] = useState(() => parseData()); // lazy
```

**Farq:** Value har render'da hisoblanadi. Lazy faqat **birinchi render'da** chaqiriladi.

```tsx
function Bad() {
  // ❌ JSON.parse har render'da chaqiriladi (re-render'da ham!)
  const [data, setData] = useState(JSON.parse(largeJsonString));
  // Lekin natija e'tiborsiz qoldiriladi (chunki birinchi render'dan keyin state queue'dan olinadi)
  // Performance overhead — keraksiz parse
}

function Good() {
  // ✅ JSON.parse faqat birinchi render'da
  const [data, setData] = useState(() => JSON.parse(largeJsonString));
}
```

**Qachon lazy initial kerak:**

1. **Expensive computation** — `JSON.parse`, `Array(10000).fill(...)`, complex object creation
2. **Storage read** — `localStorage.getItem`, `sessionStorage.getItem`
3. **Browser API** — `window.innerWidth` (lekin SSR'da window undefined!)

```tsx
function ThemeProvider() {
  const [theme, setTheme] = useState<'light' | 'dark'>(() => {
    // ✅ Lazy — SSR-safe (function faqat client'da chaqiriladi initial render'da)
    if (typeof window === 'undefined') return 'light';
    return (localStorage.getItem('theme') as 'light' | 'dark') ?? 'light';
  });
  
  // ...
}
```

**Lazy initial — Pure function:**

Lazy initial **render purity** invariantiga bo'ysunadi (cross-ref [`09-component-basics.md`](09-component-basics.md)):

- Side effects yo'q (faqat read)
- Deterministic (har komponent instance uchun bir xil natija — agar input bir xil)
- Idempotent

```tsx
// ❌ Lazy initial'da side effect
const [count, setCount] = useState(() => {
  fetch('/api/init').then(/* ... */);  // ❌ side effect
  return 0;
});

// ✅ Side effect — useEffect ichida
const [count, setCount] = useState(0);
useEffect(() => {
  fetch('/api/init').then((data) => setCount(data.count));
}, []);
```

**Initial value — props'dan:**

```tsx
type Props = { initialCount: number };

function Counter({ initialCount }: Props) {
  const [count, setCount] = useState(initialCount);
  // ⚠️ initialCount o'zgarsa, state automatically yangilanmaydi
  // Chunki initial value faqat birinchi render'da o'qiladi
}

// Agar initialCount o'zgarganda state reset kerak bo'lsa — key trick:
function App() {
  const [userId, setUserId] = useState(1);
  return <Counter key={userId} initialCount={userId * 10} />;
  // userId o'zgarsa, Counter unmount/remount → state reset
}
```

`key` o'zgarishi → unmount + mount (cross-ref [`08-list-rendering.md`](08-list-rendering.md) — `key` va Komponent Identity).

<details>
<summary><strong>Under the Hood</strong></summary>

`mountState` initial value handling:

```ts
function mountState<T>(initialState: T | (() => T)): [...] {
  const hook = mountWorkInProgressHook();
  
  const computed = typeof initialState === 'function'
    ? (initialState as () => T)()  // Lazy — chaqiriladi
    : initialState;                 // Value — to'g'ridan-to'g'ri
  
  hook.memoizedState = computed;
  hook.baseState = computed;
  // ...
}
```

`typeof initialState === 'function'` — runtime check. Agar function bo'lsa — chaqiriladi, aks holda — qiymat sifatida saqlanadi.

**Update path — initial value e'tiborsiz:**

```ts
function updateState<T>(initialState: T): [...] {
  return updateReducer(basicStateReducer, initialState);
  //                                       ↑ initialState faqat reducer'ga uzatiladi (kerak emas)
  //                                       Lekin updateReducer hook.memoizedState'ni ishlatadi
}
```

`updateReducer` ichida `hook.memoizedState` (eski state) ishlatiladi — initialState e'tiborsiz qoldiriladi:

```ts
function updateReducer<S, I, A>(reducer: (s: S, a: A) => S, initialArg: I, init?: (i: I) => S): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  // initialArg, init — e'tiborsiz
  
  // queue'dan latest state hisoblanadi
  const newState = computeStateFromQueue(hook.memoizedState, queue);
  hook.memoizedState = newState;
  
  return [newState, dispatch];
}
```

**Lazy initial — performance demo:**

```tsx
function Bad() {
  console.log('initialState computed');
  const [val, setVal] = useState(JSON.parse(huge));
  // Har render'da: "initialState computed"
  // (lekin natija e'tiborsiz qoldiriladi)
}

function Good() {
  const [val, setVal] = useState(() => {
    console.log('lazy initial computed');
    return JSON.parse(huge);
  });
  // Faqat 1-render'da: "lazy initial computed"
}
```

**Strict Mode 2x render — lazy initial:**

```tsx
const [val, setVal] = useState(() => {
  console.log('lazy');
  return computeInitial();
});

// StrictMode dev mount:
// "lazy"
// "lazy"  ← 2 marta (intentional double-invoke)
// State qiymati — ikkinchi chaqiruvning natijasi ishlatiladi (birinchisi tashlanadi)
```

`StrictMode` komponenti dev mode'da function component'ni 2 marta render qiladi va lazy initializer ham 2 marta chaqiriladi — render purity invariantini (function bir xil input uchun bir xil output qaytarishi shart) tekshirish uchun. Bu — `StrictMode` o'rab olgan tree uchun. Production build'da — har biri 1 marta. Cross-ref [`09-component-basics.md`](09-component-basics.md) — Strict Mode.

**`useMemo` vs lazy initial:**

`useMemo` — har render'da deps tekshiriladi va cache'ga olinadi. Lazy initial — faqat birinchi render'da. Vazifaga qarab tanlash:

```tsx
// ✅ useState lazy — value bir marta hisoblanadi va Fiber memoizedState'da saqlanadi
const [items] = useState(() => parseExpensive());

// ✅ useMemo — deps o'zgarsa qaytadan hisoblash
const computed = useMemo(() => expensiveDerivation(items), [items]);
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

LocalStorage'dan lazy initial:

```tsx
import { useState, useEffect } from 'react';

function ThemeToggle() {
  const [theme, setTheme] = useState<'light' | 'dark'>(() => {
    // ✅ Lazy — birinchi render'da localStorage o'qiladi
    if (typeof window === 'undefined') return 'light';  // SSR fallback
    return (localStorage.getItem('theme') as 'light' | 'dark') ?? 'light';
  });
  
  useEffect(() => {
    document.documentElement.dataset.theme = theme;
    localStorage.setItem('theme', theme);
  }, [theme]);
  
  return (
    <button onClick={() => setTheme((t) => t === 'light' ? 'dark' : 'light')}>
      {theme === 'light' ? '🌙' : '☀️'}
    </button>
  );
}
```

Expensive parse:

```tsx
type Config = { theme: string; locale: string; features: string[] };

function App({ rawConfig }: { rawConfig: string }) {
  // ✅ JSON.parse faqat 1-render'da
  const [config, setConfig] = useState<Config>(() => JSON.parse(rawConfig));
  
  return (
    <Stack gap={8}>
      <h1>Theme: {config.theme}</h1>
      <h2>Locale: {config.locale}</h2>
      <ul>{config.features.map((f) => <li key={f}>{f}</li>)}</ul>
    </Stack>
  );
}
```

Default array generation:

```tsx
function Calendar({ year }: { year: number }) {
  const [days, setDays] = useState(() => {
    // ✅ Bir marta yaratiladi
    const result = [];
    for (let month = 0; month < 12; month++) {
      const daysInMonth = new Date(year, month + 1, 0).getDate();
      for (let day = 1; day <= daysInMonth; day++) {
        result.push({ year, month, day });
      }
    }
    return result;
  });
  
  return (
    <ul>
      {days.map((d) => (
        <li key={`${d.year}-${d.month}-${d.day}`}>
          {d.year}-{d.month + 1}-{d.day}
        </li>
      ))}
    </ul>
  );
}
```

Anti-pattern — value har render:

```tsx
function BadCalendar({ year }: { year: number }) {
  const [days, setDays] = useState(generateDays(year));
  // ❌ generateDays har render'da chaqiriladi
  // Birinchi render'da natija ishlatiladi, qolganlarda — natija tashlab yuboriladi
  // Lekin har safar generation cost
  
  return <ul>{days.map(/* ... */)}</ul>;
}

function generateDays(year: number) {
  const result = [];
  // ... heavy computation
  return result;
}
```

Initial value props'dan + key trick:

```tsx
type Props = { initialValue: number };

function Counter({ initialValue }: Props) {
  const [count, setCount] = useState(initialValue);
  
  return (
    <Stack gap={4}>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </Stack>
  );
}

function App() {
  const [resetKey, setResetKey] = useState(0);
  const [initial, setInitial] = useState(0);
  
  return (
    <Stack gap={8}>
      <Counter key={resetKey} initialValue={initial} />
      <input
        type="number"
        value={initial}
        onChange={(e) => setInitial(Number(e.target.value))}
      />
      <button onClick={() => setResetKey(k => k + 1)}>
        Reset Counter (with current initial)
      </button>
    </Stack>
  );
  // initial input o'zgarsa — Counter state o'zgarmaydi (initial faqat 1-render)
  // Reset button — key o'zgaradi → Counter remount → yangi initial bilan boshlanadi
}
```

</details>

---

## Functional Updates va Closure Trap

### Nazariya

**Functional update** — `setValue(prev => newValue)` pattern. Setter'ga **funksiya** uzatiladi, React uni keyingi render'da chaqirib, eng so'nggi state'ni uzatadi.

```tsx
const [count, setCount] = useState(0);

setCount(5);             // Direct: count = 5
setCount(c => c + 1);    // Functional: count = c + 1 (c — eng so'nggi)
```

**Closure trap muammosi:**

JavaScript closure — funksiya o'zining yaratilgan paytdagi scope'ni "ushlab qoladi". Har render — yangi funksiya, yangi closure.

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  
  const handleAsync = async () => {
    setCount(count + 1);   // count — render-time snapshot
    await delay(1000);     // user yana bosishi mumkin
    setCount(count + 1);   // count hali ham eski snapshot!
  };
  
  return <button onClick={handleAsync}>{count}</button>;
}
```

User 1-marta bosadi: `count = 0`. `handleAsync` boshlanadi, 1 sec kutadi.

User 1 sec ichida yana 5 marta bosadi: count = 5.

Eski `handleAsync` davom etadi: `setCount(count + 1)` chaqiradi, lekin **eski `count = 0`** snapshot ishlatiladi → `setCount(1)`. Latest state e'tiborsiz qoldiriladi.

**Functional update yechimi:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  
  const handleAsync = async () => {
    setCount(c => c + 1);  // c — chaqiruv paytidagi latest state
    await delay(1000);
    setCount(c => c + 1);  // c — yangilangan latest state
  };
  
  return <button onClick={handleAsync}>{count}</button>;
}
```

`c => c + 1` — React queue'ga function'ni qo'shadi. Keyingi render'da `c` parametri eng so'nggi state bilan chaqiriladi (closure'siz).

**Bir handler ichida ketma-ket setState'lar:**

```tsx
function MultiIncrement() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setCount(count + 1);  // count = 0, set 1
    setCount(count + 1);  // count = 0, set 1 (snapshot)
    setCount(count + 1);  // count = 0, set 1
    // Natija: count = 1 (3 emas!)
  };
  
  return <button onClick={handleClick}>{count}</button>;
}
```

Functional update bilan:

```tsx
const handleClick = () => {
  setCount(c => c + 1);  // queue: [c => c + 1]
  setCount(c => c + 1);  // queue: [c => c + 1, c => c + 1]
  setCount(c => c + 1);  // queue: [c => c + 1, c => c + 1, c => c + 1]
  // React: 0 → 1 → 2 → 3
  // Natija: count = 3
};
```

**Qachon functional update kerak:**

1. **Bir handler ichida ketma-ket setState'lar** (3 ta increment kabi)
2. **Async kontekst** — promise, setTimeout, fetch callback
3. **Event handler** — keyingi render bilan stale state olishi mumkin
4. **Custom hook'lar** — closure trap'ning eng keng tarqalgan manbasi

**Qachon direct value OK:**

1. **Sync kontekst, bir setState** — closure trap yo'q
2. **State boshqasiga bog'liq emas** — `setCount(5)` (boshqa state'ga ishlamaydi)
3. **Reset operatsiyalari** — `setCount(0)`, `setUser(null)`

```tsx
const reset = () => setCount(0);  // ✅ Direct OK
```

**TypeScript signature:**

```ts
type Dispatch<A> = (value: A) => void;
type SetStateAction<S> = S | ((prevState: S) => S);

const [count, setCount]: [number, Dispatch<SetStateAction<number>>] = useState(0);

setCount(5);                    // S
setCount((prev) => prev + 1);   // (prevState: S) => S
```

`SetStateAction<S>` — union, ikki signature qabul qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`dispatchSetState` internal:**

```ts
function dispatchSetState<S>(fiber: Fiber, queue: UpdateQueue<S>, action: S | ((s: S) => S)) {
  const lane = requestUpdateLane(fiber);
  const update: Update<S> = {
    lane,
    action,
    hasEagerState: false,
    eagerState: null,
    next: null,
  };
  
  // Eager bailout — agar yangi qiymat eski bilan teng
  if (queue.pending === null) {
    const lastRenderedState = queue.lastRenderedState;
    const eagerState = basicStateReducer(lastRenderedState, action);
    update.eagerState = eagerState;
    update.hasEagerState = true;
    
    if (Object.is(eagerState, lastRenderedState)) {
      // Bailout — render trigger qilmaymiz
      return;
    }
  }
  
  enqueueUpdate(fiber, queue, update, lane);
  scheduleUpdateOnFiber(fiber, lane);
}
```

**Update queue — circular linked list:**

```ts
queue.pending → updateA ⟶ updateB ⟶ updateC ⟶ updateA  (circular)
                                                  ↑
                                                last → first
```

Yangi update'lar oxiriga qo'shiladi (circular list — `pending` oxirgi node'ga ishora qiladi, `pending.next` esa birinchi node).

**Render Phase — `processUpdateQueue`:**

```ts
function updateReducer<S, A>(reducer: (s: S, a: A) => S, ...): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  
  let baseState = hook.baseState;
  let pending = queue.pending;
  
  if (pending !== null) {
    const first = pending.next;
    let update = first;
    
    do {
      // action — direct value yoki functional
      const action = update.action;
      baseState = reducer(baseState, action);
      // basicStateReducer: typeof action === 'function' ? action(state) : action
      
      update = update.next;
    } while (update !== first);
    
    queue.pending = null;
  }
  
  hook.memoizedState = baseState;
  return [baseState, queue.dispatch];
}
```

Update queue iterate qilinadi: har action `basicStateReducer` orqali apply qilinadi. Functional updates — `action(state)` chaqiriladi va natija keyingi action'ga uzatiladi.

**Eager bailout — direct value:**

```ts
setCount(5);
// dispatchSetState ichida:
// eagerState = basicStateReducer(0, 5) = 5
// Object.is(0, 5) === false → enqueue + schedule
```

```ts
setCount(0);  // eski 0 ga teng
// eagerState = 0
// Object.is(0, 0) === true → bailout (render trigger qilmaymiz)
```

**Functional update — eager bailout:**

```ts
setCount(c => c + 1);
// eagerState = basicStateReducer(0, c => c + 1) = 1
// Object.is(0, 1) === false → enqueue + schedule
```

Functional update'da function chaqiriladi va natija eski bilan tekshiriladi.

**Stale closure — pseudo-code:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);  // count = render-time snapshot
  
  // Yangi handleClick yaratiladi har render
  const handleClick = () => {
    setCount(count + 1);  // count — closure capture, render-time qiymat
  };
  
  return <button onClick={handleClick}>{count}</button>;
}
```

`handleClick` har render'da yangi function reference. U closure ichida shu render'ning `count` qiymatini "ushlab qoladi". Async kontekstda eski reference saqlansa — eski count ishlatiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Closure trap demo:

```tsx
function StaleClosureDemo() {
  const [count, setCount] = useState(0);
  
  const handleAsync = () => {
    setTimeout(() => {
      setCount(count + 1);
      // ❌ count — handleAsync chaqirilgan paytdagi snapshot
      // User 5 marta tez bossa: barcha timeout'lar count=0 dan 1 ga qaytaradi
    }, 1000);
  };
  
  return (
    <Inline gap={8}>
      <p>Count: {count}</p>
      <button onClick={handleAsync}>+1 (after 1s)</button>
    </Inline>
  );
}

// Functional update yechimi:
function FixedDemo() {
  const [count, setCount] = useState(0);
  
  const handleAsync = () => {
    setTimeout(() => {
      setCount(c => c + 1);  // ✅ c — latest state
      // 5 marta bosilsa — har timeout 1 ga oshiradi (5 ta increment)
    }, 1000);
  };
  
  return (
    <Inline gap={8}>
      <p>Count: {count}</p>
      <button onClick={handleAsync}>+1 (after 1s)</button>
    </Inline>
  );
}
```

Multi-increment:

```tsx
function MultiIncrement() {
  const [count, setCount] = useState(0);
  
  const incrementByThree = () => {
    setCount(c => c + 1);
    setCount(c => c + 1);
    setCount(c => c + 1);
    // Natija: +3 (har functional update queue ichida 1 ga oshiradi)
  };
  
  return (
    <button onClick={incrementByThree}>
      Count: {count} (+3 each click)
    </button>
  );
}
```

Async + await:

```tsx
async function fetchAndIncrement(setCount: (fn: (c: number) => number) => void) {
  await delay(1000);
  setCount(c => c + 1);
  await delay(1000);
  setCount(c => c + 1);
  await delay(1000);
  setCount(c => c + 1);
}

function App() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => fetchAndIncrement(setCount)}>
      {count}
    </button>
  );
  // Har 1 sek'da count 1 ga oshadi
  // Functional update'siz: hammasi count=0 dan 1 ga qaytarilardi (race condition)
}
```

Toggle pattern — direct value:

```tsx
function Toggle() {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <Stack gap={8}>
      {/* ❌ Bir tugma har bosishda toggle */}
      <button onClick={() => setIsOpen(!isOpen)}>
        {isOpen ? 'Hide' : 'Show'}
      </button>
      
      {/* ✅ Tezroq bosish bilan ishonchli */}
      <button onClick={() => setIsOpen(prev => !prev)}>
        {isOpen ? 'Hide' : 'Show'}
      </button>
    </Stack>
  );
  // Sync handler bilan ikkalasi ham ishlaydi
  // Lekin functional version — ehtiyotkor pattern (closure trap'ga qarshi)
}
```

useEffect dep'siz interval — closure trap classic:

```tsx
function ClosureTrapInterval() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);
      // ❌ count — useEffect chaqirilgan paytdagi snapshot (mount paytida 0)
      // Har 1 sek count = 1 (har doim)
    }, 1000);
    return () => clearInterval(id);
  }, []);  // [] dependency — effect 1 marta
  
  return <h1>{count}</h1>;
}

// ✅ Functional update — closure trap'ni hal qiladi
function FixedInterval() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + 1);  // ✅ Latest state
    }, 1000);
    return () => clearInterval(id);
  }, []);
  
  return <h1>{count}</h1>;
}
```

</details>

---

## Update Queueing va Batching

### Nazariya

**Batching** — bir vaqtda chaqirilgan setState'lar bitta render'ga birlashtiriladi. **Queueing** — setState'lar tartib bilan saqlanadi va ketma-ket apply qilinadi.

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');
  
  const handleClick = () => {
    setCount(c => c + 1);
    setName('Alice');
    // Bitta render — ikki state birga yangilanadi
  };
}
```

**Bir handler — bir render:**

React event handler ichidagi barcha setState'larni bir batch'ga to'playdi va bitta render trigger qiladi. Bu — performance optimization (har setState alohida render qilmaslik).

> **🕐 Versiya evolyutsiyasi (Automatic Batching):**
> - **R17 va undan oldin:** Faqat React event handler ichida batched. `setTimeout`, `Promise`, native event listener, async/await — har setState alohida render qilardi.
> - **R18+ (2022):** Automatic batching — barcha kontekstlarda batched (setTimeout, Promise, async, native listener).
> - **Sabab:** Concurrent rendering uchun zarur. Render uziluvchi bo'lgani sababli batching deterministic xatti-harakatni ta'minlaydi.

```tsx
// R18+ — barcha kontekstlarda batched
function R18Example() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);
  
  setTimeout(() => {
    setA(1);
    setB(2);
    // R17: 2 ta render
    // R18+: 1 ta render
  }, 100);
  
  fetch('/api').then(() => {
    setA(3);
    setB(4);
    // R17: 2 ta render
    // R18+: 1 ta render
  });
}
```

**`flushSync` — opt-out batching:**

Ba'zan batching xohlamaganida — masalan, scroll position'ni DOM yangilanishidan keyin o'qish kerak bo'lsa:

```tsx
import { flushSync } from 'react-dom';

function ScrollExample() {
  const [items, setItems] = useState<string[]>([]);
  const containerRef = useRef<HTMLDivElement>(null);
  
  const addAndScroll = () => {
    flushSync(() => {
      setItems((prev) => [...prev, `Item ${prev.length}`]);
    });
    // flushSync — DOM darhol yangilanadi (sync render)
    
    // Endi scroll qilamiz
    containerRef.current?.scrollTo({
      top: containerRef.current.scrollHeight,
      behavior: 'smooth',
    });
  };
  
  return (
    <Stack gap={8}>
      <button onClick={addAndScroll}>Add and Scroll</button>
      <div ref={containerRef} style={{ height: 200, overflow: 'auto' }}>
        {items.map((item, i) => <p key={i}>{item}</p>)}
      </div>
    </Stack>
  );
}
```

`flushSync` — performance cost (concurrent benefits yo'q). Faqat DOM API integration kerak bo'lganda ishlatilsin (cross-ref [`02-rendering.md`](02-rendering.md) — flushSync).

**Update queueing — functional updates ketma-ket:**

```tsx
const handleClick = () => {
  setCount(c => c + 1);
  setCount(c => c + 1);
  setCount(c => c + 1);
};

// Queue: [c => c + 1, c => c + 1, c => c + 1]
// Render Phase:
// state = 0
// 0 → c+1 → 1
// 1 → c+1 → 2
// 2 → c+1 → 3
// final: 3
```

**Mixed direct + functional:**

```tsx
const handleClick = () => {
  setCount(c => c + 1);  // queue: [c+1]
  setCount(10);           // queue: [c+1, 10]
  setCount(c => c + 1);   // queue: [c+1, 10, c+1]
};

// Render Phase:
// state = 0
// c+1 → 1
// 10 → 10 (override functional update)
// c+1 → 11
// final: 11
```

Direct value action — eski qiymatni override qiladi (`basicStateReducer(state, 10) = 10`, state e'tiborsiz).

<details>
<summary><strong>Under the Hood</strong></summary>

**R18+ Automatic Batching mexanizmi:**

R18 yangi `createRoot` API bilan keldi (cross-ref [`02-rendering.md`](02-rendering.md)). Internal'da scheduling Lanes model bilan ishlaydi (cross-ref [`05-scheduler-lanes.md`](05-scheduler-lanes.md)).

```ts
function dispatchSetState(fiber, queue, action) {
  const lane = requestUpdateLane(fiber);
  const update = createUpdate(action, lane);
  enqueueUpdate(fiber, queue, update, lane);
  scheduleUpdateOnFiber(fiber, lane);
}

function scheduleUpdateOnFiber(fiber, lane) {
  // Lane'ni Fiber tree'ga propagate qiladi
  markUpdateLaneFromFiberToRoot(fiber, lane);
  
  // Render Scheduler orqali planlanadi (bitta event loop tick ichida
  // barcha setState'lar yig'iladi, bitta render trigger qilinadi)
  ensureRootIsScheduled(root);
}

function ensureRootIsScheduled(root) {
  // Agar render allaqachon scheduled bo'lsa va bir xil priority — yangi schedule qilinmaydi
  if (root.callbackNode !== null && root.callbackPriority === lane) {
    return;
  }
  
  // Scheduler — internal cooperative scheduler, lane priority asosida
  scheduleCallback(schedulerPriorityFromLane(lane), performWorkOnRoot.bind(null, root));
}
```

**Sync vs scheduler:**

R18+'da automatic batching — event handler yoki async kontekstda barcha setState'lar bitta render'ga birlashtiriladi. Scheduler tick'da pending update'lar yig'iladi:

```ts
// Pseudo timeline R18+ event handler
event handler boshlanadi
  setCount(1)   → enqueue update, ensureRootIsScheduled (Sync lane)
  setName('A')  → enqueue update, already scheduled (skip)
event handler tugadi (executionContext bo'shaydi)
↓
flushSyncCallbacks — Sync lane darhol render qiladi
↓
performWorkOnRoot — render barcha pending updates bilan
↓
DOM yangilanadi
```

**`flushSync` — sync render trigger:**

```ts
function flushSync<T>(fn: () => T): T {
  const prevExecutionContext = executionContext;
  executionContext |= LegacyUnbatchedContext;
  
  try {
    const result = fn();
    
    // Pending update'larni darhol bajarish
    flushSyncCallbacks();
    
    return result;
  } finally {
    executionContext = prevExecutionContext;
    flushSyncCallbacks();
  }
}
```

Bu — concurrent rendering'ga zid (sync, blocking). Lekin DOM measure/manipulate uchun zarur.

**Queue processing — `processUpdateQueue`:**

```ts
function processUpdateQueue<S>(workInProgress: Fiber, props: any, instance: any): void {
  const queue = workInProgress.updateQueue;
  let firstBaseUpdate = queue.firstBaseUpdate;
  let lastBaseUpdate = queue.lastBaseUpdate;
  let pendingQueue = queue.shared.pending;
  
  // Pending queue'ni base'ga qo'shamiz
  if (pendingQueue !== null) {
    queue.shared.pending = null;
    // ... merge logic
  }
  
  if (firstBaseUpdate !== null) {
    let newState = queue.baseState;
    let update = firstBaseUpdate;
    
    do {
      const action = update.action;
      newState = getStateFromUpdate(workInProgress, queue, update, newState, props, instance);
      update = update.next;
    } while (update !== null);
    
    workInProgress.memoizedState = newState;
  }
}
```

Linked list iterate qilinadi, har update apply qilinadi, final state Fiber'ga commit qilinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Batch ichida ko'p state:

```tsx
function ProfileEditor() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [bio, setBio] = useState('');
  
  const reset = () => {
    setName('');
    setEmail('');
    setBio('');
    // R18+: Bitta render — uch state birga
  };
  
  return (
    <Stack gap={8}>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <input value={email} onChange={(e) => setEmail(e.target.value)} />
      <textarea value={bio} onChange={(e) => setBio(e.target.value)} />
      <button onClick={reset}>Reset</button>
    </Stack>
  );
}
```

Async batching (R18+):

```tsx
function AsyncBatch() {
  const [data, setData] = useState<Data | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  
  const fetchData = async () => {
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch('/api/data');
      const json = await response.json();
      
      // R18+: ikki state birga, bitta render
      setData(json);
      setLoading(false);
    } catch (err) {
      // R18+: ikki state birga
      setError(err as Error);
      setLoading(false);
    }
  };
  
  return (
    <Stack gap={8}>
      <button onClick={fetchData}>Load</button>
      {loading && <p>Loading...</p>}
      {error && <p>Error: {error.message}</p>}
      {data && <p>Loaded: {data.name}</p>}
    </Stack>
  );
}
```

flushSync — DOM measurement:

```tsx
import { useState, useRef } from 'react';
import { flushSync } from 'react-dom';

function ScrollOnAdd() {
  const [items, setItems] = useState<string[]>([]);
  const listRef = useRef<HTMLUListElement>(null);
  
  const addItem = () => {
    flushSync(() => {
      setItems((prev) => [...prev, `Item ${prev.length}`]);
    });
    // flushSync tugagandan keyin DOM yangilangan
    
    // Endi DOM'ni o'qiy olamiz
    listRef.current?.scrollTo({
      top: listRef.current.scrollHeight,
      behavior: 'smooth',
    });
  };
  
  return (
    <Stack gap={8}>
      <button onClick={addItem}>Add</button>
      <ul ref={listRef} style={{ height: 200, overflow: 'auto' }}>
        {items.map((item, i) => <li key={i}>{item}</li>)}
      </ul>
    </Stack>
  );
}
```

Mixed direct + functional:

```tsx
function MixedUpdates() {
  const [count, setCount] = useState(0);
  
  const handleMixed = () => {
    setCount(c => c + 1);  // queue: [c+1]
    setCount(10);           // queue: [c+1, 10] — direct override
    setCount(c => c + 1);   // queue: [c+1, 10, c+1]
    // Natija: count = 11
  };
  
  return (
    <button onClick={handleMixed}>
      Count: {count} (click to set 11)
    </button>
  );
}
```

Direct value bailout demo:

```tsx
function BailoutDemo() {
  const [count, setCount] = useState(0);
  const renderRef = useRef(0);
  renderRef.current++;
  
  return (
    <Stack gap={8}>
      <p>Render count: {renderRef.current}</p>
      <button onClick={() => setCount(0)}>setCount(0) — bailout</button>
      <button onClick={() => setCount(c => c)}>functional same — render trigger</button>
      {/* setCount(0) — eski 0 ga teng → bailout, render bo'lmaydi */}
      {/* setCount(c => c) — function chaqiriladi, eager state hisoblash, c=0 → bailout */}
    </Stack>
  );
}
```

</details>

---

## Immutable Updates — Objects va Arrays

### Nazariya

State Object yoki Array bo'lsa, immutable update'lar — yangi reference yaratish — kritik. Mutating method'lar (push, sort, splice) — TAQIQ.

**Object — spread + override:**

```tsx
const [user, setUser] = useState({ name: 'Alice', age: 30, email: 'a@x.com' });

// ✅ Spread + override
setUser({ ...user, age: 31 });

// ✅ Functional spread
setUser((u) => ({ ...u, age: u.age + 1 }));

// ❌ Mutation
user.age = 31; setUser(user);  // bailout
```

**Object — nested update:**

```tsx
const [profile, setProfile] = useState({
  user: { name: 'Alice', address: { city: 'Tashkent' } },
  settings: { theme: 'dark' },
});

// ✅ Nested spread
setProfile((p) => ({
  ...p,
  user: {
    ...p.user,
    address: {
      ...p.user.address,
      city: 'Samarkand',
    },
  },
}));

// ❌ Mutation
profile.user.address.city = 'Samarkand';
setProfile(profile);
```

Nested update — har level uchun spread. Murakkablik oshadi → Immer afzal (keyingi section).

**Array — common operations:**

```tsx
const [items, setItems] = useState<Item[]>([]);

// Append
setItems([...items, newItem]);
setItems((prev) => [...prev, newItem]);

// Prepend
setItems([newItem, ...items]);

// Insert at index
setItems([...items.slice(0, i), newItem, ...items.slice(i)]);

// Remove by id
setItems(items.filter((item) => item.id !== id));

// Update by id
setItems(items.map((item) => item.id === id ? { ...item, ...changes } : item));

// Sort (yangi array)
setItems([...items].sort((a, b) => a.name.localeCompare(b.name)));

// Reverse (yangi array)
setItems([...items].reverse());
```

**Array — JSON va structuredClone:**

Deep clone uchun:

```tsx
// Deep clone — modern browser/Node API (Node 17+, Chrome 98+, Firefox 94+, Safari 15.4+)
const cloned = structuredClone(items);

// Universal fallback — JSON
const cloned = JSON.parse(JSON.stringify(items));
// ⚠️ Cheklovlar:
//  - Function, undefined, Symbol — yo'qotiladi
//  - Date — ISO string'ga aylanadi (Date obyekt emas)
//  - Map, Set — bo'sh object {} ga aylanadi
//  - NaN, Infinity — null bo'lib qoladi
//  - Cyclic reference — TypeError
```

Deep clone — performance cost. Ko'p holatda **structural sharing** (faqat o'zgargan path yangi reference) afzal.

**Map / Set:**

```tsx
const [userMap, setUserMap] = useState<Map<number, User>>(new Map());

// ✅ Yangi Map
const newMap = new Map(userMap);
newMap.set(userId, newUser);
setUserMap(newMap);

// ❌ Mutation
userMap.set(userId, newUser);
setUserMap(userMap);  // Object.is === true → bailout
```

**Tuple — `as const`:**

```tsx
type Coords = readonly [number, number];

const [coords, setCoords] = useState<Coords>([0, 0]);

setCoords([5, 10]);  // ✅ yangi array
// coords[0] = 5;  // ❌ TS error — readonly
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Reference identity tekshiruv `Object.is`:**

```ts
function basicStateReducer<S>(state: S, action: S | ((s: S) => S)): S {
  return typeof action === 'function' ? action(state) : action;
}

// Bailout check (eager)
if (Object.is(eagerState, lastRenderedState)) {
  // No render
  return;
}
```

Object reference bir xil → bailout. Yangi object reference → render.

**Spread copy semantics:**

```ts
const obj = { a: 1, b: 2, nested: { x: 10 } };
const copy = { ...obj, b: 3 };

// copy.a = obj.a  (primitive — value copy)
// copy.b = 3      (override)
// copy.nested === obj.nested  // ⚠️ shallow copy — nested reference bir xil
```

Shallow copy — top-level property'lar copy qilinadi, nested object'lar reference orqali share qilinadi. Nested mutation hali ham buzg'unchi:

```ts
copy.nested.x = 99;  // obj.nested.x = 99 (chunki bir xil reference)
```

Nested update uchun har level spread:

```ts
const copy = { ...obj, nested: { ...obj.nested, x: 99 } };
// copy.nested — yangi object
// obj.nested.x — o'zgarmagan
```

**Array spread vs slice:**

```ts
const arr = [1, 2, 3];

const copy1 = [...arr];     // spread
const copy2 = arr.slice();   // slice (no args = full copy)
const copy3 = Array.from(arr);

// Hammasi yangi array, primitive value'lar copy qilingan
```

Array of objects — shallow copy:

```ts
const items = [{ a: 1 }, { a: 2 }];
const copy = [...items];

copy[0] === items[0];  // true (object reference share)

copy[0] = { a: 99 };   // copy o'zgardi (yangi reference index 0)
items[0].a = 99;       // ⚠️ items va copy ikkalasiga ham ta'sir qiladi
```

Object array'da har item'ni alohida copy qilish kerak (deep map):

```ts
const newItems = items.map((item) => ({ ...item }));
newItems[0].a = 99;  // ✅ items.a = 1 (mustaqil)
```

**`structuredClone` — Deep clone (HTML/Web API):**

```ts
const cloned = structuredClone(complexObject);
// Recursive clone, including Date, Map, Set, ArrayBuffer, RegExp
// Excludes: functions, DOM nodes, Error objects
```

HTML Living Standard'da definirlangan (Structured Clone Algorithm). React versiyasidan mustaqil — Node 17+, Chrome 98+, Firefox 94+, Safari 15.4+. React'ning o'zi ishlatmaydi, lekin user code'da deep clone kerak bo'lganda standart usul.

**Performance — shallow vs deep:**

| Operation | Time complexity |
|-----------|-----------------|
| Spread shallow | O(n) — n properties |
| Deep clone | O(total node count) — recursive |
| Structural sharing (Immer) | O(changed path depth) |

Yirik state'larda deep clone — sezilarli xarajat. Structural sharing — faqat o'zgargan branch'lar yangi reference, qolganlar share.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Object update — single field:

```tsx
type User = { id: number; name: string; email: string; age: number };

function UserEditor({ initialUser }: { initialUser: User }) {
  const [user, setUser] = useState(initialUser);
  
  const updateField = <K extends keyof User>(field: K, value: User[K]) => {
    setUser((u) => ({ ...u, [field]: value }));
  };
  
  return (
    <Stack gap={8}>
      <input
        value={user.name}
        onChange={(e) => updateField('name', e.target.value)}
        placeholder="Name"
      />
      <input
        value={user.email}
        onChange={(e) => updateField('email', e.target.value)}
        placeholder="Email"
      />
      <input
        type="number"
        value={user.age}
        onChange={(e) => updateField('age', Number(e.target.value))}
        placeholder="Age"
      />
    </Stack>
  );
}
```

Nested object update:

```tsx
type Address = { city: string; street: string; zip: string };
type Profile = {
  user: { name: string; email: string };
  address: Address;
};

function NestedEditor() {
  const [profile, setProfile] = useState<Profile>({
    user: { name: '', email: '' },
    address: { city: '', street: '', zip: '' },
  });
  
  const updateUser = <K extends keyof Profile['user']>(field: K, value: Profile['user'][K]) => {
    setProfile((p) => ({
      ...p,
      user: { ...p.user, [field]: value },
    }));
  };
  
  const updateAddress = <K extends keyof Address>(field: K, value: Address[K]) => {
    setProfile((p) => ({
      ...p,
      address: { ...p.address, [field]: value },
    }));
  };
  
  return (
    <Stack gap={16}>
      <fieldset>
        <legend>User</legend>
        <Stack gap={4}>
          <input
            value={profile.user.name}
            onChange={(e) => updateUser('name', e.target.value)}
            placeholder="Name"
          />
          <input
            value={profile.user.email}
            onChange={(e) => updateUser('email', e.target.value)}
            placeholder="Email"
          />
        </Stack>
      </fieldset>
      
      <fieldset>
        <legend>Address</legend>
        <Stack gap={4}>
          <input
            value={profile.address.city}
            onChange={(e) => updateAddress('city', e.target.value)}
            placeholder="City"
          />
          <input
            value={profile.address.street}
            onChange={(e) => updateAddress('street', e.target.value)}
            placeholder="Street"
          />
          <input
            value={profile.address.zip}
            onChange={(e) => updateAddress('zip', e.target.value)}
            placeholder="ZIP"
          />
        </Stack>
      </fieldset>
    </Stack>
  );
}
```

Array — CRUD operations:

```tsx
type Todo = { id: string; text: string; done: boolean };

function TodoApp() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [draft, setDraft] = useState('');
  
  // Add
  const addTodo = () => {
    if (!draft.trim()) return;
    setTodos((prev) => [
      ...prev,
      { id: crypto.randomUUID(), text: draft, done: false },
    ]);
    setDraft('');
  };
  
  // Update (toggle done)
  const toggleTodo = (id: string) => {
    setTodos((prev) =>
      prev.map((todo) =>
        todo.id === id ? { ...todo, done: !todo.done } : todo
      )
    );
  };
  
  // Delete
  const deleteTodo = (id: string) => {
    setTodos((prev) => prev.filter((todo) => todo.id !== id));
  };
  
  // Edit text
  const editTodo = (id: string, text: string) => {
    setTodos((prev) =>
      prev.map((todo) =>
        todo.id === id ? { ...todo, text } : todo
      )
    );
  };
  
  // Reorder (sort)
  const sortByText = () => {
    setTodos((prev) => [...prev].sort((a, b) => a.text.localeCompare(b.text)));
  };
  
  // Clear completed
  const clearCompleted = () => {
    setTodos((prev) => prev.filter((todo) => !todo.done));
  };
  
  return (
    <Stack gap={8}>
      <Inline gap={4}>
        <input
          value={draft}
          onChange={(e) => setDraft(e.target.value)}
          onKeyDown={(e) => e.key === 'Enter' && addTodo()}
          placeholder="What to do?"
        />
        <button onClick={addTodo}>Add</button>
      </Inline>
      
      <Inline gap={4}>
        <button onClick={sortByText}>Sort</button>
        <button onClick={clearCompleted}>Clear Completed</button>
      </Inline>
      
      <ul>
        {todos.map((todo) => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.done}
              onChange={() => toggleTodo(todo.id)}
            />
            <input
              value={todo.text}
              onChange={(e) => editTodo(todo.id, e.target.value)}
              style={{ textDecoration: todo.done ? 'line-through' : 'none' }}
            />
            <button onClick={() => deleteTodo(todo.id)}>×</button>
          </li>
        ))}
      </ul>
    </Stack>
  );
}
```

Map state:

```tsx
function UserCache() {
  const [cache, setCache] = useState<Map<number, User>>(new Map());
  
  const setUser = (user: User) => {
    setCache((prev) => {
      const next = new Map(prev);
      next.set(user.id, user);
      return next;
      // ✅ Yangi Map — reference farq
    });
  };
  
  const removeUser = (id: number) => {
    setCache((prev) => {
      const next = new Map(prev);
      next.delete(id);
      return next;
    });
  };
  
  const clearCache = () => {
    setCache(new Map());
  };
  
  // Anti-pattern (silent bug):
  const badSetUser = (user: User) => {
    cache.set(user.id, user);  // Mutation
    setCache(cache);            // Object.is(cache, cache) === true → bailout
  };
  
  return (
    <Stack gap={4}>
      <p>Cached users: {cache.size}</p>
      <button onClick={clearCache}>Clear</button>
    </Stack>
  );
}
```

</details>

---

## Immer Alternative

### Nazariya

Nested object update'lar uchun spread chain qiyin va xato'larga moyil:

```tsx
setProfile((p) => ({
  ...p,
  user: {
    ...p.user,
    address: {
      ...p.user.address,
      city: 'New',
    },
  },
}));
```

**Immer** — copy-on-write technique bilan immutable update'larni mutate-like sintaksis bilan yozish imkonini beradi:

```tsx
import { produce } from 'immer';

setProfile(produce(profile, (draft) => {
  draft.user.address.city = 'New';
  // mutate-like, lekin yangi reference qaytaradi
}));
```

**`useImmer` hook (use-immer kutubxona):**

```tsx
import { useImmer } from 'use-immer';

function NestedEditor() {
  const [profile, updateProfile] = useImmer<Profile>({
    user: { name: '', email: '' },
    address: { city: '', street: '', zip: '' },
  });
  
  const updateCity = (city: string) => {
    updateProfile((draft) => {
      draft.address.city = city;
      // ✅ Immer Proxy track qiladi va yangi reference yaratadi
    });
  };
  
  return (
    <input
      value={profile.address.city}
      onChange={(e) => updateCity(e.target.value)}
    />
  );
}
```

**Immer ishlash printsipi:**

1. `draft` — Proxy, asl object'ni o'rab oladi
2. Property access — Proxy track qiladi
3. Mutation — Proxy "draft state"'da yozadi
4. Final — yangi object yaratiladi (faqat o'zgargan path), o'zgarmagan branch'lar — eski reference (structural sharing)

**Foyda:**

- **Sintaksis sodda** — nested update mutate-like
- **Type-safe** — TS bilan yaxshi integration
- **Performance** — structural sharing (memory efficient)

**Cheklov:**

- **Bundle size** — Immer ~12-15KB (gzipped)
- **Learning curve** — Proxy semantikasi yangi paradigma
- **Some patterns ishlamaydi** — Map/Set, Set delete (Immer 6+ qo'llab-quvvatlaydi)

**Qachon Immer foydali:**

- Deeply nested state (3+ level)
- Ko'p sonli update operatsiyalari
- Redux Toolkit ishlatilayotgan loyiha (RTK ichida Immer mavjud)

**Qachon Immer kerak emas:**

- Flat state (1-2 level)
- Kichik state — spread oddiyroq
- Bundle size kritik (mobile, embed)

Bu — kursdan tashqari kutubxona, lekin awareness uchun mention. Asosiy texnika — vanilla immutable update.

<details>
<summary><strong>Under the Hood</strong></summary>

**Immer Proxy mexanizmi:**

```ts
import { produce } from 'immer';

const next = produce(state, (draft) => {
  draft.user.name = 'Bob';
});
```

Internal pseudo-code:

```ts
function produce(state, recipe) {
  const draft = createProxy(state, {
    get(target, prop) {
      if (prop in modified) return modified[prop];
      const value = target[prop];
      if (typeof value === 'object') {
        return createProxy(value, ...);  // recursive
      }
      return value;
    },
    set(target, prop, value) {
      modified[prop] = value;
      // Mark this path as changed
      return true;
    },
  });
  
  recipe(draft);
  
  // Build new object only for modified paths
  return finalizeStructure(state, modified);
}
```

**Structural sharing:**

```
Original state:
  {
    user: { name: 'Alice', email: 'a@x.com' },
    address: { city: 'Tashkent', street: 'X' },
    settings: { theme: 'dark' }
  }

After: draft.address.city = 'Samarkand'

Immer output:
  {
    user: <same reference>,           // unchanged
    address: { city: 'Samarkand', street: 'X' },  // new
    settings: <same reference>,       // unchanged
  }

Bu — partial new object. user va settings o'zgarmagan, eski reference bilan share.
```

**Performance comparison:**

```
Deeply nested update (5 level):

Vanilla spread:
  const next = { ...s, a: { ...s.a, b: { ...s.a.b, c: { ...s.a.b.c, d: 'x' } } } };
  Memory: 5 yangi object yaratiladi
  Manual coding overhead

Immer:
  produce(s, draft => { draft.a.b.c.d = 'x'; });
  Memory: 5 yangi object (structural sharing — lekin same logic)
  Coding: 1 line, mutate-like
```

Immer bilan yangi memory allocation deyarli bir xil — lekin coding ergonomics yaxshi.

**Auto-freeze:**

Immer dev mode'da output'ni `Object.freeze` qiladi (recursive). Mutation aniqlanadi:

```ts
const next = produce(state, (draft) => { draft.user.name = 'Bob'; });

next.user.name = 'X';  // ❌ TypeError: Cannot assign to read-only property
// Auto-freeze: kelajakdagi mutation'larga qarshi himoya
```

Production'da auto-freeze o'chirilishi mumkin (performance).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Immer with useState — vanilla:

```tsx
import { useState } from 'react';
import { produce } from 'immer';

type Profile = {
  user: { name: string; email: string };
  address: { city: string; street: string };
};

function ProfileEditor() {
  const [profile, setProfile] = useState<Profile>({
    user: { name: '', email: '' },
    address: { city: '', street: '' },
  });
  
  const updateUser = (field: keyof Profile['user'], value: string) => {
    setProfile(produce((draft) => {
      draft.user[field] = value;
    }));
  };
  
  const updateAddress = (field: keyof Profile['address'], value: string) => {
    setProfile(produce((draft) => {
      draft.address[field] = value;
    }));
  };
  
  return (
    <Stack gap={8}>
      <input
        value={profile.user.name}
        onChange={(e) => updateUser('name', e.target.value)}
        placeholder="Name"
      />
      <input
        value={profile.address.city}
        onChange={(e) => updateAddress('city', e.target.value)}
        placeholder="City"
      />
    </Stack>
  );
}
```

useImmer hook (use-immer):

```tsx
import { useImmer } from 'use-immer';

function TodoApp() {
  const [todos, updateTodos] = useImmer<Todo[]>([]);
  
  const addTodo = (text: string) => {
    updateTodos((draft) => {
      draft.push({ id: crypto.randomUUID(), text, done: false });
      // ✅ Immer Proxy: push mutate-like, lekin yangi array
    });
  };
  
  const toggleTodo = (id: string) => {
    updateTodos((draft) => {
      const todo = draft.find((t) => t.id === id);
      if (todo) todo.done = !todo.done;
      // ✅ Mutate, structural sharing
    });
  };
  
  const deleteTodo = (id: string) => {
    updateTodos((draft) => {
      const i = draft.findIndex((t) => t.id === id);
      if (i !== -1) draft.splice(i, 1);
    });
  };
  
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.done}
            onChange={() => toggleTodo(todo.id)}
          />
          {todo.text}
          <button onClick={() => deleteTodo(todo.id)}>×</button>
        </li>
      ))}
    </ul>
  );
}
```

Comparison — vanilla vs Immer:

```tsx
// Deeply nested update
type State = {
  app: {
    settings: {
      preferences: {
        theme: 'light' | 'dark';
        density: 'compact' | 'comfortable';
      };
    };
  };
};

// ❌ Vanilla — verbose
function updateThemeVanilla(state: State, theme: 'light' | 'dark'): State {
  return {
    ...state,
    app: {
      ...state.app,
      settings: {
        ...state.app.settings,
        preferences: {
          ...state.app.settings.preferences,
          theme,
        },
      },
    },
  };
}

// ✅ Immer — concise
function updateThemeImmer(state: State, theme: 'light' | 'dark'): State {
  return produce(state, (draft) => {
    draft.app.settings.preferences.theme = theme;
  });
}
```

</details>

---

## Fiber State Queue — Linked List

### Nazariya

State updates Fiber'ning `queue.pending` slot'idagi **circular linked list**'da saqlanadi. Render Phase'da bu list iterate qilinib, final state hisoblanadi.

**Update obyekti struktura:**

```ts
type Update<S> = {
  lane: Lane;             // priority lane (cross-ref 05)
  action: S | ((s: S) => S);  // direct value yoki functional
  hasEagerState: boolean; // bailout uchun
  eagerState: S | null;   // pre-computed
  next: Update<S> | null; // linked list pointer
};
```

**Circular linked list:**

```
queue.pending → updateC ⟶ updateA ⟶ updateB ⟶ updateC  (circular)
                  ↑                              ↓
                last.next = first ←──────────────┘
```

`pending` — **oxirgi** update'ga ishora qiladi. `pending.next` — birinchi. Bu — efficient append (O(1)) va iterate (O(n)).

**Nima uchun circular?**

Standart linked list — head pointer kerak. Circular — faqat bitta pointer (`pending`) yetadi va u har doim oxirgi node'ga ishora qiladi (`pending.next` — birinchi).

```ts
function enqueueUpdate<S>(queue: UpdateQueue<S>, update: Update<S>) {
  const pending = queue.pending;
  if (pending === null) {
    update.next = update;  // self-reference (1 element circular)
  } else {
    update.next = pending.next;  // yangi node birinchi node'ga
    pending.next = update;       // oxirgi node yangi node'ga
  }
  queue.pending = update;  // pending — oxirgi node
}
```

**Multiple state hooks — har biri o'z queue:**

```tsx
function Component() {
  const [a, setA] = useState(0);  // hook 0 — alohida queue
  const [b, setB] = useState(0);  // hook 1 — alohida queue
  const [c, setC] = useState(0);  // hook 2 — alohida queue
}
```

Har hook'ning `queue` slot'i mustaqil. `setA` faqat hook 0'ning queue'ga qo'shadi.

**Hooks linked list:**

```
fiber.memoizedState 
  → hook0 (useState a)  → memoizedState: 0, queue: {pending: ...}
    → hook1 (useState b) → memoizedState: 0, queue: {pending: ...}
      → hook2 (useState c) → memoizedState: 0, queue: {pending: ...}
        → null
```

**Lanes va update priority:**

Har update'ning `lane` field'i bor (cross-ref [`05-scheduler-lanes.md`](05-scheduler-lanes.md)):

```ts
update.lane = SyncLane;             // event handler — sync priority
update.lane = DefaultLane;          // useEffect — default priority
update.lane = TransitionLane1;      // useTransition — transition priority
```

Render Phase faqat **joriy lane'dagi update'larni** apply qiladi. Quyi priority update'lar keyingi render uchun saqlanadi (`baseQueue`).

<details>
<summary><strong>Under the Hood</strong></summary>

**`enqueueUpdate` — full code:**

```ts
function enqueueUpdate<S>(
  fiber: Fiber,
  queue: UpdateQueue<S>,
  update: Update<S>,
  lane: Lane
) {
  // queue.shared.pending — circular linked list
  const sharedQueue = queue.shared;
  const pending = sharedQueue.pending;
  
  if (pending === null) {
    // Birinchi update — self-reference (circular, 1 element)
    update.next = update;
  } else {
    // Yangi update'ni circular list'ga qo'shish
    update.next = pending.next;  // yangi → birinchi
    pending.next = update;        // eski oxirgi → yangi
  }
  
  sharedQueue.pending = update;  // pending — har doim oxirgi
  
  // Schedule render
  scheduleUpdateOnFiber(fiber, lane);
}
```

**`processUpdateQueue` — render paytida:**

```ts
function processUpdateQueue<S>(
  workInProgress: Fiber,
  queue: UpdateQueue<S>,
  hook: Hook,
  renderLanes: Lanes
): S {
  // Pending'ni base queue'ga ko'chiramiz
  let pendingQueue = queue.shared.pending;
  if (pendingQueue !== null) {
    queue.shared.pending = null;
    
    // Circular list'ni linear qilish (oxirgi → birinchi)
    const lastPendingUpdate = pendingQueue;
    const firstPendingUpdate = lastPendingUpdate.next;
    lastPendingUpdate.next = null;  // break circle
    
    // Append base queue
    if (hook.baseQueue === null) {
      hook.baseQueue = firstPendingUpdate;
    } else {
      hook.baseQueue.tail.next = firstPendingUpdate;
      hook.baseQueue.tail = lastPendingUpdate;
    }
  }
  
  // Iterate base queue va state hisoblash
  let newState = hook.baseState;
  let update: Update<S> | null = hook.baseQueue?.head ?? null;
  let newBaseState: S | null = null;
  let newBaseQueue: { head: Update<S>; tail: Update<S> } | null = null;
  
  while (update !== null) {
    const updateLane = update.lane;
    
    if (!isSubsetOfLanes(renderLanes, updateLane)) {
      // Bu update boshqa lane uchun — keyingi render'ga saqlash
      const clone: Update<S> = { ...update, next: null };
      if (newBaseQueue === null) {
        newBaseQueue = { head: clone, tail: clone };
        newBaseState = newState;
      } else {
        newBaseQueue.tail.next = clone;
        newBaseQueue.tail = clone;
      }
    } else {
      // Joriy lane — apply qilish
      if (newBaseQueue !== null) {
        // Boshqa lane skip qilingan bo'lsa, bu ham saqlash kerak (consistency)
        const clone: Update<S> = { ...update, next: null };
        newBaseQueue.tail.next = clone;
        newBaseQueue.tail = clone;
      }
      
      // Reducer apply
      const action = update.action;
      newState = basicStateReducer(newState, action);
    }
    
    update = update.next;
  }
  
  if (newBaseQueue === null) {
    newBaseState = newState;
  }
  
  hook.memoizedState = newState;
  hook.baseState = newBaseState;
  hook.baseQueue = newBaseQueue;
  
  return newState;
}
```

**Lane-based scheduling:**

Update'lar lane'ga qarab apply qilinadi. Misol:

```tsx
function App() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');
  
  // High priority (event handler)
  const handleClick = () => setCount(c => c + 1);
  
  // Low priority (useTransition)
  const handleSearch = (q: string) => {
    startTransition(() => {
      setText(q);
    });
  };
}
```

Concurrent rendering paytida:
- High priority render: `count` update'lari apply qilinadi
- Low priority render: `text` update'lari (transition lane) apply qilinadi

`baseQueue` — saqlangan low-priority update'lar.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Queue iteration demo:

```tsx
function QueueDemo() {
  const [count, setCount] = useState(0);
  const renderRef = useRef(0);
  renderRef.current++;
  
  const handleClick = () => {
    console.log('Before:', count);
    setCount(c => {
      console.log('Update 1:', c);
      return c + 1;
    });
    setCount(c => {
      console.log('Update 2:', c);
      return c + 1;
    });
    setCount(c => {
      console.log('Update 3:', c);
      return c + 1;
    });
    // 3 ta update queue'ga qo'shildi
    // Render Phase: queue iterate qilinadi
    // count = 0 → 1 → 2 → 3
  };
  
  return (
    <Stack gap={4}>
      <p>Count: {count}</p>
      <p>Renders: {renderRef.current}</p>
      <button onClick={handleClick}>+3</button>
    </Stack>
  );
  
  // Console output:
  // Before: 0
  // Update 1: 0
  // Update 2: 1
  // Update 3: 2
  // Render: count = 3 (renders: 2)
}
```

Lane-based — useTransition:

```tsx
import { useTransition, useState } from 'react';

function SearchableList({ items }: { items: Item[] }) {
  const [query, setQuery] = useState('');
  const [pending, startTransition] = useTransition();
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    
    // High priority — input darhol yangilanadi
    setQuery(value);
    
    // Low priority — transition lane
    startTransition(() => {
      // Heavy filter operation
      // Bu update transition lane'ga assign qilinadi
    });
  };
  
  return (
    <Stack gap={8}>
      <input value={query} onChange={handleChange} />
      {pending && <p>Filtering...</p>}
      <ItemList items={items.filter((i) => i.name.includes(query))} />
    </Stack>
  );
  // Cross-ref 22-concurrent-hooks.md — useTransition batafsil
}
```

Multiple state — alohida queue:

```tsx
function MultiState() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);
  
  const updateBoth = () => {
    setA(prev => prev + 1);  // hook 0 queue
    setB(prev => prev + 10); // hook 1 queue
    setA(prev => prev * 2);  // hook 0 queue
    
    // hook 0 queue: [+1, *2]
    // hook 1 queue: [+10]
    
    // Render Phase:
    // a: 0 → 1 → 2
    // b: 0 → 10
  };
  
  return (
    <Stack gap={4}>
      <p>A: {a}, B: {b}</p>
      <button onClick={updateBoth}>Update both</button>
    </Stack>
  );
}
```

</details>

---

## `mountState` vs `updateState`

### Nazariya

React state hook'lari **ikki implementation**'ga ega: mount paytida (`mountState`) va update paytida (`updateState`).

**Hooks dispatcher swap:**

Render boshlanganda dispatcher set qilinadi:

```ts
ReactCurrentDispatcher.current = current === null
  ? HooksDispatcherOnMount    // Birinchi render — mountState
  : HooksDispatcherOnUpdate;   // Keyingi render — updateState
```

**`mountState` — birinchi render:**

```ts
function mountState<S>(initialState: S | (() => S)): [S, Dispatch<SetStateAction<S>>] {
  // 1. Yangi hook obyekt yaratish
  const hook = mountWorkInProgressHook();
  
  // 2. Initial value hisoblash
  const computed = typeof initialState === 'function'
    ? (initialState as () => S)()
    : initialState;
  
  // 3. Saqlash
  hook.memoizedState = computed;
  hook.baseState = computed;
  
  // 4. Queue yaratish
  const queue = {
    pending: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: computed,
  };
  hook.queue = queue;
  
  // 5. Setter yaratish (bound function)
  const dispatch = dispatchSetState.bind(null, currentlyRenderingFiber, queue);
  queue.dispatch = dispatch;
  
  return [computed, dispatch];
}
```

**`updateState` — keyingi render'lar:**

```ts
function updateState<S>(initialState: S): [S, Dispatch<SetStateAction<S>>] {
  return updateReducer(basicStateReducer, initialState);
  //                                       ↑ initialState e'tiborsiz
}

function updateReducer<S, A>(reducer: (s: S, a: A) => S, ...): [S, Dispatch<A>] {
  // 1. Mavjud hook'ni topish (linked list traversal)
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  
  // 2. Pending updates'ni apply qilish
  let baseState = hook.baseState;
  let pending = queue.pending;
  
  if (pending !== null) {
    // Iterate va apply
    let update = pending.next;  // birinchi
    do {
      baseState = reducer(baseState, update.action);
      update = update.next;
    } while (update !== pending.next);
    
    queue.pending = null;
  }
  
  // 3. Memoize
  hook.memoizedState = baseState;
  
  return [baseState, queue.dispatch];
}
```

**Farq:**

| Aspect | `mountState` | `updateState` |
|--------|--------------|---------------|
| Initial value | Hisoblanadi va saqlanadi | E'tiborsiz qoldiriladi |
| Hook obyekt | Yaratiladi | Topiladi (linked list) |
| Queue | Yaratiladi | Mavjud queue process |
| Dispatch | Yangi function (bound) | Mavjud function reference |

**Re-render — `updateState` chaqiriladi:**

```tsx
function Component() {
  const [count, setCount] = useState(0);  // birinchi render: mountState
  // keyingi render'larda: updateState
  // initialState (0) — e'tiborsiz qoldiriladi
}
```

`useState(0)` — birinchi render'da 0 saqlaydi. Keyingi render'larda 0 e'tiborsiz, queue'dan latest state qaytariladi.

**Mount → unmount → mount — yangi state:**

```tsx
function Parent() {
  const [show, setShow] = useState(true);
  return (
    <>
      {show && <Counter />}  {/* show toggle qilsa, Counter unmount/mount */}
      <button onClick={() => setShow(s => !s)}>Toggle</button>
    </>
  );
}

// User click "Toggle":
// show: true → false → Counter unmount
// show: false → true → Counter MOUNT (mountState chaqiriladi yangi instance uchun)
// count: 0 (initial qiymat)
```

Unmount → mount — bu **yangi component instance**, yangi Fiber, yangi hook'lar list. State reset.

<details>
<summary><strong>Under the Hood</strong></summary>

**`mountWorkInProgressHook` — linked list build:**

```ts
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };
  
  if (workInProgressHook === null) {
    // Birinchi hook — linked list head
    currentlyRenderingFiber.memoizedState = hook;
    workInProgressHook = hook;
  } else {
    // Append linked list
    workInProgressHook.next = hook;
    workInProgressHook = hook;
  }
  
  return workInProgressHook;
}
```

**`updateWorkInProgressHook` — linked list traverse:**

```ts
function updateWorkInProgressHook(): Hook {
  let nextWorkInProgressHook;
  
  if (workInProgressHook === null) {
    // Birinchi hook — Fiber'ning memoizedState
    const current = currentlyRenderingFiber.alternate;
    if (current !== null) {
      nextWorkInProgressHook = current.memoizedState;
    }
  } else {
    nextWorkInProgressHook = workInProgressHook.next;
  }
  
  if (nextWorkInProgressHook !== null) {
    // Mavjud hook — clone qilish (workInProgress fiber uchun)
    const clone: Hook = {
      memoizedState: nextWorkInProgressHook.memoizedState,
      baseState: nextWorkInProgressHook.baseState,
      baseQueue: nextWorkInProgressHook.baseQueue,
      queue: nextWorkInProgressHook.queue,
      next: null,
    };
    
    workInProgressHook = workInProgressHook === null
      ? (currentlyRenderingFiber.memoizedState = clone)
      : (workInProgressHook.next = clone);
  } else {
    throw new Error('Rendered more hooks than during the previous render.');
    // ↑ Rules of Hooks violation — conditional hook
  }
  
  return workInProgressHook;
}
```

**Rules of Hooks — pozitsiya muhim:**

```tsx
function Bad({ cond }: { cond: boolean }) {
  if (cond) {
    const [a] = useState(1);  // Position 0 — faqat cond=true
  }
  const [b] = useState(2);    // Position 0 yoki 1 — depends on cond!
}
```

`updateWorkInProgressHook` linked list pozitsiyasiga ko'ra hook'larni topadi. Conditional hook — pozitsiya buziladi:

- Render 1 (cond=true): hook list = [a, b]
- Render 2 (cond=false): hook list = [b]
  - React: b'ning state = a'ning state? a'ning queue?
  - Linked list traversal pozitsiya 0 → b'ni topa olmaydi yoki noto'g'ri hook bilan eshlashtiradi
  - Throw: "Rendered fewer hooks than during the previous render"

**Hooks chain example:**

```tsx
function Component() {
  const [a, setA] = useState(1);
  const [b, setB] = useState(2);
  useEffect(() => {}, []);
  const [c, setC] = useState(3);
}

// Hook list (linked):
// hook0 (useState a) → hook1 (useState b) → hook2 (useEffect) → hook3 (useState c) → null
```

Update render'da:
```ts
updateState(initialState=1) — hook0 ni topadi
updateState(initialState=2) — hook1 ni topadi
updateEffect(...) — hook2 ni topadi
updateState(initialState=3) — hook3 ni topadi
```

Pozitsiya kafolat — linked list traversal `hook.next` orqali.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Initial value e'tiborsiz keyingi render'larda:

```tsx
function Counter({ initialCount }: { initialCount: number }) {
  const [count, setCount] = useState(initialCount);
  // initialCount faqat birinchi render'da o'qiladi
  // Keyingi render'larda — queue'dan state olinadi
  
  return (
    <Stack gap={4}>
      <p>Count: {count}</p>
      <p>Initial (prop): {initialCount}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </Stack>
  );
}

function App() {
  const [base, setBase] = useState(10);
  
  return (
    <Stack gap={8}>
      <input
        type="number"
        value={base}
        onChange={(e) => setBase(Number(e.target.value))}
      />
      <Counter initialCount={base} />
      {/* base o'zgarsa, Counter "initialCount" prop yangilanadi */}
      {/* Lekin Counter ichida count state — birinchi initialCount bilan saqlanadi */}
    </Stack>
  );
}
```

key trick — yangi mount uchun:

```tsx
function App() {
  const [base, setBase] = useState(10);
  
  return (
    <Stack gap={8}>
      <input
        type="number"
        value={base}
        onChange={(e) => setBase(Number(e.target.value))}
      />
      <Counter key={base} initialCount={base} />
      {/* key={base} — base o'zgarsa Counter unmount/mount */}
      {/* Yangi instance — mountState chaqiriladi, count = base */}
    </Stack>
  );
}
```

Conditional hook — Rules of Hooks violation:

```tsx
function Bad({ cond }: { cond: boolean }) {
  if (cond) {
    const [a] = useState(1);  // ❌ Position 0 — faqat cond=true
  }
  const [b] = useState(2);    // ❌ Position 0 yoki 1 — buziladi
  
  return <div>{b}</div>;
}

// ✅ To'g'ri — har doim har hook chaqiriladi
function Good({ cond }: { cond: boolean }) {
  const [a] = useState(1);   // Always position 0
  const [b] = useState(2);   // Always position 1
  
  return <div>{cond && a}, {b}</div>;
}

// ✅ Yoki — early return (lekin all hooks before return)
function Good2({ cond }: { cond: boolean }) {
  const [a] = useState(1);
  const [b] = useState(2);
  
  if (!cond) return null;
  return <div>{a}, {b}</div>;
}
```

ESLint plugin (`eslint-plugin-react-hooks`) `react-hooks/rules-of-hooks` qoidasi bu pattern'ni topadi.

</details>

---

## Bailout va `Object.is`

### Nazariya

**Bailout** — Reconciler render'ni "skip" qilish optimizatsiyasi (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — Bailout). State context'da: agar yangi state eski bilan `Object.is` teng bo'lsa, render trigger qilinmaydi.

**Eager bailout — `dispatchSetState` ichida:**

```ts
function dispatchSetState<S>(fiber: Fiber, queue: UpdateQueue<S>, action: S | ((s: S) => S)) {
  const lane = requestUpdateLane(fiber);
  const update: Update<S> = {
    lane,
    action,
    hasEagerState: false,
    eagerState: null,
    next: null,
  };
  
  // Pre-compute next state (faqat fiber'da hech qanday pending update yo'q bo'lsa)
  if (fiber.lanes === NoLanes && (fiber.alternate === null || fiber.alternate.lanes === NoLanes)) {
    const lastRenderedState = queue.lastRenderedState;
    const eagerState = basicStateReducer(lastRenderedState, action);
    
    update.hasEagerState = true;
    update.eagerState = eagerState;
    
    if (Object.is(eagerState, lastRenderedState)) {
      // Bailout — render trigger qilmaymiz
      return;
    }
  }
  
  enqueueUpdate(fiber, queue, update, lane);
  scheduleUpdateOnFiber(fiber, lane);
}
```

**Direct value bailout demo:**

```tsx
const [count, setCount] = useState(0);

setCount(0);  // Eager bailout: Object.is(0, 0) === true → no render
setCount(1);  // No bailout: Object.is(0, 1) === false → render
setCount(c => c);  // Functional, eager: c=0, return 0, Object.is(0, 0) → bailout
```

**Object/array — bailout deyarli ishlamaydi:**

```tsx
const [items, setItems] = useState([1, 2, 3]);

setItems([1, 2, 3]);  // Yangi array reference → no bailout, render
// (Lekin items value bir xil — Reconciler shallowEqual children re-render qiladi)

setItems(items);  // Bir xil reference → eager bailout
// (Lekin sizda eski items reference yo'q odatda — re-render uchun yangi yarataasiz)
```

Reference identity asosidagi bailout — performance optimization, lekin object/array state'da shubhali. `useMemo` bilan stabilize:

```tsx
const memoItems = useMemo(() => items, [items]);
// items reference o'zgarmasa, memoItems ham bir xil
```

**Bailout ikki turli joyda yuz beradi:**

1. **Eager bailout** (`dispatchSetState`): yangi state pre-compute qilinadi, eski bilan teng bo'lsa render umuman trigger qilinmaydi
2. **Late bailout** (Reconciliation): render trigger qilinadi, lekin Reconciler memoizedProps va memoizedState bir xil ekanligini topib children re-render qilmaydi

```tsx
const [count, setCount] = useState(0);

setCount(0);          // Eager bailout — render yo'q (Object.is(0, 0) === true)
setCount(c => c);     // Functional eager bailout — function chaqiriladi, c=0, return 0
                       // basicStateReducer(0, fn) === 0, Object.is(0, 0) === true → bailout
```

Functional update'da React function'ni eager hisoblaydi va natijani `lastRenderedState` bilan taqqoslaydi. Bir xil bo'lsa — bailout. Eager bailout faqat hech qanday pending update yo'q bo'lganda ishlaydi (`fiber.lanes === NoLanes`).

**Subtle case — non-deterministic functional update:**

```tsx
setCount(c => Math.random());  // Har chaqiruv yangi qiymat
// Eager bailout: function ikki marta chaqiriladi (eager + render Phase). Har safar
// yangi Math.random() — ikki natija farqli bo'lsa render commit boshqa qiymat ko'rsatadi.
// Bu — render purity invariantini buzadi: functional updater pure bo'lishi shart.
```

Functional updater'lar **pure** bo'lishi shart (cross-ref [`09-component-basics.md`](09-component-basics.md) — Render Purity). Side effects, random, Date.now — taqiqlanadi. Aks holda Strict Mode ham bu invariantni bezovta qiladi.

**Bailout chain — nested components:**

Parent render qilinsa, lekin child memoization qilingan bo'lsa va props bir xil — child render qilmaydi:

```tsx
const Child = memo(function Child({ value }: { value: number }) {
  return <p>{value}</p>;
});

function Parent() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);
  
  // a o'zgarsa, Parent re-render
  // Child uchun props (value=b) bir xil bo'lsa, memo bailout
}
```

Cross-ref [`33-optimization.md`](33-optimization.md) — `React.memo`.

<details>
<summary><strong>Under the Hood</strong></summary>

**Eager bailout flow:**

```ts
function dispatchSetState(fiber, queue, action) {
  const lane = requestUpdateLane(fiber);
  const update = createUpdate(lane, action);
  
  // Eager state computation
  if (
    fiber.lanes === NoLanes &&
    (alternate === null || alternate.lanes === NoLanes)
  ) {
    // Hech qanday pending update yo'q — eager hisoblash xavfsiz
    const lastRenderedState = queue.lastRenderedState;
    const eagerState = basicStateReducer(lastRenderedState, action);
    
    update.hasEagerState = true;
    update.eagerState = eagerState;
    
    if (Object.is(eagerState, lastRenderedState)) {
      // Bailout — return early
      return;
    }
  }
  
  enqueueUpdate(fiber, queue, update, lane);
  scheduleUpdateOnFiber(fiber, lane);
}
```

`fiber.lanes === NoLanes` — Fiber'da hech qanday pending work yo'q. Aks holda eager bailout xavfsiz emas (chunki keyingi update'lar pending state'ni o'zgartirishi mumkin).

**Late bailout — Reconciliation:**

```ts
function bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes) {
  if (current !== null) {
    // Pending work tekshirish
    if (!includesSomeLane(renderLanes, workInProgress.lanes)) {
      // No pending work — bailout
      return null;
    }
  }
  
  // Otherwise — proceed with render
  return cloneChildFibers(current, workInProgress);
}
```

Reconciler `current.memoizedProps === workInProgress.pendingProps` (Object.is) → bailout (cross-ref `04-reconciliation.md`).

**`Object.is` ECMAScript spec (SameValue, ECMA-262 §7.2.11):**

```
SameValue(x, y):
  1. If Type(x) is different from Type(y), return false.
  2. If Type(x) is Number or BigInt, return SameValueNonNumber/SameValueZero variants:
     - If x is NaN and y is NaN, return true.
     - If x is +0 and y is -0 (or vice versa), return false.
     - If x and y are the same Number value, return true.
     - Return false.
  3. If Type(x) is String, return true if same code unit sequence, else false.
  4. If Type(x) is Boolean/Null/Undefined/Symbol/Object — standard equality rules.
```

Tafovut'lar `===`'dan:
- `Object.is(NaN, NaN)` → true (`===` false)
- `Object.is(+0, -0)` → false (`===` true)
- Boshqa hamma joyda — bir xil

React `Object.is` ishlatadi chunki `NaN` state'ni to'g'ri taqqoslash kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Bailout demo — render counter:

```tsx
function BailoutDemo() {
  const [count, setCount] = useState(0);
  const renderRef = useRef(0);
  renderRef.current++;
  
  return (
    <Stack gap={4}>
      <p>Count: {count} | Renders: {renderRef.current}</p>
      
      <Inline gap={4}>
        <button onClick={() => setCount(0)}>setCount(0)</button>
        {/* Eager bailout — render bo'lmaydi */}
        
        <button onClick={() => setCount(c => c)}>setCount(c => c)</button>
        {/* Functional eager bailout */}
        
        <button onClick={() => setCount(count + 0)}>setCount(count + 0)</button>
        {/* Eager bailout (0 + 0 = 0) */}
        
        <button onClick={() => setCount(c => c + 1)}>+1</button>
        {/* Render */}
      </Inline>
    </Stack>
  );
}
```

NaN handling:

```tsx
function NumericInput() {
  const [num, setNum] = useState<number>(0);
  
  const handleChange = (val: string) => {
    setNum(Number(val));
    // 'abc' → NaN
    // setNum(NaN) chaqirilsa, eski NaN bo'lsa → Object.is(NaN, NaN) === true → bailout
    // === bo'lsa: NaN === NaN === false → render trigger (lekin value bir xil)
  };
  
  return (
    <input
      value={Number.isNaN(num) ? '' : String(num)}
      onChange={(e) => handleChange(e.target.value)}
    />
  );
  // React Object.is sabab — NaN state stable
}
```

Object reference — bailout YO'Q:

```tsx
function NoBailoutDemo() {
  const [config, setConfig] = useState({ theme: 'light' });
  const renderRef = useRef(0);
  renderRef.current++;
  
  return (
    <Stack gap={4}>
      <p>Theme: {config.theme} | Renders: {renderRef.current}</p>
      
      <button onClick={() => setConfig({ theme: 'light' })}>
        setConfig({"theme:'light'"}) — yangi obyekt
      </button>
      {/* Object.is({...}, {...}) === false — render trigger */}
      
      <button onClick={() => setConfig(config)}>
        setConfig(config) — bir xil reference
      </button>
      {/* Eager bailout */}
      
      <button onClick={() => setConfig((c) => c)}>
        setConfig(c => c) — functional same
      </button>
      {/* Eager bailout */}
    </Stack>
  );
}
```

`React.memo` chain:

```tsx
import { memo } from 'react';

const Child = memo(function Child({ value, label }: { value: number; label: string }) {
  console.log('Child render:', label);
  return <p>{label}: {value}</p>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const [unrelated, setUnrelated] = useState(0);
  
  return (
    <Stack gap={4}>
      <Child value={count} label="A" />
      <Child value={42} label="B" />
      
      <Inline gap={4}>
        <button onClick={() => setCount(c => c + 1)}>+ Count</button>
        <button onClick={() => setUnrelated(u => u + 1)}>+ Unrelated</button>
      </Inline>
    </Stack>
  );
  
  // setCount → Parent re-render
  //   Child A: value yangi → render
  //   Child B: value=42 stable → memo bailout
  
  // setUnrelated → Parent re-render
  //   Child A: value (count) stable → memo bailout
  //   Child B: value=42 stable → memo bailout
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: State'ni Render'da O'zgartirishga Urinish

```tsx
function Bad() {
  const [count, setCount] = useState(0);
  
  if (count === 0) {
    setCount(1);  // ⚠️ Render'da setState
  }
  
  return <div>{count}</div>;
}
```

React render paytida o'zining state'iga (yoki ancestor state'iga emas, faqat ushbu component) setState chaqirishni qabul qiladi: joriy render bekor qilinadi va yangi render boshlanadi (yangi state bilan). Ko'pchilik holatda bu pattern'dan qochish kerak — `useEffect` yoki initial value yoki memoization afzal (cross-ref [`09-component-basics.md`](09-component-basics.md) — Render Purity).

Cheklov: `setCount` faqat **conditional** chaqirilishi kerak (cheksiz loop oldini olish). Aks holda — `Too many re-renders. React limits the number of renders to prevent an infinite loop`.

---

### Gotcha 2: State Update Async — Read After Set

```tsx
const [count, setCount] = useState(0);

const handleClick = () => {
  setCount(5);
  console.log(count);  // ❌ Hali 0 (snapshot)
};
```

`setCount(5)` keyingi render uchun queue'ga qo'shadi, lekin joriy render'da `count` o'zgarmaydi (immutable snapshot).

Yangi qiymatni `useEffect` yoki keyingi render'da o'qish kerak.

---

### Gotcha 3: Initial State Object — Har Mount Yangi Reference

```tsx
function Component() {
  const [config, setConfig] = useState({ theme: 'light' });
  // {} har mount'da yangi reference
}
```

Ko'pchilik holatda muammo emas (har component instance alohida config). Lekin tashqi consumer'lar (parent passing config to child) — har Mount yangi reference.

---

### Gotcha 4: Functional Update Bilan State Reference Mutation

```tsx
const [items, setItems] = useState<Item[]>([]);

setItems((prev) => {
  prev.push(newItem);  // ❌ Mutation
  return prev;          // Bir xil reference → eager bailout
});

// ✅ To'g'ri
setItems((prev) => [...prev, newItem]);
```

Functional update parameter (`prev`) — joriy state reference. Mutation — array'ni o'zgartiradi, lekin React reference orqali bailout qiladi (Object.is === true).

---

### Gotcha 5: Conditional `useState`

```tsx
function Bad({ enabled }: { enabled: boolean }) {
  if (enabled) {
    const [count, setCount] = useState(0);  // ❌ Conditional hook
  }
}
```

Hook'lar har render'da bir xil tartibda chaqirilishi shart (Rules of Hooks). Conditional — pozitsiya o'zgaradi, hook list buziladi.

ESLint plugin `react-hooks/rules-of-hooks` bu pattern'ni topadi.

---

## Common Mistakes

### ❌ Xato 1: State Mutation

```tsx
// ❌
const [items, setItems] = useState([1, 2, 3]);
items.push(4);
setItems(items);  // bailout, no render

// ✅
setItems([...items, 4]);
```

**Sabab:** `Object.is(eski, yangi)` === true → bailout. Yangi reference yaratish kerak.

---

### ❌ Xato 2: Direct Value With Stale State

```tsx
// ❌ 3 marta increment
setCount(count + 1);
setCount(count + 1);
setCount(count + 1);
// count = 1 (snapshot bir xil)

// ✅
setCount(c => c + 1);
setCount(c => c + 1);
setCount(c => c + 1);
// count = 3
```

**Sabab:** Render snapshot — `count` immutable. Functional update queue ichida latest state.

---

### ❌ Xato 3: useState Initial — Expensive Computation

```tsx
// ❌ Har render'da JSON.parse
const [data, setData] = useState(JSON.parse(huge));

// ✅ Lazy initial
const [data, setData] = useState(() => JSON.parse(huge));
```

**Sabab:** Lazy initial faqat birinchi render'da chaqiriladi. Direct value — har render hisoblanadi (lekin natija e'tiborsiz qoldiriladi).

---

### ❌ Xato 4: Derived State — useState + useEffect

```tsx
// ❌ State'dan state hisoblash
const [items, setItems] = useState<Item[]>([]);
const [count, setCount] = useState(0);

useEffect(() => {
  setCount(items.length);
}, [items]);

// ✅ Derive during render
const [items, setItems] = useState<Item[]>([]);
const count = items.length;
```

**Sabab:** `useEffect` qo'shimcha render trigger qiladi. Pure derivation — tezroq, soddaroq.

---

### ❌ Xato 5: Conditional `useState`

```tsx
// ❌
function Component({ cond }: { cond: boolean }) {
  if (cond) {
    const [a] = useState(1);
  }
  const [b] = useState(2);
}

// ✅
function Component({ cond }: { cond: boolean }) {
  const [a] = useState(1);
  const [b] = useState(2);
  return cond ? <div>{a}, {b}</div> : <div>{b}</div>;
}
```

**Sabab:** Hooks pozitsiyasi linked list orqali eshlashtiriladi. Conditional — pozitsiya buziladi.

---

## Amaliy Mashqlar

### Mashq 1: Counter with Reset (Oson)

`Counter` komponent yarating: `count`, `+`, `-`, `Reset` (0 ga). Functional update ishlatish.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <Stack gap={8}>
      <p>Count: {count}</p>
      <Inline gap={4}>
        <button onClick={() => setCount(c => c + 1)}>+1</button>
        <button onClick={() => setCount(c => c - 1)}>-1</button>
        <button onClick={() => setCount(0)}>Reset</button>
      </Inline>
    </Stack>
  );
}
```

Functional update (`c => c + 1`) — closure trap'ga qarshi himoya. Reset — direct value (boshqa state'ga ishlamaydi, OK).

</details>

---

### Mashq 2: Lazy Initial — LocalStorage (Oson)

Theme toggle: `light` / `dark`, qiymat localStorage'da saqlanadi. Initial qiymat localStorage'dan o'qiladi (lazy).

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState, useEffect } from 'react';

function ThemeToggle() {
  const [theme, setTheme] = useState<'light' | 'dark'>(() => {
    if (typeof window === 'undefined') return 'light';
    const stored = localStorage.getItem('theme');
    return stored === 'dark' ? 'dark' : 'light';
  });
  
  useEffect(() => {
    document.documentElement.dataset.theme = theme;
    localStorage.setItem('theme', theme);
  }, [theme]);
  
  return (
    <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
      Theme: {theme}
    </button>
  );
}
```

Lazy initial — birinchi render'da localStorage o'qiladi. SSR fallback (`window === undefined`).

</details>

---

### Mashq 3: Closure Trap Fix (O'rta)

Quyidagi komponentdagi closure trap'ni toping va tuzating. User tugma bossa, 1 sekund'dan keyin count 1 ga oshishi kerak. Lekin tezda ko'p marta bossa, ularning soni count'ga qo'shilishi kerak.

```tsx
function ClosureTrap() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setTimeout(() => {
      setCount(count + 1);
    }, 1000);
  };
  
  return (
    <button onClick={handleClick}>
      Count: {count} (+1 after 1s)
    </button>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

**Bug:** `setCount(count + 1)` — `count` closure capture, handleClick chaqirilgan paytdagi snapshot. User 5 marta tezda bossa, har timeout'da `count = 0` (snapshot) — barchasi `setCount(1)` ishlaydi → final count = 1 (5 emas).

**Tuzatish:**

```tsx
function ClosureTrapFixed() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setTimeout(() => {
      setCount(c => c + 1);  // ✅ Functional update — latest state
    }, 1000);
  };
  
  return (
    <button onClick={handleClick}>
      Count: {count} (+1 after 1s)
    </button>
  );
}

// 5 marta tezda bosilsa:
// 5 ta timeout queue
// Har biri 1 sek'dan keyin: c => c + 1 (latest state ishlatadi)
// 0 → 1 → 2 → 3 → 4 → 5
```

`setCount(c => c + 1)` — React state queue'ga function'ni qo'shadi. Render Phase'da queue iterate qilinib, har function latest state bilan chaqiriladi.

</details>

---

### Mashq 4: Immutable Update (O'rta)

Todo list yarating: add, toggle done, edit text, delete, sort. Barcha update'lar immutable bo'lishi kerak.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState } from 'react';

type Todo = { id: string; text: string; done: boolean };

function TodoApp() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [draft, setDraft] = useState('');
  
  const addTodo = () => {
    if (!draft.trim()) return;
    setTodos((prev) => [
      ...prev,
      { id: crypto.randomUUID(), text: draft, done: false },
    ]);
    setDraft('');
  };
  
  const toggleDone = (id: string) => {
    setTodos((prev) =>
      prev.map((todo) =>
        todo.id === id ? { ...todo, done: !todo.done } : todo
      )
    );
  };
  
  const editText = (id: string, text: string) => {
    setTodos((prev) =>
      prev.map((todo) =>
        todo.id === id ? { ...todo, text } : todo
      )
    );
  };
  
  const deleteTodo = (id: string) => {
    setTodos((prev) => prev.filter((todo) => todo.id !== id));
  };
  
  const sortByText = () => {
    setTodos((prev) => [...prev].sort((a, b) => a.text.localeCompare(b.text)));
  };
  
  return (
    <Stack gap={8}>
      <Inline gap={4}>
        <input
          value={draft}
          onChange={(e) => setDraft(e.target.value)}
          onKeyDown={(e) => e.key === 'Enter' && addTodo()}
        />
        <button onClick={addTodo}>Add</button>
        <button onClick={sortByText}>Sort</button>
      </Inline>
      
      <ul>
        {todos.map((todo) => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.done}
              onChange={() => toggleDone(todo.id)}
            />
            <input
              value={todo.text}
              onChange={(e) => editText(todo.id, e.target.value)}
              style={{ textDecoration: todo.done ? 'line-through' : 'none' }}
            />
            <button onClick={() => deleteTodo(todo.id)}>×</button>
          </li>
        ))}
      </ul>
    </Stack>
  );
}
```

**Asosiy nuqtalar:**

1. **Add** — `[...prev, newItem]` (yangi array)
2. **Toggle/Edit** — `prev.map((t) => t.id === id ? { ...t, ...changes } : t)` (yangi array, yangi item)
3. **Delete** — `prev.filter((t) => t.id !== id)` (yangi array)
4. **Sort** — `[...prev].sort(...)` (copy + sort, original mutate qilinmaydi)

</details>

---

### Mashq 5: Custom Hook with Functional Updates (Qiyin)

`useCounter` custom hook yarating: `count` state, `increment`, `decrement`, `reset`, `setTo` action'lar. Functional update ishlatish (closure trap'siz). TypeScript bilan to'liq.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState, useCallback } from 'react';

type CounterActions = {
  increment: (by?: number) => void;
  decrement: (by?: number) => void;
  reset: () => void;
  setTo: (value: number) => void;
};

type UseCounterReturn = [number, CounterActions];

function useCounter(initialValue: number = 0): UseCounterReturn {
  const [count, setCount] = useState(initialValue);
  
  const increment = useCallback((by: number = 1) => {
    setCount((c) => c + by);
  }, []);
  
  const decrement = useCallback((by: number = 1) => {
    setCount((c) => c - by);
  }, []);
  
  const reset = useCallback(() => {
    setCount(initialValue);
  }, [initialValue]);
  
  const setTo = useCallback((value: number) => {
    setCount(value);
  }, []);
  
  return [count, { increment, decrement, reset, setTo }];
}

// Ishlatish:
function App() {
  const [count, { increment, decrement, reset, setTo }] = useCounter(10);
  
  return (
    <Stack gap={8}>
      <p>Count: {count}</p>
      <Inline gap={4}>
        <button onClick={() => increment()}>+1</button>
        <button onClick={() => increment(5)}>+5</button>
        <button onClick={() => decrement()}>-1</button>
        <button onClick={() => decrement(5)}>-5</button>
        <button onClick={() => setTo(0)}>0</button>
        <button onClick={() => setTo(100)}>100</button>
        <button onClick={reset}>Reset</button>
      </Inline>
    </Stack>
  );
}
```

**Asosiy nuqtalar:**

1. **Functional updates** (`(c) => c + by`) — async kontekstda ishonchli
2. **`useCallback`** — `actions` object stable references (parent re-render'da yangi function reference yo'q)
3. **`reset` deps** — `initialValue` o'zgarsa, callback yangi (kutilgan)
4. **TypeScript** — Tuple return type, action'lar object

Ishlatish JSX'da:
```tsx
const [count, { increment }] = useCounter();
// count: number, increment: function
```

Custom hook pattern — useState + useCallback birikmasi. Closure trap'siz, type-safe.

</details>

---

## Xulosa

**Mental Model:**
- **State** — komponent ichki xotirasi: persistent across renders, per-instance, triggers re-render, snapshot per render, async updates
- **State vs Props** — state ichki (komponent o'zi boshqaradi), props tashqaridan (parent), state'ga lift qilish — sibling sharing
- **Immutability invariant** — `Object.is` reference comparison, mutation Reconciler bailout buziladi (yangi reference shart)

**`useState` API:**
- `[value, setValue] = useState(initial)` — tuple return, generic `<T>`
- **Lazy initial** `useState(() => ...)` — birinchi render'da bir marta hisoblanadi (expensive computation, localStorage)
- **Functional update** `setValue(prev => ...)` — closure trap'ga qarshi yagona ishonchli yechim, async/batched kontekst
- **Update queueing** — bir handler ichida ketma-ket setState'lar queue'ga qo'shiladi va ketma-ket apply qilinadi
- **R18+ Automatic Batching** — barcha kontekstlarda (setTimeout, Promise, async) batched, R17 faqat React event handler
- **`flushSync`** — opt-out batching (DOM API integration uchun, performance cost)

**Immutable Updates:**
- **Object** — `{ ...obj, field: value }` (spread + override)
- **Array** — `[...arr, item]`, `arr.filter(...)`, `arr.map(...)`, `[...arr].sort(...)`
- **Nested** — har level uchun spread (yoki Immer)
- **Map/Set** — yangi instance (`new Map(prev)`)
- **Immer** — copy-on-write Proxy, mutate-like API, structural sharing (kursdan tashqari, awareness uchun)

**Under the Hood:**
- **Fiber state queue** — circular linked list, har hook'ning o'z queue'si
- **`mountState`** birinchi render'da (initial value hisoblash + queue yaratish), **`updateState`** keyingi render'larda (initialState e'tiborsiz, queue'dan latest state)
- **Hooks linked list** — pozitsiya muhim, conditional hook → Rules of Hooks violation
- **Bailout** — `Object.is(eski, yangi) === true` → render skip (eager dispatchSetState'da, late Reconciliation'da)

Keyingi bo'limda Event Handling — SyntheticEvent, event delegation history (R16 document → R17+ root container), event pooling olib tashlanishi (R17), R19 `<form action={fn}>`, va TypeScript event types yoritiladi.

---

**Keyingi bo'lim:** [13-event-handling.md](13-event-handling.md) — Event handlers (`onClick`, `onChange`, `onSubmit`), SyntheticEvent va cross-browser normalization, event delegation history (R16 vs R17+), event pooling deprecation (R17), R19 `<form action={fn}>` client-side actions integration, TypeScript event types (`MouseEvent<HTMLButtonElement>`, `ChangeEvent<HTMLInputElement>`).
