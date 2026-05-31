# Bo'lim 4: Reconciliation Algorithm

> Reconciliation — React'ning eski Fiber tree va yangi Element tree o'rtasida farq topib, qaysi DOM operatsiyalarini bajarishni aniqlovchi algoritm. Akademik tree diff O(n³) bo'lardi — React **ikkita heuristic** bilan amaliy O(n)'ga tushirgan: **different types = rebuild** va **keys = stable identity**. Bu bo'lim algoritmni qadam-baqadam yoritadi: type comparison, sibling matching, bailout sabablari, update propagation.

---

## Mundarija

- [Reconciliation Nima](#reconciliation-nima)
- [Naive O(n³) vs React O(n) Heuristics](#naive-on-vs-react-on-heuristics)
- [Type Comparison](#type-comparison)
- [Sibling Matching — Keyless](#sibling-matching--keyless)
- [Sibling Matching — Keyed](#sibling-matching--keyed)
- [Key-based Identity](#key-based-identity)
- [Bailout Algorithm](#bailout-algorithm)
- [Update Propagation](#update-propagation)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Reconciliation Nima

### Nazariya

**Reconciliation** — React'ning **diffing algoritmi**. Komponent qayta render bo'lganda, yangi React Element tree (komponent funksiyasi qaytarib bergani) va eski Fiber tree (oldingi commit'da saqlangani) solishtiriladi. Algoritm aniqlaydi:

1. Qaysi Fiber'lar **yangilanishi** kerak (props o'zgargan)
2. Qaysi Fiber'lar **insert qilinishi** kerak (yangi qo'shilgan)
3. Qaysi Fiber'lar **o'chirilishi** kerak (yo'qolgan)
4. Qaysi Fiber'lar **bir joydan boshqasiga ko'chirilishi** kerak (lists'da reorder)

Natijada har Fiber'ga mos **flag** o'rnatiladi (`Update`, `Placement`, `ChildDeletion`, va h.k.). Commit Phase'da shu flag'lar asosida real DOM operatsiyalari bajariladi.

**Reconciler — bu nima emas:**

- Reconciler **DOM'ga to'g'ridan tegmaydi** — faqat Fiber tree'da flag o'rnatadi
- Reconciler **komponent funksiyasini chaqirmaydi** — bu ish `beginWork`da bajariladi (hooks ham shunda ishga tushadi)
- Reconciler **render qilmaydi** — u faqat eski va yangi tree o'rtasida farqni hisoblaydi

Reconciler — bu **diff calculator**. Real ish (rendering, DOM mutation, effect chaqirish) — boshqa bosqichlarda.

### Reconciler qayerda chaqiriladi

`beginWork` har Fiber uchun chaqiriladi. Haqiqiy signature (manba: `react-reconciler/src/ReactFiberBeginWork.js`):

```typescript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null;
```

`current` — eski tree'dagi Fiber (mount uchun `null`). `workInProgress` — yangi tree'dagi Fiber (rebuild yoki reuse). Qaytariladi: **birinchi child Fiber** (work loop pastga tushadi), yoki **`null`** (child yo'q → completeWork → sibling/parent).

```typescript
// React internal pipeline (soddalashtirilgan)
function beginWork(current, workInProgress, renderLanes) {
  // 1. Komponent funksiyasini chaqirish (FunctionComponent uchun)
  const newChildren = renderWithHooks(...);
  
  // 2. Yangi children'ni eski Fiber tree bilan reconcile qilish
  reconcileChildren(current, workInProgress, newChildren, renderLanes);
  
  // 3. Birinchi child'ni qaytarish (work loop pastga tushadi)
  return workInProgress.child;
}
```

`reconcileChildren` — **Reconciliation algoritmining markazi**. U `mountChildFibers` (mount paytida) yoki `reconcileChildFibers` (update paytida) ni chaqiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Reconciliation pipeline batafsil:**

```
beginWork(fiber) chaqirildi
  ↓
Komponent funksiyasi (yoki host element handler) chaqiriladi
  - Function: renderWithHooks() — hooks bajariladi, JSX qaytariladi
  - Host: pendingProps DOM property'lariga prepare qilinadi
  ↓
reconcileChildren(current, workInProgress, newChildren, lanes)
  ↓
  Agar current === null → mount path
    mountChildFibers() — yangi Fiber'lar yaratiladi (alternate yo'q)
  
  Agar current !== null → update path
    reconcileChildFibers() — yangi children eski child'lar bilan diff qilinadi
      ↓
      newChildren turi ko'riladi:
      - JSX element (object) → reconcileSingleElement
      - Array → reconcileChildrenArray
      - String/Number → reconcileSingleTextNode
      - null → o'chirish (eski child'larni delete)
      - Iterator → reconcileChildrenIterator
```

**`reconcileChildFibers` qaytaradigan natija:**

Reconciler yangi child Fiber'larning **birinchi**'sini qaytaradi (yoki `null`). Sibling'lar `child.sibling` orqali bog'langan. Bu — `workInProgress.child` ga assign qilinadi.

```typescript
// React internal (soddalashtirilgan)
function reconcileChildren(current, workInProgress, nextChildren, renderLanes) {
  if (current === null) {
    workInProgress.child = mountChildFibers(workInProgress, null, nextChildren, renderLanes);
  } else {
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,  // eski child Fiber
      nextChildren,
      renderLanes
    );
  }
}
```

**Mount vs Update path farqi:**

`mountChildFibers` — Fiber'lar uchun **Placement flag o'rnatmaydi** (root mount paytida hech kim ham ko'rmaydi DOM'da, faqat root commit qilinganda paydo bo'ladi).

`reconcileChildFibers` — yangi Fiber'lar uchun **Placement flag o'rnatadi** (Commit'da DOM'ga insert qilinadi).

Bu farq ko'p miqdordagi DOM mutations'ni minimallashtiradi: initial mount paytida har element'ga alohida `Placement` flag qo'yilmaydi — root mount qilinganda butun tree birga insert qilinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Reconciler chaqirilishi misol bilan:

```tsx
function App() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <h1>Count: {count}</h1>
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}

// Initial render:
// 1. App() chaqiriladi → JSX qaytariladi
// 2. reconcileChildren(null, AppFiber, JSX) → mount path
// 3. Yangi Fiber'lar yaratiladi: div, h1, "Count: 0", button, "+"
// 4. AppFiber.child = divFiber

// + bosildi (count=1):
// 1. App() qayta chaqiriladi → yangi JSX qaytariladi (count=1 bilan)
// 2. reconcileChildren(currentApp, workInProgressApp, newJSX) → update path
// 3. reconcileChildFibers — eski va yangi children solishtiriladi:
//    - <div> → mos type → Fiber reuse; div props o'zgarmagan → Update flag YO'Q
//      - <h1> → mos type → Fiber reuse; h1 attribute'lari o'zgarmagan → Update flag YO'Q
//        - Text "Count: 1" → eski text "Count: 0" bilan farqli → text Fiber'ga Update flag
//      - <button> → mos type, props bir xil → no flag
//        - Text "+" → bir xil → no flag
// 4. Faqat text node ("Count: ...") uchun Update flag → minimal DOM mutation
//    (host element'ga Update flag completeWork'da prop diff bo'lsagina qo'yiladi)
```

DevTools Profiler "Why did this render?" faqat **komponent**'lar darajasida ishlaydi (host element yoki text node emas). Yuqoridagi `App` uchun:

```
Profiler "Why did this render?":
- App: Hook 1 changed (useState: 0 → 1)
```

`div`, `h1`, `button` — host element'lar, ular Profiler'da alohida "why" sababiga ega emas; ular `App` render qilingani uchun reconcile qilinadi. Real flag'lar (Update / Placement) Fiber tree'da turadi, Profiler'da emas.

Reconciler DOM'ga tegmaydi — faqat flag'lar:

```tsx
function Demo() {
  const [items, setItems] = useState([1, 2, 3]);
  
  return (
    <ul>
      {items.map(i => <li key={i}>{i}</li>)}
    </ul>
  );
}

// setItems([1, 2, 3, 4]) chaqirildi:
// Reconciler natijasi (Fiber tree'da flag'lar):
// li(key=1): no flag (mos)
// li(key=2): no flag
// li(key=3): no flag
// li(key=4): Placement flag (yangi)
// 
// DOM hali yangilanmagan — Commit Phase boshlanmaguncha DOM eski holatda
// Commit Phase, Mutation:
// - Placement flag bilan li(4) uchun DOM'ga insertBefore yoki appendChild
// - DOM hozir yangi
```

</details>

---

## Naive O(n³) vs React O(n) Heuristics

### Nazariya

**Akademik tree diff muammosi:**

Ikki tree o'rtasidagi minimal "edit distance" (qancha insert/delete/update operatsiya kerak ekanligi) hisoblash — algoritmik nuqtai nazardan murakkab masala. Eng yaxshi ma'lum algoritmlar (Zhang-Shasha, 1989) **O(n³)** murakkablikda ishlaydi.

**Amaliy ta'sir (asimptotik):**

| Tree o'lcham | Operatsiya soni (O(n³)) |
|--------------|--------------------------|
| 100 node | 10⁶ (1 million) |
| 1,000 node | 10⁹ (1 milliard) |
| 10,000 node | 10¹² (1 trillion) |

10¹² ta operatsiya — har qanday CPU uchun UI rendering'da amaliy emas. Aniq wall-clock vaqt CPU IPS, payload va engine optimization'lariga bog'liq, lekin polynomial o'sish bilan O(n³) algorithm interactive UI uchun yaroqsiz.

**React'ning ikki heuristic'i** bu muammoni hal qiladi:

### Heuristic 1: Different types = rebuild

Agar element type **o'zgargan** bo'lsa (masalan `<div>` → `<p>`), Reconciler eski subtree'ni butunlay o'chiradi va yangidan quradi. Eski va yangi child'larni solishtirishga **urinmaydi**.

```tsx
// Eski tree:
<div>
  <Counter />
  <UserList />
</div>

// Yangi tree:
<p>           {/* type o'zgardi: div → p */}
  <Counter />
  <UserList />
</p>

// Reconciler qaror:
// - <p> yangi type → eski <div> ni butunlay unmount qilish
// - <div> ichidagi Counter, UserList — barcha state yo'qoladi
// - Yangi <p> mount qilinadi, Counter va UserList — yangi instance
//
// Counter ichidagi useState count — yangidan 0 dan boshlaydi
// UserList ichidagi fetched data — yo'qolgan, qayta fetch qilinadi
```

Bu — qattiq qaror. Ammo amaliyotda **type kamdan-kam o'zgaradi** (`<div>` `<p>`'ga aylantirilmaydi odatda). Type o'zgargan holatlar — odatda butunlay boshqa UI section.

### Heuristic 2: Keys = stable identity

Lists'da React `key` prop'ini ishlatadi:

```tsx
{users.map(user => <UserCard key={user.id} user={user} />)}
```

`key` — har item'ning **identity belgi**'si. Reconciler eski va yangi list'larni `key` orqali matching qiladi:

- Bir xil `key` → bir xil item (update)
- Yangi `key` → yangi item (mount)
- Eski `key` yangida yo'q → o'chirilgan (unmount)
- Bir xil `key`, lekin pozitsiya farqli → ko'chirildi (move)

**Keysiz** lists uchun React **index-based** matching qiladi (pozitsiya 0 ga pozitsiya 0, va h.k.). Bu reorder, insert middle, delete middle holatlarida samarasiz.

### Bu heuristic'larning O(n)'ga olib kelishi

Bu ikki heuristic bilan Reconciler **har Fiber'ni faqat bir marta** ko'radi:
- Type tekshirish: O(1)
- Key matching: O(n) (har item'ni bir marta ko'rib chiqish, key map ishlatish)
- Recursive descent: barcha child'lar bo'ylab yurish

Jami: **O(n)** — har Fiber bir marta tekshiriladi.

**Trade-off:**

React optimal diff'ni topmaydi — agar `<div>` → `<section>` o'zgarish bo'lsa, ikkalasi ham block-level container bo'lib, child'larni saqlab qolish nazariy jihatdan mumkin edi. Lekin algoritm bu holatni qidirmaydi (ko'p hisob talab qilardi). React **amaliy bo'lgan, optimal bo'lmagan** algoritm tanlagan — natijada tezroq ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Akademik diff vs amaliy diff:**

Tree diff problem'i (Zhang-Shasha, 1989) — bu ikki tree o'rtasidagi minimal **edit script**'ni topish:
- Insert node
- Delete node
- Update node label

Algoritm dynamic programming bilan O(n³) (yoki ba'zi optimizationlar bilan O(n² log n)).

React **optimal diff'ni topmaydi**. Misol:

```tsx
// Eski:
<div>
  <Header />
  <List />
</div>

// Yangi:
<section>
  <Header />
  <List />
</section>
```

Optimal yechim: outer wrapper'ni o'zgartirish, child'larni saqlash. Bu **2 ta DOM operatsiya**.

React'ning yechimi: butun subtree'ni unmount, qayta mount. **N ta DOM operatsiya** (N = barcha child'lar va nestedlari soni).

**Nima uchun React bunday qilgan:**

1. **Type o'zgarishi kamdan-kam** — practice'da `<div>` → `<section>` o'zgarishi yo'q (ikkalasi semantik farqli)
2. **Algoritm sodda va deterministik** — debugging va memory predict osonlashadi
3. **Real-world performance optimal** — academic yondashuv O(n³) bilan butun app'ni sekinlashtirardi
4. **Refactor encouragement** — agar type o'zgarishi kerak bo'lsa, developer ongli ravishda komponent darajasida qiladi (komponent rebuild, child'lar ham yangilanadi — kutilgan xulq-atvor)

**Pathological case'lar:**

React heuristic'lari uchun "yomon" misollar:

```tsx
// 1. Type oscillation
{showAlt ? <div>{children}</div> : <section>{children}</section>}
// Toggling — har safar children'ning state'i yo'qoladi

// 2. Wrapping/unwrapping
{wrapInForm ? <form><Input /></form> : <Input />}
// Toggle bo'lganda Input state yo'qoladi

// 3. Index as key with reorder
{shuffledItems.map((item, i) => <Card key={i} item={item} />)}
// Reorder bo'lganda key bir xil qoladi, lekin item boshqa — props bilan force update
```

Bu pathological case'lar real applicationda kamdan-kam — odatda ongli dizayn qaroriga aylanadi (state preserve qilish kerak yoki yo'qmi degan masala).

**O(n) prooflari (mental):**

`reconcileChildrenArray` har element uchun:
- Eski Fiber'larni map'da saqlash: O(n) (bir marta)
- Yangi har element uchun map'da topish: O(1) average
- Jami: O(n)

Recursive descent: tree bo'ylab yurish — har Fiber bir marta — O(N) (N — total Fiber soni).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Type o'zgarishi state yo'qolishi misol:

```tsx
function Toggle() {
  const [isWrapped, setIsWrapped] = useState(false);
  
  return (
    <div>
      <button onClick={() => setIsWrapped(w => !w)}>Toggle wrapper</button>
      {isWrapped ? (
        <section>
          <Counter />
        </section>
      ) : (
        <div>
          <Counter />
        </div>
      )}
    </div>
  );
}

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// Counter bosildi: 0 → 5
// "Toggle wrapper" bosildi: <div> → <section>
// Reconciler: type o'zgardi → unmount old subtree, mount new
// Counter unmount qilindi va yangi mount qilindi
// Counter state: 5 → 0 (yangi instance)
```

Same type, faqat children o'zgargan — state saqlanadi:

```tsx
function ConditionalContent() {
  const [showHeader, setShowHeader] = useState(true);
  
  return (
    <div>
      {showHeader && <h1>Header</h1>}
      <Counter />  {/* div ichida — wrapping element same type */}
      <button onClick={() => setShowHeader(s => !s)}>Toggle header</button>
    </div>
  );
}

// Counter bosildi: 0 → 5
// Toggle bosildi: header yo'qoldi
// Reconciler:
// - <div> same type → no rebuild
// - <h1> yo'qolgan → ChildDeletion flag
// - Counter — pozitsiya o'zgargan? Hayir, h1 indexsi 0 edi, Counter 1 edi
//   Endi Counter index 0 — keysiz, lekin reconciler index-based match qiladi
//   Counter type same → update path → state saqlanadi
// Counter state: 5 (saqlandi)
```

Kichik nuance — agar h1 va Counter o'rinlarini almashtirsangiz:

```tsx
function ConditionalReorder() {
  const [headerFirst, setHeaderFirst] = useState(true);
  
  return (
    <div>
      {headerFirst ? (
        <>
          <h1>Header</h1>
          <Counter />
        </>
      ) : (
        <>
          <Counter />
          <h1>Header</h1>
        </>
      )}
    </div>
  );
}

// Reconciler keysiz fragment ichidagi siblings:
// Eski: [h1, Counter]
// Yangi: [Counter, h1]
// Index-based match:
// position 0: h1 (eski) vs Counter (yangi) — DIFFERENT TYPE
//   → eski h1 unmount, yangi Counter mount
// position 1: Counter (eski) vs h1 (yangi) — DIFFERENT TYPE
//   → eski Counter unmount, yangi h1 mount
// Counter state YO'QOLADI

// To'g'ri: keys ishlatish
{headerFirst ? (
  <>
    <h1 key="h">Header</h1>
    <Counter key="c" />
  </>
) : (
  <>
    <Counter key="c" />
    <h1 key="h">Header</h1>
  </>
)}
// Endi key="c" Counter ikki render holatida mos topadi → state saqlanadi
```

</details>

---

## Type Comparison

### Nazariya

Reconciler eski Fiber va yangi React Element'ni solishtirayotganda birinchi navbatda **type**'ni tekshiradi.

**Type bir xil bo'lsa:**

```tsx
// Eski Fiber: <div className="old">
// Yangi Element: <div className="new">

// type bir xil ('div')
// → Fiber qayta ishlatiladi (alternate orqali)
// → pendingProps yangilanadi: { className: 'new' }
// → Update flag o'rnatiladi
// → child'lar reconcile qilinadi (recursive)
// → Commit Phase'da: element.className = 'new'
```

**Type farq qilsa:**

```tsx
// Eski Fiber: <div>
// Yangi Element: <p>

// type farqli ('div' vs 'p')
// → Eski Fiber'ga ChildDeletion flag o'rnatiladi (parent'da)
// → Yangi Fiber yaratiladi <p> uchun, Placement flag bilan
// → Eski child'lar barchasi unmount qilinadi (state, effect cleanup)
// → Yangi child'lar mount qilinadi
// → Commit Phase'da: removeChild(div), createElement(p), appendChild
```

**Komponent type holatlari:**

| Eski type | Yangi type | Qaror |
|-----------|------------|-------|
| `Counter` (function) | `Counter` (same function) | Update — reuse Fiber, state saqlanadi |
| `Counter` | `OtherCounter` (boshqa function) | Unmount + mount — state yo'qoladi |
| `<div>` | `<div>` | Update — reuse Fiber |
| `<div>` | `<p>` | Unmount + mount |
| `<Counter />` | `<MemoCounter />` (memo wrapped) | Unmount + mount (memo wrapper boshqa type) |
| `<Counter />` | `<ForwardCounter />` (forwardRef wrapped) | Unmount + mount (R19'da `forwardRef` function komponent uchun keraksiz — ref oddiy prop, lekin wrap qilingan elementni o'rab olish hali ham type'ni o'zgartiradi) |

**Type reference tenglik:**

React `===` (strict equality) ishlatadi: `child.elementType === elementType` taqqoslashi `react-reconciler/src/ReactChildFiber.js` ichida. Type qiymatlari function yoki object (memo/forwardRef wrappers) bo'lgani uchun bu **reference identity** taqqoslash bilan teng (`Object.is` ham bir xil natija beradi function/object'lar uchun, faqat `NaN` va `±0` uchun farq qiladi — bunday qiymatlar Fiber type'da uchramaydi). Ya'ni:
- **Bir xil function reference** — reuse
- **Yangi function har render'da** — unmount + remount (state yo'qoladi!)

```tsx
function Parent() {
  // ❌ Inline component — har render'da yangi function reference
  const Inline = () => <div>Inline</div>;
  
  return <Inline />;
}

// Har Parent render'da yangi `Inline` function yaratiladi
// Reconciler type'ni tekshiradi: prevType !== nextType (yangi function)
// → Inline component unmount qilinadi va qayta mount qilinadi
// → Inline ichidagi har qanday state, effect — yo'qoladi
```

```tsx
// ✅ Outside-defined component — bir xil reference
const Outer = () => <div>Outer</div>;

function Parent() {
  return <Outer />;
}
// Outer function reference doim bir xil → reuse → state saqlanadi
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Type comparison reconciler ichida:**

```typescript
// React internal (soddalashtirilgan)
function reconcileSingleElement(returnFiber, currentFirstChild, element, lanes) {
  const key = element.key;
  let child = currentFirstChild;
  
  while (child !== null) {
    // 1. Key tekshirish (lists uchun, single element'da har doim null)
    if (child.key === key) {
      const elementType = element.type;
      
      // 2. Type tekshirish
      if (child.elementType === elementType) {
        // SAME TYPE — reuse Fiber
        deleteRemainingChildren(returnFiber, child.sibling);
        const existing = useFiber(child, element.props);
        existing.return = returnFiber;
        return existing;
      } else {
        // DIFFERENT TYPE — delete all and start fresh
        deleteRemainingChildren(returnFiber, child);
        break;
      }
    } else {
      // Key boshqa — delete this and try next
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }
  
  // Yangi Fiber yaratish
  const created = createFiberFromElement(element, returnFiber.mode, lanes);
  created.return = returnFiber;
  return created;
}
```

**`elementType` vs `type`:**

Fiber'ning ikkita shu nomdagi maydoni bor:

- **`elementType`** — JSX'dagi original `type` (memo wrapper, forwardRef wrapper, va h.k. bilan)
- **`type`** — resolved type (memo'ning ichki komponenti, forwardRef'ning render funksiyasi)

```typescript
// Misol: React.memo(MyComp)
const MemoMyComp = memo(MyComp);

<MemoMyComp />

// Fiber:
// elementType = MemoMyComp (memo obyekt: { $$typeof, type: MyComp, compare: null })
// type = MyComp (resolved — ichki function)
```

Reconciler `elementType` bilan solishtiradi (chunki bu — JSX'dagi original). Bu tushunarli xulq-atvor: `MemoMyComp` `MyComp` (memosiz)'ga almashtirilsa — type farqli, unmount + mount.

**Class komponent type:**

```typescript
class MyClass extends React.Component { /* ... */ }

<MyClass />

// Fiber:
// elementType = MyClass (class)
// type = MyClass
// stateNode = new MyClass() (instance)
```

Class instance Fiber'da saqlanadi. Type bir xil → instance reuse → state saqlanadi.

**Lazy va Suspense:**

```typescript
const Lazy = React.lazy(() => import('./Lazy'));

<Lazy />

// Fiber tag = LazyComponent
// elementType = Lazy obyekt: { $$typeof, _payload: ... }
//
// Komponent yuklangach:
// elementType yangilanadi → resolved Component
// Lekin tag o'zgarmaydi (LazyComponent qoladi)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Inline component anti-pattern:

```tsx
// ❌ ANTI-PATTERN — har Parent render'da yangi MyChild function
function Parent() {
  const [count, setCount] = useState(0);
  
  function MyChild() {
    const [localState, setLocalState] = useState(0);
    return (
      <button onClick={() => setLocalState(s => s + 1)}>
        Local: {localState}
      </button>
    );
  }
  
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Parent: {count}</button>
      <MyChild />
    </>
  );
}

// Parent button bosildi (count: 0 → 1):
// Parent re-render → yangi MyChild function yaratildi
// Reconciler: prevType (oldingi MyChild) !== nextType (yangi MyChild)
// → MyChild unmount qilinadi
// → MyChild qayta mount qilinadi
// → localState 0'dan boshlanadi (yo'qoldi!)

// To'g'ri: MyChild ni Parent dan tashqarida e'lon qilish
function MyChild() {
  const [localState, setLocalState] = useState(0);
  return (
    <button onClick={() => setLocalState(s => s + 1)}>
      Local: {localState}
    </button>
  );
}

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Parent: {count}</button>
      <MyChild />
    </>
  );
}
// Endi MyChild reference doim bir xil → state saqlanadi
```

Conditional rendering — ikki branch'da farqli komponent:

```tsx
// ❌ Type oscillation — switch holatida state yo'qoladi
function Profile({ user }: { user: User }) {
  if (user.isPremium) {
    return <PremiumProfile user={user} />;
  } else {
    return <StandardProfile user={user} />;
  }
}

// user.isPremium o'zgarganda → komponent type o'zgaradi
// → har subscription/state yo'qoladi
// → re-fetch, re-subscribe, va h.k.
```

```tsx
// ✅ Conditional content ichida — state saqlanadi
function Profile({ user }: { user: User }) {
  return (
    <div>
      {user.isPremium && <PremiumBadge />}
      <UserDetails user={user} />  {/* type doim bir xil */}
      <SubscriptionInfo isPremium={user.isPremium} />
    </div>
  );
}
```

Memo wrapper farqi:

```tsx
const MemoButton = React.memo(function Button({ onClick, label }: Props) {
  return <button onClick={onClick}>{label}</button>;
});

// Use site:
const [memoEnabled, setMemoEnabled] = useState(true);

return memoEnabled
  ? <MemoButton onClick={handleClick} label="Save" />
  : <Button onClick={handleClick} label="Save" />;

// memoEnabled toggle qilinganda:
// elementType: MemoButton → Button (boshqa obyektlar!)
// → unmount + mount
// → Button componentidagi state, effect — yo'qoladi
```

</details>

---

## Sibling Matching — Keyless

### Nazariya

Lists yoki bir nechta sibling'lar uchun (key bo'lmaganda) Reconciler **index-based matching** ishlatadi: pozitsiya 0 ni pozitsiya 0 bilan, pozitsiya 1 ni pozitsiya 1 bilan, va h.k.

**Algoritm (mental):**

1. Eski va yangi children'larni parallel ravishda yurish
2. Har pozitsiyada:
   - Type bir xil bo'lsa → update Fiber
   - Type boshqa bo'lsa → unmount eski, mount yangi
3. Yangi'da ko'proq element bo'lsa — qo'shimchalar mount qilinadi
4. Eski'da ko'proq element bo'lsa — qo'shimchalar unmount qilinadi

**Misol — append (yaxshi case):**

```tsx
// Eski:
<ul>
  <li>Item 1</li>
  <li>Item 2</li>
</ul>

// Yangi:
<ul>
  <li>Item 1</li>
  <li>Item 2</li>
  <li>Item 3</li>  {/* yangi qo'shildi */}
</ul>

// Reconciler:
// Position 0: <li>Item 1</li> vs <li>Item 1</li> → same type, props bir xil → no flag
// Position 1: <li>Item 2</li> vs <li>Item 2</li> → same type, props bir xil → no flag
// Position 2: yangi <li>Item 3</li> → Placement flag (yangi mount)
//
// Natija: 1 ta DOM mutation (appendChild)
```

**Misol — prepend (yomon case):**

```tsx
// Eski:
<ul>
  <li>Item 1</li>
  <li>Item 2</li>
</ul>

// Yangi:
<ul>
  <li>Item 0</li>  {/* boshiga qo'shildi */}
  <li>Item 1</li>
  <li>Item 2</li>
</ul>

// Reconciler (keysiz):
// Position 0: <li>Item 1</li> (eski) vs <li>Item 0</li> (yangi) → same type → Update (text "1" → "0")
// Position 1: <li>Item 2</li> vs <li>Item 1</li> → Update (text "2" → "1")
// Position 2: yangi <li>Item 2</li> → Placement
//
// Natija: 2 ta DOM update + 1 ta append
// Eski Item 1 va Item 2 endi boshqa textni ko'rsatadi (state yo'qoladi agar bo'lsa)
```

**Misol — middle remove (yomon case):**

```tsx
// Eski: [A, B, C, D]
// Yangi: [A, C, D]  (B o'chirildi)

// Position 0: A vs A → same → no flag
// Position 1: B vs C → same type, props farqli → Update (B → C)
// Position 2: C vs D → Update (C → D)
// Position 3: D vs (yo'q) → ChildDeletion
//
// Natija: 2 update + 1 delete
// B, C, D komponentlari "ko'chirilmadi", balki props yangilandi (state agar bo'lsa, noto'g'ri item'da qoladi)
```

### Index-based matching ning muammolari

1. **State chalkashligi** — komponent state oldingi pozitsiyadagi item bilan bog'lab qoladi
2. **DOM identity yo'qolishi** — input focus, scroll position, animation o'zgarishi mumkin
3. **Performance** — keraksiz update'lar (text, attributes)

Bu muammolarni hal qilish — **`key` prop** ishlatish (keyingi bo'limda).

<details>
<summary><strong>Under the Hood</strong></summary>

**Keyless matching algoritmi:**

```typescript
// React internal (soddalashtirilgan)
function reconcileChildrenArray(returnFiber, currentFirstChild, newChildren, lanes) {
  let resultingFirstChild = null;
  let previousNewFiber = null;
  
  let oldFiber = currentFirstChild;
  let newIdx = 0;
  let lastPlacedIndex = 0;
  
  // Faza 1: Parallel iteration
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    if (oldFiber.index > newIdx) {
      // Edge case
      break;
    }
    
    const newFiber = updateSlot(returnFiber, oldFiber, newChildren[newIdx], lanes);
    
    if (newFiber === null) {
      // Type farqli — to'xtash, faza 2 ga o'tish
      break;
    }
    
    if (newFiber.alternate === null) {
      // Yangi Fiber yaratildi (eski o'chirilmoqda)
      deleteChild(returnFiber, oldFiber);
    }
    
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    
    if (previousNewFiber === null) {
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
    
    oldFiber = oldFiber.sibling;
  }
  
  // Faza 2: Yangi children'da qoldiq
  if (newIdx === newChildren.length) {
    // Eski'larda qoldiq — delete
    deleteRemainingChildren(returnFiber, oldFiber);
    return resultingFirstChild;
  }
  
  // Faza 3: Eski'larda qoldiq, lekin yangi'da ham — keyed matching kerak
  if (oldFiber === null) {
    // Eski'da qoldiq yo'q — qolgan yangi'lar mount qilinadi
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
      // ... append to result
    }
    return resultingFirstChild;
  }
  
  // Faza 4: Aralash — keyed matching ishlatish (keyingi bo'limda)
  // ...
}
```

**`updateSlot` mexanikasi:**

```typescript
function updateSlot(returnFiber, oldFiber, newChild, lanes) {
  const key = oldFiber !== null ? oldFiber.key : null;
  
  if (typeof newChild === 'object' && newChild !== null) {
    // Element
    if (newChild.key !== key) {
      // Key farqli — slot mos emas
      return null;
    }
    return updateElement(returnFiber, oldFiber, newChild, lanes);
  }
  
  // Text/string/number
  if (key !== null) {
    return null;  // text key bo'lmaydi, slot mos emas
  }
  return updateTextNode(returnFiber, oldFiber, '' + newChild, lanes);
}
```

`updateSlot` slot match bo'lganda Fiber qaytaradi, mos kelmasa null. Mos kelmasa, `reconcileChildrenArray` keyed matching faza'siga o'tadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Keyless matching effekti — input focus yo'qolishi:

```tsx
function ItemList() {
  const [items, setItems] = useState([
    { id: 'a', text: 'Item A' },
    { id: 'b', text: 'Item B' },
  ]);

  function prepend() {
    setItems([{ id: 'new', text: 'Item NEW' }, ...items]);
  }

  return (
    <>
      <button onClick={prepend}>Prepend</button>
      <ul>
        {items.map(item => (
          <li>  {/* ❌ key yo'q */}
            <input defaultValue={item.text} />
          </li>
        ))}
      </ul>
    </>
  );
}

// Initial: [A, B]
// User input "A" ga focus va text yozadi: "A modified"
// 
// "Prepend" bosildi → [NEW, A, B]
// 
// Keyless matching (index bo'yicha):
// Position 0: <li><input> (eski A)  vs  <li><input> (yangi NEW)
//   → same type → Fiber va DOM node reuse (yangi DOM yaratilmaydi)
//   → defaultValue faqat mount paytida qo'llanadi, reuse'da DOM value o'zgarmaydi
//   → DOM input'da hali "A modified" turibdi, focus ham shu node'da
//   → endi bu node logik jihatdan "NEW" item'ga tegishli — moslik buzilgan
// Position 1: <li><input> (eski B)  vs  <li><input> (yangi A) → reuse
// Position 2: yangi <li> → Placement (NEW emas, B uchun qo'shimcha slot)
//
// Natija: yangi DOM mount bo'lmadi (faqat oxirida 1 ta li qo'shildi), lekin
// "A modified" matni endi NEW item ostida ko'rinadi — item↔DOM bog'lanishi siljidi
```

```tsx
// ✅ TO'G'RI — key bilan
function ItemList() {
  const [items, setItems] = useState([...]);

  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>  {/* key bilan */}
          <input defaultValue={item.text} />
        </li>
      ))}
    </ul>
  );
}

// "Prepend" bosildi → [NEW, A, B]
// Keyed matching:
// key="new": yangi → Placement (boshiga insert qilinadi)
// key="a": pozitsiya 0 → 1 → ko'chirildi (lekin Fiber reuse, input identity saqlanadi)
// key="b": pozitsiya 1 → 2 → ko'chirildi (Fiber reuse)
//
// Natija: A va B input'lari pozitsiya o'zgartirsin, lekin DOM node identity saqlanadi
// User'ning yozgan matni va focus pozitsiyasi saqlanadi
```

Mental model — keyless matching uchun **append** OK, **prepend/middle insert** muammoli:

```tsx
// ✅ Keyless OK — append only
function Logger({ messages }: { messages: string[] }) {
  return (
    <ul>
      {messages.map(msg => <li>{msg}</li>)}  {/* keyless OK agar faqat append */}
    </ul>
  );
}

// ⚠️ Keyless yomon — har xil operations
function TodoList({ todos }: { todos: Todo[] }) {
  // Insert middle, delete, reorder bo'lishi mumkin
  return (
    <ul>
      {todos.map(t => <li>{t.text}</li>)}  {/* ❌ key kerak */}
    </ul>
  );
}
```

Eslatma: ESLint `react/jsx-key` rule bu xato'ni topadi (default rule modern setup'larda).

</details>

---

## Sibling Matching — Keyed

### Nazariya

`key` prop bilan lists uchun React **identity-based matching** ishlatadi. Algoritm 2 fazadan iborat:

**Faza 1 — sequential matching (left-to-right):**

Eski va yangi children'larni parallel yurish, har pozitsiyada `key` taqqoslash:
- Yangi children[i].key === eski Fiber'ning key'i → match (update yoki no-op)
- Mos kelmasa → faza 2 ga o'tish

Bu faza **eng tez yo'l** — agar list o'zgarmagan bo'lsa, butun matching O(n) bilan tugaydi.

**Faza 2 — key map matching (qoldiq uchun):**

Faza 1 to'xtagandan keyin:
1. Qolgan eski Fiber'lardan `Map<key, Fiber>` quriladi
2. Qolgan yangi children'lar bo'ylab yurish:
   - Map'dan key bo'yicha eski Fiber qidirish
   - Topilsa → match (move yoki update)
   - Topilmasa → yangi (Placement)
3. Map'dagi qoldiq Fiber'lar → o'chirish (Deletion)

Bu faza ham O(n) — har Fiber map'da O(1)'da topiladi.

### Move detection — `lastPlacedIndex`

Reconciler `lastPlacedIndex` ni ishlatadi — bu eng oxirgi "joyiga qo'yilgan" yangi child'ning eski index'i:

```typescript
function placeChild(newFiber, lastPlacedIndex, newIndex) {
  newFiber.index = newIndex;
  
  const current = newFiber.alternate;
  if (current !== null) {
    // Reuse'd Fiber — eski pozitsiyasi current.index'da
    const oldIndex = current.index;
    
    if (oldIndex < lastPlacedIndex) {
      // Bu Fiber pozitsiyasi orqada — KO'CHIRISH kerak
      newFiber.flags |= Placement;
      return lastPlacedIndex;
    } else {
      // Bu Fiber pozitsiyasi old'inda yoki bir xil — KO'CHIRISH yo'q
      return oldIndex;
    }
  } else {
    // Yangi Fiber — har holda Placement
    newFiber.flags |= Placement;
    return lastPlacedIndex;
  }
}
```

**Mental model:**

```
Eski: [A, B, C, D, E]      indices: A=0, B=1, C=2, D=3, E=4

Yangi: [B, A, C, D, E]
       
Faza 1 boshlanadi:
  newIdx=0: B (yangi) vs A (eski position 0) → key farqli → faza 1 to'xtaydi

Faza 2 (keyed map):
  Map: { A: oldFiber0, B: oldFiber1, C: oldFiber2, D: oldFiber3, E: oldFiber4 }
  
  newIdx=0: B → topildi (oldIndex=1)
    lastPlacedIndex=0; oldIndex=1 >= 0 → no move, lastPlacedIndex = 1
  
  newIdx=1: A → topildi (oldIndex=0)
    lastPlacedIndex=1; oldIndex=0 < 1 → MOVE (Placement flag)
    lastPlacedIndex = 1 (o'zgarmagan)
  
  newIdx=2: C → topildi (oldIndex=2)
    lastPlacedIndex=1; oldIndex=2 >= 1 → no move, lastPlacedIndex = 2
  
  newIdx=3: D → topildi (oldIndex=3)
    lastPlacedIndex=2; oldIndex=3 >= 2 → no move, lastPlacedIndex = 3
  
  newIdx=4: E → topildi (oldIndex=4)
    lastPlacedIndex=3; oldIndex=4 >= 3 → no move, lastPlacedIndex = 4

Natija: Faqat A — moved (1 ta DOM mutation)
```

**Optimization:** Algoritm "minimal moves" qilmaydi (LCS — Longest Common Subsequence — O(n²)). U **greedy** — left-to-right yurib, har element'ning oldingi yoki orqaligini ko'radi. Bu algoritm O(n) lekin ba'zi holatlarda optimal'dan ko'proq move qiladi (acceptable trade-off).

<details>
<summary><strong>Under the Hood</strong></summary>

**`reconcileChildrenArray` to'liq mantiqi:**

```typescript
function reconcileChildrenArray(returnFiber, currentFirstChild, newChildren, lanes) {
  let resultingFirstChild = null;
  let previousNewFiber = null;
  let oldFiber = currentFirstChild;
  let lastPlacedIndex = 0;
  let newIdx = 0;
  let nextOldFiber = null;
  
  // ============================================
  // FAZA 1: Sequential matching (fast path)
  // ============================================
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    if (oldFiber.index > newIdx) {
      nextOldFiber = oldFiber;
      oldFiber = null;
    } else {
      nextOldFiber = oldFiber.sibling;
    }
    
    const newFiber = updateSlot(returnFiber, oldFiber, newChildren[newIdx], lanes);
    if (newFiber === null) {
      // Slot mos emas (key farqli) → faza 1 to'xtaydi
      if (oldFiber === null) oldFiber = nextOldFiber;
      break;
    }
    
    if (newFiber.alternate === null) {
      deleteChild(returnFiber, oldFiber);
    }
    
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    
    if (previousNewFiber === null) resultingFirstChild = newFiber;
    else previousNewFiber.sibling = newFiber;
    previousNewFiber = newFiber;
    oldFiber = nextOldFiber;
  }
  
  // ============================================
  // FAZA 2: Yangi tugagan
  // ============================================
  if (newIdx === newChildren.length) {
    deleteRemainingChildren(returnFiber, oldFiber);
    return resultingFirstChild;
  }
  
  // ============================================
  // FAZA 3: Eski tugagan, yangi'lar mount
  // ============================================
  if (oldFiber === null) {
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
      if (newFiber === null) continue;
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      if (previousNewFiber === null) resultingFirstChild = newFiber;
      else previousNewFiber.sibling = newFiber;
      previousNewFiber = newFiber;
    }
    return resultingFirstChild;
  }
  
  // ============================================
  // FAZA 4: Aralash — keyed map matching
  // ============================================
  const existingChildren = mapRemainingChildren(returnFiber, oldFiber);
  
  for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = updateFromMap(
      existingChildren,
      returnFiber,
      newIdx,
      newChildren[newIdx],
      lanes
    );
    if (newFiber !== null) {
      if (newFiber.alternate !== null) {
        // Reuse'd — map'dan o'chirish
        existingChildren.delete(newFiber.key === null ? newIdx : newFiber.key);
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      if (previousNewFiber === null) resultingFirstChild = newFiber;
      else previousNewFiber.sibling = newFiber;
      previousNewFiber = newFiber;
    }
  }
  
  // FAZA 5: Map'dagi qoldiq Fiber'lar — o'chirish
  existingChildren.forEach(child => deleteChild(returnFiber, child));
  
  return resultingFirstChild;
}
```

**`mapRemainingChildren`:**

```typescript
function mapRemainingChildren(returnFiber, currentFirstChild) {
  const existingChildren = new Map();
  let existingChild = currentFirstChild;
  
  while (existingChild !== null) {
    if (existingChild.key !== null) {
      existingChildren.set(existingChild.key, existingChild);
    } else {
      // Keyless — index'ni key sifatida ishlatish
      existingChildren.set(existingChild.index, existingChild);
    }
    existingChild = existingChild.sibling;
  }
  
  return existingChildren;
}
```

**Why greedy va not LCS:**

Optimal "minimum number of moves" algoritmi — Longest Common Subsequence (LCS) — O(n²). React'ning greedy algoritmi O(n) — har element bir marta ko'riladi.

Trade-off:
- LCS optimal — har item uchun mutloq minimal mutation
- Greedy — ko'pchilik holatlarda yaxshi, ba'zilarida qo'shimcha move qiladi

```
Misol — LCS vs Greedy:

Eski: [A, B, C, D]
Yangi: [D, A, B, C]

LCS optimal: faqat D ni boshiga ko'chirish (1 move)
Greedy:
  D — yangi pozitsiya 0
    lastPlacedIndex = 0 → D oldIndex (3) >= 0 → no move, lastPlacedIndex = 3
  A — yangi pozitsiya 1
    A oldIndex (0) < 3 → MOVE
  B — yangi pozitsiya 2
    B oldIndex (1) < 3 → MOVE
  C — yangi pozitsiya 3
    C oldIndex (2) < 3 → MOVE

Greedy: 3 ta MOVE (suboptimal)
LCS: 1 ta MOVE (optimal, lekin algorithm O(n²))
```

React jamoasi greedy'ni tanlagan: O(n) murakkablik > optimal moves soni.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Keyed matching ko'rgazma:

```tsx
function ReorderableList() {
  const [items, setItems] = useState([
    { id: 1, name: 'Apple' },
    { id: 2, name: 'Banana' },
    { id: 3, name: 'Cherry' },
  ]);

  function reverse() {
    setItems([...items].reverse());
  }

  function shuffle() {
    // Fisher-Yates uniform shuffle (`[...items].sort(() => Math.random() - 0.5)`
    // statistik bir tekis emas — namuna uchun ham yaroqsiz)
    const shuffled = [...items];
    for (let i = shuffled.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
    }
    setItems(shuffled);
  }

  return (
    <>
      <button onClick={reverse}>Reverse</button>
      <button onClick={shuffle}>Shuffle</button>
      <ul>
        {items.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </>
  );
}

// Initial: [Apple(1), Banana(2), Cherry(3)]
// "Reverse" bosildi: [Cherry(3), Banana(2), Apple(1)]
//
// Keyed matching:
// Faza 1: newIdx=0 → key=3 (yangi) vs key=1 (eski) → mos emas, faza 1 to'xtaydi
// Faza 4: existingChildren = { 1: AppleFiber, 2: BananaFiber, 3: CherryFiber }
//   newIdx=0: key=3 → CherryFiber topildi (oldIndex=2)
//     lastPlacedIndex=0; oldIndex=2 >= 0 → no move, lastPlacedIndex=2
//   newIdx=1: key=2 → BananaFiber topildi (oldIndex=1)
//     lastPlacedIndex=2; oldIndex=1 < 2 → MOVE
//   newIdx=2: key=1 → AppleFiber topildi (oldIndex=0)
//     lastPlacedIndex=2; oldIndex=0 < 2 → MOVE
//
// Natija: Banana va Apple — moved. Cherry — yo'q.
// 2 ta DOM mutation (insertBefore)
```

Insert middle:

```tsx
function InsertMiddle() {
  const [items, setItems] = useState([
    { id: 'a', text: 'A' },
    { id: 'c', text: 'C' },
  ]);

  function insertB() {
    setItems([
      { id: 'a', text: 'A' },
      { id: 'b', text: 'B' },  // yangi
      { id: 'c', text: 'C' },
    ]);
  }

  return (
    <>
      <button onClick={insertB}>Insert B</button>
      <ul>
        {items.map(i => <li key={i.id}>{i.text}</li>)}
      </ul>
    </>
  );
}

// Initial: [A(a), C(c)]
// "Insert B" bosildi: [A(a), B(b), C(c)]
//
// Keyed matching:
// Faza 1:
//   newIdx=0: key=a (yangi) vs key=a (eski) → match
//     A reuse, no move (lastPlacedIndex=0)
//   newIdx=1: key=b (yangi) vs key=c (eski) → mos emas, faza 1 to'xtaydi
// Faza 4: existingChildren = { c: CFiber }
//   newIdx=1: key=b → topilmadi → yangi mount, Placement flag
//   newIdx=2: key=c → topildi (oldIndex=1)
//     lastPlacedIndex=0; oldIndex=1 >= 0 → no move
//
// Natija: B mount qilinadi (insertBefore C). A va C — daxlsiz.
// 1 ta DOM mutation (insertBefore B)
```

Index as key — anti-pattern misol:

```tsx
function IndexKeyAnti() {
  const [items, setItems] = useState(['A', 'B', 'C']);

  function prepend() {
    setItems(['Z', ...items]);
  }

  return (
    <>
      <button onClick={prepend}>Prepend Z</button>
      <ul>
        {items.map((item, i) => (
          <li key={i}>  {/* ❌ index as key */}
            <input defaultValue={item} />
          </li>
        ))}
      </ul>
    </>
  );
}

// Initial: [A, B, C], indices: 0, 1, 2
// User input position 0 ga "A modified" yozadi
//
// "Prepend Z" → [Z, A, B, C], indices: 0, 1, 2, 3
//
// Keyed matching (lekin key=index!):
// key=0: A vs Z → match (key bir xil!) → Update (defaultValue ignored, lekin Fiber reuse)
// key=1: B vs A → match → Update
// key=2: C vs B → match → Update
// key=3: yangi → Placement (yangi <li>)
//
// Natija: 3 ta Update + 1 ta Placement
// User'ning "A modified" matni endi "Z" pozitsiyasida (chunki input element shu Fiber bilan bog'langan)
// Bu — index-based matching effekti, lekin React buni "to'g'ri matching" deb hisoblaydi
```

Misol uchun bu — pozitsiya-based bug, va u faqat **stable, unique key** bilan hal qilinadi (item.id, UUID, va h.k.).

</details>

---

## Key-based Identity

### Nazariya

`key` prop — list item'ning **identity belgi**si. React `key` orqali eski va yangi item'larni juftlashtiradi. Bu — Reconciliation algoritmining asosiy "informatsiya manbai".

**Key qoidalari:**

1. **Unique** — bir parent ichidagi sibling'lar orasida `key` unique bo'lishi shart
2. **Stable** — bir xil item har render'da bir xil `key`'ga ega bo'lishi kerak
3. **Predictable** — `key` random yoki Math.random()-based bo'lmasligi kerak
4. **Scope** — `key` faqat parent ichidagi sibling'lar orasida unique. Boshqa parent'larda bir xil key ishlatish OK.

**Yaxshi key sources:**

- ✅ Database ID (`item.id`)
- ✅ Server-generated unique key (`item.uuid`)
- ✅ Stable composite key (`${userId}_${productId}`)
- ✅ Generated UUID at creation time (`crypto.randomUUID()` once, then stored)

**Yomon key sources:**

- ❌ Array index (`map((item, i) => ... key={i})`) — agar list o'zgarsa
- ❌ `Math.random()` — har render'da yangi
- ❌ `Date.now()` — har render'da yangi
- ❌ Object reference (`key={item}`) — agar item obyekt har render'da yangi yaratilsa

### Index as key — qachon OK

Index'ni key sifatida ishlatish **OK** quyidagi shartlarda:

1. List **statik** (hech qachon o'zgarmaydi — items qo'shilmaydi, o'chirilmaydi, qayta tartiblanmaydi)
2. Items'ning ichki state'i yo'q (no input, no useState, no animation)
3. Items mustaqil va bir-biriga bog'liq emas

```tsx
// ✅ OK — statik list, no internal state
const NAVIGATION_ITEMS = ['Home', 'About', 'Contact'];

function Nav() {
  return (
    <ul>
      {NAVIGATION_ITEMS.map((item, i) => (
        <li key={i}>{item}</li>
      ))}
    </ul>
  );
}
```

```tsx
// ❌ Yomon — dynamic list, internal state (input)
function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      {todos.map((todo, i) => (
        <li key={i}>  {/* ❌ — todos qo'shilishi/o'chishi mumkin */}
          <input defaultValue={todo.text} />
        </li>
      ))}
    </ul>
  );
}
```

### Keys va komponent state

`key` o'zgarganda — React bu komponent'ni **butunlay yangi instance** deb hisoblaydi:
- State yo'qoladi (yangidan initial value)
- Effect'lar qayta ishga tushadi (mount cycle)
- Refs qayta yaratiladi

Bu — `key`'ning **state reset trick** sifatida ishlatilishi:

```tsx
function Form({ userId }: { userId: number }) {
  // userId o'zgarganda FormFields qayta mount qilinadi
  // Barcha input state — initial qiymatga qaytadi
  return <FormFields key={userId} />;
}

function FormFields() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  return (
    <>
      <input value={name} onChange={e => setName(e.target.value)} />
      <input value={email} onChange={e => setEmail(e.target.value)} />
    </>
  );
}
```

Bu pattern `33-optimization.md` da yana ishlatiladi (state reset bilan optimization).

### Key warning'lari

ESLint plugin `react/jsx-key` `key` puxta o'rnatilganini tekshiradi. React DevTools console'da warning beradi:

```
Warning: Each child in a list should have a unique "key" prop.
Check the render method of `MyComponent`.
```

Bu warning'ni **e'tiborsiz qoldirish xato** — lekin u silent ravishda noto'g'ri xulq-atvor keltiradi (state chalkashligi, performance).

<details>
<summary><strong>Under the Hood</strong></summary>

**Key Reconciler ichida qanday ishlatiladi:**

`updateSlot` (faza 1) va `updateFromMap` (faza 4) — har ikkalasi `key`'ni tekshiradi:

```typescript
function updateSlot(returnFiber, oldFiber, newChild, lanes) {
  const key = oldFiber !== null ? oldFiber.key : null;
  
  if (typeof newChild === 'object' && newChild !== null) {
    if (newChild.key !== key) {
      return null;  // Key farqli → slot mos emas
    }
    return updateElement(returnFiber, oldFiber, newChild, lanes);
  }
  
  return null;
}

function updateFromMap(existingChildren, returnFiber, newIdx, newChild, lanes) {
  if (typeof newChild === 'object' && newChild !== null) {
    const matchedFiber = existingChildren.get(
      newChild.key === null ? newIdx : newChild.key
    );
    
    if (matchedFiber === null || matchedFiber === undefined) {
      // Topilmadi — yangi
      return createChild(returnFiber, newChild, lanes);
    }
    
    return updateElement(returnFiber, matchedFiber, newChild, lanes);
  }
}
```

`existingChildren` Map'ida key=null Fiber'lar — index orqali saqlanadi. Ya'ni — keyless fiber'lar ham faza 4'da matching ishtirok etadi (lekin index bilan).

**`createFiberFromElement` key'ni nusxalaydi:**

```typescript
function createFiberFromElement(element, mode, lanes) {
  // ...
  const fiber = createFiber(fiberTag, element.props, element.key, mode);
  // ...
}
```

Key Fiber strukturasida saqlanadi va keyingi reconciliation paytida ishlatiladi.

**`__DEV__` warning:**

```typescript
function reconcileChildrenArray(...) {
  if (__DEV__) {
    // Key uniqueness tekshirish
    const knownKeys = new Set();
    for (const child of newChildren) {
      if (child !== null && typeof child === 'object' && child.key !== null) {
        if (knownKeys.has(child.key)) {
          console.error('Encountered two children with the same key, "%s"', child.key);
        } else {
          knownKeys.add(child.key);
        }
      }
    }
  }
  
  // ...actual reconciliation
}
```

Production build'da `__DEV__` `false` bo'ladi — warning'lar olib tashlanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

State reset via key:

```tsx
function UserSettings({ userId }: { userId: number }) {
  return (
    // userId o'zgarganda — Form qayta mount, barcha state reset
    <SettingsForm key={userId} userId={userId} />
  );
}

function SettingsForm({ userId }: { userId: number }) {
  const [theme, setTheme] = useState('light');
  const [notifications, setNotifications] = useState(true);
  
  // userId o'zgarganda useEffect qayta ishga tushadi (key change → unmount + mount)
  useEffect(() => {
    fetchUserSettings(userId).then(s => {
      setTheme(s.theme);
      setNotifications(s.notifications);
    });
  }, [userId]);
  
  return (
    <form>
      <select value={theme} onChange={e => setTheme(e.target.value)}>...</select>
      <input
        type="checkbox"
        checked={notifications}
        onChange={e => setNotifications(e.target.checked)}
      />
    </form>
  );
}
```

Composite key — multiple data'ga asoslangan:

```tsx
function Calendar({ events }: { events: Event[] }) {
  return (
    <div>
      {events.map(event => (
        <EventCard
          // Composite key — event ID + date (agar event qayta-qayta bo'lsa)
          key={`${event.id}_${event.date.toISOString()}`}
          event={event}
        />
      ))}
    </div>
  );
}
```

Generated UUID at creation:

```tsx
interface TodoItem {
  id: string;  // crypto.randomUUID() — bir marta yaratilganda
  text: string;
}

function TodoApp() {
  const [todos, setTodos] = useState<TodoItem[]>([]);
  
  function addTodo(text: string) {
    setTodos([...todos, {
      id: crypto.randomUUID(),  // ✅ bir marta generation, har item uchun
      text,
    }]);
  }
  
  return (
    <ul>
      {todos.map(t => <li key={t.id}>{t.text}</li>)}
    </ul>
  );
}

// ❌ Anti-pattern: crypto.randomUUID() har render'da
function BadTodoApp() {
  const [todos, setTodos] = useState<{text: string}[]>([]);
  
  return (
    <ul>
      {todos.map(t => (
        <li key={crypto.randomUUID()}>{t.text}</li>  // ❌ har render'da yangi key
      ))}
    </ul>
  );
}
// Har render'da yangi key → har item unmount + remount → state yo'qoladi, performance buziladi
```

Index OK — statik list:

```tsx
const TABS = ['Profile', 'Settings', 'Notifications', 'Security'];

function TabBar({ active, onChange }: TabBarProps) {
  return (
    <ul>
      {TABS.map((tab, i) => (
        <li
          key={i}  // ✅ TABS doim bir xil, hech qachon o'zgarmaydi
          className={i === active ? 'active' : ''}
          onClick={() => onChange(i)}
        >
          {tab}
        </li>
      ))}
    </ul>
  );
}
```

</details>

---

## Bailout Algorithm

### Nazariya

**Bailout** — Reconciler subtree'ni qayta render qilishni **skip** qilish qarori. Bu — React'ning eng muhim performance optimizationsi. Bailout 4 ta sababdan biri orqali yuz berishi mumkin:

### Sabab 1: Element identity (`===`)

Agar yangi element **bir xil reference** bo'lsa (`===` strict equality):

```tsx
const memoizedElement = useMemo(() => <Child data={someData} />, [someData]);

return <Parent>{memoizedElement}</Parent>;
```

`memoizedElement` doim bir xil reference (agar deps o'zgarmasa). Reconciler ko'radi: `prevElement === nextElement` (strict equality, NOT `Object.is`) → `Child` Fiber'ni qayta ishlash kerak emas, bailout.

> **Eslatma:** Props uchun React `===` ishlatadi (`oldProps === newProps`), state uchun esa `Object.is` (`useState`'ning eager bailout path'ida). Bu — ikki kod yo'lining tarixiy farqi.

Bu — **eng tezkor** bailout — Reconciler `beginWork`'ni umuman chaqirmaydi.

### Sabab 2: React.memo shallow check

`React.memo()` bilan o'ralgan komponentlar:

```tsx
const MemoChild = React.memo(function Child({ count }: { count: number }) {
  return <div>{count}</div>;
});
```

`MemoChild` Fiber tag'i — `MemoComponent`. `beginWork` chaqirilganda `updateMemoComponent` ishga tushadi:

1. Eski va yangi props shallow-equal solishtiriladi (`Object.is` har property uchun)
2. Equal bo'lsa → bailout (child render'i skip)
3. Equal emas — child render qilinadi

```typescript
// React internal (soddalashtirilgan)
function updateMemoComponent(current, workInProgress, ...) {
  if (current === null) {
    // Mount — har doim render
    // ...
    return updateFunctionComponent(...);
  }
  
  const prevProps = current.memoizedProps;
  const nextProps = workInProgress.pendingProps;
  
  const compare = workInProgress.type.compare;
  const equal = compare !== null
    ? compare(prevProps, nextProps)
    : shallowEqual(prevProps, nextProps);
  
  if (equal && current.ref === workInProgress.ref) {
    // Bailout — child render qilinmaydi
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }
  
  // Render davom etadi
  return updateFunctionComponent(...);
}
```

**Custom comparator:**

```tsx
const Child = React.memo(
  function Child({ user }: { user: User }) {
    return <div>{user.name}</div>;
  },
  (prevProps, nextProps) => prevProps.user.id === nextProps.user.id
);
// Faqat user.id o'zgargandagina re-render
```

### Sabab 3: useMemo / useCallback stable reference

`useMemo` va `useCallback` — reference barqarorlik vositasi. Bu pattern `React.memo` bilan birga ishlaydi:

```tsx
const MemoChild = React.memo(function Child({ onClick }: { onClick: () => void }) {
  return <button onClick={onClick}>Click</button>;
});

// ❌ Anti-pattern — har render'da yangi function reference
function BadParent() {
  const [count, setCount] = useState(0);
  return <MemoChild onClick={() => console.log('click')} />;
  // Har BadParent render'da yangi function → MemoChild props farqli (Object.is false)
  // → bailout YO'Q, MemoChild har doim re-render bo'ladi
}

// ✅ Stable function via useCallback
function GoodParent() {
  const [count, setCount] = useState(0);
  const handleClick = useCallback(() => console.log('click'), []);
  return <MemoChild onClick={handleClick} />;
  // handleClick doim bir xil reference → shallow equal → bailout BOR
  // → MemoChild render skip qilinadi
}
```

### Sabab 4: State equality (Object.is)

`setState(currentValue)` chaqirilsa, React `Object.is(prevState, nextState)` ni tekshiradi:

```tsx
const [count, setCount] = useState(0);

setCount(0);  // count=0 → 0
// React: Object.is(0, 0) → true → state o'zgarmagan
// → Komponent re-render qilinmaydi (bailout)
```

Bu — primitiv qiymatlar uchun yaxshi ishlaydi. Lekin obyekt'lar uchun reference farqli bo'lsa — render qilinadi:

```tsx
const [user, setUser] = useState({ name: 'Ali' });

setUser({ name: 'Ali' });  // yangi obyekt!
// React: Object.is({ name: 'Ali' }, { name: 'Ali' }) → false (boshqa reference)
// → Komponent re-render
```

### Bailout va childLanes

`childLanes` — komponentning subtree'sida pending update bo'lgani belgisi. Reconciler bailout qilganda **childLanes ham 0 bo'lishi shart** — aks holda subtree'da pending update bor (boshqa komponent setState chaqirgan). Bu holda Reconciler subtree'ga tushib, faqat tegishli Fiber'lar uchun render qiladi.

```typescript
function bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes) {
  // Subtree'da hech qanday pending update yo'q
  if (!includesSomeLane(workInProgress.childLanes, renderLanes)) {
    return null;  // Butun subtree skip
  }
  
  // Childlar'da ish bor — recursive descent kerak
  cloneChildFibers(current, workInProgress);
  return workInProgress.child;
}
```

Bu mexanizm — context'lar, useReducer, va boshqa cross-component state update'lar uchun zarur.

<details>
<summary><strong>Under the Hood</strong></summary>

**Bailout sababi 1 — element identity:**

`beginWork` ichida props va lanes tekshiriladi. Early bailout 3 ta invariant talab qiladi (manba: `react-reconciler/src/ReactFiberBeginWork.js`, `attemptEarlyBailoutIfNoScheduledUpdate`):

1. **`oldProps === newProps`** — reference identity (Object.is emas, `===`)
2. **Fiber'ning `lanes`'i `renderLanes` bilan kesishmaydi** — `!includesSomeLane(renderLanes, workInProgress.lanes)`
3. **`hasLegacyContextChanged() === false`** — eski (legacy) context API o'zgarmagan

Uchchala shart bajarilsa, Reconciler `beginWork`'ning to'liq yo'lini chetlab o'tadi:

```typescript
function beginWork(current, workInProgress, renderLanes) {
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    
    if (oldProps !== newProps || hasLegacyContextChanged()) {
      // Props o'zgargan yoki legacy context — render
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, workInProgress.lanes)) {
      // 3 ta shart bajarildi — early bailout
      didReceiveUpdate = false;
      return attemptEarlyBailoutIfNoScheduledUpdate(current, workInProgress, renderLanes);
    }
  }
  // ...full render path
}
```

`useMemo` ishlatilgan element bir xil reference bilan keladi:

```tsx
function Parent({ data }) {
  const child = useMemo(() => <Child data={data} />, [data]);
  return <div>{child}</div>;
}

// data o'zgarmaganda:
// child — bir xil React Element obyekt (memoized)
// Parent qayta render qilinganda <Child> Fiber'ning pendingProps === memoizedProps
// (chunki props obyekti — element.props — bir xil reference)
// → early bailout ishlaydi
```

**Bailout sababi 2 — React.memo shallow:**

`shallowEqual` implementation'i:

```typescript
function shallowEqual(prevProps, nextProps) {
  if (Object.is(prevProps, nextProps)) return true;
  
  if (typeof prevProps !== 'object' || prevProps === null
   || typeof nextProps !== 'object' || nextProps === null) {
    return false;
  }
  
  const prevKeys = Object.keys(prevProps);
  const nextKeys = Object.keys(nextProps);
  
  if (prevKeys.length !== nextKeys.length) return false;
  
  for (const key of prevKeys) {
    if (!Object.prototype.hasOwnProperty.call(nextProps, key)
     || !Object.is(prevProps[key], nextProps[key])) {
      return false;
    }
  }
  
  return true;
}
```

Shallow — faqat **birinchi daraja** taqqoslash. Nested obyekt'lar reference orqali solishtiriladi:

```tsx
const prev = { user: { name: 'Ali' } };
const next = { user: { name: 'Ali' } };
shallowEqual(prev, next);  // false (user obyekti yangi reference)

const sameUser = { name: 'Ali' };
const prev2 = { user: sameUser };
const next2 = { user: sameUser };
shallowEqual(prev2, next2);  // true (user reference bir xil)
```

**Bailout sababi 4 — state equality:**

`useState` setter implementation'i:

```typescript
function dispatchSetState(fiber, queue, action) {
  const lane = requestUpdateLane(fiber);
  
  // Eager state computation (optimization)
  const update = { lane, action, eagerState: null, hasEagerState: false };
  const alternate = fiber.alternate;
  
  if (fiber.lanes === NoLanes && (alternate === null || alternate.lanes === NoLanes)) {
    // Hech qanday pending update yo'q — eager hisoblash
    const lastRenderedReducer = queue.lastRenderedReducer;
    const currentState = queue.lastRenderedState;
    const eagerState = lastRenderedReducer(currentState, action);
    
    update.eagerState = eagerState;
    update.hasEagerState = true;
    
    if (Object.is(eagerState, currentState)) {
      // Bailout — state o'zgarmagan; React `enqueueConcurrentHookUpdateAndEagerlyBailout`
      // chaqirib, update'ni queue'ga keyingi commit uchun saqlaydi-yu,
      // `scheduleUpdateOnFiber`'ni chaqirmaydi — render boshlanmaydi.
      return;
    }
  }
  
  // Normal scheduling
  enqueueUpdate(fiber, queue, update);
  scheduleUpdateOnFiber(fiber, lane);
}
```

Agar eager bailout muvaffaqiyatli — Scheduler chaqirilmaydi, render umuman boshlanmaydi.

**`childLanes` bailout — context misol:**

```tsx
const ThemeContext = React.createContext('light');

function App() {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <Layout />  {/* memo'd */}
    </ThemeContext.Provider>
  );
}

const Layout = React.memo(function Layout() {
  return (
    <div>
      <Header />  {/* useContext(ThemeContext) — consumer */}
      <Main />
    </div>
  );
});

// setTheme chaqirildi:
// - App re-render
// - ThemeContext.Provider o'rnatildi
// - Layout: shallow check → props o'zgarmagan → bailout
//   LEKIN: Layout subtree'da Header — Theme consumer
//   childLanes set qilingan (context update propagation tufayli)
//   → bailoutOnAlreadyFinishedWork bu holatni aniqlaydi
//   → child'lar bo'ylab tushadi (faqat Header re-render bo'ladi)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Element identity bailout (useMemo bilan):

```tsx
function Parent({ items }: { items: Item[] }) {
  const [count, setCount] = useState(0);
  
  // Memoized children — items o'zgarmasa, bir xil reference
  const itemList = useMemo(
    () => <ItemList items={items} />,
    [items]
  );
  
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      {itemList}  {/* count o'zgarganda — itemList reference bir xil → bailout */}
    </>
  );
}

function ItemList({ items }: { items: Item[] }) {
  console.log('ItemList render');  // count o'zgarganda chiqmaydi (bailout)
  return <ul>{items.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}
```

React.memo bailout misol:

```tsx
const ExpensiveChild = React.memo(function ExpensiveChild({ data }: { data: number[] }) {
  console.log('ExpensiveChild render');
  
  // Heavy computation
  const sum = data.reduce((a, b) => a + b, 0);
  
  return <div>Sum: {sum}</div>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const data = [1, 2, 3, 4, 5];  // ⚠️ har render'da yangi array
  
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <ExpensiveChild data={data} />
    </>
  );
}

// Count bosildi:
// - Parent re-render
// - data — yangi array (yangi reference)
// - ExpensiveChild: shallow check → data farqli reference → BAILOUT YO'Q
// - "ExpensiveChild render" chiqadi
// 
// Bu — ko'p uchraydigan xato. data har render'da yangi yaratiladi.
```

```tsx
// ✅ TO'G'RI — useMemo bilan stable reference
function Parent() {
  const [count, setCount] = useState(0);
  
  const data = useMemo(() => [1, 2, 3, 4, 5], []);  // Bir marta yaratiladi
  
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <ExpensiveChild data={data} />
    </>
  );
}

// Count bosildi:
// - Parent re-render
// - data — bir xil reference (useMemo)
// - ExpensiveChild: shallow check → data bir xil → BAILOUT
// - "ExpensiveChild render" CHIQMAYDI
```

State equality bailout:

```tsx
function ToggleableSection() {
  const [open, setOpen] = useState(false);
  
  console.log('Render');
  
  return (
    <>
      <button onClick={() => setOpen(true)}>Open</button>
      <button onClick={() => setOpen(false)}>Close</button>
    </>
  );
}

// Initial render: "Render"
// "Open" bosildi (open: false → true): "Render"
// "Open" yana bosildi (open: true → true): bailout, "Render" CHIQMAYDI
//   Object.is(true, true) → true → React eager bailout, scheduling yo'q
// "Close" bosildi (open: true → false): "Render"
```

useReducer bailout:

```tsx
const reducer = (state: number, action: 'inc' | 'reset') => {
  if (action === 'inc') return state + 1;
  if (action === 'reset') return state;  // ⚠️ same value
  return state;
};

function Counter() {
  const [count, dispatch] = useReducer(reducer, 0);
  
  console.log('Render', count);
  
  return (
    <>
      <button onClick={() => dispatch('inc')}>+</button>
      <button onClick={() => dispatch('reset')}>Reset</button>
    </>
  );
}

// + bosildi (count: 0 → 1): "Render 1"
// + bosildi (count: 1 → 2): "Render 2"
// "Reset" bosildi (count: 2 → 2 — reducer same value qaytaradi): bailout, "Render" CHIQMAYDI
```

Custom comparator misol:

```tsx
interface UserCardProps {
  user: { id: number; name: string; lastSeen: Date };
  onClick: () => void;
}

const UserCard = React.memo(
  function UserCard({ user, onClick }: UserCardProps) {
    return (
      <div onClick={onClick}>
        <h3>{user.name}</h3>
        <p>{user.lastSeen.toLocaleString()}</p>
      </div>
    );
  },
  (prevProps, nextProps) => {
    // Faqat id o'zgargandagina re-render
    // (lastSeen har polling'da o'zgaradi, lekin bizga muhim emas)
    return prevProps.user.id === nextProps.user.id;
  }
);
```

</details>

---

## Update Propagation

### Nazariya

`setState` chaqirilganda update qaerdan boshlanadi va qayerga yetadi?

**Asosiy mexanika:**

1. `setState(fiber)` chaqiriladi — bu Fiber `lanes` bilan belgilanadi (priority)
2. Fiber'ning **barcha ancestor'lari** `childLanes` bilan belgilanadi (subtree'da update bor)
3. Render Phase — root'dan boshlanadi (har doim root'dan!)
4. Har Fiber'da:
   - Agar `fiber.lanes` set bo'lsa → bu Fiber re-render
   - Agar `fiber.childLanes` set bo'lsa → child'lar bo'ylab tushish
   - Agar ikkalasi ham 0 bo'lsa → bailout

**Diagram:**

```
       Root
        │ (childLanes set)
        ▼
       App
        │ (childLanes set — Counter ostida)
        ▼
       Layout (memo'd, props o'zgarmagan, lekin childLanes set!)
        │ (subtree'da update bor — descend kerak)
        ▼
       Sidebar    Main
       (no work)   │ (childLanes set)
                   ▼
                 Counter (lanes set — re-render qiladi)
```

**Why root'dan boshlanadi:**

Reconciler har vaqtda butun tree'dan o'tadi (root'dan boshlab). Bu — concurrent rendering uchun zarur. Agar Reconciler "faqat o'zgargan komponent'dan" boshlasa:
- Boshqa komponent'lardan kelgan update'lar e'tibordan chetda qolar edi
- Tree consistent emas (yarim eski, yarim yangi)
- Concurrent invariants buzilardi

`childLanes` orqali Reconciler tree bo'ylab tezkor tushadi — bailout bo'lgan branch'lar darhol skip qilinadi.

### `lanes` va `childLanes` mexanikasi

Har Fiber'da ikkita bayroq mavjud:

| Maydon | Ma'no |
|--------|-------|
| `lanes` | Bu Fiber'ning o'zida pending update bor (setState qilingan) |
| `childLanes` | Subtree'da hech bo'lmaganda bitta Fiber'da `lanes` set qilingan |

`lanes` va `childLanes` — **bitmask** (bir nechta lane priority'ni bir vaqtda saqlash uchun).

**Update propagation mexanikasi:**

```typescript
// React internal (soddalashtirilgan)
function markUpdateLaneFromFiberToRoot(sourceFiber, lane) {
  // 1. O'zining lanes'iga lane qo'shish
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  
  let alternate = sourceFiber.alternate;
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
  
  // 2. Parent'lar bo'ylab childLanes propagate qilish
  let parent = sourceFiber.return;
  while (parent !== null) {
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    
    if (parent.alternate !== null) {
      parent.alternate.childLanes = mergeLanes(parent.alternate.childLanes, lane);
    }
    
    if (parent.tag === HostRoot) {
      // Root'ga yetildi
      break;
    }
    
    parent = parent.return;
  }
  
  // 3. Root'dan render rejalashtirish
  scheduleUpdateOnRoot(...);
}
```

Bu — **child'dan root'gacha bayroq qo'yish**. Render esa boshqa yo'nalishda — root'dan child'larga.

<details>
<summary><strong>Under the Hood</strong></summary>

**Render Phase tree walk va childLanes:**

```typescript
function beginWork(current, workInProgress, renderLanes) {
  // Bailout tekshirish
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    
    if (oldProps !== newProps || hasLegacyContextChanged()) {
      // Props o'zgargan — render kerak
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, workInProgress.lanes)) {
      // Bu Fiber uchun update yo'q — bailout
      didReceiveUpdate = false;
      
      // Lekin childLanes tekshirish kerak
      return attemptEarlyBailoutIfNoScheduledUpdate(current, workInProgress, renderLanes);
    }
  }
  
  // Render davom etadi — komponent funksiyasi chaqiriladi
  // ...
}

function attemptEarlyBailoutIfNoScheduledUpdate(current, workInProgress, renderLanes) {
  // Agar subtree'da pending update bo'lsa, child'larga tushish kerak
  if (!includesSomeLane(workInProgress.childLanes, renderLanes)) {
    // Subtree'da ham update yo'q — butun subtree skip
    return null;
  }
  
  // Child Fiber'larni clone qilish (alternate'dan reuse)
  cloneChildFibers(current, workInProgress);
  return workInProgress.child;
}
```

Bu mexanizm `childLanes` orqali Reconciler tree'ni **skip-friendly** qiladi:
- Update yo'q bo'lgan branch'lar darhol skip
- Update bo'lgan branch'larga tushib, faqat tegishli Fiber'lar uchun render

**Lanes bilmask afzalligi:**

`lanes` 31-bit bitmask. Bir Fiber'da bir nechta priority lane bo'lishi mumkin:

```typescript
// Manba: react/packages/react-reconciler/src/ReactFiberLane.js
// TotalLanes = 31 (V8 smi diapazonida bitmask sifatida saqlash uchun)
// Pastki bit = yuqori priority. Birinchi lane'lar (stabil tartib):
const SyncHydrationLane     = 0b0000000000000000000000000000001;  // bit 0
const SyncLane              = 0b0000000000000000000000000000010;  // bit 1
const InputContinuousHydrationLane
                            = 0b0000000000000000000000000000100;  // bit 2
const InputContinuousLane   = 0b0000000000000000000000000001000;  // bit 3
const DefaultHydrationLane  = 0b0000000000000000000000000010000;  // bit 4
const DefaultLane           = 0b0000000000000000000000000100000;  // bit 5
// Yuqoriroq bit'lar: TransitionHydrationLane, TransitionLane1..TransitionLaneN
// (rotating — starvation oldini olish uchun), RetryLane'lar,
// SelectiveHydrationLane, IdleHydrationLane, IdleLane, OffscreenLane.
// To'liq ro'yxat va aniq bit pozitsiyalari React versiyasiga qarab o'zgaradi —
// 05-scheduler-lanes.md da source bilan birga keltirilgan.

// Bir Fiber'da bir vaqtda bir nechta lane bo'lishi mumkin:
fiber.lanes = SyncLane | TransitionLane1;

// Tekshirish
if (fiber.lanes & SyncLane) { ... }
if (includesSomeLane(fiber.lanes, renderLanes)) { ... }
```

Lanes `05-scheduler-lanes.md` da chuqur yoritiladi.

**Provider va childLanes:**

Context Provider value o'zgarganda — barcha consumer Fiber'larining `lanes`'ini set qilish kerak. Bu propagate qilish uchun React tree bo'ylab tushadi:

```typescript
function propagateContextChange(workInProgress, context, renderLanes) {
  let fiber = workInProgress.child;
  
  while (fiber !== null) {
    // Bu Fiber Context'ni consume qilayaptimi?
    if (fiber.dependencies !== null) {
      let dependency = fiber.dependencies.firstContext;
      while (dependency !== null) {
        if (dependency.context === context) {
          // Match — fiber.lanes update
          fiber.lanes = mergeLanes(fiber.lanes, renderLanes);
          break;
        }
        dependency = dependency.next;
      }
    }
    
    // Recursive descent
    if (fiber.child !== null) fiber = fiber.child;
    else if (fiber === workInProgress) break;
    else {
      while (fiber.sibling === null) {
        if (fiber.return === null || fiber.return === workInProgress) return;
        fiber = fiber.return;
      }
      fiber = fiber.sibling;
    }
  }
}
```

Bu — Context Provider performance considerations'ning sababi. `useContext` chuqur `19-usecontext.md` da.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Update propagation visualization:

```tsx
function Root() {
  return <App />;
}

function App() {
  return (
    <Layout>
      <Sidebar />
      <Main>
        <Content />
        <Counter />
      </Main>
    </Layout>
  );
}

const Layout = React.memo(function Layout({ children }) {
  console.log('Layout render');
  return <div>{children}</div>;
});

function Sidebar() {
  console.log('Sidebar render');
  return <aside>Sidebar</aside>;
}

function Main({ children }) {
  console.log('Main render');
  return <main>{children}</main>;
}

function Content() {
  console.log('Content render');
  return <article>Content</article>;
}

function Counter() {
  const [count, setCount] = useState(0);
  console.log('Counter render');
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// Counter button bosildi:
// 1. Counter Fiber lanes set
// 2. Parent'lar (Main, Layout, App, Root) — childLanes set
// 3. Render boshlanadi root'dan:
//    - Root: childLanes set → descend
//    - App: childLanes set → descend
//    - Layout: lanes=0 → bailout candidate
//      - childLanes set → descend kerak (subtree'da update bor)
//      - Layout funksiyasi qayta CHAQIRILMAYDI (bailout)
//      - Children clone qilinadi alternate'dan
//      - Sidebar, Main child'lar ko'rib chiqiladi
//    - Sidebar: lanes=0, childLanes=0 → BUTUN SUBTREE SKIP
//    - Main: lanes=0, props o'zgarmagan (Layout bailout qilib children'ni
//        clone qilgan — Main'ning pendingProps === memoizedProps) → bailout candidate
//      - childLanes set → descend kerak (Counter ostida)
//      - Main funksiyasi qayta CHAQIRILMAYDI (early bailout)
//      - Content: lanes=0, childLanes=0 → SKIP
//      - Counter: lanes set → re-render

// Console log:
// "Counter render"
// (Layout, Main, Sidebar, Content — render YO'Q: faqat Counter)
//
// Muhim: Main memo emas, lekin parent (Layout) bailout qilgani uchun
// Main'ga clone'langan, o'zgarmagan props yetib keldi va Main'ning o'z lanes'i 0.
// Non-memo komponent "har doim parent bilan birga render bo'ladi" emas —
// agar parent funksiyasi qayta chaqirilmasa (bailout), child yangi props olmaydi.
```

childLanes context misol:

```tsx
const UserContext = React.createContext<User | null>(null);

function App() {
  const [user, setUser] = useState<User>({ name: 'Ali', age: 25 });
  
  return (
    <UserContext.Provider value={user}>
      <button onClick={() => setUser({ name: 'Vali', age: 30 })}>
        Change user
      </button>
      <Layout />  {/* memo'd */}
    </UserContext.Provider>
  );
}

const Layout = React.memo(function Layout() {
  console.log('Layout render');
  return (
    <div>
      <Header />  {/* useContext consumer */}
      <Sidebar />  {/* no context */}
    </div>
  );
});

function Header() {
  const user = useContext(UserContext);
  console.log('Header render');
  return <h1>Hello, {user?.name}</h1>;
}

function Sidebar() {
  console.log('Sidebar render');
  return <aside>Menu</aside>;
}

// "Change user" bosildi:
// - User state o'zgardi → UserContext.Provider value yangi
// - Provider Reconciliation: barcha consumer Fiber'larning lanes set
//   - Header.lanes = SyncLane (consumer)
//   - Layout.childLanes = SyncLane (Header ostida)
// - Render:
//   - App re-render
//   - Layout: lanes=0, props bir xil → bailout candidate
//     - childLanes set → descend
//     - Layout function qayta chaqirilmaydi
//   - Header: lanes set → re-render (yangi user)
//   - Sidebar: lanes=0, childLanes=0 → SKIP
//
// Console:
// "Header render"
// (Layout, Sidebar — yo'q)
```

</details>

---

## Edge Cases va Gotchas

### Sibling Fragment'lar — keys

```tsx
// ❌ Shorthand `<>...</>` fragment key qabul QILMAYDI — bunday yozish syntax xato:
//    <key={item.id}>...</>  ← bu kod parse bo'lmaydi
//
// Shu sababli list'da fragment'ga key kerak bo'lsa, to'liq Fragment ishlatiladi:

// ✅ To'liq Fragment — key qabul qiladi
import { Fragment } from 'react';
{items.map(item => (
  <Fragment key={item.id}>
    <h3>{item.title}</h3>
    <p>{item.body}</p>
  </Fragment>
))}
```

`<>...</>` shorthand JSX `<Fragment>` ga qisqartirilganda key qabul qilmaydi — bu syntax cheklov.

---

### `key` qoidalari faqat list'lar uchun

```tsx
// Single child uchun key kerak emas
function App() {
  return <Counter key="something" />;  // ⚠️ key bu yerda useless
}

// Faqat siblings array bo'lsa key kerak
function List({ items }) {
  return (
    <ul>
      {items.map(item => <li key={item.id}>{item.name}</li>)}  {/* key kerak */}
    </ul>
  );
}
```

`key` faqat siblings (parent ichidagi multiple child'lar) orasida identity uchun kerak. Single element'da key effekt'siz.

---

### Type comparison reference — anonymous komponentlar

```tsx
// ❌ Render davomida component yaratish — har render yangi reference
function Parent() {
  return <div>{React.createElement(() => <span>...</span>)}</div>;
  // Anonymous function har render'da yangi → unmount + remount
}

// ✅ Stable reference
const Span = () => <span>...</span>;

function Parent() {
  return <div><Span /></div>;
}
```

---

### Array'da turli tipdagi children

```tsx
function App() {
  const [showHeader, setShowHeader] = useState(true);
  
  return (
    <div>
      {showHeader && <h1>Header</h1>}  {/* false bo'lsa — null */}
      <p>Body</p>
    </div>
  );
}

// showHeader=true: children = [<h1>, <p>]
// showHeader=false: children = [false, <p>]
//
// React `false`/`null`/`undefined`/`true` ni "no fiber" deb hisoblaydi —
// bu slot uchun Fiber yaratilmaydi, lekin pozitsiya YO'QOLMAYDI (newIdx aynan
// shu newChildren array index'i bo'yicha yuritiladi). Demak:
//
// Reconciler keyless (index-based) matching:
//   newIdx=0, newChild=false → updateSlot null qaytaradi → faza 1 to'xtaydi
//   Faza 4 (keyed map): existingChildren = { 0: h1Fiber, 1: pFiber }
//     newIdx=0 (false) → skip (no fiber)
//     newIdx=1 (<p>) → existingChildren.get(1) → pFiber MATCH → reuse
//   Tugagach: existingChildren'da qoldiq h1 → ChildDeletion
//
// Natija: h1 unmount, p REUSE — state SAQLANADI (DOM node identity ham)
//
// ⚠️ Lekin bu xulq-atvor faqat `{condition && <Element />}` shaklida — element
//    pozitsiyasi siblings array ichida o'zgarmaganda — to'g'ri ishlaydi.
//    Element pozitsiyasi o'zgarsa (masalan ikki conditional aralashsa) —
//    keys ishlatish xavfsizroq:
// {showHeader && <h1 key="header">Header</h1>}
// <p key="body">Body</p>
```

---

### Bailout React.memo'da — children prop muammosi

```tsx
const MemoChild = React.memo(function Child({ children }) {
  console.log('Child render');
  return <div>{children}</div>;
});

function Parent() {
  const [count, setCount] = useState(0);
  
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <MemoChild>
        <span>Static</span>  {/* har render'da yangi <span> element */}
      </MemoChild>
    </>
  );
}

// Count bosildi:
// - children prop = <span>Static</span> — har render'da yangi React element
// - shallowEqual({children: oldSpan}, {children: newSpan}) → false
// - MemoChild bailout YO'Q
// - "Child render" CHIQADI

// To'g'ri: useMemo bilan stable children
function Parent() {
  const [count, setCount] = useState(0);
  const child = useMemo(() => <span>Static</span>, []);
  
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <MemoChild>{child}</MemoChild>
    </>
  );
}
// Endi children stable reference → bailout
```

---

## Common Mistakes

### ❌ Xato 1: Index as key dynamic list'da

```tsx
// ❌ Items qo'shilishi/o'chirilishi mumkin
{todos.map((todo, i) => <Todo key={i} todo={todo} />)}

// ✅ Stable unique ID
{todos.map(todo => <Todo key={todo.id} todo={todo} />)}
```

---

### ❌ Xato 2: `Math.random()` yoki `Date.now()` key sifatida

```tsx
// ❌ Har render'da yangi key
{items.map(item => (
  <li key={Math.random()}>{item.name}</li>
))}
// Har render — barcha item'lar unmount + remount → state yo'qoladi, performance buziladi

// ✅ Item'ning stable ID'si
{items.map(item => (
  <li key={item.id}>{item.name}</li>
))}
```

---

### ❌ Xato 3: Inline component har render'da yangi

```tsx
// ❌ Inline function — har render'da yangi reference
function Parent() {
  const Card = () => <div>Card</div>;
  return <Card />;
  // Har Parent render'da yangi Card function → unmount + remount Card
}

// ✅ Outside-defined
const Card = () => <div>Card</div>;
function Parent() {
  return <Card />;
}
```

---

### ❌ Xato 4: Object/array literal prop React.memo bilan

```tsx
const MemoChild = React.memo(function Child({ config }) {
  return <div>{config.label}</div>;
});

// ❌ Har render'da yangi obyekt
function Parent() {
  return <MemoChild config={{ label: 'Save' }} />;
  // Memo bailout YO'Q (config har safar yangi reference)
}

// ✅ Stable reference
function Parent() {
  const config = useMemo(() => ({ label: 'Save' }), []);
  return <MemoChild config={config} />;
}
```

---

### ❌ Xato 5: Type oscillation — wrap toggle

```tsx
// ❌ Toggle wrapping — child state yo'qoladi
function Parent({ wrapInForm }) {
  return wrapInForm
    ? <form><Input /></form>
    : <Input />;
  // Toggle bo'lganda Input — boshqa parent type → unmount + mount
}

// ✅ Same parent, conditional inner element
function Parent({ wrapInForm }) {
  if (wrapInForm) {
    return <form><Input /></form>;
  }
  return <Input />;
}
// Bu hali ham xato — Input ikki branch'da boshqa pozitsiyada

// ✅ Same parent type:
function Parent({ wrapInForm }) {
  const inputElement = <Input />;
  return wrapInForm ? <form>{inputElement}</form> : <div>{inputElement}</div>;
}
// Hali ham — div va form turli type
//
// Eng to'g'ri:
function Parent({ wrapInForm }) {
  return (
    <div>
      <Input />
      {wrapInForm && <button type="submit">Submit</button>}
    </div>
  );
}
// Input doim bir xil pozitsiyada, parent div doim bir xil → state saqlanadi
```

---

## Amaliy Mashqlar

### Mashq 1: Type comparison qaror (Oson)

Quyidagi holatlarda Reconciler nima qiladi?

```tsx
// Holat 1
// Eski: <div className="a"><Counter /></div>
// Yangi: <div className="b"><Counter /></div>

// Holat 2
// Eski: <div><Counter /></div>
// Yangi: <section><Counter /></section>

// Holat 3
// Eski: <Counter />
// Yangi: <MemoCounter />
```

<details>
<summary><strong>Javob</strong></summary>

**Holat 1:** `<div>` type bir xil → reuse Fiber → className update. Counter — same type → reuse, state SAQLANADI.

**Holat 2:** `<div>` → `<section>` type farqli → unmount old subtree, mount new. Counter — yangi instance, state YO'QOLADI.

**Holat 3:** `Counter` (FunctionComponent) va `MemoCounter` (memo wrapper, MemoComponent tag) — turli type. Unmount + mount. Counter ichidagi state YO'QOLADI.

</details>

---

### Mashq 2: Keyless vs keyed reorder (O'rta)

Quyidagi reorder uchun (eski → yangi) **keyless** va **keyed** matching qancha DOM mutation qiladi?

```
Eski: [A, B, C, D]
Yangi: [B, A, C, D]  (A va B almashildi)
```

<details>
<summary><strong>Javob</strong></summary>

**Keyless (index-based):**
- Position 0: A → B → same type, props farqli → Update
- Position 1: B → A → Update
- Position 2: C → C → no change
- Position 3: D → D → no change

**2 ta DOM mutation** (ikkita Update). Lekin agar A, B, C, D — komponentlar bo'lsa, ularning state'i pozitsiya bo'yicha qoladi (B endi A pozitsiyasida, lekin Fiber identity A pozitsiyasi bilan bog'liq).

**Keyed:**
- Faza 1: position 0 → A vs B (key farqli) → faza 1 to'xtaydi
- Faza 4: existingChildren = { A, B, C, D }
  - newIdx=0: B → topildi (oldIndex=1), lastPlacedIndex=1, no move
  - newIdx=1: A → topildi (oldIndex=0), 0 < 1 → MOVE (Placement flag)
  - newIdx=2: C → topildi (oldIndex=2), 2 >= 1, no move
  - newIdx=3: D → topildi (oldIndex=3), 3 >= 2, no move

**1 ta DOM mutation** (faqat A move). Komponent state'lari A va B Fiber'larida saqlanadi (ko'chiriladi-yu, identity saqlanadi).

</details>

---

### Mashq 3: Bailout sabab aniqlash (O'rta)

Quyidagi har holatda qaysi bailout sababi ishlaydi (yoki ishlamaydi)?

```tsx
// Holat 1
const memoChild = useMemo(() => <Child />, []);
return <Parent>{memoChild}</Parent>;

// Holat 2
const MemoChild = React.memo(Child);
return <MemoChild value={42} />;

// Holat 3
const [count, setCount] = useState(0);
setCount(0);

// Holat 4
const handleClick = useCallback(() => doSomething(), []);
return <MemoButton onClick={handleClick} />;

// Holat 5
return <MemoChild config={{ theme: 'dark' }} />;
```

<details>
<summary><strong>Javob</strong></summary>

**Holat 1:** Element identity bailout. `memoChild` doim bir xil reference (useMemo) → Reconciler `prevElement === nextElement` → bailout. Eng tezkor bailout.

**Holat 2:** React.memo shallow check. `value=42` primitive, har render'da bir xil → shallowEqual true → bailout.

**Holat 3:** State equality (Object.is). `Object.is(0, 0) === true` → eager bailout, scheduling yo'q. Komponent re-render qilinmaydi.

**Holat 4:** useCallback + React.memo birgalikda. `handleClick` doim bir xil reference → MemoButton shallow check pass → bailout.

**Holat 5:** **Bailout YO'Q.** `{ theme: 'dark' }` har render'da yangi obyekt. ShallowEqual reference farqli → render davom etadi.

To'g'rilash: `const config = useMemo(() => ({ theme: 'dark' }), []);` keyin bailout ishlaydi.

</details>

---

### Mashq 4: childLanes propagation (Qiyin)

Quyidagi tree'da `Counter` ichidagi `setCount` chaqirildi. Render Phase qaysi komponent'larda `beginWork` chaqiradi va qaysi'larda bailout qiladi?

```tsx
function Root() {
  return <App />;
}

function App() {
  return (
    <Layout>
      <Sidebar />
      <Main>
        <Article />
        <Counter />
      </Main>
    </Layout>
  );
}

const Layout = React.memo(function Layout({ children }) {
  return <div>{children}</div>;
});

const Sidebar = React.memo(function Sidebar() {
  return <aside>Menu</aside>;
});

function Main({ children }) {
  return <main>{children}</main>;
}

function Article() {
  return <article>Static content</article>;
}

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

<details>
<summary><strong>Javob</strong></summary>

`setCount` chaqirilganda faqat **Counter Fiber'ning `lanes`'i** set bo'ladi. Ancestor'larda esa faqat `childLanes` set bo'ladi — ularning o'z `lanes`'i 0 qoladi:

1. **Counter Fiber:** `lanes` set
2. **Parent'lar:** `childLanes` set (o'z `lanes` emas):
   - Main.childLanes set
   - Layout.childLanes set
   - App.childLanes set
   - Root.childLanes set

Asosiy nuqta: ancestor `beginWork`'ga yetilganda, uning `pendingProps === memoizedProps` (clone'langan, o'zgarmagan) va o'z `lanes`'i 0 → **early bailout** path. Bailout `cloneChildFibers` bilan child'larni clone qiladi va `childLanes` orqali pastga tushadi, lekin **komponent funksiyasini qayta CHAQIRMAYDI**. Demak ancestor yangi `children` element ham yaratmaydi — pastdagi memo'd komponentlarning props'i o'zgarmaydi.

Render Phase tree walk:

| Komponent | `lanes` | `childLanes` | Funksiya qayta chaqiriladimi? | Sabab |
|-----------|---------|--------------|-------------------------------|-------|
| Root | 0 | set | Yo'q (bailout) | props o'zgarmagan + lanes 0 → early bailout; childLanes → clone + descend |
| App | 0 | set | Yo'q (bailout) | clone'langan props === eski; lanes 0 → early bailout; childLanes → descend |
| Layout | 0 | set | Yo'q (bailout) | App bailout qilgani uchun `children` prop yangi emas (clone); memo + lanes 0 → bailout; childLanes → descend |
| Sidebar | 0 | 0 | Yo'q (skip) | Memo + lanes 0 + childLanes 0 → butun subtree skip |
| Main | 0 | set | Yo'q (bailout) | clone'langan props === eski; lanes 0 → early bailout (memo bo'lmasa ham); childLanes → descend |
| Article | 0 | 0 | Yo'q (skip) | lanes 0 + childLanes 0 → skip |
| Counter | set | 0 | **Ha** | lanes set — yagona haqiqiy re-render |

**Eslatma:** Bu yerda hech bir ancestor funksiyasi qayta chaqirilmaydi. Sababi — `setCount` faqat Counter'ning `lanes`'ini belgilaydi; ancestor'larda esa faqat `childLanes`. Ancestor `beginWork`'da `pendingProps === memoizedProps` (clone) va `lanes === 0` shartlari bajarilib, early bailout sodir bo'ladi: funksiya chaqirilmaydi, faqat child'lar clone qilinib `childLanes` bo'yicha pastga tushiladi. Layout memo'd bo'lishi bu yerda hal qiluvchi emas — App bailout qilgani uchun Layout'ga umuman yangi `children` reference yetib bormaydi.

**Haqiqatan funksiyasi qayta chaqiriladigan (render bo'ladigan) komponent:** faqat **Counter**.

**Descend qiladi-yu funksiyasi chaqirilmaydigan (bailout + clone):** Root, App, Layout, Main — `childLanes` set bo'lgani uchun child'larga tushadi, lekin o'zlari render bo'lmaydi.

**To'liq skip qilinadigan (subtree'ga umuman tushilmaydi):** Sidebar, Article — `lanes` ham, `childLanes` ham 0.

Bu — `childLanes` mexanizmining mohiyati: bitta chuqur komponentdagi `setState` butun yuqori tree'ni qayta render qilmaydi, balki faqat update bor branch bo'ylab tushib, yagona tegishli komponentni render qiladi. Bir komponentni "har doim parent bilan birga render bo'ladi" deyish noto'g'ri — agar parent funksiyasi bailout tufayli chaqirilmasa, child yangi props olmaydi va o'zi ham bailout qiladi. Optimization patterns `33-optimization.md` da chuqur.

</details>

---

### Mashq 5: List reorder optimization (Qiyin)

Quyidagi list `setItems([...items].reverse())` chaqirilganda qancha DOM mutation bo'ladi (keyed bilan)? (Eslatma: `items.reverse()` array'ni in-place mutate qiladi — state mutation anti-pattern; immutable variant ishlatiladi.)

```
Eski: [A, B, C, D, E]
Yangi: [E, D, C, B, A]
```

<details>
<summary><strong>Javob</strong></summary>

Greedy algoritm:

```
Faza 1: newIdx=0, key=E vs key=A → mos emas → faza 4
Faza 4: existingChildren = { A, B, C, D, E }

newIdx=0: E → topildi (oldIndex=4)
  lastPlacedIndex=0; oldIndex=4 >= 0 → no move, lastPlacedIndex=4

newIdx=1: D → topildi (oldIndex=3)
  lastPlacedIndex=4; oldIndex=3 < 4 → MOVE

newIdx=2: C → topildi (oldIndex=2)
  lastPlacedIndex=4; oldIndex=2 < 4 → MOVE

newIdx=3: B → topildi (oldIndex=1)
  lastPlacedIndex=4; oldIndex=1 < 4 → MOVE

newIdx=4: A → topildi (oldIndex=0)
  lastPlacedIndex=4; oldIndex=0 < 4 → MOVE
```

**4 ta MOVE** (Placement flag) — D, C, B, A.

LCS analiz:
- To'liq teskari list'da (`[A,B,C,D,E] → [E,D,C,B,A]`) LCS uzunligi = 1. Sabab: ikkala ketma-ketlikda ham tartibni saqlovchi juftlik yo'q — eski'da x dan keyin kelgan y, teskari list'da x dan oldin keladi. Demak uzunligi 2 bo'lgan umumiy subsequence mavjud emas; har qanday bitta element o'zicha uzunlik 1 beradi
- Optimal move count = 4 (n − LCS = 5 − 1 = 4): LCS'dagi bitta element joyida qoladi, qolgan 4 ta ko'chiriladi
- To'liq reversal'da greedy ham, optimal ham 4 ta move qiladi — bu yerda ular teng
- Greedy boshqa misol'larda suboptimal bo'ladi (masalan `[A,B,C,D] → [D,A,B,C]` — greedy 3 move, optimal 1 move)

Greedy bu holatda optimal natija bilan teng (4 move). Boshqa holatlarda greedy ko'proq move qilishi mumkin, lekin O(n) murakkablik O(n²) LCS'dan ustun — React jamoasi shu trade-off'ni tanlagan.

</details>

---

## Xulosa

Bu bo'limda Reconciliation algoritmining barcha qismlari yoritildi:

- **Reconciliation** — eski Fiber tree va yangi Element tree o'rtasida farq topish
- **2 ta heuristic** — different types = rebuild, keys = stable identity → O(n)
- **Type comparison** — `===` strict equality (`child.elementType === elementType`), function reference muhim
- **Sibling matching** — keyless (index-based, yomon) vs keyed (Map-based, yaxshi)
- **Key qoidalari** — unique, stable, predictable; index OK faqat statik list'da
- **Bailout 4 sabab** — element identity, React.memo shallow, useMemo/useCallback, state equality
- **Update propagation** — child'dan root'gacha lanes/childLanes, render root'dan boshlanadi
- **Greedy vs LCS** — React O(n) greedy tanlagan, ba'zi holatlarda suboptimal moves

Bu algoritmni tushunish keyingi bo'limlardagi optimization, memoization, va concurrent rendering xulq-atvorini oydinlashtiradi:
- **Scheduler** (`05-`) — lanes priority bilan render rejalashtirish
- **List rendering** (`08-`) — keys deep dive, virtualization
- **State** (`12-`) — bailout va re-render trigger'lari
- **Memoization** (`21-`) — useMemo/useCallback mexanikasi
- **Optimization** (`33-`) — bailout pattern'lar real qo'llanish
- **Rendering Behavior** (`32-`) — re-render trigger debugging

Keyingi bo'limda Scheduler va Lanes mexanizmi yoritiladi — Reconciler render'ini priority bilan boshqarish va concurrent rendering uchun zarur infrastructure.

---

**Keyingi bo'lim:** [05-scheduler-lanes.md](05-scheduler-lanes.md) — Scheduler & Lanes: concurrent rendering uchun nima kerak, Scheduler package, priority levels (Immediate/UserBlocking/Normal/Low/Idle), **Lanes model R18+** (bitmap of priority lanes), lane types (SyncLane, InputContinuousLane, DefaultLane, TransitionLane, IdleLane), time slicing, interruptible rendering, expiration (starvation prevention).
