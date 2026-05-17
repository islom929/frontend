# Bo'lim 1: React Nima va Qanday Ishlaydi

> React — Facebook (Meta) tomonidan ochilgan declarative UI library. Component-based architecture, one-way data flow va platform-agnostic renderer modeliga asoslangan. Bu bo'lim React'ning umumiy mental modelini, tarixini va keyingi bo'limlar uchun zarur tushunchalarni o'rnatadi.

---

## Mundarija

- [React Nima](#react-nima)
- [React Tarixi](#react-tarixi)
- [Declarative vs Imperative](#declarative-vs-imperative)
- [Component-Based Architecture](#component-based-architecture)
- [Virtual DOM Tushunchasi](#virtual-dom-tushunchasi)
- [One-Way Data Flow](#one-way-data-flow)
- [Renderer Concept](#renderer-concept)
- [React 18 → 19 Qisqa](#react-18--19-qisqa)
- [React Compiler Intro](#react-compiler-intro)
- [React Server Components Intro](#react-server-components-intro)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## React Nima

### Nazariya

React — JavaScript bilan UI qurish uchun mo'ljallangan **declarative**, **component-based** library. Facebook (hozir Meta) tomonidan 2011-yilda ichki loyiha sifatida boshlanib, 2013-yilda ochiq manba qilingan. Asosiy vazifasi — komponent (function yoki class) sifatida e'lon qilingan UI tasviriga qarab, real DOM (yoki boshqa platform target) ni samarali yangilash.

React **framework EMAS, library**. Bu farq texnik jihatdan ham, ekosistemada ham muhim:

| Tushuncha | Framework | Library |
|-----------|-----------|---------|
| **Boshqaruv** | Framework siz qoidalarga amal qilishingizni majbur qiladi (Inversion of Control) | Siz library'ni o'zingiz xohlagan joyda chaqirasiz |
| **Misollar** | Angular, Next.js, Remix, NestJS | React, Lodash, Zod |
| **Loyiha strukturasi** | Frameworkda belgilangan (folder layout, conventions) | Erkin |
| **Routing, build, server** | Frameworkda kelgan | Tashqi yechimlar (siz tanlaysiz) |
| **O'rganish egri chizig'i** | Yuqori (ko'p convention) | Pastroq, lekin tanlash mas'uliyati katta |

Shu sababli **React'ning yagona vazifasi — UI rendering**. Routing uchun React Router yoki TanStack Router, server uchun Next.js yoki Remix, state uchun Redux yoki Zustand — har bir muammoga ekosistemada bir nechta yechim mavjud. Bu erkinlik React'ning kuchli tomoni, ammo "qaysi library tanlash" qarorini sizning zimmangizga yuklaydi.

React paketlarining asosiy modulyar tuzilishi:

- **`react`** — core paket: hooks (`useState`, `useEffect`), `createElement`, component abstraktlar. Bu paket **DOM, native, CLI haqida hech narsa bilmaydi**.
- **`react-dom`** — web renderer (brauzer DOM). Faqat brauzerda ishlaydi.
- **`react-native`** — mobile renderer (iOS/Android native komponentlar).
- **`react-reconciler`** — diffing va scheduling algoritmi. Renderer'lar shu paketni `Host Config` interfeysi orqali ishlatadi.

Bu modulyar tuzilish — React'ning "**reconciler + renderer**" architectura'sining asosiy g'oyasi. Bitta `react` paketi ko'plab renderer bilan ishlay oladi: `react-dom` (web), `react-native` (mobile), `react-three-fiber` (Three.js/WebGL, community), `ink` (CLI, community), `react-pdf` (PDF, community), `react-test-renderer` (testing), va h.k. Barchasi `react-reconciler` paketini ortidan ishlatadi.

<details>
<summary><strong>Under the Hood</strong></summary>

React'ning ichki tuzilishi quyidagi qatlamlardan iborat:

```
┌─────────────────────────────────────┐
│  Sizning komponentingiz (App.tsx)   │
│  function App() { return <h1>...</> } │
└──────────────┬──────────────────────┘
               │ JSX → React Elements (oddiy obyektlar)
               ▼
┌─────────────────────────────────────┐
│  react (core paket)                 │
│  - createElement / jsx-runtime      │
│  - hooks (useState, useEffect, ...) │
│  - Component, Fragment, Suspense    │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  react-reconciler                   │
│  - Fiber tree (current/workInProgress)│
│  - Diffing algorithm                │
│  - Scheduler (priority lanes)       │
└──────────────┬──────────────────────┘
               │ Host Config interface
               │ (createInstance, appendChild,
               │  commitUpdate, removeChild, ...)
               ▼
┌─────────────────────────────────────┐
│  Renderer                           │
│  - react-dom (web)                  │
│  - react-native (mobile)            │
│  - react-three-fiber (WebGL)        │
│  - ink (terminal)                   │
└──────────────┬──────────────────────┘
               │
               ▼
        Real platform output
        (DOM, Native UI, Canvas, terminal text)
```

Reconciler **algoritm** sifatida ishlaydi: u qaysi komponent yangilanishi kerak, qaysi tartibda, qachon to'xtatib turish mumkinligini hisoblaydi. Lekin u **real DOM ga tegmaydi** — buning uchun renderer'ning Host Config metodlarini chaqiradi (`createInstance`, `appendChild`, va h.k.).

Bu ajratish (separation) — React 16-versiyada (Fiber rewrite) joriy etilgan asosiy architectural qaror. Eski Stack Reconciler (React 15 va undan oldin) DOM bilan chambarchas bog'langan edi, shuning uchun React Native uchun mustaqil renderer yozish qiyin edi. Yangi reconciler — `react-reconciler` npm paketi sifatida ham mustaqil ishlatish mumkin (custom renderer yozish uchun).

Reconciler va Renderer alohida ekanligi quyidagi imkoniyatlarni beradi:
- **Concurrent rendering** — reconciler ishni qism-qism (5ms chunk) qila oladi, renderer'ga to'liq tayyor bo'lganda commit qiladi.
- **Cross-platform code reuse** — bitta komponent (logic) bir nechta platformada ishlay oladi (agar platform-specific API ishlatilmasa).
- **Custom renderer'lar** — `react-reconciler` paketi orqali siz o'z renderer'ingizni yozishingiz mumkin (masalan, React → AWS Cloud Resources, React → SVG-only, va h.k.).

Reconciler arxitekturasi `03-fiber-architecture.md` da chuqur yoritiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Eng oddiy React komponenti:

```tsx
// App.tsx
function App() {
  return <h1>Salom, React!</h1>;
}

export default App;
```

Render qilish (web entry point):

```tsx
// main.tsx
import { createRoot } from 'react-dom/client';
import App from './App';

const container = document.getElementById('root');
if (!container) throw new Error('Root element topilmadi');

const root = createRoot(container);
root.render(<App />);
```

Brauzer DOM da hosil bo'ladigan natija:

```html
<div id="root">
  <h1>Salom, React!</h1>
</div>
```

JSX syntax kompilyatordan keyin (Babel yoki SWC) shunday ko'rinadi:

```typescript
// JSX automatic transform (R17+, hozirgi default):
import { jsx as _jsx } from 'react/jsx-runtime';

function App() {
  return _jsx('h1', { children: 'Salom, React!' });
}
```

`_jsx('h1', { children: '...' })` — oddiy JavaScript funksiyasi chaqiruvi. U React Element deb ataladigan oddiy obyekt qaytaradi:

```typescript
{
  type: 'h1',
  props: { children: 'Salom, React!' },
  key: null,
  // ...va React ichki maydonlari
}
```

Bu obyekt — UI tasvirining "blueprint" — keyin reconciler tomonidan haqiqiy DOM elementiga aylantirilmaguncha hech qanday DOM yaratilmagan. JSX detallari `07-jsx.md` da batafsil yoritiladi.

</details>

---

## React Tarixi

### Nazariya

React'ning rivojlanishi 15 yildan ortiq vaqtni qamrab oladi. Asosiy bosqichlar:

| Yil | Voqea | Ahamiyati |
|-----|-------|-----------|
| **2011** | Facebook'da ichki loyiha (Newsfeed) | Jordan Walke yaratdi, faqat FB ichida ishlatildi |
| **2013 (May)** | JSConf US'da open source | Birinchi public release; ko'plari "JSX'ni HTML'ni JS'ga aralashtirgan" deb tanqid qildi |
| **2015 (Mart)** | React Native | Mobile platforma — bitta library web va mobile uchun (cross-platform code reuse) |
| **2016 (Aprel)** | React 15 | DOM optimizatsiyalar, ammo asosiy o'zgarishlar 16-da |
| **2017 (Sentabr)** | React 16 — **Fiber rewrite** | Yangi reconciler architectura; interruptible rendering uchun asos |
| **2018** | Context API (stable), `React.lazy` + Suspense | Code splitting deklarativ formada |
| **2019 (Fevral)** | React 16.8 — **Hooks** | Class komponentlarisiz state va lifecycle; React'ning eng katta API o'zgarishi |
| **2020 (Oktabr)** | React 17 | Yangi xususiyat YO'Q. JSX automatic transform, event delegation root'ga ko'chdi (microfrontend qo'llab-quvvatlash uchun) |
| **2022 (Mart)** | React 18 | **Concurrent Rendering** stable, Automatic Batching, `useTransition`/`useDeferredValue`, Suspense for SSR |
| **2024 (Dekabr)** | React 19 | Actions, `use()` hook, `useActionState`/`useFormStatus`/`useOptimistic`, ref oddiy prop, Document Metadata API, Server Components stable spec |
| **2024+** | React Compiler | Build-time auto-memoization (beta, stable yo'lida) |

**Asosiy tendensiyalar:**

1. **Class → Function** (2019, Hooks). Class komponentlar hali ishlaydi, lekin yangi kod deyarli butunlay function komponentlarda yoziladi.
2. **Sync → Concurrent** (2022, R18). Rendering uziluvchi (interruptible) bo'ldi — React priority asosida ishni to'xtatib, qayta yo'naltira oladi.
3. **Manual → Auto memoization** (2024+, Compiler). `useMemo`, `useCallback`, `React.memo` qo'lda qo'yish o'rniga build-time tahlil.
4. **Client → Server + Client** (2024, RSC). Komponentlar ikki turga bo'lindi — server'da render bo'ladigan (JS yo'q) va client'da hydrate bo'ladigan.

Har bir bosqich oldingisining muammolariga javob bo'ldi. Masalan, Hooks — class'larda logic reuse qiyinligi (`HOC`, render props bilan kurashish) uchun yechim. Fiber — Stack Reconciler'ning "uziluvchi emasligi" muammosini hal qildi (uzun render UI'ni bloklab qo'yardi).

> **Eslatma:** Bu kursda **React 19 default deb hisoblanadi**. R18 farqi har relevant bo'limda inline callout sifatida ko'rsatiladi.

---

## Declarative vs Imperative

### Nazariya

**Imperative** kod — kompyuterga **qanday qilish** kerakligini qadam-baqadam aytadi: "elementni yarat, attribute o'rnat, listener qo'sh, qiymatni o'zgartir, parentga qo'sh". Bu yondashuv vanilla JavaScript va jQuery'ning asosi.

**Declarative** kod — **nima ko'rinishda bo'lishi kerakligini** tasvirlaydi, qadamlarni esa library hal qiladi: "menga shu state bilan shu UI kerak". Bu React, Vue, SwiftUI, Jetpack Compose'ning asosiy paradigmasi.

Farqning amaliy ahamiyati:

- **Imperative** — siz har bir DOM o'zgarishini qo'lda boshqarasiz. Bug'lar ko'p (DOM va state desync), kod o'sgan sayin murakkablashadi.
- **Declarative** — siz faqat **state** ni boshqarasiz. UI state ning **funksiyasi** sifatida hisoblanadi: `UI = f(state)`. State o'zgarsa — React UI ni yangilaydi.

Bu formula React'ning butun mental modelining asosi. Kursda har bir tushuncha shu formula atrofida quriladi.

> **Tarixiy kontekst:** Declarative UI 2010-yillarning oxirida frontend'ga keng kirib keldi (React 2013, Elm 2012). Lekin ilk g'oyalar ancha eski — Functional Reactive Programming (Conal Elliott, 1997), React'ga ta'sir qilgan asosiy ilhom manbalari `XHP` (Facebook), Standard ML va Haskell.

<details>
<summary><strong>Under the Hood</strong></summary>

Declarative UI'ning ichki mexanizmi: React har render'da yangi React Element tree quradi, uni eski tree bilan **diff** qiladi (Reconciliation algoritmi), va faqat **farq qilgan qismni** real DOM'ga commit qiladi.

Misol — counter o'sganda:

```
Render 1 (count = 0):                Render 2 (count = 1):
┌─────────────────────┐              ┌─────────────────────┐
│ <button>            │              │ <button>            │
│   Click count: 0    │              │   Click count: 1    │
│ </button>           │              │ </button>           │
└─────────────────────┘              └─────────────────────┘
        │                                    │
        └──────────────┬─────────────────────┘
                       ▼
              Reconciler diff:
              Same type (<button>)
              text changed: "0" → "1"
                       │
                       ▼
              Real DOM mutation:
              button.textContent = "Click count: 1"
              (faqat textContent o'zgartirildi,
               element qayta yaratilmadi)
```

Imperative kodda siz har o'zgarishda **qaysi DOM atributi yangilanishini** o'zingiz hisoblashingiz kerak. React buni avtomatik qiladi — siz faqat "yangi state'da UI shunday bo'lishi kerak" deb yozasiz.

Reconciler algoritmi `04-reconciliation.md` da batafsil yoritiladi (heuristics, key-based identity, bailout sabablari).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Bir xil counter ikki yondashuvda:

**Imperative (vanilla JavaScript):**

```typescript
// Imperative — har qadamni o'zingiz boshqarasiz
const button = document.createElement('button');
let count = 0;

function updateButton() {
  button.textContent = `Click count: ${count}`;
}

button.addEventListener('click', () => {
  count++;
  updateButton(); // ← state o'zgargandan keyin DOM'ni qo'lda yangilash
});

updateButton(); // ← initial render ham qo'lda
document.body.appendChild(button);
```

**Declarative (React):**

```tsx
import { useState } from 'react';

// Declarative — faqat "state'ga qarab UI shunday bo'lishi kerak" deb yozasiz
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Click count: {count}
    </button>
  );
}
```

Imperative variantning muammolari:
- `updateButton()` chaqirishni unutib qo'ysangiz — UI state'dan ortda qoladi
- Bir nechta DOM elementi bor bo'lsa — har biri uchun update funksiyasi yozish kerak
- Conditional rendering (`if`) bilan kod tez murakkablashadi
- State va DOM ikkita parallel "haqiqat" — desync xavfi

Declarative variant:
- `setCount(count + 1)` chaqirilsa — React qolganini hal qiladi
- UI doim state'ning aks ettirishi (no desync)
- Conditional rendering oddiy `{condition && <Component />}` bilan
- State bitta haqiqat manbai

Murakkabroq misol — TODO ro'yxati:

```tsx
// Declarative TODO list — faqat state strukturasi
interface Task {
  id: number;
  text: string;
  done: boolean;
}

function TodoList() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [input, setInput] = useState('');

  function addTask() {
    if (!input.trim()) return;
    setTasks([...tasks, { id: Date.now(), text: input, done: false }]);
    setInput('');
  }

  function toggleTask(id: number) {
    setTasks(tasks.map(t => t.id === id ? { ...t, done: !t.done } : t));
  }

  return (
    <div>
      <input
        value={input}
        onChange={(e) => setInput(e.target.value)}
        placeholder="Yangi vazifa"
      />
      <button onClick={addTask}>Qo'shish</button>
      <ul>
        {tasks.map(task => (
          <li
            key={task.id}
            onClick={() => toggleTask(task.id)}
            style={{ textDecoration: task.done ? 'line-through' : 'none' }}
          >
            {task.text}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

Imperative versiya bu xil ro'yxat uchun ~3-4 marta uzun bo'ladi: har element uchun `<li>` yaratish, `appendChild`, click listener qo'shish, toggle uchun textContent va style yangilash, delete uchun removeChild — barchasini qo'lda boshqarish.

</details>

---

## Component-Based Architecture

### Nazariya

**Component** — React'ning eng asosiy qurilish bloki. Texnik jihatdan — **React Element qaytaradigan funksiya** (yoki class). Konseptual jihatdan — UI'ning mustaqil, qayta ishlatish mumkin bo'lgan qismi.

Komponent quyidagi xususiyatlarga ega:

1. **Encapsulation** — komponent o'z state, logic va UI ni o'z ichida saqlaydi
2. **Composability** — komponentlar bir-birini ichiga joylashtirib (compose qilib) murakkab UI qurish mumkin
3. **Reusability** — bir komponent ko'p marta ishlatilishi mumkin (turli prop'lar bilan)
4. **Isolation** — bir komponent ichidagi state boshqasiga ta'sir qilmaydi (agar maxsus boshqarilmasa)

Component naming konventsiyasi: **PascalCase** (`UserCard`, `LoginForm`). Bu JSX transform qoidasi bilan bog'liq:
- `<userCard />` (lowercase) — HTML tag deb hisoblanadi (`<usercard>` element)
- `<UserCard />` (capitalized) — React component deb hisoblanadi

Bu farq Babel/SWC darajasida hal qilinadi — kompilyator birinchi harf katta bo'lsa `_jsx(UserCard, {...})` deb yozadi, kichik bo'lsa `_jsx('userCard', {...})` deb yozadi (string sifatida — DOM element deb hisoblaydi).

**Funksiya komponent vs Class komponent:**

Modern React'da **function komponent default**. Class komponent faqat **Error Boundary** uchun majburan ishlatiladi (boshqa hech qanday holatda kerak emas). 2019-yilda Hooks joriy etilganidan beri function komponent class'ning barcha imkoniyatlariga ega:
- State (`useState`, `useReducer`)
- Lifecycle (`useEffect`, `useLayoutEffect`)
- Context (`useContext`)
- Refs (`useRef`)
- Memoization (`useMemo`, `useCallback`)

> **Versiya evolyutsiyasi:**
> - **Pre-R16.8 (2019-yilgacha):** Class komponentlar majburiy edi — state va lifecycle uchun. Function komponentlar "stateless functional component" deb atalardi va faqat oddiy UI uchun ishlatilardi.
> - **R16.8+ (Hooks):** Function komponentlarga Hooks bilan barcha imkoniyatlar berildi. Class — faqat Error Boundary uchun.
> - **Sabab:** Class'da logic reuse qiyin edi (HOC va render props bilan "wrapper hell"), `this` binding murakkab edi, lifecycle methodlar bir-biriga aralash logic'ni majbur qilardi (`componentDidMount` da setup, `componentWillUnmount` da cleanup — bir-biridan uzoqda).

<details>
<summary><strong>Under the Hood</strong></summary>

React Komponent **Fiber node** sifatida ichki tarzda ifodalanadi. Har bir komponent instansiyasi uchun React bitta Fiber yaratadi va uni Fiber tree'ga joylashtiradi.

Fiber node tuzilishi (soddalashtirilgan):

```
Fiber {
  type: function | string,    // FunctionComponent yoki 'div' kabi
  tag: number,                // FunctionComponent = 0, HostComponent = 5, va h.k.
  stateNode: ...,             // class instance yoki DOM node
  memoizedState: ...,         // hooks linked list (function component uchun)
  memoizedProps: ...,         // joriy props
  child: Fiber | null,        // birinchi farzand
  sibling: Fiber | null,      // keyingi qardosh
  return: Fiber | null,       // ota Fiber
  alternate: Fiber | null,    // double buffering: current vs workInProgress
  flags: number,              // effect flag bitmask
}
```

Komponent render qilinganda:
1. React komponent funksiyasini chaqiradi (`App()`)
2. Funksiya React Element tree qaytaradi
3. React Element'lar Fiber'larga aylantiriladi (`reconcileChildren`)
4. Yangi Fiber tree (workInProgress) eski tree (current) bilan diff qilinadi
5. Farq qilgan qismlar real DOM'ga commit qilinadi

Fiber arxitekturasi `03-fiber-architecture.md` da chuqur yoritiladi (tag types, double buffering, effect list).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Oddiy komponent (props bilan):

```tsx
interface UserCardProps {
  name: string;
  email: string;
  avatarUrl?: string;
}

function UserCard({ name, email, avatarUrl }: UserCardProps) {
  return (
    <article className="user-card">
      {avatarUrl && <img src={avatarUrl} alt={name} />}
      <h2>{name}</h2>
      <a href={`mailto:${email}`}>{email}</a>
    </article>
  );
}
```

Komponent qayta ishlatish (composition):

```tsx
function UserList() {
  const users = [
    { id: 1, name: 'Ali', email: 'ali@example.com' },
    { id: 2, name: 'Vali', email: 'vali@example.com' },
    { id: 3, name: 'Sami', email: 'sami@example.com' },
  ];

  return (
    <section>
      <h1>Foydalanuvchilar</h1>
      {users.map(user => (
        <UserCard
          key={user.id}
          name={user.name}
          email={user.email}
        />
      ))}
    </section>
  );
}
```

Bir-biriga joylashtirilgan komponentlar (nesting):

```tsx
function App() {
  return (
    <Layout>
      <Header title="Mening sahifam" />
      <Main>
        <UserList />
      </Main>
      <Footer year={2026} />
    </Layout>
  );
}
```

Anti-pattern — lowercase komponent nomi:

```tsx
// ❌ NOTO'G'RI — userCard (lowercase) HTML tag deb qaraladi
function userCard({ name }: { name: string }) {
  return <h2>{name}</h2>;
}

// JSX'da:
<userCard name="Ali" />  // ← React `'userCard'` string sifatida HTML element deb interpretatsiya qiladi
// Brauzerda `<usercard>` deb yaratiladi, React komponenti chaqirilmaydi
```

```tsx
// ✅ TO'G'RI — PascalCase
function UserCard({ name }: { name: string }) {
  return <h2>{name}</h2>;
}

<UserCard name="Ali" />  // ← React komponent deb tan oladi, funksiya chaqiriladi
```

</details>

---

## Virtual DOM Tushunchasi

### Nazariya

"**Virtual DOM**" (VDOM) — React'ning eng ko'p eslatib o'tiladigan, ammo eng ko'p tushunmovchilik tug'diradigan tushunchasi. Asl ma'no: **real DOM ning JavaScript obyektlari ko'rinishidagi yengil tasviri**. React har render'da bunday "virtual" tree quradi, eski va yangi tree'larni solishtiradi (diff), va faqat farq qilgan qismni haqiqiy DOM'ga commit qiladi.

**Lekin muhim haqiqat:** zamonaviy React (R16+, 2017'dan beri) **toza Virtual DOM emas**. U Fiber arxitekturasi asosida ishlaydi — bu doubly-linked tree va work units sistemasi. "Virtual DOM" — endi **mental model**, **implementation emas**.

Bu tafovut nima uchun muhim:

1. **react.dev rasmiy hujjatlari** (2023 yangilanishidan beri) "Virtual DOM" terminini ishlatmaydi. Faqat "render output" yoki "render tree" deb yoziladi.
2. Eski tutoriallar "React VDOM bilan tez" deb yozadi — bu ham noto'g'ri framing. Tez bo'lishning sababi VDOM emas, balki **diffing algoritmi** (Reconciler) va **batched updates** (Scheduler).
3. Vue, Preact, SolidJS kabi boshqa library'lar ham "Virtual DOM" ishlatadi, ammo har birining implementatsiyasi farqli (Vue 3 — proxy-based reactivity + VDOM hybrid, Preact — strict VDOM, SolidJS — Compiled reactivity, VDOMsiz).

**To'g'ri mental model:**

```
"VDOM" = Idea (g'oya):
  "UI tasvirini JS obyektlari sifatida saqla, eski va yangi solishtir, faqat farqni DOMga yoz"

Fiber = Implementation (amaliy yechim):
  "Bu g'oyani interruptible rendering, prioritization, time slicing
   bilan kengaytirgan doubly-linked tree va work units sistemasi"
```

Kursda biz "Virtual DOM" iborasini **mental model** sifatida ishlatamiz, ammo haqiqiy mexanizmni Fiber atamalarida yoritamiz. Fiber chuqur — `03-fiber-architecture.md` da.

<details>
<summary><strong>Under the Hood</strong></summary>

React har render'da nima qiladi (soddalashtirilgan):

```
1. Component render() chaqiriladi
   ↓
2. React Element tree quriladi (oddiy JS obyektlar)
   { type: 'div', props: { children: [...] } }
   ↓
3. React Element tree → Fiber tree (workInProgress)
   reconcileChildren() — har element uchun Fiber yaratiladi yoki yangilanadi
   ↓
4. workInProgress tree current tree bilan diff qilinadi
   - Same type? → update Fiber, mark for update
   - Different type? → unmount old, mount new
   - List with keys? → match by key, reorder
   ↓
5. Effect list quriladi (Placement, Update, Deletion flag bilan Fiber'lar)
   ↓
6. Commit phase: effect list bo'ylab DOM mutations qo'llaniladi
   - createElement, appendChild, removeChild, setAttribute, ...
```

**Eski (pre-R16) Stack Reconciler:**
- Sinkron, uziluvchi emas (uninterruptible)
- Render boshlangach to'liq tugaguncha to'xtab bo'lmasdi
- Uzun render UI'ni bloklab qo'yardi (input lag, jank)
- Implementation oddiy recursion edi

**Yangi (R16+) Fiber Reconciler:**
- Linked tree (har Fiber'da `child`, `sibling`, `return` pointers — parent va sibling'larga to'g'ridan-to'g'ri kirish)
- Work loop interruptible — qisqa vaqt intervallarida ishlab, brauzerga muntazam yo'l beradi (yield)
- Priority lanes (urgent vs non-urgent)
- Restartable (high-priority work kelganda current ishni tashlab, qayta boshlaydi)

Shu sababli "Virtual DOM" termini React kontekstida **2017-yildan beri eskirgan**. Fiber butunlay boshqa narsa, lekin "Virtual DOM" g'oyasini saqlab qoldi (deklarativ, diffing-based).

Boshqa library'larda holat:
- **Vue 3** — "Virtual DOM" + Proxy-based reactivity (kombinatsiya)
- **Preact** — minimal VDOM (3KB, React API mos)
- **SolidJS** — Compiled fine-grained reactivity, **VDOM yo'q** (compile-time'da DOM mutations generatsiya qilinadi)
- **Svelte** — Compiled reactivity, **VDOM yo'q** (har component'ni o'z imperative kodiga aylantiradi)

VDOM/Fiber farqi va Reconciler algoritmi `03-fiber-architecture.md` va `04-reconciliation.md` da chuqur yoritiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

React Element — JSX kompilyatordan keyin shunday ko'rinadi:

```tsx
// JSX:
const element = <h1 className="title">Salom</h1>;

// JSX automatic transform (R17+) keyin:
import { jsx as _jsx } from 'react/jsx-runtime';
const element = _jsx('h1', { className: 'title', children: 'Salom' });

// Hosil bo'ladigan obyekt (React Element):
{
  type: 'h1',
  props: {
    className: 'title',
    children: 'Salom',
  },
  key: null,
  ref: null,
  // React ichki maydonlari
}
```

Tree ko'rinishi:

```tsx
const tree = (
  <div>
    <h1>Sarlavha</h1>
    <p>Matn</p>
  </div>
);

// Hosil bo'ladigan tree:
{
  type: 'div',
  props: {
    children: [
      { type: 'h1', props: { children: 'Sarlavha' } },
      { type: 'p', props: { children: 'Matn' } },
    ],
  },
}
```

Bu obyekt — **plain JavaScript** (sodda obyekt). Hech qanday DOM yaratilmagan. React keyin bu obyektni Fiber tree'ga aylantirib, oxirida real DOM'ga commit qiladi.

Diff misoli (mental model):

```tsx
// Render 1 (count = 0):
<div>
  <h1>Sarlavha</h1>
  <p>Soni: 0</p>
</div>

// Render 2 (count = 1):
<div>
  <h1>Sarlavha</h1>
  <p>Soni: 1</p>
</div>

// Reconciler diff:
// - <div> same type → keep
// - <h1> same type, same props → no update
// - <p> same type, children changed: "Soni: 0" → "Soni: 1" → update text only
//
// Real DOM mutation:
// pTextNode.nodeValue = "Soni: 1"  ← faqat text node yangilanadi
```

</details>

---

## One-Way Data Flow

### Nazariya

React'da ma'lumot **bitta yo'nalishda** oqadi: **yuqoridan pastga** (parent → child, props orqali). Pastdan yuqoriga aloqa — **callback funksiyalar** orqali (event handler'larni prop sifatida o'tkazish).

Bu yondashuvning rasmiy nomi — **"unidirectional data flow"** yoki **"props down, events up"**.

```
       ┌──────────┐
       │  Parent  │  ← state shu yerda
       └────┬─────┘
            │
        props ▼   ▲ callback
            │   │
       ┌────┴───┴────┐
       │   Child     │
       └─────────────┘
```

**Nima uchun bir yo'nalishli:**

1. **Predictability** (oldindan aytib bo'ladiganlik) — state qaerdan kelganini va qayerga ketishini aniq bilish oson
2. **Debugging osonligi** — bug topilganda, state'ni o'zgartirgan komponent'ni topish oson (faqat parent o'zgartira oladi)
3. **Single Source of Truth** — har data uchun bitta "haqiqat manbai" (eng yuqori parent)
4. **Konkurent rendering bilan uyg'unlik** — React 18 concurrent mode'da rendering interruptible bo'lgani uchun ikki yo'nalishli binding race condition'larga olib kelishi mumkin edi

**Ikki yo'nalishli binding (two-way binding) — Vue va Angular yondashuvi:**

Vue'da `v-model="value"` direktivasi input qiymatini ham o'qiydi, ham yozadi — siz callback yozmaysiz, framework avtomatik sinxronizatsiya qiladi.

React'da bu pattern **explicit** yoziladi:

```tsx
<input value={value} onChange={(e) => setValue(e.target.value)} />
```

Bu kod ko'proq qator, lekin **ko'rinarli** — qiymat qaerdan kelgan va qayerga boryapti aniq.

> **Eslatma:** "Two-way binding" Vue/Angular'da ham aslida bir yo'nalishli ostida sintaktik shakar (sugar) — Vue ham `v-model` ni `:value + @input` ga aylantiradi. Farq faqat sintaktik.

<details>
<summary><strong>Under the Hood</strong></summary>

React'ning data flow modelining ichki sabablari:

**1. Render funksiyasi pure bo'lishi kerak.** Komponent funksiyasi `props → JSX` aylantirish bo'lib, bir xil kirish bilan bir xil chiqishni qaytarishi shart (idempotent). Agar child parent state'ni mutate qila olsa — render davomida side effect bo'lardi va Reconciler boshqotirma holatga tushardi.

**2. Strict Mode purity tekshiruvi.** Strict Mode (R16.3+) development'da komponent funksiyasini ikki marta chaqiradi — render purity'ni tekshirish uchun. R18'dan boshlab effect'lar uchun ham ikki marta cycle qo'shildi (mount → cleanup → mount). Agar render side effect bilan bo'lsa (parent state mutate qilish) — bug ikki marta yuz beradi va aniq ko'rinadi.

**3. Concurrent rendering interruptibility.** R18 concurrent mode'da render to'xtatib, qayta boshlanishi mumkin. Agar child render davomida parent state'ni o'zgartirsa — restart paytida holat noaniq bo'lardi.

**4. Time travel debugging.** Redux DevTools kabi vositalar state o'zgarishlarini "rewind" qiladi. Bir yo'nalishli oqim bunday "vaqt sayohati" ni mumkin qiladi.

**Texnik amalga oshirish:**

Props read-only (immutable) — JavaScript darajasida emas, lekin **React konventsiyasi** sifatida. React `Object.freeze` qilmaydi (performance sababli), lekin props'ni mutate qilish — anti-pattern. ESLint plugin'lar bunday holatlarni topishga yordam beradi.

```tsx
function Child({ user, onChange }: { user: User; onChange: (id: number, name: string) => void }) {
  // ❌ React aniqlamaydi, lekin anti-pattern:
  user.name = "O'zgartirildi"; // ← parent ko'ra olmaydi

  // ✅ Parent'ga callback orqali xabar berish:
  return <button onClick={() => onChange(user.id, "O'zgartirildi")}>...</button>;
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Props down (parent → child):

```tsx
interface GreetingProps {
  name: string;
  language: 'uz' | 'en';
}

function Greeting({ name, language }: GreetingProps) {
  const greeting = language === 'uz' ? 'Salom' : 'Hello';
  return <p>{greeting}, {name}!</p>;
}

function App() {
  return (
    <div>
      <Greeting name="Ali" language="uz" />
      <Greeting name="Bob" language="en" />
    </div>
  );
}
```

Events up (child → parent, callback orqali):

```tsx
interface CounterButtonProps {
  count: number;
  onIncrement: () => void;
}

function CounterButton({ count, onIncrement }: CounterButtonProps) {
  return (
    <button onClick={onIncrement}>
      Bosildi: {count}
    </button>
  );
}

function App() {
  const [count, setCount] = useState(0);

  return (
    <CounterButton
      count={count}
      onIncrement={() => setCount(count + 1)}
    />
  );
}
```

E'tibor bering: `count` state **`App`** komponentida saqlanadi (parent). `CounterButton` faqat ko'rsatadi va o'zgartirish so'rovini parent'ga jo'natadi (`onIncrement` callback orqali).

Murakkabroq misol — controlled input:

```tsx
interface SearchInputProps {
  value: string;
  onChange: (value: string) => void;
  placeholder?: string;
}

function SearchInput({ value, onChange, placeholder }: SearchInputProps) {
  return (
    <input
      type="text"
      value={value}
      onChange={(e) => onChange(e.target.value)}
      placeholder={placeholder}
    />
  );
}

function ProductList() {
  const [search, setSearch] = useState('');
  const products = [
    { id: 1, name: 'Olma' },
    { id: 2, name: 'Anor' },
    { id: 3, name: 'Banan' },
  ];

  const filtered = products.filter(p =>
    p.name.toLowerCase().includes(search.toLowerCase())
  );

  return (
    <div>
      <SearchInput
        value={search}
        onChange={setSearch}
        placeholder="Mahsulot qidirish..."
      />
      <ul>
        {filtered.map(p => <li key={p.id}>{p.name}</li>)}
      </ul>
    </div>
  );
}
```

`search` state `ProductList`'da saqlanadi. `SearchInput` faqat `value` prop'ni ko'rsatadi va o'zgarish so'rovini `onChange` orqali parent'ga jo'natadi. Parent state'ni yangilab, qayta render qiladi — `SearchInput` yangi `value` prop oladi.

Controlled vs Uncontrolled inputs `14-lifting-and-controlled.md` da batafsil yoritiladi.

</details>

---

## Renderer Concept

### Nazariya

React'ning eng ko'p eslatilmaydigan, ammo eng kuchli arxitektura xususiyati — **Reconciler va Renderer ajratilgan**. Ya'ni `react` paketi DOM haqida hech narsa bilmaydi. U faqat komponent tree'ni boshqaradi, real platform target'ga (DOM, Native, va h.k.) ulanish — alohida renderer paketning vazifasi.

Asosiy renderer'lar:

| Renderer | Platform | Maqsad | Egasi |
|----------|----------|--------|-------|
| **`react-dom`** | Web (HTML/DOM, JSDOM) | Brauzerda ishlash; `createRoot`, `hydrateRoot` API | React jamoasi |
| **`react-native`** | iOS / Android | Native komponentlar (UIView, Android View) | Meta |
| **`react-test-renderer`** | Testing | Komponentlarni DOM'siz JS obyekt sifatida render qilish | React jamoasi |
| **`react-three-fiber`** | Three.js (WebGL) | 3D scene grafikasi React komponentlari sifatida | Community (Poimandres) |
| **`ink`** | Terminal (CLI) | Terminal CLI'lar React komponentlari bilan | Community (Vadim Demedes) |
| **`react-pdf`** | PDF | PDF hujjatlar React komponentlari bilan | Community |

Bularning hammasi `react-reconciler` paketi orqali **bir xil `react` core**ni ishlatadi. Komponentlar (logic) — bir xil. Faqat renderer (Host Config) farq qiladi.

**Nima uchun bu g'oya muhim:**

1. **Cross-platform code reuse** — bir komponent web va mobile'da (yoki boshqa platform'da) ishlatish mumkin (agar platform-specific API ishlatilmasa).
2. **Custom renderer yozish** — `react-reconciler` paketi orqali siz o'z renderer'ingizni yarata olasiz. Masalan, React → email HTML (`react-email`), React → terminal text (`ink`), React → 3D scene (`react-three-fiber`) — barchasi shu yondashuv bilan qurilgan.
3. **Concurrent rendering universalligi** — Reconciler renderer'dan mustaqil bo'lgani uchun, concurrent rendering xususiyatlari (lanes, time slicing) barcha renderer'lar uchun ishlatish mumkin.

**Custom renderer asoslari:**

`react-reconciler` paketi Host Config interfeysini kutadi — siz quyidagi metodlarni implement qilasiz:

| Metod | Ma'no |
|-------|-------|
| `createInstance(type, props)` | Yangi platform element yaratish |
| `createTextInstance(text)` | Text node yaratish |
| `appendChild(parent, child)` | Element'ni parent'ga qo'shish |
| `removeChild(parent, child)` | Element'ni o'chirish |
| `commitUpdate(instance, prevProps, nextProps)` | Element propertyrlarini yangilash |
| `prepareForCommit()` | Commit boshidan oldin tayyorgarlik |
| `resetAfterCommit()` | Commit oxiridan keyin tozalash |

Bu metodlar implement qilingach, React Reconciler sizning platform target'ingiz uchun ishlay boshlaydi.

> **Tarixiy kontekst:** Reconciler/Renderer ajratilishi React 16 (Fiber rewrite) bilan to'liq amalga oshirildi. Eski React 15'da Reconciler DOM bilan chambarchas bog'langan edi — shuning uchun React Native ham alohida fork sifatida ishlangan edi. Yangi arxitekturada `react-reconciler` umumiy paket, har platform uni ishlatadi.

<details>
<summary><strong>Under the Hood</strong></summary>

React'ning paket arxitekturasi:

```
                    ┌──────────────┐
                    │    react     │  ← core abstraktlar
                    │  (createElement, hooks, Component) │
                    └──────┬───────┘
                           │ (peer dependency)
                           │
              ┌────────────┴────────────┐
              │                         │
    ┌─────────▼──────────┐    ┌─────────▼──────────┐
    │  react-reconciler  │    │  react-server      │  ← RSC infrastructure
    │  (algoritm)        │    └────────────────────┘
    └─────────┬──────────┘
              │ (Host Config interface)
              │
    ┌─────────┴─────────────────────────────────┐
    │                                           │
┌───▼───────┐  ┌──────────────┐  ┌──────────┐  ┌▼─────────┐
│ react-dom │  │react-native  │  │   ink    │  │  custom  │
└───┬───────┘  └──────┬───────┘  └────┬─────┘  └──────────┘
    │                 │               │
    ▼                 ▼               ▼
   DOM            Native UI       Terminal
```

Har bir renderer Host Config'ni implement qiladi va `react-reconciler` paketga uzatadi:

```typescript
import Reconciler from 'react-reconciler';

const hostConfig = {
  createInstance(type, props) {
    // Platform element yaratish
    return platformAPI.create(type, props);
  },
  appendChild(parent, child) {
    parent.appendChild(child);
  },
  // ...va boshqa metodlar
};

const reconciler = Reconciler(hostConfig);

export function render(element, container) {
  const root = reconciler.createContainer(container, ...);
  reconciler.updateContainer(element, root);
}
```

**`react-reconciler` paketi** keng tarqalmagan, lekin u haqiqiy npm paketi. Asosiy renderer'lar (react-dom, react-native) shu paketni ishlatadi.

**Component logic platform-agnostic:**

```tsx
// Bu komponent platformaga bog'liq emas — DOM yo'q
function Counter() {
  const [count, setCount] = useState(0);
  return /* JSX qaytaradi, lekin element type'i platformga qarab interpretatsiya qilinadi */;
}
```

Lekin JSX element tag'lari platform'ga xos:
- Web: `<div>`, `<button>`, `<input>`
- Native: `<View>`, `<Text>`, `<TouchableOpacity>`
- Three.js: `<mesh>`, `<boxGeometry>`, `<meshStandardMaterial>`
- Ink: `<Box>`, `<Text>`

Shu sababli "logic shared" lekin "view layer platform-specific" — faqat hooks va business logic'ni shared qilib, komponentlarni platform per implementatsiya qilish kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Bir xil `Counter` logic, har xil renderer:

**Web (react-dom):**

```tsx
import { createRoot } from 'react-dom/client';
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount(count + 1)}>
      Soni: {count}
    </button>
  );
}

const container = document.getElementById('root')!;
createRoot(container).render(<Counter />);
```

**Mobile (react-native):**

```tsx
import { AppRegistry, View, Text, TouchableOpacity } from 'react-native';
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  return (
    <TouchableOpacity onPress={() => setCount(count + 1)}>
      <Text>Soni: {count}</Text>
    </TouchableOpacity>
  );
}

AppRegistry.registerComponent('MyApp', () => Counter);
```

**Terminal (ink):**

```tsx
import { render, Box, Text, useInput } from 'ink';
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  useInput((input) => {
    if (input === ' ') setCount(count + 1);
  });
  return (
    <Box>
      <Text>Soni: {count} (probel bosing)</Text>
    </Box>
  );
}

render(<Counter />);
```

**3D scene (react-three-fiber):**

```tsx
import { Canvas, useFrame } from '@react-three/fiber';
import { useState, useRef } from 'react';
import { Mesh } from 'three';

function SpinningCube() {
  const meshRef = useRef<Mesh>(null);
  const [hovered, setHovered] = useState(false);

  useFrame(() => {
    if (meshRef.current) {
      meshRef.current.rotation.y += 0.01;
    }
  });

  return (
    <mesh
      ref={meshRef}
      onPointerOver={() => setHovered(true)}
      onPointerOut={() => setHovered(false)}
    >
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color={hovered ? 'orange' : 'gray'} />
    </mesh>
  );
}

function App() {
  return (
    <Canvas>
      <ambientLight />
      <SpinningCube />
    </Canvas>
  );
}
```

E'tibor bering: `useState`, `useRef`, `useFrame` (custom hook) — barchasi bir xil React hooks API. Faqat JSX tag'lari platform'ga xos.

Bu kursda biz **`react-dom` (web) ga fokuslanamiz**, lekin barcha hooks va concept'lar boshqa renderer'larda ham bir xil ishlaydi.

</details>

---

## React 18 → 19 Qisqa

### Nazariya

React 18 (2022 mart) va React 19 (2024 dekabr) — eng katta ikki versiya. Quyida asosiy o'zgarishlar (har biri keyingi bo'limlarda chuqur yoritiladi):

**React 18 (2022) asosiy yangiliklar:**

| Xususiyat | Tavsif | Qayerda yoritiladi |
|-----------|--------|---------------------|
| **Concurrent Rendering** | Interruptible rendering, lanes, time slicing | `05-scheduler-lanes.md`, `30-concurrent-react.md` |
| **Automatic Batching** | `setTimeout`, `Promise`, native event'larda ham batching | `02-rendering.md` |
| **`useTransition`** | Non-urgent updates uchun hook | `22-concurrent-hooks.md` |
| **`useDeferredValue`** | Qiymatni "kechiktirish" — eski qiymat'da render | `22-concurrent-hooks.md` |
| **`useId`** | SSR-safe unique ID generation | `22-concurrent-hooks.md` |
| **`useSyncExternalStore`** | External store subscription, tearing prevention | `22-concurrent-hooks.md` |
| **`useInsertionEffect`** | CSS-in-JS library'lar uchun (DOM mutationdan oldin) | `17-uselayouteffect.md` |
| **`createRoot`** | Yangi root API (eski `ReactDOM.render` deprecated) | `02-rendering.md` |
| **Suspense for SSR** | Streaming SSR, selective hydration | `06-hydration.md` |
| **Strict Mode 2x effect** | Dev rejimida `useEffect` ikki marta ishga tushadi | `16-useeffect.md` |

**React 19 (2024) asosiy yangiliklar:**

| Xususiyat | Tavsif | Qayerda yoritiladi |
|-----------|--------|---------------------|
| **`use()` hook** | Promise va Context'ni render'da o'qish | `23-r19-hooks.md` |
| **Actions** | `<form action={fn}>` — client va server actions | `13-event-handling.md`, `39-rsc-server-actions.md` |
| **`useActionState`** | Form action state (avval `useFormState`) | `23-r19-hooks.md` |
| **`useFormStatus`** | Form submission holati | `23-r19-hooks.md` |
| **`useOptimistic`** | Optimistic UI updates | `23-r19-hooks.md` |
| **Ref as prop** | `forwardRef` wrapper kerak emas — ref oddiy prop | `18-useref.md` |
| **`<Context value>`** | `<Context.Provider value>` o'rniga to'g'ridan-to'g'ri | `19-usecontext.md` |
| **Document Metadata** | `<title>`, `<meta>` har komponentda — react-helmet kerak emas | `37-react-19-document-apis.md` |
| **Stylesheet support** | `<link rel="stylesheet" precedence>` Suspense bilan | `37-react-19-document-apis.md` |
| **Resource APIs** | `preload()`, `preinit()`, `prefetchDNS()`, `preconnect()` | `37-react-19-document-apis.md` |
| **Async Scripts** | `<script async>` har komponentda, deduplication | `37-react-19-document-apis.md` |
| **Server Components stable** | RSC stable spec (framework kerak) | `39-rsc-server-actions.md` |
| **Web Components interop** | Properties va custom events to'g'ri ishlaydi | `38-web-components.md` |
| **Error reporting** | `onCaughtError`, `onUncaughtError`, `onRecoverableError` | `27-error-boundaries.md` |
| **`propTypes` / `defaultProps` (function)** | Olib tashlandi (TS bilan almashtirildi) | `10-props.md` |
| **Legacy Context API** | Olib tashlandi (`contextTypes`) | `19-usecontext.md` |
| **String refs** | Olib tashlandi (eski `ref="myInput"` syntaxi) | `18-useref.md` |

> **Eslatma:** Bu jadval — qisqa overview. Har bir xususiyat keyingi bo'limlarda **inline versiya callout** sifatida (`> **Versiya evolyutsiyasi:**` formati) batafsil yoritiladi. Migration uchun alohida sahifa yo'q — har mavzu o'z evolyutsiyasi bilan keladi.

---

## React Compiler Intro

### Nazariya

**React Compiler** — Meta tomonidan yaratilgan **build-time** optimizatsiya vositasi. Bog'lab turilgan komponent kodini tahlil qiladi va **avtomatik memoization** qo'llaydi: `useMemo`, `useCallback`, `React.memo` qo'lda yozish kerak emas.

Hozirgi (2026 boshi) holat: **beta** — production'da Meta o'z mahsulotlarida ishlatadi (Instagram web), lekin keng public release stable yo'lida.

**Asosiy g'oya:**

```tsx
// Compiler'siz (qo'lda memoization):
function ProductList({ products, filter }: Props) {
  const filtered = useMemo(
    () => products.filter(p => p.category === filter),
    [products, filter]
  );

  const handleClick = useCallback(
    (id: number) => console.log(id),
    []
  );

  return <List items={filtered} onClick={handleClick} />;
}

// Compiler bilan (avtomatik):
function ProductList({ products, filter }: Props) {
  const filtered = products.filter(p => p.category === filter);
  const handleClick = (id: number) => console.log(id);
  return <List items={filtered} onClick={handleClick} />;
}
// Compiler build paytida useMemo/useCallback'ni qo'shib qo'yadi
```

**Qanday ishlaydi:**

1. Compiler — Babel plugin (`babel-plugin-react-compiler`)
2. Build paytida har komponent funksiyasini AST darajasida tahlil qiladi
3. Qaysi qiymatlar har render'da o'zgarmasligini aniqlaydi
4. Avtomatik ravishda memoization qo'shadi
5. Generated kodda Compiler `useMemoCache` runtime helper'idan foydalanib har komponentga internal cache slot'lar qo'shadi (yakuniy semantika `useMemo`/`useCallback` bilan ekvivalent, lekin manual hook chaqiriqlari emas)

**Cheklovlar:**

- Komponent **"Rules of React"** ga amal qilishi shart (purity, no mutation, no side effects in render)
- `eslint-plugin-react-compiler` qoidalarni nazorat qiladi
- Mavjud kodga **incremental adoption** mumkin (faqat ba'zi komponentlarda yoqib ko'rish)

**Foydasi:**

- Performance code qo'lda yozish kerak emas (ko'p hollarda manual memoization noto'g'ri ishlatiladi yoki unutiladi)
- Kod tozaroq, o'qish oson
- Bug'larni kamaytiradi (manual `useMemo` deps array'ni unutish — keng tarqalgan xato)

> **Cross-ref:** React Compiler chuqur — `31-react-compiler.md` da. Manual memoization mexanizmi va kogda kerak bo'lishi — `21-usememo-usecallback.md` da.

---

## React Server Components Intro

### Nazariya

**React Server Components (RSC)** — komponentlar ikki turga bo'linadi:

| Tur | Qaerda render bo'ladi | JS browser'ga jo'natiladimi |
|-----|----------------------|------------------------------|
| **Server Component** | Server'da (Node.js, Deno, Bun) | YO'Q — faqat HTML va RSC payload |
| **Client Component** | Brauzerda (yoki SSR + hydration) | HA — bundle ichida |

**Asosiy g'oya:**

Komponent boshlanishida directive bilan turini belgilang:

```tsx
// "use server" — server function (action)
"use server";
async function saveUser(formData: FormData) {
  await db.users.create({ name: formData.get('name') });
}

// "use client" — client component
"use client";
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

// Default — Server Component (R19'da)
async function ProductList() {
  const products = await db.products.findMany(); // Server'da DB query
  return (
    <ul>
      {products.map(p => <li key={p.id}>{p.name}</li>)}
    </ul>
  );
}
```

**Foydasi:**

1. **Bundle size kichikroq** — server-only logic (DB query, file system, secrets) browser bundle'ga tushmaydi
2. **Performance** — server'da render qilingan HTML darhol ko'rsatiladi (no JS yuklanish kutmasligi)
3. **Direct backend access** — Server Component'larda `await db.query()` yozish mumkin (REST/GraphQL kerak emas)
4. **SEO** — server HTML to'liq tayyor, crawler'lar uchun yaxshi

**Cheklovlar:**

- **Framework majburiy** — vanilla React + Vite'da RSC ishlamaydi. Next.js (App Router), Remix, Waku kabi framework'lar kerak.
- **Server runtime** — Node.js, Deno yoki Bun
- **Yangi mental model** — Server vs Client farqi, serialization boundary, `"use client"` directive — yangi konsept o'rganish

**Server Actions** — RSC bilan kelgan ikkinchi katta xususiyat. Server'dagi funksiyani client'dan to'g'ridan-to'g'ri chaqirish:

```tsx
// Server Action
"use server";
async function deleteProduct(id: number) {
  await db.products.delete({ where: { id } });
}

// Client Component'dan chaqirish
"use client";
function DeleteButton({ id }: { id: number }) {
  return (
    <form action={() => deleteProduct(id)}>
      <button type="submit">O'chirish</button>
    </form>
  );
}
```

> **Cross-ref:** RSC va Server Actions chuqur — `39-rsc-server-actions.md` da (conceptual). To'liq implementatsiya — alohida `/next/` kursida bo'ladi.

---

## Edge Cases va Gotchas

### React komponent nomi katta harf bilan

```tsx
// ❌ Kichik harf — React HTML tag deb interpretatsiya qiladi
function userCard() { return <div />; }
<userCard />  // ← `<usercard>` HTML element yaratiladi, komponent EMAS

// ✅ Katta harf — React komponent deb tan oladi
function UserCard() { return <div />; }
<UserCard />  // ← UserCard funksiyasi chaqiriladi
```

**Sabab:** JSX transform qoidasi — Babel/SWC birinchi harf katta bo'lsa `_jsx(UserCard, {...})`, kichik bo'lsa `_jsx('userCard', {...})` deb yozadi (string sifatida HTML tag).

---

### Komponent funksiyasi pure bo'lishi shart

```tsx
let counter = 0;

// ❌ Render davomida tashqi state mutate qilish
function BadComponent() {
  counter++; // ← side effect render ichida
  return <div>{counter}</div>;
}

// ✅ Side effect uchun useEffect
function GoodComponent() {
  useEffect(() => {
    counter++; // ← effect ichida
  }, []);
  return <div>{counter}</div>;
}
```

**Sabab:** Strict Mode (R16.3+) development'da komponent funksiyasini ikki marta chaqiradi — render purity'ni tekshirish uchun. Side effect bo'lsa, ikki marta yuz beradi va bug aniq ko'rinadi. R18'dan boshlab effect'lar uchun ham ikki marta cycle qo'shildi (mount → cleanup → mount).

---

### `react-dom` va `react` versiyalari mos kelishi shart

```bash
# ❌ NOTO'G'RI — mismatched versions
npm install react@19 react-dom@18

# ✅ TO'G'RI — bir xil versiyalar
npm install react@19 react-dom@19
```

**Sabab:** `react-dom` `react`'ning ichki API'lariga bog'liq (private modules). Versiyalar mos kelmasa — runtime error yoki imkoniyatsiz xatti-harakat.

---

### "Library" emas "Framework"

React'ni "framework" deb chaqirish keng tarqalgan, ammo texnik jihatdan **library**. Bu farq quyidagi joylarda muhim:

- React **routing, build, server-side rendering, state management** ni o'z ichiga olmaydi
- Bu xizmatlarni **Next.js, Remix** kabi **frameworklar** taqdim etadi (React'ni o'z ichiga olib)
- React documentation'ida "library" termini ishlatiladi

---

### `import React from 'react'` (R17+) kerak emas

```tsx
// ❌ Eski (R16 va undan oldin) — JSX uchun majburiy edi
import React from 'react';
function App() { return <div />; }

// ✅ R17+ (JSX automatic transform default'da)
function App() { return <div />; }
// React import keraksiz — Babel/SWC `import { jsx } from 'react/jsx-runtime'` qo'shadi
```

**Sabab:** R17+ JSX automatic transform default'ga aylangan. Lekin agar eski tooling ishlatsangiz (eski Babel config) — hali kerak bo'lishi mumkin. Hozirgi Vite, Next.js, Create React App default'da automatic transform.

---

## Common Mistakes

### ❌ Xato 1: React'ni framework deb hisoblash

Yangi boshlovchilar React'dan "routing qayerda?", "data fetching qayerda?", "form validation qayerda?" deb so'rashadi. Bu — React'ning library ekanligini tushunmaslik.

**To'g'ri yondashuv:** React'ni faqat UI uchun deb qabul qiling. Routing — React Router, server — Next.js, state — Zustand/Redux, forms — React Hook Form. Har biri mustaqil tanlanadi.

---

### ❌ Xato 2: "Virtual DOM React'ni tez qiladi" deyish

Bu eskirgan iddao. React'ning samaraliligining sabablari bir nechta:

1. **Diffing algoritmi** (Reconciler) — faqat farqni DOM'ga yozadi
2. **Batched updates** — bir nechta `setState`'larni guruhlash
3. **Concurrent rendering** (R18+) — ish bo'laklarda, interruptible
4. **Event delegation** — har element'ga listener qo'shish o'rniga root container'da bitta

VDOM o'zi performance manbai emas — u faqat g'oya. Real performance Reconciler va Scheduler'dan keladi. Boshqa tomondan, SolidJS va Svelte kabi library'lar butunlay VDOMsiz, compile-time reactivity bilan ishlaydi — bu boshqa trade-off (tezroq mutation'lar, lekin reconciliation moslashuvchanligi kamroq).

---

### ❌ Xato 3: Class komponent yozish (Error Boundary'dan tashqari)

```tsx
// ❌ Modern React'da class kerak emas (faqat Error Boundary uchun)
class Counter extends React.Component {
  state = { count: 0 };
  render() {
    return (
      <button onClick={() => this.setState({ count: this.state.count + 1 })}>
        {this.state.count}
      </button>
    );
  }
}

// ✅ Function komponent — barcha imkoniyatlar mavjud
function Counter() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

**Sabab:** R16.8'dan beri Hooks bor — function komponentlarga state, lifecycle, context, refs hammasi mavjud. Class — faqat Error Boundary uchun majburan ishlatiladi (`getDerivedStateFromError`, `componentDidCatch`).

---

### ❌ Xato 4: TSX runtime'da JSX'dan farqli deb tushunish

```tsx
// TSX = JSX + .tsx extension + TypeScript types
// Runtime'da TSX va JSX bir xil _jsx() chaqiruvlariga aylanadi

// .jsx fayl:
function Button({ label, onClick }) { /* ... */ }

// .tsx fayl (turlar bilan):
interface ButtonProps { label: string; onClick: () => void; }
function Button({ label, onClick }: ButtonProps) { /* ... */ }

// Runtime'da ikkalasi ham bir xil _jsx('button', { ... }) chaqiradi.
// Farq faqat compile-time'da: TS compiler turlarni tekshiradi.
```

**Sabab:** TypeScript turlari **compile-time only** — production bundle'ga tushmaydi (`@babel/preset-typescript` yoki `tsc` type annotation'larni olib tashlaydi). TSX = JSX + qo'shimcha tip xavfsizligi qatlami; runtime semantikasi bir xil.

---

### ❌ Xato 5: `react` va `react-dom` versiyalari mos kelmasligi

```bash
# ❌ Bir xil emas — runtime xato
npm install react@19 react-dom@18

# ✅ Mos versiyalar
npm install react@19 react-dom@19
```

`package.json`:

```json
{
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  }
}
```

**Sabab:** `react-dom` `react`'ning **internal** modullariga bog'langan (`react/internals`, scheduler). Versiyalar mos kelmasa — `Cannot read property 'X' of undefined` kabi xatolar yoki silent bug'lar.

---

## Amaliy Mashqlar

### Mashq 1: Hello World (Oson)

`Greeting` komponenti yarating — `name: string` prop oladi va `<h1>Salom, {name}!</h1>` qaytaradi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
interface GreetingProps {
  name: string;
}

function Greeting({ name }: GreetingProps) {
  return <h1>Salom, {name}!</h1>;
}

// Ishlatish:
function App() {
  return <Greeting name="Ali" />;
}
```

</details>

---

### Mashq 2: Counter (Oson)

`Counter` komponentini yarating — bosilganda `count` ni 1 ga oshiradigan tugma.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Bosildi: {count}
    </button>
  );
}
```

</details>

---

### Mashq 3: Imperative → Declarative (O'rta)

Quyidagi imperative kodni React komponentga aylantiring:

```javascript
let isVisible = false;
const div = document.createElement('div');
div.textContent = 'Salom!';
div.style.display = 'none';

const button = document.createElement('button');
button.textContent = 'Toggle';
button.addEventListener('click', () => {
  isVisible = !isVisible;
  div.style.display = isVisible ? 'block' : 'none';
});

document.body.appendChild(button);
document.body.appendChild(div);
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState } from 'react';

function Toggle() {
  const [isVisible, setIsVisible] = useState(false);

  return (
    <>
      <button onClick={() => setIsVisible(!isVisible)}>
        Toggle
      </button>
      {isVisible && <div>Salom!</div>}
    </>
  );
}
```

E'tibor bering: `style.display` boshqarish o'rniga **conditional rendering** (`{isVisible && <div>...</div>}`) ishlatildi. Bu deklarativ usul — React DOM'ni o'zi qo'shadi/o'chiradi.

</details>

---

### Mashq 4: Props down, events up (O'rta)

`ToggleSwitch` komponenti yarating — `value: boolean` va `onChange: (value: boolean) => void` prop'larini oladi. Bosilganda `onChange(!value)` chaqiradi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
interface ToggleSwitchProps {
  value: boolean;
  onChange: (value: boolean) => void;
}

function ToggleSwitch({ value, onChange }: ToggleSwitchProps) {
  return (
    <button onClick={() => onChange(!value)}>
      {value ? 'Yoqilgan' : "O'chirilgan"}
    </button>
  );
}

// Ishlatish:
function Settings() {
  const [notifications, setNotifications] = useState(true);

  return (
    <div>
      <label>Bildirishnomalar:</label>
      <ToggleSwitch value={notifications} onChange={setNotifications} />
    </div>
  );
}
```

E'tibor bering: state `Settings` (parent)'da, `ToggleSwitch` (child) faqat ko'rsatadi va o'zgarish so'rovini parent'ga jo'natadi.

</details>

---

### Mashq 5: Component composition (O'rta)

`Card` komponenti yarating — `title: string` va `children: ReactNode` oladi. Quyidagi natija beradigan UI ko'rsating:

```
┌─────────────────────┐
│ Title               │
├─────────────────────┤
│ (children content)  │
└─────────────────────┘
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
import type { ReactNode } from 'react';

interface CardProps {
  title: string;
  children: ReactNode;
}

function Card({ title, children }: CardProps) {
  return (
    <article style={{ border: '1px solid #ccc', padding: '1rem' }}>
      <h3 style={{ borderBottom: '1px solid #ccc', paddingBottom: '0.5rem' }}>
        {title}
      </h3>
      <div>{children}</div>
    </article>
  );
}

// Ishlatish (composition):
function App() {
  return (
    <Card title="Foydalanuvchi ma'lumotlari">
      <p>Ism: Ali</p>
      <p>Email: ali@example.com</p>
      <button>Tahrirlash</button>
    </Card>
  );
}
```

`children` prop — har qanday JSX qabul qiladi (string, element, array, fragment). `ReactNode` — eng keng turdagi children type. Composition `11-composition.md` da chuqur yoritiladi.

</details>

---

## Xulosa

Bu bo'limda React'ning umumiy mental modeli o'rnatildi:

- **React = library, framework EMAS** — UI'ga fokuslangan, qolganini siz tanlaysiz
- **Declarative UI** — siz state'ni boshqarasiz, React DOM'ni o'zi yangilaydi (`UI = f(state)`)
- **Component-based** — composability, reusability, isolation
- **"Virtual DOM" — mental model**, **Fiber — implementation** (R16+)
- **One-way data flow** — props down, events up
- **Reconciler + Renderer ajratilgan** — bitta `react` paketi ko'p platform uchun ishlaydi
- **R18 — Concurrent Rendering**, **R19 — Actions, RSC, Compiler** asosiy yangiliklar

Bu mental model keyingi 38 ta bo'lim uchun asos bo'ladi. Har bir tushuncha — Fiber, hooks, Reconciliation, Scheduler — shu fundamental g'oyalar ustiga quriladi.

Keyingi bo'limda render jarayonining ichki qismi — `createRoot`, Render Phase va Commit Phase, batching, hydration intro — yoritiladi.

---

**Keyingi bo'lim:** [02-rendering.md](02-rendering.md) — Render va Commit Phases, `createRoot`, Strict Mode, R18 Automatic Batching, passive vs layout effects, Render/Commit pipeline mexanikasi, hydration intro.
