# Bo'lim 6: Hydration

> Hydration — server'da render qilingan HTML'ga client-side React'ni "biriktirish" jarayoni. React mavjud DOM node'larni reuse qilib, ular ustiga Fiber tree quradi va event listener'larni ulaydi. Bu bo'lim SSR + Hydration mexanikasini, mismatch sabablarini, R18 selective/streaming hydration va R19 yaxshilanishlarni yoritadi.

---

## Mundarija

- [SSR va Hydration Nima](#ssr-va-hydration-nima)
- [hydrateRoot vs createRoot](#hydrateroot-vs-createroot)
- [Hydration Pipeline](#hydration-pipeline)
- [Hydration Mismatch Sabablari](#hydration-mismatch-sabablari)
- [suppressHydrationWarning](#suppresshydrationwarning)
- [Selective Hydration (R18)](#selective-hydration-r18)
- [Streaming Hydration (R18)](#streaming-hydration-r18)
- [R19 Hydration Improvements](#r19-hydration-improvements)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## SSR va Hydration Nima

### Nazariya

**Server-Side Rendering (SSR)** — React komponent tree'ni **server'da** (Node.js, Deno, Bun, va h.k.) render qilib, oldindan tayyorlangan HTML string'ni client'ga yuborish. Brauzer HTML'ni darhol ko'rsatadi (JavaScript yuklanguncha kutilmaydi).

**Hydration** — server HTML keldi-yu, brauzer uni ko'rsatdi. Endi React client-side ishga tushishi kerak: state, event listener, hooks. Lekin **DOM allaqachon mavjud** — yangi yaratish kerak emas. Hydration — **mavjud DOM'ga React'ni biriktirish**.

**Pipeline:**

```
1. Foydalanuvchi sahifani so'raydi
   ↓
2. Server React komponent tree'ni HTML'ga aylantiradi
   - renderToString / renderToReadableStream / renderToPipeableStream
   - Hech qanday event listener YO'Q
   - State faqat initial render bilan
   ↓
3. Server HTML'ni clientga yuboradi
   ↓
4. Brauzer HTML'ni parse qiladi va ko'rsatadi (FIRST PAINT)
   - Foydalanuvchi sahifani ko'radi
   - Lekin click ishlamaydi (JS yo'q)
   ↓
5. Brauzer JS bundle'ni yuklaydi va ishlaydi
   ↓
6. React client'da hydrate qiladi
   - hydrateRoot(container, <App />)
   - Fiber tree quriladi MAVJUD DOM ustiga
   - Event listener'lar ulanadi
   - Hooks initialize qilinadi (useState lazy initial values)
   ↓
7. Sahifa to'liq INTERAKTIV
```

**SSR foydalari:**

1. **First Contentful Paint (FCP) tezroq** — HTML darhol ko'rinadi, JS yuklanish kutilmaydi
2. **SEO** — search bot'lar HTML'da to'liq kontent ko'radi (JS execute qilmasdan ham)
3. **Slow connection** — eski qurilmalarda JS sekin yuklanadi, lekin HTML allaqachon ko'rinadi
4. **JS off** — JavaScript o'chirilgan bo'lsa, asosiy kontent hali ko'rinadi

**SSR kamchiliklari:**

1. **Time To Interactive (TTI) kech** — HTML ko'rinsa-da, hydrate tugamaguncha click ishlamaydi (R18'gacha)
2. **Server resource'i** — har request uchun React render (CPU)
3. **Hydration mismatch** — server va client render farqli bo'lsa — bug
4. **JS bundle hali kerak** — SSR JS'ni almashtirmaydi (faqat birinchi paint'ni tezlashtiradi)

### Hydration vs Initial Render

| Aspect | Initial Render (`createRoot`) | Hydration (`hydrateRoot`) |
|--------|-------------------------------|---------------------------|
| **DOM mavjudmi** | Yo'q — React yaratadi | Ha — server HTML mavjud |
| **DOM operations** | Hammasi (createElement, appendChild) | Faqat event listener attach |
| **Performance** | Sekinroq (DOM yaratish) | Tezroq (faqat attach) |
| **State** | Component'lar mount qilinadi | State server initial bilan boshlanadi |
| **Effect'lar** | Mount effect'lari ishga tushadi | Mount effect'lari ishga tushadi (client'da) |
| **Mismatch xavfi** | Yo'q | Ha — server vs client render farqi |

<details>
<summary><strong>Under the Hood</strong></summary>

**SSR API'lar:**

React'da SSR uchun bir nechta API mavjud:

| API | Output | Streaming | Suspense for data |
|-----|--------|-----------|-------------------|
| `renderToString` | String | YO'Q | Cheklangan |
| `renderToStaticMarkup` | String (hydration metadata yo'q) | YO'Q | YO'Q |
| `renderToReadableStream` (R18) | Web Streams ReadableStream | HA | HA |
| `renderToPipeableStream` (R18) | Node Streams | HA | HA |

> **Eslatma:** `data-reactroot` attribute R17 va undan oldin emitted edi. R18'dan boshlab olib tashlangan — hydration marker'lar (comment node'lar) ishlatiladi. `renderToStaticMarkup` esa Suspense marker va boshqa hydration metadata'larni umuman emit qilmaydi (faqat statik HTML uchun mo'ljallangan).

**`renderToString` — sinxron:**

```typescript
import { renderToString } from 'react-dom/server';

const html = renderToString(<App />);
// "<div><h1>Salom</h1></div>"
// R18+: data-reactroot attribute olib tashlangan — hydration marker'lar (comment node'lar) bilan almashtirilgan

// Server response
res.send(`
  <!DOCTYPE html>
  <html>
    <body>
      <div id="root">${html}</div>
      <script src="/bundle.js"></script>
    </body>
  </html>
`);
```

`renderToString` — **sinxron, blocking**. Butun tree render qilinmaguncha qaytmaydi. Suspense bilan ishlash cheklangan (R18'gacha umuman ishlamadi).

**`renderToReadableStream` — Web Streams (modern):**

```typescript
import { renderToReadableStream } from 'react-dom/server';

const stream = await renderToReadableStream(<App />, {
  bootstrapScripts: ['/bundle.js'],
  onError(error) {
    console.error('SSR error:', error);
  },
});

// stream — ReadableStream (Web Streams API)
// HTML chunk'lar yuboriladi:
// 1. Initial shell (sync rendered) — birinchi macrotask'da
// 2. Resolved Suspense boundary'lari — async data tayyor bo'lganda
//
// Promise resolve faqat shell tayyor bo'lganda (onShellReady ekvivalenti).
// Streaming hali davom etadi — boundary'lar resolved bo'lganda chunk qo'shiladi.
```

**`renderToPipeableStream` — Node Streams:**

```typescript
import { renderToPipeableStream } from 'react-dom/server';

app.get('/', (req, res) => {
  const { pipe, abort } = renderToPipeableStream(<App />, {
    bootstrapScripts: ['/bundle.js'],
    onShellReady() {
      // Initial shell tayyor — pipe boshlanadi
      res.setHeader('Content-Type', 'text/html');
      pipe(res);
    },
    onShellError(error) {
      res.statusCode = 500;
      res.send('<h1>Error</h1>');
    },
    onError(error) {
      console.error(error);
    },
  });

  // Timeout
  setTimeout(abort, 10000);
});
```

`renderToPipeableStream` — Node.js streams ishlatadi. Streaming SSR'ning real-world implementatsiyasi (Next.js, Remix shu API'ni ishlatadi).

**Hydration internal:**

```typescript
// React internal (soddalashtirilgan)
function hydrateRoot(container, initialChildren, options) {
  const root = createHydrationContainer(
    initialChildren,
    null,
    container,
    ConcurrentRoot,
    null,
    false,
    '',
    options?.onRecoverableError ?? defaultOnRecoverableError,
    options?.transitionCallbacks ?? null,
  );
  
  // Mavjud DOM'ni tag qilish
  markContainerAsRoot(root.current, container);
  
  // Event delegation o'rnatish
  listenToAllSupportedEvents(container);
  
  return new ReactDOMRoot(root);
}
```

`hydrateRoot` `createRoot`'dan farqli — DOM yaratmaydi, mavjud node'larni "claim" qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Standart SSR setup (Express + React):

```typescript
// server.tsx
import express from 'express';
import { renderToString } from 'react-dom/server';
import App from './App';

const app = express();

app.get('/', (req, res) => {
  const appHtml = renderToString(<App />);
  
  res.send(`
    <!DOCTYPE html>
    <html lang="uz">
      <head>
        <title>SSR App</title>
      </head>
      <body>
        <div id="root">${appHtml}</div>
        <script src="/bundle.js"></script>
      </body>
    </html>
  `);
});

app.listen(3000);
```

```tsx
// client.tsx (browser entry)
import { hydrateRoot } from 'react-dom/client';
import App from './App';

const container = document.getElementById('root');
if (!container) throw new Error('Root container topilmadi');

hydrateRoot(container, <App />);
```

```tsx
// App.tsx (shared, server va client uchun)
function App() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <h1>SSR App</h1>
      <p>Bu kontent server'da render qilingan</p>
      <button onClick={() => setCount(c => c + 1)}>
        Click count: {count}
      </button>
    </div>
  );
}
```

**Pipeline:**

1. Server `renderToString(<App />)` chaqiradi → `<div><h1>...</h1>...<button>Click count: 0</button></div>`
2. HTML clientga yuboriladi
3. Brauzer parse qiladi va ko'rsatadi (paint)
4. `bundle.js` yuklanadi
5. `hydrateRoot` chaqiriladi
6. React tree mavjud DOM ustiga quriladi
7. Event listener'lar attach qilinadi
8. Click endi ishlaydi

Streaming SSR (R18+):

```typescript
// server.tsx
import { renderToPipeableStream } from 'react-dom/server';

app.get('/', (req, res) => {
  let didError = false;
  
  const { pipe, abort } = renderToPipeableStream(
    <App />,
    {
      bootstrapScripts: ['/bundle.js'],
      onShellReady() {
        res.statusCode = didError ? 500 : 200;
        res.setHeader('Content-Type', 'text/html; charset=utf-8');
        pipe(res);
      },
      onError(error) {
        didError = true;
        console.error(error);
      },
    },
  );

  setTimeout(abort, 10000);
});
```

```tsx
// App.tsx
import { Suspense } from 'react';

function App() {
  return (
    <html>
      <head>
        <title>Streaming SSR</title>
      </head>
      <body>
        <Header />
        <Suspense fallback={<div>Loading articles...</div>}>
          <Articles />
        </Suspense>
        <Suspense fallback={<div>Loading comments...</div>}>
          <Comments />
        </Suspense>
        <Footer />
      </body>
    </html>
  );
}
```

**Streaming oqimi:**

1. Server initial shell yuboradi (Header, Suspense fallbacks, Footer) — TTFB kichik
2. Brauzer initial HTML'ni ko'rsatadi (foydalanuvchi sarlavhalarni darhol ko'radi)
3. Server Articles fetched bo'lganda — chunk yuboradi (`<script>` orqali fallback'ni almashtirib)
4. Brauzer mavjud Suspense fallback'ni yangi kontent bilan almashtiradi
5. Comments tayyor bo'lganda — yana chunk
6. Foydalanuvchi qism-qism yuklanishni ko'radi (ekran "to'lib boradi")

</details>

---

## hydrateRoot vs createRoot

### Nazariya

R18'dan boshlab React DOM'da ikkita root API:

| API | Vazifa | DOM holati |
|-----|--------|------------|
| **`createRoot(container)`** | Client-only rendering | DOM bo'sh — React yaratadi |
| **`hydrateRoot(container, element)`** | SSR'dan keyin client'side hydration | DOM mavjud — React reuse qiladi |

### `hydrateRoot` xususiyatlari

```tsx
import { hydrateRoot } from 'react-dom/client';

const root = hydrateRoot(
  container,
  <App />,
  {
    onRecoverableError(error, errorInfo) {
      console.error('Recoverable error:', error);
    },
    onCaughtError(error, errorInfo) {     // R19+
      // Error Boundary tomonidan ushlangan
    },
    onUncaughtError(error, errorInfo) {   // R19+
      // Hech qanday boundary ushlamagan
    },
    identifierPrefix: 'app-',              // useId prefix (multi-root sahifalarda kerak)
    formState: serializedFormState,        // R19: SSR form state restoration
  }
);

root.render(<UpdatedApp />);
root.unmount();
```

Barcha options optional. To'liq ro'yxat (R19 stable): `onRecoverableError`, `onCaughtError`, `onUncaughtError`, `identifierPrefix`, `formState`.

**`createRoot` bilan farqlar:**

1. **Sintaksis** — `hydrateRoot(container, element, options)` (element parametr birinchi `render`'da emas)
2. **`onRecoverableError`** — hydration mismatch'ni ushlash uchun
3. **DOM expectation** — container ichida mos HTML kerak

```tsx
const root = createRoot(container);
root.render(<App />);

const root = hydrateRoot(container, <App />);
```

### Hydration mode farqi

`hydrateRoot` **hydration mode**'ni yoqadi. Bu mode'da React:
- DOM yaratmaydi — mavjudni reuse qiladi
- Mismatch'ni warning bilan log qiladi
- Mismatch holatda tegishli subtree'ni qayta render qiladi (`onRecoverableError`)

> **🕐 Versiya evolyutsiyasi (Root API):**
> - **Pre-R18 (`ReactDOM.hydrate`):** Eski API, sync hydration. Concurrent features yo'q.
> - **R18+ (`hydrateRoot`):** Yangi API. Concurrent rendering, selective hydration, streaming bilan integratsiya.
> - **Sabab:** Concurrent features faqat yangi root API'lar bilan ishlaydi (cross-ref `02-rendering.md`).

<details>
<summary><strong>Under the Hood</strong></summary>

**`hydrateRoot` ichki ishi:**

```typescript
export function hydrateRoot(
  container: Element | Document,
  initialChildren: ReactNode,
  options?: HydrateRootOptions,
): RootType {
  if (!isValidContainer(container)) {
    throw new Error('Target container is not a valid DOM element.');
  }
  
  warnIfReactDOMContainerInDEV(container);
  
  const root = createHydrationContainer(
    initialChildren,
    null,
    container,
    ConcurrentRoot,
    null,
    false,
    '',
    options?.onRecoverableError ?? defaultOnRecoverableError,
    options?.transitionCallbacks ?? null,
    options?.formState ?? null,
  );
  
  markContainerAsRoot(root.current, container);
  
  const rootContainerElement: Element | Document =
    container.nodeType === COMMENT_NODE ? container.parentNode : container;
  listenToAllSupportedEvents(rootContainerElement);
  
  return new ReactDOMHydrationRoot(root);
}
```

**Event listener attachment — event delegation:**

```typescript
function listenToAllSupportedEvents(rootContainerElement) {
  const listenerSet = rootContainerElement[listeningMarker] ?? new Set();
  rootContainerElement[listeningMarker] = listenerSet;
  
  for (const event of allNativeEvents) {
    if (!nonDelegatedEvents.has(event)) {
      listenToNativeEvent(event, false, rootContainerElement);
      listenToNativeEvent(event, true, rootContainerElement);
    }
  }
}
```

R17+'dan boshlab event listener'lar **root container**'ga (`#root` element) attach qilinadi (avval `document` edi). Bu — multiple React app'lar bir sahifada ishlash imkonini beradi (microfrontend).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Static site'da `createRoot` (no SSR):

```tsx
import { createRoot } from 'react-dom/client';
import App from './App';

const container = document.getElementById('root')!;
createRoot(container).render(<App />);
```

SSR + `hydrateRoot`:

```tsx
import { hydrateRoot } from 'react-dom/client';
import App from './App';

const container = document.getElementById('root')!;
hydrateRoot(container, <App />);
```

`onRecoverableError` ishlatish:

```tsx
import { hydrateRoot } from 'react-dom/client';

hydrateRoot(container, <App />, {
  onRecoverableError(error, errorInfo) {
    if (import.meta.env.PROD) {
      reportToSentry(error, {
        componentStack: errorInfo.componentStack,
        type: 'hydration_recoverable',
      });
    } else {
      console.error('[Hydration recoverable error]', error);
    }
  },
});
```

R19'da error boundaries kengaytirildi:

```tsx
hydrateRoot(container, <App />, {
  onCaughtError(error, errorInfo) {
    console.log('Caught:', error.message);
  },
  
  onUncaughtError(error, errorInfo) {
    console.error('Uncaught:', error);
  },
  
  onRecoverableError(error, errorInfo) {
    console.warn('Recoverable:', error.message);
  },
});
```

Multi-root setup (microfrontend):

```tsx
const widget1Container = document.getElementById('widget-1')!;
const widget2Container = document.getElementById('widget-2')!;

hydrateRoot(widget1Container, <WidgetOne />);
hydrateRoot(widget2Container, <WidgetTwo />);
```

</details>

---

## Hydration Pipeline

### Nazariya

Hydration jarayoni quyidagi qadamlarni o'z ichiga oladi:

```
1. hydrateRoot(container, <App />) chaqiriladi
   ↓
2. FiberRoot yaratiladi (ConcurrentRoot mode)
   - createHydrationContainer ishlatiladi
   - hostRoot Fiber'iga isDehydrated marker o'rnatiladi
   ↓
3. Event listener'lar root container'ga attach qilinadi
   - listenToAllSupportedEvents
   ↓
4. Initial render rejalashtiriladi
   - DefaultHydrationLane (user interaction kelganda SelectiveHydrationLane priority elevation)
   ↓
5. Render Phase boshlanadi
   - Har Fiber uchun beginWork chaqiriladi
   - isHydrating global flag tekshiriladi
   ↓
6. Komponent funksiyasi chaqiriladi
   - Hooks ishga tushadi (useState lazy initial)
   - JSX qaytariladi
   ↓
7. Mavjud DOM'ga "matching" (tryToClaimNextHydratableInstance)
   - Har element uchun mavjud DOM node topiladi
   - Type, key, props bilan solishtiriladi
   - Mismatch bo'lsa — onRecoverableError chaqiriladi va recovery
   ↓
8. fiber.stateNode = existingDOMNode (claim)
   - Real DOM yaratilmaydi — mavjud reuse qilinadi
   ↓
9. Commit Phase
   - BeforeMutation: snapshot (class component'lar)
   - Mutation: faqat event listener bog'lash + ref detach (DOM struktura mutation YO'Q)
   - Layout: refs attach, useLayoutEffect, componentDidMount
   - Paint
   - Passive: useEffect
```

**Asosiy farq Initial Render bilan:**

| Qadam | Initial Render | Hydration |
|-------|----------------|-----------|
| Fiber yaratish | Ha | Ha |
| Komponent funksiyasi chaqirilishi | Ha | Ha |
| Hooks ishga tushishi | Ha | Ha |
| DOM yaratish | Ha (createElement) | YO'Q (existing reuse) |
| DOM mutation (Commit) | Ha (appendChild) | YO'Q (faqat listener) |
| Event listener attach | Ha | Ha |
| Mount effect'lari | Ha | Ha |

### Hydration mismatch detection

Render Phase davomida har element uchun:

1. **Type tekshirish** — Fiber tag va DOM node nomini solishtirish
2. **Children tekshirish** — text content, children count
3. **Attributes tekshirish** — class, id, style, va h.k.

Mismatch bo'lganda React:
- Console'ga warning
- `onRecoverableError` chaqiriladi
- Tegishli subtree client'da qayta render qilinadi (DOM almashtiriladi)

<details>
<summary><strong>Under the Hood</strong></summary>

**Hydration mode'da `beginWork`:**

```typescript
function updateHostComponent(current, workInProgress, renderLanes) {
  if (current === null) {
    if (isHydrating) {
      const isMatch = tryToClaimNextHydratableInstance(workInProgress);
      
      if (isMatch) {
        workInProgress.stateNode = nextHydratableInstance;
      } else {
        workInProgress.flags |= Placement;
      }
    } else {
      workInProgress.flags |= Placement;
    }
  }
}
```

**`tryToClaimNextHydratableInstance`:**

```typescript
function tryToClaimNextHydratableInstance(fiber: Fiber): boolean {
  if (!isHydrating) return false;
  
  const nextInstance = nextHydratableInstance;
  if (!nextInstance) {
    insertNonHydratedInstance(hydrationParentFiber, fiber);
    isHydrating = false;
    hydrationParentFiber = fiber;
    return false;
  }
  
  const fiberType = fiber.type;
  const nextInstanceType = nextInstance.nodeName.toLowerCase();
  
  if (
    fiberType !== nextInstanceType ||
    isHydrationMismatch(fiber, nextInstance)
  ) {
    warnNonhydratedInstance(hydrationParentFiber, fiber);
    isHydrating = false;
    return false;
  }
  
  fiber.stateNode = nextInstance;
  hydrationParentFiber = fiber;
  nextHydratableInstance = getFirstHydratableChildInstance(nextInstance);
  
  return true;
}
```

**Effect list va hydration:**

Hydration mode'da Mutation Phase boshqacha — DOM mutation kerak emas. Lekin **Layout Phase** to'liq ishga tushadi:
- Refs attach (mavjud DOM node'larga)
- useLayoutEffect callback'lar
- componentDidMount

Passive Phase ham normal — useEffect hammasi ishga tushadi (mount holatda).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Hydration log:

```tsx
function App() {
  const [count, setCount] = useState(0);
  
  console.log('App render');
  
  useLayoutEffect(() => {
    console.log('useLayoutEffect — DOM ready');
  }, []);
  
  useEffect(() => {
    console.log('useEffect — paint dan keyin');
  }, []);
  
  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}

// Server log (renderToString):
// "App render"

// Client log (hydrateRoot):
// "App render"
// "useLayoutEffect — DOM ready"
// "useEffect — paint dan keyin"

// User click qiladi:
// "App render"
```

Heavy server-side computation client'da qaytarmaslik:

```tsx
function ProductList() {
  const products = useMemo(
    () => expensiveComputation(rawData),
    [rawData]
  );
  
  return (
    <ul>
      {products.map(p => <li key={p.id}>{p.name}</li>)}
    </ul>
  );
}
```

</details>

---

## Hydration Mismatch Sabablari

### Nazariya

**Hydration mismatch** — server va client render natijasi farqli bo'lganda yuz beradi. Eng ko'p uchraydigan sabablar:

### 1. Time-based qiymatlar

```tsx
// ❌ Mismatch
function Timestamp() {
  return <p>Now: {new Date().toLocaleString()}</p>;
}

// Server'da: "Now: 2026-05-02 12:00:00"
// Client'da: "Now: 2026-05-02 12:00:03"
```

### 2. Random qiymatlar

```tsx
// ❌ Mismatch
function RandomColor() {
  return <div style={{ color: `#${Math.random().toString(16).slice(2, 8)}` }}>...</div>;
}
```

```tsx
// useId hook bu muammoni hal qiladi (R18+)
function ID() {
  const id = useId();
  return <div id={id}>...</div>;
}
```

### 3. Browser-only API'lar

```tsx
// ❌ window — server'da yo'q
function ScreenWidth() {
  if (typeof window === 'undefined') {
    return <p>Width: ?</p>;
  }
  return <p>Width: {window.innerWidth}px</p>;
}
```

```tsx
// ❌ localStorage — server'da yo'q
function UserPref() {
  const theme = localStorage.getItem('theme') ?? 'light';
  return <div className={theme}>...</div>;
}

// Yechim: useEffect ichida
function UserPref() {
  const [theme, setTheme] = useState('light');
  
  useEffect(() => {
    setTheme(localStorage.getItem('theme') ?? 'light');
  }, []);
  
  return <div className={theme}>...</div>;
}
```

### 4. Environment-specific render

```tsx
// ❌ Mismatch
function App() {
  return (
    <div>
      {process.env.NODE_ENV === 'development' && <DevPanel />}
    </div>
  );
}
```

### 5. User Agent / Locale farqi

```tsx
// ❌ Server'da default UTC, client'da local timezone
function FormattedDate({ date }: { date: Date }) {
  return <p>{date.toLocaleDateString()}</p>;
}
```

### 6. Conditional client-only rendering

```tsx
// ❌ Browser feature detection render davomida
function Modal() {
  const supportsDialog = typeof HTMLDialogElement !== 'undefined';
  return supportsDialog ? <dialog>...</dialog> : <div>...</div>;
}
```

### Yechimlar

| Muammo | Yechim |
|--------|--------|
| Time/Random | `useId` (ID uchun), client-only `useEffect` (display uchun) |
| Browser API | `useEffect` ichida o'qish, initial state SSR-safe |
| localStorage/sessionStorage | useEffect ichida, initial state default qiymat |
| Environment | Bir xil environment server va client'da |
| Timezone/Locale | Server va client'da bir xil locale ishlatish |
| Browser feature | `useEffect` ichida feature detection, after-mount render |

<details>
<summary><strong>Under the Hood</strong></summary>

**`useId` hook ishlash:**

`useId` SSR-safe unique ID generation. Server va client'da bir xil ID hosil qiladi.

```typescript
function mountId(): string {
  const hook = mountWorkInProgressHook();
  
  const root = getWorkInProgressRoot();
  const identifierPrefix = root.identifierPrefix;
  
  let id;
  
  if (getIsHydrating()) {
    const treeId = getTreeId();
    id = ':' + identifierPrefix + 'R' + treeId;
    
    const localId = localIdCounter++;
    if (localId > 0) {
      id += 'H' + localId.toString(32);
    }
    id += ':';
  } else {
    const globalClientId = globalClientIdCounter++;
    id = ':' + identifierPrefix + 'r' + globalClientId.toString(32) + ':';
  }
  
  hook.memoizedState = id;
  return id;
}
```

`treeContextId` — Fiber tree'da bu komponent qayerda joylashganini ifodalovchi unique ID. Server va client'da bir xil tree → bir xil ID. Mismatch bo'lmaydi.

**Mismatch detection algoritmi:**

```typescript
function diffHydratedProperties(
  domElement,
  fiberType,
  fiberProps,
  parentNamespace,
  rootContainerElement,
) {
  let updatePayload = null;
  
  if (domElement.tagName.toLowerCase() !== fiberType) {
    return;
  }
  
  for (const propKey in fiberProps) {
    const fiberValue = fiberProps[propKey];
    const domValue = getDOMValueForProp(domElement, propKey);
    
    if (!isEqual(fiberValue, domValue)) {
      updatePayload = updatePayload ?? [];
      updatePayload.push(propKey, fiberValue);
      
      if (__DEV__) {
        warnForPropDifference(propKey, domValue, fiberValue);
      }
    }
  }
  
  if (typeof fiberProps.children === 'string') {
    const domTextContent = domElement.textContent;
    if (fiberProps.children !== domTextContent) {
      if (__DEV__) {
        warnForTextDifference(domTextContent, fiberProps.children);
      }
    }
  }
  
  return updatePayload;
}
```

R18'da bu recovery **per-Suspense-boundary**. R19'da yanada yaxshilanish — element-level recovery.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Time-based mismatch yechimi:

```tsx
// ✅ useState lazy initial + useEffect
function CurrentTime() {
  const [now, setNow] = useState<string | null>(null);
  
  useEffect(() => {
    setNow(new Date().toLocaleString());
    const id = setInterval(() => {
      setNow(new Date().toLocaleString());
    }, 1000);
    return () => clearInterval(id);
  }, []);
  
  return <p>Now: {now ?? 'Loading...'}</p>;
}
```

Browser API mismatch yechim:

```tsx
function ScreenInfo() {
  const [width, setWidth] = useState<number | null>(null);
  
  useEffect(() => {
    setWidth(window.innerWidth);
    
    const onResize = () => setWidth(window.innerWidth);
    window.addEventListener('resize', onResize);
    return () => window.removeEventListener('resize', onResize);
  }, []);
  
  return <p>Width: {width ?? '?'}px</p>;
}
```

useId — SSR-safe unique IDs:

```tsx
function FormField({ label, type }: { label: string; type: string }) {
  const id = useId();
  
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} type={type} />
    </div>
  );
}
```

localStorage SSR-safe wrapper:

```tsx
function useLocalStorage<T>(key: string, defaultValue: T) {
  const [value, setValue] = useState<T>(defaultValue);
  
  useEffect(() => {
    try {
      const stored = localStorage.getItem(key);
      if (stored) {
        setValue(JSON.parse(stored));
      }
    } catch {}
  }, [key]);
  
  function update(newValue: T) {
    setValue(newValue);
    try {
      localStorage.setItem(key, JSON.stringify(newValue));
    } catch {}
  }
  
  return [value, update] as const;
}
```

Component'larni client-only qilish:

```tsx
function ClientOnly({ children }: { children: ReactNode }) {
  const [mounted, setMounted] = useState(false);
  
  useEffect(() => {
    setMounted(true);
  }, []);
  
  if (!mounted) return null;
  return <>{children}</>;
}

function App() {
  return (
    <div>
      <Header />
      <ClientOnly>
        <BrowserOnlyComponent />
      </ClientOnly>
      <Footer />
    </div>
  );
}
```

</details>

---

## suppressHydrationWarning

### Nazariya

**`suppressHydrationWarning`** — DOM element'ga qo'yiladigan **maxsus prop**. Bu prop bo'lgan element uchun React hydration mismatch warning'ni **chiqarmaydi**. Lekin **mismatch hali ham yuz beradi** — faqat warning ko'rsatilmaydi.

```tsx
function CurrentTime() {
  return (
    <span suppressHydrationWarning>
      {new Date().toLocaleString()}
    </span>
  );
}
```

### Qachon ishlatish

`suppressHydrationWarning` **kamdan-kam** ishlatilishi kerak:

1. **Tasodifiy time/date display** — "X minutes ago" indicator
2. **A/B test variant** — server va client variant farqli bo'lishi qabul qilinadi
3. **Critical performance** — alternativalar juda murakkab bo'lganda
4. **Third-party widget** — sizning kontrolingizda emas

### Cheklovlar

1. **Faqat shu element uchun** — child'larga propagatsiya qilinmaydi
2. **Children warnings hali ham chiqadi** — agar child'larda mismatch bo'lsa
3. **Client render server'dan farq qilishi mumkin** — foydalanuvchi flicker ko'radi
4. **Real bug'larni yashirishi mumkin** — diagnostic qiyin bo'ladi

### Anti-pattern: keng qo'llanilishi

```tsx
// ❌ Anti-pattern
function App() {
  return (
    <div suppressHydrationWarning>
      ...
    </div>
  );
}
```

```tsx
// ✅ To'g'ri
function App() {
  return (
    <div>
      <Header />
      <p>Server time: <span suppressHydrationWarning>{new Date().toLocaleString()}</span></p>
      <Articles />
    </div>
  );
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`suppressHydrationWarning` mexanizmi:**

```typescript
const SUPPRESS_HYDRATION_WARNING = 'suppressHydrationWarning';

function diffHydratedProperties(
  domElement,
  fiberType,
  fiberProps,
  ...
) {
  if (fiberProps[SUPPRESS_HYDRATION_WARNING] === true) {
    return null;
  }
  // ...
}
```

`suppressHydrationWarning` faqat `__DEV__` da effect beradi (production'da warning bormaydi). Lekin developers production'da ham e'tibor berishlari kerak — mismatch hali yuz beradi (silent ravishda).

`suppressContentEditableWarning` ga o'xshash boshqa prop — `contentEditable` bo'lgan element'larda children warning'ni suppress qiladi.

R17 va undan oldin: butun tree client'da qayta render qilinardi. R18+: faqat tegishli Suspense boundary qayta render qilinadi. R19: element-level recovery.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Time display'da `suppressHydrationWarning`:

```tsx
function ServerTimestamp() {
  return (
    <p>
      Sahifa yuklandi: <span suppressHydrationWarning>{new Date().toLocaleTimeString()}</span>
    </p>
  );
}
```

A/B test variant:

```tsx
function ABTestBanner() {
  return (
    <div suppressHydrationWarning>
      <BannerContent />
    </div>
  );
}
```

To'g'ri va noto'g'ri qo'llanilish:

```tsx
// ❌ Noto'g'ri — bug'larni yashiradi
function App() {
  return (
    <div suppressHydrationWarning>
      <Header />
      <Main />
      <Footer />
    </div>
  );
}

// ✅ To'g'ri — aniq nuqta
function Header() {
  return (
    <header>
      <h1>My App</h1>
      <span suppressHydrationWarning>
        {new Date().toLocaleTimeString()}
      </span>
    </header>
  );
}
```

Alternative: `<ClientOnly>` pattern flicker'siz:

```tsx
function ClientOnly({ children, fallback = null }: ClientOnlyProps) {
  const [mounted, setMounted] = useState(false);
  
  useEffect(() => {
    setMounted(true);
  }, []);
  
  return mounted ? <>{children}</> : <>{fallback}</>;
}

function App() {
  return (
    <div>
      <ClientOnly fallback={<span>Loading...</span>}>
        <CurrentTime />
      </ClientOnly>
    </div>
  );
}
```

</details>

---

## Selective Hydration (R18)

### Nazariya

**Selective Hydration** — R18'da kiritilgan xususiyat. **`<Suspense>` boundary'lar** orqali tree alohida bo'laklarga bo'linadi va har boundary **mustaqil** hydrate bo'ladi.

**Asosiy g'oya:**

```
Pre-R18 hydration:
1. Server butun HTML yuboradi
2. Client JS yuklaydi
3. React butun tree'ni hydrate qiladi (sync, blocking)
4. To'liq hydrate tugamaguncha — hech narsa interaktiv emas

R18 selective hydration:
1. Server initial shell + Suspense fallbacks yuboradi
2. Server async data tayyor bo'lganda — qism-qism HTML yuboradi
3. Client har Suspense boundary uchun mustaqil hydrate qiladi
4. Foydalanuvchi click qilgan elementning Suspense boundary BIRINCHI hydrate qilinadi
5. Boshqalari background'da hydrate
```

### User Interaction Priority

R18 asosiy yangiligi — **user interaction'ga responsivity**:

1. React click'ni "ushlaydi"
2. Click qilingan element'ning Suspense boundary'ini topadi
3. **Shu boundary'ni birinchi hydrate qiladi** (priority)
4. Hydration tugagach — click event chaqiriladi
5. Boshqa boundary'lar normal priority bilan hydrate

```tsx
function App() {
  return (
    <div>
      <Header />
      <Suspense fallback={<Skeleton />}>
        <Sidebar />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <Articles />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <Comments />
      </Suspense>
    </div>
  );
}
```

### Hydration Lane'lar

R18'da hydration uchun maxsus lane'lar (cross-ref `05-scheduler-lanes.md`):

| Lane | Vazifa |
|------|--------|
| `SyncHydrationLane` | Eng yuqori priority hydration |
| `InputContinuousHydrationLane` | Input event'lardan kelib chiqqan hydration |
| `DefaultHydrationLane` | Default hydration |
| `SelectiveHydrationLane` | User interaction priority |
| `IdleHydrationLane` | Background hydration |

User click'ga `SelectiveHydrationLane` priority beriladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Selective hydration mexanizmi:**

```typescript
function attemptHydrationAtCurrentPriority(fiber) {
  if (fiber.tag === SuspenseComponent && fiber.memoizedState !== null) {
    const lane = pickHighestPriorityLane(fiber.lanes);
    scheduleUpdateOnFiber(fiber, lane, eventTime);
  }
}

function dispatchEventForPluginEventSystem(
  domEventName,
  eventSystemFlags,
  nativeEvent,
  targetInst,
  targetContainer
) {
  if (targetInst === null) {
    const fiber = findClosestNonHydratedFiber(targetInst);
    if (fiber !== null) {
      attemptToDispatchEvent(domEventName, ...);
      attemptHydrationAtCurrentPriority(fiber);
    }
    return;
  }
  
  dispatchEventForLegacyPluginEventSystem(...);
}
```

**Priority elevation example:**

```typescript
function attemptToDispatchEvent(domEventName, eventSystemFlags, nativeEvent) {
  const targetInst = getClosestInstanceFromNode(nativeEvent.target);
  
  if (targetInst !== null) {
    if (targetInst.tag === HostComponent) {
      return targetInst;
    }
    
    if (targetInst.tag === SuspenseComponent) {
      const instance = targetInst.memoizedState.dehydrated;
      if (instance !== null) {
        const root = getRootFromHydrationContext(targetInst);
        const lane = getCurrentEventPriority();
        markRetryLaneIfNotHydrated(targetInst, lane);
        ensureRootIsScheduled(root);
        return null;
      }
    }
  }
  
  return null;
}
```

**Event replay:**

User click qilganida, agar element hali hydrate bo'lmagan bo'lsa:
1. React event'ni "saqlab qoladi" (replay queue'da)
2. Suspense boundary high-priority hydrate qilinadi
3. Hydration tugagach — saqlangan event replay qilinadi
4. Click event normal handler'ga yetadi

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Selective hydration ishlash misol:

```tsx
import { Suspense } from 'react';

function App() {
  return (
    <html>
      <head><title>Selective Hydration Demo</title></head>
      <body>
        <Header />
        <main>
          <Suspense fallback={<SidebarSkeleton />}>
            <Sidebar data={SidebarData} />
          </Suspense>
          
          <Suspense fallback={<ArticleSkeleton />}>
            <Article data={ArticleData} />
          </Suspense>
          
          <Suspense fallback={<CommentsSkeleton />}>
            <Comments data={CommentsData} />
          </Suspense>
        </main>
        <Footer />
      </body>
    </html>
  );
}

function Comments({ data }: { data: Resource<Comment[]> }) {
  const comments = data.read();
  
  return (
    <ul>
      {comments.map(c => (
        <li key={c.id}>
          <button onClick={() => alert('Reply!')}>
            Reply to {c.author}
          </button>
        </li>
      ))}
    </ul>
  );
}
```

Multiple boundaries — granular control:

```tsx
function ProductPage() {
  return (
    <div>
      <ProductHeader />
      <ProductImage />
      <ProductPrice />
      <BuyButton />
      
      <Suspense fallback={<Skeleton />}>
        <ProductDescription />
      </Suspense>
      
      <Suspense fallback={<Skeleton />}>
        <ProductReviews />
      </Suspense>
      
      <Suspense fallback={<Skeleton />}>
        <RelatedProducts />
      </Suspense>
    </div>
  );
}
```

</details>

---

## Streaming Hydration (R18)

### Nazariya

**Streaming SSR** — server butun HTML'ni bir vaqtda emas, **qism-qism** yuboradi. Bu — `renderToReadableStream` (Web Streams) yoki `renderToPipeableStream` (Node Streams) bilan ishlaydi.

**Asosiy g'oya:**

```
Pre-R18 SSR:
1. Server fetchData()  ← async, kutiladi
2. fetchData tugagach renderToString
3. HTML clientga yuboriladi
4. Client paint
5. JS yuklanadi
6. Hydrate
TTFB (Time To First Byte): kech (data wait)

R18 streaming SSR:
1. Server initial shell + Suspense fallback'larni darhol yuboradi
2. Foydalanuvchi initial shell'ni ko'radi
3. Server async data fetch qiladi (parallel)
4. Data tayyor bo'lganda → server yangi HTML chunk yuboradi
5. Brauzer mavjud skeleton'ni yangi content bilan almashtiradi
6. Hydration ham parallel ravishda har boundary uchun
```

### Streaming pipeline

```typescript
import { renderToPipeableStream } from 'react-dom/server';
import { Suspense } from 'react';

app.get('/', async (req, res) => {
  let didError = false;
  
  const { pipe, abort } = renderToPipeableStream(
    <App />,
    {
      onShellReady() {
        res.statusCode = didError ? 500 : 200;
        res.setHeader('Content-Type', 'text/html; charset=utf-8');
        pipe(res);
      },
      
      onShellError(error) {
        res.statusCode = 500;
        res.setHeader('Content-Type', 'text/html; charset=utf-8');
        res.send('<h1>Server Error</h1>');
      },
      
      onAllReady() {
        // Optional
      },
      
      onError(error) {
        didError = true;
        console.error(error);
      },
      
      bootstrapScripts: ['/bundle.js'],
    },
  );

  setTimeout(abort, 10000);
});
```

### Streaming HTML format

Server qism-qism HTML yuboradi. React buni qanday qiladi?

**Yechim — comment marker + script injection:**

React Suspense boundary'larini HTML comment marker'lar (`<!--$?-->`, `<!--/$-->`) bilan ifodalaydi va resolved content alohida `<div hidden>` ichida yuboriladi:

```html
<!-- 1-chunk: initial shell -->
<!DOCTYPE html>
<html>
<head><title>...</title></head>
<body>
  <div id="root">
    <div>Header</div>
    <!--$?--><template id="B:0"></template>Loading sidebar...<!--/$-->
    <!--$?--><template id="B:1"></template>Loading articles...<!--/$-->
  </div>
  <script src="/bundle.js"></script>

<!-- 2-chunk: Sidebar tayyor -->
<div hidden id="S:0">
  <div>Sidebar real content</div>
</div>
<script>$RC("B:0", "S:0")</script>

<!-- 3-chunk: Articles tayyor -->
<div hidden id="S:1">
  <div>Articles real content</div>
</div>
<script>$RC("B:1", "S:1")</script>

</body>
</html>
```

> **Eslatma:** Aniq encoding (comment marker'lar, template element'lari, `<div hidden>` placeholders, `$RC`/`$RM`/`$RB` function nomlari) — bu React internal implementation detail. R18-R19 versiyalarida bu detallar o'zgarishi mumkin. Asosiy printsip: HTML chunk'lar mavjud DOM'ga script orqali "patched" qilinadi.

**`$RC` — `replaceContent`:**

```javascript
function $RC(boundaryId, contentId) {
  const boundary = document.getElementById(boundaryId);
  const content = document.getElementById(contentId);
  
  while (content.firstChild) {
    boundary.parentNode.insertBefore(content.firstChild, boundary);
  }
  
  boundary.remove();
  content.remove();
}
```

Bu — DOM mutation orqali ishlaydi. React JS yuklanguncha ham streaming ishlaydi (HTML + inline script bilan).

<details>
<summary><strong>Under the Hood</strong></summary>

**`renderToPipeableStream` ichki ishi:**

```typescript
export function renderToPipeableStream(
  children: ReactNode,
  options?: RenderToPipeableStreamOptions
): PipeableStream {
  const request = createRequest(
    children,
    createResources(),
    createResponseState(
      options?.bootstrapScriptContent,
      options?.bootstrapScripts,
      options?.bootstrapModules,
    ),
    createRootFormatContext(options?.namespaceURI),
    options?.progressiveChunkSize,
    options?.onError,
    options?.onAllReady,
    options?.onShellReady,
    options?.onShellError,
    options?.onPostpone,
  );
  
  startWork(request);
  
  return {
    pipe(destination: Writable) {
      startFlowing(request, destination);
    },
    abort(reason?: any) {
      abort(request, reason);
    },
  };
}
```

**Suspense boundary ishlash:**

Suspense boundary `Suspense` Fiber sifatida render qilinadi. Server'da:

1. Suspense ichidagi child'lar render qilinadi
2. Agar child promise throw qilsa (Suspense suspends):
   - Boundary uchun fallback render qilinadi
   - Promise resolution kutilmoqda
   - Boundary "pending" deb belgilanadi
3. Initial shell — barcha boundary'lar fallback bilan
4. Promise resolved bo'lganda — boundary qayta render qilinadi
5. Yangi HTML chunk yuboriladi

**Bootstrap script:**

```typescript
const bootstrapChunk = `
  <script>
    function $RC(b, s) { ... }
  </script>
`;
```

Bu script — minified, inline. JS bundle yuklanguncha ishlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Streaming SSR — full app:

```tsx
import { Suspense } from 'react';
import { use } from 'react';

function App() {
  return (
    <html>
      <head><title>Streaming SSR</title></head>
      <body>
        <Header />
        <main>
          <Suspense fallback={<UserSkeleton />}>
            <UserProfile />
          </Suspense>
          <Suspense fallback={<PostsSkeleton />}>
            <Posts />
          </Suspense>
          <Suspense fallback={<CommentsSkeleton />}>
            <Comments />
          </Suspense>
        </main>
        <Footer />
      </body>
    </html>
  );
}

function UserProfile() {
  const user = use(fetchUser());
  return <div>User: {user.name}</div>;
}

function Posts() {
  const posts = use(fetchPosts());
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}

function Comments() {
  const comments = use(fetchComments());
  return <ul>{comments.map(c => <li key={c.id}>{c.text}</li>)}</ul>;
}
```

Streaming + Selective Hydration kombinatsiyasi:

```tsx
function App() {
  return (
    <html>
      <body>
        <div>Static header</div>
        <Suspense fallback={<Skeleton />}>
          <FastBoundary />
        </Suspense>
        <Suspense fallback={<Skeleton />}>
          <SlowBoundary />
        </Suspense>
      </body>
    </html>
  );
}
```

</details>

---

## R19 Hydration Improvements

### Nazariya

React 19 hydration'ni quyidagi yo'nalishlarda yaxshiladi:

### 1. Better error messages

R18 va undan oldin hydration mismatch uchun umumiy warning. R19'da batafsilroq diff:

```
Hydration failed because the server rendered HTML didn't match the client.
As a result this tree will be regenerated on the client. This can happen if
a SSR-ed Client Component used:

- A server/client branch `if (typeof window !== 'undefined')`.
- Variable input such as `Date.now()` or `Math.random()`.
- Date formatting in a user's locale that doesn't match the server.
- External changing data without sending a snapshot of it along with the HTML.
- Invalid HTML tag nesting.

  <App>
    <div>
      ^ Server: "Hello"
        Client: "World"
```

### 2. Recoverable error callbacks

R19'da `hydrateRoot` uchun yangi callbacks:

```tsx
hydrateRoot(container, <App />, {
  onCaughtError(error, errorInfo) {
    logError(error, { caught: true, componentStack: errorInfo.componentStack });
  },
  
  onUncaughtError(error, errorInfo) {
    logError(error, { caught: false, componentStack: errorInfo.componentStack });
  },
  
  onRecoverableError(error, errorInfo) {
    logWarning(error, { recovered: true });
  },
});
```

| Callback | Qachon chaqiriladi |
|----------|---------------------|
| `onCaughtError` | Error Boundary ushladi |
| `onUncaughtError` | Hech qanday boundary ushlamadi (rare, fatal) |
| `onRecoverableError` | Hydration mismatch yoki concurrent restart |

### 3. Element-level recovery (R19)

R18'da hydration mismatch bo'lganda Reconciler **yaqin Suspense boundary**'ni topib, butun shu boundary client'da qayta render qilardi. R19'da text/attribute mismatch holatida **faqat tegishli HostComponent/HostText** client'da almashtiriladi — boundary darajasiga ko'tarish shart emas:

```tsx
function App() {
  return (
    <Suspense fallback={<Skeleton />}>
      <Header />
      <Body />
      <Footer />
    </Suspense>
  );
}

// R18: butun Suspense boundary qayta render (Header, Body, Footer hammasi)
// R19: faqat farqli HostComponent/HostText qayta render (boundary daxlsiz)
```

> **Eslatma:** Komponent-level mismatch (masalan, server `<Article>`, client `<Comment>`) hamon **boundary-level recovery** talab qiladi. Element-level recovery faqat host-level mismatch (text content, attribute) uchun.

### 4. Hydration mismatch — silent recovery

R17 va undan oldin hydration mismatch warning + butun tree'ni client'da qayta render edi. R18'da bu **per-Suspense-boundary recovery**'ga o'zgartirildi va `onRecoverableError` callback orqali kuzatish mumkin bo'ldi — app crash qilmaydi. R19'da mexanizm yanada precise: element-level recovery + uchinchi tomon DOM mutation'larni "external" deb tan olish.

### 5. Third-party DOM modifications

R19 third-party browser extensions (masalan, Grammarly, AdBlock) DOM'ni o'zgartirsa — bu mismatch'ni "external" deb belgilaydi va silent ravishda recover qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**R19 hydration error pipeline:**

```typescript
function recoverFromHydrationMismatch(fiber, error) {
  const root = getRootForFiber(fiber);
  
  if (canRecoverElementLevel(fiber)) {
    fiber.flags |= Placement;
    return;
  }
  
  const suspenseBoundary = findClosestSuspenseBoundary(fiber);
  if (suspenseBoundary !== null) {
    suspenseBoundary.flags |= ForceClientRender;
  }
  
  if (root.onRecoverableError) {
    root.onRecoverableError(error, {
      digest: 'HYDRATION_FAILED',
      componentStack: getComponentStack(fiber),
    });
  }
}
```

**Element-level vs boundary-level recovery:**

```typescript
function canRecoverElementLevel(fiber: Fiber): boolean {
  if (fiber.tag === HostComponent || fiber.tag === HostText) {
    return true;
  }
  
  if (fiber.tag === FunctionComponent || fiber.tag === ClassComponent) {
    return false;
  }
  
  return false;
}
```

DOM-level mismatch (text, attribute) — element-level recovery. Komponent-level mismatch — boundary-level.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

R19 error callbacks misol:

```tsx
import { hydrateRoot } from 'react-dom/client';
import App from './App';

const container = document.getElementById('root')!;

hydrateRoot(container, <App />, {
  onCaughtError(error, errorInfo) {
    if (import.meta.env.PROD) {
      Sentry.captureException(error, {
        contexts: {
          react: { componentStack: errorInfo.componentStack },
        },
        tags: { handled: true },
      });
    }
  },
  
  onUncaughtError(error, errorInfo) {
    Sentry.captureException(error, {
      contexts: {
        react: { componentStack: errorInfo.componentStack },
      },
      tags: { handled: false, fatal: true },
      level: 'fatal',
    });
  },
  
  onRecoverableError(error, errorInfo) {
    if (import.meta.env.DEV) {
      console.warn('[Recoverable]', error.message);
    } else {
      Sentry.captureMessage('Hydration recoverable', {
        level: 'warning',
        contexts: {
          react: { componentStack: errorInfo.componentStack },
        },
      });
    }
  },
});
```

R19 mismatch handling silent:

```tsx
function CurrentTime() {
  return <span>{new Date().toLocaleTimeString()}</span>;
}

// R18 console:
// "Warning: Text content did not match..."
// "Hydration failed because..."

// R19 console:
// (silent — faqat onRecoverableError chaqiriladi)
// User'ga ko'rinmaydi
```

Browser extension handling (R19 silent):

```tsx
function App() {
  return <div>Hello, world</div>;
}

// Server HTML: <div>Hello, world</div>
// Client (Grammarly bilan): <div data-gramm="false">Hello, world</div>

// R19: silent — recognize external modification
```

</details>

---

## Edge Cases va Gotchas

### Hydration tugaguncha click — replay

```tsx
function App() {
  return (
    <button onClick={() => alert('Click!')}>
      Click me
    </button>
  );
}

// User click button ni bosadi:
// - R18'gacha: hech narsa bo'lmaydi
// - R18+: click "saqlanadi" → hydration tugagach replay → alert ishlaydi
```

R18 click replay faqat **discrete event'lar** uchun (click, submit). Continuous event'lar (mousemove) replay qilinmaydi.

---

### Conditional rendering va mismatch

```tsx
// ❌ Mismatch — environment-dependent
function App() {
  const isDev = process.env.NODE_ENV === 'development';
  return <div>{isDev && <DevTools />}</div>;
}
```

---

### Initial state — server initial bilan mos kelishi

```tsx
function Counter() {
  const [count, setCount] = useState(() => Math.random());
  return <p>{count}</p>;
}
```

---

### Hydration error during render — Suspense boundary recovery

```tsx
function App() {
  return (
    <Suspense fallback={<Skeleton />}>
      <Header />
      <Throws />
      <Footer />
    </Suspense>
  );
}
```

---

### Streaming SSR + Authentication

```tsx
function App({ user }: { user: User | null }) {
  return user
    ? <AuthenticatedView user={user} />
    : <LoginPage />;
}
```

Yechim 1: server'dan client'ga user'ni serialize qilish
Yechim 2: useEffect ichida fetch user
Yechim 3: framework (Next.js) — getInitialProps yoki RSC

---

## Common Mistakes

### ❌ Xato 1: `Date.now()` yoki `Math.random()` render davomida

```tsx
// ❌ Mismatch
function ID() {
  return <div id={`item-${Date.now()}`}>...</div>;
}

// ✅ useId
function ID() {
  const id = useId();
  return <div id={id}>...</div>;
}
```

---

### ❌ Xato 2: `localStorage` / `window` access render davomida

```tsx
// ❌ ReferenceError server'da
function Theme() {
  const theme = localStorage.getItem('theme');
  return <div className={theme ?? 'light'}>...</div>;
}

// ✅ useEffect ichida
function Theme() {
  const [theme, setTheme] = useState('light');
  
  useEffect(() => {
    setTheme(localStorage.getItem('theme') ?? 'light');
  }, []);
  
  return <div className={theme}>...</div>;
}
```

---

### ❌ Xato 3: `suppressHydrationWarning` keng qo'llanilishi

```tsx
// ❌ Anti-pattern — root level
<div suppressHydrationWarning>
  <App />
</div>
```

```tsx
// ✅ Aniq nuqta
<header>
  <h1>Title</h1>
  <span suppressHydrationWarning>{new Date().toLocaleTimeString()}</span>
</header>
```

---

### ❌ Xato 4: Eski `ReactDOM.hydrate` ishlatish

```tsx
// ❌ R18+ deprecated
import ReactDOM from 'react-dom';
ReactDOM.hydrate(<App />, container);
```

```tsx
// ✅ Yangi API
import { hydrateRoot } from 'react-dom/client';
hydrateRoot(container, <App />);
```

---

### ❌ Xato 5: `<ClientOnly>` wrapper'siz client-only komponent

```tsx
// ❌ Browser-only komponent SSR'da render qilinadi
function VideoPlayer() {
  if (!HTMLVideoElement.prototype.requestPictureInPicture) {
    return <div>Not supported</div>;
  }
  return <video controls .../>;
}
```

```tsx
// ✅ ClientOnly wrapper
function ClientOnly({ children }: { children: ReactNode }) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);
  return mounted ? <>{children}</> : null;
}

function App() {
  return (
    <ClientOnly>
      <VideoPlayer />
    </ClientOnly>
  );
}
```

---

## Amaliy Mashqlar

### Mashq 1: Hydration mismatch aniqlash (Oson)

Quyidagi har komponentda mismatch bormi? Sabab bilan tushuntiring.

```tsx
// A
function Greeting() {
  return <p>Hello, world!</p>;
}

// B
function Greeting({ name }: { name: string }) {
  return <p>Hello, {name}!</p>;
}

// C
function Greeting() {
  return <p>Random: {Math.random()}</p>;
}

// D
function Greeting() {
  const [count, setCount] = useState(0);
  return <p>Count: {count}</p>;
}

// E
function Greeting() {
  const id = useId();
  return <p id={id}>Hello!</p>;
}
```

<details>
<summary><strong>Javob</strong></summary>

| Komponent | Mismatch? | Sabab |
|-----------|-----------|-------|
| A | ❌ Yo'q | Static text, server va client bir xil |
| B | ❌ Yo'q | Prop server'dan keladi (initial render bir xil) |
| C | ✅ Ha | `Math.random()` har chaqiriqda boshqa qiymat |
| D | ❌ Yo'q | `useState(0)` — server va client'da initial 0 |
| E | ❌ Yo'q | `useId` SSR-safe, server va client bir xil ID hosil qiladi |

</details>

---

### Mashq 2: SSR-safe komponent yozish (O'rta)

`WindowSize` komponentni yozing — brauzer'ning innerWidth va innerHeight'ini ko'rsatadi. SSR-safe bo'lishi shart.

<details>
<summary><strong>Javob</strong></summary>

```tsx
function WindowSize() {
  const [size, setSize] = useState<{ width: number; height: number } | null>(null);
  
  useEffect(() => {
    function update() {
      setSize({
        width: window.innerWidth,
        height: window.innerHeight,
      });
    }
    
    update();
    
    window.addEventListener('resize', update);
    return () => window.removeEventListener('resize', update);
  }, []);
  
  if (!size) {
    return <p>Window size: ?</p>;
  }
  
  return <p>Window size: {size.width} × {size.height}</p>;
}
```

</details>

---

### Mashq 3: Selective hydration tahlil (O'rta)

Quyidagi senaryo'da React qaysi tartibda hydrate qiladi?

```tsx
function App() {
  return (
    <div>
      <Header />
      <Suspense fallback={<Skeleton />}>
        <Sidebar />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <Article />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <Comments />
      </Suspense>
    </div>
  );
}

// T=0: HTML keldi, JS yuklanmoqda
// T=100: JS yuklandi, hydration boshlanadi
// T=150: User Comments tugmasini bosdi
```

<details>
<summary><strong>Javob</strong></summary>

```
T=100: Hydration boshlanadi
  - Initial shell + Header sync hydrated
  - 3 Suspense boundary background hydration

T=150: User Comments tugmasini bosdi
  - React click event'ni "ushlaydi"
  - Comments boundary'ga SelectiveHydrationLane priority
  - Sidebar va Article hydration TO'XTAYDI
  - Click event "saqlanadi" (replay queue)

T=150-450: Comments priority hydration

T=450: Comments hydrated
  - Saqlangan click event replay qilinadi
  - Sidebar va Article hydration davom etadi

T=550-900: Sidebar va Article tugaydi
```

</details>

---

### Mashq 4: Streaming SSR yutuq tahlili (Qiyin)

3 yondashuv'ning Time To First Paint va TTI larini taqqoslang:

```
A. Client-only SPA (no SSR)
B. Pre-R18 SSR (renderToString, sync)
C. R18 Streaming SSR
```

<details>
<summary><strong>Javob</strong></summary>

| Metric | A (SPA) | B (Pre-R18 SSR) | C (R18 Streaming) |
|--------|---------|-----------------|-------------------|
| FCP | 200ms (skeleton) | 1600ms (to'liq) | 200ms (skeleton) |
| TTI (initial) | 200ms | 1750ms | 350ms |
| LCP | 1700ms | 1600ms | 900ms |
| SEO | ❌ | ✅ | ✅ |

C — eng yaxshi UX (FCP tez, LCP tez, SEO friendly).

</details>

---

### Mashq 5: R19 vs R18 hydration error handling (Qiyin)

```tsx
function App() {
  return (
    <Suspense fallback={<Skeleton />}>
      <Header />
      <CurrentTime />
      <Footer />
    </Suspense>
  );
}

function CurrentTime() {
  return <p>{Date.now()}</p>;
}
```

R18 va R19'da nima farqli?

<details>
<summary><strong>Javob</strong></summary>

**R18 (boundary-level recovery):**

- Console warning: text content mismatch
- Suspense boundary butunlay client'da qayta render
- Header, CurrentTime, Footer — barcha unmount + remount
- Foydalanuvchi flicker ko'radi

**R19 (element-level recovery):**

- onRecoverableError callback (silent log)
- Faqat `<p>` element replace qilinadi
- Header va Footer daxlsiz
- DOM identity saqlanadi

R19 UX sezilarli yaxshilangan — kichikroq subtree qayta render, foydalanuvchi flicker'ni kamroq ko'radi.

</details>

---

## Xulosa

Bu bo'limda Hydration mexanizmi yoritildi:

- **SSR + Hydration concept** — server HTML + client React attach
- **`hydrateRoot` vs `createRoot`** — DOM mavjud vs yo'q farqi
- **Hydration pipeline** — render Phase + DOM matching + event listener attach
- **Mismatch sabablari** — time, random, browser API, environment, locale
- **`suppressHydrationWarning`** — aniq nuqtada warning suppress
- **Selective hydration (R18)** — Suspense boundary'lar mustaqil hydrate
- **Streaming hydration (R18)** — server qism-qism HTML yuboradi
- **R19 improvements** — better errors, recoverable callbacks, element-level recovery

**QISM 2 (REACT INTERNALS) TUGADI** ✅

Kursning **fundamental qismi** quyidagi fayllarni qamrab oladi:
- `01-react-intro.md` — Mental model
- `02-rendering.md` — Render + Commit Phases
- `03-fiber-architecture.md` — Fiber tree
- `04-reconciliation.md` — Diffing algorithm
- `05-scheduler-lanes.md` — Priority sistemasi
- `06-hydration.md` — SSR + client attachment

Keyingi qismlardan boshlab — practical React (JSX, Components, State, Hooks). Internals'ni tushungandan keyin, har keyingi mavzu **shu asosga quriladi**.

---

**Keyingi bo'lim:** [07-jsx.md](07-jsx.md) — JSX Asoslari va Qoidalari: JSX vs TSX terminologiya, JSX vs HTML jadvali, Classic vs Automatic transform (R17+), JSX expressions, single root, fragments, reserved attribute names, spread attributes, `dangerouslySetInnerHTML` xavfi, conditional rendering.
