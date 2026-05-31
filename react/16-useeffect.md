# Bo'lim 16: useEffect

> `useEffect` — React component'larni tashqi tizimlar (network, browser API, timer, subscription, DOM) bilan sinxronlashtirish uchun mo'ljallangan hook. Bu bo'lim `useEffect`'ning rasmiy mental model'i ("synchronization mechanism", lifecycle EMAS), syntax, dependency array, cleanup, timing (passive flag, paint dan keyin), Strict Mode 2x effect cycle (R18+), race condition pattern'lari (AbortController), "You Might Not Need an Effect" anti-pattern'lari, va Under the Hood (effect list, `commitPassiveMountEffects`, MessageChannel scheduling) chuqur yoritiladi.

---

## Mundarija

- [`useEffect` "Lifecycle Hook" EMAS — Sync Mexanizmi](#useeffect-lifecycle-hook-emas--sync-mexanizmi)
- [Side Effects Tushunchasi](#side-effects-tushunchasi)
- [Syntax va Dependency Array](#syntax-va-dependency-array)
- [Cleanup Function](#cleanup-function)
- [Effect Timing — Passive vs Layout](#effect-timing--passive-vs-layout)
- [Effect Ordering — Bottom-Up Execution](#effect-ordering--bottom-up-execution)
- [Use Cases — Real-World Pattern'lar](#use-cases--real-world-patternlar)
- [Race Conditions va `AbortController`](#race-conditions-va-abortcontroller)
- [Stale Closure va Missing Deps](#stale-closure-va-missing-deps)
- [Object/Array Deps — Referential Identity](#objectarray-deps--referential-identity)
- [Strict Mode 2x Effect Cycle (R18+)](#strict-mode-2x-effect-cycle-r18)
- ["You Might Not Need an Effect"](#you-might-not-need-an-effect)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## `useEffect` "Lifecycle Hook" EMAS — Sync Mexanizmi

### Nazariya

Bu bo'limning eng muhim section'i — `useEffect` haqidagi noto'g'ri mental model React kodning aksariyat bug'larining sababidir. Avvalo terminologiya aniqlanadi:

| Termin | Manbai | Ma'no |
|--------|--------|-------|
| **lifecycle methods** | Class component'lar (`componentDidMount`, `componentDidUpdate`, `componentWillUnmount`) | Class instance'ning hayot davriga bog'langan imperativ method'lar |
| **lifecycle hooks** | Vue.js (`mounted`, `updated`, `beforeDestroy`) | Vue'ning rasmiy terminologiyasi |
| **Effects** | React rasmiy termini | "Effect" — render natijasini tashqi tizim bilan **sinxronlashtirish** |

React rasmiy hujjatlari (`react.dev` "Synchronizing with Effects" sahifasi) aniq ifodalaydi: *"Effects are NOT lifecycle events — they are synchronization mechanisms."* Bu shunchaki so'z tanlash farqi emas — bu **mental model** farqi.

**Mental model 1 — Lifecycle (XATO):**

> "Component mount qilganda X qil, update qilganda Y qil, unmount qilganda Z qil."

Bu yondashuv imperative — har lifecycle hodisasiga alohida qadam yoziladi. Class component'larda shunday bo'lgan, va aksariyat developer'lar `useEffect`'ni shu modelda tushunadi:

```tsx
// ❌ XATO mental model — lifecycle yondashuvi
useEffect(() => {
  console.log('mount qilindi');           // componentDidMount
  return () => console.log('unmount');    // componentWillUnmount
}, []);

useEffect(() => {
  console.log('update qilindi', userId);  // componentDidUpdate
}, [userId]);
```

Bu kod ishlaydi, lekin yondashuv noto'g'ri. Sabab keyingi paragraflarda ko'rsatiladi.

**Mental model 2 — Synchronization (TO'G'RI):**

> "Joriy state berilgan bo'lsa, tashqi tizim qanday holatda bo'lishi kerak? Effect shu holatga keltiradi."

Bu yondashuv declarative — siz tashqi tizimning **bir vaqtdagi holati**ni tasvirlaysiz. React esa state o'zgarganda effect'ni qaytadan ishlatib, tashqi tizimni joriy state bilan sync'ga keltiradi:

```tsx
// ✅ TO'G'RI mental model — sync yondashuvi
useEffect(() => {
  // "userId berilgan bo'lsa, shu user uchun chat connection ochilsin"
  const connection = createConnection(userId);
  connection.connect();
  
  return () => {
    // "Sync'dan chiqish — userId o'zgarganda yoki component unmount bo'lganda"
    connection.disconnect();
  };
}, [userId]);
```

Bu kodda "mount" yoki "update" so'zi yo'q. Faqat: **"userId berilgan bo'lsa, connection shu user'ga ochiq bo'lishi kerak"** — bu invariant. React qanday va qachon effect chaqirilishini o'zi boshqaradi.

**Sync invariant'i:**

Effect declarative aytishni anglatadi:

> "Joriy state qiymati X bo'lsa, tashqi tizim Y holatda bo'lishi kerak."

Effect callback bu holatni **o'rnatadi** (setup), cleanup function uni **bekor qiladi** (teardown). Effect har safar deps o'zgarganda qaytadan ishlaydi: avval cleanup, keyin setup. Bu — "tashqi tizim joriy state bilan sinxron bo'lib turadi" degan ma'noni beradi.

Bu mental model asosida qator dizayn qarorlari tabiiy bo'ladi:

1. **Dependency array** — sync uchun qaysi state'lar muhim. Hech qaysisi `null` bo'lmaydi: yo har render, yo `[]` (faqat bir marta), yo aniq deps.
2. **Strict Mode 2x effect cycle (R18+)** — effect "qaytadan o'rnatilishga chidamli" bo'lishi kerak. Agar mount→unmount→mount cycle effect'ni buzsa, demak cleanup to'liq emas.
3. **"You Might Not Need an Effect"** anti-pattern'lari — agar aslida sync yo'q (state'dan derivatsiya, parent'ga xabar berish) — `useEffect` o'rinsiz.

**Class lifecycle bilan mexanik moslashuv — anti-pattern:**

Interview savoli: *"`useEffect` qaysi lifecycle method'larga teng?"* — savol noto'g'ri qo'yilgan. Mexanik javob:

| Class lifecycle | "Teng" `useEffect` |
|-----------------|-------------------|
| `componentDidMount` | `useEffect(() => {...}, [])` |
| `componentDidUpdate` | `useEffect(() => {...}, [deps])` |
| `componentWillUnmount` | `useEffect(() => () => {...}, [])` cleanup |

Bu jadval **noto'g'ri yo'naltiruvchi**. Aslida `useEffect` lifecycle method'ning almashtiruvi emas — u yondashuvni o'zgartiradi. To'g'ri javob:

> "`useEffect` lifecycle method'larga teng emas. U sync mexanizmi — joriy state berilgan bo'lsa, tashqi tizim qanday holatda bo'lishi kerakligini tasvirlaydi. Class lifecycle method'lar imperative qadamlar edi — har bir hodisaga alohida method. `useEffect` declarative — bir invariant, React uni ushlab turadi."

**Nima uchun bu farq amaliyotda muhim:**

Quyidagi misol — lifecycle yondashuvi ishlaydi, lekin sync yondashuvi yashirin bug'ni topishga yordam beradi:

```tsx
// Lifecycle yondashuvi — userId o'zgarsa, connection o'zgaradimi?
function ChatRoom({ userId }: { userId: string }) {
  useEffect(() => {
    const connection = createConnection(userId);
    connection.connect();
    
    return () => connection.disconnect();
  }, []);  // ❌ deps bo'sh — userId o'zgarsa effect qayta ishlamaydi
  
  return <div>Chat for {userId}</div>;
}
```

Lifecycle mental model'da bu kod tabiiy ko'rinadi: "mount paytida connect qil, unmount paytida disconnect qil." `userId` o'zgarganda nima bo'lishi haqida o'ylanmaydi.

Sync mental model'da xato darrov ko'rinadi: "joriy `userId` uchun connection ochiq bo'lishi kerak — `userId` o'zgarganda eski connection yopilib, yangisi ochilishi kerak." Demak `userId` deps array'da bo'lishi shart:

```tsx
// ✅ Sync yondashuvi — userId deps'da
function ChatRoom({ userId }: { userId: string }) {
  useEffect(() => {
    const connection = createConnection(userId);
    connection.connect();
    
    return () => connection.disconnect();
  }, [userId]);  // ✅ userId o'zgarsa: cleanup → setup
  
  return <div>Chat for {userId}</div>;
}
```

ESLint rule `react-hooks/exhaustive-deps` bu xatoni avtomatik topadi (cross-ref Section "Stale Closure va Missing Deps"). Lekin rule sabab emas, **oqibat** — sync invariant'i buzilgan.

<details>
<summary><strong>Under the Hood</strong></summary>

React jamoasi (Dan Abramov, Andrew Clark) `useEffect`'ning sync modeliga o'tishni 2018-2019 yillarda hookslarni dizayn qilish jarayonida qabul qilgan. Sabablar texnik:

**1. Concurrent Mode bilan moslik:**

R18 dan boshlab Concurrent Mode standart. Concurrent rendering paytida React render'ni to'xtatib, qaytadan boshlashi yoki tashlab yuborishi mumkin. Pre-mount lifecycle methods bu modelga mos kelmaydi — Render Phase'da chaqirilgani uchun bir necha marta yoki tashlab yuborish bilan ishlamaydi:

- `componentWillMount` — R16.3'da legacy nom deprecated qilindi (rasmiy olib tashlash sanasi yo'q, `UNSAFE_componentWillMount` alias hali ishlaydi)
- `componentWillReceiveProps` — R16.3'da legacy nom deprecated (`UNSAFE_componentWillReceiveProps` alias)
- `componentWillUpdate` — R16.3'da legacy nom deprecated (`UNSAFE_componentWillUpdate` alias)

Bu method'lar Render Phase'da chaqirilgani uchun Concurrent rendering bilan mos kelmaydi (render tashlab yuborilishi yoki qaytadan boshlanishi mumkin). React jamoasi `getDerivedStateFromProps` (static, side-effect-free) va `getSnapshotBeforeUpdate` (Commit'dan oldin) bilan almashtirdi.

`useEffect` bu muammolarni hal qiladi — Render Phase'dan keyin (Commit Phase'dan keyin) chaqiriladi, render qaytarilsa effect run qilinmaydi.

**2. Strict Mode 2x effect (R18+) — sync invariant'ini test qilish:**

Strict Mode'da React har effect'ni mount, unmount, va yana mount qilib chaqiradi. Sabab: `Activity` (avval `Offscreen`) komponentlar (kelajakdagi feature) yoki Fast Refresh holatlarida effect remount bo'lishi mumkin. Agar effect "lifecycle" mental model'da yozilgan bo'lsa, remount uni buzadi. Sync mental model'da effect cleanup to'liq bo'lishi shart, demak remount muammosiz o'tadi.

**3. Effect — render natijasi:**

React'ning fundamental modeli: UI = `f(state)`. Effect ham shu funksiyaning bir qismi: tashqi tizim holati = `f(state)`. Render JSX'ni hisoblaydi (DOM uchun), effect tashqi tizim syncsini hisoblaydi (network, timer, subscription uchun).

**Source citation:**

Dan Abramov'ning "A Complete Guide to useEffect" (2019, overreacted.io) postida bu fikr aniq aytilgan: *"useEffect is closer to data flow than to lifecycle."* React official docs'ning "Synchronizing with Effects" (2023) sahifasi shu fikrni rasmiylashtiradi.

Bu fikr falsafa emas — texnik zaruriyat. Concurrent rendering, Strict Mode 2x effect, kelajakdagi `Activity` (avval `Offscreen`) API — barchasi sync invariant'ini taxmin qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — bir xil masala, ikki yondashuv:**

```tsx
// Talab: WebSocket ulanmasi user'ning aktivligi (online/offline) ga moslash kerak

// ❌ Lifecycle yondashuvi — har holat uchun alohida code path
function UserStatus({ userId, isOnline }: { userId: string; isOnline: boolean }) {
  useEffect(() => {
    // Mount paytida ulanish
    const ws = new WebSocket(`wss://api.example.com/${userId}`);
    
    return () => ws.close();
  }, [userId]);
  
  useEffect(() => {
    // Update paytida online status yangilash — ALOHIDA effect
    if (isOnline) {
      // ❌ Lekin yuqoridagi ws ga kirish yo'q — closure trap
      console.log('online'); 
    }
  }, [isOnline]);
}

// ✅ Sync yondashuvi — bir invariant
function UserStatus({ userId, isOnline }: { userId: string; isOnline: boolean }) {
  useEffect(() => {
    // "Berilgan userId va isOnline uchun WebSocket holati shunday bo'lishi kerak"
    const ws = new WebSocket(`wss://api.example.com/${userId}`);
    ws.onopen = () => ws.send(JSON.stringify({ type: 'status', isOnline }));
    
    return () => ws.close();
  }, [userId, isOnline]);  // Har ikki deps — holat invariant'ining qismi
}
```

**Misol 2 — sync invariant'i bilan o'ylash:**

```tsx
// Talab: search query o'zgarganda URL params ga sync qilish

// ✅ Sync — "URL search params query'ga teng bo'lishi kerak"
function SearchPage({ query }: { query: string }) {
  useEffect(() => {
    const url = new URL(window.location.href);
    url.searchParams.set('q', query);
    window.history.replaceState({}, '', url);
    
    // Cleanup kerak emas — URL state ham keyingi sync'da yangilanadi
  }, [query]);
}
```

Cleanup yo'qligi sync yondashuvida tabiiy: keyingi effect run avvalgi URL'ni qaytadan o'rnatadi. State o'rniga URL — bir o'zgaruvchan resurs.

**Misol 3 — class lifecycle dan o'tkazish (xato):**

```tsx
// Class component
class Counter extends React.Component {
  intervalId = 0;
  
  componentDidMount() {
    this.intervalId = setInterval(() => this.tick(), 1000);
  }
  
  componentWillUnmount() {
    clearInterval(this.intervalId);
  }
  
  // componentDidUpdate yo'q — interval state'dan mustaqil
}

// ❌ XATO — Class lifecycle ni mexanik o'tkazish
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    // "Mount paytida interval boshlash"
    const id = setInterval(() => setCount(c => c + 1), 1000);
    return () => clearInterval(id);
  }, []);  // ❌ Lekin nima uchun [] — sabab tushunilmagan
}

// ✅ TO'G'RI — sync invariant'i: "interval doim aktiv bo'lishi kerak"
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    // "Component'ning hayoti davomida interval aktiv"
    const id = setInterval(() => setCount(c => c + 1), 1000);
    return () => clearInterval(id);
  }, []);  // ✅ deps yo'q — interval state'dan mustaqil
}
```

Ikkala kod bir xil ishlaydi, lekin to'g'ri yondashuvda `[]`'ning ma'nosi: "interval invariant'i state'ga bog'liq emas." Xato yondashuvda `[]` shunchaki "mount paytida bir marta" — sabab yo'q.

</details>

---

## Side Effects Tushunchasi

### Nazariya

`useEffect`'ning ismi "side effect" tushunchasiga ishora qiladi. Bu termin functional programming'dan keladi.

**Pure function** — bir xil input uchun har doim bir xil output qaytaradigan, va tashqi dunyoga ta'sir qilmaydigan funksiya. Pure function:

- Faqat o'z parametrlariga qarab natija hisoblaydi
- Hech qanday tashqi state'ni o'zgartirmaydi
- Hech qanday tashqi tizimni chaqirmaydi (network, console, DOM, fayl)

**Side effect** — pure function ta'rifidan tashqari narsa: tashqi state'ga yozish, network so'rovi, DOM manipulation, console log, timer o'rnatish, subscription.

React component'ning **render funksiyasi pure bo'lishi shart**. Bu Render Purity Rule (cross-ref [`09-component-basics.md`](09-component-basics.md)). Render paytida quyidagilar TAQIQLANADI:

```tsx
function UserProfile({ userId }: { userId: string }) {
  // ❌ Side effect render paytida — TAQIQ
  document.title = `User ${userId}`;        // DOM mutation
  fetch(`/api/users/${userId}`);            // Network
  localStorage.setItem('user', userId);     // Storage mutation
  console.log('rendering');                 // Side effect (debug uchun OK lekin)
  
  return <div>{userId}</div>;
}
```

Sabab: React render'ni Concurrent Mode'da to'xtatishi, qaytadan boshlashi yoki tashlab yuborishi mumkin. Agar render side effect qilsa, ular bir necha marta yoki noto'g'ri vaqtda bajariladi → bug, race condition, performance muammosi.

**`useEffect` — side effect uchun "boshpana":**

`useEffect` callback render'dan tashqarida (Commit Phase'dan keyin) chaqiriladi. Demak side effect'lar `useEffect` ichida xavfsiz:

```tsx
function UserProfile({ userId }: { userId: string }) {
  // ✅ Side effect useEffect ichida — render'dan tashqarida
  useEffect(() => {
    document.title = `User ${userId}`;
    return () => { document.title = 'App'; };
  }, [userId]);
  
  return <div>{userId}</div>;
}
```

**Side effect turlari va ularga mos React API:**

Hamma side effect'lar uchun `useEffect` zaruriy emas — React boshqa mexanizmlarni taklif qiladi:

| Side effect turi | Tavsiya etilgan mexanizm |
|------------------|--------------------------|
| User action javobi (click, submit) | Event handler (`onClick`, `onSubmit`) — `useEffect` emas |
| External system sync (network, timer, subscription) | `useEffect` |
| Sync DOM read (measure, scroll) | `useLayoutEffect` (cross-ref [`17-uselayouteffect.md`](17-uselayouteffect.md)) |
| External store subscription | `useSyncExternalStore` (R18+) |
| State'dan derivatsiya | Render paytida hisoblash, `useMemo` |
| Reset state | `key` prop |

`useEffect`'ni "har joyda ishlatish" — anti-pattern. Faqat tashqi tizim bilan sync kerak bo'lganda ishlatiladi (cross-ref Section "You Might Not Need an Effect").

**Render purity ni saqlash sabab'lari:**

1. **Concurrent rendering** — render qaytadan boshlanishi mumkin
2. **Strict Mode** — Development'da render 2x chaqiriladi (R16.3+) test uchun
3. **Memoization** — `React.memo`, `useMemo` purity'ga tayanadi
4. **DevTools Profiler** — render hisoblash uchun
5. **Server Components** — render server'da bo'ladi; `useEffect` umuman ishlamaydi (effect faqat client'da chaqiriladi), DOM/browser API yo'q. Server'da `async/await` bilan to'g'ridan-to'g'ri data fetch qilinadi — `useEffect`'siz

<details>
<summary><strong>Under the Hood</strong></summary>

**React'ning render purity invariant'ini qanday majburlaydi:**

Strict Mode'da React har component'ning render funksiyasini ikki marta chaqiradi (development'da). Bu hidden side effect'larni topish uchun:

```tsx
// Strict Mode'da render 2x chaqiriladi
let renderCount = 0;
function Component() {
  renderCount++;  // ❌ Tashqi state mutation
  console.log(renderCount);  // Output: 1, 2, 3, 4 (2x har real render)
  return <div>{renderCount}</div>;
}
```

Agar render funksiyasida side effect bo'lsa — Strict Mode'da darrov ko'rinadi (raqamlar 2x ko'p). Bu intentional dev tool — bug'larni topishga yordam beradi.

**Concurrent rendering misoli:**

```tsx
// Concurrent rendering'da render to'xtatilishi mumkin
function ExpensiveList({ data }: { data: Item[] }) {
  fetch('/api/log', {           // ❌ Render paytida network
    method: 'POST',
    body: JSON.stringify({ data })
  });
  
  return <ul>{data.map(...)}</ul>;
}

// Agar React bu render'ni tashlab yuborsa (Suspense, transition) —
// fetch bekor qilinmaydi, server'ga keraksiz so'rov ketadi
```

`useEffect` ichida bo'lsa, fetch faqat **commit'dan keyin** chaqiriladi — ya'ni render natija real'da DOM'ga qo'llanilganda. Tashlangan render'lar effect'ni chaqirmaydi.

**Effect — pure function emas, lekin invariant ifodalaydi:**

Effect callback "side effect" bo'lsa-da, uni ifodalash declarative: deps va invariant'ga asosan, React effect'ni qachon va necha marta chaqirishni hal qiladi. Developer faqat invariant'ni yozadi.

Bu fikr quyidagi paradoksni hal qiladi: side effect'lar imperative tabiatga ega, lekin React'ning declarative paradigmasi bilan moslashishi kerak. `useEffect` bu ko'prikni quradi — siz nima sinxronlashtirishni declarative aytasiz, React qachon va qanday chaqirishni boshqaradi.

**Reference:**

ECMAScript jihatdan funksiyalar pure/impure bo'lib ajratilmaydi (TypeScript'da ham yo'q, lekin `Readonly<T>`, `as const` yordamchi). React bu invariant'ni runtime'da Strict Mode bilan, va eslint qoidalari (`react-hooks/exhaustive-deps`, `react-hooks/rules-of-hooks`) bilan majburlashga harakat qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — render'da side effect (xato) → `useEffect` (to'g'ri):**

```tsx
// ❌ Render paytida side effect
function ProductPage({ productId }: { productId: string }) {
  // Render har gal log yuboradi — Strict Mode'da 2x, Concurrent re-render'da N x
  analytics.trackPageView({ page: 'product', id: productId });
  
  return <div>Product {productId}</div>;
}

// ✅ useEffect ichida — bir marta commit'dan keyin
function ProductPage({ productId }: { productId: string }) {
  useEffect(() => {
    analytics.trackPageView({ page: 'product', id: productId });
  }, [productId]);  // productId o'zgarganda yana track
  
  return <div>Product {productId}</div>;
}
```

**Misol 2 — boshqa mexanizm yaxshiroq (event handler):**

```tsx
// ❌ useEffect notification yuborish uchun
function CommentForm({ onSubmit }: { onSubmit: (text: string) => void }) {
  const [text, setText] = useState('');
  const [submitted, setSubmitted] = useState(false);
  
  useEffect(() => {
    if (submitted) {
      onSubmit(text);                       // ❌ Side effect chain
      sendNotification('Comment posted');   // ❌ Effect ichida emas
      setSubmitted(false);                  // ❌ Yana render
    }
  }, [submitted, text, onSubmit]);
  
  const handleSubmit = () => setSubmitted(true);
  return <form onSubmit={handleSubmit}>...</form>;
}

// ✅ Event handler ichida — user action javobi
function CommentForm({ onSubmit }: { onSubmit: (text: string) => void }) {
  const [text, setText] = useState('');
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    onSubmit(text);                         // ✅ User action javobi
    sendNotification('Comment posted');     // ✅ Bir vaqtda
    setText('');
  };
  
  return <form onSubmit={handleSubmit}>...</form>;
}
```

User action (form submit) — `useEffect` uchun emas. Event handler ishlatiladi. Sync yo'q — har submit bir martalik hodisa.

**Misol 3 — derived state — `useEffect` shart emas:**

```tsx
// ❌ useEffect derived state uchun
function FullName({ firstName, lastName }: { firstName: string; lastName: string }) {
  const [fullName, setFullName] = useState('');
  
  useEffect(() => {
    setFullName(`${firstName} ${lastName}`);  // ❌ Render → effect → render
  }, [firstName, lastName]);
  
  return <div>{fullName}</div>;
}

// ✅ Render paytida hisoblash
function FullName({ firstName, lastName }: { firstName: string; lastName: string }) {
  const fullName = `${firstName} ${lastName}`;  // ✅ Pure derivation
  return <div>{fullName}</div>;
}
```

Derived state uchun `useEffect` keraksiz — render paytida pure hisoblash bilan amalga oshiriladi.

</details>

---

## Syntax va Dependency Array

### Nazariya

`useEffect` signature:

```tsx
function useEffect(
  setup: () => void | (() => void),
  dependencies?: ReadonlyArray<unknown>
): void;
```

- **`setup`** — effect callback. `void` yoki cleanup function qaytaradi.
- **`dependencies`** (optional) — effect qachon qaytadan run qilishni belgilaydigan qiymatlar massivi.

Dependency array uchta variantga ega, har biri turli xulq-atvor:

**Variant 1 — Deps array berilmaydi (`undefined`):**

```tsx
useEffect(() => {
  console.log('Har render'da');
});  // Deps yo'q
```

Effect **har render'dan keyin** chaqiriladi. Boshlang'ich render'da, har state/prop o'zgarishida.

Bu variant juda kam ishlatiladi. Sync nuqtai nazaridan ma'noli emas — har render'da remount qilish maxsus holatlarga zarur.

**Variant 2 — Bo'sh deps array (`[]`):**

```tsx
useEffect(() => {
  console.log('Faqat mount paytida');
}, []);
```

Effect **faqat boshlang'ich mount'dan keyin** chaqiriladi (Strict Mode'da R18+ — mount → unmount → mount). Cleanup esa unmount paytida.

Bu variant — "tashqi tizim bilan one-time setup, component hayoti davomida sinxron" holatlar uchun. Misollar: subscription, interval, event listener (window-level).

**Variant 3 — Aniq deps (`[a, b, c]`):**

```tsx
useEffect(() => {
  fetchUser(userId);
}, [userId]);
```

Effect **har deps array'dagi qiymat o'zgarganda** chaqiriladi (har gal cleanup → setup). Boshlang'ich mount'da ham chaqiriladi.

Comparison `Object.is` orqali. Reference equality, primitive value equality.

**Object.is comparison:**

React deps qiymatlarini `Object.is(prevDep, nextDep)` orqali solishtiradi. Bu `===` ga o'xshash, lekin ikkita farq bilan:

```ts
Object.is(NaN, NaN);     // true   ← === false bo'lardi
Object.is(0, -0);        // false  ← === true bo'lardi
Object.is({}, {});       // false  (har xil reference)
Object.is([1], [1]);     // false  (har xil reference)
```

Bu Object/Array deps'ning ko'p uchraydigan trapi (cross-ref Section "Object/Array Deps").

**Linter — `react-hooks/exhaustive-deps`:**

ESLint rule `react-hooks/exhaustive-deps` (rasmiy `eslint-plugin-react-hooks` paketida) deps array to'g'riligini tekshiradi. Effect callback ichida ishlatilgan har bir reactive value (props, state, hook qaytaradigan qiymatlar) deps'da bo'lishi shart:

```tsx
function UserGreeting({ name }: { name: string }) {
  const [greeting, setGreeting] = useState('Hello');
  
  useEffect(() => {
    document.title = `${greeting}, ${name}!`;
  }, [name]);  // ⚠️ Lint warning — greeting deps'da yo'q
}
```

Linter bu xatoni topadi. Ignorelash imkoniyati bor (`// eslint-disable-next-line`), lekin bu deyarli har doim bug yashiradi (cross-ref Section "Stale Closure").

**Reactive vs non-reactive values:**

- **Reactive** — props, state, context, hooks qaytaradigan qiymatlar. O'zgarishi component re-render keltirib chiqaradi → deps'da bo'lishi shart.
- **Non-reactive** — module-level variable'lar, ref.current (`useRef`'ning value). O'zgarishi re-render keltirmaydi → deps'da bo'lishi shart emas (lekin ESLint hozirda ref'larni "reactive emas" deb biladi).

```tsx
const ENV = 'production';  // Module-level — non-reactive

function Component() {
  const ref = useRef(0);  // ref.current — non-reactive
  
  useEffect(() => {
    console.log(ENV);          // ✅ Deps'da kerak emas
    console.log(ref.current);  // ✅ Deps'da kerak emas (lekin closure'ni tushunish kerak)
  }, []);
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

React fiber paytida deps comparison `areHookInputsEqual` funksiyasi orqali bajariladi. Source code (soddalashtirilgan):

```ts
function areHookInputsEqual(
  nextDeps: Array<unknown>,
  prevDeps: Array<unknown> | null,
): boolean {
  if (prevDeps === null) {
    // Birinchi render — deps har doim "o'zgargan" hisoblanadi
    return false;
  }
  
  if (nextDeps.length !== prevDeps.length) {
    // Deps array uzunligi o'zgarsa — bug (development'da warning)
    console.error('Hook deps length changed');
  }
  
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (Object.is(nextDeps[i], prevDeps[i])) {
      continue;  // Bir xil — keyingi
    }
    return false;  // O'zgargan — effect run qilish kerak
  }
  
  return true;  // Hammasi bir xil — effect skip
}
```

**Update path — `updateEffectImpl`:**

```ts
function updateEffectImpl(
  fiberFlags: Flags,
  hookFlags: HookFlags,
  create: () => () => void,
  deps: Array<unknown> | null,
): void {
  const hook = updateWorkInProgressHook();  // Avvalgi hook'ni topish
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;
  
  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;  // Avvalgi cleanup
    
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // Deps bir xil — effect skip, lekin hook chain'da saqlanadi
        hook.memoizedState = pushEffect(hookFlags, create, destroy, nextDeps);
        return;
      }
    }
  }
  
  // Deps o'zgargan — effect run kerak
  currentlyRenderingFiber!.flags |= fiberFlags;
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,  // HookHasEffect bayrog'i o'rnatildi
    create,
    destroy,
    nextDeps
  );
}
```

`HookHasEffect` bayrog'i — Commit Phase paytida React qaysi effect'larni run qilishni aniqlash uchun. Deps o'zgarmagan effect'lar `HookHasEffect`'siz qoladi → skip.

**Linter implementation:**

`eslint-plugin-react-hooks` `react-hooks/exhaustive-deps` rule'i AST'ni analiz qiladi:

1. `useEffect(callback, deps)` chaqiruvini topish
2. `callback` ichida ishlatilgan barcha identifier'larni yig'ish
3. Identifier `props`, state, context, hook return value ekanligini aniqlash (reactive)
4. Deps array bilan solishtirish
5. Mos kelmasa — warning

Linter `// eslint-disable` bilan ignorelanishi mumkin, lekin bu deyarli har doim bug yashiradi. Rasmiy tavsiya: warning'ni hal qilish, ignorelash emas.

**Source citation:**

`react-hooks/exhaustive-deps` rule reference: facebook/react repo, `packages/eslint-plugin-react-hooks/src/ExhaustiveDeps.js`.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — uchchala variant:**

```tsx
function Component({ userId }: { userId: string }) {
  // Variant 1 — har render'da
  useEffect(() => {
    console.log('Har render');
  });
  
  // Variant 2 — bir marta mount'da
  useEffect(() => {
    console.log('Mount');
    return () => console.log('Unmount');
  }, []);
  
  // Variant 3 — userId o'zgarganda
  useEffect(() => {
    console.log('userId changed:', userId);
    return () => console.log('userId cleanup:', userId);
  }, [userId]);
  
  return null;
}
```

**Misol 2 — Object.is gotchas:**

```tsx
function Component() {
  const [count, setCount] = useState(0);
  
  // NaN — useEffect re-run qilmaydi
  useEffect(() => {
    console.log('NaN effect');
  }, [NaN]);  // Object.is(NaN, NaN) === true → har render'da bir xil → skip
  
  // 0 vs -0 — useEffect re-run qiladi
  useEffect(() => {
    console.log('Zero effect');
  }, [0 * count, -0 * count]);  // ⚠️ Object.is(0, -0) === false
  
  return <button onClick={() => setCount(c => c + 1)}>+</button>;
}
```

`-0` deps'da kamdan-kam holatda, lekin ba'zi matematik hisob'larda kelib chiqishi mumkin (e.g., `Math.sign`).

**Misol 3 — exhaustive-deps linter:**

```tsx
function UserCard({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [theme, setTheme] = useState<'dark' | 'light'>('dark');
  
  // ⚠️ Linter warning: 'theme' deps'da yo'q
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(data => {
        // theme ishlatilgan — deps'da bo'lishi kerak
        setUser({ ...data, theme });
      });
  }, [userId]);  // ❌ theme yo'q
  
  // ✅ To'g'ri — barcha reactive deps
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(data => setUser({ ...data, theme }));
  }, [userId, theme]);  // ✅
  
  // ⚠️ Lekin bu nooptimal — theme o'zgarsa fetch qaytadan
  // Yaxshi yechim: theme'ni fetch'dan ajratish (boshqa effect yoki render time'da merge)
  
  return <div>{user?.name}</div>;
}
```

Linter to'g'ri ko'rsatadi: agar `theme` ishlatilgan bo'lsa, deps'da bo'lishi shart. Lekin bu fetch'ni har `theme` o'zgarganda qaytadan keltirib chiqaradi — sync nuqtai nazaridan to'g'ri (effect joriy holat bilan moslashadi). Performance kerak bo'lsa — `theme`'ni fetch'dan ajratish kerak (alohida effect yoki render time merge).

</details>

---

## Cleanup Function

### Nazariya

Effect callback `void` yoki **cleanup function** qaytarishi mumkin. Cleanup function React tomonidan ikki holatda chaqiriladi:

1. **Effect qaytadan run bo'lishidan oldin** — deps o'zgarganda. Avval avvalgi effect'ning cleanup'i, keyin yangi effect setup.
2. **Component unmount bo'lganda** — Fiber tree'dan o'chirilganda.

Bu — sync invariant'ining "teardown" qismi. Setup tashqi tizimni joriy state bilan moslaydi, cleanup eski moslashuvni bekor qiladi.

**Syntax:**

```tsx
useEffect(() => {
  // Setup
  const subscription = source.subscribe(handler);
  
  // Cleanup — qaytariladi
  return () => {
    subscription.unsubscribe();
  };
}, [source]);
```

Cleanup — har qanday callable (arrow function, function declaration, named function). React `typeof destroy === 'function'` tekshiradi va chaqiradi. Setup callback'idan **`undefined` yoki `void`** qaytsa — cleanup yo'q. Funksiya bo'lmagan qiymat (object, Promise) qaytarish — bug (Promise gotcha section'da yoritiladi).

**Cleanup tartib:**

Deps o'zgarganda quyidagi tartib:

```
1. Avvalgi effect'ning cleanup() chaqiriladi
2. Yangi setup() chaqiriladi
3. Yangi cleanup return qilinadi (saqlanadi keyingi cycle uchun)
```

Component unmount paytida:

```
1. Oxirgi effect'ning cleanup() chaqiriladi
2. Setup chaqirilmaydi
```

**Closure traps cleanup ichida:**

Cleanup function — setup function bilan **bir xil closure**'ga ega. Bu kuchli xususiyat: cleanup setup paytida yaratilgan resurslarga (subscription, timer ID) kirishi mumkin:

```tsx
useEffect(() => {
  const id = setInterval(() => console.log('tick'), 1000);
  
  return () => {
    clearInterval(id);  // ✅ id closure ichida
  };
}, []);
```

Lekin closure'ning quyi muammosi: cleanup avvalgi render'ning state'ini ko'radi:

```tsx
function Component({ userId }: { userId: string }) {
  useEffect(() => {
    console.log('Setup:', userId);
    
    return () => {
      console.log('Cleanup:', userId);  // Avvalgi userId qiymati
    };
  }, [userId]);
}

// Render 1: userId='alice'  → Setup: alice
// Render 2: userId='bob'    → Cleanup: alice, Setup: bob
// Render 3: userId='charlie' → Cleanup: bob, Setup: charlie
```

Cleanup'da `userId` qiymati **avvalgi** setup'dan. Bu sync modeli bilan to'g'ri — cleanup avvalgi sync'ni bekor qiladi (avvalgi `userId` uchun).

**Cleanup MAJBURIY emas, lekin tavsiya etiladi:**

Agar effect tashqi resurs yaratmasa (timer, subscription, listener), cleanup kerak emas:

```tsx
// Cleanup kerak emas — DOM read faqat
useEffect(() => {
  const width = window.innerWidth;
  console.log('Width:', width);
}, []);
```

Lekin tashqi resurs yaratilganda cleanup **shart**. Yo'qsa memory leak, double subscription, ghost listener'lar paydo bo'ladi.

**Symmetric (idempotent) cleanup:**

Cleanup setup'ni to'liq teskari qilishi shart: setup nima yaratgan bo'lsa, cleanup aynan o'shani bekor qiladi. Bu xususiyat zaruriy, chunki React effect'ni bir necha marta setup → cleanup → setup qilishi mumkin. Strict Mode'da R18+ effect cycle aynan shuni keltirib chiqaradi: mount paytida setup, so'ng cleanup, so'ng yana setup. Agar cleanup setup'ni to'liq kompensatsiya qilsa, bu cycle tashqi tizimni buzmaydi:

```tsx
let count = 0;
useEffect(() => {
  count++;
  console.log('Setup count:', count);
  
  return () => {
    count--;  // ✅ Idempotent — har cleanup count'ni 1 ga kamaytiradi
    console.log('Cleanup count:', count);
  };
}, []);
```

Strict Mode'da Output:
```
Setup count: 1
Cleanup count: 0
Setup count: 1
```

Boshlang'ich va oxirgi count bir xil (1) — cleanup setup'ni teng kompensatsiya qildi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Effect object — cleanup saqlash joyi:**

Har `useEffect` chaqiruvi `Effect` object yaratadi:

```ts
type Effect = {
  tag: HookFlags,           // HookPassive | HookLayout | HookInsertion
  create: () => () => void, // Setup function
  destroy: void | (() => void), // Cleanup function (setup return)
  deps: Array<unknown> | null,
  next: Effect,             // Circular linked list
};
```

`destroy` field — setup chaqiriladigan paytda set qilinadi:

```ts
function commitHookEffectListMount(hookFlags: HookFlags, finishedWork: Fiber) {
  const updateQueue = finishedWork.updateQueue;
  let lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    
    do {
      if ((effect.tag & hookFlags) === hookFlags) {
        // Setup chaqiriladi
        const create = effect.create;
        effect.destroy = create();  // ✅ Cleanup saqlanadi
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

**Cleanup phase — `commitHookEffectListUnmount`:**

```ts
function commitHookEffectListUnmount(hookFlags: HookFlags, finishedWork: Fiber) {
  const updateQueue = finishedWork.updateQueue;
  let lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    
    do {
      if ((effect.tag & hookFlags) === hookFlags) {
        const destroy = effect.destroy;
        effect.destroy = undefined;
        if (destroy !== undefined) {
          destroy();  // ✅ Cleanup chaqiriladi
        }
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

**Cleanup va Setup tartibi — atomic:**

R16-17'da cleanup va setup interleave qilinardi: bir effect cleanup → bir effect setup → keyingi effect cleanup → ...

R18'dan boshlab atomic: avval BARCHA effect'larning cleanup, keyin BARCHA effect'larning setup. Bu Concurrent Mode bilan moslik uchun.

**Source citation:**

facebook/react repo, `packages/react-reconciler/src/ReactFiberCommitWork.js`. `commitHookEffectListMount`, `commitHookEffectListUnmount` funksiyalari.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Subscription pattern:**

```tsx
type ChatMessage = { id: string; text: string; userId: string };

function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  
  useEffect(() => {
    const handler = (msg: ChatMessage) => {
      setMessages(prev => [...prev, msg]);
    };
    
    const subscription = chatService.subscribe(roomId, handler);
    
    return () => {
      // ✅ Cleanup — subscription bekor qilinadi
      subscription.unsubscribe();
    };
  }, [roomId]);
  
  return <ul>{messages.map(m => <li key={m.id}>{m.text}</li>)}</ul>;
}

// Hayot scenarisi:
// 1. roomId='general' → subscribe('general')
// 2. roomId='random'  → unsubscribe('general'), subscribe('random')
// 3. unmount         → unsubscribe('random')
```

**Misol 2 — Timer cleanup (kritik):**

```tsx
function PollingComponent({ url }: { url: string }) {
  const [data, setData] = useState<unknown>(null);
  
  useEffect(() => {
    const id = setInterval(async () => {
      const response = await fetch(url);
      setData(await response.json());
    }, 5000);
    
    return () => {
      clearInterval(id);  // ✅ Kritik — yo'qsa interval davom etadi
    };
  }, [url]);
  
  // Cleanup yo'q bo'lsa: url o'zgarganda eski interval davom etadi,
  // 2 marta polling, 3 marta polling, ... (har url o'zgarganda yangi interval qo'shiladi)
  
  return <pre>{JSON.stringify(data)}</pre>;
}
```

**Misol 3 — Event listener cleanup:**

```tsx
function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0);
  
  useEffect(() => {
    const handler = () => setScrollY(window.scrollY);
    
    window.addEventListener('scroll', handler);
    
    return () => {
      window.removeEventListener('scroll', handler);  // ✅ Yo'qsa duplicate listener'lar
    };
  }, []);
  
  return <div>Scroll: {scrollY}</div>;
}
```

**Misol 4 — Cleanup'da closure trap:**

```tsx
function Logger({ id }: { id: number }) {
  useEffect(() => {
    console.log('Setup:', id);
    
    return () => {
      console.log('Cleanup:', id);  // Bu eski id qiymati
    };
  }, [id]);
}

// id=1 render: "Setup: 1"
// id=2 render: "Cleanup: 1", "Setup: 2"  ← cleanup eski id ko'radi
// id=3 render: "Cleanup: 2", "Setup: 3"
// unmount:    "Cleanup: 3"
```

Bu xulq-atvor sync modeli bilan to'g'ri: avvalgi sync'ni bekor qiluvchi cleanup avvalgi state'ni bilishi shart.

**Misol 5 — Idempotent cleanup (Strict Mode safe):**

```tsx
let connectionCount = 0;

function ChatConnection({ roomId }: { roomId: string }) {
  useEffect(() => {
    connectionCount++;
    console.log('Connections:', connectionCount);
    
    return () => {
      connectionCount--;
      console.log('Connections after cleanup:', connectionCount);
    };
  }, [roomId]);
  
  return null;
}

// Strict Mode (R18+):
// Mount:   Connections: 1
// Unmount: Connections after cleanup: 0
// Re-mount:Connections: 1
// Real unmount: Connections after cleanup: 0

// ✅ Cleanup setup'ni teng kompensatsiya qiladi → Strict Mode safe
```

</details>

---

## Effect Timing — Passive vs Layout

### Nazariya

`useEffect` callback'i React'ning render lifecycle'ida aniq belgilangan vaqtda chaqiriladi. Tushunish uchun avval Render lifecycle'ni eslatish (cross-ref [`02-rendering.md`](02-rendering.md)):

```
1. Render Phase (memory'da, parallel mumkin)
   ├─ Component function chaqiriladi
   ├─ JSX → Fiber tree
   └─ Reconciliation (eski vs yangi tree)

2. Commit Phase (sync, atomic)
   ├─ Mutation sub-phase    — DOM yangilanadi
   ├─ Layout sub-phase       — useLayoutEffect chaqiriladi (sync)
   └─ Browser paint         — yangi DOM ekranda ko'rinadi

3. Passive Effects Phase (async, paint dan keyin)
   └─ useEffect chaqiriladi
```

`useEffect` — **passive effect**. Browser paint'dan **keyin** chaqiriladi. Bu degani:

- DOM yangilangan
- Foydalanuvchi yangi UI'ni ko'rgan
- Endi (asinxron tarzda) effect ishlaydi

Bu paint'ni bloklamaslik uchun. Agar effect og'ir bo'lsa (network so'rov, hisoblash), foydalanuvchi UI'ni darrov ko'radi, og'ir ish background'da ketadi.

**`useLayoutEffect` bilan farq:**

| Hook | Vaqti | Sync/Async | Use case |
|------|-------|------------|----------|
| `useLayoutEffect` | DOM mutation'dan keyin, paint'dan **oldin** | Sync (paint'ni bloklaydi) | DOM measure, scroll position, layout-dependent calc |
| `useEffect` | Paint'dan **keyin** | Async (paint'ni bloklamaydi) | Network, subscription, timer, log |

`useLayoutEffect` chuqur [`17-uselayouteffect.md`](17-uselayouteffect.md) da yoritiladi.

**`useInsertionEffect` (R18) — eng erta:**

R18'dan boshlab `useInsertionEffect` qo'shildi. Layout effect'dan ham oldin chaqiriladi — DOM mutation'dan ham oldin. Faqat CSS-in-JS library'lar (styled-components, emotion) uchun mo'ljallangan, application code'da ishlatilmaydi.

```
Commit Phase tartib:
1. useInsertionEffect (R18+, CSS-in-JS uchun)
2. DOM mutation (React DOM update)
3. useLayoutEffect (sync)
4. Browser paint
5. useEffect (async, passive)
```

**Scheduling — MessageChannel:**

Passive effect'lar `MessageChannel` orqali keyingi task'ga rejalashtiriladi. Bu — paint dan keyin, lekin tezroq mumkin. `setTimeout(fn, 0)` ishlatilmaydi (nested timer'lar uchun HTML spec'dagi 4ms minimum clamp bor), `requestIdleCallback` ham ishlatilmaydi (juda kam va kechikib chaqiriladi — idle vaqt bo'lmasa starve bo'lishi mumkin).

```
Browser paint
  ↓ (MessageChannel postMessage)
Passive Effects Phase (next macro task)
  ├─ commitPassiveUnmountEffects (cleanups)
  └─ commitPassiveMountEffects (setups)
```

**Effect mount sync emas:**

Passive effect chaqirilganida React'ning ishi tugagan, lekin browser allaqachon paint qilgan. Bu degani: agar effect DOM o'zgartirsa, foydalanuvchi **flicker** ko'radi:

```tsx
// ❌ Flicker keltiradi
function Tooltip() {
  const ref = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    if (ref.current) {
      // Bu paint dan keyin chaqiriladi
      // Foydalanuvchi avval old position'ni, keyin new position'ni ko'radi
      ref.current.style.left = '100px';
    }
  }, []);
  
  return <div ref={ref}>Tooltip</div>;
}

// ✅ useLayoutEffect — flicker yo'q
function Tooltip() {
  const ref = useRef<HTMLDivElement>(null);
  
  useLayoutEffect(() => {
    if (ref.current) {
      ref.current.style.left = '100px';  // Paint dan oldin
    }
  }, []);
  
  return <div ref={ref}>Tooltip</div>;
}
```

DOM o'zgarishi visible bo'lsa — `useLayoutEffect`. Visible bo'lmasa (network, subscription) — `useEffect`.

<details>
<summary><strong>Under the Hood</strong></summary>

**Passive effect scheduling — `flushPassiveEffects`:**

R18'da passive effect'lar `flushPassiveEffects` orqali ishga tushiriladi. Scheduling — Scheduler paketi orqali, NormalSchedulerPriority bilan:

```ts
// Soddalashtirilgan
function commitRoot(root: FiberRoot) {
  // ... mutation, layout phases ...
  
  if (rootDoesHavePassiveEffects) {
    rootDoesHavePassiveEffects = false;
    rootWithPendingPassiveEffects = root;
    pendingPassiveEffectsLanes = lanes;
    
    // MessageChannel orqali rejalashtirish
    scheduleCallback(NormalSchedulerPriority, () => {
      flushPassiveEffects();
      return null;
    });
  }
  
  // ... boshqa ish ...
}

function flushPassiveEffects(): boolean {
  if (rootWithPendingPassiveEffects !== null) {
    const root = rootWithPendingPassiveEffects;
    rootWithPendingPassiveEffects = null;
    
    // 1. Avval cleanup'lar
    commitPassiveUnmountEffects(root.current);
    
    // 2. Keyin setup'lar
    commitPassiveMountEffects(root, root.current);
  }
  return false;
}
```

**MessageChannel implementation (Scheduler):**

```ts
// Scheduler/src/forks/SchedulerDOM.js (soddalashtirilgan)
const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;

function schedulePerformWorkUntilDeadline() {
  port.postMessage(null);  // Yangi task rejalashtiradi
}
```

`MessageChannel` — `setTimeout(fn, 0)`'dan tezroq (nested timer 4ms clamp yo'q) va `requestIdleCallback`'dan ishonchli (rIC idle vaqt bo'lmasa kechikadi yoki umuman chaqirilmaydi).

**HookPassive flag:**

```ts
const HookPassive = 0b1000;
const HookHasEffect = 0b0001;
const HookLayout = 0b0100;
const HookInsertion = 0b0010;
```

Effect tag = `HookHasEffect | HookPassive` (passive effect, run kerak).

Commit Phase'da `commitPassiveMountEffects` faqat `HookPassive | HookHasEffect` bilan effect'larni chaqiradi. Layout effect'lar (`HookLayout | HookHasEffect`) Commit'ning Layout sub-phase'ida chaqiriladi.

**Source citation:**

- Scheduling: facebook/react `packages/scheduler/src/forks/SchedulerDOM.js`
- Passive effects: facebook/react `packages/react-reconciler/src/ReactFiberCommitWork.js` — `commitPassiveMountEffects`, `commitPassiveUnmountEffects`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Timing demo:**

```tsx
function TimingDemo() {
  console.log('1. Render');
  
  useLayoutEffect(() => {
    console.log('3. useLayoutEffect (paint dan oldin)');
  });
  
  useEffect(() => {
    console.log('5. useEffect (paint dan keyin)');
  });
  
  // Browser paint orasida: '4. Browser paint' (visible)
  
  console.log('2. JSX qaytariladi');
  
  return <div>Timing</div>;
}

// Output:
// 1. Render
// 2. JSX qaytariladi
// (Commit Phase — DOM yangilanadi)
// 3. useLayoutEffect (paint dan oldin)
// (4. Browser paint — UI ko'rinadi)
// 5. useEffect (paint dan keyin)
```

**Misol 2 — Flicker demonstration:**

```tsx
function FlickerExample() {
  const [width, setWidth] = useState(100);
  
  useEffect(() => {
    // Paint dan keyin — foydalanuvchi 100px ni ko'rib, keyin 500px ga o'tishini ko'radi
    const timer = setTimeout(() => setWidth(500), 0);
    return () => clearTimeout(timer);
  }, []);
  
  return <div style={{ width, height: 50, background: 'blue' }}>Flicker</div>;
}

// Visible flicker: 100px → 500px transition foydalanuvchiga ko'rinadi

// Yechim: useLayoutEffect (sync, paint dan oldin)
function NoFlicker() {
  const [width, setWidth] = useState(100);
  
  useLayoutEffect(() => {
    setWidth(500);  // Paint dan oldin — foydalanuvchi to'g'ridan-to'g'ri 500px ko'radi
  }, []);
  
  return <div style={{ width, height: 50, background: 'blue' }}>No flicker</div>;
}
```

**Misol 3 — Network — useEffect to'g'ri:**

```tsx
function UserCard({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    // Paint dan keyin OK — UI loading state ko'rsatadi, keyin update
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(setUser);
  }, [userId]);
  
  return user ? <div>{user.name}</div> : <div>Loading...</div>;
}

// Visible: avval "Loading..." ko'rinadi (paint), keyin user ma'lumoti (yangi paint)
// useLayoutEffect bu yerda foyda bermaydi: fetch baribir asinxron — javob
// paint dan keyin keladi, "Loading..." baribir ko'rinadi. Faqat effect chaqiruvini
// paint dan oldinga suradi, lekin network javobini emas. Network uchun useEffect to'g'ri
```

</details>

---

## Effect Ordering — Bottom-Up Execution

### Nazariya

Tree'da bir necha component bor. Har birida `useEffect`. Effect'lar qaysi tartibda chaqiriladi?

**Setup tartibi — bottom-up (children before parent):**

```tsx
function Parent() {
  useEffect(() => console.log('Parent effect'), []);
  return <Child />;
}

function Child() {
  useEffect(() => console.log('Child effect'), []);
  return <Grandchild />;
}

function Grandchild() {
  useEffect(() => console.log('Grandchild effect'), []);
  return null;
}

// Output:
// Grandchild effect
// Child effect
// Parent effect
```

Sabab: child component'lar mount qilingach, parent ham to'liq mount bo'ladi. Parent'ning effect'i child'larga tayanishi mumkin (e.g., DOM ref).

**Cleanup tartibi — ham bottom-up:**

R18'dan boshlab cleanup ham bottom-up tartibda. Avval barcha cleanup'lar (bottom-up), keyin barcha setup'lar (bottom-up):

```
Commit Phase (deps changed):
1. Grandchild cleanup
2. Child cleanup
3. Parent cleanup
4. Grandchild setup
5. Child setup
6. Parent setup
```

Bu atomic separation R18 Concurrent Mode bilan kelgan. R17 da cleanup va setup interleave qilinardi (har component'da cleanup → setup → keyingi component).

**Bir component ichida — declaration order:**

Bir component ichida bir necha `useEffect` bo'lsa, ular **chaqirilish tartibida** (declaration order'da) ishlaydi:

```tsx
function Component() {
  useEffect(() => console.log('Effect 1'), []);
  useEffect(() => console.log('Effect 2'), []);
  useEffect(() => console.log('Effect 3'), []);
  
  return null;
}

// Output:
// Effect 1
// Effect 2
// Effect 3
```

Cleanup ham **declaration order'da** ishlaydi — reverse order'da emas. Bir component'ning effect'lari fiber'ning `updateQueue`'sida circular singly-linked list sifatida saqlanadi, `next` pointer declaration tartibida ulanadi. React commit paytida ro'yxatni birinchi effect'dan `next` bo'ylab yuradi — ham setup, ham cleanup uchun bir xil yo'nalish. Demak Effect 3'ning cleanup'i Effect 1'nikidan keyin chaqiriladi, teskari emas.

**Concurrent Mode va effect tartibi:**

Concurrent rendering paytida React render'ni to'xtatib, qaytadan boshlashi mumkin. Effect tartibi esa **commit paytida** aniqlanadi — ya'ni render'lar tashlangan bo'lsa, effect chaqirilmaydi. Faqat haqiqiy commit'da effect'lar bottom-up run qilinadi.

**Effect ordering muhim bo'lgan holatlar:**

1. **Parent child DOM ref'iga tayanadi** — Child mount bo'lganda DOM bor, parent useLayoutEffect uni ko'radi.
2. **Subscription chain** — Child WebSocket'ga ulanadi, parent broadcast qiladi. Child avval ulanmasa, parent broadcast yo'qoladi.
3. **Analytics** — Child'lar mount log qiladi, parent "page loaded" log qilishi kerak — child'lardan keyin.

<details>
<summary><strong>Under the Hood</strong></summary>

**`commitPassiveMountEffects_complete` — recursive bottom-up:**

```ts
function commitPassiveMountEffects_complete(
  subtreeRoot: Fiber,
  root: FiberRoot,
): void {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    
    if ((fiber.flags & Passive) !== 0) {
      try {
        commitPassiveMountOnFiber(root, fiber);  // Effect chaqirish
      } catch (error) {
        captureCommitPhaseError(fiber, fiber.return, error);
      }
    }
    
    if (fiber === subtreeRoot) {
      nextEffect = null;
      return;
    }
    
    const sibling = fiber.sibling;
    if (sibling !== null) {
      sibling.return = fiber.return;
      nextEffect = sibling;
      return;
    }
    
    nextEffect = fiber.return;  // Up to parent
  }
}
```

Algoritm DFS post-order: avval barcha child'lar, keyin parent. Bu bottom-up tartibni kafolatlaydi.

**R17 vs R18 cleanup tartibi:**

R17 (interleaved):
```
Component A: cleanup → setup
Component B: cleanup → setup
Component C: cleanup → setup
```

R18 (atomic, separate passes):
```
Pass 1 (cleanup): Component A → B → C cleanup
Pass 2 (setup):   Component A → B → C setup
```

R18 atomic separation Concurrent Mode'da Suspense bilan ishlash uchun zaruriy. Render to'xtatilsa, cleanup'lar to'liq tushib, setup'lar yangi state bilan boshlanadi.

**Source citation:**

facebook/react `packages/react-reconciler/src/ReactFiberCommitWork.js` — `commitPassiveMountEffects`, `commitPassiveUnmountEffects`. R18 PR — "useEffect always cleans up before re-running" (effect cycle invariant).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Bottom-up demo:**

```tsx
function App() {
  useEffect(() => {
    console.log('App mount');
    return () => console.log('App cleanup');
  });
  
  return <Outer />;
}

function Outer() {
  useEffect(() => {
    console.log('Outer mount');
    return () => console.log('Outer cleanup');
  });
  
  return <Inner />;
}

function Inner() {
  useEffect(() => {
    console.log('Inner mount');
    return () => console.log('Inner cleanup');
  });
  
  return null;
}

// Mount tartibi:
// Inner mount
// Outer mount
// App mount

// Unmount tartibi (R18):
// Inner cleanup
// Outer cleanup
// App cleanup
```

**Misol 2 — Bir component ichida bir necha effect:**

```tsx
function MultiEffect({ id }: { id: number }) {
  useEffect(() => {
    console.log('Effect 1 setup', id);
    return () => console.log('Effect 1 cleanup', id);
  }, [id]);
  
  useEffect(() => {
    console.log('Effect 2 setup', id);
    return () => console.log('Effect 2 cleanup', id);
  }, [id]);
  
  return <div>{id}</div>;
}

// id=1 mount:
// Effect 1 setup 1
// Effect 2 setup 1

// id=2 update (id deps o'zgardi):
// Effect 1 cleanup 1
// Effect 2 cleanup 1
// Effect 1 setup 2
// Effect 2 setup 2

// Cleanup ham declaration order'da (reverse emas)
```

**Misol 3 — Parent child DOM ref dependency:**

```tsx
function Tooltip({ children }: { children: React.ReactNode }) {
  const ref = useRef<HTMLDivElement>(null);
  
  useLayoutEffect(() => {
    if (ref.current) {
      // Child'lar mount bo'lgan → DOM tayyor
      const rect = ref.current.getBoundingClientRect();
      console.log('Tooltip rect:', rect);
    }
  });
  
  return <div ref={ref}>{children}</div>;
}

function Page() {
  return (
    <Tooltip>
      <span>Hover me</span>
    </Tooltip>
  );
}

// Mount tartibi:
// 1. <span> mount (DOM yaratiladi)
// 2. <Tooltip> mount, useLayoutEffect chaqiriladi
// 3. ref.current bor — getBoundingClientRect ishlaydi
```

</details>

---

## Use Cases — Real-World Pattern'lar

### Nazariya

`useEffect`'ning haqiqiy ishlash sohalari — tashqi tizimlar bilan sync. Eng tez-tez uchraydigan pattern'lar:

**1. Subscription / Event Listener:**

External source (WebSocket, EventEmitter, MediaQuery, browser API) ga ulanish va data oqimini qabul qilish.

**2. Timers (setInterval / setTimeout):**

Polling, periodic update, debounce/throttle implementation, animation triggering.

**3. Network Requests (caveat bilan):**

API'dan data olish. Lekin bu — eng ko'p anti-pattern keltirib chiqaradigan use case. Boshqa yechimlar (Suspense + library, Server Components, framework data layer) ko'p hollarda yaxshi.

**4. DOM Manipulation (visible bo'lmagan):**

Document title, scroll position, focus management — `useLayoutEffect` emas, `useEffect`. Document title visible emas, scroll position user action emas.

**5. Logging / Analytics:**

Page view tracking, event tracking. Render'da emas — Concurrent Mode'da render qaytadan ishlashi mumkin.

Har use case'ning patternini ko'rib chiqamiz.

**Pattern 1 — Subscription:**

```tsx
function useSubscription<T>(source: Subscribable<T>): T | null {
  const [value, setValue] = useState<T | null>(null);
  
  useEffect(() => {
    const subscription = source.subscribe(setValue);
    return () => subscription.unsubscribe();
  }, [source]);
  
  return value;
}
```

R18'dan boshlab `useSyncExternalStore` ham mavjud (tearing-safe). Concurrent Mode'da external store uchun afzal (cross-ref keyingi bo'limlar).

**Pattern 2 — Polling with cleanup:**

```tsx
function usePolling<T>(fetcher: () => Promise<T>, interval: number): T | null {
  const [data, setData] = useState<T | null>(null);
  
  useEffect(() => {
    let active = true;
    
    const poll = async () => {
      const result = await fetcher();
      if (active) setData(result);  // Race condition prevention
    };
    
    poll();  // Birinchi chaqiruv
    const id = setInterval(poll, interval);
    
    return () => {
      active = false;
      clearInterval(id);
    };
  }, [fetcher, interval]);
  
  return data;
}
```

`active` flag — race condition'ni oldini olish (component unmount bo'lsa, response e'tiborga olinmaydi).

**Pattern 3 — Network fetch (caveat bilan):**

```tsx
function useUser(userId: string) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    const controller = new AbortController();
    
    setLoading(true);
    setError(null);
    
    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then(r => {
        if (!r.ok) throw new Error('Fetch failed');
        return r.json();
      })
      .then((data: User) => {
        setUser(data);
        setLoading(false);
      })
      .catch((err: Error) => {
        if (err.name !== 'AbortError') {
          setError(err);
          setLoading(false);
        }
      });
    
    return () => controller.abort();
  }, [userId]);
  
  return { user, loading, error };
}
```

Bu pattern keng tarqalgan, lekin ko'p kamchilik bilan: race conditions, cache yo'q, retry yo'q, loading/error state manual. Production'da SWR, TanStack Query, RTK Query kabi library'lar (alohida kursda) ishlatiladi.

**Pattern 4 — Document title sync:**

```tsx
function usePageTitle(title: string) {
  useEffect(() => {
    const previous = document.title;
    document.title = title;
    
    return () => {
      document.title = previous;
    };
  }, [title]);
}

// R19'dan boshlab — alternative:
function Page({ title }: { title: string }) {
  return (
    <>
      <title>{title}</title>  {/* R19 — JSX'da metadata */}
      <main>...</main>
    </>
  );
}
```

R19'da `<title>`, `<meta>`, `<link>` JSX'da yozilishi mumkin (cross-ref [`37-react-19-document-apis.md`](37-react-19-document-apis.md)). Bu `useEffect` bilan title o'rnatishni o'rnini bosadi.

**Pattern 5 — Analytics:**

```tsx
function PageView({ pageName }: { pageName: string }) {
  useEffect(() => {
    analytics.trackPageView({ page: pageName, timestamp: Date.now() });
  }, [pageName]);
  
  return null;
}
```

Analytics — render paytida emas (Concurrent Mode tashlanishi mumkin), event handler ham emas (page view event-driven emas, render-driven). `useEffect` to'g'ri.

<details>
<summary><strong>Under the Hood</strong></summary>

**`useSyncExternalStore` (R18+) — `useEffect`'ga alternativ:**

External store subscription uchun `useEffect` Concurrent Mode'da tearing keltirib chiqaradi (render paytida store qiymati o'zgarsa, ba'zi component'lar eski qiymat, ba'zilari yangi qiymat ko'radi). `useSyncExternalStore` bu muammoni hal qiladi:

```tsx
function useStore<T>(store: Store<T>): T {
  return useSyncExternalStore(
    (cb) => store.subscribe(cb),  // Subscribe
    () => store.getState(),        // Get snapshot
    () => store.getServerState()   // SSR snapshot (optional)
  );
}
```

Concurrent rendering paytida `useSyncExternalStore`:
1. Render boshida snapshot olinadi
2. Render davomida store o'zgarsa, render yangi snapshot bilan qayta boshlanadi (yo'qolish yo'q)

Bu — `useEffect`'da subscribe qilishdan ko'ra xavfsiz.

**Network fetch — Suspense + library:**

R18 Suspense + R19 `use()` hook bilan, fetch declarative bo'ladi:

```tsx
// ❌ useEffect bilan
function UserCard({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);
  useEffect(() => { fetch(...).then(setUser); }, [userId]);
  if (!user) return <Loading />;
  return <Card user={user} />;
}

// ✅ Suspense + use() (R19)
function UserCard({ userId }: { userId: string }) {
  const user = use(fetchUser(userId));  // Promise → value
  return <Card user={user} />;
}

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <UserCard userId="alice" />
    </Suspense>
  );
}
```

`use()` — React 19'da promise'ni "throw" qilib, Suspense'ga loading state ko'rsatadi. Cache library tomonidan ta'minlanadi.

**Source citation:**

- `useSyncExternalStore` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`
- `use()` — R19 RFC, React docs

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Window resize listener:**

```tsx
function useWindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });
  
  useEffect(() => {
    const handler = () => {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    };
    
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  }, []);
  
  return size;
}

function ResponsiveLayout() {
  const { width } = useWindowSize();
  return <div>{width < 768 ? <Mobile /> : <Desktop />}</div>;
}
```

**Misol 2 — MediaQuery subscription:**

```tsx
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() => window.matchMedia(query).matches);
  
  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
    
    mediaQuery.addEventListener('change', handler);
    setMatches(mediaQuery.matches);  // Sync joriy holat
    
    return () => mediaQuery.removeEventListener('change', handler);
  }, [query]);
  
  return matches;
}

function DarkModeIcon() {
  const isDark = useMediaQuery('(prefers-color-scheme: dark)');
  return <span>{isDark ? '🌙' : '☀️'}</span>;
}
```

**Misol 3 — Debounced search:**

```tsx
function useDebouncedValue<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  
  return debounced;
}

function SearchBox() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebouncedValue(query, 500);
  
  useEffect(() => {
    if (debouncedQuery) {
      // Network so'rov debouncedQuery o'zgarganda
      fetch(`/api/search?q=${debouncedQuery}`).then(r => r.json());
    }
  }, [debouncedQuery]);
  
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

**Misol 4 — Local storage sync:**

```tsx
function useLocalStorage<T>(key: string, initialValue: T): [T, (v: T) => void] {
  const [value, setValue] = useState<T>(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initialValue;
  });
  
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);
  
  return [value, setValue];
}

function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage<'dark' | 'light'>('theme', 'dark');
  
  return (
    <button onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}>
      Theme: {theme}
    </button>
  );
}
```

**Misol 5 — Online status:**

```tsx
function useOnlineStatus(): boolean {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  
  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);
    
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);
  
  return isOnline;
}

function NetworkBanner() {
  const isOnline = useOnlineStatus();
  if (isOnline) return null;
  return <div>Offline mode</div>;
}
```

</details>

---

## Race Conditions va `AbortController`

### Nazariya

Race condition — `useEffect` ichida async operation (fetch, promise) bo'lganda, deps tezroq o'zgarsa va eski response keyin kelsa, eski qiymat yangi state'ni bosib o'tirib oladi.

**Klassik race condition:**

```tsx
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(setUser);
  }, [userId]);
  
  return <div>{user?.name}</div>;
}

// Hayot scenarisi:
// t=0:    userId='alice' → fetch('/api/users/alice') boshladi (sekin server)
// t=100ms: userId='bob'  → fetch('/api/users/bob') boshladi (tez server)
// t=200ms: bob response keldi → setUser(bob)
// t=500ms: alice response keldi → setUser(alice)  ❌ NOTO'G'RI
//
// Foydalanuvchi 'bob' ko'rishi kerak, lekin 'alice' ko'rsatildi
```

Bu real bug. Production'da o'tkazilishi mumkin, ayniqsa tezkor input'larda (search, autocomplete).

**Yechim 1 — `AbortController`:**

`fetch` API `AbortController` orqali bekor qilinishi mumkin. Cleanup function'da abort:

```tsx
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    const controller = new AbortController();
    
    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then(r => r.json())
      .then(setUser)
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error(err);
        }
      });
    
    return () => controller.abort();  // Eski fetch bekor qilinadi
  }, [userId]);
  
  return <div>{user?.name}</div>;
}

// Hayot scenarisi:
// t=0:    userId='alice' → fetch('/api/users/alice', signal: controller1.signal)
// t=100ms: userId='bob'  → cleanup: controller1.abort() → eski fetch bekor
//                       → fetch('/api/users/bob', signal: controller2.signal)
// t=200ms: bob response → setUser(bob) ✅
// t=500ms: alice fetch — bekor qilingan, AbortError throw → catch'da ignore
```

**Yechim 2 — `ignore` flag:**

`AbortController` har joyda mavjud emas (eski browser, custom async API). Universal yechim — flag:

```tsx
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    let ignore = false;
    
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(data => {
        if (!ignore) setUser(data);  // Ignore eski response
      });
    
    return () => {
      ignore = true;  // Closure'da yangi cleanup'da o'rnatiladi
    };
  }, [userId]);
  
  return <div>{user?.name}</div>;
}
```

`ignore` flag — closure ichida, har effect cycle'da yangi flag. Cleanup eski flag'ni `true` qiladi — eski response ignore qilinadi.

**Yechim 3 — async/await pattern:**

Async/await da signal'ni handle qilish:

```tsx
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    const controller = new AbortController();
    
    async function loadUser() {
      try {
        const response = await fetch(`/api/users/${userId}`, {
          signal: controller.signal,
        });
        const data = await response.json();
        setUser(data);
      } catch (err) {
        if ((err as Error).name === 'AbortError') return;
        throw err;
      }
    }
    
    loadUser();
    
    return () => controller.abort();
  }, [userId]);
  
  return <div>{user?.name}</div>;
}
```

**Async function to'g'ridan-to'g'ri `useEffect` callback'da TAQIQ:**

```tsx
// ❌ XATO — useEffect async callback qabul qilmaydi
useEffect(async () => {
  const data = await fetchData();
  setData(data);
}, []);

// Sabab: useEffect callback void yoki cleanup function qaytarishi kerak
// Async function Promise qaytaradi — Promise function emas, React typeof === 'function'
// tekshiruvidan o'tmaydi → cleanup umuman ishlamaydi (race condition oldini olib bo'lmaydi)

// ✅ TO'G'RI — async function ichida e'lon qilinadi
useEffect(() => {
  async function load() {
    const data = await fetchData();
    setData(data);
  }
  load();
}, []);
```

**Strict Mode (R18+) va race condition:**

Strict Mode'da effect ikki marta chaqiriladi (mount → unmount → mount). AbortController pattern bilan birinchi fetch bekor qilinadi, ikkinchisi davom etadi. `ignore` pattern bilan ham bir xil mantiq. Demak race condition pattern'lari Strict Mode safe — bu sync invariant'ining test'i.

<details>
<summary><strong>Under the Hood</strong></summary>

**`AbortController` API:**

```ts
class AbortController {
  signal: AbortSignal;
  abort(reason?: any): void;
}

class AbortSignal {
  aborted: boolean;
  reason?: any;
  addEventListener(type: 'abort', cb: () => void): void;
}
```

`fetch` API standart `signal` qabul qiladi. Boshqa API'lar (axios, custom) `signal`'ni o'z implementation'ida tekshirishi kerak.

**Browser support:** Chrome 66+, Firefox 57+, Safari 11.1+, Node.js 15+. Bugungi kunda universal.

**Race condition matematikasi:**

`N` ta tezkor input bo'lsa, eng ko'p `N` ta concurrent fetch bo'lishi mumkin. Ularning kelish tartibi noaniq. Faqat oxirgisi qabul qilinishi shart.

`AbortController`/`ignore` pattern'lari bu invariant'ni ta'minlaydi. Lekin server-side ish bekor qilinmaydi (server fetch'ni jarayonda davom ettiradi, faqat response e'tiborga olinmaydi). Server load uchun debouncing (cross-ref Misol "Debounced search") kerak.

**TanStack Query / SWR / RTK Query:**

Production'da bu library'lar race condition'ni avtomatik hal qiladi: cache key bo'yicha eski request'lar bekor qilinadi. Ko'p kod yozish o'rniga library ishlatish — manual `useEffect + fetch` pattern'idan ko'ra xavfsizroq.

**Source citation:**

- AbortController: WHATWG DOM spec
- React docs "You Might Not Need an Effect" race condition example

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — AbortController to'liq:**

```tsx
type Product = { id: string; name: string; price: number };

function ProductDetail({ productId }: { productId: string }) {
  const [product, setProduct] = useState<Product | null>(null);
  const [error, setError] = useState<string | null>(null);
  
  useEffect(() => {
    const controller = new AbortController();
    setProduct(null);
    setError(null);
    
    fetch(`/api/products/${productId}`, { signal: controller.signal })
      .then(r => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r.json();
      })
      .then(setProduct)
      .catch((err: Error) => {
        if (err.name === 'AbortError') return;  // Bekor qilingan — ignore
        setError(err.message);
      });
    
    return () => controller.abort();
  }, [productId]);
  
  if (error) return <div>Error: {error}</div>;
  if (!product) return <div>Loading...</div>;
  return <div>{product.name}: ${product.price}</div>;
}
```

**Misol 2 — Ignore flag (custom async API):**

```tsx
function useDataFromCustomSource(key: string) {
  const [data, setData] = useState<unknown>(null);
  
  useEffect(() => {
    let ignore = false;
    
    customAsyncSource.fetch(key).then(result => {
      if (!ignore) setData(result);
    });
    
    return () => {
      ignore = true;
    };
  }, [key]);
  
  return data;
}
```

**Misol 3 — Search with debounce + AbortController:**

```tsx
type SearchResult = { id: string; title: string };

function useSearch(query: string): SearchResult[] | null {
  const [results, setResults] = useState<SearchResult[] | null>(null);
  
  useEffect(() => {
    if (!query.trim()) {
      setResults(null);
      return;
    }
    
    const controller = new AbortController();
    const debounceId = setTimeout(() => {
      fetch(`/api/search?q=${encodeURIComponent(query)}`, {
        signal: controller.signal,
      })
        .then(r => r.json())
        .then(setResults)
        .catch((err: Error) => {
          if (err.name !== 'AbortError') console.error(err);
        });
    }, 300);
    
    return () => {
      clearTimeout(debounceId);
      controller.abort();
    };
  }, [query]);
  
  return results;
}

function SearchPage() {
  const [query, setQuery] = useState('');
  const results = useSearch(query);
  
  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {results === null ? <div>No results</div> : (
        <ul>{results.map(r => <li key={r.id}>{r.title}</li>)}</ul>
      )}
    </div>
  );
}
```

Debounce + AbortController kombinatsiyasi: tezkor input'larda har 300ms'da bir so'rov, eski so'rovlar bekor qilinadi.

**Misol 4 — Async/await pattern:**

```tsx
function useAsyncData<T>(fetcher: (signal: AbortSignal) => Promise<T>) {
  const [data, setData] = useState<T | null>(null);
  
  useEffect(() => {
    const controller = new AbortController();
    
    (async () => {
      try {
        const result = await fetcher(controller.signal);
        setData(result);
      } catch (err) {
        if ((err as Error).name !== 'AbortError') throw err;
      }
    })();
    
    return () => controller.abort();
  }, [fetcher]);
  
  return data;
}
```

**Misol 5 — Async callback xato:**

```tsx
// ❌ XATO
useEffect(async () => {
  const data = await fetch('/api/data').then(r => r.json());
  setData(data);
}, []);

// Bu kodda React quyidagi muammolar:
// 1. Callback Promise qaytaradi
// 2. React Promise'ni cleanup deb tushunmaydi (Promise function emas) — cleanup ishlamaydi
// 3. Dev mode'da React warning beradi: "useEffect must not return anything besides
//    a function, which is used for clean-up. ... you returned a Promise"
// 4. TypeScript signature `() => void | (() => void)` async function'ni rad etadi
// 5. Race condition (cleanup yo'q) — eski response yangini bosib ketadi

// ✅ TO'G'RI
useEffect(() => {
  let ignore = false;
  
  (async () => {
    const data = await fetch('/api/data').then(r => r.json());
    if (!ignore) setData(data);
  })();
  
  return () => { ignore = true; };
}, []);
```

</details>

---

## Stale Closure va Missing Deps

### Nazariya

`useEffect` callback — closure. Closure paytida component render'ning state va props qiymatlarini "ushlab" oladi. Agar deps to'g'ri kelmasa, callback **eski qiymatlarni ko'radi** — bu **stale closure** muammosi.

**Klassik stale closure misoli:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      console.log('Count:', count);  // ❌ Eski count qiymati
      setCount(count + 1);             // ❌ Eski count + 1
    }, 1000);
    
    return () => clearInterval(id);
  }, []);  // ❌ count deps'da yo'q
  
  return <div>{count}</div>;
}

// Kutilgan: 0, 1, 2, 3, 4, ...
// Aslida:   0, 1, 1, 1, 1, ...
//
// Sabab: callback faqat birinchi render'dagi count=0 ni biladi
// setCount(0 + 1) → state=1, lekin closure eski count=0 ko'radi
// Keyingi tick: setCount(0 + 1) → state hali 1
```

**Yechim 1 — deps to'g'ri:**

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    
    return () => clearInterval(id);
  }, [count]);  // ✅ count o'zgarsa interval qaytadan
  
  return <div>{count}</div>;
}

// Lekin bu interval har sekund qaytadan yaratiladi (har count'da)
// Performance optimal emas, lekin to'g'ri
```

**Yechim 2 — Functional update:**

`setCount` setter funksiyasi qabul qiladi: `setCount(prev => ...)`. Funksiya joriy state'ni qabul qiladi (eski count emas):

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      setCount(prev => prev + 1);  // ✅ Joriy state ishlatiladi
    }, 1000);
    
    return () => clearInterval(id);
  }, []);  // ✅ deps yo'q — count o'qimaymiz
  
  return <div>{count}</div>;
}

// Kutilgan: 0, 1, 2, 3, 4, ... ✅
```

`setCount(prev => prev + 1)` joriy state'ni `prev` argumentida oladi. Closure'dagi eski qiymat ishlatilmaydi.

**Yechim 3 — useRef latest pattern:**

Murakkab holatlarda `useRef` orqali "latest" qiymatni saqlash:

```tsx
function useLatest<T>(value: T): React.MutableRefObject<T> {
  const ref = useRef(value);
  
  useEffect(() => {
    ref.current = value;
  });  // Har render'da yangilanadi
  
  return ref;
}

function ChatRoom({ message }: { message: string }) {
  const latestMessage = useLatest(message);
  
  useEffect(() => {
    const handler = () => {
      console.log('Latest message:', latestMessage.current);  // Doim yangi
    };
    
    socket.on('event', handler);
    return () => socket.off('event', handler);
  }, []);  // message deps'da yo'q
  
  return <div>{message}</div>;
}
```

`useRef` qiymatini o'zgartirish re-render keltirib chiqarmaydi, lekin closure latest qiymatni ko'radi. Anti-pattern emas, lekin ehtiyotkorlik bilan ishlatiladi.

**`react-hooks/exhaustive-deps` linter:**

ESLint rule barcha reactive deps'ni majbur qiladi. Stale closure'ni topishda asosiy vosita:

```tsx
function Component({ userId }: { userId: string }) {
  useEffect(() => {
    fetch(`/api/users/${userId}`);
  }, []);  // ⚠️ Linter: 'userId' deps'da yo'q
}
```

Linter warning'ni `// eslint-disable-next-line` bilan o'chirish ko'p hollarda bug yashiradi. To'g'ri yondashuv:

1. Deps qo'shish (effect qaytadan ishlaydi — yo'qotish bormi?)
2. Functional update ishlatish (state read kerak emas bo'lsa)
3. Effect dizaynini o'zgartirish (effect noto'g'ri joyda)

**`useEffectEvent` (R19 RFC, eksperimental):**

`useEffectEvent` (avval `useEvent`) — reactive bo'lmasdan, callback ichida latest qiymatni ishlatish uchun. Hali stable emas (R19 dan keyin keladi):

```tsx
// Eksperimental — production'da ishlatish tavsiya etilmaydi
function ChatRoom({ roomId, theme }: { roomId: string; theme: string }) {
  const onConnected = useEffectEvent(() => {
    showNotification(`Connected to ${roomId}`, theme);  // theme latest
  });
  
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.on('connected', () => onConnected());
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);  // theme deps'da kerak emas
}
```

`useEffectEvent` qachon stable bo'ladi — aniq emas. Bugungi kunda `useRef` latest pattern ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Closure mexanizmi (cross-ref [`../js/05-closures.md`](../js/05-closures.md)):**

JavaScript'da har function yaratilganda joriy lexical environment'ga reference saqlaydi (`[[Environment]]` internal slot). Function chaqirilganida shu environment'dan qiymatlarni o'qiydi.

`useEffect` callback har render'da yangi function (closure). Lekin React deps o'zgarmasa, callback chaqirilmaydi (`HookHasEffect` flag o'rnatilmaydi). Deps o'zgarsa — yangi callback (yangi closure) chaqiriladi.

```ts
// Render 1
useEffect(() => {
  // Bu closure render 1 ning state'ini ushlaydi
  console.log(count);  // count = 0
}, []);

// Render 2 (state o'zgardi)
useEffect(() => {
  // Bu yangi closure, count=1 ni ushlaydi
  console.log(count);  // count = 1
}, []);

// Lekin deps=[] → React render 2 ning callback'ini ishlatmaydi
// Render 1 ning callback'i interval ichida — count=0 ushlab turadi
```

**Functional update qanday hal qiladi:**

`setCount(prev => prev + 1)` setter funksiyani qabul qiladi. React bu funksiyani **joriy state** bilan chaqiradi, eski emas:

```ts
// React internal (soddalashtirilgan):
// Update'lar queue.pending'da circular singly-linked list sifatida turadi:
// pending — oxirgi update, pending.next — birinchi update.
type Update = { action: unknown; lane: Lane; next: Update };

function dispatchSetState(fiber: Fiber, queue: UpdateQueue, action: unknown) {
  const update = { action, lane: requestUpdateLane(fiber) } as Update;
  const pending = queue.pending;
  if (pending === null) {
    update.next = update;  // O'z-o'ziga ulanadi (circular)
  } else {
    update.next = pending.next;  // Birinchiga ulanadi
    pending.next = update;
  }
  queue.pending = update;  // Yangi oxirgi
  scheduleUpdateOnFiber(fiber);
}

// Render paytida (updateReducer ichida):
function processQueue(queue: UpdateQueue, prevState: any): any {
  let state = prevState;
  const pending = queue.pending;
  if (pending !== null) {
    const first = pending.next;  // Circular ro'yxatning birinchisi
    let update = first;
    do {
      if (typeof update.action === 'function') {
        state = update.action(state);  // Functional update — joriy state
      } else {
        state = update.action;          // Direct update
      }
      update = update.next;
    } while (update !== first);  // To'liq aylanib chiqilgach to'xtaydi
  }
  return state;
}
```

Functional update closure'ga bog'liq emas — har doim joriy state'ni ishlatadi. Bu race condition'larda ham foydali (multiple `setCount(prev => prev + 1)` chaqiruvlari to'g'ri qo'shiladi).

**Linter implementation:**

`react-hooks/exhaustive-deps` AST analiz qiladi:

1. `useEffect(callback, deps)` topish
2. `callback` ichida ishlatilgan barcha identifier'larni AST'dan olish
3. Identifier'larni "reactive" yoki "non-reactive" deb ajratish:
   - Reactive: `useState`/`useReducer` qaytaradigan, props, context, custom hook qaytaradigan
   - Non-reactive: module-level, ref.current, setter funksiyalar
4. Reactive identifier'larni deps bilan solishtirish

Linter `useState`/`useReducer` setter'larini stable deb biladi va ularni deps talab qilmaydi (setter identity re-render'da o'zgarmaydi). Ba'zi murakkab holatlarda (masalan, ref orqali uzatilgan callback) linter chegaraviy noaniqlik ko'rsatishi mumkin, lekin umuman warning'larni jiddiy qabul qilish kerak.

**Source citation:**

- `react-hooks/exhaustive-deps`: facebook/react `packages/eslint-plugin-react-hooks/src/ExhaustiveDeps.js`
- `useEffectEvent` RFC: react.dev "Separating Events from Effects"

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Stale closure va functional update:**

```tsx
function Stopwatch() {
  const [seconds, setSeconds] = useState(0);
  
  // ❌ Stale closure
  useEffect(() => {
    const id = setInterval(() => {
      setSeconds(seconds + 1);  // Eski seconds
    }, 1000);
    return () => clearInterval(id);
  }, []);
  
  // ✅ Functional update
  useEffect(() => {
    const id = setInterval(() => {
      setSeconds(prev => prev + 1);  // Joriy state
    }, 1000);
    return () => clearInterval(id);
  }, []);
  
  return <div>{seconds}s</div>;
}
```

**Misol 2 — Multiple state updates:**

```tsx
function MultiCounter() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    // ❌ Stale — har uchchala bir xil count'ni ishlatadi
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
    // count 0 dan 1 ga (3 emas)
    
    // ✅ Functional — qiymatlar zanjir
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
    // count 0 dan 3 ga
  };
  
  return <button onClick={handleClick}>{count}</button>;
}
```

**Misol 3 — useRef latest pattern:**

```tsx
function useLatest<T>(value: T) {
  const ref = useRef(value);
  ref.current = value;  // ⚠️ Render paytida mutate — texnik jihatdan render purity buzilishi
  // To'g'ri: useEffect bilan
  return ref;
}

// To'g'ri pattern (purity-safe):
function useLatest<T>(value: T) {
  const ref = useRef(value);
  
  useEffect(() => {
    ref.current = value;
  });
  
  return ref;
}

function EventHandler({ callback }: { callback: () => void }) {
  const latestCallback = useLatest(callback);
  
  useEffect(() => {
    const handler = () => latestCallback.current();
    
    window.addEventListener('click', handler);
    return () => window.removeEventListener('click', handler);
  }, []);  // callback deps'da yo'q — latest ref orqali
  
  return null;
}
```

**Misol 4 — Linter warning va to'g'ri yechim:**

```tsx
function SearchResults({ query, onResultsLoaded }: {
  query: string;
  onResultsLoaded: (results: unknown[]) => void;
}) {
  // ❌ Linter warning: 'onResultsLoaded' deps'da yo'q
  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(r => r.json())
      .then(onResultsLoaded);
  }, [query]);
  
  // ✅ Yechim 1 — deps qo'shish
  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(r => r.json())
      .then(onResultsLoaded);
  }, [query, onResultsLoaded]);
  
  // Lekin onResultsLoaded har render'da yangi function bo'lsa (parent'da inline) —
  // har render'da fetch qaytadan
  
  // ✅ Yechim 2 — useCallback parent'da (cross-ref 21-usememo-usecallback.md)
  // Parent: const handler = useCallback((results) => {...}, []);
  
  return null;
}
```

**Misol 5 — Effect dizaynini o'zgartirish:**

```tsx
// ❌ Stale closure muammosi
function ChatRoom({ roomId, lastReadAt }: { roomId: string; lastReadAt: Date }) {
  useEffect(() => {
    const interval = setInterval(() => {
      console.log('lastReadAt:', lastReadAt);  // Eski qiymat
      pollForNewMessages(roomId, lastReadAt);
    }, 5000);
    
    return () => clearInterval(interval);
  }, [roomId]);  // lastReadAt deps'da yo'q (qaytadan polling istamayman)
}

// ✅ Yechim — useRef bilan latest
function ChatRoom({ roomId, lastReadAt }: { roomId: string; lastReadAt: Date }) {
  const lastReadRef = useRef(lastReadAt);
  
  useEffect(() => {
    lastReadRef.current = lastReadAt;
  });
  
  useEffect(() => {
    const interval = setInterval(() => {
      pollForNewMessages(roomId, lastReadRef.current);  // Latest
    }, 5000);
    
    return () => clearInterval(interval);
  }, [roomId]);  // ✅ Polling roomId o'zgarsa qaytadan
}
```

</details>

---

## Object/Array Deps — Referential Identity

### Nazariya

`useEffect` deps comparison `Object.is` orqali. Object va array deps'lari **reference equality** bo'yicha solishtiriladi — qiymat emas, identity (xotirada bir xil joymi).

**Klassik trap:**

```tsx
function Component({ userId }: { userId: string }) {
  const userOptions = { userId, includeProfile: true };  // ⚠️ Har render'da yangi object
  
  useEffect(() => {
    fetchUser(userOptions);
  }, [userOptions]);  // ❌ Har render'da userOptions yangi reference → effect qaytadan
  
  return null;
}

// Har render'da effect chaqiriladi (hatto userId o'zgarmasa ham)
// Sabab: { userId, includeProfile: true } — har render'da yangi object literal
// Object.is(prevOptions, newOptions) → false (har xil reference)
```

**Yechim 1 — Primitive deps:**

Object/array o'rniga primitive qiymatlarni deps'da ishlatish:

```tsx
function Component({ userId }: { userId: string }) {
  useEffect(() => {
    fetchUser({ userId, includeProfile: true });  // Object effect ichida yaratiladi
  }, [userId]);  // ✅ Faqat primitive deps
  
  return null;
}
```

**Yechim 2 — `useMemo` (object kerak bo'lsa):**

Agar object/array komponentning boshqa joyiga ham kerak bo'lsa, `useMemo` bilan saqlash:

```tsx
function Component({ userId, includeProfile }: { userId: string; includeProfile: boolean }) {
  const userOptions = useMemo(
    () => ({ userId, includeProfile }),
    [userId, includeProfile]
  );
  
  useEffect(() => {
    fetchUser(userOptions);
  }, [userOptions]);  // ✅ userOptions deps o'zgarmasa bir xil reference
  
  return null;
}
```

`useMemo` chuqur [`21-usememo-usecallback.md`](21-usememo-usecallback.md) da yoritiladi.

**Yechim 3 — Function deps va `useCallback`:**

Function deps ham bir xil muammo. Har render'da yangi funksiya:

```tsx
function Parent() {
  const handler = (data: unknown) => {  // ⚠️ Har render'da yangi function
    console.log(data);
  };
  
  return <Child onUpdate={handler} />;
}

function Child({ onUpdate }: { onUpdate: (data: unknown) => void }) {
  useEffect(() => {
    socket.on('update', onUpdate);
    return () => socket.off('update', onUpdate);
  }, [onUpdate]);  // ❌ onUpdate har render'da yangi → effect qaytadan
}

// Yechim — useCallback parent'da:
function Parent() {
  const handler = useCallback((data: unknown) => {
    console.log(data);
  }, []);  // ✅ Doimiy reference
  
  return <Child onUpdate={handler} />;
}
```

**Yechim 4 — JSON.stringify (anti-pattern, lekin holatga qarab):**

Ba'zan deep equality kerak. JSON.stringify deps'da — anti-pattern, lekin oddiy holatlarda ishlaydi:

```tsx
function Component({ filters }: { filters: Filters }) {
  // ⚠️ Anti-pattern, lekin oddiy holatlarda ishlaydi
  const filtersKey = JSON.stringify(filters);
  
  useEffect(() => {
    fetchData(filters);
  }, [filtersKey]);  // ⚠️ Har render'da JSON.stringify ishlaydi (sekin)
}
```

Production'da `useMemo` bilan filters'ni saqlash yaxshi — JSON.stringify har render'da ishlatish performance'ga ta'sir.

**Sabab — Object.is mexanikasi:**

```ts
const a = { x: 1 };
const b = { x: 1 };

Object.is(a, a);  // true  — bir xil reference
Object.is(a, b);  // false — har xil reference (qiymat bir xil)

const arr1 = [1, 2, 3];
const arr2 = [1, 2, 3];

Object.is(arr1, arr2);  // false — har xil reference
```

JavaScript'da object/array — heap'da. Har literal yangi heap allocation. Reference (pointer) har gal yangi.

**React'ning dizayn qarori:**

React deep equality ishlatmaydi (default). Sabab:

1. **Performance** — deep equality O(n) (n = object size). Har render'da har deps'da O(n) — sekinroq.
2. **Predictability** — reference comparison O(1), aniq xulq-atvor.
3. **User control** — kerak bo'lsa `useMemo` bilan reference saqlash mumkin.

Library'lar (use-deep-compare-effect, fast-deep-equal) deep equality alternative beradi, lekin bu rasmiy emas. React documentation primitive deps + `useMemo` tavsiya qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`Object.is` vs `===`:**

```ts
// Object.is — Object.is(value1, value2)
// SameValue algoritmi (ECMAScript spec)

Object.is(NaN, NaN);     // true   ← === false bo'lardi
Object.is(0, -0);        // false  ← === true bo'lardi
Object.is(+0, +0);       // true
Object.is('a', 'a');     // true
Object.is({}, {});       // false  (har xil reference)
Object.is(null, null);   // true
Object.is(undefined, undefined); // true
```

`Object.is` — primitives uchun value equality, objects uchun reference equality. NaN va `-0` uchun `===`'dan farqli.

**JavaScript object identity:**

JavaScript engine'lar object'lar uchun reference (pointer) bilan ishlaydi. Har object literal yangi heap allocation:

```ts
function makeObject() {
  return { x: 1 };  // Har chaqiruvda yangi heap allocation
}

const a = makeObject();
const b = makeObject();
console.log(a === b);  // false — har xil pointers
```

V8 da object identity — pointer addressi. Identity comparison O(1) — pointer comparison.

**Hidden classes va inline caches (cross-ref [`../js/object-internals.md`](../js/object-internals.md)):**

V8 har object uchun "Hidden Class" yaratadi (Map). Bir xil shape'dagi object'lar bir xil Hidden Class. Lekin **identity** bu emas — Hidden Class shape, identity object addressi.

Bu nima uchun React reference comparison ishlatadi: pointer comparison engine darajasida tezkor (CPU register comparison).

**`useMemo` mexanikasi:**

```ts
function updateMemo<T>(create: () => T, deps: any[] | null): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  
  if (nextDeps !== null && prevState !== null) {
    const prevDeps = prevState[1];
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      return prevState[0];  // ✅ Eski qiymat — bir xil reference
    }
  }
  
  const nextValue = create();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

`useMemo` deps o'zgarmasa eski reference qaytaradi. Bu — referential identity'ni saqlash.

**Source citation:**

- `Object.is` — ECMAScript spec `SameValue` algorithm
- `useMemo` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Object dep trap:**

```tsx
function ProductList({ category }: { category: string }) {
  // ❌ Har render'da yangi object
  const filters = { category, inStock: true, sortBy: 'name' };
  
  useEffect(() => {
    fetchProducts(filters);
  }, [filters]);  // Effect har render'da
  
  return null;
}

// ✅ Primitive deps
function ProductList({ category }: { category: string }) {
  useEffect(() => {
    fetchProducts({ category, inStock: true, sortBy: 'name' });
  }, [category]);  // Effect faqat category o'zgarganda
  
  return null;
}

// ✅ useMemo (filters JSX'da ham kerak bo'lsa)
function ProductList({ category }: { category: string }) {
  const filters = useMemo(
    () => ({ category, inStock: true, sortBy: 'name' }),
    [category]
  );
  
  useEffect(() => {
    fetchProducts(filters);
  }, [filters]);
  
  return <div>Filters: {JSON.stringify(filters)}</div>;
}
```

**Misol 2 — Array dep trap:**

```tsx
function TagList({ baseTag }: { baseTag: string }) {
  // ❌ Har render'da yangi array
  const tags = [baseTag, 'react', 'typescript'];
  
  useEffect(() => {
    syncTags(tags);
  }, [tags]);  // Effect har render'da
  
  return null;
}

// ✅ Primitive deps
function TagList({ baseTag }: { baseTag: string }) {
  useEffect(() => {
    syncTags([baseTag, 'react', 'typescript']);
  }, [baseTag]);
  
  return null;
}

// ✅ Spread va memo (massiv kerak)
function TagList({ baseTag, additional }: { baseTag: string; additional: string[] }) {
  const tags = useMemo(
    () => [baseTag, ...additional],
    [baseTag, additional]  // additional ham reference dep — caller useMemo bilan bersin
  );
  
  useEffect(() => {
    syncTags(tags);
  }, [tags]);
  
  return null;
}
```

**Misol 3 — Function dep trap:**

```tsx
// ❌ Parent har render'da yangi handler
function Parent() {
  const [count, setCount] = useState(0);
  
  return (
    <Child
      onUpdate={(data) => console.log(count, data)}  // Har render'da yangi
    />
  );
}

function Child({ onUpdate }: { onUpdate: (data: unknown) => void }) {
  useEffect(() => {
    socket.on('event', onUpdate);
    return () => socket.off('event', onUpdate);
  }, [onUpdate]);  // ❌ Har render'da effect qaytadan
}

// ✅ useCallback parent'da
function Parent() {
  const [count, setCount] = useState(0);
  
  const handleUpdate = useCallback((data: unknown) => {
    console.log(count, data);
  }, [count]);  // count o'zgarsa yangi (lekin har render'da emas)
  
  return <Child onUpdate={handleUpdate} />;
}
```

**Misol 4 — Object identity test:**

```tsx
function IdentityTest() {
  const [count, setCount] = useState(0);
  
  // Render orasida bir xil reference saqlanadi
  const a = useRef({ value: 1 });
  
  // Har render'da yangi object literal
  const b = { value: count };
  
  useEffect(() => {
    console.log('a.current:', a.current);  // Bir xil reference — ref
    console.log('b:', b);                  // Har render'da yangi
  });
  
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// useRef qiymati render orasida saqlanadi (bir xil reference)
// Object literal har render'da yangi
```

**Misol 5 — Anti-pattern: JSON.stringify:**

```tsx
function ExpensiveSync({ data }: { data: unknown[] }) {
  // ⚠️ Anti-pattern — JSON.stringify har render'da
  const dataKey = JSON.stringify(data);
  
  useEffect(() => {
    syncData(data);
  }, [dataKey]);  // Deep comparison emulation
  
  // Yaxshi yechim: parent useMemo bilan data'ni saqlasin
  // Yoki: data id ishlatilsin (data.id, data.version, etc.)
  
  return null;
}

// ✅ Versionlash yondashuvi
function ExpensiveSync({ data, dataVersion }: { data: unknown[]; dataVersion: number }) {
  useEffect(() => {
    syncData(data);
  }, [dataVersion]);  // ✅ Primitive — version o'zgarsa effect
}
```

</details>

---

## Strict Mode 2x Effect Cycle (R18+)

### Nazariya

R18'dan boshlab Strict Mode'da har `useEffect` development'da ikki marta chaqiriladi:

```
Mount → Cleanup → Mount (yana)
```

Bu — **intentional** xulq-atvor, bug emas. Sabab: effect "qaytadan o'rnatilishga chidamli" bo'lishi kerak. Production'da effect bir marta chaqiriladi, lekin development'da bug'larni topish uchun cycle simulate qilinadi.

**Versiya tarixi:**

| React versiyasi | Render 2x | Effect 2x cycle |
|------------------|-----------|------------------|
| Pre-R16.3 | ❌ Yo'q | ❌ Yo'q |
| R16.3 — R17 | ✅ Strict Mode'da | ❌ Yo'q |
| R18+ | ✅ Strict Mode'da | ✅ Strict Mode'da |

**Muhim chalkashlik:** Strict Mode "render 2x" R16.3 da kelgan, lekin "effect 2x cycle" faqat R18 da. Ko'p resurslar bu ikki narsani aralashtiradi. Aniq:

- **Render 2x** — komponent funksiyasi 2 marta chaqiriladi (R16.3+). Render purity test.
- **Effect 2x cycle** — effect mount → unmount → mount sikli (R18+). Cleanup invariant test.

**Misol:**

```tsx
function Component() {
  console.log('Render');
  
  useEffect(() => {
    console.log('Setup');
    return () => console.log('Cleanup');
  }, []);
  
  return null;
}

// Strict Mode (R18+) Output:
// Render
// Render        ← R16.3+ render 2x
// Setup         ← Mount
// Cleanup       ← Unmount (simulate)
// Setup         ← Re-mount
//
// Production Output:
// Render
// Setup
```

**Sabab — sync invariant'i:**

R18 dan kelajakdagi feature'lar uchun React effect remount'ni qo'llab-quvvatlashi kerak:

1. **Activity API** (avval `Offscreen` deb atalgan) — komponent vaqtinchalik ekrandan yashiriladi (state saqlanadi), keyin qaytadan ko'rsatiladi. Effect remount kerak.
2. **Fast Refresh** — development'da hot reload paytida component state saqlanib, effect remount.
3. **Error recovery** — error boundary error tutsa, component remount qilinishi mumkin.

Bu holatlarda effect cleanup to'liq bo'lmasa — bug. Strict Mode 2x cycle bu bug'ni development'da topadi.

**Misol — Strict Mode topadigan bug:**

```tsx
// ❌ Cleanup yo'q
function ChatRoom({ roomId }: { roomId: string }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    // ❌ Cleanup yo'q
  }, [roomId]);
}

// Strict Mode (R18+):
// Mount:    connection1.connect()
// Unmount:  (cleanup yo'q — connection1 hali ochiq)
// Mount:    connection2.connect()
// 
// Natija: 2 ta ulanish ochiq → memory leak, double messages

// ✅ Cleanup bilan
function ChatRoom({ roomId }: { roomId: string }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();  // ✅
  }, [roomId]);
}

// Strict Mode (R18+):
// Mount:    connection1.connect()
// Unmount:  connection1.disconnect()  ✅
// Mount:    connection2.connect()
// 
// Natija: faqat 1 ta ulanish ochiq → to'g'ri
```

**Strict Mode'ni o'chirish — anti-pattern:**

Ba'zi developer'lar Strict Mode'ni o'chirib qo'yadi (effect 2x irritate qiladi):

```tsx
// ❌ ANTI-PATTERN
function App() {
  return (
    // <StrictMode>  ← O'chirildi
      <Router>...</Router>
    // </StrictMode>
  );
}
```

Bu **xato** yondashuv. Strict Mode bug'larni topishga yordam beradi. O'chirish — bug'larni yashirish.

**To'g'ri yondashuv** — effect'larni cleanup bilan to'g'ri yozish. Production'da 2x cycle yo'q (faqat development'da). Production'da Strict Mode no-op.

**Real holatlar — ehtiyot bo'lish kerak:**

1. **Analytics fire 2x** — development'da page view 2x log qilinadi. Yechim: analytics service development'da no-op qilish.
2. **POST request 2x** — development'da POST 2x yuboriladi. Yechim: POST'ni event handler'ga ko'chirish (POST asosan user action javobi).
3. **Modal open 2x** — modal animation 2x ishlaydi. Yechim: modal state'ini parent'da boshqarish.

Bu — anti-pattern emas, **dizayn kamchiligi** — effect noto'g'ri vazifaga ishlatilgan.

**`useEffectEvent` (R19 RFC):**

Strict Mode 2x cycle'ni hal qilishning bir usuli — `useEffectEvent` (eksperimental). Lekin hali stable emas. Bugungi kunda to'g'ri yondashuv: cleanup yozish va effect'ni sync invariant'i bilan dizayn qilish.

<details>
<summary><strong>Under the Hood</strong></summary>

**Strict Mode implementation:**

```ts
// Soddalashtirilgan
function commitMountEffectsInDev(fiber: Fiber) {
  // Production
  commitHookEffectListMount(HookHasEffect | HookPassive, fiber);
  
  // Development + Strict Mode
  if (__DEV__ && (fiber.mode & StrictMode) !== 0) {
    commitHookEffectListUnmount(HookHasEffect | HookPassive, fiber);  // Simulate cleanup
    commitHookEffectListMount(HookHasEffect | HookPassive, fiber);     // Re-mount
  }
}
```

Strict Mode bayrog'i Fiber'da. Faqat Strict Mode subtree'da effect 2x cycle.

**Production'da yo'q:**

```ts
if (__DEV__) {
  // Strict Mode 2x cycle
}
```

`__DEV__` — bundler tomonidan compile-time'da `true` (development) yoki `false` (production). Production build'da bu blok dead code — bundle hajmida bo'lmaydi.

**Strict Mode invariants:**

1. Function component body ikki marta chaqiriladi (render purity test)
2. Effect setup → cleanup → setup cycle (R18+)
3. `useState`/`useMemo`/`useReducer`'ga uzatilgan initializer va updater funksiyalari ikki marta chaqiriladi (impurity'ni topish uchun; natija deduplicate qilinadi)
4. Ref callback'lar setup → cleanup → setup cycle (R18+)

Hammasi development'da bug'larni topish uchun. Production'da no-op.

**`Offscreen` API (kelajakdagi, hozir `<Activity>`):**

R18 RFC'da `Offscreen` component sifatida taklif qilingan API keyinchalik `<Activity>` deb nomlandi (hali stable emas):

```tsx
<Activity mode="hidden">
  <ExpensiveComponent />
</Activity>

// State saqlanadi, lekin DOM yo'q
// Effect cleanup chaqiriladi (component "yashirildi")
// Mode "visible" bo'lganda effect yana mount

// Strict Mode 2x cycle bu holatni development'da test qiladi
```

**Source citation:**

- Strict Mode 2x effect: facebook/react `packages/react-reconciler/src/ReactFiberCommitWork.js`
- Offscreen RFC: reactjs/rfcs `0213-suspense-in-react-18.md`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Strict Mode bug topadi:**

```tsx
// ❌ Subscription leak
function NewsTickerBad({ topic }: { topic: string }) {
  useEffect(() => {
    eventBus.on(`news:${topic}`, handler);
    // ❌ Cleanup yo'q
  }, [topic]);
}

// Strict Mode console:
// Setup: news:tech subscribed
// Setup: news:tech subscribed  ← Duplicate!
// Production'da bug ko'rinmaydi, lekin Strict Mode topadi

// ✅ To'g'ri
function NewsTickerGood({ topic }: { topic: string }) {
  useEffect(() => {
    eventBus.on(`news:${topic}`, handler);
    return () => eventBus.off(`news:${topic}`, handler);
  }, [topic]);
}
```

**Misol 2 — Idempotent counter (Strict Mode safe):**

```tsx
let activeConnections = 0;

function Connection() {
  useEffect(() => {
    activeConnections++;
    console.log('Active:', activeConnections);
    
    return () => {
      activeConnections--;
      console.log('Active after cleanup:', activeConnections);
    };
  }, []);
  
  return null;
}

// Strict Mode:
// Active: 1
// Active after cleanup: 0
// Active: 1
// (final cleanup): Active after cleanup: 0
//
// Boshlang'ich va oxirgi count = 0 ✅
// Cleanup setup'ni teng kompensatsiya qildi
```

**Misol 3 — Analytics 2x (development'da):**

```tsx
// Sababli yondashuv — development'da 2x log
function PageView({ page }: { page: string }) {
  useEffect(() => {
    if (process.env.NODE_ENV === 'production') {
      analytics.trackPageView({ page });
    }
    // Development'da log qilinmaydi → 2x cycle ko'rinmaydi
  }, [page]);
}

// Yoki — analytics service development'da no-op qilish
const analytics = process.env.NODE_ENV === 'production'
  ? realAnalytics
  : { trackPageView: () => {} };

function PageView({ page }: { page: string }) {
  useEffect(() => {
    analytics.trackPageView({ page });  // Dev'da no-op
  }, [page]);
}
```

**Misol 4 — POST request — event handler'ga ko'chirish:**

```tsx
// ❌ POST useEffect ichida — Strict Mode'da 2x
function CreateUser({ userData }: { userData: UserData }) {
  useEffect(() => {
    fetch('/api/users', {
      method: 'POST',
      body: JSON.stringify(userData),
    });
  }, []);
}

// ✅ POST event handler ichida — user action javobi
function CreateUser({ userData }: { userData: UserData }) {
  const handleSubmit = () => {
    fetch('/api/users', {
      method: 'POST',
      body: JSON.stringify(userData),
    });
  };
  
  return <button onClick={handleSubmit}>Create</button>;
}
```

POST asosan user action javobi — `useEffect` uchun emas. Sync emas.

**Misol 5 — Modal — state parent'da:**

```tsx
// ❌ Modal effect ichida ochiladi — Strict Mode'da 2x animation
function Page({ showModal }: { showModal: boolean }) {
  useEffect(() => {
    if (showModal) {
      modalService.open();  // 2x animation
    }
    return () => modalService.close();
  }, [showModal]);
}

// ✅ Modal — declarative JSX
function Page({ showModal, onClose }: { showModal: boolean; onClose: () => void }) {
  return (
    <>
      <main>...</main>
      {showModal && <Modal onClose={onClose} />}
    </>
  );
}
```

Modal — UI qismi, declarative JSX'da. `useEffect` uchun emas.

</details>

---

## "You Might Not Need an Effect"

### Nazariya

Dan Abramov va React jamoasining 2023 yilda yozgan rasmiy qo'llanmasi (`react.dev` "You Might Not Need an Effect") — `useEffect`'ni noto'g'ri ishlatishning eng keng tarqalgan anti-pattern'larini sanaydi.

Asosiy fikr: **`useEffect` — faqat tashqi tizim bilan sync uchun**. Boshqa hamma vazifalar uchun yaxshi alternativlar bor.

Quyida 9 ta anti-pattern va to'g'ri yondashuvlar:

**Anti-pattern 1 — Derived state from props/state:**

```tsx
// ❌ useEffect derived state uchun
function Bill({ items }: { items: Item[] }) {
  const [total, setTotal] = useState(0);
  
  useEffect(() => {
    setTotal(items.reduce((sum, item) => sum + item.price, 0));
  }, [items]);  // ❌ Render → effect → render
  
  return <div>Total: ${total}</div>;
}

// ✅ Render paytida hisoblash
function Bill({ items }: { items: Item[] }) {
  const total = items.reduce((sum, item) => sum + item.price, 0);
  return <div>Total: ${total}</div>;
}

// ✅ Og'ir hisoblash bo'lsa — useMemo
function Bill({ items }: { items: Item[] }) {
  const total = useMemo(
    () => items.reduce((sum, item) => sum + item.price, 0),
    [items]
  );
  return <div>Total: ${total}</div>;
}
```

State'dan derivatsiya — render paytida. Extra render kerak emas.

**Anti-pattern 2 — Reset state on prop change:**

```tsx
// ❌ Effect bilan reset
function ProfileForm({ userId }: { userId: string }) {
  const [name, setName] = useState('');
  
  useEffect(() => {
    setName('');  // ❌ userId o'zgarganda reset
  }, [userId]);
}

// ✅ key prop bilan
function App({ userId }: { userId: string }) {
  return <ProfileForm key={userId} />;  // ✅ key o'zgarsa component remount
}

function ProfileForm() {
  const [name, setName] = useState('');  // Mount paytida boshlang'ich
  return <input value={name} onChange={e => setName(e.target.value)} />;
}
```

`key` prop o'zgarganda React component'ni unmount va remount qiladi (cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md) "Reset state with key").

**Anti-pattern 3 — Notify parent of changes:**

```tsx
// ❌ Effect parent'ga xabar
function Toggle({ onChange }: { onChange: (on: boolean) => void }) {
  const [isOn, setIsOn] = useState(false);
  
  useEffect(() => {
    onChange(isOn);  // ❌ State o'zgarsa parent'ga
  }, [isOn, onChange]);
}

// ✅ Event handler ichida
function Toggle({ onChange }: { onChange: (on: boolean) => void }) {
  const [isOn, setIsOn] = useState(false);
  
  const handleClick = () => {
    const next = !isOn;
    setIsOn(next);
    onChange(next);  // ✅ Bir vaqtda state va parent
  };
  
  return <button onClick={handleClick}>{isOn ? 'On' : 'Off'}</button>;
}
```

Yoki — controlled component (parent state'ni boshqaradi):

```tsx
// ✅ Controlled — state parent'da
function Toggle({ isOn, onChange }: { isOn: boolean; onChange: (on: boolean) => void }) {
  return <button onClick={() => onChange(!isOn)}>{isOn ? 'On' : 'Off'}</button>;
}
```

**Anti-pattern 4 — Pass data to parent:**

```tsx
// ❌ Child fetch qiladi va parent'ga effect bilan
function Child({ onLoaded }: { onLoaded: (data: Data) => void }) {
  const [data, setData] = useState<Data | null>(null);
  
  useEffect(() => {
    fetchData().then(setData);
  }, []);
  
  useEffect(() => {
    if (data) onLoaded(data);  // ❌ Effect chain
  }, [data, onLoaded]);
}

// ✅ Lift state up — parent fetch qiladi
function Parent() {
  const [data, setData] = useState<Data | null>(null);
  
  useEffect(() => {
    fetchData().then(setData);
  }, []);
  
  return <Child data={data} />;
}

function Child({ data }: { data: Data | null }) {
  return data ? <div>{data.name}</div> : <div>Loading...</div>;
}
```

**Anti-pattern 5 — Subscribe to external store:**

```tsx
// ❌ useEffect external store uchun
function Component() {
  const [snapshot, setSnapshot] = useState(externalStore.getSnapshot());
  
  useEffect(() => {
    return externalStore.subscribe(() => {
      setSnapshot(externalStore.getSnapshot());
    });
  }, []);
}

// ✅ useSyncExternalStore (R18+)
function Component() {
  const snapshot = useSyncExternalStore(
    externalStore.subscribe,
    externalStore.getSnapshot
  );
}
```

`useSyncExternalStore` Concurrent Mode'da tearing'ni oldini oladi.

**Anti-pattern 6 — Initialize app:**

```tsx
// ❌ Effect ichida app init
function App() {
  useEffect(() => {
    setupAnalytics();
    setupErrorTracking();
    setupServiceWorker();
  }, []);  // ❌ Strict Mode'da 2x
}

// ✅ Module-level
setupAnalytics();
setupErrorTracking();
setupServiceWorker();

function App() {
  // ...
}
```

App init — module load paytida bir marta. Component lifecycle bilan bog'liq emas.

**Anti-pattern 7 — Send POST request on mount:**

```tsx
// ❌ POST mount paytida
function NewUserPage({ userId }: { userId: string }) {
  useEffect(() => {
    fetch('/api/users', { method: 'POST', body: JSON.stringify({ userId }) });
  }, [userId]);  // ❌ Strict Mode'da 2x POST
}

// ✅ Event handler ichida
function NewUserForm() {
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    fetch('/api/users', { method: 'POST', body: JSON.stringify({ ... }) });
  };
  
  return <form onSubmit={handleSubmit}>...</form>;
}
```

**Anti-pattern 8 — Chains of effects:**

```tsx
// ❌ Effect chain — har biri keyingisini trigger qiladi
function Game() {
  const [card, setCard] = useState<Card | null>(null);
  const [goldCardCount, setGoldCardCount] = useState(0);
  const [round, setRound] = useState(1);
  const [isGameOver, setIsGameOver] = useState(false);
  
  useEffect(() => {
    if (card?.gold) {
      setGoldCardCount(c => c + 1);
    }
  }, [card]);
  
  useEffect(() => {
    if (goldCardCount > 3) {
      setRound(r => r + 1);
      setGoldCardCount(0);
    }
  }, [goldCardCount]);
  
  useEffect(() => {
    if (round > 5) {
      setIsGameOver(true);
    }
  }, [round]);
}

// ✅ Event handler — barcha update'lar bir joyda
function Game() {
  const [card, setCard] = useState<Card | null>(null);
  const [goldCardCount, setGoldCardCount] = useState(0);
  const [round, setRound] = useState(1);
  const [isGameOver, setIsGameOver] = useState(false);
  
  const handleCardPlay = (newCard: Card) => {
    setCard(newCard);
    
    if (newCard.gold) {
      const nextCount = goldCardCount + 1;
      
      if (nextCount > 3) {
        const nextRound = round + 1;
        setRound(nextRound);
        setGoldCardCount(0);
        
        if (nextRound > 5) {
          setIsGameOver(true);
        }
      } else {
        setGoldCardCount(nextCount);
      }
    }
  };
}
```

Effect chain — debug qiyin, race condition mumkin. Event handler bilan barcha logic bir joyda.

**Anti-pattern 9 — Initializing state from props:**

```tsx
// ❌ Effect bilan props → state
function Form({ initialName }: { initialName: string }) {
  const [name, setName] = useState('');
  
  useEffect(() => {
    setName(initialName);  // ❌ Mount paytida
  }, []);
}

// ✅ Lazy initial state
function Form({ initialName }: { initialName: string }) {
  const [name, setName] = useState(initialName);  // ✅ Bir marta mount paytida
  return <input value={name} onChange={e => setName(e.target.value)} />;
}

// initialName keyin o'zgarsa, name avtomatik o'zgarmaydi (intentional)
// O'zgarishi kerak bo'lsa — key prop bilan reset
```

**Decision Guide:**

Quyidagi savollarni o'zingizga bering:

1. **Bu derived state'mi?** → Render paytida hisoblang yoki `useMemo`
2. **Bu user action javobi'mi?** → Event handler
3. **Bu reset state'mi?** → `key` prop
4. **Bu external store subscription'mi?** → `useSyncExternalStore`
5. **Bu app init'mi?** → Module-level code
6. **Bu lifecycle hodisaga bog'liq deb o'ylayapsizmi?** → To'xtang. Sync invariant'ini ifodalang.

Agar javob "tashqi tizim bilan sync" bo'lsa — `useEffect` to'g'ri tanlov. Boshqa har holatda — alternativ.

<details>
<summary><strong>Under the Hood</strong></summary>

**Effect chain — race condition source:**

Bir necha effect bir-birini trigger qilsa, har effect alohida render cycle. Concurrent Mode'da render to'xtatilishi mumkin → state o'rtada bo'lib qoladi.

```ts
// Effect chain timeline
Render 1: card update → useState (card)
  Commit 1
    Effect 1: card.gold → setGoldCardCount(+1)
    
Render 2: goldCardCount update
  Commit 2
    Effect 2: count > 3 → setRound(+1), setGoldCardCount(0)
    
Render 3: round update
  Commit 3
    Effect 3: round > 5 → setIsGameOver(true)

// 3 ta render cycle bir ish uchun
// Render orasida user input bo'lsa — race condition
```

Event handler ichida barcha update'lar bir batch'da (R18 automatic batching) — bir render.

**`useSyncExternalStore` mexanikasi:**

```ts
function useSyncExternalStore<T>(
  subscribe: (cb: () => void) => () => void,
  getSnapshot: () => T,
  getServerSnapshot?: () => T,
): T {
  const value = getSnapshot();  // Render paytida snapshot
  
  // React render davomida snapshot o'zgarmaganini tekshiradi
  // O'zgargan bo'lsa — render qaytadan boshlanadi (no tearing)
  
  useEffect(() => {
    const handleStoreChange = () => {
      const newValue = getSnapshot();
      // React force re-render
    };
    return subscribe(handleStoreChange);
  }, [subscribe]);
  
  return value;
}
```

Bu — `useEffect` + `useState` kombinatsiyasi'dan ko'ra Concurrent Mode'da xavfsiz.

**Source citation:**

- "You Might Not Need an Effect" — react.dev/learn/you-might-not-need-an-effect
- `useSyncExternalStore` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`, React docs "useSyncExternalStore"

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Filter — derived state:**

```tsx
// ❌ useEffect filter uchun
function ProductList({ products, query }: { products: Product[]; query: string }) {
  const [filtered, setFiltered] = useState<Product[]>([]);
  
  useEffect(() => {
    setFiltered(products.filter(p => p.name.includes(query)));
  }, [products, query]);
  
  return <ul>{filtered.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}

// ✅ Render paytida
function ProductList({ products, query }: { products: Product[]; query: string }) {
  const filtered = products.filter(p => p.name.includes(query));
  return <ul>{filtered.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}

// ✅ Og'ir filter — useMemo
function ProductList({ products, query }: { products: Product[]; query: string }) {
  const filtered = useMemo(
    () => products.filter(p => p.name.includes(query)),
    [products, query]
  );
  return <ul>{filtered.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

**Misol 2 — Reset form — key:**

```tsx
// ❌ Reset effect bilan
function CommentForm({ articleId }: { articleId: string }) {
  const [comment, setComment] = useState('');
  
  useEffect(() => {
    setComment('');  // ❌ articleId o'zgarsa reset
  }, [articleId]);
  
  return <textarea value={comment} onChange={e => setComment(e.target.value)} />;
}

// ✅ key bilan
function ArticlePage({ articleId }: { articleId: string }) {
  return (
    <article>
      ...
      <CommentForm key={articleId} />
    </article>
  );
}
```

**Misol 3 — Cascading state — event handler:**

```tsx
// ❌ Effect chain
function ShippingForm({ city }: { city: string }) {
  const [areas, setAreas] = useState<Area[]>([]);
  const [selectedArea, setSelectedArea] = useState<string | null>(null);
  
  useEffect(() => {
    fetchAreas(city).then(setAreas);
  }, [city]);
  
  useEffect(() => {
    if (areas.length > 0) {
      setSelectedArea(areas[0].id);  // ❌ Effect chain
    }
  }, [areas]);
}

// ✅ Yagona effect
function ShippingForm({ city }: { city: string }) {
  const [areas, setAreas] = useState<Area[]>([]);
  const [selectedArea, setSelectedArea] = useState<string | null>(null);
  
  useEffect(() => {
    fetchAreas(city).then(loadedAreas => {
      setAreas(loadedAreas);
      setSelectedArea(loadedAreas[0]?.id ?? null);  // ✅ Bir effect ichida
    });
  }, [city]);
}
```

**Misol 4 — Init app — module-level:**

```tsx
// ❌ Component'da
function App() {
  useEffect(() => {
    Sentry.init({ dsn: '...' });  // ❌ Strict Mode'da 2x
  }, []);
}

// ✅ Module-level
Sentry.init({ dsn: '...' });

function App() {
  // App component
}

// Yoki — entry file (main.tsx)
import { Sentry } from './sentry';
Sentry.init({ dsn: '...' });

createRoot(document.getElementById('root')).render(<App />);
```

**Misol 5 — External store — useSyncExternalStore:**

```tsx
// ❌ useEffect bilan
function CartCount() {
  const [count, setCount] = useState(cart.getItemCount());
  
  useEffect(() => {
    const unsubscribe = cart.subscribe(() => setCount(cart.getItemCount()));
    return unsubscribe;
  }, []);
  
  return <span>{count}</span>;
}

// ✅ useSyncExternalStore (R18+)
function CartCount() {
  const count = useSyncExternalStore(
    cart.subscribe,
    () => cart.getItemCount()
  );
  
  return <span>{count}</span>;
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1 — Async function `useEffect` callback'ida TAQIQ

`useEffect` callback `void` yoki cleanup function qaytarishi kerak. Async function har doim Promise qaytaradi — Promise function emas, shuning uchun React `typeof === 'function'` tekshiruvidan o'tmaydi va uni cleanup sifatida ishlatmaydi → cleanup umuman ishlamaydi. Dev mode'da React warning beradi: *"useEffect must not return anything besides a function, which is used for clean-up. ... you returned a Promise. Instead, write the async function inside your effect and call it immediately."*

```tsx
// ❌ Bug — async callback Promise qaytaradi (cleanup yo'q)
useEffect(async () => {
  const data = await fetch('/api/data').then(r => r.json());
  setData(data);
}, []);

// ✅ Async function ichida e'lon
useEffect(() => {
  async function load() {
    const data = await fetch('/api/data').then(r => r.json());
    setData(data);
  }
  load();
}, []);
```

React rasmiy warning bu pattern uchun chiqaradi. `react-hooks/exhaustive-deps` rule deps tekshiruvi uchun, async callback'ni alohida sintaktik chek qilmaydi — lekin TypeScript signature `() => void | (() => void)` async function'ni rad etadi.

### Gotcha 2 — `setState` cleanup'da memory leak warning (R17 dan keyin yo'q)

R16-R17'da component unmount bo'lganidan keyin `setState` chaqirilsa, "Can't perform a React state update on an unmounted component" warning chiqar edi. R18'dan boshlab bu warning olib tashlandi (PR #22114 — false positive ko'p edi, real memory leak indikator emas).

```tsx
useEffect(() => {
  fetchData().then(data => {
    setData(data);  // R16-17'da unmount keyin warning, R18+ warning yo'q
  });
}, []);
```

Lekin race condition hali bor — `AbortController` yoki `ignore` flag pattern (cross-ref Section "Race Conditions") ishlatish kerak.

### Gotcha 3 — Effect deps lengthi o'zgarishi runtime warning

Deps array uzunligi render orasida o'zgarsa — React warning beradi (development'da):

```tsx
function Component({ flag }: { flag: boolean }) {
  useEffect(() => {
    console.log('effect');
  }, flag ? [1, 2] : [1]);  // ❌ Uzunlik o'zgaradi
}

// Console: "The final argument passed to useEffect changed size between renders."
```

Deps array uzunligi statik bo'lishi shart. Conditional dep o'rniga conditional effect logic.

### Gotcha 4 — Effect callback throw qilsa

Effect callback (passive yoki layout) ichida synchronous throw qilingan error — Error Boundary'ga ushlanadi (`commitPassiveMountEffects` va `commitLayoutEffects` try/catch bilan o'ralgan, `captureCommitPhaseError` chaqiriladi). Lekin error'ning kelishi paint'dan keyin bo'ladi (passive effect uchun) — bu UX ga ta'sir qiladi:

```tsx
useEffect(() => {
  throw new Error('Effect error');  // → Error Boundary fallback paint'dan keyin
}, []);
```

**Async error (fetch reject, promise reject, setTimeout ichidagi throw) hech qachon Error Boundary'ga uzatilmaydi** — JS engine'ning microtask/macrotask queue React commit tree'dan tashqarida. `.catch` yoki `try/catch` bilan handle qilish shart.

### Gotcha 5 — Cleanup va Suspense bilan birga

Component Suspense fallback'ga tushsa (suspense throw promise), effect cleanup chaqirilmaydi. Component "tirik", lekin pause holatda. Resume bo'lganda effect davom etadi.

```tsx
// Suspense fallback paytida cleanup yo'q
function UserCard({ userId }: { userId: string }) {
  const user = use(fetchUser(userId));  // R19 — promise throw
  
  useEffect(() => {
    console.log('Setup');
    return () => console.log('Cleanup');
  }, [userId]);
  
  return <div>{user.name}</div>;
}
```

Effect mount sync invariant'iga ko'ra ishlaydi — agar real effect Suspense bilan bog'liq holatlarda muammo qiladi, dizayn'ni qayta ko'rib chiqish kerak.

---

## Common Mistakes

### ❌ Xato 1 — Effect ichida `setState` infinite loop

```tsx
// ❌ Infinite loop
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    setCount(count + 1);  // ❌ Render → effect → setState → render → effect → ...
  });  // Deps yo'q — har render'da
}

// ✅ Deps bilan
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    if (count < 5) setCount(count + 1);
  }, [count]);
}
```

`setState` effect ichida — deps to'g'ri bo'lishi shart, yoki shart bilan (`if`) chaqirish.

### ❌ Xato 2 — Cleanup yo'q (subscription/timer leak)

```tsx
// ❌ Cleanup yo'q
useEffect(() => {
  const id = setInterval(() => console.log('tick'), 1000);
  // ❌ clearInterval yo'q
}, []);

// ✅ Cleanup
useEffect(() => {
  const id = setInterval(() => console.log('tick'), 1000);
  return () => clearInterval(id);
}, []);
```

Tashqi resurs yaratilganda cleanup MAJBURIY. Yo'qsa memory leak, ghost listener, double subscription.

### ❌ Xato 3 — Object/array deps har render'da yangi

```tsx
// ❌ Har render'da yangi object → effect har render'da
function Component({ userId }: { userId: string }) {
  useEffect(() => {
    fetchUser({ userId, includeProfile: true });
  }, [{ userId, includeProfile: true }]);  // ❌ Object literal deps'da
}

// ✅ Primitive deps
function Component({ userId }: { userId: string }) {
  useEffect(() => {
    fetchUser({ userId, includeProfile: true });
  }, [userId]);
}
```

Object/array deps reference comparison bilan — har render'da yangi reference. `useMemo` yoki primitive deps.

### ❌ Xato 4 — Race condition handle qilinmagan

```tsx
// ❌ Race condition
function UserCard({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    fetch(`/api/users/${userId}`).then(r => r.json()).then(setUser);
  }, [userId]);
  // userId tez o'zgarsa — eski response yangini bosib ketadi
}

// ✅ AbortController
function UserCard({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    const controller = new AbortController();
    
    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then(r => r.json())
      .then(setUser)
      .catch((err: Error) => {
        if (err.name !== 'AbortError') throw err;
      });
    
    return () => controller.abort();
  }, [userId]);
}
```

Async operation effect ichida — race condition pattern (AbortController yoki ignore flag) majburiy.

### ❌ Xato 5 — `useEffect` derived state uchun

```tsx
// ❌ useEffect derived state
function FullName({ firstName, lastName }: { firstName: string; lastName: string }) {
  const [fullName, setFullName] = useState('');
  
  useEffect(() => {
    setFullName(`${firstName} ${lastName}`);  // ❌ Render → effect → render
  }, [firstName, lastName]);
}

// ✅ Render paytida
function FullName({ firstName, lastName }: { firstName: string; lastName: string }) {
  const fullName = `${firstName} ${lastName}`;
  return <div>{fullName}</div>;
}
```

State'dan derivatsiya — `useEffect` shart emas. Render paytida hisoblash, og'ir bo'lsa `useMemo`.

---

## Amaliy Mashqlar

### Mashq 1 — `useDebounce` Hook (Oson)

`useDebounce` custom hook yozing: `value` o'zgarsa, `delay` ms'dan keyin yangi qiymatni qaytaradi. Tezkor o'zgarishlar paytida faqat oxirgi qiymat saqlanadi.

```tsx
function useDebounce<T>(value: T, delay: number): T {
  // Implement
}

function SearchBox() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 500);
  
  // debouncedQuery 500ms tinch turganidan keyin yangilanadi
  
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    
    return () => clearTimeout(id);  // Cleanup eski timeout
  }, [value, delay]);
  
  return debounced;
}
```

**Tushuntirish:**

- `value` o'zgarsa, `useEffect` qaytadan ishlaydi
- Avval cleanup: eski `setTimeout` bekor qilinadi
- Yangi `setTimeout` `delay` ms'dan keyin `debounced` state'ni yangilaydi
- Tezkor o'zgarishlar — har gal eski timeout bekor → faqat oxirgisi ishga tushadi

Sync invariant: "joriy `value` `delay` ms tinch turgandan keyin `debounced`'ga teng bo'lsin."

</details>

### Mashq 2 — `useOnlineStatus` Hook (Oson)

Browser online/offline holatini qaytaruvchi hook yozing.

```tsx
function useOnlineStatus(): boolean {
  // Implement
}

function NetworkBadge() {
  const isOnline = useOnlineStatus();
  return <span>{isOnline ? '🟢' : '🔴'}</span>;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useOnlineStatus(): boolean {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  
  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);
    
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);
  
  return isOnline;
}
```

**Tushuntirish:**

- Initial state — `navigator.onLine` (browser API)
- `online` va `offline` event'larga listener qo'shiladi
- Cleanup'da listener'lar olib tashlanadi (yo'qsa memory leak)
- Strict Mode safe — har listener cleanup bilan teng kompensatsiya

R18+ alternative: `useSyncExternalStore` (Concurrent Mode'da xavfsizroq):

```tsx
function useOnlineStatus(): boolean {
  return useSyncExternalStore(
    (cb) => {
      window.addEventListener('online', cb);
      window.addEventListener('offline', cb);
      return () => {
        window.removeEventListener('online', cb);
        window.removeEventListener('offline', cb);
      };
    },
    () => navigator.onLine
  );
}
```

</details>

### Mashq 3 — `useFetch` Race Condition'siz (O'rta)

`useFetch` hook yozing: URL bo'yicha fetch qiladi, race condition'larni AbortController bilan oldini oladi, loading va error state'larni qaytaradi.

```tsx
type FetchState<T> = {
  data: T | null;
  loading: boolean;
  error: Error | null;
};

function useFetch<T>(url: string): FetchState<T> {
  // Implement
}

function UserCard({ userId }: { userId: string }) {
  const { data, loading, error } = useFetch<User>(`/api/users/${userId}`);
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  return <div>{data?.name}</div>;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useFetch<T>(url: string): FetchState<T> {
  const [state, setState] = useState<FetchState<T>>({
    data: null,
    loading: true,
    error: null,
  });
  
  useEffect(() => {
    const controller = new AbortController();
    
    setState({ data: null, loading: true, error: null });
    
    fetch(url, { signal: controller.signal })
      .then(r => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r.json();
      })
      .then((data: T) => {
        setState({ data, loading: false, error: null });
      })
      .catch((err: Error) => {
        if (err.name === 'AbortError') return;  // Bekor qilingan
        setState({ data: null, loading: false, error: err });
      });
    
    return () => controller.abort();
  }, [url]);
  
  return state;
}
```

**Tushuntirish:**

- `AbortController` har effect cycle'da yangi
- `url` o'zgarsa: cleanup eski fetch'ni bekor qiladi → yangi fetch boshlanadi
- `AbortError` ignore qilinadi (intentional bekor qilish)
- Other error'lar `state.error`'ga yoziladi
- Loading state — fetch boshida `true`, tugaganda `false`

Sync invariant: "joriy `url` uchun fetch state ko'rsatilgan."

Production'da TanStack Query, SWR ishlatish tavsiya — cache, retry, dedupe avtomatik.

</details>

### Mashq 4 — `useInterval` Hook (O'rta)

`useInterval` hook yozing: callback'ni har `delay` ms'da chaqiradi. Callback latest closure'ni ko'rishi kerak (stale closure trapi yo'q).

```tsx
function useInterval(callback: () => void, delay: number | null): void {
  // Implement
  // delay === null bo'lsa interval to'xtatiladi
}

function Counter() {
  const [count, setCount] = useState(0);
  
  useInterval(() => {
    console.log('count:', count);  // ✅ Doim latest count
    setCount(c => c + 1);
  }, 1000);
  
  return <div>{count}</div>;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useInterval(callback: () => void, delay: number | null): void {
  const savedCallback = useRef(callback);
  
  // Latest callback'ni saqlash
  useEffect(() => {
    savedCallback.current = callback;
  });
  
  // Interval setup
  useEffect(() => {
    if (delay === null) return;  // Interval o'chirildi
    
    const id = setInterval(() => {
      savedCallback.current();  // Latest callback chaqiriladi
    }, delay);
    
    return () => clearInterval(id);
  }, [delay]);
}
```

**Tushuntirish:**

- `useRef` callback'ni saqlash uchun
- Birinchi `useEffect` — har render'da latest callback'ni `ref.current`'ga yozadi (deps yo'q)
- Ikkinchi `useEffect` — interval setup, faqat `delay` o'zgarganda qayta
- Interval ichida `savedCallback.current()` chaqiriladi — doim latest closure
- `delay === null` — interval to'xtatiladi

Bu Dan Abramov'ning "Making setInterval Declarative with React Hooks" (overreacted.io) post'idan klassik pattern.

Alternativ — `delay` deps'da, lekin har gal interval qaytadan yaratiladi (timer reset). useRef pattern interval'ni saqlaydi, faqat callback'ni yangilaydi.

</details>

### Mashq 5 — `useEventListener` Generic Hook (Qiyin)

Generic event listener hook yozing: window, element ref, yoki document'ga event listener qo'shadi. TypeScript'da event type aniq bo'lsin.

```tsx
function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (event: WindowEventMap[K]) => void,
  element?: Window
): void;

function useEventListener<K extends keyof HTMLElementEventMap>(
  eventName: K,
  handler: (event: HTMLElementEventMap[K]) => void,
  element: HTMLElement | null
): void;

// Usage:
function ScrollDetector() {
  useEventListener('scroll', (e) => console.log(window.scrollY));
}

function ButtonClickCounter() {
  const ref = useRef<HTMLButtonElement>(null);
  const [count, setCount] = useState(0);
  
  useEventListener('click', () => setCount(c => c + 1), ref.current);
  
  return <button ref={ref}>Clicks: {count}</button>;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function useEventListener<KW extends keyof WindowEventMap, KH extends keyof HTMLElementEventMap, T extends HTMLElement | Window | Document = Window>(
  eventName: KW | KH,
  handler: (event: WindowEventMap[KW] | HTMLElementEventMap[KH] | Event) => void,
  element?: T | null
): void {
  // Latest handler ref
  const savedHandler = useRef(handler);
  
  useEffect(() => {
    savedHandler.current = handler;
  }, [handler]);
  
  useEffect(() => {
    const targetElement: EventTarget | null = element ?? window;
    
    if (!targetElement?.addEventListener) return;
    
    const eventListener: typeof handler = (event) => {
      savedHandler.current(event);
    };
    
    targetElement.addEventListener(eventName, eventListener);
    
    return () => {
      targetElement.removeEventListener(eventName, eventListener);
    };
  }, [eventName, element]);
}
```

**Tushuntirish:**

- TypeScript overload — `Window`, `HTMLElement`, `Document` event maps
- `savedHandler` ref — latest handler closure
- Element default — `window`
- Element `null` (ref hali tayinlanmagan) — guard
- `addEventListener` / `removeEventListener` cleanup'da symmetric

Bu pattern — ko'p UI library'lar (e.g., usehooks-ts) ichida. Production'da test qilingan.

Strict Mode safe: cleanup listener'ni olib tashlaydi → 2x cycle'da duplicate yo'q.

Latest handler pattern — handler har render'da yangi function bo'lsa ham, listener qaytadan ulanmaydi (faqat ref yangilanadi). Performance optimal.

</details>

---

## Xulosa

`useEffect` — React'da eng ko'p tushunmovchilik keltirib chiqaradigan hook. Bu bo'limning asosiy fikrlari:

- **`useEffect` lifecycle method emas, sync mexanizmi.** "Mount paytida X qil, unmount paytida Y qil" mental model'i — anti-pattern. To'g'ri yondashuv: "joriy state berilgan bo'lsa, tashqi tizim qanday holatda bo'lishi kerak — effect shu holatga keltiradi."
- **Side effect'lar render'dan tashqarida bo'lishi kerak.** Render funksiyasi pure (Render Purity Rule). Side effect'lar `useEffect` ichida — Commit Phase'dan keyin chaqiriladi.
- **Dependency array** — sync uchun qaysi state'lar muhim. `Object.is` orqali comparison. `react-hooks/exhaustive-deps` linter — barcha reactive deps majbur qiladi.
- **Cleanup function** — sync'ni bekor qilish. Tashqi resurs (subscription, timer, listener) yaratilganda MAJBURIY.
- **Effect timing** — paint dan keyin (passive). Visible DOM o'zgarishi bo'lsa — `useLayoutEffect` (cross-ref [`17-uselayouteffect.md`](17-uselayouteffect.md)).
- **Effect ordering** — bottom-up (children before parent). R18'dan cleanup va setup atomic separation.
- **Strict Mode 2x effect cycle (R18+)** — mount → unmount → mount sikli. Sync invariant'ini test qiladi (effect remount'ga chidamli bo'lishi shart). Strict Mode'ni o'chirish anti-pattern.
- **Race condition'lar** — async operation'lar uchun `AbortController` yoki `ignore` flag pattern.
- **Stale closure** — deps to'g'ri kelmasa eski qiymatlar. Functional update (`setX(prev => ...)`) yoki `useRef` latest pattern.
- **Object/array deps** — referential identity comparison. Primitive deps yoki `useMemo` (cross-ref [`21-usememo-usecallback.md`](21-usememo-usecallback.md)).
- **"You Might Not Need an Effect"** — derived state (render paytida), reset state (`key` prop), event reaction (event handler), external store (`useSyncExternalStore`), app init (module-level), POST request (event handler), state init (lazy initial state). `useEffect`'ni faqat tashqi tizim syncsi uchun.
- **Under the hood** — Effect — hook chain'da AND effect chain'da (Fiber.updateQueue circular linked list). `HookPassive` flag, `commitPassiveMountEffects` Commit'dan keyin MessageChannel orqali rejalashtiriladi.

Keyingi bo'lim: `useLayoutEffect` — sync timing, paint'dan oldin chaqiriladigan effect, DOM measurement use cases, va `useInsertionEffect` (R18) — CSS-in-JS library'lar uchun.

---

**Keyingi bo'lim:** [17-uselayouteffect.md](17-uselayouteffect.md) — `useLayoutEffect` vs `useEffect` timing farq (sync paint dan oldin vs async paint dan keyin), Layout phase Commit'da, DOM measurement/scroll/focus use cases, performance pitfalls (paint kechikish), SSR cheklov, va `useInsertionEffect` (R18) CSS-in-JS uchun.
