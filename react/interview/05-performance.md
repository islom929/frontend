# React Performance — Interview Savollari

> React.memo, re-render mechanics, useMemo/useCallback, React Compiler (R19), Profiling, Code splitting va Virtualization. Komponent darajasidagi performance optimizatsiyaning to'liq spektri.

---

## Mundarija

- [**QISM A: React.memo**](#qism-a) (savollar 1-4)
- [**QISM B: Re-render Mechanics**](#qism-b) (savollar 5-8)
- [**QISM C: useMemo / useCallback Usage**](#qism-c) (savollar 9-11)
- [**QISM D: React Compiler**](#qism-d) (savollar 12-14)
- [**QISM E: Profiling**](#qism-e) (savollar 15-17)
- [**QISM F: Code Splitting & Virtualization**](#qism-f) (savollar 18-22)
- [**QISM G: Web Vitals & Compiler Status**](#qism-g) (savollar 23-27)

**Jami:** 27 savol — Junior+ (4), Middle (8), Middle+ (8), Senior (7)



---

## QISM A: React.memo

<a id="qism-a"></a>

### 1. React.memo nima va u qanday shallow comparison bajaradi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`React.memo` — Higher-Order Component bo'lib, komponentni **shallow props comparison** bilan o'rab oladi. Yangi render'da props avvalgisi bilan **`Object.is`** orqali har bir top-level kalit bo'yicha solishtirilganda teng bo'lsa — komponent **bailout** qiladi (yangi render bajarilmaydi, oxirgi commit'dagi natija qayta ishlatiladi). Faqat referens-darajadagi solishtirish, chuqur tenglik tekshiruvi bajarilmaydi.

### To'liq tushuntirish

`React.memo` Reconciler'da Fiber'ning `tag` qiymatini `MemoComponent` (`14`) yoki `SimpleMemoComponent` (`15`) ga o'zgartiradi. Bu tag'lar uchun Reconciler render fazasida quyidagi qadamlarni bajaradi:

1. Yangi `pendingProps` va eski `memoizedProps` olinadi
2. `compare` funksiyasi chaqiriladi (default: `shallowEqual`)
3. Agar `true` qaytsa va `context` o'zgarmagan bo'lsa — `bailoutOnAlreadyFinishedWork` chaqiriladi
4. `bailout` natijasida child fiber'lar **yangidan render qilinmaydi**, faqat allaqachon mavjud sub-tree qayta ishlatiladi (lanes priority kengaytirish bilan)

**Shallow comparison qoidasi (`shallowEqual`):**

```typescript
function shallowEqual(objA: object, objB: object): boolean {
  if (Object.is(objA, objB)) return true;
  if (typeof objA !== "object" || objA === null) return false;
  if (typeof objB !== "object" || objB === null) return false;

  const keysA = Object.keys(objA);
  const keysB = Object.keys(objB);
  if (keysA.length !== keysB.length) return false;

  for (const key of keysA) {
    if (
      !Object.prototype.hasOwnProperty.call(objB, key) ||
      !Object.is(objA[key], objB[key])
    ) {
      return false;
    }
  }
  return true;
}
```

**Object.is vs ===:**

| Holat | `===` | `Object.is` |
|-------|-------|-------------|
| `NaN === NaN` | `false` | `true` |
| `+0 === -0` | `true` | `false` |
| Boshqa | bir xil | bir xil |

React `Object.is` ishlatadi — `NaN` props uchun stabil bailout va `+0/-0` uchun strict differentiation.

**`React.memo` bailout shartlari:**

- Props shallow equal
- Context value o'zgarmagan
- State (agar `useState` ishlatilgan bo'lsa) o'zgarmagan
- `forceUpdate` (class) chaqirilmagan

Agar barchasi qoniqarli bo'lsa — komponent qayta render qilinmaydi.

### Kod misoli

```tsx
import React, { memo } from "react";

interface UserCardProps {
  user: { id: string; name: string; email: string };
  onSelect: (id: string) => void;
}

// Default shallow comparison
const UserCard = memo(function UserCard({ user, onSelect }: UserCardProps) {
  console.log(`UserCard render: ${user.name}`);
  return (
    <div onClick={() => onSelect(user.id)}>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  );
});

// Parent
function UserList() {
  const [users, setUsers] = useState<User[]>([
    { id: "u1", name: "Ali", email: "ali@example.com" },
    { id: "u2", name: "Bek", email: "bek@example.com" },
  ]);
  const [filter, setFilter] = useState("");

  // ❌ Har render'da yangi function reference
  // memo bailout BUZILADI — har UserCard qayta render
  const handleSelect = (id: string) => {
    console.log(`Selected: ${id}`);
  };

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      {users.map((u) => (
        <UserCard key={u.id} user={u} onSelect={handleSelect} />
      ))}
    </>
  );
}
```

**Yechim — `useCallback`:**

```tsx
function UserList() {
  // ... users, filter ...

  // ✅ Stable reference across renders
  const handleSelect = useCallback((id: string) => {
    console.log(`Selected: ${id}`);
  }, []);

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      {users.map((u) => (
        <UserCard key={u.id} user={u} onSelect={handleSelect} />
      ))}
    </>
  );
}
// Filter typing'da UserCard'lar qayta render QILINMAYDI
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Reconciler ichidagi MemoComponent flow (soddalashtirilgan):**

```typescript
// react-reconciler/src/ReactFiberBeginWork.js (mental model)
function updateMemoComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes,
): Fiber | null {
  if (current === null) {
    // First mount — render normally
    const child = createFiberFromTypeAndProps(
      Component.type,
      null,
      nextProps,
      workInProgress,
      workInProgress.mode,
      renderLanes,
    );
    child.return = workInProgress;
    workInProgress.child = child;
    return child;
  }

  const currentChild = current.child as Fiber;
  const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
    current,
    renderLanes,
  );

  if (!hasScheduledUpdateOrContext) {
    const prevProps = currentChild.memoizedProps;
    let compare = Component.compare;
    compare = compare !== null ? compare : shallowEqual;

    if (compare(prevProps, nextProps) && current.ref === workInProgress.ref) {
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
  }

  // Render child normally
  const newChild = createWorkInProgress(currentChild, nextProps);
  newChild.return = workInProgress;
  workInProgress.child = newChild;
  return newChild;
}
```

**`bailoutOnAlreadyFinishedWork` nima qiladi:**

```typescript
function bailoutOnAlreadyFinishedWork(
  current: Fiber,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  // Children'da pending update bormi?
  if (!includesSomeLane(workInProgress.childLanes, renderLanes)) {
    // Yo'q — to'liq sub-tree skip
    return null;
  }
  // Ha — child'larni clone qilib davom etish
  cloneChildFibers(current, workInProgress);
  return workInProgress.child;
}
```

**Tag farqlari:**

```typescript
// react-reconciler/src/ReactWorkTags.js (selected)
export const FunctionComponent = 0;
export const MemoComponent = 14;          // memo() with custom compare
export const SimpleMemoComponent = 15;    // memo() with default compare (shallowEqual)
```

**SimpleMemoComponent — optimizatsiya:**

Custom `compare` funksiya berilmagan bo'lsa, React `SimpleMemoComponent` tag'ini ishlatadi. Bu tag'da Reconciler `MemoComponent` flow'ini chetlab o'tib, to'g'ridan-to'g'ri shallow check + bailout qiladi (ortiqcha indirection yo'q).

**Performance — shallowEqual murakkabligi:**

`shallowEqual` O(n) — props key'lar soni bo'yicha. Tipik komponent ~5-10 props → mikrosaniyalarda. Lekin bu solishtirish har **render** da bajariladi (parent render bo'lganda) — minglab komponentlar uchun cumulative cost sezilarli bo'lishi mumkin.

> **Performance note:** `React.memo` o'zi qo'shimcha overhead — har render'da shallow check + tag dispatch. Agar komponent kichkina bo'lsa (oddiy `<span>`) — `memo` qo'yish **zarar** keltirishi mumkin (memoization cost > render cost). Profiler bilan o'lchang.

</details>

### Edge Cases

- **Props ichida object literal**: `<Component config={{ size: 10 }} />` — har render'da yangi reference. Bailout buziladi. `useMemo` bilan stabilize qilish kerak.
- **Children prop**: `<Memoized><Child /></Memoized>` — `children` har JSX tree'da yangi React element. Element'ning `type` va `props` `Object.is` bilan teng emas — bailout buziladi.
- **`forwardRef` bilan ishlash**: `memo(forwardRef(...))` — `ref` `compare`'da tekshiriladi (alohida solishtirish), props bo'lib hisoblanmaydi.

### Follow-up savollar

- "Agar komponent state'i bor bo'lsa, memo state o'zgarganda nima qiladi?" — State o'zgarsa, komponent **scheduled update** bor — `checkScheduledUpdateOrContext` `true` qaytaradi, `compare` chaqirilmaydi, normal render bajariladi.
- "memo Fiber tree'da qancha joy oladi?" — `SimpleMemoComponent` (custom compare yo'q) uchun alohida Fiber wrapper yaratilmaydi — mavjud Fiber'ning `tag`i o'zgaradi. `MemoComponent` (custom compare bilan) uchun bitta qo'shimcha wrapper Fiber bo'ladi (child'ni o'z ichiga oladi).
- "Class component uchun `memo` ishlaydimi?" — Ha, lekin `PureComponent` (built-in shallow check) odatda yetarli. `memo(class)` orqali HOC qatlami ortiqcha bo'ladi.

</details>

---

### 2. Custom `areEqual` funksiyasini qachon va qanday yozish kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`memo`'ning ikkinchi argumenti — `(prevProps, nextProps) => boolean` formatdagi custom comparator. Default `shallowEqual` yetmaganda yoziladi: nested object'lar (deep equality), props ichidan faqat ma'lum kalitlarni tekshirish, yoki konkretik field'ga selektiv solishtirish kerak bo'lganda. **Xulosa: `true` qaytarish — bailout (re-render YO'Q)**, `false` — render bajariladi. `===` qoidalariga teskari (e'tibor!).

### To'liq tushuntirish

`React.memo(Component, areEqual)` — ikkinchi argument signature:

```typescript
type Comparator<P> = (prevProps: Readonly<P>, nextProps: Readonly<P>) => boolean;
```

**MUHIM: qaytariladigan qiymat semantikasi:**

| Return | Ma'no | Natija |
|--------|-------|--------|
| `true` | Props teng | Bailout (re-render YO'Q) |
| `false` | Props teng emas | Re-render |

Bu `Array.prototype.sort`'ning comparator'iga teskari intuitsiya — adashish odat tusiga aylangan.

**Custom comparator zarurati:**

1. **Nested object props** — default shallow yetmaydi:

   ```typescript
   const props1 = { user: { id: 1, name: "A" } };
   const props2 = { user: { id: 1, name: "A" } };
   shallowEqual(props1, props2); // false — user reference farqli
   ```

2. **Selective comparison** — props'ning faqat bir qismi muhim:

   ```typescript
   // Faqat user.id muhim, qolganlari render'ga ta'sir qilmaydi
   const areEqual = (prev, next) => prev.user.id === next.user.id;
   ```

3. **Function/array reference noise** — parent yangi function reference yaratayotganda:

   ```typescript
   // onSubmit'ning identity emas, behavior muhim
   const areEqual = (prev, next) => prev.formId === next.formId;
   ```

### Kod misoli

```tsx
import React, { memo } from "react";

interface UserProfileProps {
  user: { id: string; name: string; lastSeen: Date };
  onEdit: () => void;
  theme: { primary: string; secondary: string };
}

// Default shallow yetmaydi — user va theme har render'da yangi
const UserProfile = memo(
  function UserProfile({ user, theme }: UserProfileProps) {
    return (
      <div style={{ color: theme.primary }}>
        <h2>{user.name}</h2>
        <small>Last seen: {user.lastSeen.toISOString()}</small>
      </div>
    );
  },
  (prev, next) => {
    // Faqat user.id va theme.primary muhim
    return (
      prev.user.id === next.user.id &&
      prev.user.name === next.user.name &&
      prev.theme.primary === next.theme.primary
    );
  },
);

// onEdit har render'da yangi function — mahsus comparator orqali ignore
```

**Production-real misol — chat list:**

```tsx
interface MessageProps {
  message: { id: string; text: string; sentAt: number; status: "sent" | "read" };
  onRetry: (id: string) => void;
}

const MessageItem = memo(
  function MessageItem({ message }: MessageProps) {
    return (
      <li>
        <span>{message.text}</span>
        <small>{new Date(message.sentAt).toLocaleTimeString()}</small>
        {message.status === "read" && <span>✓✓</span>}
      </li>
    );
  },
  (prev, next) => {
    // Aniq: id, text, status muhim
    // sentAt — ID dan implicitly bog'liq, alohida tekshirish shart emas
    return (
      prev.message.id === next.message.id &&
      prev.message.text === next.message.text &&
      prev.message.status === next.message.status
    );
  },
);
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Comparator chaqirilish daqiqasi:**

```typescript
// updateMemoComponent ichida (soddalashtirilgan):
if (!hasScheduledUpdateOrContext) {
  const prevProps = currentChild.memoizedProps;
  const compare = Component.compare ?? shallowEqual;

  if (compare(prevProps, nextProps) && current.ref === workInProgress.ref) {
    // BAILOUT: prev tree qayta ishlatiladi
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }
}
// false — render davom etadi
```

**`ref` props'da emas — alohida tekshiriladi:**

`React.memo` `ref` ni `compare`'ga uzatmaydi (props'da bo'lmaydi). U `current.ref === workInProgress.ref` orqali alohida solishtiriladi. `forwardRef` bilan birga ishlatilganda — ref reference o'zgarsa, bailout buziladi.

**Anti-pattern — over-comparing:**

```typescript
// ❌ JSON.stringify — katta object'lar uchun sezilarli sekin
const areEqual = (prev, next) =>
  JSON.stringify(prev) === JSON.stringify(next);
```

`JSON.stringify` katta object'lar uchun sezilarli vaqt oladi — bu render cost'idan ko'p bo'lishi mumkin. Memoization'ning ma'nosi yo'qoladi.

**Anti-pattern — under-comparing (false positive):**

```typescript
// ❌ Faqat id ni tekshirib, text ni tashlab ketish
const areEqual = (prev, next) => prev.message.id === next.message.id;
// Message.text yangilansa — UI yangilanmaydi (stale render)
```

**To'g'ri yondashuv:**

- Faqat **render-ga ta'sir qiladigan** field'lar tekshiriladi
- Har field uchun primitiv tenglik (`===` yoki `Object.is`) kifoya
- Function/array reference'lardan **iloji boricha qochish** (callback'ni custom comparator'da ignore qilish — agar function ichidagi closure stale bo'lib qolmasa)

> **Performance note:** Comparator'ning o'zi har render'da bajariladi — uning xarajati render xarajatidan kam bo'lishi shart. `n` ta props field'ni `===` orqali solishtirish O(n) — sub-mikrosekund. Lekin har bir nested object/array iteration — qo'shimcha milisekund qo'shadi.

**Stale closure xavfi:**

```typescript
const MyComponent = memo(
  function MyComponent({ data, onAction }: Props) {
    return <button onClick={onAction}>Click</button>;
  },
  (prev, next) => prev.data.id === next.data.id, // onAction ignored
);

// ❌ MUAMMO: agar parent onAction'ni o'zgartirgan bo'lsa
// (closure ichida yangi state captured), MyComponent eski function ushlab qoladi
// Click bosilganda — eski state ishlatiladi (stale)
```

`useCallback` bilan onAction'ni stable qilish va default `shallowEqual` ishlatish — odatda xavfsizroq.

</details>

### Edge Cases

- **`prev` va `next` `null`/`undefined`**: React kafolatlaydi — har ikkala argument object bo'lib uzatiladi (props har doim object). Lekin keys bo'sh bo'lishi mumkin.
- **Comparator throw qilsa**: Render fail qiladi (Error Boundary tushiradi). Try-catch ichida tekshirish kerak (lekin defensive coding — yomon belgi).
- **Re-renders despite `true`**: State o'zgarganda yoki context yangilanganda comparator chaqirilmaydi. Bu kafolatlangan re-render trigger'lar.

### Follow-up savollar

- "Default shallowEqual ko'pchilik holatda yetarli — qachon custom yozish ortiqcha?" — Agar parent props'ni `useMemo`/`useCallback` bilan stabilize qila olsa — custom comparator ortiqcha. Source of truth'da reference'larni stable saqlash afzal.
- "Custom comparator vs immer (immutable updates)" — Immer immutable updates'ni avtomat qiladi → reference'lar stable bo'ladi → default `shallowEqual` yetadi. Custom comparator kerakmas.
- "Class komponent uchun ekvivalent?" — `shouldComponentUpdate(nextProps, nextState)` — `false` qaytarsa skip. `PureComponent` esa shallow auto-check.

</details>

---

### 3. `memo` va `useCallback` nima uchun bir-biriga bog'liq? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`memo` props'ni shallow solishtiradi — har function reference yangi bo'lsa, **bailout buziladi**. Parent har render'da yangi function literal yaratganda (closure'da fresh state capture), `useCallback` shu function'ni mount paytida bir marta yaratib, deps array'ga ko'ra **memoized reference**ni qaytaradi. Shu sababli `memo` (child) + `useCallback` (parent) **juftlikda** ishlaydi — biri bo'lmasa, ikkinchisining samarasi yo'qoladi.

### To'liq tushuntirish

**Muammoning manbai:**

JSX'da inline function har render'da yangi reference bo'lib yaratiladi:

```tsx
function Parent() {
  return <Child onClick={() => console.log("hi")} />;
  //                    ^^^^^^^^^^^^^^^^^^^^^^^^^
  // Har render — yangi function object
}
```

JavaScript'da `(() => {}) === (() => {})` — `false`. Function ham reference type. Har JSX evaluation yangi function instance yaratadi.

**`memo` shallow comparison'ning natijasi:**

```tsx
const Child = memo(function Child({ onClick }) {
  return <button onClick={onClick}>Click</button>;
});

function Parent() {
  const [count, setCount] = useState(0);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child onClick={() => console.log("hi")} />  {/* Har render yangi! */}
    </>
  );
}

// Click bosish: setCount → Parent re-render → Child onClick yangi reference
// → memo bailout buziladi → Child re-render
// → memo() ortiqcha edi
```

**`useCallback` yechim:**

```tsx
function Parent() {
  const [count, setCount] = useState(0);

  // Memoized — deps bo'lmaganda har render'da bir xil reference
  const handleClick = useCallback(() => {
    console.log("hi");
  }, []);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child onClick={handleClick} />
    </>
  );
}
// Click bosilganda: setCount → Parent re-render → handleClick reference o'zgarmagan
// → memo bailout ishlaydi → Child re-render YO'Q
```

**`useMemo` (object/array reference uchun) — bir xil printsip:**

```tsx
function Parent() {
  const config = useMemo(() => ({ size: 10, color: "red" }), []);
  return <Child config={config} />;
}
```

### Kod misoli

**To'liq misol — search filter list:**

```tsx
import React, { memo, useCallback, useState } from "react";

interface Product {
  id: string;
  name: string;
  price: number;
}

interface ProductCardProps {
  product: Product;
  onAddToCart: (id: string) => void;
}

const ProductCard = memo(function ProductCard({
  product,
  onAddToCart,
}: ProductCardProps) {
  console.log(`ProductCard render: ${product.name}`);
  return (
    <div>
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button onClick={() => onAddToCart(product.id)}>Add to Cart</button>
    </div>
  );
});

function ProductList({ products }: { products: Product[] }) {
  const [search, setSearch] = useState("");
  const [cart, setCart] = useState<string[]>([]);

  // ✅ useCallback — stable reference
  const handleAddToCart = useCallback((id: string) => {
    setCart((prev) => [...prev, id]);
    // setCart functional update — `cart` deps shart emas
  }, []);

  const filtered = products.filter((p) =>
    p.name.toLowerCase().includes(search.toLowerCase()),
  );

  return (
    <>
      <input
        value={search}
        onChange={(e) => setSearch(e.target.value)}
        placeholder="Search..."
      />
      <p>Cart: {cart.length} items</p>
      {filtered.map((p) => (
        <ProductCard key={p.id} product={p} onAddToCart={handleAddToCart} />
      ))}
    </>
  );
}

// Search typing: ProductList re-render → filtered o'zgaradi
// Lekin ProductCard'lar (filtered ichida bo'lganlari) bailout qiladi
// — chunki product reference o'zgarmagan VA handleAddToCart stable
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`useCallback` Hook ichida nima saqlanadi:**

```typescript
// react/src/ReactHooks.js (mental model)
function mountCallback<T>(callback: T, deps: Array<unknown> | null): T {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = [callback, deps];
  return callback;
}

function updateCallback<T>(callback: T, deps: Array<unknown> | null): T {
  const hook = updateWorkInProgressHook();
  const prevState = hook.memoizedState as [T, Array<unknown> | null];

  if (prevState !== null) {
    if (deps !== null) {
      const prevDeps = prevState[1];
      if (areHookInputsEqual(deps, prevDeps)) {
        return prevState[0]; // ← OLD CALLBACK reference qaytariladi
      }
    }
  }

  hook.memoizedState = [callback, deps];
  return callback; // Yangi callback
}

function areHookInputsEqual(
  nextDeps: Array<unknown>,
  prevDeps: Array<unknown> | null,
): boolean {
  if (prevDeps === null) return false;
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (Object.is(nextDeps[i], prevDeps[i])) continue;
    return false;
  }
  return true;
}
```

**E'tibor:**
- `useCallback(fn, deps)` — `useMemo(() => fn, deps)` ekvivalenti
- Har render'da fresh `callback` argument hook'ga uzatiladi (closure fresh)
- Lekin deps teng bo'lsa — eski reference qaytariladi (yangi callback ignore qilinadi)
- Bu `[callback, deps]` array fiber'ning `memoizedState` zanjirida saqlanadi (linked list)

**Stale closure xavfi va `useCallback`:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  // ❌ deps: [] — count har doim 0 (stale closure)
  const increment = useCallback(() => {
    setCount(count + 1); // count = 0 forever
  }, []);

  // ✅ deps: [count] — har count o'zgarsa, yangi callback
  // Ammo: bu memoization'ni effectively bekor qiladi
  const incrementCorrect = useCallback(() => {
    setCount(count + 1);
  }, [count]);

  // ✅✅ Functional update — closure'siz, deps bo'sh
  const incrementBest = useCallback(() => {
    setCount((c) => c + 1);
  }, []);

  return <button onClick={incrementBest}>{count}</button>;
}
```

**`memo` + `useCallback` kombinatsiya jadvali:**

| Memo'lash | Bailout | Sabab |
|-----------|---------|-------|
| `memo` yo'q + `useCallback` yo'q | Yo'q | Default behavior — re-render |
| `memo` bor + `useCallback` yo'q | Yo'q | Function reference yangi → bailout buzildi |
| `memo` yo'q + `useCallback` bor | Yo'q | Memo yo'qligida solishtirish bo'lmaydi |
| `memo` bor + `useCallback` bor | **Ha** | Props stable → bailout |

**Cost-Benefit:**

`memo` + `useCallback` ortiqcha bo'lishi mumkin:
- Komponent kichik (`<span>`, `<button>` simple)
- Re-render'lar tez (sub-millisecond)
- `useCallback` o'zi memory + dependency check overhead qo'shadi

> **Cost-Benefit:** `useCallback` o'z-o'zidan hook overhead qo'shadi (slot allocation, deps array, `Object.is` taqqoslash). Komponent rendering vaqti bu overhead'dan kichik bo'lsa — memoization net foyda bermasligi mumkin. Aniq tahlil uchun React DevTools Profiler bilan o'lchang.

**Children prop edge case:**

```tsx
const Wrapper = memo(({ children }) => <div>{children}</div>);

function Parent() {
  return <Wrapper><span>hi</span></Wrapper>;
  // ❌ children — har render'da yangi React element
  // memo bailout buziladi
}

// ✅ Yechim: children'ni stabilize qilish
function Parent() {
  const content = useMemo(() => <span>hi</span>, []);
  return <Wrapper>{content}</Wrapper>;
}
```

</details>

### Edge Cases

- **`useCallback` deps unstable**: Agar deps har render'da yangi (object/array literal), `useCallback` har render'da yangi function qaytaradi — memoization yo'qoladi.
- **`useCallback` o'zi yetmaydi, props ham stable bo'lishi kerak**: Object/array props uchun `useMemo`. Agar bitta props unstable bo'lsa — barchasi bekor.
- **React Compiler (R19) bekor qiladi**: Compiler ishlatilsa, `useCallback`/`useMemo` qo'l bilan yozish ortiqcha. Compiler avtomat memoization qiladi.

### Follow-up savollar

- "Closure'da fresh state olish va memoization — kelishishi qiyin?" — Functional state updates (`setX(x => ...)`), `useReducer` (action dispatch), `useRef` (mutable storage) — ularning hammasi closure'siz ishlaydi.
- "`useEvent` (proposal) nima vazifani bajaradi?" — RFC: stable callback identity, lekin har chaqirilganda fresh state ko'radi. Hozircha `useEffectEvent` shaklida React'da mavjud (experimental API — stable release'ga kiritilmagan).
- "Inline function har doim performance muammomi?" — Yo'q. Faqat `memo`'langan komponentga props sifatida uzatilsa muhim. Oddiy DOM `<button onClick={() => ...}>` — DOM event listener, performance ta'siri yo'q.

</details>

---

### 4. `memo` qachon ishlamaydi (bypass scenarios)? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`memo` quyidagi 4 ta scenarioda **bypass qilinadi** (bailout ishlamaydi): (1) Inline object/array/function props — har render yangi reference. (2) `children` prop berilganda — JSX element har gal yangi. (3) Context value o'zgarganda — `compare` chaqirilmaydi. (4) Komponent ichida `useState`/`useReducer`/`forceUpdate` orqali state o'zgarganda. Yana — Strict Mode 2x render dev'da memo'lashtirilgan komponent ham 2 marta render qilinadi.

### To'liq tushuntirish

**Bypass scenario 1 — Inline reference props:**

```tsx
const Child = memo(({ config }) => <div>{config.size}</div>);

function Parent() {
  return <Child config={{ size: 10 }} />;
  //              ^^^^^^^^^^^^^^^^^^
  // Har render — yangi object literal
  // shallowEqual({size:10}, {size:10}) === false
  // → memo BAILOUT YO'Q
}
```

`{ size: 10 } === { size: 10 }` — JS'da `false`. Object/array/function literal — har JSX evaluation'da fresh allocation.

**Bypass scenario 2 — `children` prop:**

```tsx
const Wrapper = memo(({ children }) => <div>{children}</div>);

function Parent() {
  return (
    <Wrapper>
      <span>Hello</span>  {/* Har render: yangi React element */}
    </Wrapper>
  );
}

// JSX <span>Hello</span> har Parent render'da:
// React.createElement("span", null, "Hello")
// → yangi { type: "span", props: { children: "Hello" }, key: null, ref: null }
// → shallowEqual(prevChildren, nextChildren) === false
// → BAILOUT BUZILADI
```

**Bypass scenario 3 — Context value o'zgarish:**

```tsx
const ThemeContext = createContext({ primary: "blue" });

const ThemedButton = memo(function ThemedButton({ label }) {
  const theme = useContext(ThemeContext);
  return <button style={{ color: theme.primary }}>{label}</button>;
});

// ThemeContext.Provider value o'zgarsa:
// → checkScheduledUpdateOrContext returns true
// → memo `compare` CHAQIRILMAYDI
// → Komponent qayta render qilinadi (kerakli)
```

Bu kafolatlangan re-render — context o'zgarsa, subscriber'lar yangilanishi kerak.

**Bypass scenario 4 — internal state:**

```tsx
const Counter = memo(function Counter({ initial }) {
  const [count, setCount] = useState(initial);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
});

// setCount chaqirilsa:
// → Counter Fiber lanes'iga update qo'shiladi
// → checkScheduledUpdateOrContext returns true
// → compare ignore qilinadi
// → Re-render kerakli (state ko'rsatish uchun)
```

**Bypass 5 — Strict Mode dev:**

Development build + `<StrictMode>` da har render 2 marta bajariladi (idempotency tekshirish). `memo` ham 2x render qiladi — bu kutilgan.

**Bypass 6 — `key` o'zgarish:**

```tsx
function Parent({ id }) {
  return <Child key={id} />;
}
// id o'zgarsa — Child unmount + remount, mavjud Fiber tashlanadi
// memo nazarda tutilmaydi — yangi instance
```

### Kod misoli

**Real-world scenario — barcha bypass'larni ko'rish:**

```tsx
import React, { memo, createContext, useContext, useState, useMemo, useCallback } from "react";

const ThemeContext = createContext({ primary: "blue" });

interface ProductCardProps {
  product: { id: string; name: string };
  config: { showPrice: boolean };
  onSelect: (id: string) => void;
  children: React.ReactNode;
}

const ProductCard = memo(function ProductCard({
  product,
  config,
  onSelect,
  children,
}: ProductCardProps) {
  const theme = useContext(ThemeContext);
  return (
    <div style={{ color: theme.primary }}>
      <h3>{product.name}</h3>
      {config.showPrice && <p>Price hidden</p>}
      {children}
      <button onClick={() => onSelect(product.id)}>Select</button>
    </div>
  );
});

// ❌ Bypass-prone
function BadProductList() {
  const [count, setCount] = useState(0);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <ProductCard
        product={{ id: "p1", name: "Phone" }}        // ❌ inline object
        config={{ showPrice: true }}                  // ❌ inline object
        onSelect={(id) => console.log(id)}            // ❌ inline function
      >
        <span>Featured!</span>                        {/* ❌ children element */}
      </ProductCard>
    </>
  );
}
// count++ → BadProductList re-render → ProductCard barcha props yangi → memo bailout buziladi

// ✅ Memoization-friendly
function GoodProductList({ products }: { products: Array<{ id: string; name: string }> }) {
  const [count, setCount] = useState(0);

  const config = useMemo(() => ({ showPrice: true }), []);
  const handleSelect = useCallback((id: string) => console.log(id), []);
  const featuredBadge = useMemo(() => <span>Featured!</span>, []);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      {products.map((p) => (
        <ProductCard
          key={p.id}
          product={p}              // Stable — products array'dan
          config={config}
          onSelect={handleSelect}
        >
          {featuredBadge}
        </ProductCard>
      ))}
    </>
  );
}
// count++ → ProductCard'lar bailout qiladi (props stable)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Reconciler ichidagi bailout shartlari kombinatsiyasi:**

```typescript
// updateMemoComponent (mental model)
function updateMemoComponent(current, workInProgress, Component, nextProps, renderLanes) {
  if (current === null) {
    // First mount — bailout impossible
    return mountChild(...);
  }

  // 1. Check scheduled update or context change
  const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(current, renderLanes);

  if (hasScheduledUpdateOrContext) {
    // BYPASS: render davom etadi
    return updateChild(...);
  }

  // 2. Compare props
  const prevProps = current.child.memoizedProps;
  const compare = Component.compare ?? shallowEqual;

  if (!compare(prevProps, nextProps) || current.ref !== workInProgress.ref) {
    // BYPASS: render davom etadi
    return updateChild(...);
  }

  // BAILOUT
  return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
}
```

**`checkScheduledUpdateOrContext` ichi:**

```typescript
function checkScheduledUpdateOrContext(current: Fiber, renderLanes: Lanes): boolean {
  // 1. Update queue'da pending update bormi?
  const updateLanes = current.lanes;
  if (includesSomeLane(updateLanes, renderLanes)) {
    return true;
  }
  // 2. Context'lar yangilanganmi?
  if (enableLazyContextPropagation) {
    const dependencies = current.dependencies;
    if (dependencies !== null && checkIfContextChanged(dependencies)) {
      return true;
    }
  }
  return false;
}
```

**Object literal va engine optimization:**

V8 Hidden Class — bir xil shape'dagi object'lar bir xil class. Ammo:

```typescript
function render() {
  return { size: 10 }; // Har chaqirilganda yangi object instance,
                        // shape (hidden class) bir xil — lekin reference farq.
}
render() === render(); // false
```

V8 escape analysis ba'zida stack-allocate qiladi (kichik object'lar). Lekin React Reconciler darajasida — shallow comparison reference bo'yicha — bu optimizatsiyalar ahamiyatli emas.

**`children` element identity:**

```typescript
// JSX:
<Wrapper><span>hi</span></Wrapper>

// Compiled (jsx-runtime):
jsx(Wrapper, {
  children: jsx("span", { children: "hi" }),
});

// Har Parent render — outer jsx() chaqiriladi → yangi { type, props, ... } object
```

**Bypass workaround — children memoization:**

```tsx
function Parent() {
  // useMemo bilan element'ni cache
  const cachedChild = useMemo(() => <span>hi</span>, []);
  return <Wrapper>{cachedChild}</Wrapper>;
}

// Yoki — `children` o'rniga callback prop:
const Wrapper = memo(({ render }) => <div>{render()}</div>);
function Parent() {
  const stable = useCallback(() => <span>hi</span>, []);
  return <Wrapper render={stable} />;
}
```

**Profiler with bypass detection:**

React DevTools Profiler — "Why did this render?" sababini ko'rsatadi:
- "Props changed: children, onClick"
- "Hooks changed"
- "Context changed: ThemeContext"

Bu xabar bypass'ni aniqlashtirish uchun eng qulay tool.

</details>

### Edge Cases

- **`forwardRef(memo(...))` order**: `memo(forwardRef(...))` to'g'ri — memo outer. Aksi `forwardRef(memo(...))` ham ishlaydi, lekin outer ref propagation chalkash bo'ladi.
- **`Provider` value har render'da yangi object**: Context value object literal bo'lsa, har render'da `Provider` value yangi → barcha consumers re-render. `useMemo` bilan stabilize qilish kerak.
- **Bypass dev-only check**: Strict Mode dev 2x — production'da bir martalik. Production performance test dev numbers'ga teng emas.

### Follow-up savollar

- "Babel / Vite — JSX compile-time'da object literal hoist qilarmidi?" — Yo'q. Object/array literal har JSX evaluation'da yangi. React Compiler (R19) buni hal qiladi avtomat memoization orqali.
- "Profiler bypass'ni qanday ko'rsatadi?" — "Why did this render?" panel — sababni real-time'da. "Props" + qaysi key'lar o'zgargan.
- "`memo` versus `PureComponent`" — Class-based equivalent. `PureComponent` o'zining `shouldComponentUpdate`'ida shallow check. Function `memo` — Hooks-friendly, bir xil semantika.

</details>

---

<a id="qism-b"></a>

### 5. `React.memo` + Context — bypass scenario [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Context subscriber komponent **`React.memo`'ni bypass qiladi** — Context value o'zgarganda barcha consumer'lar re-render bo'ladi (memo shallow props check qilmasdan). Sabab: Context o'qish — Reconciler ichki subscription, props comparison'dan oldin happens. Yechim: Context selector library (`use-context-selector`), context splitting (state vs dispatch), yoki external store (Zustand).

### Kod misoli

```tsx
const CountContext = createContext(0);

const Child = memo(function Child() {
  console.log("Child render");
  const count = useContext(CountContext);  // ← subscribes
  return <p>{count}</p>;
});

function App() {
  const [count, setCount] = useState(0);
  return (
    <CountContext.Provider value={count}>
      <button onClick={() => setCount(c => c + 1)}>+</button>
      <Child />  {/* memo wrapper, but renders each setState */}
    </CountContext.Provider>
  );
}

// Click button:
// "Child render" — every time, despite memo
// Reason: context value change → subscriber re-render
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Reconciler context subscription:**

```typescript
function useContext(Context) {
  const dispatcher = ReactCurrentDispatcher.current;
  return dispatcher.useContext(Context);
}

function readContext(Context) {
  // Subscribe fiber to context
  if (lastContextDependency === null) {
    currentlyRenderingFiber.dependencies = {
      lanes: NoLanes,
      firstContext: { context: Context, next: null },
    };
  }
  return Context._currentValue;
}
```

Har `useContext` chaqiruvi fiber'ni context'ga subscribe qiladi. Provider value o'zgarganda — barcha subscriber'lar qayta render qilinadi (memo'ga qaramay).

**Context splitting pattern:**

```tsx
// ❌ Single context — all consumers re-render
const StoreContext = createContext({ count: 0, name: "" });

// ✅ Split contexts — granular subscriptions
const CountContext = createContext(0);
const NameContext = createContext("");

// Consumer reads only one — re-renders only on that context change
const ChildA = memo(() => {
  const count = useContext(CountContext);
  return <p>{count}</p>;
});

const ChildB = memo(() => {
  const name = useContext(NameContext);
  return <p>{name}</p>;
});
```

**Selector library:**

```tsx
import { createContext, useContextSelector } from "use-context-selector";

const StoreContext = createContext({ count: 0, name: "" });

const ChildCount = memo(function ChildCount() {
  // Only re-renders if count changes
  const count = useContextSelector(StoreContext, (s) => s.count);
  return <p>{count}</p>;
});
```

**External store alternative:**

```tsx
// Zustand — selector-based, fine-grained
import { create } from "zustand";

const useStore = create<State>((set) => ({
  count: 0,
  name: "",
  increment: () => set((s) => ({ count: s.count + 1 })),
}));

const ChildCount = memo(function ChildCount() {
  const count = useStore((s) => s.count);  // selective subscription
  return <p>{count}</p>;
});
```

**`useMemo` Provider value:**

```tsx
function Provider({ children }) {
  const [count, setCount] = useState(0);
  const [name, setName] = useState("");

  // ❌ New object each render — even memo'd consumers re-render
  const value = { count, name };

  // ✅ Memoize
  const value = useMemo(() => ({ count, name }), [count, name]);

  return <StoreContext.Provider value={value}>{children}</StoreContext.Provider>;
}
```

</details>

### Edge Cases

- **Context object value**: Har render'da yangi object — barcha consumer'lar re-render. `useMemo` bilan stabilize qilish kerak.
- **Bir komponentda bir nechta context**: Har biri alohida subscribe. Istalgan biri o'zgarganda — re-render.
- **Provider ichida setState**: Provider re-render → consumer'lar re-render.

### Follow-up savollar

- "React nima uchun avtomat bailout qilmaydi?" — Context subscription model'da ishlaydi. Selector pattern React core'da yo'q — tashqi library.
- "use-context-selector library?" — dai-shi'ning library'si. Context'ga selector-based subscription qo'shadi.

</details>

---

## QISM B: Re-render Mechanics

### 6. Komponent qayta render bo'lishining asosiy sabablari nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Komponent quyidagi 3 ta asosiy sabab bilan re-render qilinadi: (1) **State o'zgarishi** — `useState`/`useReducer` chaqiruvi. (2) **Parent re-render** — default tarzda parent qayta render bo'lsa, barcha child'lar ham qayta render qilinadi (`memo` bo'lmaganda). (3) **Context o'zgarishi** — komponent `useContext` orqali subscribe qilgan provider value yangilandi. **Props o'zgarishi o'z-o'zidan trigger EMAS** — parent re-render natijasida child re-render bo'ladi, props yangi bo'lishi bu re-render'ning sababiy emas, oqibati.

### To'liq tushuntirish

**1. State o'zgarishi:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
// setCount → Fiber.lanes ga update qo'shiladi → render scheduled
```

`setCount(newValue)` quyidagilarni amalga oshiradi:
1. `Object.is(newValue, currentValue)` — agar teng bo'lsa, **bailout** (re-render YO'Q)
2. Teng emas — Fiber'ning `lanes`'iga lane qo'shiladi
3. Scheduler'ga ish qo'shiladi → keyingi tick'da render

**2. Props o'zgarishi:**

Props o'zgarishi parent render bilan birga keladi — props o'zlari render trigger emas, balki parent render natijasi.

**3. Context o'zgarishi:**

```tsx
const ThemeContext = createContext({ primary: "blue" });

function App() {
  const [theme, setTheme] = useState({ primary: "blue" });
  return (
    <ThemeContext.Provider value={theme}>
      <NestedComponent />
    </ThemeContext.Provider>
  );
}

function NestedComponent() {
  const theme = useContext(ThemeContext);
  return <div style={{ color: theme.primary }}>Hi</div>;
}
// setTheme → Provider value yangi → useContext consumer'lar qayta render
```

**4. Parent re-render (default propagation):**

```tsx
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child />  {/* Parent re-render → Child ham re-render */}
    </>
  );
}

function Child() {
  console.log("Child render");
  return <div>Static</div>;
}
// Click: Parent re-render → Child re-render (ham qayta render)
```

Bu **default cascade** — `memo` bilan to'sib qo'yiladi, aks holda parent har re-render'da barcha descendant'lar ham re-render.

### Kod misoli

```tsx
import { useState, useContext, createContext, memo } from "react";

const ThemeContext = createContext("light");

// 1. State trigger
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// 2. Props trigger (via parent re-render)
function ChildA({ value }: { value: number }) {
  console.log("ChildA render");
  return <p>{value}</p>;
}

// 3. Context trigger
function ChildB() {
  const theme = useContext(ThemeContext);
  console.log("ChildB render");
  return <p>{theme}</p>;
}

// 4. Parent re-render trigger (no memo)
function ChildC() {
  console.log("ChildC render — default cascade");
  return <p>Static</p>;
}

function App() {
  const [count, setCount] = useState(0);
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext.Provider value={theme}>
      <button onClick={() => setCount(c => c + 1)}>+</button>
      <button onClick={() => setTheme(t => (t === "light" ? "dark" : "light"))}>
        Toggle Theme
      </button>
      <ChildA value={count} />  {/* count o'zgarsa props yangi → render */}
      <ChildB />                {/* theme o'zgarsa context yangi → render */}
      <ChildC />                {/* App har render'da → cascade render */}
    </ThemeContext.Provider>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`setState` ichidagi `Object.is` bailout:**

```typescript
// react-reconciler/src/ReactFiberHooks.js (mental model)
function dispatchSetState(fiber, queue, action) {
  const lane = requestUpdateLane(fiber);
  const update = { lane, action, hasEagerState: false, eagerState: null, next: null };

  // Eager bailout — agar fiber idle bo'lsa, lazily compute
  if (fiber.lanes === NoLanes && (fiber.alternate === null || fiber.alternate.lanes === NoLanes)) {
    const lastRenderedReducer = queue.lastRenderedReducer;
    if (lastRenderedReducer !== null) {
      const currentState = queue.lastRenderedState;
      const eagerState = lastRenderedReducer(currentState, action);
      update.hasEagerState = true;
      update.eagerState = eagerState;
      if (Object.is(eagerState, currentState)) {
        // BAILOUT: re-render trigger qilinmaydi
        return;
      }
    }
  }

  // Schedule render
  scheduleUpdateOnFiber(fiber, lane);
}
```

Eager state bailout — `setState(sameValue)` chaqirsa, render scheduled bo'lmaydi (optimal). Lekin agar concurrent update'lar bor bo'lsa, eager evaluation skip qilinadi.

**Context propagation algoritmi:**

```typescript
// Provider value o'zgarganda:
function propagateContextChange(workInProgress, context, renderLanes) {
  let fiber = workInProgress.child;
  while (fiber !== null) {
    // Har descendant Fiber tekshiriladi
    if (fiber.dependencies !== null) {
      let dependency = fiber.dependencies.firstContext;
      while (dependency !== null) {
        if (dependency.context === context) {
          // Bu Fiber shu context'ni read qiladi
          fiber.lanes = mergeLanes(fiber.lanes, renderLanes);
          break;
        }
        dependency = dependency.next;
      }
    }
    fiber = nextFiber(fiber);
  }
}
```

R18+ "lazy context propagation" — faqat haqiqatan ham consumer fiber'lar yangilanadi, qolganlari bypass.

**Render lanes va priorities:**

```typescript
// Scheduler priority lanes (selected)
SyncLane = 0b0000000000000000000000000000010;
InputContinuousLane = 0b0000000000000000000000000000100;
DefaultLane = 0b0000000000000000000000000010000;
TransitionLane1 = 0b0000000000000000000000010000000;
// ... up to TransitionLane16
IdleLane = 0b0100000000000000000000000000000;
```

Har sabab uchun lane:
- State setter — DefaultLane (yoki SyncLane event handler ichida)
- `startTransition` — TransitionLane
- `useDeferredValue` — TransitionLane
- Click handler ichida setState — SyncLane (immediate)

**Re-render cascade — "render the world" mental model:**

React'ning default holatida, parent re-render barcha child'larni qayta render qiladi. Bu qoida xulq-atvori `memo` bilan to'sib qo'yiladi, lekin React arxitekturasining asosi — **declarative paradigm**'ning tabiiy oqibati: parent o'z holatini elon qiladi, child'lar undan kelib chiqadi.

**Profiler — re-render reasons:**

DevTools Profiler "Why did this render?" panel:
- "This is the first time the component rendered"
- "Hooks changed"
- "Props changed"
- "Parent component rendered"
- "The parent component rendered as a result of context change"

</details>

### Edge Cases

- **`setState` bilan bir xil qiymat**: `setCount(0)` agar `count === 0` bo'lsa — eager bailout (no render). `Object.is` solishtirish.
- **Object/array setState**: `setUser({ ...user, name: "Ali" })` — shape o'xshash bo'lsa ham, reference yangi → re-render bajariladi.
- **Asynchronous setState**: `await fetch(...)` keyin `setState` — promise resolve bo'lganda lane priority `DefaultLane`.

### Follow-up savollar

- "Hooks o'zlari render trigger qila oladimi?" — Faqat `useState`/`useReducer`/`useSyncExternalStore` orqali state o'zgarishi. `useEffect` o'zi render trigger qilmaydi (post-render run), lekin effect ichida setState chaqirsa — re-render.
- "`forceUpdate` qachon ishlatiladi?" — Function component'larda `forceUpdate` to'g'ridan-to'g'ri yo'q. `useReducer(x => x + 1, 0)` orqali simulate qilinadi (`forceUpdate hack`), kamdan-kam holatlarda.
- "Concurrent rendering re-render'larni qanday boshqaradi?" — Lanes priority bilan. High-priority (event handler) — Sync. Low-priority (transitions) — yield qiladi.

</details>

---

### 7. Nima uchun parent re-render qilsa, child'lar ham re-render bo'ladi (memo'siz)? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JSX'da `<Child />` — `React.createElement(Child, null)` ga compile qilinadi. Parent har render'da JSX'ni qaytadan evaluate qiladi → yangi React element object yaratiladi. Reconciler bu yangi elementni mavjud Child Fiber bilan diff qiladi. **`memo` yo'qligida**, Reconciler component function'ni qaytadan chaqiradi (default behavior). Bu — declarative paradigm'ning oqibati: parent o'z output'ini elon qiladi, Reconciler bu output'ni Fiber tree'da reconcile qiladi.

### To'liq tushuntirish

**JSX'ning compile natijasi:**

```tsx
function Parent() {
  return <Child name="Ali" />;
}

// Babel/SWC compile (R17+):
function Parent() {
  return jsx(Child, { name: "Ali" });
}

// jsx() qaytaradi:
// {
//   $$typeof: Symbol(react.element),
//   type: Child,         // Function reference
//   props: { name: "Ali" },
//   key: null,
//   ref: null,
// }
```

Har Parent re-render — `jsx()` qayta chaqiriladi → yangi element object yaratiladi.

**Reconciler nima qiladi:**

```typescript
// reconcileChildren — har parent render'da chaqiriladi
function reconcileChildFibers(returnFiber, currentFirstChild, newChild, lanes) {
  // newChild — JSX evaluation natijasi (ReactElement)
  // currentFirstChild — mavjud Fiber

  if (currentFirstChild !== null && currentFirstChild.type === newChild.type) {
    // Same type — update existing Fiber
    const existing = useFiber(currentFirstChild, newChild.props);
    existing.return = returnFiber;
    return existing;
  }

  // Different type — create new Fiber, delete old
  const created = createFiberFromElement(newChild, lanes);
  // ...
}
```

**`updateFunctionComponent` — re-render bajarish:**

```typescript
function updateFunctionComponent(current, workInProgress, Component, nextProps, renderLanes) {
  // Hooks dispatcher set
  prepareToReadContext(workInProgress, renderLanes);
  let nextChildren;

  // Component function chaqiriladi
  nextChildren = renderWithHooks(
    current,
    workInProgress,
    Component,       // ← Child function
    nextProps,
    null,
    renderLanes,
  );

  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

`renderWithHooks` — Child function'ni qaytadan ishga tushiradi. `memo` yo'qligida, ushbu chaqiriq har Parent re-render'da takrorlanadi.

**Parent re-render → Child re-render cascade:**

1. Parent state o'zgaradi (yoki context, yoki o'z props)
2. Parent function chaqiriladi → JSX evaluate
3. `<Child />` element yaratiladi
4. Reconciler Parent.child Fiber'ni topadi (Child uchun)
5. `updateFunctionComponent(Child, ...)` chaqiriladi
6. Child function chaqiriladi → uning JSX evaluate
7. Cascade davom etadi (Child'ning child'lariga)

### Kod misoli

```tsx
import { useState, memo } from "react";

// Default — cascade
function Parent() {
  const [count, setCount] = useState(0);

  console.log("Parent render");
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ChildA />
      <ChildB />
    </>
  );
}

function ChildA() {
  console.log("ChildA render");
  return <p>A</p>;
}

function ChildB() {
  console.log("ChildB render");
  return <p>B</p>;
}

// Click natijasi:
// Parent render
// ChildA render
// ChildB render
// (har click bosishda 3 ta log)
```

**`memo` bilan cascade to'xtatish:**

```tsx
const ChildA = memo(function ChildA() {
  console.log("ChildA render");
  return <p>A</p>;
});

const ChildB = memo(function ChildB() {
  console.log("ChildB render");
  return <p>B</p>;
});

// Click natijasi:
// Parent render
// (ChildA, ChildB — render YO'Q, props o'zgarmagan)
```

**Children prop bilan optimizatsiya (memo'siz):**

```tsx
function Parent({ children }: { children: React.ReactNode }) {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      {children}  {/* Static — Parent render'ga bog'liq emas */}
    </>
  );
}

function App() {
  return (
    <Parent>
      <ChildA />  {/* App scope'da yaratilgan, Parent re-render'ga ta'sir qilmaydi */}
      <ChildB />
    </Parent>
  );
}

// Click: Parent render
// ChildA, ChildB — render YO'Q (ular App'ning child'lari, Parent state o'zgarishi ulardan ta'sir qilmaydi)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Children prop trick — nima sodir bo'ladi:**

```typescript
// App render natijasi:
const childAElement = jsx(ChildA, null);    // ← App render'da yaratiladi
const childBElement = jsx(ChildB, null);
const parentElement = jsx(Parent, {
  children: [childAElement, childBElement], // ← stable, App'ning state'ga bog'liq emas
});

// Parent render:
// 1. Parent function chaqiriladi
// 2. Parent JSX qaytaradi: { type: ..., children: [stable refs] }
// 3. Reconciler children'ni reconcile qiladi
// 4. children element'lari prev render bilan === (App re-render bo'lmagan)
// 5. SimpleMemoComponent ishlatilmaganda ham, bailout mumkin
```

**Reconciler bailout — "same element" check:**

```typescript
// reconcileChildFibers ichida (oldFiber.elementType === newChild.type)
function reconcileSingleElement(returnFiber, currentFirstChild, element, lanes) {
  const key = element.key;
  let child = currentFirstChild;
  while (child !== null) {
    if (child.key === key && child.elementType === element.type) {
      // Same identity — update existing
      const existing = useFiber(child, element.props);
      existing.return = returnFiber;
      return existing;
    }
    child = child.sibling;
  }
  // ... new Fiber
}
```

`element.type === Child` — function reference. Lekin bu **Fiber**'ni qayta ishlatishni anglatadi, **render'ni skip etishni emas**. `updateFunctionComponent` baribir `renderWithHooks` chaqiradi.

**`bailoutOnAlreadyFinishedWork` qachon chaqiriladi:**

Faqat:
1. `MemoComponent`/`SimpleMemoComponent` tag, props teng
2. `ContextProvider` value teng
3. `LazyComponent` cached
4. Bir nechta boshqa special tag'lar

Default `FunctionComponent` tag uchun bailout YO'Q — har parent render'da render bajariladi.

**"Same element" bypass — `props === oldProps`:**

```typescript
// react-reconciler/src/ReactFiberBeginWork.js
function attemptEarlyBailoutIfNoScheduledUpdate(current, workInProgress, renderLanes) {
  switch (workInProgress.tag) {
    case FunctionComponent: {
      // No early bailout for function components by default
      break;
    }
    case MemoComponent:
    case SimpleMemoComponent:
      // Bailout possible
      break;
    // ...
  }
}
```

**Sabab nima uchun shunday:**

React function komponent — shunchaki function. React function output'ni avval ko'ra olmaydi. Memoization uchun explicit `memo()` kerak. Bu **explicit > implicit** principle — developer ataylab tanlaydi.

> **Performance reality:** Default cascade re-renders odatda muammo emas — virtual DOM diff bilan haqiqiy DOM update faqat zarurat bor joyda bajariladi. Lekin component function'ning o'zi qimmat (heavy compute, ko'p hooks) bo'lsa — `memo` foyda qiladi.

</details>

### Edge Cases

- **State equal old state**: `setState` bilan bir xil qiymat — eager bailout, parent re-render trigger qilinmaydi (no cascade).
- **Children prop bilan static**: Children static refs bo'lsa, parent re-render'da children'lar bypass qilinishi mumkin (Reconciler element identity check).
- **`React.Fragment`**: Fragment Fiber'i bor, lekin u to'g'ridan-to'g'ri children'larini render qiladi. Parent → Fragment → Children — barchasi cascade.

### Follow-up savollar

- "`useMemo` JSX cache qila oladimi?" — Ha. `useMemo(() => <Child />, [deps])` — element cache. Cascade'ni to'xtatish trick.
- "`React.Compiler` (R19) bu cascade'ni hal qiladimi?" — Ha. Compiler statik analiz orqali "bu props/deps o'zgarmagan" deb aniqlaydi va memoization avtomat qo'shadi.
- "Default cascade qachon muammo?" — Komponent ichida heavy compute (`O(n^2)` array operations, complex JSX tree). Profiler bilan o'lchang.

</details>

---

### 8. Reconciler bailout algoritmi qanday ishlaydi (chuqur)? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Reconciler render fazasida har Fiber uchun `beginWork` chaqiradi. Agar Fiber'ning **`lanes`'ida** pending update **YO'Q** va **context'lari o'zgarmagan** bo'lsa — `attemptEarlyBailoutIfNoScheduledUpdate` chaqiriladi, bu Fiber'ni "`already finished`" deb belgilab, child'larini ham skip qiladi (`childLanes` orqali). `MemoComponent` uchun qo'shimcha props comparison bor. Bailout — render fazasi optimizatsiyasi: Fiber qayta hisoblanmaydi, balki oldingi `child`/`sibling`/`memoizedState` reuse qilinadi.

### To'liq tushuntirish

**Bailout 4 ta darajada ishlaydi:**

| Daraja | Sabab | Optimizatsiya |
|--------|-------|---------------|
| 1. Fiber whole-tree | `lanes`'da update yo'q + context o'zgarmagan | `attemptEarlyBailoutIfNoScheduledUpdate` — to'liq sub-tree skip |
| 2. MemoComponent | Props shallow equal | `bailoutOnAlreadyFinishedWork` — child render skip |
| 3. ContextProvider | Value teng | Skip propagation |
| 4. State | `Object.is(newState, prevState)` | `setState` bailout, render trigger yo'q |

**Reconciler `beginWork`:**

```typescript
function beginWork(current: Fiber | null, workInProgress: Fiber, renderLanes: Lanes): Fiber | null {
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (oldProps !== newProps || hasLegacyContextChanged()) {
      // Props o'zgargan — full render
      didReceiveUpdate = true;
    } else {
      // Props bir xil — early bailout possible
      const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(current, renderLanes);
      if (!hasScheduledUpdateOrContext) {
        // BAILOUT
        didReceiveUpdate = false;
        return attemptEarlyBailoutIfNoScheduledUpdate(current, workInProgress, renderLanes);
      }
      didReceiveUpdate = false;
    }
  }

  // Render davom etadi
  switch (workInProgress.tag) {
    case FunctionComponent: { ... }
    case MemoComponent: { ... }
    // ...
  }
}
```

**`attemptEarlyBailoutIfNoScheduledUpdate`:**

```typescript
function attemptEarlyBailoutIfNoScheduledUpdate(current, workInProgress, renderLanes) {
  // Fiber turiga qarab — context'larni clone, etc
  switch (workInProgress.tag) {
    case HostRoot: { ... }
    case ContextProvider: {
      const newValue = workInProgress.memoizedProps.value;
      const context = workInProgress.type._context;
      pushProvider(workInProgress, context, newValue);
      break;
    }
    // ...
  }
  return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
}

function bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes): Fiber | null {
  if (current !== null) {
    workInProgress.dependencies = current.dependencies;
  }
  // childLanes — descendant'larda update bormi?
  if (!includesSomeLane(workInProgress.childLanes, renderLanes)) {
    // BUTUN SUB-TREE SKIP
    return null;
  }
  // Children'lar bo'yicha qisman bailout
  cloneChildFibers(current, workInProgress);
  return workInProgress.child;
}
```

**`childLanes` — descendant tracking:**

Har Fiber `childLanes` field'ini saqlaydi — descendant'larida pending update bor lane'lar. `setState` chaqirilganda, lane shu Fiber va uning **barcha ancestor'lariga** propagate qilinadi (`scheduleUpdateOnFiber` orqali).

```typescript
function markUpdateLaneFromFiberToRoot(sourceFiber, lane) {
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  let alternate = sourceFiber.alternate;
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
  let node = sourceFiber.return;
  while (node !== null) {
    node.childLanes = mergeLanes(node.childLanes, lane);
    if (node.alternate !== null) {
      node.alternate.childLanes = mergeLanes(node.alternate.childLanes, lane);
    }
    node = node.return;
  }
}
```

Bu mexanizm orqali:
- `childLanes` empty → butun sub-tree skip
- `childLanes` non-empty → descendant'larga descend, lekin path-bo'yicha non-update Fiber'lar skip

### Kod misoli

**Bailout natijasi - render trace:**

```tsx
import { useState, memo, useContext, createContext } from "react";

const ThemeContext = createContext("light");

const Leaf = memo(function Leaf({ value }: { value: string }) {
  console.log(`Leaf(${value}) render`);
  return <span>{value}</span>;
});

function Subtree({ data }: { data: string[] }) {
  console.log("Subtree render");
  return (
    <ul>
      {data.map((d) => (
        <Leaf key={d} value={d} />
      ))}
    </ul>
  );
}

function App() {
  const [count, setCount] = useState(0);
  const [theme, setTheme] = useState("light");
  const data = ["a", "b", "c"]; // Static

  console.log(`App render — count=${count}, theme=${theme}`);
  return (
    <ThemeContext.Provider value={theme}>
      <button onClick={() => setCount(c => c + 1)}>+</button>
      <Subtree data={data} />
    </ThemeContext.Provider>
  );
}

// 1. Initial mount:
// App render — count=0, theme=light
// Subtree render
// Leaf(a) render
// Leaf(b) render
// Leaf(c) render

// 2. Click + button (setCount):
// App render — count=1, theme=light
// Subtree render  ← Subtree memo'siz — re-render
// (Leaf'lar — props o'zgarmagan, memo bailout — render YO'Q)

// 3. setTheme:
// App render — count=1, theme=dark
// Subtree render  ← memo'siz, parent re-render → child re-render
// (Leaf'lar — bailout)
```

**Aniq bailout — `Subtree` ham memo bilan:**

```tsx
const Subtree = memo(function Subtree({ data }: { data: string[] }) {
  console.log("Subtree render");
  return (
    <ul>
      {data.map((d) => <Leaf key={d} value={d} />)}
    </ul>
  );
});

// setCount bosish:
// App render — count=1
// (Subtree — props (data) reference bir xil → bailout → render YO'Q)
// Faqat 1 ta log
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`childLanes` magic — sub-tree fully skipped:**

```typescript
// Mental model:
// App.lanes = SyncLane (count update)
// App.childLanes = NoLanes (descendant'lar update yo'q)
// Subtree.lanes = NoLanes
// Subtree.childLanes = NoLanes

// Reconciler workInProgress = Subtree:
// 1. oldProps === newProps (data ref stable)
// 2. checkScheduledUpdateOrContext → false (Subtree.lanes empty + context unchanged)
// 3. attemptEarlyBailoutIfNoScheduledUpdate
// 4. bailoutOnAlreadyFinishedWork
//    - Subtree.childLanes && renderLanes → empty intersection
//    - return null  ← ENTIRE SUBTREE SKIPPED
```

`childLanes` bo'sh bo'lganda, Reconciler descendant'larga descend qilmaydi — ENTIRE SUB-TREE skip. Bu eng samarali optimizatsiya — minglab Fiber'lar bypass qilinadi bir nechta nanosekundda.

**Context propagation va `childLanes`:**

```typescript
// Provider value o'zgarganda:
function propagateContextChange(workInProgress, context, renderLanes) {
  // Eski (eager) propagation:
  let fiber = workInProgress.child;
  while (fiber !== null) {
    let dep = fiber.dependencies?.firstContext;
    while (dep !== null) {
      if (dep.context === context) {
        // Bu Fiber consumer
        fiber.lanes = mergeLanes(fiber.lanes, renderLanes);
        // Ancestor'larga childLanes
        let parent = fiber.return;
        while (parent !== null) {
          parent.childLanes = mergeLanes(parent.childLanes, renderLanes);
          parent = parent.return;
        }
        break;
      }
      dep = dep.next;
    }
    fiber = nextFiber(fiber);
  }
}
```

R18+ "lazy context propagation" — faqat haqiqatan ham consumer'lar `lanes` belgilanadi. Boshqa Fiber'lar bypass qilinadi.

**Bailout vs Memoization farqi:**

| Konsept | Mexanizm | Cost |
|---------|----------|------|
| Bailout | Fiber render skip (function chaqirilmaydi) | Faqat fiber pointer copy |
| Memoization | Function render bajariladi, sub-tree reused | Render time + reconcile |

`bailout` < `memoization` cost. Bailout function umuman chaqirmaydi — render path butunlay skip qilinadi.

**`didReceiveUpdate` flag:**

```typescript
// Global flag in Reconciler
let didReceiveUpdate: boolean = false;

// beginWork natijasi:
// - true: render full
// - false: bailout possible (lekin context yoki state baribir tekshiriladi)
```

Bu flag har beginWork chaqirig'ida reset qilinadi.

**Fiber traversal o'rni:**

```
Render phase: beginWork (top-down)
              ↓
              completeWork (bottom-up)
```

Bailout `beginWork`'da — top-down. Sub-tree bypass'i tabiiy: children'larga descend qilinmaydi.

**`childLanes` empty optimization rasmi:**

```
App (lanes: SyncLane, childLanes: NoLanes)
├── Subtree (lanes: NoLanes, childLanes: NoLanes)  ← bailout, sub-tree skip
│   ├── Leaf("a")  ← never visited
│   ├── Leaf("b")  ← never visited
│   └── Leaf("c")  ← never visited
└── Button (rendered)
```

Reconciler `Subtree`'ga kelganda — `bailoutOnAlreadyFinishedWork` `null` qaytaradi. Leaf'lar Fiber'lar tashrif qilinmaydi.

</details>

### Edge Cases

- **`current === null` (first mount)**: Bailout impossible — birinchi render majbur. `attemptEarlyBailoutIfNoScheduledUpdate` skip.
- **`hasLegacyContextChanged`**: Class komponent legacy `contextTypes` bilan ishlaganda — bailout disable.
- **`PerformedWork` flag**: Concurrent rendering yoki Suspense bilan — bailout shartlari murakkablashadi (work-in-progress vs current tree).

### Follow-up savollar

- "Bailout va re-render farqini Profiler'da qanday ko'rasiz?" — "Why did this render?" panel — agar render bo'lmasa, komponent ro'yxatda yo'q.
- "Concurrent rendering bailout'ni o'zgartiradimi?" — Yo'q. Bailout — render fazasi optimizatsiyasi. Lekin lanes priority bailout'ning **qachonligi**ga ta'sir qiladi.
- "Aslida bailout ishlamasa nima bo'ladi?" — Default cascade. React 16 dan oldin har setState — full tree re-render. Fiber + lanes + childLanes — incremental rendering imkonini berdi.

</details>

---

<a id="qism-c"></a>

### 9. Output savol — re-render trace [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa savol

```tsx
import { useState, memo, useCallback } from "react";

const ChildA = memo(function ChildA({ name }: { name: string }) {
  console.log("A");
  return <p>{name}</p>;
});

const ChildB = memo(function ChildB({ onClick }: { onClick: () => void }) {
  console.log("B");
  return <button onClick={onClick}>click</button>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState("Ali");

  const handleClick = useCallback(() => {
    console.log("clicked");
  }, []);

  console.log("P");
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>+</button>
      <button onClick={() => setName("Bek")}>change</button>
      <ChildA name={name} />
      <ChildB onClick={handleClick} />
    </>
  );
}
```

**Savol:** Initial mount + 1-button bosilganda + 2-button bosilganda — har holatda log ketma-ketligi qanday bo'ladi?

### Javob

**Initial mount:**
```
P
A
B
```

Sabab: birinchi render — barchasi mount qilinadi.

**1-button (`setCount`) bosilganda:**
```
P
```

Sabab:
- `count` state o'zgardi → Parent re-render
- `name` o'zgarmagan → ChildA props (`{name: "Ali"}`) bir xil → memo bailout
- `handleClick` `useCallback` bilan deps `[]` — stable → ChildB props bir xil → memo bailout

**2-button (`setName`) bosilganda:**
```
P
A
```

Sabab:
- `name` state o'zgardi → Parent re-render
- ChildA props (`{name: "Bek"}`) yangi → memo bailout BUZILADI → render
- ChildB props bir xil → memo bailout → render YO'Q

<details>
<summary><strong>Deep Dive</strong></summary>

**Qadam-baqadam analiz — 1-button click:**

```
1. Click event fired
2. setCount dispatched → Object.is(0, 1) → false → Fiber.lanes = SyncLane
3. scheduleUpdateOnFiber → Scheduler enqueue
4. Browser tick (microtask boundary)
5. Reconciler beginWork from HostRoot
6. App Fiber: beginWork
   - oldProps === newProps (root)
   - lanes empty? NO (childLanes has SyncLane)
   - render → Parent
7. Parent Fiber: beginWork
   - lanes has SyncLane → render full
   - renderWithHooks(Parent, { count: 1, name: "Ali", handleClick: <stable>, ... })
   - JSX evaluates → ChildA element, ChildB element
   - "P" logged
8. ChildA Fiber: beginWork
   - tag = SimpleMemoComponent
   - hasScheduledUpdate? NO (no setState in ChildA)
   - prevProps = { name: "Ali" }
   - nextProps = { name: "Ali" }
   - shallowEqual → true
   - bailout → "A" NOT logged
9. ChildB Fiber: beginWork
   - tag = SimpleMemoComponent
   - prevProps = { onClick: <ref1> }
   - nextProps = { onClick: <ref1> } (useCallback deps [] stable)
   - shallowEqual → true
   - bailout → "B" NOT logged
```

**`useCallback` deps `[]` — stable reference yoki yo'q:**

```typescript
// Mount:
hook.memoizedState = [() => console.log("clicked"), []];
return memoizedState[0]; // ref1

// Re-render (1-button click):
nextDeps = [];
prevDeps = [];
areHookInputsEqual([], []) → true
return prevState[0]; // ref1 (SAME REFERENCE)
```

`useCallback(fn, [])` — har render'da bir xil reference qaytaradi (mount paytidagi). Stable.

**Qadam-baqadam analiz — 2-button click:**

```
1. setName dispatched → Object.is("Ali", "Bek") → false
2. ... (same as before)
3. Parent: render → "P"
   - JSX: jsx(ChildA, { name: "Bek" })  ← name yangi
   - JSX: jsx(ChildB, { onClick: <ref1> })  ← stable
4. ChildA: beginWork
   - prevProps = { name: "Ali" }
   - nextProps = { name: "Bek" }
   - shallowEqual → false
   - render → "A"
5. ChildB: beginWork
   - prevProps = { onClick: <ref1> }
   - nextProps = { onClick: <ref1> }
   - shallowEqual → true
   - bailout
```

**Tricky variation — agar `useCallback` o'rniga inline:**

```tsx
function Parent() {
  const handleClick = () => console.log("clicked"); // ❌ inline
  return <ChildB onClick={handleClick} />;
}

// 1-button click:
// Parent re-render → handleClick yangi reference
// ChildB props { onClick: <NEW REF> } !== old
// shallowEqual → false
// "B" rendered → log: P, B
```

**Tricky variation — agar `name` `useState` initial bir xil:**

```tsx
const [name, setName] = useState("Ali");
// 2-button: setName("Ali")  ← BIR XIL qiymat

// dispatchSetState ichida eager bailout:
// Object.is("Ali", "Ali") → true
// Render TRIGGER QILINMAYDI
// Log: () — bo'sh
```

**Strict Mode dev 2x:**

```tsx
<StrictMode>
  <Parent />
</StrictMode>

// Mount logs:
// P  (1st render)
// P  (2nd render — idempotency check)
// A
// A
// B
// B
```

</details>

### Edge Cases

- **`Object.is` bilan NaN**: `setCount(NaN)` keyin yana `setCount(NaN)` — `Object.is(NaN, NaN) === true` → bailout (no render).
- **Inline arrow uchun `useCallback` deps misss**: `useCallback(() => x, [])` — `x` deps'da yo'q → stale closure. Lint qoidasi (`react-hooks/exhaustive-deps`) tutadi.
- **Render order**: Reconciler depth-first traverse. Sibling order: ChildA dan oldin ChildB.

### Follow-up savollar

- "Concurrent rendering bu trace'ni o'zgartirib yuboradimi?" — Asosan yo'q. Lekin priority lanes har xil bo'lsa (transitions vs sync), render flush'lari boshqacha vaqtda bo'lishi mumkin.
- "Strict Mode 2x render production'da bormi?" — Yo'q. Faqat dev. Production builds 1x render.
- "Bu trace test qilinadimi?" — `@testing-library/react` + `console.log` spy. Yoki React DevTools Profiler.

</details>

---

### 10. Reference equality — inline object/function gotchas [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Inline object (`{a:1}`), array (`[1,2]`), function (`() => x`) — har render yangi reference. `Object.is` comparison fail → `React.memo` bypass, `useEffect` deps loop, `useMemo` cache miss. Yechim: `useMemo`/`useCallback` stabilize, primitive deps, factory functions outside component.

### Kod misoli

```tsx
// ❌ Inline object — memo bypass
const Child = memo(({ config }: { config: { theme: string } }) => {
  return <p>{config.theme}</p>;
});

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child config={{ theme: "dark" }} />  {/* ❌ new object each render */}
    </>
  );
}

// ✅ Stable
function ParentGood() {
  const [count, setCount] = useState(0);
  const config = useMemo(() => ({ theme: "dark" }), []);  // memo
  // Or: define outside
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child config={config} />
    </>
  );
}

// Or — outside component
const CONFIG = { theme: "dark" };
function ParentBest() {
  return <Child config={CONFIG} />;  // module-level, truly stable
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Reference instability sources:**

```tsx
// 1. Object literal
<Child obj={{ a: 1 }} />

// 2. Array literal
<Child arr={[1, 2, 3]} />

// 3. Inline function
<Child onClick={() => doX()} />

// 4. Computed
<Child user={users.find(u => u.id === id)} />

// 5. Spread
<Child {...{ a: 1, b: 2 }} />
```

**`useEffect` dep loop:**

```tsx
function MyComponent({ filters }) {
  useEffect(() => {
    fetchData(filters);
  }, [filters]);  // ⚠️ filters from parent — may be inline object
}

// Parent passes inline:
<MyComponent filters={{ active: true }} />
// Each render new filters → useEffect runs each render → infinite loop possible
```

**Compiler auto-stabilize:**

```tsx
// Pre-Compiler
function Parent() {
  const config = { theme: "dark" };  // new each render
  return <Child config={config} />;
}

// Post-Compiler (auto-memo)
function Parent() {
  const $ = useMemoCache(1);
  const config = $[0] ?? ($[0] = { theme: "dark" });
  return <Child config={config} />;
}
```

**Deep equality alternative:**

```tsx
// Custom memo with deep compare
import { isEqual } from "lodash";
const Child = memo(MyComponent, (prev, next) => isEqual(prev.config, next.config));

// ⚠️ Trade-off: deep equal cost > render cost (ko'p hollarda)
// Use only for expensive renders
```

**Function stability via reducer:**

```tsx
function Parent() {
  // ❌ New function each render
  const [count, setCount] = useState(0);
  const handler = (newCount: number) => setCount(newCount);

  // ✅ Stable via dispatch from useReducer
  const [count, dispatch] = useReducer(reducer, 0);
  // dispatch reference stable across renders
}
```

</details>

### Edge Cases

- **Primitive props**: Number, string, boolean — `Object.is` ishlaydi. Muammo yo'q.
- **`null`/`undefined`**: Doim bir xil reference (`Object.is` true).
- **Nested object o'zgarishi**: Tashqi reference o'zgarmagan — `Object.is` true → memo bailout (stale UI xavfi).

### Follow-up savollar

- "React nima uchun deep compare qilmaydi?" — Performance: shallow O(n keys), deep O(tree size). Default arzon.
- "Deep compare qachon to'g'ri keladi?" — Qimmat child render (katta list, murakkab hisoblash). Avval Profiler bilan o'lchang.

</details>

---

## QISM C: useMemo / useCallback Usage

### 11. Qachon `useMemo`/`useCallback` ishlatish kerak (decision tree)? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useMemo`/`useCallback` quyidagi 3 ta haqiqiy holatda foydali: (1) **Hisoblash qimmat** — `O(n)`+ kompleksligi, ko'p elementlar (sort, filter, complex map). (2) **Reference identity muhim** — `memo`'lashtirilgan child'ga props sifatida uzatiladigan object/array/function. (3) **`useEffect`/`useMemo` deps'ida** — deps array'i o'zgaruvchanligini boshqarish. Boshqa holatlarda — over-engineering. R19 React Compiler avtomat memoization qiladi → manual ishlatish ehtiyoji kamayadi.

### To'liq tushuntirish

**Decision tree:**

```
useMemo/useCallback kerakmi?
├── 1. Hisoblash qimmatmi (1ms+)?
│   ├── HA → useMemo
│   └── YO'Q → keyingisiga
├── 2. Memo'langan child'ga uzatiladimi?
│   ├── HA → useCallback (function) yoki useMemo (object/array)
│   └── YO'Q → keyingisiga
├── 3. useEffect deps'ida ishlatiladimi?
│   ├── HA, va deps stability talab qilinadi → useCallback/useMemo
│   └── YO'Q → keyingisiga
└── Kerakmas → oddiy const/inline
```

**1. Qimmat hisoblash uchun:**

```tsx
function ProductList({ products }: { products: Product[] }) {
  const [filter, setFilter] = useState("");

  // ✅ Filter + sort — katta dataset'lar (1K+) uchun sezilarli vaqt olishi mumkin
  const visibleProducts = useMemo(() => {
    return products
      .filter((p) => p.name.includes(filter))
      .sort((a, b) => a.price - b.price);
  }, [products, filter]);

  return <ul>{visibleProducts.map((p) => <Item key={p.id} {...p} />)}</ul>;
}

// Filter o'zgarmasa — qimmat compute qayta bajarilmaydi
```

**2. Memo'langan child uchun:**

```tsx
const Chart = memo(function Chart({ data, options }: ChartProps) {
  // Heavy chart render
  return <canvas ref={...} />;
});

function Dashboard() {
  const [count, setCount] = useState(0);

  // ✅ data va options — Chart props uchun stable kerak
  const data = useMemo(() => fetchData(), []);
  const options = useMemo(() => ({ width: 800, height: 600 }), []);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Chart data={data} options={options} />
    </>
  );
}
// count++ bosilganda Chart qayta render YO'Q (props stable)
```

**3. `useEffect` deps:**

```tsx
function SearchBox({ onSearch }: { onSearch: (q: string) => void }) {
  const [query, setQuery] = useState("");

  // onSearch parent'dan keladi — deps'da bo'lishi kerak
  useEffect(() => {
    const timer = setTimeout(() => onSearch(query), 300);
    return () => clearTimeout(timer);
  }, [query, onSearch]);

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}

// Parent:
function App() {
  const [results, setResults] = useState<string[]>([]);

  // ✅ useCallback — onSearch reference stable
  const handleSearch = useCallback((q: string) => {
    fetch(`/search?q=${q}`).then(r => r.json()).then(setResults);
  }, []);

  return <SearchBox onSearch={handleSearch} />;
}
// handleSearch yangi reference bo'lsa — useEffect har render qayta ishlaydi (debounce buziladi)
```

### Kod misoli

**Anti-pattern — har joyga useMemo qo'shish:**

```tsx
function BadDashboard({ user }: { user: User }) {
  // ❌ Trivial computation — over-engineering
  const greeting = useMemo(() => `Hello, ${user.name}`, [user.name]);

  // ❌ Primitive — useMemo ortiqcha
  const isAdmin = useMemo(() => user.role === "admin", [user.role]);

  // ❌ Inline JSX literal — render uchun nyu element baribir yaratiladi
  const header = useMemo(() => <h1>{greeting}</h1>, [greeting]);

  return (
    <div>
      {header}
      {isAdmin && <p>Admin panel</p>}
    </div>
  );
}

// useMemo cost > render cost
// Hook overhead: deps comparison + memoization slot
```

**To'g'ri yondashuv:**

```tsx
function GoodDashboard({ user }: { user: User }) {
  const greeting = `Hello, ${user.name}`;        // ✅ Pure expression
  const isAdmin = user.role === "admin";          // ✅ Primitive comparison

  return (
    <div>
      <h1>{greeting}</h1>
      {isAdmin && <p>Admin panel</p>}
    </div>
  );
}
```

**Real qimmat hisoblash:**

```tsx
function DataGrid({ rows }: { rows: Row[] }) {
  const [sortBy, setSortBy] = useState<keyof Row>("id");
  const [filterText, setFilterText] = useState("");

  // ✅ Katta dataset × filter + sort — har keystroke'da qayta hisoblash qimmat
  const processed = useMemo(() => {
    return rows
      .filter((r) =>
        Object.values(r).some((v) =>
          String(v).toLowerCase().includes(filterText.toLowerCase()),
        ),
      )
      .sort((a, b) => {
        const av = a[sortBy];
        const bv = b[sortBy];
        return av < bv ? -1 : av > bv ? 1 : 0;
      });
  }, [rows, sortBy, filterText]);

  return (
    <table>
      <tbody>
        {processed.map((r) => <Row key={r.id} {...r} />)}
      </tbody>
    </table>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`useMemo` ichidagi mexanizm:**

```typescript
// react/src/ReactHooks.js (mental model)
function mountMemo<T>(nextCreate: () => T, deps: Array<unknown> | null): T {
  const hook = mountWorkInProgressHook();
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, deps];
  return nextValue;
}

function updateMemo<T>(nextCreate: () => T, deps: Array<unknown> | null): T {
  const hook = updateWorkInProgressHook();
  const prevState = hook.memoizedState as [T, Array<unknown> | null];

  if (prevState !== null && deps !== null) {
    const prevDeps = prevState[1];
    if (areHookInputsEqual(deps, prevDeps)) {
      return prevState[0]; // OLD VALUE
    }
  }

  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, deps];
  return nextValue;
}
```

**`areHookInputsEqual`:**

```typescript
function areHookInputsEqual(
  nextDeps: Array<unknown>,
  prevDeps: Array<unknown> | null,
): boolean {
  if (prevDeps === null) return false;
  if (nextDeps.length !== prevDeps.length) return false;
  for (let i = 0; i < prevDeps.length; i++) {
    if (Object.is(nextDeps[i], prevDeps[i])) continue;
    return false;
  }
  return true;
}
```

`Object.is` har dep uchun — primitive teng yoki same reference. Object/array deps — reference comparison.

**`useMemo` cache size 1 — eng oxirgi qiymat:**

`useMemo(create, [a])` — `a` har xil bo'lsa, har gal `create()` qayta bajariladi (cache LRU emas, faqat oxirgi 1 ta saqlanadi). Bu **`React.cache`** (RSC) yoki **`useDeferredValue`** + custom memoization'dan farq qiladi.

**Cost-Benefit analysis:**

```tsx
// useMemo cost komponentlari:
// 1. Hook slot allocation (mount) — Fiber linked list'ga yangi node
// 2. Hook slot read (update) — pointer traversal
// 3. areHookInputsEqual: O(deps.length) × Object.is
// 4. memoizedState update (cache miss): allocation overhead

// Net: deps tekshiruv + cached value qaytarish overhead'i bor.
// Kichik computation uchun bu overhead asl computation'dan qimmat bo'lishi mumkin.
```

**Kichik komponent uchun:**

```tsx
function Trivial({ name }: { name: string }) {
  // String concat o'z-o'zidan tezdir.
  // useMemo qo'shsak — hook overhead asl computation'dan qimmat bo'ladi.
  const greeting = useMemo(() => `Hello, ${name}`, [name]);
  return <p>{greeting}</p>;
}

// vs:
function Trivial({ name }: { name: string }) {
  return <p>Hello, {name}</p>;
}
```

Trivial computation uchun `useMemo` — performance LOSS bo'lishi mumkin. Profiler bilan tasdiqlang.

**`useMemo` haqiqiy use case:**

| Foydali | Foydasiz |
|---------|----------|
| Filter/sort 1K+ elementlar | String concat |
| Date formatting (Intl.DateTimeFormat) | Boolean check |
| `JSON.parse` 10KB+ | Number arithmetic |
| `crypto.subtle.digest` | Property access |
| Regex compilation | Inline JSX (memo'lashtirilmagan child) |

**`useCallback` haqiqiy use case:**

`useCallback(fn, deps)` === `useMemo(() => fn, deps)`. Foydali:
- Memo'langan child'ga uzatiladigan handler
- `useEffect` deps'ida bo'lgan handler
- Custom hook'dan qaytariladigan stable handler

</details>

### Edge Cases

- **`useMemo` deps `undefined`**: `useMemo(create)` — deps yo'q → har render fresh (ortiqcha). Lint warning beradi.
- **`useMemo` non-pure factory**: `useMemo(() => Math.random(), [])` — pure emas, lekin React kafolat bermaydi (cache evict qilishi mumkin Strict Mode dev'da).
- **`useMemo` semantic guarantee yo'q**: React docs aytadi: "Treat useMemo as a performance hint, not a guarantee." Cache evict bo'lishi mumkin (kelajakda).

### Follow-up savollar

- "`useMemo` value cache evict bo'ladimi?" — Hozircha (R19) yo'q. Lekin React docs explicit: future versiyalarda cache eviction bo'lishi mumkin. Logic'ni hech qachon faqat memoization'ga tayanmang.
- "`React.useMemo` vs `useMemo` farqi bormi?" — Yo'q. `React.useMemo` named export ekvivalenti.
- "`useMemo`'ni qachon `useState` bilan almashtirish kerak?" — Agar value lifecycle mustaqil bo'lsa va explicit reset talab qilsa — `useState`. `useMemo` — derived value uchun.

</details>

---

<a id="qism-d"></a>

### 12. `useMemo`/`useCallback` qachon ortiqcha (over-engineering)? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useMemo`/`useCallback` quyidagi holatlarda **ortiqcha**: (1) Trivial hisoblash (primitive arifmetika, string concat, property access). (2) Komponent kichik va memo'lashtirilmagan child'larga uzatilmaydi. (3) Deps `[]` bo'lib, value barcha render'larda bir xil — `const` outside component yetadi. (4) Inline JSX'ni cache qilish (`useMemo(() => <X />, [...])` — element re-creation cost minimal). Hook'larning o'zi memory + comparison overhead qo'shadi. R19 Compiler bu kabi holatlarni avtomat hal qiladi.

### To'liq tushuntirish

**Anti-pattern 1 — Trivial computation:**

```tsx
// ❌ Over-engineering
function Component({ price }: { price: number }) {
  const formatted = useMemo(() => `$${price}`, [price]);
  const isExpensive = useMemo(() => price > 100, [price]);
  const halfPrice = useMemo(() => price / 2, [price]);
  return <p>{formatted}, half: {halfPrice}, exp: {isExpensive ? "y" : "n"}</p>;
}

// 3 ta useMemo qo'shimcha hook overhead'i (slot allocation, deps check) keltiradi.
// Asl computation (string template, comparison, arithmetic) o'z-o'zidan tez —
// memoization net foyda bermaydi, faqat overhead qo'shadi.

// ✅ Inline
function Component({ price }: { price: number }) {
  const formatted = `$${price}`;
  const isExpensive = price > 100;
  const halfPrice = price / 2;
  return <p>{formatted}, half: {halfPrice}, exp: {isExpensive ? "y" : "n"}</p>;
}
```

**Anti-pattern 2 — Static value with `useMemo([])`:**

```tsx
// ❌ Hook overhead bekorga
function Component() {
  const config = useMemo(() => ({ theme: "dark", lang: "en" }), []);
  return <Provider value={config}>...</Provider>;
}

// ✅ Module scope const
const APP_CONFIG = { theme: "dark", lang: "en" };

function Component() {
  return <Provider value={APP_CONFIG}>...</Provider>;
}
```

**Anti-pattern 3 — Inline JSX `useMemo`:**

```tsx
// ❌ Element re-creation odatda arzon
function Component({ count }: { count: number }) {
  const header = useMemo(() => <h1>Header</h1>, []);
  return <>{header}<p>{count}</p></>;
}

// React element creation (`jsx()` chaqiruvi + POJO allocation) — arzon.
// useMemo qo'shimcha hook overhead va deps tekshiruvi qo'shadi.
// Element static bo'lsa ham, memoization deyarli net foyda bermaydi.

// ✅ Inline
function Component({ count }: { count: number }) {
  return <><h1>Header</h1><p>{count}</p></>;
}
```

Element re-creation arzon — `jsx()` chaqiruvi va POJO allocation. Memoization'ning amortized benefit yo'q.

**Anti-pattern 4 — `useCallback` `memo` bo'lmagan child uchun:**

```tsx
// ❌ Child memo'lashtirilmagan — useCallback ortiqcha
function Component() {
  const handleClick = useCallback(() => console.log("click"), []);
  return <button onClick={handleClick}>Click</button>;
  // <button> — DOM element, "memo" konsepsiyasi yo'q
  // Stable reference DOM event listener uchun ahamiyatsiz
}

// ✅
function Component() {
  return <button onClick={() => console.log("click")}>Click</button>;
}
```

**Anti-pattern 5 — Premature optimization "har holda":**

```tsx
// ❌ Hech qachon profiling qilmadan
function Form() {
  const handleSubmit = useCallback(...);
  const fields = useMemo(() => [...], [...]);
  const validation = useMemo(() => createValidator(...), [...]);
  // ...
}
```

> "Premature optimization is the root of all evil" — Donald Knuth (Tony Hoare)

Profiling **avval**, optimizatsiya **keyin**.

### Kod misoli

**Tipik holatlarda `useMemo` ortiqcha:**

```tsx
function ProductCard({ product }: { product: Product }) {
  // ❌ Barcha quyidagilar over-engineering
  const name = useMemo(() => product.name.toUpperCase(), [product.name]);
  const priceStr = useMemo(() => `$${product.price}`, [product.price]);
  const isOnSale = useMemo(() => product.price < product.originalPrice, [product.price, product.originalPrice]);

  return (
    <div>
      <h3>{name}</h3>
      <p>{priceStr}</p>
      {isOnSale && <span>Sale!</span>}
    </div>
  );
}

// ✅ Oddiy
function ProductCard({ product }: { product: Product }) {
  const name = product.name.toUpperCase();
  const priceStr = `$${product.price}`;
  const isOnSale = product.price < product.originalPrice;

  return (
    <div>
      <h3>{name}</h3>
      <p>{priceStr}</p>
      {isOnSale && <span>Sale!</span>}
    </div>
  );
}
```

**Stable callback'lar tashqarida:**

```tsx
// ❌ Component ichida useCallback
function App() {
  const formatDate = useCallback((d: Date) => d.toISOString(), []);
  return <DatePicker formatter={formatDate} />;
}

// ✅ Module scope — eng tabiy stable
function formatDate(d: Date): string {
  return d.toISOString();
}

function App() {
  return <DatePicker formatter={formatDate} />;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Hook overhead — sifat tahlili:**

| Hook | Mount cost | Update cost |
|------|-----------|-------------|
| `useState` | Slot allocation + initial value | Pointer traversal + Object.is check |
| `useMemo` | Slot + deps array + create() call | Slot read + areHookInputsEqual + cache return/miss |
| `useCallback` | Slot + deps array + reference store | Slot read + areHookInputsEqual + cache return/miss |
| `useEffect` | Slot + create()/destroy registration | Slot + deps check + commit phase scheduling |

Aniq raqamlar engine (V8/JSC/SpiderMonkey), build mode (DEV/PROD), va kontekstga bog'liq. Minglab hook'lar bilan komponent'larda summa sezilarli bo'lishi mumkin. O'z scenarioda Profiler bilan o'lchang.

**Cost > Benefit holatlari:**

```typescript
// Inline computation: `${a}-${b}` — JS engine'lar uchun optimized.
// useMemo: cache check (deps comparison) + create() chaqiruvi (cache miss'da).
// Cache HIT'da ham hook slot read va Object.is taqqoslash bor.
//
// Trivial computation uchun memoization overhead'i hisoblashning o'zidan qimmat
// bo'lib qolishi mumkin. Aniq solishtirish uchun Profiler.

// Hook'siz: faqat string concat har render
// useMemo: slot read + Object.is(deps) + cache return/miss path
// Cache HIT'da ham hook overhead bor. Trivial computation uchun
// useMemo doim sekinroq bo'lishi mumkin — over-engineering.
```

**`React.Compiler` (R19) approach:**

```typescript
// Compiler input:
function Component({ products }) {
  const filtered = products.filter(p => p.active);
  return <List items={filtered} />;
}

// Compiler output (mental model):
function Component({ products }) {
  const $ = useMemoCache(2);
  const t0 = $[0] !== products;
  const filtered = t0 ? products.filter(p => p.active) : $[1];
  if (t0) {
    $[0] = products;
    $[1] = filtered;
  }
  return jsx(List, { items: filtered });
}
```

Compiler statik analiz orqali avtomat memoization qiladi — manual `useMemo` ortiqcha bo'lib qoladi.

**Trade-off — readability vs perf:**

```tsx
// ✅ Readable
const result = data.filter(...).sort(...).map(...);

// ❌ Hook noise
const filtered = useMemo(() => data.filter(...), [data]);
const sorted = useMemo(() => filtered.sort(...), [filtered]);
const mapped = useMemo(() => sorted.map(...), [sorted]);
```

3 ta `useMemo` — 3 ta hook slot, 3 ta deps array, 3 ta cache check. Performance benefit minimal (kichik data uchun), lekin readability sezilarli kamayadi.

**`useMemo` correctness guarantee yo'q:**

React docs: "useMemo should be used as a performance optimization, not as a semantic guarantee." Cache evict bo'lishi mumkin (kelajakda Offscreen component yoki Strict Mode dev'da). Logic'ni faqat cache'ga tayanmang.

```tsx
// ❌ ID generator memoization — cache evict bo'lsa duplicate ID
const id = useMemo(() => generateId(), []);

// ✅ useState — guaranteed stable
const [id] = useState(() => generateId());
// Yoki:
const [id] = useState(generateId); // initializer
```

</details>

### Edge Cases

- **`useMemo` `[]` o'rniga `useState` initializer**: Stable values uchun `useState(() => initialValue)` — guaranteed stability. `useMemo` — performance hint.
- **Hook count yuqori limit**: Komponent juda ko'p Hooks bilan to'lib ketsa — har biri overhead. 50+ Hooks — komponentni split qilish.
- **`useMemo` returning component**: `useMemo(() => <Child />, deps)` — element cached, lekin Child funksiyasi qayta render qilinmaydi (Reconciler element identity check'i bor). Bu sub-tree skip imkonini beradi.

### Follow-up savollar

- "Profiling qaysi vositalar bilan?" — React DevTools Profiler (component-level), Chrome DevTools Performance (browser-level), `performance.measure` (custom marks).
- "Compiler ishlatilsa qadamlardan foydalanmaslik mumkinmi?" — Ha. Compiler avtomat memoization → `useMemo`/`useCallback` qo'shish kerak emas. Ammo `useEffect` deps stability uchun ba'zan kerak qoladi.
- "Premature optimization aniq belgilari?" — Profiling qilinmaganda; faqat "ehtimol" gipoteza bilan; readability sezilarli kamayganda.

</details>

---

### 13. React Compiler (R19) avtomat memoization'ni qanday hal qiladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React Compiler — Babel'ga o'xshash build-time tool. Komponent kodini analyse qilib, har **derived value** va **callback** uchun avtomat memoization qo'shadi (cache slot + deps check). Natija: `useMemo`/`useCallback` qo'l bilan yozish kerak emas. Compiler "Rules of React" (purity, immutability) ga rioya qilingan kodni o'zgartiradi; qoidabuzar kod (mutation, side effects in render) — skip qiladi (graceful fallback). R19 da experimental → opt-in via `babel-plugin-react-compiler`.

### To'liq tushuntirish

**Compiler input/output:**

```tsx
// Input (developer wrote):
function ProductList({ products, filter }: Props) {
  const filtered = products.filter(p => p.name.includes(filter));
  const handleClick = (id: string) => console.log(id);
  return (
    <ul>
      {filtered.map(p => <Item key={p.id} product={p} onClick={handleClick} />)}
    </ul>
  );
}

// Output (compiler generated, mental model):
function ProductList({ products, filter }: Props) {
  const $ = useMemoCache(4);

  // Memoize filtered
  let t0;
  if ($[0] !== products || $[1] !== filter) {
    t0 = products.filter(p => p.name.includes(filter));
    $[0] = products;
    $[1] = filter;
    $[2] = t0;
  } else {
    t0 = $[2];
  }
  const filtered = t0;

  // Memoize handleClick (no deps — stable)
  let t1;
  if ($[3] === undefined) {
    t1 = (id) => console.log(id);
    $[3] = t1;
  } else {
    t1 = $[3];
  }
  const handleClick = t1;

  // ... JSX
}
```

**`useMemoCache` — Compiler runtime:**

```typescript
// React internal
function useMemoCache(size: number): Array<unknown> {
  const dispatcher = ReactCurrentDispatcher.current;
  return dispatcher.useMemoCache(size);
}

// Hook implementation:
function mountUseMemoCache(size: number): Array<unknown> {
  const hook = mountWorkInProgressHook();
  const cache = new Array(size).fill(REACT_MEMO_CACHE_SENTINEL);
  hook.memoizedState = cache;
  return cache;
}
```

Cache — array bilan slot'lar. Har slot uchun deps va value alternativ saqlanadi.

**Compiler "Rules of React" talab qiladi:**

1. **Purity**: Render bo'lib bajarilayotganda — no side effects (no DOM mutation, no fetch, no setState in body)
2. **Immutability**: Props/state mutate qilinmaydi
3. **Hooks rules**: Top-level only, no conditional/loop hooks
4. **Single responsibility**: Komponent function — render uchun, hook'lar — alohida concerns

Qoidabuzar kod — Compiler skip qiladi (warning emit, fallback to standard render).

### Kod misoli

**Compiler bilan/siz farqi:**

```tsx
// ❌ Manual (compiler'siz)
function Dashboard({ data, filter }: Props) {
  const filtered = useMemo(
    () => data.filter(d => d.matches(filter)),
    [data, filter]
  );

  const sortedFiltered = useMemo(
    () => [...filtered].sort((a, b) => a.priority - b.priority),
    [filtered]
  );

  const handleSelect = useCallback(
    (id: string) => console.log(`Selected: ${id}`),
    []
  );

  const config = useMemo(
    () => ({ theme: "dark", showActions: true }),
    []
  );

  return <Table rows={sortedFiltered} config={config} onSelect={handleSelect} />;
}

// ✅ Compiler bilan
function Dashboard({ data, filter }: Props) {
  const filtered = data.filter(d => d.matches(filter));
  const sortedFiltered = [...filtered].sort((a, b) => a.priority - b.priority);
  const handleSelect = (id: string) => console.log(`Selected: ${id}`);
  const config = { theme: "dark", showActions: true };

  return <Table rows={sortedFiltered} config={config} onSelect={handleSelect} />;
}
// Compiler avtomat: filtered, sortedFiltered, handleSelect, config — barchasi memoized
```

**Setup (R19 experimental):**

```bash
npm install babel-plugin-react-compiler
```

`babel.config.js`:

```javascript
module.exports = {
  plugins: [
    ['babel-plugin-react-compiler', {
      target: '19', // React 19
    }],
  ],
};
```

Vite/Next.js — alohida integratsiya plugin'lari mavjud.

**Opt-out — directive:**

```tsx
"use no memo";

function MyComponent({ data }: Props) {
  // Compiler shu komponentni skip qiladi
  return <div>{data}</div>;
}
```

Foydali: legacy code yoki bug uchragan komponent.

<details>
<summary><strong>Deep Dive</strong></summary>

**Compiler architecture:**

```
Source (.tsx) → Babel parse → AST
                    ↓
            React Compiler analyze
                    ↓
        ┌───────────┴───────────┐
        Pure?                  Impure?
        ↓                       ↓
   Memoize transforms       Skip (warning)
        ↓                       ↓
        └───────────┬───────────┘
                    ↓
            Babel print → Output (.js)
```

**Compiler analyze qadamlar:**

1. **HIR construction** — Compiler intermediate representation (control flow graph)
2. **Mutation analysis** — qaysi value'lar mutate qilinishini aniqlash
3. **Reactive scope detection** — qaysi expression'lar reactive (deps'ga bog'liq)
4. **Memoization placement** — har reactive scope uchun cache slot
5. **Code generation** — `useMemoCache` calls bilan output

**Reactive scopes:**

```tsx
function Component({ a, b, c }: Props) {
  const x = a + b;       // Reactive scope: { deps: [a, b], output: x }
  const y = c * 2;       // Reactive scope: { deps: [c], output: y }
  const z = x + y;       // Reactive scope: { deps: [x, y], output: z }
  return <p>{z}</p>;
}

// Compiler granular memoization — har "scope" alohida cache
```

**`a`, `b` o'zgargan: x va z qayta hisoblanadi, y emas.**
**Faqat `c` o'zgargan: y va z qayta hisoblanadi, x emas.**

**Mutation tracking:**

```tsx
function Component({ items }: Props) {
  const list = items.map(i => i.id);
  list.push("extra"); // ❌ Mutation
  return <ul>{list.map(...)}</ul>;
}

// Compiler aniqlaydi: list mutate qilinadi
// Memoization SKIP — har render fresh
// Warning emit (lint rule)
```

**Function component vs Hooks:**

Compiler komponentlarni va custom hook'larni transform qiladi (function name `use`* prefix yoki `Component` PascalCase).

```tsx
// Compiler memoizes:
function MyComponent() { ... }      // ✅ Komponent
function useCustomHook() { ... }    // ✅ Hook

// Compiler skip:
function helperFunction() { ... }   // ❌ Plain function — skip
const obj = { method() { ... } };  // ❌ Method — skip
```

**`useMemoCache` slot allocation:**

Compiler har component uchun maximum slot count'ni aniqlaydi (worst-case). Mount paytida cache array shu o'lchamda yaratiladi. Misol: 4 ta reactive scope → cache size 8 (deps + value har biri uchun).

**Bug uchragan misol — Compiler conservativeness:**

```tsx
function Component({ user }: Props) {
  // Compiler: bu mutation? Aniq emas → conservative skip
  user.lastSeen = Date.now(); // ❌ Mutation — Compiler memoization yo'q

  return <p>{user.name}</p>;
}
```

`user` props ekanligi va mutate qilinishi aniq bo'lsa, Compiler memoization'ni skip qiladi (xavfsiz fallback).

**Compiler bug detection — ESLint integration:**

```bash
npm install eslint-plugin-react-compiler
```

`.eslintrc`:

```javascript
{
  "plugins": ["react-compiler"],
  "rules": {
    "react-compiler/react-compiler": "error",
  },
}
```

Mutation, side effects, va Rules of React buzilishi linting orqali aniqlash imkonini beradi.

> **Performance reality:** Compiler manual memoization'dan tezroq emas — bir xil mexanizm (cache slots). Lekin developer ergonomics — `useMemo`/`useCallback` boilerplate yo'qoladi. Code review'da fokus business logic'ga.

</details>

### Edge Cases

- **`useMemo` qo'lda yozilgan + Compiler enabled**: Compiler ortiqcha `useMemo`'ni o'chirmaydi (transform'ga ta'sir qilmaydi). Ammo Compiler o'zining wrapping'ini qo'shmaydi (already memoized).
- **Conditional component — `if (cond) return ...`**: Compiler conditional return support qiladi, lekin reactive scope'lar mos bo'lishi shart.
- **Closure'lar va effect'lar**: Compiler effect callback'larni ham memoize qiladi (deps avtomat aniqlash). Lekin `useEffect` deps'ni baribir noto'g'ri bo'lsa — Compiler tuzatmaydi.

### Follow-up savollar

- "Compiler R19 stable holatdami?" — R19 da `experimental` channel'da, production'da ishlatish boshlandi (Meta — Instagram, Facebook). Stable release — kelajakda.
- "Compiler vs `useMemo`/`useCallback` performance farqi bormi?" — Bir xil mexanizm — `useMemoCache`. Performance teng. Compiler — DX (developer experience) afzalligi.
- "Migration strategy?" — (1) Compiler enable, (2) ESLint plugin enable, (3) Lint warnings fix, (4) Manual `useMemo`/`useCallback` ni asta-sekin olib tashlash.

</details>

---

## QISM D: React Compiler

### 14. React Compiler ichki mexanizmi qanday ishlaydi (HIR, reactive scopes)? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React Compiler — Babel-based AOT (Ahead Of Time) optimizer. Komponent kodini parse qilib, **HIR (High-level Intermediate Representation)** ga aylantiradi — bu control flow graph'ga o'xshash struktura. Keyin **reactive scopes** aniqlanadi (ifoda + uning input deps + output value). Har scope uchun `useMemoCache` slot ajratiladi va memoization kod insert qilinadi. Final output — original kod + cache wrapping. Source map'lar saqlanadi (debugging uchun).

### To'liq tushuntirish

**Compiler pipeline:**

```
Input: TSX source
  ↓
1. Parse (Babel) → AST
  ↓
2. Build HIR (Compiler IR)
  ↓
3. Inference passes:
   - Type inference
   - Mutability analysis (qaysi values mutate qilinadi)
   - Aliasing analysis (qaysi values shared)
  ↓
4. Reactive scope detection
  ↓
5. Memoization placement
  ↓
6. Code generation (back to AST)
  ↓
7. Print (Babel) → Output JS
```

**HIR (mental model):**

```typescript
// Source:
function Component({ x, y }) {
  const sum = x + y;
  return <p>{sum}</p>;
}

// HIR (simplified):
{
  blocks: [
    {
      id: "bb0",
      instructions: [
        { lvalue: "x", op: "PropertyLoad", object: "props", property: "x" },
        { lvalue: "y", op: "PropertyLoad", object: "props", property: "y" },
        { lvalue: "sum", op: "BinaryOp", left: "x", right: "y", operator: "+" },
        { lvalue: "p", op: "JSX", type: "p", children: ["sum"] },
        { op: "Return", value: "p" },
      ],
    },
  ],
}
```

**Reactive scopes:**

Reactive scope — input → output bog'liqligi:

```typescript
// Reactive scope:
{
  deps: ["x", "y"],
  output: "sum",
  range: { start: instr2, end: instr2 },
}

// Compiler shu scope'ni cache qiladi:
let sum;
if (cache[0] !== x || cache[1] !== y) {
  sum = x + y;
  cache[0] = x;
  cache[1] = y;
  cache[2] = sum;
} else {
  sum = cache[2];
}
```

**Granularity — fine-grained vs coarse-grained:**

```tsx
function Component({ a, b, c }: Props) {
  const x = a * 2;        // Scope 1: deps [a]
  const y = b + 1;        // Scope 2: deps [b]
  const z = x + y;        // Scope 3: deps [x, y] (transitively a, b)
  return <Card data={z} title={c} />;  // Scope 4: deps [z, c]
}
```

Compiler 4 ta alohida scope yaratadi — har biri o'zining cache slot'lari bilan. `c` o'zgarganda faqat scope 4 qayta hisoblanadi.

**Compiler vs manual memoization:**

| Aspect | Manual | Compiler |
|--------|--------|----------|
| Granularity | Developer choice | Avtomat fine-grained |
| Boilerplate | High (`useMemo`/`useCallback`) | Zero |
| Errors | Easy to forget deps | Avtomat |
| Refactoring cost | Yuqori (har o'zgarish — deps update) | Past |
| Build cost | Yo'q | Compile-time overhead |

### Kod misoli

**Granular memoization Compiler bilan:**

```tsx
// Source
function Profile({ user, posts }: { user: User; posts: Post[] }) {
  const fullName = `${user.firstName} ${user.lastName}`;
  const recentPosts = posts.filter(p => p.timestamp > Date.now() - 86400000);
  const summary = `${user.firstName} has ${recentPosts.length} recent posts`;

  return (
    <div>
      <h1>{fullName}</h1>
      <p>{summary}</p>
      <PostList posts={recentPosts} />
    </div>
  );
}

// Compiler output (mental model)
function Profile({ user, posts }: { user: User; posts: Post[] }) {
  const $ = useMemoCache(8);

  // Scope 1: fullName
  let fullName;
  const c0 = $[0] !== user.firstName || $[1] !== user.lastName;
  if (c0) {
    fullName = `${user.firstName} ${user.lastName}`;
    $[0] = user.firstName;
    $[1] = user.lastName;
    $[2] = fullName;
  } else {
    fullName = $[2];
  }

  // Scope 2: recentPosts
  let recentPosts;
  const c1 = $[3] !== posts;
  if (c1) {
    recentPosts = posts.filter(p => p.timestamp > Date.now() - 86400000);
    $[3] = posts;
    $[4] = recentPosts;
  } else {
    recentPosts = $[4];
  }

  // Scope 3: summary (deps recentPosts.length, user.firstName)
  let summary;
  const c2 = $[5] !== user.firstName || $[6] !== recentPosts.length;
  if (c2) {
    summary = `${user.firstName} has ${recentPosts.length} recent posts`;
    $[5] = user.firstName;
    $[6] = recentPosts.length;
    $[7] = summary;
  } else {
    summary = $[7];
  }

  // JSX scopes ham memoize qilinadi (omitted for brevity)
  return (
    <div>
      <h1>{fullName}</h1>
      <p>{summary}</p>
      <PostList posts={recentPosts} />
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**HIR — control flow:**

```tsx
function Component({ items, filter }) {
  if (filter) {
    return <FilterView items={items} filter={filter} />;
  }
  return <DefaultView items={items} />;
}

// HIR control flow graph:
// bb0 (entry):
//   t1 = props.filter
//   if t1 goto bb1 else bb2
// bb1:
//   t2 = props.items
//   t3 = JSX(FilterView, items=t2, filter=t1)
//   return t3
// bb2:
//   t4 = props.items
//   t5 = JSX(DefaultView, items=t4)
//   return t5
```

Branch'lar bo'yicha Compiler reactive scopes'ni har branch'da alohida tahlil qiladi.

**Mutability analysis — frozen vs mutable:**

```typescript
// Compiler tracks:
type Effect =
  | "Read"        // user.name (qiymat o'qilgan)
  | "Mutate"      // user.name = value (qiymat o'zgartirilgan)
  | "Capture"     // closure'da olingan
  | "Store";      // boshqa joyga assigned

// Mutation aniqlanganda — memoization unsafe
const obj = { count: 0 };
obj.count++;       // Mutate effect → Compiler memoization skip
```

**Aliasing analysis — shared values:**

```tsx
function Component({ items }: Props) {
  const list = items;  // ← alias
  list.push("new");    // mutates `items` indirectly
  return <ul>...</ul>;
}

// Compiler tracks: list aliases items
// `items.push` da mutation — props mutated → invalidate
```

**Type inference:**

Compiler type inference TypeScript types'ga tayanmaydi (alohida system). Lekin TypeScript types deklaratsiyalari qulay bo'ladi (lint warnings).

**Effect kinds:**

```typescript
enum Effect {
  Capture,      // Stored, lekin mutate qilinmagan
  ConditionallyMutate, // Branch'ga qarab
  Mutate,       // Definitively mutated
  Store,        // Assigned to escape
  Read,         // Read-only
}
```

Compiler har value uchun effect'ni aniqlaydi — memoization xavfsizligini baholash uchun.

**Reactive scope determination algorithm:**

```typescript
// Mental model:
function findReactiveScopes(hir: HIR): ReactiveScope[] {
  const scopes: ReactiveScope[] = [];
  for (const block of hir.blocks) {
    for (const instr of block.instructions) {
      if (isPure(instr) && hasReactiveDeps(instr)) {
        // Compute effect — agar safe bo'lsa, scope yaratamiz
        scopes.push({
          deps: collectDeps(instr),
          output: instr.lvalue,
          range: { start: instr, end: instr },
        });
      }
    }
  }
  return mergeOverlappingScopes(scopes);
}
```

**Cache size hisobi:**

Compiler har komponent uchun maximum cache size'ni hisoblaydi (worst-case): har scope deps + value uchun slot. Mount paytida `useMemoCache(N)` chaqiriladi.

**Source maps:**

Compiler source maps yaratadi — debugging'da original kod ko'rinadi. Production'da output minified, lekin original line/column numbers bilan map qilinadi.

</details>

### Edge Cases

- **Conditional render**: `if (cond) return null` — Compiler control flow analyse qiladi, har branch'da reactive scope alohida.
- **`try`/`catch` in render**: Compiler skip qiladi (xavfsizlik uchun) yoki conservative behavior.
- **Mutation detection limitations**: Indirect mutation (function arg passing) ba'zida miss qilinadi → conservative skip.

### Follow-up savollar

- "Compiler bug yangilanish bilan kelishini qanday tekshirasiz?" — ESLint plugin xato'larni warn qiladi. Production'da problemali komponentlarni `"use no memo"` direktiva bilan bypass.
- "Compiler runtime overhead bormi?" — `useMemoCache` hook qo'shadi (manual `useMemo` kabi hook slot + cache array). Boshqa runtime monkey-patch yo'q.
- "Bundle size impact?" — Compiler avtomat memoization uchun qo'shimcha kod kiritadi (cache logic). Hajm o'sishi ilovaga bog'liq; gzip ostida odatda kichik bo'ladi. Aniq raqam uchun bundle analyzer ishlating.

</details>

---

<a id="qism-e"></a>

### 15. "Rules of React" — Compiler nimani majbur qiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Rules of React — Compiler ishlay olishi uchun komponent kodi rioya qilinadigan qoidalar to'plami: (1) **Components must be pure** — render paytida side effects yo'q. (2) **Props/state immutable** — mutate qilinmaydi. (3) **Hooks rules** — top-level only, conditional/loop hooks taqiq. (4) **No conditional component types** — render'da qaysi komponent turini qaytarish o'zgarmaydi. Bu qoidalar bo'yicha komponent **referential transparency**ga ega bo'ladi → memoization safe. ESLint rule (`react-compiler`) violation'larni dev'da topadi.

### To'liq tushuntirish

**Rule 1: Component purity**

```tsx
// ❌ Side effect in render
function Component() {
  document.title = "New Title"; // ❌ DOM mutation
  return <p>Hi</p>;
}

// ✅ Side effect in useEffect
function Component() {
  useEffect(() => {
    document.title = "New Title";
  }, []);
  return <p>Hi</p>;
}
```

Sabab: Compiler memoization render skip qilishi mumkin → side effect bajarilmaydi → user-visible bug. Render — pure function: `props → JSX`.

**Rule 2: Props/state immutability**

```tsx
// ❌ Props mutation
function Component({ user }: { user: User }) {
  user.lastSeen = Date.now(); // ❌ mutates parent's state
  return <p>{user.name}</p>;
}

// ✅ New object
function Component({ user, onUpdate }: Props) {
  useEffect(() => {
    onUpdate({ ...user, lastSeen: Date.now() });
  }, []);
  return <p>{user.name}</p>;
}
```

**Rule 3: Hooks rules**

```tsx
// ❌ Conditional hook
function Component({ enabled }: { enabled: boolean }) {
  if (enabled) {
    const [count, setCount] = useState(0); // ❌
  }
  return <p>Hi</p>;
}

// ❌ Loop hook
function Component({ items }: { items: Item[] }) {
  for (const item of items) {
    useEffect(() => {}, []); // ❌
  }
  return <ul>...</ul>;
}

// ✅ Top-level
function Component({ enabled }: { enabled: boolean }) {
  const [count, setCount] = useState(0); // Always called
  if (!enabled) return null;
  return <p>{count}</p>;
}
```

Sabab: Hook'lar Fiber'ning linked list slot'larida saqlanadi. Tartib stable bo'lishi shart — har render bir xil tartibda chaqiriladi.

**Rule 4: Stable component types**

```tsx
// ❌ Component definition in render
function Parent() {
  function Child() { return <p>Hi</p>; } // ❌ har render — yangi function reference
  return <Child />;
}

// ✅ Outside
function Child() { return <p>Hi</p>; }
function Parent() {
  return <Child />;
}
```

`<Child />` element'ning `type` field — function reference. Har render'da yangi function — Reconciler uni yangi tip deb ko'radi → unmount + mount (state lost!).

**Rule 5: Pure event handlers**

```tsx
// ✅ OK: event handler ichida side effect (event-driven)
function Component() {
  return <button onClick={() => fetch("/api")}>Click</button>;
}
```

Event handler'lar render'dan tashqari bajariladi — pure bo'lishi shart emas.

**Rule 6: Refs only in effects/handlers**

```tsx
// ❌ Read ref in render
function Component() {
  const ref = useRef(0);
  console.log(ref.current); // ❌ tearing risk in concurrent
  return <p>Hi</p>;
}

// ✅ Read in effect
function Component() {
  const ref = useRef(0);
  useEffect(() => {
    console.log(ref.current);
  });
  return <p>Hi</p>;
}
```

### Kod misoli

**Rules of React audit — checklist:**

```tsx
import { useState, useEffect, useRef } from "react";

function ProductCard({ product, onAddToCart }: Props) {
  const [quantity, setQuantity] = useState(1);
  const renderCountRef = useRef(0);

  // ✅ Rule 1 — pure render (no side effects)
  // ✅ Rule 2 — no mutation of props
  // ✅ Rule 3 — top-level hooks
  // ✅ Rule 5 — handler with side effect

  useEffect(() => {
    // ✅ Side effect in useEffect
    console.log(`ProductCard rendered: ${product.id}`);
  });

  const handleAdd = () => {
    // ✅ Side effect in handler
    onAddToCart(product.id, quantity);
    setQuantity(1);
  };

  return (
    <div>
      <h3>{product.name}</h3>
      <input
        type="number"
        value={quantity}
        onChange={(e) => setQuantity(Number(e.target.value))}
      />
      <button onClick={handleAdd}>Add to Cart</button>
    </div>
  );
}
```

**Anti-pattern audit:**

```tsx
// ❌ Multiple violations
function BadComponent({ items, config }: Props) {
  // ❌ Rule 4 — component definition in render
  function ItemRow({ item }: { item: Item }) {
    return <li>{item.name}</li>;
  }

  // ❌ Rule 2 — props mutation
  config.timestamp = Date.now();

  // ❌ Rule 3 — conditional hook
  if (items.length > 0) {
    const [first] = useState(items[0]);
  }

  // ❌ Rule 1 — side effect in render
  localStorage.setItem("lastView", new Date().toISOString());

  // ❌ Rule 4 — every render new component → unmount/remount
  return (
    <ul>
      {items.map(i => <ItemRow key={i.id} item={i} />)}
    </ul>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**ESLint rules — automated detection:**

```json
{
  "plugins": ["react-compiler", "react-hooks"],
  "rules": {
    "react-compiler/react-compiler": "error",
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

**Compiler skip behavior:**

```typescript
// Compiler "Bailout":
// 1. Komponent qoidabuzar deb aniqlanadi
// 2. Original kod o'zgarishsiz qoladi (no useMemoCache)
// 3. Warning emit (build log)

// Bu graceful fallback — kod baribir ishlaydi, faqat optimization yo'q
```

**Why purity matters — abort scenarios:**

```typescript
// React render abort possibilities:
// 1. Higher priority interrupt
// 2. Suspense throw
// 3. Concurrent rendering
// 4. Strict Mode 2x

// Pure render: same input → same output → safe to abort
// Impure: side effect bajarilgan → state divergence
```

**Strict Mode dev — invariant testing:**

```tsx
<StrictMode>
  <App />
</StrictMode>
```

Effects:
- Components rendered 2x (idempotency test)
- Effects setup-cleanup-setup (cleanup correctness)
- Refs created 2x

Pure komponentlar 2x render'ga immune. Impure — bug'lar (counter increment, fetch'lar duplicated, etc).

**Compiler enforces purity at compile-time, React enforces at runtime:**

| Layer | Detection method |
|-------|-----------------|
| ESLint `react-compiler` | Static analysis (build) |
| Compiler skip | Build-time bailout |
| `<StrictMode>` 2x | Runtime dev exposure |
| Effect cleanup | Runtime correctness check |
| Concurrent abort | Runtime resilience |

**Hooks rules — Fiber slot mechanics:**

```typescript
// Fiber.memoizedState — Hook linked list
// {
//   memoizedState: 0,        // useState slot
//   next: {
//     memoizedState: [...],  // useEffect slot
//     next: { ... }
//   }
// }

// Render'da Hook'lar tartib bo'yicha resolve qilinadi
// Conditional hook → tartib o'zgaradi → wrong slot read → crash
```

**Component identity rule:**

```tsx
// Element { type: ComponentRef, props }
// Reconciler: prevType === nextType?
// Yes → update Fiber
// No → unmount old, mount new (state lost)

// Inline component definition:
function Parent() {
  function Child() {} // each render: NEW reference
  return <Child />;
  // Each render: type !== prevType → unmount + remount
}
```

</details>

### Edge Cases

- **`useMemo` factory bilan side effect**: Factory pure bo'lishi shart. `useMemo(() => fetch(...), [])` — Promise yaratish OK, lekin actual side effect (`fetch` execute) — yo'q (fetch lazily started).
- **Async render — RSC**: Server Components async function bo'lishi mumkin (await fetch in body). Bu RSC-specific exception.
- **Class component lifecycle**: `render()` pure bo'lishi shart. `componentDidMount` — side effects OK.

### Follow-up savollar

- "Strict Mode 2x render production'da bormi?" — Yo'q. Faqat development. Production single render.
- "Hook'larni conditional ishlatish hech qachon mumkinmi?" — Yo'q. Workaround — komponent split: parent conditional, child unconditional hooks.
- "Compiler bilan ESLint rule kombinatsiyasi qanday?" — Compiler runtime safety, ESLint dev-time warning. Birgalikda — full coverage.

</details>

---

### 16. Compiler limitations va opt-out qachon kerak? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Compiler **graceful fallback** — qoidabuzar yoki murakkab kod uchun memoization skip qiladi (warning bilan). Aniq limitations: (1) **Aliasing chuqurligi** — indirect mutation tracking cheklangan. (2) **Class komponentlar** — qisman support. (3) **Dynamic component types** — runtime'da aniqlanadigan tip. (4) **External library hooks** — agar hook ichi mutation qilsa. Opt-out — `"use no memo"` directive komponent yoki fayl darajasida. Migration paytida problemali komponentlarni opt-out qilish, asta-sekin tuzatish.

### To'liq tushuntirish

**Limitation 1 — Indirect aliasing:**

```tsx
function Component({ data }: { data: Item[] }) {
  const list = data;        // alias
  const list2 = list;       // chain alias
  list2.push("extra");      // mutates `data` (indirect)

  return <ul>{data.map(...)}</ul>;
}

// Compiler tracking depth limited
// Conservative behavior: skip memoization (xavfsiz)
```

**Limitation 2 — Function passed to external API:**

```tsx
function Component({ items }: { items: Item[] }) {
  const sorted = items.toSorted((a, b) => {
    // Compiler doesn't track inside JS built-in:
    // a, b ichida mutation bormi?
    return a.priority - b.priority;
  });
  return <ul>{sorted.map(...)}</ul>;
}

// Compiler conservative — assumes external function may mutate
// Memoization may be skipped or partial
```

**Limitation 3 — Custom hooks ichidagi mutation:**

```tsx
// External library hook
function useExternalState(initial: object): object {
  const ref = useRef(initial);
  ref.current.lastAccessed = Date.now(); // ❌ mutation in render
  return ref.current;
}

function Component() {
  const state = useExternalState({ count: 0 });
  // Compiler can't analyse useExternalState body
  // Conservative — skip
  return <p>{state.count}</p>;
}
```

**Limitation 4 — Class komponentlar:**

```tsx
class OldComponent extends React.Component {
  render() {
    // Compiler partially supports
    // Class methods, lifecycles — limited optimization
    return <p>{this.props.value}</p>;
  }
}
```

R19 Compiler asosan function components uchun. Class — partial.

**Limitation 5 — Dynamic dispatch:**

```tsx
const components = {
  card: Card,
  list: List,
  table: Table,
};

function Component({ type, data }: Props) {
  const C = components[type]; // dynamic lookup
  return <C data={data} />;
}

// Compiler can't statically determine `C`
// Memoization conservative
```

**Opt-out — `"use no memo"`:**

```tsx
"use no memo";

// Entire file skipped by Compiler
function ProblematicComponent({ data }: Props) {
  // Compiler bypass
  return <div>{data.value}</div>;
}
```

```tsx
// Per-component opt-out
function ProblematicComponent({ data }: Props) {
  "use no memo";
  return <div>{data.value}</div>;
}
```

### Kod misoli

**Limitation example — diagnose va fix:**

```tsx
// ❌ Compiler unable to memoize
function BadList({ items, sortBy }: Props) {
  // External lib (lodash) — Compiler can't introspect
  const sorted = _.sortBy(items, sortBy);
  return <ul>{sorted.map(i => <li>{i.name}</li>)}</ul>;
}

// ✅ Native — Compiler analyzes
function GoodList({ items, sortBy }: Props) {
  const sorted = [...items].sort((a, b) => {
    if (a[sortBy] < b[sortBy]) return -1;
    if (a[sortBy] > b[sortBy]) return 1;
    return 0;
  });
  return <ul>{sorted.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}
```

**Migration progressive opt-in:**

```tsx
// Phase 1: enable Compiler globally, opt-out problematic
// Phase 2: fix opt-out komponentlarni asta-sekin
// Phase 3: opt-out o'chirib tashlash

// File-level opt-out:
"use no memo";

import { LegacyTable } from "./LegacyTable";
// LegacyTable — class komponent, complex lifecycles
```

**Test va Profiler bilan validation:**

```tsx
import { render } from "@testing-library/react";
import { Profiler } from "react";

test("Component memoization correctness", () => {
  const onRender = jest.fn();

  const { rerender } = render(
    <Profiler id="test" onRender={onRender}>
      <Component count={1} name="Ali" />
    </Profiler>
  );

  // Force re-render same props
  rerender(
    <Profiler id="test" onRender={onRender}>
      <Component count={1} name="Ali" />
    </Profiler>
  );

  // Compiler bilan: ikkinchi render — minimal work
  // Profiler onRender callback'da actualDuration ~0ms
});
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Compiler bailout reasons (build log):**

```
[react-compiler] Skipping component "BadComponent":
  - "Cannot determine mutability of identifier 'list'"
  - "Function call 'externalLib.process' has unknown side effects"

[react-compiler] Compiled "GoodComponent" successfully (3 reactive scopes)
```

Build log'da explicit reasons.

**Compiler internal effect inference (mental model):**

```typescript
function inferEffects(hir: HIR): Map<Identifier, Effect[]> {
  // 1. Direct effects from instructions
  // 2. Function calls — unknown effects (conservative)
  // 3. Property access — Read
  // 4. Assignment — Mutate
  // 5. Spread operator — Capture (object props)

  // Aliasing tracking:
  const aliases = computeAliases(hir);
  // Effects propagate via aliases
}
```

**Conservative vs aggressive analysis:**

| Approach | Pros | Cons |
|----------|------|------|
| Conservative (Compiler default) | Safe — no false memoization | Less optimization |
| Aggressive | More memoization | Risk of incorrectness |

R19 Compiler conservative. Future versions — aggressive optimization (more careful tracking).

**Workarounds for limitations:**

1. **Convert to native**: Replace lodash with native `Array.prototype.sort`
2. **Move out of render**: Computation in `useMemo` (manual) or `useEffect`
3. **Pure wrappers**: Wrap external lib calls in pure functions
4. **Opt-out targeted**: `"use no memo"` for specific komponent

**Cost of opt-out:**

- Manual `useMemo`/`useCallback` add qilish kerak (Compiler benefit yo'q)
- Re-render performance default cascade
- Strict Mode dev'da aniq behavior

**Compiler future direction:**

- Better aliasing (type-aware)
- Class komponent full support
- Conditional memoization (runtime feature flags)
- Hot reload preservation (memoization cache survive HMR)

**Profiler — Compiler effectiveness measurement:**

```tsx
import { Profiler } from "react";

function App() {
  return (
    <Profiler
      id="App"
      onRender={(id, phase, actualDuration, baseDuration) => {
        // baseDuration: render without memoization
        // actualDuration: render with memoization
        // ratio = actualDuration / baseDuration → memoization effectiveness
        if (actualDuration / baseDuration > 0.8) {
          console.warn(`Memoization ineffective in ${id}`);
        }
      }}
    >
      <ChildComponents />
    </Profiler>
  );
}
```

**Real-world Compiler adoption:**

Meta — Instagram'da ishlatish boshlandi (2024). Internal reports render time qisqarishini ko'rsatgan (large list'lar uchun). Bug rate impact: minimal — Compiler conservative.

</details>

### Edge Cases

- **Hot Module Replacement (HMR)**: Compiler cache survive HMR — kelajakda. Hozircha cache reset HMR'da.
- **Server Components**: Compiler RSC support — partial. Async function components — alohida transformation.
- **`React.lazy` bilan**: Lazy komponent dynamic import → Compiler bypass (statically unknown).

### Follow-up savollar

- "Compiler `__DEV__` build'da behavior farqi?" — Yo'q (warning'lar dev'da ko'p, lekin transformation bir xil).
- "Compiler bilan production debugging qiyin emasmi?" — Source maps saqlanadi. Ammo memoized output behavior — manual yozilgan emas. Profiler + ESLint warnings yetadi.
- "Compiler kerak bo'lmagan loyiha bormi?" — Static sites (no interactivity) — Compiler benefit yo'q. Lekin overhead ham yo'q (no transformation needed).

</details>

---

## QISM E: Profiling

### 17. React DevTools Profiler nima qiladi va u qanday ishlatiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React DevTools Profiler — browser extension'ning panel'i. Komponent render'larini **commit-by-commit** qayd qiladi, har bir komponentning render duration'ini, render reason'ini va Fiber tree'dagi taqsimotini ko'rsatadi. **Flamegraph** — har commit'dagi render hierarchy. **Ranked chart** — eng qimmat komponentlar tartibi. **Component view** — selected komponentning props/state/hooks.  Asosiy ish jarayoni: Record → User interaction → Stop → Analyze.

### To'liq tushuntirish

**Setup:**

1. Chrome/Firefox'ga "React Developer Tools" extension o'rnatish
2. Development build (`NODE_ENV !== 'production'`) yoki `react-dom/profiling` ishlatish
3. DevTools → Profiler tab

**Workflow:**

1. **Record** ⏺ — boshlash
2. User interaction (click, form input, route change)
3. **Stop** ⏹ — to'xtatish
4. Analyze commits

**Profiler views:**

| View | Nima ko'rsatadi |
|------|----------------|
| Flamegraph | Har commit Fiber tree (komponent hierarchy + render duration) |
| Ranked | Komponentlar render duration bo'yicha sortlangan |
| Timeline | Commit'lar vaqt o'qida (browser timeline bilan) |
| Component | Selected komponent — props, hooks, render reason |

**Render reason ("Why did this render?"):**

Profiler DevTools settings'da yoqilganda — har komponent render sababini ko'rsatadi:
- "This is the first time the component rendered"
- "Hooks changed" (qaysi hook)
- "Props changed" (qaysi prop)
- "Parent component rendered"
- "Context changed: ContextName"

**Render duration metrics:**

| Metric | Ma'no |
|--------|-------|
| `actualDuration` | Bu render uchun haqiqiy vaqt (sub-tree bilan) |
| `baseDuration` | Memoization yo'qligida nominal duration |
| `startTime` | Commit boshlangan vaqt |
| `commitTime` | Commit tugagan vaqt |

`baseDuration / actualDuration` — memoization effectiveness ratio.

### Kod misoli

**Setup development build:**

```tsx
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  // Profiler avtomat dev build'da yoqiladi
});
```

**Production profiling — opt-in:**

```tsx
// Production build with profiling
// next.config.js
module.exports = {
  reactStrictMode: true,
  compiler: {
    reactProfiling: true, // Next.js 14+
  },
};
```

Yoki manual:

```bash
# Vite — react-dom/profiling alias
```

```typescript
// vite.config.ts
export default defineConfig({
  resolve: {
    alias: {
      "react-dom$": "react-dom/profiling",
      "scheduler/tracing": "scheduler/tracing-profiling",
    },
  },
});
```

**Profiler-friendly komponent (debug helper):**

```tsx
import { useEffect, useRef } from "react";

function useRenderCount(name: string) {
  const count = useRef(0);
  count.current++;
  useEffect(() => {
    console.log(`${name} rendered ${count.current} times`);
  });
  return count.current;
}

function ProductCard({ product }: Props) {
  useRenderCount("ProductCard");
  return <div>{product.name}</div>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Profiler ishlash mexanizmi:**

```typescript
// react-dom/profiling — Reconciler hooks instrumented
// Har Fiber render'da:

function commitRoot(root: FiberRoot) {
  const finishedWork = root.finishedWork;
  // Commit phases bajariladi
  // Profiler timer:
  const startTime = performance.now();
  commitMutationEffects(finishedWork);
  commitLayoutEffects(finishedWork);
  const commitDuration = performance.now() - startTime;

  // DevTools'ga onCommitRoot dispatch
  if (rootHasProfiler) {
    DevToolsHook.onCommitFiberRoot(rendererId, root, commitDuration);
  }
}
```

**`hot-key` — DevTools commands:**

| Action | Shortcut |
|--------|----------|
| Start recording | `Cmd/Ctrl + R` (Profiler tab focused) |
| Stop recording | Same |
| Reload + Profile | `Cmd/Ctrl + Shift + R` |

**Profiler limitations:**

1. **Production overhead**: Profiler instrumented build sekinroq (har Fiber uchun timer overhead). Faqat debugging uchun.
2. **Concurrent rendering edge cases**: Aborted renders ham qayd qilinishi mumkin (DevTools indicator bor).
3. **Sub-second granularity**: 0.1ms aniqlikda. Mikrosekund-darajadagi farqlar ko'rinmaydi.

**Filtering options:**

- Hide commits below threshold (e.g., < 1ms)
- Show only memoization changes
- Filter by component name

**Common patterns to spot:**

1. **Cascade re-render** — parent render qildi, barcha child'lar ham (memo yo'q)
2. **Wasteful render** — bir xil props bilan render bo'lgan komponent
3. **Slow render** — actual duration > 16ms (1 frame)
4. **Render in loop** — bir necha commit ketma-ket (state update bilan)

**`Highlight updates when components render`:**

DevTools settings → "Highlight updates when components render" — har render bo'lgan komponent atrofida rangli border highlight. Real-time visual feedback.

> **Performance note:** Highlighting o'zi rendering overhead qo'shadi. Faqat debugging uchun yoqing.

**Profiler integration with Performance tab:**

Chrome DevTools Performance — full browser timeline (JS, paint, layout, GC). React Profiler — komponent darajasida. Birgalikda ishlatish:

```
1. Performance tab — overall trace
2. React Profiler — komponent darajasi
3. User Timing API — custom marks
```

Custom marks:

```typescript
performance.mark("ProductList:filter:start");
const filtered = products.filter(...);
performance.mark("ProductList:filter:end");
performance.measure("filter", "ProductList:filter:start", "ProductList:filter:end");
```

</details>

### Edge Cases

- **Async render abort**: Profiler "discarded" indicator. Aborted render — UI'ga ta'sir qilmaydi, lekin compute vaqti ketgan.
- **Suspense boundary**: Profiler Suspense fallback ko'rinishini qayd qiladi. Suspended state — alohida commit.
- **Strict Mode 2x**: Dev'da har commit 2 marta. Profiler buni ko'rsatadi (2 ta commit).

### Follow-up savollar

- "Production'da Profiler qanday ishlatiladi?" — `react-dom/profiling` alias bilan opt-in. Lekin bundle size va overhead. Faqat staging/canary'da.
- "Profiler vs Chrome DevTools Performance — qaysi qachon?" — React Profiler — komponent darajasi (render reasons, props). Chrome — browser darajasi (paint, layout, JS frame). Birgalikda full picture.
- "Profiler interaction tracing nima edi?" — R16-17 da `unstable_trace` API edi. R18+ deprecated. O'rniga `Profiler` component yoki Chrome User Timing.

</details>

---

<a id="qism-f"></a>

### 18. `<Profiler>` komponent va `onRender` callback'i programmatik qanday ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`<Profiler>` — built-in React komponent, sub-tree render duration'ini programmatik o'lchaydi. Props: `id` (identifikator) va `onRender` callback. Callback har commit'da chaqiriladi va render metrics'ni qaytaradi (`actualDuration`, `baseDuration`, `phase`, `startTime`, `commitTime`). Foydalanish: production telemetry (slow renders log), tests (memoization correctness), profiling without DevTools.

### To'liq tushuntirish

**API:**

```tsx
<Profiler
  id="UniqueId"
  onRender={(id, phase, actualDuration, baseDuration, startTime, commitTime) => {
    // Metrics
  }}
>
  <ChildSubtree />
</Profiler>
```

**Callback parameters:**

| Parametr | Type | Ma'no |
|----------|------|-------|
| `id` | `string` | `<Profiler>` `id` prop |
| `phase` | `"mount" \| "update" \| "nested-update"` | Render fazasi |
| `actualDuration` | `number` (ms) | Bu render duration (sub-tree bilan) |
| `baseDuration` | `number` (ms) | Memoization yo'qligidagi expected duration |
| `startTime` | `number` (ms) | Render boshlangan vaqt |
| `commitTime` | `number` (ms) | Commit tugagan vaqt |

**Phase ma'nolari:**

- `mount` — birinchi render
- `update` — keyingi re-render
- `nested-update` — render'ning natijasi (effect ichida setState) yana render trigger

### Kod misoli

**Production telemetry:**

```tsx
import { Profiler } from "react";
import { logTelemetry } from "./telemetry";

const onRenderCallback = (
  id: string,
  phase: "mount" | "update" | "nested-update",
  actualDuration: number,
  baseDuration: number,
  startTime: number,
  commitTime: number,
) => {
  // Slow render — log
  if (actualDuration > 16) {
    logTelemetry({
      type: "slow_render",
      component: id,
      phase,
      duration: actualDuration,
      baseDuration,
      memoizationRatio: actualDuration / baseDuration,
      timestamp: commitTime,
    });
  }
};

function App() {
  return (
    <Profiler id="App" onRender={onRenderCallback}>
      <Profiler id="Header" onRender={onRenderCallback}>
        <Header />
      </Profiler>
      <Profiler id="MainContent" onRender={onRenderCallback}>
        <MainContent />
      </Profiler>
      <Profiler id="Footer" onRender={onRenderCallback}>
        <Footer />
      </Profiler>
    </Profiler>
  );
}
```

**Test — memoization correctness:**

```tsx
import { render, fireEvent } from "@testing-library/react";
import { Profiler } from "react";

test("Memo bailout works for unchanged props", () => {
  const onRender = jest.fn();

  const { rerender } = render(
    <Profiler id="test" onRender={onRender}>
      <ProductList products={products} filter="" />
    </Profiler>
  );

  expect(onRender).toHaveBeenCalledTimes(1);
  expect(onRender.mock.calls[0][1]).toBe("mount");

  // Re-render with same props
  rerender(
    <Profiler id="test" onRender={onRender}>
      <ProductList products={products} filter="" />
    </Profiler>
  );

  expect(onRender).toHaveBeenCalledTimes(2);
  expect(onRender.mock.calls[1][1]).toBe("update");

  // Memo bailout — actualDuration < baseDuration
  const [, , actualDuration, baseDuration] = onRender.mock.calls[1];
  expect(actualDuration).toBeLessThan(baseDuration);
});
```

**Custom hooks for measurement:**

```tsx
import { Profiler, useCallback } from "react";

interface RenderMetrics {
  count: number;
  totalDuration: number;
  averageDuration: number;
  slowestRender: number;
}

function useRenderMetrics(id: string): {
  Profiler: React.FC<{ children: React.ReactNode }>;
  metrics: RenderMetrics;
} {
  const [metrics, setMetrics] = useState<RenderMetrics>({
    count: 0,
    totalDuration: 0,
    averageDuration: 0,
    slowestRender: 0,
  });

  const onRender = useCallback(
    (_: string, __: string, actualDuration: number) => {
      setMetrics((prev) => {
        const newCount = prev.count + 1;
        const newTotal = prev.totalDuration + actualDuration;
        return {
          count: newCount,
          totalDuration: newTotal,
          averageDuration: newTotal / newCount,
          slowestRender: Math.max(prev.slowestRender, actualDuration),
        };
      });
    },
    [],
  );

  const ProfilerWrapper = useCallback(
    ({ children }: { children: React.ReactNode }) => (
      <Profiler id={id} onRender={onRender}>{children}</Profiler>
    ),
    [id, onRender],
  );

  return { Profiler: ProfilerWrapper, metrics };
}

// Usage
function App() {
  const { Profiler: P, metrics } = useRenderMetrics("App");
  return (
    <P>
      <p>Avg render: {metrics.averageDuration.toFixed(2)}ms</p>
      <ChildComponent />
    </P>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`<Profiler>` Reconciler internals:**

```typescript
// react-reconciler/src/ReactFiberWorkLoop.js
function commitProfilerEffects(finishedWork: Fiber) {
  // Profiler Fiber detected
  if (finishedWork.tag === Profiler) {
    const { id, onRender } = finishedWork.memoizedProps;
    const profilerInstance = finishedWork.stateNode;

    if (typeof onRender === "function") {
      onRender(
        id,
        finishedWork.alternate === null ? "mount" : "update",
        profilerInstance.actualDuration,    // render vaqti (sub-tree bilan)
        profilerInstance.treeBaseDuration,  // memoization'siz nominal duration
        profilerInstance.startTime,
        commitTime,
      );
    }
  }
}
```

**Profiler tag (Fiber):**

```typescript
// ReactWorkTags.js
export const Profiler = 12; // Tag value
```

**`actualDuration` hisobi:**

```typescript
// Profiler tracks render time for entire sub-tree
function profilerStartTime() {
  return performance.now();
}

// Render begin:
fiber.actualStartTime = performance.now();

// Render end:
fiber.actualDuration = performance.now() - fiber.actualStartTime;
// + sum of child actualDurations (sub-tree total)
```

**`baseDuration` — memoization-naive estimate:**

```typescript
// Worst-case render duration if everything was rendered
fiber.baseDuration = sum of all child render durations
                    even if they bailed out
```

`actualDuration` < `baseDuration` — memoization saved time.

**Production overhead:**

`<Profiler>` tag enabled — Reconciler tracking timer overhead per render. Production bundle:

- `react-dom/profiling` — full instrumentation, sezilarli overhead (har Fiber uchun timer)
- `react-dom` (default) — `<Profiler>` API present, lekin minimal overhead

> **Performance note:** `<Profiler>` ko'p bo'lsa (ko'p sub-tree'lar) — overhead summasi sezilarli bo'ladi. Production'da nuanced: faqat critical path'larga.

**Sample telemetry — Web Vitals:**

```tsx
import { onLCP, onINP, onCLS } from "web-vitals";

// React render metrics + Web Vitals
function App() {
  useEffect(() => {
    onLCP(reportToAnalytics);
    onINP(reportToAnalytics);
    onCLS(reportToAnalytics);
  }, []);

  return (
    <Profiler id="App" onRender={reportRenderToAnalytics}>
      <AppContent />
    </Profiler>
  );
}
```

**Aggregation strategy:**

Per-render telemetry — too verbose. Aggregate:

```tsx
const buffer: RenderEvent[] = [];

const onRenderCallback = (id, phase, actualDuration) => {
  buffer.push({ id, phase, duration: actualDuration, ts: performance.now() });

  // Flush periodically
  if (buffer.length >= 100) {
    sendBatch(buffer);
    buffer.length = 0;
  }
};
```

Yoki — `requestIdleCallback` orqali idle flush.

**`startTransition` interaction:**

```tsx
import { startTransition, Profiler } from "react";

function App() {
  return (
    <Profiler id="App" onRender={onRender}>
      <button onClick={() => startTransition(() => setQuery("new"))}>
        Search
      </button>
    </Profiler>
  );
}

// Transition'lar — separate commit phase
// Profiler buni "update" deb qayd qiladi (default phase)
// Lower priority lane — actualDuration boshqa render'larga qo'shilmaydi
```

</details>

### Edge Cases

- **Nested `<Profiler>`**: `actualDuration` inner profiler — outer'ga ham qo'shiladi (sub-tree). `baseDuration` ham hisoblab beriladi.
- **Suspense fallback**: Suspended sub-tree render — alohida commit. Profiler buni qayd qiladi.
- **Error Boundary**: Error throw bo'lganda — render incomplete. Profiler `onRender` chaqirilmasligi mumkin (commit bo'lmagan).

### Follow-up savollar

- "`<Profiler>` production'da ishlatish xavfsizmi?" — Ha. Default React build'da `<Profiler>` API ishlaydi (minimal overhead). Lekin to'liq instrumented build (`react-dom/profiling`) ko'proq overhead.
- "Telemetry'ni qanday strukturasi optimal?" — Aggregated batches (100 events), p50/p95/p99 percentiles, slow render threshold (16ms).
- "Server Components'da `<Profiler>` ishlaydimi?" — Yo'q. Profiler client-only API. Server render — alohida `react-dom/server` profiling.

</details>

---

### 19. Production'da performance monitoring qanday tashkillashtiriladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Production performance monitoring 4 ta darajada amalga oshiriladi: (1) **Web Vitals** (Core: LCP, INP, CLS) — user experience metrics. (2) **React Profiler API** — komponent render metrics, slow renders detection. (3) **Custom marks** — `performance.measure` bilan business operations. (4) **Real User Monitoring (RUM)** — DataDog, New Relic, Sentry'da real user data. Bundle size va render frequency cost'i bor — sampling strategy talab qilinadi (10-20% users).

### To'liq tushuntirish

**Web Vitals — INP-focused (R19+ era):**

| Metric | Target | Ma'no |
|--------|--------|-------|
| LCP (Largest Contentful Paint) | < 2.5s | Asosiy contentning ko'rinishi |
| INP (Interaction to Next Paint) | < 200ms | User interaction reactiveness |
| CLS (Cumulative Layout Shift) | < 0.1 | Visual stability |
| FCP (First Contentful Paint) | < 1.8s | Birinchi piksel paint |
| TTFB (Time to First Byte) | < 800ms | Server response |

R19'da FID o'rniga **INP** asosiy metric (Google Core Web Vitals 2024'dan).

**React-specific metrics:**

- **Render duration** (actualDuration) — slow renders > 16ms
- **Re-render frequency** — bir xil komponent N marta/min
- **Suspense fallback duration** — loading states
- **Hydration duration** — SSR → CSR transition

**Sampling strategy:**

Performance metrics — har user'dan to'planmasin (cost). Sample:

```typescript
const samplingRate = 0.1; // 10% users

if (Math.random() < samplingRate) {
  // Enable detailed profiling for this session
  enablePerformanceTracking();
}
```

**Tools va integratsiya:**

| Tool | Use case |
|------|---------|
| `web-vitals` | Core Web Vitals (Google library) |
| Sentry Performance | RUM, error correlation |
| DataDog RUM | RUM, logs, metrics |
| New Relic Browser | RUM, AJAX tracking |
| Custom backend | Self-hosted, full control |

### Kod misoli

**`web-vitals` integration:**

```tsx
import { onLCP, onCLS, onINP, onFCP, onTTFB } from "web-vitals";
import { sendToAnalytics } from "./analytics";

function reportWebVitals(metric: any) {
  // Send to backend
  sendToAnalytics({
    name: metric.name,
    value: metric.value,
    rating: metric.rating, // "good" | "needs-improvement" | "poor"
    id: metric.id,
    delta: metric.delta,
    navigationType: metric.navigationType,
  });
}

// At app initialization
function App() {
  useEffect(() => {
    onLCP(reportWebVitals);
    onINP(reportWebVitals);
    onCLS(reportWebVitals);
    onFCP(reportWebVitals);
    onTTFB(reportWebVitals);
  }, []);

  return <AppContent />;
}
```

**React Profiler + telemetry:**

```tsx
import { Profiler } from "react";
import { sendTelemetry } from "./telemetry";

const SLOW_RENDER_THRESHOLD = 16; // ms (1 frame at 60fps)

const onProfilerRender = (
  id: string,
  phase: "mount" | "update" | "nested-update",
  actualDuration: number,
  baseDuration: number,
  startTime: number,
  commitTime: number,
) => {
  if (actualDuration > SLOW_RENDER_THRESHOLD) {
    sendTelemetry({
      type: "slow_render",
      component: id,
      phase,
      actualDuration,
      baseDuration,
      memoizationRatio: actualDuration / baseDuration,
      timestamp: commitTime,
      url: window.location.href,
      userAgent: navigator.userAgent,
    });
  }
};

function App() {
  return (
    <Profiler id="App" onRender={onProfilerRender}>
      <AppContent />
    </Profiler>
  );
}
```

**Custom marks va measure:**

```tsx
function ProductDashboard({ productId }: Props) {
  const [data, setData] = useState<Product | null>(null);

  useEffect(() => {
    performance.mark("dashboard:fetch:start");
    fetch(`/api/products/${productId}`)
      .then((r) => r.json())
      .then((d) => {
        performance.mark("dashboard:fetch:end");
        performance.measure(
          "dashboard:fetch",
          "dashboard:fetch:start",
          "dashboard:fetch:end",
        );

        const measures = performance.getEntriesByName("dashboard:fetch");
        const lastMeasure = measures[measures.length - 1];
        sendTelemetry({
          type: "fetch_duration",
          duration: lastMeasure.duration,
          productId,
        });

        setData(d);
      });
  }, [productId]);

  if (!data) return <Spinner />;
  return <ProductDetails product={data} />;
}
```

**Sentry Performance integration:**

```tsx
import * as Sentry from "@sentry/react";

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  integrations: [
    Sentry.reactRouterV6BrowserTracingIntegration({
      useEffect,
      useLocation,
      useNavigationType,
      createRoutesFromChildren,
      matchRoutes,
    }),
  ],
  tracesSampleRate: 0.1, // 10% sampling
});

const App = Sentry.withProfiler(AppRoot);

// Now all renders, transitions, fetches — auto-tracked
```

<details>
<summary><strong>Deep Dive</strong></summary>

**INP — Interaction to Next Paint (R19 era):**

INP sessiya davomidagi eng sekin interaction latency'ni approximatsiya qiladi (ko'p interaction'lar bo'lsa, eng yomon interaction'lardan biri tashlab qo'yiladi). FID — faqat birinchi interaction. INP — UX'ni aniqroq ifodalaydi.

```typescript
// web-vitals onINP triggers:
// 1. User interaction (click, key, tap)
// 2. JS handler runs
// 3. Browser paint
// → Total time = INP

// React'da INP optimization:
// - useTransition() — yield priority
// - startTransition() — non-urgent updates deferred
// - Concurrent rendering — keep main thread responsive
```

**Long Animation Frames API (LoAF):**

```typescript
// Browser API — long frames detection
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 50) {
      sendTelemetry({
        type: "long_frame",
        duration: entry.duration,
        renderTime: entry.renderTime,
        scripts: entry.scripts, // ← which JS caused
      });
    }
  }
}).observe({ type: "long-animation-frame", buffered: true });
```

LoAF — frames > 50ms (jank). React Concurrent rendering avoids this.

**Bundle size monitoring:**

```javascript
// webpack-bundle-analyzer
// vite-bundle-visualizer
// rollup-plugin-visualizer
```

Bundle size impact — dynamic imports initial bundle'ni sezilarli kamaytiradi, LCP yaxshilanadi. Aniq raqamlar ilova va tarmoq sharoitlariga bog'liq — bundle analyzer va Lighthouse bilan o'lchang.

**Code splitting telemetry:**

```tsx
const LazyDashboard = React.lazy(() => import("./Dashboard"));

// Track lazy load duration:
function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Profiler
        id="LazyDashboard"
        onRender={(id, phase, actualDuration) => {
          if (phase === "mount") {
            sendTelemetry({
              type: "lazy_mount",
              component: id,
              duration: actualDuration,
            });
          }
        }}
      >
        <LazyDashboard />
      </Profiler>
    </Suspense>
  );
}
```

**Dashboards va alerting:**

```typescript
// Aggregation per minute:
{
  metric: "react.render.duration",
  component: "ProductCard",
  p50: 5.2,
  p95: 18.7,
  p99: 42.3,
  count: 1543,
  sampleRate: 0.1,
}

// Alerts:
// - p95 > 50ms for 5 min → page oncall
// - error_rate > 1% → notify
// - LCP regression > 200ms → bisect deployments
```

**Real User Monitoring (RUM) vs Synthetic:**

| Type | Pros | Cons |
|------|------|------|
| RUM | Real user data, edge cases | Privacy, sampling cost |
| Synthetic (Lighthouse, WPT) | Reproducible, controlled | Lab conditions ≠ real |

Best practice: **both**. Synthetic for regression detection, RUM for user experience.

**Privacy considerations:**

- Hash user IDs
- Strip URL query params (PII risk)
- Geolocation — country-level only
- Compliance: GDPR, CCPA — opt-out support

**Cost analysis:**

Telemetry cost sessiya hajmi (Web Vitals events + render events + custom marks), sampling rate va kunlik aktiv foydalanuvchilar soniga bog'liq. Sampling rate (1-10%) ni ilova traffic'iga qarab tanlash kerak. Aniq cost provider (DataDog, Sentry, custom backend) pricing'iga bog'liq — har ilovada alohida hisoblash talab qilinadi.

**Server-Timing header — backend correlation:**

```http
Server-Timing: db;dur=53.2, render;dur=12.5, total;dur=67.0
```

Server-Timing — backend → frontend correlation. RUM tools auto-parse.

</details>

### Edge Cases

- **Browser back/forward cache (bfcache)**: Page restore — Web Vitals reset. `web-vitals` library handles this.
- **Tab switch / background**: Performance API'da timing inaccurate (browser throttle). Filter `document.visibilityState === "hidden"` events.
- **Service Worker cache**: First-page TTFB very low (cache hit). Distinguish first-load vs cached.

### Follow-up savollar

- "Sampling rate qanday tanlash?" — Trade-off: cost vs statistical power. 10% — typical. High-traffic site (1M+/day): 1-5%. Low-traffic: 50-100%.
- "Performance budget qanday belgilash?" — Web Vitals targets (LCP < 2.5s, INP < 200ms, CLS < 0.1 — Google web.dev'da e'lon qilingan). Bundle size va render budget — ilovaga bog'liq, Lighthouse va Profiler bilan aniqlang.
- "Regression detection qanday avtomatlash?" — CI'da Lighthouse CI (synthetic). Production'da RUM dashboards + alerts. Bisect on regression.

</details>

---

## QISM F: Code Splitting & Virtualization

### 20. `React.lazy` + `Suspense` qanday ishlaydi va u qachon ishlatiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`React.lazy(() => import("./Component"))` — komponentni dynamic import bilan code-split qiladi. Bundler (Webpack/Vite) shu chaqiruvni alohida chunk'ga ajratadi. Komponent **birinchi render**ida — chunk async yuklanadi (Promise). `Suspense` boundary bu Promise'ni catch qiladi va `fallback` ko'rsatadi. Yuklash tugagach — komponent render. Use case: route-level splitting, modal'lar, kam ishlatiladigan features (charts, editors). Initial bundle qisqaradi → LCP yaxshilanadi.

### To'liq tushuntirish

**API:**

```tsx
import { lazy, Suspense } from "react";

const Dashboard = lazy(() => import("./Dashboard"));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Dashboard />
    </Suspense>
  );
}
```

**Internal mexanizmi:**

1. `lazy(loader)` — `LazyComponent` element yaratadi (Fiber tag = `LazyComponent`)
2. Render paytida — `loader()` chaqiriladi, Promise qaytadi
3. Reconciler Promise'ni "throw" qiladi (special signal)
4. Suspense boundary catch qiladi, fallback render
5. Promise resolve bo'lganda — re-render, lazy component mount

**`React.lazy` requirements:**

- Default export'li modulni qaytarish: `import("./X").then(mod => ({ default: mod.X }))` (named import bilan)
- Top-level (komponent ichida emas — har render'da yangi `lazy()` instance)
- `Suspense` boundary majburiy

**Use cases:**

| Scenario | Code split | Sabab |
|----------|-----------|-------|
| Route'lar (`/dashboard`, `/settings`) | ✅ | Har route alohida chunk |
| Modal'lar / dialog'lar | ✅ | Faqat ochilganda yuklanadi |
| Editor (Monaco, CodeMirror) | ✅ | Katta bundle hajmi |
| Chart libraries | ✅ | Faqat dashboard'da |
| Footer komponent | ❌ | Initial render'da kerak |
| Header / Nav | ❌ | Always visible |

### Kod misoli

**Route-level splitting:**

```tsx
import { lazy, Suspense } from "react";
import { BrowserRouter, Routes, Route } from "react-router-dom";

const Home = lazy(() => import("./pages/Home"));
const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings = lazy(() => import("./pages/Settings"));
const NotFound = lazy(() => import("./pages/NotFound"));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageSkeleton />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
          <Route path="*" element={<NotFound />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}

// Initial bundle: App + router + skeleton
// Each route — alohida chunk, faqat navigation'da yuklanadi
```

**Modal — lazy:**

```tsx
import { lazy, Suspense, useState } from "react";

const SettingsModal = lazy(() => import("./SettingsModal"));

function App() {
  const [showSettings, setShowSettings] = useState(false);

  return (
    <>
      <button onClick={() => setShowSettings(true)}>Settings</button>
      {showSettings && (
        <Suspense fallback={<div>Loading settings...</div>}>
          <SettingsModal onClose={() => setShowSettings(false)} />
        </Suspense>
      )}
    </>
  );
}

// SettingsModal chunk — faqat tugma bosilganda yuklanadi
```

**Named exports — workaround:**

```tsx
// utils.ts:
export const Dashboard = () => <div>...</div>;
export const Settings = () => <div>...</div>;

// app.tsx:
const Dashboard = lazy(() =>
  import("./utils").then((mod) => ({ default: mod.Dashboard }))
);
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`lazy()` ichi:**

```typescript
// react/src/ReactLazy.js (mental model)
type LazyState =
  | { _status: -1; _result: () => Promise<{ default: T }> }   // Uninitialized
  | { _status: 0; _result: Promise<{ default: T }> }           // Pending
  | { _status: 1; _result: T }                                  // Resolved
  | { _status: 2; _result: Error };                             // Rejected

export function lazy<T>(ctor: () => Promise<{ default: T }>): LazyExoticComponent<T> {
  const payload: LazyState = {
    _status: -1,
    _result: ctor,
  };

  return {
    $$typeof: REACT_LAZY_TYPE,
    _payload: payload,
    _init: lazyInitializer,
  };
}

function lazyInitializer<T>(payload: LazyState): T {
  if (payload._status === -1) {
    // Uninitialized — start loading
    const ctor = payload._result;
    const thenable = ctor();
    thenable.then(
      (moduleObject) => {
        if (payload._status === 0) {
          payload._status = 1;
          payload._result = moduleObject.default;
        }
      },
      (error) => {
        payload._status = 2;
        payload._result = error;
      },
    );
    payload._status = 0;
    payload._result = thenable;
  }

  if (payload._status === 1) {
    return payload._result;
  }
  // Throw promise — Suspense catches
  throw payload._result;
}
```

**Reconciler `LazyComponent` handling:**

```typescript
// react-reconciler/src/ReactFiberBeginWork.js
function mountLazyComponent(current, workInProgress, elementType, renderLanes) {
  const lazyComponent = elementType;
  const payload = lazyComponent._payload;
  const init = lazyComponent._init;
  const Component = init(payload); // Throws if not resolved

  workInProgress.type = Component;
  // Continue mount as normal component
  return mountIndeterminateComponent(...);
}
```

**Suspense catch mechanism:**

```typescript
// When Promise thrown during render:
// 1. Reconciler catches in throwException
// 2. Find nearest Suspense boundary (return fiber chain)
// 3. Mark Suspense boundary as suspended
// 4. Render fallback in place of children
// 5. Attach .then() to Promise — re-render on resolve
```

**Webpack chunk naming:**

```typescript
// /* webpackChunkName: "settings" */
const Settings = lazy(() =>
  import(/* webpackChunkName: "settings" */ "./Settings")
);

// Output: dist/settings.[hash].chunk.js
```

Vite uses different syntax (manual chunks via `rollupOptions`).

**Network behavior:**

```
1. User navigates to /dashboard
2. React renders <Dashboard /> (lazy)
3. lazy.init() throws Promise — Suspense catches
4. Browser: <link rel="preload"> sent (modulepreload)
5. Network fetch: /assets/dashboard.[hash].js
6. Promise resolves
7. React re-renders → Dashboard mounted
```

**Caching — long-term:**

```javascript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          react: ["react", "react-dom"],
          ui: ["@radix-ui/react-*"],
          charts: ["recharts"],
        },
      },
    },
  },
});
```

Hash-based filenames + Cache-Control: max-age=31536000 → repeat visits — 0 network.

> **Performance note:** Code splitting initial bundle hajmini sezilarli kamaytiradi, bu esa LCP'ni yaxshilaydi (ayniqsa past tezlikdagi tarmoqlarda). Aniq raqamlar ilova va tarmoq tezligiga bog'liq — Profiler va Lighthouse bilan o'lchang.

</details>

### Edge Cases

- **Network failure during lazy load**: Promise rejects → Error Boundary catches. Fallback UI bilan retry tugmasi.
- **Multiple `Suspense` boundaries**: Inner boundary first catches. Outer boundary — only if inner re-throws.
- **`lazy()` + `useEffect`**: Lazy component mount'da — useEffect ham mount'da ishlaydi (hech narsa boshqacha emas).

### Follow-up savollar

- "`lazy` async function components'ga ishlaydi (RSC)?" — RSC alohida — server-side load. Lazy client-side bundle splitting uchun.
- "Race conditions — multiple lazy loads"? — `lazy()` o'zi cached (`_status`). Bir component bir necha marta render bo'lsa — bir Promise ishlatiladi.
- "`Suspense` boundary qayerga qo'yish kerak?" — Loose granularity (route'lar) yaxshi UX. Faqat one big Suspense — bitta lazy load butun page'ni bloklaydi.

</details>

---

### 21. Bundle size optimization texnikalari [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Bundle size'ni 5 ta darajada optimizatsiya qilinadi: (1) **Code splitting** — `React.lazy`, dynamic imports. (2) **Tree shaking** — ESM imports, named imports (named lodash o'rniga `lodash-es`). (3) **Library replacement** — Moment → date-fns, lodash → es-toolkit. (4) **Asset optimization** — image formats (WebP, AVIF), font subsetting. (5) **Bundle analysis** — `webpack-bundle-analyzer`, `vite-bundle-visualizer`. Production target ilovaga bog'liq — bundle analyzer va Lighthouse bilan o'lchab, performance budget belgilang.

### To'liq tushuntirish

**1. Code splitting:**

- Route-level
- Component-level (`React.lazy`)
- Library-level (manual chunks)

**2. Tree shaking:**

```typescript
// ❌ Default import — butun library bundle'ga kiradi
import _ from "lodash";
_.debounce(fn, 300);

// ✅ Named import (ESM library) — tree-shaking ishlaydi
import { debounce } from "lodash-es";
debounce(fn, 300);

// ✅ Direct path import — eng kichik bundle
import debounce from "lodash-es/debounce";
debounce(fn, 300);
```

**3. Library size comparison:**

| Original | Replacement | Natija |
|----------|------------|--------|
| `moment` | `date-fns` | Sezilarli kichikroq (tree-shakable) |
| `lodash` | `es-toolkit` | Kichikroq (ESM, tree-shakable) |
| `redux-toolkit` | `zustand` | Yengilroq bundle |
| `axios` | `fetch` (native) | 0 KB qo'shimcha (browser native) |
| `formik` | `react-hook-form` | Kichikroq bundle + kamroq re-render |

> Aniq hajm farqlari versiyaga bog'liq — `bundlephobia.com` da tekshiring.

**4. Asset optimization:**

```html
<!-- Modern image formats with fallback -->
<picture>
  <source srcset="hero.avif" type="image/avif" />
  <source srcset="hero.webp" type="image/webp" />
  <img src="hero.jpg" alt="Hero" />
</picture>
```

**5. Bundle analysis:**

```bash
# Vite
npm install -D rollup-plugin-visualizer
```

```typescript
// vite.config.ts
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: true, gzipSize: true, brotliSize: true }),
  ],
});
```

### Kod misoli

**Manual chunk splitting (Vite):**

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: (id) => {
          if (id.includes("node_modules")) {
            if (id.includes("react") || id.includes("scheduler")) {
              return "react-vendor";
            }
            if (id.includes("@radix-ui")) {
              return "ui-vendor";
            }
            if (id.includes("recharts") || id.includes("d3")) {
              return "charts-vendor";
            }
            return "vendor";
          }
        },
      },
    },
  },
});
```

**Dynamic imports for rare features:**

```tsx
// Heavy library — load on demand
async function exportToPdf(data: any) {
  const { jsPDF } = await import("jspdf");
  const doc = new jsPDF();
  doc.text(data.toString(), 10, 10);
  doc.save("export.pdf");
}

// jsPDF katta library — initial bundle'da yo'q
function ExportButton({ data }: Props) {
  return <button onClick={() => exportToPdf(data)}>Export PDF</button>;
}
```

**Conditional polyfills:**

```typescript
// Modern browsers — no polyfill
async function loadPolyfills() {
  if (!Array.prototype.toSorted) {
    await import("array.prototype.tosorted/auto");
  }
  if (!String.prototype.replaceAll) {
    await import("core-js/proposals/string-replace-all");
  }
}

await loadPolyfills();
```

**`react-helmet` removal (R19):**

```tsx
// ❌ Old (R18) — react-helmet (qo'shimcha bundle)
import { Helmet } from "react-helmet";
<Helmet><title>Page</title></Helmet>

// ✅ R19 — built-in, 0 KB extra
function MyPage() {
  return (
    <>
      <title>Page</title>
      <meta name="description" content="..." />
      <ArticleContent />
    </>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Tree shaking — sideEffects flag:**

```json
// package.json
{
  "name": "my-lib",
  "sideEffects": false
}
```

Bundler (Webpack/Rollup) — `sideEffects: false` → unused exports removed (dead code elimination).

```json
// Specific files have side effects:
{
  "sideEffects": ["./src/polyfills.ts", "*.css"]
}
```

**Tree shaking ishlamaydigan holatlar:**

```typescript
// ❌ Dynamic property access
const handlers = { create: () => {}, update: () => {} };
handlers[methodName](); // bundler can't statically determine

// ❌ CommonJS require (no static analysis)
const lib = require("lib");

// ❌ Side effects in module body
import "lib"; // ← may have side effects
```

**Brotli vs Gzip:**

| Compression | Size | Time |
|-------------|------|------|
| None | 800KB | - |
| Gzip | 250KB | Fast |
| Brotli (level 11) | 200KB | Slow (build-time OK) |

Brotli — better compression, supported in modern browsers.

**Dynamic imports — webpack/vite differences:**

```typescript
// Webpack:
import(/* webpackChunkName: "feature" */ "./Feature");

// Vite (no chunk naming syntax — uses Rollup):
import("./Feature"); // chunk auto-generated
// Manual via rollupOptions.output.manualChunks
```

**HTTP/2 push vs preload:**

```html
<!-- Preload critical resources -->
<link rel="preload" href="/main.js" as="script" />
<link rel="modulepreload" href="/dashboard.js" />
```

`modulepreload` — ESM modules. Fetch + parse, but don't execute.

**Bundle size budgets — CI enforcement:**

```yaml
# .github/workflows/ci.yml
- name: Bundle size check
  uses: andresz1/size-limit-action@v1
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
```

```json
// package.json
{
  "size-limit": [
    { "path": "dist/main.js", "limit": "100 KB" },
    { "path": "dist/vendor.js", "limit": "150 KB" }
  ]
}
```

**Source maps in production:**

- **Inline** — bundle ichida (no extra request, but bundle bigger)
- **External** — `.map` file (separate request, but only if devtools open)
- **Hidden source maps** — uploaded to error tracking (Sentry), not served to users

**Long-term caching:**

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        entryFileNames: "assets/[name].[hash].js",
        chunkFileNames: "assets/[name].[hash].js",
        assetFileNames: "assets/[name].[hash][extname]",
      },
    },
  },
});
```

Filename hash — content-based. Content o'zgarsa hash o'zgaradi → cache invalidation. CDN: `Cache-Control: max-age=31536000, immutable`.

</details>

### Edge Cases

- **Conditional imports** — `if (cond) await import(...)` — chunk yaratiladi, lekin lazy load.
- **Polyfills duplicate**: Bir necha polyfill bir xil API uchun — bundle'da takror. `core-js` strategy bilan tekshirish.
- **Vendored CSS — bundle ichida**: PostCSS modules — JS bundle'ga inline. Strategy: extract CSS via `vite-plugin-css-extract`.

### Follow-up savollar

- "Bundle analysis qachon qilish kerak?" — Production deploy oldidan har gal. CI'da automated check.
- "300KB bundle yomonmi?" — Context'ga bog'liq. Mobile 3G — sezilarli sekin (parse + execute vaqti uzoq). Desktop fiber — yetarli. Lighthouse va target qurilmada real test qiling.
- "Polyfill strategy modern browsers uchun?" — `<script type="module">` (ESM only) — modern browsers. Legacy fallback: `<script nomodule>`. Differential bundling.

</details>

---

### 22. Prefetching va preloading strategiyalari [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Prefetching — kelajakda kerak bo'ladigan resurslarni oldindan yuklab qo'yish. 4 ta strategiya: (1) **`<link rel="preload">`** — critical resources (fonts, hero image). (2) **`<link rel="prefetch">`** — keyingi sahifa resurslari (low-priority). (3) **`<link rel="modulepreload">`** — JS chunks (Vite auto). (4) **Programmatic prefetch** — hover'da, idle'da `import()` chaqirish. R19 yangi API'lar: `preload`, `preinit`, `prefetchDNS`, `preconnect` — komponent ichida resurs hint'lari.

### To'liq tushuntirish

**HTML link rel preferences:**

| Tag | Priority | Use case |
|-----|----------|----------|
| `preload` | High | Current page critical resources |
| `prefetch` | Low | Next page resources (idle prefetch) |
| `modulepreload` | High | ESM modules |
| `dns-prefetch` | Network | DNS resolution upfront |
| `preconnect` | Network | DNS + TCP + TLS handshake |

**1. Preload — critical resources:**

```html
<head>
  <link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin />
  <link rel="preload" href="/hero.webp" as="image" />
</head>
```

`as` attribute majburiy — browser priority hisoblash uchun.

**2. Prefetch — speculative:**

```html
<link rel="prefetch" href="/dashboard.js" as="script" />
```

Browser idle paytida yuklaydi (low priority). Keyingi navigation fast.

**3. Programmatic prefetch — hover bilan:**

```tsx
import { lazy } from "react";

const Dashboard = lazy(() => import("./Dashboard"));

function NavLink() {
  const prefetchDashboard = () => {
    // Initiate import — chunk download starts
    import("./Dashboard");
  };

  return (
    <a
      href="/dashboard"
      onMouseEnter={prefetchDashboard}
      onFocus={prefetchDashboard}
    >
      Dashboard
    </a>
  );
}

// User hovers tugmaga — chunk yuklanadi (background)
// User clicks — instant navigation (chunk already in browser cache)
```

**4. Idle prefetch:**

```tsx
useEffect(() => {
  if ("requestIdleCallback" in window) {
    requestIdleCallback(() => {
      import("./Dashboard");
      import("./Settings");
    });
  } else {
    setTimeout(() => {
      import("./Dashboard");
    }, 2000);
  }
}, []);
```

### Kod misoli

**React 19 resource hints (built-in):**

```tsx
import { preload, preinit, prefetchDNS, preconnect } from "react-dom";

function App() {
  // DNS prefetch — DNS resolution'ni oldindan bajaradi
  prefetchDNS("https://api.example.com");

  // TCP/TLS preconnect — DNS + TCP + TLS handshake'ni oldindan bajaradi
  preconnect("https://cdn.example.com");

  // Preload font/script
  preload("/fonts/inter.woff2", { as: "font", type: "font/woff2", crossOrigin: "anonymous" });

  // Preinit script — execute as soon as available
  preinit("/analytics.js", { as: "script" });

  return <AppContent />;
}
```

R19'da bu API'lar — render'dan tashqari ham chaqirilishi mumkin (idempotent).

**Hover prefetch with debounce:**

```tsx
import { useRef } from "react";

function NavLink({ href, children, importPath }: Props) {
  const timeoutRef = useRef<number | null>(null);

  const startPrefetch = () => {
    timeoutRef.current = window.setTimeout(() => {
      import(/* @vite-ignore */ importPath);
    }, 150); // Wait 150ms — avoid prefetch on quick hover
  };

  const cancelPrefetch = () => {
    if (timeoutRef.current !== null) {
      clearTimeout(timeoutRef.current);
      timeoutRef.current = null;
    }
  };

  return (
    <a
      href={href}
      onMouseEnter={startPrefetch}
      onMouseLeave={cancelPrefetch}
      onFocus={startPrefetch}
      onBlur={cancelPrefetch}
    >
      {children}
    </a>
  );
}
```

**IntersectionObserver — lazy prefetch:**

```tsx
function FooterLinks({ items }: { items: Link[] }) {
  const ref = useRef<HTMLElement>(null);

  useEffect(() => {
    if (!ref.current) return;
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            // Footer visible — likely user reaching bottom
            // Prefetch next-likely page
            import("./pages/AboutUs");
            observer.disconnect();
          }
        });
      },
      { rootMargin: "100px" }
    );
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  return <footer ref={ref}>...</footer>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Browser priority levels:**

```
Highest priority:
  - HTML
  - CSS (render-blocking)
  - Fonts (preload)
  - Hero images (preload as=image)

Medium:
  - Critical JS (entry chunks)
  - Above-fold images

Low:
  - Below-fold images (lazy load)
  - Prefetch resources
  - Analytics scripts
```

**`fetchpriority` attribute:**

```html
<img src="hero.webp" fetchpriority="high" alt="Hero" />
<link rel="prefetch" href="/dashboard.js" fetchpriority="low" />
```

R19'da:

```tsx
preload("/hero.webp", { as: "image", fetchPriority: "high" });
```

**Predictive prefetching algorithms:**

1. **Static analysis** (build-time): Determine likely next pages from link graph
2. **User behavior** (runtime): Track navigation patterns, prefetch top-3 candidates
3. **ML-based**: Tools like `quicklink` — predict based on viewport intersection + click patterns

**`<link rel="prerender">` (deprecated):**

Old API — full page render in background. Replaced by **Speculation Rules API**:

```html
<script type="speculationrules">
{
  "prerender": [{
    "where": { "href_matches": "/articles/*" },
    "eagerness": "moderate"
  }]
}
</script>
```

Browser evaluate'lab, sahifa render qiladi (DOM, CSS, JS). User click — instant.

**`navigator.connection` — adaptive prefetching:**

```typescript
interface NetworkInformation {
  saveData: boolean;
  effectiveType: string;
}

function shouldPrefetch(): boolean {
  const conn = (navigator as unknown as { connection?: NetworkInformation }).connection;
  if (!conn) return true;
  // Save data mode
  if (conn.saveData) return false;
  // Slow connection
  if (conn.effectiveType === "slow-2g" || conn.effectiveType === "2g") return false;
  return true;
}
```

Save user's mobile data on slow connections.

**Service Worker prefetching:**

```javascript
// service-worker.js
self.addEventListener("install", (event) => {
  event.waitUntil(
    caches.open("v1").then((cache) =>
      cache.addAll([
        "/dashboard.js",
        "/settings.js",
      ])
    )
  );
});
```

Service worker pre-caches resurslarni install paytida. Offline-first.

**Module preloading order:**

```html
<!-- Bad — main loaded after dependencies -->
<script type="module" src="/utils.js"></script>
<script type="module" src="/main.js"></script>

<!-- Good — modulepreload all dependencies -->
<link rel="modulepreload" href="/utils.js" />
<link rel="modulepreload" href="/main.js" />
<script type="module" src="/main.js"></script>
```

Vite avtomat `modulepreload` qo'shadi har built chunk uchun.

**Resource Timing API — prefetch effectiveness:**

```typescript
const entries = performance.getEntriesByType("resource");
entries.forEach((entry) => {
  if (entry.name.includes("dashboard.js")) {
    console.log({
      duration: entry.duration,
      transferSize: entry.transferSize,
      cached: entry.transferSize === 0, // ← prefetched
    });
  }
});
```

`transferSize === 0` — cached / prefetched.

**Cost-benefit:**

```
Prefetched but unused: wasted bandwidth
Not prefetched: slow next navigation

Strategy:
- Hover: ~70% conversion (high benefit)
- Idle: prefetch top-3 likely (50% conversion)
- Aggressive (all links): low conversion, high waste
```

**`requestIdleCallback` deadline:**

```typescript
requestIdleCallback((deadline) => {
  while (deadline.timeRemaining() > 0 && pendingPrefetches.length > 0) {
    const next = pendingPrefetches.shift();
    if (next) prefetchModule(next);
  }
});
```

Browser idle time'da prefetch — main thread block qilinmaydi.

</details>

### Edge Cases

- **CORS — preconnect/preload**: Cross-origin uchun `crossorigin` attribute majburiy. Without — different connection used.
- **Prefetch waterfall**: Bir prefetch boshqasidan keyin keladi (chained imports) — first prefetch'ning child chunks ham kerak.
- **Cache busting on deploy**: Filename hash o'zgarsa — prefetched eski chunk ishlamaydi. Service Worker invalidation kerak.

### Follow-up savollar

- "Mobile users uchun prefetch agressiv bo'lish kerakmi?" — Yo'q. `navigator.connection.saveData` va slow connection — disable. Data costs.
- "Speculation Rules production ready?" — Chrome'da stable, Firefox/Safari hozircha yo'q. Progressive enhancement strategy.
- "`Suspense` Inner prefetch nima?" — Component lazy yuklanayotganda — Suspense boundary bilan ishlatish. R19 `use()` Promise unwrap.

</details>

---

<a id="qism-g"></a>

### 23. Virtualization (windowing) konseptsiyasi [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Virtualization — uzun list'lar (10K+ items) uchun render optimizatsiyasi. Faqat **viewport'da ko'rinadigan** elementlar DOM'da qoladi (~10-20 ta). Scroll paytida elementlar dynamically qo'shiladi/o'chiriladi. Container'ning **total height** virtual hisoblanadi (item.height × count) — scrollbar correct bo'lishi uchun. Trade-off: memory + render arzon, lekin scroll position management va keyboard navigation murakkab. Library'lar: `react-window`, `@tanstack/react-virtual`, `react-virtuoso`.

### To'liq tushuntirish

**Muammo:**

```tsx
// 10K items render — 10K Fiber, 10K DOM nodes
// Initial render: 200-500ms
// Scroll: jank (each scroll tick — 10K DOM hit-test)
function Bad({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((i) => <li key={i.id}>{i.name}</li>)}
    </ul>
  );
}
```

**Yechim — windowing:**

```tsx
// Faqat ~20 visible items DOM'da
// Container height — virtual (10K × 50px = 500K pixels)
// Scroll: visible window slides
function Good({ items }: { items: Item[] }) {
  // Window: items[startIndex..endIndex]
  // Total height: items.length × itemHeight
  // Items y-position: index × itemHeight
}
```

**Algoritm:**

1. `containerRef` height va `scrollTop` o'lchanadi
2. `startIndex = Math.floor(scrollTop / itemHeight)`
3. `endIndex = Math.ceil((scrollTop + containerHeight) / itemHeight)`
4. `visible = items.slice(startIndex, endIndex + 1)`
5. Inner container `height = items.length × itemHeight` (scrollbar)
6. Items absolutely positioned: `top = index × itemHeight`

**Variants:**

| Type | Use case |
|------|---------|
| Fixed-size | Bir xil balandlikdagi items (eng oddiy) |
| Variable-size | Har xil balandlik (measurement kerak) |
| Grid | 2D virtualization (rows × columns) |
| Masonry | Pinterest-style — variable height columns |
| Infinite scroll | Lazy load + virtualization |

### Kod misoli

**`react-window` — fixed size:**

```tsx
import { FixedSizeList } from "react-window";

interface ListItemProps {
  index: number;
  style: React.CSSProperties;
}

function Row({ index, style }: ListItemProps) {
  return (
    <div style={style}>
      Row {index}: {items[index].name}
    </div>
  );
}

function VirtualList({ items }: { items: Item[] }) {
  return (
    <FixedSizeList
      height={600}
      width={300}
      itemCount={items.length}
      itemSize={50}
    >
      {Row}
    </FixedSizeList>
  );
}
```

**Variable size:**

```tsx
import { VariableSizeList } from "react-window";

function VirtualList({ items }: { items: Item[] }) {
  const getItemSize = (index: number) => items[index].height ?? 50;

  return (
    <VariableSizeList
      height={600}
      width={300}
      itemCount={items.length}
      itemSize={getItemSize}
    >
      {Row}
    </VariableSizeList>
  );
}
```

**`@tanstack/react-virtual` — modern API:**

```tsx
import { useVirtualizer } from "@tanstack/react-virtual";

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: "auto" }}>
      <div
        style={{
          height: virtualizer.getTotalSize(),
          position: "relative",
        }}
      >
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.key}
            style={{
              position: "absolute",
              top: 0,
              left: 0,
              width: "100%",
              height: virtualRow.size,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            {items[virtualRow.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Performance benefit:**

Virtualization'siz — item soni oshganda render vaqti chiziqli (yoki undan yuqori) o'sadi. 10K+ item'larda browser jank (main thread block) seziladi. Virtualization bilan — faqat visible window (~20-50 item) render qilinadi, shuning uchun item soni oshsa ham render vaqti deyarli o'zgarmaydi.

Breakeven point odatda 500-1000 items atrofida — undan kam item'larda virtualization overhead foydasiz. Aniq raqamlar item murakkabligi va qurilmaga bog'liq — Profiler bilan o'lchang.

**Memory:**

```
Without virtualization (10K items):
- 10K React elements
- 10K Fiber nodes
- 10K DOM nodes

With virtualization:
- 20-50 React elements (faqat visible window)
- 20-50 Fiber nodes
- 20-50 DOM nodes
```

Memory farqi sezilarli — Fiber va DOM node'lar soni mingdan o'ntalabga tushadi.

**Overscan — buffer for smooth scroll:**

```typescript
// Render 5 items above + 5 below visible window
// Scroll faster than render — items missing → jank
// Overscan: pre-render buffer to absorb scroll velocity
```

**Variable height — measurement strategy:**

```typescript
// Strategy 1: Estimated heights, measure on render
// - estimateSize() — rough estimate
// - On mount, ResizeObserver measures actual
// - Update virtualizer with measured size
// - Re-position items based on measurements

// Strategy 2: Pre-computed heights
// - Compute all heights upfront (expensive)
// - O(1) position lookup

// Strategy 3: Cached heights (lazy)
// - Measure on first scroll past
// - Cache for subsequent scrolls
```

**Keyboard accessibility:**

Virtualized list — keyboard navigation tricky:

```tsx
function AccessibleVirtualList({ items }: { items: Item[] }) {
  const [activeIndex, setActiveIndex] = useState(0);

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === "ArrowDown") {
      const next = Math.min(activeIndex + 1, items.length - 1);
      setActiveIndex(next);
      // Scroll into view
      virtualizer.scrollToIndex(next);
    } else if (e.key === "ArrowUp") {
      const prev = Math.max(activeIndex - 1, 0);
      setActiveIndex(prev);
      virtualizer.scrollToIndex(prev);
    }
  };

  return <div onKeyDown={handleKeyDown} tabIndex={0}>{/* ... */}</div>;
}
```

**Search/filter integration:**

```tsx
function FilteredVirtualList({ items, query }: Props) {
  const filtered = useMemo(
    () => items.filter((i) => i.name.includes(query)),
    [items, query]
  );

  // Virtualizer key bilan reset
  const virtualizer = useVirtualizer({
    count: filtered.length,
    // ...
  });

  // Filter o'zgarganda virtualizer reset emas
  // Faqat indices o'zgaradi (filtered.length)

  return <VirtualList items={filtered} virtualizer={virtualizer} />;
}
```

**Sticky headers in virtualized list:**

```typescript
// Scroll'da sticky header — section beginning'idan tushadi
// Virtual headers — track which section is currently in view
// Render only relevant section header
```

**Server-side rendering (SSR):**

Virtualization SSR'da murakkab:
- Server'da window calculation kerak
- Hydration paytida initial render 50 items emas, balki barchasi (visible-equivalent)
- `react-virtuoso` SSR support qiladi

**Bidirectional virtualization:**

Chat applications — yangi xabarlar tepa yoki pastdan kelishi mumkin:

```typescript
// Reverse scrolling
// Items insertion at top — scroll position must adjust
// Virtualizer must support "anchored" scroll
```

**`IntersectionObserver` alternative:**

```tsx
// Manual virtualization — IntersectionObserver bilan
function ManualVirtual({ items }: Props) {
  const [visibleIndices, setVisibleIndices] = useState<Set<number>>(new Set());

  // Observer per item — track visibility
  // ... (complex, library afzal)
}
```

> **Performance reality:** Virtualization — necessary evil. Adds complexity (scroll position, focus, accessibility). Faqat haqiqatan kerak bo'lganda (1000+ items).

</details>

### Edge Cases

- **Items dynamic content (images load)**: Image height o'zgaradi → re-measurement kerak. ResizeObserver bilan integration.
- **Virtualization + search**: Filter natijasi qisqa bo'lsa — virtualization overhead foydasiz. Threshold bilan toggle.
- **Mobile touch scroll**: Native momentum scroll — virtualizer rendering speed matters. Overscan boost mobile'da.

### Follow-up savollar

- "Virtualization vs pagination — qaysi qachon?" — Pagination — clear chunks (1-10, 11-20). Virtualization — seamless scroll. UX'ga bog'liq.
- "Why doesn't React virtualize automatically?" — Generic component requirements (height measurement, scroll behavior) — domain-specific. Library shu uchun mavjud.
- "Performance budget — virtualization qachon kerak?" — Initial render > 100ms or scroll FPS < 60. Profile va aniqlash.

</details>

---

### 24. Pure React virtualized list implementatsiyasi [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Pure React virtualization 5 ta qadam: (1) Container ref, scrollTop tracking via `onScroll`. (2) Visible range hisoblash: `startIndex = floor(scrollTop / itemHeight)`, `endIndex = startIndex + Math.ceil(containerHeight / itemHeight)`. (3) Inner container `height = items.length * itemHeight` (scrollbar). (4) Visible items absolutely positioned: `top = index * itemHeight`. (5) Overscan buffer (5-10 items har tomondan). Optimization: `useMemo` visible items, `React.memo` Row, throttled scroll.

### To'liq tushuntirish

**Algoritm qadamlari:**

```
1. Track scroll position (scrollTop)
2. Compute visible window:
   - startIndex = floor(scrollTop / itemHeight) - overscan
   - endIndex = ceil((scrollTop + containerHeight) / itemHeight) + overscan
   - clamp to [0, items.length]
3. Render only items[startIndex..endIndex]
4. Position each item absolutely:
   - top = index * itemHeight
5. Inner container takes total scroll height:
   - height = items.length * itemHeight
```

**Layout structure:**

```html
<div className="container" style="height: 600px; overflow: auto;">
  <!-- Inner — full virtual height -->
  <div style="height: 500000px; position: relative;">
    <!-- Visible items, absolutely positioned -->
    <div style="position: absolute; top: 0; height: 50px;">Row 0</div>
    <div style="position: absolute; top: 50px; height: 50px;">Row 1</div>
    <!-- ... -->
  </div>
</div>
```

### Kod misoli

**Pure React virtualization (fixed-size):**

```tsx
import {
  useState,
  useRef,
  useMemo,
  useCallback,
  useLayoutEffect,
  memo,
  CSSProperties,
} from "react";

interface VirtualListProps<T> {
  items: T[];
  itemHeight: number;
  containerHeight: number;
  overscan?: number;
  renderItem: (item: T, index: number) => React.ReactNode;
}

function VirtualList<T>({
  items,
  itemHeight,
  containerHeight,
  overscan = 5,
  renderItem,
}: VirtualListProps<T>) {
  const [scrollTop, setScrollTop] = useState(0);
  const containerRef = useRef<HTMLDivElement>(null);

  const handleScroll = useCallback((e: React.UIEvent<HTMLDivElement>) => {
    setScrollTop(e.currentTarget.scrollTop);
  }, []);

  // Visible range
  const { startIndex, endIndex, visibleItems, totalHeight } = useMemo(() => {
    const totalHeight = items.length * itemHeight;
    const start = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan);
    const end = Math.min(
      items.length,
      Math.ceil((scrollTop + containerHeight) / itemHeight) + overscan
    );

    const visibleItems = items.slice(start, end);

    return {
      startIndex: start,
      endIndex: end,
      visibleItems,
      totalHeight,
    };
  }, [items, itemHeight, scrollTop, containerHeight, overscan]);

  return (
    <div
      ref={containerRef}
      onScroll={handleScroll}
      style={{
        height: containerHeight,
        overflow: "auto",
        position: "relative",
      }}
    >
      <div style={{ height: totalHeight, position: "relative" }}>
        {visibleItems.map((item, idx) => {
          const actualIndex = startIndex + idx;
          const style: CSSProperties = {
            position: "absolute",
            top: actualIndex * itemHeight,
            left: 0,
            right: 0,
            height: itemHeight,
          };
          return (
            <div key={actualIndex} style={style}>
              {renderItem(item, actualIndex)}
            </div>
          );
        })}
      </div>
    </div>
  );
}

// Usage:
interface User {
  id: string;
  name: string;
  email: string;
}

function UserListPage({ users }: { users: User[] }) {
  return (
    <VirtualList
      items={users}
      itemHeight={50}
      containerHeight={600}
      renderItem={(user) => (
        <div style={{ padding: "8px" }}>
          <strong>{user.name}</strong> — {user.email}
        </div>
      )}
    />
  );
}
```

**Optimizatsiya — memoized Row:**

```tsx
interface RowProps<T> {
  item: T;
  index: number;
  height: number;
  renderItem: (item: T, index: number) => React.ReactNode;
}

const Row = memo(function Row<T>({ item, index, height, renderItem }: RowProps<T>) {
  const style: CSSProperties = {
    position: "absolute",
    top: index * height,
    left: 0,
    right: 0,
    height,
  };
  return <div style={style}>{renderItem(item, index)}</div>;
}) as <T>(props: RowProps<T>) => React.ReactElement;

// Replace inline render with <Row>
{visibleItems.map((item, idx) => {
  const actualIndex = startIndex + idx;
  return (
    <Row
      key={actualIndex}
      item={item}
      index={actualIndex}
      height={itemHeight}
      renderItem={renderItem}
    />
  );
})}
```

**Throttled scroll handler:**

```tsx
import { useRef, useCallback } from "react";

function useThrottledScroll(handler: (scrollTop: number) => void, ms = 16) {
  const lastTimeRef = useRef(0);
  const rafRef = useRef<number | null>(null);

  return useCallback(
    (e: React.UIEvent<HTMLDivElement>) => {
      const now = Date.now();
      const scrollTop = e.currentTarget.scrollTop;

      if (now - lastTimeRef.current >= ms) {
        handler(scrollTop);
        lastTimeRef.current = now;
      } else {
        if (rafRef.current !== null) cancelAnimationFrame(rafRef.current);
        rafRef.current = requestAnimationFrame(() => {
          handler(scrollTop);
          lastTimeRef.current = Date.now();
        });
      }
    },
    [handler, ms]
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Variable size implementation:**

```tsx
interface VirtualListVariableProps<T> {
  items: T[];
  estimatedItemHeight: number;
  containerHeight: number;
  getItemHeight: (index: number) => number;
  renderItem: (item: T, index: number) => React.ReactNode;
}

function VirtualListVariable<T>({
  items,
  estimatedItemHeight,
  containerHeight,
  getItemHeight,
  renderItem,
}: VirtualListVariableProps<T>) {
  const [scrollTop, setScrollTop] = useState(0);

  // Cumulative offsets
  const offsets = useMemo(() => {
    const arr: number[] = [];
    let sum = 0;
    for (let i = 0; i < items.length; i++) {
      arr[i] = sum;
      sum += getItemHeight(i);
    }
    return arr;
  }, [items.length, getItemHeight]);

  const totalHeight = useMemo(() => {
    return offsets[items.length - 1] + getItemHeight(items.length - 1);
  }, [offsets, items.length, getItemHeight]);

  // Binary search for startIndex
  const startIndex = useMemo(() => {
    let lo = 0;
    let hi = items.length - 1;
    while (lo < hi) {
      const mid = Math.floor((lo + hi) / 2);
      if (offsets[mid] < scrollTop) lo = mid + 1;
      else hi = mid;
    }
    return Math.max(0, lo - 1);
  }, [scrollTop, offsets, items.length]);

  const endIndex = useMemo(() => {
    let i = startIndex;
    while (i < items.length && offsets[i] < scrollTop + containerHeight) {
      i++;
    }
    return Math.min(items.length, i + 1);
  }, [startIndex, scrollTop, containerHeight, offsets, items.length]);

  // ... render
}
```

**Performance optimizations:**

```typescript
// 1. RAF-throttled scroll handler
const handleScroll = (e) => {
  if (!rafRef.current) {
    rafRef.current = requestAnimationFrame(() => {
      setScrollTop(e.target.scrollTop);
      rafRef.current = null;
    });
  }
};

// 2. Passive event listener
useEffect(() => {
  const el = containerRef.current;
  if (!el) return;
  el.addEventListener("scroll", handleScroll, { passive: true });
  return () => el.removeEventListener("scroll", handleScroll);
}, []);

// 3. Use transform for positioning (GPU-accelerated)
style={{ transform: `translateY(${index * itemHeight}px)` }}
// instead of:
style={{ top: index * itemHeight }}
```

**Sticky scroll on data change:**

```tsx
const prevItemsLength = useRef(items.length);

useLayoutEffect(() => {
  // Items prepended (e.g., new chat messages on top)
  if (items.length > prevItemsLength.current) {
    const diff = items.length - prevItemsLength.current;
    const adjustment = diff * itemHeight;
    if (containerRef.current) {
      containerRef.current.scrollTop += adjustment;
    }
  }
  prevItemsLength.current = items.length;
}, [items.length, itemHeight]);
```

**Resize handling:**

```tsx
useLayoutEffect(() => {
  if (!containerRef.current) return;
  const observer = new ResizeObserver((entries) => {
    for (const entry of entries) {
      // Container resized — re-compute visible range
      forceUpdate();
    }
  });
  observer.observe(containerRef.current);
  return () => observer.disconnect();
}, []);
```

**Accessibility — ARIA roles:**

```tsx
<div role="list" aria-label="User list" aria-rowcount={items.length}>
  {visibleItems.map((item, idx) => {
    const actualIndex = startIndex + idx;
    return (
      <div
        key={actualIndex}
        role="listitem"
        aria-rowindex={actualIndex + 1}
        aria-posinset={actualIndex + 1}
        aria-setsize={items.length}
      >
        {/* ... */}
      </div>
    );
  })}
</div>
```

**Memoizatsiya cost — full implementation:**

```tsx
// Render cost per scroll tick:
// 1. Visible range calculation: O(1) for fixed, O(log n) for variable
// 2. Items slice: O(window_size)
// 3. React reconciliation: O(window_size)
// 4. DOM updates: minimal (transform-only changes)

// Total: 60fps target 16.67ms — visible window kichik bo'lganda yetarli
```

**Comparison with `react-window`:**

| Feature | Pure React | `react-window` |
|---------|-----------|----------------|
| Bundle size | 0 | ~3KB |
| API | Custom | Standard |
| Variable size | Manual | `VariableSizeList` |
| Grid | Manual | `FixedSizeGrid` |
| Imperative scroll | Manual | `scrollToItem` |
| Accessibility | Manual | Some default |
| SSR | Possible | Limited |

For production: `@tanstack/react-virtual` (modern API, hooks-based) yoki `react-virtuoso` (richer features).

</details>

### Edge Cases

- **Items length 0**: Empty container, no items rendered. Total height 0.
- **Container 0 height**: Visible range empty. No items rendered.
- **Scroll beyond range**: Browser clamps scrollTop. Last items rendered at bottom.
- **Float itemHeight**: Sub-pixel positioning — `transform` afzal `top` o'rniga (GPU sub-pixel accurate).

### Follow-up savollar

- "Variable height items uchun strategy?" — Estimate + measure on render. ResizeObserver bilan track. Cache measurements.
- "Scroll restoration — back navigation?" — `window.scrollTo` faqat document scroll. Container scroll — manual save/restore (sessionStorage).
- "Production'da pure React virtualization yozish kerakmi?" — Yo'q, library afzal (battle-tested edge cases). Faqat custom requirements bo'lsa (specific scroll behavior).

</details>

---

## QISM G: Web Vitals

### 25. Core Web Vitals (LCP, INP, CLS) nima va React'da ularni qanday optimize qilamiz? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Core Web Vitals** — Google tomonidan e'lon qilingan, real foydalanuvchi tajribasini o'lchaydigan uchta metric: **LCP** (Largest Contentful Paint — eng katta visible element render bo'lish vaqti), **INP** (Interaction to Next Paint — user interaction'dan keyingi paint'gacha bo'lgan latency), va **CLS** (Cumulative Layout Shift — kutilmagan layout shift'lar summasi). Bu metric'lar **React'ga emas, browser'ga tegishli**, lekin React patterns ularga ta'sir qiladi: lazy loading, image priority, font loading — LCP; concurrent rendering, transitions — INP; reserved space, skeleton — CLS.

### To'liq tushuntirish

**Web Vitals — qisqacha:**

| Metric | Nima o'lchaydi | "Good" threshold |
|--------|----------------|------------------|
| **LCP** | Viewport'dagi eng katta content element paint bo'lish vaqti | < 2.5s |
| **INP** | Page lifetime davomida user interaction'dan keyingi paint'gacha vaqt (longest, top 2%'siz) | < 200ms |
| **CLS** | Visible elements'ning kutilmagan layout shift'lari summasi (visual stability) | < 0.1 |

> **Eslatma:** Threshold'lar Google web.dev'da e'lon qilingan. INP **2024 mart**da FID (First Input Delay)'ni almashtirgan. Aniq raqamlar uchun Google docs (`web.dev/vitals`) tekshiring.

**Bu React metric'lari emas — browser'ning user experience metric'lari.** React patterns ularga ta'sir qiladi, lekin metric Google'ning Web Vitals tashabbusiga tegishli.

**LCP — React optimizatsiyalari:**

```tsx
// 1. Hero image priority — fetchpriority="high"
function Hero() {
  return (
    <img
      src="/hero.jpg"
      alt="Hero"
      fetchPriority="high"  // R19 attribute — browser hint
      width={1200}
      height={600}
    />
  );
}

// 2. Preload critical resources
// public/index.html yoki Next.js head:
<link rel="preload" as="image" href="/hero.jpg" fetchpriority="high" />
<link rel="preload" as="font" href="/font.woff2" crossOrigin="anonymous" />

// 3. Code splitting — initial bundle kichik
const HeavyChart = lazy(() => import("./HeavyChart"));

// 4. SSR/RSC — server'da render → HTML tezroq paint
// Next.js, Remix — first render = HTML, JS keyin hydrate
```

**INP — React optimizatsiyalari:**

```tsx
// 1. startTransition — heavy update'larni transition'ga
function Search() {
  const [query, setQuery] = useState("");
  const [isPending, startTransition] = useTransition();

  return (
    <input onChange={(e) => {
      setQuery(e.target.value); // sync — input responsive
      startTransition(() => {
        // Heavy filter — interruptible
      });
    }} />
  );
}

// 2. useDeferredValue — output stale
function Results({ query }: { query: string }) {
  const deferredQuery = useDeferredValue(query);
  const results = expensiveSearch(deferredQuery);
  return <List items={results} />;
}

// 3. Concurrent rendering — large render uziladi, input prioritet
// createRoot (R18+) — default Concurrent
```

**CLS — React optimizatsiyalari:**

```tsx
// 1. Image dimensions
<img src="/photo.jpg" width={400} height={300} /> // aspect-ratio reserved

// 2. Skeleton/placeholder — same dimensions
function UserCard({ user }: { user?: User }) {
  if (!user) {
    return <div style={{ width: 300, height: 100 }} className="skeleton" />;
  }
  return <div style={{ width: 300, height: 100 }}>{user.name}</div>;
}

// 3. Suspense fallback — same height
<Suspense fallback={<div style={{ minHeight: 400 }}>Loading...</div>}>
  <Content />
</Suspense>

// 4. Avoid late-injected content
// ❌ Banner suddenly appears above content → shifts everything down
// ✅ Reserve space initially yoki bottom'da
```

### Kod misoli

**Web Vitals real-time monitoring (R19, `web-vitals` library):**

```tsx
import { useEffect } from "react";
import { onLCP, onINP, onCLS, type Metric } from "web-vitals";

function reportWebVital(metric: Metric) {
  // Analytics'ga yuborish
  fetch("/api/vitals", {
    method: "POST",
    body: JSON.stringify({
      name: metric.name,
      value: metric.value,
      rating: metric.rating, // "good" | "needs-improvement" | "poor"
      id: metric.id,
    }),
    keepalive: true,
  });
}

function App() {
  useEffect(() => {
    onLCP(reportWebVital);
    onINP(reportWebVital);
    onCLS(reportWebVital);
  }, []);

  return <Main />;
}
```

**Next.js — built-in Web Vitals reporting:**

```tsx
// app/layout.tsx (App Router) yoki pages/_app.tsx
export function reportWebVitals(metric: Metric) {
  // Next.js avtomat chaqiradi
  if (metric.name === "LCP") {
    // ...
  }
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**INP vs FID — nima farq?**

- **FID** (eski metric) — faqat **birinchi** interaction'ning input delay'i (event handler boshlanishigacha). Paint'gacha emas.
- **INP** (yangi) — barcha interaction'larning **paint'gacha** vaqti. Worst case (top 2% trimmed) hisobga olinadi.

INP — more realistic UX measure (user input'idan to natijani ko'rgunigacha). FID 2024 mart'da deprecated.

**INP fazalari:**

```
User input → event listener → React state update → render → commit → paint
    |             |                  |               |        |       |
    |--- Input ---|--- Processing ---|--- Render ----|--Commit-|-Paint-|
              delay              presentation delay

INP = total time from input to next paint
```

**React'ning INP'ga real ta'siri:**

1. **Long render** — heavy component tree render uzoq → paint kechikadi
2. **Sync state update** — handler ichida sync update → render blokirovka
3. **Layout thrashing** — `useLayoutEffect` ichida sync DOM measure/write

**Optimization decision tree:**

| Muammo | Yechim |
|--------|--------|
| Input lag | `startTransition` heavy update'lar uchun |
| Large list render | Virtualization |
| Sync data fetch | Suspense + `use()` (R19) |
| Layout shift | Reserve space (dimensions, aspect-ratio) |
| Late-loading font | `font-display: swap` + preload |
| Late hero image | `fetchpriority="high"` + preload |

**LCP element identifikatsiya:**

Browser DevTools → Performance → Web Vitals lane → LCP marker (largest paint element).

```javascript
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log("LCP element:", entry.element);
    console.log("LCP time:", entry.startTime);
  }
}).observe({ type: "largest-contentful-paint", buffered: true });
```

**CLS shift score calculation:**

```
shift score = impact fraction × distance fraction
```

- **Impact fraction** — viewport'ning qanchasi affect bo'ldi (0-1)
- **Distance fraction** — element qancha siljidi (viewport'ga nisbatan)

Misol: viewport 75% i 25% siljisa → CLS = 0.75 × 0.25 = 0.1875 (poor).

**Layout shift exclusion** — user interaction'dan keyingi 500ms ichida bo'lgan shift'lar CLS'ga kirmaydi (expected).

**React Compiler va INP:**

Compiler auto-memoization → unnecessary re-renders kamayadi → CPU work kamayadi → INP yaxshilanadi (especially keyboard input'da).

**RSC (R19 Server Components) va Web Vitals:**

- **LCP**: Server'da render → HTML JS'siz keladi → eng tez visible content
- **CLS**: Server'da to'liq HTML → client'da reflow kam
- **INP**: Client interactivity kechroq hydrate qilinadi, lekin static content user'ga darhol available

</details>

### Edge Cases

- **SPA navigation INP**: First page load Web Vitals'ga kiradi. Soft navigation'lar (client-side route) — alohida metric (Navigation Timing API). React Router/Next.js avtomat tracking.
- **Background tab**: Vitals faqat foreground page'da o'lchanadi. Background renderlar ignorelanadi.
- **Bot traffic**: Headless browsers (Lighthouse, PageSpeed Insights) sintetik raqamlar beradi. RUM (Real User Monitoring) — actual user data.
- **CLS animation exception**: `transform` orqali animation CLS'ga kirmaydi (compositor layer). `top`/`left` siljish — kiradi.

### Follow-up savollar

- "INP'ni qanday production'da o'lchaymiz?" — `web-vitals` library + analytics endpoint. Google Analytics 4 / Vercel Analytics / Datadog RUM avtomat tracking qiladi.
- "Lighthouse va RUM farqi?" — Lighthouse: sintetik (controlled env). RUM: real users (real devices, networks). Production decision uchun RUM majburiy.
- "Web Vitals — SEO'ga ta'siri bormi?" — Ha, Google Search 2021'dan Core Web Vitals ranking signal sifatida ishlatadi (Page Experience update). Aniq weight Google docs.

</details>

---

### 26. React Compiler beta status va `'use memo'` direktiv qanday ishlatiladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React Compiler 2026 boshida hali **rasman beta** statusda (production'da Meta'da ishlatiladi, lekin barcha React app'lar uchun stable e'lon qilinmagan). O'rnatish: `babel-plugin-react-compiler` Babel/Vite/Next.js'ga qo'shiladi va `eslint-plugin-react-compiler` Rules of React violations'ni dev'da warn qiladi. Default — **opt-in per directory yoki per file** (config'ga `compilationMode` orqali). File-level granular control uchun: `"use memo"` directive (yoqish), `"use no memo"` (o'chirish). Production'ga chiqarishdan oldin ESLint plugin clean bo'lishi va critical path'larni Profiler bilan tasdiqlash kerak.

### To'liq tushuntirish

**Beta status (2026):**

- Meta production'da (Instagram, Facebook) ishlatadi
- Public release — Beta (stable e'lon qilinmagan)
- ESLint plugin va Babel plugin npm'da mavjud: `babel-plugin-react-compiler`, `eslint-plugin-react-compiler`
- React 19 minimum (yangi hook `useMemoCache` ishlatiladi)

> **Status tekshiruvi:** React rasmiy blog (`react.dev/blog`) yoki Compiler GitHub repo (`facebook/react`) status'ni doim aniqlang. Stable e'lon qilingan bo'lishi mumkin.

**Setup (Babel):**

```javascript
// babel.config.js
module.exports = {
  plugins: [
    ['babel-plugin-react-compiler', {
      compilationMode: 'annotation', // yoki 'all' yoki 'infer'
      target: '19',
    }],
  ],
};
```

**`compilationMode` opsiyalari:**

| Mode | Behavior |
|------|----------|
| `all` | Barcha komponent'lar/hook'lar compile qilinadi (default) |
| `annotation` | Faqat `"use memo"` directive'li fayllar |
| `infer` | Heuristic — komponent shaklini avtomat tan oladi |

**File-level directive — `"use memo"`:**

```tsx
"use memo"; // Faylda Compiler yoqilgan

function ProductList({ products }: Props) {
  // Avtomat memoization
  const filtered = products.filter(p => p.active);
  return <ul>{filtered.map(...)}</ul>;
}
```

**File-level opt-out — `"use no memo"`:**

```tsx
"use no memo"; // Compiler bu fayl'ni skip qiladi

function LegacyComponent() {
  // Manual useMemo/useCallback
  // Yoki Rules of React violation (Compiler crash bermasin)
}
```

**Function-level directive:**

```tsx
function SpecificComponent() {
  "use no memo";
  // Bu function uchun Compiler off
  // ...
}
```

### Kod misoli

**Next.js (App Router) setup:**

```javascript
// next.config.js
const nextConfig = {
  experimental: {
    reactCompiler: true, // Built-in Next.js support
  },
};

module.exports = nextConfig;
```

**Vite setup:**

```javascript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react({
      babel: {
        plugins: [['babel-plugin-react-compiler', {}]],
      },
    }),
  ],
});
```

**ESLint plugin:**

```javascript
// .eslintrc.json
{
  "plugins": ["react-compiler"],
  "rules": {
    "react-compiler/react-compiler": "error"
  }
}
```

ESLint Rules of React violations'ni warn qiladi:

```tsx
function Component() {
  let value = computeValue();
  value = transform(value); // ❌ Mutation outside hook — ESLint error
  return <div>{value}</div>;
}
```

**Gradual adoption strategy:**

```javascript
// 1. ESLint plugin avval — violations'ni topish
{
  "plugins": ["react-compiler"],
  "rules": { "react-compiler/react-compiler": "warn" } // warn, not error
}

// 2. Annotation mode — opt-in directorylar
// babel.config.js
{
  compilationMode: 'annotation',
}

// 3. Toza fayllarga "use memo" qo'shish
// "use memo"
// export function CleanComponent() { ... }

// 4. Hammasi compile bo'lganda — full mode
{
  compilationMode: 'all',
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Compiler internal hook — `useMemoCache`:**

Compiler ishlab chiqargan kod `useMemoCache(N)` hook'ni chaqiradi (NOT `useMemo` analog — alohida React internal). Bu hook fiber'da N-slot array allocate qiladi.

```tsx
// Compiler output (mental model):
function Component(props) {
  const $ = useMemoCache(4); // 4 slot allocated
  let t0;

  if ($[0] !== props.items) {
    t0 = props.items.filter(p => p.active);
    $[0] = props.items;
    $[1] = t0;
  } else {
    t0 = $[1];
  }

  let t1;
  if ($[2] !== t0) {
    t1 = <List items={t0} />;
    $[2] = t0;
    $[3] = t1;
  } else {
    t1 = $[3];
  }

  return t1;
}
```

`useMemoCache` — `useMemo` emas. Public API emas, lekin Compiler bilan tushuntirilganda muhokama qilinadi.

**HIR (High-level IR) Pipeline:**

```
Source → Parse → CFG → HIR → Reactive Scopes → SSA → Optimization → Codegen
                              |
                              +-- Mutation analysis
                              +-- Aliasing analysis
                              +-- Effect detection
```

Compiler statik analiz orqali:

1. **Reactive scopes** — qaysi expressions reactive (props/state'ga bog'liq)
2. **Mutation tracking** — qaysi values mutate qilinadi
3. **Memoization placement** — har scope uchun cache slot
4. **Dependency inference** — har scope qaysi inputs'ga bog'liq

**Compiler limits — qachon skip qiladi:**

- Try/catch in render → conservative skip
- Mutation outside scope (ref reassignment) → skip
- Hooks conditional → ESLint error, Compiler skip
- Non-pure render → analysis abort

**`'use memo'` vs `'use no memo'` precedence:**

- File-level beats config (annotation mode'da)
- Function-level beats file-level
- Hooks within annotated component inherit (faqat Rule of Hooks o'qiladi)

**Bundle impact:**

Compiler kod oshiradi (cache array, conditional checks). Lekin re-render kamayganligi sababli runtime CPU work kamayadi. Net trade-off ilovaga bog'liq — bundle analyzer + Profiler bilan o'lchang.

**Migration checklist:**

```
1. React 19 minimum
2. ESLint plugin add → fix all violations (Rules of React)
3. Mutating patterns refactor (let → const + immutable ops)
4. Annotation mode → critical paths "use memo"
5. Profile before/after — Profiler API
6. Test in production-like env (Strict Mode 2x render)
7. Gradual rollout → all mode
```

</details>

### Edge Cases

- **TypeScript types**: Compiler TS code'ni qabul qiladi (Babel + TS preset). Type errors compile'ga ta'sir qilmaydi (TS alohida tsc / @babel/preset-typescript).
- **React Native**: Compiler qo'llab-quvvatlanadi (Hermes engine compatible). Babel plugin RN'da ham ishlaydi.
- **Server Components**: RSC'larda Compiler kerakmas (qayta render bo'lmaydi). Client Components'da ishlatiladi.
- **Hot reload**: Vite/Webpack HMR Compiler bilan ishlaydi (re-compile on save).

### Follow-up savollar

- "Compiler ishlatsam manual `useMemo`/`useCallback` o'chirish kerakmi?" — Yo'q, majburiy emas. Compiler ularni "no-op" sifatida ko'radi. Yangi kod uchun manual hook'larni yozmaslik mumkin, lekin existing kod ishlaydi.
- "Compiler buyruq beradigan API'lar bormi?" — `'use memo'`, `'use no memo'` directive'lar. Hozircha boshqa runtime API yo'q.
- "Ish faoliyatim Compiler chiqishigacha — manual yozaverishim kerakmi?" — Ha, Profiler bilan critical path'larni manual memoize. Compiler stable bo'lgach migration.

</details>

---

### 27. Bundle analysis — qanday tools va metrics [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Bundle analysis tools: **rollup-plugin-visualizer** (Vite), **webpack-bundle-analyzer** (Webpack), **source-map-explorer** (CLI). Metrics: **gzipped size** (network), **brotli size** (modern), **parse + execute time** (JS engine), **dependency tree** (tree-shaking gaps). Threshold ilovaga bog'liq — Lighthouse va bundle analyzer bilan aniqlang.

### Kod misoli

```bash
# Vite — install
npm i -D rollup-plugin-visualizer

# vite.config.ts
import { visualizer } from "rollup-plugin-visualizer";
export default defineConfig({
  plugins: [react(), visualizer({ open: true, gzipSize: true })],
});

# Build → opens treemap in browser
npm run build
```

```bash
# Source map explorer — any bundler
npm i -D source-map-explorer
npx source-map-explorer dist/assets/*.js

# Output: hierarchical breakdown by file
```

```bash
# Bundle phobia — check package size before install
npx bundle-phobia <package-name>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Common bundle bloat sources:**

1. **Lodash** — full import vs `lodash-es` tree-shake
2. **Moment.js** — 280kb. Replace with `date-fns` (modular) or `dayjs` (2kb)
3. **Polyfills** — bundle for old browsers
4. **Source maps** — production should not include
5. **Unused exports** — re-exports break tree-shake

**Tree-shake gotchas:**

```typescript
// ❌ Side effect import — entire module loaded
import "library";

// ✅ Specific import
import { specificFn } from "library";

// ❌ Default export object
import lib from "library";  // entire object
lib.fn();

// ✅ Named exports
import { fn } from "library";  // tree-shake'd
```

**Webpack bundle analyzer:**

```javascript
// webpack.config.js
const { BundleAnalyzerPlugin } = require("webpack-bundle-analyzer");
module.exports = {
  plugins: [
    process.env.ANALYZE && new BundleAnalyzerPlugin(),
  ].filter(Boolean),
};

// Run: ANALYZE=true npm run build
```

**Code splitting impact:**

```tsx
// Before:
const Heavy = require("./Heavy");  // initial bundle +100kb

// After:
const Heavy = lazy(() => import("./Heavy"));  // separate chunk
```

**Real-world thresholds (2026):**

| Size (gzipped) | Status |
|----------------|--------|
| <50kb | Fast |
| 50-100kb | Moderate |
| 100-200kb | Slow on 4G mobile |
| 200kb+ | Very slow |

LCP (Largest Contentful Paint) — directly impacted by initial bundle.

**CI integration:**

```yaml
# GitHub Actions example
- name: Bundle size check
  uses: andresz1/size-limit-action@v1
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
```

`size-limit` package — fail CI if bundle exceeds threshold.

</details>

### Edge Cases

- **CSS-in-JS bundles**: Some libraries (styled-components) include runtime. Vanilla CSS no runtime cost.
- **Image imports**: Inline base64 — bundles in JS. Prefer `<img src>` for large.
- **JSON imports**: Bundled in JS. Move to API for large datasets.

### Follow-up savollar

- "When code-splitting hurts?" — Too aggressive — many small chunks, network overhead. Aim for 50-100kb chunks.
- "RSC bundle size?" — Server components — 0 bundle (server-only). Client components — bundled.

</details>

---

## Xulosa

Bu fayl React performance'ning to'liq spektrini qamrab oldi:

- **QISM A — React.memo** (4 savol): Shallow comparison, custom comparators, `useCallback` paired, bypass scenarios
- **QISM B — Re-render Mechanics** (4 savol): 4 ta render trigger, parent → child cascade, bailout algorithm, output trace
- **QISM C — useMemo/useCallback** (3 savol): Decision tree, over-engineering, Compiler era
- **QISM D — React Compiler** (3 savol): HIR + reactive scopes, Rules of React, limitations
- **QISM E — Profiling** (3 savol): DevTools Profiler, programmatic `<Profiler>`, production monitoring
- **QISM F — Code Splitting & Virtualization** (5 savol): `React.lazy`, bundle optimization, prefetching, virtualization concept va implementation
- **QISM G — Web Vitals** (2 savol): Core Web Vitals (LCP/INP/CLS) va React optimizatsiya patterns, React Compiler beta status va directive'lar

**Asosiy mental model'lar:**

1. **Bailout — kichik xarajat, katta foyda** (Fiber.lanes + childLanes)
2. **Memo + useCallback — paired juftlik** (props stability)
3. **Compiler avtomat memoization** — manual ehtiyoji kamayadi
4. **Profile avval, optimize keyin** — premature optimization xato
5. **Virtualization faqat 1000+ items** uchun

**Keyingi fayl:** `06-concurrent-suspense.md` — Concurrent React, startTransition, Suspense, Streaming SSR.

