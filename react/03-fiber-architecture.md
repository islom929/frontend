# Bo'lim 3: Fiber Architecture

> Fiber — React 16 (2017)'dan beri Reconciler'ning ichki tuzilmasi. Har Fiber **ikki rolga ega**: (1) **work unit** — Reconciler bo'lib-bo'lib bajaradigan ish; (2) **tree node** — komponent yoki host element'ning Reconciler ko'rinishidagi vakili. Bu bo'lim Fiber architecture'sining barcha qismlarini yoritadi: tag types, traversal pointers, double buffering, effect list optimization'i va Stack Reconciler bilan farqi.

---

## Mundarija

- [Fiber Nima](#fiber-nima)
- [Fiber Tree vs Component Tree](#fiber-tree-vs-component-tree)
- [Fiber Tag Types](#fiber-tag-types)
- [Tree Traversal Pointers](#tree-traversal-pointers)
- [Current vs WorkInProgress — Double Buffering](#current-vs-workinprogress--double-buffering)
- [Alternate Pointer](#alternate-pointer)
- [Why Fiber — Stack Reconciler bilan Farq](#why-fiber--stack-reconciler-bilan-farq)
- [Effect List va Subtree Flags](#effect-list-va-subtree-flags)
- [Fiber Lifecycle — Mount va Update](#fiber-lifecycle--mount-va-update)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Fiber Nima

### Nazariya

**Fiber** — React Reconciler'ning ichki **ish birligi** (work unit) va **tree node**'i. Har bir komponent (function yoki class) va har bir host element (DOM node) uchun React **bitta Fiber obyekti** yaratadi. Bu obyektlar bir-biriga bog'langan tree (linked tree) hosil qiladi va Reconciler shu tree bo'ylab yurib ish bajaradi.

**Ikki tomoni:**

1. **Work unit** — Reconciler ishni bo'lib-bo'lib bajarishi uchun atom birlik. Concurrent rendering'da React har Fiber'ni alohida ishlaydi va orasida browser'ga yo'l beradi (yield).
2. **Tree node** — Fiber tree'da bitta tugun. Komponent hierarchysini va render natijasini ifodalaydi.

**Asosiy maydonlar (Fiber struktura):**

| Maydon | Tur | Vazifa |
|--------|-----|--------|
| `tag` | `WorkTag` (number) | Fiber turi (FunctionComponent, HostComponent, va h.k.) |
| `key` | `string \| null` | List item identity (cross-ref `04-reconciliation.md`) |
| `elementType` | any | JSX'dagi original `type` (memo/forwardRef wrapper bilan) |
| `type` | any | Resolved type (memo'ning ichki komponenti) |
| `stateNode` | any | Class instance, DOM node, yoki FiberRoot |
| `return` | `Fiber \| null` | Ota Fiber (parent reference) |
| `child` | `Fiber \| null` | Birinchi farzand Fiber |
| `sibling` | `Fiber \| null` | Keyingi qardosh Fiber |
| `index` | number | Parent children array'idagi pozitsiya |
| `ref` | `Ref \| null` | Ref obyekt yoki callback |
| `pendingProps` | any | Yangi (kelgusi) props |
| `memoizedProps` | any | Oxirgi commit qilingan props |
| `updateQueue` | `UpdateQueue \| null` | Pending update'lar (setState chaqiruvlari) |
| `memoizedState` | any | Function: hooks linked list. Class: instance state |
| `dependencies` | `Dependencies \| null` | Context dep'lar (consumer fiber'lar uchun) |
| `mode` | `TypeOfMode` (number) | ConcurrentMode, StrictMode bayroqlari |
| `flags` | `Flags` (number) | Effect bayroqlari (Placement, Update, ChildDeletion, ...) |
| `subtreeFlags` | `Flags` (number) | Bu subtree'dagi barcha flag'larning OR |
| `deletions` | `Array<Fiber> \| null` | O'chiriladigan child fiber'lar ro'yxati |
| `lanes` | `Lanes` (number) | Pending update'larning priority lanes |
| `childLanes` | `Lanes` (number) | Subtree'da pending lane'lar |
| `alternate` | `Fiber \| null` | Boshqa tree'dagi juftlik (current ↔ workInProgress) |

> **Manba:** `react-reconciler/src/ReactInternalTypes.js`. Maydon tartibi va nomlari shu fayldan olingan; ba'zi DevTools-only va perf-tracking maydonlar (masalan, `actualDuration`, `_debugSource`) yuqoridagi jadvalga kiritilmagan.

**Bitta Fiber — bitta render unit:**

Reconciler render davomida har Fiber uchun ish bajaradi (komponent funksiyasini chaqirish, child Fiber'larni reconcile qilish). Bitta Fiber ish'i tugagandan keyin React **`shouldYield()`** ni tekshiradi — agar browser'ga yo'l berish vaqti kelgan bo'lsa, work loop to'xtatiladi va MessageChannel orqali keyinroq davom etadi. Bu mexanizm — interruptible rendering'ning asosi.

### Fiber yaratilishi

Birinchi render'da React komponent tree bo'ylab yurib har element uchun yangi Fiber yaratadi. Update'larda esa **mavjud Fiber qayta ishlatiladi** — `pendingProps` yangilanadi, `flags` o'rnatiladi, lekin Fiber obyekti yangidan yaratilmaydi (memory allocation kamaytiriladi).

```typescript
// React internal (soddalashtirilgan) — react-reconciler/src/ReactFiber.js
function FiberNode(tag, pendingProps, key, mode) {
  // Instance
  this.tag = tag;
  this.key = key;
  this.elementType = null;
  this.type = null;
  this.stateNode = null;

  // Fiber tree pointers
  this.return = null;
  this.child = null;
  this.sibling = null;
  this.index = 0;

  this.ref = null;

  this.pendingProps = pendingProps;
  this.memoizedProps = null;
  this.updateQueue = null;
  this.memoizedState = null;
  this.dependencies = null;

  this.mode = mode;

  // Effects
  this.flags = NoFlags;
  this.subtreeFlags = NoFlags;
  this.deletions = null;

  this.lanes = NoLanes;
  this.childLanes = NoLanes;

  this.alternate = null;
}

const createFiber = function (tag, pendingProps, key, mode) {
  return new FiberNode(tag, pendingProps, key, mode);
};
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Fiber yaratish rejimlari:**

React Fiber yaratishning ikkita asosiy yo'lini ishlatadi:

1. **`createFiberFromElement(element)`** — JSX element'idan yangi Fiber yaratish (mount paytida)
2. **`createWorkInProgress(current, pendingProps)`** — mavjud Fiber'ning **clone**'ini yaratish (update paytida)

`createWorkInProgress` muhim — u **alternate** Fiber'ni reuse qiladi. Birinchi update'da yangi clone yaratiladi va `current.alternate = workInProgress`, `workInProgress.alternate = current` o'rnatiladi. Keyingi update'larda esa shu alternate Fiber qayta ishlatiladi (yangi obyekt yaratilmaydi).

```typescript
// React internal (soddalashtirilgan)
function createWorkInProgress(current, pendingProps) {
  let workInProgress = current.alternate;
  
  if (workInProgress === null) {
    // Birinchi marta — yangi alternate yaratiladi
    workInProgress = createFiber(
      current.tag,
      pendingProps,
      current.key,
      current.mode
    );
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;
    
    // Ikki tomonlama bog'lanish
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    // Mavjud alternate'ni reuse — flag va props yangilash
    workInProgress.pendingProps = pendingProps;
    workInProgress.type = current.type;
    workInProgress.flags = NoFlags;
    workInProgress.subtreeFlags = NoFlags;
    workInProgress.deletions = null;
  }
  
  // Mavjud state'ni copy qilish
  workInProgress.lanes = current.lanes;
  workInProgress.childLanes = current.childLanes;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;
  workInProgress.updateQueue = current.updateQueue;
  
  return workInProgress;
}
```

Bu pattern — **memory allocation optimization'i**. Har render'da yangi obyekt yaratish (katta application'da o'n minglab Fiber bo'lishi mumkin) GC bosimini oshiradi. Reuse esa minimal allocation bilan ishlaydi.

**Fiber'ning V8 optimization'i:**

Fiber obyekti **monomorphic** struktura uchun mo'ljallangan — har Fiber bir xil maydon nomlari va tartibida o'rnatiladi. Bu V8'ga **bir xil hidden class** ishlatish imkonini beradi (cross-ref [`/js/01-js-engine.md`](../js/01-js-engine.md)). Property access optimization qilingan, polymorphism kamaytirilgan.

React source code'da (`react-reconciler/src/ReactFiber.js`) Fiber `FiberNode` **constructor function** orqali yaratiladi — `function FiberNode(tag, pendingProps, key, mode) { this.tag = tag; ... }`, va `createFiber` esa `new FiberNode(...)` chaqiradi. Constructor ichida barcha maydonlar **har doim bir xil tartibda** `this.field = ...` ko'rinishida o'rnatiladi. Aynan bu doimiy tartib V8 hidden class transitionlarini minimallashtiradi: har Fiber bitta umumiy hidden class'ga ega bo'ladi, shu sababli prototip yuqorida `createFiber` literal sifatida emas, constructor sifatida yozilgan.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Fiber strukturasini ko'rsatuvchi misol (mental model):

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

// Render natijasi (mental model — Fiber tree):
const counterFiber = {
  tag: 0, // FunctionComponent
  type: Counter,
  key: null,
  stateNode: null, // Function komponent uchun null
  
  // Hooks linked list memoizedState'da
  memoizedState: {
    next: null,
    memoizedState: 0,  // useState'ning 0 qiymati
    queue: { /* dispatch va update queue */ },
    baseState: 0,
  },
  
  pendingProps: {},
  memoizedProps: {},
  
  // Tree pointers
  child: buttonFiber,
  sibling: null,
  return: parentFiber,
  
  flags: 0, // NoFlags (initial render uchun Placement bo'lar edi)
  
  // ...
};

const buttonFiber = {
  tag: 5, // HostComponent
  type: 'button',
  stateNode: htmlButtonElement, // Real DOM node
  
  pendingProps: {
    onClick: /* ... */,
    children: 0,
  },
  
  child: textFiber,
  sibling: null,
  return: counterFiber,
  
  // ...
};

const textFiber = {
  tag: 6, // HostText
  stateNode: textNode, // DOM Text node
  pendingProps: '0',
  
  child: null,
  sibling: null,
  return: buttonFiber,
};
```

DevTools'da Fiber'ni ko'rish:

```tsx
// React DevTools'da har komponent — bitta Fiber'ga mos
// "Components" tab'da tree ko'rinadi:

// <App />              ← App Fiber (FunctionComponent)
//   <Header />          ← Header Fiber (FunctionComponent)
//     <h1>             ← h1 Fiber (HostComponent)
//       "Sarlavha"     ← Text Fiber (HostText)
//   <Main>             ← Main Fiber (FunctionComponent)
//     <Article />      ← Article Fiber (MemoComponent agar React.memo bo'lsa)
```

Production'da Fiber'lar visible emas (DevTools faqat dev'da ulanadi), ammo `__REACT_DEVTOOLS_GLOBAL_HOOK__` orqali tekshirish mumkin (advanced debugging).

</details>

---

## Fiber Tree vs Component Tree

### Nazariya

**Component tree** (application kodida yozilgan):

```jsx
<App>
  <Header />
  <Main>
    <Article />
  </Main>
</App>
```

Bu — **JSX ko'rinishi**. Faqat sizning komponentlaringiz ko'rinadi.

**Fiber tree** (Reconciler ichida):

Fiber tree **batafsilroq**. U ikki turdagi node'larni o'z ichiga oladi:
1. **Komponent fiber'lari** — sizning komponentlaringiz uchun (App, Header, Main, Article)
2. **Host fiber'lari** — komponent ichidagi DOM element'lari uchun (`<div>`, `<h1>`, `<p>`, va h.k.)

Misol:

```tsx
function Header() {
  return (
    <header>
      <h1>Sarlavha</h1>
    </header>
  );
}

function App() {
  return (
    <div>
      <Header />
      <p>Matn</p>
    </div>
  );
}
```

**Fiber tree** ko'rinishi:

```
HostRoot (#root)
  │
  └─ App (FunctionComponent)
       │
       └─ div (HostComponent)
            ├─ Header (FunctionComponent)
            │    └─ header (HostComponent)
            │         └─ h1 (HostComponent)
            │              └─ "Sarlavha" (HostText)
            └─ p (HostComponent)
                 └─ "Matn" (HostText)
```

Komponent fiber'lari (`App`, `Header`) — **render funksiyalar**ni saqlaydi va child host fiber'larini yaratadi. Host fiber'lari (`div`, `header`, `h1`, `p`) — **DOM node'larga reference**'ni saqlaydi va commit phase'da real DOM'ga commit qilinadi.

**Fragmentlar ham Fiber:**

```tsx
function List() {
  return (
    <>
      <li>Item 1</li>
      <li>Item 2</li>
    </>
  );
}

// Fiber tree:
// List (FunctionComponent)
//   └─ Fragment (Fiber tag = 7)
//        ├─ li (HostComponent)
//        └─ li (HostComponent)
```

Fragment — alohida Fiber tag (WorkTag = 7). DOM'ga hech narsa render qilmaydi, lekin Fiber tree'da haqiqiy tugun sifatida mavjud.

### Komponent vs Host fiber farqi

| Xususiyat | Komponent Fiber | Host Fiber |
|-----------|----------------|------------|
| `tag` | FunctionComponent (0), ClassComponent (1) | HostComponent (5), HostText (6) |
| `type` | Komponent funksiyasi/class | HTML tag string ('div', 'button') |
| `stateNode` | Class instance yoki null (function uchun) | Real DOM node |
| Render bajaradi mi? | Ha — komponent funksiyasi chaqiriladi | Yo'q — props DOM'ga qo'llaniladi |
| Hooks ishlatadi mi? | Ha (function komponent uchun) | Yo'q |
| Commit Phase | DOM'ga to'g'ridan tegmaydi | DOM mutation bajaradi |

<details>
<summary><strong>Under the Hood</strong></summary>

**Reconciler komponent vs host fiber'larni qanday farqlaydi:**

`reconcileChildren` chaqirilganda, har JSX element uchun React `createFiberFromElement` chaqiradi:

```typescript
// React internal (soddalashtirilgan)
function createFiberFromElement(element, mode, lanes) {
  const type = element.type;
  
  if (typeof type === 'function') {
    // Komponent
    if (shouldConstruct(type)) {
      return createFiberFromTypeAndProps(type, element.key, element.props, mode, lanes);
      // tag: ClassComponent
    } else {
      return createFiberFromTypeAndProps(type, element.key, element.props, mode, lanes);
      // tag: FunctionComponent
    }
  }
  
  if (typeof type === 'string') {
    // Host element ('div', 'button', ...)
    return createFiberFromHostInstance(type, element.key, element.props);
    // tag: HostComponent
  }
  
  if (type === REACT_FRAGMENT_TYPE) {
    return createFiberFromFragment(element.props.children, mode, lanes, element.key);
    // tag: Fragment
  }
  
  // ... va boshqa maxsus turlar (Suspense, Provider, ...)
}
```

`shouldConstruct(type)` — bu funksiya class komponent yoki function komponent ekanligini aniqlaydi. Class'lar `prototype.isReactComponent` flag'iga ega bo'ladi (`React.Component` extend qilingani uchun).

```typescript
function shouldConstruct(Component) {
  return !!(Component.prototype && Component.prototype.isReactComponent);
}
```

**Komponent fiber render qachon bajariladi:**

`beginWork(fiber)` chaqirilganda fiber'ning tag'iga qarab maxsus handler chaqiriladi:

```typescript
// React internal
function beginWork(current, workInProgress, renderLanes) {
  switch (workInProgress.tag) {
    case FunctionComponent:
      return updateFunctionComponent(current, workInProgress, ...);
    
    case ClassComponent:
      return updateClassComponent(current, workInProgress, ...);
    
    case HostComponent:
      return updateHostComponent(current, workInProgress, ...);
    
    case HostText:
      return updateHostText(current, workInProgress);
    
    case Fragment:
      return updateFragment(current, workInProgress, ...);
    
    case ContextProvider:
      return updateContextProvider(current, workInProgress, ...);
    
    case MemoComponent:
      return updateMemoComponent(current, workInProgress, ...);
    
    // ... va h.k.
  }
}
```

Har handler o'zining ish mantiqiga ega:
- `updateFunctionComponent` — komponent funksiyasini chaqiradi, hooks bajaradi
- `updateHostComponent` — props diff qiladi, host instance prepare qiladi
- `updateHostText` — text content yangilanadi
- `updateMemoComponent` — props shallow-equal bo'lsa, child render skip qilinadi (bailout)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Component tree va Fiber tree taqqoslash:

```tsx
// Sizning kodingiz:
function UserCard({ user }: { user: User }) {
  return (
    <article className="card">
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </article>
  );
}

function UserList() {
  const users = [
    { id: 1, name: 'Ali', email: 'ali@example.com' },
    { id: 2, name: 'Vali', email: 'vali@example.com' },
  ];
  
  return (
    <section>
      {users.map(u => <UserCard key={u.id} user={u} />)}
    </section>
  );
}
```

**Component tree (mental):**

```
UserList
├─ UserCard (key=1)
└─ UserCard (key=2)
```

**Fiber tree (haqiqiy ichki ko'rinish):**

```
UserList (FunctionComponent)
└─ section (HostComponent)
   ├─ UserCard (FunctionComponent, key=1)
   │  └─ article (HostComponent)
   │     ├─ h3 (HostComponent)
   │     │  └─ "Ali" (HostText)
   │     └─ p (HostComponent)
   │        └─ "ali@example.com" (HostText)
   └─ UserCard (FunctionComponent, key=2)
      └─ article (HostComponent)
         ├─ h3 (HostComponent)
         │  └─ "Vali" (HostText)
         └─ p (HostComponent)
            └─ "vali@example.com" (HostText)
```

E'tibor bering — Fiber tree komponent fiber'lar (`UserList`, `UserCard`) va host fiber'lar (`section`, `article`, `h3`, `p`) ni bir-biriga o'rab oladi. Komponent fiber'larida DOM element yo'q (`stateNode = null`), faqat host fiber'larida real DOM node bor.

Custom component'siz pure DOM:

```tsx
function App() {
  return (
    <div>
      <p>Salom</p>
    </div>
  );
}

// Fiber tree:
// HostRoot
//   └─ App (FunctionComponent)
//      └─ div (HostComponent) → <div> DOM node
//         └─ p (HostComponent) → <p> DOM node
//            └─ "Salom" (HostText) → Text node
```

</details>

---

## Fiber Tag Types

### Nazariya

Har Fiber'ning **`tag`** maydoni Fiber turini belgilaydi. Bu — number constant. React Reconciler `tag` ga qarab maxsus handler chaqiradi.

**Eng ko'p uchraydigan tag'lar:**

| Tag | Constant nomi | Fiber turi |
|-----|---------------|------------|
| 0 | `FunctionComponent` | Function komponent (`function MyComp() { return ... }`) |
| 1 | `ClassComponent` | Class komponent (`class MyComp extends React.Component`) |
| 2 | `IndeterminateComponent` | **R18 va undan oldin:** Hali aniqlanmagan (mount paytida class vs function aniqlanguncha). **R19'da olib tashlangan** — function komponent darhol `FunctionComponent` tag'i bilan yaratiladi |
| 3 | `HostRoot` | Tree'ning ildizi (`createRoot` natijasida) |
| 4 | `HostPortal` | Portal (`createPortal`) |
| 5 | `HostComponent` | DOM element (`<div>`, `<button>`, va h.k.) |
| 6 | `HostText` | Text node (string children) |
| 7 | `Fragment` | `<></>` yoki `<Fragment>` |
| 8 | `Mode` | `<StrictMode>` va boshqa mode wrapper |
| 9 | `ContextConsumer` | `<Context.Consumer>` (legacy) |
| 10 | `ContextProvider` | `<Context.Provider>` yoki R19 `<Context value>` |
| 11 | `ForwardRef` | `forwardRef(...)` natijasi |
| 12 | `Profiler` | `<Profiler>` |
| 13 | `SuspenseComponent` | `<Suspense>` |
| 14 | `MemoComponent` | `React.memo(Comp, compare)` — custom `compare` bilan |
| 15 | `SimpleMemoComponent` | `React.memo(Comp)` — `compare` berilmagan, fast path |
| 16 | `LazyComponent` | `React.lazy(...)` natijasi |

**R18-R19'da qo'shilgan yangi tag'lar:**

| Constant nomi | Vazifa |
|---------------|--------|
| `ActivityComponent` | `<Activity>` — content'ni "hidden" yoki "visible" rejimda DOM'dan olib tashlamasdan state'ni saqlab turish uchun. Avval eksperimental `<Offscreen>` (internal `OffscreenComponent` tag) deb nomlangan; public API `<Activity>` ko'rinishida React 19.2'da stable bo'ldi |
| `HostHoistable` | `<title>`, `<meta>`, `<link>` — head'ga hoisted host element'lar |
| `HostSingleton` | `<html>`, `<head>`, `<body>` — singleton host element'lar |

> **Eslatma:** Tag raqamlari React versiyasiga qarab o'zgaradi va internal API hisoblanadi. Manba: `react-reconciler/src/ReactWorkTags.js`. Yuqoridagi 0-16 raqamlar R18-R19'da barqaror; yangi tag'lar qo'shilganda raqamlar shifted bo'lishi mumkin. App kodda constant nom emas, **rasmiy React API'lar** bilan ishlash kerak — tag'lar to'g'ridan-to'g'ri ishlatilmaydi.

### Tag turlari va amaliy ahamiyat

**`FunctionComponent` (0)** — modern React'ning asosiy tag'i. Komponent funksiyasi har render'da chaqiriladi, hooks bajariladi.

**`HostComponent` (5)** — har JSX'dagi HTML tag uchun (`<div>`, `<button>`). `stateNode` real DOM node'ga ishora qiladi.

**`HostText` (6)** — string children uchun (masalan, `<p>Matn</p>` ichidagi "Matn"). Alohida Fiber sifatida saqlanadi — bu React'ga text node'larni alohida boshqarish imkonini beradi (DOM node identity).

**`Fragment` (7)** — DOM'ga hech narsa render qilmaydi. Tree'da virtual tugun. Fragmentlar `key` ola oladi (`<Fragment key={...}>` formatida) — list'larda foydali.

**`MemoComponent` (14)** va **`SimpleMemoComponent` (15)** — `React.memo()` natijasi. Fiber dastlab `MemoComponent` tag'i bilan yaratiladi; agar `compare` argument berilmagan va ichki komponent oddiy function bo'lsa, Reconciler uni `SimpleMemoComponent` fast path'ga downgrade qiladi (default shallow compare). Custom `compare` bilan `MemoComponent` saqlanadi. Har ikkala holatda bailout tekshiriladi — props equal bo'lsa, child render skip qilinadi.

**`ForwardRef` (11)** — `forwardRef()` natijasi. R19'da function component'lar uchun `ref` oddiy prop sifatida o'qiladi, shuning uchun yangi kodda `forwardRef` shart emas. `forwardRef` hozircha deprecated emas va ishlaydi; kelajakda deprecate qilish rejalashtirilgan.

**`SuspenseComponent` (13)** — `<Suspense>` boundary. Child Fiber promise throw qilsa, Suspense boundary fallback'ni ko'rsatadi va promise resolution'ni kutadi.

**`HostRoot` (3)** — tree'ning ildizi. `createRoot()` natijasi. `stateNode` `FiberRootNode`'ga ishora qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Tag aniqlanishi mexanikasi:**

JSX element ichida `type` maydoni — komponent funksiyasi, class, string, yoki maxsus React symbol'i bo'ladi:

```typescript
// React.createElement / jsx-runtime
function jsx(type, props, key) {
  return {
    $$typeof: REACT_ELEMENT_TYPE,
    type,           // <-- har xil bo'lishi mumkin
    key,
    props,
    // ...
  };
}
```

`type` qiymati va Fiber tag'i o'rtasida moslik:

| `type` qiymati | Fiber tag |
|----------------|-----------|
| `function MyComp() {...}` | `FunctionComponent` (0) |
| `class extends React.Component` | `ClassComponent` (1) |
| `'div'`, `'button'` (string) | `HostComponent` (5) |
| `string` (text children uchun) | `HostText` (6) |
| `React.Fragment` (Symbol) | `Fragment` (7) |
| `React.StrictMode` (Symbol) | `Mode` (8) |
| Object with `$$typeof: REACT_CONTEXT_TYPE` (R19) yoki `REACT_PROVIDER_TYPE` (R18) | `ContextProvider` (10) |
| Object with `$$typeof: REACT_FORWARD_REF_TYPE` | `ForwardRef` (11) |
| Object with `$$typeof: REACT_MEMO_TYPE` | `MemoComponent` (14) |
| Object with `$$typeof: REACT_LAZY_TYPE` | `LazyComponent` (16) |

`$$typeof` — React'ning maxsus marker'i (Symbol). U JSON injection attack'lar (XSS)'dan himoya qiladi: serializable JSON faqat string/number/object/array bo'la oladi, Symbol JSON'da ifodalanmaydi. Shu sababli untrusted JSON React Element'ga aylantirilolmaydi.

```typescript
// React.memo natijasi (mental model)
function memo(Component, compare) {
  return {
    $$typeof: REACT_MEMO_TYPE,
    type: Component,
    compare: compare === undefined ? null : compare,
  };
}
// Fiber dastlab har doim MemoComponent tag'i bilan yaratiladi.
// updateMemoComponent ichida: `compare === null` va inner komponent oddiy function bo'lsa,
// fiber.tag SimpleMemoComponent'ga downgrade qilinadi (default shallowEqual fast path).
// `compare !== null` bo'lsa — MemoComponent tag'i saqlanadi.

// React.forwardRef natijasi
function forwardRef(render) {
  return {
    $$typeof: REACT_FORWARD_REF_TYPE,
    render,
  };
}
```

Reconciler'da `createFiberFromElement` chaqirilganda `type.$$typeof`'ga qarab tegishli Fiber tag'i berilgan:

```typescript
// R19 source (soddalashtirilgan — manba: react-reconciler/src/ReactFiber.js)
function createFiberFromTypeAndProps(type, key, props, mode, lanes) {
  let fiberTag: WorkTag = FunctionComponent;  // R19'da default
  let resolvedType = type;

  if (typeof type === 'function') {
    if (shouldConstruct(type)) {
      fiberTag = ClassComponent;
    }
    // function bo'lsa — fiberTag FunctionComponent bo'lib qoladi
  } else if (typeof type === 'string') {
    fiberTag = HostComponent;
  } else {
    // Object types — $$typeof tekshirish
    switch (type.$$typeof) {
      case REACT_FRAGMENT_TYPE: fiberTag = Fragment; break;
      case REACT_MEMO_TYPE: fiberTag = MemoComponent; break;
      case REACT_FORWARD_REF_TYPE: fiberTag = ForwardRef; break;
      case REACT_CONTEXT_TYPE: fiberTag = ContextProvider; break;
        // R19: Context object o'zi Provider sifatida ishlatiladi.
        // REACT_PROVIDER_TYPE symbol R19'da olib tashlandi.
      case REACT_CONSUMER_TYPE: fiberTag = ContextConsumer; break;
      case REACT_LAZY_TYPE: fiberTag = LazyComponent; break;
      // ... va h.k.
    }
  }

  const fiber = createFiber(fiberTag, props, key, mode);
  fiber.elementType = type;
  fiber.type = resolvedType;
  fiber.lanes = lanes;
  return fiber;
}
```

**`IndeterminateComponent` versiya tarixi:**

R18 va undan oldin function komponent birinchi mount paytida `IndeterminateComponent` (tag 2) bilan yaratilardi. `beginWork`'da React komponent funksiyasini chaqirib, natijaga qarab tag'ni `FunctionComponent` yoki `ClassComponent`'ga o'zgartirardi (legacy `createClass` orqali yaratilgan komponent'larni tutib olish uchun).

R19'da bu legacy chek olib tashlandi — function komponent darhol `FunctionComponent` tag'i bilan yaratiladi. `IndeterminateComponent` constant'i `ReactWorkTags.js`'dan olib tashlandi; uning eski qiymati (2) qayta ishlatilmadi va qolgan tag'lar o'z raqamlarini saqlab qoldi (`ClassComponent` = 1, `HostRoot` = 3).

**HostHoistable va HostSingleton (R19):**

R19'da `<title>`, `<meta>`, `<link>` Document Metadata API uchun maxsus tag'lar qo'shildi:

```tsx
function ProductPage({ product }: { product: Product }) {
  return (
    <article>
      <title>{product.name}</title>      {/* HostHoistable */}
      <meta name="description" content={product.description} />  {/* HostHoistable */}
      <h1>{product.name}</h1>            {/* HostComponent */}
    </article>
  );
}

// React HostHoistable fiber'larini avtomatik <head>'ga ko'chiradi
// HostComponent o'rniga maxsus tag — chunki commit semantikasi farqli
```

`HostSingleton` esa `<html>`, `<head>`, `<body>` uchun. Bu element'lar har doim mavjud (singleton), shuning uchun React ularni qayta yaratmaydi — mavjud DOM node'ga attach qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Har xil tag'larni ishlatuvchi misol:

```tsx
import { memo, forwardRef, Suspense, lazy, createContext, Fragment } from 'react';

const ThemeContext = createContext('light');
const LazyComp = lazy(() => import('./LazyComp'));

const MemoButton = memo(function Button({ label }: { label: string }) {
  return <button>{label}</button>;
});

const InputWithRef = forwardRef<HTMLInputElement, { value: string }>(
  ({ value }, ref) => <input ref={ref} value={value} />
);

function App() {
  return (
    <ThemeContext.Provider value="dark">  {/* ContextProvider */}
      <Fragment>                            {/* Fragment */}
        <MemoButton label="Save" />         {/* MemoComponent */}
        <InputWithRef value="" ref={null} /> {/* ForwardRef */}
        <Suspense fallback={<div>Loading...</div>}>  {/* SuspenseComponent */}
          <LazyComp />                       {/* LazyComponent */}
        </Suspense>
      </Fragment>
    </ThemeContext.Provider>
  );
}

// Fiber tree:
// App (FunctionComponent)
//   └─ ThemeContext.Provider (ContextProvider)
//      └─ Fragment (Fragment)
//         ├─ MemoButton (MemoComponent)
//         │  └─ Button (FunctionComponent — memo'ning ichki komponent)
//         │     └─ button (HostComponent)
//         ├─ InputWithRef (ForwardRef)
//         │  └─ input (HostComponent)
//         └─ Suspense (SuspenseComponent)
//            └─ LazyComp (LazyComponent → keyin FunctionComponent)
```

DevTools'da bu tag'larni ko'rish:

React DevTools "Components" tab'ida har komponent o'z tagiga mos icon bilan ko'rinadi:
- Memo komponent — `Memo` prefiksi bilan
- ForwardRef komponent — `ForwardRef` prefiksi bilan
- Suspense — alohida ikonka

Production build'da DevTools hali ulanadi, lekin minifier (Terser, SWC) komponent funksiyalar nomlarini qisqartirsa, ba'zi komponent'lar "Anonymous" yoki bitta-ikki harfli nom bilan ko'rinadi. Real React production build'da nomlarni saqlash uchun `keep_fnames` (Terser) yoki `keep_names` (SWC) options'ni yoqish kerak.

`React.memo` natijasini tekshirish:

```tsx
import { memo } from 'react';

const Button = memo(function Button({ label }: { label: string }) {
  return <button>{label}</button>;
});

// Button — bu function emas, obyekt:
console.log(Button);
// {
//   $$typeof: Symbol(react.memo),
//   type: function Button() { ... },
//   compare: null,  // compare berilmagan → Reconciler default shallowEqual ishlatadi
// }

// Reconciler $$typeof'ni ko'rib MemoComponent fiber yaratadi
// (yoki SimpleMemoComponent agar compare === null)
```

</details>

---

## Tree Traversal Pointers

### Nazariya

Fiber tree'da har Fiber **uchta tree pointer**'iga ega:

| Pointer | Ma'no | Misol |
|---------|-------|-------|
| **`child`** | Birinchi farzand Fiber | `<div>` ning birinchi child element |
| **`sibling`** | Keyingi qardosh Fiber | `<li>`'larning ketma-ketligi |
| **`return`** | Ota Fiber (parent reference) | Child'dan parent'ga qaytish |

E'tibor bering — **`children` array YO'Q**. React tree'ni **linked list** sifatida saqlaydi. Bu — Reconciler'ning iterativ ish loop'i uchun zarur.

**Tree ko'rinishi:**

```tsx
function App() {
  return (
    <div>
      <h1>Sarlavha</h1>
      <p>Matn 1</p>
      <p>Matn 2</p>
    </div>
  );
}
```

**Pointer struktura:**

```
       App (FunctionComponent)
         │ child
         ▼
       div (HostComponent)
         │ child
         ▼
       h1 ──sibling──> p1 ──sibling──> p2
        │                │              │
       child           child          child
        ▼                ▼              ▼
       "Sarlavha"     "Matn 1"      "Matn 2"
       
       Har birining return App.div ga ishora qiladi:
       h1.return = div, p1.return = div, p2.return = div
       div.return = App, App.return = HostRoot
```

### Walk order — depth-first

React tree'ni **depth-first** tartibda yuradi:
1. Avval `child`'ga tushadi (eng pastga yetguncha)
2. Keyin `sibling`'lar bo'ylab yuradi
3. Sibling tugagach, `return` bilan parent'ga qaytadi va parent'ning `sibling`'iga o'tadi

**Walk order misoli:**

```
Tree:
       A
      / \
     B   E
    / \   \
   C   D   F

Walk: A → B → C → (back to B) → D → (back to A) → E → F → (back to A) → DONE
```

Iterative walk loop:

```typescript
function performUnitOfWork(fiber: Fiber): Fiber | null {
  // 1. beginWork — bu Fiber uchun ish
  let next = beginWork(fiber);
  
  if (next !== null) {
    // 2. Child bor — pastga tushish
    return next;
  }
  
  // 3. Child yo'q — completeWork va sibling/parent'ga qaytish
  let current = fiber;
  while (current !== null) {
    completeWork(current);
    
    if (current.sibling !== null) {
      return current.sibling;  // Sibling'ga o'tish
    }
    
    current = current.return;  // Parent'ga qaytish
  }
  
  return null;  // Tree tugadi
}
```

Bu iterative algoritm Stack Reconciler'dagi recursive call'lardan farq qiladi — har Fiber'dan keyin `shouldYield()` tekshirish mumkin (interruptibility).

### Index field — sibling'lar tartibi

Har Fiber'ning **`index`** maydoni ham bor — parent'ning children array'idagi pozitsiyasi (0, 1, 2, ...). Bu Reconciliation algoritmida key matching uchun ishlatiladi (cross-ref `04-reconciliation.md`).

<details>
<summary><strong>Under the Hood</strong></summary>

**Nima uchun linked list, array emas:**

Eski Stack Reconciler (pre-R16) recursive descent ishlatardi:

```typescript
// Stack Reconciler (soddalashtirilgan)
function renderTree(element) {
  const children = element.children || [];
  for (const child of children) {
    renderTree(child);  // ← recursive call
  }
}
```

Bu yondashuvda har komponent renderi JS call stack'da bitta frame'ni egallaydi. Tree katta bo'lsa — chuqur recursion, stack overflow xavfi. Eng muhimi — **uziluvchi emas**: agar yarmida browser'ga yo'l berish kerak bo'lsa — recursion'ni to'xtatib qayta tiklash imkoni yo'q (call stack'ni saqlay olmaysiz).

Fiber tree linked list bilan iterative algoritm ishlatadi:

```typescript
// Fiber Reconciler
let workInProgress: Fiber | null = root;

while (workInProgress !== null && !shouldYield()) {
  workInProgress = performUnitOfWork(workInProgress);
}

// Bu yerda biz to'xtab tursak — keyinroq qaytib boshlash mumkin
// `workInProgress` shunchaki saqlanib turadi (variable)
// Browser ishi tugagach, MessageChannel orqali davom etadi
```

JS call stack'ni saqlash o'rniga, "hozirgi joy" — bu shunchaki `workInProgress` o'zgaruvchisidagi Fiber. Uni saqlab, browser'ga yo'l berib, keyin shu joydan davom etish mumkin.

**`return` pointer — nima uchun "return" deb nomlangan:**

JavaScript'da function `return` qilganda, oldingi frame'ga qaytadi. Fiber tree'da `child` Fiber tugagach, `return` pointer orqali parent'ga "qaytamiz" — concept'ga mos.

`return` "parent" deb nomlanmagan, chunki **u har doim haqiqiy parent emas**. Misol — Portal:

```tsx
function App() {
  return (
    <div>
      <Modal>Content</Modal>  {/* Portal — boshqa DOM joyida */}
    </div>
  );
}

// Component tree:
// App
//   └─ div
//      └─ Modal (Portal)
//         └─ Content

// DOM tree:
// <div>...</div>
// <div id="modal-root">  <- Portal target
//   Content
// </div>

// Fiber tree (return pointer):
// Modal.child = Portal Fiber (HostPortal tag)
// Portal Fiber'ning DOM ichidagi parent'i — modal-root
// Lekin Fiber tree'da return — div fiber (App ichida)
// Portal — DOM tree'ni bo'lib oladi, lekin React tree'da Modal ichida
```

`return` — **Reconciler tree'da kim parent**? Bu DOM tree'da kim parent'dan farq qilishi mumkin (Portal holati).

**Tree traversal va effect propagation:**

`return` pointer effect propagation uchun ham ishlatiladi:

```typescript
// Fiber'da effect bo'lganda, parent'larga subtreeFlags propagate qilinadi
function bubbleProperties(completedWork: Fiber) {
  let newSubtreeFlags = NoFlags;
  
  // Child'lar va sibling'lar bo'ylab yurish
  let child = completedWork.child;
  while (child !== null) {
    newSubtreeFlags |= child.subtreeFlags;
    newSubtreeFlags |= child.flags;
    child = child.sibling;
  }
  
  completedWork.subtreeFlags = newSubtreeFlags;
}

// Bu funksiya `completeWork` davomida har Fiber uchun chaqiriladi
// Natijada root fiber'ning subtreeFlags butun tree'dagi effect'lar yig'indisi bo'ladi
```

`subtreeFlags` Commit Phase'da ishlatiladi — agar fiber'ning `(subtreeFlags & MutationMask) === 0` bo'lsa, butun subtree skip qilinadi (DOM mutation kerak emas).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Tree pointer'larni mental model bilan ko'rish:

```tsx
function List() {
  return (
    <ul>
      <li>Item 1</li>
      <li>Item 2</li>
      <li>Item 3</li>
    </ul>
  );
}

// Fiber tree va pointer'lar:

// List Fiber:
//   child → ul Fiber
//   sibling → null
//   return → parent (App yoki HostRoot)

// ul Fiber:
//   child → li1 Fiber
//   sibling → null
//   return → List Fiber

// li1 Fiber:
//   child → "Item 1" Text Fiber
//   sibling → li2 Fiber
//   return → ul Fiber

// li2 Fiber:
//   child → "Item 2" Text Fiber
//   sibling → li3 Fiber
//   return → ul Fiber

// li3 Fiber:
//   child → "Item 3" Text Fiber
//   sibling → null
//   return → ul Fiber
```

Walk order:

```
1. List          (beginWork)
2. ul            (beginWork)
3. li1           (beginWork)
4. "Item 1"      (beginWork, completeWork)
5. li1           (completeWork — child tugadi)
6. li2           (beginWork — sibling)
7. "Item 2"      (beginWork, completeWork)
8. li2           (completeWork)
9. li3           (beginWork — sibling)
10. "Item 3"     (beginWork, completeWork)
11. li3          (completeWork)
12. ul           (completeWork — child'lar tugadi)
13. List         (completeWork)
14. (root)       (completeWork — DONE)
```

Conditional rendering — Fragment va shartli child'lar:

```tsx
function ConditionalList({ items, showHeader }: Props) {
  return (
    <>
      {showHeader && <h1>Items</h1>}
      <ul>
        {items.map(item => <li key={item.id}>{item.name}</li>)}
      </ul>
    </>
  );
}

// showHeader=true bo'lsa Fiber tree:
// Fragment (root return)
//   ├─ h1 (sibling: ul)
//   └─ ul (sibling: null)
//      └─ li1 (sibling: li2 ...)

// showHeader=false bo'lsa:
// Fragment (root return)
//   └─ ul (sibling: null)
//      └─ li1 (sibling: li2 ...)

// Reconciler farqni aniqlaydi va h1 fiber uchun Deletion flag o'rnatadi
```

</details>

---

## Current vs WorkInProgress — Double Buffering

### Nazariya

React har vaqtda **ikki Fiber tree** bilan ishlaydi:

| Tree | Vazifa |
|------|--------|
| **`current`** | Hozirgi commit qilingan tree — ekranda ko'rinayotgan UI |
| **`workInProgress`** | Hozir qurilayotgan yangi tree — keyingi commit uchun tayyorlanmoqda |

Bu — **double buffering** texnikasi, kompyuter grafikasidan olingan g'oya. Render off-screen buffer'da bajariladi, tayyor bo'lganda ekranga "swap" qilinadi.

**Asosiy g'oya:**

```
Render Phase boshida:
  current tree → "displayed"
  workInProgress = createWorkInProgress(current) → clone yaratiladi
  
Render Phase davomida:
  workInProgress tree o'zgartiriladi (yangi props, hooks, child'lar)
  current tree DAXLSIZ (foydalanuvchi hali eski UI'ni ko'radi)
  
Commit Phase oxirida:
  fiberRoot.current = workInProgress  ← TREE SWAP
  Eski "current" endi "alternate" sifatida saqlanadi (keyingi render uchun)
```

**Nima uchun double buffering:**

1. **Atomic commit** — yangi tree to'liq tayyor bo'lguncha foydalanuvchi yarim holatni ko'rmaydi
2. **Concurrent rendering xavfsizligi** — render uzilib qolsa yoki tashlansa, current tree ta'sir ko'rmaydi
3. **Memory reuse** — eski Fiber'lar GC'ga jo'natilmaydi, balki keyingi render uchun reuse qilinadi
4. **Restart imkoniyati** — high-priority update kelganda workInProgress tashlanib, qaytadan boshlanishi mumkin

### Tree swap mexanikasi

Commit Phase'ning **Mutation sub-phase oxirida** `fiberRoot.current = workInProgress` o'rnatiladi:

```typescript
// React internal (soddalashtirilgan)
function commitRoot(root) {
  // 1. Before Mutation Phase
  commitBeforeMutationEffects(root);
  
  // 2. Mutation Phase
  commitMutationEffects(root);
  
  // ⬇️ TREE SWAP shu yerda bo'ladi
  root.current = root.finishedWork;
  
  // 3. Layout Phase
  commitLayoutEffects(root);
  
  // 4. Schedule passive effects
  scheduleCallback(NormalPriority, flushPassiveEffects);
}
```

Tree swap **Mutation va Layout o'rtasida** bo'ladi. Sabab:
- Mutation paytida DOM hali yangilanmoqda — `fiberRoot.current` `workInProgress`'ga o'rnatish noto'g'ri (yarim holat)
- Layout Phase boshlanganda DOM to'liq yangi — endi `current` ham yangi tree

`useLayoutEffect` ichidan `fiber.alternate` o'qilganda — **eski tree** ko'rinadi (chunki swap allaqachon bo'lgan, hozirgi `current` — yangi tree, alternate — eski).

<details>
<summary><strong>Under the Hood</strong></summary>

**FiberRootNode struktura:**

```typescript
type FiberRootNode = {
  containerInfo: any,        // DOM container ('#root' element)
  current: Fiber,            // Joriy commit qilingan tree root
  finishedWork: Fiber | null,  // Render tugagan, commit kutilayotgan tree
  
  // Update queue
  pendingLanes: Lanes,
  // ...va boshqa scheduling maydonlari
};

type Fiber = {
  // ...boshqa maydonlar
  alternate: Fiber | null,   // Boshqa tree'dagi juftlik
};
```

**Tree swap to'liq:**

```
BEFORE COMMIT:
  fiberRoot.current   → Tree A (old, displayed)
  fiberRoot.finishedWork → Tree B (new, completed render)
  
  Tree A root.alternate → Tree B root
  Tree B root.alternate → Tree A root

DURING MUTATION PHASE:
  DOM mutations applied (insertions, deletions, updates)
  Refs detached from old DOM nodes

AFTER MUTATION PHASE:
  fiberRoot.current = fiberRoot.finishedWork
  Tree B endi "current"
  Tree A endi "alternate"

DURING LAYOUT PHASE:
  Refs attached to new DOM nodes
  useLayoutEffect, componentDidMount/Update
  ⚠️ root.current.alternate === Tree A (old)

AFTER PAINT:
  passive effects (useEffect) fire
  ⚠️ root.current.alternate === Tree A (still old)

NEXT RENDER STARTS:
  workInProgress = createWorkInProgress(current)
  Bu Tree A'ni reuse qiladi (alternate'dan):
    workInProgress = current.alternate (Tree A)
    workInProgress.pendingProps = ...
    workInProgress.flags = NoFlags
  Tree A endi yangi "workInProgress" — ish boshlanadi.
```

**Memory profile:**

Har Fiber **ikki nusxada** mavjud (current va alternate). Lekin:
- Komponent'lar mount/unmount bo'lganda, alternate ham mos ravishda yaratiladi/o'chiriladi
- Fiber obyekti har bir node uchun kompakt struktura (asosiy maydonlar `ReactInternalTypes.js`'da Fiber type'ida ro'yxatlangan); katta application'da ham memory'da o'rinli
- `stateNode` ikki tree'da bir xil (DOM node bittagina) — duplikatsiya qilinmaydi

Demak, double buffering **bitta Fiber'ga ikki nusxa metadata** + **bitta DOM node** keltiradi — DOM node duplikatsiya qilinmagani uchun umumiy memory ortishi cheklangan (asosan Fiber struct'lari hisobiga).

**Concurrent rendering bilan integration:**

```typescript
function performWorkOnRoot(root: FiberRoot, lanes: Lanes) {
  // 1. workInProgress yaratish (module-level o'zgaruvchi)
  let workInProgress = createWorkInProgress(root.current, null);

  // 2. Render loop
  while (workInProgress !== null) {
    if (shouldYield()) {
      // High-priority update keldi yoki budget tugadi
      // workInProgress saqlanadi (module-level variable orqali)
      // Keyinroq qaytib davom etamiz
      return;
    }
    workInProgress = performUnitOfWork(workInProgress);
  }

  // 3. Render tugadi — tugallangan tree root'i current.alternate
  // (loop tugaganda lokal workInProgress null bo'ladi, shuning uchun
  //  finishedWork sifatida root.current.alternate olinadi)
  root.finishedWork = root.current.alternate;
  commitRoot(root);
}
```

Agar render uzilib qolsa:
- `current` tree daxlsiz — foydalanuvchi eski UI'ni ko'rishda davom etadi
- `workInProgress` xotirada saqlanadi
- Keyinroq render davom etadi (yoki tashlanadi)

Agar high-priority update kelsa va render tashlansa:
- `workInProgress` butun tree tashlanadi (alternate hali eski state'da)
- Yangi update uchun yangi `workInProgress` yaratiladi (`createWorkInProgress(current, ...)`)
- Eski ish butunlay yo'qoladi — shu sababli render pure bo'lishi shart (side effect bo'lsa, "yarim ish" tashlanib ketadi)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Double buffering visualizationsi (mental model):

```tsx
// Initial mount:
function App() {
  const [count, setCount] = useState(0);
  return <div>{count}</div>;
}

// Birinchi render:
//   current = null
//   workInProgress = quriladi (App, div, "0" Text)
// Commit:
//   current = workInProgress
//   workInProgress = null


// setCount(1) chaqirildi:
//   current = (App, div, "0")
//   workInProgress = createWorkInProgress(current)
//                  = (App', div', "0")  ← clone, alternate orqali
//   
//   Har Fiber uchun:
//     App.alternate = App'
//     App'.alternate = App
//     div.alternate = div'
//     div'.alternate = div
//
//   Render davomida:
//     App' funksiya chaqiriladi → count=1
//     div' yangilanadi (text "0" → "1")
//     workInProgress.flags |= Update
//
//   Commit:
//     Mutation: text node "0" → "1"
//     current = workInProgress  ← swap
//     Endi current = (App', div', "1")
//     Eski (App, div, "0") = alternate (reuse uchun)


// setCount(2) chaqirildi:
//   current = (App', div', "1")
//   workInProgress = createWorkInProgress(current)
//                  = (App, div, "1")  ← eski Tree A reuse qilinadi!
//   
//   Memory allocation YO'Q — eski Fiber obyektlari qayta ishlatilmoqda
//   Faqat pendingProps va flag'lar yangilanadi
```

`alternate` orqali eski tree'ga kirish (debugging uchun):

```tsx
function DebugFiber() {
  const ref = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    const node = ref.current;
    if (node === null) return;

    // Internal API — production'da ishlatish tavsiya qilinmaydi.
    // DOM node'ga React qo'shgan internal Fiber property dinamik key bilan saqlanadi.
    const internal = node as unknown as Record<string, { alternate: unknown }>;
    const fiberKey = Object.keys(internal).find(k => k.startsWith('__reactFiber$'));
    if (fiberKey === undefined) return;

    const fiber = internal[fiberKey];
    console.log('Current fiber:', fiber);
    console.log('Alternate (oldingi):', fiber.alternate);
  }, []);
  
  return <div ref={ref}>Hello</div>;
}

// __reactFiber$xxx — React internal property
// Production'da ishlatish noto'g'ri (API yo'q rasmiy)
// Faqat debugging va React DevTools uchun
```

Concurrent rendering — render tashlanganda current daxlsiz:

```tsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  function handleChange(value: string) {
    setQuery(value);
    
    startTransition(() => {
      // Heavy computation — render qilish ko'p vaqt oladi
      const filtered = expensiveFilter(allItems, value);
      setResults(filtered);
    });
  }

  return (
    <>
      <input value={query} onChange={e => handleChange(e.target.value)} />
      {isPending && <Spinner />}
      <ResultList items={results} />
    </>
  );
}

// Foydalanuvchi tezda yozayotganda:
// 1. "ab" — render boshlanadi (workInProgress)
// 2. "abc" — yangi update keldi
//    - Eski workInProgress TASHLANADI
//    - current daxlsiz (foydalanuvchi "ab" natijalarini ko'rishda davom etadi)
//    - Yangi workInProgress yaratiladi "abc" uchun
// 3. "abcd" — yana yangi update
//    - "abc" workInProgress tashlanadi
//    - Yangi workInProgress "abcd" uchun
//
// Bu mexanizm — concurrent rendering'ning asosi
// Agar workInProgress tashlanganda side effect bo'lsa — "yarim ish" yo'qolardi
// Shuning uchun render PURE bo'lishi shart
```

</details>

---

## Alternate Pointer

### Nazariya

**`alternate`** — Fiber'ning **boshqa tree'dagi juftligi**. Har Fiber'da bu pointer mavjud:

```typescript
fiber.alternate === otherTreeFiber
otherTreeFiber.alternate === fiber  // ikki tomonlama bog'lanish
```

**Qachon alternate bor:**
- Mount paytida (birinchi render): `current` = fiber, `alternate` = null (workInProgress hali yo'q)
- Birinchi update'dan keyin: alternate yaratiladi va doim mavjud bo'ladi (reuse qilinadi)

**Alternate'ning rollari:**

1. **Memory reuse** — yangi render boshlanganda yangi Fiber yaratish o'rniga alternate reuse qilinadi
2. **State diff** — `current.memoizedProps` vs `workInProgress.pendingProps` solishtirish
3. **Bailout detection** — `current.memoizedProps === workInProgress.pendingProps` (reference equality) yoki `memo`'da shallow-equal bo'lsa, render skip qilinadi. React hech qachon deep equal qilmaydi
4. **Effect tracking** — eski (current) vs yangi (workInProgress) state farqini aniqlash

### Alternate qachon yaratiladi

`createWorkInProgress(current, pendingProps)` chaqirilganda:

```typescript
function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {
  let workInProgress = current.alternate;
  
  if (workInProgress === null) {
    // Birinchi marta — yangi alternate
    workInProgress = createFiber(current.tag, pendingProps, current.key, current.mode);
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;
    
    // Bog'lanish
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    // Mavjud alternate — reuse
    workInProgress.pendingProps = pendingProps;
    // ...
  }
  
  return workInProgress;
}
```

### Alternate'dan o'qish

Concurrent rendering'da `alternate` orqali eski state'ni o'qish mumkin:

```typescript
function reconcileChildren(current, workInProgress, ...) {
  if (current === null) {
    // Mount — eski state yo'q
    workInProgress.child = mountChildFibers(...);
  } else {
    // Update — eski child'lar bilan diff qilish
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,  // ← eski tree'dan child
      newChildren
    );
  }
}
```

`current.child` — eski tree'ning child'lari (eski commit'da edi). Reconciler bu bilan yangi child'larni solishtirib, mos kelganlarini reuse qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Alternate va Fiber lifecycle:**

```
Mount:
  Fiber A yaratildi
  Fiber A.alternate = null
  Commit
  Fiber A — current

Update 1:
  Fiber A' yaratildi (alternate of A)
  Fiber A.alternate = Fiber A'
  Fiber A'.alternate = Fiber A
  Render workInProgress (Fiber A')
  Commit
  Fiber A' — current
  Fiber A — alternate

Update 2:
  Fiber A reuse qilinadi (alternate of A')
  Fiber A.pendingProps = new props
  Fiber A.flags = NoFlags
  Render workInProgress (Fiber A)
  Commit
  Fiber A — current
  Fiber A' — alternate

Update 3:
  Fiber A' reuse qilinadi (alternate of A)
  ...

Bu cycle har update'da takrorlanadi — yangi obyekt yaratilmaydi.
```

**Memory leak xavfi yo'q:**

Bir tomondan, alternate orqali eski tree'ni ham ushlab turamiz. Lekin GC bu tree'ni tozalamaydi, chunki bizga kerak (keyingi render uchun reuse).

Komponent unmount bo'lganda:
- Fiber `Deletion` flag bilan belgilanadi
- Mutation Phase'da DOM'dan o'chiriladi
- `effects` cleanup chaqiriladi
- Fiber obyekti — endi bog'lanmagan, GC oladi

`alternate` ham bog'liqlik orqali GC oladi (bog'lash to'xtatiladi).

**Alternate'ning DevTools'da ko'rinishi:**

React DevTools "Profiler" tab'i alternate orqali oldingi commit'larni saqlab, taqqoslash imkonini beradi:

```
Commit 1: Fiber tree A
  Fiber A1, A2, A3 ...

Commit 2: Fiber tree B (clone of A)
  Fiber A1.alternate = B1
  Fiber A2.alternate = B2
  ...

Profiler saqlaydi:
  prevFiber: A (eski tree)
  newFiber: B (yangi tree)
  diff: A1.memoizedProps vs B1.memoizedProps
        — qaysi props o'zgargan, qaysi yo'q
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Alternate via React internal API (debugging only):

```tsx
type FiberLike = {
  tag: number;
  type: unknown;
  memoizedProps: unknown;
  alternate: FiberLike | null;
};

function FiberInspector() {
  const ref = useRef<HTMLDivElement>(null);
  const [count, setCount] = useState(0);

  useEffect(() => {
    if (!ref.current) return;
    
    const internal = ref.current as unknown as Record<string, FiberLike>;
    const fiberKey = Object.keys(internal).find(k =>
      k.startsWith('__reactFiber$')
    );
    if (fiberKey === undefined) return;

    const fiber = internal[fiberKey];
    console.log('Current div fiber:', {
      tag: fiber.tag,
      type: fiber.type,
      memoizedProps: fiber.memoizedProps,
    });
    
    if (fiber.alternate) {
      console.log('Alternate (oldingi commit):', {
        memoizedProps: fiber.alternate.memoizedProps,
      });
    } else {
      console.log('Alternate yo\'q (birinchi commit)');
    }
  });

  return (
    <div ref={ref}>
      Count: {count}
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}

// Birinchi render:
// "Current div fiber: {...}"
// "Alternate yo'q (birinchi commit)"

// + bosildi (count=1):
// "Current div fiber: {memoizedProps: {children: 'Count: 1'}}"
// "Alternate (oldingi commit): {memoizedProps: {children: 'Count: 0'}}"

// + yana bosildi (count=2):
// "Current: {children: 'Count: 2'}"
// "Alternate: {children: 'Count: 1'}"  ← oldingi commit
```

> **⚠️ Eslatma:** `__reactFiber$xxx` — React internal property, kafolatlanmagan API. Production kodda ishlatish noto'g'ri. Faqat debugging va React DevTools internal'lari uchun. Yangi React versiyalarda nom o'zgarishi mumkin.

---

Mental model — alternate va memory:

```tsx
// Initial mount
//   current = Fiber A  (alternate: null)
//   workInProgress = Fiber A (clone YO'Q, mount uchun yangi)

// 1-update
//   workInProgress = Fiber A' (yangi clone yaratildi)
//   A.alternate = A'
//   A'.alternate = A
//   Commit:
//     current = A'
//     A' (yangi current)
//     A (alternate, reuse uchun saqlanadi)

// 2-update
//   workInProgress = A (reuse, yangi obyekt YO'Q)
//   A.pendingProps yangilanadi
//   Commit:
//     current = A
//     A (yangi current)
//     A' (alternate)

// 3-update
//   workInProgress = A' (reuse)
//   ...

// Memory'da har vaqtda 2 ta Fiber obyekti (current + alternate)
// Yangi memory allocation YO'Q (initial mount va birinchi update'dan keyin)
```

Profiler bilan alternate ko'rish:

```tsx
import { Profiler } from 'react';

function App() {
  return (
    <Profiler
      id="app"
      onRender={(id, phase, actualDuration, baseDuration, startTime, commitTime) => {
        console.log(`${id} ${phase} ${actualDuration.toFixed(2)}ms`);
        // phase: "mount" yoki "update"
      }}
    >
      <Counter />
    </Profiler>
  );
}

// Console (raqamlar har holatda farq qiladi — device va render hajmiga bog'liq):
// Mount: "app mount 1.23ms"
// Update: "app update 0.45ms"
//
// actualDuration — bu Profiler subtree'sini render qilish uchun
// real vaqt (commit'da measured). baseDuration — memoizationsiz holatdagi
// estimated vaqt. Ikkalasini taqqoslab memoization samarasini aniqlash mumkin.
```

</details>

---

## Why Fiber — Stack Reconciler bilan Farq

### Nazariya

React 16 (sentabr 2017)'gacha React **Stack Reconciler** ishlatardi. Stack Reconciler — recursive descent algoritmi: har komponent render JavaScript call stack'da bitta frame'ni egallaydi va to'liq synchronous ishlaydi.

**Stack Reconciler kamchiliklari:**

1. **Synchronous, uziluvchi emas** — render boshlanganidan keyin to'liq tugaguncha to'xtab bo'lmasdi
2. **Long render UI'ni bloklaydi** — uzun list yoki murakkab tree → input lag, animation jank
3. **Prioritization yo'q** — barcha update'lar bir xil priority'da
4. **Restartable emas** — yangi update kelganda eski ish davom etardi
5. **Stack overflow xavfi** — chuqur recursion JS engine'ning configured call stack chegarasidan oshib ketishi mumkin edi (har engine va platforma uchun stack size farq qiladi — V8'da `--stack-size` flag bilan boshqariladi); Stack Reconciler tree'ning chuqurligi bo'yicha har komponent uchun JS frame ishlatardi

**Fiber Reconciler (R16+) infrastructure yechimlari:**

1. **Iterative algoritm** — recursive descent o'rniga linked list bo'ylab while loop (har doim, sync va concurrent rendering uchun)
2. **Stack overflow yo'q** — JavaScript call stack yetarlicha sayoz (har `performUnitOfWork` — bitta frame); concurrent va sync rendering ikkalasida amal qiladi
3. **Concurrent infrastructure** — quyidagi xususiyatlar **faqat concurrent rendering'da** (R16-R17'da `unstable_AsyncMode` opt-in; R18+ `createRoot` bilan default):
   - **Interruptible** — `workLoopConcurrent` har Fiber'dan keyin `shouldYield()` tekshiradi va browser'ga yo'l beradi (sync render — `workLoopSync` — yo'l bermaydi)
   - **Priority lanes** — har update o'z priority'siga ega (cross-ref `05-scheduler-lanes.md`)
   - **Restartable** — high-priority update kelganda eski workInProgress tashlanib, yangi boshlanadi

### Real-world misol

**Eski Stack Reconciler — uzun list:**

```tsx
function ProductList({ products }: { products: Product[] }) {
  return (
    <ul>
      {products.map(p => <ComplexProductCard key={p.id} product={p} />)}
    </ul>
  );
}

// Katta list (masalan, o'n minglab product) — render uzoq vaqt davom etishi mumkin
// (aniq vaqt komponent murakkabligi, element soni va device CPU'ga bog'liq)
// Stack Reconciler:
// 1. handleClick chaqiriladi → setState
// 2. Render boshlanadi
// 3. Barcha ComplexProductCard'lar recursive render qilinadi
// 4. Render davom etgan vaqt davomida main thread bloklanadi:
//    - input event'lar javob bermaydi
//    - animation freeze
//    - tugma click navbatga turadi
// 5. Render tugadi → DOM yangilandi
```

**Fiber Reconciler — bir xil list (concurrent rendering'da):**

```tsx
// Bir xil komponent, lekin createRoot (R18+) va concurrent rendering bilan:
// 1. handleClick → setState
// 2. Render boshlanadi (`workLoopConcurrent`)
// 3. `frameYieldMs = 5` ms budget tugagach React `shouldYield()` true qaytaradi
//    (qiymat manbasi: scheduler/src/SchedulerFeatureFlags.js;
//     ishlatilishi: scheduler/src/forks/Scheduler.js — frameInterval = frameYieldMs)
//    - Browser'ga yo'l beriladi
//    - Input, animation javob beradi
// 4. Render bir necha bo'lakka taqsimlanadi, har 5 ms'dan keyin yield
// 5. Foydalanuvchi main thread'ni bloklanmagan deb sezadi (concurrent yield)

// Plus useTransition bilan:
const [isPending, startTransition] = useTransition();
function handleClick() {
  startTransition(() => {
    setProducts(newList);  // non-urgent — boshqa update'lar oldinda
  });
}
```

### Migration jarayoni

Fiber rewrite — React'ning eng katta texnik o'zgarishi. **2 yil davom etgan** (2015 boshlangach 2017 sentabrda release; manba: `acdlite/react-fiber-architecture` README — "over two years of research by the React team"). Asosiy nuance'lar:

- API o'zgarmadi — application kodi Fiber'ga o'tishda hech narsa yozilmagan
- Internal'lar to'liq qayta yozildi
- **`componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate`** — bu lifecycle method'lar `UNSAFE_` prefiksi bilan deprecated bo'ldi (concurrent rendering bilan mos emas — render uziluvchi bo'lgani uchun bir necha marta chaqirilishi mumkin)
- **R18'da concurrent features stable** bo'ldi (R16-R17'da hali experimental edi)

> **Eslatma:** Fiber rewrite'dan keyin **dastlabki R16'da concurrent rendering hali yoqilmagan edi** — faqat infrastructure tayyor edi. R17'gacha React har doim synchronous ishlardi (Fiber bo'lsa-da). R18'da `createRoot` bilan concurrent rendering aktiv bo'ldi. Demak, "Fiber + Concurrent" — ikki alohida bosqich.

<details>
<summary><strong>Under the Hood</strong></summary>

**Stack Reconciler implementation'i (mental model):**

```typescript
// Stack Reconciler (R15)
function reconcile(component) {
  const newElement = component.render();
  const oldElement = component._lastRendered;
  
  if (oldElement === null) {
    // Mount
    mountElement(newElement);
  } else {
    // Update
    diff(oldElement, newElement);
  }
  
  // Recursive — child'lar uchun
  for (const child of newElement.children) {
    if (child.isComponent) {
      reconcile(child);  // ← recursive call, JS stack'da frame
    }
  }
  
  component._lastRendered = newElement;
}

// Render boshlanganda — to'liq tugamaguncha to'xtab bo'lmaydi
// JavaScript synchronous nature'i tufayli
```

Bu yondashuv — to'liq synchronous, JS event loop'ni bloklaydi.

**Fiber Reconciler implementation'i (mental model):**

```typescript
// Fiber Reconciler (R16+)
let workInProgress: Fiber | null = root;

function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(fiber: Fiber): Fiber | null {
  // 1. beginWork — komponent renderi va child fiber'lar
  let next = beginWork(fiber);
  
  if (next !== null) {
    return next;  // Pastga tushish
  }
  
  // 2. Child yo'q — completeWork va sibling/parent
  let current: Fiber | null = fiber;
  while (current !== null) {
    completeWork(current);
    if (current.sibling !== null) return current.sibling;
    current = current.return;
  }
  
  return null;  // Tree tugadi
}

// Render davomida har `frameYieldMs` (=5ms) chamasi shouldYield() true qaytaradi:
function shouldYield(): boolean {
  return getCurrentTime() >= deadline;
}

// Yield bo'lganda:
// 1. workInProgress saqlanadi (variable)
// 2. workLoop to'xtaydi
// 3. JavaScript event loop davom etadi (input, animation)
// 4. Browser task tugagach, MessageChannel orqali React qayta uyg'onadi:
function performWorkUntilDeadline() {
  scheduler.scheduledHostCallback(true);
  // Yangi `frameYieldMs` budget bilan workLoop davom etadi
}
```

**MessageChannel ishlatish sababi:**

Eski versiya `requestIdleCallback` ishlatardi:
```typescript
window.requestIdleCallback(performWork);
```

Lekin Scheduler yozilgan paytda `requestIdleCallback`:
- Cross-browser support yetarli emas edi (Safari uni faqat 16.4'da, 2023'da qo'shdi)
- Chaqirilish chastotasi va vaqti deterministik emas — browser qachon "idle" deb hisoblashiga bog'liq, ko'p hollarda kerakli tezlikda chaqirilmaydi

`MessageChannel` ishlatish:
```typescript
const channel = new MessageChannel();
channel.port1.onmessage = performWork;

function scheduleWork() {
  channel.port2.postMessage(null);
}
```

`MessageChannel` har macrotask'dan keyin chaqiriladi — deterministik, har browser'da ishlaydi, kichik kechikish.

**Why Fiber — bitta jumla bilan:**

> "Fiber — bu **tree traversal'ni JavaScript call stack'dan ajratish** mexanizmi. Stack o'rniga linked list ishlatib, render'ni qism-qism bajarish va istagan vaqtda to'xtatib qaytib davom ettirish imkonini beradi."

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Pre-Fiber lifecycle methods (deprecated) — concurrent bilan mos emas:

```tsx
// ❌ UNSAFE_ prefiksli lifecycle methodlar — Fiber/Concurrent'da xavfli
class OldComponent extends React.Component {
  UNSAFE_componentWillMount() {
    // Bu method R16'da deprecated, lekin hali ishlaydi
    // Concurrent mode'da bir necha marta chaqirilishi mumkin
    // (render uziluvchi va qayta boshlanishi mumkin)
    this.fetchData();  // ❌ side effect, bir necha marta yuz beradi
  }
  
  UNSAFE_componentWillReceiveProps(nextProps: Props) {
    // ❌ Render davomida props o'zgarishi
    // Concurrent'da bir necha marta chaqirilishi mumkin
    if (nextProps.id !== this.props.id) {
      this.fetchData();
    }
  }
  
  UNSAFE_componentWillUpdate(nextProps: Props, nextState: State) {
    // ❌ Render uzilsa, update tashlanishi mumkin — yarim ish
    document.title = `Updating: ${nextProps.title}`;
  }
}

// ✅ Modern alternativalar:
class NewComponent extends React.Component {
  componentDidMount() {
    // ✅ Commit Phase'da — commit har doim atomic
    this.fetchData();
  }
  
  componentDidUpdate(prevProps: Props) {
    // ✅ Commit Phase'da — yangi DOM bilan
    if (prevProps.id !== this.props.id) {
      this.fetchData();
    }
  }
}

// Yoki Hooks bilan (kursning asosiy yo'nalishi):
function ModernComponent({ id }: { id: number }) {
  useEffect(() => {
    fetchData(id);
  }, [id]);
  
  return <div>...</div>;
}
```

Concurrent rendering ko'rgazma — tashlanuvchi render:

```tsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  function handleChange(value: string) {
    setQuery(value);  // urgent — input darhol javob beradi
    
    startTransition(() => {
      // Heavy render — interruptible bo'lishi shart
      const filtered = expensiveFilter(allItems, value);
      setResults(filtered);
    });
  }

  return (
    <div>
      <input
        value={query}
        onChange={e => handleChange(e.target.value)}
      />
      {isPending && <p>Yangilanmoqda...</p>}
      <ul>
        {results.map(item => <li key={item.id}>{item.name}</li>)}
      </ul>
    </div>
  );
}

// Foydalanuvchi tezda yozadi: "a", "ab", "abc"
//
// Stack Reconciler bo'lganda:
// 1. "a" — render boshlanadi, sync (Stack Reconciler), input bloklanadi
// 2. "ab" — keyingi keystroke kutib turadi; yangi render input javobini kechiktiradi
// 3. "abc" — har keystroke render'ni boshlaydi, oldingisi tugamasa interruption yo'q
// → Foydalanuvchi main thread bloklangan deb sezadi (input/animation javob bermaydi)
//
// Fiber Reconciler + useTransition:
// 1. "a" — render boshlanadi (low priority)
// 2. "ab" — input urgent, "a" render TASHLANADI
//    Yangi render "ab" uchun
// 3. "abc" — yana tashlanadi
//    Yangi render "abc" uchun
// 4. Foydalanuvchi to'xtagach — render tugaydi va natija ko'rinadi
// → UI har doim javob beradi (input darhol updates)
```

</details>

---

## Effect List va Subtree Flags

### Nazariya

**Effect** — Commit Phase'da bajarilishi kerak bo'lgan ish: DOM mutation, ref attach/detach, lifecycle callback, va h.k. Reconciler render davomida har Fiber uchun mos **flag**'ni o'rnatadi:

| Flag | Constant nomi | Ma'no |
|------|---------------|-------|
| bit 0 | `PerformedWork` | Bu Fiber render qilingan |
| bit 1 | `Placement` | Yangi DOM node insert qilish kerak |
| bit 2 | `Update` | Mavjud DOM node yangilash kerak |
| bit 4 | `ChildDeletion` | Child Fiber o'chirish kerak |
| bit 5 | `ContentReset` | Text content reset kerak |
| bit 6 | `Callback` | Callback chaqirish (componentDidMount, va h.k.) |
| bit 9 | `Ref` | Ref attach/detach |
| bit 10 | `Snapshot` | getSnapshotBeforeUpdate |
| bit 11 | `Passive` | useEffect (passive flag) |

> **Eslatma:** Bit pozitsiyalari React versiyasiga qarab o'zgaradi (manba: `react-reconciler/src/ReactFiberFlags.js`). Yuqoridagi tartib mental model uchun — aniq bit qiymatlarini emas, **bitmask printsipi**'ni eslab qoling. Real source code'da `flags` 32-bit bitmask sifatida ifodalangan.

**Bitmask afzalligi:** bitta number'da bir nechta flag (`fiber.flags = Update | Ref` — ikki flag bitta operationda).

### Effect List — eski yondashuv (R16)

R16'da effect list **singly-linked list** sifatida tutilardi:

```typescript
type Fiber = {
  // ...
  firstEffect: Fiber | null,
  lastEffect: Fiber | null,
  nextEffect: Fiber | null,
};
```

`completeWork` davomida har Fiber uchun, agar effect bo'lsa, parent'ning effect list'iga qo'shilardi. Commit Phase'da bu list bo'ylab yurib har Fiber uchun effect bajarilardi.

**Kamchiliklar:**
- Har Fiber'da 3 ta qo'shimcha pointer (memory)
- List boshqarish murakkab (insert, remove)
- Parent'larning effect list'lari tree state'ga qarab quriladi (tortib chiqadi)

### Subtree Flags — yangi yondashuv (R17+)

R17'dan boshlab effect list optimization'i qo'llandi: **`subtreeFlags`** maydoni qo'shildi.

```typescript
type Fiber = {
  flags: Flags,         // O'z effect bayroqlari
  subtreeFlags: Flags,  // Bu subtree'dagi barcha flag'larning OR
};
```

`subtreeFlags` — bu subtree'da hech bo'lmasa bitta Fiber'da shu flag bormi yo'qmi degan tezkor tekshirish. `completeWork`'da har Fiber `subtreeFlags`'ni child'larining flags + subtreeFlags'idan hisoblaydi:

```typescript
function bubbleProperties(fiber: Fiber) {
  let newSubtreeFlags = NoFlags;
  
  let child = fiber.child;
  while (child !== null) {
    newSubtreeFlags |= child.subtreeFlags;
    newSubtreeFlags |= child.flags;
    child = child.sibling;
  }
  
  fiber.subtreeFlags = newSubtreeFlags;
}
```

Commit Phase'da Reconciler `subtreeFlags`'ni tekshirib, hech qanday effect bo'lmagan subtree'larni butunlay skip qiladi:

```typescript
function commitMutationEffects(root, finishedWork) {
  // Root'dan boshlab DFS
  let nextEffect = finishedWork;
  
  while (nextEffect !== null) {
    if ((nextEffect.subtreeFlags & MutationMask) !== NoFlags) {
      // Bu subtree'da mutation effect bor — pastga tushish
      nextEffect = nextEffect.child;
    } else {
      // Bu subtree'da hech narsa yo'q — skip qilamiz
      // Sibling'ga yoki parent'ga qaytamiz
      while (nextEffect !== null) {
        if (nextEffect.flags !== NoFlags) {
          commitEffect(nextEffect);  // O'z flag'i bo'lsa — commit
        }
        if (nextEffect.sibling !== null) {
          nextEffect = nextEffect.sibling;
          break;
        }
        nextEffect = nextEffect.return;
      }
    }
  }
}
```

**Optimization foydasi:**

Katta tree (masalan, mingdan ortiq Fiber)'da faqat bir nechta komponentda effect bo'lsa:

- **Eski yondashuv (R16 — effect list pointers):** har Fiber tekshiriladi, lekin alohida linked list (`firstEffect → lastEffect`) ishlatilardi
- **Yangi yondashuv (R17+ `subtreeFlags`):** parent'da `(subtreeFlags & MutationMask) === 0` bo'lsa, butun subtree skip qilinadi — faqat effect bor branch'lar visit qilinadi

Traversal hajmi effect tarqalishiga bog'liq: agar effect'lar tree'da kam joyda to'plangan bo'lsa, skip ko'p bo'ladi.

Bu optimization `subtreeFlags` orqali — boshqa state, "alohida data structure" kerak emas.

<details>
<summary><strong>Under the Hood</strong></summary>

**Bitmask manipulation:**

```typescript
// Flag'lar (R18 source kodidan, soddalashtirilgan)
const NoFlags         = 0b0000000000000000;
const Placement       = 0b0000000000000010;
const Update          = 0b0000000000000100;
const ChildDeletion   = 0b0000000000010000;
const Callback        = 0b0000000001000000;
const Ref             = 0b0000001000000000;
const Snapshot        = 0b0000010000000000;
const Passive         = 0b0000100000000000;

// Flag o'rnatish
fiber.flags |= Placement;          // Placement qo'shish
fiber.flags |= Update | Ref;       // Update va Ref qo'shish

// Flag tekshirish
if (fiber.flags & Placement) { ... }  // Placement borligini tekshirish
if (fiber.flags & (Update | Ref)) { ... }  // Update yoki Ref bormi

// Flag o'chirish
fiber.flags &= ~Placement;  // Placement olib tashlash

// Flag mask'lar (R19 source — react-reconciler/src/ReactFiberFlags.js)
const MutationMask =
  Placement | Update | ChildDeletion | ContentReset |
  Ref | Hydrating | Visibility | FormReset;
const LayoutMask = Update | Callback | Ref | Visibility;
const PassiveMask = Passive | Visibility | ChildDeletion;

// Tekshirish
if (fiber.subtreeFlags & MutationMask) { ... }
// Bu subtree'da Placement yoki Update yoki ChildDeletion bormi
```

**Subtree flags propagation:**

```
Tree:
       App (subtreeFlags=?)
      /              \
   Header           Main (subtreeFlags=?)
   (flags=0)       /        \
                  Article    Sidebar
                  (flags=Update)  (flags=0)

completeWork bo'ylab pastdan tepaga:

Article: flags=Update, subtreeFlags=0 (no children)
Sidebar: flags=0, subtreeFlags=0
Main:    flags=0, subtreeFlags = Article.flags | Article.subtreeFlags
                                | Sidebar.flags | Sidebar.subtreeFlags
                              = Update
Header:  flags=0, subtreeFlags=0
App:     flags=0, subtreeFlags = Header.flags | Header.subtreeFlags
                                | Main.flags  | Main.subtreeFlags
                              = Update

Commit Phase boshlanganda:
- App tekshiriladi: subtreeFlags & MutationMask → Update bor → pastga tushish
- Header tekshiriladi: subtreeFlags & MutationMask → 0 → SKIP butun subtree
- Main tekshiriladi: subtreeFlags & MutationMask → Update bor → pastga tushish
  - Article tekshiriladi: flags & MutationMask → Update bor → commitUpdate
  - Sidebar tekshiriladi: subtreeFlags=0 → SKIP

Article uchun bitta DOM mutation bajariladi.
Header va Sidebar subtree'lari butunlay tekshirilmaydi.
```

**`useEffect` skip optimization'i:**

`PassiveMask` flag bilan ham bir xil mantiq — agar subtree'da hech qanday `useEffect` yo'q bo'lsa, butun subtree passive effect phase'da skip qilinadi.

Bu optimization `React.memo`'siz ham ishlaydi — agar komponent'da `useEffect` yo'q va child'larida ham yo'q bo'lsa, passive phase'da bu Fiber tegmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Effect flag'lar — qaysi kod qaysi flag'ni o'rnatadi:

```tsx
function Demo() {
  const [count, setCount] = useState(0);
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    console.log('Effect');
  }, [count]);

  useLayoutEffect(() => {
    console.log('Layout effect');
  }, []);

  return (
    <div ref={ref}>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}

// Initial mount — Demo Fiber flag'lari:
// - Placement (yangi insert qilinmoqda)
// - Ref (ref attach kerak)
// - Passive (useEffect bor — passive phase scheduling)
// - Update (useLayoutEffect bor — Layout phase scheduling uchun)
// NOTE: Initial mount'da Update flag faqat layout effect (yoki ref) tufayli o'rnatiladi,
// "props bor" sababli emas — chunki yangi mount'da props "diff" yo'q (eski holat yo'q).

// Update (count o'zgargan):
// - Update (text node yangilanmoqda)
// - Passive (useEffect deps o'zgargan — qayta ishga tushish)
```

`subtreeFlags` mental model:

```tsx
function App() {
  return (
    <div>
      <Header />          {/* effect yo'q */}
      <UserProfile />     {/* useEffect bor */}
      <Footer />          {/* effect yo'q */}
    </div>
  );
}

function Header() {
  return <h1>Title</h1>;
}

function UserProfile() {
  useEffect(() => {
    console.log('User loaded');
  }, []);
  return <div>User</div>;
}

function Footer() {
  return <footer>©</footer>;
}

// Mount paytida flag'lar:
// App.flags = Placement
// App.subtreeFlags = Placement | Passive (UserProfile dan)
//
// Header.flags = Placement
// Header.subtreeFlags = 0 (no children effects)
//
// UserProfile.flags = Placement | Passive
// UserProfile.subtreeFlags = 0
//
// Footer.flags = Placement
// Footer.subtreeFlags = 0

// Commit Phase'da Passive Mask traversal:
// App.subtreeFlags & Passive → bor → pastga
// Header.subtreeFlags & Passive → 0 → SKIP butun subtree
// UserProfile.flags & Passive → bor → useEffect schedule qilinadi
// Footer.subtreeFlags & Passive → 0 → SKIP butun subtree
```

`flags` ni DevTools'da ko'rish (debugging):

```tsx
// React DevTools "Profiler" tab'ida "Why did this commit?" deb yozilgan
// — har Fiber'ning flags asosida tushuntirish:

// Fiber: <ProductCard>
// Why: "props o'zgardi" → Update flag
//
// Fiber: <UserList>
// Why: "state o'zgardi" → Update flag + Placement (yangi item)
//
// Fiber: <Sidebar>
// Why: "render skipped" → flags=0, subtreeFlags=0 → bailout
```

</details>

---

## Fiber Lifecycle — Mount va Update

### Nazariya

Fiber'ning hayot davri uch holatdan iborat:

| Holat | Nima bo'ladi |
|-------|--------------|
| **Mount** | Yangi Fiber yaratiladi, DOM'ga insert qilinadi, mount effect'lari ishga tushadi |
| **Update** | Mavjud Fiber qayta ishlatiladi (alternate orqali), props yangilanadi, update effect'lari ishga tushadi |
| **Unmount** | Fiber tree'dan o'chiriladi, cleanup'lar bajariladi, alternate ham GC oladi |

**Mount jarayoni:**

```
1. JSX element React.createElement orqali yaratiladi
2. Reconciler bu element uchun yangi Fiber yaratadi (createFiberFromElement)
3. Fiber tree'ga insert qilinadi (parent.child yoki sibling pointer'lari yangilanadi)
4. Komponent funksiyasi chaqiriladi (mount path)
   - Hooks mount path: mountState, mountEffect, va h.k.
   - memoizedState linked list quriladi
5. Child Fiber'lar yaratiladi (recursive 1-4)
6. Placement flag o'rnatiladi (Commit'da DOM'ga insert)
7. Commit Phase:
   - Mutation: parent.appendChild(child)
   - Layout: useLayoutEffect, refs attach, componentDidMount
   - Paint
   - Passive: useEffect
```

**Update jarayoni:**

```
1. setState chaqirilgan (yoki props o'zgargan)
2. Reconciler workInProgress yaratadi (alternate'dan reuse)
3. Komponent funksiyasi qayta chaqiriladi (update path)
   - Hooks update path: updateState, updateEffect
   - memoizedState linked list bo'ylab yurish (state, deps, callback solishtirish)
4. Yangi JSX vs eski child Fiber'lar diff qilinadi (Reconciliation)
5. Flag'lar o'rnatiladi (Update, ChildDeletion, va h.k.)
6. Commit Phase:
   - Mutation: changedProperty yangilash
   - Layout: useLayoutEffect cleanup → setup, refs re-attach
   - Paint
   - Passive: useEffect cleanup → setup
```

**Unmount jarayoni:**

```
1. Reconciliation paytida komponent yo'qolgani aniqlanadi
2. ChildDeletion flag parent Fiber'ga o'rnatiladi
3. deletions array'iga unmount qilinadigan Fiber qo'shiladi
4. Commit Phase, Mutation sub-phase:
   - Cleanup'lar chaqiriladi (useEffect cleanup, componentWillUnmount)
   - DOM'dan o'chiriladi (parent.removeChild)
   - Refs detach (ref.current = null)
5. Fiber obyekti endi bog'lanmagan — GC oladi
6. Alternate ham bog'lanmaydi — u ham GC oladi
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Mount path vs Update path — komponent funksiya darajasida:**

React komponent funksiyasini chaqirishdan oldin **dispatcher** ni o'rnatadi. Dispatcher — hooks implementation'i (mount yoki update path).

```typescript
// React internal
const HooksDispatcherOnMount = {
  useState: mountState,
  useEffect: mountEffect,
  useRef: mountRef,
  // ...
};

const HooksDispatcherOnUpdate = {
  useState: updateState,
  useEffect: updateEffect,
  useRef: updateRef,
  // ...
};

function renderWithHooks(current, workInProgress, Component, props, ...) {
  // Dispatcher tanlash
  if (current === null || current.memoizedState === null) {
    ReactCurrentDispatcher.current = HooksDispatcherOnMount;
  } else {
    ReactCurrentDispatcher.current = HooksDispatcherOnUpdate;
  }
  
  // Komponent funksiyasini chaqirish
  const children = Component(props);
  
  // Dispatcher'ni o'chirish (boshqa joyda hooks chaqirilmasin uchun)
  ReactCurrentDispatcher.current = ContextOnlyDispatcher;
  
  return children;
}
```

`useState` aslida `ReactCurrentDispatcher.current.useState(initial)` chaqiruvi:

```typescript
// React'da useState
function useState<S>(initialState: S | (() => S)) {
  const dispatcher = ReactCurrentDispatcher.current;
  return dispatcher.useState(initialState);
  // Mount: mountState(initialState) → yangi hook obyekti yaratiladi
  // Update: updateState() → mavjud hook obyektidan yangi qiymat olinadi
}
```

Bu mexanizm — Hooks'ning "render davomida ishlatish" va "call order matters" qoidasining asosi (cross-ref `15-hooks-fundamentals.md`).

**Unmount va alternate cleanup:**

```typescript
// React internal — fiber unmount
function commitDeletionEffects(root, parentFiber, deletedFiber) {
  // 1. Cleanup'lar chaqirish (useEffect cleanup, refs detach)
  commitDeletionEffectsOnFiber(root, parentFiber, deletedFiber);
  
  // 2. DOM'dan o'chirish
  removeChild(parentInstance, deletedFiber.stateNode);
  
  // 3. Fiber pointer'larni tozalash
  detachFiberAfterEffects(deletedFiber);
}

function detachFiberAfterEffects(fiber: Fiber) {
  const alternate = fiber.alternate;
  if (alternate !== null) {
    // Alternate'ni ham detach qilish
    detachFiberAfterEffects(alternate);
  }
  
  // Pointer'larni null qilish — Fiber GC uchun unreachable bo'lsin
  fiber.child = null;
  fiber.sibling = null;
  fiber.return = null;
  fiber.dependencies = null;
  fiber.memoizedProps = null;
  fiber.memoizedState = null;
  fiber.pendingProps = null;
  fiber.stateNode = null;
  fiber.updateQueue = null;
  fiber.alternate = null;
}
```

Bu cleanup muhim — Fiber tree katta bo'lsa, eski Fiber'lar bog'liq bo'lib qolib memory leak keltirishi mumkin. React bu cleanup'ni har unmount'da qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Mount, Update, Unmount log misol:

```tsx
function ChildComponent({ id }: { id: number }) {
  console.log(`Render: ${id}`);
  
  useEffect(() => {
    console.log(`Mount/Update effect: ${id}`);
    return () => console.log(`Cleanup: ${id}`);
  }, [id]);
  
  useEffect(() => {
    console.log(`Mount only: ${id}`);
    return () => console.log(`Unmount: ${id}`);
  }, []);
  
  return <div>{id}</div>;
}

function Parent() {
  const [items, setItems] = useState([1, 2, 3]);
  
  return (
    <>
      <button onClick={() => setItems([1, 2, 3, 4])}>+</button>
      <button onClick={() => setItems([1, 2])}>-</button>
      <button onClick={() => setItems(items.map(x => x + 10))}>shift</button>
      {items.map(id => <ChildComponent key={id} id={id} />)}
    </>
  );
}

// Initial mount: items=[1,2,3]
// "Render: 1", "Render: 2", "Render: 3"
// "Mount/Update effect: 1", "Mount/Update effect: 2", "Mount/Update effect: 3"
// "Mount only: 1", "Mount only: 2", "Mount only: 3"

// "+" bosildi: items=[1,2,3,4]
// "Render: 4" (yangi child mount)
// "Mount/Update effect: 4" (id deps yangi child uchun)
// "Mount only: 4" (mount)
// (1, 2, 3 — re-render bo'lmaydi, key bir xil, props o'zgarmagan)

// "-" bosildi: items=[1,2]
// "Cleanup: 3"  (eski Mount/Update effect cleanup)
// "Unmount: 3" (Mount only cleanup)
// (3-child Fiber unmount, DOM'dan o'chirildi)

// "shift" bosildi: items=[11,12]
// (Reconciliation bo'yicha key=1 va key=11 farqli — eski 1, 2 unmount, yangi 11, 12 mount)
// "Cleanup: 1", "Cleanup: 2"
// "Unmount: 1", "Unmount: 2"
// "Render: 11", "Render: 12"
// "Mount/Update effect: 11", "Mount/Update effect: 12"
// "Mount only: 11", "Mount only: 12"
```

Mount path vs Update path Hooks bilan:

```tsx
function Counter() {
  console.log('Komponent funksiyasi');
  
  // Mount: mountState(0) — yangi hook obyekti yaratiladi
  // Update: updateState() — mavjud hook obyektidan o'qiladi
  const [count, setCount] = useState(0);
  
  // Mount: mountEffect(() => ..., [count])
  //   Effect obyekti yaratiladi
  //   memoizedState linked list'ga qo'shiladi
  // Update: updateEffect(() => ..., [count])
  //   Mavjud effect topiladi
  //   deps solishtiriladi (Object.is)
  //   Agar farq bo'lsa — update flag o'rnatiladi
  useEffect(() => {
    console.log('Effect:', count);
    return () => console.log('Cleanup:', count);
  }, [count]);
  
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// Mount log:
// "Komponent funksiyasi" (1-render Strict Mode'da 2 marta)
// (commit + paint)
// "Effect: 0"

// + bosildi (count=1):
// "Komponent funksiyasi"
// (commit + paint)
// "Cleanup: 0"  ← oldingi effect cleanup
// "Effect: 1"   ← yangi effect setup
```

</details>

---

## Edge Cases va Gotchas

### Function komponent'da `stateNode` null

```tsx
function MyComp() {
  return <div>...</div>;
}

// Fiber:
// {
//   tag: 0, // FunctionComponent
//   stateNode: null, // ← har doim null function uchun
//   memoizedState: { /* hooks linked list */ },
// }
```

**Sabab:** Function komponent'lar instance'siz. State hooks linked list'da `memoizedState` ichida saqlanadi. `stateNode` faqat class komponent (instance) yoki host fiber (DOM node) uchun ishlatiladi.

---

### Fragment va `key`

```tsx
{items.map((item, i) => (
  <Fragment key={item.id}>
    <h3>{item.title}</h3>
    <p>{item.text}</p>
  </Fragment>
))}
```

`<Fragment key>` — `key` (yoki React 19'gacha bo'lgan `<Fragment>` faqat `key` prop) qabul qila oladigan yagona "virtual" Fiber. `<>...</>` shorthand syntax esa hech qanday attribute, jumladan `key` ham qabul qilmaydi — shorthand'da prop yozish uchun joy yo'q:

```tsx
// ❌ <>...</> shorthand'ga key qo'yib bo'lmaydi — attribute uchun sintaksis yo'q
{items.map(item => (
  <key={item.id}>...</>   // bunaqa yozuv umuman valid JSX emas
))}

// ✅ key kerak bo'lsa — to'liq <Fragment> syntax
import { Fragment } from 'react';
{items.map(item => (
  <Fragment key={item.id}>...</Fragment>
))}
```

---

### `alternate` initial mount'da yo'q

```tsx
// Mount paytida:
// fiber.alternate === null (hali workInProgress yo'q edi)

// Birinchi update'dan keyin:
// fiber.alternate !== null (clone yaratildi, ikki tomonlama bog'lanish)
```

**Amaliy ta'sir:** Reconciler `current === null` (mount) va `current !== null` (update) farqini shu orqali aniqlaydi:

```typescript
function reconcileChildren(current, workInProgress, ...) {
  if (current === null) {
    // MOUNT — eski child'lar yo'q
  } else {
    // UPDATE — eski child'lar bilan diff
  }
}
```

---

### Reconciler turli tag'lar uchun farqli xulq-atvor

```tsx
function App() {
  const [count, setCount] = useState(0);
  return (
    <>
      <RegularChild count={count} />          {/* har count o'zgarishida re-render */}
      <MemoChild count={count} />              {/* shallow check */}
      <MemoChild constantProp="never changes" /> {/* re-render skip */}
    </>
  );
}

const MemoChild = React.memo(function MemoChild(props) {
  return <div>{Object.values(props).join('')}</div>;
});
```

**Sabab:** `MemoComponent` tag'i `beginWork`'da maxsus handler chaqiradi — props shallow-equal bo'lsa, child render qilinmaydi (bailout). `RegularChild` (FunctionComponent) — har doim render qilinadi.

---

### `flags` o'rnatilmaydi — bailout bo'lsa

```tsx
const Memo = React.memo(function Memo({ value }) {
  return <div>{value}</div>;
});

function Parent({ data }) {
  return <Memo value={data.fixed} />;
  // data.fixed o'zgarmasa — Memo Fiber bailout
  // beginWork tashlanadi (komponent funksiyasi chaqirilmaydi)
  // flags=0, subtreeFlags=0 (no effects)
  // Commit Phase'da bu Fiber subtree butunlay skip qilinadi
}
```

**Optimization:** `subtreeFlags` orqali butun subtree commit'da skip qilinadi — Reconciler shu Fiber'ga umuman tegmaydi.

---

## Common Mistakes

### ❌ Xato 1: Fiber tree'ni "lightweight DOM copy" deb tushunish

Eski hujjatlarda Fiber "lightweight Virtual DOM" deb tasvirlangan. Bu **noto'g'ri framing**:

- Fiber **tree node + work unit** (ikki rolda)
- DOM copy emas — ko'plab qo'shimcha metadata bor (hooks state, effect flags, lanes, alternate)
- Memory'da Fiber tree DOM tree'dan **kattaroq** — har Fiber ko'p maydonga ega (`flags`, `lanes`, `child`/`sibling`/`return` pointers, `memoizedState`, `pendingProps`, va h.k. — to'liq ro'yxat `ReactInternalTypes.js`'da)

**To'g'ri framing:** Fiber — Reconciler'ning ichki ish strukturasi. DOM emas, lekin DOM bilan parallel ishlovchi tree.

---

### ❌ Xato 2: Render davomida `__reactFiber$xxx` ga kirish

```tsx
function Bad() {
  const ref = useRef<HTMLDivElement>(null);
  
  // ❌ Render davomida internal API
  if (ref.current) {
    const fiber = (ref.current as any)[Object.keys(ref.current).find(...)];
    fiber.memoizedState = { custom: 'value' };  // ❌ Mutation
  }
  
  return <div ref={ref} />;
}
```

**Sabab:** `__reactFiber$xxx` — internal property, kafolatlanmagan API. Property nom React versiyasiga qarab o'zgaradi. Mutation qilish — React state'ni buzadi, behavior unpredictable bo'ladi.

**To'g'ri:** Faqat React rasmiy API'larini ishlatish (hooks, refs, props).

---

### ❌ Xato 3: Tag raqamlariga ishonish

```tsx
// ❌ Hardcoded tag number
if (fiber.tag === 0) { /* FunctionComponent */ }

// React versiya o'zgarganda raqam o'zgarishi mumkin (kamdan-kam, lekin)
// Yangi tag'lar qo'shilganda numbering ham o'zgarishi mumkin
```

**To'g'ri:** Bu ishni amalda qilmaslik kerak — internal API. Lekin mental model uchun constant nom bilan o'ylash:

```typescript
// React internal'da
import { FunctionComponent, HostComponent } from 'react-reconciler/src/ReactWorkTags';
// Bunday import bizga taqiq — internal package
```

---

### ❌ Xato 4: Stack Reconciler'ning `componentWillMount` shart deb hisoblash

```tsx
// ❌ Concurrent rendering bilan mos emas — UNSAFE_ prefiks bilan deprecated
class OldStyle extends React.Component {
  UNSAFE_componentWillMount() {  // R16.3'dan UNSAFE_ prefiksi majburiy
    this.fetchData();
  }
}
```

**Sabab:** `componentWillMount` Render Phase'da chaqiriladi (Commit'dan oldin). Concurrent rendering paytida render uziluvchi va qayta boshlanishi mumkin — fetch bir necha marta ishga tushishi mumkin. Shu sababli R16.3'dan boshlab `UNSAFE_` prefiksi bilan deprecated qilingan.

**To'g'ri:** `componentDidMount` (Commit Phase'da, atomic) yoki `useEffect` (function komponent).

---

### ❌ Xato 5: Fiber tree'ni Component tree bilan adashtirish

```tsx
// "Bu komponent N marta render qilinadi" deb hisoblash
function App() {
  return (
    <div>
      <Header />
      <Main />
    </div>
  );
}

// Component tree: 3 ta komponent (App, Header, Main)
// Fiber tree: 4+ Fiber (App, div, Header, Main, va h.k.)
```

**To'g'ri framing:** Performance tahlilida Fiber soni komponent sonidan ko'p (host fiber'lar ham bor). DevTools "Components" tab'i komponent tree'ni ko'rsatadi (sodda framing), lekin internal'da Fiber tree'ning fragmentlari.

---

## Amaliy Mashqlar

### Mashq 1: Fiber tag aniqlash (Oson)

Quyidagi JSX uchun har element'ning Fiber tag'ini ayting:

```tsx
import { memo, forwardRef, Suspense, lazy, Fragment, createContext } from 'react';

const ThemeContext = createContext('light');
const Lazy = lazy(() => import('./Comp'));
const Memo = memo(function MemoComp() { return <span>memo</span>; });
const Forward = forwardRef<HTMLDivElement>((props, ref) => <div ref={ref}>fwd</div>);

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Fragment>
        <Memo />
        <Forward />
        <Suspense fallback={<p>...</p>}>
          <Lazy />
        </Suspense>
        <h1>Title</h1>
      </Fragment>
    </ThemeContext.Provider>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

| Element | Fiber tag |
|---------|-----------|
| `App` | `FunctionComponent` (0) |
| `<ThemeContext.Provider>` | `ContextProvider` (10) |
| `<Fragment>` | `Fragment` (7) |
| `<Memo />` | `MemoComponent` (14) yoki `SimpleMemoComponent` (15) |
| `MemoComp` (Memo ichida) | `FunctionComponent` (0) |
| `<span>` (memo'da) | `HostComponent` (5) |
| `<Forward />` | `ForwardRef` (11) |
| `<div>` (Forward ichida) | `HostComponent` (5) |
| `<Suspense>` | `SuspenseComponent` (13) |
| `<Lazy />` | `LazyComponent` (16) → resolved'da `FunctionComponent` |
| `<h1>` | `HostComponent` (5) |
| `"Title"` | `HostText` (6) |

</details>

---

### Mashq 2: Tree traversal walk order (O'rta)

Quyidagi tree uchun `performUnitOfWork` qaysi tartibda chaqiriladi (begin/complete order)?

```
       A
      / \
     B   D
    /   / \
   C   E   F
```

<details>
<summary><strong>Javob</strong></summary>

```
beginWork(A)
  → A.child = B
beginWork(B)
  → B.child = C
beginWork(C)
  → C.child = null
completeWork(C)
  → C.sibling = null, parent = B
completeWork(B)
  → B.sibling = D
beginWork(D)
  → D.child = E
beginWork(E)
  → E.child = null
completeWork(E)
  → E.sibling = F
beginWork(F)
  → F.child = null
completeWork(F)
  → F.sibling = null, parent = D
completeWork(D)
  → D.sibling = null, parent = A
completeWork(A)
  → DONE
```

Walk: A(begin), B(begin), C(begin), C(complete), B(complete), D(begin), E(begin), E(complete), F(begin), F(complete), D(complete), A(complete).

</details>

---

### Mashq 3: subtreeFlags optimization (O'rta)

Quyidagi tree'da har Fiber'ning `flags` va `subtreeFlags` ni ko'rsating (initial mount uchun, Placement flag'i bilan):

```
App
├─ Header
├─ Main
│  ├─ Article (useEffect bor)
│  └─ Sidebar
└─ Footer
```

<details>
<summary><strong>Javob</strong></summary>

```
App.flags = Placement
App.subtreeFlags = Placement | Passive (Article'dan ko'tariladi)

Header.flags = Placement
Header.subtreeFlags = 0

Main.flags = Placement
Main.subtreeFlags = Placement | Passive

Article.flags = Placement | Passive
Article.subtreeFlags = 0

Sidebar.flags = Placement
Sidebar.subtreeFlags = 0

Footer.flags = Placement
Footer.subtreeFlags = 0
```

`subtreeFlags` `completeWork` davomida pastdan tepaga bubble qilinadi — har parent o'z child'larining `flags | subtreeFlags`'ini OR bilan birlashtiradi.

Commit Phase, Passive Mask traversal:
- App.subtreeFlags & Passive → bor → pastga tushish
- Header.subtreeFlags & Passive → 0 → SKIP
- Main.subtreeFlags & Passive → bor → pastga
  - Article.flags & Passive → bor → useEffect schedule
  - Sidebar.subtreeFlags & Passive → 0 → SKIP
- Footer.subtreeFlags & Passive → 0 → SKIP

Faqat Article uchun useEffect bajariladi. Header, Sidebar, Footer subtree'lari tekshirilmaydi.

</details>

---

### Mashq 4: Double buffering — alternate aniqlash (Qiyin)

Komponent 3 marta render qilingan. Har render'dan keyin `current` va `alternate` qaysi tree'ga ishora qiladi?

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  return <div>{count}</div>;
}

// Mount: count=0
// 1-update: count=1
// 2-update: count=2
// 3-update: count=3
```

Har bosqichda `fiberRoot.current.child` (Counter Fiber) `count`'ni qaysi qiymat bilan saqlaydi va `alternate.memoizedState` qaysi qiymat?

<details>
<summary><strong>Javob</strong></summary>

```
Mount (count=0):
  current Fiber A: memoizedState = 0
  A.alternate = null

1-update (count=1):
  workInProgress = createWorkInProgress(A) → Fiber A' yaratildi
  A.alternate = A'
  A'.alternate = A
  Render: A'.memoizedState = 1
  Commit: current = A'
  A' (current): memoizedState = 1
  A (alternate): memoizedState = 0

2-update (count=2):
  workInProgress = createWorkInProgress(A') → A reuse qilinadi
  Render: A.memoizedState = 2
  Commit: current = A
  A (current): memoizedState = 2
  A' (alternate): memoizedState = 1

3-update (count=3):
  workInProgress = createWorkInProgress(A) → A' reuse qilinadi
  Render: A'.memoizedState = 3
  Commit: current = A'
  A' (current): memoizedState = 3
  A (alternate): memoizedState = 2
```

Pattern: `alternate` har doim **oldingi commit qiymatini** saqlaydi. `current` — joriy commit. Har update'da ikki Fiber rolini almashtiradi (toggle).

</details>

---

### Mashq 5: Stack vs Fiber farq (Qiyin)

Quyidagi komponent Stack Reconciler (R15) va Fiber Reconciler (R16+) da qanday boshqacha ishlardi? Concurrent rendering R18 bilan qanday yana yaxshilanadi?

```tsx
function HeavyList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map(item => <ComplexCard key={item.id} item={item} />)}
    </ul>
  );
}

// items.length === 10000
// Faraz qilaylik: ComplexCard har biri ma'lum vaqt render qiladi
// va jami render device CPU'siga qarab uzoq cho'ziladi
// (aniq raqamlar har holatda farq qiladi — bu yerda nisbiy taqqoslash uchun)
```

<details>
<summary><strong>Javob</strong></summary>

**Stack Reconciler (R15):**
- Render to'liq synchronous — render davomida JS thread bloklanadi
- Foydalanuvchi: input javob bermaydi, animation freeze, click navbatga turadi
- Recursive descent — 10,000 ta `ComplexCard` recursive funksiya chaqiruvi
- Stack overflow xavfi (juda chuqur tree bo'lsa)

**Fiber Reconciler (R16, sync rendering hali):**
- Render hali synchronous (Concurrent yo'q edi R16-R17'da)
- Iterative algoritm — call stack overflow xavfi yo'q
- Bloklash hali ham bor, lekin tuzilish jihatidan concurrent uchun tayyor

**Fiber + Concurrent (R18+, useTransition bilan):**
```tsx
function Page() {
  const [items, setItems] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();
  
  function loadItems(newItems: Item[]) {
    startTransition(() => {
      setItems(newItems);  // non-urgent
    });
  }
  
  return (
    <>
      {isPending && <Spinner />}
      <HeavyList items={items} />
    </>
  );
}
```

- Render har `frameYieldMs` (=5ms, manba: `scheduler/src/SchedulerFeatureFlags.js`) chamasi to'xtaydi (`shouldYield`)
- Browser'ga yo'l beriladi — input/animation javob beradi
- Foydalanuvchi main thread bloklanmagan deb sezadi (concurrent yield)
- Yangi update kelsa (boshqa setState), eski render TASHLANADI
- Faqat oxirgi to'liq tugagan render commit qilinadi

**Reconciler bosqichlari taqqoslash:**

| Reconciler | JS bloklash davomiyligi | UI javob | Stack overflow xavfi |
|------------|------------------------|----------|---------------------|
| Stack (R15) | Render to'liq tugaguncha | YO'Q | Bor |
| Fiber sync (R16-17) | Render to'liq tugaguncha | YO'Q | Yo'q |
| Fiber concurrent (R18+) | `frameYieldMs` (≈5ms) har chunk | Bor (input darhol javob) | Yo'q |

</details>

---

## Xulosa

Bu bo'limda Fiber architecture'sining barcha asosiy qismlari yoritildi:

- **Fiber** — work unit + tree node ikki rolda
- **Fiber tag types** — har komponent/element turi uchun maxsus handler (FunctionComponent, HostComponent, MemoComponent, va h.k.)
- **Tree traversal** — child/sibling/return pointer'lar bilan iterative DFS walk
- **Double buffering** — `current` va `workInProgress` ikki tree, atomic commit
- **Alternate pointer** — ikki tree'dagi Fiber'lar juftligi, memory reuse uchun
- **Why Fiber** — Stack Reconciler'ning synchronous, uziluvchi emasligi muammosini hal qildi
- **subtreeFlags optimization'i (R17+)** — effect bo'lmagan subtree'larni Commit'da skip
- **Fiber lifecycle** — mount/update/unmount jarayonlari, hooks dispatcher swap

Bu architecture keyingi bo'limlarning asosi:
- **Reconciliation** (`04-`) — Fiber tree'larni qanday diff qiladi
- **Scheduler & Lanes** (`05-`) — Fiber render'ini priority bilan boshqarish
- **Hydration** (`06-`) — server HTML'ni Fiber tree'ga ulash
- **Hooks** (`15-`) — `memoizedState` linked list mexanizmi

Keyingi bo'limda Fiber tree'larini diff qiluvchi Reconciliation algoritmi yoritiladi.

---

**Keyingi bo'lim:** [04-reconciliation.md](04-reconciliation.md) — Reconciliation Algorithm: O(n) heuristics, type comparison, sibling matching (keyless vs keyed), key-based identity, **bailout algorithm** (4 sabab: element identity, React.memo, useMemo/useCallback, state equality), update propagation root → leaf.
