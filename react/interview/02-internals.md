# React Internals — Interview Savollari

> Fiber Architecture, Reconciliation Algorithm, Scheduler & Lanes, Hydration — kursning **yuragi**. Modern React'ning ichki mexanizmlari, V8/JS engine darajasida.
>
> Senior'lar uchun majburiy — har savolda **Deep Dive** chuqur tushuntirish bilan.

---

## Mundarija

- [**QISM A: Fiber Architecture** (savollar 1-13)](#qism-a)
- [**QISM B: Reconciliation Algorithm** (savollar 14-25)](#qism-b)
- [**QISM C: Scheduler & Lanes** (savollar 26-34)](#qism-c)
- [**QISM D: Hydration** (savollar 35-42)](#qism-d)

---

## QISM A: Fiber Architecture

<a id="qism-a"></a>

### 1. Fiber nima va Stack Reconciler bilan farqi nimada? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Fiber** (R16+) — React'ning rendering algoritm va data structure'i. Har komponent/element uchun **Fiber node** yaratiladi va ular linked list bilan bog'lanadi. **Stack Reconciler** (R15 va undan oldin) — recursive algoritm, JS call stack'da ishlardi, **interrupt qilib bo'lmasdi**. Fiber — interruptible (yield qila oladi), prioritized (lanes), restartable.

### To'liq tushuntirish

**Stack Reconciler muammolari (pre-R16):**

1. **Recursive** — har komponent reconcile chaqiruvi yangi stack frame
2. **Synchronous** — to'xtatib bo'lmaydi, oxirigacha ishlaydi
3. **Long task** — katta tree (1000+ komponent) — ko'p ms, browser bloklanadi
4. **Janky UI** — animation drop, input lag, scroll stutter
5. **No prioritization** — kichik update va katta update bir xil treatment

**Fiber yechimi (R16+):**

1. **Linked list traversal** — `child`/`sibling`/`return` pointers, recursion yo'q
2. **Work loop** — `while (nextUnitOfWork && !shouldYield()) performUnitOfWork()`
3. **Time slicing** — 5ms chunk'lar, browser'ga yield
4. **Priority lanes** — high pri (input) low pri (background)'ni interrupt qiladi
5. **Restartable** — render phase pure → kerak bo'lsa qayta boshlanadi

### Kod misoli

```tsx
// Fiber node strukturasi (sodda)
type Fiber = {
  // Identity
  tag: WorkTag;          // FunctionComponent (0), ClassComponent (1), HostComponent (5), ...
  type: any;             // 'div' string yoki Component function/class
  key: string | null;

  // Tree linked list
  child: Fiber | null;    // first child
  sibling: Fiber | null;  // next sibling
  return: Fiber | null;   // parent

  // Double buffering
  alternate: Fiber | null;

  // State
  memoizedState: any;     // hooks linked list
  memoizedProps: any;
  pendingProps: any;
  updateQueue: any;

  // Effects
  flags: Flags;
  subtreeFlags: Flags;

  // Scheduling
  lanes: Lanes;
  childLanes: Lanes;
};

// JSX
function App() {
  return (
    <div>
      <h1>Hello</h1>
      <p>World</p>
    </div>
  );
}

// Fiber tree (visualization)
//        Fiber(App, FunctionComponent)
//             ↓ child
//        Fiber(div, HostComponent)
//             ↓ child
//        Fiber(h1, HostComponent)
//             ↓ sibling
//        Fiber(p, HostComponent)
//             ↑ return → div, return → App
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Fiber traversal algorithm:**

```typescript
// Sodda traverse — depth-first with sibling
function performUnitOfWork(fiber: Fiber): Fiber | null {
  // 1. Begin work — render this fiber
  beginWork(fiber);

  // 2. Move to child if exists
  if (fiber.child) return fiber.child;

  // 3. No child — go to sibling, or back up
  let current: Fiber | null = fiber;
  while (current) {
    completeWork(current);  // finalize this fiber
    if (current.sibling) return current.sibling;
    current = current.return;  // up to parent, try parent's sibling
  }

  return null;  // tree complete
}

// Work loop
function workLoop() {
  while (nextUnitOfWork && !shouldYield()) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }

  if (!nextUnitOfWork) {
    commitRoot();  // render finished
  } else {
    scheduleCallback(workLoop);  // resume later
  }
}

function shouldYield(): boolean {
  return performance.now() >= deadline;  // 5ms slice
}
```

**Stack Reconciler (R15) algorithm:**

```typescript
// Recursive — JS call stack
function reconcile(parent, oldChildren, newChildren) {
  for (let i = 0; i < newChildren.length; i++) {
    const old = oldChildren[i];
    const newChild = newChildren[i];

    if (old.type === newChild.type) {
      updateProps(old, newChild);
      reconcile(old, old.children, newChild.children);  // ← recursive
    } else {
      mount(parent, newChild);
      unmount(parent, old);
    }
  }
}

// 1000 components — 1000 stack frames, no yield
```

**Memory layout — V8 optimization:**

```
Fiber (heap):
  | tag     | type    | key   | ... |
  | child   | sibling | return| ... |
  | alternate                  ... |
  | memoizedState              ... |
  | flags                      ... |
  ↑ V8 hidden class — monomorphic IC fast property access
```

V8 har Fiber bir xil hidden class — monomorphic IC fast property access.

**Tag types (asosiy):**

```typescript
const enum WorkTag {
  FunctionComponent = 0,
  ClassComponent = 1,
  IndeterminateComponent = 2,
  HostRoot = 3,
  HostPortal = 4,
  HostComponent = 5,        // 'div', 'button', etc.
  HostText = 6,             // text node
  Fragment = 7,
  Mode = 8,
  ContextConsumer = 9,
  ContextProvider = 10,
  ForwardRef = 11,
  Profiler = 12,
  SuspenseComponent = 13,
  MemoComponent = 14,
  SimpleMemoComponent = 15,
  LazyComponent = 16,
  // ...
}
```

Reconciler `tag`'ga qarab har xil mantiq ishlatadi (memo bailout, suspense throw, context propagation).

**Fiber lifecycle:**

1. **Create** — `createFiber(type, props, key)` — initial mount
2. **Update** — alternate yaratiladi (clone), props yangilanadi
3. **Reuse** — alternate fiber memory'sini qayta ishlatish (no GC)
4. **Delete** — parent'da `ChildDeletion` flag, `deletions` array'ga qo'shiladi

**Performance characteristics:**

| Metric | Stack Reconciler | Fiber |
|--------|------------------|-------|
| Render style | Recursive, sync | Iterative, async |
| Interrupt | ❌ | ✅ 5ms slices |
| Priority | ❌ flat | ✅ Lanes (31 lanes, bitmap) |
| Yield | ❌ blocking | ✅ between fibers |
| Memory | Stack frames | Heap (Fiber objects) + recyclable alternate |

</details>

### Edge Cases

- **Async render leak**: Render phase yieldlarsiz — long task bo'lsa, Lighthouse "Total Blocking Time" yomonlashadi. Fiber time slicing buni hal qiladi.
- **`useState` mid-render**: Render phase set state — React limited support (queue va re-render shu fiber). Anti-pattern.
- **Fiber memory**: Har fiber object — `tag`, `type`, `key`, `child`, `sibling`, `return`, `alternate`, `memoizedState`, `memoizedProps`, `pendingProps`, `flags`, `lanes` va boshqa ko'p field'lar (V8 hidden class). Katta tree'da fiber memory sezilarli, lekin alternate recycle bilan amortize qilinadi.

### Follow-up savollar

- "Fiber'ning asl maqsadi nima edi?" — Concurrent rendering uchun groundwork. R16'da introduced (async features off), R18'da to'liq realize.
- "Fiber tree component tree'ga teng-mi?" — Yo'q. Komponent tree — JSX hierarchy. Fiber tree — har komponent + Host element + Fragment + Provider va h.k. uchun fiber. 1 komponent JSX'da — 1+ fiber.
- "React Native Fiber ishlatadi-mi?" — Ha, bir xil reconciler (`react-reconciler` package). Faqat host config farq.

</details>

---

### 2. Fiber tree strukturasi — `child`, `sibling`, `return` pointers [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Fiber tree **linked list** sifatida quriladi: har Fiber'da 3 ta pointer — **`child`** (birinchi bola), **`sibling`** (keyingi aka-uka), **`return`** (parent). Recursive tree yo'q — pointer'lar bilan traverse qilinadi. Bu structure **interruptible rendering** uchun zarur — joriy fiber pozitsiyani saqlab, keyingi work cycle'da davom etish mumkin.

### To'liq tushuntirish

**3 ta pointer maqsadi:**

| Pointer | Yo'nalish | Maqsad |
|---------|-----------|--------|
| `child` | Pastga (birinchi child) | Children tree'ga kirish |
| `sibling` | O'ng tomonga | Sibling fiber'ga (depth-first traversal) |
| `return` | Yuqoriga (parent) | Bola tugagandan keyin parent'ga qaytish |

### Kod misoli

```tsx
function App() {
  return (
    <div>
      <Header>
        <Logo />
        <Menu />
      </Header>
      <Main>
        <Article />
      </Main>
    </div>
  );
}
```

**Fiber tree (linked list):**

```
                    App
                     │ child
                     ↓
                    div
                     │ child
                     ↓
            ┌─── Header ───→ Main ───→ null (sibling)
            │ child            │ child
            ↓                  ↓
       ┌── Logo ──→ Menu      Article
       │            │ sibling   │ sibling
       │            ↓           ↓
       │           null        null
       │
       │ return ↑
       Header     (Logo, Menu both: return → Header)
                 (Header, Main: return → div)
                 (div: return → App)
                 (App: return → root)
```

**Traversal algorithm (depth-first):**

```typescript
function traverseTree(root: Fiber) {
  let current: Fiber | null = root;

  while (current) {
    console.log(current.type);

    // 1. Try to descend
    if (current.child) {
      current = current.child;
      continue;
    }

    // 2. No child — try sibling, else return up
    while (current) {
      if (current.sibling) {
        current = current.sibling;
        break;
      }
      current = current.return;  // up to parent
    }
  }
}

// Output for App example:
// App, div, Header, Logo, Menu, Main, Article
// (depth-first, pre-order)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Traversal — work in vs complete out:**

```typescript
function performUnitOfWork(fiber: Fiber): Fiber | null {
  // beginWork — komponent render, JSX olish, children'ni reconcile
  const next = beginWork(fiber);

  if (next !== null) {
    // child mavjud — return child, work loop child'ni handle qiladi
    return next;
  }

  // Children yo'q yoki barcha completed — completeWork (DOM update prep)
  let current: Fiber | null = fiber;
  while (current) {
    completeWork(current);  // sub-tree complete
    if (current.sibling) return current.sibling;  // next sibling
    current = current.return;
  }

  return null;
}
```

`beginWork` — komponent function chaqirish, hooks, children reconcile.
`completeWork` — DOM instance create (HostComponent), props set, append children.

**Why linked list (not array):**

```typescript
// Array — random access OK, lekin insert/remove O(n)
type ArrayChildren = Fiber[];  // [a, b, c]
// Insert at index 1: O(n) shift

// Linked list — sequential access, insert/remove O(1)
type LinkedChildren = {
  child: Fiber | null;  // a
  // a.sibling = b
  // b.sibling = c
};
// Insert between a-b: change pointers (O(1))
```

Reconciliation — children'larga ko'p insert/move/delete bo'ladi → linked list yaxshi.

**Effect list (deprecated R18+):**

R16-R17'da Fiber `nextEffect` pointer ham bor edi — barcha effect'larni alohida list bilan kuzatardi. R18'da bu olib tashlandi (subtreeFlags optimization).

```typescript
// R17 (eski)
fiber.firstEffect = ...;
fiber.lastEffect = ...;
fiber.nextEffect = ...;
// Commit phase'da nextEffect bo'ylab traverse

// R18 (yangi)
fiber.flags = bitmask;
fiber.subtreeFlags = bitmask;  // children flags OR'ed
// Commit phase'da subtreeFlags = 0 → skip subtree
```

`subtreeFlags` — fast skip optimization. Subtree'da hech qanday effect yo'q bo'lsa, butun subtree skip.

**Fiber traversal complexity:**

```typescript
// Naive recursive
function recurse(fiber) {
  for (const child of fiber.children) recurse(child);  // depth-recursion
}
// Stack frames: O(depth)
// 1000 deep — stack overflow risk

// Linked list iterative
function iterate(fiber) {
  let current = fiber;
  while (current) {
    if (current.child) { current = current.child; continue; }
    while (current && !current.sibling) current = current.return;
    if (!current) break;
    current = current.sibling;
  }
}
// Stack frames: O(1)
// No overflow risk
```

**Pause and resume:**

```typescript
// Render phase yields
function workLoop() {
  while (nextUnitOfWork && !shouldYield()) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }

  if (nextUnitOfWork) {
    // Yielded — schedule resume
    scheduleCallback(NormalPriority, workLoop);
  } else {
    finishConcurrentRender();
  }
}

// Resume:
// nextUnitOfWork — saqlangan Fiber pointer
// workLoop davom etadi — child/sibling/return navigation
```

`nextUnitOfWork` saqlanganligi uchun yield-resume seamless.

**Sibling traversal — same parent siblings:**

```
Parent
  ├── Child1 (sibling → Child2)
  ├── Child2 (sibling → Child3)
  └── Child3 (sibling → null)

Child1.return → Parent
Child2.return → Parent  ← all siblings same return
Child3.return → Parent
```

</details>

### Edge Cases

- **Single child — no sibling**: `<div><span /></div>` — `div.child = span`, `span.sibling = null`.
- **Empty parent**: `<div />` — `div.child = null`. Traversal: complete immediately.
- **Fragment fibers**: `<>...</>` — Fragment fiber, children are siblings. Sibling chain has no Fragment in between.

### Follow-up savollar

- "Fiber pointer'lari mutable-mi?" — Ha. Reconciliation reorder bo'lsa — `sibling` pointer'lar yangilanadi. `child` parent'dan birinchi bola.
- "BFS vs DFS — Reconciler qaysi-ni ishlatadi?" — DFS (depth-first). Sabab: child render parent'dan oldin (children'ning props parent natijalariga bog'liq emas).
- "Multiple roots — har biri o'z tree-mi?" — Ha. Har `createRoot` — alohida fiber tree. `HostRoot` fiber top-level.

</details>

---

### 3. `current` vs `workInProgress` — double buffering [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React **double buffering** ishlatadi — ikki Fiber tree: **`current`** (hozirgi DOM bilan sinxron, render qilingan) va **`workInProgress`** (yangi update'da quriladigan, hali commit qilinmagan). Render phase WIP'ga yoziladi, current daxlsiz qoladi. Commit muvaffaqiyatli bo'lsa, swap: `current = workInProgress`. Render xatosi bo'lsa, `workInProgress` tashlanadi, `current` saqlanadi.

### To'liq tushuntirish

**Double buffering nima:**

Computer graphics'dan kelgan termin — ikkita buffer (frame): biri ekranda ko'rinadi (front), boshqasiga rasm chiziladi (back). Tugagach swap. Tearing yo'q (yarim-rasm ko'rinmaydi).

**React'da:**

```
   FiberRoot
   ├─ current ────────→ <CurrentTree>  (render qilingan, DOM bilan sync)
   └─ workInProgress ─→ <WIPTree>       (yangi update'da quriladi)
                              ↓
                         render finished
                              ↓
                            commit
                              ↓
                       current = WIPTree
                       WIPTree → null (or recycled as next alternate)
```

### Kod misoli

```tsx
// Conceptual — internal state visualisation
type FiberRoot = {
  current: Fiber;          // hozirgi tree (DOM'da reflected)
  workInProgress: Fiber | null;  // yangi tree quriladi (render phase'da)
  // ...
};

// Initial mount
function initialMount() {
  const root: FiberRoot = {
    current: createHostRootFiber(),
    workInProgress: null,
  };

  // 1. Yangi WIP tree yaratish (clone of current)
  const wip = cloneFiberTree(root.current);
  root.workInProgress = wip;

  // 2. Render phase — wip tree quriladi
  while (workLoop(wip)) { /* ... */ }

  // 3. Commit
  commitRoot(wip);

  // 4. Swap
  root.current = wip;
  root.workInProgress = null;
}

// Re-render (state update)
function update() {
  const root = getCurrentRoot();

  // 1. WIP yaratish — current.alternate ishlatish (recycled)
  const wip = root.current.alternate ?? cloneFiber(root.current);
  root.workInProgress = wip;

  // 2. Render phase
  while (workLoop(wip)) { /* ... */ }

  // 3. Commit (success)
  if (commitSuccessful) {
    root.current = wip;  // ← swap
  } else {
    // Render abort — current daxlsiz
  }
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why double buffering matters:**

1. **Render abort safety** — render phase'da error throw bo'lsa yoki interrupt bo'lsa, current tree o'zgarmaydi. UI hech qachon "yarim" holatda qolmaydi.

2. **Commit atomicity** — commit yagona swap operation'i (current = wip). Tearing impossible.

3. **Concurrent rendering** — bir render davom etayotganda, yangi high-pri update kelsa, eski WIP tashlanadi va yangi WIP boshlanadi (current saqlanadi).

4. **Memory efficiency** — alternate pair recycle: ikki tree memory'sini qayta ishlatish, GC pressure kam.

**`alternate` pair:**

```typescript
// Har Fiber'da:
fiber.alternate: Fiber | null;

// Initial mount: alternate yo'q
currentFiber.alternate === null;

// First update: WIP yaratish — alternate'ni create
const wipFiber = createWorkInProgress(currentFiber, props);
// wipFiber.alternate === currentFiber
// currentFiber.alternate === wipFiber
// (bidirectional)

// Commit: swap
// Now: workInProgress fiber → current
// Old current → workInProgress (recycled)
```

**`createWorkInProgress` — recycle pattern:**

```typescript
function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {
  let workInProgress = current.alternate;

  if (workInProgress === null) {
    // First time — create fresh fiber
    workInProgress = createFiber(current.tag, pendingProps, current.key);
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;

    // Set up alternate links
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    // Recycle existing — just reset
    workInProgress.pendingProps = pendingProps;
    workInProgress.flags = 0;
    workInProgress.deletions = null;
  }

  // Copy scheduling state from current
  workInProgress.childLanes = current.childLanes;
  workInProgress.lanes = current.lanes;
  workInProgress.child = current.child;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;
  workInProgress.updateQueue = current.updateQueue;

  return workInProgress;
}
```

**Visualization (3 update cycles):**

```
Cycle 1 (mount):
  current = null
  WIP = NewTreeA (fresh)
  → commit
  current = NewTreeA, WIP.alternate = none yet

Cycle 2 (update):
  current = NewTreeA
  WIP = clone(NewTreeA) → TreeB (alternate of A)
  → commit
  current = TreeB, A.alternate = B, B.alternate = A

Cycle 3 (update):
  current = TreeB
  WIP = recycle(B.alternate) = TreeA (reset)
  → commit
  current = TreeA  ← old TreeA recycled, now TreeA again!
```

`alternate` pair persists — recycling ping-pong.

**Render abort scenario:**

```typescript
function tryRender() {
  const wip = createWorkInProgress(root.current);

  try {
    while (workLoop(wip)) { }
    commitRoot(wip);  // success
    root.current = wip;
  } catch (error) {
    // Render error — wip tashlanadi
    // root.current daxlsiz
    // alternate links ham daxlsiz (current.alternate = wip still, but unused)
    handleError(error);
  }
}
```

Browser DOM — current bilan sync, hech narsa o'zgarmagan.

**Concurrent rendering interrupt:**

```typescript
// Low priority render boshlandi
const lowPriWIP = createWorkInProgress(root.current);
// Render phase'da yieldsiz progress

// User input — high priority update keldi
// React: current low-pri WIP'ni tashla, high-pri uchun yangi WIP yarat
abandonWorkInProgress(lowPriWIP);
const highPriWIP = createWorkInProgress(root.current);
// New render starts

// Commit
root.current = highPriWIP;
```

Tashlangan WIP — alternate pair'da qoladi (next cycle'da recycled).

**Memory analysis (mental model):**

```
1 komponent — 1 yoki 2 fiber (current + alternate, mount'dan keyin)
Alternate recycling: ikki tree memory ping-pong sifatida qayta ishlatiladi
Without recycling — GC pressure (har render allocate + free)
Recycling tradeoff: ~2x current tree memory baseline, lekin GC pause kam
```

</details>

### Edge Cases

- **Single render** (initial mount): `current` initial Host Root — empty tree. WIP `null`. Render boshida WIP create.
- **Multiple roots** (`createRoot` ko'p): Har root o'z `current/WIP` pair'iga ega. Independent.
- **`useEffect` cleanup**: Old fiber's effect cleanup chaqiriladi (current → WIP transition'da). Effect references old state correctly.

### Follow-up savollar

- "Why not single tree mutated in place?" — Render abort yoki concurrent interrupt'da partial state visible bo'lardi (tearing).
- "Memory overhead acceptable?" — Yes. Pre-R16 React ham analogous (instances + element). Modern engines (V8) hidden class optimize qiladi.
- "Cross-frame issues?" — Concurrent rendering: render phase ko'p frame davom etishi mumkin (yield orqali). Current tree shu vaqt mobaynida DOM'da consistent — user input current tree bilan ishlaydi.

</details>

---

### 4. `alternate` pointer roli — recycle va swap [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`alternate`** — har Fiber'dagi pointer, **pair fiber**'ga (current ↔ workInProgress). Bidirectional: `currentFiber.alternate === wipFiber` va vice versa. Maqsad: render cycle'larda fiber memory'sini **recycle** qilish (har render fresh allocate qilmasdan), va commit'da **atomic swap** (current va WIP roli almashinadi). GC pressure kamaytiradi.

### To'liq tushuntirish

**`alternate` 3 ta vazifa:**

1. **Recycling** — keyingi render'da alternate fiber'ni qayta ishlatish (no GC)
2. **Swap reference** — commit'da current/WIP pointers swap
3. **Old state access** — `useEffect` cleanup, ref detach paytida eski state'ga kirish

### Kod misoli

```typescript
// Initial mount — alternate yo'q
const root = createHostRoot();
// root.current.alternate === null

// First render
const wip = createWorkInProgress(root.current, props);
// Internal:
//   wip = createFiber(...)
//   wip.alternate = root.current
//   root.current.alternate = wip
// Now: bidirectional pair

// Commit
root.current = wip;
// Now: new current.alternate = old current
//      old current.alternate = new current

// Second update
const wip2 = createWorkInProgress(root.current);
// createWorkInProgress checks alternate:
//   if (current.alternate !== null) {
//     // recycle: reset alternate fiber
//     return alternate;
//   }
// Returns old current (now reset as WIP) — no allocation!
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Recycle implementation:**

```typescript
function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {
  let workInProgress = current.alternate;

  if (workInProgress === null) {
    // ============================================
    // First-time: yangi fiber yaratish
    // ============================================
    workInProgress = createFiber(current.tag, pendingProps, current.key);
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;

    // Bidirectional alternate
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    // ============================================
    // Recycle existing alternate
    // ============================================
    workInProgress.pendingProps = pendingProps;
    workInProgress.type = current.type;

    // Reset effect-related fields
    workInProgress.flags = NoFlags;
    workInProgress.subtreeFlags = NoFlags;
    workInProgress.deletions = null;
  }

  // Reset lanes, copy from current
  workInProgress.lanes = current.lanes;
  workInProgress.childLanes = current.childLanes;

  // Copy state references
  workInProgress.child = current.child;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;
  workInProgress.updateQueue = current.updateQueue;
  workInProgress.dependencies = current.dependencies;
  workInProgress.sibling = current.sibling;
  workInProgress.index = current.index;
  workInProgress.ref = current.ref;

  return workInProgress;
}
```

**Visualization — alternate pair lifecycle:**

```
T0 (initial state):
  Tree A: [a, b, c, d, e]  ← current
  Tree B: null              ← no alternate yet

T1 (first update):
  Tree A: [a, b, c, d, e]  ← current
  Tree B: [a', b', c', d', e']  ← WIP (allocated, alternate of A)

  a.alternate = a'
  a'.alternate = a
  (and so on for each fiber)

T2 (commit T1):
  Tree A: [a, b, c, d, e]  ← now WIP (alternate of B)
  Tree B: [a', b', c', d', e']  ← current

  Pointers same as before — only role swapped.

T3 (second update):
  Tree A: recycled, reset to fresh state
  Tree B: [a', b', c', d', e']  ← current

  No new allocation! Memory reused.
```

**`useEffect` cleanup uses alternate:**

```typescript
function commitPassiveUnmount(fiber: Fiber) {
  // Joriy fiber — yangi current
  // Cleanup eski state'ga kirishi kerak — alternate'dan olinadi

  const oldFiber = fiber.alternate;
  if (oldFiber === null) return;  // mount, no cleanup

  const oldEffect = oldFiber.memoizedState;  // old hooks state
  // Run cleanup with old closure
  oldEffect.destroy();
}
```

`alternate` — eski state'ga "tunnel".

**Effect — same fiber instance, different render:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log("effect:", count);  // ← closure'da count
    return () => console.log("cleanup:", count);  // ← bu closure'dagi count
  }, [count]);

  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

// Mount:
// effect: 0  (count=0 closure)

// Click — setCount(1):
// Re-render: count=1
// Old effect cleanup runs: "cleanup: 0" (alternate's closure with count=0)
// New effect runs: "effect: 1" (current's closure with count=1)
```

Cleanup closure'i — alternate fiber'ning `memoizedState`'idan.

**`commitRoot` swap:**

```typescript
function commitRoot(root: FiberRoot) {
  const finishedWork = root.finishedWork;  // WIP

  // 1. Commit phases (mutation, layout, ...)
  commitMutation(finishedWork);
  commitLayout(finishedWork);

  // 2. ATOMIC SWAP
  root.current = finishedWork;
  // After this line: WIP (now current) reflects DOM
  //                  Old current (now WIP-alternate) — recycle pool

  // 3. Schedule passive effects
  scheduleCallback(NormalPriority, flushPassiveEffects);
}
```

Single line `root.current = finishedWork` — atomic state transition.

**Memory pressure analysis:**

| Scenario | Without recycling | With recycling |
|----------|-------------------|-----------------|
| 100 renders, 1000 fibers each | 100,000 Fiber allocations + GC | 2,000 Fibers, reused |
| GC pause | Frequent (every cycle) | Minimal |
| Memory steady state | 1 tree (1000 fibers) | 2 trees (2000 fibers) |

Recycling: **2x baseline memory** for **massively** less GC pressure.

**Edge case — abandoned WIP:**

```typescript
// Concurrent rendering — high pri interrupt
const lowPriWIP = createWorkInProgress(root.current);
// ... half-rendered ...

// Interrupt
abandonWorkInProgress(lowPriWIP);

// What happens to alternate?
// lowPriWIP still reachable from root.current.alternate
// On next render: createWorkInProgress reuses lowPriWIP (now reset)
```

Abandoned WIP — alternate'da qoladi, keyingi render'da recycle.

</details>

### Edge Cases

- **`alternate === null`**: Initial mount fiber (first render, no previous tree).
- **Cleared during commit**: Some refs cleared to break cycles (memory hygiene), but alternate persists for recycle.
- **Fiber tree restructure**: Significant tree changes (suspend, switch component) — alternate pair may not match exact structure, partial recycle.

### Follow-up savollar

- "Alternate pair — circular reference?" — Ha (a.alternate = b, b.alternate = a). V8 GC handles cycles correctly (mark-sweep, not refcounting).
- "DevTools'da alternate ko'rsatadi-mi?" — React DevTools internal'da alternate'dan foydalanadi (current props vs pending). User-facing UI'da alohida ko'rinmaydi.
- "Why not array of fibers?" — Object pointer simplicity, no index lookup overhead.

</details>

---

### 5. Fiber tag types — qaysi turlar va ahamiyati [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Fiber tag** — har fiber'ning **turini** belgilaydi (internal enum — aniq raqam qiymatlari versiya bo'yicha o'zgarishi mumkin). Reconciler `tag`'ga qarab har xil mantiq ishlatadi: function vs class komponent render, host element DOM operatsiyasi, Memo bailout, Suspense throw handling, Context propagation. Asosiy turlar: `FunctionComponent`, `ClassComponent`, `HostRoot`, `HostComponent`, `HostText`, `Fragment`, `ContextProvider`, `MemoComponent`, `SuspenseComponent`, `ForwardRef`, `LazyComponent`.

### To'liq tushuntirish

**Asosiy tag types (React 19 internal — taxminiy ro'yxat, aniq qiymatlar versiya bo'yicha o'zgarishi mumkin):**

```typescript
// React internal — sodda ro'yxat (raqamlar versiyaga bog'liq)
const enum WorkTag {
  FunctionComponent,        // function App() {}
  ClassComponent,           // class App extends Component
  IndeterminateComponent,   // function/class noma'lum
  HostRoot,                 // ReactDOM root container
  HostPortal,               // createPortal target
  HostComponent,            // 'div', 'button', 'span', ...
  HostText,                 // text content (string/number child)
  Fragment,                 // <Fragment>, <>...</>
  Mode,                     // <StrictMode>
  ContextConsumer,          // <Context.Consumer>
  ContextProvider,          // <Context.Provider>, R19: <Context>
  ForwardRef,               // forwardRef(...)
  Profiler,                 // <Profiler>
  SuspenseComponent,        // <Suspense>
  MemoComponent,            // memo(Component, areEqual)
  SimpleMemoComponent,      // memo(Component) — shallow equal fast path
  LazyComponent,            // lazy(...) (loading)
  DehydratedFragment,       // SSR hydration boundary
  OffscreenComponent,       // <Offscreen> (Activity API, R19 unstable)
  CacheComponent,           // RSC cache
  // ... va h.k.
}
```

### Kod misoli

```tsx
// JSX → Fiber tags

function App() {              // → FunctionComponent
  return (
    <div>                     // → HostComponent "div"
      <h1>Hello</h1>           // → HostComponent "h1"
                              // → HostText "Hello"
      <>                       // → Fragment
        <Custom />              // → FunctionComponent (Custom)
      </>
      <Context.Provider value={x}>  // → ContextProvider
        <Consumer />
      </Context.Provider>
      <Suspense fallback={<L />}>   // → SuspenseComponent
        <Lazy />                     // → LazyComponent (resolve'gacha)
      </Suspense>
    </div>
  );
}

// Class component
class OldComponent extends React.Component {  // → ClassComponent
  render() {
    return <span />;  // → HostComponent
  }
}

// memo
const MemoButton = React.memo(Button);  // → MemoComponent yoki SimpleMemoComponent (areEqual yo'q bo'lsa)

// forwardRef
const FwdInput = React.forwardRef(Input);  // → ForwardRef
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Per-tag rendering logic:**

```typescript
function beginWork(current: Fiber | null, workInProgress: Fiber): Fiber | null {
  switch (workInProgress.tag) {
    case FunctionComponent:
      return updateFunctionComponent(current, workInProgress);

    case ClassComponent:
      return updateClassComponent(current, workInProgress);

    case HostRoot:
      return updateHostRoot(current, workInProgress);

    case HostComponent:
      return updateHostComponent(current, workInProgress);

    case HostText:
      return null;  // text — no children

    case Fragment:
      return updateFragment(current, workInProgress);

    case ContextProvider:
      return updateContextProvider(current, workInProgress);

    case ContextConsumer:
      return updateContextConsumer(current, workInProgress);

    case ForwardRef:
      return updateForwardRef(current, workInProgress);

    case MemoComponent:
      return updateMemoComponent(current, workInProgress);

    case SuspenseComponent:
      return updateSuspenseComponent(current, workInProgress);

    case LazyComponent:
      return mountLazyComponent(current, workInProgress);

    case Profiler:
      return updateProfiler(current, workInProgress);

    // ... 20+ tags
  }
}
```

Har tag — alohida `update*` function. Fiber pipeline tag-dispatched.

**Tag transitions:**

Fiber tag deyarli o'zgarmaydi (immutable). Exception: `IndeterminateComponent`:

```typescript
// JSX <Foo /> — Foo function vs class noma'lum (`typeof Foo` insufficient)
// Initial render: tag = IndeterminateComponent
// First render output bo'yicha:
//   - has render() method → tag = ClassComponent
//   - returns JSX → tag = FunctionComponent
```

Modern React'da tag detection compile-time (Babel transform JSX `<Foo />` to `_jsx(Foo, ...)` — Foo runtime type'i tekshiriladi).

**`HostRoot` — root fiber:**

```typescript
// createRoot(container) — yaratiladigan fiber
const hostRootFiber = {
  tag: HostRoot,
  stateNode: { containerInfo: container, current: thisFiber },
  // ...
};
```

`HostRoot` — har React tree'ning yuqori darajadagi fiber'i. App'ning oddiy parent'i.

**`HostPortal` — portal target:**

```tsx
import { createPortal } from "react-dom";

function Modal({ children }) {
  return createPortal(children, document.getElementById("modal-root"));
}

// Fiber: HostPortal (4)
// stateNode.containerInfo: alternate DOM node
// Children render to different DOM, lekin React tree'da Modal'ning child'i
```

**`Fragment` — no DOM:**

```tsx
<>
  <h1>A</h1>
  <p>B</p>
</>

// Fiber: Fragment (7)
// stateNode: null (no DOM)
// children: h1, p (siblings of Fragment in tree, but DOM children of Fragment.return)
```

**`MemoComponent` (14) vs `SimpleMemoComponent` (15):**

```tsx
// Custom areEqual → MemoComponent
const A = memo(Component, (prev, next) => prev.id === next.id);
// Tag = 14, areEqual function check

// No areEqual → SimpleMemoComponent
const B = memo(Component);
// Tag = 15, shallow equal optimization (faster path)
```

**`SuspenseComponent` (13) — special handling:**

```typescript
function updateSuspenseComponent(current, workInProgress) {
  const props = workInProgress.pendingProps;

  // Check if children threw a Promise
  if (childrenSuspended) {
    // Render fallback
    workInProgress.flags |= DidCapture;
    return mountSuspenseFallbackChildren(workInProgress, props.fallback);
  }

  // Render children
  return mountSuspensePrimaryChildren(workInProgress, props.children);
}
```

**`LazyComponent` (16) — async loading:**

```tsx
const LazyAbout = lazy(() => import("./About"));

// First render:
// Fiber: LazyComponent (16)
// Module not loaded yet → throw Promise (Suspense catches)
// Suspense renders fallback

// Module loaded:
// Fiber updated: LazyComponent → resolved to FunctionComponent or ClassComponent
// Re-render
```

**`ContextProvider` propagation:**

```typescript
function updateContextProvider(current, workInProgress) {
  const newValue = workInProgress.pendingProps.value;

  // Compare with previous value (Object.is)
  if (current && Object.is(current.memoizedProps.value, newValue)) {
    // Same value — no propagation needed
  } else {
    // Propagate to descendants — mark Context Consumers as needing update
    propagateContextChange(workInProgress, contextType, lanes);
  }

  return workInProgress.child;
}
```

**`OffscreenComponent` (22) — R19 Activity API:**

```tsx
import { unstable_Activity as Activity } from "react";

<Activity mode="hidden">
  <ExpensiveComponent />
</Activity>

// Fiber: OffscreenComponent
// Mode "hidden": render lekin DOM commit YO'Q, state preserved
// Mode "visible": commit
```

R19 — pre-render or hidden tab content (offscreen).

</details>

### Edge Cases

- **`IndeterminateComponent`**: Rare — modern React detects function/class compile-time.
- **Class with `render` returning Class instance**: Yo'q, JSX must be Element. Class instance JSX'ni qaytaradi.
- **`HostText` separate fiber**: `<p>Hello {name}</p>` — `Hello ` HostText, `{name}` HostText (or other tag). 2 ta sibling.

### Follow-up savollar

- "Tag bo'yicha optimization?" — `SimpleMemoComponent` faster path (no areEqual function call). `Fragment` no DOM allocation.
- "Custom Fiber tag yozish mumkinmi?" — Yo'q. Tag — internal enum. Custom rendering logic — komponent-level (memo, forwardRef, etc.).
- "DevTools tag ko'rsatadi-mi?" — DevTools "Components" tab fiber type'ni ko'rsatadi (Function, Class, Memo, etc.) — tag bilan corresponding.

</details>

---

### 6. Effect list — `flags` va `subtreeFlags` [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`flags`** — har Fiber'da bitmask — fiber'ning side effect'larini belgilaydi (Placement, Update, Deletion, Passive, Layout, Ref, Snapshot, ...). **`subtreeFlags`** — children'ning flags OR'ed — fast skip optimization (subtree'da effect yo'q bo'lsa, butun subtree skip). R18'da effect linked list (`nextEffect`) o'rniga shu pattern qabul qilindi — saqlash va traverse tezroq.

### To'liq tushuntirish

**`flags` bitmask:**

```typescript
const enum Flags {
  NoFlags         = 0b00000000000000000000000000,
  PerformedWork   = 0b00000000000000000000000001,  // 1
  Placement       = 0b00000000000000000000000010,  // 2 — DOM insert
  Update          = 0b00000000000000000000000100,  // 4 — DOM property update
  ChildDeletion   = 0b00000000000000000000010000,  // 16 — parent'da child o'chirilishi
  ContentReset    = 0b00000000000000000000100000,  // 32
  Callback        = 0b00000000000000000001000000,  // 64
  DidCapture      = 0b00000000000000000010000000,  // 128 — error caught
  ForceClientRender = 0b00000000000000000100000000,  // 256
  Ref             = 0b00000000000000001000000000,   // 512
  Snapshot        = 0b00000000000000010000000000,   // 1024 — getSnapshotBeforeUpdate
  Passive         = 0b00000000000000100000000000,   // 2048 — useEffect
  Hydrating       = 0b00000000000001000000000000,   // 4096
  Visibility      = 0b00000000000010000000000000,   // 8192
  // ...
}
```

**Bitwise operations:**

```typescript
// Set flag
fiber.flags |= Update;  // OR

// Check flag
if (fiber.flags & Update) { /* has Update */ }  // AND

// Clear flag
fiber.flags &= ~Update;  // AND NOT

// Multiple flags
fiber.flags |= Placement | Update | Ref;
```

### Kod misoli

```tsx
// Initial render
function MyComponent({ name }: { name: string }) {
  const ref = useRef<HTMLDivElement>(null);
  const [count, setCount] = useState(0);

  useEffect(() => { console.log("effect"); }, [count]);

  return (
    <div ref={ref} onClick={() => setCount(c => c + 1)}>
      {name}: {count}
    </div>
  );
}

// Initial mount Fiber flags:
// MyComponent fiber:
//   flags |= PerformedWork
//   flags |= Passive       (useEffect)
//
// div fiber:
//   flags |= Placement     (insert into DOM)
//   flags |= Ref           (ref attach)
//
// HostText (name) fiber:
//   flags |= Placement
//
// HostText (count) fiber:
//   flags |= Placement

// Re-render (count: 0 → 1):
// MyComponent fiber:
//   flags |= PerformedWork
//   flags |= Passive       (deps [count] changed)
//
// div fiber:
//   flags = NoFlags        (no DOM change)
//   subtreeFlags |= Update (HostText changed)
//
// HostText (count):
//   flags |= Update        (text content "0" → "1")
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`subtreeFlags` — fast skip:**

```typescript
function commitMutationEffects(finishedWork: Fiber) {
  // Quick check: subtree has no flags?
  if ((finishedWork.subtreeFlags & MutationMask) === NoFlags &&
      (finishedWork.flags & MutationMask) === NoFlags) {
    return;  // Skip entire subtree
  }

  // Has flags — recurse into children
  let child = finishedWork.child;
  while (child !== null) {
    commitMutationEffects(child);
    child = child.sibling;
  }

  // Apply this fiber's effects
  if (finishedWork.flags & Placement) {
    commitPlacement(finishedWork);
  }
  if (finishedWork.flags & Update) {
    commitUpdate(finishedWork);
  }
  // ... etc
}
```

`subtreeFlags` — children flags OR'ed bottom-up. Aggregation `completeWork`'da:

```typescript
function completeWork(workInProgress: Fiber) {
  // ... commit logic for this fiber

  // Bubble up subtreeFlags
  let subtreeFlags = NoFlags;
  let child = workInProgress.child;
  while (child !== null) {
    subtreeFlags |= child.subtreeFlags;
    subtreeFlags |= child.flags;
    child = child.sibling;
  }
  workInProgress.subtreeFlags = subtreeFlags;
}
```

**Effect mask categories:**

```typescript
// Mutation phase mask
const MutationMask = Placement | Update | Deletion | ContentReset | Ref | ...;

// Layout phase mask
const LayoutMask = Update | Callback | Ref | Snapshot;

// Passive effect mask (useEffect)
const PassiveMask = Passive | Visibility;

// Quick check during commit phase
if ((subtreeFlags & MutationMask) === 0) {
  // No mutations in subtree — skip
}
```

**R17 vs R18 — effect list change:**

R17:
```typescript
// Each fiber: nextEffect pointer
fiber.nextEffect = ... ;

// Commit traversal:
let effect = root.firstEffect;
while (effect) {
  applyEffect(effect);
  effect = effect.nextEffect;
}
// O(N) where N = fibers with effects
```

R18:
```typescript
// No nextEffect, use subtreeFlags
function commitMutationEffects(fiber) {
  if (fiber.subtreeFlags & MutationMask) {
    // Has effects in subtree — recurse
    commitMutationEffects(fiber.child);
    if (fiber.sibling) commitMutationEffects(fiber.sibling);
  }
  if (fiber.flags & MutationMask) {
    applyEffect(fiber);
  }
}
// O(N + S) where N = fibers with effects, S = skipped subtrees
```

R18 advantages:
- No effect list maintenance during render
- Better skip optimization (whole subtrees)
- Simpler code, fewer bugs

**Concrete flag scenarios:**

| Scenario | Flag set |
|----------|----------|
| New component mounted | `Placement` |
| Props changed | `Update` |
| Component unmounted | `ChildDeletion` (parent's deletions array'ga qo'shiladi) |
| `useEffect` to run | `Passive` |
| `useLayoutEffect` to run | `Update` (Layout phase processing) |
| Ref attach/detach | `Ref` |
| `getSnapshotBeforeUpdate` | `Snapshot` |
| Error caught in boundary | `DidCapture`, `ShouldCapture` |
| Hydration | `Hydrating` |

**`Deletion` — special handling:**

```typescript
// Deletion stored in parent's deletions array, not flags
parent.deletions = [...(parent.deletions ?? []), deletedFiber];
parent.flags |= ChildDeletion;
```

Sabab: deleted fiber'ning o'zi WIP tree'da yo'q — parent referencing kerak.

**`PerformedWork` — render bypass marker:**

```typescript
// Bailout — no work performed
if (canBailout(fiber)) {
  // No changes
  return null;
}

// Did some work
fiber.flags |= PerformedWork;
```

DevTools "Highlight Updates" — `PerformedWork` flag'ga qaraydi.

**Memory layout:**

```
Fiber:
  flags: number (bitwise integer, 28+ bits ishlatiladi)
  subtreeFlags: number (bitwise integer)
```

Bit operations — fastest CPU operations (no allocation, branch prediction-friendly).

</details>

### Edge Cases

- **`flags = 0`**: No work to do for this fiber. Reused frequently (memoized component bailout).
- **`subtreeFlags = 0` but `flags != 0`**: This fiber needs work, but no descendants do. Common (terminal flagged fiber).
- **Deletion in middle of tree**: `parent.deletions = [child]` — child fiber sub-tree's effects handled during deletion (cleanup `useEffect`, ref detach).

### Follow-up savollar

- "Bitmask vs object?" — Bitmask — ikki number field (flags + subtreeFlags). Object {placement: true, update: true, ...} — ko'p property, V8 hidden class overhead. CPU bit ops property access'dan tezroq. Critical hot path.
- "More than 26 flags?" — Use 64-bit (BigInt) — performance penalty. R team designed flag count carefully.
- "Effect ordering — flags don't encode?" — Order determined by traversal (depth-first child-first). Flags only encode *what*.

</details>

---

### 7. Why Fiber over recursion — interruptibility, prioritization [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Stack Reconciler (R15)** recursion ishlatardi — `reconcile()` `reconcile()`'ni chaqirardi → JS call stack'ga yangi frame qo'shilardi → 1000 komponent — 1000 stack frame, JS engine'dan chiqib bo'lmasdi (yield yo'q). **Fiber (R16+)** — linked list + work loop — joriy fiber pozitsiyasini saqlab, kerak bo'lganda yield qilib, keyingi cycle'da davom etish mumkin. Bu **interruptibility** va **prioritization** asosi.

### To'liq tushuntirish

**Recursion muammolari:**

1. **Synchronous** — start to finish, no pause
2. **Stack growth** — har frame stack'ga qo'shiladi, depth high → memory + risk of overflow
3. **No yield** — browser idle bo'lishni xohlasa ham — ishlatish mumkin emas
4. **No prioritization** — kichik high-pri update vs katta low-pri update — sequential

**Fiber yechimi:**

1. **Iterative work loop** — `while`, no recursion
2. **Yieldable** — har 5ms `shouldYield()` true → loop chiqadi
3. **Resumable** — `nextUnitOfWork` saqlangan, keyingi cycle'da davom
4. **Prioritizable** — high-pri kelganda current low-pri tashlab, high-pri'dan boshlash
5. **Restartable** — render phase pure, kerak bo'lsa qayta boshlanadi

### Kod misoli

```typescript
// ============================================
// Recursion (Stack Reconciler R15) — broken model
// ============================================
function recursiveReconcile(parent, oldChildren, newChildren) {
  for (let i = 0; i < newChildren.length; i++) {
    const old = oldChildren[i];
    const newChild = newChildren[i];

    if (old.type === newChild.type) {
      updateProps(old, newChild);
      // ❌ Recursive call — adds stack frame
      recursiveReconcile(old, old.children, newChild.children);
    }
  }
}

// 5-level deep tree, 100 children each:
// Stack frames: 5 * 100 = 500
// All synchronous, no break

// ============================================
// Fiber (R16+) — work loop
// ============================================
function workLoop() {
  while (nextUnitOfWork && !shouldYield()) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }

  if (!nextUnitOfWork) {
    // Tree complete — commit
    commitRoot();
  } else {
    // Yielded — schedule resume
    scheduleCallback(NormalPriority, workLoop);
  }
}

function performUnitOfWork(fiber: Fiber): Fiber | null {
  beginWork(fiber);

  if (fiber.child) return fiber.child;  // descend

  let current: Fiber | null = fiber;
  while (current) {
    completeWork(current);
    if (current.sibling) return current.sibling;  // sibling
    current = current.return;  // parent
  }

  return null;  // tree complete
}

function shouldYield(): boolean {
  return performance.now() >= deadline;  // 5ms slice
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Browser frame budget:**

```
60fps target → 16.67ms per frame budget

Frame budget — JS, Style/Layout, Paint, Composite orasida taqsimlanadi.
React Scheduler `frameYieldMs = 5` — frame budget'ning bir qismini oladi,
qolganini browser task'lariga qoldiradi.
Beyond frame budget → frame drop, user notices "lag"
```

Fiber 5ms slice — frame budget ichida.

**`shouldYield` implementation:**

```typescript
let deadline = 0;
const yieldInterval = 5;  // ms

function shouldYield(): boolean {
  return performance.now() >= deadline;
}

function workLoop() {
  deadline = performance.now() + yieldInterval;

  while (nextUnitOfWork && !shouldYield()) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }

  // ... handle yielded or complete
}
```

`performance.now()` — sub-millisecond resolution timer. Fast (no syscall).

**Yield mechanism — MessageChannel:**

```typescript
// React Scheduler internal
const channel = new MessageChannel();
const port = channel.port2;

port.onmessage = function() {
  workLoop();
};

function scheduleCallback() {
  channel.port1.postMessage(null);  // schedule next slice
}

// MessageChannel — macrotask (task queue'da)
// Faster than setTimeout (4ms minimum delay yo'q)
// More reliable than requestIdleCallback (Safari support)
```

**Comparison: setTimeout vs MessageChannel vs requestIdleCallback:**

| Method | Delay | Cross-browser | Predictable |
|--------|-------|----------------|-------------|
| `setTimeout(fn, 0)` | 4ms minimum | ✅ All | Yes |
| `MessageChannel` | ~0ms | ✅ All | Yes |
| `requestIdleCallback` | Variable | ✅ Safari 16.4+ (2023) | No |
| `requestAnimationFrame` | Until next paint | ✅ All | No (paint-tied) |

React: MessageChannel for fast yield-resume.

**Recursion stack overflow risk:**

```typescript
// Deep tree (e.g., commenting system, recursive data)
function Comment({ comment }) {
  return (
    <div>
      <p>{comment.text}</p>
      {comment.replies.map(r => <Comment comment={r} />)}  // recursive
    </div>
  );
}

// 10,000 deep nested → recursive Reconciler: stack overflow
// Fiber: linked list traversal, O(1) stack regardless of depth
```

**Konseptual taqqoslash (sodda model):**

```typescript
// Katta tree re-render with state change
// 5ms render budget per frame

// Stack Reconciler (R15):
// Sync, blocking — browser butun render davomida bloklanadi
// Long task — frame drop, input lag

// Fiber (R16+):
// Yield har 5ms — chunklar
// Total wall time biroz uzunroq (yield-resume overhead)
// Lekin: browser har slice orasida paint/input handle qila oladi
// Frame drops: 0
// User input mid-render handled ✅
```

Fiber — biroz **uzunroq total**, lekin **smooth** UX.

**Render abort:**

```typescript
// Low priority render mid-flight
let lowPriRender = scheduleRender(LowPriorityLane);
// 50% complete...

// User input — high priority
let highPriRender = scheduleRender(SyncLane);

// React: abandon low-pri WIP, start high-pri WIP from current
abandonWorkInProgress();
const newWIP = createWorkInProgress(currentTree);
// High-pri render proceeds

// After high-pri commit, low-pri can be re-attempted
```

Pure render — abort safe (no DOM mutated yet).

**Why this couldn't work in R15:**

R15 reconcile recursively mutated DOM during reconciliation (no separate commit phase). Mid-reconcile abort → partial DOM mutation → tearing.

R16+ separated render (pure) and commit (mutation). Render abort safe; commit atomic.

**`Fiber` term origin:**

- **Coroutine / fiber** — language concept (Lua, Ruby fibers): cooperative threading
- React Fiber — JS doesn't have native fibers, but Fiber emulates: pauseable computation
- "Fiber" hints: thinner than thread, lightweight, cooperative

</details>

### Edge Cases

- **Long sync work in component**: `for (let i = 0; i < 1e9; i++)` in render — Fiber yieldsiz uzun task, browser blocks. Yield only between fibers.
- **Synchronous priority**: `flushSync(() => setState(x))` — bypasses Fiber yield, runs synchronously to completion (legacy compatibility).
- **Suspense fallback**: When promise thrown, current WIP abandoned, fallback rendered fast (avoid janky loading state).

### Follow-up savollar

- "Why 5ms (not 16ms) yield interval?" — 5ms gives buffer for browser tasks (input, animation). 16ms would consume entire frame budget.
- "Microtask vs macrotask for yield?" — MessageChannel is macrotask-like (between frames). Microtask (Promise) doesn't yield to browser paint.
- "Web Workers replacement?" — React work is synchronous main thread. Web Workers — separate thread, but no DOM access (only data). Fiber concurrent rendering is "soft concurrent".

</details>

---

### 8. Output: Fiber traversal order [Output] [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi komponent uchun Fiber traversal order (depth-first) qanday bo'ladi? Va `console.log` ketma-ketligi nima?

```tsx
function App() {
  console.log("App render");
  return (
    <div>
      <Header />
      <Main />
    </div>
  );
}

function Header() {
  console.log("Header render");
  return (
    <header>
      <Logo />
      <Menu />
    </header>
  );
}

function Logo() { console.log("Logo render"); return <img />; }
function Menu() { console.log("Menu render"); return <ul />; }

function Main() {
  console.log("Main render");
  return <main><Article /></main>;
}

function Article() { console.log("Article render"); return <article />; }

// Initial mount: createRoot(...).render(<App />)
```

### Javob

**Render order (beginWork) — top-down depth-first:**

```
App render
Header render
Logo render
Menu render
Main render
Article render
```

**Tushuntirish (Fiber traversal):**

```
1. beginWork(App)            → "App render"
   App.child = div            → descend

2. beginWork(div)             → no log (HostComponent)
   div.child = Header         → descend

3. beginWork(Header)          → "Header render"
   Header.child = header      → descend

4. beginWork(header)          → no log
   header.child = Logo        → descend

5. beginWork(Logo)            → "Logo render"
   Logo.child = img           → descend

6. beginWork(img)             → no log (leaf)
   completeWork(img)          → no children, complete
   sibling? null              → return up

7. completeWork(Logo)         → complete
   Logo.sibling = Menu        → traverse sibling

8. beginWork(Menu)            → "Menu render"
   Menu.child = ul            → descend

9. beginWork(ul)              → no log
   completeWork(ul)           → complete
   sibling? null              → return

10. completeWork(Menu)        → complete
    Menu.sibling? null
    return up to header

11. completeWork(header)      → complete
    header.sibling? null
    return up to Header

12. completeWork(Header)      → complete
    Header.sibling = Main     → traverse sibling (Main)

13. beginWork(Main)            → "Main render"
    Main.child = main         → descend

14. beginWork(main)           → no log
    main.child = Article      → descend

15. beginWork(Article)        → "Article render"
    Article.child = article   → descend

16. beginWork(article)        → leaf
    completeWork(article)     → complete
    sibling? null

17. completeWork(Article)     → complete
... and so on, completing back up to App
```

**Visual order:**

```
        App
         │
        div
       /   \
   Header  Main
    /  \      \
  Logo  Menu   Main DOM
   |     |      |
  img    ul   Article
                │
              article
```

**Pre-order DFS:** App → Header → Logo → Menu → Main → Article (parent before children, child before sibling).

<details>
<summary><strong>Deep Dive</strong></summary>

**Two-pass traversal — beginWork + completeWork:**

```typescript
function performUnitOfWork(fiber) {
  // PASS 1: beginWork — render this fiber, reconcile children
  const next = beginWork(fiber);  // returns child fiber if has children
  fiber.memoizedProps = fiber.pendingProps;

  if (next === null) {
    // No children — complete bottom up
    completeUnitOfWork(fiber);
  } else {
    return next;  // descend
  }
}

function completeUnitOfWork(fiber) {
  let current = fiber;
  while (current) {
    completeWork(current);  // PASS 2: finalize, prepare DOM updates

    if (current.sibling) {
      return current.sibling;  // traverse sibling
    }
    current = current.return;  // up to parent
  }
}
```

**`beginWork` vs `completeWork`:**

| Phase | Direction | Operation |
|-------|-----------|-----------|
| `beginWork` | Top-down | Component render, reconcile children, create fibers |
| `completeWork` | Bottom-up | DOM instance create (HostComponent), props ready, append children |

**`completeWork` — DOM creation:**

```typescript
function completeWork(workInProgress: Fiber) {
  switch (workInProgress.tag) {
    case HostComponent:  // 'div', 'span', etc.
      if (workInProgress.alternate === null) {
        // Mount — create DOM instance
        const instance = createInstance(
          workInProgress.type,  // 'div'
          workInProgress.pendingProps,
          rootContainerInstance
        );
        workInProgress.stateNode = instance;

        // Append all children that have been completed
        appendAllChildren(instance, workInProgress);
      } else {
        // Update — schedule props update
        prepareUpdate(workInProgress);
      }
      break;

    case HostText:
      workInProgress.stateNode = createTextInstance(workInProgress.pendingProps);
      break;

    case FunctionComponent:
    case ClassComponent:
      // Already done in beginWork (component fn called)
      break;

    // ... other tags
  }

  // Aggregate subtreeFlags
  bubbleProperties(workInProgress);
}
```

**Why bottom-up DOM creation:**

DOM tree builds bottom-up: children created first, appended to parent. Parent created next, appended to grandparent. Etc.

```typescript
// Pseudocode for completeWork(parent):
const parentInstance = createDOMElement(parent.type);
let child = parent.child;
while (child) {
  parentInstance.appendChild(child.stateNode);  // child's DOM
  child = child.sibling;
}
parent.stateNode = parentInstance;
// Parent now has children attached, ready to be appended to grandparent
```

**Yieldable check:**

```typescript
function workLoop() {
  while (nextUnitOfWork && !shouldYield()) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }
}
```

Yield can happen at any fiber boundary (between performUnitOfWork calls). Resumable from `nextUnitOfWork`.

**Render order vs commit order:**

- **Render** (beginWork): App → Header → Logo → Menu → Main → Article (top-down)
- **Complete** (completeWork): img → Logo → ul → Menu → header → Header → article → Article → main → Main → div → App (bottom-up)
- **Commit** (mutation): Bottom-up (children DOM before parent DOM)
- **Effects** (useEffect): Bottom-up (child effects before parent — child unmount cleanup happens before parent unmount)

**`useEffect` order (commit):**

```tsx
// Same tree
function App() {
  useEffect(() => console.log("App effect"), []);
  return <Header />;
}
function Header() {
  useEffect(() => console.log("Header effect"), []);
  return <Logo />;
}
function Logo() {
  useEffect(() => console.log("Logo effect"), []);
  return <img />;
}

// Output:
// Logo effect    ← child first
// Header effect
// App effect     ← parent last
```

Effects depth-first post-order (child effects before parent effects).

**Concurrent rendering — same order:**

Yield-resume doesn't change order. Fiber traversal pre-order DFS regardless of yields.

</details>

### Edge Cases

- **Sibling with multiple children**: Traverse first sibling fully (depth-first), then next sibling.
- **Empty parent**: `<div />` — no children, descend skipped, complete immediately.
- **Conditional render**: `{cond && <Component />}` — `false`/`null` skipped (no fiber created).

### Follow-up savollar

- "Effects bottom-up — why?" — Cleanup invariant: child unmount before parent unmount. Parent might depend on child setup; cleanup reverse.
- "Render order break with Suspense?" — Suspense fallback during render — sub-tree render aborted, fallback rendered. Order changes for that branch.
- "useLayoutEffect vs useEffect order?" — Both bottom-up. useLayoutEffect synchronously after Layout sub-phase; useEffect async after paint.

</details>

---

### 9. Component tree vs Fiber tree — farq nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Component tree** — JSX hierarchy (developer ko'radigan tree, `<App><Header /></App>`). **Fiber tree** — React internal data structure: har komponent + Host element + Fragment + Provider + Memo + Suspense uchun **alohida Fiber node**. Component tree'dagi 1 ta komponent — Fiber tree'da 1+ fiber. Fiber tree component tree'dan **kengroq va detallashtirilgan**.

### To'liq tushuntirish

**Misol:**

```tsx
function App() {
  return (
    <Provider value={x}>
      <Suspense fallback={<L />}>
        <MainContent />
      </Suspense>
    </Provider>
  );
}
```

**Component tree (JSX):**
```
App
└── Provider
    └── Suspense
        └── MainContent
```

**Fiber tree:**
```
HostRoot                       (tag 3)
└── App                         (FunctionComponent, tag 0)
    └── ContextProvider          ← Provider
        └── SuspenseComponent     ← Suspense
            └── MainContent       (FunctionComponent, tag 0)
                └── (its children — div, ...)
```

### Kod misoli

```tsx
function App() {
  return (
    <>
      <Header>
        <Logo />
      </Header>
      <Main />
    </>
  );
}

// Component tree:
//   App
//   ├── Header
//   │   └── Logo
//   └── Main

// Fiber tree:
//   HostRoot
//   └── App (FunctionComponent)
//       └── Fragment                  ← <> </> creates Fragment fiber
//           ├── Header (FunctionComponent)
//           │   └── header (HostComponent)
//           │       └── Logo (FunctionComponent)
//           │           └── img (HostComponent)
//           └── Main (FunctionComponent)
//               └── main (HostComponent)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why Fiber tree more detailed:**

Fiber tree captures every aspect of rendering:
- **Component fibers** — your function/class components
- **Host fibers** — DOM elements (`div`, `button`, ...)
- **Wrapper fibers** — Fragment, Suspense, Provider, etc.
- **Memo wrappers** — `React.memo()` adds MemoComponent fiber
- **ForwardRef** — adds ForwardRef fiber

**Each fiber has its own work:**
- Lifecycle (mount, update, unmount)
- Hooks state (memoizedState linked list)
- Props
- Effects flags

**Component tree is conceptual, Fiber tree is implementation:**

```tsx
// Component tree perspective
const memoButton = React.memo(Button);
function App() {
  return <memoButton onClick={handler} />;
}

// You think:
// App
// └── Button (with props: onClick)

// Fiber tree:
// App (FunctionComponent)
// └── MemoComponent ← memo wrapper fiber
//     └── Button (FunctionComponent, tag 0)
//         └── button (HostComponent)
```

**Memo bailout works at MemoComponent level:**

```typescript
function updateMemoComponent(current, workInProgress) {
  const compare = workInProgress.type.compare;
  const prevProps = current.memoizedProps;
  const nextProps = workInProgress.pendingProps;

  if (compare(prevProps, nextProps)) {
    // Bailout — don't render Button child
    return bailoutOnAlreadyFinishedWork(current, workInProgress);
  }

  // Render Button as usual
  const innerComponent = workInProgress.type.type;
  // ...
}
```

**`forwardRef` adds wrapper fiber:**

```tsx
const Input = forwardRef<HTMLInputElement>(function Input(props, ref) {
  return <input ref={ref} {...props} />;
});

// Fiber tree:
// App
// └── ForwardRef ← wrapper
//     └── Input (FunctionComponent) ← inner function
//         └── input (HostComponent)
```

R19 — ref oddiy prop, no `forwardRef` wrapper, simpler tree:

```tsx
function Input({ ref, ...props }: { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...props} />;
}

// Fiber tree:
// App
// └── Input (FunctionComponent) ← no wrapper
//     └── input (HostComponent)
```

**Provider fiber:**

```tsx
const ThemeContext = createContext("light");

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Page />
    </ThemeContext.Provider>
  );
}

// Fiber tree:
// App (FunctionComponent)
// └── ContextProvider                  ← own fiber
//     └── Page (FunctionComponent)
```

ContextProvider fiber tracks: previous value, propagation, context updates.

**Fragment fiber:**

```tsx
// Without Fragment — single root rule
function App() {
  return <>
    <h1>A</h1>
    <p>B</p>
  </>;
}

// Fiber tree:
// App
// └── Fragment                ← wrapper, no DOM
//     ├── h1 (HostComponent)
//     └── p (HostComponent)
```

Fragment — Fiber, no DOM. children become DOM siblings of Fragment's parent.

**Component tree visualization (DevTools):**

React DevTools "Components" panel shows:
- Component fibers (Function, Class)
- Memo, ForwardRef wrappers (often collapsed)
- Suspense, Provider boundaries
- Hides Fragment (cluttering)
- Hides HostText
- Hides HostComponent details

User-facing: simplified component tree.
Internal: full Fiber tree with all wrappers.

**Setting parent — `return` pointer:**

```typescript
// Each fiber.return points to its parent fiber:
fragment.return = App;
h1.return = Fragment;
p.return = Fragment;
// h1.return ≠ App (it's Fragment, not App)
```

Reconciler walks `return` to bubble up effects, propagate context, etc.

**Tree depth comparison:**

```
Component tree depth: developer ko'radigan komponent ierarxiyasi (App, Page, Section)
Fiber tree depth: yuqori — HostRoot, Fragment, Provider, Memo wrapper'lar har biri alohida fiber
```

Real React app'da Fiber tree komponent tree'dan kattaroq — har JSX element + wrapper (Fragment, Memo, ForwardRef, Provider) uchun alohida Fiber.

</details>

### Edge Cases

- **`<Component />` returns null**: Fiber created with `tag = FunctionComponent`, but no children. Just a leaf.
- **Multiple roots**: Each `createRoot` — separate HostRoot fiber, separate tree.
- **Portal — different DOM, same Fiber tree**: `createPortal` fiber's children render to alternate DOM, but Fiber tree maintains parent-child link.

### Follow-up savollar

- "Component tree DevTools'da ko'rinadigan tree-mi?" — Ha, simplified version. Internal HostComponent, Fragment, etc. yashirin.
- "Fiber count app size'ga bog'liqmi?" — Linear: komponent soni + host element + wrapper fiber'lar (Fragment, Memo, Provider) soni.
- "Suspense fallback rendered — Fiber tree o'zgaradimi?" — Ha. Primary children replaced with fallback subtree (different fibers).

</details>

---

### 10. FiberRoot vs Fiber — root tugun nima va nima uchun alohida? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**FiberRoot** (yoki HostRoot) — Fiber tree'ning root container'i, **alohida obyekt** (Fiber emas — Fiber'ga ham link). Saqlaydi: `current` Fiber pointer (committed tree), `pendingLanes` (kutilayotgan ishlar), `callbackPriority`, `containerInfo` (DOM node), `onRecoverableError` callback. **Fiber** — komponent/element uchun work unit. FiberRoot — **scheduler ↔ tree bridge**.

### To'liq tushuntirish

**FiberRoot strukturasi:**

```typescript
interface FiberRoot {
  // Tree pointers
  current: Fiber;           // committed tree root
  finishedWork: Fiber | null;  // workInProgress before commit

  // Container
  containerInfo: any;       // DOM container (createRoot argument)
  pendingChildren: any;

  // Scheduling
  pendingLanes: Lanes;      // bitmap of pending work
  suspendedLanes: Lanes;
  pingedLanes: Lanes;
  expirationTimes: number[];
  callbackNode: SchedulerCallback | null;
  callbackPriority: PriorityLevel;

  // Concurrent
  isDehydrated: boolean;    // SSR pre-hydration
  hydrateLanes: Lanes;

  // Error handling (R19)
  onCaughtError: (error: Error, info: ErrorInfo) => void;
  onUncaughtError: (error: Error, info: ErrorInfo) => void;
  onRecoverableError: (error: Error, info: ErrorInfo) => void;

  // Identifier prefix (useId)
  identifierPrefix: string;
}
```

### Kod misoli

```tsx
// createRoot creates FiberRoot
const container = document.getElementById("root");
if (!container) throw new Error("Root container missing");

const root = createRoot(container, {
  identifierPrefix: "app-",
  onRecoverableError: (e) => console.warn(e),
});

// Internal:
// fiberRoot = {
//   current: Fiber(HostRoot),  // initial empty tree
//   containerInfo: <div id="root">,
//   pendingLanes: 0,
//   identifierPrefix: "app-",
//   ...
// }

root.render(<App />);
// fiberRoot.current → updated tree (App rendered)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**FiberRoot vs HostRoot Fiber:**

```
FiberRoot (root container)
   ↓ current
Fiber(HostRoot, tag: 3)   ← actual root Fiber
   ↓ child
Fiber(App, tag: FunctionComponent)
   ↓ child
Fiber(div, tag: HostComponent)
   ...
```

FiberRoot — outer container. HostRoot Fiber — first node in tree. They reference each other:
- `fiberRoot.current = hostRootFiber`
- `hostRootFiber.stateNode = fiberRoot`

**Why separate from Fiber:**

1. **Scheduling state** — pendingLanes, callbacks, priorities. Not per-component.
2. **Container info** — DOM container (createRoot argument).
3. **Lifecycle hooks** — onCaughtError, onUncaughtError per app.
4. **Multiple roots** — bir page'da bir nechta `createRoot` call → bir nechta FiberRoot.

**Multiple roots example:**

```tsx
// Microfrontend
const widgetRoot = createRoot(document.getElementById("widget")!);
widgetRoot.render(<ChatWidget />);

const appRoot = createRoot(document.getElementById("app")!);
appRoot.render(<App />);

// 2 FiberRoot'lar — alohida pendingLanes, alohida scheduling
// Bir-biri bilan independent (no cross-root batching default)
```

**`finishedWork`:**

Render Phase tugagach, workInProgress tree `finishedWork`'ga assign qilinadi:

```typescript
// React internal
function commitRoot(root) {
  const finishedWork = root.finishedWork;  // workInProgress tree
  root.current = finishedWork;            // commit (swap)
  root.finishedWork = null;
  // Old current → workInProgress (alternate)
}
```

**`identifierPrefix` for useId:**

```tsx
const root = createRoot(container, { identifierPrefix: "app-" });
// useId() → "app-:r0:" (instead of ":r0:")
// Useful for SSR + multiple frameworks (avoid collision)
```

**Hydration state:**

```tsx
const root = hydrateRoot(container, <App />);
// FiberRoot.isDehydrated = true (initially)
// After hydration complete → false
```

**Scheduler integration:**

```typescript
// React internal
function scheduleUpdateOnFiber(fiber, lane) {
  const root = getRootForUpdatedFiber(fiber);
  root.pendingLanes |= lane;  // mark lane as pending

  // Schedule callback at lane's priority
  if (root.callbackNode === null) {
    root.callbackNode = Scheduler.scheduleCallback(
      lanePriorityToSchedulerPriority(lane),
      () => performWorkOnRoot(root)
    );
  }
}
```

**Render mode:**

```typescript
// Legacy mode (R17 ReactDOM.render)
fiberRoot.tag = LegacyRoot;  // sync rendering

// Concurrent mode (R18+ createRoot)
fiberRoot.tag = ConcurrentRoot;  // concurrent rendering
```

R18'dan default ConcurrentRoot. `ReactDOM.render` (deprecated) — LegacyRoot.

</details>

### Edge Cases

- **`root.unmount()`**: Cleanup — komponentlar unmounted, FiberRoot freed.
- **Calling `root.render` multiple times**: Re-renders with new element, FiberRoot reused.
- **Same container, two `createRoot` calls**: Warning + first root replaced.

### Follow-up savollar

- "Server-side render ham FiberRoot ishlatadi-mi?" — `renderToString` simpler model (no committed tree). `renderToReadableStream` — bor (streaming state).
- "FiberRoot memory size?" — Ko'p property'larga ega (scheduling, lanes, error callbacks). Per-root overhead.
- "Multi-root batching?" — R18'dan default no. Cross-root setState alohida cycle. `flushSync` per root.

</details>

---

### 11. `stateNode` — Fiber → DOM/Instance bridge [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`fiber.stateNode` — Fiber'ning **runtime instance** referensi. **HostComponent** uchun — DOM Element (`<div>`, `<button>`). **ClassComponent** uchun — class instance (`new MyClass(props)`). **FunctionComponent** — `null` (state hooks `memoizedState` linked list'da). **HostRoot** — FiberRoot. Reconciler `stateNode`'ga DOM mutation, ref attach, lifecycle method chaqiradi.

### Kod misoli

```typescript
// Fiber types va stateNode
const hostFiber = {
  tag: HostComponent,
  type: "div",
  stateNode: <div>,  // Real DOM Element
};

const classFiber = {
  tag: ClassComponent,
  type: MyClass,
  stateNode: myClassInstance,  // class instance
};

const functionFiber = {
  tag: FunctionComponent,
  type: MyFunctionComponent,
  stateNode: null,  // function — no instance, hooks linked list ichida
};

const rootFiber = {
  tag: HostRoot,
  stateNode: fiberRoot,  // FiberRoot reference
};
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Commit phase ishlatish:**

```typescript
// Mutation phase
function commitMutation(fiber) {
  if (fiber.flags & Placement) {
    // Insert: parent.appendChild(fiber.stateNode)
    parentFiber.stateNode.appendChild(fiber.stateNode);
  }
  if (fiber.flags & Update) {
    // Update DOM properties on stateNode
    updateDOM(fiber.stateNode, fiber.memoizedProps, fiber.pendingProps);
  }
  if (fiber.flags & Deletion) {
    // Remove from parent
    parentFiber.stateNode.removeChild(fiber.stateNode);
  }
}
```

**Refs — stateNode reference:**

```tsx
function MyComponent() {
  const ref = useRef<HTMLDivElement>(null);

  // After commit:
  // ref.current = fiber.stateNode (DOM <div>)

  return <div ref={ref}>Hello</div>;
}
```

**Class instance lifecycle:**

```tsx
class MyClass extends React.Component {
  componentDidMount() {
    console.log(this);  // ← `this` === fiber.stateNode (instance)
  }
}

// React internal:
// fiber.stateNode.componentDidMount();
```

**Function components — no stateNode:**

```tsx
function MyFunc() {
  const [count, setCount] = useState(0);
  // Where is `count`?
  // → fiber.memoizedState linked list (hooks)
  // NOT in stateNode (which is null)
}
```

**HostText — text DOM node:**

```typescript
const textFiber = {
  tag: HostText,
  stateNode: document.createTextNode("Hello"),  // Text node
};
```

**Portal — stateNode is portal target:**

```tsx
import { createPortal } from "react-dom";

<>
  {createPortal(<Modal />, document.body)}
</>

// Fiber tree:
// Fiber(Portal, tag: HostPortal, stateNode: document.body)
//   ↓ child
// Fiber(Modal)
```

**Suspense — stateNode tracks state:**

```typescript
const suspenseFiber = {
  tag: SuspenseComponent,
  stateNode: {
    dehydrated: null,  // SSR
    treeContext: null,
    retryLane: 0,
  },
};
```

</details>

### Edge Cases

- **stateNode null in function components**: Bu bug emas — function komponent stateless (state hooks'da).
- **stateNode access in render — anti-pattern**: stateNode commit'dan keyin assign qilinadi. Render'da access — old reference.
- **stateNode garbage collection**: Component unmount → DOM removed → stateNode dereferenced.

### Follow-up savollar

- "Why class components have stateNode but function don't?" — Class instance — long-lived (lifecycle methods). Function — pure call, state external (hooks).
- "Can I access stateNode directly?" — DevTools yes. Production code — `useRef` (proper API).
- "Memory leak via stateNode?" — Closure references DOM node — yes (clear on unmount).

</details>

---

### 12. `memoizedState` linked list — hooks storage internals [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Function component'da `useState`, `useEffect`, `useMemo` va boshqa hooks **fiber.memoizedState** linked list'da saqlanadi — har hook bitta node. Render time'da React linked list traversal qiladi (`current.next`), call order'iga mos hook'ni topadi. **Rules of Hooks** (top-level only, no conditional) — shu invariant'dan kelib chiqadi: index mismatch → bug.

### Kod misoli

```tsx
function MyComponent() {
  const [count, setCount] = useState(0);     // hook 1
  const [name, setName] = useState("Ali");   // hook 2

  useEffect(() => {                          // hook 3
    console.log(count);
  }, [count]);

  const doubled = useMemo(() => count * 2, [count]);  // hook 4

  return <p>{count}, {name}, {doubled}</p>;
}

// fiber.memoizedState linked list:
// hook1: { state: 0, queue: {...}, next: hook2 }
// hook2: { state: "Ali", queue: {...}, next: hook3 }
// hook3: { deps: [count], create: fn, destroy: fn, next: hook4 }
// hook4: { state: 0, deps: [count], next: null }
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Hook node structure:**

```typescript
interface Hook {
  memoizedState: any;   // hook value (state, deps, etc.)
  baseState: any;       // initial state
  baseQueue: Update | null;  // pending updates
  queue: UpdateQueue | null;
  next: Hook | null;    // next hook in list
}
```

**Mount path — `mountState`:**

```typescript
function mountState(initialState) {
  const hook = mountWorkInProgressHook();  // create + append to list
  hook.memoizedState = typeof initialState === "function"
    ? initialState()
    : initialState;
  hook.baseState = hook.memoizedState;
  hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: hook.memoizedState,
  };

  const dispatch = dispatchSetState.bind(null, currentFiber, hook.queue);
  hook.queue.dispatch = dispatch;

  return [hook.memoizedState, dispatch];
}
```

**Update path — `updateState`:**

```typescript
function updateState(initialState) {
  const hook = updateWorkInProgressHook();  // get next hook from list
  let newState = hook.memoizedState;

  // Apply pending updates from queue
  if (hook.queue.pending) {
    let update = hook.queue.pending.next;
    do {
      const action = update.action;
      newState = typeof action === "function"
        ? action(newState)
        : action;
      update = update.next;
    } while (update !== hook.queue.pending.next);

    hook.queue.pending = null;
  }

  hook.memoizedState = newState;
  return [newState, hook.queue.dispatch];
}
```

**Why call order matters:**

```tsx
function Bad() {
  if (cond) {
    const [a] = useState(1);  // ← conditional
  }
  const [b] = useState(2);    // ← b sometimes index 0, sometimes 1
}

// Render 1 (cond=true):
// hook[0] = a state
// hook[1] = b state

// Render 2 (cond=false):
// hook[0] = b state  ← INDEX MISMATCH!
// React: "Rendered fewer hooks than expected"
```

**Hooks dispatcher swap:**

```typescript
let HooksDispatcher;

function renderWithHooks(fiber, Component, props) {
  if (fiber.alternate === null) {
    HooksDispatcher = HooksDispatcherOnMount;
  } else {
    HooksDispatcher = HooksDispatcherOnUpdate;
  }

  // ReactCurrentDispatcher.current = HooksDispatcher;
  // useState/useEffect chaqiruvi shu dispatcher orqali

  const children = Component(props);

  // ReactCurrentDispatcher.current = null;
  return children;
}

const HooksDispatcherOnMount = {
  useState: mountState,
  useEffect: mountEffect,
  useMemo: mountMemo,
  // ...
};

const HooksDispatcherOnUpdate = {
  useState: updateState,
  useEffect: updateEffect,
  useMemo: updateMemo,
  // ...
};
```

**Custom hook — same linked list:**

```tsx
function useCustomHook() {
  const [state, setState] = useState(0);  // adds to fiber's memoizedState
  return state;
}

function Component() {
  const a = useCustomHook();  // hook 1 in list
  const b = useCustomHook();  // hook 2 in list (different state)
  const [c] = useState("c");  // hook 3
  // memoizedState: hook1 → hook2 → hook3 → null
}
```

Custom hook — flat extension. Each `useState` inside adds a hook to caller's fiber.

**Effect storage:**

```typescript
interface EffectHook {
  memoizedState: {
    tag: HookFlags;       // Passive | Layout | Insertion
    create: () => void;
    destroy: () => void | null;
    deps: any[] | null;
    next: EffectHook | null;  // effect list (separate from hooks list)
  };
  next: Hook;  // next hook
}
```

Effects — both in hooks list AND separate "effect list" (per-fiber).

**`useReducer` shares queue with `useState`:**

```typescript
function mountReducer(reducer, initialArg) {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = initialArg;
  hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: reducer,  // ← reducer here
    lastRenderedState: initialArg,
  };
  return [hook.memoizedState, dispatch];
}

// useState internally calls useReducer with basicStateReducer
function basicStateReducer(state, action) {
  return typeof action === "function" ? action(state) : action;
}
```

</details>

### Edge Cases

- **useEffect during render**: Hook node created but `create` queued for commit. Render-time hook order preserved.
- **Hook re-order (component update)**: Same fiber.memoizedState reused. Different hook order in code → mismatch.
- **`useState` initializer in StrictMode**: Initializer 2x in dev (purity check). Hook state set to last value.

### Follow-up savollar

- "Why linked list, not array?" — Array would require pre-allocation. Linked list — dynamic, supports custom hooks naturally.
- "Memory size of hooks?" — Har hook object (memoizedState, queue, next, ...) heap'da alohida allocation. Ko'p hooks + ko'p komponent — sezilarli memory.
- "Can I access fiber.memoizedState?" — DevTools yes. Production code — no public API.

</details>

---

### 13. Output: alternate swap visualization [Output] [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi setState chaqiruvidan keyin `current` va `workInProgress` Fiber tree'lar qanday ko'rinadi?

```tsx
function App() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <span>Hello</span>
    </div>
  );
}

// Initial render → click button (count: 0 → 1)
```

### Javob

**Initial mount (after first commit):**

```
FiberRoot
   ↓ current
Fiber(HostRoot)
   ↓ child
Fiber(App)
   ↓ child
Fiber(div)
   ↓ child
Fiber(button) → sibling → Fiber(span)
   ↓ child                    ↓ child
Fiber(text "0")            Fiber(text "Hello")

(workInProgress = null, no pending render)
```

**During Render Phase (after click, count: 0 → 1):**

```
FiberRoot
   ↓ current                    ↓ workInProgress
Fiber(HostRoot)              Fiber(HostRoot, alternate)
   ↓ child                       ↓ child
Fiber(App, count=0)          Fiber(App, count=1, alternate)
   ↓                              ↓ child
   ...                          Fiber(div, alternate)
                                   ↓ child
                                 Fiber(button, alternate) → ...
                                   ↓ child
                                 Fiber(text "1", alternate)

current — eski tree (count=0, displayed)
workInProgress — yangi tree (count=1, being built)
Each fiber has `alternate` pointer to its pair
```

**After Commit (current swap):**

```
FiberRoot
   ↓ current
Fiber(HostRoot)  ← was workInProgress
   ↓ child
Fiber(App, count=1)  ← updated tree
   ...

(Old current tree → now workInProgress, ready for next render)
(DOM updated: button text "0" → "1")
```

### Tushuntirish

**Double buffering mexanizmi:**

1. **Render Phase**: workInProgress tree quriladi, current tree tegmaydi
2. **Commit Phase**: `root.current = workInProgress` (swap)
3. **Old current → workInProgress** (alternate, ready for next render)

Foyda:
- Render abort qilingan paytda current toza qoladi (no UI tearing)
- Memory efficient — alternate'lar reused, no GC

<details>
<summary><strong>Deep Dive</strong></summary>

**Alternate pointer initialization:**

```typescript
// First render — no alternate
function createWorkInProgress(current, pendingProps) {
  let workInProgress = current.alternate;

  if (workInProgress === null) {
    // First render — create new fiber
    workInProgress = createFiber(current.tag, pendingProps, current.key);
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    // Reuse existing alternate
    workInProgress.pendingProps = pendingProps;
    workInProgress.flags = NoFlags;
    workInProgress.subtreeFlags = NoFlags;
  }

  return workInProgress;
}
```

**Commit swap:**

```typescript
function commitRoot(root) {
  const finishedWork = root.finishedWork;

  // 1. Before mutation
  commitBeforeMutationEffects(finishedWork);

  // 2. Mutation (DOM changes)
  commitMutationEffects(finishedWork);

  // 3. Switch trees
  root.current = finishedWork;  // ← SWAP

  // 4. Layout (sync effects)
  commitLayoutEffects(finishedWork);
}
```

**Memory layout:**

```
Render 1 (initial):
  fiber.alternate = null
  workInProgress created from scratch

Render 2:
  fiber.alternate = oldFiber (from previous workInProgress)
  Reuse: workInProgress = fiber.alternate
  Reset flags, update props

Render 3:
  fiber.alternate = oldFiber (from render 1)
  Reuse alternate again

(Continuous reuse — no allocation per render)
```

**Why double buffering, not single tree:**

```
Single tree mutation (theoretical):
  - Mutate fiber in-place during render
  - Render error → tree corrupted
  - Concurrent abort → invalid state
  - User sees half-rendered UI

Double buffering:
  - workInProgress isolated
  - Error → discard workInProgress, current intact
  - Abort safe — current never touched until commit
  - Atomic swap — no tearing
```

**Bailout — alternate reuse without mutation:**

```typescript
function bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes) {
  if (current !== null) {
    // Reuse current's children (no need to render)
    workInProgress.child = current.child;

    // But check if children have pending work
    if (includesSomeLane(workInProgress.childLanes, renderLanes)) {
      // Re-render children
      cloneChildFibers(current, workInProgress);
      return workInProgress.child;
    }
  }

  // No work needed
  return null;
}
```

**StrictMode 2x render — alternate handling:**

```
Render 1 (StrictMode dev):
  workInProgress[1] = render(...)
  workInProgress[2] = render(...)  ← second pass (Strict Mode)
  Both should produce identical output (purity check)

Commit:
  root.current = workInProgress[2]
```

</details>

### Edge Cases

- **First render — no alternate**: Created on demand. Subsequent renders — reused.
- **Suspense throw — alternate state**: Render aborted, workInProgress discarded, alternate marker preserved.
- **Concurrent abort**: workInProgress reset (not committed), current intact, alternate reused.

### Follow-up savollar

- "What's in workInProgress that's not in current?" — Pending props (yet to be committed), updated state (mid-render), new flags.
- "Memory overhead of alternate?" — 2x Fiber memory (current + alternate). Katta tree'larda sezilarli, lekin GC pressure kamayadi (recycle).
- "Why doesn't React just clone fibers?" — Performance: clone has GC pressure. Alternate reuse — stable allocation.

</details>

---

## QISM B: Reconciliation Algorithm

<a id="qism-b"></a>

### 14. Reconciliation algorithm — O(n) sabab nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Reconciliation** — eski Fiber tree va yangi element tree'ni solishtirib (diff), DOM o'zgarishlarni minimal qilib aniqlash. Naive **O(n³)** algoritm — ikki tree'ni to'liq tahlil qilish (Levenshtein-like). React **O(n)** ishlatadi — 2 ta heuristic asosida: (1) **different element types — different subtrees** (same type → update, different → replace), (2) **list keys for stable identity**. Bu heuristic'lar real-world UI patterns'ga mos.

### To'liq tushuntirish

**Naive tree diff: O(n³):**

Optimal tree-to-tree diff algorithm complexity is O(n³) where n = number of nodes. For 1000 nodes — billion operations.

**React heuristics (O(n)):**

1. **Type changes → unmount/mount entire subtree** — same type position-wise → update, different type → replace
2. **List keys** — `key` prop for explicit identity (prevent unnecessary remount in reorder)

**Tradeoff:**
- Heuristics imperfect — sometimes suboptimal moves
- But near-O(n) performance for typical UI updates

### Kod misoli

```tsx
// Heuristic 1: Type comparison
// Old:
<div>
  <Counter />
</div>

// New:
<div>
  <Counter />     ← same type at same position → UPDATE
</div>
// Result: Counter fiber kept, props updated

// vs.

// Old:
<div>
  <Counter />
</div>

// New:
<span>
  <Counter />
</span>
// Result: span ≠ div → UNMOUNT div tree, MOUNT span tree
//         Counter inside also unmounted/remounted (state lost!)
```

```tsx
// Heuristic 2: Keys
// Without keys
const items = ["A", "B", "C"];
const newItems = ["X", "A", "B", "C"];

// React without keys:
// new[0] (X) ↔ old[0] (A) → update A → X
// new[1] (A) ↔ old[1] (B) → update B → A
// new[2] (B) ↔ old[2] (C) → update C → B
// new[3] (C) ↔ no match → mount C
// Result: 3 updates, 1 mount

// With keys
{[
  { id: "X", text: "X" },  // new
  { id: "A", text: "A" },
  { id: "B", text: "B" },
  { id: "C", text: "C" }
].map(item => <Item key={item.id} text={item.text} />)}

// React with keys:
// X (new key) → mount
// A (key matches) → reuse fiber, no change
// B (key matches) → reuse fiber, no change
// C (key matches) → reuse fiber, no change
// Result: 1 mount, 0 updates (better!)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Naive tree diff complexity:**

```
Tree edit distance problem:
- Compare two trees node by node
- Find minimum edit operations (insert, delete, update, move)
- Optimal solution: O(n³) where n = nodes
- Levenshtein-like algorithm with dynamic programming
- 1000 nodes — 1 billion operations
- Browser blocks for seconds
```

**React O(n) algorithm:**

```typescript
function reconcile(currentFiber: Fiber | null, element: Element): Fiber {
  // Heuristic 1: Type comparison
  if (currentFiber && currentFiber.type === element.type) {
    // Same type — update
    return updateFiber(currentFiber, element.props);
  }

  // Different type — unmount + mount
  if (currentFiber) deleteFiber(currentFiber);
  return createFiber(element.type, element.props);
}

function reconcileChildren(parent: Fiber, oldChildren: Fiber[], newElements: Element[]) {
  // Heuristic 2: Keys
  if (hasKeys(newElements)) {
    return reconcileChildrenWithKeys(parent, oldChildren, newElements);
  }
  // No keys — index based
  return reconcileChildrenByIndex(parent, oldChildren, newElements);
}
```

**Index-based child diff (no keys):**

```typescript
function reconcileChildrenByIndex(parent, oldChildren, newElements) {
  const minLength = Math.min(oldChildren.length, newElements.length);

  for (let i = 0; i < minLength; i++) {
    // Recursively reconcile each pair
    reconcileFiber(oldChildren[i], newElements[i]);
  }

  // Extra new elements: mount
  for (let i = minLength; i < newElements.length; i++) {
    mountElement(parent, newElements[i]);
  }

  // Extra old children: unmount
  for (let i = minLength; i < oldChildren.length; i++) {
    deleteFiber(oldChildren[i]);
  }
}
// Complexity: O(n) where n = max(oldChildren.length, newElements.length)
```

**Keyed child diff:**

```typescript
function reconcileChildrenWithKeys(parent, oldChildren, newElements) {
  // Build map: key → old fiber
  const oldByKey = new Map();
  for (const old of oldChildren) {
    if (old.key !== null) oldByKey.set(old.key, old);
  }

  // Process new elements
  for (let i = 0; i < newElements.length; i++) {
    const elem = newElements[i];
    const matchedOld = oldByKey.get(elem.key);

    if (matchedOld) {
      // Reuse fiber, possibly move position
      moveOrUpdateFiber(matchedOld, elem, i);
      oldByKey.delete(elem.key);  // mark as used
    } else {
      // New element
      mountElement(parent, elem);
    }
  }

  // Remaining old fibers without match: unmount
  for (const remaining of oldByKey.values()) {
    deleteFiber(remaining);
  }
}
// Complexity: O(n) — Map operations O(1)
```

**Why O(n) heuristics work:**

Real UI patterns:
1. **Components rarely change type at same position** — `<Header>` stays `<Header>`, doesn't become `<Footer>`
2. **List items have natural identity** — DB rows, user list, etc. have IDs
3. **Animations stable** — same elements moving, not type-changing

When heuristics fail:
- Conditional rendering swapping component types: `cond ? <A /> : <B />` — full unmount/mount
- List without keys + reorder: bug-prone

**Reconciliation vs Diff:**

- **Diff**: comparing two data structures
- **Reconciliation**: React's diff + adjustment of Fiber tree

**Multi-level reconciliation:**

Reconciliation is recursive (per-fiber):
1. Compare current fiber with new element
2. If same type, render component (get new children)
3. Reconcile children (recursive call)
4. If different type, unmount + mount, then reconcile new tree from scratch

```typescript
function reconcileFiber(currentFiber, element) {
  if (currentFiber.type !== element.type) {
    // Unmount current, mount new
    unmount(currentFiber);
    return mount(element);
  }

  // Same type — update props
  currentFiber.pendingProps = element.props;

  // Render component
  if (isComponent(currentFiber)) {
    const newChildren = currentFiber.type(element.props);
    reconcileChildren(currentFiber, currentFiber.children, newChildren);
  }

  return currentFiber;
}
```

**Complexity comparison:**

| Approach | Complexity | Real-world |
|----------|-----------|-------------|
| Optimal tree edit distance | O(n³) | Impractical |
| React heuristics (no keys) | O(n) | Suitable, suboptimal reorder |
| React heuristics (with keys) | O(n) | Optimal moves |

**Limitations:**

- Wrapping component changes: `<div><Comp /></div>` → `<section><Comp /></section>` — Comp unmounted (parent type changed). Optimization: keep `<Comp>` if structurally similar — React doesn't.
- Non-keyed reorder: O(n) but suboptimal moves.

</details>

### Edge Cases

- **Same component, different key**: Same type, different key → unmount + mount (key change forces remount, even with same type).
- **Different component, same key**: Type wins over key. Unmount + mount.
- **Null/undefined children**: Skipped in reconciliation, no fiber created.

### Follow-up savollar

- "React diff algorithm name?" — No formal name, "React reconciliation". Based on heuristics, not classical algorithm.
- "Vue diff algorithm farqli?" — Vue 3 uses similar heuristics (PatchFlag) + compile-time hints (Block tree). More aggressive optimization possible due to template static analysis.
- "Solid no diff algorithm — better?" — Solid compiles JSX to DOM mutations directly (no virtual diff). Faster microbenchmarks. But React's diff enables features like Suspense, transitions.

</details>

---

### 15. Element type comparison — same vs different [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Reconciler position'da eski va yangi element'ni `===` bilan **type** field'ini solishtiradi. **Same type** (e.g., both `'div'` or both `Button` function) → existing fiber reuse, props update. **Different type** → unmount old subtree (cleanup, refs detach, effects), mount new subtree (fresh state, new fibers, mount effects). Type comparison element-level — komponent ichidagi children'lar ham reconciliation davom etadi.

### To'liq tushuntirish

**Type comparison qoidalari:**

```typescript
function reconcileFiber(current: Fiber, newElement: Element) {
  // Compare types
  if (current.type === newElement.type && current.key === newElement.key) {
    // SAME — update path
    return updateFiber(current, newElement.props);
  }

  // DIFFERENT — replace
  unmountFiber(current);
  return createFiber(newElement);
}
```

**Element type:**
- `'div'`, `'span'`, etc. — string (HostComponent type)
- `Button`, `App` — function reference (FunctionComponent type)
- `class Foo` — class reference (ClassComponent type)
- `Symbol(react.fragment)` — Fragment
- Special: `Symbol(react.memo)`, `Symbol(react.forward_ref)` (R19'da ref oddiy prop, forwardRef deprecated), etc.

### Kod misoli

```tsx
// SAME type — update
function App({ count }: { count: number }) {
  return <div>{count}</div>;
}

// Render 1: count=0
// JSX: <div>0</div>

// Render 2: count=1
// JSX: <div>1</div>

// Reconciliation:
// old.type = 'div', new.type = 'div' — same
// → reuse div fiber, update text child
// DOM: <div>0</div> → <div>1</div> (text updated)
// Effect: input focus preserved, animations smooth
```

```tsx
// DIFFERENT type — replace
function App({ isLoading }: { isLoading: boolean }) {
  return isLoading ? <Spinner /> : <Content />;
}

// isLoading: true
// Reconciliation:
// new.type = Spinner

// isLoading: false (re-render)
// new.type = Content
// old.type = Spinner

// types: Spinner !== Content → DIFFERENT
// → unmount Spinner (cleanup, refs detach)
// → mount Content (fresh)
// Effect: Spinner state lost, Content state fresh
```

```tsx
// Same component, different position — different fibers
function App({ swap }: { swap: boolean }) {
  return swap ? (
    <>
      <Counter id="A" />
      <Counter id="B" />
    </>
  ) : (
    <>
      <Counter id="B" />
      <Counter id="A" />
    </>
  );
}

// Without keys — position-based:
// Both render: 2 Counter fibers at positions 0, 1

// Render 1 (swap=false):
// position[0] = Counter (id="B")
// position[1] = Counter (id="A")

// Render 2 (swap=true):
// position[0] = Counter (id="A")  ← same fiber as before (Counter)
// position[1] = Counter (id="B")  ← same fiber

// Type same (both Counter), so update path
// id prop changes → component re-renders
// State (count) attached to position, NOT to id!

// Counter at position 0:
//   was id="B" with count=5
//   now id="A" — but state.count = 5 stays
// BUG: count=5 follows position, not id
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Type identity rules:**

```typescript
// Strict reference equality
oldType === newType

// HostComponent: string equality
'div' === 'div'  // true
'div' === 'span' // false

// FunctionComponent: reference equality
const Comp1 = () => <p />;
const Comp2 = () => <p />;
Comp1 === Comp1  // true
Comp1 === Comp2  // false (different references)

// ClassComponent: reference equality
class A extends Component { render() { return null; } }
A === A  // true
```

**Pitfall: Inline component definition:**

```tsx
function Parent() {
  // ❌ Inline component — every render new function reference
  function InlineChild() { return <p />; }

  return <InlineChild />;
}

// Render 1: InlineChild = function@0x1234
// Render 2: InlineChild = function@0x5678 (new reference)

// Reconciler: old.type !== new.type → UNMOUNT + MOUNT
// All InlineChild's state lost on every render
// Severe performance penalty
```

**Fix:**

```tsx
// ✅ Hoist component out
function InlineChild() { return <p />; }

function Parent() {
  return <InlineChild />;
}

// Render 1, 2, ...: InlineChild same reference
// Reconciler: same type → update only
```

**Same type, different props — update flow:**

```typescript
// old fiber: { type: Button, props: { label: "Save" } }
// new element: { type: Button, props: { label: "Submit" } }

// Reconciler:
// type match → reuse fiber
// props differ → schedule re-render
// Re-render Button():
//   Button("Submit") returns <button>Submit</button>
//   Reconcile children (recursive)
//   button child: text "Save" → "Submit" (Update flag)
```

**Type change cascade:**

```tsx
function App({ mode }: { mode: 'A' | 'B' }) {
  return (
    <div>
      {mode === 'A' ? (
        <Section>
          <DataView />
        </Section>
      ) : (
        <main>
          <DataView />
        </main>
      )}
    </div>
  );
}

// Mode A → B
// Reconcile:
// div same type → update path
// Children: Section vs main (different)
// → unmount Section subtree (DataView inside, all state lost)
// → mount main subtree (new DataView, fresh state)

// Even though DataView component "appears" in both,
// it's mounted/unmounted because its parent changed type
```

**Memo + type stability:**

```tsx
const MemoizedExpensive = memo(ExpensiveComponent);

function App() {
  return <MemoizedExpensive data={data} />;
}

// MemoizedExpensive — stable reference (memo wraps once)
// Type same across renders
// Memoization works
```

**Lazy components — type changes after load:**

```tsx
const LazyAbout = lazy(() => import("./About"));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <LazyAbout />
    </Suspense>
  );
}

// Initial render:
// LazyAbout — lazy wrapper (LazyComponent fiber, tag 16)
// Module not loaded → throw Promise → Suspense fallback

// After module loads:
// LazyAbout's internal `_payload` updated with About component
// Suspense re-renders
// LazyComponent fiber persists, but renders About internally
```

**`React.memo` type identity:**

```tsx
// memo creates wrapper component (different reference)
const A = memo(Button);
const B = memo(Button);
A === B  // false (different memo wrappers)

// Stable instance:
const StableMemoButton = memo(Button);  // hoist, single instance
function App() {
  return <StableMemoButton />;  // same A every render
}
```

**HostComponent string interning:**

```typescript
// 'div' === 'div' always true (string interning)
// React doesn't worry about HostComponent type identity
```

**Custom equality fails:**

```tsx
// ❌ Cannot make fundamentally different components "same"
const ComponentA = () => <p />;
const ComponentB = () => <p />;
// React: ComponentA !== ComponentB → unmount + mount

// React respects strict reference equality, no custom override
```

**Practical: Don't change parent type for conditional rendering:**

```tsx
// ❌ Type change cascade — children remount
function Page({ loading }: { loading: boolean }) {
  return loading ? (
    <div className="loading-page">
      <Article />
    </div>
  ) : (
    <main className="loaded-page">
      <Article />
    </main>
  );
}
// Article remounts every loading state change

// ✅ Stable parent type
function Page({ loading }: { loading: boolean }) {
  return (
    <div className={loading ? "loading-page" : "loaded-page"}>
      <Article />
    </div>
  );
}
// Article preserves
```

</details>

### Edge Cases

- **`null` element**: Skipped in reconciliation. Old fiber at same position — unmount.
- **Conditional rendering switching null ↔ element**: Mount/unmount cycle each toggle.
- **Component returning Fragment**: Fragment fiber type stable — but Fragment children reconcile recursively.

### Follow-up savollar

- "ESLint rule against inline components?" — `react/no-unstable-nested-components` — flags inline component definitions inside parent.
- "Higher-order components type stability?" — HOC creates wrapper component reference. If created inside render — unstable. Hoist outside render scope.
- "TypeScript type changes — runtime'da affect-mi?" — TypeScript erased at compile time. Runtime: function/class reference. TS type unrelated.

</details>

---

### 16. Bailout algorithm — 4 ta sabab [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Bailout** — Reconciler subtree'ni qayta render qilmaydi (skip optimization). 4 ta asosiy sabab: (1) **Element identity** (`oldProps === newProps` reference equality) — same JSX reference, (2) **`React.memo` shallow check** — props same, (3) **`useMemo`/`useCallback`** — stable reference for child memo bypass, (4) **State equality** — `setState` bilan same value (`Object.is` bailout, no re-render).

### To'liq tushuntirish

**Bailout sabab'lari:**

| # | Sabab | Mexanizm | Misol |
|---|-------|----------|-------|
| 1 | Element identity | `oldProps === newProps` (reference equality) | Children prop forwarded |
| 2 | `React.memo` shallow | Shallow props equality | `memo(Component)` |
| 3 | `useMemo`/`useCallback` deps | Deps unchanged | `useMemo(() => x, [a])` |
| 4 | State equality | `Object.is(prevState, newState)` | `setX(sameValue)` |

### Kod misoli

```tsx
// 1. Element identity bailout
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Inc: {count}</button>
      <ExpensiveChild />  {/* JSX created in render */}
    </div>
  );
}

// Each render: <ExpensiveChild /> — new element object
// → new element !== old element
// → ExpensiveChild re-rendered (no bailout)

// ✅ Children prop forwarding — element identity stable
function App() {
  return (
    <Parent>
      <ExpensiveChild />  {/* Created in App, passed to Parent */}
    </Parent>
  );
}

function Parent({ children }: { children: React.ReactNode }) {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Inc: {count}</button>
      {children}  {/* same element reference across Parent re-renders */}
    </div>
  );
}
// children — same element from App perspective
// Parent state change → Parent re-renders, but children element same → bailout for ExpensiveChild
```

```tsx
// 2. React.memo bailout
const ExpensiveChild = memo(function ExpensiveChild({ data }: { data: Data }) {
  // expensive render
  return <div>{data.value}</div>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const data = { value: "constant" };  // new object every render
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveChild data={data} />  {/* new data reference */}
    </>
  );
}
// Parent re-renders: data is new object → memo shallow check fails → re-render

// ✅ Stable data reference
function Parent() {
  const [count, setCount] = useState(0);
  const data = useMemo(() => ({ value: "constant" }), []);  // memoized
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveChild data={data} />  {/* stable reference */}
    </>
  );
}
// data stable → memo bailout
```

```tsx
// 3. useMemo bailout
function Parent({ data }: { data: Data[] }) {
  const expensiveValue = useMemo(() => {
    return data.reduce((sum, d) => sum + d.value, 0);
  }, [data]);

  return <div>{expensiveValue}</div>;
}
// data unchanged → useMemo returns cached value
// data changed → recompute
```

```tsx
// 4. State equality bailout
function Counter() {
  const [count, setCount] = useState(0);
  console.log("render");

  return (
    <button onClick={() => setCount(0)}>  {/* same value as current */}
      {count}
    </button>
  );
}
// Click: setCount(0)
// Reconciler: Object.is(0, 0) = true → bailout, no re-render
// "render" not logged
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Bailout 1: Element identity (children prop):**

```typescript
function reconcileSingleElement(currentFiber, newElement) {
  if (Object.is(currentFiber.element, newElement)) {
    // Same element reference → no need to re-render
    return bailoutOnAlreadyFinishedWork(currentFiber);
  }
  // Otherwise reconcile
}
```

`children` prop pattern enables this:

```tsx
// ❌ Without children pattern — element re-created
function Parent() {
  const [_, setX] = useState(0);
  return <Child>{<Expensive />}</Child>;  // <Expensive /> new every render
}

// ✅ Children pattern — element stable
function App() {
  return <Parent><Expensive /></Parent>;  // Expensive element created in App
}
function Parent({ children }) {
  const [_, setX] = useState(0);
  return <div>{children}</div>;  // children — same reference from App
}
```

**Bailout 2: React.memo:**

```typescript
function updateMemoComponent(current, workInProgress) {
  const prevProps = current.memoizedProps;
  const nextProps = workInProgress.pendingProps;
  const compare = workInProgress.type.compare ?? shallowEqual;

  if (compare(prevProps, nextProps)) {
    // Bailout
    return bailoutOnAlreadyFinishedWork(current, workInProgress);
  }

  // Re-render
  // ...
}

function shallowEqual(a, b) {
  if (Object.is(a, b)) return true;
  if (typeof a !== "object" || typeof b !== "object") return false;
  if (a === null || b === null) return false;

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;

  for (const key of keysA) {
    if (!Object.is(a[key], b[key])) return false;
  }
  return true;
}
```

**Bailout 3: useMemo:**

```typescript
function updateMemo<T>(create: () => T, deps: any[]): T {
  const hook = updateHook();
  const prevState = hook.memoizedState;

  if (prevState !== null) {
    const [prevValue, prevDeps] = prevState;
    if (areHookInputsEqual(deps, prevDeps)) {
      return prevValue;  // bailout — return cached value
    }
  }

  const newValue = create();
  hook.memoizedState = [newValue, deps];
  return newValue;
}
```

`useCallback`:

```typescript
function updateCallback(callback: any, deps: any[]) {
  return updateMemo(() => callback, deps);
}
// useCallback(fn, deps) ≡ useMemo(() => fn, deps)
```

**Bailout 4: State equality:**

```typescript
function dispatchSetState(fiber, queue, action) {
  const update = { action, next: null };
  enqueueUpdate(queue, update);

  // Optimization: same value → bailout immediately
  if (typeof action !== "function") {
    if (Object.is(action, fiber.memoizedState)) {
      // Same value, no work needed
      // Don't even schedule re-render
      return;
    }
  }

  scheduleUpdateOnFiber(fiber);
}
```

But this optimization only works for direct value (not functional updates). For `setX(prev => x)`, React doesn't know if result equals prev until it runs.

**During render bailout (subtree):**

```typescript
function bailoutOnAlreadyFinishedWork(current, workInProgress) {
  // Skip rendering this fiber
  // BUT: check if children have pending work
  if (current.childLanes & renderLanes) {
    // Some descendants need re-render
    cloneChildFibers(workInProgress);
    return workInProgress.child;  // continue rendering children
  }

  // No descendants need re-render either
  return null;  // skip entire subtree
}
```

Bailout doesn't always skip entire subtree — descendants might have pending work (state change in deep child).

**`childLanes` propagation:**

When state changes in a fiber, React marks `lanes` on that fiber. Then walks up `return` parents, OR'ing `childLanes`:

```typescript
function markUpdateLaneFromFiberToRoot(fiber, lane) {
  fiber.lanes |= lane;
  let parent = fiber.return;
  while (parent !== null) {
    parent.childLanes |= lane;
    parent = parent.return;
  }
}
```

This way Reconciler knows: "subtree X has pending work in lane L".

**Combined bailout — practical example:**

```tsx
const ExpensiveChild = memo(function ExpensiveChild({ data, onClick }) {
  return <div onClick={onClick}>{data.value}</div>;
});

function Parent({ data }) {
  const [count, setCount] = useState(0);

  // Stable callback
  const onClick = useCallback(() => console.log("clicked"), []);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveChild data={data} onClick={onClick} />
    </>
  );
}

// State change (count) → Parent re-renders
// data: same reference (passed from above) → shallow check OK
// onClick: useCallback stable → shallow check OK
// memo bailout → ExpensiveChild not re-rendered
```

**Bailout limits:**

```tsx
// Even with memo + useCallback, things break:
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveChild
        data={{ value: count }}  // ← inline object (new every render)
      />
    </>
  );
}
// memo shallow check fails (different data reference)
// Re-renders despite memo
```

**React Compiler eliminates many bailouts:**

```tsx
// Pre-compiler — manual memoization
const data = useMemo(() => ({ value: count }), [count]);
const onClick = useCallback(() => action(), []);

// Compiler-era — automatic
const data = { value: count };  // compiler memoizes this
const onClick = () => action();  // compiler memoizes this
```

</details>

### Edge Cases

- **Memo with custom areEqual**: Override shallow check. `memo(Component, (prev, next) => prev.id === next.id)`. Be careful — incomplete check causes stale UI.
- **State bailout — `Object.is(NaN, NaN)`**: Returns true (Object.is treats NaN equal to NaN). So `setState(NaN)` from NaN → bailout.
- **State bailout — `setState({...})`**: Object literal — new reference, no bailout (even if values same). Use `useState` with comparison or normalize.

### Follow-up savollar

- "Bailout DevTools'da ko'rinadi-mi?" — Profiler shows fibers with "Did not render" status. Highlight Updates option.
- "Bailout state queue'ga ta'sir qiladi-mi?" — Bailout 4 prevents state queue update. Bailouts 1-3 prevent render but state queue intact.
- "useState'da prevState === newState bo'lsa effects ham skip-mi?" — Yes, no re-render → no new effects.

</details>

---

### 17. Sibling matching — keyless vs keyed [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Reconciler children'ni sibling-by-sibling traverse qiladi (linked list). **Keyless** — index-based: `old[0] ↔ new[0]`, `old[1] ↔ new[1]`. **Keyed** — key-based: `Map<key, fiber>` lookup. Keyed mode insertion/reordering'da efficient (O(n)) va correct (state preserved). Keyless reorder buggy state pollution.

### To'liq tushuntirish

**Keyless (index-based) — sodda algoritm:**

```typescript
function reconcileChildrenByIndex(parent: Fiber, oldChildren: Fiber, newChildren: Element[]) {
  let oldFiber = oldChildren;
  let newIdx = 0;

  while (oldFiber !== null && newIdx < newChildren.length) {
    const newChild = newChildren[newIdx];

    if (oldFiber.type === newChild.type && oldFiber.key === newChild.key) {
      // Same type → reuse
      updateFiber(oldFiber, newChild);
    } else {
      // Different type → break, switch to keyed
      break;
    }

    oldFiber = oldFiber.sibling;
    newIdx++;
  }

  // Remaining old fibers — delete
  while (oldFiber !== null) {
    deleteFiber(oldFiber);
    oldFiber = oldFiber.sibling;
  }

  // Remaining new elements — mount
  for (; newIdx < newChildren.length; newIdx++) {
    mountElement(parent, newChildren[newIdx]);
  }
}
```

**Keyed (Map-based) — handle reorders:**

```typescript
function reconcileChildrenWithKeys(parent: Fiber, oldChildren: Fiber, newChildren: Element[]) {
  // Build Map of remaining old fibers by key
  const oldByKey = new Map<string | number, Fiber>();
  let oldFiber = oldChildren;
  while (oldFiber !== null) {
    const key = oldFiber.key ?? oldFiber.index;
    oldByKey.set(key, oldFiber);
    oldFiber = oldFiber.sibling;
  }

  // Process new elements
  for (let newIdx = 0; newIdx < newChildren.length; newIdx++) {
    const newChild = newChildren[newIdx];
    const key = newChild.key ?? newIdx;
    const matchedOld = oldByKey.get(key);

    if (matchedOld && matchedOld.type === newChild.type) {
      // Reuse + position update
      reuseFiber(matchedOld, newChild, newIdx);
      oldByKey.delete(key);
    } else {
      // No match → mount new
      mountElement(parent, newChild);
    }
  }

  // Remaining old → delete
  for (const old of oldByKey.values()) {
    deleteFiber(old);
  }
}
```

### Kod misoli

```tsx
// SCENARIO: Insert at front
// Old: [A, B, C]
// New: [X, A, B, C]

// Keyless (index):
function ListBad({ items }: { items: string[] }) {
  return items.map((item, i) => <Item key={i} text={item} />);
}

// Reconciler trace:
// new[0] = X, key=0 ↔ old[0] = A, key=0 → SAME KEY → reuse fiber, props X (was A)
// new[1] = A, key=1 ↔ old[1] = B, key=1 → SAME KEY → reuse fiber, props A (was B)
// new[2] = B, key=2 ↔ old[2] = C, key=2 → SAME KEY → reuse fiber, props B (was C)
// new[3] = C, key=3 ↔ no old → MOUNT
// Result: 3 props update, 1 mount
// Item state attached to position — wrong if Item has state

// Keyed (stable ID):
function ListGood({ items }: { items: { id: string; text: string }[] }) {
  return items.map((item) => <Item key={item.id} text={item.text} />);
}

// Reconciler trace:
// Old by key: {A, B, C}
// new[0] = X (new key) → no match → MOUNT
// new[1] = A → match old A → reuse (move position 0 → 1)
// new[2] = B → match old B → reuse (move position 1 → 2)
// new[3] = C → match old C → reuse (move position 2 → 3)
// Result: 1 mount, 3 moves
// Item state preserved (attached to identity, not position)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Hybrid algorithm (React internals):**

React doesn't strictly choose keyed vs keyless — it tries iterative keyless first, switches to keyed when mismatch:

```typescript
function reconcileChildrenArray(returnFiber, currentFirstChild, newChildren) {
  let resultingFirstChild = null;
  let previousNewFiber = null;
  let oldFiber = currentFirstChild;
  let lastPlacedIndex = 0;
  let newIdx = 0;
  let nextOldFiber = null;

  // Phase 1: Iterate as long as types match (fast path)
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    if (oldFiber.index > newIdx) {
      nextOldFiber = oldFiber;
      oldFiber = null;
    } else {
      nextOldFiber = oldFiber.sibling;
    }

    const newFiber = updateSlot(returnFiber, oldFiber, newChildren[newIdx]);
    if (newFiber === null) {
      // Type mismatch — break, switch to keyed mode
      if (oldFiber === null) oldFiber = nextOldFiber;
      break;
    }

    // Update fiber
    if (shouldTrackSideEffects && oldFiber && newFiber.alternate === null) {
      deleteChild(returnFiber, oldFiber);
    }

    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);

    if (previousNewFiber === null) {
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
    oldFiber = nextOldFiber;
  }

  // Phase 2: All new exhausted — delete remaining old
  if (newIdx === newChildren.length) {
    deleteRemainingChildren(returnFiber, oldFiber);
    return resultingFirstChild;
  }

  // Phase 3: All old exhausted — mount remaining new
  if (oldFiber === null) {
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = createChild(returnFiber, newChildren[newIdx]);
      // ... append to chain
    }
    return resultingFirstChild;
  }

  // Phase 4: Both still have items — switch to keyed (Map-based)
  const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

  for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = updateFromMap(existingChildren, returnFiber, newIdx, newChildren[newIdx]);
    if (newFiber !== null) {
      // ... reuse from map, position updated
      existingChildren.delete(newFiber.key === null ? newIdx : newFiber.key);
    }
    // ... append to chain
  }

  // Phase 5: Delete remaining map entries (no match)
  if (shouldTrackSideEffects) {
    existingChildren.forEach(child => deleteChild(returnFiber, child));
  }

  return resultingFirstChild;
}
```

**Phase summary:**

| Phase | Description | Complexity |
|-------|-------------|------------|
| 1 | Fast iteration while types match | O(n) |
| 2 | Delete remaining old | O(remaining) |
| 3 | Mount remaining new | O(remaining) |
| 4 | Map-based for keyed reorders | O(n) |
| 5 | Delete unmatched | O(unmatched) |

**`placeChild` — track moves:**

```typescript
function placeChild(newFiber, lastPlacedIndex, newIndex) {
  newFiber.index = newIndex;
  if (!shouldTrackSideEffects) return lastPlacedIndex;

  const current = newFiber.alternate;
  if (current !== null) {
    const oldIndex = current.index;
    if (oldIndex < lastPlacedIndex) {
      // This is a move
      newFiber.flags |= Placement;
      return lastPlacedIndex;
    }
    return oldIndex;
  }

  // New mount
  newFiber.flags |= Placement;
  return lastPlacedIndex;
}
```

`Placement` flag set on moved or new fibers.

**Reorder example trace:**

```
Old: [A(idx=0), B(idx=1), C(idx=2), D(idx=3)]
New: [B, D, A, C]

Phase 1 (fast path):
  newIdx=0, newChild=B, oldFiber=A
  type same? key match B vs A? key mismatch
  → break (switch to keyed)

Phase 4 (keyed):
  existingChildren = {A, B, C, D}
  newIdx=0, newChild=B → match B → place at index 0 (was 1, now 0 — move)
    lastPlacedIndex = 1 (B's old index)
  newIdx=1, newChild=D → match D → place at index 1 (was 3 — move forward, no flag)
    lastPlacedIndex = 3 (D's old index)
  newIdx=2, newChild=A → match A → place at index 2 (was 0 — move backward → Placement flag)
    lastPlacedIndex stays 3
  newIdx=3, newChild=C → match C → place at index 3 (was 2 — move backward → Placement flag)
    lastPlacedIndex stays 3

Phase 5: existingChildren empty, no deletes

Result:
  A: Placement flag (moved from 0 to 2)
  C: Placement flag (moved from 2 to 3)
  B, D: in place (forward moves don't need flag)
```

`Placement` flag — DOM `insertBefore` will be called.

**Performance characteristics:**

```typescript
// Best case: append-only (last item added)
// Old: [A, B, C], New: [A, B, C, D]
// Phase 1 (fast): A==A, B==B, C==C — all match
// Phase 3: mount D
// Total: O(n) — no Map needed

// Worst case: reverse order
// Old: [A, B, C], New: [C, B, A]
// Phase 1: first child mismatch (C vs A keys)
// Phase 4: Map-based, all moves
// Total: O(n) but with Map overhead
```

**Map allocation pattern:**

Phase 4 only triggered when needed. For pure append/prepend, Phase 1 + Phase 3 sufficient. Map allocation avoided.

</details>

### Edge Cases

- **All keys removed**: Delete all children, mount new.
- **Mixed keyed/unkeyed**: Behavior tricky. Position-based fallback for unkeyed.
- **Duplicate keys**: Reconciler picks first match (order not guaranteed). Warning emitted.

### Follow-up savollar

- "Phase 1 vs Phase 4 boundary?" — First mismatch at any position. Append-only never reaches Phase 4.
- "Reorder with deletion?" — Phase 4 handles. Deleted items remain in `existingChildren` map, deleted in Phase 5.
- "Why not always Phase 4?" — Phase 1 fast path avoids Map allocation for common case (no reorder).

</details>

---

### 18. Update propagation — qaysi node'lar render qilinadi [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

State update bo'lganda, React **shu Fiber'dan boshlab pastga** (descendants) render qiladi. Default: parent re-renders → barcha children'ni re-render qiladi (cascade). Lekin **bailout mexanizmlari** (memo, useMemo, element identity) subtree'ni skip qila oladi. Sibling'lar va ancestor'lar **avtomatik re-render qilinmaydi** (faqat ularning state o'zgarsa).

### To'liq tushuntirish

**Update propagation algorithm:**

1. State change happens in fiber X (via `setState`)
2. Mark `X.lanes` with the priority lane
3. Walk up `return` parents, OR `childLanes` on each ancestor
4. Reconciler starts from root, traverses tree
5. At each fiber, check `lanes & renderLanes` — if pending work, render
6. Else if `childLanes & renderLanes` — bailout self, recurse children
7. Else (`(lanes & renderLanes) === 0 && (childLanes & renderLanes) === 0`) — skip entire subtree

### Kod misoli

```tsx
function App() {
  console.log("App render");
  return (
    <div>
      <Header />
      <Main />
    </div>
  );
}

function Header() {
  console.log("Header render");
  return <header>Header</header>;
}

function Main() {
  const [count, setCount] = useState(0);
  console.log("Main render");
  return (
    <main>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Sidebar />
    </main>
  );
}

function Sidebar() {
  console.log("Sidebar render");
  return <aside>Sidebar</aside>;
}

// Initial render output:
// App render
// Header render
// Main render
// Sidebar render

// Click button (setCount in Main):
// Main render
// Sidebar render
// (App, Header NOT re-rendered — siblings/ancestors)
```

```tsx
// With memo bailout
const MemoSidebar = memo(Sidebar);

function Main() {
  const [count, setCount] = useState(0);
  return (
    <main>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <MemoSidebar />  {/* No props change — memo bailout */}
    </main>
  );
}

// Click button:
// Main render (state changed)
// (MemoSidebar BAILOUT — no Sidebar log)
```

```tsx
// Cascade — parent state change re-renders all
function App() {
  const [theme, setTheme] = useState("light");
  return (
    <div>
      <Header />        {/* Re-renders — parent state changed */}
      <Main />          {/* Re-renders */}
      <Footer />        {/* Re-renders */}
    </div>
  );
}

// Click theme toggle (setTheme in App):
// App re-renders
// Header re-renders
// Main re-renders (and its children Sidebar etc.)
// Footer re-renders
// Cascade through all descendants
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Lanes mark up tree:**

```typescript
function markUpdateLaneFromFiberToRoot(sourceFiber, lane) {
  // Mark source fiber
  sourceFiber.lanes |= lane;

  // Walk up parents, mark childLanes
  let parent = sourceFiber.return;
  while (parent !== null) {
    parent.childLanes |= lane;
    if (parent.alternate !== null) {
      parent.alternate.childLanes |= lane;
    }
    parent = parent.return;
  }
}
```

After this, traversal can quickly determine which subtrees have pending work.

**Render time:**

```typescript
function beginWork(current: Fiber | null, workInProgress: Fiber, renderLanes: Lanes) {
  // Has pending work?
  const lanes = workInProgress.lanes;
  if ((lanes & renderLanes) !== NoLanes) {
    // This fiber needs work
    didReceiveUpdate = true;
  } else if (current !== null) {
    // No pending work for this fiber, but children might have
    if ((workInProgress.childLanes & renderLanes) === NoLanes) {
      // No work in subtree — skip everything
      return null;
    }
    // Children have work — render them, but bailout this fiber
    didReceiveUpdate = false;
  }

  switch (workInProgress.tag) {
    case FunctionComponent:
      return updateFunctionComponent(current, workInProgress, ...);
    // ...
  }
}
```

**Bailout vs work:**

| State | Action |
|-------|--------|
| `lanes & renderLanes` | Render this fiber |
| `childLanes & renderLanes` (no own lanes) | Bailout self, recurse children |
| Neither | Skip subtree |

**Practical impact:**

```tsx
// Tree:
//   App
//   ├── Header
//   ├── Main (state lives here)
//   │   ├── Article
//   │   └── Sidebar
//   └── Footer

// State update in Main:
// 1. Main.lanes |= updateLane
// 2. Walk up: App.childLanes |= updateLane, root.childLanes |= updateLane

// Render from root:
// root: childLanes set → recurse
// App: own lanes 0, childLanes set → bailout self, recurse children
//   Header: own lanes 0, childLanes 0 → skip subtree (no log)
//   Main: own lanes set → render (renders Article + Sidebar by default)
//   Footer: own lanes 0, childLanes 0 → skip subtree
```

So Header and Footer are skipped — bailout works at parent level (App).

**Why parent re-render → children re-render (default):**

```typescript
// Function component re-render
function App() {
  // Function called, returns new element tree
  return (
    <div>
      <Header />   {/* New <Header /> element every call */}
      <Main />
    </div>
  );
}

// New element references (Object.is === false) → element identity bailout fails
// Children re-render unless wrapped in memo
```

**`React.memo` bailout:**

```tsx
const MemoHeader = memo(Header);

function App() {
  const [_, setX] = useState(0);
  return (
    <div>
      <MemoHeader />  {/* No props — memo always bails out (props always equal) */}
      <Main />
    </div>
  );
}

// State change in App:
// MemoHeader: shallow check (no props) → equal → bailout
// Main: not memoized → re-render
```

**`useContext` propagation:**

```typescript
// When Provider value changes, propagate to ALL Consumers in subtree
function propagateContextChange(workInProgress, contextType, renderLanes) {
  let fiber = workInProgress.child;
  while (fiber !== null) {
    if (fiberSubscribesToContext(fiber, contextType)) {
      // Mark this fiber and parents
      markUpdateLaneFromFiberToRoot(fiber, renderLanes);
    }

    // Recurse children
    if (fiber.child) fiber = fiber.child;
    else if (fiber.sibling) fiber = fiber.sibling;
    else { /* up */ }
  }
}
```

Context propagation — special: Provider value change marks ALL consumers, not just immediate children.

**Sibling propagation — none:**

State change in fiber A doesn't directly mark sibling B. Only parent and self.

**State management library bypass:**

```tsx
// Redux: connect()/useSelector — selective subscription
const Component = connect(state => ({ count: state.count }))(Counter);

// Only re-renders when state.count changes
// Despite being deep in tree, doesn't depend on parent re-render
```

External stores via `useSyncExternalStore` — bypass React's tree-based propagation, store-based subscriptions.

**Update batching impact:**

Multiple `setState` in same batch → all marks, single render pass. Render visits all marked fibers in tree order.

</details>

### Edge Cases

- **`setState` in unmounting component**: Warning, no render (component going away).
- **`setState` in render body**: Loop! React detects and breaks (max iterations, throws).
- **`setState` from event during render**: Queued for next render (concurrent rendering).

### Follow-up savollar

- "Sibling re-render needed — solution?" — Lift state to common ancestor, or use Context, or external store.
- "DevTools shows skipped fibers?" — Profiler "Render commit" tab shows which components rendered. Highlight Updates option.
- "Force entire tree re-render?" — Top-most state change, no memoization. Or `key` change on root.

</details>

---

### 19. `React.memo` bailout — shallow comparison va custom equality [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`React.memo`** — komponentni props o'zgarmasdan re-render qilmaslik uchun wrapper. Default — **shallow comparison** (har props key uchun `Object.is`). Custom **`areEqual`** function bilan to'liq nazorat. Bailout ishlamaydi: (1) parent re-render bo'lsa va prop reference yangilangan bo'lsa (inline object/array), (2) children prop bilan (har gal yangi element).

### To'liq tushuntirish

**`memo` API:**

```tsx
import { memo } from "react";

// Default — shallow equality
const MemoButton = memo(Button);

// Custom areEqual
const MemoCard = memo(Card, (prevProps, nextProps) => {
  return prevProps.id === nextProps.id;
});
```

**Shallow equality:**

```typescript
function shallowEqual(a, b) {
  if (Object.is(a, b)) return true;
  if (typeof a !== "object" || typeof b !== "object") return false;
  if (a === null || b === null) return false;

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;

  for (const key of keysA) {
    if (!Object.is(a[key], b[key])) return false;
  }
  return true;
}
```

### Kod misoli

```tsx
// 1. Basic memo
const Card = memo(function Card({ name, age }: { name: string; age: number }) {
  console.log("Card render:", name);
  return <div>{name}, {age}</div>;
});

function App() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Card name="Ali" age={25} />  {/* Primitive props — bailout works */}
    </>
  );
}
// Click button: only App renders, Card bails out
```

```tsx
// 2. Object prop — bailout fails
const Card = memo(function Card({ user }: { user: { name: string; age: number } }) {
  console.log("Card render");
  return <div>{user.name}, {user.age}</div>;
});

function App() {
  const [count, setCount] = useState(0);
  const user = { name: "Ali", age: 25 };  // ❌ new object every render
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Card user={user} />
    </>
  );
}
// Click: App renders, Card ALSO renders (user is new reference)

// ✅ Fix: useMemo
function App() {
  const [count, setCount] = useState(0);
  const user = useMemo(() => ({ name: "Ali", age: 25 }), []);  // stable
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Card user={user} />
    </>
  );
}
```

```tsx
// 3. Function prop — bailout fails
const Button = memo(function Button({ onClick, label }: { onClick: () => void; label: string }) {
  console.log("Button render:", label);
  return <button onClick={onClick}>{label}</button>;
});

function App() {
  const [count, setCount] = useState(0);
  return (
    <>
      <p>{count}</p>
      <Button label="Inc" onClick={() => setCount(c => c + 1)} />  {/* ❌ new fn every render */}
    </>
  );
}
// Click: Button re-renders despite memo

// ✅ Fix: useCallback
function App() {
  const [count, setCount] = useState(0);
  const handleClick = useCallback(() => setCount(c => c + 1), []);
  return (
    <>
      <p>{count}</p>
      <Button label="Inc" onClick={handleClick} />
    </>
  );
}
```

```tsx
// 4. Custom areEqual
interface UserCardProps {
  user: { id: string; name: string; lastSeen: Date };
  onSelect: (id: string) => void;
}

const UserCard = memo(
  function UserCard({ user, onSelect }: UserCardProps) {
    return (
      <div onClick={() => onSelect(user.id)}>
        {user.name}
      </div>
    );
  },
  (prev, next) => {
    // Custom: ignore lastSeen, compare only id and name
    return (
      prev.user.id === next.user.id &&
      prev.user.name === next.user.name &&
      prev.onSelect === next.onSelect
    );
  }
);

// lastSeen changes don't trigger re-render — desirable for some UIs
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`memo` internals:**

```typescript
function memo<P>(Component: ComponentType<P>, areEqual?: (prev: P, next: P) => boolean): ComponentType<P> {
  return {
    $$typeof: REACT_MEMO_TYPE,
    type: Component,
    compare: areEqual ?? null,
  };
}

// Reconciler check:
function updateMemoComponent(current, workInProgress) {
  const prevProps = current.memoizedProps;
  const nextProps = workInProgress.pendingProps;
  const compare = workInProgress.type.compare ?? shallowEqual;

  if (compare(prevProps, nextProps)) {
    // Bailout — but children might have own state changes
    if ((current.lanes & renderLanes) === NoLanes) {
      return bailoutOnAlreadyFinishedWork(current, workInProgress);
    }
  }

  // Re-render
  // ...
}
```

**Bailout limitations:**

```tsx
const MemoChild = memo(function MemoChild() {
  const [count, setCount] = useState(0);  // own state
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
});

function Parent() {
  return <MemoChild />;
}
// Parent re-renders: MemoChild bailout (no props change)
// MemoChild's own state change: re-renders self
// Bailout doesn't prevent self-state-driven re-renders
```

**`SimpleMemoComponent` optimization:**

```tsx
// Without areEqual — fast path
const Simple = memo(Comp);
// Tag = SimpleMemoComponent (15) — uses shallowEqual directly

// With areEqual — generic path
const Custom = memo(Comp, customCompare);
// Tag = MemoComponent (14) — calls compare function
```

`SimpleMemoComponent` tag check:

```typescript
function updateSimpleMemoComponent(current, workInProgress) {
  if (current !== null) {
    const prevProps = current.memoizedProps;
    if (shallowEqual(prevProps, workInProgress.pendingProps) &&
        current.ref === workInProgress.ref) {
      didReceiveUpdate = false;
      return bailoutOnAlreadyFinishedWork(current, workInProgress);
    }
  }
  return updateFunctionComponent(...);
}
```

**Children prop trap:**

```tsx
const MemoCard = memo(Card);

function App() {
  const [_, setX] = useState(0);
  return (
    <MemoCard>
      <ExpensiveContent />  {/* New element every App render */}
    </MemoCard>
  );
}

// Card props: { children: <ExpensiveContent /> }
// Each render: new element reference
// memo shallow check: children !== prevChildren → fail
// MemoCard re-renders despite memo
```

Workaround:

```tsx
// Move children up
function App() {
  const [_, setX] = useState(0);
  const cachedContent = useMemo(() => <ExpensiveContent />, []);
  return <MemoCard>{cachedContent}</MemoCard>;
}
```

Or render prop:

```tsx
const MemoCard = memo(Card);

function App() {
  return (
    <MemoCard>
      {() => <ExpensiveContent />}  {/* Function — children prop stable per render */}
    </MemoCard>
  );
}
```

But function reference still changes... best:

```tsx
const renderContent = () => <ExpensiveContent />;
function App() {
  return <MemoCard>{renderContent}</MemoCard>;  // hoisted, stable
}
```

**`memo` + `forwardRef`:**

```tsx
// R18 — forwardRef + memo
const Input = memo(forwardRef<HTMLInputElement, Props>(function Input(props, ref) {
  return <input ref={ref} {...props} />;
}));

// R19 — ref as prop, simpler
const Input = memo(function Input({ ref, ...props }: Props & { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...props} />;
});
```

**Custom areEqual — deep equality:**

```tsx
import { isEqual } from "lodash-es";

const DeepMemo = memo(Component, (prev, next) => isEqual(prev, next));

// ⚠️ Deep comparison expensive — only worth if Component render is even more expensive
```

**Performance tradeoff:**

```typescript
// Cost: shallowEqual O(props.length)
// Benefit: skip render (function call + diff + DOM update)

// Worth it when:
// - Component renders heavy DOM
// - Many child fibers
// - Frequent parent renders

// Not worth:
// - Trivial component
// - Props always change (memo never bails)
```

**`React.memo` vs `PureComponent`:**

```tsx
// Old class — PureComponent uses shouldComponentUpdate with shallow check
class OldCard extends React.PureComponent {
  render() {
    return <div>{this.props.name}</div>;
  }
}

// Function equivalent
const NewCard = memo(function NewCard({ name }) {
  return <div>{name}</div>;
});
```

Same shallow check semantic.

**React Compiler implication:**

```tsx
// Pre-Compiler
const MemoCard = memo(Card);
const handleClick = useCallback(() => action(), []);

// Compiler-era
function Card(props) { /* ... */ }  // no manual memo

function Parent() {
  const handleClick = () => action();  // no useCallback
  return <Card onClick={handleClick} />;
}
// Compiler memoizes both Card render and handleClick
```

</details>

### Edge Cases

- **`memo` with no props**: Always bails out (props always shallow equal — both are `{}`).
- **`memo` with `children`**: Children element reference changes → re-render unless explicitly stable.
- **areEqual returning wrong**: Bug-prone. Stale UI if check too lenient.

### Follow-up savollar

- "memo overuse?" — Memoization isn't free. shallowEqual + props copy. Use for expensive components only.
- "memo + Context — does it bypass context?" — No. Context Consumer ALWAYS re-renders on context change, regardless of memo wrapper.
- "memo + reducer dispatch?" — Dispatch reference stable (returned from useReducer). Pass dispatch directly, not closure.

</details>

---

### 20. Output: bailout chain trace [Output] [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi komponent strukturasida click bo'lganda qaysi komponent log qiladi?

```tsx
const MemoChild = memo(function Child({ name }: { name: string }) {
  console.log("Child render:", name);
  return <p>{name}</p>;
});

const MemoExpensive = memo(function Expensive() {
  console.log("Expensive render");
  return <div>expensive</div>;
});

function Sibling() {
  console.log("Sibling render");
  return <div>sibling</div>;
}

function Parent() {
  const [count, setCount] = useState(0);
  console.log("Parent render");
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <MemoChild name="Ali" />
      <MemoExpensive />
      <Sibling />
    </div>
  );
}

function App() {
  console.log("App render");
  return <Parent />;
}

// Initial render → click button
```

### Javob

**Initial render:**
```
App render
Parent render
Child render: Ali
Expensive render
Sibling render
```

**Click button (count: 0 → 1):**
```
Parent render
Sibling render
```

**Tushuntirish:**

State `Parent` ichida o'zgargani uchun:
- `Parent.lanes |= UpdateLane`
- Walk up: `App.childLanes |= UpdateLane` (lekin `App.lanes = 0`)
- Render time'da Reconciler ham `lanes`, ham `childLanes`'ga qaraydi:
  - `App`: `lanes & renderLanes === 0`, `childLanes & renderLanes !== 0` → **bailout self**, recurse children (App'ning render funksiyasi chaqirilmaydi — `console.log` chiqmaydi)
  - `Parent`: `lanes & renderLanes !== 0` → render, "Parent render" log
  - `MemoChild`: memo wrapper, props (`name="Ali"`) o'zgarmadi → bailout (no log)
  - `MemoExpensive`: memo wrapper, props yo'q → har doim equal → bailout (no log)
  - `Sibling`: memo yo'q, parent re-rendered → element identity yangi → render, "Sibling render" log

**Click output:**

```
Parent render
Sibling render
```

`App`, `MemoChild`, `MemoExpensive` log chiqmaydi (bailout).

<details>
<summary><strong>Deep Dive</strong></summary>

**Step-by-step trace:**

```
Initial state:
  Tree: Root → App → Parent → div → [button, MemoChild, MemoExpensive, Sibling]

Click handler runs:
  setCount(c => c + 1)
  → Parent.lanes |= UpdateLane

Walk up to root:
  div.childLanes |= UpdateLane
  Parent.childLanes — wait, Parent.lanes set, not childLanes
  Actually: state lives in Parent fiber → Parent.lanes |= UpdateLane
  Walk up:
    App.childLanes |= UpdateLane
    Root.childLanes |= UpdateLane

Render starts from root:

beginWork(Root):
  Root.lanes & renderLanes = 0 (no own work)
  Root.childLanes & renderLanes != 0 (children have work)
  → bailout self, recurse to App

beginWork(App):
  App.lanes = 0
  App.childLanes != 0
  → bailout self, do NOT call App() function
  → recurse to Parent

  No "App render" logged

beginWork(Parent):
  Parent.lanes != 0 (has pending work)
  → didReceiveUpdate = true
  → call Parent() function
  → "Parent render" logged
  → state queue applied: count = 1
  → returns new JSX tree

Continue with Parent's children:

beginWork(div):
  div is HostComponent, no state
  Reconcile children: button, MemoChild, MemoExpensive, Sibling

beginWork(button):
  Update text "0" → "1"
  No log

beginWork(MemoChild):
  Memo wrapper. Compare props: {name: "Ali"} vs {name: "Ali"}
  shallowEqual → true → bailout
  No "Child render" logged

beginWork(MemoExpensive):
  Memo wrapper. Compare props: {} vs {}
  shallowEqual → true → bailout
  No "Expensive render" logged

beginWork(Sibling):
  Plain function component, no memo
  Sibling.lanes = 0
  Sibling.childLanes = 0
  Sibling has alternate (mounted before)
  But: parent re-rendered with new JSX, current.element !== new.element
  → didReceiveUpdate = true (always true for regular function components when parent re-renders)
  → call Sibling()
  → "Sibling render" logged
```

**Why Sibling re-renders without memo:**

Without memo, plain function components don't have bailout logic. Parent re-renders → creates new JSX `<Sibling />` element → element reference changes → child reconciles → component called.

**`didReceiveUpdate` flag:**

```typescript
let didReceiveUpdate = false;

function beginWork(current, workInProgress, renderLanes) {
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (oldProps !== newProps || hasContextChanged()) {
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, updateLanes)) {
      // No props change, no own update
      didReceiveUpdate = false;
      return attemptEarlyBailoutIfNoScheduledUpdate(...);
    }
  }
  // ...
}
```

**Memo's special bailout:**

```typescript
function updateMemoComponent(current, workInProgress) {
  const prevProps = current.memoizedProps;
  const nextProps = workInProgress.pendingProps;
  const compare = workInProgress.type.compare ?? shallowEqual;

  if (compare(prevProps, nextProps) && current.ref === workInProgress.ref) {
    // Bailout
    didReceiveUpdate = false;
    if ((current.lanes & renderLanes) === NoLanes) {
      return bailoutOnAlreadyFinishedWork(current, workInProgress);
    }
  }
  // Otherwise re-render
}
```

**Output rationale per component:**

- **App**: `console.log` inside App's render. Render not called (bailout). No log.
- **Parent**: render called (state owned). "Parent render" log.
- **MemoChild**: memo bailout. Render not called. No log.
- **MemoExpensive**: memo bailout. Render not called. No log.
- **Sibling**: parent re-rendered → element identity changed → render called. "Sibling render" log.

**Order: depth-first, parent before children:**

```
Parent render  (parent first)
Sibling render (sibling later, after MemoChild and MemoExpensive bailout)
```

</details>

### Edge Cases

- **App had logged "App render" — why initial yes, click no?** Initial mount: no current, all fibers render. Click: bailout chain.
- **State change in Sibling**: Sibling.lanes set. App, Parent bailout (childLanes only). Only Sibling renders.
- **`MemoChild` with name prop changing**: `name="Ali"` → `name="Vali"` shallow check fails → re-render.

### Follow-up savollar

- "App not rendering — JSX tree fresh from where?" — From cached element tree (current fiber). Bailout reuses old children references.
- "How to force App re-render?" — Move state to App (then App owns the lane).
- "Profiler shows App as rendered?" — DevTools "Highlighted Updates" only shows fibers where `PerformedWork` flag set (i.e., render function called).

</details>

---

### 21. Bug fix: stale list bug [Bug Fix] [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi kod bug'ga ega — "Add" tugmasi bosilganda yangi item input'larida eski qiymatlar ko'rinadi. Topish va tuzatish:

```tsx
function ItemEditor() {
  const [items, setItems] = useState([
    { name: "Apple" },
    { name: "Banana" },
  ]);

  const addItem = () => {
    setItems([{ name: "Grape" }, ...items]);
  };

  return (
    <div>
      <button onClick={addItem}>Add at front</button>
      <ul>
        {items.map((item, i) => (
          <li key={i}>
            <input defaultValue={item.name} />
          </li>
        ))}
      </ul>
    </div>
  );
}

// Steps:
// 1. User edits Apple input to "Apple Pie"
// 2. User edits Banana input to "Banana Bread"
// 3. User clicks "Add at front"
// 4. New item added, but inputs show wrong text
```

### Javob

**Bug:**
- After clicking Add: inputs show `["Apple Pie", "Banana Bread", undefined]` instead of `["Grape", "Apple Pie", "Banana Bread"]`.
- Sabab: `key={i}` (index-based key). Items reorder, but Reconciler reuses fibers position-wise (key match → same fiber). `<input defaultValue>` only set on mount — DOM input.value untouched on update. State stays at old position.

**Fix — stable ID:**

```tsx
function ItemEditor() {
  const [items, setItems] = useState([
    { id: "1", name: "Apple" },
    { id: "2", name: "Banana" },
  ]);

  const addItem = () => {
    setItems([{ id: crypto.randomUUID(), name: "Grape" }, ...items]);
  };

  return (
    <div>
      <button onClick={addItem}>Add at front</button>
      <ul>
        {items.map((item) => (
          <li key={item.id}>  {/* ✅ Stable ID */}
            <input defaultValue={item.name} />
          </li>
        ))}
      </ul>
    </div>
  );
}
```

After fix: each input is tied to item identity (id). Add new at front:
- `key="new-uuid"` — no match, new fiber, fresh `<input defaultValue="Grape">`
- `key="1"` — match Apple's old fiber, input retains "Apple Pie"
- `key="2"` — match Banana's old fiber, input retains "Banana Bread"

<details>
<summary><strong>Deep Dive</strong></summary>

**Reconciliation trace (buggy version):**

```
Old fibers (after user edits):
[
  Fiber(li, key=0, child=Fiber(input, DOM.value="Apple Pie")),
  Fiber(li, key=1, child=Fiber(input, DOM.value="Banana Bread"))
]

After addItem() — new elements:
[
  Element(li, key=0, child=<input defaultValue="Grape">),
  Element(li, key=1, child=<input defaultValue="Apple">),
  Element(li, key=2, child=<input defaultValue="Banana">)
]

Reconciler:
  newKeys=[0,1,2], oldKeys=[0,1]
  Phase 1 (fast):
    new[0] (key=0) matches old[0] (key=0)
      Type same (li), reuse fiber
      props: child element changes from "Apple" input to "Grape" input
      Reconcile children of li:
        Old: Fiber(input, DOM.value="Apple Pie")
        New: <input defaultValue="Grape">
        type same (input)
        Reconcile props:
          - defaultValue prop: "Apple" → "Grape"
          - BUT: defaultValue applies only at MOUNT
          - React doesn't update DOM input.value
          - DOM input.value stays "Apple Pie" ← BUG

    new[1] (key=1) matches old[1] (key=1)
      Same logic — DOM.value stays "Banana Bread"

  Phase 3 (mount remaining):
    new[2] — no old match → mount new li with input
      Fresh input, defaultValue="Banana"
      DOM.value = "Banana"

Final DOM:
  <li><input value="Apple Pie"></li>      ← was Apple's, now showing "Grape" item
  <li><input value="Banana Bread"></li>   ← was Banana's, now showing "Apple"
  <li><input value="Banana"></li>          ← new fiber for "Banana"

User confused: typed in Apple's box but seeing "Apple Pie" on "Grape" item.
```

**Reconciliation trace (fixed version):**

```
Old fibers (after user edits):
[
  Fiber(li, key="1", child=Fiber(input, DOM.value="Apple Pie")),
  Fiber(li, key="2", child=Fiber(input, DOM.value="Banana Bread"))
]

After addItem() — new elements:
[
  Element(li, key="new-uuid", child=<input defaultValue="Grape">),
  Element(li, key="1", child=<input defaultValue="Apple">),
  Element(li, key="2", child=<input defaultValue="Banana">)
]

Reconciler:
  Phase 1: keys mismatch first → break
  Phase 4 (keyed Map):
    existingChildren = {"1": fiberApple, "2": fiberBanana}

    new[0] (key="new-uuid") → no match → MOUNT
      New li with fresh input, DOM.value = "Grape"

    new[1] (key="1") → match fiberApple → MOVE
      Reuse fiber, position 0 → 1
      DOM li moved (insertBefore)
      input child: DOM.value preserved = "Apple Pie" ✅

    new[2] (key="2") → match fiberBanana → MOVE
      DOM.value preserved = "Banana Bread" ✅

Final DOM:
  <li><input value="Grape"></li>
  <li><input value="Apple Pie"></li>
  <li><input value="Banana Bread"></li>
✅ All inputs follow item identity
```

**Why `defaultValue` doesn't update:**

```tsx
<input defaultValue="Apple" />
// Mount: React sets DOM input.value = "Apple"
// User types: DOM input.value = "Apple Pie"
// Update render: defaultValue prop changes to "Grape"
//   React: defaultValue is "uncontrolled mount-only" → doesn't touch DOM
//   DOM input.value stays "Apple Pie"
```

**`value` (controlled) wouldn't have this bug:**

```tsx
<input value={item.name} onChange={...} />
// Each render: React sets DOM input.value = props.value
// After reorder with index key: same fiber, but value="Grape" → DOM updated
```

But controlled needs onChange handling (more complex).

**Pattern: stable ID generation:**

```tsx
function ItemEditor() {
  // Initial items might come without IDs (from props)
  const [items, setItems] = useState(() => initialItems.map(i => ({
    ...i,
    id: i.id ?? crypto.randomUUID()
  })));

  // ... rest
}
```

**Counter-based IDs:**

```tsx
function ItemEditor() {
  const [items, setItems] = useState<Array<{id: number; name: string}>>([]);
  const counterRef = useRef(0);

  const addItem = (name: string) => {
    setItems((prev) => [...prev, { id: ++counterRef.current, name }]);
  };

  return items.map((item) => <Item key={item.id} item={item} />);
}
```

**Why this is "list rendering" + "reconciliation" hybrid bug:**

It's about reconciliation (key match → fiber reuse) + DOM behavior (defaultValue mount-only) + state management (uncontrolled state lives in DOM).

Three concepts interact — typical real-world bug.

**Fix patterns:**

1. **Stable ID** — best, semantic correctness
2. **Controlled inputs** — value+onChange, React fully owns state
3. **Force remount** — `key={`${i}-${item.name}`}` — but every char typed → remount
4. **`useState` for input** — local state per item, reset via key

</details>

### Edge Cases

- **All items at front (queue)**: Index key bug for every operation.
- **Filtering removes items**: Index shifts → wrong items appear.
- **Drag-drop reorder**: Index key fully broken.

### Follow-up savollar

- "ESLint rule for this?" — `react/no-array-index-key` — warns. Not bullet-proof (index OK for static lists), but flag worth investigating.
- "Test how to detect?" — RTL: type in inputs, perform reorder, check input values match items.
- "Strict Mode catches?" — No, Strict Mode targets render purity, not key/state issues.

</details>

---

### 22. Update propagation with Context [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Context value o'zgarganda, Reconciler **provider'ning subtree'idagi barcha consumer'larni** topib, ularni `lanes`'ga mark qiladi (qaysi lane'da update bo'lgan). Memo wrapper'lar **bypass qilinadi** — context o'zgarsa, memo bailout ishlamaydi. Lekin **non-consumer fiber'lar** (subtree'da context ishlatmaydigan) — memo bailout normal ishlaydi.

### To'liq tushuntirish

**Context update propagation:**

```typescript
function propagateContextChange(
  workInProgress: Fiber,
  context: ReactContext,
  changedBits: number,
  renderLanes: Lanes
) {
  let fiber = workInProgress.child;

  while (fiber !== null) {
    let nextFiber: Fiber | null;

    // Check if fiber subscribes to this context
    const list = fiber.dependencies?.firstContext;
    if (list !== null) {
      let dependency = list;
      while (dependency !== null) {
        if (dependency.context === context) {
          // Match — schedule update
          fiber.lanes |= renderLanes;
          markUpdateLaneFromFiberToRoot(fiber, renderLanes);
          // Don't propagate further into this fiber's subtree (it'll re-render)
          break;
        }
        dependency = dependency.next;
      }
    }

    // Recurse children
    if (fiber.tag === ContextProvider && fiber.type === workInProgress.type) {
      // Same context provider — children read from new provider, no propagation
      nextFiber = null;
    } else {
      nextFiber = fiber.child;
    }

    if (nextFiber !== null) {
      fiber = nextFiber;
    } else {
      // Walk to sibling/parent
      while (fiber !== null) {
        if (fiber === workInProgress) return;
        if (fiber.sibling) { fiber = fiber.sibling; break; }
        fiber = fiber.return;
      }
    }
  }
}
```

### Kod misoli

```tsx
const ThemeContext = createContext<"light" | "dark">("light");

function App() {
  const [theme, setTheme] = useState<"light" | "dark">("light");
  console.log("App render");
  return (
    <ThemeContext.Provider value={theme}>
      <button onClick={() => setTheme(t => t === "light" ? "dark" : "light")}>
        Toggle
      </button>
      <Tree />
    </ThemeContext.Provider>
  );
}

const Tree = memo(function Tree() {
  console.log("Tree render");
  return (
    <div>
      <DeepConsumer />
      <NonConsumer />
    </div>
  );
});

function DeepConsumer() {
  const theme = useContext(ThemeContext);
  console.log("DeepConsumer render:", theme);
  return <p>Theme: {theme}</p>;
}

const NonConsumer = memo(function NonConsumer() {
  console.log("NonConsumer render");
  return <p>non-consumer</p>;
});

// Initial:
// App render
// Tree render
// DeepConsumer render: light
// NonConsumer render

// Click toggle (theme: light → dark):
// App render
// (Tree memo bailout — context bypass, but Tree itself doesn't consume context)
// DeepConsumer render: dark
// (NonConsumer memo bailout — no context consumption)
```

Wait — actually `Tree` is memo'd. App re-renders, Tree props don't change → memo bailout. But `DeepConsumer` is INSIDE Tree's subtree. How does it re-render?

**Key insight:** Context propagation `useContext` consumer'larini bevosita `lanes`'ga mark qiladi. Memo wrapper'lar context update'ni **bloklamaydi** — Reconciler `childLanes` orqali consumer'gacha yetib boradi va render qiladi.

```
Click → App.lanes |= Lane (theme state)
ContextProvider value changed → propagateContextChange()
  DeepConsumer (uses ThemeContext): DeepConsumer.lanes |= Lane
  markUpdateLaneFromFiberToRoot(DeepConsumer):
    div.childLanes |= Lane
    Tree.childLanes |= Lane
    Provider.childLanes |= Lane

Render:
  App: lanes set (theme state) → "App render"
  Provider: render with new value (value prop changed)
  Tree: lanes=0, childLanes!=0 →
    memo wrapper: shallow check passes (no props change) → bailout call,
    BUT childLanes set → don't skip subtree, recurse children via cloneChildFibers
    (Tree's render function NOT called, no "Tree render" log)
  div: descend
    DeepConsumer: lanes set → "DeepConsumer render: dark"
    NonConsumer: lanes=0, childLanes=0 → skip subtree (no log)
```

**Output:**

```
App render
DeepConsumer render: dark
```

`Tree` va `NonConsumer` bailout — context propagation deep consumer'gacha yetib boradi memo bilan ham.

<details>
<summary><strong>Deep Dive</strong></summary>

**Why memo doesn't block context propagation:**

```typescript
function propagateContextChange(workInProgress, context, renderLanes) {
  let fiber = workInProgress.child;
  while (fiber !== null) {
    if (fiberSubscribesToContext(fiber, context)) {
      fiber.lanes |= renderLanes;  // ← directly mark deep fiber
      // Walk up parents to set childLanes
      markUpdateLaneFromFiberToRoot(fiber, renderLanes);
    }
    // Recurse — DOESN'T STOP at memo
    fiber = nextFiber(fiber);
  }
}
```

Context propagation is direct — marks consumer fiber and propagates `childLanes` up to root.

**`fiber.dependencies` linked list:**

```typescript
type ContextDependency = {
  context: ReactContext;
  next: ContextDependency | null;
};

fiber.dependencies = {
  firstContext: ContextDependency | null,
  lanes: Lanes,
};
```

When `useContext(MyContext)` called:
- Adds entry to `fiber.dependencies.firstContext`
- Reads current value from provider

**Provider-level optimization — same value bailout:**

```typescript
function updateContextProvider(current, workInProgress) {
  const newValue = workInProgress.pendingProps.value;
  const context = workInProgress.type._context;

  if (current && Object.is(current.memoizedProps.value, newValue)) {
    // Value unchanged — don't propagate
    return bailoutOnAlreadyFinishedWork(current, workInProgress);
  }

  // Value changed — propagate
  propagateContextChange(workInProgress, context, renderLanes);

  // Render children
  return workInProgress.child;
}
```

Provider bailout — value reference equality. So:

```tsx
// ❌ New object every render
<ThemeContext.Provider value={{ theme: "light" }}>
  ...
</ThemeContext.Provider>

// Provider always propagates (Object.is fail)
// All consumers re-render every parent re-render

// ✅ Stable value
const value = useMemo(() => ({ theme }), [theme]);
<ThemeContext.Provider value={value}>
  ...
</ThemeContext.Provider>
```

**`use-context-selector` library (dai-shi):**

```tsx
// React core: any context change re-renders ALL consumers
const { theme } = useContext(ThemeContext);

// use-context-selector: subscribes to slice
import { useContextSelector } from "use-context-selector";
const theme = useContextSelector(ThemeContext, (v) => v.theme);
// Re-renders only when v.theme changes
```

External library — uses `useSyncExternalStore` and subscription model. Not part of React core.

**Multiple contexts:**

```tsx
function Component() {
  const theme = useContext(ThemeContext);
  const user = useContext(UserContext);
  const settings = useContext(SettingsContext);

  // dependencies linked list:
  // ThemeContext → UserContext → SettingsContext
}

// Any of 3 contexts changes → component re-renders
```

**Context with memo — subtle:**

```tsx
const MemoChild = memo(function Child() {
  const theme = useContext(ThemeContext);
  return <p>{theme}</p>;
});

// MemoChild bypasses memo when ThemeContext changes
// (context propagation marks lanes directly)
```

But MemoChild bypasses memo for **parent re-render without context change**:

```tsx
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <MemoChild />  {/* Theme didn't change, parent state did */}
    </>
  );
}
// Click: Parent renders, MemoChild bails (memo, no props change)
// Theme change: MemoChild renders despite memo (context lane set)
```

**Context value timing:**

```tsx
function Provider() {
  const [value, setValue] = useState(0);
  return (
    <Context.Provider value={value}>  {/* value=0 */}
      <Consumer />  {/* useContext returns 0 */}
    </Context.Provider>
  );
}

// State change → setValue(1)
// Re-render Provider with value=1
// Reconciler propagates to consumers
// Consumer re-renders with new value
```

**Performance issue — over-propagation:**

```tsx
// Single big context — every consumer re-renders on any state slice change
const AppContext = createContext({ user: null, theme: "light", lang: "en" });

// User changes → consumer subscribed only to theme also re-renders!
// Solution: split contexts
const UserContext = createContext(null);
const ThemeContext = createContext("light");
const LangContext = createContext("en");
```

</details>

### Edge Cases

- **Context with memoized value but property change**: `useMemo(() => ({...}), [a])` — ref stable, but consumer's deep field comparison may need attention.
- **Nested providers with same context**: Inner provider shadows outer. Consumers see inner value.
- **Provider unmounts**: Default value used (from `createContext`).

### Follow-up savollar

- "Why context bypasses memo?" — Marks consumer fiber's lanes directly, so Reconciler must walk to it (childLanes set on path).
- "Selector pattern alternative?" — `useSyncExternalStore`, or external state library (Redux, Zustand) with selectors.
- "Multiple consumers — performance?" — All re-render on context change. Split contexts or use selector library.

</details>

---

### 23. Reconciliation 2 ta heuristic — type identity va keys [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Reconciliation algoritm **2 ta asosiy heuristic** asosida ishlaydi: (1) **Different element types** — tree subtree'ni **butunlay rebuild** qilish (`<div>` → `<span>` = unmount old, mount new). (2) **Keyed lists** — `key` prop bilan stabil identity (ko'p element'larni reorder/move qilish). Bu ikkita heuristic O(n³) → O(n) reduksiyani ta'minlaydi (theoretical optimal: tree edit distance O(n³)).

### To'liq tushuntirish

**Heuristic 1: Different types = full rebuild**

```tsx
// Render 1
<div>
  <Counter />
</div>

// Render 2 — type changed
<span>
  <Counter />
</span>

// Reconciler: <div> → <span>, type different
// Action: unmount Fiber(div), unmount Fiber(Counter), mount Fiber(span), mount Fiber(Counter)
// Counter loses ALL state (new instance)
```

**Heuristic 2: Keys for stable identity**

```tsx
// Render 1
<ul>
  {[<li key="a">A</li>, <li key="b">B</li>, <li key="c">C</li>]}
</ul>

// Render 2 — order changed
<ul>
  {[<li key="c">C</li>, <li key="a">A</li>, <li key="b">B</li>]}
</ul>

// Reconciler: match by key
// Move c to position 0, a to 1, b to 2
// All fibers preserved (state intact)
```

### Kod misoli

**Type change destroys state:**

```tsx
function Toggle() {
  const [showAsDiv, setShow] = useState(true);

  return (
    <button onClick={() => setShow(!showAsDiv)}>
      {showAsDiv
        ? <div><CounterWithState /></div>
        : <section><CounterWithState /></section>
      }
    </button>
  );
}

function CounterWithState() {
  const [count, setCount] = useState(0);
  return <p onClick={() => setCount(c => c + 1)}>{count}</p>;
}

// Toggle clicked:
// <div> → <section> (parent type change)
// CounterWithState — new fiber, count resets to 0
```

**Same type — props update only:**

```tsx
function App() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      <span style={{ color: count > 5 ? "red" : "black" }}>{count}</span>
    </button>
  );
}

// Render 1 → 2: <span style={...black}> → <span style={...black}>
// Same type, props differ
// Reconciler: reuse span fiber, update props (style)
// Span DOM node — same, just style attribute updated
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why 2 heuristics:**

Tree edit distance — algorithmically O(n³) (Tai 1979). React makes 2 simplifying assumptions:

1. **Type assumption**: Different types produce vastly different trees — rebuild faster than diff.
2. **Stability assumption**: Developer provides keys for unstable lists — Reconciler trusts them.

Result: O(n) where n = total nodes.

**Algorithm pseudocode:**

```typescript
function reconcileChildren(parent, oldFirstChild, newChildren) {
  let oldFiber = oldFirstChild;
  let newFiber = null;
  let prevSibling = null;
  let i = 0;

  // First pass — index-based match
  while (oldFiber && i < newChildren.length) {
    const newElement = newChildren[i];

    if (oldFiber.type === newElement.type && oldFiber.key === newElement.key) {
      // Same type + key — reuse
      newFiber = updateFiber(oldFiber, newElement);
    } else if (oldFiber.key === newElement.key) {
      // Same key, different type — should be rare (unmount + mount at same position)
      newFiber = createFiber(newElement);
      oldFiber.flags |= Deletion;
    } else {
      // Key mismatch — bail to keyed pass
      break;
    }

    // ... linked list connections
    i++;
    oldFiber = oldFiber.sibling;
  }

  // Second pass — keyed match (Map-based)
  if (oldFiber !== null) {
    const oldByKey = new Map();
    while (oldFiber) {
      oldByKey.set(oldFiber.key ?? oldFiber.index, oldFiber);
      oldFiber = oldFiber.sibling;
    }

    while (i < newChildren.length) {
      const newElement = newChildren[i];
      const matched = oldByKey.get(newElement.key);
      if (matched) {
        newFiber = updateFiber(matched, newElement);
        oldByKey.delete(newElement.key);
      } else {
        newFiber = createFiber(newElement);
      }
      // ... connect
      i++;
    }

    // Remaining old fibers — mark for deletion
    oldByKey.forEach(fiber => fiber.flags |= Deletion);
  }
}
```

**O(n³) optimal vs O(n) React:**

```
Optimal tree diff:
  Compare every node with every node — find minimal edit sequence
  Worst case: n × n × log(n) operations

React heuristics:
  Type mismatch → rebuild subtree (skip diff inside)
  Keyed match → linear pass with Map lookup
  Result: linear pass, O(n) per render
```

**Heuristic limitations:**

```tsx
// Bad case — type changes for slight variation
function Card({ isPremium }: { isPremium: boolean }) {
  return isPremium
    ? <div className="premium-card">{/* expensive subtree */}</div>
    : <section className="normal-card">{/* same subtree */}</section>;
}

// Toggle isPremium:
// div ↔ section — full unmount + mount (expensive)
// Subtree state lost

// Better: same type, conditional class
function Card({ isPremium }) {
  return (
    <div className={isPremium ? "premium-card" : "normal-card"}>
      {/* subtree preserved */}
    </div>
  );
}
```

**Composition vs reconciliation:**

```tsx
// ❌ Inline component definition — type recreated each render
function Parent() {
  const [count, setCount] = useState(0);

  // ❌ New Inline component reference each render
  const Inline = () => <p>Hello</p>;

  return (
    <button onClick={() => setCount(c => c + 1)}>
      <Inline />  {/* type changes each render → unmount/mount */}
    </button>
  );
}

// ✅ Defined outside
const Outer = () => <p>Hello</p>;
function Parent() {
  return (
    <button>
      <Outer />  {/* same type reference */}
    </button>
  );
}
```

**Conditional rendering — fiber preservation:**

```tsx
{condition ? <ComponentA /> : <ComponentB />}
// condition true → ComponentA
// condition false → ComponentB (different type → full unmount/mount)
// State NOT shared
```

**Same component, different children:**

```tsx
{condition ? <Wrapper>A</Wrapper> : <Wrapper>B</Wrapper>}
// Same Wrapper type
// children prop differ
// Reconciler: reuse Wrapper fiber, update children
// Wrapper state preserved
```

</details>

### Edge Cases

- **Type as string vs function**: `<button>` vs `<Button>` — different types (string vs function). Always different in Reconciler.
- **Anonymous function components**: `() => <div />` defined inline — new function each render. Always different type → unmount+mount.
- **Class component re-render**: Same class, props change — reuse instance, call componentDidUpdate.

### Follow-up savollar

- "Why doesn't React detect 'similar' types (e.g., div with same content as section)?" — O(n³) algorithm. Heuristics simplify.
- "Can I customize reconciliation?" — Limited. `React.memo` for child re-render skip. No full diff customization.
- "Vue's diff algorithm — same?" — Vue 2 — similar heuristics. Vue 3 — fine-grained reactivity (less reliance on diff).

</details>

---

### 24. Element identity bailout — same `<Component />` reference [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Reconciler **element identity bailout** — agar yangi va eski element **same reference** bo'lsa (`Object.is`), Reconciler render'ni butunlay skip qiladi. Bu `useMemo`'da JSX'ni cache qilganda, yoki `children` prop sifatida pass qilganda ishlaydi. Patternlar: `useMemo(() => <Child />, [deps])`, `<Wrapper>{children}</Wrapper>` (children pass-through).

### Kod misoli

```tsx
// 1. useMemo bilan JSX cache
function Parent() {
  const [count, setCount] = useState(0);

  // ✅ Stable element reference — Child re-render skipped
  const cachedChild = useMemo(() => <ExpensiveChild />, []);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      {cachedChild}
    </div>
  );
}

// 2. children prop pattern
function Wrapper({ children }: { children: React.ReactNode }) {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      {children}  {/* same reference — bailout */}
    </div>
  );
}

function App() {
  return (
    <Wrapper>
      <ExpensiveChild />  {/* App render'da bir marta yaratilgan */}
    </Wrapper>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Bailout check in Reconciler:**

```typescript
function reconcileSingleElement(returnFiber, currentFirstChild, element) {
  const key = element.key;
  let child = currentFirstChild;

  while (child !== null) {
    if (child.key === key) {
      if (child.elementType === element.type) {
        // Same type — check element identity
        if (Object.is(child.memoizedProps, element.props)) {
          // ⚡ Bailout: same props reference, skip render
          return cloneAndBailout(child);
        }

        // Same type, different props — update path
        return useFiber(child, element.props);
      }
      break;
    }
    child = child.sibling;
  }

  // Create new fiber
  return createFiber(element);
}
```

**`children` prop — natural bailout:**

```tsx
function Layout() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <Counter count={count} setCount={setCount} />
      <Sidebar />  {/* Sidebar reference changes each render */}
    </div>
  );
}

// Vs children pattern:
function Layout({ children }: { children: React.ReactNode }) {
  const [count, setCount] = useState(0);
  return (
    <div>
      <Counter count={count} setCount={setCount} />
      {children}  {/* Stable from parent */}
    </div>
  );
}

function App() {
  return (
    <Layout>
      <Sidebar />  {/* Created in App render — stable across Layout re-renders */}
    </Layout>
  );
}
```

**`useMemo` JSX caching:**

```tsx
function Parent({ data }: { data: Data }) {
  const [count, setCount] = useState(0);

  // ✅ Stable element if data unchanged
  const expensiveChild = useMemo(
    () => <ExpensiveList data={data} />,
    [data]
  );

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      {expensiveChild}  {/* Reused if data unchanged */}
    </div>
  );
}
```

**Bailout vs `React.memo`:**

| Approach | Mexanizm | Use case |
|----------|----------|----------|
| Element identity | Reference compare | Specific element reused |
| `React.memo` | Shallow props compare | Component reused with potentially different props |
| `useMemo(JSX)` | Element + memo wrapper | Expensive subtree, deps-controlled |
| `children` pass-through | Element stable from parent | Layout/wrapper components |

**Limitation — JSX created in render:**

```tsx
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child />  {/* New element each render — no bailout */}
    </div>
  );
}

// Each render: <Child /> creates new element { type: Child, props: {} }
// Reconciler: new element (reference), reconcile (function call)
// Bailout doesn't kick in
```

**`React.memo` complements:**

```tsx
const Child = React.memo(function Child() {
  console.log("Child render");
  return <p>Child</p>;
});

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child />  {/* New element, but memo bailout (props empty/same) */}
    </div>
  );
}
```

**Element bailout in concurrent rendering:**

Same mechanism in concurrent — Reconciler same logic. Bailout means no work, no scheduling overhead.

</details>

### Edge Cases

- **Spread props**: `<Child {...props} />` — new props object each render, no identity bailout.
- **Inline objects**: `<Child style={{ color: 'red' }} />` — new style object, no bailout.
- **Conditional element**: `{cond && <Child />}` — element created inside ternary. Even if cond stable, element new each render.

### Follow-up savollar

- "Why doesn't React always cache JSX?" — Cost of cache lookup vs render. Default — render. Opt-in via `useMemo`.
- "React Compiler auto-applies this?" — Yes. Compiler memoizes JSX expressions.
- "Element vs Component reference?" — Element = `_jsx(...)` output. Component = function/class definition.

</details>

---

### 25. Output: keyed list reconciliation — insert/move/delete [Output] [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi 2 ta render orasidagi Reconciler harakatlarini izohlang.

```tsx
// Render 1
[
  <Item key="a" />,
  <Item key="b" />,
  <Item key="c" />,
  <Item key="d" />,
]

// Render 2
[
  <Item key="b" />,  // moved from index 1 → 0
  <Item key="d" />,  // moved from index 3 → 1
  <Item key="e" />,  // new (didn't exist in render 1)
  <Item key="a" />,  // moved from index 0 → 3
]
// Note: "c" deleted
```

### Javob

**Reconciler operatsiyalari:**

```
Old: [a, b, c, d]   keys
New: [b, d, e, a]   keys

Step 1: First pass (index match)
  new[0]=b vs old[0]=a → key mismatch, break to second pass

Step 2: Build oldByKey map
  oldByKey = { "a": Fiber(a), "b": Fiber(b), "c": Fiber(c), "d": Fiber(d) }

Step 3: Second pass (keyed match)
  new[0]=b → match Fiber(b) from map → MOVE to index 0, remove from map
  new[1]=d → match Fiber(d) → MOVE to index 1, remove
  new[2]=e → no match → CREATE new Fiber(e)
  new[3]=a → match Fiber(a) → MOVE to index 3, remove

Step 4: Remaining in map → DELETE
  Fiber(c) marked for deletion

Operations:
  - Fiber(b): MOVE (DOM: parent.insertBefore(b.stateNode, ...))
  - Fiber(d): MOVE
  - Fiber(e): CREATE + INSERT
  - Fiber(a): MOVE
  - Fiber(c): DELETE (DOM: parent.removeChild(c.stateNode))
```

**Performance: 4 operations (3 moves + 1 create + 1 delete) — optimal.**

### Tushuntirish

Keysiz (index-based) edi bo'lsa:
```
new[0] vs old[0]: position 0, props change (a → b)
new[1] vs old[1]: position 1, props change (b → d)
new[2] vs old[2]: position 2, props change (c → e)
new[3] vs old[3]: position 3, props change (d → a)
```

Hech qanday move yo'q, ammo har bir Fiber NEW props oladi — state confusion (input value, scroll, etc. wrong row'da).

<details>
<summary><strong>Deep Dive</strong></summary>

**DOM operations sequence:**

```
Initial DOM:
<ul>
  <li id="a"></li>  ← Fiber(a)
  <li id="b"></li>  ← Fiber(b)
  <li id="c"></li>  ← Fiber(c)
  <li id="d"></li>  ← Fiber(d)
</ul>

After reconciliation operations:
<ul>
  <li id="b"></li>  ← moved (insertBefore)
  <li id="d"></li>  ← moved
  <li id="e"></li>  ← created + inserted
  <li id="a"></li>  ← moved
  // <li id="c"></li> removed
</ul>
```

**Reconciler internal — keyed iteration:**

```typescript
function reconcileChildrenArray(returnFiber, currentFirstChild, newChildren) {
  let oldFiber = currentFirstChild;
  let newIdx = 0;
  let lastPlacedIndex = 0;
  let nextOldFiber = null;
  let resultingFirstChild = null;
  let previousNewFiber = null;

  // First pass: index-based match
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    const newFiber = updateSlot(returnFiber, oldFiber, newChildren[newIdx]);

    if (newFiber === null) break;  // mismatch — break to keyed

    if (oldFiber && newFiber.alternate === null) {
      // Was created (mismatch) — old needs deletion
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

  // Second pass: keyed map
  if (oldFiber !== null && newIdx < newChildren.length) {
    const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = updateFromMap(existingChildren, returnFiber, newIdx, newChildren[newIdx]);

      if (newFiber !== null) {
        if (newFiber.alternate !== null) {
          existingChildren.delete(newFiber.key ?? newIdx);
        }
        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        // ... linked list
      }
    }

    // Mark remaining for deletion
    existingChildren.forEach(child => deleteChild(returnFiber, child));
  }

  return resultingFirstChild;
}
```

**`placeChild` — move detection:**

```typescript
function placeChild(newFiber, lastPlacedIndex, newIndex) {
  newFiber.index = newIndex;

  const current = newFiber.alternate;
  if (current !== null) {
    const oldIndex = current.index;
    if (oldIndex < lastPlacedIndex) {
      // Move — needs flag
      newFiber.flags |= Placement;
      return lastPlacedIndex;
    } else {
      // No move — children stays in place
      return oldIndex;
    }
  } else {
    // New — needs Placement
    newFiber.flags |= Placement;
    return lastPlacedIndex;
  }
}
```

**Why `lastPlacedIndex`:**

Algorithm tracks "last placed index" to detect moves. If `oldIndex < lastPlacedIndex` — must move (out of order).

```
Old: [a(0), b(1), c(2), d(3)]
New: [b, d, e, a]

new[0]=b: oldIndex=1, lastPlaced=0 → b stays in place (no move?), lastPlaced=1
  Wait, this is weird because new position is 0, but b was at 1.
  Actually: lastPlaced tracks oldIndex relative ordering.
  b.oldIndex=1 ≥ lastPlaced=0 → no move flag, lastPlaced=1

new[1]=d: oldIndex=3, lastPlaced=1 → 3≥1, no move, lastPlaced=3

new[2]=e: alternate=null → CREATE + Placement flag, lastPlaced=3

new[3]=a: oldIndex=0, lastPlaced=3 → 0<3 → MOVE, Placement flag
```

Algorithm minimizes moves — only items that "go backwards" relative to last placed need moving.

**O(n) achievement:**

- First pass: O(n) — single linear walk
- Map build: O(n) — remaining old fibers
- Second pass: O(n) — single walk with O(1) map lookup
- Total: O(n)

</details>

### Edge Cases

- **All keys same**: `key="x"` for all — first match, others duplicate warning, undefined behavior.
- **No keys at all**: Index-based first pass succeeds (no key check). Good for static lists.
- **Keys outside range**: `key={Math.random()}` — every render new keys, no matches, full rebuild.
- **Nested keys**: Outer list keys + inner list keys independent (different sibling sets).

### Follow-up savollar

- "Optimal move algorithm exists?" — Yes (Longest Increasing Subsequence). Vue 3 uses LIS for fewer DOM ops. React uses simpler O(n) heuristic.
- "What if I `key={i}` and items reorder?" — Same fibers reused (positions match), but state attached to position, not item — bug.
- "Performance — 1000 item reorder cost?" — O(n) reconciliation + O(moves) DOM ops. DOM mutation cost tree hajmi va browser engine'ga bog'liq.

</details>

---

## QISM C: Scheduler & Lanes

<a id="qism-c"></a>

### 26. Concurrent rendering nima uchun kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Concurrent rendering** (R18+) — render phase'ni **interruptible** qilish, **prioritization** qo'shish. Sabab: legacy sync rendering — long task'lar (1000+ komponent re-render) browser'ni bloklab, user input lag, animation drop hosil qiladi. Concurrent — high-pri (input) low-pri (background data, transition)'ni interrupt qila oladi, 5ms slice'larda yield qilib, browser'ga vaqt beradi.

### To'liq tushuntirish

**Sync rendering muammolari:**

```
Long task: 100ms render
Browser blocked entire 100ms
- No frame paint (60fps = 16.67ms budget — exceeded 6x)
- User input ignored
- Animation jank
- Lighthouse Total Blocking Time: bad
```

**Concurrent rendering yechimi:**

```
Long render split into 5ms slices:
[5ms work][yield][5ms work][yield][5ms work][yield]...
- Browser repaints between slices
- User input handled mid-render
- Animation smooth
- High-priority updates can interrupt low-priority
```

**Use cases:**

1. **Long lists** — 1000+ items render
2. **Search/filter** — typing during result render
3. **Tab switching** — heavy tab content
4. **Animations** — running while data updates
5. **Suspense boundaries** — async loading without blocking

### Kod misoli

```tsx
// 1. useTransition — mark non-urgent updates
function Search() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const newQuery = e.target.value;
    setQuery(newQuery);  // urgent — keystroke immediate

    startTransition(() => {
      // Non-urgent — can be interrupted
      setResults(searchItems(newQuery));
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <ResultsList items={results} />
    </>
  );
}

// User types fast: input updates immediately (urgent),
// results render in background, can be interrupted by next keystroke
```

```tsx
// 2. useDeferredValue — defer expensive child render
function ProductPage() {
  const [filter, setFilter] = useState("");
  const deferredFilter = useDeferredValue(filter);

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      {/* Heavy filtered list uses deferred value */}
      <ExpensiveList filter={deferredFilter} />
    </>
  );
}
// Input updates immediately, ExpensiveList renders with old value first,
// then re-renders with new (concurrent, interruptible)
```

```tsx
// 3. Suspense — async loading
function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <AsyncDashboard />
    </Suspense>
  );
}
// AsyncDashboard suspends (Promise thrown)
// Concurrent: render fallback, continue when resolved
// Without concurrent: blocking
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Without concurrent (sync) — flame graph:**

```
| 100ms render long task                    | UI updates  |
|↑                                            |
User clicks button                          ← input ignored 100ms
```

**With concurrent — flame graph:**

```
| 5ms || 5ms || 5ms || 5ms || 5ms || 5ms |...|  paints  |
|     ↑     ↑     ↑     ↑                |
       paint  user input handled
              (interrupt low-pri render)
```

**Priority levels (lanes):**

| Priority | Lane | Use case |
|----------|------|----------|
| Sync (Immediate) | SyncLane | flushSync, mount commit |
| Continuous | InputContinuousLane | drag, scroll, mousemove |
| Default | DefaultLane | useState in event handler |
| Transition | TransitionLane | useTransition, useDeferredValue |
| Idle | IdleLane | offscreen pre-render |

**Time slicing implementation:**

```typescript
const yieldInterval = 5; // ms
let deadline = 0;

function workLoop() {
  deadline = performance.now() + yieldInterval;

  while (nextUnitOfWork && !shouldYield()) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }

  if (nextUnitOfWork) {
    // Yielded — schedule continuation
    scheduleCallback(NormalPriority, workLoop);
  } else {
    // Done
    commitRoot();
  }
}

function shouldYield() {
  return performance.now() >= deadline;
}
```

**Interrupt mechanism:**

```typescript
// Low-pri render in progress
let lowPriRoot = scheduleRender(WIPRoot, DefaultLane);
// 50% complete...

// User input — high pri
const inputUpdate = scheduleRender(WIPRoot, SyncLane);
// React: detect higher priority lane
// → abandon current low-pri WIP
// → restart with high-pri lanes
// → after high-pri commit, retry low-pri
```

**Restart safety — render purity:**

Concurrent requires render functions to be **pure**:
- No side effects in render body (no setState, no DOM mutation)
- Idempotent — same props/state produces same JSX
- No reading mutable values (Date.now, Math.random)

Strict Mode 2x render — catches violations.

**Tearing prevention:**

Without concurrent: state read once, render committed atomically. With concurrent: state could change mid-render (multiple updates).

`useSyncExternalStore` — ensures external store reads consistent across re-tries.

```tsx
// External store + concurrent
const cart = useSyncExternalStore(cartStore.subscribe, cartStore.getSnapshot);
// React ensures `cart` consistent across render attempts
```

**Migration from R17:**

```tsx
// R17 (legacy mode)
ReactDOM.render(<App />, container);
// All renders sync, no concurrent

// R18 (concurrent mode)
const root = createRoot(container);
root.render(<App />);
// Concurrent enabled, opt-in features
```

`createRoot` — concurrent default. Features (`useTransition`) opt-in usage.

**Browser thread sharing:**

JS single-threaded. React work shares main thread with:
- Browser layout/paint
- User input handling
- Other JS (timers, microtasks)

5ms slice — leaves time for browser tasks. 60fps target frame budget.

</details>

### Edge Cases

- **`flushSync`** — bypasses concurrent, force sync render. Use sparingly (DOM measurement, animation step).
- **Event handlers** — sync by default (urgent). Wrap in `startTransition` for non-urgent updates.
- **Render abort** — partial render thrown away, retried later. Effects don't run for abandoned renders.

### Follow-up savollar

- "Concurrent rendering opt-in or default?" — `createRoot` opts into concurrent capability. Features (`useTransition`, Suspense for SSR, etc.) — usage is opt-in via API.
- "Performance better with concurrent?" — Total work isn't faster, but **perceived** performance better (responsive UI). Frame budget respected.
- "Web Workers vs concurrent rendering?" — Workers — separate thread, but no DOM access. Concurrent — main thread, time slicing. Different problem domains.

</details>

---

### 27. Scheduler package va priority levels [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Scheduler** (`scheduler` package) — alohida package, React'dan ajratilgan, `requestIdleCallback` analogi. **Priority levels** (5 ta): **Immediate** (0), **UserBlocking** (250ms timeout), **Normal** (5000ms timeout), **Low** (10000ms timeout), **Idle** (no timeout). Har task priority bilan scheduled, scheduler queue'ga qo'shiladi, browser idle bo'lganda chaqiriladi (`MessageChannel`).

### To'liq tushuntirish

**Scheduler API:**

```typescript
// scheduler package
import {
  unstable_scheduleCallback as scheduleCallback,
  unstable_cancelCallback as cancelCallback,
  unstable_NormalPriority as NormalPriority,
  unstable_UserBlockingPriority as UserBlockingPriority,
  unstable_ImmediatePriority as ImmediatePriority,
  unstable_LowPriority as LowPriority,
  unstable_IdlePriority as IdlePriority,
} from "scheduler";

// Schedule a callback
const taskHandle = scheduleCallback(NormalPriority, () => {
  console.log("Task ran");
});

// Cancel
cancelCallback(taskHandle);
```

**Priority levels and timeouts:**

```typescript
// Priority levels (internal enum values):
// ImmediatePriority = 1, UserBlockingPriority = 2,
// NormalPriority = 3, LowPriority = 4, IdlePriority = 5

// Timeout values (ms) per priority:
const PriorityTimeouts = {
  ImmediatePriority: -1,        // expires immediately (darhol ishga tushadi)
  UserBlockingPriority: 250,    // 250ms — input, click handlers
  NormalPriority: 5000,         // 5s — default
  LowPriority: 10000,           // 10s — non-critical
  IdlePriority: maxSigned31BitInt, // ~12.4 kun (amalda cheksiz)
};
```

### Kod misoli

```typescript
// Scheduler internal usage in React
function scheduleUpdateOnFiber(fiber: Fiber, lane: Lane) {
  const priorityLevel = lanesToPriority(lane);
  scheduleCallback(priorityLevel, performWorkOnRoot);
}

function lanesToPriority(lanes: Lanes): SchedulerPriority {
  if (lanes & SyncLane) return ImmediatePriority;
  if (lanes & InputContinuousLane) return UserBlockingPriority;
  if (lanes & DefaultLane) return NormalPriority;
  if (lanes & TransitionLane) return NormalPriority;
  if (lanes & IdleLane) return IdlePriority;
  return NormalPriority;
}
```

```tsx
// User-level: priority through APIs

// Sync (Immediate)
flushSync(() => {
  setState(newValue);
});

// User Blocking (event handlers — default)
function handleClick() {
  setState(newValue);  // UserBlocking priority
}

// Normal (default state changes)
useEffect(() => {
  setState(newValue);  // Normal priority
}, []);

// Transition (low priority, interruptible)
startTransition(() => {
  setState(newValue);  // Transition lane → NormalPriority but interruptible
});
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Scheduler implementation (sodda):**

```typescript
// scheduler/src/forks/Scheduler.ts (simplified)

type Task = {
  id: number;
  callback: Function | null;
  priorityLevel: number;
  startTime: number;
  expirationTime: number;
};

let taskQueue: Task[] = [];        // ready tasks (min-heap by expirationTime)
let timerQueue: Task[] = [];       // delayed tasks (min-heap by startTime)
let currentTask: Task | null = null;
let isPerformingWork = false;
let isHostCallbackScheduled = false;

function scheduleCallback(priorityLevel: number, callback: Function, options?: { delay: number }) {
  const currentTime = performance.now();
  const startTime = options?.delay ? currentTime + options.delay : currentTime;

  let timeout;
  switch (priorityLevel) {
    case ImmediatePriority: timeout = -1; break;
    case UserBlockingPriority: timeout = 250; break;
    case IdlePriority: timeout = MAX_SAFE_INTEGER; break;
    case LowPriority: timeout = 10000; break;
    case NormalPriority:
    default: timeout = 5000; break;
  }

  const expirationTime = startTime + timeout;

  const task: Task = {
    id: nextTaskId++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
  };

  if (startTime > currentTime) {
    // Delayed — push to timer queue
    push(timerQueue, task);
  } else {
    // Ready — push to task queue
    push(taskQueue, task);
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return task;
}
```

**Work loop (`flushWork`):**

```typescript
function flushWork(hasTimeRemaining: boolean, initialTime: number): boolean {
  isPerformingWork = true;
  let currentTime = initialTime;

  try {
    advanceTimers(currentTime);  // Move ready timers to taskQueue
    currentTask = peek(taskQueue);

    while (currentTask !== null) {
      if (currentTask.expirationTime > currentTime && (!hasTimeRemaining || shouldYieldToHost())) {
        // No time remaining, yield to host
        break;
      }

      const callback = currentTask.callback;
      if (typeof callback === "function") {
        currentTask.callback = null;
        const continuationCallback = callback(currentTask.expirationTime <= currentTime);
        currentTime = performance.now();

        if (typeof continuationCallback === "function") {
          // Task wants to continue — keep in queue
          currentTask.callback = continuationCallback;
        } else {
          // Done — remove from queue
          if (currentTask === peek(taskQueue)) {
            pop(taskQueue);
          }
        }

        advanceTimers(currentTime);
      }

      currentTask = peek(taskQueue);
    }

    return currentTask !== null;
  } finally {
    isPerformingWork = false;
  }
}
```

**`requestHostCallback` — yield to browser:**

```typescript
function requestHostCallback(callback: Function) {
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    schedulePerformWorkUntilDeadline();
  }
}

const schedulePerformWorkUntilDeadline = (() => {
  // MessageChannel — fast yield
  const channel = new MessageChannel();
  const port = channel.port2;

  channel.port1.onmessage = performWorkUntilDeadline;

  return () => port.postMessage(null);
})();

function performWorkUntilDeadline() {
  if (scheduledHostCallback !== null) {
    const currentTime = performance.now();
    deadline = currentTime + yieldInterval;

    const hasMoreWork = scheduledHostCallback(true, currentTime);

    if (hasMoreWork) {
      schedulePerformWorkUntilDeadline();  // continue next slice
    } else {
      isMessageLoopRunning = false;
      scheduledHostCallback = null;
    }
  }
}
```

**`shouldYieldToHost`:**

```typescript
function shouldYieldToHost() {
  return performance.now() >= deadline;
}
```

**Scheduler yield constants (React internal):**

```typescript
// React 18+ Scheduler internal'da uchta konstanta:
const frameYieldMs = 5;        // default yield budget
const continuousYieldMs = 50;  // continuous priority task max work
const maxYieldMs = 300;        // long-task yield ceiling
```

"5ms" — `frameYieldMs` default. Continuous va max yieldMs alohida holatlar uchun.

**Priority comparison:**

```typescript
// expirationTime smaller = higher priority (expires sooner)
// Min-heap on expirationTime — peek() returns next to run

// Example:
// Immediate: now + (-1) = now - 1 (darhol expired)
// UserBlocking: now + 250
// Normal: now + 5000
// Idle: now + 1073741823 (≈12.4 kun)

// Heap order: Immediate < UserBlocking < Normal < Idle
```

**Starvation prevention:**

```typescript
// Idle priority expirationTime maxSigned31BitInt (≈12.4 kun) — amalda expire bo'lmaydi
// Lekin high-pri task'lar orasida idle vaqt bo'lsa, idle task ham ishlaydi

// Real starvation case:
// Continuous high-pri tasks (typing) — low-pri starves
// Solution: lane expirationTime moves up
```

R18 React doesn't strictly use scheduler timeouts for lane priority; lanes have separate expiration logic.

**Custom scheduler — replacement:**

```typescript
// React internal abstraction:
import * as Scheduler from "scheduler";

// You can replace with custom (advanced):
// - requestIdleCallback only (Safari has limited support)
// - setImmediate (Node.js)
// - Custom polyfill
```

Default: `MessageChannel` (works everywhere, fast).

**`scheduler-tracing`:**

Experimental package for tracing render work — DevTools integration.

**`requestIdleCallback` — why not used:**

```typescript
requestIdleCallback(callback, { timeout: 100 });
// - Variable timing (no fixed slice)
// - Safari 16.4+ (2023) qo'shildi, lekin React allaqachon MessageChannel tanlagan
// - Long task budget unpredictable

// vs MessageChannel:
// - Fixed 5ms slice
// - All browsers support
// - Predictable
```

</details>

### Edge Cases

- **Very long callback (>5ms)**: Yield happens between callbacks (between fibers in React's case), not mid-callback.
- **Sync priority deadlock**: `flushSync` inside another flushSync — handled (already in batch context).
- **Browser tab inactive**: Scheduler tasks may pause (browser throttles inactive tabs).

### Follow-up savollar

- "Why separate scheduler package?" — Reusable for non-React work. Future RFC: standardize browser scheduler API (`postTask`).
- "`postTask` API replacement?" — Newer browser API for priority-based scheduling. Polyfill for older browsers via `MessageChannel`.
- "Custom priority?" — Internal scheduler accepts only 5 levels. React lanes — finer-grained (32 lanes), but mapped to 5 priorities.

</details>

---

### 28. Lanes model bitmap [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Lanes** (R18+) — 31-bit bitmap har Fiber'da. Har bit — alohida priority lane: `SyncLane` (highest), `InputContinuousLane`, `DefaultLane`, `TransitionLane` (16 ta), `RetryLane`, `IdleLane` (lowest), va boshqalar. Updates har xil lane'larga belong qiladi (priority bo'yicha). Reconciler render lanes'ni tanlaydi va shu lanes'dagi work'ni bajaradi. **OR/AND** bit operations — fast scheduling.

### To'liq tushuntirish

**Lane definitions (sodda):**

```typescript
const NoLanes: Lanes = 0b0000000000000000000000000000000;
const SyncHydrationLane: Lane = 0b0000000000000000000000000000001;
const SyncLane: Lane = 0b0000000000000000000000000000010;
const InputContinuousHydrationLane: Lane = 0b0000000000000000000000000000100;
const InputContinuousLane: Lane = 0b0000000000000000000000000001000;
const DefaultHydrationLane: Lane = 0b0000000000000000000000000010000;
const DefaultLane: Lane = 0b0000000000000000000000000100000;
// Transition lanes — 16 lanes for granular transitions
const TransitionHydrationLane: Lane = 0b0000000000000000000000000100000;
const TransitionLane1: Lane = 0b0000000000000000000000001000000;
const TransitionLane2: Lane = 0b0000000000000000000000010000000;
// ...
const TransitionLane16: Lane = 0b0000000000000000010000000000000;

const RetryLane1: Lane = 0b0000000000000000100000000000000;
// ... 4 retry lanes

const SelectiveHydrationLane: Lane = 0b0000000010000000000000000000000;
const IdleHydrationLane: Lane = 0b0000000100000000000000000000000;
const IdleLane: Lane = 0b0000001000000000000000000000000;
const OffscreenLane: Lane = 0b0000010000000000000000000000000;
```

### Kod misoli

```typescript
// Bit operations on lanes
const fiberLanes: Lanes = SyncLane | DefaultLane;  // OR — combine

// Check if has SyncLane
if (fiberLanes & SyncLane) {
  // Has sync work
}

// Remove SyncLane
const remainingLanes = fiberLanes & ~SyncLane;

// Get highest priority
function getHighestPriorityLane(lanes: Lanes): Lane {
  return lanes & -lanes;  // Bit trick: isolate lowest set bit
}

// Get lanes priority order:
const lanes = SyncLane | DefaultLane | IdleLane;
let highest = getHighestPriorityLane(lanes);
console.log(highest === SyncLane);  // true
lanes &= ~highest;
highest = getHighestPriorityLane(lanes);
console.log(highest === DefaultLane);  // true
```

```typescript
// Real React internals example
function scheduleUpdateOnFiber(fiber: Fiber, lane: Lane) {
  // Mark fiber and ancestors
  fiber.lanes |= lane;
  let parent = fiber.return;
  while (parent !== null) {
    parent.childLanes |= lane;
    parent = parent.return;
  }

  // Schedule render
  ensureRootIsScheduled(rootFiber);
}

function getNextLanes(root: FiberRoot, wipLanes: Lanes): Lanes {
  const pendingLanes = root.pendingLanes;
  if (pendingLanes === NoLanes) return NoLanes;

  // Highest priority pending lanes
  const nonIdlePendingLanes = pendingLanes & NonIdleLanes;
  if (nonIdlePendingLanes !== NoLanes) {
    return getHighestPriorityLanes(nonIdlePendingLanes);
  }
  return getHighestPriorityLanes(pendingLanes);
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Lane vs Priority — relationship:**

```typescript
// Priority is a concept (Immediate, UserBlocking, Normal, Low, Idle)
// Lane is a specific bit position within priority groups

// Sync lanes (highest)
SyncLane = 0b10        // bit 1
// Input continuous (high)
InputContinuousLane = 0b1000  // bit 3
// Default (normal user interactions)
DefaultLane = 0b100000  // bit 5
// Transitions (lower priority, can be interrupted)
TransitionLane1, TransitionLane2, ..., TransitionLane16

// 16 transition lanes — for handling many concurrent transitions
// Each transition gets a unique lane
```

**Why 16 transition lanes:**

```tsx
// Multiple concurrent transitions
function App() {
  const [, startSearch] = useTransition();
  const [, startSort] = useTransition();
  const [, startFilter] = useTransition();

  // Each gets its own lane
  // Reconciler can prioritize differently
  // Multiple transitions in flight: each tracked separately
}
```

**Render lanes selection:**

```typescript
function performConcurrentWorkOnRoot(root: FiberRoot) {
  const lanes = getNextLanes(root, root.workInProgressLanes);
  if (lanes === NoLanes) return null;

  // Render with these lanes
  renderRootConcurrent(root, lanes);

  // ... commit if successful
}
```

**Lane intersection — bailout decision:**

```typescript
function beginWork(current: Fiber, workInProgress: Fiber, renderLanes: Lanes) {
  // Has work in current render lanes?
  if (workInProgress.lanes & renderLanes) {
    // Render this fiber
  } else if (workInProgress.childLanes & renderLanes) {
    // Children have work — bailout self, recurse
  } else {
    // No work in render lanes — skip subtree
    return null;
  }
}
```

**Priority bumping (starvation):**

```typescript
function markStarvedLanesAsExpired(root: FiberRoot, currentTime: number) {
  let lanes = root.pendingLanes;
  while (lanes !== NoLanes) {
    const lane = getHighestPriorityLane(lanes);
    if (root.expirationTimes[laneIndex] <= currentTime) {
      // Lane expired — bump priority (treated as Sync)
      root.expiredLanes |= lane;
    }
    lanes &= ~lane;
  }
}
```

If transition starves (waits too long), expired and prioritized.

**Multiple updates same lane:**

```typescript
// Two setState in event handler — both DefaultLane
setName("Ali");      // DefaultLane
setAge(25);          // DefaultLane

// fiber.lanes already has DefaultLane after first setState
// Second OR-equal — no change
fiber.lanes |= DefaultLane;  // idempotent
```

Both updates in same lane → same render cycle (batching).

**Lane vs lanes (singular vs plural):**

- `Lane` (singular) — single bit, e.g., `SyncLane`
- `Lanes` (plural) — bitmask of multiple lanes, e.g., `SyncLane | DefaultLane`

**Bit isolation tricks:**

```typescript
function getHighestPriorityLane(lanes: Lanes): Lane {
  return lanes & -lanes;
  // Two's complement trick
  // -lanes flips all bits + 1 (e.g., 0b1100 → 0b...0011 + 1 = 0b...0100)
  // AND with original isolates lowest set bit
}

// Examples:
// lanes = 0b1100 → 0b0100 (highest priority bit, lower index)
// lanes = 0b0001 → 0b0001
// lanes = 0b1000000 → 0b1000000
```

**`pickArbitraryLane`:**

```typescript
function pickArbitraryLane(lanes: Lanes): Lane {
  return getHighestPriorityLane(lanes);
}
// Picks one lane (highest priority bit)
```

**Hydration lanes — separate:**

Each priority has hydration variant:
- `SyncHydrationLane` — sync hydration boundary
- `InputContinuousHydrationLane`
- `DefaultHydrationLane`
- `TransitionHydrationLane`

Hydration lanes — special-case during SSR hydration.

**`SuspenseLane` (renamed RetryLane in R18):**

```typescript
const RetryLane1: Lane = 0b...;
// 4 retry lanes — for Suspense retries when promise resolves
```

When suspended subtree's promise resolves, retry render in retry lane.

**Scheduling decision flow:**

```
1. setState() called
   → fiber.lanes |= lane (based on context: SyncLane, DefaultLane, TransitionLane, ...)
   → walk up: parents.childLanes |= lane

2. ensureRootIsScheduled():
   → root.pendingLanes |= lane
   → schedule render via Scheduler with appropriate priority

3. workLoop():
   → renderLanes = getNextLanes(root)
   → renderRoot(renderLanes)
   → for each fiber: check fiber.lanes & renderLanes

4. Commit:
   → root.pendingLanes &= ~renderedLanes (clear committed)
   → effects run
```

**Lane in action — concurrent example:**

```tsx
function Search() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  return (
    <>
      <input
        onChange={(e) => {
          setQuery(e.target.value);  // SyncLane (input event)
          startTransition(() => {
            setResults(search(e.target.value));  // TransitionLane1
          });
        }}
        value={query}
      />
      <ResultsList items={results} />
    </>
  );
}

// Fast typing:
// 1. Type "a" → SyncLane work (query update) + TransitionLane1 work (results)
// 2. SyncLane completes (input updates immediately)
// 3. TransitionLane1 starts...
// 4. Type "b" before TransitionLane1 completes
//    → SyncLane (new query) + TransitionLane2 (new results)
// 5. SyncLane priority > TransitionLane1 → interrupt TransitionLane1
// 6. SyncLane completes, then TransitionLane2
// 7. TransitionLane1 abandoned (newer transition supersedes)
```

</details>

### Edge Cases

- **All lanes empty (`NoLanes`)**: No render scheduled.
- **Lanes spanning many priorities**: Render highest priority lanes first, then re-render for remaining.
- **Lane exhaustion**: 16 transition lanes max. Older transitions abandoned if exceeded.

### Follow-up savollar

- "Why bitmap not array?" — Bit ops O(1), no allocation. Hot path.
- "Why 31 bits not 32?" — JS number — 32-bit integer for bitwise ops, but signed (sign bit reserved).
- "Lane debugging?" — DevTools doesn't show lanes. Internal. Manual logging via React internals (advanced).

</details>

---

### 29. Time slicing — 5ms work loop [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Time slicing** — render work'ni **5ms slice'lar**ga ajratish. Har 5ms'da `shouldYield()` true qaytaradi → work loop chiqadi → browser idle bo'ladi (input handle, paint, animation) → React `MessageChannel` orqali next slice schedule qiladi → kontinuatsiya. Bu UI responsiveness'ni saqlash uchun asosiy mexanizm.

### To'liq tushuntirish

**5ms tanlovi:**

- 60fps target → 16.67ms frame budget
- React work — 5ms (max ~1/3 frame)
- Qolgan ~11ms — browser layout, paint, input, JS

### Kod misoli

```typescript
// React Scheduler — work loop with time slicing
const yieldInterval = 5; // ms
let deadline = 0;

function workLoop(): void {
  deadline = performance.now() + yieldInterval;

  while (nextUnitOfWork !== null && !shouldYield()) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }

  if (nextUnitOfWork === null) {
    // All work done
    commitRoot();
  } else {
    // Yielded — schedule continuation
    scheduleNextSlice();
  }
}

function shouldYield(): boolean {
  return performance.now() >= deadline;
}

function scheduleNextSlice(): void {
  // MessageChannel — fast yield
  port.postMessage(null);
}

// MessageChannel onmessage handler
port1.onmessage = function() {
  workLoop();  // continue
};
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why 5ms (not 16ms):**

```
60fps frame budget: 16.67ms

If React uses entire frame:
- No buffer for browser tasks (layout, paint, input)
- Risk: dropped frames if anything else blocks

5ms allocation:
- React: 5ms work
- Browser: 11ms (layout, paint, input handlers, idle JS)
- Healthy buffer
```

**`MessageChannel` for yielding:**

```typescript
const channel = new MessageChannel();
const port1 = channel.port1;
const port2 = channel.port2;

port1.onmessage = function(event) {
  // Runs in next macrotask
  performWork();
};

// Schedule
port2.postMessage(null);
```

`MessageChannel` properties:
- **Macrotask** — between frames (after paint)
- **Fast** — no `setTimeout` 4ms minimum delay
- **Universal** — all browsers support
- **Predictable** — no `requestIdleCallback` variability

**Alternative scheduling strategies:**

```typescript
// 1. setTimeout(0) — old approach
setTimeout(performWork, 0);
// Issues: 4ms minimum delay, throttled in inactive tabs

// 2. requestIdleCallback — variable
requestIdleCallback(performWork, { timeout: 100 });
// Issues: Safari spotty support, unpredictable timing

// 3. MessageChannel — React's choice
port.postMessage(null); // → onmessage handler
// Best: fast, predictable, universal

// 4. requestAnimationFrame — paint-tied
requestAnimationFrame(performWork);
// Issues: paint-aligned, blocks paint if work too long

// 5. Microtask (Promise) — too eager
Promise.resolve().then(performWork);
// Issues: doesn't yield to browser paint
```

**Yield points — between fibers:**

```typescript
function workLoop() {
  while (nextUnitOfWork && !shouldYield()) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    // Yield only between fibers, not mid-fiber
  }
}
```

If a single component takes >5ms (heavy computation in render), can't yield mid-component.

```tsx
// ❌ Heavy synchronous render
function HeavyComponent() {
  // 50ms computation
  const result = heavyCompute();
  return <div>{result}</div>;
}
// React can't yield during this
// 50ms blocking task

// ✅ Memoize / split
const result = useMemo(() => heavyCompute(), [deps]);
```

**Yield intervals (React Scheduler internal):**

```typescript
// React Scheduler — three constants:
const frameYieldMs = 5;         // default yield budget
const continuousYieldMs = 50;   // continuous priority work cap
const maxYieldMs = 300;         // ceiling for long-running work
```

Default `5ms` — eng ko'p ishlatiladi. `shouldYield()` shu deadline'ga qaraydi.

**Concurrent rendering scenario:**

```
T0: render starts (1000 components, ~50ms total work)

T0-5ms: process ~100 components
T5ms: shouldYield() → true → exit loop
T5ms: postMessage to schedule continuation
T5-11ms: browser handles paint, input

T11ms: MessageChannel onmessage fires → workLoop again
T11-16ms: process ~100 more components
T16ms: yield, paint
... continue

T50ms: all components processed
T50ms: commit (sync, ~1ms)
T51ms: paint with new UI
```

**Interrupt during slice:**

```
T0: render low-pri (DefaultLane) starts
T0-5ms: 100 components processed
T5ms: yield

T5-7ms: user clicks button
T7ms: setState → SyncLane priority

T11ms: onmessage → workLoop
  Check: pendingLanes higher priority?
  YES — abandon low-pri WIP
  Restart with SyncLane (immediate, no slicing)
T11-15ms: SyncLane render (sync)
T15ms: commit, paint

Resume low-pri? Yes — re-schedule on next slice
T16ms: low-pri restart
```

**`flushSync` bypass time slicing:**

```typescript
function flushSync(fn) {
  // Force sync — no slicing
  const prevExecutionContext = executionContext;
  executionContext |= BatchedContext;
  try {
    fn();
  } finally {
    executionContext = prevExecutionContext;
    flushSyncCallbacks();  // sync flush
  }
}
```

`flushSync` — no yield, runs to completion regardless of duration. Use sparingly.

**Performance impact:**

```typescript
// Without time slicing
// 50ms work — 1 frame dropped (if 16ms budget)
// User input lag: 50ms

// With time slicing
// 50ms work split into 10x 5ms slices
// Total wall time: ~50-60ms (slight overhead)
// User input lag: 5ms (next slice can yield to input)
// Frames: smooth
```

**`shouldYield` cost:**

```typescript
function shouldYield() {
  return performance.now() >= deadline;
}
```

`performance.now()` — sub-microsecond. Called between fibers (~0.001ms overhead per call). Negligible.

**Browser tab inactive:**

When tab inactive, browser throttles `MessageChannel` (similar to setTimeout). React work pauses. Resumes when tab visible.

**Web Worker alternative — not used:**

Web Workers — separate thread, no DOM access. React work needs DOM (commit, refs). Time slicing solves the problem within main thread constraints.

**Offscreen rendering (R19 Activity):**

```tsx
import { unstable_Activity as Activity } from "react";

<Activity mode="hidden">
  <ExpensivePage />  {/* Pre-render in idle, no commit */}
</Activity>
```

R19 Activity — render offscreen content during idle (Idle priority lane). Fast switch when needed.

</details>

### Edge Cases

- **Single very long component**: Can't yield mid-component. Optimize component or split.
- **Synchronous task in event handler**: Bypasses yielding (sync priority).
- **`while(true)` in component**: Infinite — can't yield. Browser hang.

### Follow-up savollar

- "Yield interval configurable?" — Internal, not user-configurable. R18 internal default 5ms.
- "Web Workers for React?" — `react-three-fiber` partially uses Workers (3D scene). Main React DOM — not feasible.
- "RAF (requestAnimationFrame) better?" — RAF tied to paint cycle, blocks paint if work too long. MessageChannel decoupled.

</details>

---

### 30. Interruptible rendering — high priority interrupt [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Interruptible rendering** — low priority render mid-flight, high priority update keladi → low-pri render **abandon qilinadi** (workInProgress tashlanadi, current daxlsiz qoladi), high-pri uchun yangi WIP yaratiladi va render boshlanadi. Low-pri render keyinroq qayta urinilishi mumkin (re-schedule). Render purity kafolat — abort/retry safe.

### To'liq tushuntirish

**Interrupt mexanizmi:**

```typescript
function performConcurrentWorkOnRoot(root: FiberRoot) {
  // Get current rendering lanes
  const lanes = getNextLanes(root, root.workInProgressLanes);

  // Higher priority work appeared?
  if (root.pendingLanes & higherPriorityThan(lanes)) {
    // Abandon current WIP
    abandonWorkInProgress();
    // Re-schedule with new (higher) priority
    return performConcurrentWorkOnRoot(root);
  }

  // Continue with current lanes
  renderRootConcurrent(root, lanes);

  // ... commit
}
```

**Abort safety — render purity:**

- Render phase mutates only WIP fiber tree (DOM untouched)
- Aborted render → WIP discarded (no state lost in current)
- Effects ran? No — effects only after commit. Aborted render = no effects.
- State queue updates persist (still in pending lanes)

### Kod misoli

```tsx
// Real-world: search with concurrent rendering
function SearchPage() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const newQuery = e.target.value;
    setQuery(newQuery);  // SyncLane (high priority)

    startTransition(() => {
      setResults(searchItems(newQuery));  // TransitionLane (low priority)
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <ResultsList items={results} />  {/* Heavy render */}
    </>
  );
}

// Fast typing scenario:
// T0: type "a" → SyncLane (query) + TransitionLane (results for "a")
//     SyncLane render: query="a" (input updates immediately, ~1ms)
//     TransitionLane render starts (heavy ResultsList)
// T5: 5ms slice processed, yield
// T11: continue TransitionLane render...
// T15: type "b" → new SyncLane (query="b") + new TransitionLane (results for "ab")
//     React: SyncLane higher priority → abandon current TransitionLane WIP
//     SyncLane render: query="ab" (input updates)
// T16: continue with new TransitionLane (results for "ab")
//     Old TransitionLane render thrown away (no commit, no effects)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Render abort during work loop:**

```typescript
function workLoop() {
  while (nextUnitOfWork && !shouldYield()) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }

  if (nextUnitOfWork) {
    // Yielded — but check for higher priority
    const nextLanes = getNextLanes(root, workInProgressLanes);
    if (nextLanes !== workInProgressLanes &&
        higherPriority(nextLanes, workInProgressLanes)) {
      // New higher priority — abandon current WIP
      abandonRender();
      // Schedule new render
      ensureRootIsScheduled(root);
      return;
    }

    // Same priority — continue
    scheduleCallback(workLoop);
  } else {
    commitRoot();
  }
}
```

**`abandonRender`:**

```typescript
function abandonRender() {
  // workInProgress tree daxlsiz qoladi (alternate slot)
  // Just clear the active render state
  workInProgressRoot = null;
  workInProgressLanes = NoLanes;
  workInProgress = null;
  // Pending lanes still on root
  // → re-render later from current
}
```

**Pending lanes persist:**

```typescript
// Before render
root.pendingLanes = TransitionLane1 | DefaultLane;

// Start TransitionLane render
root.workInProgressLanes = TransitionLane1;
root.pendingLanes = DefaultLane;  // remaining

// New SyncLane update during render
root.pendingLanes |= SyncLane;
// → SyncLane higher priority

// Abandon TransitionLane render
root.pendingLanes |= TransitionLane1;  // restore (re-add)
root.workInProgressLanes = NoLanes;
```

After abandonment, TransitionLane work re-added to pendingLanes — eligible for next render.

**Effects don't run on aborted render:**

```typescript
// Effects scheduled in commit phase
function commitRoot(root: FiberRoot, finishedWork: Fiber) {
  // ... mutation
  // ... layout effects
  scheduleCallback(NormalPriority, flushPassiveEffects);
}

// Aborted render — no commit
// → flushPassiveEffects NOT called
// → no effect runs

// Render-time side effects (in component body) — those would run!
// That's why purity matters — no DOM mutation in render
```

**Re-render after abort:**

```typescript
// After SyncLane render completes, abandoned TransitionLane re-render:
function performConcurrentWorkOnRoot(root) {
  const nextLanes = getNextLanes(root, NoLanes);
  // → TransitionLane1 (still pending)
  renderRootConcurrent(root, nextLanes);
  // Render with current state (latest)
  // Old WIP discarded — start fresh from current
  commitRoot(root);
}
```

**State coherence — render uses latest state:**

```tsx
function Component() {
  const [count, setCount] = useState(0);
  // ...
}

// Multiple updates pending
// setState(5) → DefaultLane
// setState(10) → DefaultLane

// Render with state queue applied:
// state = 0 (initial)
// state = 5 (apply first)
// state = 10 (apply second)
// memoizedState = 10
```

**Concurrent rendering invariants:**

For interrupt safety, components must satisfy:

1. **Render is pure** — no side effects in body
2. **State immutable** — only setState modifies
3. **Refs not read in render** — refs unstable across renders
4. **External data via `useSyncExternalStore`** — consistent reads

**Tearing prevention:**

```tsx
// External store read directly in render (BAD)
function App() {
  const count = externalStore.value;  // ❌ unsafe
  return <p>{count}</p>;
}

// External store can change mid-render (interruptible)
// One render path: count=5
// Re-render after interrupt: count=6
// Visible inconsistency

// ✅ useSyncExternalStore — guarantees consistency
function App() {
  const count = useSyncExternalStore(
    externalStore.subscribe,
    externalStore.getSnapshot
  );
  return <p>{count}</p>;
}
```

**`useTransition` API:**

```tsx
const [isPending, startTransition] = useTransition();

startTransition(() => {
  setState(newValue);  // marked as TransitionLane
});

// isPending: true while transition is rendering
// false when committed
```

`startTransition` — wraps setState calls in transition lane priority.

**`useDeferredValue`:**

```tsx
const value = "fast input";
const deferredValue = useDeferredValue(value);

// deferredValue — lags behind value during high pri renders
// Updates with TransitionLane priority
```

**Concurrent vs sync mode tradeoffs:**

| Aspect | Sync (legacy) | Concurrent (R18+) |
|--------|---------------|-------------------|
| Render mode | Atomic, blocking | Interruptible, sliced |
| Render purity | Recommended | Required |
| Effects | Run after sync render | Run after concurrent commit |
| Throughput | Slightly higher | Slightly lower (yield overhead) |
| Latency | Fixed wait | Variable, but UI responsive |
| User experience | Janky on long renders | Smooth |

**Render abort frequency in production:**

- Most renders complete without interruption
- Abort happens during fast typing, drag-drop, animation + state change combo
- Aniq abort frekansi app pattern'iga bog'liq (interactive app'larda ko'proq)

</details>

### Edge Cases

- **All updates in same lane**: No interrupt — single render cycle.
- **Strict Mode + abort**: Strict 2x render means double work. Abort scenarios more common during dev.
- **Class component side effects in render**: Buggy with concurrent. Strict Mode catches via 2x render warning.

### Follow-up savollar

- "Render abort observable from user code?" — No direct API. Indirect: effects don't run for aborted render. `useEffect` runs only after commit.
- "Performance impact of frequent aborts?" — CPU wasted on aborted work. But UI smoother — net positive UX.
- "Can I prevent interrupt?" — `flushSync` — sync render, can't be interrupted (by design — for cases needing immediate result).

</details>

---

### 31. Starvation prevention — expiration mechanism [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Starvation** — low-priority work doim high-priority bilan interrupt qilinadi, hech qachon commit bo'lmaydi. **Yechim**: har lane uchun **expiration time** — agar lane ko'p kutsa, `expirationTime` o'tib ketadi → lane **expired** deb belgilanadi → priority bumped (Sync sifatida treated). Bu fairness kafolat: hech qanday update cheksiz uzoq kutmaydi.

### To'liq tushuntirish

**Expiration mexanizmi:**

```typescript
type LaneExpirations = number[];  // 31 ta lane uchun expiration time

function markStarvedLanesAsExpired(root: FiberRoot, currentTime: number) {
  const pendingLanes = root.pendingLanes;
  const expirationTimes = root.expirationTimes;

  let lanes = pendingLanes;
  while (lanes !== NoLanes) {
    const lane = pickArbitraryLane(lanes);
    const laneIndex = laneToIndex(lane);
    const expirationTime = expirationTimes[laneIndex];

    if (expirationTime === NoTimestamp) {
      // First time this lane is pending — set expiration
      expirationTimes[laneIndex] = computeExpirationTime(lane, currentTime);
    } else if (expirationTime <= currentTime) {
      // Expired — bump priority
      root.expiredLanes |= lane;
    }

    lanes &= ~lane;
  }
}

function computeExpirationTime(lane: Lane, currentTime: number): number {
  switch (lane) {
    case SyncLane:
    case InputContinuousLane:
      return currentTime + 250;  // 250ms timeout
    case DefaultLane:
      return currentTime + 5000;  // 5s
    case TransitionLane1:
      return currentTime + 5000;
    case IdleLane:
      return NoTimestamp;  // Idle lane expiration set qilinmaydi (starvation'dan himoyalanmagan)
    default:
      return currentTime + 5000;
  }
}
```

### Kod misoli

```tsx
// Starvation scenario
function App() {
  const [, startTransition] = useTransition();
  const [data, setData] = useState<Data>(initial);

  // Continuous high-pri updates (e.g., animation, mouse tracking)
  useEffect(() => {
    const id = setInterval(() => {
      setMousePos(getMousePosition());  // SyncLane (urgent)
    }, 16);
    return () => clearInterval(id);
  }, []);

  // Low-pri update (transition)
  const refreshData = () => {
    startTransition(() => {
      setData(fetchData());  // TransitionLane
    });
  };

  // Without expiration:
  // setData() in transition lane
  // Continuous mouse updates higher priority
  // setData() never gets render slot — STARVATION

  // With expiration:
  // setData() lane expirationTime = +5s
  // After 5s, lane marked expired
  // Forced into Sync mode → renders next opportunity
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Lane expiration timeline:**

```
T=0: setData() — TransitionLane1
     pendingLanes |= TransitionLane1
     expirationTimes[TransitionLane1Index] = 5000

T=0-5s: continuous SyncLane updates (mouse tracking)
        each render: TransitionLane1 deferred (lower priority)

T=5s: markStarvedLanesAsExpired runs
       expirationTime (5000) <= currentTime (5000)
       expiredLanes |= TransitionLane1

T=5s+: getNextLanes detects expiredLanes
        treats as Sync priority
        next render includes TransitionLane1 work
```

**`markStarvedLanesAsExpired` integration:**

```typescript
function performConcurrentWorkOnRoot(root: FiberRoot, didTimeout: boolean) {
  const currentTime = performance.now();

  // Check for expired lanes — bump priority
  markStarvedLanesAsExpired(root, currentTime);

  // Get lanes for this render
  const lanes = getNextLanes(root, root.workInProgressLanes);

  // If has expired lanes — render synchronously
  const shouldTimeSlice =
    !includesBlockingLane(root, lanes) &&
    !includesExpiredLane(root, lanes);

  if (shouldTimeSlice) {
    renderRootConcurrent(root, lanes);
  } else {
    renderRootSync(root, lanes);  // sync — no yielding
  }
}
```

**Expired lanes — sync mode:**

```typescript
function renderRootSync(root: FiberRoot, lanes: Lanes) {
  // Sync render — no time slicing, no yielding
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
    // No shouldYield() check
  }

  commitRoot(root);
}
```

When lane expired:
- Time slicing disabled
- Sync render to completion
- Browser may block 50+ms (acceptable trade-off vs starvation)

**Lane index calculation:**

```typescript
function laneToIndex(lane: Lane): number {
  return 31 - Math.clz32(lane);
}

// SyncLane (0b1) — index 0
// InputContinuousLane (0b100) — index 2
// DefaultLane (0b10000) — index 4
// ...
```

**Expiration timeouts (R18 sodda):**

```typescript
const SyncLaneTimeout = -1;          // immediate (allaqachon eng yuqori)
const InputContinuousTimeout = 250;  // 250ms
const DefaultTimeout = 5000;         // 5s
const TransitionTimeout = 5000;      // 5s
// IdleLane — expiration set qilinmaydi (NoTimestamp)

// Lower priority — longer timeout (more patient)
```

**Continuous high-pri scenario:**

```tsx
function App() {
  const [count, setCount] = useState(0);
  const [pos, setPos] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handler = (e: MouseEvent) => {
      setPos({ x: e.clientX, y: e.clientY });  // continuous SyncLane
    };
    document.addEventListener("mousemove", handler);
    return () => document.removeEventListener("mousemove", handler);
  }, []);

  return (
    <>
      <div style={{ position: "absolute", left: pos.x, top: pos.y }}>•</div>
      <ExpensiveComponent count={count} />  {/* updates with count */}
    </>
  );
}

// User: setCount(5) (DefaultLane)
// Mouse moves continuously: SyncLane updates
// DefaultLane renders deferred...
// After 5s, DefaultLane expires → forced sync render
// User sees count update (after up to 5s lag)
```

This 5s wait is the worst case — typical apps don't hit it because high-pri work doesn't continuously fire.

**Re-priority — render restart:**

```typescript
function ensureRootIsScheduled(root: FiberRoot) {
  const nextLanes = getNextLanes(root, NoLanes);
  const newCallbackPriority = lanesToEventPriority(nextLanes);

  if (newCallbackPriority !== root.callbackPriority) {
    // Cancel previous, schedule new
    if (root.callbackNode !== null) {
      cancelCallback(root.callbackNode);
    }
    root.callbackNode = scheduleCallback(newCallbackPriority, performWorkOnRoot);
    root.callbackPriority = newCallbackPriority;
  }
}
```

When lane expires, priority changes → re-schedule.

**Adaptive timeouts (future):**

R18+ Adaptive scheduling — react to device conditions:
- High-end device: 5ms slice, 5s transition timeout
- Low-end: 16ms slice, 10s timeout (more patient)

**Mitigation strategies:**

If starvation observed:
1. Throttle/debounce high-pri updates
2. Use `useDeferredValue` instead of `useTransition` (React handles deferral)
3. Move heavy work off render path (Web Worker, server)

**Suspense + expiration:**

```tsx
<Suspense fallback={<Spinner />}>
  <SlowAsync />
</Suspense>
```

Suspense uses retry lanes. Each retry has timeout. After timeout, fallback shown (avoid waiting forever).

**Test scenario — manual starvation:**

```tsx
function StarvationDemo() {
  const [, startTransition] = useTransition();
  const [count, setCount] = useState(0);

  // Continuous high-pri
  useEffect(() => {
    const id = setInterval(() => {
      setCount(Date.now());  // forces re-render every tick
    }, 1);
    return () => clearInterval(id);
  }, []);

  // Low-pri (would starve without expiration)
  const handleClick = () => {
    startTransition(() => {
      setData(heavyOperation());
    });
  };
}
```

</details>

### Edge Cases

- **Idle lane never expires?**: IdleLane'ga expiration set qilinmaydi (`NoTimestamp`). Faqat high-pri task'lar orasida idle vaqt bo'lganda ishlaydi.
- **Sync lane already expired**: Default expiration -1 (immediate) — always rendered first.
- **Concurrent expired lanes**: All forced sync, rendered together.

### Follow-up savollar

- "Expiration tunable?" — Internal constants. Not user-configurable.
- "Frequent expiration — bug?" — Yes, indicates app overloading high-pri updates. Profile, throttle.
- "Can I observe expiration?" — Indirect. DevTools Profiler shows commit timing — if transition takes 5s+, likely expired.

</details>

---

### 32. `requestIdleCallback` vs `MessageChannel` — React's choice [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React **`MessageChannel`** ishlatadi (`requestIdleCallback` emas). Sabab: (1) `requestIdleCallback` Safari'da 16.4+ (2023) da qo'shildi, lekin React allaqachon o'z scheduler'ini yozgan edi, (2) timing **variable** (browser idle detection), (3) idle slot davomiyligi kafolatlanmaydi. `MessageChannel` — universal, fast, predictable 5ms slice. React custom scheduler `requestIdleCallback` funksionalligini o'zi implement qiladi.

### To'liq tushuntirish

**`requestIdleCallback` API:**

```typescript
requestIdleCallback((deadline: IdleDeadline) => {
  while (deadline.timeRemaining() > 0 && hasMoreWork) {
    doWork();
  }
}, { timeout: 1000 });

interface IdleDeadline {
  timeRemaining(): number;  // ms remaining in idle slot
  didTimeout: boolean;       // true if timeout exceeded
}
```

**`MessageChannel` API:**

```typescript
const channel = new MessageChannel();
channel.port1.onmessage = () => {
  performWork();
};

// Schedule
channel.port2.postMessage(null);
```

### Kod misoli

```typescript
// React Scheduler abstraction
const requestHostCallback = (() => {
  if (typeof MessageChannel !== "undefined") {
    // Use MessageChannel (preferred)
    const channel = new MessageChannel();
    const port = channel.port2;
    channel.port1.onmessage = () => performWorkUntilDeadline();

    return (callback) => {
      scheduledHostCallback = callback;
      port.postMessage(null);
    };
  } else if (typeof setImmediate !== "undefined") {
    // Node.js fallback
    return (callback) => {
      scheduledHostCallback = callback;
      setImmediate(performWorkUntilDeadline);
    };
  } else {
    // setTimeout fallback (4ms minimum delay)
    return (callback) => {
      scheduledHostCallback = callback;
      setTimeout(performWorkUntilDeadline, 0);
    };
  }
})();
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Comparison table:**

| Method | Browser support | Timing | Predictability | Speed |
|--------|-----------------|--------|----------------|-------|
| `MessageChannel` | All (IE10+) | Macrotask | Predictable | Fast |
| `requestIdleCallback` | Chrome/FF/Safari 16.4+ | Variable (idle-driven) | Unpredictable | Variable |
| `setTimeout(0)` | All | Macrotask | Predictable | 4ms min delay |
| `setImmediate` | Node.js, IE | Macrotask | Predictable | Fast (Node only) |
| `requestAnimationFrame` | All | Paint-aligned | Predictable | Frame-rate |
| Microtask (Promise) | All | Microtask | Same task | Very fast |

**Why React doesn't use `requestIdleCallback`:**

```typescript
// Issue 1: Safari 16.4+ (2023) — React'ning scheduler'i bundan oldin yaratilgan
// Oldingi Safari'larda polyfill kerak edi

// Issue 2: Variable timing
requestIdleCallback((deadline) => {
  console.log(deadline.timeRemaining());
  // Could be 0ms (no idle), 50ms (lots of idle)
  // Unpredictable
});

// Issue 3: Idle detection heuristics differ
// Browser-specific: Chrome's "idle" != Firefox's "idle"

// Issue 4: Long timeout option doesn't always honored
requestIdleCallback(callback, { timeout: 100 });
// Sometimes fires later than 100ms
```

**Why `MessageChannel`:**

```typescript
// Pros:
// 1. Universal — IE10+, all modern browsers
// 2. Fast — minimal overhead (no setTimeout 4ms delay)
// 3. Macrotask — yields to browser layout/paint between calls
// 4. Predictable — deterministic timing

// Cons:
// 1. Doesn't know about CPU idle (always schedules)
// 2. React must implement own time slicing logic (5ms deadline)
```

**`MessageChannel` implementation details:**

```typescript
function performWorkUntilDeadline() {
  if (scheduledHostCallback !== null) {
    const currentTime = performance.now();
    deadline = currentTime + yieldInterval;  // 5ms

    const hasMoreWork = scheduledHostCallback(true, currentTime);

    if (hasMoreWork) {
      // More work — schedule next slice
      port.postMessage(null);
    } else {
      isMessageLoopRunning = false;
      scheduledHostCallback = null;
    }
  }
}

function shouldYieldToHost() {
  return performance.now() >= deadline;
}
```

**Scheduling timing flow:**

```
T0: setState → scheduleCallback(NormalPriority, performWork)
T0: requestHostCallback(performWork)
T0: port.postMessage(null)
T0+0.1ms: onmessage fires → performWorkUntilDeadline
T0+0.1: deadline = T0+5.1ms
T0+0.1 to T0+5: scheduledHostCallback(true, currentTime)
                  → React work
                  → shouldYieldToHost() returns false
                  → continue working
T0+5: shouldYieldToHost() returns true
T0+5: scheduledHostCallback returns true (more work)
T0+5: port.postMessage(null) (schedule next slice)
T0+5 to T0+~16: browser handles paint, input, etc.
T0+~16: onmessage fires (next macrotask)
T0+16: continue work...
```

**`postTask` API (future browser standard):**

```typescript
// Newer (2026) — not yet universal
scheduler.postTask(callback, { priority: "user-blocking" });

// Priority: "user-blocking", "user-visible", "background"
// Future: React might migrate when adoption sufficient
```

**Custom scheduler — replacing default:**

```typescript
// scheduler package — internal, not user-facing
// Advanced: replace via package.json resolutions or custom build
```

Most apps don't customize.

**Performance — qualitative:**

```typescript
// MessageChannel — minimal scheduling overhead, no enforced delay
scheduleCallback(() => doWork());

// setTimeout — HTML spec'ga ko'ra 4ms minimum delay (nested timeout'larda kuchayadi)
setTimeout(() => doWork(), 0);

// React scheduler MessageChannel tanlagani: scheduling overhead deyarli nol va
// 4ms minimum delay yo'q
```

**Yield to specific browser tasks:**

```
MessageChannel onmessage = macrotask
Between macrotasks:
- Browser repaint (16ms cycle)
- Layout/style recalc
- User input dispatch
- IndexedDB transactions
- HTML parsing
- Image decode
```

By yielding via macrotask, React allows browser to handle these.

**`requestAnimationFrame` — when used:**

React doesn't use `rAF` for general scheduling. But:
- Animation libraries use `rAF` for smooth 60fps animations
- React's `useLayoutEffect` runs synchronously in commit (before paint)
- `<ViewTransition>` (R19) integrates with browser View Transitions API (paint-aligned)

**Microtask vs macrotask yielding:**

```typescript
// Microtask — same task, no browser yield
queueMicrotask(callback);
Promise.resolve().then(callback);
// Doesn't yield to browser paint

// Macrotask — between tasks, browser yields
setTimeout(callback, 0);
MessageChannel.postMessage();
// Browser can paint, handle input
```

React needs **macrotask** for proper UI responsiveness. Microtasks would be too eager (no paint between).

**`MessageChannel` browser support:**

```
Chrome: 4+ (2010)
Firefox: 41+ (2015)
Safari: 5+ (2010)
IE: 10+
Node.js: 11+ (worker_threads), polyfill before
```

Universal coverage for modern apps.

**Polyfill for Node.js:**

```typescript
// Node.js < 11
import { MessageChannel } from "worker_threads";
// Or polyfill: setImmediate
```

`setImmediate` Node-only equivalent.

**`process.nextTick` (Node.js):**

```typescript
process.nextTick(callback);
// Microtask — too eager for time slicing
// Doesn't yield to event loop
```

Not used for React scheduling.

</details>

### Edge Cases

- **Tab inactive**: `MessageChannel` throttled (browser behavior). React work pauses.
- **`postMessage` failure**: Should not happen normally (no payload). Fallback to setTimeout if needed.
- **Custom port message handlers**: Could interfere if user code uses MessageChannel — React uses dedicated channel.

### Follow-up savollar

- "Why not Web Workers?" — DOM access required for commit phase. Workers no DOM.
- "`postTask` adoption?" — Chrome supports, Firefox/Safari behind. React monitors but hasn't migrated.
- "Can React work in older browsers?" — IE10+ — `MessageChannel` available. IE9 — setTimeout fallback (slower).

</details>

---

### 33. Lane operations — bitmap manipulation [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React Lanes — **31-bit bitmap** (TypeScript: `Lanes = number`). Har lane bir priority level. Operations: `mergeLanes(a, b) = a | b` (combine), `intersectLanes(a, b) = a & b` (overlap), `removeLanes(set, removed) = set & ~removed`, `getHighestPriorityLane(lanes) = lanes & -lanes` (rightmost bit). Bitmap'lar tez (CPU-level), 31 lanes har xil priority darajasi yetadi.

### Kod misoli

```typescript
// React internal lanes (sodda)
const NoLanes              = 0b0000000000000000000000000000000;
const SyncLane             = 0b0000000000000000000000000000010;  // bit 1
const InputContinuousLane  = 0b0000000000000000000000000001000;  // bit 3
const DefaultLane          = 0b0000000000000000000000000100000;  // bit 5
const TransitionLane1      = 0b0000000000000000000001000000000;
const TransitionLane2      = 0b0000000000000000000010000000000;
const IdleLane             = 0b0010000000000000000000000000000;

// Operations
function mergeLanes(a: Lanes, b: Lanes): Lanes {
  return a | b;
}

function intersectLanes(a: Lanes, b: Lanes): Lanes {
  return a & b;
}

function removeLanes(set: Lanes, subset: Lanes): Lanes {
  return set & ~subset;
}

function getHighestPriorityLane(lanes: Lanes): Lane {
  return lanes & -lanes;  // isolate rightmost (highest pri) bit
}

function includesSomeLane(set: Lanes, subset: Lanes): boolean {
  return (set & subset) !== NoLanes;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why bitmap:**

- **CPU operations** — bitwise ops are fast (single instruction)
- **Compact** — 31 lanes in 4 bytes (vs array of priority labels)
- **Composable** — combine multiple lanes with single OR
- **Comparable** — set intersection with single AND

**Lane assignment per update:**

```typescript
function requestUpdateLane(fiber): Lane {
  if (currentEventTransitionLane !== NoLane) {
    return currentEventTransitionLane;  // useTransition context
  }

  if (isInsideEventHandler) {
    return SyncLane;  // user input
  }

  return DefaultLane;  // generic
}
```

**Per-update tracking:**

```typescript
function scheduleUpdateOnFiber(fiber, lane) {
  const root = getRootForUpdatedFiber(fiber);

  // Mark lane as pending
  root.pendingLanes |= lane;

  // Mark fiber subtree
  let parent = fiber.return;
  while (parent !== null) {
    parent.childLanes |= lane;
    parent = parent.return;
  }
}
```

**Render phase — lane filtering:**

```typescript
function performWorkOnRoot(root) {
  // Pick highest priority lane
  const nextLanes = getHighestPriorityLanes(root);

  // Render only fibers in those lanes
  renderRootSync(root, nextLanes);
}

function shouldFiberRender(fiber, lanes) {
  return (fiber.lanes & lanes) !== 0 ||
         (fiber.childLanes & lanes) !== 0;
}
```

**Bailout via lane check:**

```typescript
function bailoutOnAlreadyFinishedWork(workInProgress, renderLanes) {
  if (!includesSomeLane(workInProgress.childLanes, renderLanes)) {
    // No work in this subtree for these lanes — skip
    return null;
  }
  // Re-render children
}
```

**Multiple priorities — lane interleaving:**

```typescript
// User input (SyncLane) interrupts background work (DefaultLane)
root.pendingLanes = SyncLane | DefaultLane;

// performWorkOnRoot picks highest first
const nextLanes = getHighestPriorityLanes(root);  // SyncLane

// After SyncLane render done, schedule DefaultLane
root.pendingLanes &= ~SyncLane;
// Continue with DefaultLane
```

**Lane priority hierarchy:**

```
SyncLane (bit 1)          ← highest, sync render
InputContinuousLane (bit 3) ← drag, scroll
DefaultLane (bit 5)       ← generic setState
TransitionLane1-16        ← useTransition
RetryLane                 ← Suspense retry
IdleLane                  ← lowest, requestIdleCallback-like
```

**31-bit constraint:**

JavaScript bitwise ops 32-bit signed integer sifatida ishlaydi. Sign bit (bit 31) manfiy son uchun — React undan foydalanmaydi, shuning uchun 31 lane mavjud. `maxSigned31BitInt = 1073741823` (2^31 - 1, ≈12.4 kun ms sifatida).

</details>

### Edge Cases

- **`includesAllLanes`**: `(set & subset) === subset` — all subset lanes in set.
- **`isSubsetOfLanes`**: Same as includesAllLanes.
- **Lane overflow (>31 transitions)**: React reuses transition lanes (round-robin).

### Follow-up savollar

- "Why 31 lanes?" — JS bitwise ops 32-bit signed, sign bit reserved.
- "Can I create custom lanes?" — No, React-internal. `useTransition` provides API.
- "Does Vue have lanes?" — Vue 3 — `nextTick` queue, simpler priority model.

</details>

---

### 34. `useTransition` internals — TransitionLane assignment [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useTransition()` returns `[isPending, startTransition]`. `startTransition(callback)` ichidagi setState'lar **TransitionLane**'ga (low priority) assign qilinadi. `isPending` — transition lane'da pending work bor-yo'qligi (boolean). React render time'da lower priority lane'larni yield qiladi → user input (SyncLane) bloklamaydi.

### Kod misoli

```tsx
function SearchPage() {
  const [query, setQuery] = useState("");
  const [isPending, startTransition] = useTransition();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value);  // SyncLane (urgent)

    startTransition(() => {
      setSearchResults(filter(items, e.target.value));  // TransitionLane (non-urgent)
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <Results />
    </>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`startTransition` internal:**

```typescript
function startTransition(scope) {
  const prevTransition = currentTransition;
  currentTransition = { /* transition object */ };

  // Assign TransitionLane to setState'lar within scope
  const previousLane = currentEventTransitionLane;
  currentEventTransitionLane = pickArbitraryLane(TransitionLanes);

  try {
    scope();  // ← setState chaqiruvlari shu lane'ga assigned
  } finally {
    currentEventTransitionLane = previousLane;
    currentTransition = prevTransition;
  }
}
```

**Lane priority — render order:**

```typescript
// User types in input (SyncLane) + transition pending (TransitionLane)
root.pendingLanes = SyncLane | TransitionLane1;

// performWorkOnRoot picks SyncLane first (highest)
renderRootSync(root, SyncLane);

// After SyncLane done:
root.pendingLanes &= ~SyncLane;  // SyncLane done
// Now: pendingLanes = TransitionLane1
// Schedule render at TransitionLane priority (yield to browser)
```

**`isPending` derivation:**

```typescript
function useTransition() {
  const [isPending, setPending] = useState(false);

  const startTransition = useCallback((scope) => {
    setPending(true);  // SyncLane (urgent — show spinner)

    // Assign TransitionLane for inner setState
    React_startTransition(() => {
      scope();
      setPending(false);  // TransitionLane (when transition done)
    });
  }, []);

  return [isPending, startTransition];
}
```

**Multiple transitions — alternating lanes:**

```typescript
function pickArbitraryLane(lanes: Lanes): Lane {
  return getHighestPriorityLane(lanes);
}

// 16 transition lanes available
const TransitionLane1 = 0b00000000000000000000000001000000;
const TransitionLane2 = 0b00000000000000000000000010000000;
// ...

// Multiple useTransition'lar — alohida lanes (parallel transitions)
```

**Interruption mechanics:**

```
TransitionLane render running:
  Fiber A processed
  Fiber B in progress
  ↓
SyncLane update arrives (user input)
  ↓
shouldYield() returns true
  ↓
Save state, yield to browser
  ↓
Sync render of input update
  ↓
After sync done, resume TransitionLane
```

**`startTransition` outside `useTransition`:**

```tsx
import { startTransition } from "react";

function handleClick() {
  startTransition(() => {
    setExpensiveState(newValue);  // TransitionLane
  });
}
// No isPending boolean — manual loading state
```

**`useDeferredValue` — similar mechanism:**

```tsx
function Search() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);  // ← TransitionLane internal

  const filtered = items.filter(i => i.name.includes(deferredQuery));
  return <List items={filtered} />;
}
```

`useDeferredValue` — auto-creates transition for value change tracking.

</details>

### Edge Cases

- **`startTransition` with sync API**: `startTransition(() => fetch(...))` — fetch sync emas. Wrap state update only.
- **Nested transitions**: Inner `startTransition` reuses outer transition lane.
- **`startTransition` in event handler**: Outer setState SyncLane, inner setState TransitionLane.

### Follow-up savollar

- "When to use useTransition vs useDeferredValue?" — `useTransition` — control state directly. `useDeferredValue` — derive deferred from existing value.
- "Performance overhead of transitions?" — Minimal. Lane bookkeeping ~5%, but user-perceived smoothness huge gain.
- "Compiler auto-applies transitions?" — No. Manual `useTransition` for now.

</details>

---

## QISM D: Hydration

<a id="qism-d"></a>

### 35. SSR + Hydration nima va nima uchun kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**SSR (Server-Side Rendering)** — React app server'da run qilinadi (Node.js/Edge), HTML string yoki stream chiqaradi, browser'ga yuboriladi. Browser HTML'ni darhol ko'rsatadi (fast First Paint, SEO). **Hydration** — browser'da React Fiber tree'ni qurib, mavjud DOM bilan **bog'laydi** (event listeners attach, state init). User interaktivlik uchun shu kerak.

### To'liq tushuntirish

**SSR pipeline:**

```
1. User → GET /page
2. Server: render <App /> with React DOM Server
   → renderToString() / renderToReadableStream()
   → produces HTML string
3. HTTP response: HTML
4. Browser: receives HTML, paints (fast First Contentful Paint)
5. Browser: downloads JS bundle
6. Browser: hydrates — runs React, attaches to DOM
7. App becomes interactive (Time to Interactive)
```

**CSR (Client-Side) vs SSR:**

| Aspekt | CSR | SSR |
|--------|-----|-----|
| Initial paint | Slow (JS download + parse + render) | Fast (HTML ready) |
| TTI | Slow | Slower (JS + hydrate) |
| SEO | Poor (empty HTML) | Good (rendered HTML) |
| Server cost | None | High (render per request) |
| Caching | CDN static | More complex |

**Hydration mexanizmi:**

- React traverses DOM va Fiber tree
- Element-by-element, position-by-position
- Server HTML va client expected match assumption
- Event listeners attach, state init, refs attach
- DOM **rebuild qilinmaydi** — preserve qilinadi

### Kod misoli

```tsx
// Server (Node.js)
import { renderToString } from "react-dom/server";
import { App } from "./App";

const html = renderToString(<App />);
// → "<div><h1>Hello</h1><button>Click</button></div>"

// Send full HTML
res.send(`
  <!DOCTYPE html>
  <html>
    <head><title>App</title></head>
    <body>
      <div id="root">${html}</div>
      <script src="/bundle.js"></script>
    </body>
  </html>
`);
```

```tsx
// Client (browser)
import { hydrateRoot } from "react-dom/client";
import { App } from "./App";

// Browser: HTML already rendered
// Hydrate: attach React to existing DOM
hydrateRoot(document.getElementById("root")!, <App />);

// Now interactive: clicks work, state updates DOM
```

**Hydration mismatch — common issue:**

```tsx
function App() {
  // ❌ Date.now() — different between server and client
  return <p>Now: {new Date().toLocaleString()}</p>;
}

// Server (render time): "Now: 2026-05-09 10:00:00"
// Client (hydrate time): "Now: 2026-05-09 10:00:01"
// React: "Hydration failed because the initial UI does not match..."
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`renderToString` vs `renderToReadableStream`:**

```typescript
// Static — full HTML at once
const html = renderToString(<App />);
res.send(html);

// Streaming — chunk by chunk
const stream = await renderToReadableStream(<App />);
res.send(stream);  // Web Streams API
// Each chunk sent as ready, progressive HTML
```

R19 default: `renderToReadableStream` (streaming).

**Hydration walks DOM and Fiber tree:**

```typescript
function hydrateRoot(container: Element, initialChildren: ReactNode) {
  const root = createHydrationRoot(container);

  // Walk DOM, create Fiber tree, match positions
  performWorkOnRoot(root);

  return root;
}

// During render — special hydration mode:
function tryToClaimNextHydratableInstance(fiber: Fiber) {
  const nextDOMNode = nextHydratableInstance;  // current DOM position

  if (nextDOMNode && fiber.type === nextDOMNode.tagName.toLowerCase()) {
    // Match — claim this DOM node
    fiber.stateNode = nextDOMNode;
    nextHydratableInstance = nextDOMNode.firstChild;
  } else {
    // Mismatch
    handleMismatch(fiber, nextDOMNode);
  }
}
```

**Mismatch handling (R18+):**

```typescript
function handleMismatch(fiber: Fiber, domNode: Element | null) {
  // Console error in dev
  console.error("Hydration mismatch...");

  // Recovery: discard server HTML for this subtree, render client-only
  fiber.flags |= ForceClientRender;

  // Clear DOM, fresh render
  removeServerRenderedNodes(domNode);
}
```

R18: granular mismatch recovery. R17: full client re-render (entire tree).

**Why hydrate, why not just CSR?**

CSR cons:
- Empty HTML — bad SEO (Google partially handles JS, others don't)
- Slow First Paint — must wait JS download
- Janky — flash of empty content (FOEC)

SSR pros:
- Pre-rendered HTML — fast FCP
- SEO indexable
- Progressive loading

Hydration — cost: extra JS work to "claim" existing DOM. Worth it for FCP + SEO.

**Hydration cost:**

```
Pure CSR:
  Download HTML (small, ~3kb)
  Download JS (200kb)
  Parse JS
  Execute App
  Render to DOM
  → User sees content (slow)

SSR + Hydration:
  Download HTML (large, ~30kb with content)
  Browser paints (fast — user sees content!)
  Download JS (200kb)
  Parse JS
  Execute App (hydrate mode)
  Walk DOM, match Fibers, attach listeners
  → User can interact

Hydration time — app size'ga bog'liq (DOM walk + Fiber tree create + listener attach)
```

**`renderToString` vs `renderToStaticMarkup`:**

```typescript
// Includes data-reactroot, hydration-required attributes
renderToString(<App />);
// → "<div data-reactroot=''>...</div>"

// Static — no React internals (used for static HTML, no hydration)
renderToStaticMarkup(<App />);
// → "<div>...</div>"
```

`renderToStaticMarkup` — for purely static pages (email templates, marketing content).

**Hydration order:**

Hydration depth-first, top-down:
1. Root fiber created
2. App component renders (client)
3. Walk to first child element
4. Match with first DOM child
5. Continue recursively

**Streaming + Hydration:**

```tsx
// Server (R18+ streaming)
import { renderToReadableStream } from "react-dom/server";

async function handler(req) {
  const stream = await renderToReadableStream(<App />, {
    bootstrapModules: ["/main.js"],
  });
  return new Response(stream, { headers: { "Content-Type": "text/html" } });
}

// Browser receives:
// <html>...<div id="root">  ← starts paint
// <ChunkedComponent /> rendered, chunk sent
// More chunks...
// </div></html>

// Client hydration starts as soon as bootstrap script runs
// Hydrates progressively, even if HTML still streaming
```

**Selective hydration (R18):**

```tsx
<Suspense fallback={<Spinner />}>
  <SlowDataView />
</Suspense>
```

Suspense boundaries — independent hydration units. Each boundary hydrates when its data ready.

**`hydrateRoot` API:**

```typescript
hydrateRoot(container, element, options?: {
  identifierPrefix?: string;
  onCaughtError?: (err) => void;
  onUncaughtError?: (err) => void;
  onRecoverableError?: (err) => void;
});
```

Options for fine-grained error handling.

**SSR frameworks:**

- **Next.js**: App Router (RSC), Pages Router (SSR/SSG)
- **Remix**: Loaders, actions, SSR-first
- **Astro**: SSG default, islands of interactivity
- **Gatsby**: SSG with React (legacy)
- **TanStack Start**: SSR + RSC (in development)

**Vanilla SSR setup:**

```tsx
// server.tsx
import express from "express";
import { renderToString } from "react-dom/server";
import { App } from "./App";

const app = express();
app.get("/", (req, res) => {
  const html = renderToString(<App />);
  res.send(`
    <!DOCTYPE html>
    <html>
      <head><title>App</title></head>
      <body>
        <div id="root">${html}</div>
        <script src="/main.js"></script>
      </body>
    </html>
  `);
});
app.listen(3000);
```

```tsx
// client.tsx
import { hydrateRoot } from "react-dom/client";
import { App } from "./App";

hydrateRoot(document.getElementById("root")!, <App />);
```

</details>

### Edge Cases

- **`useEffect` doesn't run during SSR**: Server doesn't run effects (no DOM). Effects fire on client after hydration.
- **`useLayoutEffect` warning on server**: Same — no DOM. Use `useEffect` or skip for SSR.
- **Browser-only APIs**: `window`, `localStorage` — not available on server. Guard with `typeof window !== "undefined"`.

### Follow-up savollar

- "SSR vs SSG?" — SSG (Static Site Generation) — render at build time, static HTML. SSR — render per request. SSG cheaper, SSR dynamic.
- "Hydration vs Resumability?" — Qwik resumability: serialize app state in HTML, no full re-execute on client. React doesn't have native resumability (but RSC moves towards this).
- "Streaming SSR + RSC?" — RSC + streaming = optimal. Server components stream HTML, client components hydrate progressively.

</details>

---

### 36. `hydrateRoot` vs `createRoot` — qachon qaysi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`createRoot`** — DOM **bo'sh** kontainerga client-side rendering (CSR) — React dan boshqa hech nima yo'q. **`hydrateRoot`** — server-rendered HTML mavjud kontainerga, React **fiber tree quradi** va mavjud DOM **bilan bog'laydi** (event listeners, state). `hydrateRoot` SSR/SSG context'da ishlatiladi, `createRoot` SPA-only context'da.

### To'liq tushuntirish

**`createRoot` use case:**

```tsx
import { createRoot } from "react-dom/client";

// HTML: <div id="root"></div>  ← empty
const root = createRoot(document.getElementById("root")!);
root.render(<App />);
// React creates entire DOM from scratch
```

**`hydrateRoot` use case:**

```tsx
import { hydrateRoot } from "react-dom/client";

// HTML: <div id="root">...server-rendered content...</div>
const root = hydrateRoot(document.getElementById("root")!, <App />);
// React traverses existing DOM, attaches Fiber tree, doesn't recreate DOM
```

**Asosiy farqlar:**

| Aspekt | `createRoot` | `hydrateRoot` |
|--------|--------------|----------------|
| Initial DOM | Bo'sh kontainer | Server-rendered content |
| Build phase | Create elements | Match elements |
| Mismatch handling | N/A | Warning + recovery |
| Initial render speed | Full render | Walk + attach |
| Bundle pre-requisite | Just JS | JS + server HTML |
| Dev errors | Common React errors | Hydration mismatch errors |

### Kod misoli

```tsx
// 1. createRoot — pure SPA
import { createRoot } from "react-dom/client";

const container = document.getElementById("root");
if (!container) throw new Error("Container missing");

const root = createRoot(container);
root.render(<App />);

// Re-render
root.render(<App key="2" />);

// Cleanup
root.unmount();
```

```tsx
// 2. hydrateRoot — SSR'd app
import { hydrateRoot } from "react-dom/client";

const container = document.getElementById("root");
if (!container) throw new Error("Container missing");

const root = hydrateRoot(container, <App />, {
  onRecoverableError: (error, errorInfo) => {
    console.warn("Hydration recovered:", error, errorInfo);
  },
});

// Re-render — same as createRoot
root.render(<App key="updated" />);
```

```tsx
// 3. Conditional — detect SSR vs CSR
function mount() {
  const container = document.getElementById("root")!;

  // Detect server-rendered (e.g., look for placeholder or attribute)
  const wasSSR = container.hasAttribute("data-ssr");

  if (wasSSR) {
    hydrateRoot(container, <App />);
  } else {
    createRoot(container).render(<App />);
  }
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`hydrateRoot` internals:**

```typescript
function hydrateRoot(
  container: Element,
  initialChildren: ReactNode,
  options?: HydrateRootOptions
): Root {
  const root = createHydrationRoot(container, options);

  // Schedule render with hydration mode
  updateContainer(initialChildren, root);

  return {
    render: (children) => updateContainer(children, root),
    unmount: () => updateContainer(null, root),
  };
}

function createHydrationRoot(container: Element, options) {
  const fiberRoot = createFiberRoot(container, ConcurrentRoot, /* hydrate */ true, options);

  // Mark all DOM children as hydratable (need to be claimed)
  const initialState = createInitialDocumentState(container);
  fiberRoot.current.memoizedState = initialState;

  return fiberRoot;
}
```

**Hydration mode flag:**

```typescript
type FiberRoot = {
  hydrate: boolean;
  // ...
};

// During render:
function updateHostComponent(fiber) {
  if (root.hydrate) {
    // Claim existing DOM node
    tryToClaimNextHydratableInstance(fiber);
  } else {
    // Create new DOM node
    createInstance(fiber);
  }
}
```

**Claim algorithm:**

```typescript
let nextHydratableInstance: Element | null;

function tryToClaimNextHydratableInstance(fiber: Fiber) {
  const node = nextHydratableInstance;

  if (node && canHydrateInstance(node, fiber.type)) {
    // Match — assign DOM node to fiber
    fiber.stateNode = node;
    nextHydratableInstance = getNextHydratableSibling(node);
    return;
  }

  // Mismatch — error / recovery
  fiber.flags |= Hydrating;
  // ... handle mismatch
}

function canHydrateInstance(node: Element, type: string): boolean {
  return (
    node.nodeType === ELEMENT_NODE &&
    node.tagName.toLowerCase() === type.toLowerCase()
  );
}
```

**`canHydrateInstance` checks:**

- Node type (element vs text)
- Tag name match (e.g., 'div' vs 'span')
- Doesn't check attributes/props (those reconcile after match)

**Mismatch types:**

```typescript
// 1. Tag mismatch
// Server: <div>...</div>
// Client: <span>...</span>
// → mismatch, force client-side render of subtree

// 2. Text mismatch
// Server: "Hello"
// Client: "Hi"
// → warning, use client value (no force re-render)

// 3. Attribute mismatch
// Server: <div className="dark">
// Client: <div className="light">
// → no warning by default, client wins (with R19 enhancements)

// 4. Extra/missing children
// Server: <div><p /><p /></div>
// Client: <div><p /></div>
// → mismatch, force re-render
```

**`onRecoverableError` callback:**

```typescript
const root = hydrateRoot(container, <App />, {
  onRecoverableError: (error: Error, errorInfo: { componentStack: string }) => {
    // Log to monitoring
    Sentry.captureException(error, { extra: errorInfo });
  },
});
```

R18+ — recoverable errors during hydration: subtree fell back to client render, but app keeps working. R19 — more detailed error info.

**Re-render after hydrate — same as createRoot:**

```tsx
const root = hydrateRoot(container, <InitialApp />);

// Later:
root.render(<UpdatedApp />);
// → Normal render path, no hydration logic
// → Just like createRoot.render()
```

**Hydration limitations:**

```tsx
// ❌ Server-only data not in DOM
function App() {
  const isClient = typeof window !== "undefined";
  return <p>{isClient ? "client" : "server"}</p>;
}
// Server HTML: "server"
// Client hydrate: "client"
// Mismatch!

// ✅ useEffect for client-only
function App() {
  const [isClient, setIsClient] = useState(false);
  useEffect(() => setIsClient(true), []);
  return <p>{isClient ? "client" : "server"}</p>;
}
// Server: "server", Client initial: "server" (matches!), then useEffect → "client"
```

**Multiple roots — both APIs work:**

```tsx
const widgetRoot = createRoot(document.getElementById("widget"));  // CSR
widgetRoot.render(<Widget />);

const appRoot = hydrateRoot(document.getElementById("app"), <App />);  // SSR
// Both can coexist
```

**Performance:**

| Operation | createRoot | hydrateRoot |
|-----------|-----------|-------------|
| Time to interactive | JS download + render | JS download + walk DOM |
| First Paint | After JS render | Server HTML (fast) |
| Memory | Same | Same |
| CPU during hydrate | Render new | Match existing |

`hydrateRoot` full re-render'dan tezroq — DOM allocation yo'q, faqat existing node'larni "claim" qiladi.

**Streaming hydration:**

```tsx
// Streaming SSR + Suspense
<Suspense fallback={<Spinner />}>
  <Comments />
</Suspense>

// Server: streams Spinner first, then Comments when ready
// Client: hydrates Spinner immediately, replaces with Comments when streamed
```

`hydrateRoot` works with streaming — selective hydration per Suspense boundary.

**`unmount`:**

```typescript
const root = hydrateRoot(container, <App />);

// Later:
root.unmount();
// Tears down React tree, cleans up effects, removes DOM (kept after unmount? — yes, but handlers detached)
```

**`onCaughtError` and `onUncaughtError` (R19):**

```tsx
const root = hydrateRoot(container, <App />, {
  onCaughtError: (error, errorInfo) => {
    // Caught by error boundary
    console.error("Caught:", error);
  },
  onUncaughtError: (error, errorInfo) => {
    // Not caught — error escaped to root
    Sentry.captureException(error);
  },
  onRecoverableError: (error, errorInfo) => {
    // Hydration mismatch, recovered
    console.warn("Recovered:", error);
  },
});
```

R19 root-level error callbacks.

</details>

### Edge Cases

- **`hydrateRoot` on empty container**: Mismatch every node — fully re-renders client-side.
- **Container with non-React content**: React tries to claim, may corrupt non-React DOM.
- **`createRoot` on SSR'd container**: DOM cleared, re-rendered (no hydration). Slow + flash.

### Follow-up savollar

- "Why two APIs not one?" — Behavior fundamentally different. Single API would have ambiguous semantics.
- "Hot reload during dev?" — Frameworks (Next.js, Vite) handle. Use `hydrateRoot` initially, then state-preserving HMR.
- "Hydration error in production — how to debug?" — `onRecoverableError` log + Sentry. Compare server HTML vs client expected.

</details>

---

### 37. Hydration mismatch — sabablari va fix [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Hydration mismatch** — server HTML va client expected DOM bir-biriga mos kelmaydi. Sabablar: (1) **`Date.now()`, `Math.random()`** — render time'da farqli, (2) **Browser-only APIs** (`window`, `localStorage`) server'da yo'q, (3) **Conditional rendering env-based** (`typeof window !==`), (4) **User locale/timezone** server vs client farqli, (5) **Browser extensions inject HTML**, (6) **Server caching stale**, (7) **Whitespace/newline differences**.

### To'liq tushuntirish

**Common mismatch causes:**

```tsx
// 1. Date/time
function App() {
  return <p>{new Date().toLocaleString()}</p>;
  // Server: "10:00:00"
  // Client (1s later): "10:00:01"
  // Mismatch
}

// 2. Random
function App() {
  return <p>ID: {Math.random()}</p>;
  // Server: "ID: 0.123"
  // Client: "ID: 0.456"
  // Mismatch
}

// 3. Window/document
function App() {
  return <p>{window.innerWidth}px</p>;
  // Server: ReferenceError (window undefined)
  // Won't even render
}

// 4. Locale
function App() {
  return <p>{(1234.5).toLocaleString()}</p>;
  // Server (en-US): "1,234.5"
  // Client (de-DE): "1.234,5"
  // Mismatch
}

// 5. Conditional rendering
function App() {
  const isMobile = typeof window !== "undefined" && window.innerWidth < 600;
  return <div>{isMobile ? <Mobile /> : <Desktop />}</div>;
  // Server: Desktop (no window)
  // Client: Mobile (window.innerWidth = 400)
  // Mismatch
}
```

**Fix patterns:**

```tsx
// ✅ Pattern 1: useEffect for client-only
function ClientTime() {
  const [time, setTime] = useState<string | null>(null);

  useEffect(() => {
    setTime(new Date().toLocaleString());
  }, []);

  return <p>{time ?? "Loading..."}</p>;
}
// Server: "Loading..."
// Client initial: "Loading..." (matches!)
// useEffect runs: setTime → re-render with actual time

// ✅ Pattern 2: useId for stable IDs
function App() {
  const id = useId();  // Same on server and client
  return <input id={id} />;
}

// ✅ Pattern 3: suppressHydrationWarning
function Time() {
  return (
    <p suppressHydrationWarning>
      {new Date().toLocaleString()}
    </p>
  );
}
// Mismatch suppressed (only for known cases like time)

// ✅ Pattern 4: Mounted state pattern
function ClientOnly({ children }: { children: React.ReactNode }) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);
  if (!mounted) return null;
  return <>{children}</>;
}

// Usage:
<ClientOnly>
  <BrowserOnlyComponent />
</ClientOnly>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Detection algorithm:**

```typescript
function tryToClaimNextHydratableInstance(fiber: Fiber) {
  const node = nextHydratableInstance;

  if (!canHydrateInstance(node, fiber.type, fiber.pendingProps)) {
    // Mismatch
    if (fiber.tag === HostText) {
      handleTextMismatch(fiber, node);
    } else {
      throwOnHydrationMismatch(fiber);
    }
  }

  // Match — claim
  fiber.stateNode = node;
}

function canHydrateInstance(node, type, props) {
  return (
    node.nodeType === ELEMENT_NODE &&
    node.tagName.toLowerCase() === type.toLowerCase()
  );
}
```

**Text mismatch handling:**

```typescript
function handleTextMismatch(fiber, node) {
  // Console error
  if (__DEV__) {
    console.error(
      "Text content did not match. Server: %s Client: %s",
      node.textContent,
      fiber.pendingProps
    );
  }

  // Update DOM with client value
  node.textContent = fiber.pendingProps;
  fiber.flags |= Update;
}
```

Text mismatch — soft (warning + update). Element mismatch — hard (force re-render).

**Element mismatch handling (R18):**

```typescript
function throwOnHydrationMismatch(fiber: Fiber) {
  if (__DEV__) {
    console.error(
      "Hydration failed because the initial UI does not match what was rendered on the server."
    );
  }

  // Force client-side render — abandon SSR'd DOM for this subtree
  fiber.flags |= ForceClientRender;

  throw new Error("Hydration failed because the server rendered HTML didn't match the client.");
}
```

The thrown error caught at root level — fall back to client-only render of subtree.

**`onRecoverableError`:**

```tsx
hydrateRoot(container, <App />, {
  onRecoverableError: (error: Error, errorInfo) => {
    if (error.message.includes("Hydration")) {
      // Logged: hydration mismatch caused subtree re-render
      logToMonitoring("hydration-mismatch", { error, errorInfo });
    }
  },
});
```

**`suppressHydrationWarning` attribute:**

```tsx
<div suppressHydrationWarning>
  Server time: {new Date().toLocaleString()}
</div>
```

Effect:
- Server renders: <div>Server time: 10:00:00</div>
- Client renders: <div>Server time: 10:00:01</div>
- React: mismatch detected
- `suppressHydrationWarning`: skip warning, use client value

⚠️ **Use sparingly** — masks real bugs. Only for known unstable values (time, random IDs not from `useId`).

**`useId` hook (R18):**

```tsx
function Form() {
  const id = useId();  // Stable: server gives same as client
  return (
    <>
      <label htmlFor={id}>Email</label>
      <input id={id} type="email" />
    </>
  );
}

// Server: id = ":r0:"
// Client: id = ":r0:"  (matches!)
```

`useId` — based on fiber tree path, stable across server/client. Prevents random ID mismatches.

**Browser extensions issue:**

```html
<!-- Browser extensions (Grammarly, ad blockers) inject elements -->
<div id="root">
  <div data-grammarly-ignore="">  ← injected by extension
    <App />
  </div>
</div>

<!-- Hydration: React expects <App>, finds extension's <div> -->
<!-- Mismatch! -->
```

**Mitigation:**
- `<body>` level injection less common
- Test in clean browser
- Some extensions skip pages with React signals

**Cache-related mismatch:**

```typescript
// Stale server cache — old HTML
// Client deployed new bundle
// Hydration: HTML structure from old version, JS from new version
// Mismatch!
```

**Fix:** Cache busting on bundle deploy, or use service worker for atomic updates.

**Whitespace mismatch:**

```jsx
// Server JSX
return <p>Hello {name}</p>;
// HTML: <p>Hello Ali</p>

// Client JSX (formatted differently)
return (
  <p>
    Hello {name}
  </p>
);
// HTML: <p>Hello Ali</p> ← whitespace removed
```

JSX usually consistent. Manual HTML — beware whitespace handling.

**SSG with dynamic content:**

```tsx
// SSG (Static Site Generation) at build time
function App() {
  return <p>Built at: {new Date().toISOString()}</p>;  // ❌ build time
}

// Build: "Built at: 2026-05-09T10:00:00"
// Hydrate: matches HTML (build time hardcoded)
// But conceptually wrong — value frozen at build
```

Use `useEffect` for runtime values.

**Server vs client DOM differences React allows:**

```tsx
// Server output: <input value="ali" />
// Hydration: React assigns DOM input.value = "ali"
// User types "abc"
// React: noticed user input, allows DOM mutation (no warning)
```

React tolerates DOM mutation post-hydration (uncontrolled inputs).

**`use client` directive — RSC context:**

```tsx
// app/page.tsx (Server Component)
import { ClientWidget } from "./ClientWidget";

export default async function Page() {
  const data = await fetchData();
  return (
    <div>
      <h1>{data.title}</h1>
      <ClientWidget initialData={data} />
    </div>
  );
}

// ClientWidget.tsx
"use client";

export function ClientWidget({ initialData }) {
  const [data, setData] = useState(initialData);
  // ... interactive logic
}
```

Server-rendered HTML on initial load, client takes over via hydration. RSC manages boundary.

**Common mistakes:**

```tsx
// ❌ Reading window in render
const isClient = typeof window !== "undefined";
return <p>{isClient ? "client" : "server"}</p>;

// ✅ State-based
const [isClient, setIsClient] = useState(false);
useEffect(() => setIsClient(true), []);
return <p>{isClient ? "client" : "server"}</p>;
```

**Detection in production:**

```tsx
hydrateRoot(container, <App />, {
  onRecoverableError: (error, info) => {
    // Send to error tracking
    Sentry.captureException(error, {
      tags: { type: "hydration" },
      extra: { componentStack: info.componentStack },
    });
  },
});
```

</details>

### Edge Cases

- **`<table>` whitespace**: Browser parsers add whitespace text nodes. `<table>{rows}</table>` — children might mismatch.
- **`<select value>`**: Special handling — server renders `<option selected>`, client uses controlled value.
- **Conditional Suspense fallback**: Different boundary on server vs client → mismatch.

### Follow-up savollar

- "Production mismatch frequency?" — Well-architected app'da kam. Common in: time-sensitive UI, browser extension targets, A/B testing variant'lari.
- "Worth fixing all mismatches?" — Yes. Each mismatch = subtree client re-render = perceived flash. UX degradation.
- "ESLint rules?" — `eslint-plugin-react-hooks` doesn't catch directly. Custom rules / framework-specific (Next.js).

</details>

---

### 38. Selective Hydration (R18) — Suspense bilan [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Selective Hydration** (R18) — `<Suspense>` boundaries — alohida hydration unit'lar. Server-rendered HTML butun page bilan keladi, lekin client har Suspense boundary'ni **mustaqil va asynchronously** hydrate qiladi. **User interaktivlik priority** — agar user click qilsa Suspense boundary'da (hali hydrate bo'lmagan), shu boundary **prioritized** — birinchi hydrate qilinadi.

### To'liq tushuntirish

**Pre-R18 hydration:**
- Top-down, all-or-nothing
- One slow component blocks entire page
- User clicks unresponsive area until full hydration

**R18 selective hydration:**
- Suspense boundaries = independent units
- Hydrate in any order
- User interaction prioritizes specific boundary
- React handles concurrent hydration

### Kod misoli

```tsx
// SSR render
async function ServerApp() {
  return (
    <html>
      <body>
        <Header />
        <Suspense fallback={<NavSkeleton />}>
          <Navigation />
        </Suspense>
        <Suspense fallback={<MainSkeleton />}>
          <MainContent />
        </Suspense>
        <Suspense fallback={<SidebarSkeleton />}>
          <Sidebar />  {/* slow data dependency */}
        </Suspense>
        <Footer />
      </body>
    </html>
  );
}
```

**Hydration timeline:**

```
T=0: HTML received with all sections rendered (or fallbacks for slow ones)
     User can see content immediately

T=100ms: bundle loaded, hydration starts
T=100-200ms: Header hydrated (no Suspense, fast)
T=200-400ms: Navigation hydrated (Suspense boundary)
T=400ms: User clicks Sidebar button (still not hydrated!)
        React: prioritize Sidebar hydration (out of order!)
T=400-600ms: Sidebar hydrated, click handler runs
T=600-800ms: MainContent hydrated
T=800-900ms: Footer hydrated
```

User clicks worked even before that section was hydrated — React queued the event and replayed after hydration.

<details>
<summary><strong>Deep Dive</strong></summary>

**Hydration boundary semantics:**

```tsx
<Suspense fallback={<Spinner />}>
  <SubTree />
</Suspense>
```

- **Server**: Renders SubTree if data ready, else fallback
- **Client**: Hydrates SubTree as separate unit
- **Independent**: Other Suspense boundaries don't block this one

**Selective hydration mechanism:**

```typescript
// Each Suspense boundary becomes a "hydration root"
function attemptHydrationFromEvent(event: SyntheticEvent) {
  const target = event.target;
  const boundaryFiber = findNearestSuspenseBoundary(target);

  if (boundaryFiber && boundaryFiber.flags & DehydratedFragment) {
    // Boundary not hydrated — bump priority
    scheduleUpdateOnFiber(boundaryFiber, SyncLane);
    // → Hydration runs immediately for this boundary
  }
}

document.addEventListener("click", attemptHydrationFromEvent, true);
```

**`DehydratedFragment` fiber tag:**

```typescript
// Internal tag — number value version-specific
// Suspense boundary in dehydrated state
const fiber = {
  tag: DehydratedFragment,  // internal enum
  // ...
};
```

When Suspense boundary not yet hydrated, React keeps DOM as-is and tracks dehydrated state.

**Event replay:**

User click on dehydrated boundary:
1. Event captured
2. React detects boundary not hydrated
3. Trigger hydration of that boundary (priority)
4. After hydration, replay event with proper handler

```typescript
// During hydration
function dispatchEvent(event) {
  if (boundaryNotHydrated) {
    queueEvent(event);
    triggerHydration(boundary);
    return;
  }

  // Normal dispatch
  fireEvent(event);
}

// After hydration
function onBoundaryHydrated() {
  for (const queued of queuedEvents) {
    fireEvent(queued);
  }
}
```

**Streaming SSR — out-of-order rendering:**

```tsx
// Server (R18 streaming)
async function App() {
  return (
    <>
      <Suspense fallback={<Spinner />}>
        <SlowProductList />  {/* depends on /api/products (1s) */}
      </Suspense>
      <Suspense fallback={<Spinner />}>
        <FastReviews />  {/* depends on /api/reviews (200ms) */}
      </Suspense>
    </>
  );
}
```

Streaming behavior:
1. T=0: Server starts rendering. Sends shell with both fallbacks.
2. T=200ms: FastReviews data arrives. Server inserts FastReviews HTML.
3. T=1000ms: SlowProductList data arrives. Server inserts.
4. T=1000ms: Stream closes.

Browser sees:
- T=0: Skeleton page with 2 spinners
- T=200ms: Spinner replaced with FastReviews
- T=1000ms: Spinner replaced with SlowProductList

Out-of-order — fast content first.

**Hydration interleaving with streaming:**

- React on client starts hydrating root immediately
- As Suspense boundary streamed in, that boundary hydrates
- User can interact with already-hydrated parts

**`useTransition` interaction with hydration:**

```tsx
function App() {
  const [tab, setTab] = useState("home");
  const [, startTransition] = useTransition();

  return (
    <>
      <button onClick={() => startTransition(() => setTab("about"))}>
        About
      </button>
      <Suspense fallback={<Spinner />}>
        {tab === "home" ? <Home /> : <About />}
      </Suspense>
    </>
  );
}
```

Hydration + transition: transition lower priority during hydration. Hydration completes first.

**Priority bumping for user interaction:**

```typescript
function handleClickOnDehydratedBoundary(event) {
  const boundaryFiber = findBoundary(event.target);
  if (boundaryFiber && isDehydrated(boundaryFiber)) {
    // Bump from default to user-blocking priority
    scheduleUpdateOnFiber(boundaryFiber, InputContinuousLane);
  }
}
```

User intent → priority boost.

**Limitations:**

```tsx
// Boundary too coarse — entire boundary hydrates as unit
<Suspense fallback={<Spinner />}>
  <HugeContent />  {/* 1000 components, slow hydration */}
</Suspense>
```

Solution: nested boundaries.

```tsx
<Suspense fallback={<Spinner />}>
  <Header />
  <Suspense fallback={<Spinner />}>
    <SlowContent />
  </Suspense>
</Suspense>
```

**SuspenseList (experimental — stable React paketda YO'Q):**

```tsx
// Faqat experimental/canary build'da mavjud
<SuspenseList revealOrder="forwards">
  <Suspense fallback={<S />}>
    <First />
  </Suspense>
  <Suspense fallback={<S />}>
    <Second />
  </Suspense>
</SuspenseList>
```

`revealOrder="forwards"` — Suspense reveals in order (avoid late items showing first). React 19 stable release'da hali yo'q — `react@experimental` paketida.

**RSC + selective hydration:**

```tsx
// Server Component (RSC)
async function Page() {
  return (
    <>
      <Header />
      <Suspense fallback={<Spinner />}>
        <ClientInteractive />  {/* "use client" */}
      </Suspense>
    </>
  );
}
```

Server renders Page synchronously. ClientInteractive (client component) hydrates separately.

**Performance metrics:**

| Metric | Pre-R18 | R18 Selective |
|--------|---------|---------------|
| First Paint | Same (server HTML) | Same |
| Time to Interactive (TTI) | Until full hydration | Per-boundary, faster |
| User-perceived latency on click | Wait for full hydration | Boundary-targeted, fast |

**`onRecoverableError` for selective:**

```tsx
hydrateRoot(container, <App />, {
  onRecoverableError: (error, info) => {
    if (error.message.includes("Hydration")) {
      // Boundary fell back to client render
      // App still works, just slower for that boundary
      logHydrationFailure(error, info);
    }
  },
});
```

</details>

### Edge Cases

- **Click on outer non-Suspense element**: No boundary to prioritize — top-down hydration order continues.
- **Nested Suspense boundaries**: Each independent. Inner boundary may hydrate before outer.
- **Suspense fallback during hydration**: Promise still pending → fallback shown until ready.

### Follow-up savollar

- "Hydration prioritization observable?" — Indirect via Profiler. Internal mechanism.
- "Does selective hydration help SEO?" — No, SEO from server HTML. SH improves UX (TTI).
- "Concurrent hydration in different parts?" — Possible — React schedules with lanes. Multiple boundaries can interleave.

</details>

---

### 39. Streaming SSR — progressive HTML [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Streaming SSR** (R18+) — server HTML'ni **progressive yo'l bilan** yuboradi (chunk-by-chunk), to'liq tugamasdan oldin. Web Streams API (`renderToReadableStream`) yoki Node Streams (`renderToPipeableStream`) ishlatadi. Suspense boundaries — **streaming chegaralari**: tez tayyor bo'lganlar darhol stream qilinadi, slow ones — fallback bilan, keyin replace.

### To'liq tushuntirish

**Streaming SSR pipeline:**

```
1. Request comes in
2. Server starts rendering
3. Shell ready (basic HTML structure) → flush to client
4. Client paints shell (fast TTFB)
5. Server continues rendering, completes Suspense boundary
6. Server flushes that boundary's HTML
7. Client receives, replaces fallback
8. Repeat for each boundary
9. Stream closes when all done
```

**APIs:**

| API | Environment | Stream type |
|-----|-------------|-------------|
| `renderToReadableStream` | Edge runtime, Cloudflare Workers, Deno | Web Streams |
| `renderToPipeableStream` | Node.js | Node Streams |
| `renderToString` | All | Sync, no stream |

### Kod misoli

```tsx
// Edge / Web Streams
import { renderToReadableStream } from "react-dom/server";

export async function handler(req: Request) {
  const stream = await renderToReadableStream(<App />, {
    bootstrapModules: ["/main.js"],
    onError(error) {
      console.error("SSR error:", error);
    },
  });

  // Wait for all content to be ready (no error during shell)
  await stream.allReady;

  return new Response(stream, {
    headers: { "Content-Type": "text/html" },
  });
}

// Or, for true streaming, don't await allReady:
return new Response(stream, ...);
```

```tsx
// Node.js / Node Streams
import { renderToPipeableStream } from "react-dom/server";
import express from "express";

const app = express();
app.get("/", (req, res) => {
  const { pipe } = renderToPipeableStream(<App />, {
    bootstrapModules: ["/main.js"],
    onShellReady() {
      // Shell ready — flush to client
      res.setHeader("Content-Type", "text/html");
      pipe(res);
    },
    onShellError(error) {
      res.status(500).send("Server error");
    },
    onError(error) {
      console.error("SSR error:", error);
    },
  });
});
```

```tsx
// Component with Suspense for streaming
async function App() {
  return (
    <html>
      <body>
        <Header />  {/* Fast */}
        <Suspense fallback={<Skeleton />}>
          <SlowProducts />  {/* await db */}
        </Suspense>
        <Suspense fallback={<Skeleton />}>
          <SlowReviews />  {/* await api */}
        </Suspense>
      </body>
    </html>
  );
}

async function SlowProducts() {
  const products = await db.product.findMany();
  return <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Stream chunks structure:**

```html
<!-- Chunk 1: Shell -->
<!DOCTYPE html>
<html>
<head><title>App</title></head>
<body>
<div id="root">
  <header>...</header>
  <!-- Suspense fallbacks (placeholders) -->
  <template id="P:1">
    <div class="skeleton"></div>
  </template>
  <template id="P:2">
    <div class="skeleton"></div>
  </template>
</div>
<!-- React script for hydration coordination -->
<script>$RC = function(b,c,e){... // boundary completion handler}</script>

<!-- Chunk 2: First boundary ready (200ms later) -->
<div hidden id="S:1">
  <ul>...</ul>
</div>
<script>$RC("B:1", "S:1")</script>  <!-- swap fallback with content -->

<!-- Chunk 3: Second boundary ready (1s later) -->
<div hidden id="S:2">
  <article>...</article>
</div>
<script>$RC("B:2", "S:2")</script>

</body>
</html>
```

**`$RC` — boundary completion script:**

```javascript
$RC = function(boundaryID, contentID) {
  const boundary = document.getElementById(boundaryID);
  const content = document.getElementById(contentID);
  if (boundary && content) {
    // Move content into boundary's place
    boundary.parentNode.replaceChild(content, boundary);
  }
};
```

When server flushes new boundary, includes inline `$RC(...)` call → browser executes → DOM updated.

**`bootstrapModules`:**

```typescript
renderToReadableStream(<App />, {
  bootstrapModules: ["/main.js"],
});
```

- Server includes `<script type="module" src="/main.js"></script>` in HTML
- Client downloads, runs hydration
- Bootstrap script can start hydrating before all chunks streamed

**`onShellReady` callback:**

```typescript
const { pipe } = renderToPipeableStream(<App />, {
  onShellReady() {
    // Shell HTML ready — basic structure, no slow content yet
    // Flush to client immediately
    res.setHeader("Content-Type", "text/html");
    pipe(res);  // start streaming
  },
});
```

Shell — shrill structure (HTML + head + body skeleton). Flush early for fast TTFB.

**`onAllReady` callback:**

```typescript
const { pipe } = renderToPipeableStream(<App />, {
  onAllReady() {
    // All Suspense boundaries resolved
    // Use this for crawlers / SEO (need full content)
    pipe(res);
  },
});
```

For SEO bots, wait until everything ready (no streaming).

**Conditional bot detection:**

```typescript
const userAgent = req.headers["user-agent"];
const isCrawler = /bot|googlebot/i.test(userAgent);

const { pipe } = renderToPipeableStream(<App />, {
  onShellReady() {
    if (!isCrawler) {
      // Stream for users
      pipe(res);
    }
  },
  onAllReady() {
    if (isCrawler) {
      // Wait for all for bots (SEO)
      pipe(res);
    }
  },
});
```

**Error handling during streaming:**

```typescript
const { pipe } = renderToPipeableStream(<App />, {
  onShellError(error) {
    // Shell render failed — send 500
    res.status(500).send("Server error");
  },
  onError(error, errorInfo) {
    // Inside Suspense boundary error
    // Don't crash whole stream — log
    console.error("SSR error:", error);
  },
});
```

If Suspense child throws during render, that boundary falls back to client (error info sent to client).

**Stream timing — performance:**

```
T=0: Request received
T=10ms: Shell rendered, flushed
        TTFB: 10ms ✅ (fast)
T=200ms: Browser parses shell, paints fallbacks
T=300ms: First boundary done, flushed
T=350ms: Browser receives, swaps fallback
T=1000ms: Second boundary done
T=1100ms: Stream closes
```

vs. non-streaming `renderToString`:
```
T=0: Request
T=1000ms: All content rendered
T=1010ms: Response sent (1010ms TTFB)
```

Streaming dramatically improves TTFB and TTI for slow pages.

**Hydration with streaming:**

```typescript
// Client
import { hydrateRoot } from "react-dom/client";
import { App } from "./App";

hydrateRoot(document.getElementById("root")!, <App />);

// Browser:
// 1. HTML chunk received → DOM updated incrementally
// 2. Bootstrap script runs (after first chunk?)
// 3. hydrateRoot starts hydration
// 4. Boundary not in DOM yet — DehydratedFragment fiber, awaits
// 5. Stream chunk arrives, boundary content inserted
// 6. Hydration of that boundary triggered automatically
```

**`renderToPipeableStream` vs `renderToReadableStream`:**

| API | Environment | Stream type |
|-----|-------------|-------------|
| Pipeable | Node.js | Node `Readable` stream |
| Readable | Edge, Deno, Cloudflare | Web `ReadableStream` |

```typescript
// Pipeable (Node)
const { pipe, abort } = renderToPipeableStream(<App />, options);
pipe(res);  // Express response

// Readable (Web)
const stream = await renderToReadableStream(<App />, options);
return new Response(stream);  // Fetch Response
```

**Backpressure handling:**

Streaming respects backpressure (client slow to consume → server slows down). Built into stream APIs.

**Caching streams:**

Tricky — streams are single-use. Need to buffer response or use cached `renderToString`. Strategies:
- Static page cache (HTML files)
- ISR (Incremental Static Regeneration) - Next.js
- Edge caching (Cloudflare)

**Server timing:**

```
Time to First Byte (TTFB): 10ms (shell)
First Contentful Paint (FCP): 50ms (shell rendered)
Largest Contentful Paint (LCP): 300-1000ms (per boundary)
Time to Interactive (TTI): per boundary, parallel
```

</details>

### Edge Cases

- **`useEffect` during streaming**: Server doesn't run effects. Effects fire on client after hydration.
- **Multiple Suspense in same parent**: Independent — each has own fallback, streams when ready.
- **Suspense in nested location**: Depth doesn't matter — each Suspense = boundary.

### Follow-up savollar

- "Streaming + RSC — interaction?" — RSC payload streams as components resolve. Combined with HTML streaming for full SSR.
- "Cache streaming response?" — Hard. Buffer stream first, cache buffer. Or pre-render to file.
- "Backpressure handling?" — Native stream APIs handle. React Scheduler doesn't override.

</details>

---

### 40. R19 hydration improvements [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**R19 hydration improvements**: (1) **Better error messages** — aniq ko'rsatadi qaysi DOM node mismatch, server vs client farqi, (2) **`onCaughtError`/`onUncaughtError`/`onRecoverableError` callbacks** — root level error handling, (3) **Hydration error diff** — server va client tree visualization, (4) **`<title>`, `<meta>` hoisting** — duplicates'ni avoid qiladi, (5) **Stylesheet integration** — Suspense bilan stylesheet load coordination.

### To'liq tushuntirish

**R18 mismatch error:**

```
Warning: Hydration failed because the initial UI does not match what was rendered on the server.
```

— vague, doesn't specify which component or what differs.

**R19 mismatch error:**

```
Warning: Hydration error
- Component: Header (App.tsx:23)
- Server rendered: <span>Server</span>
- Client tried to render: <span>Client</span>
- DOM context: <header>...
```

— precise diagnostics.

### Kod misoli

```tsx
// R19 root callbacks
import { hydrateRoot } from "react-dom/client";

const root = hydrateRoot(container, <App />, {
  // Caught by error boundary
  onCaughtError: (error, errorInfo) => {
    console.error("Caught:", error);
    Sentry.captureException(error, {
      tags: { type: "react-caught" },
      extra: errorInfo,
    });
  },

  // Not caught — escaped to root
  onUncaughtError: (error, errorInfo) => {
    console.error("Uncaught:", error);
    Sentry.captureException(error, {
      tags: { type: "react-uncaught" },
      extra: errorInfo,
    });
  },

  // Hydration mismatch, recovered (subtree client-rendered)
  onRecoverableError: (error, errorInfo) => {
    console.warn("Recovered:", error);
    Sentry.captureMessage("Hydration recovered", {
      level: "warning",
      extra: { error, errorInfo },
    });
  },
});
```

```tsx
// R19 Document metadata hoisting (no react-helmet)
function ProductPage({ product }: { product: Product }) {
  return (
    <article>
      {/* These auto-hoist to <head> */}
      <title>{product.name} - Online Store</title>
      <meta name="description" content={product.description} />
      <meta property="og:image" content={product.image} />
      <link rel="canonical" href={`https://store.com/products/${product.id}`} />

      {/* Body content */}
      <h1>{product.name}</h1>
      <p>{product.description}</p>
    </article>
  );
}

// SSR output:
// <html>
//   <head>
//     <title>Phone - Online Store</title>
//     <meta name="description" content="..." />
//     <meta property="og:image" content="..." />
//     <link rel="canonical" href="..." />
//   </head>
//   <body>
//     <article>
//       <h1>Phone</h1>
//       <p>...</p>
//     </article>
//   </body>
// </html>
// Hydration: <head> tags identified, deduplicated, no mismatch
```

```tsx
// R19 Stylesheet support with Suspense
function ProductPage() {
  return (
    <Suspense fallback={<Skeleton />}>
      <link rel="stylesheet" href="/styles/product.css" precedence="default" />
      <ProductDetails />
    </Suspense>
  );
}

// React waits for stylesheet to load before showing content
// Avoids FOUC (Flash of Unstyled Content)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**R19 mismatch error generation:**

```typescript
function logHydrationMismatch(fiber: Fiber, serverNode: Element, expectedClient: any) {
  const componentStack = getComponentStack(fiber);
  const fileName = fiber._debugSource?.fileName;
  const lineNumber = fiber._debugSource?.lineNumber;

  console.error(`Hydration error
- Component: ${getComponentName(fiber)} (${fileName}:${lineNumber})
- Server rendered: ${serverNode.outerHTML}
- Client tried to render: ${describeExpected(expectedClient)}
- DOM context: ${describeContext(serverNode)}
- Component stack:
${componentStack}`);
}
```

DevTools also shows the mismatch in Components panel with diff.

**`onRecoverableError` enhanced detail:**

```typescript
type ErrorInfo = {
  componentStack: string;
  digest?: string;  // R19 — error digest for matching server logs
};

hydrateRoot(container, <App />, {
  onRecoverableError: (error: Error, info: ErrorInfo) => {
    if (info.digest) {
      // Match with server log entry
      logToMonitoring({
        type: "hydration",
        clientError: error,
        serverDigest: info.digest,
      });
    }
  },
});
```

`digest` — short hash, helps correlate client error with server log.

**Document metadata hoisting:**

```tsx
// Pre-R19: react-helmet required
import { Helmet } from "react-helmet";

function Page() {
  return (
    <>
      <Helmet>
        <title>Page</title>
        <meta name="description" content="..." />
      </Helmet>
      <div>...</div>
    </>
  );
}

// R19: native, no library
function Page() {
  return (
    <>
      <title>Page</title>
      <meta name="description" content="..." />
      <div>...</div>
    </>
  );
}
// React hoists <title>, <meta>, <link> to <head>
```

**Hoisting algorithm:**

```typescript
function commitInsertion(fiber, parent, beforeChild) {
  if (isHoistableElement(fiber)) {
    // Move to <head>
    document.head.appendChild(fiber.stateNode);
  } else {
    // Normal insertion
    parent.insertBefore(fiber.stateNode, beforeChild);
  }
}

function isHoistableElement(fiber): boolean {
  if (fiber.type === "title") return true;
  if (fiber.type === "meta") return true;
  if (fiber.type === "link" && fiber.props.rel) {
    // <link rel="stylesheet">, <link rel="canonical">
    return true;
  }
  return false;
}
```

**Deduplication:**

```tsx
// Multiple components rendering meta tags
function App() {
  return (
    <>
      <PageMeta title="App" />
      <ProductMeta />  {/* might have its own title */}
    </>
  );
}

function PageMeta() {
  return <title>App</title>;
}

function ProductMeta() {
  return <title>Product Page</title>;
}

// Result: only one <title> in <head>
// React picks last-rendered (later = more specific)
```

**Stylesheet precedence:**

```tsx
<link rel="stylesheet" href="/base.css" precedence="default" />
<link rel="stylesheet" href="/component.css" precedence="default" />
<link rel="stylesheet" href="/override.css" precedence="high" />
```

`precedence` — order in `<head>`. Higher precedence later (overrides earlier rules).

**Suspense + stylesheet:**

```tsx
<Suspense fallback={<Spinner />}>
  <link rel="stylesheet" href="/feature.css" precedence="default" />
  <FeatureComponent />
</Suspense>
```

React waits for stylesheet load before showing content. Prevents FOUC.

**Async scripts:**

```tsx
function Page() {
  return (
    <>
      <script async src="/analytics.js" />
      <main>...</main>
    </>
  );
}
// Script hoisted to <head>, deduplicated
```

**Resource preloading APIs (R19):**

```tsx
import { preload, preinit, prefetchDNS, preconnect } from "react-dom";

function App() {
  // Preload — fetch resource early
  preload("/data.json", { as: "fetch" });

  // Preinit — fetch + execute (script/style)
  preinit("/critical.css", { as: "style", precedence: "default" });

  // DNS resolve early
  prefetchDNS("https://api.example.com");

  // Open connection (DNS + TCP + TLS)
  preconnect("https://api.example.com");

  return <Page />;
}
```

These hooks generate `<link rel="preload">`, `<link rel="dns-prefetch">`, etc. in `<head>`.

**`onCaughtError`/`onUncaughtError`/`onRecoverableError` distinction:**

```typescript
// Caught: error boundary handled it
class Boundary extends Component {
  componentDidCatch(error) {
    // Boundary catches → onCaughtError fires
  }
}

// Uncaught: no boundary, error escaped
function Component() {
  throw new Error("oops");  // → onUncaughtError
}

// Recoverable: subtle issues, app still works
// Hydration mismatch → recovers via client render → onRecoverableError
```

**Error boundary integration:**

```tsx
class ErrorBoundary extends React.Component {
  state = { error: null };

  static getDerivedStateFromError(error) {
    return { error };
  }

  render() {
    if (this.state.error) return <ErrorView />;
    return this.props.children;
  }
}

const root = hydrateRoot(container, <App />, {
  onCaughtError: (error) => {
    // ErrorBoundary caught — log
    console.error("Caught:", error);
  },
  onUncaughtError: (error) => {
    // No boundary — app probably broken
    console.error("Uncaught:", error);
  },
});
```

**`useId` improvements (R18+):**

```tsx
function Form() {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>Email</label>
      <input id={id} type="email" />
    </>
  );
}
```

R18 introduced. R19: refined for streaming and selective hydration.

**Error digest matching:**

```typescript
// Server logs:
console.error("Hydration error", { digest: "abc123", details: "..." });

// Client error:
hydrateRoot(container, <App />, {
  onRecoverableError: (error, info) => {
    // info.digest === "abc123"
    // Match with server log to correlate
  },
});
```

**Performance impact (R19 vs R18):**

- Mismatch error generation: faqat dev mode'da diagnostics, runtime overhead minimal
- Document metadata hoisting: negligible runtime cost
- Resource preloading: load time'ni yaxshilaydi (proactive prefetch)
- Stylesheet Suspense: UX yaxshilaydi (FOUC oldini oladi)

</details>

### Edge Cases

- **`<title>` in fallback**: Both shown briefly during hydration. Eventually deduplicated.
- **Stylesheet precedence conflict**: Same precedence — render order wins.
- **Preload + actual resource missing**: Browser warning (preload but no use).

### Follow-up savollar

- "react-helmet still works in R19?" — Yes (custom hooks for compat). But native preferred.
- "Why hoist title?" — `<title>` belongs in `<head>` per HTML spec. JSX in `<body>` requires hoisting.
- "Stylesheet in Suspense — synchronous wait?" — Yes, blocks until stylesheet load. UX trade-off vs FOUC.

</details>

---

### 41. `suppressHydrationWarning` — qachon va xavfli paytlar [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`suppressHydrationWarning={true}` — single element'ga qo'yiladigan opt-out flag. React shu element ichida hydration mismatch bo'lsa **warning chiqarmaydi** va server HTML'ni saqlaydi (client value ignored). Use case: **sana/vaqt** (server vs client farqli), **random ID** (legacy code). Xavf: real bug'larni yashiradi (data mismatch). Faqat **bilib turgan** mismatch'larga.

### Kod misoli

```tsx
function CurrentTime() {
  return (
    <p suppressHydrationWarning>
      Server time: {new Date().toLocaleString()}
    </p>
  );
}
// Server: "Server time: 2026-01-01 12:00:00"
// Client: "Server time: 2026-01-01 12:00:05" (5s delay)
// suppressHydrationWarning — no warning, server value displayed
// (Client value lost — text static after hydration)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Behavior:**

- React skips text content comparison
- Server-rendered text remains in DOM
- Client value (from new render) ignored
- ⚠️ Only first text child — nested elements still compared

**Limitations:**

```tsx
// ❌ Doesn't suppress nested mismatch
<div suppressHydrationWarning>
  <ChildComponent />  {/* If ChildComponent has random ID — still warns */}
</div>

// ✅ Suppresses direct text only
<p suppressHydrationWarning>{Date.now()}</p>  {/* OK */}
```

**Better pattern — `useEffect` for client-only:**

```tsx
function CurrentTime() {
  const [time, setTime] = useState<string | null>(null);

  useEffect(() => {
    setTime(new Date().toLocaleString());
  }, []);

  return <p>Time: {time ?? "Loading..."}</p>;
}
// Server: <p>Time: Loading...</p>
// Client hydrate: <p>Time: Loading...</p>  (match!)
// useEffect: setTime → re-render with current time
// No warning, dynamic value
```

**`useSyncExternalStore` SSR pattern:**

```tsx
function useTime() {
  return useSyncExternalStore(
    (callback) => {
      const id = setInterval(callback, 1000);
      return () => clearInterval(id);
    },
    () => Date.now().toString(),  // client snapshot
    () => "0"                      // server snapshot (stable)
  );
}
```

**Real-world danger:**

```tsx
// ❌ Hides real bug
function UserGreeting({ userId }: { userId: string }) {
  // BUG: server fetches one user, client fetches different
  const user = useUser(userId);
  return <p suppressHydrationWarning>Hello, {user.name}</p>;
}
// User sees server-rendered name, doesn't refresh — bug hidden
```

**`onRecoverableError` for tracking:**

```tsx
const root = hydrateRoot(container, <App />, {
  onRecoverableError: (error) => {
    if (error.message.includes("Hydration")) {
      analytics.track("hydration_mismatch", {
        path: location.pathname,
      });
    }
  },
});
```

</details>

### Edge Cases

- **Multiple children**: Only direct text suppressed. Nested elements still checked.
- **Attribute mismatch**: Not suppressed — only text content.
- **`<Suspense>` boundary**: Mismatch in boundary triggers fallback render, not warning suppression.

### Follow-up savollar

- "Better than suppressHydrationWarning?" — `useEffect` for client-only data, `useSyncExternalStore` for streams.
- "Does it work in concurrent mode?" — Yes. Hydration concurrent-aware in R18+.
- "ESLint rule against this?" — No standard. Consider custom rule for code review.

</details>

---

### 42. Bug fix: hydration mismatch scenarios [Bug Fix] [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi SSR komponent bir nechta hydration mismatch'ga olib keladi. Topish va tuzatish:

```tsx
function Dashboard({ posts }) {
  const isMobile = window.innerWidth < 768;
  const time = new Date().toLocaleString();

  return (
    <div>
      <h1>Dashboard ({time})</h1>
      {isMobile ? <MobileNav /> : <DesktopNav />}

      <ul>
        {posts.map((post, i) => (
          <li key={Math.random()}>
            <a id={`link-${Math.random()}`} href={post.url}>
              {post.title}
            </a>
          </li>
        ))}
      </ul>

      <p suppressHydrationWarning>
        User logged in at: {Date.now()}
      </p>
    </div>
  );
}
```

### Javob

**Topilgan muammolar:**

1. **❌ `window.innerWidth` in render** — server'da `window` undefined → crash yoki null
2. **❌ `new Date()` in render** — server vs client farqli
3. **❌ `Math.random()` for key** — different keys server vs client
4. **❌ `Math.random()` for id** — different IDs
5. **❌ `Date.now()` in render** — different time
6. **❌ `suppressHydrationWarning` real bug yashiradi** — Date.now bug, aslida fix kerak
7. **❌ TypeScript yo'q**

**Tuzatilgan kod:**

```tsx
// app/dashboard/page.tsx — Server Component yoki SSR
interface Post {
  id: string;
  url: string;
  title: string;
}

interface DashboardProps {
  posts: Post[];
  loginTime: string;  // ISO string from server (passed as prop)
}

function Dashboard({ posts, loginTime }: DashboardProps) {
  return (
    <div>
      <h1>Dashboard</h1>

      {/* ✅ Responsive nav — CSS media query yoki ResponsiveContainer */}
      <ResponsiveNav />

      <ul>
        {posts.map((post) => (
          <li key={post.id}>  {/* ✅ Stable backend ID */}
            <PostLink post={post} />  {/* useId ichida */}
          </li>
        ))}
      </ul>

      <p>
        Logged in at: <FormatTime iso={loginTime} />
      </p>
    </div>
  );
}

// Client Component — handles browser-only rendering
"use client";
import { useEffect, useState } from "react";

function ResponsiveNav() {
  const [isMobile, setIsMobile] = useState(false);

  useEffect(() => {
    const check = () => setIsMobile(window.innerWidth < 768);
    check();
    window.addEventListener("resize", check);
    return () => window.removeEventListener("resize", check);
  }, []);

  // SSR-safe: server renders DesktopNav (default)
  // Client effect updates if mobile
  return isMobile ? <MobileNav /> : <DesktopNav />;
}

// Or with useSyncExternalStore (concurrent-safe)
function ResponsiveNavBetter() {
  const isMobile = useSyncExternalStore(
    (callback) => {
      window.addEventListener("resize", callback);
      return () => window.removeEventListener("resize", callback);
    },
    () => window.innerWidth < 768,  // client snapshot
    () => false                      // server snapshot
  );
  return isMobile ? <MobileNav /> : <DesktopNav />;
}

// Client Component for ID
"use client";
function PostLink({ post }: { post: Post }) {
  const linkId = useId();  // ✅ SSR-safe per-instance

  return (
    <a id={linkId} href={post.url}>
      {post.title}
    </a>
  );
}

// Format time — locale-aware
function FormatTime({ iso }: { iso: string }) {
  const [formatted, setFormatted] = useState(iso);  // ISO fallback

  useEffect(() => {
    setFormatted(new Date(iso).toLocaleString());
  }, [iso]);

  return <time dateTime={iso}>{formatted}</time>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Bug 1: `window` in SSR:**

```typescript
// Server: window === undefined
const isMobile = window.innerWidth < 768;  // TypeError: window is not defined
```

Fixes:
- `typeof window !== "undefined"` check
- `useEffect` (client-only)
- `useSyncExternalStore` (concurrent-safe)
- `"use client"` directive (RSC)

**Bug 2-3: Date in render:**

```typescript
// Server: 2026-01-01 12:00:00
// Client: 2026-01-01 12:00:05 (delay)
new Date().toLocaleString();  // Different output
```

Fix: Server pass ISO string, client format. Or `useEffect` for client-only display.

**Bug 4-5: Random IDs:**

```typescript
// Each render: new value
key={Math.random()}        // Reconciler can't match
id={`link-${Math.random()}`}  // DOM IDs differ
```

Fix: Backend ID for keys, `useId` for component IDs.

**Bug 6: Misuse of suppressHydrationWarning:**

```tsx
// ❌ Hides bug
<p suppressHydrationWarning>{Date.now()}</p>
// Server time persists, client never updates
// User sees stale time

// ✅ Real fix
<FormatTime iso={loginTimeIso} />
// Component handles SSR-safe rendering
```

**Bug 7: TypeScript:**

```typescript
function Dashboard({ posts }) {  // ❌ no type
function Dashboard({ posts }: DashboardProps) {  // ✅ type
```

**Test for SSR mismatches:**

```typescript
import { renderToString } from "react-dom/server";

test("dashboard renders without window error", () => {
  const posts = [{ id: "1", url: "/p1", title: "Post 1" }];
  expect(() => renderToString(<Dashboard posts={posts} loginTime="2026-01-01T12:00:00Z" />))
    .not.toThrow();
});
```

**Production monitoring:**

```typescript
const root = hydrateRoot(container, <App />, {
  onRecoverableError: (error, info) => {
    Sentry.captureMessage(`Hydration: ${error.message}`, {
      level: "warning",
      extra: { componentStack: info.componentStack },
    });
  },
});
```

**Common patterns summary:**

| Issue | Fix |
|-------|-----|
| `window`/`document` in render | `typeof window` guard, `useEffect`, `useSyncExternalStore` |
| `Date.now()`, `new Date()` | Server pass ISO, client format |
| Random IDs | Backend ID, `useId` |
| Random keys | Backend ID |
| Locale formatting | Client-side via `useEffect` |
| User-specific data (cookies, auth) | RSC reads cookie server-side, pass as prop |

</details>

### Edge Cases

- **`navigator` API**: Same as `window` — guard or useEffect.
- **`document.cookie` in render**: Server has cookies (request), but mismatch if client doesn't.
- **SSR + i18n**: Server may detect locale from header, client may differ. Pass locale as prop.

### Follow-up savollar

- "How to debug hydration mismatch?" — Browser console error includes component stack. React DevTools highlights mismatch.
- "Prevent all hydration issues — single rule?" — "Render must be deterministic given SSR data." Pass everything via props.
- "Streaming SSR + Suspense — different debugging?" — Suspended subtree may render different on resolve. Same principle: stable data.

</details>

---

## Xulosa

Bu faylda quyidagilar yoritildi:

**QISM A — Fiber Architecture (1-13)**: Fiber nima va Stack Reconciler farqi, tree linked list (child/sibling/return), double buffering (current/workInProgress), alternate pointer recycling, fiber tag types (FunctionComponent, HostComponent, etc.), effect list (flags + subtreeFlags), iterative work loop, traversal order, Component vs Fiber tree, FiberRoot vs Fiber, stateNode, memoizedState hooks linked list.

**QISM B — Reconciliation (14-25)**: O(n) heuristics, type comparison (same vs different), bailout algorithm (4 sabab: element identity, memo, useMemo, state equality), sibling matching (keyless vs keyed), update propagation (lanes + childLanes), `React.memo` shallow check, output traces, list rendering bug fixes, Context propagation, keyed list reconciliation.

**QISM C — Scheduler & Lanes (26-34)**: Concurrent rendering rationale, scheduler package + 5 priority levels, lanes bitmap (31 lanes, OR/AND ops), time slicing (5ms work loop), interruptible rendering (high-pri abort), starvation prevention (lane expiration), MessageChannel vs requestIdleCallback choice, lane operations, useTransition internals.

**QISM D — Hydration (35-42)**: SSR + Hydration concept, `hydrateRoot` vs `createRoot`, hydration mismatch causes + fixes, selective hydration (R18 Suspense), streaming SSR (`renderToReadableStream`/`renderToPipeableStream`), R19 improvements (better errors, document metadata hoisting, stylesheet precedence, root error callbacks), `suppressHydrationWarning`, hydration mismatch bug fix.

**Keyingi:** [03-hooks.md](03-hooks.md) — Hooks fundamentals + 10 ta hook (useState, useEffect, useLayoutEffect, useRef, useContext, useReducer, useMemo, useCallback, concurrent hooks, R19 hooks, custom hooks). Eng katta interview fayli — hooks mexanikasi, Rules of Hooks, linked list internal, real-world patterns.





