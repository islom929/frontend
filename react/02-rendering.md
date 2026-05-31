# Bo'lim 2: Rendering — Render va Commit Phases

> Rendering — React'ning komponent tree'ni real platformga (DOM) aylantirish jarayoni. Ikki asosiy bosqichdan iborat: **Render Phase** (pure, uziluvchi, workInProgress tree quradi) va **Commit Phase** (atomic, DOM mutation va effect'lar bajariladi). Bu bo'lim createRoot, Strict Mode, batching, va effect timing'ning ichki mexanikasini yoritadi.

---

## Mundarija

- [createRoot va Entry Point](#createroot-va-entry-point)
- [Strict Mode](#strict-mode)
- [Initial Render vs Re-render](#initial-render-vs-re-render)
- [Render Phase](#render-phase)
- [Commit Phase](#commit-phase)
- [Effects Timing — Passive vs Layout](#effects-timing--passive-vs-layout)
- [Automatic Batching](#automatic-batching)
- [flushSync](#flushsync)
- [Concurrent Features — Qisqa Overview](#concurrent-features--qisqa-overview)
- [Hydration — Qisqa Overview](#hydration--qisqa-overview)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## createRoot va Entry Point

### Nazariya

`createRoot` — React 18'dan boshlab joriy etilgan **yangi root API**. U browser DOM elementiga React komponent tree'ni "biriktirish" uchun ishlatiladi. Asosiy maqsad — concurrent features (interruptible rendering, Suspense improvements, automatic batching) uchun zarur infrastructureni o'rnatish.

```tsx
import { createRoot } from 'react-dom/client';

const container = document.getElementById('root');
if (!container) throw new Error('Root element topilmadi');

const root = createRoot(container);
root.render(<App />);
```

`createRoot(container, options?)` ikkinchi argument sifatida ixtiyoriy `options` obyektini qabul qiladi:

| Option | Vazifa | Versiya |
|--------|--------|---------|
| `identifierPrefix` | SSR `useId` uchun prefix (mismatch oldini olish) | R18+ |
| `onCaughtError(error, info)` | Error boundary ushlagan xato uchun callback | R19+ |
| `onUncaughtError(error, info)` | Error boundary tomonidan ushlanmagan xato uchun callback | R19+ |
| `onRecoverableError(error, info)` | Render davomida tiklangan xato (concurrent restart, hydration mismatch) | R18+ |

`createRoot` qaytaradigan `root` obyekti ikkita asosiy metodga ega:

| Metod | Vazifa |
|-------|--------|
| `root.render(element)` | React element'ni mount qilish yoki yangilash |
| `root.unmount()` | Tree'ni butunlay o'chirish, cleanup'lar bilan |

> **🕐 Versiya evolyutsiyasi (Root API):**
> - **Pre-R18 (`ReactDOM.render`):** `ReactDOM.render(<App />, container)` — synchronous rendering, concurrent features yo'q.
> - **R18+ (`createRoot`):** `createRoot(container).render(<App />)` — concurrent features yoqilgan, automatic batching, Suspense improvements.
> - **Sabab:** Eski API bilan concurrent rendering'ni opt-in qilib bo'lmasdi (existing code'larni buzgan bo'lar edi). Yangi API yangi semantikani aniq belgiladi — eski `render()` "legacy mode" sifatida hali ishlaydi (warning bilan), lekin yangi loyihalar `createRoot` ishlatishi shart.

`hydrateRoot` — SSR (Server-Side Rendering) uchun o'xshash API. Server tomonida hosil qilingan HTML'ga client-side React'ni "biriktirib" qo'yadi (cross-ref [`06-hydration.md`](06-hydration.md)).

### Render qachon chaqiriladi

`root.render()` ikki holatda chaqiriladi:

1. **Initial render** — birinchi marta (entry point'da). Bu — komponent tree'ni mount qilish.
2. **External re-render** — komponent state mexanizmidan tashqarida, `root.render(<App {...newProps} />)` ni qayta chaqirish orqali butun tree'ni yangi props bilan render qilish. Bu kamdan-kam ishlatiladi (odatda komponent ichidagi `setState` afzal).

**Eslatma:** Komponent ichidagi `setState` chaqiriqlari `root.render()` ni qayta chaqirmaydi. State update'lari tree ichidagi mexanizm orqali ishlaydi (Fiber update queue). `root.render()` faqat tashqi entry point uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

`createRoot` ichida quyidagi qadamlar bajariladi:

```
1. createRoot(container) chaqiriladi
   ↓
2. FiberRootNode yaratiladi (root container fiber)
   - container DOM node'iga reference saqlanadi
   - update queue boshlang'ich holatga keltiriladi
   - lanes bitmap (priority sistemasi) boshlang'ich qiymat bilan initialize qilinadi
   ↓
3. HostRoot fiber yaratiladi
   - tag: HostRoot (3)
   - stateNode: FiberRootNode reference
   ↓
4. Concurrent mode flag o'rnatiladi
   - Fiber tree concurrent rendering bilan ishlashga tayyor
   ↓
5. root obyekti qaytariladi
   - { render(element), unmount() }
```

Birinchi `root.render(<App />)` chaqirilganda:

```
1. updateContainer(<App />, fiberRoot, ...)
   ↓
2. requestUpdateLane() — priority hisoblanadi (createRoot initial render = DefaultLane; eski ReactDOM.render esa SyncLane ishlatardi)
   ↓
3. createUpdate() — update obyekt yaratiladi
   ↓
4. enqueueUpdate() — update HostRoot'ning queue'siga qo'shiladi
   ↓
5. scheduleUpdateOnFiber() — Scheduler'ga ishni rejalashtirish
   ↓
6. performWorkOnRoot() — render+commit jarayoni boshlanadi
```

`FiberRootNode` (root) va `HostRoot` (fiber) — ikki alohida obyekt. Farq:
- **`FiberRootNode`** — meta-information (container reference, pending lanes, callback queue)
- **`HostRoot` fiber** — tree'ning eng yuqori tugun (root), boshqa fiber'lar bilan bir xil tuzilmaga ega

Ikki tomonlama bog'lanish: `fiberRoot.current === hostRootFiber`, va `hostRootFiber.stateNode === fiberRoot`. Bu — double buffering uchun (cross-ref [`03-fiber-architecture.md`](03-fiber-architecture.md)).

**Eski `ReactDOM.render` farqi:**

Eski API ham FiberRoot yaratardi, lekin **legacy mode** flag bilan. Legacy mode'da:
- Concurrent features yo'q (rendering har doim sync)
- Automatic batching faqat React event handler ichida
- `useTransition`, `useDeferredValue` ishlamaydi (no-op yoki warning)
- Suspense for SSR yo'q

R18'da ikki mode birga yashaydi (`createRoot` = concurrent, `ReactDOM.render` = legacy + warning). Kelgusi major versiyalarda legacy mode butunlay olib tashlanishi mumkin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Standart entry point (Vite, Next.js, va boshqa modern setup'larda):

```tsx
// main.tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';

const container = document.getElementById('root');
if (!container) {
  throw new Error('Root element "#root" topilmadi');
}

const root = createRoot(container);

root.render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

`index.html`:

```html
<!DOCTYPE html>
<html>
<body>
  <div id="root"></div>
  <script type="module" src="/main.tsx"></script>
</body>
</html>
```

Unmount qilish (kamdan-kam ishlatiladi, lekin testing yoki micro-frontend setup'da kerak bo'ladi):

```tsx
const root = createRoot(container);
root.render(<App />);

// Keyinchalik:
root.unmount();
// Tree'dagi barcha komponentlar unmount bo'ladi (cleanup'lar bilan)
// container ichidagi DOM elementlar o'chiriladi
```

External re-render (kamdan-kam — odatda komponent ichida `setState` ishlatiladi):

```tsx
const root = createRoot(container);

// Initial render
root.render(<App theme="light" />);

// Tashqaridan theme o'zgartirish (kamdan-kam pattern)
function changeTheme(newTheme: 'light' | 'dark') {
  root.render(<App theme={newTheme} />);
}
```

Eski API (R18'gacha — hozir ham ishlaydi, ammo warning beradi):

```tsx
// ❌ Eski (R18+ da deprecated, legacy mode'da ishlaydi)
import ReactDOM from 'react-dom';
ReactDOM.render(<App />, document.getElementById('root'));

// Console warning:
// "ReactDOM.render is no longer supported in React 18.
//  Use createRoot instead. Until you switch to the new API,
//  your app will behave as if it's running React 17."
```

```tsx
// ✅ Yangi (R18+)
import { createRoot } from 'react-dom/client';

const container = document.getElementById('root');
if (!container) throw new Error('Root element topilmadi');
createRoot(container).render(<App />);
```

Multi-root setup (bir sahifada bir nechta React app — masalan, microfrontend yoki widget'lar):

```tsx
import { createRoot } from 'react-dom/client';

const widgetAContainer = document.getElementById('widget-a');
const widgetBContainer = document.getElementById('widget-b');
if (!widgetAContainer || !widgetBContainer) {
  throw new Error('Widget containerlari topilmadi');
}

const widgetA = createRoot(widgetAContainer);
const widgetB = createRoot(widgetBContainer);

widgetA.render(<UserWidget />);
widgetB.render(<NotificationWidget />);

// Har bir root mustaqil ishlaydi — alohida update queue, alohida render cycle
```

</details>

---

## Strict Mode

### Nazariya

`<StrictMode>` — React komponentlari uchun **development-only** tekshiruv vositasi. U **hech qanday UI render qilmaydi** va **production'da no-op** (umuman ishlamaydi). Vazifasi — kodda concurrent rendering bilan mos kelmaydigan **anti-pattern'larni topish**.

```tsx
import { StrictMode } from 'react';

createRoot(container).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

`<StrictMode>` development'da quyidagi qo'shimcha chaqiriqlarni o'tkazadi (faqat dev'da, prod'da no-op):

1. **Function komponent tanasi ikki marta chaqiriladi** (R18+ — R16.3'da bu faqat class konstruktor/render/shouldComponentUpdate/getDerivedStateFromProps uchun ishlardi; R16.8 hooks bilan function body double-invocation qo'shildi va R18'da effect cycle ham qo'shildi)
2. **`useState`/`useReducer` initializer funksiyalari ikki marta chaqiriladi** — initial state pure ekanligini tekshirish uchun
3. **`useState` setter va `useReducer` reducer funksiyalari ikki marta chaqiriladi** (functional update'lar) — pure state transition tekshirish uchun
4. **`useMemo` va `useCallback` factory funksiyalari ikki marta chaqiriladi** — pure derivation tekshirish uchun
5. **Effect'lar mount paytida `mount → cleanup → mount` cycle bajariladi** (R18+) — synchronization invariant'ini tekshirish uchun
6. **Deprecated API'lar haqida warning** — `findDOMNode`, legacy Context (`contextTypes`), string refs (oxirgi ikkitasi R19'da olib tashlandi)

### Nima uchun ikki marta chaqirish

Concurrent rendering'da React render'ni **uzib qo'yib, qayta boshlashi** mumkin (high-priority update kelganda). Ya'ni komponent funksiyasi va effect'lar bir nechta marta chaqirilishi mumkin. Agar ular **pure** va **idempotent** bo'lmasa — bug yuz beradi.

Strict Mode bu xatti-harakatni **dev'da majburan reproduce qiladi** — render davomida bug bo'lsa, u ikki marta chaqirilish tufayli aniq ko'rinadi.

**Misol — bug:**

```tsx
let totalRenders = 0; // ❌ tashqi mutable state

function Counter() {
  totalRenders++; // ❌ render davomida mutation
  return <div>Renders: {totalRenders}</div>;
}
```

```tsx
// ✅ To'g'ri — useRef render counter uchun (mutation faqat effect ichida)
function Counter() {
  const renderCount = useRef(0);
  useEffect(() => { renderCount.current += 1; });
  return <div>Renders: {renderCount.current}</div>;
}
```

Strict Mode'da birinchi versiya'da `totalRenders` har render'da **2 ga oshadi** — bu render'ning "side effect bilan" ekanligini darhol ko'rsatadi. Ikkinchi versiya — render pure, mutation effect ichida.

> **🕐 Versiya evolyutsiyasi (Strict Mode):**
> - **R16.3 (2018):** Strict Mode joriy etildi — class konstruktor, render, shouldComponentUpdate, getDerivedStateFromProps, useState initializer, useReducer reducer ikki marta chaqirilardi.
> - **R16.8 (2019):** Hooks bilan birga function komponent body ham ikki marta chaqiriladigan bo'ldi (purity check).
> - **R18 (2022):** Effect'lar uchun ham ikki marta cycle qo'shildi (`mount → cleanup → mount`). Effect "qaytadan o'rnatilishga chidamli" bo'lishi kerak — bu synchronization invariant'ini tekshirish uchun.
> - **Sabab:** Concurrent rendering invariant'larini dev'da topish. R18 effect 2x cycle "Reusable State" deb nomlangan kelajakdagi xususiyat (komponent unmount → remount qilinganda eski state'ni saqlab, qayta tiklash) uchun zamin yaratadi.

### StrictMode wrapper joylashuvi

`<StrictMode>` — komponent. Uni har qanday subtree'ga qo'llash mumkin:

```tsx
// Butun app uchun (eng keng tarqalgan)
<StrictMode>
  <App />
</StrictMode>

// Faqat ma'lum subtree uchun
<App>
  <StrictMode>
    <NewFeature /> {/* faqat shu yerda strict checks */}
  </StrictMode>
  <OldFeature />
</App>
```

Production build'da `<StrictMode>` komponent o'z joyida qoladi (bundle'dan olib tashlanmaydi), lekin **no-op** sifatida ishlaydi — barcha development-only tekshiruvlar (double invocation, deprecated API warnings, va h.k.) `process.env.NODE_ENV === 'production'` shartiga qarab o'tkazib yuboriladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Strict Mode'ning ichki mexanizmi:

```
Function komponent render (Strict Mode aktiv):
1. Komponent funksiyasi 1-marta chaqiriladi
   - hooks dispatcher orqali execute qilinadi
   - JSX qaytariladi (bu natija reconciliation'da ishlatilmaydi)
2. Komponent funksiyasi 2-marta chaqiriladi (DEV ONLY)
   - hooks bir xil saqlangan state bilan qayta execute
     (idempotent o'qish — yangi memoizedState yaratilmaydi)
   - JSX qaytariladi
3. Ikkinchi invocation natijasi reconciliation uchun ishlatiladi
   (purity qoidasi bo'yicha ikkala natija aynan bir xil bo'lishi shart —
    React solishtirib tekshirmaydi, faqat ikkinchisini ishlatadi)

React explicit purity tekshiruvi qilmaydi — agar render davomida
side effect bo'lsa (counter++, log, fetch), u 2x execute bo'lib
visible bo'lib qoladi. Bug ko'rinadi, lekin warning chiqarilmaydi.
```

R18 effect 2x cycle (mount holatida):

```
Mount cycle (Strict Mode dev'da):
1. Komponent mount qilinadi
2. useEffect setup chaqiriladi
3. useEffect cleanup chaqiriladi  ← Simulation: unmount-remount
4. useEffect setup chaqiriladi  ← qayta-mount
5. (browser paint)

Production'da:
1. Komponent mount qilinadi
2. useEffect setup chaqiriladi
3. (browser paint)
```

Bu cycle **faqat dev mode'da** va **faqat mount paytida**. Update yoki unmount holatlarida normal sikl ishlaydi.

**Strict Mode'ning ichki implementation:**

`<StrictMode>` Fiber'da `StrictLegacyMode | StrictEffectsMode` bayroqlari bilan belgilanadi. Reconciler render paytida Fiber'ning `mode` flag'ini tekshirib, ushbu Fiber subtree uchun double-invocation qoidalarini qo'llaydi.

```typescript
// React internal (soddalashtirilgan — manba: react/packages/react-reconciler/src/ReactTypeOfMode.js)
const NoMode = 0b0000000;
const ConcurrentMode = 0b0000001;
const ProfileMode = 0b0000010;
const DebugTracingMode = 0b0000100;   // bit 2 — enableDebugTracing experimental flag uchun
const StrictLegacyMode = 0b0001000;   // bit 3
const StrictEffectsMode = 0b0010000;  // bit 4
const NoStrictPassiveEffectsMode = 0b1000000;  // bit 6

if (fiber.mode & StrictLegacyMode && __DEV__) {
  // Komponent funksiyasini ikki marta chaqirish
  reactComponent(); // birinchi marta
  reactComponent(); // ikkinchi marta — side effect detection
}

if (fiber.mode & StrictEffectsMode && __DEV__) {
  // Effect lifecycle simulation
  effect.create();
  effect.destroy();
  effect.create();
}
```

**Strict Mode va Console:**

R17 va undan oldin Strict Mode'da ikkala invocation'ning `console.log` chiqishlari oddiy tarzda ko'rinardi (ikkilanish foydalanuvchini chalkashtirardi). R18'da yangi behavior qo'shildi — ikkinchi render'dagi `console.log/warn/error` chiqishlari **xira shrift bilan** dim qilinadi (suppression), React DevTools'da `Hide logs during second render in Strict Mode` opsiyasi orqali to'liq yashirish mumkin. R19'da bu standart sozlama saqlanib qoldi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Strict Mode topadigan bug — render side effect:

```tsx
// ❌ ANTI-PATTERN — render davomida tashqi state mutation
const cache = new Map<string, User>();

function UserProfile({ userId, fallbackUser }: { userId: string; fallbackUser: User }) {
  // Render ichida cache update — Strict Mode'da 2 marta yuz beradi
  if (!cache.has(userId)) {
    console.log('Cache miss for', userId);
    cache.set(userId, fallbackUser);  // ❌ tashqi Map'ni mutate qilish
  }

  const user = cache.get(userId);
  return <div>{user?.name}</div>;
}

// Strict Mode'da log:
// "Cache miss for user-123" ← 1-render
// "Cache miss for user-123" ← 2-render (bug ko'rinadi — render side effect bilan)
```

```tsx
// ✅ TO'G'RI — useState lazy initializer + useEffect
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    let cancelled = false;
    fetchUser(userId).then(u => {
      if (!cancelled) setUser(u);
    });
    return () => { cancelled = true; };
  }, [userId]);

  if (!user) return <div>Yuklanmoqda...</div>;
  return <div>{user.name}</div>;
}
```

Strict Mode topadigan bug — effect cleanup yo'q:

```tsx
// ❌ ANTI-PATTERN — subscription cleanup yo'q
function ChatRoom({ roomId }: { roomId: string }) {
  useEffect(() => {
    const conn = connectToChat(roomId);
    // ❌ cleanup return YO'Q
  }, [roomId]);

  return <div>Chat: {roomId}</div>;
}

// Strict Mode dev'da nima bo'ladi:
// 1. Mount → connectToChat() chaqiriladi → 1 ta connection
// 2. SIMULATED unmount → cleanup yo'q → connection ochiq qoladi
// 3. SIMULATED remount → connectToChat() yana → 2 ta connection
// Result: 2 ta yashirin connection (memory leak)
```

```tsx
// ✅ TO'G'RI — cleanup return bilan
function ChatRoom({ roomId }: { roomId: string }) {
  useEffect(() => {
    const conn = connectToChat(roomId);
    return () => conn.disconnect(); // ✅ cleanup
  }, [roomId]);

  return <div>Chat: {roomId}</div>;
}

// Strict Mode dev'da:
// 1. Mount → connect → 1 ta connection
// 2. Cleanup → disconnect → 0 ta connection
// 3. Remount → connect → 1 ta connection
// Result: 1 ta connection (synchronization invariant'i bajarildi)
```

Conditional Strict Mode (faqat ma'lum subtree uchun):

```tsx
function App() {
  return (
    <Layout>
      <Header />  {/* No strict checks */}
      <StrictMode>
        <ExperimentalFeature />  {/* Strict checks */}
      </StrictMode>
      <Footer />  {/* No strict checks */}
    </Layout>
  );
}
```

Production behavior:

```tsx
// Bu kod dev va prod'da farqli ishlaydi
function Counter() {
  const [count, setCount] = useState(0);
  console.log('Render');
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

createRoot(container).render(
  <StrictMode>
    <Counter />
  </StrictMode>
);

// Dev: "Render" ikki marta chiqadi (Strict Mode 2x)
// Prod: "Render" bir marta chiqadi (Strict Mode no-op)
```

</details>

---

## Initial Render vs Re-render

### Nazariya

React rendering ikki holatga bo'linadi:

| Holat | Qachon | Nima qiladi |
|-------|--------|-------------|
| **Initial Render (Mount)** | Komponent birinchi marta tree'ga qo'shilganda | Yangi Fiber tree quradi, DOM yaratadi, mount effect'lari ishga tushadi |
| **Re-render (Update)** | State, props, context o'zgarganda | Eski tree'ni yangi bilan diff qiladi, faqat o'zgarganini yangilaydi, update effect'lari ishga tushadi |

**Re-render trigger sabablari:**

1. **Local state o'zgargan** — `useState` yoki `useReducer` setter chaqirilgan va yangi qiymat eskidan farq qiladi (`Object.is` solishtiruv)
2. **Props o'zgargan** — parent re-render qilingan va yangi props uzatgan
3. **Parent re-render bo'lgan** — agar komponent `React.memo` bilan o'rab olinmagan bo'lsa, parent re-render avtomatik child re-render keltiradi
4. **Context value o'zgargan** — `useContext` ishlatuvchi har bir komponent re-render qilinadi
5. **`forceUpdate()`** — class komponentdagi instance metod (kamdan-kam ishlatiladi)

**Re-render YO'Q:**
- State setter eski qiymat bilan chaqirilsa (`setCount(0)` agar `count === 0`) — React `Object.is` orqali tekshiradi va bailout qiladi
- `React.memo` o'ralgan komponent shallow-equal props oladi
- `useMemo` / `useCallback` natija qiymati bir xil reference

### Mount vs Update farqi (hooks ichida)

`useState` ikki yo'l bilan ishlaydi:

```tsx
function Counter() {
  // Mount paytida: initial value (0) o'rnatiladi
  // Update paytida: oldingi qiymat saqlangan joydan o'qiladi
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

Mount: `mountState(0)` — Fiber'ning `memoizedState` linked list'iga yangi hook obyekti qo'shiladi, initial qiymat `0` saqlanadi.

Update: `updateState()` — mavjud hook obyekti topiladi (call order'iga qarab), update queue'dan yangi qiymat qo'llanadi.

Bu mexanizm `15-hooks-fundamentals.md` da chuqur yoritiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Initial render jarayoni:

```
1. root.render(<App />) chaqiriladi
   ↓
2. updateContainer() — root fiber'ga update qo'shiladi
   ↓
3. performWorkOnRoot() — render boshlanadi
   ↓
4. beginWork() — birinchi fiber (HostRoot) ishlanadi
   ↓
5. <App /> uchun yangi Fiber yaratiladi
   - tag: FunctionComponent
   - memoizedState: null (hali hook'lar ishlamagan)
   ↓
6. App() chaqiriladi (komponent funksiyasi)
   - hooks bajariladi (mountState, mountEffect, ...)
   - JSX qaytariladi
   ↓
7. Reconciliation: JSX'dagi har element uchun child Fiber yaratiladi
   ↓
8. Recursive ravishda har child uchun 4-7 qadamlar
   ↓
9. completeWork() — host fiber'lar uchun host instance (DOM node) yaratiladi
   - createInstance() — react-dom Host Config
   - properties set qilinadi (className, onClick, ...)
   ↓
10. Commit Phase boshlanadi (alohida bo'limda)
   - Mutation: parent.appendChild(child) — DOM tree quriladi
   - Layout: useLayoutEffect, refs attached
   - (paint)
   - Passive: useEffect (async)
```

Re-render jarayoni:

```
1. setState() chaqiriladi
   ↓
2. enqueueUpdate() — update queue'ga qo'shiladi
   ↓
3. scheduleUpdateOnFiber() — Scheduler'ga ish rejalashtiriladi
   ↓
4. performWorkOnRoot() — render boshlanadi
   ↓
5. beginWork() — bu Fiber uchun
   ↓
6. Komponent funksiyasi qayta chaqiriladi
   - updateState() — yangi qiymat update queue'dan olinadi
   - hooks call order saqlanadi (memoizedState linked list)
   ↓
7. Yangi JSX qaytariladi
   ↓
8. Reconciliation (beginWork ichida): yangi JSX vs eski Fiber children
   - Same type → update existing fiber
   - Different type → unmount old, mount new
   - Keyed list → match by key
   - Placement/ChildDeletion flag'lari shu yerda o'rnatiladi
     (placeChild yangi/ko'chgan child uchun, deleteChild esa parent'ga)
   ↓
9. completeWork() — host fiber uchun Update flag o'rnatiladi (markUpdate),
   agar props o'zgargan bo'lsa; subtreeFlags pastdan yuqoriga bubble qilinadi
   ↓
10. Commit Phase
   - Mutation: faqat o'zgargan DOM operations
   - Layout: useLayoutEffect cleanup → setup, refs re-attached
   - (paint)
   - Passive: useEffect cleanup → setup
```

**Bailout — re-render skip qilish sabablari:**

1. **State Object.is equal:** `setState(currentValue)` — React `Object.is(prev, next)` solishtiradi, agar teng bo'lsa render skip qiladi
2. **React.memo:** props shallow-equal bo'lsa, child render qilinmaydi
3. **Context bailout:** `useContext` provider value o'zgarmasa
4. **No update queue:** komponent uchun pending update yo'q va parent shu komponentdan past tushgan bo'lsa

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Mount va Update farqini ko'rsatuvchi misol:

```tsx
function Logger({ label }: { label: string }) {
  console.log(`Render: ${label}`);

  useEffect(() => {
    console.log(`Mount: ${label}`);
    return () => console.log(`Unmount: ${label}`);
  }, []); // empty deps — faqat mount/unmount

  useEffect(() => {
    console.log(`Update: ${label}`);
  }); // no deps — har render

  return <div>{label}</div>;
}

function App() {
  const [show, setShow] = useState(true);
  const [count, setCount] = useState(0);

  return (
    <div>
      {show && <Logger label="A" />}
      <button onClick={() => setShow(s => !s)}>Toggle</button>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
    </div>
  );
}

// Initial mount:
// "Render: A"
// "Mount: A"
// "Update: A"

// "Count" tugmasi bosildi (state o'zgardi, lekin Logger ham re-render):
// "Render: A"
// "Update: A"

// "Toggle" tugmasi bosildi (Logger unmount):
// "Unmount: A"

// "Toggle" yana bosildi (Logger qayta mount):
// "Render: A"
// "Mount: A"
// "Update: A"
```

State setter Object.is bailout misoli:

```tsx
function StableState() {
  const [user, setUser] = useState({ name: 'Ali', age: 25 });

  console.log('Render');

  return (
    <div>
      <button onClick={() => setUser({ name: 'Ali', age: 25 })}>
        Set same value (object literal)
      </button>
      <button onClick={() => setUser(user)}>
        Set same reference
      </button>
      <button onClick={() => setUser({ name: 'Bob', age: 30 })}>
        Set different
      </button>
    </div>
  );
}

// "Set same value" bosildi:
// "Render" CHIQADI ← har bosishda yangi obyekt (yangi reference), Object.is false

// "Set same reference" bosildi:
// "Render" CHIQMAYDI ← bir xil reference, Object.is true → bailout

// "Set different" bosildi:
// "Render" CHIQADI ← yangi qiymat
```

`React.memo` bilan re-render bailout:

```tsx
interface ChildProps {
  value: number;
}

const Child = React.memo(function Child({ value }: ChildProps) {
  console.log('Child render');
  return <div>Value: {value}</div>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const [unrelated, setUnrelated] = useState('');

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <input value={unrelated} onChange={e => setUnrelated(e.target.value)} />
      <Child value={count} />
    </div>
  );
}

// "Count" bosildi → Child render: ✅ chiqadi (value o'zgargan)
// Input'ga yozildi → Child render: ❌ chiqmaydi (value o'zgarmagan, React.memo bailout)
```

</details>

---

## Render Phase

### Nazariya

**Render Phase** — React'ning birinchi asosiy bosqichi. Bu bosqichda React komponent funksiyalarini chaqiradi, JSX qaytarib oladi, va **workInProgress** Fiber tree'ni quradi yoki yangilaydi. Render Phase **eski tree'ni o'zgartirmaydi**, **DOM'ga tegmaydi**, va **uziluvchi** (interruptible) bo'lishi mumkin (concurrent mode'da).

**Render Phase'ning asosiy invariant'lari (Rules of React):**

1. **Pure / Deterministic** — bir xil props/state/context bilan har doim bir xil JSX qaytariladi (`Math.random`, `Date.now` kabi non-deterministic manbalar ishlatilmaydi)
2. **Side-effect free** — render davomida DOM mutation, fetch, subscribe (`addEventListener` va h.k.), boshqa komponent'da `setState`, tashqi mutable state'ga yozish — ishlatilmaydi
3. **Read-only** — props, state, context, ref — faqat o'qiladi, mutate qilinmaydi. `ref.current` render davomida o'qilmasligi va yozilmasligi kerak (istisno: lazy initialization pattern, `ref.current ??= initialValue`); boshqa render holatlarida ref qiymati commit'dan oldin/keyin farq qiladi (non-deterministic)
4. **Idempotent va Restartable** — render qancha marta chaqirilsa, natija bir xil. Concurrent rendering high-priority update kelganda joriy render'ni tashlab, qaytadan boshlashi mumkin

Bu invariant'lar **concurrent rendering uchun zarur**. Render uziluvchi bo'lgani uchun, React render'ni **bir nechta marta chaqirishi** mumkin (high-priority update kelganda eski render tashlanib, qayta boshlanadi). Agar render side effect bilan bo'lsa — bu effect'lar bir nechta marta yuz beradi (yoki tashlangan render'dagi effect'lar "yo'qolib ketadi").

**Render Phase'da nima qilish mumkin:**
- ✅ Hooks chaqirish (top-level)
- ✅ Local variable yaratish
- ✅ Pure hisob-kitoblar (`const sum = a + b`)
- ✅ JSX qaytarish
- ✅ Props/state/context o'qish

**Render Phase'da nima qilish TAQIQ:**
- ❌ DOM manipulatsiyasi (`document.querySelector`, `element.style = ...`)
- ❌ `fetch`, `XMLHttpRequest`, async ish
- ❌ `setState` (boshqa komponent'da yoki render davomida — exception: lazy init)
- ❌ Subscription qo'shish (`addEventListener`, va h.k.)
- ❌ `Math.random()`, `Date.now()` — render davomida o'qish
- ❌ Tashqi mutable state'ni o'zgartirish

Lazy init exception:

```tsx
// ✅ Lazy init useState'da OK — faqat birinchi render'da chaqiriladi
const [data, setData] = useState(() => expensiveComputation());

// ✅ "Render-phase update" — useState'ni o'zining ichida o'zgartirish OK
// (faqat o'sha komponent'ning o'z setState'i; boshqa komponent setState'i — runtime error)
function Counter({ initial }: { initial: number }) {
  const [count, setCount] = useState(initial);
  const [prevInitial, setPrevInitial] = useState(initial);

  // Derived state pattern: parent prop o'zgarsa, lokal state'ni reset qilish
  if (initial !== prevInitial) {
    setPrevInitial(initial);
    setCount(initial);
    // ❌ `return null` shart EMAS — React bu render natijasini avtomatik tashlaydi
    //    va yangi state qiymatlari bilan darhol qayta render qiladi.
  }

  return <div>{count}</div>;
}
```

Lekin bu pattern kamdan-kam kerak — odatda derived state'ni hisob-kitob orqali olish yaxshi.

### Render Phase pipeline

```
1. setState chaqiriladi (yoki initial render)
   ↓
2. Scheduler render'ni rejalashtiradi (priority bilan)
   ↓
3. workLoop boshlanadi
   ↓
4. Har Fiber uchun beginWork():
   - FunctionComponent → komponent funksiyasini chaqirish
   - Hooks bajariladi (memoizedState linked list)
   - JSX qaytariladi
   - reconcileChildren() — yangi child Fiber'lar yaratiladi
   ↓
5. Recursive ravishda child'larga tushish
   ↓
6. completeWork() — har Fiber uchun:
   - Host element bo'lsa (DOM node) — instance yaratish (yo'q bo'lsa)
   - Properties diff (yangi vs eski props), update payload tayyorlash
   - Host'da props o'zgargan bo'lsa Update flag (markUpdate); subtreeFlags bubble
   - (Placement/ChildDeletion flag'lari beginWork/reconciliation'da o'rnatilgan)
   ↓
7. Yuqoriga qaytish (sibling yoki parent.return)
   ↓
8. Root tugagandan keyin Commit Phase
```

<details>
<summary><strong>Under the Hood</strong></summary>

Render Phase work loop (concurrent mode'da):

```typescript
// React internal (soddalashtirilgan)
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function shouldYield(): boolean {
  // Browser'ga yo'l berish vaqti keldi mi?
  // - 5ms budget tugadimi? (`frameYieldMs`, manba: scheduler/src/forks/Scheduler.js)
  // - High-priority input bormi? (R18+ `isInputPending`)
  return performance.now() >= deadline;
}
```

`performUnitOfWork` har Fiber'ni "ishlaydi" — bu **bitta unit**:

```typescript
function performUnitOfWork(fiber: Fiber) {
  // 1. beginWork — komponent funksiyasini chaqirish, child Fiber'lar yaratish
  let next = beginWork(fiber, ...);
  
  if (next === null) {
    // 2. Bu fiber'ning child'lari yo'q — completeWork (yuqoriga)
    completeUnitOfWork(fiber);
  } else {
    // 3. Child bor — pastga tushish
    workInProgress = next;
  }
}
```

Concurrent mode'da `frameYieldMs = 5` ms budget tugagach `shouldYield()` `true` qaytaradi va React work loop'ni to'xtatadi (manba: `react/packages/scheduler/src/forks/Scheduler.js`). Browser tasks (input, animation) ishlash imkonini oladi. Keyin `MessageChannel.postMessage` orqali React qayta uyg'onadi va qolgan ishni davom ettiradi.

**Render davomida side effect — nima sodir bo'ladi:**

Misol — render ichida `fetch` chaqirilgan:

```tsx
// ❌ Anti-pattern — render ichida fetch
function UserProfile({ userId }: { userId: string }) {
  fetch(`/api/users/${userId}`);  // ← har render'da yangi request
  return <div>Loading...</div>;
}
```

Concurrent mode'da:
1. Render boshlanadi → `fetch` chaqiriladi (request 1)
2. High-priority update keladi → render uziladi
3. Render qayta boshlanadi → `fetch` qayta chaqiriladi (request 2)
4. Uzilish necha marta yuz bersa, request soni ham shuncha oshadi — son
   oldindan aniqlanmaydi (concurrent restart soni o'zgaruvchan). Strict Mode dev'da
   komponent body 2x chaqirilgani uchun request soni yana ko'payadi.

```tsx
// ✅ To'g'ri pattern — useEffect (yoki R19 `use()` + Suspense)
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  useEffect(() => {
    let cancelled = false;
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then((data: User) => { if (!cancelled) setUser(data); });
    return () => { cancelled = true; };
  }, [userId]);
  return user ? <div>{user.name}</div> : <div>Loading...</div>;
}
```

**Reconciliation render phase ichida:**

Render Phase'da **diffing** ishi ham bajariladi (Reconciler algoritmi). Lekin bu **DOM'ga tegmaydi** — faqat workInProgress Fiber tree'da `flags` (Placement, Update, ChildDeletion) bayroqlari o'rnatiladi. Real DOM mutation Commit Phase'da bo'ladi.

Reconciliation algoritmi `04-reconciliation.md` da chuqur yoritiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Pure render — to'g'ri yondashuv:

```tsx
// ✅ Pure — bir xil props bilan bir xil natija
interface ProductCardProps {
  product: { name: string; price: number; discount?: number };
}

function ProductCard({ product }: ProductCardProps) {
  // Pure hisob-kitob OK
  const finalPrice = product.discount
    ? product.price * (1 - product.discount)
    : product.price;
  
  return (
    <div>
      <h3>{product.name}</h3>
      <p>Narx: {finalPrice.toFixed(2)}</p>
    </div>
  );
}
```

Render-phase side effect — anti-pattern:

```tsx
// ❌ ANTI-PATTERN — render davomida fetch
function UserList() {
  const [users, setUsers] = useState<User[]>([]);
  
  fetch('/api/users')  // ❌ render davomida network request
    .then(r => r.json())
    .then(setUsers);  // ❌ render davomida setState
  
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
// Bu kod infinite loop keltiradi: fetch → setState → re-render → fetch → ...
```

```tsx
// ✅ TO'G'RI — useEffect ichida
function UserList() {
  const [users, setUsers] = useState<User[]>([]);
  
  useEffect(() => {
    fetch('/api/users')
      .then(r => r.json())
      .then(setUsers);
  }, []); // empty deps — faqat mount'da
  
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

Render-phase mutable read — anti-pattern:

```tsx
// ❌ Date.now() — render davomida o'qish
function Timestamp() {
  const now = Date.now();  // ❌ har render'da boshqa qiymat
  return <div>Hozir: {now}</div>;
}
// Render Phase pure bo'lishi shart — bir xil props/state bilan bir xil natija.
// Date.now() har chaqiruvda turli qiymat beradi → render non-deterministic.
// Concurrent restart yoki Strict Mode 2x invocation paytida natija mos kelmaydi.
```

```tsx
// ✅ Snapshot — useState yoki useSyncExternalStore
function Timestamp() {
  const [now, setNow] = useState(() => Date.now()); // lazy init
  
  useEffect(() => {
    const id = setInterval(() => setNow(Date.now()), 1000);
    return () => clearInterval(id);
  }, []);
  
  return <div>Hozir: {now}</div>;
}
```

Pure derived state vs unnecessary state:

```tsx
// ❌ Derived state'ni alohida useState'da saqlash
function ProductList({ products }: { products: Product[] }) {
  const [count, setCount] = useState(products.length);  // ❌ duplikat
  
  useEffect(() => {
    setCount(products.length);  // ❌ extra render kerak
  }, [products]);
  
  return <div>Soni: {count}</div>;
}
```

```tsx
// ✅ Pure derivation — render davomida hisoblash
function ProductList({ products }: { products: Product[] }) {
  const count = products.length;  // ✅ pure hisob
  return <div>Soni: {count}</div>;
}
```

</details>

---

## Commit Phase

### Nazariya

**Commit Phase** — React'ning ikkinchi asosiy bosqichi. Render Phase tugaganidan keyin (workInProgress tree to'liq tayyor) Commit Phase boshlanadi. Bu bosqichda React **real DOM'ga mutation'lar qo'llaydi**, **refs attach/detach qiladi**, va **lifecycle callback'larini chaqiradi**.

**Asosiy xususiyatlar:**

- **Synchronous** — uziluvchi emas (concurrent mode'da ham)
- **Atomik** — boshlandimi, to'liq tugaydi (yarim holat yo'q)
- **Side-effects ruxsat etilgan** — DOM mutation, ref'lar, callback'lar
- **Tezkor** — DOM operations imkon qadar minimal qilinadi (faqat farq)

Commit Phase **uchta sub-phase'ga** bo'linadi:

| Sub-phase | Nima bo'ladi |
|-----------|--------------|
| **1. Before Mutation** | `getSnapshotBeforeUpdate` (legacy class), eski DOM holati capture qilinadi |
| **2. Mutation** | DOM o'zgartiriladi (insert, update, delete), refs detach qilinadi (eski node'lardan) |
| **3. Layout** | Refs attach qilinadi (yangi node'larga), `useLayoutEffect`, `componentDidMount/Update` chaqiriladi |

Commit Phase tugagandan **keyin** (paint'dan keyin):
- **Passive effects** — `useEffect` async chaqiriladi

### Commit Phase mexanikasi

**1. Before Mutation Phase:**

Bu sub-phase asosan **legacy class component'lar uchun** kerak. `getSnapshotBeforeUpdate(prevProps, prevState)` chaqiriladi — bu metod DOM yangilanishidan oldin **eski DOM holatini capture qiladi**.

Tipik ishlatilish: scroll position'ni saqlab qolish (DOM o'zgargandan keyin tiklash uchun).

```tsx
// Legacy class — kamdan-kam ishlatiladi
class ChatList extends React.Component {
  getSnapshotBeforeUpdate(prevProps) {
    // DOM o'zgartirilishidan oldin scroll holati saqlanadi
    if (prevProps.messages.length < this.props.messages.length) {
      return this.containerRef.current.scrollHeight;
    }
    return null;
  }

  componentDidUpdate(prevProps, prevState, snapshot) {
    // DOM yangilangandan keyin scroll'ni tiklash
    if (snapshot !== null) {
      this.containerRef.current.scrollTop =
        this.containerRef.current.scrollHeight - snapshot;
    }
  }

  render() { /* ... */ }
}
```

Function komponent'larda bu pattern `useLayoutEffect` bilan amalga oshiriladi (DOM allaqachon yangilangan, lekin paint'dan oldin) yoki `useRef` + `useLayoutEffect` kombinatsiyasi bilan.

**2. Mutation Phase:**

Bu sub-phase'da **real DOM o'zgaradi**:

- `Placement` flag'li Fiber'lar uchun: yangi DOM element yaratiladi va parent'ga `appendChild` qilinadi
- `Update` flag'li Fiber'lar uchun: properties yangilanadi (`element.className = newValue`)
- `ChildDeletion` flag'li parent'lar uchun: o'chiriladigan child'lar `deletions` ro'yxatidan olinib, `removeChild` qilinadi va cleanup'lar (effects) chaqiriladi
- **Refs detach** — eski DOM node'lardan ref'lar yechiladi (`ref.current = null`)

Bu phase'da DOM **inconsistent state'da** bo'ladi — ba'zi node'lar yangi, ba'zilari hali eski. Shuning uchun bu phase'da DOM'ni o'qish (masalan, `getBoundingClientRect`) **xatarli** — siz nimani o'qiyapsiz aniq emas.

**3. Layout Phase:**

Bu sub-phase'da DOM butunlay yangilangan, lekin browser hali paint qilmagan. Quyidagilar bajariladi:

- **Refs attach** — yangi DOM node'larga ref'lar bog'lanadi (`ref.current = newDOMNode`)
- **`useLayoutEffect`** callbacks chaqiriladi (cleanup → setup tartibida)
- **Class lifecycle**: `componentDidMount` (yangi komponent'lar) yoki `componentDidUpdate` (mavjud komponent'lar)

Layout Phase **synchronous** — to'liq tugamaguncha browser paint qilmaydi. Shu sababli bu phase'da DOM measurement (`offsetWidth`, `getBoundingClientRect`) xavfsiz va aniq.

**Layout Phase tugagandan keyin:**
1. Browser paint qilish imkoniyatini oladi (foydalanuvchi yangilangan UI'ni ko'radi)
2. **Passive effects** queue'dan `useEffect` callback'lari async chaqiriladi: Scheduler `scheduleCallback(NormalPriority, flushPassiveEffects)` orqali rejalashtiradi (`MessageChannel.postMessage` bilan yangi task ochiladi). Paint passive effects'dan oldin yoki keyin yuz berishi browser qaroriga bog'liq (React faqat **commit-before-passive** kafolatini beradi, **paint-before-passive**'ni emas).

<details>
<summary><strong>Under the Hood</strong></summary>

Commit Phase to'liq pipeline:

```
Render Phase tugadi → workInProgress tree tayyor

┌─────────────────────────────────────────┐
│  0. commitRootImpl boshida               │
│  - Oldingi commit'ning pending passive   │
│    effect'lari flush qilinadi (agar      │
│    bo'lsa) — flushPassiveEffects()       │
│    sub-phase'lardan OLDIN, eng yuqorida  │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│  1. BEFORE MUTATION PHASE               │
│  - commitBeforeMutationEffects()        │
│  - getSnapshotBeforeUpdate (class)      │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│  2. MUTATION PHASE                      │
│  - commitMutationEffects()              │
│  - Placement: appendChild, insertBefore │
│  - Update: setAttribute, textContent    │
│  - ChildDeletion: removeChild + cleanup │
│  - Refs detach (eski → null)            │
│  ↓                                      │
│  current = workInProgress (tree swap)   │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│  3. LAYOUT PHASE                        │
│  - commitLayoutEffects()                │
│  - Refs attach (yangi DOM node'larga)   │
│  - useLayoutEffect (cleanup → setup)    │
│  - componentDidMount / Update           │
└──────────────┬──────────────────────────┘
               ↓
       BROWSER PAINT (foydalanuvchi ko'radi)
               ↓
┌─────────────────────────────────────────┐
│  4. PASSIVE EFFECTS (async)             │
│  - flushPassiveEffects()                │
│  - useEffect (cleanup → setup)          │
└─────────────────────────────────────────┘
```

**Tree swap** — Mutation Phase oxirida `fiberRoot.current = workInProgress`. Bu — double buffering. Eski tree (current edi) endi alternate sifatida saqlanadi (keyingi render uchun reuse qilinadi). Cross-ref `03-fiber-architecture.md`.

**Effect list traversal:**

React har sub-phase'da **effect list** bo'ylab yuradi. Effect list — fiber tree ichidagi `flags` (effect bayroqlari) bo'lgan Fiber'larning ro'yxati. Bu list `subtreeFlags` orqali optimize qilingan — effect yo'q subtree butunlay skip qilinadi.

```typescript
// React internal (soddalashtirilgan — R18+ recursive traversal)
function commitMutationEffectsOnFiber(finishedWork: Fiber) {
  // subtreeFlags — child'lar ichida mutation bormi? Bo'lmasa butun subtree skip
  if (finishedWork.subtreeFlags & MutationMask) {
    let child = finishedWork.child;
    while (child !== null) {
      commitMutationEffectsOnFiber(child);
      child = child.sibling;
    }
  }

  // Shu fiber'ning o'z flag'lari
  if (finishedWork.flags & Placement) {
    commitPlacement(finishedWork);
  }
  if (finishedWork.flags & Update) {
    commitUpdate(finishedWork);
  }
  // ChildDeletion, Ref, ...
}
```

**Layout Phase synchronously blocks paint:**

Browser rendering pipeline:

```
JavaScript executes
    ↓
React Commit Phase (mutation + layout)
    ↓
Style recalculation (browser)
    ↓
Layout (browser)
    ↓
Paint (foydalanuvchi ko'radi)
    ↓
Composite
    ↓
JavaScript continues (passive effects, useEffect)
```

`useLayoutEffect` Layout Phase'da bajarilgani uchun **paint'dan oldin** ishlaydi. Agar bu effect ichida `setState` chaqirilsa — React **darhol qayta render qilib, qayta commit qiladi** (ikkinchi cycle), va faqat shundan keyin paint qiladi. Bu xulq-atvor flicker oldini olishga yordam beradi (UI bir holatda ko'rinib, keyin boshqasiga sakrab o'tmaydi).

`useEffect` esa Passive Phase'da (paint'dan keyin) ishlaydi. `useEffect` ichida `setState` qilish — yangi render+commit cycle'ni keltiradi, lekin foydalanuvchi avval eski natijani ko'rib turadi.

Shu sababli:
- DOM measurement → `useLayoutEffect` (paint'dan oldin, flicker yo'q)
- Subscription, fetch, timer → `useEffect` (paint'dan keyin, paint'ni bloklamaydi)

Cross-ref `16-useeffect.md` va `17-uselayouteffect.md`.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Commit Phase fazalarining ketma-ketligini ko'rsatuvchi misol:

```tsx
function Demo() {
  const [count, setCount] = useState(0);
  const ref = useRef<HTMLDivElement>(null);

  console.log('1. Render Phase: komponent funksiyasi');

  useLayoutEffect(() => {
    console.log('3. Layout Phase: useLayoutEffect setup');
    console.log('   ref.current:', ref.current); // ✅ DOM mavjud
    return () => console.log('3. Layout Phase: useLayoutEffect cleanup');
  });

  useEffect(() => {
    console.log('4. Passive Phase: useEffect setup (paint dan keyin)');
    return () => console.log('4. Passive Phase: useEffect cleanup');
  });

  return (
    <div ref={ref}>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

// Initial mount log:
// 1. Render Phase: komponent funksiyasi
// 3. Layout Phase: useLayoutEffect setup
//    ref.current: <div>...</div>
//    [browser paint]
// 4. Passive Phase: useEffect setup (paint dan keyin)

// Increment bosildi:
// 1. Render Phase: komponent funksiyasi
// 3. Layout Phase: useLayoutEffect cleanup
// 3. Layout Phase: useLayoutEffect setup
//    [browser paint]
// 4. Passive Phase: useEffect cleanup
// 4. Passive Phase: useEffect setup
```

useLayoutEffect — DOM measurement (paint'dan oldin):

```tsx
// ✅ Tooltip pozitsiyasini DOM'dan o'lchash
function Tooltip({ targetRef, text }: TooltipProps) {
  const tooltipRef = useRef<HTMLDivElement>(null);
  const [position, setPosition] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    if (!targetRef.current || !tooltipRef.current) return;

    const targetRect = targetRef.current.getBoundingClientRect();
    const tooltipRect = tooltipRef.current.getBoundingClientRect();

    setPosition({
      top: targetRect.bottom + 8,
      left: targetRect.left + (targetRect.width - tooltipRect.width) / 2,
    });
  }, [targetRef]);

  return (
    <div
      ref={tooltipRef}
      style={{ position: 'fixed', top: position.top, left: position.left }}
    >
      {text}
    </div>
  );
}

// useEffect ishlatilsa — tooltip avval (0,0) da ko'rinadi (flicker), keyin to'g'ri pozitsiyaga ko'chadi
// useLayoutEffect — pozitsiya paint'dan oldin to'g'rilanadi, flicker yo'q
```

Cleanup → setup tartibi:

```tsx
function Subscription({ id }: { id: string }) {
  useEffect(() => {
    console.log('Subscribe:', id);
    const unsubscribe = subscribe(id);
    return () => {
      console.log('Unsubscribe:', id);
      unsubscribe();
    };
  }, [id]);

  return <div>ID: {id}</div>;
}

// id "user-1" dan "user-2" ga o'zgardi:
// 1. Render Phase: yangi JSX
// 2. Mutation Phase: text node "ID: user-1" → "ID: user-2"
// 3. Layout Phase: useLayoutEffect (yo'q bu komponent'da)
//    [paint]
// 4. Passive Phase:
//    "Unsubscribe: user-1"  ← cleanup
//    "Subscribe: user-2"     ← setup
```

Refs attach/detach timing:

```tsx
function FocusInput() {
  const inputRef = useRef<HTMLInputElement>(null);
  const [show, setShow] = useState(true);

  useEffect(() => {
    console.log('useEffect ref:', inputRef.current);
    // show=true: <input> mavjud → ref.current = HTMLInputElement
    // show=false: <input> mavjud emas → ref.current = null (detach bo'lgan)
  });

  return (
    <>
      <button onClick={() => setShow(s => !s)}>Toggle</button>
      {show && <input ref={inputRef} />}
    </>
  );
}

// show=true → false:
// Mutation Phase: input DOM o'chirildi, ref.current = null (detach)
// Layout Phase: ref attach yangi node'ga (yo'q — input yo'q)
// Passive Phase: useEffect chaqiriladi
//    log: "useEffect ref: null"

// show=false → true:
// Mutation Phase: input qayta yaratildi
// Layout Phase: ref attach yangi input'ga
// Passive Phase: useEffect
//    log: "useEffect ref: <input>"
```

</details>

---

## Effects Timing — Passive vs Layout

### Nazariya

React'da **uch xil effect timing** mavjud:

| Hook | Phase | Timing | Use case |
|------|-------|--------|----------|
| **`useInsertionEffect`** | Mutation Phase ichida (DOM mutation'dan oldin) | Refs hali attach qilinmagan, `setState` ishlatib bo'lmaydi | CSS-in-JS library'lar uchun (stylesheet inject) |
| **`useLayoutEffect`** | Layout Phase | DOM mutation'dan **keyin**, paint'dan **oldin**, refs attach qilingan | DOM measurement, scroll position, focus |
| **`useEffect`** | Passive Phase | Commit'dan keyin async (paint'dan oldin yoki keyin — browser qaroriga bog'liq) | Subscription, fetch, timer, log |

**Vizual timeline:**

```
Render Phase (pure)
  ↓
Commit Phase:
  ├─ useInsertionEffect (CSS injection)
  ├─ DOM Mutation
  └─ useLayoutEffect + Refs attach + componentDidMount
  ↓
Browser Paint
  ↓
useEffect (passive, async)
```

### Qachon qaysi:

**`useEffect` (default tanlov):**
- Foydalanuvchi paint'ni ko'rishi muhim, effect kechiktirish mumkin
- Network requests (`fetch`)
- Subscriptions (`addEventListener`, WebSocket)
- Timers (`setInterval`, `setTimeout`)
- Log, analytics
- Side effect'lar UI'ga visual ravishda ta'sir qilmaydi

**`useLayoutEffect` (DOM measurement uchun):**
- Foydalanuvchi flicker'ni ko'rmasligi kerak
- DOM o'lchovi (`getBoundingClientRect`, `offsetWidth`, `scrollTop`)
- Scroll position'ni saqlash/tiklash
- Tooltip, popover, modal pozitsiyalash
- Focus management (yangi mount qilingan input'ga focus)
- Animation triggers (DOM mutation'dan darhol keyin)

**`useInsertionEffect` (kamdan-kam, faqat library'lar uchun):**
- CSS-in-JS library'lar (`styled-components`, `emotion`) ichki ishlatiladi
- Application code'da deyarli ishlatilmaydi

### Performance ta'siri

`useLayoutEffect` **paint'ni bloklaydi** — agar effect ichida og'ir hisob bo'lsa, foydalanuvchi UI muzlab qolganini sezadi. `useEffect` esa paint'dan keyin ishlaydi — foydalanuvchi UI'ni ko'rib bo'lib, keyin effect ishga tushadi.

Default tanlov — `useEffect`. `useLayoutEffect` faqat **DOM measurement** yoki **synchronous DOM mutation** kerak bo'lganda ishlatiladi.

> **Eslatma:** `useEffect` async ishlasa-da, **bir render uchun barcha useEffect'lar ketma-ket** chaqiriladi. Async — bu render+commit'dan ajratilgan, lekin har bir cycle ichida deterministik tartibda.

<details>
<summary><strong>Under the Hood</strong></summary>

`useEffect` ning passive flag'i:

```typescript
// React internal (soddalashtirilgan — manba: ReactFiberHooks.js)
function mountEffect(create, deps) {
  return mountEffectImpl(
    Passive | PassiveStatic,                // Fiber flags (ReactFiberFlags.js)
    HookHasEffect | HookPassive,            // Hook effect tags (ReactHookEffectTags.js)
    create,
    deps
  );
}

function mountLayoutEffect(create, deps) {
  return mountEffectImpl(
    Update | LayoutStatic,                  // Fiber flags — layout phase
    HookHasEffect | HookLayout,
    create,
    deps
  );
}
```

Commit Phase'da:

```typescript
// Layout Phase (sync, paint'dan oldin)
function commitLayoutEffects(finishedWork, root) {
  // useLayoutEffect callback'lari chaqiriladi
  invokeLayoutEffects(finishedWork);
}

// Passive Phase (async, paint'dan keyin)
function flushPassiveEffects() {
  // useEffect callback'lari chaqiriladi
  invokePassiveEffects(rootWithPendingPassiveEffects);
}
```

Passive effects scheduling:

```typescript
// Commit Phase oxirida
if (rootHasPassiveEffects) {
  Scheduler.scheduleCallback(NormalPriority, flushPassiveEffects);
  // MessageChannel orqali async rejalashtiriladi
}
```

**Browser rendering timeline (tartib — aniq ms qiymatlari hardware/payload'ga bog'liq):**

```
[1] JS executes (event handler, setState)
[2] React render phase (workInProgress tree)
[3] React commit phase — mutation phase
[4] useLayoutEffect callbacks (sync, paint'dan oldin)
[5] Browser: style recalc + layout
[6] Browser: paint — foydalanuvchi UI'ni ko'radi
[7] Browser: composite
[8] React passive effects scheduled (Scheduler `MessageChannel`)
[9] useEffect callbacks (paint'dan keyin)
```

`useLayoutEffect` paint'dan oldin ishlaydi (DOM yangilanishi va paint orasida — flicker yo'q). `useEffect` paint'dan keyin keyingi task'da ishlaydi (foydalanuvchi UI'ni ko'rib bo'lganidan keyin). Aniq ms qiymatlari berilmaydi — ular hardware, payload va R18 Scheduler concurrent rendering qaroriga bog'liq.

**SSR cheklov:**

`useLayoutEffect` server'da chaqirilmaydi (DOM yo'q). React dev'da warning beradi:

```
Warning: useLayoutEffect does nothing on the server, because its effect cannot
be encoded into the server renderer's output format.
```

Yechim:
- Komponent client-only bo'lsa — `useEffect` ishlatish
- Hybrid bo'lsa — `useIsomorphicLayoutEffect` pattern (typeof window check bilan switch)

```tsx
// useIsomorphicLayoutEffect pattern
import { useEffect, useLayoutEffect } from 'react';

const useIsomorphicLayoutEffect =
  typeof window !== 'undefined' ? useLayoutEffect : useEffect;
```

`17-uselayouteffect.md` da chuqur yoritiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`useEffect` vs `useLayoutEffect` — flicker farqi:

```tsx
// ❌ useEffect bilan flicker
function Tooltip({ children }: { children: ReactNode }) {
  const ref = useRef<HTMLDivElement>(null);
  const [width, setWidth] = useState(0);

  useEffect(() => {
    if (ref.current) {
      setWidth(ref.current.offsetWidth);
    }
  }, []);

  return (
    <div
      ref={ref}
      style={{ marginLeft: -width / 2 }}  // ← width=0 dan boshlaydi
    >
      {children}
    </div>
  );
}

// Foydalanuvchi ko'radi:
// 1. Tooltip pozitsiyasi to'g'ri emas (marginLeft = 0)
// 2. Paint
// 3. useEffect chaqiriladi, setState → re-render
// 4. Tooltip to'g'ri pozitsiyaga sakrab o'tadi
// → Visual flicker
```

```tsx
// ✅ useLayoutEffect — flicker yo'q
function Tooltip({ children }: { children: ReactNode }) {
  const ref = useRef<HTMLDivElement>(null);
  const [width, setWidth] = useState(0);

  useLayoutEffect(() => {
    if (ref.current) {
      setWidth(ref.current.offsetWidth);
    }
  }, []);

  return (
    <div
      ref={ref}
      style={{ marginLeft: -width / 2 }}
    >
      {children}
    </div>
  );
}

// Foydalanuvchi ko'radi:
// 1. Render → mutation → useLayoutEffect (sync) → setWidth → re-render
// 2. Yangi render → mutation → layout effects
// 3. Paint (to'g'ri pozitsiya bilan)
// → Flicker yo'q
```

`useEffect` — fetch (paint'ni bloklamaslik):

```tsx
function ProductPage({ productId }: { productId: number }) {
  const [product, setProduct] = useState<Product | null>(null);

  useEffect(() => {
    let cancelled = false;
    fetch(`/api/products/${productId}`)
      .then(r => r.json())
      .then(p => {
        if (!cancelled) setProduct(p);
      });
    return () => { cancelled = true; };
  }, [productId]);

  if (!product) return <Skeleton />;
  return <ProductDetails product={product} />;
}

// Foydalanuvchi avval Skeleton'ni darhol ko'radi (paint tezda)
// Keyin product yuklanganda — yangi render + paint
```

`useLayoutEffect` — focus management:

```tsx
function Modal({ isOpen, onClose }: ModalProps) {
  const dialogRef = useRef<HTMLDialogElement>(null);

  useLayoutEffect(() => {
    if (isOpen && dialogRef.current) {
      dialogRef.current.focus();  // ✅ paint'dan oldin focus
    }
  }, [isOpen]);

  if (!isOpen) return null;
  return (
    <dialog ref={dialogRef} tabIndex={-1}>
      <button onClick={onClose}>Yopish</button>
    </dialog>
  );
}

// useEffect ishlatilsa — dialog avval ko'rinadi (focus yo'q), keyin focus jamlanadi
// useLayoutEffect — paint paytida focus allaqachon dialog'da
```

`useLayoutEffect` ichida `setState` — qayta cycle:

```tsx
function AutoResize({ value }: { value: string }) {
  const ref = useRef<HTMLTextAreaElement>(null);
  const [height, setHeight] = useState('auto');

  useLayoutEffect(() => {
    if (ref.current) {
      ref.current.style.height = 'auto';  // reset
      const newHeight = `${ref.current.scrollHeight}px`;
      setHeight(newHeight);  // ← qayta render+commit cycle
    }
  }, [value]);

  return (
    <textarea
      ref={ref}
      value={value}
      style={{ height }}
      readOnly
    />
  );
}

// Cycle:
// 1. Render → commit → useLayoutEffect → setState
// 2. React darhol yangi render+commit (sync, paint'dan oldin)
// 3. Endi paint
// → Foydalanuvchi to'g'ri height bilan birinchi marta ko'radi (flicker yo'q)
```

</details>

---

## Automatic Batching

### Nazariya

**Batching** — bir nechta `setState` chaqiriqlarini **bitta render+commit cycle'ga birlashtirish**. Misol:

```tsx
function handleClick() {
  setCount(c => c + 1);   // setState 1
  setName('Yangilandi');  // setState 2
  setActive(true);        // setState 3
}
// Batched: 3 ta setState → 1 ta re-render
```

Bu — performance optimization. Har `setState` alohida render qilinsa, 3 ta render+commit cycle bo'lardi (3x DOM mutation, 3x paint). Batching bilan — 1 ta cycle.

> **🕐 Versiya evolyutsiyasi (Batching):**
> - **Pre-R18 (R17 va undan oldin):** Batching faqat **React event handler ichida** ishlardi (`onClick`, `onChange`, va h.k.). `setTimeout`, `Promise.then`, `async/await`, native DOM listener (`addEventListener`) ichida har `setState` alohida render qilardi.
> - **R18+:** **Automatic Batching** — barcha kontekstlarda batched. `setTimeout`, `Promise`, `async/await`, native event listener — hammasi avtomatik birlashtiriladi.
> - **Sabab:** Concurrent rendering uchun consistency — render davomida state inconsistent bo'lmasligi kerak. Qo'shimcha sabab — developer'lar uchun "nega bu yerda batch ishlamayapti" muammosini bartaraf etish (R17 da bu eng ko'p uchraydigan chalkashlik edi).

**R17 vs R18 misol:**

```tsx
function handleClick() {
  setTimeout(() => {
    setCount(c => c + 1);
    setActive(true);
    setName('Updated');
  }, 100);
}

// R17: 3 ta render (setTimeout ichida batching yo'q edi)
// R18+: 1 ta render (automatic batching)
```

```tsx
async function handleSubmit() {
  await fetch('/api/save');
  setLoading(false);
  setSuccess(true);
}

// R17: 2 ta render (await dan keyin batching yo'q edi)
// R18+: 1 ta render (automatic batching)
```

### Batching qoidalari

R18+ Automatic Batching ishlash qoidalari:

1. **Bir microtask ichida** chaqirilgan barcha `setState`'lar birlashtiriladi
2. **`async/await` yoki `.then` chegarasini kesib o'tganda** batching to'xtaydi va yangi cycle boshlanadi
3. **Functional update** (`setCount(c => c + 1)`) — batching bilan eng xavfsiz pattern
4. **`flushSync`** — batching'ni majburan to'xtatish (kerak bo'lganda)

```tsx
// ✅ Bitta event handler ichida — batched
function handleClick() {
  setA(1);
  setB(2);
  setC(3);
  // 1 ta render
}

// ✅ Async/await ichida ham — batched (R18+)
async function handleAsync() {
  await fetch('/api');
  setA(1);
  setB(2);
  // 1 ta render
}

// ❌ await chegarasini kesib o'tgan — har segment alohida batch
async function handleSegmented() {
  setA(1);    // batch 1 boshlanadi
  setB(2);
  await fetch('/api');  // batch 1 commit qilinadi
  setC(3);    // batch 2 boshlanadi
  setD(4);
  // 2 ta render
}
```

### Functional update va batching

`setState(newValue)` (qiymat) vs `setState(prev => newValue)` (funksiya):

```tsx
function handleClick() {
  // ❌ Qiymat — eski state'ga asoslangan
  setCount(count + 1);  // count=0 dan +1
  setCount(count + 1);  // count=0 dan +1 (eski qiymat!)
  setCount(count + 1);  // count=0 dan +1 (yana eski!)
  // Result: count=1 (3 ta +1 emas!)
}

function handleClickFn() {
  // ✅ Funksiya — har safar latest state
  setCount(c => c + 1);  // 0 → 1
  setCount(c => c + 1);  // 1 → 2
  setCount(c => c + 1);  // 2 → 3
  // Result: count=3
}
```

Sabab: birinchi pattern'da `count` — render paytidagi snapshot (closure). Batching ichida `count` o'zgarmaydi (faqat batch oxirida render bo'ladi). Functional update — React'ga "oldingi qiymatga +1 qo'sh" deb aytadi, va React har calldan oldin oxirgi qiymatni beradi.

<details>
<summary><strong>Under the Hood</strong></summary>

R18 Automatic Batching mechanism (soddalashtirilgan):

```typescript
// Manba: ReactFiberHooks.js, ReactFiberConcurrentUpdates.js, ReactFiberWorkLoop.js
// React'da GLOBAL pendingUpdates Set'i YO'Q — har Fiber'da o'z updateQueue (linked list),
// root'da esa pendingLanes bitmask. Quyida soddalashtirilgan model:

function dispatchSetState(fiber, queue, action) {
  const lane = requestUpdateLane(fiber);
  const update = { lane, action, next: null };

  // 1. Update'ni fiber'ning queue.pending linked list'iga qo'shish
  enqueueConcurrentHookUpdate(fiber, queue, update, lane);

  // 2. Root'ni topib `root.pendingLanes |= lane` qilish
  const root = getRootForUpdatedFiber(fiber);

  // 3. Scheduler'ga ish rejalashtirish.
  // Joriy synchronous segment (event handler, microtask) tugamaguncha
  // render boshlanmaydi — shu segment ichidagi setState'lar bitta render'da birlashadi.
  ensureRootIsScheduled(root);
}
```

Real implementation'da Scheduler'ning `scheduleCallback` funksiyasi `MessageChannel.postMessage` orqali yangi task rejalashtiradi. R18 automatic batching esa `queueMicrotask` orqali flush rejalashtiradi: joriy synchronous segment tugagach microtask ishga tushadi va batch commit qilinadi. `await` joriy microtask'ni tugatib flush'ni triggerlaydi → continuation yangi microtask'da boshlanib yangi batch ochadi (shu sababli `await` chegarasi batching'ni uzadi).

R17 va undan oldin esa batching faqat **`unstable_batchedUpdates`** ichida bo'lardi:

```typescript
// R17 internal
function dispatchSetState_R17(fiber, newValue) {
  enqueueUpdate(fiber, newValue);
  
  if (insideUnstableBatchedUpdates) {
    // Batched
  } else {
    // Darhol render
    scheduleUpdateOnFiber(fiber);
  }
}

// React event handler:
function handleEvent(event) {
  unstable_batchedUpdates(() => {
    listener(event);  // listener ichidagi setState'lar batched
  });
}

// setTimeout, Promise.then — bu wrap'dan tashqarida → batching yo'q
```

R18 — `unstable_batchedUpdates` har joyda avtomatik qo'llaniladi.

**Microtask vs macrotask batching boundary:**

```tsx
async function handleAsync() {
  setA(1);              // batch 1 (1-sync segment)
  setB(2);              // batch 1
  await Promise.resolve();  // microtask boundary — sync segment tugadi
  // Bu yerga yetganda batch 1 flush bo'lgan
  setC(3);              // batch 2 (2-sync segment, yangi microtask)
  setD(4);              // batch 2
}
// Result: 2 ta render
```

`await` chegarasi — JavaScript microtask cheklovi. React batch'ni shu joyda flush qiladi (joriy synchronous segment tugadi, yangi segment yangi batch boshlaydi).

**Functional update queue:**

```typescript
// React internal (soddalashtirilgan — manba: ReactFiberHooks.js, updateReducer)
// queue.pending — CIRCULAR singly-linked list. `pending` oxirgi update'ga,
// `pending.next` esa birinchi update'ga ishora qiladi.

function dispatchSetState(fiber, queue, action) {
  const update = { action, next: null };  // action: function yoki value
  // queue.pending circular list'iga qo'shish (pending = oxirgi, pending.next = birinchi)
  enqueueConcurrentHookUpdate(fiber, queue, update);
}

function updateReducer(queue, baseState, reducer) {
  const pending = queue.pending;
  if (pending === null) return baseState;

  const first = pending.next;   // circular list — birinchi update
  let newState = baseState;
  let update = first;
  do {
    const action = update.action;
    newState = typeof action === 'function'
      ? action(newState)    // functional updater
      : action;             // value
    update = update.next;
  } while (update !== first);  // circular: birinchiga qaytguncha

  return newState;
}
```

Functional updater'lar **render paytida** (`updateReducer` ichida) chaqiriladi, value updater'lar esa **dispatch paytida** belgilangan qiymatni saqlaydi. Shu sababli functional updater'lar har doim oxirgi qiymatga asoslangan.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

R17 vs R18 batching farqi (real misol):

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('Initial');

  function handleSyncClick() {
    setCount(c => c + 1);
    setName('After click');
    // Render Soni:
    // R17: 1 ✓ (React event handler — batched)
    // R18: 1 ✓
  }

  function handleAsyncClick() {
    setTimeout(() => {
      setCount(c => c + 1);
      setName('After timeout');
      // Render Soni:
      // R17: 2 ❌ (setTimeout — har setState alohida)
      // R18: 1 ✓ (automatic batching)
    }, 100);
  }

  async function handleAwaitClick() {
    await fetch('/api/data');
    setCount(c => c + 1);
    setName('After fetch');
    // R17: 2 ❌
    // R18: 1 ✓
  }

  return (
    <div>
      <p>Count: {count}, Name: {name}</p>
      <button onClick={handleSyncClick}>Sync</button>
      <button onClick={handleAsyncClick}>Async (setTimeout)</button>
      <button onClick={handleAwaitClick}>Await</button>
    </div>
  );
}
```

Stale closure muammosi (functional update bilan yechiladi):

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  function tripleIncrementBad() {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
    // count=0 dan keyin: count=1 (3 ta +1 emas!)
    // Sabab: har setCount() chaqirilganda count=0 (closure'dan)
  }

  function tripleIncrementGood() {
    setCount(c => c + 1);  // 0 → 1
    setCount(c => c + 1);  // 1 → 2
    setCount(c => c + 1);  // 2 → 3
    // Result: count=3
    // Sabab: React har functional updater'ga oxirgi qiymatni beradi
  }

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={tripleIncrementBad}>Bad +3</button>
      <button onClick={tripleIncrementGood}>Good +3</button>
    </div>
  );
}
```

Async batching ko'rgazmasi:

```tsx
function FormSubmit() {
  const [loading, setLoading] = useState(false);
  const [success, setSuccess] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit() {
    setLoading(true);
    setError(null);
    setSuccess(false);
    // 3 ta setState — 1 ta render (R18+)

    try {
      await fetch('/api/save');
      // await dan keyin yangi batch boshlanadi
      setLoading(false);
      setSuccess(true);
      // 2 ta setState — 1 ta render (R18+)
    } catch (err) {
      setLoading(false);
      setError('Xatolik yuz berdi');
      // 2 ta setState — 1 ta render (R18+)
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit" disabled={loading}>
        {loading ? 'Yuborilmoqda...' : 'Yuborish'}
      </button>
      {success && <p>Muvaffaqiyatli!</p>}
      {error && <p>{error}</p>}
    </form>
  );
}

// R18'da: jami 2 ta render (avval batch 1: 3 setState; await; keyin batch 2: 2 setState)
// R17'da: jami 3 ta render — async function'ning sync prefix (await'gacha) React event
//         handler context ichida bo'lib hisoblanadi va batched edi (1 render);
//         await'dan keyingi setState'lar esa har biri alohida (2 render).
```

</details>

---

## flushSync

### Nazariya

`flushSync` — `react-dom` paketidagi API. U `setState` chaqiriqlarini **majburan synchronous** qilib, batching'ni chetlab o'tadi va **darhol render+commit** cycle'ni bajaradi.

```tsx
import { flushSync } from 'react-dom';

flushSync(() => {
  setCount(c => c + 1);
});
// Bu satrga yetganda — DOM allaqachon yangilangan
```

**Qachon kerak:**

1. **Third-party DOM library bilan integration** — masalan, `<select>` opsiyalarini React yangilaganidan keyin native scrollIntoView'ni chaqirish kerak
2. **`onChange` callback'ida darhol DOM'ni o'qish** — masalan, focus o'zgartirish
3. **Print/export** — yangi DOM holati bilan PDF hosil qilish

**Qachon ishlatmaslik:**

- Default holatda — batching'ning afzalliklari (kam render, performance) yo'qoladi
- `flushSync` qancha ko'p ishlatilsa, app shunchalik sekin
- Ko'pincha boshqa pattern (useLayoutEffect, useImperativeHandle) yaxshiroq

### Syntax

```tsx
import { flushSync } from 'react-dom';

// Pattern 1: setState'ni o'rab olish
flushSync(() => {
  setCount(c => c + 1);
});
// Bu yerga yetganda — re-render tugagan, DOM yangilangan

// Pattern 2: Bir nechta setState'larni o'rab olish
flushSync(() => {
  setCount(c => c + 1);
  setName('Updated');
});
// 1 ta render (flushSync ichida bir necha setState ham batched)
```

`flushSync` qaytaradigan qiymat — callback'ning return qiymati:

```tsx
const result = flushSync(() => {
  setCount(c => c + 1);
  return 'done';
});
// result === 'done'
```

<details>
<summary><strong>Under the Hood</strong></summary>

`flushSync` ning ichki ishlashi:

```typescript
// react-dom internal (soddalashtirilgan)
function flushSync<T>(fn: () => T): T {
  const prevExecutionContext = executionContext;
  // Callback ichidagi state update'lar SyncLane bilan rejalashtirilsin
  // va callback tugagach darhol flush qilinsin
  executionContext |= BatchedContext;

  try {
    const result = fn();
    // Sync queue'dagi pending update'larni darhol bajarish
    flushSyncCallbacks();
    return result;
  } finally {
    executionContext = prevExecutionContext;
  }
}
```

`flushSync` callback ichidagi state update'larni `SyncLane` priority bilan rejalashtiradi — bu Concurrent Scheduler'da eng yuqori priority — va callback tugagandan keyin `flushSyncCallbacks()` orqali darhol render+commit cycle'ni majburan ishga tushiradi (boshqa pending updates ham birga flush bo'lishi mumkin). Lane'lar va priority haqida `05-scheduler-lanes.md` da batafsil.

**Performance implications:**

`flushSync` ichida render+commit **synchronous** bajariladi. Ya'ni:
- JS thread bloklanadi (foydalanuvchi input javob bermaydi)
- Browser paint qila olmaydi (UI muzlaydi)
- Concurrent features (interruptibility) ushlab turilmaydi

Shu sababli `flushSync` faqat **kichik, tezkor** state update'lar uchun ishlatilishi kerak. Katta tree update bo'lsa — UI sezilarli darajada muzlaydi.

**`flushSync` vs eski `unstable_batchedUpdates`:**

R18'gacha batching'ni majburan qo'llash uchun `unstable_batchedUpdates` ishlatilardi. R18'da batching default — shuning uchun bu API kerak emas. `flushSync` esa **teskari** vazifa — batching'ni majburan **o'chirish**.

```tsx
// R17 (batching opt-in — setTimeout/Promise ichida batching uchun majburan ishlatilardi)
unstable_batchedUpdates(() => {
  setA(1);
  setB(2);
});

// R18+ (batching default, flushSync opt-out)
// Har flushSync alohida sync render+commit cycle keltiradi:
flushSync(() => {
  setA(1);
});  // 1-render+commit
flushSync(() => {
  setB(2);
});  // 2-render+commit
// Default holatda esa setA(1); setB(2) — bitta render (automatic batching)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`flushSync` — third-party library integration:

```tsx
// Avto-scroll yangi message qo'shilganda
function ChatList({ messages }: { messages: Message[] }) {
  const [list, setList] = useState(messages);
  const containerRef = useRef<HTMLDivElement>(null);

  function addMessage(msg: Message) {
    flushSync(() => {
      setList(prev => [...prev, msg]);
    });
    // flushSync ichidagi setState commit bo'ldi → DOM yangilangan
    // Endi yangi DOM bilan scrollIntoView ishlaydi
    containerRef.current?.lastElementChild?.scrollIntoView({
      behavior: 'smooth',
    });
  }

  return (
    <div ref={containerRef}>
      {list.map(msg => <div key={msg.id}>{msg.text}</div>)}
    </div>
  );
}
```

```tsx
// Agar flushSync ishlatmasangiz:
function addMessage(msg: Message) {
  setList(prev => [...prev, msg]);
  // Bu yerda DOM hali yangilanmagan (batching kutilmoqda)
  containerRef.current?.lastElementChild?.scrollIntoView();
  // ← eski oxirgi element'ga scroll bo'ladi (yangi yo'q)
}
```

`flushSync` — print uchun:

```tsx
function PrintableInvoice() {
  const [showAllItems, setShowAllItems] = useState(false);

  function handlePrint() {
    flushSync(() => {
      setShowAllItems(true);
    });
    // DOM endi to'liq invoice bilan yangilangan
    window.print();
    // Print tugagandan keyin holatni qaytarish
    setShowAllItems(false);
  }

  return (
    <div>
      <button onClick={handlePrint}>Chop etish</button>
      <Invoice expanded={showAllItems} />
    </div>
  );
}
```

`flushSync` — focus management (useLayoutEffect mos kelmagan holatda):

```tsx
function NewItemForm() {
  const [items, setItems] = useState<Item[]>([]);
  const inputRef = useRef<HTMLInputElement>(null);

  function addItem() {
    const newItem: Item = { id: Date.now(), text: '' };
    
    flushSync(() => {
      setItems(prev => [...prev, newItem]);
    });
    // Yangi <input> DOM'da paydo bo'ldi
    
    // Yangi item'ning input'iga focus
    const inputs = document.querySelectorAll('.item-input');
    const lastInput = inputs[inputs.length - 1] as HTMLInputElement;
    lastInput?.focus();
  }

  return (
    <div>
      {items.map(item => (
        <input
          key={item.id}
          className="item-input"
          defaultValue={item.text}
        />
      ))}
      <button onClick={addItem}>Qo'shish</button>
    </div>
  );
}
```

Anti-pattern — `flushSync`'ni har joyda ishlatish:

```tsx
// ❌ Keraksiz flushSync — batching afzalligini yo'qotadi
function handleClick() {
  flushSync(() => setA(1));  // 1 render
  flushSync(() => setB(2));  // 2 render
  flushSync(() => setC(3));  // 3 render
}

// ✅ Default batching ishlatish
function handleClick() {
  setA(1);
  setB(2);
  setC(3);
  // 1 render (R18+ automatic batching)
}
```

</details>

---

## Concurrent Features — Qisqa Overview

### Nazariya

**Concurrent Rendering** — React 18'da stable bo'lgan yangi rendering modeli. Asosiy g'oya — render'ni **uziluvchi** (interruptible) qilish: high-priority update kelganda joriy ishni to'xtatib, urgent ishga o'tib, keyin qaytib davom ettirish.

> **Terminologiya:** "**Concurrent Mode**" — R18 oldidan ishlatilgan experimental nom (`<ConcurrentMode>` / `<ConcurrentRoot>` wrapper komponentlari bilan opt-in qilinardi). R18+'da bu wrapper'lar yo'q — `createRoot` ishlatish bilan **Concurrent Features** avtomatik yoqiladi va kerakli hooks (`useTransition`, `useDeferredValue`) bilan opt-in bo'ladi. Hujjatlarda hozir asosan "**concurrent rendering**" yoki "**concurrent features**" termini ishlatiladi; eski "concurrent mode" iborasi shartli ishlatilishi mumkin, lekin tushuncha sifatida modal flag emas.

**Asosiy hooks va API'lar:**

| API | Vazifa |
|-----|--------|
| `useTransition` | Non-urgent update'larni belgilash (UI input javob bersin, lekin update kechiktirish mumkin) |
| `useDeferredValue` | Qiymatni "kechiktirish" — eski qiymat'da render |
| `<Suspense>` | Loading state'larni declarative tarzda boshqarish |
| `startTransition` | useTransition'ning hooksiz versiyasi |

**Concurrent rendering invariant'lari:**

- Render Phase **pure** bo'lishi shart (uziluvchi va qayta boshlanuvchi)
- Effect'lar **idempotent** bo'lishi kerak (Strict Mode tekshiradi)
- External store subscription'lar **`useSyncExternalStore`** orqali (tearing prevention)

> **Eslatma:** Bu — qisqa overview. Concurrent rendering mental model va invariants — `30-concurrent-react.md` da chuqur. Hooks (`useTransition`, `useDeferredValue`, `useSyncExternalStore`, `useId`) — `22-concurrent-hooks.md` da. Scheduler va lanes mexanikasi — `05-scheduler-lanes.md` da.

```tsx
// Qisqa misol — useTransition
import { useState, useTransition } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  function handleChange(value: string) {
    setQuery(value);  // urgent — input darhol javob beradi
    
    startTransition(() => {
      // non-urgent — natijalar update kechiktirish mumkin
      setResults(searchHeavy(value));
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
```

---

## Hydration — Qisqa Overview

### Nazariya

**Hydration** — server'da render qilingan HTML'ga client-side React'ni "biriktirish" jarayoni. SSR (Server-Side Rendering) yoki SSG (Static Site Generation) ishlatadigan loyihalar uchun zarur.

```tsx
// Server: HTML hosil qilish
import { renderToString } from 'react-dom/server';
const html = renderToString(<App />);
// → "<div><h1>Salom</h1></div>"

// Client: hydrate
import { hydrateRoot } from 'react-dom/client';
const container = document.getElementById('root');
if (!container) throw new Error('Root element topilmadi');
hydrateRoot(container, <App />);
```

`hydrateRoot` `createRoot` o'rniga ishlatiladi — server HTML'ni "tan oladi" va event listener'larni biriktiradi (yangi DOM yaratmasdan).

**Hydration mismatch** — server va client render natijasi farq qilsa (masalan, `Date.now()` ishlatish, conditional rendering env'ga asosli, browser-only API'lar). Bu — bug'ga olib keladi.

> **Eslatma:** Hydration chuqur — `06-hydration.md` da. Mismatch sabablari, `suppressHydrationWarning`, R18 selective hydration, R19 streaming hydration — barchasi shu yerda.

---

## Edge Cases va Gotchas

### `setState` darhol state'ni yangilamaydi

```tsx
function handleClick() {
  setCount(count + 1);
  console.log(count);  // ← eski qiymat (state hali yangilanmagan)
}
```

**Sabab:** `setState` faqat re-render'ni rejalashtiradi. Joriy render davomida `count` o'zgarmaydi (closure'da snapshot). Yangi qiymat keyingi render'da bo'ladi.

**Yechim:** Yangi qiymatni darhol o'qish kerak bo'lsa — local variable'ga saqlash:

```tsx
function handleClick() {
  const newCount = count + 1;
  setCount(newCount);
  console.log(newCount);  // ✅ to'g'ri qiymat
}
```

---

### `useEffect` ichida setState — render+commit cycle

```tsx
function Bad() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    setCount(count + 1);  // ❌ infinite loop — deps array yo'q, har render'da chaqiriladi
  });

  return <div>{count}</div>;
}
```

**Sabab:** `useEffect` `setCount` chaqiradi → re-render → `useEffect` qayta chaqiriladi → `setCount` → ... cheksiz takrorlanadi.

**Yechim:** Dependency array bilan cheklash, yoki effect'dan voz kechish (derived state pure hisob bilan):

```tsx
useEffect(() => {
  setCount(c => c + 1);
}, []);  // ✅ faqat mount'da bir marta
```

---

### Render davomida `Math.random()` — concurrent'da bug

```tsx
function Lottery() {
  const number = Math.random();  // ❌ render'da non-deterministic
  return <div>Raqam: {number}</div>;
}
```

**Sabab:** Render Phase pure va idempotent bo'lishi shart. `Math.random()` har chaqiriqda yangi qiymat beradi — bu render purity invariant'ini buzadi. Concurrent mode'da render uzilib qayta boshlansa yoki Strict Mode 2x invocation paytida — har safar yangi qiymat hosil bo'ladi va komponent natijasi non-deterministic bo'lib qoladi.

**Yechim:** Random qiymatni `useState` lazy init'da hosil qilish:

```tsx
function Lottery() {
  // Production'da bir marta chaqiriladi; Strict Mode dev'da 2 marta (purity check),
  // lekin faqat ikkinchi natija foydalaniladi.
  const [number] = useState(() => Math.random());  // ✅ render uchun stable
  return <div>Raqam: {number}</div>;
}
```

---

### `flushSync` ichida boshqa komponent setState — synchronous

```tsx
function handleClick() {
  flushSync(() => {
    setA(1);  // bu komponent
    notifyListener();  // boshqa komponent setState chaqiradi
  });
  // Hammasi synchronous commit qilingan
}
```

**Sabab:** `flushSync` o'z callback ichidagi **barcha** setState chaqiruvlarini bitta synchronous cycle'ga qo'yadi (boshqa komponent'larniki ham).

**Foydalanish:** Cross-component sync update kerak bo'lganda. **Xavf:** kutilmagan tomonlama ta'sirlar (boshqa komponent'lar ham synchronous yangilanadi).

---

### Concurrent rendering'da `console.log` ikki marta

```tsx
function Counter() {
  console.log('Render');  // dev'da 2 marta chiqadi
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**Sabab:** Strict Mode + R18+ dev mode — komponent funksiyasini ikki marta chaqiradi (purity check).

**To'g'ri qabul qilish:** Bu bug emas, tekshirish. Production'da bir marta chiqadi. Side effect (mutation) yo'q bo'lsa — xavf yo'q.

---

## Common Mistakes

### ❌ Xato 1: `ReactDOM.render` ishlatish (R18+ deprecated)

```tsx
// ❌ Eski API — concurrent features yo'q
import ReactDOM from 'react-dom';
ReactDOM.render(<App />, document.getElementById('root'));

// Console: "ReactDOM.render is no longer supported in React 18..."
```

```tsx
// ✅ Yangi API
import { createRoot } from 'react-dom/client';

const container = document.getElementById('root');
if (!container) throw new Error('Root element topilmadi');
createRoot(container).render(<App />);
```

---

### ❌ Xato 2: Render davomida `setState`

```tsx
interface Product { id: string; name: string; price: number; }

// ❌ Props'ni state'ga ko'chirish — derived state anti-pattern
function ProductListBuggy({ products }: { products: Product[] }) {
  const [items, setItems] = useState<Product[]>([]);

  if (items.length === 0 && products.length > 0) {
    setItems(products);  // guard tufayli infinite loop YO'Q, lekin keyin
  }                      // products o'zgarsa items eskirib qoladi (sync yo'q)

  return <ProductGrid items={items} />;
}
```

```tsx
// ✅ useEffect bilan — props o'zgarsa state sync qilinadi
function ProductListWithEffect({ products }: { products: Product[] }) {
  const [items, setItems] = useState<Product[]>([]);

  useEffect(() => {
    setItems(products);
  }, [products]);

  return <ProductGrid items={items} />;
}

// ✅ Yoki — derived state (state'siz, pure hisob — afzal)
function ProductList({ products }: { products: Product[] }) {
  return <ProductGrid items={products} />;
}
```

---

### ❌ Xato 3: Functional update'siz batching'ga ishonish

```tsx
// ❌ Eski qiymatga asoslangan
function handleTriple() {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
  // count: 0 → 1 (3 ta +1 emas)
}
```

```tsx
// ✅ Functional update — har safar latest
function handleTriple() {
  setCount(c => c + 1);
  setCount(c => c + 1);
  setCount(c => c + 1);
  // count: 0 → 1 → 2 → 3
}
```

---

### ❌ Xato 4: `useLayoutEffect`ni keraksiz ishlatish

```tsx
// ❌ Fetch uchun useLayoutEffect — paint'ni bloklaydi
function UserProfile() {
  const [user, setUser] = useState<User | null>(null);
  
  useLayoutEffect(() => {  // ❌ paint'ni bloklaydi
    fetch('/api/me').then(r => r.json()).then(setUser);
  }, []);
  
  return user ? <div>{user.name}</div> : null;
}
```

```tsx
// ✅ useEffect — paint'ni bloklamaydi
function UserProfile() {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {  // ✅ paint'dan keyin
    fetch('/api/me').then(r => r.json()).then(setUser);
  }, []);
  
  return user ? <div>{user.name}</div> : null;
}
```

---

### ❌ Xato 5: `flushSync` ni har joyda ishlatish

```tsx
// ❌ Default batching afzalligini yo'qotish
function handleClick() {
  flushSync(() => setA(1));
  flushSync(() => setB(2));
  flushSync(() => setC(3));
  // 3 ta render+commit (UI 3 marta muzlaydi)
}
```

```tsx
// ✅ Default batching — 1 render
function handleClick() {
  setA(1);
  setB(2);
  setC(3);
}
```

`flushSync` faqat **third-party DOM library bilan integration** yoki **darhol DOM o'qish kerak** bo'lganda ishlatiladi.

---

## Amaliy Mashqlar

### Mashq 1: createRoot setup (Oson)

`main.tsx` faylida `App` komponentini StrictMode bilan o'rab, `createRoot` orqali render qiling. Container topilmasa — error throw qiling.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';

const container = document.getElementById('root');
if (!container) {
  throw new Error('Root element "#root" topilmadi');
}

const root = createRoot(container);

root.render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

</details>

---

### Mashq 2: Batching tushunish (O'rta)

Quyidagi kodda nechta render bo'ladi (R18+'da)?

```tsx
async function handleClick() {
  setA(1);
  setB(2);
  await fetch('/api');
  setC(3);
  setD(4);
}
```

<details>
<summary><strong>Javob</strong></summary>

**2 ta render** (R18+).

- `setA(1)` va `setB(2)` — birinchi synchronous segment, batched → 1 ta render
- `await fetch('/api')` — microtask boundary, batch flush
- `setC(3)` va `setD(4)` — yangi synchronous segment, batched → 1 ta render

**R17'da 3 ta render** (async context'da batching'ning yo'qligi sababli):
- `setA(1)` va `setB(2)` — async function'ning sync prefix React event handler context ichida → batched (1 render)
- `await` dan keyin React event handler context tugadi
- `setC(3)` — alohida render (Promise continuation, R17'da batching yo'q)
- `setD(4)` — alohida render

</details>

---

### Mashq 3: useLayoutEffect vs useEffect (O'rta)

Quyidagi `Tooltip` komponentida `useEffect` o'rniga `useLayoutEffect` ishlatish kerakmi yoki yo'qmi? Sabab bilan tushuntiring.

```tsx
function Tooltip({ targetRef, text }: TooltipProps) {
  const tooltipRef = useRef<HTMLDivElement>(null);
  const [position, setPosition] = useState({ top: 0, left: 0 });

  useEffect(() => {
    if (!targetRef.current || !tooltipRef.current) return;
    const targetRect = targetRef.current.getBoundingClientRect();
    setPosition({ top: targetRect.bottom, left: targetRect.left });
  }, [targetRef]);

  return (
    <div ref={tooltipRef} style={{ position: 'fixed', ...position }}>
      {text}
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

**Ha, `useLayoutEffect` ishlatish kerak.**

`useEffect` paint'dan keyin ishlaydi, ya'ni:
1. Birinchi render: `position={top:0,left:0}` — tooltip noto'g'ri pozitsiyada paint qilinadi
2. `useEffect` chaqiriladi → setState → re-render
3. Yangi pozitsiya bilan paint

Foydalanuvchi **tooltip flicker'ni ko'radi** (avval `(0,0)` da, keyin to'g'ri joyga sakraydi).

`useLayoutEffect` paint'dan oldin ishlaydi:
1. Render → mutation → useLayoutEffect → setState
2. React darhol qayta render+commit (sync, paint'dan oldin)
3. Endi paint to'g'ri pozitsiya bilan

Foydalanuvchi flicker'ni ko'rmaydi.

```tsx
// ✅ To'g'ri:
useLayoutEffect(() => { ... }, [targetRef]);
```

</details>

---

### Mashq 4: flushSync use case (Qiyin)

`flushSync` ishlatilishi **shart** bo'lgan misol yozing — third-party library yoki native browser API bilan.

<details>
<summary><strong>Javob</strong></summary>

```tsx
// Yangi message qo'shilganda chat'ni pastki qismiga scroll
function ChatPanel() {
  const [messages, setMessages] = useState<Message[]>([]);
  const containerRef = useRef<HTMLDivElement>(null);

  function handleSend(text: string) {
    const newMessage: Message = { id: Date.now(), text };
    
    flushSync(() => {
      setMessages(prev => [...prev, newMessage]);
    });
    // flushSync tugagandan keyin DOM yangilangan
    // Endi yangi message bilan scrollIntoView ishlaydi
    
    const lastChild = containerRef.current?.lastElementChild;
    lastChild?.scrollIntoView({ behavior: 'smooth' });
  }

  return (
    <>
      <div ref={containerRef} className="chat-list">
        {messages.map(msg => (
          <div key={msg.id} className="chat-message">
            {msg.text}
          </div>
        ))}
      </div>
      <ChatInput onSend={handleSend} />
    </>
  );
}
```

`flushSync` shart, chunki `scrollIntoView` darhol chaqiriladi — agar batching kutilsa, eski `lastElementChild` (yangi message qo'shilishidan oldingisiga) scroll qilinardi.

</details>

---

### Mashq 5: Concurrent invariant (Qiyin)

Quyidagi komponentda render purity bilan bog'liq qaysi muammolar bor? Ularni topib tuzating.

```tsx
let lastRenderTime = 0;

function ProductPage({ id }: { id: number }) {
  lastRenderTime = Date.now();
  
  const product = JSON.parse(
    localStorage.getItem(`product-${id}`) || '{}'
  );
  
  if (!product.name) {
    fetch(`/api/products/${id}`)
      .then(r => r.json())
      .then(p => localStorage.setItem(`product-${id}`, JSON.stringify(p)));
  }
  
  return <div>{product.name || 'Yuklanmoqda...'}</div>;
}
```

<details>
<summary><strong>Javob</strong></summary>

**Aniqlangan muammolar:**

1. **`lastRenderTime = Date.now()`** — render davomida tashqi mutable state mutation
2. **`localStorage.getItem` render davomida** — synchronous I/O, lekin har render'da tashqi state'ni o'qish (concurrent'da inconsistent)
3. **`fetch` render davomida** — eng katta xato; har render'da network request
4. **`localStorage.setItem` render davomida** — side effect, render uziluvchi

**Tuzatilgan versiya:**

```tsx
function ProductPage({ id }: { id: number }) {
  const [product, setProduct] = useState<Product | null>(null);

  // Mount va id o'zgarganda — fetch
  useEffect(() => {
    let cancelled = false;
    
    // 1. localStorage cache'dan o'qishga harakat
    const cached = localStorage.getItem(`product-${id}`);
    if (cached) {
      const parsed = JSON.parse(cached) as Product;
      if (!cancelled) setProduct(parsed);
      return;
    }
    
    // 2. Cache yo'q — fetch
    fetch(`/api/products/${id}`)
      .then(r => r.json())
      .then((p: Product) => {
        if (cancelled) return;
        localStorage.setItem(`product-${id}`, JSON.stringify(p));
        setProduct(p);
      });
    
    return () => { cancelled = true; };
  }, [id]);

  if (!product) return <div>Yuklanmoqda...</div>;
  return <div>{product.name}</div>;
}
```

Tuzatishlar:
- `lastRenderTime` o'chirildi (kerak emas; agar kerak bo'lsa — `useRef`'da saqlash)
- `localStorage` o'qish/yozish — `useEffect` ichida
- `fetch` — `useEffect` ichida, cleanup bilan (race condition prevention)
- Loading state'ni `useState` orqali boshqarish

</details>

---

## Xulosa

Bu bo'limda React'ning rendering pipeline'ining ichki mexanikasi yoritildi:

- **`createRoot`** — R18+ entry point, concurrent features uchun zamin
- **Strict Mode** — render purity (R16.3+) va effect idempotency (R18+) tekshiruvi
- **Render Phase** — pure, uziluvchi, workInProgress tree quradi
- **Commit Phase 3 sub-phase** — Before Mutation → Mutation → Layout, atomic va synchronous
- **Effect timing** — `useInsertionEffect` (CSS), `useLayoutEffect` (paint'dan oldin), `useEffect` (paint'dan keyin)
- **Automatic Batching (R18+)** — barcha kontekstlarda setState'lar birlashtiriladi
- **`flushSync`** — batching'dan opt-out, third-party integration uchun
- **Concurrent Features va Hydration** — qisqa intro, chuqur 22, 30, 06 da

Bu mechanism'ni tushunish keyingi bo'limlardagi har bir hook va pattern'ning xatti-harakatini oydinlashtiradi. Misol uchun, `useEffect`ning "lifecycle hook EMAS, synchronization mechanism" g'oyasi (`16-useeffect.md`) — concurrent rendering + Strict Mode 2x effect cycle'idan kelib chiqadi.

Keyingi bo'lim — [`03-fiber-architecture.md`](03-fiber-architecture.md): Reconciler'ning ichki tuzilmasi (Fiber architecture) chuqur yoritiladi. Bu render+commit pipeline'ning **qanday** ishlashining texnik tafsiloti.

---

**Keyingi bo'lim:** [03-fiber-architecture.md](03-fiber-architecture.md) — Fiber Architecture: work unit, current vs workInProgress, double buffering, alternate pointer, Fiber tag types (FunctionComponent, HostComponent, va h.k.), tree traversal pointers (child, sibling, return), effect list, sabab — nima uchun React Stack Reconciler'dan Fiber'ga o'tdi.
